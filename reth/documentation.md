# Privacy Enhancements for Reth: Confidential Storage and Transactions

### Introduction

Welcome to the documentation for the privacy enhancements added to **Reth**, the execution layer of the Ethereum blockchain. These enhancements introduce confidential storage and transaction capabilities, enabling developers to handle sensitive data securely within smart contracts and transactions. By building upon the existing Reth infrastructure, we've implemented changes to ensure ease of adoption and maintain compatibility.

This documentation highlights the differences and new features introduced, focusing on the modifications that make Reth privacy-aware. We recommend familiarizing yourself with the standard Reth documentation alongside this guide.

---

### Table of Contents

1. [Overall Changes](#1-overall-changes)
2. [Storage Modifications](#2-storage-modifications)
    - [2.1 Private Storage Flag](#21-private-storage-flag)
    - [2.2 State Root Calculation](#22-state-root-calculation)
    - [2.3 `eth_storageAt` RPC Modification](#23-eth_storageat-rpc-modification)
    - [2.4 Storage Hashing Parallelization](#24-storage-hashing-parallelization)
3. [Transactions](#3-transactions)
    - [3.1 TEE Client and Arguments](#31-tee-client-and-arguments)
    - [3.2 `txSeismic` Transaction Type](#32-txseismic-transaction-type)
    - [3.3 RPC Method Changes](#33-rpc-method-changes)
    - [3.4 Performance Testing](#34-performance-testing)
4. [RPC Modifications](#4-rpc-modifications)
    - [4.1 Summary of Modified Endpoints](#41-summary-of-modified-endpoints)
5. [Backup Mechanism](#5-backup-mechanism)
6. [Testing](#6-testing)
7. [Future Considerations](#7-future-considerations)

---

### 1. Overall Changes

We have introduced several changes to make Reth privacy-aware, enabling private storage values and transactions. The key modifications include:

- **Private Storage Flag**: Added an `is_private` flag to all storage values, changing the value type from `U256` to `FlaggedStorage`.
- **EVM Inspectors**: Updated Reth to work with changes from `seismic-revm` and require REVM inspectors for execution.

---

### 2. Storage Modifications

#### 2.1 Private Storage Flag

Previously, storage values were of type `U256`. With the privacy enhancements, we've introduced a new type called `FlaggedStorage`, which includes an `is_private` flag to indicate whether a storage value is confidential.

- **Implementation**: This change aligns with modifications in `seismic-revm` ([Pull Request #9](https://github.com/SeismicSystems/seismic-revm/pull/9)) and requires the use of REVM inspectors ([Pull Request #1](https://github.com/SeismicSystems/seismic-revm-inspectors/pull/1)).

#### 2.2 State Root Calculation

- **Modification**: The `is_private` flag is **not** encoded during the state root calculation. This decision is reflected in the code [here](https://github.com/SeismicSystems/seismic-reth/pull/4/commits/5a69f1ea359d3f4e95dd6a825e604548b0e11579#diff-a69280a7601140010b48c98e07c58431efd9e6f45180dcfcd2e0d423e4588a98R162).
- **Consideration**: We may want to include the `is_private` flag as part of the state since a storage slot can transition from public to private. This is an open point for future development.

#### 2.3 `eth_storageAt` RPC Modification

- **Behavior**: Modified the `eth_storageAt` RPC method to handle private storage.
    - If `is_private` is `true`, the RPC call returns `0`.
- **Rationale**:
    - **Prevent Information Leakage**: Since storage can transition from private to public, exposing the storage type could leak information through enumeration.
    - **Potential Misleading Data**: Returning `0` might be misleading if there is a value being stored. Developers should be aware of this behavior.
- **Code Reference**: [Commit](https://github.com/SeismicSystems/seismic-reth/pull/4/commits/f26de3b8ff74a4b23de0df548c8b629c2479d907)
- **Impact**: For a complete set of code paths affected, refer to all places where `encode_fixed_size()` is called.

#### 2.4 Storage Hashing Parallelization

- **Modification**: We include the `is_private` flag along with `addr_key` as the key instead of combining it with the value during parallelization of the `StorageHashingStage`.
- **Code Reference**: `seismic-reth/crates/stages/stages/src/stages/hashing_storage.rs:106`

---

### 3. Transactions

To support private transactions, we've made several modifications:

#### 3.1 TEE Client and Arguments

- **Addition**: Implemented a Trusted Execution Environment (TEE) client and arguments to interact with a server for decryption and encryption tasks.
- **Functionality**: Decryption occurs when the EVM initializes with the corresponding transaction, ensuring that the input data remains confidential until execution.

#### 3.2 `txSeismic` Transaction Type

- **Definition**: Introduced `txSeismic`, which defines fields for seismic transactions. In this transaction type, only the `input` field is encrypted.
- **Purpose**: Enables the handling of encrypted transaction data within the EVM.

#### 3.3 RPC Method Changes

- **Modified Methods**:
    - `eth_sendTransaction`
    - `eth_sendRawTransaction`
    - `eth_call`
- **Changes**:
    - Updated to support the new `txSeismic` transaction type.
    - `eth_call` is modified similarly to `trace_rawTransaction`, requiring a signature for execution.

#### 3.4 Performance Testing

We conducted end-to-end tests for the above changes. The performance metrics are as follows:

| **Block Time with HTTP Request** | **0 Calls**    | **1400 Calls**  | **5200 Calls**    |
|----------------------------------|----------------|-----------------|-------------------|
| **1400 Normal Transactions**     | 2.018 seconds  | 5.273 seconds   | 10.257 seconds    |
| **1400 Encrypted Transactions**  | 6.601 seconds  | 11.523 seconds  | 21.790 seconds    |

- **Observation**: The HTTP call latency contributes approximately **40%** of the total latency.
- **Note**: These tests include end-to-end scenarios, demonstrating the overhead introduced by the encryption and decryption processes.

---

### 4. RPC Modifications

#### 4.1 Summary of Modified Endpoints

We have modified several RPC endpoints to support privacy features:

- **Modified RPC Methods**:
    - **`eth_storageAt`**:
        - Returns `0` for private storage slots.
        - **Modification Location**: [Code Reference](https://github.com/SeismicSystems/seismic-reth/pull/4/commits/f26de3b8ff74a4b23de0df548c8b629c2479d907)
    - **`eth_sendTransaction`**:
        - Supports the new `txSeismic` transaction type with encrypted input.
    - **`eth_sendRawTransaction`**:
        - Updated to handle encrypted transactions.
    - **`eth_call`**:
        - Modified to require a signature, similar to `trace_rawTransaction`, to support encrypted data.

#### 4.2 SeismicAPI RPC Endpoints

- **Addition**: We have added SeismicAPI RPC endpoints for potential future use.
- **Current Status**: While these code paths are tested, we have decided to modify existing Ethereum RPC endpoints for faster development and to reduce code redundancy at this stage.

---

### 5. Backup Mechanism

- **Feature**: Reth will save the database state every time it reaches a certain canonical block production, controlled by the `BACKUP_SLOTS` parameter.
- **Consideration**: This feature requires further specification depending on how the consensus layer interacts with Reth.
- **Purpose**: Enables state snapshots at defined intervals, which can be crucial for recovery and auditing.

---

### 6. Testing

To ensure the integrity of the privacy enhancements, you can run end-to-end tests using the following command:

```bash
RUST_BACKTRACE=1 RUST_LOG=trace cargo test --package reth-node-ethereum --test e2e -- backup::backup --exact --show-output --nocapture
```

This command runs the `e2e` testing crate with block production, allowing you to verify the privacy features in a full execution context.

**Note**: Two lock-file tests may fail due to [Issue #9381](https://github.com/paradigmxyz/reth/issues/9381).

Additionally, we've modified:

- `seismic-reth/crates/storage/provider/src/writer/mod.rs:731` to verify that the `is_private` flag is committed to the database from the EVM execution outcome.
- `seismic-reth/crates/trie/trie/src/state.rs:347` to ensure that the `is_private` flag is propagated from `hashedStorages` to `postHashedStorages`.

---

### 7. Future Considerations

There are several areas that require attention and potential future development:

1. **Witness Auditing**:
    - **Action**: The `witness()` function needs to be audited to ensure it correctly handles private data.
    - **Importance**: To prevent potential leaks or mishandling of confidential information.

2. **Chain ID and Specification Changes**:
    - **Action**: Update Seismic's chain ID and related chain specifications.
    - **Reason**: To align with the privacy enhancements and ensure proper network identification.

3. **State Root Inclusion of `is_private` Flag**:
    - **Consideration**: Including the `is_private` flag in the state root calculation may be necessary to accurately represent the state where storage slots can transition between public and private.

4. **RPC Method Enhancements**:
    - **Encrypted Events and Data**: Future improvements may include supporting encrypted events, enabling the emission of confidential data without compromising privacy.
