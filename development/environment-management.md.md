# Environment Management Standard

> Authoritative environment management standards ensuring configuration consistency, secret safety, and parity across development, staging, and production.

## Purpose

Define how application environments are named, configured, and managed to ensure consistent behavior across dev, staging, and production with secure secret handling.

## Core Principles

1. **Environment parity** - Config is identical except secrets and scale
2. **Secret isolation** - No secrets in code, config files, or logs
3. **Explicit configuration** - All settings defined in environment
4. **Validation on startup** - App fails fast if configuration is invalid
5. **Immutable production** - Production config cannot be changed at runtime
6. **Audit trail** - All configuration changes logged

## Environment Naming Conventions

### Standard Environment Names

```
development  (local machine)
  ├─ NODE_ENV=development
  ├─ LOG_LEVEL=debug
  └─ Features: Hot reload, verbose errors

staging      (pre-production replica)
  ├─ NODE_ENV=staging
  ├─ LOG_LEVEL=info
  └─ Features: Real data (anonymized), full testing

production   (live environment)
  ├─ NODE_ENV=production
  ├─ LOG_LEVEL=warn
  └─ Features: Real data, performance optimized

preview      (optional: feature preview environment)
  ├─ NODE_ENV=staging
  ├─ LOG_LEVEL=info
  └─ Features: For PR preview links
```

### Naming Rules

```bash
# Environment variable name format
# {NAMESPACE}_{COMPONENT}_{PROPERTY}

# Examples (GOOD)
DATABASE_URL
REDIS_URL
JWT_SECRET
LOG_LEVEL
NODE_ENV
API_RATE_LIMIT_PER_MINUTE

# Examples (BAD)
DB_URL                    # Missing context
database_url              # Wrong case
REDIS_PASS               # Too vague (use URL)
secret123                # Not descriptive
```

## Environment Variable Management

### Build-Time vs Runtime Variables

```javascript
// Next.js example - build-time vs runtime

// BUILD-TIME (compiled into bundle, visible to browser)
// Prefix: NEXT_PUBLIC_*
// Defined in: .env.production (committed)
// Usage: Static configuration, API endpoints for browser
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_ANALYTICS_KEY=ak_1234567890

// RUNTIME (server-side only, not visible in bundle)
// Prefix: None (or custom, e.g., PRIVATE_*)
// Defined in: Environment file on server
// Usage: Database URLs, secrets, API keys
DATABASE_URL=postgresql://user:pass@host/db
JWT_SECRET=base64-encoded-secret-key
API_KEY_STRIPE=sk_live_xxxxxxxxxxxx
```

### Environment File Conventions

Applications use a multi-file convention that separates local development, public build-time, and production secrets:

#### Backend

| File | Purpose | Git Status |
|------|---------|------------|
| `.env.local` | Local development secrets and config | **Gitignored** |
| `.env` | Production secrets (copied to server `/etc` dir) | **Gitignored** |

#### Frontend (Next.js)

| File | Purpose | Git Status |
|------|---------|------------|
| `.env.local` | Local development overrides | **Gitignored** |
| `.env.production` | Public `NEXT_PUBLIC_*` vars (loaded at build time) | **Committed** |
| `.env` | Production secrets (copied to server `/etc` dir) | **Gitignored** |

#### File Contents

```bash
# Backend .env.local (local dev - NEVER COMMIT)
DATABASE_URL=mysql://user:pass@localhost:3306/dev_db
JWT_SECRET=dev-secret-key-for-local-testing-only
NODE_ENV=development
LOG_LEVEL=debug

# Backend .env (production secrets - NEVER COMMIT)
# Maintained locally, copied to /etc/{app_name}.env on server
DATABASE_URL=mysql://user:pass@prod-host:3306/prod_db
JWT_SECRET=production-base64-encoded-secret

# Frontend .env.production (public build-time vars - COMMITTED)
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_SITE_URL=https://example.com
NEXT_PUBLIC_ANALYTICS_KEY=ak_1234567890

# Frontend .env.local (local dev overrides - NEVER COMMIT)
NEXT_PUBLIC_API_URL=http://localhost:3002/api/v1
NEXT_PUBLIC_SITE_URL=http://localhost:3000

# Frontend .env (production secrets - NEVER COMMIT)
# Server-side secrets only (NextAuth, backend URL, etc.)
NEXTAUTH_SECRET=production-secret
BACKEND_URL=http://localhost:3002
```

#### Gitignore Rules

```gitignore
# Backend
.env          # Production secrets
.env*.local   # Local dev

# Frontend
.env          # Production secrets
.env*.local   # Local dev
# .env.production is NOT ignored (committed, public vars only)
```

> **Warning: `.env.local` overrides `.env.production` during local development.** Next.js loads env files in this order: `.env` → `.env.local` → `.env.{NODE_ENV}` → `.env.{NODE_ENV}.local`. If `.env.local` defines a `NEXT_PUBLIC_*` variable, it silently overrides the value in `.env.production`. This means a wrong value in `.env.production` will work fine locally but break in production (where only `.env.production` is used during `next build`). Always verify `.env.production` values independently — do not assume local testing validates them.

### Secret Injection Patterns

```typescript
// Configuration module with validation
import { z } from 'zod';

const configSchema = z.object({
  // Database
  database: z.object({
    url: z.string().url(),
    pool: z.number().min(1).max(20).default(5),
    timeout: z.number().default(30000),
  }),

  // Redis
  redis: z.object({
    url: z.string().url(),
    db: z.number().min(0).max(15).default(0),
  }),

  // JWT
  jwt: z.object({
    secret: z.string().min(32),  // Enforce minimum length
    expiresIn: z.string().default('15m'),
    algorithm: z.enum(['HS256', 'RS256']).default('HS256'),
  }),

  // Application
  app: z.object({
    env: z.enum(['development', 'staging', 'production']),
    logLevel: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
    rateLimitPerMinute: z.number().min(1).default(100),
  }),
});

// Parse and validate at startup
const rawConfig = {
  database: {
    url: process.env.DATABASE_URL,
    pool: process.env.DB_POOL_SIZE ? parseInt(process.env.DB_POOL_SIZE) : 5,
  },
  redis: {
    url: process.env.REDIS_URL,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRY,
  },
  app: {
    env: process.env.NODE_ENV,
    logLevel: process.env.LOG_LEVEL,
    rateLimitPerMinute: parseInt(process.env.RATE_LIMIT_PER_MINUTE || '100'),
  },
};

try {
  const config = configSchema.parse(rawConfig);
  export default config;
} catch (error) {
  console.error('Configuration validation failed:', error.errors);
  process.exit(1);
}
```

## Secret Management Patterns

### Environment File with File Permissions

```bash
# Server environment file
# /etc/{app_name}.env (for systemd EnvironmentFile=)
DATABASE_URL=postgresql://user:pass@localhost/db
REDIS_URL=redis://localhost:6379
JWT_SECRET=base64-encoded-32-char-secret
API_KEY=sk_live_xxxxxxxxxxxxx

# CRITICAL: Permissions
chmod 600 /etc/{app_name}.env      # Owner read/write only
chown {deploy_user}:{deploy_user} /etc/{app_name}.env

# Verify (should show: -rw------- 1)
ls -l /etc/{app_name}.env
stat /etc/{app_name}.env
```

### Kubernetes Secrets

```yaml
# Don't do this - secrets in YAML files!
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
      - name: api
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: jwt-secret
---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
stringData:
  database-url: postgresql://user:pass@postgres:5432/db
  jwt-secret: base64-encoded-secret
```

### Secret Injection via Secret Manager

```typescript
// AWS Secrets Manager example
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

async function loadSecrets() {
  const client = new SecretsManagerClient({ region: "us-east-1" });

  const command = new GetSecretValueCommand({
    SecretId: `${process.env.NODE_ENV}/api/secrets`
  });

  const response = await client.send(command);
  const secrets = JSON.parse(response.SecretString);

  // Inject into process.env
  Object.assign(process.env, secrets);
}

// Call at startup
await loadSecrets();
```

## Environment Parity

### Concept

```
Development        Staging          Production
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Code: v1.2.3 │  │ Code: v1.2.3 │  │ Code: v1.2.3 │
│ Node: 20.x   │  │ Node: 20.x   │  │ Node: 20.x   │
│ Deps: locked │  │ Deps: locked │  │ Deps: locked │
│ Config: same │  │ Config: same │  │ Config: same │
│ DB schema: ✓ │  │ DB schema: ✓ │  │ DB schema: ✓ │
└──────────────┘  └──────────────┘  └──────────────┘
        │                  │                │
        └──────────────────┴────────────────┘
     Only difference: Secrets & Scale
```

### Testing Environment Parity

```bash
# Test on staging should match production behavior
# .github/workflows/test-parity.yml

name: Environment Parity Check

on:
  push:
    branches: [main]

jobs:
  test-parity:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [development, staging, production]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Load Config - ${{ matrix.environment }}
        run: |
          cp .env.${{ matrix.environment }} .env

      - name: Validate Configuration
        run: npm run validate:config

      - name: Check Dependency Versions
        run: npm list --depth=0

      - name: Test Application Startup
        run: timeout 30 npm start &
        env:
          NODE_ENV: ${{ matrix.environment }}
```

## Environment-Specific Configuration

### Configuration by Environment

```typescript
// config/environment.ts
const environments = {
  development: {
    logLevel: 'debug',
    cache: { ttl: 0 },  // Disable caching
    database: { ssl: false },
    api: { timeout: 60000 },
    features: {
      debugMode: true,
      mockExternalAPIs: true,
    },
  },

  staging: {
    logLevel: 'info',
    cache: { ttl: 3600 },  // 1 hour
    database: { ssl: true },
    api: { timeout: 30000 },
    features: {
      debugMode: false,
      mockExternalAPIs: false,
    },
  },

  production: {
    logLevel: 'warn',
    cache: { ttl: 86400 },  // 24 hours
    database: { ssl: true, poolSize: 20 },
    api: { timeout: 15000 },
    features: {
      debugMode: false,
      mockExternalAPIs: false,
    },
  },
};

export const getConfig = () => {
  const env = process.env.NODE_ENV || 'development';
  return environments[env] || environments.development;
};
```

## Environment Variable Validation

### Startup Validation

```typescript
// lib/config.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_EXPIRY: z.string().default('15m'),
  API_RATE_LIMIT: z.coerce.number().min(1).default(100),
  NEXT_PUBLIC_API_URL: z.string().url(),
});

// Parse at module load time (app startup)
const config = envSchema.parse(process.env);

export default config;

// If validation fails, error is thrown and app cannot start
// Error output:
// Error: Configuration validation failed
// - NODE_ENV: Expected 'development' | 'staging' | 'production'
// - JWT_SECRET: String must contain at least 32 character(s)
```

## Secret Rotation

### Rotation Pattern

```bash
# Automated secret rotation monthly
0 0 1 * * /usr/local/bin/rotate-secrets.sh

# Script: rotate-secrets.sh
#!/bin/bash
set -euo pipefail

ENV_FILE="/etc/{app_name}.env"
BACKUP_DIR="/var/backups/secrets"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup current secrets
cp $ENV_FILE $BACKUP_DIR/env_${DATE}.backup
chmod 600 $BACKUP_DIR/env_${DATE}.backup

# Generate new secrets
NEW_JWT_SECRET=$(openssl rand -base64 32)
NEW_API_KEY=$(openssl rand -hex 32)

# Update environment file
sed -i "s/JWT_SECRET=.*/JWT_SECRET=${NEW_JWT_SECRET}/" $ENV_FILE
sed -i "s/API_KEY=.*/API_KEY=${NEW_API_KEY}/" $ENV_FILE

# Restart application
systemctl restart {app_name}

# Verify service restarted
sleep 5
if ! curl -f http://localhost:3000/health > /dev/null; then
  # Restore from backup
  cp $BACKUP_DIR/env_${DATE}.backup $ENV_FILE
  systemctl restart {app_name}
  echo "ERROR: Rotation failed, restored from backup" | \
    mail -s "Secret rotation failed" ops@example.com
  exit 1
fi

# Log rotation
logger -t secret-rotation "Rotated JWT_SECRET and API_KEY"
echo "Secrets rotated successfully at $(date)" >> /var/log/secret-rotation.log
```

## Environment Documentation Requirements

### Required Documentation

```markdown
# Environment Configuration

## Environments

### Development
- **Purpose**: Local development
- **Node Version**: 20.x
- **Database**: Local PostgreSQL
- **Database Name**: myapp_dev
- **Caching**: Disabled
- **Secrets**: Use .env.local

**Setup**:
```
npm install
# Create .env.local with local credentials (see project README)
npm run dev
```

### Staging
- **Purpose**: Pre-production testing
- **Node Version**: 20.x
- **Database**: Shared staging database
- **Database Name**: myapp_staging
- **Caching**: Enabled (1 hour TTL)
- **Backup**: Daily

**Access**: https://staging-api.example.com

### Production
- **Purpose**: Live service
- **Node Version**: 20.x
- **Database**: Production database
- **Database Name**: myapp_prod
- **Caching**: Enabled (24 hour TTL)
- **Backup**: Hourly

**Access**: https://api.example.com

## Configuration Variables

| Variable | Type | Required | Example | Notes |
|----------|------|----------|---------|-------|
| NODE_ENV | string | ✓ | production | One of: development, staging, production |
| DATABASE_URL | string | ✓ | postgresql://... | Full connection string |
| REDIS_URL | string | ✓ | redis://... | Full connection string |
| JWT_SECRET | string | ✓ | (32+ chars) | Base64-encoded secret |
| LOG_LEVEL | string | | warn | debug, info, warn, error |
| API_RATE_LIMIT | number | | 100 | Requests per minute |

## Secrets Management

### Development
- Store in .env.local (never commit)
- Can use test/dummy secrets

### Staging
- Store in /etc/myapp.env on server
- Use real (but staging) API keys
- Rotate monthly

### Production
- Store in /etc/myapp.env on server
- Use real production API keys
- Rotate monthly
- Access restricted to ops team
```

## Checklist

### Configuration

- [ ] Environment names follow standards (development, staging, production)
- [ ] `.env` and `.env*.local` added to .gitignore
- [ ] `.env.production` committed (public vars only, no secrets)
- [ ] Configuration validation implemented
- [ ] App fails fast on invalid config

### Secrets

- [ ] No hardcoded secrets in code
- [ ] No secrets in committed `.env.production`
- [ ] Environment file permissions set to 600
- [ ] Secret injection working for all environments
- [ ] Secrets not logged or exposed

### Environment Parity

- [ ] Configuration identical across environments (except secrets, scale)
- [ ] Node/runtime versions match
- [ ] Dependency lockfiles committed
- [ ] Database schemas synchronized
- [ ] Tested on staging before production deploy

### Documentation

- [ ] Environment setup documented
- [ ] Configuration variables documented
- [ ] Secret rotation process documented
- [ ] Troubleshooting guide created
- [ ] Access procedures documented

### Validation

- [ ] Configuration validated at startup
- [ ] Env variables type-checked
- [ ] Required variables enforced
- [ ] Invalid config causes startup failure
- [ ] Error messages clear and actionable

## References

- [The Twelve-Factor App - Config](https://12factor.net/config)
- [Node.js Environment Variables Best Practices](https://nodejs.org/en/docs/guides/nodejs-security/)
- [Zod Schema Validation](https://zod.dev/)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
