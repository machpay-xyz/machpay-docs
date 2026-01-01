# Phase 6: Polish & Launch - Detailed Implementation Prompts

> **Role:** You are the CTO of MachPay, preparing for the public launch of the CLI.
> **Context:** Phases 1-5 complete. The CLI is functional. Now we make it production-ready.
> **Priority:** P0 - Critical for launch
> **Timeline:** 1 week

---

## Pre-Launch Checklist

Before starting Phase 6, verify:
- [ ] CLI builds: `goreleaser build --snapshot --clean`
- [ ] All tests pass: `go test ./...`
- [ ] Install scripts work: Test on clean VM
- [ ] Desktop app builds: `npm run tauri:build`

---

## Prompt 6.1: CLI README

### Context
Create a comprehensive README that serves as the primary documentation entry point.

### Task
Create/update `README.md` in `machpay-cli`.

### Requirements

1. **Hero Section**
   - Project name and logo
   - One-line description
   - Badges (build status, version, license)

2. **Quick Start**
   - Installation (all methods)
   - First commands
   - 30-second demo

3. **Command Reference**
   - All commands with examples
   - Common flags
   - Configuration options

4. **Guides**
   - Agent setup guide
   - Vendor setup guide
   - Troubleshooting

5. **Development**
   - Building from source
   - Running tests
   - Contributing guidelines

### File Structure
```markdown
# MachPay CLI

[![Build](https://github.com/machpay/machpay-cli/actions/workflows/ci.yml/badge.svg)](...)
[![Release](https://img.shields.io/github/v/release/machpay/machpay-cli)](...)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](...)

The unified command-line interface for the MachPay AI payment network.

## Quick Start

### Installation

**Homebrew (macOS/Linux):**
\`\`\`bash
brew install machpay/tap/machpay
\`\`\`

**Curl:**
\`\`\`bash
curl -fsSL https://machpay.xyz/install.sh | sh
\`\`\`

**Windows:**
\`\`\`powershell
iwr machpay.xyz/install.ps1 | iex
\`\`\`

### First Steps

\`\`\`bash
# Authenticate with MachPay
machpay login

# Interactive setup
machpay setup

# Check status
machpay status
\`\`\`

## Commands

| Command | Description |
|---------|-------------|
| `login` | Authenticate with MachPay |
| `logout` | Clear stored credentials |
| `setup` | Interactive setup wizard |
| `status` | Show current status |
| `serve` | Start vendor gateway |
| `open` | Launch web console |
| `update` | Update CLI and gateway |
| `version` | Show version info |

### machpay login

Link your CLI to your MachPay account.

\`\`\`bash
machpay login              # Opens browser for authentication
machpay login --no-browser # Print URL instead
\`\`\`

### machpay setup

Interactive wizard to configure your node.

\`\`\`bash
machpay setup                    # Interactive mode
machpay setup --non-interactive  # CI/CD mode (uses env vars)
\`\`\`

**Environment Variables (non-interactive):**
- `MACHPAY_ROLE`: `agent` or `vendor`
- `MACHPAY_NETWORK`: `mainnet` or `devnet`
- `MACHPAY_UPSTREAM`: Upstream URL (vendor only)

### machpay serve

Start the vendor payment gateway.

\`\`\`bash
machpay serve                     # Foreground
machpay serve --detach            # Background (daemon)
machpay serve --port 9000         # Custom port
machpay serve --upstream http://localhost:11434
\`\`\`

### machpay status

Show current configuration and gateway status.

\`\`\`bash
machpay status           # Human-readable
machpay status --json    # JSON output
machpay status --watch   # Live updates
\`\`\`

## Configuration

Configuration is stored in `~/.machpay/config.yaml`:

\`\`\`yaml
auth:
  token: "eyJ..."

role: vendor
network: mainnet

vendor:
  upstream: http://localhost:11434
  port: 8402

wallet:
  path: ~/.machpay/wallet.json
\`\`\`

## Troubleshooting

### "command not found: machpay"

Add to your PATH:
\`\`\`bash
export PATH="$HOME/.local/bin:$PATH"
\`\`\`

### "not logged in"

Run `machpay login` to authenticate.

### Gateway won't start

1. Check if port is in use: `lsof -i :8402`
2. Check logs: `machpay logs`
3. Restart: `machpay restart`

## Development

### Build from source

\`\`\`bash
git clone https://github.com/machpay/machpay-cli.git
cd machpay-cli
go build -o machpay ./cmd/machpay
\`\`\`

### Run tests

\`\`\`bash
go test -v ./...
\`\`\`

### Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT - see [LICENSE](LICENSE).
```

### Acceptance Criteria
- [ ] All commands documented with examples
- [ ] Installation instructions for all platforms
- [ ] Troubleshooting section covers common issues
- [ ] Badges show build status

---

## Prompt 6.2: Help Text Enhancement

### Context
Ensure every command has excellent `--help` output with examples.

### Task
Review and enhance help text for all commands.

### Requirements

1. **Consistent Format**
   - Short description (one line)
   - Long description (optional)
   - Usage examples
   - Available flags

2. **Commands to Review**
   - `root.go` - Main help
   - `login.go`
   - `logout.go`
   - `setup.go`
   - `status.go`
   - `serve.go`
   - `open.go`
   - `update.go`
   - `logs.go`

### Example Enhancement
```go
var loginCmd = &cobra.Command{
    Use:   "login",
    Short: "Authenticate with MachPay",
    Long: `Link your CLI to your MachPay account.

This command opens a browser window for secure authentication.
After login, your credentials are saved locally.`,
    Example: `  # Standard login (opens browser)
  machpay login

  # Headless mode (for SSH/CI)
  machpay login --no-browser

  # Specify custom console URL
  machpay login --console-url https://console.machpay.xyz`,
    RunE: runLogin,
}
```

### Acceptance Criteria
- [ ] Every command has Short description
- [ ] Complex commands have Long description
- [ ] At least 2 examples per command
- [ ] Consistent formatting across all commands

---

## Prompt 6.3: Man Pages Generation

### Context
Generate Unix man pages for system-level documentation.

### Task
Add man page generation to the build process.

### Requirements

1. **Use cobra-doc or mango**
   - Generate man pages from Cobra commands
   - Include in release artifacts

2. **Installation**
   - Include in Homebrew formula
   - Symlink during install

### Implementation

```go
// cmd/machpay/docs.go

//go:build ignore

package main

import (
    "log"
    
    "github.com/spf13/cobra/doc"
    "github.com/machpay/machpay-cli/internal/cmd"
)

func main() {
    header := &doc.GenManHeader{
        Title:   "MACHPAY",
        Section: "1",
        Source:  "MachPay CLI",
        Manual:  "MachPay Manual",
    }
    
    err := doc.GenManTree(cmd.RootCmd(), header, "./man/")
    if err != nil {
        log.Fatal(err)
    }
}
```

```makefile
# Makefile

.PHONY: man
man:
	@mkdir -p man
	go run cmd/machpay/docs.go
```

### GoReleaser Addition
```yaml
# .goreleaser.yaml

archives:
  - files:
      - LICENSE
      - README.md
      - man/*
```

### Homebrew Formula Update
```ruby
def install
  bin.install "machpay"
  man1.install Dir["man/*.1"]
end
```

### Acceptance Criteria
- [ ] Man pages generated during build
- [ ] Included in release archives
- [ ] Homebrew installs man pages
- [ ] `man machpay` works after install

---

## Prompt 6.4: Comprehensive Test Suite

### Context
Achieve >80% test coverage with unit, integration, and E2E tests.

### Task
Expand test coverage across all packages.

### Requirements

1. **Unit Tests**
   - `internal/config/` - Config loading/saving
   - `internal/auth/` - Token handling
   - `internal/wallet/` - Key generation
   - `internal/gateway/` - Process management
   - `internal/tui/` - Prompt helpers

2. **Integration Tests**
   - Auth flow with mock server
   - Gateway download with mock GitHub API
   - Config persistence

3. **E2E Tests**
   - Full setup wizard flow
   - Gateway start/stop cycle

### Test Structure
```
internal/
├── auth/
│   ├── browser.go
│   ├── browser_test.go      # NEW
│   └── token_test.go        # NEW
├── config/
│   ├── config.go
│   └── config_test.go       # ENHANCE
├── gateway/
│   ├── downloader.go
│   ├── downloader_test.go   # ENHANCE
│   ├── process.go
│   └── process_test.go      # ENHANCE
└── wallet/
    ├── wallet.go
    └── wallet_test.go       # EXISTS
```

### Example: Auth Token Test
```go
// internal/auth/token_test.go

package auth

import (
    "testing"
    "time"
)

func TestParseToken(t *testing.T) {
    tests := []struct {
        name    string
        token   string
        wantErr bool
    }{
        {
            name:    "valid token",
            token:   generateTestToken(time.Hour),
            wantErr: false,
        },
        {
            name:    "expired token",
            token:   generateTestToken(-time.Hour),
            wantErr: true,
        },
        {
            name:    "malformed token",
            token:   "not.a.token",
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := ParseToken(tt.token)
            if (err != nil) != tt.wantErr {
                t.Errorf("ParseToken() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### Example: Gateway Download Test
```go
// internal/gateway/downloader_test.go

func TestDownloader_GetLatestVersion(t *testing.T) {
    // Mock GitHub API
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{
            "tag_name": "v1.2.3",
        })
    }))
    defer server.Close()
    
    d := NewDownloader()
    d.apiURL = server.URL
    
    version, err := d.GetLatestVersion()
    if err != nil {
        t.Fatalf("GetLatestVersion() error = %v", err)
    }
    
    if version != "v1.2.3" {
        t.Errorf("GetLatestVersion() = %v, want v1.2.3", version)
    }
}
```

### Coverage Targets
| Package | Current | Target |
|---------|---------|--------|
| `auth` | 40% | 80% |
| `config` | 60% | 85% |
| `gateway` | 50% | 80% |
| `wallet` | 90% | 90% |
| `tui` | 30% | 70% |
| `cmd` | 20% | 60% |

### Acceptance Criteria
- [ ] Overall coverage >80%
- [ ] All critical paths tested
- [ ] Mock servers for external dependencies
- [ ] Race condition tests (`-race` flag)

---

## Prompt 6.5: Telemetry System

### Context
Add opt-in anonymous telemetry for understanding usage patterns.

### Task
Implement telemetry collection that respects user privacy.

### Requirements

1. **Opt-in by Default**
   - Ask during `machpay setup`
   - Can be disabled anytime
   - Clear documentation on what's collected

2. **Data Collected**
   - Command invocations (no args)
   - OS/Arch
   - CLI version
   - Error types (no stack traces)

3. **What's NOT Collected**
   - Personal info
   - API keys
   - Wallet addresses
   - Command arguments

### Implementation

```go
// internal/telemetry/telemetry.go

package telemetry

import (
    "bytes"
    "encoding/json"
    "net/http"
    "os"
    "runtime"
    "sync"
    "time"
    
    "github.com/google/uuid"
)

type Client struct {
    enabled   bool
    endpoint  string
    sessionID string
    version   string
    mu        sync.Mutex
    events    []Event
}

type Event struct {
    Timestamp time.Time `json:"timestamp"`
    Event     string    `json:"event"`
    OS        string    `json:"os"`
    Arch      string    `json:"arch"`
    Version   string    `json:"version"`
    SessionID string    `json:"session_id"`
}

func New(version string) *Client {
    return &Client{
        enabled:   isEnabled(),
        endpoint:  "https://telemetry.machpay.xyz/v1/events",
        sessionID: uuid.New().String()[:8],
        version:   version,
    }
}

func isEnabled() bool {
    // Check config
    if os.Getenv("MACHPAY_TELEMETRY") == "false" {
        return false
    }
    // Check config file
    // ...
    return true
}

func (c *Client) Track(event string) {
    if !c.enabled {
        return
    }
    
    c.mu.Lock()
    c.events = append(c.events, Event{
        Timestamp: time.Now(),
        Event:     event,
        OS:        runtime.GOOS,
        Arch:      runtime.GOARCH,
        Version:   c.version,
        SessionID: c.sessionID,
    })
    c.mu.Unlock()
}

func (c *Client) Flush() {
    if !c.enabled || len(c.events) == 0 {
        return
    }
    
    c.mu.Lock()
    events := c.events
    c.events = nil
    c.mu.Unlock()
    
    // Send in background
    go func() {
        data, _ := json.Marshal(events)
        http.Post(c.endpoint, "application/json", bytes.NewReader(data))
    }()
}
```

### Config Option
```yaml
# ~/.machpay/config.yaml
telemetry:
  enabled: true  # or false
```

### Privacy Notice
```
Telemetry helps us improve MachPay CLI.

We collect:
- Command names (not arguments)
- OS and architecture
- CLI version
- Anonymous session ID

We DO NOT collect:
- Personal information
- API keys or tokens
- Wallet addresses
- File paths or arguments

To disable: machpay config set telemetry.enabled false
```

### Acceptance Criteria
- [ ] Opt-in during setup
- [ ] Easy to disable
- [ ] No PII collected
- [ ] Non-blocking (async)
- [ ] Works offline (queues events)

---

## Prompt 6.6: Error Reporting

### Context
Capture and report errors to improve reliability.

### Task
Implement error reporting with user consent.

### Requirements

1. **Capture**
   - Panics
   - Unhandled errors
   - Failed commands

2. **Include**
   - Error message
   - OS/Arch
   - CLI version
   - Command name

3. **Exclude**
   - Stack traces with file paths
   - Environment variables
   - Arguments

### Implementation

```go
// internal/errors/reporter.go

package errors

import (
    "bytes"
    "encoding/json"
    "net/http"
    "runtime"
    "time"
)

type ErrorReport struct {
    Timestamp   time.Time `json:"timestamp"`
    Message     string    `json:"message"`
    Command     string    `json:"command"`
    OS          string    `json:"os"`
    Arch        string    `json:"arch"`
    Version     string    `json:"version"`
    ErrorType   string    `json:"error_type"`
}

var reporter *Reporter

type Reporter struct {
    enabled  bool
    endpoint string
    version  string
}

func Init(version string, enabled bool) {
    reporter = &Reporter{
        enabled:  enabled,
        endpoint: "https://telemetry.machpay.xyz/v1/errors",
        version:  version,
    }
}

func Report(cmd string, err error) {
    if reporter == nil || !reporter.enabled {
        return
    }
    
    report := ErrorReport{
        Timestamp: time.Now(),
        Message:   sanitize(err.Error()),
        Command:   cmd,
        OS:        runtime.GOOS,
        Arch:      runtime.GOARCH,
        Version:   reporter.version,
        ErrorType: getErrorType(err),
    }
    
    // Send async
    go func() {
        data, _ := json.Marshal(report)
        http.Post(reporter.endpoint, "application/json", bytes.NewReader(data))
    }()
}

func sanitize(msg string) string {
    // Remove file paths, tokens, etc.
    // ...
    return msg
}
```

### Acceptance Criteria
- [ ] Respects telemetry setting
- [ ] Sanitizes error messages
- [ ] Non-blocking
- [ ] Rate limited (max 10/min)

---

## Prompt 6.7: Launch Checklist Automation

### Context
Create automated verification for launch readiness.

### Task
Create a launch checklist script that verifies everything is ready.

### Requirements

1. **Binary Verification**
   - All platforms build
   - Version correct
   - Signatures valid

2. **Distribution Verification**
   - GitHub release exists
   - Homebrew formula works
   - Install scripts work
   - Docker image works

3. **Documentation Verification**
   - README complete
   - All links valid
   - Examples work

### Script

```bash
#!/bin/bash
# scripts/launch-check.sh

set -e

VERSION="${1:-latest}"
PASS=0
FAIL=0

check() {
    local name="$1"
    local cmd="$2"
    
    printf "Checking %s... " "$name"
    if eval "$cmd" >/dev/null 2>&1; then
        echo "✓"
        ((PASS++))
    else
        echo "✗"
        ((FAIL++))
    fi
}

echo "=========================================="
echo "MachPay CLI Launch Checklist"
echo "Version: $VERSION"
echo "=========================================="
echo ""

echo "## Binaries"
check "Darwin/amd64" "curl -sLI https://github.com/machpay/machpay-cli/releases/download/$VERSION/machpay_darwin_amd64.tar.gz | grep '200'"
check "Darwin/arm64" "curl -sLI https://github.com/machpay/machpay-cli/releases/download/$VERSION/machpay_darwin_arm64.tar.gz | grep '200'"
check "Linux/amd64" "curl -sLI https://github.com/machpay/machpay-cli/releases/download/$VERSION/machpay_linux_amd64.tar.gz | grep '200'"
check "Linux/arm64" "curl -sLI https://github.com/machpay/machpay-cli/releases/download/$VERSION/machpay_linux_arm64.tar.gz | grep '200'"
check "Windows/amd64" "curl -sLI https://github.com/machpay/machpay-cli/releases/download/$VERSION/machpay_windows_amd64.zip | grep '200'"
check "Checksums" "curl -sL https://github.com/machpay/machpay-cli/releases/download/$VERSION/checksums.txt | grep -q machpay"
echo ""

echo "## Distribution"
check "Homebrew tap" "brew tap machpay/tap 2>/dev/null"
check "Homebrew install" "brew info machpay/tap/machpay 2>/dev/null"
check "Install script" "curl -sL https://raw.githubusercontent.com/machpay/machpay-cli/main/scripts/install.sh | head -1 | grep -q '#!'"
check "Docker image" "docker manifest inspect ghcr.io/machpay/cli:$VERSION 2>/dev/null"
echo ""

echo "## Documentation"
check "README exists" "curl -sL https://raw.githubusercontent.com/machpay/machpay-cli/main/README.md | grep -q 'MachPay'"
check "LICENSE exists" "curl -sL https://raw.githubusercontent.com/machpay/machpay-cli/main/LICENSE | grep -q 'MIT'"
echo ""

echo "## Console Integration"
check "CLI auth page" "curl -sL https://console.machpay.xyz/auth/cli | grep -q 'html'"
echo ""

echo "=========================================="
echo "Results: $PASS passed, $FAIL failed"
echo "=========================================="

if [ $FAIL -gt 0 ]; then
    exit 1
fi
```

### GitHub Action
```yaml
# .github/workflows/launch-check.yml

name: Launch Check

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to check'
        required: true

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run launch check
        run: ./scripts/launch-check.sh ${{ github.event.inputs.version }}
```

### Acceptance Criteria
- [ ] All binaries verified
- [ ] All distribution channels verified
- [ ] Documentation verified
- [ ] Can run from CI

---

## Prompt 6.8: Performance Benchmarks

### Context
Establish baseline performance metrics.

### Task
Create benchmark tests for critical paths.

### Requirements

1. **Startup Time**
   - Target: <100ms for simple commands

2. **Gateway Operations**
   - Download speed
   - Start time

3. **Config Operations**
   - Load time
   - Save time

### Implementation

```go
// internal/cmd/benchmark_test.go

package cmd

import (
    "testing"
)

func BenchmarkStatusCommand(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // Simulate status command
        runStatus(nil, nil)
    }
}

func BenchmarkConfigLoad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        config.Load()
    }
}
```

### Benchmark Script
```bash
#!/bin/bash
# scripts/benchmark.sh

echo "MachPay CLI Benchmarks"
echo "======================"

# Startup time
echo ""
echo "## Startup Time"
time machpay version >/dev/null
time machpay --help >/dev/null

# Command execution
echo ""
echo "## Command Execution"
time machpay status >/dev/null 2>&1

# Run Go benchmarks
echo ""
echo "## Go Benchmarks"
go test -bench=. -benchmem ./...
```

### Acceptance Criteria
- [ ] Baseline metrics established
- [ ] No regressions in CI
- [ ] Startup <100ms

---

## Summary

| Prompt | Deliverable | Priority |
|--------|-------------|----------|
| 6.1 | README.md | P0 |
| 6.2 | Enhanced help text | P0 |
| 6.3 | Man pages | P2 |
| 6.4 | Test suite (>80%) | P0 |
| 6.5 | Telemetry system | P1 |
| 6.6 | Error reporting | P1 |
| 6.7 | Launch checklist | P0 |
| 6.8 | Performance benchmarks | P2 |

---

## Execution Order

1. **6.1** README.md (documentation foundation)
2. **6.2** Help text (user-facing)
3. **6.4** Test suite (quality assurance)
4. **6.7** Launch checklist (verification)
5. **6.5 + 6.6** Telemetry & errors (parallel)
6. **6.3** Man pages
7. **6.8** Benchmarks

---

## Launch Day Checklist

### 24 Hours Before
- [ ] All tests pass
- [ ] Launch check script passes
- [ ] Announcement drafted
- [ ] Social media scheduled

### Launch
- [ ] Tag release
- [ ] Verify all downloads work
- [ ] Monitor error reports
- [ ] Post announcement
- [ ] Update website

### Post-Launch
- [ ] Monitor telemetry
- [ ] Respond to issues
- [ ] Collect feedback
- [ ] Plan next iteration

---

**Estimated Time:** 5-7 days
**Priority:** P0 - Critical for launch

*Let's ship this thing.* 🚀

