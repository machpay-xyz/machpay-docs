# MachPay CLI - Phase 2 Implementation Prompts

> **Role Setup:** You are the CTO and Lead Architect of MachPay, building a revolutionary AI payment platform. You have zero tolerance for technical debt, dead code, or hacky solutions. Every line of code should be production-ready, well-tested, and follow Go best practices.

---

## Phase 2 Overview

**Goal:** Interactive setup wizard and utility commands

| Prompt | Component | Description |
|--------|-----------|-------------|
| 2.1 | TUI Prompts | Bubble Tea interactive prompt components |
| 2.2 | Setup Command | `machpay setup` wizard |
| 2.3 | Wallet Generation | Solana keypair generation/import |
| 2.4 | API Key Creation | Backend integration for API keys |
| 2.5 | Open Command | `machpay open` with route support |
| 2.6 | Enhanced Status | `machpay status --json --watch` |
| 2.7 | Integration Tests | E2E tests for setup flow |

---

## Prompt 2.1: TUI Prompt Components

### Context

We need reusable Bubble Tea components for the interactive setup wizard. These will be used across multiple commands and should provide a consistent, beautiful terminal experience.

### Task

Create `internal/tui/prompts.go` with the following components:

### Requirements

1. **SelectPrompt** - Single selection from list
   ```go
   type SelectPrompt struct {
       Question string
       Options  []SelectOption
       Selected int
   }
   
   type SelectOption struct {
       Label       string
       Description string
       Value       string
   }
   ```

2. **TextInputPrompt** - Text input with validation
   ```go
   type TextInputPrompt struct {
       Question    string
       Placeholder string
       Validator   func(string) error
       Value       string
   }
   ```

3. **ConfirmPrompt** - Yes/No confirmation
   ```go
   type ConfirmPrompt struct {
       Question string
       Default  bool
   }
   ```

### Implementation Details

```go
// internal/tui/prompts.go

package tui

import (
    "github.com/charmbracelet/bubbles/textinput"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
)

// SelectPrompt allows user to select from a list of options
type SelectPrompt struct {
    question string
    options  []SelectOption
    cursor   int
    done     bool
    aborted  bool
}

func NewSelectPrompt(question string, options []SelectOption) *SelectPrompt {
    return &SelectPrompt{
        question: question,
        options:  options,
    }
}

func (m SelectPrompt) Init() tea.Cmd {
    return nil
}

func (m SelectPrompt) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        case "ctrl+c", "q":
            m.aborted = true
            return m, tea.Quit
        case "up", "k":
            if m.cursor > 0 {
                m.cursor--
            }
        case "down", "j":
            if m.cursor < len(m.options)-1 {
                m.cursor++
            }
        case "enter":
            m.done = true
            return m, tea.Quit
        }
    }
    return m, nil
}

func (m SelectPrompt) View() string {
    // Beautiful rendering with lipgloss
    // Show question, options with cursor indicator
    // Selected option highlighted in emerald
}

func (m SelectPrompt) Selected() (SelectOption, bool) {
    if m.aborted {
        return SelectOption{}, false
    }
    return m.options[m.cursor], true
}
```

### Styling

- Use the existing `internal/tui/styles.go` color palette
- Question text: Bold white
- Options: Muted gray, selected option in emerald
- Cursor: `❯` character in emerald
- Description: Smaller, muted text below option

### Files to Create

1. `internal/tui/prompts.go` - Core prompt components
2. `internal/tui/prompts_test.go` - Unit tests

### Acceptance Criteria

- [ ] SelectPrompt with keyboard navigation (j/k, up/down)
- [ ] TextInputPrompt with real-time validation
- [ ] ConfirmPrompt with Y/n default handling
- [ ] All prompts support Ctrl+C to abort
- [ ] Consistent styling matching app theme
- [ ] Unit tests for each component

---

## Prompt 2.2: Setup Command Core

### Context

The `machpay setup` command is an interactive wizard that guides users through initial configuration. It must handle both agent and vendor roles with different flows.

### Task

Create `internal/cmd/setup.go` with the interactive setup wizard.

### Requirements

1. **Check Prerequisites**
   - Must be logged in (show error if not)
   - Check if already configured (offer to reconfigure)

2. **Role Selection**
   ```
   ? What do you want to do?
     ❯ Run an AI Agent (I want to use APIs)
       Run a Vendor Node (I want to sell APIs)
   ```

3. **Network Selection**
   ```
   ? Select network:
     ❯ Devnet (Testing - free tokens)
       Mainnet (Real money)
   ```

4. **Branch by Role**
   - Agent: Wallet setup → API key generation → Show quick start
   - Vendor: Service info → Wallet → Registration → Show next steps

### Implementation Structure

```go
// internal/cmd/setup.go

package cmd

import (
    "github.com/spf13/cobra"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/machpay/machpay-cli/internal/tui"
    "github.com/machpay/machpay-cli/internal/auth"
    "github.com/machpay/machpay-cli/internal/config"
)

var setupNonInteractive bool

var setupCmd = &cobra.Command{
    Use:   "setup",
    Short: "Interactive setup wizard",
    Long:  `Configure your MachPay CLI as an agent or vendor.`,
    RunE:  runSetup,
}

func init() {
    setupCmd.Flags().BoolVar(&setupNonInteractive, "non-interactive", false, 
        "Use environment variables instead of prompts (for CI/CD)")
}

func runSetup(cmd *cobra.Command, args []string) error {
    // 1. Check if logged in
    if !auth.IsLoggedIn() {
        return fmt.Errorf("not logged in. Run 'machpay login' first")
    }

    // 2. Show banner
    printSetupBanner()

    // 3. Handle non-interactive mode
    if setupNonInteractive {
        return runNonInteractiveSetup()
    }

    // 4. Run interactive wizard
    return runInteractiveSetup()
}

func runInteractiveSetup() error {
    // Step 1: Role selection
    role, err := promptRole()
    if err != nil {
        return err
    }

    // Step 2: Network selection
    network, err := promptNetwork()
    if err != nil {
        return err
    }

    // Step 3: Branch by role
    switch role {
    case "agent":
        return setupAgent(network)
    case "vendor":
        return setupVendor(network)
    }
    
    return nil
}

func setupAgent(network string) error {
    // 1. Wallet setup
    // 2. Generate API key via backend
    // 3. Save config
    // 4. Show quick start
}

func setupVendor(network string) error {
    // 1. Collect service info (name, description, category)
    // 2. Collect upstream URL and pricing
    // 3. Wallet setup
    // 4. Register with backend
    // 5. Save config
    // 6. Show next steps
}
```

### ASCII Art Banner

```go
func printSetupBanner() {
    banner := `
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
`
    fmt.Print(tui.Primary(banner))
}
```

### Non-Interactive Mode

Support environment variables for CI/CD:
```bash
MACHPAY_ROLE=agent \
MACHPAY_NETWORK=devnet \
MACHPAY_WALLET_PATH=~/.machpay/wallet.json \
machpay setup --non-interactive
```

### Files to Create/Modify

1. `internal/cmd/setup.go` - Main setup command
2. `internal/cmd/root.go` - Uncomment/add setupCmd

### Acceptance Criteria

- [ ] Login check with helpful error message
- [ ] Role selection with clear descriptions
- [ ] Network selection with risk warning for mainnet
- [ ] Proper branching to agent/vendor flows
- [ ] Non-interactive mode with env vars
- [ ] Beautiful ASCII banner
- [ ] Ctrl+C handling at any step

---

## Prompt 2.3: Wallet Generation

### Context

Both agents and vendors need a Solana wallet. We need to support generating new keypairs and importing existing ones.

### Task

Create `internal/wallet/wallet.go` for wallet management.

### Requirements

1. **Generate New Keypair**
   - Generate Ed25519 keypair
   - Save to `~/.machpay/wallet.json` (Solana CLI format)
   - Show public key and backup warning

2. **Import Existing Keypair**
   - Prompt for file path
   - Validate keypair file format
   - Copy to MachPay config directory

3. **Keypair File Format** (Solana CLI compatible)
   ```json
   [1,2,3,...,64]  // 64-byte array (32 private + 32 public)
   ```

### Implementation

```go
// internal/wallet/wallet.go

package wallet

import (
    "crypto/ed25519"
    "crypto/rand"
    "encoding/json"
    "fmt"
    "os"
    "path/filepath"
)

// Keypair represents a Solana Ed25519 keypair
type Keypair struct {
    PublicKey  ed25519.PublicKey
    PrivateKey ed25519.PrivateKey
}

// Generate creates a new random keypair
func Generate() (*Keypair, error) {
    pub, priv, err := ed25519.GenerateKey(rand.Reader)
    if err != nil {
        return nil, fmt.Errorf("generate keypair: %w", err)
    }
    return &Keypair{
        PublicKey:  pub,
        PrivateKey: priv,
    }, nil
}

// LoadFromFile loads a keypair from a Solana CLI format file
func LoadFromFile(path string) (*Keypair, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read file: %w", err)
    }

    var bytes []byte
    if err := json.Unmarshal(data, &bytes); err != nil {
        return nil, fmt.Errorf("parse keypair: %w", err)
    }

    if len(bytes) != 64 {
        return nil, fmt.Errorf("invalid keypair length: got %d, want 64", len(bytes))
    }

    return &Keypair{
        PrivateKey: ed25519.PrivateKey(bytes),
        PublicKey:  ed25519.PublicKey(bytes[32:]),
    }, nil
}

// SaveToFile saves the keypair in Solana CLI format
func (k *Keypair) SaveToFile(path string) error {
    // Ensure directory exists
    dir := filepath.Dir(path)
    if err := os.MkdirAll(dir, 0700); err != nil {
        return fmt.Errorf("create directory: %w", err)
    }

    // Marshal as JSON array of bytes
    data, err := json.Marshal([]byte(k.PrivateKey))
    if err != nil {
        return fmt.Errorf("marshal keypair: %w", err)
    }

    // Write with secure permissions (600)
    if err := os.WriteFile(path, data, 0600); err != nil {
        return fmt.Errorf("write file: %w", err)
    }

    return nil
}

// PublicKeyBase58 returns the base58-encoded public key (Solana address)
func (k *Keypair) PublicKeyBase58() string {
    return base58Encode(k.PublicKey)
}

// base58Encode encodes bytes to base58 (Bitcoin/Solana alphabet)
func base58Encode(data []byte) string {
    // Implement base58 encoding or use a library
}
```

### Setup Integration

```go
// In internal/cmd/setup.go

func promptWallet() (*wallet.Keypair, error) {
    // Ask: Generate new or Import existing?
    choice, err := tui.RunSelect("Wallet setup:", []tui.SelectOption{
        {Label: "Generate new wallet", Value: "generate", Description: "Recommended for new users"},
        {Label: "Import existing keypair", Value: "import", Description: "Use existing Solana wallet"},
    })
    if err != nil {
        return nil, err
    }

    switch choice.Value {
    case "generate":
        return generateNewWallet()
    case "import":
        return importExistingWallet()
    }
    return nil, fmt.Errorf("invalid choice")
}

func generateNewWallet() (*wallet.Keypair, error) {
    kp, err := wallet.Generate()
    if err != nil {
        return nil, err
    }

    walletPath := filepath.Join(config.GetDir(), "wallet.json")
    if err := kp.SaveToFile(walletPath); err != nil {
        return nil, err
    }

    // Show success with warning
    fmt.Println()
    tui.PrintSuccess("Generated new wallet")
    fmt.Println()
    fmt.Printf("   Address: %s\n", tui.Primary(kp.PublicKeyBase58()))
    fmt.Printf("   Saved to: %s\n", tui.Muted(walletPath))
    fmt.Println()
    fmt.Println(tui.Warning("⚠️  BACKUP THIS FILE! It contains your private key."))
    fmt.Println()

    return kp, nil
}
```

### Files to Create

1. `internal/wallet/wallet.go` - Wallet management
2. `internal/wallet/base58.go` - Base58 encoding (or use library)
3. `internal/wallet/wallet_test.go` - Unit tests

### Acceptance Criteria

- [ ] Generate valid Ed25519 keypair
- [ ] Save in Solana CLI compatible format
- [ ] Load and validate existing keypair files
- [ ] Secure file permissions (0600)
- [ ] Clear backup warning displayed
- [ ] Base58 address encoding matches Solana
- [ ] Unit tests for generate/load/save

---

## Prompt 2.4: API Key Creation (Backend Integration)

### Context

After wallet setup, agents need an API key for the MachPay SDK. This requires a backend API call.

### Task

Create `internal/api/client.go` for backend communication and implement API key generation.

### Backend Endpoint (Already Exists)

```
POST /v1/agent/apikey
Authorization: Bearer <access_token>

Request:
{
  "wallet_address": "7xK9mN3pQr...",
  "name": "CLI Generated Key"
}

Response:
{
  "api_key": "mp_live_7xK9...",
  "created_at": "2024-01-01T00:00:00Z"
}
```

### Implementation

```go
// internal/api/client.go

package api

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"

    "github.com/machpay/machpay-cli/internal/config"
    "github.com/machpay/machpay-cli/internal/auth"
)

// Client is the MachPay API client
type Client struct {
    baseURL    string
    httpClient *http.Client
}

// NewClient creates a new API client
func NewClient() *Client {
    cfg := config.Get()
    baseURL := "https://api.machpay.xyz"
    if cfg.Network == "devnet" {
        baseURL = "https://api-dev.machpay.xyz"
    }

    return &Client{
        baseURL: baseURL,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

// CreateAPIKey generates a new API key for the agent
func (c *Client) CreateAPIKey(walletAddress, name string) (*APIKey, error) {
    token := auth.GetToken()
    if token == "" {
        return nil, fmt.Errorf("not authenticated")
    }

    payload := map[string]string{
        "wallet_address": walletAddress,
        "name":           name,
    }

    body, err := json.Marshal(payload)
    if err != nil {
        return nil, fmt.Errorf("marshal request: %w", err)
    }

    req, err := http.NewRequest("POST", c.baseURL+"/v1/agent/apikey", bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+token)

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("send request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK && resp.StatusCode != http.StatusCreated {
        body, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("API error (%d): %s", resp.StatusCode, string(body))
    }

    var result APIKey
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("decode response: %w", err)
    }

    return &result, nil
}

// APIKey represents an API key response
type APIKey struct {
    Key       string    `json:"api_key"`
    CreatedAt time.Time `json:"created_at"`
}
```

### Setup Integration

```go
// In setupAgent()

func setupAgent(network string) error {
    // 1. Wallet setup
    kp, err := promptWallet()
    if err != nil {
        return err
    }

    // 2. Generate API key
    fmt.Println(tui.Muted("Generating API key..."))
    
    client := api.NewClient()
    apiKey, err := client.CreateAPIKey(kp.PublicKeyBase58(), "CLI Generated")
    if err != nil {
        return fmt.Errorf("create API key: %w", err)
    }

    // 3. Save config
    cfg := config.Get()
    cfg.Role = "agent"
    cfg.Network = network
    cfg.Wallet.KeypairPath = filepath.Join(config.GetDir(), "wallet.json")
    cfg.Wallet.PublicKey = kp.PublicKeyBase58()
    // Note: API key stored separately or in keyring for security

    if err := config.Save(); err != nil {
        return fmt.Errorf("save config: %w", err)
    }

    // 4. Show success
    printAgentSuccess(kp.PublicKeyBase58(), apiKey.Key)

    return nil
}

func printAgentSuccess(address, apiKey string) {
    // Mask API key for display
    masked := apiKey[:12] + "████████" + apiKey[len(apiKey)-4:]

    fmt.Println()
    tui.PrintSuccess("Agent setup complete!")
    fmt.Println()
    fmt.Printf("   Wallet: %s\n", tui.Primary(address))
    fmt.Printf("   API Key: %s\n", tui.Muted(masked))
    fmt.Println()
    
    // Quick start box
    fmt.Println(tui.Box(`
pip install machpay

from machpay import MachPay
client = MachPay()  # Uses ~/.machpay config
response = client.call("weather-api", "/forecast")
`))
    
    fmt.Println()
    fmt.Printf("   Next: Fund your wallet at %s\n", 
        tui.Primary(config.GetConsoleURL()+"/agent/fund"))
}
```

### Files to Create

1. `internal/api/client.go` - API client
2. `internal/api/agent.go` - Agent-specific endpoints
3. `internal/api/vendor.go` - Vendor-specific endpoints
4. `internal/api/client_test.go` - Tests with mock server

### Acceptance Criteria

- [ ] API client with auth headers
- [ ] Network-aware base URL selection
- [ ] Proper error handling for API failures
- [ ] API key masked in terminal output
- [ ] Quick start guide displayed
- [ ] Config saved with wallet info
- [ ] Unit tests with HTTP mocks

---

## Prompt 2.5: Open Command

### Context

The `machpay open` command launches the MachPay console, either as a desktop app (if installed) or in the browser.

### Task

Create `internal/cmd/open.go` with route support.

### Requirements

1. **Basic Usage**
   ```bash
   machpay open              # Opens console home
   machpay open marketplace  # Opens marketplace page
   machpay open funding      # Opens funding page
   machpay open --web        # Force browser (not desktop app)
   ```

2. **Platform Detection**
   - macOS: Check for `/Applications/MachPay.app`
   - Windows: Check for `%LOCALAPPDATA%\MachPay\MachPay.exe`
   - Linux: Check for `~/.local/share/applications/machpay.desktop`

3. **Route Mapping**
   ```go
   var routes = map[string]string{
       "":            "/",
       "home":        "/",
       "marketplace": "/marketplace",
       "funding":     "/agent/finance",
       "settings":    "/settings",
       "analytics":   "/agent/analytics",
       "endpoints":   "/vendor/endpoints",
   }
   ```

### Implementation

```go
// internal/cmd/open.go

package cmd

import (
    "fmt"
    "os"
    "os/exec"
    "runtime"

    "github.com/pkg/browser"
    "github.com/spf13/cobra"

    "github.com/machpay/machpay-cli/internal/config"
    "github.com/machpay/machpay-cli/internal/tui"
)

var openWeb bool

var openCmd = &cobra.Command{
    Use:   "open [route]",
    Short: "Launch MachPay Console",
    Long: `Open the MachPay Console in your default browser or desktop app.

Available routes:
  (none)       Home / Dashboard
  marketplace  API Marketplace
  funding      Fund your wallet
  settings     Account settings
  analytics    Usage analytics
  endpoints    Vendor endpoints`,
    Args: cobra.MaximumNArgs(1),
    RunE: runOpen,
}

func init() {
    openCmd.Flags().BoolVar(&openWeb, "web", false, "Force open in browser (not desktop app)")
}

var routes = map[string]string{
    "":            "/explorer",
    "home":        "/explorer",
    "marketplace": "/marketplace",
    "funding":     "/agent/finance",
    "settings":    "/settings",
    "analytics":   "/agent/analytics",
    "endpoints":   "/vendor/endpoints",
}

func runOpen(cmd *cobra.Command, args []string) error {
    // Determine route
    route := ""
    if len(args) > 0 {
        route = args[0]
    }

    path, ok := routes[route]
    if !ok {
        return fmt.Errorf("unknown route: %s\nRun 'machpay open --help' for available routes", route)
    }

    // Build full URL
    consoleURL := config.GetConsoleURL() + path

    // Try desktop app first (unless --web)
    if !openWeb {
        if launchDesktopApp(path) {
            tui.PrintSuccess("Opened MachPay Console")
            return nil
        }
    }

    // Fall back to browser
    fmt.Printf("Opening %s...\n", tui.Primary(consoleURL))
    if err := browser.OpenURL(consoleURL); err != nil {
        return fmt.Errorf("failed to open browser: %w", err)
    }

    tui.PrintSuccess("Opened in browser")
    return nil
}

func launchDesktopApp(route string) bool {
    appPath := getDesktopAppPath()
    if appPath == "" {
        return false
    }

    // Check if app exists
    if _, err := os.Stat(appPath); os.IsNotExist(err) {
        return false
    }

    // Launch with deep link
    var cmd *exec.Cmd
    switch runtime.GOOS {
    case "darwin":
        // macOS: open -a "MachPay" --args --route=/marketplace
        cmd = exec.Command("open", "-a", appPath, "--args", "--route="+route)
    case "windows":
        // Windows: Start-Process with arguments
        cmd = exec.Command("cmd", "/c", "start", "", appPath, "--route="+route)
    case "linux":
        // Linux: Just run the binary
        cmd = exec.Command(appPath, "--route="+route)
    }

    if cmd == nil {
        return false
    }

    return cmd.Start() == nil
}

func getDesktopAppPath() string {
    switch runtime.GOOS {
    case "darwin":
        return "/Applications/MachPay.app"
    case "windows":
        return os.Getenv("LOCALAPPDATA") + "\\MachPay\\MachPay.exe"
    case "linux":
        home, _ := os.UserHomeDir()
        return home + "/.local/bin/machpay-console"
    default:
        return ""
    }
}
```

### Files to Create/Modify

1. `internal/cmd/open.go` - Open command
2. `internal/cmd/root.go` - Add openCmd

### Acceptance Criteria

- [ ] Opens console in browser by default
- [ ] Route argument support (marketplace, funding, etc.)
- [ ] Desktop app detection and launch (macOS, Windows, Linux)
- [ ] `--web` flag to force browser
- [ ] Helpful error for unknown routes
- [ ] Shows URL being opened

---

## Prompt 2.6: Enhanced Status Command

### Context

The status command from Phase 1 needs enhancements for scriptability and monitoring.

### Task

Enhance `internal/cmd/status.go` with JSON output and watch mode.

### Requirements

1. **JSON Output**
   ```bash
   machpay status --json | jq '.wallet.balance'
   ```

2. **Watch Mode**
   ```bash
   machpay status --watch  # Updates every 5 seconds
   ```

3. **Balance Fetching**
   - Connect to Solana RPC
   - Fetch SOL and USDC balances
   - Cache for 30 seconds

### Implementation

```go
// internal/cmd/status.go additions

var (
    statusJSON  bool
    statusWatch bool
)

func init() {
    statusCmd.Flags().BoolVar(&statusJSON, "json", false, "Output as JSON")
    statusCmd.Flags().BoolVar(&statusWatch, "watch", false, "Continuous monitoring (every 5s)")
}

type StatusOutput struct {
    Auth struct {
        LoggedIn bool   `json:"logged_in"`
        Email    string `json:"email,omitempty"`
    } `json:"auth"`
    Config struct {
        Role    string `json:"role"`
        Network string `json:"network"`
    } `json:"config"`
    Wallet struct {
        Address string  `json:"address,omitempty"`
        SOL     float64 `json:"sol_balance,omitempty"`
        USDC    float64 `json:"usdc_balance,omitempty"`
    } `json:"wallet,omitempty"`
    Gateway struct {
        Running bool   `json:"running"`
        PID     int    `json:"pid,omitempty"`
        Version string `json:"version,omitempty"`
        Port    int    `json:"port,omitempty"`
        Uptime  string `json:"uptime,omitempty"`
    } `json:"gateway,omitempty"`
}

func runStatus(cmd *cobra.Command, args []string) error {
    if statusWatch {
        return runStatusWatch()
    }
    
    return runStatusOnce()
}

func runStatusOnce() error {
    status := gatherStatus()
    
    if statusJSON {
        return outputJSON(status)
    }
    
    return outputHuman(status)
}

func runStatusWatch() error {
    // Clear screen, show status, refresh every 5s
    // Handle Ctrl+C to exit
}

func gatherStatus() *StatusOutput {
    status := &StatusOutput{}
    
    // Auth
    status.Auth.LoggedIn = auth.IsLoggedIn()
    if user := auth.GetUser(); user != nil {
        status.Auth.Email = user.Email
    }
    
    // Config
    cfg := config.Get()
    status.Config.Role = cfg.Role
    status.Config.Network = cfg.Network
    
    // Wallet balance (if configured)
    if cfg.Wallet.PublicKey != "" {
        status.Wallet.Address = cfg.Wallet.PublicKey
        // Fetch balances from RPC (with caching)
        sol, usdc := fetchBalances(cfg.Wallet.PublicKey, cfg.Network)
        status.Wallet.SOL = sol
        status.Wallet.USDC = usdc
    }
    
    // Gateway (if vendor)
    if cfg.Role == "vendor" {
        status.Gateway = getGatewayStatus()
    }
    
    return status
}
```

### Files to Modify

1. `internal/cmd/status.go` - Add flags and JSON/watch modes
2. `internal/solana/rpc.go` (new) - RPC client for balance queries

### Acceptance Criteria

- [ ] `--json` flag outputs valid JSON
- [ ] `--watch` mode updates every 5 seconds
- [ ] SOL and USDC balance fetching
- [ ] Balance caching (30s TTL)
- [ ] Clean exit on Ctrl+C in watch mode
- [ ] Human-readable format unchanged for default

---

## Prompt 2.7: Integration Tests

### Context

Phase 2 introduces complex interactive flows that need E2E testing.

### Task

Create integration tests for the setup wizard and other Phase 2 commands.

### Test Structure

```go
// internal/cmd/setup_test.go

package cmd

import (
    "os"
    "path/filepath"
    "testing"
    
    "github.com/machpay/machpay-cli/internal/config"
)

func TestSetupNonInteractive_Agent(t *testing.T) {
    // Setup temp directory
    tmpDir := t.TempDir()
    os.Setenv("HOME", tmpDir)
    os.Setenv("MACHPAY_ROLE", "agent")
    os.Setenv("MACHPAY_NETWORK", "devnet")
    
    // Run setup in non-interactive mode
    err := runNonInteractiveSetup()
    if err != nil {
        t.Fatalf("Setup failed: %v", err)
    }
    
    // Verify config
    cfg := config.Get()
    if cfg.Role != "agent" {
        t.Errorf("Role = %s, want agent", cfg.Role)
    }
    if cfg.Network != "devnet" {
        t.Errorf("Network = %s, want devnet", cfg.Network)
    }
    
    // Verify wallet was created
    walletPath := filepath.Join(tmpDir, ".machpay", "wallet.json")
    if _, err := os.Stat(walletPath); os.IsNotExist(err) {
        t.Error("Wallet file not created")
    }
}

func TestSetupNonInteractive_Vendor(t *testing.T) {
    // Similar test for vendor flow
}

func TestSetup_RequiresLogin(t *testing.T) {
    // Ensure setup fails if not logged in
}
```

### Test Files to Create

1. `internal/cmd/setup_test.go` - Setup command tests
2. `internal/cmd/open_test.go` - Open command tests
3. `internal/wallet/wallet_test.go` - Wallet generation tests
4. `internal/api/client_test.go` - API client tests with mocks

### Mock Server

```go
// internal/testutil/mock_server.go

package testutil

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
)

func NewMockAPIServer() *httptest.Server {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/v1/agent/apikey", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{
            "api_key": "mp_test_xxxxxxxxxxxxxxxxxxxxx",
        })
    })
    
    mux.HandleFunc("/v1/vendor/register", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]interface{}{
            "id":   "vendor_123",
            "slug": "test-service",
        })
    })
    
    return httptest.NewServer(mux)
}
```

### Acceptance Criteria

- [ ] Non-interactive setup tests for agent and vendor
- [ ] Login requirement validation
- [ ] Wallet generation verification
- [ ] API client tests with mock server
- [ ] Open command route tests
- [ ] Status JSON output validation
- [ ] All tests passing with `go test ./...`

---

## Implementation Order

1. **2.1** TUI Prompts → Foundation for interactive wizard
2. **2.3** Wallet Generation → Needed by setup
3. **2.4** API Client → Needed by setup
4. **2.2** Setup Command → Main feature
5. **2.5** Open Command → Quick win
6. **2.6** Enhanced Status → Improvement
7. **2.7** Integration Tests → Verification

---

## Testing Checklist

After implementing all prompts:

```bash
# Build
go build -o machpay ./cmd/machpay

# Test login → setup flow
./machpay login
./machpay setup

# Test commands
./machpay status
./machpay status --json
./machpay open marketplace

# Run all tests
go test ./... -v
```

---

*Phase 2 Estimated Time: 1 week*
*Next: Phase 3 - Gateway Integration*

