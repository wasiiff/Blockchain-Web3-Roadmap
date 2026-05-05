# PixelSwap Architecture

## System Overview

```
                        ┌─────────────────────────────────────┐
                        │         Frontend (Next.js)          │
                        │                                     │
                        │  /swap    /pool    /analytics       │
                        │                                     │
                        │  Wagmi v2 + Viem + RainbowKit       │
                        └──────────────┬──────────────────────┘
                                       │ reads/writes
                                       ▼
                        ┌─────────────────────────────────────┐
                        │         PixelSwapRouter             │
                        │                                     │
                        │  addLiquidity()                     │
                        │  removeLiquidity()                  │
                        │  swapExactTokensForTokens()         │
                        │  swapExactETHForTokens()            │
                        │  getAmountsOut()                    │
                        └──────────────┬──────────────────────┘
                                       │ calls
                  ┌────────────────────┼─────────────────────┐
                  │                    │                     │
                  ▼                    ▼                     ▼
     ┌────────────────────┐ ┌──────────────────┐ ┌──────────────────┐
     │  PixelSwapFactory  │ │ PixelSwapPair A/B│ │ PixelSwapPair B/C│
     │                    │ │                  │ │                  │
     │  createPair()      │ │  swap()          │ │  swap()          │
     │  allPairs[]        │ │  mint()          │ │  mint()          │
     │  getPair mapping   │ │  burn()          │ │  burn()          │
     │                    │ │  LP token (ERC20)│ │  LP token (ERC20)│
     └────────────────────┘ └──────────────────┘ └──────────────────┘
```

## Data Flow: Swap A → B

```
User clicks "Swap" on frontend
        │
        │  1. Frontend calls: router.getAmountsOut(amountIn, [A, B])
        │     Returns: expected output amount
        │
        │  2. Frontend shows: amountOut, price impact, min received
        │
        │  3. User confirms in MetaMask
        │
        │  4. Token A approve(router, amountIn)  [if not already approved]
        │
        │  5. router.swapExactTokensForTokens(amountIn, minOut, [A, B], to, deadline)
        │     │
        │     ├── IERC20(A).safeTransferFrom(user → pairAB, amountIn)
        │     │
        │     ├── pairAB.swap(0, amountOut, user, "")
        │     │   │
        │     │   ├── IERC20(B).transfer(user, amountOut)  ← optimistic
        │     │   │
        │     │   └── verify: balance0*balance1 × 1000000 ≥ reserve0*reserve1 × 997²
        │     │
        │     └── revert if amountOut < minOut
        │
        └── User receives Token B
```

## Data Flow: Add Liquidity

```
User specifies amounts and clicks "Add"
        │
        │  1. router.addLiquidity(A, B, amountADesired, amountBDesired, ...)
        │     │
        │     ├── Create pair if needed: factory.createPair(A, B)
        │     │
        │     ├── Calculate optimal amounts (maintain current ratio):
        │     │   If first deposit: use amountADesired, amountBDesired
        │     │   Otherwise: quote(amountA, reserveA, reserveB) = amountB optimal
        │     │
        │     ├── Transfer A → pair, B → pair
        │     │
        │     └── pair.mint(to)
        │         │
        │         ├── If first deposit: LP = sqrt(amountA * amountB) - MINIMUM_LIQUIDITY
        │         │   [burn 1000 LP to address(0)]
        │         │
        │         ├── Otherwise: LP = min(amountA/reserveA, amountB/reserveB) * totalSupply
        │         │
        │         └── Mint LP tokens to `to`
        │
        └── User receives LP tokens
```

## Smart Contract Inheritance

```
PixelSwapPair
    └── ERC20 (OpenZeppelin)
            └── IERC20
            └── ERC20Permit (optional)

PixelSwapFactory
    └── Ownable (OpenZeppelin)

PixelSwapRouter
    └── (no inheritance, pure logic)
```

## Key Invariants

```
1. K invariant (per pair):
   After every swap: balance0 * balance1 ≥ reserve0 * reserve1
   [accounting for the 0.3% fee: (balance0 × 1000 - amount0In × 3) × (balance1 × 1000 - amount1In × 3) ≥ reserve0 × reserve1 × 1,000,000]

2. LP token value:
   LP_token_value = (reserve0 / totalSupply, reserve1 / totalSupply)
   [always monotonically increasing as fees accrue]

3. Token ordering:
   For every pair: token0 < token1 (by address)
   [ensures deterministic pair addresses via CREATE2]

4. Minimum liquidity:
   MINIMUM_LIQUIDITY (1000) always locked in pair
   [prevents share price manipulation attacks]
```

## Storage Layout (PixelSwapPair)

```
Slot 0: factory address (20 bytes) | (12 bytes unused)
Slot 1: token0 address (20 bytes) | (12 bytes unused)
Slot 2: token1 address (20 bytes) | (12 bytes unused)
Slot 3: reserve0 (14 bytes, uint112) | reserve1 (14 bytes, uint112) | blockTimestampLast (4 bytes, uint32)
Slot 4: price0CumulativeLast (32 bytes, uint256)
Slot 5: price1CumulativeLast (32 bytes, uint256)
Slot 6: kLast (32 bytes, uint256)
Slot 7+: ERC20 state (balances, allowances, totalSupply)
```

## Gas Estimates (Approximate)

| Operation | Gas |
|-----------|-----|
| Create pair | ~2,500,000 |
| Add liquidity (first) | ~180,000 |
| Add liquidity (subsequent) | ~140,000 |
| Remove liquidity | ~130,000 |
| Swap (single hop) | ~120,000 |
| Swap ETH → Token | ~130,000 |
| Swap (2-hop) | ~200,000 |
