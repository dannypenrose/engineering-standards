# Database Design Standards

> Authoritative database standards for consistent schema design, migration safety, and query optimization across all projects.

## Purpose

Establish consistent patterns for database design, migrations, indexing, and query optimization across all projects and database technologies.

## Core Principles

1. **Schema is code** - Version control all database changes
2. **Migrate safely** - Zero-downtime migrations as default
3. **Index intentionally** - Every index has a cost
4. **Query efficiently** - N+1 is never acceptable
5. **Plan for scale** - Design for 10x current load
6. **Backup religiously** - Test restores regularly

## Schema Design

### Naming Conventions

| Element | Convention | Example |
| ------- | ---------- | ------- |
| Tables | snake_case, plural | `users`, `order_items` |
| Columns | snake_case | `created_at`, `user_id` |
| Primary keys | `id` | `id` |
| Foreign keys | `{table}_id` | `user_id`, `order_id` |
| Indexes | `idx_{table}_{columns}` | `idx_users_email` |
| Unique constraints | `uq_{table}_{columns}` | `uq_users_email` |
| Check constraints | `chk_{table}_{description}` | `chk_orders_positive_total` |

### Required Columns

Every table should include:

```sql
-- Standard audit columns
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- ... business columns ...
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE -- For soft deletes
);

-- Trigger for updated_at
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';
```

### Primary Key Strategy

```typescript
// UUID vs Auto-increment decision matrix
const PRIMARY_KEY_STRATEGY = {
  // Use UUID when:
  uuid: {
    useCases: [
      'Distributed systems',
      'Client-generated IDs needed',
      'Security (non-guessable)',
      'Multi-tenant with data migration needs',
    ],
    implementation: 'UUID v4 or UUID v7 (time-sortable)',
  },

  // Use auto-increment when:
  autoIncrement: {
    useCases: [
      'Single database',
      'Performance critical (smaller index)',
      'Legacy system compatibility',
      'Simple reporting needs',
    ],
    implementation: 'BIGSERIAL for future-proofing',
  },
};

// Prisma schema example
model User {
  id        String   @id @default(uuid()) @db.Uuid
  email     String   @unique
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("users")
}
```

### Foreign Key Guidelines

```sql
-- Always define foreign keys with explicit actions
ALTER TABLE orders
ADD CONSTRAINT fk_orders_user
FOREIGN KEY (user_id) REFERENCES users(id)
ON DELETE RESTRICT  -- Prevent accidental cascade
ON UPDATE CASCADE;  -- Allow ID updates

-- For optional relationships
ALTER TABLE posts
ADD CONSTRAINT fk_posts_author
FOREIGN KEY (author_id) REFERENCES users(id)
ON DELETE SET NULL;

-- For required relationships that should cascade
ALTER TABLE order_items
ADD CONSTRAINT fk_order_items_order
FOREIGN KEY (order_id) REFERENCES orders(id)
ON DELETE CASCADE;
```

### Relationship Patterns

```typescript
// One-to-Many
model User {
  id    String  @id @default(uuid())
  posts Post[]
}

model Post {
  id       String @id @default(uuid())
  userId   String @map("user_id")
  user     User   @relation(fields: [userId], references: [id])

  @@index([userId])
}

// Many-to-Many (explicit join table)
model Post {
  id   String    @id @default(uuid())
  tags PostTag[]
}

model Tag {
  id    String    @id @default(uuid())
  name  String    @unique
  posts PostTag[]
}

model PostTag {
  postId String @map("post_id")
  tagId  String @map("tag_id")
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag    Tag    @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@index([tagId])
}
```

## Indexing Strategy

### Index Decision Framework

```
Should I add an index?

1. Is this column in WHERE clauses? → Consider index
2. Is this column in JOIN conditions? → Likely need index
3. Is this column in ORDER BY? → Consider index
4. What's the cardinality? → High cardinality = better index candidate
5. What's the table size? → Small tables (<1000 rows) rarely need indexes
6. What's the read/write ratio? → High reads = more indexes OK
```

### Index Types

```sql
-- B-tree (default) - equality and range queries
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Composite index - order matters!
-- Supports: (a), (a, b), (a, b, c) but NOT (b), (c), (b, c)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Partial index - index subset of rows
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- Expression index - index computed values
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- GIN index - for arrays, JSONB, full-text search
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);

-- Covering index - include non-key columns
CREATE INDEX idx_orders_user_covering ON orders(user_id)
INCLUDE (status, total);
```

### Index Anti-patterns

```sql
-- ❌ Don't: Index every column
-- ✅ Do: Index based on query patterns

-- ❌ Don't: Create redundant indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_name ON users(email, name);
-- The second index covers the first

-- ❌ Don't: Index low-cardinality columns alone
CREATE INDEX idx_users_is_active ON users(is_active);
-- Better as partial index or composite

-- ❌ Don't: Forget index maintenance
-- ✅ Do: Regularly analyze and reindex
ANALYZE users;
REINDEX INDEX CONCURRENTLY idx_users_email;
```

## Migration Standards

### Migration Principles

```markdown
## Zero-Downtime Migration Rules

1. **Never drop columns in a single migration**
   - Step 1: Stop writing to column
   - Step 2: Deploy code that doesn't read column
   - Step 3: Drop column in separate migration

2. **Add columns as nullable or with defaults**
   - New columns must have DEFAULT or be NULL
   - Never add NOT NULL without default on existing tables

3. **Create indexes concurrently**
   - Use CONCURRENTLY to avoid locking
   - Accept longer creation time for availability

4. **Test migrations on production-size data**
   - Small test DBs hide performance issues
```

### Safe Migration Patterns

```sql
-- Adding a column (safe)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Adding NOT NULL column (safe with default)
ALTER TABLE users ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';

-- Adding index (safe with CONCURRENTLY)
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);

-- Renaming column (requires app coordination)
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN full_name VARCHAR(100);
-- Step 2: Backfill data
UPDATE users SET full_name = name WHERE full_name IS NULL;
-- Step 3: Update app to write to both, read from new
-- Step 4: Stop writing to old column
-- Step 5: Drop old column
ALTER TABLE users DROP COLUMN name;

-- Changing column type (requires care)
-- Option 1: Add new column, migrate, drop old
-- Option 2: Use ALTER with USING (locks table!)
ALTER TABLE orders ALTER COLUMN total TYPE DECIMAL(12,2) USING total::DECIMAL(12,2);
```

### Prisma Migration Workflow

```bash
# Development: Generate migration from schema changes
pnpm prisma migrate dev --name add_user_phone

# Production: Apply pending migrations
pnpm prisma migrate deploy

# Emergency: Reset (NEVER in production)
pnpm prisma migrate reset
```

```typescript
// prisma/migrations/20240115120000_add_user_phone/migration.sql
-- AlterTable
ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(20);

-- CreateIndex (CONCURRENTLY not supported in Prisma, run manually if needed)
CREATE INDEX "idx_users_phone" ON "users"("phone");
```

### Entity Framework Migrations

```bash
# Generate migration
dotnet ef migrations add AddUserPhone

# Apply to database
dotnet ef database update

# Generate SQL script (for review/production)
dotnet ef migrations script
```

```csharp
// Safe migration example
public partial class AddUserPhone : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Add nullable column (safe)
        migrationBuilder.AddColumn<string>(
            name: "Phone",
            table: "Users",
            type: "varchar(20)",
            maxLength: 20,
            nullable: true);

        // Create index
        migrationBuilder.CreateIndex(
            name: "IX_Users_Phone",
            table: "Users",
            column: "Phone");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex(
            name: "IX_Users_Phone",
            table: "Users");

        migrationBuilder.DropColumn(
            name: "Phone",
            table: "Users");
    }
}
```

## Query Optimization

### N+1 Query Prevention

```typescript
// ❌ N+1 Problem
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { userId: user.id }
  });
  // This executes 1 + N queries!
}

// ✅ Eager loading with include
const users = await prisma.user.findMany({
  include: {
    posts: true
  }
});

// ✅ Or use separate query with IN clause
const users = await prisma.user.findMany();
const userIds = users.map(u => u.id);
const posts = await prisma.post.findMany({
  where: { userId: { in: userIds } }
});
```

### Query Patterns

```typescript
// Pagination - use cursor-based for large datasets
// Offset pagination (simple, but slow on large offsets)
const users = await prisma.user.findMany({
  skip: (page - 1) * pageSize,
  take: pageSize,
  orderBy: { createdAt: 'desc' }
});

// Cursor pagination (efficient for large datasets)
const users = await prisma.user.findMany({
  take: pageSize,
  cursor: lastId ? { id: lastId } : undefined,
  skip: lastId ? 1 : 0,
  orderBy: { createdAt: 'desc' }
});

// Select only needed fields
const users = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true,
    // Don't select large fields unnecessarily
  }
});

// Use raw queries for complex operations
const result = await prisma.$queryRaw`
  SELECT
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as count
  FROM orders
  WHERE created_at > ${startDate}
  GROUP BY DATE_TRUNC('day', created_at)
  ORDER BY date DESC
`;
```

### Explain Analyze

```sql
-- Always explain slow queries
EXPLAIN ANALYZE
SELECT u.*, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id
ORDER BY order_count DESC
LIMIT 10;

-- Look for:
-- - Seq Scan on large tables (need index?)
-- - High actual rows vs estimated (statistics stale?)
-- - Nested loops with many iterations (need different join?)
-- - Sort operations (can index support this?)
```

## Connection Management

### Connection Pooling

```typescript
// Prisma connection configuration
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  // Connection pool settings in URL:
  // ?connection_limit=10&pool_timeout=30
}

// For serverless (Prisma Accelerate or PgBouncer)
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  directUrl = env("DIRECT_URL") // For migrations
}
```

```typescript
// NestJS connection configuration
// app.module.ts
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT, 10),
      username: process.env.DB_USERNAME,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_DATABASE,

      // Connection pool settings
      extra: {
        max: 20, // Maximum pool size
        min: 5,  // Minimum pool size
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 5000,
      },
    }),
  ],
})
export class AppModule {}
```

### .NET Connection Configuration

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=myapp;Username=user;Password=pass;Maximum Pool Size=20;Minimum Pool Size=5;Connection Idle Lifetime=300"
  }
}

// Program.cs
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        npgsqlOptions =>
        {
            npgsqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorCodesToAdd: null);
            npgsqlOptions.CommandTimeout(30);
        }));
```

## Backup & Recovery

### Backup Strategy

| Backup Type | Frequency | Retention | Use Case |
| ----------- | --------- | --------- | -------- |
| Full backup | Daily | 30 days | Complete restore |
| Incremental | Hourly | 7 days | Point-in-time recovery |
| Transaction logs | Continuous | 7 days | Minimal data loss |
| Snapshot | Before migrations | 48 hours | Quick rollback |

### Recovery Testing

```markdown
## Monthly Recovery Test Procedure

1. **Restore to test environment**
   - Restore latest backup to isolated environment
   - Verify data integrity

2. **Point-in-time recovery test**
   - Restore to specific timestamp
   - Verify expected state

3. **Document results**
   - Time to restore
   - Any issues encountered
   - Data verification results

4. **Update runbook if needed**
```

## Multi-Tenancy Patterns

### Strategy Comparison

| Strategy | Isolation | Complexity | Cost | Use Case |
| -------- | --------- | ---------- | ---- | -------- |
| Shared database, shared schema | Low | Low | Low | SaaS MVP |
| Shared database, separate schema | Medium | Medium | Medium | SMB SaaS |
| Separate database | High | High | High | Enterprise, compliance |

### Row-Level Security (PostgreSQL)

```sql
-- Enable RLS on table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Set tenant context in application
SET app.tenant_id = 'tenant-uuid-here';

-- All queries automatically filtered
SELECT * FROM orders; -- Only returns current tenant's orders
```

```typescript
// Prisma middleware for tenant isolation
prisma.$use(async (params, next) => {
  const tenantId = getTenantId(); // From request context

  if (params.action === 'findMany' || params.action === 'findFirst') {
    params.args.where = {
      ...params.args.where,
      tenantId,
    };
  }

  if (params.action === 'create') {
    params.args.data.tenantId = tenantId;
  }

  return next(params);
});
```

## Checklist

### Schema Design

- [ ] All tables have id, created_at, updated_at
- [ ] Foreign keys defined with appropriate ON DELETE
- [ ] Naming conventions followed consistently
- [ ] Soft delete implemented where needed
- [ ] Appropriate column types chosen

### Indexing

- [ ] Primary keys indexed (automatic)
- [ ] Foreign keys indexed
- [ ] Frequently queried columns indexed
- [ ] Composite indexes match query patterns
- [ ] No redundant indexes

### Migrations

- [ ] Migrations are reversible
- [ ] Zero-downtime patterns used
- [ ] Indexes created concurrently
- [ ] Large data migrations batched
- [ ] Migrations tested on production-size data

### Performance

- [ ] N+1 queries eliminated
- [ ] Pagination implemented correctly
- [ ] Connection pooling configured
- [ ] Slow query logging enabled
- [ ] Query plans reviewed for critical paths

### Operations

- [ ] Backup strategy documented
- [ ] Recovery tested regularly
- [ ] Monitoring/alerts configured
- [ ] Connection limits appropriate
- [ ] Maintenance windows scheduled

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Prisma Best Practices](https://www.prisma.io/docs/guides/performance-and-optimization)
- [PlanetScale Schema Design](https://planetscale.com/docs/learn/schema-design-best-practices)
- [Uber's Schema Evolution](https://eng.uber.com/schemaless-rewrite/)
