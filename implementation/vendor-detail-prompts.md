# Vendor Detail Page - Implementation Prompts

## Role Setup (Use for all prompts)

```
You are the CTO and Lead Product Architect of MachPay - the revolutionary payment 
infrastructure for AI agents. You think in systems, prioritize developer experience, 
and build for scale. Every feature should feel native to the Bloomberg Terminal 
aesthetic of the console.
```

---

## Prompt 1: Seed Data Enhancement

**Objective:** Update seed data with complete vendor profiles, contract info, and endpoints.

**Context:**
The Marketplace needs realistic vendor data including:
- Complete pricing/contract information in `pricing_model_json`
- Multiple API endpoints per vendor
- Full vendor profiles (logo, website, support email)

**Files to Modify:**
- `machpay-backend/scripts/seed_marketplace.sql`

**Task:**

1. **Update pricing_model_json for all vendors** with this structure:
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

2. **Add vendor_endpoints** for each seeded vendor:
```sql
-- OpenAI Proxy endpoints
INSERT INTO vendor_endpoints (vendor_id, path, method, name, description, price_per_call, is_active)
SELECT v.id, '/v1/chat/completions', 'POST', 'Chat Completion', 'Generate chat responses with GPT models', 0.003, TRUE
FROM vendors v WHERE v.slug = 'openai-proxy'
ON CONFLICT (vendor_id, path, method) DO NOTHING;

-- Add 2-3 endpoints per vendor
```

3. **Suggested endpoints per vendor:**

| Vendor | Endpoints |
|--------|-----------|
| OpenAI Proxy | `/v1/chat/completions` ($0.003), `/v1/embeddings` ($0.0001), `/v1/models` (FREE) |
| Anthropic Mirror | `/v1/messages` ($0.003), `/v1/complete` ($0.002) |
| Chainlink Feed | `/v1/prices/:pair` ($0.0002), `/v1/feeds` (FREE) |
| Weather API Pro | `/v1/current` ($0.0005), `/v1/forecast` ($0.001), `/v1/alerts` ($0.0003) |
| GPU Cluster | `/v1/inference` ($0.05), `/v1/batch` ($0.04) |
| Arweave Gateway | `/v1/store` ($0.005), `/v1/retrieve` (FREE) |

4. **Add profile fields:**
```sql
UPDATE vendors SET 
    logo_url = 'https://cdn.machpay.xyz/vendors/openai-proxy/logo.svg',
    website_url = 'https://openai-proxy.machpay.xyz',
    support_email = 'support@openai-proxy.machpay.xyz'
WHERE slug = 'openai-proxy';
```

**Verification:**
```sql
-- Check endpoints exist
SELECT v.service_name, COUNT(e.id) as endpoint_count 
FROM vendors v 
LEFT JOIN vendor_endpoints e ON v.id = e.vendor_id 
GROUP BY v.service_name;

-- Check contract info
SELECT slug, pricing_model_json->>'type' as pricing_type, 
       pricing_model_json->>'slaUptime' as sla
FROM vendors WHERE slug LIKE '%-%';
```

---

## Prompt 2: Backend API - Vendor Detail Endpoint

**Objective:** Create a comprehensive vendor detail API endpoint.

**Context:**
We need `/v1/marketplace/vendors/:idOrSlug` to return complete vendor information including:
- Basic info + metrics (existing)
- Contract terms (from pricing_model_json)
- Authorization requirements
- Endpoints list
- Health history (daily aggregates)

**Files to Create/Modify:**

1. **`internal/models/sql/vendor_detail.go`** (NEW)
```go
package models

// VendorDetailResponse - Complete vendor information for detail page
type VendorDetailResponse struct {
    // Embed existing VendorDiscoveryResult
    VendorDiscoveryResult
    
    // Extended profile
    LogoURL      string `json:"logoUrl,omitempty"`
    WebsiteURL   string `json:"websiteUrl,omitempty"`
    SupportEmail string `json:"supportEmail,omitempty"`
    
    // Contract & Pricing
    Contract VendorContract `json:"contract"`
    
    // Authorization info
    Authorization VendorAuthorization `json:"authorization"`
    
    // API Endpoints
    Endpoints []EndpointInfo `json:"endpoints"`
    
    // Health history (last 30 days)
    HealthHistory []HealthDaySummary `json:"healthHistory"`
}

type VendorContract struct {
    PricingModel     string  `json:"pricingModel"`
    BasePriceUSD     float64 `json:"basePriceUsd"`
    Currency         string  `json:"currency"`
    Network          string  `json:"network"`
    MinimumCharge    float64 `json:"minimumCharge"`
    FreeQuota        int     `json:"freeQuota"`
    RateLimit        int     `json:"rateLimit"`
    TimeoutMs        int     `json:"timeoutMs"`
    SLAUptime        float64 `json:"slaUptime"`
    DisputeWindowHrs int     `json:"disputeWindowHrs"`
    SettlementType   string  `json:"settlementType"`
}

type VendorAuthorization struct {
    PaymentProtocol  string   `json:"paymentProtocol"`
    RequiredHeaders  []string `json:"requiredHeaders"`
    AuthType         string   `json:"authType"`
    IntegrationGuide string   `json:"integrationGuide"`
}

type EndpointInfo struct {
    ID          string  `json:"id"`
    Path        string  `json:"path"`
    Method      string  `json:"method"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    PriceUSD    float64 `json:"priceUsd"`
    IsActive    bool    `json:"isActive"`
}

type HealthDaySummary struct {
    Date       string  `json:"date"`
    UptimePct  float64 `json:"uptimePct"`
    AvgLatency int     `json:"avgLatencyMs"`
    Checks     int     `json:"checks"`
}
```

2. **Update `internal/repository/vendor_repo.go`:**
   - Add `GetDetailBySlugOrID(ctx, idOrSlug string) (*VendorDetailResponse, error)`
   - Query joins vendors + vendor_metrics + vendor_endpoints
   - Parse pricing_model_json into VendorContract struct

3. **Update `internal/repository/health_check_repo.go`:**
   - Add `GetDailySummary(ctx, vendorID int64, days int) ([]HealthDaySummary, error)`
   - SQL: Group by date, aggregate uptime/latency

4. **Update `internal/handlers/sql/discovery_handler.go`:**
   - Add `HandleGetVendorDetail(ctx context.Context, c *app.RequestContext)`
   - Parse `:idOrSlug` param
   - Call service layer
   - Return JSON response

5. **Register route in `cmd/api/main.go`:**
```go
marketplaceGroup.GET("/vendors/:idOrSlug", discoveryHandler.HandleGetVendorDetail)
```

**API Response Example:**
```json
{
  "id": 5,
  "appId": "mp_app_oai001",
  "slug": "openai-proxy",
  "serviceName": "OpenAI Proxy",
  "description": "High-performance GPT-4 proxy...",
  "category": "ai",
  "isVerified": true,
  "tags": ["llm", "gpt4", "enterprise"],
  "logoUrl": "https://cdn.machpay.xyz/vendors/openai-proxy/logo.svg",
  "websiteUrl": "https://openai-proxy.machpay.xyz",
  "metrics": {
    "uptimePct": 99.99,
    "avgLatencyMs": 45,
    "ratingScore": 4.95,
    "totalRequests": 2500000
  },
  "contract": {
    "pricingModel": "per_request",
    "basePriceUsd": 0.0025,
    "currency": "USDC",
    "network": "solana",
    "slaUptime": 99.9,
    "rateLimit": 1000
  },
  "authorization": {
    "paymentProtocol": "x402",
    "requiredHeaders": ["X-Payment", "X-Agent-Id"],
    "authType": "payment_intent",
    "integrationGuide": "/docs/integration/openai-proxy"
  },
  "endpoints": [
    {
      "path": "/v1/chat/completions",
      "method": "POST",
      "name": "Chat Completion",
      "priceUsd": 0.003,
      "isActive": true
    }
  ],
  "healthHistory": [
    {"date": "2024-01-15", "uptimePct": 100, "avgLatencyMs": 42, "checks": 1440}
  ]
}
```

**Verification:**
```bash
curl -s "http://localhost:8081/v1/marketplace/vendors/openai-proxy" | jq .
```

---

## Prompt 3: Console - Vendor Detail Page

**Objective:** Create a comprehensive vendor detail page with all sections.

**Context:**
Create `/marketplace/:slug` route that displays complete vendor information in a professional, Bloomberg Terminal-inspired layout.

**Files to Create:**

1. **`src/pages/marketplace/VendorDetailPage.jsx`** (Main page)
2. **`src/pages/marketplace/components/VendorHeader.jsx`**
3. **`src/pages/marketplace/components/MetricsGrid.jsx`**
4. **`src/pages/marketplace/components/UptimeChart.jsx`**
5. **`src/pages/marketplace/components/ContractCard.jsx`**
6. **`src/pages/marketplace/components/AuthorizationCard.jsx`**
7. **`src/pages/marketplace/components/EndpointsTable.jsx`**
8. **`src/pages/marketplace/components/QuickStartCode.jsx`**

**Files to Modify:**
- `src/App.jsx` - Add route
- `src/api.js` - Add `getVendorDetail` function
- `src/pages/Marketplace.jsx` - Navigate to detail on card click

**Design Specifications:**

1. **VendorHeader:**
   - Large logo (80x80)
   - Vendor name (h1)
   - Slug, category badge, verified checkmark
   - Star rating with count
   - "Connect to Vendor" button (primary CTA)
   - Back to Marketplace link

2. **MetricsGrid:**
   - 4-column grid (responsive)
   - Cards: Uptime %, Avg Latency, Rating, Total Requests
   - Color-coded (green/yellow/red based on thresholds)

3. **UptimeChart:**
   - Recharts AreaChart
   - 30-day health history
   - Toggleable: 24h / 7d / 30d
   - Green fill for uptime

4. **ContractCard:**
   - Pricing model display
   - Price per request (large, green)
   - Terms list (SLA, rate limit, dispute window)
   - Network badge (Solana)

5. **AuthorizationCard:**
   - Payment protocol (x402)
   - Required headers list
   - "View Integration Guide" link

6. **EndpointsTable:**
   - Sortable table
   - Columns: Method, Path, Name, Price, Status
   - Method badges (GET=blue, POST=green)

7. **QuickStartCode:**
   - Tabbed: Python / JavaScript / cURL
   - Copy button
   - Syntax highlighted

**Layout (Tailwind Grid):**
```jsx
<div className="grid grid-cols-12 gap-6">
  {/* Full width header */}
  <div className="col-span-12">
    <VendorHeader />
  </div>
  
  {/* Left: Metrics + Chart */}
  <div className="col-span-12 lg:col-span-8 space-y-6">
    <MetricsGrid />
    <UptimeChart />
    <EndpointsTable />
    <QuickStartCode />
  </div>
  
  {/* Right: Contract + Auth */}
  <div className="col-span-12 lg:col-span-4 space-y-6">
    <ContractCard />
    <AuthorizationCard />
  </div>
</div>
```

**API Integration:**
```javascript
// src/api.js
discovery: {
  // ... existing
  getVendorDetail: async (idOrSlug) => {
    const response = await apiFetch(`/marketplace/vendors/${idOrSlug}`);
    return response;
  },
}
```

**Verification:**
- Navigate to `/marketplace/openai-proxy`
- All sections load with data
- Chart renders health history
- Responsive on mobile
- Back navigation works

---

## Prompt 4: Connect to Vendor Flow

**Objective:** Implement the "Connect to Vendor" button logic with agent onboarding integration.

**Context:**
When a user clicks "Connect to Vendor":
- **Not logged in:** Redirect to auth, return after
- **Logged in, not agent:** Start agent onboarding with vendor pre-selected
- **Logged in, is agent:** Add vendor to allowlist via API

**Files to Modify:**

1. **`src/pages/marketplace/VendorDetailPage.jsx`:**
```jsx
const handleConnect = async () => {
  if (!isAuthenticated) {
    navigate('/auth', { 
      state: { returnTo: location.pathname } 
    });
    return;
  }
  
  if (!user.isAgent) {
    navigate('/onboarding/agent', {
      state: { 
        preselectedVendor: vendor,
        returnTo: location.pathname
      }
    });
    return;
  }
  
  // User is agent - add to allowlist
  try {
    await api.agents.addVendorToAllowlist(vendor.id);
    toast.success(`Connected to ${vendor.serviceName}!`);
    setIsConnected(true);
  } catch (err) {
    toast.error('Failed to connect: ' + err.message);
  }
};
```

2. **`src/pages/AgentOnboarding.jsx`:**
   - Read `preselectedVendor` from location state
   - Pre-populate in "Allowed Vendors" step
   - Show vendor card in onboarding UI
   - Navigate to `returnTo` after completion

3. **`src/api.js`:**
```javascript
agents: {
  addVendorToAllowlist: async (vendorId) => {
    return apiFetch('/agents/me/allowlist', {
      method: 'POST',
      body: JSON.stringify({ vendorId }),
    });
  },
}
```

4. **Backend: `internal/handlers/sql/agent_handler.go`:**
```go
// POST /v1/agents/me/allowlist
func (h *AgentHandler) HandleAddToAllowlist(ctx context.Context, c *app.RequestContext) {
    // Get current agent from JWT
    // Insert into agent_allowed_vendors
    // Return success
}
```

5. **Register route in `cmd/api/main.go`:**
```go
agentGroup.POST("/me/allowlist", agentHandler.HandleAddToAllowlist)
```

**UI States:**
- Button: "Connect to Vendor" (default)
- Button: "Connecting..." (loading)
- Button: "✓ Connected" (success, disabled)

**Verification:**
1. Logged out user → clicks Connect → Auth page → returns to vendor
2. Non-agent user → clicks Connect → Agent onboarding with vendor shown
3. Agent user → clicks Connect → Success toast → Button shows connected

---

## Prompt 5: Playwright Tests for Vendor Detail

**Objective:** Create E2E tests for the vendor detail page.

**Files to Create:**
- `machpay-console/tests/vendor-detail.spec.js`

**Test Cases:**

```javascript
// 1. Page loads with vendor data
test('vendor detail page loads correctly', async ({ page }) => {
  await page.goto('/marketplace/openai-proxy');
  await expect(page.getByRole('heading', { name: /OpenAI Proxy/i })).toBeVisible();
  await expect(page.getByText('Uptime')).toBeVisible();
  await expect(page.getByText('Connect to Vendor')).toBeVisible();
});

// 2. Metrics display correctly
test('metrics show correct values', async ({ page }) => {
  await page.goto('/marketplace/openai-proxy');
  await expect(page.getByText(/99\.\d+%/)).toBeVisible(); // Uptime
  await expect(page.getByText(/\d+ms/)).toBeVisible(); // Latency
});

// 3. Endpoints table renders
test('endpoints table shows API endpoints', async ({ page }) => {
  await page.goto('/marketplace/openai-proxy');
  await expect(page.getByText('/v1/chat/completions')).toBeVisible();
});

// 4. Navigate from marketplace to detail
test('clicking vendor card navigates to detail', async ({ page }) => {
  await page.goto('/marketplace');
  await page.getByRole('button', { name: /OpenAI Proxy/i }).first().click();
  await expect(page).toHaveURL(/\/marketplace\/openai-proxy/);
});

// 5. Back navigation works
test('back button returns to marketplace', async ({ page }) => {
  await page.goto('/marketplace/openai-proxy');
  await page.getByRole('link', { name: /Back/i }).click();
  await expect(page).toHaveURL('/marketplace');
});

// 6. Connect button for non-agent
test('connect redirects non-agent to onboarding', async ({ page }) => {
  // Setup mock auth as non-agent
  await setupMockAuth(page, { isAgent: false });
  await page.goto('/marketplace/openai-proxy');
  await page.getByRole('button', { name: /Connect/i }).click();
  await expect(page).toHaveURL(/\/onboarding\/agent/);
});
```

---

## Execution Order

| Step | Prompt | Description | Dependencies |
|------|--------|-------------|--------------|
| 1 | Prompt 1 | Seed data with endpoints & contract info | None |
| 2 | Prompt 2 | Backend vendor detail API | Prompt 1 |
| 3 | Prompt 3 | Console vendor detail page | Prompt 2 |
| 4 | Prompt 4 | Connect to vendor flow | Prompt 3 |
| 5 | Prompt 5 | Playwright tests | Prompt 3, 4 |

**Start with:** `Prompt 1` (Seed Data Enhancement)

---

## Verification Checklist

After all prompts are complete:

- [ ] `curl localhost:8081/v1/marketplace/vendors/openai-proxy` returns full detail
- [ ] Navigate to `/marketplace/openai-proxy` shows detail page
- [ ] All sections render (header, metrics, chart, contract, endpoints)
- [ ] Uptime chart shows 30 days of history
- [ ] Endpoints table lists API paths
- [ ] Quick start code is copyable
- [ ] Connect button works for all user states
- [ ] Mobile responsive layout
- [ ] Playwright tests pass

