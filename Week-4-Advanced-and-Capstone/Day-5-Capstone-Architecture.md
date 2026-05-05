# Day 26 (Week 4, Day 5) — Capstone: Architecture Design

> **Time:** 6–8 hours  
> **Goal:** Design your capstone project architecture in full detail before writing code.

---

## Capstone Project: Mini Uniswap V2 Clone

**Name:** PixelSwap — A minimal DEX with factory, pairs, and router

**Why this?** It combines everything from Weeks 1-4:
- Solidity fundamentals (Week 1)
- AMM mechanics (Week 2)
- ERC-20 tokens (Week 1/2)
- Frontend with Wagmi (Week 2)
- Security patterns (Week 4)
- Optional: Chainlink price feeds (Week 4)

---

## Architecture Diagram

```
PixelSwap Architecture
═══════════════════════════════════════════════════════════════

  User (MetaMask)
       │
       │ calls
       ▼
┌─────────────────────────────────────────────────────────────┐
│                    PixelSwapRouter                          │
│  - addLiquidity()                                           │
│  - removeLiquidity()                                        │
│  - swapExactTokensForTokens()                               │
│  - swapExactETHForTokens()                                  │
│  - swapTokensForExactTokens()                               │
│  - Handles WETH wrapping/unwrapping                         │
│  - Multi-hop routing: A→B→C                                 │
└────────────────────┬────────────────────────────────────────┘
                     │ calls
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    PixelSwapFactory                         │
│  - createPair(tokenA, tokenB) → PixelSwapPair               │
│  - allPairs[] (list of all pairs)                           │
│  - getPair(tokenA, tokenB) → pair address                   │
│  - Deploys pair contracts using CREATE2 (deterministic)     │
└────────────────────┬────────────────────────────────────────┘
                     │ creates
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  PixelSwapPair (×N)                         │
│  - ERC-20 LP token (inherits from)                          │
│  - reserve0, reserve1 (token balances)                      │
│  - swap()                                                   │
│  - mint() (add liquidity)                                   │
│  - burn() (remove liquidity)                                │
│  - price0CumulativeLast, price1CumulativeLast (TWAP oracle) │
└─────────────────────────────────────────────────────────────┘

Frontend (Next.js + Wagmi)
  ├── /swap           — swap tokens
  ├── /pool           — manage liquidity positions
  ├── /pool/add       — add liquidity
  └── /analytics      — pool stats (volume, TVL, APY)
```

---

## Contract Design

### PixelSwapFactory

```
Storage:
  mapping(address => mapping(address => address)) getPair  // tokenA → tokenB → pair
  address[] allPairs

Functions:
  createPair(address tokenA, address tokenB) → address pair
    - Validate: tokenA != tokenB, neither is address(0)
    - Sort tokens: token0 < token1 (by address, for determinism)
    - Deploy via CREATE2 (deterministic address = no need to store in some cases)
    - Store in mapping (both directions)
    - Emit PairCreated

  allPairsLength() → uint256
```

### PixelSwapPair (extends ERC-20)

```
Storage:
  address token0, token1
  uint112 reserve0, reserve1
  uint32 blockTimestampLast
  uint256 price0CumulativeLast   (for TWAP)
  uint256 price1CumulativeLast
  uint256 kLast                  (for fee calculation, optional)
  
  uint256 constant MINIMUM_LIQUIDITY = 1000

Functions:
  mint(address to) → uint256 liquidity
    - Calculate LP tokens to mint
    - If first: sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY
    - If subsequent: min(amount0/reserve0, amount1/reserve1) * totalSupply
    - Update reserves
    - Emit Mint

  burn(address to) → (uint256 amount0, uint256 amount1)
    - Calculate tokens to return proportionally
    - Burn LP tokens
    - Transfer tokens
    - Update reserves
    - Emit Burn

  swap(uint256 amount0Out, uint256 amount1Out, address to, bytes data)
    - Optimistic transfer
    - Call flash swap callback if data.length > 0
    - Verify k invariant with 0.3% fee
    - Update reserves
    - Emit Swap

  sync()   — force reserves to match actual balances
  skim(address to) — take excess tokens
  _update() — internal, updates reserves + TWAP
  getReserves() → (uint112, uint112, uint32)
```

### PixelSwapRouter

```
Storage:
  address factory
  address WETH

Key algorithms:
  getAmountsOut(uint256 amountIn, address[] path) → uint256[] amounts
    - Traverses path: A→B→C
    - Calls getAmountOut() for each hop

  addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin, to, deadline)
    - Create pair if doesn't exist
    - Calculate optimal amounts (maintain current ratio)
    - Transfer tokens to pair
    - Call pair.mint()

  swapExactTokensForTokens(amountIn, amountOutMin, path[], to, deadline)
    - Calculate amounts for each hop
    - Verify amountOut >= amountOutMin
    - Execute swaps hop by hop
    - Final token delivered to `to`
```

---

## Database / Indexing Design (off-chain)

For a real DEX, you need to index events for analytics:

```
Events to index:
  Factory: PairCreated(token0, token1, pair, pairIndex)
  Pair: Swap(sender, amount0In, amount1In, amount0Out, amount1Out, to)
  Pair: Mint(sender, amount0, amount1)
  Pair: Burn(sender, amount0, amount1, to)

Store in:
  PostgreSQL / Supabase — for structured queries
  OR
  The Graph — decentralized indexing protocol

Useful queries:
  - Total volume (24h, 7d, all-time)
  - TVL per pool
  - Fees earned by LPs
  - Price history (from swap events)
  - User's liquidity positions
```

---

## Security Checklist (Design Phase)

Before writing a line of code, answer these:

- [ ] **Reentrancy:** All state changes happen BEFORE external calls (CEI)
- [ ] **Access control:** Who can call each function? Document it.
- [ ] **Integer overflow:** Using 0.8.24, unchecked only where provably safe
- [ ] **Oracle:** Not using spot prices; TWAP from accumulated price feeds
- [ ] **Flash loans:** k invariant checked with fees prevents borrowing without repaying
- [ ] **Front-running:** Deadline parameter, minAmountOut parameter
- [ ] **MINIMUM_LIQUIDITY:** Burned to prevent share price manipulation
- [ ] **Token sorting:** token0 < token1 always for deterministic pair addresses
- [ ] **Approve before transfer:** Router must be approved before addLiquidity
- [ ] **Fee-on-transfer tokens:** Not supported (document this limitation)

---

## Frontend Pages Design

### /swap
```
┌─────────────────────────────────┐
│ SWAP                            │
│                                 │
│ You pay                         │
│ [1.0    ] [ETH ▼]               │
│           Balance: 2.5 ETH      │
│                                 │
│     ⇅ (reverse swap)            │
│                                 │
│ You receive                     │
│ [2987.23] [USDC ▼]              │
│                                 │
│ Price impact: 0.12%             │
│ Route: ETH → USDC               │
│                                 │
│ [Swap]                          │
└─────────────────────────────────┘
```

### /pool
```
Your Positions:
┌─────────────────────────────────┐
│ ETH/USDC    0.3% fee            │
│ $1,245.30   ↑ 2.3% fees (7d)   │
│ [Manage]                        │
└─────────────────────────────────┘

Top Pools by TVL:
[Table: Pool, TVL, Volume 24h, Fees 24h, APY]
```

---

## Tomorrow

[Day 27 → Capstone Implementation](./Day-6-Capstone-Implementation.md)
