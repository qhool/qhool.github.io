---
layout: post
title:  "Debugging Elixir"
date:   2014-02-06 21:26:00
categories: elixir
---
Erlang has a graphical debugger, which you can use to step through source code, set breakpoints, etc.  You can start the debugger from the shell, and select a source file to debug from the menu, or (as shown), by calling the `int` module (the debugger's code interpreter http://www.erlang.org/doc/man/int.html)

{% highlight erlang %}
1> debugger:start().
{ok,<0.39.0>}
2> int:i("./src/mymodule.erl").
{module,mymodule}
{% endhighlight %}

Now lets try it with elixir: (Note single-quote erlang-style strings)

{% highlight elixir %}
Interactive Elixir (0.12.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> :debugger.start()
{:ok, #PID<0.48.0>}
iex(2)> :int.i('./lib/mymodule.ex')
** Invalid beam file or no abstract code: "./lib/mymodule.ex"
:error
{% endhighlight %}

Same results if we try loading the `.beam` file instead:

{% highlight elixir %}
iex(4)> :int.i('./ebin/Elixir.Qhool.MyModule.beam')
** Invalid beam file or no abstract code: "./ebin/Elixir.Qhool.MyModule.beam"
:error
{% endhighlight %}

Looking at `int.erl` in the otp source (https://github.com/erlang/otp), we see:
{% highlight erlang %}
i(AbsMods) -> i2(AbsMods, local, ok).
{% endhighlight %}

which leads us to:

{% highlight erlang %}
i2(AbsMod, Dist, _Acc) when is_atom(AbsMod); is_list(AbsMod); is_tuple(AbsMod) ->
    int_mod(AbsMod, Dist).
{% endhighlight %}

which goes to:
{% highlight erlang %}
int_mod(AbsMod, Dist) when is_atom(AbsMod); is_list(AbsMod) ->
    case check(AbsMod) of
{% endhighlight %}

Eventually, this winds up loading the source of the module, and attempting to parse it, which fails, because it's elixir.  If you pass the module name instead, it fails when it looks for `src/Elixir.Qhool.MyModule.erl`. (offending code is in `check_file/1` and `find_src/1`, respectively).

Going back to `int_mod/2`, there's another clause:

{% highlight erlang %}
int_mod({Mod, Src, Beam, BeamBin}, Dist)
  when is_atom(Mod), is_list(Src), is_list(Beam), is_binary(BeamBin) ->
{% endhighlight %}

Since the first argument to `i/1` passes through unchanged to int_mod, we can call it with this 4-tuple directly.  `Mod` is the module name, `Src` and `Beam` are just the filenames, and `BeamBin` is a binary (aka elixir string) with the contents of the .beam -- all we need to do is load the beam file contents into a variable, and construct the tuple:

{% highlight elixir %}
iex(9)> {:ok,beam_bin} = File.read("./ebin/Elixir.Qhool.MyModule.beam")
{:ok,
 <<70, 79, 82, 49, 0, 0, 7, 84, 66, 69, 65, 77, 65, 116, 111, 109, 0, 0, 0, 113, 0, 0, 0, 11, 21, 69, 108, 105, 120, 105, 114, 46, 81, 104, 111, 111, 108, 46, 77, 121, 77, 111, 100, 117, 108, 101, 8, 95, 95, 105, ...>>}
iex(10)> :int.i({Qhool.MyModule,'./lib/mymodule.ex','./ebin/Elixir.Qhool.MyModule.beam',beam_bin})
{:module, Qhool.MyModule}
{% endhighlight %}

And now the module shows up in the debugger.  All the usual stuff will now work; you can set breakpoints, step through execution, and so forth.

Here's the same procedure in erlang:

{% highlight erlang %}
Eshell V5.10.3  (abort with ^G)
1> debugger:start().
{ok,<0.39.0>}
2> Beam = "./ebin/Elixir.Qhool.MyModule.beam".
"./ebin/Elixir.Qhool.MyModule.beam"
3> {ok,BeamBin} = file:read_file(Beam).
{ok,<<70,79,82,49,0,0,7,84,66,69,65,77,65,116,111,109,0,
      0,0,113,0,0,0,11,21,69,108,...>>}
4> int:i({'Elixir.Qhool.MyModule',"./lib/mymodule.ex",Beam,BeamBin}).
{module,'Elixir.Qhool.MyModule'}
{% endhighlight %}

One interesting thing to note, in the elixir version, you see the bare module name `Qhool.MyModule` in the input and output to ':int.i()'; elixir is really representing this internally with the atom `'Elixir.Qhool.MyModule'` and this is what 'int:i/1' receives, and returns in both versions:

{% highlight elixir %}
iex(1)> :io.format('~p~n',[Qhool.MyModule])
'Elixir.Qhool.MyModule'
:ok
{% endhighlight %}
