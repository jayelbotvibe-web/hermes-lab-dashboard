# Multi-Profile Manager — AI Orchestration Layer

> **Supervisor pattern for Hermes Agent** — one lightweight manager profile routes forensic and pentesting tasks to two specialized worker profiles via one-shot CLI dispatch.

## What This Is

You already have two labs on one machine:
- **Pentest Lab** — Docker containers (kali-web, kali-net, osint-tools, neo4j), WireGuard VPN, LUKS vault
- **Forensics Lab** — Docker tools (volatility3, plaso, MFT), SIFT VM via SSH, MemProcFS

Each lab has its own Hermes Agent **profile** — a fully independent AI agent with its own SOUL (persona), skills, memory, and tool access. The forensics profile knows DFIR. The pentest profile knows pentesting.

**This guide adds a third profile — the manager** — that sits between you and the labs. You tell the manager what you need. The manager checks if the lab is ready, routes the task, dispatches it, validates the results, and presents them.

```
You: "analyze this memory dump"
  → Manager: checks forensics health → dispatches → validates → presents findings

You: "scan example.com"
  → Manager: checks pentest health → dispatches → validates → presents findings
```

No switching profiles. No typing commands into the right profile. One conversation surface.

## Architecture

```
USER (niel)
  │
  ▼
┌──────────────────────────────────────┐
│  MANAGER PROFILE (default)            │
│  cwd: /home/niel                      │
│  Role: route, health-check, dispatch  │
│  Does NOT run tools                   │
└──────┬───────────────────┬────────────┘
       │ dispatch          │ dispatch
       ▼                   ▼
┌──────────────┐   ┌──────────────────┐
│ FORENSICS    │   │ PENTEST          │
│ PROFILE      │   │ PROFILE          │
│              │   │                  │
│ SOUL: DFIR   │   │ SOUL: pentest    │
│ 10 skills    │   │ 5 skills         │
│              │   │                  │
│ TOOLS:       │   │ TOOLS:           │
│ Docker vol3  │   │ Docker kali-web  │
│ Docker plaso │   │ Docker kali-net  │
│ Docker MFT   │   │ Docker osint     │
│ MemProcFS    │   │ Docker neo4j     │
│ SIFT VM/SSH  │   │ WireGuard VPN    │
│              │   │ LUKS vault       │
└──────────────┘   └──────────────────┘
```

### The dispatch mechanism

```bash
# One-shot dispatch — print ONLY the final response, then exit
hermes -z "<self-contained prompt>" --profile forensics
hermes -z "<self-contained prompt>" --profile pentest
```

`-z` (`--oneshot`) is Hermes Agent's official one-shot mode. The profile boots fresh, loads its SOUL + skills + memory, executes the task, prints the result, and exits. Designed for scripts, pipes, and programmatic orchestration.

## Prerequisites

- Hermes Agent v0.17.0+ (any version with `-z` one-shot and `--profile` support)
- [Hermes Pentest Lab](https://github.com/jayelbotvibe-web/hermes-pentest-lab) — configured with Docker compose, VPN, LUKS
- [Hermes Forensics Lab](https://github.com/jayelbotvibe-web/hermes-forensics-lab) — Docker images built, SIFT VM accessible
- API keys in each profile's `.env` file

## Setup — Step by Step

### 1. Create the worker profiles

```bash
# Forensics profile
hermes profile create forensics

# Pentest profile
hermes profile create pentest
```

### 2. Configure each profile

Each profile needs `config.yaml`:

```yaml
# ~/.hermes/profiles/forensics/config.yaml
agent:
  max_turns: 120
  tool_use_enforcement: true
terminal:
  backend: local          # MUST be local — profile needs host file access
  cwd: /home/niel/forensics  # optional, -z mode inherits parent cwd
  timeout: 600
model:
  default: deepseek-v4-pro  # or your preferred model
  provider: deepseek
memory:
  memory_enabled: true
approvals:
  mode: manual            # never YOLO on evidence
security:
  redact_secrets: true
```

### 3. Add API keys

Each profile needs its own `.env` with provider API keys:

```bash
# Copy from default profile
cp ~/.hermes/.env ~/.hermes/profiles/forensics/.env
cp ~/.hermes/.env ~/.hermes/profiles/pentest/.env
chmod 600 ~/.hermes/profiles/{forensics,pentest}/.env
```

### 4. Set up profile SOULs and skills

The forensics profile needs its DFIR persona (`SOUL.md`) and skills. The pentest profile needs its security persona and skills. These live in `~/.hermes/profiles/<name>/`.

### 5. Install the health check

```bash
cat > ~/.hermes/scripts/check-labs << 'EOF'
#!/bin/bash
set -e
echo "=== PENTEST ===" && bash /home/niel/pentest-repo/scripts/pentest-canary.sh 2>&1 | tail -3
echo "" && echo "=== FORENSICS ===" && bash /home/niel/forensics/scripts/session-canary.sh 2>&1 | tail -3
EOF
chmod +x ~/.hermes/scripts/check-labs
```

### 6. Install the lab-manager skill

```bash
mkdir -p ~/.hermes/skills/devops/lab-manager
```

Create `~/.hermes/skills/devops/lab-manager/SKILL.md` — see the [template below](#lab-manager-skill-template).

## Daily Workflow

### Check lab status

```bash
bash ~/.hermes/scripts/check-labs
```

```
=== PENTEST ===
=== Results: 14 passed, 0 failed ===
✓ All services operational — ready for engagement

=== FORENSICS ===
=== Results: 18 passed, 0 failed ===
⚠ DEGRADED: sift-rip.pl (triage only)
```

### Dispatch a forensics task

From the default profile conversation:

```
You: "analyze memory dump at /home/niel/forensics/cases/case-001/memory/memory.dd"

Manager: checks health → dispatches:
  hermes -z "Run canary. Analyze dump at /absolute/path. Use MemProcFS primary,
  vol3 for cross-val. Report in F-examiner-NNN format with confidence levels."
  --profile forensics

Manager: validates output → presents findings
```

### Dispatch a pentest task

```
You: "passive OSINT on example.com — zero packets only"

Manager: checks health (VPN up? containers running?) → dispatches:
  hermes -z "Run canary. Passive OSINT on example.com. Zero packets.
  Use theHarvester, crt.sh. Return subdomains, emails, exposed tech.
  No active scanning."
  --profile pentest

Manager: validates output → presents findings
```

## Key Design Decisions

### Why the manager doesn't run tools

The manager is thin by design. It has zero knowledge of `docker run` flags, volatility3 plugins, nmap rate limits, or CVSS scoring. That knowledge lives in the worker profiles' skills.

If the manager tried to run tools directly, it would:
- Inflate its context with domain-specific knowledge (context pollution)
- Drift from the worker profiles' skill updates
- Become a single point of failure for two distinct domains

### Why one-shot dispatch, not persistent sessions

Each `hermes -z` creates a fresh session. The worker boots, loads its SOUL + skills, does the task, and exits. This means:

- **Filesystem changes are visible.** Write a skill, add evidence, update a Docker image — next dispatch sees it.
- **Conversational context does NOT carry.** Each dispatch is a clean slate. The manager bridges continuity by including relevant facts from prior dispatches in subsequent prompts.
- **No stale state.** No half-finished investigations clogging context windows.

### Ponytail minimalism

The forensics lab has exactly **3 scripts** + **1 data file**:

| File | Lines | Why it exists |
|------|-------|---------------|
| `session-canary.sh` | 126 | Validates all tools — deterministic, the agent can't validate itself |
| `sift-exec.sh` | 24 | SSH wrapper — prevents HOME-sandbox key-path footgun |
| `down.sh` | 40 | Cleanup — prevents stale mounts, running VMs, FUSE leftovers |
| `tool-catalog.yaml` | 120 | Data — single source of truth for tool versions, entrypoints, pitfalls |

Everything else (case creation, evidence registration, IOC scanning, report generation) is handled by the forensics profile's skills through agent reasoning. Scripts are for the deterministic, repetitive, footgun-prone bits.

## Critical Pitfalls

### 1. SSH key paths in sandboxed HOME

Hermes profiles sandbox `$HOME` to `~/.hermes/profiles/<name>/home`. SSH commands **must** use absolute identity paths:

```bash
# WRONG — sandboxed HOME doesn't have .ssh/
ssh sansforensics@172.16.146.128 "command"

# RIGHT — absolute key path
ssh -i /home/niel/.ssh/id_rsa sansforensics@172.16.146.128 "command"
```

The `sift-exec.sh` wrapper handles this automatically.

### 2. No separate forensics LUKS volume

Unlike the pentest lab (which has its own LUKS vault at `/dev/mapper/pentest-vault`), forensics may live on the root filesystem. The canary reports this as "INFO — no separate LUKS vault" — this is correct, not a failure. Evidence encryption is optional for forensics.

### 3. SIFT VM networking

The SIFT VM uses VMware bridged networking. IP: `172.16.146.128`. Port 2222 forwarding is NOT configured — SSH directly to the bridged IP. This is more reliable than NAT port forwarding.

### 4. Docker entrypoint gotchas

- `forensics-volatility3:2.7.0` — entrypoint IS the volatility3 binary. Commands start with flags directly: `-f dump.mem windows.pslist` (NOT `vol -f dump.mem ...`)
- `forensics-mft-tools:1.2.0.0` — entrypoint is bash. Use `--entrypoint python3` to run Python tools.

### 5. Profile .env required

Every profile that uses a cloud provider MUST have its own `.env` file. Without it, dispatch fails with "Provider has no API key." Copy from the default profile if all profiles use the same provider.

### 6. ConnectTimeout for SSH

Use 10 seconds — 3 seconds causes intermittent failures when run immediately after Docker commands (system briefly busy).

## Files Map

| Path | Purpose |
|------|---------|
| `~/.hermes/profiles/forensics/` | Forensics profile — SOUL, skills, config, .env, sessions |
| `~/.hermes/profiles/pentest/` | Pentest profile — SOUL, skills, config, .env, sessions |
| `~/.hermes/scripts/check-labs` | 5-line health check — runs both canaries |
| `~/.hermes/skills/devops/lab-manager/SKILL.md` | Manager skill — routing, pitfalls, dispatch patterns |
| `~/forensics/scripts/session-canary.sh` | Forensics canary — 3 runtimes, 18 checks |
| `~/forensics/scripts/sift-exec.sh` | SIFT VM SSH wrapper |
| `~/forensics/scripts/down.sh` | Safe cleanup |
| `~/forensics/tools/tool-catalog.yaml` | Tool registry — versions, entrypoints, pitfalls |
| `~/pentest-repo/scripts/pentest-canary.sh` | Pentest canary — 14 checks |
| `~/pentest-repo/scripts/pentest-up.sh` | Pentest startup |
| `~/pentest-repo/scripts/pentest-repair.sh` | Pentest auto-repair |
| `~/profile-manager-architecture.html` | Dark-theme SVG architecture diagram |

## Supervisor Pattern — Industry Context

The manager-worker architecture follows the **Supervisor (Orchestrator-Worker) pattern** — one of four proven multi-agent patterns alongside Router, Pipeline, and Swarm. Microsoft Azure, Anthropic, and LangGraph all recommend this pattern for structured, compliance-heavy, multi-domain workflows.

Key principle from Azure Architecture Center: *"The manager agent communicates directly with specialized agents... The supervisor maintains context and coordinates handoffs."*

## lab-manager skill template

```markdown
---
name: lab-manager
description: "Route tasks to forensics/pentest profiles via one-shot dispatch."
version: 1.1.0
category: devops
---

# Lab Manager — ponytail edition

You route forensics and pentest tasks to the right profile. That's it.

## Dispatch

    hermes -z "<self-contained prompt>" --profile forensics
    hermes -z "<self-contained prompt>" --profile pentest

## Health

    bash /home/niel/.hermes/scripts/check-labs

Canary failures tell you what's wrong. Agent reasoning handles recovery.

## Prompt rules

Every dispatch prompt: canary first, then task. Absolute paths. Let the profile decide how.

## Key pitfalls

- SSH keys: profiles sandbox HOME. Use `-i /home/niel/.ssh/id_rsa`.
- SIFT VM: bridged IP 172.16.146.128, no port forward.
- No forensics LUKS: on root NVMe. Canary says INFO — not a failure.
- Pentest .env: required at ~/.hermes/profiles/pentest/.env.
- vol3 entrypoint: don't prefix `vol`. In tool-catalog.
```

## Related Repos

| Repo | Role |
|------|------|
| [hermes-pentest-lab](https://github.com/jayelbotvibe-web/hermes-pentest-lab) | Pentest worker — Docker tools, VPN, reports |
| [hermes-forensics-lab](https://github.com/jayelbotvibe-web/hermes-forensics-lab) | Forensics worker — Docker + SIFT VM + MemProcFS |
| **hermes-lab-management-dashboard** | ← you are here — tray launcher + manager docs |

## Quick Start (after setup)

```bash
# Check both labs
check-labs

# Route a forensic task
hermes -z "Run canary. Analyze /path/to/dump. Use MemProcFS." --profile forensics

# Route a pentest task
hermes -z "Run canary. OSINT on example.com. Zero packets." --profile pentest
```
