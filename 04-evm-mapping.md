# FVM <> EVM mapping

This document describes the mappings of structures, types, and procedures between the native FVM environment (WASM), and the EVM foreign VM (emulated).

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Integration](#integration)
  - [State tree](#state-tree)
  - [Mechanics and interfaces](#mechanics-and-interfaces)
- [Contract memory model](#contract-memory-model)
- [Account storage model](#account-storage-model)
- [Addressing scheme](#addressing-scheme)
  - [Proposed solution: universal stable addresses](#proposed-solution-universal-stable-addresses)
- [Gas accounting and execution halt semantics](#gas-accounting-and-execution-halt-semantics)
- [Ethereum logs/events](#ethereum-logsevents)
- [Blockchain timing](#blockchain-timing)
- [Precompiles](#precompiles)
- [Cryptographic primitives](#cryptographic-primitives)
- [Chain-specific opcodes](#chain-specific-opcodes)
- [Cross-contract calls](#cross-contract-calls)
- [References](#references)
- [Annex A: Addressing solutions considered](#annex-a-addressing-solutions-considered)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Integration

### State tree

There won't be a segregated world state for Ethereum actors in the Filecoin blockchain. We will not use the Patricia Merkle Trie structure. Instead, EVM account state will be folded and docked into the Filecoin state tree.

### Mechanics and interfaces

EVM smart contracts (also known as EVM foreign actors in Filecoin terminology), are represented this way in the Filecoin state tree:

```
{
	Code:       cid('fil/<version>/evm/<runtime_version>'),
	Head:       cid(<actor_state>),
	Nonce:      uint64(<nonce>),   
	Balance:    bigint(<fil_balance>),
}
```

Notice that EVM foreign actors are typed in the state tree with a CodeCID that does not correspond to their EVM bytecode. Instead, the CodeCID points to the WASM bytecode of the **EVM foreign runtime**.

The EVM init code is passed as a constructor parameter during instantiation, when calling the `Exec` method of the `InitActor`:

```
{
    CodeCID: cid('fil/<version>/evm/<runtime_version>'),
    ConstructorParams: {
        InitCode: <evm_init_code>
    }
}
```

The constructor of the EVM foreign runtime evaluates the EVM init code, generates the runtime bytecode, and stores it in the actor's state.

When the actor is called, the FVM loads the code of the EVM foreign runtime, which in turn loads the EVM bytecode from the actor state for _interpretation_. In the future, we may consider optimizing for performance through compilation or transpilation. This is arguably improbable, because the EVM runtime exists mainly for ecosystem compatibility, and is not the primary Filecoin actor runtime.

The CodeCID refers to a concrete version of the EVM foreign runtime. This permits evolution, and code upgrade operations as per the the [FVM Architecture document](01-architecture.md) apply.

The EVM actor's state schema is as follows:

```
{
    Bytecode:       <evm_runtime_bytecode>,
    StorageRoot:    <hamt_root_cid (see below)>,
    LogsRoot:       <vector_root>,
}
```

## Contract memory model

In the EVM, contract memory is volatile and its lifetime is scoped to the execution of a transaction. Contract memory is a simple word-addressed byte array, where the word size is 256 bits (32 bytes). Contract memory is effectively heap, and is technically unlimited but de-facto bounded by gas limits. Conversely, stack is limited to 1024 words (also 256-bit sized).

Memory costs gas; and memory operations pay gas too. The memory cost is determined by a polynomial function over the count of words that cover the address ranges referenced during execution. This price is charged just-in-time as memory is expanded to accommodate address ranges supplied to `MLOAD` and `MSTORE`. In addition to those variable JIT costs, the `MLOAD` and `MSTORE` opcodes have a fixed "very low" cost (currently 3 gas units).

The current Filecoin VM does not track memory usage. It also doesn't charge for memory usage explicitly. The fundamental heuristic shaping the current gas schedule is wall-clock execution time of specific operations and syscalls. This model is only possible because all logic on-chain is defined ahead of time, and thus can be studied deterministically for resource utilisation.

With the FVM, that invariant will be invalidated, and memory utilisation of user-defined code will need to be accounted and charged for. The exact model is under analysis, but what's clear is that it'll be tightly linked to [WASM's own memory primitives](https://radu-matei.com/blog/practical-guide-to-wasm-memory/).

Note that WASM memory is unmanaged. Modules obtain memory pages from the WASM host (page size is currently fixed to 64KiB). Modules rely on allocators, such as Rust's default allocator, or slimmer options like [wee_alloc](https://github.com/rustwasm/wee_alloc), to manage that memory. It's likely, but not decided, that FVM's memory-specific gas costs will be applied on host memory requests, by intercepting and metering those operations, as those are indicative of actual system resource usage.

Concerning the EVM <> FVM memory model mapping, memory referencing, the instruction set, and memory usage accounting and consequent gas charging, is largely self-contained within the EVM interpreter. Hence these specificities are opaque to the FVM.

However, keep in mind that execution is ultimately controlled by FVM gas and not EVM gas (see below). Hence, the memory-efficiency of the chosen EVM interpreter ([SputnikVM](https://github.com/rust-blockchain/evm) in the reference FVM) —specifically under a WASM environment— is technically what matters when emulating EVM smart contracts atop the FVM.

To avoid divergences, the WASM bytecode of the EVM foreign runtime actor must be identical across Filecoin node implementations.

**Recommendation:** We may have to invest time and effort to optimise the WASM memory footprint of the EVM runtime of choice.

## Account storage model

Account storage in Ethereum takes the form of a map `{ uint256: rlp(uint256) }`. This format will be mapped to an IPLD HAMT in the FVM, and stored in the actor's state. We expect no complications, besides those that may arise from gas accounting divergence in storage operations (IPLD vs. Ethereum data model), storage clearing, and cold/warm storage differentiation.

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
- Class `2` (non-account actor addresses): payload is a blake2b-160 hash of a payload generated by the init actor (account sender, nonce, number of actors created within the execution). Fixed 20 bytes.
- Class `3` (BLS key): payload is an inlined BLS public key. Fixed 48 bytes.

In conclusion, the maximum address length in Filecoin is 49 bytes or 392 bits (class 3 address). This creates two problems:

1. The worst case scenario is larger than the width of the Ethereum address type. Even if BLS addresses were prohibited in combination with EVM actors, class 1 and class 2 still miss the limit by 1 byte (due to the prefix).
2. It exceeds the EVM's 256 bit architecture.

Problem 1 renders Solidity smart contracts instantly incompatible with the Filecoin addressing scheme, as well as EVM opcodes that take or return addresses for arguments, e.g. CALLER, CALL, CALLCODE, DELEGATECALL, COINBASE, etc. This problem is hard to work around, and would require a fork of the EVM to modify existing opcodes for semantic awareness of addresses (although this is really hard to get right), or to introduce a Filecoin-specific opcode family to deal Filecoin addresses (e.g. FCALL, FCALLCODE, etc.) The latter would break as-is deployability of existing smart contracts.

Problem 2 can be workable by spilling over and combining values in the stack, through Filecoin-specific Solidity libraries.

Find the proposed solution next, and a summary of alternatives considered in Annex A.

### Proposed solution: universal stable addresses

The idea is to promote address class 2 into a unified fixed-size stable address space all actors, **including account actors**. At present, this class only identifies **non-account actors**. The only address space that covers all actors is the ID class, which is not reorg-stable and thus unsafe.

The width of this address space is already 20 bytes. EVM actors would only interact with class 2 addresses, with the prefix stripped to honour the 160-bit address expectation, for full EVM bytecode compatibility.

Nevertheless, it has been noted that the security of 20-byte addreses is beginning to be insufficient, with an estimated cost of $10bn to find an arbitrary collision by carrying out 2^80 computations.

Therefore, the actual proposal is **to introduce a new address class 4 for a 256-bit stable address space**. This class would become the **canonical stable address class**, with class 2 addresses (20-byte) mapping over to their class 4 equivalents (32 bytes). Class 2 can thus be thought of as an alias/symlink to class 4, to be used in environments that are unable handle larger widths.

The state tree would track class 2 and class 4 address for all actors by maintaining the necessary indices.

**Class 2 and 4 address derivation**

Class 2 and 4 addresses are protocol-derived, by hashing the relevant cryptographic or input material.

With a high probability, the hash function of class 4 will remain BLAKE2, with a 256-bit digest size. Class 2 will continue relying on blake2b-160.

1. For secp256k1 account actors (class 1), the preimage is the pubkey. The value of the class 2 address is identical to that of class 1.
2. For BLS account actors (class 3), the preimage is the pubkey.
3. For FVM native actors, the preimage is `sender || nonce || # of actors created during message execution`.
4. For EVM foreign actors, the preimage is inherited from CREATE and CREATE2.

Note that preimages are not user-controlled, but some constituents of them may be (e.g. EVM CREATE2 opcode).

**Resulting address space taxonomy**

| Class | Desc                              | Actor type | Payload width | Total width | Payload value                                         | Usage                                                                                | Stable? |
| ----- | --------------------------------- | ---------- | ------------- | ----------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------ | ------- |
| 0     | ID address                        | All        | 1-9 bytes     | 2-10 bytes  | uvarint64 counter                                     | Internal, compact representation in state tree; unsafe to use externally until final | N       |
| 1     | secp256k1 pubkey (account actor)  | Account    | 20 bytes      | 21 bytes    | blake2b-160 hash of secp256k1 pubkey                  | Externally, to refer to an account actor with its pubkey                             | Y       |
| 2     | aliased universal actor address   | All        | 20 bytes      | 21 bytes    | protocol-derived from relevant cryptographic material | Externally and internally to refer to any actor                                      | Y       |
| 3     | bls pubkey (account actor)        | Account    | 48 bytes      | 49 bytes    | inlined bls public key                                | Externally, to refer to an account actor with its pubkey                             | Y       |
| 4     | canonical universal actor address | All        | 32 bytes      | 33 bytes    | protocol-derived from relevant cryptographic material | Externally and internally to refer to any actor                                      | Y       |

## Gas accounting and execution halt semantics

The execution halt is determined by Filecoin gas and not by EVM gas. Therefore, EVM runtime is made to run with unlimited gas. The FVM is responsible for metering execution and halting it when gas limits are exceeded. Refer to the [Gas accounting](01-architecture.md#gas-accounting) section of the Architecture doc for more details.

The exit code of the execution matches the Filecoin value for "out of gas" situations.

## Ethereum logs/events

TBD.

## Blockchain timing

Ethereum target block times are ~10 seconds, whereas Filecoin's is ~30 seconds. A priori, this difference has no impact on the protocol or this spec, but it may impact the behaviour of smart contracts ported over from Ethereum that expect 10-second block timing.

## Precompiles

Precompiles from Ethereum will be honoured.

However, because the full EVM runtime is compiled to WASM, so are the precompiles provided in the source code of the EVM runtime. Therefore, technically speaking, they are not _native precompiles_.

In the future, we may optimise by having them call a Filecoin syscall to escape to native land. This would imply a traversal of Boundary A. See the [Domains and boundaries](01-architecture.md#domains-and-boundaries) section of the architecture doc.

## Cryptographic primitives

TBD.

## Chain-specific opcodes

The following are some notable chain specific opcodes whose behaviour is translated to the Filecoin environment as specified:

* `BLOCKHASH`: returns the hash of the first block in the specified lookback tipset. 
* `GASLIMIT`: returns the gas limit as per Filecoin gas system.
* `CHAINID`: returns a fixed value `0`.
* `GAS`: returns the gas remaining as per Filecoin gas system. One divergence from Ethereum is the return value does not include the full cost of this operation (because the cost of stack copy and program advance is not known when the value is captured).
* `COINBASE`: returns the Filecoin class 4 address of the block producer including this message.
* `NUMBER`: returns the chain epoch where the message gets included. Note that this is one epoch behind the execution epoch (Filecoin defers execution to the next tipset).
* `DIFFICULTY`: returns fixed value `0`.
* `CODESIZE`: returns the size of the EVM bytecode.

## Contract creation

Both `CREATE` and `CREATE2` are supported. Under the hood, these opcodes will invoke `InitActor#Exec` with the the same `CodeCID` (hence, the same EVM foreign runtime version) as the currently running EVM actor. The returned address is a class `4` address, or 0 if the operation failed (respecting Ethereum's behaviour).

The computed address matches the relevant expectations for the opcode in question. That is, the ability for `CREATE2` to generate addresses suitable for counterfactual creations, or prospective actor interactions, is preserved.

## Cross-contract calls

Both **EVM <> EVM** and **FVM <> EVM** calls are supported.

Solidity code will generate the input parameters conforming to the callee ABI by default, suitable for EVM calls.

For EVM actors to be able to call FVM actors, a Solidity library needs to be written to pack the call's input data in conformance with the appropriate calling convention or call template, as defined by the target actor's IPLD Interface Definition.

## References

- [Ethereum Yellow Paper (Berlin Version 888949c – 2021-10-15)](https://ethereum.github.io/yellowpaper/paper.pdf)
- https://github.com/ethereum/EIPs/issues/684
- [EIP-1014: Skinny CREATE2](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1014.md). Clarifications section elaborates on collisions. 
- [EIP-684: Prevent overwriting contracts](https://github.com/ethereum/EIPs/issues/684)
- [EIP-161: State trie clearing (invariant-preserving alternative)](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-161.md)

## Annex A: Addressing solutions considered

_⚠️  These solutions are UNSOUND and have been discarded._

**Using ID addresses**

The sizing issues can be overcome by using Filecoin ID addresses (max. 10 bytes)inside EVM execution. This class of addresses fits within the bounds of the Ethereum address width (20 bytes).

However, this comes with challenges:

1. EVM smart contracts would be unable to send FIL to yet-uninitialized, pubkey account addresses (and rely on account actor auto-creation), due to Problem 1.
    - Potential solution: have the caller create the account on chain prior to  
      invoking the EVM smart contract. This would imply changes to the code of the smart contract.

2. EVM smart contracts would be unable to send FIL to yet-uninitialized actors, due to Problem 1, but also because the Filecion protocol doesn't support sending value to inexisting actors, unless they are account actors (and this interaction is affected by the challenge above).

3. ID addresses are vulnerable to reorg within the current finality window, so 
submitting EVM transactions involving actors created recently (900 epochs; 7.5 hours) would be unsafe, as they could be reassigned to different backing actors in case of a reorg.
    - Potential solution A: have the runtime detect and fail calls involving 
      recently-created actors (undesirable).
    - Potential solution B: introduce the ability for the user to assert that an ID address is mapped to a concrete stable address. This would be done through a new address class `4`, the asserted ID address. Read below.

**Address handles**

We can consider using _address references/handles_ in the FVM EVM calling convention. Input parameters would be enveloped in a tuple:

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

**Address class 4: the asserted ID address**

The asserted ID address class decorates the standard ID address (class `0`) with an assertion about the mapping of that address to the underlying stable/pubkey address.

It is valuable in reorg-sensitive scenarios where, for some reason, relying on stable/pubkey addresses is unfeasible.

When dealing with a class `4` address, the code MUST verify that the ID address is effectively bound to the corresponding stable address. Only then will the address be a valid reference; else, the address MUST be rejected.

The format of a class `4` address is as follows:

```
byte idx    contents
--------    --------
[0]         address class (fixed value 0x04)
[1..9]      payload of class 0 address (uvarint64, worst case scenario; max. uint64 value)
[10]        length prefix denoting assertion byte count
[11..11+L]  assertion; L trailing bytes from stable address
```

The length of the assertion is variable, and is specified by the byte immediately following the address class 0 payload.

For the purposes of this EVM<>FVM mapping, the total length must not exceed 20 bytes; this leaves us with:

- a 9-byte assertion, in the worst case scenario
- a 16-byte assertion, in the best case scenario (non-system actors begin at ID 1000, which is represented by 2 bytes in uvarint64)

The guarantees to be derived from the assertion are probabilistic, with the operative risk being that an attacker manages to find a 9-byte collision (in the worst case scenario) and pulls of a chain reorg to remap the ID address to the colliding address, all within the current finality window, targeting a specific transaction. 

This solution is UNSOUND because an attacker could mine 25% of the entire address space with 2^(72-2) computations and 
