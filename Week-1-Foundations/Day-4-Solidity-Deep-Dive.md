# Day 4 — Solidity In-Depth

> **Time:** 6–8 hours  
> **Goal:** Write production-quality Solidity — not tutorial code. Understand storage layout, gas optimization, access control, events, and common patterns.

---

## Part 1: Solidity Core Concepts (3 hours)

### 1.1 Types and Storage Layout

Understanding **storage layout** is critical for gas optimization and security.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract StorageLayout {
    // Each state variable occupies storage slots (32 bytes each)
    // Slot 0:
    uint256 public a;        // 32 bytes → fills slot 0
    
    // Slot 1:
    uint128 public b;        // 16 bytes ┐
    uint128 public c;        // 16 bytes ┘ packed into slot 1!
    
    // Slot 2:
    uint64 public d;         // 8 bytes  ┐
    uint64 public e;         // 8 bytes  │ packed into slot 2
    uint128 public f;        // 16 bytes ┘
    
    // Slot 3:
    bool public flag1;       // 1 byte   ┐
    bool public flag2;       // 1 byte   │ packed
    address public owner;    // 20 bytes ┘ packed into slot 3
    
    // Slot 4, 5, 6... (mappings use keccak slot derivation)
    mapping(address => uint256) public balances;
    
    // Dynamic array: length at slot N, elements at keccak256(N) + index
    uint256[] public values;
}
```

**Variable packing saves gas:**
```solidity
// EXPENSIVE: 3 separate storage slots
contract Unpacked {
    uint256 a; // slot 0
    uint256 b; // slot 1  
    uint256 c; // slot 2
}

// CHEAP: 1 storage slot
contract Packed {
    uint128 a; // ┐ slot 0
    uint64 b;  // │
    uint64 c;  // ┘
}
```

---

### 1.2 Value Types vs Reference Types

```solidity
contract TypesBehavior {
    // VALUE TYPES: copied on assignment
    // uint, int, bool, address, bytes1-32, enums
    
    function valueTypeExample() external pure {
        uint256 x = 10;
        uint256 y = x;  // y is a COPY of x
        y = 20;         // x is still 10
    }
    
    // REFERENCE TYPES: structs, arrays, mappings
    // Must specify data location: storage, memory, calldata
    
    struct User {
        address addr;
        uint256 balance;
        string name;
    }
    
    mapping(address => User) public users;
    
    // calldata: read-only, cheapest for external function args
    function createUser(string calldata name) external {
        users[msg.sender] = User({
            addr: msg.sender,
            balance: 0,
            name: name
        });
    }
    
    // memory: temporary, for internal processing
    function processUser(address addr) external view returns (string memory) {
        User memory user = users[addr]; // copy from storage to memory
        return user.name;
    }
    
    // storage pointer: directly references storage (no copy)
    function incrementBalance(address addr) external {
        User storage user = users[addr]; // pointer, not copy
        user.balance += 1;              // directly modifies storage
    }
}
```

---

### 1.3 Functions In-Depth

```solidity
contract Functions {
    uint256 private _value;
    
    // Visibility
    // public: callable externally and internally
    // external: only from outside (cheaper for external calls)
    // internal: only within contract and derived contracts
    // private: only within this exact contract
    
    // State mutability
    // pure: doesn't read or write state
    // view: reads state, doesn't write
    // payable: can receive ETH
    // (none): reads and writes state
    
    function setValue(uint256 val) external {
        _value = val;
    }
    
    function getValue() external view returns (uint256) {
        return _value;
    }
    
    function double(uint256 x) external pure returns (uint256) {
        return x * 2;
    }
    
    // Receive ETH
    receive() external payable {
        // called when ETH is sent with no data
    }
    
    fallback() external payable {
        // called when no function matches, or ETH with data
    }
    
    // Function overloading
    function transfer(address to, uint256 amount) external returns (bool) { ... }
    function transfer(address to, uint256 amount, bytes calldata data) external returns (bool) { ... }
}
```

---

### 1.4 Events — The Blockchain's Logging System

Events are crucial for:
- Frontend apps listening to contract activity
- Off-chain indexing (The Graph)
- Cheap storage (events are in logs, not contract storage)

```solidity
contract EventDemo {
    // Indexed params can be searched/filtered (max 3 indexed params)
    event Transfer(
        address indexed from,
        address indexed to,
        uint256 amount     // not indexed — full value stored
    );
    
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
    
    // Anonymous event (no topic for event signature, slightly cheaper)
    event Log(bytes32 indexed data) anonymous;
    
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        balances[to] += amount;
        
        emit Transfer(msg.sender, to, amount); // emit keyword
    }
}
```

**Listening to events with ethers.js:**
```javascript
const contract = new ethers.Contract(address, abi, provider)

// Listen to future events
contract.on("Transfer", (from, to, amount, event) => {
  console.log(`Transfer: ${from} → ${to}: ${ethers.formatEther(amount)} ETH`)
  console.log("Block:", event.blockNumber)
  console.log("Tx hash:", event.transactionHash)
})

// Query past events
const filter = contract.filters.Transfer("0xFromAddress")
const events = await contract.queryFilter(filter, -1000) // last 1000 blocks
events.forEach(e => console.log(e.args))
```

---

### 1.5 Access Control Patterns

```solidity
// Pattern 1: Simple owner
contract Ownable {
    address public owner;
    
    error NotOwner();
    error ZeroAddress();
    
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        if (msg.sender != owner) revert NotOwner();
        _;
    }
    
    function transferOwnership(address newOwner) external onlyOwner {
        if (newOwner == address(0)) revert ZeroAddress();
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
}

// Pattern 2: Role-based access (OpenZeppelin AccessControl)
import "@openzeppelin/contracts/access/AccessControl.sol";

contract RoleBasedContract is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
    }
    
    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        // only MINTER_ROLE addresses can call this
    }
    
    function burn(address from, uint256 amount) external onlyRole(BURNER_ROLE) {
        // only BURNER_ROLE addresses can call this
    }
}
```

---

### 1.6 Error Handling — Modern Solidity

```solidity
contract ErrorHandling {
    // Custom errors (cheaper than string reverts — saves gas!)
    error InsufficientBalance(uint256 available, uint256 required);
    error Unauthorized(address caller);
    error InvalidAmount();
    
    mapping(address => uint256) public balances;
    
    function withdraw(uint256 amount) external {
        uint256 available = balances[msg.sender];
        
        // Custom error: cheapest option
        if (amount == 0) revert InvalidAmount();
        if (available < amount) revert InsufficientBalance(available, amount);
        
        balances[msg.sender] -= amount;
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed"); // require with string (legacy but still used)
    }
    
    // assert: for invariants that should NEVER be false
    // If assert fails → all gas consumed (use sparingly)
    function criticalOperation() external {
        uint256 before = address(this).balance;
        // ... do stuff ...
        uint256 after = address(this).balance;
        assert(after >= before); // should NEVER decrease in this function
    }
}
```

**Gas comparison:**
```
require("string message") → ~3000 gas for the string
revert CustomError()      → ~250 gas
assert(condition)         → all gas consumed on failure
```

---

### 1.7 Modifiers — Reusable Guards

```solidity
contract ModifierPatterns {
    bool private _locked;
    bool public paused;
    address public owner;
    
    // Reentrancy guard — CRITICAL for DeFi
    modifier nonReentrant() {
        require(!_locked, "ReentrancyGuard: reentrant call");
        _locked = true;
        _;
        _locked = false;
    }
    
    // Pause mechanism
    modifier whenNotPaused() {
        require(!paused, "Paused");
        _;
    }
    
    // Ownership
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // Parameter validation
    modifier validAddress(address addr) {
        require(addr != address(0), "Zero address");
        _;
    }
    
    // Composable modifiers
    function criticalWithdraw(address to, uint256 amount) 
        external 
        nonReentrant    // first: lock
        whenNotPaused   // second: check pause
        onlyOwner       // third: check ownership
        validAddress(to) // fourth: validate param
    {
        // safe to execute
    }
}
```

---

### 1.8 Inheritance and Interfaces

```solidity
// Interface defines what, not how
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

// Abstract contract: partial implementation
abstract contract ERC20Base is IERC20 {
    mapping(address => uint256) internal _balances;
    uint256 internal _totalSupply;
    
    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }
    
    // transfer left as abstract (must implement in derived)
    function transfer(address to, uint256 amount) external virtual override returns (bool);
}

// Concrete implementation
contract MyToken is ERC20Base {
    string public name = "MyToken";
    string public symbol = "MTK";
    uint8 public decimals = 18;
    
    constructor(uint256 initialSupply) {
        _totalSupply = initialSupply;
        _balances[msg.sender] = initialSupply;
    }
    
    function transfer(address to, uint256 amount) external override returns (bool) {
        require(_balances[msg.sender] >= amount, "Insufficient");
        _balances[msg.sender] -= amount;
        _balances[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    // Must implement remaining interface methods...
}
```

---

## Part 2: Hands-On Coding (3 hours)

### Build a Production Vault Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/// @title Token Vault — deposit ERC-20 tokens and earn shares
contract TokenVault is ReentrancyGuard, Ownable, Pausable {
    using SafeERC20 for IERC20;
    
    IERC20 public immutable token;
    
    uint256 public totalShares;
    mapping(address => uint256) public shares;
    
    uint256 public constant MINIMUM_LIQUIDITY = 1000; // prevent share price manipulation
    
    error ZeroAmount();
    error ZeroShares();
    error InsufficientShares(uint256 have, uint256 need);
    
    event Deposited(address indexed user, uint256 amount, uint256 shares);
    event Withdrawn(address indexed user, uint256 shares, uint256 amount);
    
    constructor(address _token) Ownable(msg.sender) {
        token = IERC20(_token);
    }
    
    /// @notice Returns total tokens held in vault
    function totalAssets() public view returns (uint256) {
        return token.balanceOf(address(this));
    }
    
    /// @notice Convert token amount to shares
    function previewDeposit(uint256 assets) public view returns (uint256) {
        uint256 supply = totalShares;
        if (supply == 0) return assets;
        return (assets * supply) / totalAssets();
    }
    
    /// @notice Convert shares to token amount
    function previewRedeem(uint256 sharesToRedeem) public view returns (uint256) {
        uint256 supply = totalShares;
        if (supply == 0) return 0;
        return (sharesToRedeem * totalAssets()) / supply;
    }
    
    /// @notice Deposit tokens, receive shares
    function deposit(uint256 amount) external nonReentrant whenNotPaused {
        if (amount == 0) revert ZeroAmount();
        
        uint256 sharesToMint = previewDeposit(amount);
        if (sharesToMint == 0) revert ZeroShares();
        
        token.safeTransferFrom(msg.sender, address(this), amount);
        shares[msg.sender] += sharesToMint;
        totalShares += sharesToMint;
        
        emit Deposited(msg.sender, amount, sharesToMint);
    }
    
    /// @notice Burn shares, receive tokens
    function withdraw(uint256 sharesToBurn) external nonReentrant whenNotPaused {
        if (sharesToBurn == 0) revert ZeroAmount();
        if (shares[msg.sender] < sharesToBurn) {
            revert InsufficientShares(shares[msg.sender], sharesToBurn);
        }
        
        uint256 amount = previewRedeem(sharesToBurn);
        if (amount == 0) revert ZeroAmount();
        
        shares[msg.sender] -= sharesToBurn;
        totalShares -= sharesToBurn;
        
        token.safeTransfer(msg.sender, amount);
        
        emit Withdrawn(msg.sender, sharesToBurn, amount);
    }
    
    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
}
```

**Write tests for this contract using Hardhat:**
```javascript
const { expect } = require("chai")
const { ethers } = require("hardhat")

describe("TokenVault", function() {
  let vault, token, owner, user1, user2
  
  beforeEach(async function() {
    [owner, user1, user2] = await ethers.getSigners()
    
    // Deploy mock ERC20
    const Token = await ethers.getContractFactory("MockERC20")
    token = await Token.deploy("Mock Token", "MTK", ethers.parseEther("1000000"))
    
    // Deploy vault
    const Vault = await ethers.getContractFactory("TokenVault")
    vault = await Vault.deploy(await token.getAddress())
    
    // Give users tokens
    await token.transfer(user1.address, ethers.parseEther("1000"))
    await token.transfer(user2.address, ethers.parseEther("1000"))
  })
  
  it("should deposit and give correct shares", async function() {
    const depositAmount = ethers.parseEther("100")
    
    await token.connect(user1).approve(await vault.getAddress(), depositAmount)
    await vault.connect(user1).deposit(depositAmount)
    
    expect(await vault.shares(user1.address)).to.equal(depositAmount)
    expect(await vault.totalShares()).to.equal(depositAmount)
    expect(await vault.totalAssets()).to.equal(depositAmount)
  })
  
  it("should redeem shares for correct token amount", async function() {
    const depositAmount = ethers.parseEther("100")
    await token.connect(user1).approve(await vault.getAddress(), depositAmount)
    await vault.connect(user1).deposit(depositAmount)
    
    const sharesBefore = await vault.shares(user1.address)
    const balanceBefore = await token.balanceOf(user1.address)
    
    await vault.connect(user1).withdraw(sharesBefore)
    
    const balanceAfter = await token.balanceOf(user1.address)
    expect(balanceAfter - balanceBefore).to.equal(depositAmount)
  })
  
  it("should revert when withdrawing more shares than owned", async function() {
    await expect(
      vault.connect(user1).withdraw(ethers.parseEther("100"))
    ).to.be.revertedWithCustomError(vault, "InsufficientShares")
  })
})
```

---

## Part 3: Exercises (1.5 hours)

### Challenge 1: Find the Storage Layout Bug

```solidity
// This contract has a storage collision bug.
// Can you find it and fix it?

contract BuggyStorage {
    bool public initialized;   // slot 0: 1 byte
    uint256 public value;      // slot 1: 32 bytes
    uint8 public flag;         // slot 0: 1 byte (packed with initialized)
    address public admin;      // ... which slot?
    
    // Question: What is the slot of each variable?
    // Question: Is there actually a bug here, or is it just inefficient packing?
}
```

### Challenge 2: Gas Optimization

The following function costs 150,000 gas. Make it cost < 30,000 gas without changing functionality:

```solidity
contract GasWaster {
    address[] public users;
    mapping(address => uint256) public points;
    
    function addPointsToAll(uint256 amount) external {
        for (uint256 i = 0; i < users.length; i++) {
            address user = users[i];
            uint256 current = points[user]; // SLOAD each iteration
            points[user] = current + amount; // SSTORE each iteration
        }
    }
}
```

### Challenge 3: Implement Missing Functions

Complete the ERC-20 approve/transferFrom pattern (allowance mechanism):

```solidity
contract MyERC20 {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    // TODO: implement approve()
    // TODO: implement transferFrom()
    // These are the two functions you must write
}
```

---

## Resources for Day 4

| Resource | Link | Type |
|----------|------|------|
| Solidity Docs | https://docs.soliditylang.org/ | Docs |
| OpenZeppelin Contracts | https://docs.openzeppelin.com/contracts/ | Docs |
| Solidity by Example | https://solidity-by-example.org/ | Tutorial |
| EVM Storage Layout | https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html | Docs |
| Hardhat Docs | https://hardhat.org/docs | Docs |
| Solidity Patterns | https://fravoll.github.io/solidity-patterns/ | Reference |
| Smart Contract Security | https://consensys.github.io/smart-contract-best-practices/ | Guide |

---

## Tomorrow

[Day 5 → Hardhat Setup + First Deployment](./Day-5-Hardhat-Setup-Deployment.md)
