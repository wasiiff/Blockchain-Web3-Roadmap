# Day 28 (Week 4, Day 7) — Capstone: Final Deploy & Review

> **Time:** Full day  
> **Goal:** Deploy PixelSwap to Sepolia, verify all contracts, deploy frontend, write your portfolio README.

---

## Final Deployment Checklist

### Smart Contracts
- [ ] All tests pass (`npx hardhat test`)
- [ ] Coverage > 90% (`npx hardhat coverage`)
- [ ] Slither analysis run, high/medium issues addressed
- [ ] Factory deployed + verified
- [ ] Router deployed + verified
- [ ] WETH address used correctly
- [ ] Initial test pairs created
- [ ] Initial liquidity added
- [ ] At least one swap executed on-chain

### Frontend
- [ ] Connected to Sepolia
- [ ] Swap page functional
- [ ] Add liquidity functional
- [ ] Pool stats displayed (reserves, price, TVL)
- [ ] Deployed to Vercel/Netlify

### Documentation
- [ ] README with contract addresses
- [ ] Architecture diagram
- [ ] How to run locally
- [ ] Known limitations

---

## Portfolio README Template

```markdown
# PixelSwap — Decentralized Exchange

A Uniswap V2-inspired AMM built as a 4-week blockchain learning capstone.

## Live Demo
https://pixelswap.vercel.app

## Contracts (Sepolia Testnet)
| Contract | Address |
|---------|---------|
| Factory | 0x... |
| Router | 0x... |
| WETH | 0x... |
| Test TokenA | 0x... |
| Test TokenB | 0x... |

## Architecture
- **Factory**: Deploys pair contracts via CREATE2
- **Pair (×N)**: ERC-20 LP token + x·y=k AMM logic + TWAP oracle  
- **Router**: User-facing — add liquidity, swap, multi-hop routing

## Tech Stack
- Solidity 0.8.24
- Hardhat + Foundry
- OpenZeppelin contracts
- React + Next.js 14
- Wagmi v2 + Viem
- RainbowKit
- Deployed on Sepolia testnet

## Features
- ✅ Create token pairs
- ✅ Add/remove liquidity (get LP tokens)
- ✅ Swap tokens (0.3% fee)
- ✅ Multi-hop routing (A→B→C)
- ✅ Slippage protection + deadline
- ✅ ETH → WETH wrapping
- ✅ TWAP price oracle
- ✅ Fully verified on Etherscan

## Running Locally
\`\`\`bash
git clone https://github.com/you/pixelswap
cd pixelswap

# Contracts
cd contracts
npm install
npx hardhat test
npx hardhat run scripts/deploy.js --network sepolia

# Frontend
cd ../frontend
npm install
cp .env.example .env.local
# Edit .env.local with your contract addresses
npm run dev
\`\`\`
```

---

## Course Complete! What You've Learned

### Week 1 — Foundations
- ✅ How blockchains work cryptographically (hashing, Merkle trees, ECDSA)
- ✅ Transaction lifecycle (wallet → mempool → block → finality)
- ✅ PoW vs PoS consensus
- ✅ Solidity in-depth (storage, events, modifiers, inheritance)
- ✅ Hardhat development workflow
- ✅ ERC-20 standard (including Permit/EIP-2612)

### Week 2 — DeFi & AMMs
- ✅ x·y=k AMM formula (price impact, slippage)
- ✅ Impermanent loss calculation
- ✅ Uniswap V2/V3 architecture
- ✅ DeFi protocols (Aave, Curve, staking)
- ✅ Wagmi v2 + Viem + RainbowKit frontend

### Week 3 — NFTs & Marketplaces
- ✅ ERC-721 and ERC-1155 token standards
- ✅ IPFS and decentralized metadata storage
- ✅ NFT marketplace architecture (escrow model)
- ✅ OpenSea's Seaport off-chain order system
- ✅ ERC-2981 royalties
- ✅ Complete NFT frontend (mint, gallery, buy)

### Week 4 — Advanced & Capstone
- ✅ Layer 2 scaling (Optimistic and ZK-Rollups)
- ✅ Smart contract security (reentrancy, flash loans, front-running)
- ✅ MEV and the Flashbots ecosystem
- ✅ Chainlink price feeds and VRF
- ✅ Production DEX (PixelSwap capstone)

---

## What's Next?

### Immediate Next Steps
1. **Get your contracts audited** — submit to Code4rena (competitive audit) or Sherlock
2. **Deploy to mainnet** — after thorough testing
3. **Build in public** — share on Twitter/X, write blog posts about what you built
4. **Contribute to open source** — Uniswap, Aave, OpenZeppelin all accept PRs

### Job Search Path
- Build a portfolio of 3-5 deployed projects
- Share on GitHub with good README files
- Write about your projects on Mirror or Substack
- Attend EthDenver, EthCC, Devcon (great for networking)
- Apply to: Uniswap Labs, OpenSea, Aave, ConsenSys, Alchemy, Chainlink, Trail of Bits

### Deeper Learning
- **DeFi Security:** Secureum Bootcamp, Code4rena, Sherlock audits
- **ZK Proofs:** ZK Hack, 0xPARC ZK Learning Group
- **Protocol Research:** Read Uniswap V3, Curve, Aave governance forums
- **Account Abstraction:** EIP-4337, Stackup, Alchemy's AA SDK
- **Cross-chain:** LayerZero, Wormhole, Axelar
