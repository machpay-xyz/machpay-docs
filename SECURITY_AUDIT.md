# MachPay Security & Architecture Audit

**Date:** December 29, 2024  
**Status:** Active  
**Severity Legend:** 🔴 Critical | 🟠 High | 🟡 Medium | ⚪ Low

---

## Executive Summary

This document outlines security vulnerabilities and architectural issues identified across the MachPay platform. Issues are categorized by component and prioritized by severity and impact on the protocol's integrity.

**Critical Issues:** 2  
**High Issues:** 2  
**Medium Issues:** 4  
**Low Issues:** 2

---

## Table of Contents

1. [Critical: Strict Nonce Concurrency Trap](#1-critical-strict-nonce-concurrency-trap)
2. [Critical: Distributed Double-Spend Vulnerability](#2-critical-distributed-double-spend-vulnerability)
3. [High: Inline Sync Denial-of-Service](#3-high-inline-sync-denial-of-service)
4. [High: Telemetry Data Loss](#4-high-telemetry-data-loss)
5. [Medium: SDK Nonce State Drift](#5-medium-sdk-nonce-state-drift)
6. [Medium: Internal Error Leakage](#6-medium-internal-error-leakage)
7. [Medium: Gas Inefficiency](#7-medium-gas-inefficiency)
8. [Medium: Hardcoded Capabilities Manifest](#8-medium-hardcoded-capabilities-manifest)
9. [Low: Configuration Drift](#9-low-configuration-drift)
10. [Low: Naming Confusion](#10-low-naming-confusion)

---

## 1. Critical: Strict Nonce Concurrency Trap

**Component:** `machpay-contract`  
**File:** `programs/machpay-contracts/src/lib.rs` (Line 257)  
**Status:** 🔴 Open

### Issue

The on-chain contract enforces strictly increasing nonces for replay protection:

```rust
require!(
    state.nonce > agent_account.last_settled_nonce,
    MachPayError::NonceReplay
);
```

### Attack Scenario

1. Agent sends **Nonce 10** to Vendor A
2. Agent sends **Nonce 11** to Vendor B
3. Relayer settles Vendor B first → Contract sets `last_settled_nonce = 11`
4. Relayer attempts to settle Vendor A (Nonce 10) → **REJECTED** (10 < 11)

### Impact

- **Vendor A never gets paid**
- Agents cannot use multiple vendors simultaneously
- Protocol is fundamentally broken for multi-vendor use cases

### Proposed Fix

Implement **Bitmask / Sliding Window** replay protection:

```rust
// Instead of: nonce > last_settled_nonce
// Use: Bitmap tracking for nonces within a window

pub struct AgentAccount {
    // ... existing fields
    pub nonce_window_start: u64,      // Start of the 256-nonce window
    pub nonce_bitmap: [u64; 4],       // 256-bit bitmap (4 x u64)
}

fn is_nonce_used(account: &AgentAccount, nonce: u64) -> bool {
    if nonce < account.nonce_window_start {
        return true; // Too old, considered used
    }
    let offset = nonce - account.nonce_window_start;
    if offset >= 256 {
        return false; // Beyond window, allow (will shift window)
    }
    let word = (offset / 64) as usize;
    let bit = offset % 64;
    (account.nonce_bitmap[word] & (1 << bit)) != 0
}
```

### Tasks

- [ ] Design sliding window replay protection (256-nonce window)
- [ ] Update `AgentAccount` struct with bitmap fields
- [ ] Modify `settle` instruction to use bitmap checks
- [ ] Add window sliding logic when nonces exceed current window
- [ ] Update SDK to request available nonces from Gateway
- [ ] Comprehensive test coverage for out-of-order settlements

---

## 2. Critical: Distributed Double-Spend Vulnerability

**Component:** `machpay-gateway`  
**File:** `internal/adapter/secondary/repository/factory.go`  
**Status:** 🔴 Open

### Issue

The Gateway defaults to SQLite for liability tracking when Redis is not configured:

```go
func NewLiabilityRepository(redisAddr string) (ports.LiabilityRepository, error) {
    if redisAddr == "" {
        store, err := sqlite.NewSQLiteStore("machpay.db")
        // ...
    }
}
```

### Attack Scenario

1. Vendor runs 2+ Gateway instances behind a load balancer (for uptime)
2. Instances do NOT share SQLite state (file-based, local)
3. Attacker connects to Instance A → spends full bond
4. Attacker connects to Instance B → spends bond AGAIN
5. Relayer settles both → Vendor loses money

### Impact

- Vendors lose funds equal to the double-spent amount
- Trust in the platform is destroyed

### Proposed Fix

**Option A: Force Shared Storage in Production**

```go
// cmd/gateway/main.go
func validateConfig(cfg *config.Config) error {
    if cfg.Environment == "production" && cfg.RedisAddr == "" {
        return fmt.Errorf("FATAL: Redis/Postgres required for production. SQLite is single-instance only")
    }
    return nil
}
```

**Option B: Add Startup Warning + Documentation**

```go
if redisAddr == "" {
    log.Warn("⚠️  Running in SQLite mode - DO NOT use with multiple instances")
    log.Warn("⚠️  Configure MACHPAY_REDIS_ADDR for production deployments")
}
```

### Tasks

- [ ] Add environment validation in `cmd/gateway/main.go`
- [ ] Fail startup if `ENV=production` and no Redis configured
- [ ] Add prominent warning in documentation
- [ ] Add health check endpoint that reports storage mode

---

## 3. High: Inline Sync Denial-of-Service

**Component:** `machpay-gateway`  
**File:** `internal/service/proxy/handler.go` (Line 63)  
**Status:** 🟠 Open

### Issue

Bond synchronization for new users is protected by a small semaphore:

```go
syncSem: make(chan struct{}, 20), // Allow 20 concurrent inline syncs
```

### Attack Scenario

1. Attacker generates 21+ unique Agent keypairs
2. Sends requests from all agents simultaneously
3. Semaphore (20 slots) is exhausted
4. All legitimate new users receive `503 Service Unavailable`

### Impact

- New users cannot onboard during attack
- Vendor loses revenue from legitimate traffic

### Proposed Fix

**Option A: Background Worker Pool**

```go
type BondSyncWorker struct {
    queue    chan SyncRequest
    workers  int
    rpcLimit *rate.Limiter
}

func (w *BondSyncWorker) EnqueueSync(agentID string) <-chan SyncResult {
    result := make(chan SyncResult, 1)
    select {
    case w.queue <- SyncRequest{AgentID: agentID, Result: result}:
        return result
    default:
        // Queue full - return cached/stale data or reject
        result <- SyncResult{Error: ErrSyncQueueFull}
        return result
    }
}
```

**Option B: Per-IP Rate Limiting on Sync Path**

```go
// Add IP-based rate limiting before semaphore
if !syncRateLimiter.Allow(c.IP()) {
    return fiber.ErrTooManyRequests
}
```

### Tasks

- [ ] Implement background sync worker pool
- [ ] Add per-IP rate limiting on sync endpoint
- [ ] Return cached/estimated bond for repeat offenders
- [ ] Add metrics for sync queue depth and rejections

---

## 4. High: Telemetry Data Loss

**Component:** `machpay-gateway`  
**File:** `internal/worker/telemetry.go` (Line 94-106)  
**Status:** 🟠 Open

### Issue

The telemetry worker uses a non-blocking drop strategy:

```go
func (tw *TelemetryWorker) Dispatch(event *domain.TelemetryEvent) {
    select {
    case tw.events <- event:
        tw.eventsDispatched.Add(1)
    default:
        // Channel full - drop event
        tw.eventsDropped.Add(1)
    }
}
```

If the backend receiver is unreachable, **100% of telemetry is silently dropped**.

### Impact

- Zero visibility into revenue or errors
- Cannot debug issues or generate invoices
- Vendor has no proof of payments processed

### Proposed Fix

**Spill-to-Disk Fallback**

```go
func (tw *TelemetryWorker) Dispatch(event *domain.TelemetryEvent) {
    select {
    case tw.events <- event:
        tw.eventsDispatched.Add(1)
    default:
        // Channel full - spill to disk
        if err := tw.spillToDisk(event); err != nil {
            tw.eventsDropped.Add(1)
            tw.logger.Error("Failed to spill event", zap.Error(err))
        } else {
            tw.eventsSpilled.Add(1)
        }
    }
}

func (tw *TelemetryWorker) spillToDisk(event *domain.TelemetryEvent) error {
    // Append to NDJSON file: /data/telemetry-spill-{date}.ndjson
    f, err := os.OpenFile(tw.spillPath, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer f.Close()
    return json.NewEncoder(f).Encode(event)
}
```

**Recovery Worker**

```go
// Periodically check for spill files and retry sending
func (tw *TelemetryWorker) recoverSpilledEvents() {
    // Read spill files, batch send to backend, delete on success
}
```

### Tasks

- [ ] Implement `spillToDisk()` function with NDJSON format
- [ ] Add recovery worker to retry spilled events
- [ ] Add metrics for spilled events count
- [ ] Add disk space monitoring/rotation for spill files
- [ ] Document data recovery procedures

---

## 5. Medium: SDK Nonce State Drift

**Component:** `machpay-py`  
**File:** `src/machpay/payment/negotiator.py`, `tests/e2e/sim_agent.py`  
**Status:** 🟡 Open

### Issue

The SDK manages nonces locally in some code paths:

```python
# sim_agent.py
self.nonce = int(time.time() * 1000)  # Local timestamp-based nonce
```

If the Agent application crashes/restarts, the nonce counter resets, causing:
- Old nonces being reused (Replay rejected)
- "403 Forbidden" errors for legitimate payments

### Proposed Fix

**Fetch Next Nonce from Gateway During 402 Handshake**

```python
class PaymentNegotiator:
    async def negotiate(self, request: Request) -> Response:
        # Step 1: Send request, get 402 challenge
        challenge = await self._get_challenge(request)
        
        # Step 2: Use nonce FROM challenge (Gateway provides it)
        intent = self.signer.sign_intent(
            nonce=challenge.nonce,  # Always use server-provided nonce
            # ...
        )
```

**Gateway Provides Next Expected Nonce**

```go
// Gateway 402 response includes next expected nonce
type Challenge struct {
    VendorID     string `json:"vendor_id"`
    Price        uint64 `json:"price"`
    NextNonce    uint64 `json:"next_nonce"`  // ADD THIS
    Deadline     int64  `json:"deadline"`
}
```

### Tasks

- [ ] Update Gateway to include `next_nonce` in 402 challenge
- [ ] Update SDK to always use server-provided nonce
- [ ] Remove local nonce management from SDK
- [ ] Add nonce synchronization on client startup

---

## 6. Medium: Internal Error Leakage

**Component:** `machpay-gateway`  
**File:** `internal/service/proxy/handler.go` (Line 335)  
**Status:** 🟡 Open

### Issue

Internal Go errors are wrapped and returned to clients:

```go
return 0, fmt.Errorf("internal error: %w", err)
```

This exposes infrastructure details (SQL errors, Redis timeouts) to external clients.

### Impact

- Information disclosure to attackers
- Debugging information leaks infrastructure topology

### Proposed Fix

```go
// Sanitize errors before returning to client
func sanitizeError(err error) error {
    // Log full error internally
    logger.Error("Internal error", zap.Error(err))
    
    // Return generic error to client
    return fiber.NewError(
        fiber.StatusInternalServerError,
        "internal_error",
    )
}
```

### Tasks

- [ ] Create `sanitizeError()` helper function
- [ ] Audit all error returns in handler.go
- [ ] Ensure full errors are logged internally
- [ ] Return generic errors to clients

---

## 7. Medium: Gas Inefficiency

**Component:** `machpay-relayer`  
**File:** `internal/core/domain/settlement.go`  
**Status:** 🟡 Open

### Issue

Each settlement is submitted as a separate transaction, paying full base fees.

### Current State (Mitigating Factor)

The contract already uses **cumulative amount netting**:

```rust
let delta = state.amount
    .checked_sub(agent_account.settled_cumulative_amount)
```

This means we settle the delta, not individual payments. However, we still pay per-transaction fees for each agent-vendor pair.

### Proposed Fix

**Batch Settlement Instruction**

```rust
pub fn settle_batch(
    ctx: Context<SettleBatch>,
    intents: Vec<PaymentIntent>,
) -> Result<()> {
    for intent in intents {
        // Verify and settle each intent
        // Share transaction overhead
    }
}
```

### Tasks

- [ ] Design `settle_batch` instruction (max 10-20 intents per TX)
- [ ] Update Relayer to batch settlements by deadline
- [ ] Benchmark gas savings
- [ ] Update SDK with batch settlement support

---

## 8. Medium: Hardcoded Capabilities Manifest

**Component:** `machpay-gateway`  
**File:** `internal/service/proxy/handler.go` (Line 222)  
**Status:** 🟡 Open

### Issue

The `/machpay.json` manifest returns hardcoded capabilities:

```go
Capabilities: []string{"reverse-proxy"},
```

Does not reflect:
- Actual gateway configuration
- Dynamic pricing status
- Health/availability

### Proposed Fix

```go
func (b *Bouncer) serveManifest(c *fiber.Ctx) error {
    capabilities := []string{"reverse-proxy"}
    
    if b.cfg.DynamicPricing {
        capabilities = append(capabilities, "dynamic-pricing")
    }
    if b.healthChecker.IsHealthy() {
        capabilities = append(capabilities, "healthy")
    }
    
    manifest := domain.ServiceManifest{
        Capabilities: capabilities,
        Status:       b.getStatus(),
        // ...
    }
}
```

### Tasks

- [ ] Make capabilities dynamic based on config
- [ ] Add health status to manifest
- [ ] Add version information
- [ ] Document all possible capabilities

---

## 9. Low: Configuration Drift

**Component:** `machpay-backend` (Receiver)  
**Status:** ⚪ Open

### Issue

The Gateway defaults to `https://telemetry.machpay.xyz` but local development requires pointing to the local backend.

### Proposed Fix

Update `docker-compose.yml` defaults:

```yaml
services:
  gateway:
    environment:
      MACHPAY_TELEMETRY_URL: "http://backend:8080"  # Local backend
```

### Tasks

- [ ] Update docker-compose defaults for local development
- [ ] Add environment-specific configuration examples
- [ ] Document telemetry URL configuration

---

## 10. Low: Naming Confusion

**Component:** Documentation  
**Status:** ⚪ Open

### Issue

Documentation references `machpay-recon` but code uses `machpay-relayer`.

### Proposed Fix

Standardize on `machpay-relayer` (current repo name) or rename to `machpay-settler` for clarity.

### Tasks

- [ ] Audit all documentation for naming inconsistencies
- [ ] Update docs to use consistent naming
- [ ] Consider rename to `machpay-settler` if appropriate

---

## Implementation Priority

| Priority | Issue | Effort | Owner | Target |
|----------|-------|--------|-------|--------|
| P0 | Strict Nonce Trap | High | TBD | Week 1-2 |
| P0 | SQLite Double-Spend | Low | TBD | Week 1 |
| P1 | Inline Sync DoS | Medium | TBD | Week 2 |
| P1 | Telemetry Loss | Medium | TBD | Week 2 |
| P2 | SDK Nonce Drift | Low | TBD | Week 3 |
| P2 | Error Leakage | Low | TBD | Week 3 |
| P2 | Gas Inefficiency | High | TBD | Week 4 |
| P3 | Manifest | Low | TBD | Backlog |
| P3 | Config Drift | Low | TBD | Backlog |
| P3 | Naming | Low | TBD | Backlog |

---

## Changelog

| Date | Author | Changes |
|------|--------|---------|
| 2024-12-29 | Security Audit | Initial document creation |

---

## References

- [MachPay Contract Source](../machpay-contract/programs/machpay-contracts/src/lib.rs)
- [Gateway Handler](../machpay-gateway/internal/service/proxy/handler.go)
- [Telemetry Worker](../machpay-gateway/internal/worker/telemetry.go)
- [Python SDK](../machpay-py/src/machpay/)


