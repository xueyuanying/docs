# MCP Server TronLink 

## Overview

**GitHub**: [https://github.com/TronLink/mcp-server-tronlink](https://github.com/TronLink/mcp-server-tronlink)

**mcp-server-tronlink** is a production-ready Model Context Protocol (MCP) server that enables AI agents (Claude, GPT, etc.) to interact with the TRON blockchain through natural language. Built on `@tronlink/tronlink-mcp-core`, it provides **55+ tools** across two complementary operation modes.

**Key Highlights:**
- Dual-mode architecture: **Playwright** (browser automation) + **Direct API** (on-chain operations)
- 32 built-in Flow Recipes with pre-checks and dependency resolution
- Non-custodial local transaction signing via encrypted `agent-wallet`
- Multi-signature management with real-time WebSocket monitoring
- Gas-free TRC20 transfers via GasFree service integration

---

## Architecture

```
AI Agent (Claude Desktop / Claude Code)
         | (MCP Protocol — stdio / JSON-RPC 2.0)
         v
TronLink MCP Server
├── Playwright Mode ─── TronLinkSessionManager
│   └── Browser automation + TronLink extension UI control
├── Direct API Mode
│   ├── TronLinkOnChainCapability   (14 tools)
│   ├── TronLinkMultiSigCapability  (5 tools)
│   └── TronLinkGasFreeCapability   (3 tools)
├── Utility Capabilities
│   ├── TronLinkBuildCapability     (extension build)
│   ├── TronLinkStateSnapshotCapability (UI state extraction)
│   └── TRON Crypto Utils           (address derivation, signing, Base58)
└── Flow Recipes (32 built-in recipes with pre-checks)
         |
         v
TronGrid API / Multi-Sig Service / GasFree Service
         |
         v
TRON Blockchain
```

Both modes can run simultaneously and tools are auto-enabled based on configuration.

---

## Dual-Mode Operation

### Mode 1: Playwright (Browser Automation)

Controls the TronLink Chrome extension via Playwright Chromium. Ideal for **E2E testing, UI validation, and DApp interaction**.

**Capabilities:**
- Launch browser with `--load-extension` flag for TronLink
- Auto-detect extension ID from Chrome API
- Multi-tab tracking with automatic role classification (extension / notification / dapp / other)
- DOM-based state extraction (TRON address, TRX balance, network detection)
- Screenshot capture with base64 encoding
- Automatic browser dialog handling (alerts, confirms, prompts)

**27 Playwright tools include:** `tl_launch`, `tl_cleanup`, `tl_navigate`, `tl_click`, `tl_type`, `tl_screenshot`, `tl_accessibility_snapshot`, `tl_describe_screen`, etc.

### Mode 2: Direct API (On-Chain)

Operates directly against TronGrid REST API — no browser required. Ideal for **account queries, transfers, swaps, staking, and multi-sig management**.

**25 API tools grouped into:**

| Group | Tools | Description |
|-------|-------|-------------|
| On-Chain | 14 | Transfer, stake, swap, query, multisig setup |
| Multi-Signature | 5 | Permission query, tx submit, WebSocket monitoring |
| GasFree | 3 | Zero-gas TRC20 transfers |
| Wallet Management | 3 | List wallets, auto-create a wallet, switch the active wallet |

---

## Core Components

### 1. TronLinkSessionManager

Full browser lifecycle management:

| Method | Description |
|--------|-------------|
| `launch()` | Initialize browser with TronLink extension |
| `getExtensionState()` | Extract wallet state from UI |
| `navigateToUrl()` | Navigate to a specific URL |
| `navigateToNotification()` | Open TronLink notification popup |
| `screenshot()` | Capture current UI state |
| `getTrackedPages()` | List all open browser tabs |
| `cleanup()` | Graceful shutdown of all resources |

**Screen Detection:** Auto-detects 15 TronLink screens: `home`, `login`, `settings`, `send`, `receive`, `sign`, `broadcast`, `assets`, `address_book`, `node_management`, `dapp_list`, `create_wallet`, `import_wallet`, `notification`, `unknown`.

### 2. TronLinkOnChainCapability (14 Tools)

Direct API wrapper for TronGrid:

**Query Operations:**
- `getAddress()` — Get the TRON address from encrypted local `agent-wallet`
- `getAccount()` — Balance, bandwidth, energy, permissions
- `getTokens()` — TRC10 and TRC20 token balances
- `getTransaction()` — Transaction details by txID
- `getHistory()` — Transaction history with pagination
- `getStakingInfo()` — Staking status (frozen amounts, votes, unfreezing)

**Transaction Operations:**
- `send()` — Transfer TRX, TRC10, or TRC20 tokens
- `stake()` — Freeze/unfreeze TRX for bandwidth or energy (Stake 2.0)
- `resource()` — Delegate/undelegate bandwidth or energy
- `swap()` — Token swap via SunSwap V2
- `swapV3()` — Token swap via SunSwap V3 Smart Router

**Multi-Sig Operations:**
- `setupMultisig()` — Configure multi-sig permissions
- `createMultisigTx()` — Create unsigned multi-sig transaction
- `signMultisigTx()` — Sign multi-sig transaction

### 3. TronLinkMultiSigCapability (5 Tools)

REST + WebSocket API for TRON multi-signature service:

- `queryAuth()` — Query multi-sig permissions (owner/active, thresholds, weights)
- `submitTransaction()` — Submit signed transaction (auto-broadcast when threshold reached)
- `queryTransactionList()` — List transactions with filtering
- `connectWebSocket()` — Real-time transaction monitoring
- `disconnectWebSocket()` — Stop monitoring

**Implementation:** HmacSHA256 signature generation for API auth, UUID-based request signing, supports both Nile testnet and Mainnet credentials.

### 4. TronLinkGasFreeCapability (3 Tools)

Zero-gas TRC20 transfers via GasFree service:

- `getAccount()` — Query eligibility, supported tokens, daily quota
- `getTransactions()` — Query gas-free transaction history
- `send()` — Send TRC20 with zero gas fee

### 5. Wallet Management (3 Tools)

Runtime wallet management via `@bankofai/agent-wallet` (encrypted `local_secure` storage):

- `tl_wallet_list` — List all wallets with IDs, types, active status, and TRON addresses
- `tl_wallet_create` — Auto-generate an encrypted wallet and attach it to the running MCP session
- `tl_wallet_set_active` — Switch the active wallet by ID (hot-swaps into all capabilities)

If no wallet exists at startup, the server prompts two paths: call `tl_wallet_create` to auto-generate one, or create one manually via CLI and set `AGENT_WALLET_PASSWORD`.
The auto-create path generates a random password, saves it to `~/.agent-wallet/runtime_secrets.json`, creates an encrypted `main` wallet, and enables the running session to use it.

### 6. TRON Cryptography Utils

Pure cryptographic functions — no external service calls:

```
signTransaction()          raw_data_hex → 65-byte signature (via agent-wallet)
base58CheckEncode()        Payload → base58check address
base58CheckDecode()        TRON address → 21-byte payload
addressToHex()             T-address → 0x41... hex
hexToAddress()             0x41... → T-address
```

Uses `@noble/curves` (secp256k1 ECDSA) and `@noble/hashes` (Keccak-256, SHA256). Private keys are never exposed — all signing is done through the encrypted `agent-wallet`.

---

## Flow Recipes (32 Built-In)

Pre-configured multi-step workflows with dependency checks and parameter templates.

### Playwright Flows
| Flow | Description |
|------|-------------|
| `switchNetworkFlow` | Switch to Mainnet/Nile/Shasta |
| `enableTestNetworksFlow` | Enable testnet visibility |
| `transferTrxFlow` | TRX transfer via UI |
| `transferTokenFlow` | Token transfer via UI |

### On-Chain Flows (11)
| Flow | Description |
|------|-------------|
| `chainCheckBalanceFlow` | Query balance |
| `chainTransferTrxFlow` | TRX transfer with pre-checks |
| `chainTransferTrc20Flow` | TRC20 transfer with pre-checks |
| `chainStakeFlow` | Stake TRX |
| `chainUnstakeFlow` | Unstake TRX |
| `chainGetStakingFlow` | Query staking info |
| `chainDelegateResourceFlow` | Delegate bandwidth/energy |
| `chainUndelegateResourceFlow` | Undelegate resources |
| `chainSetupMultisigFlow` | Setup multi-sig permissions |
| `chainCreateMultisigTxFlow` | Create unsigned multi-sig tx |
| `chainSwapV3Flow` | SunSwap V3 token swap |

### Multi-Sig Flows (6)
| Flow | Description |
|------|-------------|
| `multisigQueryAuthFlow` | Query permissions |
| `multisigListTransactionsFlow` | List pending transactions |
| `multisigMonitorFlow` | WebSocket real-time monitoring |
| `multisigStopMonitorFlow` | Stop monitoring |
| `multisigSubmitTxFlow` | Submit signed transaction |
| `multisigCheckFlow` | Full status check |

### GasFree Flows (3)
| Flow | Description |
|------|-------------|
| `gasfreeCheckAccountFlow` | Query eligibility |
| `gasfreeTransactionHistoryFlow` | Query history |
| `gasfreeSendFlow` | Gas-free TRC20 transfer |

---

## Configuration

### Environment Variables

**Playwright Mode:**

| Variable | Description |
|----------|-------------|
| `TRONLINK_EXTENSION_PATH` | TronLink extension build directory |
| `TRONLINK_SOURCE_PATH` | Enable build capability |
| `TL_MODE` | `e2e` (test) or `prod` (production) |
| `TL_HEADLESS` | Browser headless mode |
| `TL_SLOW_MO` | Playwright slow-motion delay (ms) |

**TronGrid API:**

| Variable | Description |
|----------|-------------|
| `TL_TRONGRID_URL` | Full-node API URL |
| `TL_TRONGRID_API_KEY` | API key (required for Mainnet) |
| `TL_SUNSWAP_ROUTER` | SunSwap V2 router address |
| `TL_SUNSWAP_V3_ROUTER` | SunSwap V3 smart router address |
| `TL_WTRX_ADDRESS` | WTRX contract address |

**Wallet (`agent-wallet`):**

| Variable | Description |
|----------|-------------|
| `AGENT_WALLET_PASSWORD` | Encryption password (optional if using `tl_wallet_create`; required for manual or existing wallets) |
| `AGENT_WALLET_DIR` | Custom wallet storage directory |
| `TL_OWNER_WALLET_ID` | Owner wallet ID for multisig signing |
| `TL_COSIGNER_WALLET_ID` | Co-signer wallet ID for multisig |

**Multi-Signature Service:**

| Variable | Description |
|----------|-------------|
| `TL_MULTISIG_BASE_URL` | API base URL |
| `TL_MULTISIG_SECRET_ID` | Project credential |
| `TL_MULTISIG_SECRET_KEY` | HmacSHA256 signing key |
| `TL_MULTISIG_CHANNEL` | Channel/project name |

**GasFree Service:**

| Variable | Description |
|----------|-------------|
| `TL_GASFREE_BASE_URL` | Service URL |
| `TL_GASFREE_API_KEY` | API key |
| `TL_GASFREE_API_SECRET` | API secret |

### Integration Options

**1. Project-Level MCP Config (`.mcp.json`)**

Auto-detected by Claude Code:
```json
{
  "mcpServers": {
    "tronlink": {
      "command": "node",
      "args": ["dist/index.js"],
      "cwd": ".",
      "env": {
        "TL_TRONGRID_URL": "https://nile.trongrid.io"
      }
    }
  }
}
```

If no wallet exists yet, startup shows two paths:

- Auto-create in the running MCP session: call `tl_wallet_create`
- Manual setup: create the wallet locally with `agent-wallet start local_secure --generate --wallet-id main`, then set `AGENT_WALLET_PASSWORD` and restart

If you choose auto-create, the server generates a random password, saves it to `~/.agent-wallet/runtime_secrets.json`, creates an encrypted `main` wallet, and continues with the current session.

For a ready-to-use Nile setup with the common fields already filled, you can extend the config like this:

```json
{
  "mcpServers": {
    "tronlink": {
      "command": "node",
      "args": ["dist/index.js"],
      "cwd": ".",
      "env": {
        "TRONLINK_EXTENSION_PATH": "/path/to/tronlink-extension/dist",
        "TL_MODE": "prod",
        "TL_HEADLESS": "false",
        "TL_TRONGRID_URL": "https://nile.trongrid.io",
        "AGENT_WALLET_PASSWORD": "your-wallet-password",
        "TL_SUNSWAP_ROUTER": "TKzxdSv2FZKQrEqkKVgp5DcwEXBEKMg2Ax",
        "TL_SUNSWAP_V3_ROUTER": "TB6xBCixqRPUSKiXb45ky1GhChFJ7qrfFj",
        "TL_MULTISIG_BASE_URL": "https://apinile.walletadapter.org",
        "TL_MULTISIG_SECRET_ID": "TEST",
        "TL_MULTISIG_SECRET_KEY": "TESTTESTTEST",
        "TL_MULTISIG_CHANNEL": "test",
        "TL_GASFREE_BASE_URL": "https://open-test.gasfree.io/nile/",
        "TL_GASFREE_API_KEY": "your_gasfree_api_key",
        "TL_GASFREE_API_SECRET": "your_gasfree_api_secret"
      }
    }
  }
}
```

If you only need direct API tools and do not need browser automation, you can keep the same structure and omit the Playwright-related fields:

```json
{
  "mcpServers": {
    "tronlink": {
      "command": "node",
      "args": ["dist/index.js"],
      "cwd": ".",
      "env": {
        "TL_TRONGRID_URL": "https://nile.trongrid.io",
        "AGENT_WALLET_PASSWORD": "your-wallet-password",
        "TL_MULTISIG_BASE_URL": "https://apinile.walletadapter.org",
        "TL_MULTISIG_SECRET_ID": "TEST",
        "TL_MULTISIG_SECRET_KEY": "TESTTESTTEST",
        "TL_MULTISIG_CHANNEL": "test",
        "TL_GASFREE_BASE_URL": "https://open-test.gasfree.io/nile/",
        "TL_GASFREE_API_KEY": "your_gasfree_api_key",
        "TL_GASFREE_API_SECRET": "your_gasfree_api_secret"
      }
    }
  }
}
```

**2. Claude Desktop**

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "tronlink": {
      "command": "node",
      "args": ["/absolute/path/to/dist/index.js"],
      "env": { ... }
    }
  }
}
```

**3. Claude Code Global Settings**

Edit `~/.claude/settings.json` or `.claude/settings.json`.

**4. Any MCP Client**

Supports stdio transport protocol — compatible with any MCP-compliant client.

---

## Project Structure

```
mcp-server-tronlink/
├── src/
│   ├── index.ts                    # Server entry, config, capability registration
│   ├── wallet.ts                   # Encrypted wallet loading and password handling
│   ├── wallet-tools.ts             # Wallet list/create/switch tools
│   ├── session-manager.ts          # Browser lifecycle (TronLinkSessionManager)
│   ├── capabilities/
│   │   ├── on-chain.ts             # 14 on-chain operations (TronGrid)
│   │   ├── multisig.ts             # 5 multi-sig operations (REST + WS)
│   │   ├── gasfree.ts              # 3 gas-free transfer operations
│   │   ├── build.ts                # Extension webpack build
│   │   ├── state-snapshot.ts       # UI state extraction
│   │   └── tron-crypto.ts          # Address derivation, signing, Base58
│   └── flows/
│       ├── index.ts                # Flow registry (32 recipes)
│       ├── switch-network.ts       # Network switching flows
│       ├── transfer-trx.ts         # Transfer flows
│       ├── multisig.ts             # 6 multi-sig flows
│       ├── onchain.ts              # 11 on-chain flows
│       └── gasfree.ts              # 3 gas-free flows
├── dist/                           # Compiled output
├── .mcp.json                       # MCP configuration
├── .env.example                    # Environment variable reference
├── package.json
├── tsconfig.json
└── README.md
```

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@noble/curves` | ^2.0.1 | secp256k1 ECDSA signing |
| `@noble/hashes` | ^2.0.1 | Keccak-256, SHA256 |
| `@tronlink/tronlink-mcp-core` | local | Core MCP server framework |
| `playwright` | ^1.49.0 | Browser automation |
| `@bankofai/agent-wallet` | latest | Encrypted local wallet management (`local_secure`) |
| `ws` | ^8.18.0 | WebSocket (multi-sig monitoring) |

---

## Security Model

| Aspect | Implementation |
|--------|----------------|
| Key storage | Encrypted local wallet managed by `@bankofai/agent-wallet` |
| Key exposure | No key material logged to stderr |
| Signing | Local transaction signing via encrypted `agent-wallet` — plain-text private keys are not supported |
| Pre-checks | All transactions validate before execution |
| Git safety | Config files in `.gitignore` prevent accidental commits |
| Default network | Nile testnet with safe defaults |

---

## Typical Usage Scenarios

1. **Wallet Operations** — List wallets, auto-create or switch the active wallet, check balance, send transfers
2. **DApp Testing** — Launch browser, connect wallet, sign transactions, verify state
3. **On-Chain Trading** — Direct API swaps, staking, token transfers without browser
4. **Multi-Sig Workflows** — Set up permissions, submit/monitor transactions
5. **Gas-Free Operations** — TRC20 transfers without TRX balance requirements
6. **Infrastructure Testing** — Contract deployment, fixture management, mock servers

---

## Quick Start

```bash
# 1. Build
npm install && npm run build

# 2. Configure .mcp.json (Nile testnet example)
# Add:
#   TL_TRONGRID_URL=https://nile.trongrid.io
#
# 3. If no wallet exists, choose one path:
#   Option A: call tl_wallet_create after startup
#   Option B: create one locally, then set AGENT_WALLET_PASSWORD

# 4. Use with Claude Code
# "Check my TRX balance"
# "Send 10 TRX to TAddress..."
# "Swap 100 TRX for USDT on SunSwap V3"
```
