---
layout: post
title: "Incremental Compilation"
author: Michael Woerister
description: "Incremental compilation for exponential joy and happiness."
---

I remember when, during the 1.0 anniversary presentation at the
[Bay Area Meetup][meetup], Aaron Turon talked about Dropbox so far having been
pretty happy with using Rust in production there. *The core
team has been in touch with them regularly*, he said, *asking them, you know,
what do you need? And their answer is always: faster compiles ...* To the
learned Rust user it is no surprise that this solicited a knowing chuckle from
the audience. Improving compile times has actually been a major development
focus after Rust reached 1.0 -- although, up to this point, much of the work
towards this goal has gone into laying [architectural foundations][mir] within
the compiler and we are only slowly beginning to see actual results.

One of the projects that is building on these foundations, and that should
help **improve compile times** a lot for some specific but very common workflows,
is **incremental compilation**. This new feature, once it is fully implemented,
will allow you to iterate on your code much more quickly.

Today we are announcing an **alpha version** of incremental compilation, which
marks an important milestone in the development of the feature: For the first
time since implementation started towards the end of last year, all of the
[basic components][incr-comp-rfc] are in place, the bulk of the groundwork has
been done. You can give it a try in the nightly version of the compiler:

```
rustc -Zincremental=<path> ./main.rs
```

This will start the compiler in **incremental mode**, using whatever `<path>`
you've provided as the incremental compilation cache directory.

With all that being said, incremental compilation is **not production-ready** yet:
You might see crashes, you might see cases where there is no actual reduction
in compile times and, most importantly, we still have to write extensive
regression tests that make sure that incrementally compiled programs are
always correct — so don't use it anywhere yet where it really matters. Over
the next few weeks and months, however, our focus will be on making the
implementation rock-solid from a correctness point of view and you will see
continuous, gradual improvements in the feature's efficiency, up to a point
where it will be transformative to your development experience.

This blog post will go through why and when incremental compilation is useful
to begin with, how our implementation of it works, what its current development
status is, and finally what's planned for the future and how you can contribute,
should you be so inclined.


Why Incremental Compilation in the First Place?
===============================================
Much of a programmer's time is spent in an **edit-compile-debug** workflow:

- you make a **small change** (often in a single module or even function),
- you let the compiler **translate the code into a binary**, and finally
- you **run the program** or a bunch of unit tests in order to see results of
  the change.

After that it's back to step one, making the next small change informed
by the knowledge gained in the previous iteration.
In order for this essential feedback loop, that is at the core of our daily
work, to be smooth and effective, the time being stalled while waiting for the
compiler to produce an executable program must be as short as possible, ideally
short enough as not to warrant a time-consuming and stress-inducing mental
context switch: You want to be able to keep working, stay in the zone. After
all, there is only so much [regressive fun][compiling] to be had while `rustc`
bootstraps.

Incremental compilation is a way of **exploiting** the **temporal coherence** a
code base exhibits during the regular programming workflow: Many, if not most,
of the changes done in between two compilation sessions only have local impact
on the machine code in the output binary, while the rest of the program,
same as at the source level, will end up exactly the same, bit for bit.
Incremental compilation aims at retaining as much of those unchanged
parts as possible while redoing only that amount of work that actually *must*
be redone. The next few paragraphs will give a little tour of the basic strategy
we are pursuing in order to implement incremental compilation in the Rust compiler,
of the kind of compile time speedups and runtime trade-offs that are to be
expected, and finally, the kind of progress the actual implementation has made
so far.


How Do You Make Something "Incremental"?
========================================
We have already heard that computing something incrementally means updating
only those parts of the computation's output that need to be changed in
response to a given change in the computation's inputs.
One basic strategy we can employ to achieve this is to view one big computation
(like compiling a program) as a **composite** of many smaller, interrelated
computations that build up on each other. Each of those smaller computations
will yield an **intermediate result** that can be **cached** and hopefully
**re-used** in a later iteration, sparing us the need to re-compute that
particular intermediate result again.

Let's take a look at a simple example from algebra to make things more
concrete. Let's see what it means to evaluate an expression of the form
`a+b×c` incrementally. This will involve evaluating the expression once
with one set of values for `a`, `b`, and `c`, and then evaluating it a second
time with a different value for `a`. For the first time around, `a` will be
`1`, `b` will be `2`, and `c` will be `3`:

<!--
       a+b×c=7
       /    \
      /     b×c=6
     /     /   \
    a=1   b=2  c=3
-->
![Initial Computation of a+b×c][algebra-initial]{:class="center"}

Assume that we "saved" the intermediate results at each step, that is, we
remember somewhere that `b×c` is `6` and `a+b×c` is `7`. Now, in the second
round, we want to know what `a+b×c` is if we change the value of `a` to
`4`. When we recompute the value of the expression, however, we see that we
already know that `b×c = 6`, so we don't have to perform that computation
again, and can rather skip directly to `(a = 4) + (b×c = 6)`. We thus have computed
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

Let's see how this scheme translates to the compiler.


An Incremental Compiler
=======================
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
information and the [MIR][mir] for each function. After that, if analysis
did not find any errors, the codegen phase will transform the MIR version of
the program into its machine code version, producing one object file per
source-level module. In the last step all the object files get linked together
into the final output binary which may be a library or an executable.

Comparing that with our algebra example from above, the pieces of AST correspond
to `a`, `b`, and `c`, that is, they are the **inputs** to our incremental
computation and they determine what needs to be updated as we make our way
through the compilation process. The pieces of type information and MIR and the
object files, on the other hand, are our **intermediate results**, that is, they
correspond to the incremental computation cache entries we stored for
`b×c` and `a+b×c`. Where a cache entry looks like `b×c = 6` in the
algebra example, it would look something like
`translate_to_obj_file(mir1, mir2, mir3) = <bits of the obj-file>` in the case
of the compiler.

So, this seems pretty simple so far: Instead of computing something a second
time, just load the value from the cache. Things get tricky though when we need
to **find out if** it's actually valid to **use a value from the cache** or if we
have to **re-compute** it because of some changed input.


Dependency Graphs
=================
There is a formal method that can be used to model a computation's intermediate
results and their "up-to-dateness" in a straightforward way, so-called
**dependency graphs**. It looks like this: Each input and each **intermediate
result** is represented as a **node** in a directed graph. The **edges** in the
graph, on the other hand, represent which intermediate result or input can have
an **influence** on some other intermediate result. More formally, let `v` be
some intermediate result and `N(v)` be the dependency graph node representing
`v` in the graph. Further, let `xᵢ` be the intermediate results and inputs
that have to be **read** when computing `v` and let `N(xᵢ)`
be their representing nodes. Then the dependency graph will contain the edges
`N(v) → N(xᵢ)` for all `i`. In other words, each edge records a dependency
of one intermediate result on another.

Let's go back to our algebra example to see what this looks like in
practice:

<!--
       a+b×c
       /   \
      /    b×c
     /    /   \
    a    b     c
-->
![Dependency Graph of a+b×c][algebra-dep-graph]{:class="center"}

As you can see, we have nodes for the inputs `a`, `b`, and `c`, and nodes for
the intermediate results `b×c` and `a+b×c`. The edges should come as no
surprise: There is an edge from `b×c` to `b` and one to `c` because those are
the values we need to read when computing `b×c`. For `a+b×c` it's
exactly the same. Note, by the way, that the above graph is a tree just because
the computation it models has the form of a tree. In general, dependency graphs
are **directed acyclic graphs**, as would be the case if we would
add another intermediate result `b×c+c` to our computation:

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
When compiling in incremental mode, we always build the dependency graph of the
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
(=AST nodes) have changed by comparing them to the previous version. Given the
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


"Faster! Up to 15% or More."[*][up-to-or-more]
=============================

Let's take a look at some of the implications of what we've learned so far:

  - The dependency graph reflects the actual dependencies between parts of the
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
that? Let's take a look at the compilation phases again and especially at how
much time is spent in each one on average:

<!--
         AST        type info      object
                       MIR          code        binary

       A A A A     T T T M M M     O O O O
       A A A A     T T T M M M     O O O O        B
       A A A A     T T T M M M     O O O O
    >===========>===============>===========>===========>
        parse       analysis       codegen     linking
                                   optimize

TIME:    5%            20%         **65%**       10%
-->
![Relative Cost of Compilation Phases][compiler-phases-cost]{:class="center"}

As you can see, the Rust compiler spends most of its time in the optimization
and codegen passes. Consequently, if this phase can be skipped at least for
part of a code base, this is where the biggest impact on compile times can be
achieved.

With that in mind, we can also give an upper bound on how much time this
initial version of incremental compilation can save: If the compiler spends X
seconds optimizing when compiling your crate, then incremental compilation will
reduce compile times at most by those X seconds.

Another area that has a large influence on the actual effectiveness of the
alpha version is dependency tracking granularity: It's up to us how fine-grained
we make our dependency graphs, and the current implementation makes it rather
coarse in places. For example, the dependency graph only knows a single node
for all methods in an `impl`. As a consequence, the compiler will assume *all*
methods of that `impl` as changed if just one of them is changed. This of course
will mean that more code will be re-compiled than is strictly necessary. The
current implementation is still a proof of concept more than anything else.

Performance Numbers
===================

> TODO
>
> * Here are some numbers for different crates:
>  - Graph with perfect re-use for syntex, some crate with little code but lots
>    of monomorphizations (rustc_driver), some crate with little to optimize
>
>    => Interpretation: IC-alpha shines where there's lots of optimization
>
>  - Graph with regular compilation vs initial compile with IC
>
>    => Interpretation: initial compilation slightly slower b/c of dependency
>                       tracking. Should not make much of a difference
>
>  - Graph with changing different functions in syntex
>
>    => Interpretation: illustrates input sensitivity
>

Future Plans
============
The alpha version represents minimal end-to-end implementation of incremental
compilation for Rust compiler, so there is lots of room for improvement. The
paragraph on the current status already laid out the two major axis along which
we will pursue increased efficiency:

- **Cache more** intermediate results, like MIR and type information, which will
  allow the compiler to skip more and more steps.
- Make **dependency tracking more precise**, so that the compiler encounters fewer
  false positives during cache invalidation.

Improvements in both of these directions will make IC more effective as the
implementation matures.

In terms of correctness, we tried to err on the side of caution from the get-go,
rather making the compiler recompute something if we were not sure if our
dependency tracking did the right thing, but there is still more that can be
done.

- We could use many **more auto-tests** that make sure that various basic components
  of the system **don't regress**. This is an area where interested people can
  start contributing with relative ease, since one only needs to understand the
  Rust language and the test framework, but not the more complicated innards of
  the compiler's implementation. [TODO: add link to GH meta issue]

- We have a neat little [tool][cargo-incremental] (implemented as a Cargo
  subcommand for hassle-free installation and usage) that will walk a projects
  git history, compiling successive versions of the source code and
   **collecting data** on the **efficiency** and **correctness** of
  incremental versus regular compilation. If you're interested in helping out,
  consider yourself invited t o download the tool and run it on a project of
  yours!

> TODO: Conclusion. Where to go for information.

[meetup]: https://air.mozilla.org/bay-area-rust-meetup-may-2016/
[mir]: https://blog.rust-lang.org/2016/04/19/MIR.html
[incr-comp-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1298-incremental-compilation.md
[compiling]: https://xkcd.com/303/
[up-to-or-more]: https://xkcd.com/870/
[algebra-initial]: /images/2016-08-Incremental/algebra-initial.svg
[algebra-update]: /images/2016-08-Incremental/algebra-update.svg
[algebra-dep-graph]: /images/2016-08-Incremental/algebra-dep-graph.svg
[algebra-dep-graph-dag]: /images/2016-08-Incremental/algebra-dep-graph-dag.svg
[compiler-phases]: /images/2016-08-Incremental/compiler-phases.svg
[compiler-dep-graph]: /images/2016-08-Incremental/compiler-dep-graph.svg
[compiler-cache-purge]: /images/2016-08-Incremental/compiler-cache-purge.svg
[compiler-phases-cost]: /images/2016-08-Incremental/compiler-phases-cost.svg
[cargo-incremental]: https://github.com/nikomatsakis/cargo-incremental
