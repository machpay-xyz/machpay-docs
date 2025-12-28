# MachPay Receiver Implementation Plan
## Gap Analysis & Phased Implementation Roadmap

---

## 1. System Overview

The MachPay telemetry pipeline consists of:

```
Gateway ──Events──> Receiver ──Persist──> PostgreSQL ──Query──> Backend APIs ──SSE──> Console
```

### Components Under Scope

| Component | Repository | Role |
|-----------|------------|------|
| Gateway | `machpay-gateway` | Emits CloudEvents + TelemetryPackets |
| Receiver | `machpay-backend/cmd/receiver` | High-throughput ingestion service |
| Database | PostgreSQL | Persistence layer |
| Backend | `machpay-backend/cmd/api` | Console-facing REST APIs |
| SSE Broker | `machpay-backend/internal/realtime` | Real-time event streaming |

---

## 2. Gap Analysis

### 2.1 Gateway (`machpay-gateway`)

| Feature | Status | Gap Description |
|---------|--------|-----------------|
| TelemetryWorker | ✅ Complete | CloudEvents dispatch working |
| MetricsStreamer | ✅ Complete | 60s telemetry packets implemented |
| BusinessMetricsCollector | ✅ Complete | Per-route stats collection working |
| PaymentIntent emission | ⚠️ Partial | Sent to settlement queue but NOT to receiver |
| Relayer URL config | ⚠️ Partial | Defaults to empty, needs proper configuration |
| GatewayTelemetryPacket schema | ⚠️ Partial | Missing `config` block emission |

**Gaps to Fix:**
1. Payment intents should be sent to receiver in addition to settlement queue
2. `relayer_url` should default to receiver endpoint in init config
3. Config block should be included in telemetry packet when changed

---

### 2.2 Receiver (`machpay-backend/cmd/receiver`)

| Feature | Status | Gap Description |
|---------|--------|-----------------|
| HTTP ingestion endpoint | ✅ Complete | `POST /v1/ingest` working |
| Batch processor | ✅ Complete | 1s/1000 event flush logic |
| CloudEvents parsing | ⚠️ Partial | Only stores raw JSONB, no type routing |
| GatewayTelemetryPacket parsing | ❌ Missing | Not implemented |
| PaymentIntent parsing | ❌ Missing | Not implemented |
| Multi-table persistence | ❌ Missing | All events go to single `telemetry_events` table |
| SSE broadcast integration | ❌ Missing | No connection to realtime broker |
| Metrics (Prometheus) | ❌ Missing | No observability |
| Health check details | ⚠️ Partial | Basic stats only |

**Gaps to Fix:**
1. Route events by `type` field to appropriate tables
2. Parse `GatewayTelemetryPacket` into structured `gateway_telemetry` table
3. Parse `PaymentAuthorized` into structured `payment_authorized` table
4. Integrate with SSE broker for real-time broadcasts
5. Add Prometheus metrics for monitoring
6. Parse `RequestBlocked` and `SolvencyAlert` events

---

### 2.3 Database Schema

| Table | Status | Gap Description |
|-------|--------|-----------------|
| `telemetry_events` | ✅ Exists | Raw event storage (partitioned) |
| `payment_authorized` | ✅ Exists | Structured payment events |
| `request_blocked` | ✅ Exists | Security events |
| `solvency_alerts` | ✅ Exists | Risk events |
| `gateway_telemetry` | ❌ Missing | 60s deep telemetry packets |
| `gateway_stats_hourly` | ❌ Missing | Hourly rollup aggregates |
| `payment_intents` | ❌ Missing | Settlement-ready intents with status |
| Partition management | ❌ Missing | No automated partition creation/cleanup |
| Indexes for analytics | ⚠️ Partial | Missing composite indexes for dashboard queries |

**Gaps to Fix:**
1. Create `gateway_telemetry` table for 60s packets
2. Create `gateway_stats_hourly` for pre-computed aggregates
3. Create `payment_intents` table with status tracking
4. Add partition management functions/jobs
5. Add optimized indexes for common query patterns

---

### 2.4 Backend APIs (`machpay-backend/cmd/api`)

| Endpoint | Status | Gap Description |
|----------|--------|-----------------|
| `/v1/vendor/stats` | ✅ Exists | Basic vendor statistics |
| `/v1/vendor/revenue-history` | ✅ Exists | 30-day revenue data |
| `/v1/vendor/consumers` | ✅ Exists | Active agents list |
| `/v1/vendor/endpoints` | ✅ Exists | Endpoint CRUD |
| `/v1/vendor/metrics` | ❌ Missing | Real-time telemetry metrics |
| `/v1/vendor/payments` | ❌ Missing | Payment history with filters |
| `/v1/vendor/alerts` | ❌ Missing | Security alerts dashboard |
| `/v1/agent/activity` | ❌ Missing | Agent activity timeline |
| `/v1/agent/spending` | ❌ Missing | Spending breakdown by vendor |
| `/v1/telemetry/live` | ❌ Missing | SSE stream endpoint |
| `/v1/admin/gateways` | ❌ Missing | Gateway health monitoring |

**Gaps to Fix:**
1. Create TelemetryHandler for metrics endpoints
2. Implement payment history with pagination/filtering
3. Implement security alerts endpoint
4. Implement agent analytics endpoints
5. Create SSE streaming endpoint
6. Add admin endpoints for gateway monitoring

---

### 2.5 SSE Real-time Broker

| Feature | Status | Gap Description |
|---------|--------|-----------------|
| Broker core | ✅ Complete | Client management, broadcasting |
| Topic routing | ✅ Complete | Public + user-specific topics |
| Stream handler | ✅ Complete | In `stream_handler.go` |
| Receiver integration | ❌ Missing | Receiver doesn't publish to broker |
| Gateway-specific topics | ❌ Missing | `vendor:{gateway_id}` topics |
| Agent-specific topics | ❌ Missing | `agent:{agent_id}` topics |

**Gaps to Fix:**
1. Connect receiver to broker for real-time publishing
2. Add gateway/vendor specific topic routing
3. Add agent specific topic routing

---

### 2.6 E2E Integration

| Test | Status | Gap Description |
|------|--------|-----------------|
| Unit tests | ⚠️ Partial | Some handlers tested |
| Integration tests | ⚠️ Partial | Basic auth flow tested |
| Gateway → Receiver flow | ❌ Missing | Not tested |
| Receiver → DB persistence | ❌ Missing | Not tested |
| DB → Backend API queries | ❌ Missing | Not tested |
| SSE real-time flow | ❌ Missing | Not tested |
| Stress testing | ❌ Missing | No throughput benchmarks |
| Docker Compose E2E | ❌ Missing | No multi-service test setup |

**Gaps to Fix:**
1. Create E2E test harness with all services
2. Test complete data flow
3. Add stress testing for throughput validation
4. Create Docker Compose for E2E environment

---

## 3. Phased Implementation Plan

### Phase 1: Database Foundation
**Duration**: 2-3 days  
**Priority**: 🔴 Critical  
**Dependencies**: None

#### Objectives
- Create missing tables for telemetry storage
- Implement proper partitioning strategy
- Add optimized indexes for analytics queries

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 1.1 | Create `gateway_telemetry` table with all LLD fields | `scripts/sql/02_receiver_tables.sql` |
| 1.2 | Create `gateway_stats_hourly` rollup table | `scripts/sql/02_receiver_tables.sql` |
| 1.3 | Create `payment_intents` table with status tracking | `scripts/sql/02_receiver_tables.sql` |
| 1.4 | Add composite indexes for dashboard queries | `scripts/sql/02_receiver_tables.sql` |
| 1.5 | Create partition management helper functions | `scripts/sql/03_partition_management.sql` |
| 1.6 | Update `schema.sql` to include all tables | `internal/db/sql/schema.sql` |

#### Deliverables
- [ ] Migration script tested on local PostgreSQL
- [ ] All tables created with proper constraints
- [ ] Indexes verified with `EXPLAIN ANALYZE`
- [ ] Partition creation verified

---

### Phase 2: Receiver Event Routing
**Duration**: 3-4 days  
**Priority**: 🔴 Critical  
**Dependencies**: Phase 1

#### Objectives
- Implement event type routing in receiver
- Add structured parsing for each event type
- Multi-table persistence

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 2.1 | Define Go structs for all event types | `internal/receiver/models.go` |
| 2.2 | Create event router by `type` field | `internal/receiver/router.go` (new) |
| 2.3 | Implement `PaymentAuthorized` parser | `internal/receiver/parsers.go` (new) |
| 2.4 | Implement `RequestBlocked` parser | `internal/receiver/parsers.go` |
| 2.5 | Implement `GatewayTelemetryPacket` parser | `internal/receiver/parsers.go` |
| 2.6 | Implement `SolvencyAlert` parser | `internal/receiver/parsers.go` |
| 2.7 | Update batch flush to multi-table insert | `internal/receiver/worker.go` |
| 2.8 | Add table-specific `COPY FROM` batches | `internal/receiver/worker.go` |

#### Deliverables
- [ ] All event types routed correctly
- [ ] Structured data persisted to appropriate tables
- [ ] Raw events still stored as fallback
- [ ] Batch flush handles multiple tables

---

### Phase 3: Receiver SSE Integration
**Duration**: 2 days  
**Priority**: 🟠 High  
**Dependencies**: Phase 2

#### Objectives
- Connect receiver to SSE broker
- Broadcast events in real-time
- Add topic routing for vendors/agents

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 3.1 | Create streamer component | `internal/receiver/streamer.go` (new) |
| 3.2 | Initialize broker connection in receiver | `cmd/receiver/main.go` |
| 3.3 | Broadcast `PaymentAuthorized` to public ticker | `internal/receiver/streamer.go` |
| 3.4 | Broadcast to vendor-specific topic | `internal/receiver/streamer.go` |
| 3.5 | Broadcast to agent-specific topic | `internal/receiver/streamer.go` |
| 3.6 | Add new topic constants | `internal/realtime/broker.go` |

#### Deliverables
- [ ] Real-time events broadcast on ingestion
- [ ] Vendor dashboard receives live updates
- [ ] Agent dashboard receives live updates

---

### Phase 4: Receiver Observability
**Duration**: 1-2 days  
**Priority**: 🟡 Medium  
**Dependencies**: Phase 2

#### Objectives
- Add Prometheus metrics to receiver
- Enhanced health endpoint
- Logging improvements

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 4.1 | Add Prometheus metrics | `internal/receiver/metrics.go` (new) |
| 4.2 | Expose `/metrics` endpoint | `cmd/receiver/main.go` |
| 4.3 | Add metrics: events_ingested, batch_duration, buffer_depth | `internal/receiver/metrics.go` |
| 4.4 | Enhanced `/health` with component status | `cmd/receiver/main.go` |
| 4.5 | Structured logging with zap | `internal/receiver/worker.go` |

#### Deliverables
- [ ] Prometheus metrics exposed
- [ ] Health endpoint shows detailed status
- [ ] Structured JSON logging

---

### Phase 5: Backend Telemetry APIs
**Duration**: 3-4 days  
**Priority**: 🟠 High  
**Dependencies**: Phase 1

#### Objectives
- Create new API endpoints for telemetry data
- Implement pagination and filtering
- Add repository layer for new tables

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 5.1 | Create TelemetryRepository | `internal/repository/telemetry_repo.go` (new) |
| 5.2 | Create TelemetryHandler | `internal/handlers/sql/telemetry_handler.go` (new) |
| 5.3 | Implement `GET /v1/vendor/metrics` | `telemetry_handler.go` |
| 5.4 | Implement `GET /v1/vendor/payments` | `telemetry_handler.go` |
| 5.5 | Implement `GET /v1/vendor/alerts` | `telemetry_handler.go` |
| 5.6 | Implement `GET /v1/agent/activity` | `telemetry_handler.go` |
| 5.7 | Implement `GET /v1/agent/spending` | `telemetry_handler.go` |
| 5.8 | Add routes to main API router | `cmd/api/main.go` |
| 5.9 | Update Swagger documentation | `docs/` |

#### Deliverables
- [ ] All new endpoints functional
- [ ] Pagination working correctly
- [ ] Filters by date, agent, gateway
- [ ] Swagger docs updated

---

### Phase 6: SSE Streaming Endpoint
**Duration**: 1-2 days  
**Priority**: 🟠 High  
**Dependencies**: Phase 3, Phase 5

#### Objectives
- Create SSE endpoint for Console
- Topic subscription management
- Authentication integration

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 6.1 | Implement `GET /v1/telemetry/live` SSE endpoint | `internal/handlers/sql/stream_handler.go` |
| 6.2 | Add topic parameter parsing | `stream_handler.go` |
| 6.3 | Integrate with JWT auth for user-specific topics | `stream_handler.go` |
| 6.4 | Add heartbeat for connection keepalive | `stream_handler.go` |
| 6.5 | Add connection limits per user | `stream_handler.go` |

#### Deliverables
- [ ] SSE endpoint working
- [ ] Authenticated topic subscription
- [ ] Connection stable for 24h+

---

### Phase 7: Gateway Enhancements
**Duration**: 1-2 days  
**Priority**: 🟡 Medium  
**Dependencies**: Phase 2

#### Objectives
- Ensure gateway events reach receiver
- Add missing event emissions
- Configuration improvements

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 7.1 | Emit PaymentIntent as CloudEvent (in addition to settlement) | `internal/service/proxy/handler.go` |
| 7.2 | Include config block in telemetry packet on startup | `internal/worker/metrics_streamer.go` |
| 7.3 | Update default `relayer_url` in init config | `internal/adapter/primary/cli/init.go` |
| 7.4 | Add receiver health check on gateway startup | `internal/adapter/primary/cli/serve.go` |
| 7.5 | Log when events dropped due to receiver unavailable | `internal/worker/telemetry.go` |

#### Deliverables
- [ ] All events reaching receiver
- [ ] Config changes broadcasted
- [ ] Proper defaults in generated config

---

### Phase 8: E2E Integration Testing
**Duration**: 3-4 days  
**Priority**: 🔴 Critical  
**Dependencies**: All previous phases

#### Objectives
- Create multi-service test environment
- Test complete data flow
- Validate throughput requirements

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 8.1 | Create E2E Docker Compose | `machpay-backend/docker-compose.e2e.yml` (new) |
| 8.2 | Create test harness with mock gateway | `tests/e2e_receiver_test.go` (new) |
| 8.3 | Test: Event ingestion flow | `e2e_receiver_test.go` |
| 8.4 | Test: SSE broadcast flow | `e2e_receiver_test.go` |
| 8.5 | Test: Metrics aggregation | `e2e_receiver_test.go` |
| 8.6 | Test: High-throughput (10k events/sec) | `e2e_receiver_test.go` |
| 8.7 | Add Makefile targets for E2E | `Makefile` |
| 8.8 | CI/CD integration (GitHub Actions) | `.github/workflows/e2e.yml` (new) |

#### Deliverables
- [ ] All E2E tests passing
- [ ] 10,000 events/sec throughput verified
- [ ] No data loss under load
- [ ] CI pipeline running tests

---

### Phase 9: Hourly Rollup Worker
**Duration**: 2 days  
**Priority**: 🟡 Medium  
**Dependencies**: Phase 1, Phase 2

#### Objectives
- Implement background worker for aggregations
- Pre-compute dashboard metrics
- Optimize query performance

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 9.1 | Create rollup worker | `internal/tasks/rollup_worker.go` (new) |
| 9.2 | Aggregate `gateway_telemetry` → `gateway_stats_hourly` | `rollup_worker.go` |
| 9.3 | Schedule hourly execution | `cmd/worker/main.go` |
| 9.4 | Add rollup metrics | `rollup_worker.go` |
| 9.5 | Backfill logic for gaps | `rollup_worker.go` |

#### Deliverables
- [ ] Hourly aggregates computed
- [ ] Dashboard queries use pre-computed data
- [ ] Backfill handles gaps

---

### Phase 10: Data Retention & Cleanup
**Duration**: 1-2 days  
**Priority**: 🟡 Medium  
**Dependencies**: Phase 1

#### Objectives
- Implement partition lifecycle management
- Automated cleanup of old data
- Archive to cold storage (optional)

#### Tasks
| # | Task | Files Affected |
|---|------|----------------|
| 10.1 | Create partition management worker | `internal/tasks/partition_manager.go` (new) |
| 10.2 | Auto-create future partitions (7 days ahead) | `partition_manager.go` |
| 10.3 | Drop old telemetry partitions (>14 days) | `partition_manager.go` |
| 10.4 | Archive old payment_intents (>1 year) | `partition_manager.go` |
| 10.5 | Add retention configuration | `internal/config/config.go` |

#### Deliverables
- [ ] Automatic partition creation
- [ ] Old data cleaned up per policy
- [ ] Storage growth controlled

---

## 4. Implementation Timeline

```
Week 1: Foundation
├── Phase 1: Database Schema (Day 1-3)
└── Phase 2: Receiver Event Routing (Day 3-5)

Week 2: Core Features
├── Phase 3: SSE Integration (Day 1-2)
├── Phase 4: Receiver Observability (Day 2-3)
└── Phase 5: Backend APIs (Day 3-5)

Week 3: Integration
├── Phase 6: SSE Endpoint (Day 1-2)
├── Phase 7: Gateway Enhancements (Day 2-3)
└── Phase 8: E2E Testing (Day 3-5)

Week 4: Polish
├── Phase 9: Rollup Worker (Day 1-2)
└── Phase 10: Data Retention (Day 3-4)
```

---

## 5. Dependencies Graph

```
Phase 1 (DB Schema)
    │
    ├──────────────────────┬──────────────────────┐
    ▼                      ▼                      ▼
Phase 2 (Event Routing)  Phase 5 (Backend APIs)  Phase 9 (Rollup)
    │                      │                      │
    ├──────────┐           │                      │
    ▼          ▼           │                      │
Phase 3    Phase 4         │                      ▼
(SSE Int)  (Observ)        │                  Phase 10
    │                      │                  (Retention)
    └──────────┬───────────┘
               ▼
         Phase 6 (SSE Endpoint)
               │
               ▼
         Phase 7 (Gateway)
               │
               ▼
         Phase 8 (E2E Tests)
```

---

## 6. Success Criteria

| Metric | Target | Validation Method |
|--------|--------|-------------------|
| Event ingestion throughput | ≥ 10,000 events/sec | Stress test |
| P99 ingestion latency | ≤ 50ms | Prometheus metrics |
| SSE broadcast latency | ≤ 100ms | E2E test |
| Dashboard query latency | ≤ 200ms | `EXPLAIN ANALYZE` |
| Data loss | 0% under normal load | E2E test |
| System uptime | 99.9% | Monitoring |

---

## 7. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| DB write bottleneck | Medium | High | Use `COPY FROM` batching; partition aggressively |
| Event loss during spikes | Medium | Medium | In-memory buffer with disk fallback |
| SSE connection storms | Low | Medium | Rate limit connections; use heartbeats |
| Schema migration breaks prod | Low | High | Use `IF NOT EXISTS`; test in staging |
| Receiver downtime | Low | Low | Gateway drops events gracefully (lossy-tolerant) |

---

## 8. Files to Create/Modify Summary

### New Files
| File | Phase |
|------|-------|
| `scripts/sql/02_receiver_tables.sql` | 1 |
| `scripts/sql/03_partition_management.sql` | 1 |
| `internal/receiver/router.go` | 2 |
| `internal/receiver/parsers.go` | 2 |
| `internal/receiver/streamer.go` | 3 |
| `internal/receiver/metrics.go` | 4 |
| `internal/repository/telemetry_repo.go` | 5 |
| `internal/handlers/sql/telemetry_handler.go` | 5 |
| `internal/tasks/rollup_worker.go` | 9 |
| `internal/tasks/partition_manager.go` | 10 |
| `docker-compose.e2e.yml` | 8 |
| `tests/e2e_receiver_test.go` | 8 |
| `.github/workflows/e2e.yml` | 8 |

### Files to Modify
| File | Phase |
|------|-------|
| `internal/db/sql/schema.sql` | 1 |
| `internal/receiver/models.go` | 2 |
| `internal/receiver/worker.go` | 2, 3 |
| `cmd/receiver/main.go` | 3, 4 |
| `internal/realtime/broker.go` | 3 |
| `cmd/api/main.go` | 5 |
| `internal/handlers/sql/stream_handler.go` | 6 |
| `internal/service/proxy/handler.go` (gateway) | 7 |
| `internal/worker/metrics_streamer.go` (gateway) | 7 |
| `internal/adapter/primary/cli/init.go` (gateway) | 7 |
| `cmd/worker/main.go` | 9 |
| `Makefile` | 8 |

---

*Document Version: 1.0*  
*Created: December 2025*  
*Last Updated: December 2025*
