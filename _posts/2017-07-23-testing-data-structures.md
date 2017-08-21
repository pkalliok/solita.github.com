---
layout: post
title: How much does it take to properly test a data structure?
author: pkalliok
tags:
- testing
- invariant
- generative testing
- clojure
- spec
excerpt: >
 TODO
---

Testing is hard, but some kinds of code are even harder to test than
others.  A well-known one is user-interface code: user interaction is
tricky to automate, so UI tests are often very much work to write and
tend to break in various random ways.  But data structures pose a
totally different kind of testing problem.  They have many properties
that require extensive, flawless, and many-aspect testing:

- Data structures have very many uses and it is almost impossible to
  predict all the use cases for a data structure.
- Data structures have often complicated contracts, not only giving
  input-output guarantees but also (amortised) performance guarantees,
  sharing guarantees, and interdependent contracts between different API
  calls.
- Many data structures have (infinitely) many internal representations
  for the same external state.  Some of these representations may be
  only reachable through a very specific path of API calls.
- Even extensively enumerating the external states of a data structure
  easily leads to a combinatorial explosion.

## Examples of data structure bugs

The same that goes for "pure" data structure libraries also holds for
other libraries that internally deal with complicated data structures.
For instance, while developing a [fast
implementation](http://cs.angelo.edu/~mmotl/4301/rx.pdf) (Rx) of POSIX
regular expressions, Tom Lord also developed a relatively extensive test
harness for his library.  Upon running the same tests against other
POSIX RE implementations, he found out that _every one_ would fail some
tests.  This is pretty noteworthy as the tests didn't even have
performance tests.

Sometimes bugs are found in very well matured data structure
implementations.  In the Euroclojure conference, Michał Marczyk just
held a [speech](http://2017.euroclojure.org/spec-loves-data-structures/)
where he mentioned one bug in Clojure's data.avl that was only
discovered in a very peculiar case of usage, and two more that were
caught after that by his explicit effort to bring the test harness
beyond unit testing.  One of them was a bug that leaves the AVL trees
unbalanced after a specific kind of split.  It is a bug that only
manifests as a performance problem, after a specific usage pattern,
_after_ the call where the actual bug is triggered.  It's very hard to
spot, but even then it might affect very many users - no one knows, as
the actual usage patterns are buried in the runtime state of processes.

I'm very much indebted to Michał for providing inspiration for this blog
post.  The ideas here are based on his, but some are also added from my
experience about the difficulties of testing a FSA (finite-state
automaton) library written in Scheme.

## Generative testing

One of the problems with data structures is that their APIs have too
many cases to test.  For instance, even only taking into account
different cases of root nodes and their direct descendants for red-black
trees, a merge of two trees has 20 x 20 = 400 different cases.  Some
kinds of trees are rotated in a way that depends on the whole tree
structure under the nodes that are being rotated.  For three nodes,
there are about 2<sup>9</sup>=512 different unlabeled directed graphs.
And an FSA of three nodes can represent somewhere on the order of 512 x
30 ([partition
number](http://mathworld.wolfram.com/PartitionFunctionP.html) of 9)
different grammars.

In such a setting, it is not sufficient to find examples for unit tests
manually.  It doesn't hurt to do such tests, but they may very well
reflect the developer's (possibly faulty) intuition about how the data
structure will be used and which are the situations most likely to break
it.  

Generative testing is the practice of generating test cases.  Basically,
it means the generation for data to test on.  For API calls with many
arguments, you need to generate for each of the arguments.  For
instance, to test a regular expression match, you need to generate both
the regular expressions and the strings to test them against.

One problem with data generation is that the generation itself may be
faulty.  It might skip some alternatives altogether which will then
cause some cases to be left forever untested.  To help with this,
generation can be based on the type definitions of inputs (so that all
allowed values will be covered).  Clojure's [core.spec]() provides
generators in this vein.

