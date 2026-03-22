# Mobile Development Standards

> Authoritative mobile development standards for Expo/React Native applications across all projects.

## Purpose

Establish consistent patterns for building, testing, and distributing mobile applications using Expo and React Native within a monorepo environment.

## Core Principles

1. **Platform conventions first** — Follow iOS Human Interface Guidelines and Material Design patterns
2. **Shared-first architecture** — Reuse business logic, types, and API clients across web and mobile
3. **Native when needed** — Use native modules (dev builds) for hardware features; Expo Go for pure-JS development
4. **Offline-aware** — Design for intermittent connectivity from day one
5. **Permission-respectful** — Request only necessary permissions with clear user-facing explanations

---

## Project Structure

### Standard Mobile App Layout

```
apps/{app-name}/
├── App.tsx                  # Root component (providers, navigation)
├── index.js                 # Custom entry point (NOT expo/AppEntry)
├── app.config.ts            # Expo config (dynamic, TypeScript)
├── eas.json                 # EAS Build profiles per environment
├── metro.config.js          # Metro bundler config
├── babel.config.js          # Babel config
├── tsconfig.json            # TypeScript config
├── package.json             # Dependencies
├── assets/                  # Static assets (icons, splash)
├── src/
│   ├── components/          # Reusable UI components
│   ├── screens/             # Screen components (one per route)
│   ├── navigation/          # Navigator definitions
│   ├── hooks/               # Custom hooks (data fetching, state)
│   ├── services/            # API clients, external service wrappers
│   ├── contexts/            # React context providers
│   ├── types/               # TypeScript type definitions
│   └── theme.ts             # Theme configuration
└── __tests__/               # Test files
```

### Entry Point

**Never use `"main": "expo/AppEntry"`** in monorepo environments with pnpm. The hardcoded `../../App` relative import breaks with pnpm's symlinked `node_modules` structure.

```js
// index.js — custom entry point
import { registerRootComponent } from 'expo';
import App from './App';

registerRootComponent(App);
```

Set `"main": "index.js"` in `package.json`.

---

## Environment Configuration

### API URL Management

API URLs must be configurable per build profile. Never hardcode production URLs in source code.

```typescript
// app.config.ts
export default ({ config }) => ({
  ...config,
  extra: {
    apiUrl: process.env.API_URL || 'http://forge-hub-dev.axiomstudio.io',
  },
});
```

```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": { "API_URL": "http://forge-hub-dev.axiomstudio.io" }
    },
    "preview": {
      "distribution": "internal",
      "env": { "API_URL": "https://api.example.com" }
    },
    "production": {
      "env": { "API_URL": "https://api.example.com" }
    }
  }
}
```

### Physical Device Development

Simulators share the host machine's `localhost`, but physical devices cannot reach it. For physical device testing:

1. Use the host machine's local IP address (e.g. `http://192.168.1.50:3000`)
2. Configure via `.env.local` (gitignored)
3. Rebuild after changing — `app.config.ts` reads env vars at build time

---

## Native Permissions & Plugins

### Permission Configuration

All native permissions must be declared in `app.config.ts` with user-facing descriptions explaining why access is needed.

```typescript
plugins: [
  ['expo-camera', {
    cameraPermission: 'Camera access is needed to scan barcodes.',
  }],
  ['expo-location', {
    locationAlwaysAndWhenInUsePermission: 'Location is used to find nearby stores.',
  }],
],
```

### Dev Builds vs Expo Go

| Feature | Expo Go | Dev Build |
|---------|---------|-----------|
| JS-only libraries | Yes | Yes |
| Native modules (camera, etc.) | No | Yes |
| Custom native code | No | Yes |
| Setup required | None | `npx expo prebuild` |
| Distribution | QR scan | Install build |

**Rule**: Any app using native modules (camera, biometrics, NFC, etc.) must use dev builds. Update the README/CLAUDE.md when adding native dependencies.

```bash
# Create/update native projects
npx expo prebuild --clean

# Build and run
npx expo run:ios           # simulator
npx expo run:ios --device  # physical device (requires code signing)
```

---

## Barcode & Camera Integration

### Scanner Pattern

When implementing barcode/QR scanning:

1. **Use `expo-camera`** (SDK 52+ — `expo-barcode-scanner` was removed)
2. **Request permissions gracefully** — Show explanation before requesting, provide fallback UI
3. **Prevent double-scanning** — Use a ref-based lock, not state (avoids re-render race conditions)
4. **Provide visual feedback** — Show viewfinder overlay, loading state during lookup, error with auto-reset
5. **Limit barcode types** — Only scan types relevant to the use case (e.g. `['ean13', 'ean8']` for ISBN)

```typescript
// Anti-pattern: using state for scan lock (re-render delays allow double-fire)
const [scanning, setScanning] = useState(false);

// Correct: using ref for immediate lock
const scanningRef = useRef(false);

const handleScan = async (result) => {
  if (scanningRef.current) return;
  scanningRef.current = true;
  // ... process scan
};
```

### Scanner UX Flow

1. User navigates to scanner → camera opens immediately (if permission granted)
2. Viewfinder overlay shows scan target area
3. Barcode detected → immediate visual feedback ("Looking up...")
4. Result found → confirmation screen with details
5. Not found → error message, auto-reset after 3 seconds
6. User confirms → action taken, navigate back

---

## Navigation Patterns

### Bottom Tab + Stack Structure

```
Root (auth gate)
├── Login Screen
└── Main Tabs
    ├── Tab 1 → Stack Navigator
    │   ├── List Screen
    │   ├── Detail Screen
    │   └── Add/Edit Screen
    ├── Tab 2 → Stack Navigator
    │   └── ...
    └── More Tab → Single Screen
```

### Screen Mode Pattern

For screens with multiple input methods (search, manual entry, scan), use segmented controls:

```typescript
const [mode, setMode] = useState('search');

// Skip KeyboardAvoidingView for camera modes
if (mode === 'scan') {
  return (
    <View>
      <SegmentedButtons value={mode} onValueChange={setMode} buttons={MODES} />
      <ScanMode />
    </View>
  );
}

return (
  <KeyboardAvoidingView>
    <View>
      <SegmentedButtons value={mode} onValueChange={setMode} buttons={MODES} />
      {mode === 'search' ? <SearchMode /> : <ManualMode />}
    </View>
  </KeyboardAvoidingView>
);
```

---

## API Client Standards

### Authentication Pattern

```typescript
// Interceptor-based JWT management
apiClient.interceptors.request.use(async (config) => {
  const token = await SecureStore.getItemAsync('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Automatic 401 refresh with deduplication
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status !== 401 || original._retry) {
      return Promise.reject(error);
    }
    // Deduplicate concurrent refresh attempts
    if (!refreshPromise) {
      refreshPromise = refreshToken().finally(() => { refreshPromise = null; });
    }
    const newToken = await refreshPromise;
    // Retry original request
  }
);
```

### Token Storage

- **Use `expo-secure-store`** for tokens (encrypted, keychain-backed)
- **Never use AsyncStorage** for sensitive data (unencrypted)
- **Clear tokens on logout** — delete from SecureStore, not just state

---

## Code Signing & Distribution

### iOS Code Signing

| Account Type | Cost | Capabilities |
|--------------|------|-------------|
| Free Personal Team | $0 | Run on own device only (7-day cert) |
| Apple Developer Program | $99/year | TestFlight, App Store, ad-hoc (100 devices) |

#### Personal Team Setup (development only)

1. Xcode → Settings → Accounts → add Apple ID
2. Select Personal Team in project signing
3. Change bundle identifier to a unique value
4. Connect device via USB
5. Trust developer certificate on device: Settings → General → VPN & Device Management

### Distribution Methods

| Method | Platform | Cost | Audience | Limit |
|--------|----------|------|----------|-------|
| TestFlight | iOS | $99/yr | Public link | 10,000 testers |
| Ad-hoc (EAS internal) | iOS | $99/yr | Registered UDIDs | 100 devices/year |
| APK direct share | Android | Free | Anyone | Unlimited |
| Google Play Internal Testing | Android | $25 one-time | Email list | Unlimited |
| App Store | iOS | $99/yr | Public | Unlimited |
| Google Play Store | Android | $25 one-time | Public | Unlimited |

### EAS Build Commands

```bash
# Development build (dev client, internal distribution)
eas build --platform ios --profile development

# Preview build (production-like, internal distribution)
eas build --platform ios --profile preview

# Production build (for TestFlight/App Store submission)
eas build --platform ios --profile production

# Submit to TestFlight
eas submit --platform ios

# Android APK (free, no account needed)
eas build --platform android --profile preview
```

### TestFlight Workflow

1. Enrol in Apple Developer Program ($99/year)
2. Configure `eas.json` with production API URL
3. Build: `eas build --platform ios --profile production`
4. Submit: `eas submit --platform ios`
5. In App Store Connect → TestFlight → add testers by email or create public link
6. Testers install via TestFlight app

---

## Monorepo Integration

### pnpm Compatibility

Three critical issues when using Expo in a pnpm monorepo with `node-linker=isolated`:

1. **Custom entry point** — `expo/AppEntry` uses hardcoded `../../App` relative path that breaks with pnpm's `.pnpm` store structure
2. **Metro symlink support** — Add `config.resolver.unstable_enableSymlinks = true` in `metro.config.js`
3. **Lockfile staleness** — When recreating apps with the same package name, run `pnpm --filter @scope/app update <pkg>@<version>` to force re-resolution

### Shared Code Boundaries

Mobile apps typically **cannot** import web-only shared packages (Next.js, Prisma, MUI) directly. Use shared packages only for:

- TypeScript type definitions
- Pure utility functions (no Node.js or browser APIs)
- Validation schemas (Zod, if tree-shakeable)
- API client patterns (if abstracted from platform-specific storage)

For web-only packages, create mobile-specific implementations that follow the same interfaces.

---

## Testing

### Test Framework

- **Unit tests**: Jest + `@testing-library/react-native`
- **Component tests**: Render with providers, test user interactions
- **Integration tests**: Test navigation flows, API mocking with MSW or Axios mocks

### Test Patterns

```typescript
// Always wrap in providers
function renderWithProviders(ui: React.ReactElement) {
  return render(
    <SafeAreaProvider>
      <PaperProvider>
        <NavigationContainer>
          {ui}
        </NavigationContainer>
      </PaperProvider>
    </SafeAreaProvider>
  );
}
```

---

## Performance Guidelines

1. **FlatList over ScrollView** for lists — virtualised rendering for large datasets
2. **Memoize expensive renders** — `React.memo`, `useMemo`, `useCallback` for list items
3. **Optimise images** — Use appropriate sizes, cache with `expo-image` for large galleries
4. **Minimise re-renders** — Use refs for values that don't need to trigger re-renders (scan locks, timers)
5. **Lazy load screens** — React Navigation lazy-loads tabs by default; don't disable this

---

## In-App Purchases (IAP)

### Apple App Store Requirements

**Guideline 3.1.1**: If your app gates features behind a subscription, you MUST use native in-app purchases on iOS. Linking to external purchase pages (e.g. Stripe checkout on your website) will result in rejection.

| Allowed | Not Allowed |
|---------|-------------|
| Native IAP via StoreKit / RevenueCat | `Linking.openURL()` to external payment page |
| "Reader" apps showing no purchase option | Buttons directing users to buy on your website |
| Mentioning premium exists without a link | Telling users about cheaper web pricing |

### Recommended Stack

Use [RevenueCat](https://www.revenuecat.com) (`react-native-purchases`) for IAP. It handles Apple/Google receipt validation, subscription management, and provides webhooks to your backend.

```bash
npx expo install react-native-purchases expo-dev-client
```

**Important**: IAP does NOT work in Expo Go or the iOS Simulator. You must use dev builds (`expo-dev-client`) or release builds on a physical device.

### Dual Subscription Architecture

If your app also offers web-based subscriptions (e.g. Stripe), both paths should update the same user field:

```
Mobile IAP (RevenueCat) ──→ POST /webhooks/revenuecat ──→ User.subscriptionTier = 'pro'
Web (Stripe)            ──→ POST /webhooks/stripe      ──→ User.subscriptionTier = 'pro'
```

- Either active subscription = premium access
- IAP expiry should NOT downgrade if a Stripe sub is still active (and vice versa)
- Track subscription source to handle cancellation webhooks correctly

### RevenueCat Integration Pattern

```typescript
// 1. Create a RevenueCat context
// - Initialize SDK with platform-specific API key
// - Identify user by app user ID (sync with your auth system)
// - Track entitlement state via CustomerInfo listener
// - Expose purchasePackage() and restorePurchases()

// 2. Gate features by checking BOTH sources
const { isPremium } = useAuth();           // Server-side (Stripe)
const { hasIapEntitlement } = useRevenueCat(); // Client-side (IAP)
const hasPremium = isPremium || hasIapEntitlement;

// 3. Backend webhook endpoint
// - Receive RevenueCat events (INITIAL_PURCHASE, RENEWAL, CANCELLATION, etc.)
// - Update User.subscriptionTier accordingly
// - Verify webhook auth header
```

### Testing IAP

- **Sandbox accounts**: Create test Apple IDs in App Store Connect (Users & Access > Sandbox)
- **Accelerated renewals**: Sandbox annual subscriptions renew every 1 hour, monthly every 5 minutes
- **RevenueCat dashboard**: Shows all sandbox transactions in real-time
- **Physical device required**: Sign out of real Apple ID, use sandbox account for purchases

### App Store Submission Checklist

#### Apple App Store (iOS)

- [ ] In-app purchases use native IAP (not external payment links)
- [ ] Privacy policy URL provided
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) declared for iOS 17+
- [ ] App icon 1024x1024 provided
- [ ] Screenshots for required device sizes
- [ ] EAS project ID configured in `app.config.ts`
- [ ] Bundle identifier matches App Store Connect
- [ ] No `localhost` URLs in production builds
- [ ] `expo-dev-client` not included in production builds
- [ ] "Restore Purchases" button accessible in the app

#### Google Play Store (Android)

- [ ] Privacy policy URL provided
- [ ] Content rating questionnaire completed
- [ ] Data safety section completed
- [ ] Signing key configured (Play App Signing via EAS)
- [ ] Target audience declared

---

## Security Checklist

- [ ] Tokens stored in `expo-secure-store` (not AsyncStorage)
- [ ] API URLs use HTTPS in all non-local environments
- [ ] No secrets in `app.config.ts`, `eas.json`, or committed `.env` files
- [ ] Camera/location permissions have clear user-facing descriptions
- [ ] Certificate pinning considered for production API calls
- [ ] Biometric authentication for sensitive operations (if applicable)
- [ ] ProGuard/code obfuscation enabled for Android release builds
