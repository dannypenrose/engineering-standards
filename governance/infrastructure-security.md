# Infrastructure Security Standard

> Authoritative infrastructure security standards aligned with CIS Benchmarks, NIST, and industry best practices.

## Purpose

Establish security practices that protect server infrastructure, network systems, and service deployment environments from unauthorized access, data breaches, and operational compromise.

## Core Principles

1. **Defense in depth** - Multiple layers of network and system security controls
2. **Least privilege** - Minimum access and permissions required for functionality
3. **Secure by default** - Safe configurations applied at deployment
4. **Automated hardening** - Security applied through infrastructure code and automation
5. **Continuous monitoring** - Comprehensive audit logging and alerting
6. **Secure secrets** - No credentials in configuration, environment, or logs

## SSH Security

### SSH Configuration

```bash
# /etc/ssh/sshd_config - Hardened SSH settings
Port 22                              # Use non-standard port in production
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

# Authentication
PermitRootLogin no                   # Disable root login
PasswordAuthentication no             # SSH keys only
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3

# Session security
X11Forwarding no                     # Disable X11
AllowTcpForwarding no
PermitTTY yes
ClientAliveInterval 300
ClientAliveCountMax 2
MaxSessions 10

# Cryptography
HostKey /etc/ssh/ssh_host_ed25519_key
KexAlgorithms curve25519-sha256
Ciphers chacha20-poly1305@openssh.com,aes-256-gcm@openssh.com
MACs umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Banner
Banner /etc/ssh/banner.txt

# Logging
SyslogFacility AUTH
LogLevel VERBOSE
```

### SSH Key Management

```bash
# Generate strong SSH keys for deployment users
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -C "deploy-user@$(hostname)"

# Client-side SSH config (~/.ssh/config)
Host production-server
  HostName api.example.com
  User deploy
  IdentityFile ~/.ssh/deploy_key
  AddKeysToAgent yes
  IdentitiesOnly yes
  ServerAliveInterval 60
  ServerAliveCountMax 5
  StrictHostKeyChecking accept-new
  UserKnownHostsFile ~/.ssh/known_hosts
```

### Enforce ED25519 Keys

```bash
# Only accept ED25519 (no legacy RSA/DSA)
find /home/*/.ssh/authorized_keys -exec grep -L "ssh-ed25519" {} \;

# Audit non-compliant keys
echo "Audit: Non-ED25519 keys found. Rotate immediately."
```

## Firewall & Network Segmentation

### UFW (Uncomplicated Firewall) Rules

```bash
# Enable UFW
ufw enable

# Default policies
ufw default deny incoming
ufw default allow outgoing
ufw default deny routed

# SSH (rate-limited)
ufw limit 22/tcp comment 'SSH'

# HTTP/HTTPS (for web services)
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'

# Application-specific ports (limit by source IP if possible)
ufw allow from 10.0.0.0/8 to any port 3306 comment 'MySQL from internal'

# Status and verify
ufw status verbose
```

### Network Segmentation

```
┌─────────────────────────────────────────────┐
│           INTERNET                          │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────▼──────────┐
        │  Reverse Proxy      │  (Caddy/Nginx)
        │  (Port 80, 443)     │
        └──────────┬──────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    │    API       │    Cache     │    Services
    │   (3000)     │   (6379)     │   (internal)
    │              │              │
    │ ┌──────────┐ │ ┌──────────┐ │ ┌─────────┐
    │ │ NestJS   │ │ │ Redis    │ │ │ Worker  │
    │ │ Instance │ │ │ Instance │ │ │ Queue   │
    │ └──────────┘ │ └──────────┘ │ └─────────┘
    │              │              │
    └──────────────┼──────────────┘
                   │
        ┌──────────▼──────────┐
        │  PostgreSQL         │
        │  (5432, internal)   │
        └─────────────────────┘
```

**Firewall Rules by Zone:**

| Service | Source | Port | Protocol | Notes |
|---------|--------|------|----------|-------|
| SSH | Admin IP range | 22 | TCP | Rate-limited |
| HTTP | 0.0.0.0 | 80 | TCP | Reverse proxy only |
| HTTPS | 0.0.0.0 | 443 | TCP | Reverse proxy only |
| API | 127.0.0.1 | 3000 | TCP | Localhost only |
| Redis | 127.0.0.1 | 6379 | TCP | Localhost only |
| PostgreSQL | 127.0.0.1 | 5432 | TCP | Localhost only |

## Secrets on Servers

### Environment File Conventions

```bash
# /etc/{app_name}.env (for systemd EnvironmentFile=)
DATABASE_URL=postgresql://user:pass@localhost/db
JWT_SECRET=base64-encoded-secret
API_KEY=sk_live_xxxxxxxxxxxx
REDIS_URL=redis://localhost:6379

# File permissions (CRITICAL)
chmod 600 /etc/{app_name}.env      # Owner read/write only
chown {deploy_user}:{deploy_user} /etc/{app_name}.env

# Verify permissions
ls -l /etc/{app_name}.env          # Should show: -rw------- 1 deploy deploy
```

### Secrets Storage Best Practices

```bash
# NEVER
echo "DATABASE_URL=..." >> .env      # Don't commit .env
echo "SECRET=..." > config.json      # Don't hardcode in code
git log --all --source --grep="DATABASE"  # Don't commit secrets

# Better - Use environment files with strict permissions
# Manage via configuration management (Ansible, Terraform)
# Rotate regularly (monthly for production)

# Use systemd with EnvironmentFile=
# [Service]
# Type=simple
# User=appuser
# EnvironmentFile=/etc/myapp.env
# ExecStart=/usr/bin/node /opt/myapp/server.js
```

### Secret Rotation

```bash
# Automate rotation monthly
0 0 1 * * /usr/local/bin/rotate-secrets.sh

# rotation script example
#!/bin/bash
NEW_SECRET=$(openssl rand -base64 32)
sed -i "s/^JWT_SECRET=.*/JWT_SECRET=${NEW_SECRET}/" /etc/{app_name}.env
systemctl restart {app_name}
logger -t secret-rotation "Rotated JWT_SECRET for {app_name}"
```

## Fail2ban Configuration

### Basic Setup

```bash
# Install
apt install fail2ban

# /etc/fail2ban/jail.d/sshd.conf
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
findtime = 600
bantime = 3600

# [recidive] for repeat offenders
[recidive]
enabled = true
filter = recidive
action = iptables-multiport[name=Recidive, port="http,https,ssh"]
logpath = /var/log/fail2ban.log
bantime = 86400
findtime = 86400
maxretry = 3
```

### Fail2ban Commands

```bash
# Start service
systemctl start fail2ban
systemctl enable fail2ban

# View ban status
fail2ban-client status
fail2ban-client status sshd

# Unban IP
fail2ban-client set sshd unbanip 192.168.1.100

# Monitor bans in real-time
tail -f /var/log/fail2ban.log
```

## TLS Configuration

### Certificate Management

```bash
# Using Let's Encrypt with Certbot
apt install certbot python3-certbot-nginx  # or certbot-dns-*

# Generate certificate
certbot certonly --standalone -d api.example.com

# Auto-renewal
systemctl enable certbot.timer
systemctl start certbot.timer

# Verify renewal
certbot renew --dry-run
```

### Nginx TLS Configuration

```nginx
# /etc/nginx/snippets/ssl-params.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;

# HSTS header
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

# OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/api.example.com/chain.pem;
```

### Server Block with TLS

```nginx
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$server_name$request_uri;  # Redirect HTTP → HTTPS
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    include /etc/nginx/snippets/ssl-params.conf;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Container Security

### Docker Best Practices

```dockerfile
# Use minimal base images
FROM node:20-alpine

# Run as non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Copy files with correct ownership
COPY --chown=nodejs:nodejs . /app

# No sudo, no passwords
USER nodejs

# Read-only root filesystem where possible
WORKDIR /app
RUN chmod -R 555 /app

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD node healthcheck.js
```

### Container Registry Security

```bash
# Scan images for vulnerabilities
docker scan my-app:latest

# Sign images (Docker Content Trust)
export DOCKER_CONTENT_TRUST=1
docker push my-registry/my-app:latest

# Use private registries
# Never use public registries for proprietary code
```

## Automated Patching

### Unattended Upgrades (Debian/Ubuntu)

```bash
# Install
apt install unattended-upgrades apt-listchanges

# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
};

Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Mail "ops@example.com";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";

# Schedule
# /etc/apt/apt.conf.d/02periodic
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

### Kernel Updates

```bash
# Check for updates
sudo apt list --upgradable | grep linux

# Plan maintenance window for kernel updates
# Requires reboot, communicate in advance

# Automatic reboot if needed (configure carefully)
Unattended-Upgrade::Automatic-Reboot "false"
Unattended-Upgrade::Automatic-Reboot-Time "02:00"
```

## Audit Logging

### Syslog Configuration

```bash
# /etc/rsyslog.d/90-infrastructure.conf
:msg, contains, "security" /var/log/security.log
:msg, contains, "auth" /var/log/auth.log
:msg, contains, "failed" /var/log/failures.log

# Centralized logging (optional)
*.* @@remote-syslog-server:514
```

### Auditd Setup

```bash
# Install
apt install auditd

# Rules for authentication
auditctl -a always,exit -F path=/etc/shadow -F perm=wa -F auid>=1000 -F auid!=-1 -k shadow_changes
auditctl -a always,exit -F path=/etc/passwd -F perm=wa -F auid>=1000 -F auid!=-1 -k passwd_changes

# Rules for system calls
auditctl -a always,exit -F arch=b64 -S execve -k exec

# View logs
auditctl -l
tail -f /var/log/audit/audit.log

# Parse audit logs
ausearch -m EXECVE -ts recent
```

### Log Retention

```bash
# /etc/logrotate.d/rsyslog
/var/log/auth.log
/var/log/security.log
{
    daily
    missingok
    rotate 90
    compress
    delaycompress
    notifempty
    create 640 root adm
    sharedscripts
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

## DNS Security

### DNSSEC

```bash
# Check DNSSEC status
dig +dnssec example.com

# Validate DNSSEC in resolver
# /etc/systemd/resolved.conf
[Resolve]
DNSSEC=yes
DNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4
```

### DNS Records (SPF, DKIM, DMARC)

```dns
# SPF record
example.com IN TXT "v=spf1 include:_spf.google.com ~all"

# DMARC policy
_dmarc.example.com IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"

# Verify records
nslookup -type=TXT example.com
```

## Checklist

### SSH Security

- [ ] Root login disabled
- [ ] Password authentication disabled (keys only)
- [ ] SSH keys are ED25519 or RSA 4096+
- [ ] SSH config hardened (no X11, no TCP forwarding)
- [ ] SSH port non-standard (optional, requires documentation)
- [ ] Known hosts verified

### Firewall

- [ ] UFW or equivalent enabled
- [ ] Inbound defaults to deny
- [ ] Only required ports open
- [ ] SSH rate-limited
- [ ] Firewall rules documented

### Secrets

- [ ] No hardcoded secrets in code
- [ ] Environment files with chmod 600
- [ ] Secrets managed per environment
- [ ] Secret rotation scheduled
- [ ] No secrets in logs

### TLS

- [ ] HTTPS enforced (HTTP redirects)
- [ ] TLS 1.2+ only
- [ ] Certificates auto-renewed
- [ ] HSTS header set
- [ ] Certificate monitoring in place

### Container Security

- [ ] Base images are minimal
- [ ] Containers run as non-root
- [ ] Health checks configured
- [ ] Images scanned for vulnerabilities

### Patching

- [ ] Automated security updates enabled
- [ ] Patch testing process documented
- [ ] Kernel updates tested before deployment
- [ ] Update logs maintained

### Audit Logging

- [ ] Syslog configured
- [ ] Authentication events logged
- [ ] Failed access attempts logged
- [ ] Log retention policy in place
- [ ] Log aggregation configured (optional)

## References

- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [SSH Best Practices](https://man.openbsd.org/sshd_config)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)
