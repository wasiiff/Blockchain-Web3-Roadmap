# Day 25 (Week 4, Day 4) — Oracles & Chainlink

> **Time:** 6–8 hours  
> **Goal:** Understand the oracle problem. Use Chainlink Price Feeds, VRF, and Automation in production contracts.

---

## Part 1: The Oracle Problem (1 hour)

### Why Oracles Are Needed

Smart contracts are deterministic and sandboxed — they cannot access real-world data:
- ETH/USD price
- Sports results
- Random numbers
- Weather data
- Stock prices

An oracle is a service that **bridges real-world data onto the blockchain.**

### The Trust Problem

If one entity provides data, they can manipulate it. Solutions:
1. **Decentralized oracle networks** (DON) — many independent nodes, aggregated answer
2. **Reputation and staking** — oracle nodes stake tokens, slashed for bad data
3. **Median aggregation** — outliers filtered by taking the median

---

## Part 2: Chainlink Price Feeds (2 hours)

### 2.1 How Chainlink Works

```
Real-world price data
        │
        ▼
Chainlink Oracle Nodes (many independent nodes)
Each node:
  - Fetches price from multiple API sources
  - Aggregates internally
  - Signs and submits on-chain
        │
        ▼
Aggregator Contract (on-chain)
  - Receives responses from many nodes
  - Takes median (removes outliers)
  - Updates if deviation > threshold (e.g., 0.5%)
  - Updates at minimum once per hour (heartbeat)
        │
        ▼
Your Contract reads: latestRoundData()
```

### 2.2 Using Price Feeds

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceFeedConsumer {
    AggregatorV3Interface internal dataFeed;
    
    // ETH/USD on Sepolia: 0x694AA1769357215DE4FAC081bf1f309aDC325306
    // BTC/USD on Sepolia: 0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43
    // LINK/USD on Sepolia: 0xc59E3633BAAC79493d908e63626716e204A45EdF
    
    constructor(address priceFeedAddress) {
        dataFeed = AggregatorV3Interface(priceFeedAddress);
    }
    
    function getLatestPrice() public view returns (int256 price, uint256 timestamp) {
        (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = dataFeed.latestRoundData();
        
        // IMPORTANT: validate oracle data!
        require(updatedAt > 0, "Round not complete");
        require(answer > 0, "Negative price");
        require(block.timestamp - updatedAt <= 3600, "Stale price"); // 1 hour staleness check
        require(answeredInRound >= roundId, "Stale round");
        
        return (answer, updatedAt);
    }
    
    function getDecimals() external view returns (uint8) {
        return dataFeed.decimals(); // typically 8 for price feeds
    }
    
    function getPriceInUSD() external view returns (uint256) {
        (int256 price,) = getLatestPrice();
        uint8 decimals = dataFeed.decimals();
        // Convert to 18 decimal standard
        return uint256(price) * 10**(18 - decimals);
    }
}
```

### 2.3 Price-Denominated Contract (Real Example)

```solidity
/// @title ETHSubscription — Pay in ETH for a USD-denominated subscription
contract ETHSubscription {
    AggregatorV3Interface private ethUsdFeed;
    
    uint256 public constant SUBSCRIPTION_PRICE_USD = 10 * 10**8; // $10 in 8 decimals (Chainlink standard)
    
    mapping(address => uint256) public subscriptionExpiry;
    
    constructor(address _ethUsdFeed) {
        ethUsdFeed = AggregatorV3Interface(_ethUsdFeed);
    }
    
    /// @notice Subscribe by paying equivalent of $10 in ETH
    function subscribe() external payable {
        uint256 requiredETH = getRequiredETH();
        require(msg.value >= requiredETH, "Insufficient ETH");
        
        // Extend subscription by 30 days
        uint256 start = block.timestamp > subscriptionExpiry[msg.sender] 
            ? block.timestamp 
            : subscriptionExpiry[msg.sender];
        subscriptionExpiry[msg.sender] = start + 30 days;
        
        // Refund excess
        if (msg.value > requiredETH) {
            payable(msg.sender).transfer(msg.value - requiredETH);
        }
    }
    
    function getRequiredETH() public view returns (uint256) {
        (,int256 ethUsdPrice,,,) = ethUsdFeed.latestRoundData();
        require(ethUsdPrice > 0, "Invalid price");
        
        // requiredETH = (subscriptionPriceUSD / ethUsdPrice) × 10^18
        // Both in 8 decimals, cancel out, result in wei (18 decimals)
        return (SUBSCRIPTION_PRICE_USD * 10**18) / uint256(ethUsdPrice);
    }
    
    function isSubscribed(address user) external view returns (bool) {
        return subscriptionExpiry[user] > block.timestamp;
    }
}
```

---

## Part 3: Chainlink VRF v2.5 — Verifiable Randomness (2 hours)

### 3.1 The Problem with On-Chain Randomness

```solidity
// NEVER use this for randomness — attackers can manipulate:
uint256 bad1 = block.timestamp % 10;   // validator can manipulate timestamp
uint256 bad2 = block.prevrandao % 10;  // better but still manipulable by validators
uint256 bad3 = uint256(blockhash(block.number - 1)) % 10; // only for past blocks, manipulable
```

**Chainlink VRF:** Off-chain random number generation with on-chain verification proof.

```
1. Your contract requests randomness → emits event
2. Chainlink VRF node observes event
3. Node generates random number + cryptographic proof
4. Node submits both to your contract
5. Your contract verifies proof on-chain (mathematically guaranteed)
6. Uses verified random number
```

### 3.2 VRF Implementation (Subscription Method)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@chainlink/contracts/src/v0.8/vrf/interfaces/VRFCoordinatorV2Interface.sol";
import "@chainlink/contracts/src/v0.8/vrf/VRFConsumerBaseV2.sol";

/// @title VRFRaffle — Fair raffle using Chainlink VRF
contract VRFRaffle is VRFConsumerBaseV2 {
    VRFCoordinatorV2Interface private vrfCoordinator;
    
    // Sepolia VRF Coordinator: 0x8103B0A8A00be2DDC778e6e7eaa21791Cd364625
    // Key hash (Sepolia 150 gwei): 0x474e34a077df58807dbe9c96d3c009b23b3c6d0cce433e59bbf5b34f823bc56c
    
    uint64 private subscriptionId;
    bytes32 private keyHash;
    uint32 private callbackGasLimit = 100000;
    uint16 private requestConfirmations = 3;
    uint32 private numWords = 1;
    
    address[] public players;
    mapping(uint256 => address) public requestToSender;
    
    uint256 public ticketPrice = 0.01 ether;
    bool public raffleOpen;
    address public lastWinner;
    
    event RaffleEntered(address indexed player);
    event RandomnessRequested(uint256 indexed requestId);
    event WinnerSelected(address indexed winner, uint256 amount);
    
    constructor(
        address _vrfCoordinator,
        uint64 _subscriptionId,
        bytes32 _keyHash
    ) VRFConsumerBaseV2(_vrfCoordinator) {
        vrfCoordinator = VRFCoordinatorV2Interface(_vrfCoordinator);
        subscriptionId = _subscriptionId;
        keyHash = _keyHash;
        raffleOpen = true;
    }
    
    /// @notice Enter the raffle
    function enterRaffle() external payable {
        require(raffleOpen, "Raffle closed");
        require(msg.value >= ticketPrice, "Insufficient ticket price");
        players.push(msg.sender);
        emit RaffleEntered(msg.sender);
    }
    
    /// @notice Request random winner (owner calls this to end raffle)
    function pickWinner() external {
        require(msg.sender == owner, "Not owner");
        require(players.length > 0, "No players");
        
        raffleOpen = false;
        
        uint256 requestId = vrfCoordinator.requestRandomWords(
            keyHash,
            subscriptionId,
            requestConfirmations,
            callbackGasLimit,
            numWords
        );
        
        emit RandomnessRequested(requestId);
    }
    
    /// @notice Called by Chainlink VRF with the random number
    function fulfillRandomWords(
        uint256 requestId,
        uint256[] memory randomWords
    ) internal override {
        uint256 randomIndex = randomWords[0] % players.length;
        address winner = players[randomIndex];
        lastWinner = winner;
        
        // Transfer prize pool to winner
        uint256 prize = address(this).balance;
        (bool success,) = winner.call{value: prize}("");
        require(success, "Transfer failed");
        
        // Reset
        delete players;
        raffleOpen = true;
        
        emit WinnerSelected(winner, prize);
    }
    
    // Needed for parent contract
    address public owner = msg.sender;
    
    function getPlayers() external view returns (address[] memory) {
        return players;
    }
}
```

---

## Part 4: Chainlink Automation (30 min)

Chainlink Automation (formerly Keepers) automatically calls your contract functions based on conditions.

```solidity
import "@chainlink/contracts/src/v0.8/automation/AutomationCompatible.sol";

/// @title AutomatedVault — Automatically rebalances portfolio when out of balance
contract AutomatedVault is AutomationCompatibleInterface {
    uint256 public threshold = 10; // rebalance if drift > 10%
    
    /// @notice Chainlink checks this — should we perform upkeep?
    function checkUpkeep(bytes calldata) 
        external view override returns (bool upkeepNeeded, bytes memory performData) 
    {
        uint256 drift = calculateDrift(); // how far from target allocation
        upkeepNeeded = drift > threshold;
        performData = abi.encode(drift);
    }
    
    /// @notice Chainlink calls this if checkUpkeep returns true
    function performUpkeep(bytes calldata performData) external override {
        uint256 drift = abi.decode(performData, (uint256));
        require(drift > threshold, "No rebalancing needed");
        
        _rebalance(); // your rebalancing logic
    }
    
    function _rebalance() internal {
        // swap tokens back to target allocation
    }
    
    function calculateDrift() public view returns (uint256) {
        // calculate current vs target allocation
        return 0; // placeholder
    }
}
```

---

## Part 5: Chainlink Price Feed Addresses (Reference)

### Mainnet
| Feed | Address |
|------|---------|
| ETH/USD | 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419 |
| BTC/USD | 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c |
| LINK/USD | 0x2c1d072e956AFFC0D435Cb7AC308d97e0995f0 |

### Sepolia Testnet
| Feed | Address |
|------|---------|
| ETH/USD | 0x694AA1769357215DE4FAC081bf1f309aDC325306 |
| BTC/USD | 0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43 |
| LINK/USD | 0xc59E3633BAAC79493d908e63626716e204A45EdF |

Full list: https://docs.chain.link/data-feeds/price-feeds/addresses

---

## Resources for Day 25

| Resource | Link | Type |
|----------|------|------|
| Chainlink Docs | https://docs.chain.link/ | Docs |
| Chainlink VRF | https://docs.chain.link/vrf | Docs |
| Chainlink Automation | https://docs.chain.link/chainlink-automation | Docs |
| Price Feed Addresses | https://docs.chain.link/data-feeds/price-feeds/addresses | Reference |
| Chainlink Faucet (Sepolia) | https://faucets.chain.link/ | Faucet |

---

## Tomorrow

[Day 26 → Capstone Architecture Design](./Day-5-Capstone-Architecture.md)
