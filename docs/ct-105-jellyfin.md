# CT 105 — Jellyfin

- **IP:** 192.168.1.14
- **UI:** http://192.168.1.14:8096
- **External:** `jellyfin-152-89-105-204.sslip.io` (via Caddy on VPS)
- **Status:** running, healthy
- **Specs:** unprivileged, nesting, 2c/2G/512swap, **10G rootfs** on local-lvm, Debian 13

## Remote playback fix — reverse proxy not differentiating remote clients (2026-07-04)

**Symptom:** off-LAN (via `sslip.io`), **video wouldn't start** while **music played fine**.

**Root cause:** CT107 masquerades all tunnel traffic (`-s 10.99.0.0/24 -o eth0 MASQUERADE`),
so Jellyfin saw every remote client as the LAN gateway `192.168.1.16`. With
`KnownProxies` empty, Jellyfin ignored Caddy's `X-Forwarded-For` → treated remote
sessions as **local**. Proof: real remote IP `1.127.29.8` appeared **0×** in the log; gateway
`192.168.1.16` was logged as the client for **all** remote WebSocket sessions.
Combined with `RemoteClientBitrateLimit=0` (unlimited), the server offered remote video at
direct-play / full bitrate the ~30 Mbps home uplink can't sustain → video never buffers.
Music (~hundreds of kbps) fits the uplink → plays.

**Fix (both needed, restart required):**
- `/etc/jellyfin/network.xml`: `<KnownProxies><string>192.168.1.16</string></KnownProxies>`
  → Jellyfin trusts Caddy's XFF and reads the real client IP → classifies remote correctly.
- `/etc/jellyfin/system.xml`: `<RemoteClientBitrateLimit>8000000</RemoteClientBitrateLimit>`
  (8 Mbps) → remote video transcodes down to fit the ~30 Mbps uplink without saturating it.

Backups: `network.xml.bak-20260704`, `system.xml.bak-20260704`.
**Verified:** a failed auth through Caddy now logs `IP: "152.89.105.204"` (real upstream),
not `192.168.1.16` → XFF honored. Tune the cap in **Dashboard → Playback → Remote client
bitrate limit**.

**...but the real blocker was a broken GPU transcode.** With the proxy fix in place, a
remote video test correctly issued a capped ~1.4 Mbps HLS transcode — which then died:
`FFmpeg exited with code 187` on segment 0 (`Device creation failed … driver=iHD`). Remote
always transcodes (phones can't direct-play the XVID/AVI library); music and LAN direct-play
don't, which is why only remote video was dead. **Root cause: the DRI render nodes swapped**
(NVIDIA driver now loads before i915, likely since the Frigate GPU work) — `renderD128`
became the NVIDIA card and `renderD129` the Intel iGPU, but CT105 still passed `renderD128`,
so QSV pointed at the NVIDIA node. See the iGPU section below for the fix (pass **both** nodes
+ point Jellyfin at `renderD129`). Verified: the exact failing ffmpeg command now runs on QSV
at ~23× realtime.

**...and even then, one more blocker: tunnel throughput.** With the GPU fixed, the transcode
completed (`FFmpeg exited with code 0`) but the client still only got a poster — the media
couldn't reach it fast enough. The home→VPS upload leg (AU→DE, ~280 ms RTT) was collapsing to
~1.5 Mbps under **cubic**. Fixed by switching to **BBR** congestion control (→ ~19 Mbps).
Full detail in [remote-access.md](remote-access.md) § Throughput. So remote video needed
**three** infra fixes: (1) proxy remote-detection + bitrate cap, (2) GPU render-node swap,
(3) BBR on the intercontinental upload path — plus a **fourth, client-codec** fix below.

### The final wall: MP3-in-MPEG-TS won't play on Jellyfin Android (2026-07-04)

After the three infra fixes, the access log proved the Android app fetched `PlaybackInfo`,
`master.m3u8`, `main.m3u8` and segments `0.ts`–`10.ts` **all HTTP 200 full-bytes** — yet still
showed a placeholder ("Playback stopped … Stopped at 0 ms"). So it's **decode, not delivery**.

Root cause: remote forces the XVID/AVI video to transcode to h264. The app's profile sends
`AudioCodec=aac,mp3`, so Jellyfin **copies** the source MP3 into the MPEG-TS segments
(`-codec:a:0 copy`, advertised as `mp4a.40.34`). ExoPlayer can't play MP3-in-TS/`mp4a.40.34`
→ errors at frame 0. Verified: forcing `AudioCodec=aac` yields clean `h264 + mp4a.40.2 (AAC)`
segments. Music works because it's plain MP3 *audio* playback (different path); local video
works because it **direct-plays** the raw file (no transcode).

The audio-copy decision is driven by the **client's** device profile (sent per request), which
Jellyfin has no server-side override for. Fixes: **(a) client** — in the Jellyfin Android app
switch the player to **libVLC** (handles MP3-in-TS) or update the app; **(b) server** —
re-encode the MP3 audio of affected files to AAC.

### Resolution: batch MP3→AAC remux of all AVIs (2026-07-04)

Chose the server-side fix. Confirmed the diagnosis first with a controlled test on the user's own
device: **Better Call Saul** (h264 **+ AAC**, mp4) direct-plays smooth remotely, while **Angry Boys**
(XVID **+ MP3**, avi) is jerky/dead — *same* 280 ms AU↔DE hairpin, same phone → the differentiator is
the audio codec, not the network. Proof file `Angry Boys - S01E99.mkv` (video stream-copied, MP3→AAC)
played smooth remotely, so the fix was validated *before* touching the library.

Then batched every AVI (296 files, 111 GB) with the **same lossless audio-fix**:
```
ffmpeg -nostdin -y -fflags +genpts -i in.avi -map 0:v -map 0:a \
       -c:v copy -c:a aac -b:a 192k out.mkv        # video bit-for-bit copy; only audio re-encoded
```
- `-fflags +genpts` is **required** — AVI/XVID packets carry no timestamps and MKV refuses to mux
  without them (`Can't write packet with unknown timestamp`).
- `-nostdin` is **required** — inside a `find … | while read` loop ffmpeg otherwise eats the piped
  file list as interactive keypresses and the loop runs one file then dies.
- Ran `nice -n15 ionice -c2 -n7`, one file at a time, verify-duration-then-delete-source, because the
  media drive is the Frigate-shared spindle (the system's scarce resource — see README I/O section).
- XVID video is **still transcoded on remote play** (XVID isn't remote-direct-play-friendly), but the
  audio is now clean AAC, so the transcode path no longer produces MP3-in-TS. Result: **294/296 clean.**
- **2 truncated sources** (not a conversion fault): `Bones - S01E18` (data ends ~19 of 23 min) and
  `Ape-Man … Body` (ends ~8 of 45 min) — a full decode+re-encode recovered the same short duration,
  proving the tail bytes don't exist. Kept the partial MKV, removed the AVI; **both need replacing.**

Script `/root/fix-avi-audio.sh` on CT105. A Jellyfin **library scan** is needed after (filenames
changed `.avi`→`.mkv`); this also drops watched/resume state for these renamed items.

## Setup

Native Jellyfin (apt repo `install-debuntu.sh`, not Docker — single app, lighter).

SSH: same `proxmox_ed25519` key authorized for `root@192.168.1.14`.

## iGPU Transcoding (Intel UHD 630 QSV)

**Why iGPU:** GTX 1650 reserved for Frigate + Immich (always-on VRAM). Intel iGPU was free
after Frigate moved to NVIDIA.

Passthrough in `/etc/pve/lxc/105.conf` — **pass BOTH render nodes** (see render-node-swap
warning below):
```
dev0: /dev/dri/renderD128,gid=992
dev1: /dev/dri/renderD129,gid=992
```
Host render gid=993 → idmapped to gid=992 inside CT. `jellyfin` user is in `render` + `video` (44)
groups by installer.

**⚠ Render-node numbering is NOT stable across reboots.** Which card is `renderD128` vs
`renderD129` depends on driver load order (nvidia vs i915). It **flipped on 2026-07-04** and
broke all remote video (QSV pointed at the NVIDIA node → `Device creation failed`). Map is by
**PCI address**, which IS stable: `00:02.0` = Intel iGPU, `01:00.0` = NVIDIA GTX 1650. Check with
`ls -l /dev/dri/by-path/` on the host.

**Fix that survives future flips:** pass *both* nodes into the CT (above). Jellyfin's ffmpeg
auto-selects the Intel GPU by `-init_hw_device vaapi=va:,vendor_id=0x8086` — so whichever number
Intel lands on, it's found, as long as both nodes are present. `VaapiDevice` in `encoding.xml` is
set to `/dev/dri/renderD129` (Intel today); the vendor_id auto-select is what actually drives
selection, so a flip won't rebreak it.

Verified as the **jellyfin user**: `vainfo` (iHD driver) shows UHD 630 H264 VLD decode + EncSlice
encode on `renderD129`, and a live `h264_qsv` transcode runs at ~23× realtime.

QSV device configured in Jellyfin admin (`encoding.xml` VaapiDevice): `/dev/dri/renderD129`.

## Media Drive

WD My Passport 465G USB, GPT + ext4, `LABEL=media`, `UUID=d45b1e52-e60e-4a4b-b6bc-da5f43dc2fed`, `-m 0`.

Host `/etc/fstab`:
```
UUID=d45b1e52-... /mnt/media ext4 defaults,nofail,x-systemd.device-timeout=10 0 2
```
Bind into CT via `/etc/pve/lxc/105.conf`:
```
mp0: /mnt/media,mp=/media
```
Ownership on host: `chown 100102:100105` (= idmapped CT jellyfin 102:105), `chmod 775`.
Folders: `/media/{movies,tv,music,podcasts}`. Media is OFF the thin pool.

## Media Synced

- Movies: 15G
- TV: 193G
- Podcasts: 61G
- Music: ~69G (initial copy from exFAT HDD)

Total ~268G, 2572 files. Source exFAT drive can be unplugged.

## Music Sync (ongoing)

**SECURITY CONSTRAINT:** OneDrive credentials must NOT be stored on Proxmox host.
rclone config + cron were removed from the host.

Ongoing sync is Windows-side via robocopy from dev workstation over SMB:
```
robocopy "C:\Users\tim\OneDrive\Music" "\\192.168.1.3\media\music" /MIR /Z /R:3 /W:5
```
Set up Windows Task Scheduler for ongoing sync, or run manually.
**`/MIR` mirrors** → anything in `/media/music` not in `OneDrive\Music` gets deleted on next sync.
(Only **Music** has this OneDrive `/MIR` sync. Movies/TV/Podcasts were one-time copies — no mirror.)

### Sara's music collection — provenance via genre tag (2026-06-29/30)

Merged Sara's beets-organized collection (from `sara_tmp` cleanup) into the **one** Music library and
marked it with a **genre `"Sara"`** so it's filterable in Jellyfin (Music library → filter Genre →
"Sara"). `"Sara"` is **appended** to existing genres, so normal genre browsing is unaffected.

Merge rule (script `tag_sara.py` + `merge_plan.tsv`, keyed on normalized artist+album):
- **135 non-colliding albums** copied into `/media/music`, tagged `Sara`.
- **26 collisions** (album already in main library) → **kept the main copy**, tagged it `Sara` (so the
  collection filter includes it regardless of which physical file won).
- **1 exception** — `Clare Bowditch and The Feeding Set/The Moon Looked On` (Sara's 12 vs main's 11
  tracks): kept main + tagged `Sara`; Sara's fuller copy parked at **`/mnt/media/_sara_review/`** for
  manual reconciliation of the extra track.
- 1,912 files tagged total, perms 644 (Jellyfin-readable), owner root:root.

**OneDrive consistency:** because Music is `/MIR`-synced, Sara's additions were pushed **back into
`C:\Users\tim\OneDrive\Music`** (135 albums copied there + the 27 collision albums re-tagged `Sara`
in place) so OneDrive == the Jellyfin collection and a future `/MIR` won't delete Sara's music.
All Windows-side (OneDrive creds never on the server). Re-tag via `tag_sara_onedrive.py`.

## Libraries

| Library | Path |
|-|-|
| Movies | /media/movies |
| TV | /media/tv |
| Podcasts | /media/podcasts |
| Music | /media/music |

## Disk Management

**Disk resized 2026-06-28:** 8G → 10G to accommodate transcode cache.

LXC disk resize procedure (shrink — REVERSE order from growing):
```bash
# On Proxmox host (CT must be stopped)
pct stop 105
e2fsck -f /dev/pve/vm-105-disk-0
resize2fs /dev/pve/vm-105-disk-0 9G          # resize fs first
lvreduce -L 10G /dev/pve/vm-105-disk-0       # then shrink LV
pct start 105
```

**Transcode cache cleanup cron** (`/etc/cron.d/jellyfin-transcode-cleanup` inside CT 105):
```
0 * * * * root find /var/cache/jellyfin/transcodes -type f -mmin +60 -delete
```
At ~6 Mbps transcode bitrate, a 2hr movie ≈ 2.7GB. **Changed 2026-06-29 from daily-3am → hourly**
after the disk hit 100% mid-session (see below).

**Disk-full incident 2026-06-29:** the 10G rootfs filled to 100% during playback (6.8G of stale
transcode cache from a TV repeatedly restarting the stream). Symptoms: playback stalls, and a
Jellyfin restart while full silently failed to write config. Cleared with the manual command below.
The daily cron wasn't enough mid-session → switched to hourly. If a single long film still fills the
disk, options: **lower the profile transcode bitrate** or **grow the CT disk** (thin pool is
overcommitted per README — check real pool usage first).

Manual cleanup if needed:
```
pct exec 105 -- bash -c "rm -rf /var/cache/jellyfin/transcodes/*"
```

## DLNA (installed 2026-06-28)

DLNA moved to a separate plugin in Jellyfin 10.9+. Plugin version: **11.0.0.0**.

Install: downloaded `dlna_11.0.0.0.zip` from `repo.jellyfin.org`, extracted to
`/var/lib/jellyfin/plugins/Dlna/` (had to clear 5GB stale transcode cache first to make room).

### Hisense VIDAA TV (192.168.1.109) — DLNA PlayTo working

Device profile: `/var/lib/jellyfin/plugins/Dlna/profiles/Hisense VIDAA TV.xml`
Local copy: `C:\Users\tim\OneDrive\dev\proxmox\docs\assets\hisense-vidaa-profile.xml`

**Design:** No video `DirectPlayProfiles` — forces ALL video through TS transcoding (h264/aac).
`ResponseProfile` maps `ts/mpegts → video/mpeg`. This is necessary because:
- Jellyfin DLNA `ContentDirectory` always serves native file MIME type (ignores device profiles)
- ffprobe identifies MP4 as format `"mov,mp4,..."` → Jellyfin uses `"mov"` → `video/quicktime`
- Hisense TV rejects `video/quicktime` as "unsupported file"
- `ResponseProfiles` only apply to the **PlayTo path** (not ContentDirectory browse)
- Fix: use DLNA PlayTo (Jellyfin Android app → cast icon → select TV) so the TS stream with
  `video/mpeg` MIME is what the TV receives

**How to play:** Jellyfin Android app → select content → cast icon → select "Hisense VIDAA TV"

**DMP (TV browsing Jellyfin):** Still shows "unsupported file" for MP4s — ContentDirectory MIME
type issue, not fixable via profile. Use PlayTo instead.

### Default.xml changes (2026-06-28)

`/var/lib/jellyfin/plugins/Dlna/profiles/Default.xml` was modified during DLNA debugging:

1. `ResponseProfile` updated: `container="mp4,m4v,mov"` with `mimeType="video/mp4"` and `<Conditions />`
2. **Video DirectPlay wildcard REMOVED:** `<DirectPlayProfile container="" type="Video" />` was
   deleted. If non-Hisense DLNA clients have "unsupported" issues browsing ContentDirectory,
   restore this line to `<DirectPlayProfiles>`.

### Lip-sync, seek & HW fixes (2026-06-29)

Debugging a lip-sync complaint on `The Witches (1990).avi` (XVID/MP3) over DLNA PlayTo. Profile
backups on CT: `Hisense VIDAA TV.xml.bak-20260629`, `encoding.xml.bak-20260629`.

- **Lip-sync → MP3-in-TS was being copied.** The TS `TranscodingProfile` had
  `audioCodec="aac,ac3,mp3"`, so for MP3 sources Jellyfin used `-codec:a copy` → MP3 muxed into
  MPEG-TS (a bad combo; TS expects AAC/AC3), drifting against the QSV-re-encoded video.
  **Fix:** removed `mp3` → `audioCodec="aac,ac3"`, so MP3 now transcodes to AAC. Verified in the
  ffmpeg log (`-codec:a:0 libfdk_aac`).
- **Seek = "network error" then "restarts movie".** TS profile had `estimateContentLength="false"`
  → no content length → TV can't do positioned (byte-range) requests, so it refetches from 0.
  **Fix attempt:** `estimateContentLength="true"` + `transcodeSeekInfo="Bytes"` on the TS video
  profile. **Untested by user** — DLNA positioned-seek on a live transcode is inherently flaky on
  VIDAA; if it still restarts, the durable fix is a native Jellyfin client (Fire Stick / Chromecast
  w/ Google TV), which also drops the whole forced-transcode chain.
- **HW transcode regression:** `HardwareAccelerationType` was `none` (user had toggled HW transcode
  off), dropping to software libx264. Restored to **`qsv`** in `/etc/jellyfin/encoding.xml`
  (VaapiDevice `/dev/dri/renderD128`, EnableHardwareEncoding true).

**Recurring theme:** the DLNA path is a stack of workarounds (MIME rejection → forced transcode →
MP3 sync → seek failures). A native Jellyfin client avoids all of it via DirectPlay. See action J.

### DLNA Architecture Notes

- **DMP** (Digital Media Player): TV browses Jellyfin ContentDirectory and requests streams itself
- **DMR** (Digital Media Renderer): TV accepts push commands from a controller
- **DLNA PlayTo** = Jellyfin acting as DMC (controller), pushing `SetAVTransportURI` to TV (DMR)
- ContentDirectory always serves native MIME regardless of device profile — `ResponseProfiles`
  only affect the PlayTo path

## Chromecast

**1st gen Chromecast is incompatible with Jellyfin (confirmed 2026-06-28).**

Google EOL'd the Cast SDK used by 1st gen Chromecast in April 2024. Jellyfin's Cast receiver
requires a newer SDK. YouTube still works on 1st gen because it uses a native receiver baked into
firmware — not the same code path.

No fix possible — hardware limitation.

**Recommendation:** Chromecast with Google TV or Amazon Fire Stick for native Jellyfin app.

## Pending

- [ ] Enable 2FA / strong credentials (Jellyfin is internet-exposed via sslip.io)
- [ ] Caddy rate-limit / fail2ban on VPS
- [ ] Consider restoring video DirectPlay wildcard to Default.xml if other DLNA clients have issues
- [ ] **Native client (Fire Stick / Chromecast w/ Google TV)** — fixes DLNA seek/lip-sync for good
- [ ] Verify DLNA seek works after the `estimateContentLength`/`Bytes` change (untested)
- [ ] Watch disk: if a single film fills the 10G rootfs, lower transcode bitrate or grow disk
