# Week 4 Exercises — Advanced & Security

## Exercise Set 11: Security Challenges

### E11.1 — Ethernaut Levels
Complete these Ethernaut challenges in order:
1. **Level 0** — Hello Ethernaut (warm up)
2. **Level 1** — Fallback (receive/fallback functions)
3. **Level 4** — Telephone (tx.origin vs msg.sender)
4. **Level 6** — Delegation (delegatecall)
5. **Level 7** — Force (forcefully send ETH to a contract)
6. **Level 8** — Vault (reading private variables)
7. **Level 10** — Reentrancy (the classic attack)
8. **Level 11** — Elevator (interface tricks)
9. **Level 15** — Naught Coin (ERC-20 approve/transferFrom bypass)

URL: https://ethernaut.openzeppelin.com/

### E11.2 — Audit This Contract
Find ALL vulnerabilities in this contract (there are 8):
```solidity
pragma solidity ^0.7.0;  // V1

contract VulnerableVault {
    address owner;
    mapping(address => uint256) deposits;
    bool locked = false;
    
    constructor() {
        owner == msg.sender;  // V2
    }
    
    function deposit() external payable {
        deposits[msg.sender] = deposits[msg.sender] + msg.value;
        // V3: no event emitted
    }
    
    function withdraw(uint256 amount) external {
        require(deposits[msg.sender] >= amount);
        msg.sender.call{value: amount}("");  // V4: reentrancy
        deposits[msg.sender] -= amount;
    }
    
    function adminWithdraw() external {
        require(tx.origin == owner);  // V5: tx.origin
        msg.sender.transfer(address(this).balance);  // V6: sends to msg.sender not owner
    }
    
    function setOwner(address _owner) external {  // V7: no access control
        owner = _owner;
    }
    
    function getBalance() external view returns (uint256) {
        return deposits[msg.sender] + 1;  // V8: wrong balance calculation
    }
}
```

### E11.3 — Flash Loan Attack Simulation
On a mainnet fork:
1. Find a protocol that uses spot price for valuation
2. Simulate a flash loan price manipulation attack
3. Calculate the maximum profit
4. Explain how to fix the vulnerability

---

## Exercise Set 12: Advanced Patterns

### E12.1 — Implement TWAP Oracle
Add a TWAP oracle to the MiniAMM from Week 2:
```solidity
// Add to MiniAMM:
uint256 public price0CumulativeLast;
uint256 public price1CumulativeLast;
uint32 public blockTimestampLast;

function _update(uint256 balance0, uint256 balance1, uint112 _reserve0, uint112 _reserve1) private {
    uint32 blockTimestamp = uint32(block.timestamp % 2**32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast;
    
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
        price0CumulativeLast += uint256(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
        price1CumulativeLast += uint256(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }
    
    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
    blockTimestampLast = blockTimestamp;
}

// Compute TWAP over a window
function consult(address token, uint256 amountIn, uint32 windowSize) external view returns (uint256) {
    // ...
}
```

### E12.2 — Chainlink Integration
Build a contract that:
- Accepts ETH and stores a USD value (using Chainlink ETH/USD price)
- Allows withdrawal at the original USD value (regardless of ETH price changes)
- Uses staleness check on the price feed

### E12.3 — VRF Lottery
Extend the VRFRaffle contract to:
- Support multiple prize tiers (1st: 50%, 2nd: 30%, 3rd: 20%)
- Use multiple random words (`numWords = 3`)
- Handle edge cases: fewer than 3 players

---

## Capstone Code Review

### Self-Review Checklist for PixelSwap
Read through your own code with fresh eyes:

**Security:**
- [ ] Every external call follows CEI (Checks-Effects-Interactions)
- [ ] All user inputs validated
- [ ] Reentrancy guard on state-changing functions
- [ ] No tx.origin usage
- [ ] Slippage protection on swaps
- [ ] Deadline parameter on time-sensitive functions

**Gas:**
- [ ] Storage variables packed where possible
- [ ] Use `calldata` instead of `memory` for external function arrays
- [ ] Use `unchecked` where overflow is impossible
- [ ] Cache storage reads in local variables inside loops
- [ ] Use custom errors instead of string reverts

**Code Quality:**
- [ ] NatSpec comments on all public/external functions
- [ ] Events emitted for all state changes
- [ ] No magic numbers (use named constants)
- [ ] Consistent naming conventions

**Testing:**
- [ ] 100% function coverage
- [ ] Edge cases tested (zero amounts, max amounts, re-entrancy attempts)
- [ ] Fuzz tests for mathematical functions
- [ ] Integration test covering full user journey
