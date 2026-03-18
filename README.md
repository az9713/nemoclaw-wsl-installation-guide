# NemoClaw WSL2 Installation Guide

A comprehensive guide for installing [NVIDIA NemoClaw](https://docs.nvidia.com/nemoclaw/latest/index.html) on Windows 11 via WSL2 (Ubuntu).

## Why This Guide Exists

The official NemoClaw installer (`curl -fsSL https://nvidia.com/nemoclaw.sh | bash`) assumes a native Linux environment. Running it inside WSL2 triggers a cascade of issues — SSL failures, permission errors, line ending corruption, npm placeholder traps, and Docker cgroup misconfigs — that are not documented anywhere.

This guide was born from a real installation session that encountered and resolved **8 distinct problems** before NemoClaw was successfully installed.

## What's Inside

- **[INSTALLATION_GUIDE.md](INSTALLATION_GUIDE.md)** — Getting NemoClaw installed on WSL2:
  - **TL;DR one-shot install scripts** — two copy-paste blocks (one PowerShell, one WSL bash) that get NemoClaw installed with zero detours
  - **Step-by-step walkthrough** — 9 detailed steps with explanations
  - **Detailed problem analysis** — 8 problems with symptoms, failed attempts, root causes, and solutions
  - **Quick reference table** — all problems and solutions at a glance
  - **Background** — how the official installer actually works and why it fails in WSL2

- **[CHEATSHEET.md](CHEATSHEET.md)** — Copy-paste command sequences for every workflow:
  - Fresh install, onboarding, daily use, reconnect after reboot
  - Fix API keys, Telegram bridge, Web UI, file extraction
  - Quick troubleshooting table for every known error
  - Make MTU fix permanent (no more password prompts)

- **[ONBOARDING_GUIDE.md](ONBOARDING_GUIDE.md)** — Running `nemoclaw onboard` on WSL2:
  - **TL;DR one-shot onboarding** — prerequisites, exact command, and what to type at every prompt
  - **5 problems documented** — Docker, CRLF, cgroup v2, API key surprise, and false GPU detection
  - **Deep dive: false GPU detection** — the most confusing WSL2-specific issue, where Docker Desktop exposes a virtual GPU that causes silent sandbox creation failure
  - **Quick reference table** — all onboarding problems at a glance

## The Core Insight

> WSL2 has broken SSL for both system curl and Node.js due to MTU-related TLS packet fragmentation. The solution: do **all downloading on the Windows side** (PowerShell/git), then install from local copies in WSL.

## Problems Covered

| # | Problem | Root Cause |
|---|---------|------------|
| 1 | SSL cipher errors in WSL2 | WSL2 MTU too large + Node.js bundled OpenSSL |
| 2 | npm permission denied (EACCES) | System Node.js global prefix requires root |
| 3 | Windows .npmrc conflicts | npm reads Windows config from /mnt/c/ paths |
| 4 | Node.js version too old | Ubuntu ships Node 18; NemoClaw requires 20+ |
| 5 | npm placeholder package | npmjs.com `nemoclaw` is empty; real install is from GitHub |
| 6 | Windows line endings (CRLF) | Windows git converts LF to CRLF; breaks bash scripts |
| 7 | Docker cgroup v2 misconfiguration | Docker Desktop needs explicit cgroupns=host setting |
| 8 | OpenShell CRLF errors | Same line ending issue affects dependency installers |
| 9 | False GPU detection during onboarding | Docker Desktop exposes virtual GPU; sandbox creation silently fails |

## Prerequisites

- Windows 11 with WSL2 (Ubuntu 22.04 or 24.04)
- Docker Desktop for Windows
- Git for Windows
- An NVIDIA developer account (for API key during onboarding)

## Quick Start

See the [TL;DR section](INSTALLATION_GUIDE.md#tldr--one-shot-install-scripts) for the fastest path.

## License

This guide is provided as-is for informational purposes. NemoClaw is a product of NVIDIA Corporation. This repository is not affiliated with or endorsed by NVIDIA.
