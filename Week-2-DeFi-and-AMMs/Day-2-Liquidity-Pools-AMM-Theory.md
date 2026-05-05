# Day 9 (Week 2, Day 2) — Liquidity Pools & AMM Theory

> **Time:** 6–8 hours  
> **Goal:** Master the mathematics of automated market makers — how prices are set, the impact of trades, and why liquidity providers face impermanent loss.

---

## Part 1: What Problem Do AMMs Solve? (30 min)

### Traditional Order Books (CEX model)

In centralized exchanges (Coinbase, Binance):
- Buyers post "bid" orders: "I'll buy 1 ETH at $3,000"
- Sellers post "ask" orders: "I'll sell 1 ETH at $3,050"
- When bid meets ask → trade executes
- Market makers provide liquidity by constantly posting both sides

**Problems for DeFi:**
1. Order books require an off-chain server → not decentralized
2. Liquidity is fragmented and thin for small tokens
3. Market makers need capital on both sides and active management

**AMM Solution:**
Replace order books with a mathematical formula. Liquidity is pooled into a smart contract. Anyone can provide liquidity. Prices are determined algorithmically, not by orders.

---

## Part 2: The x·y=k Constant Product Formula (2 hours)

### 2.1 The Core Formula

```
x · y = k

Where:
x = reserve of Token A
y = reserve of Token B
k = constant (invariant)
```

The key insight: **k never changes** (except when liquidity is added/removed). Every trade must maintain this product.

### 2.2 Visual Intuition

```
                    y
                    │
                    │ x·y=k (hyperbola)
          y=200     ●
                    │  \
          y=100     │    ●  (current state)
                    │       \
           y=50     │          ●
                    │             \
                    └──────────────────── x
                   x=100  x=200  x=400
```

The price point moves **along the curve**. You can never reach x=0 or y=0 — there's always some liquidity.

### 2.3 Calculating a Swap

**Setup:** ETH/USDC pool with:
- x = 100 ETH
- y = 300,000 USDC
- k = 100 × 300,000 = 30,000,000

**Current price:** 300,000 / 100 = 3,000 USDC per ETH

**Question:** Alice wants to buy 10 ETH. How much USDC does she pay?

```
After the trade:
x_new = 100 - 10 = 90 ETH

We need: x_new · y_new = k
90 · y_new = 30,000,000
y_new = 30,000,000 / 90 = 333,333.33 USDC

Alice paid: y_new - y_old = 333,333.33 - 300,000 = 33,333.33 USDC

Effective price: 33,333.33 / 10 = 3,333.33 USDC per ETH
Market price was: 3,000 USDC per ETH
Price impact: (3,333.33 - 3,000) / 3,000 = 11.1%
```

**This is price impact** — larger trades move the price more.

### 2.4 The Swap Formula (Derived)

Given: want to buy `dx` of token X (giving token Y)
```
x · y = (x - dx) · (y + dy)
x · y = x·y - dx·y + dy·x - dx·dy

Solve for dy (amount of Y you must give):
dy = (y · dx) / (x - dx)
```

In practice, Uniswap adds a 0.3% fee:
```
dy = (y · dx · 997) / (x · 1000 - dx · 997)
```
(The 0.3% fee = 3/1000, applied as 997/1000 factor)

### 2.5 Slippage vs Price Impact

**Price impact:** How much your trade moves the price (calculated from the pool state).
**Slippage:** The difference between expected price and actual execution price (due to other txs in the same block changing the pool state before yours).

```javascript
// Slippage tolerance example
const expectedOutput = 33333n
const slippageTolerance = 0.005 // 0.5%
const minOutput = expectedOutput * (1000n - 5n) / 1000n // 99.5% of expected

// If actual output < minOutput → tx reverts (slippage protection)
```

---

## Part 3: Impermanent Loss — The LP Provider's Risk (2 hours)

### 3.1 What is Impermanent Loss?

Impermanent loss (IL) is the difference between:
- What you'd have if you just **held** both tokens
- What you actually have **inside the pool**

It's called "impermanent" because if prices return to the original ratio, the loss disappears.

### 3.2 Calculating Impermanent Loss

**Setup:** Alice provides liquidity to ETH/USDC pool:
- Deposits: 1 ETH + 3,000 USDC (at ETH price = $3,000)
- Pool state: 100 ETH, 300,000 USDC, k = 30,000,000
- Alice's share: 1% of pool

**Scenario: ETH price doubles to $6,000**

Since ETH is now worth more, arbitrageurs will buy ETH from the pool until the pool price matches the market price:

```
New pool state:
x · y = k = 30,000,000
Price = y/x = 6,000

x² = k / price = 30,000,000 / 6,000 = 5,000
x = √5,000 = 70.71 ETH

y = 6,000 × 70.71 = 424,264 USDC
```

Alice's 1% share is now:
- 0.7071 ETH = 0.7071 × $6,000 = $4,242.64
- 4,242.64 USDC
- Total = $8,485.28

If Alice had just **held** (1 ETH + 3,000 USDC):
- 1 ETH = $6,000
- 3,000 USDC = $3,000
- Total = $9,000

**Impermanent Loss = ($9,000 - $8,485.28) / $9,000 = 5.72%**

### 3.3 IL Formula

```
IL = 2√(price_ratio) / (1 + price_ratio) - 1

Where price_ratio = new_price / initial_price
```

| Price Change | Impermanent Loss |
|-------------|-----------------|
| +25% | -0.6% |
| +50% | -2.0% |
| +100% (2x) | -5.7% |
| +200% (3x) | -13.4% |
| +400% (5x) | -25.5% |
| +900% (10x) | -42.5% |
| -50% | -5.7% |
| -75% | -20.0% |

IL is symmetric — it doesn't matter if the price goes up or down by the same ratio.

### 3.4 When is Providing Liquidity Still Profitable?

LP providers earn trading fees. If fees > IL, providing liquidity is profitable.

```
Profitability = Trading Fees Earned - Impermanent Loss

For Uniswap V2 (0.3% fee, high-volume pool):
- USDC/ETH earns significant fees even with high IL
- Correlated pairs (USDC/USDT, stablecoin pairs) have near-zero IL
- Volatile pairs (ETH/meme coin) risk large IL

Breakeven calculation:
Required volume = IL / fee_rate
IL = 5.7% for 2x price change
Fee rate = 0.3%
Required volume = pool_size × (0.057 / 0.003) = pool_size × 19
Need to turn over the pool 19x to break even
```

### 3.5 Why Uniswap V3 Changed This

Uniswap V3 introduced **concentrated liquidity**: instead of providing liquidity across the entire price range (0 to ∞), LPs choose a **price range** (e.g., ETH between $2,500 and $3,500).

Benefits:
- 100x-4000x more capital efficient (same capital, more fees)
- Less IL if price stays in range

Risk:
- If price moves outside your range → 0 fees, fully exposed to one token

---

## Part 4: LP Tokens (30 min)

When you add liquidity, you receive LP tokens representing your pool share:

```
LP_tokens_minted = totalSupply × (amount_deposited / total_reserve)

Example:
Pool has: 100 ETH, 300,000 USDC, 10,000 LP tokens
Alice deposits: 10 ETH + 30,000 USDC (1% of pool)
Alice receives: 10,000 × 1% = 100 LP tokens

When Alice removes liquidity:
Burn 100 LP tokens → receive 1% of current pool
```

**LP token value always increases** from fees (fees are added to pool reserves, not distributed separately).

---

## Part 5: Hands-On Math Exercises (2 hours)

### Exercise 1: Manual AMM Calculations

Pool: 50 WBTC + 1,500,000 USDC

1. What is the current BTC price?
2. Alice buys 5 WBTC. How much USDC does she pay? What is the effective price?
3. What is the price impact?
4. After Alice's trade, Bob buys 5 more WBTC. Why does Bob pay more than Alice?

### Exercise 2: Implement the AMM Formula in JavaScript

```javascript
class AMM {
  constructor(reserveA, reserveB) {
    this.reserveA = BigInt(reserveA)
    this.reserveB = BigInt(reserveB)
    this.k = this.reserveA * this.reserveB
  }
  
  // Calculate how much B you get for giving `amountA` of A
  // With 0.3% fee
  getAmountOut(amountA) {
    const amountAWithFee = BigInt(amountA) * 997n
    const numerator = amountAWithFee * this.reserveB
    const denominator = this.reserveA * 1000n + amountAWithFee
    return numerator / denominator
  }
  
  // Execute a swap
  swap(amountA) {
    const amountB = this.getAmountOut(amountA)
    const priceImpact = Number(amountB * 10000n / (this.reserveB)) / 100
    
    this.reserveA += BigInt(amountA)
    this.reserveB -= amountB
    
    return { amountB, priceImpact }
  }
  
  getPrice() {
    return Number(this.reserveB) / Number(this.reserveA)
  }
}

// Test it
const pool = new AMM(100e18, 300000e6)  // 100 ETH, 300k USDC
console.log("ETH price:", pool.getPrice(), "USDC")

const swap1 = pool.swap(10e18) // buy 10 ETH worth of USDC
console.log("USDC paid:", swap1.amountB.toString())
console.log("Price impact:", swap1.priceImpact, "%")
console.log("New ETH price:", pool.getPrice(), "USDC")
```

### Exercise 3: Calculate Your Impermanent Loss

Write a function that given:
- Initial deposit amounts
- Current prices
Returns: current LP value, hold value, and impermanent loss percentage

```javascript
function calculateIL(
  initialEthAmount,
  initialUsdcAmount,
  currentEthPrice  // in USDC
) {
  const initialEthPrice = initialUsdcAmount / initialEthAmount
  const priceRatio = currentEthPrice / initialEthPrice
  
  // New pool ETH amount (after arbitrage)
  const newEthAmount = initialEthAmount / Math.sqrt(priceRatio)
  const newUsdcAmount = initialUsdcAmount * Math.sqrt(priceRatio)
  
  const lpValue = newEthAmount * currentEthPrice + newUsdcAmount
  const holdValue = initialEthAmount * currentEthPrice + initialUsdcAmount
  
  const il = (lpValue - holdValue) / holdValue * 100
  
  return {
    lpValue: lpValue.toFixed(2),
    holdValue: holdValue.toFixed(2),
    impermanentLoss: il.toFixed(2) + "%",
    lpEth: newEthAmount.toFixed(4),
    lpUsdc: newUsdcAmount.toFixed(2)
  }
}

// Test: deposited 1 ETH + 3000 USDC, ETH doubles to $6000
console.log(calculateIL(1, 3000, 6000))
```

---

## Resources for Day 9

| Resource | Link | Type |
|----------|------|------|
| Uniswap V2 Whitepaper | https://uniswap.org/whitepaper.pdf | Paper |
| IL Calculator | https://dailydefi.org/tools/impermanent-loss-calculator/ | Tool |
| Uniswap V3 Whitepaper | https://uniswap.org/whitepaper-v3.pdf | Paper |
| Curve Finance Whitepaper | https://curve.fi/files/stableswap-paper.pdf | Paper |
| AMM Deep Dive (Paradigm) | https://research.paradigm.xyz/uniswaps-ux | Article |
| Uniswap V2 Explained | https://jeiwan.net/posts/programming-defi-uniswap-1/ | Tutorial |

---

## Tomorrow

[Day 10 → Uniswap Deep Dive](./Day-3-Uniswap-Deep-Dive.md)
