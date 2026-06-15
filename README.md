#  Fork-Free High-Throughput PoS Consensus 
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

The Consensus Engine operates as a deterministic state machine driven by time-based **Slots**. It ensures that all validators agree on the sequence of blocks, valid transactions, and the global state root.

### 4.1 Validator Scheduling & VRF
- **Slot Time**: The network progresses in discrete time segments called "Slots". In the current architecture, a slot lasts precisely 2 seconds (2000ms). It is logically divided into a **Primary Proposal Window** (the first 1000ms) and a **Backup/Propagation Window** (the remaining 1000ms).
- **VRF Sortition**: To decide which validator has the right to propose a block in a given slot, a Verifiable Random Function (VRF) is simulated. A global `SEED` ("genesis_seed") is hashed iteratively using `keccak256`. 
  - For Slot `N`, the `SEED` is hashed `N` times.
  - The first 8 bytes of the resulting hash are converted to an integer.
  - Modulo arithmetic (`integer % TOTAL_VALIDATORS`) selects the index of the **Primary Proposer**.
  - A secondary index (`(index + 1) % TOTAL_VALIDATORS`) selects the **Backup Proposer**.

### 4.2 Exact Lifecycle of a Slot

When a new slot begins (`startNextSlot()`), validators check their public keys against the schedule:

**Phase 1: Primary Proposal (0ms - 1000ms)**
1. **Mempool Processing**: If a node determines it is the **Primary Proposer**, it instantly pulls all valid pending transactions from its `Mempool`.
2. **State Simulation**: The node clones the global `StateManager`. It iterates through the transactions, deducts `amount + fee` from senders, increments senders' `nonces`, adds `amount` to recipients, and adds `fee`s to its own balance.
3. **Block Creation**: The node calculates the new `stateRoot` from this simulated MPT, creates a new `Block` object containing the transactions and the `stateRoot`, signs the block hash with its BLS private key, and broadcasts a `BLOCK_PROPOSED` message to the P2P network.

**Phase 2: Backup Recovery (1000ms - 2000ms)**
- If the Primary is offline or partitioned, the other nodes will wait for precisely the first half of the slot (1000ms). 
- Upon timeout, a `[TIMEOUT] Primary missed slot!` event triggers. 
- The designated **Backup Proposer** then executes the exact same steps (Mempool Processing -> State Simulation -> Block Creation), flagging the block as `isBackup = true`, and broadcasts it to save the slot.

**Phase 3: Slot Failure (> 2000ms)**
- If the full 2000ms slot duration expires without a block (primary or backup) being finalized, the slot is considered entirely dead. 
- A "Skipped Slot" dummy block is appended locally to advance the chain height, and the network moves to the next slot to prevent stalling.

### 4.3 Validation & Voting Process
When any validator receives a `BLOCK_PROPOSED` message, they run `validateProposal(block)`:
1. **Sanity Checks**: Verify block height aligns with their local chain length and the proposer matches the VRF schedule.
2. **Transaction Replay**: The receiving node clones its own local `StateManager` and replays every transaction in the block.
   - It strictly rejects the block if any sender's `nonce` does not perfectly match the MPT.
   - It strictly rejects the block if any sender's balance is `< amount + fee`.
3. **State Root Verification**: After replaying, the node calculates its own local `stateRoot`. If it does not perfectly match the `stateRoot` proposed in the block, the block is maliciously altered and rejected.
4. **Voting**: If valid, the node signs the block hash with its BLS private key and broadcasts a `PARTIAL_SIGNATURE` message.

### 4.4 BLS Signature Aggregation & Commit
- Validators accumulate incoming `PARTIAL_SIGNATURE` messages in the `pendingBlock`'s signature map.
- During every network loop, `tryAggregateAndCommit()` is executed.
- It counts the number of collected signatures. If `signatures >= THRESHOLD` (e.g., 3 out of 4 nodes):
  1. **Aggregation**: The `@noble/bls12-381` library mathematically adds the individual BLS signatures together to create a single, compact `aggregatedSignature`.
  2. **Commit**: The block is appended to the local `chain` array.
  3. **State Application**: The transactions are permanently applied to the global `StateManager`.
  4. **Next Slot**: The node immediately terminates the current slot timers and advances to the next slot early.

### 4.5 Transaction Fees
- Transaction fees are not arbitrarily minted; they are transferred directly from the sender to the node that successfully proposed the block. This logic is embedded directly into the State Simulation and Replay validation steps described above.

### 4.6 Probabilistic Finality
- The blockchain implements a `K_CONFIRMATIONS` constant (e.g., `K=5`).
- While a block is structurally sound once it has threshold signatures, true network **Finality** is declared via the `_updateFinality()` hook.
- A block at `Height X` is only marked as `FINAL` when the chain reaches `Height X + 5`. This ensures that even in the event of extreme network partitions, the finalized block cannot be reorganized out of the canonical chain.

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

The Consensus Engine operates as a deterministic state machine driven by time-based **Slots**. It ensures that all validators agree on the sequence of blocks, valid transactions, and the global state root.

### 4.1 Validator Scheduling & VRF
- **Slot Time**: The network progresses in discrete time segments called "Slots". In the current architecture, a slot lasts precisely 2 seconds (2000ms). It is logically divided into a **Primary Proposal Window** (the first 1000ms) and a **Backup/Propagation Window** (the remaining 1000ms).
- **VRF Sortition**: To decide which validator has the right to propose a block in a given slot, a Verifiable Random Function (VRF) is simulated. A global `SEED` ("genesis_seed") is hashed iteratively using `keccak256`. 
  - For Slot `N`, the `SEED` is hashed `N` times.
  - The first 8 bytes of the resulting hash are converted to an integer.
  - Modulo arithmetic (`integer % TOTAL_VALIDATORS`) selects the index of the **Primary Proposer**.
  - A secondary index (`(index + 1) % TOTAL_VALIDATORS`) selects the **Backup Proposer**.

### 4.2 Exact Lifecycle of a Slot

When a new slot begins (`startNextSlot()`), validators check their public keys against the schedule:

**Phase 1: Primary Proposal (0ms - 1000ms)**
1. **Mempool Processing**: If a node determines it is the **Primary Proposer**, it instantly pulls all valid pending transactions from its `Mempool`.
2. **State Simulation**: The node clones the global `StateManager`. It iterates through the transactions, deducts `amount + fee` from senders, increments senders' `nonces`, adds `amount` to recipients, and adds `fee`s to its own balance.
3. **Block Creation**: The node calculates the new `stateRoot` from this simulated MPT, creates a new `Block` object containing the transactions and the `stateRoot`, signs the block hash with its BLS private key, and broadcasts a `BLOCK_PROPOSED` message to the P2P network.

**Phase 2: Backup Recovery (1000ms - 2000ms)**
- If the Primary is offline or partitioned, the other nodes will wait for precisely the first half of the slot (1000ms). 
- Upon timeout, a `[TIMEOUT] Primary missed slot!` event triggers. 
- The designated **Backup Proposer** then executes the exact same steps (Mempool Processing -> State Simulation -> Block Creation), flagging the block as `isBackup = true`, and broadcasts it to save the slot.

**Phase 3: Slot Failure (> 2000ms)**
- If the full 2000ms slot duration expires without a block (primary or backup) being finalized, the slot is considered entirely dead. 
- A "Skipped Slot" dummy block is appended locally to advance the chain height, and the network moves to the next slot to prevent stalling.

### 4.3 Validation & Voting Process
When any validator receives a `BLOCK_PROPOSED` message, they run `validateProposal(block)`:
1. **Sanity Checks**: Verify block height aligns with their local chain length and the proposer matches the VRF schedule.
2. **Transaction Replay**: The receiving node clones its own local `StateManager` and replays every transaction in the block.
   - It strictly rejects the block if any sender's `nonce` does not perfectly match the MPT.
   - It strictly rejects the block if any sender's balance is `< amount + fee`.
3. **State Root Verification**: After replaying, the node calculates its own local `stateRoot`. If it does not perfectly match the `stateRoot` proposed in the block, the block is maliciously altered and rejected.
4. **Voting**: If valid, the node signs the block hash with its BLS private key and broadcasts a `PARTIAL_SIGNATURE` message.

### 4.4 BLS Signature Aggregation & Commit
- Validators accumulate incoming `PARTIAL_SIGNATURE` messages in the `pendingBlock`'s signature map.
- During every network loop, `tryAggregateAndCommit()` is executed.
- It counts the number of collected signatures. If `signatures >= THRESHOLD` (e.g., 3 out of 4 nodes):
  1. **Aggregation**: The `@noble/bls12-381` library mathematically adds the individual BLS signatures together to create a single, compact `aggregatedSignature`.
  2. **Commit**: The block is appended to the local `chain` array.
  3. **State Application**: The transactions are permanently applied to the global `StateManager`.
  4. **Next Slot**: The node immediately terminates the current slot timers and advances to the next slot early.

### 4.5 Transaction Fees
- Transaction fees are not arbitrarily minted; they are transferred directly from the sender to the node that successfully proposed the block. This logic is embedded directly into the State Simulation and Replay validation steps described above.

### 4.6 Probabilistic Finality
- The blockchain implements a `K_CONFIRMATIONS` constant (e.g., `K=5`).
- While a block is structurally sound once it has threshold signatures, true network **Finality** is declared via the `_updateFinality()` hook.
- A block at `Height X` is only marked as `FINAL` when the chain reaches `Height X + 5`. This ensures that even in the event of extreme network partitions, the finalized block cannot be reorganized out of the canonical chain.

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

The Consensus Engine operates as a deterministic state machine driven by time-based **Slots**. It ensures that all validators agree on the sequence of blocks, valid transactions, and the global state root.

### 4.1 Validator Scheduling & VRF
- **Slot Time**: The network progresses in discrete time segments called "Slots". In the current implementation, a slot consists of a `PROPOSAL_WINDOW` (5000ms) and a `PROPAGATION_WINDOW` (3000ms).
- **VRF Sortition**: To decide which validator has the right to propose a block in a given slot, a Verifiable Random Function (VRF) is simulated. A global `SEED` ("genesis_seed") is hashed iteratively using `keccak256`. 
  - For Slot `N`, the `SEED` is hashed `N` times.
  - The first 8 bytes of the resulting hash are converted to an integer.
  - Modulo arithmetic (`integer % TOTAL_VALIDATORS`) selects the index of the **Primary Proposer**.
  - A secondary index (`(index + 1) % TOTAL_VALIDATORS`) selects the **Backup Proposer**.

### 4.2 Exact Lifecycle of a Slot

When a new slot begins (`startNextSlot()`), validators check their public keys against the schedule:

**Phase 1: Primary Proposal (0ms - 5000ms)**
1. **Mempool Processing**: If a node determines it is the **Primary Proposer**, it instantly pulls all valid pending transactions from its `Mempool`.
2. **State Simulation**: The node clones the global `StateManager`. It iterates through the transactions, deducts `amount + fee` from senders, increments senders' `nonces`, adds `amount` to recipients, and adds `fee`s to its own balance.
3. **Block Creation**: The node calculates the new `stateRoot` from this simulated MPT, creates a new `Block` object containing the transactions and the `stateRoot`, signs the block hash with its BLS private key, and broadcasts a `BLOCK_PROPOSED` message to the P2P network.

**Phase 2: Backup Recovery (5000ms - 8000ms)**
- If the Primary is offline or partitioned, the other nodes will wait precisely `PROPOSAL_WINDOW_MS` (5 seconds). 
- Upon timeout, a `[TIMEOUT] Primary missed slot!` event triggers. 
- The designated **Backup Proposer** then executes the exact same steps (Mempool Processing -> State Simulation -> Block Creation), flagging the block as `isBackup = true`, and broadcasts it.

**Phase 3: Slot Failure (> 8000ms)**
- If another timeout of `PROPAGATION_WINDOW_MS` (3 seconds) expires without a backup block being finalized, the slot is considered entirely dead. 
- A "Skipped Slot" dummy block is appended locally to advance the chain height, and the network moves to the next slot to prevent stalling.

### 4.3 Validation & Voting Process
When any validator receives a `BLOCK_PROPOSED` message, they run `validateProposal(block)`:
1. **Sanity Checks**: Verify block height aligns with their local chain length and the proposer matches the VRF schedule.
2. **Transaction Replay**: The receiving node clones its own local `StateManager` and replays every transaction in the block.
   - It strictly rejects the block if any sender's `nonce` does not perfectly match the MPT.
   - It strictly rejects the block if any sender's balance is `< amount + fee`.
3. **State Root Verification**: After replaying, the node calculates its own local `stateRoot`. If it does not perfectly match the `stateRoot` proposed in the block, the block is maliciously altered and rejected.
4. **Voting**: If valid, the node signs the block hash with its BLS private key and broadcasts a `PARTIAL_SIGNATURE` message.

### 4.4 BLS Signature Aggregation & Commit
- Validators accumulate incoming `PARTIAL_SIGNATURE` messages in the `pendingBlock`'s signature map.
- During every network loop, `tryAggregateAndCommit()` is executed.
- It counts the number of collected signatures. If `signatures >= THRESHOLD` (e.g., 3 out of 4 nodes):
  1. **Aggregation**: The `@noble/bls12-381` library mathematically adds the individual BLS signatures together to create a single, compact `aggregatedSignature`.
  2. **Commit**: The block is appended to the local `chain` array.
  3. **State Application**: The transactions are permanently applied to the global `StateManager`.
  4. **Next Slot**: The node immediately terminates the current slot timers and advances to the next slot early.

### 4.5 Transaction Fees
- Transaction fees are not arbitrarily minted; they are transferred directly from the sender to the node that successfully proposed the block. This logic is embedded directly into the State Simulation and Replay validation steps described above.

### 4.6 Probabilistic Finality
- The blockchain implements a `K_CONFIRMATIONS` constant (e.g., `K=5`).
- While a block is structurally sound once it has threshold signatures, true network **Finality** is declared via the `_updateFinality()` hook.
- A block at `Height X` is only marked as `FINAL` when the chain reaches `Height X + 5`. This ensures that even in the event of extreme network partitions, the finalized block cannot be reorganized out of the canonical chain.

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
