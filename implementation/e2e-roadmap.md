# MachPay E2E Implementation Roadmap

> **Status:** Active Development  
> **Last Updated:** December 28, 2024  
> **Owner:** MachPay Core Team

---

## Executive Summary

This document outlines the phased implementation plan to complete the MachPay ecosystem, from E2E test coverage through production mainnet deployment.

### Current State (Completed ✅)

| Component | Status | Notes |
|-----------|--------|-------|
| Gateway (x402 Proxy) | ✅ Complete | Payment verification, telemetry, bond check |
| Backend API | ✅ Complete | Auth, agent/vendor dashboards, SSE |
| Receiver (Telemetry) | ✅ Complete | High-throughput ingestion |
| Python SDK | ✅ Complete | `pay_and_call()`, continuous E2E runner |
| Solana Contract | ✅ Built | Not deployed to localnet |
| E2E Docker Stack | ✅ Working | Agent → Gateway → Vendor flow |

### What's Missing

| Component | Priority | Phase |
|-----------|----------|-------|
| Agent Linking Flow | HIGH | 1 |
| Vendor Discovery API | MEDIUM | 1 |
| Alert System | LOW | 1 |
| Relayer Service | HIGH | 2 |
| On-Chain Bond Deposit | HIGH | 3 |
| Settlement Execution | HIGH | 3 |
| Slashing System | HIGH | 4 |
| Load Testing | HIGH | 5 |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              MACHPAY ECOSYSTEM                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────┐         ┌─────────────┐         ┌─────────────┐                   │
│   │   Agent     │ ──x402──│   Gateway   │────────▶│   Vendor    │                   │
│   │ (Python SDK)│         │  (Go Proxy) │         │   (API)     │                   │
│   └──────┬──────┘         └──────┬──────┘         └─────────────┘                   │
│          │                       │                                                   │
│          │                       ▼                                                   │
│          │                ┌─────────────┐                                           │
│          │                │   Relayer   │◄──── Phase 2                              │
│          │                │  (Go/Rust)  │                                           │
│          │                └──────┬──────┘                                           │
│          │                       │                                                   │
│          ▼                       ▼                                                   │
│   ┌─────────────┐         ┌─────────────┐                                           │
│   │  Receiver   │         │   Solana    │◄──── Phase 3                              │
│   │ (Telemetry) │         │  Program    │                                           │
│   └──────┬──────┘         └─────────────┘                                           │
│          │                                                                           │
│          ▼                                                                           │
│   ┌─────────────┐                                                                   │
│   │  Postgres   │                                                                   │
│   │  (Backend)  │                                                                   │
│   └─────────────┘                                                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: E2E Test Completeness

**Timeline:** 1-2 weeks  
**Goal:** All application-layer flows tested without on-chain settlement

### 1.1 Agent Linking Flow

**Priority:** HIGH  
**Effort:** 2 days

The agent linking flow allows users to associate their Python SDK agents with their MachPay console account for unified dashboard access.

#### Tasks

- [ ] E2E test: `POST /v1/link/init` returns challenge
- [ ] E2E test: `POST /v1/link/confirm` with signed message
- [ ] E2E test: `GET /v1/link/status` shows linked agent
- [ ] Python SDK: Add `link_to_user()` method
- [ ] Verify dashboard shows linked agent data

#### API Endpoints

```yaml
POST /v1/link/init:
  request:
    agent_pubkey: "BEMW59te5wTwwuAxPndA1dCgFAgLTTYQE9ub98Q9NVu4"
  response:
    challenge: "Sign this message to link agent: nonce=abc123"
    expires_at: 1735123456

POST /v1/link/confirm:
  request:
    agent_pubkey: "BEMW59te5wTwwuAxPndA1dCgFAgLTTYQE9ub98Q9NVu4"
    signature: "base64-ed25519-signature"
  response:
    status: "linked"
    user_id: "uuid"

GET /v1/link/status?agent=BEMW59...:
  response:
    linked: true
    user_id: "uuid"
    linked_at: "2024-12-28T00:00:00Z"
```

#### Files to Create/Modify

```
machpay-py/
├── src/machpay/client.py          # Add link_to_user(), get_link_status()
└── tests/e2e/test_linking.py      # E2E test

machpay-backend/
└── tests/e2e_linking_test.go      # Go E2E test
```

#### Python SDK Example

```python
async with MachPayClient(config) as client:
    # Get challenge from backend
    challenge = await client.init_link(user_jwt="eyJ...")
    
    # Sign challenge with agent key (automatic)
    result = await client.confirm_link(challenge)
    
    print(f"Agent linked to user: {result.user_id}")
```

---

### 1.2 Vendor Discovery API

**Priority:** MEDIUM  
**Effort:** 2 days

Enable agents to discover available paid APIs in the MachPay network.

#### Tasks

- [ ] Implement `GET /v1/vendors/search` endpoint
- [ ] Seed vendor registry in E2E database
- [ ] Aggregate `machpay.json` manifests from gateways
- [ ] Python SDK: Add `search_vendors()` method
- [ ] E2E test: Search returns registered vendors

#### API Endpoint

```yaml
GET /v1/vendors/search?q=weather&category=data:
  response:
    vendors:
      - id: "vendor-uuid"
        name: "WeatherAPI Pro"
        gateway_url: "https://weather.machpay.xyz"
        endpoints:
          - path: "/weather"
            price: 1000  # atomic units
            description: "Current weather data"
        categories: ["data", "weather"]
        rating: 4.8
        total_requests: 150000
```

#### Files to Create/Modify

```
machpay-backend/
├── internal/handlers/sql/vendor_discovery.go
├── internal/repository/vendor_registry.go
├── internal/db/sql/schema.sql  # Add vendor_registry table
└── tests/e2e_discovery_test.go

machpay-py/
├── src/machpay/client.py       # Add search_vendors()
└── tests/e2e/test_discovery.py
```

#### Python SDK Example

```python
async with MachPayClient(config) as client:
    vendors = await client.search_vendors(
        query="weather",
        category="data",
        max_price=5000,  # max 0.005 USDC per request
    )
    
    for vendor in vendors:
        print(f"{vendor.name}: {vendor.gateway_url}")
        for endpoint in vendor.endpoints:
            print(f"  {endpoint.path} - ${endpoint.price / 1_000_000:.4f}")
```

---

### 1.3 Alert System

**Priority:** LOW  
**Effort:** 1 day

Generate alerts when agents approach budget limits or when anomalies are detected.

#### Tasks

- [ ] Implement alert generation from telemetry worker
- [ ] Budget threshold alerts (80%, 100%)
- [ ] `GET /v1/agent/{id}/alerts` returns alerts
- [ ] E2E test: Trigger alert by exceeding budget

#### Alert Types

| Alert Type | Trigger | Severity |
|------------|---------|----------|
| `budget_warning` | 80% of daily limit | WARN |
| `budget_exceeded` | 100% of daily limit | ERROR |
| `vendor_price_change` | Vendor increased price >10% | INFO |
| `unusual_spending` | Spending 2x normal rate | WARN |

#### Files to Create/Modify

```
machpay-backend/
├── internal/tasks/alert_worker.go
├── internal/repository/alerts.go
├── internal/db/sql/schema.sql    # Add alerts table
└── tests/e2e_alerts_test.go
```

---

## Phase 2: Relayer MVP

**Timeline:** 2-3 weeks  
**Goal:** Enable cross-gateway coordination and settlement batching

### 2.1 Relayer Service Scaffold

**Priority:** HIGH  
**Effort:** 3 days

Create the central relayer service that coordinates multiple gateways.

#### Relayer Responsibilities

1. **Heartbeat Collection** - Track gateway health and availability
2. **Settlement Aggregation** - Receive and batch payment intents
3. **Liability Reporting** - Provide cross-gateway liability data
4. **Settlement Execution** - Submit batched settlements to Solana (Phase 3)

#### Tasks

- [ ] Create `machpay-relayer` repository (Go)
- [ ] `POST /heartbeat` - Receive gateway health
- [ ] `POST /settlements` - Receive settlement batches
- [ ] `GET /liability/{agent}` - Return cross-gateway liability
- [ ] Store settlements in Postgres (not settled yet)
- [ ] Add to `docker-compose.e2e.yml`

#### Repository Structure

```
machpay-relayer/
├── cmd/
│   └── relayer/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── handlers/
│   │   ├── heartbeat.go      # Gateway health check-ins
│   │   ├── settlements.go    # Receive settlement batches
│   │   └── liability.go      # Cross-gateway liability query
│   ├── repository/
│   │   ├── settlements.go    # Settlement storage
│   │   ├── gateways.go       # Gateway registry
│   │   └── liability.go      # Liability aggregation
│   └── worker/
│       ├── aggregator.go     # Batch settlements for execution
│       └── health_checker.go # Mark stale gateways as down
├── Dockerfile
├── docker-compose.yml
└── go.mod
```

#### API Endpoints

```yaml
POST /heartbeat:
  request:
    gateway_id: "F471gGhitv96pA6GeFRwku4Sdwi4bJN3yZ3NYkX9fn9x"
    version: "v1.0.0"
    uptime_seconds: 3600
    stats:
      requests_total: 15000
      requests_per_minute: 250
      active_agents: 45
  response:
    status: "ok"
    registered: true

POST /settlements:
  request:
    gateway_id: "F471gG..."
    batch:
      - agent: "BEMW59..."
        vendor: "F471gG..."
        amount: 1000
        nonce: 123456
        deadline: 1735123456
        signature: "base64..."
  response:
    accepted: 5
    rejected: 0

GET /liability/{agent}?exclude_gateway={gateway_id}:
  response:
    agent: "BEMW59..."
    external_liability: 50000  # Liability on OTHER gateways
    gateways:
      - gateway_id: "ABC..."
        liability: 30000
      - gateway_id: "DEF..."
        liability: 20000
```

---

### 2.2 Gateway → Relayer Integration

**Priority:** HIGH  
**Effort:** 2 days

Connect the gateway to the relayer for settlement and liability sync.

#### Tasks

- [ ] Point `MACHPAY_RELAYER_URL` to relayer in E2E
- [ ] Verify settlements flow to relayer
- [ ] Verify external liability returns correct values
- [ ] Test multi-gateway scenario (2 gateways, 1 relayer)
- [ ] Verify heartbeat registration works

#### Docker Compose Update

```yaml
# docker-compose.e2e.yml
services:
  relayer:
    build:
      context: ../machpay-relayer
      dockerfile: Dockerfile
    container_name: machpay-relayer-e2e
    ports:
      - "8083:8080"
    environment:
      DATABASE_URL: postgres://machpay_e2e:machpay_e2e_secret@postgres:5432/machpay_e2e_db?sslmode=disable
      LOG_LEVEL: debug
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - machpay-network

  gateway:
    environment:
      # Add relayer URL
      MACHPAY_RELAYER_URL: "http://relayer:8080"
```

#### E2E Test: Multi-Gateway

```go
// tests/e2e_multi_gateway_test.go
func TestMultiGatewayLiabilitySync(t *testing.T) {
    // 1. Agent pays 50 USDC through Gateway A
    // 2. Query relayer for agent liability (should be 50)
    // 3. Agent pays 30 USDC through Gateway B
    // 4. Query relayer for agent liability (should be 80)
    // 5. Gateway A queries external liability (should be 30)
    // 6. Gateway B queries external liability (should be 50)
}
```

---

### 2.3 Settlement File Recovery

**Priority:** MEDIUM  
**Effort:** 1 day

Handle recovery when settlements were written to disk due to relayer downtime.

#### Tasks

- [ ] CLI: `machpay-relayer import --file settlements.jsonl`
- [ ] Gateway startup: Check for unsynced settlements
- [ ] Retry mechanism for failed HTTP settlements
- [ ] E2E test: Recover settlements after relayer restart

#### CLI Command

```bash
# Import settlements from backup file
machpay-relayer import \
  --file /data/settlements.jsonl \
  --relayer http://relayer:8080 \
  --dry-run  # Preview what would be imported

# Output:
# Found 156 settlements in file
# - 142 already processed (skipping)
# - 14 new settlements to import
# 
# Importing... done
# Successfully imported 14 settlements
```

---

## Phase 3: On-Chain Integration

**Timeline:** 3-4 weeks  
**Goal:** Deploy Solana program and enable real settlement

### 3.1 Local Program Deployment

**Priority:** HIGH  
**Effort:** 3 days

Deploy the MachPay Solana program to the local test validator.

#### Tasks

- [ ] Add Anchor build to E2E pipeline
- [ ] Deploy `machpay-contract` to solana-test-validator
- [ ] Initialize GlobalBond PDA with test USDC mint
- [ ] Create test vendor account
- [ ] Verify program responds to RPC queries

#### Deployment Script

```bash
#!/bin/bash
# scripts/deploy-local.sh

set -e

echo "🔨 Building Anchor program..."
cd /workspace/machpay-contract
anchor build

echo "🚀 Deploying to local validator..."
anchor deploy --provider.cluster localnet

echo "📦 Initializing GlobalBond..."
npx ts-node scripts/initialize-bond.ts \
  --mint "$USDC_MINT" \
  --admin "$ADMIN_KEYPAIR"

echo "✅ Deployment complete!"
echo "Program ID: $(cat target/deploy/machpay_contracts-keypair.json | jq -r '.pubkey')"
```

#### Docker Compose Update

```yaml
# docker-compose.e2e.yml
services:
  contract-deployer:
    build:
      context: ../machpay-contract
      dockerfile: Dockerfile.deploy
    container_name: machpay-deployer-e2e
    environment:
      SOLANA_RPC_URL: "http://solana:8899"
      ADMIN_KEYPAIR: "/keys/admin.json"
    volumes:
      - ./tests/e2e_data:/keys:ro
    depends_on:
      solana:
        condition: service_healthy
    networks:
      - machpay-network
```

---

### 3.2 Agent Bond Deposit Flow

**Priority:** HIGH  
**Effort:** 3 days

Enable agents to deposit USDC as collateral for API payments.

#### Tasks

- [ ] Python SDK: `deposit_bond(amount)` method
- [ ] Create agent PDA on first deposit
- [ ] Transfer USDC to GlobalBond vault
- [ ] Gateway reads real on-chain balance
- [ ] E2E test: Deposit → Balance → Pay → Balance decreases

#### Python SDK Implementation

```python
# machpay-py/src/machpay/bond.py

from solders.keypair import Keypair
from solders.pubkey import Pubkey
from solana.rpc.async_api import AsyncClient
from anchorpy import Program, Provider

class BondManager:
    """Manage agent bond deposits and withdrawals."""
    
    def __init__(self, client: AsyncClient, program: Program, agent: Keypair):
        self.client = client
        self.program = program
        self.agent = agent
    
    async def deposit(self, amount: int) -> str:
        """
        Deposit USDC into the GlobalBond vault.
        
        Args:
            amount: Amount in atomic units (1 USDC = 1_000_000)
        
        Returns:
            Transaction signature
        """
        # Derive PDAs
        agent_pda = self._derive_agent_pda()
        global_bond_pda = self._derive_global_bond_pda()
        vault_pda = self._derive_vault_pda()
        
        # Build instruction
        ix = self.program.instruction["deposit_bond"](
            amount,
            ctx=Context(
                accounts={
                    "agent_account": agent_pda,
                    "global_bond": global_bond_pda,
                    "vault": vault_pda,
                    "agent_token_account": self._get_agent_token_account(),
                    "authority": self.agent.pubkey(),
                    "token_program": TOKEN_PROGRAM_ID,
                },
                signers=[self.agent],
            ),
        )
        
        # Send transaction
        tx = Transaction().add(ix)
        sig = await self.client.send_transaction(tx, self.agent)
        
        return str(sig)
    
    async def get_balance(self) -> int:
        """Get current bond balance for this agent."""
        agent_pda = self._derive_agent_pda()
        account = await self.program.account["AgentAccount"].fetch(agent_pda)
        return account.bond_balance
```

#### E2E Test

```python
# tests/e2e/test_bond_deposit.py

async def test_deposit_and_pay():
    """Test full bond deposit and payment flow."""
    
    # 1. Check initial balance (should be 0)
    initial_balance = await client.get_bond_balance()
    assert initial_balance == 0
    
    # 2. Deposit 10 USDC
    deposit_amount = 10_000_000  # 10 USDC
    tx_sig = await client.deposit_bond(deposit_amount)
    print(f"Deposit tx: {tx_sig}")
    
    # 3. Verify balance increased
    new_balance = await client.get_bond_balance()
    assert new_balance == deposit_amount
    
    # 4. Make a paid API call (costs 0.001 USDC)
    response = await client.pay_and_call(
        url="http://gateway:8080/weather",
        method="GET",
    )
    assert response.status_code == 200
    
    # 5. Verify bond balance decreased
    # Note: Actual settlement happens asynchronously
    # Gateway tracks liability locally until settlement
```

---

### 3.3 Settlement Execution

**Priority:** HIGH  
**Effort:** 4 days

Execute batched settlements on-chain to pay vendors.

#### Tasks

- [ ] Relayer: Batch settlements into `settle_cumulative()` tx
- [ ] Build Anchor instruction with all payment proofs
- [ ] Submit transaction to Solana
- [ ] Update vendor balances on-chain
- [ ] E2E test: Agent pays → Vendor balance increases

#### Settlement Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SETTLEMENT EXECUTION FLOW                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Gateways send settlements to Relayer                               │
│     ┌──────────┐                                                       │
│     │ Gateway  │──POST /settlements──▶┌──────────┐                     │
│     └──────────┘                      │ Relayer  │                     │
│                                       │          │                     │
│  2. Relayer batches by vendor         │  batch   │                     │
│     Agent A paid Vendor X: 1000       │    ↓     │                     │
│     Agent B paid Vendor X: 2000       │ aggregate│                     │
│     Agent C paid Vendor X: 500        └────┬─────┘                     │
│     ─────────────────────────────          │                           │
│     Total for Vendor X: 3500               │                           │
│                                            ▼                           │
│  3. Build settle_cumulative instruction                                │
│     ┌─────────────────────────────────────────────────────┐           │
│     │ settle_cumulative(                                   │           │
│     │   vendor: Vendor X,                                  │           │
│     │   total_amount: 3500,                                │           │
│     │   proofs: [                                          │           │
│     │     { agent: A, amount: 1000, sig: ... },           │           │
│     │     { agent: B, amount: 2000, sig: ... },           │           │
│     │     { agent: C, amount: 500, sig: ... },            │           │
│     │   ]                                                  │           │
│     │ )                                                    │           │
│     └─────────────────────────────────────────────────────┘           │
│                                            │                           │
│  4. Submit to Solana                       ▼                           │
│     ┌──────────────────────────────────────────────────────┐          │
│     │                    SOLANA                             │          │
│     │  ┌─────────────┐    ┌─────────────┐                  │          │
│     │  │ GlobalBond  │───▶│ Vendor ATA  │  +3500 USDC      │          │
│     │  │   Vault     │    │             │                  │          │
│     │  └─────────────┘    └─────────────┘                  │          │
│     │                                                       │          │
│     │  Agent A: bond -= 1000, liability = 0                │          │
│     │  Agent B: bond -= 2000, liability = 0                │          │
│     │  Agent C: bond -= 500, liability = 0                 │          │
│     └──────────────────────────────────────────────────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Relayer Settlement Executor

```go
// machpay-relayer/internal/settler/executor.go

type Executor struct {
    rpcClient  *rpc.Client
    program    solana.PublicKey
    adminKey   solana.PrivateKey
    logger     *zap.Logger
}

func (e *Executor) SettleBatch(ctx context.Context, batch *SettlementBatch) error {
    // 1. Build instruction
    ix, err := e.buildSettleInstruction(batch)
    if err != nil {
        return fmt.Errorf("build instruction: %w", err)
    }
    
    // 2. Create transaction
    recent, err := e.rpcClient.GetLatestBlockhash(ctx)
    if err != nil {
        return fmt.Errorf("get blockhash: %w", err)
    }
    
    tx, err := solana.NewTransaction(
        []solana.Instruction{ix},
        recent.Value.Blockhash,
        solana.TransactionPayer(e.adminKey.PublicKey()),
    )
    if err != nil {
        return fmt.Errorf("create tx: %w", err)
    }
    
    // 3. Sign and send
    tx.Sign(func(key solana.PublicKey) *solana.PrivateKey {
        if key == e.adminKey.PublicKey() {
            return &e.adminKey
        }
        return nil
    })
    
    sig, err := e.rpcClient.SendTransaction(ctx, tx)
    if err != nil {
        return fmt.Errorf("send tx: %w", err)
    }
    
    e.logger.Info("Settlement submitted",
        zap.String("signature", sig.String()),
        zap.Int("proofs", len(batch.Proofs)),
        zap.Uint64("total_amount", batch.TotalAmount),
    )
    
    return nil
}
```

---

### 3.4 Agent Withdrawal Flow

**Priority:** MEDIUM  
**Effort:** 2 days

Allow agents to withdraw unbonded USDC.

#### Tasks

- [ ] Python SDK: `withdraw_bond(amount)` method
- [ ] `withdraw_bond` instruction with timelock check
- [ ] E2E test: Deposit → Wait → Withdraw → Balance correct

#### Withdrawal Rules

1. **Minimum Bond** - Cannot withdraw below minimum required bond
2. **Timelock** - 24-hour delay after last payment (configurable)
3. **Liability Check** - Cannot withdraw if outstanding liability exists

```python
async def withdraw_bond(self, amount: int) -> str:
    """
    Withdraw USDC from bond.
    
    Args:
        amount: Amount to withdraw in atomic units
    
    Raises:
        InsufficientBondError: If withdrawal would go below minimum
        TimelockError: If timelock hasn't expired
        OutstandingLiabilityError: If there's unsettled liability
    
    Returns:
        Transaction signature
    """
    # Check eligibility
    bond_info = await self.get_bond_info()
    
    if bond_info.balance - amount < MINIMUM_BOND:
        raise InsufficientBondError(
            f"Cannot withdraw {amount}. Minimum bond: {MINIMUM_BOND}"
        )
    
    if bond_info.last_payment_at + TIMELOCK_SECONDS > time.time():
        raise TimelockError(
            f"Timelock expires at {bond_info.last_payment_at + TIMELOCK_SECONDS}"
        )
    
    if bond_info.liability > 0:
        raise OutstandingLiabilityError(
            f"Outstanding liability: {bond_info.liability}"
        )
    
    # Execute withdrawal
    return await self._execute_withdraw(amount)
```

---

## Phase 4: Security & Slashing

**Timeline:** 2 weeks  
**Goal:** Enable fraud detection and punishment

### 4.1 Double-Spend Detection

**Priority:** HIGH  
**Effort:** 3 days

Detect and punish agents who attempt to double-spend across gateways.

#### Tasks

- [ ] Relayer: Cross-check nonces across gateways
- [ ] Detect same nonce used on different gateways
- [ ] Build `slash_equivocation` instruction
- [ ] Submit slashing proof to chain
- [ ] E2E test: Simulate double-spend → Agent slashed

#### Double-Spend Detection Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOUBLE-SPEND DETECTION                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Honest Agent (Normal Flow):                                           │
│  ┌────────┐  nonce=1  ┌───────────┐                                    │
│  │ Agent  │──────────▶│ Gateway A │  ✅ Accepted                       │
│  └────────┘           └───────────┘                                    │
│       │                                                                 │
│       │     nonce=2  ┌───────────┐                                     │
│       └─────────────▶│ Gateway B │  ✅ Accepted                        │
│                      └───────────┘                                     │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  Malicious Agent (Double-Spend):                                       │
│  ┌────────┐  nonce=1  ┌───────────┐                                    │
│  │ Agent  │──────────▶│ Gateway A │  ✅ Accepted (first)               │
│  └────────┘           └───────────┘                                    │
│       │                     │                                          │
│       │     nonce=1        │ settlement                                │
│       └──────────┐         ▼                                           │
│                  │    ┌──────────┐                                     │
│                  │    │ Relayer  │◄─── Detects duplicate nonce!        │
│                  │    └────┬─────┘                                     │
│                  │         │                                           │
│                  ▼         │                                           │
│             ┌───────────┐  │                                           │
│             │ Gateway B │  │  ✅ Accepted (unaware)                    │
│             └───────────┘  │                                           │
│                  │         │                                           │
│                  │ settlement                                          │
│                  └────────▶│                                           │
│                            │                                           │
│                            ▼                                           │
│                    ┌──────────────┐                                    │
│                    │ SLASH AGENT  │                                    │
│                    │              │                                    │
│                    │ • Forfeit bond                                    │
│                    │ • Pay reporter reward                             │
│                    │ • Blacklist agent                                 │
│                    └──────────────┘                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Slashing Proof Structure

```rust
// machpay-contract/programs/machpay-contracts/src/lib.rs

#[derive(AnchorSerialize, AnchorDeserialize)]
pub struct EquivocationProof {
    /// First signed payment intent
    pub intent_a: PaymentIntent,
    pub signature_a: [u8; 64],
    pub gateway_a: Pubkey,
    
    /// Second signed payment intent (same nonce, different gateway)
    pub intent_b: PaymentIntent,
    pub signature_b: [u8; 64],
    pub gateway_b: Pubkey,
}

pub fn slash_equivocation(
    ctx: Context<SlashEquivocation>,
    proof: EquivocationProof,
) -> Result<()> {
    // 1. Verify both signatures are valid
    require!(
        verify_signature(&proof.intent_a, &proof.signature_a),
        ErrorCode::InvalidSignature
    );
    require!(
        verify_signature(&proof.intent_b, &proof.signature_b),
        ErrorCode::InvalidSignature
    );
    
    // 2. Verify same agent, same nonce, different gateways
    require!(
        proof.intent_a.agent == proof.intent_b.agent,
        ErrorCode::DifferentAgents
    );
    require!(
        proof.intent_a.nonce == proof.intent_b.nonce,
        ErrorCode::DifferentNonces
    );
    require!(
        proof.gateway_a != proof.gateway_b,
        ErrorCode::SameGateway
    );
    
    // 3. Slash agent's bond
    let slash_amount = ctx.accounts.agent_account.bond_balance;
    ctx.accounts.agent_account.bond_balance = 0;
    ctx.accounts.agent_account.is_slashed = true;
    
    // 4. Reward reporter (10% of slashed amount)
    let reporter_reward = slash_amount / 10;
    transfer_tokens(
        &ctx.accounts.vault,
        &ctx.accounts.reporter_token_account,
        reporter_reward,
    )?;
    
    emit!(AgentSlashed {
        agent: proof.intent_a.agent,
        amount: slash_amount,
        reporter: ctx.accounts.reporter.key(),
        reason: "equivocation".to_string(),
    });
    
    Ok(())
}
```

---

### 4.2 Gateway Misbehavior Detection

**Priority:** MEDIUM  
**Effort:** 2 days

Detect and report misbehaving gateways.

#### Misbehavior Types

| Type | Detection Method | Consequence |
|------|------------------|-------------|
| Invalid signature acceptance | Settlement proof check | Gateway blacklist |
| Over-reporting liability | Cross-reference with other gateways | Reputation penalty |
| Selective denial | Agent complaints + proof | Investigation |
| Settlement manipulation | On-chain proof mismatch | Severe penalty |

---

## Phase 5: Production Hardening

**Timeline:** 2-3 weeks  
**Goal:** Stress testing and mainnet readiness

### 5.1 Load Testing

**Priority:** HIGH  
**Effort:** 3 days

Verify system can handle production load.

#### Target Metrics

| Metric | Target | Current |
|--------|--------|---------|
| Requests per second | 1,000 | TBD |
| Latency p50 | <50ms | ~50ms |
| Latency p99 | <200ms | TBD |
| Settlement throughput | 100 tx/min | TBD |

#### Load Test Script

```go
// machpay-gateway/cmd/loadtest/main.go

func main() {
    // Configuration
    targetRPS := 1000
    duration := 5 * time.Minute
    agents := generateAgents(100) // 100 concurrent agents
    
    // Metrics
    var (
        successCount atomic.Uint64
        errorCount   atomic.Uint64
        latencies    []time.Duration
    )
    
    // Run load test
    start := time.Now()
    var wg sync.WaitGroup
    
    for _, agent := range agents {
        wg.Add(1)
        go func(a *Agent) {
            defer wg.Done()
            ticker := time.NewTicker(time.Second / time.Duration(targetRPS/len(agents)))
            defer ticker.Stop()
            
            for {
                select {
                case <-ticker.C:
                    if time.Since(start) > duration {
                        return
                    }
                    
                    reqStart := time.Now()
                    err := a.MakePayment()
                    latency := time.Since(reqStart)
                    
                    if err != nil {
                        errorCount.Add(1)
                    } else {
                        successCount.Add(1)
                        latencies = append(latencies, latency)
                    }
                }
            }
        }(agent)
    }
    
    wg.Wait()
    
    // Report results
    printResults(successCount.Load(), errorCount.Load(), latencies)
}
```

---

### 5.2 Mainnet Deployment

**Priority:** HIGH  
**Effort:** 5 days

Deploy to Solana mainnet for production use.

#### Deployment Checklist

- [ ] Audit smart contract code
- [ ] Deploy contract to mainnet
- [ ] Initialize GlobalBond with real USDC mint
- [ ] Configure production relayer
- [ ] Set up monitoring (Grafana, PagerDuty)
- [ ] Create operations runbook
- [ ] Test with small amounts first
- [ ] Gradual rollout (beta users)

---

## Summary Timeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        IMPLEMENTATION TIMELINE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Week 1-2:  ████████░░░░░░░░░░░░  Phase 1: E2E Completeness            │
│             • Agent Linking                                             │
│             • Vendor Discovery                                          │
│             • Alert System                                              │
│                                                                         │
│  Week 3-5:  ░░░░░░░░████████████  Phase 2: Relayer MVP                 │
│             • Relayer Service                                           │
│             • Gateway Integration                                       │
│             • Settlement Recovery                                       │
│                                                                         │
│  Week 6-9:  ░░░░░░░░░░░░░░░░░░██████████████  Phase 3: On-Chain        │
│             • Local Program Deployment                                  │
│             • Agent Bond Deposit                                        │
│             • Settlement Execution                                      │
│             • Agent Withdrawal                                          │
│                                                                         │
│  Week 10-11: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████  Phase 4: Security │
│             • Double-Spend Detection                                    │
│             • Gateway Audit                                             │
│                                                                         │
│  Week 12-14: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████  Phase 5   │
│             • Load Testing                                              │
│             • Mainnet Deployment                                        │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│  Total: ~14 weeks (3.5 months) for full production readiness           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Appendix: File Inventory

### New Repositories

| Repo | Purpose | Phase |
|------|---------|-------|
| `machpay-relayer` | Settlement aggregation & execution | 2 |

### New Files by Phase

#### Phase 1
```
machpay-py/
├── src/machpay/client.py          # Add link_to_user(), search_vendors()
├── tests/e2e/test_linking.py
└── tests/e2e/test_discovery.py

machpay-backend/
├── internal/handlers/sql/vendor_discovery.go
├── internal/repository/vendor_registry.go
├── internal/tasks/alert_worker.go
├── internal/repository/alerts.go
├── tests/e2e_linking_test.go
├── tests/e2e_discovery_test.go
└── tests/e2e_alerts_test.go
```

#### Phase 2
```
machpay-relayer/
├── cmd/relayer/main.go
├── internal/handlers/heartbeat.go
├── internal/handlers/settlements.go
├── internal/handlers/liability.go
├── internal/repository/settlements.go
├── internal/repository/gateways.go
├── internal/worker/aggregator.go
├── Dockerfile
└── go.mod

machpay-backend/
└── docker-compose.e2e.yml         # Add relayer service
```

#### Phase 3
```
machpay-py/
├── src/machpay/bond.py
└── tests/e2e/test_bond_deposit.py

machpay-relayer/
├── internal/settler/executor.go
└── internal/settler/instruction_builder.go

machpay-contract/
├── scripts/deploy-local.sh
├── scripts/initialize-bond.ts
└── Dockerfile.deploy
```

#### Phase 4
```
machpay-relayer/
├── internal/fraud/detector.go
├── internal/fraud/slasher.go
└── internal/reputation/scorer.go
```

#### Phase 5
```
machpay-gateway/
└── cmd/loadtest/main.go

docs/
├── operations/runbook.md
└── operations/monitoring.md
```

---

## Next Steps

1. **Review this plan** with the team
2. **Create GitHub issues** for Phase 1 tasks
3. **Start Phase 1.1** (Agent Linking) immediately
4. **Begin Phase 2.1** (Relayer scaffold) in parallel

---

*Document maintained by MachPay Core Team*





