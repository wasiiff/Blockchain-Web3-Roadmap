# Day 11 (Week 2, Day 4) — DeFi Protocols: Lending, Staking & Yield

> **Time:** 6–8 hours  
> **Goal:** Understand the major DeFi protocols — how Aave, Compound, Curve, and staking work under the hood.

---

## Part 1: Lending Protocols (2 hours)

### 1.1 How Aave Works

Aave is a decentralized lending protocol where:
- **Depositors** supply assets and earn interest (as lenders)
- **Borrowers** take loans by providing collateral
- Interest rates are algorithmically set based on **utilization rate**

```
Utilization Rate = Total Borrowed / Total Supplied

Example: Pool has 1,000,000 USDC
         700,000 USDC is borrowed
         Utilization = 70%

At 70% utilization:
- Borrow APY: ~8% (high demand = high rate)
- Supply APY: ~5.6% (borrow rate × utilization)
```

**Overcollateralization:** To borrow $100, you must deposit $130+ worth of collateral. (130% collateral ratio)

**The liquidation mechanism:**
```
If collateral value drops → Health Factor falls
Health Factor = (Collateral × Liquidation Threshold) / Total Debt

If Health Factor < 1.0 → liquidation can occur
Liquidator repays part of the debt
Liquidator receives collateral at discount (5-10% bonus)
This creates a profitable liquidation bot ecosystem
```

### 1.2 Interest Rate Model

```
                     Optimal Utilization (80%)
                              │
Supply APY                    │  STEEP INCREASE
     │                        │  (panic zone — too much borrowed)
 12% │                      ╱ │
 10% │                    ╱   │
  8% │                  ╱     │
  6% │                ╱       │
  4% │              ╱         │
  2% │────────────╱           │
  0% └──────────────────────────────── Utilization
     0%         50%      80%   100%
```

When utilization exceeds the "optimal" point, rates spike dramatically. This discourages all capital being borrowed (leaving no liquidity for withdrawals).

### 1.3 Interact with Aave V3

```javascript
// Aave V3 Pool on mainnet
const AAVE_POOL = "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2"

const POOL_ABI = [
  "function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external",
  "function borrow(address asset, uint256 amount, uint256 interestRateMode, uint16 referralCode, address onBehalfOf) external",
  "function repay(address asset, uint256 amount, uint256 interestRateMode, address onBehalfOf) external returns (uint256)",
  "function withdraw(address asset, uint256 amount, address to) external returns (uint256)",
  "function getUserAccountData(address user) external view returns (uint256 totalCollateralBase, uint256 totalDebtBase, uint256 availableBorrowsBase, uint256 currentLiquidationThreshold, uint256 ltv, uint256 healthFactor)",
]

async function supplyAndBorrow() {
  const pool = new ethers.Contract(AAVE_POOL, POOL_ABI, signer)
  const usdc = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, signer)
  
  // 1. Supply USDC as collateral
  const supplyAmount = ethers.parseUnits("1000", 6) // 1000 USDC
  await usdc.approve(AAVE_POOL, supplyAmount)
  await pool.supply(USDC_ADDRESS, supplyAmount, myAddress, 0)
  
  // 2. Check how much we can borrow
  const accountData = await pool.getUserAccountData(myAddress)
  console.log("Health factor:", ethers.formatUnits(accountData.healthFactor, 18))
  console.log("Available borrows (USD):", ethers.formatUnits(accountData.availableBorrowsBase, 8))
  
  // 3. Borrow ETH (variable rate = 2)
  const borrowAmount = ethers.parseEther("0.1") // 0.1 ETH
  await pool.borrow(WETH_ADDRESS, borrowAmount, 2, 0, myAddress)
  
  console.log("Borrowed 0.1 WETH")
}
```

---

## Part 2: Curve Finance (1.5 hours)

### 2.1 Stableswap — The AMM for Stablecoins

Uniswap's x·y=k works poorly for stablecoins (USDC/USDT/DAI) because:
- These should always trade near 1:1
- x·y=k moves price significantly even for small trades
- Creates unnecessary slippage for $1 = $1 swaps

Curve's **StableSwap formula** is a hybrid:
```
A·n^n·Σxi + D = A·D·n^n + D^(n+1) / (n^n·Πxi)

When A (amplification) is large: behaves like constant sum (x+y=k)
When A is small: behaves like constant product (x*y=k)
In between: flat curve near peg, steep curve away from peg
```

**Visualization:**
```
Price                Stableswap (A=100)
  │                 ┌────────────────────┐
1.01│                │ very flat near peg │
1.00│─────────────────────────────────────
0.99│                └────────────────────┘
   │ x·y=k (Uniswap)
   └────────────────────────────────────── ratio
```

Curve allows 10-100x less slippage for stable pairs.

### 2.2 Curve Pools

Types of Curve pools:
- **3pool (3CRV):** USDC + USDT + DAI (most liquid, reference stablecoin pool)
- **Tricrypto:** ETH + WBTC + USDT (crypto assets)
- **metapools:** A new stablecoin paired with 3CRV (bootstraps liquidity)

**CRV Token and veTokenomics:**
- CRV = Curve governance + reward token
- Lock CRV for veCRV (vote-escrow) — up to 4 years
- veCRV earns: trading fees + boosted CRV rewards + governance votes
- Governance votes direct CRV emissions to pools ("gauge weights")
- Creates "Curve Wars" — protocols bribe veCRV holders to vote for their pools

---

## Part 3: Staking and Yield Farming (1 hour)

### 3.1 Simple Staking Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title StakingRewards — Synthetix-style staking rewards
contract StakingRewards is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    IERC20 public stakingToken;    // token you stake (e.g., LP token)
    IERC20 public rewardToken;     // token you earn (e.g., governance token)
    
    uint256 public rewardRate;     // rewards per second
    uint256 public lastUpdateTime;
    uint256 public rewardPerTokenStored;
    
    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;
    mapping(address => uint256) public balanceOf;
    
    uint256 public totalSupply;
    
    address public owner;
    uint256 public periodFinish;
    uint256 public rewardsDuration = 7 days;
    
    constructor(address _stakingToken, address _rewardToken) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
        owner = msg.sender;
    }
    
    // ─── View Functions ───────────────────────────────────────────────
    
    function lastTimeRewardApplicable() public view returns (uint256) {
        return block.timestamp < periodFinish ? block.timestamp : periodFinish;
    }
    
    function rewardPerToken() public view returns (uint256) {
        if (totalSupply == 0) return rewardPerTokenStored;
        return rewardPerTokenStored + (
            (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply
        );
    }
    
    function earned(address account) public view returns (uint256) {
        return (
            balanceOf[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
        ) + rewards[account];
    }
    
    // ─── Mutating Functions ───────────────────────────────────────────
    
    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = lastTimeRewardApplicable();
        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }
    
    function stake(uint256 amount) external nonReentrant updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");
        totalSupply += amount;
        balanceOf[msg.sender] += amount;
        stakingToken.safeTransferFrom(msg.sender, address(this), amount);
    }
    
    function withdraw(uint256 amount) external nonReentrant updateReward(msg.sender) {
        require(amount > 0, "Cannot withdraw 0");
        totalSupply -= amount;
        balanceOf[msg.sender] -= amount;
        stakingToken.safeTransfer(msg.sender, amount);
    }
    
    function claimReward() external nonReentrant updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardToken.safeTransfer(msg.sender, reward);
        }
    }
    
    function exit() external {
        withdraw(balanceOf[msg.sender]);
        claimReward();
    }
    
    // ─── Admin ────────────────────────────────────────────────────────
    
    function notifyRewardAmount(uint256 reward) external updateReward(address(0)) {
        require(msg.sender == owner);
        if (block.timestamp >= periodFinish) {
            rewardRate = reward / rewardsDuration;
        } else {
            uint256 remaining = periodFinish - block.timestamp;
            uint256 leftover = remaining * rewardRate;
            rewardRate = (reward + leftover) / rewardsDuration;
        }
        
        lastUpdateTime = block.timestamp;
        periodFinish = block.timestamp + rewardsDuration;
    }
}
```

---

## Part 4: DeFi Protocol Comparison (30 min)

| Protocol | Type | TVL | Token | Revenue Source |
|---------|------|-----|-------|----------------|
| Uniswap | DEX (AMM) | $6B+ | UNI | Trading fees |
| Aave | Lending | $12B+ | AAVE | Interest spread |
| Curve | DEX (Stable) | $2B+ | CRV | Trading fees |
| Compound | Lending | $2B+ | COMP | Interest spread |
| MakerDAO | CDP/Stablecoin | $8B+ | MKR | Stability fees |
| Lido | Liquid Staking | $30B+ | LDO | Staking commission |
| Convex | Yield | $3B+ | CVX | Curve fee boost |

---

## Part 5: Exercises (1.5 hours)

### Exercise 1: Fork Mainnet and Interact with Aave

```javascript
// hardhat.config.js: enable forking
// Then write a test that:
// 1. Impersonates a whale address with lots of USDC
// 2. Supplies USDC to Aave V3
// 3. Borrows ETH against it
// 4. Checks health factor
// 5. Repays some debt
// 6. Checks health factor improved

const { impersonateAccount } = require("@nomicfoundation/hardhat-network-helpers")

it("should supply to Aave and borrow", async function() {
  const whale = "0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503" // USDC whale
  await impersonateAccount(whale)
  const signer = await ethers.getSigner(whale)
  // ... implement the test
})
```

### Exercise 2: Calculate Yield APY

Write a function that fetches Aave's current supply APY for any token:

```javascript
const DATA_PROVIDER = "0x7B4EB56E7CD4b454BA8ff71E4518426369a138a3"
const ABI = ["function getReserveData(address asset) view returns (...)"]

async function getAaveAPY(tokenAddress) {
  const provider = new ethers.JsonRpcProvider(process.env.ALCHEMY_URL)
  const dataProvider = new ethers.Contract(DATA_PROVIDER, ABI, provider)
  const data = await dataProvider.getReserveData(tokenAddress)
  
  // liquidityRate is in RAY units (1e27)
  const liquidityRateRay = data.currentLiquidityRate
  const supplyAPY = ((1 + Number(liquidityRateRay) / 1e27 / 365) ** 365 - 1) * 100
  
  return supplyAPY.toFixed(2) + "%"
}
```

---

## Resources for Day 11

| Resource | Link | Type |
|----------|------|------|
| Aave V3 Docs | https://docs.aave.com/developers/getting-started/readme | Docs |
| Aave V3 Source | https://github.com/aave/aave-v3-core | Code |
| Curve Docs | https://curve.readthedocs.io/ | Docs |
| Compound Docs | https://docs.compound.finance/ | Docs |
| DeFi MOOC (Berkeley) | https://defi-learning.org/ | Course |
| Synthetix Staking | https://github.com/Synthetixio/synthetix | Code |

---

## Tomorrow

[Day 12 → Frontend: Ethers.js & Wagmi](./Day-5-Frontend-Ethers-Wagmi.md)
