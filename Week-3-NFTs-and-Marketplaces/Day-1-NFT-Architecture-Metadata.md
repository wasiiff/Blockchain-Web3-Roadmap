# Day 15 (Week 3, Day 1) — NFT Architecture & Metadata

> **Time:** 6–8 hours  
> **Goal:** Understand every component of an NFT — from tokenId to metadata to image storage.

---

## Part 1: How NFTs Actually Work (2 hours)

### 1.1 The Big Picture

An NFT is surprisingly simple at the contract level:

```
NFT Contract (ERC-721)
    │
    ├── tokenId → ownerOf (who owns it)
    │   balances[address] → how many they hold
    │   
    └── tokenId → tokenURI (a URL/URI string)
                         │
                         ▼
                   Metadata JSON (off-chain)
                   {
                     "name": "CryptoArt #1",
                     "description": "...",
                     "image": "ipfs://...",
                     "attributes": [...]
                   }
                         │
                    image field ─────▶  Image File (IPFS/Arweave/HTTP)
```

The contract itself only stores:
1. Who owns each tokenId
2. A URI string pointing to metadata

**The actual image and metadata live off-chain.**

---

### 1.2 Metadata Standard (OpenSea Compatible)

OpenSea and all major marketplaces read this JSON format:

```json
{
  "name": "ArtBlock #42",
  "description": "A unique piece from the ArtBlock collection.",
  "image": "ipfs://QmXxxxx/42.png",
  "external_url": "https://artblock.io/token/42",
  "background_color": "1a1a1a",
  "animation_url": "ipfs://QmXxxxx/42.mp4",
  "youtube_url": "https://www.youtube.com/watch?v=...",
  "attributes": [
    {
      "trait_type": "Background",
      "value": "Deep Space"
    },
    {
      "trait_type": "Body",
      "value": "Holographic"
    },
    {
      "trait_type": "Eyes",
      "value": "Laser"
    },
    {
      "display_type": "number",
      "trait_type": "Generation",
      "value": 3
    },
    {
      "display_type": "boost_percentage",
      "trait_type": "Speed Boost",
      "value": 15
    },
    {
      "display_type": "date",
      "trait_type": "Created",
      "value": 1546360800
    }
  ]
}
```

**Attribute display types:**
- No `display_type`: shown as string/text
- `"number"`: shown as a number
- `"boost_number"`: shown as +N
- `"boost_percentage"`: shown as +N%
- `"date"`: shown as date (unix timestamp)

---

### 1.3 On-Chain vs Off-Chain Metadata

| Approach | Where Data Lives | Gas Cost | Immutability |
|---------|-----------------|----------|--------------|
| IPFS | IPFS network | Low | Permanent if pinned |
| Arweave | Arweave blockchain | Medium | Permanent (pay once) |
| HTTP URL | Central server | Zero | Can be changed/deleted |
| Base64 on-chain | On the blockchain | High | Permanent |
| IPFS with on-chain hash | IPFS + on-chain verification | Medium | Verified permanently |

**Best practice:** Use IPFS for images and metadata, store the IPFS CID (content hash) on-chain.

**Why IPFS is special:**
- Content-addressed: the URL is the hash of the content
- `ipfs://QmXxx...` — if you modify the file, you get a different CID
- The CID in the contract is proof that the metadata hasn't changed
- Multiple IPFS nodes can host the same content (decentralized)

**The rug-pull problem:** If an NFT uses `https://api.myproject.com/token/1`, the team can change what that URL returns, effectively "pulling the rug" on your metadata.

---

### 1.4 Types of tokenURI Implementations

**Type 1: Static IPFS Base URI (most common)**
```solidity
string private baseURI = "ipfs://QmCollectionHash/";

function tokenURI(uint256 tokenId) public view override returns (string memory) {
    require(_exists(tokenId), "Token does not exist");
    return string(abi.encodePacked(baseURI, tokenId.toString(), ".json"));
}
// tokenURI(42) = "ipfs://QmCollectionHash/42.json"
```

**Type 2: Per-token URI (ERC721URIStorage)**
```solidity
// Each token has its own URI stored on-chain
mapping(uint256 => string) private _tokenURIs;

function tokenURI(uint256 tokenId) public view override returns (string memory) {
    string memory _tokenURI = _tokenURIs[tokenId];
    string memory base = _baseURI();
    if (bytes(base).length == 0) return _tokenURI;
    if (bytes(_tokenURI).length > 0) return string(abi.encodePacked(base, _tokenURI));
    return super.tokenURI(tokenId);
}
// More flexible, but more storage per token (more gas)
```

**Type 3: Fully On-Chain SVG (expensive but immortal)**
```solidity
function tokenURI(uint256 tokenId) public view override returns (string memory) {
    string memory svg = generateSVG(tokenId);
    string memory json = Base64.encode(bytes(string(abi.encodePacked(
        '{"name": "Token #', tokenId.toString(), '",',
        '"description": "Fully on-chain NFT",',
        '"image": "data:image/svg+xml;base64,', Base64.encode(bytes(svg)), '"}'
    ))));
    return string(abi.encodePacked("data:application/json;base64,", json));
}

function generateSVG(uint256 tokenId) internal view returns (string memory) {
    // Generate SVG programmatically based on tokenId
    return string(abi.encodePacked(
        '<svg xmlns="http://www.w3.org/2000/svg" width="500" height="500">',
        '<rect fill="#', getColor(tokenId), '" width="500" height="500"/>',
        '<text x="250" y="250" text-anchor="middle" fill="white" font-size="40">',
        '#', tokenId.toString(),
        '</text></svg>'
    ));
}
```

---

### 1.5 Reveal Mechanics

Many NFT projects use a "reveal" — tokens show a placeholder until a reveal date:

**Pre-reveal:**
```solidity
bool public revealed = false;
string public notRevealedURI = "ipfs://QmPlaceholder/hidden.json";

function tokenURI(uint256 tokenId) public view override returns (string memory) {
    if (!revealed) return notRevealedURI;
    return string(abi.encodePacked(baseURI, tokenId.toString(), ".json"));
}
```

**Randomization problem:** If token IDs are sequential and metadata is at `baseURI/1.json`, `baseURI/2.json`, etc., people can mint only the rare tokens by peeking at IPFS before minting. Solutions:
1. Use a **provenance hash**: hash of all metadata committed before minting starts
2. Use **Chainlink VRF** to randomly assign metadata to minted tokenIds after reveal
3. Deploy metadata with shuffled IDs

---

## Part 2: Build a Feature-Rich NFT Collection (3 hours)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Royalty.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

contract ArtBlockNFT is ERC721, ERC721Royalty, Ownable, Pausable {
    using Strings for uint256;
    
    // ─── State ────────────────────────────────────────────────────────
    uint256 private _tokenIdCounter;
    uint256 public constant MAX_SUPPLY = 10_000;
    uint256 public constant MAX_PER_WALLET = 5;
    uint256 public mintPrice = 0.05 ether;
    
    string private _baseTokenURI;
    string public provenance; // SHA256 of all concatenated images (committed before reveal)
    
    bool public revealed;
    string public notRevealedURI;
    
    bool public publicSaleActive;
    bool public whitelistSaleActive;
    
    mapping(address => bool) public whitelist;
    mapping(address => uint256) public mintedCount;
    
    // ─── Errors ───────────────────────────────────────────────────────
    error SaleNotActive();
    error NotWhitelisted();
    error MaxSupplyReached();
    error MaxPerWalletReached();
    error InsufficientPayment();
    error WithdrawFailed();
    
    // ─── Events ──────────────────────────────────────────────────────
    event Revealed(string baseURI);
    event Minted(address indexed to, uint256 indexed tokenId);
    
    constructor(
        address initialOwner,
        string memory notRevealedURI_,
        uint96 royaltyBps  // e.g., 500 = 5%
    ) 
        ERC721("ArtBlock Collection", "ARTB")
        Ownable(initialOwner)
    {
        notRevealedURI = notRevealedURI_;
        _setDefaultRoyalty(initialOwner, royaltyBps);
    }
    
    // ─── Minting ──────────────────────────────────────────────────────
    
    function publicMint(uint256 quantity) external payable whenNotPaused {
        if (!publicSaleActive) revert SaleNotActive();
        _mintTokens(msg.sender, quantity);
    }
    
    function whitelistMint(uint256 quantity) external payable whenNotPaused {
        if (!whitelistSaleActive) revert SaleNotActive();
        if (!whitelist[msg.sender]) revert NotWhitelisted();
        _mintTokens(msg.sender, quantity);
    }
    
    function ownerMint(address to, uint256 quantity) external onlyOwner {
        if (_tokenIdCounter + quantity > MAX_SUPPLY) revert MaxSupplyReached();
        for (uint256 i = 0; i < quantity;) {
            _safeMint(to, _tokenIdCounter++);
            unchecked { ++i; }
        }
    }
    
    function _mintTokens(address to, uint256 quantity) internal {
        if (_tokenIdCounter + quantity > MAX_SUPPLY) revert MaxSupplyReached();
        if (mintedCount[to] + quantity > MAX_PER_WALLET) revert MaxPerWalletReached();
        if (msg.value < mintPrice * quantity) revert InsufficientPayment();
        
        mintedCount[to] += quantity;
        
        for (uint256 i = 0; i < quantity;) {
            uint256 tokenId = _tokenIdCounter++;
            _safeMint(to, tokenId);
            emit Minted(to, tokenId);
            unchecked { ++i; }
        }
    }
    
    // ─── Metadata ─────────────────────────────────────────────────────
    
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);
        if (!revealed) return notRevealedURI;
        return string(abi.encodePacked(_baseTokenURI, tokenId.toString(), ".json"));
    }
    
    function reveal(string calldata baseURI) external onlyOwner {
        _baseTokenURI = baseURI;
        revealed = true;
        emit Revealed(baseURI);
    }
    
    function setProvenance(string calldata provenanceHash) external onlyOwner {
        provenance = provenanceHash;
    }
    
    // ─── Admin ────────────────────────────────────────────────────────
    
    function setWhitelist(address[] calldata addresses, bool status) external onlyOwner {
        for (uint256 i = 0; i < addresses.length;) {
            whitelist[addresses[i]] = status;
            unchecked { ++i; }
        }
    }
    
    function togglePublicSale() external onlyOwner { publicSaleActive = !publicSaleActive; }
    function toggleWhitelistSale() external onlyOwner { whitelistSaleActive = !whitelistSaleActive; }
    function setMintPrice(uint256 newPrice) external onlyOwner { mintPrice = newPrice; }
    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
    
    function setRoyalty(address receiver, uint96 feeNumerator) external onlyOwner {
        _setDefaultRoyalty(receiver, feeNumerator);
    }
    
    function withdraw() external onlyOwner {
        (bool success,) = owner().call{value: address(this).balance}("");
        if (!success) revert WithdrawFailed();
    }
    
    function totalSupply() external view returns (uint256) {
        return _tokenIdCounter;
    }
    
    // ─── Overrides ────────────────────────────────────────────────────
    
    function supportsInterface(bytes4 interfaceId)
        public view override(ERC721, ERC721Royalty) returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

---

## Part 3: Exercises (2 hours)

### Exercise 1: Decode Metadata from Any NFT

```javascript
const { ethers } = require('ethers')

async function getNFTMetadata(contractAddress, tokenId) {
  const provider = new ethers.JsonRpcProvider(process.env.ALCHEMY_MAINNET_URL)
  
  const abi = ["function tokenURI(uint256) view returns (string)"]
  const nft = new ethers.Contract(contractAddress, abi, provider)
  
  const uri = await nft.tokenURI(tokenId)
  console.log("Raw URI:", uri)
  
  let metadata
  if (uri.startsWith("data:application/json;base64,")) {
    // On-chain base64 encoded JSON
    const json = Buffer.from(uri.split(",")[1], "base64").toString()
    metadata = JSON.parse(json)
  } else if (uri.startsWith("ipfs://")) {
    const url = uri.replace("ipfs://", "https://ipfs.io/ipfs/")
    metadata = await fetch(url).then(r => r.json())
  } else {
    metadata = await fetch(uri).then(r => r.json())
  }
  
  console.log("Name:", metadata.name)
  console.log("Description:", metadata.description)
  console.log("Image:", metadata.image)
  console.log("Attributes:", JSON.stringify(metadata.attributes, null, 2))
  
  return metadata
}

// Try with different collections:
// BAYC:    0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D
// Azuki:   0xED5AF388653567Af2F388E6224dC7C4b3241C544
// CryptoPunks: Not ERC-721! But has tokenURI-like function
// Loot:    0xFF9C1b15B16263C61d017ee9F65C50e4AE0113D7 (fully on-chain)

getNFTMetadata("0xFF9C1b15B16263C61d017ee9F65C50e4AE0113D7", 1) // Loot (on-chain)
```

### Exercise 2: Generate On-Chain SVG NFT

Create a smart contract that generates unique SVGs using on-chain randomness:

```solidity
function generateSVG(uint256 tokenId) internal view returns (string memory) {
    uint256 seed = uint256(keccak256(abi.encodePacked(tokenId, block.prevrandao)))
    
    string[6] memory colors = ["ff6b6b", "4ecdc4", "45b7d1", "96ceb4", "ffeaa7", "dda0dd"]
    string memory bgColor = colors[seed % 6]
    string memory fgColor = colors[(seed >> 8) % 6]
    uint256 shapeType = (seed >> 16) % 3 // 0=circle, 1=rect, 2=triangle
    
    // Build SVG based on seed
    // ...
}
```

---

## Resources for Day 15

| Resource | Link | Type |
|----------|------|------|
| OpenSea Metadata Standards | https://docs.opensea.io/docs/metadata-standards | Docs |
| EIP-721 | https://eips.ethereum.org/EIPS/eip-721 | Spec |
| OpenZeppelin ERC721 | https://docs.openzeppelin.com/contracts/5.x/erc721 | Docs |
| IPFS Docs | https://docs.ipfs.tech/ | Docs |
| Arweave Docs | https://www.arweave.org/ | Docs |
| Loot Project (on-chain) | https://etherscan.io/address/0xFF9C1b15B16263C61d017ee9F65C50e4AE0113D7 | Example |

---

## Tomorrow

[Day 16 → IPFS & Decentralized Storage](./Day-2-IPFS-Decentralized-Storage.md)
