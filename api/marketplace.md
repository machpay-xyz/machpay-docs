# Marketplace API Reference

> **Version**: v1  
> **Base URL**: `https://api.machpay.xyz/v1`  
> **Authentication**: None required (public API)

The MachPay Marketplace API enables AI agents and developers to discover, compare, and evaluate API vendors based on real-time performance metrics.

---

## Overview

### Rate Limiting

| Tier | Limit | Burst |
|------|-------|-------|
| Public | 100 requests/minute | 20 requests/second |
| Authenticated | 1000 requests/minute | 50 requests/second |

Rate limit headers:
- `X-RateLimit-Limit`: Maximum requests per window
- `X-RateLimit-Remaining`: Remaining requests in current window
- `X-RateLimit-Reset`: Unix timestamp when window resets

### Response Format

All responses are JSON with the following wrapper:

```json
{
  "data": { ... },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

Error responses:
```json
{
  "error": "error_code",
  "message": "Human-readable error message",
  "details": { ... }
}
```

---

## Endpoints

### Search Vendors

Search and filter vendors in the marketplace.

```
GET /v1/marketplace/search
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `q` | string | `""` | Search query (natural language, searches name & description) |
| `category` | string | `""` | Filter by category: `ai`, `weather`, `data`, `finance`, `other` |
| `sort` | string | `"rating"` | Sort order: `rating`, `uptime`, `popular`, `fastest`, `recent` |
| `min_uptime` | float | - | Minimum uptime percentage (0-100) |
| `min_rating` | float | - | Minimum rating score (0-5) |
| `max_latency` | int | - | Maximum average latency in milliseconds |
| `verified` | bool | - | Only return verified vendors |
| `limit` | int | `20` | Results per page (max: 100) |
| `offset` | int | `0` | Pagination offset |

#### Example Request

```bash
curl -X GET "https://api.machpay.xyz/v1/marketplace/search?q=weather&category=weather&min_uptime=99&sort=rating&limit=10"
```

#### Response

```json
{
  "vendors": [
    {
      "id": 123,
      "appId": "mp_app_xyz123",
      "slug": "openweather-pro",
      "serviceName": "OpenWeather Pro",
      "description": "Professional weather data API with global coverage",
      "category": "weather",
      "logoUrl": "https://cdn.machpay.xyz/logos/openweather.png",
      "baseUrl": "https://api.openweather.pro",
      "defaultPriceUsd": 0.001,
      "tags": ["enterprise", "forecast", "global"],
      "isVerified": true,
      "featuredOrder": 1,
      "uptimePct": 99.95,
      "avgLatencyMs": 45,
      "p95LatencyMs": 120,
      "p99LatencyMs": 250,
      "ratingScore": 4.8,
      "totalRequests": 1500000,
      "successRate": 99.8,
      "uniqueAgents": 342,
      "lastRequestAt": "2024-01-15T10:30:00Z"
    }
  ],
  "total": 42,
  "limit": 10,
  "offset": 0,
  "hasMore": true,
  "categories": [
    { "category": "weather", "count": 8 },
    { "category": "ai", "count": 25 }
  ]
}
```

---

### Advanced Search

Full-featured search with request body for complex queries.

```
POST /v1/marketplace/search
```

#### Request Body

```json
{
  "query": "weather forecast API",
  "categories": ["weather", "data"],
  "tags": ["enterprise", "global"],
  "minUptime": 99.0,
  "maxLatency": 100,
  "minRating": 4.0,
  "verifiedOnly": true,
  "sortBy": "rating",
  "limit": 20,
  "offset": 0
}
```

#### Response

Same as Search Vendors response.

---

### Get Featured Vendors

Returns curated list of featured vendors, hand-picked for quality.

```
GET /v1/marketplace/featured
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | int | `10` | Number of featured vendors (max: 20) |

#### Example Request

```bash
curl -X GET "https://api.machpay.xyz/v1/marketplace/featured?limit=5"
```

#### Response

```json
{
  "vendors": [
    {
      "id": 123,
      "appId": "mp_app_xyz123",
      "slug": "openweather-pro",
      "serviceName": "OpenWeather Pro",
      "description": "Professional weather data API",
      "category": "weather",
      "isVerified": true,
      "featuredOrder": 1,
      "uptimePct": 99.95,
      "avgLatencyMs": 45,
      "ratingScore": 4.8,
      "defaultPriceUsd": 0.001
    }
  ]
}
```

---

### Get Vendor Details

Get detailed information for a specific vendor.

```
GET /v1/marketplace/vendors/:identifier
```

The `identifier` can be:
- Numeric ID: `/v1/marketplace/vendors/123`
- Slug: `/v1/marketplace/vendors/openweather-pro`

#### Example Request

```bash
curl -X GET "https://api.machpay.xyz/v1/marketplace/vendors/openweather-pro"
```

#### Response

```json
{
  "id": 123,
  "appId": "mp_app_xyz123",
  "slug": "openweather-pro",
  "serviceName": "OpenWeather Pro",
  "description": "Professional weather data API with global coverage, supporting current conditions, forecasts, and historical data.",
  "category": "weather",
  "logoUrl": "https://cdn.machpay.xyz/logos/openweather.png",
  "websiteUrl": "https://openweather.pro",
  "baseUrl": "https://api.openweather.pro",
  "defaultPriceUsd": 0.001,
  "tags": ["enterprise", "forecast", "global", "weather"],
  "isVerified": true,
  "featuredOrder": 1,
  "createdAt": "2024-01-01T00:00:00Z",
  "metrics": {
    "uptimePct": 99.95,
    "avgLatencyMs": 45,
    "p95LatencyMs": 120,
    "p99LatencyMs": 250,
    "ratingScore": 4.8,
    "totalRequests": 1500000,
    "successRate": 99.8,
    "uniqueAgents": 342,
    "totalRevenueUsd": 15432.50,
    "lastRequestAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:35:00Z"
  },
  "endpoints": [
    {
      "path": "/current",
      "method": "GET",
      "name": "Current Weather",
      "description": "Get current weather for a location",
      "priceUsd": 0.0005
    },
    {
      "path": "/forecast",
      "method": "GET",
      "name": "Weather Forecast",
      "description": "Get 7-day weather forecast",
      "priceUsd": 0.002
    }
  ]
}
```

---

### Get Vendor Health History

Get recent health check results for a vendor.

```
GET /v1/marketplace/vendors/:id/health
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | int | `24` | Number of health checks (max: 100) |

#### Example Request

```bash
curl -X GET "https://api.machpay.xyz/v1/marketplace/vendors/123/health?limit=24"
```

#### Response

```json
{
  "vendorId": 123,
  "checks": [
    {
      "id": 10001,
      "isHealthy": true,
      "responseTimeMs": 42,
      "statusCode": 200,
      "checkedAt": "2024-01-15T10:30:00Z"
    },
    {
      "id": 10000,
      "isHealthy": false,
      "responseTimeMs": 5000,
      "statusCode": 503,
      "error": "Service Unavailable",
      "checkedAt": "2024-01-15T10:25:00Z"
    }
  ],
  "summary": {
    "last24h": {
      "uptime": 99.17,
      "avgLatencyMs": 48,
      "checksTotal": 24,
      "checksPassed": 23
    }
  }
}
```

---

### Get Categories

Get all vendor categories with counts.

```
GET /v1/marketplace/categories
```

#### Example Request

```bash
curl -X GET "https://api.machpay.xyz/v1/marketplace/categories"
```

#### Response

```json
{
  "categories": [
    { "category": "ai", "count": 45 },
    { "category": "data", "count": 23 },
    { "category": "weather", "count": 12 },
    { "category": "finance", "count": 8 },
    { "category": "other", "count": 15 }
  ],
  "total": 103
}
```

---

### Get Vendors by Category

Get all vendors in a specific category.

```
GET /v1/marketplace/categories/:category
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sort` | string | `"rating"` | Sort order |
| `limit` | int | `20` | Results per page |
| `offset` | int | `0` | Pagination offset |

#### Example Request

```bash
curl -X GET "https://api.machpay.xyz/v1/marketplace/categories/ai?sort=popular&limit=10"
```

#### Response

Same as Search Vendors response.

---

### Quick Search (Autocomplete)

Lightweight search optimized for autocomplete UX.

```
GET /v1/marketplace/quick-search
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `q` | string | **required** | Search query (min 2 chars) |
| `limit` | int | `5` | Number of suggestions (max: 10) |

#### Example Request

```bash
curl -X GET "https://api.machpay.xyz/v1/marketplace/quick-search?q=weath&limit=5"
```

#### Response

```json
{
  "results": [
    {
      "id": 123,
      "serviceName": "OpenWeather Pro",
      "slug": "openweather-pro",
      "category": "weather",
      "rating": 4.8
    },
    {
      "id": 124,
      "serviceName": "Weather Underground",
      "slug": "weather-underground",
      "category": "weather",
      "rating": 4.5
    }
  ]
}
```

---

## Data Types

### VendorDiscoveryResult

Complete vendor information returned in search results.

```typescript
interface VendorDiscoveryResult {
  // Identity
  id: number;
  appId: string;              // "mp_app_xyz123"
  slug: string;               // URL-safe identifier
  serviceName: string;
  description: string;
  
  // Branding
  logoUrl?: string;
  websiteUrl?: string;
  category: string;           // "ai" | "weather" | "data" | "finance" | "other"
  tags: string[];
  
  // Pricing
  defaultPriceUsd: number;    // Price per request in USD
  
  // Status
  isVerified: boolean;
  featuredOrder?: number;     // Position in featured list (null if not featured)
  
  // Performance Metrics
  uptimePct: number;          // 0-100
  avgLatencyMs: number;
  p95LatencyMs: number;
  p99LatencyMs: number;
  ratingScore: number;        // 0-5
  successRate: number;        // 0-100
  
  // Usage
  totalRequests: number;
  uniqueAgents: number;
  lastRequestAt?: string;     // ISO 8601 timestamp
  
  // Search
  relevanceScore?: number;    // Present when searching with query
}
```

### HealthCheck

Individual health check result.

```typescript
interface HealthCheck {
  id: number;
  isHealthy: boolean;
  responseTimeMs: number;
  statusCode?: number;
  error?: string;
  checkedAt: string;          // ISO 8601 timestamp
}
```

### CategoryInfo

Category with vendor count.

```typescript
interface CategoryInfo {
  category: string;
  count: number;
}
```

---

## Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `invalid_request` | Malformed request parameters |
| 400 | `invalid_query` | Search query too short or invalid |
| 404 | `vendor_not_found` | Vendor with given ID/slug not found |
| 404 | `category_not_found` | Category does not exist |
| 429 | `rate_limited` | Too many requests, try again later |
| 500 | `internal_error` | Unexpected server error |

### Error Response Example

```json
{
  "error": "vendor_not_found",
  "message": "No vendor found with identifier 'nonexistent-api'",
  "details": {
    "identifier": "nonexistent-api"
  }
}
```

---

## SDK Examples

### Python (MachPay SDK)

```python
from machpay import MachPayClient

async with MachPayClient.from_env() as client:
    # Search for weather APIs
    results = await client.discovery.search(
        query="weather forecast",
        categories=["weather"],
        min_uptime=99.5,
        sort_by="rating"
    )
    
    print(f"Found {results.total} vendors")
    
    # Get the best vendor
    best = results.top_by_rating(1)[0]
    print(f"Best: {best.name} ({best.metrics.uptime_pct}% uptime)")
    
    # Make a call through the vendor
    response = await client.pay_and_call(
        url=f"{best.gateway_url()}/current",
        json={"city": "NYC"}
    )
```

### JavaScript / TypeScript

```typescript
// Using fetch
const response = await fetch(
  "https://api.machpay.xyz/v1/marketplace/search?q=weather&min_uptime=99"
);
const { vendors, total } = await response.json();

console.log(`Found ${total} vendors`);
for (const vendor of vendors) {
  console.log(`${vendor.serviceName}: ${vendor.uptimePct}% uptime`);
}

// Get vendor details
const details = await fetch(
  `https://api.machpay.xyz/v1/marketplace/vendors/${vendors[0].slug}`
).then(r => r.json());
```

### cURL

```bash
# Search for AI APIs with high uptime
curl -X GET "https://api.machpay.xyz/v1/marketplace/search?q=LLM&category=ai&min_uptime=99.5&sort=rating"

# Get featured vendors
curl -X GET "https://api.machpay.xyz/v1/marketplace/featured?limit=5"

# Get vendor by slug
curl -X GET "https://api.machpay.xyz/v1/marketplace/vendors/openai-gateway"

# Get health history
curl -X GET "https://api.machpay.xyz/v1/marketplace/vendors/123/health?limit=10"

# Get all categories
curl -X GET "https://api.machpay.xyz/v1/marketplace/categories"

# Advanced search with POST
curl -X POST "https://api.machpay.xyz/v1/marketplace/search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "image generation",
    "categories": ["ai"],
    "minUptime": 99.0,
    "maxLatency": 200,
    "verifiedOnly": true,
    "sortBy": "popular",
    "limit": 10
  }'
```

---

## Best Practices

### Caching

- Cache `categories` response for 5 minutes
- Cache `featured` response for 5 minutes
- Cache individual vendor details for 1 minute
- Search results should not be cached (dynamic)

### Pagination

- Use `limit` and `offset` for large result sets
- Check `hasMore` to determine if more pages exist
- Maximum `limit` is 100

### Search Tips

- Use natural language queries: "weather API with forecast"
- Combine filters for best results: `?q=weather&min_uptime=99&category=weather`
- Use `quick-search` for autocomplete with debounce (300ms)

### Sorting

| Sort Value | Description |
|------------|-------------|
| `rating` | Highest rated first (default) |
| `uptime` | Most reliable first |
| `popular` | Most used (by request count) first |
| `fastest` | Lowest latency first |
| `recent` | Most recently active first |

---

## Changelog

### v1.0.0 (2024-01-15)

- Initial release of Marketplace API
- Endpoints: search, featured, details, health, categories, quick-search
- Sorting: rating, uptime, popular, fastest, recent
- Filtering: category, min_uptime, min_rating, max_latency, verified

