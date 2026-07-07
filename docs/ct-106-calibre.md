# CT 106 — Calibre-Web (ebooks)

- **IP:** 192.168.1.15 (proposed — next free after Jellyfin .14)
- **UI:** http://192.168.1.15:8083
- **External:** TBD via Caddy/WireGuard (CT 107), same pattern as Jellyfin — see Pending
- **Status:** BUILT 2026-06-28, running. **5,662 books** loaded from "Sara Backup" (2026-06-29).
  SMTP + Send-to-Kindle working from the web UI. NOTE: `config_log_level` left at DEBUG(10).
- **Specs:** unprivileged, nesting, 2c / 1G / 512swap, 10G rootfs on local-lvm, Debian 13.1-2
- **Versions:** calibre 8.5 (apt), Calibre-Web 0.6.26 (venv at `/opt/calibre-web`)

## Design

Two pieces, one running service:

- **Calibre binaries** (`calibre` apt pkg — `calibredb`, `ebook-convert`): headless library
  management + format conversion. No GUI/daemon; just CLI tools on PATH.
- **Calibre-Web** (native, Python venv): the web frontend + reader + user accounts + **OPDS feed**.
  Calls `ebook-convert` for on-the-fly conversion and send-to-Kindle.

Lighter than running headless Calibre-server as a second daemon, and consistent with CT 105's
"native, single service" approach.

### Format policy
`.mobi` is a dead format (Amazon retired it). Let Calibre convert the library to **`.epub`** as
canonical (best reader/OPDS support everywhere). Keep `.mobi` only for actual Kindle send-to-device.

## Build steps

### 1. Create the LXC (on Proxmox host)
```bash
pct create 106 local:vztmpl/debian-13-standard_*_amd64.tar.zst \
  --hostname calibre \
  --unprivileged 1 --features nesting=1 \
  --cores 2 --memory 1024 --swap 512 \
  --rootfs local-lvm:10 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.15/24,gw=192.168.1.1 \
  --onboot 1 --start 1
```

### 2. SSH key (match the rest of the fleet)
```bash
pct exec 106 -- mkdir -p /root/.ssh
pct exec 106 -- bash -c 'cat > /root/.ssh/authorized_keys' < ~/.ssh/proxmox_ed25519.pub
```

### 3. Bind the media drive (books folder)
The `media` drive is already mounted on the host at `/mnt/media` (see ct-105). Add a `books`
subfolder and bind it into CT 106. Edit `/etc/pve/lxc/106.conf`:
```
mp0: /mnt/media,mp=/media
```
Calibre-Web runs as **root inside the CT** (in-CT uid 0 → host uid **100000** for unprivileged).
On the host, create + own the books dirs accordingly:
```bash
mkdir -p /mnt/media/books/incoming /mnt/media/books/library
chown -R 100000:100000 /mnt/media/books && chmod -R 775 /mnt/media/books
```
(Note: existing `/mnt/media` content is owned `100102:100105` = CT 105 jellyfin. The `books/`
subtree is owned by CT 106 root, separate. If you later switch to a dedicated `calibre` service
user, re-chown to that user's idmapped host uid.)

### 4. Install Calibre binaries (conversion + metadata)
```bash
pct exec 106 -- apt update
pct exec 106 -- apt install -y calibre python3-venv python3-pip imagemagick
# headless: pulls Qt deps but no X needed for calibredb/ebook-convert
```

### 5. Install Calibre-Web (native venv)
```bash
pct exec 106 -- bash -lc '
  python3 -m venv /opt/calibre-web
  /opt/calibre-web/bin/pip install --upgrade pip
  /opt/calibre-web/bin/pip install calibreweb
'
```

### 6. Initialize an empty Calibre library
```bash
pct exec 106 -- bash -lc '
  mkdir -p /media/books/library
  calibredb add --empty --library-path /media/books/library
'
# this creates /media/books/library/metadata.db that Calibre-Web points at
```

### 7. systemd service for Calibre-Web
`/etc/systemd/system/calibre-web.service` inside CT 106:
```ini
[Unit]
Description=Calibre-Web
After=network.target

[Service]
Type=simple
User=root
ExecStart=/opt/calibre-web/bin/cps -p /opt/calibre-web/app.db
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```bash
pct exec 106 -- systemctl enable --now calibre-web
```
(Consider a dedicated `calibre` service user instead of root before exposing externally —
update the bind-mount ownership in step 3 to match its uid.)

### 8. First-run config (web UI, http://192.168.1.15:8083)
- Default login: **admin / \<vendor default — see calibre-web docs\>** → change immediately.
- **Database location** (Admin → "Location of Calibre Database") = `/media/books/library` (the dir
  containing `metadata.db`). This field rejects a file path — it wants the *directory*.
- **Converter** (Admin → Basic Config → **Calibre E-Book Converter Settings** → **"Path to Calibre
  Binaries"**) = `/usr/bin` — note this is the **directory** holding `ebook-convert`, NOT the binary
  path itself. Enables conversion + send-to-Kindle.
- **OPDS** is a built-in feed at `http://192.168.1.15:8083/opds` (no toggle to "enable"; point reader
  apps — KOReader, Marvin, Moon+, Apple Books — at that URL + login). Keep "Use Basic Authentication
  for OPDS" on, especially before external exposure.
- Create per-user accounts; restrict public/anon access if exposing externally.

## Getting books onto the server

For small ongoing additions, simplest is the **Calibre-Web UI upload** (Admin must enable
"Enable Uploads" in Feature Configuration). For bulk loads see the "Initial library load" section
below — the SMB share needs the `mediauser` credential, and large transfers over Wi-Fi are slow.

Batch-import staged files, then convert if desired:
```bash
pct exec 106 -- calibredb add -r /media/books/incoming --library-path /media/books/library
# optional: convert mobi -> epub (canonical)
pct exec 106 -- bash -lc 'for f in /media/books/incoming/*.mobi; do ebook-convert "$f" "${f%.mobi}.epub"; done'
```

## Initial library load — "Sara Backup" (2026-06-29)

Loaded a large old ebook collection (a zipped Calibre library, `Books Backup.zip`, ~5.5 GB,
6,044 mobi + their covers/opf = 28,453 entries) plus 16 loose epub classics.

### Bulk-transfer technique (the media drive is on a spinning USB HDD)
- **SMB share needs the `mediauser` password** (`valid users = mediauser` in smb.conf) — an
  unauthenticated Windows session gets read-only and every write is denied. scp-over-SSH as root
  is the no-creds path.
- **Workstation is on Wi-Fi** → scp to the host crawled (single-digit MB/s). For tens of GB, far
  faster to **physically move the drive to the workstation and mount ext4 via WSL2**:
  - Release on host: stop CT 105 + 106, `umount /mnt/media` (kill any holding `smbd`/processes
    first — `fuser -m /mnt/media`), unplug.
  - On Windows (elevated): `wsl --mount \\.\PHYSICALDRIVE<N> --partition 1 --type ext4`.
  - Copy from WSL **as root** (`wsl -u root`) — the default user can't write dirs owned by the
    idmapped uids. Then `chown` to match: movies/tv `100102:100105`, books `100000:100000`.
  - Return: `wsl --unmount \\.\PHYSICALDRIVE<N>`, replug to host, `e2fsck -p /dev/sda1`,
    `mount /mnt/media`, start CTs.
- **Spinning HDD = small files are brutally slow.** Big sequential files (movies) fly; thousands of
  tiny files (extracting 6 k ebooks) thrash it. Transfer the **single zip** (sequential, fast) and
  **extract on the server**, not file-by-file over WSL.
- **Stuck `find`/`du` probes on a busy/hung USB drive go into uninterruptible (D) sleep** and can't
  be `kill -9`'d; they hold the mount so `wsl --unmount` fails ("Resource device"). Recover with
  `wsl --shutdown` then physically replug the drive.

### Extraction gotcha — this zip is non-standard zip64
`tar`, .NET `System.IO.Compression`, python `zipfile`, **and Info-ZIP `unzip` 6.0 all fail** on it
(unzip: "bad zipfile offset 4294967296" = 2³² overflow + false zip-bomb trip; python: "Bad magic
number for file header"). **Only `7-Zip` (`p7zip-full`, `7z x`) extracts it reliably.** The central
directory is intact (all tools can *list* it), so it's a parser problem, not corruption. p7zip-full
is now installed on the host.

### Import procedure used
Host script `/root/ebook_import.sh` (gated — only deletes the zip after ≥6,000 files extract):
```bash
7z x "/mnt/media/books/incoming/Books Backup.zip" -o/mnt/media/books/incoming/ -y
chown -R 100000:100000 /mnt/media/books/incoming
pct exec 106 -- calibredb add -r /media/books/incoming --library-path /media/books/library
```
NOTE on paths: extraction runs on the **host** (`/mnt/media`), but `calibredb` runs **inside CT 106**
(`/media`) — the bind mount maps `/mnt/media` → `/media`. Mixing them up = FileNotFoundError.
`calibredb add` **copies** books into the library; it does NOT consume `incoming/` — clear the
staging dir manually after a verified import.

## Library load result (2026-06-29)

Import finished: **5,662 books** (5,644 ebook files). The raw import created ~6,852 records but
**1,189 were phantom/empty** (no format file — DRM/unreadable mobi + `.opf`-only entries from the old
backup being itself a Calibre library, + a messy multi-attempt import). Cleaned via
`/root/calibre_housekeep.py` (removes format-less records + exact title/author dupes, keeps lowest id);
also removed a `Books Backup.zip` accidentally imported as a "book". Verified no real files lost
(folder count + library size unchanged after removal). Staging `/media/books/incoming/` cleared.

## Email / Send-to-Kindle (2026-06-29)

Three separate things must all be right; debugging notes:

1. **SMTP encryption vs port.** Server `mail.thornaero.com:465` (SiteGround, AUTH LOGIN/PLAIN).
   Calibre-Web encryption codes are **`0=None, 1=STARTTLS, 2=SSL/TLS`** (counterintuitive order).
   Port 465 is implicit SSL → needs **`mail_use_ssl=2`**. It was `0` (none) → fixed to `2`.
   (`1`/STARTTLS on 465 gives `SMTPServerDisconnected: Connection unexpectedly closed` — plaintext
   connect to an SSL port.) Settings live in `app.db` `settings` table; restart cps after changes.
2. **Per-user requirements.** "Send to E-Reader" only appears if the user has the **Download role**
   AND a **`kindle_mail`** set. `tim` was `role=0` (no perms) → set to `478` (all but admin); his
   `kindle_mail` was already set. `admin` has no kindle_mail (so neither account showed the button,
   for different reasons).
3. **Amazon approved-sender.** `no-reply@thornaero.com` must be on Amazon's *Approved Personal
   Document E-mail List* (Manage Your Content & Devices → Preferences → Personal Document Settings)
   or Amazon **silently discards** the mail — server reports success, book never arrives.

Useful: DEBUG logging via `app.db settings.config_log_level=10`; log at `/root/.calibre-web/calibre-web.log`.

## "Calibre Web Companion" app — not compatible with vanilla CW

The Flutter app **Calibre Web Companion** is built for **Calibre-Web Automated (CWA)**, a fork with
extra API endpoints. Against vanilla Calibre-Web 0.6.26 its calls **404** (confirmed in client log:
`findMagicShelvesContainingBook` → 404; `sendBookViaEmail` produced **no** server-side `convert`
line) — so its "Send to email" silently no-ops while the app reports success. SMTP/roles are fine;
it's an app↔server API mismatch.
- **Workaround:** use the Calibre-Web **web UI** to send (works).
- **To make the app work:** migrate to **CWA**, which is Docker-only. Feasible in-place on CT 106
  (already `nesting=1`): run the CWA container, point at the existing `/media/books/library` (standard
  Calibre format → all books carry over), retire the native venv. Caveats: 10G rootfs is tight
  (Docker+image ~2–3G → likely needs disk grow), 1G RAM is modest. **Not implemented** — decision pending.

## Pending

- [x] Build the CT — done 2026-06-28
- [x] First-run web config — DB location, converter dir (`/usr/bin`), OPDS — done 2026-06-29
- [x] Import 5,662 books, clean phantom/dup records, clear staging — done 2026-06-29
- [x] SMTP fixed (`mail_use_ssl=2`) + `tim` granted roles (478) — done 2026-06-29
- [ ] Add `no-reply@thornaero.com` to Amazon approved-sender list (else sends silently dropped)
- [ ] Change admin password from vendor default (still unchanged — live credential, do not record here)
- [ ] Set `config_log_level` back to 20 (INFO) — currently 10 (DEBUG) from troubleshooting
- [ ] Decide: migrate to **CWA (Docker)** for companion-app support, or stay vanilla + web-UI sends
- [ ] Decide dedicated service user vs root before external exposure
- [ ] Expose via Caddy/WireGuard (CT 107) — apply same 2FA / hardening as Jellyfin
- [ ] Optional: convert library mobi → epub (canonical format)
