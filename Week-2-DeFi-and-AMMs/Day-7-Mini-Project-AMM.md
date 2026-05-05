# Day 14 (Week 2, Day 7) — Mini Project: Deploy Your Own AMM

> **Time:** Full day  
> **Goal:** Deploy a complete AMM (SimpleSwap) — two-token pool with add/remove liquidity and token swaps. Frontend included.

---

## Project: SimpleSwap AMM

**Features:**
- Deploy two ERC-20 test tokens (TokenA, TokenB)
- Deploy MiniAMM pool for A/B pair
- Frontend: swap, add liquidity, remove liquidity, pool stats
- All on Sepolia testnet

---

## Step 1: Deploy Test Tokens + Pool

```javascript
// scripts/deploySimpleSwap.js
const { ethers, run } = require("hardhat")

async function main() {
  const [deployer] = await ethers.getSigners()
  console.log("Deployer:", deployer.address)
  
  // Deploy TokenA
  const TokenA = await ethers.getContractFactory("DevToken")
  const tokenA = await TokenA.deploy("Simple A", "SMPA", ethers.parseEther("1000000"), deployer.address)
  await tokenA.waitForDeployment()
  console.log("TokenA:", await tokenA.getAddress())
  
  // Deploy TokenB
  const TokenB = await ethers.getContractFactory("DevToken")
  const tokenB = await TokenB.deploy("Simple B", "SMPB", ethers.parseEther("1000000"), deployer.address)
  await tokenB.waitForDeployment()
  console.log("TokenB:", await tokenB.getAddress())
  
  // Deploy AMM
  const MiniAMM = await ethers.getContractFactory("MiniAMM")
  const amm = await MiniAMM.deploy(await tokenA.getAddress(), await tokenB.getAddress())
  await amm.waitForDeployment()
  console.log("MiniAMM:", await amm.getAddress())
  
  // Seed initial liquidity: 100,000 TokenA + 300,000 TokenB (price: 1 A = 3 B)
  const amountA = ethers.parseEther("100000")
  const amountB = ethers.parseEther("300000")
  
  await tokenA.approve(await amm.getAddress(), amountA)
  await tokenB.approve(await amm.getAddress(), amountB)
  await amm.addLiquidity(amountA, amountB, deployer.address)
  
  const [r0, r1] = await amm.getReserves()
  console.log("\nPool seeded!")
  console.log("Reserve0 (TokenA):", ethers.formatEther(r0))
  console.log("Reserve1 (TokenB):", ethers.formatEther(r1))
  console.log("Initial price: 1 TokenA =", Number(r1)/Number(r0), "TokenB")
  
  // Save addresses
  const fs = require("fs")
  fs.writeFileSync("./simpleswap-addresses.json", JSON.stringify({
    tokenA: await tokenA.getAddress(),
    tokenB: await tokenB.getAddress(),
    amm: await amm.getAddress(),
    deployer: deployer.address,
    network: "sepolia",
  }, null, 2))
}

main().catch(console.error)
```

---

## Step 2: Full Tests

```javascript
// test/SimpleSwap.test.js
const { expect } = require("chai")
const { ethers } = require("hardhat")
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers")

describe("SimpleSwap Integration", function () {
  async function deployAll() {
    const [owner, alice, bob] = await ethers.getSigners()
    
    const Token = await ethers.getContractFactory("DevToken")
    const tokenA = await Token.deploy("Token A", "TKA", ethers.parseEther("1000000"), owner.address)
    const tokenB = await Token.deploy("Token B", "TKB", ethers.parseEther("1000000"), owner.address)
    
    const AMM = await ethers.getContractFactory("MiniAMM")
    const amm = await AMM.deploy(await tokenA.getAddress(), await tokenB.getAddress())
    
    // Seed liquidity
    const seedA = ethers.parseEther("10000")
    const seedB = ethers.parseEther("30000")
    await tokenA.approve(await amm.getAddress(), seedA)
    await tokenB.approve(await amm.getAddress(), seedB)
    await amm.addLiquidity(seedA, seedB, owner.address)
    
    // Give alice some tokens
    await tokenA.transfer(alice.address, ethers.parseEther("1000"))
    await tokenB.transfer(alice.address, ethers.parseEther("1000"))
    
    return { amm, tokenA, tokenB, owner, alice, bob }
  }
  
  describe("Swap A → B", function () {
    it("should give more tokenB when price is 1:3", async function () {
      const { amm, tokenA, tokenB, alice } = await loadFixture(deployAll)
      
      const swapAmountA = ethers.parseEther("100")
      const balanceBefore = await tokenB.balanceOf(alice.address)
      
      // Approve + swap
      await tokenA.connect(alice).approve(await amm.getAddress(), swapAmountA)
      
      // Calculate expected output
      const [r0, r1] = await amm.getReserves()
      const expectedOut = await amm.getAmountOut(swapAmountA, r0, r1)
      console.log("Expected TokenB out:", ethers.formatEther(expectedOut))
      
      // Swap: we give A, get B
      await amm.connect(alice).swap(0, expectedOut, alice.address)
      
      const balanceAfter = await tokenB.balanceOf(alice.address)
      expect(balanceAfter - balanceBefore).to.be.closeTo(expectedOut, ethers.parseEther("0.01"))
    })
    
    it("should increase reserves on swap", async function () {
      const { amm, tokenA, alice } = await loadFixture(deployAll)
      const [r0Before] = await amm.getReserves()
      
      const swapAmount = ethers.parseEther("100")
      await tokenA.connect(alice).approve(await amm.getAddress(), swapAmount)
      const [, r1] = await amm.getReserves()
      const out = await amm.getAmountOut(swapAmount, r0Before, r1)
      await amm.connect(alice).swap(0, out, alice.address)
      
      const [r0After] = await amm.getReserves()
      expect(r0After).to.be.gt(r0Before)
    })
  })
  
  describe("Add Liquidity", function () {
    it("alice can add liquidity and get LP tokens", async function () {
      const { amm, tokenA, tokenB, alice } = await loadFixture(deployAll)
      
      const amountA = ethers.parseEther("100")
      const amountB = ethers.parseEther("300")
      
      await tokenA.connect(alice).approve(await amm.getAddress(), amountA)
      await tokenB.connect(alice).approve(await amm.getAddress(), amountB)
      
      const tx = await amm.connect(alice).addLiquidity(amountA, amountB, alice.address)
      
      const lpBalance = await amm.balanceOf(alice.address)
      expect(lpBalance).to.be.gt(0n)
      console.log("Alice's LP tokens:", ethers.formatEther(lpBalance))
    })
  })
  
  describe("Remove Liquidity", function () {
    it("should return proportional tokens", async function () {
      const { amm, tokenA, tokenB, owner } = await loadFixture(deployAll)
      
      const lpBalance = await amm.balanceOf(owner.address)
      const halfLP = lpBalance / 2n
      
      const balA = await tokenA.balanceOf(owner.address)
      const balB = await tokenB.balanceOf(owner.address)
      
      await amm.removeLiquidity(halfLP, owner.address)
      
      const newBalA = await tokenA.balanceOf(owner.address)
      const newBalB = await tokenB.balanceOf(owner.address)
      
      expect(newBalA).to.be.gt(balA)
      expect(newBalB).to.be.gt(balB)
    })
  })
})
```

---

## Step 3: Final Checklist

- [ ] TokenA deployed and verified on Sepolia
- [ ] TokenB deployed and verified on Sepolia
- [ ] MiniAMM deployed and verified on Sepolia
- [ ] Pool seeded with initial liquidity
- [ ] Frontend connects to MetaMask
- [ ] Swap A→B and B→A work
- [ ] Price impact shown on UI
- [ ] Add liquidity works
- [ ] Remove liquidity works
- [ ] Pool stats update after each action

---

## Week 2 Complete!

**What you built this week:**
- Deep understanding of x·y=k AMM mechanics
- Impermanent loss calculations
- Uniswap V2/V3 architecture
- DeFi protocols: Aave, Curve, staking
- Wagmi + Viem React frontend
- Deployed working AMM with frontend

**Next:** [Week 3 → NFTs & Marketplaces](../Week-3-NFTs-and-Marketplaces/README.md)
