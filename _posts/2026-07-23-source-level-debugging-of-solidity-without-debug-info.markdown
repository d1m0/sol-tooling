---
layout: post
title:  "Debugging Solidity Without Debug Information"
date:   2026-07-23 08:48:33 -1000
categories: interpreter
---

# Introduction

This post is the first in a series exploring a new approach to source-level
debugging of Solidity contracts. Debugging is a core part of any language
ecosystem, but in the case of Solidity contracts it has been lacking. Our work
took an alternative approach to debugging, centered around interpretation, that
achieved excellent results. We successfully debugged 99.8% of the in-scope[^1]
mainnet transactions across a sample of ~100K TXs.

This first post will serve as a brief overview of our work. In the following
posts we will explore the design of the Solidity interpreter we've built, the
algorithms for verifiable replay of EVM transactions using an interpreter, and
some interesting engineering challenges we've had to overcome.

# The Problem

For compiled languages, debugging relies on additional information emitted by
the compiler. This usually includes source maps (which map instructions back to
source locations), stack maps (which map low-level stack locations back to
source-level local variables) and global symbol tables. Broadly, debugging information
allows us to tie parts of the compiled program back to specific parts of the
source code.

In the case of Solidity the compiler does not yet emit stack maps, and source
maps have been [broken](TODO) for optimized source code. While there is ongoing
work to [address](TODO) these issues in future versions of the compiler, all
existing compiled contracts still suffer from partial and sometimes broken debug
information. Due to the immutable nature of the Ethereum blockchain, these
existing contracts are here to stay, and will continue to interact with new code
in perpetuity.

This leads to the core problem: Its difficult to provide a debugging
experience both for existing optimized contracts and newly compiled contracts.

# Idea

## Different perspective on debugging

At a high-level the goal of debugging is to provide an explanation at the
source code level of what a particular low-level execution did. Lacking the
additional information from the compiler makes this task very difficult for
arbitrary optimized code.

There are [existing](TODO) approaches that attempt to use a combination of
[symbolic execution](TODO) and [heuristics](TODO) to infer as much of the
missing debug information as possible, but these are incomplete. More
specifically they may fail for optimized code with broken source maps.

Our core insight, is that instead of trying to map an EVM execution trace back
onto the source code, we can just **interpret** the source code, using the same
starting state as the low-level execution trace, and then **verify** that the
interpreter trace produces the same observable events as the low-level execution
trace, and the same final state. At the heart of this idea is a full Solidity
interpreter that interprets Solidity (ASTs)[^2] but works over low-level EVM
state. The design choice of having the interpreter work over state as close as
possible to the low-level EVM state is crucial for this work, to enable being
able to compare the observable behavior produced by the interpreter, to what the
EVM is doing.

As a nice bonus, this enabled us to build an interpreter that can restart in many places in the middle of a trace, by getting its state from whatever the EVM state is at that point.
This allowed a design that can cover a lot more of an EVM execution trace, in
the face of low-level exception and other causes of "misalignment" between the
interpreter and the EVM.
More on this recovery will be covered in future posts.

## Pros/Cons

This idea has several pros/cons:

Pros:

1. This approach doesn't require additional debug information from the compiler. We don't even use source maps.
2. This approach produces a Solidity Interpreter as an additional artifact

Cons:

1. Its a significant engineering effort to build and maintain a full interpreter for Solidity
2. This approach relies on the compiler not re-ordering "externally visible events"

A full language interpreter is essentially an additional implementation of the
language semantics. While it is a significant engineering effort (con), it is
useful as a second implementation against which we can differentially fuzz a
compiler(pro).

In fact, while replaying mainnet TXs we observed differences in
behavior between the interpreter and the compiled code that were due to bugs
later fixed in the compiler!

A source-level interpreter is also a useful template upon which we can build a
symbolic execution engine for Solidity ASTs.

# Debugging with Interpretation

## Motivating Example

To illustrate our work, lets consider the Solidity sample below. It contains 2 contracts, `A` and `B`, each with a method `foo()` with `A.foo()` invoking `B.foo()`.

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
3) the remaining instructions in `A.foo()` after that call.

![Trace Segments](/sol-tooling/assets/images/trace01.jpeg)

The first segment contains the instructions for evaluating `x + 1`, `1` and setting up the call to `b.foo(x + 1, 1)`.
The second segment, contains the evaluation of `a + b` inside `B.foo()`.
The third segment contains the multiplication of the result of the call by `2` and returning the resulting value.

## Observable events

The boundaries of those external segments are externally observable events. We consider several types of externally observable events[^3]:

| Event Type | Example |
| --- | --- | 
| External Call | `b.foo(...)` |
| External Return | `return ...` |
| Exception | `require(...)`, `assert(...)`, `revert ...`, builtin exceptions, etc |
| Event Emission | `emit Transfer(...)` |

A core requirement of our approach is the idea that a compiler would not:

1. Re-order these events, as that would change the observable behavior of the program[^4]

2. Move storage writes across external call/return boundaries as an optimization. (as that may change the behavior of the program in the case of callbacks)

Assuming that the compiler respects our 2 requirements, then at those boundaries
we can check that the interpreter has the same intermediate state (storage), and
that the interpreter is doing the exact same sequence of external calls, returns
and exceptions as the underlying compiled code.

An additional advantage of comparing the state of the interpreter and the EVM
execution at each intermediate boundary is that even if we encounter a
misalignment half way through a trace, it is still possible to resume debugging
at certain points later in the trace. More details on how this works will be in
future posts.

Lets see how this would work for our example call above.

## Debugging the example trace

To begin debugging the call to `A.foo(1)`, we initialize an interpreter with an
initial state containing an empty memory, storage equivalent to the storage of
`A` at the start of the EVM trace and the same `msg.data` as the EVM trace's
`msg.data`. (i.e. the ebi-encoded argument `1`).

At this point we let the interpreter execute until it reaches an observable
event. This would be the external call to `B.foo(x, 1)`.

The interpreter evaluates `x` to `1`, and thus builds up a low-level message data
for the call to `B.foo(x,1)` containing `B.foo()`'s, selector and the arguments
`(1, 1)`.  At this point we scan the EVM trace forward looking for the first
low-level observable event.

We verify that the encountered low-level observable event is a `CALL` instruction,
that the target address matches the address of the high-level call (`B`'s
address), and that two calls shave the same `msg.data` and value (see Fig 2).

![Debugging the First Segment](/sol-tooling/assets/images/first_segment_640.gif)

At this point we have established that the interpreter matches the *externally* observable behavior of the EVM trace up to the first call.
That is, we confirmed that the interpreter reaches the same external call as the EVM trace, and leaves the calling contract in the same state as the EVM trace before the call.

At this point, interpretation continues in the context of `B.foo()`. We
instantiate a new interpreter instance with the storage of `B` before the call,
an empty memory, and the message arguments and value from the end of the first
segment.  Again we run the interpreter in `B.foo()` until it reaches an
externally observable event - in this case the return from `B.foo()`. 

Next we scan forward in the EVM trace until we also hit an observable event. We
verify that the low-level event is a `RETURN` instruction, that the returned
data is the same between the interpreter and the EVM trace, and finally that the
storage of `B` is the same between the interpreter and the EVM trace at the
`RETURN` instruction (see Fig 3).

![Debugging the Second Segment](/sol-tooling/assets/images/second_segment_640.gif)

This establishes that the interpreter exactly matches the observable behavior of
the EVM up to the end of the second segment.  Finally we repeat the same steps
for the 3rd segment - continue interpreting in the original Interpreter instance
(i.e. in the context of `A.foo()`), using the returned data from `B.foo()`. We run
the interpreter until it hits the final observable event - the return from
`A.foo()`. Next we scan forward in the EVM trace until it hits an observable event,
and verify that it's a `RETURN` with the same return data as the interpreter. We also
verify that the final state of the `A` contract in the interpreter is the same
as the final state of `A` in the EVM trace (see Fig 4).

![Debugging the Third Segment](/sol-tooling/assets/images/third_segment_640.gif)

Together, all these checks establish that the Interpreter produced the same
observable behavior as the EVM trace. The core idea here is, that even if there
are minor difference in the execution of the interpreter and the EVM (e.g. the
compiled code optimizing away some redundant computations), this
check-and-verify process produces a *verifiably* correct explanation of what a
particular EVM trace does.

We will go more in depth in separate posts on both the design of the
interpreter as well as the exact algorithm for comparing the intermediate states
of interpretation with the EVM traces (we call this process *trace alignment*).
Crucial, we will explore in our future blog posts how we recover from cases
where the intermediate state does not match (e.g. due to an Out-of-Gas
exception), and still manage to continue interpreting, this covering much larger
parts of the trace.

# Evaluation

Our evaluation setup consisted of collecting 737 Ethereum Blocks at regular intervals starting from block 1150000 until block 24150001.
Of those blocks we extracted 101045 TXs.  

For each of those TXs we:

1. Obtained the expected storage of all contracts involved in the TX from Quicknode
2. Tried getting source code/compiler artifacts from Etherscan for each contract touched by the TX. For the contracts with source code we tried compiling it locally
3. For any segments in the TX, where we had successfully obtained a compilation artifact, we tried running the interpreter for those segments, and checked whether the interpreter produced the same observable behavior, and resulted in the same contract states as the EVM trace.

Note that at this level, there is one major language feature still not implemented - inline assembly. We left inline assembly as future work, due to the sheer amount of work it required just to implement the language interpreter. Based on this algorithm, and the limitation of a missing inline assembly implementations, trace segments in our experiment can be in one of several categories:

| Segment Type | Explanation |
| --- | --- | 
| No Source | Etherscan didn't have source code for this segment, or its source didn't compile locally, or it produced bytecode that diverged from the mainnet implementation [^6] |
| Aligned | This segment is part of a group of segments that were correctly replayed by the interpreter |
| Misaligned:Inline Assembly | This segment could not be replayed because it (or a previous segment) contained Inline Assembly |
| Misaligned:Low-level Exception | This segment could not be replayed because it (or a previous segment)contains a low-level exception (Out-of-Gas, Stack Overflow) |
| Misaligned:Error | This segment could not be replayed because it (or a previous segment) triggered a bug or unimplemented feature in our tooling |

Of the above categories, *No Source* is of no interest to us - there is nothing to do if we don't have sourcecode.

The *Misaligned:Inline Assembly* we consider out-of-scope right now. There is a clear path to resolving these, and they are just a matter of engineering effort.

The real interesting question for this approach is, how often do we encounter the remaining 2 categories - *Misaligned:Low-level Exception* and *Misaligned:Error*.

The first category is an event that we fundamentally can't do much about. At the solidity level its impossible to model the concepts of *gas* and *EVM stack size* (However we do handle the `gaselft()` function correctly! More on this in later posts ;)).
The second category measures remaining bugs in the current implementation.

So lets look at some ...

# Results

The 101045 TXs touched code in ~36K contracts, spanning all compiler versions from 0.4.13 to 0.8.29. The execution traces were split in a total of 971747 segments, with the following breakdown by segment type:

| Segment Type | Count |
| --- | --- | 
| No Source | 225407 |
| Aligned | 530505 |
| Misaligned:Inline Assembly | 215066 |
| Misaligned:Low-level Exception | 268 |
| Misaligned:Error | 501 |
| Total | 971747 |

As mentioned earlier, there is nothing to do for segments with no source code.
Furthermore inline assembly is not yet implemented as a feature, so for now we
consider those out-of-scope. There is a clear path for handling inline assembly.
The design decisions to use the same storage format as the EVM, and to use a
memory layout for objects that is also the same as the EVM enable a relatively
easy integration for inline assembly. Below we see the breakdown of all segments, as well as the breakdown of the segments of interest (i.e. ignoring no-source and inline assembly segments).


![Breakdown of All Segments](/sol-tooling/assets/images/breakdown_all_segments.jpg)

![Breakdown of All Segments](/sol-tooling/assets/images/breakdown_segments_of_interest.jpg)

The core takeaway here is, that our approach covers **99.8%** of all trace
segments for which we have source code, and which are not precluded by an inline
assembly block!

The above experiment demonstrates, that it is possible to **fully** and **verifiably**
explain the behavior of EVM traces at the source code level, for **arbitrary
optimized code** without even using source map information.

# Conclusion

This short article is only a brief overview of this work, that glosses over many
details. In the next few posts I will try to dig into more detail into the
design of the interpreter, the full algorithm for replaying EVM traces with the
interpreter and some of the interesting edge cases we observed (including known
compiler bugs!). I hope this was enough to keep you curious for more! In the meantime, check out the [code](https://github.com/d1m0/sol-interp/).

# Footnotes

[^1]: As 'in-scope' we considered any region of a transaction trace that a) has source code and b) is not precluded by an earlier inline-assembly block. Inline assembly is currently considered future work.

[^2]: Technically, the interpreter works over the JSON compiler artifacts emitted by solc. On top of the ASTs, it also uses the link and immutable references and the bytecode (for example to provide the `contract.code` builtin). Crucially, the interpreter doesn't look at source maps.

[^3]: On top of these, `GASLEFT` instructions are also technically an "observable event". This is to align calls to the `gasleft()` builtin with corresponding `GASLEFT` instructions, to allow the correct value for `gasleft()` to be returned in the interpreter. More on this in later posts.

[^4]: One interesting complication here is undefined behavior. For example the ordering of `a.foo() + b.foo()` is not strictly defined, so the compiler is free to re-order those. And for example we found differences in the ordering between the old codegen pipeline and the new IR pipeline. More on this in later posts.

[^6]: When re-compiling contracts locally, even when using the exact settings and artifact from Etherscan, we still sometimes ran into differences in the metadata hashes embedded in the bytecode. We implemented a more "fuzzy" match to account for this.