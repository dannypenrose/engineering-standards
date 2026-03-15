# Chrome Extension Development Standards

> Authoritative Chrome extension development standards for Manifest V3 extensions, aligned with Chrome Web Store review policies, the Chrome Extensions documentation, and web platform security best practices.

## Purpose

Establish consistent patterns for building, reviewing, and publishing Chrome extensions (Manifest V3) that pass Chrome Web Store review on the first submission. These standards apply to any extension built within the monorepo, regardless of its target application.

## Core Principles

1. **Minimal permissions** — Request only what the extension needs; prefer narrow `host_permissions` with path patterns over broad wildcards
2. **No development artifacts in production** — No localhost URLs, debug logging, or dev-only code paths in submitted builds
3. **Icons at all required sizes** — 16x16, 48x48, 128x128 PNG; referenced in both top-level `icons` and `action.default_icon`
4. **Privacy by default** — Declare a privacy policy URL, explain data usage in the Web Store listing, collect the minimum data possible
5. **Content script isolation** — Content scripts must not leak data between origins or expose sensitive tokens to page contexts

---

## Manifest V3 Requirements

### Required Fields

Every extension must include these fields in `manifest.json`:

```json
{
  "manifest_version": 3,
  "name": "Extension Name",
  "version": "1.0.0",
  "description": "A short description of what the extension does (under 132 characters).",
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

| Field | Requirement | Notes |
|-------|-------------|-------|
| `manifest_version` | Required | Must be `3` for new submissions |
| `name` | Required | 45 characters max |
| `version` | Required | Semver-style (`1.0.0`) |
| `description` | Required | 132 characters max; shown in Web Store search results |
| `icons` | Required for Web Store | All three sizes (16, 48, 128) |

### Recommended Fields

```json
{
  "homepage_url": "https://example.com",
  "action": {
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    },
    "default_popup": "popup.html"
  },
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://example.com/*"],
      "js": ["content.js"],
      "css": ["content.css"]
    }
  ]
}
```

| Field | Purpose |
|-------|---------|
| `homepage_url` | Link shown on Web Store listing; aids review credibility |
| `action.default_icon` | Toolbar icon; must reference the same icons as top-level `icons` |
| `action.default_popup` | HTML file for the popup UI |
| `background.service_worker` | Entry point for the extension's background logic |
| `content_scripts` | Declarative injection into matching pages |

---

## Permission Minimisation

Every permission requested must be justified. Chrome Web Store reviewers will reject extensions with unused or overly broad permissions.

### Permission Reference

| Permission | When Required | Alternatives / Notes |
|-----------|---------------|---------------------|
| `storage` | Almost always. Required for persisting user settings, cached data, or authentication state. | None; this is the standard state management mechanism. |
| `activeTab` | One-time access to the current tab when the user clicks the extension icon. Grants temporary host permission for that tab only. | Preferred over `tabs` when you only need to interact with the page the user is actively viewing. |
| `tabs` | When you need to read tab URLs, titles, or favicons across all tabs, or listen to `onUpdated`/`onCreated` events. | Use `activeTab` if you only need the current tab. Never request both `tabs` and `activeTab` — pick one. |
| `scripting` | For programmatic script injection via `chrome.scripting.executeScript()`. | Not needed if all injection is handled via declarative `content_scripts` in the manifest. |
| `alarms` | For scheduling periodic background tasks (e.g., sync every 30 minutes). | None; required for any recurring background work in Manifest V3 (service workers are ephemeral). |
| `notifications` | For displaying system-level notifications to the user. | Only request if the extension actively notifies users; do not request speculatively. |
| `cookies` | For reading or modifying cookies on specific domains. | Requires corresponding `host_permissions` for the target domains. |
| `identity` | For OAuth2 flows via `chrome.identity.getAuthToken()`. | Only needed for Google account authentication; use standard OAuth redirects for other providers. |

### Host Permissions

Always use the narrowest match pattern possible:

```json
// WRONG: Grants access to every page on every site
"host_permissions": ["*://*/*"]

// WRONG: Grants access to all pages on the domain
"host_permissions": ["https://api.example.com/*"]

// CORRECT: Grants access to only the specific API path
"host_permissions": ["https://api.example.com/v1/*"]
```

**Rules:**
- Never use `<all_urls>` or `*://*/*` unless the extension genuinely operates on every website
- Use path patterns to restrict access to the specific endpoints the extension calls
- If the extension only needs access after a user action, prefer `activeTab` over `host_permissions` entirely

### Critical: Do Not Request Both `tabs` and `activeTab`

These permissions overlap. `activeTab` grants temporary access to the active tab on user gesture. `tabs` grants persistent access to all tab metadata. Requesting both signals to reviewers that the developer does not understand the permission model, and the submission may be rejected.

| Scenario | Use |
|----------|-----|
| Read current page URL on icon click | `activeTab` |
| Monitor all tab URL changes | `tabs` |
| Inject script into current tab on click | `activeTab` + `scripting` |
| Read titles of all open tabs | `tabs` |

---

## Project Structure

### Standard Extension Layout

```
apps/{app-name}/extension/
├── manifest.json           # Extension manifest (Manifest V3)
├── popup.html              # Popup UI (opened from toolbar icon)
├── popup.js                # Popup logic
├── popup.css               # Popup styles
├── background.js           # Service worker (background logic)
├── content.js              # Content script (injected into pages)
├── content.css             # Content script styles
├── icons/
│   ├── icon16.png          # Favicon, toolbar (16x16)
│   ├── icon48.png          # Extensions page (48x48)
│   └── icon128.png         # Web Store, install dialog (128x128)
└── assets/                 # Additional static assets
```

### File Responsibilities

| File | Responsibility | Runs In |
|------|---------------|---------|
| `manifest.json` | Extension metadata, permissions, entry points | N/A (declarative) |
| `background.js` | Auth relay, API calls, message routing, alarm handlers | Service worker context |
| `content.js` | DOM interaction, page data extraction, UI injection | Page context (isolated world) |
| `popup.html/js` | User-facing UI, settings, status display | Popup context |

---

## Service Worker Patterns (background.js)

The service worker is the extension's central coordinator. In Manifest V3, it is ephemeral — Chrome terminates it after ~30 seconds of inactivity, so it must be stateless.

### Stateless Design

```javascript
// WRONG: Global state is lost when the service worker terminates
let authToken = null;

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'SET_TOKEN') {
    authToken = message.token; // Lost on next wake
  }
});

// CORRECT: Use chrome.storage for all persistent state
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'SET_TOKEN') {
    chrome.storage.local.set({ authToken: message.token });
  }
  if (message.type === 'GET_TOKEN') {
    chrome.storage.local.get('authToken', (result) => {
      sendResponse({ token: result.authToken });
    });
    return true; // Keep message channel open for async response
  }
});
```

### Message-Based Architecture

All communication between extension contexts (popup, content script, service worker) uses message passing:

```javascript
// content.js — Send data to background
chrome.runtime.sendMessage(
  { type: 'PAGE_DATA', payload: { url: window.location.href, title: document.title } },
  (response) => {
    if (response.success) {
      // Handle acknowledgement
    }
  }
);

// background.js — Route messages
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  switch (message.type) {
    case 'PAGE_DATA':
      handlePageData(message.payload, sender.tab)
        .then((result) => sendResponse({ success: true, data: result }))
        .catch((err) => sendResponse({ success: false, error: err.message }));
      return true; // Required for async sendResponse
    default:
      sendResponse({ success: false, error: 'Unknown message type' });
  }
});
```

### Network Request Error Handling

```javascript
async function fetchFromAPI(endpoint, token) {
  try {
    const response = await fetch(endpoint, {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`API error: ${response.status} ${response.statusText}`);
    }

    return await response.json();
  } catch (error) {
    console.error('API request failed:', error.message);
    // Store error state for popup to display
    await chrome.storage.local.set({
      lastError: { message: error.message, timestamp: Date.now() },
    });
    throw error;
  }
}
```

### No Development URLs in Production

```javascript
// WRONG: Hardcoded environment map left in production builds
const API_URLS = {
  development: 'http://localhost:3052',
  production: 'https://api.example.com',
};

// CORRECT: Single production URL; dev config lives outside the extension
const API_BASE_URL = 'https://api.example.com';
```

If environment-specific URLs are needed during development, use a build step that replaces the value before packaging. The submitted `.zip` must never contain `localhost`, `127.0.0.1`, or other development-only references.

---

## Content Script Patterns

Content scripts run in an isolated world within the page's origin. They share the DOM but not the JavaScript execution context.

### Origin Isolation

Never share sensitive data across origins. A content script injected into `https://site-a.com` must not pass tokens or user data to `https://site-b.com`.

```javascript
// WRONG: Storing tokens in the DOM (accessible to the page's own scripts)
document.body.setAttribute('data-auth-token', token);

// CORRECT: Keep tokens in extension storage, accessed via message passing
chrome.runtime.sendMessage({ type: 'GET_TOKEN' }, (response) => {
  // Use token for this request only, do not persist in page context
  fetchWithToken(response.token);
});
```

### Minimal DOM Interaction

Content scripts should interact with the page DOM as little as possible:

1. **Read what you need** — Extract specific elements or data points, not entire page trees
2. **Clean up injected UI** — Remove any injected elements when the extension is disabled or the page navigates
3. **Avoid global listeners** — Prefer targeted selectors over `MutationObserver` on `document.body`

### CSS Isolation

Injected styles must not affect the host page, and the host page's styles must not break injected UI.

```javascript
// Preferred: Shadow DOM for injected UI
const host = document.createElement('div');
const shadow = host.attachShadow({ mode: 'closed' });
shadow.innerHTML = `
  <style>
    .ext-container { /* Styles scoped to shadow DOM */ }
  </style>
  <div class="ext-container">Extension UI here</div>
`;
document.body.appendChild(host);
```

If Shadow DOM is not feasible, namespace all CSS classes with a unique prefix (e.g., `myext-panel`, `myext-button`) to avoid collisions.

---

## Security Standards

### Storage

| Storage Method | Use For | Security |
|---------------|---------|----------|
| `chrome.storage.local` | Tokens, API keys, sensitive preferences | Encrypted at rest by Chrome; not accessible to page scripts |
| `chrome.storage.sync` | User preferences that sync across devices | Synced via Google account; do not store secrets |
| `localStorage` (in popup) | Never use for sensitive data | Scoped to popup origin but unencrypted |

### Message Validation

Always validate the structure and source of messages:

```javascript
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  // Validate sender is from this extension
  if (sender.id !== chrome.runtime.id) {
    sendResponse({ success: false, error: 'Unauthorized sender' });
    return;
  }

  // Validate message structure
  if (!message.type || typeof message.type !== 'string') {
    sendResponse({ success: false, error: 'Invalid message format' });
    return;
  }

  // Process known message types
  // ...
});
```

### Content Security Policy

Manifest V3 enforces a strict CSP by default. Extensions must comply:

- **No `eval()`** — Dynamic code execution is blocked
- **No inline scripts** — All JavaScript must be in separate `.js` files
- **No inline event handlers** — Use `addEventListener()` instead of `onclick="..."`

```html
<!-- WRONG: Inline script in popup -->
<button onclick="handleClick()">Click</button>
<script>
  function handleClick() { /* ... */ }
</script>

<!-- CORRECT: External script with event listener -->
<button id="action-btn">Click</button>
<script src="popup.js"></script>
```

```javascript
// popup.js
document.getElementById('action-btn').addEventListener('click', () => {
  // Handle click
});
```

### HTTPS Only

All external requests from the extension must use HTTPS. HTTP requests will be blocked by Chrome's default CSP and will cause Web Store rejection.

---

## Testing and Validation

### Manual Testing Checklist

Before every submission, verify each item:

- [ ] Load extension unpacked (`chrome://extensions` → Developer mode → Load unpacked)
- [ ] Popup opens and displays correctly
- [ ] All toolbar icon sizes render (check 16px in toolbar, 48px on extensions page, 128px in detail view)
- [ ] Content scripts inject and function on target pages
- [ ] Background service worker handles messages correctly
- [ ] Extension works after Chrome restart (service worker re-initialisation)
- [ ] Permissions prompt shows only expected permissions on install
- [ ] No errors in the extension's service worker console (`chrome://extensions` → Inspect views)
- [ ] No errors in the popup's DevTools console
- [ ] No errors in content script console on target pages

### Localhost Reference Audit

Before packaging for submission, verify no development URLs remain:

```bash
# Run from the extension directory
grep -r "localhost" .
grep -r "127\.0\.0\.1" .
grep -r "0\.0\.0\.0" .
grep -r ":3000\|:3050\|:3052\|:8080" .
```

All four commands must return no results. If any match, the build is not ready for submission.

### Permission Audit

Review the manifest and verify:

1. Every permission in `permissions` is actively used in the codebase
2. Every origin in `host_permissions` is called by the extension
3. No permission can be replaced with a narrower alternative
4. `activeTab` and `tabs` are not both present

```bash
# Check which chrome.* APIs are actually used
grep -r "chrome\.\(storage\|tabs\|scripting\|alarms\|notifications\|cookies\|identity\)" . \
  --include="*.js" --include="*.ts"
```

Compare the output against the declared permissions. Remove any permission that has no corresponding API usage.

---

## Web Store Submission Checklist

Complete every item before submitting to the Chrome Web Store:

- [ ] `manifest_version` is `3`
- [ ] `name` is under 45 characters
- [ ] `description` is under 132 characters and accurately describes functionality
- [ ] `version` follows semver and is incremented from the previous submission
- [ ] `icons` includes 16, 48, and 128 PNG files referenced in both `icons` and `action.default_icon`
- [ ] `homepage_url` is set and points to a live page
- [ ] No `localhost`, `127.0.0.1`, or development URLs anywhere in the extension code
- [ ] Privacy policy URL is prepared and accessible
- [ ] All permissions are justified and documented (reviewers may ask)
- [ ] No unused permissions in the manifest
- [ ] No inline scripts in any HTML file (CSP compliance)
- [ ] All external resources listed in `web_accessible_resources` if needed
- [ ] Screenshots prepared at 1280x800 or 640x400 (at least one required)
- [ ] Promotional images prepared if applicable (440x280 small tile)
- [ ] Extension tested in a clean Chrome profile (no other extensions)
- [ ] Extension tested after Chrome restart (service worker cold start)
- [ ] `grep -r "localhost"` returns no results in the extension directory
- [ ] `grep -r "console.log"` reviewed — remove debug logging or replace with conditional checks

---

## Icon Generation

All three icon sizes (16, 48, 128) must be present as PNG files. Generate them from a single source image to ensure consistency.

### Python Script (Pillow)

```python
#!/usr/bin/env python3
"""Generate Chrome extension icons from a source image."""

from PIL import Image
import sys
import os

SIZES = [16, 48, 128]

def generate_icons(source_path, output_dir="icons"):
    os.makedirs(output_dir, exist_ok=True)
    source = Image.open(source_path)

    for size in SIZES:
        icon = source.resize((size, size), Image.LANCZOS)
        output_path = os.path.join(output_dir, f"icon{size}.png")
        icon.save(output_path, "PNG")
        print(f"Generated {output_path} ({size}x{size})")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python generate_icons.py <source_image.png>")
        sys.exit(1)
    generate_icons(sys.argv[1])
```

**Requirements:**
- Source image should be at least 128x128 pixels
- Use a square image; non-square images will be distorted
- PNG format with transparency support
- Consistent with the application's branding and colour palette

### Alternative: ImageMagick (one-liner)

```bash
for size in 16 48 128; do
  convert source.png -resize ${size}x${size} icons/icon${size}.png
done
```

---

## Common Review Rejection Reasons

Issues that cause Chrome Web Store review rejection, ordered by frequency:

| Reason | Fix |
|--------|-----|
| Missing or incorrect icon sizes | Provide all three sizes (16, 48, 128) in both `icons` and `action.default_icon` |
| Overly broad permissions | Narrow `host_permissions` to specific paths; remove unused permissions |
| Missing privacy policy | Create and link a privacy policy, even if the extension collects no data |
| `localhost` references in code | Remove all development URLs before packaging |
| Inline scripts in HTML | Move all JavaScript to external `.js` files |
| Description does not match functionality | Ensure the Web Store description accurately reflects what the extension does |
| Missing `homepage_url` | Add a link to the extension's website or repository |
| Deceptive functionality | Ensure the extension does exactly what it claims — no hidden behaviour |

---

## Build and Packaging

### Creating the Submission Package

The Chrome Web Store accepts a `.zip` file of the extension directory:

```bash
# From the app root
cd apps/{app-name}/extension

# Verify no dev artifacts
grep -r "localhost" . && echo "FAIL: localhost found" && exit 1

# Create zip (exclude hidden files and development artifacts)
zip -r ../extension.zip . \
  -x ".*" \
  -x "__MACOSX/*" \
  -x "*.map" \
  -x "node_modules/*"
```

### Version Bumping

The `version` field in `manifest.json` must be incremented for every Web Store submission. Chrome rejects uploads with a version that is not strictly greater than the current published version.

```bash
# Check current version
grep '"version"' manifest.json
```

Use semver conventions:
- **Patch** (`1.0.1`) — Bug fixes, minor adjustments
- **Minor** (`1.1.0`) — New features, non-breaking changes
- **Major** (`2.0.0`) — Breaking changes, major redesigns

---

## Security Checklist

- [ ] No secrets (API keys, tokens) hardcoded in extension source files
- [ ] Tokens stored in `chrome.storage.local`, not in `localStorage` or cookies
- [ ] All `onMessage` handlers validate `sender.id` matches `chrome.runtime.id`
- [ ] No use of `eval()`, `new Function()`, or dynamic code generation
- [ ] No inline scripts or inline event handlers in HTML files
- [ ] All external HTTP requests use HTTPS
- [ ] Content scripts do not expose tokens or sensitive data to page contexts
- [ ] `host_permissions` use the narrowest possible match patterns
- [ ] `web_accessible_resources` does not expose sensitive files to web pages
- [ ] Error messages shown to users do not reveal internal URLs or implementation details
