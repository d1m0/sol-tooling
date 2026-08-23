---
layout: post
title:  "Trace Alignment"
date:   2026-08-20 08:48:33 -1000
categories: interpreter
author: Dimitar Bounov
---

In previous posts I gave an
[overview](/sol-tooling/interpreter/2026/07/23/source-level-debugging-of-solidity-without-debug-info.html)
of this work and discussed the [interpreter
design](/sol-tooling/interpreter/2026/08/08/interpreter-design.html).  In this
post I will drill down into (1) how the interpreter runs for eligible[^1] segments
in an EVM trace, (2) the verification that the interpreter produces the same sequence of
observable events as the EVM trace and (3) how it recovers from any
errors/misalignments. I refer to this process as *trace alignment*.

# Trace Alignment Algorithm

First I will provide pseudo code for the trace alignment algorithm. The core
algorithm is implemented in the [`AlignedTraceBuilder`](https://d1m0.github.io/sol-interp/classes/AlignedTraceBuilder.html) class as a
callback-driven state machine. However, for ease of understanding here I will
describe the algorithm in an imperative fashion. Note that while the pseudocode
broadly resembles TypeScript, I will be abusing the language syntax for brevity
and clarity. First lets establish several definitions.

## Definitions

Lets define the possible states of the `AlignedTraceBuilder` algorithm as:

```
type AlignmentState = Aligned | Misaligned | NoSource
```

The algorithm iteratively scans the low-level EVM trace, and at each step is in one of the 3 states.
 - In the `Aligned` state we are running an interpreter for the current segment and it so far matches the EVM behavior.
 - In the `Misaligned` state there must have been an earlier failure of alignment in the same execution context (i.e. the same external call depth). Due to this the interpreter cannot run for the current trace segment.
 - In the `NoSource` state there is no source code for the current contract (so the interpreter cannot run for the current trace segment).

 Next we can define the low-level observable events as:

 ```
type EVMObservableEvent =
    EVMCall(m: SolMessage) |
    EVMCreate(m: SolMessage) |
    EVMReturn(data: Uint8Array) |
    EVMException(data: Uint8Array) |
    EVMEmit(event: EventDesc) |
 ```

 And the corresponding high-level observable events as:

 ```
type SolObservableEvent =
    SolCall(m: SolMessage) |
    SolCreate(m: SolMessage) |
    SolReturn(data: Uint8Array) |
    SolException(data: Uint8Array) |
    SolEmit(event: EventDesc)
 ```

[`SolMessage`](https://d1m0.github.io/sol-interp/classes/SolMessage.html) and
[`EventDesc`](https://d1m0.github.io/sol-interp/interfaces/_internal_.EventDesc.html)
are as described in the documentation.

Trace alignment seeks to match up `EVMObservableEvent`s in the low-level trace
with `SolObservableEvent`s in the interpreter trace. Thus low-level and high
level events are matched first by type (e.g. an `EVMCall` should be matched
with an `SolCall`). Corresponding low-level and high-level events (e.g.
`EVMCall` and `SolCall`) have the exact same payload. This makes it trivial to
compare an `EVMObservableEvent` and a `SolObservableEvent` for exact equality.

The output of the alignment process is an aligned trace, defined below:

```
type AlignedTracePair = [[number, number], BaseStep[], AlignmentState]
type AlignedTrace = AlignedTracePair[]
```

Here [`BaseStep`](https://d1m0.github.io/sol-interp/classes/BaseStep.html) is the type of an `Interpreter` trace step. See the [documentation](https://d1m0.github.io/sol-interp/classes/BaseStep.html) for details.
The final `AlignedTrace` data type, is a list of triples, each tripple containing:

- `[number, number]` - a range (start, end) in the EVM trace
- `BaseStep[]` - a list of interpreter steps corresponding to the low-level steps in the range (may be empty)
- `AlignmentState` - the type of this aligned segment (Aligned, Misaligned, NoSource)

The ranges in an `AlignedTrace` exactly cover the underlying EVM trace, and will include tripples both for parts of the trace where there is no source code, and where we encountered a misalignment.

Finally, lets assume that we have the following helper functions:

- `eq(e1: EVMObservableEvent, e2: SolObservableEvent, state: State, step:
  EVMStep): bool` - returns True IFF the low-level event `e1` is equivalent to
the high-level event `e2`. Equivalence means that they are of the same type
(e.g. both calls), and that they have the same payload. Also for call, create and return event pairs we check that the interpreter storage is the same as the storage at the current evm step.
- `interpretUntilSolObservableEvent(s: State): (State, SolObservableEvent, BaseStep[])` - run the interpreter starting in the given state `s` until it hits a high-level observable event `e`. At that point return the final state of the interpreter, the observable event `e` as well as a list of interpreter [`BaseStep`](https://d1m0.github.io/sol-interp/classes/BaseStep.html)s describing the execution
- `seekUntilEVMObservableEvent(trace: EVMStep[], i: number): (number,
  EVMObservableEvent)` - find the first step in the trace `trace`
after `i`, where an `EVMObservableEvent` occurs. Return the index of
the step, as well as the event itself.  <!--Guaranteed to return results as long as
`i < trace.length - 1` since the end of the trace is either an `EVMReturn` or
`EVMException` event. Its illegal to call this with `i >= trace.length - 1` -->
- `hasSource(step: EVMStep): bool` - returns True IFF we have source code for the contract executing at the given EVM step.
- `buildStateFromStep(step: EVMStep): State` - given the first evm `step` in a
  new execution context, build interpreter state from it. This involves copying
storage from the EVM, looking up the contract's compiler artifact in the
`ArtifactManager`, initializing an empty memory, building the correct
`SolMessage` that lead to this call/deployment, initializing an empty internal
call stack, etc.
- `buildCallResultFromStep(step: EVMStep): CallResult` - given the last evm `step` in the current execution context (i.e. a `RETURN`, `STOP` or an exception), compute the proper [`CallResult`](https://d1m0.github.io/sol-interp/interfaces/CallResult.html) that would be returned.

## Algorithm

The core algorithm is presented below in pseudocode:

```
// Align a trace starting at index start. Its expected that trace[start] is the
// first instruction in a new execution context
align(trace: EVMStep[], start: number): (number, CallResult, AlignedTrace) {
    let i = start;
    let interpState: State
    let alignedTrace: AlignedTrace = []
    let state: AlignmentState
    let interpTrace: BaseStep[]

    if (hasSource(trace[start])) {
        state = Aligned;
        interpState = buildStateFromStep(trace[start]);
    } else {
        state = NoSource;
    }

    let solEvent: SolObservableEvent
    let evmEvent: EVMObservableEvent

    while (i < trace.length) {
        (nextI, evmEvent) = seekUntilEVMObservableEvent(trace, i)
            
        if (state == Aligned) {
            (interpState, solEvent, interpTrace) = interpretUntilSolObservableEvent(interpState)

            if (!eq(evmEvent, solEvent, interpState, trace[i])) {
                state = Misalignment;
            }
        } else {
            interpTrace = []
        }

        alignedTrace.push([trace.slice(i, nextI + 1), interpTrace, state])
        // At this point trace[i] is the next observable event
        i = nextI

        if (evmEvent is EVMCall || evmEvent is EVMCreate) {
            (i, res, subAlignedTrace) = align(trace, i + 1);
            alignedTrace.push(...subAlignedTrace);
        } else if (evmEvent is EVMReturn || evmEvent is EVMException) {
            return (i, buildCallResultFromStep(trace[i]), alignedTrace);
        }

        // Increment i as the current observable event is already at the end of the last segment
        i++
    }
}
```

The `align` function is recusrively invoked for every execution context in the
trace. It iteratively scans the low-level trace forward looking for
`EVMObservableEvent`s. For each event found, it also runs the interpreter (if
we have source code and haven't hit a misalignment already) until it hits an
observable event, and then it verifies that the events are the same.  The
equality check function `eq` also checks that storage is the same between
interpreter and the evm at call, create and return exception boundaries. We do
allow storage to diverge right before an exception is thrown, since all storage
changes are reverted. However we still check that the exceptions are identical.

# Misalignment Recovery 

One nice property of the current alignment algorithm is that any misalignments
only affect trace segments in the current execution context. Any segments in
execution contexts at different call depths are not affected.

For example consider a contract `A` with mehod foo calling
another contract's `B`'s `foo` method, where `B.foo()` runs out of gas.
In Fig. 1 below, on the bottom axis we have the EVM trace, with signifficant observable events marked.
Above it, we have a visiual representation of the interpreter's execution of the code.

Notably, while the EVM trace runs out of gas in the body of `B.foo()`,
the interpreter returns successfully from `B.foo()`. This leads to a misalignment (marked with
the dotted red line in Fig 1.). The `align` procedure would thus add a misaligned pair for the segment (2) of
the trace. However, since the misalignment happened in a separate execution
context (the context of `B.foo()`), alignment continues running the interpreter in the rest
of `A.foo()` and we add an aligned pair for segment (3). This way we covered
more of the trace, instead of giving up at the first failure.

![Misalignment Callee](/sol-tooling/assets/images/alignment_oog.jpg)

When resuming interpretation in a caller context, after a misalignment in a callee (e.g. when resuming `A.foo()` after the call to `B`), it is important to capture any state changes that may have
happened after the misalignment.  Luckily, since the interpreter already
re-loads its storage from the `EthereumEnvInterface` upon returning from a
call, the `AlignedTracesBuilder` already gets the correct storage directly from
the EVM trace at the point of resuming. In essence, the resuming interpreter
maintains its memory and locals state from before the call, but just reloads
its storage from the EVM trace.

We can similarly cover more of a trace when we encounter call to a new context
after a misalignment. In Fig 2 we have a call to `B.foo()` from an execution
context that is alread misaligned.  Since this is a new execution context, it
will have its own interpreter instantiated, with state built from the first EVM
step of the segment.  This way we will successfully build an aligned trace pair
for the segment of `B.foo()` - (2), even though the 2 surrounding segments -
(1) and (3) are both misaligned.

![Call From Misaligned](/sol-tooling/assets/images/call_from_misaligned.jpg)

Being able to recover from misalignment or missing source info is cruicial for
the performance of the alignment algorithm. In mainnet transactions execution
oftens starts in, or passes through upgradeable proxy contracts for example.
Most of those utilize inline assembly - thus resulting in barriers to
interpretation (currently). Furthermore, contracts without source are also
plentiful. Thus, interruptions in interpretation are actually the norm in the
real world, even ignoring misalignments due to Out-of-Gas or bugs.

The ability to recover allowed the trace alignment to cover **99.8%** of trace
segments, for which we have source code, and which are not precluded by an
earlier inline asembly block. The remaining **0.2%** were misalignments due to
either Out-of-Gas exceptions, compiler bugs, or bugs in the interpreter.

# Implementation

The alignment algorithm is presented here in an imperative form, as a single
function, for clarity. The actual implementation is a call-back driven state
machine living in
[`AlignedTracesBuilder`](https://github.com/d1m0/sol-interp/blob/e958db4e9bb7cba28c4fbd0e77ffd4d0aaeca94c/src/alignment/trace_builder.ts#L77)
class. The bulk of the implementation is in the
[`AlignedTracesBuilder.execMsg`](https://github.com/d1m0/sol-interp/blob/e958db4e9bb7cba28c4fbd0e77ffd4d0aaeca94c/src/alignment/trace_builder.ts#L316)
method.

# Gasleft

Another neat trick that alignment allows is to support the `gasleft()`
Solidity builtin. Obviously when we are strictly interpreting we cannot talk
about gas left at the Solidity level.

However when aligning traces, whenever we encounter a `gasleft()` builtin we
*also* have a corresponding EVM trace segment.  Thus, we can scan linearly
through that segment until we hit a `GASLEFT` instruction, and get the correct
value from the trace itself! However what if there are many invocations of `GASLEFT` in a single segment?

To support those, we have another internal observable event pair -
`EVMGasLeft/SolGasLeft` - so every pair of `gasleft()` builtin/`GASLEFT` opcode
become their own small aligned trace pair boundary. Thus a segment with many `GASLEFT` opcodes is broken down into multiple smaller segments.

That code was actually exercised nicely during the replay of Tx [0xbb962e806709a99909a7b534d16c6e6bea0dff044e09c98ebf74ae93d1a77056](https://etherscan.io/tx/0xbb962e806709a99909a7b534d16c6e6bea0dff044e09c98ebf74ae93d1a77056). Part of that function involved calling the following function to burn a specified amount of gas:

```solidity


library Burn {
...
    /// @notice Burns a given amount of gas.
    /// @param _amount Amount of gas to burn.
    function gas(uint256 _amount) internal view {
        uint256 i = 0;
        uint256 initialGas = gasleft();
        while (initialGas - gasleft() < _amount) {
            ++i;
        }
    }
...
}
```

Alignment correctly handled this function, matcing the exact amount of calls ot
`gasleft()` with the correct `GASLEFT` instructions in the segment.

An astute reader may note here that the compiler inserts implicit `GASLEFT`
instructions before (some) external calls. However, since the matching
algorithm is bounded in a single segment, that may have only one external
call at the end, all this means is that during matching there may be at most 1 extra `GASLEFT`
opcode at the end of the segment, which doesn't impact it.

Should the compiler ever insert additional implicit `GASLEFT`, or optimze some
away for some reason, this would result in a misalignment, which as was shown
we can easily recovers from.  During the last evaluation run, with over 100k
transactions replayed, I have not run into misalignments due to mismatched
`GASLEFT` opcodes.

# Footnotes

[^1]: Any segment that (1) has source code info and (2) is not precluded by an earlier inline assembly block in the same execution context is considered eligible.
Inline assembly is not yet implemented, and left for future work, which is why blocks precluded by it are considered out-of-scope. There is a clear implementation path for inline assembly.
