# Data Migration Standards

> Authoritative data migration standards for safe, zero-downtime data transformation across all projects.

## Purpose

Define how to plan, execute, validate, and roll back data migrations — operations that transform existing data in a running system without causing downtime or data loss. This is distinct from schema migrations (covered in Database Standards), which change table structure.

## Core Principles

1. **Reversibility** — Every migration has a documented rollback procedure tested before execution
2. **Idempotency** — Running a migration twice produces the same result as running it once
3. **Zero downtime** — Migrations execute while the application serves traffic
4. **Validation at every step** — Pre-migration checks, progress monitoring, post-migration verification
5. **Small batches** — Process data in chunks to control resource usage and enable pause/resume
6. **Dry run first** — Every migration runs in dry-run mode before actual execution

## Migration Types

| Type | Description | Example | Risk Level |
| ---- | ----------- | ------- | ---------- |
| **Backfill** | Populate new columns with computed values | Fill `full_name` from `first_name` + `last_name` | Low-Medium |
| **Transform** | Change data format or structure | Convert `price` from float to integer (pence) | Medium |
| **Platform** | Move data between systems | Legacy DB → new service, CSV → database | High |
| **Cleanup** | Remove orphaned or stale data | Delete soft-deleted records older than 2 years | Medium |
| **Merge/Split** | Combine or separate data | Merge duplicate user accounts | High |

## Planning

### Pre-Migration Checklist

Before writing any migration code:

- [ ] **Impact assessment**: How many rows? Which tables? Estimated duration?
- [ ] **Rollback plan**: How do you undo this migration?
- [ ] **Dependencies**: Are other services reading/writing this data during migration?
- [ ] **Coordination**: Do application code changes need to deploy before/after?
- [ ] **Backup**: Take a snapshot before starting
- [ ] **Window**: When is lowest traffic? (Even for zero-downtime, less traffic = less risk)
- [ ] **Monitoring**: What metrics will you watch during execution?

### Runbook Template

```markdown
# Data Migration Runbook: [Name]

## Summary
- **What**: [Describe the data transformation]
- **Why**: [Business/technical reason]
- **Rows affected**: ~[estimate]
- **Estimated duration**: [time]
- **Risk level**: Low / Medium / High

## Pre-Conditions
- [ ] Application version >= X.Y.Z deployed
- [ ] Database backup taken at [timestamp]
- [ ] Feature flag `migration_xyz` is OFF
- [ ] Monitoring dashboard open: [link]

## Execution Steps
1. Run dry-run: `pnpm migrate:xyz --dry-run`
2. Verify dry-run output matches expectations
3. Run migration: `pnpm migrate:xyz`
4. Monitor: [what to watch]
5. Validate: [verification queries]

## Rollback Procedure
1. [Step-by-step rollback instructions]
2. [Verification that rollback succeeded]

## Post-Migration
- [ ] Verify row counts match expectations
- [ ] Verify data integrity checks pass
- [ ] Remove migration code (or mark as completed)
- [ ] Update documentation
```

## Implementation Patterns

### Batched Processing

Never process all rows in a single transaction. Use cursor-based batching:

```typescript
// TypeScript (Prisma)
async function migrateInBatches(options: {
  batchSize: number;
  dryRun: boolean;
  onProgress: (processed: number, total: number) => void;
}) {
  const { batchSize, dryRun, onProgress } = options;
  const total = await prisma.user.count({ where: { fullName: null } });
  let processed = 0;
  let cursor: string | undefined;

  while (true) {
    const batch = await prisma.user.findMany({
      where: { fullName: null },
      take: batchSize,
      ...(cursor ? { skip: 1, cursor: { id: cursor } } : {}),
      orderBy: { id: 'asc' },
    });

    if (batch.length === 0) break;

    if (!dryRun) {
      await prisma.$transaction(
        batch.map(user =>
          prisma.user.update({
            where: { id: user.id },
            data: { fullName: `${user.firstName} ${user.lastName}`.trim() },
          })
        )
      );
    }

    cursor = batch[batch.length - 1].id;
    processed += batch.length;
    onProgress(processed, total);
  }

  return { total, processed, dryRun };
}
```

### .NET Batched Processing

```csharp
public async Task<MigrationResult> MigrateAsync(
    int batchSize = 1000,
    bool dryRun = false,
    CancellationToken cancellationToken = default)
{
    var total = await _context.Users.CountAsync(u => u.FullName == null, cancellationToken);
    var processed = 0;
    string? lastId = null;

    while (true)
    {
        var query = _context.Users
            .Where(u => u.FullName == null)
            .OrderBy(u => u.Id)
            .Take(batchSize);

        if (lastId is not null)
            query = query.Where(u => u.Id.CompareTo(lastId) > 0);

        var batch = await query.ToListAsync(cancellationToken);
        if (batch.Count == 0) break;

        if (!dryRun)
        {
            foreach (var user in batch)
            {
                user.FullName = $"{user.FirstName} {user.LastName}".Trim();
            }
            await _context.SaveChangesAsync(cancellationToken);
        }

        lastId = batch[^1].Id;
        processed += batch.Count;
        _logger.LogInformation("Progress: {Processed}/{Total}", processed, total);
    }

    return new MigrationResult(total, processed, dryRun);
}
```

### Python Batched Processing

```python
async def migrate_in_batches(
    session: AsyncSession,
    batch_size: int = 1000,
    dry_run: bool = False,
) -> dict:
    total = await session.scalar(
        select(func.count()).where(User.full_name.is_(None))
    )
    processed = 0
    last_id = None

    while True:
        query = (
            select(User)
            .where(User.full_name.is_(None))
            .order_by(User.id)
            .limit(batch_size)
        )
        if last_id:
            query = query.where(User.id > last_id)

        result = await session.execute(query)
        batch = result.scalars().all()

        if not batch:
            break

        if not dry_run:
            for user in batch:
                user.full_name = f"{user.first_name} {user.last_name}".strip()
            await session.commit()

        last_id = batch[-1].id
        processed += len(batch)
        logger.info(f"Progress: {processed}/{total}")

    return {"total": total, "processed": processed, "dry_run": dry_run}
```

## Idempotency

Migrations must be safe to run multiple times:

```typescript
// ✅ Idempotent: Only updates rows that haven't been migrated
await prisma.user.updateMany({
  where: {
    fullName: null,  // Only rows not yet migrated
    firstName: { not: null },
  },
  data: {
    fullName: prisma.raw(`CONCAT(first_name, ' ', last_name)`),
  },
});

// ❌ Not idempotent: Running twice doubles the price
await prisma.product.updateMany({
  data: { price: { multiply: 100 } }, // Convert dollars to cents
});

// ✅ Idempotent version: Check if already converted
await prisma.product.updateMany({
  where: { priceFormat: 'dollars' }, // Guard condition
  data: {
    price: { multiply: 100 },
    priceFormat: 'cents',
  },
});
```

## Dual-Write Pattern

For high-risk migrations where you need to verify correctness before committing:

```
Phase 1: Dual Write
┌─────────┐     ┌─────────────┐     ┌──────────┐
│  App     │────▶│  Old Column  │     │ New Column│
│  Write   │────▶│  (primary)   │────▶│ (shadow)  │
└─────────┘     └─────────────┘     └──────────┘

Phase 2: Backfill (migrate historical data)

Phase 3: Verify (compare old vs new for consistency)

Phase 4: Switch Read
┌─────────┐     ┌─────────────┐     ┌──────────┐
│  App     │────▶│  Old Column  │     │ New Column│
│  Read    │     │  (shadow)    │◀────│ (primary) │
└─────────┘     └─────────────┘     └──────────┘

Phase 5: Stop writing to old column
Phase 6: Drop old column (separate migration)
```

## Validation

### Pre-Migration Validation

```typescript
async function preMigrationChecks(): Promise<void> {
  // Count rows to be migrated
  const count = await prisma.user.count({ where: { fullName: null } });
  console.log(`Rows to migrate: ${count}`);

  // Sample check — verify transformation logic on a few rows
  const sample = await prisma.user.findMany({
    where: { fullName: null },
    take: 10,
  });

  for (const user of sample) {
    const expected = `${user.firstName} ${user.lastName}`.trim();
    console.log(`${user.id}: "${user.firstName}" + "${user.lastName}" → "${expected}"`);
  }

  // Check for edge cases
  const nullNames = await prisma.user.count({
    where: { fullName: null, firstName: null },
  });
  if (nullNames > 0) {
    console.warn(`${nullNames} users have NULL firstName — will produce empty fullName`);
  }
}
```

### Post-Migration Validation

```sql
-- Row count matches expectation
SELECT COUNT(*) FROM users WHERE full_name IS NOT NULL;

-- No data loss
SELECT COUNT(*) FROM users WHERE full_name IS NULL AND first_name IS NOT NULL;
-- Expected: 0

-- Spot check random samples
SELECT id, first_name, last_name, full_name
FROM users
ORDER BY RANDOM()
LIMIT 20;

-- Verify no truncation or corruption
SELECT id, full_name FROM users
WHERE LENGTH(full_name) > 200
OR full_name LIKE '%null%'
OR full_name = '';
```

## Monitoring During Migration

### Key Metrics to Watch

| Metric | Normal | Alert |
| ------ | ------ | ----- |
| Database CPU | < 60% | > 80% |
| Active connections | < 80% pool | > 90% pool |
| Replication lag | < 1s | > 5s |
| API response time (p99) | < 500ms | > 1s |
| Error rate | < 0.1% | > 1% |
| Migration progress | Increasing | Stalled for > 5 min |

### Pause/Resume

Design migrations to be pausable:

```typescript
// Check for stop signal between batches
const STOP_FILE = '/tmp/migration-stop';

while (true) {
  if (fs.existsSync(STOP_FILE)) {
    console.log('Stop signal received. Pausing migration.');
    console.log(`Last processed ID: ${cursor}`);
    break;
  }

  // Process next batch...
}

// Resume: pass --cursor=<last-id> to continue from where it stopped
```

## Checklist

### Planning

- [ ] Impact assessment completed (row count, duration estimate, affected services)
- [ ] Runbook written with rollback procedure
- [ ] Database backup taken
- [ ] Dry run executed successfully
- [ ] Migration tested on staging with production-size data

### Implementation

- [ ] Migration is idempotent (safe to run multiple times)
- [ ] Processes in batches (not single transaction)
- [ ] Uses cursor-based pagination (not OFFSET)
- [ ] Includes progress logging
- [ ] Supports dry-run mode
- [ ] Supports pause/resume

### Execution

- [ ] Monitoring dashboard open during execution
- [ ] Database metrics within normal range
- [ ] API response times unaffected
- [ ] Pre-migration validation passed
- [ ] Post-migration validation passed (row counts, spot checks, integrity)

### Cleanup

- [ ] Migration code removed or marked as completed
- [ ] Old columns/tables dropped (in a separate, later migration)
- [ ] Runbook archived
- [ ] Documentation updated

## References

- [Stripe — Online Migrations at Scale](https://stripe.com/blog/online-migrations)
- [GitHub — Moving Millions of Rows](https://github.blog/engineering/infrastructure/mysql-online-schema-migrations/)
- [Prisma — Data Migrations](https://www.prisma.io/docs/guides/database/seed-database)
- [Django — Data Migrations](https://docs.djangoproject.com/en/5.0/topics/migrations/#data-migrations)
