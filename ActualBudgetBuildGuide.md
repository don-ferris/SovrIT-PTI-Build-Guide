# Actual Budget and SovrIT VPS Baseline Stack Setup Guide (Ubuntu 24.04)

This guide deploys:

## Application Stack

- Actual Budget
- Uptime Kuma
- Autoheal

## Infrastructure

- Docker
- Docker Compose
- Caddy reverse proxy
- automatic HTTPS/TLS

## Reliability

- Docker healthchecks
- Autoheal
- Uptime monitoring
- ntfy push notifications

## Backup Strategy

- Git versioning
- private GitHub replication
- cron automation
- per-service repositories

---

# 1. Initial Server Access

SSH into the VPS:

```bash
ssh root@YOUR_SERVER_IP
```

Verify you are root:

```bash
whoami
```

Expected:
- Output should be `root`

---

# 2. Update Ubuntu

```bash
apt update
```

```bash
apt upgrade -y
```

Validation:
- Command completes without errors

Optional reboot if kernel updates were installed:

```bash
reboot
```

Reconnect via SSH afterward.

---

# 3. Install Docker

```bash
apt install -y docker.io
```

Validation:

```bash
systemctl status docker
```

Look for:
- `active (running)`

---

# 4. Install Docker Compose

```bash
apt install -y docker-compose-v2
```

Validation:

```bash
docker compose version
```

Look for:
- Docker Compose version output

---

# 5. Install Caddy

```bash
apt install -y caddy
```

Validation:

```bash
caddy version
```

Look for:
- a version number

---

# 6. Create Docker Directory Structure

```bash
mkdir -p /root/docker
mkdir -p /root/scripts
```

---

# 7. Configure DNS

Create DNS A records:

| Hostname | Points To |
|---|---|
| budget.yourdomain.com | VPS IP |
| uptime.yourdomain.com | VPS IP |

If using Cloudflare:
- use “DNS Only” (gray cloud)

Validate DNS propagation:

```bash
dig +short budget.yourdomain.com
```

```bash
dig +short uptime.yourdomain.com
```

Look for:
- your VPS IP

---

# 8. Deploy Actual Budget

## Create Directory

```bash
mkdir -p /root/docker/actual-budget
cd /root/docker/actual-budget
```

---

## Create `.env`

```bash
nano .env
```

Contents:

```env
ACTUAL_CONTAINER_NAME=actual-budget
ACTUAL_PORT=5006
ACTUAL_DATA_DIR=./data
```

---

## Create `docker-compose.yml`

```bash
nano docker-compose.yml
```

Contents:

```yaml
services:
  actual:
    image: actualbudget/actual-server:latest
    container_name: ${ACTUAL_CONTAINER_NAME}

    ports:
      - "${ACTUAL_PORT}:5006"

    volumes:
      - ${ACTUAL_DATA_DIR}:/data

    restart: unless-stopped

    labels:
      autoheal: "true"

    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "fetch('http://localhost:5006').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

---

## Validate Compose

```bash
docker compose config
```

Look for:
- rendered YAML
- no errors

---

## Start Actual Budget

```bash
docker compose up -d
```

Validation:

```bash
docker ps
```

Look for:
- `actual-budget`
- `(healthy)`

---

# 9. Configure Caddy for Actual Budget

Edit Caddyfile:

```bash
nano /etc/caddy/Caddyfile
```

Contents:

```caddy
budget.yourdomain.com {
    reverse_proxy localhost:5006
}
```

Validate:

```bash
caddy validate --config /etc/caddy/Caddyfile
```

Look for:
- `Valid configuration`

Reload:

```bash
systemctl reload caddy
```

Validation:
- Visit:
  - `https://budget.yourdomain.com`
- Actual Budget setup page should appear
- Browser should show valid HTTPS

---

# 10. Deploy Autoheal

## Create Directory

```bash
mkdir -p /root/docker/autoheal
cd /root/docker/autoheal
```

---

## Create Compose File

```bash
nano docker-compose.yml
```

Contents:

```yaml
services:
  autoheal:
    image: willfarrell/autoheal:latest
    container_name: autoheal

    environment:
      AUTOHEAL_CONTAINER_LABEL: autoheal

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

    restart: unless-stopped
```

---

## Start Autoheal

```bash
docker compose up -d
```

Validation:

```bash
docker ps
```

Look for:
- `autoheal`
- `(healthy)`

---

# 11. Deploy Uptime Kuma

## Create Directory

```bash
mkdir -p /root/docker/uptime-kuma
cd /root/docker/uptime-kuma
```

---

## Create `.env`

```bash
nano .env
```

Contents:

```env
KUMA_CONTAINER_NAME=uptime-kuma
KUMA_PORT=3033
KUMA_DATA_DIR=./data
```

---

## Create Compose File

```bash
nano docker-compose.yml
```

Contents:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: ${KUMA_CONTAINER_NAME}

    ports:
      - "${KUMA_PORT}:3001"

    volumes:
      - ${KUMA_DATA_DIR}:/app/data

    restart: unless-stopped

    labels:
      autoheal: "true"

    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "fetch('http://localhost:3001').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

---

## Start Uptime Kuma

```bash
docker compose up -d
```

Validation:

```bash
docker ps
```

Look for:
- `uptime-kuma`
- `(healthy)`

---

# 12. Configure Caddy for Uptime Kuma

Edit:

```bash
nano /etc/caddy/Caddyfile
```

Contents:

```caddy
budget.yourdomain.com {
    reverse_proxy localhost:5006
}

uptime.yourdomain.com {
    reverse_proxy localhost:3033
}
```

Validate:

```bash
caddy validate --config /etc/caddy/Caddyfile
```

Reload:

```bash
systemctl reload caddy
```

Validation:
- Visit:
  - `https://uptime.yourdomain.com`
- Uptime Kuma setup page should appear
- HTTPS should work

---

# 13. Configure ntfy Notifications

Install ntfy app:

- Android:
  https://play.google.com/store/apps/details?id=io.heckel.ntfy
- iPhone:
  https://apps.apple.com/us/app/ntfy/id1625396347

Create:
- unique topic name

Test manually:

```bash
curl -d "Test notification" https://ntfy.sh/YOUR_TOPIC
```

Validation:
- phone notification arrives

---

# 14. Configure Uptime Kuma Monitoring

Visit:

```text
https://uptime.yourdomain.com
```

Create admin account.

Add monitor:

| Setting | Value |
|---|---|
| Type | HTTP(s) |
| URL | https://budget.yourdomain.com |
| Interval | 60s |

Validation:
- monitor eventually shows `UP`

---

# 15. Configure ntfy in Uptime Kuma

In Uptime Kuma:
- Settings
- Notifications
- Setup Notification

Values:

| Field | Value |
|---|---|
| Type | ntfy |
| URL | https://ntfy.sh |
| Topic | your topic |

Test notification.

Validation:
- notification arrives on phone

Attach notification to Actual Budget monitor.

---

# 16. Configure GitHub SSH Access

## Generate SSH Key

```bash
ssh-keygen -t ed25519 -C "selfhosted-backups"
```

Suggested filename:

```text
/root/.ssh/github_selfhosted
```

---

## Create SSH Config

```bash
nano /root/.ssh/config
```

Contents:

```sshconfig
Host github.com
    HostName github.com
    User git
    IdentityFile /root/.ssh/github_selfhosted
    IdentitiesOnly yes
```

Permissions:

```bash
chmod 600 /root/.ssh/config
```

---

## Add Public Key to GitHub

Display key:

```bash
cat /root/.ssh/github_selfhosted.pub
```

Add contents to:
- GitHub
- Settings
- SSH and GPG Keys

---

## Test GitHub SSH

```bash
ssh -T git@github.com
```

Validation:
- GitHub authentication success message

---

# 17. Configure Git Repositories

Create PRIVATE GitHub repositories:

| Repository |
|---|
| SelfHosted.SovrIT-VPS.Actual-Budget |
| SelfHosted.SovrIT-VPS.Uptime-Kuma |
| SelfHosted.SovrIT-VPS.Autoheal |

DO NOT:
- initialize with README
- add license
- add .gitignore

---

# 18. Configure Actual Budget Git Backup

## Initialize Repo

```bash
cd /root/docker/actual-budget
git init
```

---

## Create `.gitignore`

```bash
nano .gitignore
```

Contents:

```gitignore
*.log
.DS_Store
Thumbs.db
docker-compose.override.yml
tmp/
temp/
```

---

## Initial Commit

```bash
git add .
git commit -m "Initial Actual Budget setup"
```

---

## Add Remote

```bash
git remote add origin git@github.com:YOUR_USERNAME/SelfHosted.SovrIT-VPS.Actual-Budget.git
```

---

## Push

```bash
git push -u origin master
```

Validation:
- repo appears on GitHub

---

# 19. Configure Actual Budget Backup Script

Create script:

```bash
nano /root/scripts/actual-budget-backup.sh
```

Contents:

```bash
#!/usr/bin/env bash

set -e

REPO_DIR="/root/docker/actual-budget"

cd "$REPO_DIR"

git add .

if ! git diff --cached --quiet; then
    git commit -m "Automated backup $(date '+%Y-%m-%d %H:%M:%S')"
    git push
else
    echo "No changes to commit."
fi
```

Make executable:

```bash
chmod +x /root/scripts/actual-budget-backup.sh
```

---

# 20. Configure Uptime Kuma Git Backup

Repeat similar steps for:

```text
/root/docker/uptime-kuma
```

Backup script:

```bash
nano /root/scripts/uptime-kuma-backup.sh
```

Contents:

```bash
#!/usr/bin/env bash

set -e

REPO_DIR="/root/docker/uptime-kuma"

cd "$REPO_DIR"

git add .

if ! git diff --cached --quiet; then
    git commit -m "Automated backup $(date '+%Y-%m-%d %H:%M:%S')"
    git push
else
    echo "No changes to commit."
fi
```

Make executable:

```bash
chmod +x /root/scripts/uptime-kuma-backup.sh
```

---

# 21. Configure Autoheal Git Backup

Repeat Git setup for:

```text
/root/docker/autoheal
```

No cron job required.

---

# 22. Configure Cron Jobs

Edit crontab:

```bash
crontab -e
```

Add:

```cron
0 * * * * /root/scripts/actual-budget-backup.sh >> /var/log/actual-budget-backup.log 2>&1
0 3 * * 0 /root/scripts/uptime-kuma-backup.sh >> /var/log/uptime-kuma-backup.log 2>&1
```

Validation:

```bash
crontab -l
```

Look for:
- both cron entries present

---

# Final Result

## Application Stack

- Actual Budget
- Uptime Kuma
- Autoheal

## Infrastructure

- Docker
- Docker Compose
- Caddy reverse proxy
- automatic HTTPS/TLS

## Reliability

- Docker healthchecks
- Autoheal
- Uptime monitoring
- ntfy push notifications

## Backup Strategy

- Git versioning
- private GitHub replication
- cron automation
- per-service repositories
