# Implementation notes

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [VM Considerations](#vm-considerations)
- [WASM](#wasm)
  - [Interpreters](#interpreters)
  - [Floating Point Operations](#floating-point-operations)
  - [32bit or 64bit](#32bit-or-64bit)
  - [Limits](#limits)
- [IPLD Memory Model](#ipld-memory-model)
  - [State Transaction](#state-transaction)
    - [Options 1 & 2](#options-1--2)
    - [Option 3 (selectors)](#option-3-selectors)
  - [Security Considerations](#security-considerations)
- [Global State](#global-state)
- [Calling Convention](#calling-convention)
  - [Sending Data](#sending-data)
  - [EVM Comparison & Tradeoffs](#evm-comparison--tradeoffs)
    - [Method Dispatch](#method-dispatch)
    - [Delegate Call/Call Code](#delegate-callcall-code)
- [EVM Contracts](#evm-contracts)
  - [Multiple VMs](#multiple-vms)
  - [AOT Solidity Compiler](#aot-solidity-compiler)
  - [AOT EVM Transpiler](#aot-evm-transpiler)
  - [EVM Emulation (or JIT)](#evm-emulation-or-jit)
    - [Deploy](#deploy)
    - [JIT](#jit)
- [Gas Accounting](#gas-accounting)
- [Runtime](#runtime)
  - [Static Environment](#static-environment)
  - [Allocation & Variable Sizes](#allocation--variable-sizes)
- [Built-in Actors](#built-in-actors)
  - [Cron](#cron)
- [Resources](#resources)
  - [Tools](#tools)
  - [Rust → WASM](#rust-%E2%86%92-wasm)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# VM Considerations

While we've [considered](https://docs.google.com/document/d/1sUl0uxebpY8mDse24a4WBzwVYX0DgDW8mNQD0bbTaqI/edit) alternative VM implementations, we're currently leaning towards WASM. This section dives into VM-related design considerations.

# WASM VM

## Engine / interpreter

Ideally the FVM won't *dictate* which underlying WASM VM must be used. However, we'll need to pick *a* VM implementation.

- Wasmer: Fancy, has a company behind it.
- Wasmtime: More "open", written in rust.
- WasmEdge: Designed for smart contracts.
- Node.js: We could just use node. Not great for embedding.
- gasm: un-optimized interpreter, so probably slower, but golang native

## Floating Point Operations

EWASM forbids floats and the EVM doesn't have them, but in WASM, at least, only *NaN* is non-deterministic. In theory, we could support floats and just normalize results when *NaN*.

**Tentative Decision:** no floats for now, we can add them later if necessary.

- Near is supporting floats: [https://github.com/near/nearcore/issues/1987](https://github.com/near/nearcore/issues/1987)
- EWASM does not.
- Context: [Document why NaN bits are not fully deterministic · Issue #619 · WebAssembly/design](https://github.com/WebAssembly/design/issues/619)

## 32bit or 64bit

The EVM uses 256bit everything, but the EVM is special there.

WASM supports both as separate specs.

**Tentative Decision:** FVM will use 32bit "pointers":

- Better supported.
- Less memory.
- We don't need more than a 4GiB address space on a blockchain (for WASM memory, at least).
- The 32bit VM still supports 64bit integers, just not 64bit memory "pointers".

## Limits

WASM's "[non-determinism](https://github.com/WebAssembly/design/blob/main/Nondeterminism.md)" section is a bit sparse on this.

Importantly, we need to figure out how to measure and limit the stack size this because the stack contains *objects*, it's not just an array of bytes. In addition to numbers, it contains jump labels and frames (which can contain locals, etc.). To avoid having to modify the VM, we'll likely want to instrument branches/calls to check the stack.

# IPLD Memory Model

All state will be stored and retrieved as IPLD. This is important as the VM itself needs to understand the IPLD so we can do garbage collection, etc.

Codecs should ideally be defined as WASM libraries, imported through the WASM import system. Ideally, ADLs and selectors would be implemented in the same way.

Constraints:

1. It must be possible to determine which data is "reachable" by actors, and which isn't. Furthermore, actors must only be allowed to read and/or reference "reachable" data.
2. We'll need a way to reference piece CIDs etc. *without* considering them to be *reachable*.
3. The caller must be charged gas for all work, including any work done by IPLD codecs.

There are three ways to go about this:

1. Hard-code "unreachable" codecs like we currently do. This is, IMO, a non-option.
2. Require that *everything* be reachable, possibly introducing some form of "non-traversable link" kind into IPLD, kind of like a symlink.
3. Use selectors to specify what data is reachable and what isn't.

Out of these options, 3 is likely the most flexible, but it's also hard. We'll likely want to start with a single hard-coded "selector", then upgrade to arbitrary selectors later.

## State Transaction

Below are algorithms for implementing options 1, 2, and 3 as described below. Really, there are two conceptual ways to do this:

1. Let actors access "abstract" IPLD objects, managed by the runtime.
2. Give actors the raw blocks, running any relevant codec code in the actor's context.

I'm focusing on the second approach here because:

1. Language agnostic abstract APIs can be built on-top-of the raw-blocks approach.
2. The "raw blocks" approach gives the actor more flexibility and avoids lots of context switching. The second one is the important part as option 1 would require context switching from the actor, into the runtime, into a codec sandbox, and back for every IPLD *node* operation (e.g., reading a field from an IPLD object).

The main drawback to this approach is security: the codec code will be running in the context of the actor. See the [security considerations](#security-considerations) section for more discussion on this.

Notes:

- Each block is only read once and cached for the duration of the transactions.
- Blocks are only written on commit, if needed, in reverse topological order.
- Where possible, IPLD codec operations are performed inside the actor's context, not inside the runtime, for better performance and flexibility. Unfortunately, full selector support may complicate this.
- Context transitions are clearly marked as **[A → B]**.

Ideally we'd be able to use multiple memories and read-only memory, but these features are still experimental in WASM and not widely supported. Especially because nobody really knows how to *address* multiple memories.

**TODO**: Both of these algorithms require hashing twice: once in the runtime and once in the actor. We should hash once in the runtime only by moving the "write cache" into the runtime.

### Options 1 & 2

We can implement this as follows, assuming IPLD codecs implement some `Links() -> []CID` function.

**Algorithm:**

1. **[actor → runtime]** Open a transaction.
    1. Define a `WriteCache` for written blocks in the actor's memory. Nothing will be persisted till the end of the transaction.
    2. Define a `ReadCache` for read blocks in the actor's memory.
    3. Define a `ReachableSet` set of CIDs inside the runtime.
    4. Initialize `ReachableSet` to the actor's state-root.
2. **[actor]** When loading a block:
    1. **[actor]** If the block is in one of the caches, return it.
    2. **[actor → runtime]** Ask the runtime to load a block:
        1. If the CID is not in the `ReachableSet` set, abort.
        2. Otherwise, load the block into the `ReadCache`.
        3. **[runtime → codec]** Call the codec's `Links` function.
        4. Add the returned CIDs (possibly filtered) to the `ReachableSet` set.
    3. **[actor]** Load the block from the cache.
3. **[actor]** When storing a block, put it into the `WriteCache`.
4. **[actor → runtime]** Commit the transaction:
    1. Initialize an empty `WriteStack` (blocks).
    2. Initialize an `ExploreStack` to the new state root (CID).
    3. Until the `ExploreStack` is empty:
        1. Pop a CID off the `ExploreStack`.
        2. If the CID is in the `ReachableSet`, continue.
        3. Load the block from the `WriteCache`. If it's missing, abort.
        4. **[runtime → codec]** Call `Links` on the block, pushing the result onto the `ExploreStack`. A gas multiplier will be applied to cover the cost of garbage collection, VM flushing, etc.
        5. Push the block onto the `WriteStack`.
        6. Add the block to the `ReachableSet`.
    4. Until the `WriteStack` is empty, pop blocks from the top and persist them.

### Option 3 (selectors)

In this variant, actors would declare a `StateSelector` (at deploy time, likely) specifying which sub-dag should be *reachable* at runtime. Instead of providing a simple `Links` function, codecs must support selector traversal.

**Algorithm:** *(differences are marked with ⭐)*

1. **\[actor → runtime]** Open a transaction.
    1. Define a `WriteCache` for written blocks in the actor's memory. Nothing will be persisted till the end of the transaction.
    2. Define a `ReadCache` for read blocks in the actor's memory.
    3. ⭐ Define a `ReachableSet` of `(Cid, Selector)` tuples, inside the runtime.
    4. Initialize `ReachableSet` to the actor's state-root and root selector.
2. **\[actor]** When loading a block:
    1. **\[actor]** If the block is in one of the caches, return it.
    2. **[actor → runtime]** Ask the runtime to load a block:
        1. If the CID is not in the `ReachableSet` set, abort.
        2. Otherwise, load the block into the `ReadCache`.
        3. ⭐ For each selector in associated with the CID in the `ReachableSet`:
            1. **[runtime → codec]** Execute the selector on the block up to the block boundary.
            2. Add any selected links (CIDs) along with their associated sub-selectors to the `ReachableSet`.
    3. **\[actor]** Load the block from the cache.
3. **\[actor]** When storing a block, put it into the `WriteCache`.
4. **[actor → runtime]** Commit the transaction:
    1. Initialize an empty `WriteStack` (blocks).
    2. ⭐ Initialize an `ExploreStack` and push `(StateRoot, Selector)`.
    3. ⭐ While the `ExploreStack` is non-empty:
        1. Pop a `(Cid, Selector)` pair off the `ExploreStack`.
        2. If the pair is in the `ReachableSet`, skip it and continue.
        3. Load the block from the `WriteCache`. If missing, abort.
        4. **[runtime → codec]** Traverse the `Cid` according to the selector. At all block boundaries, push `(Cid, Selector)` onto the `ExploreStack`.
        5. Push the block onto the `WriteStack`.
        6. Add the `(Cid, Selector)` pair to the `ReachableSet`.
    4. Until the `WriteStack` is empty, pop blocks from the top and persist them.

## Security Considerations

We'd like to support user-defined codecs. However, there are some security considerations.

The first consideration is GC, reachability, etc. `Links` and/or selector execution could, in theory, take an arbitrary amount of time (gas).  On the other hand, execution will be deterministic so we can always *reject* new blocks in the state if operating over said blocks would take more than some per-determined amount of gas. Furthermore, a gas multiplier could be charged to cover future GC and state operations.

Second, there's a trade-off between statically "trusted" codecs and "arbitrary" codecs. Ideally, actors would be able work with any IPLD data. However, these arbitrary codecs would need to be run in a separate sandbox, potentially costing us significant runtime overhead on every IPLD operation. Furthermore, even when run in a sandbox, a malicious codec can still return inconsistent results leading to potential security issues.

Alternatively, "trusted" codecs will likely get us most of the way there. However, we'll need to be able to determine *which* codecs to trust. We can:

- Statically link codecs (importing them with WASM imports).
- Provide a registry of "blessed" codec implementations that the network trusts.

We'll likely start with "trusted" codec implementations and introduce the ability for actors to use sandboxed "untrusted" codecs later (mostly for receiving IPLD data from other actors).

# Global State

It should be possible to support global, shared static IPLD data which actors can declare in their "imports". This "imported" data would be implicitly added to the "reachable" set inside transactions.

It would also be nice to support *dynamic* data, but this may be better served by sending messages between actors.

# Calling Convention

Proposed calling conventions:

- `Send` invoke a method on another actor.
- `Call` call a *function* exported by another actor and/or static code object. This should cover the EVM's "dynamic call" use-case.
- `CallSafe` call a *function* exported by another actor and/or static code object in a transient sandbox. The EVM doesn't support this, but the the ability to [send arbitrary DAGs](#sending-data) in the FVM makes this *very* useful.

In terms of implementation:

1. `Send` will always call the target actor's exported `Invoke` function. Actor method dispatch will happen internally as it's done in the EVM (allowing for things like dynamic dispatch).
2. There are a few options for for `Call` and `CallSafe` , but they likely *won't* call `Invoke`.

**TODO:** We still need to figure out how to actually pass/format data.

## Sending Data

Ideally we'd be able to be able to *send* an IPLD DAG from one contract to another, without having to send all the data directly. Basically:

1. When an IPLD object is sent to a different actor, the runtime would validate that all referenced blocks are in the sender's "accessible" set (see above).
2. When an actor receives an IPLD block, linked blocks (and associated selectors if selector-based reachability is being used) would be added to this actor's reachable set.

At the moment, messages are defined to be "raw bytes" interpreted by the actor. We'd have to define the method parameters as an IPLD object.

## EVM Comparison & Tradeoffs

This section discusses design tradeoffs with respect to the EVM.

### Method Dispatch

The EVM has no concept of methods, contracts handle dispatch internally. The current Filecoin VM does method dispatch externally. The FVM will likely follow the EVM here.

Trade-offs:

- Handling method numbers externally may allow for better type checks, and possible type introspection.
    - Actors can also just "export" type information. Honestly, this is probably the best approach.
- Handling method numbers internally allows for better dynamic dispatch.
    - This could also be done through a pythonic "dynamic dispatch" function, kind of like python's `__getattr__`.

### Delegate Call/Call Code

The EVM supports something called "delegate" calls where one contract can call another without changing to the other contract's context.

"Call code" is an older version that doesn't preserve the message sender/value.

While a bit of a security concern, the FVM will need to support *something* like this to allow contract code to be dynamically updated.  The proposed `Call` should cover this case.

# EVM Contracts

The FVM will need to support EVM contracts out of the box. We have a few possible approaches:

1. Multiple VMs: We could support both the EVM and a WASM based VM, calling between them.
2. AOT Solidity Compiler: We can compile Solidity code directly to WASM.
3. AOT EVM Transpiler: We could transpile from EVM byte code, or compile Solidity code directly to WASM.
4. EVM Emulation: We could emulate the EVM at runtime, possibly performing some amount of JIT at deploy time.

**Opcode List:**

[GitHub - crytic/evm-opcodes: Ethereum opcodes and instruction reference](https://github.com/crytic/evm-opcodes)

## Multiple VMs

This will bring the best compatibility, but will also be a large maintenance/portability burden.

## AOT Solidity Compiler

This option would provide the best performance, but:

1. It would only support Solidity contracts, not arbitrary EVM contracts. 
2. The compiler backend, likely LLVM, may make optimization decisions that could make debugging difficult, or could lead to runtime bugs (due to undefined behavior).

## AOT EVM Transpiler

This option would provide the best performance/compatibility tradeoff.

## EVM Emulation (or JIT)

This option would allow EVM contracts to be deployed without modification to the Filecoin network and would likely provide the best EVM compatibility short of having multiple VMs.

The biggest downside is performance, but this may be alleviated with a JIT. The next biggest downside is deployment cost.

### Deploy

In this scenario, there would need to be a special EVM "deploy" contract. This contract would need to:

1. Take an EVM contract as a parameter.
2. Compile/JIT it.
3. Deploy a new "EVM" contract with the generated code.

### JIT

It should be possible to "JIT" an EVM contract on deploy to improve runtime performance and reduce gas usage.

The first stumbling block is 32 byte words (instead of the usual 4/8 byte words). There isn't really a great way to optimize this...

The second stumbling block is control flow. In WASM, control flow is highly restricted and typed, while in the EVM:

1. A program can jump to any JUMPDEST instruction (single byte, 0x5e).
2. Contracts often contain *arbitrary* *data* in their code section.

To JIT, we'll have to build a massive branch table with all possible JUMPDEST instructions.

The stack is going to be a bit weird as WASM expects blocks to exit after having consumed/pushed a deterministic number of items to the stack. EVM contracts will likely need a split stack. We could store the entire stack in a memory, but that won't perform very well.

1. A stack in a memory for items used across jump boundaries.
2. Followed by a normal WASM stack. Items remaining on this stack at the end of a block will need to be copied to the "permanent" stack.

Optimizations:

1. It may be possible to detect function calls, but that optimization can be implemented later. The main benefit would be better stack management, maybe.
2. It may be possible to detect loops, which will help us avoid migrating the stack around.

# Gas Accounting

We two general categories of approaches:

1. Instrumentation (insert gas accounting instructions).
2. VM support.

In theory, implementations can chose what they want to do, but instrumentation is likely going to be easier. Here we're going to focus on instrumentation because "VM support" will be VM specific. However, the main concern is performance: ideally instrumentation wouldn't require *calling* into the runtime.

The efficient options are: multiple memories (not well supported) or globals (well supported). Both can be used, in conjunction with static analysis, to safely perform gas accounting without calling into the runtime.

With globals, we can:

1. Statically analyze the code to find an unused global (or define the "gas accounting global" and verify that it's not written to).
2. Insert instructions that perform gas accounting in this global.

Unfortunately, this is going to cost us 4 instructions (global.load, const, add, global.store) at every branch target. But that's still faster than calling a runtime function.

To ensure that *all* costs are accounted for, branch targets ("labels") will have to be taken into account in gas accounting.

# Runtime

The actor will need to be able to retrieve information from the runtime, call other actors, etc.

Experimental rust code lives in:

[GitHub - filecoin-project/fvm-runtime-experiment](https://github.com/filecoin-project/fvm-runtime-experiment)

## Static Environment

Actors will need to be able to retrieve static data from the environment. We can either pre-load this data into the actor's memory before calling, or expose runtime functions to retrieve this data.

The simplest is to let the actor import functions to call into the runtime. Unfortunately, it also has a runtime cost switching between the actor and the runtime. This is what [eWASM does](https://github.com/ewasm/ewasm-rust-api/blob/master/src/native.rs) and this is really the *only* way to call other actors.

```rust
extern "C" {
  pub fn fvm_getCaller(...);
}
```

The second option, which only works for static information, is to let the actor statically allocate memory for the runtime fields it needs, and export symbols referring to these static allocations. The runtime would fill these in before invoking the actor. The upside is that there is no runtime overhead whenever the actor accesses this information. Unfortunately:

1. In my experiments, these regions get included as zero bytes in the actual binary. This will bloat chain state and likely makes it a non-option.
2. They won't be dynamically sized so we can only pass fixed-sized fields this way (or over-allocate).

```rust
#[no_mangle]
static CALLER: Address = BLANK_ADDRESS;
```

Which gets WASMified as:

```wasm
...
(global (;1;) i32 (i32.const 1048576))
...
(export "thing" (global 1))
...
(; Note the inlined zeros ;)
(data (;0;) (i32.const 1048576) "\00\00...")
```

In theory, it should be possible for an actor to "import" a global exported by another module. However, there doesn't appear to be a way to get rust to "import" a global in general. Unfortunately, it's not looking like that's going to change: https://github.com/rust-lang/rust/issues/60825.

## Allocation & Variable Sizes

The EVM and eWASM only needs "variable sized" objects in two cases: call data and return data. To access this, the actor:

1. Asks the runtime for the *size* of the data.
2. Allocates an appropriately sized buffer.
3. Asks the runtime to copy (at least some of) the data into this buffer.

However, in Filecoin, we have CIDs and blocks which can *all* be variable sized. Asking how big some object is, then calling into the runtime again to copy could get expensive.

We have two alternatives:

1. Just allocate some buffer up-front with the maximum supported size. This is fast, simple, and allows the caller to easily set limits. Unfortunately, it requires a potentially large allocation up-front, and likely an additional copy.
2. Have the actor expose an "allocate" function. Unfortunately, this means the runtime would need to call back into the actor to perform allocations, imposing additional complexity and determinism requirements.

The current plan is to follow Unix's example and use file "descriptors" to reference blocks.

1. Ask the runtime to "load" a block, get back a handle, codec, and size.
2. Ask the runtime to "store" a block, get back a handle.
3. Read from the handle into a local buffer.

# Built-in Actors

Filecoin has some "privileged" built-in actors. Most of these will continue to operate as-is, however:

1. The init actor will need to be extended to support user deployable actors.
2. Ideally, cron (or at least *a* cron implementation) will be usable by end-users. Unfortunately, the *current* version isn't secure for these purposes as it bypasses the gas system.

## Cron

As-implemented, Filecoin Cron isn't safely usable by arbitrary actors. However, it should be possible to implement an unprivileged cron as long as some of the guarantees are relaxed a bit.

Basically, we implement two methods:

- `Schedule` — called by actors to schedule a future job.
- `Run` — called by anyone (but usually the block producer) to execute a subset of the available jobs.

Unlike the current cron, here there would be no guarantee that callbacks will get invoked. Instead, it's up to the calling actor to make sure that invoking their callback is profitable to the block producer.

Example `Schedule` params:

```go
type ScheduleParams struct {
	// Height is the target height at which the method should be invoked.
	Height ChainEpoch
	// Timeout is the number of epochs after Height by which the method must
	// be invoked before the scheduled invocation becomes invalid.
	Timeout ChainEpoch

	// The specified method will be invoked on the _caller_ with the specified
	// arguments.
	Method MethodNumber
	Params []byte

	// MaxGas is the maximum amount of gas the message can use.
	MaxGas uint64
	// MaxBaseFee is the maximum base-fee at which the method should be invoked.
	MaxBaseFee TokenAmount
	// A reward paid to the caller of Invoke. The reward will be paid regardless
	// of the gas used.
	Reward TokenAmount // should probably be an auction, not fixed.
}
```

Notably, callbacks must specify a maximum gas (for efficient message packing) and a maximum base-fee (to compete with normal messages). Additionally, the calling actor must escrow $MaxGas \times MaxBaseFee + Reward + ExpirationDeposit$ (any left-over funds will be returned when the callback is executed or expires). The $ExpirationDeposit$ covers the cost of cleaning up expired callbacks.

Example `Invoke` params:

```go
type InvokeParams struct {
	Callbacks BitField
}
```

When called, the *caller* pays gas for *all* cron callbacks. However, the cron actor will pay the *caller* (not the miner) $BaseFee\times MessageGas + Reward$ for invoked callbacks, and $ExpirationDeposit$ for expired callbacks. 

Given how lucrative this message will likely be, we'd expect it'll only be invoked by the block miner itself.

Notes:

- Instead of specifying a set of callbacks, the caller could specify a minimum acceptable reward and execute messages greedily. That's more fair but could also be more expensive to execute on-chain.
- In this construction, an attacker could take a miner's "invoke" message from a block that didn't get executed, and copy it to a block that did (but *after* a conflicting message) forcing the miner to waste a lot of gas. We probably need to allow miners to specify more complex message constraints to make this work.

# Resources

## Tools

Decompilers/inspectors: [https://github.com/WebAssembly/wabt](https://github.com/WebAssembly/wabt)

Importantly, this includes `wasm2wat` and `wat2wasm` for de-compiling and re-compiling. 

## Rust → WASM

*NOTE: You don't need `wasm-pack`, or `wasm-bindgen`. Those are for shipping NPM modules with JavaScript interfaces.*

Install rustup and install the wasm target: `rustup target add wasm32-unknown-unknown`

Create a rust project: `cargo new --lib some-project`

Make it a "cdylib" by adding the following to the `Cargo.toml`:

```
[lib]
crate-type = ["cdylib"]
```

Declare "imports" with `extern` blocks. These will be imported from `"env"`. NOTE: Imports won't be declared if unused.

```rust
extern "C" {
    fn foo();
}
```

Export by marking functions and statics as `#[no_mangle]`:

```rust
// This will be exported as:
// 1. A global specifying the location in memory where "thing" resides.
// 2. A 20 byte chunk of zeros in the default memory.
#[no_mangle]
static thing: [u8; 20] = [0; 20];

// This will be exported as a single "main" function.
#[no_mangle]
fn main() {
    println!("Hello, world!");
}
```

Build with `cargo build -t wasm32-unknown-unknown --release`
