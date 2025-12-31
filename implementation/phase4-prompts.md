# Phase 4: SDK Integration — Implementation Prompts

> **Phase**: Python SDK Discovery Module  
> **Duration**: 1 day  
> **Dependencies**: Phase 2 Backend API Complete  
> **Focus**: Python SDK discovery client for programmatic vendor search

---

## Role Setup Prompt

```
You are the CTO and Chief Product Architect for MachPay, building the next revolutionary 
payment infrastructure for AI agents. You have deep expertise in:

- Python 3.10+ with modern typing (dataclasses, TypedDict, Protocols)
- HTTP client design (requests, httpx, aiohttp)
- SDK/Library design patterns
- API client best practices
- Clean, Pythonic code with comprehensive docstrings

MISSION: Extend the MachPay Python SDK with a DiscoveryClient that enables AI agents 
to programmatically search, discover, and evaluate API vendors before making calls.

USE CASE: An AI agent can use the SDK to:
1. Search for vendors matching a capability ("I need a weather API")
2. Compare vendors by uptime, latency, price, rating
3. Select the best vendor programmatically
4. Make calls through the selected vendor

EXISTING CODEBASE CONTEXT:
- SDK: machpay-py/
- Main client: src/machpay/client.py
- Uses dataclasses for models
- Uses requests for HTTP
- Follows existing patterns in the codebase
```

---

## Task 4.1: Discovery Models

**Goal**: Define dataclasses for discovery API responses.

**File**: `machpay-py/src/machpay/discovery.py`

**Prompt**:
```
CONTEXT: Phase 4 SDK Integration for MachPay Marketplace.
FILE: machpay-py/src/machpay/discovery.py (NEW)

TASK: Create the discovery module with dataclasses and DiscoveryClient.

DATACLASSES TO DEFINE:

1. VendorMetrics:
   - uptime_pct: float          # 0-100 percentage
   - avg_latency_ms: int        # Average response time
   - p95_latency_ms: int        # 95th percentile
   - p99_latency_ms: int        # 99th percentile  
   - rating_score: float        # 0.0-5.0 stars
   - total_requests: int        # Lifetime request count
   - success_rate: float        # 0-100 percentage
   - unique_agents: int         # Distinct agents served
   - last_request_at: Optional[datetime]

2. VendorPricing:
   - model: str                 # "per_request", "per_token", etc.
   - price_usd: float           # Price per unit
   - currency: str              # "USDC"
   - rate_limit: Optional[int]  # Requests per minute
   - timeout_ms: int            # Request timeout

3. VendorEndpoint:
   - path: str                  # "/current"
   - method: str                # "GET", "POST"
   - name: str                  # "Get Current Weather"
   - description: Optional[str]
   - price_usd: Optional[float] # Override price (None = use default)

4. VendorInfo:
   - id: int
   - app_id: str                # "mp_app_xyz123"
   - slug: str                  # "openweather-pro"
   - name: str                  # "OpenWeather Pro"
   - description: str
   - category: str              # "weather", "ai", "data"
   - logo_url: Optional[str]
   - website_url: Optional[str]
   - is_verified: bool
   - tags: List[str]
   - pricing: VendorPricing
   - metrics: VendorMetrics
   - endpoints: List[VendorEndpoint]
   - base_url: str              # For direct reference

5. CategoryInfo:
   - category: str              # "ai"
   - count: int                 # Number of vendors

6. SearchResult:
   - vendors: List[VendorInfo]
   - total: int
   - limit: int
   - offset: int
   - has_more: bool
   - categories: List[CategoryInfo]  # Faceted counts

7. QuickSearchResult:
   - id: int
   - service_name: str
   - slug: str
   - category: str
   - rating: float

HELPER METHODS:

VendorInfo:
  - gateway_url() -> str: Returns "https://gateway.machpay.xyz/v/{app_id}"
  - best_price() -> float: Returns lowest endpoint price or default
  - to_dict() -> dict: Serialize to dict

SearchResult:
  - top_by_rating(n: int) -> List[VendorInfo]: Top N by rating
  - top_by_uptime(n: int) -> List[VendorInfo]: Top N by uptime
  - cheapest(n: int) -> List[VendorInfo]: Top N by lowest price
  - fastest(n: int) -> List[VendorInfo]: Top N by lowest latency

IMPLEMENTATION NOTES:
- Use @dataclass decorator
- Use Optional[] for nullable fields
- Use List[] for arrays
- Provide from_dict() class method for each dataclass
- Handle snake_case to camelCase conversion from API
```

---

## Task 4.2: DiscoveryClient Class

**Goal**: HTTP client for discovery endpoints.

**File**: `machpay-py/src/machpay/discovery.py` (continued)

**Prompt**:
```
CONTEXT: Phase 4 SDK Integration for MachPay Marketplace.
FILE: machpay-py/src/machpay/discovery.py (continue from Task 4.1)

TASK: Implement the DiscoveryClient class.

CLASS DEFINITION:

class DiscoveryClient:
    """
    Client for MachPay Marketplace vendor discovery.
    
    Enables AI agents to search, compare, and select API vendors
    based on real-time performance metrics.
    
    Example:
        >>> from machpay import MachPayClient
        >>> client = MachPayClient()
        >>> 
        >>> # Search for weather APIs
        >>> results = client.discovery.search(query="weather", min_uptime=99.0)
        >>> 
        >>> # Get the top-rated vendor
        >>> best = results.top_by_rating(1)[0]
        >>> print(f"{best.name}: {best.metrics.uptime_pct}% uptime")
        >>> 
        >>> # Make a call through the vendor
        >>> agent.call(best.app_id, "/current", {"city": "NYC"})
    """

CONSTRUCTOR:
    def __init__(
        self,
        base_url: str = "https://api.machpay.xyz",
        api_key: Optional[str] = None,
        timeout: float = 30.0
    ):
        """
        Initialize DiscoveryClient.
        
        Args:
            base_url: MachPay API base URL
            api_key: Optional API key (not required for public discovery)
            timeout: Request timeout in seconds
        """

METHODS TO IMPLEMENT:

1. search(
       query: str = "",
       categories: Optional[List[str]] = None,
       tags: Optional[List[str]] = None,
       min_uptime: Optional[float] = None,
       max_latency: Optional[int] = None,
       min_rating: Optional[float] = None,
       sort_by: str = "rating",
       limit: int = 20,
       offset: int = 0
   ) -> SearchResult:
   """
   Search for vendors with filters.
   
   Args:
       query: Natural language search query
       categories: Filter by categories (e.g., ["ai", "weather"])
       tags: Filter by tags (e.g., ["enterprise", "fast"])
       min_uptime: Minimum uptime percentage (0-100)
       max_latency: Maximum average latency in ms
       min_rating: Minimum rating (0-5)
       sort_by: Sort option: rating, uptime, popular, fastest, recent
       limit: Results per page (max 100)
       offset: Pagination offset
   
   Returns:
       SearchResult with vendors and pagination info
   
   Raises:
       MachPayError: On API error
   """

2. get_featured(limit: int = 10) -> List[VendorInfo]:
   """Get curated featured vendors."""

3. get_vendor(vendor_id: Union[int, str]) -> VendorInfo:
   """Get vendor by ID or slug."""

4. get_vendor_health(vendor_id: int, limit: int = 24) -> List[HealthCheck]:
   """Get recent health check history."""

5. get_categories() -> List[CategoryInfo]:
   """Get all categories with vendor counts."""

6. get_vendors_by_category(
       category: str,
       limit: int = 20,
       offset: int = 0
   ) -> SearchResult:
   """Get vendors in a specific category."""

7. quick_search(query: str, limit: int = 5) -> List[QuickSearchResult]:
   """Fast search for autocomplete suggestions."""

INTERNAL METHODS:

- _request(method: str, path: str, **kwargs) -> dict:
  """Make HTTP request with error handling."""

- _build_url(path: str) -> str:
  """Build full API URL."""

- _parse_vendor(data: dict) -> VendorInfo:
  """Parse vendor from API response."""

ERROR HANDLING:
- Raise MachPayError for API errors
- Include status code and message
- Handle network timeouts

HTTP DETAILS:
- Use requests library
- Set User-Agent header
- Handle JSON responses
- Timeout handling
```

---

## Task 4.3: Integration with Main Client

**Goal**: Add discovery property to MachPayClient.

**File**: `machpay-py/src/machpay/client.py`

**Prompt**:
```
CONTEXT: Phase 4 SDK Integration for MachPay Marketplace.
FILE: machpay-py/src/machpay/client.py

TASK: Add lazy discovery property to the main client.

CHANGES:

1. Import at top of file:
   from .discovery import DiscoveryClient

2. In __init__, initialize placeholder:
   self._discovery: Optional[DiscoveryClient] = None

3. Add property:
   @property
   def discovery(self) -> DiscoveryClient:
       """
       Access the marketplace discovery client.
       
       Example:
           >>> client = MachPayClient()
           >>> vendors = client.discovery.search("weather API")
           >>> for v in vendors.vendors:
           ...     print(f"{v.name}: ${v.pricing.price_usd}/req")
       
       Returns:
           DiscoveryClient instance
       """
       if self._discovery is None:
           self._discovery = DiscoveryClient(
               base_url=self._base_url,
               api_key=self._api_key,
               timeout=self._timeout
           )
       return self._discovery

NOTES:
- Lazy initialization (created on first access)
- Shares configuration with main client
- No additional setup required
```

---

## Task 4.4: Package Exports

**Goal**: Export discovery classes from package.

**File**: `machpay-py/src/machpay/__init__.py`

**Prompt**:
```
CONTEXT: Phase 4 SDK Integration for MachPay Marketplace.
FILE: machpay-py/src/machpay/__init__.py

TASK: Add discovery module exports.

CHANGES:

Add imports:
from .discovery import (
    DiscoveryClient,
    VendorInfo,
    VendorMetrics,
    VendorPricing,
    VendorEndpoint,
    SearchResult,
    CategoryInfo,
    QuickSearchResult,
)

Update __all__ list to include new exports.

VERIFY: Check existing exports and add alongside them.
```

---

## Task 4.5: Usage Examples

**Goal**: Add discovery examples to README.

**File**: `machpay-py/README.md`

**Prompt**:
```
CONTEXT: Phase 4 SDK Integration for MachPay Marketplace.
FILE: machpay-py/README.md

TASK: Add discovery usage examples to the README.

SECTION TO ADD (after existing usage examples):

## Vendor Discovery

Search and discover API vendors in the MachPay marketplace:

```python
from machpay import MachPayClient

client = MachPayClient()

# Search for weather APIs with high uptime
results = client.discovery.search(
    query="weather forecast",
    categories=["weather"],
    min_uptime=99.5,
    sort_by="rating"
)

print(f"Found {results.total} vendors")

for vendor in results.vendors:
    print(f"\n{vendor.name}")
    print(f"  Category: {vendor.category}")
    print(f"  Price: ${vendor.pricing.price_usd}/request")
    print(f"  Uptime: {vendor.metrics.uptime_pct}%")
    print(f"  Rating: {vendor.metrics.rating_score}/5")
    print(f"  Latency: {vendor.metrics.avg_latency_ms}ms")
    print(f"  Gateway: {vendor.gateway_url()}")

# Get featured vendors
featured = client.discovery.get_featured(limit=5)

# Get vendor by ID or slug
vendor = client.discovery.get_vendor("openweather-pro")

# Get all categories
categories = client.discovery.get_categories()
for cat in categories:
    print(f"{cat.category}: {cat.count} vendors")

# Find the cheapest vendor
cheapest = results.cheapest(1)[0]
print(f"Cheapest: {cheapest.name} at ${cheapest.best_price()}/req")

# Find the fastest vendor
fastest = results.fastest(1)[0]
print(f"Fastest: {fastest.name} at {fastest.metrics.avg_latency_ms}ms")
```

### Smart Vendor Selection

Programmatically select the best vendor for your use case:

```python
# Search with strict requirements
results = client.discovery.search(
    query="LLM API",
    min_uptime=99.9,
    min_rating=4.5,
    max_latency=100,
    sort_by="popular"
)

if not results.vendors:
    raise Exception("No vendors match requirements")

# Select best vendor
best = results.vendors[0]

# Use with Agent
from machpay import Agent

agent = Agent(wallet="...")
response = agent.call(
    app_id=best.app_id,
    endpoint="/completions",
    payload={"prompt": "Hello, world!"}
)
```
```

---

## Task 4.6: Unit Tests

**Goal**: Create tests for discovery module.

**File**: `machpay-py/tests/test_discovery.py`

**Prompt**:
```
CONTEXT: Phase 4 SDK Integration for MachPay Marketplace.
FILE: machpay-py/tests/test_discovery.py (NEW)

TASK: Create unit tests for the discovery module.

TEST CASES:

1. test_vendor_info_from_dict():
   - Parse vendor from API response dict
   - Verify all fields populated correctly
   - Handle optional fields (None)

2. test_search_result_helpers():
   - Test top_by_rating()
   - Test top_by_uptime()
   - Test cheapest()
   - Test fastest()

3. test_vendor_gateway_url():
   - Verify gateway URL format
   - "https://gateway.machpay.xyz/v/{app_id}"

4. test_vendor_best_price():
   - Returns lowest endpoint price
   - Falls back to default price

5. test_search_with_filters():
   - Mock API response
   - Verify filters passed correctly
   - Verify response parsed

6. test_get_featured():
   - Mock API response
   - Verify limit parameter

7. test_get_categories():
   - Mock API response
   - Verify parsing

8. test_error_handling():
   - Mock 404 response
   - Verify MachPayError raised
   - Mock network timeout
   - Verify timeout error

9. test_quick_search():
   - Mock API response
   - Verify result format

TESTING APPROACH:
- Use pytest
- Mock HTTP requests with responses library or unittest.mock
- Test both success and error cases
- Test edge cases (empty results, missing fields)

FIXTURES:
- sample_vendor_dict: Complete vendor response
- sample_search_response: Search API response
- sample_categories: Categories response
```

---

## Verification Prompt

After completing all tasks, use this prompt to verify:

```
VERIFICATION CHECKLIST for Phase 4:

Run through each item and confirm implementation:

1. Dataclasses (discovery.py):
   [ ] VendorMetrics defined with all fields
   [ ] VendorPricing defined with all fields
   [ ] VendorEndpoint defined with all fields
   [ ] VendorInfo defined with helper methods
   [ ] CategoryInfo defined
   [ ] SearchResult defined with helper methods
   [ ] QuickSearchResult defined
   [ ] All have from_dict() class methods

2. DiscoveryClient:
   [ ] Constructor accepts base_url, api_key, timeout
   [ ] search() method works with all parameters
   [ ] get_featured() method works
   [ ] get_vendor() method works with ID and slug
   [ ] get_vendor_health() method works
   [ ] get_categories() method works
   [ ] get_vendors_by_category() method works
   [ ] quick_search() method works
   [ ] Error handling raises MachPayError

3. Integration:
   [ ] MachPayClient has discovery property
   [ ] Lazy initialization works
   [ ] Shares base_url and api_key

4. Exports:
   [ ] All classes exported from __init__.py
   [ ] Import works: from machpay import DiscoveryClient

5. Documentation:
   [ ] README has discovery examples
   [ ] Docstrings on all public methods

6. Tests:
   [ ] test_discovery.py exists
   [ ] All test cases pass
   [ ] pytest runs successfully

BUILD TEST:
pip install -e .
python -c "from machpay import DiscoveryClient; print('OK')"

INTEGRATION TEST (requires running backend):
python -c "
from machpay import MachPayClient
client = MachPayClient(base_url='http://localhost:8080')
results = client.discovery.search(query='weather')
print(f'Found {results.total} vendors')
"
```

---

## Progress Tracking

| Task | Description | Status |
|------|-------------|--------|
| 4.1 | Discovery Models (dataclasses) | ✅ Complete |
| 4.2 | DiscoveryClient Class | ✅ Complete |
| 4.3 | Integration with Main Client | ✅ Complete |
| 4.4 | Package Exports | ✅ Complete |
| 4.5 | Usage Examples (README) | ✅ Complete |
| 4.6 | Unit Tests | ✅ Complete |
| ✓ | Verification | ✅ Complete |

