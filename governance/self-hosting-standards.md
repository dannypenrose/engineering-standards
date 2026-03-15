# Self-Hosting Standard

> Authoritative standards for self-hosted infrastructure, covering deployment, operations, and security of applications on dedicated servers.

## Purpose

Establish best practices for operating self-hosted services on dedicated infrastructure, ensuring reliability, security, and maintainability.

## Core Principles

1. **Automation first** - Infrastructure as code, no manual configuration
2. **Process management** - Automatic restart and recovery
3. **Security hardened** - Network isolation, minimal attack surface
4. **Monitored reliability** - Observability into all layers
5. **Easy updates** - Minimal downtime for application and OS updates
6. **Disaster recovery** - Defined backup and recovery procedures

## Self-Hosting Decision Matrix

### When to Self-Host

| Scenario | Self-Host | Cloud | Reason |
|----------|-----------|-------|--------|
| Compliance requires data residency | ✅ | ❌ | Geographic control |
| Fixed, predictable workload | ✅ | ❌ | Cost savings |
| Stable, non-scaling service | ✅ | ❌ | Operational simplicity |
| High burst traffic | ❌ | ✅ | Autoscaling needed |
| Regulatory requirements (HIPAA, etc) | ✅ | ✅ | Either viable |
| Startup/uncertain scale | ❌ | ✅ | Flexibility needed |
| Open source, containerized app | ✅ | ✅ | Either viable |

## Reverse Proxy Configuration (Caddy)

### Basic Setup

```caddy
# /etc/caddy/Caddyfile
{
  email ops@example.com
  acme_ca https://acme-v02.api.letsencrypt.org/directory
}

# Redirect HTTP → HTTPS
http://api.example.com {
  redir https://{host}{uri} permanent
}

# Main HTTPS server
https://api.example.com {
  # Proxy to backend service
  reverse_proxy localhost:3000 {
    # Health check
    uri /health
    interval 10s
    timeout 5s

    # Connection settings
    header_up X-Real-IP {http.request.remote}
    header_up X-Forwarded-For {http.request.remote}
    header_up X-Forwarded-Proto {http.request.proto}
    header_up Host {http.request.host}
  }

  # Security headers
  header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
  header X-Content-Type-Options nosniff
  header X-Frame-Options DENY
  header Referrer-Policy "strict-origin-when-cross-origin"

  # Logging
  log {
    output file /var/log/caddy/access.log {
      roll_size 100mb
      roll_keep 10
    }
    format json
  }
}
```

### Caddy Management

```bash
# Start service
systemctl start caddy
systemctl enable caddy

# Reload configuration (no downtime)
caddy reload -c /etc/caddy/Caddyfile

# View logs
journalctl -u caddy -f

# Check certificate status
caddy list-certs
caddy trust

# Verify SSL
curl -I https://api.example.com
```

## Process Management (systemd)

### Service Unit File

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target postgresql.service

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js

# Environment
EnvironmentFile=/etc/myapp.env
Environment="NODE_ENV=production"
Environment="LOG_LEVEL=info"

# Restart policy
Restart=always
RestartSec=10
StartLimitInterval=600
StartLimitBurst=3

# Resource limits
LimitNOFILE=65535
MemoryLimit=1G

# Security
NoNewPrivileges=true
PrivateTmp=yes
ProtectHome=yes
ProtectSystem=strict
ReadWritePaths=/var/log/myapp /var/cache/myapp

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```

### Service Management

```bash
# Start service
systemctl start myapp

# Enable on boot
systemctl enable myapp

# View status
systemctl status myapp

# View logs
journalctl -u myapp -f
journalctl -u myapp --since "2024-01-15 10:00"

# Restart service (with brief downtime)
systemctl restart myapp

# Reload config (no downtime, if supported by app)
systemctl reload myapp

# Stop service
systemctl stop myapp
```

### Health Checks in systemd

```ini
# ExecHealthCheck runs periodically
ExecHealthCheck=/usr/local/bin/health-check.sh

# Script: /usr/local/bin/health-check.sh
#!/bin/bash
curl -sf http://localhost:3000/health > /dev/null || exit 1
```

## Container Orchestration (Docker)

### Docker Compose Setup

```yaml
version: '3.8'
services:
  api:
    image: myregistry.azurecr.io/myapp:latest
    restart: always
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@postgres:5432/db
      - REDIS_URL=redis://redis:6379
    volumes:
      - /var/log/myapp:/app/logs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - internal

  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=myapp
    volumes:
      - /data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

  redis:
    image: redis:7-alpine
    restart: always
    ports:
      - "127.0.0.1:6379:6379"
    volumes:
      - /data/redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

networks:
  internal:
    driver: bridge
```

### Docker Management

```bash
# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f api

# Update images and restart
docker-compose pull
docker-compose up -d

# Stop all services
docker-compose down

# Backup volumes
docker run --rm -v postgres_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/postgres.tar.gz -C /data .
```

## SSL/TLS Certificate Management

### Let's Encrypt with Caddy (Automatic)

```bash
# Caddy automatically obtains and renews certificates
# Just specify https:// and a valid domain

# Manual renewal check
caddy renew

# Force renewal
certbot renew --force-renewal

# Check expiration
openssl x509 -in /etc/caddy/certificates/.../cert.pem -noout -dates
```

### Manual Certificate Management

```bash
# Generate certificate
certbot certonly --standalone -d api.example.com

# Renew manually
certbot renew

# Cron job for auto-renewal
0 3 * * * certbot renew --quiet

# Verify renewal
ls -la /etc/letsencrypt/live/api.example.com/
```

### Certificate Alerts

```bash
# Monitor expiration (in cron)
#!/bin/bash
CERTFILE="/etc/letsencrypt/live/api.example.com/cert.pem"
EXPIREDATE=$(openssl x509 -enddate -noout -in $CERTFILE | cut -d= -f2)
EXPIREDATE_EPOCH=$(date -d "$EXPIREDATE" +%s)
NOW=$(date +%s)
DAYS_LEFT=$(( ($EXPIREDATE_EPOCH - $NOW) / 86400 ))

if [ $DAYS_LEFT -lt 14 ]; then
  echo "ALERT: Certificate expires in $DAYS_LEFT days" | \
    mail -s "Certificate expiration warning" ops@example.com
fi
```

## DNS Configuration

### DNS Records

```dns
# A record
api.example.com A 203.0.113.42

# AAAA record (IPv6)
api.example.com AAAA 2001:db8::1

# CNAME (optional subdomain)
www.example.com CNAME api.example.com

# TXT records for security
# SPF
example.com TXT "v=spf1 include:_spf.google.com ~all"

# DMARC
_dmarc.example.com TXT "v=DMARC1; p=quarantine"

# DNSSEC
example.com DNSKEY ...
```

### DNS Verification

```bash
# Check A record
nslookup api.example.com
dig api.example.com

# Check all records
dig api.example.com ANY

# Check with specific nameserver
nslookup api.example.com 8.8.8.8

# Check propagation globally
# Use https://www.whatsmydns.net
```

## Monitoring & Alerting

### Prometheus Monitoring

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'api'
    static_configs:
      - targets: ['localhost:3000']
    metrics_path: '/metrics'

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']
```

### Alert Rules

```yaml
groups:
  - name: application
    rules:
      - alert: ServiceDown
        expr: up{job="api"} == 0
        for: 1m
        annotations:
          summary: "API service is down"

      - alert: HighErrorRate
        expr: rate(http_requests_total{status="5xx"}[5m]) > 0.01
        for: 5m
        annotations:
          summary: "High error rate detected"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 5m
        annotations:
          summary: "Disk space below 10%"
```

## Backup & Recovery

### Service Backup Strategy

```bash
# Daily backup script
#!/bin/bash

BACKUP_DIR="/backups/myapp"
DATE=$(date +%Y%m%d_%H%M%S)

# Stop application
systemctl stop myapp

# Backup code
tar czf $BACKUP_DIR/code_$DATE.tar.gz /opt/myapp

# Backup configuration
tar czf $BACKUP_DIR/config_$DATE.tar.gz /etc/myapp.env /etc/systemd/system/myapp.service

# Backup database (if running)
pg_dump myapp_db | gzip > $BACKUP_DIR/database_$DATE.sql.gz

# Restart application
systemctl start myapp

# Verify service is healthy
sleep 5
curl -f http://localhost:3000/health || {
  echo "ERROR: Service did not recover"
  exit 1
}

# Upload to offsite storage
aws s3 sync $BACKUP_DIR s3://backups-prod/myapp --delete

# Cleanup old backups (>30 days)
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

logger -t backup "Backup completed for myapp"
```

### Cron Schedule

```bash
# Daily backup at 2 AM
0 2 * * * /usr/local/bin/backup-myapp.sh

# Weekly full backup on Sunday
0 3 * * 0 /usr/local/bin/backup-full-myapp.sh
```

## Automated Updates

### System Updates

```bash
# Enable automatic security updates
apt install unattended-upgrades apt-listchanges

# Configure in /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::Remove-Unused-Packages "true";

# Test
unattended-upgrade --dry-run
```

### Application Updates

```bash
# Update Docker images with zero downtime
#!/bin/bash
cd /opt/myapp

# Pull latest image
docker-compose pull

# Start new container with old container still running
docker-compose up -d --no-deps --scale api=2 api

# Wait for health check
sleep 10
docker-compose exec api curl http://localhost:3000/health

# Remove old container
docker-compose up -d --no-deps --scale api=1 api
```

## Auto-Start & Service Recovery

### Systemd Auto-Restart

Already configured in service unit:
```ini
Restart=always
RestartSec=10
StartLimitInterval=600
StartLimitBurst=3
```

This ensures:
- Service restarts automatically on crash
- Waits 10 seconds between restart attempts
- Limits to 3 restarts per 10-minute window
- After 3 failures, requires manual intervention

### Monitoring Service Health

```bash
#!/bin/bash
# Monitoring script to alert on service issues

SERVICE="myapp"
ENDPOINT="http://localhost:3000/health"
ALERT_EMAIL="ops@example.com"

# Check if service is running
if ! systemctl is-active --quiet $SERVICE; then
  echo "ERROR: $SERVICE is not running" | mail -s "Service alert" $ALERT_EMAIL
  systemctl start $SERVICE
  exit 1
fi

# Check if service is healthy
if ! curl -sf $ENDPOINT > /dev/null; then
  echo "ERROR: $SERVICE health check failed" | mail -s "Service alert" $ALERT_EMAIL
  systemctl restart $SERVICE
  exit 1
fi

logger -t service-monitor "$SERVICE is healthy"
```

Schedule in cron:
```bash
# Every 5 minutes
*/5 * * * * /usr/local/bin/monitor-service.sh
```

## Checklist

### Infrastructure Setup

- [ ] Server provisioned and hardened
- [ ] SSH configured with key authentication
- [ ] Firewall rules configured
- [ ] Reverse proxy (Caddy/Nginx) installed
- [ ] SSL/TLS certificates obtained

### Service Configuration

- [ ] Systemd service unit created
- [ ] Health check endpoints working
- [ ] Environment variables configured
- [ ] Restart policy configured
- [ ] Resource limits set

### Networking & DNS

- [ ] DNS records created (A, AAAA)
- [ ] DNS propagation verified
- [ ] Reverse DNS configured (optional)
- [ ] Domain verification complete

### Monitoring

- [ ] Prometheus/monitoring agent installed
- [ ] Service metrics exposed
- [ ] Alert rules configured
- [ ] Log aggregation set up
- [ ] Dashboard created

### Backup & Recovery

- [ ] Backup script created
- [ ] Backup schedule configured
- [ ] Offsite backup destination tested
- [ ] Restore procedure documented and tested
- [ ] RPO/RTO targets defined

### Updates & Maintenance

- [ ] Automatic security updates enabled
- [ ] Update testing procedure defined
- [ ] Maintenance window scheduled
- [ ] Rollback procedure documented

## References

- [Caddy Web Server Documentation](https://caddyserver.com/docs/)
- [Systemd Service Unit Documentation](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Let's Encrypt Certificate Management](https://letsencrypt.org/getting-started/)
- [Prometheus Monitoring](https://prometheus.io/docs/introduction/overview/)
