# Analysis of the state-of-the-art

> _As of Fall 2021._

This document enumerates other WASM-related resources that are relevant to FVM, and analyzes them from the viewpoint of prior art.

## WASM resources

- Wasmer.io
- eWASM

### [eWASM](https://ewasm.readthedocs.io/en/mkdocs/)

Some interesting considerations for Blockchain VMs. Work seems to have stalled in 2019.

- [WASM syscall interface](https://github.com/ewasm/ewasm-rust-api/blob/master/src/lib.rs)

## WASM-based VMs in other blockchains

- [NEAR](https://docs.near.org/docs/develop/contracts/overview), including [Aurora](https://github.com/aurora-is-near/aurora-engine) (EVM for NEAR)
- Polkadot/Substrate
- [CosmWasm](https://cosmwasm.com/) (Cosmos)
- [SVM](https://spacemesh.io/blog/spacemesh-virtual-machine-svm/) (Spacemesh)
- [Arwen VM](https://docs.elrond.com/technology/the-arwen-wasm-vm/) (Elrond)
- [Aqua VM](https://doc.fluence.dev/docs/concepts) (Fluence)
- [Mokoto](https://dfinity.org/faq/why-does-motoko-compile-to-webassembly/) (Dfinity)
- [ETH2 execution environments/engines](https://ethresear.ch/t/eth-execution-environment-proposal/5507) -- appears stangant.
- \<more>

### [Substrate](https://substrate.dev/docs/en/knowledgebase/smart-contracts/contracts-pallet)

Supports WASM and the EVM, but with separate VMs. It also supports cross-VM calling.

### [NEAR](https://github.com/near)

WASM based VM, but they're adding EVM support with [project Aurora](https://github.com/aurora-is-near/aurora-engine/).

Has an async (and parallel?) calling convention. [Calling](https://docs.near.org/docs/tutorials/contracts/cross-contract-calls) is entirely based on callbacks and promises. Of course, it does all this without closures so it's more like erlang than javascript.

- [WASM syscall interface](https://github.com/near/near-sdk-rs/blob/master/sys/src/lib.rs)

