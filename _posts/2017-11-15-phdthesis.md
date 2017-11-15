---
title: "PhD Thesis: Tierless Web programming in ML"
layout: post
author: "Drup"
tags: OCaml Eliom Ocsigen PhD announcement
---

I'm happy to announce that I successfully defended my PhD Thesis!
<!--more-->

My thesis, titled "Tierless Web programming in ML", describes the
formalization and the implementation of Eliom,
an extension of OCaml for tierless Web programming,
as part of the [ocsigen project](https://ocsigen.org).
You can find all the details [here](https://www.irif.fr/~gradanne/phdthesis.html),
along with [the thesis](https://www.irif.fr/~gradanne/papers/phdthesis.pdf)
and [the slides](https://www.irif.fr/~gradanne/papers/talk_phdthesis.pdf). The
abstract can be found below.

If you have any questions, feel free to ask them by email, on
[discuss](https://discuss.ocaml.org/t/tierless-web-programming-in-ml/1125) or
on [reddit](https://dd.reddit.com/r/ocaml/comments/7d5uec/tierless_web_programming_in_ml/).


### Abstract

Eliom is a dialect of OCaml for Web programming in which server and client
pieces of code can be mixed in the same file using syntactic annotations. This allows to
build a whole application as a single distributed program, in which it is possible to define
in a composable way reusable widgets with both server and client behaviors.

Eliom is type-safe, as it ensures that communications are well-behaved through novel
language constructs that match the specificity of Web programming. Eliom is also
efficient, it provides static slicing which separates client and server parts at compile time
and avoids back-and-forth communications between the client and the server. Finally,
Eliom supports modularity and encapsulation thanks to an extension of the OCaml
module system featuring tierless annotations that specify whether some definitions should
be on the server, on the client, or both.

This thesis presents the design, the formalization and the implementation of the Eliom
language.
