---
layout: post
title:  "Trace Alignment"
date:   2026-08-20 08:48:33 -1000
categories: interpreter
author: Dimitar Bounov
---

In previous posts I've given an
[overview](/sol-tooling/interpreter/2026/07/23/source-level-debugging-of-solidity-without-debug-info.html)
of this work and discussed the [interpreter
design](/sol-tooling/interpreter/2026/08/08/interpreter-design.html).  In this
post I will drill down into (1) how the interpreter runs for eligible[^1] segments
in an EVM trace, (2) the checks that interpreter produces the same sequence of
observable events as the EVM trace and (3) how it recovers from any
errors/misalignments. I refer to this process as *trace alignment*.

# Trace Alignment Algorithm

First I will provide pseudo code for the trace alignment algorithm. The core algorithm is implemented in the [`AlignedTraceBuilder`]() class as a callback-driven state machine. However, for ease of understanding I will
describe the trace alignment algorithm in an imperative fashion. Note, while the pseudocode broadly resembles `TypeScript`, I will be abusing the language syntax for brevity and clarity.
First lets first establish several definitions.

## Definitions

Lets define the possible states of the `AlignedTraceBuilder` algorithm as:

```
type AlignmentState = Aligned | Misaligned | NoSource
```

The algorithm iteratively scans the low-level EVM trace, and at each step is in one of the 3 states.
 - If its in an `Aligned` state, then we are running an interpreter for the current segment, that so far matches the low-level behavior.
 - If its in a `Misaligned` state, then there was an earlier failure of alignment in the same execution contract (i.e. the same call), due to which we are unable to run the interpreter for this trace segment.
 - If its in a `NoSource` state, then we don't have source code for the contract executing in the current trace segment.

 Next we can define the low-level observable events as:

 ```
type EVMObservableEvent =
    EVMCall(m: SolMessage) |
    EVMCreate(m: SolMessage) |
    EVMReturn(data: bytes) |
    EVMException(data: bytes) |
    EVMEmit(event: EventDesc)
 ```

 And the corresponding high-level observable events as:

 ```
type SolObservableEvent =
    SolCall(m: SolMessage) |
    SolCreate(m: SolMessage) |
    SolReturn(data: bytes) |
    SolException(data: bytes) |
    SolEmit(event: EventDesc)
 ```

[`SolMessage`](https://d1m0.github.io/sol-interp/classes/SolMessage.html) and
[`EventDesc`](https://d1m0.github.io/sol-interp/interfaces/_internal_.EventDesc.html)
are as described in the documentation.  Note that each pair of corresponding low
and high-level events (e.g. `EVMCall` and `SolCall`) have the exact same
payload. This makes it trivial to compare an `EVMObservableEvent` and a
`SolObservableEvent` for exact equality.

Finally the lets define the aligned data structure we are computing:

```
type AlignedTracePair = [[number, number], BaseStep[], AlignmentState]
type AlignedTrace = AlignedTracePair[]
```

Here [`BaseStep`](https://d1m0.github.io/sol-interp/classes/BaseStep.html) is the type of an `Interpreter` trace step. See the [documentation](https://d1m0.github.io/sol-interp/classes/BaseStep.html) for details.
The final `AlignedTrace` data type, is a list of triples, each tripple containing:

- `[number, number]` - a range (start, end) in the EVM trace
- `BaseStep[]` - a list of interpreter steps corresponding to the low-level steps in the range (may be empty)
- `AlignmentState` - the type of this aligned segment

All the entries in a `AlignedTrace` should completely cover the underlying EVM trace, and will include entries both for parts of the trace where there is no source code, and where we encountered a misalignment.

Finally, lets assume that we have the following helper functions:

- `eq(e1: EVMObservableEvent, e2: SolObservableEvent): bool` - returns True IFF the low-level event `e1` is equivalent to the high-level event `e2`. Equivalence means that they are of the same time (e.g. both calls, or both returns), and that they have the same payload.
- `interpretUntilSolObservableEvent(s: State): (State, SolObservableEvent, BaseStep[])` - runs the interpreter starting in the given state `s` until it hits a high-level observable event `e`. At that point it returns the final state `s1` (at the observable event `e`) the observable event `e` as well as a list of interpreter [`BaseStep`](https://d1m0.github.io/sol-interp/classes/BaseStep.html)s describing the execution
- `seekUntilEVMObservableEvent(trace: EVMStep[], i: number): (number,
  EVMObservableEvent)` - given an EVM `trace` (i.e. a sequence of
[`EVMStep`s](https://d1m0.github.io/sol-interp/interfaces/_internal_.EVMStep.html)).
Return the next step in the trace after `i` where there is an
`EVMObservableEvent`, as well as the event itself. Guaranteed to return
results as long as `i < trace.length - 1` since the end of the trace is either
an `EVMReturn` or `EVMException` event. Its illegal to call this with `i >=
trace.length - 1`
- `hasSource(step: EVMStep): bool` - returns True IFF we have source code for the contract executing at the given EVM step.
- `buildStateFromStep(step: EVMStep): State` - given an `EVMStep`, that is the first step in a new execution context, build interpreter state for it. This involves copying storage from the `EVMStep`, looking up the contract's compiler artifact in the `ArtifactManager`, initializing an empty memory,
building the correct `SolMessage` that lead to this call/deployment, initializing an empty internal call stack, etc.s
- `buildCallResultFromStep(step: EVMStep): CallResult` - given an `EVMStep` `step` that is the last one at the current execution context (i.e. an `RETURN`, `STOP` or an exception), compute the proper [`CallResult`](https://d1m0.github.io/sol-interp/interfaces/CallResult.html) that would be returned from the current step.


## Algorithm

The core algorithm is presented below in pseudocode:

```
// Align a trace starting at index start. Its expected that trace[start] is the first instruction in a 
// new execution context
align(trace: EVMStep[], start: number): (number, CallResult, AlignedTrace) {
    let i = start;
    let interpState: State
    let interp: Interpreter | undefined = undefined
    let alignedTrace: AlignedTrace = []
    let state: AlignmentState
    let interpTrace: BaseStep[]

    if (hasSource(trace[start])) {
        state = Aligned;
        interpState = buildStateFromStep(trace[start]);
        interp = new Interpreter(...);
    } else {
        state = NoSource;
    }

    let solEvent: SolObservableEvent
    let evmEvent: EVMObservableEvent

    while (i < trace.length) {
        (nextI, evmEvent) = seekUntilEVMObservableEvent(trace, i)
            
        if (state == Aligned) {
            (interpState, solEvent, interpTrace) = interpretUntilSolObservableEvent(interp, interpState)

            if (!eq(evmEvent, solEvent)) {
                state = Misalignment;
            }
        } else {
            interpTrace = []
        }

        alignedTrace.push([trace.slice(i, nextI + 1), interpTrace, state])

        if (evmEvent is EVMCall || evmEvent is EVMCreate) {
            (i, res, subAlignedTrace) = align(trace, i + 1);
            alignedTrace.push(...subAlignedTrace);
        } else if (evmEvent is EVMReturn || evmEvent is EVMException) {
            return (i, buildCallResultFromStep(trace[i]), alignedTrace);
        }
    }
}
```

# Implementation

# Examples

# Footnotes

[^1]: Any segment that (1) has source code info and (2) is not precluded by an earlier inline assembly block in the same execution context is considered eligible.
Inline assembly is not yet implemented, and left for future work, which is why blocks precluded by it are considered out-of-scope. There is a clear implementation path for inline assembly.
