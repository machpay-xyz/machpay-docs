# MachPay CLI & Distribution Integration Plan

> **Status:** PLANNING  
> **Priority:** P0 - Critical  
> **Owner:** Core Team  
> **Last Updated:** 2024-12-31

---

## Executive Summary

This document outlines the complete integration plan for the MachPay CLI Orchestrator and multi-platform distribution strategy. This is a **make-or-break** initiative that will define the developer experience for MachPay.

### Distribution Matrix

| Platform | Method | What's Included |
|----------|--------|-----------------|
| **macOS** | DMG download | Console App + CLI + Gateway (bundled) |
| **macOS** | `brew install machpay` | CLI only (gateway downloaded on-demand) |
| **Windows** | EXE installer | Console App + CLI + Gateway (bundled) |
| **Linux** | AppImage/deb | Console App + CLI + Gateway (bundled) |
| **All** | `curl \| sh` | CLI only (gateway downloaded on-demand) |

### Repos Affected

| Repo | Changes | Priority |
|------|---------|----------|
| `machpay-cli` | **NEW** - Lightweight orchestrator | P0 |
| `machpay-gateway` | Add version endpoint, release automation | P0 |
| `machpay-backend` | Device auth flow endpoints | P0 |
| `machpay-console` | Tauri integration for desktop bundling | P1 |
| `machpay-website` | Download page, install script hosting | P1 |
| `homebrew-tap` | **NEW** - Homebrew formula repo | P2 |

---

## Phase 1: Foundation (Week 1)

### 1.1 Create `machpay-cli` Repository

**Objective:** Scaffold the new CLI repository with proper Go project structure.

#### Directory Structure

```
machpay-cli/
├── cmd/
│   └── machpay/
│       └── main.go                 # Entry point
├── internal/
│   ├── cmd/                        # Cobra commands
│   │   ├── root.go                 # Root command + global flags
│   │   ├── login.go                # Device flow auth
│   │   ├── logout.go               # Clear credentials
│   │   ├── setup.go                # Interactive wizard
│   │   ├── serve.go                # Download gateway + run
│   │   ├── status.go               # Health check
│   │   ├── open.go                 # Launch console
│   │   ├── update.go               # Self-update + gateway update
│   │   └── version.go              # Version info
│   ├── config/                     # Configuration management
│   │   ├── config.go               # Viper loader
│   │   ├── paths.go                # ~/.machpay paths
│   │   └── schema.go               # Config struct definitions
│   ├── auth/                       # Authentication
│   │   ├── device_flow.go          # OAuth device flow
│   │   ├── token.go                # Token storage/refresh
│   │   └── client.go               # API client with auth
│   ├── gateway/                    # Gateway management
│   │   ├── downloader.go           # Download from GitHub releases
│   │   ├── process.go              # Start/stop gateway process
│   │   ├── version.go              # Version checking
│   │   └── health.go               # Gateway health checks
│   ├── tui/                        # Terminal UI components
│   │   ├── spinner.go              # Loading spinners
│   │   ├── progress.go             # Download progress bar
│   │   ├── prompt.go               # Interactive prompts
│   │   ├── table.go                # Data tables
│   │   └── styles.go               # Lip Gloss styles
│   └── util/                       # Utilities
│       ├── browser.go              # Open browser cross-platform
│       ├── system.go               # OS/Arch detection
│       └── update.go               # Self-update logic
├── scripts/
│   └── install.sh                  # Curl installer script
├── .goreleaser.yaml                # Release automation
├── .github/
│   └── workflows/
│       └── release.yml             # CI/CD for releases
├── go.mod
├── go.sum
├── LICENSE
└── README.md
```

#### Dependencies

```go
// go.mod
module github.com/machpay/machpay-cli

go 1.22

require (
    github.com/spf13/cobra v1.8.0           // CLI framework
    github.com/spf13/viper v1.18.0          // Config management
    github.com/charmbracelet/bubbletea v0.25.0  // TUI framework
    github.com/charmbracelet/lipgloss v0.9.0    // TUI styling
    github.com/charmbracelet/bubbles v0.18.0    // TUI components
    github.com/schollz/progressbar/v3 v3.14.0   // Progress bars
    github.com/pkg/browser v0.0.0-20210911075715 // Open browser
    go.uber.org/zap v1.26.0                 // Logging
)
```

#### Tasks

- [ ] Initialize Go module with dependencies
- [ ] Create directory structure
- [ ] Implement root command with global flags (`--debug`, `--config`)
- [ ] Implement `version` command (hardcoded for now)
- [ ] Set up basic logging with Zap
- [ ] Create initial README with project overview

---

### 1.2 Backend: Device Authentication Flow

**Objective:** Implement OAuth 2.0 Device Authorization Grant in the backend.

#### New API Endpoints

```
POST /v1/auth/device/init
POST /v1/auth/device/token
POST /v1/auth/device/approve (called by web console)
```

#### Endpoint Specifications

**POST /v1/auth/device/init**

Request:
```json
{
  "client_id": "machpay-cli",
  "scope": "agent vendor"
}
```

Response:
```json
{
  "device_code": "GmRhYjQ5N2...",
  "user_code": "XK9-7NP",
  "verification_uri": "https://console.machpay.xyz/device",
  "verification_uri_complete": "https://console.machpay.xyz/device?code=XK9-7NP",
  "expires_in": 900,
  "interval": 5
}
```

**POST /v1/auth/device/token**

Request:
```json
{
  "client_id": "machpay-cli",
  "device_code": "GmRhYjQ5N2...",
  "grant_type": "urn:ietf:params:oauth:grant-type:device_code"
}
```

Response (pending):
```json
{
  "error": "authorization_pending"
}
```

Response (success):
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "id": "user_123",
    "email": "user@example.com",
    "name": "John Doe"
  }
}
```

#### Database Schema

```sql
-- migrations/XXXXXX_device_auth.up.sql

CREATE TABLE device_auth_requests (
    id              BIGSERIAL PRIMARY KEY,
    device_code     VARCHAR(64) UNIQUE NOT NULL,
    user_code       VARCHAR(16) UNIQUE NOT NULL,
    client_id       VARCHAR(64) NOT NULL,
    scope           VARCHAR(256),
    user_id         BIGINT REFERENCES users(id),  -- NULL until approved
    status          VARCHAR(20) DEFAULT 'pending', -- pending, approved, denied, expired
    expires_at      TIMESTAMP NOT NULL,
    approved_at     TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_device_auth_device_code ON device_auth_requests(device_code);
CREATE INDEX idx_device_auth_user_code ON device_auth_requests(user_code);
CREATE INDEX idx_device_auth_expires ON device_auth_requests(expires_at);
```

#### Tasks

- [ ] Create migration for `device_auth_requests` table
- [ ] Implement `POST /v1/auth/device/init` handler
- [ ] Implement `POST /v1/auth/device/token` handler (polling)
- [ ] Implement `POST /v1/auth/device/approve` handler (web console calls this)
- [ ] Add cleanup job for expired device codes
- [ ] Write unit tests for device flow
- [ ] Update API documentation

---

### 1.3 Console: Device Approval Page

**Objective:** Create the web page where users approve CLI login requests.

#### Route

```
/device
/device?code=XK9-7NP (pre-filled)
```

#### UI Components

```jsx
// src/pages/DeviceAuth.jsx

export function DeviceAuth() {
  const [code, setCode] = useState('');
  const [status, setStatus] = useState('input'); // input, verifying, success, error
  const [error, setError] = useState(null);
  
  // If code in URL, pre-fill and auto-verify
  useEffect(() => {
    const urlCode = new URLSearchParams(location.search).get('code');
    if (urlCode) {
      setCode(urlCode);
      handleVerify(urlCode);
    }
  }, []);

  const handleVerify = async (userCode) => {
    setStatus('verifying');
    try {
      await api.verifyDeviceCode(userCode);
      setStatus('success');
    } catch (err) {
      setError(err.message);
      setStatus('error');
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center">
      <Panel className="max-w-md w-full p-8">
        {status === 'input' && (
          <>
            <h1>Link MachPay CLI</h1>
            <p>Enter the code shown in your terminal</p>
            <CodeInput value={code} onChange={setCode} />
            <Button onClick={() => handleVerify(code)}>
              Authorize
            </Button>
          </>
        )}
        
        {status === 'verifying' && (
          <Spinner>Verifying...</Spinner>
        )}
        
        {status === 'success' && (
          <>
            <CheckCircle className="text-emerald-500 w-16 h-16" />
            <h1>CLI Linked!</h1>
            <p>You can close this window and return to your terminal.</p>
          </>
        )}
        
        {status === 'error' && (
          <>
            <XCircle className="text-red-500 w-16 h-16" />
            <h1>Verification Failed</h1>
            <p>{error}</p>
            <Button onClick={() => setStatus('input')}>Try Again</Button>
          </>
        )}
      </Panel>
    </div>
  );
}
```

#### Tasks

- [ ] Create `/device` route in App.jsx
- [ ] Implement `DeviceAuth` page component
- [ ] Create `CodeInput` component (6-character input with formatting)
- [ ] Add `api.verifyDeviceCode()` and `api.approveDeviceCode()` methods
- [ ] Handle auto-verification when code is in URL
- [ ] Add success/error animations
- [ ] Test flow end-to-end

---

## Phase 2: CLI Core Commands (Week 2)

### 2.1 `machpay login` Command

**Objective:** Implement device flow authentication in CLI.

#### User Flow

```
$ machpay login

🔐 Authenticate with MachPay

  1. Your verification code is: XK9-7NP
  2. Press [Enter] to open browser...

  Opening https://console.machpay.xyz/device?code=XK9-7NP

  Waiting for approval... ◐

✅ Logged in as abhishek@machpay.xyz

  Your credentials have been saved to ~/.machpay/config.yaml
```

#### Implementation

```go
// internal/cmd/login.go

package cmd

import (
    "github.com/spf13/cobra"
    "github.com/machpay/machpay-cli/internal/auth"
    "github.com/machpay/machpay-cli/internal/tui"
)

var loginCmd = &cobra.Command{
    Use:   "login",
    Short: "Authenticate with MachPay",
    Long:  "Link your CLI to your MachPay account using secure device flow authentication.",
    RunE:  runLogin,
}

func runLogin(cmd *cobra.Command, args []string) error {
    // 1. Check if already logged in
    if auth.IsLoggedIn() {
        fmt.Println("Already logged in. Use 'machpay logout' first.")
        return nil
    }

    // 2. Initialize device flow
    deviceResp, err := auth.InitDeviceFlow()
    if err != nil {
        return fmt.Errorf("failed to initialize login: %w", err)
    }

    // 3. Display code and prompt
    tui.PrintHeader("🔐 Authenticate with MachPay")
    fmt.Printf("\n  1. Your verification code is: %s\n", tui.Bold(deviceResp.UserCode))
    fmt.Println("  2. Press [Enter] to open browser...")
    
    tui.WaitForEnter()

    // 4. Open browser
    browser.Open(deviceResp.VerificationURIComplete)

    // 5. Poll for approval with spinner
    spinner := tui.NewSpinner("Waiting for approval...")
    spinner.Start()
    
    tokenResp, err := auth.PollForToken(deviceResp.DeviceCode, deviceResp.Interval)
    spinner.Stop()
    
    if err != nil {
        return fmt.Errorf("authentication failed: %w", err)
    }

    // 6. Save credentials
    if err := auth.SaveCredentials(tokenResp); err != nil {
        return fmt.Errorf("failed to save credentials: %w", err)
    }

    // 7. Success message
    tui.PrintSuccess(fmt.Sprintf("Logged in as %s", tokenResp.User.Email))
    fmt.Println("\n  Your credentials have been saved to ~/.machpay/config.yaml")
    
    return nil
}
```

#### Tasks

- [ ] Implement `auth.InitDeviceFlow()` API call
- [ ] Implement `auth.PollForToken()` with exponential backoff
- [ ] Implement `auth.SaveCredentials()` to config file
- [ ] Create TUI components (spinner, bold text, success message)
- [ ] Handle Ctrl+C gracefully during polling
- [ ] Add `--no-browser` flag for headless environments
- [ ] Write integration tests

---

### 2.2 `machpay setup` Command

**Objective:** Interactive wizard for configuring agent or vendor mode.

#### User Flow (Agent)

```
$ machpay setup

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   ███╗   ███╗ █████╗  ██████╗██╗  ██╗██████╗  █████╗ ██╗   │
│   ████╗ ████║██╔══██╗██╔════╝██║  ██║██╔══██╗██╔══██╗╚██╗  │
│   ██╔████╔██║███████║██║     ███████║██████╔╝███████║ ██║  │
│   ██║╚██╔╝██║██╔══██║██║     ██╔══██║██╔═══╝ ██╔══██║ ██║  │
│   ██║ ╚═╝ ██║██║  ██║╚██████╗██║  ██║██║     ██║  ██║██╔╝  │
│   ╚═╝     ╚═╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝╚═╝   │
│                                                              │
│   Setup Wizard                                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘

? What do you want to do?
  ❯ Run an AI Agent (I want to use APIs)
    Run a Vendor Node (I want to sell APIs)

───────────────────────────────────────────────────────────────

? Select network:
  ❯ Mainnet (Real money)
    Devnet (Testing - free tokens)

───────────────────────────────────────────────────────────────

? Wallet setup:
  ❯ Generate new wallet (Recommended)
    Import existing keypair

───────────────────────────────────────────────────────────────

🔑 Generated new wallet

   Address: 7xK9mN3pQr...
   Saved to: ~/.machpay/wallet.json
   
   ⚠️  BACKUP THIS FILE! It contains your private key.

───────────────────────────────────────────────────────────────

✅ Agent setup complete!

   Your API Key: mp_live_7xK9...████████████...3nP
   
   Quick Start:
   ┌────────────────────────────────────────────────────────┐
   │ pip install machpay                                    │
   │                                                        │
   │ from machpay import MachPay                            │
   │ client = MachPay()  # Uses ~/.machpay config           │
   │ response = client.call("weather-api", "/v1/forecast")  │
   └────────────────────────────────────────────────────────┘
   
   Next: Fund your wallet at https://console.machpay.xyz/agent/fund
```

#### User Flow (Vendor)

```
$ machpay setup

... (same header) ...

? What do you want to do?
    Run an AI Agent (I want to use APIs)
  ❯ Run a Vendor Node (I want to sell APIs)

───────────────────────────────────────────────────────────────

? Service name: LLaMA Inference API

? Description: Fast local LLM inference using Ollama

? Category:
  ❯ AI/ML
    Data
    Finance
    Compute
    Other

───────────────────────────────────────────────────────────────

? Where is your API running? (upstream URL)
  › http://localhost:11434

? Default price per request (USDC):
  › 0.001

───────────────────────────────────────────────────────────────

? Select network:
  ❯ Mainnet
    Devnet

───────────────────────────────────────────────────────────────

? Wallet setup:
  ❯ Generate new wallet
    Import existing keypair

🔑 Generated wallet: 9xM2...7qR
   Saved to: ~/.machpay/wallet.json

───────────────────────────────────────────────────────────────

✅ Vendor setup complete!

   Service registered: LLaMA Inference API
   Config saved to: ~/.machpay/config.yaml
   
   Start your gateway:
   $ machpay serve
   
   Your API will be available at:
   https://llama-inference.machpay.network

───────────────────────────────────────────────────────────────

? Start gateway now? [Y/n]
```

#### Implementation Notes

- Use Bubble Tea for interactive prompts
- Validate inputs (URL format, price range)
- Register service with backend during vendor setup
- Generate secure keypair using Solana SDK
- Create API key during agent setup

#### Tasks

- [ ] Implement role selection prompt
- [ ] Implement network selection prompt
- [ ] Implement wallet generation/import
- [ ] Implement vendor service registration API call
- [ ] Implement agent API key creation
- [ ] Create ASCII art banner
- [ ] Save config to `~/.machpay/config.yaml`
- [ ] Add `--non-interactive` flag with env vars for CI/CD
- [ ] Write unit tests for each step

---

### 2.3 `machpay status` Command

**Objective:** Quick health check showing current state.

#### Output

```
$ machpay status

MachPay Status
══════════════════════════════════════════════════════

  Account:    abhishek@machpay.xyz ✓
  Role:       Vendor
  Network:    Mainnet

  Gateway:
    Status:   ● Running (PID 12345)
    Version:  v1.2.0
    Uptime:   2h 34m
    Port:     8402

  Wallet:
    Address:  7xK9...3nP
    Balance:  $125.50 USDC

══════════════════════════════════════════════════════
```

#### Tasks

- [ ] Check authentication status
- [ ] Check gateway process status
- [ ] Fetch wallet balance from RPC
- [ ] Format output with Lip Gloss
- [ ] Add `--json` flag for scripting
- [ ] Add `--watch` flag for continuous monitoring

---

### 2.4 `machpay open` Command

**Objective:** Launch the console (desktop app or web).

#### Behavior

1. Check if Console.app is installed
2. If yes, launch it
3. If no, open web console in browser

```go
func runOpen(cmd *cobra.Command, args []string) error {
    // Check for desktop app
    appPath := getConsoleAppPath() // Platform-specific
    
    if _, err := os.Stat(appPath); err == nil {
        // Desktop app exists, launch it
        return exec.Command("open", appPath).Run() // macOS
    }
    
    // Fall back to web console
    return browser.Open("https://console.machpay.xyz")
}
```

#### Tasks

- [ ] Detect Console.app on macOS
- [ ] Detect Console.exe on Windows
- [ ] Launch desktop app if available
- [ ] Fall back to web browser
- [ ] Add `--web` flag to force browser
- [ ] Handle specific routes: `machpay open funding`, `machpay open marketplace`

---

## Phase 3: Gateway Integration (Week 3)

### 3.1 Gateway Downloader

**Objective:** Download and manage the gateway binary.

#### Download Flow

```
$ machpay serve

  Checking gateway installation...
  
  ✗ Gateway not found
  
  Downloading machpay-gateway v1.2.0 for darwin/arm64...
  ████████████████████████████████████████ 100%  32.1 MB
  
  Installing to ~/.machpay/bin/machpay-gateway
  ✓ Gateway v1.2.0 installed
  
  Starting gateway...
```

#### Implementation

```go
// internal/gateway/downloader.go

package gateway

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
    "runtime"
)

const (
    GatewayRepo   = "machpay/machpay-gateway"
    GatewayBinary = "machpay-gateway"
)

type Downloader struct {
    installDir string
    httpClient *http.Client
}

func NewDownloader() *Downloader {
    return &Downloader{
        installDir: filepath.Join(os.Getenv("HOME"), ".machpay", "bin"),
        httpClient: &http.Client{},
    }
}

func (d *Downloader) IsInstalled() bool {
    _, err := os.Stat(d.BinaryPath())
    return err == nil
}

func (d *Downloader) BinaryPath() string {
    binary := GatewayBinary
    if runtime.GOOS == "windows" {
        binary += ".exe"
    }
    return filepath.Join(d.installDir, binary)
}

func (d *Downloader) GetLatestVersion() (string, error) {
    // GET https://api.github.com/repos/machpay/machpay-gateway/releases/latest
    // Parse response for tag_name
}

func (d *Downloader) InstalledVersion() (string, error) {
    // Run: ~/.machpay/bin/machpay-gateway --version
    // Parse output
}

func (d *Downloader) Download(version string, progress io.Writer) error {
    // 1. Construct download URL
    url := fmt.Sprintf(
        "https://github.com/%s/releases/download/%s/machpay-gateway_%s_%s.tar.gz",
        GatewayRepo, version, runtime.GOOS, runtime.GOARCH,
    )
    
    // 2. Download with progress
    // 3. Extract tarball
    // 4. Make executable
    // 5. Verify checksum
}

func (d *Downloader) NeedsUpdate() (bool, string, error) {
    installed, err := d.InstalledVersion()
    if err != nil {
        return true, "", nil // Not installed
    }
    
    latest, err := d.GetLatestVersion()
    if err != nil {
        return false, "", err
    }
    
    return installed != latest, latest, nil
}
```

#### Tasks

- [ ] Implement GitHub API client for releases
- [ ] Implement download with progress bar
- [ ] Implement tarball extraction
- [ ] Verify SHA256 checksum
- [ ] Make binary executable (chmod +x)
- [ ] Handle download interruption gracefully
- [ ] Add retry logic for failed downloads
- [ ] Write tests with mock HTTP server

---

### 3.2 Gateway Process Manager

**Objective:** Start, stop, and monitor the gateway process.

#### Implementation

```go
// internal/gateway/process.go

package gateway

import (
    "os"
    "os/exec"
    "syscall"
)

type ProcessManager struct {
    binaryPath string
    pidFile    string
    logFile    string
    config     *Config
}

func (pm *ProcessManager) Start() error {
    // 1. Check if already running
    if pm.IsRunning() {
        return ErrAlreadyRunning
    }
    
    // 2. Build command with config
    cmd := exec.Command(pm.binaryPath,
        "--config", pm.config.Path(),
        "--port", pm.config.Port,
        "--upstream", pm.config.Upstream,
    )
    
    // 3. Set up logging
    logFile, _ := os.Create(pm.logFile)
    cmd.Stdout = logFile
    cmd.Stderr = logFile
    
    // 4. Start process
    if err := cmd.Start(); err != nil {
        return err
    }
    
    // 5. Save PID
    return pm.savePID(cmd.Process.Pid)
}

func (pm *ProcessManager) Stop() error {
    pid, err := pm.loadPID()
    if err != nil {
        return ErrNotRunning
    }
    
    process, err := os.FindProcess(pid)
    if err != nil {
        return err
    }
    
    return process.Signal(syscall.SIGTERM)
}

func (pm *ProcessManager) IsRunning() bool {
    pid, err := pm.loadPID()
    if err != nil {
        return false
    }
    
    process, err := os.FindProcess(pid)
    if err != nil {
        return false
    }
    
    // Check if process is alive
    return process.Signal(syscall.Signal(0)) == nil
}

func (pm *ProcessManager) Logs(follow bool) error {
    if follow {
        // Tail -f the log file
    } else {
        // Cat the log file
    }
}
```

#### Tasks

- [ ] Implement `Start()` with proper process isolation
- [ ] Implement `Stop()` with graceful shutdown
- [ ] Implement `Restart()` 
- [ ] Implement `IsRunning()` check
- [ ] Implement `Logs()` with follow mode
- [ ] Save PID to `~/.machpay/gateway.pid`
- [ ] Handle orphaned processes
- [ ] Add health check after start
- [ ] Write integration tests

---

### 3.3 `machpay serve` Command

**Objective:** The main vendor command that downloads gateway and runs it.

#### Behavior

```
$ machpay serve

┌──────────────────────────────────────────────────────────────┐
│  MachPay Gateway v1.2.0                                      │
│  ════════════════════════════════════════════════════════════│
│                                                              │
│  Service:    LLaMA Inference API                             │
│  Upstream:   http://localhost:11434                          │
│  Gateway:    http://localhost:8402                           │
│  Public:     https://llama-inference.machpay.network         │
│                                                              │
│  Status:     ● LIVE                                          │
│  ════════════════════════════════════════════════════════════│
│                                                              │
│  [12:34:01] Gateway started                                  │
│  [12:34:01] Registered with MachPay network                  │
│  [12:34:02] Health check passed                              │
│  [12:34:05] Ready to accept payments                         │
│                                                              │
│  Press Ctrl+C to stop                                        │
└──────────────────────────────────────────────────────────────┘
```

#### Implementation

```go
func runServe(cmd *cobra.Command, args []string) error {
    // 1. Verify logged in and vendor role
    if !auth.IsLoggedIn() {
        return fmt.Errorf("not logged in. Run 'machpay login' first")
    }
    
    cfg := config.Load()
    if cfg.Role != "vendor" {
        return fmt.Errorf("not configured as vendor. Run 'machpay setup' first")
    }

    // 2. Ensure gateway is installed
    dl := gateway.NewDownloader()
    if !dl.IsInstalled() {
        fmt.Println("Gateway not found. Downloading...")
        if err := downloadWithProgress(dl); err != nil {
            return err
        }
    }

    // 3. Check for updates (non-blocking)
    go checkGatewayUpdate(dl)

    // 4. Start gateway
    pm := gateway.NewProcessManager(dl.BinaryPath(), cfg)
    
    // 5. Display status panel
    panel := tui.NewGatewayPanel(cfg)
    panel.Show()

    // 6. Stream logs to panel
    go pm.StreamLogs(panel)

    // 7. Wait for interrupt
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig

    // 8. Graceful shutdown
    fmt.Println("\nShutting down...")
    return pm.Stop()
}
```

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--port` | 8402 | Gateway listen port |
| `--upstream` | from config | Override upstream URL |
| `--detach` | false | Run in background |
| `--debug` | false | Enable debug logging |

#### Tasks

- [ ] Implement serve command structure
- [ ] Integrate gateway downloader
- [ ] Integrate process manager
- [ ] Create gateway status panel (Bubble Tea)
- [ ] Stream logs to panel
- [ ] Handle Ctrl+C gracefully
- [ ] Implement `--detach` mode
- [ ] Add health check after start
- [ ] Write integration tests

---

### 3.4 Gateway Updates in `machpay-gateway` Repo

**Objective:** Prepare the gateway repo for CLI integration.

#### Required Changes

1. **Version Endpoint**
```go
// Add to gateway HTTP server
GET /healthz/version
Response: {"version": "1.2.0", "commit": "abc123"}
```

2. **Config File Support**
```yaml
# ~/.machpay/gateway.yaml (loaded by gateway)
upstream: http://localhost:11434
port: 8402
telemetry:
  enabled: true
  endpoint: https://telemetry.machpay.xyz
```

3. **GoReleaser Config**
```yaml
# .goreleaser.yaml in machpay-gateway
project_name: machpay-gateway

builds:
  - main: ./cmd/gateway
    binary: machpay-gateway
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]

archives:
  - format: tar.gz
    name_template: "{{ .ProjectName }}_{{ .Os }}_{{ .Arch }}"

checksum:
  name_template: 'checksums.txt'
  algorithm: sha256

release:
  github:
    owner: machpay
    name: machpay-gateway
```

#### Tasks

- [ ] Add `/healthz/version` endpoint
- [ ] Add `--version` CLI flag
- [ ] Add config file loading
- [ ] Add `.goreleaser.yaml`
- [ ] Add release GitHub Action
- [ ] Test release process on dev branch
- [ ] Document release process

---

## Phase 4: Desktop Bundling (Week 4)

### 4.1 Tauri Integration

**Objective:** Bundle the React console as a native desktop app with CLI included.

#### Why Tauri over Electron?

| Aspect | Tauri | Electron |
|--------|-------|----------|
| Binary size | ~10MB | ~150MB |
| Memory usage | ~50MB | ~200MB |
| Security | Rust backend | Node.js |
| Build speed | Fast | Slow |

#### Tauri Setup

```bash
# In machpay-console
npm install -D @tauri-apps/cli @tauri-apps/api
npm run tauri init
```

#### Directory Structure

```
machpay-console/
├── src/                      # React app (existing)
├── src-tauri/                # NEW - Tauri Rust backend
│   ├── src/
│   │   └── main.rs           # Tauri entry point
│   ├── tauri.conf.json       # Tauri config
│   ├── icons/                # App icons
│   └── Cargo.toml
├── package.json              # Add tauri scripts
└── ...
```

#### Tauri Configuration

```json
// src-tauri/tauri.conf.json
{
  "productName": "MachPay",
  "version": "1.0.0",
  "identifier": "xyz.machpay.console",
  "build": {
    "distDir": "../dist",
    "devPath": "http://localhost:5173"
  },
  "bundle": {
    "active": true,
    "targets": ["dmg", "msi", "appimage", "deb"],
    "identifier": "xyz.machpay.console",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "resources": [
      "bin/machpay",
      "bin/machpay-gateway"
    ],
    "macOS": {
      "minimumSystemVersion": "10.15"
    },
    "windows": {
      "wix": {
        "language": "en-US"
      }
    }
  },
  "tauri": {
    "allowlist": {
      "all": false,
      "shell": {
        "open": true,
        "execute": true,
        "sidecar": true
      },
      "fs": {
        "readFile": true,
        "writeFile": true,
        "scope": ["$APP/*", "$HOME/.machpay/*"]
      }
    },
    "windows": [
      {
        "title": "MachPay Console",
        "width": 1400,
        "height": 900,
        "minWidth": 1000,
        "minHeight": 600,
        "center": true
      }
    ]
  }
}
```

#### Bundle CLI and Gateway

The Tauri build process needs to include the CLI and Gateway binaries:

```rust
// src-tauri/build.rs

use std::process::Command;

fn main() {
    // Download CLI binary for current platform
    download_binary("machpay-cli", "machpay");
    
    // Download Gateway binary for current platform
    download_binary("machpay-gateway", "machpay-gateway");
    
    tauri_build::build();
}

fn download_binary(repo: &str, name: &str) {
    // Download from GitHub releases
    // Place in src-tauri/bin/
}
```

#### First Launch Setup

```rust
// src-tauri/src/main.rs

#[tauri::command]
fn install_cli() -> Result<(), String> {
    // Symlink bundled CLI to /usr/local/bin/machpay
    let cli_path = get_bundled_cli_path();
    let symlink_path = "/usr/local/bin/machpay";
    
    std::os::unix::fs::symlink(cli_path, symlink_path)
        .map_err(|e| e.to_string())
}

#[tauri::command]
fn start_gateway(config: GatewayConfig) -> Result<(), String> {
    // Start bundled gateway as sidecar process
    let gateway_path = get_bundled_gateway_path();
    
    Command::new(gateway_path)
        .args(&["--config", &config.path])
        .spawn()
        .map_err(|e| e.to_string())?;
    
    Ok(())
}
```

#### Tasks

- [ ] Initialize Tauri in machpay-console
- [ ] Configure `tauri.conf.json`
- [ ] Create build script to bundle CLI and Gateway
- [ ] Implement `install_cli` command
- [ ] Implement `start_gateway` command
- [ ] Create app icons for all platforms
- [ ] Build DMG for macOS
- [ ] Build MSI/EXE for Windows
- [ ] Build AppImage/deb for Linux
- [ ] Test installation on all platforms
- [ ] Add code signing (macOS notarization, Windows signing)
- [ ] Create auto-updater

---

### 4.2 First Launch Experience (Desktop App)

**Objective:** Guide users through setup when they first open the app.

#### Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                          │  │
│   │              Welcome to MachPay! 🚀                      │  │
│   │                                                          │  │
│   │   ┌─────────────────────────────────────────────────┐   │  │
│   │   │                                                  │   │  │
│   │   │   MachPay CLI                                    │   │  │
│   │   │   ─────────────────                              │   │  │
│   │   │                                                  │   │  │
│   │   │   Install CLI to your terminal for:              │   │  │
│   │   │   • Quick commands (machpay status)              │   │  │
│   │   │   • Running vendor gateway                       │   │  │
│   │   │   • SDK integration                              │   │  │
│   │   │                                                  │   │  │
│   │   │   Location: /usr/local/bin/machpay               │   │  │
│   │   │                                                  │   │  │
│   │   │   ☑️ Install CLI (recommended)                    │   │  │
│   │   │                                                  │   │  │
│   │   └─────────────────────────────────────────────────┘   │  │
│   │                                                          │  │
│   │                              [Skip]  [Continue →]        │  │
│   │                                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Tasks

- [ ] Create `FirstLaunchModal` component
- [ ] Detect if CLI is installed in PATH
- [ ] Implement CLI symlink creation (with permission request)
- [ ] Save first launch completion to config
- [ ] Skip modal on subsequent launches

---

## Phase 5: Distribution (Week 5)

### 5.1 Homebrew Tap

**Objective:** Create and maintain Homebrew formula for CLI.

#### Repository Setup

```bash
# Create repo: github.com/machpay/homebrew-tap

homebrew-tap/
├── Formula/
│   ├── machpay.rb           # CLI formula
│   └── machpay-gateway.rb   # Gateway formula (optional)
└── README.md
```

#### CLI Formula

```ruby
# Formula/machpay.rb

class Machpay < Formula
  desc "MachPay CLI - Orchestrator for AI agent payments"
  homepage "https://machpay.xyz"
  version "1.0.0"
  license "MIT"

  on_macos do
    on_intel do
      url "https://github.com/machpay/machpay-cli/releases/download/v1.0.0/machpay_darwin_amd64.tar.gz"
      sha256 "abc123..."
    end
    on_arm do
      url "https://github.com/machpay/machpay-cli/releases/download/v1.0.0/machpay_darwin_arm64.tar.gz"
      sha256 "def456..."
    end
  end

  on_linux do
    on_intel do
      url "https://github.com/machpay/machpay-cli/releases/download/v1.0.0/machpay_linux_amd64.tar.gz"
      sha256 "ghi789..."
    end
    on_arm do
      url "https://github.com/machpay/machpay-cli/releases/download/v1.0.0/machpay_linux_arm64.tar.gz"
      sha256 "jkl012..."
    end
  end

  def install
    bin.install "machpay"
  end

  def caveats
    <<~EOS
      MachPay CLI installed! 🚀
      
      Get started:
        machpay login     # Link your account
        machpay setup     # Configure your node
      
      The gateway will be downloaded automatically when you run:
        machpay serve
        
      Documentation: https://docs.machpay.xyz/cli
    EOS
  end

  test do
    assert_match "machpay version", shell_output("#{bin}/machpay version")
  end
end
```

#### GoReleaser Homebrew Integration

```yaml
# In machpay-cli/.goreleaser.yaml

brews:
  - name: machpay
    repository:
      owner: machpay
      name: homebrew-tap
      token: "{{ .Env.HOMEBREW_TAP_TOKEN }}"
    folder: Formula
    homepage: "https://machpay.xyz"
    description: "MachPay CLI - Orchestrator for AI agent payments"
    license: "MIT"
    install: |
      bin.install "machpay"
    caveats: |
      MachPay CLI installed! 🚀
      
      Get started:
        machpay login
        machpay setup
    test: |
      system "#{bin}/machpay", "version"
```

#### Tasks

- [ ] Create `machpay/homebrew-tap` repository
- [ ] Add initial formula
- [ ] Configure GoReleaser to update formula automatically
- [ ] Create `HOMEBREW_TAP_TOKEN` secret in machpay-cli repo
- [ ] Test: `brew tap machpay/tap && brew install machpay`
- [ ] Document in README

---

### 5.2 Install Script

**Objective:** Provide curl-based installation for all platforms.

#### Script Location

Host at: `https://machpay.xyz/install.sh`

#### Implementation

```bash
#!/bin/sh
# MachPay CLI Installer
# Usage: curl -fsSL https://machpay.xyz/install.sh | sh

set -e

REPO="machpay/machpay-cli"
INSTALL_DIR="${MACHPAY_INSTALL_DIR:-/usr/local/bin}"
BINARY_NAME="machpay"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

print_banner() {
    echo ""
    echo "${BLUE}███╗   ███╗ █████╗  ██████╗██╗  ██╗██████╗  █████╗ ██╗${NC}"
    echo "${BLUE}████╗ ████║██╔══██╗██╔════╝██║  ██║██╔══██╗██╔══██╗╚██╗${NC}"
    echo "${BLUE}██╔████╔██║███████║██║     ███████║██████╔╝███████║ ██║${NC}"
    echo "${BLUE}██║╚██╔╝██║██╔══██║██║     ██╔══██║██╔═══╝ ██╔══██║ ██║${NC}"
    echo "${BLUE}██║ ╚═╝ ██║██║  ██║╚██████╗██║  ██║██║     ██║  ██║██╔╝${NC}"
    echo "${BLUE}╚═╝     ╚═╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝╚═╝${NC}"
    echo ""
    echo "CLI Installer"
    echo ""
}

detect_platform() {
    OS=$(uname -s | tr '[:upper:]' '[:lower:]')
    ARCH=$(uname -m)
    
    case "$ARCH" in
        x86_64)  ARCH="amd64" ;;
        aarch64) ARCH="arm64" ;;
        arm64)   ARCH="arm64" ;;
        *)
            echo "${RED}Error: Unsupported architecture: $ARCH${NC}"
            exit 1
            ;;
    esac
    
    case "$OS" in
        darwin) OS="darwin" ;;
        linux)  OS="linux" ;;
        mingw*|msys*|cygwin*)
            OS="windows"
            BINARY_NAME="machpay.exe"
            ;;
        *)
            echo "${RED}Error: Unsupported OS: $OS${NC}"
            exit 1
            ;;
    esac
    
    echo "Detected: ${OS}/${ARCH}"
}

get_latest_version() {
    VERSION=$(curl -sL "https://api.github.com/repos/${REPO}/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
    
    if [ -z "$VERSION" ]; then
        echo "${RED}Error: Could not determine latest version${NC}"
        exit 1
    fi
    
    echo "Latest version: ${VERSION}"
}

download_and_install() {
    URL="https://github.com/${REPO}/releases/download/${VERSION}/${BINARY_NAME}_${OS}_${ARCH}.tar.gz"
    CHECKSUM_URL="https://github.com/${REPO}/releases/download/${VERSION}/checksums.txt"
    
    echo "Downloading from: ${URL}"
    
    # Create temp directory
    TMP_DIR=$(mktemp -d)
    trap "rm -rf $TMP_DIR" EXIT
    
    # Download binary
    curl -sL "$URL" | tar xz -C "$TMP_DIR"
    
    # Download and verify checksum
    echo "Verifying checksum..."
    EXPECTED=$(curl -sL "$CHECKSUM_URL" | grep "${BINARY_NAME}_${OS}_${ARCH}" | cut -d ' ' -f 1)
    ACTUAL=$(shasum -a 256 "$TMP_DIR/$BINARY_NAME" | cut -d ' ' -f 1)
    
    if [ "$EXPECTED" != "$ACTUAL" ]; then
        echo "${RED}Error: Checksum verification failed${NC}"
        exit 1
    fi
    
    # Install
    if [ -w "$INSTALL_DIR" ]; then
        mv "$TMP_DIR/$BINARY_NAME" "$INSTALL_DIR/"
    else
        echo "Installing to $INSTALL_DIR (requires sudo)..."
        sudo mv "$TMP_DIR/$BINARY_NAME" "$INSTALL_DIR/"
    fi
    
    chmod +x "$INSTALL_DIR/$BINARY_NAME"
}

print_success() {
    echo ""
    echo "${GREEN}✅ MachPay CLI installed successfully!${NC}"
    echo ""
    echo "Get started:"
    echo "  ${YELLOW}machpay login${NC}     # Link your account"
    echo "  ${YELLOW}machpay setup${NC}     # Configure your node"
    echo ""
    echo "Documentation: https://docs.machpay.xyz/cli"
    echo ""
}

main() {
    print_banner
    detect_platform
    get_latest_version
    download_and_install
    print_success
}

main
```

#### Tasks

- [ ] Create install script
- [ ] Add checksum verification
- [ ] Handle Windows (PowerShell version)
- [ ] Add to machpay-website repo
- [ ] Configure CDN/hosting
- [ ] Add telemetry (anonymous install count)
- [ ] Test on macOS, Linux, WSL

---

### 5.3 Website Download Page

**Objective:** Create compelling download page with all options.

#### URL

`https://machpay.xyz/download`

#### Design

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│                    Download MachPay                             │
│                                                                 │
│   Choose your preferred installation method                     │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                          │  │
│   │   🖥️  Desktop App (Recommended)                          │  │
│   │   ─────────────────────────────                          │  │
│   │   Full console with CLI and Gateway bundled              │  │
│   │                                                          │  │
│   │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │  │
│   │   │   macOS      │ │   Windows    │ │   Linux      │   │  │
│   │   │  ┌────────┐  │ │  ┌────────┐  │ │  ┌────────┐  │   │  │
│   │   │  │  DMG   │  │ │  │  EXE   │  │ │  │AppImage│  │   │  │
│   │   │  └────────┘  │ │  └────────┘  │ │  └────────┘  │   │  │
│   │   │  Intel │ ARM │ │    64-bit    │ │ Intel │ ARM │   │  │
│   │   └──────────────┘ └──────────────┘ └──────────────┘   │  │
│   │                                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                          │  │
│   │   ⌨️  CLI Only                                            │  │
│   │   ─────────────                                          │  │
│   │   Lightweight command-line tool                          │  │
│   │                                                          │  │
│   │   macOS:  brew install machpay/tap/machpay              │  │
│   │                                                          │  │
│   │   Linux:  curl -fsSL https://machpay.xyz/install.sh | sh│  │
│   │                                                          │  │
│   │   Windows: iwr machpay.xyz/install.ps1 | iex            │  │
│   │                                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                          │  │
│   │   🐳 Docker                                              │  │
│   │   ──────────                                             │  │
│   │   For CI/CD and containerized environments               │  │
│   │                                                          │  │
│   │   docker pull ghcr.io/machpay/cli:latest                │  │
│   │                                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Tasks

- [ ] Create download page in machpay-website
- [ ] Add platform detection (auto-select)
- [ ] Add download tracking analytics
- [ ] Add version info and changelog link
- [ ] Create PowerShell install script for Windows
- [ ] Add Docker pull command
- [ ] Test all download links

---

## Phase 6: Polish & Launch (Week 6)

### 6.1 Documentation

- [ ] CLI README with all commands
- [ ] `--help` text for every command
- [ ] Man pages generation
- [ ] Integration with docs.machpay.xyz
- [ ] Troubleshooting guide
- [ ] Video walkthrough

### 6.2 Testing

- [ ] Unit tests for all packages (>80% coverage)
- [ ] Integration tests for auth flow
- [ ] E2E tests for setup wizard
- [ ] Cross-platform testing matrix
- [ ] Performance benchmarks

### 6.3 Telemetry & Analytics

- [ ] Anonymous install tracking
- [ ] Command usage analytics (opt-in)
- [ ] Error reporting
- [ ] Crash reports

### 6.4 Launch Checklist

- [ ] All binaries built and tested
- [ ] Homebrew formula published
- [ ] Install script deployed
- [ ] Desktop apps signed and notarized
- [ ] Download page live
- [ ] Documentation complete
- [ ] Announcement blog post
- [ ] Social media assets
- [ ] Product Hunt submission ready

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Code signing delays | High | Start notarization process early |
| Homebrew core rejection | Medium | Use tap initially, apply to core later |
| Cross-platform bugs | High | Extensive testing matrix |
| Download server overload | Medium | Use GitHub releases + CDN |
| Gateway download failures | Medium | Retry logic + fallback mirrors |

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Install success rate | >95% | Telemetry |
| Time to first command | <2 min | User testing |
| Support tickets | <5% of installs | Zendesk |
| Homebrew stars | 100 in first month | GitHub |
| Desktop app adoption | 40% of users | Analytics |

---

## Appendix: Command Reference

```
machpay - MachPay CLI Orchestrator

USAGE:
    machpay [command]

COMMANDS:
    login       Authenticate with MachPay
    logout      Clear stored credentials
    setup       Interactive setup wizard
    serve       Start the vendor gateway
    status      Show current status
    open        Launch the console
    update      Update CLI and gateway
    version     Show version information

FLAGS:
    -h, --help      Show help
    -v, --verbose   Verbose output
    --debug         Debug mode
    --config        Config file path (default: ~/.machpay/config.yaml)

EXAMPLES:
    # First time setup
    machpay login
    machpay setup
    
    # Start vendor gateway
    machpay serve
    
    # Check status
    machpay status
    
    # Open console
    machpay open

DOCUMENTATION:
    https://docs.machpay.xyz/cli
```

---

## Timeline Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| Phase 1: Foundation | Week 1 | CLI repo, device auth, approval page |
| Phase 2: CLI Core | Week 2 | login, setup, status, open commands |
| Phase 3: Gateway | Week 3 | Downloader, process manager, serve command |
| Phase 4: Desktop | Week 4 | Tauri bundling, first launch flow |
| Phase 5: Distribution | Week 5 | Homebrew, install script, download page |
| Phase 6: Polish | Week 6 | Docs, testing, launch |

**Total: 6 weeks to launch** 🚀

---

*Document version: 1.0*  
*Let's fucking build this.*

