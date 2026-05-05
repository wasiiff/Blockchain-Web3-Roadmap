# Web3 Blockchain Mastery — 4-Week Intensive Curriculum

> **Target:** Software engineer with React, Node.js, MongoDB background + basic Solidity  
> **Goal:** Industry-ready Web3 developer (DeFi + NFTs + Smart Contracts)  
> **Duration:** 4 weeks, ~6–8 hours/day

---

## Learning Objectives

### Technical Objectives
- Deploy production-grade smart contracts on Ethereum mainnet and testnets
- Build full DeFi applications (AMMs, lending protocols, yield aggregators)
- Create and deploy NFT collections with on-chain/off-chain metadata
- Build NFT marketplaces with listing, buying, royalty, and auction flows
- Integrate smart contracts with React frontends using Ethers.js, Wagmi, Viem
- Understand and protect against smart contract security vulnerabilities
- Work with Layer 2 networks (Arbitrum, Optimism, zkSync)
- Use professional tools: Hardhat, Foundry, OpenZeppelin, Chainlink

### Conceptual Objectives
- Deeply understand how blockchains work at the protocol level
- Understand the full transaction lifecycle from wallet to finality
- Master AMM mechanics (Uniswap v2/v3, x·y=k curve, impermanent loss)
- Understand DeFi primitives: DEXs, lending, staking, flash loans, MEV
- Know how NFT marketplaces work end-to-end (OpenSea model)

---

## Expected Outcomes After 4 Weeks

| Skill | Level |
|-------|-------|
| Smart Contract Development | Advanced |
| DeFi Protocol Understanding | Deep |
| NFT Systems | Production-ready |
| Frontend Web3 Integration | Professional |
| Security Awareness | Solid |
| Layer 2 Networks | Working knowledge |
| Professional Tools | Proficient |

---

## Required Tools & Setup

### Core Development Environment
```bash
# Node.js (v18+)
node --version

# Package managers
npm install -g pnpm

# Hardhat
npm install --save-dev hardhat

# Foundry (Rust-based, fastest testing)
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Forge + Cast + Anvil (comes with Foundry)
forge --version
cast --version
anvil --version
```

### Wallets & Browser Tools
- **MetaMask** — https://metamask.io (browser extension)
- **Rabby Wallet** — more advanced alternative
- **WalletConnect** — multi-wallet support

### Testnets & Faucets
| Network | RPC | Faucet |
|---------|-----|--------|
| Sepolia (Ethereum) | https://rpc.sepolia.org | https://sepoliafaucet.com |
| Mumbai (Polygon) | https://rpc-mumbai.maticvigil.com | https://faucet.polygon.technology |
| Arbitrum Goerli | https://goerli-rollup.arbitrum.io/rpc | https://faucet.arbitrum.io |
| Optimism Goerli | https://goerli.optimism.io | https://faucet.optimism.io |

### Key Libraries
```bash
# Smart contract dev
npm install --save-dev @openzeppelin/contracts
npm install --save-dev @chainlink/contracts

# Frontend
npm install ethers wagmi viem @wagmi/connectors
npm install @rainbow-me/rainbowkit  # beautiful wallet modal

# Testing
npm install --save-dev chai mocha
```

### APIs & Services
| Service | Purpose | Free Tier |
|---------|---------|-----------|
| **Alchemy** (https://alchemy.com) | RPC Node Provider | 300M compute units/mo |
| **Infura** (https://infura.io) | Alternative RPC | 100K req/day |
| **Etherscan** (https://etherscan.io) | Block explorer + contract verification | Free |
| **Pinata** (https://pinata.cloud) | IPFS pinning for NFT metadata | 1GB free |
| **OpenZeppelin Defender** | Contract monitoring + automation | Free tier |

### VS Code Extensions
- **Solidity** by Nomic Foundation (syntax highlighting, compilation)
- **Hardhat for Visual Studio Code**
- **GitLens**
- **Prettier** (with Solidity plugin)

---

## Curriculum Map

```
Week 1 — Blockchain Foundations
├── Day 1: How Blockchains Work (Deep)
├── Day 2: Transactions, Gas, Mempool
├── Day 3: Consensus (PoW vs PoS)
├── Day 4: Solidity In-Depth
├── Day 5: Hardhat + First Deployment
├── Day 6: ERC-20 Token Standard
└── Day 7: Mini Project — Launch Your Own Token

Week 2 — DeFi & AMMs
├── Day 1: ERC-721 & ERC-1155 Standards
├── Day 2: Liquidity Pools & AMM Theory
├── Day 3: Uniswap Deep Dive (x·y=k)
├── Day 4: DeFi Protocols (DEX, Lending, Staking)
├── Day 5: Frontend — Ethers.js & Wagmi
├── Day 6: Full DeFi Frontend Integration
└── Day 7: Mini Project — Simple AMM Contract

Week 3 — NFTs & Marketplaces
├── Day 1: NFT Architecture & Metadata
├── Day 2: IPFS & Decentralized Storage
├── Day 3: NFT Marketplace Architecture
├── Day 4: OpenSea Model Deep Dive
├── Day 5: Royalties & ERC-2981
├── Day 6: NFT Frontend (Mint + Gallery + Buy)
└── Day 7: Mini Project — NFT Marketplace

Week 4 — Advanced Topics & Capstone
├── Day 1: Layer 2 (Optimism, Arbitrum, ZK-Rollups)
├── Day 2: Smart Contract Security
├── Day 3: MEV & Flash Loans
├── Day 4: Oracles & Chainlink
├── Day 5: Capstone Architecture Design
├── Day 6: Capstone Implementation
└── Day 7: Capstone — Final Deploy & Review
```

---

## Navigation

| Folder | Contents |
|--------|----------|
| [Week 1 — Foundations](./Week-1-Foundations/) | Blockchain core, Solidity, ERC-20 |
| [Week 2 — DeFi & AMMs](./Week-2-DeFi-and-AMMs/) | Uniswap, liquidity, DeFi protocols |
| [Week 3 — NFTs & Marketplaces](./Week-3-NFTs-and-Marketplaces/) | NFTs, OpenSea model, IPFS |
| [Week 4 — Advanced & Capstone](./Week-4-Advanced-and-Capstone/) | L2, security, MEV, final project |
| [Resources](./Resources/) | Docs, repos, videos, whitepapers |
| [Exercises](./Exercises/) | Coding challenges, bug fixes, gas optimization |
| [Capstone](./Capstone/) | Final project architecture and specs |

---

## Daily Time Allocation (6–8 hours)

| Block | Duration | Activity |
|-------|----------|----------|
| Morning | 2 hrs | Deep theory + concept study |
| Midday | 2 hrs | Coding exercises + tutorials |
| Afternoon | 2 hrs | Mini project / capstone work |
| Evening | 1 hr | Review, debug, reflect, notes |

---

> Start with [Week 1 → Day 1](./Week-1-Foundations/Day-1-Blockchain-Fundamentals.md)
