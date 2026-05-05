# Day 27 (Week 4, Day 6) — Capstone Implementation

> **Time:** Full day  
> **Goal:** Implement PixelSwap — Factory, Pair, Router contracts with full tests.

---

## Part 1: PixelSwapFactory

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./PixelSwapPair.sol";

contract PixelSwapFactory {
    address public feeTo;        // platform fee recipient (if enabled)
    address public feeToSetter; // who can change feeTo
    
    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;
    
    error IdenticalAddresses();
    error ZeroAddress();
    error PairExists();
    
    event PairCreated(address indexed token0, address indexed token1, address pair, uint256 pairIndex);
    
    constructor(address _feeToSetter) {
        feeToSetter = _feeToSetter;
    }
    
    function allPairsLength() external view returns (uint256) {
        return allPairs.length;
    }
    
    function createPair(address tokenA, address tokenB) external returns (address pair) {
        if (tokenA == tokenB) revert IdenticalAddresses();
        
        // Sort tokens — deterministic ordering
        (address token0, address token1) = tokenA < tokenB 
            ? (tokenA, tokenB) 
            : (tokenB, tokenA);
        
        if (token0 == address(0)) revert ZeroAddress();
        if (getPair[token0][token1] != address(0)) revert PairExists();
        
        // Deploy pair using CREATE2 — address is deterministic
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        pair = address(new PixelSwapPair{salt: salt}(token0, token1));
        
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // both directions
        allPairs.push(pair);
        
        emit PairCreated(token0, token1, pair, allPairs.length);
    }
    
    function setFeeTo(address _feeTo) external {
        require(msg.sender == feeToSetter, "Not allowed");
        feeTo = _feeTo;
    }
    
    function setFeeToSetter(address _feeToSetter) external {
        require(msg.sender == feeToSetter, "Not allowed");
        feeToSetter = _feeToSetter;
    }
    
    /// @notice Calculate the deterministic pair address (without deploying)
    function pairFor(address tokenA, address tokenB) public view returns (address) {
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        
        return address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            salt,
            keccak256(type(PixelSwapPair).creationCode)
        )))));
    }
}
```

---

## Part 2: PixelSwapRouter (Key Functions)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "./interfaces/IPixelSwapFactory.sol";
import "./interfaces/IPixelSwapPair.sol";
import "./interfaces/IWETH.sol";

contract PixelSwapRouter {
    using SafeERC20 for IERC20;
    
    address public immutable factory;
    address public immutable WETH;
    
    modifier ensure(uint256 deadline) {
        require(deadline >= block.timestamp, "PixelSwap: EXPIRED");
        _;
    }
    
    constructor(address _factory, address _WETH) {
        factory = _factory;
        WETH = _WETH;
    }
    
    receive() external payable {
        require(msg.sender == WETH, "Only WETH can send ETH");
    }
    
    // ─── Internal Library Functions ──────────────────────────────────
    
    function _pairFor(address tokenA, address tokenB) internal view returns (address pair) {
        return IPixelSwapFactory(factory).getPair(tokenA, tokenB);
    }
    
    function _getReserves(address tokenA, address tokenB) 
        internal view returns (uint256 reserveA, uint256 reserveB) 
    {
        (address token0,) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        (uint256 reserve0, uint256 reserve1,) = IPixelSwapPair(_pairFor(tokenA, tokenB)).getReserves();
        (reserveA, reserveB) = tokenA == token0 ? (reserve0, reserve1) : (reserve1, reserve0);
    }
    
    function _quote(uint256 amountA, uint256 reserveA, uint256 reserveB)
        internal pure returns (uint256 amountB)
    {
        require(amountA > 0, "Insufficient amount");
        require(reserveA > 0 && reserveB > 0, "Insufficient liquidity");
        amountB = amountA * reserveB / reserveA;
    }
    
    function getAmountOut(uint256 amountIn, uint256 reserveIn, uint256 reserveOut)
        public pure returns (uint256 amountOut)
    {
        require(amountIn > 0, "Insufficient input");
        require(reserveIn > 0 && reserveOut > 0, "Insufficient liquidity");
        uint256 amountInWithFee = amountIn * 997;
        amountOut = (amountInWithFee * reserveOut) / (reserveIn * 1000 + amountInWithFee);
    }
    
    function getAmountsOut(uint256 amountIn, address[] memory path)
        public view returns (uint256[] memory amounts)
    {
        require(path.length >= 2, "Invalid path");
        amounts = new uint256[](path.length);
        amounts[0] = amountIn;
        
        for (uint256 i = 0; i < path.length - 1; i++) {
            (uint256 reserveIn, uint256 reserveOut) = _getReserves(path[i], path[i + 1]);
            amounts[i + 1] = getAmountOut(amounts[i], reserveIn, reserveOut);
        }
    }
    
    // ─── Add Liquidity ───────────────────────────────────────────────
    
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountADesired,
        uint256 amountBDesired,
        uint256 amountAMin,
        uint256 amountBMin,
        address to,
        uint256 deadline
    ) external ensure(deadline) returns (uint256 amountA, uint256 amountB, uint256 liquidity) {
        
        // Create pair if it doesn't exist
        if (IPixelSwapFactory(factory).getPair(tokenA, tokenB) == address(0)) {
            IPixelSwapFactory(factory).createPair(tokenA, tokenB);
        }
        
        (uint256 reserveA, uint256 reserveB) = _getReserves(tokenA, tokenB);
        
        if (reserveA == 0 && reserveB == 0) {
            // First liquidity: use desired amounts exactly
            (amountA, amountB) = (amountADesired, amountBDesired);
        } else {
            // Maintain ratio: quote amountB based on amountA
            uint256 amountBOptimal = _quote(amountADesired, reserveA, reserveB);
            
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, "Insufficient B");
                (amountA, amountB) = (amountADesired, amountBOptimal);
            } else {
                uint256 amountAOptimal = _quote(amountBDesired, reserveB, reserveA);
                require(amountAOptimal <= amountADesired && amountAOptimal >= amountAMin, "Insufficient A");
                (amountA, amountB) = (amountAOptimal, amountBDesired);
            }
        }
        
        address pair = _pairFor(tokenA, tokenB);
        IERC20(tokenA).safeTransferFrom(msg.sender, pair, amountA);
        IERC20(tokenB).safeTransferFrom(msg.sender, pair, amountB);
        liquidity = IPixelSwapPair(pair).mint(to);
    }
    
    // ─── Swaps ────────────────────────────────────────────────────────
    
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external ensure(deadline) returns (uint256[] memory amounts) {
        amounts = getAmountsOut(amountIn, path);
        require(amounts[amounts.length - 1] >= amountOutMin, "Insufficient output");
        
        // Transfer first token to first pair
        IERC20(path[0]).safeTransferFrom(
            msg.sender,
            _pairFor(path[0], path[1]),
            amounts[0]
        );
        
        _swap(amounts, path, to);
    }
    
    function swapExactETHForTokens(
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external payable ensure(deadline) returns (uint256[] memory amounts) {
        require(path[0] == WETH, "Invalid path");
        
        amounts = getAmountsOut(msg.value, path);
        require(amounts[amounts.length - 1] >= amountOutMin, "Insufficient output");
        
        // Wrap ETH → WETH
        IWETH(WETH).deposit{value: amounts[0]}();
        IERC20(WETH).safeTransfer(_pairFor(path[0], path[1]), amounts[0]);
        
        _swap(amounts, path, to);
    }
    
    function _swap(uint256[] memory amounts, address[] memory path, address _to) internal {
        for (uint256 i = 0; i < path.length - 1; i++) {
            (address input, address output) = (path[i], path[i + 1]);
            (address token0,) = input < output ? (input, output) : (output, input);
            
            uint256 amountOut = amounts[i + 1];
            (uint256 amount0Out, uint256 amount1Out) = input == token0 
                ? (uint256(0), amountOut) 
                : (amountOut, uint256(0));
            
            // If this is the last hop, send to `to`, otherwise to next pair
            address to = i < path.length - 2 
                ? _pairFor(output, path[i + 2]) 
                : _to;
            
            IPixelSwapPair(_pairFor(input, output)).swap(amount0Out, amount1Out, to, new bytes(0));
        }
    }
}
```

---

## Part 3: Test Suite (Comprehensive)

```javascript
// test/PixelSwap.integration.test.js
const { expect } = require("chai")
const { ethers } = require("hardhat")
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers")

describe("PixelSwap Integration", function () {
  async function deployFixture() {
    const [owner, alice, bob, carol] = await ethers.getSigners()
    
    // Deploy tokens
    const Token = await ethers.getContractFactory("DevToken")
    const tokenA = await Token.deploy("Token A", "TKA", ethers.parseEther("10000000"), owner.address)
    const tokenB = await Token.deploy("Token B", "TKB", ethers.parseEther("10000000"), owner.address)
    const tokenC = await Token.deploy("Token C", "TKC", ethers.parseEther("10000000"), owner.address)
    
    // Deploy WETH
    const WETH = await ethers.getContractFactory("WETH9")
    const weth = await WETH.deploy()
    
    // Deploy Factory and Router
    const Factory = await ethers.getContractFactory("PixelSwapFactory")
    const factory = await Factory.deploy(owner.address)
    
    const Router = await ethers.getContractFactory("PixelSwapRouter")
    const router = await Router.deploy(await factory.getAddress(), await weth.getAddress())
    
    // Fund users
    for (const user of [alice, bob, carol]) {
      await tokenA.transfer(user.address, ethers.parseEther("100000"))
      await tokenB.transfer(user.address, ethers.parseEther("100000"))
      await tokenC.transfer(user.address, ethers.parseEther("100000"))
    }
    
    return { factory, router, weth, tokenA, tokenB, tokenC, owner, alice, bob, carol }
  }
  
  describe("Factory", function () {
    it("creates a pair deterministically", async function () {
      const { factory, tokenA, tokenB } = await loadFixture(deployFixture)
      
      const pairAddress = await factory.callStatic.createPair(
        await tokenA.getAddress(),
        await tokenB.getAddress()
      )
      
      await factory.createPair(await tokenA.getAddress(), await tokenB.getAddress())
      
      expect(await factory.getPair(
        await tokenA.getAddress(),
        await tokenB.getAddress()
      )).to.equal(pairAddress)
      
      expect(await factory.allPairsLength()).to.equal(1)
    })
    
    it("reverts on duplicate pair creation", async function () {
      const { factory, tokenA, tokenB } = await loadFixture(deployFixture)
      
      await factory.createPair(await tokenA.getAddress(), await tokenB.getAddress())
      
      await expect(
        factory.createPair(await tokenA.getAddress(), await tokenB.getAddress())
      ).to.be.revertedWithCustomError(factory, "PairExists")
    })
  })
  
  describe("Add Liquidity & Swap", function () {
    it("adds initial liquidity and swaps", async function () {
      const { router, tokenA, tokenB, alice, bob } = await loadFixture(deployFixture)
      
      const liquidityA = ethers.parseEther("10000")
      const liquidityB = ethers.parseEther("30000")  // price: 1A = 3B
      
      // Alice adds initial liquidity
      await tokenA.connect(alice).approve(await router.getAddress(), liquidityA)
      await tokenB.connect(alice).approve(await router.getAddress(), liquidityB)
      
      const deadline = Math.floor(Date.now() / 1000) + 3600
      
      await router.connect(alice).addLiquidity(
        await tokenA.getAddress(),
        await tokenB.getAddress(),
        liquidityA, liquidityB,
        0, 0,
        alice.address, deadline
      )
      
      console.log("✓ Liquidity added: 10,000 TokenA + 30,000 TokenB")
      
      // Bob swaps 100 TokenA → TokenB
      const swapAmount = ethers.parseEther("100")
      const balanceBefore = await tokenB.balanceOf(bob.address)
      
      await tokenA.connect(bob).approve(await router.getAddress(), swapAmount)
      
      const amounts = await router.getAmountsOut(swapAmount, [
        await tokenA.getAddress(),
        await tokenB.getAddress()
      ])
      
      const minOut = amounts[1] * 995n / 1000n // 0.5% slippage
      
      await router.connect(bob).swapExactTokensForTokens(
        swapAmount, minOut,
        [await tokenA.getAddress(), await tokenB.getAddress()],
        bob.address, deadline
      )
      
      const balanceAfter = await tokenB.balanceOf(bob.address)
      const received = balanceAfter - balanceBefore
      
      console.log("✓ Swapped 100 TokenA → ", ethers.formatEther(received), "TokenB")
      expect(received).to.be.gt(ethers.parseEther("296")) // approximately 297 TokenB
    })
    
    it("supports multi-hop swaps (A→B→C)", async function () {
      const { router, tokenA, tokenB, tokenC, alice, bob } = await loadFixture(deployFixture)
      
      const deadline = Math.floor(Date.now() / 1000) + 3600
      
      // Add A/B liquidity
      await tokenA.connect(alice).approve(await router.getAddress(), ethers.parseEther("10000"))
      await tokenB.connect(alice).approve(await router.getAddress(), ethers.parseEther("30000"))
      await router.connect(alice).addLiquidity(
        await tokenA.getAddress(), await tokenB.getAddress(),
        ethers.parseEther("10000"), ethers.parseEther("30000"), 0, 0,
        alice.address, deadline
      )
      
      // Add B/C liquidity
      await tokenB.connect(alice).approve(await router.getAddress(), ethers.parseEther("30000"))
      await tokenC.connect(alice).approve(await router.getAddress(), ethers.parseEther("90000"))
      await router.connect(alice).addLiquidity(
        await tokenB.getAddress(), await tokenC.getAddress(),
        ethers.parseEther("30000"), ethers.parseEther("90000"), 0, 0,
        alice.address, deadline
      )
      
      // Bob swaps A→C (through B)
      const swapAmount = ethers.parseEther("100")
      await tokenA.connect(bob).approve(await router.getAddress(), swapAmount)
      
      const amounts = await router.getAmountsOut(swapAmount, [
        await tokenA.getAddress(),
        await tokenB.getAddress(),
        await tokenC.getAddress()
      ])
      
      await router.connect(bob).swapExactTokensForTokens(
        swapAmount,
        amounts[2] * 99n / 100n,
        [await tokenA.getAddress(), await tokenB.getAddress(), await tokenC.getAddress()],
        bob.address,
        deadline
      )
      
      console.log("✓ Multi-hop: 100 A →", ethers.formatEther(amounts[1]), "B →", ethers.formatEther(amounts[2]), "C")
    })
  })
})
```

---

## Resources for Day 27

| Resource | Link | Type |
|----------|------|------|
| Uniswap V2 Core | https://github.com/Uniswap/v2-core | Reference |
| Uniswap V2 Periphery | https://github.com/Uniswap/v2-periphery | Reference |
| CREATE2 Explained | https://eips.ethereum.org/EIPS/eip-1014 | Spec |

---

## Tomorrow

[Day 28 → Capstone Final Deploy & Review](./Day-7-Capstone-Final.md)
