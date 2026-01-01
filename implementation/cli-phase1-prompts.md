# Phase 1: CLI Foundation - Implementation Prompts

> **Status:** READY FOR IMPLEMENTATION  
> **Duration:** Week 1  
> **Priority:** P0 - Critical Path
> **Auth Flow:** Browser Redirect (like TradingView, Vercel CLI)

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

## Authentication Flow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  TERMINAL                                                       │
│                                                                 │
│  $ machpay login                                                │
│                                                                 │
│  🌐 Opening browser for authentication...                       │
│  ⏳ Waiting for login (press Ctrl+C to cancel)                  │
│                                                                 │
│  ✅ Logged in as john@example.com                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

The flow:
1. CLI starts local HTTP server on random port (e.g., localhost:54321)
2. CLI opens browser to: console.machpay.xyz/auth/cli?port=54321
3. User logs in normally (Google, Wallet, Email OTP)
4. After login, console redirects to: localhost:54321/callback?token=JWT
5. CLI receives token, saves to ~/.machpay/config.yaml, closes server
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
│   │   ├── root.go              # Root command + global flags
│   │   ├── login.go             # Login command
│   │   └── version.go           # Version command
│   ├── config/                  # Configuration management
│   │   ├── config.go            # Viper loader
│   │   └── paths.go             # ~/.machpay paths
│   ├── auth/                    # Authentication
│   │   └── browser.go           # Browser redirect auth
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
- `github.com/pkg/browser` - Open browser cross-platform
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
- `BinDir()` → `~/.machpay/bin/`
- `LogDir()` → `~/.machpay/logs/`
- Create directories if they don't exist

### Acceptance Criteria
- [ ] `go build ./cmd/machpay` compiles without errors
- [ ] `./machpay --help` shows clean help text
- [ ] `./machpay version` prints version info
- [ ] `./machpay --debug version` enables debug logging
- [ ] Config directories are created on first run
- [ ] No unused imports or dead code
- [ ] All exported functions have doc comments

---

## Prompt 1.2: CLI Login Command with Browser Redirect

### Context
Implement the `machpay login` command that authenticates users via browser redirect.

### Task
Create the login command in `internal/cmd/login.go` and the auth helper in `internal/auth/browser.go`.

### Requirements

**Login Command Flow:**

```go
// internal/cmd/login.go

func runLogin(cmd *cobra.Command, args []string) error {
    // 1. Find a free port
    port, err := findFreePort()
    
    // 2. Start local callback server
    tokenChan := make(chan string)
    errChan := make(chan error)
    server := startCallbackServer(port, tokenChan, errChan)
    
    // 3. Build login URL with callback
    loginURL := fmt.Sprintf("%s/auth/cli?port=%d", consoleURL, port)
    
    // 4. Open browser
    fmt.Println("🌐 Opening browser for authentication...")
    browser.OpenURL(loginURL)
    
    // 5. Wait for callback (with timeout)
    select {
    case token := <-tokenChan:
        saveToken(token)
        fmt.Println("✅ Logged in successfully!")
    case err := <-errChan:
        return err
    case <-time.After(5 * time.Minute):
        return errors.New("login timed out")
    }
    
    // 6. Shutdown server
    server.Shutdown(context.Background())
    return nil
}
```

**Browser Auth Helper (internal/auth/browser.go):**

```go
// startCallbackServer starts an HTTP server to receive the auth callback
func startCallbackServer(port int, tokenChan chan<- string, errChan chan<- error) *http.Server {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/callback", func(w http.ResponseWriter, r *http.Request) {
        token := r.URL.Query().Get("token")
        if token == "" {
            errChan <- errors.New("no token received")
            http.Error(w, "No token", 400)
            return
        }
        
        // Send success page
        w.Header().Set("Content-Type", "text/html")
        w.Write([]byte(successHTML))
        
        tokenChan <- token
    })
    
    server := &http.Server{
        Addr:    fmt.Sprintf("localhost:%d", port),
        Handler: mux,
    }
    
    go server.ListenAndServe()
    return server
}

// Success HTML shown in browser after login
const successHTML = `
<!DOCTYPE html>
<html>
<head>
    <title>MachPay CLI - Login Success</title>
    <style>
        body { font-family: system-ui; display: flex; justify-content: center; 
               align-items: center; height: 100vh; margin: 0; background: #111; color: #fff; }
        .container { text-align: center; }
        .check { font-size: 64px; margin-bottom: 20px; }
        h1 { margin: 0 0 10px; }
        p { color: #888; }
    </style>
</head>
<body>
    <div class="container">
        <div class="check">✅</div>
        <h1>Login Successful!</h1>
        <p>You can close this window and return to your terminal.</p>
    </div>
</body>
</html>
`
```

**Token Storage:**

```go
// Save token to config file
func saveToken(token string) error {
    cfg := config.Load()
    cfg.Auth.Token = token
    cfg.Auth.LoginTime = time.Now()
    return cfg.Save()
}
```

### Flags
- `--no-browser` - Print URL instead of opening browser (for SSH/headless)

### Acceptance Criteria
- [ ] `machpay login` opens browser
- [ ] Token saved to `~/.machpay/config.yaml` after login
- [ ] `machpay login --no-browser` prints URL
- [ ] Timeout after 5 minutes with clear message
- [ ] Success page shown in browser
- [ ] Works on macOS, Linux, Windows

---

## Prompt 1.3: Console - CLI Login Callback Page

### Context
The web console needs a page that handles CLI login and redirects back to the CLI's local server.

### Task
Create `/auth/cli` page in machpay-console that:
1. Shows login UI if not authenticated
2. Redirects to CLI's localhost callback with token if authenticated

### Requirements

**Route:** `/auth/cli?port=54321`

**Page Logic:**

```jsx
// src/pages/CLILogin.jsx

export default function CLILogin() {
  const { user, isAuthenticated } = useAuth();
  const [searchParams] = useSearchParams();
  const port = searchParams.get('port');
  
  useEffect(() => {
    if (isAuthenticated && port) {
      // Redirect to CLI callback with token
      const token = getToken();
      window.location.href = `http://localhost:${port}/callback?token=${token}`;
    }
  }, [isAuthenticated, port]);
  
  if (!port) {
    return <ErrorPage message="Invalid CLI login request" />;
  }
  
  if (isAuthenticated) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <Loader2 className="animate-spin" />
        <span>Redirecting to CLI...</span>
      </div>
    );
  }
  
  // Show normal login UI with cli_port in state
  return <Auth returnTo={`/auth/cli?port=${port}`} />;
}
```

**Route Registration (App.jsx):**
```jsx
<Route path="/auth/cli" element={<CLILogin />} />
```

### Security Considerations
- Only redirect to localhost (not arbitrary URLs)
- Validate port is a number in valid range (1024-65535)
- Token is short-lived JWT
- HTTPS not needed for localhost callback

### Acceptance Criteria
- [ ] `/auth/cli?port=54321` shows login if not authenticated
- [ ] After login, redirects to `localhost:54321/callback?token=...`
- [ ] Error shown if port missing or invalid
- [ ] Loading state while redirecting
- [ ] Works with all login methods (Google, Wallet, Email)

---

## Prompt 1.4: CLI Status & Logout Commands

### Context
Users need to check their login status and logout.

### Task
Create `status` and `logout` commands.

### Requirements

**Status Command:**
```
$ machpay status

MachPay CLI v1.0.0

Authentication:
  ✅ Logged in as john@example.com
  🕐 Token expires in 23 hours
  
Role: Agent
Wallet: 0x71C7...976F (Solana)
```

**Logout Command:**
```
$ machpay logout

Are you sure? [y/N]: y
✅ Logged out successfully. Token removed.
```

### Acceptance Criteria
- [ ] `machpay status` shows auth info
- [ ] Shows "Not logged in" if no token
- [ ] `machpay logout` clears token
- [ ] Confirmation prompt for logout

---

## Prompt 1.5: Config File Structure

### Context
Define the config file structure stored at `~/.machpay/config.yaml`.

### Task
Create config types and loader in `internal/config/config.go`.

### Requirements

**Config Structure:**
```yaml
# ~/.machpay/config.yaml
version: "1.0"

auth:
  token: "eyJ..."
  login_time: "2024-01-01T12:00:00Z"
  expires_at: "2024-01-02T12:00:00Z"

user:
  id: 123
  email: "john@example.com"
  role: "agent"  # agent, vendor, or observer

gateway:
  binary_path: "~/.machpay/bin/machpay-gateway"
  version: "1.2.0"
  last_updated: "2024-01-01T12:00:00Z"

settings:
  auto_update: true
  telemetry: true
```

**Go Types:**
```go
type Config struct {
    Version  string         `yaml:"version"`
    Auth     AuthConfig     `yaml:"auth"`
    User     UserConfig     `yaml:"user"`
    Gateway  GatewayConfig  `yaml:"gateway"`
    Settings SettingsConfig `yaml:"settings"`
}

type AuthConfig struct {
    Token     string    `yaml:"token"`
    LoginTime time.Time `yaml:"login_time"`
    ExpiresAt time.Time `yaml:"expires_at"`
}
```

### Acceptance Criteria
- [ ] Config file created on first login
- [ ] YAML format, human-readable
- [ ] Sensitive data (token) stored securely
- [ ] File permissions: 0600 (owner read/write only)

---

## Prompt 1.6: Integration Testing

### Context
Test the full login flow end-to-end.

### Task
Create integration tests for CLI authentication.

### Requirements

**Test Scenarios:**

1. **Happy Path:**
   - CLI opens browser
   - Simulate token callback
   - Verify token saved

2. **Timeout:**
   - Start login
   - Don't complete within timeout
   - Verify clean error

3. **Invalid Callback:**
   - Callback without token
   - Verify error handling

4. **No Browser (Headless):**
   - `--no-browser` flag
   - Verify URL printed

### Acceptance Criteria
- [ ] All test scenarios pass
- [ ] Tests work in CI (mock browser open)
- [ ] Coverage > 80%

---

## Execution Order

Execute prompts in this order:

1. **1.1** - CLI repo scaffold (new repo)
2. **1.3** - Console CLI login page
3. **1.2** - CLI login command
4. **1.4** - Status/logout commands
5. **1.5** - Config structure
6. **1.6** - Integration tests

Console first, then CLI.

---

## Definition of Done (Phase 1)

- [ ] `machpay-cli` repo created and initialized
- [ ] `machpay login` opens browser and receives token
- [ ] `/auth/cli` page in console handles login redirect
- [ ] `machpay status` shows current auth state
- [ ] `machpay logout` clears auth
- [ ] Config file stored at `~/.machpay/config.yaml`
- [ ] Works on macOS, Linux, Windows
- [ ] Tests passing
- [ ] No tech debt
- [ ] No dead code

---

*Phase 1 Complete → Proceed to Phase 2: Gateway Integration*
