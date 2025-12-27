# Low-Level Design: MachPay Recon (Solana)

## 1. Introduction
`machpay-recon` manages the settling of funds on the Solana Blockchain. Leveraging Solana's high speed and low cost, we can settle batches more frequently than on EVM chains.

## 2. Architecture

### 2.1. Aggregation Strategy
-   **Input**: Stream of valid `PaymentIntents`.
-   **Batching**: Group intents by `Agent` and `Gateway`.
-   **Netting**: Calculate `TotalAmount` per Agent->Gateway pair.

### 2.2. Settlement Execution
Instead of complex Merkle Roots, we can use **Instruction Packing**.
1.  **Construct Tx**: Create a transaction with multiple `Settle` instructions.
    -   Solana Tx Limit: ~1232 bytes (~20-30 simple instructions).
    -   Address Lookup Tables (LUTs): Use Versioned Transactions to pack more addresses.
2.  **Instruction**: `machpay_program::settle(ctx, amount, nonce_list)`.
3.  **Submit**: Send to RPC (Leader).

### 2.3. Risk Engine (The "Batch Microbatch" Lib)
Before any settlement batch is finalized, it passes through the Risk Engine.

#### A. Checks (Validation Layer)
1.  **Signature Validity**: `Ed25519_Verify(intent)`.
2.  **Nonce Linearity**: Ensure `Nonce` > `LastSettledNonce` (Anti-Replay).
3.  **Equivocation (Double Signing)**:
    -   *Check*: Does `Nonce N` exist for `Agent A` with different hash?
    -   *Result*: If yes, **Immediate Slash**.
4.  **Global Solvency (Defaulter Check)**:
    -   *Logic*: `Sum(PendingIntents_AllGateway) + Unsettled_Fees > OnChain_Bond`.
    -   *Result*: If insolvent, **Trigger Liquidation**.

### 2.4. Mitigation Flows

#### Flow 1: The "Defaulter" (Insolvency)
When an Agent overspends their Bond:
1.  **Freeze**: Recon stops accepting new intents for this Agent.
2.  **Liquidate**: Recon submits a special `Liquidate` transaction.
    -   **Action**: Transfers *all* remaining Bond to a "Settlement Pool".
3.  **Pro-Rata Payout**: Vendors are paid directly from the Pool based on % of claim. `Paid = Claim * (Bond / TotalClaims)`.

#### Flow 2: The "Malicious Actor" (Slashing)
When an Agent double-signs (equivocates):
1.  **Proof Construction**: Recon bundles the two conflicting signed intents.
2.  **Slash Transaction**: Calls `machpay_contracts::slash_equivocation(intent_A, intent_B)`.
3.  **On-Chain Action**:
    -   Verifies signatures.
    -   **Burns** 50% of Bond (Protocol Burn).
    -   **Rewards** 50% to Recon (Reporter Reward).
4.  **Blacklist**: Agent key is permanently banned.

### 2.5. Payout Worker (Orphaned Funds Resolver)
While `payment_intents` are high-frequency, Vendor Payouts are lower frequency (e.g., Daily/Weekly). The Backend creates these requests, and Recon executes them.

1.  **Poll**: Select rows from `payouts` where `status = 'PENDING'`.
    -   *Locking*: `SELECT ... FOR UPDATE SKIP LOCKED` (Prevents double-payouts).
2.  **Execute**:
    -   Constructs Solana Transfer Instruction (`GlobalBond` -> `Destination`).
    -   Signs with `RelayerKey` (as authorization).
    -   Submits to Chain.
3.  **Update DB**:
    -   **Success**: `status='CONFIRMED'`, `tx_hash='...'`.
    -   **Failure**: `status='FAILED'`, `error_msg='...'` (Backend alerts vendor).

## 3. Data Flow (The Pipeline)
1.  **Ingest**: `Receiver` -> `DB (intents table)`.
2.  **Risk Micro-Batcher (Every 500ms)**:
    -   Loads pending intents.
    -   Runs **Solvency Check** (Aggregating across all Gateways).
    -   Runs **Equivocation Check**.
3.  **Decision**:
    -   *Green*: Group into `SettlementBatch`.
    -   *Red*: Group into `SlashingBatch` or `LiquidationBatch`.
4.  **Execution**:
    -   Submit to Solana (Versioned Tx).
    -   **Result**:
        -   *Success*: `SETTLED`.
        -   *Failure (Generic)*: Retry (Network glitch).
        -   *Failure (0x102 Insufficient Bond)*:
            -   Mark `FAILED_NSF`.
            -   **Actions**:
                1.  Triggers `Liquidate` workflow for Agent.
                2.  **Emits Event**: `settlement.failed` (payload: `{ intent_id, vendor_id, amount }`).
                3.  *Backend consumes this to reverse the Vendor's credit.*
    -   Wait for Confirmation.
5.  **Finalize**: Update DB status.

## 4. Integration & Communication

### 4.1. The Shared Database Pattern
`machpay-recon` and `machpay-receiver` are decoupled services that communicate **strictly via PostgreSQL**. There is no direct HTTP/gRPC link between them.

1.  **Producer (`machpay-receiver`)**:
    -   Writes high-velocity `PaymentIntent` records.
    -   Status: `PENDING`.
    -   **Optimization**: Writes are "Fire and Forget" for the Receiver.

2.  **Consumer (`machpay-recon`)**:
    -   Polls for `PENDING` records (Interval: 500ms).
    -   **Locking**: Uses `SELECT ... FOR UPDATE SKIP LOCKED` to claim a batch of intents without blocking the Ingester.
    -   **State Transition**:
        -   `PENDING` -> `PROCESSING` (Tx submitted).
        -   `PROCESSING` -> `SETTLED` (Tx confirmed).
        -   `PROCESSING` -> `FAILED` (Tx dropped/reverted).

### 4.2. Database Schema (Shared)
Matches `lld-backend.md` and `lld-receiver.md`.

```sql
CREATE TABLE payment_intents (
    id UUID PRIMARY KEY,
    agent_pubkey VARCHAR(44),
    gateway_pubkey VARCHAR(44),
    amount_atomic NUMERIC(20,0),
    nonce NUMERIC(20,0),
    signature TEXT,
    status VARCHAR(20) DEFAULT 'PENDING',
    tx_signature VARCHAR(88), -- Populated by Recon
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payouts (
    id VARCHAR(64) PRIMARY KEY,
    vendor_id BIGINT,
    destination_wallet VARCHAR(44),
    amount_atomic NUMERIC(20,0),
    status VARCHAR(20) DEFAULT 'PENDING', -- PENDING -> CONFIRMED | FAILED
    tx_hash VARCHAR(88),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    error_msg TEXT
);
```

### 4.3. Notification (Optional)
On critical events (Slashing / Liquidation), Recon publishes a message to Redis/NATS for the Backend to consume (e.g., to send an email to the User).

## 5. Configuration
```yaml
recon:
  batch_interval_ms: 1000
  db_url: "postgres://..." # Shared DB
  solana:
    rpc_url: "..."
    payer_keypair: "..."
    priority_fee_microlamports: 5000
```
