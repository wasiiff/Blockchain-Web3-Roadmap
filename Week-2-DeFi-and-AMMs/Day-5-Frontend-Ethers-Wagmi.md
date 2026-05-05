# Day 12 (Week 2, Day 5) — Web3 Frontend: Ethers.js & Wagmi/Viem

> **Time:** 6–8 hours  
> **Goal:** Build a production Web3 React frontend using Wagmi v2 + Viem. Connect wallets, read contract data, send transactions.

---

## Part 1: Wagmi + Viem Setup (1 hour)

### Why Wagmi + Viem over raw Ethers.js?

| Feature | Ethers.js v5/v6 | Wagmi v2 + Viem |
|---------|----------------|-----------------|
| React hooks | Manual | Built-in hooks |
| Type safety | Good | Excellent (full ABI inference) |
| Bundle size | Large | Smaller (tree-shakeable) |
| Caching | Manual | Automatic |
| Auto reconnect | Manual | Built-in |
| Multi-wallet | Manual | RainbowKit/ConnectKit |
| TypeScript | OK | First-class |

**Rule of thumb:**
- Scripts / backend / tests → use Ethers.js or Viem
- React frontend → use Wagmi + Viem

### 1.1 Project Setup

```bash
npx create-next-app@latest my-web3-app --typescript --tailwind --app
cd my-web3-app

npm install wagmi viem @tanstack/react-query
npm install @rainbow-me/rainbowkit  # beautiful wallet modal

# For raw blockchain interaction without React
npm install viem  # used by wagmi under the hood, also standalone
```

### 1.2 Provider Configuration

```typescript
// src/config/wagmi.ts
import { http, createConfig } from 'wagmi'
import { mainnet, sepolia, arbitrum, optimism, polygon } from 'wagmi/chains'
import { injected, metaMask, walletConnect, coinbaseWallet } from 'wagmi/connectors'

export const config = createConfig({
  chains: [mainnet, sepolia, arbitrum, optimism, polygon],
  connectors: [
    injected(),       // MetaMask and other injected wallets
    metaMask(),
    walletConnect({
      projectId: process.env.NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID!,
    }),
    coinbaseWallet({
      appName: 'My Web3 App',
    }),
  ],
  transports: {
    [mainnet.id]: http(process.env.NEXT_PUBLIC_ALCHEMY_MAINNET_URL),
    [sepolia.id]: http(process.env.NEXT_PUBLIC_ALCHEMY_SEPOLIA_URL),
    [arbitrum.id]: http(),
    [optimism.id]: http(),
    [polygon.id]: http(),
  },
})

declare module 'wagmi' {
  interface Register {
    config: typeof config
  }
}
```

```typescript
// src/app/providers.tsx
'use client'
import { WagmiProvider } from 'wagmi'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { RainbowKitProvider, darkTheme } from '@rainbow-me/rainbowkit'
import '@rainbow-me/rainbowkit/styles.css'
import { config } from '@/config/wagmi'

const queryClient = new QueryClient()

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider theme={darkTheme()}>
          {children}
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  )
}
```

---

## Part 2: Core Wagmi Hooks (2 hours)

### 2.1 Wallet Connection

```typescript
// components/WalletButton.tsx
'use client'
import { ConnectButton } from '@rainbow-me/rainbowkit'
import { useAccount, useBalance, useDisconnect, useChainId } from 'wagmi'

export function WalletInfo() {
  const { address, isConnected, status } = useAccount()
  const { data: balance } = useBalance({ address })
  const chainId = useChainId()
  const { disconnect } = useDisconnect()
  
  if (!isConnected) {
    return <ConnectButton />
  }
  
  return (
    <div>
      <p>Address: {address}</p>
      <p>Balance: {balance?.formatted} {balance?.symbol}</p>
      <p>Chain ID: {chainId}</p>
      <button onClick={() => disconnect()}>Disconnect</button>
    </div>
  )
}
```

### 2.2 Reading Contract Data

```typescript
// hooks/useToken.ts
import { useReadContract, useReadContracts } from 'wagmi'
import { erc20Abi, formatUnits } from 'viem'

export function useTokenBalance(tokenAddress: `0x${string}`, userAddress: `0x${string}`) {
  // Single read
  const { data: balance, isLoading, error } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [userAddress],
  })
  
  // Batch read (multiple contracts in one RPC call)
  const { data: tokenData } = useReadContracts({
    contracts: [
      { address: tokenAddress, abi: erc20Abi, functionName: 'name' },
      { address: tokenAddress, abi: erc20Abi, functionName: 'symbol' },
      { address: tokenAddress, abi: erc20Abi, functionName: 'decimals' },
      { address: tokenAddress, abi: erc20Abi, functionName: 'totalSupply' },
      { address: tokenAddress, abi: erc20Abi, functionName: 'balanceOf', args: [userAddress] },
    ],
  })
  
  const [name, symbol, decimals, totalSupply, userBalance] = tokenData?.map(d => d.result) ?? []
  
  return {
    balance: userBalance && decimals
      ? formatUnits(userBalance as bigint, decimals as number)
      : '0',
    name: name as string,
    symbol: symbol as string,
    decimals: decimals as number,
    totalSupply: totalSupply && decimals
      ? formatUnits(totalSupply as bigint, decimals as number)
      : '0',
    isLoading,
    error,
  }
}
```

### 2.3 Writing to Contracts (Transactions)

```typescript
// hooks/useTokenTransfer.ts
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi'
import { erc20Abi, parseUnits } from 'viem'

export function useTokenTransfer(tokenAddress: `0x${string}`) {
  const { writeContract, data: hash, error, isPending } = useWriteContract()
  
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  })
  
  const transfer = (to: `0x${string}`, amount: string, decimals: number) => {
    writeContract({
      address: tokenAddress,
      abi: erc20Abi,
      functionName: 'transfer',
      args: [to, parseUnits(amount, decimals)],
    })
  }
  
  return {
    transfer,
    hash,
    isPending,     // waiting for MetaMask confirmation
    isConfirming,  // waiting for block inclusion
    isSuccess,     // tx confirmed!
    error,
  }
}

// Usage in component:
function TransferForm({ tokenAddress, decimals }) {
  const { transfer, isPending, isConfirming, isSuccess, error } = useTokenTransfer(tokenAddress)
  const [to, setTo] = useState('')
  const [amount, setAmount] = useState('')
  
  return (
    <form onSubmit={(e) => { e.preventDefault(); transfer(to as `0x${string}`, amount, decimals) }}>
      <input value={to} onChange={e => setTo(e.target.value)} placeholder="Recipient" />
      <input value={amount} onChange={e => setAmount(e.target.value)} placeholder="Amount" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Confirm in wallet...' : 
         isConfirming ? 'Confirming...' : 
         'Send'}
      </button>
      {isSuccess && <p>Transfer successful!</p>}
      {error && <p>Error: {error.message}</p>}
    </form>
  )
}
```

### 2.4 Watching Events

```typescript
// hooks/useTransferEvents.ts
import { useWatchContractEvent } from 'wagmi'
import { erc20Abi } from 'viem'

export function useTransferWatcher(tokenAddress: `0x${string}`, userAddress: `0x${string}`) {
  const [transfers, setTransfers] = useState<any[]>([])
  
  // Watch all Transfer events where user is sender or receiver
  useWatchContractEvent({
    address: tokenAddress,
    abi: erc20Abi,
    eventName: 'Transfer',
    args: { from: userAddress }, // filter by indexed parameter
    onLogs(logs) {
      setTransfers(prev => [...logs, ...prev])
    },
  })
  
  return transfers
}
```

---

## Part 3: Full DApp Example (3 hours)

### Token Dashboard Component

```typescript
// app/dashboard/page.tsx
'use client'
import { useState } from 'react'
import { useAccount, useReadContracts, useWriteContract, useWaitForTransactionReceipt } from 'wagmi'
import { erc20Abi, parseUnits, formatUnits } from 'viem'
import { ConnectButton } from '@rainbow-me/rainbowkit'

const TOKEN_ADDRESS = '0xYourTokenAddress' as const

const TOKEN_ABI = [
  ...erc20Abi,
  {
    name: 'mint',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'to', type: 'address' }, { name: 'amount', type: 'uint256' }],
    outputs: [],
  },
] as const

export default function Dashboard() {
  const { address, isConnected } = useAccount()
  const [transferTo, setTransferTo] = useState('')
  const [transferAmount, setTransferAmount] = useState('')
  
  const { data: tokenData, refetch } = useReadContracts({
    contracts: [
      { address: TOKEN_ADDRESS, abi: TOKEN_ABI, functionName: 'name' },
      { address: TOKEN_ADDRESS, abi: TOKEN_ABI, functionName: 'symbol' },
      { address: TOKEN_ADDRESS, abi: TOKEN_ABI, functionName: 'decimals' },
      { address: TOKEN_ADDRESS, abi: TOKEN_ABI, functionName: 'totalSupply' },
      { address: TOKEN_ADDRESS, abi: TOKEN_ABI, functionName: 'balanceOf', args: [address!] },
    ],
    query: { enabled: !!address },
  })
  
  const [name, symbol, decimals, totalSupply, balance] = 
    tokenData?.map(d => d.result) ?? []
  
  const { writeContract, data: txHash, isPending } = useWriteContract()
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash: txHash })
  
  const handleTransfer = () => {
    if (!transferTo || !transferAmount || !decimals) return
    writeContract({
      address: TOKEN_ADDRESS,
      abi: TOKEN_ABI,
      functionName: 'transfer',
      args: [transferTo as `0x${string}`, parseUnits(transferAmount, decimals as number)],
    })
  }
  
  if (!isConnected) {
    return (
      <div className="flex flex-col items-center justify-center min-h-screen">
        <h1 className="text-2xl mb-4">Token Dashboard</h1>
        <ConnectButton />
      </div>
    )
  }
  
  return (
    <div className="max-w-2xl mx-auto p-6">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-2xl font-bold">
          {name as string} ({symbol as string})
        </h1>
        <ConnectButton />
      </div>
      
      {/* Token Stats */}
      <div className="grid grid-cols-2 gap-4 mb-6">
        <div className="bg-gray-800 p-4 rounded-lg">
          <p className="text-gray-400 text-sm">Your Balance</p>
          <p className="text-2xl font-bold">
            {balance && decimals 
              ? `${formatUnits(balance as bigint, decimals as number)} ${symbol as string}`
              : '...'}
          </p>
        </div>
        <div className="bg-gray-800 p-4 rounded-lg">
          <p className="text-gray-400 text-sm">Total Supply</p>
          <p className="text-2xl font-bold">
            {totalSupply && decimals
              ? `${parseFloat(formatUnits(totalSupply as bigint, decimals as number)).toLocaleString()} ${symbol as string}`
              : '...'}
          </p>
        </div>
      </div>
      
      {/* Transfer Form */}
      <div className="bg-gray-800 p-4 rounded-lg">
        <h2 className="text-lg font-semibold mb-4">Transfer Tokens</h2>
        <input
          className="w-full bg-gray-700 p-3 rounded mb-3 text-white"
          placeholder="Recipient address (0x...)"
          value={transferTo}
          onChange={e => setTransferTo(e.target.value)}
        />
        <input
          className="w-full bg-gray-700 p-3 rounded mb-3 text-white"
          placeholder="Amount"
          type="number"
          value={transferAmount}
          onChange={e => setTransferAmount(e.target.value)}
        />
        <button
          className="w-full bg-blue-600 hover:bg-blue-700 p-3 rounded font-semibold disabled:opacity-50"
          onClick={handleTransfer}
          disabled={isPending || isConfirming || !transferTo || !transferAmount}
        >
          {isPending ? 'Confirm in MetaMask...' :
           isConfirming ? 'Waiting for confirmation...' :
           isSuccess ? '✓ Sent!' :
           'Send'}
        </button>
        
        {txHash && (
          <p className="text-sm text-gray-400 mt-2">
            Tx: <a 
              href={`https://sepolia.etherscan.io/tx/${txHash}`}
              target="_blank"
              className="text-blue-400 hover:underline"
            >
              {txHash.slice(0, 10)}...
            </a>
          </p>
        )}
      </div>
    </div>
  )
}
```

---

## Part 4: Viem for Scripts/Backend (1 hour)

```typescript
// scripts/readChain.ts
import { createPublicClient, createWalletClient, http, parseEther, formatEther } from 'viem'
import { sepolia } from 'viem/chains'
import { privateKeyToAccount } from 'viem/accounts'

// Read-only client
const publicClient = createPublicClient({
  chain: sepolia,
  transport: http(process.env.ALCHEMY_SEPOLIA_URL!),
})

// Write client (for sending transactions)
const account = privateKeyToAccount(`0x${process.env.PRIVATE_KEY!}`)
const walletClient = createWalletClient({
  account,
  chain: sepolia,
  transport: http(process.env.ALCHEMY_SEPOLIA_URL!),
})

async function main() {
  // Read block
  const block = await publicClient.getBlock()
  console.log('Block:', block.number)
  
  // Read balance
  const balance = await publicClient.getBalance({ address: account.address })
  console.log('Balance:', formatEther(balance), 'ETH')
  
  // Read contract
  const tokenName = await publicClient.readContract({
    address: '0xYourToken',
    abi: [{ name: 'name', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'string' }] }],
    functionName: 'name',
  })
  
  // Write contract
  const hash = await walletClient.writeContract({
    address: '0xYourToken',
    abi: [{ name: 'transfer', type: 'function', inputs: [{ name: 'to', type: 'address' }, { name: 'amount', type: 'uint256' }], outputs: [{ type: 'bool' }] }],
    functionName: 'transfer',
    args: ['0xRecipient', parseEther('1')],
  })
  
  // Wait for receipt
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  console.log('Gas used:', receipt.gasUsed)
}
```

---

## Resources for Day 12

| Resource | Link | Type |
|----------|------|------|
| Wagmi Docs | https://wagmi.sh/ | Docs |
| Viem Docs | https://viem.sh/ | Docs |
| RainbowKit | https://www.rainbowkit.com/ | Docs |
| Wagmi Examples | https://github.com/wevm/wagmi/tree/main/examples | Code |
| Next.js + Wagmi Template | https://github.com/wevm/wagmi/tree/main/examples/next | Code |
| WalletConnect Dashboard | https://cloud.walletconnect.com/ | Service |

---

## Tomorrow

[Day 13 → Full DeFi Frontend Integration](./Day-6-DeFi-Frontend-Integration.md)
