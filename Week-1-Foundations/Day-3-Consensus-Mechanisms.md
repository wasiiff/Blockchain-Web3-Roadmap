# Day 3 — Consensus Mechanisms: PoW vs PoS

> **Time:** 6–8 hours  
> **Goal:** Understand how decentralized nodes agree on a single version of truth — and why Ethereum switched from Proof of Work to Proof of Stake.

---

## Part 1: Theory (2.5 hours)

### 1.1 The Byzantine Generals Problem

Before understanding consensus, you need to understand the problem it solves.

**Scenario:** Several generals surround a city. They must agree on a battle plan (attack or retreat). They communicate by messenger. Some generals may be traitors who send contradictory messages.

**The problem:** How do honest generals reach consensus when some participants are dishonest?

This maps to blockchain:
- Generals = nodes on the network
- City = blockchain state
- Traitors = malicious nodes trying to double-spend or fork the chain

**Consensus algorithms solve this.** Byzantine Fault Tolerance (BFT) means the system works as long as fewer than 1/3 of participants are malicious.

---

### 1.2 Proof of Work (PoW) — How Bitcoin & Original Ethereum Worked

**Core idea:** To add a block, you must solve a computationally expensive puzzle. The difficulty of the puzzle adjusts so that the network finds a solution roughly every 10 minutes (Bitcoin) or 12 seconds (Ethereum pre-Merge).

**The Puzzle:**
```
Find a nonce such that:
keccak256(blockData + nonce) < targetDifficulty

Example:
keccak256("block_data_" + 8394729) = 0x000000003a2b1c...  ← starts with enough zeros
keccak256("block_data_" + 8394730) = 0x9fe3201b2a8d7c...  ← doesn't start with zeros
```

The only way to find the nonce is **brute force** — try billions of values per second.

**Mining Analogy:**
Think of a dice game where you roll a million-sided die and need to roll under 100. The only strategy is to roll it over and over. But verifying "you rolled a 47" takes one second.

This asymmetry is critical:
- **Hard to find:** Takes enormous computation (energy/hardware)
- **Easy to verify:** Any node can check in milliseconds

**Security model:**
- To rewrite history, you need >50% of total network hashrate
- With Bitcoin: ~$20 billion in hardware + $10M/day in electricity
- Making attacks economically infeasible

**PoW Problems:**
1. Enormous energy consumption (Bitcoin = ~150 TWh/year, comparable to a medium country)
2. Hardware centralization (ASIC farms dominate)
3. Slow finality (Bitcoin: ~60 minutes for safe finality)
4. No "slashing" — attacking PoW just wastes your electricity

---

### 1.3 Proof of Stake (PoS) — How Ethereum Works Now

Ethereum switched to PoS in September 2022 ("The Merge"). Energy use dropped by ~99.95%.

**Core idea:** Instead of competing with computation, validators lock up ("stake") ETH as collateral. If they behave honestly, they earn rewards. If they cheat or are offline, they get "slashed" (lose some ETH).

**Ethereum PoS numbers:**
- Minimum stake: 32 ETH per validator
- ~1 million validators (as of 2025)
- Total staked: ~32 million ETH (~26% of supply)
- Validator reward: ~3-5% APY

**Validator roles:**
```
Validator A → Proposes a block (chosen randomly, weighted by stake)
Validators B, C, D, ... → Attest (vote) that the block is valid
2/3 of validators must attest → Block is included
```

**Slots and Epochs:**
- **Slot:** 12 seconds — one block proposed
- **Epoch:** 32 slots (6.4 minutes)
- **Checkpoint:** End of each epoch — finality votes happen here
- **Finality:** After 2 epochs (~12.8 minutes), a block is finalized (irreversible)

**Slashing — the game theory:**
If a validator:
- Votes for two different blocks in the same slot (equivocation)
- Votes for a block that conflicts with a finalized block

→ They lose 1/32 of their stake immediately  
→ Forced exit from validator set  
→ 36-day withdrawal period (more time for correlation penalty)

**This is far stronger security than PoW** — attacking Ethereum requires staking $50B+, and you lose it all if caught.

---

### 1.4 The Ethereum Consensus Deep Dive (Gasper)

Ethereum's PoS uses a protocol called **Gasper** = Casper FFG + LMD-GHOST.

**LMD-GHOST (Latest Message Driven Greediest Heaviest Observed SubTree):**
- Used to select the canonical chain (which chain to follow during forks)
- Each validator's most recent vote counts
- Follow the fork with the most validator weight

**Casper FFG (Friendly Finality Gadget):**
- Handles finality — making blocks irreversible
- After 2 epochs, the checkpoint is finalized
- Once finalized, reverting requires destroying at least 1/3 of all staked ETH

**Why two protocols?** LMD-GHOST is fast (fork choice), Casper FFG is safe (finality).

---

### 1.5 Delegated Proof of Stake & Other Variants

| Mechanism | Examples | Notes |
|-----------|---------|-------|
| PoW | Bitcoin, Litecoin | Highest energy use |
| PoS | Ethereum | Best decentralization |
| DPoS | EOS, TRON | Elected "delegates" |
| PoA (Proof of Authority) | Private chains | Centralized but fast |
| PoH (Proof of History) | Solana | Timestamping mechanism |
| BFT variants | Cosmos, Tendermint | Instant finality |

---

### 1.6 Validators, Staking Pools, and Liquid Staking

**Running your own validator:**
- Requires exactly 32 ETH (~$100,000+)
- 24/7 uptime requirement
- Technical setup: execution client + consensus client

**Staking pools (e.g., Lido, RocketPool):**
- Pool multiple users' ETH
- Lido issues stETH (liquid staking derivative — you earn rewards while holding)
- RocketPool requires only 8 ETH for "mini-pool" validators
- stETH/rETH can be used in DeFi (collateral, LP, etc.)

**Centralized staking:**
- Coinbase's cbETH, Binance's BETH
- Trade-off: convenience vs. centralization risk

**Systemic risk:** If Lido controls >33% of validators, they could prevent finality. As of 2025, Lido has ~28% — watched closely.

---

## Part 2: Hands-On (2 hours)

### Exercise 1: Compare PoW vs PoS

Create a simple simulation in JavaScript showing why PoW uses more energy:

```javascript
// Simulate PoW mining difficulty
function mineBlock(data, difficulty) {
  let nonce = 0
  const target = '0'.repeat(difficulty)
  
  console.time('mining')
  while (true) {
    const attempt = `${data}${nonce}`
    // Simple hash simulation (in real life, use keccak256)
    const hash = simpleHash(attempt)
    
    if (hash.startsWith(target)) {
      console.timeEnd('mining')
      console.log(`Found nonce: ${nonce}`)
      console.log(`Hash: ${hash}`)
      console.log(`Attempts: ${nonce}`)
      return { nonce, hash }
    }
    nonce++
  }
}

// Very simple hash for demonstration (NOT cryptographically secure)
function simpleHash(str) {
  let hash = 0
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i)
    hash = ((hash << 5) - hash) + char
    hash = hash & hash // 32-bit int
  }
  return Math.abs(hash).toString(16).padStart(8, '0')
}

// Try difficulty 2 vs 4 vs 6 — notice exponential time increase
mineBlock("block_data_100", 2) // fast
mineBlock("block_data_100", 4) // slower
```

---

### Exercise 2: Explore Ethereum Validators

Go to https://beaconcha.in (Ethereum consensus layer explorer)

1. Check current validator count
2. Find a validator with recent attestations
3. Check a validator's performance (attestation rate, proposals)
4. Look at the current epoch and slot

**Questions:**
- What is the current APY for validators?
- What is the "participation rate" for the last epoch?
- How many ETH are currently staked?

---

### Exercise 3: Staking Protocol Comparison

Research and fill in this table:

| Protocol | Minimum Stake | Token | APY | Decentralized? |
|---------|--------------|-------|-----|----------------|
| Solo Validator | 32 ETH | N/A | | Fully |
| Lido | Any | stETH | | No (but distributed) |
| RocketPool | 0.01 ETH | rETH | | Yes |
| Coinbase | Any | cbETH | | No |

Sources:
- https://lido.fi
- https://rocketpool.net
- https://www.coinbase.com/staking

---

### Exercise 4: Read a Consensus Layer Transaction

Go to https://beaconcha.in and find:
1. A block with >100 attestations
2. A slashing event (search "slashings" tab)
3. The validator that proposed the latest block

---

## Part 3: Written Exercises (1 hour)

### Explain-it-to-a-child Challenge

Write a 5-sentence explanation of why Ethereum switched from PoW to PoS that a non-technical person could understand.

### Attack Scenario Analysis

For each attack, calculate the approximate cost and whether it succeeds:

1. **51% attack on Ethereum PoS:**
   - Need to control 51% of ~32M staked ETH
   - ETH price: ~$3,000
   - Can you actually attack? What happens after?

2. **Long-range attack on PoS:**
   - Attacker acquires old validator keys from validators who have exited
   - They try to rewrite history from a year ago
   - Why doesn't this work? (Hint: weak subjectivity)

---

## Part 4: Reflection (30 min)

1. If PoS is more efficient, why doesn't Bitcoin switch to PoS?
2. What is "nothing at stake" problem in naive PoS, and how does slashing solve it?
3. Why does Ethereum have both LMD-GHOST and Casper FFG? What does each solve?
4. What is "weak subjectivity" and why do new nodes joining PoS need it?
5. How does liquid staking (stETH) create systemic DeFi risk?

---

## Resources for Day 3

| Resource | Link | Type |
|----------|------|------|
| Ethereum PoS Explained | https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/ | Docs |
| Beaconcha.in Explorer | https://beaconcha.in | Tool |
| Casper FFG Paper | https://arxiv.org/abs/1710.09437 | Paper |
| Gasper Paper | https://arxiv.org/abs/2003.03052 | Paper |
| The Merge (official) | https://ethereum.org/en/upgrades/merge/ | Docs |
| Lido Docs | https://docs.lido.fi | Docs |
| RocketPool Docs | https://docs.rocketpool.net | Docs |
| PoS vs PoW Energy | https://ethereum.org/en/energy-consumption/ | Article |
| Ultrasound.money | https://ultrasound.money | Dashboard |

---

## Tomorrow

[Day 4 → Solidity In-Depth](./Day-4-Solidity-Deep-Dive.md)
