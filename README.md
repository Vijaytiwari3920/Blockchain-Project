# Custom Blockchain: Technical Specifications

This document outlines the bit-by-bit technical details of the custom Proof-of-Stake (PoS) blockchain developed in this repository. The system transitions from an initial Python simulation to a fully functional JavaScript Peer-to-Peer (P2P) network using an Account-based ledger model inspired by Ethereum.

## 1. Cryptography

### 1.1 Hashing Algorithm
- **Algorithm**: `Keccak256` (Ethereum Standard)
- **Implementation**: The `js-sha3` library is utilized in the JS Node.
- **Usage**: Used extensively for hashing transactions, generating block hashes, computing Merkle Patricia Trie roots, and providing pseudo-random seeds for the consensus schedule.

### 1.2 Digital Signatures
- **Algorithm**: BLS Signatures (Boneh–Lynn–Shacham) on the BLS12-381 curve.
- **Implementation**: Uses `@noble/bls12-381`.
- **Key Features**:
  - **Deterministic Keypairs**: Private keys are deterministically generated from numeric seeds (e.g., Validator ID) for predictable local testing.
  - **Signature Aggregation**: The network can aggregate multiple validator signatures into a single, compact signature on a block, vastly reducing block size and verification overhead.

## 2. Core Data Structures

### 2.1 Transaction Model
The blockchain employs an **Account-Based Model** (unlike Bitcoin's UTXO model). 
A `Transaction` contains the following fields:
- `sender`: Public key of the sender.
- `recipient`: Public key of the receiver.
- `amount`: The value being transferred.
- `fee`: The tip given to the validator who proposes the block.
- `nonce`: A strictly increasing integer representing the number of transactions sent from a given address. Prevents replay attacks.
- `timestamp`: Epoch time of creation.
- `signature`: A BLS signature over the Keccak256 hash of all the fields above.

### 2.2 Block Header
A `Block` consists of:
- `height`: The sequence number in the chain.
- `prevHash`: The Keccak256 hash of the preceding block.
- `proposerPubKey`: The public key of the validator who proposed the block.
- `transactions`: An array of `Transaction` objects included in this block.
- `transactionsRoot`: The root hash of a Merkle Tree containing all transactions in the block.
- `stateRoot`: The root hash of the Merkle Patricia Trie representing the global account state *after* this block is applied.
- `partialSignatures`: A map of validator public keys to their individual BLS signatures.
- `aggregatedSignature`: The final aggregated BLS signature (set once the threshold is met).

## 3. State Management

### 3.1 Global State
- Accounts are mapped by their public keys.
- Each account stores a `balance` and a `nonce`.

### 3.2 Merkle Patricia Trie (MPT)
- **Purpose**: To provide a cryptographic, deterministic fingerprint (`stateRoot`) of the entire ledger state at any given block.
- **Implementation**: A simplified, custom in-memory implementation of Ethereum's MPT (`mpt.js`).
- **Mechanism**: Account public keys are converted to hex strings and broken into arrays of nibbles (half-bytes). The trie stores balances and nonces at leaf nodes, computing hashes recursively up to the root.

## 4. Consensus Engine

### 4.1 Validator Scheduling (VRF Simulation)
- The chain operates in fixed time **Slots**.
- A deterministic pseudo-random schedule is generated. A `SEED` is repeatedly hashed with Keccak256 to simulate a Verifiable Random Function (VRF).
- For every slot, a Primary Proposer and a Backup Proposer are selected via modulo math over the Validator pool.

### 4.2 Block Proposal & Aggregation
1. **Primary Turn**: The scheduled primary gathers transactions from its Mempool, verifies nonces and balances, computes the hypothetical new `stateRoot`, and broadcasts a `BLOCK_PROPOSED` message.
2. **Backup Turn**: If the primary is offline and misses the time window, the backup proposer steps in and proposes a recovery block.
3. **Voting**: Other validators receive the proposal, independently validate the transactions and `stateRoot`, and reply with a `PARTIAL_SIGNATURE`.
4. **Aggregation**: Once a node collects enough partial signatures to meet the predefined `THRESHOLD` (e.g., 3 out of 4 nodes), it aggregates them using BLS math and permanently adds the block to the chain.

### 4.3 Transaction Fees & Incentives
- When a block is successfully finalized, the block's proposer receives the sum of all `fee` fields from the transactions included in that block. 
- The fees are deducted from the senders' balances during state execution.

### 4.4 Finality
- The chain uses a `K_CONFIRMATIONS` rule.
- A block is considered highly probable, but true **Finality** is declared only when `K` (e.g., 5) subsequent valid blocks have been chained on top of it.

## 5. Network Architecture (P2P)

### 5.1 WebSockets
- The network relies on a full-mesh P2P topology using `ws` (WebSockets).
- Each Node acts as both a Server (listening for incoming peer connections) and a Client (dialing out to known peers).

### 5.2 Message Types
- `NEW_TRANSACTION`: Broadcasts a user transaction to all validators' mempools.
- `BLOCK_PROPOSED`: Broadcasts a block candidate for voting.
- `PARTIAL_SIGNATURE`: Broadcasts a validator's vote on a block.
- `REQUEST_STATE` / `STATE_UPDATE`: Used for initial node synchronization. When a node spins up late, it requests the full chain history from peers and rapidly catches up its local state.

## 6. Directory Breakdown

### `/sim` (Initial Prototype)
- Contains the initial Python proof-of-concept.
- Proved out the BLS aggregation math and consensus looping before migrating to JS.

### `/p2p-node` (Active Implementation)
- `crypto.js`: BLS signature wrappers and Keccak256 hashing.
- `mpt.js`: Custom Merkle Patricia Trie logic.
- `types.js`: Defines `Block` and `Transaction` classes.
- `state.js`: High-level wrapper for Account balances and nonces.
- `p2p.js`: WebSocket networking and broadcasting.
- `node.js`: The central Daemon. Initializes the P2P network, manages the Mempool, and runs the Consensus state machine loops.
- `test_cluster.js`: A comprehensive local integration test that spawns 4 node processes and injects transactions to verify end-to-end consensus.
