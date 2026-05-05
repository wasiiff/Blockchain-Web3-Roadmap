# Day 17 (Week 3, Day 3) — NFT Marketplace Architecture

> **Time:** 6–8 hours  
> **Goal:** Design and implement an NFT marketplace — the listing, escrow, buying, and cancellation flow.

---

## Part 1: Marketplace Models (1 hour)

### Three Models for NFT Marketplaces

**Model 1: Escrow-Based**
```
Seller → transfers NFT to marketplace contract
Buyer → sends ETH to marketplace contract
Marketplace → gives NFT to buyer, ETH (minus fee) to seller
```
- Pros: Simple, clear ownership
- Cons: Seller loses custody, gas for transfer on list AND on buy

**Model 2: Approval-Based (OpenSea model)**
```
Seller → setApprovalForAll(marketplace, true) (one time)
Seller → signs an "order" off-chain (no gas!)
Buyer → calls marketplace.fulfillOrder() with the signed order
Marketplace → transferFrom(seller, buyer, tokenId)  simultaneously with ETH payment
```
- Pros: Listing is free (off-chain), seller keeps NFT until sold
- Cons: More complex, seller must maintain approval

**Model 3: Hybrid**
- Orders signed off-chain, stored on central server (like OpenSea)
- When buying, on-chain execution validates signature + transfers

**We'll build Model 1 (escrow)** as it's simpler and teaches the core concepts.

---

## Part 2: Full Marketplace Contract (3 hours)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/interfaces/IERC2981.sol";

/// @title NFT Marketplace — List, buy, cancel with royalty support
contract NFTMarketplace is ReentrancyGuard, Ownable {
    
    // ─── State ────────────────────────────────────────────────────────
    
    struct Listing {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;        // in ETH (wei)
        bool active;
    }
    
    // listingId → Listing
    mapping(uint256 => Listing) public listings;
    uint256 private _listingIdCounter;
    
    // Platform fee (in basis points: 250 = 2.5%)
    uint256 public platformFeeBps = 250;
    uint256 private constant MAX_FEE_BPS = 1000; // max 10%
    address public feeRecipient;
    
    // ─── Errors ───────────────────────────────────────────────────────
    error NotOwner();
    error NotActive();
    error InsufficientPayment(uint256 sent, uint256 required);
    error PriceTooLow();
    error TransferFailed();
    error FeeTooHigh();
    error ZeroAddress();
    
    // ─── Events ──────────────────────────────────────────────────────
    event Listed(
        uint256 indexed listingId,
        address indexed seller,
        address indexed nftContract,
        uint256 tokenId,
        uint256 price
    );
    event Sold(
        uint256 indexed listingId,
        address indexed buyer,
        address indexed nftContract,
        uint256 tokenId,
        uint256 price
    );
    event Cancelled(uint256 indexed listingId);
    event PriceUpdated(uint256 indexed listingId, uint256 oldPrice, uint256 newPrice);
    
    constructor(address initialOwner, address _feeRecipient) Ownable(initialOwner) {
        if (_feeRecipient == address(0)) revert ZeroAddress();
        feeRecipient = _feeRecipient;
    }
    
    // ─── Listing ──────────────────────────────────────────────────────
    
    /// @notice List an NFT for sale
    /// @dev NFT must be approved to this contract first (setApprovalForAll or approve)
    function list(
        address nftContract,
        uint256 tokenId,
        uint256 price
    ) external returns (uint256 listingId) {
        if (price == 0) revert PriceTooLow();
        
        IERC721 nft = IERC721(nftContract);
        
        // Verify caller owns the NFT
        if (nft.ownerOf(tokenId) != msg.sender) revert NotOwner();
        
        // Transfer NFT to marketplace (escrow)
        nft.transferFrom(msg.sender, address(this), tokenId);
        
        listingId = _listingIdCounter++;
        listings[listingId] = Listing({
            seller: msg.sender,
            nftContract: nftContract,
            tokenId: tokenId,
            price: price,
            active: true
        });
        
        emit Listed(listingId, msg.sender, nftContract, tokenId, price);
    }
    
    // ─── Buying ───────────────────────────────────────────────────────
    
    /// @notice Buy a listed NFT
    function buy(uint256 listingId) external payable nonReentrant {
        Listing storage listing = listings[listingId];
        
        if (!listing.active) revert NotActive();
        if (msg.value < listing.price) revert InsufficientPayment(msg.value, listing.price);
        
        // Mark inactive first (checks-effects-interactions)
        listing.active = false;
        
        uint256 price = listing.price;
        address seller = listing.seller;
        address nftContract = listing.nftContract;
        uint256 tokenId = listing.tokenId;
        
        // Calculate fees
        uint256 platformFee = (price * platformFeeBps) / 10000;
        uint256 royaltyFee = 0;
        address royaltyRecipient = address(0);
        
        // Check ERC-2981 royalties
        if (IERC165(nftContract).supportsInterface(type(IERC2981).interfaceId)) {
            (address receiver, uint256 royaltyAmount) = IERC2981(nftContract).royaltyInfo(tokenId, price);
            if (receiver != address(0) && royaltyAmount > 0) {
                royaltyFee = royaltyAmount;
                royaltyRecipient = receiver;
            }
        }
        
        uint256 sellerProceeds = price - platformFee - royaltyFee;
        
        // Transfer NFT to buyer
        IERC721(nftContract).transferFrom(address(this), msg.sender, tokenId);
        
        // Distribute payments
        _sendETH(feeRecipient, platformFee);
        if (royaltyFee > 0 && royaltyRecipient != address(0)) {
            _sendETH(royaltyRecipient, royaltyFee);
        }
        _sendETH(seller, sellerProceeds);
        
        // Refund excess payment
        if (msg.value > price) {
            _sendETH(msg.sender, msg.value - price);
        }
        
        emit Sold(listingId, msg.sender, nftContract, tokenId, price);
    }
    
    // ─── Cancel & Update ──────────────────────────────────────────────
    
    /// @notice Cancel your listing and get NFT back
    function cancel(uint256 listingId) external nonReentrant {
        Listing storage listing = listings[listingId];
        if (!listing.active) revert NotActive();
        if (listing.seller != msg.sender) revert NotOwner();
        
        listing.active = false;
        
        IERC721(listing.nftContract).transferFrom(address(this), msg.sender, listing.tokenId);
        
        emit Cancelled(listingId);
    }
    
    /// @notice Update listing price
    function updatePrice(uint256 listingId, uint256 newPrice) external {
        Listing storage listing = listings[listingId];
        if (!listing.active) revert NotActive();
        if (listing.seller != msg.sender) revert NotOwner();
        if (newPrice == 0) revert PriceTooLow();
        
        uint256 oldPrice = listing.price;
        listing.price = newPrice;
        
        emit PriceUpdated(listingId, oldPrice, newPrice);
    }
    
    // ─── View Functions ───────────────────────────────────────────────
    
    /// @notice Get all active listings (use off-chain indexing in production!)
    function getActiveListings() external view returns (Listing[] memory) {
        uint256 activeCount = 0;
        for (uint256 i = 0; i < _listingIdCounter; i++) {
            if (listings[i].active) activeCount++;
        }
        
        Listing[] memory active = new Listing[](activeCount);
        uint256 idx = 0;
        for (uint256 i = 0; i < _listingIdCounter; i++) {
            if (listings[i].active) {
                active[idx++] = listings[i];
            }
        }
        return active;
    }
    
    /// @notice Calculate breakdown of fees for a given price
    function getFeeBreakdown(address nftContract, uint256 tokenId, uint256 price)
        external view returns (
            uint256 platformFee,
            uint256 royaltyFee,
            address royaltyRecipient,
            uint256 sellerProceeds
        )
    {
        platformFee = (price * platformFeeBps) / 10000;
        royaltyFee = 0;
        royaltyRecipient = address(0);
        
        if (IERC165(nftContract).supportsInterface(type(IERC2981).interfaceId)) {
            (address receiver, uint256 royaltyAmount) = IERC2981(nftContract).royaltyInfo(tokenId, price);
            royaltyFee = royaltyAmount;
            royaltyRecipient = receiver;
        }
        
        sellerProceeds = price - platformFee - royaltyFee;
    }
    
    // ─── Admin ────────────────────────────────────────────────────────
    
    function setPlatformFee(uint256 feeBps) external onlyOwner {
        if (feeBps > MAX_FEE_BPS) revert FeeTooHigh();
        platformFeeBps = feeBps;
    }
    
    function setFeeRecipient(address recipient) external onlyOwner {
        if (recipient == address(0)) revert ZeroAddress();
        feeRecipient = recipient;
    }
    
    // ─── Internal ─────────────────────────────────────────────────────
    
    function _sendETH(address to, uint256 amount) internal {
        if (amount == 0) return;
        (bool success,) = payable(to).call{value: amount}("");
        if (!success) revert TransferFailed();
    }
}
```

---

## Part 3: Marketplace Tests (2 hours)

```javascript
// test/NFTMarketplace.test.js
const { expect } = require("chai")
const { ethers } = require("hardhat")
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers")

describe("NFTMarketplace", function () {
  async function deployFixture() {
    const [owner, seller, buyer, feeRecipient] = await ethers.getSigners()
    
    // Deploy NFT
    const ArtBlockNFT = await ethers.getContractFactory("ArtBlockNFT")
    const nft = await ArtBlockNFT.deploy(owner.address, "ipfs://hidden/", 500) // 5% royalty
    
    // Deploy Marketplace
    const Marketplace = await ethers.getContractFactory("NFTMarketplace")
    const marketplace = await Marketplace.deploy(owner.address, feeRecipient.address)
    
    // Mint NFT to seller
    await nft.togglePublicSale()
    await nft.setMintPrice(0)
    await nft.connect(seller).publicMint(1)
    
    return { nft, marketplace, owner, seller, buyer, feeRecipient }
  }
  
  it("should list an NFT", async function () {
    const { nft, marketplace, seller } = await loadFixture(deployFixture)
    
    const tokenId = 0
    const price = ethers.parseEther("1")
    
    await nft.connect(seller).approve(await marketplace.getAddress(), tokenId)
    await expect(
      marketplace.connect(seller).list(await nft.getAddress(), tokenId, price)
    ).to.emit(marketplace, "Listed")
    
    // NFT should be held by marketplace (escrow)
    expect(await nft.ownerOf(tokenId)).to.equal(await marketplace.getAddress())
  })
  
  it("should complete a sale and pay all parties", async function () {
    const { nft, marketplace, seller, buyer, feeRecipient } = await loadFixture(deployFixture)
    
    const tokenId = 0
    const price = ethers.parseEther("1")
    
    await nft.connect(seller).approve(await marketplace.getAddress(), tokenId)
    await marketplace.connect(seller).list(await nft.getAddress(), tokenId, price)
    
    const sellerBalBefore = await ethers.provider.getBalance(seller.address)
    const feeRecipBalBefore = await ethers.provider.getBalance(feeRecipient.address)
    
    await expect(
      marketplace.connect(buyer).buy(0, { value: price })
    ).to.emit(marketplace, "Sold")
    
    // Buyer got the NFT
    expect(await nft.ownerOf(tokenId)).to.equal(buyer.address)
    
    // Seller got 92.5% (2.5% platform fee + 5% royalty)
    const sellerBalAfter = await ethers.provider.getBalance(seller.address)
    const expectedSeller = price * 925n / 1000n
    expect(sellerBalAfter - sellerBalBefore).to.be.closeTo(expectedSeller, ethers.parseEther("0.001"))
    
    // Fee recipient got 2.5%
    const feeRecipBalAfter = await ethers.provider.getBalance(feeRecipient.address)
    expect(feeRecipBalAfter - feeRecipBalBefore).to.equal(price * 25n / 1000n)
  })
  
  it("should cancel listing and return NFT", async function () {
    const { nft, marketplace, seller } = await loadFixture(deployFixture)
    
    const tokenId = 0
    await nft.connect(seller).approve(await marketplace.getAddress(), tokenId)
    await marketplace.connect(seller).list(await nft.getAddress(), tokenId, ethers.parseEther("1"))
    
    await expect(marketplace.connect(seller).cancel(0))
      .to.emit(marketplace, "Cancelled")
    
    expect(await nft.ownerOf(tokenId)).to.equal(seller.address)
  })
  
  it("should not allow non-seller to cancel", async function () {
    const { nft, marketplace, seller, buyer } = await loadFixture(deployFixture)
    
    const tokenId = 0
    await nft.connect(seller).approve(await marketplace.getAddress(), tokenId)
    await marketplace.connect(seller).list(await nft.getAddress(), tokenId, ethers.parseEther("1"))
    
    await expect(
      marketplace.connect(buyer).cancel(0)
    ).to.be.revertedWithCustomError(marketplace, "NotOwner")
  })
})
```

---

## Resources for Day 17

| Resource | Link | Type |
|----------|------|------|
| Seaport Protocol | https://github.com/ProjectOpenSea/seaport | Code |
| OpenSea API Docs | https://docs.opensea.io/reference/ | Docs |
| EIP-2981 | https://eips.ethereum.org/EIPS/eip-2981 | Spec |
| Marketplace Security | https://github.com/d-xo/weird-erc721 | Reference |

---

## Tomorrow

[Day 18 → OpenSea Model Deep Dive](./Day-4-OpenSea-Model-Deep-Dive.md)
