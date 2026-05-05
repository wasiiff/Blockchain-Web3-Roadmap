# Week 3 Exercises — NFTs & Marketplaces

## Exercise Set 8: NFT Contract Challenges

### E8.1 — Trait Rarity Analysis
Given a collection of 100 NFTs, calculate rarity scores:
```javascript
// Metadata array with attributes
const collection = [
  { tokenId: 0, attributes: [{ trait_type: "Background", value: "Blue" }, ...] },
  // ... 99 more
]

function calculateRarity(collection) {
  // For each trait combination, count how many NFTs have it
  // Rarity score = sum of (1 / traitFrequency) for each trait
  // Higher score = rarer
}
```

### E8.2 — Metadata Validation
Write a script that validates all your NFT metadata:
- Each tokenId has a corresponding metadata file
- `name` field present and not empty
- `image` field is a valid IPFS URI
- `attributes` array has at least 3 traits
- No duplicate tokenIds

### E8.3 — On-Chain SVG NFT
Create an NFT where the entire image is generated on-chain:
- Color changes based on tokenId
- Text shows the token number
- Different shapes for different rarity tiers

```solidity
function generateSVG(uint256 tokenId) internal pure returns (string memory) {
    string[6] memory colors = ["ff6b6b", "4ecdc4", "45b7d1", "96ceb4", "ffeaa7", "dda0dd"];
    string memory color = colors[tokenId % 6];
    
    return string(abi.encodePacked(
        '<svg xmlns="http://www.w3.org/2000/svg" width="400" height="400">',
        '<rect width="400" height="400" fill="#', color, '"/>',
        '<text x="200" y="200" text-anchor="middle" fill="white" font-size="48" font-family="Arial">',
        '#', Strings.toString(tokenId),
        '</text></svg>'
    ));
}
```

---

## Exercise Set 9: Marketplace Challenges

### E9.1 — Batch Listing
Add a function to NFTMarketplace that allows listing multiple NFTs in one transaction:
```solidity
function batchList(
    address nftContract,
    uint256[] calldata tokenIds,
    uint256[] calldata prices
) external returns (uint256[] memory listingIds) {
    // implement
}
```

### E9.2 — Offer System
Add an offer/bid system to the marketplace:
- Buyers can make offers on any NFT (even if not listed)
- Sellers can accept any pending offer
- Offers expire after N days
- Offers must be in ETH (msg.value)
- Rejected/expired offers can be withdrawn

### E9.3 — Price History
Write a script that queries all `Sold` events and builds a price history for a specific NFT:
```javascript
async function getPriceHistory(nftContract, tokenId) {
  const marketplace = /* ... */
  const filter = marketplace.filters.Sold(null, null, nftContract, tokenId)
  const events = await marketplace.queryFilter(filter)
  
  return events.map(e => ({
    price: ethers.formatEther(e.args.price),
    buyer: e.args.buyer,
    date: /* get block timestamp */
  }))
}
```

---

## Exercise Set 10: IPFS & Storage

### E10.1 — Complete Upload Pipeline
Write a Node.js script that:
1. Takes a directory of images
2. Generates metadata JSON for each (with deterministic traits based on tokenId)
3. Uploads all images to IPFS via Pinata
4. Updates metadata with IPFS image URLs
5. Uploads all metadata to IPFS
6. Returns the base URI ready for the NFT contract

### E10.2 — Metadata Retrieval & Display
Build a component that:
- Takes any ERC-721 contract address + tokenId
- Fetches and displays metadata
- Handles IPFS, HTTP, and base64 encoded URIs
- Shows loading + error states
- Displays all attributes as chips/badges

---

## Debugging Challenges

### Bug Hunt: Marketplace Reentrancy
```solidity
// This marketplace has a reentrancy vulnerability. Find and fix it.
contract VulnerableMarketplace {
    struct Listing {
        address seller;
        uint256 price;
        bool active;
    }
    
    mapping(uint256 => Listing) public listings;
    
    function buy(uint256 listingId) external payable {
        Listing storage listing = listings[listingId];
        require(listing.active, "Not active");
        require(msg.value >= listing.price);
        
        // Send ETH to seller BEFORE marking inactive
        (bool success,) = listing.seller.call{value: listing.price}("");  // BUG: reentrancy!
        require(success);
        
        // Transfer NFT
        IERC721(nftContract).transferFrom(listing.seller, msg.sender, listingId);
        
        listing.active = false;  // Too late!
    }
}
```

### Gas Optimization: Metadata Storage
```solidity
// Storing per-token URIs on-chain is expensive.
// Optimize this for a 10,000 item collection.
contract ExpensiveNFT {
    mapping(uint256 => string) public tokenURIs;  // 10,000 strings stored on-chain!
    
    function setTokenURI(uint256 tokenId, string calldata uri) external {
        tokenURIs[tokenId] = uri;
    }
    
    // How can you store all 10,000 URIs with 1 SSTORE instead of 10,000?
    // Hint: Use a base URI + tokenId approach
}
```
