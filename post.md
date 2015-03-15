Whilst talking with an ex-colleague, a question came up on how to implement the Stable Marriage problem using a message passing approach. Naturally, I wanted to answer that question with Erlang!

Let’s first dissect the problem and decide what processes we need and how they need to interact with one another.

> The stable marriage problem is commonly stated as:
> 
> Given n men and n women, where each person has ranked all members of the opposite sex with a unique number between 1 and n in order of preference, marry the men and women together such that there are no two people of opposite sex who would both rather have each other than their current partners. If there are no such people, all the marriages are “stable”. (It is assumed that the participants are binary gendered and that marriages are not same-sex).
>
> -- <cite>Wikipedia</cite>

From the problem description, we can see that we need:
* a module for *man*
* a module for *woman*
* a module for orchestrating the experiment itself

In terms of interaction between the different modules, I imagined something along the line of the following:
![diagram](http://theburningmonk.com/WordPress/wp-content/uploads/2015/03/image38.png)

The proposal communications need to be synchronous as the man cannot proceed until he gets an answer from his proposal. But all other communications can be asynchronous.

Remember though, a synchronous call in the Erlang OTP-sense is not the same as a synchronous call in Java/C# where the calling thread is blocked.
In this case the communication still happens via asynchronous message passing, but the calling process asynchronously waits for a reply before moving on.

From here, the implementation itself is pretty straight forward.
First, for the `man` module, which models the behvaiour of a man in the experiment.

```erlang
-module(man).
-behaviour(gen_server).

-export([start_link/2, run/1, reject/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2, code_change/3]).
 
-record(state, {name = "", preferences = [], match = none}).

start_link(Name, Prefs) -> 
  gen_server:start_link({global, Name}, ?MODULE, [Name, Prefs], []).
run(Name) -> 
  gen_server:cast({global, Name}, run).
reject(Name, Woman) -> 
  gen_server:cast({global, Name}, {reject, Woman}).
 
init([Name, Prefs]) -> {ok, #state{name=Name, preferences=Prefs}}.
handle_call(_Request, _From, State) -> {reply, unsupported, State}.
 
handle_cast(_, State=#state{name=Name, preferences=[], match=none}) ->
  ok = stable_marriage:impossible(Name),
  {noreply, State};
handle_cast(run, State=#state{name=Name, preferences=[Woman|T], match=none}) ->
  case woman:propose(Woman, Name) of
    accept -> 
      ok = stable_marriage:matched(Name, Woman),
      {noreply, State#state{preferences=T, match=Woman}};
    reject ->
      run(Name), %% propose to the next woman
      {noreply, State#state{preferences=T}}
  end;
handle_cast({reject, Woman}, State=#state{name=Name, match=Woman}) ->
  ok = stable_marriage:unmatched(Name, Woman),
  run(Name), %% propose to the next woman
  {noreply, State#state{match=none}}.
 
handle_info(_Info, State) -> {noreply, State}.
terminate(_Reason, _State) -> ok.
code_change(_OldVsn, State, _Extra) -> {ok, State}.
```

The main thing to note from this module are the 3 `handle_cast` function clauses that, in order of execution:
1. if all preferences have been exhausted and we still haven't found a match, then concede that we can't match this man and report back to the **stable_marriage** module responsible for orchestrating and keeping track of matches;
2. when receives a command to **run** another round of the experiment, make proposal to the most preferred woman not yet proposed to. If proposal was accepted then update the state to remember the woman, and report to the **stable_marriage** module of the match
3. when received a rejection message from the previously matched woman, report to the **stable_marriage** module, and then send a message to self to propose to the next woman.

Similarly, for the `woman` module.

```erlang
-module(woman).
-behaviour(gen_server).
 
-export([start_link/2, propose/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2, code_change/3]).
 
-record(state, {name="", preferences=[], match=none }).
 
start_link(Name, Prefs) -> gen_server:start_link({global, Name}, ?MODULE, [Name, Prefs], []).
propose(Name, Man) -> gen_server:call({global, Name}, {proposal, Man}).
 
init([Name, Prefs]) -> {ok, #state{name=Name, preferences=Prefs}}.
 
handle_call({proposal, Man}, _From, State=#state{match=none}) ->
  {reply, accept, State#state{match=Man}};
handle_call({proposal, Man}, _From, State=#state{name=Name, preferences=Prefs, match=Match}) ->
  case is_better(Prefs, Match, Man) of
    true -> %% trade up
      man:reject(Match, Name),
      {reply, accept, State#state{match=Man}};
    false ->
      {reply, reject, State}
  end.
 
handle_cast(_Cast, State)  -> {noreply, State}.
handle_info(_Info, State)  -> {noreply, State}.
terminate(_Reason, _State) -> ok.
code_change(_OldVsn, State, _Extra) -> {ok, State}.
 
is_better([H|T], Current, New) ->
  case H of
    New -> true;
    Current -> false;
    _ -> is_better(T, Current, New)
  end.
```

Here, we need to handle proposal from a man, and accept the proposal if either:
* not yet matched to any man; or
* proposal came from a man that is more preferred, in which case asynchronously send a rejection message to the previously matched man, and update the state accordingly

To see if the new proposal came from a man that's more preferred to the current proposal, the `is_better` function recursively iterates through the woman's preferences until either man is found.
This way, we can get the answer without having to iterate through the entire list. (we iterate at most N-1 items)
A more naive approach would be to find the index of both man in the preferences list and compare them.

And finally, let's look at the `stable_marriage` module that is responsible for:
* orchestrating the experiment, and
* keeping track of who's matched to whom

```erlang
-module(stable_marriage).
-behaviour(gen_server).
 
-export([start/0, matched/2, unmatched/2, impossible/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2, code_change/3]).
 
-define(IDLE_TIMEOUT, 100). % if no activities after 100ms, then the couples have stablized
-record(state, {matched = maps:new() }).
 
start() -> gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).
matched(Man, Woman)   -> gen_server:cast(?MODULE, {matched, Man, Woman}).
unmatched(Man, Woman) -> gen_server:cast(?MODULE, {unmatched, Man, Woman}).
impossible(Man)       -> gen_server:cast(?MODULE, {impossible, Man}).
 
init([]) ->
  man:start_link("abe",  ["abi","eve","cath","ivy","jan","dee","fay","bea","hope","gay"]),
  man:start_link("bob",  ["cath","hope","abi","dee","eve","fay","bea","jan","ivy","gay"]),
  man:start_link("col",  ["hope","eve","abi","dee","bea","fay","ivy","gay","cath","jan"]),
  man:start_link("dan",  ["ivy","fay","dee","gay","hope","eve","jan","bea","cath","abi"]),
  man:start_link("ed",   ["jan","dee","bea","cath","fay","eve","abi","ivy","hope","gay"]),
  man:start_link("fred", ["bea","abi","dee","gay","eve","ivy","cath","jan","hope","fay"]),
  man:start_link("gav",  ["gay","eve","ivy","bea","cath","abi","dee","hope","jan","fay"]),
  man:start_link("hal",  ["abi","eve","hope","fay","ivy","cath","jan","bea","gay","dee"]),
  man:start_link("ian",  ["hope","cath","dee","gay","bea","abi","fay","ivy","jan","eve"]),
  man:start_link("jon",  ["abi","fay","jan","gay","eve","bea","dee","cath","ivy","hope"]),
 
  woman:start_link("abi",  ["bob","fred","jon","gav","ian","abe","dan","ed","col","hal"]),
  woman:start_link("bea",  ["bob","abe","col","fred","gav","dan","ian","ed","jon","hal"]),
  woman:start_link("cath", ["fred","bob","ed","gav","hal","col","ian","abe","dan","jon"]),
  woman:start_link("dee",  ["fred","jon","col","abe","ian","hal","gav","dan","bob","ed"]),
  woman:start_link("eve",  ["jon","hal","fred","dan","abe","gav","col","ed","ian","bob"]),
  woman:start_link("fay",  ["bob","abe","ed","ian","jon","dan","fred","gav","col","hal"]),
  woman:start_link("gay",  ["jon","gav","hal","fred","bob","abe","col","ed","dan","ian"]),
  woman:start_link("hope", ["gav","jon","bob","abe","ian","dan","hal","ed","col","fred"]),
  woman:start_link("ivy",  ["ian","col","hal","gav","fred","bob","abe","ed","jon","dan"]),
  woman:start_link("jan",  ["ed","hal","gav","abe","bob","jon","col","ian","fred","dan"]),
	
  man:run("abe"),
  man:run("bob"),
  man:run("col"),
  man:run("dan"),
  man:run("ed"),
  man:run("fred"),
  man:run("gav"),
  man:run("hal"),
  man:run("ian"),
  man:run("jon"),
 
  {ok, #state{}}.
 
handle_call(_Req, _From, State) -> {reply, unsupported, State}.
 
handle_cast({matched, Man, Woman}, State=#state{matched=Matched}) ->
  io:format("~s is engaged to ~s~n", [Man, Woman]),
  NewMatched = maps:put(Man, Woman, Matched),
  {noreply, #state{matched=NewMatched}, ?IDLE_TIMEOUT};
handle_cast({unmatched, Man, Woman}, State=#state{matched=Matched}) ->
  io:format("~s is separated from ~s~n", [Man, Woman]),
  NewMatched = maps:remove(Man, Matched),
  {noreply, #state{matched=NewMatched}, ?IDLE_TIMEOUT};
handle_cast({impossible, Man}, State) ->
  io:format("~s is bound to die alone...~n", [Man]),
  {noreply, State, ?IDLE_TIMEOUT}.
 
handle_info(timeout, State=#state{matched=Matched}) ->
  io:format("~nMarriages have stablized...~n", []),
  [io:format("~s is married to ~s~n", [Man, Woman]) 
   || {Man, Woman} <- maps:to_list(Matched)],
  {stop, normal, State}.
 
terminate(_Reason, _State)          -> ok.
code_change(_OldVsn, State, _Extra) -> {ok, State}.
```



Notice that I've taken the approach of "letting them run until they're quiet" here, with anidle timeout of 100ms - i.e. if no man has reported a new match or a broken match, then all marriages have stablized.

The dataset is taken from [Rosetta Code](http://rosettacode.org/wiki/Stable_marriage_problem), and running this experiment yields the following results:

![result](http://theburningmonk.com/WordPress/wp-content/uploads/2015/03/image39.png)

You can also get the source code for this solution on [github](https://github.com/theburningmonk/erlang-stable-marriage).
Hope you enjoyed this post, please feel free to check out my other Erlang-related [posts](http://theburningmonk.com/tags/erlang/).
