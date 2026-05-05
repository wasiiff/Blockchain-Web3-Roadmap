# Day 22 (Week 4, Day 1) — Layer 2 & Rollups

> **Time:** 6–8 hours  
> **Goal:** Understand the Layer 2 ecosystem — how Optimistic Rollups and ZK-Rollups work. Deploy contracts on Arbitrum and Optimism.

---

## Part 1: Why Layer 2? (1 hour)

### The Scalability Trilemma

Ethereum must balance:
- **Decentralization:** Many validators, permissionless participation
- **Security:** Hard to attack the chain
- **Scalability:** High throughput, low fees

Choosing two means compromising the third. Ethereum chose decentralization + security.

**Ethereum L1 limitations:**
- ~15-30 TPS (transactions per second)
- $1-100+ gas fees during congestion
- 12-second block times

**Layer 2 solutions** move computation off Ethereum while inheriting its security.

---

## Part 2: Optimistic Rollups (1.5 hours)

### How Optimistic Rollups Work

```
Optimistic = "we assume transactions are valid"

L2 Sequencer (e.g., Arbitrum/Optimism):
1. Collects many transactions (100s-1000s per batch)
2. Executes them off-chain
3. Posts compressed batch data to Ethereum L1
4. Posts a "state root" — hash of all account balances after execution

Fraud Proof Window (7 days):
Anyone can challenge a batch by submitting a fraud proof
If fraud is proven → batch is rolled back, sequencer loses stake
If no challenge → batch is finalized after 7 days

Security inheritance:
→ Invalid state cannot persist because challengers watch the chain
→ Ethereum stores the data, so anyone can reconstruct L2 state
```

**Withdrawal delay:** Because of the 7-day challenge window, bridging from L2→L1 takes 7 days natively. "Fast bridges" (LayerZero, Hop, Across) bypass this using liquidity pools.

### Arbitrum vs Optimism

| Feature | Arbitrum One | Optimism |
|---------|-------------|---------|
| Fraud proofs | Multi-round, interactive | Single-round |
| EVM compatibility | Full (AVM) | Full (EVM equiv.) |
| Gas fees | ~10x cheaper than L1 | ~10x cheaper than L1 |
| Native token | ARB | OP |
| Ecosystem | Largest L2 TVL | OP Stack (Superchain) |
| Challenge period | 7 days | 7 days |

---

## Part 3: ZK-Rollups (1.5 hours)

### How ZK-Rollups Work

```
ZK = Zero Knowledge (validity proofs)

Instead of "assume valid until challenged":
"Prove mathematically that the state transition is valid"

L2 Prover:
1. Executes transactions off-chain
2. Generates a ZK proof (SNARK or STARK)
   "I executed these transactions and the new state is X"
3. Proof is verified on-chain (fast, small, cheap)
4. If proof is valid → state is finalized IMMEDIATELY

No fraud window needed! Withdrawals are fast.
```

**Types of ZK proofs:**
| Type | Examples | Proof size | Proof time |
|------|---------|-----------|-----------|
| zkSNARK | Groth16, PLONK | Small (~200 bytes) | Hours |
| zkSTARK | StarkWare | Large (~100KB) | Faster |

### ZK-EVM — The Hard Problem

EVM opcodes were not designed to be "ZK-friendly." Creating a ZK proof of EVM execution is extremely complex.

**ZK-EVM categories:**
| Type | EVM Compatibility | Speed | Example |
|------|-----------------|-------|---------|
| Type 1 | Perfect (Ethereum-equivalent) | Slow | Taiko |
| Type 2 | Fully EVM equivalent | Slow-medium | Scroll |
| Type 3 | Almost EVM equivalent | Medium | Polygon zkEVM |
| Type 4 | High-level language only | Fast | zkSync Era |

**Future:** ZK-Rollups are considered the end-game for L2 scaling, as they offer instant finality and stronger security than optimistic rollups.

---

## Part 4: Major L2 Networks (30 min)

| Network | Type | TVL | Key Feature |
|---------|------|-----|------------|
| Arbitrum One | Optimistic | $15B+ | Largest L2 |
| Optimism | Optimistic | $7B+ | OP Stack/Superchain |
| Base | Optimistic (OP Stack) | $8B+ | Coinbase's L2 |
| zkSync Era | ZK (Type 4) | $1B+ | Fast, cheap |
| Starknet | ZK (STARK) | $1B+ | Cairo language |
| Polygon zkEVM | ZK (Type 3) | $200M+ | EVM compatible |
| Linea | ZK (Type 2) | $500M+ | ConsenSys |
| Scroll | ZK (Type 2) | $500M+ | Bytecode equiv |

Check live stats: https://l2beat.com

---

## Part 5: Deploy to L2 (2 hours)

### 5.1 Add Networks to Hardhat

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    // Arbitrum Sepolia (testnet)
    arbitrumSepolia: {
      url: process.env.ARBITRUM_SEPOLIA_URL || "https://sepolia-rollup.arbitrum.io/rpc",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 421614,
    },
    
    // Optimism Sepolia (testnet)
    optimismSepolia: {
      url: process.env.OPTIMISM_SEPOLIA_URL || "https://sepolia.optimism.io",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 11155420,
    },
    
    // Base Sepolia (testnet)
    baseSepolia: {
      url: process.env.BASE_SEPOLIA_URL || "https://sepolia.base.org",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 84532,
    },
  },
  
  etherscan: {
    apiKey: {
      arbitrumSepolia: process.env.ARBISCAN_API_KEY,
      optimismSepolia: process.env.OPTIMISM_ETHERSCAN_KEY,
      baseSepolia: process.env.BASESCAN_API_KEY,
    },
    customChains: [
      {
        network: "arbitrumSepolia",
        chainId: 421614,
        urls: {
          apiURL: "https://api-sepolia.arbiscan.io/api",
          browserURL: "https://sepolia.arbiscan.io",
        },
      },
      {
        network: "optimismSepolia",
        chainId: 11155420,
        urls: {
          apiURL: "https://api-sepolia-optimistic.etherscan.io/api",
          browserURL: "https://sepolia-optimism.etherscan.io",
        },
      },
    ],
  },
}
```

### 5.2 Deploy Same Contract to Multiple Chains

```bash
# Get testnet funds first
# Arbitrum Sepolia faucet: https://faucet.arbitrum.io
# Optimism Sepolia faucet: https://faucet.optimism.io
# Base Sepolia faucet: https://faucet.base.org

# Deploy to Arbitrum Sepolia
npx hardhat run scripts/deploy.js --network arbitrumSepolia

# Deploy to Optimism Sepolia
npx hardhat run scripts/deploy.js --network optimismSepolia

# Deploy to Base Sepolia
npx hardhat run scripts/deploy.js --network baseSepolia
```

### 5.3 Gas Comparison Script

```javascript
// scripts/compareGas.js — Deploy same contract to multiple L2s and compare cost

async function measureGasCost(networkName) {
  const [deployer] = await ethers.getSigners()
  const provider = deployer.provider
  
  const feeData = await provider.getFeeData()
  const gasPrice = feeData.gasPrice
  
  const DevToken = await ethers.getContractFactory("DevToken")
  const tx = DevToken.getDeployTransaction("Test", "TST", 0n, deployer.address)
  const gasEstimate = await provider.estimateGas(tx)
  
  const totalCostWei = gasEstimate * gasPrice
  
  console.log(`\n${networkName}:`)
  console.log(`  Gas estimate: ${gasEstimate.toString()}`)
  console.log(`  Gas price: ${ethers.formatUnits(gasPrice, "gwei")} gwei`)
  console.log(`  Total cost: ${ethers.formatEther(totalCostWei)} ETH`)
}
```

### 5.4 Bridge ETH from L1 to L2

```javascript
// Native bridge to Optimism
const L1_BRIDGE = "0xFBb0621E0B23b5478B630BD55a5f21f67730B0F1" // OP L1 Bridge (Sepolia)

const L1_BRIDGE_ABI = [
  "function depositETH(uint32 _minGasLimit, bytes calldata _extraData) external payable"
]

async function bridgeToOptimism(amount) {
  const provider = new ethers.JsonRpcProvider(process.env.SEPOLIA_RPC)
  const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider)
  
  const bridge = new ethers.Contract(L1_BRIDGE, L1_BRIDGE_ABI, signer)
  
  const tx = await bridge.depositETH(
    200000, // min gas on L2
    "0x",   // extra data
    { value: ethers.parseEther(amount.toString()) }
  )
  
  console.log("Bridge tx:", tx.hash)
  console.log("Expected arrival on Optimism: ~2-3 minutes")
  
  await tx.wait()
}
```

---

## Part 6: Exercises (1 hour)

### Exercise 1: L2 Analysis

Go to https://l2beat.com and answer:

1. What is the current TVL of Arbitrum One?
2. What is the "state validation" method for zkSync?
3. Which L2 has the highest number of active addresses?
4. How does Base compare to Optimism in terms of security?

### Exercise 2: Deploy and Compare

Deploy your DevToken to:
1. Sepolia (Ethereum L1 testnet)
2. Arbitrum Sepolia
3. Optimism Sepolia

Record gas costs and compare. Which is cheapest? Why?

---

## Resources for Day 22

| Resource | Link | Type |
|----------|------|------|
| L2Beat | https://l2beat.com | Dashboard |
| Arbitrum Docs | https://docs.arbitrum.io/ | Docs |
| Optimism Docs | https://docs.optimism.io/ | Docs |
| Base Docs | https://docs.base.org/ | Docs |
| zkSync Docs | https://docs.zksync.io/ | Docs |
| Rollup Economics | https://drive.google.com/file/d/1EgaEDMBqJqFBSAJhI4-1gMQHgQqB5JQ2/view | Paper |
| Hop Bridge | https://hop.exchange/ | Bridge |
| Across Bridge | https://across.to/ | Bridge |

---

## Tomorrow

[Day 23 → Smart Contract Security](./Day-2-Smart-Contract-Security.md)
