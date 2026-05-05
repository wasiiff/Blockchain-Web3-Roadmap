# Day 5 — Hardhat Setup + First Real Deployment

> **Time:** 6–8 hours  
> **Goal:** Set up a professional Hardhat development environment, write tests, and deploy a verified contract to Sepolia testnet.

---

## Part 1: Hardhat Project Setup (1 hour)

### 1.1 Initialize Project

```bash
mkdir my-defi-project
cd my-defi-project
npm init -y
npm install --save-dev hardhat

# Initialize Hardhat (choose "Create a JavaScript project")
npx hardhat init

# Install essential plugins
npm install --save-dev @nomicfoundation/hardhat-toolbox
npm install --save-dev @nomicfoundation/hardhat-verify
npm install @openzeppelin/contracts
npm install dotenv
```

### 1.2 Project Structure
```
my-defi-project/
├── contracts/           ← Solidity source files
│   └── Lock.sol
├── scripts/             ← Deployment scripts
│   └── deploy.js
├── test/                ← Test files
│   └── Lock.test.js
├── ignition/            ← Hardhat Ignition modules (new deployment system)
├── artifacts/           ← Compiled contracts (auto-generated)
├── cache/               ← Hardhat cache (auto-generated)
├── .env                 ← Secret keys (NEVER commit!)
├── .gitignore
├── hardhat.config.js
└── package.json
```

### 1.3 hardhat.config.js — Full Professional Config

```javascript
require("@nomicfoundation/hardhat-toolbox")
require("dotenv").config()

const ALCHEMY_SEPOLIA_URL = process.env.ALCHEMY_SEPOLIA_URL || ""
const PRIVATE_KEY = process.env.PRIVATE_KEY || "0x" + "0".repeat(64)
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY || ""

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,  // 200 = optimize for runtime calls (not deployment)
      },
      viaIR: true, // new compiler pipeline, better optimization
    },
  },
  
  networks: {
    // Local development
    hardhat: {
      chainId: 31337,
      // Fork mainnet for testing (comment out if not needed)
      // forking: {
      //   url: process.env.ALCHEMY_MAINNET_URL,
      //   blockNumber: 19000000,
      // }
    },
    
    localhost: {
      url: "http://127.0.0.1:8545",
      chainId: 31337,
    },
    
    // Sepolia testnet
    sepolia: {
      url: ALCHEMY_SEPOLIA_URL,
      accounts: PRIVATE_KEY ? [PRIVATE_KEY] : [],
      chainId: 11155111,
      gasMultiplier: 1.2, // add 20% buffer to gas estimates
    },
    
    // Polygon Mumbai
    mumbai: {
      url: process.env.POLYGON_MUMBAI_URL || "",
      accounts: PRIVATE_KEY ? [PRIVATE_KEY] : [],
      chainId: 80001,
    },
  },
  
  // Etherscan verification
  etherscan: {
    apiKey: {
      sepolia: ETHERSCAN_API_KEY,
      mainnet: ETHERSCAN_API_KEY,
    },
  },
  
  // Gas reporting
  gasReporter: {
    enabled: process.env.REPORT_GAS === "true",
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
    token: "ETH",
  },
  
  // Code coverage
  coverage: {
    enabled: true,
  },
}
```

### 1.4 .env File

```bash
# .env - NEVER commit to git!
ALCHEMY_SEPOLIA_URL=https://eth-sepolia.g.alchemy.com/v2/YOUR_KEY
ALCHEMY_MAINNET_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
PRIVATE_KEY=your_wallet_private_key_without_0x_prefix
ETHERSCAN_API_KEY=your_etherscan_api_key
REPORT_GAS=true
COINMARKETCAP_API_KEY=optional_for_usd_gas_costs
```

**Add to .gitignore:**
```
.env
artifacts/
cache/
coverage/
```

---

## Part 2: Write, Test, Deploy an ERC-20 (4 hours)

### 2.1 The Contract

```solidity
// contracts/DevToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/// @title DevToken — A production-quality ERC-20 with burnable, permit, and pausable
/// @notice Used as example for Week 1 deployment exercise
contract DevToken is ERC20, ERC20Burnable, ERC20Permit, Ownable, Pausable {
    
    uint256 public constant MAX_SUPPLY = 1_000_000 * 10**18; // 1 million tokens
    
    error MaxSupplyExceeded(uint256 currentSupply, uint256 mintAmount, uint256 maxSupply);
    
    event MintCapReached(uint256 totalSupply);
    
    constructor(
        string memory name,
        string memory symbol,
        uint256 initialSupply,
        address initialOwner
    ) 
        ERC20(name, symbol) 
        ERC20Permit(name)
        Ownable(initialOwner) 
    {
        if (initialSupply > MAX_SUPPLY) {
            revert MaxSupplyExceeded(0, initialSupply, MAX_SUPPLY);
        }
        _mint(initialOwner, initialSupply);
    }
    
    /// @notice Mint new tokens (owner only, respects MAX_SUPPLY)
    function mint(address to, uint256 amount) external onlyOwner {
        uint256 currentSupply = totalSupply();
        if (currentSupply + amount > MAX_SUPPLY) {
            revert MaxSupplyExceeded(currentSupply, amount, MAX_SUPPLY);
        }
        _mint(to, amount);
        
        if (totalSupply() == MAX_SUPPLY) {
            emit MintCapReached(MAX_SUPPLY);
        }
    }
    
    /// @notice Pause all transfers
    function pause() external onlyOwner { _pause(); }
    
    /// @notice Unpause transfers
    function unpause() external onlyOwner { _unpause(); }
    
    /// @notice Override _update to add pause check
    function _update(address from, address to, uint256 value) 
        internal 
        override 
        whenNotPaused 
    {
        super._update(from, to, value);
    }
}
```

### 2.2 Comprehensive Tests

```javascript
// test/DevToken.test.js
const { expect } = require("chai")
const { ethers } = require("hardhat")
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers")

describe("DevToken", function () {
  
  async function deployTokenFixture() {
    const [owner, addr1, addr2, addr3] = await ethers.getSigners()
    
    const initialSupply = ethers.parseEther("100000") // 100k tokens
    const DevToken = await ethers.getContractFactory("DevToken")
    const token = await DevToken.deploy(
      "Dev Token",
      "DEV",
      initialSupply,
      owner.address
    )
    
    return { token, owner, addr1, addr2, addr3, initialSupply }
  }
  
  describe("Deployment", function () {
    it("should set correct name and symbol", async function () {
      const { token } = await loadFixture(deployTokenFixture)
      expect(await token.name()).to.equal("Dev Token")
      expect(await token.symbol()).to.equal("DEV")
    })
    
    it("should mint initial supply to owner", async function () {
      const { token, owner, initialSupply } = await loadFixture(deployTokenFixture)
      expect(await token.balanceOf(owner.address)).to.equal(initialSupply)
      expect(await token.totalSupply()).to.equal(initialSupply)
    })
    
    it("should set correct MAX_SUPPLY", async function () {
      const { token } = await loadFixture(deployTokenFixture)
      expect(await token.MAX_SUPPLY()).to.equal(ethers.parseEther("1000000"))
    })
  })
  
  describe("Minting", function () {
    it("should allow owner to mint", async function () {
      const { token, owner, addr1 } = await loadFixture(deployTokenFixture)
      const mintAmount = ethers.parseEther("50000")
      
      await expect(token.mint(addr1.address, mintAmount))
        .to.emit(token, "Transfer")
        .withArgs(ethers.ZeroAddress, addr1.address, mintAmount)
      
      expect(await token.balanceOf(addr1.address)).to.equal(mintAmount)
    })
    
    it("should revert when non-owner tries to mint", async function () {
      const { token, addr1 } = await loadFixture(deployTokenFixture)
      
      await expect(
        token.connect(addr1).mint(addr1.address, ethers.parseEther("1000"))
      ).to.be.revertedWithCustomError(token, "OwnableUnauthorizedAccount")
    })
    
    it("should revert when minting exceeds MAX_SUPPLY", async function () {
      const { token, owner, addr1 } = await loadFixture(deployTokenFixture)
      const overMax = ethers.parseEther("1000000") // would exceed max
      
      await expect(
        token.mint(addr1.address, overMax)
      ).to.be.revertedWithCustomError(token, "MaxSupplyExceeded")
    })
    
    it("should emit MintCapReached when max supply hit", async function () {
      const { token, owner, addr1, initialSupply } = await loadFixture(deployTokenFixture)
      const remaining = ethers.parseEther("1000000") - initialSupply
      
      await expect(token.mint(addr1.address, remaining))
        .to.emit(token, "MintCapReached")
    })
  })
  
  describe("Transfers", function () {
    it("should transfer tokens between accounts", async function () {
      const { token, owner, addr1 } = await loadFixture(deployTokenFixture)
      const amount = ethers.parseEther("100")
      
      await token.transfer(addr1.address, amount)
      expect(await token.balanceOf(addr1.address)).to.equal(amount)
    })
    
    it("should fail when paused", async function () {
      const { token, owner, addr1 } = await loadFixture(deployTokenFixture)
      
      await token.pause()
      
      await expect(
        token.transfer(addr1.address, ethers.parseEther("100"))
      ).to.be.revertedWithCustomError(token, "EnforcedPause")
    })
    
    it("should allow transfers when unpaused", async function () {
      const { token, owner, addr1 } = await loadFixture(deployTokenFixture)
      await token.pause()
      await token.unpause()
      
      await expect(
        token.transfer(addr1.address, ethers.parseEther("100"))
      ).to.not.be.reverted
    })
  })
  
  describe("Burning", function () {
    it("should allow token holders to burn their tokens", async function () {
      const { token, owner } = await loadFixture(deployTokenFixture)
      const burnAmount = ethers.parseEther("1000")
      const supplyBefore = await token.totalSupply()
      
      await token.burn(burnAmount)
      
      expect(await token.totalSupply()).to.equal(supplyBefore - burnAmount)
    })
  })
  
  describe("Gas Usage", function () {
    it("should report gas for transfer", async function () {
      const { token, owner, addr1 } = await loadFixture(deployTokenFixture)
      const tx = await token.transfer(addr1.address, ethers.parseEther("100"))
      const receipt = await tx.wait()
      console.log("Transfer gas used:", receipt.gasUsed.toString())
    })
  })
})
```

### 2.3 Deployment Script

```javascript
// scripts/deploy.js
const { ethers, run } = require("hardhat")

async function main() {
  console.log("Deploying DevToken...")
  
  const [deployer] = await ethers.getSigners()
  console.log("Deploying with account:", deployer.address)
  console.log("Balance:", ethers.formatEther(await ethers.provider.getBalance(deployer.address)), "ETH")
  
  const initialSupply = ethers.parseEther("100000") // 100k tokens
  
  const DevToken = await ethers.getContractFactory("DevToken")
  const token = await DevToken.deploy(
    "Dev Token",
    "DEV",
    initialSupply,
    deployer.address
  )
  
  await token.waitForDeployment()
  
  const address = await token.getAddress()
  console.log("DevToken deployed to:", address)
  console.log("Etherscan:", `https://sepolia.etherscan.io/address/${address}`)
  
  // Wait for a few blocks before verifying
  console.log("Waiting 5 blocks for Etherscan to index...")
  await token.deploymentTransaction().wait(5)
  
  // Verify on Etherscan
  console.log("Verifying contract on Etherscan...")
  try {
    await run("verify:verify", {
      address: address,
      constructorArguments: [
        "Dev Token",
        "DEV",
        initialSupply.toString(),
        deployer.address,
      ],
    })
    console.log("Contract verified!")
  } catch (e) {
    if (e.message.toLowerCase().includes("already verified")) {
      console.log("Already verified!")
    } else {
      console.error("Verification error:", e.message)
    }
  }
  
  // Save deployment info
  const deployInfo = {
    network: "sepolia",
    address: address,
    deployer: deployer.address,
    deployedAt: new Date().toISOString(),
    txHash: token.deploymentTransaction().hash,
  }
  
  const fs = require("fs")
  fs.writeFileSync("./deployment.json", JSON.stringify(deployInfo, null, 2))
  console.log("Deployment info saved to deployment.json")
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })
```

### 2.4 Run Everything

```bash
# Compile contracts
npx hardhat compile

# Run tests on local network
npx hardhat test

# Run tests with gas report
REPORT_GAS=true npx hardhat test

# Run test coverage
npx hardhat coverage

# Deploy to local node
npx hardhat node                          # Terminal 1
npx hardhat run scripts/deploy.js --network localhost  # Terminal 2

# Deploy to Sepolia
npx hardhat run scripts/deploy.js --network sepolia

# Verify manually (if auto-verify failed)
npx hardhat verify --network sepolia CONTRACT_ADDRESS "Dev Token" "DEV" "100000000000000000000000" "OWNER_ADDRESS"
```

---

## Part 3: Hardhat Useful Commands (1 hour)

### Console — Interact with Deployed Contract

```bash
# Start a console connected to your network
npx hardhat console --network sepolia

# Inside the console:
const [deployer] = await ethers.getSigners()
const token = await ethers.getContractAt("DevToken", "0xYOUR_CONTRACT_ADDRESS")
const balance = await token.balanceOf(deployer.address)
console.log(ethers.formatEther(balance))
await token.transfer("0xRecipient", ethers.parseEther("100"))
```

### Forking Mainnet Locally

```javascript
// In hardhat.config.js — enable forking:
hardhat: {
  forking: {
    url: process.env.ALCHEMY_MAINNET_URL,
    blockNumber: 19000000, // pin to specific block for reproducibility
  }
}
```

```bash
# Now run tests against a fork of mainnet state!
# You have access to all mainnet contracts, tokens, prices
npx hardhat test
```

This lets you test integrations with Uniswap, Aave, etc., using real mainnet state — without spending real ETH.

---

## Part 4: Foundry Alternative (Optional but Recommended)

Foundry is the professional alternative to Hardhat — faster tests, written in Solidity.

```bash
# Install Foundry
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Initialize new Foundry project
forge init my-foundry-project
cd my-foundry-project

# Install OpenZeppelin
forge install OpenZeppelin/openzeppelin-contracts

# Run tests
forge test

# Run tests with gas report
forge test --gas-report

# Deploy
forge script script/Deploy.s.sol --rpc-url $SEPOLIA_RPC --broadcast --verify
```

**Foundry test example:**
```solidity
// test/DevToken.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/DevToken.sol";

contract DevTokenTest is Test {
    DevToken token;
    address owner = makeAddr("owner");
    address alice = makeAddr("alice");
    
    function setUp() public {
        vm.prank(owner);
        token = new DevToken("Dev Token", "DEV", 100_000e18, owner);
    }
    
    function test_InitialSupply() public {
        assertEq(token.balanceOf(owner), 100_000e18);
    }
    
    function test_MintByOwner() public {
        vm.prank(owner);
        token.mint(alice, 1000e18);
        assertEq(token.balanceOf(alice), 1000e18);
    }
    
    function test_RevertMintByNonOwner() public {
        vm.prank(alice);
        vm.expectRevert();
        token.mint(alice, 1000e18);
    }
    
    // Fuzz test — forge generates 256 random inputs automatically!
    function testFuzz_Transfer(uint256 amount) public {
        amount = bound(amount, 1, token.balanceOf(owner));
        vm.prank(owner);
        token.transfer(alice, amount);
        assertEq(token.balanceOf(alice), amount);
    }
}
```

---

## Resources for Day 5

| Resource | Link | Type |
|----------|------|------|
| Hardhat Docs | https://hardhat.org/docs | Docs |
| Hardhat Ignition | https://hardhat.org/ignition/docs/getting-started | Docs |
| Foundry Book | https://book.getfoundry.sh/ | Docs |
| OpenZeppelin Contracts | https://docs.openzeppelin.com/contracts/5.x/ | Docs |
| Alchemy Dashboard | https://dashboard.alchemy.com/ | Service |
| Etherscan Verify | https://etherscan.io/verifyContract | Tool |
| Sepolia Faucet | https://sepoliafaucet.com | Faucet |
| Hardhat Network Helpers | https://hardhat.org/hardhat-network-helpers/docs/overview | Docs |

---

## Tomorrow

[Day 6 → ERC-20 Token Standard Deep Dive](./Day-6-ERC20-Token-Standard.md)
