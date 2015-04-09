---
title: First steps in mirage
layout: post
author: "Drup"
tags: OCaml Mirage
---

This was long overdue, after
[a lot](https://github.com/ocsigen/ocsigenserver/pull/63) of
[ocsigen-cohttp hacking](https://github.com/ocsigen/ocsigenserver/pull/64) ([duh](http://i.imgur.com/YLI3fuc.png)), I finally started trying mirage! The first step was obviously to ignore the tutorials and went investigating which little ARM to buy...[^1] Ok, stop browsing ARM stores and actually starting the tutorial now.

![It's so cute](https://hackspark.fr/media/catalog/product/cache/1/image/650x/6244465d93851bfb41b9f76e241a8d6d/c/u/cubieboard_top.jpg)


The start of the tutorial went rather smoothly. The automagical `mirage configure` command is very magical and I wanted to have a better grasp of how it worked, but I postponed that. I skipped very quickly over the block part (far too low level for me~~) and played with the kv store a bit.

At this point I wondered what was exactly in the [Mirage][Mirage] module and how the whole functor instanciation business was working, so I went looking.
It was very interesting, it's funny to see the relation with all the other ocamllabs libraries I know (for example, ctypes and the [functor combinators](https://github.com/mirage/mirage/blob/master/lib/mirage.mli#L28-L40) are basically the same). The [Mirage][Mirage] module first lists the available `typ` and `implementation` corresponding to module signatures ([example for clock](https://github.com/mirage/mirage/blob/master/lib/mirage.mli#L60-L69)).

[Mirage]:https://github.com/mirage/mirage/blob/master/lib/mirage.mli

Then, later it defines the [IMPL](https://github.com/mirage/mirage/blob/master/lib/mirage.mli#L433-L456) to query information from an `'a impl`. Ok, so each `impl` contains the opam and ocamlfind packages along with the module name it represents and basically everything you need to configuration and pre-build.

At this point I fired utop and asked `Mirage.(Impl.packages default_console)` and got the answer `["mirage-console"; "mirage-unix"]`. Nice. Interactive pre-build.

Then it defines the [CONFIGURABLE](https://github.com/mirage/mirage/blob/master/lib/mirage.mli#L491-L523) interface, and this little beauty:

{% highlight ocaml %}
val implementation:
  'a -> 'b -> (module CONFIGURABLE with type t = 'b) -> 'a impl
(** Extend the library with an external configuration. *)
{% endhighlight %}
which explains pretty much all the underlying structure and also the automagical `mirage configure`! It is used like that:

{% highlight ocaml %}
type foo = FOO
let foo param : foo impl =
  implementation FOO param (module Foo)
{% endhighlight %}
and tada, you can compose stuffs. It's very elegant, I'm rather fond of it.

-----

Ok, let's get back to actual mirage programs, instead of mirage configuration, I still have to use the network. Apparently, `tuntap` is not working at all
on my box (regular archlinux installation). I circled a long time around the issue, and ended up using sockets instead by frustration. I played a bit with the stackv4 example and modified it until it answered everything I was saying through a socket. It was not difficult.

![Such application](https://i.imgur.com/hRXTGzz.png)

The next step of the tutorial was how to serve a website with mirage, which I was not so interested (I wanted a pause with all this http handling). The one after was about CI and deployment, which is I was already accustomed with. Apparently, there are no other entry-level tutorials, which is a bit unsatisfying. I suppose most people are interested in how to build webservers, but I wanted to do domotics and pilot my music from my kitchen with an ARM little thingy (my avatar is an octopus, yes, I know).

So before remote play, let's do local play: How do I talk to a speaker plugged by jack in mirage ? Apparently, it's not implemented (Or I didn't find where). The xen audio stack ... seems complicated, so I put that aside. Emitting a beep on the internal "speaker" is also not available in mirage, but I can understand that (it's literally criminal to produce music that way). I think a tutorial about how to create a new connector from scratch would be a good thing to have. Implementing how to emit on a jack seem a bit complicated just for now, so I went looking for something else.

----

Apparently, someone started [an mpd-client library](https://github.com/johnelse/ocaml-mpd-client), the immediate following question is: how do I talk to someone with mirage. I know how to let people talk to me (tanks to the stackv4 tutorial), but not the other way around. Ok, let's open the [implemented module signatures](https://github.com/mirage/mirage/blob/master/types/V1.mli).
All those signatures are nicely arranged by inclusion (being the maintainer of tyxml, I'm not confused by mirage's  [functor-stack-oriented-development](https://github.com/ocsigen/tyxml/blob/master/lib/html5_sigs.mli#L1193-L1218)).
I learned two things:
- I want a module answering the TCP signature, which is included in the stack
- The concept of FLOW (which is basically something that sends data to someone) would be useful to implement talking to speakers.

I then looked at the http-fetch example in the mirage-skeleton repository and I get slightly lost looking at what a resolver is and what conduit is and what is the relationship with cohttp and why do I even need this thing but then, I started to wonder how functorized typical mirage applications are.

![I have questions](http://i.imgur.com/ja3S5UF.gif)

You see, in ocsigen, all the user code is dynlinked by the server. This is done for two reasons:

- Bypass configuration for ocsipersist and all the code that uses it, which is almost all eliom, in particular the service library. Ocsipersist can be implemented with sqlite and dbm, and functorizing the 44k lines of eliom was deemed inconvenient at the time. Mirage people seemed to have been less shy about it.
- Delay and control side effect evaluation, which is very important for services and javascript code. This could be equally done with functors.

It is possible to statically link ocsigen applications by ... functorizing everything (sic). In the current state, it's possible, if slightly messy.

It seem to me that mirage's solution, using the `Mirage` module and the little dsl is more flexible and offer better control. The question is : how heavy (and annoying) the usage of functor is. In the little examples I saw, it was still manageable, but everything is manageable in a 50-line program (ok, [there](https://en.wikipedia.org/wiki/Befunge) [are](https://en.wikipedia.org/wiki/Shakespeare_%28programming_language%29) [exceptions](https://en.wikipedia.org/wiki/Esoteric_programming_language#Piet)).
In the case of big ocsigen applications, I'm pretty convinced than most of the code could be out of functors (only services and database access would be inside functors). Making FRP works in this context seems tricky, but it might actually help in the end, by decoupling the various parts enough to be functorized independently.

I also wonder how much the [big-functor](http://www.ocamlpro.com/blog/2011/08/10/ocaml-pack-functors.html) proposal by ocamlpro would help for this.

I'll wait for an answer by some more experienced mirage user, but in the meantime, it's 5 in the morning, I have 10 instances of emacs full of module signatures open and I should probably sleep.


[^1]: Apprently, a cubieboard.
