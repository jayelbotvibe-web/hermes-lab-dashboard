# Hermes Lab Dashboard

> **Ubuntu system tray app** — control panel for the [Hermes Pentest Lab](https://github.com/jayelbotvibe-web/hermes-pentest-lab) and [Hermes Forensics Lab](https://github.com/jayelbotvibe-web/hermes-forensics-lab) running on a single Linux laptop.

## What This Is

You run two security labs on one Ubuntu machine — pentesting tools in Docker containers, forensics tools on a SIFT VM. Both have encrypted LUKS storage. Both need containers/VMs started before you can work.

This tray app sits in your system notification area. It shows whether each lab is ready, lets you start them with one click, and gives you independent VPN control for pentesting.

```
┌──────────────────────────────────────────┐
│  🔽  System Tray (top-right of screen)   │
│                                          │
│  Pentest: ✓ READY  |  Forensics: ✓ READY │
│  ─────────────────────────────────────── │
│  ▶  Start Pentest                        │
│  ■  Stop Pentest                         │
│  🔒 VPN ON                               │
│  🔓 VPN OFF                              │
│  ▶  Start Forensics                      │
│  ■  Stop Forensics                       │
│  ─────────────────────────────────────── │
│  Quit                                    │
└──────────────────────────────────────────┘
```

## How It Fits Together

```
HERMES LAB LAPTOP (Ubuntu)
│
├── hermes-pentest-lab/          ← pentesting tools, Docker compose, reports
│   └── pentest-up.sh            ← containers + tools verification
│
├── hermes-forensics-lab/        ← forensics tools, SIFT VM, evidence
│   └── forensics-up.sh          ← VM startup + canary
│
├── hermes-lab-dashboard/  ◄── THIS REPO
│   ├── hermes-tray              ← Gtk system tray app (auto-starts on login)
│   ├── pentest                  ← wrapper: LUKS mount → pentest-up.sh
│   └── forensics                ← wrapper: forensics-up.sh
│
└── ~/.local/bin/                ← where the scripts actually live
    ├── hermes-tray
    ├── pentest
    └── forensics
```

## Boot Flow

```
Laptop boots
  → SIFT VM auto-starts (systemd)
  → Forensics LUKS auto-mounts (crypttab + fstab)
  → Tray appears on login (autostart .desktop)

You click:
  ▶ Start Pentest → terminal opens → sudo password → LUKS mounts → containers up → ✓ READY
  🔒 VPN ON       → WireGuard connects (when ready to scan)

Done working:
  ■ Stop Pentest   → containers down → LUKS unmounts
  ■ Stop Forensics → VM stops → LUKS unmounts
  → Safe to shut down
```

## Prerequisites

This is built for a SPECIFIC setup:

- Ubuntu 24.04+ with GNOME desktop
- [Hermes Pentest Lab](https://github.com/jayelbotvibe-web/hermes-pentest-lab) installed and configured
- [Hermes Forensics Lab](https://github.com/jayelbotvibe-web/hermes-forensics-lab) installed and configured
- LUKS keyfiles at `~/.pentest-keyfile` and `~/.forensics-keyfile`
- SIFT VM at `/home/niel/vmware/SIFT/SIFT.vmx` (auto-started via systemd)
- Docker pre-built images: `hermes-kali-web`, `hermes-kali-net`, `hermes-osint-tools`

## Install

```bash
# GTK dependencies (Ubuntu)
sudo apt install python3-gi gir1.2-gtk-3.0 gir1.2-appindicator3-0.1

# Copy scripts to your PATH
cp hermes-tray pentest forensics ~/.local/bin/
chmod +x ~/.local/bin/hermes-tray ~/.local/bin/pentest ~/.local/bin/forensics

# Auto-start on login
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/hermes-tray.desktop << 'DESKTOP'
[Desktop Entry]
Type=Application
Name=Hermes Lab Tray
Exec=/home/niel/.local/bin/hermes-tray
Terminal=false
X-GNOME-Autostart-enabled=true
DESKTOP
```

Update paths in the scripts if your username isn't `niel` or vault paths differ.

## Files

| File | What It Is |
|------|-----------|
| `hermes-tray` | Python Gtk system tray — refreshes every 5s, always on top |
| `pentest` | Bash wrapper: LUKS keyfile mount → pentest-up.sh → canary |
| `forensics` | Bash wrapper: forensics-up.sh (LUKS + SIFT VM) |
| `hermes-lab-boot-flow.html` | Architecture diagram — open in browser |

## Related Repos

| Repo | Purpose |
|------|---------|
| [hermes-pentest-lab](https://github.com/jayelbotvibe-web/hermes-pentest-lab) | 22+ tools, Docker compose, AI subagents, PDF reports |
| [hermes-forensics-lab](https://github.com/jayelbotvibe-web/hermes-forensics-lab) | 12 forensic tools, SIFT VM, evidence processing |
| **hermes-lab-dashboard** | ← you are here — cross-repo startup control |
