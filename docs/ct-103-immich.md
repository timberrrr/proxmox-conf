# CT 103 — Immich

- **IP:** 192.168.1.12
- **UI:** http://192.168.1.12:2283
- **Status:** running, healthy
- **Specs:** unprivileged, nesting, 4c/10G RAM/4G swap, 250G disk, Debian 13

RAM was increased from 4G to 10G (live, no restart) after ML worker OOM-killed loading SigLIP2.
Host has 15GB total — watch host RAM (limits not reserved; Ollama stopped helps).

## Setup

Docker-in-LXC (unprivileged + nesting). Official Compose stack at `/opt/immich`.

**Key paths:**
- Compose: `/opt/immich/docker-compose.yml` + `docker-compose.override.yml`
- Library: `/opt/immich/library/`
- Postgres: `/opt/immich/postgres/`

**`.env`:**
```
UPLOAD_LOCATION=/opt/immich/library
DB_DATA_LOCATION=/opt/immich/postgres
TZ=Australia/Brisbane
DB_PASSWORD=<randomized>
```

**Admin account:** Tim, timbo@beanicus.com
NOTE: `shouldChangePassword` still set — change when convenient.

**API key** (for `/api` on `:2283`, e.g. dedup/jobs/config):
```
PTRYf2st9fB9Uyrhhld7lyODJMJVHFaksAiSquMhMC4
```
_(SENSITIVE — regenerate in Account Settings if leaked)_

## GPU Passthrough

- Same cgroup/bind pattern as CT 101 (GTX 1650 passthrough)
- NVIDIA 595.71.05 userspace in CT (`--no-kernel-module`)
- `nvidia-container-toolkit` with `no-cgroups=true` (required for unprivileged LXC)
- ML via `immich-machine-learning:v2-cuda` with `runtime: nvidia, NVIDIA_VISIBLE_DEVICES=all`
  in `docker-compose.override.yml` (more reliable than hwaccel deploy-reservation in LXC)

## Library State (as of 2026-06-22)

- **14,260 assets** (13,653 images + 607 videos, ~89GB)
- Smart search: working (SigLIP2-384)
- Face recognition: 477 people clustered
- Duplicate detection: complete
- Storage template: `{{y}}/{{y}}-{{MM}}/{{filename}}` — reorg complete, 0 failed

## Photo Import

Transferred from OneDrive via Windows laptop → tar-over-ssh → `/opt/immich/import-staging` →
`immich-cli` managed upload.

**Why not rclone:** rclone's shared OneDrive `client_id` is refused for content downloads on
OneDrive Personal (metadata/listing works; file downloads return "Unauthenticated"). ~728MB got
through before the shared client was cut. If rclone path is ever needed: create own Azure app
registration for `client_id`/`secret`.

**Transfer method used:** OneDrive desktop client hydrates Pictures locally ("Always keep on this
device"), then LAN transfer via `tar -czf - <dir> | ssh root@192.168.1.12 "tar -xzf - -C /opt/immich/import-staging"`.

## ML Model

**Current:** `ViT-SO400M-16-SigLIP2-384__webli` (SigLIP2, 384px)

Model progression tried:
| Model | Fits 4GB? | Notes |
|-|-|-|
| ViT-B-32__openai | Yes | Default, weak |
| ViT-SO400M-14-SigLIP-384__webli | Yes (~3.3GB) | SigLIP v1, English-only |
| ViT-SO400M-16-SigLIP2-384__webli | Yes (~3.4GB) | **Current — multilingual** |
| ViT-SO400M-16-SigLIP2-512__webli | NO | CUBLAS_STATUS_ALLOC_FAILED + SIGKILL |

**4GB GPU caveat:** SigLIP2 multilingual → 256k vocab text tower (1.1GB) + visual model (1.6GB)
cannot co-reside on 4GB. Normal use: idle-unload TTL (300s) makes them take turns → search works.
Symptom if contention: smart search 500s ("textual model failed to load"). Mitigation: lower
`ML_MODEL_TTL`, or switch to SigLIP v1 (English-only, small text tower, co-resides cleanly —
needs another ~1.5hr re-index).

**Job restart gotcha:** Changing model mid-run leaves stuck "active" jobs blocked on model load;
`force-start` returns 400 while `active>0`. Fix: empty queue + restart `immich_server` to clear,
then `force-start` with `force:true` (re-encodes all for consistent embedding space).

Endpoint note: system config is `/api/system-config` (NOT `/api/admin/config` → 404).

## Pending

- [ ] Fix 7 bogus-date photos: 5 in "2037" folder, 2 in "2026" (import-date fallback) — edit
  dates in Immich UI (bulk edit). Harmless otherwise.
- [ ] Change admin password (`shouldChangePassword` still set)
- [ ] Immich → OneDrive backup (see below — after old HDD reconcile)

## HDD Reconciliation

**2026-06-22:** External exFAT HDD "my pictures" (22,924 files/106GB) reconciled vs Immich by
SHA-1 (`/api/assets/bulk-upload-check`). 19,241/19,261 media already archived; imported the 20
missing + 1 extensionless AVI (gporo1) + unique zip media; then deleted all hash-confirmed files
+ junk. "my pictures" folder removed; rest of HDD untouched; drive unmounted.

**Still to reconcile (user):** loose images in HDD root + other folders (my videos, etc.) — not
yet checked against Immich.

## Planned: Immich → OneDrive Backup

**SECURITY CONSTRAINT:** OneDrive credentials must NOT be stored on Proxmox host.
(User: "the OneDrive is too sensitive were it compromised in any way.")

Backup path: Immich → rclone on CT 103 → OneDrive (one-way, passive mirror).

**What to back up:**
- `/opt/immich/library/library/` (originals, year/month)
- `/opt/immich/library/backups/` (nightly DB dump — needed for full restore)
- SKIP `thumbs/` + `encoded-video/` (regenerable)

**Use `rclone copy` or `sync --backup-dir=.../deleted/<date>`** — NOT plain `sync` (plain sync
propagates deletions and would wipe the backup).

**Sequencing:** Keep current `OneDrive\Pictures` until new Immich→OneDrive backup is verified
(always one offsite copy). Old `OneDrive\Pictures` becomes redundant after and can be replaced
by the Immich mirror.

**Caveat:** rclone shared OneDrive client failed on downloads (Personal throttling). Test upload
first; if unreliable, create own Azure app `client_id` (durable fix).
