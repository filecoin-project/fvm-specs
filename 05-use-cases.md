# Use Cases

Potential use cases for user-generated Filecoin actors in the Filecoin Virtual Machine (FVM), suggested by the ecosystem.

**Contents**

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Multi-generational storage](#multi-generational-storage)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Multi-generational storage

Suggested by Alex Fox - Neonix <alex.fox@neonix.io>

In the 1960's, the British Broadcasting Company (BBC), erased master copies of episodes of the popular television series Doctor Who. In the blink of an eye, this culturally significant data was lost to history. This story is repeated countless times -- no matter if the data is held by an individual, an organization, or a government. How can we guarantee reliable archival storage over periods of hundreds to thousands of years, or beyond?

What if we could outsource responsibility of storing and maintaining humanity's most important data to professional Storage Providers (SPs) who are continuously incentivized to persist the data on reliable storage mediums.

With a smart contract, we could lock a FIL balance as a bounty. The bounty is distributed over time to multiple Storage Providers (SPs) who prove that they have a copy of the data. New SPs can begin to claim a share of the bounty at any time. The deflationary nature of the FIL token should ensure that the bounty remains lucrative in future.

raulk (Filecoin Slack): "I would imagine that storage providers would claim portions of the bounty by presenting the sector ID and a merkle proof that traces the commP (deal data) to the commR of the sector. The contract would then call the miner actor to verify that the sector is alive, healthy and is being consistently proven."

