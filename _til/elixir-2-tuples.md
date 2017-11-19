---
title:      "Elixir tuples' AST"
date:       2017-11-11
---

TIL that the
[AST](http://elixir-lang.github.io/getting-started/meta/quote-and-unquote.html#quoting)
of Elixir tuples isn't entirely consistent - the
2-tuples are literals:
```
assert {:{}, _line, []} = Code.string_to_quoted!("{}")
assert {:{}, _line, [1]} = Code.string_to_quoted!("{1}")
assert {1, 2} = Code.string_to_quoted!("{1, 2}")
assert {:{}, _line, [1, 2, 3]} = Code.string_to_quoted!("{1, 2, 3}")
assert {:{}, _line, [1, 2, 3, 4]} = Code.string_to_quoted!("{1, 2, 3, 4}")
assert {:{}, _line, [1, 2, 3, 4, 5]} = Code.string_to_quoted!("{1, 2, 3, 4, 5}")
```

Apparently it is this way to make [Keywords](https://hexdocs.pm/elixir/Keyword.html)
literals as well.
