---
title:      "String.to_existing_atom/1"
subtitle:   "...is a double-edged sword"
disqus_identifier: "String.to_existing_atom/1 is a double-edged sword"
date:       2018-12-04 13:21:00
published:  true
---

I'd argue Elixir has relatively few gotchas. It's a simple and consistent language
and when you first learn it there's only a few things that are genuinely counter-intuitive
and catch you by surprise.

One of the examples could be the difference between
[binaries and charlists](https://elixir-lang.org/getting-started/binaries-strings-and-char-lists.html)
and why iex sometimes seems to do weird things to your lists:

```elixir
iex> l = [19, 7, 16, 119, 97, 116]
[19, 7, 16, 119, 97, 116]
iex> Enum.drop(l, 1)
[7, 16, 119, 97, 116]
iex> Enum.drop(l, 2)
[16, 119, 97, 116]
iex> Enum.drop(l, 3)
'wat'
```

One of the other ones comes when you start working with atoms and get a
little too trigger-happy with them. What you could hear from your more experienced
teammates is something like this:

> You shouldn't really use `String.to_atom/1` on user-supplied data. The BEAM has a limit
> on how many different atoms you can have and they're not garbage collected!
>
> With data coming from outside the system, stick to strings or use 
> `String.to_existing_atom/1` instead!

This is good advice and the [official docs](https://hexdocs.pm/elixir/String.html#to_atom/1)
agree. It seems like an easy choice too - if you take the approach that all atoms
you expect to see in the system are known at compile time and you won't be creating any new ones
during runtime, there's no reason not to do it! You get all the safety and no problems!

From my personal experience it's true the vast majority of time. But there are situations where
it could blow up when you least expect it (or just in production). Pull up a chair, let me 
tell you a story...

<!--more-->

### Storing atoms in Postgres

In [Curl](https://paywithcurl.com/) we use Postgres and Ecto for some of our data storage. 
There are some situations where we use the structs representing database rows almost directly,
so it makes sense to have the stored data as close to the desired Elixir representation as possible.

Let's say we have a table representing users and we expect all users to be either 
"active" or "inactive" (whatever it means in the business context). In our Elixir
code we'd like to see it as something like `%User{status: :active}` - an atom
struct field value.

Postgres isn't aware or bothered about what Elixir atoms are, but will happily
store them for us as strings. We can hide the boilerplate string <-> atom
conversion code by creating a 
[custom ecto type](https://hexdocs.pm/ecto/Ecto.Type.html). The code and
experience we end up with is very similar to the one described by Lew Parker in his
[A quick Dip into Ecto Types](https://www.glydergun.com/a-quick-dip-into-ecto-types/) blog post.

Here's pretty much the code we ended up with:

```elixir
defmodule MyApp.Ecto.AtomType do
  @behaviour Ecto.Type

  @type t :: atom

  @impl true
  def type(), do: :string

  @impl true
  def cast(atom) when is_atom(atom), do: {:ok, atom}

  def cast(string) when is_binary(string), do: safe_string_to_atom(string)

  def cast(_), do: :error

  @impl true
  def load(value), do: safe_string_to_atom(value)

  @impl true
  def dump(atom) when is_atom(atom), do: {:ok, Atom.to_string(atom)}

  def dump(_), do: :error

  @spec safe_string_to_atom(String.t()) :: {:ok, atom} | :error
  defp safe_string_to_atom(str) do
    try do
      {:ok, String.to_existing_atom(str)}
    rescue
      ArgumentError -> :error
    end
  end
end
```

Looks pretty reasonable to me, and again: the `String.to_exsting_atom/1` should
never be a problem, because we have the `:active` and `:inactive` atom literals
in our codebase and we even have a database constraint to make sure those
are the only two values that can be stored in the `status` column. 

### Some time passes...

Time passes, features are added, refactors happen. One day we deploy to production
(code that has behaved well on staging environment for a while and passed all
system and unit tests with flying colours) and AppSignal starts notifying
us about errors:

```
cannot load `"inactive"` as type MyApp.Ecto.AtomType for field `status` in schema (...)
```

What gives?! I know `:inactive` is an atom that exists and the same code has
had no problems in the staging environment! We investigate and come up
with a theory that gets confirmed by a 
[two-year-old github comment by José](https://github.com/elixir-lang/elixir/issues/4832#issuecomment-227099444).

Here's what happened:

The `:inactive` literal was in our codebase, but not in the modules
that get used the most frequently on a day-to-day basis. The atoms in them
don't get added to the atoms table until the module gets loaded, which happens
lazily - when a function in it is called, for example.

Those modules got loaded
when system tests were run in the staging environment (when testing scenarios with
users getting deactivated) but not straight away in production. In production
the modules weren't loaded (yet) and when an `inactive` user was retrieved
from the database, the field loading errored out breaking some features.

### Fixing it

How do we fix it then? 

First we make sure the production system works - we go through some user
scenario which uses a Module with the atoms we need. All systems are nominal again.
Now we can approach the root cause with less urgency.

There's a couple of ways of fixing the problem itself. One was
[suggested by José](https://github.com/elixir-lang/elixir/issues/4832#issuecomment-227099444):
Make sure whenever there's a chance we'll need the atoms we're
depending on, we'll load their modules. In our case we could be doing this 
in our [`Repo`](https://hexdocs.pm/ecto/Ecto.Repo.html) module, which always 
gets used whenever we talk to the database:

```elixir
# in lib/my_app/repo.ex

@on_load :load_atoms

def load_atoms() do
  relevant_modules = [
    MyApp.User,
    MyApp.DisablingFlow 
  ]
  
  Enum.each(relevant_modules, &Code.ensure_loaded?/1)
  :ok
end
```

You can see we're using 
[module's `@on_load`](https://hexdocs.pm/elixir/Module.html#module-on_load)
attribute to hook into the module's lifecycle and "cascade" the module loading.

That's not the only possible solution though. Another, technically simpler
solution is just reverting to `String.to_atom/1` when we load
data from the database. That would be working under the assumption
we knew what we were doing when we were storing them in the first place :)
That would just be changing the `AtomType.load/1` function:

```elixir
# Before:
def load(value), do: safe_string_to_atom(value)

# After:
def load(value), do: {:ok, String.to_atom(value)}
```

The latter approach might look a bit naive, but it saves us from
the hassle of what looks like manually tracking dependencies between
modules.

There's more possible approaches here: We could potentially start the
system in 
[embedded mode](http://erlang.org/doc/system_principles/system_principles.html#code_loading),
where all code is loaded at startup,
[provided we're deploying the app using releases](https://github.com/elixir-lang/elixir/issues/7208#issuecomment-358603587).
While simplifying the app's lifecycle like this sounds like a
clean solution, I think it's conservative to not to depend on some deployment
details for the correctness of your app. 
Getting started with releases requires some extra work too.

So anyways, crisis averted and we learned something!

### End notes

This is not actually how we model our users and it's a different entity
which made the problem surface - it's just a simplification for the sake of the 
blog post so I wouldn't have to get too deep into describing how we model Curl's
domain.

There's probably some even cleaner ways of solving the problem we ran into.
If you have some ideas about them, leave a comment below, I'd love to hear
about it!
