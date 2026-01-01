# Phase 1: CLI Foundation - Implementation Prompts

> **Status:** READY FOR IMPLEMENTATION  
> **Duration:** Week 1  
> **Priority:** P0 - Critical Path

---

## Role Setup (Include in Every Prompt)

```
You are the CTO and principal engineer of MachPay, a revolutionary payment protocol for AI agents. You have 15+ years of experience building developer tools, CLIs, and distributed systems at companies like Stripe, Vercel, and Cloudflare.

Your engineering principles:
1. NO TECH DEBT - Every line of code must be production-ready
2. NO DEAD CODE - Remove unused imports, functions, and files immediately
3. NO MAGIC - Explicit is better than implicit
4. NO SHORTCUTS - If it's worth doing, it's worth doing right
5. SHIP QUALITY - We'd rather delay than ship broken code

You are building a world-class CLI experience that will define how developers interact with MachPay. This is make-or-break for the company.
```

---

## Prompt 1.1: Create machpay-cli Repository

### Context
We need to create a new Go CLI repository that will serve as the lightweight orchestrator for MachPay. This CLI will handle authentication, setup wizards, and gateway management.

### Task
Initialize the `machpay-cli` repository with proper Go project structure, dependencies, and foundational code.

### Requirements

**Directory Structure:**
```
machpay-cli/
├── cmd/machpay/main.go          # Entry point
├── internal/
│   ├── cmd/                     # Cobra commands
│   │   └── root.go              # Root command + global flags
│   ├── config/                  # Configuration management
│   │   ├── config.go            # Viper loader
│   │   └── paths.go             # ~/.machpay paths
│   └── version/                 # Version info
│       └── version.go           # Build-time version
├── go.mod
├── go.sum
├── .gitignore
├── LICENSE (MIT)
└── README.md
```

**Dependencies (go.mod):**
- `github.com/spf13/cobra` v1.8+ - CLI framework
- `github.com/spf13/viper` v1.18+ - Config management
- `go.uber.org/zap` v1.26+ - Structured logging

**Root Command Requirements:**
- Binary name: `machpay`
- Global flags: `--debug`, `--config <path>`
- Default config path: `~/.machpay/config.yaml`
- Version subcommand that prints version, commit, build date
- Clean help text with examples

**Config Paths (internal/config/paths.go):**
- `ConfigDir()` → `~/.machpay/`
- `ConfigFile()` → `~/.machpay/config.yaml`
- `WalletFile()` → `~/.machpay/wallet.json`
- `BinDir()` → `~/.machpay/bin/`
- `LogDir()` → `~/.machpay/logs/`
- Create directories if they don't exist

**README.md:**
- Project overview
- Installation instructions (placeholder)
- Quick start guide (placeholder)
- Link to docs

### Acceptance Criteria
- [ ] `go build ./cmd/machpay` compiles without errors
- [ ] `./machpay --help` shows clean help text
- [ ] `./machpay version` prints version info
- [ ] `./machpay --debug version` enables debug logging
- [ ] Config directories are created on first run
- [ ] No unused imports or dead code
- [ ] All exported functions have doc comments

### Anti-patterns to Avoid
- Don't create placeholder commands that do nothing
- Don't add dependencies we won't use immediately
- Don't create empty files "for later"
- Don't use init() functions - explicit initialization only

---

## Prompt 1.2: Backend Device Auth - Database Schema

### Context
We need to implement OAuth 2.0 Device Authorization Grant (RFC 8628) in the MachPay backend. This allows CLI users to authenticate by approving a code in their browser.

### Task
Create the database migration for device authentication requests.

### Requirements

**New Table: `device_auth_requests`**

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGSERIAL | PRIMARY KEY | Auto-increment ID |
| device_code | VARCHAR(64) | UNIQUE NOT NULL | Opaque code for CLI polling |
| user_code | VARCHAR(16) | UNIQUE NOT NULL | Human-readable code (e.g., "XK9-7NP") |
| client_id | VARCHAR(64) | NOT NULL | Always "machpay-cli" for now |
| scope | VARCHAR(256) | | Requested scopes (e.g., "agent vendor") |
| user_id | BIGINT | FK → users(id), NULL | Set when user approves |
| status | VARCHAR(20) | DEFAULT 'pending' | pending, approved, denied, expired |
| expires_at | TIMESTAMPTZ | NOT NULL | When this request expires (15 min) |
| approved_at | TIMESTAMPTZ | | When user clicked approve |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | |

**Indexes:**
- `idx_device_auth_device_code` on `device_code` (unique lookups)
- `idx_device_auth_user_code` on `user_code` (user input lookups)
- `idx_device_auth_expires` on `expires_at` (cleanup job)
- `idx_device_auth_status` on `status` WHERE status = 'pending'

**Migration Location:**
- `internal/db/sql/schema.sql` - Add to existing schema
- Add cleanup function to delete expired requests (>24 hours old)

### Acceptance Criteria
- [ ] Table created with all columns and constraints
- [ ] All indexes created
- [ ] Foreign key to users table works
- [ ] Migration is idempotent (safe to run multiple times)
- [ ] Cleanup function exists and works

### Notes
- User code format: 3 alphanumeric + hyphen + 3 alphanumeric (e.g., "XK9-7NP")
- Device code: 32-byte random hex string
- Expiration: 15 minutes from creation
- Only one pending request per device_code at a time

---

## Prompt 1.3: Backend Device Auth - Repository Layer

### Context
We need a repository to manage device authentication requests in the database.

### Task
Create `internal/repository/device_auth_repo.go` with all necessary database operations.

### Requirements

**Repository Interface:**

```
DeviceAuthRepository interface {
    // Create new device auth request, returns device_code and user_code
    Create(ctx, clientID, scope string) (*DeviceAuthRequest, error)
    
    // Get by device code (for CLI polling)
    GetByDeviceCode(ctx, deviceCode string) (*DeviceAuthRequest, error)
    
    // Get by user code (for web approval)
    GetByUserCode(ctx, userCode string) (*DeviceAuthRequest, error)
    
    // Approve request (set user_id and status)
    Approve(ctx, userCode string, userID int64) error
    
    // Deny request
    Deny(ctx, userCode string) error
    
    // Delete expired requests (cleanup job)
    DeleteExpired(ctx) (int64, error)
}
```

**Model (internal/models/sql/device_auth.go):**

```
DeviceAuthRequest struct {
    ID          int64
    DeviceCode  string
    UserCode    string
    ClientID    string
    Scope       string
    UserID      *int64     // nil until approved
    Status      string     // pending, approved, denied, expired
    ExpiresAt   time.Time
    ApprovedAt  *time.Time
    CreatedAt   time.Time
}
```

**User Code Generation:**
- Format: `XXX-XXX` where X is alphanumeric (excluding confusing chars: 0, O, I, L, 1)
- Use charset: `23456789ABCDEFGHJKMNPQRSTUVWXYZ`
- Retry if collision (up to 3 times)

**Device Code Generation:**
- 32 bytes of crypto/rand, hex encoded (64 chars)

### Acceptance Criteria
- [ ] All repository methods implemented
- [ ] Proper error handling (ErrNotFound, ErrExpired, ErrAlreadyApproved)
- [ ] User code generation is collision-resistant
- [ ] Device code is cryptographically random
- [ ] SQL queries use parameterized statements (no injection)
- [ ] Context timeout respected
- [ ] Unit tests for user code generation

### Anti-patterns to Avoid
- Don't use string concatenation for SQL
- Don't ignore context cancellation
- Don't return nil error with nil result
- Don't log sensitive data (device codes)

---

## Prompt 1.4: Backend Device Auth - API Handlers

### Context
We need three API endpoints for the device authorization flow.

### Task
Create handlers in `internal/handlers/sql/device_auth_handler.go`.

### Requirements

**Endpoint 1: POST /v1/auth/device/init**

Initialize a new device authorization request.

Request:
```json
{
  "client_id": "machpay-cli",
  "scope": "agent vendor"
}
```

Response (200 OK):
```json
{
  "device_code": "abc123...",
  "user_code": "XK9-7NP",
  "verification_uri": "https://console.machpay.xyz/device",
  "verification_uri_complete": "https://console.machpay.xyz/device?code=XK9-7NP",
  "expires_in": 900,
  "interval": 5
}
```

Validation:
- client_id must be "machpay-cli" (for now, we'll expand later)
- scope is optional, defaults to "agent vendor"

---

**Endpoint 2: POST /v1/auth/device/token**

Poll for token (CLI calls this every 5 seconds).

Request:
```json
{
  "client_id": "machpay-cli",
  "device_code": "abc123...",
  "grant_type": "urn:ietf:params:oauth:grant-type:device_code"
}
```

Response (400 - pending):
```json
{
  "error": "authorization_pending",
  "error_description": "The user has not yet approved this request"
}
```

Response (400 - slow down):
```json
{
  "error": "slow_down",
  "error_description": "Polling too frequently"
}
```

Response (400 - expired):
```json
{
  "error": "expired_token",
  "error_description": "The device code has expired"
}
```

Response (400 - denied):
```json
{
  "error": "access_denied",
  "error_description": "The user denied the request"
}
```

Response (200 - approved):
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "id": 123,
    "email": "user@example.com",
    "displayName": "John Doe",
    "isAgent": true,
    "isVendor": false
  }
}
```

Rate Limiting:
- Track last poll time per device_code
- If polled within `interval` seconds, return "slow_down"
- Use in-memory cache or Redis for rate tracking

---

**Endpoint 3: POST /v1/auth/device/approve**

Called by web console when user clicks "Authorize".

Request (requires JWT auth):
```json
{
  "user_code": "XK9-7NP"
}
```

Response (200 OK):
```json
{
  "success": true,
  "message": "CLI has been authorized"
}
```

Response (404):
```json
{
  "error": "invalid_code",
  "message": "Invalid or expired code"
}
```

Response (409):
```json
{
  "error": "already_used",
  "message": "This code has already been used"
}
```

Authentication:
- Requires valid JWT token
- User ID extracted from JWT is used for approval

### Route Registration

Add to main.go route setup:
```
POST /v1/auth/device/init   → HandleDeviceInit (public)
POST /v1/auth/device/token  → HandleDeviceToken (public)
POST /v1/auth/device/approve → HandleDeviceApprove (requires auth)
```

### Acceptance Criteria
- [ ] All three endpoints implemented
- [ ] Proper HTTP status codes per OAuth spec
- [ ] Rate limiting on token polling
- [ ] JWT validation on approve endpoint
- [ ] Swagger/OpenAPI annotations
- [ ] Integration test for full flow
- [ ] No sensitive data in logs

### Security Considerations
- Device codes must not be guessable
- User codes are short but expire quickly
- Rate limit polling to prevent brute force
- Approve endpoint requires authentication
- Don't reveal if a code exists (timing attacks)

---

## Prompt 1.5: Backend Device Auth - Cleanup Job

### Context
Expired device auth requests should be cleaned up periodically to prevent database bloat.

### Task
Add a cleanup task to the worker that deletes expired device auth requests.

### Requirements

**Cleanup Logic:**
- Run every hour
- Delete requests where:
  - `expires_at < NOW() - INTERVAL '1 hour'` (expired + grace period)
  - OR `status IN ('approved', 'denied')` AND `created_at < NOW() - INTERVAL '24 hours'`
- Log number of deleted records

**Integration:**
- Add to existing worker task system
- Use existing worker infrastructure (don't create new binary)
- Add configuration for cleanup interval

**Configuration (config.go):**
```
DeviceAuthCleanupInterval: 1 hour (default)
DeviceAuthRetentionHours: 24 (for completed requests)
```

### Acceptance Criteria
- [ ] Cleanup job runs on schedule
- [ ] Deletes expired requests
- [ ] Deletes old completed requests
- [ ] Logs cleanup statistics
- [ ] Doesn't block other workers
- [ ] Handles database errors gracefully

---

## Prompt 1.6: Console - Device Approval Page

### Context
Users need a web page where they can approve CLI login requests by entering the user code shown in their terminal.

### Task
Create the `/device` page in machpay-console.

### Requirements

**Route:**
- `/device` - Manual code entry
- `/device?code=XK9-7NP` - Pre-filled code (auto-verify)

**Page States:**

1. **Input State** (default)
   - Header: "Link MachPay CLI"
   - Subtext: "Enter the code shown in your terminal"
   - Code input: 6-character input with auto-uppercase and hyphen formatting
   - "Authorize" button (disabled until valid format)
   - Link to "What is this?" help

2. **Verifying State**
   - Spinner animation
   - Text: "Verifying code..."

3. **Success State**
   - Green checkmark icon
   - Header: "CLI Linked!"
   - Subtext: "You can close this window and return to your terminal."
   - Confetti animation (subtle)

4. **Error State**
   - Red X icon
   - Header: "Verification Failed"
   - Error message (from API)
   - "Try Again" button

**Behavior:**
- If `?code=` query param exists and user is logged in → auto-verify
- If `?code=` exists but user not logged in → redirect to login, then back
- Code input accepts: letters and numbers, formats to `XXX-XXX`
- Auto-submit when 6 chars entered (after hyphen stripped)

**API Integration:**
- Add to `src/api.js`:
  ```
  api.auth.verifyDeviceCode(userCode) → POST /v1/auth/device/approve
  ```

**Styling:**
- Match existing app theme (use Panel, Button components)
- Centered card layout
- Mobile responsive

### Acceptance Criteria
- [ ] Page accessible at `/device`
- [ ] Code input with proper formatting
- [ ] Auto-verify with query param
- [ ] Redirect to login if needed
- [ ] All states render correctly
- [ ] Error handling for all API errors
- [ ] Mobile responsive
- [ ] Matches app design system
- [ ] No console errors

### Files to Create/Modify
- `src/pages/DeviceAuth.jsx` - New page component
- `src/components/CodeInput.jsx` - Reusable code input (optional)
- `src/App.jsx` - Add route
- `src/api.js` - Add API method

---

## Prompt 1.7: Console - API Client Update

### Context
The console needs API methods to interact with the device auth endpoints.

### Task
Add device auth methods to `src/api.js`.

### Requirements

**New API Methods:**

```javascript
api.auth = {
  // ... existing methods ...
  
  // Approve a device code (requires auth)
  approveDeviceCode: async (userCode) => {
    return apiFetch('/v1/auth/device/approve', {
      method: 'POST',
      body: JSON.stringify({ user_code: userCode })
    });
  }
};
```

**Error Handling:**
- Handle 404 → "Invalid or expired code"
- Handle 409 → "Code already used"
- Handle 401 → Redirect to login
- Handle network errors → "Connection failed"

### Acceptance Criteria
- [ ] API method added
- [ ] Proper error transformation
- [ ] TypeScript types (if using TS)
- [ ] No breaking changes to existing code

---

## Prompt 1.8: Integration Testing

### Context
We need to verify the entire device auth flow works end-to-end.

### Task
Create integration tests for the device authorization flow.

### Requirements

**Test Scenarios:**

1. **Happy Path:**
   - CLI calls /device/init → gets codes
   - CLI polls /device/token → gets "pending"
   - User calls /device/approve with valid JWT
   - CLI polls /device/token → gets tokens
   - Verify tokens are valid

2. **Expired Code:**
   - CLI calls /device/init
   - Wait for expiration (or mock time)
   - CLI polls /device/token → gets "expired_token"

3. **Rate Limiting:**
   - CLI calls /device/init
   - CLI polls /device/token rapidly
   - Should get "slow_down" error

4. **Invalid Code:**
   - User calls /device/approve with wrong code
   - Should get 404

5. **Unauthenticated Approve:**
   - Call /device/approve without JWT
   - Should get 401

**Test Location:**
- `tests/e2e_device_auth_test.go`

### Acceptance Criteria
- [ ] All test scenarios pass
- [ ] Tests are independent (no shared state)
- [ ] Tests clean up after themselves
- [ ] Tests run in CI pipeline
- [ ] Coverage report generated

---

## Prompt 1.9: Documentation Update

### Context
The new device auth endpoints need to be documented.

### Task
Update API documentation and add Swagger annotations.

### Requirements

**Swagger Annotations:**
- Add to all three handlers
- Include request/response schemas
- Document all error codes
- Add examples

**API Docs:**
- Update `docs/swagger.yaml` (auto-generated)
- Verify Swagger UI shows new endpoints

**README Updates:**
- Add device auth section to backend README
- Document the flow with sequence diagram

### Acceptance Criteria
- [ ] Swagger annotations complete
- [ ] Swagger UI renders correctly
- [ ] All error codes documented
- [ ] Examples provided
- [ ] README updated

---

## Execution Order

Execute prompts in this order:

1. **1.2** - Database schema (backend)
2. **1.3** - Repository layer (backend)
3. **1.4** - API handlers (backend)
4. **1.5** - Cleanup job (backend)
5. **1.6** - Device approval page (console)
6. **1.7** - API client update (console)
7. **1.8** - Integration tests
8. **1.9** - Documentation
9. **1.1** - CLI repository (separate repo)

Backend first, then console, then CLI repo.

---

## Definition of Done (Phase 1)

- [ ] Device auth table exists in production database
- [ ] All three endpoints deployed and working
- [ ] Cleanup job running in production
- [ ] /device page live in console
- [ ] Integration tests passing in CI
- [ ] Swagger docs updated
- [ ] CLI repo initialized with version command
- [ ] No tech debt introduced
- [ ] No dead code
- [ ] Code reviewed and approved

---

*Phase 1 Complete → Proceed to Phase 2: CLI Core Commands*

