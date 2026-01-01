# Phase 5: Distribution - Detailed Implementation Prompts

> **Role:** You are the CTO of MachPay, building the next revolutionary AI payment network.
> **Context:** Phase 1-4 complete (CLI, core commands, gateway integration, desktop bundling). Now we make it easy for everyone to install.
> **Priority:** P0 - Critical for adoption
> **Timeline:** 1 week

---

## Pre-Implementation Checklist

Before starting Phase 5, verify:
- [ ] CLI builds successfully: `cd machpay-cli && go build ./...`
- [ ] All CLI tests pass: `go test ./...`
- [ ] Tauri compiles: `cd machpay-console/src-tauri && cargo check`
- [ ] GoReleaser installed: `goreleaser --version`

---

## Prompt 5.1: GoReleaser Configuration

### Context
Configure GoReleaser in `machpay-cli` for automated multi-platform releases with Homebrew integration.

### Task
Create `.goreleaser.yaml` in the `machpay-cli` repository.

### Requirements

1. **Build Configuration**
   - Binary name: `machpay`
   - Targets: darwin/amd64, darwin/arm64, linux/amd64, linux/arm64, windows/amd64
   - Set `CGO_ENABLED=0` for static binaries
   - Inject version via `-ldflags`

2. **Archive Configuration**
   - Format: `tar.gz` for Unix, `zip` for Windows
   - Name template: `machpay_{{ .Os }}_{{ .Arch }}`
   - Include: binary, LICENSE, README.md

3. **Checksum**
   - Algorithm: SHA256
   - File: `checksums.txt`

4. **Homebrew Integration**
   - Repository: `machpay/homebrew-tap`
   - Formula folder: `Formula`
   - Include caveats with getting started info

5. **Changelog**
   - Auto-generate from commits
   - Group by type (feat, fix, docs)

6. **Docker**
   - Image: `ghcr.io/machpay/cli`
   - Tags: version, latest
   - Dockerfile path: `Dockerfile`

### File Structure
```yaml
# .goreleaser.yaml
project_name: machpay

before:
  hooks:
    - go mod tidy
    - go generate ./...

builds:
  - id: machpay
    main: ./cmd/machpay
    binary: machpay
    env:
      - CGO_ENABLED=0
    goos:
      - darwin
      - linux
      - windows
    goarch:
      - amd64
      - arm64
    ignore:
      - goos: windows
        goarch: arm64
    ldflags:
      - -s -w
      - -X github.com/machpay/machpay-cli/internal/cmd.Version={{.Version}}
      - -X github.com/machpay/machpay-cli/internal/cmd.Commit={{.Commit}}
      - -X github.com/machpay/machpay-cli/internal/cmd.Date={{.Date}}

archives:
  - id: default
    name_template: >-
      {{ .ProjectName }}_{{ .Os }}_{{ .Arch }}
    format_overrides:
      - goos: windows
        format: zip
    files:
      - LICENSE
      - README.md

checksum:
  name_template: 'checksums.txt'
  algorithm: sha256

changelog:
  sort: asc
  filters:
    exclude:
      - '^docs:'
      - '^test:'
      - '^chore:'
  groups:
    - title: Features
      regexp: '^feat'
    - title: Bug Fixes
      regexp: '^fix'
    - title: Performance
      regexp: '^perf'

brews:
  - name: machpay
    repository:
      owner: machpay
      name: homebrew-tap
      token: "{{ .Env.HOMEBREW_TAP_TOKEN }}"
    folder: Formula
    homepage: "https://machpay.xyz"
    description: "MachPay CLI - AI payment network orchestrator"
    license: "MIT"
    install: |
      bin.install "machpay"
    caveats: |
      MachPay CLI installed! 🚀
      
      Get started:
        machpay login     # Link your account
        machpay setup     # Configure your node
        machpay serve     # Start vendor gateway
      
      The gateway downloads automatically when needed.
      
      Documentation: https://docs.machpay.xyz/cli
    test: |
      system "#{bin}/machpay", "version"

dockers:
  - image_templates:
      - "ghcr.io/machpay/cli:{{ .Tag }}"
      - "ghcr.io/machpay/cli:v{{ .Major }}"
      - "ghcr.io/machpay/cli:v{{ .Major }}.{{ .Minor }}"
      - "ghcr.io/machpay/cli:latest"
    dockerfile: Dockerfile
    build_flag_templates:
      - "--label=org.opencontainers.image.created={{.Date}}"
      - "--label=org.opencontainers.image.revision={{.FullCommit}}"
      - "--label=org.opencontainers.image.version={{.Version}}"
      - "--platform=linux/amd64,linux/arm64"

release:
  github:
    owner: machpay
    name: machpay-cli
  draft: false
  prerelease: auto
  name_template: "MachPay CLI v{{.Version}}"
  header: |
    ## MachPay CLI v{{.Version}}
    
    ### Installation
    
    **Homebrew (macOS/Linux):**
    ```bash
    brew install machpay/tap/machpay
    ```
    
    **Curl (Linux/macOS):**
    ```bash
    curl -fsSL https://machpay.xyz/install.sh | sh
    ```
    
    **Docker:**
    ```bash
    docker pull ghcr.io/machpay/cli:{{.Tag}}
    ```
```

### Acceptance Criteria
- [ ] `goreleaser check` passes
- [ ] `goreleaser build --snapshot --clean` produces binaries for all platforms
- [ ] Version info injected correctly via ldflags

---

## Prompt 5.2: Homebrew Tap Repository

### Context
Create the Homebrew tap repository for distributing the CLI via `brew install`.

### Task
Create and configure the `machpay/homebrew-tap` repository.

### Requirements

1. **Repository Structure**
   ```
   homebrew-tap/
   ├── Formula/
   │   └── machpay.rb      # Auto-updated by GoReleaser
   ├── .github/
   │   └── workflows/
   │       └── audit.yml   # Formula validation
   ├── LICENSE
   └── README.md
   ```

2. **Initial Formula (placeholder)**
   ```ruby
   # Formula/machpay.rb
   # This file is auto-updated by GoReleaser
   class Machpay < Formula
     desc "MachPay CLI - AI payment network orchestrator"
     homepage "https://machpay.xyz"
     version "0.0.0"
     license "MIT"
     
     on_macos do
       on_arm do
         url "https://github.com/machpay/machpay-cli/releases/download/v0.0.0/machpay_darwin_arm64.tar.gz"
         sha256 "PLACEHOLDER"
       end
       on_intel do
         url "https://github.com/machpay/machpay-cli/releases/download/v0.0.0/machpay_darwin_amd64.tar.gz"
         sha256 "PLACEHOLDER"
       end
     end
     
     on_linux do
       on_arm do
         url "https://github.com/machpay/machpay-cli/releases/download/v0.0.0/machpay_linux_arm64.tar.gz"
         sha256 "PLACEHOLDER"
       end
       on_intel do
         url "https://github.com/machpay/machpay-cli/releases/download/v0.0.0/machpay_linux_amd64.tar.gz"
         sha256 "PLACEHOLDER"
       end
     end

     def install
       bin.install "machpay"
     end

     test do
       assert_match "machpay version", shell_output("#{bin}/machpay version")
     end
   end
   ```

3. **Audit Workflow**
   ```yaml
   # .github/workflows/audit.yml
   name: Audit
   on:
     push:
       paths:
         - 'Formula/*.rb'
     pull_request:
       paths:
         - 'Formula/*.rb'
   jobs:
     audit:
       runs-on: macos-latest
       steps:
         - uses: actions/checkout@v4
         - name: Audit Formula
           run: |
             brew audit --strict Formula/machpay.rb
   ```

4. **README.md**
   ```markdown
   # MachPay Homebrew Tap
   
   Official Homebrew tap for MachPay CLI.
   
   ## Installation
   
   ```bash
   brew tap machpay/tap
   brew install machpay
   ```
   
   ## Upgrade
   
   ```bash
   brew upgrade machpay
   ```
   
   ## Formulas
   
   | Formula | Description |
   |---------|-------------|
   | machpay | MachPay CLI - AI payment network orchestrator |
   
   ## Documentation
   
   - [CLI Documentation](https://docs.machpay.xyz/cli)
   - [MachPay Website](https://machpay.xyz)
   ```

### Acceptance Criteria
- [ ] Repository created at `machpay/homebrew-tap`
- [ ] Formula passes `brew audit --strict`
- [ ] Audit workflow configured

---

## Prompt 5.3: CLI Release Workflow

### Context
Create GitHub Actions workflow for automated CLI releases.

### Task
Create `.github/workflows/release.yml` in `machpay-cli`.

### Requirements

1. **Trigger on version tags**
   - Pattern: `v*` (e.g., v1.0.0)
   
2. **Jobs**
   - Test: Run all tests before release
   - Release: Run GoReleaser
   
3. **Secrets Required**
   - `GITHUB_TOKEN`: Auto-provided
   - `HOMEBREW_TAP_TOKEN`: PAT for tap repo
   - `DOCKER_USERNAME`, `DOCKER_PASSWORD`: GHCR access

### File
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write
  packages: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          
      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...
        
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.out

  release:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v5
        with:
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HOMEBREW_TAP_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}
          
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: binaries
          path: dist/*.tar.gz
```

### Acceptance Criteria
- [ ] Workflow triggers on version tags
- [ ] Tests run before release
- [ ] GoReleaser produces all artifacts
- [ ] Homebrew formula updated automatically
- [ ] Docker image pushed to GHCR

---

## Prompt 5.4: Install Script (Unix)

### Context
Create a curl-friendly install script for quick CLI installation on macOS and Linux.

### Task
Create `scripts/install.sh` in `machpay-cli` (and host at `machpay.xyz/install.sh`).

### Requirements

1. **Platform Detection**
   - Detect OS: darwin, linux
   - Detect arch: amd64, arm64
   - Handle unsupported platforms gracefully

2. **Installation Process**
   - Get latest version from GitHub API
   - Download appropriate tarball
   - Verify SHA256 checksum
   - Install to `/usr/local/bin` (or `$HOME/.local/bin` if no sudo)
   - Make executable

3. **User Experience**
   - ASCII art banner
   - Progress indicators
   - Success message with next steps
   - Error messages with troubleshooting

4. **Security**
   - HTTPS only
   - Checksum verification
   - No execution of downloaded code

### File
```bash
#!/bin/sh
# MachPay CLI Installer
# Usage: curl -fsSL https://machpay.xyz/install.sh | sh
#
# Environment variables:
#   MACHPAY_INSTALL_DIR - Installation directory (default: /usr/local/bin)
#   MACHPAY_VERSION     - Specific version to install (default: latest)

set -e

# Configuration
GITHUB_REPO="machpay/machpay-cli"
BINARY_NAME="machpay"
INSTALL_DIR="${MACHPAY_INSTALL_DIR:-}"

# Colors (only if terminal supports them)
if [ -t 1 ]; then
    RED='\033[0;31m'
    GREEN='\033[0;32m'
    YELLOW='\033[1;33m'
    BLUE='\033[0;34m'
    CYAN='\033[0;36m'
    BOLD='\033[1m'
    NC='\033[0m'
else
    RED=''
    GREEN=''
    YELLOW=''
    BLUE=''
    CYAN=''
    BOLD=''
    NC=''
fi

# Print functions
info() { printf "${CYAN}→${NC} %s\n" "$1"; }
success() { printf "${GREEN}✓${NC} %s\n" "$1"; }
warn() { printf "${YELLOW}!${NC} %s\n" "$1"; }
error() { printf "${RED}✗${NC} %s\n" "$1" >&2; exit 1; }

# Banner
print_banner() {
    printf "\n"
    printf "${BLUE}███╗   ███╗ █████╗  ██████╗██╗  ██╗██████╗  █████╗ ██╗${NC}\n"
    printf "${BLUE}████╗ ████║██╔══██╗██╔════╝██║  ██║██╔══██╗██╔══██╗╚██╗${NC}\n"
    printf "${BLUE}██╔████╔██║███████║██║     ███████║██████╔╝███████║ ██║${NC}\n"
    printf "${BLUE}██║╚██╔╝██║██╔══██║██║     ██╔══██║██╔═══╝ ██╔══██║ ██║${NC}\n"
    printf "${BLUE}██║ ╚═╝ ██║██║  ██║╚██████╗██║  ██║██║     ██║  ██║██╔╝${NC}\n"
    printf "${BLUE}╚═╝     ╚═╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝╚═╝${NC}\n"
    printf "\n"
    printf "${BOLD}CLI Installer${NC}\n"
    printf "\n"
}

# Detect platform
detect_platform() {
    OS="$(uname -s | tr '[:upper:]' '[:lower:]')"
    ARCH="$(uname -m)"
    
    case "$ARCH" in
        x86_64|amd64)   ARCH="amd64" ;;
        aarch64|arm64)  ARCH="arm64" ;;
        armv7l)         ARCH="arm" ;;
        *)              error "Unsupported architecture: $ARCH" ;;
    esac
    
    case "$OS" in
        darwin)         OS="darwin" ;;
        linux)          OS="linux" ;;
        mingw*|msys*|cygwin*)
            error "Windows detected. Please use:\n  iwr machpay.xyz/install.ps1 | iex"
            ;;
        *)              error "Unsupported OS: $OS" ;;
    esac
    
    info "Platform: ${OS}/${ARCH}"
}

# Determine install directory
determine_install_dir() {
    if [ -n "$INSTALL_DIR" ]; then
        # User specified
        return
    fi
    
    # Try /usr/local/bin first
    if [ -d "/usr/local/bin" ] && [ -w "/usr/local/bin" ]; then
        INSTALL_DIR="/usr/local/bin"
    elif [ -w "/usr/local/bin" ] || command -v sudo >/dev/null 2>&1; then
        INSTALL_DIR="/usr/local/bin"
        NEED_SUDO=1
    else
        # Fall back to user directory
        INSTALL_DIR="$HOME/.local/bin"
        mkdir -p "$INSTALL_DIR"
        # Add to PATH hint
        ADD_TO_PATH=1
    fi
    
    info "Install directory: $INSTALL_DIR"
}

# Get latest version
get_version() {
    if [ -n "$MACHPAY_VERSION" ]; then
        VERSION="$MACHPAY_VERSION"
    else
        info "Fetching latest version..."
        VERSION=$(curl -sL "https://api.github.com/repos/${GITHUB_REPO}/releases/latest" | \
            grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
    fi
    
    if [ -z "$VERSION" ]; then
        error "Could not determine version. Check your internet connection."
    fi
    
    info "Version: $VERSION"
}

# Download and install
download_and_install() {
    TARBALL="${BINARY_NAME}_${OS}_${ARCH}.tar.gz"
    URL="https://github.com/${GITHUB_REPO}/releases/download/${VERSION}/${TARBALL}"
    CHECKSUM_URL="https://github.com/${GITHUB_REPO}/releases/download/${VERSION}/checksums.txt"
    
    # Create temp directory
    TMP_DIR=$(mktemp -d)
    trap 'rm -rf "$TMP_DIR"' EXIT
    
    info "Downloading ${TARBALL}..."
    if ! curl -sL -o "$TMP_DIR/$TARBALL" "$URL"; then
        error "Download failed. URL: $URL"
    fi
    
    # Verify checksum
    info "Verifying checksum..."
    CHECKSUMS=$(curl -sL "$CHECKSUM_URL")
    EXPECTED=$(echo "$CHECKSUMS" | grep "$TARBALL" | cut -d ' ' -f 1)
    
    if command -v sha256sum >/dev/null 2>&1; then
        ACTUAL=$(sha256sum "$TMP_DIR/$TARBALL" | cut -d ' ' -f 1)
    elif command -v shasum >/dev/null 2>&1; then
        ACTUAL=$(shasum -a 256 "$TMP_DIR/$TARBALL" | cut -d ' ' -f 1)
    else
        warn "Cannot verify checksum (no sha256sum or shasum)"
        ACTUAL="$EXPECTED"  # Skip verification
    fi
    
    if [ "$EXPECTED" != "$ACTUAL" ]; then
        error "Checksum verification failed!\n  Expected: $EXPECTED\n  Actual:   $ACTUAL"
    fi
    success "Checksum verified"
    
    # Extract
    info "Extracting..."
    tar -xzf "$TMP_DIR/$TARBALL" -C "$TMP_DIR"
    
    # Install
    info "Installing to $INSTALL_DIR..."
    if [ "$NEED_SUDO" = "1" ]; then
        sudo mv "$TMP_DIR/$BINARY_NAME" "$INSTALL_DIR/"
        sudo chmod +x "$INSTALL_DIR/$BINARY_NAME"
    else
        mv "$TMP_DIR/$BINARY_NAME" "$INSTALL_DIR/"
        chmod +x "$INSTALL_DIR/$BINARY_NAME"
    fi
    
    success "Installed to $INSTALL_DIR/$BINARY_NAME"
}

# Print success message
print_success() {
    printf "\n"
    printf "${GREEN}╔════════════════════════════════════════════════════════════╗${NC}\n"
    printf "${GREEN}║${NC}          ${BOLD}MachPay CLI installed successfully!${NC}            ${GREEN}║${NC}\n"
    printf "${GREEN}╚════════════════════════════════════════════════════════════╝${NC}\n"
    printf "\n"
    
    if [ "$ADD_TO_PATH" = "1" ]; then
        printf "${YELLOW}Note:${NC} Add this to your shell profile:\n"
        printf "  ${CYAN}export PATH=\"\$HOME/.local/bin:\$PATH\"${NC}\n"
        printf "\n"
    fi
    
    printf "Get started:\n"
    printf "  ${CYAN}machpay login${NC}     # Link your account\n"
    printf "  ${CYAN}machpay setup${NC}     # Configure your node\n"
    printf "  ${CYAN}machpay serve${NC}     # Start vendor gateway\n"
    printf "\n"
    printf "Documentation: ${BLUE}https://docs.machpay.xyz/cli${NC}\n"
    printf "\n"
}

# Main
main() {
    print_banner
    detect_platform
    determine_install_dir
    get_version
    download_and_install
    print_success
}

main
```

### Acceptance Criteria
- [ ] Script works on macOS (Intel and ARM)
- [ ] Script works on Linux (Ubuntu, Debian, Alpine)
- [ ] Checksum verification works
- [ ] Falls back to user directory if no sudo
- [ ] Clear error messages

---

## Prompt 5.5: Install Script (Windows PowerShell)

### Context
Create a PowerShell install script for Windows users.

### Task
Create `scripts/install.ps1` in `machpay-cli` (and host at `machpay.xyz/install.ps1`).

### Requirements

1. **Platform Detection**
   - Detect architecture: amd64 (arm64 not supported yet)
   - Verify PowerShell version

2. **Installation Process**
   - Download zip archive
   - Verify checksum
   - Extract to `%LOCALAPPDATA%\MachPay`
   - Add to user PATH

### File
```powershell
# MachPay CLI Installer for Windows
# Usage: iwr machpay.xyz/install.ps1 | iex

$ErrorActionPreference = "Stop"

$GitHubRepo = "machpay/machpay-cli"
$BinaryName = "machpay.exe"
$InstallDir = "$env:LOCALAPPDATA\MachPay"

function Write-Banner {
    Write-Host ""
    Write-Host "███╗   ███╗ █████╗  ██████╗██╗  ██╗██████╗  █████╗ ██╗" -ForegroundColor Blue
    Write-Host "████╗ ████║██╔══██╗██╔════╝██║  ██║██╔══██╗██╔══██╗╚██╗" -ForegroundColor Blue
    Write-Host "██╔████╔██║███████║██║     ███████║██████╔╝███████║ ██║" -ForegroundColor Blue
    Write-Host "██║╚██╔╝██║██╔══██║██║     ██╔══██║██╔═══╝ ██╔══██║ ██║" -ForegroundColor Blue
    Write-Host "██║ ╚═╝ ██║██║  ██║╚██████╗██║  ██║██║     ██║  ██║██╔╝" -ForegroundColor Blue
    Write-Host "╚═╝     ╚═╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝╚═╝" -ForegroundColor Blue
    Write-Host ""
    Write-Host "CLI Installer for Windows" -ForegroundColor White
    Write-Host ""
}

function Get-LatestVersion {
    Write-Host "→ Fetching latest version..." -ForegroundColor Cyan
    $release = Invoke-RestMethod -Uri "https://api.github.com/repos/$GitHubRepo/releases/latest"
    return $release.tag_name
}

function Install-MachPay {
    Write-Banner
    
    $version = Get-LatestVersion
    Write-Host "→ Version: $version" -ForegroundColor Cyan
    
    $arch = if ([Environment]::Is64BitOperatingSystem) { "amd64" } else { "386" }
    $zipName = "machpay_windows_$arch.zip"
    $downloadUrl = "https://github.com/$GitHubRepo/releases/download/$version/$zipName"
    $checksumUrl = "https://github.com/$GitHubRepo/releases/download/$version/checksums.txt"
    
    # Create install directory
    if (-not (Test-Path $InstallDir)) {
        New-Item -ItemType Directory -Path $InstallDir -Force | Out-Null
    }
    
    $tempDir = Join-Path $env:TEMP "machpay-install-$(Get-Random)"
    New-Item -ItemType Directory -Path $tempDir -Force | Out-Null
    
    try {
        # Download
        Write-Host "→ Downloading $zipName..." -ForegroundColor Cyan
        $zipPath = Join-Path $tempDir $zipName
        Invoke-WebRequest -Uri $downloadUrl -OutFile $zipPath
        
        # Verify checksum
        Write-Host "→ Verifying checksum..." -ForegroundColor Cyan
        $checksums = Invoke-WebRequest -Uri $checksumUrl | Select-Object -ExpandProperty Content
        $expectedHash = ($checksums -split "`n" | Where-Object { $_ -match $zipName } | Select-Object -First 1) -replace "\s.*", ""
        $actualHash = (Get-FileHash -Path $zipPath -Algorithm SHA256).Hash.ToLower()
        
        if ($expectedHash -ne $actualHash) {
            throw "Checksum verification failed!"
        }
        Write-Host "✓ Checksum verified" -ForegroundColor Green
        
        # Extract
        Write-Host "→ Extracting..." -ForegroundColor Cyan
        Expand-Archive -Path $zipPath -DestinationPath $tempDir -Force
        
        # Install
        Write-Host "→ Installing to $InstallDir..." -ForegroundColor Cyan
        Copy-Item -Path (Join-Path $tempDir $BinaryName) -Destination $InstallDir -Force
        
        # Add to PATH
        $userPath = [Environment]::GetEnvironmentVariable("Path", "User")
        if ($userPath -notlike "*$InstallDir*") {
            Write-Host "→ Adding to PATH..." -ForegroundColor Cyan
            [Environment]::SetEnvironmentVariable("Path", "$userPath;$InstallDir", "User")
        }
        
        Write-Host ""
        Write-Host "╔════════════════════════════════════════════════════════════╗" -ForegroundColor Green
        Write-Host "║          MachPay CLI installed successfully!               ║" -ForegroundColor Green
        Write-Host "╚════════════════════════════════════════════════════════════╝" -ForegroundColor Green
        Write-Host ""
        Write-Host "Restart your terminal, then run:" -ForegroundColor White
        Write-Host "  machpay login     # Link your account" -ForegroundColor Cyan
        Write-Host "  machpay setup     # Configure your node" -ForegroundColor Cyan
        Write-Host ""
        Write-Host "Documentation: https://docs.machpay.xyz/cli" -ForegroundColor Blue
        Write-Host ""
        
    } finally {
        Remove-Item -Path $tempDir -Recurse -Force -ErrorAction SilentlyContinue
    }
}

Install-MachPay
```

### Acceptance Criteria
- [ ] Script works on Windows 10/11
- [ ] Adds to user PATH
- [ ] Checksum verification
- [ ] Clear success message

---

## Prompt 5.6: Docker Image

### Context
Create a minimal Docker image for CI/CD and containerized usage.

### Task
Create `Dockerfile` in `machpay-cli`.

### Requirements

1. **Multi-stage build**
   - Build stage: Go compilation
   - Runtime stage: Minimal scratch/alpine

2. **Size optimization**
   - Target: <20MB
   - Use scratch base if possible

### File
```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder

RUN apk add --no-cache git ca-certificates

WORKDIR /build

COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG VERSION=dev
ARG COMMIT=unknown

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X github.com/machpay/machpay-cli/internal/cmd.Version=${VERSION} -X github.com/machpay/machpay-cli/internal/cmd.Commit=${COMMIT}" \
    -o machpay \
    ./cmd/machpay

# Runtime stage
FROM alpine:3.19

RUN apk add --no-cache ca-certificates tzdata

COPY --from=builder /build/machpay /usr/local/bin/machpay

# Create non-root user
RUN adduser -D -u 1000 machpay
USER machpay

WORKDIR /home/machpay

ENTRYPOINT ["machpay"]
CMD ["--help"]
```

### Acceptance Criteria
- [ ] Image builds successfully
- [ ] Size < 30MB
- [ ] Runs as non-root
- [ ] All commands work

---

## Prompt 5.7: Website Download Page

### Context
Create a beautiful download page for the MachPay website.

### Task
Create download page component in `machpay-website`.

### Requirements

1. **Platform Detection**
   - Auto-detect user's OS
   - Highlight recommended download

2. **Download Options**
   - Desktop App (DMG, EXE, AppImage)
   - CLI (Homebrew, curl, PowerShell)
   - Docker

3. **Design**
   - Match MachPay brand
   - Clear CTAs
   - Version info and changelog link

### React Component Structure
```jsx
// src/pages/Download.jsx

export default function Download() {
  const [platform, setPlatform] = useState(detectPlatform());
  const [version, setVersion] = useState(null);
  
  // Platform detection
  // Version fetching from GitHub
  // Download tracking
  
  return (
    <div className="min-h-screen bg-void">
      {/* Hero */}
      <section className="py-20 text-center">
        <h1>Download MachPay</h1>
        <p>Choose your preferred installation method</p>
      </section>
      
      {/* Desktop App - Primary */}
      <section className="py-16">
        <h2>Desktop App (Recommended)</h2>
        <p>Full console with CLI and Gateway bundled</p>
        <PlatformCards />
      </section>
      
      {/* CLI Only */}
      <section className="py-16">
        <h2>CLI Only</h2>
        <p>Lightweight command-line tool</p>
        <CodeBlocks />
      </section>
      
      {/* Docker */}
      <section className="py-16">
        <h2>Docker</h2>
        <DockerInstructions />
      </section>
      
      {/* Version info */}
      <VersionInfo version={version} />
    </div>
  );
}
```

### Acceptance Criteria
- [ ] Auto-detects platform
- [ ] All download links work
- [ ] Version displayed
- [ ] Mobile responsive
- [ ] Analytics tracking

---

## Prompt 5.8: Integration Tests

### Context
Ensure all distribution methods work correctly.

### Task
Create integration tests for the distribution.

### Test Cases

1. **Homebrew**
   ```bash
   # Test tap and install
   brew tap machpay/tap
   brew install machpay
   machpay version
   brew uninstall machpay
   ```

2. **Install Script**
   ```bash
   # Test curl install
   curl -fsSL https://machpay.xyz/install.sh | sh
   machpay version
   rm /usr/local/bin/machpay
   ```

3. **Docker**
   ```bash
   # Test Docker image
   docker run --rm ghcr.io/machpay/cli version
   docker run --rm ghcr.io/machpay/cli status
   ```

4. **Desktop App**
   - Test DMG mount and install
   - Test CLI symlink creation
   - Test gateway sidecar

### File
```yaml
# .github/workflows/test-distribution.yml
name: Test Distribution

on:
  workflow_dispatch:
  release:
    types: [published]

jobs:
  test-homebrew:
    runs-on: macos-latest
    steps:
      - name: Tap and install
        run: |
          brew tap machpay/tap
          brew install machpay
          machpay version
          
  test-install-script-macos:
    runs-on: macos-latest
    steps:
      - name: Run install script
        run: |
          curl -fsSL https://machpay.xyz/install.sh | sh
          machpay version
          
  test-install-script-linux:
    runs-on: ubuntu-latest
    steps:
      - name: Run install script
        run: |
          curl -fsSL https://machpay.xyz/install.sh | sh
          machpay version
          
  test-docker:
    runs-on: ubuntu-latest
    steps:
      - name: Test Docker image
        run: |
          docker run --rm ghcr.io/machpay/cli:latest version
```

### Acceptance Criteria
- [ ] All distribution methods pass tests
- [ ] Tests run on release
- [ ] Failures alert team

---

## Summary

| Prompt | Deliverable | Repo |
|--------|-------------|------|
| 5.1 | GoReleaser config | machpay-cli |
| 5.2 | Homebrew tap | homebrew-tap |
| 5.3 | Release workflow | machpay-cli |
| 5.4 | Unix install script | machpay-cli |
| 5.5 | Windows install script | machpay-cli |
| 5.6 | Docker image | machpay-cli |
| 5.7 | Download page | machpay-website |
| 5.8 | Integration tests | machpay-cli |

---

## Execution Order

1. **5.1** GoReleaser config (foundation for everything)
2. **5.2** Homebrew tap repo (needed for releases)
3. **5.3** Release workflow (automates the process)
4. **5.4 + 5.5** Install scripts (parallel)
5. **5.6** Docker image
6. **5.7** Download page
7. **5.8** Integration tests

---

**Estimated Time:** 5-7 days
**Priority:** P0 - Critical for launch

