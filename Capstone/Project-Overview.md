# PixelSwap — Capstone Project Overview

## Project Summary

PixelSwap is a Uniswap V2-inspired decentralized exchange implementing the x·y=k constant product AMM formula. It demonstrates mastery of all concepts from the 4-week curriculum.

## Components

| Component | Contract/File | Status |
|-----------|--------------|--------|
| Factory | `PixelSwapFactory.sol` | Build in Day 27 |
| Pair | `PixelSwapPair.sol` | Build in Day 27 |
| Router | `PixelSwapRouter.sol` | Build in Day 27 |
| WETH | `WETH9.sol` | Use standard |
| Frontend | `Next.js app` | Build in Day 27-28 |

## Deployed Addresses (Fill After Deploy)
```
Network: Sepolia
Factory:  0x...
Router:   0x...
WETH:     0x...
TokenA:   0x...
TokenB:   0x...
```

## Features

### Core
- [x] Token pair creation (Factory with CREATE2)
- [x] Add liquidity (receive LP tokens)
- [x] Remove liquidity (burn LP tokens, receive tokens)
- [x] Swap token A → token B (0.3% fee)
- [x] Swap ETH → token (WETH wrapping)
- [x] Multi-hop swaps (A → B → C)
- [x] Slippage protection (minAmountOut)
- [x] Transaction deadline
- [x] TWAP oracle

### Security
- [x] Reentrancy guards
- [x] CEI pattern throughout
- [x] Overflow-safe (Solidity 0.8.24)
- [x] Minimum liquidity lock (1000)
- [x] Token sorting (deterministic pair addresses)

### Frontend
- [x] Wallet connection (MetaMask, WalletConnect, Coinbase)
- [x] Token balance display
- [x] Swap interface with price impact
- [x] Liquidity management
- [x] Transaction status and history

## Bonus Improvements

1. **Price Charts:** Display price history from Swap events
2. **APY Calculator:** Estimate LP yield based on volume
3. **Multi-hop Routing:** Find best path through multiple pools
4. **Permit Integration:** Gasless approvals for supported tokens
5. **Deploy to Arbitrum/Base:** Lower gas costs for users
6. **The Graph Integration:** Fast event indexing for analytics

## Architecture

See [Architecture.md](./Architecture.md) for detailed diagrams.

## Testing

```bash
# Run all tests
npx hardhat test

# With gas report
REPORT_GAS=true npx hardhat test

# Coverage
npx hardhat coverage

# Foundry fuzz tests
forge test --fuzz-runs 1000
```

## Known Limitations

1. No fee-on-transfer token support
2. No rebasing token support (stETH)
3. No price oracle beyond TWAP
4. Simple routing (no optimal path finding)
5. No liquidity manager UI (Uniswap V3-style range selection)
