---
layout: post
title:  "Undefined Behavior"
date:   2026-08-24 08:48:33 -1000
categories: interpreter
author: Dimitar Bounov
---

# The Undefined Behavior Issue

One major concern when trying to match compiler behavior with an interpreter is
undefined behavior (UB). The Solidity language semantics do not define the order
of evaluation of `e1 + e2` or `e1.foo(e2)`  for example.  Either `e1` or `e2`
can be evaluated first in each instance.  However in the interpreter we need to
pick an order for evaluation. This brings a potential
*risk* that the interpreter will just not match compiled programs because UB
allowed the compiler to changed the order during optimization.

# Why we *just* might be ok

The saving grace for this approach, is that when comparing an
interpreter trace with an EVM trace, I only care about the order of
*observable* events. This leads to the first hypothesis upon which this work rests:

> The compiler will emit code in some default order, and if that code involves *observable behaviors*
> the optimizer will not re-order them (even if it can leverage UB)

So for example given code like `a.foo() + b.foo()`, where
both expressions are external calls, the compiler would not freely re-order
those. This hypothesis restricts the effects of UB with respect to the observable events we care about.

This hypothesis comes from the general mental model that a compiler
first does a codegen pass, followed by multiple optimization and lowering
passes.  The hope is that there is some default order for code emission baked
in the code of the codegen pass, and that re-ordering due to UB happens mostly
in the optimization and lowering passes. In those passes, code that
involves externally observable events would hopefully not be re-ordered.

For most of the events that we discussed - calls, returns, exceptions, event
emission, I think this hypothesis makes sense.  Re-ordering those events changes the
external behavior of a contract, and most sane compiler implementations would
hopefully not do that. The internal `EVMGasLeft` event mentioned in a [previous
post](/sol-tooling/interpreter/2026/08/20/trace-alignment.html) is different, as its an internal event to a contract. 

The hope is that `GASLEFT` instructions would also not be re-ordered, as that
would change their return value. However that one is more up in the air. On the
bright side, code relying on `GASLEFT` order seems to also be more rare based on
my current experimentation over 100k transactions from mainnet. So even if there
are instances of such re-ordering, in practice they should happen *very* rarely.

Of course, even if the ordering of observable events w.r.t UB is stable across one compiler version,
there is still the potential issue of that ordering changing *across* compiler versions. So this led to the second hypothesis of this work:

> The compiler will tend to preserve the default[^1] ordering of code for snippets with UB across compiler versions.

Basically, my hope is that the compiler will not decide randomly from one version to the next, that the default codegen of `e1 + e2` should change from left-to-right to right-to-left.

Note that some change across versions **ARE OK**. The compiler version is
actually **included** along with the compilation artifact as an **input** to the
interpreter. And in fact the interpreter does have some behaviors conditional on
the compiler version. For example the scoping for locals changed between the 0.4.x
and 0.5.x versions of the language. Also the arguments and return values of the
`address.call()` family of builtins also changed between 0.4.x and 0.5.x. And
more fundamentally the behavior of arithmetic changed from 0.7.x to 0.8.x,
with arithmetic becoming checked by default.

These changes are a natural part of the language evolution, and the interpreter has to deal with them, since its
stated goal is to handle **all** Solidity language constructs from 0.4.13 onward.

However if something as fundamental as the ordering of binary operations or
argument/receiver evaluation would change often, this would be a rather annoying (but not impossible)
inconvenience to deal with.

So... 

# How did our hypotheses hold up?

During my evaluation I gathered more than 100K transactions from 737 blocks at
regular intervals from block 1150000 until block 24150001. Replaying those
required interpretation involving roughly 36K contracts spanning all versions of
Solidity from 0.4.13 to 0.8.29. This gave a very good sample of real world code
on which to gauge the validity of the hypothesis presented above.

Across all the replayed transaction, I did run into
[*one*](https://github.com/d1m0/sol-interp/issues/98) instance where the order
of evaluation changed between compilers.  Specifically the order of
evaluation between function arguments and function receiver did change. For
example during the replay of
[this](etherscan.io/tx/0x6fed55489bef3836a63f474f71f2191f4368af4f0a17a38bd5145dedd921a02a)
transaction I hit a mismatch between the interpreter and the EVM while
interpreting this snippet of the `yVault` contract at address
`0x9ca85572e6a3ebf24dedd195623f188735a5179f`:

```solidity
    function balance() public view returns (uint) {
        return token.balanceOf(address(this))
                .add(Controller(controller).balanceOf(address(token)));
    }
```

The top-level function call callee `token.balanceOf(address(this)).add` involves the static call `token.balanceOf(address(this))`. The arguments to the top-level call also involve a static call - `Controller(controller).balanceOf(address(token))`.
The interpreter evaluated first the receiver static call, while the EVM first evaluated the argument static call.

This was a bit weird, as in thousands of other instances, the EVM and the interpreter do match.

It turned out that the default order *did* change, but it was not a matter of
a compiler version, but a difference between the **old optimization
pipeline** and the new **ir pipeline** in the compiler.

The fix? Now we include which optimization pipeline is used as an input to the interpreter. The interpreter handles the ordering of function arguments and receivers conditionally based on the optimization pipeline.

Finally, given that the remaining mismatches account for less than `0.2%` of all
trace segments, and that I have already inspected many of those to be unrelated
to UB, it appears that UB is not a problem for the verifiable replay of mainnet
transactions.

# Conclusion

There are two reasons which allow this work to be resilient to UB:

1. We only care about the relative ordering of observable events, which the compiler seems reluctant to not mess with during optimization

2. With some exceptions (old vs ir pipeline), the default ordering of observable events during initial codegen seems stable across compiler versions from 0.4.13 to 0.8.29

Of course, there is not guarantee that future compiler versions would not more actively take advantage of UB or try to change the default evaluation order of codegen more often.
Should that come though, we have the recourse of making the interpreter evaluation order conditional for those specific versions as well.

# Footnotes

[^1]: Default here meaning "the ordering immediately after the codegen pass"