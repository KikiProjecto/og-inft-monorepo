# 0G INFT Monorepo

A complete implementation of Intelligent NFTs (INFTs) on the 0G Network using ERC-7857 standard. This monorepo contains smart contracts, an executor service for AI inference, and a React frontend.

## Architecture Overview

```
og-inft-monorepo/
├── packages/
│   ├── 0g-agent-inft/     # Smart contracts (Hardhat + Solidity)
│   ├── executor-service/   # Backend API for AI inference (Hono + TypeScript)
│   └── frontend/           # React frontend (Vite + wagmi + RainbowKit)
```

**Flow:**
1. User mints an INFT with agent config (name, model, system prompt, etc.)
2. Config hash is stored on-chain, actual config is registered with executor service
3. Authorized users can chat with the INFT via the frontend
4. Executor service verifies on-chain authorization and runs inference

## Prerequisites

- Node.js 18+
- Yarn 4 (enabled via corepack)
- A wallet with 0G Galileo Testnet tokens ([faucet](https://faucet.0g.ai))

## Quick Start

```bash
# Enable Yarn 4
corepack enable

# Install all dependencies
yarn install
```

---

## Step 1: Deploy Smart Contracts

### 1.1 Setup Environment

```bash
cd packages/0g-agent-inft

# Create .env file
cat > .env << EOF
PRIVATE_KEY=your_private_key_here
EOF
```

### 1.2 Setup Parameters

```bash
# Copy the example parameters file
cp ignition/parameters.example.json ignition/parameters.json
```

Edit `ignition/parameters.json` and replace all occurrences of `0xYourDeployerAddress` with your wallet address.

### 1.3 Compile Contracts

```bash
yarn hardhat compile
```

### 1.4 Deploy to 0G Galileo Testnet

The deployment uses Hardhat Ignition modules which handle the full deployment:
- TEEVerifier (TEE verification)
- Verifier (with beacon proxy pattern)
- AgentNFT (with beacon proxy pattern)

```bash
# Deploy and verify all contracts
yarn hardhat ignition deploy ignition/modules/AgentNFT.ts \
  --network zgTestnet \
  --parameters ignition/parameters.json \
  --verify
```

### 1.5 Verify Deployment

After deployment, you'll see deployed addresses:

![Deployed Addresses](docs/images/deployed-addresses.png)

**Important:** Use the **proxy addresses** (at the bottom, without `Impl` or `Beacon` suffix):

| Contract | Use Address From |
|----------|------------------|
| TEEVerifier | `TEEVerifierModule#TEEVerifier` |
| Verifier | `VerifierModule#Verifier` |
| AgentNFT | `AgentNFTModule#AgentNFT` |

Do **NOT** use the `*Impl` (implementation) or `*Beacon` addresses directly.

Save these proxy addresses - you'll need them for the executor service and frontend.

### 1.6 Verify on Block Explorer (Optional)

If you deployed without `--verify` or verification failed, you can verify manually:

```bash
yarn hardhat ignition verify chain-16602 --network zgTestnet
```

View on explorer: https://chainscan-galileo.0g.ai

---

## Step 2: Run Executor Service

The executor service handles:
- Agent config storage (off-chain, hash-verified)
- On-chain authorization verification
- AI inference via 0G Compute or mock responses

### 2.1 Setup Environment

```bash
cd packages/executor-service

# Create .env file
cat > .env << EOF
# Server
PORT=3001

# 0G Network
OG_RPC_URL=https://evmrpc-testnet.0g.ai
AGENT_NFT_ADDRESS=0x... # Your deployed AgentNFT address

# 0G Compute API (optional - for real AI responses)
OG_COMPUTE_URL=https://api.0g.ai/v1
OG_COMPUTE_API_KEY=your_api_key
EOF
```

To get your 0G Compute API key (run from monorepo root):

```bash
cd /path/to/og-inft-monorepo
yarn 0g-compute-cli inference get-secret --provider 0xa48f01287233509FD694a22Bf840225062E67836
```

### 2.2 Run Development Server

```bash
# Development mode with hot reload
yarn dev
```

The service will start on `http://localhost:3001`

### 2.3 Verify Service

```bash
# Health check
curl http://localhost:3001/health

# List registered agents
curl http://localhost:3001/api/agents/registered
```

### 2.4 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Health check |
| GET | `/api/agents` | List preset agent types |
| GET | `/api/agents/registered` | List all registered INFT agents |
| GET | `/api/auth/:tokenId/:address` | Check authorization |
| GET | `/api/token/:tokenId` | Get token details |
| GET | `/api/config/:tokenId` | Get stored agent config |
| POST | `/api/register` | Register agent config (owner only) |
| POST | `/api/inference` | Run AI inference |

### 2.5 Production Build

```bash
yarn build
yarn start
```

---

## Step 3: Run Frontend

The frontend provides:
- Wallet connection (RainbowKit)
- Mint new INFTs with custom agent configs
- View your owned INFTs
- Chat marketplace to discover and use agents

### 3.1 Setup Environment

```bash
cd packages/frontend

# Create .env file
cat > .env << EOF
VITE_EXECUTOR_URL=http://localhost:3001
EOF
```

### 3.2 Update Contract Addresses

Edit `src/config/contracts.ts` with your deployed addresses:

```typescript
export const CONTRACTS = {
  AgentNFT: "0x..." as Address,      // Your deployed AgentNFT
  AgentMarket: "0x..." as Address,   // Your deployed AgentMarket (if any)
  Verifier: "0x..." as Address,      // Your deployed Verifier
  TEEVerifier: "0x..." as Address,   // Your deployed TEEVerifier
};
```

### 3.3 Run Development Server

```bash
yarn dev
```

Open http://localhost:5173

### 3.4 Features

**Mint Page (`/mint`)**
- Configure agent: name, model, system prompt, personality
- Select capabilities and adjust temperature/max tokens
- Preview config hash before minting
- Auto-registers config with executor after mint

**My INFTs Page (`/my-infts`)**
- View all INFTs you own or have access to
- Manage authorizations (grant/revoke access)
- View on-chain intelligent data

**Chat Page (`/chat`)**
- Marketplace view of all registered agents
- Shows authorization status for each agent
- Chat with authorized agents
- Signature-based authentication for each message

### 3.5 Production Build

```bash
yarn build
yarn preview
```

---

## Full Workflow Example

### 1. Mint an INFT

1. Connect wallet on frontend
2. Go to Mint page
3. Configure your agent:
   - Name: "My AI Assistant"
   - Model: "qwen/qwen-2.5-7b-instruct"
   - System Prompt: "You are a helpful coding assistant..."
   - Personality: "technical and precise"
   - Capabilities: ["code", "explain"]
4. Click "Mint INFT"
5. Approve transaction in wallet
6. Config is automatically registered with executor

### 2. Chat with Your INFT

1. Go to Chat page
2. Find your agent in the marketplace
3. Click to select (you have access as owner)
4. Type a message and click Send
5. Sign the message in your wallet
6. Receive AI response

### 3. Grant Access to Others

1. Go to My INFTs page
2. Click on your INFT
3. Enter an address to authorize
4. Click "Authorize"
5. That address can now chat with your INFT

---

## Development

### Workspace Commands

```bash
# Install all dependencies
yarn install

# Run all packages in dev mode
yarn workspaces foreach -A run dev

# Build all packages
yarn workspaces foreach -A run build
```

### Contract Development

```bash
cd packages/0g-agent-inft

# Compile
yarn hardhat compile

# Test
yarn hardhat test

# Deploy to local network
yarn hardhat node
yarn hardhat ignition deploy ignition/modules/AgentNFT.ts --network localhost
```

### Adding New Agent Models

Edit `packages/frontend/src/pages/Mint.tsx`:

```typescript
const AVAILABLE_MODELS = [
  { id: "qwen/qwen-2.5-7b-instruct", name: "Qwen 2.5 7B Instruct" },
  { id: "your-model-id", name: "Your Model Name" },
];
```

---

## Configuration Reference

### Agent Config Schema

```typescript
interface AgentConfig {
  name: string;           // Agent display name
  model: string;          // Model ID (e.g., "qwen/qwen-2.5-7b-instruct")
  systemPrompt: string;   // System instructions for the AI
  personality: string;    // Personality description
  capabilities: string[]; // List of capabilities
  temperature: number;    // 0-2, controls randomness
  maxTokens: number;      // 100-4096, max response length
}
```

### Supported Models

- `qwen/qwen-2.5-7b-instruct` - Qwen 2.5 7B
- `qwen/qwen-2.5-72b-instruct` - Qwen 2.5 72B
- `meta-llama/llama-3.1-70b-instruct` - Llama 3.1 70B
- `deepseek-ai/DeepSeek-V3` - DeepSeek V3

---

## Troubleshooting

### "No intelligent data found"
- The INFT was minted without proper config hash
- Re-mint with the frontend which auto-registers config

### "Not authorized"
- You're not the owner and haven't been authorized
- Ask the owner to call `authorizeUsage(tokenId, yourAddress)`

### "Config hash mismatch"
- The stored config doesn't match on-chain hash
- Owner needs to re-register config via `/api/register`

### Executor service not responding
- Check if service is running on port 3001
- Verify AGENT_NFT_ADDRESS is correct in .env
- Check RPC URL is accessible

---

## Links

- [0G Network](https://0g.ai)
- [0G Galileo Testnet Faucet](https://faucet.0g.ai)
- [Block Explorer](https://chainscan-galileo.0g.ai)
- [ERC-7857 Standard](https://eips.ethereum.org/EIPS/eip-7857)

## License

MIT
