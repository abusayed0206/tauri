# Tauri MSIX Template

A ready-to-scaffold template for building **Windows desktop apps** with [Tauri v2](https://tauri.app) + Vanilla TypeScript, packaged as an **unsigned MSIX bundle** ready for the **Microsoft Store** (and an NSIS `.exe` installer for sideloading).

This template is pre-wired with [`@choochmeque/tauri-windows-bundle`](https://github.com/Choochmeque/tauri-windows-bundle) so you can go from `git clone` to a Store-ready `.msixbundle` in minutes.

---

## Table of Contents

- [What you get](#what-you-get)
- [Prerequisites](#prerequisites)
- [Quick start (scaffold a new app)](#quick-start-scaffold-a-new-app)
- [Placeholders to replace](#placeholders-to-replace)
- [Project structure](#project-structure)
- [Development commands](#development-commands)
- [Build optimization](#build-optimization)
- [Security defaults](#security-defaults)
- [Building the MSIX bundle](#building-the-msix-bundle)
- [MSIX configuration explained](#msix-configuration-explained)
- [App capabilities](#app-capabilities)
- [Windows extensions (optional)](#windows-extensions-optional)
- [App icons](#app-icons)
- [Code signing (and going unsigned for the Store)](#code-signing-and-going-unsigned-for-the-store)
- [Product identity (Microsoft Store)](#product-identity-microsoft-store)
- [Submitting to the Microsoft Store](#submitting-to-the-microsoft-store)
- [GitHub Actions release pipeline](#github-actions-release-pipeline)
- [Troubleshooting](#troubleshooting)

---

## What you get

- **Tauri v2** app shell (Rust backend + Vite/TypeScript frontend).
- **MSIX packaging pipeline** via `tauri-windows-bundle` — produces an unsigned `.msixbundle` the Store will re-sign.
- **NSIS `.exe` installer** as a bonus sideload artifact.
- **Two GitHub Actions workflows**: `ci.yml` (lint + typecheck + clippy + fmt on every push/PR) and `release.yml` (build + publish on `v*` tags).
- **Dependabot** configured for npm, Cargo, and GitHub Actions — opens weekly PRs to keep packages current.
- **Pre-generated MSIX config**: `bundle.config.json`, `AppxManifest.xml.template`, and tile/logo assets.
- **Windows-specific config overlay** (`tauri.windows.conf.json`) using JSON Merge Patch (RFC 7396).
- **Build optimizations baked in**: static CRT linking, lld linker (2-5x faster), LTO, binary stripping, pinned stable Rust toolchain, and optional Cranelift dev backend.
- **Security defaults**: `withGlobalTauri: false`, strict CSP, deny-by-default capabilities scoped to the main window.
- **Cross-editor consistency**: `.editorconfig` + `.gitattributes` for line-ending normalization.

---

## Prerequisites

Install these once on your machine:

| Tool | Why | Install |
|------|-----|---------|
| **Rust** | Compiles the Tauri backend | https://rustup.rs |
| **Node.js 20+** | Runs the frontend + build scripts | https://nodejs.org |
| **Windows SDK** | Required by the MSIX bundler for `makepri`/signing | via Visual Studio Installer ("Desktop development with C++") |

Add the Windows Rust target (the `rust-toolchain.toml` in this repo auto-installs it, but you can do it manually too):

```powershell
rustup target add x86_64-pc-windows-msvc
```

> **x64 only.** This template targets `x86_64-pc-windows-msvc` exclusively. To build an arm64 MSIX as well, install `aarch64-pc-windows-msvc` and use `npm run msix:build:multi`.

---

## Quick start (scaffold a new app)

```powershell
# 1. Copy this template
git clone <your-template-url> MyApp
cd MyApp

# 2. Install dependencies
npm install

# 3. Do the global search/replace described below
#    (rename placeholders: MyApp, myapp, com.example.myapp, ...)

# 4. Run in dev
npm run tauri:dev

# 5. Build a Store-ready MSIX bundle
npm run msix:build:multi
```

That's it — your `.msixbundle` lands in `src-tauri/target/msix/`.

---

## Placeholders to replace

After cloning, do a **global find-and-replace** across the project for each token below. Use your editor's "Replace in Files" (VS Code: `Ctrl+Shift+H`).

| Placeholder | Replace with | Example | Where it appears |
|-------------|--------------|---------|------------------|
| `MyApp` | Your app's **display name** (keep it a single word, no spaces — see [exe naming](#troubleshooting)) | `ShobdoJaal` | `tauri.conf.json` (`productName`, window `title`), `tauri.windows.conf.json`, `index.html` (`<title>`) |
| `myapp` | Your app's **lowercase name** (matches the Cargo crate name) | `shobdojaal` | `package.json` (`name`), `Cargo.toml` (`name`, lib `name`), `src-tauri/src/main.rs` (`myapp_lib::run()` → `<name>_lib::run()`) |
| `com.example.myapp` | Your **reverse-domain identifier** | `com.sayed.shobdojaal` | `tauri.conf.json` (`identifier`), `tauri.windows.conf.json` (`identifier`) |
| `CN=YourName` | Your **MSIX publisher identity** (from [Product identity](#product-identity-microsoft-store)) | `CN=8EA73F0A-...` (Store) or `CN=Sayed` (local) | `tauri.conf.json` (`bundle.publisher`), `tauri.windows.conf.json` (`bundle.publisher`), `src-tauri/gen/windows/bundle.config.json` (`publisher`) |
| `Your Name` | **Publisher display name** (from Partner Center) | `abusayed.dev` | `src-tauri/gen/windows/bundle.config.json` (`publisherDisplayName`) |
| `A MyApp built with Tauri` | Your **short description** | `A Bengali word game` | `tauri.conf.json` (`shortDescription`, `longDescription`), `Cargo.toml` (`description`) |
| `Utility` | Your **Store category** | `Games` | `tauri.conf.json` (`bundle.category`) |
| `0.1.0` | Your **version** (use semver) | `1.0.0` | `package.json`, `tauri.conf.json`, `Cargo.toml` |

> **Tip:** The `myapp` → `<name>` rename also requires updating `src-tauri/src/main.rs`: change `myapp_lib::run()` to `<name>_lib::run()` (e.g. `shobdojaal_lib::run()`). The `_lib` suffix in `Cargo.toml`'s `[lib].name` must match.

### One-shot rename script (PowerShell)

If you're renaming to a single-word app, this covers most of it:

```powershell
$Display = "ShobdoJaal"      # PascalCase, no spaces
$Lower   = "shobdojaal"      # lowercase crate name
$Id      = "com.sayed.shobdojaal"
$Pub     = "CN=Sayed"
$PubName = "Sayed"
$Desc    = "A Bengali word game"

# Replace in all source/config files (excluding node_modules, target, .git)
Get-ChildItem -Recurse -File -Exclude *.png,*.ico,*.icns,*.svg,*.lock `
  -Path . | Where-Object { $_.FullName -notmatch '\\(node_modules|target|\.git)\\' } |
  ForEach-Object {
    $c = Get-Content $_.FullName -Raw
    $c = $c.Replace('MyApp', $Display).Replace('myapp', $Lower).Replace('com.example.myapp', $Id).Replace('CN=YourName', $Pub).Replace('Your Name', $PubName).Replace('A MyApp built with Tauri', $Desc)
    Set-Content -Path $_.FullName -Value $c -NoNewline
  }
```

> After running the script, also update `src-tauri/src/main.rs` so `myapp_lib::run()` becomes `${Lower}_lib::run()` (the script handles `myapp`→`shobdojaal` so `myapp_lib` → `shobdojaal_lib` automatically).

---

## Project structure

```
.
├── .editorconfig                        # Cross-editor formatting rules
├── .gitattributes                       # Line-ending normalization (LF repo, CRLF checkout for Windows scripts)
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                       # Lint + typecheck + clippy + fmt (every push/PR)
│   │   └── release.yml                  # NSIS + MSIX build → GitHub Release (on v* tags)
│   └── dependabot.yml                   # Weekly dependency update PRs (npm, Cargo, Actions)
├── index.html                           # Frontend entry
├── package.json                         # npm scripts + deps
├── vite.config.ts                       # Vite (dev server on :1420)
├── tsconfig.json                        # Frontend TS config (references tsconfig.node.json)
├── tsconfig.node.json                   # TS config for vite.config.ts (Node context)
├── src/                                 # Frontend (TypeScript + CSS)
│   ├── main.ts
│   ├── styles.css
│   └── assets/
└── src-tauri/
    ├── .cargo/config.toml              # Build opts: static CRT + lld linker (+ nightly hooks)
    ├── Cargo.toml                       # Rust crate (name = your lowercase app name)
    ├── rust-toolchain.toml              # Pins stable Rust + components + x64 target
    ├── tauri.conf.json                  # Main Tauri config (CSP, window defaults, capabilities)
    ├── tauri.windows.conf.json          # Windows-specific overrides (Merge Patch)
    ├── build.rs
    ├── capabilities/default.json        # Tauri permission capabilities (deny-by-default)
    ├── icons/                           # Source icons (run `npm run tauri:icon`)
    ├── src/
    │   ├── main.rs                      # Binary entry → calls <name>_lib::run()
    │   └── lib.rs                       # Tauri builder + commands
    └── gen/windows/                     # MSIX bundler config (generated by `init`)
        ├── bundle.config.json           # MSIX-specific settings
        ├── AppxManifest.xml.template    # AppxManifest template
        └── Assets/                      # Store tile/logo PNGs
```

---

## Development commands

| Command | What it does |
|---------|--------------|
| `npm run dev` | Start the Vite dev server only (frontend) |
| `npm run typecheck` | Typecheck the frontend (`tsc --noEmit`) without emitting |
| `npm run tauri:dev` | Run the full app in dev (Rust + frontend, hot reload) |
| `npm run build` | Build the frontend (`tsc` + `vite build`) |
| `npm run tauri:build` | Build a release bundle (NSIS/MSI via Tauri's own bundler) |
| `npm run tauri:build:debug` | Same, in debug mode |
| `npm run tauri:icon` | Regenerate all platform icons from a source PNG (see [icons](#app-icons)) |
| `npm run msix:build` | Build an **x64 MSIX** via `tauri-windows-bundle` |
| `npm run msix:build:multi` | Build an **x64 + arm64 `.msixbundle`** |
| `npm run msix:build:debug` | Build MSIX in debug mode |
| `npm run msix:init` | Re-run the bundler's `init` (regenerates `gen/windows/` from `tauri.conf.json`) |

> **Daily dev:** `npm run tauri:dev`
> **Ship to Store:** `npm run msix:build:multi`

---

## Build optimization

This template ships with aggressive build optimizations that speed up **dev iteration** and shrink **release binaries** — all configured in two files, no manual flags needed.

### What's configured

| Optimization | Where | Effect |
|---|---|---|
| **lld linker** | `.cargo/config.toml` → `linker = "rust-lld.exe"` | 2-5x faster linking than MSVC `link.exe`. Helps both dev and CI. |
| **Static CRT** | `.cargo/config.toml` → `target-feature=+crt-static` | Statically links the Visual C++ Runtime. The exe/MSIX doesn't depend on the VC++ redistributable being installed — cleaner Store package. |
| **LTO** | `Cargo.toml` → `[profile.release] lto = true` | Link-Time Optimization across all crates. Smaller, faster binary. |
| **codegen-units = 1** | `Cargo.toml` → `[profile.release]` | Lets LLVM optimize across the entire binary instead of splitting into parallel units. |
| **opt-level = "s"** | `Cargo.toml` → `[profile.release]` | Optimizes for binary size. Use `"z"` for even smaller, `"3"` for max CPU speed. |
| **strip = true** | `Cargo.toml` → `[profile.release]` | Removes all debug symbols and tables. Smaller binary, no debug info leaked. |
| **panic = "abort"** | `Cargo.toml` → `[profile.release]` | Disables panic unwinding; removes heavy cleanup code. Only in release — dev builds still unwind for debugging. |

### How it works

- **Dev builds** (`tauri dev`): use the `[profile.dev]` settings + lld linker. Fast iteration via lld. No LTO/strip (those are release-only).
- **Release builds** (`tauri build`, `msix:build`, CI): automatically pick up `[profile.release]` — LTO, single codegen unit, size optimization, stripping, abort-on-panic. The lld linker also speeds up the release link step.
- **CI** (`.github/workflows/release.yml`): sets `CARGO_INCREMENTAL=0` (LTO + incremental don't mix) and `CARGO_NET_RETRY=5` for flaky networks. Everything else comes from the config files.

### Optional: Nightly-only dev speedups

For even faster local `cargo check` / `cargo build`, switch to nightly and enable parallel frontend threads or the Cranelift backend:

```powershell
# Override the toolchain for this project only (rust-toolchain.toml pins stable)
rustup override set nightly

# Install the Cranelift codegen backend component
rustup component add rustc-codegen-cranelift-preview
```

Then uncomment two blocks:

**1. `.cargo/config.toml`** — parallel frontend threads:
```toml
[build]
rustflags = ["-Z", "threads=8"]
```

**2. `Cargo.toml`** — Cranelift for instant dev builds:
```toml
[profile.dev]
codegen-backend = "cranelift"
```

Cranelift compiles unoptimized machine code almost instantly — great for local iteration. It has no effect on release builds (which always use LLVM). **Don't use nightly in CI** — the workflow pins `stable` for reproducibility.

> **Stable users:** skip this section entirely. The lld linker alone gives a big speedup on stable. The `rust-toolchain.toml` in `src-tauri/` pins `stable` + `rustfmt` + `clippy` + the x64 target so every clone starts consistent.

---

## Security defaults

This template ships with production-minded security defaults so you don't start from an insecure baseline:

| Setting | Value | Why |
|---|---|---|
| `app.withGlobalTauri` | `false` | The frontend uses ES module imports (`@tauri-apps/api/core`), so the global `window.__TAURI__` is unnecessary and can cause a race condition on Windows. |
| `app.security.csp` | strict policy | A Content Security Policy is enforced (not `null`). Restricts script/style/img/connect sources to `self`, the custom protocol, and IPC. Tauri auto-injects nonces for bundled code at build time. |
| `capabilities/default.json` | deny-by-default | Permissions are scoped to the `main` window only — not `["*"]`. Includes `core:default` + window/event/app/resources/menu defaults + `opener:default`. Add only what your app needs. |

### Customizing the CSP

The CSP in `tauri.conf.json` is a good baseline. If you load external fonts, images from a CDN, or talk to a backend API, extend the relevant directives:

```jsonc
"csp": {
  "default-src": "'self' customprotocol: asset:",
  "connect-src": "ipc: http://ipc.localhost https://api.yourdomain.com",
  "img-src": "'self' asset: http://asset.localhost blob: data: https:",
  "style-src": "'unsafe-inline' 'self' https://fonts.googleapis.com",
  "font-src": "'self' https://fonts.gstatic.com",
  "object-src": "'none'",
  "base-uri": "'self'",
  "form-action": "'self'",
  "frame-ancestors": "'none'"
}
```

> **Avoid** `dangerousDisableAssetCspModification`, `dangerousRemoteUrlIpcAccess`, and broad `assetProtocol` scopes unless you fully understand the implications.

---

## Building the MSIX bundle

### Local build (unsigned, x64)

```powershell
npm install
npm run msix:build
```

Output:

```
src-tauri/target/msix/
├── MyApp_x64.msix
└── MyApp.msixbundle      # ← upload this to the Store
```

> Need arm64 too? Install the target (`rustup target add aarch64-pc-windows-msvc`) and run `npm run msix:build:multi` instead.

### Useful flags

```powershell
# x64 only (faster, no arm64 target needed)
npm run msix:build

# Debug build
npm run msix:build:debug

# Use a different frontend runner (defaults to cargo)
npm run msix:build -- --runner npm

# Regenerate the Assets/ folder from your icons (overwrites manual edits)
npm run msix:build -- --regenerate-assets

# Set a minimum Windows version (default 10.0.17763.0)
npm run msix:build -- --min-windows 10.0.19041.0
```

### Install the unsigned MSIX locally (for testing)

Unsigned MSIX packages require **Developer Mode** on Windows:

```powershell
# Enable Developer Mode (admin PowerShell)
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" /v AllowDevelopmentWithoutDevLicense /t REG_DWORD /d 1 /f

# Install the bundle
Add-AppxPackage -Path "src-tauri\target\msix\MyApp.msixbundle"
```

---

## MSIX configuration explained

Three files control the MSIX output. The bundler merges them in this priority order (highest wins):

1. **`src-tauri/gen/windows/bundle.config.json`** — MSIX-specific settings (publisher, capabilities, signing, resource index).
2. **`src-tauri/tauri.windows.conf.json`** — Windows-only overrides applied via JSON Merge Patch (RFC 7396) on top of `tauri.conf.json`.
3. **`src-tauri/tauri.conf.json`** — base config (productName, version, identifier, bundle icons/resources).

### `bundle.config.json`

```jsonc
{
  "publisher": "CN=YourName",          // MSIX <Identity Publisher>. For the Store, use your Partner Center publisher.
  "publisherDisplayName": "Your Name", // Shown in the Store/Settings.
  "capabilities": {
    "general": ["internetClient"],     // <Capability>
    "device": [],                      // <DeviceCapability> e.g. webcam, microphone
    "restricted": []                   // <rescap:Capability> e.g. broadFileSystemAccess (needs Store approval)
  },
  "extensions": { ... },               // See Windows extensions below
  "signing": {
    "pfx": null,                       // Path to a .pfx for signing (null = unsigned)
    "pfxPassword": null                // Or set MSIX_PFX_PASSWORD env var
  },
  "resourceIndex": {
    "enabled": false,                  // Set true to generate resources.pri (needed for scale-variant icons)
    "keepConfig": false
  }
}
```

> `runFullTrust` is always auto-added (Tauri apps require it).

### `tauri.windows.conf.json`

Lets you set Windows-only values without touching the cross-platform `tauri.conf.json`. Anything here overrides the base config for Windows builds:

```jsonc
{
  "productName": "MyApp",
  "identifier": "com.example.myapp",
  "bundle": {
    "publisher": "CN=YourName",
    "windows": {
      "certificateThumbprint": null     // For cert-store signing (see signing)
    }
  }
}
```

### `AppxManifest.xml.template`

The bundler fills `{{PLACEHOLDERS}}` at build time:

| Placeholder | Source |
|-------------|--------|
| `{{PACKAGE_NAME}}` | derived from identifier |
| `{{PUBLISHER}}` | `bundle.config.json` → `tauri.conf.json` `bundle.publisher` |
| `{{VERSION}}` | `tauri.conf.json` `version` (auto-padded to 4-part: `1.0.0` → `1.0.0.0`) |
| `{{DISPLAY_NAME}}` | `productName` |
| `{{DESCRIPTION}}` | `bundle.shortDescription` |
| `{{EXECUTABLE}}` | `productName` with whitespace stripped + `.exe` |
| `{{ARCH}}` | `x64` / `arm64` |
| `{{MIN_VERSION}}` | `--min-windows` flag (default `10.0.17763.0`) |
| `{{EXTENSIONS}}` / `{{CAPABILITIES}}` | generated from `bundle.config.json` |

You normally don't edit this template unless you need a custom manifest structure.

---

## App capabilities

Declare what your app can access in `bundle.config.json` → `capabilities`. Three buckets:

```jsonc
"capabilities": {
  "general":   ["internetClient", "internetClientServer", "privateNetworkClientServer"],
  "device":    ["webcam", "microphone", "location", "bluetooth"],
  "restricted": ["broadFileSystemAccess", "allowElevation"]
}
```

- **`general`** → standard `<Capability>` (internet, local network).
- **`device`** → `<DeviceCapability>` (camera, mic, GPS, bluetooth).
- **`restricted`** → `<rescap:Capability>` (requires Store approval).

`runFullTrust` is added automatically — don't add it yourself.

---

## Windows extensions (optional)

Add Store integrations via the bundler CLI:

```powershell
# File type association (e.g. open .ics files)
npx @choochmeque/tauri-windows-bundle extension add file-association

# URL protocol handler (myapp://)
npx @choochmeque/tauri-windows-bundle extension add protocol

# Receive shared content from other apps
npx @choochmeque/tauri-windows-bundle extension add share-target

# Run on Windows login
npx @choochmeque/tauri-windows-bundle extension add startup-task

# Right-click context menu in Explorer
npx @choochmeque/tauri-windows-bundle extension add context-menu

# Run from the command line by app name
npx @choochmeque/tauri-windows-bundle extension add app-execution-alias

# Toast notification activation
npx @choochmeque/tauri-windows-bundle extension add toast-activation

# List what's configured
npx @choochmeque/tauri-windows-bundle extension list

# Remove
npx @choochmeque/tauri-windows-bundle extension remove share-target
```

File associations can also be declared directly in `tauri.conf.json` and are picked up automatically:

```jsonc
"bundle": {
  "fileAssociations": [
    { "ext": ["ics"], "name": "iCalendar", "description": "iCalendar event file" }
  ]
}
```

---

## App icons

1. Put a high-res source PNG (1024×1024 recommended) somewhere, e.g. `app-icon.png`.
2. Generate all Tauri icons:

   ```powershell
   npm run tauri:icon -- app-icon.png
   ```

   This fills `src-tauri/icons/` with all required sizes + `.ico`/`.icns`.

3. Regenerate the MSIX tile assets from those icons:

   ```powershell
   npm run msix:build -- --regenerate-assets
   ```

   Or re-run `npm run msix:init` to rebuild the whole `gen/windows/` folder.

### Scale/theme variants (optional)

For DPI-aware and theme-aware tiles, generate suffixed variants at init:

```powershell
npx @choochmeque/tauri-windows-bundle init --all-variants
```

This enables `resourceIndex.enabled` automatically (scale variants are resolved through `resources.pri`).

---

## Code signing (and going unsigned for the Store)

This template defaults to **unsigned** MSIX (`signing.pfx: null`) — exactly what the Microsoft Store expects, because **the Store re-signs your package on submission**. You do not need a certificate to submit to the Store.

### If you want to sign for sideloading outside the Store

**Option A — PFX file:**

```jsonc
// bundle.config.json
"signing": {
  "pfx": "certs/my-cert.pfx",
  "pfxPassword": null
}
```

```powershell
$env:MSIX_PFX_PASSWORD = "your-password"
npm run msix:build:multi
```

**Option B — Windows certificate store (by thumbprint):**

```jsonc
// tauri.windows.conf.json
"bundle": { "windows": { "certificateThumbprint": "ABC123..." } }
```

---

## Product identity (Microsoft Store)

When you reserve an app name in [Partner Center](https://partner.microsoft.com), the Store assigns your product a **unique identity**. If you build your package manually (as this template does), you must include these identity values in your package manifest — the bundler reads them from `bundle.config.json` and `tauri.conf.json`.

### Where to find your identity

In Partner Center, go to your app → **Product identity** page:

> **Example (Shobdo Jaal):** [partner.microsoft.com/en-us/dashboard/products/9NLC6HGKJV6J/identity](https://partner.microsoft.com/en-us/dashboard/products/9NLC6HGKJV6J/identity)

Replace the product ID (`9NLC6HGKJV6J`) in that URL with your own once you've reserved an app name.

### Identity values and where they go

| Partner Center field | Value (example) | Where to put it in this template |
|---|---|---|
| **Package/Identity/Name** | `abusayed.dev.1798587E48974` | `tauri.conf.json` → `identifier` (and `tauri.windows.conf.json` → `identifier`) |
| **Package/Identity/Publisher** | `CN=8EA73F0A-0DC1-449F-A896-493A34649C49` | `bundle.config.json` → `publisher` (and `tauri.conf.json` → `bundle.publisher`) |
| **Package/Properties/PublisherDisplayName** | `abusayed.dev` | `bundle.config.json` → `publisherDisplayName` |

> The `identifier` and `publisher` together declare your app's identity and establish the **package family** to which all its packages belong. Keep them stable across versions — changing them after publishing breaks Store updates.

### Calculated forms (not in the manifest, but useful)

| Form | Example |
|---|---|
| **Package Family Name (PFN)** | `abusayed.dev.1798587E48974_a7ccwk4vg4vf2` |
| **Package SID** | `S-1-15-2-1979345739-2622596665-787591717-3961879037-3985081424-1149783788-348623872` |

### Store links

| Link type | Example |
|---|---|
| **Store URL** | `https://apps.microsoft.com/detail/9NLC6HGKJV6J` |
| **Store ID** | `9NLC6HGKJV6J` |
| **Store protocol** | `ms-windows-store://pdp/?productid=9NLC6HGKJV6J` |
| **MSA app Id** | `e5a38189-3ebd-4d8b-a914-68d8ebeeae24` |

Share the Store URL with customers so they can find and install your app directly.

### Putting it all together (example)

After reserving your app name, update these three files:

```jsonc
// src-tauri/tauri.conf.json
{
  "identifier": "abusayed.dev.1798587E48974",   // ← Package/Identity/Name
  "bundle": {
    "publisher": "CN=8EA73F0A-0DC1-449F-A896-493A34649C49"  // ← Package/Identity/Publisher
  }
}
```

```jsonc
// src-tauri/gen/windows/bundle.config.json
{
  "publisher": "CN=8EA73F0A-0DC1-449F-A896-493A34649C49",  // ← Package/Identity/Publisher
  "publisherDisplayName": "abusayed.dev"                   // ← Package/Properties/PublisherDisplayName
}
```

Then build and upload — the bundler injects these into the `AppxManifest.xml` automatically.

---

## Submitting to the Microsoft Store

1. **Build the bundle:** `npm run msix:build:multi` → `src-tauri/target/msix/MyApp.msixbundle`.
2. **Create a Partner Center account** at https://partner.microsoft.com (one-time, ~$19).
3. **Reserve a new app name** and copy your **Product identity** values (see [Product identity](#product-identity-microsoft-store) above). Update `bundle.config.json` → `publisher` + `publisherDisplayName`, and `tauri.conf.json` → `identifier` + `bundle.publisher` to match exactly — the `<Identity Publisher>` and `<Identity Name>` in the AppxManifest must match your Partner Center identity.
4. **Upload the package:** Partner Center → your app → *Start submission* → *Packages* → upload `MyApp.msixbundle`. The Store validates and re-signs it.
5. Fill in listing details (description, screenshots, category = `bundle.category`), then submit for certification.

> The `identifier` in `tauri.conf.json` is your app's package family name base — keep it stable across versions; changing it after publishing breaks Store updates.

---

## GitHub Actions pipelines

Two workflows live in `.github/workflows/`:

### CI (`ci.yml`) — runs on every push and pull request

Catches issues before they land on `main`:

- **Frontend job:** `npm ci` → `npm run typecheck` → `npm run build`
- **Rust job:** `cargo fmt --check` → `cargo clippy -- -D warnings` → `cargo test`

Both run on `windows-latest` with Rust caching via `Swatinem/rust-cache`.

### Release (`release.yml`) — runs on `v*` tags

Builds both artifacts and creates a GitHub Release whenever you push a `v*` tag (or run it manually from the Actions tab).

```powershell
git tag v1.0.0
git push origin v1.0.0
```

The workflow will:
1. Set up Node 20 + Rust (x64 target) + caching.
2. Build the **NSIS `.exe`** installer via `tauri-action` (uses the optimized release profile from `Cargo.toml` + lld linker from `.cargo/config.toml`).
3. Build the **unsigned `.msixbundle`** (x64) via `tauri-windows-bundle` (same optimizations apply).
4. Create a GitHub Release with auto-generated notes and attach both files.

> The release profile optimizations (LTO, `opt-level=s`, `strip`, `panic=abort`, static CRT) and the lld linker are all picked up automatically from `Cargo.toml` and `.cargo/config.toml` — no extra CI flags needed.

### Dependabot

`.github/dependabot.yml` opens weekly PRs to bump npm, Cargo, and GitHub Actions dependencies. Merge them to keep the template current.

### Adjusting the workflows

- **Add arm64:** install the `aarch64-pc-windows-msvc` target and use `npm run msix:build:multi` in the release workflow.
- **Skip NSIS:** delete the "Build Tauri app (NSIS)" step and the `NSIS_FILE` reference.
- **Sign in CI:** add your PFX as a repository secret and set `MSIX_PFX_PASSWORD` in the workflow env.
- **Different package manager:** swap `npm` for `pnpm`/`yarn` and pass `--runner pnpm`/`--runner yarn` to the build.

---

## Troubleshooting

### "The exe name in the MSIX doesn't match the built binary"

The bundler names the executable as **`productName` with all whitespace stripped** + `.exe`. Tauri names the compiled binary after the **Cargo crate name** (`Cargo.toml` → `[package].name`).

They must resolve to the same file. Two safe approaches:

- **Recommended:** keep `productName` a **single word with no spaces** (e.g. `MyApp`) and make the Cargo crate name the lowercase equivalent (`myapp`). Windows' case-insensitive filesystem makes `myapp.exe` match the manifest's `MyApp.exe`.
- **Explicit:** set `mainBinaryName` in `tauri.conf.json` to the exact exe name you want (Tauri renames the binary verbatim, no sanitization).

> If your `productName` contains spaces (e.g. `My App`), the bundler expects `MyApp.exe` but Tauri builds `myapp.exe` — the manifest then points at a non-existent file and the package fails to install. **Avoid spaces in `productName`**, or set `mainBinaryName`.

### `aarch64-pc-windows-msvc` build errors

Install the target: `rustup target add aarch64-pc-windows-msvc`. If you don't need arm64, build x64 only (`npm run msix:build`).

### `makepri` / `resources.pri` errors

You only need `resourceIndex.enabled` if you use scale-variant icons (`--scale`, `--target-size`). If you didn't generate variants, leave it `false`.

### MSIX won't install locally (unsigned)

Enable Developer Mode (see [above](#install-the-unsigned-msix-locally-for-testing)) and use `Add-AppxPackage`. Unsigned packages won't install on a locked-down machine — that's expected; the Store handles signing for distribution.

### `tauri-windows-bundle` not found after `npm install`

It's a `devDependency` — make sure you're not installing with `--production`. On non-Windows hosts or with `--omit=optional`, the `msixbundle-cli` sidecar may be missing; run `cargo install msixbundle-cli` to provide it.

### Port 1420 already in use

Vite is pinned to port 1420 (`strictPort: true` in `vite.config.ts`). Kill whatever is using it, or change the port in both `vite.config.ts` and `tauri.conf.json` → `build.devUrl`.

### `rust-lld.exe` not found / linker errors

`rust-lld.exe` ships with Rust and should be on your `PATH` automatically. If you get "linker not found":
- Ensure you're using a recent Rust toolchain (`rustup update`).
- If you installed Rust via a non-standard path, the binary may be in `%USERPROFILE%\.cargo\bin\` or `%RUSTUP_HOME%\toolchains\<stable>\lib\rustlib\x86_64-pc-windows-msvc\bin\`. Add it to `PATH` or remove the `linker` line from `.cargo/config.toml` to fall back to the default MSVC linker.

### Binary is large despite optimizations

Check that you're building in **release** mode (the default for `tauri build` and `msix:build`). Dev builds (`tauri dev`, `msix:build:debug`) skip LTO/strip/size-optimization. If still large, try `opt-level = "z"` in `Cargo.toml` for maximum size reduction.

---

## Recommended IDE Setup

- [VS Code](https://code.visualstudio.com/) + [Tauri](https://marketplace.visualstudio.com/items?itemName=tauri-apps.tauri-vscode) + [rust-analyzer](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer)

---

## Credits

- [Tauri](https://tauri.app) — app framework
- [`@choochmeque/tauri-windows-bundle`](https://github.com/Choochmeque/tauri-windows-bundle) — MSIX bundling tool