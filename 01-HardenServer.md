# 01. SovrIT Server Hardening (Part 1 — System Preparation)

> **IMPORTANT**
>
> This runbook assumes:
>
> - a fresh installation of Ubuntu Server 24.04
> - you are logged in as `root`
> - setup and configuration will be performed as `root`
> - root SSH access will ultimately be restricted to **SSH key authentication only**
> - password authentication will be disabled
>
> During setup, we intentionally prioritize:
>
> - reproducibility
> - recoverability
> - deterministic validation
> - avoiding accidental lockout

---

## 01.1. Update Ubuntu

Update package indexes:

```bash
apt update
```

Install available updates:

```bash
apt upgrade -y
```

Validation:

Look for:

```text
0 upgraded, 0 newly installed, 0 to remove
```

(or similar output indicating updates completed successfully).

Optional reboot if kernel/system packages were upgraded:

```bash
reboot
```

Reconnect to the VPS after reboot.

Validation:

Reconnect successfully as:

```text
root
```

---

## 01.2. Install Baseline Packages

Install baseline utilities used throughout SovrIT runbooks:

```bash
apt install -y \
curl \
wget \
git \
nano \
vim \
htop \
tree \
unzip \
zip \
jq \
ca-certificates \
gnupg \
lsb-release \
software-properties-common \
apt-transport-https \
net-tools \
dnsutils \
openssl \
ufw \
fail2ban \
logrotate
```

Why:

| Package | Purpose |
|---|---|
| curl / wget | downloads, testing APIs |
| git | backups and repo replication |
| nano / vim | text editing |
| htop | system monitoring |
| tree | directory visualization |
| jq | JSON parsing |
| dnsutils | DNS troubleshooting (`dig`) |
| net-tools | network diagnostics |
| openssl | TLS diagnostics |
| ufw | firewall |
| logrotate | log management |

Validation:

```bash
git --version
curl --version
nano --version
ufw version
```

Look for:
- version output
- no errors

IMPORTANT:

`fail2ban` is installed temporarily for compatibility and transition safety.

SovrIT will standardize on:

```text
CrowdSec
```

later in this runbook.

---

## 01.3. Configure Hostname and Timezone

Set hostname.

Example format:

```text
vps-primary
```

Replace with your preferred hostname:

```bash
hostnamectl set-hostname YOUR_HOSTNAME
```

Example:

```bash
hostnamectl set-hostname vps-primary
```

Validation:

```bash
hostnamectl
```

Look for:

```text
Static hostname: YOUR_HOSTNAME
```

---

Set timezone.

View available timezones:

```bash
timedatectl list-timezones
```

Common examples:

```text
America/New_York
America/Chicago
America/Denver
America/Los_Angeles
UTC
```

Set timezone:

```bash
timedatectl set-timezone YOUR_TIMEZONE
```

Example:

```bash
timedatectl set-timezone America/New_York
```

Validation:

```bash
timedatectl
```

Look for:

```text
Time zone: America/New_York
System clock synchronized: yes
```

---

## 01.4. Generate SSH Key Pair (Local Machine)

IMPORTANT:

Perform this step on **your local computer**, not the VPS.

SovrIT standardizes on SSH key authentication.

Password login will later be disabled.

Generate a new SSH keypair:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/sovrit_root
```

When prompted:

```text
Enter passphrase
```

You may:

- use a passphrase (recommended)
- leave blank for convenience

This creates:

```text
~/.ssh/sovrit_root
~/.ssh/sovrit_root.pub
```

Validation:

```bash
ls -lah ~/.ssh
```

Look for:

```text
sovrit_root
sovrit_root.pub
```

Display public key:

```bash
cat ~/.ssh/sovrit_root.pub
```

Look for output similar to:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
```

Keep this terminal open.

You will install this key onto the VPS in the next step.
