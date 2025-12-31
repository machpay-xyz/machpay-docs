# MachPay Marketplace — Launch Checklist

> **Feature**: Vendor Discovery Marketplace  
> **Version**: v1.0.0  
> **Target Launch Date**: _______________

---

## Pre-Launch Verification

### 1. Code Quality ✅

| Item | Status | Notes |
|------|--------|-------|
| Backend builds: `go build ./...` | ⬜ | |
| Backend tests pass: `go test ./...` | ⬜ | |
| No linter errors: `golangci-lint run` | ⬜ | |
| Console builds: `npm run build` | ⬜ | |
| Console tests pass: `npm run test` | ⬜ | |
| SDK tests pass: `pytest` | ⬜ | |
| All 34 SDK discovery tests pass | ✅ | Verified |

### 2. Database ✅

| Item | Status | Notes |
|------|--------|-------|
| Migration `001_marketplace_tables.sql` applied | ⬜ | |
| Rollback script tested | ⬜ | |
| `vendor_metrics` table exists | ⬜ | |
| `vendor_health_checks` table exists | ⬜ | |
| `vendors.is_verified` column exists | ⬜ | |
| `vendors.featured_order` column exists | ⬜ | |
| `vendors.tags` column exists | ⬜ | |
| Indexes verified with `EXPLAIN ANALYZE` | ⬜ | |
| Partitions created for next 3 months | ⬜ | |
| Initial `vendor_metrics` rows populated | ⬜ | |

### 3. Backend API ✅

| Endpoint | Status | Notes |
|----------|--------|-------|
| `GET /v1/marketplace/search` | ⬜ | |
| `POST /v1/marketplace/search` | ⬜ | |
| `GET /v1/marketplace/featured` | ⬜ | |
| `GET /v1/marketplace/vendors/:id` | ⬜ | |
| `GET /v1/marketplace/vendors/:id/health` | ⬜ | |
| `GET /v1/marketplace/categories` | ⬜ | |
| `GET /v1/marketplace/categories/:cat` | ⬜ | |
| `GET /v1/marketplace/quick-search` | ⬜ | |
| Rate limiting active (100/min) | ⬜ | |
| CORS configured correctly | ⬜ | |
| Error responses formatted | ⬜ | |

### 4. Background Workers ✅

| Item | Status | Notes |
|------|--------|-------|
| `HealthCheckWorker` registered | ✅ | `cmd/api/main.go` |
| `MetricsAggregatorWorker` registered | ✅ | `cmd/api/main.go` |
| Worker CLI flags available | ✅ | `cmd/worker/main.go` |
| `PartitionManager` includes `vendor_health_checks` | ✅ | |
| SSRF protection verified | ✅ | Private IPs blocked |
| Graceful shutdown works | ⬜ | Test with SIGTERM |

### 5. Console UI ✅

| Item | Status | Notes |
|------|--------|-------|
| `/marketplace` route accessible | ✅ | |
| Search functionality works | ⬜ | |
| Category filter works | ⬜ | |
| Sort dropdown works | ⬜ | |
| Pagination (Load More) works | ⬜ | |
| Vendor details panel works | ⬜ | |
| Featured vendors display | ⬜ | |
| Loading states work | ⬜ | |
| Error states work | ⬜ | |
| Empty state works | ⬜ | |
| Responsive on mobile | ⬜ | |
| Matches console theme | ✅ | Bloomberg Terminal aesthetic |
| Navigation item in sidebar | ✅ | Store icon, all roles |

### 6. Python SDK ✅

| Item | Status | Notes |
|------|--------|-------|
| `DiscoveryClient` class | ✅ | |
| `search()` method | ✅ | |
| `get_featured()` method | ✅ | |
| `get_vendor()` method | ✅ | |
| `get_vendor_health()` method | ✅ | |
| `get_categories()` method | ✅ | |
| `quick_search()` method | ✅ | |
| Helper methods (`find_best_vendor`, etc.) | ✅ | |
| `MachPayClient.discovery` property | ✅ | |
| Package exports updated | ✅ | |
| README examples added | ✅ | |
| All tests pass (34/34) | ✅ | |

### 7. Documentation ✅

| Item | Status | Notes |
|------|--------|-------|
| API Reference (`api/marketplace.md`) | ✅ | Complete |
| SDK usage examples | ✅ | In README |
| Phase prompts documented | ✅ | All 5 phases |
| Implementation plan | ✅ | |

### 8. Security ✅

| Item | Status | Notes |
|------|--------|-------|
| No exposed secrets | ⬜ | Check env vars |
| SSRF protection in health checks | ✅ | Private IPs blocked |
| Rate limiting active | ⬜ | 100 req/min/IP |
| Input validation complete | ⬜ | Query, limit, offset |
| SQL injection prevention | ✅ | Parameterized queries |

---

## Launch Day Procedure

### Step 1: Database Migration

```bash
# Apply migration
psql $DATABASE_URL -f internal/db/migrations/001_marketplace_tables.sql

# Verify tables exist
psql $DATABASE_URL -c "SELECT COUNT(*) FROM vendor_metrics;"
psql $DATABASE_URL -c "SELECT COUNT(*) FROM vendor_health_checks;"

# Initialize metrics for existing vendors
# This will be done by the MetricsAggregatorWorker
```

### Step 2: Deploy Backend

```bash
# Build and deploy
docker build -t machpay-backend:v1.x.x .
docker push machpay-backend:v1.x.x

# Deploy with new workers enabled
kubectl apply -f k8s/backend-deployment.yaml

# Verify workers started
kubectl logs -l app=machpay-backend | grep -E "HealthCheck|Metrics"
```

### Step 3: Verify Workers

```bash
# Check health check worker running
curl http://localhost:8080/health/workers

# Expected response:
# {
#   "status": "healthy",
#   "workers": {
#     "healthChecker": { "running": true, "lastRun": "..." },
#     "metricsAggregator": { "running": true, "lastRun": "..." }
#   }
# }
```

### Step 4: Deploy Console

```bash
# Build and deploy console
npm run build
# Deploy to CDN/hosting

# Verify marketplace page loads
curl -I https://console.machpay.xyz/marketplace
```

### Step 5: Smoke Tests

```bash
# Test search endpoint
curl "https://api.machpay.xyz/v1/marketplace/search?q=weather" | jq .

# Test featured endpoint
curl "https://api.machpay.xyz/v1/marketplace/featured" | jq .

# Test categories
curl "https://api.machpay.xyz/v1/marketplace/categories" | jq .

# Test vendor details
curl "https://api.machpay.xyz/v1/marketplace/vendors/1" | jq .
```

### Step 6: Monitor

- [ ] Check error rates in monitoring dashboard (< 1%)
- [ ] Check API latencies (p95 < 200ms)
- [ ] Check worker status (running)
- [ ] Check database connections (< 80% capacity)
- [ ] Check memory usage (no leaks)

---

## Post-Launch (24h) Verification

### Monitoring Checks

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Search p95 latency | < 200ms | ___ms | ⬜ |
| Featured p95 latency | < 100ms | ___ms | ⬜ |
| Error rate | < 1% | ___% | ⬜ |
| Worker uptime | 100% | ___% | ⬜ |
| DB connection pool | < 80% | ___% | ⬜ |

### Data Checks

- [ ] `vendor_metrics` rows populated for all vendors
- [ ] `vendor_health_checks` recording (check row count)
- [ ] Partitions creating automatically
- [ ] No stale data (metrics updated within 5 min)

### Functionality Checks

- [ ] Search returns relevant results
- [ ] Filters work correctly
- [ ] Sorting produces expected order
- [ ] Pagination works without duplicates
- [ ] Health history accurate

### User Feedback

- [ ] Console UX is smooth
- [ ] Search is fast
- [ ] Results are relevant
- [ ] No confusion on navigation

---

## Rollback Procedure

If critical issues are found:

### 1. Rollback Backend

```bash
# Revert to previous version
kubectl rollout undo deployment/machpay-backend

# Verify old version running
kubectl get pods -l app=machpay-backend
```

### 2. Rollback Database (if needed)

```bash
# Run rollback script
psql $DATABASE_URL -f internal/db/migrations/001_marketplace_tables_rollback.sql

# Verify tables removed
psql $DATABASE_URL -c "SELECT to_regclass('vendor_metrics');"
# Should return NULL
```

### 3. Rollback Console

```bash
# Deploy previous version
# Or: Remove marketplace route from nginx/CDN
```

---

## Known Issues / Technical Debt

| Issue | Severity | Workaround | Future Fix |
|-------|----------|------------|------------|
| No Redis caching | Low | DB handles load | Add Redis layer |
| No semantic search | Medium | Full-text works | Add pgvector |
| Basic rating calc | Low | Works for MVP | Add ML model |

---

## Sign-off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| CTO | | | |
| Lead Engineer | | | |
| QA Lead | | | |
| DevOps | | | |

---

**Launch Approved**: ⬜ Yes / ⬜ No

**Notes**:

