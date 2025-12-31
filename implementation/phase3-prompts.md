# Phase 3: Console UI — Implementation Prompts

> **Phase**: Console UI  
> **Duration**: 3 days  
> **Dependencies**: Phase 2 Backend API Complete  
> **Focus**: Marketplace page, vendor discovery components, real-time metrics

---

## Role Setup Prompt

```
You are the CTO and Chief Product Architect for MachPay, building the next revolutionary 
payment infrastructure for AI agents. You have deep expertise in:

- React 18+ with hooks, context, and modern patterns
- Tailwind CSS and component-driven design systems
- Framer Motion for production-grade animations
- Real-time data visualization and live metrics
- Bloomberg Terminal / Stripe Dashboard aesthetics
- Accessibility (WCAG 2.1 AA) and responsive design

MISSION: Build a world-class Marketplace UI that enables AI agents to discover, compare, 
and integrate with API vendors. The interface should feel like a professional trading terminal 
meets modern SaaS dashboard.

DESIGN PRINCIPLES:
1. Information density without clutter
2. Real-time metrics front and center
3. Instant search with zero loading states (optimistic)
4. Dark theme with emerald accents (existing theme)
5. Mobile-first but desktop-optimized

EXISTING CODEBASE CONTEXT:
- Console: machpay-console/
- Uses React + Vite + TailwindCSS
- Component library in src/components/
- API client in src/api.js
- Pages in src/pages/
- Theme defined in tailwind.config.js and src/styles/index.css
- Auth context, role-based routing already implemented
```

---

## Task 3.1: API Client Functions

**Goal**: Add marketplace API integration to the frontend client.

**File**: `machpay-console/src/api.js`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/api.js

TASK: Add the `marketplace` namespace with all discovery API functions.

BACKEND ENDPOINTS (already implemented in Phase 2):
- GET /v1/marketplace/vendors - Search vendors with query params
- GET /v1/marketplace/vendors/featured - Get featured vendors
- GET /v1/marketplace/vendors/:id - Get vendor details
- GET /v1/marketplace/vendors/:id/health - Get health check history
- GET /v1/marketplace/categories - Get categories with counts
- GET /v1/marketplace/categories/:category - Get vendors by category
- GET /v1/marketplace/quick-search?q= - Autocomplete search
- POST /v1/marketplace/search - Advanced search with body

FUNCTIONS TO ADD:

1. marketplace.getVendors(params)
   - GET /v1/marketplace/vendors
   - Params: { q, category, sort, min_uptime, min_rating, limit, offset }
   - Returns: { vendors: [], total, hasMore }

2. marketplace.search(body)
   - POST /v1/marketplace/search
   - Body: { query, categories, tags, minUptime, maxLatency, minRating, sortBy, limit, offset }
   - Returns: DiscoverySearchResponse

3. marketplace.quickSearch(query, limit)
   - GET /v1/marketplace/quick-search?q=&limit=
   - Returns: { results: [{id, serviceName, slug, category, rating}] }

4. marketplace.getFeatured(limit = 10)
   - GET /v1/marketplace/vendors/featured?limit=
   - Returns: { vendors: [] }

5. marketplace.getVendorDetails(idOrSlug)
   - GET /v1/marketplace/vendors/:id
   - Returns: VendorDiscoveryResult

6. marketplace.getVendorHealth(vendorId, limit = 24)
   - GET /v1/marketplace/vendors/:id/health?limit=
   - Returns: { checks: [] }

7. marketplace.getCategories()
   - GET /v1/marketplace/categories
   - Returns: { categories: [{category, count}] }

8. marketplace.getVendorsByCategory(category, limit, offset)
   - GET /v1/marketplace/categories/:category
   - Returns: { vendors: [], total, hasMore }

IMPLEMENTATION REQUIREMENTS:
- Follow existing patterns in the file
- Use the apiClient helper for requests
- Handle errors consistently
- No authentication required for these endpoints

VERIFY: Check existing `discovery` namespace if it exists and either update or replace it.
```

---

## Task 3.2: VendorCard Component

**Goal**: Create the main vendor display card with metrics.

**File**: `machpay-console/src/components/marketplace/VendorCard.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/components/marketplace/VendorCard.jsx (NEW)

TASK: Create a VendorCard component that displays vendor info with live metrics.

PROPS:
- vendor: {
    id, appId, slug, serviceName, description, logoUrl, category,
    defaultPriceUsd, isVerified, featuredOrder, tags,
    uptimePct, avgLatencyMs, ratingScore, totalRequests, successRate, uniqueAgents
  }
- onClick: (vendor) => void
- className?: string

VISUAL DESIGN (Bloomberg Terminal meets Stripe):

┌────────────────────────────────────────────────────────┐
│ [Logo]  Service Name              ✓ Verified  ★ 4.9   │
│         Category Badge                                  │
├────────────────────────────────────────────────────────┤
│ Description text goes here, maximum two lines with     │
│ ellipsis overflow...                                   │
├────────────────────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│ │ $0.0001  │ │  99.98%  │ │   42ms   │ │  1.2M    │   │
│ │ per req  │ │  uptime  │ │  latency │ │ requests │   │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
└────────────────────────────────────────────────────────┘

STYLING REQUIREMENTS:
1. Use existing Panel/Card patterns from the codebase
2. Match theme: bg-gray-800/50, border-gray-700, emerald accents
3. Hover state: border-emerald-500/50, slight scale, glow effect
4. Verified badge: emerald checkmark icon
5. Category badge: colored pill (ai=purple, data=blue, weather=cyan, finance=green)
6. Price: bright green, monospace font
7. Uptime: color coded (green >99.5, yellow >99, red <99)
8. Rating: golden star icon + number

METRICS FORMATTING:
- Price: formatPrice(0.0001) → "$0.0001"
- Uptime: formatUptime(99.98) → "99.98%"
- Latency: formatLatency(42) → "42ms"
- Requests: formatNumber(1234567) → "1.2M"

ANIMATIONS (Framer Motion):
- Staggered mount animation (parent controls delay)
- Hover: scale(1.02), borderColor transition
- Click: scale(0.98) momentarily

ICONS (lucide-react):
- Verified: BadgeCheck
- Star: Star (filled)
- Category icons: Cpu (ai), Database (data), Cloud (weather), DollarSign (finance)

ACCESSIBILITY:
- role="article" or role="button"
- aria-label with vendor name
- Keyboard focusable with focus-visible ring

EXPORT: Named export { VendorCard }
```

---

## Task 3.3: VendorDetailModal Component

**Goal**: Full vendor details in a modal overlay.

**File**: `machpay-console/src/components/marketplace/VendorDetailModal.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/components/marketplace/VendorDetailModal.jsx (NEW)

TASK: Create a detailed vendor modal with metrics, endpoints, and integration info.

PROPS:
- vendor: Full vendor object (same as VendorCard + endpoints)
- isOpen: boolean
- onClose: () => void

LAYOUT (Slide-over panel from right):

┌─────────────────────────────────────────────────────────────────┐
│ ← Back to Marketplace                                      [X]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [LOGO]   OpenWeather Pro                    ✓ Verified         │
│           Weather Data API                                       │
│           weather • data                                         │
│                                                                  │
│  Real-time weather data for 200,000+ cities worldwide.          │
│  Includes forecasts, historical data, and alerts.               │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  PERFORMANCE METRICS                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ 99.98%  │ │  42ms   │ │  89ms   │ │  ★ 4.9  │ │  1.2M   │   │
│  │ Uptime  │ │ Avg     │ │ P95     │ │ Rating  │ │Requests │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│                                                                  │
│  UPTIME HISTORY (Last 24h)                                       │
│  ██████████████████████████████████████████ 100%                 │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  PRICING                                                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ $0.0001 / request                    Model: per_request    │  │
│  │ Rate Limit: 1000 req/min             Timeout: 30s          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  ENDPOINTS                                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ GET  /current   Current Weather         $0.0001/req        │  │
│  │ GET  /forecast  5-Day Forecast          $0.0005/req        │  │
│  │ GET  /history   Historical Data         $0.001/req         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  INTEGRATION                                                     │
│  Gateway URL:                                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ https://gateway.machpay.xyz/v/{app_id}              [Copy] │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Example (Python):                                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ from machpay import Agent                                  │  │
│  │ agent = Agent(wallet="...")                                │  │
│  │ response = agent.call("mp_app_xyz", "/current", {...})     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  [Copy Gateway URL]  [View Documentation]  [View on Explorer]   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

SECTIONS TO IMPLEMENT:

1. Header:
   - Back button (closes modal)
   - Close X button
   - Logo (or category icon fallback)
   - Name + verified badge
   - Category + tags

2. Description:
   - Full description text
   - Website link (if available)

3. Metrics Grid:
   - Uptime, Avg Latency, P95 Latency, Rating, Total Requests
   - Color-coded values

4. Uptime History:
   - Fetch health checks via api.marketplace.getVendorHealth()
   - Display as horizontal bar chart or grid of dots
   - Green = up, red = down, yellow = degraded

5. Pricing Card:
   - Price per request (large, green)
   - Pricing model
   - Rate limit (if set)
   - Timeout

6. Endpoints Table:
   - Method badge (GET=green, POST=blue, PUT=yellow, DELETE=red)
   - Path
   - Name/Description
   - Price override (if different from default)

7. Integration Section:
   - Gateway URL (copyable with click-to-copy)
   - Python code snippet
   - cURL example (collapsible)

8. Action Buttons:
   - Copy Gateway URL (with success toast)
   - View Documentation (external link)
   - View on Explorer (internal link)

ANIMATIONS:
- Slide in from right
- Fade backdrop
- Staggered section reveal

ACCESSIBILITY:
- Focus trap when open
- ESC to close
- aria-modal="true"
- Return focus on close

DEPENDENCIES:
- @headlessui/react Dialog (if available) or custom modal
- Prism.js or similar for code highlighting (optional)

EXPORT: Named export { VendorDetailModal }
```

---

## Task 3.4: CategoryPills Component

**Goal**: Horizontal scrollable category filter.

**File**: `machpay-console/src/components/marketplace/CategoryPills.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/components/marketplace/CategoryPills.jsx (NEW)

TASK: Create a horizontal scrollable category filter component.

PROPS:
- categories: [{ category: string, count: number }]
- selected: string (category ID or "all")
- onSelect: (category: string) => void

LAYOUT:

┌──────────────────────────────────────────────────────────────────┐
│ [All (142)] [🤖 AI (45)] [📊 Data (28)] [🌤️ Weather (12)] [💹 Finance (8)] → │
└──────────────────────────────────────────────────────────────────┘

FEATURES:
1. "All" option always first (count = sum of all categories)
2. Each pill shows: Icon + Name + (count)
3. Active pill: emerald background, white text
4. Inactive pill: gray-700 background, gray-300 text
5. Horizontal scroll on overflow (mobile)
6. Scroll shadow indicators on edges
7. Smooth scroll snap to each pill

CATEGORY STYLING:
- all: no icon, default gray
- ai: 🤖 or Cpu icon, purple-500 accent
- data: 📊 or Database icon, blue-500 accent
- weather: 🌤️ or Cloud icon, cyan-500 accent
- finance: 💹 or DollarSign icon, green-500 accent
- other: 📦 or Package icon, gray-500 accent

ANIMATIONS:
- Active state: scale bounce
- Scroll: smooth CSS scroll-behavior

ACCESSIBILITY:
- role="tablist" on container
- role="tab" on each pill
- aria-selected for active
- Keyboard navigation (arrow keys)

EXPORT: Named export { CategoryPills }
```

---

## Task 3.5: SortDropdown Component

**Goal**: Dropdown for sort options.

**File**: `machpay-console/src/components/marketplace/SortDropdown.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/components/marketplace/SortDropdown.jsx (NEW)

TASK: Create a dropdown component for sorting marketplace results.

PROPS:
- value: string (current sort option)
- onChange: (value: string) => void

SORT OPTIONS:
| Value      | Label                    | Icon        |
|------------|--------------------------|-------------|
| rating     | ⭐ Rating (High → Low)   | Star        |
| uptime     | 📈 Uptime (High → Low)   | TrendingUp  |
| popular    | 🔥 Most Popular          | Flame       |
| fastest    | ⚡ Fastest Response      | Zap         |
| price_asc  | 💰 Price (Low → High)    | ArrowUp     |
| price_desc | 💰 Price (High → Low)    | ArrowDown   |
| recent     | 🕐 Recently Active       | Clock       |

STYLING:
- Match existing select/dropdown patterns in codebase
- Dark theme: bg-gray-800, border-gray-700
- Hover: bg-gray-700
- Selected: emerald checkmark
- Monospace font for values
- Min-width to fit longest option

FEATURES:
- Keyboard navigation (up/down arrows)
- Type-ahead search
- Close on outside click
- Close on ESC

USE: Headless UI Listbox if available, otherwise custom implementation

EXPORT: Named export { SortDropdown }
```

---

## Task 3.6: SearchInput Component

**Goal**: Debounced search input with autocomplete.

**File**: `machpay-console/src/components/marketplace/SearchInput.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/components/marketplace/SearchInput.jsx (NEW)

TASK: Create a search input with debouncing and autocomplete suggestions.

PROPS:
- value: string
- onChange: (value: string) => void
- placeholder?: string (default: "Search vendors...")
- debounceMs?: number (default: 300)
- onQuickSelect?: (vendor: QuickSearchResult) => void

FEATURES:
1. Debounced onChange (300ms default)
2. Autocomplete dropdown from quick-search API
3. Clear button when has value
4. Search icon (left)
5. Loading spinner during search
6. Keyboard navigation in dropdown

LAYOUT:
┌─────────────────────────────────────────────────────────────────┐
│ 🔍 Search vendors, categories, or features...           [Clear] │
└─────────────────────────────────────────────────────────────────┘
     │
     ▼ (autocomplete dropdown)
┌─────────────────────────────────────────────────────────────────┐
│ 🤖 OpenAI GPT-4 API          ai     ★ 4.9                       │
│ 🤖 Anthropic Claude           ai     ★ 4.8                       │
│ 🌤️ OpenWeather Pro          weather  ★ 4.9                       │
└─────────────────────────────────────────────────────────────────┘

AUTOCOMPLETE LOGIC:
- Trigger after 2+ characters
- Call api.marketplace.quickSearch(query, 5)
- Show loading state while fetching
- Display: icon + name + category + rating
- Click or Enter selects and closes
- ESC closes dropdown

STYLING:
- Full width
- Dark theme input
- Focus ring emerald
- Dropdown below input
- Smooth open/close animation

ACCESSIBILITY:
- aria-autocomplete="list"
- aria-controls for dropdown
- aria-activedescendant for highlighted option
- Keyboard: up/down to navigate, enter to select, esc to close

EXPORT: Named export { SearchInput }
```

---

## Task 3.7: Marketplace Page

**Goal**: Main marketplace page with search, filters, and vendor grid.

**File**: `machpay-console/src/pages/Marketplace.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/pages/Marketplace.jsx (NEW or UPDATE existing)

TASK: Create the main Marketplace page that orchestrates all components.

PAGE LAYOUT:
┌─────────────────────────────────────────────────────────────────────────────┐
│  🏪 MARKETPLACE                                    142 Vendors • 2.4M Reqs  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 🔍 Search vendors, categories, or features...                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  [All (142)] [🤖 AI (45)] [📊 Data (28)] [🌤️ Weather (12)] [💹 Finance]   │
│                                                                             │
│  Sort: [⭐ Rating (High → Low) ▼]           Showing 1-20 of 142            │
│                                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ VendorCard  │ │ VendorCard  │ │ VendorCard  │ │ VendorCard  │          │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ VendorCard  │ │ VendorCard  │ │ VendorCard  │ │ VendorCard  │          │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                             │
│                        [Load More] or infinite scroll                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

STATE MANAGEMENT:
- searchQuery: string
- selectedCategory: string ("all" default)
- sortBy: string ("rating" default)
- vendors: VendorDiscoveryResult[]
- total: number
- loading: boolean
- error: Error | null
- hasMore: boolean
- offset: number
- selectedVendor: VendorDiscoveryResult | null (for modal)
- categories: CategoryCount[]

EFFECTS:
1. useEffect on mount:
   - Fetch categories
   - Fetch initial vendors (no filters)

2. useEffect on [searchQuery, selectedCategory, sortBy]:
   - Reset offset to 0
   - Fetch vendors with current filters
   - Debounce search query

3. loadMore function:
   - Increment offset
   - Append to vendors array

DATA FETCHING:
- Use api.marketplace.search() for POST search
- Or api.marketplace.getVendors() for GET with params
- Handle loading and error states

URL SYNC (Optional but recommended):
- Sync searchQuery, category, sort to URL params
- useSearchParams hook
- Enable shareable search URLs

LOADING STATES:
- Initial load: Full-page skeleton grid
- Search: Inline loading indicator
- Load more: Button shows spinner

ERROR HANDLING:
- Display error message with retry button
- Toast for transient errors

EMPTY STATE:
- "No vendors found matching your criteria"
- Suggestions to broaden search
- Link to clear filters

RESPONSIVE:
- Desktop: 4 columns
- Tablet: 3 columns
- Mobile: 1-2 columns

ANIMATIONS:
- Staggered card entrance (Framer Motion)
- Smooth filter transitions
- Modal slide-in

ACCESSIBILITY:
- Skip link to main content
- Announce result count changes
- aria-live for dynamic updates

EXPORT: Named export { Marketplace }
```

---

## Task 3.8: Route Registration

**Goal**: Add marketplace route to App.jsx.

**File**: `machpay-console/src/App.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/App.jsx

TASK: Register the Marketplace route in the application router.

CHANGES:
1. Import: import { Marketplace } from './pages/Marketplace'

2. Add route inside the authenticated routes section:
   <Route path="/marketplace" element={<RoleLayout />}>
     <Route index element={<Marketplace />} />
   </Route>

NOTES:
- Place after /explorer or similar pages
- Use RoleLayout for consistent sidebar
- All roles should have access (observer, architect, operator)
- Route should not require auth for marketplace viewing (it's public discovery)
  - But keep inside auth layout for consistent UI

VERIFY: Check if route already exists and update if needed.
```

---

## Task 3.9: Sidebar Navigation

**Goal**: Add Marketplace to sidebar navigation.

**File**: `machpay-console/src/components/Layout.jsx`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/components/Layout.jsx

TASK: Add Marketplace item to the sidebar navigation.

CHANGES:
1. Import: import { Store } from 'lucide-react'

2. Add to baseNavItems array (after Explorer or at appropriate position):
   {
     id: 'marketplace',
     label: 'Marketplace',
     icon: Store,
     path: '/marketplace',
     roles: ['observer', 'architect', 'operator']
   }

POSITION:
- After "Explorer" if it exists
- Before "Settings" or admin sections
- In the main navigation group

VERIFY: Check existing nav items and place appropriately.
```

---

## Task 3.10: Component Index Export

**Goal**: Create barrel export for marketplace components.

**File**: `machpay-console/src/components/marketplace/index.js`

**Prompt**:
```
CONTEXT: Phase 3 Console UI for MachPay Marketplace.
FILE: machpay-console/src/components/marketplace/index.js (NEW)

TASK: Create a barrel export file for all marketplace components.

CONTENT:
export { VendorCard } from './VendorCard';
export { VendorDetailModal } from './VendorDetailModal';
export { CategoryPills } from './CategoryPills';
export { SortDropdown } from './SortDropdown';
export { SearchInput } from './SearchInput';

PURPOSE:
- Clean imports in Marketplace.jsx
- Single import point for all marketplace components
- Enables: import { VendorCard, CategoryPills } from '@/components/marketplace'
```

---

## Verification Prompt

After completing all tasks, use this prompt to verify:

```
VERIFICATION CHECKLIST for Phase 3:

Run through each item and confirm implementation:

1. API Client (api.js):
   [ ] marketplace.getVendors() works
   [ ] marketplace.search() works
   [ ] marketplace.quickSearch() works
   [ ] marketplace.getFeatured() works
   [ ] marketplace.getVendorDetails() works
   [ ] marketplace.getVendorHealth() works
   [ ] marketplace.getCategories() works

2. Components Created:
   [ ] VendorCard.jsx exists and exports correctly
   [ ] VendorDetailModal.jsx exists and exports correctly
   [ ] CategoryPills.jsx exists and exports correctly
   [ ] SortDropdown.jsx exists and exports correctly
   [ ] SearchInput.jsx exists and exports correctly
   [ ] index.js barrel export works

3. Marketplace Page:
   [ ] Page renders without errors
   [ ] Search input works with debouncing
   [ ] Category filters work
   [ ] Sort dropdown works
   [ ] Vendor cards display correctly
   [ ] Load more/pagination works
   [ ] Detail modal opens on card click
   [ ] Empty state displays properly
   [ ] Loading skeletons show during fetch
   [ ] Error state with retry button works

4. Routing:
   [ ] /marketplace route accessible
   [ ] Sidebar shows Marketplace item
   [ ] All roles can access

5. Styling:
   [ ] Matches existing theme (dark mode, emerald accents)
   [ ] Responsive on mobile/tablet/desktop
   [ ] Animations smooth and not jarring

6. Accessibility:
   [ ] Keyboard navigation works
   [ ] Screen reader announces changes
   [ ] Focus management in modal

BUILD TEST:
npm run build (should complete without errors)

DEV TEST:
npm run dev
Navigate to /marketplace
Test all interactions
```

---

## Progress Tracking

| Task | Description | Status |
|------|-------------|--------|
| 3.1 | API Client Functions | ⬜ Pending |
| 3.2 | VendorCard Component | ⬜ Pending |
| 3.3 | VendorDetailModal Component | ⬜ Pending |
| 3.4 | CategoryPills Component | ⬜ Pending |
| 3.5 | SortDropdown Component | ⬜ Pending |
| 3.6 | SearchInput Component | ⬜ Pending |
| 3.7 | Marketplace Page | ⬜ Pending |
| 3.8 | Route Registration | ⬜ Pending |
| 3.9 | Sidebar Navigation | ⬜ Pending |
| 3.10 | Component Index Export | ⬜ Pending |
| ✓ | Verification | ⬜ Pending |

---

## Notes

- Check if any marketplace components already exist before creating new ones
- Follow existing code patterns in the console codebase
- Use existing UI components (Panel, Button, etc.) where available
- Test on multiple screen sizes
- Ensure dark theme consistency

