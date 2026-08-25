---
layout: post
title:  "Interpreter Design"
date:   2026-08-08 08:48:33 -1000
categories: interpreter
author: Dimitar Bounov
---

In an earlier
[post](/sol-tooling/interpreter/2026/07/23/source-level-debugging-of-solidity-without-debug-info.html)
I gave a brief overview of this project. Now its time to drill more in depth in
the design of the [interpreter](https://d1m0.github.io/sol-interp/classes/Interpreter.html) itself.

This post will be of interest to people who want to hack on this project. If this is not your cup of tea, feel free to skip this one.

# Quickstart

If you want to jump in and play around with the interpreter, check out the README sections on using it through the [CLI](https://github.com/d1m0/sol-interp#cli-usage) and [programmatically](https://github.com/d1m0/sol-interp#programmatic-usage). That should let you start experimenting immediately!


# Interpreter

The interpreter 'interprets' **Solidity ASTs**, but works over **low-level state** very close to the EVM state itself (in fact, the interpreter storage is bitwise identical with EVM storage).

Most of the interpreter logic is contained within the
[`Interpreter`](https://github.com/d1m0/sol-interp/blob/main/src/interp/interp.ts)
class. The logic for decoding high-level solidity data types out of low-level
storage/memory lives within the
[sol-dbg](https://github.com/consensysdiligence/sol-dbg) library's
[`View`](https://github.com/ConsenSysDiligence/sol-dbg/blob/70a363a9c817e65f7af19dbba491db831f9e21ce/src/debug/decoding/view.ts#L32)
class hierarchy.

There is one `Interpreter` instance for each execution context (i.e. each external call to a contract). Generally the `Interpreter` class doesn't own any of the execution runtime state. It only holds references to:

 - [`EthereumEnvInterface`](https://d1m0.github.io/sol-interp/interfaces/EthereumEnvInterface.html) - the interpreter's interface to the blockchain
 - [`ArtifactManager`](https://github.com/ConsenSysDiligence/sol-dbg/blob/70a363a9c817e65f7af19dbba491db831f9e21ce/src/debug/artifact_manager/artifact_manager.ts#L173) - a container for the set of known contracts

The `ArtifactManager` is crucial for interpretation. It holds information for all the currently known contracts. For each contract it stores the corresponding compilation artifact, including the AST for that contract, creation and deployment bytecodes and linking and immutables reference info. The actual runtime state is passed around as an argument to Interpreter methods, as we shall see next.

# Entrypoints

The main Interpreter entrypoints are the [`call`](https://d1m0.github.io/sol-interp/classes/Interpreter.html#call) and [`create`](https://d1m0.github.io/sol-interp/classes/Interpreter.html#create) methods, which respectively execute a call to a contract, or deploy a new contract.
These methods repeatedly invoke the [`exec`](https://d1m0.github.io/sol-interp/classes/Interpreter.html#exec) method to execute individual [`Statement`](https://consensysdiligence.github.io/solc-typed-ast/classes/Statement.html)s, which itself will invoke the [`eval`](https://d1m0.github.io/sol-interp/classes/Interpreter.html#eval) method to evaluate individual [`Expressions`](https://consensysdiligence.github.io/solc-typed-ast/classes/Expression.html) within statements.

Note that all of the above methods expect a [`State`](https://d1m0.github.io/sol-interp/interfaces/_internal_.State.html) object as their last argument. All methods (including `eval` - expressions are impure in Solidity) will destructively modify the given `State`.

The `exec()` method returns a single [enum](https://d1m0.github.io/sol-interp/enums/_internal_.ControlFlow.html), determining how control flow proceeds from this statement. The `eval` method returns the [`Value`](https://d1m0.github.io/sol-interp/types/_internal_.Value.html) to which the given expression evaluates in the current state.

While you *can* interact directly with the `Interpreter`, the supported way to
use it is to instead create an instance of `EthereumEnvInterface` (e.g. you
could use the
[`BaseEEI`](https://d1m0.github.io/sol-interp/classes/BaseEEI.html) class) and
invoke its `execMsg()` method. The `execMsg()` method is
responsible for instantiating an `Interpreter` and calling its expected
entrypoint. If that interpreter instance needs to make external calls of its
own, it will recursively call back into the `EthereumEnvInterface`, which may
further instantiate new `Interpreter` instances, or do something else (as we
will see in future posts on trace alignment).

Next up, lets look at...

# State

The more important parts of the interpreter runtime state struct are shown below:

```typescript
export interface State {
    // Account info for the currently executing account
    account: AccountInfo;
    // Account info of actual code executing. Is defined only for delegate calls
    codeAccount: AccountInfo | undefined;
    // Scratch space for the deployed bytecode being created inside the constructor
    partialDeployedBytecode: Uint8Array | undefined;
    // The current memory
    memory: Memory;
    // The current memory allocator. The default allocator works the same way as Solidity compiled allocation as of 0.8.29
    memAllocator: Allocator;
    // The `SolMessage` for the current execution context
    msg: SolMessage;
    // Internal call-stack. Contains the values of all local variables on the call stack
    intCallStack: InternalCallFrame[];
    // Current  *syntactic* scope being evaluated. Each scope contains a pointer to its parent scope for symbol resolution
    scope: BaseScope | undefined;
    // Flag whether the current context is a STATICCALL (i.e. state is readonly)
    storageReadOnly: boolean;
    // Current block
    block: Block;
    // Current root TX
    tx: TypedTransaction;
}
```

`account` contains a reference to the [`AccountInfo`](https://d1m0.github.io/sol-interp/interfaces/AccountInfo.html) of the currently executing
contract. This includes the account's balance, nonce and storage. Additionally,
it will contain a reference to the contract compilation artifact, if we have
source code for this account.  In the case of `DELEGATECALL`s, the actual
running code is different from the account whose storage we are using. For
those cases `codeAccount` will contain the `AccountInfo` of the actual running
code.

The current storage lives in the `account.storage` struct. Its type is
`ImmMap<bigint, Uint8Array>`, and it maps `keccak256(<store key>)` to the actual
word that lives there in storage. This map is bitwise exactly the same as the
storage of the EVM.

`partialDeployedBytecode` is a field only used during contract deployment to
support `immutable` state variables. Immutable state variables are set 
only during contract deployment, and their values are embedded in the deployed
bytecode. In the interpreter I follow this exact convention as well, and as
such, during deployment I incrementally encode the values of the immutable state
vars, as they become available, into the final deployed bytecode. `partialDeployedBytecode` is the storage
space for this incremental computation. This design decision has the added bonus
that the interpreter's `create` method returns the exact deployed bytecode we
would expect to see on the block chain.

`memory` is a linear byte array (`Uint8Array`). For any individual
Solidity object (e.g. array, struct, string), its layout in interpreter memory
is bitwise identical to its layout in the compiled code. However the interpreter
may do allocations in a different order from compiled code, and additionally may
have different temporary allocations. As a result, while the interpreter memory
will hold many bitwise identical regions as compiled code memory, they will be
in a different order, interspersed with other mismatched temporary allocations.
As a result, interpreter memory is different from compiled code memory.

The choice of the object layout being the same as with compiled code does have
the additional benefit, that it makes it possible to replay inline assembly
unchanged. While inline assembly can make assumptions about the layout of
individual objects, it cannot make assumptions about the
relative layouts of objects in different allocation blocks.

Additionally the first 4 32-bytes slots in interpreter memory have the same
[intended
purpose](https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html#layout-in-memory)
as they do in compiled code memory.

`memoryAllocator` contains an instance of `Allocator` responsible for growing and allocating chunks of memory.
We currently use the `DefaultAllocator` class, which matches the Solidity behavior of incrementing a free pointer at 0x40.

`msg` contains the [`SolMessage`](https://d1m0.github.io/sol-interp/classes/SolMessage.html) that gave rise to the current execution context.
It includes the raw `msg.data`, the value being sent and other expected fields
(sender, recipient, nonce, etc.)

`intCallStack` contains a list of [`InternalCallFrame`](https://d1m0.github.io/sol-interp/interfaces/_internal_.InternalCallFrame-1.html)s. As the name implies,
each contains the info for a single call frame.  This includes a reference to
the currently executing context (e.g. `FunctionDefinition` for a normal function call) and a `LocalScope`
object, which contains the arguments to this function call.
`InternalCallFrame`s also keep track of the currently executing modifier (if
any). Note that modifiers have their own argument and locals 'scopes', separate
from the arguments scope of the function call they are a part of.

`scope` is the head of a linked-list of [syntactic scopes](https://d1m0.github.io/sol-interp/classes/_internal_.BaseScope.html) used for identifier resolution. Note that they are different from the list internal call frames, as they are a syntactic construct.
There are several types of scopes - (1) [functions scopes/block scopes](https://d1m0.github.io/sol-interp/classes/_internal_.LocalsScope.html), (2) contracts scopes (containing state variables), (3) global scope (global constants, global variables, contracts, etc) as well as the (4) builtin scope.
For more info read through the code in [scope.ts](https://github.com/d1m0/sol-interp/blob/main/src/interp/scope.ts).

`storageReadOnly` is a flag set when we are executing in a `STATICCALL` context. Its used to determine if we should trap on storage modification in the given context.

`block` and `tx` hold references to the current [`Block`](https://github.com/ethereumjs/ethereumjs-monorepo/blob/master/packages/block/docs/classes/Block.md) and [`TypedTransaction`](https://github.com/ethereumjs/ethereumjs-monorepo/blob/master/packages/tx/docs/type-aliases/TypedTransaction.md) respectively.

# Ethereum Environment Interface

As mentioned earlier, the interpreter is meant to be invoked through an instance
of
[`EthereumEnvInterface`](https://d1m0.github.io/sol-interp/interfaces/EthereumEnvInterface.html).
The
[`EthereumEnvInterface`](https://d1m0.github.io/sol-interp/interfaces/EthereumEnvInterface.html)
is the window for the Interpreter to interact with the world around it (i.e. the
blockchain). The two have a mutually recursive relation. The
[`EthereumEnvInterface`](https://d1m0.github.io/sol-interp/interfaces/EthereumEnvInterface.html)
creates a new `Interpreter` instance for each call to `execMsg`.
`Interpreter` instances all hold a reference back to the parent
`EthereumEnvInterface`. In their turn, `Interpreter` instances call back to the
`EthereumEnvInterface` whenever they need to interact with the blockchain,
including to make new calls using `execMsg`, which would recursively spawn more instances, and so on.

We currently provide 2 implementations of `EthereumEnvInterface`:

- [`BaseEEI`](https://d1m0.github.io/sol-interp/classes/BaseEEI.html) provides an environment with a single block and a single transaction, in which only interpreted contracts exist. It is useful as a playground for interpreting several contracts with source code
- [AlignedTraceBuilder](https://d1m0.github.io/sol-interp/classes/AlignedTraceBuilder.html) is a more elaborate environment. It contains a single EVM execution trace, along with a set of known contract source codes. The `AlignedTraceBuilder` finds all segments in the trace that 
correspond to contracts with known source codes, and for those segments launches interpreter instances that can interpret the original source for that segment. We will go in depth in the behavior of `AlignedTraceBuilder` in the next post.

`EthereumEnvInterface` currently provides the following callbacks:

```typescript
export interface EthereumEnvInterface {
    execMsg(msg: SolMessage): CallResult;
    getAccount(address: string | Address): AccountInfo | undefined;
    setAccount(address: string | Address, account: AccountInfo): void;
    updateAccount(account: AccountInfo): void;
    getBlock(number: bigint): Block | undefined;
    gasleft(): bigint;
}
```

`execMsg` allows an Interpreter to execute an external message (i.e. a call or
contract deployment). The `EthereumEnvInterface` instance is responsible for
establishing a new interpreter instance, or finding some other way to compute the result of
the call. This separation of responsibilities between the `Interpreter` and `EthereumEnvInterface` has the nice side effect, that the
`Interpreter` is completely agnostic of whether its being used in a purely
interpreted environment, or if its being used to replay an EVM trace. The
crucial difference between the two, is that when replaying a trace, the target
of a call may be a contract without source code. In that case the `EthereumEnvInterface` will compute the correct return data by scanning through the
trace to find a matching `RETURN` instruction. This complexity is entirely
contained in the `EthereumEnvInterface` methods, leaving the `Interpreter`
with a much simpler implementation that only cares for Solidity semantics.

`getAccount` allows the Interpreter to query the block chain for the state of
any account at the moment. This is especially useful when returning from a call
to another contract, as the current interpreter instance needs to know if its
own storage has been modified by a callback.

`setAccount` and `updateAccount` allow the Interpreter to notify the blockchain
of changes it has made to accounts on the block chain. This includes changes to
both balances and storage. These are usually invoked at points before execution
leaves the current interpreter instance - (1) before we make an external call or
(2) before we return from the current context.

Finally `getBlock()` and `gasleft()` respectively return information about a given block on the chain and about the amount of gas left for the current execution.
For purely interpreted contracts we don't know the amount of gas left, and as such `BaseEEI.gasleft()` throws an exception.
However when we align to a real trace, it turns out we *can* determine the correct return value of `gasleft()`, and we do so in `AlignedTraceBuilder.gasleft()`. More on this in the next post.

# Example

To tie everything together, lets consider the example from the [first post](/sol-tooling/interpreter/2026/07/23/source-level-debugging-of-solidity-without-debug-info.html) again:

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

Lets trace the sequence of calls between the [`BaseEEI`](https://d1m0.github.io/sol-interp/classes/BaseEEI.html) and the [Interpreter](https://d1m0.github.io/sol-interp/classes/Interpreter.html) that happen when we interpret a call to `A.foo(...)`:

1. First we invoke `BaseEEI.execMsg(m)` with a `SolMessage` `m` that contains the address of the contract A, and a `msg.data` corresponding to the ABI encoded call to `A.foo(...)`.

2. `BaseEEI.execMsg(m)` creates a new [Interpreter](https://d1m0.github.io/sol-interp/classes/Interpreter.html) instance - `i0` and creates a `State` object `s` corresponding to the state right before a call to `A.foo(...)`

3. `BaseEEI.execMsg(m)` invokes `i0.call(s)` where `s` is the [`State`](https://d1m0.github.io/sol-interp/interfaces/_internal_.State.html) created in the previous step.

4. `i0.call(s)` (through its helpers) ends up calling `Interpreter.exec(...)` for each of the statements in `A.foo(...)` and `Interpreter.eval(...)` for each expression inside.

5. When executing `Interpreter.eval` on the external call `b.foo(x+1, 1)` inside of `A.foo()`, the Interpreter `i0` will:

    5.1 Call `BaseEEI.updateAccount(this, state.account)` to update the state of `A`'s account on the blockchain with any changes that occurred during the interpreter execution so far (e.g. storage writes).

    5.2 Create a new `SolMessage` object `m1` corresponding to the call to `B.foo(x+1, 1)`

    5.3 Invoke `BaseEEI.execMsg(m1)`


6. `BaseEEI.execMsg(m1)` will create a new interpreter instance `i1` and a new `State` object `s1` corresponding to the state of `B` right before the call to `B.foo(...)`

7. `BaseEEI.execMsg(m1)` will invoke `i1.call(s1)`

8. `i1.call(s1)` will interpret the call to `B.foo()` (by repeatedly invoking `Interpreter.exec()` and `Interpreter.eval()`)

9. After `i1` is done interpreting `B.foo()` it will:

    9.1 Call `BaseEEI.updateAccount(this, state.account)` to update the state of `B` on the blockchain with all the changes that happened to `B` during the interpretation of `B.foo()`

    9.2 Execution returns from `i1.call(s1)` and `BaseEEI.execMsg(m1)`, returning the return data for `B.foo(...)` in a [`CallResult`](https://d1m0.github.io/sol-interp/interfaces/CallResult.html) object

10. Back in `i0.eval(...)`, as soon as we return from `execMsg` we invoke
`BaseEEI.getAccount(this)` to re-load the state of `A`'s account from the
blockchain. We need to do this upon returning from external calls to other
contracts, as callbacks into ourselves may have resulted in modifications to our
own state.  Thus the interpreter needs to update the state of the executing
contract before resuming.

11. `i0` continues interpretation until `A.foo()` is complete, finally returning
the execution result. Again before returning we update the state of `A` on the
blockchain by invoking `BaseEEI.updateAccount(this, state.account)`. The call
result gets propagated back through `BaseEEI.execMsg(m)` to the caller as a
[`CallResult`](https://d1m0.github.io/sol-interp/interfaces/CallResult.html)
object

Note the use of `updateAccount` and `getAccount` to communicate state changes between the interpreter and the `EthereumEnvInterface` object.

# Conclusion

This post gives a brief introduction of the central objects and data structures involved in interpretation and their interaction.
This is meant to serve as a high-level map for anyone interested in hacking on the code base itself.
In our next post, we will look at how trace alignment works.