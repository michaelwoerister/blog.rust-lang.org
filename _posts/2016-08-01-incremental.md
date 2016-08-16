---
layout: post
title: "Incremental Compilation"
author: Michael Woerister
description: "Incremental compilation for exponential joy and happiness."
---

I remember when, during the 1.0 anniversary presentation at the
[Bay Area Meetup][meetup], Aaron Turon talked about Dropbox having been
pretty happy with using Rust in production there. "The core
team has been in touch with them regularly", he said, "asking them, you know,
what do you need? And their answer is always: faster compiles ..." To the
learned Rust user it's no surprise that this solicited a knowing chuckle from
the audience. Well, improving compilation speed has been a major development
focus after Rust reached 1.0 -- even though up to this point much of the work
towards this goal has gone into laying [architectural foundations][mir] within
the compiler and we are only starting to see actual results. One of the
projects targeted at improving compilation speed is *incremental compilation*,
which has been worked since the end of last year and which, as of recently, has
progressed to a stage where we are beginning to get comfortable with letting
people get their hands on it.

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
there are only so many dishes to be done, only so much [regressive fun][compiling]
to be had while `rustc` bootstraps.

Incremental compilation is a way of exploiting the temporal coherence a code
base exhibits during the regular programming workflow: Many, if not most,
of the changes done in between two compilation sessions only have local impact
on the machine code in the output binary, while the rest of the program,
same as at the source level, will end up just the same, bit for bit.
Incremental compilation aims at retaining as much of those unchanged
parts while redoing only that amount of work that actually *has* to be redone.
The next few paragraphs will give a little tour of the basic strategy we are
pursuing in order to implement incremental compilation in the Rust compiler,
of the kind of compile time speedups and runtime trade-offs that are to be
expected, and finally, the kind of progress the actual implementation has made
so far.


### Concepts

In order to understand how the specific flavor of incremental compilation we
are implementing in the Rust compiler works, we need to know some basics about
how the compiler works in general.

![...](/images/2016-08-Incremental/phases.svg){:style="float:left"}

The Rust compiler follows an architecture that is rather common for ahead-of-time
compilers: Each compilation session consists of a number of consecutive,
non-overlapping phases that analyze their input and produce some output, which
in turn will be the input to a later phase. For example, the parsing phase takes
the source code as input and produces the AST (abstract syntax tree) from it,
the name resolution phases takes the AST and produces the name resolution tables,
which in turn, together with the AST are used to produce the HIR (high-level
intermediate representation), and so on and so forth.



### "Faster! Up to 15% or more." [*][up-to-or-more]

change central data type




[meetup]: https://air.mozilla.org/bay-area-rust-meetup-may-2016/
[mir]: https://blog.rust-lang.org/2016/04/19/MIR.html
[compiling]: https://xkcd.com/303/
[up-to-or-more]: https://xkcd.com/870/
