# Day 13 (Week 2, Day 6) — Full DeFi Frontend Integration

> **Time:** 6–8 hours  
> **Goal:** Build a real DeFi frontend — connect to your deployed AMM, show pool state, enable swaps and liquidity management.

---

## Part 1: AMM Frontend Architecture

```
src/
├── config/
│   ├── wagmi.ts          ← chain + wallet config
│   └── contracts.ts      ← contract addresses + ABIs
├── hooks/
│   ├── usePool.ts        ← pool state (reserves, price, LP supply)
│   ├── useSwap.ts        ← swap logic (quotes + execution)
│   ├── useLiquidity.ts   ← add/remove liquidity
│   └── useTokens.ts      ← ERC-20 balance + approval
├── components/
│   ├── SwapCard.tsx      ← swap UI
│   ├── LiquidityCard.tsx ← add/remove liquidity UI
│   ├── PoolStats.tsx     ← pool info display
│   └── TxStatus.tsx      ← transaction status modal
└── app/
    └── page.tsx          ← main layout
```

---

## Part 2: Contract Config

```typescript
// config/contracts.ts
import { erc20Abi } from 'viem'

export const MINI_AMM_ADDRESS = '0xYourDeployedAMMAddress' as const

export const MINI_AMM_ABI = [
  // Read
  { name: 'getReserves', type: 'function', stateMutability: 'view', inputs: [], 
    outputs: [{ name: '_reserve0', type: 'uint112' }, { name: '_reserve1', type: 'uint112' }] },
  { name: 'totalSupply', type: 'function', stateMutability: 'view', inputs: [], 
    outputs: [{ type: 'uint256' }] },
  { name: 'balanceOf', type: 'function', stateMutability: 'view', 
    inputs: [{ name: 'address', type: 'address' }], outputs: [{ type: 'uint256' }] },
  { name: 'token0', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'address' }] },
  { name: 'token1', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'address' }] },
  { name: 'getAmountOut', type: 'function', stateMutability: 'pure',
    inputs: [{ name: 'amountIn', type: 'uint256' }, { name: 'reserveIn', type: 'uint256' }, { name: 'reserveOut', type: 'uint256' }],
    outputs: [{ type: 'uint256' }] },
  // Write
  { name: 'swap', type: 'function', stateMutability: 'nonpayable',
    inputs: [{ name: 'amount0Out', type: 'uint256' }, { name: 'amount1Out', type: 'uint256' }, { name: 'to', type: 'address' }],
    outputs: [] },
  { name: 'addLiquidity', type: 'function', stateMutability: 'nonpayable',
    inputs: [{ name: 'amount0Desired', type: 'uint256' }, { name: 'amount1Desired', type: 'uint256' }, { name: 'to', type: 'address' }],
    outputs: [{ name: 'liquidity', type: 'uint256' }] },
  { name: 'removeLiquidity', type: 'function', stateMutability: 'nonpayable',
    inputs: [{ name: 'lpAmount', type: 'uint256' }, { name: 'to', type: 'address' }],
    outputs: [{ name: 'amount0', type: 'uint256' }, { name: 'amount1', type: 'uint256' }] },
  // Events
  { name: 'Swap', type: 'event', inputs: [
    { name: 'sender', type: 'address', indexed: true },
    { name: 'amount0In', type: 'uint256' }, { name: 'amount1In', type: 'uint256' },
    { name: 'amount0Out', type: 'uint256' }, { name: 'amount1Out', type: 'uint256' },
    { name: 'to', type: 'address', indexed: true },
  ]},
] as const
```

---

## Part 3: Core Hooks

```typescript
// hooks/usePool.ts
import { useReadContracts } from 'wagmi'
import { formatUnits } from 'viem'
import { MINI_AMM_ADDRESS, MINI_AMM_ABI } from '@/config/contracts'

export function usePool() {
  const { data, isLoading, refetch } = useReadContracts({
    contracts: [
      { address: MINI_AMM_ADDRESS, abi: MINI_AMM_ABI, functionName: 'getReserves' },
      { address: MINI_AMM_ADDRESS, abi: MINI_AMM_ABI, functionName: 'totalSupply' },
      { address: MINI_AMM_ADDRESS, abi: MINI_AMM_ABI, functionName: 'token0' },
      { address: MINI_AMM_ADDRESS, abi: MINI_AMM_ABI, functionName: 'token1' },
    ],
  })
  
  const [reserves, totalSupply, token0, token1] = data?.map(d => d.result) ?? []
  
  const reserve0 = (reserves as [bigint, bigint])?.[0] ?? 0n
  const reserve1 = (reserves as [bigint, bigint])?.[1] ?? 0n
  
  const price0In1 = reserve0 > 0n 
    ? Number(formatUnits(reserve1, 18)) / Number(formatUnits(reserve0, 18))
    : 0
  const price1In0 = reserve1 > 0n
    ? Number(formatUnits(reserve0, 18)) / Number(formatUnits(reserve1, 18))
    : 0
  
  return {
    reserve0,
    reserve1,
    totalSupply: (totalSupply as bigint) ?? 0n,
    token0: token0 as `0x${string}`,
    token1: token1 as `0x${string}`,
    price0In1,
    price1In0,
    isLoading,
    refetch,
  }
}
```

```typescript
// hooks/useSwap.ts
import { useState, useCallback } from 'react'
import { useReadContract, useWriteContract, useWaitForTransactionReceipt } from 'wagmi'
import { parseUnits, formatUnits } from 'viem'
import { MINI_AMM_ADDRESS, MINI_AMM_ABI } from '@/config/contracts'
import { erc20Abi } from 'viem'

export function useSwap(
  tokenIn: `0x${string}`,
  tokenOut: `0x${string}`,
  tokenInDecimals: number,
  tokenOutDecimals: number,
  isToken0In: boolean
) {
  const [amountIn, setAmountIn] = useState('')
  
  const { data: pool } = useReadContract({
    address: MINI_AMM_ADDRESS,
    abi: MINI_AMM_ABI,
    functionName: 'getReserves',
  })
  
  const [reserve0, reserve1] = (pool as [bigint, bigint]) ?? [0n, 0n]
  const reserveIn = isToken0In ? reserve0 : reserve1
  const reserveOut = isToken0In ? reserve1 : reserve0
  
  const amountInParsed = amountIn ? parseUnits(amountIn, tokenInDecimals) : 0n
  
  const { data: amountOut } = useReadContract({
    address: MINI_AMM_ADDRESS,
    abi: MINI_AMM_ABI,
    functionName: 'getAmountOut',
    args: [amountInParsed, reserveIn, reserveOut],
    query: { enabled: amountInParsed > 0n && reserveIn > 0n },
  })
  
  // Approvals
  const { writeContract: approve, data: approveHash, isPending: isApproving } = useWriteContract()
  const { isSuccess: isApproved } = useWaitForTransactionReceipt({ hash: approveHash })
  
  // Swap
  const { writeContract: executeSwap, data: swapHash, isPending: isSwapping } = useWriteContract()
  const { isLoading: isSwapConfirming, isSuccess: isSwapSuccess } = useWaitForTransactionReceipt({ hash: swapHash })
  
  const handleApprove = useCallback(() => {
    approve({
      address: tokenIn,
      abi: erc20Abi,
      functionName: 'approve',
      args: [MINI_AMM_ADDRESS, amountInParsed],
    })
  }, [approve, tokenIn, amountInParsed])
  
  const handleSwap = useCallback((slippageBps: bigint = 50n) => {
    if (!amountOut) return
    const minOut = (amountOut as bigint) * (10000n - slippageBps) / 10000n
    
    const amount0Out = isToken0In ? 0n : minOut
    const amount1Out = isToken0In ? minOut : 0n
    
    // Note: in production, user's address comes from useAccount()
    executeSwap({
      address: MINI_AMM_ADDRESS,
      abi: MINI_AMM_ABI,
      functionName: 'swap',
      args: [amount0Out, amount1Out, '0xUSER_ADDRESS' as `0x${string}`],
    })
  }, [executeSwap, amountOut, isToken0In])
  
  const priceImpact = amountOut && reserveOut > 0n
    ? (1 - Number(amountOut) / Number(reserveOut)) * 100
    : 0
  
  return {
    amountIn, setAmountIn,
    amountOut: amountOut 
      ? formatUnits(amountOut as bigint, tokenOutDecimals) 
      : '0',
    priceImpact: priceImpact.toFixed(2),
    handleApprove,
    handleSwap,
    isApproving,
    isApproved,
    isSwapping,
    isSwapConfirming,
    isSwapSuccess,
  }
}
```

---

## Part 4: UI Components

```typescript
// components/SwapCard.tsx
'use client'
import { useState } from 'react'
import { useAccount } from 'wagmi'
import { usePool } from '@/hooks/usePool'
import { useSwap } from '@/hooks/useSwap'
import { formatUnits } from 'viem'

export function SwapCard() {
  const { address } = useAccount()
  const pool = usePool()
  const [isToken0In, setIsToken0In] = useState(true)
  
  const {
    amountIn, setAmountIn, amountOut, priceImpact,
    handleApprove, handleSwap,
    isApproving, isApproved, isSwapping, isSwapConfirming, isSwapSuccess,
  } = useSwap(
    isToken0In ? pool.token0 : pool.token1,
    isToken0In ? pool.token1 : pool.token0,
    18, 18, // decimals
    isToken0In
  )
  
  if (!address) return (
    <div className="bg-gray-900 rounded-2xl p-6 max-w-md mx-auto">
      Connect wallet to swap
    </div>
  )
  
  return (
    <div className="bg-gray-900 rounded-2xl p-6 max-w-md mx-auto">
      <div className="flex justify-between mb-4">
        <h2 className="text-xl font-bold">Swap</h2>
        <span className="text-gray-400 text-sm">
          Price: {pool.price0In1.toFixed(4)} Token1/Token0
        </span>
      </div>
      
      {/* Input Token */}
      <div className="bg-gray-800 rounded-xl p-4 mb-2">
        <div className="flex justify-between mb-2">
          <span className="text-gray-400 text-sm">You pay</span>
          <span className="text-gray-400 text-sm">
            Balance: {isToken0In 
              ? formatUnits(pool.reserve0, 18) 
              : formatUnits(pool.reserve1, 18)}
          </span>
        </div>
        <div className="flex items-center gap-3">
          <input
            className="bg-transparent text-2xl flex-1 outline-none"
            placeholder="0.0"
            type="number"
            value={amountIn}
            onChange={e => setAmountIn(e.target.value)}
          />
          <button
            className="bg-gray-700 px-3 py-1 rounded-xl text-sm font-semibold"
            onClick={() => setIsToken0In(!isToken0In)}
          >
            {isToken0In ? 'TOKEN0' : 'TOKEN1'} ↕
          </button>
        </div>
      </div>
      
      {/* Output Token */}
      <div className="bg-gray-800 rounded-xl p-4 mb-4">
        <div className="flex justify-between mb-2">
          <span className="text-gray-400 text-sm">You receive</span>
        </div>
        <div className="flex items-center gap-3">
          <span className="text-2xl flex-1 text-gray-300">
            {parseFloat(amountOut || '0').toFixed(6)}
          </span>
          <span className="bg-gray-700 px-3 py-1 rounded-xl text-sm font-semibold">
            {isToken0In ? 'TOKEN1' : 'TOKEN0'}
          </span>
        </div>
      </div>
      
      {/* Price Impact Warning */}
      {parseFloat(priceImpact) > 1 && (
        <div className={`p-3 rounded-xl mb-4 text-sm ${
          parseFloat(priceImpact) > 5 ? 'bg-red-900 text-red-300' : 'bg-yellow-900 text-yellow-300'
        }`}>
          Price Impact: {priceImpact}%
          {parseFloat(priceImpact) > 5 && ' ⚠️ High price impact!'}
        </div>
      )}
      
      {/* Action Buttons */}
      {!isApproved ? (
        <button
          className="w-full bg-blue-600 hover:bg-blue-700 p-4 rounded-xl font-bold disabled:opacity-50"
          onClick={handleApprove}
          disabled={!amountIn || isApproving}
        >
          {isApproving ? 'Approving...' : 'Approve Token'}
        </button>
      ) : (
        <button
          className="w-full bg-blue-600 hover:bg-blue-700 p-4 rounded-xl font-bold disabled:opacity-50"
          onClick={() => handleSwap()}
          disabled={!amountIn || isSwapping || isSwapConfirming}
        >
          {isSwapping ? 'Confirm in wallet...' :
           isSwapConfirming ? 'Confirming...' :
           isSwapSuccess ? '✓ Swapped!' : 'Swap'}
        </button>
      )}
    </div>
  )
}
```

---

## Part 5: Pool Statistics Display

```typescript
// components/PoolStats.tsx
import { usePool } from '@/hooks/usePool'
import { formatUnits } from 'viem'

export function PoolStats() {
  const { reserve0, reserve1, totalSupply, price0In1, price1In0 } = usePool()
  
  const tvl = parseFloat(formatUnits(reserve0, 18)) * price0In1 * 2
  
  return (
    <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
      <StatCard label="Reserve Token0" value={`${parseFloat(formatUnits(reserve0, 18)).toFixed(2)}`} />
      <StatCard label="Reserve Token1" value={`${parseFloat(formatUnits(reserve1, 18)).toFixed(2)}`} />
      <StatCard label="TVL (est.)" value={`$${tvl.toLocaleString()}`} />
      <StatCard label="LP Tokens" value={parseFloat(formatUnits(totalSupply, 18)).toFixed(4)} />
      <StatCard label="Price Token0" value={`${price0In1.toFixed(4)} T1/T0`} />
      <StatCard label="Price Token1" value={`${price1In0.toFixed(4)} T0/T1`} />
    </div>
  )
}

function StatCard({ label, value }: { label: string, value: string }) {
  return (
    <div className="bg-gray-800 rounded-xl p-4">
      <p className="text-gray-400 text-sm mb-1">{label}</p>
      <p className="text-xl font-bold">{value}</p>
    </div>
  )
}
```

---

## Resources for Day 13

| Resource | Link | Type |
|----------|------|------|
| Wagmi Examples | https://github.com/wevm/wagmi/tree/main/examples | Code |
| RainbowKit Components | https://www.rainbowkit.com/docs/introduction | Docs |
| Uniswap Interface Source | https://github.com/Uniswap/interface | Code |
| Next.js App Router | https://nextjs.org/docs/app | Docs |
| TailwindCSS | https://tailwindcss.com/docs | Docs |

---

## Tomorrow

[Day 14 → Mini Project: Deploy Your AMM](./Day-7-Mini-Project-AMM.md)
