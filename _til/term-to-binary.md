---
title:      "erlang:term_to_binary/1 compatibility"
date:       2018-12-13
---

TIL that Erlang guarantees for the `:erlang.term_to_binary/1` format to be compatible
accross the span of at least two releases - it is surely going to be possible to
decode terms encoded in R18 in R20, but not necessarily in R21.

Its seems to be common knowledge for some, but I couldn't find any hard sources
for this, the closest thing is [this SO answer](https://stackoverflow.com/a/8283955/246337)
with messed up links. Do let me know if that's not correct.
