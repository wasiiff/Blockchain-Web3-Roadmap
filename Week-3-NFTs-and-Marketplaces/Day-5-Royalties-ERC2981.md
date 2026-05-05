# Day 19 (Week 3, Day 5) — Royalties & ERC-2981

> **Time:** 6–8 hours  
> **Goal:** Implement ERC-2981 on-chain royalties. Understand how royalties work, why enforcement is hard, and the current state of creator economics.

---

## Part 1: ERC-2981 — The Royalty Standard (1.5 hours)

### 1.1 What Is ERC-2981?

ERC-2981 is a standard for **on-chain royalty information**. The NFT contract declares:
- Who receives royalties
- What percentage

Any marketplace that checks ERC-2981 can automatically pay royalties on secondary sales.

```solidity
interface IERC2981 {
    /// @notice Called with the sale price to determine how much royalty
    ///         is owed and to whom.
    /// @param tokenId - the NFT asset queried for royalty information
    /// @param salePrice - the sale price of the NFT asset specified by tokenId
    /// @return receiver - address of who should be sent the royalty payment
    /// @return royaltyAmount - the royalty payment amount for salePrice
    function royaltyInfo(
        uint256 tokenId,
        uint256 salePrice
    ) external view returns (
        address receiver,
        uint256 royaltyAmount
    );
}
```

### 1.2 Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/interfaces/IERC2981.sol";

// OpenZeppelin provides a full implementation:
import "@openzeppelin/contracts/token/common/ERC2981.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract RoyaltyNFT is ERC721, ERC2981, Ownable {
    
    constructor(address initialOwner) ERC721("Royalty NFT", "RNFT") Ownable(initialOwner) {
        // Set default royalty: initialOwner gets 5% on all sales
        _setDefaultRoyalty(initialOwner, 500); // 500 = 5%
    }
    
    // Set per-token royalty (overrides default for specific token)
    function setTokenRoyalty(uint256 tokenId, address receiver, uint96 feeNumerator) 
        external onlyOwner 
    {
        _setTokenRoyalty(tokenId, receiver, feeNumerator);
    }
    
    // Split royalties between multiple addresses (manual split)
    function setRoyaltyWithSplit(address[] calldata receivers, uint256[] calldata shares) 
        external onlyOwner 
    {
        // Note: ERC-2981 only supports one receiver!
        // For splits, point to a Splitter contract (like 0xSplits)
        require(receivers.length == shares.length);
        // Deploy a splitter and set it as royalty receiver
        // See: https://docs.0xsplits.xyz/
    }
    
    // Required override for multiple inheritance
    function supportsInterface(bytes4 interfaceId)
        public view override(ERC721, ERC2981) returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
    
    // ─── Royalty Query Examples ───────────────────────────────────────
    
    function getRoyaltyInfo(uint256 tokenId, uint256 salePrice) 
        external view returns (address receiver, uint256 amount) 
    {
        return royaltyInfo(tokenId, salePrice);
    }
}
```

### 1.3 Royalty Fee Calculation

```
ERC-2981 fee denominator = 10,000 (like basis points)

feeNumerator = 500 → 5%
feeNumerator = 250 → 2.5%
feeNumerator = 1000 → 10%
feeNumerator = 10000 → 100% (invalid in practice)

royaltyAmount = (salePrice × feeNumerator) / 10,000

Example:
Sale price = 1 ETH = 10^18 wei
Royalty = 5% = 500 bps
royaltyAmount = 10^18 × 500 / 10,000 = 5 × 10^16 wei = 0.05 ETH
```

---

## Part 2: The Royalty Enforcement Problem (1 hour)

### Why Royalties Are Hard to Enforce

ERC-2981 only **declares** royalties — it does not **enforce** them.

A marketplace can read `royaltyInfo()` and pay accordingly, OR it can simply ignore it.

**The 2022-2023 Royalty Wars:**

1. **Before 2022:** OpenSea enforced royalties. All marketplaces did.
2. **Late 2022:** X2Y2 and LooksRare started offering optional royalties to attract sellers.
3. **Blur** (Nov 2022): Launched with 0% royalties, attracted massive volume.
4. **The Effect:** NFT collections lost significant creator revenue. Many artists saw income drop 80%+.
5. **Response:** 
   - NFT contracts started using transfer restrictions (operator filter)
   - OpenSea adopted "Operator Filter Registry"
   - Creator Earnings Protection Tools emerged
6. **2023 onwards:** Most major collections accept that royalties are partially optional off-chain.

### The Operator Filter Approach

```solidity
// One approach: block non-royalty-paying marketplaces from transfers
// OpenSea's OperatorFilter (controversial, eventually dropped)

import {OperatorFilterer} from "operator-filter-registry/src/OperatorFilterer.sol";

contract FilteredNFT is ERC721, OperatorFilterer {
    constructor() ERC721("Filtered", "FLTD") OperatorFilterer(
        0x3cc6CddA760b79bAfa08dF41ECFA224f810dCeB6, // default subscription
        true
    ) {}
    
    // Only operators not on the blocklist can transfer
    function transferFrom(address from, address to, uint256 tokenId)
        public override onlyAllowedOperator(from) 
    {
        super.transferFrom(from, to, tokenId);
    }
    
    function safeTransferFrom(address from, address to, uint256 tokenId)
        public override onlyAllowedOperator(from)
    {
        super.safeTransferFrom(from, to, tokenId);
    }
}
```

**Why this failed:**
- It blocks legitimate use cases (gifting to wallets, DeFi collateral)
- Centralized: a "registry" owner could censor any marketplace
- OpenSea itself dropped it in 2023
- The community backlash was significant

### The 0xSplits Solution for Royalties

```
0xSplits: A protocol for trustless, on-chain revenue splits

Use case: NFT with 3 co-creators, each gets 33.3% of royalties

Deploy a Split contract → receives royalties → auto-distributes to [Alice 33%, Bob 33%, Carol 33%]
```

```javascript
// 0xSplits SDK
const { SplitMainABI, SPLIT_MAIN_ADDRESS } = require('@0xsplits/splits-sdk')

async function createRoyaltySplit(recipients, allocations) {
  const splitMain = new ethers.Contract(SPLIT_MAIN_ADDRESS, SplitMainABI, signer)
  
  const tx = await splitMain.createSplit(
    recipients,    // [alice, bob, carol]
    allocations,   // [333333, 333333, 333334] (must sum to 1,000,000)
    0,             // distributor fee (0% = no incentive for distribution)
    ethers.ZeroAddress  // no controller (immutable)
  )
  
  const receipt = await tx.wait()
  // Get the new split address from event
  const splitAddress = receipt.events[0].args.split
  
  // Now set this address as the royalty receiver in your NFT contract
  return splitAddress
}
```

---

## Part 3: Advanced Royalty Patterns (1.5 hours)

### Dutch Auction Minting

A Dutch Auction starts at a high price and decreases over time. Final mint price is the clearing price:

```solidity
// Dutch Auction
contract DutchAuctionMint {
    uint256 public constant AUCTION_START_PRICE = 1 ether;
    uint256 public constant AUCTION_END_PRICE = 0.1 ether;
    uint256 public constant AUCTION_DURATION = 1 hours;
    
    uint256 public auctionStartTime;
    
    function startAuction() external onlyOwner {
        auctionStartTime = block.timestamp;
    }
    
    function getCurrentPrice() public view returns (uint256) {
        if (block.timestamp >= auctionStartTime + AUCTION_DURATION) {
            return AUCTION_END_PRICE;
        }
        
        uint256 elapsed = block.timestamp - auctionStartTime;
        uint256 priceDrop = (AUCTION_START_PRICE - AUCTION_END_PRICE) * elapsed / AUCTION_DURATION;
        return AUCTION_START_PRICE - priceDrop;
    }
    
    function mint() external payable {
        uint256 currentPrice = getCurrentPrice();
        require(msg.value >= currentPrice, "Below current price");
        
        // Refund excess
        if (msg.value > currentPrice) {
            payable(msg.sender).transfer(msg.value - currentPrice);
        }
        
        // Mint NFT...
    }
}
```

### English Auction (Time-based bidding)

```solidity
contract EnglishAuction {
    struct Auction {
        address seller;
        uint256 tokenId;
        uint256 highestBid;
        address highestBidder;
        uint256 endTime;
        bool ended;
    }
    
    mapping(uint256 => Auction) public auctions;
    mapping(address => uint256) public pendingReturns; // outbid amounts to claim
    
    function startAuction(uint256 tokenId, uint256 startingBid, uint256 duration) external {
        IERC721(nftContract).transferFrom(msg.sender, address(this), tokenId);
        auctions[tokenId] = Auction({
            seller: msg.sender,
            tokenId: tokenId,
            highestBid: startingBid,
            highestBidder: address(0),
            endTime: block.timestamp + duration,
            ended: false
        });
    }
    
    function bid(uint256 tokenId) external payable {
        Auction storage auction = auctions[tokenId];
        require(block.timestamp < auction.endTime, "Auction ended");
        require(msg.value > auction.highestBid, "Bid too low");
        
        // Refund previous bidder
        if (auction.highestBidder != address(0)) {
            pendingReturns[auction.highestBidder] += auction.highestBid;
        }
        
        auction.highestBid = msg.value;
        auction.highestBidder = msg.sender;
    }
    
    function endAuction(uint256 tokenId) external {
        Auction storage auction = auctions[tokenId];
        require(block.timestamp >= auction.endTime, "Not ended");
        require(!auction.ended, "Already ended");
        
        auction.ended = true;
        
        if (auction.highestBidder != address(0)) {
            IERC721(nftContract).transferFrom(address(this), auction.highestBidder, tokenId);
            payable(auction.seller).transfer(auction.highestBid);
        } else {
            IERC721(nftContract).transferFrom(address(this), auction.seller, tokenId);
        }
    }
    
    function claimRefund() external {
        uint256 amount = pendingReturns[msg.sender];
        if (amount > 0) {
            pendingReturns[msg.sender] = 0;
            payable(msg.sender).transfer(amount);
        }
    }
}
```

---

## Resources for Day 19

| Resource | Link | Type |
|----------|------|------|
| EIP-2981 | https://eips.ethereum.org/EIPS/eip-2981 | Spec |
| 0xSplits | https://docs.0xsplits.xyz/ | Docs |
| OpenZeppelin ERC2981 | https://docs.openzeppelin.com/contracts/5.x/api/token/common#ERC2981 | Docs |
| Royalty Registry | https://royaltyregistry.xyz/ | Tool |
| The Royalty Wars (Analysis) | https://nftgo.io/analytics/ | Reference |

---

## Tomorrow

[Day 20 → NFT Frontend](./Day-6-NFT-Frontend.md)
