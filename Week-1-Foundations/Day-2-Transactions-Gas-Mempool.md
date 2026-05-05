# Day 2 — Transactions, Gas & the Mempool

> **Time:** 6–8 hours  
> **Goal:** Understand the full lifecycle of a transaction — from the moment you click "Send" to finality on-chain.

---

## Part 1: Theory — Transaction Lifecycle (2 hours)

### 1.1 The Full Transaction Lifecycle

```
User signs tx in MetaMask
        │
        ▼
Transaction broadcast to network
(via RPC node — e.g., Alchemy/Infura)
        │
        ▼
Mempool (pending transaction pool)
(all validators see it, waiting to be picked up)
        │
        ▼
Validator selects tx (based on gas price priority)
        │
        ▼
EVM executes the transaction
        │
        ├── Success → state changes committed
        └── Revert → state reverted, gas consumed anyway
        │
        ▼
Transaction included in a block
        │
        ▼
Block propagated to other nodes
        │
        ▼
Other nodes validate and add to their chain
        │
        ▼
After ~12 seconds: 1 confirmation
After ~2-3 minutes: "safe" (~12 confirmations)
After ~15 minutes: "finalized" (absolute safety)
```

---

### 1.2 Anatomy of an Ethereum Transaction

A raw Ethereum transaction contains exactly these fields:

```javascript
{
  nonce: 42,           // How many txs this sender has sent (prevents replay attacks)
  to: "0xRecipient",  // Recipient (null = contract deployment)
  value: "1000000000000000000", // Amount in wei (1 ETH = 10^18 wei)
  data: "0x...",       // For contract calls: ABI-encoded function + args
  gasLimit: 21000,     // Max gas you're willing to use
  maxFeePerGas: "30000000000",  // EIP-1559: max total fee (gwei)
  maxPriorityFeePerGas: "2000000000", // EIP-1559: tip to validator
  chainId: 1,          // Prevents replay on other chains
  // --- Added by signing ---
  v: 28,               // Recovery id (part of ECDSA signature)
  r: "0x...",          // ECDSA signature r
  s: "0x..."           // ECDSA signature s
}
```

**Nonce is critical:** If you send tx with nonce 42, then nonce 43 — you can't submit nonce 45 until 43 is mined. This creates **stuck transactions** (a common Web3 bug).

---

### 1.3 Gas — The Fuel of Ethereum

**Why gas exists:**
The EVM is a shared computer. Without a cost per operation, someone could write an infinite loop and halt every node on the network. Gas prevents this.

**Gas Analogy:**
Think of gas like the fuel in a car:
- `gasLimit` = size of your fuel tank (max you'll spend)
- `gasPrice` = cost per liter of fuel
- Actual cost = `gasUsed × gasPrice`
- Unused gas is refunded
- If you run out of gas mid-execution → transaction reverts, **but you still pay for gas used**

**Gas Costs for Common Operations:**
```
ETH Transfer:           21,000 gas (fixed)
ERC-20 Transfer:        ~65,000 gas
Contract Deployment:    ~500,000–2,000,000 gas
Uniswap V3 Swap:        ~150,000 gas
SSTORE (new slot):      20,000 gas
SSTORE (update):        2,900 gas
SLOAD:                  2,100 gas
ADD/MUL/DIV:            3–5 gas
SHA3 (keccak256):       30 gas + 6/word
```

---

### 1.4 EIP-1559 Gas Model (Post London Upgrade)

Before EIP-1559 (legacy): `fee = gasUsed × gasPrice` — you bid blindly.

After EIP-1559:
```
Total Fee = gasUsed × (baseFee + priorityFee)

- baseFee: algorithmically set by the protocol, BURNED (removed from supply)
- priorityFee (tip): goes to the validator
- maxFeePerGas: cap on what you'll pay total
```

**Base fee adjustment:**
- Block > 50% full → baseFee increases (up to +12.5%)
- Block < 50% full → baseFee decreases
- Creates predictable fee market

**What gets burned:** The baseFee is permanently removed from ETH supply. This makes ETH deflationary during high-usage periods. Check: https://ultrasound.money

```javascript
// Example: checking current gas price with ethers.js
const provider = new ethers.JsonRpcProvider("YOUR_ALCHEMY_URL")
const feeData = await provider.getFeeData()

console.log("Base fee:", ethers.formatUnits(feeData.gasPrice, "gwei"), "gwei")
console.log("Max fee per gas:", ethers.formatUnits(feeData.maxFeePerGas, "gwei"), "gwei")
console.log("Priority fee:", ethers.formatUnits(feeData.maxPriorityFeePerGas, "gwei"), "gwei")
```

---

### 1.5 The Mempool — Where Transactions Wait

The mempool (memory pool) is the **waiting room** for unconfirmed transactions.

**Key properties:**
- Every node has its own mempool (not a single global pool)
- Transactions propagate peer-to-peer across the network
- Validators pick transactions to include based on **priority fee** (highest tips first)
- Mempool has a size limit — very low-fee txs get dropped

**Mempool states:**
```
Pending:  tx is in mempool, waiting to be included
Queued:   tx has a nonce gap (e.g., nonce 45 but nonce 43 not yet mined)
Dropped:  tx was evicted (too low fee, or replaced)
```

**Transaction replacement:** You can replace a pending transaction by submitting a new one with the **same nonce** but a **10%+ higher gas price**. This is how "speed up" and "cancel" in MetaMask work.

```javascript
// Cancel a stuck transaction: send 0 ETH to yourself with same nonce, higher gas
const cancelTx = {
  nonce: stuckNonce,        // same nonce
  to: myAddress,            // to yourself
  value: 0,                 // 0 ETH
  gasLimit: 21000,
  maxFeePerGas: currentMaxFee * 1.2,  // 20% higher
  maxPriorityFeePerGas: currentTip * 1.2
}
await signer.sendTransaction(cancelTx)
```

---

### 1.6 Transaction Types

**Type 0 (Legacy):** Pre-EIP-1559, single `gasPrice` field
**Type 1 (EIP-2930):** Adds access lists for cheaper storage reads
**Type 2 (EIP-1559):** Current standard, `maxFeePerGas` + `maxPriorityFeePerGas`
**Type 3 (EIP-4844 — "blobs"):** For L2 data availability (Dencun upgrade, 2024)

---

### 1.7 Transaction Finality

**Confirmation ≠ Finality**

```
1 block (12s):   Probabilistic — could theoretically be reorged
12 blocks (2m):  "Safe" — very unlikely to reorg
64 blocks (12m): "Finalized" — Ethereum's checkpoint finality
                 (requires 2/3 validators to attest)
```

For high-value operations (DEX trades, large transfers), wait for **finality**.

---

## Part 2: Practical Coding (2 hours)

### Exercise 1: Send Your First Transaction with Ethers.js

```javascript
// install: npm install ethers dotenv
const { ethers } = require('ethers')
require('dotenv').config()

async function sendTransaction() {
  // Connect to Sepolia testnet via Alchemy
  const provider = new ethers.JsonRpcProvider(process.env.ALCHEMY_SEPOLIA_URL)
  
  // Load wallet from private key (NEVER hardcode in production!)
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider)
  
  console.log("Sender:", wallet.address)
  console.log("Balance:", ethers.formatEther(await wallet.provider.getBalance(wallet.address)), "ETH")
  
  // Get current gas prices
  const feeData = await provider.getFeeData()
  console.log("Base fee:", ethers.formatUnits(feeData.gasPrice, "gwei"), "gwei")
  
  // Send transaction
  const tx = await wallet.sendTransaction({
    to: "0xRecipientAddress",
    value: ethers.parseEther("0.001"),  // 0.001 ETH
    maxFeePerGas: feeData.maxFeePerGas,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas,
  })
  
  console.log("Transaction hash:", tx.hash)
  console.log("View on Etherscan:", `https://sepolia.etherscan.io/tx/${tx.hash}`)
  
  // Wait for 1 confirmation
  const receipt = await tx.wait(1)
  console.log("Gas used:", receipt.gasUsed.toString())
  console.log("Effective gas price:", ethers.formatUnits(receipt.gasPrice, "gwei"), "gwei")
  console.log("Total fee paid:", ethers.formatEther(receipt.gasUsed * receipt.gasPrice), "ETH")
}

sendTransaction()
```

**Setup .env file:**
```
ALCHEMY_SEPOLIA_URL=https://eth-sepolia.g.alchemy.com/v2/YOUR_API_KEY
PRIVATE_KEY=your_test_wallet_private_key_here
```

---

### Exercise 2: Monitor the Mempool

```javascript
// Watch for pending transactions involving a specific address
const provider = new ethers.WebSocketProvider(process.env.ALCHEMY_WS_URL)

const targetAddress = "0xUniswapV3Router" // example

provider.on("pending", async (txHash) => {
  try {
    const tx = await provider.getTransaction(txHash)
    if (tx && tx.to && tx.to.toLowerCase() === targetAddress.toLowerCase()) {
      console.log("Pending tx to Uniswap:", txHash)
      console.log("Gas price:", ethers.formatUnits(tx.gasPrice || tx.maxFeePerGas, "gwei"), "gwei")
      console.log("Value:", ethers.formatEther(tx.value), "ETH")
    }
  } catch (e) {
    // Transaction may have been dropped
  }
})

console.log("Watching mempool for Uniswap transactions...")
```

---

### Exercise 3: Decode Transaction Input Data

```javascript
const { Interface } = require('ethers')

// ERC-20 ABI (just the transfer function)
const erc20ABI = [
  "function transfer(address to, uint256 amount)"
]

const iface = new Interface(erc20ABI)

// A raw tx input (this is a USDC transfer)
const inputData = "0xa9059cbb000000000000000000000000recipient0000000000000000000000000000000000000000000000000000000000000de0b6b3a7640000"

const decoded = iface.parseTransaction({ data: inputData })
console.log("Function:", decoded.name)
console.log("To:", decoded.args[0])
console.log("Amount:", ethers.formatUnits(decoded.args[1], 6), "USDC") // USDC has 6 decimals
```

---

### Exercise 4: Calculate Real Transaction Cost

Write a function that given a contract function call, estimates gas + cost in USD:

```javascript
async function estimateTransactionCost(
  contractAddress,
  abi,
  functionName,
  args,
  ethPriceUSD = 3000
) {
  const provider = new ethers.JsonRpcProvider(process.env.ALCHEMY_SEPOLIA_URL)
  const contract = new ethers.Contract(contractAddress, abi, provider)
  
  const gasEstimate = await contract[functionName].estimateGas(...args)
  const feeData = await provider.getFeeData()
  
  const totalGasCostWei = gasEstimate * feeData.maxFeePerGas
  const totalGasCostETH = parseFloat(ethers.formatEther(totalGasCostWei))
  const totalGasCostUSD = totalGasCostETH * ethPriceUSD
  
  return {
    gasEstimate: gasEstimate.toString(),
    gasPriceGwei: ethers.formatUnits(feeData.maxFeePerGas, "gwei"),
    costETH: totalGasCostETH.toFixed(6),
    costUSD: totalGasCostUSD.toFixed(4)
  }
}
```

---

## Part 3: Deep Dive Exercises (1.5 hours)

### Challenge: Trace a DeFi Transaction

1. Go to https://phalcon.blocksec.com/
2. Find any Uniswap swap transaction
3. Trace the internal calls:
   - What contracts were called?
   - How many internal transactions?
   - What was the gas breakdown?

**Questions:**
- What is the difference between the "value" field and ERC-20 token transfers?
- Why does a Uniswap swap touch multiple contracts?

---

### Gas Optimization Challenge

Look at this contract and identify gas waste:
```solidity
// GAS INEFFICIENT - can you spot all 5 problems?
contract Inefficient {
    uint256[] public values;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function addValues(uint256[] memory newValues) external {
        for (uint256 i = 0; i < newValues.length; i++) {
            values.push(newValues[i]);  // Problem 1: storage in loop
        }
    }
    
    function sumValues() external view returns (uint256) {
        uint256 total = 0;
        for (uint256 i = 0; i < values.length; i++) {  // Problem 2
            total += values[i];  // Problem 3: SLOAD in loop
        }
        return total;
    }
    
    function getOwner() public view returns (address) {
        return owner;  // Problem 4: redundant, use public variable
    }
    
    function transferOwner(address newOwner) external {
        require(msg.sender == owner);  // Problem 5: should be custom error
        owner = newOwner;
    }
}
```

Write the optimized version.

---

## Part 4: Reflection (30 min)

1. If the base fee is 50 gwei, your maxFeePerGas is 60 gwei, and priority fee is 2 gwei — how much do you actually pay? What if baseFee suddenly jumps to 65 gwei?
2. Why can't you submit two transactions with the same nonce from the same address?
3. What happens to your gas if your transaction reverts?
4. Why would someone pay a HIGHER gas price than necessary?
5. What is the MEV opportunity in the mempool? (Preview for Week 4)

---

## Resources for Day 2

| Resource | Link | Type |
|----------|------|------|
| EIP-1559 Explained | https://metamask.io/1559/ | Article |
| Ethereum Gas Tracker | https://etherscan.io/gastracker | Tool |
| Ultrasound Money (ETH burn) | https://ultrasound.money | Dashboard |
| Tx Visualizer (Phalcon) | https://phalcon.blocksec.com | Tool |
| Tenderly (tx traces) | https://tenderly.co | Tool |
| Ethers.js v6 Docs | https://docs.ethers.org/v6/ | Docs |
| EVM Opcodes (gas costs) | https://www.evm.codes/ | Reference |
| Mempool.space | https://mempool.space | Tool |

---

## Tomorrow

[Day 3 → Consensus Mechanisms: PoW vs PoS](./Day-3-Consensus-Mechanisms.md)
