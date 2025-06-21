# Flow Blockchain AI NFT Chatbot - Complete Development Guide

## Project Overview
Build an AI chatbot agent that can automatically create NFTs on Flow blockchain from user-provided images. The system will handle image processing, metadata generation, smart contract deployment, and NFT minting through a conversational interface.

## Architecture Components

### 1. Frontend (React + TypeScript)
- Chat interface for user interaction
- Image upload functionality
- NFT preview and confirmation
- Wallet integration (Flow CLI/FCL)

### 2. Backend (Node.js + Express)
- AI processing engine
- Image handling and IPFS storage
- Flow blockchain integration
- Smart contract deployment automation

### 3. Smart Contracts (Cadence)
- NFT collection contract
- Minting functionality
- Metadata standards compliance

### 4. External Services
- IPFS for image and metadata storage
- AI image processing
- Flow blockchain RPC

## Step-by-Step Development

## Phase 1: Environment Setup

### 1.1 Install Flow CLI
```bash
# Install Flow CLI
sh -ci "$(curl -fsSL https://raw.githubusercontent.com/onflow/flow-cli/master/install.sh)"

# Verify installation
flow version
```

### 1.2 Initialize Flow Project
```bash
# Create project directory
mkdir flow-nft-chatbot
cd flow-nft-chatbot

# Initialize Flow project
flow init

# Install dependencies
npm init -y
npm install @onflow/fcl @onflow/types @onflow/sdk
npm install express cors multer axios dotenv
npm install react react-dom @types/react @types/react-dom
npm install ipfs-http-client openai
```

### 1.3 Project Structure
```
flow-nft-chatbot/
├── cadence/
│   ├── contracts/
│   │   └── NFTCollection.cdc
│   ├── transactions/
│   │   ├── setup_account.cdc
│   │   └── mint_nft.cdc
│   └── scripts/
│       └── get_nft_info.cdc
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── services/
│   │   └── utils/
│   └── public/
├── backend/
│   ├── routes/
│   ├── services/
│   └── controllers/
├── flow.json
└── package.json
```

## Phase 2: Smart Contract Development

### 2.1 NFT Collection Contract (Cadence)

Create `cadence/contracts/NFTCollection.cdc`:

```cadence
import NonFungibleToken from 0x1d7e57aa55817448
import MetadataViews from 0x1d7e57aa55817448

pub contract AIGeneratedNFT: NonFungibleToken {
    
    pub var totalSupply: UInt64
    
    pub event ContractInitialized()
    pub event Withdraw(id: UInt64, from: Address?)
    pub event Deposit(id: UInt64, to: Address?)
    pub event Minted(id: UInt64, to: Address, metadata: {String: String})
    
    pub let CollectionStoragePath: StoragePath
    pub let CollectionPublicPath: PublicPath
    pub let MinterStoragePath: StoragePath
    
    pub resource NFT: NonFungibleToken.INFT, MetadataViews.Resolver {
        pub let id: UInt64
        pub let name: String
        pub let description: String
        pub let image: String
        pub let creator: Address
        pub let createdAt: UFix64
        pub let metadata: {String: String}
        
        init(
            id: UInt64,
            name: String,
            description: String,
            image: String,
            creator: Address,
            metadata: {String: String}
        ) {
            self.id = id
            self.name = name
            self.description = description
            self.image = image
            self.creator = creator
            self.createdAt = getCurrentBlock().timestamp
            self.metadata = metadata
        }
        
        pub fun getViews(): [Type] {
            return [
                Type<MetadataViews.Display>(),
                Type<MetadataViews.Royalties>(),
                Type<MetadataViews.ExternalURL>(),
                Type<MetadataViews.NFTCollectionData>(),
                Type<MetadataViews.NFTCollectionDisplay>(),
                Type<MetadataViews.Serial>(),
                Type<MetadataViews.Traits>()
            ]
        }
        
        pub fun resolveView(_ view: Type): AnyStruct? {
            switch view {
                case Type<MetadataViews.Display>():
                    return MetadataViews.Display(
                        name: self.name,
                        description: self.description,
                        thumbnail: MetadataViews.HTTPFile(
                            url: self.image
                        )
                    )
                case Type<MetadataViews.Serial>():
                    return MetadataViews.Serial(
                        self.id
                    )
                case Type<MetadataViews.Traits>():
                    let traits: [MetadataViews.Trait] = []
                    for key in self.metadata.keys {
                        traits.append(MetadataViews.Trait(
                            name: key,
                            value: self.metadata[key]!,
                            displayType: nil,
                            rarity: nil
                        ))
                    }
                    return MetadataViews.Traits(traits)
            }
            return nil
        }
    }
    
    pub resource interface AIGeneratedNFTCollectionPublic {
        pub fun deposit(token: @NonFungibleToken.NFT)
        pub fun getIDs(): [UInt64]
        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT
        pub fun borrowAIGeneratedNFT(id: UInt64): &AIGeneratedNFT.NFT? {
            post {
                (result == nil) || (result?.id == id):
                    "Cannot borrow AIGeneratedNFT reference: the ID of the returned reference is incorrect"
            }
        }
    }
    
    pub resource Collection: AIGeneratedNFTCollectionPublic, NonFungibleToken.Provider, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic, MetadataViews.ResolverCollection {
        pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}
        
        init () {
            self.ownedNFTs <- {}
        }
        
        pub fun withdraw(withdrawID: UInt64): @NonFungibleToken.NFT {
            let token <- self.ownedNFTs.remove(key: withdrawID) ?? panic("missing NFT")
            emit Withdraw(id: token.id, from: self.owner?.address)
            return <-token
        }
        
        pub fun deposit(token: @NonFungibleToken.NFT) {
            let token <- token as! @AIGeneratedNFT.NFT
            let id: UInt64 = token.id
            let oldToken <- self.ownedNFTs[id] <- token
            emit Deposit(id: id, to: self.owner?.address)
            destroy oldToken
        }
        
        pub fun getIDs(): [UInt64] {
            return self.ownedNFTs.keys
        }
        
        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT {
            return (&self.ownedNFTs[id] as &NonFungibleToken.NFT?)!
        }
        
        pub fun borrowAIGeneratedNFT(id: UInt64): &AIGeneratedNFT.NFT? {
            if self.ownedNFTs[id] != nil {
                let ref = (&self.ownedNFTs[id] as auth &NonFungibleToken.NFT?)!
                return ref as! &AIGeneratedNFT.NFT
            }
            return nil
        }
        
        pub fun borrowViewResolver(id: UInt64): &AnyResource{MetadataViews.Resolver} {
            let nft = (&self.ownedNFTs[id] as auth &NonFungibleToken.NFT?)!
            let aiNFT = nft as! &AIGeneratedNFT.NFT
            return aiNFT as &AnyResource{MetadataViews.Resolver}
        }
        
        destroy() {
            destroy self.ownedNFTs
        }
    }
    
    pub fun createEmptyCollection(): @NonFungibleToken.Collection {
        return <- create Collection()
    }
    
    pub resource NFTMinter {
        pub fun mintNFT(
            recipient: &{NonFungibleToken.CollectionPublic},
            name: String,
            description: String,
            image: String,
            metadata: {String: String}
        ): UInt64 {
            let newNFT <- create NFT(
                id: AIGeneratedNFT.totalSupply,
                name: name,
                description: description,
                image: image,
                creator: recipient.owner!.address,
                metadata: metadata
            )
            
            let tokenID = newNFT.id
            
            emit Minted(
                id: tokenID,
                to: recipient.owner!.address,
                metadata: metadata
            )
            
            recipient.deposit(token: <-newNFT)
            
            AIGeneratedNFT.totalSupply = AIGeneratedNFT.totalSupply + UInt64(1)
            
            return tokenID
        }
    }
    
    init() {
        self.totalSupply = 0
        
        self.CollectionStoragePath = /storage/AIGeneratedNFTCollection
        self.CollectionPublicPath = /public/AIGeneratedNFTCollection
        self.MinterStoragePath = /storage/AIGeneratedNFTMinter
        
        let collection <- create Collection()
        self.account.save(<-collection, to: self.CollectionStoragePath)
        
        self.account.link<&AIGeneratedNFT.Collection{NonFungibleToken.CollectionPublic, AIGeneratedNFT.AIGeneratedNFTCollectionPublic, MetadataViews.ResolverCollection}>(
            self.CollectionPublicPath,
            target: self.CollectionStoragePath
        )
        
        let minter <- create NFTMinter()
        self.account.save(<-minter, to: self.MinterStoragePath)
        
        emit ContractInitialized()
    }
}
```

### 2.2 Setup Account Transaction

Create `cadence/transactions/setup_account.cdc`:

```cadence
import NonFungibleToken from 0x1d7e57aa55817448
import AIGeneratedNFT from 0xf8d6e0586b0a20c7

transaction {
    prepare(signer: AuthAccount) {
        if signer.borrow<&AIGeneratedNFT.Collection>(from: AIGeneratedNFT.CollectionStoragePath) == nil {
            let collection <- AIGeneratedNFT.createEmptyCollection()
            signer.save(<-collection, to: AIGeneratedNFT.CollectionStoragePath)
            
            signer.link<&AIGeneratedNFT.Collection{NonFungibleToken.CollectionPublic, AIGeneratedNFT.AIGeneratedNFTCollectionPublic}>(
                AIGeneratedNFT.CollectionPublicPath,
                target: AIGeneratedNFT.CollectionStoragePath
            )
        }
    }
}
```

### 2.3 Mint NFT Transaction

Create `cadence/transactions/mint_nft.cdc`:

```cadence
import NonFungibleToken from 0x1d7e57aa55817448
import AIGeneratedNFT from 0xf8d6e0586b0a20c7

transaction(
    recipient: Address,
    name: String,
    description: String,
    image: String,
    metadata: {String: String}
) {
    let minter: &AIGeneratedNFT.NFTMinter
    let recipientCollectionRef: &{NonFungibleToken.CollectionPublic}
    
    prepare(signer: AuthAccount) {
        self.minter = signer.borrow<&AIGeneratedNFT.NFTMinter>(from: AIGeneratedNFT.MinterStoragePath)
            ?? panic("Could not borrow a reference to the NFT minter")
        
        self.recipientCollectionRef = getAccount(recipient)
            .getCapability(AIGeneratedNFT.CollectionPublicPath)
            .borrow<&{NonFungibleToken.CollectionPublic}>()
            ?? panic("Could not get receiver reference to the NFT Collection")
    }
    
    execute {
        self.minter.mintNFT(
            recipient: self.recipientCollectionRef,
            name: name,
            description: description,
            image: image,
            metadata: metadata
        )
    }
}
```

## Phase 3: Backend Development

### 3.1 Main Server File

Create `backend/server.js`:

```javascript
const express = require('express');
const cors = require('cors');
const multer = require('multer');
const path = require('path');
require('dotenv').config();

const nftRoutes = require('./routes/nft');
const chatRoutes = require('./routes/chat');

const app = express();
const PORT = process.env.PORT || 3001;

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// File upload configuration
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        cb(null, Date.now() + '-' + file.originalname);
    }
});

const upload = multer({ 
    storage: storage,
    limits: {
        fileSize: 10 * 1024 * 1024 // 10MB limit
    },
    fileFilter: (req, file, cb) => {
        if (file.mimetype.startsWith('image/')) {
            cb(null, true);
        } else {
            cb(new Error('Only image files are allowed!'), false);
        }
    }
});

// Routes
app.use('/api/nft', nftRoutes);
app.use('/api/chat', chatRoutes);

// File upload endpoint
app.post('/api/upload', upload.single('image'), (req, res) => {
    if (!req.file) {
        return res.status(400).json({ error: 'No file uploaded' });
    }
    
    res.json({
        success: true,
        filename: req.file.filename,
        filepath: req.file.path,
        originalname: req.file.originalname
    });
});

// Static files
app.use('/uploads', express.static('uploads'));

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

### 3.2 NFT Service

Create `backend/services/nftService.js`:

```javascript
const * as fcl from "@onflow/fcl";
const * as t from "@onflow/types";
const fs = require('fs');
const { create } = require('ipfs-http-client');

// Configure FCL
fcl.config({
    "accessNode.api": process.env.FLOW_ACCESS_NODE || "https://rest-testnet.onflow.org",
    "discovery.wallet": process.env.FLOW_WALLET_DISCOVERY || "https://fcl-discovery.onflow.org/testnet/authn",
    "0xProfile": process.env.FLOW_PROFILE_ADDRESS,
    "flow.network": "testnet"
});

// IPFS client
const ipfs = create({
    host: 'ipfs.infura.io',
    port: 5001,
    protocol: 'https',
    headers: {
        authorization: `Basic ${Buffer.from(
            `${process.env.IPFS_PROJECT_ID}:${process.env.IPFS_PROJECT_SECRET}`
        ).toString('base64')}`
    }
});

class NFTService {
    
    async uploadToIPFS(filePath, metadata) {
        try {
            // Upload image to IPFS
            const imageFile = fs.readFileSync(filePath);
            const imageResult = await ipfs.add(imageFile);
            const imageUrl = `https://ipfs.io/ipfs/${imageResult.path}`;
            
            // Create metadata object
            const nftMetadata = {
                name: metadata.name,
                description: metadata.description,
                image: imageUrl,
                attributes: metadata.attributes || [],
                created_by: "AI NFT Chatbot",
                created_at: new Date().toISOString()
            };
            
            // Upload metadata to IPFS
            const metadataResult = await ipfs.add(JSON.stringify(nftMetadata));
            const metadataUrl = `https://ipfs.io/ipfs/${metadataResult.path}`;
            
            return {
                imageUrl,
                metadataUrl,
                metadata: nftMetadata
            };
            
        } catch (error) {
            console.error('Error uploading to IPFS:', error);
            throw error;
        }
    }
    
    async setupAccount(userAddress) {
        try {
            const setupTransaction = fs.readFileSync(
                path.join(__dirname, '../../cadence/transactions/setup_account.cdc'),
                'utf8'
            );
            
            const txId = await fcl.mutate({
                cadence: setupTransaction,
                proposer: fcl.authz,
                payer: fcl.authz,
                authorizations: [fcl.authz],
                limit: 1000
            });
            
            return await fcl.tx(txId).onceSealed();
            
        } catch (error) {
            console.error('Error setting up account:', error);
            throw error;
        }
    }
    
    async mintNFT(recipientAddress, nftData) {
        try {
            const mintTransaction = fs.readFileSync(
                path.join(__dirname, '../../cadence/transactions/mint_nft.cdc'),
                'utf8'
            );
            
            const txId = await fcl.mutate({
                cadence: mintTransaction,
                args: (arg, t) => [
                    arg(recipientAddress, t.Address),
                    arg(nftData.name, t.String),
                    arg(nftData.description, t.String),
                    arg(nftData.imageUrl, t.String),
                    arg(nftData.metadata, t.Dictionary({ key: t.String, value: t.String }))
                ],
                proposer: fcl.authz,
                payer: fcl.authz,
                authorizations: [fcl.authz],
                limit: 1000
            });
            
            return await fcl.tx(txId).onceSealed();
            
        } catch (error) {
            console.error('Error minting NFT:', error);
            throw error;
        }
    }
    
    async getNFTInfo(address, tokenId) {
        try {
            const script = fs.readFileSync(
                path.join(__dirname, '../../cadence/scripts/get_nft_info.cdc'),
                'utf8'
            );
            
            const result = await fcl.query({
                cadence: script,
                args: (arg, t) => [
                    arg(address, t.Address),
                    arg(tokenId, t.UInt64)
                ]
            });
            
            return result;
            
        } catch (error) {
            console.error('Error getting NFT info:', error);
            throw error;
        }
    }
}

module.exports = new NFTService();
```

### 3.3 AI Chat Service

Create `backend/services/chatService.js`:

```javascript
const { Configuration, OpenAIApi } = require('openai');

const configuration = new Configuration({
    apiKey: process.env.OPENAI_API_KEY,
});

const openai = new OpenAIApi(configuration);

class ChatService {
    
    async processUserMessage(message, imageData = null) {
        try {
            let systemPrompt = `You are an AI assistant that helps users create NFTs on the Flow blockchain. 
            You can help with:
            1. Analyzing uploaded images and suggesting NFT metadata
            2. Guiding users through the NFT creation process
            3. Explaining Flow blockchain and NFT concepts
            4. Processing natural language requests for NFT creation
            
            When a user uploads an image, analyze it and suggest:
            - A creative name for the NFT
            - A detailed description
            - Relevant attributes/traits
            - Estimated rarity or uniqueness
            
            Always be helpful, creative, and technically accurate about Flow blockchain.`;
            
            let userPrompt = message;
            
            if (imageData) {
                userPrompt += `\n\nI've uploaded an image for NFT creation. Please analyze it and provide suggestions for the NFT metadata.`;
            }
            
            const response = await openai.createChatCompletion({
                model: "gpt-4-vision-preview",
                messages: [
                    {
                        role: "system",
                        content: systemPrompt
                    },
                    {
                        role: "user",
                        content: imageData ? [
                            { type: "text", text: userPrompt },
                            { type: "image_url", image_url: { url: imageData } }
                        ] : userPrompt
                    }
                ],
                max_tokens: 1000,
                temperature: 0.7
            });
            
            return response.data.choices[0].message.content;
            
        } catch (error) {
            console.error('Error processing chat message:', error);
            throw error;
        }
    }
    
    async generateNFTMetadata(imagePath, userPreferences = {}) {
        try {
            // Convert image to base64 for OpenAI Vision API
            const fs = require('fs');
            const imageBuffer = fs.readFileSync(imagePath);
            const base64Image = imageBuffer.toString('base64');
            const imageDataUrl = `data:image/jpeg;base64,${base64Image}`;
            
            const prompt = `Analyze this image and create comprehensive NFT metadata. Consider:
            1. Visual elements, colors, style, composition
            2. Artistic technique or medium
            3. Emotional impact or mood
            4. Unique characteristics that make it valuable
            5. Potential categories or collections it could belong to
            
            User preferences: ${JSON.stringify(userPreferences)}
            
            Provide a JSON response with:
            - name: Creative, engaging title
            - description: Detailed, compelling description (2-3 sentences)
            - attributes: Array of key-value pairs describing traits
            - rarity_score: Estimated rarity (1-100)
            - suggested_price: Recommended starting price range`;
            
            const response = await openai.createChatCompletion({
                model: "gpt-4-vision-preview",
                messages: [
                    {
                        role: "user",
                        content: [
                            { type: "text", text: prompt },
                            { type: "image_url", image_url: { url: imageDataUrl } }
                        ]
                    }
                ],
                max_tokens: 800,
                temperature: 0.7
            });
            
            // Parse the JSON response
            const content = response.data.choices[0].message.content;
            const jsonMatch = content.match(/\{[\s\S]*\}/);
            
            if (jsonMatch) {
                return JSON.parse(jsonMatch[0]);
            } else {
                // Fallback if JSON parsing fails
                return {
                    name: "AI Generated NFT",
                    description: "A unique digital artwork created with AI assistance",
                    attributes: [
                        { trait_type: "Created By", value: "AI Assistant" },
                        { trait_type: "Generation", value: "Auto" }
                    ],
                    rarity_score: 50,
                    suggested_price: "0.1-1.0 FLOW"
                };
            }
            
        } catch (error) {
            console.error('Error generating NFT metadata:', error);
            throw error;
        }
    }
}

module.exports = new ChatService();
```

## Phase 4: Frontend Development

### 4.1 Main React Component

Create `frontend/src/components/NFTChatbot.jsx`:

```jsx
import React, { useState, useRef, useEffect } from 'react';
import * as fcl from '@onflow/fcl';
import './NFTChatbot.css';

// Configure FCL
fcl.config({
    'accessNode.api': process.env.REACT_APP_FLOW_ACCESS_NODE || 'https://rest-testnet.onflow.org',
    'discovery.wallet': process.env.REACT_APP_FLOW_WALLET_DISCOVERY || 'https://fcl-discovery.onflow.org/testnet/authn',
    'flow.network': 'testnet'
});

const NFTChatbot = () => {
    const [messages, setMessages] = useState([
        {
            type: 'bot',
            content: 'Hello! I\'m your AI NFT assistant. Upload an image and I\'ll help you create an NFT on Flow blockchain!',
            timestamp: new Date()
        }
    ]);
    const [inputMessage, setInputMessage] = useState('');
    const [uploadedImage, setUploadedImage] = useState(null);
    const [imagePreview, setImagePreview] = useState(null);
    const [isProcessing, setIsProcessing] = useState(false);
    const [user, setUser] = useState({ loggedIn: null });
    const [nftMetadata, setNftMetadata] = useState(null);
    const [showConfirmation, setShowConfirmation] = useState(false);
    
    const fileInputRef = useRef(null);
    const messagesEndRef = useRef(null);
    
    useEffect(() => {
        fcl.currentUser.subscribe(setUser);
    }, []);
    
    useEffect(() => {
        scrollToBottom();
    }, [messages]);
    
    const scrollToBottom = () => {
        messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
    };
    
    const addMessage = (type, content, extra = {}) => {
        setMessages(prev => [...prev, {
            type,
            content,
            timestamp: new Date(),
            ...extra
        }]);
    };
    
    const handleImageUpload = async (event) => {
        const file = event.target.files[0];
        if (!file) return;
        
        setUploadedImage(file);
        setImagePreview(URL.createObjectURL(file));
        
        addMessage('user', 'I\'ve uploaded an image for NFT creation.', { image: URL.createObjectURL(file) });
        
        setIsProcessing(true);
        
        try {
            const formData = new FormData();
            formData.append('image', file);
            
            const uploadResponse = await fetch('/api/upload', {
                method: 'POST',
                body: formData
            });
            
            const uploadResult = await uploadResponse.json();
            
            if (uploadResult.success) {
                // Process image with AI
                const analysisResponse = await fetch('/api/chat/analyze-image', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        imagePath: uploadResult.filepath,
                        message: 'Please analyze this image and suggest NFT metadata.'
                    })
                });
                
                const analysisResult = await analysisResponse.json();
                
                addMessage('bot', analysisResult.response);
                
                if (analysisResult.metadata) {
                    setNftMetadata(analysisResult.metadata);
                    setShowConfirmation(true);
                    addMessage('bot', 'Would you like to proceed with creating this NFT? Click "Create NFT" to continue!', {
                        showCreateButton: true
                    });
                }
            }
        } catch (error) {
            console.error('Error processing image:', error);
            addMessage('bot', 'Sorry, there was an error processing your image. Please try again.');
        } finally {
            setIsProcessing(false);
        }
    };
    
    const handleSendMessage = async () => {
        if (!inputMessage.trim()) return;
        
        addMessage('user', inputMessage);
        const userMessage = inputMessage;
        setInputMessage('');
        setIsProcessing(true);
        
        try {
            const response = await fetch('/api/chat/message', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    message: userMessage,
                    hasImage: !!uploadedImage
                })
            });
            
            const result = await response.json();
            addMessage('bot', result.response);
            
        } catch (error) {
            console.error('Error sending message:', error);
            addMessage('bot', 'Sorry, there was an error processing your message. Please try again.');
        } finally {
            setIsProcessing(false);
        }
    };
    
    const handleCreateNFT = async () => {
        if (!user.loggedIn) {
            addMessage('bot', 'Please connect your wallet first to create an NFT.');
            return;
        }
        
        if (!nftMetadata || !uploadedImage) {
            addMessage('bot', 'Please upload an image first.');
            return;
        }
        
        setIsProcessing(true);
        addMessage('bot', 'Creating your NFT on Flow blockchain... This may take a few moments.');
        
        try {
            const formData = new FormData();
            formData.append('image', uploadedImage);
            formData.append('metadata', JSON.stringify(nftMetadata));
            formData.append('recipientAddress', user.addr);
            
            const response = await fetch('/api/nft/create', {
                method: 'POST',
                body: formData
            });
            
            const result = await response.json();
            
            if (result.success) {
                addMessage('bot', `🎉 Congratulations! Your NFT has been successfully created on Flow blockchain!
                
                Transaction ID: ${result.transactionId}
                NFT Name: ${nftMetadata.name}
                
                You can view your NFT in your wallet or on Flow blockchain explorers.`, {
                    nftData: result.nftData
                });
                
                setShowConfirmation(false);
                setNftMetadata(null);
                setUploadedImage(null);
                setImagePreview(null);
                
            } else {
                addMessage('bot', `Sorry, there was an error creating your NFT: ${result.error}`);
            }
            
        } catch (error) {
            console.error('Error creating NFT:', error);
            addMessage('bot', 'Sorry, there was an error creating your NFT. Please try again.');
        } finally {
            setIsProcessing(false);
        }
    };
    
    const connectWallet = () => {
        fcl.logIn();
    };
    
    const disconnectWallet = () => {
        fcl.unauthenticate();
    };
    
    return (
        <div className="nft-chatbot">
            <div className="chatbot-header">
                <h2>🤖 AI NFT Creator for Flow</h2>
                <div className="wallet-section">
                    {user.loggedIn ? (
                        <div className="wallet-connected">
                            <span>Connected: {user.addr?.slice(0, 8)}...</span>
                            <button onClick={disconnectWallet} className="disconnect-btn">
                                Disconnect
                            </button>
                        </div>
                    ) : (
                        <button onClick={connectWallet} className="connect-btn">
                            Connect Wallet
                        </button>
                    )}
                </div>
            </div>
            
            <div className="messages-container">
                {messages.map((message, index) => (
                    <div key={index} className={`message ${message.type}`}>
                        <div className="message-content">
                            {message.image && (
                                <img src={message.image} alt="Uploaded" className="message-image" />
                            )}
                            <p>{message.content}</p>
                            {message.showCreateButton && (
                                <button 
                                    onClick={handleCreateNFT}
                                    className="create-nft-btn"
                                    disabled={!user.loggedIn || isProcessing}
                                >
                                    {user.loggedIn ? 'Create NFT on Flow' : 'Connect Wallet First'}
                                </button>
                            )}
                            {message.nftData && (
                                <div className="nft-result">
                                    <h4>Your NFT Details:</h4>
                                    <p><strong>Name:</strong> {message.nftData.name}</p>
                                    <p><strong>Description:</strong> {message.nftData.description}</p>
                                    <p><strong>Token ID:</strong> {message.nftData.tokenId}</p>
                                </div>
                            )}
                        </div>
                        <div className="message-timestamp">
                            {message.timestamp.toLocaleTimeString()}
                        </div>
                    </div>
                ))}
                {isProcessing && (
                    <div className="message bot">
                        <div className="message-content">
                            <div className="typing-indicator">
                                <span></span>
                                <span></span>
                                <span></span>
                            </div>
                        </div>
                    </div>
                )}
                <div ref={messagesEndRef} />
            </div>
            
            <div className="input-section">
                <div className="input-controls">
                    <input
                        type="file"
                        accept="image/*"
                        onChange={handleImageUpload}
                        ref={fileInputRef}
                        style={{ display: 'none' }}
                    />
                    <button 
                        onClick={() => fileInputRef.current?.click()}
                        className="upload-btn"
                        disabled={isProcessing}
                    >
                        📷 Upload Image
                    </button>
                    <input
                        type="text"
                        value={inputMessage}
                        onChange={(e) => setInputMessage(e.target.value)}
                        onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
                        placeholder="Ask me anything about creating NFTs..."
                        className="message-input"
                        disabled={isProcessing}
                    />
                    <button 
                        onClick={handleSendMessage}
                        className="send-btn"
                        disabled={isProcessing || !inputMessage.trim()}
                    >
                        Send
                    </button>
                </div>
            </div>
            
            {imagePreview && (
                <div className="image-preview">
                    <img src={imagePreview} alt="Preview" />
                    <button 
                        onClick={() => {
                            setImagePreview(null);
                            setUploadedImage(null);
                        }}
                        className="remove-image-btn"
                    >
                        ✕
                    </button>
                </div>
            )}
        </div>
    );
};

export default NFTChatbot;
```

### 4.2 CSS Styling

Create `frontend/src/components/NFTChatbot.css`:

```css
.nft-chatbot {
    max-width: 800px;
    margin: 0 auto;
    height: 100vh;
    display: flex;
    flex-direction: column;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.chatbot-header {
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    padding: 20px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid rgba(255, 255, 255, 0.2);
}

.chatbot-header h2 {
    color: white;
    margin: 0;
    font-size: 1.5rem;
}

.wallet-section {
    display: flex;
    align-items: center;
    gap: 10px;
}

.wallet-connected {
    display: flex;
    align-items: center;
    gap: 10px;
    color: white;
    font-size: 0.9rem;
}

.connect-btn, .disconnect-btn {
    background: #00D4FF;
    color: white;
    border: none;
    padding: 8px 16px;
    border-radius: 20px;
    cursor: pointer;
    font-weight: 500;
    transition: all 0.3s ease;
}

.connect-btn:hover, .disconnect-btn:hover {
    background: #00B8E6;
    transform: translateY(-2px);
}

.messages-container {
    flex: 1;
    overflow-y: auto;
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 15px;
}

.message {
    display: flex;
    flex-direction: column;
    max-width: 70%;
    animation: fadeIn 0.3s ease-in;
}

.message.user {
    align-self: flex-end;
}

.message.bot {
    align-self: flex-start;
}

.message-content {
    background: rgba(255, 255, 255, 0.9);
    padding: 15px;
    border-radius: 15px;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.message.user .message-content {
    background: #00D4FF;
    color: white;
}

.message.bot .message-content {
    background: rgba(255, 255, 255, 0.95);
    color: #333;
}

.message-content p {
    margin: 0;
    line-height: 1.5;
    white-space: pre-wrap;
}

.message-image {
    max-width: 200px;
    border-radius: 10px;
    margin-bottom: 10px;
}

.message-timestamp {
    font-size: 0.7rem;
    color: rgba(255, 255, 255, 0.7);
    margin-top: 5px;
    align-self: flex-end;
}

.message.user .message-timestamp {
    align-self: flex-start;
}

.create-nft-btn {
    background: #4CAF50;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 25px;
    cursor: pointer;
    font-weight: 500;
    margin-top: 10px;
    transition: all 0.3s ease;
}

.create-nft-btn:hover:not(:disabled) {
    background: #45a049;
    transform: translateY(-2px);
}

.create-nft-btn:disabled {
    background: #ccc;
    cursor: not-allowed;
}

.nft-result {
    background: rgba(76, 175, 80, 0.1);
    padding: 15px;
    border-radius: 10px;
    margin-top: 10px;
    border-left: 4px solid #4CAF50;
}

.nft-result h4 {
    margin: 0 0 10px 0;
    color: #4CAF50;
}

.nft-result p {
    margin: 5px 0;
    font-size: 0.9rem;
}

.input-section {
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    padding: 20px;
    border-top: 1px solid rgba(255, 255, 255, 0.2);
}

.input-controls {
    display: flex;
    gap: 10px;
    align-items: center;
}

.upload-btn {
    background: #FF6B6B;
    color: white;
    border: none;
    padding: 10px 15px;
    border-radius: 20px;
    cursor: pointer;
    font-weight: 500;
    white-space: nowrap;
    transition: all 0.3s ease;
}

.upload-btn:hover:not(:disabled) {
    background: #FF5252;
    transform: translateY(-2px);
}

.message-input {
    flex: 1;
    padding: 12px 15px;
    border: none;
    border-radius: 25px;
    background: rgba(255, 255, 255, 0.9);
    font-size: 1rem;
    outline: none;
}

.message-input:focus {
    background: white;
    box-shadow: 0 0 0 2px #00D4FF;
}

.send-btn {
    background: #00D4FF;
    color: white;
    border: none;
    padding: 12px 20px;
    border-radius: 25px;
    cursor: pointer;
    font-weight: 500;
    transition: all 0.3s ease;
}

.send-btn:hover:not(:disabled) {
    background: #00B8E6;
    transform: translateY(-2px);
}

.send-btn:disabled {
    background: #ccc;
    cursor: not-allowed;
}

.image-preview {
    position: fixed;
    bottom: 120px;
    right: 20px;
    background: white;
    padding: 10px;
    border-radius: 10px;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
    max-width: 200px;
}

.image-preview img {
    width: 100%;
    border-radius: 5px;
}

.remove-image-btn {
    position: absolute;
    top: 5px;
    right: 5px;
    background: #FF6B6B;
    color: white;
    border: none;
    width: 25px;
    height: 25px;
    border-radius: 50%;
    cursor: pointer;
    font-size: 12px;
    display: flex;
    align-items: center;
    justify-content: center;
}

.typing-indicator {
    display: flex;
    gap: 5px;
    align-items: center;
    padding: 10px 0;
}

.typing-indicator span {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: #666;
    animation: typing 1.4s infinite;
}

.typing-indicator span:nth-child(2) {
    animation-delay: 0.2s;
}

.typing-indicator span:nth-child(3) {
    animation-delay: 0.4s;
}

@keyframes typing {
    0%, 60%, 100% {
        transform: translateY(0);
    }
    30% {
        transform: translateY(-10px);
    }
}

@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

/* Responsive Design */
@media (max-width: 768px) {
    .nft-chatbot {
        height: 100vh;
    }
    
    .chatbot-header {
        padding: 15px;
    }
    
    .chatbot-header h2 {
        font-size: 1.2rem;
    }
    
    .messages-container {
        padding: 15px;
    }
    
    .message {
        max-width: 85%;
    }
    
    .input-controls {
        flex-direction: column;
        gap: 10px;
    }
    
    .message-input {
        width: 100%;
    }
    
    .image-preview {
        bottom: 180px;
        right: 10px;
        max-width: 150px;
    }
}
```

## Phase 5: Backend Routes

### 5.1 NFT Routes

Create `backend/routes/nft.js`:

```javascript
const express = require('express');
const router = express.Router();
const nftService = require('../services/nftService');
const multer = require('multer');
const path = require('path');

const upload = multer({
    storage: multer.diskStorage({
        destination: 'uploads/',
        filename: (req, file, cb) => {
            cb(null, Date.now() + '-' + file.originalname);
        }
    }),
    limits: { fileSize: 10 * 1024 * 1024 },
    fileFilter: (req, file, cb) => {
        if (file.mimetype.startsWith('image/')) {
            cb(null, true);
        } else {
            cb(new Error('Only image files allowed'));
        }
    }
});

// Create NFT endpoint
router.post('/create', upload.single('image'), async (req, res) => {
    try {
        const { recipientAddress } = req.body;
        const metadata = JSON.parse(req.body.metadata);
        
        if (!req.file) {
            return res.status(400).json({ error: 'No image uploaded' });
        }
        
        if (!recipientAddress) {
            return res.status(400).json({ error: 'Recipient address required' });
        }
        
        // Upload to IPFS
        const ipfsResult = await nftService.uploadToIPFS(req.file.path, metadata);
        
        // Setup account if needed
        await nftService.setupAccount(recipientAddress);
        
        // Mint NFT
        const nftData = {
            name: metadata.name,
            description: metadata.description,
            imageUrl: ipfsResult.imageUrl,
            metadata: {
                ...metadata.attributes.reduce((acc, attr) => {
                    acc[attr.trait_type] = attr.value;
                    return acc;
                }, {}),
                ipfs_metadata: ipfsResult.metadataUrl,
                created_at: new Date().toISOString()
            }
        };
        
        const mintResult = await nftService.mintNFT(recipientAddress, nftData);
        
        res.json({
            success: true,
            transactionId: mintResult.transactionId,
            nftData: {
                ...nftData,
                tokenId: mintResult.events.find(e => e.type.includes('Minted'))?.data?.id
            }
        });
        
    } catch (error) {
        console.error('Error creating NFT:', error);
        res.status(500).json({
            success: false,
            error: error.message
        });
    }
});

// Get NFT info
router.get('/info/:address/:tokenId', async (req, res) => {
    try {
        const { address, tokenId } = req.params;
        const nftInfo = await nftService.getNFTInfo(address, tokenId);
        
        res.json({
            success: true,
            nftInfo
        });
        
    } catch (error) {
        console.error('Error getting NFT info:', error);
        res.status(500).json({
            success: false,
            error: error.message
        });
    }
});

module.exports = router;
```

### 5.2 Chat Routes

Create `backend/routes/chat.js`:

```javascript
const express = require('express');
const router = express.Router();
const chatService = require('../services/chatService');

// Handle chat messages
router.post('/message', async (req, res) => {
    try {
        const { message, hasImage } = req.body;
        
        const response = await chatService.processUserMessage(message);
        
        res.json({
            success: true,
            response
        });
        
    } catch (error) {
        console.error('Error processing message:', error);
        res.status(500).json({
            success: false,
            error: 'Failed to process message'
        });
    }
});

// Analyze uploaded image
router.post('/analyze-image', async (req, res) => {
    try {
        const { imagePath, message } = req.body;
        
        // Generate AI analysis and metadata
        const metadata = await chatService.generateNFTMetadata(imagePath);
        const response = await chatService.processUserMessage(message, imagePath);
        
        res.json({
            success: true,
            response,
            metadata
        });
        
    } catch (error) {
        console.error('Error analyzing image:', error);
        res.status(500).json({
            success: false,
            error: 'Failed to analyze image'
        });
    }
});

module.exports = router;
```

## Phase 6: Cadence Scripts

### 6.1 Get NFT Info Script

Create `cadence/scripts/get_nft_info.cdc`:

```cadence
import NonFungibleToken from 0x1d7e57aa55817448
import AIGeneratedNFT from 0xf8d6e0586b0a20c7
import MetadataViews from 0x1d7e57aa55817448

pub fun main(address: Address, tokenId: UInt64): {String: AnyStruct}? {
    let account = getAccount(address)
    
    if let collection = account.getCapability<&AIGeneratedNFT.Collection{NonFungibleToken.CollectionPublic, AIGeneratedNFT.AIGeneratedNFTCollectionPublic}>(AIGeneratedNFT.CollectionPublicPath).borrow() {
        
        if let nft = collection.borrowAIGeneratedNFT(id: tokenId) {
            let display = nft.resolveView(Type<MetadataViews.Display>())! as! MetadataViews.Display
            let traits = nft.resolveView(Type<MetadataViews.Traits>())! as! MetadataViews.Traits
            
            return {
                "id": nft.id,
                "name": nft.name,
                "description": nft.description,
                "image": nft.image,
                "creator": nft.creator,
                "createdAt": nft.createdAt,
                "metadata": nft.metadata,
                "display": {
                    "name": display.name,
                    "description": display.description,
                    "thumbnail": display.thumbnail.uri()
                },
                "traits": traits.traits.map(fun (trait: MetadataViews.Trait): {String: AnyStruct} {
                    return {
                        "name": trait.name,
                        "value": trait.value,
                        "displayType": trait.displayType,
                        "rarity": trait.rarity?.score
                    }
                })
            }
        }
    }
    
    return nil
}
```

## Phase 7: Configuration Files

### 7.1 Flow Configuration

Create `flow.json`:

```json
{
    "version": "1.0",
    "networks": {
        "testnet": {
            "host": "access.devnet.nodes.onflow.org:9000",
            "chain": "flow-testnet"
        },
        "mainnet": {
            "host": "access.mainnet.nodes.onflow.org:9000",
            "chain": "flow-mainnet"
        }
    },
    "accounts": {
        "testnet-account": {
            "address": "0xf8d6e0586b0a20c7",
            "key": "$FLOW_PRIVATE_KEY"
        }
    },
    "contracts": {
        "AIGeneratedNFT": {
            "source": "./cadence/contracts/NFTCollection.cdc",
            "aliases": {
                "testnet": "0xf8d6e0586b0a20c7"
            }
        }
    },
    "deployments": {
        "testnet": {
            "testnet-account": [
                "AIGeneratedNFT"
            ]
        }
    }
}
```

### 7.2 Environment Variables

Create `.env`:

```env
# Server Configuration
PORT=3001
NODE_ENV=development

# Flow Blockchain Configuration
FLOW_ACCESS_NODE=https://rest-testnet.onflow.org
FLOW_WALLET_DISCOVERY=https://fcl-discovery.onflow.org/testnet/authn
FLOW_PRIVATE_KEY=your_flow_private_key_here
FLOW_PROFILE_ADDRESS=0xf8d6e0586b0a20c7

# IPFS Configuration
IPFS_PROJECT_ID=your_infura_ipfs_project_id
IPFS_PROJECT_SECRET=your_infura_ipfs_project_secret

# OpenAI Configuration
OPENAI_API_KEY=your_openai_api_key_here

# Frontend Configuration
REACT_APP_FLOW_ACCESS_NODE=https://rest-testnet.onflow.org
REACT_APP_FLOW_WALLET_DISCOVERY=https://fcl-discovery.onflow.org/testnet/authn
```

### 7.3 Package.json

Create `package.json`:

```json
{
  "name": "flow-nft-chatbot",
  "version": "1.0.0",
  "description": "AI-powered NFT creation chatbot for Flow blockchain",
  "main": "backend/server.js",
  "scripts": {
    "start": "node backend/server.js",
    "dev": "nodemon backend/server.js",
    "frontend": "cd frontend && npm start",
    "build": "cd frontend && npm run build",
    "deploy-contracts": "flow deploy --network testnet",
    "test": "jest"
  },
  "dependencies": {
    "@onflow/fcl": "^1.6.0",
    "@onflow/types": "^1.0.5",
    "@onflow/sdk": "^1.3.0",
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "multer": "^1.4.5-lts.1",
    "axios": "^1.4.0",
    "dotenv": "^16.3.1",
    "ipfs-http-client": "^60.0.0",
    "openai": "^4.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1",
    "@types/react": "^18.2.15",
    "@types/react-dom": "^18.2.7"
  }
}
```

## Phase 8: Deployment & Testing

### 8.1 Setup Instructions

1. **Install Dependencies:**
```bash
npm install
cd frontend && npm install
```

2. **Setup Flow Account:**
```bash
# Generate a Flow account
flow accounts create

# Fund your account (testnet)
# Visit https://testnet-faucet.onflow.org/

# Deploy contracts
flow deploy --network testnet
```

3. **Configure Services:**
- Set up Infura IPFS project
- Get OpenAI API key
- Configure environment variables

4. **Start Development:**
```bash
# Start backend
npm run dev

# Start frontend (in another terminal)
npm run frontend
```

### 8.2 Testing Checklist

- [ ] Wallet connection works
- [ ] Image upload functions
- [ ] AI analysis generates metadata
- [ ] IPFS upload successful
- [ ] NFT minting completes
- [ ] Transaction appears on blockchain
- [ ] NFT visible in wallet

## Phase 9: Advanced Features

### 9.1 Batch NFT Creation
- Support multiple image uploads
- Bulk minting functionality
- Collection management

### 9.2 Enhanced AI Features
- Style transfer options
- Custom prompt generation
- Rarity scoring algorithms

### 9.3 Marketplace Integration
- List created NFTs for sale
- Price recommendations
- Trading functionality

### 9.4 Social Features
- Share NFT creations
- Community collections
- User profiles

## Conclusion

This comprehensive guide provides everything needed to build a fully functional AI chatbot that creates NFTs on Flow blockchain. The system includes:

- Complete smart contract implementation
- Full-stack application with React frontend
- AI-powered image analysis and metadata generation
- IPFS integration for decentralized storage
- Flow blockchain integration
- User-friendly chat interface

Follow the step-by-step instructions to build your own AI NFT creation platform on Flow blockchain!
