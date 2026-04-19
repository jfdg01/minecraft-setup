# The free 24/7 Minecraft hosting playbook for 3-5 players

**Oracle Cloud's Always Free ARM Ampere A1 instance is the only genuinely free cloud option that comfortably runs a 3-5 player Minecraft Java server 24/7 in 2026** — with 4 ARM cores, 24 GB RAM, 200 GB storage, and 10 TB/month egress, perpetually free for the life of the account. Every other major cloud has either killed its free tier (Fly.io, Heroku, Railway), made it time-limited (AWS, Azure), or restricted it to HTTP-only workloads that can't bind TCP 25565 (Render, Koyeb). Google Cloud's e2-micro is technically perpetual but too cramped for 3-5 players and capped at 1 GB egress per month. The catch with Oracle: provisioning ARM capacity can take days, and accounts occasionally get terminated without warning — so treat the world data as something that must be backed up off-site, not trusted in place. For this user, who already has local server files and playit.gg configured, the optimal path is to provision an Oracle A1 VM, transfer the world with rsync or tar-over-SSH, run **PaperMC on OpenJDK 21 under a systemd service**, and open port 25565 at both the VCN security list and iptables layers. The playit.gg tunnel becomes redundant (the VPS has a real public IP) but can remain running as a convenience.

## What the Oracle Always Free tier actually gives you

The Always Free tier is **not** the 30-day $300 trial credit — the two are separate, and the Always Free resources persist indefinitely after the trial expires. The centerpiece for Minecraft is the **VM.Standard.A1.Flex shape (Ampere Altra ARM, Neoverse-N1 @ ~3.0 GHz)** with an allowance of **4 OCPUs and 24 GB RAM** that can be provisioned as one fat VM or split across up to four smaller ones. You also get **200 GB of block storage**, **10 TB of outbound egress per month**, two AMD VM.Standard.E2.1.Micro instances (1/8 OCPU / 1 GB — too small for Minecraft but useful for a playit relay or backup target), 20 GB of object storage, and a flexible load balancer.

**This is dramatically over-spec'd for 3-5 players.** Community benchmarks and Oracle's own blog show a 4-core/24 GB A1 can host 20+ vanilla players or ~10 in a moderate modpack. For this use case, **allocate the full 4 OCPU / 24 GB** (you might as well — it's free) and give the JVM `-Xms18G -Xmx18G`, leaving ~4 GB headroom for the OS, off-heap Netty buffers, and disk cache. Aikar specifically warns against heaps beyond ~20 GB because G1GC efficiency drops.

Credit card verification is mandatory at signup (typically a ~$1 hold, never charged unless you manually upgrade). On Always-Free-only accounts, Oracle **rate-limits rather than bills** for overages — you literally cannot exceed the 200 GB block storage because you can't provision more. Real surprise charges almost always trace back to users who accidentally selected a paid shape during onboarding or provisioned in a non-home region.

## The gotchas that matter, in order of how likely they are to bite

**Provisioning capacity is the #1 headache.** Ashburn, Frankfurt, Tokyo, and Amsterdam have had on-and-off "Out of host capacity" errors for ARM instances since 2022, and February 2025 Reddit reports confirm it's still a problem in 2025-2026. The standard workaround is the community [hitrov/oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity) script that polls the LaunchInstance API until capacity frees up — success typically within hours to days. **Counterintuitive fix: upgrading to Pay-As-You-Go costs $0 if you stay within Always Free limits but gives you provisioning priority** and exempts your instance from idle reclamation.

**Idle reclamation** is real but easily avoided. Oracle reclaims Always Free compute only when **all three** conditions are simultaneously true over 7 days: CPU under 20%, network under 20%, and memory under 20%. A Minecraft server idling with a few chunks loaded, plus the JVM holding its full 18 GB heap, trivially clears the memory threshold — you won't be touched.

**Account termination without explanation** is the scariest failure mode and the one you must plan for. Multiple 2024-2025 Reddit reports — including a February 2025 case of a PAYG user running exactly a Paper Minecraft server behind TCPShield — describe accounts getting a "detected unusual and potentially harmful activity" email followed by 30-day wind-down, with support unable or unwilling to reverse it. **Treat OCI as ephemeral and always maintain off-site backups** (Oracle Object Storage in a separate account, Backblaze B2, or rsync to your home PC).

**The dual-firewall trap** catches almost every first-timer. Oracle has cloud-layer security lists *and* host-level iptables/firewalld, both default-deny. Opening one without the other produces "connection timed out" with the server clearly running.

## Why every other free option falls short

**Google Cloud e2-micro** is the only other perpetually free cloud VM — 2 shared vCPU, 1 GB RAM, 30 GB standard persistent disk, in us-west1/us-central1/us-east1. It can technically host 3-5 players running PaperMC with a 900 MB heap and view-distance=6, but the **1 GB/month egress cap** is the killer: a single 3-hour session with 4 active players can push enough chunk data to blow through it, and Google bills overage at ~$0.12/GB. Also, the default "Balanced PD" disk costs money — you must manually select Standard PD. Usable for 1-2 players; risky for 3-5.

**AWS** overhauled its free tier on July 15, 2025: new accounts now get a **$100-$200 credit valid 6 months**, not a perpetual t2/t3.micro. Pre-July 2025 accounts still have the 12-month 1 GB t2.micro, but it converts to paid after a year. **Not free forever.**

**Azure B1s** is 12 months only and converts to ~$7.60/month pay-as-you-go. The exception is **Azure for Students** — $100/year credit that renews while enrolled, no credit card, verified via .edu email or the GitHub Student Developer Pack. A B1s runs ~13 months per $100, so it's effectively perpetually free for actual students. Similarly, the GitHub Student Pack bundles **$200 DigitalOcean credit** good for roughly 16 months on a 2 GB droplet.

**Fly.io** killed its free tier on October 7, 2024. **Railway** removed its free tier in August 2023. **Heroku** ended free dynos in November 2022. **Render** and **Koyeb** offer free HTTP services that spin down on idle and can't bind arbitrary TCP ports — fundamentally wrong for game servers. **IBM Cloud Lite** has no free VMs, only free SaaS APIs.

The honest comparison:

| Option | Truly free forever? | RAM | Fit for 3-5 players? |
|---|---|---|---|
| **Oracle Cloud A1** | Yes (with caveats) | 24 GB | Excellent |
| GCP e2-micro | Yes (US-only) | 1 GB | Marginal; 1 GB egress cap is risky |
| AWS post-July-2025 | No — 6 months of credits | Varies | Expires |
| Azure B1s | No — 12 months only | 1 GB | Expires |
| Azure for Students | While enrolled | Up to B2 | Yes, if you're a student |
| Fly/Railway/Render/Koyeb | Not for Minecraft | — | No |
| Home PC + playit.gg | Yes + electricity | User's | Yes |

## The home hosting alternative is quietly excellent

Since this user **already has playit.gg configured and working locally**, leaving the main PC running 24/7 is a completely legitimate path. On a typical modern desktop drawing 80-100 W at idle, the power cost works out to roughly **$10-12/month at US average electricity rates** — comparable to a cheap paid VPS but with no provisioning anxiety and no risk of account termination. A Minecraft server for 3-5 players uses 1-2 GB RAM and 1-2 CPU cores, which is negligible overhead even while gaming on the same machine if you cap it with `-Xmx2G` and set CPU affinity.

The better long-term move, if the user wants to reduce both electricity and the risk of their main PC being tied up, is a **one-time $55-150 purchase of a Raspberry Pi 5 (8 GB) or a used mini-PC** (Lenovo M710q, Dell 3060 Micro). These draw 5-15 W and cost under $2/month in electricity while handling 3-5 players on PaperMC comfortably. Combined with playit.gg, there's no port forwarding, no ISP issues, and the device can sit silently on a shelf for years. If the Oracle path fails or feels too fragile, this is the backup plan — and arguably should be the primary plan for anyone who values reliability over fashion.

## Transferring your existing world to the VPS

Stop the local server cleanly with `stop` in the console (never Ctrl+C — region files can corrupt mid-write). Then **tar the world first** rather than copying individual files; Minecraft worlds contain thousands of small `.mca` region files that transfer painfully slowly over SFTP. From your local PC:

```bash
tar -czvf mc-backup.tar.gz world world_nether world_the_end \
  server.properties whitelist.json ops.json banned-players.json \
  plugins/ bukkit.yml spigot.yml paper-global.yml
```

Transfer with rsync, which is resumable and handles network drops:

```bash
rsync -avzP -e "ssh -i ~/.ssh/oci_key" mc-backup.tar.gz ubuntu@<VPS_IP>:/tmp/
```

Windows users can do the same with **WinSCP** or **FileZilla** — drag-and-drop GUI, just load your Oracle-generated SSH key in the authentication settings. On the VPS, unpack into `/opt/minecraft` owned by a dedicated `minecraft` user. **Do not copy `server.jar`, `logs/`, `cache/`, `libraries/`, or `versions/`** — download Paper fresh on the VPS. And **re-accept the EULA** on the new machine (`echo "eula=true" > eula.txt`) — Mojang requires it per-install.

If you're on vanilla today, this is the moment to **switch to PaperMC**. It's a drop-in replacement (same world format, same `server.properties`, same ops/whitelist JSONs), and on ARM hardware — where per-core performance is ~20% behind x86 — Paper's async chunk loading and GC optimizations typically deliver **30-50% better TPS**. Download the latest build from `api.papermc.io` for your Minecraft version and swap the jar.

## Running 24/7: skip screen and tmux, use systemd

`screen` and `tmux` work, but they rely on you (or your login session) staying alive in some sense, and they don't auto-restart on crash or survive reboots cleanly. **The correct answer is a systemd unit file** — it auto-starts on boot, restarts on failure, logs to `journalctl`, and handles OCI maintenance reboots gracefully. A minimal working unit:

```ini
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
```

Save as `/etc/systemd/system/minecraft.service`, then `sudo systemctl enable --now minecraft`. The `start.sh` should invoke Java with **Aikar's flags** tuned for a large heap (G1NewSizePercent=40, G1MaxNewSizePercent=50, G1HeapRegionSize=16M, InitiatingHeapOccupancyPercent=20 for heaps ≥12 GB) and set `-Xms18G -Xmx18G` — equal min and max to prevent heap-resize pauses. Live logs with `sudo journalctl -u minecraft -f`. If you want interactive console access, wrap the process in a tmux session inside the unit and attach with `tmux attach -t mc`, or enable RCON and use `mcrcon`.

## The firewall step that trips up everyone

Oracle runs **two independent firewalls** you must open separately, and forgetting the instance-level one is the #1 cause of "I can see the server is running but nobody can connect" threads on Reddit:

1. **VCN Security List** (Oracle console → Networking → VCN → subnet → Default Security List → Add Ingress Rule): source `0.0.0.0/0`, TCP, destination port `25565`.
2. **Host iptables** (Ubuntu on OCI): `sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 25565 -j ACCEPT` then `sudo netfilter-persistent save`. Critical: use `-I INPUT 6` (insert above the default REJECT rule), not `-A` (which appends below REJECT and does nothing).
3. **Oracle Linux alternative**: `sudo firewall-cmd --permanent --add-port=25565/tcp && sudo firewall-cmd --reload`.

Test from your home machine with `nc -vz <VPS_IP> 25565`. "Filtered" means the VCN list is wrong; "refused" means the server isn't listening; "no route" means the host firewall is blocking.

## Should you keep playit.gg on the VPS?

**Technically no — you don't need it.** Oracle gives you a real public IPv4, and once both firewall layers are open, players connect directly to `<VPS_IP>:25565`. playit.gg was designed for home users behind CGNAT or locked-down routers.

**But it doesn't hurt to keep it.** If you install playit via its apt repo (`apt install playit`), it registers as a systemd service and provides you a free `*.joinmc.link` subdomain that's easier for friends to remember than a raw IP, plus a layer of DDoS shielding. If your players have already memorized the existing playit address, that's the path of least resistance. In this setup you don't even need to open port 25565 in the VCN security list — playit dials outbound and tunnels inbound traffic through itself.

## Backups matter more than you think

Given the non-zero rate of Oracle account terminations, **backups are not optional**. Drop this into `/opt/minecraft/backup.sh` and run it every 6 hours via cron:

```bash
#!/bin/bash
BACKUP_DIR=/opt/minecraft/backups
mkdir -p "$BACKUP_DIR"
tar --exclude=backups --exclude=logs --exclude=cache \
  -czf "$BACKUP_DIR/world-$(date +%Y%m%d-%H%M).tar.gz" \
  -C /opt/minecraft world world_nether world_the_end
find "$BACKUP_DIR" -name "world-*.tar.gz" -mtime +7 -delete
```

For true off-site protection, use `rclone` to push the archive to Backblaze B2 (10 GB free), Google Drive, or even back to your home PC over SSH. If you want atomic backups that guarantee no region-file corruption, enable RCON and wrap the tar with `mcrcon ... "save-off" "save-all flush"` before and `"save-on"` after.

## The recommendation

**Primary path: Oracle Cloud Ampere A1 with the full 4 OCPU / 24 GB** running Ubuntu 22.04 LTS, OpenJDK 21, PaperMC with Aikar's flags (`-Xms18G -Xmx18G`), managed by systemd, firewalled at both layers, backed up every 6 hours to off-site storage. If you cannot provision an A1 instance due to capacity, run the hitrov capacity script overnight, or upgrade to Pay-As-You-Go (still $0 under Always Free limits) for provisioning priority. Keep playit.gg installed as a fallback addressing method. This setup costs $0/month and performs at the level of an $80-100/month paid VPS.

**Fallback path, and arguably the better one for reliability: stay on the home PC with playit.gg**, or invest ~$80 in a Raspberry Pi 5 to offload from your main rig. You already have playit working, you control the hardware, and no cloud provider can terminate your account. The only ongoing cost is electricity (~$10/month on a desktop, ~$2/month on a Pi), which isn't free but isn't a subscription either.

**What to avoid:** AWS and Azure free tiers (both expire), Fly.io and Railway (free tiers gone), Render/Koyeb (HTTP-only, will not run Minecraft), and GCP e2-micro unless you're willing to accept cramped RAM and the risk of egress overage charges.

The honest summary: free 24/7 Minecraft cloud hosting exists, it's called Oracle Cloud Always Free, and it's genuinely excellent hardware — but you're trading monthly cost for operational risk (capacity, termination). Pair it with aggressive off-site backups and you have the best free setup available in 2026.