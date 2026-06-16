# Technical Architecture & Details 

This document outlines the complete technical architecture of the custom Proof-of-Stake (PoS) Blockchain, including its consensus mechanisms, EVM smart contract execution, transaction handling, and cryptographic primitives.

## 1. Core Architecture Overview
The blockchain is designed as a standalone Layer-1 network simulation built in Python. It heavily focuses on a modular architecture, separating consensus logic (VRF & Heaviest Chain) from state execution (Ethereum Virtual Machine). 

**Key Components:**
- **Consensus Engine**: Handles Proof-of-Stake validator sortition, fork-choice rules, and block validation.
- **EVM Adapter**: A bridge to an active EVM state machine that natively executes Ethereum Smart Contracts.
- **Mempool & Transactions**: Collects and validates pending operations.
- **Storage Layer**: A persistent SQLite database for blockchain state.

---

## 2. Cryptography & Accounts (`crypto.py`)
The system uses asymmetric cryptography to secure accounts, sign transactions, and prove block authorship.

- **RSA Keypairs**: Each validator generates a unique 2048-bit RSA keypair.
- **EVM-Compatible Addresses**: To interact seamlessly with the EVM module, an Ethereum-style address (`0x` + 40 hex characters) is derived for every validator. The RSA public key is serialized, hashed using Ethereum's `keccak256`, and the last 20 bytes are extracted and checksummed.
- **Signatures**: Both Transactions and Blocks are cryptographically signed using `PKCS1_v1_5` with SHA-256 hashes to prevent tampering.

---

## 3. Consensus Engine: PoS + VRF (`consensus.py` & `validator.py`)
The consensus mechanism mimics modern scalable blockchains (like Algorand or Ethereum 2.0) by avoiding energy-intensive mining.

### Staking
- Accounts must submit a `STAKE` transaction to lock funds.
- Only accounts with active stakes are registered as valid block-producing nodes.

### Verifiable Random Function (VRF) Sortition
- ***Note: The VRF is currently a Mock implementation.***
- To prevent predictability, the network uses a VRF-style lottery to elect block proposers for each time slot.
- **Mechanism**: Every slot, each validator computes a pseudo-random hash using their private key and the `previous_block_hash`. 
- **Threshold**: If the resulting hash falls below a certain target threshold (e.g., 10% probability), the validator "wins" the right to propose the block.
- **Validation**: Other nodes verify the VRF proof to ensure the proposer didn't cheat the lottery.

### Fork Choice Rule (Heaviest Chain)
- Due to the nature of probabilistic VRF, multiple validators might win the lottery in the same slot, creating a fork.
- The network resolves forks using a **Heaviest Chain Rule**. The "weight" of a chain is calculated by accumulating the stakes of the validators who proposed the blocks.
- The engine dynamically tracks multiple branches and always sets the `heaviest_hash` tip to the chain with the most accumulated stake.

---

## 4. EVM & Smart Contracts (`evm_adapter.py`)
The blockchain natively supports Turing-complete Smart Contracts. Instead of reinventing a custom Virtual Machine, it embeds the official Ethereum Virtual Machine using `py-evm`, `eth-tester`, and `web3.py`.

### EVM Integration Workflow
1. **Solidity Compilation**: Smart contracts written in raw Solidity (`.sol`) are dynamically compiled into EVM bytecode using `py-solc-x`.
2. **Global State Singleton**: A persistent EVM state machine (`EVMAdapter`) is initialized and shared across the network to track balances, nonces, and contract storage.
3. **Transaction Routing**:
   - `DEPLOY` transactions forward the raw bytecode to the EVM via `w3.eth.send_transaction`. The adapter captures the resulting `receipt.contractAddress` and commits it to the blockchain.
   - `CALL` transactions allow validators to execute functions on deployed contracts by passing the method's `keccak256` function selector (e.g., `0x305f72b7` for `calls()`).
4. **State Transitions**: Every block that passes VRF and signature validation is processed, and its EVM-compatible transactions update the internal state machine cleanly. If an execution reverts, the exception is caught, and the state rollback is handled safely.

---

## 5. Blocks & Transactions (`block.py` & `transaction.py`)
### Transactions
Transactions represent state changes and require:
- `sender`: The EVM address of the sender.
- `receiver`: The destination address (or empty for contract deployments).
- `amount`: Token value transferred.
- `fee`: The transaction tip awarded to the block proposer.
- `tx_type`: Enum string representing the operation (`TRANSFER`, `STAKE`, `DEPLOY`, `CALL`).
- `data`: Hex-encoded calldata for smart contract execution.

### Blocks
Blocks securely batch transactions together. A block header contains:
- `height`: The chain index.
- `slot`: The time interval index.
- `timestamp`: Creation time.
- `prev_hash`: Cryptographic link to the parent block.
- `proposer_pub_key` & `evm_address`: The identity of the node that mined the block.
- `vrf_proof`: The cryptographic lottery ticket verifying the proposer's right to mine.

---

## 6. Storage & Database Persistence
To ensure node resilience and fast restarts, the blockchain maintains persistent storage using `sqlite3`.

- **Database Initialization**: A local file (`chain.db`) is created to store serialized blocks.
- **Serialization**: Blocks are converted into JSON strings and stored alongside their cryptographic hashes and heights.
- **Node Restarts**: When a node goes offline and restarts, it queries the database, rebuilds the block tree in memory, recalculates the Heaviest Chain, and re-syncs the EVM state up to the current tip.

---

## 7. Next Steps & Future Work
While the architecture is robust and functionally complete for Layer-1 simulation, future enhancements to reach production-grade "real world" status include:
1. **Real VRF Implementation**: Upgrading the mock VRF to a standard cryptographic VRF (like `ECVRF` using elliptic curves).
2. **Economic Security & Slashing**: Implementing penalties (burning stakes) for nodes that propose invalid blocks or attempt to "Equivocate" (signing multiple competing blocks at the same height).
3. **P2P Networking**: Replacing the local in-memory simulation array with asynchronous TCP/UDP peer-to-peer sockets, allowing distributed nodes to gossip blocks and mempool transactions over a real network.
