# Phase 1: Database Foundation — AI Prompts

> **Purpose**: Copy-paste these prompts to execute Phase 1 of Marketplace implementation  
> **Duration**: 2 days  
> **Prerequisites**: Access to machpay-backend repository

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

Current context:
- We're building the Vendor Marketplace feature
- This enables agents to DISCOVER vendors by performance metrics
- Phase 1 focuses on database schema foundations
- The existing schema is in: machpay-backend/internal/db/sql/schema.sql
- We use raw SQL (no ORM) for maximum control

Your responses should be:
- Production-ready code, not examples
- Thoroughly commented for junior devs
- Following existing codebase conventions
- Security-conscious by default
```

---

## 📋 Phase 1 Task Prompts

### Task 1.1: Create vendor_metrics Table

```
CONTEXT:
We're building the Marketplace feature for MachPay. Agents need to discover vendors based on real performance metrics like uptime, latency, and rating.

CURRENT STATE:
- The vendors table exists in machpay-backend/internal/db/sql/schema.sql
- Vendors have basic info but NO performance metrics
- Gateway telemetry data exists but isn't aggregated per vendor

YOUR TASK:
Add the `vendor_metrics` table to our schema. This table stores pre-computed performance metrics for each vendor, updated by a background worker.

REQUIREMENTS:

1. Table must have these columns:
   - vendor_id (PK, FK to vendors)
   - uptime_pct (DECIMAL 5,2) — calculated from health checks, 0-100%
   - avg_latency_ms (INTEGER) — average response time
   - p95_latency_ms (INTEGER) — 95th percentile latency
   - p99_latency_ms (INTEGER) — 99th percentile latency
   - rating_score (DECIMAL 3,2) — 0-5.0 stars
   - total_requests (BIGINT) — lifetime request count
   - success_rate (DECIMAL 5,2) — % of successful requests
   - error_rate (DECIMAL 5,2) — % of failed requests
   - total_revenue_usd (DECIMAL 18,6) — lifetime revenue
   - unique_agents (INTEGER) — distinct agents served
   - last_request_at (TIMESTAMP WITH TIME ZONE) — most recent request
   - updated_at (TIMESTAMP WITH TIME ZONE) — last metrics refresh

2. Default values:
   - uptime_pct: 100.00
   - rating_score: 5.00
   - All counters: 0
   - success_rate: 100.00
   - error_rate: 0.00

3. Indexes needed for:
   - Sorting by rating (DESC)
   - Sorting by uptime (DESC)
   - Sorting by total_requests (DESC)
   - Sorting by last_request_at (DESC NULLS LAST)

4. Add this to the existing schema.sql file in the appropriate section

DELIVERABLE:
- Show me the exact SQL to add to schema.sql
- Include section header comments matching existing style
- Include all CREATE TABLE and CREATE INDEX statements
```

---

### Task 1.2: Create vendor_health_checks Table

```
CONTEXT:
We need to track individual health check results to calculate vendor uptime. A background worker will ping each vendor's health endpoint every minute and store the result.

CURRENT STATE:
- Vendors have a health_endpoint column (e.g., "/health")
- No health check history exists
- We need to calculate uptime % from historical data

YOUR TASK:
Add the `vendor_health_checks` table to store individual health check results.

REQUIREMENTS:

1. Table must have these columns:
   - id (SERIAL PK)
   - vendor_id (FK to vendors, NOT NULL)
   - status (VARCHAR 10) — 'up', 'down', 'degraded'
   - response_time_ms (INTEGER) — NULL if failed
   - status_code (INTEGER) — HTTP status code, NULL if timeout
   - error_message (TEXT) — error details if failed
   - checked_at (TIMESTAMP WITH TIME ZONE, DEFAULT NOW())

2. This table will grow FAST (1 row per vendor per minute)
   - 100 vendors × 60 checks/hour × 24 hours = 144,000 rows/day
   - We need partitioning by time

3. Partitioning strategy:
   - Partition by RANGE on checked_at
   - Monthly partitions
   - Include partition creation for current and next month

4. Indexes:
   - Composite index on (vendor_id, checked_at DESC) for uptime queries
   - Index on checked_at for partition pruning

5. Add comment explaining retention policy (90 days)

DELIVERABLE:
- Complete SQL for partitioned table
- Partition creation statements
- All indexes
- Match existing schema.sql style
```

---

### Task 1.3: Alter vendors Table

```
CONTEXT:
The existing vendors table needs new columns to support marketplace features like verification badges, featured placement, and searchable tags.

CURRENT STATE:
Read the current vendors table definition from:
machpay-backend/internal/db/sql/schema.sql

YOUR TASK:
Add new columns and indexes to the vendors table for marketplace features.

REQUIREMENTS:

1. New columns to add:
   - is_verified (BOOLEAN DEFAULT FALSE) — verified vendor badge
   - featured_order (INTEGER DEFAULT NULL) — position in featured list (NULL = not featured)
   - tags (TEXT[] DEFAULT '{}') — searchable tags array

2. New indexes to create:
   - GIN index on tags array for fast tag searches
   - Full-text search index on (service_name, description) using to_tsvector
   - Partial index on featured_order WHERE featured_order IS NOT NULL
   - Partial index on is_verified WHERE is_verified = TRUE

3. The ALTER statements should be idempotent (use IF NOT EXISTS where possible)

4. Add to the vendors section of schema.sql with clear comments

DELIVERABLE:
- ALTER TABLE statements for new columns
- CREATE INDEX statements for new indexes
- Comments explaining each addition
```

---

### Task 1.4: Create Migration Script

```
CONTEXT:
We need a migration script that can be run on existing production databases. The script must be idempotent (safe to run multiple times) and include rollback capability.

CURRENT STATE:
- Migrations are in: machpay-backend/internal/db/migrations/
- Existing migrations follow pattern: 00X_description.sql
- Latest migration number needs to be checked

YOUR TASK:
Create a complete migration script for all Phase 1 database changes.

REQUIREMENTS:

1. File: machpay-backend/internal/db/migrations/007_marketplace_foundation.sql

2. Structure:
   ```
   -- Migration: 007_marketplace_foundation
   -- Description: Add tables and columns for Marketplace feature
   -- Author: [CTO]
   -- Date: 2024-12-31
   
   -- ============================================================
   -- UP MIGRATION
   -- ============================================================
   
   [All CREATE and ALTER statements]
   
   -- ============================================================
   -- SEED DATA
   -- ============================================================
   
   [Insert default vendor_metrics for existing vendors]
   
   -- ============================================================
   -- DOWN MIGRATION (Rollback)
   -- ============================================================
   -- To rollback, run these statements in reverse order:
   
   [All DROP and ALTER DROP statements as comments]
   ```

3. Seed data requirement:
   - For each existing vendor, insert a row in vendor_metrics with default values
   - Use INSERT ... SELECT to do this in one statement

4. Rollback must be complete and reversible

5. Add transaction wrapper (BEGIN/COMMIT) for atomicity

DELIVERABLE:
- Complete migration file
- Tested rollback statements
- Clear section comments
```

---

### Task 1.5: Create Go Model Structs

```
CONTEXT:
We need Go structs to represent the new database tables. These will be used by repositories to scan SQL results.

CURRENT STATE:
- Models are in: machpay-backend/internal/models/sql/
- Existing pattern: vendor.go has Vendor struct
- We use database/sql types (sql.NullString, sql.NullTime, etc.)

YOUR TASK:
Add model structs for vendor_metrics and related types.

REQUIREMENTS:

1. Add to vendor.go (or create vendor_metrics.go if cleaner):

   VendorMetrics struct:
   - All fields from vendor_metrics table
   - Use appropriate Go types (float64 for DECIMAL, int64 for BIGINT)
   - Use sql.NullTime for nullable timestamps
   
   VendorWithMetrics struct:
   - Embed Vendor
   - Include Metrics VendorMetrics field
   
   VendorSearchResult struct (for API responses):
   - ID, AppID, Slug, Name, Description, Category
   - LogoURL, IsVerified
   - Pricing (nested struct)
   - Metrics (nested struct with subset of fields)

   VendorPricingInfo struct:
   - Model string
   - PriceUSD float64
   - Currency string

   VendorMetricsInfo struct:
   - Uptime float64
   - AvgLatencyMs int
   - Rating float64
   - TotalRequests int64
   - SuccessRate float64

2. Add ToSearchResult() method on VendorWithMetrics

3. Add ToMetricsInfo() method on VendorMetrics

4. Follow existing code style and comments

DELIVERABLE:
- Complete Go code for all structs
- All methods implemented
- Proper JSON tags for API responses
- Comments matching existing style
```

---

### Task 1.6: Create VendorMetricsRepo Interface and Implementation

```
CONTEXT:
We need a repository to handle CRUD operations on the vendor_metrics table.

CURRENT STATE:
- Repositories are in: machpay-backend/internal/repository/
- Pattern: interface + implementation in same file
- Raw SQL queries as constants
- Scanner helper functions

YOUR TASK:
Create the VendorMetricsRepo for the vendor_metrics table.

REQUIREMENTS:

1. New file: machpay-backend/internal/repository/vendor_metrics_repo.go

2. Interface definition:
   ```go
   type VendorMetricsRepo interface {
       GetByVendorID(ctx context.Context, vendorID int64) (*models.VendorMetrics, error)
       Upsert(ctx context.Context, metrics *models.VendorMetrics) error
       BulkUpsert(ctx context.Context, metrics []*models.VendorMetrics) error
       GetTopByRating(ctx context.Context, limit int) ([]*models.VendorMetrics, error)
       GetTopByUptime(ctx context.Context, limit int) ([]*models.VendorMetrics, error)
       IncrementRequests(ctx context.Context, vendorID int64, count int64) error
       UpdateLastRequest(ctx context.Context, vendorID int64) error
   }
   ```

3. Implementation:
   - VendorMetricsRepoImpl struct with *sql.DB
   - Constructor: NewVendorMetricsRepo(db *sql.DB) VendorMetricsRepo
   - SQL query constants at top of file
   - scanVendorMetrics helper function
   - Proper error handling with wrapped errors
   - Logging with zap

4. SQL queries to implement:
   - SELECT with all columns for GetByVendorID
   - INSERT ... ON CONFLICT DO UPDATE for Upsert
   - Batch insert for BulkUpsert (use CTE or VALUES list)
   - ORDER BY rating_score DESC LIMIT for GetTopByRating
   - ORDER BY uptime_pct DESC LIMIT for GetTopByUptime
   - UPDATE ... SET total_requests = total_requests + $2 for IncrementRequests
   - UPDATE ... SET last_request_at = NOW() for UpdateLastRequest

5. Follow existing patterns from vendor_repo.go

DELIVERABLE:
- Complete repository file
- All methods implemented
- SQL queries as constants
- Error handling and logging
- Follow existing code conventions
```

---

## ✅ Phase 1 Verification Prompt

After completing all tasks, use this prompt to verify:

```
I've completed Phase 1 of the Marketplace implementation. Please review my changes and verify:

FILES MODIFIED/CREATED:
1. machpay-backend/internal/db/sql/schema.sql
2. machpay-backend/internal/db/migrations/007_marketplace_foundation.sql
3. machpay-backend/internal/models/sql/vendor.go (or vendor_metrics.go)
4. machpay-backend/internal/repository/vendor_metrics_repo.go

VERIFICATION CHECKLIST:

Database Schema:
- [ ] vendor_metrics table exists with all required columns
- [ ] vendor_health_checks table exists with partitioning
- [ ] vendors table has is_verified, featured_order, tags columns
- [ ] All indexes created for sorting and searching
- [ ] Full-text search index on vendors

Migration:
- [ ] Migration file numbered correctly
- [ ] UP migration creates all objects
- [ ] Seed data populates vendor_metrics for existing vendors
- [ ] DOWN migration documented for rollback
- [ ] Transaction wrapper included

Models:
- [ ] VendorMetrics struct matches table
- [ ] VendorWithMetrics embeds Vendor + Metrics
- [ ] VendorSearchResult for API responses
- [ ] JSON tags correct
- [ ] Conversion methods implemented

Repository:
- [ ] Interface defined with all methods
- [ ] Implementation complete
- [ ] SQL queries as constants
- [ ] Error handling with wrapped errors
- [ ] Logging included

Please verify each item and flag any issues.
```

---

## 🧪 Testing Prompt

Use this to generate tests:

```
CONTEXT:
Phase 1 of Marketplace is complete. We need to test the new database objects and repository.

YOUR TASK:
Create SQL scripts and Go tests to verify Phase 1 implementation.

REQUIREMENTS:

1. SQL test script to run manually:
   - Verify all tables exist
   - Verify all columns have correct types
   - Verify all indexes exist
   - Insert test data
   - Query to verify indexes are used (EXPLAIN)

2. Go test file for vendor_metrics_repo:
   - Test GetByVendorID (found and not found cases)
   - Test Upsert (insert and update cases)
   - Test GetTopByRating with limit
   - Test GetTopByUptime with limit
   - Test IncrementRequests
   - Test UpdateLastRequest

3. Use testcontainers-go for isolated PostgreSQL instance

4. Follow existing test patterns in the codebase

DELIVERABLE:
- SQL test script
- Go test file
- Test data fixtures
```

---

## 📊 Progress Tracking

Copy this to track completion:

```markdown
## Phase 1 Progress

### Day 1
- [ ] Task 1.1: vendor_metrics table — 1 hour
- [ ] Task 1.2: vendor_health_checks table — 1.5 hours
- [ ] Task 1.3: vendors table alterations — 1 hour
- [ ] Task 1.4: Migration script — 1.5 hours

### Day 2
- [ ] Task 1.5: Go model structs — 2 hours
- [ ] Task 1.6: VendorMetricsRepo — 3 hours
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
3. Read current schema: `machpay-backend/internal/db/sql/schema.sql`
4. Execute Task 1.1 through 1.6 in order
5. Run verification prompt
6. Run migration on local database
7. Run tests

---

## Next Phase

After Phase 1 is complete, proceed to:
**`machpay-docs/implementation/phase2-prompts.md`** (to be created)

