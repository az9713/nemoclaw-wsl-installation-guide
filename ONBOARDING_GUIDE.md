# NemoClaw Onboarding Guide: Windows 11 via WSL2

**Platform:** Windows 11 Home (build 10.0.26200) with WSL2 (Ubuntu 24.04 LTS)
**Last Updated:** March 2026

This guide documents every problem encountered running `nemoclaw onboard` after a successful installation, explains the root causes, and provides a clean path that avoids all of them.

---

## Table of Contents

1. [TL;DR — One-Shot Onboarding](#tldr--one-shot-onboarding)
2. [Problems Encountered](#problems-encountered)
   - [Problem 1: Docker Desktop Not Running](#problem-1-docker-desktop-not-running)
   - [Problem 2: OpenShell CLI Installation Failed with CRLF Errors](#problem-2-openshell-cli-installation-failed-with-crlf-errors)
   - [Problem 3: cgroup v2 Not Configured for Docker Desktop](#problem-3-cgroup-v2-not-configured-for-docker-desktop)
   - [Problem 4: NVIDIA API Key Prompt During setup-spark](#problem-4-nvidia-api-key-prompt-during-setup-spark)
   - [Problem 5: False GPU Detection (Docker Desktop Virtual GPU)](#problem-5-false-gpu-detection-docker-desktop-virtual-gpu)
   - [Problem 6: HTTP 403 When Chatting (API Key Not Reaching the Sandbox)](#problem-6-http-403-when-chatting-api-key-not-reaching-the-sandbox)
   - [Problem 7: Telegram Bridge — "Agent exited with code 255"](#problem-7-telegram-bridge--agent-exited-with-code-255)
3. [Quick Reference Table](#quick-reference-table)
4. [The False GPU Detection Problem — Deep Dive](#the-false-gpu-detection-problem--deep-dive)

---

## TL;DR — One-Shot Onboarding

> **The core insight for onboarding:** Docker Desktop exposes a virtual GPU through WSL2 that `nvidia-smi` reports as a real 4096 MB NVIDIA device. NemoClaw's GPU detection picks this up and tries to create a GPU sandbox, which the gateway rejects. The sandbox appears to be created successfully but does not actually exist. Use `NEMOCLAW_NO_GPU=1` to bypass false GPU detection entirely.

This section gives you the complete pre-flight checklist and the exact command sequence. Do every step in order before running `nemoclaw onboard`.

### Step 1: Start Docker Desktop

Open Docker Desktop from the Windows Start menu or system tray. Wait until the whale icon in the taskbar shows a green "running" state. Do not proceed until Docker Desktop is fully started.

> **Docker Desktop must remain running for the entire NemoClaw lifecycle** — during onboarding, during usage, and whenever you run any `nemoclaw` command. Closing Docker Desktop will cause all NemoClaw commands to fail with "Docker is not running."

### Step 2: Configure Docker Desktop for cgroup v2

Ubuntu 24.04 uses cgroup v2. Docker Desktop must be explicitly configured to use the host cgroup namespace, or the gateway's k3s process will fail to start.

1. Open Docker Desktop > **Settings** > **Docker Engine**
2. Locate the JSON configuration block
3. Add `"default-cgroupns-mode": "host"` to the root of the JSON object

The resulting JSON should look similar to:

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "default-cgroupns-mode": "host"
}
```

4. Click **Apply & Restart** and wait for Docker to come back up

> **Do not use `nemoclaw setup-spark` or `systemctl restart docker` to apply this fix.** Docker Desktop does not use systemd. The `docker.service` unit does not exist in this environment, and `nemoclaw setup-spark` will fail trying to restart it.

### Step 3: Fix Line Endings in the NemoClaw Repository

If the NemoClaw repository was cloned using Windows git, all shell scripts have CRLF line endings and will fail when executed by WSL bash. Fix this before running onboard:

```bash
# In WSL
sudo apt-get install -y dos2unix

find /mnt/c/Users/simon/Downloads/nemoclaw/NemoClaw/scripts -type f -exec dos2unix {} +
dos2unix /mnt/c/Users/simon/Downloads/nemoclaw/NemoClaw/bin/*
```

### Step 4: Run Onboarding with NEMOCLAW_NO_GPU=1

```bash
# In WSL
NEMOCLAW_NO_GPU=1 nemoclaw onboard
```

The `NEMOCLAW_NO_GPU=1` environment variable overrides false GPU detection. Even though `nvidia-smi` reports a GPU, this flag forces NemoClaw to treat the system as GPU-less and create a standard sandbox without the `--gpu` flag.

### What to Type at Each Prompt

The onboarding wizard has 7 steps. Here is what to expect and enter at each prompt:

| Step | Prompt | What to Enter |
|------|--------|---------------|
| 3 — Creating sandbox | `Sandbox name [my-assistant]:` | Press Enter to accept default, or type a name such as `nemo-1` |
| 3 — Creating sandbox | `Sandbox 'X' already exists. Recreate? [y/N]:` | Type `y` if you are re-onboarding after a failed attempt |
| 4 — Configuring inference | `Choose [2]:` | Press Enter to accept Cloud API (option 2), or type `2` explicitly |
| 4 — Configuring inference | NVIDIA API key prompt | Paste your key from [build.nvidia.com](https://build.nvidia.com) |
| 7 — Policy presets | `Apply suggested presets (pypi, npm)? [Y/n/list]:` | Press Enter to accept, or type `n` to skip |

After all 7 steps complete, connect to your sandbox:

```bash
nemoclaw <sandbox-name> connect
```

---

## Problems Encountered

### Problem 1: Docker Desktop Not Running

**Symptom:**

```
Error: Docker is not running. Please start Docker and try again.
```

This appeared immediately at Step 1 (Preflight checks) when `nemoclaw onboard` was run.

**Root Cause:**

NemoClaw uses Docker for all sandbox operations. Every command — from preflight through sandbox creation — requires Docker to be running. The check at the start of `onboard.js` calls `docker info` and exits immediately if it fails.

**Solution:**

Start Docker Desktop from the Windows Start menu. Look for the whale icon in the system tray. Once it shows a green running indicator, re-run `nemoclaw onboard`.

> **Important:** Docker Desktop must remain running throughout the entire NemoClaw lifecycle. It is not sufficient to start it just for onboarding — it must be running whenever any `nemoclaw` command is executed. If you close Docker Desktop, all subsequent nemoclaw commands will fail with the same error.

---

### Problem 2: OpenShell CLI Installation Failed with CRLF Errors

**Symptom:**

Step 1 attempted to install the OpenShell CLI automatically (because `openshell` was not yet on the PATH). The installation failed with:

```
./install.sh: line 2: $'\r': command not found
./install.sh: line 5: set: pipefail: invalid option
```

**Root Cause:**

The NemoClaw repository was cloned using Windows git, which has `core.autocrlf = true` set by default. This causes git to convert all Unix line endings (`\n`, LF) to Windows line endings (`\r\n`, CRLF) on checkout.

When WSL bash tries to execute these scripts, the `\r` character at the end of each line is treated as an unknown token. The `set -euo pipefail\r` line fails because bash sees the option as `pipefail\r`, which does not exist. Every subsequent line that starts a command also fails.

This affects every shell script in the repository — including `scripts/install.sh`, which is the script that `onboard.js` calls to install OpenShell:

```javascript
// From onboard.js — installOpenshell()
run(`bash "${path.join(SCRIPTS, "install.sh")}"`, { ignoreError: true });
```

**Solution:**

Install `dos2unix` and convert all scripts before running onboard:

```bash
sudo apt-get install -y dos2unix
find /mnt/c/Users/simon/Downloads/nemoclaw/NemoClaw/scripts -type f -exec dos2unix {} +
dos2unix /mnt/c/Users/simon/Downloads/nemoclaw/NemoClaw/bin/*
```

After running `dos2unix`, re-run `nemoclaw onboard`. The OpenShell CLI installed successfully on the second attempt.

> **Note:** This fix must be applied before running `nemoclaw onboard`, not after the failure. Once the scripts are converted, onboarding can detect that OpenShell is not installed and complete the installation without errors.

---

### Problem 3: cgroup v2 Not Configured for Docker Desktop

**Symptom:**

Step 1 failed with:

```
!! cgroup v2 detected but Docker is not configured for cgroupns=host.
   OpenShell's gateway runs k3s inside Docker, which will fail with:

     openat2 /sys/fs/cgroup/kubepods/pids.max: no such file or directory

   To fix, run:

     nemoclaw setup-spark
```

When `nemoclaw setup-spark` was run to apply the fix, it failed with:

```
Failed to restart docker.service: Unit docker.service not found.
```

**Root Cause — Part A (Why the check fails):**

Ubuntu 24.04 uses the cgroup v2 unified hierarchy by default. NemoClaw's gateway runs k3s inside a Docker container. k3s needs access to the host cgroup namespace to manage cgroup hierarchies for the pods it creates. Without `"default-cgroupns-mode": "host"` in Docker's daemon configuration, the k3s kubelet fails because it cannot find the cgroup paths it needs.

The `preflight.js` check detects this by reading `/etc/docker/daemon.json` and verifying the `default-cgroupns-mode` key is set to `"host"`. In a fresh Docker Desktop installation, this key is not present, so the check fails.

**Root Cause — Part B (Why `nemoclaw setup-spark` fails):**

The `setup-spark` command is designed for native Linux environments where Docker is managed by systemd. It attempts to restart Docker after writing the config change:

```bash
sudo systemctl restart docker
```

Docker Desktop on WSL2 is not managed by systemd. The Docker daemon runs inside the Docker Desktop Windows application — not as a Linux service. There is no `docker.service` systemd unit. Running `systemctl restart docker` therefore fails with "Unit docker.service not found."

**Solution:**

Apply the cgroup configuration manually through the Docker Desktop UI:

1. Open Docker Desktop
2. Go to **Settings** > **Docker Engine**
3. Add `"default-cgroupns-mode": "host"` to the JSON configuration
4. Click **Apply & Restart**

This is the only supported way to change Docker daemon configuration when using Docker Desktop. Do not use `systemctl`, `service docker restart`, or `nemoclaw setup-spark` in this environment.

> **Checklist before running `nemoclaw onboard`:**
> - Docker Desktop is running
> - `"default-cgroupns-mode": "host"` is present in Docker Engine JSON
> - Docker has been restarted after adding that setting
> - You can verify with: `docker info | grep -i cgroup` — it should show `cgroupns: host`

---

### Problem 4: NVIDIA API Key Prompt During setup-spark

**Symptom:**

Before attempting the cgroup fix, `nemoclaw setup-spark` prompted unexpectedly for an NVIDIA API key:

```
Enter your NVIDIA API key (from https://build.nvidia.com): _
```

The key was entered, then the command failed on the `systemctl restart docker` line (as described in Problem 3 above).

**Root Cause:**

`nemoclaw setup-spark` combines two responsibilities: writing the Docker cgroup configuration and configuring the NVIDIA API credentials. It prompts for the API key first, then attempts the Docker restart. Because the Docker restart fails, the overall command fails — but the API key has already been written to disk.

**Outcome:**

The API key is saved to `~/.nemoclaw/credentials.json` with permissions `600` (owner read/write only). This file persists across onboarding attempts. When `nemoclaw onboard` later reaches Step 4 (Configuring inference) and calls `ensureApiKey()`, it reads the saved key and does not prompt again.

**What this means for you:**

- If you ran `setup-spark` and entered your API key, the key is already saved. You will not be prompted for it again during `nemoclaw onboard`.
- If you need to update the key, edit `~/.nemoclaw/credentials.json` directly, or run any command that calls `ensureApiKey()` after deleting that file.
- The cgroup fix from `setup-spark` still needs to be applied manually via the Docker Desktop UI, regardless of whether the key was saved.

---

### Problem 5: False GPU Detection (Docker Desktop Virtual GPU)

This was the most persistent and confusing problem. See also the [dedicated deep-dive section](#the-false-gpu-detection-problem--deep-dive) below.

**Symptom:**

Step 1 reported GPU detection despite the machine having no discrete NVIDIA GPU:

```
  ✓ NVIDIA GPU detected: 1 GPU(s), 4096 MB VRAM
```

The onboarding continued through all 7 steps and printed a success dashboard. However, running `nemoclaw nemo-1 connect` afterward failed:

```
Error: sandbox not found
```

**Root Cause:**

Docker Desktop on Windows exposes a virtual GPU device to WSL2. This device responds to `nvidia-smi` queries and reports 4096 MB of VRAM. NemoClaw's `detectGpu()` function in `nim.js` calls `nvidia-smi --query-gpu=memory.total` and interprets any output as a real GPU:

```javascript
// From nim.js — detectGpu()
const output = runCapture(
  "nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits",
  { ignoreError: true }
);
if (output) {
  const lines = output.split("\n").filter((l) => l.trim());
  const perGpuMB = lines.map((l) => parseInt(l.trim(), 10)).filter((n) => !isNaN(n));
  if (perGpuMB.length > 0) {
    return { type: "nvidia", count: perGpuMB.length, totalMemoryMB, nimCapable: true };
  }
}
```

Because `nvidia-smi` returns valid output for the virtual device, `detectGpu()` returns a GPU object with `nimCapable: true`. The `onboard()` function then passes this GPU object through to `createSandbox()`, which adds `--gpu` to the `openshell sandbox create` command.

The OpenShell gateway has no real allocatable GPUs. It rejects the `--gpu` request with an error. However, the create command's output is piped through `awk` to de-duplicate log lines, and the onboard script's error handling for this step uses `ignoreError: true` equivalently (the command exits non-zero but the script continues). The script then writes the sandbox to the local registry and prints:

```
  ✓ Sandbox 'nemo-1' created
```

The sandbox exists in the local registry file but was never successfully created in the gateway. Connecting to it fails because the gateway has no record of it.

**What did NOT work:**

- Running `nemoclaw onboard` again — hit the same GPU error every time; even though it asked about recreating the sandbox, the underlying `--gpu` problem remained
- Running `nemoclaw nemo-1 destroy` followed by re-onboarding — same result
- Choosing "NVIDIA Cloud API" (option 2) at the inference step — the GPU flag is passed at sandbox creation time in Step 3, before the inference choice in Step 4; changing the inference provider does not affect whether the sandbox is created with `--gpu`

**Solution:**

The `NEMOCLAW_NO_GPU` environment variable was added to `onboard.js` to allow overriding the GPU detection result. The variable is checked in the `onboard()` function immediately after `preflight()` returns:

```javascript
// From onboard.js — onboard()
let gpu = await preflight();
// Force disable GPU if NEMOCLAW_NO_GPU=1 (e.g. Docker Desktop virtual GPU with no real allocatable GPUs)
if (process.env.NEMOCLAW_NO_GPU === "1") {
  console.log("  ⓘ NEMOCLAW_NO_GPU=1 — forcing cloud inference (no GPU sandbox)");
  gpu = null;
}
```

With `gpu` set to `null`, neither `startGateway()` nor `createSandbox()` passes `--gpu` to any OpenShell command.

To apply this fix:

```bash
# 1. Destroy the broken sandbox (if one was registered from a previous failed attempt)
nemoclaw nemo-1 destroy

# 2. Re-run onboarding with the override
NEMOCLAW_NO_GPU=1 nemoclaw onboard
```

After running with `NEMOCLAW_NO_GPU=1`:
- Step 1 still reports the GPU detection, but immediately after prints the override message
- Step 2 starts the gateway without the `--gpu` flag
- Step 3 creates the sandbox without the `--gpu` flag
- The sandbox is successfully allocated in the gateway
- `nemoclaw nemo-1 connect` works and the OpenClaw TUI launches

---

### Problem 6: HTTP 403 When Chatting (API Key Not Reaching the Sandbox)

**Symptom:**

The OpenClaw TUI launches successfully and you can type messages, but every message returns:

```
HTTP 403: status code (no body)
```

or:

```
run error: 403 status code (no body)
```

The status bar shows the model name and connection status as normal, but no response is generated.

**Root Cause:**

The NVIDIA Cloud API is rejecting the inference request with a 403 Forbidden. This can happen for several reasons:

1. **Invalid or expired API key** — the key stored during onboarding may have been rotated, revoked, or entered incorrectly
2. **Key saved in the wrong location** — NemoClaw stores credentials in `~/.nemoclaw/credentials.json` on the host WSL filesystem, but the inference request goes through the **OpenShell gateway's provider**, which has its own copy of the key. Updating the local file does not update the gateway.
3. **Key entered under sudo** — if `nemoclaw setup-spark` was run with `sudo` before onboarding, the API key may have been saved to `/root/.nemoclaw/credentials.json` instead of `/home/<user>/.nemoclaw/credentials.json`. When onboarding later runs without sudo, it may not find the key and either prompt again or use a stale value.

**What does NOT work:**

```bash
# This updates the HOST file, but the gateway provider has its own copy
vi ~/.nemoclaw/credentials.json
```

Editing the local credentials file has no effect on the running gateway provider. The gateway reads the key when the provider is created or updated, not at request time from the filesystem.

**Solution:**

Update the API key directly in the OpenShell gateway provider:

```bash
# Exit the sandbox first (Ctrl+C to exit TUI, then 'exit' to leave sandbox)

# Update the provider credential in the gateway
openshell provider update nvidia-nim --credential "NVIDIA_API_KEY=nvapi-your-new-key-here"

# Reconnect and test
nemoclaw nemo-1 connect
openclaw tui
```

> **Note:** If you need a new API key, generate one at https://build.nvidia.com/settings/api-keys. The key starts with `nvapi-`.

**Verification:**

After updating the provider, send a message in the TUI. If the agent responds with actual content instead of a 403 error, the key is working. The first one or two messages after a key change may still return 403 if they were queued before the update — send a fresh message to confirm.

> **Tip:** To avoid this problem entirely, have your NVIDIA API key ready before starting onboarding. When Step 4 prompts for it, paste the correct key. It will be stored in both the local credentials file and the gateway provider in one step.

---

### Problem 7: Telegram Bridge — "Agent exited with code 255"

> **Status:** Partially resolved. The root cause was identified but full end-to-end Telegram functionality was not confirmed in this WSL2 + Docker Desktop environment.

**Symptom:**

After setting up the Telegram bridge and sending a message to the bot, the bot replies with:

```
Agent exited with code 255. Error: × status: NotFound ...
```

**Root Cause (identified):**

The Telegram bridge script (`scripts/telegram-bridge.js`) defaults to a sandbox named `"nemoclaw"`:

```javascript
const SANDBOX = process.env.SANDBOX_NAME || "nemoclaw";
```

If your sandbox has a different name (e.g. `nemo-1`), the bridge tries to connect to a non-existent sandbox called `nemoclaw`, which fails with "NotFound".

**Additional contributing factors (suspected):**

1. **Sandbox must be actively running** — the bridge uses `openshell sandbox ssh-config` to SSH into the sandbox. If the sandbox is not running or connected, this fails. You need `nemoclaw <name> connect` running in a separate terminal to keep the sandbox alive.

2. **WSL2 networking limitations** — the bridge makes HTTPS calls to the Telegram API (`api.telegram.org`) from WSL2. Depending on whether the MTU fix is still in effect and the state of Node.js SSL, these calls may fail silently.

3. **Multiple environment variables required** — the bridge requires both `TELEGRAM_BOT_TOKEN` and `NVIDIA_API_KEY` to be set. If `NVIDIA_API_KEY` is missing from the environment when `nemoclaw start` is called, the bridge process starts but cannot forward messages to the inference provider.

**Attempted fix:**

```bash
nemoclaw stop
export TELEGRAM_BOT_TOKEN="your-token"
export SANDBOX_NAME="nemo-1"
nemoclaw start
```

This sets the correct sandbox name. In a separate terminal:

```bash
nemoclaw nemo-1 connect
```

This keeps the sandbox running. However, the bridge still returned errors, suggesting additional connectivity or configuration issues specific to the WSL2 + Docker Desktop environment.

**What does work:**

The **OpenClaw TUI** (`openclaw tui` from inside the sandbox) works reliably. If Telegram integration is essential, consider:

- Running NemoClaw on a native Linux machine or VM where networking is straightforward
- Deploying to a remote GPU instance (`nemoclaw deploy`) which has native networking
- Using the Web UI at `http://127.0.0.1:18789/` (requires port forwarding to be active: `openshell forward start --background 18789 nemo-1`)

> **Note for future investigation:** The bridge log output is not written to a file by default, making debugging difficult. To capture logs, start the bridge manually:
> ```bash
> TELEGRAM_BOT_TOKEN="your-token" NVIDIA_API_KEY="your-key" SANDBOX_NAME="nemo-1" \
>   node /mnt/c/users/simon/Downloads/nemoclaw/NemoClaw/scripts/telegram-bridge.js 2>&1 | tee /tmp/telegram-bridge.log
> ```
> This captures all output to `/tmp/telegram-bridge.log` for inspection.

---

## Quick Reference Table

| # | Problem | Symptom | Root Cause | Solution |
|---|---------|---------|------------|----------|
| 1 | Docker Desktop not running | `Error: Docker is not running` at Step 1 | Docker Desktop not started before running `nemoclaw onboard` | Start Docker Desktop from Start menu; keep it running at all times |
| 2 | CRLF errors in OpenShell installer | `$'\r': command not found` and `invalid option: pipefail` | Windows git converts LF to CRLF; WSL bash cannot execute CRLF scripts | `sudo apt-get install -y dos2unix` then convert scripts and bin/ before onboarding |
| 3 | cgroup v2 not configured | `cgroup v2 detected but Docker is not configured for cgroupns=host` | Ubuntu 24.04 uses cgroup v2; Docker Desktop requires explicit `cgroupns=host` setting | Add `"default-cgroupns-mode": "host"` to Docker Engine JSON in Docker Desktop Settings, then Apply & Restart |
| 3b | `nemoclaw setup-spark` fails | `Unit docker.service not found` | Docker Desktop does not create a systemd unit; `setup-spark` assumes systemd | Never use `systemctl` or `setup-spark` for Docker Desktop — use the Docker Desktop UI only |
| 4 | Unexpected API key prompt | `nemoclaw setup-spark` asks for NVIDIA API key before failing | `setup-spark` combines credential storage and cgroup config in one command | Key is saved to `~/.nemoclaw/credentials.json`; no re-entry needed during onboarding |
| 5 | False GPU detection | Onboarding completes but `connect` fails with "sandbox not found" | Docker Desktop virtual GPU responds to `nvidia-smi`; NemoClaw creates GPU sandbox which gateway rejects; silent failure | Destroy existing sandbox, then run `NEMOCLAW_NO_GPU=1 nemoclaw onboard` |
| 6 | HTTP 403 when chatting | `HTTP 403: status code (no body)` in OpenClaw TUI | API key invalid, expired, or not propagated to the gateway provider | Run `openshell provider update nvidia-nim --credential "NVIDIA_API_KEY=nvapi-..."` from the host (outside the sandbox) |
| 7 | Telegram bridge fails | `Agent exited with code 255` in Telegram | Bridge defaults to sandbox name `"nemoclaw"` instead of your actual sandbox name; sandbox may not be running | Set `SANDBOX_NAME=nemo-1`, keep sandbox connected in separate terminal; **not fully resolved on WSL2** |

---

## The False GPU Detection Problem — Deep Dive

This section gives a full account of the false GPU problem because it is the most confusing issue: the onboarding wizard appears to succeed, but the sandbox does not exist.

### Why Docker Desktop Exposes a Virtual GPU

Docker Desktop for Windows uses WSL2 as its backend. To support GPU-accelerated containers (for users who do have a real NVIDIA GPU), Docker Desktop passes a virtual GPU device through to the WSL2 environment. This virtual device responds to `nvidia-smi` queries with plausible-looking output:

```
$ nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits
4096
```

This is not a real GPU. It is a virtualization layer. It has no allocatable compute resources. When Docker or OpenShell tries to actually schedule GPU work against it, the request fails.

### Why NemoClaw Cannot Distinguish It

NemoClaw's GPU detection (`detectGpu()` in `nim.js`) determines whether a GPU is present by calling `nvidia-smi` and parsing the VRAM output. If `nvidia-smi` returns numeric data, the function returns a GPU object with `nimCapable: true`. It does not perform any deeper validation — such as actually attempting to run a GPU container, or querying whether the device supports CUDA compute. The virtual device passes this shallow check.

### The Failure Chain

Here is the exact sequence of events during a failed onboarding:

1. **Step 1:** `detectGpu()` returns `{ type: "nvidia", count: 1, totalMemoryMB: 4096, nimCapable: true }`
2. **Step 1:** Console prints: `✓ NVIDIA GPU detected: 1 GPU(s), 4096 MB VRAM`
3. **Step 2:** `startGateway()` is called with the GPU object. Because `gpu.nimCapable` is true, it passes `--gpu` to `openshell gateway start`
4. **Step 3:** `createSandbox()` is called with the GPU object. Because `gpu.nimCapable` is true, it adds `--gpu` to `openshell sandbox create`
5. **Gateway:** The OpenShell gateway receives the `sandbox create --gpu` request and checks its resource pool. It has no allocatable GPUs. It returns an error: `"GPU sandbox requested, but the active gateway has no allocatable GPUs."`
6. **Step 3:** The error is printed to the terminal, but the command's non-zero exit is handled such that execution continues
7. **Step 3:** `registry.registerSandbox()` is called and writes the sandbox entry to the local registry file
8. **Step 3:** Console prints: `✓ Sandbox 'nemo-1' created`
9. **Steps 4–7:** Continue normally, writing inference config and policies against a sandbox name that has no backing in the gateway
10. **After onboarding:** `nemoclaw nemo-1 connect` calls `openshell sandbox connect nemo-1`, which asks the gateway. The gateway has no record of a sandbox named `nemo-1`. The command fails: `Error: sandbox not found`

### Why Changing the Inference Option Doesn't Help

A natural first troubleshooting step is to re-run onboarding and choose "NVIDIA Cloud API" instead of a local GPU option at Step 4. This does not fix the problem because:

- The inference choice at Step 4 affects which model and provider are configured
- The `--gpu` flag is passed at Step 3 (sandbox creation), which runs before Step 4
- The gateway rejection happens at Step 3 regardless of what inference option is later selected

The only way to prevent the GPU flag from being passed is to intervene before Step 3 — specifically, before `createSandbox()` is called. The `NEMOCLAW_NO_GPU=1` override does exactly this: it sets `gpu = null` immediately after `preflight()` returns, before any downstream function receives the GPU object.

### The Fix in onboard.js

The patch adds a check in the `onboard()` function between `preflight()` and `startGateway()`:

```javascript
async function onboard() {
  console.log("");
  console.log("  NemoClaw Onboarding");
  console.log("  ===================");

  let gpu = await preflight();
  // Force disable GPU if NEMOCLAW_NO_GPU=1 (e.g. Docker Desktop virtual GPU with no real allocatable GPUs)
  if (process.env.NEMOCLAW_NO_GPU === "1") {
    console.log("  ⓘ NEMOCLAW_NO_GPU=1 — forcing cloud inference (no GPU sandbox)");
    gpu = null;
  }
  await startGateway(gpu);
  const sandboxName = await createSandbox(gpu);
  // ...
}
```

Setting `gpu = null` causes both `startGateway()` and `createSandbox()` to skip their `if (gpu && gpu.nimCapable)` branches, so no `--gpu` flag is ever passed to OpenShell.

### Cleaning Up a Broken Sandbox Before Re-onboarding

If you already ran `nemoclaw onboard` without `NEMOCLAW_NO_GPU=1` and have a broken sandbox entry, clean it up first:

```bash
# Remove the broken sandbox (this clears the local registry entry)
nemoclaw <sandbox-name> destroy

# Verify the sandbox is gone
nemoclaw list
```

Then re-run with the override:

```bash
NEMOCLAW_NO_GPU=1 nemoclaw onboard
```

### How to Know the Fix Worked

During onboarding with `NEMOCLAW_NO_GPU=1`, you will see this line printed after the GPU detection message:

```
  ⓘ NEMOCLAW_NO_GPU=1 — forcing cloud inference (no GPU sandbox)
```

After onboarding completes, run:

```bash
nemoclaw <sandbox-name> connect
```

If the OpenClaw TUI launches successfully, the sandbox was created correctly. If you still see "sandbox not found", a broken registry entry may remain — run `nemoclaw list` and `nemoclaw <name> destroy` to clean up, then onboard again.

### Should NEMOCLAW_NO_GPU=1 Be Permanent?

Set it in your shell profile if you will always be running on a machine without a real allocatable GPU:

```bash
echo 'export NEMOCLAW_NO_GPU=1' >> ~/.bashrc
source ~/.bashrc
```

If you later move to a machine with a real NVIDIA GPU and want to use local NIM inference, remove this line and re-onboard.
