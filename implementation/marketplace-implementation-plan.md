# 🏪 MachPay Marketplace — Multi-Phase Implementation Plan

> **Status**: Planning  
> **Author**: CTO  
> **Created**: 2024-12-31  
> **Last Updated**: 2024-12-31

---

## Executive Summary

This document outlines the complete implementation plan for the **Vendor Discovery Marketplace** feature. The marketplace enables AI agents to discover, compare, and connect to API vendors based on real-time metrics including uptime, latency, pricing, and reputation.

### Goals

1. **Discoverability**: Agents can search vendors by natural language queries
2. **Transparency**: Real metrics (uptime, latency, rating) visible before connecting
3. **Comparison**: Sort and filter by price, performance, category
4. **Integration**: One-click access to vendor integration details

### Timeline Overview

| Phase | Name | Duration | Focus |
|-------|------|----------|-------|
| **Phase 1** | Database Foundation | 2 days | Schema + migrations |
| **Phase 2** | Backend API | 3 days | Discovery endpoints + metrics workers |
| **Phase 3** | Console UI | 3 days | Marketplace page + components |
| **Phase 4** | SDK Integration | 1 day | Python SDK discovery module |
| **Phase 5** | Polish & Launch | 2 days | Testing, docs, optimization |

**Total Estimated Duration: 11 days**

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CONSOLE UI                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  /marketplace                                                    │    │
│  │  ┌──────────┐ ┌──────────────────────────────────────────────┐  │    │
│  │  │ Search   │ │ VendorCard  VendorCard  VendorCard           │  │    │
│  │  │ Filters  │ │ VendorCard  VendorCard  VendorCard           │  │    │
│  │  │ Sort     │ │ VendorCard  VendorCard  VendorCard           │  │    │
│  │  └──────────┘ └──────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            BACKEND API                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  GET /v1/discovery/vendors?q=weather&category=data&sort=rating  │    │
│  │  GET /v1/discovery/vendors/featured                             │    │
│  │  GET /v1/discovery/vendors/:id                                  │    │
│  │  GET /v1/discovery/categories                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            DATABASE                                      │
│  ┌────────────────┐  ┌──────────────────┐  ┌────────────────────────┐   │
│  │    vendors     │  │  vendor_metrics  │  │  vendor_health_checks  │   │
│  │  (existing)    │◄─│  (new)           │◄─│  (new)                 │   │
│  └────────────────┘  └──────────────────┘  └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                         BACKGROUND WORKERS                               │
│  ┌─────────────────────────┐  ┌─────────────────────────────────────┐   │
│  │  HealthCheckWorker      │  │  MetricsAggregatorWorker            │   │
│  │  (every 1 min)          │  │  (every 5 min)                      │   │
│  │  - Ping vendor health   │  │  - Aggregate gateway_telemetry      │   │
│  │  - Record up/down       │  │  - Calculate uptime, latency, rate  │   │
│  └─────────────────────────┘  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Database Foundation

**Duration**: 2 days  
**Dependencies**: None  
**Owner**: Backend Engineer

### 1.1 New Table: `vendor_metrics`

**Purpose**: Store aggregated performance metrics for each vendor, updated by background worker.

**File to modify**: `machpay-backend/internal/db/sql/schema.sql`

**Table Schema**:

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| `vendor_id` | INTEGER | - | PK, FK to vendors(id) |
| `uptime_pct` | DECIMAL(5,2) | 100.00 | Percentage uptime (0-100) |
| `avg_latency_ms` | INTEGER | 0 | Average response time |
| `p95_latency_ms` | INTEGER | 0 | 95th percentile latency |
| `p99_latency_ms` | INTEGER | 0 | 99th percentile latency |
| `rating_score` | DECIMAL(3,2) | 5.00 | Rating out of 5.0 |
| `total_requests` | BIGINT | 0 | Lifetime request count |
| `success_rate` | DECIMAL(5,2) | 100.00 | % of successful requests |
| `error_rate` | DECIMAL(5,2) | 0.00 | % of failed requests |
| `total_revenue_usd` | DECIMAL(18,6) | 0 | Lifetime revenue |
| `unique_agents` | INTEGER | 0 | Distinct agents served |
| `last_request_at` | TIMESTAMP | NULL | Most recent request time |
| `updated_at` | TIMESTAMP | NOW() | Last metrics update |

**Indexes to create**:
- `idx_vendor_metrics_rating` on `(rating_score DESC)`
- `idx_vendor_metrics_uptime` on `(uptime_pct DESC)`
- `idx_vendor_metrics_requests` on `(total_requests DESC)`
- `idx_vendor_metrics_last_request` on `(last_request_at DESC NULLS LAST)`

---

### 1.2 New Table: `vendor_health_checks`

**Purpose**: Store individual health check results for uptime calculation.

**File to modify**: `machpay-backend/internal/db/sql/schema.sql`

**Table Schema**:

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| `id` | SERIAL | - | Primary key |
| `vendor_id` | INTEGER | - | FK to vendors(id) |
| `status` | VARCHAR(10) | - | 'up', 'down', 'degraded' |
| `response_time_ms` | INTEGER | NULL | Response time if successful |
| `status_code` | INTEGER | NULL | HTTP status code |
| `error_message` | TEXT | NULL | Error details if failed |
| `checked_at` | TIMESTAMP | NOW() | Check timestamp |

**Indexes to create**:
- `idx_health_checks_vendor_time` on `(vendor_id, checked_at DESC)`
- `idx_health_checks_checked_at` on `(checked_at)` for partition pruning

**Partitioning**:
- Partition by RANGE on `checked_at` (monthly partitions)
- Retention: Keep last 90 days, drop older partitions

---

### 1.3 Alter Table: `vendors`

**Purpose**: Add columns for marketplace features.

**File to modify**: `machpay-backend/internal/db/sql/schema.sql`

**New columns to add**:

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| `is_verified` | BOOLEAN | FALSE | Verified vendor badge |
| `featured_order` | INTEGER | NULL | Position in featured list (NULL = not featured) |
| `tags` | TEXT[] | '{}' | Searchable tags array |

**Indexes to create**:
- `idx_vendors_tags` using GIN on `(tags)`
- `idx_vendors_fts` using GIN on `to_tsvector('english', service_name || ' ' || COALESCE(description, ''))`
- `idx_vendors_featured` on `(featured_order)` WHERE `featured_order IS NOT NULL`
- `idx_vendors_verified` on `(is_verified)` WHERE `is_verified = TRUE`

---

### 1.4 Migration Script

**New file to create**: `machpay-backend/internal/db/migrations/007_marketplace_foundation.sql`

**Migration steps**:
1. Create `vendor_metrics` table
2. Create `vendor_health_checks` table with partitioning
3. Alter `vendors` table to add new columns
4. Create all indexes
5. Insert initial `vendor_metrics` row for each existing vendor with default values
6. Create function for automatic partition management

**Rollback script**:
1. Drop indexes
2. Drop `vendor_health_checks` table
3. Drop `vendor_metrics` table
4. Alter `vendors` to drop new columns

---

### 1.5 Phase 1 Acceptance Criteria

- [ ] Migration runs successfully on fresh database
- [ ] Migration runs successfully on existing database with vendor data
- [ ] All indexes created and verified with `\di` in psql
- [ ] Rollback script tested and works
- [ ] vendor_metrics has one row per existing vendor

---

## Phase 2: Backend API

**Duration**: 3 days  
**Dependencies**: Phase 1 complete  
**Owner**: Backend Engineer

### 2.1 New Model Structs

**File to modify**: `machpay-backend/internal/models/sql/vendor.go`

**New structs to add**:

```
VendorMetrics
├── VendorID        int64
├── UptimePct       float64
├── AvgLatencyMs    int
├── P95LatencyMs    int
├── P99LatencyMs    int
├── RatingScore     float64
├── TotalRequests   int64
├── SuccessRate     float64
├── ErrorRate       float64
├── TotalRevenueUSD float64
├── UniqueAgents    int
├── LastRequestAt   *time.Time
└── UpdatedAt       time.Time

VendorWithMetrics
├── Vendor          (embedded)
└── Metrics         VendorMetrics

VendorSearchResult (for API responses)
├── ID              int64
├── AppID           string
├── Slug            string
├── Name            string
├── Description     string
├── Category        string
├── LogoURL         string
├── IsVerified      bool
├── Pricing         VendorPricingInfo
└── Metrics         VendorMetricsInfo

VendorPricingInfo
├── Model           string   (per_request, per_token, etc.)
├── PriceUSD        float64
└── Currency        string

VendorMetricsInfo
├── Uptime          float64
├── AvgLatencyMs    int
├── Rating          float64
├── TotalRequests   int64
└── SuccessRate     float64

SearchQuery
├── Query           string   (search text)
├── Category        string   (filter by category)
├── Sort            string   (rating, uptime, price_asc, price_desc, popular, fastest)
├── MinUptime       float64  (minimum uptime %)
├── MinRating       float64  (minimum rating)
├── Limit           int      (pagination)
├── Offset          int      (pagination)

SearchResponse
├── Vendors         []VendorSearchResult
├── Total           int
├── Limit           int
├── Offset          int
└── HasMore         bool
```

**Methods to add**:
- `(v *VendorWithMetrics) ToSearchResult() *VendorSearchResult`

---

### 2.2 New Repository: VendorMetricsRepo

**New file to create**: `machpay-backend/internal/repository/vendor_metrics_repo.go`

**Interface definition**:

```
VendorMetricsRepo interface {
    GetByVendorID(ctx, vendorID int64) (*VendorMetrics, error)
    Upsert(ctx, metrics *VendorMetrics) error
    BulkUpsert(ctx, metrics []*VendorMetrics) error
    GetTopByRating(ctx, limit int) ([]*VendorMetrics, error)
    GetTopByUptime(ctx, limit int) ([]*VendorMetrics, error)
    IncrementRequests(ctx, vendorID int64, count int64) error
    UpdateLastRequest(ctx, vendorID int64) error
}
```

**SQL queries to implement**:
- `sqlMetricsSelectByVendor`: SELECT * FROM vendor_metrics WHERE vendor_id = $1
- `sqlMetricsUpsert`: INSERT ... ON CONFLICT (vendor_id) DO UPDATE
- `sqlMetricsTopRating`: SELECT * FROM vendor_metrics ORDER BY rating_score DESC LIMIT $1
- `sqlMetricsTopUptime`: SELECT * FROM vendor_metrics ORDER BY uptime_pct DESC LIMIT $1

---

### 2.3 Enhanced VendorRepo

**File to modify**: `machpay-backend/internal/repository/vendor_repo.go`

**New interface methods to add**:

```
VendorRepo interface {
    // ... existing methods ...
    
    // New methods for marketplace
    SearchWithMetrics(ctx, query *SearchQuery) ([]*VendorWithMetrics, int, error)
    GetFeatured(ctx, limit int) ([]*VendorWithMetrics, error)
    GetByCategory(ctx, category string, limit, offset int) ([]*VendorWithMetrics, error)
    GetCategories(ctx) ([]CategoryCount, error)
    GetByIDWithMetrics(ctx, id int64) (*VendorWithMetrics, error)
}

CategoryCount struct {
    Category string
    Count    int
}
```

**New SQL query: `sqlVendorSearchWithMetrics`**

Query logic:
1. JOIN vendors with vendor_metrics
2. WHERE clause:
   - `status = 'active'`
   - Full-text search on `to_tsvector(service_name || description)` if query provided
   - ILIKE fallback: `service_name ILIKE '%query%' OR description ILIKE '%query%'`
   - Category filter: `category = $category` if provided
   - Uptime filter: `uptime_pct >= $min_uptime` if provided
   - Rating filter: `rating_score >= $min_rating` if provided
3. ORDER BY clause based on sort parameter:
   - `rating`: `rating_score DESC, total_requests DESC`
   - `uptime`: `uptime_pct DESC, rating_score DESC`
   - `price_asc`: `default_price_usd ASC NULLS LAST`
   - `price_desc`: `default_price_usd DESC NULLS FIRST`
   - `popular`: `total_requests DESC, rating_score DESC`
   - `fastest`: `avg_latency_ms ASC NULLS LAST`
4. LIMIT and OFFSET for pagination
5. Separate COUNT(*) query for total

---

### 2.4 New Handler: DiscoveryHandler

**New file to create**: `machpay-backend/internal/handlers/sql/discovery_handler.go`

**Handler struct**:

```
DiscoveryHandler struct {
    vendorRepo       VendorRepo
    vendorMetrics    VendorMetricsRepo
}
```

**Endpoints to implement**:

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | `/v1/discovery/vendors` | `HandleSearchVendors` | Search with filters |
| GET | `/v1/discovery/vendors/featured` | `HandleGetFeatured` | Featured vendors |
| GET | `/v1/discovery/vendors/:id` | `HandleGetVendorDetail` | Single vendor detail |
| GET | `/v1/discovery/categories` | `HandleGetCategories` | Category list with counts |
| GET | `/v1/discovery/stats` | `HandleGetStats` | Marketplace statistics |

**HandleSearchVendors query parameters**:

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `q` | string | "" | Search query |
| `category` | string | "" | Category filter |
| `sort` | string | "rating" | Sort option |
| `min_uptime` | float | 0 | Minimum uptime % |
| `min_rating` | float | 0 | Minimum rating |
| `limit` | int | 20 | Results per page (max 100) |
| `offset` | int | 0 | Pagination offset |

**Response format for HandleSearchVendors**:

```json
{
  "vendors": [
    {
      "id": 1,
      "appId": "mp_app_xyz123",
      "slug": "openweather-api",
      "name": "OpenWeather Pro",
      "description": "Real-time weather data...",
      "category": "weather",
      "logoUrl": "https://...",
      "isVerified": true,
      "pricing": {
        "model": "per_request",
        "priceUsd": 0.0001,
        "currency": "USDC"
      },
      "metrics": {
        "uptime": 99.98,
        "avgLatencyMs": 42,
        "rating": 4.9,
        "totalRequests": 1250000,
        "successRate": 99.95
      }
    }
  ],
  "total": 142,
  "limit": 20,
  "offset": 0,
  "hasMore": true
}
```

---

### 2.5 New Worker: MetricsAggregatorWorker

**New file to create**: `machpay-backend/internal/worker/metrics_aggregator.go`

**Worker struct**:

```
MetricsAggregatorWorker struct {
    db              *sql.DB
    metricsRepo     VendorMetricsRepo
    interval        time.Duration  // default: 5 minutes
    shutdownCh      chan struct{}
}
```

**Methods to implement**:

| Method | Description |
|--------|-------------|
| `Run()` | Main loop, runs aggregation every interval |
| `Stop()` | Graceful shutdown |
| `aggregateAll()` | Orchestrates all aggregation steps |
| `aggregateLatency(vendorID)` | Calculate avg/p95/p99 from gateway_telemetry |
| `aggregateSuccessRate(vendorID)` | Calculate success % from gateway_telemetry |
| `aggregateUptime(vendorID)` | Calculate uptime % from vendor_health_checks |
| `aggregateRevenue(vendorID)` | Sum revenue from transactions |
| `aggregateUniqueAgents(vendorID)` | Count distinct agents from transactions |
| `calculateRating(vendorID)` | Base 5.0 minus penalties for disputes/slashes |

**Aggregation SQL queries**:

1. **Latency aggregation** (from gateway_telemetry, last 24h):
   - AVG(response_time_ms) → avg_latency_ms
   - PERCENTILE_CONT(0.95) → p95_latency_ms
   - PERCENTILE_CONT(0.99) → p99_latency_ms

2. **Success rate** (from gateway_telemetry, last 24h):
   - COUNT(*) WHERE status = 'success' / COUNT(*) * 100

3. **Uptime** (from vendor_health_checks, last 7 days):
   - COUNT(*) WHERE status = 'up' / COUNT(*) * 100

4. **Revenue** (from transactions, all time):
   - SUM(amount) WHERE vendor_id matches

5. **Rating calculation**:
   - Start: 5.0
   - Per unresolved dispute: -0.1
   - Per slash event: -0.5
   - Minimum: 1.0

---

### 2.6 New Worker: HealthCheckWorker

**New file to create**: `machpay-backend/internal/worker/health_checker.go`

**Worker struct**:

```
HealthCheckWorker struct {
    db              *sql.DB
    vendorRepo      VendorRepo
    interval        time.Duration  // default: 1 minute
    timeout         time.Duration  // default: 10 seconds
    concurrency     int            // default: 10
    shutdownCh      chan struct{}
    httpClient      *http.Client
}
```

**Methods to implement**:

| Method | Description |
|--------|-------------|
| `Run()` | Main loop, runs checks every interval |
| `Stop()` | Graceful shutdown |
| `checkAllVendors()` | Iterate all active vendors, dispatch checks |
| `checkVendor(vendor)` | Make HTTP request to health endpoint |
| `recordResult(vendorID, status, latency, statusCode, error)` | Insert to vendor_health_checks |

**Health check logic**:

1. Get all vendors WHERE status = 'active' AND health_endpoint IS NOT NULL
2. For each vendor (with concurrency limit of 10):
   - Build URL: `vendor.base_url + vendor.health_endpoint`
   - Make HTTP GET request with 10s timeout
   - Determine status:
     - 2xx response → 'up'
     - 5xx response → 'down'
     - 4xx response → 'degraded'
     - Timeout/error → 'down'
   - Record response_time_ms if successful
   - Insert result to vendor_health_checks

**SSRF Protection** (reuse existing from vendor_handler.go):
- Block private IPs (10.x, 172.16.x, 192.168.x, 127.x)
- Block link-local (169.254.x - AWS metadata!)
- Block localhost

---

### 2.7 Route Registration

**File to modify**: `machpay-backend/cmd/api/routes.go`

**Changes to make**:

1. Create DiscoveryHandler instance with dependencies
2. Create route group: `discovery := v1.Group("/discovery")`
3. Register routes:
   - `discovery.GET("/vendors", discoveryHandler.HandleSearchVendors)`
   - `discovery.GET("/vendors/featured", discoveryHandler.HandleGetFeatured)`
   - `discovery.GET("/vendors/:id", discoveryHandler.HandleGetVendorDetail)`
   - `discovery.GET("/categories", discoveryHandler.HandleGetCategories)`
   - `discovery.GET("/stats", discoveryHandler.HandleGetStats)`
4. No authentication required (public API)
5. Apply rate limiting: 100 requests/minute per IP

---

### 2.8 Worker Registration

**File to modify**: `machpay-backend/cmd/worker/main.go`

**Changes to make**:

1. Initialize MetricsAggregatorWorker with 5-minute interval
2. Initialize HealthCheckWorker with 1-minute interval
3. Start both workers in goroutines
4. Add to graceful shutdown handler

---

### 2.9 Phase 2 Acceptance Criteria

- [ ] All discovery endpoints return correct data
- [ ] Search with query returns relevant results
- [ ] Category filter works correctly
- [ ] All sort options work correctly
- [ ] Pagination works (limit, offset, hasMore)
- [ ] MetricsAggregator runs every 5 minutes
- [ ] HealthChecker runs every 1 minute
- [ ] Health checks respect SSRF protections
- [ ] Metrics are updated in vendor_metrics table
- [ ] Health checks are recorded in vendor_health_checks table
- [ ] Rate limiting applied to discovery endpoints

---

## Phase 3: Console UI

**Duration**: 3 days  
**Dependencies**: Phase 2 complete  
**Owner**: Frontend Engineer

### 3.1 New Route Registration

**File to modify**: `machpay-console/src/App.jsx`

**Changes to make**:

1. Import Marketplace page: `import { Marketplace } from './pages/Marketplace'`
2. Add route inside AuthenticatedRoutes:
   ```jsx
   <Route path="/marketplace" element={<RoleLayout />}>
     <Route index element={<Marketplace />} />
   </Route>
   ```

---

### 3.2 Sidebar Navigation

**File to modify**: `machpay-console/src/components/Layout.jsx`

**Changes to make**:

1. Import Store icon: `import { Store } from 'lucide-react'`
2. Add to `baseNavItems` array (after Explorer):
   ```javascript
   { 
     id: 'marketplace', 
     label: 'Marketplace', 
     icon: Store, 
     path: '/marketplace', 
     roles: ['observer', 'architect', 'operator'] 
   }
   ```

---

### 3.3 API Client Functions

**File to modify**: `machpay-console/src/api.js`

**Changes to make**:

Add `discovery` namespace with functions:

| Function | HTTP Call | Description |
|----------|-----------|-------------|
| `discovery.searchVendors(params)` | GET /v1/discovery/vendors | Search with filters |
| `discovery.getFeatured(limit)` | GET /v1/discovery/vendors/featured | Featured vendors |
| `discovery.getVendorDetail(id)` | GET /v1/discovery/vendors/:id | Single vendor |
| `discovery.getCategories()` | GET /v1/discovery/categories | Categories list |
| `discovery.getStats()` | GET /v1/discovery/stats | Marketplace stats |

**Parameter mapping for searchVendors**:

```javascript
{
  q: params.query,
  category: params.category,
  sort: params.sort,
  min_uptime: params.minUptime,
  min_rating: params.minRating,
  limit: params.limit,
  offset: params.offset
}
```

---

### 3.4 New Page: Marketplace

**New file to create**: `machpay-console/src/pages/Marketplace.jsx`

**Component hierarchy**:

```
Marketplace
├── MarketplaceHeader
│   └── Stats: total vendors, total requests, avg latency
├── SearchSection
│   ├── SearchInput (debounced)
│   ├── SortDropdown
│   └── CategoryPills
├── VendorGrid
│   └── VendorCard (repeated)
├── Pagination
│   └── LoadMoreButton or InfiniteScroll
└── VendorDetailModal (conditional)
```

**State to manage**:

| State | Type | Default | Description |
|-------|------|---------|-------------|
| `searchQuery` | string | "" | Debounced search input |
| `selectedCategory` | string | "all" | Current category filter |
| `sortBy` | string | "rating" | Current sort option |
| `vendors` | array | [] | Search results |
| `total` | number | 0 | Total matching vendors |
| `loading` | boolean | true | Loading state |
| `error` | Error | null | Error state |
| `hasMore` | boolean | false | More results available |
| `offset` | number | 0 | Current pagination offset |
| `selectedVendor` | object | null | Vendor for detail modal |
| `categories` | array | [] | Available categories |
| `stats` | object | null | Marketplace statistics |

**Effects to implement**:

1. On mount: Fetch categories, stats, initial vendors
2. On searchQuery change: Debounce 300ms, then fetch vendors
3. On category change: Reset offset, fetch vendors
4. On sort change: Reset offset, fetch vendors
5. On load more: Increment offset, append vendors

**URL sync** (optional but recommended):
- Sync searchQuery, category, sort to URL query params
- Enable shareable search URLs

---

### 3.5 New Component: VendorCard

**New file to create**: `machpay-console/src/components/marketplace/VendorCard.jsx`

**Props**:

| Prop | Type | Description |
|------|------|-------------|
| `vendor` | object | Vendor data from API |
| `onClick` | function | Click handler |

**Visual elements to include**:

1. **Logo/Icon**: Vendor logo or category fallback icon
2. **Header row**: Name + verified badge (if verified)
3. **Category badge**: Colored badge with category name
4. **Description**: 2-line truncated description
5. **Metrics row**:
   - Price: `$0.0001/req` in green
   - Uptime: `99.98%` (green >99.5, yellow >99, red <99)
   - Rating: `★ 4.9` in yellow
   - Latency: Sparkline + `42ms`
6. **Hover state**: Border glow, arrow indicator

**Styling**:
- Use existing Panel component style
- Match theme colors from tailwind.config.js
- Add motion animation on mount (staggered)

---

### 3.6 New Component: VendorDetailModal

**New file to create**: `machpay-console/src/components/marketplace/VendorDetailModal.jsx`

**Props**:

| Prop | Type | Description |
|------|------|-------------|
| `vendor` | object | Vendor data |
| `isOpen` | boolean | Modal visibility |
| `onClose` | function | Close handler |

**Sections to include**:

1. **Header**:
   - Logo (large)
   - Name + verified badge
   - Category badge
   - Website link (if available)

2. **Description**:
   - Full description text
   - Tags list

3. **Metrics Grid** (2x3 grid):
   - Uptime %
   - Avg Latency
   - P95 Latency
   - Rating
   - Total Requests
   - Success Rate

4. **Pricing**:
   - Price per request
   - Pricing model
   - Rate limit (if any)

5. **Endpoints** (if available):
   - Table of endpoints with path, method, price
   - Expandable for more details

6. **Integration**:
   - Gateway URL (copyable)
   - Code snippet (Python, cURL)
   - "View Docs" button

7. **Actions**:
   - Copy Gateway URL button
   - View Documentation button (if docs URL available)

---

### 3.7 New Component: CategoryPills

**New file to create**: `machpay-console/src/components/marketplace/CategoryPills.jsx`

**Props**:

| Prop | Type | Description |
|------|------|-------------|
| `categories` | array | [{id, name, count}] |
| `selected` | string | Currently selected ID |
| `onSelect` | function | Selection handler |

**Features**:
- "All" option always first
- Horizontal scrollable on mobile
- Count badge for each category
- Active state styling (emerald background)

**Category icons** (emoji or lucide):
- AI/ML: 🤖 or Cpu
- Weather: 🌤️ or Cloud
- Data: 📊 or Database
- Finance: 💹 or DollarSign
- Geo: 🌍 or MapPin
- Security: 🔐 or Shield

---

### 3.8 New Component: SortDropdown

**New file to create**: `machpay-console/src/components/marketplace/SortDropdown.jsx`

**Props**:

| Prop | Type | Description |
|------|------|-------------|
| `value` | string | Current sort option |
| `onChange` | function | Change handler |

**Sort options**:

| Value | Label |
|-------|-------|
| `rating` | ⭐ Rating (High → Low) |
| `uptime` | 📈 Uptime (High → Low) |
| `price_asc` | 💰 Price (Low → High) |
| `price_desc` | 💰 Price (High → Low) |
| `popular` | 🔥 Most Popular |
| `fastest` | ⚡ Fastest |

**Styling**:
- Match existing select inputs in console
- Monospace font for values

---

### 3.9 Component Exports

**New file to create**: `machpay-console/src/components/marketplace/index.js`

**Exports**:

```javascript
export { VendorCard } from './VendorCard';
export { VendorDetailModal } from './VendorDetailModal';
export { CategoryPills } from './CategoryPills';
export { SortDropdown } from './SortDropdown';
```

---

### 3.10 Update ServiceExplorer (Optional)

**File to modify**: `machpay-console/src/components/ServiceExplorer.jsx`

**Changes to make** (if keeping in Explorer page):

1. Update API call to use `api.discovery.searchVendors()`
2. Map response to expected format
3. Add uptime, rating columns to table
4. Keep simpler table layout (vs card layout in Marketplace)

---

### 3.11 Phase 3 Acceptance Criteria

- [ ] /marketplace route accessible from sidebar
- [ ] Search input works with debouncing
- [ ] Category filters work correctly
- [ ] All sort options work correctly
- [ ] Vendor cards display all metrics
- [ ] Vendor detail modal opens on card click
- [ ] Load more pagination works
- [ ] Empty state shown when no results
- [ ] Loading skeletons during fetch
- [ ] Error state with retry button
- [ ] Mobile responsive layout
- [ ] Matches console theme exactly

---

## Phase 4: SDK Integration

**Duration**: 1 day  
**Dependencies**: Phase 2 complete  
**Owner**: Backend Engineer

### 4.1 New Module: discovery

**New file to create**: `machpay-py/src/machpay/discovery.py`

**Dataclasses to define**:

```python
@dataclass
class VendorMetricsInfo:
    uptime: float
    avg_latency_ms: int
    rating: float
    total_requests: int
    success_rate: float

@dataclass
class VendorPricingInfo:
    model: str
    price_usd: float
    currency: str

@dataclass
class VendorInfo:
    id: int
    app_id: str
    slug: str
    name: str
    description: str
    category: str
    logo_url: Optional[str]
    is_verified: bool
    pricing: VendorPricingInfo
    metrics: VendorMetricsInfo
    base_url: str

@dataclass
class CategoryInfo:
    id: str
    name: str
    count: int

@dataclass
class SearchResult:
    vendors: List[VendorInfo]
    total: int
    limit: int
    offset: int
    has_more: bool

@dataclass
class MarketplaceStats:
    total_vendors: int
    total_requests: int
    avg_latency_ms: int
```

**DiscoveryClient class**:

```python
class DiscoveryClient:
    def __init__(self, base_url: str, api_key: Optional[str] = None):
        ...
    
    def search(
        self,
        query: str = "",
        category: str = "",
        sort: str = "rating",
        min_uptime: float = 0,
        min_rating: float = 0,
        limit: int = 20,
        offset: int = 0
    ) -> SearchResult:
        """Search for vendors with filters."""
        ...
    
    def get_featured(self, limit: int = 10) -> List[VendorInfo]:
        """Get featured vendors."""
        ...
    
    def get_vendor(self, vendor_id: int) -> VendorInfo:
        """Get a single vendor by ID."""
        ...
    
    def get_categories(self) -> List[CategoryInfo]:
        """Get all categories with counts."""
        ...
    
    def get_stats(self) -> MarketplaceStats:
        """Get marketplace statistics."""
        ...
```

---

### 4.2 Update Client

**File to modify**: `machpay-py/src/machpay/client.py`

**Changes to make**:

1. Import DiscoveryClient: `from .discovery import DiscoveryClient`
2. Add lazy property:
   ```python
   @property
   def discovery(self) -> DiscoveryClient:
       if self._discovery is None:
           self._discovery = DiscoveryClient(self._base_url, self._api_key)
       return self._discovery
   ```
3. Initialize `self._discovery = None` in `__init__`

---

### 4.3 Update Package Exports

**File to modify**: `machpay-py/src/machpay/__init__.py`

**Changes to make**:

Add exports:
```python
from .discovery import (
    DiscoveryClient,
    VendorInfo,
    VendorMetricsInfo,
    VendorPricingInfo,
    SearchResult,
    CategoryInfo,
    MarketplaceStats,
)
```

---

### 4.4 Usage Example

**For documentation**:

```python
from machpay import MachPayClient

client = MachPayClient(api_key="mp_sk_...")

# Search for weather APIs
results = client.discovery.search(
    query="weather",
    category="weather",
    min_uptime=99.0,
    sort="rating"
)

for vendor in results.vendors:
    print(f"{vendor.name}: ${vendor.pricing.price_usd}/req")
    print(f"  Uptime: {vendor.metrics.uptime}%")
    print(f"  Rating: {vendor.metrics.rating}/5")
    print(f"  Latency: {vendor.metrics.avg_latency_ms}ms")

# Get featured vendors
featured = client.discovery.get_featured(limit=5)

# Get categories
categories = client.discovery.get_categories()
for cat in categories:
    print(f"{cat.name}: {cat.count} vendors")
```

---

### 4.5 Phase 4 Acceptance Criteria

- [ ] DiscoveryClient class implemented
- [ ] All dataclasses defined with proper types
- [ ] client.discovery property works
- [ ] search() method works with all parameters
- [ ] get_featured() method works
- [ ] get_vendor() method works
- [ ] get_categories() method works
- [ ] get_stats() method works
- [ ] Package exports updated
- [ ] Type hints complete

---

## Phase 5: Polish & Launch

**Duration**: 2 days  
**Dependencies**: Phases 1-4 complete  
**Owner**: Full Team

### 5.1 Backend Testing

**New files to create**:

| File | Tests |
|------|-------|
| `internal/handlers/sql/discovery_handler_test.go` | Search, filters, pagination, empty results |
| `internal/repository/vendor_metrics_repo_test.go` | CRUD operations |
| `internal/repository/vendor_repo_search_test.go` | Search query, full-text search |
| `internal/worker/metrics_aggregator_test.go` | Aggregation logic |
| `internal/worker/health_checker_test.go` | Health check logic, SSRF protection |

**Test scenarios**:
- Search with empty query returns all
- Search with query returns relevant results
- Category filter returns correct vendors
- Sort options order correctly
- Pagination limit/offset works
- Empty results return empty array
- Invalid sort option defaults to rating
- Health check blocks private IPs

---

### 5.2 Console Testing

**Test files to create**:

| File | Tests |
|------|-------|
| `src/pages/Marketplace.test.jsx` | Search, filters, loading, error states |
| `src/components/marketplace/VendorCard.test.jsx` | Rendering, click handler |
| `src/components/marketplace/VendorDetailModal.test.jsx` | Open/close, data display |

**Test scenarios**:
- Search input debounces correctly
- Category pills toggle correctly
- Sort dropdown changes sort
- Loading skeleton shows during fetch
- Error state shows on API error
- Empty state shows when no results
- Vendor card renders all data
- Modal opens on card click

---

### 5.3 SDK Testing

**Test file to create**: `tests/test_discovery.py`

**Test scenarios**:
- Search returns SearchResult
- Filters apply correctly
- Empty results handled
- API errors raise exceptions
- Type hints validated

---

### 5.4 Documentation

**New files to create**:

| File | Content |
|------|---------|
| `machpay-docs/api/discovery.md` | API reference for all discovery endpoints |
| `machpay-docs/console/marketplace.md` | User guide for marketplace UI |
| `machpay-docs/sdk/discovery.md` | SDK discovery module documentation |

**Update existing files**:

| File | Changes |
|------|---------|
| `machpay-py/README.md` | Add discovery examples |
| `machpay-docs/README.md` | Add marketplace to feature list |

---

### 5.5 Performance Optimization

**Backend optimizations**:

1. **Redis caching**:
   - Cache search results: key=`discovery:search:{hash}`, TTL=60s
   - Cache featured vendors: key=`discovery:featured`, TTL=300s
   - Cache categories: key=`discovery:categories`, TTL=300s
   - Cache stats: key=`discovery:stats`, TTL=60s

2. **Database optimizations**:
   - Run EXPLAIN ANALYZE on search query
   - Add any missing indexes
   - Consider materialized view for hot searches

3. **Connection pooling**:
   - Verify pool size adequate for worker load

**Console optimizations**:

1. **React.memo**: Wrap VendorCard to prevent re-renders
2. **Virtualization**: Use react-window for large result sets (>50 vendors)
3. **Image lazy loading**: Lazy load vendor logos
4. **Skeleton loaders**: Show during initial load and search

---

### 5.6 Monitoring & Alerting

**Prometheus metrics to add**:

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `discovery_search_duration_seconds` | Histogram | - | Search endpoint latency |
| `discovery_search_results_total` | Histogram | - | Result count distribution |
| `health_check_duration_seconds` | Histogram | vendor_id | Health check latency |
| `health_check_status_total` | Counter | vendor_id, status | Health check results |
| `metrics_aggregation_duration_seconds` | Histogram | - | Aggregator run time |

**Alerts to configure**:

| Alert | Condition | Severity |
|-------|-----------|----------|
| HealthCheckerDown | Worker not running > 5m | Critical |
| MetricsAggregatorDown | Worker not running > 10m | Warning |
| DiscoveryHighLatency | p95 latency > 500ms | Warning |
| DiscoveryErrorRate | Error rate > 1% | Warning |

---

### 5.7 Launch Checklist

**Pre-launch**:
- [ ] All tests passing
- [ ] Migration tested on staging
- [ ] Workers running and producing data
- [ ] Search returning results
- [ ] Console UI working on staging
- [ ] Documentation complete
- [ ] Monitoring dashboards created
- [ ] Alerts configured

**Launch day**:
- [ ] Run migration on production
- [ ] Deploy backend with new workers
- [ ] Verify workers started
- [ ] Deploy console with marketplace
- [ ] Verify search works
- [ ] Monitor error rates
- [ ] Monitor latencies

**Post-launch**:
- [ ] Monitor for 24 hours
- [ ] Check vendor_metrics populated
- [ ] Check health_checks recording
- [ ] Gather user feedback
- [ ] Fix any bugs discovered

---

## File Change Summary

### Backend (`machpay-backend`) — 12 files

| File | Action | Phase |
|------|--------|-------|
| `internal/db/sql/schema.sql` | MODIFY | 1 |
| `internal/db/migrations/007_marketplace_foundation.sql` | CREATE | 1 |
| `internal/models/sql/vendor.go` | MODIFY | 2 |
| `internal/repository/vendor_metrics_repo.go` | CREATE | 2 |
| `internal/repository/vendor_repo.go` | MODIFY | 2 |
| `internal/handlers/sql/discovery_handler.go` | CREATE | 2 |
| `internal/handlers/sql/discovery_handler_test.go` | CREATE | 5 |
| `internal/worker/metrics_aggregator.go` | CREATE | 2 |
| `internal/worker/health_checker.go` | CREATE | 2 |
| `cmd/api/routes.go` | MODIFY | 2 |
| `cmd/worker/main.go` | MODIFY | 2 |

### Console (`machpay-console`) — 10 files

| File | Action | Phase |
|------|--------|-------|
| `src/App.jsx` | MODIFY | 3 |
| `src/components/Layout.jsx` | MODIFY | 3 |
| `src/api.js` | MODIFY | 3 |
| `src/pages/Marketplace.jsx` | CREATE | 3 |
| `src/components/marketplace/VendorCard.jsx` | CREATE | 3 |
| `src/components/marketplace/VendorDetailModal.jsx` | CREATE | 3 |
| `src/components/marketplace/CategoryPills.jsx` | CREATE | 3 |
| `src/components/marketplace/SortDropdown.jsx` | CREATE | 3 |
| `src/components/marketplace/index.js` | CREATE | 3 |
| `src/components/ServiceExplorer.jsx` | MODIFY | 3 |

### SDK (`machpay-py`) — 3 files

| File | Action | Phase |
|------|--------|-------|
| `src/machpay/discovery.py` | CREATE | 4 |
| `src/machpay/client.py` | MODIFY | 4 |
| `src/machpay/__init__.py` | MODIFY | 4 |

### Docs (`machpay-docs`) — 3 files

| File | Action | Phase |
|------|--------|-------|
| `api/discovery.md` | CREATE | 5 |
| `console/marketplace.md` | CREATE | 5 |
| `sdk/discovery.md` | CREATE | 5 |

---

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Health checks overwhelm vendors | High | Medium | Rate limit to 1/min, respect Retry-After |
| Search query slow on large dataset | Medium | Medium | Full-text search index, caching |
| Worker crashes cause stale metrics | Medium | Low | Monitoring, auto-restart, stale data indicator |
| Migration fails on production | High | Low | Test on staging, prepare rollback |

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Search latency p95 | < 200ms | Prometheus histogram |
| Search result relevance | > 80% satisfaction | User feedback |
| Marketplace page load | < 1.5s | Lighthouse |
| Worker uptime | > 99.9% | Prometheus uptime |
| Vendor coverage (metrics) | 100% | vendor_metrics row count |

---

## Appendix: API Reference

### GET /v1/discovery/vendors

Search for vendors with filters.

**Query Parameters**:

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| q | string | No | "" | Search query |
| category | string | No | "" | Category filter |
| sort | string | No | "rating" | Sort option |
| min_uptime | float | No | 0 | Minimum uptime % |
| min_rating | float | No | 0 | Minimum rating |
| limit | int | No | 20 | Results per page (max 100) |
| offset | int | No | 0 | Pagination offset |

**Response**:

```json
{
  "vendors": [...],
  "total": 142,
  "limit": 20,
  "offset": 0,
  "hasMore": true
}
```

### GET /v1/discovery/vendors/featured

Get featured vendors.

**Query Parameters**:

| Param | Type | Required | Default |
|-------|------|----------|---------|
| limit | int | No | 10 |

### GET /v1/discovery/vendors/:id

Get single vendor with full details.

### GET /v1/discovery/categories

Get all categories with vendor counts.

**Response**:

```json
{
  "categories": [
    { "id": "ai", "name": "AI & ML", "count": 45 },
    { "id": "weather", "name": "Weather", "count": 12 }
  ]
}
```

### GET /v1/discovery/stats

Get marketplace statistics.

**Response**:

```json
{
  "totalVendors": 142,
  "totalRequests": 2400000,
  "avgLatencyMs": 47,
  "topCategory": "ai"
}
```

