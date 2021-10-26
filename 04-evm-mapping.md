# FVM <> EVM mapping

This document describes the mappings of structures, types, and procedures between the native FVM environment (WASM), and the EVM foreign VM (emulated).

## World state / state tree

World state 

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

## Address semantics

## Gas accounting and execution halt semantics

## Ethereum logs/events

## Blockchain timing

## Precompiles

## Cryptographic primitives

## References

- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
