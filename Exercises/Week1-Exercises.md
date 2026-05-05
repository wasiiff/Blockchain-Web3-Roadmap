# Week 1 Exercises — Blockchain Foundations

## Exercise Set 1: Cryptography & Hashing

### E1.1 — Hash Exploration
Using the Keccak-256 tool at https://emn178.github.io/online-tools/keccak_256.html:
1. Hash your full name
2. Hash your name with one character changed
3. Calculate the hash of `"0"` (string zero)
4. Calculate `keccak256(keccak256("hello"))` — hash of a hash

**Question:** Why does a single character change produce a completely different hash?

### E1.2 — Address Derivation
```javascript
// Derive an Ethereum address from a private key
const { ethers } = require('ethers')

// Generate a random wallet
const wallet = ethers.Wallet.createRandom()
console.log("Private Key:", wallet.privateKey)
console.log("Public Key:", wallet.publicKey)
console.log("Address:", wallet.address)

// Verify the derivation manually:
// 1. address = last 20 bytes of keccak256(public_key)
const pubKey = wallet.publicKey.slice(4) // remove '0x04' prefix
const hash = ethers.keccak256('0x' + pubKey)
const derivedAddress = '0x' + hash.slice(-40)
console.log("Derived address matches:", derivedAddress.toLowerCase() === wallet.address.toLowerCase())
```

### E1.3 — Block Explorer
On https://etherscan.io:
1. Find block #19,000,000
2. Record: timestamp, transactions count, gas used, gas limit
3. Find the Merkle root (stateRoot) in the block header
4. Click 3 random transactions and decode their "Input Data"

---

## Exercise Set 2: Solidity Fundamentals

### E2.1 — Storage Layout Analysis
```solidity
// What are the storage slots of each variable?
// Draw the slot layout diagram.
contract StorageTest {
    uint256 a;      // slot ?
    uint128 b;      // slot ?
    uint128 c;      // slot ? (packed with b?)
    bool d;         // slot ?
    address e;      // slot ? (packed with d?)
    uint256[] f;    // slot ? (dynamic array)
    mapping(address => uint256) g; // slot ?
}
```

### E2.2 — Gas Optimization
Rewrite this function to use at least 50% less gas:
```solidity
function sumArray(uint256[] memory arr) public returns (uint256) {
    uint256 sum = 0;
    for (uint256 i = 0; i < arr.length; i++) {
        sum = sum + arr[i];  // find 3 gas optimizations here
    }
    totalSumCalled++;  // state variable
    return sum;
}
```

### E2.3 — Implement Missing Functions
Complete the ERC-20 contract:
```solidity
contract IncompleteERC20 {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // TODO: implement transfer()
    // TODO: implement approve()
    // TODO: implement transferFrom()
    // Hint: each must emit the appropriate event
}
```

---

## Exercise Set 3: Hardhat & Testing

### E3.1 — Write Full Test Suite
For the `DevToken` contract from Day 5, write tests for these cases that DON'T appear in the provided tests:
1. Transfer to address(0) should revert
2. Transfer of 0 tokens should still emit an event (behavior varies — test it!)
3. Multiple consecutive mints sum correctly
4. Owner can renounce ownership

### E3.2 — Fork Mainnet Challenge
```javascript
// Fork Ethereum mainnet and impersonate a USDC whale
// Transfer USDC to a test address and verify the balance

const { impersonateAccount } = require("@nomicfoundation/hardhat-network-helpers")

const USDC_WHALE = "0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503"
const USDC = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"

it("should transfer USDC from whale", async function() {
  // Impersonate and transfer
  // ...
})
```

### E3.3 — Gas Reporter Analysis
Run `REPORT_GAS=true npx hardhat test` on your DevToken.
Answer:
1. What is the gas cost of `transfer()`?
2. What is the gas cost of `approve()` followed by `transferFrom()`?
3. Which function is most expensive and why?

---

## Bonus Challenge: Fix the Bug

```solidity
// This token has a critical bug — find it and explain why it's dangerous
contract BuggyToken {
    mapping(address => uint256) public balances;
    uint256 public totalSupply;
    
    constructor(uint256 _supply) {
        balances[msg.sender] = _supply;
        totalSupply = _supply;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        balances[msg.sender] -= amount;
        balances[to] += amount;
        return true;
    }
    
    function transferBatch(address[] calldata recipients, uint256 amount) external {
        for (uint256 i = 0; i < recipients.length; i++) {
            balances[msg.sender] -= amount;  // can this go below zero?
            balances[recipients[i]] += amount;
        }
    }
}
```

**Hint:** What happens if `balances[msg.sender]` is 100 and you call `transferBatch([a, b, c, d], 30)`?
