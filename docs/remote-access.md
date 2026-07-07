# Remote Access — WireGuard + Caddy

## Architecture

Home is behind CGNAT (no inbound). A rented VPS with a public IP is the always-on hub;
home dials OUT (CGNAT-friendly) and holds the tunnel open.

```
Internet → VPS Caddy → WireGuard tunnel → CT 107 → LAN services
```

Two flows:
- **INBOUND (done):** internet → VPS Caddy → tunnel → Jellyfin + HA only
- **OUTBOUND (deferred):** route a dedicated exit-VLAN's internet traffic out via VPS
  (blocked on IoT VLAN + router decision; revisit with OPNsense/OpenWrt)

## VPS

- **IP:** 152.89.105.204
- **OS:** Debian 13, hostname `bingo.hotsrv.de`
- **SSH:** `ssh vps` (`~/.ssh/vps_ed25519` — dedicated key, isolated from proxmox key)
- **Caddy:** v2.11, apt cloudsmith repo, `/etc/caddy/Caddyfile`, auto Let's Encrypt

```
Host vps
    HostName 152.89.105.204
    User root
    IdentityFile ~/.ssh/vps_ed25519
    StrictHostKeyChecking accept-new
```

**Note:** `bingo.hotsrv.de` is a provider wildcard → 46.38.243.234 (unrelated to this VPS).
`sslip.io` chosen for privacy: public name embeds the already-public IP; no domain owned by user;
nothing in Certificate Transparency logs tied to user.

## CT 107 — wg-gateway

- **IP:** 192.168.1.16
- **Status:** running, onboot=1
- **Specs:** unprivileged, 1c/512M/4G, Debian 13

`tun` passthrough in `107.conf`:
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
Same `proxmox_ed25519` key authorized. Acts as WireGuard spoke + LAN subnet router (masquerade).

## WireGuard

**Subnet:** `10.99.0.0/24` | **Port:** UDP 51820

| Peer | WG IP | Mode |
|-|-|-|
| VPS | 10.99.0.1 | hub |
| CT 107 (home) | 10.99.0.2 | LAN gateway (masquerade) |
| Windows laptop | 10.99.0.4 | full tunnel + LAN bypass |
| Tim phone | 10.99.0.5 | full tunnel + LAN bypass |
| Sara phone | 10.99.0.6 | full tunnel + LAN bypass |

**All clients use full tunnel + LAN bypass** (2026-07-05; laptop converted from a short-lived
services-only split config same day — goal is privacy from the local ISP everywhere; drop the
tunnel manually for latency-sensitive work like calls or big downloads).

**Full tunnel + LAN bypass** = `AllowedIPs = <0.0.0.0/0 minus 192.168.1.0/24
as 24 explicit CIDRs>, ::/0` — ALL internet traffic exits via the VPS (whatsmyip shows the German
IP), but 192.168.1.0/24 is never tunneled, so at home everything local stays direct: AdGuard DHCP
DNS → split-horizon rewrite → local Caddy (192.168.1.16), plus Frigate/Calibre/printers by IP.
No DNS override in either profile — that's what keeps split-horizon DNS working at home. Away,
DNS is the carrier/hotspot resolver (public sslip.io resolution → VPS Caddy); accepted tradeoff.
IPv6: phones get `fd99:99::5/6` + `::/0` for leak-free v6 (same as the old full-tunnel phone).

wireguard kernel module loaded + persisted on Proxmox host AND on VPS.
`wg-quick@wg0` enabled both ends.

**VPS `wg0` peer (home) AllowedIPs:** (live value 2026-07-05, `wg show wg0 allowed-ips`)
```
10.99.0.2/32, 192.168.1.2/32, 192.168.1.14/32, 192.168.1.11/32
```
HA, Jellyfin, and fbmp (CT102) reachable from VPS — not the full LAN. (`192.168.1.11/32` also has a
kernel route `192.168.1.11 dev wg0`; both confirmed present, so the fbmp re-exposure needed no WG change.)

**CT 107 `wg0`:**
```
Endpoint=152.89.105.204:51820
AllowedIPs=10.99.0.0/24
PersistentKeepalive=25
PostUp: iptables -t nat -A POSTROUTING -s 10.99.0.0/24 -o eth0 -j MASQUERADE
PostUp: iptables -A FORWARD -i wg0 -j ACCEPT
```

**Keys:** VPS `/etc/wireguard/` holds ONLY `wg0.conf` (own private key inline, peer pubkeys in
`[Peer]` blocks) — all loose key files, client confs, QRs, and the stale `.bak` were purged
2026-07-05. Client private keys exist only on their devices. CT 107 keeps `home.key`/`home.pub`.
To add a client: `wg genkey`/`wg pubkey`, `wg set wg0 peer <pub> allowed-ips <ip>/32`, append the
`[Peer]` to `wg0.conf`, build the client conf, deliver, then delete the server-side copies.

Handshake verified: from VPS `curl http://192.168.1.14:8096` and `http://192.168.1.2:8123` = 200.

## Caddy (VPS)

`/etc/caddy/Caddyfile` (JSON access logging added 2026-07-05 via `(logged)` snippet →
`/var/log/caddy/access.log`, owned `caddy:caddy`, 10 MiB roll × 5):
```
(logged) {
    log { output file /var/log/caddy/access.log { roll_size 10MiB roll_keep 5 }
          format json }
}
jellyfin-152-89-105-204.sslip.io       { import logged; reverse_proxy 192.168.1.14:8096 }
ha-152-89-105-204.sslip.io             { import logged; reverse_proxy 192.168.1.2:8123  }
fbmp-152-89-105-204.sslip.io           { import logged; reverse_proxy 192.168.1.11:8787 }
fbmp-friend-152-89-105-204.sslip.io    { import logged; reverse_proxy 192.168.1.11:8788 }
```
**fbmp/fbmp-friend RE-EXPOSED publicly 2026-07-05** (reverses the earlier tunnel-only pull).
The reason they were pulled — a secrets-writing admin UI behind only a static token, listening
24/7 — no longer holds: the frontend is now **ephemeral**. Its listener binds **only during an
explicitly-opened window** (a Telegram `web on`, default 60 min, auto-closes) and is fully torn
down otherwise, so an off-window scan gets a **502** (verified). Layered defense when open:
AU geoblock (443) → TLS → a **per-window session token** minted on `web on` and delivered over
Telegram as a `#t=` deep link (rotates every open) → `caddy-fbmp` fail2ban jail → auto-close.
The static `WEB_AUTH_TOKEN` remains only the enable-gate + first-run bootstrap credential.
Backends live on CT102; VPS→`192.168.1.11` already routed (WG AllowedIPs + kernel route present).

| Service | URL | Status |
|-|-|-|
| Jellyfin | jellyfin-152-89-105-204.sslip.io | LIVE, HTTP 200, valid cert |
| Home Assistant | ha-152-89-105-204.sslip.io | cert OK; `trusted_proxies` set (2026-06-27) |
| fbmp (owner) | fbmp-152-89-105-204.sslip.io | LIVE; 502 when window closed, 200 after `web on` (ephemeral) |
| fbmp-friend | fbmp-friend-152-89-105-204.sslip.io | LIVE; auto-opens a bootstrap window (no Telegram yet) |

### Why hostname-per-service (not one hostname + ports/paths)

Each service gets its own `*-<ip>.sslip.io` name, all on **443**, routed by SNI/`Host` header;
the LAN-side port is hidden inside `reverse_proxy`. This is deliberate. The alternatives are worse:

- **One hostname + a port per service** (`home-<ip>.sslip.io:8096`): non-standard outbound ports are
  routinely blocked by hotel/mobile/corporate/captive-portal networks — exactly where you roam. **443
  is open everywhere.** This alone decides it for internet-exposed services.
- **One hostname + a path per service** (`/jellyfin`, `/ha`): most of these apps (Jellyfin, HA, Immich,
  Frigate, Calibre) assume they live at web **root** and emit absolute URLs / websockets / redirects.
  Subpath hosting means per-app base-URL config and endless "UI works but the websocket 404s" debugging
  (HA is especially painful here). Hostname routing lets every backend think it's at `/`.

Other wins: Caddy auto-provisions a **separate Let's Encrypt cert per hostname**; auth **cookies are
scoped per-hostname** so services can't collide; adding a service is just another 3-line Caddy block
(+ its `/32` in the VPS WG `AllowedIPs`).

**One downside:** `sslip.io` bakes the IP into the name, so if the VPS IP changes, every hostname
changes. That's a property of the sslip.io privacy choice, not of hostname-vs-port. A cheap owned domain
with a wildcard record + wildcard cert would give stable short names — but reintroduces the Certificate
Transparency footprint deliberately avoided (see VPS note above). Only revisit if the IP starts changing.

## Phone Roaming

Original full-tunnel peer (`10.99.0.3`, added 2026-06-25, `DNS = 1.1.1.1`, `AllowedIPs =
0.0.0.0/0, ::/0`) **RETIRED 2026-07-05** — removed from VPS `wg0` (live + conf) and from the
phone. Superseded by the "full tunnel + LAN bypass" profiles (`10.99.0.5`/`.6`, see peer table),
which keep the same route-everything-out-via-VPS behavior while bypassing `192.168.1.0/24` so
split-horizon DNS and local services work directly at home. Note the retired profile hardcoded
`DNS = 1.1.1.1`; the new profiles deliberately set no DNS (AdGuard at home via DHCP).
VPS WG ULA `fd99:99::/64` remains for leak-free IPv6 on the new profiles.

**Windows laptop peer added 2026-07-05:** `10.99.0.4/32, fd99:99::4/128` — same **full tunnel +
LAN bypass** profile as the phones (initially services-only split; converted same day). No DNS
override. Split-horizon intact: away, public DNS → VPS Caddy path; at home with tunnel up,
AdGuard rewrite → `192.168.1.16` stays on-link (not tunnel-routed) → local Caddy. Keys: —

**⚠ WireGuard-for-Windows kill-switch gotcha (hit + fixed 2026-07-05):** if `AllowedIPs` contains
a literal default route (`0.0.0.0/0` **or `::/0`**), the Windows client silently enables a
lockdown firewall that blocks ALL non-tunnel traffic — including LAN traffic and DNS to AdGuard
(and with no `DNS =` line, all DNS dies → "tunnel up, no traffic"). This defeats the LAN-bypass
design. Fix: never use literal default routes on Windows — split them (`::/1, 8000::/1`, and the
v4 list is already split). The laptop conf uses this. Android/iOS clients don't have this
behavior, so phone configs keep `::/0`.
key files cleaned off the VPS after import 2026-07-05 (private key lives only in the Windows
WireGuard app; peer pubkey is persisted in `wg0.conf`).

Additional clients: same pattern, next IPs `10.99.0.5` / `fd99:99::5`.

**Note:** VPS needed `iptables` package installed (Debian 13 minimal had none) or `wg-quick PostUp` fails.

## Throughput — BBR congestion control (2026-07-04)

**The VPS is in Germany; home is in Australia → ~280 ms RTT.** This is a long-fat path and the
default **cubic** congestion control **collapses on the upload leg** (home→VPS), which is the
direction all remote media flows (Jellyfin → tunnel → VPS → client, an AU→DE→AU hairpin).

Measured home→VPS upload: **~1.5–3 Mbps on cubic** (download from VPS was fine, ~11 Mbps —
asymmetric). That's enough for music but **starved remote video** — it was the real reason
remote Jellyfin video wouldn't start (poster shows, player never buffers). Not WireGuard, disk,
or the VPS (VPS has ~215 Mbps to the German internet; tunnel had 0% loss).

**Fix: BBR.** Switching the sender's congestion control to BBR took the same path from ~1.5 Mbps
to **~19–22 Mbps** — a >7× win, well above the 8 Mbps remote video cap.

Applied (persistent):
- **Proxmox host + LXC containers** `/etc/sysctl.d/99-bbr.conf` (`net.core.default_qdisc=fq`,
  `net.ipv4.tcp_congestion_control=bbr`) + `/etc/modules-load.d/bbr.conf` (`tcp_bbr`). New
  container netns inherit BBR. CT105 (Jellyfin) is the media sender — set live via
  `nsenter -t <pid> -n sysctl -w net.ipv4.tcp_congestion_control=bbr` and inherits BBR on restart.
- **VM 100 (HAOS) — NOT fixed.** HAOS has a read-only immutable filesystem; BBR module isn't in the
  kernel anyway. HA Core image downloads remain slow (~760KB/s on cubic) but eventually succeed
  with retries. See `vm-100-haos.md` for details.
- **VPS** same `sysctl.d` + `modules-load.d` (helps the VPS→client second leg).

**Verified:** real `stream.mp4` pull VPS→Jellyfin over the tunnel = ~18.7 Mbps (was ~1.5).
Note unprivileged LXCs can't set `net.*` sysctls from inside — must be done on the host (default
inheritance) or via `nsenter` into the container netns.

## Hardening (VPS edge) — done 2026-07-05

All edge hardening lives on the **VPS**, not per-service and not on CT107: the VPS is the sole
ingress chokepoint and the only place the real client IP exists (past CT107's `MASQUERADE` the
source collapses to the tunnel IP, so a downstream fail2ban would ban the proxy, not the attacker).
Dropping at the VPS also stops abuse before it enters the AU↔DE tunnel.

- **nftables host firewall** — was wide open (INPUT ACCEPT, empty). Now default-**drop** input,
  allow-list only: established/related, `lo`, ICMP/ICMPv6 (kept for PMTUD on the fat path),
  `22/tcp` + `80/tcp` global, `51820/udp` (WG hub — home + phone dial IN; **omitting this kills the
  whole tunnel**), and `443/tcp` **geo-gated** (see Geoblock below). Table `inet homelab_fw`;
  persisted via `include` in `/etc/nftables.conf`,
  `nftables.service` enabled. Applied behind a `systemd-run --on-active=180` auto-revert (same safe
  pattern as the Proxmox net change) — verified SSH + WG handshake + proxied 200 before committing.
- **fail2ban** (Debian 13 → **nftables** actions, `banaction = nftables-multiport`; own `f2b-table`,
  coexists with `homelab_fw`). Jails: `caddy-jellyfin` (watches Caddy JSON log; filter scoped to
  **status 401 on `/Users/AuthenticateByName` only** — Jellyfin returns 401 on *any* unauth API
  call, so an unscoped jail would self-ban expired-token clients; 5 fails/10m → 1h ban), `caddy-fbmp`
  (added 2026-07-05 with the fbmp public re-exposure; filter scoped to **status 401 on the
  `fbmp(-friend)-…sslip.io` hosts** so it can't trip on Jellyfin/HA 401s — the fbmp UI only 401s on a
  bad bearer token; **10** fails/10m → 1h ban, deliberately tolerant since the remote friend can
  fat-finger the bootstrap token and `ignoreip` won't cover them; verified matching a live 401 via
  `fail2ban-regex`), and `sshd`.
  `ignoreip` = loopback + VPS IP + `10.99.0.0/24` + `192.168.1.0/24`. Ban path proven end-to-end
  (test IP appeared in the nft set, then unbanned).
- **unattended-upgrades** installed + daily (security patches were not auto-applied).
- **Geoblock on 443 (AU-only, Starlink-safe) — added 2026-07-05.** New TCP connections to the reverse
  proxy (**443 only**) must come from an **AU** CIDR *or* **Starlink AS14593** (both v4+v6); everything
  else hits `tcp dport 443 counter drop`. nft interval sets `allow_v4`/`allow_v6` in `homelab_fw`.
  - **Starlink safety:** Starlink AU IPs (esp. IPv6) often geolocate outside AU / are registered to
    SpaceX (US), so AU-geo alone would drop them. We allow **all of AS14593** regardless of geo — a
    deliberate tradeoff (a Starlink user *anywhere* is allowed; they still face fail2ban + app auth).
    ⚠ Completeness is on faith — **not yet tested against a real Starlink client's live v4+v6.**
  - **Why 443 only, not 80:** Let's Encrypt validates ACME from US/EU vantage points and sslip.io
    can't do DNS-01, so the challenge must arrive on 80/443 from non-AU. **Port 80 stays global**
    (serves only the ACME challenge + HTTPS redirect). Geoblocking it would break cert renewal
    silently ~60 days out. **SSH (22) and WireGuard (51820) also stay global** — WG must work while
    roaming abroad.
  - **Data:** AU from ipdeny aggregated zones; Starlink from RIPEstat `announced-prefixes/AS14593`.
    Refresh: `/usr/local/sbin/update-geoblock.sh` (fails closed to the old set on any download/sanity
    error — min 4000 v4 / 800 v6 or it aborts without wiping). Weekly `geoblock-update.timer`
    (Sun 04:00). Persisted set elements in `/etc/nftables/geoblock-sets.nft` (loaded at boot after
    `homelab_fw.nft`, so no empty-set window). Current: ~7.5k v4 + ~2.6k v6 CIDRs.
  - Applied behind the same `systemd-run` auto-revert; verified AU-in / US-out / Starlink-in (v4+v6)
    and survives a full `nftables` reload.

**HA brute-force is NOT covered by the VPS fail2ban — closed with HA's native ban (2026-07-05).**
HA returns **HTTP 200** with a JSON error body on a *failed* login (only Jellyfin returns 401), so
there is no edge-side signal to match. Applied on VM 100: `ip_ban_enabled: true` +
`login_attempts_threshold: 3` with `trusted_proxies` incl. `192.168.1.16`, so bans record the real
client IP via XFF. Details in `vm-100-haos.md`.

## Outbound routing fix (2026-07-05 hardening regression)

When nftables hardening was applied 2026-07-05, the POSTROUTING masquerade rule was not
migrated from iptables to nftables. Outbound tunnel traffic from LAN clients was not NATted,
so return packets never arrived.

**Fix:** Added `/etc/nat.nft` with POSTROUTING masquerade for `10.99.0.0/24` (v4) and
`fd99:99::/64` (v6) exiting on `eth0`. Included in `/etc/nftables.conf`, rules live-loaded.
Removed stale iptables PostUp/PostDown from WireGuard config (nftables now owns masquerade).

## Seamless local access — split-horizon DNS + local Caddy (2026-07-05)

**Problem:** Remote access works via `jellyfin-152-89-105-204.sslip.io` and `ha-152-89-105-204.sslip.io`
routed through VPS Caddy. But from the LAN, you still have to remember port numbers (8096, 8123) or use
direct IP addresses.

**Solution:** Split-horizon DNS + local reverse proxy.

- **CT 107 (wg-gateway)** now runs **Caddy on HTTPS:443** (+ auto HTTP→HTTPS redirect on :80), with
  the same reverse_proxy config as the VPS and the **same Let's Encrypt certs** (synced daily — see
  "Local HTTPS" below).
- **AdGuard DNS rewrites** (split-horizon) return different IPs depending on source network:
  - **From LAN (192.168.1.0/24):** return `192.168.1.16` (CT 107 IP) → local Caddy on 443
  - **From remote/VPN:** fall through to public sslip.io resolution → VPS Caddy on port 443

**How it works across all scenarios:**
- **On LAN without VPN:** `jellyfin-152-89-105-204.sslip.io` → DNS rewrites to 192.168.1.16 → local
  Caddy → `192.168.1.14:8096` (fast, valid HTTPS, no port in browser)
- **On LAN with VPN tunnel up:** same as above (CT 107 is reachable first, local path always wins)
- **Away from home (via VPN):** `jellyfin-152-89-105-204.sslip.io` → sslip.io resolves to 152.89.105.204
  → VPS Caddy on 443 → tunnel → local services (full HTTPS)

**Setup (done 2026-07-05):**
- Installed Caddy on CT 107: `apt install caddy`
- `/etc/caddy/Caddyfile` (backup of the original HTTP-only version at `Caddyfile.bak`):
  ```
  jellyfin-152-89-105-204.sslip.io {
    tls /etc/caddy/certs/jellyfin-152-89-105-204.sslip.io.crt /etc/caddy/certs/jellyfin-152-89-105-204.sslip.io.key
    reverse_proxy 192.168.1.14:8096
  }

  ha-152-89-105-204.sslip.io {
    tls /etc/caddy/certs/ha-152-89-105-204.sslip.io.crt /etc/caddy/certs/ha-152-89-105-204.sslip.io.key
    reverse_proxy 192.168.1.2:8123
  }
  ```
- Service enabled and running: `systemctl status caddy`

**Local HTTPS — cert sync from VPS (done 2026-07-05):**
Browsers that have seen the site externally insist on HTTPS, so HTTP-only local Caddy caused
"can't connect" on the LAN. Local Caddy can't get its own Let's Encrypt certs (HTTP-01 challenges
resolve to the VPS public IP; no DNS-01 for sslip.io), so CT 107 **pulls the VPS's certs** instead:
- `/usr/local/bin/sync-caddy-certs.sh` on CT 107: scp's `{jellyfin,ha}-….sslip.io.{crt,key}` from
  the VPS Caddy store (`/var/lib/caddy/.local/share/caddy/certificates/acme-v02…/`) over the
  tunnel (`root@10.99.0.1`), installs to `/etc/caddy/certs/` (root:caddy 640), reloads Caddy only
  on change.
- Runs daily via `/etc/cron.d/certsync` (04:20). LE certs renew ~every 60 days, so daily sync
  keeps the local copy fresh with huge margin.
- Auth: dedicated key `/root/.ssh/vps_certsync` on CT 107, pubkey (`certsync@ct107`) in VPS
  `authorized_keys`. Direction is deliberate — CT 107 (trusted side) holds the key into the
  exposed VPS, not the reverse.

**AdGuard configuration (split-horizon DNS rewrite):**
In AdGuard Home (or your DNS provider), add **conditional rewrites** for your LAN subnet:
- **Rewrite rule 1 (Jellyfin):**
  - Domain: `jellyfin-152-89-105-204.sslip.io`
  - Answer: `192.168.1.16` (CT 107's IP)
  - **Apply only for:** `192.168.1.0/24` (your LAN subnet)
- **Rewrite rule 2 (Home Assistant):**
  - Domain: `ha-152-89-105-204.sslip.io`
  - Answer: `192.168.1.16`
  - **Apply only for:** `192.168.1.0/24`

**Important:** These rewrites must be **conditional** (scoped to your LAN subnet). Remote/VPN clients
(which don't match 192.168.1.0/24) will fall through to the public sslip.io nameservers and get the
VPS IP (152.89.105.204), which is correct.

**Result:** Same hostname, valid HTTPS everywhere — LAN clients terminate TLS at CT 107 with the
synced cert; remote/VPN clients terminate at the VPS. Verified from LAN workstation 2026-07-05:
both `https://` hostnames return 200 with a trusted chain.

**Testing:**
After AdGuard rewrites are configured, test from a LAN client:
```bash
# From your workstation on the LAN:
nslookup jellyfin-152-89-105-204.sslip.io   # Should resolve to 192.168.1.16
curl http://jellyfin-152-89-105-204.sslip.io   # Should proxy to Jellyfin
curl http://ha-152-89-105-204.sslip.io   # Should proxy to Home Assistant
```

Or from any web browser on the LAN:
- `http://jellyfin-152-89-105-204.sslip.io` — Jellyfin UI
- `http://ha-152-89-105-204.sslip.io` — Home Assistant UI

Once AdGuard rewrites are live, no `/etc/hosts` or manual port typing needed — just use the hostnames.

## Pending

- [x] **HA native IP ban — DONE 2026-07-05** (applied directly, no handoff needed). Kept HA public,
      protected with HA's own ban (VPS fail2ban can't — HA returns 200 not 401). Applied with
      `login_attempts_threshold: 3` + stale `external_url` fixed (`home.peachester.com` →
      `ha-152-89-105-204.sslip.io`). Full config in `vm-100-haos.md`. Original plan:
      ```yaml
      http:
        use_x_forwarded_for: true
        trusted_proxies:
          - 192.168.1.16   # CT107 — HA sees this as the source (CT107 masquerades tunnel→LAN)
        ip_ban_enabled: true
        login_attempts_threshold: 5
      ```
      **Caveat:** `trusted_proxies` MUST be `192.168.1.16` (CT107's masquerade addr) or bans record
      the proxy, not the real client. Verify against the value already set 2026-06-27. Bans →
      `/config/ip_bans.yaml`.
- [x] **fbmp exposure RE-DECIDED 2026-07-05 (later same day)** — now **public via ephemeral window**,
      superseding the earlier tunnel-only pull. Safe to re-expose because the UI no longer listens
      24/7: it binds only during a Telegram-opened, auto-closing window, with a per-window session
      token (Telegram `#t=` deep link) + `caddy-fbmp` fail2ban + AU geoblock. Both hostnames LIVE;
      owner stack 502s until `web on`, friend stack auto-opens for first-run. (See Caddy + fail2ban
      sections.) Driver: the friend needs a public hostname to bootstrap and to use the frontend.
- [x] **HA exposure decided 2026-07-05** — stays public + native ban (handoff item above).
- [ ] Enable 2FA / strong credentials on Jellyfin and HA (both internet-exposed)
- [ ] **Optional** Caddy rate-limit — needs `caddy add-package github.com/mholt/caddy-ratelimit`
      (⚠ a future `apt upgrade caddy` reverts the plugin build → `apt-mark hold caddy`); scope to
      login paths only, NOT global (Jellyfin streaming = high request volume). Deferred; fail2ban is
      the primary brute-force defense.
- [ ] Outbound exit-VLAN routing (deferred, blocked on IoT network + router decision)
