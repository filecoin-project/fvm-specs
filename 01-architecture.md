# Filecoin VM architecture

This documents provides an overview of the architecture of the FVM, including a small summary of the current VM as an annex at the end.

## Overview

![Proposed VM architecture](img/proposed-vm-arch.png)

**Hypervisor-inspired architecture**

<!-- Hypervisor abstraction, native runtime, Wasmer vs. Wasmtime -->

The FVM aims to (a) support a multitude of programming models and (b) facilitate the onboarding of smart contracts and programs written for other environments, so they can leverage the storage capabilities of the Filecoin network.

**Native runtime**

The native runtime will be based on WebAssembly (WASM), with potential usage of a limited set of WebAssembly System Interface (WASI) APIs.

**Built-in system actors**

The existing built-in system actors will be compiled to WASM and will run entirely in WASM space, thus preventing any associated FFI barrier costs.

For the sake of analogy, built-in system actors in Filecoin receive comparable treatment to "precompiled contracts" in other platforms. They are specified by the protocol, implicitly deployed, addressed at fixed locations/mailboxes, and they evolve through system upgrades.

_Canonical system actors_

This transition to WASM opens up the opportunity to converge on a single codebase of actors for all Filecoin implementations.

In the past, each team would re-implement actors in their language (although some relied on FFI with Go actors, a slower approach). With FVM, a single codebase can be compiled to WASM (its new portable executable form) and be adopted across all implementations.

We acknowledge this strategy has tradeoffs, but we won't elaborate on them here.

**Actor sandbox**

The actor sandbox is the tightly constrained and instrumented environment in which Filecoin actor code runs.

The actor sandbox must be formally specified, and potentially versioned. 

User-deployed actors may have to specify the sandbox version they expect, as this facilitates sandbox evolution while preserving backwards compatibility in an obvious manner.

Sandbox responsibilities:

1. To inject/bind/link implicit module imports that provide capabilities such as state access, chain access, syscalls, IPLD codec processing, and more.
2. To provide a toolbox of common functions and types, such as BigInt arithmetic, parameter unpacking, method dispatch tables, etc.
3. To manage memory usage and halt execution when it exceeds the limits.
4. To load input call parameters and handle return values.
5. To intercept cross contract calls and delegate to the outer environment.
6. To perform gas accounting by instrumenting bytecode.
7. To halt execution when available gas is exhausted.

(1) and (2) together form the basis of the **FVM stdlib**: the set of APIs that are universally available for actors to call inside the actor sandbox.

One key constraint is that actor code size/footprint should be kept as low as possible to avoid state bloat and gas costs. (2) is important, as it precludes repetitive/common code from being statically linked to user-deployed actors.

We could also add the ability to deploy libraries as on-chain actors; this would lead to a richer development experience and we'd expect collections of reusable libraries to emerge and be deployed on-chain, for others to consume. This is similar to the use cases of the `CALLDELEGATE` opcode in the EVM. But we'd probably tighten the security characteristics by introducing the notion of "capabilities", such that the calling actor can specify exactly what the target can access.

**Test sandbox**

The FVM project will need to ship a test actor sandbox for developers to use during their development process.

**Foreign runtimes**

Support for foreign runtimes, such as the EVM, will be introduced by emulating those runtimes over WASM by compiling the respective interpreter to WASM bytecode.

We expect a performance penalty from doing this (the extent of it is to be determined), but this tradeoff is workable in exchange for the highest execution fidelity/parity.

Note that the more performant alternatives imply compiling the respective high-level languages (e.g. Solidity) to WASM, which bears additional risks that are undesirable. Doing so would also render usesless tools that operate on their native target bytecode (e.g. auditing tools that analyse EVM bytecode, EVM bytecode inspectors, etc.) Thus we prefer to adopt an approach that allows usage of the full catalogue of preexisting tools to be preserved.

Each foreign runtime will require a shim to translate the storage model, gas accounting, account/address schema, gas metering, cross-actor calls, etc. to the native FVM runtime. The shim will also handle foreign chain/VM constructs by intercepting them and adapting them to the FVM native runtime.

This is the case of logs/events in the EVM, which may be stored in a central EVM logs actor.

**IPLD everything**

<!-- ipld-wasm library, codecs, selectors -->

**State access**

<!-- avoiding excessive boundary costs, garbage collection -->

**Chain access**

**Actor deployment**

**Cross-actor calls**

**Calling conventions**

<!-- single entrypoint; sync only; future async; parallel execution; IPLD schemas -->


**Syscalls**

**Gas accounting**

**WASM interpreters**

**JSON-RPC API**

**Interoperability with other networks**

**Formal verifiability**

**Upgradability**

<!-- at different layers, incl. evm shim -->

# Annex: Current VM

![Current VM architecture](img/current-vm-arch.png)

The term _Actor_ is a reference to the [actor model](https://en.wikipedia.org/wiki/Actor_model), a concurrent computation paradigm that inspires Filecoin's runtime and scalability primitives.

Actors operate on the state tree. Nothing else can modify the state tree in normal circumstances, other than actor logic. The single exception is state migration logic during a network upgrades. It can conduct bulk modifications, both to the content and the structure of the state tree.

The state tree is an IPLD object, containing the root of an [HAMT](https://ipld.io/specs/advanced-data-layouts/hamt/spec/#appendix-filecoin-hamt-variant) which in turn contains all actors, keyed by ID address.

Each actor has a type (represented by a CID), and a state root. The VM enforces actor state isolation, thus actors are prevented from accessing each other states.

Message passing is used to communicate between actors, even when simple state accesses are required. Mutations to state may only be applied within transactions.

Actor code is triggered through _messages_. Messages can be:

1. explicit: on-chain messages, leading to a _message receipt_ posted on chain as the result.
2. internal: between actors while processing a chain message, or triggered by a system event such as cron ticking.

Read more about the [structure of messages](https://spec.filecoin.io/#section-systems.filecoin_vm.message.message-syntax-validation).

Messages specify the actor method to invoke. Actors supply a method export table to the environment, and the VM performs method dispatch. This model will be revisited with the FVM, likely moving to actors exposing a single entrypoint, and dispatching internally with the assistance an SDK library.

Actor invocations are entirely synchronous. Actors can register entries in the cron actor, to schedule deferred execution at future epochs. Asynchronous calls are a desire of the upcoming FVM implementation, but not an immediate priority.
