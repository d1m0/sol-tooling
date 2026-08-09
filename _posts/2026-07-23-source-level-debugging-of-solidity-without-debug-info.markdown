---
layout: post
title:  "Debugging Solidity Without Debug Information"
date:   2026-07-23 08:48:33 -1000
categories: interpreter
author: Dimitar Bounov
---

# Introduction

This post is the first in a series exploring a new approach to source-level
debugging for Solidity contracts. Debugging is a core part of any language
ecosystem, but has been lacking in the case of Solidity contracts, due to
missing and/or broken debug information emitted from the compiler. My approach
to debugging instead focuses on interpretation and achieves a **99.8%**
coverage[^1] while debugging a sample of **~100K** transactions (TXs) from
mainnet, covering **~36K** different real world deployed Solidity contracts.

To achieve this I built a *full*[^12] Solidity Interpreter (i.e. an **interpreter for
Solidity ASTs**). The interpreter handles almost all major Solidity language features
with the exception of inline assembly (left as future work).

While the interpreter executes Solidity ASTs, it works over low level state
almost bitwise identical with the EVM state[^13].

This first post will give a brief overview of this work. Future 
posts will explore the design of the Solidity interpreter built, the
algorithm for verifiable replay of EVM transactions using the interpreter, and some interesting engingeering challenges that arose during this project.

# The Problem

For compiled languages, debugging relies on additional information emitted by
the compiler. This usually includes source maps (which map instructions back to
source locations), stack maps (which map low-level stack locations back to
source-level local variables) and global symbol tables. Broadly, debugging
information ties parts of the compiled program back to specific parts of the
source code.

In the case of Solidity the compiler does not yet emit stack maps, and source
maps have been [broken](https://github.com/argotorg/solidity/issues/14930) for optimized source code. While there is ongoing
work to [address](https://github.com/argotorg/solidity/issues/15884) these issues in future versions of the compiler, all
existing compiled contracts still suffer from partial and sometimes broken debug
information. Due to the immutable nature of the Ethereum blockchain, these
existing contracts are here to stay, and will continue to interact with new code
in perpetuity.

This leads to the core problem: Its difficult to provide a debugging experience
both for existing optimized contracts and newly compiled contracts, due to
inconsistent and partial debug information.

# Idea

## Different perspective on debugging

At a high level, the goal of debugging is to provide an explanation at the
source code level of what a particular low-level execution did. The lack of
necessary debug information from the Solidity compiler complicates this task for
arbitrary optimized code.

There are existing[^7][^8][^9][^10][^11] debuggers for Solidity, but they either
rely on source maps, and thus suffer from the compiler bugs around source maps,
or are still in an early experimental phase.

The core innovation of this work is its focus on **interpretation** of the
source code itself, instead of mapping each instruction of an EVM execution
trace back onto the source code. The additional verification of equivalent
observable behavior between the intereter and the EVM provides a high assurance
for the correctness of the debugger behavior.  From a technical point of view,
this approach builds a simulation relation for an individual pair of execution
traces.

The first core contribution of this work is a full Solidity interpreter that interprets
Solidity ASTs[^2] but works over low-level EVM state (EVM storage, linear byte
memory, etc). The design choice of having the interpreter work over state as
close as possible to the low-level EVM state is crucial to this work. This
choice of state allows for easiler comparisong between the observable behaviors
produced by the interpreter and the EVM.

The second core contribution is tooling that runs the interpeter over segments of EVM execution traces, and produces both a high-level execution trace (useful for debugging), as well as the simulation relation that certifies that the interperter's observable behavior matches the EVM's.

As a nice bonus, the design of the interpeter and the surrounding tooling allows
replay/debugging to be restarted in many places in the middle of a trace, by
re-building interpreter state from the EVM state at that point. This design can
cover more of an EVM execution trace, in the face of low-level exception and
other causes of "misalignment" between the interpreter and the EVM. Future posts will
delve further into this recovery mechanism.

## Pros/Cons

This approach has several pros/cons:

Pros:

1. Doesn't require additional debug information from the compiler; doesn't even use source maps.
2. Produces a Solidity Interpreter as an additional artifact. This can be independently used (e.g. for differential fuzzing of compilers)

Cons:

1. Its a significant engineering effort to build and maintain a full interpreter for Solidity
2. This approach relies on the compiler not re-ordering "externally visible events"

A full language interpreter is essentially an executable formalization of the
language semantics. While it is a significant engineering effort (con), it is
useful as a second implementation against which one can differentially fuzz a
compiler(pro).

In fact, while replaying mainnet TXs, I observed differences in
behavior between the interpreter and the compiled code that were due to bugs
later fixed in the compiler!

A source-level interpreter is also a useful template for a symbolic execution
engine for Solidity.

# Debugging with Interpretation

## Motivating Example

Consider the Solidity sample below. It contains 2 contracts, `A` and `B`, each with a method `foo()` with `A.foo()` invoking `B.foo()`.

{% highlight solidity %}
contract A {
    B b;
    function foo(uint x) public returns (uint) {
        return b.foo(x + 1, 1) * 2;
    }
}

contract B {
    function foo(uint a, uint b) public (returns uint) {
        return a + b;
    }
}
{% endhighlight %}

Next consider an execution trace for the call `A.foo(1)`. The trace can be split
in 3 segments, as shown in Fig 1. below - 1) any instructions in `A.foo()` prior
to the call to `b.foo(x + 1, 1)`, 2) any instructions inside `B.foo()` and finally
3) the remaining instructions in `A.foo()` after the call.

![Trace Segments](/sol-tooling/assets/images/trace01.jpeg)

The first segment contains the instructions for evaluating `x + 1`, `1` and setting up the call to `b.foo(x + 1, 1)`.
The second segment, contains the evaluation of `a + b` inside `B.foo()`.
The third segment contains the multiplication of the result of the call by `2` and returning the resulting value.

## Observable events

The boundaries of those external segments are externally observable events. Consider the following type sof externally observable vents [^3]:

| Event Type | Example |
| --- | --- | 
| External Call | `b.foo(...)` |
| External Return | `return ...` |
| Exception | `require(...)`, `assert(...)`, `revert ...`, builtin exceptions, etc |
| Event Emission | `emit Transfer(...)` |

A core requirement for this approach is that a compiler will not:

1. Re-order observable events

2. Move storage writes across external call/return boundaries

I believe both expectations are reasonable, given that (1) would directly change the observable
behavior of the contract[^4] and (2) may change the behavior of the contract in
case of callbacks.

Assuming that the compiler respects the given requirements, at the given observable event boundaries 
I can check that the interpreter has the same intermediate state (storage)[^14], and
that the interpreter is doing the exact same observable event as the EVM.

An additional advantage of comparing the state of the interpreter and the EVM
execution at each intermediate boundary is that even if it encounters a
misalignment half way through a trace, it is still possible to resume debugging
at certain points later in the trace. More details on how this works will be in
a future post.

Lets see how this would work for the example call above.

## Debugging the example trace

To begin debugging the call to `A.foo(1)`, the interpreter is initialized with a
state containing an empty memory, storage equivalent to the storage of
`A` at the start of the EVM trace and the same `msg.data` as the EVM trace's
`msg.data`.

At this point interpreter executes until it reaches an observable
event. This would be the external call to `B.foo(x, 1)` (see Fig 2). 

The interpreter evaluates `x` to `1`, and thus builds up a low-level message data
for the call to `B.foo(x,1)` containing `B.foo()`'s, selector and the arguments
`(1, 1)`. At this point I identify the corresponding low-level observable event by scanning forward in the EVM trace.
I verify that the encountered low-level observable event is a `CALL` instruction,
that the target address matches the address of the high-level call (`B`'s
address), and that two calls share the same `msg.data` and value.

![Debugging the First Segment](/sol-tooling/assets/images/first_segment3_pre_crop.gif)

At this point the above checks establish that the interpreter matches the
*externally* observable behavior of the EVM trace up to the first call.  That
is, the interpreter reaches the same external call as the EVM trace, and leaves
the calling contract in the same state as the EVM trace before the call.

Interpretation continues in the context of `B.foo()`.  A new interpreter
instance is created with the storage of `B` before the call, an empty memory,
and the message arguments and value from the end of the first segment.  The
interpreter runs in `B.foo()` until it reaches an externally observable event -
in this case the return from `B.foo()` (see Fig 3). 

Again the corresponding low-level observable event is identified by scanning forward in the EVM trace.
I verify that the low-level event is a `RETURN` instruction, that the returned
data is the same between the interpreter and the EVM trace, and that the storage
of `B` is the same between the interpreter and the EVM trace at the `RETURN`
instruction.

![Debugging the Second Segment](/sol-tooling/assets/images/second_segment2_pre_crop.gif)

This establishes that the interpreter exactly matches the observable behavior of
the EVM up to the end of the second segment.

Finally I repeat the same steps for the 3rd segment - continue interpreting in
the original interpreter instance (i.e. in the context of `A.foo()`), using the
returned data from `B.foo()`. Next run the interpreter until it hits the final
observable event - the return from `A.foo()`. Finally scan forward in the EVM
trace until it hits an observable event as well, and verify that it's a `RETURN` with
the same return data as the interpreter. I also verify that the final state of
the `A` contract in the interpreter is the same as the final state of `A` in the
EVM trace (see Fig 4).

![Debugging the Third Segment](/sol-tooling/assets/images/third_segment2_pre_crop.gif)

Together, all these checks establish that the interpreter produced the same
observable behavior as the EVM trace.

This procedure establishes a [simulation
relation](https://en.wikipedia.org/wiki/Simulation_(computer_science)) between
an individual execution of the interpreter and the EVM. The simulation relation
serves as a witness, that the two have the same observable behavior for this
particular execution.

Future posts will go into further depth on the design of the interpreter as well as the exact algorithm for matcing and comparing observable events (called *trace alignment* here). Crucially, future blog posts will explore
how I recover from cases where I run into a mismatch at some observable event
pair (e.g. due to an Out-of-Gas exception), and still manage to continue
interpreting later in the trace. This allows for a greater cover of a given
execution trace, when faced with low-level exceptions that cannot be modeled, or bugs
in the interpreter.

# Evaluation

To evaluate this work I collected 737 Ethereum Blocks at regular intervals starting from block 1150000 until block 24150001.
Next I extracted 101045 TXs out of those blocks. The extracted transactions interact with roughly 36K different deployed contracts, spanning all versions from 0.4.13 to 0.8.29.

For each of those TXs, I:

1. Obtained the expected storage of all contracts involved in the TX
2. Tried getting source code/compiler artifacts from Etherscan for each contract touched by the TX. For the contracts with source code I tried compiling it locally
3. For any segments in the TX, where I had successfully obtained a compilation artifact, I tried running the interpreter for those segments, and checked whether the interpreter produced the same observable behavior, and resulted in the same contract states as the EVM trace.

Currently there is one major language feature still not implemented - inline
assembly. Inline assembly was left as future work, due to the sheer amount of
time and effort required to implement the language interpreter as it stands. 

Trace segments can be split into several categories:

| Segment Type | Explanation |
| --- | --- | 
| No Source | Etherscan didn't have source code for this segment, or its source didn't compile locally, or it produced bytecode that diverged from the mainnet implementation [^6] |
| Aligned | This segment is part of a group of segments that were correctly replayed by the interpreter |
| Misaligned:Inline Assembly | This segment could not be replayed because it (or a previous segment) contained Inline Assembly |
| Misaligned:Low-level Exception | This segment could not be replayed because it (or a previous segment) contained a low-level exception (mostly Out-of-Gas) |
| Misaligned:Error | This segment could not be replayed because it (or a previous segment) triggered a bug or unimplemented feature in the interpeter|

Of the above categories, *No Source* is of no interest - there is nothing to do if there is no source code.

I consider *Misaligned:Inline Assembly* out-of-scope right now. There is a clear path to implementing inline assembly, so covering these segments is just a matter of engineering effort.

The most interesting question for this approach is, how often it encounters
the remaining two categories - *Misaligned:Low-level Exception* and
*Misaligned:Error*.

The first category is an event that fundamentally can't be handled. At the
Solidity level it's impossible to model the concepts of *gas* and *EVM stack
size* (However I do model the `gaselft()` function correctly! More on this in
future posts ;)).  The second category measures remaining bugs in the current
implementation.

So lets look at some ...

# Results

The 101045 TXs touched code in ~36K contracts, spanning all compiler versions from 0.4.13 to 0.8.29. The execution traces were split into a total of 971747 segments, with the following breakdown by segment type:

| Segment Type | Count |
| --- | --- | 
| No Source | 225407 |
| Aligned | 530505 |
| Misaligned:Inline Assembly | 215066 |
| Misaligned:Low-level Exception | 268 |
| Misaligned:Error | 501 |
| Total | 971747 |

As mentioned earlier, there is nothing to do for segments with no source code.
Furthermore, inline assembly is not yet implemented, so for now, I
consider those to be out-of-scope. There is a clear path for handling inline assembly.
The design decisions to use the same storage format as the EVM, and to use a
memory layout for objects that is also the same as the EVM enable a relatively
easy integration for inline assembly.

Below is the breakdown of all segments, as well as the breakdown of the
segments of interest (i.e. ignoring no-source and inline assembly segments).


![Breakdown of All Segments](/sol-tooling/assets/images/breakdown_all_segments.jpg)

![Breakdown of All Segments](/sol-tooling/assets/images/breakdown_segments_of_interest.jpg)

The core takeaway here is, that this approach covers **99.8%** of all trace
segments for which there is source code, and which are not precluded by an inline
assembly block!

The above experiment demonstrates, that it is possible to **fully** and **verifiably**
explain the behavior of EVM traces at the source code level, for **arbitrary
optimized code** without even using source map information.

# Conclusion

This article provides a brief overview of my work on debugging Solidity
contracts using an interpreter. The next few posts will delve further into the
design of the interpreter, the full algorithm for aligning observable events
between the interpretere and the EVM (i.e., building the silumation relation),
and finally some interesting edge cases (including known compiler bugs!). I hope
this was enough to keep you curious for more! In the meantime, check out the
[code](https://github.com/d1m0/sol-interp/).

# Acknowledgements

This work is graciously supported by a grant from the Ethereum Foundation. I
developed some of the underlying libraries
([sol-dbg](https://github.com/consensysdiligence/sol-dbg) and parts of
[solc-typed-ast](https://github.com/consensysdiligence/solc-typed-ast)) while
working with the wonderful folks at [Consensys
Diligence](https://diligence.security/).

Special shout-outs: to [Pavel Zevrev](https://github.com/blitz-1306) who built
most of [solc-typed-ast](https://github.com/consensysdiligence/solc-typed-ast)
himself, to Valentin Wustholz and Joran Honning for all their useful discussions
and engineering insights, and to all the current and former Dili folks - y'all
are awesome!

# Funding

This project is actively looking for funding! Please [reach out](mailto:sol-tooling@proton.me) if interested!
Additional funding would allow me to:

 - Implement inline assembly (covering the largest gap of missing traces)
 - Add support for more recent compilers (0.8.30 onwards)
 - Re-factor the code base to be deployable directly in the browser, clientside
 - ...and more :)

# Footnotes

[^1]: More specifically, I debugged *all* 101045 TXs, and each one was split into segments as explained later in this article. I successfully debugged 99.8% of all in-scope trace segments. A segment was considered 'in-scope' if a) it has source code and b) it does not contain inline assembly, or is precluded by an earlier segment with inline assembly. Inline assembly is currently considered future work.

[^2]: Technically, the interpreter works over the JSON compiler artifacts emitted by solc. On top of the ASTs, it also uses the link and immutable references and the bytecode (for example to provide the `contract.code` builtin). Crucially, the interpreter doesn't look at source maps.

[^3]: On top of these, `GASLEFT` instructions are also technically an "observable event". This is to align calls to the `gasleft()` builtin with corresponding `GASLEFT` instructions, to allow the correct value for `gasleft()` to be returned in the interpreter. More on this in later posts.

[^4]: One interesting complication here is undefined behavior. For example the ordering of `a.foo() + b.foo()` is not strictly defined, so the compiler is free to re-order those. And I found differences in the ordering between the old codegen pipeline and the new IR pipeline. More on this in later posts.

[^6]: When re-compiling contracts locally, even when using the exact settings and artifact from Etherscan, I still sometimes ran into differences in the metadata hashes embedded in the bytecode. I implemented a more "fuzzy" match to account for this.

[^7]: http://github.com/remix-project-org/remix-project/tree/master/libs/remix-debug

[^8]: https://github.com/edb-rs/edb

[^9]: https://github.com/ConsenSys-archive/truffle/tree/develop/packages/debugger

[^10]: https://github.com/foundry-rs/foundry/tree/master/crates/debugger/src

[^11]: https://docs.runtimeverification.com/simbolik

[^12]: The only major features missing currently are inline assembly, transient state variables and the `layout at` construct. Additionally, there are a couple of builtins still not implemented. All of these are just a matter of time and engineering effort.

[^13]: Storage, message data, return data and exception data are bitwise
identical between the interpreter and the EVM. In interpreter memory individual
objects have bitwise identical layout as they would in the EVM. The only
difference in memory is the ordering of allocations, and the existence of
intermediate temporary allocations in both the interpreter and the compiled
code.

[^14]: Note that I don't compare storages at (and only at) event emission and `GASLEFT`. Instead only the emitted events are checked to be identical. This is to allow for the compiler reordering storage writes across event emission, as that doesn't change the program behavior.