# OpenClaw VPS Install Procedure (Tested 2026-02-25)

A battle-tested guide for deploying OpenClaw on a low-RAM KVM VPS with ChatGPT OAuth and Telegram. Based on a real deployment on a 4 GB RAM / Ubuntu 24.04 VPS.

---

## Prerequisites

- A KVM VPS (4 GB RAM recommended, 2 GB minimum with swap)
- Ubuntu 22.04 or 24.04 LTS
- Root SSH access
- An active ChatGPT subscription (Plus/Pro/Max)
- A Telegram bot token (from @BotFather)
- An OpenAI API key for embeddings (from platform.openai.com)
- A local machine with a browser (for the one-time OAuth sign-in)

---

## Step 1 — Initial VPS Setup

SSH in as root and update:

```bash
ssh root@YOUR_VPS_IP
apt update && apt upgrade -y
apt install -y curl git build-essential python3 ufw
```

### Swap (if RAM <= 4 GB)

Check existing swap first:

```bash
swapon --show
free -h
```

If total swap is under 4 GB, add more:

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

> **Why extra swap matters:** npm install for OpenClaw is memory-hungry. On a 4 GB VPS the OOM killer will terminate the SSH session mid-install without enough swap. We needed 6 GB total swap (2 GB partition + 4 GB file) to survive the install.

### Increase file descriptor limits

OpenClaw has ~700 npm dependencies. The default `nofile` limit (1024) causes TAR_ENTRY_ERROR failures during extraction:

```bash
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
```

### Create a dedicated user

```bash
adduser openclaw --disabled-password --gecos ""
usermod -aG sudo openclaw
echo "openclaw ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/openclaw
chmod 440 /etc/sudoers.d/openclaw
```

Copy SSH keys so you can SSH directly as `openclaw`:

```bash
mkdir -p /home/openclaw/.ssh
cp /root/.ssh/authorized_keys /home/openclaw/.ssh/
chown -R openclaw:openclaw /home/openclaw/.ssh
chmod 700 /home/openclaw/.ssh
chmod 600 /home/openclaw/.ssh/authorized_keys
```

Enable lingering (so systemd user services survive SSH disconnect):

```bash
loginctl enable-linger openclaw
```

---

## Step 2 — Install Node.js 22+

```bash
ssh openclaw@YOUR_VPS_IP
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version   # Should show v22.x.x
```

---

## Step 3 — Install OpenClaw (use tmux!)

> **Critical:** Do NOT install OpenClaw via a direct SSH command. The npm install is memory-intensive and will kill your SSH connection (OOM). Use tmux instead.

Create an install script:

```bash
cat > /tmp/install-openclaw.sh <<'SCRIPT'
#!/bin/bash
export PATH="$HOME/.npm-global/bin:$PATH"
ulimit -n 65536
npm install -g openclaw@latest > /tmp/openclaw-install.log 2>&1
echo "EXIT_CODE=$?" >> /tmp/openclaw-install.log
SCRIPT
chmod +x /tmp/install-openclaw.sh
```

Run it inside tmux:

```bash
tmux new-session -d -s install 'bash /tmp/install-openclaw.sh'
```

Monitor progress (safe to disconnect and reconnect):

```bash
tail -f /tmp/openclaw-install.log
```

Wait until you see `added XXX packages` and `DONE`. Then verify:

```bash
export PATH="$HOME/.npm-global/bin:$PATH"
openclaw --version
```

> Add the PATH to your shell profile so it persists:
> ```bash
> echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
> ```

---

## Step 4 — Create a Telegram Bot

1. Open Telegram, search for **@BotFather**
2. Send `/newbot`
3. Pick a display name and a username ending in `_bot`
4. Save the **bot token** BotFather gives you
5. (Optional) Send `/setprivacy` → select your bot → "Disable"

---

## Step 5 — SSH Tunnel for OAuth

The OAuth callback redirects to `http://localhost:1455/auth/callback`. On a headless VPS there's no browser, so you forward port 1455 from your local machine.

From your **local machine**:

```bash
ssh -L 1455:localhost:1455 -L 18789:localhost:18789 openclaw@YOUR_VPS_IP
```

Keep this terminal open for the onboarding process.

---

## Step 6 — Run Onboarding Wizard (interactive, via tmux)

> **Key lesson:** The `--non-interactive` flag does NOT work with OAuth (`openai-codex`). You must run the wizard interactively.

Since the wizard is a TUI app that needs a terminal, run it in tmux:

```bash
cat > /tmp/run-onboard.sh <<'SCRIPT'
#!/bin/bash
export PATH="$HOME/.npm-global/bin:$PATH"
openclaw onboard \
  --accept-risk \
  --auth-choice openai-codex \
  --install-daemon \
  --skip-channels \
  --gateway-bind loopback \
  --gateway-port 18789 \
  --gateway-auth token
SCRIPT
chmod +x /tmp/run-onboard.sh
tmux new-session -d -s onboard 'bash /tmp/run-onboard.sh'
```

Attach to watch the wizard:

```bash
tmux attach -t onboard
```

### Walking through the wizard prompts

1. **Onboarding mode** → Select "QuickStart", press Enter
2. **OAuth URL** → The wizard prints a URL like:
   ```
   https://auth.openai.com/oauth/authorize?response_type=code&client_id=...
   ```
   Copy this URL and open it in your **local browser** (works thanks to SSH tunnel).
3. **Sign in** to your ChatGPT account (Google/email/etc.)
4. After sign-in, the browser redirects to `http://localhost:1455/auth/callback?code=...&state=...`
5. **Copy the full redirect URL** from your browser's address bar
6. **Paste it** into the wizard's "Paste the redirect URL" prompt
7. **Skills** → Select "No" (configure later)
8. **Hooks** → Select "Skip for now"

The wizard saves config to `~/.openclaw/openclaw.json`.

> **Note:** If systemd service install fails during onboarding (common when running via `su -` without a full user session), install it manually afterward:
> ```bash
> export XDG_RUNTIME_DIR=/run/user/$(id -u)
> openclaw gateway install
> ```

---

## Step 7 — Enable Telegram Plugin and Add Channel

The Telegram plugin is **disabled by default**. You must enable it before adding the channel:

```bash
openclaw plugins enable telegram
openclaw channels add --channel telegram --token "YOUR_TELEGRAM_BOT_TOKEN"
```

Verify:

```bash
openclaw channels list
```

---

## Step 8 — Embeddings API Key

ChatGPT Codex subscriptions do NOT include embeddings. You need a separate OpenAI API key (costs pennies):

1. Go to [platform.openai.com](https://platform.openai.com) → API Keys → Create new key
2. Add it to the environment file:

```bash
echo 'OPENAI_API_KEY=sk-your-embeddings-key' >> ~/.openclaw/.env
chmod 600 ~/.openclaw/.env
```

> **FAQ:** Logging into platform.openai.com does NOT affect the ChatGPT OAuth on the VPS. They are completely separate auth systems.

Also disable unnecessary mDNS:

```bash
echo 'OPENCLAW_DISABLE_BONJOUR=1' >> ~/.openclaw/.env
```

---

## Step 9 — Security Hardening

### Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw limit 22/tcp
sudo ufw enable
```

Do NOT open port 18789 — the gateway binds to loopback only.

### fail2ban

```bash
sudo apt install -y fail2ban
```

### Verify gateway config

Your `~/.openclaw/openclaw.json` should have:

```json
{
  "gateway": {
    "bind": "loopback",
    "port": 18789,
    "auth": {
      "mode": "token"
    }
  }
}
```

---

## Step 10 — Start and Verify

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
openclaw gateway start
openclaw gateway status    # Should show "running"
openclaw models status     # Should show OAuth valid
openclaw doctor            # Run diagnostics
openclaw doctor --fix      # Auto-fix issues
```

---

## Step 11 — Telegram Pairing

1. Open Telegram, find your bot
2. Send "Hello"
3. The bot replies with a **pairing code**
4. Approve it on the VPS:

```bash
openclaw pairing approve telegram YOUR_PAIRING_CODE
```

5. Send another message — the bot should now respond

---

## Troubleshooting

### SSH connection drops during npm install
Use tmux (see Step 3). The OOM killer targets SSH sessions. tmux survives.

### TAR_ENTRY_ERROR during npm install
Increase file descriptor limits (see Step 1). The default 1024 is too low for 700+ packages.

### "OAuth requires interactive mode"
You cannot use `--non-interactive` with `--auth-choice openai-codex`. Run the wizard interactively inside tmux.

### "Unknown channel: telegram"
The Telegram plugin is disabled by default. Run `openclaw plugins enable telegram` first.

### Systemd service install fails during onboarding
Run separately after onboarding:
```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
openclaw gateway install
```

### "Failed to connect to bus" errors
Add to `~/.bashrc`:
```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
```

### OAuth token expired
Reconnect with port forwarding and re-authenticate:
```bash
# From local machine
ssh -L 1455:localhost:1455 openclaw@YOUR_VPS_IP

# On VPS
openclaw models auth login --provider openai-codex
# Open the printed URL in local browser, sign in again
openclaw gateway restart
```

---

## Useful Commands

```bash
openclaw gateway status          # Gateway status
openclaw gateway restart         # Restart gateway
openclaw logs --follow           # Tail logs
openclaw models status           # Auth and usage
openclaw doctor                  # Diagnose issues
openclaw channels list           # Show channels
openclaw plugins list            # Show plugins
openclaw security audit --deep   # Security check
```

---

## Architecture Summary

```
┌──────────────┐   HTTPS polling   ┌──────────────────────┐
│   Telegram   │ ◄───────────────► │  OpenClaw Gateway    │
│   (users)    │                   │  127.0.0.1:18789     │
└──────────────┘                   │                      │
                                   │  ┌─ Telegram plugin  │
                                   │  ├─ OAuth (Codex)    │
                                   │  └─ Embeddings (API) │
                                   └──────────────────────┘
                                          │
                                          ▼ OAuth
                                   ┌──────────────────────┐
                                   │  ChatGPT / Codex API │
                                   │  (your subscription) │
                                   └──────────────────────┘
```

No inbound ports needed besides SSH. Telegram uses outbound HTTPS long-polling. The gateway only listens on localhost.
