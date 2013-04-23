# lye

[![Build Status](https://travis-ci.org/elementerl/lye.png)](https://travis-ci.org/elementerl/lye)

Check the structure of your data. A bit like schemas, only less painful. 

lye is [semantically versioned](http://semver.org/).

## Usage

`lye:check/2` takes some structure as its first argument, a list of singleton
functions or a module that exports a bunch singleton functions ending in
`_spec` and maps these functions over the structure.

Say we have a module like this:

```erlang
-module(host_collection).

-export([must_have_dns_name_spec/1, must_have_latency_spec/1]).
-export([latency_must_be_less_than_9us_spec/1]).

must_have_dns_name_spec({network_host, Props}) ->
    case proplists:is_defined(dns_name, Props) of
        true -> ok;
        false -> {error, <<"must have dns_name">>}
    end.

must_have_latency_spec({network_host, Props}) ->
    case proplists:is_defined(latency_us, Props) of
        true -> ok;
        false -> {error, <<"must have latency">>}
    end.

latency_must_be_less_than_9us_spec({network_host, Props}) ->
    case proplists:get_value(latency_us, Props, infinity) < 9 of
        true -> ok;
        false -> {error, latency_too_high}
    end.
```

The interaction with lye in the REPL is:

```
> make repl
Erlang R15B02 (erts-5.9.2) [source] [64-bit] [smp:8:8] [async-threads:0] [hipe] [kernel-poll:false]

Eshell V5.9.2  (abort with ^G)
1> c("examples/host_collection").
{ok,host_collection}
2> ValidStructure = {network_host, [{latency_us, 5}, {dns_name, <<"foo.example.com">>}]}.
{network_host,[{latency_us,5},
               {dns_name,<<"foo.example.com">>}]}
3> NoLatencyStructure = {network_host, [{dns_name, <<"nolat.example.com">>}]}.
{network_host,[{dns_name,<<"nolat.example.com">>}]}
4> NoNameStructure = {network_host, [{latency_us, 10}]}.
{network_host,[{latency_us,10}]}
5> lye:check(ValidStructure, [host_collection]).
ok
6> lye:check(NoLatencyStructure, [host_collection]).
[{error,<<"must have latency">>},{error,latency_too_high}]
7> lye:check(NoNameStructure, [host_collection]).
[{error,<<"must have dns_name">>},{error,latency_too_high}]
8> MustBeNetworkHostSpec = fun({network_host, _}) -> ok end.
#Fun<erl_eval.6.82930912>
9> lye:check(ValidStructure, [host_collection, MustBeNetworkHostSpec]).
ok
```

`lye:check/2` is much nicer to work with when you aren't limited entirely to
defining funs in the REPL.

## Configuration

You can configure the blacklist of check function names and the check function
selector regex by adding a section for lye in your app's config:

```
[
    {lye, [
        {spec_pattern, ".*"},
        {blacklisted_funs, [bar, qux]}
    ]}
]
```

That will ensure that all functions in the check modules are treated as specs,
except the check functions named `bar` and `qux`. Note that the `module_info`
function is *always* added to the blacklist. (And yes, we know that technically
`bar` is not an Erlang function name because it is lacking an arity, but check
functions must have an arity of 1, which makes`'bar/1'` redundant in addition
to being hard to type.)

## Examples

See the `examples` directory.

## License

lye is released under the [MIT license](http://opensource.org/licenses/MIT). See
LICENSE.md for more details.
