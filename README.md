# Collection of Erlang Parse Transforms

**Author**: Serge Aleynikov <saleyn(at)gmail.com>

**License**: MIT License

[![build](https://github.com/saleyn/etran/actions/workflows/erlang.yml/badge.svg)](https://github.com/saleyn/etran/actions/workflows/erlang.yml)
[![Hex.pm](https://img.shields.io/hexpm/v/etran.svg)](https://hex.pm/packages/etran)
[![Hex.pm](https://img.shields.io/hexpm/dt/etran.svg)](https://hex.pm/packages/etran)

This library includes useful parse transforms including Elixir-like pipeline operator for
cascading function calls.

# Content

| Module                | Description                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ |
| erlpipe               | Elixir-like pipeline for Erlang                                                      |
| mapreduce             | MapReduce: Fold Comprehension and FoldMap Comprehension                              |
| iif                   | Ternary if function including `iif/3`, `iif/4`, `ife/3`, `ife/4` parse transforms    |
| str                   | Stringification functions including `str/1`, `str/2`, and `throw/2` parse transforms |

## Erlang Pipeline (`erlpipe`)

Inspired by the Elixir's `|>` pipeline operator.
This transform makes code with cascading function calls much more readable by using the `/` as the
pipeline operator. The result of evaluation of the LHS expression is passed as an argument to
the RHS expression.

It transforms code from:

```erlang
print(L) when is_list(L) ->
  [lists:split(3, L)]  / element(1, _)          %% Function calls must be enclosed in `[...]`
                       / io:format("~s\n", [_]).

test1(Arg1, Arg2, Arg3) ->
  [Arg1, Arg2]                                  %% Variables must be enclosed in `[...]`
  / fun1
  / mod:fun2
  / fun3()                                      %% In function calls parenthesis are optional
  / fun4(Arg3, _)
  / io_lib:format("~p\n", [_])
  / fun6([1,2,3], _, other_param)
  / fun7.

test2() ->
  3       = "abc"      / length,                %% Strings  can be passed to '/' as is
  "abc"   = <<"abc">>  / binary_to_list,        %% Binaries can be passed to '/' as is
  "1,2,3" = {$1,$2,$3} / tuple_to_list          %% Tuples   can be passed to '/' as is
                       / [[I] || I <- _]
                       / string:join(_, ","),
  "abc\n" = "abc"      / (_ ++ "\n"),          %% Can use operators on the right hand side
  2.0     = 4.0        / max(1.0, 2.0),         %% Expressions with lhs floats are unmodified
  2       = 4          / max(1, 2).             %% Expressions with lhs integers are unmodified
```

to the following equivalent:

```erlang
print(L) when is_list(L) ->
  io:format("~s\n", [element(1, lists:split(3, L))]).

test1(Arg1, Arg2, Arg3) ->
  fun7(fun6([1,2,3],
            io_lib:format("~p\n", [fun4(Arg3, fun3(mod2:fun2(fun1(Arg1, Arg2))))]),
            other_param)).

test2() ->
  3       = length("abc"),
  "abc"   = binary_to_list(<<"abc">>),
  "1,2,3" = string:join([[I] || I <- tuple_to_list({$1,$2,$3})], ","),
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

## Map-Reduce: Fold and MapFold Comprehensions (`mapreduce`)

### Fold Comprehension

To invoke the fold comprehension transform include the initial state
assignment into a comprehension that returns a non-tuple expression:
```erlang
[S+I || S = 1, I <- L].
 ^^^    ^^^^^
```

In this example the `S` variable gets assigned the initial state `1`, and
the `S+I` expression represents the body of the fold function that
is passed the iteration variable `I` and the state variable `S`:
```erlang
lists:foldl(fun(I, S) -> S+I end, 1, L).
```

### MapFold Comprehension

To invoke the mapfold comprehension transform include the initial state
assignment into a comprehension, and return a tuple expression:
```erlang
[{I, S+I} || S = 1, I <- L].
 ^^^^^^^^    ^^^^^
```

In this example the `S` variable gets assigned the initial state `1`, and
the `{I, S+I}` two-elements tuple expression represents the body of the fold
function that is passed the iteration variable `I` and the state variable `S`:
```erlang
lists:mapfoldl(fun(I, S) -> S+I end, 1, L).
```

## Ternary if (`iif`)

This transform improves the code readability for cases that involve simple conditional tests.
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

## String transforms (`str`)

This module implements a transform to stringify an Erlang term.

* `str(Term)`       is equivalent to `lists:flatten(io_lib:format("~p", [Term]))`.
* `str(Fmt, Args)`  is equivalent to `lists:flatten(io_lib:format(Fmt,    Args))`.
* `throw(Fmt,Args)` is equivalent to `throw(lists:flatten(io_lib:format(Fmt, Args)))`.

## Dowloading

* [Github](https://github.com/saleyn/etran)
* [Hex.pm](https://hex.pm/packages/etran)

# Building and Using

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
