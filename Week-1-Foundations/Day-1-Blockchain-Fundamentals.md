# Day 1 — How Blockchains Work (Deep Fundamentals)

> **Time:** 6–8 hours  
> **Goal:** Understand blockchain from cryptographic first principles — not surface level.

---

## Part 1: Theory (2 hours)

### 1.1 What is a Blockchain? (Real Analogy)

Imagine a shared Google Doc that:
- **No one can delete rows** — you can only append
- **Every row references the previous** — tampering breaks the chain
- **Everyone has a full copy** — no single point of failure
- **Rows are verified by math**, not by trust in a company

That is a blockchain. The "math" is cryptographic hashing.

---

### 1.2 Cryptographic Hashing

A hash function takes **any input → fixed-length output**. Properties:
- **Deterministic** — same input always gives same output
- **One-way** — you can't reverse the hash to find the input
- **Avalanche effect** — changing one character completely changes the output
- **Collision resistant** — two different inputs producing the same hash is computationally infeasible

**Ethereum uses Keccak-256 (SHA-3 variant)**

```
"Hello" → keccak256 → 0x06b3dfaec148fb1bb2b066f10ec285e7c9bf402ab32aa78a5d38e34566810cd2
"hello" → keccak256 → 0x1c8aff950685c2ed4bc3174f3472287b56d9517b9c948127319a09a7a36deac8
```

These are completely different. This is the **avalanche effect**.

#### Why this matters for blockchain:
- Each block contains the **hash of the previous block**
- If you tamper with block 50, its hash changes
- Block 51's "previous hash" no longer matches — the chain breaks
- You'd need to re-mine every subsequent block, which is computationally impossible

---

### 1.3 Block Structure

```
Block #12345
┌─────────────────────────────────────────┐
│ Block Header                             │
│  ├── Previous Block Hash: 0x4a7f...     │
│  ├── Merkle Root: 0xab12...             │
│  ├── Timestamp: 1699000000              │
│  ├── Nonce: 8394729                     │
│  ├── Difficulty Target: 0x000000...     │
│  └── Block Number: 12345               │
├─────────────────────────────────────────┤
│ Transaction List (Body)                  │
│  ├── Tx1: Alice → Bob, 1 ETH           │
│  ├── Tx2: Bob → Carol, 0.5 ETH         │
│  └── Tx3: Deploy Contract 0xABC...     │
└─────────────────────────────────────────┘
```

**The block hash = keccak256(entire block header)**

---

### 1.4 Merkle Trees — The Transaction Index

A Merkle tree lets you **prove a transaction is in a block** without downloading the entire block.

```
          Merkle Root
          /          \
       Hash(AB)     Hash(CD)
       /    \        /    \
   Hash(A) Hash(B) Hash(C) Hash(D)
     |        |       |       |
    Tx A     Tx B   Tx C    Tx D
```

**Why important:**
- SPV wallets (like MetaMask in some modes) only need block headers
- A Merkle proof proves Tx A is included using just: Hash(B), Hash(CD), Root
- Tampering with any transaction changes all hashes up to the root

---

### 1.5 Digital Signatures (ECDSA)

Every Ethereum account has:
- **Private key** — 256 random bits. **NEVER share this.**
- **Public key** — derived mathematically from private key (secp256k1 curve)
- **Address** — last 20 bytes of keccak256(public key)

```
Private Key (256 bits) 
    → [secp256k1 elliptic curve multiplication]
    → Public Key (512 bits)
    → [keccak256 hash]
    → Public Key Hash (256 bits)
    → [take last 160 bits]
    → Ethereum Address (0x...)
```

**Signing a transaction:**
```
Signature = ECDSA.sign(keccak256(transaction_data), private_key)
```

**Verifying:**
```
recovered_address = ECDSA.recover(transaction_data, signature)
if recovered_address == sender_address → valid!
```

No one can forge your signature without your private key. This is the math behind "only you can send from your wallet."

---

### 1.6 The Ethereum Virtual Machine (EVM)

The EVM is a **stack-based virtual machine** that executes smart contract bytecode.

```
Smart Contract (Solidity)
    → [solc compiler]
    → EVM Bytecode (hex)
    → Deployed to blockchain address
    → Every node runs it identically
    → Deterministic output
```

**EVM is sandboxed:**
- Cannot access the internet directly
- Cannot read files
- Can only access on-chain data and call other contracts

**EVM Storage Model:**
```
contract MyContract {
    // STORAGE — persisted on-chain, expensive (20,000 gas to set)
    uint256 public counter;
    mapping(address => uint256) public balances;
    
    function example(uint256 x) external {
        // MEMORY — temporary, cleared after function call, cheap
        uint256[] memory tempArray = new uint256[](10);
        
        // CALLDATA — read-only, where function arguments live
        // (x is in calldata)
        
        // STACK — max 1024 items, where computation happens
        uint256 result = x * 2;
        
        counter = result; // writes to STORAGE
    }
}
```

**Storage cost breakdown:**
| Operation | Gas Cost |
|-----------|----------|
| SSTORE (new slot) | 20,000 gas |
| SSTORE (update) | 2,900 gas |
| SLOAD | 2,100 gas |
| MSTORE (memory) | 3 gas |
| ADD | 3 gas |

---

### 1.7 Accounts: EOA vs Contract

**Externally Owned Account (EOA):**
- Has private key → controls the account
- Can initiate transactions
- Has ETH balance
- No code

**Contract Account:**
- Has code (bytecode) — runs on EVM
- Has ETH balance + storage
- Cannot initiate transactions — only responds to calls
- No private key

```
EOA (e.g., MetaMask wallet)
│
│ sends tx
▼
Contract Account (e.g., Uniswap)
│
│ internal call
▼
Another Contract Account (e.g., ERC-20 token)
```

---

## Part 2: Hands-On Exercises (2 hours)

### Exercise 1: Verify Keccak-256 Hashing

Go to https://emn178.github.io/online-tools/keccak_256.html

Try:
1. Hash your name — note the output
2. Change one character — observe the avalanche effect
3. Hash "0" vs "0 " (with a space) — completely different outputs

**Questions to answer:**
- Why can't you reverse a hash?
- Why does changing one character change the entire hash?

---

### Exercise 2: Explore a Real Block on Etherscan

Go to https://etherscan.io

1. Search for the latest block
2. Find the **Parent Hash** — click it, verify it matches the previous block's hash
3. Click any transaction — note: from, to, value, gas used, input data
4. Find the **Merkle Root** in the block details

**Write down:**
- Block number you explored
- How many transactions in it
- The gas used vs gas limit ratio

---

### Exercise 3: Decode a Raw Transaction

Go to https://etherscan.io/tx/0x5c504ed432cb51138bcf09aa5e8a410dd4a1e204ef84bfed1be16dfba1b22060
(First ever Ethereum transaction)

Decode what you see:
- From → To
- Value
- Gas price
- What was the purpose?

---

### Exercise 4: Calculate an Ethereum Address

Using Node.js:
```javascript
const { keccak256, toBytes } = require('viem')

// A public key (example, 64 bytes uncompressed, without 04 prefix)
const publicKey = '0x...' // paste a public key

// Ethereum address derivation
const hash = keccak256(publicKey)
const address = '0x' + hash.slice(-40) // last 20 bytes
console.log('Address:', address)
```

Install viem: `npm install viem`

---

### Exercise 5: Run a Local Blockchain

```bash
# Install Hardhat
mkdir blockchain-day1 && cd blockchain-day1
npm init -y
npm install --save-dev hardhat

# Start local blockchain
npx hardhat node
```

You'll see 20 pre-funded test accounts. In another terminal:
```bash
# Check balance of account 0
npx hardhat console --network localhost

# In the console:
const accounts = await ethers.getSigners()
const balance = await ethers.provider.getBalance(accounts[0].address)
console.log(ethers.formatEther(balance)) // Should be 10000 ETH
```

---

## Part 3: Deep Dive Reading (1 hour)

### Must-Read Articles Today

1. **[How does Ethereum work?](https://preethikasireddy.com/post/how-does-ethereum-work-anyway)** by Preethi Kasireddy
   - Best non-technical → technical explanation
   - Cover: state machine, EVM, gas

2. **[EVM Deep Dive Part 1](https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy)**
   - Stack, memory, opcodes
   - Read sections 1-3 today

3. **[Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)** (optional, skim)
   - The formal specification
   - Look at the block structure section

---

## Part 4: Reflection (30 min)

Answer these in your notes:

1. If I change one transaction in block #100, what exactly breaks and why?
2. Why can't a smart contract generate a random number by itself?
3. What's the difference between a hash and an encryption? Why does blockchain use hashing?
4. If Ethereum is "just a global computer," what is the EVM's CPU? RAM? Hard drive?
5. Why does deploying a contract cost more gas than calling it?

---

## Today's Key Takeaways

- Blockchain = append-only, cryptographically linked list of blocks
- Immutability comes from **hashing**, not from a database lock
- Merkle trees enable efficient transaction proofs
- ECDSA signatures prove ownership without revealing private key
- EVM is a deterministic, sandboxed virtual machine
- Storage is expensive; memory/calldata are cheap

---

## Resources for Day 1

| Resource | Link | Type |
|----------|------|------|
| Ethereum Whitepaper | https://ethereum.org/en/whitepaper/ | Reading |
| Ethereum Book (Ch 1-4) | https://github.com/ethereumbook/ethereumbook | Book |
| EVM Codes Reference | https://www.evm.codes/ | Reference |
| Keccak-256 Online | https://emn178.github.io/online-tools/keccak_256.html | Tool |
| Etherscan | https://etherscan.io | Explorer |
| Preethi's Ethereum Explained | https://preethikasireddy.com/post/how-does-ethereum-work-anyway | Article |
| EVM Deep Dive (Noxx) | https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy | Article |
| Hardhat Docs | https://hardhat.org/docs | Docs |

---

## Tomorrow

[Day 2 → Transactions, Gas, and the Mempool](./Day-2-Transactions-Gas-Mempool.md)
