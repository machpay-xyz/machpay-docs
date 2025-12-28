# MachPay Python SDK Integration Plan

## Multi-Phase Implementation Roadmap

---

## Overview

This document outlines the integration plan to connect the `machpay-py` SDK with the MachPay backend infrastructure. The goal is to enable AI agents to:

1. Make paid API calls via x402 protocol
2. Emit telemetry to the Receiver
3. View analytics in the Console
4. Track spending and solvency

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Integration Architecture                                 │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
  │   AI Agent   │         │   Gateway    │         │   Vendor     │
  │  (Python)    │◄───────►│   (Go)       │◄───────►│   Backend    │
  └──────┬───────┘   x402  └──────┬───────┘         └──────────────┘
         │                        │
         │ Telemetry              │ Telemetry
         ▼                        ▼
  ┌──────────────────────────────────────────────────┐
  │                 Receiver (Go)                     │
  │  ┌────────────────────────────────────────────┐  │
  │  │  POST /v1/ingest     (Gateway telemetry)   │  │
  │  │  POST /v1/agent      (Agent telemetry)     │  │ ◄── NEW
  │  └────────────────────────────────────────────┘  │
  └──────────────────────┬───────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────┐
  │                  PostgreSQL                       │
  │  ┌──────────────┐  ┌──────────────┐              │
  │  │gateway_      │  │agent_        │              │
  │  │telemetry     │  │telemetry     │ ◄── NEW     │
  │  └──────────────┘  └──────────────┘              │
  │  ┌──────────────┐  ┌──────────────┐              │
  │  │payment_      │  │agent_        │              │
  │  │intents       │  │interactions  │ ◄── NEW     │
  │  └──────────────┘  └──────────────┘              │
  └──────────────────────┬───────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────┐
  │               Backend API (Go)                    │
  │  ┌────────────────────────────────────────────┐  │
  │  │  GET /v1/agent/metrics    (Agent dashboard)│  │ ◄── NEW
  │  │  GET /v1/agent/history    (Spending log)   │  │ ◄── NEW
  │  │  GET /v1/agent/events     (SSE stream)     │  │ ◄── NEW
  │  └────────────────────────────────────────────┘  │
  └──────────────────────┬───────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────┐
  │               Console (Frontend)                  │
  │  ┌────────────────────────────────────────────┐  │
  │  │  Agent Dashboard  │  Spending Charts       │  │ ◄── NEW
  │  │  Vendor Breakdown │  Low Balance Alerts    │  │ ◄── NEW
  │  └────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────┘
```

---

## Phase 1: SDK Core Completion

**Objective:** Complete the Python SDK's core modules for payment and telemetry.

**Duration:** 3-4 days

### 1.1 Payment Module (`machpay-py`)

| File | Description | Status |
|------|-------------|--------|
| `src/machpay/payment/__init__.py` | Module exports | TODO |
| `src/machpay/payment/x402.py` | X402Negotiator class | TODO |
| `src/machpay/payment/challenge.py` | PaymentChallenge parser | TODO |
| `src/machpay/payment/intent.py` | PaymentIntent model | TODO |

**Key Implementation:**
- `X402Negotiator.authorize_request()` - Handle 402 → sign → retry loop
- `PaymentChallenge.from_response()` - Parse 402 response JSON
- Integration with existing `PaymentSigner` from auth module

### 1.2 Telemetry Module (`machpay-py`)

| File | Description | Status |
|------|-------------|--------|
| `src/machpay/telemetry/__init__.py` | Module exports | TODO |
| `src/machpay/telemetry/emitter.py` | TelemetryEmitter class | TODO |
| `src/machpay/telemetry/buffer.py` | RingBuffer for batching | TODO |
| `src/machpay/telemetry/models.py` | TelemetryPacket schemas | TODO |

**Key Implementation:**
- `TelemetryEmitter.start()` / `.stop()` - Background flush loop
- `TelemetryEmitter.record_interaction()` - Per-request metrics
- CloudEvents format for packets (matches receiver expectations)

### 1.3 Main Client (`machpay-py`)

| File | Description | Status |
|------|-------------|--------|
| `src/machpay/client.py` | MachPayClient class | TODO |
| `src/machpay/__init__.py` | Package exports | TODO |

**Key Implementation:**
- `MachPayClient.from_env()` - Factory from environment
- `MachPayClient.pay_and_call()` - Main entry point for paid requests
- `MachPayClient.get_balance()` - On-chain balance check
- Async context manager support (`async with`)

### 1.4 Tests

| File | Description |
|------|-------------|
| `tests/test_x402.py` | X402 negotiation tests |
| `tests/test_telemetry.py` | Telemetry emission tests |
| `tests/test_client.py` | Integration tests |

---

## Phase 2: Database Schema for Agent Telemetry

**Objective:** Create PostgreSQL tables to store agent metrics.

**Duration:** 1-2 days

### 2.1 New Tables (`machpay-backend`)

**File:** `internal/db/sql/schema.sql` (append)

```sql
-- =============================================================================
-- AGENT TELEMETRY TABLES
-- =============================================================================

-- Agent telemetry snapshots (periodic health reports from SDK)
CREATE TABLE IF NOT EXISTS agent_telemetry (
    time                    TIMESTAMPTZ NOT NULL,
    agent_id                VARCHAR(64) NOT NULL,
    
    -- Wallet state
    sol_balance_lamports    BIGINT,
    usdc_balance_atomic     BIGINT,
    low_balance_warnings    INT DEFAULT 0,
    
    -- Period metrics
    period_seconds          INT,
    total_requests          INT DEFAULT 0,
    total_payments          INT DEFAULT 0,
    total_spend_atomic      BIGINT DEFAULT 0,
    
    -- Per-vendor breakdown (JSONB for flexibility)
    interactions            JSONB,
    
    -- SDK metadata
    sdk_version             VARCHAR(32),
    
    PRIMARY KEY (time, agent_id)
) PARTITION BY RANGE (time);

-- Agent interaction log (individual payment events)
CREATE TABLE IF NOT EXISTS agent_interactions (
    time                TIMESTAMPTZ NOT NULL,
    agent_id            VARCHAR(64) NOT NULL,
    vendor_id           VARCHAR(64) NOT NULL,
    
    -- Request details
    request_url         VARCHAR(512),
    request_method      VARCHAR(16),
    
    -- Payment details
    amount_atomic       BIGINT,
    token_mint          VARCHAR(64),
    nonce               BIGINT,
    
    -- Outcome
    success             BOOLEAN,
    latency_ms          INT,
    error_type          VARCHAR(64),
    error_message       TEXT,
    
    -- Signature for audit
    signature           VARCHAR(128),
    
    PRIMARY KEY (time, agent_id, nonce)
) PARTITION BY RANGE (time);

-- Hourly agent stats rollup (for dashboard charts)
CREATE TABLE IF NOT EXISTS agent_stats_hourly (
    bucket              TIMESTAMPTZ NOT NULL,
    agent_id            VARCHAR(64) NOT NULL,
    
    -- Aggregates
    total_requests      BIGINT DEFAULT 0,
    successful_requests BIGINT DEFAULT 0,
    failed_requests     BIGINT DEFAULT 0,
    total_spend_atomic  BIGINT DEFAULT 0,
    avg_latency_ms      FLOAT,
    p99_latency_ms      FLOAT,
    
    -- Vendor breakdown
    vendor_breakdown    JSONB,
    
    PRIMARY KEY (bucket, agent_id)
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_agent_telemetry_agent 
    ON agent_telemetry (agent_id, time DESC);

CREATE INDEX IF NOT EXISTS idx_agent_interactions_agent 
    ON agent_interactions (agent_id, time DESC);

CREATE INDEX IF NOT EXISTS idx_agent_interactions_vendor 
    ON agent_interactions (vendor_id, time DESC);

CREATE INDEX IF NOT EXISTS idx_agent_stats_hourly_agent 
    ON agent_stats_hourly (agent_id, bucket DESC);
```

### 2.2 Initial Partitions

```sql
-- Create default and current month partitions
CREATE TABLE IF NOT EXISTS agent_telemetry_default 
    PARTITION OF agent_telemetry DEFAULT;

CREATE TABLE IF NOT EXISTS agent_telemetry_y2025m01 
    PARTITION OF agent_telemetry 
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE IF NOT EXISTS agent_interactions_default 
    PARTITION OF agent_interactions DEFAULT;

CREATE TABLE IF NOT EXISTS agent_interactions_y2025m01 
    PARTITION OF agent_interactions 
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

### 2.3 Partition Manager Update

**File:** `internal/tasks/partition_manager.go`

- Add `agent_telemetry` and `agent_interactions` to partition rotation
- Configure retention: 30 days for telemetry, 90 days for interactions

---

## Phase 3: Receiver - Agent Telemetry Endpoint

**Objective:** Add a dedicated endpoint for agent telemetry ingestion.

**Duration:** 2-3 days

### 3.1 New Endpoint (`machpay-backend`)

**Route:** `POST /v1/agent`

**File:** `cmd/receiver/main.go`

```go
// Add new route
v1.POST("/agent", agentHandler.HandleIngest)
```

### 3.2 Agent Handler

**File:** `internal/receiver/agent_handler.go` (NEW)

```go
type AgentHandler struct {
    worker *AgentTelemetryWorker
    logger *zap.Logger
}

func (h *AgentHandler) HandleIngest(ctx context.Context, c *app.RequestContext) {
    // 1. Validate X-MachPay-Agent header
    // 2. Parse CloudEvents envelope
    // 3. Dispatch to worker
}
```

### 3.3 Agent Telemetry Worker

**File:** `internal/receiver/agent_worker.go` (NEW)

```go
type AgentTelemetryWorker struct {
    dbPool          *pgxpool.Pool
    ingestChan      chan *AgentTelemetryPacket
    interactionChan chan *AgentInteraction
}

func (w *AgentTelemetryWorker) flushTelemetry(batch []*AgentTelemetryPacket) error {
    // COPY to agent_telemetry
}

func (w *AgentTelemetryWorker) flushInteractions(batch []*AgentInteraction) error {
    // COPY to agent_interactions
}
```

### 3.4 Parser Models

**File:** `internal/receiver/agent_models.go` (NEW)

```go
type AgentTelemetryPacket struct {
    AgentID             string                 `json:"agent_id"`
    PeriodStart         int64                  `json:"period_start"`
    PeriodEnd           int64                  `json:"period_end"`
    Wallet              WalletSnapshot         `json:"wallet"`
    Interactions        map[string]VendorStats `json:"interactions"`
    SDKVersion          string                 `json:"sdk_version"`
}

type WalletSnapshot struct {
    SOLBalanceLamports  int64 `json:"sol_balance_lamports"`
    USDCBalanceAtomic   int64 `json:"usdc_balance_atomic"`
    LowBalanceWarnings  int   `json:"low_balance_warnings"`
}

type VendorStats struct {
    Attempts          int            `json:"attempts"`
    Successes         int            `json:"success"`
    Failures          int            `json:"failures"`
    PaymentLatencyP50 int            `json:"payment_latency_p50"`
    PaymentLatencyP99 int            `json:"payment_latency_p99"`
    SpendAtomic       int64          `json:"spend_atomic"`
    Errors            map[string]int `json:"errors"`
}
```

### 3.5 Auth Middleware

**File:** `internal/receiver/middleware.go`

- Add `AgentAuthMiddleware` - validates agent pubkey exists in `agents` table
- Or allow permissive ingestion (agent auto-registers on first telemetry)

---

## Phase 4: Backend API - Agent Dashboard Endpoints

**Objective:** Create REST APIs for the Console to display agent metrics.

**Duration:** 2-3 days

### 4.1 Repository Layer

**File:** `internal/repository/agent_repo.go` (NEW)

```go
type AgentRepository struct {
    pool *pgxpool.Pool
}

// Get agent's aggregated metrics for dashboard
func (r *AgentRepository) GetAgentMetrics(
    ctx context.Context, 
    agentID string, 
    timeRange string,
) (*AgentMetrics, error)

// Get agent's spending history
func (r *AgentRepository) GetAgentSpendingHistory(
    ctx context.Context, 
    agentID string, 
    page, limit int,
) ([]*AgentInteraction, int, error)

// Get agent's vendor breakdown
func (r *AgentRepository) GetVendorBreakdown(
    ctx context.Context, 
    agentID string, 
    timeRange string,
) ([]*VendorSpend, error)

// Get agent's balance history
func (r *AgentRepository) GetBalanceHistory(
    ctx context.Context, 
    agentID string, 
    timeRange string,
) ([]*BalancePoint, error)
```

### 4.2 Handler Layer

**File:** `internal/handlers/sql/agent_handler.go` (NEW)

```go
type AgentHandler struct {
    repo *repository.AgentRepository
}

// GET /v1/agent/metrics
func (h *AgentHandler) HandleGetMetrics(ctx context.Context, c *app.RequestContext)

// GET /v1/agent/history
func (h *AgentHandler) HandleGetHistory(ctx context.Context, c *app.RequestContext)

// GET /v1/agent/vendors
func (h *AgentHandler) HandleGetVendors(ctx context.Context, c *app.RequestContext)

// GET /v1/agent/balance
func (h *AgentHandler) HandleGetBalance(ctx context.Context, c *app.RequestContext)
```

### 4.3 API Routes

**File:** `cmd/api/main.go`

```go
// Agent dashboard routes (protected by JWT)
agentGroup := v1.Group("/agent", jwtMiddleware)
{
    agentGroup.GET("/metrics", agentHandler.HandleGetMetrics)
    agentGroup.GET("/history", agentHandler.HandleGetHistory)
    agentGroup.GET("/vendors", agentHandler.HandleGetVendors)
    agentGroup.GET("/balance", agentHandler.HandleGetBalance)
}
```

### 4.4 Response Models

```go
type AgentMetricsResponse struct {
    AgentID           string           `json:"agent_id"`
    TimeRange         string           `json:"time_range"`
    TotalRequests     int64            `json:"total_requests"`
    SuccessRate       float64          `json:"success_rate"`
    TotalSpendUSD     float64          `json:"total_spend_usd"`
    AvgLatencyMs      float64          `json:"avg_latency_ms"`
    CurrentBalanceUSD float64          `json:"current_balance_usd"`
    RunwayHours       float64          `json:"runway_hours"`
    ChartData         []MetricPoint    `json:"chart_data"`
}

type SpendingHistoryResponse struct {
    Data     []*AgentInteraction `json:"data"`
    Page     int                 `json:"page"`
    Total    int                 `json:"total"`
    HasMore  bool                `json:"has_more"`
}

type VendorBreakdownResponse struct {
    Vendors []VendorSpend `json:"vendors"`
}

type VendorSpend struct {
    VendorID       string  `json:"vendor_id"`
    RequestCount   int64   `json:"request_count"`
    SpendUSD       float64 `json:"spend_usd"`
    AvgLatencyMs   float64 `json:"avg_latency_ms"`
    SuccessRate    float64 `json:"success_rate"`
}
```

---

## Phase 5: Real-Time Agent Events (SSE)

**Objective:** Stream agent events to the Console in real-time.

**Duration:** 1-2 days

### 5.1 NOTIFY from Receiver

**File:** `internal/receiver/agent_worker.go`

```go
// After flushing interactions, emit NOTIFY
func (w *AgentTelemetryWorker) notifyInteractions(tx pgx.Tx, interactions []*AgentInteraction) error {
    for _, i := range interactions {
        payload := fmt.Sprintf(`{"topic":"agent:%s","type":"interaction","data":%s}`, 
            i.AgentID, i.ToJSON())
        _, err := tx.Exec(ctx, "NOTIFY machpay_events, $1", payload)
        if err != nil {
            return err
        }
    }
    return nil
}
```

### 5.2 SSE Endpoint

**File:** `internal/handlers/sql/stream_handler.go`

```go
// Extend HandleStreamEvents to support agent topics
// Topic format: "agent:{agent_id}"

func (h *StreamHandler) HandleStreamEvents(ctx context.Context, c *app.RequestContext) {
    topic := c.Query("topic")
    
    // Security: Verify user owns this agent
    if strings.HasPrefix(topic, "agent:") {
        agentID := strings.TrimPrefix(topic, "agent:")
        if !h.verifyAgentOwnership(userID, agentID) {
            c.JSON(403, gin.H{"error": "forbidden"})
            return
        }
    }
    
    // ... rest of SSE logic
}
```

### 5.3 Agent SSE Topics

| Topic Pattern | Description | Events |
|---------------|-------------|--------|
| `agent:{id}` | Single agent's events | interaction, balance_update, low_balance |
| `agent:*` | All owned agents (admin) | All agent events |

---

## Phase 6: Agent Rollup Worker

**Objective:** Aggregate agent metrics for faster dashboard queries.

**Duration:** 1-2 days

### 6.1 Update Rollup Worker

**File:** `internal/tasks/rollup_worker.go`

```go
// Add agent stats rollup
func (w *RollupWorker) rollupAgentStats(ctx context.Context, hour time.Time) error {
    query := `
        INSERT INTO agent_stats_hourly (bucket, agent_id, total_requests, ...)
        SELECT 
            date_trunc('hour', time) as bucket,
            agent_id,
            COUNT(*) as total_requests,
            SUM(CASE WHEN success THEN 1 ELSE 0 END) as successful_requests,
            SUM(CASE WHEN NOT success THEN 1 ELSE 0 END) as failed_requests,
            SUM(amount_atomic) as total_spend_atomic,
            AVG(latency_ms) as avg_latency_ms,
            PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) as p99_latency_ms,
            jsonb_object_agg(vendor_id, ...) as vendor_breakdown
        FROM agent_interactions
        WHERE time >= $1 AND time < $2
        GROUP BY date_trunc('hour', time), agent_id
        ON CONFLICT (bucket, agent_id) DO UPDATE SET ...
    `
    _, err := w.pool.Exec(ctx, query, hour, hour.Add(time.Hour))
    return err
}
```

### 6.2 Scheduler Update

**File:** `cmd/worker/main.go`

```go
// Add agent rollup to worker
go rollupWorker.StartAgentRollup(ctx)
```

---

## Phase 7: Agent Registration & Linking

**Objective:** Allow agents to register and link to user accounts.

**Duration:** 2-3 days

### 7.1 Agent Registration Table

**File:** `internal/db/sql/schema.sql`

```sql
-- Agent registration (linked to user accounts)
CREATE TABLE IF NOT EXISTS agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pubkey          VARCHAR(64) UNIQUE NOT NULL,
    user_id         UUID REFERENCES users(id),
    name            VARCHAR(128),
    
    -- Status
    linked_at       TIMESTAMPTZ,
    last_seen_at    TIMESTAMPTZ,
    status          VARCHAR(32) DEFAULT 'pending',
    
    -- Limits
    daily_spend_limit_atomic BIGINT DEFAULT 100000000, -- 100 USDC
    
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_agents_user ON agents (user_id);
CREATE INDEX idx_agents_pubkey ON agents (pubkey);
```

### 7.2 Link Code Flow

**SDK Side (`machpay-py`):**
```python
async def link(self) -> str:
    # 1. Generate link code from Console API
    response = await self._http.post(
        f"{self.config.console_url}/api/link/init",
        json={"pubkey": str(self.pubkey)},
    )
    code = response.json()["code"]
    
    # 2. Return URL for user to open
    return f"{self.config.console_url}/link?code={code}"
```

**Backend Side (`machpay-backend`):**
```go
// POST /v1/link/init - Create link code
func (h *LinkHandler) HandleInitLink(ctx context.Context, c *app.RequestContext)

// POST /v1/link/confirm - Confirm link (called by Console after user approval)
func (h *LinkHandler) HandleConfirmLink(ctx context.Context, c *app.RequestContext)
```

---

## Phase 8: Low Balance Alerts

**Objective:** Notify users when agent balance drops below threshold.

**Duration:** 1-2 days

### 8.1 Alert Table

```sql
CREATE TABLE IF NOT EXISTS agent_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID REFERENCES agents(id),
    alert_type      VARCHAR(32) NOT NULL,
    severity        VARCHAR(16) NOT NULL, -- info, warning, critical
    message         TEXT,
    data            JSONB,
    acknowledged    BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 8.2 Alert Detection

**File:** `internal/receiver/agent_worker.go`

```go
func (w *AgentTelemetryWorker) checkAlerts(packet *AgentTelemetryPacket) {
    // Check low balance
    balanceUSD := float64(packet.Wallet.USDCBalanceAtomic) / 1e6
    if balanceUSD < w.config.LowBalanceThreshold {
        w.emitAlert(packet.AgentID, "low_balance", "warning", ...)
    }
    
    // Check high error rate
    // Check unusual spending
}
```

### 8.3 Alert Notification

- Emit via SSE to connected Console sessions
- Optional: Webhook to user-configured URL
- Optional: Email notification

---

## Phase 9: SDK CLI & Discovery

**Objective:** Complete CLI commands and vendor discovery.

**Duration:** 2-3 days

### 9.1 CLI Commands (`machpay-py`)

**File:** `src/machpay/cli/main.py`

| Command | Description |
|---------|-------------|
| `machpay init` | Generate keypair |
| `machpay link` | Link to Console |
| `machpay whoami` | Show pubkey |
| `machpay balance` | Check balance |
| `machpay search <query>` | Search vendors |
| `machpay test <url>` | Test paid request |
| `machpay stats` | Show spending stats |

### 9.2 Vendor Discovery

**File:** `src/machpay/discovery/search.py`

```python
class VendorSearch:
    async def search(self, query: str, ...) -> list[VendorConfig]:
        # Query MachPay API for registered vendors
        response = await self._http.get(
            f"{self.config.api_url}/v1/vendors/search",
            params={"q": query, ...},
        )
        return [VendorConfig(**v) for v in response.json()["vendors"]]
```

### 9.3 Backend - Vendor Search Endpoint

**File:** `internal/handlers/sql/vendor_handler.go`

```go
// GET /v1/vendors/search
func (h *VendorHandler) HandleSearch(ctx context.Context, c *app.RequestContext) {
    // Query vendors table with filters
    // Include uptime, latency, pricing from telemetry aggregates
}
```

---

## Phase 10: E2E Integration Testing

**Objective:** Validate the complete flow from SDK to Console.

**Duration:** 2-3 days

### 10.1 Test Scenarios

| Test | Description |
|------|-------------|
| `test_agent_registration` | Agent links to user account |
| `test_paid_request` | SDK makes paid request via Gateway |
| `test_telemetry_flow` | SDK → Receiver → DB → API |
| `test_sse_streaming` | Agent events stream to Console |
| `test_low_balance_alert` | Alert triggers on low balance |
| `test_spending_history` | History appears in Console |

### 10.2 Docker Compose Update

**File:** `docker-compose.e2e.yml`

```yaml
services:
  # Existing services...
  
  agent-simulator:
    build:
      context: ../machpay-py
      dockerfile: Dockerfile
    environment:
      MACHPAY_AGENT_SECRET: "[test-keypair-json]"
      MACHPAY_GATEWAY_URL: "http://gateway:8080"
      MACHPAY_RECEIVER_URL: "http://receiver:8080"
    command: ["python", "-m", "machpay.test.e2e_agent"]
    depends_on:
      - gateway
      - receiver
```

### 10.3 Test Runner

**File:** `tests/e2e_agent_test.go`

```go
func TestAgentTelemetryFlow(t *testing.T) {
    // 1. Start agent simulator
    // 2. Wait for telemetry to arrive
    // 3. Query API for agent metrics
    // 4. Assert data integrity
}
```

---

## Phase 11: Documentation & Examples

**Objective:** Complete SDK documentation and examples.

**Duration:** 2-3 days

### 11.1 SDK Documentation

| File | Description |
|------|-------------|
| `machpay-py/README.md` | Installation, quickstart |
| `machpay-py/docs/getting-started.md` | Step-by-step guide |
| `machpay-py/docs/api-reference.md` | Full API docs |
| `machpay-py/docs/examples.md` | Code examples |

### 11.2 Example Projects

| Example | Description |
|---------|-------------|
| `examples/basic_request.py` | Simple paid request |
| `examples/openai_agent.py` | OpenAI API integration |
| `examples/multi_vendor.py` | Multiple vendor usage |
| `examples/mcp_tool.py` | MCP tool integration |

### 11.3 Backend API Documentation

- Update OpenAPI spec with agent endpoints
- Add to `machpay-docs/api/`

---

## Implementation Timeline

```
Week 1:
├── Phase 1: SDK Core Completion (3-4 days)
└── Phase 2: Database Schema (1-2 days)

Week 2:
├── Phase 3: Receiver Agent Endpoint (2-3 days)
└── Phase 4: Backend API Endpoints (2-3 days)

Week 3:
├── Phase 5: Real-Time SSE (1-2 days)
├── Phase 6: Agent Rollup Worker (1-2 days)
└── Phase 7: Agent Registration (2-3 days)

Week 4:
├── Phase 8: Low Balance Alerts (1-2 days)
├── Phase 9: CLI & Discovery (2-3 days)
└── Phase 10: E2E Testing (2-3 days)

Week 5:
└── Phase 11: Documentation (2-3 days)
```

---

## Dependencies Between Phases

```
Phase 1 (SDK Core)
    │
    ├──► Phase 2 (Database) ──► Phase 3 (Receiver)
    │                               │
    │                               ├──► Phase 4 (API)
    │                               │       │
    │                               │       └──► Phase 5 (SSE)
    │                               │
    │                               └──► Phase 6 (Rollup)
    │
    ├──► Phase 7 (Registration)
    │       │
    │       └──► Phase 8 (Alerts)
    │
    └──► Phase 9 (CLI/Discovery)

All phases ──► Phase 10 (E2E Testing) ──► Phase 11 (Docs)
```

---

## Success Criteria

### Functional
- [ ] SDK can make paid requests to Gateway
- [ ] Telemetry arrives in PostgreSQL within 60s
- [ ] Console displays agent metrics
- [ ] SSE streams agent events in real-time
- [ ] Low balance alerts trigger correctly

### Performance
- [ ] Telemetry ingestion: 1000 packets/sec
- [ ] API response time: < 100ms p99
- [ ] SSE latency: < 500ms from event to client

### Quality
- [ ] Unit test coverage: > 80%
- [ ] E2E tests pass in CI
- [ ] Documentation complete

---

*Document Version: 1.0*
*Created: December 2024*
*Author: MachPay Engineering*

