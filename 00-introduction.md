# Introduction

The existing Virtual Machine (VM) in the Filecoin system acts as the execution environment for code deployed on-chain in the form of _Actors_. Actors are the Filecoin-equivalent of what the industry calls _"smart contracts"_.

Today's system revolves around a predefined set of [built-in system actors](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors). Collectively, these actors implement Filecoin-specific behaviour, such as storage provider management, sector tracking, protocol rewards, power accounting, storage markets, etc.

There's just one problem: users are unable install and run new _actor_ types. This results in Filecoin lacking general programmability.

This repo contains research and design material for the FVM: a revision of the VM subsystem of the Filecoin network in order to enable the deployment of user-defined actors to the Filecoin network.

# Filecoin implementations

For the purposes of this document, we consider four Filecoin node implementations:

1. [Lotus](https://github.com/filecoin-project/lotus) (written in Go; reference implementation), own actors implementation.
2. [Forest](https://github.com/ChainSafe/forest) (written in Rust), own actors implementation.
3. [Venus](https://github.com/filecoin-project/venus) (written in Go; former go-filecoin implementation), reuses Lotus actors via go import.
4. [Fuhon](https://github.com/filecoin-project/cpp-filecoin) (written in C++), reuses Lotus actors via FFI.

# Motivations

1. **Proliferation of innovative solutions in user space.** Giving users the ability to program new behaviours on top of the existing Filecoin primitives will unlock degrees of freedom, potential for innovation, and composability/stacking of primitives to form innovative solutions in a DeFi fashion.
2. **Lower dependence on system actor evolution.** Features that would otherwise require changes to system actors could now be implemented in a trustless manner in user space.
3. **Unlocking layer 2 solutions.** Currently layer 2 solutions can only exist as sidechains, but it's impossible to create 
5. **Universally executable spec / "Code is Law".** A single version of system actors running in a deterministic environment enables all clients to univocally arrive to consensus. (Counterpoint: This also poses some risks)
6. **Quicker protocol evolution.** Protocol upgrades can be implemented once. Rollout of FIPs will no longer bottleneck on finishing implementation across clients. Less coordination needed across implementers.
7. **Basis for computation-over-data.** There's good reason to believe that IPLD-ready WASM-based VMs are a useful stepping stone to enable computations
8. **Enabling governance-driven automatic protocol upgrades**. As more elements of the Filecoin protocol migrate to WASM space (e.g. block validation, fork choice rule, etc.), it becomes possible to deploy protocol changes as WASM modules to all clients, conditional upon on-chain voting.

# Requirements

1. Must support multiple VM runtimes, making it possible to deploy EVM bytecode (contracts likely written in Solidity) with full execution parity in 95% of cases, as well as contracts written natively and specifically for Filecoin.
2. Must enable interfacing seamlessly with built-in system actors.
3. Must support calls across contracts targeting different VM runtimes.
4. Must support synchronous calls, and should eventually support asynchronous calls.
6. Must be IPLD-native, handling all IPLD access patterns efficiently.
7. Must track reachability of IPLD objects upon state mutations to prevent indiscriminate state growth.
8. Must account for real computation costs (high-fidelity gas metering).
9. User code must not prevent system messages from executing (TBD).
10. Should support formalising interface contracts (methods, arguments, return values) with IPLD schemas.
11. Must be simple, approachable, and enjoyable to program for.

# Risks

1. **State size.** Without rent/expiration, gas will need to be carefully tuned to avoid a further explosion of state size. Furthermore, consideration will have to be taken for state expiration (or we'll have to design for it up-front?.
2. **Chain bandwidth.** Without sharding, we'll likely need to implement multiple "lanes" (as we've discussed before) to ensure that user actors don't clog necessary traffic like deal making.
3. **Performance degradation.** If we compile our actors to WASM, execution time could increase.