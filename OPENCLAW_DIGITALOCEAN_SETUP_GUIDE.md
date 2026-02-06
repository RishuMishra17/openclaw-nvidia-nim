# 🦞 OpenClaw on DigitalOcean with NVIDIA NIM Kimi K2.5

> **Complete Setup Guide** — NVIDIA NIM is a **built-in provider** in the setup wizard.
> No manual JSON editing required — just select it during onboarding.
> 
> **Cost:** $6/month (or $4/mo with reserved pricing)

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Create a Droplet](#1-create-a-droplet)
3. [Connect via SSH](#2-connect-via-ssh)
4. [Install OpenClaw (Custom Build)](#3-install-openclaw-custom-build)
5. [Run Onboarding with NVIDIA NIM](#4-run-onboarding-with-nvidia-nim)
6. [Non-Interactive Setup (Alternative)](#5-non-interactive-setup-alternative)
7. [Verify the Gateway](#6-verify-the-gateway)
8. [Access the Dashboard](#7-access-the-dashboard)
9. [Connect Channels](#8-connect-channels)
10. [Manual Config (Advanced)](#manual-config-advanced)
11. [Optimizations for 1GB RAM](#optimizations-for-1gb-ram)
12. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- **DigitalOcean account** — [Signup with $200 free credit](https://m.do.co/c/signup)
- **SSH key pair** (recommended) or password auth
- **NVIDIA NIM API Key** — Get from [NVIDIA Build](https://build.nvidia.com/settings/api-keys) (starts with `nvapi-`)
- **Node.js 22+**
- ~20 minutes

---

## 1) Create a Droplet

1. Log into [DigitalOcean](https://cloud.digitalocean.com/)
2. Click **Create → Droplets**
3. Choose:
   - **Region:** Closest to you (e.g., BLR1 for India)
   - **Image:** Ubuntu 24.04 LTS
   - **Size:** Basic → Regular → **$6/mo** (1 vCPU, 1GB RAM, 25GB SSD)
   - **Authentication:** SSH key (recommended)
4. Click **Create Droplet**
5. Note the IP address

---

## 2) Connect via SSH

```bash
ssh root@YOUR_DROPLET_IP
```

If using an SSH key file:
```bash
ssh -i /path/to/your/key root@YOUR_DROPLET_IP
```

---

## 3) Install OpenClaw (Custom Build)

This guide uses our **custom build** with built-in NVIDIA NIM provider support.

```bash
# Update system
apt update && apt upgrade -y

# Install Node.js 22 + pnpm
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs git
npm install -g pnpm

# Clone our custom fork with NVIDIA NIM support
git clone https://github.com/RishuMishra17/openclaw-nvidia-nim.git /opt/openclaw
cd /opt/openclaw

# Install dependencies and build
pnpm install
pnpm build

# Link the CLI globally
npm link

# Verify installation
openclaw --version
```

> **Why a custom build?** NVIDIA NIM is now a **first-class provider** in the setup wizard.
> You select it during onboarding — no manual JSON editing needed.

### Alternative: Docker Installation

If you prefer Docker:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Clone our custom fork
git clone https://github.com/RishuMishra17/openclaw-nvidia-nim.git
cd openclaw-nvidia-nim

# Run Docker setup
./docker-setup.sh
```

> **Note:** The Docker image runs as `node` (uid 1000). If you see permission errors, fix with:
> ```bash
> sudo chown -R 1000:1000 ~/.openclaw
> ```

---

## 4) Run Onboarding with NVIDIA NIM

```bash
openclaw onboard --install-daemon
```

The wizard walks you through each step interactively:

### Step-by-Step Wizard Flow

1. **Select provider** — When prompted for auth, select **"NVIDIA NIM"** from the provider groups:
   ```
   ◆ Select authentication method
   │
   │  ▸ NVIDIA NIM
   │    └ NVIDIA NIM API key  (Kimi K2.5 via NVIDIA NIM (nvapi-...))
   │
   │  ▸ Moonshot AI (Kimi K2.5)
   │  ▸ OpenAI
   │  ▸ Anthropic
   │  ... (other providers)
   ```

2. **Enter API key** — Paste your `nvapi-...` key from [NVIDIA Build](https://build.nvidia.com/settings/api-keys):
   ```
   ◆ NVIDIA NIM
   │  NVIDIA NIM provides access to Kimi K2.5 and other models
   │  via an OpenAI-compatible API.
   │  Get your API key at: https://build.nvidia.com/
   │  API keys start with 'nvapi-'.
   │
   ◆ Enter NVIDIA NIM API key
   │  nvapi-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

3. **Confirm default model** — The wizard sets `nvidia-nim/moonshotai/kimi-k2.5` (Kimi K2.5) as default.

4. **Channel setup** — Connect Telegram, WhatsApp, Discord, etc.

5. **Gateway config** — Token auto-generated, daemon installed via systemd.

> **That's it!** No manual JSON editing needed. The wizard handles all provider config,
> credential storage, and model selection automatically.

---

## 5) Non-Interactive Setup (Alternative)

For automated deployments or scripts, use the non-interactive flag:

```bash
# Set API key as env var first
export NVIDIA_API_KEY="nvapi-your-key-here"

# Run fully non-interactive onboarding
openclaw onboard \
  --non-interactive \
  --accept-risk \
  --auth-choice nvidia-nim-api-key \
  --install-daemon
```

Or pass the key inline:

```bash
openclaw onboard \
  --non-interactive \
  --accept-risk \
  --auth-choice apiKey \
  --token-provider nvidia-nim \
  --token "nvapi-your-key-here" \
  --install-daemon
```

### Auto-Discovery via Environment Variable

If `NVIDIA_API_KEY` is set in the environment (e.g., in `~/.profile` or `~/.bashrc`),
OpenClaw will **automatically discover** the NVIDIA NIM provider and register the
Kimi K2.5 model without any wizard interaction:

```bash
# Add to ~/.profile for persistence
echo 'export NVIDIA_API_KEY="nvapi-your-key-here"' >> ~/.profile
source ~/.profile

# OpenClaw auto-detects the provider
openclaw models list
# Output includes: nvidia-nim/moonshotai/kimi-k2.5  Kimi K2.5 (NVIDIA NIM)
```

---

## 6) Verify the Gateway

```bash
# Check status
openclaw status

# Check systemd service
systemctl --user status openclaw-gateway.service

# View logs
journalctl --user -u openclaw-gateway.service -f

# Run diagnostics
openclaw doctor --non-interactive

# Verify NVIDIA NIM model is registered
openclaw models list | grep nvidia-nim
# Expected: nvidia-nim/moonshotai/kimi-k2.5  Kimi K2.5 (NVIDIA NIM)

# Test the model
openclaw agent --message "Hello, what model are you?" --thinking low
```

---

## 7) Access the Dashboard

The gateway binds to loopback by default. To access the Control UI:

### Option A: SSH Tunnel (Recommended)

From your **local machine**:
```bash
ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
```

Then open: **http://localhost:18789**

### Option B: Tailscale Serve (HTTPS)

```bash
# On the droplet
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Configure Gateway to use Tailscale Serve
openclaw config set gateway.tailscale.mode serve
openclaw gateway restart
```

Open: `https://<magicdns>/`

---

## 8) Connect Channels

### Telegram

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

### WhatsApp

```bash
openclaw channels login whatsapp
# Scan QR code
```

### Discord

```bash
openclaw channels add --channel discord --token "<YOUR_BOT_TOKEN>"
```

See [Channels docs](https://docs.openclaw.ai/channels) for other providers.

---

## Manual Config (Advanced)

If you prefer to configure manually instead of using the wizard, edit `~/.openclaw/openclaw.json`:

```bash
nano ~/.openclaw/openclaw.json
```

### NVIDIA NIM (Kimi K2.5)

```json5
{
  "env": {
    "NVIDIA_API_KEY": "nvapi-your-key-here"
  },
  "agents": {
    "defaults": {
      "model": { "primary": "nvidia-nim/moonshotai/kimi-k2.5" },
      "models": {
        "nvidia-nim/moonshotai/kimi-k2.5": { "alias": "Kimi K2.5 (NIM)" }
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "nvidia-nim": {
        "baseUrl": "https://integrate.api.nvidia.com/v1",
        "apiKey": "${NVIDIA_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "moonshotai/kimi-k2.5",
            "name": "Kimi K2.5 (NVIDIA NIM)",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 131072,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

### Moonshot API Direct (Alternative)

```json5
{
  "env": {
    "MOONSHOT_API_KEY": "sk-your-key-here"
  },
  "agents": {
    "defaults": {
      "model": { "primary": "moonshot/kimi-k2.5" },
      "models": {
        "moonshot/kimi-k2.5": { "alias": "Kimi K2.5" }
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "moonshot": {
        "baseUrl": "https://api.moonshot.ai/v1",
        "apiKey": "${MOONSHOT_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "kimi-k2.5",
            "name": "Kimi K2.5",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 256000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

After editing, restart the gateway:
```bash
openclaw gateway restart
```

---

## Optimizations for 1GB RAM

The $6 droplet only has 1GB RAM. To keep things running smoothly:

### Add Swap (Recommended)

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Monitor Memory

```bash
free -h
htop
```

---

## Persistence & Backup

All state lives in:
- `~/.openclaw/` — config, credentials, session data
- `~/.openclaw/workspace/` — workspace (SOUL.md, memory, etc.)

Backup periodically:
```bash
tar -czvf openclaw-backup.tar.gz ~/.openclaw
```

---

## Troubleshooting

### Permission Denied (EACCES)

```bash
# Fix ownership (Docker runs as uid 1000)
sudo chown -R 1000:1000 ~/.openclaw
```

### Gateway Won't Start

```bash
openclaw gateway status
openclaw doctor --non-interactive
journalctl --user -u openclaw-gateway.service -n 50
```

### Port Already in Use

```bash
lsof -i :18789
kill <PID>
```

### Out of Memory

```bash
# Check memory
free -h

# Add more swap or upgrade to $12/mo droplet (2GB RAM)
```

### Node User Doesn't Exist

```bash
# Use numeric UID/GID instead
sudo chown -R 1000:1000 ~/.openclaw
```

---

## Quick Reference Commands

```bash
# --- Onboarding ---
openclaw onboard --install-daemon                              # Interactive wizard
openclaw onboard --non-interactive --auth-choice nvidia-nim-api-key  # Non-interactive

# --- Gateway ---
openclaw gateway start
openclaw gateway restart
openclaw gateway status

# --- Models ---
openclaw models list                                           # List all available models
openclaw models list | grep nvidia-nim                         # Check NVIDIA NIM model
openclaw config show                                           # View full configuration

# --- Testing ---
openclaw agent --message "Hello, what model are you?"          # Send test message
openclaw agent --message "Explain quantum computing"           # Test longer response

# --- Health ---
openclaw doctor                                                # Run diagnostics
openclaw status                                                # Quick status check

# --- Channels ---
openclaw pairing approve <channel> <code>                      # Approve DM pairing
openclaw channels login whatsapp                               # WhatsApp QR scan

# --- Maintenance ---
openclaw gateway restart                                       # Restart after config changes
journalctl --user -u openclaw-gateway.service -f               # Stream logs
```

---

## Resources

| Resource | Link |
|----------|------|
| **OpenClaw Website** | [https://openclaw.ai](https://openclaw.ai) |
| **Documentation** | [https://docs.openclaw.ai](https://docs.openclaw.ai) |
| **GitHub Repository** | [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) |
| **Discord Community** | [https://discord.gg/clawd](https://discord.gg/clawd) |
| **NVIDIA NIM Kimi K2.5** | [https://build.nvidia.com/moonshotai/kimi-k2.5](https://build.nvidia.com/moonshotai/kimi-k2.5) |
| **NVIDIA NIM API Keys** | [https://build.nvidia.com/settings/api-keys](https://build.nvidia.com/settings/api-keys) |

---

## NVIDIA NIM vs Moonshot API

| Feature | Moonshot API | NVIDIA NIM (built-in) |
|---------|--------------|------------|
| **Setup** | Wizard → Moonshot AI | Wizard → **NVIDIA NIM** |
| **Endpoint** | `https://api.moonshot.ai/v1` | `https://integrate.api.nvidia.com/v1` |
| **OpenClaw Model Ref** | `moonshot/kimi-k2.5` | `nvidia-nim/moonshotai/kimi-k2.5` |
| **NIM Model ID** | `kimi-k2.5` | `moonshotai/kimi-k2.5` |
| **Context Window** | 256,000 tokens | 131,072 tokens |
| **Max Output** | 8,192 tokens | 8,192 tokens |
| **Input** | Text | Text + Image |
| **API Type** | OpenAI-compatible | OpenAI-compatible |
| **API Key Prefix** | `sk-` | `nvapi-` |
| **Env Var** | `MOONSHOT_API_KEY` | `NVIDIA_API_KEY` |
| **Auto-Discovery** | ✅ | ✅ |

---

## What We Changed (Custom Build)

This custom build adds NVIDIA NIM as a first-class provider. Files modified:

| File | Change |
|------|--------|
| `src/commands/onboard-types.ts` | Added `nvidia-nim-api-key` auth choice |
| `src/commands/auth-choice-options.ts` | Added NVIDIA NIM group to setup menu |
| `src/commands/auth-choice.preferred-provider.ts` | Mapped auth choice → provider |
| `src/commands/onboard-auth.models.ts` | Model constants + builder for Kimi K2.5 |
| `src/commands/onboard-auth.credentials.ts` | `setNvidiaNimApiKey()` credential store |
| `src/commands/onboard-auth.config-core.ts` | Config applier functions |
| `src/commands/auth-choice.apply.api-providers.ts` | Wizard handler (prompt + store + apply) |
| `src/agents/model-auth.ts` | `NVIDIA_API_KEY` env var mapping |
| `src/agents/models-config.providers.ts` | Auto-discovery builder |
| `src/commands/onboard-auth.ts` | Barrel re-exports |

---

> **Last Updated:** February 2026
> 
> **Source:** Custom OpenClaw build with NVIDIA NIM provider support.
> Based on official OpenClaw architecture + [NVIDIA NIM API docs](https://docs.api.nvidia.com/nim/reference/moonshotai-kimi-k2.5).
