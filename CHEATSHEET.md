# NemoClaw WSL2 Cheat Sheet

**The essential command sequences for NemoClaw on Windows 11 + WSL2 + Docker Desktop.**
Every command here accounts for known WSL2 pitfalls. Copy-paste with confidence.

---

## Prerequisites (one-time)

Before any of the workflows below, ensure these are done:

| Prerequisite | How to verify | How to fix |
|---|---|---|
| Docker Desktop running | Whale icon green in system tray | Start from Windows Start menu |
| WSL Ubuntu integration | Docker Desktop > Settings > Resources > WSL Integration > Ubuntu toggled on | Toggle on, Apply & restart |
| cgroup v2 configured | Docker Desktop > Settings > Docker Engine > JSON has `"default-cgroupns-mode": "host"` | Add it, Apply & restart |
| dos2unix installed | `which dos2unix` | `sudo apt-get install -y dos2unix` |
| NemoClaw scripts converted | No `\r` errors when running nemoclaw | `find /mnt/c/.../NemoClaw/scripts -type f -exec dos2unix {} +` and `dos2unix /mnt/c/.../NemoClaw/bin/*` |
| NVIDIA API key ready | https://build.nvidia.com/settings/api-keys | Generate one (starts with `nvapi-`) |

---

## 1. Fresh Install (first time only)

Run in **Windows PowerShell**:

```powershell
cd $env:USERPROFILE\Downloads
git clone https://github.com/NVIDIA/NemoClaw.git
cd NemoClaw
npm install --ignore-scripts
```

Run in **WSL Ubuntu**:

```bash
# Fix MTU (fixes system SSL)
sudo ip link set dev eth0 mtu 1400

# Install dos2unix and fix line endings
sudo apt-get install -y dos2unix
cd /mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')/Downloads/NemoClaw
find scripts -type f -exec dos2unix {} + 2>/dev/null
dos2unix bin/* 2>/dev/null

# Configure git HTTPS (avoids SSH key requirement)
git config --global url."https://github.com/".insteadOf "git+ssh://git@github.com/"

# Install NemoClaw globally
sudo npm link

# Verify
nemoclaw --help
```

---

## 2. Onboarding (first time only)

```bash
# CRITICAL: NEMOCLAW_NO_GPU=1 prevents false GPU detection from Docker Desktop
NEMOCLAW_NO_GPU=1 nemoclaw onboard
```

**Prompts and what to type:**

| Step | Prompt | Type |
|------|--------|------|
| [1/7] Preflight | (no prompt) | Wait |
| [2/7] Gateway | (no prompt) | Wait (takes 1-2 min) |
| [3/7] Sandbox name | `Sandbox name [my-assistant]:` | `nemo-1` (or any name) |
| [4/7] Inference | `Choose [1]:` or `Choose [2]:` | Pick **NVIDIA Cloud API** option |
| [4/7] API key | `NVIDIA API Key:` | Paste your `nvapi-...` key |
| [5/7] Provider | (no prompt) | Wait |
| [6/7] OpenClaw | (no prompt) | Wait |
| [7/7] Presets | `Apply suggested presets? [Y/n/list]:` | `Y` (or `list` then `pypi,npm,slack,telegram`) |

---

## 3. Daily Use — Connect and Chat

```bash
# Step 1: Ensure MTU is fixed (resets on WSL restart)
sudo ip link set dev eth0 mtu 1400

# Step 2: Connect to sandbox
nemoclaw nemo-1 connect

# Step 3: Start chat (inside sandbox)
openclaw tui --session $(date +%Y%m%d)
```

> Using `--session $(date +%Y%m%d)` gives you a clean session per day, avoiding stale 403 errors from previous sessions.

---

## 4. Reconnect After Reboot

```bash
# 1. Start Docker Desktop (from Windows Start menu, wait for green whale)

# 2. Open WSL terminal and fix MTU
sudo ip link set dev eth0 mtu 1400

# 3. Try connecting directly
nemoclaw nemo-1 connect

# 4. If step 3 fails with gateway errors, restart the gateway first:
openshell gateway start --name nemoclaw
nemoclaw nemo-1 connect

# 5. Inside the sandbox:
openclaw tui --session $(date +%Y%m%d)
```

---

## 5. Fix API Key (403 Errors)

If you see `HTTP 403: status code (no body)` in the TUI:

```bash
# Exit TUI: Ctrl+C
# Exit sandbox: exit

# Update the key in the gateway (NOT the credentials file)
openshell provider update nvidia-nim --credential "NVIDIA_API_KEY=nvapi-your-new-key"

# Reconnect with a fresh session
nemoclaw nemo-1 connect
# then inside sandbox:
openclaw tui --session fresh
```

---

## 6. Telegram Bridge (experimental on WSL2)

> **Warning:** Telegram bridge has known issues on WSL2 + Docker Desktop. The TUI is more reliable.

**Terminal 1** — keep sandbox alive:

```bash
nemoclaw nemo-1 connect
# Leave this terminal open, don't type anything
```

**Terminal 2** — start the bridge:

```bash
export TELEGRAM_BOT_TOKEN="your-token-from-botfather"
export NVIDIA_API_KEY="nvapi-your-key"
export SANDBOX_NAME="nemo-1"
nemoclaw start
```

**Debug if it fails:**

```bash
# Run bridge manually with visible logs
nemoclaw stop
TELEGRAM_BOT_TOKEN="your-token" \
NVIDIA_API_KEY="nvapi-your-key" \
SANDBOX_NAME="nemo-1" \
  node /mnt/c/Users/simon/Downloads/NemoClaw/scripts/telegram-bridge.js 2>&1 | tee /tmp/telegram-bridge.log
```

---

## 7. Web UI

```bash
# From WSL (outside the sandbox)
openshell forward start --background 18789 nemo-1
```

Then open `http://127.0.0.1:18789/` in your Windows browser.

If it refuses to connect, ensure the sandbox is running (`nemoclaw nemo-1 connect` in another terminal).

---

## 8. Network Access — Let the Agent Use the Web

By default, the sandbox **blocks all outbound network requests** except domains allowed by your policy presets (pypi, npm, etc.). If the agent tries to reach a website (e.g. Hacker News), it will fail silently or fall back to mock data. This is by design — NemoClaw's value is that you control what the agent can access.

### Option A: Approve requests in real time (recommended)

Open a **second WSL terminal** while the agent is running:

```bash
openshell term
```

This shows a live feed of blocked network requests. When the agent tries to reach a URL, you see it and can approve or deny. This is NemoClaw's intended workflow — a human-in-the-loop firewall.

### Option B: Add policy presets

```bash
# See what presets are available
nemoclaw nemo-1 policy-list

# Add a preset
nemoclaw nemo-1 policy-add
```

### Option C: Tasks that don't need the web

If you just want to test the agent without network access, try prompts that work entirely within the sandbox:

```
Write a Python script that generates the first 20 Fibonacci numbers, saves them to fibonacci.json, runs it, and shows the results.
```

```
Create a simple Node.js HTTP server on port 3000, start it, curl it, then shut it down. Show me all the code and output.
```

```
Write a bash script that checks disk usage, memory, and running processes in this sandbox, then run it.
```

### Why the network is locked down

| Scenario | Claude Code | NemoClaw |
|----------|-------------|----------|
| Agent accesses any URL | Allowed (no restrictions) | Blocked unless you approve |
| Running untrusted prompts | Risk: agent could exfiltrate data | Safe: network policy prevents it |
| Audit trail of external calls | None | Every request logged and governed |
| Autonomous agent (Telegram/Slack) | No guardrails | You control exactly which domains it reaches |
| Self-hosted models on-prem | Data goes to Anthropic | Data stays on your network |

> **Bottom line:** For casual personal dev work, Claude Code is simpler. NemoClaw is for when the agent needs to run autonomously, handle untrusted input, or operate in regulated environments where you need to control and audit every external call.

---

## 9. Get Files Out of the Sandbox

```bash
# From WSL (outside the sandbox)
openshell sandbox cp nemo-1:/sandbox/myfile.py /mnt/c/Users/simon/Downloads/

# Or from inside the sandbox, just cat the file and copy-paste:
cat myfile.py
```

---

## 10. Destroy and Recreate

```bash
# Destroy
nemoclaw nemo-1 destroy

# Recreate (always use NEMOCLAW_NO_GPU=1 on WSL2 + Docker Desktop)
NEMOCLAW_NO_GPU=1 nemoclaw onboard
```

---

## 11. Stop Everything

```bash
# Stop Telegram bridge and services
nemoclaw stop

# Exit sandbox (if connected)
exit

# Stop the gateway (frees Docker resources)
openshell gateway destroy -g nemoclaw

# Optionally shut down WSL
wsl --shutdown
```

---

## Quick Troubleshooting

| Symptom | Fix |
|---------|-----|
| `SSL cipher` / `decryption failed` errors | `sudo ip link set dev eth0 mtu 1400` |
| `sandbox not found` on connect | `NEMOCLAW_NO_GPU=1 nemoclaw onboard` (GPU detection issue) |
| `HTTP 403` in TUI | `openshell provider update nvidia-nim --credential "NVIDIA_API_KEY=nvapi-..."` |
| `\r': command not found` | `dos2unix` on the affected script |
| `Docker is not running` | Start Docker Desktop, wait for green whale |
| `cgroup v2 detected` | Docker Desktop > Settings > Docker Engine > add `"default-cgroupns-mode": "host"` |
| `config prefix cannot be changed` | Work from `~` not `/mnt/c/`; use `nvm use --delete-prefix` |
| `Unit docker.service not found` | Don't use `systemctl` — manage Docker via Docker Desktop UI |
| Stale messages in TUI session | `openclaw tui --session new-name` |
| Gateway not responding after reboot | `openshell gateway start --name nemoclaw` |
| Telegram: `Agent exited with code 255` | Set `SANDBOX_NAME=nemo-1`; keep sandbox connected in separate terminal |
| Can't reach Web UI | `openshell forward start --background 18789 nemo-1` |

---

## Make MTU Fix Permanent

To avoid typing the MTU fix every time WSL starts:

```bash
# Add passwordless sudo for just the MTU command
echo "$USER ALL=(ALL) NOPASSWD: /sbin/ip link set dev eth0 mtu 1400" | sudo tee /etc/sudoers.d/wsl-mtu
sudo chmod 440 /etc/sudoers.d/wsl-mtu

# Add to bashrc
echo 'sudo ip link set dev eth0 mtu 1400 2>/dev/null' >> ~/.bashrc
```

Now the MTU is fixed automatically on every new terminal without a password prompt.
