---
title:      "A 65 byte binary is large"
date:       2017-11-15
---

I knew that in Erlang/Elixir each of the processes has its own heap and sending
messages between processes copies the terms sent, which is why sending large
amounts of data might be inefficient. I also knew that "large binaries (strings)"
live in a shared heap and processes operate on references to them, so sending
those around is cheap.

TIL that any binary longer than 64 bytes is considered a
[large binary](http://erlang.org/doc/efficiency_guide/binaryhandling.html#refc_binary),
which was probably a deliberate design choice.
