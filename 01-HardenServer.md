# 01. SovrIT Server Hardening (Part 1 — System Preparation)

**IMPORTANT**

This runbook assumes:
- a fresh installation of Ubuntu Server 24.04
- you are logged in as `root`
- setup and configuration will be performed as `root`
- root SSH access will ultimately be restricted to **SSH key authentication only**
- password authentication will later be disabled

During setup, we intentionally prioritize:
- reproducibility
- recoverability
- deterministic validation
- idempotence
- avoiding accidental lockout

---

## 01.1. Update Ubuntu

Update package indexes and install upgrades:

```bash
apt update && apt upgrade -y
```

Validation:

Look for output similar to:

```text
0 upgraded, 0 newly installed, 0 to remove
```

(or confirmation that upgrades completed successfully).

If kernel or system packages were upgraded:

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

Install baseline utilities used throughout SovrIT:

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
logrotate
```

Why:

| Package | Purpose |
|---|---|
| curl / wget | downloads, API testing |
| git | GitHub backups |
| nano / vim | editing |
| htop | system monitoring |
| tree | filesystem visualization |
| jq | JSON parsing |
| dnsutils | DNS troubleshooting (`dig`) |
| net-tools | network diagnostics |
| openssl | TLS diagnostics |
| ufw | firewall |
| logrotate | log management |

Validation:

```bash
git --version && \
curl --version && \
nano --version && \
ufw version
```

Look for:

- version output
- no errors

---

## 01.3. Configure Hostname and Timezone

Set hostname.

Replace:

```text
YOUR_HOSTNAME
```

with your preferred hostname.

Example:

```text
vps-primary
```

Set hostname and immediately validate:

```bash
hostnamectl set-hostname YOUR_HOSTNAME && \
hostname && \
hostnamectl
```

Validation:

Look for:

```text
YOUR_HOSTNAME
```

in both:

- `hostname`
- `hostnamectl`

Example:

```text
vps-primary
```

You should also see:

```text
Static hostname: YOUR_HOSTNAME
```

> **IMPORTANT**
>
> Your terminal prompt may continue showing the old hostname until the shell session refreshes.
>
> This is normal and does **not** indicate failure.

If the prompt still shows the old hostname:

```bash
exec bash
```

Expected:

Your prompt updates from:

```text
root@old-hostname:~#
```

to:

```text
root@YOUR_HOSTNAME:~#
```

If needed, disconnect and reconnect your SSH session.

---

List available timezones:

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

Set timezone and validate:

```bash
timedatectl set-timezone YOUR_TIMEZONE && \
timedatectl
```

Example:

```bash
timedatectl set-timezone America/New_York && \
timedatectl
```

Validation:

Look for:

```text
Time zone: YOUR_TIMEZONE
System clock synchronized: yes
```

Example:

```text
Time zone: America/New_York
System clock synchronized: yes
```

---

## 01.4. Generate SSH Key Pair (Local Computer)

**IMPORTANT**

Perform this step on **your local computer**, not the VPS.

SovrIT standardizes on SSH key authentication.

Generate an SSH keypair:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/sovrit_root
```

When prompted for a passphrase:

```text
Enter passphrase
```

You may:

- use a passphrase (**recommended**)
- leave blank for convenience

This creates:

```text
~/.ssh/sovrit_root
~/.ssh/sovrit_root.pub
```

Validation:

```bash
ls -lah ~/.ssh && cat ~/.ssh/sovrit_root.pub
```

Look for:

```text
sovrit_root
sovrit_root.pub
```

and output similar to:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
```

Keep this terminal open.

You will install this key onto the VPS in Part 2.

# 01. SovrIT Server Hardening (Part 2 — SSH Hardening)

**IMPORTANT**

Do **not** close your current SSH session during this section.

We will:

- install SSH keys
- move SSH to port `2222`
- disable password authentication
- keep root login enabled via SSH key only
- validate login before proceeding

 Only close older sessions **after validation succeeds**.

---

## 01.5. Define Persistent SSH Variables

> **IMPORTANT**
>
> To reduce copy/paste errors, improve reproducibility, and survive new terminal sessions, SovrIT defines reusable SSH variables in:
>
> ```text
> ~/.sovrit_env
> ```
>
> These variables are automatically loaded into future terminals.

**Run on: LOCAL COMPUTER**

Create persistent SovrIT environment variables:

```bash
cat > ~/.sovrit_env <<'EOF'
export SERVER_IP="YOUR_SERVER_IP"
export SSH_PORT=2222
export SSH_KEY=~/.ssh/vps_sandbox
EOF
```

Example:

```bash
cat > ~/.sovrit_env <<'EOF'
export SERVER_IP="203.0.113.22"
export SSH_PORT=8288
export SSH_KEY=~/.ssh/vps_sandbox
EOF
```

---

Automatically load variables in future terminals:

```bash
grep -q "source ~/.sovrit_env" ~/.bashrc || \
echo 'source ~/.sovrit_env' >> ~/.bashrc
```

Load variables immediately in the current shell and validate:

```bash
source ~/.sovrit_env && \
echo $SERVER_IP && \
echo $SSH_PORT && \
echo $SSH_KEY
```

Validation:

Look for output similar to:

```text
107.172.201.9
8288
~/.ssh/vps_sandbox
```

(or your custom values).

Validation (new terminal test):

Open a **new terminal window** and run:

```bash
echo $SERVER_IP && \
echo $SSH_PORT && \
echo $SSH_KEY
```

Expected:

Variables are automatically populated.

This confirms:

- variables persist across terminals
- SSH commands remain reproducible
- interruption/restart is safe

---

## 01.6. Install SSH Public Key on VPS

> **IMPORTANT**
>
> This step uses:
>
> - your **LOCAL COMPUTER**
> - the **VPS**
>
> Pay attention to execution context.

### 01.6.1. Display Public Key (LOCAL COMPUTER)

**Run on: LOCAL COMPUTER**

Display your public key:

```bash
cat ${SSH_KEY}.pub
```

Validation:

Look for output similar to:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... user@computer
```

Copy the **entire line** to your clipboard.

Do not close this terminal.

---

### 01.6.2. Create SSH Directory (VPS)

**Run on: VPS**

Create the SSH directory and validate:

```bash
mkdir -p ~/.ssh && \
chmod 700 ~/.ssh && \
ls -lah ~
```

Validation:

Look for:

```text
.ssh
```

in the home directory listing.

---

### 01.6.3. Install Public Key (VPS)

**Run on: VPS**

Open:

```bash
nano ~/.ssh/authorized_keys
```

Paste the SSH public key copied earlier.

Save and exit.

---

### 01.6.4. Apply Permissions and Validate (VPS)

**Run on: VPS**

Apply permissions and validate:

```bash
chmod 600 ~/.ssh/authorized_keys && \
ls -lah ~/.ssh
```

Validation:

Look for:

```text
drwx------ .ssh
-rw------- authorized_keys
```

This confirms:

- SSH directory exists
- permissions are correct
- public key installed
- VPS ready for SSH key authentication

---

## 01.7. Validate SSH Key Authentication

> **IMPORTANT**
>
> Open a **new terminal window**.
>
> Do **not** close your existing VPS session.

**Run on: LOCAL COMPUTER**

Confirm variables loaded automatically:

```bash
echo $SERVER_IP && \
echo $SSH_PORT && \
echo $SSH_KEY
```

Validation:

Look for output similar to:

```text
203.0.113.22
8288
~/.ssh/vps_sandbox
```

(or your custom values).

If variables are empty:

```bash
source ~/.sovrit_env
```

Then repeat validation.

---

Test SSH key authentication.

```bash
ssh -i $SSH_KEY -p $SSH_PORT root@$SERVER_IP || \
ssh -i $SSH_KEY root@$SERVER_IP
```

Validation:

You should successfully log in **without using a password**.

Expected:

```text
root@your-hostname:~#
```

Connection may occur via:

```text
$SSH_PORT
```

or:

```text
22
```

depending on system state.

If successful:

Leave both SSH sessions open.

If unsuccessful:

Stop and fix SSH key authentication before continuing.

Do **not** proceed until login succeeds.
---

## 01.8. Configure SSH Hardening

> **IMPORTANT**
>
> Complete the remainder of SSH hardening in **one continuous session**.
>
> Do **not** walk away, disconnect, suspend your laptop, or allow SSH to idle-timeout until:
>
> ```text
> 01.11 Final SSH Validation
> ```
>
> succeeds.
>
> During this section:
>
> - SSH configuration changes
> - SSH port changes
> - SSH socket activation is disabled
> - SSH service restarts
>
> Interrupting the process may temporarily lock you out.
>
> Keep at least one working SSH session open at all times.

**Run on: VPS**

Create a backup and validate:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak && \
ls -lah /etc/ssh/sshd_config*
```

Validation:

Look for:

```text
/etc/ssh/sshd_config
/etc/ssh/sshd_config.bak
```

---

Apply hardened SSH configuration.

> **IMPORTANT**
>
> Replace:
>
> ```text
> 2222
> ```
>
> below with your chosen SSH port if different.

**Run on: VPS**

Copy/paste:

```bash
sed -i 's/^#\?Port .*/Port 2222/' /etc/ssh/sshd_config

sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config

sed -i 's/^#\?PubkeyAuthentication .*/PubkeyAuthentication yes/' /etc/ssh/sshd_config

sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config

sed -i 's/^#\?KbdInteractiveAuthentication .*/KbdInteractiveAuthentication no/' /etc/ssh/sshd_config

grep -q '^ChallengeResponseAuthentication' /etc/ssh/sshd_config \
&& sed -i 's/^#\?ChallengeResponseAuthentication .*/ChallengeResponseAuthentication no/' /etc/ssh/sshd_config \
|| echo 'ChallengeResponseAuthentication no' >> /etc/ssh/sshd_config

sed -i 's/^#\?UsePAM .*/UsePAM yes/' /etc/ssh/sshd_config
```

Why this works:

- handles commented/uncommented directives
- overwrites incorrect values
- safely adds missing directives
- idempotent
- deterministic
- safe to rerun
- does not depend on hidden shell variables

Validation:

```bash
grep -E '^(Port|PermitRootLogin|PubkeyAuthentication|PasswordAuthentication|KbdInteractiveAuthentication|ChallengeResponseAuthentication|UsePAM)' \
/etc/ssh/sshd_config
```

Look for:

```text
Port 2222
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
```

(Substitute your custom port if applicable.)

---

## 01.9. Validate and Restart SSH (Ubuntu 24.04 Safe)

> **IMPORTANT**
>
> Keep your existing SSH session open.
>
> Do **not** disconnect until:
>
> ```text
> 01.11 Final SSH Validation
> ```
>
> succeeds.

> **IMPORTANT**
>
> Ubuntu Server 24.04 cloud images commonly enable:
>
> ```text
> ssh.socket
> ```
>
> which forces SSH to listen on port `22`
> regardless of:
>
> ```text
> /etc/ssh/sshd_config
> ```
>
> We explicitly disable socket activation so SSH obeys the configured port.

**Run on: VPS**

Validate syntax:

```bash
sshd -t && echo "sshd config valid"
```

Expected:

```text
sshd config valid
```

---

Disable socket activation (safe to rerun):

```bash
systemctl disable --now ssh.socket || true

rm -f /etc/systemd/system/ssh.service.d/00-socket.conf

systemctl daemon-reload
```

Validation:

```bash
systemctl status ssh.socket --no-pager
```

Look for:

```text
inactive (dead)
```

or:

```text
disabled
```

---

Restart SSH and validate runtime configuration.

> Replace:
>
> ```text
> 2222
> ```
>
> with your chosen SSH port if different.

```bash
systemctl restart ssh && \
sleep 2 && \
sshd -T | grep -i port && \
ss -tulpn | grep 2222
```

Validation:

Look for:

```text
port 2222
```

and:

```text
0.0.0.0:2222
[::]:2222
```

(Substitute your custom port if applicable.)

This confirms:

- SSH config parsed correctly
- socket activation disabled
- SSH runtime matches intended configuration
- SSH listening on expected port

---

## 01.10. Validate Login on Hardened Port

> **IMPORTANT**
>
> Open a **new terminal window**.
>
> Do **not** close existing sessions.

**Run on: LOCAL COMPUTER**

Confirm variables loaded:

```bash
echo $SERVER_IP && \
echo $SSH_PORT && \
echo $SSH_KEY
```

If variables are empty:

```bash
source ~/.sovrit_env
```

---

Test hardened SSH login:

```bash
ssh -i $SSH_KEY -p $SSH_PORT root@$SERVER_IP
```

Validation:

Successful login:

```text
root@your-hostname:~#
```

If login fails:

- do **not** close existing sessions
- validate:
  - SSH port
  - sshd status
  - firewall rules (later)

---

## 01.11. Final SSH Validation

**Run on: LOCAL COMPUTER**

Test login without explicitly supplying a key:

```bash
ssh -p $SSH_PORT root@$SERVER_IP
```

Expected:

Either:

```text
successful login
```

(if SSH agent loaded the key)

OR:

```text
Permission denied (publickey)
```

Both outcomes are acceptable.

Now explicitly test key login again:

```bash
ssh -i $SSH_KEY -p $SSH_PORT root@$SERVER_IP
```

Expected:

Successful login.

This confirms:

- SSH keys work
- password authentication disabled
- root access is key-only
- socket activation disabled
- SSH moved off port `22`
- SSH hardening succeeded

Only after successful validation should older sessions be closed.

---

# 01. SovrIT Server Hardening (Part 3 — Firewall / UFW)

**IMPORTANT**

We will configure a firewall **without locking ourselves out**.

Do **not** enable UFW until required rules are in place.

SSH access on:

```text
2222
```

must already be working before continuing.

---

## 01.11. Verify SSH Connectivity Before Firewall Changes

Confirm SSH access works on the hardened port.

From your local computer:

```bash
ssh -i ~/.ssh/sovrit_root -p 2222 root@YOUR_SERVER_IP
```

Validation:

You should successfully log in.

Expected:

```text
root@your-hostname:~#
```

Do **not** continue if SSH access is unreliable.

---

## 01.12. Verify UFW Installation and Current Status

Confirm UFW is installed and inspect status:

```bash
ufw version && ufw status verbose
```

Validation:

Look for:

```text
Status: inactive
```

This is normal on a fresh installation.

---

## 01.13. Configure Firewall Defaults

Configure secure defaults:

```bash
ufw default deny incoming && \
ufw default allow outgoing && \
ufw status verbose
```

Validation:

Look for:

```text
Default: deny (incoming)
Default: allow (outgoing)
```

---

## 01.14. Add Required Firewall Rules (Idempotent)

Add SSH, HTTP, and HTTPS rules.

These commands are written to avoid duplicate firewall rules if re-run.

```bash
ufw status | grep -q "2222/tcp" || ufw allow 2222/tcp comment 'SSH'

ufw status | grep -q "80/tcp" || ufw allow 80/tcp comment 'HTTP'

ufw status | grep -q "443/tcp" || ufw allow 443/tcp comment 'HTTPS'

ufw status numbered
```

**Reminder**

If you chose a different SSH port earlier, substitute it consistently.

Why:

| Port | Purpose |
|---|---|
| 2222 | SSH administration |
| 80 | HTTP (ACME + redirect) |
| 443 | HTTPS services |

Validation:

Look for rules similar to:

```text
2222/tcp ALLOW IN
80/tcp   ALLOW IN
443/tcp  ALLOW IN
```

Only one entry per rule should exist.

---

## 01.15. Enable Firewall Safely

**IMPORTANT**

Before enabling UFW:

- ensure you still have an active SSH session
- confirm SSH already works on port `2222`

Enable firewall:

```bash
ufw enable && ufw status verbose
```

When prompted:

```text
Proceed with operation (y|n)?
```

Answer:

```text
y
```

Validation:

Look for:

```text
Status: active
```

and rules for:

```text
2222/tcp
80/tcp
443/tcp
```

---

## 01.16. Validate SSH Through Firewall

**IMPORTANT**

Open a **new terminal window**.

Do **not** close your existing sessions.

Test SSH access:

```bash
ssh -i ~/.ssh/sovrit_root -p 2222 root@YOUR_SERVER_IP
```

Validation:

Successful login.

Expected:

```text
root@your-hostname:~#
```

If login fails:

- do **not** close existing sessions
- inspect firewall rules
- correct configuration before proceeding

---

## 01.17. Optional Firewall Diagnostics

Inspect firewall rules:

```bash
ufw status numbered
```

Inspect listening ports:

```bash
ss -tulpn
```

Look for:

- SSH listening on `2222`
- only expected services exposed

This confirms:

- firewall is active
- SSH administration remains accessible
- HTTP/HTTPS are allowed
- inbound traffic is denied by default
- the server is ready for internet-facing services

# 01. SovrIT Server Hardening (Part 4 — CrowdSec)

**IMPORTANT**

SovrIT standardizes on:

```text
CrowdSec
```

instead of:

```text
Fail2ban
```

CrowdSec provides:

- behavioral attack detection
- SSH protection
- web attack protection
- shared threat intelligence
- automatic blocking through a firewall bouncer

Detection without enforcement is insufficient.

We will install:

- CrowdSec engine
- nftables firewall bouncer

---

## 01.18. Install CrowdSec Repository

Install prerequisites and CrowdSec repository:

```bash
apt install -y curl gnupg apt-transport-https && \
mkdir -p /usr/share/keyrings && \
curl -fsSL https://packagecloud.io/crowdsec/crowdsec/gpgkey | \
gpg --dearmor -o /usr/share/keyrings/crowdsec.gpg
```

Add CrowdSec repository:

```bash
cat > /etc/apt/sources.list.d/crowdsec.list <<'EOF'
deb [signed-by=/usr/share/keyrings/crowdsec.gpg] https://packagecloud.io/crowdsec/crowdsec/ubuntu/ noble main
EOF
```

Update package indexes:

```bash
apt update
```

Validation:

Look for:

```text
crowdsec
```

repository entries during update.

No repository errors should appear.

---

## 01.19. Install CrowdSec + Firewall Bouncer

Install CrowdSec and the firewall bouncer:

```bash
apt install -y crowdsec crowdsec-firewall-bouncer-nftables
```

Why:

| Component | Purpose |
|---|---|
| crowdsec | attack detection |
| firewall bouncer | automatic blocking |

Without a bouncer:

```text
detections occur
but malicious traffic is not blocked
```

Validate services:

```bash
systemctl status crowdsec --no-pager && \
systemctl status crowdsec-firewall-bouncer --no-pager
```

Validation:

Look for:

```text
active (running)
```

for both services.

---

## 01.20. Install Core CrowdSec Collections

Install SSH, Linux, firewall, and web protections:

```bash
cscli collections install crowdsecurity/sshd && \
cscli collections install crowdsecurity/linux && \
cscli collections install crowdsecurity/ufw && \
cscli collections install crowdsecurity/nginx && \
systemctl restart crowdsec
```

Why nginx?

Caddy logs are compatible with CrowdSec’s nginx parsers.

This later protects:

- Caddy
- reverse-proxied applications
- internet-facing services

Validation:

```bash
cscli collections list
```

Look for:

```text
crowdsecurity/sshd
crowdsecurity/linux
crowdsecurity/ufw
crowdsecurity/nginx
```

with status:

```text
enabled
```

Re-running this step is safe.

Already-installed collections will simply report:

```text
already installed
```

---

## 01.21. Validate CrowdSec Health

Check CrowdSec metrics:

```bash
cscli metrics
```

Validation:

Look for sections such as:

```text
Acquisition Metrics
Parser Metrics
```

No major errors should appear.

Validate service health:

```bash
systemctl status crowdsec --no-pager && \
systemctl status crowdsec-firewall-bouncer --no-pager
```

Look for:

```text
active (running)
```

for both services.

---

## 01.22. Validate Firewall Enforcement

Check active CrowdSec decisions:

```bash
cscli decisions list
```

Initially this may show:

```text
No active decisions
```

This is normal.

Inspect nftables rules:

```bash
nft list ruleset | grep -i crowdsec
```

Validation:

Look for CrowdSec-related entries such as:

```text
crowdsec
```

or:

```text
crowdsec-chain
```

This confirms:

- CrowdSec is installed
- attack detection is active
- firewall enforcement is active
- SSH protection is enabled
- web protections are ready for future services
