# Day 21 (Week 3, Day 7) — Mini Project: Full NFT Marketplace

> **Time:** Full day  
> **Goal:** Deploy a complete NFT marketplace — ERC-721 collection + marketplace contract + React frontend.

---

## Project: ArtBlock Marketplace

**Contracts:**
- `ArtBlockNFT.sol` — ERC-721 with royalties
- `NFTMarketplace.sol` — Escrow marketplace

**Frontend:**
- Mint page
- Gallery (all NFTs)
- Token detail page
- My NFTs page (list for sale)
- Active listings page

---

## Final Checklist

### Contracts
- [ ] ArtBlockNFT deployed + verified on Sepolia
- [ ] NFTMarketplace deployed + verified on Sepolia
- [ ] 5% ERC-2981 royalty configured
- [ ] Test NFTs minted (at least 5)
- [ ] Test listing created (at least 1)

### Frontend
- [ ] Connects to MetaMask
- [ ] Mint page works (handles sale active/inactive)
- [ ] Gallery shows all minted NFTs with metadata
- [ ] Token page shows owner, traits, listing info
- [ ] List for sale works (approve + list)
- [ ] Buy works (sends ETH, transfers NFT)
- [ ] Cancel listing works

### Advanced (Bonus)
- [ ] Upload your own image to IPFS via Pinata
- [ ] Handle different image URIs (IPFS, HTTP, base64)
- [ ] Show royalty info on token page
- [ ] Transaction history per token

---

## Deployment Script

```javascript
// scripts/deployAll.js
async function main() {
  const [deployer] = await ethers.getSigners()
  
  // Deploy NFT
  const ArtBlockNFT = await ethers.getContractFactory("ArtBlockNFT")
  const nft = await ArtBlockNFT.deploy(
    deployer.address,                          // owner
    "ipfs://QmPlaceholder/hidden.json",        // pre-reveal URI
    500                                        // 5% royalty
  )
  await nft.waitForDeployment()
  
  // Deploy Marketplace
  const NFTMarketplace = await ethers.getContractFactory("NFTMarketplace")
  const marketplace = await NFTMarketplace.deploy(
    deployer.address,  // owner
    deployer.address   // fee recipient (use multisig in production)
  )
  await marketplace.waitForDeployment()
  
  // Enable public sale
  await nft.togglePublicSale()
  
  console.log("NFT:", await nft.getAddress())
  console.log("Marketplace:", await marketplace.getAddress())
  
  // Save to config file for frontend
  const fs = require("fs")
  fs.writeFileSync("../frontend/src/config/addresses.json", JSON.stringify({
    nft: await nft.getAddress(),
    marketplace: await marketplace.getAddress(),
    network: "sepolia",
    chainId: 11155111,
  }, null, 2))
}
```

---

## Week 3 Complete!

**What you built this week:**
- Complete ERC-721 NFT collection with reveal, whitelist, and royalties
- IPFS metadata pipeline with Pinata
- Full NFT marketplace with escrow, listing, buying, cancellation
- Deep understanding of OpenSea's Seaport architecture
- ERC-2981 royalty standard implementation
- Complete React frontend: mint, gallery, buy

**Next:** [Week 4 → Advanced Topics & Capstone](../Week-4-Advanced-and-Capstone/README.md)
