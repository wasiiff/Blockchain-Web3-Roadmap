# Week 2 Exercises — DeFi & AMMs

## Exercise Set 4: AMM Mathematics

### E4.1 — Manual Pool Calculations
Given a pool with: 500 ETH, 1,500,000 USDC

1. What is the current price of ETH in USDC?
2. Alice buys 50 ETH — how much USDC does she pay? What is her effective price?
3. What is the price impact of Alice's trade?
4. After Alice's trade, what is the new price?
5. Bob immediately sells 50 ETH back — does he get more or less than 1,500,000 USDC?

### E4.2 — Impermanent Loss Calculation
Alice provides liquidity:
- Deposits: 10 ETH + 30,000 USDC (ETH price = $3,000)

Calculate her impermanent loss if:
1. ETH goes to $6,000 (2x)
2. ETH goes to $12,000 (4x)
3. ETH drops to $1,500 (0.5x)

Formula: `IL = 2√(r) / (1 + r) - 1` where `r = new_price / initial_price`

### E4.3 — Implement AMM in JavaScript
Build a complete AMM simulator with:
- Add liquidity
- Remove liquidity
- Swap A→B and B→A
- LP token tracking
- Price impact calculation

```javascript
class AMM {
  constructor(tokenA, tokenB) { /* ... */ }
  addLiquidity(amountA, amountB) { /* returns LP tokens */ }
  removeLiquidity(lpTokens) { /* returns amountA, amountB */ }
  swap(tokenIn, amountIn) { /* returns amountOut */ }
  getPrice(token) { /* returns price */ }
}
```

---

## Exercise Set 5: DeFi Protocol Interaction

### E5.1 — Aave Integration (Mainnet Fork)
```javascript
// Write a test that:
// 1. Supplies 10,000 USDC to Aave V3
// 2. Borrows 3,000 USDC worth of ETH
// 3. Checks health factor (should be > 1.5)
// 4. Transfers some ETH out to lower health factor
// 5. Checks health factor is now below 1.5

const AAVE_POOL = "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2"
// ... implement the test
```

### E5.2 — Curve Pool Analysis
Query the 3pool (USDC/USDT/DAI) on mainnet fork:
- What are the current balances of each token?
- What is the current virtual price?
- Swap 1,000,000 USDC → USDT — what do you get? What's the slippage?

---

## Exercise Set 6: Debugging Challenges

### E6.1 — Find the Bug
```solidity
// This AMM has an error. What is it?
contract BuggyAMM {
    uint256 public reserve0 = 1000e18;
    uint256 public reserve1 = 3000e18;
    
    function swap(uint256 amountIn, bool zeroForOne) external {
        uint256 k = reserve0 * reserve1;
        
        if (zeroForOne) {
            uint256 newReserve0 = reserve0 + amountIn;
            uint256 newReserve1 = k / newReserve0;
            uint256 amountOut = reserve1 - newReserve1;
            
            // Missing: fee deduction!
            // Missing: check amountOut > 0
            // Missing: transfer tokens!
            
            reserve0 = newReserve0;
            reserve1 = newReserve1;
        }
    }
}
```

### E6.2 — Gas Optimization Challenge
Optimize this staking contract — target: reduce gas by 40%:
```solidity
contract GasHeavyStaking {
    mapping(address => uint256) public stakedAmount;
    mapping(address => uint256) public stakingTimestamp;
    mapping(address => uint256) public rewardDebt;
    address[] public stakers;  // Problem: O(n) iteration
    
    function stake(uint256 amount) external {
        if (stakedAmount[msg.sender] == 0) {
            stakers.push(msg.sender);  // Problem: unbounded array
        }
        stakedAmount[msg.sender] += amount;
        stakingTimestamp[msg.sender] = block.timestamp;
    }
    
    function calculateRewards(address user) external view returns (uint256) {
        uint256 timeStaked = block.timestamp - stakingTimestamp[user];  // 2 SLOADs
        uint256 staked = stakedAmount[user];                            // 1 SLOAD
        return staked * timeStaked / 365 days;
    }
}
```

---

## Exercise Set 7: Frontend Challenges

### E7.1 — Price Quote Widget
Build a React component that:
- Takes tokenIn, tokenOut, amount as inputs
- Queries your MiniAMM for the output amount
- Shows: output amount, price impact, minimum received (with 0.5% slippage)
- Updates in real-time as user types (debounced)
- Shows loading state during query

### E7.2 — Event History
Display the last 10 Swap events from your AMM:
```typescript
// Using ethers.js:
const filter = amm.filters.Swap()
const events = await amm.queryFilter(filter, -1000)
// Display each event: sender, tokens in/out, price
```
