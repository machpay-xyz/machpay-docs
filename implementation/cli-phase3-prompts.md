# MachPay CLI - Phase 3 Implementation Prompts

> **Role Setup:** You are the CTO and Lead Architect of MachPay, building a revolutionary AI payment platform. You have zero tolerance for technical debt, dead code, or hacky solutions. Every line of code should be production-ready, well-tested, and follow Go best practices.

---

## Phase 3 Overview

**Goal:** Gateway Integration - Download, manage, and run the payment gateway

| Prompt | Component | Description |
|--------|-----------|-------------|
| 3.1 | Gateway Downloader | Download gateway from GitHub releases |
| 3.2 | Progress Bar | Terminal progress bar for downloads |
| 3.3 | Process Manager | Start, stop, monitor gateway process |
| 3.4 | Serve Command | `machpay serve` with live status |
| 3.5 | Gateway Logs | `machpay logs` command |
| 3.6 | Update Command | `machpay update` for CLI and gateway |
| 3.7 | Gateway Repo Changes | Prepare machpay-gateway for CLI integration |
| 3.8 | Integration Tests | E2E tests for gateway lifecycle |

---

## Prompt 3.1: Gateway Downloader

### Context

The CLI needs to download the `machpay-gateway` binary from GitHub releases. This should work across platforms (macOS, Linux, Windows) and architectures (amd64, arm64).

### Task

Create `internal/gateway/downloader.go` for downloading and managing the gateway binary.

### Requirements

1. **Check if installed** - Look for binary in `~/.machpay/bin/`
2. **Get latest version** - Query GitHub releases API
3. **Download with progress** - Show download progress in terminal
4. **Extract tarball** - Handle `.tar.gz` archives
5. **Verify checksum** - SHA256 verification from `checksums.txt`
6. **Make executable** - `chmod +x` on Unix systems

### Implementation

```go
// internal/gateway/downloader.go

package gateway

import (
    "archive/tar"
    "compress/gzip"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
    "runtime"
    "strings"
)

const (
    GatewayRepo   = "machpay/machpay-gateway"
    GatewayBinary = "machpay-gateway"
    GitHubAPI     = "https://api.github.com"
)

// Downloader manages gateway binary downloads
type Downloader struct {
    installDir string
    httpClient *http.Client
}

// NewDownloader creates a new downloader
func NewDownloader() *Downloader {
    home, _ := os.UserHomeDir()
    return &Downloader{
        installDir: filepath.Join(home, ".machpay", "bin"),
        httpClient: &http.Client{Timeout: 30 * time.Second},
    }
}

// BinaryPath returns the full path to the gateway binary
func (d *Downloader) BinaryPath() string {
    binary := GatewayBinary
    if runtime.GOOS == "windows" {
        binary += ".exe"
    }
    return filepath.Join(d.installDir, binary)
}

// IsInstalled checks if the gateway is installed
func (d *Downloader) IsInstalled() bool {
    _, err := os.Stat(d.BinaryPath())
    return err == nil
}

// InstalledVersion returns the version of the installed gateway
func (d *Downloader) InstalledVersion() (string, error) {
    if !d.IsInstalled() {
        return "", fmt.Errorf("gateway not installed")
    }
    
    // Run: machpay-gateway --version
    cmd := exec.Command(d.BinaryPath(), "--version")
    output, err := cmd.Output()
    if err != nil {
        return "", fmt.Errorf("get version: %w", err)
    }
    
    // Parse "machpay-gateway v1.2.0" format
    parts := strings.Fields(string(output))
    if len(parts) >= 2 {
        return strings.TrimPrefix(parts[1], "v"), nil
    }
    return "", fmt.Errorf("unable to parse version")
}

// GitHubRelease represents a GitHub release
type GitHubRelease struct {
    TagName string `json:"tag_name"`
    Assets  []struct {
        Name               string `json:"name"`
        BrowserDownloadURL string `json:"browser_download_url"`
    } `json:"assets"`
}

// GetLatestRelease fetches the latest release info from GitHub
func (d *Downloader) GetLatestRelease() (*GitHubRelease, error) {
    url := fmt.Sprintf("%s/repos/%s/releases/latest", GitHubAPI, GatewayRepo)
    
    req, err := http.NewRequest("GET", url, nil)
    if err != nil {
        return nil, err
    }
    req.Header.Set("Accept", "application/vnd.github.v3+json")
    req.Header.Set("User-Agent", "machpay-cli")
    
    resp, err := d.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("fetch release: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("GitHub API error: %s", resp.Status)
    }
    
    var release GitHubRelease
    if err := json.NewDecoder(resp.Body).Decode(&release); err != nil {
        return nil, fmt.Errorf("parse release: %w", err)
    }
    
    return &release, nil
}

// Download downloads and installs a specific version
func (d *Downloader) Download(version string, progress io.Writer) error {
    // 1. Construct asset name
    assetName := fmt.Sprintf("machpay-gateway_%s_%s.tar.gz", runtime.GOOS, runtime.GOARCH)
    checksumName := "checksums.txt"
    
    // 2. Get release info
    release, err := d.getRelease(version)
    if err != nil {
        return err
    }
    
    // 3. Find asset URLs
    var assetURL, checksumURL string
    for _, asset := range release.Assets {
        if asset.Name == assetName {
            assetURL = asset.BrowserDownloadURL
        }
        if asset.Name == checksumName {
            checksumURL = asset.BrowserDownloadURL
        }
    }
    
    if assetURL == "" {
        return fmt.Errorf("no asset found for %s/%s", runtime.GOOS, runtime.GOARCH)
    }
    
    // 4. Download checksum file
    expectedChecksum := ""
    if checksumURL != "" {
        expectedChecksum, err = d.fetchChecksum(checksumURL, assetName)
        if err != nil {
            // Non-fatal: continue without checksum verification
            fmt.Fprintf(progress, "Warning: Could not fetch checksum\n")
        }
    }
    
    // 5. Download tarball
    tmpFile, err := d.downloadFile(assetURL, progress)
    if err != nil {
        return fmt.Errorf("download: %w", err)
    }
    defer os.Remove(tmpFile)
    
    // 6. Verify checksum
    if expectedChecksum != "" {
        actualChecksum, err := d.fileChecksum(tmpFile)
        if err != nil {
            return fmt.Errorf("compute checksum: %w", err)
        }
        if actualChecksum != expectedChecksum {
            return fmt.Errorf("checksum mismatch: expected %s, got %s", expectedChecksum, actualChecksum)
        }
    }
    
    // 7. Extract and install
    if err := d.extractAndInstall(tmpFile); err != nil {
        return fmt.Errorf("install: %w", err)
    }
    
    return nil
}

// downloadFile downloads a file with progress reporting
func (d *Downloader) downloadFile(url string, progress io.Writer) (string, error) {
    resp, err := d.httpClient.Get(url)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return "", fmt.Errorf("download failed: %s", resp.Status)
    }
    
    // Create temp file
    tmpFile, err := os.CreateTemp("", "machpay-gateway-*.tar.gz")
    if err != nil {
        return "", err
    }
    defer tmpFile.Close()
    
    // Create progress wrapper if provided
    var reader io.Reader = resp.Body
    if progress != nil && resp.ContentLength > 0 {
        reader = &ProgressReader{
            Reader: resp.Body,
            Total:  resp.ContentLength,
            Writer: progress,
        }
    }
    
    _, err = io.Copy(tmpFile, reader)
    return tmpFile.Name(), err
}

// extractAndInstall extracts the tarball and installs the binary
func (d *Downloader) extractAndInstall(tarPath string) error {
    // Ensure install directory exists
    if err := os.MkdirAll(d.installDir, 0755); err != nil {
        return err
    }
    
    // Open tarball
    file, err := os.Open(tarPath)
    if err != nil {
        return err
    }
    defer file.Close()
    
    gzr, err := gzip.NewReader(file)
    if err != nil {
        return err
    }
    defer gzr.Close()
    
    tr := tar.NewReader(gzr)
    
    for {
        header, err := tr.Next()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        
        // Only extract the binary
        if header.Typeflag != tar.TypeReg {
            continue
        }
        if !strings.Contains(header.Name, GatewayBinary) {
            continue
        }
        
        // Write binary
        outPath := d.BinaryPath()
        outFile, err := os.OpenFile(outPath, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0755)
        if err != nil {
            return err
        }
        
        if _, err := io.Copy(outFile, tr); err != nil {
            outFile.Close()
            return err
        }
        outFile.Close()
        
        // Ensure executable on Unix
        if runtime.GOOS != "windows" {
            os.Chmod(outPath, 0755)
        }
        
        return nil
    }
    
    return fmt.Errorf("binary not found in archive")
}

// fileChecksum computes the SHA256 checksum of a file
func (d *Downloader) fileChecksum(path string) (string, error) {
    file, err := os.Open(path)
    if err != nil {
        return "", err
    }
    defer file.Close()
    
    hash := sha256.New()
    if _, err := io.Copy(hash, file); err != nil {
        return "", err
    }
    
    return hex.EncodeToString(hash.Sum(nil)), nil
}

// NeedsUpdate checks if an update is available
func (d *Downloader) NeedsUpdate() (bool, string, error) {
    installed, err := d.InstalledVersion()
    if err != nil {
        return true, "", nil // Not installed = needs "update"
    }
    
    release, err := d.GetLatestRelease()
    if err != nil {
        return false, "", err
    }
    
    latest := strings.TrimPrefix(release.TagName, "v")
    return installed != latest, latest, nil
}
```

### Files to Create

1. `internal/gateway/downloader.go` - Download logic
2. `internal/gateway/downloader_test.go` - Tests with mock HTTP

### Acceptance Criteria

- [ ] Detect OS/architecture correctly
- [ ] Download from GitHub releases
- [ ] Show progress during download
- [ ] Extract tar.gz archives
- [ ] Verify SHA256 checksum
- [ ] Make binary executable
- [ ] Handle network errors gracefully
- [ ] Unit tests with mock HTTP server

---

## Prompt 3.2: Progress Bar

### Context

Downloads need a visual progress indicator. Create a simple terminal progress bar without external dependencies.

### Task

Create `internal/gateway/progress.go` with a progress bar implementation.

### Implementation

```go
// internal/gateway/progress.go

package gateway

import (
    "fmt"
    "io"
    "strings"
    "sync"
)

// ProgressReader wraps an io.Reader to report progress
type ProgressReader struct {
    Reader   io.Reader
    Total    int64
    Current  int64
    Writer   io.Writer
    mu       sync.Mutex
    lastPct  int
}

func (pr *ProgressReader) Read(p []byte) (int, error) {
    n, err := pr.Reader.Read(p)
    
    pr.mu.Lock()
    pr.Current += int64(n)
    pct := int(float64(pr.Current) / float64(pr.Total) * 100)
    
    // Only update on percentage change
    if pct != pr.lastPct {
        pr.lastPct = pct
        pr.render()
    }
    pr.mu.Unlock()
    
    return n, err
}

func (pr *ProgressReader) render() {
    width := 40
    filled := int(float64(width) * float64(pr.Current) / float64(pr.Total))
    bar := strings.Repeat("█", filled) + strings.Repeat("░", width-filled)
    
    mb := float64(pr.Current) / 1024 / 1024
    totalMB := float64(pr.Total) / 1024 / 1024
    
    fmt.Fprintf(pr.Writer, "\r  %s %3d%% %.1f/%.1f MB", bar, pr.lastPct, mb, totalMB)
    
    if pr.Current >= pr.Total {
        fmt.Fprintln(pr.Writer)
    }
}

// ProgressWriter is a simple progress indicator for operations without known size
type ProgressWriter struct {
    Writer  io.Writer
    Message string
    frames  []string
    current int
}

func NewProgressWriter(w io.Writer, message string) *ProgressWriter {
    return &ProgressWriter{
        Writer:  w,
        Message: message,
        frames:  []string{"⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"},
    }
}

func (pw *ProgressWriter) Tick() {
    fmt.Fprintf(pw.Writer, "\r  %s %s", pw.frames[pw.current], pw.Message)
    pw.current = (pw.current + 1) % len(pw.frames)
}

func (pw *ProgressWriter) Done(success bool) {
    if success {
        fmt.Fprintf(pw.Writer, "\r  ✓ %s\n", pw.Message)
    } else {
        fmt.Fprintf(pw.Writer, "\r  ✗ %s\n", pw.Message)
    }
}
```

### Acceptance Criteria

- [ ] Progress bar with percentage and MB
- [ ] Spinner for indeterminate progress
- [ ] No external dependencies
- [ ] Thread-safe updates
- [ ] Clean line overwriting

---

## Prompt 3.3: Process Manager

### Context

The CLI needs to start, stop, and monitor the gateway process. This includes PID file management, log streaming, and health checks.

### Task

Create `internal/gateway/process.go` for process lifecycle management.

### Implementation

```go
// internal/gateway/process.go

package gateway

import (
    "bufio"
    "context"
    "fmt"
    "io"
    "os"
    "os/exec"
    "path/filepath"
    "strconv"
    "syscall"
    "time"
)

var (
    ErrNotRunning     = fmt.Errorf("gateway is not running")
    ErrAlreadyRunning = fmt.Errorf("gateway is already running")
)

// ProcessManager manages the gateway process lifecycle
type ProcessManager struct {
    binaryPath string
    configDir  string
    port       int
    upstream   string
    debug      bool
}

// NewProcessManager creates a new process manager
func NewProcessManager(binaryPath string, port int, upstream string) *ProcessManager {
    home, _ := os.UserHomeDir()
    return &ProcessManager{
        binaryPath: binaryPath,
        configDir:  filepath.Join(home, ".machpay"),
        port:       port,
        upstream:   upstream,
    }
}

// PIDFile returns the path to the PID file
func (pm *ProcessManager) PIDFile() string {
    return filepath.Join(pm.configDir, "gateway.pid")
}

// LogFile returns the path to the log file
func (pm *ProcessManager) LogFile() string {
    return filepath.Join(pm.configDir, "gateway.log")
}

// Start starts the gateway process
func (pm *ProcessManager) Start() error {
    if pm.IsRunning() {
        return ErrAlreadyRunning
    }

    // Ensure config dir exists
    os.MkdirAll(pm.configDir, 0755)

    // Build command
    args := []string{
        "--port", strconv.Itoa(pm.port),
    }
    if pm.upstream != "" {
        args = append(args, "--upstream", pm.upstream)
    }
    if pm.debug {
        args = append(args, "--debug")
    }

    cmd := exec.Command(pm.binaryPath, args...)

    // Set up logging
    logFile, err := os.OpenFile(pm.LogFile(), os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
    if err != nil {
        return fmt.Errorf("open log file: %w", err)
    }

    cmd.Stdout = logFile
    cmd.Stderr = logFile

    // Start process
    if err := cmd.Start(); err != nil {
        logFile.Close()
        return fmt.Errorf("start gateway: %w", err)
    }

    // Save PID
    if err := pm.savePID(cmd.Process.Pid); err != nil {
        cmd.Process.Kill()
        return fmt.Errorf("save PID: %w", err)
    }

    // Detach - don't wait for process
    go func() {
        cmd.Wait()
        os.Remove(pm.PIDFile())
    }()

    return nil
}

// StartForeground starts the gateway in the foreground (blocking)
func (pm *ProcessManager) StartForeground(ctx context.Context, stdout, stderr io.Writer) error {
    if pm.IsRunning() {
        return ErrAlreadyRunning
    }

    args := []string{
        "--port", strconv.Itoa(pm.port),
    }
    if pm.upstream != "" {
        args = append(args, "--upstream", pm.upstream)
    }
    if pm.debug {
        args = append(args, "--debug")
    }

    cmd := exec.CommandContext(ctx, pm.binaryPath, args...)
    cmd.Stdout = stdout
    cmd.Stderr = stderr

    // Save PID after start
    if err := cmd.Start(); err != nil {
        return fmt.Errorf("start gateway: %w", err)
    }

    pm.savePID(cmd.Process.Pid)
    defer os.Remove(pm.PIDFile())

    return cmd.Wait()
}

// Stop stops the gateway process gracefully
func (pm *ProcessManager) Stop() error {
    pid, err := pm.loadPID()
    if err != nil {
        return ErrNotRunning
    }

    process, err := os.FindProcess(pid)
    if err != nil {
        os.Remove(pm.PIDFile())
        return ErrNotRunning
    }

    // Send SIGTERM for graceful shutdown
    if err := process.Signal(syscall.SIGTERM); err != nil {
        os.Remove(pm.PIDFile())
        return ErrNotRunning
    }

    // Wait for process to exit (with timeout)
    done := make(chan error, 1)
    go func() {
        _, err := process.Wait()
        done <- err
    }()

    select {
    case <-done:
        os.Remove(pm.PIDFile())
        return nil
    case <-time.After(10 * time.Second):
        // Force kill if graceful shutdown takes too long
        process.Kill()
        os.Remove(pm.PIDFile())
        return nil
    }
}

// IsRunning checks if the gateway is running
func (pm *ProcessManager) IsRunning() bool {
    pid, err := pm.loadPID()
    if err != nil {
        return false
    }

    process, err := os.FindProcess(pid)
    if err != nil {
        return false
    }

    // Check if process is alive (signal 0 doesn't send anything, just checks)
    err = process.Signal(syscall.Signal(0))
    return err == nil
}

// GetPID returns the PID of the running gateway
func (pm *ProcessManager) GetPID() (int, error) {
    return pm.loadPID()
}

// TailLogs streams logs from the log file
func (pm *ProcessManager) TailLogs(ctx context.Context, follow bool, writer io.Writer) error {
    file, err := os.Open(pm.LogFile())
    if err != nil {
        return fmt.Errorf("open log file: %w", err)
    }
    defer file.Close()

    if !follow {
        // Just dump the entire file
        _, err := io.Copy(writer, file)
        return err
    }

    // Follow mode - seek to end and watch for new lines
    file.Seek(0, io.SeekEnd)
    reader := bufio.NewReader(file)

    for {
        select {
        case <-ctx.Done():
            return nil
        default:
            line, err := reader.ReadString('\n')
            if err == io.EOF {
                time.Sleep(100 * time.Millisecond)
                continue
            }
            if err != nil {
                return err
            }
            fmt.Fprint(writer, line)
        }
    }
}

// HealthCheck checks if the gateway is responding
func (pm *ProcessManager) HealthCheck() error {
    url := fmt.Sprintf("http://localhost:%d/healthz", pm.port)
    
    client := &http.Client{Timeout: 5 * time.Second}
    resp, err := client.Get(url)
    if err != nil {
        return fmt.Errorf("health check failed: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("health check returned %d", resp.StatusCode)
    }
    
    return nil
}

// Internal helpers

func (pm *ProcessManager) savePID(pid int) error {
    return os.WriteFile(pm.PIDFile(), []byte(strconv.Itoa(pid)), 0644)
}

func (pm *ProcessManager) loadPID() (int, error) {
    data, err := os.ReadFile(pm.PIDFile())
    if err != nil {
        return 0, err
    }
    return strconv.Atoi(strings.TrimSpace(string(data)))
}
```

### Files to Create

1. `internal/gateway/process.go` - Process management
2. `internal/gateway/process_test.go` - Tests

### Acceptance Criteria

- [ ] Start gateway in background (detached)
- [ ] Start gateway in foreground (blocking)
- [ ] Stop with graceful shutdown (SIGTERM)
- [ ] Force kill after timeout
- [ ] PID file management
- [ ] Log file streaming
- [ ] Health check via HTTP
- [ ] Handle orphaned processes

---

## Prompt 3.4: Serve Command

### Context

The `machpay serve` command is the main vendor command. It downloads the gateway if needed, starts it, and shows a live status panel.

### Task

Create `internal/cmd/serve.go` with the serve command.

### Implementation

```go
// internal/cmd/serve.go

package cmd

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/spf13/cobra"

    "github.com/machpay/machpay-cli/internal/auth"
    "github.com/machpay/machpay-cli/internal/config"
    "github.com/machpay/machpay-cli/internal/gateway"
    "github.com/machpay/machpay-cli/internal/tui"
)

var (
    servePort     int
    serveUpstream string
    serveDetach   bool
    serveDebug    bool
)

var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start the payment gateway",
    Long: `Start the MachPay payment gateway for your vendor service.

The gateway acts as a reverse proxy in front of your API, handling
x402 payment verification and request forwarding.

If the gateway is not installed, it will be downloaded automatically.

Examples:
  machpay serve                          # Start with config defaults
  machpay serve --port 8402              # Custom port
  machpay serve --upstream localhost:11434  # Override upstream
  machpay serve --detach                 # Run in background`,
    RunE: runServe,
}

func init() {
    serveCmd.Flags().IntVar(&servePort, "port", 8402, "Gateway listen port")
    serveCmd.Flags().StringVar(&serveUpstream, "upstream", "", "Upstream API URL (overrides config)")
    serveCmd.Flags().BoolVar(&serveDetach, "detach", false, "Run in background")
    serveCmd.Flags().BoolVar(&serveDebug, "debug", false, "Enable debug logging")
}

func runServe(cmd *cobra.Command, args []string) error {
    // 1. Check prerequisites
    if !auth.IsLoggedIn() {
        tui.PrintError("Not logged in")
        fmt.Println(tui.Muted("  Run 'machpay login' first"))
        return fmt.Errorf("not logged in")
    }

    cfg := config.Get()
    if cfg.Role != "vendor" {
        tui.PrintError("Not configured as vendor")
        fmt.Println(tui.Muted("  Run 'machpay setup' and select 'Vendor'"))
        return fmt.Errorf("not a vendor")
    }

    // Use config values if flags not set
    upstream := serveUpstream
    if upstream == "" {
        upstream = cfg.Vendor.UpstreamURL
    }
    if upstream == "" {
        tui.PrintError("No upstream URL configured")
        fmt.Println(tui.Muted("  Run 'machpay setup' to configure your upstream URL"))
        return fmt.Errorf("no upstream URL")
    }

    // 2. Ensure gateway is installed
    dl := gateway.NewDownloader()
    if !dl.IsInstalled() {
        fmt.Println()
        fmt.Println(tui.Warning("Gateway not found. Downloading..."))
        fmt.Println()

        if err := downloadGateway(dl); err != nil {
            return fmt.Errorf("download gateway: %w", err)
        }
    } else {
        // Check for updates in background
        go checkForUpdates(dl)
    }

    // 3. Create process manager
    pm := gateway.NewProcessManager(dl.BinaryPath(), servePort, upstream)
    pm.SetDebug(serveDebug)

    // 4. Handle detach mode
    if serveDetach {
        return runDetached(pm, upstream)
    }

    // 5. Run in foreground
    return runForeground(pm, upstream)
}

func downloadGateway(dl *gateway.Downloader) error {
    release, err := dl.GetLatestRelease()
    if err != nil {
        return fmt.Errorf("fetch latest version: %w", err)
    }

    version := release.TagName
    fmt.Printf("  Downloading %s for %s/%s...\n", 
        tui.Primary(version), runtime.GOOS, runtime.GOARCH)

    if err := dl.Download(version, os.Stdout); err != nil {
        return err
    }

    fmt.Println()
    tui.PrintSuccess(fmt.Sprintf("Gateway %s installed", version))
    return nil
}

func runDetached(pm *gateway.ProcessManager, upstream string) error {
    if pm.IsRunning() {
        pid, _ := pm.GetPID()
        fmt.Printf("Gateway already running (PID %d)\n", pid)
        return nil
    }

    if err := pm.Start(); err != nil {
        return fmt.Errorf("start gateway: %w", err)
    }

    // Wait a moment and check health
    time.Sleep(2 * time.Second)
    
    if err := pm.HealthCheck(); err != nil {
        tui.PrintWarning("Gateway started but health check failed")
        fmt.Println(tui.Muted("  Check logs with 'machpay logs'"))
    } else {
        pid, _ := pm.GetPID()
        tui.PrintSuccess(fmt.Sprintf("Gateway started (PID %d)", pid))
        fmt.Println()
        tui.PrintKeyValue("Port", fmt.Sprintf("%d", servePort))
        tui.PrintKeyValue("Upstream", upstream)
        tui.PrintKeyValue("Logs", pm.LogFile())
    }

    return nil
}

func runForeground(pm *gateway.ProcessManager, upstream string) error {
    if pm.IsRunning() {
        pid, _ := pm.GetPID()
        return fmt.Errorf("gateway already running (PID %d). Stop it first with 'machpay stop'", pid)
    }

    // Print status header
    printGatewayHeader(upstream)

    // Handle Ctrl+C
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        <-sigChan
        fmt.Println()
        fmt.Println(tui.Muted("Shutting down..."))
        cancel()
    }()

    // Start gateway
    err := pm.StartForeground(ctx, os.Stdout, os.Stderr)
    
    if ctx.Err() != nil {
        // Graceful shutdown
        tui.PrintSuccess("Gateway stopped")
        return nil
    }

    return err
}

func printGatewayHeader(upstream string) {
    fmt.Println()
    fmt.Println(tui.Bold("MachPay Gateway"))
    fmt.Println()
    tui.PrintKeyValue("Port", fmt.Sprintf("http://localhost:%d", servePort))
    tui.PrintKeyValue("Upstream", upstream)
    fmt.Println()
    fmt.Println(tui.Muted("Press Ctrl+C to stop"))
    tui.PrintSection()
}

func checkForUpdates(dl *gateway.Downloader) {
    needsUpdate, latest, err := dl.NeedsUpdate()
    if err != nil || !needsUpdate {
        return
    }

    fmt.Println()
    tui.PrintInfo(fmt.Sprintf("Update available: %s", latest))
    fmt.Println(tui.Muted("  Run 'machpay update' to upgrade"))
}
```

### Additional Commands

Also add these helper commands:

```go
// internal/cmd/stop.go
var stopCmd = &cobra.Command{
    Use:   "stop",
    Short: "Stop the gateway",
    RunE: func(cmd *cobra.Command, args []string) error {
        dl := gateway.NewDownloader()
        pm := gateway.NewProcessManager(dl.BinaryPath(), 0, "")
        
        if !pm.IsRunning() {
            fmt.Println(tui.Muted("Gateway is not running"))
            return nil
        }
        
        if err := pm.Stop(); err != nil {
            return err
        }
        
        tui.PrintSuccess("Gateway stopped")
        return nil
    },
}
```

### Files to Create/Modify

1. `internal/cmd/serve.go` - Serve command
2. `internal/cmd/stop.go` - Stop command
3. `internal/cmd/root.go` - Add serveCmd, stopCmd

### Acceptance Criteria

- [ ] Prerequisite checks (login, vendor role)
- [ ] Auto-download gateway if not installed
- [ ] Foreground mode with Ctrl+C handling
- [ ] Detach mode (background)
- [ ] Status header with port/upstream
- [ ] Health check after start
- [ ] Update notification in background
- [ ] Stop command

---

## Prompt 3.5: Logs Command

### Context

Users need to view gateway logs for debugging.

### Task

Create `internal/cmd/logs.go` for log viewing.

### Implementation

```go
// internal/cmd/logs.go

package cmd

import (
    "context"
    "os"
    "os/signal"
    "syscall"

    "github.com/spf13/cobra"

    "github.com/machpay/machpay-cli/internal/gateway"
    "github.com/machpay/machpay-cli/internal/tui"
)

var logsFollow bool

var logsCmd = &cobra.Command{
    Use:   "logs",
    Short: "View gateway logs",
    Long: `View the gateway log output.

Examples:
  machpay logs           # Show recent logs
  machpay logs -f        # Follow logs in real-time`,
    RunE: runLogs,
}

func init() {
    logsCmd.Flags().BoolVarP(&logsFollow, "follow", "f", false, "Follow log output")
}

func runLogs(cmd *cobra.Command, args []string) error {
    dl := gateway.NewDownloader()
    pm := gateway.NewProcessManager(dl.BinaryPath(), 0, "")

    ctx := context.Background()
    
    if logsFollow {
        ctx, cancel := context.WithCancel(ctx)
        defer cancel()
        
        sigChan := make(chan os.Signal, 1)
        signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
        go func() {
            <-sigChan
            cancel()
        }()
    }

    if err := pm.TailLogs(ctx, logsFollow, os.Stdout); err != nil {
        if os.IsNotExist(err) {
            tui.PrintWarning("No logs found. Has the gateway been started?")
            return nil
        }
        return err
    }

    return nil
}
```

### Acceptance Criteria

- [ ] Show recent logs
- [ ] `-f` flag for follow mode
- [ ] Handle missing log file
- [ ] Ctrl+C to stop following

---

## Prompt 3.6: Update Command

### Context

Users need to update the CLI and gateway to the latest versions.

### Task

Create `internal/cmd/update.go` for updates.

### Implementation

```go
// internal/cmd/update.go

package cmd

import (
    "fmt"
    "os"
    "runtime"

    "github.com/spf13/cobra"

    "github.com/machpay/machpay-cli/internal/gateway"
    "github.com/machpay/machpay-cli/internal/tui"
)

var updateCmd = &cobra.Command{
    Use:   "update",
    Short: "Update CLI and gateway",
    Long: `Check for and install updates to the CLI and gateway.

Examples:
  machpay update         # Update everything
  machpay update gateway # Update gateway only`,
    RunE: runUpdate,
}

func runUpdate(cmd *cobra.Command, args []string) error {
    target := "all"
    if len(args) > 0 {
        target = args[0]
    }

    switch target {
    case "gateway":
        return updateGateway()
    case "cli":
        return updateCLI()
    case "all":
        if err := updateGateway(); err != nil {
            tui.PrintWarning(fmt.Sprintf("Gateway update failed: %v", err))
        }
        // CLI self-update would go here
        return nil
    default:
        return fmt.Errorf("unknown target: %s (use 'gateway', 'cli', or 'all')", target)
    }
}

func updateGateway() error {
    dl := gateway.NewDownloader()

    fmt.Println(tui.Bold("Checking for gateway updates..."))

    needsUpdate, latest, err := dl.NeedsUpdate()
    if err != nil {
        return fmt.Errorf("check updates: %w", err)
    }

    if !needsUpdate {
        installed, _ := dl.InstalledVersion()
        fmt.Printf("  Gateway is up to date (%s)\n", tui.Primary(installed))
        return nil
    }

    fmt.Printf("  Updating to %s...\n", tui.Primary(latest))
    fmt.Println()

    if err := dl.Download("v"+latest, os.Stdout); err != nil {
        return fmt.Errorf("download: %w", err)
    }

    fmt.Println()
    tui.PrintSuccess(fmt.Sprintf("Gateway updated to %s", latest))

    return nil
}

func updateCLI() error {
    // For now, just print instructions
    fmt.Println(tui.Bold("CLI Update"))
    fmt.Println()
    fmt.Println("To update the CLI, run:")
    fmt.Println()

    switch runtime.GOOS {
    case "darwin":
        fmt.Println("  brew upgrade machpay")
    default:
        fmt.Println("  curl -fsSL https://machpay.xyz/install.sh | sh")
    }

    return nil
}
```

### Acceptance Criteria

- [ ] Check for gateway updates
- [ ] Download and install updates
- [ ] Show current version if up to date
- [ ] Instructions for CLI update

---

## Prompt 3.7: Gateway Repo Changes

### Context

The `machpay-gateway` repo needs changes to support CLI integration.

### Task

Document required changes to the gateway repo.

### Changes Required

1. **Version Endpoint**
```go
// In gateway's HTTP server
router.GET("/healthz/version", func(c *gin.Context) {
    c.JSON(200, gin.H{
        "version": Version,
        "commit":  Commit,
        "built":   BuildDate,
    })
})
```

2. **CLI Flag for Version**
```go
var versionFlag bool

func init() {
    rootCmd.Flags().BoolVar(&versionFlag, "version", false, "Print version")
}

if versionFlag {
    fmt.Printf("machpay-gateway %s\n", Version)
    os.Exit(0)
}
```

3. **GoReleaser Configuration**
```yaml
# .goreleaser.yaml
project_name: machpay-gateway

before:
  hooks:
    - go mod tidy

builds:
  - id: gateway
    main: ./cmd/gateway
    binary: machpay-gateway
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w
      - -X main.Version={{.Version}}
      - -X main.Commit={{.Commit}}
      - -X main.BuildDate={{.Date}}

archives:
  - id: default
    format: tar.gz
    name_template: "{{ .ProjectName }}_{{ .Os }}_{{ .Arch }}"
    format_overrides:
      - goos: windows
        format: zip

checksum:
  name_template: 'checksums.txt'
  algorithm: sha256

release:
  github:
    owner: machpay
    name: machpay-gateway
  draft: false
  prerelease: auto
```

4. **GitHub Actions Workflow**
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - uses: goreleaser/goreleaser-action@v5
        with:
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Files to Create in machpay-gateway

1. `.goreleaser.yaml` - Release config
2. `.github/workflows/release.yml` - CI workflow
3. Update `cmd/gateway/main.go` - Add version flag
4. Update HTTP server - Add `/healthz/version`

---

## Prompt 3.8: Integration Tests

### Context

Phase 3 introduces complex external interactions that need testing.

### Task

Create integration tests for gateway lifecycle.

### Test Structure

```go
// internal/gateway/integration_test.go

// +build integration

package gateway

import (
    "net/http"
    "net/http/httptest"
    "os"
    "testing"
)

func TestDownloader_Integration(t *testing.T) {
    // Skip if not in integration mode
    if os.Getenv("INTEGRATION_TEST") == "" {
        t.Skip("Skipping integration test")
    }

    dl := NewDownloader()

    // Test GetLatestRelease (requires network)
    t.Run("GetLatestRelease", func(t *testing.T) {
        release, err := dl.GetLatestRelease()
        if err != nil {
            t.Fatalf("GetLatestRelease failed: %v", err)
        }
        if release.TagName == "" {
            t.Error("Expected non-empty tag name")
        }
    })
}

// Mock tests for unit testing
func TestDownloader_MockedDownload(t *testing.T) {
    // Create mock GitHub API
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        switch r.URL.Path {
        case "/repos/machpay/machpay-gateway/releases/latest":
            w.Write([]byte(`{
                "tag_name": "v1.0.0",
                "assets": [
                    {"name": "machpay-gateway_linux_amd64.tar.gz", "browser_download_url": "http://example.com/file.tar.gz"},
                    {"name": "checksums.txt", "browser_download_url": "http://example.com/checksums.txt"}
                ]
            }`))
        default:
            w.WriteHeader(404)
        }
    }))
    defer server.Close()

    // Test with mocked API
    // ...
}

func TestProcessManager_Lifecycle(t *testing.T) {
    // This would test with a mock binary or the real gateway in test mode
}
```

### Acceptance Criteria

- [ ] Mock HTTP server for download tests
- [ ] Process lifecycle tests
- [ ] Integration tests (opt-in via env var)
- [ ] All tests passing

---

## Implementation Order

```
3.2 Progress Bar ────────┐
                         ├──→ 3.1 Downloader
3.7 Gateway Repo ────────┘
                               │
                               ▼
3.3 Process Manager ─────→ 3.4 Serve Command
                               │
                               ├──→ 3.5 Logs Command
                               │
                               └──→ 3.6 Update Command

3.8 Tests ───────────────→ (After all above)
```

**Recommended sequence:**
1. **3.2** Progress Bar → Simple utility
2. **3.7** Gateway Repo → Needed for releases
3. **3.1** Downloader → Core download logic
4. **3.3** Process Manager → Process lifecycle
5. **3.4** Serve Command → Main feature
6. **3.5** Logs Command → Quick win
7. **3.6** Update Command → Maintenance
8. **3.8** Integration Tests → Verification

---

## Testing Checklist

```bash
# Build
go build -o machpay ./cmd/machpay

# Test serve flow (requires gateway release)
./machpay serve              # Download + run in foreground
./machpay serve --detach     # Run in background
./machpay logs -f            # Follow logs
./machpay stop               # Stop gateway
./machpay update gateway     # Update gateway

# Run tests
go test ./internal/gateway/... -v
INTEGRATION_TEST=1 go test ./internal/gateway/... -v -tags=integration
```

---

## Dependencies

Phase 3 uses only standard library:
- `archive/tar` - Extract tarballs
- `compress/gzip` - Decompress
- `crypto/sha256` - Checksum verification
- `os/exec` - Process management
- `net/http` - GitHub API / health checks

No new external dependencies required.

---

*Phase 3 Estimated Time: 1 week*
*Next: Phase 4 - Desktop Bundling (Tauri)*

