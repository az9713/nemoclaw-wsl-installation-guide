# NemoClaw Installation Guide: Windows 11 via WSL2

**Platform:** Windows 11 Home (build 10.0.26200) with WSL2 (Ubuntu 24.04 LTS)
**Last Updated:** March 2026

---

## Table of Contents

1. [TL;DR — One-Shot Install Scripts](#tldr--one-shot-install-scripts)
2. [Prerequisites](#prerequisites)
3. [Recommended WSL Installation Flow (Step-by-Step)](#recommended-wsl-installation-flow)
4. [Detailed Problem Analysis](#detailed-problem-analysis)
   - [Problem 1: SSL Cipher Errors in WSL2](#problem-1-ssl-cipher-errors-in-wsl2)
   - [Problem 2: npm Permission Denied (EACCES)](#problem-2-npm-permission-denied-eacces)
   - [Problem 3: Windows .npmrc Conflicts](#problem-3-windows-npmrc-conflicts)
   - [Problem 4: Node.js Version Too Old](#problem-4-nodejs-version-too-old)
   - [Problem 5: npm Placeholder Package](#problem-5-npm-placeholder-package)
   - [Problem 6: Windows Line Endings (CRLF)](#problem-6-windows-line-endings-crlf)
   - [Problem 7: Docker Not Running / cgroup v2](#problem-7-docker-not-running--cgroup-v2)
   - [Problem 8: OpenShell Installation](#problem-8-openshell-installation)
5. [Quick Reference: Problems and Solutions](#quick-reference-problems-and-solutions)
6. [Background: How the Official Installer Actually Works](#background-how-the-official-installer-actually-works)

---

## TL;DR — One-Shot Install Scripts

> **The core insight:** WSL2 has broken SSL for both system curl and Node.js. Do ALL downloading on the Windows side, then install locally in WSL. Also fix Docker cgroup config BEFORE running nemoclaw.

> **Docker Desktop must be running and must stay running.** NemoClaw uses Docker for sandbox creation and management. Start Docker Desktop before you begin and do not close it — not during installation, not during onboarding, and not while using NemoClaw. If Docker Desktop is closed, NemoClaw commands will fail with "Docker is not running."

This is the fastest path. Two scripts, two terminals, zero detours.

### Part 1: Run in Windows PowerShell (downloads everything)

Open **Windows PowerShell** and paste this entire block:

```powershell
# --- Run this in Windows PowerShell ---

# 1. Clone NemoClaw
cd $env:USERPROFILE\Downloads
git clone https://github.com/NVIDIA/NemoClaw.git

# 2. Install npm dependencies (Windows npm has no SSL issues)
cd NemoClaw
npm install --ignore-scripts

# 3. Download Node.js 22 tarball for WSL
$nodeVersion = "v22.22.1"
$url = "https://nodejs.org/dist/$nodeVersion/node-$nodeVersion-linux-x64.tar.xz"
Invoke-WebRequest -Uri $url -OutFile "$env:USERPROFILE\Downloads\node-$nodeVersion-linux-x64.tar.xz"

# 4. Download Node.js checksum
Invoke-WebRequest -Uri "https://nodejs.org/dist/$nodeVersion/SHASUMS256.txt" `
    -OutFile "$env:USERPROFILE\Downloads\SHASUMS256.txt"

Write-Host "`n=== Windows side done. Now run Part 2 in WSL. ===" -ForegroundColor Green
```

Then configure Docker Desktop (do this NOW, before Part 2):

1. Open **Docker Desktop** > **Settings** > **Resources** > **WSL Integration** > toggle on **Ubuntu** > Apply & restart
2. Open **Docker Desktop** > **Settings** > **Docker Engine** > add `"default-cgroupns-mode": "host"` to the JSON > Apply & restart

### Part 2: Run in WSL Ubuntu terminal (installs everything)

Open your **WSL Ubuntu terminal** and paste this entire block:

```bash
# --- Run this in WSL Ubuntu ---

set -e
NODE_VERSION="v22.22.1"
DOWNLOADS="/mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')/Downloads"

echo "=== 1/6 Fix WSL2 MTU (fixes system SSL) ==="
sudo ip link set dev eth0 mtu 1400

echo "=== 2/6 Install dos2unix ==="
sudo apt-get install -y dos2unix

echo "=== 3/6 Install Node.js 22 via nvm ==="
# Install nvm if needed
if [ ! -d "$HOME/.nvm" ]; then
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
fi
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Place downloaded tarball in nvm cache
NVM_CACHE="$NVM_DIR/.cache/bin/node-${NODE_VERSION}-linux-x64"
mkdir -p "$NVM_CACHE"
cp "${DOWNLOADS}/node-${NODE_VERSION}-linux-x64.tar.xz" "$NVM_CACHE/"
grep "node-${NODE_VERSION}-linux-x64.tar.xz" "${DOWNLOADS}/SHASUMS256.txt" \
    | awk '{print $1}' > "$NVM_CACHE/node-${NODE_VERSION}-linux-x64.tar.xz.sha256"
nvm install 22
nvm use --delete-prefix v${NODE_VERSION} 2>/dev/null || nvm use 22
nvm alias default 22
echo "Node.js: $(node --version), npm: $(npm --version)"

echo "=== 4/6 Fix line endings ==="
cd "${DOWNLOADS}/NemoClaw"
find scripts -type f -exec dos2unix {} + 2>/dev/null
dos2unix bin/* 2>/dev/null

echo "=== 5/6 Configure git HTTPS and install NemoClaw ==="
git config --global url."https://github.com/".insteadOf "git+ssh://git@github.com/"
cd "${DOWNLOADS}/NemoClaw"
sudo npm link

echo "=== 6/6 Verify ==="
nemoclaw --help && echo -e "\n=== SUCCESS! Run 'nemoclaw onboard' to get started. ==="
```

That's it. After both scripts complete, run:

```bash
nemoclaw onboard
```

> **Why two scripts?** Windows networking works; WSL2 networking doesn't (for Node.js SSL). Part 1 uses Windows to download everything. Part 2 uses WSL to install from the local copies. This separation is the key insight that avoids every SSL error.

---

## Prerequisites

Before beginning, ensure the following are installed and configured on your Windows 11 system:

| Requirement | Minimum Version | Notes |
|---|---|---|
| Windows 11 | 10.0.22000+ | Home or Pro |
| WSL2 | Any current | With Ubuntu 22.04 LTS or 24.04 LTS |
| Docker Desktop | Latest stable | WSL integration must be enabled |
| Node.js (in WSL) | 20+ | Recommended: 22 LTS via nvm |
| Git for Windows | Any current | Required for Windows-side cloning |
| NVIDIA OpenShell | Latest | Installed automatically by `nemoclaw onboard` |

> **Note:** NemoClaw requires a Linux environment. On Windows, WSL2 is the supported path. macOS or native Linux users do not need this guide.

---

## Recommended WSL Installation Flow

This section gives you a clean, linear path through the installation that avoids every pitfall documented in this guide. Follow these steps in order from a fresh WSL2 + Docker Desktop setup.

### Step 1: Fix WSL2 Network MTU

WSL2's virtual network adapter defaults to an MTU that is too large (1430 bytes), which causes TLS packet fragmentation. This corrupts SSL handshakes for system-level tools.

Open your WSL2 Ubuntu terminal and run:

```bash
sudo ip link set dev eth0 mtu 1400
```

To make this persistent across WSL restarts, add it to your WSL profile:

```bash
echo 'sudo ip link set dev eth0 mtu 1400' >> ~/.bashrc
```

> **Note:** This MTU fix helps system libssl (used by curl, apt, etc.) but does NOT fix Node.js, which bundles its own OpenSSL. The steps below route around Node.js SSL entirely by using Windows-side networking for all downloads.

### Step 2: Clone the NemoClaw Repository via Windows Git

Do NOT clone inside WSL. Open **Windows PowerShell** (not WSL) and clone via Windows git, which has no SSL issues:

```powershell
# In Windows PowerShell
cd C:\Users\simon\Downloads
git clone https://github.com/NVIDIA/NemoClaw.git
```

This avoids the SSL errors that occur when cloning through WSL's Node.js or npm.

> **Warning:** Do not use `git clone git+ssh://git@github.com/nvidia/NemoClaw.git` on the Windows side unless your SSH keys are configured for Windows git. HTTPS is reliable here.

### Step 3: Install Node.js 22 via nvm (Using Windows-Downloaded Tarball)

WSL Ubuntu ships with Node.js 18, which is too old. The standard `nvm install 22` will fail due to SSL errors in WSL. Use Windows PowerShell to download the tarball, then let nvm use it from cache.

**3a. Install nvm in WSL (if not already installed):**

```bash
# In WSL
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
```

**3b. Download the Node.js 22 tarball from Windows PowerShell:**

```powershell
# In Windows PowerShell — check https://nodejs.org/dist/ for latest 22.x version
$nodeVersion = "v22.14.0"
$arch = "linux-x64"
$url = "https://nodejs.org/dist/$nodeVersion/node-$nodeVersion-$arch.tar.xz"
$dest = "$env:USERPROFILE\Downloads\node-$nodeVersion-$arch.tar.xz"
Invoke-WebRequest -Uri $url -OutFile $dest
Write-Host "Downloaded to: $dest"
```

**3c. Place the tarball in nvm's cache directory and install:**

```bash
# In WSL — adjust the version string to match what you downloaded
NODE_VERSION="v22.14.0"
NVM_CACHE="$HOME/.nvm/.cache/bin/node-${NODE_VERSION}-linux-x64"
mkdir -p "$NVM_CACHE"

# Copy from Windows Downloads to nvm cache
cp "/mnt/c/Users/simon/Downloads/node-${NODE_VERSION}-linux-x64.tar.xz" \
   "$NVM_CACHE/node-${NODE_VERSION}-linux-x64.tar.xz"

# nvm will find the cached file instead of downloading
nvm install 22
nvm use 22
nvm alias default 22
```

Verify:

```bash
node --version   # should print v22.x.x
npm --version    # should print 10.x.x or higher
```

### Step 4: Install npm Dependencies via Windows npm

Running `npm install` from inside WSL will fail on the SSL-affected npm operations. Instead, run it from **Windows PowerShell** inside the cloned repo directory:

```powershell
# In Windows PowerShell
cd C:\Users\simon\Downloads\NemoClaw
npm install --ignore-scripts
```

The `--ignore-scripts` flag prevents any shell scripts (which will have CRLF line endings on Windows) from running during installation. You will fix line endings in the next step.

### Step 5: Fix Windows Line Endings (CRLF to LF)

Git on Windows converts all line endings to CRLF. When WSL bash tries to execute these scripts, the carriage return `\r` character causes errors like `\r': command not found` and `invalid option: pipefail`.

Fix this from WSL before doing anything else with the repository:

```bash
# In WSL
sudo apt-get install -y dos2unix

# Navigate to the repo (it lives on the Windows filesystem, accessible via /mnt/c/)
cd /mnt/c/Users/simon/Downloads/NemoClaw

# Convert all scripts
find scripts -type f -exec dos2unix {} +
dos2unix bin/*
```

> **Warning:** If you skip this step, NemoClaw's install scripts and the OpenShell installer will fail with cryptic `command not found` errors that have nothing to do with the commands themselves.

### Step 6: Configure Git to Use HTTPS Instead of SSH

The official NemoClaw installer internally runs `npm install -g git+ssh://git@github.com/nvidia/NemoClaw.git`. If you do not have SSH keys configured for GitHub, this will fail silently. Tell git to rewrite SSH URLs to HTTPS:

```bash
# In WSL
git config --global url."https://github.com/".insteadOf "git+ssh://git@github.com/"
```

### Step 7: Install NemoClaw Globally via npm link

Now install NemoClaw into your WSL environment from the local clone:

```bash
# In WSL — from the NemoClaw repo directory
cd /mnt/c/Users/simon/Downloads/NemoClaw
sudo npm link
```

> **Note:** `npm link` creates a global symlink to the local package, which is equivalent to `npm install -g` but works from a local directory. Using `sudo` here is required because the global npm prefix writes to `/usr/lib/node_modules/`, which requires root.

Verify the installation:

```bash
nemoclaw --version
```

### Step 8: Configure Docker Desktop for WSL2 + cgroup v2

**8a. Enable WSL integration for Ubuntu in Docker Desktop:**

1. Open Docker Desktop
2. Go to **Settings > Resources > WSL Integration**
3. Enable the toggle for your Ubuntu distribution
4. Click **Apply & Restart**

**8b. Configure cgroupns mode:**

Ubuntu 24.04 uses cgroup v2. Without this setting, NemoClaw's `setup-spark` step will fail with: `cgroup v2 detected but Docker is not configured for cgroupns=host`.

1. Open Docker Desktop
2. Go to **Settings > Docker Engine**
3. Add the following to the JSON configuration:

```json
{
  "default-cgroupns-mode": "host"
}
```

The full config block should look similar to:

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

4. Click **Apply & Restart**

> **Warning:** Do not attempt to restart `docker.service` via systemd inside WSL. When using Docker Desktop, Docker is not managed by systemd. There is no `docker.service` unit. All Docker management must be done through the Docker Desktop application.

### Step 9: Run nemoclaw onboard

With everything configured, run the onboarding command:

```bash
# In WSL
nemoclaw onboard
```

This command installs NVIDIA OpenShell CLI and completes the NemoClaw setup. If it prompts to install OpenShell and the install script fails, check for CRLF issues in the OpenShell installer scripts and re-run `dos2unix` as needed (see [Problem 8](#problem-8-openshell-installation) for details).

---

## Detailed Problem Analysis

This section documents every problem encountered during the initial installation, including what was tried, why it failed, and what ultimately solved it.

### Problem 1: SSL Cipher Errors in WSL2

**Symptom:**

```
npm ERR! code ERR_SSL_CIPHER_OPERATION_FAILED
npm ERR! errno ERR_SSL_CIPHER_OPERATION_FAILED
```

This occurred when running `curl -fsSL https://nvidia.com/nemoclaw.sh | bash` — the curl succeeded but npm failed during the package fetch.

**What was tried and did NOT work:**

```bash
# Attempt 1: Legacy OpenSSL provider flag — no effect on npm SSL errors
export NODE_OPTIONS=--openssl-legacy-provider

# Attempt 2: Patch openssl.cnf — only partially helped curl, not npm
# Commented out: openssl_conf = openssl_init
sudo nano /etc/ssl/openssl.cnf

# Attempt 3: Disable strict SSL in npm — blocked by Windows .npmrc conflict
sudo npm config set strict-ssl false
# Error: "config prefix cannot be changed from project config"
```

**Root Cause:**

WSL2's virtual network adapter (`eth0`) has a default MTU of 1430 bytes. This is large enough to cause TLS record fragmentation when combined with the encapsulation overhead of the Hyper-V virtual network. Fragmented TLS records cause cipher operation failures on the receiving end.

This is a well-known and long-standing WSL2 bug. The MTU fix (`sudo ip link set dev eth0 mtu 1400`) resolves this for system-level tools that use the OS libssl (curl, apt, wget). However, Node.js ships with its own bundled OpenSSL and is NOT affected by system libssl changes, so the MTU fix alone is insufficient for npm.

**Final Solution:**

Bypass WSL networking entirely for all downloads. Use Windows PowerShell and Windows git (which use the Windows TLS stack and are unaffected) to download tarballs and clone repositories, then install from the local copies inside WSL.

---

### Problem 2: npm Permission Denied (EACCES)

**Symptom:**

```
npm ERR! code EACCES
npm ERR! syscall mkdir
npm ERR! path /usr/lib/node_modules
npm ERR! errno -13
```

**What was tried and did NOT work:**

Running the installer with elevated privileges:

```bash
sudo -E bash -c "$(curl -fsSL https://nvidia.com/nemoclaw.sh)"
```

This re-introduced SSL errors because `sudo` did not preserve the Node.js environment in a way that bypassed the bundled OpenSSL issue.

**Root Cause:**

When Node.js is installed system-wide via `apt` or `nodesource`, the global package directory (`/usr/lib/node_modules/`) is owned by root. Installing global packages without `sudo` therefore fails with permission denied. Running with `sudo` works at the filesystem level but, in this context, it also restored the SSL error path because the environment fixes were not carried through.

**Solution:**

Install from a local clone using `sudo npm link` after all downloads have been completed via Windows networking. This separates the "download" step (done safely via Windows) from the "install" step (done in WSL with sudo where networking is not involved).

---

### Problem 3: Windows .npmrc Conflicts

**Symptom:**

```
nvm is not compatible with the npm config "prefix" option: currently set to "C:\Users\simon\AppData\Roaming\npm"
Run `npm config delete prefix` or `nvm use --delete-prefix v22.x.x` to unset it.
```

And separately:

```
npm config set strict-ssl false
npm ERR! config prefix cannot be changed from project config
```

**Root Cause:**

npm walks up the directory tree looking for `.npmrc` files. When the working directory is anywhere on `/mnt/c/`, npm finds and reads `C:\Users\simon\.npmrc` (visible as `/mnt/c/users/simon/.npmrc`). This Windows `.npmrc` contains:

```
prefix=C:\Users\simon\AppData\Roaming\npm
```

This Windows-format path is incompatible with nvm's expectation of a Linux prefix path, causing nvm to refuse to operate. It also prevents npm config changes because npm treats the `.npmrc` found in the project tree as authoritative for the `prefix` setting.

**Solution:**

Always run WSL npm commands from inside the Linux home directory (`~/`) rather than from a path under `/mnt/c/`. When nvm complains about the prefix, use:

```bash
nvm use --delete-prefix 22
```

The cleanest solution is to clone the repo to the Linux filesystem or to do all npm operations from a non-Windows path. The workflow in this guide avoids this by using Windows npm for the `npm install` step and `sudo npm link` (which runs from a path that does not have a conflicting `.npmrc` in scope in the same way) for the global install.

---

### Problem 4: Node.js Version Too Old

**Symptom:**

```
Error: NemoClaw requires Node.js 20 or higher. Current version: v18.19.1
```

Ubuntu 24.04 LTS ships with Node.js 18.19.1 in its default apt repositories. NemoClaw requires Node.js 20+.

**What was tried and did NOT work:**

```bash
# Attempt 1: nodesource setup script — binary download failed with SSL errors
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -

# Attempt 2: nvm install 22 — the tarball download failed with SSL errors
nvm install 22
```

Both approaches attempt to download the Node.js binary over the network from within WSL, which hits the SSL issue.

**Solution:**

Download the Node.js tarball from Windows PowerShell, place it in nvm's local cache directory, and then run `nvm install 22`. nvm checks its cache before attempting a network download, so it finds the pre-placed file and installs without any network access. See [Step 3](#step-3-install-nodejs-22-via-nvm-using-windows-downloaded-tarball) in the recommended flow for the exact commands.

---

### Problem 5: npm Placeholder Package

**Symptom:**

```bash
npm install -g nemoclaw
# Installs successfully, but:
nemoclaw: command not found
# Or the command exists but does nothing / has no subcommands
```

**Root Cause:**

The `nemoclaw` package on the public npm registry (`https://www.npmjs.com/package/nemoclaw`) is an **empty placeholder**. It contains only a minimal `package.json` with no executable code. It exists to reserve the package name, not to distribute the software.

The actual NemoClaw software is installed from the private GitHub repository. The official installer script (`curl -fsSL https://nvidia.com/nemoclaw.sh | bash`) does NOT use the npm registry. It runs:

```bash
npm install -g git+ssh://git@github.com/nvidia/NemoClaw.git
```

**Solution:**

Always install NemoClaw from the GitHub repository, either by:

1. Letting the official installer run (if SSL issues are resolved), or
2. Cloning the repository manually and running `npm link` (as described in this guide).

> **Warning:** If you search for NemoClaw on npmjs.com and install it from there, you will get an empty package that does nothing. This is not a bug in your installation — it is simply the wrong package.

---

### Problem 6: Windows Line Endings (CRLF)

**Symptom:**

```bash
./install.sh: line 2: $'\r': command not found
./install.sh: line 5: set: pipefail: invalid option
```

Or variations like:

```
': not a valid identifier`export
```

**Root Cause:**

Git for Windows has `core.autocrlf = true` set by default. This causes git to convert all Unix line endings (`\n`, LF) to Windows line endings (`\r\n`, CRLF) on checkout. When a script cloned this way is executed by bash inside WSL, bash reads the `\r` as part of each command and token, treating it as an unknown character. The `\r` in `set -euo pipefail\r` makes bash see the option as `pipefail\r`, which does not exist.

This affects every shell script in the repository, including NemoClaw's own scripts and any scripts pulled in by its dependencies (such as the OpenShell installer).

**Solution:**

```bash
sudo apt-get install -y dos2unix

cd /mnt/c/Users/simon/Downloads/NemoClaw

# Fix all scripts in the scripts directory
find scripts -type f -exec dos2unix {} +

# Fix executables in bin/
dos2unix bin/*
```

To prevent this from recurring, you can configure Windows git to not convert line endings for this repository:

```powershell
# In Windows PowerShell, inside the repo directory
git config core.autocrlf false
git checkout -- .
```

> **Warning:** Do not set `core.autocrlf = false` globally on Windows without understanding the implications for other Windows-native projects that depend on CRLF line endings.

---

### Problem 7: Docker Not Running / cgroup v2

This problem had two distinct phases.

**Phase A — Docker not running**

**Symptom:**

```
Error: Docker is not running. Please start Docker and try again.
```

**Solution:** Start Docker Desktop from the Windows Start menu or system tray. Wait for the whale icon in the taskbar to show a green "running" state. Then, in Docker Desktop Settings > Resources > WSL Integration, enable the toggle for your Ubuntu distro and click Apply & Restart.

---

**Phase B — cgroup v2 not configured**

**Symptom:**

```
Error: cgroup v2 detected but Docker is not configured for cgroupns=host.
Please add "default-cgroupns-mode": "host" to your Docker daemon configuration.
```

This occurred during `nemoclaw setup-spark`.

**Root Cause:**

Ubuntu 22.04 and 24.04 use the cgroup v2 unified hierarchy by default. Some container workloads (including NemoClaw's Spark setup) require that Docker be told to use the host cgroup namespace rather than a private one. The relevant Docker daemon setting is `default-cgroupns-mode`.

Additionally, `nemoclaw setup-spark` attempted to restart the Docker daemon via:

```bash
sudo systemctl restart docker
```

This fails in a WSL2 + Docker Desktop setup because Docker Desktop does not register a systemd service. The `docker.service` unit does not exist. Docker is managed entirely by the Docker Desktop process running on the Windows host.

**Solution:**

1. Open Docker Desktop
2. Navigate to **Settings > Docker Engine**
3. Add `"default-cgroupns-mode": "host"` to the daemon JSON
4. Click **Apply & Restart**

Do not attempt to manage Docker via systemctl in a Docker Desktop environment.

---

### Problem 8: OpenShell Installation

**Symptom:**

During `nemoclaw onboard`, the OpenShell CLI installation step failed with the same CRLF-related errors as the NemoClaw scripts:

```
': not a valid identifier`export
/tmp/openshell-install.sh: line 3: $'\r': command not found
```

**Root Cause:**

The OpenShell installer scripts were either bundled with NemoClaw (and therefore affected by the Windows CRLF conversion) or downloaded to a temp location and executed. In either case, the scripts contained `\r\n` line endings.

**Solution:**

After running `dos2unix` on the NemoClaw repository as described in [Step 5](#step-5-fix-windows-line-endings-crlf-to-lf), re-run `nemoclaw onboard`. If the OpenShell installer is downloaded to a temporary directory, you may need to convert it separately:

```bash
# If the installer is placed in /tmp
find /tmp -name "*.sh" -newer /tmp -exec dos2unix {} + 2>/dev/null

# Or convert specific files if the path is shown in the error
dos2unix /tmp/openshell-install.sh
bash /tmp/openshell-install.sh
```

After dos2unix conversion, the OpenShell installation completes successfully.

---

## Quick Reference: Problems and Solutions

| Problem | Symptom | Root Cause | Solution |
|---|---|---|---|
| SSL Cipher Errors in WSL2 | `ERR_SSL_CIPHER_OPERATION_FAILED` during npm operations | WSL2 eth0 MTU too large (1430), causing TLS fragmentation; Node.js uses bundled OpenSSL unaffected by system fixes | Download all packages via Windows PowerShell/git; install from local copies in WSL |
| npm Permission Denied | `EACCES /usr/lib/node_modules` | System Node.js global prefix requires root; sudo + npm re-exposes SSL issue | Use `sudo npm link` after downloading deps via Windows npm |
| Windows .npmrc Conflicts | nvm refuses to work; `prefix cannot be changed from project config` | npm finds `/mnt/c/users/simon/.npmrc` when working in Windows filesystem paths | Work from Linux home dir; use `nvm use --delete-prefix`; run npm from non-Windows paths |
| Node.js Version Too Old | NemoClaw requires Node 20+; Ubuntu ships Node 18 | Default apt repositories do not have Node 20+; nvm/nodesource download fails via WSL SSL | Download Node 22 tarball via Windows PowerShell, place in nvm cache, run `nvm install 22` |
| npm Placeholder Package | `nemoclaw` installs from npm but has no commands | The npmjs.com package is an empty placeholder; real package is on GitHub | Clone from `github.com/NVIDIA/NemoClaw` and use `npm link`, not `npm install -g nemoclaw` |
| Windows Line Endings (CRLF) | `$'\r': command not found`; `invalid option: pipefail` | Windows git auto-converts LF to CRLF; WSL bash cannot execute scripts with `\r` in them | `sudo apt-get install dos2unix` then `find scripts -type f -exec dos2unix {} +` and `dos2unix bin/*` |
| Docker Not Running | `Docker is not running` | Docker Desktop not started or WSL integration not enabled | Start Docker Desktop; enable WSL integration for Ubuntu in Settings > Resources > WSL Integration |
| cgroup v2 not configured | `cgroup v2 detected but Docker is not configured for cgroupns=host` | Ubuntu 24.04 uses cgroup v2; Docker Desktop needs explicit cgroupns config | Add `"default-cgroupns-mode": "host"` to Docker Engine JSON in Docker Desktop Settings |
| systemctl docker restart fails | `Unit docker.service not found` | Docker Desktop does not create a systemd unit; docker is not managed by systemd in WSL | Restart Docker via Docker Desktop UI only; never use systemctl for Docker Desktop |
| OpenShell CRLF errors | `command not found` errors in OpenShell installer | OpenShell scripts also affected by Windows CRLF conversion | Run `dos2unix` on OpenShell installer scripts before execution |

---

## Background: How the Official Installer Actually Works

Understanding what the official installer does helps explain why several common attempts fail.

### The Official Installer Command

```bash
curl -fsSL https://nvidia.com/nemoclaw.sh | bash
```

### What the Script Actually Does

The install script does NOT use the npm registry. Internally, it runs:

```bash
npm install -g git+ssh://git@github.com/nvidia/NemoClaw.git
```

This clones the repository via SSH and installs it as a global npm package. This means:

1. **SSH keys are required** unless you configure the HTTPS rewrite (`git config --global url."https://github.com/".insteadOf "git+ssh://git@github.com/"`)
2. **The npm registry package is a placeholder** — do not install from `npmjs.com`
3. **Network access is required** at install time — which is what makes the WSL2 SSL issue so disruptive

### Why the WSL2 Workaround Works

The recommended flow in this guide sidesteps the installer entirely:

- Windows PowerShell handles all HTTPS downloads (no WSL SSL issues)
- Windows git handles the repository clone (no WSL SSL issues)
- Windows npm installs dependencies (no WSL SSL issues)
- WSL only handles the final `npm link` step, which is a local filesystem operation with no network access

This is a workaround, not a fix. If the underlying WSL2 SSL issue is resolved in a future Windows or WSL update, the standard installer should work directly.

### Verifying Your Installation

After completing all steps:

```bash
# Check NemoClaw is installed and accessible
nemoclaw --version

# Check Node.js version is acceptable
node --version    # must be 20+

# Check Docker is accessible from WSL
docker info | grep -i cgroup    # should show "cgroupns: host"

# Run the onboarding check
nemoclaw onboard --check
```
