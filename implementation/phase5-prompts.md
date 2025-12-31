# Phase 5: Polish & Launch — Implementation Prompts

> **Phase**: Testing, Documentation, Optimization, Launch  
> **Duration**: 2 days  
> **Dependencies**: Phases 1-4 Complete  
> **Focus**: Production readiness, performance, monitoring

---

## Role Setup Prompt

```
You are the CTO and Chief Product Architect for MachPay, preparing for production launch 
of the Marketplace feature. You have deep expertise in:

- Production deployment and operational excellence
- Performance optimization and caching strategies
- Monitoring, alerting, and observability
- Technical documentation and developer experience
- Security hardening and rate limiting
- Load testing and capacity planning

MISSION: Ensure the Marketplace feature is production-ready with comprehensive testing,
documentation, performance optimization, and monitoring.

QUALITY STANDARDS:
- Test coverage > 80%
- API latency p95 < 200ms
- Zero critical security issues
- Documentation complete for all endpoints
- Monitoring dashboards operational
```

---

## Task 5.1: Backend Integration Tests

**Goal**: Comprehensive API endpoint testing.

**File**: `machpay-backend/internal/handlers/sql/discovery_handler_test.go`

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
FILE: machpay-backend/internal/handlers/sql/discovery_handler_test.go (NEW)

TASK: Create comprehensive integration tests for discovery endpoints.

TEST CASES:

1. TestHandleSearchVendors_EmptyQuery:
   - GET /v1/marketplace/vendors
   - Returns all active vendors
   - Verify default pagination (limit=20, offset=0)

2. TestHandleSearchVendors_WithQuery:
   - GET /v1/marketplace/vendors?q=weather
   - Returns only matching vendors
   - Verify full-text search works

3. TestHandleSearchVendors_CategoryFilter:
   - GET /v1/marketplace/vendors?category=ai
   - Returns only vendors in category
   - Verify category facets updated

4. TestHandleSearchVendors_SortOptions:
   - Test each sort: rating, uptime, popular, fastest, recent
   - Verify ordering is correct
   - Default sort is by relevance/rating

5. TestHandleSearchVendors_Pagination:
   - Test limit and offset parameters
   - Verify hasMore flag
   - Test max limit (100)

6. TestHandleSearchVendors_Filters:
   - Test min_uptime filter
   - Test min_rating filter
   - Combine multiple filters

7. TestHandleGetFeatured:
   - GET /v1/marketplace/vendors/featured
   - Returns vendors ordered by featured_order
   - Respects limit parameter

8. TestHandleGetVendorDetails_ByID:
   - GET /v1/marketplace/vendors/123
   - Returns full vendor details
   - Includes metrics and endpoints

9. TestHandleGetVendorDetails_BySlug:
   - GET /v1/marketplace/vendors/openweather-pro
   - Returns vendor by slug
   - Same response as by ID

10. TestHandleGetVendorDetails_NotFound:
    - GET /v1/marketplace/vendors/999999
    - Returns 404 with error message

11. TestHandleGetVendorHealth:
    - GET /v1/marketplace/vendors/123/health
    - Returns health check history
    - Respects limit parameter
    - Ordered by checked_at DESC

12. TestHandleGetCategories:
    - GET /v1/marketplace/categories
    - Returns all categories with counts
    - Sorted by count DESC

13. TestHandleQuickSearch:
    - GET /v1/marketplace/quick-search?q=open
    - Returns lightweight results
    - Fast response time

TEST SETUP:
- Use test database with fixtures
- Seed test vendors with metrics
- Clean up after tests

ASSERTIONS:
- Status codes correct
- Response body structure
- Pagination values
- Sorting order
- Empty results handling
```

---

## Task 5.2: Worker Tests

**Goal**: Test background worker logic.

**Files**: 
- `machpay-backend/internal/tasks/health_check_worker_test.go`
- `machpay-backend/internal/tasks/metrics_aggregator_worker_test.go`

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
FILES: 
- machpay-backend/internal/tasks/health_check_worker_test.go (NEW)
- machpay-backend/internal/tasks/metrics_aggregator_worker_test.go (NEW)

TASK: Create unit tests for background workers.

HEALTH CHECK WORKER TESTS:

1. TestHealthCheck_Success:
   - Mock HTTP server returns 200
   - Verify status = "up"
   - Verify response_time_ms recorded
   - Verify status_code = 200

2. TestHealthCheck_ServerError:
   - Mock HTTP server returns 500
   - Verify status = "down"
   - Verify error message recorded

3. TestHealthCheck_ClientError:
   - Mock HTTP server returns 404
   - Verify status = "degraded"

4. TestHealthCheck_Timeout:
   - Mock slow HTTP server
   - Verify status = "down"
   - Verify timeout handled

5. TestHealthCheck_SSRFProtection:
   - Test with private IP (10.0.0.1)
   - Test with localhost (127.0.0.1)
   - Test with link-local (169.254.x.x)
   - Verify all blocked

6. TestHealthCheck_Concurrency:
   - Test with multiple vendors
   - Verify concurrent execution
   - Verify no race conditions

METRICS AGGREGATOR TESTS:

1. TestAggregateUptime:
   - Insert health checks (10 up, 2 down)
   - Verify uptime = 83.33%

2. TestAggregateLatency:
   - Insert telemetry with known latencies
   - Verify avg, p95, p99 calculated

3. TestAggregateVolume:
   - Insert payment intents
   - Verify total_requests count
   - Verify success_rate

4. TestAggregateRevenue:
   - Insert settled payments
   - Verify total_revenue_usd

5. TestAggregateUniqueAgents:
   - Insert payments from different agents
   - Verify unique_agents count

6. TestCalculateRating:
   - Start at 5.0
   - Add upheld dispute
   - Verify rating reduced

7. TestMetricsUpsert:
   - Aggregate metrics
   - Verify upserted to vendor_metrics
   - Run again, verify updated

MOCKING:
- Use testify/mock for repositories
- Use httptest for HTTP servers
- Use test database for integration

CLEANUP:
- Reset database between tests
- Clean up mock servers
```

---

## Task 5.3: Console Component Tests

**Goal**: React component testing.

**Files**:
- `machpay-console/src/pages/Marketplace.test.jsx`
- `machpay-console/src/components/marketplace/VendorCard.test.jsx`

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
FILES:
- machpay-console/src/pages/Marketplace.test.jsx (NEW)
- machpay-console/src/components/marketplace/VendorCard.test.jsx (NEW)

TASK: Create React component tests.

MARKETPLACE PAGE TESTS:

1. test_renders_without_crashing:
   - Mount component
   - No errors thrown

2. test_shows_loading_skeleton:
   - Initial render
   - Loading skeleton visible
   - Vendor cards not visible

3. test_displays_vendors:
   - Mock API response
   - Vendor cards rendered
   - Correct count displayed

4. test_search_debounces:
   - Type in search input
   - Verify debounce (300ms)
   - API called once after debounce

5. test_category_filter:
   - Click category pill
   - Verify API called with category
   - Results updated

6. test_sort_dropdown:
   - Change sort option
   - Verify API called with sort
   - Results reordered

7. test_load_more:
   - Click load more button
   - Verify offset incremented
   - Vendors appended

8. test_empty_state:
   - Mock empty response
   - Empty state message shown
   - Clear filters link works

9. test_error_state:
   - Mock API error
   - Error message displayed
   - Retry button works

10. test_modal_opens:
    - Click vendor card
    - Modal opens
    - Correct vendor data displayed

VENDOR CARD TESTS:

1. test_renders_vendor_data:
   - Pass vendor prop
   - Name displayed
   - Category displayed
   - Metrics displayed

2. test_verified_badge:
   - isVerified=true → badge visible
   - isVerified=false → badge hidden

3. test_uptime_color:
   - uptime > 99.5 → green
   - uptime > 99 → yellow
   - uptime < 99 → red

4. test_click_handler:
   - Click card
   - onClick called with vendor

5. test_price_formatting:
   - price=0.0001 → "$0.0001"
   - price=1.5 → "$1.50"

6. test_request_formatting:
   - 1234 → "1.2K"
   - 1234567 → "1.2M"

TESTING LIBRARIES:
- @testing-library/react
- @testing-library/jest-dom
- vitest or jest
- Mock API with msw or vi.mock
```

---

## Task 5.4: API Documentation

**Goal**: OpenAPI/Swagger documentation.

**File**: `machpay-docs/api/marketplace.md`

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
FILE: machpay-docs/api/marketplace.md (NEW)

TASK: Create comprehensive API documentation for marketplace endpoints.

STRUCTURE:

# Marketplace API Reference

## Overview
- Base URL: https://api.machpay.xyz/v1
- Authentication: None required (public API)
- Rate Limit: 100 requests/minute per IP

## Endpoints

### Search Vendors
GET /marketplace/vendors

Search and filter vendors in the marketplace.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| q | string | "" | Search query (natural language) |
| category | string | "" | Filter by category |
| sort | string | "relevance" | Sort: relevance, rating, uptime, popular, fastest, recent |
| min_uptime | float | - | Minimum uptime percentage |
| min_rating | float | - | Minimum rating (0-5) |
| limit | int | 20 | Results per page (max 100) |
| offset | int | 0 | Pagination offset |

**Response:**
```json
{
  "vendors": [...],
  "total": 142,
  "hasMore": true,
  "categories": [
    {"category": "ai", "count": 45}
  ]
}
```

### Advanced Search
POST /marketplace/search

Full-featured search with body parameters.

**Request Body:**
```json
{
  "query": "weather API",
  "categories": ["weather", "data"],
  "tags": ["enterprise"],
  "minUptime": 99.0,
  "maxLatency": 100,
  "minRating": 4.0,
  "sortBy": "rating",
  "limit": 20,
  "offset": 0
}
```

### Get Featured Vendors
GET /marketplace/vendors/featured

Returns curated list of featured vendors.

### Get Vendor Details
GET /marketplace/vendors/:id

Get detailed information for a vendor by ID or slug.

### Get Vendor Health
GET /marketplace/vendors/:id/health

Get recent health check history.

### Get Categories
GET /marketplace/categories

Get all categories with vendor counts.

### Quick Search
GET /marketplace/quick-search

Lightweight search for autocomplete.

## Response Types

### VendorDiscoveryResult
```typescript
{
  id: number;
  appId: string;
  slug: string;
  serviceName: string;
  description: string;
  logoUrl?: string;
  category: string;
  defaultPriceUsd: number;
  tags: string[];
  isVerified: boolean;
  featuredOrder?: number;
  // Metrics
  uptimePct: number;
  avgLatencyMs: number;
  ratingScore: number;
  totalRequests: number;
  successRate: number;
  uniqueAgents: number;
  lastRequestAt?: string;
  // Search
  relevanceScore?: number;
}
```

## Error Responses

All errors follow format:
```json
{
  "error": "error_code",
  "message": "Human readable message"
}
```

| Status | Error Code | Description |
|--------|------------|-------------|
| 400 | invalid_request | Malformed request |
| 404 | not_found | Vendor not found |
| 429 | rate_limited | Too many requests |
| 500 | internal_error | Server error |

## Examples

### cURL
```bash
# Search for weather APIs
curl -X GET "https://api.machpay.xyz/v1/marketplace/vendors?q=weather&category=weather&sort=rating"

# Get vendor details
curl -X GET "https://api.machpay.xyz/v1/marketplace/vendors/openweather-pro"
```

### Python
```python
import requests

response = requests.get(
    "https://api.machpay.xyz/v1/marketplace/vendors",
    params={"q": "weather", "min_uptime": 99.0}
)
vendors = response.json()["vendors"]
```

### JavaScript
```javascript
const response = await fetch(
  "https://api.machpay.xyz/v1/marketplace/vendors?q=weather"
);
const { vendors } = await response.json();
```
```

---

## Task 5.5: Performance Optimization

**Goal**: Caching and query optimization.

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
TASK: Implement performance optimizations.

1. REDIS CACHING (if Redis available):

   File: machpay-backend/internal/services/discovery_service.go

   Add caching layer:
   - Cache key: discovery:search:{hash(params)}
   - TTL: 60 seconds for search results
   - TTL: 300 seconds for categories
   - TTL: 300 seconds for featured
   - Invalidate on vendor metrics update

2. DATABASE QUERY OPTIMIZATION:

   File: machpay-backend/internal/repository/vendor_metrics_repo.go

   Optimizations:
   - Add covering index for search
   - Use prepared statements
   - Limit returned columns in search
   - Use EXPLAIN ANALYZE to verify index usage

   Query to optimize (sqlDiscoverySearchBase):
   - Ensure ts_rank uses index
   - Avoid seq scans on large tables
   - Consider materialized view for hot searches

3. CONSOLE OPTIMIZATIONS:

   Files: machpay-console/src/pages/Marketplace.jsx
          machpay-console/src/components/marketplace/VendorCard.jsx

   Optimizations:
   - React.memo on VendorCard
   - useMemo for filtered/sorted lists
   - useCallback for handlers
   - Lazy load vendor logos
   - Skeleton loading states
   - Virtualize large lists (react-window)

4. CONNECTION POOLING:

   File: machpay-backend/cmd/worker/main.go

   Verify pool settings:
   - db.SetMaxOpenConns(10)
   - db.SetMaxIdleConns(5)
   - db.SetConnMaxLifetime(5 * time.Minute)

5. RATE LIMITING:

   File: machpay-backend/cmd/api/main.go

   Add rate limit middleware to marketplace routes:
   - 100 requests/minute per IP
   - 429 response when exceeded
   - X-RateLimit-* headers

PERFORMANCE TARGETS:
- Search p95 < 200ms
- Featured p95 < 100ms
- Categories p95 < 50ms
- Detail p95 < 150ms
```

---

## Task 5.6: Monitoring & Alerting

**Goal**: Prometheus metrics and alerts.

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
TASK: Add monitoring and alerting.

1. PROMETHEUS METRICS:

   File: machpay-backend/internal/handlers/sql/discovery_handler.go

   Add metrics:
   - discovery_search_duration_seconds (histogram)
   - discovery_search_results_total (histogram)
   - discovery_requests_total (counter, labels: endpoint)
   - discovery_errors_total (counter, labels: endpoint, error_type)

   File: machpay-backend/internal/tasks/health_check_worker.go

   Add metrics:
   - health_check_duration_seconds (histogram, labels: vendor_id)
   - health_check_status_total (counter, labels: vendor_id, status)
   - health_check_errors_total (counter)

   File: machpay-backend/internal/tasks/metrics_aggregator_worker.go

   Add metrics:
   - metrics_aggregation_duration_seconds (histogram)
   - metrics_vendors_processed_total (counter)
   - metrics_aggregation_errors_total (counter)

2. GRAFANA DASHBOARD:

   Create dashboard with panels:
   - Search latency p50/p95/p99
   - Search requests per minute
   - Error rate by endpoint
   - Health check success rate
   - Vendor uptime distribution
   - Worker status (up/down)

3. ALERTS:

   Create alerting rules:

   - DiscoveryHighLatency:
     expr: histogram_quantile(0.95, discovery_search_duration_seconds) > 0.5
     for: 5m
     severity: warning

   - DiscoveryErrorRate:
     expr: rate(discovery_errors_total[5m]) / rate(discovery_requests_total[5m]) > 0.01
     for: 5m
     severity: warning

   - HealthCheckWorkerDown:
     expr: up{job="health_checker"} == 0
     for: 5m
     severity: critical

   - MetricsAggregatorDown:
     expr: up{job="metrics_aggregator"} == 0
     for: 10m
     severity: warning

   - VendorUptimeLow:
     expr: vendor_uptime_pct < 95
     for: 15m
     severity: warning

4. HEALTH ENDPOINTS:

   File: machpay-backend/cmd/api/main.go

   Add endpoints:
   - GET /health/ready - Returns 200 if ready to serve
   - GET /health/live - Returns 200 if process alive
   - GET /health/workers - Returns worker status

   Response:
   {
     "status": "healthy",
     "workers": {
       "healthChecker": { "running": true, "lastRun": "..." },
       "metricsAggregator": { "running": true, "lastRun": "..." }
     }
   }
```

---

## Task 5.7: Launch Checklist

**Goal**: Pre-launch verification.

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
TASK: Complete launch checklist.

## Pre-Launch Checklist

### Code Quality
[ ] All tests passing (go test ./...)
[ ] No linter errors (golangci-lint run)
[ ] Console builds without errors (npm run build)
[ ] SDK tests passing (pytest)

### Database
[ ] Migration tested on staging
[ ] Rollback script tested
[ ] Indexes verified with EXPLAIN ANALYZE
[ ] vendor_metrics has rows for all vendors
[ ] Partitions created for next 3 months

### Backend
[ ] All endpoints respond correctly
[ ] Error handling verified
[ ] Rate limiting active
[ ] CORS configured correctly
[ ] Logging at appropriate levels

### Workers
[ ] HealthCheckWorker running
[ ] MetricsAggregatorWorker running
[ ] Partition manager includes vendor_health_checks
[ ] Graceful shutdown works

### Console
[ ] /marketplace route accessible
[ ] Search works with all filters
[ ] Pagination works
[ ] Responsive on mobile
[ ] Loading states work
[ ] Error states work
[ ] Accessibility audit passed

### SDK
[ ] DiscoveryClient works
[ ] All methods return correct data
[ ] Error handling works
[ ] Documentation complete

### Documentation
[ ] API reference complete
[ ] Console user guide complete
[ ] SDK examples complete
[ ] README updated

### Monitoring
[ ] Prometheus metrics exposed
[ ] Grafana dashboard created
[ ] Alerts configured
[ ] PagerDuty/Slack integration

### Security
[ ] No exposed secrets
[ ] SSRF protection verified
[ ] Rate limiting verified
[ ] Input validation complete

## Launch Day

1. [ ] Run migration on production
2. [ ] Deploy backend with new workers
3. [ ] Verify workers started (check logs)
4. [ ] Deploy console with marketplace
5. [ ] Verify /marketplace loads
6. [ ] Test search functionality
7. [ ] Monitor error rates (< 1%)
8. [ ] Monitor latencies (p95 < 200ms)

## Post-Launch (24h)

1. [ ] Check vendor_metrics populated
2. [ ] Check health_checks recording
3. [ ] Verify no memory leaks
4. [ ] Review error logs
5. [ ] Gather user feedback
6. [ ] Fix critical bugs if any
7. [ ] Document any issues found
```

---

## Task 5.8: Load Testing

**Goal**: Verify performance under load.

**Prompt**:
```
CONTEXT: Phase 5 Polish & Launch for MachPay Marketplace.
TASK: Create and run load tests.

TOOL: k6, vegeta, or hey

SCENARIOS:

1. Search Endpoint Load Test:
   Target: GET /v1/marketplace/vendors?q=weather
   Rate: 100 requests/second
   Duration: 5 minutes
   
   Success criteria:
   - p95 latency < 200ms
   - Error rate < 0.1%
   - No 5xx errors

2. Featured Endpoint Load Test:
   Target: GET /v1/marketplace/vendors/featured
   Rate: 50 requests/second
   Duration: 5 minutes
   
   Success criteria:
   - p95 latency < 100ms

3. Spike Test:
   Start: 10 rps
   Spike to: 500 rps
   Duration: 2 minutes
   
   Success criteria:
   - System recovers after spike
   - No OOM errors

4. Soak Test:
   Rate: 50 rps
   Duration: 1 hour
   
   Success criteria:
   - No memory leaks
   - Consistent latency
   - No degradation

K6 SCRIPT EXAMPLE:
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '3m', target: 100 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  let response = http.get('http://localhost:8080/v1/marketplace/vendors?q=weather');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'has vendors': (r) => JSON.parse(r.body).vendors.length > 0,
  });
  sleep(0.1);
}
```

RECORD RESULTS:
- Document baseline performance
- Compare with targets
- Identify bottlenecks
- Create optimization issues if needed
```

---

## Progress Tracking

| Task | Description | Status |
|------|-------------|--------|
| 5.1 | Backend Integration Tests | ✅ Complete |
| 5.2 | Worker Tests | ✅ Complete |
| 5.3 | Console Component Tests | ⬜ Pending |
| 5.4 | API Documentation | ✅ Complete |
| 5.5 | Performance Optimization | ⬜ Pending |
| 5.6 | Monitoring & Alerting | ⬜ Pending |
| 5.7 | Launch Checklist | ✅ Complete |
| 5.8 | Load Testing | ⬜ Pending |
| ✓ | Production Launch | ⬜ Ready |

---

## Notes

- Prioritize critical path items
- Don't let perfect be enemy of good
- Document known issues for post-launch
- Have rollback plan ready
- Monitor closely for first 24 hours

