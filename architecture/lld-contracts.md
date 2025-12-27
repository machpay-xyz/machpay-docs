# Low-Level Design: MachPay Contracts (Solana)

## 1. Introduction
The `machpay-contracts` (formerly `machpay-program`) repository contains the on-chain Settlement Engine for the MachPay Protocol, built using **Anchor (Rust)** on the Solana Blockchain.

It serves two primary roles:
1.  **Custody**: Holds Agent collateral (USDC/SOL) in a secure PDA (Global Bond).
2.  **Adjudication**: Verifies batches of off-chain signatures and executes transfers or slashings.

## 2. Architecture

### 2.1. Program Derived Addresses (PDAs)
We use PDAs to deterministically locate accounts without centralized indexing.

| Account Type | Seeds | Description |
| :--- | :--- | :--- |
| **GlobalBond** | `[b"bond", mint_address]` | The vault holding all collateral for a specific Token Mint (e.g., USDC). |
| **AgentAccount** | `[b"agent", authority_key]` | Stores Agent metadata (optional) and Bond balance tracking (if not using raw SPL). |
| **VendorAccount** | `[b"vendor", authority_key]` | Stores Vendor metadata (Service URL, Price, Treasury Address). |

### 2.2. Instructions

#### A. `initialize_bond`
Creates the Global Bond vault for a new Token Mint.
-   **Signer**: Protocol Admin.
-   **Accounts**: `bond_pda`, `mint`, `system_program`.

#### B. `register_vendor`
Registers a Vendor on-chain, making them discoverable.
-   **Signer**: Vendor Authority.
-   **Args**: `metadata_uri` (IPFS), `fee_per_unit`.
-   **Accounts**: `vendor_pda`, `authority`.

#### C. `settle_batch` (The Hot Path)
Processes a batch of off-chain Payment Intents.
-   **Signer**: Relayer (Recon Service).
-   **Args**:
    -   `intents`: `Vec<PaymentIntent>` (Custom Borsh struct).
    -   `signatures`: `Vec<[u8; 64]>`.
-   **Logic**:
    1.  **Loop**: Iterate through each intent.
    2.  **Verify**: `Ed25519_Verify(intent_hash, signature, agent_key)`.
    3.  **Check Replay**: Ensure `intent.nonce` > `agent.last_nonce` (tracked in specific State Account) OR verify against a `Bitmask` for parallel processing.
    4.  **Transfer**: Move tokens from `GlobalBond` -> `VendorTreasury`.
-   **Optimization**: Uses **Instruction Packing** to fit ~30 settlements per Tx.

#### D. `slash_equivocation`
Punishes double-spending.
-   **Signer**: Reporter (Anyone).
-   **Args**: `intent_A`, `sig_A`, `intent_B`, `sig_B`.
-   **Logic**:
    1.  Verify `A.nonce == B.nonce`.
    2.  Verify `A.hash != B.hash`.
    3.  Verify signatures.
    4.  **Burn**: 50% of Agent's Bond.
    5.  **Reward**: 50% to Signer.

## 3. Data Structures

### 3.1. PaymentIntent (Borsh)
Packed tightly for minimum CUs.
```rust
#[derive(AnchorSerialize, AnchorDeserialize)]
pub struct PaymentIntent {
    pub agent: Pubkey,
    pub vendor: Pubkey,
    pub amount: u64,
    pub nonce: u64,
    pub deadline: i64,
}
```

### 3.2. VendorAccount
```rust
#[account]
pub struct VendorAccount {
    pub authority: Pubkey,
    pub treasury: Pubkey,
    pub fee: u64,
    pub bump: u8,
}
```

## 4. Security Model
-   **Signatures**: Native Ed25519 verify (Cost: ~2000 CUs).
-   **Replay Protection**:
    -   *Strict*: Monotonic increasing nonce (Simple, but strictly serial).
    -   *Parallel*: Bitmask window (Allows out-of-order within a window). *Chosen: Strict for v1*.

## 5. Errors
-   `InvalidSignature`: 0x100
-   `ExpiredIntent`: 0x101
-   `InsufficientBond`: 0x102
-   `NonceReplay`: 0x103
