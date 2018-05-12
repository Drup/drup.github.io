---
title: "Ann: Regenerate"
layout: post
author: "Drup"
tags: OCaml Regex announcement
---

I'm happy to announce the release of [Regenerate][regenerate], 
a library, tool and [website][web] to generate test cases for regular expression engines.
<!--more-->

Let us say you just came up with a fancy new way of matching regular
expressions. You implement it and now comes the time to test it.
Writing test cases by hand is annoying and getting proper coverage is difficult,
so you want to automatically generate them!
Unfortunately, if you generate strings randomly, 
you will need an external oracle to tell you if the string
matches or not. And even then, generating strings that match a given
regular expression can be challenging (consider `a*`, the probability to generate
a string that match it tends to 0).

Instead, you can now use [Regenerate][regenerate] to directly generate matching
and non-matching strings for any regular expression!
Regenerate takes any [*regular* expression][regex] and 
generates strings that match it.
It handles most posix extended regular expressions along with
complement (`~a`) and intersection (`a&b`).
Since it handles complement, it can also generate strings that
*don't* match a given regular expression.

For example, here
is a set of samples that are matched by the regex `(1(01*0)*1|0)*`
(the multiple of 3 written in binary)
on the alphabet composed of `0` and `1`:

```
% regenerate gen -s5 --alphabet "01" "(1(01*0)*1|0)*"
01001
11000
000011
001001
001100
011000
111111
0000110
0011000
0101101
```

And here are some strings that *don't* match it:

```
% regenerate gen -s5 --alphabet "01" "~((1(01*0)*1|0)*)"
10
0010
0111
1110
01011
10001
10011
10111
0001000
```

We spent a lot of time optimizing Regenerate, so it's pretty fast, around 10⁶ words
per seconds, depending on the regular expression.
On the language above, it looks like this:

![Generating lot's of samples]({{"/assets/img/regenerate.gif" | absolute_url }})



An [online version][web] of the tool is also available for you to play around.

## Testing 

Regenerate's main purpose is testing. Hence it also provides an [OCaml library][lib] to create
test harnesses easily through [qcheck][] generators. 
[qcheck][] is an OCaml library for property testing in the style of QuickCheck. 
An example test harness for the [ocaml-re](https://github.com/ocaml/ocaml-re) library can be found [here](test/re/test_re.ml).

The main part of the test harness, shown below, creates a test generator
given an alphabet (here `abc`), a data-structure that describe words
on which the regex engine operates and a few other parameters.
This means that it can operates on utf8 or ascii strings, ropes, etc.

```ocaml
let test =
  let alphabet = ['a'; 'b'; 'c'] in
  let module Word = Regenerate.Word.String in
  let module Stream = Segments.ThunkList(Word) in
  let generator =
    Regenerate.arbitrary
      (module Word) (* Datastructure for words *)
      (module Stream) (* Datastructure for streams of words *)
      ~compl:false (* Should we generate complement operations? *)
      ~pp:Fmt.char (* Printer for characters *)
      ~samples:100 (* Average number of sampes for each regular expression *)
      alphabet (* Alphabet *)
  in
  QCheck.Test.make generator check
```

Regenerate then generates triples composed of a random regular expressions
and positive and negative samples.
Here is a *very* abridged extract of a test run.

```
Regex: ([^c]b(b|b){1,5}){2,5}
Pos: 
Neg: a, ac, ba, bc, cb, aba, abb, bba, cba, abab, abbb, abcc, baab, cbcb,
  ccba, aaabc, aaacc, aaccc, abaca, abcab, abcba, acaaa, acbac, acbbb, babaa,
  abcbaa, abcccc, acabab, acabbc, acbaab, accaaa, acccbb, acccbc, baaaab,

Regex: ([^bc]{0})(ab{0,1})(b|[^bac]*)(a&[^c])
Pos: ba, bbbbbbba, abbbbbbbbbbba, bbbbbbbbbbbbbbbbba, bbbbbbbbbbbbbbbbbbbbba,
  bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbba,
  abbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbba,
  abbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbba,
  bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbba,
  abbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbba
Neg: bb, aaaa, abbc, acbb, baba, bbbc, bbcb, cbbc, ccbc, ccca, cccc, aabac,
  ababc, abacc, abbab, abbca, abcca, acaaa, acbbc, accac, babaa, babba,
  aaaccb, aabcbb, aacabb, aacbaa, aaccca, aacccb, abaaaa, ababac, ababbc,
  ababca, abacbb, abbbcb, abbccb, abcabb, abcacb, abcbaa, abcbac, abcbcb

Regex: ((b&a)|ba+)|a|[cb]|(ac{1,2})
Pos: bababababa
Neg: aa, ab, cba, cbc, aacb, abbb, acaa, accc, baab, babc, cabc, ccaa, ccbb,
  aaaab, aabab, aabca, aacab, aacbb, aaccc, abcbc, acccb, baabc, babcc,
  baccc, bbaba, bbabc, bbaca, bbcbc, bccca, caacb, caacc, cabbc, cabca,
  cacaa, cbaaa, cbaba, cbcba, cbcca, aaaacb, aaacac, aabcba, aabccb, aacbab

Regex: (bb&[cb]c)|(cb{2,4})([acb]{0,5})
Pos: cbcbc, cbcbcb, cbcbaac, cbcbbab, cbcbbba, cbcbaabc, cbcbabaa, cbcbabbb,
  cbcbbacb, cbcbbbac, cbcbbcaa, cbcbcaba
Neg: aaa, baa

Regex: c[^acb](a&a)|((b[^bc]+)b*)
Pos: bababab, bababababbab, babbababbabab, babbabbababbab, babbababbababab,
  bababababababababab, bababababbababbabab, babbababababababbab,
  babbababababbababab, babbabababbabababab, babababababbabbabbab
Neg: aaa, aca, acc, bbb, cab, cac, cca, baaa, bbca, bcab, bcbc, cbcb, aacaa,
  aacbc, abbac, abbca, acbcc, baaba, baacb, babcb, bbacc, bbbbb, bbcbc
```

As you can see, the randomly generated regular expression can be slightly nightmarish, but Regenerate will also generate simpler regular expressions, as well
as longer words.

# Conclusion

In order to quickly generate samples for regular expressions with complement,
we came up with
some new algorithms that are described [in this paper][paper]. 
Feel free to implement
them in your favorite language and test your favorite regular expression engine!
If you find new bugs, please tell us!

There are still lot's of things to add and tweak. In particular, we want to
integrate with fuzzing tools such as afl-fuzz to directly select test cases
that will torture implementations the most. We also would like to extend 
our technique to implement more operators, such as boundaries/lookaround.

[regenerate]: https://github.com/Drup/regenerate
[web]: https://drup.github.io/regenerate/
[regex]: https://en.wikipedia.org/wiki/Regular_expression
[qcheck]: https://github.com/c-cube/qcheck/
[lib]: https://drup.github.io/regenerate/doc/dev/regenerate/Regenerate/index.html
[paper]: https://hal.archives-ouvertes.fr/hal-01788827
