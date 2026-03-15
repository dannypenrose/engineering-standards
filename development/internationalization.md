# Internationalization (i18n) and Localization (l10n) Standards

This document defines production-grade enterprise standards for building internationalized and localized applications across web, mobile, and backend platforms.

---

## 1. Unicode Handling

### UTF-8 Everywhere

All text data must use UTF-8 encoding at every layer of the stack.

**Enforcement points:**

| Layer | Requirement |
|-------|-------------|
| Source files | Save as UTF-8 (no BOM for most languages, BOM optional for C#/.NET) |
| HTML | `<meta charset="utf-8">` |
| HTTP | `Content-Type: application/json; charset=utf-8` |
| Database | `utf8mb4` character set (MySQL), `UTF8` encoding (PostgreSQL) |
| File I/O | Explicit encoding parameter on all read/write operations |

**React/Next.js:**
```tsx
// next.config.js - Next.js handles UTF-8 by default
// Ensure API routes set proper headers
export async function GET() {
  return new Response(JSON.stringify(data), {
    headers: { 'Content-Type': 'application/json; charset=utf-8' },
  });
}
```

**.NET:**
```csharp
// Program.cs - ensure UTF-8 throughout the pipeline
builder.Services.AddControllers()
    .AddJsonOptions(options => {
        options.JsonSerializerOptions.Encoder =
            System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping;
    });

// Reading files with explicit encoding
var content = await File.ReadAllTextAsync(path, Encoding.UTF8);
```

**Python:**
```python
# Always specify encoding explicitly
with open('translations.json', 'r', encoding='utf-8') as f:
    data = json.load(f)

# Django settings.py
DEFAULT_CHARSET = 'utf-8'
```

### String Normalization

Use Unicode NFC (Canonical Decomposition followed by Canonical Composition) normalization for consistent string comparison and storage.

```typescript
// TypeScript/JavaScript
function normalizeForComparison(input: string): string {
  return input.normalize('NFC');
}

// Always normalize before comparison or storage
const isEqual = normalizeForComparison(a) === normalizeForComparison(b);
```

```csharp
// .NET
string normalized = input.Normalize(NormalizationForm.FormC);

// Case-insensitive comparison respecting locale
int result = string.Compare(a, b, CultureInfo.CurrentCulture,
    CompareOptions.IgnoreCase);
```

```python
# Python
import unicodedata
normalized = unicodedata.normalize('NFC', text)
```

**Rules:**
- Normalize to NFC on input (form submission, API ingestion, file import)
- Store normalized form in the database
- Use locale-aware collation for sorting (`Intl.Collator` in JS, `CultureInfo` in .NET)
- Never use byte-length to measure user-visible string length; use grapheme clusters

```typescript
// Correct string length measurement (grapheme clusters)
function graphemeLength(str: string): number {
  const segmenter = new Intl.Segmenter(undefined, { granularity: 'grapheme' });
  return [...segmenter.segment(str)].length;
}

graphemeLength('👨‍👩‍👧‍👦');  // 1 (not 11)
graphemeLength('cafe\u0301');   // 4 (not 5)
```

---

## 2. Message Formatting

### ICU MessageFormat

ICU MessageFormat is the standard syntax for parameterized, pluralized, and gender-aware messages. It is supported by `next-intl`, `react-intl` (FormatJS), and `i18next` (via plugins).

**Basic syntax:**

```
Simple:        Hello, {name}!
Number:        You have {count, number} items.
Date:          Joined on {date, date, medium}.
Time:          Last seen at {time, time, short}.
Plural:        {count, plural, =0 {No items} one {# item} other {# items}}
Select:        {gender, select, male {He} female {She} other {They}} liked this.
Selectordinal:  It's your {year, selectordinal, one {#st} two {#nd} few {#rd} other {#th}} birthday!
```

### Pluralization

Different languages have different plural categories. Arabic has six forms (zero, one, two, few, many, other). English has two (one, other). Always include the `other` category as a mandatory fallback.

**CLDR Plural Categories:**

| Category | English Example | Used By |
|----------|----------------|---------|
| `zero` | - | Arabic, Latvian, Welsh |
| `one` | 1 item | Most languages |
| `two` | 2 items | Arabic, Hebrew, Slovenian |
| `few` | 3 items | Czech, Polish, Russian |
| `many` | 12 items | Arabic, Polish, Russian |
| `other` | 100 items | All languages (mandatory) |

**next-intl (recommended for Next.js):**

```json
// messages/en.json
{
  "Cart": {
    "items": "{count, plural, =0 {Your cart is empty} one {# item in cart} other {# items in cart}}",
    "ordinal": "You finished in {place, selectordinal, one {#st} two {#nd} few {#rd} other {#th}} place"
  }
}
```

```tsx
import { useTranslations } from 'next-intl';

function CartSummary({ count }: { count: number }) {
  const t = useTranslations('Cart');
  return <p>{t('items', { count })}</p>;
}
```

**i18next (React, React Native):**

```json
// en/translation.json (v4 format)
{
  "cart_items_one": "{{count}} item in cart",
  "cart_items_other": "{{count}} items in cart",
  "cart_items_zero": "Your cart is empty"
}
```

```typescript
// Arabic requires all six forms
// ar/translation.json
{
  "cart_items_zero": "سلة التسوق فارغة",
  "cart_items_one": "عنصر واحد في السلة",
  "cart_items_two": "عنصران في السلة",
  "cart_items_few": "{{count}} عناصر في السلة",
  "cart_items_many": "{{count}} عنصرًا في السلة",
  "cart_items_other": "{{count}} عنصر في السلة"
}
```

**FormatJS / react-intl:**

```tsx
import { FormattedMessage } from 'react-intl';

function CartSummary({ count }: { count: number }) {
  return (
    <FormattedMessage
      id="cart.items"
      defaultMessage="{count, plural, =0 {Cart is empty} one {# item} other {# items}}"
      values={{ count }}
    />
  );
}
```

### Gender and Select

```json
// messages/en.json (next-intl / ICU syntax)
{
  "Activity": {
    "liked": "{gender, select, male {He} female {She} other {They}} liked your post.",
    "combined": "{gender, select, male {He} female {She} other {They}} {count, plural, one {has # notification} other {have # notifications}}."
  }
}
```

**Rules:**
- Maximum nesting depth: 2 levels (e.g., select inside plural)
- Always provide the `other` branch for `plural`, `select`, and `selectordinal`
- Use `#` inside plural/selectordinal branches to insert the formatted number
- Keep messages as flat as possible; prefer separate keys over deeply nested ICU expressions
- Never concatenate translated strings; use a single message with placeholders

---

## 3. Date/Time Handling

### Core Principles

- Store all timestamps in UTC (ISO 8601 format) in the database
- Convert to the user's timezone only at the presentation layer
- Use the `Intl.DateTimeFormat` API (or equivalent) for locale-aware display
- Never hardcode date formats like "MM/DD/YYYY"

### Storage Format

```
// ISO 8601 with timezone offset or Z for UTC
2026-03-15T14:30:00Z
2026-03-15T14:30:00+05:30
```

### Frontend Formatting (TypeScript/React)

```typescript
// Using Intl.DateTimeFormat directly
function formatDate(date: Date, locale: string): string {
  return new Intl.DateTimeFormat(locale, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(date);
}

formatDate(new Date(), 'en-US'); // "March 15, 2026"
formatDate(new Date(), 'de-DE'); // "15. März 2026"
formatDate(new Date(), 'ja-JP'); // "2026年3月15日"
formatDate(new Date(), 'ar-SA'); // "١٥ مارس ٢٠٢٦"
```

**next-intl:**

```tsx
import { useFormatter } from 'next-intl';

function EventDate({ date }: { date: Date }) {
  const format = useFormatter();
  return (
    <time dateTime={date.toISOString()}>
      {format.dateTime(date, {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: 'numeric',
        minute: 'numeric',
        timeZoneName: 'short',
      })}
    </time>
  );
}
```

**Relative time formatting:**

```typescript
function formatRelativeTime(date: Date, locale: string): string {
  const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });
  const diffMs = date.getTime() - Date.now();
  const diffDays = Math.round(diffMs / (1000 * 60 * 60 * 24));

  if (Math.abs(diffDays) < 1) {
    const diffHours = Math.round(diffMs / (1000 * 60 * 60));
    return rtf.format(diffHours, 'hour');
  }
  return rtf.format(diffDays, 'day');
}

formatRelativeTime(yesterday, 'en');  // "yesterday"
formatRelativeTime(yesterday, 'de');  // "gestern"
formatRelativeTime(yesterday, 'ja');  // "昨日"
```

### .NET

```csharp
// Store as DateTimeOffset (preserves timezone info)
DateTimeOffset eventTime = DateTimeOffset.UtcNow;

// Format for display using the user's culture
string formatted = eventTime
    .ToOffset(userTimeZoneOffset)
    .ToString("f", new CultureInfo("de-DE"));
// "Sonntag, 15. März 2026 14:30"

// Relative time (use Humanizer or similar library)
using Humanizer;
DateTime.UtcNow.AddHours(-3).Humanize(culture: new CultureInfo("fr-FR"));
// "il y a 3 heures"
```

### Python

```python
from datetime import datetime, timezone
from babel.dates import format_datetime, format_timedelta

# Store as UTC
event_time = datetime.now(timezone.utc)

# Format for locale
format_datetime(event_time, format='long', locale='de_DE')
# "15. März 2026, 14:30:00 UTC"

# Relative time
format_timedelta(timedelta(hours=-3), locale='fr_FR')
# "il y a 3 heures"
```

---

## 4. Number and Currency Formatting

### Intl.NumberFormat (JavaScript/TypeScript)

```typescript
// Basic number formatting
new Intl.NumberFormat('en-US').format(1234567.89);  // "1,234,567.89"
new Intl.NumberFormat('de-DE').format(1234567.89);  // "1.234.567,89"
new Intl.NumberFormat('hi-IN').format(1234567.89);  // "12,34,567.89" (lakhs)

// Currency
function formatCurrency(
  amount: number,
  currency: string,
  locale: string
): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(amount);
}

formatCurrency(1234.5, 'USD', 'en-US');  // "$1,234.50"
formatCurrency(1234.5, 'EUR', 'de-DE');  // "1.234,50 €"
formatCurrency(1234.5, 'JPY', 'ja-JP');  // "￥1,235" (no decimals for JPY)

// Percentage
new Intl.NumberFormat('en', { style: 'percent' }).format(0.856);  // "86%"

// Units
new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'kilometer-per-hour',
}).format(120);  // "120 km/h"

// Compact notation
new Intl.NumberFormat('en', { notation: 'compact' }).format(1500000);  // "1.5M"
new Intl.NumberFormat('ja', { notation: 'compact' }).format(1500000);  // "150万"
```

**ICU MessageFormat integration (next-intl):**

```json
{
  "Product": {
    "price": "Price: {amount, number, currency}"
  }
}
```

```tsx
const t = useTranslations('Product');
t('price', { amount: 32000.99 }, {
  number: { currency: { style: 'currency', currency: 'EUR' } }
});
// "Price: €32,000.99" (in en locale)
```

### .NET

```csharp
// Number formatting
decimal amount = 1234567.89m;
amount.ToString("N2", new CultureInfo("de-DE"));  // "1.234.567,89"

// Currency formatting
amount.ToString("C", new CultureInfo("ja-JP"));   // "¥1,234,568"
amount.ToString("C", new CultureInfo("en-US"));   // "$1,234,567.89"

// Percentage
(0.856m).ToString("P0", new CultureInfo("en-US")); // "86%"
```

### Python

```python
from babel.numbers import format_currency, format_decimal, format_percent

format_decimal(1234567.89, locale='de_DE')           # "1.234.567,89"
format_currency(1234.5, 'USD', locale='en_US')       # "$1,234.50"
format_currency(1234.5, 'EUR', locale='de_DE')       # "1.234,50\xa0€"
format_percent(0.856, locale='en_US')                 # "86%"
```

**Rules:**
- Never format numbers by hardcoding separators (`.` or `,`)
- Store monetary amounts as integers (cents) or `Decimal` types, never floating point
- Currency code (`USD`, `EUR`, `JPY`) must be stored alongside amounts; format is locale-dependent
- Japanese Yen and similar zero-decimal currencies must not show decimal places
- Indian numbering (lakhs/crores) is handled automatically by `Intl.NumberFormat` with `hi-IN` locale

---

## 5. Right-to-Left (RTL) Support

### HTML Direction

```html
<!-- Set at document level based on locale -->
<html lang="ar" dir="rtl">

<!-- Override for specific elements (e.g., code snippets in RTL page) -->
<code dir="ltr">const x = 42;</code>

<!-- Auto-detect for user-generated content -->
<p dir="auto">{userContent}</p>
```

### CSS Logical Properties

Replace all physical directional properties with logical equivalents. This eliminates the need for separate RTL stylesheets.

| Physical Property | Logical Property |
|-------------------|------------------|
| `margin-left` | `margin-inline-start` |
| `margin-right` | `margin-inline-end` |
| `padding-left` | `padding-inline-start` |
| `padding-right` | `padding-inline-end` |
| `border-left` | `border-inline-start` |
| `text-align: left` | `text-align: start` |
| `float: left` | `float: inline-start` |
| `left: 10px` | `inset-inline-start: 10px` |
| `width` | `inline-size` |
| `height` | `block-size` |

```css
/* WRONG: requires RTL override */
.sidebar {
  margin-left: 16px;
  padding-right: 24px;
  border-left: 2px solid #ccc;
  text-align: left;
}

/* CORRECT: direction-agnostic */
.sidebar {
  margin-inline-start: 16px;
  padding-inline-end: 24px;
  border-inline-start: 2px solid #ccc;
  text-align: start;
}
```

### Flexbox and Grid

Flexbox `row` direction automatically reverses in RTL contexts. No changes needed if you avoid physical alignment values.

```css
/* This automatically works for both LTR and RTL */
.nav {
  display: flex;
  flex-direction: row;
  gap: 16px;
}
```

### Icons and Images

Some icons must be mirrored for RTL (arrows, progress indicators). Others must not (play/pause, clocks, checkmarks).

```css
/* Mirror directional icons in RTL */
[dir="rtl"] .icon-arrow-forward {
  transform: scaleX(-1);
}

/* Never mirror: play buttons, clocks, checkmarks, logos */
```

### React/Next.js Implementation

```tsx
// Layout component with RTL support
import { useLocale } from 'next-intl';

const RTL_LOCALES = ['ar', 'he', 'fa', 'ur'];

function RootLayout({ children }: { children: React.ReactNode }) {
  const locale = useLocale();
  const dir = RTL_LOCALES.includes(locale) ? 'rtl' : 'ltr';

  return (
    <html lang={locale} dir={dir}>
      <body>{children}</body>
    </html>
  );
}
```

**Material-UI RTL support:**

```tsx
import { createTheme, ThemeProvider } from '@mui/material/styles';
import rtlPlugin from 'stylis-plugin-rtl';
import { CacheProvider } from '@emotion/react';
import createCache from '@emotion/cache';
import { prefixer } from 'stylis';

const rtlCache = createCache({
  key: 'muirtl',
  stylisPlugins: [prefixer, rtlPlugin],
});

const ltrCache = createCache({
  key: 'muiltr',
});

function App({ locale }: { locale: string }) {
  const isRtl = RTL_LOCALES.includes(locale);
  const theme = createTheme({ direction: isRtl ? 'rtl' : 'ltr' });

  return (
    <CacheProvider value={isRtl ? rtlCache : ltrCache}>
      <ThemeProvider theme={theme}>
        {/* App content */}
      </ThemeProvider>
    </CacheProvider>
  );
}
```

### React Native RTL

```typescript
import { I18nManager } from 'react-native';
import * as Updates from 'expo-updates';

async function setRTL(isRtl: boolean) {
  if (I18nManager.isRTL !== isRtl) {
    I18nManager.allowRTL(isRtl);
    I18nManager.forceRTL(isRtl);
    // React Native requires a restart for RTL changes
    await Updates.reloadAsync();
  }
}
```

---

## 6. Translation File Formats and Management

### Format Comparison

| Format | Best For | Tooling Support | Translator Friendly |
|--------|----------|-----------------|---------------------|
| **JSON** (flat/nested) | Web apps (i18next, next-intl) | Excellent | Good (simple structure) |
| **PO/POT** (gettext) | Python, PHP, C/C++ | Excellent | Excellent (mature tooling) |
| **XLIFF** (1.2/2.0) | Enterprise, CAT tools | Excellent | Excellent (metadata support) |
| **ARB** | Flutter/Dart | Good | Good |
| **RESX** | .NET | Excellent | Good (Visual Studio integration) |

### JSON Structure (Recommended for Web)

```
locales/
├── en/
│   ├── common.json        # Shared strings (buttons, labels)
│   ├── auth.json           # Authentication module
│   ├── dashboard.json      # Dashboard module
│   └── errors.json         # Error messages
├── de/
│   ├── common.json
│   ├── auth.json
│   ├── dashboard.json
│   └── errors.json
└── ar/
    ├── common.json
    ├── auth.json
    ├── dashboard.json
    └── errors.json
```

**Flat JSON (recommended for TMS compatibility):**

```json
{
  "auth.login.title": "Sign In",
  "auth.login.email_label": "Email address",
  "auth.login.password_label": "Password",
  "auth.login.submit": "Sign In",
  "auth.login.error.invalid_credentials": "Invalid email or password",
  "auth.login.error.account_locked": "Account locked. Try again in {minutes, plural, one {# minute} other {# minutes}}."
}
```

**Nested JSON (better DX, used by next-intl):**

```json
{
  "Auth": {
    "login": {
      "title": "Sign In",
      "emailLabel": "Email address",
      "submit": "Sign In",
      "error": {
        "invalidCredentials": "Invalid email or password"
      }
    }
  }
}
```

### PO Files (Python/Django)

```po
# messages.po
msgid "auth.login.title"
msgstr "Anmelden"

#. This is shown when login fails
msgid "auth.login.error.invalid_credentials"
msgstr "Ungültige E-Mail-Adresse oder Passwort"

msgid "cart.items"
msgid_plural "cart.items_plural"
msgstr[0] "Ein Artikel im Warenkorb"
msgstr[1] "{count} Artikel im Warenkorb"
```

### XLIFF (Enterprise)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xliff version="2.0" xmlns="urn:oasis:names:tc:xliff:document:2.0"
       srcLang="en" trgLang="de">
  <file id="auth">
    <unit id="auth.login.title">
      <segment state="translated">
        <source>Sign In</source>
        <target>Anmelden</target>
      </segment>
      <notes>
        <note category="context">Login page heading</note>
      </notes>
    </unit>
  </file>
</xliff>
```

### .NET Resource Files (.resx)

```xml
<!-- Resources/SharedResource.en.resx -->
<root>
  <data name="LoginTitle" xml:space="preserve">
    <value>Sign In</value>
    <comment>Login page heading</comment>
  </data>
  <data name="InvalidCredentials" xml:space="preserve">
    <value>Invalid email or password</value>
  </data>
</root>
```

### Translation Management System (TMS) Workflow

```
Developer writes code with message keys
        │
        ▼
CI/CD extracts messages (formatjs extract, i18next-parser, pybabel extract)
        │
        ▼
Messages pushed to TMS (Phrase, Crowdin, Lokalise, Transifex)
        │
        ▼
Translators work in TMS (context, screenshots, glossary, TM)
        │
        ▼
CI/CD pulls translated files back (TMS CLI / API)
        │
        ▼
Build includes translations → Deploy
```

**Rules:**
- Source language (typically English) files are the source of truth and live in the repository
- Translated files may be committed or fetched at build time from the TMS
- Never hand-edit translated files directly; always go through the TMS
- Include context and screenshots for translators via TMS metadata
- Set up a glossary for product-specific terms
- Use translation memory (TM) to maintain consistency across versions

---

## 7. Frontend i18n

### next-intl (Recommended for Next.js App Router)

**Setup:**

```
src/
├── i18n/
│   ├── routing.ts          # Locale routing config
│   └── request.ts          # Server request config
├── messages/
│   ├── en.json
│   └── de.json
├── middleware.ts            # Locale detection middleware
└── app/
    └── [locale]/
        ├── layout.tsx
        └── page.tsx
```

```typescript
// src/i18n/routing.ts
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['en', 'de', 'ar', 'ja'],
  defaultLocale: 'en',
});
```

```typescript
// src/i18n/request.ts
import { getRequestConfig } from 'next-intl/server';
import { hasLocale } from 'next-intl';
import { routing } from './routing';

export default getRequestConfig(async ({ requestLocale }) => {
  const requested = await requestLocale;
  const locale = hasLocale(routing.locales, requested)
    ? requested
    : routing.defaultLocale;

  return {
    locale,
    messages: (await import(`../messages/${locale}.json`)).default,
  };
});
```

```typescript
// src/middleware.ts
import createMiddleware from 'next-intl/middleware';
import { routing } from './i18n/routing';

export default createMiddleware(routing);

export const config = {
  matcher: ['/', '/(de|en|ar|ja)/:path*'],
};
```

```tsx
// src/app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl';
import { getMessages, getLocale } from 'next-intl/server';

export default async function LocaleLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const locale = await getLocale();
  const messages = await getMessages();
  const dir = ['ar', 'he', 'fa', 'ur'].includes(locale) ? 'rtl' : 'ltr';

  return (
    <html lang={locale} dir={dir}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

```tsx
// Server Component usage
import { useTranslations } from 'next-intl';

export default function DashboardPage() {
  const t = useTranslations('Dashboard');
  return <h1>{t('title')}</h1>;
}
```

### react-i18next (React, React Native)

```typescript
// i18n/config.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import Backend from 'i18next-http-backend';
import LanguageDetector from 'i18next-browser-languagedetector';

i18n
  .use(Backend)
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    supportedLngs: ['en', 'de', 'ar', 'ja'],
    ns: ['common', 'auth', 'dashboard'],
    defaultNS: 'common',
    interpolation: {
      escapeValue: false, // React already escapes
    },
    backend: {
      loadPath: '/locales/{{lng}}/{{ns}}.json',
    },
    detection: {
      order: ['cookie', 'localStorage', 'navigator', 'htmlTag'],
      caches: ['cookie', 'localStorage'],
    },
  });

export default i18n;
```

```tsx
// Component usage
import { useTranslation, Trans } from 'react-i18next';

function WelcomeBanner({ user }: { user: User }) {
  const { t } = useTranslation('dashboard');

  return (
    <div>
      <h1>{t('welcome', { name: user.name })}</h1>
      <p>{t('cart_items', { count: user.cartCount })}</p>
      <p>
        <Trans i18nKey="terms" t={t}>
          By continuing, you agree to our <a href="/terms">Terms</a>.
        </Trans>
      </p>
    </div>
  );
}
```

### FormatJS / react-intl

```tsx
// Provider setup
import { IntlProvider } from 'react-intl';

function App({ locale, messages }: { locale: string; messages: Record<string, string> }) {
  return (
    <IntlProvider locale={locale} messages={messages} defaultLocale="en">
      <AppContent />
    </IntlProvider>
  );
}

// Component usage
import { FormattedMessage, FormattedNumber, FormattedDate, useIntl } from 'react-intl';

function ProductCard({ product }: { product: Product }) {
  const intl = useIntl();

  return (
    <article aria-label={intl.formatMessage({ id: 'product.aria_label' }, { name: product.name })}>
      <h2>{product.name}</h2>
      <p>
        <FormattedNumber
          value={product.price}
          style="currency"
          currency={product.currency}
        />
      </p>
      <p>
        <FormattedMessage
          id="product.reviews"
          defaultMessage="{count, plural, =0 {No reviews} one {# review} other {# reviews}}"
          values={{ count: product.reviewCount }}
        />
      </p>
      <time>
        <FormattedDate value={product.createdAt} year="numeric" month="long" day="numeric" />
      </time>
    </article>
  );
}
```

### Framework Selection Guide

| Criterion | next-intl | react-i18next | react-intl |
|-----------|-----------|---------------|------------|
| Next.js App Router | First-class | Possible | Possible |
| Server Components | Yes | Limited | Limited |
| ICU MessageFormat | Yes (native) | Via plugin | Yes (native) |
| Type safety | Excellent | Good (with codegen) | Good |
| Bundle size | Small | Medium | Medium |
| React Native | No | Yes | Yes |
| Ecosystem | Next.js focused | Huge (i18next) | FormatJS suite |

**Recommendation:** Use `next-intl` for Next.js projects, `react-i18next` for React Native and non-Next.js React apps.

---

## 8. Backend i18n

### API Response Localization

The backend must support returning localized content based on the client's locale preference.

**Locale resolution order:**
1. Explicit query parameter (`?locale=de`)
2. User preference stored in profile/database
3. `Accept-Language` HTTP header
4. Application default locale

### NestJS Implementation

```typescript
// locale.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

const SUPPORTED_LOCALES = ['en', 'de', 'ar', 'ja'] as const;
type Locale = typeof SUPPORTED_LOCALES[number];

@Injectable()
export class LocaleMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const locale = this.resolveLocale(req);
    req['locale'] = locale;
    res.setHeader('Content-Language', locale);
    res.setHeader('Vary', 'Accept-Language');
    next();
  }

  private resolveLocale(req: Request): Locale {
    // 1. Explicit query parameter
    const queryLocale = req.query['locale'] as string;
    if (queryLocale && this.isSupported(queryLocale)) {
      return queryLocale as Locale;
    }

    // 2. User preference (set by auth middleware)
    const userLocale = req['user']?.preferredLocale;
    if (userLocale && this.isSupported(userLocale)) {
      return userLocale as Locale;
    }

    // 3. Accept-Language header
    const acceptLanguage = req.headers['accept-language'];
    if (acceptLanguage) {
      const preferred = this.parseAcceptLanguage(acceptLanguage);
      const match = preferred.find((lang) => this.isSupported(lang));
      if (match) return match as Locale;
    }

    // 4. Default
    return 'en';
  }

  private isSupported(locale: string): boolean {
    return SUPPORTED_LOCALES.includes(locale as Locale);
  }

  private parseAcceptLanguage(header: string): string[] {
    return header
      .split(',')
      .map((part) => {
        const [lang, q] = part.trim().split(';q=');
        return { lang: lang.split('-')[0], q: q ? parseFloat(q) : 1 };
      })
      .sort((a, b) => b.q - a.q)
      .map((item) => item.lang);
  }
}
```

```typescript
// i18n.service.ts
import { Injectable } from '@nestjs/common';
import * as fs from 'fs';
import * as path from 'path';

@Injectable()
export class I18nService {
  private translations: Map<string, Record<string, string>> = new Map();

  constructor() {
    this.loadTranslations();
  }

  private loadTranslations() {
    const localesDir = path.join(__dirname, '..', 'locales');
    for (const locale of ['en', 'de', 'ar', 'ja']) {
      const filePath = path.join(localesDir, `${locale}.json`);
      if (fs.existsSync(filePath)) {
        const content = fs.readFileSync(filePath, 'utf-8');
        this.translations.set(locale, JSON.parse(content));
      }
    }
  }

  translate(key: string, locale: string, params?: Record<string, string | number>): string {
    const messages = this.translations.get(locale) ?? this.translations.get('en')!;
    let message = messages[key] ?? key;

    if (params) {
      for (const [param, value] of Object.entries(params)) {
        message = message.replace(`{${param}}`, String(value));
      }
    }

    return message;
  }
}
```

```typescript
// Localized error responses
// error.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class LocalizedExceptionFilter implements ExceptionFilter {
  constructor(private readonly i18n: I18nService) {}

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest();
    const response = ctx.getResponse();
    const locale = request['locale'] ?? 'en';
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse() as any;

    const messageKey = exceptionResponse.messageKey ?? 'error.unknown';
    const localizedMessage = this.i18n.translate(messageKey, locale, exceptionResponse.params);

    response.status(status).json({
      statusCode: status,
      message: localizedMessage,
      error: exceptionResponse.error,
    });
  }
}
```

### ASP.NET Core Implementation

```csharp
// Program.cs
builder.Services.AddLocalization(options => options.ResourcesPath = "Resources");

builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    var supportedCultures = new[] { "en", "de", "ar", "ja" };
    options.SetDefaultCulture("en");
    options.AddSupportedCultures(supportedCultures);
    options.AddSupportedUICultures(supportedCultures);

    // Locale resolution order
    options.RequestCultureProviders = new List<IRequestCultureProvider>
    {
        new QueryStringRequestCultureProvider(),      // ?culture=de
        new CookieRequestCultureProvider(),           // Cookie
        new AcceptLanguageHeaderRequestCultureProvider(), // Accept-Language
    };
});

var app = builder.Build();
app.UseRequestLocalization();
```

```csharp
// Controller usage
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IStringLocalizer<ProductsController> _localizer;

    public ProductsController(IStringLocalizer<ProductsController> localizer)
    {
        _localizer = localizer;
    }

    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        var product = _repository.Find(id);
        if (product == null)
        {
            return NotFound(new {
                message = _localizer["ProductNotFound", id].Value
            });
        }
        return Ok(product);
    }
}
```

### Django/Python Implementation

```python
# settings.py
LANGUAGE_CODE = 'en'
USE_I18N = True
USE_L10N = True

LANGUAGES = [
    ('en', 'English'),
    ('de', 'German'),
    ('ar', 'Arabic'),
    ('ja', 'Japanese'),
]

LOCALE_PATHS = [BASE_DIR / 'locale']

MIDDLEWARE = [
    'django.middleware.locale.LocaleMiddleware',
    # ...
]
```

```python
# views.py
from django.utils.translation import gettext as _
from django.utils.translation import ngettext

class CartView(APIView):
    def get(self, request):
        count = request.user.cart.count()
        message = ngettext(
            '%(count)d item in your cart',
            '%(count)d items in your cart',
            count
        ) % {'count': count}

        return Response({'message': message})
```

```python
# FastAPI with Babel
from babel import Locale
from babel.numbers import format_currency

@app.get("/api/products/{product_id}")
async def get_product(product_id: int, accept_language: str = Header("en")):
    locale = Locale.parse(accept_language.split(",")[0].split("-")[0])
    product = await get_product_by_id(product_id)

    return {
        "name": product.name,
        "price": format_currency(product.price, product.currency, locale=locale),
    }
```

---

## 9. Database Design for Multi-Language Content

### Approach 1: Separate Translation Table (Recommended for relational data)

Best for structured content with many translatable fields and a fixed set of entity types.

```sql
-- Core entity table (language-independent data)
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku         VARCHAR(50) NOT NULL UNIQUE,
    price       DECIMAL(10, 2) NOT NULL,
    currency    CHAR(3) NOT NULL DEFAULT 'USD',
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Translation table
CREATE TABLE product_translations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id  UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    locale      VARCHAR(10) NOT NULL,  -- BCP 47 tag: 'en', 'de', 'ar'
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    slug        VARCHAR(255),
    meta_title  VARCHAR(255),
    meta_description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(product_id, locale)
);

-- Index for fast locale lookups
CREATE INDEX idx_product_translations_locale ON product_translations(locale);
CREATE INDEX idx_product_translations_product_locale ON product_translations(product_id, locale);
```

**Prisma schema:**

```prisma
model Product {
  id           String   @id @default(uuid())
  sku          String   @unique
  price        Decimal
  currency     String   @default("USD")
  isActive     Boolean  @default(true)
  translations ProductTranslation[]
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}

model ProductTranslation {
  id          String  @id @default(uuid())
  productId   String
  locale      String
  name        String
  description String?
  slug        String?
  product     Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@unique([productId, locale])
  @@index([locale])
}
```

**Query pattern:**

```typescript
// Fetch product with translation for specific locale, fallback to default
async function getProduct(id: string, locale: string, fallbackLocale = 'en') {
  const product = await prisma.product.findUnique({
    where: { id },
    include: {
      translations: {
        where: {
          locale: { in: [locale, fallbackLocale] },
        },
      },
    },
  });

  if (!product) return null;

  // Prefer requested locale, fall back to default
  const translation =
    product.translations.find((t) => t.locale === locale) ??
    product.translations.find((t) => t.locale === fallbackLocale);

  return {
    id: product.id,
    sku: product.sku,
    price: product.price,
    currency: product.currency,
    name: translation?.name ?? '',
    description: translation?.description ?? '',
  };
}
```

### Approach 2: JSONB Column (Simpler setup, good for PostgreSQL)

Best for content with few translatable fields or when schema flexibility is needed.

```sql
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku         VARCHAR(50) NOT NULL UNIQUE,
    price       DECIMAL(10, 2) NOT NULL,
    currency    CHAR(3) NOT NULL DEFAULT 'USD',
    -- JSONB columns for translatable fields
    name        JSONB NOT NULL DEFAULT '{}',   -- {"en": "Widget", "de": "Gerät"}
    description JSONB NOT NULL DEFAULT '{}',   -- {"en": "A fine widget", "de": "Ein feines Gerät"}
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- GIN index for JSONB queries
CREATE INDEX idx_products_name ON products USING GIN (name);

-- Query: get product with German name, fallback to English
SELECT
    id,
    sku,
    price,
    COALESCE(name->>'de', name->>'en') AS name,
    COALESCE(description->>'de', description->>'en') AS description
FROM products
WHERE id = $1;
```

### Approach Comparison

| Criterion | Separate Table | JSONB Column |
|-----------|---------------|--------------|
| Query complexity | JOIN required | Simple COALESCE |
| Schema flexibility | Rigid (add columns for new fields) | Flexible (add keys anytime) |
| Referential integrity | Full FK support | None for translations |
| Full-text search | Standard indexes | GIN index on JSONB |
| Missing translation detection | Easy (query for missing rows) | Harder (check each key) |
| Performance at scale | Better (normalized, indexable) | Good for small field count |
| TMS integration | Easier (export rows) | Requires custom extraction |

**Recommendation:** Use the separate translation table approach for production applications with professional translation workflows. Use JSONB for prototypes or internal tools with a small number of translatable fields.

---

## 10. Content Management for Translators

### Translation Key Naming Conventions

```
{module}.{feature}.{element_type}.{description}

Examples:
  auth.login.button.submit          → "Sign In"
  auth.login.label.email            → "Email address"
  auth.login.error.invalid_creds    → "Invalid email or password"
  dashboard.stats.title.revenue     → "Total Revenue"
  common.action.save                → "Save"
  common.action.cancel              → "Cancel"
  common.validation.required        → "This field is required"
```

### Context for Translators

Always provide context. A key like `"save"` could mean "Save" (verb/button), "a save" (noun), or "Save 20%!" (promotional).

**next-intl / JSON approach:**

```json
{
  "_comment:Auth.login.submit": "Button label on the login form",
  "Auth": {
    "login": {
      "submit": "Sign In"
    }
  }
}
```

**FormatJS with `description`:**

```tsx
<FormattedMessage
  id="auth.login.submit"
  defaultMessage="Sign In"
  description="Button label on the login form. Keep short (max 15 chars)."
/>
```

**i18next with separate metadata file:**

```json
// en/auth_meta.json (consumed by TMS, not by the app)
{
  "auth.login.submit": {
    "context": "Button label on the login form",
    "maxLength": 15,
    "screenshot": "https://cdn.example.com/screenshots/login-form.png"
  }
}
```

### Translation Workflow Automation

```bash
# Extract messages from source code (FormatJS)
npx formatjs extract 'src/**/*.tsx' --out-file lang/en.json --id-interpolation-pattern '[sha512:contenthash:base64:6]'

# Extract messages (i18next-parser)
npx i18next-parser --config i18next-parser.config.js

# Extract messages (Django)
python manage.py makemessages -l de -l ar -l ja

# Compile messages (Django)
python manage.py compilemessages

# Extract messages (.NET)
# Use ResXManager or similar tooling
```

### Glossary Management

Maintain a glossary of product-specific terms that must be translated consistently:

| English Term | German | Arabic | Japanese | Notes |
|-------------|--------|--------|----------|-------|
| Dashboard | Dashboard | لوحة التحكم | ダッシュボード | Do not translate in German |
| Workspace | Arbeitsbereich | مساحة العمل | ワークスペース | |
| Sign In | Anmelden | تسجيل الدخول | サインイン | Verb form, not "Log In" |

---

## 11. Locale Detection and Negotiation

### Detection Priority (Frontend)

```
1. URL path/subdomain     /de/products    (most explicit, SEO-friendly)
2. User preference        stored in DB     (authenticated users)
3. Cookie/localStorage    locale=de        (returning visitors)
4. Accept-Language header  de-DE,de;q=0.9  (browser default)
5. GeoIP                  Country → locale (last resort, often inaccurate)
6. Application default    en               (ultimate fallback)
```

### next-intl Middleware (Recommended)

```typescript
// middleware.ts
import createMiddleware from 'next-intl/middleware';
import { routing } from './i18n/routing';

export default createMiddleware(routing);

// routing.ts
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['en', 'de', 'ar', 'ja'],
  defaultLocale: 'en',
  localePrefix: 'as-needed', // Only prefix non-default locales
  localeDetection: true,     // Use Accept-Language header
});
```

### Accept-Language Parsing

```typescript
// Robust Accept-Language parser
interface LocalePreference {
  locale: string;
  quality: number;
}

function parseAcceptLanguage(header: string): LocalePreference[] {
  if (!header) return [];

  return header
    .split(',')
    .map((entry) => {
      const [locale, ...params] = entry.trim().split(';');
      const qParam = params.find((p) => p.trim().startsWith('q='));
      const quality = qParam ? parseFloat(qParam.split('=')[1]) : 1.0;
      return { locale: locale.trim(), quality: isNaN(quality) ? 0 : quality };
    })
    .filter((entry) => entry.quality > 0)
    .sort((a, b) => b.quality - a.quality);
}

// BCP 47 matching with fallback
function negotiateLocale(
  acceptLanguage: string,
  supported: readonly string[],
  defaultLocale: string
): string {
  const preferences = parseAcceptLanguage(acceptLanguage);

  for (const pref of preferences) {
    // Exact match: en-US
    if (supported.includes(pref.locale)) return pref.locale;

    // Language match: en-US → en
    const language = pref.locale.split('-')[0];
    if (supported.includes(language)) return language;

    // Find any variant: en-US → en-GB (if en-GB is supported)
    const variant = supported.find((s) => s.startsWith(language + '-'));
    if (variant) return variant;
  }

  return defaultLocale;
}
```

### Mobile (React Native / Expo)

```typescript
import * as Localization from 'expo-localization';
import AsyncStorage from '@react-native-async-storage/async-storage';

const SUPPORTED_LOCALES = ['en', 'de', 'ar', 'ja'];
const LOCALE_STORAGE_KEY = '@app/locale';

async function detectLocale(): Promise<string> {
  // 1. Check stored user preference
  const stored = await AsyncStorage.getItem(LOCALE_STORAGE_KEY);
  if (stored && SUPPORTED_LOCALES.includes(stored)) {
    return stored;
  }

  // 2. Device locale
  const deviceLocales = Localization.getLocales();
  for (const deviceLocale of deviceLocales) {
    const lang = deviceLocale.languageCode;
    if (lang && SUPPORTED_LOCALES.includes(lang)) {
      return lang;
    }
  }

  // 3. Default
  return 'en';
}

async function setLocale(locale: string): Promise<void> {
  await AsyncStorage.setItem(LOCALE_STORAGE_KEY, locale);
  // Trigger i18n library language change
  await i18n.changeLanguage(locale);
}
```

---

## 12. Accessibility Considerations for i18n

### Language Attributes

```html
<!-- Document language -->
<html lang="de" dir="ltr">

<!-- Language changes within content -->
<p>The German word <span lang="de">Schadenfreude</span> has no direct English translation.</p>

<!-- Localized ARIA labels -->
<button aria-label={t('common.action.close')}>
  <CloseIcon />
</button>
```

### Screen Reader Considerations

```tsx
// Announce language changes to screen readers
function LanguageSwitcher() {
  const { i18n } = useTranslation();
  const [announcement, setAnnouncement] = useState('');

  const changeLanguage = async (locale: string) => {
    await i18n.changeLanguage(locale);
    setAnnouncement(t('accessibility.language_changed', { language: LOCALE_NAMES[locale] }));
  };

  return (
    <>
      <select
        onChange={(e) => changeLanguage(e.target.value)}
        aria-label={t('accessibility.select_language')}
      >
        {SUPPORTED_LOCALES.map((locale) => (
          <option key={locale} value={locale} lang={locale}>
            {LOCALE_NAMES[locale]}
          </option>
        ))}
      </select>
      {/* Live region for screen reader announcement */}
      <div role="status" aria-live="polite" className="sr-only">
        {announcement}
      </div>
    </>
  );
}
```

### Translated ARIA Labels and Alt Text

```json
// en.json
{
  "Accessibility": {
    "nav": {
      "main": "Main navigation",
      "breadcrumb": "Breadcrumb",
      "pagination": "Pagination"
    },
    "image": {
      "product_photo": "Photo of {productName}",
      "user_avatar": "{userName}'s profile picture"
    },
    "form": {
      "required_field": "{fieldName} (required)",
      "error_summary": "{count, plural, one {# error found} other {# errors found}}. Please review the form."
    }
  }
}
```

### Date and Number Accessibility

```tsx
// Provide machine-readable values alongside localized display
function LocalizedDate({ date }: { date: Date }) {
  const format = useFormatter();

  return (
    <time dateTime={date.toISOString()}>
      {format.dateTime(date, { dateStyle: 'long' })}
    </time>
  );
}

// For currencies, include the full amount for screen readers
function Price({ amount, currency }: { amount: number; currency: string }) {
  const intl = useIntl();
  const formatted = intl.formatNumber(amount, { style: 'currency', currency });

  return (
    <span aria-label={`${amount} ${currency}`}>
      {formatted}
    </span>
  );
}
```

### Checklist

- [ ] `lang` attribute set on `<html>` and updated on locale change
- [ ] `dir` attribute set correctly for RTL locales
- [ ] All `aria-label`, `aria-describedby`, and `alt` attributes are translated
- [ ] `role="status"` live region announces language changes
- [ ] Form validation messages are translated and linked via `aria-describedby`
- [ ] Date/time elements use `<time>` with `datetime` attribute
- [ ] Focus order makes sense in both LTR and RTL layouts
- [ ] Skip navigation links are translated
- [ ] Error summaries include translated counts

---

## 13. Testing i18n

### Pseudo-Localization

Pseudo-localization replaces source strings with accented/expanded versions to detect i18n issues without waiting for real translations.

**Transformation rules:**

| Technique | Purpose | Example |
|-----------|---------|---------|
| Accent characters | Detect hardcoded strings | `"Save"` becomes `"Šàvé"` |
| String expansion | Test layout flexibility | `"Save"` becomes `"[Šààvéé___]"` (30-40% longer) |
| Bracket wrapping | Spot truncation | `"Save"` becomes `"[Šààvéé]"` |
| RTL markers | Test bidirectional support | Wraps with Unicode RTL marks |
| Preserve placeholders | Verify interpolation | `"{name}"` stays as `"{name}"` |

**Implementation:**

```typescript
// pseudo-locale.ts
const ACCENTS: Record<string, string> = {
  a: 'à', b: 'ƀ', c: 'ç', d: 'ð', e: 'é', f: 'ƒ',
  g: 'ĝ', h: 'ĥ', i: 'î', j: 'ĵ', k: 'ĸ', l: 'ĺ',
  m: 'ɱ', n: 'ñ', o: 'ö', p: 'þ', q: 'ǫ', r: 'ŕ',
  s: 'š', t: 'ţ', u: 'û', v: 'ṽ', w: 'ŵ', x: 'ẋ',
  y: 'ý', z: 'ž',
  A: 'À', B: 'Ɓ', C: 'Ç', D: 'Ð', E: 'É', F: 'Ƒ',
  G: 'Ĝ', H: 'Ĥ', I: 'Î', J: 'Ĵ', K: 'Ķ', L: 'Ĺ',
  M: 'Ṁ', N: 'Ñ', O: 'Ö', P: 'Þ', Q: 'Ǫ', R: 'Ŕ',
  S: 'Š', T: 'Ţ', U: 'Û', V: 'Ṽ', W: 'Ŵ', X: 'Ẋ',
  Y: 'Ý', Z: 'Ž',
};

const PLACEHOLDER_REGEX = /(\{[^}]+\}|{{[^}]+}}|%[sd]|%\([^)]+\)[sd])/g;

function pseudoLocalize(
  message: string,
  options: { expansion?: number; brackets?: boolean } = {}
): string {
  const { expansion = 0.3, brackets = true } = options;

  // Split on placeholders to preserve them
  const parts = message.split(PLACEHOLDER_REGEX);

  let result = parts
    .map((part) => {
      // Preserve placeholders
      if (PLACEHOLDER_REGEX.test(part)) {
        PLACEHOLDER_REGEX.lastIndex = 0;
        return part;
      }
      // Apply accents
      return [...part].map((char) => ACCENTS[char] ?? char).join('');
    })
    .join('');

  // Add expansion padding
  const padLength = Math.ceil(message.length * expansion);
  const padding = '~'.repeat(padLength);
  result = result + padding;

  // Wrap in brackets
  if (brackets) {
    result = `[${result}]`;
  }

  return result;
}

// Usage
pseudoLocalize('Save');
// "[Šàvé~~~]"

pseudoLocalize('Hello, {name}! You have {count} items.');
// "[Ĥéĺĺö, {name}! Ýöû ĥàvé {count} îţéɱš.~~~~~~~~~~~~~~]"
```

**Generate pseudo-locale files:**

```typescript
// scripts/generate-pseudo.ts
import * as fs from 'fs';
import * as path from 'path';

function pseudoLocalizeJson(
  obj: Record<string, any>
): Record<string, any> {
  const result: Record<string, any> = {};
  for (const [key, value] of Object.entries(obj)) {
    if (typeof value === 'string') {
      result[key] = pseudoLocalize(value);
    } else if (typeof value === 'object' && value !== null) {
      result[key] = pseudoLocalizeJson(value);
    } else {
      result[key] = value;
    }
  }
  return result;
}

// Read English source, generate pseudo locale
const en = JSON.parse(fs.readFileSync('messages/en.json', 'utf-8'));
const pseudo = pseudoLocalizeJson(en);
fs.writeFileSync('messages/pseudo.json', JSON.stringify(pseudo, null, 2));
```

### String Expansion Testing

Expected expansion ratios by language:

| Language | Expansion vs English | Example |
|----------|---------------------|---------|
| German | +30-35% | "Save" -> "Speichern" |
| Finnish | +30-40% | "Save" -> "Tallenna" |
| French | +15-20% | "Save" -> "Enregistrer" |
| Japanese | -10-30% (but wider glyphs) | "Save" -> "保存" |
| Arabic | +25% (plus RTL) | "Save" -> "حفظ" |

**Test with expanded pseudo-locale to catch layout issues before real translations arrive.**

### Automated i18n Tests

```typescript
// __tests__/i18n.test.ts
import en from '../messages/en.json';
import de from '../messages/de.json';

describe('i18n completeness', () => {
  const enKeys = extractKeys(en);
  const deKeys = extractKeys(de);

  it('German translations have all English keys', () => {
    const missing = enKeys.filter((key) => !deKeys.includes(key));
    expect(missing).toEqual([]);
  });

  it('German translations have no orphaned keys', () => {
    const orphaned = deKeys.filter((key) => !enKeys.includes(key));
    expect(orphaned).toEqual([]);
  });

  it('no empty translation values', () => {
    const empty = deKeys.filter((key) => {
      const value = getNestedValue(de, key);
      return typeof value === 'string' && value.trim() === '';
    });
    expect(empty).toEqual([]);
  });

  it('all ICU messages parse without errors', () => {
    for (const key of enKeys) {
      const value = getNestedValue(en, key);
      if (typeof value === 'string' && value.includes('{')) {
        expect(() => {
          // Use @formatjs/icu-messageformat-parser
          parse(value);
        }).not.toThrow();
      }
    }
  });

  it('placeholders match between source and translations', () => {
    for (const key of enKeys) {
      const enValue = getNestedValue(en, key);
      const deValue = getNestedValue(de, key);
      if (typeof enValue === 'string' && typeof deValue === 'string') {
        const enPlaceholders = extractPlaceholders(enValue);
        const dePlaceholders = extractPlaceholders(deValue);
        expect(dePlaceholders).toEqual(enPlaceholders);
      }
    }
  });
});

function extractKeys(obj: any, prefix = ''): string[] {
  return Object.entries(obj).flatMap(([key, value]) => {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === 'object' && value !== null) {
      return extractKeys(value, fullKey);
    }
    return [fullKey];
  });
}

function extractPlaceholders(message: string): string[] {
  const matches = message.match(/\{(\w+)/g) ?? [];
  return matches.map((m) => m.slice(1)).sort();
}
```

### Visual Regression Testing for RTL

```typescript
// playwright i18n visual test
import { test, expect } from '@playwright/test';

const LOCALES = ['en', 'de', 'ar', 'ja'];

for (const locale of LOCALES) {
  test(`homepage renders correctly in ${locale}`, async ({ page }) => {
    await page.goto(`/${locale}`);
    await expect(page).toHaveScreenshot(`homepage-${locale}.png`, {
      fullPage: true,
      maxDiffPixelRatio: 0.01,
    });
  });
}

test('RTL layout is correct for Arabic', async ({ page }) => {
  await page.goto('/ar');
  const html = page.locator('html');
  await expect(html).toHaveAttribute('dir', 'rtl');
  await expect(html).toHaveAttribute('lang', 'ar');

  // Verify navigation is right-aligned
  const nav = page.locator('nav');
  const navBox = await nav.boundingBox();
  const pageWidth = page.viewportSize()!.width;
  // In RTL, nav should start from the right side
  expect(navBox!.x + navBox!.width).toBeCloseTo(pageWidth, -1);
});
```

### CI Pipeline Integration

```yaml
# .github/workflows/i18n-checks.yml
name: i18n Quality Checks
on:
  pull_request:
    paths:
      - 'messages/**'
      - 'locales/**'
      - 'src/**/*.tsx'
      - 'src/**/*.ts'

jobs:
  i18n-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check translation completeness
        run: pnpm test -- --testPathPattern=i18n

      - name: Extract messages and check for new untranslated keys
        run: |
          npx formatjs extract 'src/**/*.tsx' --out-file /tmp/extracted.json
          node scripts/check-missing-translations.js /tmp/extracted.json

      - name: Validate ICU message syntax
        run: node scripts/validate-icu-messages.js

      - name: Generate pseudo-locale and run visual tests
        run: |
          node scripts/generate-pseudo.js
          npx playwright test --project=pseudo-locale
```

---

## Summary of Recommendations by Platform

| Concern | React/Next.js | .NET | Python | React Native |
|---------|---------------|------|--------|-------------|
| **i18n library** | next-intl | IStringLocalizer + .resx | Django i18n / Babel | react-i18next |
| **Message format** | ICU MessageFormat | .resx resources | gettext (.po) | i18next JSON v4 |
| **Date/number** | Intl API | CultureInfo | Babel | Intl API |
| **RTL** | CSS logical props + dir attr | CSS logical props | CSS logical props | I18nManager |
| **Locale detection** | Middleware (URL path) | RequestLocalization | LocaleMiddleware | expo-localization |
| **Translation files** | JSON (nested) | .resx | .po / .mo | JSON (namespaced) |
| **TMS integration** | formatjs extract | ResXManager | pybabel extract | i18next-parser |
| **Testing** | Pseudo-locale + Playwright | xUnit + resource checks | pytest + pseudo | Detox + pseudo |

---

## Sources

- [ICU MessageFormat Syntax Guide](http://messageformat.github.io/messageformat/guide/)
- [FormatJS ICU Syntax Reference](https://formatjs.github.io/docs/core-concepts/icu-syntax/)
- [ICU Message Format Guide - Phrase](https://phrase.com/blog/posts/guide-to-the-icu-message-format/)
- [W3C Internationalization Best Practices](https://w3c.github.io/bp-i18n-specdev/)
- [Microsoft Engineering Playbook - i18n](https://microsoft.github.io/code-with-engineering-playbook/non-functional-requirements/internationalization/)
- [Best Database Structure for Multilingual Data - Phrase](https://phrase.com/blog/posts/best-database-structure-for-keeping-multilingual-data/)
- [Multi-Language Database Design - Red Gate](https://www.red-gate.com/blog/multi-language-database-design/)
- [ASP.NET Core Localization - Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/localization?view=aspnetcore-10.0)
- [Django Translation Documentation](https://docs.djangoproject.com/en/6.0/topics/i18n/translation/)
- [Expo Localization Documentation](https://docs.expo.dev/versions/latest/sdk/localization/)
- [Accept-Language Header - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Accept-Language)
- [CSS Logical Properties for RTL/LTR Layouts](https://dev.to/web_dev-usman/stop-fighting-rtl-layouts-use-css-logical-properties-for-better-design-5g3m)
- [RTL Styling 101](https://rtlstyling.com/posts/rtl-styling/)
- [Pseudo-localization for Automated i18n Testing](https://l10n.dev/help/pseudo-localization)
- [i18n for Accessibility Guide - IntlPull](https://intlpull.com/blog/i18n-accessibility-a11y-localization-guide-2026)
- [Internationalization Testing Best Practices - Aqua Cloud](https://aqua-cloud.io/internationalization-testing/)
- [next-intl Documentation](https://next-intl.dev/)
- [i18next Documentation](https://www.i18next.com/)
- [XLIFF Format - Phrase](https://support.phrase.com/hc/en-us/articles/5709613119516--XLIFF-1-2-and-2-0-XML-Localization-Interchange-File-Format-TMS)
