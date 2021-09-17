# Filecoin FVM

This is the home of the FVM project in Filecoin.

## Context and goals

Filecoin today lacks general programmability. As a result, it is not possible to deploy user-defined behaviour, or "smart contracts", to the blockchain.

The closest thing that Filecoin has is a **discrete** set of **embedded** smart contracts, denominated [system "actors"](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors). They provide the logic for elements like storage power accounting, deal-making, payment channels, scheduled execution, and more. But their functionality is hardcoded as per the specs.

The goal of this project is to add general programmability to the Filecoin blockchain. We predict this will unleash a profileration of new services and tools that can be built and deployed to the Filecoin network, without requiring network upgrades, involvement from core implementation maintainers, changes in the embedded actors, or spec alterations.

Smart contracts on Filecoin can bring huge benefits to both clients and providers — from unlocking "repair providers" that automate the process of repeat storage deals for programmatic “fire and forget” storage, to on-chain storage onboarding contracts (a la programmatic Slingshot), to collective DataDAOs that fund/monetize data on Filecoin.

Furthermore, we aim for full EVM compatibility, allowing Filecoin to leverage the vast pool of assets, talent and tools that already exist in that ecosystem.

**FVM (Filecoin Virtual Machine)** is the name of this project, as well as the name of the execution environment for smart contracts on the Filecoin blockchain.

## About this effort

<details><summary>Click to expand</summary>
FVM unlocks major new network capabilities without requiring network upgrades, core dev implementation work, or any cross-team coordination - helping increase the network’s iteration speed. However it will also *add* complexity to the protocol and needs a lot of design work to get it right.

We acknowledge that significant exploration/prototyping is necessary before ready to land. While this work is initiated by Protocol Labs, we rely on the vibrant Filecoin community to engage continuously, collaborate around ideas and designs, implement prototypes, test preview releases, build on it, come up with tooling, and ultimately, collectively own it and extend it.

Note: landing FVM will likely also have significant network scalability impacts as well that will need to be mitigated.
</details>

## About this repo

<details><summary>Click to expand</summary>
This repo acts as an entrypoint, hosting notes, design proposals, product ideas, and other documents related to this proejct.

Code and prototypes will usually be hosted in separate repos, linked from here for discovery.

This repo will incubate the [FIP (Filecoin Improvement Proposal)](https://github.com/filecoin-project/FIPs) that shall formally introduce this capability into the network.
</details>

## Content index

> ⚠️  These documents are being drafted.

- [`00-introduction.md`](./00-introduction.md): summarizes vision, goals, assumptions, and requirements of this project.
- [`01-technical-architecture.md`](./01-technical-architecture.md): details the proposed architecture.
- [`02-state-of-the-art.md`](./02-state-of-the-art.md): analyses the state of the art concerning WASM+Blockchain, and concrete case studies of programmability in other blockchains.
- [`03-impl-exploration.md`](./03-impl-exploration.md): notes on potential implementation.
- [`04-evm-compat.md`](./04-evm-compat.md): EVM compatibility proposals and notes.
- [`05-use-cases.md`](./05-use-cases.md): catalogue of brainstormed use cases the FVM should enable at some stage.
- [`06-scalability-considerations.md`](./06-scalability-considerations.md): discusses the scalability considerations concerning state size, state churn, message/transaction throughput, etc.

## Experiments

- [FVM Runtime Experiment](https://github.com/filecoin-project/fvm-runtime-experiment)

## License

Dual-licensed: [MIT](./LICENSE-MIT), [Apache Software License v2](./LICENSE-APACHE), by way of the
[Permissive License Stack](https://protocol.ai/blog/announcing-the-permissive-license-stack/).
