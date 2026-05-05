# Day 7 — Mini Project: Launch Your Own ERC-20 Token

> **Time:** 6–8 hours  
> **Goal:** Build, test, deploy, and verify a real ERC-20 token with a complete frontend. This is your Week 1 capstone.

---

## Project: "LaunchPad Token" (LPT)

**Description:** A production-quality governance token with:
- 1,000,000 max supply
- Owner-controlled minting
- Burning capability
- Gasless approvals (EIP-2612 Permit)
- Transfer pause during launch period
- Airdrop function (batch mint to multiple addresses)
- Simple frontend to view balances and send tokens

---

## Step 1: Contract (2 hours)

### contracts/LaunchPadToken.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

contract LaunchPadToken is ERC20, ERC20Burnable, ERC20Permit, Ownable, Pausable {

    uint256 public constant MAX_SUPPLY = 1_000_000 * 10**18;
    
    bool public launchMode = true; // restricted during launch
    uint256 public transferDelay;  // min blocks between transfers per address
    mapping(address => uint256) public lastTransferBlock;
    
    error MaxSupplyExceeded(uint256 requested, uint256 available);
    error LaunchModeRestriction();
    error TransferDelay(uint256 blocksLeft);
    error AirdropArrayMismatch();

    event LaunchModeDisabled();
    event AirdropCompleted(uint256 recipientCount, uint256 totalAmount);

    constructor(
        address initialOwner,
        uint256 transferDelayBlocks
    )
        ERC20("LaunchPad Token", "LPT")
        ERC20Permit("LaunchPad Token")
        Ownable(initialOwner)
    {
        transferDelay = transferDelayBlocks;
        // Mint 200k to deployer (founder allocation)
        _mint(initialOwner, 200_000 * 10**18);
    }

    /// @notice Mint tokens to a single address
    function mint(address to, uint256 amount) external onlyOwner {
        uint256 available = MAX_SUPPLY - totalSupply();
        if (amount > available) revert MaxSupplyExceeded(amount, available);
        _mint(to, amount);
    }

    /// @notice Airdrop tokens to multiple addresses in one tx
    function airdrop(address[] calldata recipients, uint256[] calldata amounts) external onlyOwner {
        if (recipients.length != amounts.length) revert AirdropArrayMismatch();
        
        uint256 total;
        for (uint256 i = 0; i < amounts.length;) {
            total += amounts[i];
            unchecked { ++i; }
        }
        
        uint256 available = MAX_SUPPLY - totalSupply();
        if (total > available) revert MaxSupplyExceeded(total, available);
        
        for (uint256 i = 0; i < recipients.length;) {
            _mint(recipients[i], amounts[i]);
            unchecked { ++i; }
        }
        
        emit AirdropCompleted(recipients.length, total);
    }

    /// @notice Disable launch mode (cannot be re-enabled)
    function disableLaunchMode() external onlyOwner {
        launchMode = false;
        emit LaunchModeDisabled();
    }

    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }

    function _update(address from, address to, uint256 value)
        internal
        override
        whenNotPaused
    {
        // During launch mode: only owner can transfer
        if (launchMode && from != address(0) && from != owner()) {
            revert LaunchModeRestriction();
        }
        
        // Transfer delay: prevents bot sniping
        if (transferDelay > 0 && from != address(0)) {
            uint256 blocksLeft = lastTransferBlock[from] + transferDelay;
            if (block.number < blocksLeft) {
                revert TransferDelay(blocksLeft - block.number);
            }
            lastTransferBlock[from] = block.number;
        }
        
        super._update(from, to, value);
    }
}
```

---

## Step 2: Tests (1.5 hours)

```javascript
// test/LaunchPadToken.test.js
const { expect } = require("chai")
const { ethers } = require("hardhat")
const { loadFixture, mine } = require("@nomicfoundation/hardhat-toolbox/network-helpers")

describe("LaunchPadToken", function () {
  async function deployFixture() {
    const [owner, alice, bob, carol] = await ethers.getSigners()
    
    const LPT = await ethers.getContractFactory("LaunchPadToken")
    const token = await LPT.deploy(owner.address, 5) // 5 block transfer delay
    
    return { token, owner, alice, bob, carol }
  }
  
  it("mints founder allocation to owner", async function () {
    const { token, owner } = await loadFixture(deployFixture)
    expect(await token.balanceOf(owner.address)).to.equal(ethers.parseEther("200000"))
  })
  
  it("blocks non-owner transfers in launch mode", async function () {
    const { token, owner, alice } = await loadFixture(deployFixture)
    await token.mint(alice.address, ethers.parseEther("1000"))
    
    await expect(
      token.connect(alice).transfer(owner.address, ethers.parseEther("100"))
    ).to.be.revertedWithCustomError(token, "LaunchModeRestriction")
  })
  
  it("allows transfers after launch mode disabled", async function () {
    const { token, owner, alice } = await loadFixture(deployFixture)
    await token.mint(alice.address, ethers.parseEther("1000"))
    await token.disableLaunchMode()
    
    await mine(5) // pass transfer delay
    await expect(
      token.connect(alice).transfer(owner.address, ethers.parseEther("100"))
    ).to.not.be.reverted
  })
  
  it("enforces transfer delay", async function () {
    const { token, owner, alice } = await loadFixture(deployFixture)
    await token.disableLaunchMode()
    
    // First transfer: OK
    await mine(5)
    await token.transfer(alice.address, ethers.parseEther("100"))
    
    // Second transfer immediately: fails
    await expect(
      token.transfer(alice.address, ethers.parseEther("100"))
    ).to.be.revertedWithCustomError(token, "TransferDelay")
    
    // After delay: OK
    await mine(5)
    await expect(
      token.transfer(alice.address, ethers.parseEther("100"))
    ).to.not.be.reverted
  })
  
  it("airdrops to multiple recipients", async function () {
    const { token, owner, alice, bob, carol } = await loadFixture(deployFixture)
    
    const recipients = [alice.address, bob.address, carol.address]
    const amounts = [
      ethers.parseEther("1000"),
      ethers.parseEther("2000"),
      ethers.parseEther("3000"),
    ]
    
    await expect(token.airdrop(recipients, amounts))
      .to.emit(token, "AirdropCompleted")
      .withArgs(3, ethers.parseEther("6000"))
    
    expect(await token.balanceOf(alice.address)).to.equal(amounts[0])
    expect(await token.balanceOf(bob.address)).to.equal(amounts[1])
    expect(await token.balanceOf(carol.address)).to.equal(amounts[2])
  })
})
```

---

## Step 3: Deploy Script (30 min)

```javascript
// scripts/deployLPT.js
const { ethers, run } = require("hardhat")
const fs = require("fs")

async function main() {
  const [deployer] = await ethers.getSigners()
  
  console.log("Deployer:", deployer.address)
  console.log("Balance:", ethers.formatEther(await ethers.provider.getBalance(deployer.address)))
  
  const LPT = await ethers.getContractFactory("LaunchPadToken")
  
  // 5 block transfer delay (about 1 minute on Ethereum)
  const token = await LPT.deploy(deployer.address, 5)
  await token.waitForDeployment()
  
  const address = await token.getAddress()
  console.log("\nLaunchPadToken deployed:", address)
  console.log("Etherscan:", `https://sepolia.etherscan.io/address/${address}`)
  
  // Save deployment
  const info = {
    network: "sepolia",
    address,
    deployer: deployer.address,
    timestamp: new Date().toISOString(),
    txHash: token.deploymentTransaction().hash,
  }
  fs.writeFileSync("deployment-lpt.json", JSON.stringify(info, null, 2))
  
  // Wait and verify
  await token.deploymentTransaction().wait(5)
  
  try {
    await run("verify:verify", {
      address,
      constructorArguments: [deployer.address, 5],
    })
  } catch (e) {
    console.log("Verify error:", e.message)
  }
}

main().catch(console.error)
```

---

## Step 4: Frontend (2 hours)

### frontend/index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>LaunchPad Token</title>
  <style>
    body { font-family: monospace; max-width: 600px; margin: 50px auto; padding: 20px; background: #0d0d0d; color: #00ff88; }
    h1 { border-bottom: 1px solid #00ff88; padding-bottom: 10px; }
    .card { background: #1a1a1a; border: 1px solid #333; padding: 20px; margin: 20px 0; border-radius: 8px; }
    input, button { background: #222; color: #00ff88; border: 1px solid #444; padding: 8px 12px; border-radius: 4px; font-family: monospace; }
    button { cursor: pointer; } button:hover { background: #333; }
    .error { color: #ff4444; } .success { color: #00ff88; }
    #status { padding: 10px; margin: 10px 0; }
  </style>
</head>
<body>
  <h1>LaunchPad Token (LPT)</h1>
  
  <div class="card">
    <button id="connectBtn">Connect Wallet</button>
    <div id="walletInfo"></div>
  </div>
  
  <div class="card">
    <h3>Token Info</h3>
    <div id="tokenInfo">Not connected</div>
  </div>
  
  <div class="card">
    <h3>Transfer Tokens</h3>
    <input type="text" id="transferTo" placeholder="Recipient address (0x...)" style="width:100%; margin-bottom:8px">
    <input type="number" id="transferAmount" placeholder="Amount" style="width:100%; margin-bottom:8px">
    <button id="transferBtn">Send LPT</button>
  </div>
  
  <div id="status"></div>
  
  <script src="https://cdn.ethers.io/lib/ethers-5.7.2.umd.min.js"></script>
  <script>
    // Paste your deployed contract address here
    const TOKEN_ADDRESS = "YOUR_DEPLOYED_ADDRESS"
    
    const TOKEN_ABI = [
      "function name() view returns (string)",
      "function symbol() view returns (string)",
      "function totalSupply() view returns (uint256)",
      "function balanceOf(address) view returns (uint256)",
      "function transfer(address to, uint256 amount) returns (bool)",
      "function launchMode() view returns (bool)",
      "event Transfer(address indexed from, address indexed to, uint256 value)"
    ]
    
    let provider, signer, contract
    
    function setStatus(msg, type = "success") {
      document.getElementById("status").innerHTML = 
        `<div class="${type}">${msg}</div>`
    }
    
    async function connectWallet() {
      if (!window.ethereum) {
        setStatus("MetaMask not found! Install it at metamask.io", "error")
        return
      }
      
      provider = new ethers.providers.Web3Provider(window.ethereum)
      await provider.send("eth_requestAccounts", [])
      signer = provider.getSigner()
      
      const address = await signer.getAddress()
      const network = await provider.getNetwork()
      
      document.getElementById("walletInfo").innerHTML = `
        <div>Connected: <strong>${address.slice(0, 6)}...${address.slice(-4)}</strong></div>
        <div>Network: <strong>${network.name} (${network.chainId})</strong></div>
      `
      document.getElementById("connectBtn").textContent = "Connected ✓"
      
      contract = new ethers.Contract(TOKEN_ADDRESS, TOKEN_ABI, signer)
      
      await loadTokenInfo()
      listenToTransfers()
    }
    
    async function loadTokenInfo() {
      const [name, symbol, supply, balance, launchMode] = await Promise.all([
        contract.name(),
        contract.symbol(),
        contract.totalSupply(),
        contract.balanceOf(await signer.getAddress()),
        contract.launchMode(),
      ])
      
      document.getElementById("tokenInfo").innerHTML = `
        <div>Name: <strong>${name}</strong></div>
        <div>Symbol: <strong>${symbol}</strong></div>
        <div>Total Supply: <strong>${ethers.utils.formatEther(supply)} LPT</strong></div>
        <div>Your Balance: <strong>${ethers.utils.formatEther(balance)} LPT</strong></div>
        <div>Launch Mode: <strong>${launchMode ? "⚠️ Active (transfers restricted)" : "✅ Disabled"}</strong></div>
      `
    }
    
    async function transferTokens() {
      const to = document.getElementById("transferTo").value
      const amount = document.getElementById("transferAmount").value
      
      if (!to || !amount) { setStatus("Fill in all fields", "error"); return }
      
      try {
        setStatus("Sending transaction...")
        const tx = await contract.transfer(to, ethers.utils.parseEther(amount))
        setStatus(`Transaction sent: ${tx.hash.slice(0, 10)}... waiting for confirmation...`)
        
        const receipt = await tx.wait()
        setStatus(`✅ Transfer confirmed! Block: ${receipt.blockNumber}, Gas: ${receipt.gasUsed.toString()}`)
        
        await loadTokenInfo()
      } catch (e) {
        setStatus(`Error: ${e.reason || e.message}`, "error")
      }
    }
    
    function listenToTransfers() {
      const myAddress = signer.getAddress()
      
      contract.on("Transfer", async (from, to, value) => {
        const me = await myAddress
        if (from.toLowerCase() === me.toLowerCase() || to.toLowerCase() === me.toLowerCase()) {
          const direction = from.toLowerCase() === me.toLowerCase() ? "Sent" : "Received"
          setStatus(`${direction} ${ethers.utils.formatEther(value)} LPT`)
          await loadTokenInfo()
        }
      })
    }
    
    document.getElementById("connectBtn").addEventListener("click", connectWallet)
    document.getElementById("transferBtn").addEventListener("click", transferTokens)
  </script>
</body>
</html>
```

---

## Step 5: Checklist

Before calling this complete, verify:

- [ ] Contract compiles without warnings
- [ ] All tests pass (100% pass rate)
- [ ] Deployed to Sepolia testnet
- [ ] Verified on Etherscan (source code visible)
- [ ] Frontend connects to MetaMask
- [ ] Frontend shows correct balance
- [ ] Transfer function works
- [ ] README created with contract address

---

## Bonus Challenges

1. **Add a token lock:** Implement `lockTokens(amount, unlockTime)` that holds tokens until a future timestamp
2. **Add snapshot:** Use OpenZeppelin's ERC20Snapshot to record balances at a specific block (for airdrops based on holdings)
3. **Add a faucet:** Anyone can request 100 tokens once every 24 hours
4. **Improve frontend:** Show transaction history (query past Transfer events)

---

## Week 1 Complete!

**What you built this week:**
- Deep understanding of how blockchains work cryptographically
- Transaction lifecycle from wallet to finality
- PoW vs PoS consensus mechanisms
- Production-quality Solidity code
- Professional Hardhat development environment
- Fully deployed and verified ERC-20 token

**Next:** [Week 2 → DeFi & AMMs](../Week-2-DeFi-and-AMMs/README.md)
