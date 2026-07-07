# CT 104 — Frigate (NVIDIA GTX 1650)

- **IP:** 192.168.1.13
- **UI:** http://192.168.1.13:5000
- **Status:** **LIVE** — 3 cameras streaming + detecting on the **GPU (ONNX/YOLOv9-s @ 640, ~27 ms)**; all 4 follow-ups done.

## Cameras LIVE (2026-06-30)

3 Tapo cameras on the IoT VLAN 30, RTSP account **`peachycams` / `Plop1234!`** (camera-account is
per-camera & separate from the TP-Link login; `.106` needed a reboot for its password to apply):

| Name | IP (VLAN30) | main `stream1` | sub `stream2` |
|-|-|-|-|
| lower_driveway | 192.168.30.105 | 1920×1080 h264 | 640×360 |
| upper_driveway | 192.168.30.107 | 1920×1080 h264 | 1280×720 |
| backyard | 192.168.30.106 | 1920×1080 h264 | 1280×720 |

RTSP URL form: `rtsp://peachycams:Plop1234!@<ip>:554/stream1` (record) and `/stream2` (detect).
Test a stream without the container running, using the image's bundled ffprobe:
```
docker run --rm --network host --entrypoint /usr/lib/ffmpeg/7.0/bin/ffprobe \
  ghcr.io/blakeblackshear/frigate:stable-tensorrt -v error -rtsp_transport tcp \
  -i "rtsp://peachycams:Plop1234!@192.168.30.105:554/stream2" -show_entries stream=codec_name,width,height -of csv=p=0
```

**Live config** (`/opt/frigate/config/config.yml`; local copy in jobs tmp `frigate-config.yml`):
- Detector: **`onnx`** (GPU, ~27 ms — YOLOv9-s @ 640, see follow-up #3). NVDEC GPU decode via `ffmpeg: hwaccel_args: preset-nvidia`.
- Objects tracked: **person, dog, car, cat** (all COCO-80).
- **Recording = object-only** (disk-safe, Frigate 0.17 schema — note `record.retain.{days,mode}` was
  REMOVED in 0.17):
  ```yaml
  record:
    enabled: true
    continuous: { days: 0 }      # no 24/7
    motion: { days: 0 }          # no keep-all-motion (driveways = too much)
    alerts: { retain: { days: 30 } }
    detections: { retain: { days: 30 } }
  ```
  → only keeps clips overlapping a tracked object, 30 days.

### ⚠️ Two independent post-reboot races (both auto-handled now)

**(A) Storage mount-race (caused the 2026-07-04 failure).** The `mp0` bind source
`/mnt/media/frigate` lives on the `nofail` USB drive (`/dev/sda1` → `/mnt/media`). On reboot,
`pve-guests` started CT 104 (`onboot:1`, first in numeric order) **before** the USB mount finished, so
the **non-optional** `mp0` bind source didn't exist → pre-start hook aborts:
```
run_buffer: Script exited with status 2
lxc_init: Failed to run lxc.hook.pre-start for container "104"
TASK ERROR: startup for container '104' failed
```
Timeline that boot: CT 104 fail @ 21:06:10, `/mnt/media` mounted @ 21:06:18 (**8 s too late**). CTs 105/106
also bind `/mnt/media` but got processed just after the mount landed. The old gpu-fix script *did nothing*
for this because it bailed on a stopped CT (`pct status … | grep -q running || exit 0`) — hence the manual
`pct start 104` needed. **Fix (2026-07-04):** ordering drop-in
`/etc/systemd/system/pve-guests.service.d/wait-media-mount.conf` (`Wants=` + `After=mnt-media.mount`,
soft so a dead drive can't block boot) holds *all* guest autostart until the USB mount settles; also
bumped fstab `x-systemd.device-timeout=30`→`60` for cold-boot spin-up.

**(B) GPU passthrough race.** After a host reboot the NVIDIA driver loads ~260 s late; if CT 104 starts
first its `/dev/nvidia*` bind-mounts become **empty placeholder files** (`nvidiactl` missing) → container
runs but Frigate fails with `NVML: Driver Not Loaded`. Device majors (195 nvidia, 506 uvm, 509 caps) match
the `lxc.cgroup2.devices.allow` rules — purely timing. **Fix = `pct reboot 104`** once `/dev/nvidiactl`
exists; auto-handled by the fix service below.

### Open follow-ups (in progress / not done)
1. **Boot-race auto-fix — DONE 2026-06-30, extended 2026-07-04.** Host oneshot service
   `/etc/systemd/system/frigate-gpu-fix.service` (enabled, `After=/Wants=pve-guests.service`) runs
   `/usr/local/bin/frigate-gpu-fix.sh`, which now covers **both** races: (A) waits up to ~2 min for
   `/mnt/media` to be a mountpoint, then `pct start 104` if it's stopped (self-heals a mount-race
   failure); (B) waits up to ~7.5 min for `/dev/nvidiactl`, then if `pct exec 104 -- nvidia-smi -L`
   fails it does `pct reboot 104` so the GPU binds re-attach. Logs via `journalctl -t frigate-gpu-fix`.
   Backup of the old script at `/usr/local/bin/frigate-gpu-fix.sh.bak-YYYYMMDD`.
   **Validated 2026-07-04** by simulating a stopped CT (`pct stop 104` → run script → started clean,
   storage + GPU + Frigate API healthy). The primary prevention (pve-guests ordering) is still **not yet
   validated through a real host reboot** — verify on the next reboot.
2. **Recordings off the thin pool — DONE 2026-06-30.** `/mnt/media/frigate` (chown `100000:100000`,
   on the USB drive) bound into CT104 via `mp0: /mnt/media/frigate,mp=/opt/frigate/storage` in
   `/etc/pve/lxc/104.conf`. Verified: `df /opt/frigate/storage` → `/dev/sda1` (USB, ~69 G), off the
   82%-full pool. The compose volume (`/opt/frigate/storage:/media/frigate`) is unchanged — it now
   resolves to the USB drive. **Note:** that USB drive is a single 2.5" spinning HDD (WD My Passport)
   *shared with Jellyfin media* — concurrent random I/O (Frigate writes + Jellyfin reads) is the host's
   real bottleneck, not the NIC/USB bus. See README "Host I/O & network topology" for the full analysis
   and the USB-SSD upgrade path.
3. **GPU detector — DONE 2026-07-01.** GTX 1650 runs the **ONNX detector** (onnxruntime-CUDA;
   TensorRT detector is Jetson-only in 0.17). Live model = **YOLOv9-small @ 640** ONNX (28.6 MB) —
   upgraded from the first tiny@320 build for much better detection of small/distant objects on the
   driveway/yard cams. Built on the *workstation* Docker (off the thin pool) via the official export
   Dockerfile (`jobs tmp/yolov9build/`, `--build-arg MODEL_SIZE=s --build-arg IMG_SIZE=640` →
   `yolov9-s-640.onnx`), copied to CT104 `/config/model_cache/yolo.onnx`. Live config:
   ```yaml
   detectors: { onnx: { type: onnx } }
   model:
     model_type: yolo-generic
     width: 640
     height: 640
     input_tensor: nchw
     input_dtype: float
     path: /config/model_cache/yolo.onnx
     labelmap_path: /labelmap/coco-80.txt
   ```
   **Verified (`/api/stats`):** inference **~27 ms** (was ~35 ms CPU / 7 ms on tiny@320), steady GPU
   15–24% (bursts to ~79% under heavy motion), VRAM ~22% (~0.9 GB resident), **0 skipped_fps** on all
   3 cams (`proc_fps == cam_fps`), decode 3% (NVDEC). Lots of headroom remains.
   **Ceiling test (2026-07-01) — -s @ 640 is the max that holds zero drops on this GTX 1650.**
   Bigger models were built and burst-tested; both fail on concurrent multi-camera motion:
   | Model @ 640 | inference | GPU under motion | skipped_fps under motion |
   |-|-|-|-|
   | **-s (LIVE)** | ~27 ms | peaks ~79% | **0** |
   | -m | ~52 ms | saturates 92–93% | 3.4–4.1 (proc_fps collapses to ~1) |
   | -c | ~70 ms | pinned 90–91% | 2.5–4.0 |
   When all 3 cams have motion at once (~19+ det/s demand) the GPU itself saturates, so -m/-c drop ~75%
   of frames *exactly when detection matters most* — and a 2nd detector instance can't help (no spare
   GPU compute). Don't bother re-trying -m/-c here. A bigger card (or fewer/lower-fps cams) would be
   needed to go past -s.
   **Fallbacks in the CT:** -s model at `model_cache/yolo-s-640.onnx.bak`, tiny at
   `yolo-t-320.onnx.bak`; configs `config.yml.bak-cpu` (CPU) and `config.yml.bak-t320` (tiny@320).
   Swap is container-only (`docker restart frigate`), no CT reboot.
4. **Wildlife (possum/kangaroo/snake/deer/pig): NOT possible with stock models** — COCO-80 has no
   Australian wildlife. Requires a **Frigate+** custom model (label snapshots from these cameras). Later.
- **Specs:** unprivileged, nesting, 4c/4G/50G, Debian 13

## What's Done

- Docker-in-LXC + GPU passthrough + `nvidia-container-toolkit` installed
- Image pulled: `frigate:stable-tensorrt` (v0.17.1)
- Frigate runs healthy, NVDEC auto-detected ("Automatically detected nvidia hwaccel")
- Compose at `/opt/frigate`, config at `/opt/frigate/config`

## Remaining (do at camera-setup time, after IoT network ready)

### 1. Wire IoT network — DONE 2026-06-30

Final design (single onboard NIC trunk, not the old nic0/vmbr1 access plan):
- **`nic1` = trunk** on the Orbi port: **native/untagged = management VLAN** (keeps `vmbr0`/192.168.1.3
  reachable — critical, or you lock yourself out), **VLAN 30 = tagged**.
- Host: `nic1.30` → `vmbr30` (see `/etc/network/interfaces`; mgmt `vmbr0` left untouched).
- Frigate CT **dual-homed**: `net0` eth0 `192.168.1.13` (mgmt+UI, default route) + `net1`
  eth1 `192.168.30.13/24` on `vmbr30` (no gateway — VLAN 30 stays on-link, default route stays on mgmt):
  ```
  pct set 104 -net1 name=eth1,bridge=vmbr30,ip=192.168.30.13/24
  ```
  (Ensure `.13` is outside the Orbi IoT DHCP pool.)
- Verified: CT104 → `192.168.30.1` (IoT gw) OK.

**Applying Proxmox network changes remotely is dangerous** — a bad trunk/native-VLAN locks out
management. Use an auto-revert safety net:
```bash
cp /etc/network/interfaces /etc/network/interfaces.bak
systemd-run --on-active=120 --unit=net-autorevert --collect /bin/sh -c \
  'cp /etc/network/interfaces.bak /etc/network/interfaces; ifreload -a'
# ...make changes... ifreload -a ; verify connectivity ; then:
systemctl stop net-autorevert.timer    # cancel revert only if mgmt held
```

### 2. Enable RTSP on Tapo cameras

In the Tapo app, per camera: Advanced → Live View → RTSP stream — enable and note the URL.
- C310: fixed 3MP
- C500 (×2): PTZ 3MP, supports ONVIF autotracking (port 2020)

### 3. Choose detect resolution

The TensorRT engine is per model + input size — built once at runtime from the ONNX. Decide
**before** the engine builds to avoid a rebuild.

| Option | Stream | Notes |
|-|-|-|
| 320 | Sub-stream (640×480) | Faster, lower VRAM, less detail |
| 640 | Main stream (3MP) | Higher-res detect, heavier decode — the goal |

Higher-res detect is the stated goal (3MP main stream + NVDEC), so 640 is the likely choice.

### 4. Supply the detector model

The yolonas ONNX is not auto-fetched in 0.17. Download and place at:
```
/opt/frigate/config/model_cache/yolo_nas_s.onnx
```
Frigate will build the TensorRT engine from it on first camera run (GPU required).

### 5. Add cameras to config

```yaml
cameras:
  c310:
    ffmpeg:
      inputs:
        - path: rtsp://<user>:<pass>@192.168.30.<ip>:554/stream1
          roles: [detect, record]
    detect:
      width: 1920
      height: 1080
  c500_1:
    ...
```

### 6. Recordings storage

Thin pool `pve/data` is already OVERCOMMITTED (384G sizes / 349G pool). Set retention limits
before enabling recordings, or add a dedicated disk for Frigate recordings.

## GPU Notes

- Device passthrough in `/etc/pve/lxc/104.conf` — same cgroup pattern as CT 103
- NVIDIA 595.71.05 userspace installed in CT (`--no-kernel-module`)
- `nvidia-container-toolkit` with `no-cgroups=true` (required for unprivileged LXC)
- Data point: Immich ML batch fills ~4GB VRAM — Frigate's always-on detect must coexist;
  contention during Immich import is expected. Schedule large imports as a window if needed.

## Camera Notes

- C310: zone-based detection
- C500: ONVIF autotrack (person + car). Port 2020 for PTZ control.
- Aim: higher-res 3MP main stream detect via NVDEC
- MQTT / Mosquitto: optional, add later (HA Mosquitto add-on)
