# Backup & Disaster Recovery Standard

> Authoritative backup and disaster recovery standards aligned with industry best practices for RPO/RTO optimization.

## Purpose

Establish backup strategies and disaster recovery procedures that protect against data loss, minimize recovery time, and ensure business continuity.

## Core Principles

1. **3-2-1 rule** - Three copies of data, two different media, one offsite
2. **Automated scheduling** - No manual backup processes
3. **Regular testing** - Backups verified monthly
4. **Clear RPO/RTO** - Defined targets per service tier
5. **Incremental efficiency** - Minimize bandwidth and storage costs
6. **Monitored reliability** - Alerts for backup failures

## RPO & RTO Targets

### Service Tiers

| Tier | RPO | RTO | Backup Strategy | Example Services |
|------|-----|-----|---|---|
| Tier 1 (Critical) | 1 hour | 15 minutes | Continuous + hourly snapshots | Payment, Auth APIs |
| Tier 2 (Important) | 4 hours | 1 hour | 4x daily snapshots | User database, Cache |
| Tier 3 (Standard) | 24 hours | 4 hours | Daily full + incremental | Content, Logs |
| Tier 4 (Non-critical) | 7 days | 24 hours | Weekly full | Archives, Exports |

**Legend:**
- **RPO** (Recovery Point Objective) - How much data loss is acceptable
- **RTO** (Recovery Time Objective) - How fast recovery must complete

## The 3-2-1 Rule

```
PRODUCTION DATA
    │
    ├─→ Backup Copy 1 (daily full)
    │   └─→ Local storage (fast restore)
    │
    ├─→ Backup Copy 2 (incremental)
    │   └─→ Different media/location (cold storage)
    │
    └─→ Backup Copy 3 (offsite)
        └─→ Remote datacenter (disaster recovery)
```

**Implementation:**

1. **Primary backup** - Local NAS/attached storage (restore within hours)
2. **Incremental backup** - External hard drive or local cold storage (immutable)
3. **Offsite backup** - Cloud storage or remote datacenter (geo-redundant)

## Backup Types

### Full Backup

```bash
# Full backup - complete data snapshot
# Use for: Initial backup, weekly/monthly archives

# Example: PostgreSQL full backup
pg_dump --format=custom --verbose \
  --file=/backups/full/database-$(date +%Y%m%d).dump \
  production_db

# Verify
pg_restore --list /backups/full/database-20240115.dump | head -20

# Storage: ~100 GB for 50 GB database (compression)
# Time: 30-60 minutes depending on size
```

### Incremental Backup

```bash
# Incremental - only changes since last backup
# Use for: Daily backups, frequent snapshots

# Example: rsync incremental with snapshots
rsync -av --delete --backup --backup-dir=/backups/incremental/$(date +%Y%m%d) \
  /data/application/ /backups/latest/

# Day 1: 100 GB (full copy)
# Day 2: 2 GB (incremental)
# Day 3: 1 GB (incremental)
# Total: 103 GB vs 300 GB with 3 full backups
```

### Differential Backup

```bash
# Differential - changes since last FULL backup
# Use for: Balance between full and incremental

# Approach: Mark files with full backup timestamp
find /data -newer /tmp/last_full_backup.marker > /tmp/changed_files.txt
tar -czf /backups/diff/database-$(date +%Y%m%d).tar.gz \
  --files-from /tmp/changed_files.txt

# Day 1: 100 GB (full)
# Day 2: 2 GB (all changes since day 1)
# Day 3: 3 GB (all changes since day 1, includes day 2 changes)
# Total: 105 GB
```

## PostgreSQL Backup Strategy

### WAL Archiving (Point-in-Time Recovery)

```bash
# postgresql.conf - Enable WAL archiving
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backups/wal/%f && cp %p /backups/wal/%f'
archive_timeout = 300  # Archive every 5 minutes max

# Verify archiving
ls -la /backups/wal/ | tail -10

# Check archive status
SELECT * FROM pg_stat_archiver;
```

### Full + Incremental Strategy

```bash
# Full backup (weekly)
0 2 * * 0 /usr/local/bin/backup-full.sh

# Incremental backup (daily)
0 2 * * 1-6 /usr/local/bin/backup-incremental.sh

# Script: backup-full.sh
#!/bin/bash
BACKUP_DIR="/backups/full"
DATE=$(date +%Y%m%d_%H%M%S)
pg_dump --format=custom --verbose \
  --file="${BACKUP_DIR}/database_${DATE}.dump" \
  production_db
gzip "${BACKUP_DIR}/database_${DATE}.dump"
# Remove backups older than 90 days
find $BACKUP_DIR -name "*.gz" -mtime +90 -delete
logger -t pg-backup "Full backup completed: ${DATE}"
```

### Recovery Procedure

```bash
# 1. Restore from full backup
pg_restore --dbname=production_db \
  --clean --create --no-acl --no-owner \
  /backups/full/database_20240115_020000.dump.gz

# 2. Recover to specific point in time (if needed)
# This is handled automatically by PostgreSQL if WAL archiving is enabled
# Recovery command in recovery.conf:
restore_command = 'cp /backups/wal/%f %p'
recovery_target_timeline = 'latest'
recovery_target_time = '2024-01-15 15:30:00'
```

## Backup Verification & Testing

### Integrity Checks

```bash
# PostgreSQL backup integrity
pg_restore --list /backups/full/database.dump | wc -l

# File backup checksum
sha256sum /backups/full/database.dump > /backups/checksums/$(date +%Y%m%d).txt
# Verify later
sha256sum -c /backups/checksums/20240115.txt

# Compression integrity
gzip -t /backups/full/database.dump.gz && echo "Backup OK" || echo "CORRUPTED"
```

### Monthly Restore Test

```bash
# Test full recovery monthly
# 1. Create restore plan
RESTORE_TEST_DB="production_test"
BACKUP_FILE="/backups/full/database_20240115.dump.gz"

# 2. Prepare test environment
createdb $RESTORE_TEST_DB

# 3. Perform restore
gunzip -c $BACKUP_FILE | pg_restore --dbname=$RESTORE_TEST_DB

# 4. Validate data integrity
psql $RESTORE_TEST_DB -c "SELECT COUNT(*) FROM users;"
psql $RESTORE_TEST_DB -c "SELECT COUNT(*) FROM transactions;"

# 5. Document results
echo "Restore test completed: $(date)" >> /var/log/restore_tests.log
echo "Record in runbook: test-restore-20240115"

# 6. Cleanup
dropdb $RESTORE_TEST_DB
```

## Retention Policies

### Standard Retention

```bash
# Database backups
# - Daily incrementals: keep 30 days
# - Weekly fulls: keep 90 days
# - Monthly archives: keep 7 years

# File backups
# - Daily incremental: keep 14 days
# - Weekly full: keep 52 weeks
# - Monthly cold storage: keep 7 years

# Application logs
# - Active logs: keep 30 days
# - Archived logs: keep 90 days
# - Compliance archive: keep 7 years
```

### Automated Retention Cleanup

```bash
# Script: cleanup-old-backups.sh
#!/bin/bash
BACKUP_DIR="/backups/full"
RETENTION_DAYS=90

find $BACKUP_DIR -name "*.dump*" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.tar*" -mtime +$RETENTION_DAYS -delete

# Log cleanup
DELETED=$(find $BACKUP_DIR -name "*.dump*" -mtime +$RETENTION_DAYS | wc -l)
logger -t backup-cleanup "Deleted $DELETED backups older than $RETENTION_DAYS days"

# Disk usage alert
USAGE=$(du -sh $BACKUP_DIR | cut -f1)
logger -t backup-cleanup "Backup storage: $USAGE"
```

## Offsite Backup Storage

### Cloud Storage (AWS S3)

```bash
# Upload backups to S3 (immutable, versioning enabled)
aws s3 cp /backups/full/database-20240115.dump.gz \
  s3://backups-prod/postgres/database-20240115.dump.gz \
  --storage-class GLACIER_IR \
  --region us-east-1

# Sync entire backup directory
aws s3 sync /backups/full s3://backups-prod/postgres \
  --delete \
  --storage-class STANDARD_IA

# Verify sync
aws s3 ls s3://backups-prod/postgres --recursive --summarize
```

### S3 Lifecycle Rules

```json
{
  "Rules": [
    {
      "Id": "ArchiveOldBackups",
      "Status": "Enabled",
      "Filter": {"Prefix": "postgres/"},
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

### Disaster Recovery Drill (Quarterly)

```bash
# Simulate complete disaster - restore from S3
# 1. Provision new server
# 2. Install PostgreSQL
# 3. Download backup from S3
aws s3 cp s3://backups-prod/postgres/database-20240115.dump.gz /tmp/

# 4. Restore
pg_restore --dbname=disaster_test /tmp/database-20240115.dump.gz

# 5. Validate
psql disaster_test -c "SELECT version();"

# 6. Document RTO
RTO=$(date -d "2024-01-15 10:00:00" +%s)
NOW=$(date +%s)
RECOVERY_TIME=$((NOW - RTO))
echo "RTO: ${RECOVERY_TIME} seconds"
```

## Backup Monitoring & Alerting

### Backup Status Checks

```bash
# Script: monitor-backups.sh
#!/bin/bash

# Check full backup freshness (must be < 24 hours old)
LAST_BACKUP=$(find /backups/full -name "*.dump*" -printf '%T@\n' | sort -rn | head -1)
NOW=$(date +%s)
AGE=$((NOW - ${LAST_BACKUP%.*}))

if [ $AGE -gt 86400 ]; then
  echo "CRITICAL: Full backup is $(($AGE / 3600)) hours old"
  # Send alert
  curl -X POST https://alerts.example.com/backups \
    -d '{"status": "CRITICAL", "message": "Backup overdue"}'
fi

# Check backup storage space
USAGE=$(du -s /backups | cut -f1)
LIMIT=$((5 * 1024 * 1024))  # 5 TB limit
if [ $USAGE -gt $LIMIT ]; then
  echo "WARNING: Backup storage at $(($USAGE / 1024 / 1024)) GB"
fi
```

### Prometheus Metrics

```yaml
# Backup metrics
backup_full_last_timestamp_seconds
backup_incremental_last_timestamp_seconds
backup_storage_bytes_used
backup_storage_bytes_free
backup_failures_total
backup_duration_seconds
backup_size_bytes
```

### Alert Rules

```yaml
- alert: BackupOverdue
  expr: (time() - backup_full_last_timestamp_seconds) > 86400
  for: 1h
  labels:
    severity: critical
  annotations:
    summary: "Full backup overdue"
    runbook: "https://wiki.example.com/runbooks/backup-overdue"

- alert: BackupStorageCritical
  expr: (backup_storage_bytes_used / backup_storage_bytes_free) > 0.8
  for: 15m
  annotations:
    summary: "Backup storage 80% full"
```

## Disaster Recovery Planning

### Recovery Procedures

| Scenario | RTO | Recovery Steps |
|----------|-----|---|
| Single table corruption | 2 hours | Restore point-in-time to clean state |
| Full database loss | 30 minutes | Restore latest full backup + replay WAL |
| Server hardware failure | 4 hours | Provision new server, restore from offsite backup |
| Regional datacenter loss | 24 hours | Promote secondary or restore from S3 |

### Documentation Requirements

```markdown
# Disaster Recovery Runbook

## Authority
- Owner: Infrastructure Team
- Last Tested: 2024-01-15
- Next Test: 2024-04-15

## Contact Chain
1. On-call engineer
2. Infrastructure lead
3. CTO

## Recovery Checklists
- [ ] Assess damage/scope
- [ ] Provision recovery infrastructure
- [ ] Download backups
- [ ] Perform restore
- [ ] Verify data integrity
- [ ] Communicate status to stakeholders
- [ ] Document post-incident
```

## Checklist

### Backup Strategy

- [ ] RPO/RTO defined per service
- [ ] 3-2-1 rule implemented
- [ ] Backup types selected (full, incremental, differential)
- [ ] Retention policy documented
- [ ] Backup scheduling automated

### Backup Execution

- [ ] Backups running on schedule
- [ ] Backup logs monitored
- [ ] Failures generating alerts
- [ ] Disk space checked regularly
- [ ] Backup ownership assigned

### Verification & Testing

- [ ] Monthly restore tests performed
- [ ] Backup integrity validated
- [ ] Test results documented
- [ ] Recovery procedures validated
- [ ] Team trained on procedures

### Offsite Storage

- [ ] Offsite backups automated
- [ ] Multiple geographic regions covered
- [ ] Versioning/immutability enabled
- [ ] Access credentials secured
- [ ] Quarterly disaster drills conducted

### Monitoring

- [ ] Backup freshness alerts configured
- [ ] Storage capacity alerts configured
- [ ] Backup failure alerts configured
- [ ] Dashboard showing backup status
- [ ] Metrics collected and trended

## References

- [PostgreSQL Backup Documentation](https://www.postgresql.org/docs/current/backup.html)
- [AWS Backup Best Practices](https://docs.aws.amazon.com/aws-backup/latest/userguide/best-practices.html)
- [3-2-1 Backup Rule](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/)
- [NIST Guidelines for Backup](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-34r1.pdf)
- [RPO/RTO Planning Guide](https://www.atlassian.com/incident-management/handbook/disaster-recovery)
