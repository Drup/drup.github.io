---
title: "My opam workflow"
layout: post
author: "Drup"
hide: yes
tags: OCaml opam
---

Several people expressed confusions about opam switches, and in particular how to use them effectively.
This is a presentation of my workflow which, while non standard, is very well supported by opam.
<!--more-->

First, a small disclaimer:
- This is not an opam tutorial. I will assume you are already familiar with opam, in particular pin and switches.
- I work both on libraries and on the compiler, so I need a different kind of flexibility than most OCaml users.
- I tend to fall in the category "poweruser" (especially about opam).
- I'm going to use the quite new (and awesome) opam 2.0.

Finally, these advises go together. It makes little sense to apply them individually.

# Don't change the switch globally

You should not change the global switch, unless exceptional circonstances (zombi apocalypse, meteor strikes, new ocaml release).
Instead, you should rely on local switches. You can switch locally using the following command:

```
eval $(opam config env --switch my_switch)
```

This will change your switch *only for the current terminal*.
This means that you can compile and edit from that terminal and enjoy everything installed on this terminal without disturbing the global setup. This also means that you can work in different switches simultaneously, in different terminals.

If you did your editor setup with [opam user-setup][], which I highly recommend,
you can also switch locally inside your editor (`M-x opam-update-env` in emacs).

[opam user-setup]: https://github.com/AltGr/opam-user-setup

# Use a tooling switch

With [opam user-setup][], if the tooling is installed in one switch, it will be available everywhere!

This means that you can use switch with all the tooling (tuareg, ocp-indent, ...) and not worry about it anymore. I usually define this switch as my "main" one (see point above).

Unfortunately, you still need merlin in each switches, since it's tied to the OCaml compiler.

# Create alias switches liberally

Alias switches are a way to name a switches differently than the name of the compiler:

```
opam switch 4.03.0 -A foo
```

I work on several related but distinct projects: ocsigen, mirage, the ocaml compiler, various independent things.
I also ocasionally answer questions on StackOverflow and IRC, so I also need a "vanilla" OCaml experience.

Alias switches allow me to have one compiler per project.

Each switch can have an arbitrary compiler version and their pins are independent: the local hacked half broken version of `js_of_ocaml` that is pinned in the `ocsigen` switch will not affect building the `mirage-www` project in the `mirage` switch. Even if my `ocsigen` switch is in complete turmoil because I broke `tyxml`, I can still answer beginner questions with the stable eliom installation in the regular switch.
In opam 2.0, repositories are also local to switches, so
I can enable the [mirage-dev][] opam remote in my mirage switch
without borking my vanilla switch.

Here is the list of switches I have currently installed:

```
# switch       compiler                     description
4.00.1         ocaml-base-compiler.4.00.1   Official 4.00.1 release
4.02.3         ocaml-base-compiler.4.02.3   Official 4.02.3 release
4.03.0         ocaml-base-compiler.4.03.0   Official 4.03.0 release
4.03.0+eliom
4.04.0+trunk   ocaml-variants.4.04.0+trunk  latest trunk snapshot
coq            ocaml-base-compiler.4.02.3   Official 4.02.3 release
eliom
flambda
flambda-names
mirage         ocaml-base-compiler.4.03.0   Official 4.03.0 release
ocaml-system   ocaml-system.4.02.3          The OCaml compiler (system version, from outside of opam)
```
