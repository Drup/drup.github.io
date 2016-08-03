---
title: "Typing Tricks: Diff lists"
layout: post
author: "Drup"
tags: OCaml gadt typing-tricks existential-overflow-crisis
---

The diff list trick is a way to compute with lists in types. It allows to create heterogeneous lists which is very useful, in particular in the `Format` module from the standard library.
<!--more-->

There are mostly two approaches when we want to put things of different types in a list: hide all the types or manage (somehow) to show all the types.

To hide all the types, nothing easier, we just use an existential:
{% highlight ocaml %}
type ex_list =
  | Nil : ex_list
  | Cons : 'a * ex_list -> ex_list (** 'a is hidden! *)
{% endhighlight %}

But if we try to write the `get` function that returns the first element (in an option):
{% highlight ocaml %}
let get = function
  | Nil -> None
  | Cons (x,_) -> Some x
{% endhighlight %}

The typechecker answers this:
{% highlight ocaml %}
Error: This expression has type $Cons_'a
       but an expression was expected of type 'a
       The type constructor $Cons_'a would escape its scope
{% endhighlight %}

In typechecker language, this means "You are trying to export typing information that you don't have", the type information is trying to escape! We have no information about the type of what's inside the list at all: it's always `ex_list`, the `'a` is hidden.

This way of making heterogeneous lists will works if either:
- We never need to get out elements.
- We have more information about what can be put inside (using type witnesses, for example).

If we want to actually export the type of the elements, we need another solution.

# Introducing: Difference lists

## A logical prelude

Difference lists comes from logic programming and, in particular, [Prolog][prolog]. It is, at its core, a way to *compute with unification*.
A prolog tutorial is available [here](http://homepages.inf.ed.ac.uk/pbrna/prologbook/node180.html). An OCaml implementation is available in the [dlist][] package.

[dlist]: https://github.com/BYVoid/Dlist

[prolog]: https://en.wikipedia.org/wiki/Prolog

The idea is the following, if we consider the function `(fun x -> 1 :: 2 :: x)`, concatenating at the end of the list is very cheap (It's O(1)). If we keep a handle that points to the end of the list, we can substitute it by whatever we want. In Prolog, the handle is a unification variable. In the [dlist][] implementation, it's a function parameter.

I encourage people to read the the Prolog tutorial above, Prolog is a very fun language. It uses a quite different paradigm than OCaml, which is always very enlightening.

## Back to OCaml

Unification is available at the value level in prolog, but it's also available for OCaml types! This is how polymorphic function become specialized:

{% highlight ocaml %}
# List.map ;;
- : ('a -> 'b) -> 'a list -> 'b list = <fun>
# (fun x -> x + 1) ;;
- : int -> int = <fun>
# List.map (fun x -> x + 1) ;;
- : int list -> int list = <fun>
{% endhighlight %}

During type checking, the type checker unifies the unification variable `'a` with `int` (as specified by the [Hidley-milner][HM] family of type systems). We will now use this to do simple computations.

[HM]: https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system

Let us create our list type. It has two type variables, one for the content of the list and one for the unification variable at the end. In the following, I will note `'ty` the **ty**pe of the content and `'v` the unification **v**ariable.

- An empty list is a list where there is no content: `'ty = 'v`
- A cons is a list where we added one element to the type.

{% highlight ocaml %}
type ('ty,'v) t =
  | Nil : ('v, 'v) t
  | Cons : 'a * ('ty, 'v) t -> ('a -> 'ty, 'v) t
{% endhighlight %}

If we consider the type of the cons function, we can see it's equivalent to adding one to the type. We count with arrows.

{% highlight ocaml %}
# let plus1 l = Cons ((),l)
val plus1 : ('ty, 'v) t -> (unit -> 'ty, 'v) t = <fun>

# let one x = Cons (x, Nil)
val one : 'a -> ('a -> 'v, 'v) t = <fun>
{% endhighlight %}

If we consider the return type of `one`, `('a -> 'v, 'v) t` we can wander in the territory of terrible arithmetic and do the following

    'ty = 'a -> 'v
    'ty - 'v = 'a

Or, phrased in English: "If we remove `'v` from `'ty`, we get the content". Of course, this arithmetic doesn't make the slightest sense[^1], but it can give a good intuition.

[^1]: Even if we consider [the algebra of types](https://chris-taylor.github.io/blog/2013/02/10/the-algebra-of-algebraic-data-types/)

{% highlight ocaml %}
# let l1 = Cons (1, Cons ("bla", Nil)) ;;
val l1 : (int -> string -> 'v1, 'v1) t
# let l2 = Cons ((), Cons (2., Nil)) ;;
val l2 : (unit -> float -> 'v2, 'v2) t
{% endhighlight %}

What should be the type of `append l1 l2` ?

{% highlight ocaml %}
# let l3 = Cons (1, Cons ("bla", Cons ((), Cons(2.,Nil))))
val l3 : (int -> bytes -> unit -> float -> 'v3, 'v3) t
{% endhighlight %}

We can take the type `'ty1 = int -> string -> 'v1` and replace `'v1` by `'ty2 = unit -> float -> 'v2`. It would also gives us `'v2 = 'v3`.

We can deduce the type of `append`

{% highlight ocaml %}
val append : ('ty1, 'ty2) t -> ('ty2, 'v) t -> ('ty1, 'v) t
{% endhighlight %}

If we try to do terrible arithmetic again:

      'ty1 - 'ty2
    + 'ty2 - 'v
      -----------
      'ty1 - 'v

Writing append is now completely straightforward:
{% highlight ocaml %}
let rec append
  : type ty1 ty2 v.
    (ty1, ty2) t ->
    (ty2, v  ) t ->
    (ty1, v  ) t
  = fun l1 l2 -> match l1 with
  | Nil -> l2
  | Cons (h, t) -> Cons (h, append t l2)
{% endhighlight %}

We can write many more functions, but writing them can be quite tricky at the time. The complete code is available in [this gist][diffgist] or [this toplevel][difftop].

[diffgist]: https://gist.github.com/Drup/f8e1564c41374bf38e2cb74dfb0c857a#file-difflist-ml
[difftop]: https://is.gd/JoZEA1

# Applications

I promised in the introduction that all this was actually useful and not just terrible type spaghetti. We will now build a small format-like data type equipped with a `printf`-like function.

What is, fundamentally, a format ?
It can be composed of:
- Constants that are present in the format.
- Holes which we will fill during `printf`.
- The end of a format.

Our goal is to track in the type a list of all the holes. For example, what is the type of `"%s | %s"` ? `"%s"` is a hole and `"|"` is a constant.

{% highlight ocaml %}
# ("%s | %s" : _ format) ;;
- : (string -> string -> 'a, 'b, 'a) format
{% endhighlight %}

This is very similar to our diff list, where we list the holes in the type. Here is a way to define such type:

{% highlight ocaml %}
type ('ty,'v) t =
  | End : ('v,'v) t
  | Constant : string * ('ty,'v) t -> ('ty,'v) t
  | Hole : ('ty, 'v) t -> (string -> 'ty, 'v) t ;;
{% endhighlight %}

There are two things of note here:
- `Constant` doesn't change the list, since it doesn't define a hole.
- `Hole` doesn't have any content, it only adds to the type. It can only be filled by a string.

{% highlight ocaml %}
# let myfmt = Hole (Constant (" | ", Hole End)) ;;
val myfmt : (string -> string -> 'v, 'v) format
{% endhighlight %}

This is the type we wanted! Now that we have a format type, we need to build up a `printf` function. What should be the type of `printf myfmt`? It has two holes to fill by strings and it should return a string, so the type should be `string -> string -> string`. Since the type of `myfmt` is `(string -> string -> 'v,`v) t`, we can use unification again.

We would have the following type:
{% highlight ocaml %}
val printf : ('ty, string) t -> 'ty
{% endhighlight %}

Let's check that it works, if we give `myfmt` to `printf`, `'ty` unifies with `string -> string -> 'v`, `'v` unifies with `string`, so `printf myfmt` is of type `string -> string -> tring`. Fabulous.

In order to implement `printf`, we first need to implement the version by continuation which takes as argument a function consuming the resulting string. `kfprintf f myfmt ...` is morally equivalent to `f (printf myfmt ...)`. However, since we can't manipulate a variable number of arguments in OCaml, we can't write the dots `...`. The solution is to always place the variable number of arguments at the end and to use continuations.

{% highlight ocaml %}
val kprintf : (string -> 'v) -> ('ty, 'v) format -> 'ty
{% endhighlight %}

Given this function, the implementation of `printf` is very simple:

{% highlight ocaml %}
let printf fmt = kprintf (fun x -> x) fmt
{% endhighlight %}

The implementation of `kprintf` is a bit more involved. We have to fold through the format and return a closure consuming all the holes argument. Here is the complete definition.

{% highlight ocaml %}
let rec kprintf
  : type ty v. (string -> v) -> (ty,v) t -> ty
  = fun k -> function
    | End -> k ""
    | Constant (const, fmt) ->
      kprintf (fun str -> k @@ const ^ str) fmt
    | Hole fmt ->
      let f s = kprintf (fun str -> k @@ s ^ str) fmt
      in f
{% endhighlight %}

And that's it! The complete code for miniformat is available in [this gist][miniformat] or in [this toplevel][miniformat.js].

[miniformat]: https://gist.github.com/Drup/f8e1564c41374bf38e2cb74dfb0c857a#file-miniformat-ml
[miniformat.js]: https://is.gd/Zy5tk2

# Because we can never have nice things

People that are very familiar with type tricks might have noticed that something is fishy[^3]. The issue here is that our type is not covariant in any of it's type variables. There are various reasons for that, but mostly, [mixing subtyping and GADT is tricky][GADTvariance]. This means we don't benefit from the [relaxed value restriction][RWOvalrestr].

[^3]: Or, as one of my teacher used to say, there is a whale under the gravel.
[GADTvariance]: http://arxiv.org/abs/1301.2903
[RWOvalrestr]: https://realworldocaml.org/v1/en/html/imperative-programming-1.html#the-value-restriction

In practice, we can't use our lists in a functional manner when using append. Here is an example using the lists defined at the beginning:

{% highlight ocaml %}
let l = append l1 l2
let l' = append l l1
let l'' = append l l2 (* This creates a type error. *)
{% endhighlight %}

This is a severe restriction to the usability of difference lists. `Format` is equally affected, but the issue is less striking due to the custom format syntax available in OCaml. We can hope this restriction is lifted one day, maybe with a notion of pure functions?

# Conclusion

We have learned how to (ab)use the type system to create heterogeneous lists that count the number of their elements, and how to use it to create a toy implementation of format. As the other Camlien Gabriel would put it, We now have to learn how not to use this.

Fortunately, code that manipulate difference lists is quite annoying to write, offering a natural deterrent to apprentice type magicians. In the case this could actually be useful (such as my own unreleased [Furl][] library), I would encourage to hide the datatypes and provide combinators, as long as examples and documentations. This is a case where types really do not help understand the API.

[Furl]: https://github.com/drup/furl

# Afterwords

## Is this really how format works?

Yes, it is, since the awesome work by Benoit Vaugon in OCaml 4.02. As a proof, let's build the format by hand, without using the fancy syntax.

```ocaml
let myformat = CamlinternalFormatBasics.(
  Format
   (String (No_padding,
     String_literal (" | ",
       String (No_padding, End_of_format)))
   ,"%s | %s"))
```

This is very similar to our "miniformat" example, except more complicated[^2]. `String_literal` is `Constant` and `String` is `Hole`. It gives us the type `(string -> string -> 'a, 'b, 'a) format`, which is also what we would expect.

Note the very scary `CamlinternalFormatBasics`, showing that you should probably not use that in your programs.

[^2]: Because everything about format is more complicated.

{% highlight ocaml %}
# Format.printf myformat "foo" "bar" ;;
foo | bar
{% endhighlight %}

## A more convenient syntax for diff lists

With the last version of OCaml, we can rebind `[]` and `::`, making this much better!

```ocaml
module M = struct

  type ('ty,'v) t =
    | [] : ('v, 'v) t
    | (::) : 'a * ('ty, 'v) t -> ('a -> 'ty, 'v) t

  let cons x l = x :: l
  let one x = [ x ]

  let rec append
    : type a b c.
      (a, b) t ->
      (b, c) t ->
      (a, c) t
    = fun l1 l2 -> match l1 with
      | [] -> l2
      | h :: t -> h :: append t l2
end

let l1 = M.[ 1 ; "bla" ]
let l2 = M.[ () ; 2. ]
let l3 = M.[ 1 ; "bla" ; () ; 2. ]
```

Isn't that fabulous? :)
