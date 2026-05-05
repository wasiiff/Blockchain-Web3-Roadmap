# Day 16 (Week 3, Day 2) — IPFS & Decentralized Storage

> **Time:** 6–8 hours  
> **Goal:** Upload NFT images and metadata to IPFS using Pinata. Understand CIDs, pinning, and why IPFS matters for NFTs.

---

## Part 1: How IPFS Works (1 hour)

### 1.1 Content-Addressed vs Location-Addressed

**HTTP (location-addressed):**
```
https://api.nftproject.com/token/1
→ "Go to this server and ask for /token/1"
→ Server can return anything, change response, or go offline
```

**IPFS (content-addressed):**
```
ipfs://QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG
→ "Find the content whose hash is QmYwAP..."
→ Content is IMMUTABLE — if content changes, hash changes
→ Multiple nodes can serve the same content
```

**CID (Content Identifier):**
- A CID is the cryptographic hash of the content
- CIDv0: starts with "Qm..." (SHA2-256)
- CIDv1: starts with "bafy..." or "bafk..." (newer format, various hash functions)
- Changing one byte changes the entire CID

### 1.2 IPFS vs Pinning

IPFS is a protocol, not a storage service. Files in IPFS are stored by nodes.

**Problem:** Nodes automatically garbage-collect files they don't actively "pin."

**Pinning:** Telling a node "keep this CID forever." Without pinning, your NFT metadata could disappear.

**Solutions:**
- Run your own IPFS node and pin files
- Use a pinning service: **Pinata**, **Infura IPFS**, **nft.storage**, **Fleek**
- Use Arweave (permanent, pay-once storage)

---

## Part 2: Pinata Setup & Upload (2 hours)

### 2.1 Create Pinata Account

1. Go to https://pinata.cloud
2. Sign up (free tier: 1GB, 100 files)
3. Create API key: Profile → API Keys → New Key
4. Save: `PINATA_API_KEY`, `PINATA_SECRET_KEY`, `PINATA_JWT`

### 2.2 Upload Single File

```javascript
// scripts/uploadToIPFS.js
const axios = require('axios')
const FormData = require('form-data')
const fs = require('fs')
require('dotenv').config()

const PINATA_JWT = process.env.PINATA_JWT

// Upload a single image file
async function uploadFile(filePath, fileName) {
  const formData = new FormData()
  
  const fileStream = fs.createReadStream(filePath)
  formData.append('file', fileStream, { filename: fileName })
  
  formData.append('pinataMetadata', JSON.stringify({
    name: fileName,
    keyvalues: {
      project: 'ArtBlock',
      type: 'image'
    }
  }))
  
  formData.append('pinataOptions', JSON.stringify({
    cidVersion: 1  // Use CIDv1 (bafk... format)
  }))
  
  const response = await axios.post(
    'https://api.pinata.cloud/pinning/pinFileToIPFS',
    formData,
    {
      headers: {
        Authorization: `Bearer ${PINATA_JWT}`,
        ...formData.getHeaders()
      }
    }
  )
  
  const ipfsHash = response.data.IpfsHash
  console.log(`Uploaded ${fileName}: ipfs://${ipfsHash}`)
  return ipfsHash
}

// Upload a directory (all images at once)
async function uploadDirectory(dirPath, dirName) {
  const formData = new FormData()
  
  const files = fs.readdirSync(dirPath)
  for (const file of files) {
    const filePath = `${dirPath}/${file}`
    const fileStream = fs.createReadStream(filePath)
    formData.append('file', fileStream, { filepath: `${dirName}/${file}` })
  }
  
  formData.append('pinataMetadata', JSON.stringify({ name: dirName }))
  formData.append('pinataOptions', JSON.stringify({ cidVersion: 1, wrapWithDirectory: true }))
  
  const response = await axios.post(
    'https://api.pinata.cloud/pinning/pinFileToIPFS',
    formData,
    { headers: { Authorization: `Bearer ${PINATA_JWT}`, ...formData.getHeaders() } }
  )
  
  console.log(`Uploaded directory: ipfs://${response.data.IpfsHash}`)
  return response.data.IpfsHash
}
```

### 2.3 NFT Metadata Upload Pipeline

```javascript
// scripts/mintingPipeline.js
// Full pipeline: generate images → upload to IPFS → upload metadata → deploy

const path = require('path')
const fs = require('fs')

const COLLECTION_SIZE = 100 // for testing

async function runPipeline() {
  console.log("Step 1: Upload images to IPFS...")
  
  // Assume images are in ./nft-images/ as 0.png, 1.png, ..., 99.png
  const imagesDirCID = await uploadDirectory('./nft-images', 'artblock-images')
  const imagesBaseURI = `ipfs://${imagesDirCID}/artblock-images/`
  console.log("Images base URI:", imagesBaseURI)
  
  console.log("\nStep 2: Generate and upload metadata...")
  
  // Generate metadata for each token
  const metadataDir = './nft-metadata'
  if (!fs.existsSync(metadataDir)) fs.mkdirSync(metadataDir)
  
  for (let i = 0; i < COLLECTION_SIZE; i++) {
    const metadata = {
      name: `ArtBlock #${i}`,
      description: "A unique piece from the ArtBlock collection.",
      image: `${imagesBaseURI}${i}.png`,
      external_url: `https://artblock.io/token/${i}`,
      attributes: generateAttributes(i)
    }
    
    fs.writeFileSync(
      `${metadataDir}/${i}.json`,
      JSON.stringify(metadata, null, 2)
    )
  }
  
  const metadataDirCID = await uploadDirectory('./nft-metadata', 'artblock-metadata')
  const metadataBaseURI = `ipfs://${metadataDirCID}/artblock-metadata/`
  console.log("Metadata base URI:", metadataBaseURI)
  
  // Save results
  fs.writeFileSync('./ipfs-results.json', JSON.stringify({
    imagesCID: imagesDirCID,
    metadataCID: metadataDirCID,
    imagesBaseURI,
    metadataBaseURI,
    uploadedAt: new Date().toISOString()
  }, null, 2))
  
  console.log("\nAll done! Use metadataBaseURI in your contract.")
  return metadataBaseURI
}

function generateAttributes(tokenId) {
  // Generate deterministic traits from tokenId
  const backgrounds = ["Forest", "Ocean", "Desert", "Space", "City"]
  const bodies = ["Human", "Robot", "Alien", "Ghost", "Dragon"]
  const accessories = ["None", "Hat", "Glasses", "Crown", "Mask"]
  
  return [
    { trait_type: "Background", value: backgrounds[tokenId % backgrounds.length] },
    { trait_type: "Body", value: bodies[Math.floor(tokenId / 5) % bodies.length] },
    { trait_type: "Accessory", value: accessories[Math.floor(tokenId / 10) % accessories.length] },
    { display_type: "number", trait_type: "Edition", value: tokenId + 1 },
  ]
}
```

---

## Part 3: NFT.Storage (Free Alternative)

nft.storage is free and specifically built for NFT metadata. Backed by Protocol Labs and Filecoin.

```javascript
// npm install nft.storage files-from-path
import { NFTStorage, File, Blob } from 'nft.storage'
import fs from 'fs'

const NFT_STORAGE_TOKEN = process.env.NFT_STORAGE_TOKEN  // get from nft.storage

async function storeNFT(imagePath, name, description, attributes) {
  const client = new NFTStorage({ token: NFT_STORAGE_TOKEN })
  
  const imageData = fs.readFileSync(imagePath)
  const image = new File([imageData], 'nft.png', { type: 'image/png' })
  
  const metadata = await client.store({
    name,
    description,
    image,
    attributes,
  })
  
  console.log('IPFS URL:', metadata.url)
  // e.g., ipfs://bafyreia.../{id}
  return metadata
}

// For batch minting an entire collection:
async function storeCollection(imagesDir, metadataArray) {
  const client = new NFTStorage({ token: NFT_STORAGE_TOKEN })
  
  const files = metadataArray.map((meta, i) => ({
    ...meta,
    image: new File(
      [fs.readFileSync(`${imagesDir}/${i}.png`)],
      `${i}.png`,
      { type: 'image/png' }
    )
  }))
  
  // Store directory
  const cid = await client.storeDirectory(
    files.map((f, i) => new File([JSON.stringify(f)], `${i}.json`))
  )
  
  return `ipfs://${cid}/`
}
```

---

## Part 4: Arweave — Permanent Storage

Arweave is an alternative to IPFS where you pay once for permanent storage.

```javascript
// npm install arweave
const Arweave = require('arweave')

const arweave = Arweave.init({
  host: 'arweave.net',
  port: 443,
  protocol: 'https',
})

async function uploadToArweave(data, contentType, walletKeyFile) {
  const wallet = JSON.parse(fs.readFileSync(walletKeyFile))
  
  const tx = await arweave.createTransaction({
    data: Buffer.from(data),
  }, wallet)
  
  tx.addTag('Content-Type', contentType)
  tx.addTag('App-Name', 'ArtBlock')
  
  await arweave.transactions.sign(tx, wallet)
  const response = await arweave.transactions.post(tx)
  
  if (response.status === 200 || response.status === 208) {
    const url = `https://arweave.net/${tx.id}`
    console.log("Arweave URL:", url)
    return url
  }
  
  throw new Error(`Upload failed: ${response.status}`)
}
```

| Feature | IPFS + Pinata | Arweave | HTTP |
|---------|--------------|---------|------|
| Permanence | If pinned | Permanent | Centralized |
| Cost | $0-20/mo | ~$0.001/KB | Server cost |
| Decentralized | Yes | Yes | No |
| Speed | Good | Good | Fast |
| Best for | Most NFTs | High-value, 1/1 | None |

---

## Part 5: Verify Your IPFS Upload

```javascript
// Verify metadata is accessible
async function verifyIPFS(cid) {
  const gateways = [
    `https://ipfs.io/ipfs/${cid}`,
    `https://cloudflare-ipfs.com/ipfs/${cid}`,
    `https://gateway.pinata.cloud/ipfs/${cid}`,
    `https://dweb.link/ipfs/${cid}`,
  ]
  
  for (const url of gateways) {
    try {
      const res = await fetch(url, { signal: AbortSignal.timeout(5000) })
      if (res.ok) {
        const data = await res.json()
        console.log(`✓ ${url} — name: ${data.name}`)
      }
    } catch (e) {
      console.log(`✗ ${url} — ${e.message}`)
    }
  }
}
```

---

## Resources for Day 16

| Resource | Link | Type |
|----------|------|------|
| Pinata | https://pinata.cloud | Service |
| NFT.Storage | https://nft.storage | Service |
| Arweave | https://www.arweave.org/ | Service |
| IPFS Docs | https://docs.ipfs.tech/ | Docs |
| CID Inspector | https://cid.ipfs.tech/ | Tool |
| IPFS Desktop | https://github.com/ipfs/ipfs-desktop | App |
| Public IPFS Gateways | https://ipfs.github.io/public-gateway-checker/ | Reference |

---

## Tomorrow

[Day 17 → NFT Marketplace Architecture](./Day-3-NFT-Marketplace-Architecture.md)
