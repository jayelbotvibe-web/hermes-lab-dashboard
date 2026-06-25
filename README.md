# Hermes Lab Dashboard

> System tray indicator for Hermes Pentest + Forensics labs. One-click startup, live status, VPN control.

## What It Does

- **System tray icon** — always visible, refreshes every 5 seconds
- **One-click startup** — click to mount LUKS (keyfile, no passphrase) + start containers
- **VPN toggle** — connect/disconnect VPN independently from lab startup
- **Auto-forensics** — SIFT VM auto-starts at boot via systemd, forensics auto-mounts
- **Live status** — tray shows Pentest: ○ OFF / ⚠ ON / ✓ READY

## Install

```bash
sudo -S -p '' apt install python3-gi gir1.2-gtk-3.0 gir1.2-appindicator3-0.1
cp hermes-tray pentest forensics ~/.local/bin/
chmod +x ~/.local/bin/hermes-tray ~/.local/bin/pentest ~/.local/bin/forensics
```

## Architecture

Open `hermes-lab-boot-flow.html` in a browser or visit the [GitHub Pages site](https://jayelbotvibe-web.github.io/hermes-lab-dashboard/).

## Related

- [Hermes Pentest Lab](https://github.com/jayelbotvibe-web/hermes-pentest-lab)
- [Hermes Forensics Lab](https://github.com/jayelbotvibe-web/hermes-forensics-lab)
