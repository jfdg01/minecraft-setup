# Minecraft Server Project

## What this is
Setting up a 24/7 Minecraft Java server for 3-5 players on Oracle Cloud Always Free (ARM A1, 4 OCPU / 24 GB RAM). Zero monthly cost.

## Current status
- Oracle Cloud account created, home region: **EU-MADRID**
- Upgraded to Pay-As-You-Go (better provisioning priority, still $0)
- Budget alert active: $1/month cap, 90% threshold
- **Blocked on Phase 2**: EU-MADRID has only one availability domain (AD-1) and A1 ARM capacity is currently unavailable

## Local setup (source machine)
- Server files: `C:\Users\gara\Downloads\minecraft\`
- Minecraft version: check `server.jar` — vanilla jar, switching to PaperMC on VPS
- World: `world/` (single dimension so far, no nether/end)
- Port: 39821 (playit.gg tunnel) — will change to 25565 on VPS
- `online-mode=false` (offline/cracked mode)
- SSH keys for Oracle: `~/.ssh/id_ed25519_jfdg01` (mapped via `~/.ssh/config` as `github.com-jfdg01`)

## Key decisions made
- PaperMC over vanilla — better TPS on ARM, drop-in compatible
- JVM heap: `-Xms18G -Xmx18G` with Aikar's G1GC flags
- systemd for process management (not screen/tmux)
- Off-site backups every 6h — Oracle account termination is a known risk
- playit.gg becomes optional once VPS has a real public IP

## Git remote
`git@github.com-jfdg01:jfdg01/minecraft-setup.git`

## Docs
- `docs/setup-oracle-vps.md` — full step-by-step setup guide with progress tracker
- `docs/report.md` — research report comparing all free hosting options
