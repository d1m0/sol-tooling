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

# Quickstart

If you want to jump in and play around with the interpter, check out the sections on using it through the [CLI](https://github.com/d1m0/sol-interp#cli-usage) and [programatically](https://github.com/d1m0/sol-interp#programmatic-usage). That should let you start experimenting immediately!


# Interpreter

The core design choice of the interpreter is that it interprets **Solidity ASTs**, but works over low-level state very close to the EVM state itself (in fact, the interpreter storage is bitwise identical with EVM storage).

Most of the interpreter logic is contained within the
[`Interpreter`](https://github.com/d1m0/sol-interp/blob/main/src/interp/interp.ts)
class. The logic for decoding high-level solidity data types out of low-levle
storage/memory lives within the
[sol-dbg](https://github.com/consensysdiligence/sol-dbg) repo's
[`View`](https://github.com/ConsenSysDiligence/sol-dbg/blob/70a363a9c817e65f7af19dbba491db831f9e21ce/src/debug/decoding/view.ts#L32)
classes.

There is one `Interpreter` instance for each execution context (i.e. each external call to a contract). Generally the `Interpreter` doesn't hold any of the execution runtime state. Instead it only holds state mostly useful for debugging, or not specific to the current execution.

Most notably the `Interpeter` holds a reference to the current
[`EthereumEnvInterface`](https://d1m0.github.io/sol-interp/interfaces/EthereumEnvInterface.html)
and an
[`ArtifactManager`](https://github.com/ConsenSysDiligence/sol-dbg/blob/70a363a9c817e65f7af19dbba491db831f9e21ce/src/debug/artifact_manager/artifact_manager.ts#L173).

The `ArtifactManager` is a container that holds information for all the currently known contracts. For each contract it holds a compilation artifact for that contract. This includes the AST for that contract, creation and deployment bytecodes and link and immutable reference info.

# Ethereum Environment Interface

The [`EthereumEnvInterface`](https://d1m0.github.io/sol-interp/interfaces/EthereumEnvInterface.html) is the window for the Interpreter to interact with the world around it (i.e. the EVM). It currently provides the following callbacks:

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

`execMsg` allows an Interpreter to execute an external messaage (i.e. a call or
contract deployment). The `EthereumEnvInteface` instance is responsible for
establishing a new instance, or finding some other way to compute the result of
the call. This separation of responsibilities had the nice side effect, that the
`Interpreter` is completely agnostic of whether its being used in a purely
intepreted environment, or if its being used as part of replaying a trace. The
cruicial difference between the two, is that when replaying a trace, the target
of a call be a contract with or without source code. In the former case, the
`EthereumEnvInteface` will instantiate a new interpreter instance. In the latter case, it will
compute the correct return data, and apply any state changes by directly scanning the EVM trace.
Additionally the logic for comparing observable events between the interpeter and the EVM trace also lives 
entirely within the callbacks of the `EthereumEnvInteface`.

This provides a nice separation of responsibilities between `Interpreter` and
`EthereumEnvInterface` and makes the `Interpeter` more portable. 

`getAccount`, `setAccount` and `updateAccount` allow the Interpeter to query the world around it for the state of various accounts on the chain (including itself, after returning from a callback).

Finally `getBlock()` and `gasleft()` allow it to get the synonymous information from the chain. More details on `gasleft()` in future posts.

More on the `EthereumEnvInteface` in future posts.

# State

The relevant parts of the actual runtime state struct are shown below:

```typescript
/**
 * Interpreter runtime state
 */
export interface State {
    // Account info for the currently executing account
    account: AccountInfo;
    // Account info of actual code executing. Is defined only for delegate calls
    codeAccount: AccountInfo | undefined;
    // Scratch space for the deployed bytecode being created inside the constructor
    partialDeployedBytecode: Uint8Array | undefined;
    // The current memory
    memory: Memory;
    ...
    // The `SolMessage` for the current execution context
    msg: SolMessage;
    ...
    // Flag whether the current context is a STATICCALL (i.e. state is readonly)
    storageReadOnly: boolean;
    // Current block
    block: Block;
    // Current root TX
    tx: TypedTransaction;
    ...
}
```

## Storage

The current storage lives in the `account.storage` struct. Its type is
`ImmMap<bigint, Uint8Array>`, and it maps `keccak256(<store key>)` to the actual
word that lives there in storage. This map is bitwise exactly the same as the
storage of the EVM.


## Memory 

Similarly `memory` is a linear byte array - `Uint8Array`. For any individual
Solidity object (e.g. array, struct, string), its layout in interpreter memory
is bitwise identical to its layout in the compiled code. However the interpreter
may do allocations in a different order from compiled code, and additionally may
have different temporary allocations. As a result, while the interpreter memory
will hold many bitwise identical regions as compiled code memory, they will be
in a different order, interspersed with other mismatched temporary allocations.
As a result, interprter memory is different from compiled code memory.

The choise of the object layout being the same as with compiled code does have
the additional benefit, that it makes it possible to replay inline assembly
unchanged. While inline assembly can make assumptions about the layout of
individual objects, its similarly very risk for it to make assumptions about the
relative layouts of objects.

Additionally the first 4 32-bytes slots in interpreter memory have the same
[intended
purpose](https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html#layout-in-memory)
as they do in compiled code memory.

# Entrypoints