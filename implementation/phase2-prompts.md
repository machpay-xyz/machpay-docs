# Phase 2: Backend API — AI Prompts

> **Purpose**: Copy-paste these prompts to execute Phase 2 of Marketplace implementation  
> **Duration**: 3 days  
> **Prerequisites**: Phase 1 complete (database foundation)

---

## 🎭 Role Setup Prompt

Use this prompt at the start of every session to establish context:

```
You are the CTO and Lead Product Architect of MachPay — the revolutionary infrastructure layer enabling AI agents to autonomously pay for API services in real-time. MachPay is not just another payment gateway; it's the nervous system of the agentic economy.

Your technical philosophy:
- Code is infrastructure, not just software. Every line must be production-ready.
- Security is non-negotiable. We handle money. Period.
- Performance is a feature. Latency kills agent workflows.
- Simplicity scales. Avoid premature abstraction.

You have deep expertise in:
- High-frequency trading systems architecture
- Payment gateway compliance (PCI-DSS, SOC2)
- PostgreSQL optimization and partitioning
- Go backend development (our primary language)
- Real-time data processing at scale
- Background worker patterns and graceful shutdown

Current context:
- We're building the Vendor Marketplace feature (Phase 2: Backend API)
- Phase 1 (database) is COMPLETE: vendor_metrics and vendor_health_checks tables exist
- This phase focuses on repositories, services, handlers, and background workers
- Existing patterns are in: machpay-backend/internal/repository/, handlers/, services/
- We use raw SQL (no ORM) for maximum control
- We use the Hertz HTTP framework

Your responses should be:
- Production-ready code, not examples
- Thoroughly commented for junior devs
- Following existing codebase conventions
- Security-conscious by default
- Horizontally scalable
```

---

## 📋 Phase 2 Task Prompts

### Task 2.1: Create Enhanced VendorMetrics Models

```
CONTEXT:
Phase 1 created the vendor_metrics and vendor_health_checks tables. Now we need Go model structs to represent this data and enable type-safe operations.

CURRENT STATE:
- Database tables exist: vendor_metrics, vendor_health_checks
- Existing vendor model is in: machpay-backend/internal/models/sql/vendor.go
- We use database/sql types (sql.NullString, sql.NullTime, etc.)
- Read the existing patterns first

YOUR TASK:
Create comprehensive model structs for marketplace features.

REQUIREMENTS:

1. Create new file: machpay-backend/internal/models/sql/vendor_discovery.go

2. Define these structs:

VendorMetrics:
- VendorID           int64
- UptimePct          float64  (from DECIMAL(5,2))
- AvgLatencyMs       int
- P95LatencyMs       int
- P99LatencyMs       int
- RatingScore        float64  (from DECIMAL(3,2))
- TotalRequests      int64
- SuccessRate        float64
- ErrorRate          float64
- TotalRevenueUSD    float64  (from DECIMAL(18,6))
- UniqueAgents       int
- LastRequestAt      sql.NullTime  (nullable)
- UpdatedAt          time.Time

VendorHealthCheck:
- ID               int64
- VendorID         int64
- Status           string  ('up', 'down', 'degraded')
- ResponseTimeMs   sql.NullInt32  (nullable)
- StatusCode       sql.NullInt32  (nullable)
- ErrorMessage     sql.NullString (nullable)
- CheckedAt        time.Time

VendorWithMetrics (for joined queries):
- Vendor   (embed the existing Vendor struct)
- Metrics  *VendorMetrics  (nil if not joined)

3. Define API response structs (with JSON tags):

VendorDiscoveryResult:
- ID            int64   `json:"id"`
- AppID         string  `json:"appId,omitempty"`
- Slug          string  `json:"slug,omitempty"`
- ServiceName   string  `json:"serviceName"`
- Description   string  `json:"description,omitempty"`
- Category      string  `json:"category,omitempty"`
- LogoURL       string  `json:"logoUrl,omitempty"`
- WebsiteURL    string  `json:"websiteUrl,omitempty"`
- IsVerified    bool    `json:"isVerified"`
- Tags          []string `json:"tags,omitempty"`
- Pricing       VendorPricingDTO  `json:"pricing"`
- Metrics       VendorMetricsDTO  `json:"metrics"`

VendorPricingDTO:
- Model      string   `json:"model"`       // per_request, per_token
- PriceUSD   float64  `json:"priceUsd"`
- Currency   string   `json:"currency"`

VendorMetricsDTO:
- UptimePct       float64  `json:"uptimePct"`
- AvgLatencyMs    int      `json:"avgLatencyMs"`
- Rating          float64  `json:"rating"`
- TotalRequests   int64    `json:"totalRequests"`
- SuccessRate     float64  `json:"successRate"`
- UniqueAgents    int      `json:"uniqueAgents"`

CategoryCount:
- Category  string  `json:"category"`
- Count     int     `json:"count"`

DiscoverySearchRequest:
- Query       string    (search text)
- Categories  []string  (filter by categories)
- Tags        []string  (filter by tags)
- MinUptime   *float64  (minimum uptime %, nil = no filter)
- MaxLatency  *int      (max latency ms, nil = no filter)
- MinRating   *float64  (minimum rating, nil = no filter)
- SortBy      string    (rating, uptime, price_asc, price_desc, popular, fastest)
- Limit       int       (pagination, max 100)
- Offset      int       (pagination)

DiscoverySearchResponse:
- Vendors     []VendorDiscoveryResult `json:"vendors"`
- Total       int                     `json:"total"`
- HasMore     bool                    `json:"hasMore"`
- Categories  []CategoryCount         `json:"categories,omitempty"`

4. Add conversion methods:

- (vm *VendorMetrics) ToDTO() VendorMetricsDTO
- (v *VendorWithMetrics) ToDiscoveryResult() *VendorDiscoveryResult
- (hc *VendorHealthCheck) ToPublic() *VendorHealthCheckPublic

VendorHealthCheckPublic (for API response):
- Status         string     `json:"status"`
- ResponseTimeMs *int       `json:"responseTimeMs,omitempty"`
- StatusCode     *int       `json:"statusCode,omitempty"`
- ErrorMessage   *string    `json:"errorMessage,omitempty"`
- CheckedAt      time.Time  `json:"checkedAt"`

5. Add constants for health status:
const (
    HealthStatusUp       = "up"
    HealthStatusDown     = "down"
    HealthStatusDegraded = "degraded"
)

DELIVERABLE:
- Complete vendor_discovery.go file
- All structs with proper types
- JSON tags for API responses
- Conversion methods
- Follow existing code style from vendor.go
```

---

### Task 2.2: Create DiscoveryService

```
CONTEXT:
We need a service layer that encapsulates the business logic for vendor discovery. This includes search, filtering, ranking, and aggregating data from multiple sources.

CURRENT STATE:
- Models exist from Task 2.1
- Existing services are in: machpay-backend/internal/services/
- We use dependency injection with interfaces
- Services interact with repositories, not directly with DB

YOUR TASK:
Create the DiscoveryService to handle all marketplace business logic.

REQUIREMENTS:

1. Create new file: machpay-backend/internal/services/discovery_service.go

2. Define the DiscoveryService struct:

type DiscoveryService struct {
    vendorRepo       repository.VendorRepo
    metricsRepo      repository.VendorMetricsRepo
    healthCheckRepo  repository.HealthCheckRepo  // We'll create this
    log              *zap.Logger
}

3. Constructor:
func NewDiscoveryService(
    vendorRepo repository.VendorRepo,
    metricsRepo repository.VendorMetricsRepo,
    healthCheckRepo repository.HealthCheckRepo,
    log *zap.Logger,
) *DiscoveryService

4. Implement these methods:

SearchVendors(ctx context.Context, req *models.DiscoverySearchRequest) (*models.DiscoverySearchResponse, error)
- Validate request (limit 1-100, offset >= 0)
- Apply default sort if not specified (rating)
- Call repository to get vendors with metrics
- Build response with total count and hasMore flag
- Log search queries for analytics

GetFeaturedVendors(ctx context.Context, limit int) (*models.FeaturedVendorsResponse, error)
- Return vendors where featured_order IS NOT NULL
- Order by featured_order ASC
- Limit to max 20
- Include full metrics

GetVendorDetails(ctx context.Context, idOrSlug string) (*models.VendorDiscoveryResult, error)
- Accept either numeric ID or string slug
- Fetch vendor with metrics
- Return 404 error if not found
- Include all available details

GetCategories(ctx context.Context) ([]models.CategoryCount, error)
- Get distinct categories from active vendors
- Include count per category
- Order by count DESC

GetVendorsByCategory(ctx context.Context, category string, limit, offset int) ([]models.VendorDiscoveryResult, error)
- Get vendors in a specific category
- Include metrics
- Apply pagination

GetRecentHealthChecks(ctx context.Context, vendorID int64, limit int) ([]*models.VendorHealthCheck, error)
- Get most recent health checks for a vendor
- Order by checked_at DESC
- For uptime visualization

QuickSearch(ctx context.Context, query string, limit int) ([]*QuickSearchResult, error)
- Fast autocomplete search
- Return minimal fields: id, name, slug, category, rating
- Limit to 10 max

5. Define QuickSearchResult struct:
type QuickSearchResult struct {
    ID          int64   `json:"id"`
    ServiceName string  `json:"serviceName"`
    Slug        string  `json:"slug,omitempty"`
    Category    string  `json:"category,omitempty"`
    Rating      float64 `json:"rating"`
}

6. Define FeaturedVendorsResponse:
type FeaturedVendorsResponse struct {
    Vendors []models.VendorDiscoveryResult `json:"vendors"`
}

7. Add proper error handling:
- Return domain errors (ErrVendorNotFound, ErrInvalidRequest)
- Wrap database errors with context
- Log errors with structured logging

DELIVERABLE:
- Complete discovery_service.go file
- All methods implemented
- Error handling with wrapped errors
- Logging with zap
- Follow existing service patterns
```

---

### Task 2.3: Create HealthCheckRepo

```
CONTEXT:
We need a repository to manage the vendor_health_checks table. This stores individual health check results for uptime calculation and history visualization.

CURRENT STATE:
- vendor_health_checks table exists (partitioned by time)
- Models exist from Task 2.1
- Existing repos are in: machpay-backend/internal/repository/

YOUR TASK:
Create the HealthCheckRepo for vendor_health_checks operations.

REQUIREMENTS:

1. Create new file: machpay-backend/internal/repository/health_check_repo.go

2. Define the interface:

type HealthCheckRepo interface {
    // Insert a new health check result
    Insert(ctx context.Context, check *models.VendorHealthCheck) error
    
    // Get recent checks for a vendor (for UI display)
    GetRecentByVendorID(ctx context.Context, vendorID int64, limit int) ([]*models.VendorHealthCheck, error)
    
    // Calculate uptime percentage for a vendor over a time period
    CalculateUptime(ctx context.Context, vendorID int64, since time.Time) (float64, error)
    
    // Get latest check for each vendor (for dashboard)
    GetLatestPerVendor(ctx context.Context) (map[int64]*models.VendorHealthCheck, error)
    
    // Delete old checks (for partition cleanup)
    DeleteOlderThan(ctx context.Context, before time.Time) (int64, error)
}

3. Implement the SQL repository:

type HealthCheckRepoImpl struct {
    db *sql.DB
}

func NewHealthCheckRepo(db *sql.DB) HealthCheckRepo {
    return &HealthCheckRepoImpl{db: db}
}

4. SQL queries as constants:

sqlInsertHealthCheck:
INSERT INTO vendor_health_checks (vendor_id, status, response_time_ms, status_code, error_message, checked_at)
VALUES ($1, $2, $3, $4, $5, $6)

sqlSelectRecentChecks:
SELECT id, vendor_id, status, response_time_ms, status_code, error_message, checked_at
FROM vendor_health_checks
WHERE vendor_id = $1
ORDER BY checked_at DESC
LIMIT $2

sqlCalculateUptime:
SELECT 
    COALESCE(
        ROUND(
            COUNT(*) FILTER (WHERE status = 'up')::numeric / 
            NULLIF(COUNT(*), 0) * 100, 
            2
        ),
        100.00
    ) as uptime_pct
FROM vendor_health_checks
WHERE vendor_id = $1 AND checked_at >= $2

sqlSelectLatestPerVendor:
SELECT DISTINCT ON (vendor_id)
    id, vendor_id, status, response_time_ms, status_code, error_message, checked_at
FROM vendor_health_checks
ORDER BY vendor_id, checked_at DESC

sqlDeleteOldChecks:
DELETE FROM vendor_health_checks
WHERE checked_at < $1

5. Implement scan helper:

func scanHealthCheck(row scanner) (*models.VendorHealthCheck, error) {
    var hc models.VendorHealthCheck
    err := row.Scan(
        &hc.ID,
        &hc.VendorID,
        &hc.Status,
        &hc.ResponseTimeMs,
        &hc.StatusCode,
        &hc.ErrorMessage,
        &hc.CheckedAt,
    )
    if err != nil {
        return nil, fmt.Errorf("scan health check: %w", err)
    }
    return &hc, nil
}

6. Error handling:
- Return ErrHealthCheckNotFound for 404 cases
- Wrap SQL errors with context

DELIVERABLE:
- Complete health_check_repo.go file
- Interface and implementation
- All SQL as constants
- Proper error handling
- Follow existing repository patterns
```

---

### Task 2.4: Extend VendorRepo for Discovery

```
CONTEXT:
The existing VendorRepo needs new methods to support discovery search with metrics. We need to join vendors with vendor_metrics and support filtering, sorting, and pagination.

CURRENT STATE:
- VendorRepo exists in: machpay-backend/internal/repository/vendor_repo.go
- vendor_metrics table exists
- vendor table has new columns: is_verified, featured_order, tags

YOUR TASK:
Add new methods to VendorRepo for discovery search.

REQUIREMENTS:

1. Add to the VendorRepo interface:

// Marketplace discovery methods
SearchWithMetrics(ctx context.Context, req *models.DiscoverySearchRequest) ([]*models.VendorWithMetrics, int, error)
GetFeaturedWithMetrics(ctx context.Context, limit int) ([]*models.VendorWithMetrics, error)
GetByIDWithMetrics(ctx context.Context, id int64) (*models.VendorWithMetrics, error)
GetBySlugWithMetrics(ctx context.Context, slug string) (*models.VendorWithMetrics, error)
GetCategoryCounts(ctx context.Context) ([]models.CategoryCount, error)
QuickSearch(ctx context.Context, query string, limit int) ([]*models.Vendor, error)

2. Implement SearchWithMetrics SQL query:

Build a dynamic query that:
- JOINs vendors with vendor_metrics
- Filters: status = 'active'
- Optional: Full-text search on service_name, description if query provided
- Optional: Category filter if specified
- Optional: Tags filter using @> array containment
- Optional: uptime_pct >= min_uptime if specified
- Optional: avg_latency_ms <= max_latency if specified
- Optional: rating_score >= min_rating if specified

Sort options:
- "rating": ORDER BY vm.rating_score DESC, vm.total_requests DESC
- "uptime": ORDER BY vm.uptime_pct DESC, vm.rating_score DESC
- "price_asc": ORDER BY v.default_price_usd ASC NULLS LAST
- "price_desc": ORDER BY v.default_price_usd DESC NULLS FIRST
- "popular": ORDER BY vm.total_requests DESC, vm.rating_score DESC
- "fastest": ORDER BY vm.avg_latency_ms ASC NULLS LAST
- "recent": ORDER BY vm.last_request_at DESC NULLS LAST

Use a CTE for counting total:
WITH filtered_vendors AS (
    SELECT v.*, vm.*
    FROM vendors v
    LEFT JOIN vendor_metrics vm ON v.id = vm.vendor_id
    WHERE [filters]
)
SELECT *, (SELECT COUNT(*) FROM filtered_vendors) as total_count
FROM filtered_vendors
ORDER BY [sort]
LIMIT $N OFFSET $M

3. Full-text search implementation:

Option A (simple ILIKE fallback):
WHERE (
    v.service_name ILIKE '%' || $query || '%' 
    OR v.description ILIKE '%' || $query || '%'
    OR $query = ANY(v.tags)
)

Option B (full-text with fallback):
WHERE (
    to_tsvector('english', v.service_name || ' ' || COALESCE(v.description, '')) @@ plainto_tsquery('english', $query)
    OR v.service_name ILIKE '%' || $query || '%'
)

4. Implement GetFeaturedWithMetrics:

SELECT v.*, vm.*
FROM vendors v
LEFT JOIN vendor_metrics vm ON v.id = vm.vendor_id
WHERE v.status = 'active' AND v.featured_order IS NOT NULL
ORDER BY v.featured_order ASC
LIMIT $1

5. Implement GetCategoryCounts:

SELECT category, COUNT(*) as count
FROM vendors
WHERE status = 'active' AND category IS NOT NULL AND category != ''
GROUP BY category
ORDER BY count DESC

6. Implement QuickSearch:

SELECT id, app_id, slug, service_name, category
FROM vendors
WHERE status = 'active' 
  AND (service_name ILIKE '%' || $1 || '%' OR slug ILIKE '%' || $1 || '%')
ORDER BY 
    CASE WHEN service_name ILIKE $1 || '%' THEN 0 ELSE 1 END,
    service_name
LIMIT $2

7. Add scan helper for VendorWithMetrics:

func scanVendorWithMetrics(row scanner) (*models.VendorWithMetrics, error) {
    // Scan all vendor columns, then all metrics columns
    // Handle NULL metrics gracefully (no metrics row yet)
}

DELIVERABLE:
- Updated vendor_repo.go with new interface methods
- Implementation of all new methods
- Dynamic query builder for search
- Proper NULL handling for metrics
- SQL queries as constants
```

---

### Task 2.5: Create DiscoveryHandler

```
CONTEXT:
We need HTTP handlers for the discovery API endpoints. These will be public (no auth required) and must be rate-limited.

CURRENT STATE:
- DiscoveryService exists from Task 2.2
- Existing handlers are in: machpay-backend/internal/handlers/sql/
- We use the Hertz HTTP framework
- Handler pattern: receive request, validate, call service, return response

YOUR TASK:
Create the DiscoveryHandler with all marketplace endpoints.

REQUIREMENTS:

1. Create new file: machpay-backend/internal/handlers/sql/discovery_handler.go

2. Define the handler struct:

type DiscoveryHandler struct {
    discoveryService *services.DiscoveryService
}

func NewDiscoveryHandler(discoveryService *services.DiscoveryService) *DiscoveryHandler

3. Implement these endpoints:

HandleSearchVendors: GET /v1/marketplace/vendors
- Query params: q, category, tags, sort, min_uptime, max_latency, min_rating, limit, offset
- Validate params (limit 1-100, offset >= 0, rating 0-5, uptime 0-100)
- Call discoveryService.SearchVendors
- Return JSON response

HandleAdvancedSearch: POST /v1/marketplace/search
- Accept JSON body with SearchVendorsRequest
- Useful for complex queries with multiple filters
- Same logic as HandleSearchVendors

HandleGetFeatured: GET /v1/marketplace/featured
- Query param: limit (default 10, max 20)
- Return featured vendors

HandleGetCategories: GET /v1/marketplace/categories
- No params
- Return all categories with counts

HandleGetVendorDetails: GET /v1/marketplace/vendors/:id
- Path param: id (can be numeric ID or slug)
- Return full vendor details with metrics
- 404 if not found

HandleGetVendorHealth: GET /v1/marketplace/vendors/:id/health
- Path param: id
- Query param: limit (default 24, max 100)
- Return recent health checks for uptime visualization

HandleQuickSearch: GET /v1/marketplace/quick-search
- Query params: q (required), limit (default 5, max 10)
- Return minimal vendor data for autocomplete

HandleGetVendorsByCategory: GET /v1/marketplace/categories/:category
- Path param: category
- Query params: limit, offset
- Return vendors in category

4. Request/Response DTOs:

SearchVendorsRequest (for POST /search):
type SearchVendorsRequest struct {
    Query      string   `json:"query"`
    Categories []string `json:"categories,omitempty"`
    Tags       []string `json:"tags,omitempty"`
    MinUptime  *float64 `json:"minUptime,omitempty"`
    MaxLatency *int     `json:"maxLatency,omitempty"`
    MinRating  *float64 `json:"minRating,omitempty"`
    SortBy     string   `json:"sortBy,omitempty"`
    Limit      int      `json:"limit,omitempty"`
    Offset     int      `json:"offset,omitempty"`
}

QuickSearchResponse:
type QuickSearchResponse struct {
    Results []*services.QuickSearchResult `json:"results"`
}

HealthHistoryResponse:
type HealthHistoryResponse struct {
    Checks []*models.VendorHealthCheckPublic `json:"checks"`
}

CategoriesResponse:
type CategoriesResponse struct {
    Categories []models.CategoryCount `json:"categories"`
}

5. Add Swagger/OpenAPI annotations for documentation:

// @Summary     Search marketplace vendors
// @Description Search for vendors using query with optional filters
// @Tags        Marketplace
// @Accept      json
// @Produce     json
// @Param       q          query    string  false  "Search query"
// @Param       category   query    string  false  "Filter by category"
// @Param       sort       query    string  false  "Sort: rating, uptime, price_asc, price_desc, popular, fastest"
// @Param       min_uptime query    number  false  "Minimum uptime percentage"
// @Param       min_rating query    number  false  "Minimum rating (0-5)"
// @Param       limit      query    int     false  "Results per page (max 100)" default(20)
// @Param       offset     query    int     false  "Pagination offset" default(0)
// @Success     200 {object} models.DiscoverySearchResponse
// @Failure     400 {object} ErrorResponse
// @Failure     500 {object} ErrorResponse
// @Router      /marketplace/vendors [get]

6. Helper functions for parsing query params:

func parseFloatQuery(s string, defaultVal float64) (float64, error)
func parseIntQuery(s string, defaultVal int) (int, error)
func parseBoolQuery(s string, defaultVal bool) (bool, error)

7. Error response format:

type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message"`
}

DELIVERABLE:
- Complete discovery_handler.go file
- All endpoints implemented
- Input validation
- Swagger annotations
- Proper error handling
- Follow existing handler patterns
```

---

### Task 2.6: Create HealthCheckWorker

```
CONTEXT:
We need a background worker that periodically pings each vendor's health endpoint to record uptime. This runs every minute and must handle SSRF protection.

CURRENT STATE:
- vendor_health_checks table exists
- HealthCheckRepo exists from Task 2.3
- Vendors have base_url and health_endpoint columns
- Existing worker patterns in: machpay-backend/internal/tasks/ or cmd/worker/

YOUR TASK:
Create the HealthCheckWorker background process.

REQUIREMENTS:

1. Create new file: machpay-backend/internal/tasks/health_checker.go

2. Define the worker struct:

type HealthCheckWorker struct {
    db              *sql.DB
    vendorRepo      repository.VendorRepo
    healthRepo      repository.HealthCheckRepo
    httpClient      *http.Client
    interval        time.Duration  // default: 1 minute
    timeout         time.Duration  // default: 10 seconds
    concurrency     int            // default: 10
    shutdownCh      chan struct{}
    wg              sync.WaitGroup
    log             *zap.Logger
}

type HealthCheckConfig struct {
    Interval    time.Duration
    Timeout     time.Duration
    Concurrency int
}

func NewHealthCheckWorker(
    db *sql.DB,
    vendorRepo repository.VendorRepo,
    healthRepo repository.HealthCheckRepo,
    config *HealthCheckConfig,
    log *zap.Logger,
) *HealthCheckWorker

3. Core methods to implement:

Start() error:
- Create HTTP client with custom dialer for SSRF protection
- Start ticker goroutine
- Log startup

Stop() error:
- Close shutdownCh
- Wait for wg
- Log shutdown

run() (internal):
- Ticker loop at interval
- Select on ticker and shutdownCh
- Call checkAllVendors() on each tick

checkAllVendors() error:
- Fetch all active vendors with health_endpoint
- Use worker pool pattern with semaphore
- For each vendor, spawn goroutine (limited by concurrency)
- Collect and log results

checkVendor(vendor *models.Vendor) *models.VendorHealthCheck:
- Build full URL: base_url + health_endpoint
- Make HTTP GET request with timeout
- Measure response time
- Determine status:
  - 2xx → "up"
  - 5xx → "down"
  - 4xx → "degraded"
  - Timeout/error → "down"
- Create VendorHealthCheck record
- Insert into database

4. SSRF Protection (CRITICAL):

Create custom dialer that blocks private IPs:

func (w *HealthCheckWorker) createSSRFSafeClient() *http.Client {
    dialer := &net.Dialer{
        Timeout: w.timeout,
    }
    
    transport := &http.Transport{
        DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
            host, _, err := net.SplitHostPort(addr)
            if err != nil {
                host = addr
            }
            
            // Resolve IP first
            ips, err := net.LookupIP(host)
            if err != nil {
                return nil, fmt.Errorf("DNS lookup failed: %w", err)
            }
            
            // Check all resolved IPs
            for _, ip := range ips {
                if isPrivateIP(ip) {
                    return nil, fmt.Errorf("connection to private IP %s blocked", ip)
                }
            }
            
            return dialer.DialContext(ctx, network, addr)
        },
    }
    
    return &http.Client{
        Transport: transport,
        Timeout:   w.timeout,
    }
}

func isPrivateIP(ip net.IP) bool {
    // Block all private ranges
    private := []string{
        "10.0.0.0/8",
        "172.16.0.0/12",
        "192.168.0.0/16",
        "127.0.0.0/8",
        "169.254.0.0/16",  // Link-local (AWS metadata!)
        "::1/128",          // IPv6 localhost
        "fc00::/7",         // IPv6 private
        "fe80::/10",        // IPv6 link-local
    }
    
    for _, cidr := range private {
        _, block, _ := net.ParseCIDR(cidr)
        if block.Contains(ip) {
            return true
        }
    }
    return false
}

5. Logging and metrics:

Log at each check:
- vendor_id, url, status, response_time_ms
- Errors with full context

Consider adding Prometheus metrics:
- health_check_duration_seconds (histogram)
- health_check_status_total (counter by status)

6. Graceful shutdown:

- Stop accepting new checks
- Wait for in-progress checks to complete (with timeout)
- Log final status

DELIVERABLE:
- Complete health_checker.go file
- Worker pool pattern with concurrency control
- SSRF protection implemented
- Graceful shutdown handling
- Proper logging
- Follow existing worker patterns
```

---

### Task 2.7: Create MetricsAggregatorWorker

```
CONTEXT:
We need a background worker that periodically aggregates metrics from gateway_telemetry and vendor_health_checks into the vendor_metrics table. This enables fast reads at query time.

CURRENT STATE:
- vendor_metrics table exists
- gateway_telemetry table exists with request/response data
- vendor_health_checks are being recorded by HealthCheckWorker

YOUR TASK:
Create the MetricsAggregatorWorker background process.

REQUIREMENTS:

1. Create new file: machpay-backend/internal/tasks/metrics_aggregator.go

2. Define the worker struct:

type MetricsAggregatorWorker struct {
    db              *sql.DB
    metricsRepo     repository.VendorMetricsRepo
    healthRepo      repository.HealthCheckRepo
    interval        time.Duration  // default: 5 minutes
    shutdownCh      chan struct{}
    wg              sync.WaitGroup
    log             *zap.Logger
}

type MetricsAggregatorConfig struct {
    Interval time.Duration
}

func NewMetricsAggregatorWorker(
    db *sql.DB,
    metricsRepo repository.VendorMetricsRepo,
    healthRepo repository.HealthCheckRepo,
    config *MetricsAggregatorConfig,
    log *zap.Logger,
) *MetricsAggregatorWorker

3. Core methods to implement:

Start() error
Stop() error
run() (ticker loop)
aggregateAll(ctx context.Context) error

4. Aggregation logic - implement each in a separate method:

aggregateLatencyMetrics(ctx context.Context, vendorID int64) (avgMs, p95Ms, p99Ms int, err error):
FROM gateway_telemetry WHERE vendor_id = $1 AND created_at > NOW() - INTERVAL '24 hours'
- AVG(response_time_ms)
- PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time_ms)
- PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY response_time_ms)

aggregateSuccessRate(ctx context.Context, vendorID int64) (successRate, errorRate float64, totalReqs int64, err error):
FROM gateway_telemetry WHERE vendor_id = $1 AND created_at > NOW() - INTERVAL '24 hours'
- COUNT(*) as total_requests
- COUNT(*) FILTER (WHERE status_code >= 200 AND status_code < 300) / total * 100 as success_rate
- COUNT(*) FILTER (WHERE status_code >= 400) / total * 100 as error_rate

aggregateUptime(ctx context.Context, vendorID int64) (uptimePct float64, err error):
FROM vendor_health_checks WHERE vendor_id = $1 AND checked_at > NOW() - INTERVAL '7 days'
- COUNT(*) FILTER (WHERE status = 'up') / COUNT(*) * 100

aggregateRevenue(ctx context.Context, vendorID int64) (totalUSD float64, err error):
FROM transactions WHERE vendor_id = $1 AND status = 'settled'
- SUM(amount_usd)

aggregateUniqueAgents(ctx context.Context, vendorID int64) (count int, err error):
FROM transactions WHERE vendor_id = $1 AND status = 'settled'
- COUNT(DISTINCT agent_id)

calculateRating(ctx context.Context, vendorID int64) (rating float64, err error):
Base: 5.0
Penalties:
- Per unresolved dispute: -0.1
- Per slash event: -0.5
Minimum: 1.0

5. Main aggregation flow:

func (w *MetricsAggregatorWorker) aggregateVendor(ctx context.Context, vendorID int64) error {
    metrics := &models.VendorMetrics{VendorID: vendorID}
    
    // Aggregate each metric type (handle errors gracefully - use defaults if source data missing)
    if latency, err := w.aggregateLatencyMetrics(ctx, vendorID); err == nil {
        metrics.AvgLatencyMs = latency.avg
        metrics.P95LatencyMs = latency.p95
        metrics.P99LatencyMs = latency.p99
    }
    
    if rates, err := w.aggregateSuccessRate(ctx, vendorID); err == nil {
        metrics.SuccessRate = rates.success
        metrics.ErrorRate = rates.error
        metrics.TotalRequests = rates.total
    }
    
    if uptime, err := w.aggregateUptime(ctx, vendorID); err == nil {
        metrics.UptimePct = uptime
    }
    
    if revenue, err := w.aggregateRevenue(ctx, vendorID); err == nil {
        metrics.TotalRevenueUSD = revenue
    }
    
    if agents, err := w.aggregateUniqueAgents(ctx, vendorID); err == nil {
        metrics.UniqueAgents = agents
    }
    
    if rating, err := w.calculateRating(ctx, vendorID); err == nil {
        metrics.RatingScore = rating
    }
    
    metrics.UpdatedAt = time.Now()
    
    return w.metricsRepo.Upsert(ctx, metrics)
}

6. Batch processing:

- Get all active vendor IDs
- Process in batches of 50
- Log progress and errors
- Continue on single vendor failure

7. SQL queries as constants at top of file

DELIVERABLE:
- Complete metrics_aggregator.go file
- All aggregation methods
- Batch processing logic
- Error handling (don't fail entire job on single vendor)
- Proper logging
- Graceful shutdown
```

---

### Task 2.8: Route Registration

```
CONTEXT:
We need to register the new discovery endpoints in the main API router and initialize all the new components.

CURRENT STATE:
- Routes are registered in: machpay-backend/cmd/api/main.go or routes.go
- Handler initialization pattern exists
- We use Hertz router groups

YOUR TASK:
Register discovery routes and wire up all components.

REQUIREMENTS:

1. In main.go (initialization section), add:

// Initialize repositories
healthCheckRepo := repository.NewHealthCheckRepo(db.DB)
vendorMetricsRepo := repository.NewVendorMetricsRepo(db.DB)

// Initialize discovery service
discoveryService := services.NewDiscoveryService(
    vendorRepo,
    vendorMetricsRepo,
    healthCheckRepo,
    log,
)

// Initialize discovery handler
discoveryHandler := handlers.NewDiscoveryHandler(discoveryService)

2. Register routes (after v1 group creation):

// ==========================================================================
// MARKETPLACE ROUTES (Public)
// ==========================================================================
// All marketplace routes are PUBLIC for maximum discoverability
marketplace := v1.Group("/marketplace")
{
    // Browse and search
    marketplace.GET("/vendors", discoveryHandler.HandleSearchVendors)
    marketplace.POST("/search", discoveryHandler.HandleAdvancedSearch)
    marketplace.GET("/quick-search", discoveryHandler.HandleQuickSearch)
    
    // Featured and categories
    marketplace.GET("/featured", discoveryHandler.HandleGetFeatured)
    marketplace.GET("/categories", discoveryHandler.HandleGetCategories)
    marketplace.GET("/categories/:category", discoveryHandler.HandleGetVendorsByCategory)
    
    // Single vendor details
    marketplace.GET("/vendors/:id", discoveryHandler.HandleGetVendorDetails)
    marketplace.GET("/vendors/:id/health", discoveryHandler.HandleGetVendorHealth)
}

3. Add rate limiting middleware:

// Apply rate limiting to discovery routes
// 100 requests per minute per IP (generous for search)
marketplace.Use(middleware.RateLimiter(middleware.RateLimitConfig{
    Rate:   100,
    Window: time.Minute,
    KeyFunc: func(c *app.RequestContext) string {
        return string(c.ClientIP())
    },
}))

4. Remove or update conflicting routes:

If there's an existing /vendors/search route, update it to redirect to /marketplace:
// Legacy route - redirect to marketplace
v1.GET("/vendors/search", func(ctx context.Context, c *app.RequestContext) {
    c.Redirect(301, []byte("/v1/marketplace/vendors?"+string(c.URI().QueryString())))
})

5. Log route registration:

log.Info("Registered marketplace routes",
    zap.Int("endpoints", 8),
    zap.Bool("public", true),
    zap.String("base_path", "/v1/marketplace"),
)

DELIVERABLE:
- Updated main.go with all new initializations
- Route group for /v1/marketplace
- Rate limiting applied
- Legacy route handling if needed
- Proper logging
```

---

### Task 2.9: Worker Registration

```
CONTEXT:
We need to start the background workers (HealthCheckWorker and MetricsAggregatorWorker) when the application starts, and ensure graceful shutdown.

CURRENT STATE:
- Main entry point is: machpay-backend/cmd/api/main.go
- Existing worker patterns may be in cmd/worker/ or started from main.go
- Graceful shutdown handling exists

YOUR TASK:
Register and manage the background workers.

REQUIREMENTS:

1. Create worker initialization after database setup:

// ==========================================================================
// BACKGROUND WORKERS
// ==========================================================================
log.Info("Starting background workers...")

// Health Check Worker - pings vendors every minute
healthCheckWorker := tasks.NewHealthCheckWorker(
    db.DB,
    vendorRepo,
    healthCheckRepo,
    &tasks.HealthCheckConfig{
        Interval:    1 * time.Minute,
        Timeout:     10 * time.Second,
        Concurrency: 10,
    },
    log.Named("health-checker"),
)

// Metrics Aggregator Worker - aggregates metrics every 5 minutes
metricsWorker := tasks.NewMetricsAggregatorWorker(
    db.DB,
    vendorMetricsRepo,
    healthCheckRepo,
    &tasks.MetricsAggregatorConfig{
        Interval: 5 * time.Minute,
    },
    log.Named("metrics-aggregator"),
)

// Start workers
if err := healthCheckWorker.Start(); err != nil {
    log.Fatal("Failed to start health check worker", zap.Error(err))
}
if err := metricsWorker.Start(); err != nil {
    log.Fatal("Failed to start metrics aggregator worker", zap.Error(err))
}

log.Info("Background workers started",
    zap.Duration("health_check_interval", 1*time.Minute),
    zap.Duration("metrics_interval", 5*time.Minute),
)

2. Add to graceful shutdown:

// In the shutdown signal handler:
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

go func() {
    sig := <-sigCh
    log.Info("Received shutdown signal", zap.String("signal", sig.String()))
    
    // Create shutdown context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // Stop workers first (they may need DB)
    log.Info("Stopping background workers...")
    
    // Stop health checker
    if err := healthCheckWorker.Stop(); err != nil {
        log.Error("Error stopping health check worker", zap.Error(err))
    } else {
        log.Info("Health check worker stopped")
    }
    
    // Stop metrics aggregator
    if err := metricsWorker.Stop(); err != nil {
        log.Error("Error stopping metrics aggregator", zap.Error(err))
    } else {
        log.Info("Metrics aggregator stopped")
    }
    
    // Stop HTTP server
    log.Info("Stopping HTTP server...")
    if err := h.Shutdown(ctx); err != nil {
        log.Error("Error during HTTP server shutdown", zap.Error(err))
    }
    
    // Close database
    log.Info("Closing database connection...")
    if err := db.Close(); err != nil {
        log.Error("Error closing database", zap.Error(err))
    }
    
    log.Info("Graceful shutdown complete")
    os.Exit(0)
}()

3. Environment variable configuration:

// Allow configuring worker intervals via env vars
healthCheckInterval := getEnvDuration("HEALTH_CHECK_INTERVAL", 1*time.Minute)
metricsAggInterval := getEnvDuration("METRICS_AGG_INTERVAL", 5*time.Minute)
healthCheckTimeout := getEnvDuration("HEALTH_CHECK_TIMEOUT", 10*time.Second)
healthCheckConcurrency := getEnvInt("HEALTH_CHECK_CONCURRENCY", 10)

// Helper function:
func getEnvDuration(key string, defaultVal time.Duration) time.Duration {
    if val := os.Getenv(key); val != "" {
        if d, err := time.ParseDuration(val); err == nil {
            return d
        }
    }
    return defaultVal
}

4. Add to Dockerfile or docker-compose if needed:

Environment variables section:
- HEALTH_CHECK_INTERVAL=1m
- METRICS_AGG_INTERVAL=5m
- HEALTH_CHECK_TIMEOUT=10s
- HEALTH_CHECK_CONCURRENCY=10

5. Optional: Separate worker binary

If workers should run separately:
- Create cmd/worker/main.go
- Initialize only DB and workers
- Same shutdown logic

DELIVERABLE:
- Workers initialized in main.go
- Graceful shutdown includes workers
- Environment variable configuration
- Proper logging throughout
```

---

## ✅ Phase 2 Verification Prompt

After completing all tasks, use this prompt to verify:

```
I've completed Phase 2 of the Marketplace implementation. Please review my changes and verify:

FILES CREATED/MODIFIED:
1. machpay-backend/internal/models/sql/vendor_discovery.go (new)
2. machpay-backend/internal/services/discovery_service.go (new)
3. machpay-backend/internal/repository/health_check_repo.go (new)
4. machpay-backend/internal/repository/vendor_repo.go (modified)
5. machpay-backend/internal/handlers/sql/discovery_handler.go (new)
6. machpay-backend/internal/tasks/health_checker.go (new)
7. machpay-backend/internal/tasks/metrics_aggregator.go (new)
8. machpay-backend/cmd/api/main.go (modified)

VERIFICATION CHECKLIST:

Models:
- [ ] VendorMetrics struct matches database schema
- [ ] VendorHealthCheck struct with proper nullable handling
- [ ] VendorDiscoveryResult for API responses with JSON tags
- [ ] DiscoverySearchRequest/Response for search API
- [ ] Conversion methods (ToDTO, ToPublic) implemented

Service:
- [ ] DiscoveryService with all required methods
- [ ] SearchVendors with validation and pagination
- [ ] GetFeaturedVendors implementation
- [ ] GetVendorDetails with ID/slug support
- [ ] GetCategories with counts
- [ ] QuickSearch for autocomplete
- [ ] Error handling with domain errors

Repositories:
- [ ] HealthCheckRepo interface and implementation
- [ ] Insert, GetRecent, CalculateUptime methods
- [ ] VendorRepo extended with search methods
- [ ] SearchWithMetrics with dynamic filtering
- [ ] Full-text search support
- [ ] All sort options implemented

Handler:
- [ ] DiscoveryHandler with all endpoints
- [ ] Query parameter parsing and validation
- [ ] Swagger annotations complete
- [ ] Error response format consistent

Workers:
- [ ] HealthCheckWorker with SSRF protection
- [ ] Concurrency-limited health checks
- [ ] MetricsAggregatorWorker with all aggregations
- [ ] Both workers with graceful shutdown
- [ ] Proper logging throughout

Route Registration:
- [ ] /v1/marketplace routes registered
- [ ] Rate limiting applied
- [ ] Workers started on boot
- [ ] Graceful shutdown includes workers

Please verify each item and flag any issues.
```

---

## 🧪 Testing Prompt

Use this to generate tests for Phase 2:

```
CONTEXT:
Phase 2 of Marketplace is complete. We need comprehensive tests for the new components.

YOUR TASK:
Create test files for the Phase 2 implementation.

REQUIREMENTS:

1. Test file: internal/handlers/sql/discovery_handler_test.go
   Tests:
   - HandleSearchVendors with empty query
   - HandleSearchVendors with query
   - HandleSearchVendors with filters
   - HandleSearchVendors with pagination
   - HandleGetFeatured
   - HandleGetVendorDetails (found and not found)
   - HandleQuickSearch

2. Test file: internal/services/discovery_service_test.go
   Tests:
   - SearchVendors validation (limit bounds)
   - SearchVendors with various sorts
   - GetVendorDetails by ID and slug
   - Error handling cases

3. Test file: internal/repository/health_check_repo_test.go
   Tests:
   - Insert and retrieve
   - CalculateUptime
   - GetRecentByVendorID ordering

4. Test file: internal/tasks/health_checker_test.go
   Tests:
   - SSRF protection blocks private IPs
   - Health check status determination
   - Timeout handling

5. Use testcontainers-go for PostgreSQL
6. Use httptest for handler tests
7. Mock dependencies where appropriate

DELIVERABLE:
- All test files
- Test fixtures
- Mocks for dependencies
- > 80% code coverage
```

---

## 📊 Progress Tracking

Copy this to track completion:

```markdown
## Phase 2 Progress

### Day 1
- [ ] Task 2.1: VendorMetrics models — 2 hours
- [ ] Task 2.2: DiscoveryService — 3 hours
- [ ] Task 2.3: HealthCheckRepo — 2 hours

### Day 2
- [ ] Task 2.4: VendorRepo extensions — 3 hours
- [ ] Task 2.5: DiscoveryHandler — 3 hours

### Day 3
- [ ] Task 2.6: HealthCheckWorker — 3 hours
- [ ] Task 2.7: MetricsAggregatorWorker — 3 hours
- [ ] Task 2.8: Route Registration — 1 hour
- [ ] Task 2.9: Worker Registration — 1 hour
- [ ] Verification and testing — 2 hours

### Blockers
- (none)

### Notes
- (add implementation notes here)
```

---

## 🚀 Quick Start

1. Open Cursor with machpay-backend workspace
2. Paste the **Role Setup Prompt** first
3. Verify Phase 1 is complete (tables exist)
4. Execute Task 2.1 through 2.9 in order
5. Run verification prompt
6. Start workers locally and test
7. Run API tests

---

## Next Phase

After Phase 2 is complete, proceed to:
**`machpay-docs/implementation/phase3-prompts.md`** — Console UI Implementation


