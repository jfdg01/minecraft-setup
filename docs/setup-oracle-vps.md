# Oracle Cloud Minecraft Server Setup

**Goal:** 24/7 PaperMC server for 3-5 players on Oracle Always Free (4 OCPU / 24 GB RAM), zero cost.

## Progress tracker

- [x] Phase 1 — Oracle account created
- [x] Phase 1b — Upgraded to Pay-As-You-Go (activation takes 5-30 min, confirmation via email)
- [x] Phase 1c — Budget alert configured ($1 limit, 90% threshold, alerts to 3 emails)
- [ ] Phase 2 — Provision A1 ARM VM ← **current blocker: EU-MADRID capacity**
- [ ] Phase 3 — Open firewall (VCN + iptables)
- [ ] Phase 4 — Set up server environment
- [ ] Phase 5 — Transfer world
- [ ] Phase 6 — systemd service
- [ ] Phase 7 — Automatic backups

---

## Phase 1 — Create Oracle Cloud account ✓

1. Go to **cloud.oracle.com** → "Start for free"
2. Pick a **Home Region** near you — cannot be changed later
   - Account uses **EU-MADRID** (single availability domain: AD-1 only)
3. Enter credit card for ~€93 verification hold (released in 3-5 days, never an actual charge)
4. **Upgrade to Pay-As-You-Go** — still $0 within Always Free limits, but gives ARM provisioning priority
   - Activation takes 5-30 minutes, confirmed by email
5. **Set up budget alert**: Billing → Budgets → Create Budget
   - Amount: $1/month, threshold: 90% actual spend, 1-year window
   - Protects against accidental paid resource provisioning

---

## Phase 2 — Provision the A1 ARM VM

1. Console → **Compute → Instances → Create Instance**
2. Name: `minecraft`
3. Image: **Ubuntu 22.04** (Canonical)
4. Shape: Change Shape → Ampere → `VM.Standard.A1.Flex` → **4 OCPUs, 24 GB RAM**
5. SSH Keys: Generate new pair, download private key (e.g. `oci_key.key`) — keep it safe
6. Boot volume: 100 GB
7. Click **Create** and wait for status `Running`
8. Note the **Public IP address**

> **EU-MADRID has only AD-1** — there is no AD-2/AD-3 to fall back to. Capacity issues here are common.
>
> **If you get "Out of host capacity":**
> 1. Wait for PAYG upgrade to fully activate (5-30 min) — it improves provisioning priority
> 2. Retry manually at off-peak hours (3-6 AM Madrid time works best)
> 3. Use the [hitrov/oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity) script to auto-retry the API until a slot opens
> 4. Do NOT use Preemptible capacity — it can be killed at any time with 2 min notice

---

## Phase 3 — Open the firewall (both layers required)

**Layer 1 — Oracle VCN Security List:**
1. Console → Networking → Virtual Cloud Networks → your VCN → Subnets → Public Subnet → Default Security List
2. Add Ingress Rule:
   - Source: `0.0.0.0/0`
   - Protocol: TCP
   - Destination Port: `25565`

**Layer 2 — Host iptables** (run after SSH-ing into the VM):

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 25565 -j ACCEPT
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

**Test connectivity from your local PC:**
```bash
nc -vz <YOUR_VPS_IP> 25565
# "refused" = server not running yet (firewall is fine)
# "filtered" = VCN security list is wrong
# "no route" = host iptables is blocking
```

---

## Phase 4 — Set up the server environment

SSH into the VM:
```bash
ssh -i path/to/oci_key.key ubuntu@<YOUR_VPS_IP>
```

Run on the VPS:
```bash
# Dedicated user + Java 21
sudo useradd -r -m -U -d /opt/minecraft -s /bin/bash minecraft
sudo apt update && sudo apt install -y openjdk-21-jre-headless

# Server directory
sudo mkdir -p /opt/minecraft
sudo chown minecraft:minecraft /opt/minecraft

# Download PaperMC — replace 1.21.4 with your actual Minecraft version
sudo -u minecraft wget -O /opt/minecraft/server.jar \
  "https://api.papermc.io/v2/projects/paper/versions/1.21.4/builds/latest/downloads/paper-1.21.4-latest.jar"

# Accept EULA
echo "eula=true" | sudo -u minecraft tee /opt/minecraft/eula.txt
```

**Create start script:**
```bash
sudo -u minecraft tee /opt/minecraft/start.sh << 'EOF'
#!/bin/bash
java -Xms18G -Xmx18G \
  -XX:+UseG1GC -XX:+ParallelRefProcEnabled \
  -XX:MaxGCPauseMillis=200 -XX:+UnlockExperimentalVMOptions \
  -XX:+DisableExplicitGC -XX:+AlwaysPreTouch \
  -XX:G1NewSizePercent=40 -XX:G1MaxNewSizePercent=50 \
  -XX:G1HeapRegionSize=16M -XX:G1ReservePercent=15 \
  -XX:G1HeapWastePercent=5 -XX:G1MixedGCCountTarget=4 \
  -XX:InitiatingHeapOccupancyPercent=20 \
  -XX:G1MixedGCLiveThresholdPercent=90 \
  -XX:G1RSetUpdatingPauseTimePercent=5 \
  -XX:SurvivorRatio=32 -XX:+PerfDisableSharedMem \
  -XX:MaxTenuringThreshold=1 \
  -jar /opt/minecraft/server.jar nogui
EOF
sudo chmod +x /opt/minecraft/start.sh
```

---

## Phase 5 — Transfer your world from local PC

**On your local PC** (Git Bash), from `C:\Users\gara\Downloads\minecraft\`:
```bash
tar -czvf mc-backup.tar.gz \
  world \
  server.properties \
  whitelist.json ops.json banned-players.json banned-ips.json

rsync -avzP -e "ssh -i path/to/oci_key.key" \
  mc-backup.tar.gz ubuntu@<YOUR_VPS_IP>:/tmp/
```

**Back on the VPS:**
```bash
sudo -u minecraft tar -xzvf /tmp/mc-backup.tar.gz -C /opt/minecraft/

# Fix port — local setup used 39821 (playit.gg), VPS uses standard 25565
sudo -u minecraft sed -i 's/server-port=39821/server-port=25565/' /opt/minecraft/server.properties
sudo -u minecraft sed -i 's/query.port=39821/query.port=25565/' /opt/minecraft/server.properties
```

---

## Phase 6 — systemd service (auto-start + auto-restart)

```bash
sudo tee /etc/systemd/system/minecraft.service << 'EOF'
[Unit]
Description=Minecraft Paper Server
After=network.target

[Service]
Type=simple
User=minecraft
WorkingDirectory=/opt/minecraft
ExecStart=/opt/minecraft/start.sh
Restart=on-failure
RestartSec=15s
SuccessExitStatus=0 143

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now minecraft
```

**Useful commands:**
```bash
sudo journalctl -u minecraft -f      # live logs
sudo systemctl status minecraft      # service status
sudo systemctl restart minecraft     # manual restart
sudo systemctl stop minecraft        # graceful stop
```

---

## Phase 7 — Automatic backups (every 6 hours, 7-day retention)

```bash
sudo -u minecraft tee /opt/minecraft/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR=/opt/minecraft/backups
mkdir -p "$BACKUP_DIR"
tar --exclude=backups --exclude=logs --exclude=cache \
  -czf "$BACKUP_DIR/world-$(date +%Y%m%d-%H%M).tar.gz" \
  -C /opt/minecraft world
find "$BACKUP_DIR" -name "world-*.tar.gz" -mtime +7 -delete
EOF
sudo chmod +x /opt/minecraft/backup.sh

# Schedule via cron
(sudo crontab -u minecraft -l 2>/dev/null; echo "0 */6 * * * /opt/minecraft/backup.sh") \
  | sudo crontab -u minecraft -
```

---

## Final setup summary

| Item | Value |
|---|---|
| Hardware | Oracle A1 — 4 ARM cores, 24 GB RAM |
| Cost | $0/month (Always Free) |
| OS | Ubuntu 22.04 LTS |
| Java | OpenJDK 21 |
| Server | PaperMC (drop-in vanilla replacement, faster on ARM) |
| JVM heap | `-Xms18G -Xmx18G` with Aikar's G1GC flags |
| Port | 25565 (standard, direct IP) |
| Process manager | systemd (survives reboots and crashes) |
| Backups | Every 6h, kept 7 days, stored in `/opt/minecraft/backups/` |

Players connect with: `<YOUR_VPS_IP>` (no port suffix needed for 25565)

---

## Troubleshooting quick reference

| Symptom | Likely cause | Fix |
|---|---|---|
| "Connection timed out" | One firewall layer still closed | Check both VCN list and iptables |
| "Connection refused" | Server not running | `systemctl status minecraft` |
| Can't provision A1 | Capacity unavailable | Retry or use hitrov script |
| Surprise charges | Accidentally picked paid shape | Check Console → Cost Analysis |
| World rollback after crash | Region files corrupted | Always stop with `systemctl stop`, never kill -9 |
