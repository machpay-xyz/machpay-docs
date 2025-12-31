# Vendor Detail Page - Implementation Plan

## Overview

Enhance the Marketplace experience with a comprehensive Vendor Detail Page that shows complete vendor information including contract terms, authorization requirements, and integration details. The "Connect to Vendor" action should initiate the Agent Onboarding flow.

---

## Current State Analysis

### What We Have
- Marketplace page with vendor cards and a side panel
- Basic vendor info: name, category, description, metrics
- Simple "Connect to Vendor" button (no action)

### What's Missing
- Dedicated vendor detail page
- Contract/terms information
- Authorization requirements
- Endpoint documentation
- Integration examples
- Health check history visualization
- Connect flow integration

---

## Target Architecture

### New Routes
```
/marketplace                    → Vendor discovery (existing)
/marketplace/:slug              → Vendor detail page (NEW)
/marketplace/:slug/endpoints    → API documentation (future)
```

### Vendor Detail Page Sections

```
┌─────────────────────────────────────────────────────────────────┐
│  ← Back to Marketplace                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [LOGO]  Vendor Name                           [Connect Button] │
│          @slug • Category • ✓ Verified                          │
│          ★ 4.95 (1,250 reviews)                                 │
│                                                                  │
├──────────────────────┬──────────────────────────────────────────┤
│                      │                                          │
│  OVERVIEW            │  PERFORMANCE METRICS                     │
│  ─────────           │  ────────────────────                    │
│  Description         │  ┌────┐ ┌────┐ ┌────┐ ┌────┐            │
│  Tags                │  │99.9│ │45ms│ │4.95│ │2.5M│            │
│                      │  │UPT │ │LAT │ │RATE│ │REQ │            │
│                      │  └────┘ └────┘ └────┘ └────┘            │
│                      │                                          │
│                      │  UPTIME CHART (24h/7d/30d)              │
│                      │  ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄                 │
│                      │                                          │
├──────────────────────┴──────────────────────────────────────────┤
│                                                                  │
│  PRICING & CONTRACT                                             │
│  ─────────────────────                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Pricing Model: Per Request                               │   │
│  │ Base Price: $0.0025 / request                           │   │
│  │ Currency: USDC (Solana)                                 │   │
│  │ Minimum Charge: $0.001                                  │   │
│  │ Free Tier: 100 requests/day                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Contract Terms:                                                │
│  • Settlement: Instant via x402 protocol                       │
│  • Dispute Window: 24 hours                                    │
│  • SLA: 99.9% uptime guaranteed                               │
│  • Rate Limit: 1000 req/min                                   │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AUTHORIZATION                                                  │
│  ─────────────                                                  │
│  Payment Method: x402 Payment Protocol                         │
│  Required Headers:                                              │
│    • X-Payment: <signed payment intent>                        │
│    • X-Agent-Id: <your agent public key>                       │
│                                                                  │
│  [View Integration Guide]                                       │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ENDPOINTS                                                      │
│  ─────────                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ POST /v1/generate                         $0.003/req     │  │
│  │      Text generation with GPT-4                          │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ POST /v1/embeddings                       $0.0001/req    │  │
│  │      Generate text embeddings                            │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ GET  /v1/models                           FREE           │  │
│  │      List available models                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  QUICK START                                                    │
│  ───────────                                                    │
│  ```python                                                      │
│  from machpay import MachPayClient                              │
│                                                                  │
│  client = MachPayClient(agent_key="...")                       │
│  response = client.call(                                        │
│      vendor="openai-proxy",                                    │
│      endpoint="/v1/generate",                                  │
│      body={"prompt": "Hello!"}                                 │
│  )                                                              │
│  ```                                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Backend API Enhancement

### 1.1 Extended Vendor Response Model

Add new fields to vendor detail response:

```go
type VendorDetailResponse struct {
    // Basic Info (existing)
    ID          int64    `json:"id"`
    AppID       string   `json:"appId"`
    Slug        string   `json:"slug"`
    ServiceName string   `json:"serviceName"`
    Description string   `json:"description"`
    Category    string   `json:"category"`
    LogoURL     string   `json:"logoUrl"`
    WebsiteURL  string   `json:"websiteUrl"`
    SupportEmail string  `json:"supportEmail"`
    IsVerified  bool     `json:"isVerified"`
    Tags        []string `json:"tags"`
    
    // Metrics (existing)
    Metrics VendorMetrics `json:"metrics"`
    
    // NEW: Contract & Pricing
    Contract VendorContract `json:"contract"`
    
    // NEW: Authorization
    Authorization VendorAuth `json:"authorization"`
    
    // NEW: Endpoints
    Endpoints []VendorEndpoint `json:"endpoints"`
    
    // NEW: Health History
    HealthHistory []HealthCheckSummary `json:"healthHistory"`
}

type VendorContract struct {
    PricingModel    string  `json:"pricingModel"`    // "per_request", "subscription"
    BasePriceUSD    float64 `json:"basePriceUsd"`
    Currency        string  `json:"currency"`        // "USDC"
    Network         string  `json:"network"`         // "solana"
    MinimumCharge   float64 `json:"minimumCharge"`
    FreeQuota       int     `json:"freeQuota"`       // Free requests per day
    RateLimit       int     `json:"rateLimit"`       // Requests per minute
    TimeoutMs       int     `json:"timeoutMs"`
    SLAUptime       float64 `json:"slaUptime"`       // 99.9
    DisputeWindowHrs int    `json:"disputeWindowHrs"` // 24
    SettlementType  string  `json:"settlementType"`  // "instant", "batch"
}

type VendorAuth struct {
    PaymentProtocol string   `json:"paymentProtocol"` // "x402"
    RequiredHeaders []string `json:"requiredHeaders"`
    AuthType        string   `json:"authType"`        // "payment_intent"
    DocsURL         string   `json:"docsUrl"`
}

type HealthCheckSummary struct {
    Date      string  `json:"date"`      // "2024-01-15"
    UptimePct float64 `json:"uptimePct"`
    AvgLatency int    `json:"avgLatencyMs"`
    Checks    int     `json:"checks"`
}
```

### 1.2 New API Endpoint

```
GET /v1/marketplace/vendors/:idOrSlug
```

Returns full vendor detail with all sections.

### 1.3 Health History Endpoint

```
GET /v1/marketplace/vendors/:id/health?days=30
```

Returns daily aggregated health stats.

---

## Phase 2: Console UI - Vendor Detail Page

### 2.1 New Route

```jsx
// App.jsx
<Route path="/marketplace/:slug" element={<VendorDetailPage />} />
```

### 2.2 Components

```
src/pages/
  └── marketplace/
      ├── VendorDetailPage.jsx      # Main detail page
      └── components/
          ├── VendorHeader.jsx       # Logo, name, verified badge
          ├── MetricsPanel.jsx       # Performance metrics cards
          ├── UptimeChart.jsx        # Health history chart
          ├── ContractSection.jsx    # Pricing & terms
          ├── AuthSection.jsx        # Authorization details
          ├── EndpointsTable.jsx     # API endpoints list
          └── QuickStartCode.jsx     # Integration example
```

### 2.3 Navigation Flow

```
Marketplace Card → Click → VendorDetailPage
                         ↓
                   [Connect Button]
                         ↓
              Check if user is agent?
                    ↙        ↘
                 YES          NO
                  ↓            ↓
           Add to Allowlist  Agent Onboarding
```

---

## Phase 3: Connect to Vendor Flow

### 3.1 Connect Button Logic

```jsx
const handleConnect = () => {
  if (!isAuthenticated) {
    navigate('/auth', { state: { returnTo: `/marketplace/${slug}` } });
    return;
  }
  
  if (!user.isAgent) {
    // Start agent onboarding with vendor pre-selected
    navigate('/onboarding/agent', { 
      state: { 
        preselectedVendor: vendor,
        returnTo: `/marketplace/${slug}` 
      } 
    });
    return;
  }
  
  // User is an agent - add vendor to allowlist
  await api.agents.addToAllowlist(vendor.id);
  toast.success('Vendor added to your allowlist!');
};
```

### 3.2 Agent Onboarding Enhancement

Modify agent onboarding to:
1. Accept pre-selected vendor from navigation state
2. Show vendor in the "Allowed Vendors" step
3. Return to vendor page after completion

---

## Implementation Prompts

### Prompt 1: Backend - Extended Vendor Detail API

```
**Role:** MachPay CTO & Backend Architect

**Context:**
We need to extend the vendor detail API to return comprehensive information including contract terms, authorization requirements, and endpoints.

**Task:**
1. Create new response structs in `internal/models/sql/vendor_detail.go`:
   - `VendorDetailResponse` with all fields
   - `VendorContract` for pricing/terms
   - `VendorAuth` for authorization info
   - `HealthCheckSummary` for daily stats

2. Update `VendorRepo` in `internal/repository/vendor_repo.go`:
   - Add `GetDetailBySlug(slug string)` method
   - Join with vendor_endpoints table
   - Include pricing_model_json parsing

3. Add health history query in `internal/repository/health_check_repo.go`:
   - `GetDailyStats(vendorID int64, days int)` - aggregate by day

4. Create new handler in `internal/handlers/sql/discovery_handler.go`:
   - `HandleGetVendorDetail` for `/v1/marketplace/vendors/:idOrSlug`
   - Parse slug vs numeric ID
   - Return full detail response

5. Register route in `cmd/api/main.go`

**Acceptance Criteria:**
- `curl localhost:8081/v1/marketplace/vendors/openai-proxy` returns full detail
- Contract info populated from pricing_model_json
- Endpoints from vendor_endpoints table
- Health history with daily aggregates
```

### Prompt 2: Console - Vendor Detail Page

```
**Role:** MachPay CTO & Frontend Architect

**Context:**
Create a comprehensive Vendor Detail Page that displays all vendor information in an organized, professional layout.

**Task:**
1. Create new page `src/pages/marketplace/VendorDetailPage.jsx`:
   - Route: `/marketplace/:slug`
   - Fetch vendor detail from API
   - Bloomberg Terminal aesthetic

2. Create sub-components:
   - `VendorHeader.jsx` - Logo, name, category, verified badge, rating
   - `MetricsPanel.jsx` - 4 metric cards (uptime, latency, rating, requests)
   - `UptimeChart.jsx` - Line chart for health history (use recharts)
   - `ContractSection.jsx` - Pricing model, terms, SLA
   - `AuthSection.jsx` - Payment protocol, required headers
   - `EndpointsTable.jsx` - List of API endpoints with prices
   - `QuickStartCode.jsx` - Python/JS code example

3. Add route to `App.jsx`

4. Update Marketplace.jsx:
   - Card click → navigate to detail page
   - Featured vendor click → navigate to detail page

**Design Requirements:**
- Dark theme consistent with console
- Responsive layout
- Loading states
- Error handling
- Back navigation

**Acceptance Criteria:**
- Navigate from marketplace to vendor detail
- All sections render with data
- Responsive on mobile
- Loading spinner while fetching
```

### Prompt 3: Connect to Vendor Flow

```
**Role:** MachPay CTO & Product Architect

**Context:**
The "Connect to Vendor" button should integrate with the Agent Onboarding flow for non-agents.

**Task:**
1. Update VendorDetailPage.jsx:
   - Add `handleConnect` function
   - Check auth state and agent status
   - Navigate appropriately

2. Update AgentOnboarding.jsx (`src/pages/AgentOnboarding.jsx`):
   - Accept `preselectedVendor` from navigation state
   - Pre-populate vendor in allowlist step
   - Show vendor info in onboarding UI
   - Return to vendor page after completion

3. Add API endpoint for adding vendor to allowlist:
   - `POST /v1/agents/:agentId/allowlist`
   - Body: `{ vendorId: number }`

4. Create success toast/modal after connection

**Flow:**
- Not logged in → Auth page → Return to vendor
- Logged in, not agent → Agent onboarding with vendor pre-selected
- Logged in, is agent → Add to allowlist API → Success toast

**Acceptance Criteria:**
- Non-agent users directed to onboarding
- Vendor pre-selected in onboarding
- Agents can add vendor with one click
- Success feedback shown
```

### Prompt 4: Update Seed Data

```
**Role:** MachPay CTO

**Context:**
Seed data needs to include realistic contract info, endpoints, and more detailed vendor profiles.

**Task:**
1. Update `scripts/seed_marketplace.sql`:
   - Add vendor_endpoints for each vendor
   - Populate pricing_model_json with full contract info
   - Add logo_url, website_url, support_email

2. Contract data structure in pricing_model_json:
   ```json
   {
     "type": "per_request",
     "basePriceUsd": 0.0025,
     "currency": "USDC",
     "network": "solana",
     "minimumCharge": 0.001,
     "freeQuota": 100,
     "rateLimit": 1000,
     "timeoutMs": 30000,
     "slaUptime": 99.9,
     "disputeWindowHrs": 24,
     "settlementType": "instant"
   }
   ```

3. Sample endpoints per vendor:
   - OpenAI Proxy: /v1/chat/completions, /v1/embeddings
   - Weather API: /v1/current, /v1/forecast
   - Chainlink: /v1/prices, /v1/feeds

**Acceptance Criteria:**
- Each vendor has 2-3 endpoints
- Full contract info in JSON
- Vendor profiles complete
```

---

## Execution Order

| Phase | Prompt | Description | Est. Time |
|-------|--------|-------------|-----------|
| 1.1 | Prompt 4 | Update seed data first | 30 min |
| 1.2 | Prompt 1 | Backend API enhancement | 1 hr |
| 2.1 | Prompt 2 | Vendor detail page | 2 hr |
| 3.1 | Prompt 3 | Connect flow | 1 hr |

**Total Estimated Time:** 4-5 hours

---

## Success Metrics

1. **Vendor Detail Page** loads in < 500ms
2. **All sections** render with data
3. **Connect flow** works for all user states
4. **Playwright tests** pass for new page
5. **Mobile responsive** layout works

