# Day 24 (Week 4, Day 3) — MEV & Flash Loans

> **Time:** 6–8 hours  
> **Goal:** Understand MEV extraction — why it exists, how bots do it, and how protocols defend against it. Implement a flash loan contract.

---

## Part 1: MEV — The Dark Forest (2 hours)

### 1.1 What Is MEV?

**MEV (Maximal Extractable Value):** The maximum value that can be extracted by including, excluding, or reordering transactions within a block, above and beyond the standard block reward.

Originally called "Miner Extractable Value" (PoW), now "Maximal Extractable Value" (PoS).

**Total MEV extracted:** $600M+ since 2020 (tracked at https://explore.flashbots.net)

### 1.2 Types of MEV

**1. DEX Arbitrage (Most common, "good" MEV):**
```
Uniswap ETH/USDC price: $3,000
Sushiswap ETH/USDC price: $3,050

Arbitrageur (bot):
  Buy ETH on Uniswap at $3,000
  Sell ETH on Sushiswap at $3,050
  Profit: $50 per ETH

Effect: Prices equalize across DEXs — makes markets more efficient
```

**2. Sandwich Attacks ("bad" MEV):**
```
Alice: swap 50,000 USDC → ETH, 0.5% slippage

MEV Bot detects Alice's pending tx:
  1. [Bot] Buy ETH before Alice (higher gas = executes first)
  2. [Alice] Buy ETH at higher price (bot moved the price)
  3. [Bot] Sell ETH after Alice (at the elevated price)

Bot profit = (price impact from step 1) × amount
Alice got worse execution than expected
```

**3. Liquidation MEV:**
```
Aave: Bob's position is at 1.0 health factor (eligible for liquidation)
Liquidator Bot:
  Monitors all positions on Aave
  When health factor drops below 1.0 → submit liquidation tx ASAP
  Gets 5-10% of collateral as bonus
  Competing bots → gas wars (who submits first?)
```

**4. NFT Mint Sniping:**
```
Popular NFT drops → bots simulate all possible traits
Identify which tokenIds have rare traits (predictable from contract code)
Submit dozens of transactions to mint specific tokenIds
```

### 1.3 How MEV Bots Work

```
MEV Bot Infrastructure:
1. Archive node + WebSocket RPC (watch pending txs in mempool)
2. Simulation engine: simulate tx outcomes without broadcasting
3. Profit calculator: is this opportunity profitable after gas?
4. Bundle builder: create a bundle of transactions
5. Flashbots relay: submit bundle directly to validators (avoiding public mempool)

Loop:
  → Watch mempool for opportunities
  → Simulate: "if I front-run Alice, do I profit?"
  → If profitable: build bundle [myFrontRun, AliceTx, myBackRun]
  → Bid to validator: "Include my bundle, I'll pay you X ETH"
  → Validator includes bundle in their block
  → Profit shared: bot gets revenue, validator gets bid
```

### 1.4 Flashbots — Order From Chaos

Flashbots (https://flashbots.net) created a private communication channel between MEV bots and validators:
- Bots submit "bundles" of ordered transactions
- Validators see the bundle and decide whether to include it
- No failed transactions in public mempool (cleaner blocks)
- MEV profit is shared with validators transparently

**MEV-Boost:** Validators use MEV-Boost to receive blocks built by specialized "builders" who optimize for MEV profit and pass the excess to validators.

---

## Part 2: Flash Loans Deep Dive (2 hours)

### 2.1 How Flash Loans Work

Flash loans let you borrow any amount from a lending protocol with zero collateral, as long as you repay within the same transaction.

```
Block execution timeline:
  tx start
    │
    ├── [Aave] lend 1,000,000 USDC to your contract
    │
    ├── [Your contract] do whatever you want with the USDC
    │   - Arbitrage
    │   - Liquidate a position
    │   - Swap collateral
    │   - Attack a vulnerable protocol
    │
    ├── [Your contract] repay 1,000,000 + 0.09% fee USDC to Aave
    │
    └── [Aave] verify repayment → success
  tx end

If repayment fails → entire transaction reverts (as if nothing happened)
Zero risk to Aave!
```

### 2.2 Flash Loan Providers & Fees

| Protocol | Token Support | Fee | 
|---------|--------------|-----|
| Aave V3 | Major tokens | 0.09% |
| Uniswap V3 (flash swap) | Pool tokens | 0.3% (or 0.01-1%) |
| dYdX | ETH, DAI, USDC | 0% (subsidized) |
| Balancer | Pool tokens | 0% |
| MakerDAO | DAI | 0% |

### 2.3 Implement an Aave V3 Flash Loan

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IPool} from "@aave/core-v3/contracts/interfaces/IPool.sol";
import {IFlashLoanSimpleReceiver} from "@aave/core-v3/contracts/flashloan/interfaces/IFlashLoanSimpleReceiver.sol";
import {IPoolAddressesProvider} from "@aave/core-v3/contracts/interfaces/IPoolAddressesProvider.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @title AaveFlashLoanBase — Template for Aave flash loans
abstract contract AaveFlashLoanBase is IFlashLoanSimpleReceiver {
    IPoolAddressesProvider public immutable ADDRESSES_PROVIDER;
    IPool public immutable POOL;
    
    constructor(address provider) {
        ADDRESSES_PROVIDER = IPoolAddressesProvider(provider);
        POOL = IPool(IPoolAddressesProvider(provider).getPool());
    }
    
    /// @notice Initiate a flash loan
    function requestFlashLoan(address token, uint256 amount) external {
        POOL.flashLoanSimple(
            address(this), // receiver
            token,
            amount,
            abi.encode(msg.sender), // pass caller as params
            0               // referral code
        );
    }
    
    /// @notice Called by Aave after sending tokens to this contract
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,     // fee amount
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        
        require(msg.sender == address(POOL), "Caller must be Aave Pool");
        require(initiator == address(this), "Initiator must be this contract");
        
        // ─── YOUR LOGIC HERE ────────────────────────────────────────
        _executeFlashLoan(asset, amount, params);
        // ────────────────────────────────────────────────────────────
        
        // Approve Aave to pull repayment
        uint256 totalRepayment = amount + premium;
        IERC20(asset).approve(address(POOL), totalRepayment);
        
        return true; // must return true to confirm repayment
    }
    
    /// @notice Override this in derived contracts to implement your strategy
    function _executeFlashLoan(
        address asset,
        uint256 amount,
        bytes calldata params
    ) internal virtual;
}

/// @title ArbitrageFlashLoan — Flash loan-powered DEX arbitrage
contract ArbitrageFlashLoan is AaveFlashLoanBase {
    
    address public immutable UNISWAP_ROUTER;
    address public immutable SUSHISWAP_ROUTER;
    address public owner;
    
    constructor(
        address provider,
        address _uniswapRouter,
        address _sushiswapRouter
    ) AaveFlashLoanBase(provider) {
        UNISWAP_ROUTER = _uniswapRouter;
        SUSHISWAP_ROUTER = _sushiswapRouter;
        owner = msg.sender;
    }
    
    /// @notice Initiate arbitrage flash loan
    function executeArbitrage(
        address tokenBorrow,
        uint256 amount,
        address tokenPair,
        bool buyOnUniswap  // which DEX to buy from first
    ) external {
        require(msg.sender == owner, "Not owner");
        
        bytes memory params = abi.encode(tokenPair, buyOnUniswap);
        POOL.flashLoanSimple(address(this), tokenBorrow, amount, params, 0);
    }
    
    function _executeFlashLoan(
        address asset,
        uint256 amount,
        bytes calldata params
    ) internal override {
        (address tokenPair, bool buyOnUniswap) = abi.decode(params, (address, bool));
        
        // Step 1: Buy on DEX A
        if (buyOnUniswap) {
            _swapOnUniswap(asset, tokenPair, amount);
        } else {
            _swapOnSushiswap(asset, tokenPair, amount);
        }
        
        // Step 2: Sell on DEX B (back to original token)
        uint256 tokenPairBalance = IERC20(tokenPair).balanceOf(address(this));
        if (buyOnUniswap) {
            _swapOnSushiswap(tokenPair, asset, tokenPairBalance);
        } else {
            _swapOnUniswap(tokenPair, asset, tokenPairBalance);
        }
        
        // Note: Profit (if any) remains in contract after repayment
    }
    
    function _swapOnUniswap(address tokenIn, address tokenOut, uint256 amountIn) internal {
        IERC20(tokenIn).approve(UNISWAP_ROUTER, amountIn);
        // Call Uniswap router.exactInputSingle(...)
    }
    
    function _swapOnSushiswap(address tokenIn, address tokenOut, uint256 amountIn) internal {
        IERC20(tokenIn).approve(SUSHISWAP_ROUTER, amountIn);
        // Call SushiSwap router...
    }
    
    function withdrawProfit(address token) external {
        require(msg.sender == owner);
        IERC20(token).transfer(owner, IERC20(token).balanceOf(address(this)));
    }
}
```

### 2.4 Flash Loan for Collateral Swap

A more common real-world use: swap your collateral on Aave without closing the position.

```
Scenario: You have WBTC collateral on Aave, borrowing USDC.
You want to switch collateral to ETH without repaying.

Flash loan flow:
1. Flash borrow enough ETH to repay WBTC position
2. Repay WBTC-collateralized USDC loan on Aave
3. Withdraw your WBTC collateral
4. Swap WBTC → ETH on Uniswap
5. Deposit ETH as new collateral on Aave
6. Borrow USDC again
7. Repay flash loan of ETH + fee
```

---

## Part 3: Protect Your Protocol from MEV (1 hour)

### Use Flashbots Protect RPC

```javascript
// Use Flashbots Protect as your RPC endpoint
// All transactions are submitted privately — no sandwich attacks!
const provider = new ethers.JsonRpcProvider("https://rpc.flashbots.net")
// OR
const provider = new ethers.JsonRpcProvider("https://rpc.mevblocker.io") // MEV Blocker
```

### Commit-Reveal Scheme

```solidity
// For auctions/game mechanics where front-running is a concern
contract CommitReveal {
    mapping(bytes32 => bool) public commits;
    mapping(address => uint256) public revealedValues;
    
    // Phase 1: Commit (keep your value secret)
    function commit(bytes32 commitment) external {
        // commitment = keccak256(abi.encodePacked(value, nonce, msg.sender))
        commits[commitment] = true;
    }
    
    // Phase 2: Reveal (after commit phase ends)
    function reveal(uint256 value, uint256 nonce) external {
        bytes32 commitment = keccak256(abi.encodePacked(value, nonce, msg.sender));
        require(commits[commitment], "Commitment not found");
        
        delete commits[commitment];
        revealedValues[msg.sender] = value;
    }
}
```

---

## Resources for Day 24

| Resource | Link | Type |
|----------|------|------|
| Flashbots Research | https://research.flashbots.net/ | Articles |
| MEV Explorer | https://explore.flashbots.net/ | Dashboard |
| Aave Flash Loan Docs | https://docs.aave.com/developers/guides/flash-loans | Docs |
| Uniswap Flash Swaps | https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/using-flash-swaps | Docs |
| Dark Forest MEV | https://www.paradigm.xyz/2020/08/ethereum-is-a-dark-forest | Article |
| Rekt News | https://rekt.news | Incident reports |

---

## Tomorrow

[Day 25 → Oracles & Chainlink](./Day-4-Oracles-Chainlink.md)
