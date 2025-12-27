# Low-Level Design: MachPay Agent SDK (Solana)

## 1. Overview
The MachPay Agent SDK (`@machpay/sdk-solana` / `machpay-solana`) enables autonomous agents to interact with the MachPay Protocol on the Solana Blockchain (SVM). It handles Ed25519 signing, PDA discovery, and high-throughput payment negotiation.

## 2. Onboarding Workflow

### 2.1. Identity Management
Agents are identified by a **Solana Keypair** (Ed25519).
-   **Generation**: `Keypair.generate()` (returns a 64-byte secret key).
-   **Storage**:
    -   **Dev**: Local file `~/.config/solana/id.json` or `.env` variable `MACHPAY_AGENT_SECRET`.
    -   **Prod**: AWS KMS / Vault with a signing oracle wrapper.

### 2.2. Funding & Capital Management
The primary failure mode for autonomous agents is "Capital starvation".

#### A. Asset Requirements
1.  **SOL**: Rent-exemption + Gas.
2.  **SPL Tokens (USDC)**: Vendor payments.

#### B. Monitoring Strategy
Agents are long-running processes; they must be monitored externally.
1.  **Console Alerts**:
    -   User configures `LowBalanceThreshold` (e.g., < $5.00) in MachPay Console.
    -   MachPay Indexer watches Agent Account balance.
    -   **Trigger**: Sends Email/Webhook/SMS when `balance < threshold`.
2.  **SDK Self-Protection**:
    -   `client.on("low_balance", (balance) => { ... })` event.
    -   If `balance < payment_amount`, `signPayment` throws `InsufficientFundsError`.
    -   **Recovery**: Agent can optionally trigger a "Top-Up" via an external exchange API (e.g., Coinbase) or notify the operator.
3.  **TGS (Token Gate Service)**:
    -   The Dashboard acts as the TGS. It visually graphs the "Burn Rate" (Spend/Hour).
    -   Predicts "Runway" (Time until insolvency) based on moving average.

### 2.3. Wallet Linking (Unified Flow)
Connecting the autonomous SDK Agent to the MachPay Console.

**Workflow**:
1.  **Init (CLI)**: Developer runs `machpay agent init` (or `client.init()`). Generates local `secret_key.json`.
2.  **Link (CLI/Lib)**: Run `machpay agent link` (or `client.link()`).
3.  **Code**: Output displays Pairing Code & URL (e.g., `https://console.machpay.xyz/link?code=XYZ`).
4.  **Pair (UI)**:
    -   User opens URL.
    -   **If New**: Prompts to create "Agent Profile" & Deposit Funds.
    -   **If Existing**: Selects Profile to link.
5.  **Result**: Backend maps `DeviceKey` to `AgentProfile`. Console executes tracking.

This enables the Console to track the Agent's remote activity.

## 3. SDK CLI (`@machpay/cli`)
The SDK includes a CLI tool for management without writing code.

`machpay agent init`      # Generate Identity
`machpay agent link`      # Start Pairing Flow
`machpay agent balance`   # Check Sol/USDC Balance
`machpay agent logs`      # Stream remote logs (via Receiver)

## 3. Core Architecture

### 3.1. Interface (TypeScript)
```typescript
interface MachPayConfig {
  network: "mainnet-beta" | "devnet";
  rpcUrl: string;
  secretKey?: Uint8Array; // If local
}

class MachPayClient {
  private connection: Connection;
  private agent: Keypair;

  constructor(config: MachPayConfig) { ... }

  /**
   * Discovers a vendor's on-chain config via their ServiceID.
   */
  async resolveVendor(serviceId: string): Promise<VendorConfig>;

  /**
   * Semantic Search: Finds vendors based on natural language query & filters.
   * Enables M2M discovery (e.g. "Find me a cheap LLM").
   */
  async searchVendors(query: string, opts?: { maxPrice?: number }): Promise<VendorConfig[]>;

  /**
   * Signs a PaymentIntent for a specific vendor.
   * Checks local solvency before signing.
   */
  async signPayment(vendor: PublicKey, amount: number): Promise<SignedIntent>;

  /**
   * Convenience wrapper: Handles the full 402 loop for a generic HTTP request.
   * Retries automatically with payment headers.
   */
  async authorizeRequest(fn: () => Promise<Response>): Promise<Response>;

  /**
   * Generates a pairing URL to link this agent to the Console UI.
   * Prints URL to stdout and returns it.
   */
  async link(): Promise<string>;

  /**
   * Checks the agent's current on-chain balance (SOL & SPL).
   */
  async getBalance(): Promise<{ sol: number; usdc: number }>;
}

```

### 3.2. Data Models

### 3.2. Data Models (Schemas)

#### A. PaymentIntent (Binary - Borsh)
Used for payment authorization.
| Field | Type | Size | Description |
| :--- | :--- | :--- | :--- |
| `discriminator` | `u8` | 1 | `0x01` |
| `agent` | `Pubkey` | 32 | Payer |
| `gateway` | `Pubkey` | 32 | Payee |
| `mint` | `Pubkey` | 32 | USDC/Token Mint |
| `amount` | `u64` | 8 | Atomic units |
| `nonce` | `u64` | 8 | Challenge |
| `deadline` | `i64` | 8 | Unix timestamp |

#### B. WalletAuthPayload (JSON)
Used for `linkWallet` (SIWS).
```json
{
  "domain": "console.machpay.xyz",
  "address": "Base58_Pubkey",
  "statement": "Sign in to MachPay Console",
  "nonce": "Random_String_From_Server",
  "issuedAt": "ISO_Timestamp"
}
```

#### C. VendorConfig (On-Chain PDA)
Returned by `resolveVendor`.
```rust
struct VendorAccount {
    authority: Pubkey,      // owner
    fee_per_unit: u64,      // price
    treasury_mint: Pubkey,  // USDC/SOL mint
    metadata_uri: String,   // IPFS link to description
    bump: u8                // PDA bump
}
```

#### D. PaymentChallenge (HTTP 402 Body)
Returned by Gateway.
```json
{
  "error": "Payment Required",
  "types": ["MachPay-v1"],
  "params": {
    "nonce": "18374712",
    "cost": 5000,
    "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "gateway_id": "Base58_Pubkey"
  }
}
```

### 3.3. Telemetry & Metrics (Agent-Level Batching)
The SDK aggregates performance data locally and emits a consolidated packet per batch interval (e.g., 60s). This allows the network to track Vendor performance/reliability.

#### A. Metric Definitions
| Metric | Type | Unit | Description |
| :--- | :--- | :--- | :--- |
| `calls_last_10s` | Gauge | count | Rolling throughput snapshot (last 10s bucket) |
| `latency_p50` | Gauge | ms | Median latency of 402 negotiation -> success |
| `latency_p99` | Gauge | ms | Tail latency |
| `success_rate` | Gauge | % | (Success / Total Attempts) * 100 |
| `dropped_requests` | Counter | count | Requests failed due to local timeout/error |

#### B. Batch Packet Schema (Deep)
Matches `lld-receiver.md`.

```json
{
  "source": "sdk",
  "agent_id": "Base58_Pubkey",
  "period_start": 1700000000,
  "period_end": 1700000060,
  "wallet": {
    "sol_balance_lamports": 50000000,
    "usdc_balance_atomic": 2000000,
    "low_balance_warnings": 0
  },
  "interactions": {
    "Vendor_A": {
      "attempts": 100,
      "success": 98,
      "payment_latency_p99": 200,
      "vendor_latency_p99": 500,
      "errors": { "timeout": 1, "402_loop_failed": 1 },
      "spend_atomic": 50000
    }
  }
}
```

### 3.4. Telemetry Buffer
**Logic**:
1.  **Accumulate**: Store raw events in a ring buffer.
2.  **Compute**: On `flush_interval` (default 60s), calculate P50/P99 locally to save bandwidth.
3.  **Emit**: POST the summary packet to `machpay-receiver`.

## 4. Payment Flow (The 402 Loop)

1.  **Request**: `POST /chat/completions` -> Gateway.
2.  **Challenge**: Gateway returns `402 Payment Required`.
    -   Header `x-machpay-nonce`: `18374712` (Random u64).
    -   Header `x-machpay-cost`: `5000` (Microlamports or SPL atomic units).
3.  **Signing**:
    -   SDK constructs `PaymentIntent` with `nonce=18374712`.
    -   SDK checks local balance (does not query chain to save latency).
    -   SDK signs intent.
4.  **Resubmit**: `POST` with header `x-machpay-auth: <base58_signature>`.

## 5. Library Specifics
-   **Python**: Built on `solana-py` and `solders`.
-   **JS/TS**: Built on `@solana/web3.js` and `@coral-xyz/anchor`.

## 6. Quickstart: Zero to Ready

### Step 1: Initialization
```typescript
import { MachPayClient } from "@machpay/sdk-solana";

// 1. Load your keypair (or generate one)
const secret = process.env.MACHPAY_SECRET ? 
  Uint8Array.from(JSON.parse(process.env.MACHPAY_SECRET)) : 
  undefined;

// 2. Initialize Client
const client = new MachPayClient({
  network: "mainnet-beta",
  rpcUrl: "https://api.mainnet-beta.solana.com",
  secretKey: secret
});

// 3. Link to Console (One-time setup)
const linkUrl = await client.link();
console.log("Link this agent:", linkUrl);

// 4. Fund Check
const balance = await client.getBalance();
if (balance.usdc < 1.0) {
  console.log("Please fund agent:", client.agent.publicKey.toBase58());
}
```

### Step 2: Discovery
```typescript
// Find a vendor by their readable ID
const vendorId = "openai-gpt4-turbo";
const config = await client.resolveVendor(vendorId);

console.log(`Found Vendor: ${config.authority}`);
console.log(`Price per req: ${config.fee_per_unit} units`);
```

### Step 3: Making a Paid Request
The SDK handles the 402 loop automatically.

```typescript
// 1. Define your standard API call
const makeCall = async () => {
  return await fetch("https://gateway.vendor.xyz/v1/chat/completions", {
    method: "POST",
    body: JSON.stringify({ model: "gpt-4", messages: [...] })
  });
};

// 2. Wrap it with MachPay Authorization
const response = await client.authorizeRequest(makeCall);

if (response.ok) {
  const data = await response.json();
  console.log("Success:", data);
}
```

### Step 4: Semantic Search (M2M)
Agents can discover vendors dynamically without knowing their IDs beforehand.

```typescript
// 1. Natural Language Query
const query = "Find me a weather API for New York with >99% uptime";

// 2. Search the MCP Marketplace
const results = await client.searchVendors(query, { maxPrice: 0.01 });

// 3. Select the best match
const bestVendor = results[0];

if (bestVendor) {
  console.log(`Matched: ${bestVendor.metadata.name}`);
  console.log(`Cost: ${bestVendor.fee_per_unit} | Latency: ${bestVendor.metrics.latency_p50}ms`);
  
  // 4. Immediately Transact
  await client.signPayment(bestVendor.authority, bestVendor.fee_per_unit);
}
```
