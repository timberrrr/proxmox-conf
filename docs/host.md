# Proxmox Host

## Hardware

- **Model:** Dell Vostro P83F (laptop)
- **CPU:** Intel i7-9750H @ 2.6GHz (6c/12t)
- **GPU:** GTX 1650 (4GB VRAM)
- **iGPU:** Intel UHD 630
- **RAM:** 16GB
- **Storage:** 477GB NVMe
- **NICs:** No built-in ethernet — 2x USB-C/USB-A 2-in-1 2.5G adapters (Umart XC5962 equiv, RTL8156)

## Proxmox

- **IP:** 192.168.1.3 (static, set during install)
- **Web UI:** https://192.168.1.3:8006
- **Version:** PVE 9.2.0, kernel 7.0.2-6-pve (Debian 13 trixie)
- Repos: `pve-no-subscription` (trixie), enterprise repos disabled, nag screen disabled

## Network

- **Router:** Orbi Pro WiFi 6 Mini SXR30 (firmware V4.3.4.200)
- **Main LAN:** 192.168.1.x / vmbr0 → nic1 (ASIX AX88179B)
- **IoT subnet:** 192.168.30.x (VLAN 40) / vmbr1 → nic0 (NO-CARRIER until Orbi port wired)
- Orbi does NOT support 802.1Q trunk on main router — port-based VLAN only
- Proxmox is connected to Orbi **satellite** (SXS30) via wired backhaul; over Ethernet backhaul
  Orbi carries non-default VLANs as 802.1Q-tagged to satellites → assign satellite LAN port to
  IoT VLAN profile in Orbi: Advanced → Advanced Setup → VLAN/Bridge Settings

### IoT VLAN setup (pending user action)
- Wire nic0 → satellite port assigned to IoT VLAN 40
- Verify: `ip addr add 192.168.30.50/24 dev nic0 && ping 192.168.30.1`
- Access-mode port delivers untagged frames — no VLAN config needed on Proxmox side
- CAVEAT: if a managed switch is in the backhaul path, tag VLANs on backhaul ports, disable
  L2 loop protection, never tag VLAN 1 (Orbi breaks)

## USB NIC Driver Notes

Both adapters are ASIX AX88179B (USB ID `0b95:1790`), same batch → identical MAC `9c:69:d3:3f:e9:04`.

**Driver:** `cdc_ncm` (binds to config 2, interfaces :2.0/:2.1). Do NOT use `ax88179_178a` —
it crashes with TX watchdog immediately on this hardware/kernel combo.

**TX watchdog fix:** ethtool offloads disabled on nic1 (tso/gso/gro/ufo off). Persistent via
`post-up` in `/etc/network/interfaces`. Applied after heavy rsync caused original crash.

**Orbi satellite ARP gotcha:** If host has carrier/TX but RX=0 (ARP requests leave, no replies),
restart the Orbi satellite. Stuck forwarding table. This was a 6-hour debug session.

If udev renames NICs differently after reboot, update `bridge-ports` in `/etc/network/interfaces`.

## GPU Plan (locked 2026-06-21)

| GPU | Assigned to | Reason |
|-|-|-|
| GTX 1650 (NVIDIA) | Frigate (always-on) + Immich (bursty) | High-res detect + CUDA ML |
| Intel UHD 630 (iGPU) | Jellyfin QSV | Free since Frigate moved to NVIDIA |
| ~~GTX 1650~~ | ~~Ollama~~ | Dropped — reserved VRAM 24/7, outclassed by Claude API |

Rationale: local Ollama (qwen2.5:3b) was rarely queried AND outclassed by Claude/GPT, yet it
reserved ~2.2GB of the 4GB GPU 24/7, forcing Frigate onto the weaker iGPU. Dropped → freed VRAM.
HA Assist uses native Claude/Anthropic conversation agent instead.

**Note:** Frigate (always-on big model) + a usable LLM can't both be resident on 4GB. If a local
LLM is ever truly needed, real fix = bigger/second GPU (8–12GB).

### Render node enumeration (verified 2026-06-27)

| Node | GPU | PCI |
|-|-|-|
| renderD128 | Intel UHD 630 (i915) | 00:02.0 |
| renderD129 | NVIDIA GTX 1650 | 01:00.0 |

Intel loads first → gets renderD128. NVIDIA loads after (via `/etc/modules-load.d/nvidia.conf`) → renderD129.

If NVIDIA module fails to load after kernel upgrade (missing DKMS rebuild):
```
dkms autoinstall -k $(uname -r) && modprobe nvidia
```

### NVIDIA Driver

- Version: 595.71.05 via `.run` installer + DKMS on host
- nouveau blacklisted (removed live, no reboot needed)
- Boot persistence: `/etc/modules-load.d/nvidia.conf` + udev rule `70-nvidia.rules`
- Device majors for LXC cgroup passthrough: 195 (nvidia/ctl/modeset), 506 (uvm), 509 (caps)
- CTs with GPU: install same version userspace `--no-kernel-module` + `nvidia-container-toolkit`

### exFAT

- `exfat` kernel module available on host; `mkfs.exfat` (exfatprogs) installed
- No `parted` — use `sfdisk`
