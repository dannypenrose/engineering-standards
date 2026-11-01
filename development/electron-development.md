# Electron Development

> Authoritative standards for building, securing, packaging and releasing Electron desktop applications, including apps built on a shared Electron shell package.

Related standards: [Desktop Development (Tauri)](/standards/development/desktop-development), [Security Guidelines](/standards/governance/security-guidelines), [Dependency Management](/standards/development/dependency-management), [CI/CD Pipelines](/standards/development/ci-cd-pipelines), [Code Sharing](/standards/development/code-sharing).

## Purpose

An Electron app ships a full browser and a Node.js runtime with the user's
privileges. One loose setting (a renderer with Node access, a preload that
exposes raw IPC, a window that navigates to someone else's page) turns any
script injection into code execution on the user's machine. Packaging has its
own traps: a dependency in the wrong list, a native module built for the wrong
CPU or an unsigned build each produce an app that works in development and
fails for users. This standard fixes one architecture, one security baseline
and one release path so those failures cannot ship.

Use Tauri ([Desktop Development](/standards/development/desktop-development))
for small utilities and connected wrappers of a web app; use Electron when the
app needs Node's ecosystem in-process (native modules such as PTYs, SQLite or
SSH clients) or Chromium's rendering everywhere.

## Core Principles

1. **Shared shell, thin app.** Lifecycle, windows, menu, tray, deep links, updates, the backend host and the security baseline live in one shared shell package. An app supplies configuration, routes and features.
2. **The renderer is untrusted.** It has no Node access, talks only through a narrow preload bridge, and only the app's own pages get that bridge.
3. **No network port.** Local backends are reached over IPC or MessagePorts, never an HTTP server on localhost.
4. **Signed or not shipped.** Every distributed build is code signed (and notarised on macOS); an unsigned build cannot auto-update.
5. **What ships is what `dependencies` says.** Runtime packages for main and the backend are `dependencies`; everything bundled is a `devDependency`.

## Architecture

| Process | Runs | May use | Must not |
| ------- | ---- | ------- | -------- |
| Main | Lifecycle, windows, menus, tray, native dialogs, updater, privileged IPC handlers | Electron and Node APIs | Run long or blocking work on its event loop |
| Preload | The bridge between one window and main | `electron` (`contextBridge`, `ipcRenderer`) and Web APIs only | Import Node built-ins or packages (the preload is sandboxed) |
| Renderer | UI (Vite + React or similar) | DOM and Web APIs, the bridge | Access Node, the file system or the network beyond its CSP |
| Backend (utility process) | Application logic: data, spawning tools, file work | Node APIs, native modules | Listen on a port; call Electron window APIs (send to main instead) |

Backend features are request handlers over a MessagePort that main hands to a
window, written as web-standard `(request) => Response` functions so they are
testable without Electron. Main-process features are small modules with a
fixed lifecycle (register, menu, tray, dispose).

### Shared shell package

| Rule | Detail |
| ---- | ------ |
| App-agnostic | The shell never names or assumes an app. App behaviour comes in through configuration objects, feature modules and options |
| Never patched from an app | A fix goes into the shell repository and reaches apps as a new version; no copies, no edits under `node_modules` |
| Pinned versions | Apps pin an exact version or tag (for a private git dependency, `github:<owner>/<repo>#vX.Y.Z`); never a branch |
| Semver | Breaking changes (removed or renamed exports, changed configuration shapes, IPC channel names, CSP defaults, raised Electron or Node floors) are a major bump, or a minor bump while below 1.0, and say in the changelog what each app must change |
| Tags are immutable | A released tag is never moved or deleted; a broken release is fixed by the next one |
| One folder per process | Shell source is split by process (`main`, `preload`, `backend`, `renderer`, `shared`, `build`); nothing imports across processes except through `shared` |
| Tree-shakeable | The package declares `"sideEffects": false` and has no module-level side effects, so a sandboxed preload never pulls in Node code it does not use |

## Security baseline

### Windows

Every window is created by one shared factory, never with `new BrowserWindow`
in feature code, so no setting can be forgotten:

```ts
webPreferences: {
  preload: preloadPath,
  contextIsolation: true,
  sandbox: true,
  nodeIntegration: false,
}
```

| Rule | Detail |
| ---- | ------ |
| No `nodeIntegration`, ever | Including in "internal" windows and pop-outs |
| No `webviewTag` | Load other content in the system browser instead |
| No `webSecurity: false`, `allowRunningInsecureContent` or `experimentalFeatures` | No exceptions for development |
| Navigation locked to the app origin | `will-navigate` to any other origin is prevented; `setWindowOpenHandler` always returns `deny` and opens `http(s)` and `mailto:` links with `shell.openExternal` |
| External URLs checked | `shell.openExternal` only for an allow-listed scheme (`https:`, `http:`, `mailto:`), never for `file:` or custom schemes from renderer input |

### Preload and IPC

| Rule | Detail |
| ---- | ------ |
| Named functions only | The bridge exposes one function per operation (`files.trash(paths)`), never `ipcRenderer`, a generic `invoke(channel, ...)` or `send(channel, ...)` |
| One channel map | Channel names are constants in one typed map shared by main and preload; no string literals at call sites |
| Validate every argument in main | Check types and ranges of every IPC argument as untrusted input (`typeof url === "string"`), whatever the renderer's types say |
| Check the sender for privileged work | A handler that touches files, runs commands or reveals data checks `event.senderFrame.url` is on the app's own origin |
| Re-check policy in main | A destructive action allowed by the backend is checked again in main before it runs (for example, a delete policy applied in both places) |
| Sandbox-safe preload | Web APIs only (no `Buffer`, `setImmediate` or Node shims since Electron 45); bundled into one file so it `require`s nothing but `electron`. A test or build check enforces it |

### Content and origin

| Rule | Detail |
| ---- | ------ |
| Own scheme in production | Packaged renderers load from a privileged custom scheme (`app://<slug>/`) answered from the app's files, with an SPA fallback to `index.html`. Not `file://` and not localhost |
| Strict CSP on every document | `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self' data:; worker-src 'self' blob:; connect-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'` |
| No inline script or `eval` | Pre-paint scripts are files (`bootstrap.js`), not inline |
| CSP changes are reviewed | Extend the policy one directive at a time with a recorded reason; never add `'unsafe-eval'` or a wildcard source |
| Backend reachable only from the app | Main hands a backend port only to frames on the app origin; the dev server URL alone gives a browser no data |

### Data at rest

- App data lives in one per-app folder (`~/.<slug>` or the platform app-data folder) and survives updates and reinstalls.
- Secrets use the OS keychain (`safeStorage`, async API from Electron 46) or authenticated encryption with a key not stored beside the data in plain text.
- Credentials never reach the renderer: responses carry a placeholder (`[encrypted]`) showing a value is set.
- Commands are built from argument arrays (`execFile`), never shell strings from user or repository data; user paths go after `--`.

## Build and packaging

| Rule | Detail |
| ---- | ------ |
| Runtime `dependencies` only | Exactly the packages main and the backend `require` at runtime (native modules, `electron-updater`). Renderer libraries and the shared shell are `devDependencies` |
| Build check before packaging | After the build, fail if the preload `require`s anything but `electron`, if a main or backend external is not in `dependencies`, or if the output holds databases, `.env` files or keys |
| Relative assets work | The renderer's `index.html` has `<base href="/">` (allowed by `base-uri 'self'`), so nested routes load their assets |
| Native modules unpacked and rebuilt | `asarUnpack` covers `**/*.node`; modules are rebuilt for Electron's ABI on install, and pinned build tools (`node-abi`) know the Electron version |
| Arch check | An after-pack hook fails a macOS build whose native modules do not match the target arch. Build each arch with native modules on a matching runner |
| Development profile | Unpackaged runs use a separate profile and single-instance lock, so a dev build runs beside the installed app |
| Electron upgrades | Read the breaking changes for every major in between, upgrade the shell first, then each app, one at a time |

## Release and auto-update

| Rule | Detail |
| ---- | ------ |
| Tag-triggered | A `vX.Y.Z` tag on `main` runs the release workflow; the version in `package.json` matches the tag |
| Public releases repository | Installers and update manifests publish to a public, releases-only repository; the source can stay private |
| One draft per release | A first job creates the draft release; build jobs upload to it rather than racing to create their own |
| Update manifests list every arch | When Mac arches build on separate runners, a final job merges `latest-mac.yml` to list both, or one arch never updates |
| Least-privilege tokens | A read-only token installs private dependencies; a write token scoped to the releases repository publishes. Checkouts use `persist-credentials: false` so the workflow token does not override them |
| Shared secret names | Signing secrets have the same names in every desktop app, so workflows stay identical |
| Signed and notarised | macOS: Developer ID, hardened runtime, notarisation; Windows: signed when a certificate is available. Unsigned builds are for local testing only |
| Minimal entitlements | Only the entitlements the app needs, each with a comment; remove ones a retired feature needed |
| Updates in the background | The updater runs in packaged builds only, downloads in the background, installs on quit and offers a restart; a manual "Check for Updates" reports errors |
| Check what users download | After publishing, verify the asset names that download pages link to exist on the latest release |

## Machine safety

Electron builds, native rebuilds and test suites are heavy. Run one app at a
time, bound test workers (`--maxWorkers=2`), package the host arch only on a
developer machine, and treat many simultaneous worker failures as starvation
rather than regressions. See [Local Development](/standards/development/local-development).

## Testing

1. Route handlers are tested by calling them with a `Request`; no Electron needed.
2. Pure logic (policies, parsers) is split out of modules that load native code, which do not load under plain Node.
3. A spec guards that the preload imports nothing but `electron`.
4. Delete and file policies have tests against a temporary folder.
5. A smoke test of the packaged app loads the root and a nested route.

## Checklist

- [ ] Every window comes from the shared factory with `contextIsolation`, `sandbox` and no `nodeIntegration`
- [ ] Navigation outside the app origin is blocked and links open in the browser
- [ ] The preload exposes named functions only, and imports nothing but `electron`
- [ ] Every IPC handler validates its arguments; privileged ones check the sender's origin
- [ ] The packaged renderer loads from the app's own scheme with the strict CSP
- [ ] No HTTP server or open port anywhere in the app
- [ ] `dependencies` lists exactly the runtime packages of main and the backend
- [ ] The shared shell is pinned to a tag and never patched from the app
- [ ] Release builds are signed and notarised, and update manifests list every arch
- [ ] Release tokens are scoped to one repository each, and checkouts do not persist credentials

## Related

- [Desktop Development (Tauri)](/standards/development/desktop-development)
- [Security Guidelines](/standards/governance/security-guidelines)
- [CI/CD Pipelines](/standards/development/ci-cd-pipelines)
- [Dependency Management](/standards/development/dependency-management)
- [Code Sharing](/standards/development/code-sharing)
