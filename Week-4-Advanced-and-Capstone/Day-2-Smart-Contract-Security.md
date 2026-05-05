# Day 23 (Week 4, Day 2) — Smart Contract Security

> **Time:** 6–8 hours  
> **Goal:** Know every major smart contract vulnerability — how each works, real exploit examples, and how to prevent them.

---

## Part 1: The Most Critical Vulnerabilities

### 1.1 Reentrancy — The DAO Hack

**What it is:** A contract makes an external call before updating its state. The called contract calls back into the original function.

**Real incident:** The DAO hack (2016) — $60M stolen, caused the Ethereum/Ethereum Classic split.

**The vulnerable pattern:**
```solidity
// VULNERABLE: reentrancy attack possible
contract VulnerableBank {
    mapping(address => uint256) public balances;
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        
        // 1. Send ETH FIRST
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        // 2. Update state AFTER (TOO LATE!)
        balances[msg.sender] -= amount;
        
        // The called contract can re-enter withdraw() before this line!
    }
}
```

**The attack:**
```solidity
contract ReentrancyAttacker {
    VulnerableBank target;
    
    constructor(address _target) {
        target = VulnerableBank(_target);
    }
    
    function attack() external payable {
        target.deposit{value: 1 ether}();
        target.withdraw(1 ether); // trigger the recursive drain
    }
    
    // Called every time target sends ETH to this contract
    receive() external payable {
        if (address(target).balance >= 1 ether) {
            target.withdraw(1 ether); // re-enter before balance is updated!
        }
    }
}
```

**The fix:**
```solidity
// FIX 1: Checks-Effects-Interactions (CEI) pattern
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient");
    
    balances[msg.sender] -= amount; // EFFECT first
    
    (bool success,) = msg.sender.call{value: amount}(""); // INTERACTION last
    require(success, "Transfer failed");
}

// FIX 2: Reentrancy guard (OpenZeppelin)
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract SecureBank is ReentrancyGuard {
    function withdraw(uint256 amount) external nonReentrant {
        // ...
    }
}
```

**Types of reentrancy:**
- Single-function: reenter the same function
- Cross-function: reenter a different function in the same contract
- Read-only reentrancy: attacker reads state mid-execution (even view functions can be exploited if they're called in checks)

---

### 1.2 Integer Overflow/Underflow

**Pre-Solidity 0.8:** Integers could silently overflow.

```solidity
// VULNERABLE (Solidity < 0.8 without SafeMath)
uint8 max = 255;
max += 1; // = 0 (overflow, wraps around!)

uint8 min = 0;
min -= 1; // = 255 (underflow, wraps around!)
```

**Post-Solidity 0.8:** Overflow/underflow automatically reverts.

**Remaining issue:** `unchecked` blocks bypass the protection:
```solidity
// Intentional unchecked (gas saving) — ONLY use when mathematically impossible to overflow
function incrementCounter() external {
    unchecked {
        counter++; // saves ~100 gas, OK if counter < 2^256 is guaranteed
    }
}
```

---

### 1.3 Access Control Failures

```solidity
// VULNERABLE: missing access control
contract AccessBug {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // MISSING onlyOwner! Anyone can call this
    function transferOwnership(address newOwner) external {
        owner = newOwner;
    }
    
    // VULNERABLE: wrong check
    function withdraw() external {
        require(tx.origin == owner, "Not owner"); // tx.origin is easily spoofed via phishing
        // Use msg.sender instead!
    }
}
```

**tx.origin vs msg.sender:**
```
Alice → calls → PhishingContract → calls → VictimContract

In VictimContract:
  msg.sender = PhishingContract (the immediate caller)
  tx.origin  = Alice (the original EOA)

NEVER use tx.origin for authentication! 
PhishingContract tricks Alice into calling it, then calls VictimContract.
VictimContract checks tx.origin == Alice → passes → funds drained.
```

---

### 1.4 Front-Running

Front-running: an attacker sees your pending transaction in the mempool and submits their own transaction with higher gas to execute before yours.

**Sandwich Attack (most common in DeFi):**
```
1. Alice submits: swap 1000 USDC for ETH, 1% slippage
2. Bot detects Alice's pending tx
3. Bot submits: buy ETH (same pool, higher gas) → price goes up
4. Alice's tx executes at worse price (within her slippage tolerance)
5. Bot submits: sell ETH → profits from the artificial price movement
```

**Real cost:** Billions of dollars extracted from DeFi users annually via MEV.

**Mitigations:**
```solidity
// 1. Use Flashbots / private mempool (submit tx directly to validators)
// 2. Use commit-reveal for sensitive operations
// 3. Use reasonable slippage tolerance (not too high)
// 4. Use deadline parameter

function swap(
    uint256 amountIn,
    uint256 minAmountOut,  // minimum acceptable output
    uint256 deadline       // transaction must execute by this time
) external {
    require(block.timestamp <= deadline, "Expired");
    // ...
    require(amountOut >= minAmountOut, "Too much slippage");
}
```

---

### 1.5 Oracle Manipulation / Price Manipulation

```solidity
// VULNERABLE: using a spot price from a DEX
function liquidate(address user) external {
    // Get price from Uniswap spot price (manipulable!)
    uint256 price = uniswapPool.slot0().sqrtPriceX96; // SPOT PRICE
    
    // Attacker can manipulate the pool price in the same tx using flash loan
    // then call liquidate() while price is manipulated
}
```

**The fix: Use TWAP (Time-Weighted Average Price)**
```solidity
// Use Uniswap V3 TWAP oracle (much harder to manipulate)
function getTWAP(uint32 twapInterval) external view returns (uint256 price) {
    (int56[] memory tickCumulatives,) = pool.observe([twapInterval, 0]);
    int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
    int24 arithmeticMeanTick = int24(tickCumulativesDelta / int56(uint56(twapInterval)));
    price = TickMath.getSqrtRatioAtTick(arithmeticMeanTick);
}

// Or: Use Chainlink price feeds (preferred for production)
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
```

---

### 1.6 Flash Loan Attacks

Flash loans enable borrowing massive amounts (often $10M+) within a single transaction. This amplifies every other vulnerability.

**Attack pattern:**
```
Flash loan: borrow $100M USDC
  → Manipulate Uniswap pool price
  → Exploit price-sensitive protocol
  → Repay flash loan + fee
Net profit without any capital!
```

**Famous flash loan hacks:**
- bZx hack (Feb 2020): $1M via oracle manipulation
- Harvest Finance (Oct 2020): $33.8M
- Cream Finance (Oct 2021): $130M
- Many others in 2022-2023

**Defense:**
- Use TWAPs instead of spot prices
- Use Chainlink oracles (external to the DEX ecosystem)
- Add checks for same-block manipulations

---

### 1.7 Unchecked Return Values

```solidity
// VULNERABLE: ignoring return value
token.transfer(recipient, amount); // if it returns false, you proceed anyway

// SAFE: use SafeERC20
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
using SafeERC20 for IERC20;
IERC20(token).safeTransfer(recipient, amount); // reverts on false return
```

---

### 1.8 Signature Replay

```solidity
// VULNERABLE: signatures can be replayed across chains or multiple times
function execute(bytes calldata signature, address target, uint256 amount) external {
    bytes32 hash = keccak256(abi.encode(target, amount));
    address signer = hash.recover(signature);
    require(signer == owner, "Not owner");
    // Execute...
    // Problem: this signature can be reused infinite times!
}

// SAFE: include nonce + chainId
function execute(bytes calldata signature, address target, uint256 amount, uint256 nonce) external {
    require(!usedNonces[nonce], "Nonce used");
    usedNonces[nonce] = true;
    
    bytes32 hash = keccak256(abi.encode(
        block.chainid,  // prevents cross-chain replay
        target, 
        amount, 
        nonce           // prevents replay on same chain
    ));
    address signer = ECDSA.recover(hash, signature);
    require(signer == owner, "Not owner");
}
```

---

## Part 2: Security Tools (1 hour)

### Slither (Static Analysis)

```bash
# Install
pip install slither-analyzer

# Run on your contracts
slither contracts/

# Check for specific detectors
slither contracts/ --detect reentrancy-eth
slither contracts/ --detect uninitialized-state

# Print summary
slither contracts/ --print human-summary
```

### Mythril (Symbolic Execution)

```bash
# Install
pip install mythril

# Analyze a specific contract
myth analyze contracts/MyContract.sol --solc-version 0.8.24

# Will find: reentrancy, integer overflows, unchecked calls, etc.
```

### Echidna (Fuzzing)

```solidity
// Write invariant tests that echidna fuzzes
contract MyContractFuzzer {
    MyContract target;
    
    constructor() {
        target = new MyContract();
    }
    
    // Invariant: totalSupply should never exceed MAX_SUPPLY
    function echidna_supply_invariant() public view returns (bool) {
        return target.totalSupply() <= target.MAX_SUPPLY();
    }
    
    // Invariant: balance of zero address should always be 0
    function echidna_zero_address_balance() public view returns (bool) {
        return target.balanceOf(address(0)) == 0;
    }
}
```

---

## Part 3: Vulnerable Contract — Fix All Bugs

Find and fix all vulnerabilities in this contract:

```solidity
// BUGGY: Can you find all 7 vulnerabilities?
pragma solidity ^0.7.0; // Problem 1: old version, no overflow protection

contract BuggyVault {
    mapping(address => uint256) balances;
    address owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount);
        msg.sender.call{value: amount}(""); // Problem 2: reentrancy
        balances[msg.sender] -= amount;     // Problem 3: unchecked return, state after call
    }
    
    function adminWithdraw(uint256 amount) external {
        require(tx.origin == owner); // Problem 4: tx.origin
        payable(msg.sender).transfer(amount); // Problem 5: who is msg.sender here?
    }
    
    function setOwner(address newOwner) external {
        owner = newOwner; // Problem 6: no access control!
    }
    
    function getPrice() external view returns (uint256) {
        // Problem 7: returns manipulable spot price
        return IUniswap(dexAddress).getSpotPrice(tokenA, tokenB);
    }
}
```

---

## Resources for Day 23

| Resource | Link | Type |
|----------|------|------|
| Secureum Security 101 | https://secureum.substack.com/ | Course |
| SWC Registry | https://swcregistry.io/ | Reference |
| Rekt News | https://rekt.news | Incident Reports |
| OpenZeppelin Security Blog | https://blog.openzeppelin.com/ | Articles |
| Smart Contract Security Checklist | https://github.com/transmissions11/solcurity | Checklist |
| DeFiHackLabs | https://github.com/SunWeb3Sec/DeFiHackLabs | Exploit Replays |
| Foundry Fuzz Testing | https://book.getfoundry.sh/forge/fuzz-testing | Docs |

---

## Tomorrow

[Day 24 → MEV & Flash Loans](./Day-3-MEV-Flashloans.md)
