---
layout: post
title: Redefining Elixir modules
subtitle: Black voodoo magic from ELIXIR
published: true
date: '2020-02-08'
tags: [elixir, iex]
---

I want to talk about one of the most AWESOME things that Elixir features. It is the ability to redefine module on the fly from `iex` shell.

For TLDR folks - scroll to the [bottom](#asciicast) and watch the `asciicast`.

So, what does that mean?

Imagine that we have a system running simple `GenServer` that prints pre-compiled `@time` variable every one second.
Unrelated btw `System.monotonic_time/0` is **the way** to track durations in Elixir, see [System](https://hexdocs.pm/elixir/System.html) for more info.

Something like this:

```elixir
defmodule ModuleRedefining.Worker do
  @moduledoc false

  use GenServer

  @time System.monotonic_time(:second)

  def start_link(args) do
    GenServer.start_link(__MODULE__, args, name: __MODULE__)
  end

  def init(_) do
    send_after()
    {:ok, %{}}
  end

  def handle_info(:print, state) do
    IO.inspect(@time)
    send_after()
    {:noreply, state}
  end

  defp send_after do
    Process.send_after(self(), :print, 1_000)
  end
end
```

So we launch our app, fire up our `GenServer` (in supervision or manually) and execute to our `iex` shell.
We should see the output something like:

```
-576460752
-576460752
-576460752
-576460752
-576460752
(...)
```

Then we copy (in clipboard) our module and paste it in the running `iex` shell.
Elixir will return warning that we are indeed redefining the module, and its new code representation (btw. if interested in Elixir's AST, check this [article](https://by-cha.se/working-with-the-elixir-ast.html) out).

```
warning: redefining module ModuleRedefining.Worker (current version loaded from _build/dev/lib/module_redefining/ebin/Elixir.ModuleRedefining.Worker.beam)
{:module, ModuleRedefining.Worker,
 <<70, 79, 82, 49, 0, 0, 15, 124, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 1, 188, 0, 0, 0, 43, 30, 69, 108, 105, 120, 105, 114 46, 77, 111, 100, 117, 108, 101, 82, 101, 100, 101, 102, 105, 110, 105, 110, ...>>, {:send_after, 0}}
```

Then we would see **changed** output to something like:

```
-576460745
-576460745
-576460745
-576460745
(...)
```

So what just happened?

When we pasted the module code, elixir recompiled it and redefined the existing module. By compiling the code, `@time` compile-time constant has changed so our output changed as well.

With great power comes great responsibility, so use this cautiously. I find this very useful when debugging some weird problems on our staging environment, so I can quickly tryout new code variations and patches. For development environment, i just change the source and `recompile` in `iex` shell.

<div id="asciicast">
  <script id="asciicast-299258" src="https://asciinema.org/a/299258.js" async></script>
</div>