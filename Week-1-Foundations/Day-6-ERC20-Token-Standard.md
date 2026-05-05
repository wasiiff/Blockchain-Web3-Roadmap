# Day 6 — ERC-20 Token Standard Deep Dive

> **Time:** 6–8 hours  
> **Goal:** Fully understand ERC-20 — every function, the allowance model, extensions, and common pitfalls. Build a production ERC-20 from scratch.

---

## Part 1: ERC-20 Standard Explained (2 hours)

### 1.1 What is ERC-20?

ERC-20 (Ethereum Request for Comments #20) is the **token standard** that defines a common interface for fungible tokens on Ethereum. "Fungible" means every unit is interchangeable — 1 USDC = 1 USDC.

**Why a standard matters:** Without ERC-20, every token would have different function names. DEXs, wallets, and DeFi apps would need to write custom code for every token. ERC-20 means any app can interact with any token using the same interface.

**Tokens built on ERC-20:**
- USDC, USDT, DAI (stablecoins)
- WETH (wrapped ETH)
- UNI, AAVE, COMP (governance tokens)
- LINK (Chainlink oracle token)
- Literally thousands of others

---

### 1.2 The Full ERC-20 Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title ERC-20 Interface — every function explained
interface IERC20 {
    
    // ─── View Functions ───────────────────────────────────────────────
    
    /// @notice Total tokens in existence
    function totalSupply() external view returns (uint256);
    
    /// @notice Token balance of an address
    function balanceOf(address account) external view returns (uint256);
    
    /// @notice How many tokens `spender` is allowed to spend on behalf of `owner`
    /// This is the core of the "approve → transferFrom" pattern
    function allowance(address owner, address spender) external view returns (uint256);
    
    // ─── Mutating Functions ───────────────────────────────────────────
    
    /// @notice Transfer tokens from msg.sender to `to`
    /// @return bool true on success (ERC-20 requires bool return)
    function transfer(address to, uint256 amount) external returns (bool);
    
    /// @notice Approve `spender` to spend up to `amount` from msg.sender's balance
    /// DANGER: Known race condition — see section 1.4
    function approve(address spender, uint256 amount) external returns (bool);
    
    /// @notice Transfer from `from` to `to`, spending allowance
    /// Used by DeFi protocols (Uniswap, Aave, etc.)
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    
    // ─── Events ──────────────────────────────────────────────────────
    
    /// @notice Emitted when tokens move
    event Transfer(address indexed from, address indexed to, uint256 value);
    
    /// @notice Emitted when allowance is set
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

---

### 1.3 The Allowance Model — How DeFi Actually Works

The `approve → transferFrom` pattern is how every DeFi protocol interacts with your tokens:

```
Step 1: You call approve(UniswapRouter, 1000 USDC)
         → UniswapRouter now has permission to spend up to 1000 USDC from your wallet

Step 2: You call UniswapRouter.swapExactTokensForETH(...)
         → Uniswap internally calls USDC.transferFrom(yourAddress, pool, amount)
         → Your USDC moves without you signing a second transaction

Your wallet            USDC Contract              Uniswap Router
    │                       │                          │
    │  approve(router, 1000)│                          │
    │──────────────────────>│                          │
    │                       │  allowance[you][router]=1000
    │                       │                          │
    │  swap(usdc→eth)       │                          │
    │─────────────────────────────────────────────────>│
    │                       │  transferFrom(you, pool, 500)
    │                       │<─────────────────────────│
    │                       │  allowance[you][router]=500
    │                       │                          │
```

**Infinite approval:** Many apps ask you to approve `type(uint256).max` — meaning unlimited. This is convenient but risky: if the contract is hacked, it can drain your entire token balance.

---

### 1.4 The Approve Race Condition (and How to Fix It)

**The bug:**
```
Initial state: approve(spender, 100)
1. Owner submits: approve(spender, 50)  ← wants to change from 100 to 50
2. Spender sees pending tx, front-runs: transferFrom(owner, spender, 100)  ← uses old allowance
3. Owner's tx confirms: approve(spender, 50)
4. Spender calls: transferFrom(owner, spender, 50)  ← uses new allowance
Result: Spender took 150 tokens when owner wanted to allow only 50
```

**Solutions:**
```solidity
// Option 1: Set to 0 first, then set new value
approve(spender, 0)
approve(spender, 50)

// Option 2: Use increaseAllowance / decreaseAllowance (OpenZeppelin adds these)
increaseAllowance(spender, 50)  // atomic increase
decreaseAllowance(spender, 100) // atomic decrease

// Option 3: Use ERC-20 Permit (EIP-2612) — gasless approvals via signature
// (covered below)
```

---

### 1.5 ERC-20 Extensions — What OpenZeppelin Adds

```
ERC20                    — base standard
  ├── ERC20Burnable      — token holders can burn (destroy) tokens
  ├── ERC20Capped        — enforce maximum supply cap
  ├── ERC20Mintable      — owner can mint new tokens
  ├── ERC20Pausable      — emergency pause all transfers
  ├── ERC20Permit        — EIP-2612 gasless approvals via signatures
  ├── ERC20Votes         — governance voting power (used in DAOs)
  ├── ERC20Snapshot      — take balance snapshots at specific blocks
  └── ERC20FlashMint     — EIP-3156 flash loan standard
```

---

### 1.6 ERC-20 Permit (EIP-2612) — The Modern Approval

**Problem:** The normal approve() requires a transaction (costs gas). Before swapping, users often need to approve the router → two transactions, two gas payments.

**EIP-2612 Permit:** Sign an approval off-chain (no gas), submit with the actual transaction.

```solidity
// Instead of:
token.approve(router, amount)         // Tx 1 (costs gas)
router.swapExactTokensForETH(...)     // Tx 2 (costs gas)

// With permit:
const signature = await user.signTypedData(domain, types, { owner, spender, value, nonce, deadline })
router.swapWithPermit(amount, deadline, v, r, s)  // Single tx: permits + swaps
```

**Implementing Permit:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract PermitToken is ERC20, ERC20Permit {
    constructor() ERC20("Permit Token", "PMIT") ERC20Permit("Permit Token") {
        _mint(msg.sender, 1_000_000 * 10**18);
    }
}
```

**Using Permit from frontend:**
```javascript
const { ethers } = require("ethers")

async function permitAndSwap(token, router, amount, deadline) {
  const signer = await provider.getSigner()
  const spender = await router.getAddress()
  const value = amount
  const nonce = await token.nonces(signer.address)
  
  // Build EIP-712 permit signature
  const domain = {
    name: await token.name(),
    version: "1",
    chainId: (await provider.getNetwork()).chainId,
    verifyingContract: await token.getAddress(),
  }
  
  const types = {
    Permit: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" },
      { name: "value", type: "uint256" },
      { name: "nonce", type: "uint256" },
      { name: "deadline", type: "uint256" },
    ],
  }
  
  const message = {
    owner: signer.address,
    spender: spender,
    value: value,
    nonce: nonce,
    deadline: deadline,
  }
  
  const signature = await signer.signTypedData(domain, types, message)
  const { v, r, s } = ethers.Signature.from(signature)
  
  // Now call permit + swap in one transaction
  await router.swapWithPermit(amount, deadline, v, r, s)
}
```

---

## Part 2: Build a Full ERC-20 From Scratch (2.5 hours)

### Build a Governance Token with Votes

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Nonces.sol";

/// @title GovernanceToken — Full-featured governance token for DAOs
contract GovernanceToken is ERC20, ERC20Burnable, ERC20Permit, ERC20Votes, Ownable {
    
    uint256 public constant MAX_SUPPLY = 100_000_000 * 10**18; // 100 million
    
    // Vesting
    struct VestingSchedule {
        uint256 total;
        uint256 released;
        uint256 start;
        uint256 duration;
        bool revoked;
    }
    mapping(address => VestingSchedule) public vestingSchedules;
    
    error MaxSupplyExceeded();
    error VestingAlreadyExists();
    error VestingNotStarted();
    error NothingToRelease();
    
    event VestingCreated(address indexed beneficiary, uint256 amount, uint256 duration);
    event TokensReleased(address indexed beneficiary, uint256 amount);
    
    constructor(address initialOwner) 
        ERC20("Governance Token", "GOV") 
        ERC20Permit("Governance Token")
        Ownable(initialOwner)
    {
        // Mint initial supply to deployer (team/treasury allocation)
        _mint(initialOwner, 10_000_000 * 10**18); // 10M initial
    }
    
    /// @notice Create a vesting schedule for team/investors
    function createVesting(
        address beneficiary,
        uint256 amount,
        uint256 startTimestamp,
        uint256 durationSeconds
    ) external onlyOwner {
        if (vestingSchedules[beneficiary].total != 0) revert VestingAlreadyExists();
        if (totalSupply() + amount > MAX_SUPPLY) revert MaxSupplyExceeded();
        
        vestingSchedules[beneficiary] = VestingSchedule({
            total: amount,
            released: 0,
            start: startTimestamp,
            duration: durationSeconds,
            revoked: false
        });
        
        // Mint tokens to this contract (held in escrow)
        _mint(address(this), amount);
        
        emit VestingCreated(beneficiary, amount, durationSeconds);
    }
    
    /// @notice Calculate vested amount
    function vestedAmount(address beneficiary) public view returns (uint256) {
        VestingSchedule memory schedule = vestingSchedules[beneficiary];
        if (schedule.total == 0 || block.timestamp < schedule.start) return 0;
        if (block.timestamp >= schedule.start + schedule.duration) return schedule.total;
        
        return (schedule.total * (block.timestamp - schedule.start)) / schedule.duration;
    }
    
    /// @notice Release vested tokens to beneficiary
    function releaseVested() external {
        VestingSchedule storage schedule = vestingSchedules[msg.sender];
        if (schedule.total == 0) revert VestingNotStarted();
        
        uint256 vested = vestedAmount(msg.sender);
        uint256 releasable = vested - schedule.released;
        
        if (releasable == 0) revert NothingToRelease();
        
        schedule.released += releasable;
        _transfer(address(this), msg.sender, releasable);
        
        emit TokensReleased(msg.sender, releasable);
    }
    
    /// @notice Owner can mint up to MAX_SUPPLY
    function mint(address to, uint256 amount) external onlyOwner {
        if (totalSupply() + amount > MAX_SUPPLY) revert MaxSupplyExceeded();
        _mint(to, amount);
    }
    
    // Required overrides for ERC20Votes
    function _update(address from, address to, uint256 amount)
        internal
        override(ERC20, ERC20Votes)
    {
        super._update(from, to, amount);
    }
    
    function nonces(address owner)
        public
        view
        override(ERC20Permit, Nonces)
        returns (uint256)
    {
        return super.nonces(owner);
    }
}
```

---

## Part 3: Common ERC-20 Pitfalls

### Pitfall 1: Fee-on-Transfer Tokens

Some tokens (like USDT on some chains, SafeMoon) take a fee on transfer:
```solidity
// If token takes 1% fee:
token.transferFrom(user, pool, 1000)
// Pool actually receives 990, not 1000!
// DeFi protocols check: balanceBefore/After to handle this
```

**Correct handling:**
```solidity
function depositFeeOnTransfer(address token, uint256 amount) external {
    uint256 balanceBefore = IERC20(token).balanceOf(address(this));
    IERC20(token).transferFrom(msg.sender, address(this), amount);
    uint256 balanceAfter = IERC20(token).balanceOf(address(this));
    uint256 actualReceived = balanceAfter - balanceBefore; // use this, not amount
}
```

### Pitfall 2: USDT's Non-Standard Approve

USDT doesn't return a bool from approve(). Using SafeERC20 from OpenZeppelin handles this:
```solidity
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

using SafeERC20 for IERC20;
IERC20(usdt).safeTransfer(to, amount);       // handles no-return tokens
IERC20(usdt).safeTransferFrom(from, to, amount);
IERC20(usdt).safeApprove(spender, amount);   // handles non-standard approve
IERC20(usdt).forceApprove(spender, amount);  // approves even if current is non-zero
```

### Pitfall 3: Rebase Tokens (stETH)

Rebase tokens change balances automatically (stETH accrues ETH staking rewards by increasing everyone's balance). DeFi protocols often use "shares" representation instead.

---

## Resources for Day 6

| Resource | Link | Type |
|----------|------|------|
| EIP-20 Original Spec | https://eips.ethereum.org/EIPS/eip-20 | Spec |
| EIP-2612 Permit | https://eips.ethereum.org/EIPS/eip-2612 | Spec |
| OpenZeppelin ERC20 | https://docs.openzeppelin.com/contracts/5.x/erc20 | Docs |
| SafeERC20 | https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#SafeERC20 | Docs |
| Token Bugs by Trail of Bits | https://github.com/crytic/building-secure-contracts | Guide |
| Weird ERC-20 Behaviors | https://github.com/d-xo/weird-erc20 | Reference |

---

## Tomorrow

[Day 7 → Mini Project: Launch Your Own Token](./Day-7-Mini-Project-Token.md)
