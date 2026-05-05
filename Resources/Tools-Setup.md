# Tools & Setup Guide

## Development Tools

### Core
| Tool | Install | Use |
|------|---------|-----|
| Node.js 18+ | https://nodejs.org | Runtime |
| Hardhat | `npm i -D hardhat` | Dev framework |
| Foundry | `curl -L https://foundry.paradigm.xyz \| bash` | Fast testing |
| MetaMask | https://metamask.io | Wallet |

### VSCode Extensions
- **Solidity** by Nomic Foundation — syntax highlighting, compilation
- **Hardhat for VS Code** — debugger integration
- **Prettier** + solidity plugin — code formatting

### Solidity Formatting (.prettierrc)
```json
{
  "plugins": ["prettier-plugin-solidity"],
  "overrides": [
    {
      "files": "*.sol",
      "options": {
        "printWidth": 100,
        "tabWidth": 4,
        "useTabs": false,
        "singleQuote": false,
        "bracketSpacing": false
      }
    }
  ]
}
```

## RPC Providers

| Provider | Free Tier | Sign Up |
|---------|-----------|---------|
| Alchemy | 300M compute units/mo | https://alchemy.com |
| Infura | 100K req/day | https://infura.io |
| QuickNode | 500M credits/mo | https://quicknode.com |
| Ankr | 500M req/mo | https://ankr.com |

## Testnet Faucets

| Network | Faucet |
|---------|--------|
| Sepolia ETH | https://sepoliafaucet.com |
| Sepolia ETH | https://faucets.chain.link (Chainlink) |
| Arbitrum Sepolia | https://faucet.arbitrum.io |
| Optimism Sepolia | https://faucet.optimism.io |
| Polygon Mumbai | https://faucet.polygon.technology |
| LINK Token | https://faucets.chain.link |

## Contract Explorers

| Network | Explorer |
|---------|---------|
| Ethereum Mainnet | https://etherscan.io |
| Ethereum Sepolia | https://sepolia.etherscan.io |
| Arbitrum | https://arbiscan.io |
| Optimism | https://optimistic.etherscan.io |
| Base | https://basescan.org |
| Polygon | https://polygonscan.com |

## IPFS & Storage

| Service | Free Tier | Use |
|---------|-----------|-----|
| Pinata | 1GB, 100 files | NFT metadata pinning |
| NFT.Storage | Unlimited (Filecoin) | NFT metadata |
| Arweave | Pay per byte | Permanent storage |

## API Keys Needed

1. **Alchemy** — RPC endpoint (1 per network)
2. **Etherscan** — contract verification
3. **Pinata** — IPFS pinning
4. **WalletConnect** — multi-wallet support in frontend
5. **CoinMarketCap** — USD gas cost reporting (optional)
