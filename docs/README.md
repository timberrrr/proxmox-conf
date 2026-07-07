# Proxmox Homelab — Overview & Status

_Last updated: 2026-07-04_

## Connection

| | |
|-|-|
| Proxmox host | `ssh pve` → 192.168.1.3 (key `~/.ssh/proxmox_ed25519`) |
| Web UI | https://192.168.1.3:8006 (root + PAM) |
| Run CT command | `ssh pve "pct exec <CTID> -- <cmd>"` |
| Direct CT SSH | `ssh -i ~/.ssh/proxmox_ed25519 root@192.168.1.<x>` |

SSH config alias (`~/.ssh/config`):
```
Host pve
    HostName 192.168.1.3
    User root
    IdentityFile ~/.ssh/proxmox_ed25519
    StrictHostKeyChecking accept-new
```

## Inventory

| ID | Name | IP | UI | Status |
|-|-|-|-|-|
| VM 100 | haos-18.0 | 192.168.1.2 | http://192.168.1.2:8123 | running |
| CT 101 | ollama | 192.168.1.10 | — | STOPPED (onboot=0, dropped) |
| CT 102 | fbmp-watcher | 192.168.1.11 | — | running |
| CT 103 | immich | 192.168.1.12 | http://192.168.1.12:2283 | running |
| CT 104 | frigate | 192.168.1.13 | http://192.168.1.13:5000 | running — 3 cameras LIVE (GPU ONNX detect, YOLOv9-s@640) |
| CT 105 | jellyfin | 192.168.1.14 | http://192.168.1.14:8096 | running |
| CT 106 | calibre | 192.168.1.15 | http://192.168.1.15:8083 | running (5,662 books) |
| CT 107 | wg-gateway | 192.168.1.16 | — | running |

Storage: thin pool `pve/data` ~349G, OVERCOMMITTED (sum of CT sizes > pool) — watch real usage.

### Host I/O & network topology (the real bottleneck is the disk, not the NIC)

Checked 2026-07-01. **Both NICs and the media drive are USB**, all on **one USB 3.1 Gen2 controller
(Bus 002, 10 Gbps root hub)**:
- `nic1` (cdc_ncm, USB link 5 Gbps / 1 GbE PHY) — the live trunk (mgmt untagged + VLAN 30 tagged)
- `nic0` (cdc_ncm, 5 Gbps / 1 GbE) — **idle spare** (vmbr1 down); reserved for camera-VLAN separation
- `sda` = **WD My Passport 465 G, 2.5" spinning USB HDD** — holds Frigate recordings *and* Jellyfin media

**Bandwidth is a non-issue.** Max aggregate demand ≈ nic1 (~1 Gb) + nic0 (~1 Gb) + disk (~1 Gb) ≈ 3 Gbps
against a 10 Gbps controller. Camera ingest is ~15–30 Mbps, Jellyfin a few × 10 Mbps — the 1 GbE link
sits at ~3 Mbps idle with 5–10× headroom. **Do not add a NIC for throughput**; bonding is pointless
(consumer Orbi, no LACP). The only thing that fills the 1 GbE link is bulk SMB media syncs (manual).

**The actual scarce resource is the My Passport spindle.** One 2.5" HDD head does triple duty: Frigate
writing 3 record streams + Jellyfin reading movies + any sync writing. Spinning disks choke on
*concurrent random* I/O long before any bus/NIC fills. **If Jellyfin stutters while cameras record, or
recordings drop, this disk is the cause** — not the network. High-value upgrade if that happens: a
**USB SSD**, or split Frigate recordings onto a separate disk from Jellyfin media (removes seek
contention). Far more impactful than anything network-side.

**Resilience caveat:** both NICs + storage share one USB controller = single point of failure. An xHCI
reset (we hit a USB drive hang earlier) would drop network *and* storage *and* cameras at once. Noted,
not worth re-architecting now. (The `nic0` traffic-separation option doesn't change this — both NICs are
on the same controller; separation only isolates the per-NIC 1 GbE *link*, e.g. so a sync can't crowd
camera ingest.)

## Migration Status

### HA-side (done on HA device's Claude instance)
- [x] HAOS VM up + HA accessible
- [x] Integrations reconnect (Tesla, Ratgdo, Tapo, etc.)
- [ ] Move UPS USB cable RPi → Vostro, verify NUT
- [ ] Install Mosquitto add-on (for Frigate MQTT — optional)
- [ ] Switch HA Assist to native Claude/Anthropic conversation agent (Ollama dropped)

### Proxmox/LXC-side
- [x] CT 101 Ollama — built then stopped/dropped (GPU plan)
- [x] CT 102 fbmp-watcher — Docker, Anthropic API, running healthy
- [x] CT 103 Immich — 14,260 photos, SigLIP2-384, faces+search, dedup, storage reorg
- [x] CT 104 Frigate — GPU passthrough + Docker + nvidia-toolkit; SAFE MODE (no cams yet)
- [x] CT 105 Jellyfin — iGPU QSV, media synced, DLNA PlayTo working (Hisense TV), 10G disk
- [x] CT 107 wg-gateway — WireGuard spoke + Caddy on VPS; Jellyfin + HA exposed
- [x] **VLAN 30 networking done (2026-06-30):** `nic1` is now a **trunk** — untagged mgmt → `vmbr0`
  (`192.168.1.3`), tagged VLAN 30 → `nic1.30`/`vmbr30`. Frigate (CT104) dual-homed: `eth0`
  `192.168.1.13` (mgmt/UI) + `eth1` `192.168.30.13` (vmbr30); reaches IoT gw `192.168.30.1`. ✅
  (Old `nic0`/`vmbr1` access-port plan abandoned; USB-NIC idea dropped.)
- [x] Enable RTSP on Tapo cameras — done (peachycams/Plop1234!, all 3)
- [x] Frigate cameras LIVE (2026-06-30): lower_driveway .105, upper_driveway .107, backyard .106;
  object-only recording (person/dog/car/cat). See [ct-104-frigate.md](ct-104-frigate.md).
- [x] **Frigate follow-up (1) boot-race auto-fix — DONE 2026-06-30, extended 2026-07-04** (host
  `frigate-gpu-fix.service`; now covers BOTH the GPU race and the USB-storage mount race that failed
  CT104 on 2026-07-04 — see [ct-104-frigate.md](ct-104-frigate.md). pve-guests now ordered after
  `mnt-media.mount`; still to validate through a real reboot)
- [x] **Frigate follow-up (2) recordings off thin pool — DONE 2026-06-30** (`mp0` → `/mnt/media/frigate`, `/dev/sda1`)
- [x] **Frigate follow-up (3) GPU detector — DONE 2026-07-01** (ONNX/YOLOv9-s @ 640, ~27 ms inference, GTX 1650)

## Next Actions

**A — Networking: DONE (2026-06-30).** `nic1` trunk (untagged mgmt + tagged VLAN 30 → `vmbr30`);
Frigate dual-homed (`eth1` `192.168.30.13`), verified reaching IoT gw. Applied with an auto-revert
timer (`systemd-run --on-active=120`) after an earlier rework knocked out management — that's the
safe way to change Proxmox networking remotely. Config in [ct-104-frigate.md](ct-104-frigate.md).

**B — Frigate cameras: LIVE.** 3 cameras streaming + detecting person/dog/car/cat, object-only
recording. Follow-ups (1) boot-race auto-fix, (2) recordings off the thin pool → `/mnt/media/frigate`,
and (3) GPU detector (YOLOv9-tiny-320 ONNX, 7 ms inference on the GTX 1650) are **all DONE**. The only
remaining item is wildlife (possum/roo/snake/deer/pig), which needs a **Frigate+** custom model — COCO-80
has no Australian fauna. See [ct-104-frigate.md](ct-104-frigate.md).

**C — HA Assist:** Switch to native Claude/Anthropic conversation agent. See [vm-100-haos.md](vm-100-haos.md).

**D — Immich → OneDrive backup:** After user finishes reconciling old HDD photos.
OneDrive credentials must NOT be stored on the server. See [ct-103-immich.md](ct-103-immich.md).

**E — Jellyfin hardening:** 2FA / strong credentials (internet-exposed via sslip.io). Caddy rate-limit/fail2ban.
See [ct-105-jellyfin.md](ct-105-jellyfin.md).

**F — Music sync (Windows-side):** Run from dev workstation over SMB:
```
robocopy "C:\Users\tim\OneDrive\Music" "\\192.168.1.3\media\music" /MIR /Z /R:3 /W:5
```
Set up Windows Task Scheduler for ongoing sync. See [ct-105-jellyfin.md](ct-105-jellyfin.md).

**G — Tapo RTSP:** Enable RTSP in Tapo app per camera before Frigate config.

**H — Optional:** Fix 7 bogus-date Immich photos (5 × 2037, 2 × 2026 import-date) in Immich UI.
Change Immich admin password (`shouldChangePassword` still set).

**I — Jellyfin Default.xml:** Consider restoring video DirectPlay wildcard removed during DLNA debug:
```xml
<DirectPlayProfile container="" type="Video" />
```
Only needed if non-Hisense DLNA clients have "unsupported" issues browsing ContentDirectory.

**J — Chromecast:** 1st gen is a dead end (Cast SDK EOL April 2024). Consider Chromecast with Google TV
or Fire Stick for native Jellyfin app.

**K — Ebooks (CT 106 calibre):** Running, **5,662 books** loaded. SMTP + Send-to-Kindle work via the
**web UI** (set `mail_use_ssl=2`; granted `tim` roles). Log level reset to INFO (20) on 2026-07-01. Remaining: add Amazon approved-sender, change
admin pw, and **decide** whether to migrate to **Calibre-Web Automated (CWA)**
so the "Calibre Web Companion" phone app works (its send 404s on vanilla CW). See [ct-106-calibre.md](ct-106-calibre.md).

**L — Media added to Jellyfin (2026-06-29):** Movies + TV (True Blood, IT Crowd, BBC docos, IMAX, etc.)
transferred to `/media/movies` + `/media/tv` from an old backup. Bulk transfer used WSL2 ext4 mount of
the USB drive (Wi-Fi/SMB too slow) — technique in [ct-106-calibre.md](ct-106-calibre.md). Jellyfin
DLNA fixes (lip-sync/seek/HW/disk) on 2026-06-29 — see [ct-105-jellyfin.md](ct-105-jellyfin.md).

**M — sara_tmp media cleanup (workstation, 2026-06-29/30):** Old "Sara Backup" on `C:\sara_tmp` deduped
+ de-junked (~70 GB freed); movies/TV → Jellyfin, ebooks → Calibre. First music batch (135 albums)
already **merged into Jellyfin Music, genre `"Sara"`**, pushed back to `OneDrive\Music` for `/MIR`
consistency (scheme in [ct-105-jellyfin.md](ct-105-jellyfin.md)).
**Second-batch music staging dirs on `C:\sara_tmp\` (workstation), pending review/ship:**
- `music_complete_review` — **22 MusicBrainz-verified-complete albums** (beets-tagged) → **review, then
  ship** via the same flow (collision-check → /media/music → genre "Sara" → OneDrive). Not yet shipped.
- `music_incomplete` — ~398 incomplete/fragment albums (1–2 track fragments, gaps, trailing-missing).
- `music_ship` — 42 albums beets couldn't match even loosened (bad tags/not in MB).
- Tools/config in jobs tmp: `frigate-config.yml`, beets at `C:\sara_tmp\beets\config.yaml`.
- Album exception parked at `/mnt/media/_sara_review/` (Clare Bowditch extra track).
Wildlife/Calibre/Jellyfin minor TODOs: Calibre admin-pw + reset DEBUG log + Amazon approved-sender +
CWA-vs-vanilla decision; verify Jellyfin DLNA seek fix.
