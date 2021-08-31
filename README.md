# Collection of Erlang Parse Transforms

**Author**: Serge Aleynikov <saleyn(at)gmail.com>

**License**: MIT License

[![build](https://github.com/saleyn/etran/actions/workflows/erlang.yml/badge.svg)](https://github.com/saleyn/etran/actions/workflows/erlang.yml)
[![Hex.pm](https://img.shields.io/hexpm/v/etran.svg)](https://hex.pm/packages/etran)
[![Hex.pm](https://img.shields.io/hexpm/dt/etran.svg)](https://hex.pm/packages/etran)

This library includes useful parse transforms including Elixir-like pipeline operator for
cascading function calls.

## Content

| Module                | Description                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ |
| erlpipe               | Elixir-like pipeline operator for Erlang                                             |
| foldcomp              | Fold Comprehension and Indexed List Comprehension                                    |
| iif                   | Ternary if function including `iif/3`, `iif/4`, `ife/3`, `ife/4` parse transforms    |
| str                   | Stringification functions including `str/1`, `str/2`, and `throw/2` parse transforms |

## `erlpipe`: Erlang Pipe Operator

Inspired by the Elixir's `|>` pipeline operator.
This transform makes code with cascading function calls much more readable by using the `/` as the
pipeline operator. In the `LHS / RHS / ... Last.` notation, the result of evaluation of the LHS
expression is passed as an argument to the RHS expression. This process continues until the `Last`
expression is evaluated.  The head element of the pipeline must be either a term to which the
arithmetic division `/` operator cannot apply (i.e. not integers, floats, functions), or if you
need to pass integer(s) or float(s), wrap them in a list brackets.

It transforms code from:

```erlang
test1(Arg1, Arg2, Arg3) ->
  [Arg1, Arg2]                                  %% Arguments must be enclosed in `[...]`
  / fun1                                        %% In function calls parenthesis are optional
  / mod:fun2
  / fun3()
  / fun4(Arg3, _)                               %% '_' is the placeholder for the return value of a previous function
  / io_lib:format("~p\n", [_])
  / fun6([1,2,3], _, other_param)
  / fun7.

print(L) when is_list(L) ->
  [3, L]                                        %% Multiple items in a list are passed as arguments to the first function
  / lists:split
  / element(1, _)
  / binary_to_list
  / io:format("~s\n", [_]).

test2() ->
  3       = abc        / atom_to_list / length, %% Atoms    can be passed to '/' as is
  3       = "abc"      / length,                %% Strings  can be passed to '/' as is
  "abc"   = <<"abc">>  / binary_to_list,        %% Binaries can be passed to '/' as is
  "1,2,3" = {$1,$2,$3} / tuple_to_list          %% Tuples   can be passed to '/' as is
                       / [[I] || I <- _]
                       / string:join(_, ","),
  "1"     = [min(1,2)] / integer_to_list,       %% Function calls, integer and float values must be passed as a list
  "1"     = [1]        / integer_to_list,
  "1.0"   = [1.0]      / float_to_list(_, [{decimals,1}]),
  "abc\n" = "abc"      / (_ ++ "\n"),           %% Can use operators on the right hand side
  2.0     = 4.0        / max(1.0, 2.0),         %% Expressions with lhs floats are unmodified
  2       = 4          / max(1, 2).             %% Expressions with lhs integers are unmodified
```

to the following equivalent:

```erlang
test1(Arg1, Arg2, Arg3) ->
  fun7(fun6([1,2,3],
            io_lib:format("~p\n", [fun4(Arg3, fun3(mod2:fun2(fun1(Arg1, Arg2))))]),
            other_param)).

print(L) when is_list(L) ->
  io:format("~s\n", [binary_to_list(element(1, lists:split(3, L)))]).

test2() ->
  3       = length(atom_to_list(abc)),
  3       = length("abc"),
  "abc"   = binary_to_list(<<"abc">>),
  "1,2,3" = string:join([[I] || I <- tuple_to_list({$1,$2,$3})], ","),
  "1"     = integer_to_list(min(1,2)),
  "1"     = integer_to_list(1),
  "1.0"   = float_to_list(1.0, [{decimals,1}]),
  "abc\n" = "abc" ++ "\n",
  2.0     = 4.0 / max(1.0, 2.0),
  2       = 4   / max(1, 2).
```

Similar attempts to tackle this pipeline transform have been done by other developers:

* https://github.com/stolen/pipeline
* https://github.com/oltarasenko/epipe
* https://github.com/clanchun/epipe
* https://github.com/pouriya/pipeline

Yet, we subjectively believe that the choice of syntax in this implementation of transform
is more succinct and elegant, and doesn't attempt to modify the meaning of the `/` operator
for arithmetic LHS types (i.e. integers and floats).

## `foldcomp`: Fold and Indexed List Comprehensions

### Indexed List Comprehension

Occasionally the body of a list comprehension needs to know the index
of the current item in the fold.  Consider this example:
```erlang
[{1,10}, {2,20}] = element(1, lists:foldmapl(fun(I, N) -> {{N, I}, N+1} end, 1, [10,20])).
```
Here the `N` variable is tracking the index of the current item `I` in the list.
While the same result in this specific case can be achieved with
`lists:zip(lists:seq(1,2), [10,20])`, in a more general case, there is no way to have
an item counter propagated with the current list comprehension syntax.

The **Indexed List Comprehension** accomplishes just that through the use of an unassigned
variable immediately to the right of the `||` operator:
```erlang
  [{Idx, I} || Idx, I <- L].
%              ^^^
%               |
%               +--- This variable becomes the index counter
```
Example:
```erlang
[{1,10}, {2,20}] = [{Idx, I} || Idx, I <- [10,20]].
```

### Fold Comprehension

To invoke the fold comprehension transform include the initial state
assignment into a comprehension that returns a non-tuple expression:
```erlang
  [S+I || S = 1, I <- L].
%  ^^^    ^^^^^
%   |       |
%   |       +--- State variable bound to the initial value
%   +----------- The body of the foldl function
```

In this example the `S` variable gets assigned the initial state `1`, and
the `S+I` expression represents the body of the fold function that
is passed the iteration variable `I` and the state variable `S`:
```erlang
lists:foldl(fun(I, S) -> S+I end, 1, L).
```

A fold comprehension can be combined with the indexed list comprehension
by using this syntax:

```erlang
  [do(Idx, S+I) || Idx, S = 10, I <- L].
%  ^^^^^^^^^^^^    ^^^  ^^^^^^
%       |           |     |
%       |           |     +--- State variable bound to the initial value (e.g. 10)
%       |           +--------- The index variable bound to the initial value of 1
%       +--------------------- The body of the foldl function can use Idx and S
```

This code is transformed to:
```erlang
element(2, lists:foldl(fun(I, {Idx, S}) -> {Idx+1, do(Idx, S+I)} end, {1, 10}, L)).
```

Example:
```erlang
33 = [S + Idx*I || Idx, S = 0, I <- [10,20]],

[print(Idx, I, S) || Idx, S=0, I <- [10,20]].
% Prints:
%   Item#1 running sum: 10
%   Item#2 running sum: 30

print(Idx, I, S) ->
  Res = S+I,
  io:format("Item#~w running sum: ~w\n", [Idx, Res]),
  Res.
```

## `iif`: Ternary if

This transform improves the code readability for cases that involve simple conditional
`if/then/else` tests in the form `iif(Condition, Then, Else)`.  Since this is a parse
transform, the `Then` and `Else` expressions are evaluated **only** if the `Condition`
evaluates to `true` or `false` respectively.

E.g.:

```erlang
iif(tuple_size(T) == 3, good, bad).

iif(some_fun(A), match, ok, error).

nvl(L, undefined).

nvl(L, nil, hd(L))
```

are transformed to:

```erlang
case tuple_size(T) == 3 of
  true      -> good;
  _         -> bad
end.

case some_fun(A) of
  match     -> ok;
  nomatch   -> error
end.

case L of
  []        -> undefined;
  false     -> undefined;
  undefined -> undefined;
  _         -> L
end.

case L of
  []        -> nil;
  false     -> nil;
  undefined -> nil;
  _         -> hd(L)
end.
```

## `str`: String transforms

This module implements a transform to stringify an Erlang term.

* `str(Term)`       is equivalent to `lists:flatten(io_lib:format("~p", [Term]))`.
* `str(Fmt, Args)`  is equivalent to `lists:flatten(io_lib:format(Fmt,    Args))`.
* `throw(Fmt,Args)` is equivalent to `throw(lists:flatten(io_lib:format(Fmt, Args)))`.

## Dowloading

* [Github](https://github.com/saleyn/etran)
* [Hex.pm](https://hex.pm/packages/etran)

## Building and Using

```
$ make
```

To use the transforms, compile your module with the `+'{parse_transform, Module}'` command-line
option, or include `-compile({parse_transform, Module}).` in your source code, where `Module`
is one of the transform modules implemented in this project.

To use all transforms implemented by the `etran` application, compile your module with this
command-line option: `+'{parse_transform, etran}'`.
```
erlc +debug_info +'{parse_transform, etran}' -o ebin YourModule.erl
```

If you are using `rebar3` to build your project, than add to `rebar.config`:
```
{erl_opts, [debug_info, {parse_transform, etran}]}.
```
