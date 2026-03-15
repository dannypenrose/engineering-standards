# Data Governance & Privacy

> Authoritative data governance standards for GDPR/CCPA compliance, PII handling, and data retention across all projects.

## Purpose

Establish policies and practices for handling personal data responsibly, ensuring regulatory compliance, and maintaining user trust through transparent data practices.

## Core Principles

1. **Privacy by design** - Build privacy into systems from the start
2. **Data minimization** - Collect only what's necessary
3. **Purpose limitation** - Use data only for stated purposes
4. **Transparency** - Be clear about data practices
5. **User control** - Enable users to manage their data
6. **Security first** - Protect data at rest and in transit

## Data Classification

### Classification Levels

| Level | Description | Examples | Handling Requirements |
| ----- | ----------- | -------- | --------------------- |
| **Public** | Non-sensitive, publicly available | Marketing content, public APIs | No restrictions |
| **Internal** | Business information, not personal | Internal docs, configs | Access control |
| **Confidential** | Sensitive business data | Financial data, contracts | Encryption, audit logs |
| **PII** | Personally Identifiable Information | Email, name, address | Encryption, consent, retention limits |
| **Sensitive PII** | High-risk personal data | SSN, health data, biometrics | Encryption, strict access, compliance |

### PII Categories

```typescript
// Data classification types
interface DataClassification {
  level: 'public' | 'internal' | 'confidential' | 'pii' | 'sensitive_pii';
  retention: RetentionPolicy;
  encryption: EncryptionRequirement;
  consent: ConsentRequirement;
}

// PII field classification
const PII_FIELDS: Record<string, DataClassification> = {
  // Standard PII
  email: {
    level: 'pii',
    retention: { duration: '3y', afterDeletion: '30d' },
    encryption: 'at_rest',
    consent: 'explicit',
  },
  name: {
    level: 'pii',
    retention: { duration: '3y', afterDeletion: '30d' },
    encryption: 'at_rest',
    consent: 'explicit',
  },
  phone: {
    level: 'pii',
    retention: { duration: '3y', afterDeletion: '30d' },
    encryption: 'at_rest',
    consent: 'explicit',
  },
  address: {
    level: 'pii',
    retention: { duration: '3y', afterDeletion: '30d' },
    encryption: 'at_rest',
    consent: 'explicit',
  },
  ipAddress: {
    level: 'pii',
    retention: { duration: '90d', afterDeletion: '0d' },
    encryption: 'at_rest',
    consent: 'legitimate_interest',
  },

  // Sensitive PII
  ssn: {
    level: 'sensitive_pii',
    retention: { duration: '7y', afterDeletion: '0d' }, // Legal requirement
    encryption: 'field_level',
    consent: 'explicit',
  },
  dateOfBirth: {
    level: 'sensitive_pii',
    retention: { duration: '3y', afterDeletion: '30d' },
    encryption: 'at_rest',
    consent: 'explicit',
  },
  healthData: {
    level: 'sensitive_pii',
    retention: { duration: '6y', afterDeletion: '0d' },
    encryption: 'field_level',
    consent: 'explicit',
  },
  financialData: {
    level: 'sensitive_pii',
    retention: { duration: '7y', afterDeletion: '0d' },
    encryption: 'field_level',
    consent: 'explicit',
  },
};
```

## Consent Management

### Consent Types

| Type | Description | Use Case | Withdrawal |
| ---- | ----------- | -------- | ---------- |
| **Explicit** | Active opt-in required | Marketing, profiling | Must be easy |
| **Implicit** | Inferred from action | Service delivery | N/A |
| **Legitimate Interest** | Business necessity | Fraud prevention, security | Must be possible |

### Consent Implementation

```typescript
// Consent record schema
interface ConsentRecord {
  userId: string;
  consentType: string;
  granted: boolean;
  timestamp: Date;
  ipAddress: string;
  userAgent: string;
  version: string; // Policy version
  source: 'web' | 'mobile' | 'api';
  withdrawnAt?: Date;
}

// Consent categories
const CONSENT_CATEGORIES = {
  essential: {
    name: 'Essential',
    description: 'Required for the service to function',
    required: true,
  },
  analytics: {
    name: 'Analytics',
    description: 'Help us understand how you use our service',
    required: false,
    default: false,
  },
  marketing: {
    name: 'Marketing',
    description: 'Personalized offers and communications',
    required: false,
    default: false,
  },
  thirdParty: {
    name: 'Third-party sharing',
    description: 'Share data with partners',
    required: false,
    default: false,
  },
};
```

### Consent UI Pattern

```typescript
// React consent banner component
function ConsentBanner() {
  const [consents, setConsents] = useState<Record<string, boolean>>({
    essential: true, // Always true
    analytics: false,
    marketing: false,
    thirdParty: false,
  });

  const handleAcceptAll = () => {
    setConsents({
      essential: true,
      analytics: true,
      marketing: true,
      thirdParty: true,
    });
    saveConsents(consents);
  };

  const handleRejectNonEssential = () => {
    setConsents({
      essential: true,
      analytics: false,
      marketing: false,
      thirdParty: false,
    });
    saveConsents(consents);
  };

  return (
    <div role="dialog" aria-label="Cookie consent">
      <h2>We value your privacy</h2>
      <p>We use cookies to enhance your experience...</p>

      <ConsentOptions consents={consents} onChange={setConsents} />

      <div className="actions">
        <button onClick={handleRejectNonEssential}>
          Reject non-essential
        </button>
        <button onClick={() => saveConsents(consents)}>
          Save preferences
        </button>
        <button onClick={handleAcceptAll}>
          Accept all
        </button>
      </div>

      <a href="/privacy-policy">Privacy Policy</a>
    </div>
  );
}
```

## Data Retention

### Retention Policies

| Data Type | Retention Period | Legal Basis | Disposal Method |
| --------- | ---------------- | ----------- | --------------- |
| User accounts | Until deletion request | Consent | Secure deletion |
| Transaction records | 7 years | Legal requirement | Archive then delete |
| Server logs | 90 days | Legitimate interest | Automatic purge |
| Analytics data | 26 months | Consent | Aggregation/deletion |
| Backup data | 30 days after primary deletion | Operational | Secure deletion |
| Marketing preferences | Until withdrawal | Consent | Immediate deletion |

### Retention Implementation

```typescript
// retention-policies.ts
interface RetentionPolicy {
  dataType: string;
  retentionDays: number;
  legalBasis: string;
  disposalMethod: 'delete' | 'anonymize' | 'archive';
  archiveDays?: number; // For archive-then-delete
}

const RETENTION_POLICIES: RetentionPolicy[] = [
  {
    dataType: 'user_account',
    retentionDays: -1, // Until deletion request
    legalBasis: 'consent',
    disposalMethod: 'delete',
  },
  {
    dataType: 'transaction_record',
    retentionDays: 2555, // 7 years
    legalBasis: 'legal_requirement',
    disposalMethod: 'archive',
    archiveDays: 365, // Archive for 1 more year then delete
  },
  {
    dataType: 'server_logs',
    retentionDays: 90,
    legalBasis: 'legitimate_interest',
    disposalMethod: 'delete',
  },
  {
    dataType: 'analytics_events',
    retentionDays: 791, // 26 months
    legalBasis: 'consent',
    disposalMethod: 'anonymize',
  },
];

// Scheduled cleanup job
async function enforceRetentionPolicies() {
  for (const policy of RETENTION_POLICIES) {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - policy.retentionDays);

    switch (policy.disposalMethod) {
      case 'delete':
        await deleteDataOlderThan(policy.dataType, cutoffDate);
        break;
      case 'anonymize':
        await anonymizeDataOlderThan(policy.dataType, cutoffDate);
        break;
      case 'archive':
        await archiveDataOlderThan(policy.dataType, cutoffDate);
        break;
    }

    logger.info('Retention policy enforced', {
      dataType: policy.dataType,
      cutoffDate,
      action: policy.disposalMethod,
    });
  }
}
```

## User Rights (GDPR/CCPA)

### Required Capabilities

| Right | GDPR | CCPA | Implementation |
| ----- | ---- | ---- | -------------- |
| **Access** | Art. 15 | § 1798.100 | Data export API |
| **Rectification** | Art. 16 | N/A | Edit profile |
| **Erasure** | Art. 17 | § 1798.105 | Delete account |
| **Portability** | Art. 20 | N/A | Data export |
| **Opt-out of sale** | N/A | § 1798.120 | Consent management |
| **Non-discrimination** | N/A | § 1798.125 | Policy compliance |

### Data Subject Request (DSR) API

```typescript
// dsr.controller.ts
@Controller('api/v1/privacy')
export class PrivacyController {
  constructor(private readonly privacyService: PrivacyService) {}

  @Get('export')
  @UseGuards(AuthGuard)
  async exportData(@User() user: AuthUser): Promise<DataExport> {
    // Collect all user data
    const data = await this.privacyService.collectUserData(user.id);

    // Log the request
    await this.privacyService.logDSR({
      userId: user.id,
      type: 'export',
      timestamp: new Date(),
    });

    return {
      exportDate: new Date().toISOString(),
      data,
      format: 'json',
    };
  }

  @Post('delete')
  @UseGuards(AuthGuard)
  async requestDeletion(
    @User() user: AuthUser,
    @Body() dto: DeletionRequestDto,
  ): Promise<DeletionResponse> {
    // Verify identity (may require additional auth)
    await this.privacyService.verifyIdentity(user.id, dto.verificationCode);

    // Schedule deletion (30 day grace period for GDPR)
    const deletionDate = new Date();
    deletionDate.setDate(deletionDate.getDate() + 30);

    await this.privacyService.scheduleDeletion({
      userId: user.id,
      scheduledDate: deletionDate,
      reason: dto.reason,
    });

    return {
      message: 'Deletion scheduled',
      scheduledDate: deletionDate.toISOString(),
      cancellationDeadline: deletionDate.toISOString(),
    };
  }

  @Post('delete/cancel')
  @UseGuards(AuthGuard)
  async cancelDeletion(@User() user: AuthUser): Promise<void> {
    await this.privacyService.cancelDeletion(user.id);
  }

  @Get('consent')
  @UseGuards(AuthGuard)
  async getConsentStatus(@User() user: AuthUser): Promise<ConsentStatus> {
    return this.privacyService.getConsentStatus(user.id);
  }

  @Put('consent')
  @UseGuards(AuthGuard)
  async updateConsent(
    @User() user: AuthUser,
    @Body() dto: UpdateConsentDto,
  ): Promise<ConsentStatus> {
    return this.privacyService.updateConsent(user.id, dto);
  }
}
```

### Data Export Format

```typescript
// Standardized data export format
interface DataExport {
  exportDate: string;
  userId: string;
  format: 'json';
  data: {
    profile: {
      email: string;
      name: string;
      createdAt: string;
      // ... other profile fields
    };
    activity: {
      logins: LoginRecord[];
      actions: ActionRecord[];
    };
    preferences: {
      notifications: NotificationPreferences;
      privacy: PrivacyPreferences;
    };
    thirdPartySharing: {
      partners: string[];
      sharedData: string[];
    };
    consents: ConsentRecord[];
  };
}
```

## Data Anonymization

### Anonymization Techniques

| Technique | Description | Use Case |
| --------- | ----------- | -------- |
| **Pseudonymization** | Replace identifiers with tokens | Analytics, testing |
| **Generalization** | Reduce precision | Age ranges, regions |
| **Suppression** | Remove field entirely | Rare values |
| **Noise addition** | Add random variation | Statistics |
| **K-anonymity** | Ensure k identical records | Public datasets |

### Implementation

```typescript
// anonymization.service.ts
@Injectable()
export class AnonymizationService {
  // Pseudonymization - reversible with key
  pseudonymize(userId: string): string {
    return crypto
      .createHmac('sha256', this.config.pseudonymizationKey)
      .update(userId)
      .digest('hex');
  }

  // Generalization - irreversible
  generalizeAge(age: number): string {
    if (age < 18) return 'under_18';
    if (age < 25) return '18-24';
    if (age < 35) return '25-34';
    if (age < 45) return '35-44';
    if (age < 55) return '45-54';
    if (age < 65) return '55-64';
    return '65+';
  }

  generalizeLocation(city: string, country: string): string {
    // Return only country for privacy
    return country;
  }

  // Email anonymization
  anonymizeEmail(email: string): string {
    const [local, domain] = email.split('@');
    const anonymizedLocal = local[0] + '***';
    return `${anonymizedLocal}@${domain}`;
  }

  // Full record anonymization for analytics
  anonymizeRecord(record: UserRecord): AnonymizedRecord {
    return {
      id: this.pseudonymize(record.id),
      ageGroup: this.generalizeAge(record.age),
      country: record.country,
      signupMonth: record.createdAt.toISOString().slice(0, 7), // YYYY-MM
      // Explicitly exclude: email, name, address, phone
    };
  }
}
```

## Logging & Audit

### What to Log

```typescript
// Privacy-related events to audit
const PRIVACY_AUDIT_EVENTS = [
  'user.consent.granted',
  'user.consent.withdrawn',
  'user.data.exported',
  'user.data.deleted',
  'user.data.accessed', // By support staff
  'user.data.modified', // By support staff
  'dsr.request.created',
  'dsr.request.completed',
  'data.shared.thirdparty',
  'data.breach.detected',
];

// Audit log structure
interface PrivacyAuditLog {
  timestamp: Date;
  eventType: string;
  userId: string;
  actorId: string; // Who performed the action
  actorType: 'user' | 'system' | 'admin';
  details: Record<string, unknown>;
  ipAddress: string;
  userAgent: string;
}
```

### What NOT to Log

```typescript
// Never log these
const NEVER_LOG = [
  'password',
  'passwordHash',
  'token',
  'refreshToken',
  'apiKey',
  'ssn',
  'creditCardNumber',
  'cvv',
  'bankAccountNumber',
];

// Sanitize before logging
function sanitizeForLogging(data: Record<string, unknown>): Record<string, unknown> {
  const sanitized = { ...data };

  for (const key of Object.keys(sanitized)) {
    if (NEVER_LOG.some(forbidden => key.toLowerCase().includes(forbidden.toLowerCase()))) {
      sanitized[key] = '[REDACTED]';
    }
  }

  return sanitized;
}
```

## Third-Party Data Sharing

### Data Processing Agreements

```markdown
## Required DPA Elements

Before sharing data with third parties:

- [ ] Data Processing Agreement (DPA) signed
- [ ] Sub-processor list obtained
- [ ] Security measures documented
- [ ] Breach notification procedures agreed
- [ ] Data deletion procedures agreed
- [ ] Audit rights established
```

### Sharing Controls

```typescript
// Third-party data sharing configuration
interface DataSharingConfig {
  partnerId: string;
  partnerName: string;
  allowedFields: string[];
  purpose: string;
  legalBasis: 'consent' | 'contract' | 'legitimate_interest';
  retentionDays: number;
  dpaSignedDate: string;
}

const DATA_SHARING_CONFIG: DataSharingConfig[] = [
  {
    partnerId: 'analytics_provider',
    partnerName: 'Analytics Inc',
    allowedFields: ['anonymized_user_id', 'country', 'page_views'],
    purpose: 'Usage analytics',
    legalBasis: 'consent',
    retentionDays: 365,
    dpaSignedDate: '2024-01-01',
  },
];

// Check before sharing
function canShareData(
  partnerId: string,
  userId: string,
  fields: string[],
): boolean {
  const config = DATA_SHARING_CONFIG.find(c => c.partnerId === partnerId);
  if (!config) return false;

  // Check all requested fields are allowed
  const allFieldsAllowed = fields.every(f => config.allowedFields.includes(f));
  if (!allFieldsAllowed) return false;

  // Check user consent if required
  if (config.legalBasis === 'consent') {
    const consent = getUserConsent(userId, `share_${partnerId}`);
    if (!consent) return false;
  }

  return true;
}
```

## Breach Response

### Breach Classification

| Severity | Examples | Notification |
| -------- | -------- | ------------ |
| **Critical** | Mass PII exposure, credentials leaked | 24 hours to regulators, affected users |
| **High** | Limited PII exposure | 72 hours to regulators |
| **Medium** | Internal data exposed | Document, assess notification need |
| **Low** | Non-sensitive data | Document internally |

### Breach Response Procedure

```markdown
## Data Breach Response Checklist

### Immediate (0-4 hours)
- [ ] Contain the breach (isolate systems if needed)
- [ ] Assess scope and severity
- [ ] Engage incident response team
- [ ] Preserve evidence

### Short-term (4-24 hours)
- [ ] Determine data types affected
- [ ] Identify affected users
- [ ] Assess regulatory notification requirements
- [ ] Prepare initial notification draft

### Notification (24-72 hours for GDPR)
- [ ] Notify supervisory authority (if required)
- [ ] Notify affected individuals (if required)
- [ ] Document notification decisions

### Remediation (Ongoing)
- [ ] Implement fixes
- [ ] Review and update security measures
- [ ] Conduct post-incident review
- [ ] Update policies if needed
```

## Checklist

### Compliance Setup

- [ ] Data classification scheme implemented
- [ ] Consent management system deployed
- [ ] Retention policies documented and automated
- [ ] DSR handling process established
- [ ] Privacy policy published and up-to-date

### Technical Controls

- [ ] PII encrypted at rest
- [ ] PII encrypted in transit
- [ ] Access controls for PII
- [ ] Audit logging enabled
- [ ] Data export capability implemented
- [ ] Data deletion capability implemented

### Process

- [ ] DPAs in place with all processors
- [ ] Staff privacy training completed
- [ ] Breach response plan documented
- [ ] Regular privacy audits scheduled
- [ ] Cookie consent mechanism deployed

## References

- [GDPR Full Text](https://gdpr-info.eu/)
- [CCPA Text](https://oag.ca.gov/privacy/ccpa)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [ICO UK GDPR Guidance](https://ico.org.uk/for-organisations/guide-to-data-protection/guide-to-the-general-data-protection-regulation-gdpr/)
- [Google Privacy Engineering](https://developers.google.com/privacy)
- [OWASP Privacy Risks](https://owasp.org/www-project-top-10-privacy-risks/)
