# Security Guidelines

> Authoritative security standards for protecting applications from common vulnerabilities and ensuring compliance.

## Purpose

Establish security practices that protect applications from common vulnerabilities, ensure compliance with security standards, and maintain user trust.

## Core Principles

1. **Defense in depth** - Multiple layers of security controls
2. **Least privilege** - Minimum access required for functionality
3. **Secure by default** - Safe configuration out of the box
4. **Fail securely** - Errors should not expose vulnerabilities
5. **Trust no input** - Validate and sanitize all external data
6. **Audit everything** - Log security-relevant events

## OWASP Top 10 Compliance Matrix

| Vulnerability | Mitigation | Implementation |
| ------------- | ---------- | -------------- |
| A01: Broken Access Control | Role-based access, resource authorization | Auth guards, ownership checks |
| A02: Cryptographic Failures | TLS 1.3, strong algorithms, proper key management | HTTPS only, bcrypt/argon2 |
| A03: Injection | Parameterized queries, input validation | ORM, prepared statements |
| A04: Insecure Design | Threat modeling, secure patterns | Security reviews, STRIDE |
| A05: Security Misconfiguration | Hardening, minimal services | Security headers, CSP |
| A06: Vulnerable Components | Dependency scanning, updates | Dependabot, npm audit |
| A07: Auth Failures | MFA, secure sessions, rate limiting | JWT best practices |
| A08: Integrity Failures | Code signing, dependency verification | Lockfile integrity |
| A09: Logging Failures | Comprehensive logging, no sensitive data | Structured logging |
| A10: SSRF | URL validation, allowlists | Network segmentation |

## Authentication Security

### Password Requirements

```typescript
const PASSWORD_POLICY = {
  minLength: 12,
  maxLength: 128,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true,
  preventCommonPasswords: true,
  preventUserInfo: true,  // No email/name in password
};
```

### Password Storage

```typescript
import * as bcrypt from 'bcrypt';
// OR
import * as argon2 from 'argon2';

// Hash password (use cost factor of 12+ for bcrypt)
const hash = await bcrypt.hash(password, 12);

// Verify password
const isValid = await bcrypt.compare(password, hash);

// Argon2 (preferred for new projects)
const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 65536,
  timeCost: 3,
  parallelism: 4,
});
```

### JWT Best Practices

```typescript
// Token configuration
const JWT_CONFIG = {
  algorithm: 'RS256',           // Use asymmetric for distributed systems
  accessTokenExpiry: '15m',     // Short-lived access tokens
  refreshTokenExpiry: '7d',     // Longer refresh tokens
  issuer: 'your-app',
  audience: 'your-app-users',
};

// Never include in JWT payload:
// - Passwords or secrets
// - Sensitive personal data
// - Data that changes frequently
```

### Session Management

- Regenerate session ID on authentication state change
- Set secure cookie flags: `Secure`, `HttpOnly`, `SameSite=Strict`
- Implement absolute and idle timeouts
- Invalidate sessions on logout (server-side)

### NextAuth.js Cookie Configuration

When running multiple Next.js applications with NextAuth on the same domain (e.g., `localhost` during development), you **must** configure unique cookie names for each app to prevent session conflicts.

**Problem**: By default, NextAuth uses the same cookie names (`next-auth.session-token`, etc.) across all apps. When multiple apps share a domain, they overwrite each other's cookies, causing:

- Logging into one app logs you out of another
- Session data corruption
- Unpredictable authentication state

**Solution**: Configure unique cookie prefixes per application:

```typescript
// lib/auth.ts
export const authOptions: NextAuthOptions = {
  // Use unique cookie names to prevent conflicts with other NextAuth apps
  cookies: {
    sessionToken: {
      name: `my-app.session-token`,  // Use app-specific prefix
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: process.env.NODE_ENV === 'production',
      },
    },
    callbackUrl: {
      name: `my-app.callback-url`,
      options: {
        sameSite: 'lax',
        path: '/',
        secure: process.env.NODE_ENV === 'production',
      },
    },
    csrfToken: {
      name: `my-app.csrf-token`,
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: process.env.NODE_ENV === 'production',
      },
    },
  },
  providers: [
    // ... your providers
  ],
  // ... rest of config
};
```

**Naming Convention**: Use the app name or identifier as the prefix (e.g., `my-app.session-token`, `admin-portal.session-token`).

**Important**: When using custom cookie names, you must also update any `getToken()` calls in middleware:

```typescript
// middleware.ts
import { getToken } from 'next-auth/jwt';

const token = await getToken({
  req: request,
  secret: process.env.NEXTAUTH_SECRET,
  cookieName: 'my-app.session-token',  // Must match your custom cookie name
});
```

## Input Validation

### Validation Rules

```typescript
// Backend validation (NestJS example)
import { IsEmail, IsString, MinLength, MaxLength, Matches } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  @IsEmail()
  @MaxLength(256)
  @Transform(({ value }) => value.toLowerCase().trim())
  email: string;

  @IsString()
  @MinLength(12)
  @MaxLength(128)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/)
  password: string;

  @IsString()
  @MinLength(1)
  @MaxLength(100)
  @Transform(({ value }) => sanitizeHtml(value))
  name: string;
}
```

### SQL Injection Prevention

```typescript
// ALWAYS use parameterized queries
// Correct - Prisma (safe)
const user = await prisma.user.findUnique({
  where: { email: userInput },
});

// Correct - Raw SQL with parameters
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE email = ${userInput}
`;

// NEVER do this
const users = await prisma.$queryRawUnsafe(
  `SELECT * FROM users WHERE email = '${userInput}'`  // SQL INJECTION!
);
```

### XSS Prevention

```typescript
// Frontend: Use framework's built-in escaping
// React automatically escapes by default
<div>{userContent}</div>  // Safe

// Dangerous - only use with sanitized content
<div dangerouslySetInnerHTML={{ __html: sanitizedHtml }} />

// Backend: Set security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'strict-dynamic'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
    },
  },
}));
```

## Security Headers

### Required Headers

```typescript
// NestJS/Express with Helmet
import helmet from 'helmet';

app.use(helmet());
app.use(helmet.contentSecurityPolicy({...}));
app.use(helmet.hsts({ maxAge: 31536000, includeSubDomains: true }));
app.use(helmet.noSniff());
app.use(helmet.frameguard({ action: 'deny' }));
app.use(helmet.referrerPolicy({ policy: 'strict-origin-when-cross-origin' }));
```

### Header Reference

| Header | Value | Purpose |
| ------ | ----- | ------- |
| Strict-Transport-Security | max-age=31536000; includeSubDomains | Force HTTPS |
| Content-Security-Policy | (see above) | Prevent XSS |
| X-Content-Type-Options | nosniff | Prevent MIME sniffing |
| X-Frame-Options | DENY | Prevent clickjacking |
| Referrer-Policy | strict-origin-when-cross-origin | Control referrer info |

## Secrets Management

### Environment Variables

```bash
# .env.example (commit this)
DATABASE_URL=
JWT_SECRET=
API_KEY=

# .env (NEVER commit)
DATABASE_URL=postgresql://user:pass@host/db
JWT_SECRET=your-256-bit-secret
API_KEY=sk_live_xxx
```

### Secret Storage Rules

| Do | Don't |
| -- | ----- |
| Use environment variables | Hardcode secrets in code |
| Use secret managers (Vault, AWS Secrets) | Commit .env files |
| Rotate secrets regularly | Share secrets via Slack/email |
| Use different secrets per environment | Use same secrets everywhere |
| Encrypt secrets at rest | Log secrets |

### Code Patterns

```typescript
// Configuration module
export const config = {
  database: {
    url: requireEnv('DATABASE_URL'),
  },
  jwt: {
    secret: requireEnv('JWT_SECRET'),
    expiresIn: process.env.JWT_EXPIRES_IN || '15m',
  },
};

function requireEnv(key: string): string {
  const value = process.env[key];
  if (!value) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
  return value;
}
```

## Logging Security Events

### What to Log

```typescript
// Security events that MUST be logged
const SECURITY_EVENTS = [
  'user.login.success',
  'user.login.failure',
  'user.logout',
  'user.password.change',
  'user.password.reset.request',
  'user.account.locked',
  'auth.token.refresh',
  'auth.token.revoke',
  'permission.denied',
  'rate.limit.exceeded',
  'suspicious.activity',
];
```

### What NOT to Log

```typescript
// NEVER log these
const NEVER_LOG = [
  'passwords',
  'tokens',
  'session IDs',
  'API keys',
  'credit card numbers',
  'SSN/national IDs',
  'full request bodies with sensitive data',
];

// Correct logging
logger.info('User logged in', { userId: user.id, ip: request.ip });

// WRONG - exposes password
logger.info('Login attempt', { email, password });
```

## Rate Limiting

### Configuration

```typescript
import rateLimit from 'express-rate-limit';

// General API rate limit
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  standardHeaders: true,
  legacyHeaders: false,
});

// Strict limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 5 attempts per 15 minutes
  skipSuccessfulRequests: true,
});

app.use('/api/', apiLimiter);
app.use('/api/v1/auth/login', authLimiter);
```

## Dependency Security

### Scanning

```bash
# Node.js
npm audit
pnpm audit

# .NET
dotnet list package --vulnerable

# Automated scanning
# Use Dependabot, Snyk, or similar
```

### Update Policy

- **Critical vulnerabilities**: Patch within 24 hours
- **High vulnerabilities**: Patch within 7 days
- **Medium vulnerabilities**: Patch within 30 days
- **Low vulnerabilities**: Patch in next release cycle

## Desktop Application Security (Tauri)

### Security Model

Tauri v2 desktop applications have a unique security profile compared to web applications:

- **Frontend is inspectable** — HTML/CSS/JS is embedded but extractable from the binary
- **Rust backend is compiled** — Native code is significantly harder to reverse-engineer
- **IPC bridge** — Frontend communicates with Rust via `invoke()`, which can be intercepted

### Critical Rule: Logic in Rust, Display in JS

All security-sensitive decisions **must** happen in Rust:

```rust
// CORRECT: Rust validates before executing
#[tauri::command]
fn protected_action(app: AppHandle) -> Result<String, String> {
    let license = load_license(&app)?;
    if license.status != LicenseStatus::Valid {
        return Err("Valid license required".into());
    }
    // ... perform the action
}

// WRONG: JS checks and calls Rust
// if (license.valid) { invoke('do_thing') }
// An attacker can patch the JS to skip the check
```

### License Validation

- All license validation in compiled Rust — never in JavaScript
- Machine fingerprinting via platform-native APIs (not user-controllable values)
- Encrypted local storage with machine-derived keys
- Grace period for offline use (not permanent offline access)
- Activation limits enforced server-side by licensing provider

### OAuth in Desktop Apps

- Use localhost redirect pattern (not custom URI schemes for Google)
- Spawn ephemeral TCP listener on random port as redirect URI
- Exchange authorization code for tokens server-side (in Rust)
- Store tokens in encrypted store, never in localStorage or plain files
- Support token refresh to avoid repeated login prompts

### Capability Permissions

Follow least-privilege for Tauri capabilities:

| Scenario | Required Permissions |
|----------|---------------------|
| Basic app | `core:default`, `shell:allow-open` |
| With licensing | Add `http:default` (for Keygen.sh API) |
| With auth | Add `http:default` (for Google OAuth) |

Only add permissions when the corresponding feature is enabled.

## Checklist

### Authentication

- [ ] Passwords hashed with bcrypt/argon2
- [ ] JWT tokens properly configured
- [ ] Session management implemented
- [ ] MFA available for sensitive operations

### Input Handling

- [ ] All input validated server-side
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (output encoding, CSP)
- [ ] File upload validation

### Infrastructure

- [ ] HTTPS enforced everywhere
- [ ] Security headers configured
- [ ] Rate limiting implemented
- [ ] CORS properly configured

### Secrets

- [ ] No hardcoded secrets
- [ ] Environment variables used
- [ ] Secrets not logged
- [ ] Different secrets per environment

### Monitoring

- [ ] Security events logged
- [ ] Alerts for suspicious activity
- [ ] Regular security audits
- [ ] Dependency scanning enabled

## References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CWE Top 25](https://cwe.mitre.org/top25/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
