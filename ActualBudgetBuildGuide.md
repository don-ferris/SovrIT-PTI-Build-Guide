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
- git SSH
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

## 8.1. Create Directory

```bash
mkdir -p /root/docker/actual-budget
cd /root/docker/actual-budget
```

---

## 8.2. Create `.env`

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

## 8.3. Create `docker-compose.yml`

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

## 8.4. Validate Compose

```bash
docker compose config
```

Look for:
- rendered YAML
- no errors

---

## 8.5. Start Actual Budget

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

## 10.1. Create Directory

```bash
mkdir -p /root/docker/autoheal
cd /root/docker/autoheal
```

---

## 10.2. Create Compose File

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

## 10.3. Start Autoheal

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

## 11.1. Create Directory

```bash
mkdir -p /root/docker/uptime-kuma
cd /root/docker/uptime-kuma
```

---

## 11.2. Create `.env`

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

## 11.3. Create Compose File

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

## 11.4. Start Uptime Kuma

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
curl \
  -H "Priority: urgent" \
  -H "Title: SovrIT Test Alert" \
  -d "Test notification" \
  https://ntfy.sh/YOUR_TOPIC
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
| Priority | urgent |
| Title | SovrIT Monitoring |
| URL | https://ntfy.sh |
| Topic | your topic |
| Tags | warning,rotating_light |

Test notification.

Validation:
- notification arrives on phone

Attach notification to Actual Budget monitor.

---

# 16. Configure GitHub SSH Access

## 16.1. Generate SSH Key

```bash
ssh-keygen -t ed25519 -C "selfhosted-backups"
```

Suggested filename:

```text
/root/.ssh/github_selfhosted
```

---

## 16.2. Create SSH Config

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

## 16.3. Add Public Key to GitHub

Display the public key:

```bash
cat /root/.ssh/github_selfhosted.pub
```

You should see output beginning with something like:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
```

Copy the ENTIRE line to your clipboard.

### Add the key in GitHub

1. Log into GitHub.

2. Click your profile picture (top-right corner).

3. Click:

```text
Settings
```

4. In the left sidebar, scroll down and click:

```text
SSH and GPG keys
```

Direct URL:

```text
https://github.com/settings/keys
```

5. Click:

```text
New SSH key
```

6. Fill out:

| Field | Value |
|---|---|
| Title | `SovrIT VPS Backups` (or any descriptive name) |
| Key type | Authentication Key |
| Key | Paste the entire public key |

Example title:

```text
SovrIT VPS Backups
```

7. Click:

```text
Add SSH key
```

8. GitHub may ask for:
- password
- passkey
- MFA confirmation

Approve the request.

### Validate GitHub SSH Access

Back on the VPS:

```bash
ssh -T git@github.com
```

Expected result:

```text
Hi YOUR_USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

This confirms:
- SSH key works
- GitHub trusts the VPS
- Git operations will authenticate successfully

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

Actual Budget changes frequently because it contains your budget database and uploaded files.

We want:

- Git versioning
- private GitHub replication
- automated offsite backup
- recoverability

Because Actual Budget changes often, we will back it up **hourly**.

---

## 18.1. Change Into the Actual Budget Directory

```bash
cd /root/docker/actual-budget
```
---

## 18.2. Initialize Repository

Initialize Git:

```bash
git init
```

Validation:

```bash
git status
```

Look for:

```text
On branch master

No commits yet
```

---

## 18.3. Create `.gitignore`

Create the file:

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

Save and exit.

---

## 18.4. Create Initial Commit

IMPORTANT:

You **must create an initial commit before pushing to GitHub**.

Stage all files:

```bash
git add . && git status
```

Look for staged files including:

- `.env`
- `.gitignore`
- `docker-compose.yml`
- `data/`

Create the commit:

```bash
git commit -m "Initial Actual Budget setup"
```

Validation:

Look for output similar to:

```text
[master (root-commit) abc1234] Initial Actual Budget setup
```

Important:

- `(root-commit)` confirms the repository has an initial commit

---

## 18.5. Create GitHub Repository

In GitHub:

Create a NEW PRIVATE repository.

Recommended name:

```text
SelfHosted.VPS.Actual-Budget
```

Important:

- PRIVATE
- NO README
- NO `.gitignore`
- NO license

---

## 18.6. Add GitHub Remote

Back on the VPS, add the remote:

```bash
git remote add origin git@github.com:YOUR_USERNAME/SelfHosted.VPS.Actual-Budget.git
```

Example format:

```text
git@github.com:YOUR_USERNAME/SelfHosted.VPS.Actual-Budget.git && git remote -v
```

Validate:

```bash
git remote -v
```

Look for:

```text
origin  git@github.com:YOUR_USERNAME/SelfHosted.VPS.Actual-Budget.git (fetch)
origin  git@github.com:YOUR_USERNAME/SelfHosted.VPS.Actual-Budget.git (push)
```

---

## 18.7. Push to GitHub

Push the repository:

```bash
git push -u origin master
```

Validation:

Look for output similar to:

```text
[new branch]      master -> master
branch 'master' set up to track 'origin/master'
```

Then open the repository in GitHub.

Look for:

- `.env`
- `.gitignore`
- `docker-compose.yml`
- `data/`

This confirms:

- Git versioning works
- SSH authentication works
- offsite replication works
- Actual Budget is recoverable

---

## 18.8. Create Backup Script

Create the backup script:

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

Save and exit.

Make executable:

```bash
chmod +x /root/scripts/actual-budget-backup.sh
```

Validation:

```bash
ls -lah /root/scripts
```

Look for:

```text
actual-budget-backup.sh
```

with executable permissions (`x`).

---

## 18.9. Configure Hourly Cron Backup

Edit root crontab:

```bash
crontab -e
```

Add:

```cron
0 * * * * /root/scripts/actual-budget-backup.sh >> /var/log/actual-budget-backup.log 2>&1
```

Meaning:

- minute `0`
- every hour
- every day
- run backup script
- log output

Validation:

```bash
crontab -l
```

Look for:

```cron
0 * * * * /root/scripts/actual-budget-backup.sh >> /var/log/actual-budget-backup.log 2>&1
```

This confirms:

- Actual Budget is automatically backed up hourly
- backups are committed to Git
- backups are replicated to GitHub
- recovery is automated
  
---

# 19. Configure Uptime Kuma Git Backup

Uptime Kuma changes occasionally because it stores:

- monitor configuration
- notification configuration
- uptime history
- alerting settings

We want:

- Git versioning
- private GitHub replication
- automated offsite backup
- recoverability

Because Uptime Kuma changes occasionally, we will back it up **weekly**.

---

## 19.1. Change Into the Uptime Kuma Directory

```bash
cd /root/docker/uptime-kuma
```

---

## Initialize Repository

Initialize Git:

```bash
git init
```

Validation:

```bash
git status
```

Look for:

```text
On branch master

No commits yet
```

---

## 19.2. Create `.gitignore`

Create the file:

```bash
nano .gitignore
```

Contents:

```gitignore
*.log
.DS_Store
Thumbs.db
```

Save and exit.

---

## 19.3. Create Initial Commit

IMPORTANT:

You **must create an initial commit before pushing to GitHub**.

Stage all files:

```bash
git add . && git status
```

Look for staged files including:

- `.env`
- `.gitignore`
- `docker-compose.yml`
- `data/`

Create the commit:

```bash
git commit -m "Initial Uptime Kuma setup"
```

Validation:

Look for output similar to:

```text
[master (root-commit) abc1234] Initial Uptime Kuma setup
```

Important:

- `(root-commit)` confirms the repository has an initial commit

---

## 19.4. Create GitHub Repository

In GitHub:

Create a NEW PRIVATE repository.

Recommended name:

```text
SelfHosted.VPS.Uptime-Kuma
```

Important:

- PRIVATE
- NO README
- NO `.gitignore`
- NO license

---

## 19.5. Add GitHub Remote

Back on the VPS, add the remote:

```bash
git remote add origin git@github.com:YOUR_USERNAME/SelfHosted.VPS.Uptime-Kuma.git
```

Example format:

```text
git@github.com:YOUR_USERNAME/SelfHosted.VPS.Uptime-Kuma.git && git remote -v
```

Validate:

```bash
git remote -v
```

Look for:

```text
origin  git@github.com:YOUR_USERNAME/SelfHosted.VPS.Uptime-Kuma.git (fetch)
origin  git@github.com:YOUR_USERNAME/SelfHosted.VPS.Uptime-Kuma.git (push)
```

---

## 19.6. Push to GitHub

Push the repository:

```bash
git push -u origin master
```

Validation:

Look for output similar to:

```text
[new branch]      master -> master
branch 'master' set up to track 'origin/master'
```

Then open the repository in GitHub.

Look for:

- `.env`
- `.gitignore`
- `docker-compose.yml`
- `data/`

This confirms:

- Git versioning works
- SSH authentication works
- offsite replication works
- Uptime Kuma is recoverable

---

## 19.7. Create Backup Script

Create the backup script:

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

Save and exit.

Make executable:

```bash
chmod +x /root/scripts/uptime-kuma-backup.sh
```

Validation:

```bash
ls -lah /root/scripts
```

Look for:

```text
uptime-kuma-backup.sh
```

with executable permissions (`x`).

---

## 19.8. Configure Weekly Cron Backup

Edit root crontab:

```bash
crontab -e
```

Add:

```cron
0 3 * * 0 /root/scripts/uptime-kuma-backup.sh >> /var/log/uptime-kuma-backup.log 2>&1
```

Meaning:

- minute `0`
- hour `3`
- Sunday (`0`)
- every week
- run backup script
- log output

Validation:

```bash
crontab -l
```

Look for:

```cron
0 3 * * 0 /root/scripts/uptime-kuma-backup.sh >> /var/log/uptime-kuma-backup.log 2>&1
```

This confirms:

- Uptime Kuma is automatically backed up weekly
- backups are committed to Git
- backups are replicated to GitHub
- recovery is automated

---

# 20. Configure Autoheal Git Backup

Autoheal changes rarely, but we still want:

- Git versioning
- private GitHub replication
- automated offsite backup
- recoverability

Because Autoheal configuration changes infrequently, we will back it up **monthly**.

---

## 20.1. Change Into the Autoheal Directory

```bash
cd /root/docker/autoheal
```

---

## 20.2. Initialize Repository

Initialize Git:

```bash
git init && git status
```

Look for:

```text
On branch master

No commits yet
```

---

## 20.3. Create `.gitignore`

Create the file:

```bash
nano .gitignore
```

Contents:

```gitignore
*.log
.DS_Store
Thumbs.db
```

Save and exit.

---

## 20.4. Create Initial Commit

IMPORTANT:

You **must create an initial commit before pushing to GitHub**.

Stage all files:

```bash
git add . && git status
```

Look for staged files including:

- `.gitignore`
- `docker-compose.yml`

Create the commit:

```bash
git commit -m "Initial Autoheal setup"
```

Validation:

Look for output similar to:

```text
[master (root-commit) abc1234] Initial Autoheal setup
```

Important:

- `(root-commit)` confirms the repository has an initial commit

---

## 20.5. Create GitHub Repository

In GitHub:

Create a NEW PRIVATE repository.

Recommended name:

```text
SelfHosted.VPS.Autoheal
```

Important:

- PRIVATE
- NO README
- NO `.gitignore`
- NO license

---

## 20.6. Add GitHub Remote

Back in the VPS, add the remote:

```bash
git remote add origin git@github.com:YOUR_USERNAME/SelfHosted.VPS.Autoheal.git
```

Example format:

```text
git@github.com:YOUR_USERNAME/SelfHosted.VPS.Autoheal.git && git remote -v
```

Validate:

```bash
git remote -v
```

Look for:

```text
origin  git@github.com:YOUR_USERNAME/SelfHosted.VPS.Autoheal.git (fetch)
origin  git@github.com:YOUR_USERNAME/SelfHosted.VPS.Autoheal.git (push)
```

---

## 20.7. Push to GitHub

Push the repository:

```bash
git push -u origin master
```

Validation:

Look for output similar to:

```text
[new branch]      master -> master
branch 'master' set up to track 'origin/master'
```

Then open the repository in GitHub.

Look for:

- `.gitignore`
- `docker-compose.yml`

This confirms:

- Git versioning works
- SSH authentication works
- offsite replication works
- Autoheal is recoverable

---

## 20.8. Create Backup Script

Create the backup script:

```bash
nano /root/scripts/autoheal-backup.sh
```

Contents:

```bash
#!/usr/bin/env bash

set -e

REPO_DIR="/root/docker/autoheal"

cd "$REPO_DIR"

git add .

if ! git diff --cached --quiet; then
    git commit -m "Automated backup $(date '+%Y-%m-%d %H:%M:%S')"
    git push
else
    echo "No changes to commit."
fi
```

Save and exit.

Make executable:

```bash
chmod +x /root/scripts/autoheal-backup.sh
```

Validation:

```bash
ls -lah /root/scripts
```

Look for:

```text
autoheal-backup.sh
```

with executable permissions (`x`).

---

## 20.9. Configure Monthly Cron Backup

Edit root crontab:

```bash
crontab -e
```

Add:

```cron
0 4 1 * * /root/scripts/autoheal-backup.sh >> /var/log/autoheal-backup.log 2>&1
```

Meaning:

- minute `0`
- hour `4`
- day `1`
- every month
- run backup script
- log output

Validation:

```bash
crontab -l
```

Look for:

```cron
0 4 1 * * /root/scripts/autoheal-backup.sh >> /var/log/autoheal-backup.log 2>&1
```

This confirms:

- Autoheal is automatically backed up monthly
- backups are committed to Git
- backups are replicated to GitHub
- recovery is automated

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
