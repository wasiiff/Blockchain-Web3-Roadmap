# Day 10 (Week 2, Day 3) — Uniswap Deep Dive

> **Time:** 6–8 hours  
> **Goal:** Read and understand Uniswap V2 source code. Understand how swaps, liquidity, and flash swaps work at the contract level.

---

## Part 1: Uniswap V2 Architecture (2 hours)

### 1.1 Three-Contract Architecture

```
UniswapV2Factory
    │
    ├── createPair(tokenA, tokenB) → UniswapV2Pair
    │   Each pair is a separate contract
    │
    └── allPairs[]  (list of all deployed pairs)

UniswapV2Pair (e.g., ETH/USDC pool)
    ├── ERC20 (LP token — inherits from here)
    ├── reserve0, reserve1 (token balances)
    ├── swap()
    ├── mint() (add liquidity)
    ├── burn() (remove liquidity)
    └── sync() / skim()

UniswapV2Router02 (user-facing)
    ├── addLiquidity()
    ├── removeLiquidity()
    ├── swapExactTokensForTokens()
    ├── swapTokensForExactTokens()
    ├── swapExactETHForTokens()
    └── ... (many swap variants)
```

**Why two layers?**
- The Pair contract is minimal and gas-efficient (core logic only)
- The Router handles user-friendly operations, deadline checks, slippage protection
- You interact with the Router, not the Pair directly

---

### 1.2 Reading the UniswapV2Pair Source

The core swap function — simplified and annotated:

```solidity
// From: https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol

function swap(
    uint amount0Out,  // how much token0 the caller wants to receive
    uint amount1Out,  // how much token1 the caller wants to receive
    address to,       // recipient of the output tokens
    bytes calldata data  // if non-empty → flash swap callback
) external lock {
    
    require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
    (uint112 _reserve0, uint112 _reserve1,) = getReserves();
    require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');
    
    // Send output tokens to `to` FIRST (optimistic transfer)
    if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out);
    if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out);
    
    // If flash swap: call receiver contract (must return tokens + fee)
    if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
    
    // Check what we actually received (balance - reserve)
    uint balance0 = IERC20(_token0).balanceOf(address(this));
    uint balance1 = IERC20(_token1).balanceOf(address(this));
    
    uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
    uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
    
    require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
    
    // Verify k invariant with fee (0.3% fee → multiply balances by 997/1000)
    {
        uint balance0Adjusted = balance0 * 1000 - amount0In * 3;
        uint balance1Adjusted = balance1 * 1000 - amount1In * 3;
        
        require(
            balance0Adjusted * balance1Adjusted >= uint(_reserve0) * _reserve1 * 1000**2,
            'UniswapV2: K'
        );
    }
    
    // Update reserves and emit event
    _update(balance0, balance1, _reserve0, _reserve1);
    emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
}
```

**The "optimistic transfer" pattern:**
1. Send tokens out FIRST
2. Then verify the math works
3. If it doesn't → revert (but the EVM reverts ALL changes including the transfer)

This is what enables flash swaps.

---

### 1.3 How Flash Swaps Work

Flash swaps let you borrow tokens from a Uniswap pool, use them, and return them (+ fee) — all in one transaction. If you don't return them, the transaction reverts.

```
Your Contract                  Uniswap Pair
     │                              │
     │  swap(0, 1000 USDC, yourAddr, "flashdata")
     │─────────────────────────────>│
     │                              │ sends 1000 USDC to your contract
     │<─────────────────────────────│
     │                              │
     │  [execute your logic here]   │
     │  (arbitrage, liquidation...) │
     │                              │
     │  uniswapV2Call(sender, 0, 1000, data) callback
     │<─────────────────────────────│
     │                              │
     │  USDC.transfer(pair, 1003)   │ return 1000 + 0.3% fee
     │─────────────────────────────>│
     │                              │
     │<─────────────────────────────│ swap() verifies K, succeeds
```

**Implementing a flash swap receiver:**
```solidity
contract FlashSwapExample {
    address constant UNISWAP_V2_FACTORY = 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;
    
    function startFlashSwap(address tokenBorrow, uint256 amount) external {
        // Find the pair containing tokenBorrow
        address weth = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
        address pair = IUniswapV2Factory(UNISWAP_V2_FACTORY).getPair(tokenBorrow, weth);
        
        // Encode what we want to borrow
        bytes memory data = abi.encode(tokenBorrow, amount);
        
        // Determine which token is token0
        address token0 = IUniswapV2Pair(pair).token0();
        uint256 amount0 = token0 == tokenBorrow ? amount : 0;
        uint256 amount1 = token0 == tokenBorrow ? 0 : amount;
        
        // Trigger the flash swap
        IUniswapV2Pair(pair).swap(amount0, amount1, address(this), data);
    }
    
    // Called by Uniswap after sending us the tokens
    function uniswapV2Call(
        address sender,
        uint256 amount0,
        uint256 amount1,
        bytes calldata data
    ) external {
        (address tokenBorrow, uint256 amount) = abi.decode(data, (address, uint256));
        
        // ─── Your logic here ────────────────────────────────────────
        // You have `amount` of `tokenBorrow` to use
        // e.g., arbitrage between two DEXs
        // e.g., liquidate a position on Aave
        // ────────────────────────────────────────────────────────────
        
        // Repay: amount + 0.3% fee
        uint256 fee = (amount * 3) / 997 + 1;
        uint256 repayAmount = amount + fee;
        
        IERC20(tokenBorrow).transfer(msg.sender, repayAmount); // msg.sender = pair
    }
}
```

---

### 1.4 Adding & Removing Liquidity

**Adding liquidity:**
```solidity
// Router calls this on the Pair
function mint(address to) external lock returns (uint liquidity) {
    (uint112 _reserve0, uint112 _reserve1,) = getReserves();
    uint balance0 = IERC20(_token0).balanceOf(address(this));
    uint balance1 = IERC20(_token1).balanceOf(address(this));
    
    uint amount0 = balance0 - _reserve0;  // tokens already deposited
    uint amount1 = balance1 - _reserve1;
    
    uint _totalSupply = totalSupply;
    if (_totalSupply == 0) {
        // First liquidity: LP tokens = sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY
        // MINIMUM_LIQUIDITY (1000) permanently locked to prevent manipulation
        liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
        _mint(address(0), MINIMUM_LIQUIDITY); // lock forever
    } else {
        // Subsequent: proportional to existing reserves
        liquidity = Math.min(
            amount0 * _totalSupply / _reserve0,
            amount1 * _totalSupply / _reserve1
        );
    }
    
    require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
    _mint(to, liquidity);
    
    _update(balance0, balance1, _reserve0, _reserve1);
}
```

**The MINIMUM_LIQUIDITY trick:** The first 1000 LP tokens are permanently burned to address(0). This prevents "vampire attacks" where someone provides a tiny amount of liquidity, manipulates the LP token price, and drains the pool.

---

### 1.5 Uniswap V3 Improvements

Uniswap V3 (2021) introduced major changes:

| Feature | V2 | V3 |
|---------|----|----|
| Liquidity | Across entire range 0→∞ | Concentrated in range [a, b] |
| Capital efficiency | Baseline | Up to 4000x more |
| LP positions | Fungible ERC-20 | Non-fungible ERC-721 |
| Fee tiers | 0.3% only | 0.01%, 0.05%, 0.3%, 1% |
| Price oracle | TWAP via reserves | Improved TWAP |

**Concentrated Liquidity Example:**
```
V2: $1M LP spread across ETH price $0 to $∞
    Only ~$5K is ever "active" near $3,000

V3: $1M LP concentrated between ETH $2,800 - $3,200
    All $1M is earning fees while price is in range
    200x more capital efficient!
```

**But:** If ETH moves to $4,000, your V3 position earns 0 fees and holds all ETH (no USDC). You're fully "out of range."

---

## Part 2: Interact with Uniswap V3 (2 hours)

### Reading Pool Data

```javascript
const { ethers } = require('ethers')

// Uniswap V3 Pool ABI (minimal)
const POOL_ABI = [
  "function slot0() view returns (uint160 sqrtPriceX96, int24 tick, uint16 observationIndex, uint16 observationCardinality, uint16 observationCardinalityNext, uint8 feeProtocol, bool unlocked)",
  "function liquidity() view returns (uint128)",
  "function token0() view returns (address)",
  "function token1() view returns (address)",
  "function fee() view returns (uint24)",
]

async function getPoolInfo(poolAddress) {
  const provider = new ethers.JsonRpcProvider(process.env.ALCHEMY_MAINNET_URL)
  const pool = new ethers.Contract(poolAddress, POOL_ABI, provider)
  
  const [slot0, liquidity, token0, token1, fee] = await Promise.all([
    pool.slot0(),
    pool.liquidity(),
    pool.token0(),
    pool.token1(),
    pool.fee(),
  ])
  
  // Decode sqrt price to human readable
  const sqrtPriceX96 = slot0.sqrtPriceX96
  const price = (Number(sqrtPriceX96) / 2**96) ** 2
  
  console.log("Token0:", token0)
  console.log("Token1:", token1)
  console.log("Fee:", fee.toString(), "bps =", fee / 10000, "%")
  console.log("Current price (token1/token0):", price)
  console.log("Current tick:", slot0.tick.toString())
  console.log("Active liquidity:", liquidity.toString())
}

// ETH/USDC 0.05% pool
getPoolInfo("0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640")
```

### Executing a Swap via V3 Router

```javascript
const ROUTER_ADDRESS = "0xE592427A0AEce92De3Edee1F18E0157C05861564" // V3 SwapRouter

const ROUTER_ABI = [
  "function exactInputSingle((address tokenIn, address tokenOut, uint24 fee, address recipient, uint256 deadline, uint256 amountIn, uint256 amountOutMinimum, uint160 sqrtPriceLimitX96)) external payable returns (uint256 amountOut)"
]

async function swapExactInput(
  provider, signer,
  tokenIn, tokenOut,
  fee,    // 500=0.05%, 3000=0.3%, 10000=1%
  amountIn,
  slippageBps = 50  // 0.5% slippage
) {
  const router = new ethers.Contract(ROUTER_ADDRESS, ROUTER_ABI, signer)
  const tokenInContract = new ethers.Contract(tokenIn, ["function approve(address, uint256) returns (bool)"], signer)
  
  // Approve router
  await tokenInContract.approve(ROUTER_ADDRESS, amountIn)
  
  // Quote (would use V3 Quoter in production)
  const deadline = Math.floor(Date.now() / 1000) + 60 * 20 // 20 minutes
  
  const params = {
    tokenIn,
    tokenOut,
    fee,
    recipient: await signer.getAddress(),
    deadline,
    amountIn,
    amountOutMinimum: 0n, // in production: use quoter to set proper minimum
    sqrtPriceLimitX96: 0n, // no limit
  }
  
  const tx = await router.exactInputSingle(params)
  const receipt = await tx.wait()
  
  console.log("Swap successful! Gas used:", receipt.gasUsed.toString())
  return receipt
}
```

---

## Part 3: Build a V2 Clone (2 hours)

Understanding by re-implementing. Build a minimal AMM:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title MiniAMM — Minimal x*y=k AMM for learning purposes
contract MiniAMM is ERC20, ReentrancyGuard {
    address public token0;
    address public token1;
    
    uint112 private reserve0;
    uint112 private reserve1;
    
    uint256 private constant MINIMUM_LIQUIDITY = 1000;
    
    error InsufficientLiquidity();
    error InsufficientOutputAmount();
    error InsufficientInputAmount();
    error InvalidK();
    
    event Mint(address indexed sender, uint256 amount0, uint256 amount1);
    event Burn(address indexed sender, uint256 amount0, uint256 amount1, address indexed to);
    event Swap(address indexed sender, uint256 amount0In, uint256 amount1In, uint256 amount0Out, uint256 amount1Out, address indexed to);
    
    constructor(address _token0, address _token1) ERC20("MiniAMM LP", "MMLP") {
        token0 = _token0;
        token1 = _token1;
    }
    
    function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1) {
        _reserve0 = reserve0;
        _reserve1 = reserve1;
    }
    
    /// @notice Add liquidity to pool, receive LP tokens
    function addLiquidity(uint256 amount0Desired, uint256 amount1Desired, address to) 
        external nonReentrant returns (uint256 liquidity) 
    {
        // Transfer tokens in
        IERC20(token0).transferFrom(msg.sender, address(this), amount0Desired);
        IERC20(token1).transferFrom(msg.sender, address(this), amount1Desired);
        
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        
        uint256 amount0 = balance0 - reserve0;
        uint256 amount1 = balance1 - reserve1;
        
        uint256 _totalSupply = totalSupply();
        
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
            _mint(address(0), MINIMUM_LIQUIDITY);
        } else {
            liquidity = Math.min(
                (amount0 * _totalSupply) / reserve0,
                (amount1 * _totalSupply) / reserve1
            );
        }
        
        if (liquidity == 0) revert InsufficientLiquidity();
        _mint(to, liquidity);
        
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        
        emit Mint(msg.sender, amount0, amount1);
    }
    
    /// @notice Burn LP tokens, receive both tokens back
    function removeLiquidity(uint256 lpAmount, address to) 
        external nonReentrant returns (uint256 amount0, uint256 amount1) 
    {
        uint256 _totalSupply = totalSupply();
        
        amount0 = (lpAmount * reserve0) / _totalSupply;
        amount1 = (lpAmount * reserve1) / _totalSupply;
        
        if (amount0 == 0 || amount1 == 0) revert InsufficientLiquidity();
        
        _burn(msg.sender, lpAmount);
        
        IERC20(token0).transfer(to, amount0);
        IERC20(token1).transfer(to, amount1);
        
        reserve0 = uint112(IERC20(token0).balanceOf(address(this)));
        reserve1 = uint112(IERC20(token1).balanceOf(address(this)));
        
        emit Burn(msg.sender, amount0, amount1, to);
    }
    
    /// @notice Swap tokenIn for tokenOut (x*y=k with 0.3% fee)
    function swap(uint256 amount0Out, uint256 amount1Out, address to) 
        external nonReentrant 
    {
        if (amount0Out == 0 && amount1Out == 0) revert InsufficientOutputAmount();
        
        (uint112 _reserve0, uint112 _reserve1) = getReserves();
        if (amount0Out >= _reserve0 || amount1Out >= _reserve1) revert InsufficientLiquidity();
        
        // Send output tokens optimistically
        if (amount0Out > 0) IERC20(token0).transfer(to, amount0Out);
        if (amount1Out > 0) IERC20(token1).transfer(to, amount1Out);
        
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        
        uint256 amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint256 amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        
        if (amount0In == 0 && amount1In == 0) revert InsufficientInputAmount();
        
        // Verify k invariant (with 0.3% fee)
        uint256 balance0Adjusted = balance0 * 1000 - amount0In * 3;
        uint256 balance1Adjusted = balance1 * 1000 - amount1In * 3;
        
        if (balance0Adjusted * balance1Adjusted < uint256(_reserve0) * _reserve1 * 1_000_000) {
            revert InvalidK();
        }
        
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
    
    /// @notice Helper: calculate output amount for a given input
    function getAmountOut(uint256 amountIn, uint256 reserveIn, uint256 reserveOut) 
        public pure returns (uint256 amountOut) 
    {
        if (amountIn == 0) revert InsufficientInputAmount();
        if (reserveIn == 0 || reserveOut == 0) revert InsufficientLiquidity();
        
        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = reserveIn * 1000 + amountInWithFee;
        amountOut = numerator / denominator;
    }
}
```

---

## Resources for Day 10

| Resource | Link | Type |
|----------|------|------|
| Uniswap V2 Core Source | https://github.com/Uniswap/v2-core | Code |
| Uniswap V2 Whitepaper | https://uniswap.org/whitepaper.pdf | Paper |
| Uniswap V3 Source | https://github.com/Uniswap/v3-core | Code |
| UniswapV3 Explained (Jeiwan) | https://jeiwan.net/posts/programming-defi-uniswapv3-1/ | Tutorial |
| Uniswap V3 Book | https://uniswapv3book.com/ | Book |
| Flashloan Examples | https://github.com/smartcontractkit/defi-minimal | Code |

---

## Tomorrow

[Day 11 → DeFi Protocols: Aave, Curve, Compound](./Day-4-DeFi-Protocols.md)
