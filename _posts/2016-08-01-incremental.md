---
layout: post
title: "Incremental Compilation"
author: Michael Woerister
description: "Incremental compilation for exponential joy and happiness."
---

<!--
* Compile times are a known issue
* Incremental compilation will improve compile times during common
  edit-compile-debug workflow by exploiting "temporal coherence" of code base
* To get better understanding of what IC can do for you, rest of this post will
  answer with following questions:
  - What does "incremental" mean more formally?
  - How does that definition translate to the compiler?
-->

I remember when, during the 1.0 anniversary presentation at the
[Bay Area Meetup][meetup], Aaron Turon talked about Dropbox having been
pretty happy with using Rust in production there. "The core
team has been in touch with them regularly", he said, "asking them, you know,
what do you need? And their answer is always: faster compiles ..." To the
learned Rust user it's no surprise that this solicited a knowing chuckle from
the audience. Well, improving compilation speed has been a major development
focus after Rust reached 1.0 -- even though up to this point much of the work
towards this goal has gone into laying [architectural foundations][mir] within
the compiler and we are only starting to see actual results.

One the projects that is building on these new foundations is **incremental
compilation**

<!--
One of the
projects targeted at improving compilation speed is *incremental compilation*,
which has been worked since the end of last year and which, as of recently, has
progressed to a stage where we are beginning to get comfortable with letting
people get their hands on it.
-->

Why Incremental Compilation in the first place?
===============================================
Much of a programmer's time is spent in an edit-compile-debug workflow:

- you make a **small change** (often in a single module or even function),
- you let the compiler **translate the code into a binary**, and finally
- you **run the program** or a bunch of unit tests in order to see results of
  the change.

After that it's back to step one, making the next small change informed
by the knowledge gained in the previous iteration.
In order for this essential feedback loop at the core of our daily work to be
smooth and effective, the time being stalled while waiting for the compiler
to produce an executable program must be as short as possible, ideally short
enough as not to warrant a time-consuming and stress-inducing mental context
switch: You want to be able to keep working, stay in the zone. After all,
there is only so much [regressive fun][compiling] to be had while `rustc`
bootstraps.

Incremental compilation is a way of exploiting the temporal coherence a code
base exhibits during the regular programming workflow: Many, if not most,
of the changes done in between two compilation sessions only have local impact
on the machine code in the output binary, while the rest of the program,
same as at the source level, will end up exactly the same, bit for bit.
Incremental compilation aims at retaining as much of those unchanged
parts while redoing only that amount of work that actually *has* to be redone.
The next few paragraphs will give a little tour of the basic strategy we are
pursuing in order to implement incremental compilation in the Rust compiler,
of the kind of compile time speedups and runtime trade-offs that are to be
expected, and finally, the kind of progress the actual implementation has made
so far.


So what does "incremental" mean exactly?
========================================
So far, we have not defined what "incremental" means, really.

<!--
* Definition: Re-use intermediate results if they are still up-to-date
  (entails being able to know if something is still up-to-date)
  => reference some formal definition maybe (e.g. Ramalingam, Reps, Acar?)

  TODO!
-->

> The abstract problem of incremental computation can be phrased as follows: The
goal is to compute a function *f* on the user's "input" data *x* --- where *x*
is often some data structure, such as a tree, graph, or matrix --- and to keep
the output *f(x)* updated as the input undergoes changes.




<!-- [* Example from algebra: computing a * b + c incrementally] -->

Let's take a look at a simple example from algebra to make things more
concrete. Let's see what it means to evaluate an expression of the form
`a + b × c` incrementally. This will involve evaluating the expression once
with one set of values for `a`, `b`, and `c` and then evaluating it a second
time with a different value for `a`. For the first time around, `a` will be
`1`, `b` will be `2`, and `c` will be `3`:

<!--
       a+b×c=7
       /    \
      /     b×c=6
     /     /   \
    a=1   b=2  c=3
-->
![Initial Computation of a + b × c][algebra-initial]{:class="center"}

Assume that we "saved" the intermediate results at each step, that is, we
remember somewhere `b × c` is `6` and `a + b × c` is `7`. Now, in the second
round, we want to know what `a + b × c` is if we changed the value of `a` to
`4`. When we recompute the value of the expression, however, we see that we
already know that `b × c = 6`, so we don't have to perform that computation
again, and can rather skip directly to `(a = 4) + (b × c = 6)`. We thus have computed
the value of our expression in just one step instead of two, sparing us an
entire, tedious multiplication.

<!--
       a+b×c=10
       /    \
      /     b×c=6
     /     /   \
    a=4   b=2  c=3
-->

![Updating the Computation][algebra-update]{:class="center"}

Let's see how this scheme translates to compiler.


The Incremental Compiler
========================
The way we chose to implement incrementality in the Rust compiler is
straightforward: An incremental compilation session follows exactly the same
steps in the same order as a batch compilation session. However, when control
flow reaches a point where it is about to compute some non-trivial intermediate
result, it will try to load that result from the incremental compilation cache
on disk instead. If there is a valid entry in the cache, the compiler can just
skip computing that particular piece of data. Let's take a look at a (simplified)
overview of the different compilation phases and the intermediate results they
produce:

<!--
* Diagram: compilation phases and their intermediate results

         AST        type info      object
                       MIR          code        binary

       A A A A     T T T M M M     O O O O
       A A A A     T T T M M M     O O O O        B
       A A A A     T T T M M M     O O O O
    >===========>===============>===========>===========>
        parse       analysis       codegen     linking
-->
![Compiler Phases and their By-Products][compiler-phases]{:class="center"}

First the compiler will parse the source code into an abstract syntax tree
(AST). The AST then goes through the analysis phase which produces type
information and the [MIR][mir-intro] for each function. After that, if analysis
did not find any errors, the codegen phase will transform the MIR version of
the program into its machine code version, producing one object file per
source-level module. In the last step all the object files get linked together
into the final output binary which may be a library or an executable.

Comparing that with our example from above, the pieces of AST correspond to
`a`, `b`, and `c`, that is, they are the inputs to our incremental computation
and they determine what needs to be updated as we make our way through the
compilation process. The pieces of type information and MIR and the object
files, on the other hand, are our intermediate results, that is, they
correspond to the incremental computation cache entries we stored for
`b × c` and `a + b × c`. Where a cache entry looks like `b × c = 6` in the
algebra example, it would look like
`translate_to_obj_file(mir1, mir2, mir3) = <bits of the obj-file>` in the case
of the compiler.

So, this seems pretty simple so far: Instead of computing something a second
time, just load the value from the cache. Things get tricky though when we need
to find out if it's actually valid to use a value from the cache or if we
*have to* re-compute it because of some changed input.


Dependency Graphs
=================
There is a formal method that can be used to model intermediate results in a
computation and their "up-to-dateness" in a straight forward way, so-called
"dependency graphs". It looks like this: Each input and each intermediate
result is represented as a node in a directed graph. When computing the value
of some intermediate result `X`, we record which inputs and other intermediate
results can influence the value of `X` by adding an edge from each node we are
reading to the node representing `X`. In other words, when we are done, we have
an edge for each node the corresponding value of which has gone into computing
`X`. Let's go back to our algebra example to see what this looks like in
practice:

<!--
       a+b×c
       /   \
      /    b×c
     /    /   \
    a    b     c
-->
![Dependency Graph of a + b × c][algebra-dep-graph]{:class="center"}

As you can see, we have nodes for the inputs `a`, `b`, and `c`, and nodes for
the intermediate results `b × c` and `a + b × c`. The edges should come as no
surprise: There is an edge from `b × c` to `b` and one to `c` because those are
the values we needed to read when computing `b × c`. For `a + b × c` it's
exactly the same. Note, however, that the above graph is a tree just because
the computation it models has the form of a tree. In general dependency graphs
are directed acyclic graphs, as would be the case if we would if we would add
another intermediate result `b × c + c` to our computation:

<!--
       a+b×c   b×c+c
       /   \  /  |
      /    b×c   |
     /    /   \ /
    a    b     c
-->
![Example of a non-tree Dependency Graph][algebra-dep-graph-dag]{:class="center"}

What makes this data structure really useful is that we can ask it questions
of the form "if X has changed, is Y still up-to-date?". We just take the node
representing `Y` and collect all the inputs `Y` depends on by transitively
following all edges originating from `Y`. If any of those inputs has changed,
any value we have cached for `Y` cannot be relied on any more.


Dependency Tracking in the Compiler
===================================
When compiling in incremental mode, we always build dependency graph of
produced data: Every time, some piece of data is written (like an object file),
we record which other pieces of data we are accessing while doing so. This is
mostly done by putting data behind a layer of abstraction (most prominently the
so-called `DepTrackingMap`) that will add dependency edges transparently behind
the scenes. At the end of the compilation sessions we have all our data nicely
linked up:

<!--
         AST        type info      object
                       MIR          code        binary

       A A A A     T T T M M M     O O O O
       A A A A     T T T M M M     O O O O        B
       A A A A     T T T M M M     O O O O
    >===========>===============>===========>===========>
        parse       analysis       codegen     linking
-->

![Dependency Graph of Compilation Data][compiler-dep-graph]{:class="center"}

This dependency graph is then stored in the incremental compilation cache
directory along with the cache entries it describes.

At the beginning of a subsequent compilation session, we detect which inputs
(=AST nodes) have changed by comparing them to previous version. Given the
graph and the set of changed input, we can easily find all cache entries that
are not up-to-date anymore and just remove them from the cache:

<!--
         AST        type info      object
                       MIR          code        binary

       A _ A _     _ _ T M _ _     O _ O _
       _ A A A     T _ T _ _ M     O _ _ _        _
       A A _ A     T _ _ M _ M     O O _ _
    >===========>===============>===========>===========>
        parse       analysis       codegen     linking
-->
![Using the Dependency Graph to Validate the Incremental Compilation Cache][compiler-cache-purge]{:class="center"}

Anything that has survived this cache validation phase can safely be re-used
during the current compilation session.


"Faster! Up to 15% or more."[*][up-to-or-more]
=============================

Let's take look at some of the implications of what we've learned so far:

  - The dependency graph reflects the actual dependencies between parts of
    source code and parts of the output binary.
  - If there is some input node that is reachable from many intermediate
    results, e.g. a central data type that is used in almost every function,
    then changing the definition of that data type will mean that everything
    has to be compiled from scratch, there's no way around it.

In other words, the effectiveness of incremental compilation is very sensitive
to the structure of the program being compiled and the change being made.
Changing a single character in the source code might very well invalidate the
whole incremental compilation cache. Hopefully though, this kind of change is
a rare case and most of the time only a small portion of the program has to be
recompiled.


The Current Status of the Implementation
========================================

For the first spike implementation of incremental compilation, what we call the
alpha version now, we chose to focus on caching object files. Why did we do
that? Let's take a look at the compilation phases again and especially how much
time is spent in each one on average:

```
         AST        type info      object
                       MIR          code        binary

       A A A A     T T T M M M     O O O O
       A A A A     T T T M M M     O O O O        B
       A A A A     T T T M M M     O O O O
    >----------->--------------->----------->----------->
        parse       analysis       codegen     linking
                                   optimize

TIME:    5%            20%         **65%**       10%
```

As you can see, the Rust compiler spends most of its time in the optimization
and codegen passes. Consequently, if this phase can be skipped at least for
part of a code base, this is where the biggest impact on compile times can be
achieved.

With that in mind, we can also give an upper bound on how much time this
initial version of incremental compilation can save: If the compiler spends X
seconds optimizing when compiling your crate, then incremental compilation will
reduce compile times at most by those X seconds.

Another area that has a large influence on the actual effectiveness of the
alpha version is dependency tracking granularity: It's up us how fine-grained
we make our dependency graphs, and the current implementation makes it rather
coarse in places. For example, the dependency graph only knows a single node
for all methods in an `impl`. As a consequence, the compiler will assume *all*
methods of type as changed if just one of them is changed. This of course will
mean that more code will be re-compiled than is strictly necessary. The current
implementation is still more a proof of concept more than anything else.

[Performance Numbers]
=====================
<!-- TODO!!
* Here are some numbers for different crates:
  - Graph with perfect re-use for syntex, some crate with little code but lots
    of monomorphizations (rustc_driver), some crate with little to optimize

    => Interpretation: IC-alpha shines where there's lots of optimization

  - Graph with regular compilation vs initial compile with IC
    => Interpretation: initial compilation slightly slower b/c of dependency
                       tracking. Should not make much of a difference

- Graph with changing different functions in syntex
    => Interpretation: illustrates input sensitivity
-->

[The Future]
============
<!-- TODO!!
* The alpha version represents minimal viable implementation of incremental
  compilation for Rust compiler, but there's lots of room for improvement.
* In terms of performance, this improvement will happen along two major axis:
  - caching more things, like MIR and type information, allowing the compiler
    to skip more and more steps
  - making dependency tracking more precise, so the compiler encounters fewer
    false positives during cache invalidation. E.g. currently no distinction
    between function signature and function body. Should make big difference in
    practice.
  Improvements in both of these directions will make IC more effective as the
  implementation matures.
* In terms of correctness, we tried to err on the side of caution from the
  get-go, however, we are still planning to improve regression tests and safety
  measures, specifically via fuzzing (?), more regression tests, ...
  (Call to action?)
-->



[meetup]: https://air.mozilla.org/bay-area-rust-meetup-may-2016/
[mir]: https://blog.rust-lang.org/2016/04/19/MIR.html
[compiling]: https://xkcd.com/303/
[up-to-or-more]: https://xkcd.com/870/
[algebra-initial]: /images/2016-08-Incremental/algebra-initial.svg
[algebra-update]: /images/2016-08-Incremental/algebra-update.svg
[algebra-dep-graph]: /images/2016-08-Incremental/algebra-dep-graph.svg
[algebra-dep-graph-dag]: /images/2016-08-Incremental/algebra-dep-graph-dag.svg
[compiler-phases]: /images/2016-08-Incremental/compiler-phases.svg
[compiler-dep-graph]: /images/2016-08-Incremental/compiler-dep-graph.svg
[compiler-cache-purge]: /images/2016-08-Incremental/compiler-cache-purge.svg
