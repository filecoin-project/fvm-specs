# FVM <> EVM mapping

This document describes the mappings of structures, types, and procedures between the native FVM environment (WASM), and the EVM foreign VM (emulated).

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [World state / state tree](#world-state--state-tree)
- [Contract memory model](#contract-memory-model)
- [Contract storage model](#contract-storage-model)
- [Gas accounting and execution halt semantics](#gas-accounting-and-execution-halt-semantics)
- [Ethereum logs/events](#ethereum-logsevents)
- [Blockchain timing](#blockchain-timing)
- [Precompiles](#precompiles)
- [Cryptographic primitives](#cryptographic-primitives)
- [References](#references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## World state / state tree

## Contract memory model

In the EVM, contract memory is volatile and its lifetime is scoped to the execution of a transaction. Contract memory is a simple word-addressed byte array, where the word size is 256 bits (32 bytes). Contract memory is effectively heap, and is technically unlimited but de-facto bounded by gas limits. Conversely, stack is limited to 1024 words (also 256-bit sized).

Memory costs gas; and memory operations pay gas too. The memory cost is determined by a polynomial function over the count of words that cover the address ranges referenced during execution. This price is charged just-in-time as memory is expanded to accommodate address ranges supplied to MLOAD and MSTORE. In addition to those variable JIT costs, the MLOAD and MSTORE opcodes have a fixed "very low" cost (currently 3 gas units).

The current Filecoin VM does not track memory usage. It also doesn't charge for memory usage explicitly. The fundamental heuristic shaping the current gas schedule is wall-clock execution time of specific operations and syscalls. This model is only possible because all logic on-chain is defined ahead of time, and thus can be studied deterministically for resource utilisation.

With the FVM, that invariant will be invalidated, and memory utilisation of user-defined code will need to be accounted and charged for. The exact model is under analysis, but what's clear is that it'll be tightly linked to [WASM's own memory primitives](https://radu-matei.com/blog/practical-guide-to-wasm-memory/).

Note that WASM memory is unmanaged. Modules obtain memory pages from the WASM host (page size is currently fixed to 64KiB). Modules rely on allocators, such as Rust's default allocator, or slimmer options like [wee_alloc](https://github.com/rustwasm/wee_alloc), to manage that memory. It's likely, but not decided, that FVM's memory-specific gas costs will be applied on host memory requests, by intercepting and metering those operations, as those are indicative of actual system resource usage.

Concerning the EVM <> FVM memory model mapping, we find that memory referencing, the instruction set, and memory usage accounting and consequent gas charging, is largely self-contained within the EVM interpreter. Hence these specificities are opaque to the FVM.

However, keep in mind that execution is ultimately controlled by FVM gas and not EVM gas (see below). Hence, the memory-efficiency of the chosen EVM interpreter ([SputnikVM](https://github.com/rust-blockchain/evm) in the reference FVM) —specifically under a WASM environment— is technically what matters when emulating EVM smart contracts atop the FVM.

**Recommendation:** We may have to invest time and effort to optimise the WASM memory footprint of the EVM interpreter of choice.

## Contract storage model

## Account model

## Addressing scheme

Ethereum uses 160-bit (20-byte) addresses. Addresses are the keccak-256 hash of the public key of an account, truncated to preserve the 20 rightmost bytes. Solidity and the [Contract ABI spec](https://docs.soliditylang.org/en/v0.5.3/abi-spec.html) represent addresses with the `address` type, equivalent to `uint160`.

There's an active, yet informal proposal to [increase the address width to 32 bytes](https://ethereum-magicians.org/t/increasing-address-size-from-20-to-32-bytes/5485).

In Filecoin, addresses are multi-class, and there are currently four recognized classes. Sidenote: they're actually called _protocols_ in the spec, but we'll refrain from using that term here because it's hopelessly overloaded.

The address byte representation is as follows:

```
class (1 byte) || payload (n bytes)
```

Thus, the total length of the address varies depending on the address class.

- Class `0` (ID addresses): payload is [multiformats-style uvarint](https://github.com/multiformats/unsigned-varint). Maximum 9 bytes.
- Class `1` (Secp256k1 key): payload is a blake2b-160 hash of the secp256k1 pubkey. Fixed 20 bytes.
- Class `2` (actor addresses): payload is a blake2b-160 hash of some payload generated by the init actor. Fixed 20 bytes.
- Class `3` (BLS key): payload is an inlined BLS public key. Fixed 48 bytes.

In conclusion, the maximum address length in Filecoin is 49 bytes or 392 bits (class 3 address). This creates two problems:

1. The worst case scenario is larger than the width of the Ethereum address type. Even if BLS addresses were prohibited in combination with EVM actors, class 1 and class 2 still miss the limit by 1 byte (due to the prefix).
2. It exceeds the EVM's 256 bit architecture.

Problem 1 renders Solidity smart contracts instantly incompatible with the Filecoin addressing scheme, as well as EVM opcodes that take or return addresses for arguments, e.g. CALLER, CALL, CALLCODE, DELEGATECALL, COINBASE, etc. This problem is hard to work around, and would require a fork of the EVM to modify existing opcodes for semantic awareness of addresses (although this is really hard to get right), or to introduce a Filecoin-specific opcode family to deal Filecoin addresses (e.g. FCALL, FCALLCODE, etc.) The latter would break as-is deployability of existing smart contracts.

Problem 2 can be workable by spilling over and combining values in the stack, through Filecoin-specific Solidity libraries.

**Solution A: using ID addresses**

However, there's a simpler solution: use Filecoin ID addresses (max. 10 bytes) everywhere inside EVM execution. However, this comes with drawbacks:

1. EVM smart contracts can't send to inexisting, stable account addresses, and rely on account actor auto-creation, as those addressess can't be used with EVM opcodes (see problem 1). Potential solution: have the caller create the account on chain prior to invoking the EVM smart contract.
2. ID addresses are vulnerable to reorg within the current finality window, so submitting EVM transactions involving actors created recently (900 epochs; 7.5 hours) would be unsafe. Potential solution: have the runtime detect and fail calls involving recently-created actors.

**Solution B: using address handles**

If these tradeoffs are unacceptable, we can consider using _address references/handles_ in the FVM EVM calling convention. Input parameters would be enveloped in a tuple:

```
(1) ABI-encoded parameters (using uint160 addr handles) || (2) { addr handle: actual addr }
```

Where:

1. ABI encoded parameters replacing address positions with indexed uint160.
2. Mapping of indices to real Filecoin addresses.

On an incoming call, the EVM <> FVM shim would unpack the call and pass only (1) as input parameters to the smart contract. It would use (2) to resolve the address whenever the smart contract called a relevant opcode. When returning, the EVM <> FVM shim would perform the inverse operation.

However, address-returning opcodes are still unsolved (e.g. CREATE, CREATE2, COINBASE, SENDER). The contract may want to persist these addresses, so making them return address handles is not an option, as they aren't safe to persist.

Finally, this approach alters the calling convention, which in turns breaks compatibility with existing Ethereum tooling like wallets (e.g. MetaMask).

**Solution C: using address guards**

Another alternative consists of adopting ID addresses (like proposed in Solution A), but when those addresses are "fresh" (i.e. created within the finality window), allowing to pack a stable address guard/assertion in a data structure similar to that of Solution B.

The EVM <> FVM shim would apply assertions prior to invoking the contract.

This solution imposes extra complexity on the caller (so as to determine address freshness). It may require extending the InitActor's state object to inline the creation epoch for ease of query.

This solution also suffers from the ecosystem tooling compatibility drawbacks, just like Solution B.

## Gas accounting and execution halt semantics

## Ethereum logs/events

## Blockchain timing

## Precompiles

## Cryptographic primitives

## References

- [Ethereum Yellow Paper (Berlin Version 888949c – 2021-10-15)](https://ethereum.github.io/yellowpaper/paper.pdf)
