# Day 20 (Week 3, Day 6) — NFT Frontend: Mint, Gallery, Buy

> **Time:** 6–8 hours  
> **Goal:** Build a complete NFT frontend — wallet connect, mint, gallery with metadata, list for sale, and buy.

---

## Part 1: Project Setup (30 min)

```bash
npx create-next-app@latest nft-marketplace --typescript --tailwind --app
cd nft-marketplace

npm install wagmi viem @tanstack/react-query
npm install @rainbow-me/rainbowkit
npm install axios  # for IPFS gateway fetching
```

---

## Part 2: NFT Hooks (1.5 hours)

```typescript
// hooks/useNFTCollection.ts
import { useReadContracts, useAccount } from 'wagmi'
import { useState, useEffect } from 'react'

const NFT_ABI = [
  { name: 'totalSupply', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'uint256' }] },
  { name: 'ownerOf', type: 'function', stateMutability: 'view', inputs: [{ name: 'tokenId', type: 'uint256' }], outputs: [{ type: 'address' }] },
  { name: 'tokenURI', type: 'function', stateMutability: 'view', inputs: [{ name: 'tokenId', type: 'uint256' }], outputs: [{ type: 'string' }] },
  { name: 'balanceOf', type: 'function', stateMutability: 'view', inputs: [{ name: 'owner', type: 'address' }], outputs: [{ type: 'uint256' }] },
  { name: 'mintPrice', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'uint256' }] },
  { name: 'publicSaleActive', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'bool' }] },
  { name: 'MAX_SUPPLY', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'uint256' }] },
] as const

export interface NFTMetadata {
  tokenId: number
  owner: string
  name: string
  description: string
  image: string
  attributes: { trait_type: string; value: string | number }[]
}

export function useCollectionInfo(nftAddress: `0x${string}`) {
  const { address } = useAccount()
  
  const { data } = useReadContracts({
    contracts: [
      { address: nftAddress, abi: NFT_ABI, functionName: 'totalSupply' },
      { address: nftAddress, abi: NFT_ABI, functionName: 'MAX_SUPPLY' },
      { address: nftAddress, abi: NFT_ABI, functionName: 'mintPrice' },
      { address: nftAddress, abi: NFT_ABI, functionName: 'publicSaleActive' },
      ...(address ? [{
        address: nftAddress, abi: NFT_ABI, functionName: 'balanceOf', args: [address]
      }] : []),
    ],
  })
  
  const [totalSupply, maxSupply, mintPrice, saleActive, userBalance] = 
    data?.map(d => d.result) ?? []
  
  return {
    totalSupply: Number(totalSupply ?? 0),
    maxSupply: Number(maxSupply ?? 0),
    mintPrice: mintPrice as bigint ?? 0n,
    saleActive: saleActive as boolean ?? false,
    userBalance: Number(userBalance ?? 0),
  }
}

// Fetch metadata for a list of tokenIds
export function useNFTMetadata(nftAddress: `0x${string}`, tokenIds: number[]) {
  const [metadata, setMetadata] = useState<Map<number, NFTMetadata>>(new Map())
  const [loading, setLoading] = useState(false)
  
  const contracts = tokenIds.flatMap(id => [
    { address: nftAddress, abi: NFT_ABI, functionName: 'tokenURI', args: [BigInt(id)] },
    { address: nftAddress, abi: NFT_ABI, functionName: 'ownerOf', args: [BigInt(id)] },
  ])
  
  const { data: contractData } = useReadContracts({ contracts: contracts as any })
  
  useEffect(() => {
    if (!contractData) return
    
    async function fetchAllMetadata() {
      setLoading(true)
      const newMetadata = new Map<number, NFTMetadata>()
      
      for (let i = 0; i < tokenIds.length; i++) {
        const tokenId = tokenIds[i]
        const uri = contractData[i * 2]?.result as string
        const owner = contractData[i * 2 + 1]?.result as string
        
        if (!uri) continue
        
        try {
          const httpURI = uri.startsWith('ipfs://')
            ? uri.replace('ipfs://', 'https://ipfs.io/ipfs/')
            : uri
          
          const meta = await fetch(httpURI).then(r => r.json())
          
          newMetadata.set(tokenId, {
            tokenId,
            owner,
            name: meta.name,
            description: meta.description,
            image: meta.image?.startsWith('ipfs://')
              ? meta.image.replace('ipfs://', 'https://ipfs.io/ipfs/')
              : meta.image,
            attributes: meta.attributes ?? [],
          })
        } catch (e) {
          console.warn(`Failed to fetch metadata for token ${tokenId}`)
        }
      }
      
      setMetadata(newMetadata)
      setLoading(false)
    }
    
    fetchAllMetadata()
  }, [contractData, tokenIds.join(',')])
  
  return { metadata, loading }
}
```

---

## Part 3: Mint Component (1 hour)

```typescript
// components/MintSection.tsx
'use client'
import { useState } from 'react'
import { useWriteContract, useWaitForTransactionReceipt, useAccount } from 'wagmi'
import { formatEther } from 'viem'
import { useCollectionInfo } from '@/hooks/useNFTCollection'

const NFT_ADDRESS = '0xYourNFTAddress' as const

const MINT_ABI = [
  { 
    name: 'publicMint', type: 'function', stateMutability: 'payable',
    inputs: [{ name: 'quantity', type: 'uint256' }], outputs: [] 
  },
] as const

export function MintSection() {
  const { address, isConnected } = useAccount()
  const { totalSupply, maxSupply, mintPrice, saleActive, userBalance } = useCollectionInfo(NFT_ADDRESS)
  const [quantity, setQuantity] = useState(1)
  
  const { writeContract, data: hash, isPending, error } = useWriteContract()
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash })
  
  const totalCost = mintPrice * BigInt(quantity)
  const remaining = maxSupply - totalSupply
  
  const handleMint = () => {
    if (!isConnected) return
    writeContract({
      address: NFT_ADDRESS,
      abi: MINT_ABI,
      functionName: 'publicMint',
      args: [BigInt(quantity)],
      value: totalCost,
    })
  }
  
  return (
    <section className="bg-gray-900 rounded-2xl p-8">
      <div className="text-center mb-6">
        <h2 className="text-3xl font-bold mb-2">ArtBlock Collection</h2>
        <p className="text-gray-400">
          {totalSupply} / {maxSupply} minted
        </p>
        
        {/* Mint Progress Bar */}
        <div className="w-full bg-gray-700 rounded-full h-3 mt-3">
          <div 
            className="bg-blue-500 h-3 rounded-full transition-all"
            style={{ width: `${maxSupply > 0 ? (totalSupply / maxSupply) * 100 : 0}%` }}
          />
        </div>
      </div>
      
      <div className="grid grid-cols-2 gap-4 mb-6">
        <div className="bg-gray-800 p-4 rounded-xl text-center">
          <p className="text-gray-400 text-sm">Price</p>
          <p className="text-xl font-bold">{formatEther(mintPrice)} ETH</p>
        </div>
        <div className="bg-gray-800 p-4 rounded-xl text-center">
          <p className="text-gray-400 text-sm">You own</p>
          <p className="text-xl font-bold">{userBalance} NFTs</p>
        </div>
      </div>
      
      {/* Quantity Selector */}
      <div className="flex items-center justify-center gap-4 mb-6">
        <button
          onClick={() => setQuantity(Math.max(1, quantity - 1))}
          className="w-10 h-10 rounded-full bg-gray-700 hover:bg-gray-600 text-xl font-bold"
        >−</button>
        <span className="text-2xl font-bold w-8 text-center">{quantity}</span>
        <button
          onClick={() => setQuantity(Math.min(5, quantity + 1))}
          className="w-10 h-10 rounded-full bg-gray-700 hover:bg-gray-600 text-xl font-bold"
        >+</button>
      </div>
      
      <div className="text-center text-gray-400 text-sm mb-4">
        Total: {formatEther(totalCost)} ETH
      </div>
      
      {!isConnected ? (
        <p className="text-center text-gray-400">Connect wallet to mint</p>
      ) : !saleActive ? (
        <div className="bg-yellow-900 text-yellow-300 p-4 rounded-xl text-center">
          ⚠️ Sale is not active yet
        </div>
      ) : remaining === 0 ? (
        <div className="bg-red-900 text-red-300 p-4 rounded-xl text-center">
          Sold out!
        </div>
      ) : (
        <button
          onClick={handleMint}
          disabled={isPending || isConfirming}
          className="w-full bg-blue-600 hover:bg-blue-700 disabled:opacity-50 p-4 rounded-xl font-bold text-lg transition-colors"
        >
          {isPending ? 'Confirm in MetaMask...' :
           isConfirming ? `Minting ${quantity} NFT${quantity > 1 ? 's' : ''}...` :
           isSuccess ? '✓ Minted!' :
           `Mint ${quantity} for ${formatEther(totalCost)} ETH`}
        </button>
      )}
      
      {error && (
        <p className="text-red-400 text-sm mt-3 text-center">
          {error.message.includes('InsufficientPayment') ? 'Insufficient ETH' :
           error.message.includes('SaleNotActive') ? 'Sale is not active' :
           error.message.includes('MaxPerWalletReached') ? 'Max per wallet reached' :
           `Error: ${error.shortMessage || error.message}`}
        </p>
      )}
    </section>
  )
}
```

---

## Part 4: Gallery Component (1 hour)

```typescript
// components/Gallery.tsx
'use client'
import { useState } from 'react'
import { useNFTMetadata } from '@/hooks/useNFTCollection'
import { NFTCard } from './NFTCard'

const NFT_ADDRESS = '0xYourNFTAddress' as const

export function Gallery({ totalSupply }: { totalSupply: number }) {
  const tokenIds = Array.from({ length: Math.min(totalSupply, 50) }, (_, i) => i)
  const { metadata, loading } = useNFTMetadata(NFT_ADDRESS, tokenIds)
  
  const [filter, setFilter] = useState<string>('')
  const [sortBy, setSortBy] = useState<'id' | 'rarity'>('id')
  
  if (loading && metadata.size === 0) {
    return (
      <div className="grid grid-cols-2 md:grid-cols-4 lg:grid-cols-5 gap-4">
        {Array.from({ length: 10 }).map((_, i) => (
          <div key={i} className="bg-gray-800 rounded-xl aspect-square animate-pulse" />
        ))}
      </div>
    )
  }
  
  const nfts = Array.from(metadata.values())
  
  return (
    <div>
      <div className="flex gap-3 mb-6">
        <input
          className="bg-gray-800 p-2 rounded-lg flex-1 text-sm"
          placeholder="Filter by trait..."
          value={filter}
          onChange={e => setFilter(e.target.value)}
        />
        <select
          className="bg-gray-800 p-2 rounded-lg text-sm"
          value={sortBy}
          onChange={e => setSortBy(e.target.value as 'id' | 'rarity')}
        >
          <option value="id">Sort by ID</option>
          <option value="rarity">Sort by Rarity</option>
        </select>
      </div>
      
      <div className="grid grid-cols-2 md:grid-cols-4 lg:grid-cols-5 gap-4">
        {nfts.map(nft => (
          <NFTCard key={nft.tokenId} nft={nft} nftAddress={NFT_ADDRESS} />
        ))}
      </div>
    </div>
  )
}

// components/NFTCard.tsx
import Link from 'next/link'
import { NFTMetadata } from '@/hooks/useNFTCollection'

export function NFTCard({ nft, nftAddress }: { nft: NFTMetadata, nftAddress: string }) {
  return (
    <Link href={`/token/${nft.tokenId}`}>
      <div className="bg-gray-800 rounded-xl overflow-hidden hover:ring-2 hover:ring-blue-500 transition-all cursor-pointer group">
        <div className="aspect-square overflow-hidden bg-gray-700">
          <img
            src={nft.image}
            alt={nft.name}
            className="w-full h-full object-cover group-hover:scale-105 transition-transform"
            loading="lazy"
          />
        </div>
        <div className="p-3">
          <p className="font-semibold text-sm truncate">{nft.name}</p>
          <p className="text-gray-400 text-xs truncate">
            {nft.owner.slice(0, 6)}...{nft.owner.slice(-4)}
          </p>
        </div>
      </div>
    </Link>
  )
}
```

---

## Part 5: Token Detail Page with Buy

```typescript
// app/token/[id]/page.tsx
'use client'
import { useParams } from 'next/navigation'
import { useReadContract, useWriteContract, useWaitForTransactionReceipt, useAccount } from 'wagmi'
import { formatEther } from 'viem'

export default function TokenPage() {
  const { id } = useParams()
  const tokenId = Number(id)
  const { address } = useAccount()
  
  // Read listing from marketplace
  // Read token metadata
  // Show: image, traits, owner, listing price
  // Buy button if listed
  
  return (
    <div className="max-w-4xl mx-auto p-6">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        {/* Left: Image */}
        <div>
          <img
            src={`/* token image from metadata */`}
            className="rounded-2xl w-full"
            alt={`Token #${tokenId}`}
          />
        </div>
        
        {/* Right: Details */}
        <div>
          <h1 className="text-3xl font-bold mb-2">ArtBlock #{tokenId}</h1>
          <p className="text-gray-400 mb-6">Owner: 0xabc...def</p>
          
          {/* Listing Info */}
          <div className="bg-gray-800 rounded-xl p-6 mb-6">
            <p className="text-gray-400 text-sm mb-1">Current Price</p>
            <p className="text-3xl font-bold">0.5 ETH</p>
            <button className="w-full bg-blue-600 hover:bg-blue-700 p-4 rounded-xl font-bold mt-4">
              Buy Now
            </button>
          </div>
          
          {/* Traits */}
          <h3 className="font-bold mb-3">Traits</h3>
          <div className="grid grid-cols-3 gap-2">
            {[
              { trait_type: 'Background', value: 'Ocean' },
              { trait_type: 'Body', value: 'Robot' },
            ].map(attr => (
              <div key={attr.trait_type} className="bg-gray-800 rounded-lg p-3 text-center">
                <p className="text-blue-400 text-xs">{attr.trait_type}</p>
                <p className="font-semibold text-sm">{String(attr.value)}</p>
              </div>
            ))}
          </div>
        </div>
      </div>
    </div>
  )
}
```

---

## Resources for Day 20

| Resource | Link | Type |
|----------|------|------|
| Wagmi Docs | https://wagmi.sh/ | Docs |
| Next.js App Router | https://nextjs.org/docs/app | Docs |
| IPFS Gateway Checker | https://ipfs.github.io/public-gateway-checker/ | Tool |
| OpenSea UI Inspiration | https://opensea.io | Reference |
| Uniswap Interface Source | https://github.com/Uniswap/interface | Code |

---

## Tomorrow

[Day 21 → Mini Project: Full NFT Marketplace](./Day-7-Mini-Project-NFT-Marketplace.md)
