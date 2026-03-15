# Desktop Development Standards

> Authoritative desktop development standards for Tauri v2 applications across all projects.

## Purpose

Establish consistent patterns for building, testing, and distributing native desktop applications using Tauri v2 with Rust backends and web-based frontends (standalone HTML/CSS/JS or wrapping existing web applications).

## Core Principles

1. **Native-first experience** — Desktop apps should feel native to the operating system, not like websites in a window
2. **Rust for system work** — Use Rust for filesystem access, system integration, and heavy computation; keep the frontend for presentation
3. **Security by default** — Follow the principle of least privilege for system access, IPC, and file operations
4. **Two architectures** — Support both standalone apps (own frontend + Rust backend) and connected apps (wrapping an existing web frontend)
5. **Cross-platform aware** — Write code that compiles on macOS, Windows, and Linux, with platform-specific branches where needed

---

## Application Modes

Desktop applications follow one of two architectural patterns:

### Standalone Mode

The application ships its own frontend (HTML/CSS/JS) served directly by Tauri. No external web server required. Ideal for utilities, tools, and apps that don't have a companion web application.

| Aspect | Detail |
|--------|--------|
| Frontend | Static HTML/CSS/JS in `src/` directory |
| Config | `withGlobalTauri: true`, `frontendDist: "../src"` |
| Dev workflow | `tauri dev` — Tauri serves frontend + Rust hot-reload |
| Build output | Single binary (`.app`, `.msi`, `.AppImage`) |
| Dependencies | No Node.js runtime needed |

### Connected Mode

The application wraps an existing web frontend in a native window, adding desktop-specific capabilities (system tray, file access, notifications). The web app runs on localhost during development.

| Aspect | Detail |
|--------|--------|
| Frontend | External web app (Next.js, React, etc.) |
| Config | `devUrl: "http://localhost:3000"`, `frontendDist: "path/to/.next"` |
| Dev workflow | Start web app first, then `tauri dev` |
| Build output | Binary embedding the built frontend |
| Dependencies | Web app must be running for development |

---

## Project Structure

### Standard Desktop App Layout

```
apps/{app-name}/
├── package.json               # Node.js integration (Tauri CLI)
├── src/                       # Frontend (standalone mode only)
│   ├── index.html             # Main HTML document
│   ├── styles.css             # Stylesheet with CSS custom properties
│   └── main.js                # Navigation, theme, Tauri command calls
├── src-tauri/
│   ├── tauri.conf.json        # Tauri configuration
│   ├── Cargo.toml             # Rust crate definition
│   ├── build.rs               # Tauri build script
│   ├── capabilities/
│   │   └── default.json       # Permission ACLs
│   ├── icons/                 # App icons (all platforms)
│   └── src/
│       ├── main.rs            # Rust entry point
│       └── lib.rs             # Commands, app builder, modules
```

### Rust Module Organisation

For larger applications, split Rust backend into focused modules:

```
src-tauri/src/
├── main.rs          # Entry point (delegates to lib)
├── lib.rs           # App builder, command registration
├── scanner.rs       # Filesystem operations
├── cleaner.rs       # Cleanup/deletion operations
├── performance.rs   # System monitoring
└── tray.rs          # System tray integration
```

---

## Tauri Configuration

### Standalone Frontend Configuration

```json
{
  "build": {
    "frontendDist": "../src",
    "beforeBuildCommand": "",
    "beforeDevCommand": ""
  },
  "app": {
    "withGlobalTauri": true,
    "windows": [{
      "title": "App Name",
      "width": 1100,
      "height": 750,
      "minWidth": 800,
      "minHeight": 600,
      "resizable": true,
      "center": true
    }],
    "security": {
      "csp": "default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline'"
    }
  }
}
```

**Key settings:**
- `withGlobalTauri: true` — Required for static HTML frontends to access `window.__TAURI__`
- `frontendDist: "../src"` — Points to the static frontend directory
- CSP must allow `unsafe-inline` for static HTML apps (no build step to externalise scripts)

### Connected Frontend Configuration

```json
{
  "build": {
    "devUrl": "http://localhost:3000",
    "frontendDist": "../../apps/my-app/frontend/.next"
  },
  "app": {
    "windows": [{
      "title": "App Name",
      "width": 1200,
      "height": 800,
      "resizable": true,
      "center": true
    }],
    "security": {
      "csp": null
    }
  }
}
```

### Capabilities & Permissions

Use Tauri v2's capability system for least-privilege access:

```json
{
  "identifier": "default",
  "description": "Default capabilities for the main window",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "shell:allow-open"
  ]
}
```

Only add permissions as needed. Common permissions:

| Permission | Use Case |
|-----------|----------|
| `core:default` | Basic window, app, event operations |
| `shell:allow-open` | Open URLs in default browser |
| `fs:default` | File system read/write |
| `dialog:default` | File picker, save dialogs |
| `notification:default` | System notifications |

---

## Rust Backend Patterns

### Command Definition

All Tauri commands follow this pattern:

```rust
use serde::Serialize;

#[derive(Serialize)]
struct SystemInfo {
    platform: String,
    arch: String,
    os_version: String,
}

#[tauri::command]
fn get_system_info() -> SystemInfo {
    SystemInfo {
        platform: std::env::consts::OS.to_string(),
        arch: std::env::consts::ARCH.to_string(),
        os_version: detect_os_version(),
    }
}
```

### Async Commands

For I/O-heavy operations, use `tokio::task::spawn_blocking` to avoid blocking the async runtime:

```rust
#[tauri::command]
async fn scan_directory(path: String) -> Result<Vec<Entry>, String> {
    tokio::task::spawn_blocking(move || {
        // Filesystem traversal happens on a blocking thread
        walk_directory(&path).map_err(|e| e.to_string())
    })
    .await
    .map_err(|e| e.to_string())?
}
```

**Rules:**
- All filesystem operations must use `spawn_blocking`
- CPU-intensive work (hashing, parsing) must use `spawn_blocking`
- Quick computations (string formatting, config reads) can be synchronous
- Always return `Result<T, String>` for fallible commands

### Error Handling

```rust
// ✅ CORRECT: Return Result with user-friendly error messages
#[tauri::command]
async fn delete_file(path: String) -> Result<String, String> {
    std::fs::remove_file(&path)
        .map(|_| format!("Deleted: {}", path))
        .map_err(|e| format!("Failed to delete {}: {}", path, e))
}

// ❌ WRONG: Panic on error
#[tauri::command]
fn delete_file(path: String) {
    std::fs::remove_file(&path).unwrap(); // Will crash the app
}
```

### Safety Patterns

When performing destructive filesystem operations:

1. **Validate paths** — Ensure paths are within expected directories (user home, app data)
2. **Never touch system paths** — Reject operations on `/System`, `/usr`, `/bin`, `/sbin`, `/Applications` (macOS)
3. **Use confirmation** — Frontend must show confirmation dialogs for destructive actions
4. **Prefer OS APIs** — Use Finder/Explorer for trash operations (respects system permissions)

```rust
fn is_safe_path(path: &str) -> bool {
    let home = std::env::var("HOME").unwrap_or_default();
    if home.is_empty() { return false; }

    let protected = ["/System", "/usr", "/bin", "/sbin", "/etc", "/var"];
    !protected.iter().any(|p| path.starts_with(p)) && path.starts_with(&home)
}
```

---

## Frontend Patterns (Standalone Mode)

### Theme System

Use CSS custom properties for dark/light theming:

```css
:root {
  --bg-primary: #1a1a2e;
  --text-primary: #e0e0e0;
  --accent: #4fc3f7;
  /* ... */
}

[data-theme="light"] {
  --bg-primary: #f5f5f5;
  --text-primary: #1c1b1f;
  --accent: #5b4a8a;
}
```

Toggle via JavaScript with localStorage persistence:

```javascript
function setTheme(theme) {
  document.documentElement.setAttribute('data-theme', theme);
  localStorage.setItem('theme', theme);
}
```

### Sidebar Navigation

The standard desktop navigation pattern is a fixed sidebar with view switching:

```javascript
document.querySelectorAll('.nav-item').forEach((item) => {
  item.addEventListener('click', () => {
    // Update active nav item
    document.querySelectorAll('.nav-item').forEach((n) => n.classList.remove('active'));
    item.classList.add('active');

    // Switch view
    document.querySelectorAll('.view').forEach((v) => v.classList.remove('active'));
    document.getElementById(`view-${item.dataset.view}`).classList.add('active');
  });
});
```

### Tauri Command Invocation

```javascript
const { invoke } = window.__TAURI__.core;

// Simple command
const greeting = await invoke('greet', { name: 'World' });

// Async command with error handling
try {
  const result = await invoke('scan_directory', { path: '/Users' });
} catch (err) {
  showToast(`Error: ${err}`, 'error');
}
```

### Window Chrome Integration

Use `-webkit-app-region: drag` to make areas of the UI draggable (acts as a title bar):

```css
/* Sidebar and headers are draggable */
#sidebar { -webkit-app-region: drag; }
.view-header { -webkit-app-region: drag; }

/* Interactive elements must opt out */
.nav-items { -webkit-app-region: no-drag; }
.btn { -webkit-app-region: no-drag; }
```

---

## System Tray Integration

For long-running or background-capable applications:

```rust
use tauri::{
    tray::{TrayIconBuilder, MouseButton, MouseButtonState},
    menu::{Menu, MenuItem},
    Manager,
};

pub fn setup_tray(app: &tauri::App) -> Result<(), Box<dyn std::error::Error>> {
    let menu = Menu::with_items(app, &[
        &MenuItem::with_id(app, "show", "Show Window", true, None::<&str>)?,
        &MenuItem::with_id(app, "quit", "Quit", true, None::<&str>)?,
    ])?;

    TrayIconBuilder::new()
        .menu(&menu)
        .on_menu_event(|app, event| {
            match event.id().as_ref() {
                "show" => { /* show main window */ },
                "quit" => app.exit(0),
                _ => {}
            }
        })
        .build(app)?;

    Ok(())
}
```

**Requirements:**
- Add `"tray-icon"` to Tauri features in `Cargo.toml`
- Add `tray-icon:default` to capabilities

---

## macOS-Specific Considerations

### Disk Information

Use `statvfs` for accurate APFS disk reporting (not `df`):

```rust
use libc::statvfs;

fn get_disk_info(path: &str) -> (u64, u64) {
    let c_path = std::ffi::CString::new(path).unwrap();
    let mut stat: statvfs = unsafe { std::mem::zeroed() };
    unsafe { libc::statvfs(c_path.as_ptr(), &mut stat) };
    let total = stat.f_blocks * stat.f_frsize;
    let available = stat.f_bavail * stat.f_frsize;
    (total, available)
}
```

### SIP and Full Disk Access

System Integrity Protection restricts access to certain directories. Desktop apps should:

1. Never attempt to modify SIP-protected paths
2. Use Finder/osascript for operations requiring Full Disk Access (e.g. emptying Trash)
3. Gracefully handle "Operation not permitted" errors with user-friendly messages
4. Document required permissions in README/distribution notes

### Memory Management

For memory-related operations, prefer `memory_pressure` (no sudo required on macOS 12+):

```rust
fn purge_memory() -> Result<String, String> {
    // Try memory_pressure first (no sudo)
    let result = std::process::Command::new("memory_pressure")
        .arg("-l")
        .arg("critical")
        .output();

    match result {
        Ok(output) if output.status.success() => Ok("Memory freed".into()),
        _ => Err("Memory purge requires elevated permissions".into()),
    }
}
```

---

## Build & Distribution

### Development Commands

```bash
# Development (hot reload)
tauri dev

# Debug build (faster, includes debug symbols)
tauri build --debug

# Production build (optimised, signed)
tauri build
```

### Build Outputs

| Platform | Format | Location |
|----------|--------|----------|
| macOS | `.app`, `.dmg` | `src-tauri/target/release/bundle/macos/` |
| Windows | `.msi`, `.exe` | `src-tauri/target/release/bundle/msi/` |
| Linux | `.AppImage`, `.deb` | `src-tauri/target/release/bundle/appimage/` |

### Code Signing (macOS)

For distribution outside the Mac App Store:

1. Obtain a Developer ID certificate from Apple Developer Program
2. Configure in `tauri.conf.json` under `bundle.macOS`
3. Notarise the app using `xcrun notarytool`
4. Staple the notarisation ticket: `xcrun stapler staple App.app`

---

## Performance Guidelines

1. **Use `spawn_blocking` for I/O** — Never block the async runtime with synchronous filesystem calls
2. **Cache expensive computations** — Store directory sizes, scan results in frontend state
3. **Cancel stale operations** — Use generation counters to discard results from superseded scans
4. **Minimise IPC calls** — Batch data rather than making many small `invoke` calls
5. **Use `walkdir` crate** — More efficient than manual recursive `fs::read_dir`
6. **Set `follows_links: false`** — Avoid infinite loops from circular symlinks

---

## Security Checklist

- [ ] Capabilities file uses minimum necessary permissions
- [ ] CSP configured appropriately for the frontend type
- [ ] Filesystem operations validate paths are within safe directories
- [ ] Destructive operations require user confirmation
- [ ] No hardcoded secrets in Rust source or frontend code
- [ ] System paths are never modified or deleted
- [ ] External URLs opened via Tauri shell plugin (not raw command execution)
- [ ] Error messages don't expose internal file paths or system details to users

---

## Software Licensing (Standalone Apps)

### Architecture Requirements

1. **All license validation must happen in Rust** — The JS frontend calls Tauri commands and receives boolean/status results. It never sees keys, signatures, or validation logic.
2. **Use Cargo feature flags** — Licensing and auth are opt-in via `licensing` and `auth` features. Apps that don't need them compile without extra dependencies.
3. **Offline-first with grace period** — First launch requires online activation. Subsequent launches validate cached data locally. Grace period allows offline use (configurable, default 30 days).
4. **Encrypted local storage** — License data and OAuth tokens stored in encrypted file-backed store using machine-derived key. Missing/corrupted store = unlicensed state.
5. **Machine binding** — License bound to hardware UUID at activation. Activation limits enforced server-side by the licensing provider (e.g., Keygen.sh).

### Provider: Keygen.sh

The standard licensing provider for Forge desktop apps is [Keygen.sh](https://keygen.sh):

- SOC 2 compliant, purpose-built for software licensing
- REST API for validation, machine activation, entitlements
- Ed25519 signed license files for offline verification
- Free tier (100 active licensed users)
- Tauri-specific documentation and community plugins

### Feature Flag Pattern

```toml
# Cargo.toml
[features]
default = []  # No features by default (template mode)
licensing = ["dep:reqwest", "dep:chrono", "dep:urlencoding"]
auth = ["dep:reqwest", "dep:chrono", "dep:urlencoding"]
```

```rust
// lib.rs — conditional module loading
#[cfg(feature = "licensing")]
mod licensing;

#[cfg(feature = "auth")]
mod auth;
```

```html
<!-- index.html — frontend feature flags -->
<html data-feature-licensing="false" data-feature-auth="false">
```

### Configuration

All credentials are compile-time constants via `option_env!()`:

| Variable | Feature | Purpose |
|----------|---------|---------|
| `KEYGEN_ACCOUNT_ID` | licensing | Keygen.sh account identifier |
| `KEYGEN_PRODUCT_ID` | licensing | Product to validate against |
| `KEYGEN_PUBLIC_KEY` | licensing | Ed25519 public key for offline verification |
| `GOOGLE_CLIENT_ID` | auth | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | auth | Google OAuth client secret |

### Standalone Authentication

Desktop apps use **localhost redirect OAuth** for Google authentication:

1. Rust spawns a one-shot TCP listener on a random port
2. Frontend opens Google consent URL in the system browser
3. Google redirects to `http://127.0.0.1:{port}/callback`
4. Rust captures the auth code, exchanges for tokens, stores in secure store
5. Frontend polls `get_auth_status` until authenticated

Connected-mode apps do **not** use standalone auth — they inherit the web app's authentication.

### Security Checklist (Licensing & Auth)

- [ ] All license checks happen in Rust, not JavaScript
- [ ] Credentials stored via `option_env!()` (compile-time), not hardcoded strings
- [ ] Secure store uses machine-derived encryption key
- [ ] Grace period configured (not infinite offline access)
- [ ] Machine binding uses platform-native UUID (not user-controlled identifier)
- [ ] OAuth tokens stored in encrypted store, not localStorage or plain files
- [ ] `http:default` capability added only when licensing or auth features are enabled
- [ ] Connected-mode apps don't include licensing/auth (inherit from web app)
