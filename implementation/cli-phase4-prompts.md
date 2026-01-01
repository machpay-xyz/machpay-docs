# MachPay CLI - Phase 4 Implementation Prompts

> **Role Setup:** You are the CTO and Lead Architect of MachPay, building a revolutionary AI payment platform. You have zero tolerance for technical debt, dead code, or hacky solutions. Every line of code should be production-ready, well-tested, and follow best practices.

---

## Phase 4 Overview

**Goal:** Desktop Bundling - Package the React console as a native desktop app with CLI included

| Prompt | Component | Description |
|--------|-----------|-------------|
| 4.1 | Tauri Setup | Initialize Tauri in machpay-console |
| 4.2 | Tauri Configuration | Configure window, permissions, bundling |
| 4.3 | Build Script | Download and bundle CLI/Gateway binaries |
| 4.4 | Rust Backend Commands | Tauri commands for CLI/Gateway integration |
| 4.5 | First Launch Modal | React component for initial setup |
| 4.6 | App Icons | Generate icons for all platforms |
| 4.7 | CI/CD Pipeline | GitHub Actions for building releases |
| 4.8 | Code Signing | macOS notarization, Windows signing |

---

## Prompt 4.1: Tauri Setup

### Context

We need to initialize Tauri in the existing `machpay-console` React application. Tauri is preferred over Electron for its smaller binary size (~10MB vs ~150MB), lower memory footprint, and Rust-based security model.

### Task

Initialize Tauri in `machpay-console` and configure the basic project structure.

### Prerequisites

```bash
# System requirements
rustup update
cargo install tauri-cli

# In machpay-console
npm install -D @tauri-apps/cli @tauri-apps/api
```

### Steps

1. **Initialize Tauri**

```bash
cd machpay-console
npm run tauri init
```

Answer prompts:
- App name: `MachPay`
- Window title: `MachPay Console`
- Web assets location: `../dist`
- Dev server URL: `http://localhost:5173`
- Frontend dev command: `npm run dev`
- Frontend build command: `npm run build`

2. **Update package.json**

Add Tauri scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "tauri": "tauri",
    "tauri:dev": "tauri dev",
    "tauri:build": "tauri build"
  }
}
```

3. **Verify Structure**

```
machpay-console/
├── src/                      # React app (existing)
├── src-tauri/                # NEW - Tauri Rust backend
│   ├── src/
│   │   └── main.rs           
│   ├── tauri.conf.json       
│   ├── icons/                
│   ├── Cargo.toml
│   └── build.rs
├── package.json              
└── ...
```

### Files to Create/Modify

1. `src-tauri/` directory (created by init)
2. `package.json` - Add Tauri scripts
3. `vite.config.js` - Ensure compatibility

### Acceptance Criteria

- [ ] Tauri initialized successfully
- [ ] `npm run tauri:dev` launches app window
- [ ] React app loads inside Tauri window
- [ ] Hot reload works in dev mode
- [ ] No console errors

---

## Prompt 4.2: Tauri Configuration

### Context

Configure Tauri for MachPay-specific requirements including window settings, security permissions, and bundle targets.

### Task

Update `src-tauri/tauri.conf.json` with proper configuration.

### Configuration

```json
{
  "$schema": "../node_modules/@tauri-apps/cli/schema.json",
  "productName": "MachPay",
  "version": "1.0.0",
  "identifier": "xyz.machpay.console",
  "build": {
    "distDir": "../dist",
    "devPath": "http://localhost:5173",
    "beforeBuildCommand": "npm run build",
    "beforeDevCommand": "npm run dev"
  },
  "bundle": {
    "active": true,
    "targets": ["dmg", "msi", "app", "appimage", "deb"],
    "identifier": "xyz.machpay.console",
    "publisher": "MachPay Inc.",
    "category": "Finance",
    "shortDescription": "AI Payment Network Console",
    "longDescription": "MachPay Console - Manage your AI agent payments, vendor services, and blockchain transactions.",
    "copyright": "© 2026 MachPay Inc.",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "resources": [],
    "externalBin": [
      "bin/machpay",
      "bin/machpay-gateway"
    ],
    "macOS": {
      "entitlements": null,
      "exceptionDomain": "",
      "frameworks": [],
      "providerShortName": null,
      "signingIdentity": null,
      "minimumSystemVersion": "10.15"
    },
    "windows": {
      "certificateThumbprint": null,
      "digestAlgorithm": "sha256",
      "timestampUrl": "",
      "wix": {
        "language": ["en-US"]
      },
      "webviewInstallMode": {
        "type": "downloadBootstrapper"
      }
    },
    "linux": {
      "appimage": {
        "bundleMediaFramework": true
      },
      "deb": {
        "depends": []
      }
    }
  },
  "tauri": {
    "allowlist": {
      "all": false,
      "shell": {
        "all": false,
        "open": true,
        "execute": false,
        "sidecar": true,
        "scope": [
          {
            "name": "bin/machpay",
            "sidecar": true,
            "args": true
          },
          {
            "name": "bin/machpay-gateway",
            "sidecar": true,
            "args": true
          }
        ]
      },
      "fs": {
        "all": false,
        "readFile": true,
        "writeFile": true,
        "readDir": true,
        "createDir": true,
        "removeFile": false,
        "removeDir": false,
        "scope": ["$APP/*", "$HOME/.machpay/*", "$CONFIG/*"]
      },
      "path": {
        "all": true
      },
      "os": {
        "all": true
      },
      "process": {
        "all": false,
        "relaunch": true,
        "exit": true
      },
      "dialog": {
        "all": true
      },
      "notification": {
        "all": true
      },
      "globalShortcut": {
        "all": false
      },
      "clipboard": {
        "all": true
      },
      "window": {
        "all": false,
        "close": true,
        "hide": true,
        "show": true,
        "maximize": true,
        "minimize": true,
        "unmaximize": true,
        "unminimize": true,
        "startDragging": true,
        "setTitle": true,
        "setFocus": true
      }
    },
    "windows": [
      {
        "title": "MachPay Console",
        "width": 1400,
        "height": 900,
        "minWidth": 1000,
        "minHeight": 600,
        "center": true,
        "resizable": true,
        "fullscreen": false,
        "decorations": true,
        "transparent": false,
        "alwaysOnTop": false,
        "skipTaskbar": false,
        "fileDropEnabled": false
      }
    ],
    "security": {
      "csp": "default-src 'self'; connect-src 'self' https://*.machpay.xyz https://*.machpay.network wss://*.machpay.xyz; style-src 'self' 'unsafe-inline'; script-src 'self'; img-src 'self' data: https:"
    },
    "updater": {
      "active": true,
      "endpoints": [
        "https://releases.machpay.xyz/console/{{target}}/{{arch}}/{{current_version}}"
      ],
      "dialog": true,
      "pubkey": ""
    }
  }
}
```

### Security Notes

- **CSP**: Restricts connections to MachPay domains only
- **Shell scope**: Only allows bundled binaries, not arbitrary commands
- **File scope**: Limited to app and config directories
- **No execute permission**: Can only run sidecars, not arbitrary commands

### Files to Modify

1. `src-tauri/tauri.conf.json` - Full configuration

### Acceptance Criteria

- [ ] Window opens with correct size
- [ ] CSP allows API connections
- [ ] Sidecar permissions configured
- [ ] File system access scoped correctly
- [ ] Auto-updater configured

---

## Prompt 4.3: Build Script

### Context

The Tauri build needs to download and bundle the CLI and Gateway binaries for the target platform. This happens during the build process via `build.rs`.

### Task

Create a build script that downloads the correct binaries for the current platform.

### Implementation

```rust
// src-tauri/build.rs

use std::env;
use std::fs;
use std::io::{self, Write};
use std::path::PathBuf;
use std::process::Command;

const CLI_REPO: &str = "machpay/machpay-cli";
const GATEWAY_REPO: &str = "machpay/machpay-gateway";

fn main() {
    // Get target info
    let target = env::var("TARGET").unwrap_or_else(|_| {
        let os = env::consts::OS;
        let arch = env::consts::ARCH;
        format!("{}-{}", arch, os)
    });

    let (os, arch) = parse_target(&target);
    
    println!("cargo:rerun-if-changed=build.rs");
    println!("Building for {}/{}", os, arch);

    // Create bin directory
    let bin_dir = PathBuf::from("bin");
    fs::create_dir_all(&bin_dir).expect("Failed to create bin directory");

    // Download binaries (skip in dev if already present)
    if should_download(&bin_dir, "machpay", os) {
        download_binary(CLI_REPO, "machpay", os, arch, &bin_dir)
            .expect("Failed to download CLI");
    }

    if should_download(&bin_dir, "machpay-gateway", os) {
        download_binary(GATEWAY_REPO, "machpay-gateway", os, arch, &bin_dir)
            .expect("Failed to download Gateway");
    }

    // Run standard Tauri build
    tauri_build::build();
}

fn parse_target(target: &str) -> (&str, &str) {
    // Parse targets like "aarch64-apple-darwin" or "x86_64-unknown-linux-gnu"
    let parts: Vec<&str> = target.split('-').collect();
    
    let arch = match parts.get(0) {
        Some(&"aarch64") | Some(&"arm64") => "arm64",
        Some(&"x86_64") | Some(&"amd64") => "amd64",
        _ => "amd64",
    };
    
    let os = if target.contains("darwin") {
        "darwin"
    } else if target.contains("windows") {
        "windows"
    } else {
        "linux"
    };
    
    (os, arch)
}

fn should_download(bin_dir: &PathBuf, name: &str, os: &str) -> bool {
    let binary_name = if os == "windows" {
        format!("{}.exe", name)
    } else {
        name.to_string()
    };
    
    let path = bin_dir.join(&binary_name);
    !path.exists()
}

fn download_binary(
    repo: &str,
    name: &str,
    os: &str,
    arch: &str,
    bin_dir: &PathBuf,
) -> Result<(), Box<dyn std::error::Error>> {
    // Get latest release version
    let version = get_latest_version(repo)?;
    println!("Downloading {} {}...", name, version);

    // Construct download URL
    let ext = if os == "windows" { "zip" } else { "tar.gz" };
    let binary_name = if os == "windows" {
        format!("{}.exe", name)
    } else {
        name.to_string()
    };
    
    let asset_name = format!("{}_{}_{}_{}.{}", name, version.trim_start_matches('v'), os, arch, ext);
    let url = format!(
        "https://github.com/{}/releases/download/{}/{}",
        repo, version, asset_name
    );

    // Download to temp file
    let temp_path = bin_dir.join(format!("{}.{}", name, ext));
    download_file(&url, &temp_path)?;

    // Extract
    if ext == "tar.gz" {
        extract_tar_gz(&temp_path, bin_dir, &binary_name)?;
    } else {
        extract_zip(&temp_path, bin_dir, &binary_name)?;
    }

    // Clean up
    fs::remove_file(&temp_path)?;

    // Make executable on Unix
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        let binary_path = bin_dir.join(&binary_name);
        let mut perms = fs::metadata(&binary_path)?.permissions();
        perms.set_mode(0o755);
        fs::set_permissions(&binary_path, perms)?;
    }

    println!("✓ Downloaded {} to {:?}", name, bin_dir.join(&binary_name));
    Ok(())
}

fn get_latest_version(repo: &str) -> Result<String, Box<dyn std::error::Error>> {
    let url = format!("https://api.github.com/repos/{}/releases/latest", repo);
    
    let output = Command::new("curl")
        .args(&["-sL", "-H", "User-Agent: tauri-build", &url])
        .output()?;
    
    let json: serde_json::Value = serde_json::from_slice(&output.stdout)?;
    let version = json["tag_name"]
        .as_str()
        .ok_or("No tag_name in release")?
        .to_string();
    
    Ok(version)
}

fn download_file(url: &str, dest: &PathBuf) -> Result<(), Box<dyn std::error::Error>> {
    let output = Command::new("curl")
        .args(&["-sL", "-o", dest.to_str().unwrap(), url])
        .output()?;
    
    if !output.status.success() {
        return Err(format!("Download failed: {}", String::from_utf8_lossy(&output.stderr)).into());
    }
    
    Ok(())
}

fn extract_tar_gz(
    archive: &PathBuf,
    dest_dir: &PathBuf,
    binary_name: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    let output = Command::new("tar")
        .args(&[
            "-xzf",
            archive.to_str().unwrap(),
            "-C",
            dest_dir.to_str().unwrap(),
            binary_name,
        ])
        .output()?;
    
    if !output.status.success() {
        // Try without specifying file (some archives have different structure)
        Command::new("tar")
            .args(&[
                "-xzf",
                archive.to_str().unwrap(),
                "-C",
                dest_dir.to_str().unwrap(),
            ])
            .output()?;
    }
    
    Ok(())
}

fn extract_zip(
    archive: &PathBuf,
    dest_dir: &PathBuf,
    binary_name: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    Command::new("unzip")
        .args(&[
            "-o",
            archive.to_str().unwrap(),
            binary_name,
            "-d",
            dest_dir.to_str().unwrap(),
        ])
        .output()?;
    
    Ok(())
}
```

### Update Cargo.toml

```toml
# src-tauri/Cargo.toml

[package]
name = "machpay-console"
version = "1.0.0"
description = "MachPay Console Desktop App"
authors = ["MachPay <hello@machpay.xyz>"]
license = "MIT"
repository = "https://github.com/machpay/machpay-console"
edition = "2021"

[build-dependencies]
tauri-build = { version = "1.5", features = [] }
serde_json = "1.0"

[dependencies]
tauri = { version = "1.5", features = ["shell-sidecar", "fs-read-file", "fs-write-file", "path-all", "os-all", "dialog-all", "notification-all", "clipboard-all", "window-all", "process-exit", "process-relaunch", "updater"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
```

### Files to Create/Modify

1. `src-tauri/build.rs` - Build script
2. `src-tauri/Cargo.toml` - Dependencies
3. `src-tauri/bin/` - Binary output directory (gitignored)

### Acceptance Criteria

- [ ] Build downloads correct binaries for target platform
- [ ] Supports darwin/linux/windows
- [ ] Supports amd64/arm64 architectures
- [ ] Binaries extracted and made executable
- [ ] Build caches binaries (doesn't re-download)

---

## Prompt 4.4: Rust Backend Commands

### Context

Tauri commands allow the React frontend to invoke Rust functions. We need commands to manage the CLI, Gateway, and first-launch setup.

### Task

Implement Tauri commands in `src-tauri/src/main.rs`.

### Implementation

```rust
// src-tauri/src/main.rs

#![cfg_attr(
    all(not(debug_assertions), target_os = "windows"),
    windows_subsystem = "windows"
)]

use std::process::Command as StdCommand;
use std::path::PathBuf;
use std::fs;
use std::sync::Mutex;
use tauri::{Manager, State, command};
use serde::{Deserialize, Serialize};

// ============================================================
// State
// ============================================================

struct GatewayState(Mutex<Option<std::process::Child>>);

// ============================================================
// Types
// ============================================================

#[derive(Serialize, Deserialize)]
pub struct CLIStatus {
    installed: bool,
    version: Option<String>,
    path: Option<String>,
}

#[derive(Serialize, Deserialize)]
pub struct GatewayStatus {
    running: bool,
    pid: Option<u32>,
    port: Option<u16>,
}

#[derive(Serialize, Deserialize)]
pub struct GatewayConfig {
    port: u16,
    upstream: String,
    debug: bool,
}

// ============================================================
// CLI Commands
// ============================================================

/// Check if CLI is installed in PATH
#[command]
fn check_cli_installed() -> CLIStatus {
    // Check if machpay is in PATH
    let which = if cfg!(target_os = "windows") {
        StdCommand::new("where").arg("machpay").output()
    } else {
        StdCommand::new("which").arg("machpay").output()
    };

    match which {
        Ok(output) if output.status.success() => {
            let path = String::from_utf8_lossy(&output.stdout)
                .trim()
                .to_string();
            
            // Get version
            let version = StdCommand::new("machpay")
                .arg("version")
                .output()
                .ok()
                .map(|o| String::from_utf8_lossy(&o.stdout).trim().to_string());
            
            CLIStatus {
                installed: true,
                version,
                path: Some(path),
            }
        }
        _ => CLIStatus {
            installed: false,
            version: None,
            path: None,
        },
    }
}

/// Install CLI to system PATH
#[command]
async fn install_cli(app: tauri::AppHandle) -> Result<String, String> {
    // Get bundled CLI path
    let resource_path = app
        .path_resolver()
        .resolve_resource("bin/machpay")
        .ok_or("CLI binary not found in bundle")?;

    // Determine install location
    let install_path = if cfg!(target_os = "windows") {
        // Windows: %LOCALAPPDATA%\MachPay\machpay.exe
        let local_app_data = std::env::var("LOCALAPPDATA")
            .map_err(|_| "LOCALAPPDATA not set")?;
        PathBuf::from(local_app_data).join("MachPay").join("machpay.exe")
    } else {
        // Unix: /usr/local/bin/machpay
        PathBuf::from("/usr/local/bin/machpay")
    };

    // Create parent directory if needed
    if let Some(parent) = install_path.parent() {
        fs::create_dir_all(parent).map_err(|e| e.to_string())?;
    }

    // Copy or symlink
    #[cfg(unix)]
    {
        // Try symlink first, fall back to copy
        if std::os::unix::fs::symlink(&resource_path, &install_path).is_err() {
            fs::copy(&resource_path, &install_path).map_err(|e| e.to_string())?;
        }
        
        // Make executable
        use std::os::unix::fs::PermissionsExt;
        let mut perms = fs::metadata(&install_path).map_err(|e| e.to_string())?.permissions();
        perms.set_mode(0o755);
        fs::set_permissions(&install_path, perms).map_err(|e| e.to_string())?;
    }

    #[cfg(windows)]
    {
        fs::copy(&resource_path, &install_path).map_err(|e| e.to_string())?;
        
        // Add to PATH via registry (optional, might need admin)
        // For now, we just install to a fixed location
    }

    Ok(install_path.to_string_lossy().to_string())
}

/// Run CLI command and return output
#[command]
async fn run_cli_command(args: Vec<String>) -> Result<String, String> {
    let output = StdCommand::new("machpay")
        .args(&args)
        .output()
        .map_err(|e| e.to_string())?;

    if output.status.success() {
        Ok(String::from_utf8_lossy(&output.stdout).to_string())
    } else {
        Err(String::from_utf8_lossy(&output.stderr).to_string())
    }
}

// ============================================================
// Gateway Commands
// ============================================================

/// Start gateway as sidecar process
#[command]
async fn start_gateway(
    config: GatewayConfig,
    app: tauri::AppHandle,
    state: State<'_, GatewayState>,
) -> Result<GatewayStatus, String> {
    let mut gateway = state.0.lock().map_err(|_| "Lock error")?;

    // Check if already running
    if let Some(ref mut child) = *gateway {
        match child.try_wait() {
            Ok(None) => {
                return Ok(GatewayStatus {
                    running: true,
                    pid: Some(child.id()),
                    port: Some(config.port),
                });
            }
            _ => {
                // Process exited, clear it
                *gateway = None;
            }
        }
    }

    // Start gateway sidecar
    let sidecar = app
        .shell()
        .sidecar("machpay-gateway")
        .map_err(|e| e.to_string())?;

    let mut args = vec![
        "--port".to_string(),
        config.port.to_string(),
        "--upstream".to_string(),
        config.upstream,
    ];
    
    if config.debug {
        args.push("--debug".to_string());
    }

    let (mut rx, child) = sidecar
        .args(&args)
        .spawn()
        .map_err(|e| e.to_string())?;

    let pid = child.pid();
    *gateway = Some(child.into());

    // Spawn task to handle output
    tauri::async_runtime::spawn(async move {
        while let Some(event) = rx.recv().await {
            // Could emit events to frontend here
            if let tauri::api::process::CommandEvent::Stdout(line) = event {
                println!("[gateway] {}", line);
            }
        }
    });

    Ok(GatewayStatus {
        running: true,
        pid: Some(pid),
        port: Some(config.port),
    })
}

/// Stop gateway
#[command]
async fn stop_gateway(state: State<'_, GatewayState>) -> Result<(), String> {
    let mut gateway = state.0.lock().map_err(|_| "Lock error")?;

    if let Some(mut child) = gateway.take() {
        child.kill().map_err(|e| e.to_string())?;
    }

    Ok(())
}

/// Check gateway status
#[command]
fn get_gateway_status(state: State<'_, GatewayState>) -> GatewayStatus {
    let gateway = state.0.lock().ok();

    match gateway {
        Some(ref g) => match g.as_ref() {
            Some(child) => {
                // Check if still running
                // Note: We can't easily check without consuming the child
                GatewayStatus {
                    running: true,
                    pid: Some(child.id()),
                    port: Some(8402), // Would need to track this
                }
            }
            None => GatewayStatus {
                running: false,
                pid: None,
                port: None,
            },
        },
        None => GatewayStatus {
            running: false,
            pid: None,
            port: None,
        },
    }
}

// ============================================================
// Config Commands
// ============================================================

/// Check if first launch
#[command]
fn is_first_launch() -> bool {
    let config_path = dirs::config_dir()
        .map(|p| p.join("machpay").join(".first_launch_complete"));

    match config_path {
        Some(path) => !path.exists(),
        None => true,
    }
}

/// Mark first launch complete
#[command]
fn complete_first_launch() -> Result<(), String> {
    let config_dir = dirs::config_dir()
        .ok_or("Could not find config directory")?
        .join("machpay");

    fs::create_dir_all(&config_dir).map_err(|e| e.to_string())?;
    fs::write(config_dir.join(".first_launch_complete"), "1")
        .map_err(|e| e.to_string())?;

    Ok(())
}

// ============================================================
// Main
// ============================================================

fn main() {
    tauri::Builder::default()
        .manage(GatewayState(Mutex::new(None)))
        .invoke_handler(tauri::generate_handler![
            // CLI
            check_cli_installed,
            install_cli,
            run_cli_command,
            // Gateway
            start_gateway,
            stop_gateway,
            get_gateway_status,
            // Config
            is_first_launch,
            complete_first_launch,
        ])
        .setup(|app| {
            // Any setup logic here
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Add dirs dependency

```toml
# Add to src-tauri/Cargo.toml [dependencies]
dirs = "5.0"
```

### Files to Modify

1. `src-tauri/src/main.rs` - Tauri commands
2. `src-tauri/Cargo.toml` - Add deps

### Acceptance Criteria

- [ ] `check_cli_installed` detects CLI in PATH
- [ ] `install_cli` creates symlink/copy
- [ ] `start_gateway` spawns sidecar process
- [ ] `stop_gateway` kills process
- [ ] `is_first_launch` checks config file
- [ ] All commands accessible from React

---

## Prompt 4.5: First Launch Modal

### Context

When users first open the desktop app, show a setup modal that offers to install the CLI to their system PATH.

### Task

Create a React component for the first launch experience.

### Implementation

```jsx
// src/components/FirstLaunchModal.jsx

import React, { useState, useEffect } from 'react';
import { invoke } from '@tauri-apps/api/tauri';
import { motion, AnimatePresence } from 'framer-motion';
import { 
  Terminal, 
  Download, 
  CheckCircle, 
  XCircle, 
  Loader2,
  ArrowRight 
} from 'lucide-react';
import { Button } from './ui/Button';
import { Panel } from './ui/Panel';

export function FirstLaunchModal({ onComplete }) {
  const [step, setStep] = useState('welcome'); // welcome, installing, done, error
  const [cliStatus, setCliStatus] = useState(null);
  const [installPath, setInstallPath] = useState(null);
  const [error, setError] = useState(null);
  const [installCli, setInstallCli] = useState(true);

  useEffect(() => {
    // Check if CLI is already installed
    checkCli();
  }, []);

  const checkCli = async () => {
    try {
      const status = await invoke('check_cli_installed');
      setCliStatus(status);
      if (status.installed) {
        setInstallCli(false); // Don't need to install
      }
    } catch (err) {
      console.error('Failed to check CLI:', err);
    }
  };

  const handleContinue = async () => {
    if (!installCli || cliStatus?.installed) {
      // Skip installation
      await completeSetup();
      return;
    }

    // Install CLI
    setStep('installing');
    try {
      const path = await invoke('install_cli');
      setInstallPath(path);
      setStep('done');
    } catch (err) {
      setError(err);
      setStep('error');
    }
  };

  const completeSetup = async () => {
    try {
      await invoke('complete_first_launch');
    } catch (err) {
      console.error('Failed to complete first launch:', err);
    }
    onComplete();
  };

  const handleSkip = async () => {
    await completeSetup();
  };

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/80 backdrop-blur-sm">
      <motion.div
        initial={{ opacity: 0, scale: 0.95 }}
        animate={{ opacity: 1, scale: 1 }}
        exit={{ opacity: 0, scale: 0.95 }}
        className="w-full max-w-lg"
      >
        <Panel className="p-8">
          <AnimatePresence mode="wait">
            {step === 'welcome' && (
              <WelcomeStep
                key="welcome"
                cliStatus={cliStatus}
                installCli={installCli}
                setInstallCli={setInstallCli}
                onContinue={handleContinue}
                onSkip={handleSkip}
              />
            )}

            {step === 'installing' && (
              <InstallingStep key="installing" />
            )}

            {step === 'done' && (
              <DoneStep
                key="done"
                installPath={installPath}
                onContinue={completeSetup}
              />
            )}

            {step === 'error' && (
              <ErrorStep
                key="error"
                error={error}
                onRetry={() => setStep('welcome')}
                onSkip={handleSkip}
              />
            )}
          </AnimatePresence>
        </Panel>
      </motion.div>
    </div>
  );
}

function WelcomeStep({ cliStatus, installCli, setInstallCli, onContinue, onSkip }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -20 }}
      className="text-center"
    >
      <div className="mb-6">
        <div className="w-16 h-16 mx-auto bg-emerald-500/20 rounded-2xl flex items-center justify-center mb-4">
          <Terminal className="w-8 h-8 text-emerald-500" />
        </div>
        <h2 className="text-2xl font-bold mb-2">Welcome to MachPay! 🚀</h2>
        <p className="text-muted-foreground">
          Let's get you set up in just a moment.
        </p>
      </div>

      <div className="bg-card border border-border rounded-lg p-4 mb-6 text-left">
        <h3 className="font-semibold mb-2 flex items-center gap-2">
          <Download className="w-4 h-4 text-emerald-500" />
          MachPay CLI
        </h3>
        <p className="text-sm text-muted-foreground mb-3">
          Install CLI to your terminal for:
        </p>
        <ul className="text-sm text-muted-foreground space-y-1 mb-4">
          <li>• Quick commands (<code className="text-emerald-500">machpay status</code>)</li>
          <li>• Running vendor gateway</li>
          <li>• SDK integration</li>
        </ul>

        {cliStatus?.installed ? (
          <div className="flex items-center gap-2 text-sm text-emerald-500">
            <CheckCircle className="w-4 h-4" />
            Already installed at {cliStatus.path}
          </div>
        ) : (
          <label className="flex items-center gap-2 cursor-pointer">
            <input
              type="checkbox"
              checked={installCli}
              onChange={(e) => setInstallCli(e.target.checked)}
              className="rounded border-border"
            />
            <span className="text-sm">Install CLI (recommended)</span>
          </label>
        )}
      </div>

      <div className="flex gap-3">
        <Button variant="ghost" onClick={onSkip} className="flex-1">
          Skip
        </Button>
        <Button onClick={onContinue} className="flex-1">
          Continue <ArrowRight className="w-4 h-4 ml-2" />
        </Button>
      </div>
    </motion.div>
  );
}

function InstallingStep() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -20 }}
      className="text-center py-8"
    >
      <Loader2 className="w-12 h-12 mx-auto text-emerald-500 animate-spin mb-4" />
      <h2 className="text-xl font-bold mb-2">Installing CLI...</h2>
      <p className="text-muted-foreground">This will only take a moment.</p>
    </motion.div>
  );
}

function DoneStep({ installPath, onContinue }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -20 }}
      className="text-center"
    >
      <CheckCircle className="w-16 h-16 mx-auto text-emerald-500 mb-4" />
      <h2 className="text-2xl font-bold mb-2">You're all set!</h2>
      <p className="text-muted-foreground mb-6">
        CLI installed to <code className="text-emerald-500">{installPath}</code>
      </p>

      <div className="bg-card border border-border rounded-lg p-4 mb-6 text-left">
        <p className="text-sm text-muted-foreground mb-2">Try it out:</p>
        <code className="block bg-black/50 rounded p-2 text-sm text-emerald-500">
          machpay status
        </code>
      </div>

      <Button onClick={onContinue} className="w-full">
        Get Started <ArrowRight className="w-4 h-4 ml-2" />
      </Button>
    </motion.div>
  );
}

function ErrorStep({ error, onRetry, onSkip }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -20 }}
      className="text-center"
    >
      <XCircle className="w-16 h-16 mx-auto text-red-500 mb-4" />
      <h2 className="text-2xl font-bold mb-2">Installation Failed</h2>
      <p className="text-muted-foreground mb-2">
        {error || 'An unexpected error occurred.'}
      </p>
      <p className="text-sm text-muted-foreground mb-6">
        You can install the CLI manually later using Homebrew or curl.
      </p>

      <div className="flex gap-3">
        <Button variant="ghost" onClick={onSkip} className="flex-1">
          Skip for now
        </Button>
        <Button onClick={onRetry} className="flex-1">
          Try Again
        </Button>
      </div>
    </motion.div>
  );
}
```

### Integration in App.jsx

```jsx
// In App.jsx or main layout

import { useState, useEffect } from 'react';
import { invoke } from '@tauri-apps/api/tauri';
import { FirstLaunchModal } from './components/FirstLaunchModal';

function App() {
  const [showFirstLaunch, setShowFirstLaunch] = useState(false);

  useEffect(() => {
    // Check if running in Tauri and first launch
    if (window.__TAURI__) {
      invoke('is_first_launch').then((isFirst) => {
        if (isFirst) {
          setShowFirstLaunch(true);
        }
      });
    }
  }, []);

  return (
    <>
      {showFirstLaunch && (
        <FirstLaunchModal onComplete={() => setShowFirstLaunch(false)} />
      )}
      {/* Rest of app */}
    </>
  );
}
```

### Files to Create/Modify

1. `src/components/FirstLaunchModal.jsx` - Modal component
2. `src/App.jsx` - Integration

### Acceptance Criteria

- [ ] Modal shows on first desktop app launch
- [ ] CLI installation works
- [ ] Skip option works
- [ ] Error handling works
- [ ] Modal doesn't show on subsequent launches

---

## Prompt 4.6: App Icons

### Context

Desktop apps need icons in multiple sizes and formats for different platforms.

### Task

Create app icons for all platforms.

### Required Formats

```
src-tauri/icons/
├── 32x32.png           # Windows, Linux tray
├── 128x128.png         # macOS Dock, Linux
├── 128x128@2x.png      # macOS Retina
├── icon.icns           # macOS app bundle
├── icon.ico            # Windows
└── icon.png            # Source (1024x1024)
```

### Icon Generation Script

```bash
#!/bin/bash
# generate-icons.sh
# Requires ImageMagick

SOURCE="icon.png"  # 1024x1024 source image

mkdir -p src-tauri/icons

# PNG sizes
convert "$SOURCE" -resize 32x32 src-tauri/icons/32x32.png
convert "$SOURCE" -resize 128x128 src-tauri/icons/128x128.png
convert "$SOURCE" -resize 256x256 src-tauri/icons/128x128@2x.png

# ICO for Windows (multi-size)
convert "$SOURCE" -define icon:auto-resize=256,128,64,48,32,16 src-tauri/icons/icon.ico

# ICNS for macOS
mkdir -p icon.iconset
convert "$SOURCE" -resize 16x16 icon.iconset/icon_16x16.png
convert "$SOURCE" -resize 32x32 icon.iconset/icon_16x16@2x.png
convert "$SOURCE" -resize 32x32 icon.iconset/icon_32x32.png
convert "$SOURCE" -resize 64x64 icon.iconset/icon_32x32@2x.png
convert "$SOURCE" -resize 128x128 icon.iconset/icon_128x128.png
convert "$SOURCE" -resize 256x256 icon.iconset/icon_128x128@2x.png
convert "$SOURCE" -resize 256x256 icon.iconset/icon_256x256.png
convert "$SOURCE" -resize 512x512 icon.iconset/icon_256x256@2x.png
convert "$SOURCE" -resize 512x512 icon.iconset/icon_512x512.png
convert "$SOURCE" -resize 1024x1024 icon.iconset/icon_512x512@2x.png
iconutil -c icns icon.iconset -o src-tauri/icons/icon.icns
rm -rf icon.iconset

echo "Icons generated!"
```

### Icon Design Guidelines

- Primary color: Emerald (#10b981)
- Simple, recognizable shape
- Works at small sizes (16x16)
- Transparent background for PNG
- Consider dark/light mode

### Files to Create

1. `src-tauri/icons/icon.png` - Source icon
2. `scripts/generate-icons.sh` - Generation script
3. All generated icon files

### Acceptance Criteria

- [ ] All required sizes present
- [ ] Icons look good at all sizes
- [ ] ICO has multiple sizes embedded
- [ ] ICNS properly formatted

---

## Prompt 4.7: CI/CD Pipeline

### Context

Automate building and releasing the desktop app for all platforms.

### Task

Create GitHub Actions workflow for Tauri builds.

### Implementation

```yaml
# .github/workflows/release-desktop.yml

name: Release Desktop App

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build-tauri:
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: macos-latest
            target: aarch64-apple-darwin
            name: macOS-arm64
          - platform: macos-latest
            target: x86_64-apple-darwin
            name: macOS-x64
          - platform: ubuntu-22.04
            target: x86_64-unknown-linux-gnu
            name: Linux-x64
          - platform: windows-latest
            target: x86_64-pc-windows-msvc
            name: Windows-x64

    runs-on: ${{ matrix.platform }}
    
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install Rust
        uses: dtolnay/rust-action@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install dependencies (Ubuntu)
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            libgtk-3-dev \
            libwebkit2gtk-4.0-dev \
            libayatana-appindicator3-dev \
            librsvg2-dev

      - name: Install npm dependencies
        run: npm ci

      - name: Build Tauri app
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          TAURI_PRIVATE_KEY: ${{ secrets.TAURI_PRIVATE_KEY }}
          TAURI_KEY_PASSWORD: ${{ secrets.TAURI_KEY_PASSWORD }}
          # macOS signing
          APPLE_CERTIFICATE: ${{ secrets.APPLE_CERTIFICATE }}
          APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
          APPLE_SIGNING_IDENTITY: ${{ secrets.APPLE_SIGNING_IDENTITY }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_PASSWORD: ${{ secrets.APPLE_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
        with:
          tagName: v__VERSION__
          releaseName: 'MachPay Console v__VERSION__'
          releaseBody: |
            ## MachPay Console v__VERSION__
            
            Download the version for your platform below.
            
            ### What's New
            See the [changelog](https://github.com/machpay/machpay-console/blob/main/CHANGELOG.md).
          releaseDraft: true
          prerelease: false
          args: --target ${{ matrix.target }}

  # Combine universal macOS binary
  build-macos-universal:
    needs: build-tauri
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download arm64 build
        uses: actions/download-artifact@v4
        with:
          name: MachPay-macos-arm64
          path: ./arm64

      - name: Download x64 build
        uses: actions/download-artifact@v4
        with:
          name: MachPay-macos-x64
          path: ./x64

      - name: Create universal binary
        run: |
          # Create universal binary using lipo
          mkdir -p universal
          lipo -create arm64/MachPay.app/Contents/MacOS/MachPay \
               x64/MachPay.app/Contents/MacOS/MachPay \
               -output universal/MachPay
          
          # Use arm64 as base and replace binary
          cp -R arm64/MachPay.app universal/
          cp universal/MachPay universal/MachPay.app/Contents/MacOS/

      - name: Create DMG
        run: |
          npm install -g create-dmg
          create-dmg universal/MachPay.app universal/
          mv "universal/MachPay "*.dmg "MachPay-universal.dmg"

      - name: Upload universal DMG
        uses: actions/upload-artifact@v4
        with:
          name: MachPay-macos-universal
          path: MachPay-universal.dmg
```

### Files to Create

1. `.github/workflows/release-desktop.yml` - Build workflow

### Required Secrets

- `TAURI_PRIVATE_KEY` - For auto-updater signing
- `TAURI_KEY_PASSWORD` - Key password
- `APPLE_CERTIFICATE` - macOS signing cert (base64)
- `APPLE_CERTIFICATE_PASSWORD` - Cert password
- `APPLE_SIGNING_IDENTITY` - "Developer ID Application: ..."
- `APPLE_ID` - Apple developer email
- `APPLE_PASSWORD` - App-specific password
- `APPLE_TEAM_ID` - Team ID

### Acceptance Criteria

- [ ] Builds for all platforms
- [ ] Creates release artifacts
- [ ] macOS universal binary created
- [ ] Artifacts uploaded to GitHub Release

---

## Prompt 4.8: Code Signing

### Context

Desktop apps need to be signed to avoid security warnings and enable auto-updates.

### Task

Configure code signing for macOS and Windows.

### macOS Notarization

1. **Get Developer ID Certificate**
   - Enroll in Apple Developer Program
   - Create "Developer ID Application" certificate
   - Export as .p12 file

2. **Configure in tauri.conf.json**

```json
{
  "bundle": {
    "macOS": {
      "signingIdentity": "Developer ID Application: MachPay Inc. (TEAM_ID)",
      "providerShortName": "TEAM_ID",
      "entitlements": "entitlements.plist"
    }
  }
}
```

3. **Create entitlements.plist**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
</dict>
</plist>
```

### Windows Signing

1. **Get Code Signing Certificate**
   - Purchase from DigiCert, Sectigo, etc.
   - Or use Azure Trusted Signing

2. **Configure in tauri.conf.json**

```json
{
  "bundle": {
    "windows": {
      "certificateThumbprint": "YOUR_THUMBPRINT",
      "digestAlgorithm": "sha256",
      "timestampUrl": "http://timestamp.digicert.com"
    }
  }
}
```

### Files to Create/Modify

1. `src-tauri/entitlements.plist` - macOS entitlements
2. `src-tauri/tauri.conf.json` - Signing config
3. GitHub Secrets for CI/CD

### Acceptance Criteria

- [ ] macOS app notarized
- [ ] No Gatekeeper warnings
- [ ] Windows app signed
- [ ] No SmartScreen warnings

---

## Implementation Order

```
4.1 Tauri Setup ─────→ 4.2 Configuration
                              │
                              ▼
4.6 App Icons ─────→ 4.3 Build Script
                              │
                              ▼
4.4 Rust Commands ─────→ 4.5 First Launch Modal
                              │
                              ▼
4.7 CI/CD ─────→ 4.8 Code Signing
```

**Recommended sequence:**
1. **4.1** Tauri Setup → Initialize project
2. **4.2** Configuration → Window, permissions
3. **4.6** App Icons → Design assets
4. **4.3** Build Script → Bundle binaries
5. **4.4** Rust Commands → Backend functions
6. **4.5** First Launch → React component
7. **4.7** CI/CD → Automation
8. **4.8** Code Signing → Production readiness

---

## Testing Checklist

```bash
# Development
npm run tauri:dev

# Production build
npm run tauri:build

# Test on each platform
# - macOS: Open .dmg, drag to Applications
# - Windows: Run .msi installer
# - Linux: Run AppImage or install .deb

# Verify
# - App opens without security warnings
# - First launch modal appears
# - CLI installation works
# - Gateway can be started
```

---

## Dependencies Added

**Rust (src-tauri/Cargo.toml):**
- `tauri` with features
- `serde`, `serde_json`
- `tokio`
- `dirs`

**Node (package.json):**
- `@tauri-apps/cli`
- `@tauri-apps/api`

---

*Phase 4 Estimated Time: 1-2 weeks*
*Next: Phase 5 - Distribution (Homebrew, Install Script, Website)*

