---
layout: post
title:  "Debuging Solidity at the Source-Level Without Debug Information"
date:   2026-07-23 08:48:33 -1000
categories: interpreter
---

# Introduction

This post is the first in a series exploring a new approach to source-level
debugging of Solidity contracts. Debugging is a core part of any language
ecosystem, but in the case of Solidity contracts it has been lacking. Our work
took an alternative approach to debugging, centered around interpretation, that
achieves promising results already. We successfully debugged 99.8% of the
in-scope[TODO] mainnet transactions across a sample of 100K TXs.

This first post will serve as a brief overview of our work. In the following
posts we will explore the design of the Solidity interpreter we've built, the
algorithmms for verifyable replay of EVM transactions using an interpreter, and
some interesing engineering challenges we've had to overcome.

# The Problem

For compiled languages, debugging relies on additional information emitted by
the compiler. This usually includes source maps (which map instructions back to
source locations), stack maps (which map low-level stack locations back to
source-level locals) and global symbol tables. Broadly, debugging information
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
[symbolic execution](TODO) and [heuristics](TODO) to infer as much of this
information as possible, but these are incomplete. More specifically they may
fail for optimized code with broken source maps.

Our core insight, is that instead of trying to map an EVM execution trace back
onto the source code, we can just **interpret** the source code, using the same
starting state as the low-level execution trace, and then **verify** that the
interpreter trace produces the same observable events as the low-level execution
trace, and the same final state. At the heart of this idea is a full Solidity
interpreter that interprets Solidity (ASTs)[TODO] but works over low-level EVM
state.

## Pros/Cons

This ideas has several pros/cons summarized below:

Pros:

1. This approach doesn't require additional debug informatition from the compiler
2. This approach produces a Solidity Interpreter as an additional artifact, that is independently useful

Cons:

1. Its a signifficant engineering effort to build and maintain a full interpreter for Solidity
2. This approach relies on the compiler not re-ordering "externally visible events"

A full language interpreter is essentially an additional implementation of the
language semantics. While it is a signifficant engineering effort (con), it is
useful as a second implementation against which we can differentially fuzz a
compiler. In fact, while replaying mainnet TXs we observed differences in
beavior between the interpreter and the compiled code that were due to bugs
later fixed in the compiler!
A source-level interpreter is also a useful template upon which we can build a
symbolic execution engine for Solidity ASTs.


# Motivating Example

To illustrate our work, lets consider the following sample Solidity code:

{% highlight solidity %}
contract A {
    B b;
    function foo(uint x) public returns (uint) {
        return b.foo(x, 1)
    }
}

contract B {
    function foo(uint a, uint b) public (returns uint) {
        return a + b;
    }
}
{% endhighlight %}