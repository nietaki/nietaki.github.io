---
title:      "I'm stealing API keys from your site"
subtitle:   "Hoplon - protecting yourself from malicious packages"
disqus_identifier: "Hoplon - protecting yourself from malicious packages"
date:       2018-12-02 18:00:00
published:  true
---

Earlier this year I presented my latest project - Hoplon - at the London
Elixir meetup. I'm thinking of putting some more work into it over Christmas,
so I figured I might gather the materials about it in one place:

[Hoplon](https://github.com/nietaki/hoplon) is an Elixir developer tool that helps 
you validate your dependencies contain no
hidden malicious code. Motivated by horror stories from the JavaScript community
such as [this hypothetical one](https://hackernoon.com/im-harvesting-credit-card-numbers-and-passwords-from-your-site-here-s-how-9a8cb347c5b5)
and [this very real one](https://github.com/dominictarr/event-stream/issues/116).

You can see the details and a live demo in the 
[recorded talk](https://skillsmatter.com/skillscasts/11856-i-m-stealing-api-keys-from-your-site-here-s-how){:target="_blank"}.

Here's [the slides](https://slides.com/nietaki/i-m-stealing-api-keys-from-your-site-here-s-how), 
with all their happy colourful diagrams:

<iframe src="//slides.com/nietaki/i-m-stealing-api-keys-from-your-site-here-s-how/embed?style=light" width="576" height="420" scrolling="no" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>


Hoplon as it is right now is pretty much a proof of concept, but I'm thinking of making
it a bit more production ready and adding some advanced herd-immunity type features.
Keep your fingers crossed!
