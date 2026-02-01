# Ubuntu Server + OpenClaw Setup Plan (Revised)

**Date:** 2026-01-27 (revised 2026-02-01, added AI automation prompts)
**Hardware:** Lenovo ThinkStation P410
**Target OS:** Ubuntu Server 24.04 LTS

## Server Details

- **Hostname:** jeff-server
- **Admin user:** mattias (sudo access, SSH login)
- **Bot user:** jeff (non-sudo, runs OpenClaw daemon)
- **Local IP:** 192.168.0.141
- **SSH:** `ssh mattias@192.168.0.141`

## Hardware Specs

- **CPU:** Intel Xeon E5-1620 v4 @ 3.50GHz (4 cores, 8 threads)
- **RAM:** 32 GB
- **Storage:** SanDisk 256GB SSD (+ spare 250GB SSDs available)
- **GPU:** NVIDIA Quadro M4000
- **BIOS:** Configured for UEFI mode

## User Context

- Primary user is mattias (Belgium, Europe/Brussels timezone)
- Two Windows PCs: "jaguar" (Win11) and "bear" (Win10)
- Bot will be named "jeff" and run under the jeff user account
- Using Telegram as the messaging channel (not WhatsApp)

---

## Overview & Architecture

**Goal:** Turn the ThinkStation P410 into a headless Ubuntu server running OpenClaw with Docker sandboxing.

```text
┌─────────────────────────────────────────────────────────────┐
│  ThinkStation P410 (headless, on your network)              │
│  ├── Ubuntu Server 24.04 LTS                                │
│  ├── SSH server (port 22) ← local access                    │
│  ├── Tailscale ← secure remote access from anywhere         │
│  ├── Docker ← sandboxed tool execution                      │
│  └── OpenClaw daemon (as user jeff)                         │
│       ├── Gateway (localhost:18789)                         │
│       ├── Tailscale Serve (HTTPS proxy)                     │
│       ├── Telegram channel                                  │
│       ├── Sandboxed tools + browser                         │
│       └── Agent workspace                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
    Your Windows PC / phone / laptop
    (SSH when home, Tailscale when away)
```

### References

**Official Documentation:**

- [Getting Started](https://docs.openclaw.ai/start/getting-started) — Zero to first message guide
- [Architecture Overview](https://docs.openclaw.ai/concepts/architecture) — System architecture concepts

**Local Repository Files:**

- [docs/concepts/architecture.md](../concepts/architecture.md) — Detailed architecture documentation
- [docs/start/getting-started.md](../start/getting-started.md) — Beginner guide

---

## AI Automation Summary

This table shows which phases can be automated by Claude Code (before OpenClaw is running) or by OpenClaw itself (once operational).

| Phase | Claude Code | OpenClaw | Manual Steps Required |
|-------|:-----------:|:--------:|----------------------|
| 1. User Account Restructuring | ✅ Full | — | Password input (interactive) |
| 2. System Preparation | ✅ Full | — | Router static IP configuration |
| 3. Tailscale Setup | ✅ Partial | — | Browser authorization, MagicDNS setup |
| 4. Claude Code Installation | — | — | Install script, browser auth (bootstrap phase) |
| 5. Docker Setup | ✅ Full | — | Re-login for group membership |
| 6. OpenClaw Installation | ✅ Pre/Post | — | Onboarding wizard (interactive) |
| 7. Sandbox Setup | ✅ Full | ✅ Config | — |
| 8. Security Hardening | ✅ Full | ✅ Audit | API keys and tokens (secrets) |
| 9. Telegram Channel Setup | ✅ Config | ✅ Troubleshoot | BotFather bot creation, user ID lookup |
| 10. Tailscale Serve | ✅ Full | ✅ Verify | NextDNS configuration |
| 11. Finalization | ✅ Full | ✅ Verify | — |

### Automation Strategy

**Optimal order for AI-assisted setup:**

1. **Phases 1-3:** Use Claude Code via SSH from Windows PC
2. **Phase 4:** Install Claude Code on server (manual, but enables remaining automation)
3. **Phases 5-6:** Use Claude Code on server for Docker + OpenClaw installation
4. **Phases 7-11:** Use Claude Code or OpenClaw (once running) for remaining configuration

**Key documentation locations for AI investigation:**

| Topic | Path |
|-------|------|
| Installation | `docs/install/` |
| Gateway configuration | `docs/gateway/configuration.md` |
| Security | `docs/cli/security.md`, `SECURITY.md` |
| Telegram channel | `docs/channels/telegram.md` |
| Tailscale integration | `docs/gateway/tailscale.md` |
| Troubleshooting | `docs/gateway/troubleshooting.md` |
| CLI commands | `docs/cli/` |

---

## Phase 1: User Account Restructuring

The server was set up with `jeff` as the admin user. We need to restructure:

- **mattias** → admin user (sudo access, SSH login, system management)
- **jeff** → bot user (non-sudo after setup, runs OpenClaw daemon)

### Steps (while logged in as jeff via SSH)

```bash
# 1. Create new admin user
sudo adduser mattias
sudo usermod -aG sudo mattias

# 2. Copy SSH keys to the new user
sudo mkdir -p /home/mattias/.ssh
sudo cp ~/.ssh/authorized_keys /home/mattias/.ssh/
sudo chown -R mattias:mattias /home/mattias/.ssh
sudo chmod 700 /home/mattias/.ssh
sudo chmod 600 /home/mattias/.ssh/authorized_keys

# 3. Test new user login (from Windows, new terminal)
ssh mattias@192.168.0.141

# 4. Set password for mattias
sudo passwd mattias
```

**Important:** Keep jeff with sudo access temporarily until setup is complete. Remove it in Phase 11.

### AI Automation (Claude Code)

**Can be automated:** Yes, after initial SSH connection is established.

**Prompt for Claude Code:**

> I'm logged into an Ubuntu server via SSH as user `jeff` (who currently has sudo access). I need to restructure user accounts:
>
> 1. Create a new admin user called `mattias` with sudo privileges
> 2. Copy SSH authorized_keys from jeff to mattias so I can SSH in as mattias
> 3. Set appropriate permissions on the .ssh directory
>
> Check the current user setup with `whoami`, `groups`, and `ls -la ~/.ssh/`. Verify the authorized_keys file exists before copying.
>
> Note: Setting the password requires interactive input - pause and let me type it when you reach that step.

**What to investigate:**

- Current SSH authorized_keys location: `~/.ssh/authorized_keys`
- Ubuntu user management: `adduser`, `usermod`
- SSH key permissions requirements (700 for .ssh, 600 for keys)

### References

**Official Documentation:**

- [Gateway Security](https://docs.openclaw.ai/gateway/security) — User separation and privilege isolation best practices

**Local Repository Files:**

- [docs/cli/security.md](../cli/security.md) — `openclaw security` command reference
- [SECURITY.md](../../SECURITY.md) — Security policies and vulnerability reporting

---

## Phase 2: System Preparation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Enable automatic security updates
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades

# Set timezone
sudo timedatectl set-timezone Europe/Brussels

# Configure firewall
sudo ufw allow ssh
sudo ufw allow 41641/udp  # Tailscale
sudo ufw enable
sudo ufw status

# Set static IP on router
# Log into router (usually 192.168.0.1)
# Find "DHCP Reservation" or "Static IP Assignment"
# Add server's MAC address → assign 192.168.0.141
```

### NPM Supply Chain Security (recommended)

After Node.js is installed (Phase 6), configure npm to disable lifecycle scripts by default:

```bash
# Disable preinstall/postinstall scripts (primary npm attack vector)
npm config set ignore-scripts true

# When you need to install a package that requires build steps (e.g., sharp):
npm install <package> --ignore-scripts=false
```

This prevents malicious packages from executing code during `npm install`.

### AI Automation (Claude Code)

**Can be automated:** Yes, fully (except router static IP which requires router admin access).

**Prompt for Claude Code:**

> I need to prepare this Ubuntu Server 24.04 system for running OpenClaw. Please:
>
> 1. Update the system packages
> 2. Enable automatic security updates via unattended-upgrades
> 3. Set the timezone to Europe/Brussels
> 4. Configure UFW firewall to allow SSH (port 22) and Tailscale (UDP 41641), then enable it
> 5. Show me the current network interface and IP so I can configure a static DHCP reservation on my router
>
> Check the current state first: what's the current timezone? Is UFW already installed? What firewall rules exist?
>
> Reference the OpenClaw security documentation at `docs/cli/security.md` and `docs/gateway/configuration.md` for recommended firewall settings.

**What to investigate:**

- Current timezone: `timedatectl`
- UFW status: `sudo ufw status verbose`
- Network info: `ip addr`, `ip route`
- Unattended-upgrades config: `/etc/apt/apt.conf.d/50unattended-upgrades`

**Manual step required:** Configure static IP reservation on router admin interface.

### References

**Official Documentation:**

- [Gateway Security](https://docs.openclaw.ai/gateway/security) — Network binding and firewall recommendations
- [Linux Platform Guide](https://docs.openclaw.ai/platforms/linux) — Linux-specific setup considerations

**Local Repository Files:**

- [docs/cli/security.md](../cli/security.md) — `openclaw security` command reference
- [docs/network.md](../network.md) — Network configuration details
- [docs/environment.md](../environment.md) — Environment variables reference

---

## Phase 3: Tailscale Setup

### What is Tailscale?

Tailscale creates a private encrypted network between your devices—regardless of where they are. Each device gets a stable IP (like `100.x.y.z`) that works anywhere.

### Install on Ubuntu Server

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Open the printed URL in browser to authorize.

### Install on Windows PCs

Download from <https://tailscale.com/download/windows> and sign in with same account.

### Get Tailscale IP

```bash
tailscale ip -4    # Returns 100.x.y.z
```

### Enable Magic DNS

In Tailscale admin console → DNS → Enable MagicDNS.

Then connect with:

```bash
ssh mattias@jeff-server
```

### AI Automation (Claude Code)

**Can be automated:** Partially. Installation is automated, but browser authorization is manual.

**Prompt for Claude Code:**

> I need to set up Tailscale on this Ubuntu server for secure remote access. Please:
>
> 1. Check if Tailscale is already installed (look for `tailscale` binary)
> 2. Install Tailscale using their official install script
> 3. Start the Tailscale authentication flow with `tailscale up`
> 4. When it outputs an authorization URL, pause and let me open it in a browser on my Windows PC
> 5. After I've authorized, verify the connection and show me the Tailscale IP
>
> Reference OpenClaw's Tailscale integration documentation at `docs/gateway/tailscale.md` for how OpenClaw uses Tailscale Serve mode.

**What to investigate:**

- Tailscale status: `tailscale status`
- Tailscale IP: `tailscale ip -4`
- OpenClaw Tailscale config: `docs/gateway/tailscale.md`
- MagicDNS setup in Tailscale admin console (manual step)

**Manual steps required:**

- Open authorization URL in browser
- Enable MagicDNS in Tailscale admin console
- Install Tailscale on Windows PCs (jaguar, bear)

### References

**Official Documentation:**

- [Tailscale Integration](https://docs.openclaw.ai/gateway/tailscale) — Serve mode, Funnel mode, authentication with Tailscale identity headers
- [Remote Access](https://docs.openclaw.ai/gateway/remote) — Remote gateway access patterns

**Local Repository Files:**

- [docs/gateway/tailscale.md](../gateway/tailscale.md) — Detailed Tailscale integration guide
- [docs/gateway/remote.md](../gateway/remote.md) — Remote access configuration
- [docs/gateway/discovery.md](../gateway/discovery.md) — Service discovery and networking

---

## Phase 4: Claude Code Installation

Installing Claude Code allows direct troubleshooting assistance during remaining setup phases.

### Install (as jeff)

```bash
# Via npm (Node.js will be installed in Phase 6)
# So install Claude Code AFTER OpenClaw installer runs, or use:
curl -fsSL https://claude.ai/install-cli | bash
```

### Authentication

```bash
claude
# Opens authentication flow - provides URL to authorize
# Open URL on Windows PC, authorize, paste token back
```

### Usage for troubleshooting

```bash
# From Windows, SSH in and start Claude Code
ssh jeff@jeff-server
claude

# Or as mattias, switch to jeff first
ssh mattias@jeff-server
sudo -i -u jeff
claude
```

### AI Automation Note

**This is the bootstrap phase.** Once Claude Code is installed and authenticated, it becomes available for automating remaining phases.

**Initial setup is manual:**

1. Run the install script
2. Complete browser-based authentication
3. Paste token back to terminal

**After installation, Claude Code enables:**

- Direct troubleshooting on the server
- Execution of remaining phases with AI assistance
- Investigation of issues if OpenClaw fails to start

**Tip:** Install Claude Code early (before OpenClaw) so you have AI assistance available for troubleshooting the more complex phases.

### References

**External Documentation:**

- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code) — Official Claude Code setup and usage

---

## Phase 5: Docker Setup

```bash
# Install Docker
sudo apt update
sudo apt install docker.io -y

# Add jeff to docker group (so OpenClaw can manage containers)
sudo usermod -aG docker jeff

# Enable Docker to start on boot
sudo systemctl enable docker

# Log out and back in for group membership to take effect
exit
# SSH back in
```

Verify Docker works:

```bash
docker run hello-world
```

### AI Automation (Claude Code)

**Can be automated:** Yes, fully.

**Prompt for Claude Code:**

> I need to set up Docker on this Ubuntu server for OpenClaw's sandboxed tool execution. Please:
>
> 1. Check if Docker is already installed
> 2. Install Docker from Ubuntu's package repository (docker.io)
> 3. Add user `jeff` to the docker group so OpenClaw can manage containers without sudo
> 4. Enable Docker to start automatically on boot
> 5. Verify Docker is working by running a test container
>
> Important: After adding jeff to the docker group, I'll need to log out and back in for group membership to take effect. Remind me of this.
>
> Reference OpenClaw's Docker documentation at `docs/install/docker.md` and sandboxing docs at `docs/gateway/sandboxing.md` to understand how OpenClaw uses Docker.

**What to investigate:**

- Docker installation status: `which docker`, `docker --version`
- Docker service: `systemctl status docker`
- User groups: `groups jeff`
- OpenClaw Docker requirements: `docs/install/docker.md`
- Sandboxing architecture: `docs/gateway/sandboxing.md`

### References

**Official Documentation:**

- [Docker Installation](https://docs.openclaw.ai/install/docker) — Docker setup for OpenClaw
- [Sandboxing](https://docs.openclaw.ai/gateway/sandboxing) — How Docker sandboxing works

**Local Repository Files:**

- [docs/install/docker.md](../install/docker.md) — Docker installation guide
- [docs/gateway/sandboxing.md](../gateway/sandboxing.md) — Sandboxing security model
- [docs/gateway/sandbox-vs-tool-policy-vs-elevated.md](../gateway/sandbox-vs-tool-policy-vs-elevated.md) — Comparing sandboxing approaches

---

## Phase 6: OpenClaw Installation

### Run the installer (as jeff)

```bash
curl -fsSL https://openclaw.bot/install.sh | bash
```

The installer handles:

- Node.js 22.12.0+ installation via NodeSource (minimum required for security patches)
- npm permission fixes (switches prefix to `~/.npm-global`)
- Global OpenClaw package installation
- Interactive onboarding wizard
- Systemd daemon setup

### Verify Node.js version

```bash
node --version  # Must be v22.12.0 or later (security requirement)
```

Required for CVE-2025-59466 (async_hooks DoS) and CVE-2026-21636 (permission model bypass) patches.

### Onboarding wizard options

If you need to re-run later:

```bash
openclaw onboard                    # Default flow
openclaw onboard --flow quickstart  # Minimal prompts
openclaw onboard --flow manual      # Full control over port/bind/auth
```

### Ubuntu systemd gotcha

If you see `"Systemd user services are unavailable"`:

```bash
# Verify your session has required environment
echo $XDG_RUNTIME_DIR    # Should show /run/user/$(id -u)
echo $DBUS_SESSION_BUS_ADDRESS  # Should be set

# Enable lingering and re-login
sudo loginctl enable-linger jeff
exit
# SSH back in as jeff
```

### Manual service creation (if wizard fails)

```bash
mkdir -p ~/.config/systemd/user/

cat > ~/.config/systemd/user/openclaw.service << 'EOF'
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
ExecStart=%h/.npm-global/bin/openclaw gateway
Restart=on-failure
RestartSec=5
Environment=PATH=%h/.npm-global/bin:/usr/local/bin:/usr/bin:/bin

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable openclaw
systemctl --user start openclaw
```

### Verify running

```bash
systemctl --user status openclaw
journalctl --user -u openclaw -f    # Live logs
openclaw status                      # Quick overview
openclaw status --deep               # Detailed health check
```

### AI Automation (Claude Code or OpenClaw)

**Can be automated:** Partially. Installation script runs automatically, but the onboarding wizard is interactive.

**Prompt for Claude Code (pre-installation checks):**

> Before installing OpenClaw, I need to verify prerequisites are in place:
>
> 1. Check Node.js version (must be v22.12.0 or later)
> 2. Verify Docker is installed and jeff is in the docker group
> 3. Check if lingering is enabled for user jeff (required for user systemd services)
> 4. Review the OpenClaw installer documentation at `docs/install/installer.md` to understand what it will do
>
> If Node.js isn't installed or is too old, the OpenClaw installer will install it. Check the installer script documentation to understand the installation flow.

**Prompt for Claude Code (post-installation troubleshooting):**

> OpenClaw installation completed but the service isn't starting properly. Please:
>
> 1. Check systemd user service status: `systemctl --user status openclaw`
> 2. View service logs: `journalctl --user -u openclaw -n 50`
> 3. Verify XDG_RUNTIME_DIR and DBUS_SESSION_BUS_ADDRESS are set (required for user systemd)
> 4. Check if lingering is enabled: `loginctl show-user jeff | grep Linger`
> 5. Review the troubleshooting section in `docs/gateway/troubleshooting.md`
>
> If user systemd services are unavailable, enable lingering and re-login.

**What to investigate:**

- Installer script source: `scripts/` directory (if cloning from source)
- Installer documentation: `docs/install/installer.md`
- Node.js requirements: `docs/install/node.md`
- Onboarding wizard: `docs/start/wizard.md`, `docs/cli/onboard.md`
- Systemd setup: `docs/gateway/background-process.md`
- Troubleshooting: `docs/gateway/troubleshooting.md`

### References

**Official Documentation:**

- [Installation Guide](https://docs.openclaw.ai/install/) — Main installation documentation
- [Getting Started Wizard](https://docs.openclaw.ai/start/wizard) — Onboarding wizard documentation
- [Updating / Rollback](https://docs.openclaw.ai/install/updating) — Update procedures

**Local Repository Files:**

- [docs/install/index.md](../install/index.md) — Installation overview
- [docs/install/installer.md](../install/installer.md) — Installer script documentation
- [docs/install/node.md](../install/node.md) — Node.js installation guide
- [docs/start/wizard.md](../start/wizard.md) — Onboarding wizard details
- [docs/start/onboarding.md](../start/onboarding.md) — Detailed onboarding process
- [docs/cli/onboard.md](../cli/onboard.md) — `openclaw onboard` command reference
- [docs/cli/status.md](../cli/status.md) — `openclaw status` command reference
- [docs/gateway/background-process.md](../gateway/background-process.md) — Background process management

---

## Phase 7: Sandbox Setup

Build sandbox images (as jeff). The official method uses shell scripts:

```bash
# Clone the repo temporarily to get the scripts
git clone --depth 1 https://github.com/ArcadeAI/openclaw.git /tmp/openclaw-scripts

# Build base sandbox image (openclaw-sandbox:bookworm-slim)
/tmp/openclaw-scripts/scripts/sandbox-setup.sh

# Build browser sandbox image
/tmp/openclaw-scripts/scripts/sandbox-browser-setup.sh

# Clean up
rm -rf /tmp/openclaw-scripts
```

### Important: Sandbox network default

By default, sandbox containers run with **no network access**. If you need sandboxed tools to access the internet (e.g., `web_fetch`), add to your config:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          network: "bridge"  // Enable network access
        }
      }
    }
  }
}
```

### Fix permissions if needed

```bash
chown -R 1000:1000 ~/.openclaw/agents/
```

### AI Automation (Claude Code or OpenClaw)

**Can be automated:** Yes, fully.

**Prompt for Claude Code:**

> I need to build the Docker sandbox images for OpenClaw. Please:
>
> 1. Check if the sandbox images already exist: `docker images | grep openclaw`
> 2. Clone the OpenClaw repo temporarily to get the build scripts
> 3. Run the sandbox setup script to build the base image
> 4. Run the browser sandbox setup script to build the browser image
> 5. Clean up the temporary clone
> 6. Verify both images were created successfully
>
> Reference the sandboxing documentation at `docs/gateway/sandboxing.md` to understand:
>
> - What these images are used for
> - Network isolation settings (default is no network)
> - When to enable bridge networking for tools that need internet access

**Prompt for OpenClaw (once running):**

> I need to configure sandbox settings. Check the current sandbox configuration in `~/.openclaw/openclaw.json`. Explain what `sandbox.mode`, `sandbox.scope`, and `docker.network` options do. Reference `docs/gateway/sandboxing.md` for details.

**What to investigate:**

- Sandbox scripts in repo: `scripts/sandbox-setup.sh`, `scripts/sandbox-browser-setup.sh`
- Sandbox documentation: `docs/gateway/sandboxing.md`
- Configuration options: `docs/gateway/configuration.md`
- Browser sandbox: `docs/cli/browser.md`
- Agent workspace structure: `docs/concepts/agent-workspace.md`

### References

**Official Documentation:**

- [Sandboxing](https://docs.openclaw.ai/gateway/sandboxing) — Docker sandbox security model and configuration
- [Configuration](https://docs.openclaw.ai/gateway/configuration) — Sandbox configuration options

**Local Repository Files:**

- [docs/gateway/sandboxing.md](../gateway/sandboxing.md) — Comprehensive sandboxing documentation
- [docs/gateway/sandbox-vs-tool-policy-vs-elevated.md](../gateway/sandbox-vs-tool-policy-vs-elevated.md) — When to use sandboxing vs other approaches
- [docs/cli/sandbox.md](../cli/sandbox.md) — `openclaw sandbox` command reference
- [docs/cli/browser.md](../cli/browser.md) — `openclaw browser` command reference
- [docs/concepts/agent-workspace.md](../concepts/agent-workspace.md) — Agent workspace structure

---

## Phase 8: Security Hardening

### File permissions (critical)

```bash
chmod 700 ~/.openclaw/
chmod 600 ~/.openclaw/openclaw.json
chmod 600 ~/.openclaw/.env
```

### Generate gateway token

```bash
openssl rand -hex 32
```

### Environment file (`~/.openclaw/.env`)

```bash
OPENCLAW_GATEWAY_TOKEN=your-generated-token-here
ANTHROPIC_API_KEY=sk-ant-...
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
```

### Secure configuration (`~/.openclaw/openclaw.json`)

```json5
{
  gateway: {
    bind: "loopback",           // Never expose directly
    port: 18789,
    auth: {
      mode: "token",
      token: "${OPENCLAW_GATEWAY_TOKEN}"
    }
  },
  channels: {
    telegram: {
      enabled: true,
      botToken: "${TELEGRAM_BOT_TOKEN}",  // Note: "botToken" not "token"
      dmPolicy: "pairing",                 // Time-limited codes for new senders
      allowFrom: [YOUR_TELEGRAM_USER_ID],  // Numeric ID (get from @userinfobot)
      groups: {
        "*": { requireMention: true }
      }
    }
  },
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",       // Your DM sessions unsandboxed, groups sandboxed
        scope: "session",        // One container per session
        workspaceAccess: "ro"    // Read-only workspace access (disables write/edit tools)
      }
    }
  },
  session: {
    dmScope: "per-channel-peer"  // Prevents cross-user context leakage
  },
  logging: {
    level: "info",
    redactSensitive: "tools"
  }
}
```

**Note on `workspaceAccess`:**

- `"none"` (default): Tools see isolated sandbox workspace only
- `"ro"`: Agent workspace mounted read-only at `/agent` (disables write/edit/apply_patch)
- `"rw"`: Agent workspace mounted read/write at `/workspace`

### Run security audit

```bash
openclaw security audit --deep
openclaw security audit --fix   # Auto-apply safe fixes
```

### Security warnings

1. **Prompt injection is not solved** — treat untrusted content (web results, attachments, emails) as hostile
2. **Session transcripts** at `~/.openclaw/agents/<id>/sessions/*.jsonl` contain raw conversations — protect with file permissions
3. **Never use** `gateway.bind: "0.0.0.0"` without authentication
4. **Use instruction-hardened models** (Claude Opus 4.5 recommended)

### AI Automation (Claude Code or OpenClaw)

**Can be automated:** Mostly. File permissions and config setup are automatable. Token generation is local.

**Prompt for Claude Code:**

> I need to harden the security configuration for OpenClaw. Please:
>
> 1. Check current file permissions on `~/.openclaw/` directory and its contents
> 2. Set restrictive permissions: 700 for directories, 600 for config files
> 3. Generate a secure random token for gateway authentication: `openssl rand -hex 32`
> 4. Review the example secure configuration in `docs/gateway/configuration-examples.md`
> 5. Help me create the `.env` file with proper placeholders for secrets
>
> Important: I'll need to manually add my actual API keys and tokens. Show me the structure but use placeholder values.
>
> Reference security documentation at `docs/cli/security.md` and `docs/gateway/authentication.md`.

**Prompt for OpenClaw (once running):**

> Run a security audit on my OpenClaw installation. Use `openclaw security audit --deep` to check for issues. Explain any findings and suggest fixes. Reference `docs/cli/security.md` for the security command documentation.

**What to investigate:**

- Security CLI: `docs/cli/security.md`
- Authentication options: `docs/gateway/authentication.md`
- Configuration reference: `docs/gateway/configuration.md`
- Configuration examples: `docs/gateway/configuration-examples.md`
- Session isolation: `docs/concepts/session.md`
- Logging and redaction: `docs/logging.md`
- Security policies: `SECURITY.md`

### References

**Official Documentation:**

- [Gateway Security](https://docs.openclaw.ai/gateway/security) — Comprehensive security guide including access control, credential management, incident response
- [Configuration](https://docs.openclaw.ai/gateway/configuration) — Full configuration reference
- [Configuration Examples](https://docs.openclaw.ai/gateway/configuration-examples) — Example secure configurations
- [Sessions](https://docs.openclaw.ai/concepts/session) — Session scoping and isolation

**Local Repository Files:**

- [docs/cli/security.md](../cli/security.md) — `openclaw security` command reference
- [SECURITY.md](../../SECURITY.md) — Security policies and vulnerability reporting
- [docs/gateway/configuration.md](../gateway/configuration.md) — Complete configuration reference
- [docs/gateway/configuration-examples.md](../gateway/configuration-examples.md) — Example configurations
- [docs/gateway/authentication.md](../gateway/authentication.md) — Gateway authentication (tokens, OAuth)
- [docs/cli/security.md](../cli/security.md) — `openclaw security` command reference
- [docs/concepts/session.md](../concepts/session.md) — Session management concepts
- [docs/logging.md](../logging.md) — Logging configuration
- [SECURITY.md](../../SECURITY.md) — Security policies and vulnerability reporting

---

## Phase 9: Telegram Channel Setup

### Get your Telegram user ID

Message [@userinfobot](https://t.me/userinfobot) on Telegram — it replies with your numeric ID.

### Create a bot via BotFather

1. Message [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot`
3. Follow prompts to name your bot
4. Save the bot token (add to `~/.openclaw/.env` as `TELEGRAM_BOT_TOKEN`)

### Update configuration

Add your user ID to `allowFrom` in the config (see Phase 8).

### Restart and test

```bash
systemctl --user restart openclaw
```

Message your bot on Telegram. First message will trigger pairing flow.

### AI Automation (Claude Code or OpenClaw)

**Can be automated:** Partially. Configuration is automatable, but BotFather interaction is manual.

**Prompt for Claude Code (configuration):**

> I've created a Telegram bot via BotFather and have my bot token. I need to configure OpenClaw to use it. Please:
>
> 1. Read the current OpenClaw configuration at `~/.openclaw/openclaw.json`
> 2. Review the Telegram channel documentation at `docs/channels/telegram.md`
> 3. Help me add the Telegram channel configuration with:
>    - `botToken` from environment variable
>    - `dmPolicy: "pairing"` for security
>    - `allowFrom` with my Telegram user ID
>    - `groups` with `requireMention: true`
> 4. Add the bot token to `~/.openclaw/.env` as `TELEGRAM_BOT_TOKEN`
>
> Important: The config key is `botToken`, not `token` (this is a common gotcha).

**Prompt for OpenClaw (troubleshooting):**

> My Telegram bot isn't responding to messages. Help me troubleshoot:
>
> 1. Check if the Telegram channel is enabled and connected: `openclaw status --deep`
> 2. Review gateway logs for Telegram-related errors
> 3. Verify the bot token is correct and the bot is reachable
> 4. Check if there's an IPv6 issue with the Telegram API (known issue)
>
> Reference `docs/channels/troubleshooting.md` and `docs/channels/telegram.md` for Telegram-specific issues.

**What to investigate:**

- Telegram channel setup: `docs/channels/telegram.md`
- grammY library details: `docs/channels/grammy.md`
- Pairing flow: `docs/start/pairing.md`
- Channel troubleshooting: `docs/channels/troubleshooting.md`
- Group message handling: `docs/concepts/groups.md`, `docs/concepts/group-messages.md`
- Channels CLI: `docs/cli/channels.md`

**Manual steps required:**

- Message @userinfobot to get your Telegram user ID
- Create bot via @BotFather and save the token

### References

**Official Documentation:**

- [Telegram Channel](https://docs.openclaw.ai/channels/telegram) — Complete Telegram integration guide including bot setup, access control, group management, troubleshooting
- [Pairing](https://docs.openclaw.ai/start/pairing) — DM pairing flow documentation
- [Groups](https://docs.openclaw.ai/concepts/groups) — Group chat handling

**Local Repository Files:**

- [docs/channels/telegram.md](../channels/telegram.md) — Detailed Telegram channel documentation
- [docs/channels/grammy.md](../channels/grammy.md) — grammY library details (underlying Telegram library)
- [docs/channels/troubleshooting.md](../channels/troubleshooting.md) — Channel troubleshooting guide
- [docs/start/pairing.md](../start/pairing.md) — Device pairing documentation
- [docs/concepts/groups.md](../concepts/groups.md) — Group chat configuration
- [docs/concepts/group-messages.md](../concepts/group-messages.md) — Group message processing
- [docs/cli/channels.md](../cli/channels.md) — `openclaw channels` command reference
- [docs/cli/pairing.md](../cli/pairing.md) — `openclaw pairing` command reference

---

## Phase 10: Tailscale Serve Integration

Tailscale Serve exposes your gateway over HTTPS on your tailnet without opening ports.

### Option A: Config-driven (recommended)

OpenClaw can auto-configure Tailscale Serve. Update your config:

```json5
{
  gateway: {
    bind: "loopback",
    port: 18789,
    tailscale: { mode: "serve" },  // OpenClaw manages Tailscale Serve
    auth: {
      mode: "token",
      token: "${OPENCLAW_GATEWAY_TOKEN}",
      allowTailscale: true  // Trust Tailscale identity headers
    }
  }
}
```

Then restart the gateway:

```bash
systemctl --user restart openclaw
```

### Option B: Manual Tailscale command

If you prefer manual control:

```bash
sudo tailscale serve --bg https / http://127.0.0.1:18789
```

### Access from anywhere on your tailnet

```
https://jeff-server.<tailnet-name>.ts.net/
```

### DNS Security (recommended)

Configure NextDNS in Tailscale admin console for DNS-level protection:

1. Go to Tailscale admin → DNS
2. Add NextDNS as resolver
3. In NextDNS dashboard, enable:
   - DNS Tunneling protection
   - Newly Registered Domains blocking
   - Threat Intelligence Feeds

This provides egress filtering at the DNS layer without needing OpenSnitch.

### AI Automation (Claude Code or OpenClaw)

**Can be automated:** Yes, fully (configuration-driven approach).

**Prompt for Claude Code:**

> I need to expose the OpenClaw gateway securely via Tailscale Serve. Please:
>
> 1. Verify Tailscale is connected: `tailscale status`
> 2. Review the Tailscale integration documentation at `docs/gateway/tailscale.md`
> 3. Update `~/.openclaw/openclaw.json` to add `tailscale: { mode: "serve" }` in the gateway section
> 4. Enable `allowTailscale: true` in the auth section to trust Tailscale identity headers
> 5. Restart the OpenClaw service: `systemctl --user restart openclaw`
> 6. Verify Tailscale Serve is running and show me the HTTPS URL
>
> Reference `docs/gateway/remote.md` for remote access patterns and `docs/gateway/authentication.md` for how Tailscale identity headers work.

**Prompt for OpenClaw:**

> Verify my Tailscale Serve integration is working correctly. Check:
>
> 1. Is the gateway accessible via the Tailscale HTTPS URL?
> 2. Are Tailscale identity headers being recognized?
> 3. Is the gateway correctly bound to loopback only?
>
> Reference `docs/gateway/tailscale.md` for the expected configuration.

**What to investigate:**

- Tailscale integration: `docs/gateway/tailscale.md`
- Remote access patterns: `docs/gateway/remote.md`
- Remote gateway setup: `docs/gateway/remote-gateway-readme.md`
- Gateway authentication: `docs/gateway/authentication.md`
- DNS configuration: `docs/cli/dns.md`

**Manual step:** Configure NextDNS in Tailscale admin console for DNS-level protection.

### References

**Official Documentation:**

- [Tailscale Integration](https://docs.openclaw.ai/gateway/tailscale) — Serve mode, Funnel mode, authentication with Tailscale identity headers
- [Remote Access](https://docs.openclaw.ai/gateway/remote) — Remote gateway access patterns
- [Gateway Security](https://docs.openclaw.ai/gateway/security) — Network binding and authentication

**Local Repository Files:**

- [docs/gateway/tailscale.md](../gateway/tailscale.md) — Detailed Tailscale integration guide
- [docs/gateway/remote.md](../gateway/remote.md) — Remote access configuration
- [docs/gateway/remote-gateway-readme.md](../gateway/remote-gateway-readme.md) — Remote gateway setup README
- [docs/gateway/authentication.md](../gateway/authentication.md) — Gateway authentication options
- [docs/cli/dns.md](../cli/dns.md) — `openclaw dns` command reference

---

## Phase 11: Finalization

### Remove sudo from jeff

```bash
# As mattias
sudo deluser jeff sudo
```

### Optionally lock jeff's password

```bash
sudo passwd -l jeff    # Lock: no password login possible
# sudo passwd -u jeff  # Unlock later if needed
```

The bot doesn't need a password during operation — the systemd service runs under jeff's user context automatically.

### Verify everything works

```bash
# As mattias
sudo -u jeff openclaw status --deep
sudo -u jeff systemctl --user status openclaw
```

### Go headless

Unplug monitor and keyboard. Server is now fully headless.

### AI Automation (Claude Code)

**Can be automated:** Yes, fully.

**Prompt for Claude Code:**

> I need to finalize the OpenClaw server setup by removing unnecessary privileges. Please:
>
> 1. Verify OpenClaw is running correctly: `sudo -u jeff openclaw status --deep`
> 2. Remove sudo access from jeff user: `sudo deluser jeff sudo`
> 3. Optionally lock jeff's password (the daemon doesn't need one)
> 4. Run a final security audit: `sudo -u jeff openclaw security audit --deep`
> 5. Verify the systemd service is still running correctly after privilege changes
> 6. Confirm I can still SSH in as mattias and access OpenClaw via Tailscale
>
> Reference `docs/cli/security.md` for the security audit command and `docs/cli/doctor.md` for the diagnostic command.

**Prompt for OpenClaw (final verification):**

> Perform a comprehensive health check on this installation:
>
> 1. Run `openclaw doctor` to check for any issues
> 2. Run `openclaw security audit --deep` to verify security configuration
> 3. Check all channels are connected and functioning
> 4. Verify sandbox containers can be created
> 5. Confirm Tailscale Serve is exposing the gateway correctly
>
> Report any warnings or issues that should be addressed before going headless.

**What to investigate:**

- Security CLI: `docs/cli/security.md`
- Doctor diagnostic: `docs/cli/doctor.md`, `docs/gateway/doctor.md`
- Troubleshooting: `docs/gateway/troubleshooting.md`
- Help hub: `docs/help/index.md`

### References

**Official Documentation:**

- [Gateway Security](https://docs.openclaw.ai/gateway/security) — Final security checklist
- [Troubleshooting](https://docs.openclaw.ai/gateway/troubleshooting) — Common issues and solutions
- [Help](https://docs.openclaw.ai/help) — Help hub with troubleshooting links

**Local Repository Files:**

- [docs/cli/security.md](../cli/security.md) — `openclaw security` command reference
- [SECURITY.md](../../SECURITY.md) — Security policies
- [docs/gateway/troubleshooting.md](../gateway/troubleshooting.md) — Gateway troubleshooting guide
- [docs/cli/doctor.md](../cli/doctor.md) — `openclaw doctor` diagnostic command
- [docs/help/index.md](../help/index.md) — Help hub

---

## Complete Installation Checklist

### Phase 1: User Account Restructuring

- [ ] Create mattias user with sudo access
- [ ] Copy SSH keys to mattias
- [ ] Test SSH login as mattias (from Windows)
- [ ] Set password for mattias

### Phase 2: System Preparation

- [ ] Update system
- [ ] Enable unattended-upgrades
- [ ] Set timezone to Europe/Brussels
- [ ] Configure UFW firewall
- [ ] Set static IP on router

### Phase 3: Tailscale

- [ ] Install Tailscale on server
- [ ] Authorize in Tailscale admin console
- [ ] Install Tailscale on Windows PCs (jaguar, bear)
- [ ] Enable MagicDNS in admin console
- [ ] Test: `ssh jeff@jeff-server` from tailnet

### Phase 4: Claude Code

- [ ] Install Claude Code
- [ ] Authenticate

### Phase 5: Docker Setup

- [ ] Install Docker
- [ ] Add jeff to docker group
- [ ] Enable Docker on boot
- [ ] Log out/in for group membership
- [ ] Verify with `docker run hello-world`

### Phase 6: OpenClaw Installation

- [ ] Run installer: `curl -fsSL https://openclaw.bot/install.sh | bash`
- [ ] Verify Node.js version: `node --version` (must be v22.12.0+)
- [ ] Configure npm security: `npm config set ignore-scripts true`
- [ ] Complete onboarding wizard
- [ ] Enable lingering: `sudo loginctl enable-linger jeff`
- [ ] Verify service: `systemctl --user status openclaw`

### Phase 7: Sandbox Setup

- [ ] Build sandbox image: `scripts/sandbox-setup.sh`
- [ ] Build browser sandbox: `scripts/sandbox-browser-setup.sh`
- [ ] Configure sandbox network if needed (default: no network)
- [ ] Fix permissions if needed

### Phase 8: Security Hardening

- [ ] Set file permissions (700/600)
- [ ] Generate gateway token
- [ ] Create ~/.openclaw/.env with secrets
- [ ] Apply secure config
- [ ] Run: `openclaw security audit --deep --fix`

### Phase 9: Telegram Channel Setup

- [ ] Get Telegram user ID from @userinfobot
- [ ] Create bot via @BotFather
- [ ] Add bot token to .env
- [ ] Configure channel in openclaw.json
- [ ] Restart and test messaging

### Phase 10: Tailscale Serve

- [ ] Update config with `tailscale: { mode: "serve" }`
- [ ] Restart gateway: `systemctl --user restart openclaw`
- [ ] Test HTTPS access: `https://jeff-server.<tailnet>.ts.net/`
- [ ] Configure NextDNS in Tailscale admin (DNS security)

### Phase 11: Finalization

- [ ] Remove sudo from jeff: `sudo deluser jeff sudo`
- [ ] Optionally lock jeff password
- [ ] Verify everything works
- [ ] Unplug monitor/keyboard

---

## Day-to-Day Commands Cheatsheet

### Connecting

```bash
# From home (local network) as admin
ssh mattias@192.168.0.141

# From anywhere (Tailscale) as admin
ssh mattias@jeff-server

# Switch to bot user
sudo -i -u jeff
```

### OpenClaw Management

```bash
systemctl --user status openclaw    # Check status
journalctl --user -u openclaw -f    # View logs
systemctl --user restart openclaw   # Restart
systemctl --user stop openclaw      # Stop
systemctl --user start openclaw     # Start
openclaw status --deep              # Detailed health check
openclaw doctor --fix               # Diagnose and repair
```

### System Maintenance

```bash
sudo apt update && sudo apt upgrade -y  # Update
df -h                                    # Disk space
free -h                                  # Memory
htop                                     # Processes (install: sudo apt install htop)
sudo reboot                              # Reboot
sudo shutdown now                        # Shutdown
```

### Tailscale

```bash
tailscale status    # Check status
tailscale ip -4     # Get IP
```

### Security

```bash
openclaw security audit --deep    # Security audit
sudo ufw status                   # Firewall status
```

### Logs

```bash
cat /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

### References

**Official Documentation:**

- [Gateway Runbook](https://docs.openclaw.ai/gateway) — Gateway lifecycle and operations

**Local Repository Files:**

- [docs/gateway/index.md](../gateway/index.md) — Gateway service runbook
- [docs/cli/index.md](../cli/index.md) — CLI reference overview
- [docs/gateway/logging.md](../gateway/logging.md) — Gateway logging

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Can't SSH in | Check IP (`ip addr`), firewall (`sudo ufw status`) |
| OpenClaw not running | `systemctl --user status openclaw`, check logs |
| Tailscale not connecting | `tailscale status`, `sudo systemctl status tailscaled` |
| Disk full | `df -h`, `sudo apt autoremove` |
| After power outage | Server auto-boots, services auto-start |
| Systemd user services unavailable | Enable lingering, re-login |
| Docker permission denied | Verify jeff in docker group, re-login |
| Sandbox permission errors | `chown -R 1000:1000 ~/.openclaw/agents/` |
| Telegram bot stops responding | IPv6 issue: check `dig +short api.telegram.org AAAA`, force IPv4 if needed |
| Config validation fails | Run `openclaw doctor --fix` to diagnose and repair |
| Node.js version too old | Must be v22.12.0+, run installer again or use NodeSource |

### References

**Official Documentation:**

- [Troubleshooting](https://docs.openclaw.ai/gateway/troubleshooting) — Common gateway issues and solutions
- [Help](https://docs.openclaw.ai/help) — Help hub with troubleshooting links

**Local Repository Files:**

- [docs/gateway/troubleshooting.md](../gateway/troubleshooting.md) — Gateway troubleshooting guide
- [docs/gateway/doctor.md](../gateway/doctor.md) — Gateway doctor diagnostic tool
- [docs/channels/troubleshooting.md](../channels/troubleshooting.md) — Channel troubleshooting
- [docs/help/index.md](../help/index.md) — Help hub
- [docs/debugging.md](../debugging.md) — Debugging guide

---

## Gotchas & Tips

1. **Node.js version:** Use Node.js **22.12.0 or later** (22.x LTS). This version includes critical security patches. Do NOT use Bun (known issues with Telegram).

2. **Config validation:** OpenClaw refuses to start with invalid config. Run `openclaw doctor` before restarting.

3. **Per-agent auth:** New agents don't inherit API keys. Copy `auth-profiles.json` or re-run onboarding.

4. **Session scoping:** Use `session.dmScope: "per-channel-peer"` to prevent context leakage between users.

5. **Minimal PATH in service:** The daemon intentionally runs with minimal PATH for security. Put runtime variables in `~/.openclaw/.env`.

6. **Prompt injection:** Not solved. Treat web results, attachments, and emails as potentially hostile.

7. **Telegram config key:** Use `botToken` (not `token`) in the Telegram channel configuration.

8. **Sandbox network:** Default is `"none"` (no network). Enable `docker.network: "bridge"` if tools need internet access.

9. **NPM security:** Run `npm config set ignore-scripts true` to disable lifecycle scripts (primary supply chain attack vector).

### References

**Official Documentation:**

- [Configuration](https://docs.openclaw.ai/gateway/configuration) — Full configuration reference
- [Sessions](https://docs.openclaw.ai/concepts/session) — Session scoping documentation
- [Telegram Channel](https://docs.openclaw.ai/channels/telegram) — Telegram-specific configuration
- [Sandboxing](https://docs.openclaw.ai/gateway/sandboxing) — Sandbox network configuration

**Local Repository Files:**

- [docs/gateway/configuration.md](../gateway/configuration.md) — Complete configuration reference
- [docs/concepts/session.md](../concepts/session.md) — Session management
- [docs/channels/telegram.md](../channels/telegram.md) — Telegram channel documentation
- [docs/gateway/sandboxing.md](../gateway/sandboxing.md) — Sandboxing documentation
- [docs/install/bun.md](../install/bun.md) — Bun runtime notes (and why not to use it with Telegram)

---

## Backups

```bash
# Backup OpenClaw data
tar -czvf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/

# Copy to another machine
scp openclaw-backup-*.tar.gz mattias@other-pc:/backups/
```

### References

**Local Repository Files:**

- [docs/concepts/agent-workspace.md](../concepts/agent-workspace.md) — What's in the agent workspace (what to back up)

---

## Security Research Notes

Answers to questions from the security research document:

### LUKS/BIOS Password

Not required for remote attack protection. These are physical security measures. Low priority unless you're concerned about physical theft.

### NPM Supply Chain Security

Not built into OpenClaw. Manually configure: `npm config set ignore-scripts true` (added to Phase 2).

### Dual-LLM Pattern (Reader/Doer)

Not built-in, but achievable with multi-agent configuration. Configure a read-only agent with restricted tools, then manually pass sanitized output to a privileged agent.

### Secret Injection (1Password/Bitwarden)

Not built-in. For runtime injection, modify the systemd unit:

```bash
ExecStart=/usr/bin/op run -- %h/.npm-global/bin/openclaw gateway
```

Requires 1Password CLI (`op`) installed and configured.

### DNS Security

Use NextDNS with Tailscale (native integration in Tailscale admin console). Provides DNS tunneling protection and newly-registered domain blocking without needing OpenSnitch.

### Egress Filtering

Sandbox `docker.network: "none"` (default) provides complete egress blocking for sandboxed sessions. For host-level filtering, install OpenSnitch manually.

### References

**Official Documentation:**

- [Gateway Security](https://docs.openclaw.ai/gateway/security) — Comprehensive security documentation
- [Multi-Agent Routing](https://docs.openclaw.ai/concepts/multi-agent) — Multi-agent configuration for dual-LLM patterns

**Local Repository Files:**

- [docs/cli/security.md](../cli/security.md) — `openclaw security` command reference
- [SECURITY.md](../../SECURITY.md) — Security policies
- [docs/concepts/multi-agent.md](../concepts/multi-agent.md) — Multi-agent setup documentation
- [docs/gateway/sandboxing.md](../gateway/sandboxing.md) — Sandbox egress filtering
- [skills/1password/references/get-started.md](../../skills/1password/references/get-started.md) — 1Password CLI integration
- [SECURITY.md](../../SECURITY.md) — Security policies
