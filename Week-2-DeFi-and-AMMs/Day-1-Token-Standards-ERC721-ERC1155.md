# Day 8 (Week 2, Day 1) — ERC-721 & ERC-1155 Token Standards

> **Time:** 6–8 hours  
> **Goal:** Understand non-fungible and semi-fungible token standards — the foundation of NFTs and gaming tokens.

---

## Part 1: ERC-721 (Non-Fungible Tokens) (2 hours)

### 1.1 What Makes a Token Non-Fungible?

**Fungible:** 1 ETH = 1 ETH = 1 ETH. Interchangeable units.  
**Non-fungible:** Token #1 ≠ Token #2 — each is unique.

Real-world analogies:
- Baseball cards — each card (player, condition, year) is unique
- Real estate — each property at a unique location
- Artwork — each painting is one of a kind

In Ethereum: each ERC-721 token has a unique `tokenId` (uint256) and belongs to exactly one address.

---

### 1.2 Full ERC-721 Interface

```solidity
interface IERC721 {
    // ─── Ownership ────────────────────────────────────────────────────
    
    /// @notice Who owns tokenId?
    function ownerOf(uint256 tokenId) external view returns (address);
    
    /// @notice How many tokens does `owner` hold?
    function balanceOf(address owner) external view returns (uint256);
    
    // ─── Transfers ────────────────────────────────────────────────────
    
    /// @notice Transfer tokenId from → to (must be approved or owner)
    function transferFrom(address from, address to, uint256 tokenId) external;
    
    /// @notice Safe transfer: checks if receiver can handle NFTs
    /// (if receiver is a contract, it must implement onERC721Received)
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
    
    // ─── Approvals ────────────────────────────────────────────────────
    
    /// @notice Approve one address to transfer one specific token
    function approve(address to, uint256 tokenId) external;
    
    /// @notice Approve/revoke operator for ALL your tokens
    /// (Marketplaces use this — approve OpenSea once, list unlimited tokens)
    function setApprovalForAll(address operator, bool approved) external;
    
    /// @notice Who is approved for this specific token?
    function getApproved(uint256 tokenId) external view returns (address);
    
    /// @notice Is `operator` approved for all of `owner`'s tokens?
    function isApprovedForAll(address owner, address operator) external view returns (bool);
    
    // ─── Events ──────────────────────────────────────────────────────
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
}

// Metadata extension (optional but universal)
interface IERC721Metadata {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    /// @notice Returns URI pointing to token metadata JSON
    function tokenURI(uint256 tokenId) external view returns (string memory);
}
```

---

### 1.3 Why safeTransferFrom vs transferFrom?

```
Scenario: You send an NFT to a smart contract address that doesn't know how to handle NFTs.
The NFT is stuck forever — the contract can't send it back.

safeTransferFrom prevents this:
- If receiver is EOA: always OK
- If receiver is a contract: calls onERC721Received() on the contract
  - If it returns the magic bytes (0x150b7a02): transfer proceeds
  - If it doesn't implement it, or returns wrong value: transfer reverts
```

**Implementing a contract that receives NFTs:**
```solidity
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";

contract NFTVault is IERC721Receiver {
    mapping(uint256 => address) public nftDepositors;
    
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external override returns (bytes4) {
        nftDepositors[tokenId] = from;
        return IERC721Receiver.onERC721Received.selector; // magic bytes
    }
}
```

---

### 1.4 Build a Basic ERC-721 Collection

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract BasicNFT is ERC721URIStorage, Ownable {
    
    uint256 private _tokenIdCounter;
    uint256 public constant MAX_SUPPLY = 10_000;
    uint256 public mintPrice = 0.01 ether;
    
    bool public publicSaleActive;
    
    mapping(address => uint256) public mintCount; // prevent over-minting
    uint256 public constant MAX_PER_WALLET = 5;
    
    error SaleNotActive();
    error MaxSupplyReached();
    error MaxPerWalletReached();
    error InsufficientPayment(uint256 sent, uint256 required);
    
    constructor(address initialOwner) 
        ERC721("BasicNFT Collection", "BNFT")
        Ownable(initialOwner)
    {}
    
    function mint(string calldata tokenURI) external payable {
        if (!publicSaleActive) revert SaleNotActive();
        if (_tokenIdCounter >= MAX_SUPPLY) revert MaxSupplyReached();
        if (mintCount[msg.sender] >= MAX_PER_WALLET) revert MaxPerWalletReached();
        if (msg.value < mintPrice) revert InsufficientPayment(msg.value, mintPrice);
        
        uint256 tokenId = _tokenIdCounter++;
        mintCount[msg.sender]++;
        
        _safeMint(msg.sender, tokenId);
        _setTokenURI(tokenId, tokenURI);
    }
    
    /// @notice Owner can mint for free (team, giveaways)
    function ownerMint(address to, string calldata tokenURI) external onlyOwner {
        uint256 tokenId = _tokenIdCounter++;
        _safeMint(to, tokenId);
        _setTokenURI(tokenId, tokenURI);
    }
    
    function toggleSale() external onlyOwner {
        publicSaleActive = !publicSaleActive;
    }
    
    function setMintPrice(uint256 newPrice) external onlyOwner {
        mintPrice = newPrice;
    }
    
    function withdraw() external onlyOwner {
        (bool success,) = owner().call{value: address(this).balance}("");
        require(success, "Withdraw failed");
    }
    
    function totalSupply() external view returns (uint256) {
        return _tokenIdCounter;
    }
}
```

---

## Part 2: ERC-1155 — Multi-Token Standard (2 hours)

### 2.1 Why ERC-1155?

ERC-1155 supports **multiple token types in one contract** — both fungible and non-fungible:

| Token Type | Example | How |
|-----------|---------|-----|
| Fungible | Gold coins (ID: 1) | Many identical copies |
| Semi-fungible | Event ticket (ID: 500, limited to 100) | Limited supply per type |
| Non-fungible | Unique sword (ID: 99999) | Quantity = 1 |

**Gas advantage:** Transfer 100 different tokens in ONE transaction (ERC-1155 batch).

**Use cases:**
- Gaming items (swords, potions, land — all in one contract)
- Music/art platforms (editions)
- DeFi receipts (Uniswap V3 LP positions are ERC-1155)

---

### 2.2 ERC-1155 Interface

```solidity
interface IERC1155 {
    // ─── Balance ─────────────────────────────────────────────────────
    
    /// @notice Balance of specific token type for an address
    function balanceOf(address account, uint256 id) external view returns (uint256);
    
    /// @notice Batch balance query (more efficient than multiple calls)
    function balanceOfBatch(
        address[] calldata accounts, 
        uint256[] calldata ids
    ) external view returns (uint256[] memory);
    
    // ─── Transfers ────────────────────────────────────────────────────
    
    /// @notice Transfer `value` of token `id` from → to
    function safeTransferFrom(
        address from, address to, uint256 id, uint256 value, bytes calldata data
    ) external;
    
    /// @notice Transfer multiple token types in one tx
    function safeBatchTransferFrom(
        address from, address to,
        uint256[] calldata ids, uint256[] calldata values, bytes calldata data
    ) external;
    
    // ─── Approvals ────────────────────────────────────────────────────
    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address account, address operator) external view returns (bool);
    
    // ─── Events ──────────────────────────────────────────────────────
    event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);
    event TransferBatch(address indexed operator, address indexed from, address indexed to, uint256[] ids, uint256[] values);
    event ApprovalForAll(address indexed account, address indexed operator, bool approved);
    event URI(string value, uint256 indexed id);
}
```

---

### 2.3 Build a Game Items Contract (ERC-1155)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract GameItems is ERC1155, Ownable {
    using Strings for uint256;
    
    // Token IDs
    uint256 public constant GOLD    = 0;   // fungible currency
    uint256 public constant SILVER  = 1;   // fungible currency
    uint256 public constant SWORD   = 2;   // semi-fungible (limited editions)
    uint256 public constant SHIELD  = 3;   // semi-fungible
    uint256 public constant POTION  = 4;   // fungible consumable
    
    // NFT IDs start at 1000
    uint256 private _nftIdCounter = 1000;
    
    string private _baseURI;
    
    event ItemCrafted(address indexed player, uint256 itemType, uint256 quantity);
    
    constructor(address initialOwner, string memory baseURI) 
        ERC1155(baseURI)
        Ownable(initialOwner)
    {
        _baseURI = baseURI;
        
        // Mint initial game economy
        _mint(initialOwner, GOLD, 1_000_000, "");
        _mint(initialOwner, SILVER, 10_000_000, "");
        _mint(initialOwner, POTION, 100_000, "");
    }
    
    /// @notice Award starting items to new players
    function newPlayerBonus(address player) external onlyOwner {
        uint256[] memory ids = new uint256[](3);
        uint256[] memory amounts = new uint256[](3);
        
        ids[0] = GOLD;    amounts[0] = 100;
        ids[1] = SILVER;  amounts[1] = 1000;
        ids[2] = POTION;  amounts[2] = 5;
        
        _mintBatch(player, ids, amounts, "");
    }
    
    /// @notice Craft a sword by burning resources
    function craftSword() external {
        // Require materials
        require(balanceOf(msg.sender, GOLD) >= 50, "Need 50 gold");
        require(balanceOf(msg.sender, SILVER) >= 200, "Need 200 silver");
        
        // Burn materials
        _burn(msg.sender, GOLD, 50);
        _burn(msg.sender, SILVER, 200);
        
        // Mint sword
        _mint(msg.sender, SWORD, 1, "");
        
        emit ItemCrafted(msg.sender, SWORD, 1);
    }
    
    /// @notice Mint unique NFT item
    function mintUniqueItem(address to, string calldata uri) external onlyOwner returns (uint256) {
        uint256 tokenId = _nftIdCounter++;
        _mint(to, tokenId, 1, "");
        // Note: ERC1155URIStorage can handle per-token URIs
        return tokenId;
    }
    
    function uri(uint256 tokenId) public view override returns (string memory) {
        return string(abi.encodePacked(_baseURI, tokenId.toString(), ".json"));
    }
}
```

---

## Part 3: Comparison Table and Exercises (1.5 hours)

### ERC-20 vs ERC-721 vs ERC-1155

| Feature | ERC-20 | ERC-721 | ERC-1155 |
|---------|--------|---------|---------|
| Fungible | Yes | No | Both |
| Per-token identity | No | Yes (tokenId) | Yes (id) |
| Batch transfer | No | No | Yes |
| Gas efficiency | High | Medium | Highest |
| Use case | Currency | Collectibles | Gaming, editions |
| Contract per type | Yes | Yes | No (all in one) |

### Exercise: Query OpenSea NFT Data

```javascript
// Query any ERC-721 contract's token data
const { ethers } = require('ethers')

const ERC721_ABI = [
  "function ownerOf(uint256 tokenId) view returns (address)",
  "function tokenURI(uint256 tokenId) view returns (string)",
  "function balanceOf(address owner) view returns (uint256)",
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function totalSupply() view returns (uint256)",
]

async function inspectNFT(contractAddress, tokenId) {
  const provider = new ethers.JsonRpcProvider(process.env.ALCHEMY_MAINNET_URL)
  const nft = new ethers.Contract(contractAddress, ERC721_ABI, provider)
  
  const [name, symbol, owner, tokenURI] = await Promise.all([
    nft.name(),
    nft.symbol(),
    nft.ownerOf(tokenId),
    nft.tokenURI(tokenId),
  ])
  
  console.log(`${name} (${symbol})`)
  console.log(`Token #${tokenId} owned by: ${owner}`)
  console.log(`Metadata URI: ${tokenURI}`)
  
  // Fetch metadata (if IPFS or HTTP)
  if (tokenURI.startsWith("ipfs://")) {
    const httpURI = tokenURI.replace("ipfs://", "https://ipfs.io/ipfs/")
    const metadata = await fetch(httpURI).then(r => r.json())
    console.log("Name:", metadata.name)
    console.log("Description:", metadata.description)
    console.log("Attributes:", metadata.attributes)
  }
}

// BAYC (Bored Ape Yacht Club) contract, token #1
inspectNFT("0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D", 1)
```

---

## Resources for Day 8

| Resource | Link | Type |
|----------|------|------|
| EIP-721 Spec | https://eips.ethereum.org/EIPS/eip-721 | Spec |
| EIP-1155 Spec | https://eips.ethereum.org/EIPS/eip-1155 | Spec |
| OpenZeppelin ERC721 | https://docs.openzeppelin.com/contracts/5.x/erc721 | Docs |
| OpenZeppelin ERC1155 | https://docs.openzeppelin.com/contracts/5.x/erc1155 | Docs |
| BAYC Contract (mainnet) | https://etherscan.io/address/0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D | Explorer |
| NFT School | https://nftschool.dev/ | Tutorial |

---

## Tomorrow

[Day 9 → Liquidity Pools & AMM Theory](./Day-2-Liquidity-Pools-AMM-Theory.md)
