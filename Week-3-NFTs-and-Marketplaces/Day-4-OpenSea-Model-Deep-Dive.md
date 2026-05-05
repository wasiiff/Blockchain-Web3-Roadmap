# Day 18 (Week 3, Day 4) — OpenSea Model Deep Dive

> **Time:** 6–8 hours  
> **Goal:** Understand exactly how OpenSea works — from listing to purchase. The Seaport protocol, off-chain orders, and on-chain execution.

---

## Part 1: How OpenSea Works End-to-End (2 hours)

### 1.1 The Flow of an OpenSea Sale

```
SELLER SIDE:
1. Seller connects MetaMask
2. Seller calls setApprovalForAll(Seaport, true) — ONE TIME transaction
   (After this, Seaport can transfer any of your NFTs)
3. Seller sets price in OpenSea UI
4. Seller signs an "order" message — NO transaction, NO gas
   The signed order is stored on OpenSea's server
   
BUYER SIDE:
5. Buyer finds the listing on opensea.io
6. Buyer clicks "Buy Now"
7. Buyer calls Seaport.fulfillOrder(signedOrder)
   — This is the ONLY transaction (buyer pays gas)
   — Atomically: transfers NFT from seller, transfers ETH to seller, fees to OpenSea, royalties to creator
8. Done!
```

**Key insight: Listing is free (off-chain signed message). Buyer pays all gas.**

### 1.2 The EIP-712 Signed Order

```javascript
// The order that the seller signs (never goes on-chain until someone buys)
const order = {
  offerer: sellerAddress,            // who is selling
  zone: "0x000...",                  // advanced: zone contract for conditional orders
  offer: [
    {
      itemType: 2,                   // ERC-721
      token: nftContractAddress,
      identifierOrCriteria: tokenId, // which token
      startAmount: 1,
      endAmount: 1,
    }
  ],
  consideration: [
    {
      itemType: 0,                   // ETH
      token: "0x000...",
      identifierOrCriteria: 0,
      startAmount: ethers.parseEther("1"), // seller gets 97.5%
      endAmount: ethers.parseEther("1"),
      recipient: sellerAddress,
    },
    {
      itemType: 0,                   // ETH
      token: "0x000...",
      identifierOrCriteria: 0,
      startAmount: ethers.parseEther("0.025"), // OpenSea gets 2.5%
      endAmount: ethers.parseEther("0.025"),
      recipient: openSeaFeeAddress,
    },
    {
      itemType: 0,                   // ETH  
      token: "0x000...",
      identifierOrCriteria: 0,
      startAmount: ethers.parseEther("0.05"), // creator royalty 5%
      endAmount: ethers.parseEther("0.05"),
      recipient: creatorAddress,
    }
  ],
  orderType: 0,                      // FULL_OPEN (anyone can fill)
  startTime: Math.floor(Date.now() / 1000),
  endTime: Math.floor(Date.now() / 1000) + 86400 * 7, // 7 days
  zoneHash: "0x000...",
  salt: Math.random().toString(),    // unique per order
  conduitKey: "0x000...",
  counter: 0,
}

// Seller signs this with EIP-712 (typedData signing)
const signature = await seller.signTypedData(domain, types, order)
```

### 1.3 Why Seaport is Revolutionary

**Before Seaport (OpenSea V1 — Wyvern protocol):**
- Listing cost ~$10-50 in gas (on-chain approval per item)
- Complex and hard to audit

**Seaport (2022):**
- Listings are free (off-chain signed orders)
- One approval covers all listings: `setApprovalForAll(Seaport, true)`
- Supports complex orders: bundles, Dutch auctions, offers, swaps
- Fully open source, anyone can build on it
- Gas optimized (heavily uses assembly, storage tricks)

**Item types:**
```
0 = Native ETH
1 = ERC-20 token
2 = ERC-721 NFT (specific tokenId)
3 = ERC-1155 token
4 = ERC-721 with criteria (any tokenId matching a Merkle tree — "any ape")
5 = ERC-1155 with criteria
```

### 1.4 Seaport Order Types

```
FULL_OPEN (0):      Anyone can fill the entire order
PARTIAL_OPEN (1):   Partial fills (e.g., sell 5 of 10 ERC-1155)
FULL_RESTRICTED (2): Only zone contract can validate filling
PARTIAL_RESTRICTED (3): Partial + zone validation
```

**Conduits:** Pre-approved transfer channels. A "conduit" is a contract that can transfer on behalf of users who approved it. Multiple marketplaces can share one conduit approval.

---

## Part 2: OpenSea's Revenue Model

### Fee Structure (Current as of 2025)

| Fee Type | Rate | Goes To |
|---------|------|---------|
| Platform fee | 2.5% | OpenSea |
| Creator royalty | 0-10% (creator sets) | NFT creator |
| Gas | Variable | Validators |

**Royalty controversy:**
- Original model: OpenSea enforced creator royalties
- 2022-2023: Blur launched with 0% royalties → drew volume away from OpenSea
- OpenSea response: Made royalties optional (off-chain enforcement)
- The "Royalty Wars" led to EIP-2981 adoption and on-chain royalty standards

### How OpenSea Makes Money

1. **Trading fees:** 2.5% of every sale
2. **Data:** On-chain activity provides valuable market data
3. **Premium features:** OpenSea Pro subscription

**Revenue scale:** OpenSea processed $20B+ in volume in 2021-2022. At 2.5% fee → ~$500M+ in fees.

---

## Part 3: Implement Off-Chain Order Signing (2 hours)

### Order Book Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title OrderBook — OpenSea-style off-chain signed orders
contract OrderBook is EIP712, ReentrancyGuard {
    using ECDSA for bytes32;
    
    // ─── Types ────────────────────────────────────────────────────────
    
    struct Order {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        uint256 startTime;
        uint256 endTime;
        uint256 nonce;       // to allow cancellation
    }
    
    bytes32 private constant ORDER_TYPEHASH = keccak256(
        "Order(address seller,address nftContract,uint256 tokenId,uint256 price,uint256 startTime,uint256 endTime,uint256 nonce)"
    );
    
    // ─── State ────────────────────────────────────────────────────────
    
    uint256 public platformFeeBps = 250; // 2.5%
    address public feeRecipient;
    
    // seller → nonce → cancelled/filled
    mapping(address => mapping(uint256 => bool)) public cancelledOrFilled;
    mapping(address => uint256) public minNonce; // cancel all orders below this nonce
    
    // ─── Events ──────────────────────────────────────────────────────
    
    event OrderFulfilled(
        bytes32 indexed orderHash,
        address indexed seller,
        address indexed buyer,
        uint256 tokenId,
        uint256 price
    );
    event OrderCancelled(bytes32 indexed orderHash, address indexed seller);
    event AllOrdersCancelled(address indexed seller, uint256 newMinNonce);
    
    constructor(address _feeRecipient) 
        EIP712("OrderBook", "1") 
    {
        feeRecipient = _feeRecipient;
    }
    
    // ─── Buying ───────────────────────────────────────────────────────
    
    /// @notice Fulfill a signed order (buyer calls this)
    function fulfillOrder(Order calldata order, bytes calldata signature) 
        external payable nonReentrant 
    {
        // Hash the order
        bytes32 orderHash = _hashOrder(order);
        
        // Verify signature
        address recoveredSeller = _hashTypedDataV4(orderHash).recover(signature);
        require(recoveredSeller == order.seller, "Invalid signature");
        
        // Validate order
        require(block.timestamp >= order.startTime, "Not started");
        require(block.timestamp <= order.endTime, "Expired");
        require(!cancelledOrFilled[order.seller][order.nonce], "Order cancelled or filled");
        require(order.nonce >= minNonce[order.seller], "Nonce too low");
        require(msg.value >= order.price, "Insufficient payment");
        
        // Mark as filled
        cancelledOrFilled[order.seller][order.nonce] = true;
        
        // Calculate fees
        uint256 platformFee = (order.price * platformFeeBps) / 10000;
        uint256 sellerProceeds = order.price - platformFee;
        
        // Execute transfers atomically
        IERC721(order.nftContract).transferFrom(order.seller, msg.sender, order.tokenId);
        
        _sendETH(feeRecipient, platformFee);
        _sendETH(order.seller, sellerProceeds);
        
        // Refund excess
        if (msg.value > order.price) {
            _sendETH(msg.sender, msg.value - order.price);
        }
        
        emit OrderFulfilled(orderHash, order.seller, msg.sender, order.tokenId, order.price);
    }
    
    // ─── Cancellation ─────────────────────────────────────────────────
    
    /// @notice Cancel a specific order by nonce
    function cancelOrder(Order calldata order, bytes calldata signature) external {
        bytes32 orderHash = _hashOrder(order);
        address recoveredSeller = _hashTypedDataV4(orderHash).recover(signature);
        
        require(recoveredSeller == msg.sender, "Not your order");
        cancelledOrFilled[msg.sender][order.nonce] = true;
        
        emit OrderCancelled(orderHash, msg.sender);
    }
    
    /// @notice Cancel ALL orders with nonce below newMinNonce
    function cancelAllOrdersBelowNonce(uint256 newMinNonce) external {
        require(newMinNonce > minNonce[msg.sender], "Must be higher");
        minNonce[msg.sender] = newMinNonce;
        emit AllOrdersCancelled(msg.sender, newMinNonce);
    }
    
    // ─── View ─────────────────────────────────────────────────────────
    
    function getOrderHash(Order calldata order) external view returns (bytes32) {
        return _hashTypedDataV4(_hashOrder(order));
    }
    
    function isValidOrder(Order calldata order, bytes calldata signature) 
        external view returns (bool) 
    {
        if (block.timestamp < order.startTime || block.timestamp > order.endTime) return false;
        if (cancelledOrFilled[order.seller][order.nonce]) return false;
        if (order.nonce < minNonce[order.seller]) return false;
        
        bytes32 orderHash = _hashTypedDataV4(_hashOrder(order));
        address recovered = orderHash.recover(signature);
        return recovered == order.seller;
    }
    
    // ─── Internal ─────────────────────────────────────────────────────
    
    function _hashOrder(Order calldata order) internal pure returns (bytes32) {
        return keccak256(abi.encode(
            ORDER_TYPEHASH,
            order.seller,
            order.nftContract,
            order.tokenId,
            order.price,
            order.startTime,
            order.endTime,
            order.nonce
        ));
    }
    
    function _sendETH(address to, uint256 amount) internal {
        (bool success,) = payable(to).call{value: amount}("");
        require(success, "ETH transfer failed");
    }
}
```

### Off-Chain Order Signing (Frontend)

```typescript
// Creating and signing a sell order
import { useSignTypedData, useAccount } from 'wagmi'

const ORDER_TYPES = {
  Order: [
    { name: 'seller', type: 'address' },
    { name: 'nftContract', type: 'address' },
    { name: 'tokenId', type: 'uint256' },
    { name: 'price', type: 'uint256' },
    { name: 'startTime', type: 'uint256' },
    { name: 'endTime', type: 'uint256' },
    { name: 'nonce', type: 'uint256' },
  ],
}

function useCreateOrder(nftContract: string, tokenId: bigint, price: bigint) {
  const { address } = useAccount()
  const { signTypedDataAsync } = useSignTypedData()
  
  const createOrder = async () => {
    const order = {
      seller: address!,
      nftContract: nftContract as `0x${string}`,
      tokenId,
      price,
      startTime: BigInt(Math.floor(Date.now() / 1000)),
      endTime: BigInt(Math.floor(Date.now() / 1000) + 86400 * 7),
      nonce: BigInt(Date.now()), // use timestamp as unique nonce
    }
    
    const signature = await signTypedDataAsync({
      domain: {
        name: 'OrderBook',
        version: '1',
        chainId: 11155111, // sepolia
        verifyingContract: ORDERBOOK_ADDRESS,
      },
      types: ORDER_TYPES,
      primaryType: 'Order',
      message: order,
    })
    
    // Store order + signature in your database / localStorage
    const listing = { order, signature }
    await saveToDatabase(listing) // your backend
    
    return listing
  }
  
  return { createOrder }
}
```

---

## Resources for Day 18

| Resource | Link | Type |
|----------|------|------|
| Seaport Protocol | https://github.com/ProjectOpenSea/seaport | Code |
| Seaport Docs | https://docs.opensea.io/reference/seaport-overview | Docs |
| OpenSea Blog | https://opensea.io/blog | Articles |
| EIP-712 TypedData | https://eips.ethereum.org/EIPS/eip-712 | Spec |
| Blur Marketplace | https://github.com/code-423n4/2022-11-blur | Code |

---

## Tomorrow

[Day 19 → Royalties & ERC-2981](./Day-5-Royalties-ERC2981.md)
