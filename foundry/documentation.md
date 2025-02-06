# Seismic Developer Tools

The Seismic Foundry Developer Tools extend the existing Foundry suite by providing specialized binaries (`sforge` and `sanvil`) that support shielded/Seismic smart contract development. These tools integrate advanced functionalities—including a dedicated Solidity compiler (`ssolc`) and an enhanced EVM (SEVM)—that natively support shielded transactions and private storage.

---

## Table of Contents
1. [Overview](#1-overview)
2. [Changes](#2-changes)
   - [2.1](#21-sforge-changes)
   - [2.2](#22-sanvil-changes)
3. [Technical Details](#3-technical-details)
   - [3.1](#31-private-storage-implementation-in-sanvil)
   - [3.2](#32-development-vs-production)

---

## 1. Overview

The Seismic Foundry suite consists of two main components: 
- `sforge`: A specialized testing framework for shielded contracts, an extension of [`forge`](https://book.getfoundry.sh/forge/)
- `sanvil`: A local Seismic node supporting private storage and transactions, an extension of [`anvil`](https://book.getfoundry.sh/anvil/)

---

## 2. Changes

### 2.1 sforge
The following changes are made in order to support the development, testing and deployment of shielded contracts using `sforge`:
- In the overall config (`config/src/lib.rs`), we add the [`seismic`](https://github.com/SeismicSystems/seismic-foundry/blob/b05fd187442241ab47ec992c44a105c2ee97f5a8/crates/config/src/lib.rs#L507) flag in order to use the Seismic Solidity compiler (`ssolc`). This flag is set to `true` by default.
- The flag, when set to `true`, will use the Seismic Solidity compiler (`ssolc`) situated at `/usr/local/bin/ssolc`. 
Developers are hence required to have the `ssolc` binary installed at this location before using `sforge`. The latest `ssolc` binary is automatically installed when using `sfoundryup`.

### 2.2 sanvil
The following changes are made in order to support deployment and testing of shielded contracts and transactions using `sanvil`:
- Private storage and transaction support is the same as in [`seismic-reth`](https://github.com/SeismicSystems/documentation/blob/main/reth/documentation.md), in order to make the node privacy-aware and support shielded transactions.
- Since `sanvil` is a local node, encryption/decryption occurs natively within the node, instead of within a TEE as is done in `seismic-reth`. 
- Both `sanvil` and `sforge` are configured to use the Seismic EVM (`SEVM`) version specified in the [`seismic_version`](https://github.com/SeismicSystems/seismic-foundry/blob/b05fd187442241ab47ec992c44a105c2ee97f5a8/crates/config/src/lib.rs#L217) field in the overall config (`config/src/lib.rs`) if set.
---


## 3. Technical Details

### 3.1 Private Storage Implementation in `sanvil`
- Native support for shielded transactions and private storage
- In development environments, encryption/decryption occurs within the node.
- Functionally equivalent to `seismic-reth`, but without TEE requirements.

### 3.2 Development vs Production
- Development: Encryption/decryption performed natively by `sanvil`
- Production: Encryption/decryption handled by TEEs in `seismic-reth`
- API compatibility maintained between environments

---

---
