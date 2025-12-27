# Low-Level Design: MachPay Gateway (Solana)

## 1. Introduction
The `machpay-gateway` is a high-performance **Reverse Proxy** that acts as the "Payment Firewall" for vendors. It sits in front of existing APIs (e.g., standard REST/gRPC services) and enforcing micropayments via the MachPay Protocol.

**Key Goals**:
-   **Zero-Latency Overhead**: Verification must happen in sub-millisecond time.
-   **Statelessness**: Verification relies on cryptographic signatures, not DB lookups.
-   **Security**: Prevent replay attacks and insolvency.

## 2. Architecture

### 2.1. Technology Stack
-   **Language**: **Go** (Golang).
-   **Web Framework**: **Fiber** (v2) - chosen for high throughput/zero allocation.
-   **Solana Library**: `gagliardetto/solana-go` (for Ed25519 & RPC).
-   **Storage**: **Hybrid** - `redis` (Cluster) or `memory` (Standalone).
-   **Config**: Viper.

### 2.2. Request Flow (The Middleware Chain)
Every incoming request passes through a strict Fiber middleware pipeline:

1.  **Rate Limiter**: DDoS protection (Fiber limiter + Storage Backend).
2.  **402 Negotiator**: Checks for `x-machpay-auth`. If missing/invalid, calculates cost and returns `402 Payment Required`.
3.  **Authenticator**: Verifies Ed25519 Signature on the `PaymentIntent`.
4.  **Replay Guard**: Checks duplicate nonces in Storage (`SETNX` equivalent).
5.  **Solvency Checker**: Verifies Agent has sufficient funds (Optimistic Cache).
6.  **Proxy**: `fiber/proxy` forwards request to upstream Vendor API.
7.  **Async Settler**: Pushes `PaymentIntent` to a buffered channel -> Insert to DB/Queue.

## 3. Core Logic Sections

### 3.1. 402 Negotiation Middleware
**Input**: Incoming HTTP Request.
**Logic**:
1.  **Match Route**: Look up `path` in `pricing_rules.yaml`.
    -   Example: `POST /v1/chat -> 5000 units`.
2.  **Check Header**: Is `x-machpay-auth` present?
    -   **No**: Generate `nonce` (u64), construct `PaymentChallenge`, return HTTP 402.
    -   **Yes**: Decode Base58 signature and proceed to Verification.

### 3.2. Verification (Ed25519)
**Input**: `x-machpay-auth` (Signature), Request Context.
**Logic**:
1.  **Reconstruct Intent**:
    ```go
    intent := PaymentIntent{
        Discriminator: 0x01,
        Agent:         ctx.Get("x-machpay-agent"),
        Gateway:       config.VendorPubkey,
        Mint:          config.TokenMint, // Enforce configured mint
        Amount:        route.Cost,
        Nonce:         ctx.Get("x-machpay-nonce"),
        Deadline:      ctx.Get("x-machpay-deadline"),
    }
    ```
2.  **Verify**:
    `intent.Verify(signature)` (wraps `solana.VerifySignature`)
3.  **Failure**: Return `401 Unauthorized` ("Invalid Signature").

### 3.3. Replay Protection & Solvency
**Storage Backend**: Abstracted via `LiabilityRepository` (Redis or Memory).

**Replay Guard**:
-   **Key**: `nonce:{agent_id}:{nonce}`
-   **Value**: `1`
-   **TTL**: `60s` (Matches `deadline` tolerance).
-   *Atomic SETNX*: If key exists, reject.

**Solvency Check (Optimistic)**:
-   **Cache**: `balance:{agent_id}` -> `{ sol: u64, usdc: u64, last_update: u64 }`.
-   **Check**: `if cache.usdc < intent.amount { throw InsolvencyError }`.
-   **Sync Worker**: Independent thread loops through active agents and calls `getAccountInfo` every 10s to update Cache.

## 4. Configuration (`config.yaml`)

```yaml
server:
  port: 8080
  workers: 4

storage:
  type: "memory" # Options: "memory", "redis"

solana:
  network: "mainnet"
  rpc_urls: ["https://api.mainnet-beta.solana.com", "https://rpc.ankr.com/solana"]
  vendor_secret: "env:VENDOR_SECRET_KEY"

# Only required if storage.type is "redis"
redis:
  addr: "localhost:6379"
  db: 0
  password: "env:REDIS_PASSWORD"

pricing:
  default: 1000
  routes:
    - path: "/v1/chat/completions"
      method: "POST"
      cost: 5000
    - path: "/v1/images/*"
      cost: 10000

upstream:
  base_url: "http://localhost:3000" # The actual service
```

## 5. Telemetry (Prometheus)

| Metric | Type | Labels | Description |
| :--- | :--- | :--- | :--- |
| `gateway_requests_total` | Counter | `status`, `route` | Throughput |
| `gateway_latency_seconds` | Histogram | `route` | Processing time (exclude upstream) |
| `gateway_payment_failures` | Counter | `reason` | `signature_invalid`, `insolvent`, `replay` |
| `gateway_revenue_units` | Counter | `token` | Total accumulated value (unsettled) |

## 6. Console Integration & Events (High-Performance In-Memory)
To avoid external infrastructure dependencies (like Redis/Kafka) and minimize latency, the Gateway uses an **In-Memory Ring Buffer** (LMAX Disruptor pattern) to handle high-velocity events.

### 6.1. Architecture
1.  **Ring Buffer**: A fixed-size, pre-allocated circular buffer (e.g., 1024 slots) stores event pointers. Lock-free writes.
2.  **Batch Processor**: A background worker claims available events from the ring.
3.  **Dispatcher**: Aggregates events into a JSON Batch and POSTs to `machpay-backend` API.
    -   *Trigger*: Every 500ms OR when batch size > 100.
    -   *Fallback*: If backend is down, drop events (Gateway prioritizes traffic over telemetry).

### 6.2. Event Schemas (CloudEvents JSON)

#### A. PaymentAuthorized
Emitted when a request passes all checks and is proxied.
```json
{
  "type": "com.machpay.payment.authorized",
  "source": "gateway-01",
  "time": "2023-10-25T12:00:00Z",
  "data": {
    "agent_id": "Base58_Pubkey",
    "amount": 5000,
    "token": "USDC_Mint",
    "service_id": "openai-gpt4",
    "latency_ms": 45
  }
}
```

#### B. RequestBlocked
Emitted when verification fails (Security/solvency).
```json
{
  "type": "com.machpay.request.blocked",
  "source": "gateway-01",
  "data": {
    "agent_id": "Base58_Pubkey",
    "reason": "solvency_check_failed", // or "invalid_signature", "replay_detected"
    "current_balance": 400,
    "required_amount": 5000
  }
}
```

#### C. SolvencyAlert
Emitted when an Agent's balance drops below a critical threshold (Low Priority, sampled).
```json
{
  "type": "com.machpay.agent.insolvency_risk",
  "data": {
    "agent_id": "Base58_Pubkey",
    "remaining_balance": 15000,
    "burn_rate_1h": 50000
  }
}
```

## 7. CLI & Onboarding
The Gateway is distributed as a single binary (`machpay-gateway`) or Docker container.

### 7.1. Commands

#### A. Init (`machpay-gateway init`)
Interactive wizard to bootstrap configuration.
-   **Output**: `config.yaml`, `vendor-keypair.json` (if new).
-   **Prompts**: Network, RPC URL, Pricing Defaults.

#### B. Link (`machpay-gateway link`)
Initiates the handshake with the MachPay Console UI to associate this Gateway Key with your User Account.
-   **Output**: Display a pairing URL (e.g., `https://console.machpay.xyz/link?code=123...`).
-   **Security**: The Gateway signs a challenge. The Console maps `GatewayPubKey` -> `UserID`.
-   **Why**: Avoids managing OAuth tokens on the server. The Gateway uses its Keypair for auth.

#### B. Register (`machpay-gateway register`)
Registers the Vendor on the Solana Blockchain AND syncs with the Backend (Dual-Write).

**Workflow**:
1.  **Step 1 (Chain)**: Submits Solana Transaction to create `VendorAccount` PDA.
2.  **Step 2 (Sync)**:
    -   Extracts `tx_signature` and `pda_address`.
    -   Sends authenticated `POST /v1/vendor/endpoints` to MachPay Backend.
    -   Includes `X-MachPay-Vendor-Sig` header for proof of ownership.
3.  **Result**: Vendor is now visible in `client.searchVendors()`.

**Resilience**:
If Step 1 succeeds but Step 2 fails, the CLI outputs a specific **CURL command** for the user to manually trigger the sync.

#### C. Run (`machpay-gateway run`)
Starts the server.
-   **Flags**: `--config ./config.yaml`, `--port 8080`.
-   **Behavior**: Blocks process, starts HTTP server + Event Workers.

### 7.2. Onboarding Workflow (Unified Flow)

1.  **Install**:
    ```bash
    curl -sSf https://machpay.xyz/install.sh | sh
    ```

2.  **Initialize**:
    ```bash
    machpay-gateway init
    # -> Checks local key. Generates /secrets/vendor.json if missing.
    ```

3.  **Link (Pairing)**:
    ```bash
    machpay-gateway link
    # Open: https://console.machpay.xyz/link?code=XYZ...
    # -> User authenticates & creates "Vendor Profile" on Console if missing.
    # -> Linked to Vendor Profile: "Alice's shop"
    ```



4.  **Register (Sync)**:
    ```bash
    machpay-gateway register
    # Syncs local key with On-Chain PDA managed by Console.
    ```

4.  **Start**:
    ```bash
    machpay-gateway run
    # Listening on :8080
    ```

## 8. Chain Interaction & Telemetry

### 8.1. Chain Interaction Model
The Gateway splits logic between "Runtime" (High-Speed) and "Setup" (On-Chain).

| Component | Role | Action | Frequency |
| :--- | :--- | :--- | :--- |
| **Server** | Reader | Polls `getAccountInfo` via RPC to update Agent Solvency Cache. | Every 10s (Async) |
| **CLI** | Writer | Sends `InitializeVendor` Transaction to create PDA. | Once (Setup) |

### 8.2. Metrics Streaming (Console)
Similar to the Agent SDK, the Gateway aggregates performance data locally and streams it to the MachPay Backend (Receiver) for the **Vendor Dashboard**.

**Frequency**: Every 60 seconds (Configurable).
**Transport**: **HTTP Batch POST** (to Receiver) + **SSE** (from Receiver to Console).

#### A. GatewayTelemetryPacket Schema (Deep)
Matches `lld-receiver.md`.

```json
{
  "source": "gateway",
  "gateway_id": "Base58_Pubkey",
  "period_start": 1700000000,
  "period_end": 1700000060,
  "config": {
    "accepted_mint": "EPjFW...USDC...",
    "price_per_unit": 5000,
    "version": "v1.2"
  },
  "system": {
    "cpu_usage_percent": 12.5,
    "ram_usage_mb": 256,
    "goroutines": 450,
    "gc_pause_ns": 15000,
    "uptime_seconds": 86400
  },
  "network": {
    "bytes_in": 1048576,
    "bytes_out": 524288,
    "active_connections": 120
  },
  "business": {
    "total_revenue": 50000,
    "routes": {
      "/chat": { "200": 500, "402": 100, "500": 2, "latency_p99": 150 },
      "/image": { "200": 50, "402": 10, "500": 0, "latency_p99": 500 }
    },
    "solvency_rejects": 15,
    "replay_attacks_blocked": 5,
    "signature_failures": 2
  }
}
```
