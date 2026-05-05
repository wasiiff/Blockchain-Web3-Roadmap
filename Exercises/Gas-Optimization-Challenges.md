# Gas Optimization Challenges

## Challenge 1: Storage Packing

Reorder these variables to minimize storage slots used:
```solidity
// Current: uses 4 slots
contract Unoptimized {
    uint256 a;          // 32 bytes → slot 0
    bool b;             // 1 byte  → slot 1 (wasted 31 bytes)
    uint256 c;          // 32 bytes → slot 2
    address d;          // 20 bytes → slot 3 (wasted 12 bytes)
    uint8 e;            // 1 byte  → slot 4 (wasted 31 bytes)
}

// Target: use 2 slots
contract Optimized {
    // Reorder to pack small types together
    // Your solution here
}
```

**Answer:** `uint256 a` → `uint256 c` → `bool b + address d + uint8 e` (packed in slot 2)

## Challenge 2: Loop Optimization

```solidity
// Gas: ~50,000 per 10 iterations. Target: ~10,000
contract LoopHog {
    address[] public stakers;
    mapping(address => uint256) public stakes;
    
    function totalStaked() external view returns (uint256 total) {
        for (uint256 i = 0; i < stakers.length; i++) {
            total += stakes[stakers[i]];  // Find 3 gas wastes here
        }
    }
}
```

**Optimizations:**
1. Cache `stakers.length` in a local variable (saves 1 SLOAD per iteration)
2. Use `unchecked { ++i; }` (saves ~30 gas per iteration in Solidity < 0.8.24 optimizer)
3. Read from `stakes[stakers[i]]` to `stakers[i]` — cache locally if used multiple times

## Challenge 3: Custom Errors vs Require Strings

Benchmark the gas difference:
```solidity
// Method A: String require (~3000 gas for the string)
function A(uint256 x) external pure {
    require(x > 0, "Value must be positive");
}

// Method B: Custom error (~250 gas)
error ValueMustBePositive();
function B(uint256 x) external pure {
    if (x == 0) revert ValueMustBePositive();
}

// Write a Hardhat test that measures and logs gas for both
```

## Challenge 4: Calldata vs Memory

```solidity
contract DataLocation {
    // Which is cheaper? Test and explain why.
    
    function processMemory(uint256[] memory data) external pure returns (uint256) {
        uint256 sum;
        for (uint256 i; i < data.length; ++i) sum += data[i];
        return sum;
    }
    
    function processCalldata(uint256[] calldata data) external pure returns (uint256) {
        uint256 sum;
        for (uint256 i; i < data.length; ++i) sum += data[i];
        return sum;
    }
    // calldata is cheaper for external functions — no copy is made!
}
```

## Challenge 5: Immutable vs Constant vs Storage

```solidity
contract CostComparison {
    // Constant: value known at compile time, embedded in bytecode
    uint256 public constant CONSTANT_VALUE = 100;       // FREE to read
    
    // Immutable: set in constructor, embedded in bytecode
    uint256 public immutable IMMUTABLE_VALUE;           // FREE to read
    
    // Storage: costs 2100 gas to read (SLOAD)
    uint256 public storageValue = 100;                  // COSTS 2100 gas
    
    constructor(uint256 _immutable) {
        IMMUTABLE_VALUE = _immutable;
    }
    
    // Which reads are free and which cost gas?
    function test() external view returns (uint256, uint256, uint256) {
        return (CONSTANT_VALUE, IMMUTABLE_VALUE, storageValue);
    }
}
```

**Challenge:** Identify all places in your Week 1-3 contracts where you stored something that should be `constant` or `immutable`.

## Challenge 6: Packing Structs in Mappings

```solidity
// Unpacked: 3 storage slots per entry
struct Unpacked {
    uint256 amount;    // slot 0
    uint256 timestamp; // slot 1
    address user;      // slot 2
    bool active;       // slot 3 (wastes 31 bytes!)
}

// Packed: 2 storage slots per entry
struct Packed {
    uint256 amount;    // slot 0
    address user;      // slot 1 (20 bytes)
    uint64 timestamp;  // slot 1 (8 bytes) — year 2554 safe!
    bool active;       // slot 1 (1 byte)
                       // slot 1: 29 bytes used, 3 bytes padding
}
```

**Exercise:** Calculate gas savings when storing 10,000 entries using Packed vs Unpacked struct.

## Gas Optimization Quick Reference

| Trick | Savings |
|-------|---------|
| Cache storage in local var | 2100 gas per repeated read |
| `unchecked { ++i; }` in loops | ~25 gas per iteration |
| Custom errors vs string revert | ~2750 gas on revert |
| `calldata` vs `memory` params | ~300 gas per 32 bytes |
| `constant`/`immutable` vs storage | 2100 gas per read |
| Pack structs/variables | 2900+ gas per SSTORE saved |
| Short-circuit evaluation | Variable (fail fast) |
| Avoid initializing to zero | ~20 gas per variable |
| `uint256` vs `uint8` in memory | uint256 is cheaper! |
| Use assembly for tight loops | 10-50% savings |
