# VM 100 — Home Assistant OS (HAOS)

- **IP:** 192.168.1.2
- **UI:** http://192.168.1.2:8123
- **Version:** HAOS 18.0
- **Status:** running, onboot=1

## Setup

Created via community script:
```
bash -c "$(wget -qO - https://github.com/community-scripts/ProxmoxVE/raw/main/vm/haos-vm.sh)"
```
Default settings used. RPi backup restored — came up at 192.168.1.2 (RPi's old static IP).
RPi shut down to avoid IP conflict during migration.

## Original RPi HA Setup (migrated)

| Component | Notes |
|-|-|
| File editor | Add-on |
| AdGuard Home | Add-on |
| NUT | Add-on (UPS USB needs moving from RPi to Vostro) |
| Cloudflared | Add-on (replaced by Caddy+WG tunnel) |
| Claude Code | Custom add-on — needs reinstall |
| Mosquitto | NOT installed — needed for Frigate MQTT (add later) |

- **Cameras:** 1x C310 (3MP fixed), 2x C500 (3MP PTZ) on IoT subnet 192.168.30.x DHCP
  - Currently use Tapo local API in HA
  - RTSP must be enabled in Tapo app for Frigate
  - C500 supports ONVIF PTZ autotracking (port 2020)
- **Other integrations:** Tesla, Ratgdo garage (ESPHome), person detection automations
- **~427 entities total**

## Known Issues

### Core update failures — TCP congestion control (2026-07-05)

**Symptom:** HA Supervisor fails to download Core images (~381MB layers) with:
```
ERROR [supervisor.docker.interface] Can't install ghcr.io/home-assistant/qemux86-64-homeassistant:2026.7.0:
failed to copy: local error: tls: bad record MAC
```

**Root cause:** HAOS kernel only has `cubic` congestion control. Cubic collapses on long-duration
transfers (~1.5 Mbps on a 280ms RTT path). BBR is not available in the HAOS kernel — it's not
compiled in (unlike Proxmox/LXC containers, which got BBR on 2026-07-04).

**Why HAOS was missed:** The 2026-07-04 BBR fix was applied to the Proxmox host kernel, which
propagates to LXCs automatically. HAOS (VM 100) has its own independent kernel and didn't inherit
it. Applying BBR requires the `tcp_bbr` module, which HAOS kernel doesn't provide:

```
$ sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic
```

HAOS uses a heavily customized, minimal kernel — it doesn't include all modules that standard Linux
distributions have.

**Workaround (blocked):** Cannot apply TCP buffer workaround because HAOS has a **read-only filesystem**.
HAOS is immutable/overlay-based — `/etc/`, `/lib/`, etc. are read-only. Even if sysctl changes are
applied live, they don't survive a reboot, and writing to `/etc/sysctl.d/` is blocked.

**Status:** *Effectively unresolved — confirmed upstream (2026-07-05).* Checked the HAOS source
(`home-assistant/operating-system`, `main` branch, kernel 6.18.y): the generic-x86-64 build (what
VM 100 runs as `qemux86-64`) uses the upstream `x86_64_defconfig` plus HAOS fragments, and **none of
them enable `CONFIG_TCP_CONG_BBR`** — no `TCP_CONG` options at all, so only the defconfig's
reno/cubic. BBR *is* enabled in HAOS kernels, but only for the arm64-rockchip boards. No open
issue/PR requests it for x86-64. Conclusion: **upgrading HAOS (18.1 or the upcoming 6.18-kernel
series) will NOT bring BBR.**

Also ruled out (2026-07-05): applying BBR on the Proxmox host does nothing for the VM — congestion
control runs in the TCP endpoint's kernel, and VM 100 has its own. (Host has had BBR since
2026-07-04; HAOS failures continued after.) That only works for LXCs, which share the host kernel.

Options:
1. **Upstream PR filed (2026-07-05):** https://github.com/home-assistant/operating-system/pull/4865
   adds `CONFIG_TCP_CONG_BBR=m` + `CONFIG_NET_SCH_FQ=m` to the shared kernel fragment
   (`buildroot-external/kernel/v6.18.y/haos.config`) — all boards, CUBIC stays default. Watch for
   review feedback; if merged, a future HAOS release ships the module.
2. **Accept the limitation:** Docker image downloads are slow but eventually succeed (with retries).
3. **Pre-cache images:** Download images externally and load them into HAOS Docker (complex; only if
   updates keep timing out).

## Pending

- [ ] **Core update stability:** Retry Core 2026.7.0 update; it will eventually succeed even if slow
      (~760KB/s observed). Upgrading HAOS won't help — confirmed no BBR in any x86-64 HAOS kernel,
      including main branch (see Known Issues). Optionally file an upstream feature request for
      `CONFIG_TCP_CONG_BBR=m` in the x86-64 kernel config.
- [ ] Move UPS USB cable from RPi → Vostro, verify NUT add-on picks it up
- [ ] Install Mosquitto add-on (required for Frigate MQTT — can defer)
- [ ] Switch HA Assist to native Claude/Anthropic conversation agent (Ollama dropped)
- [x] **HA native IP ban — DONE 2026-07-05.** Applied in `configuration.yaml` + restart:
      ```yaml
      homeassistant:
        external_url: "https://ha-152-89-105-204.sslip.io"   # was stale home.peachester.com
        internal_url: "http://192.168.1.2:8123"
      http:
        use_x_forwarded_for: true
        trusted_proxies:
          - 172.30.33.0/24   # add-on/ingress network
          - 127.0.0.1
          - 192.168.1.16     # CT107 — covers VPS-masqueraded AND local-Caddy traffic
        ip_ban_enabled: true
        login_attempts_threshold: 3
      ```
      Bans land in `/config/ip_bans.yaml` (delete entry + restart to unban — self-lockout possible
      at threshold 3). Bans apply to the forwarded real client IP, not 192.168.1.16.
- [ ] Immich admin password reset (`shouldChangePassword` still set)
- [ ] Calibre: decide on CW vs CWA for phone app support

## Remote Access

HA is exposed via Caddy on VPS: `ha-152-89-105-204.sslip.io → 192.168.1.2:8123`
See [remote-access.md](remote-access.md) for WireGuard + Caddy details.
