---
layout: post
title:  "Debuging Solidity at the Source-Level Without Debug Information"
date:   2026-07-23 08:48:33 -1000
categories: interpreter
---

# Introduction

This post is the first in a series exploring a new approach to source-level debugging of Solidity contracts. Debugging is a core part of any language ecosystem, but in the case of Solidity contracts it has faced some issues.
For compiled languages, debugging usually relies on additional debug information emitted by the compiler. This usually includes source maps (which map instructions back to source locations), stack maps (which map low-level stack locations back to source-level locals) and global symbol tables.

In the case of Solidity however, the compiler does not emit stack maps, and source maps have been [broken](TODO) for optimized source code. While there is ongoing work to [address](TODO) these issues in future versions of the compiler, due to the immutable nature of the Ethereum blockchain, code emitted by older compiler versions will be used in perpetuity and continue to interact with new code.

This leads us to the core problem: Its difficult to provide a debugging experience for optimized contracts already on the block chain, that gives full source-level info for local variables for examples.

There are [existing](TODO) approaches that attempt to use a combination of [symbolic execution](TODO) and [heuristics](TODO) to provide as much information as possible, but those are not complete.

{% highlight solidity %}
{% endhighlight %}
