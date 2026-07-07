# Exit-Node VPS (vps2) — WireGuard Full-Tunnel Egress

## Architecture

Second, independent VPS from the same hosting platform as the [remote-access](remote-access.md) hub.
Unlike that VPS, **no inbound tunneling** — this one exists purely so client devices can route all
their internet traffic out through it (privacy from local ISP/network, foreign exit IP). There is no
Caddy, no reverse proxy, no exposed web service; the only open ports are SSH and the WireGuard port.

```
Client device --(full tunnel via WireGuard)--> vps2 --masquerade--> internet
```

LAN devices (casting, printers, local services) are explicitly excluded from the tunnel via a
split `AllowedIPs` list, so they keep working normally regardless of tunnel state — same
"full tunnel + LAN bypass" pattern as the peers on the original `vps`.

## VPS

- **Hostname:** `alittledirtneverhurt.supersrv.de` (netcup) — **do not trust DNS for the IP**; this
  hostname briefly resolved to the hosting provider's generic wildcard IP (`46.38.243.234`, same
  wildcard flagged as irrelevant in [remote-access.md](remote-access.md)) before the real IP was
  confirmed by direct SSH. Always verify against the IP below, not the hostname.
- **IP:** `45.132.245.122`
- **OS:** Debian 13 (trixie), KVM (netcup VPS pico G11s)
- **SSH:** `ssh vps2` (`~/.ssh/vps2_ed25519` — dedicated key, isolated from `proxmox_ed25519` and
  the original `vps_ed25519`)

```
Host vps2
    HostName 45.132.245.122
    User root
    IdentityFile ~/.ssh/vps2_ed25519
    StrictHostKeyChecking accept-new
```

## WireGuard

**Subnet:** `10.98.0.0/24` + `fd98:98::/64` | **Port:** UDP 51820 — deliberately a **different**
subnet from the original `vps` tunnel (`10.99.0.0/24`) to keep the two unambiguous if both are ever
active on the same client at once.

| Peer | WG IP | Device |
|-|-|-|
| vps2 | 10.98.0.1 | hub (exit node) |
| toby's tablet | 10.98.0.2 | Android, full tunnel + LAN bypass |
| toby's iphone | 10.98.0.3 | iOS, full tunnel + LAN bypass |
| mel's iphone | 10.98.0.4 | iOS, full tunnel + LAN bypass |

**Server config** `/etc/wireguard/wg0.conf` — interface + `[Peer]` blocks with pubkey/AllowedIPs only,
no client private keys ever stored server-side. Client private keys are generated on the VPS,
delivered to the device, then deleted server-side immediately.

**Client `AllowedIPs`** (full tunnel + LAN bypass): IPv4 is `0.0.0.0/0` minus `192.168.1.0/24`,
expressed as 24 explicit CIDRs (computed via Python `ipaddress.address_exclude`); IPv6 is a literal
`::/0` (safe on Android/iOS — only the Windows client has the kill-switch bug noted in
[remote-access.md](remote-access.md), so literal default routes are fine here).

```
AllowedIPs = 0.0.0.0/1, 128.0.0.0/2, 192.0.0.0/9, 192.128.0.0/11, 192.160.0.0/13, 192.168.0.0/24,
192.168.2.0/23, 192.168.4.0/22, 192.168.8.0/21, 192.168.16.0/20, 192.168.32.0/19, 192.168.64.0/18,
192.168.128.0/17, 192.169.0.0/16, 192.170.0.0/15, 192.172.0.0/14, 192.176.0.0/12, 192.192.0.0/10,
193.0.0.0/8, 194.0.0.0/7, 196.0.0.0/6, 200.0.0.0/5, 208.0.0.0/4, 224.0.0.0/3, ::/0
```

**DNS — required, learned the hard way (2026-07-06):** client configs MUST set
`DNS = 1.1.1.1, 1.0.0.1` in `[Interface]`. Without it, remote clients fall back to whatever resolver
their local network hands out over DHCP, which is not reachable/usable once all traffic is forced
through the tunnel — symptom was "tunnel shows connected, zero data works." All three peers above
were rebuilt with `DNS` set after diagnosing Mel's iPhone this way. (This mirrors, but is the inverse
of, the original `vps`'s deliberate no-DNS-line choice — that one relies on LAN AdGuard being
reachable via the bypass; this one has no LAN to fall back to, so it must force a resolver.)

## Client provisioning

1. On the VPS: `wg genkey | tee <name>.key | wg pubkey > <name>.pub`
2. `wg set wg0 peer <pub> allowed-ips 10.98.0.<n>/32,fd98:98::<n>/128` (live) + append `[Peer]` block
   to `/etc/wireguard/wg0.conf` (persisted) — or `wg-quick save wg0` to snapshot live state.
3. Delete the `.key`/`.pub` files from the VPS immediately after noting the values.
4. Build the client `.conf` (template below), generate a QR with `qrencode`/Python `qrcode`
   **locally**, never upload the config/QR to any hosted service (contains the private key).
5. Deliver via QR scan (WireGuard app's "Create from QR code") or direct file transfer.
6. Delete the local `.conf`/`.png` copies once the device has imported them.

**Client template:**
```ini
[Interface]
PrivateKey = <client private key>
Address = 10.98.0.<n>/32, fd98:98::<n>/128
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = Mec8LOz6JQlpOKMMmfp+rK2CVcp4AmZN9VZ0IwHS3y8=
Endpoint = 45.132.245.122:51820
AllowedIPs = <24-CIDR v4 split above>, ::/0
PersistentKeepalive = 25
```

**Server WireGuard public key:** `Mec8LOz6JQlpOKMMmfp+rK2CVcp4AmZN9VZ0IwHS3y8=`

## Hardening

Same posture as the original `vps`, minus anything web-facing (nothing is reverse-proxied here):

- **nftables** (`table inet homelab_fw`) — default-**drop** input; allow `lo`, established/related,
  ICMP/ICMPv6, `22/tcp`, `51820/udp`. Forward chain scoped to `wg0` only. No geoblock needed (no
  exposed 443/80).
- **NAT** — separate `ip nat`/`ip6 nat` tables, masquerade `10.98.0.0/24` and `fd98:98::/64` out `eth0`.
  Done directly in nftables from the start (the original `vps` had to migrate a masquerade rule from
  iptables to nftables after a hardening regression — see its "Outbound routing fix" section — this
  server skipped that mistake).
- **IP forwarding** enabled (`net.ipv4.ip_forward`, `net.ipv6.conf.all.forwarding`) via
  `/etc/sysctl.d/99-wg-forward.conf`.
- **BBR** congestion control (`/etc/sysctl.d/99-bbr.conf` + `/etc/modules-load.d/bbr.conf`) — same
  rationale as the original VPS (long-fat path from home to a European VPS benefits from BBR over
  cubic on the sender side).
- **fail2ban** — `sshd` jail only (`nftables-multiport` banaction). No web jails (nothing web-facing
  to protect).
- **unattended-upgrades** installed.

## Diagnosing a peer

```
ssh vps2 "wg show wg0"
```
- **Endpoint + recent "latest handshake"** → peer is connected and passing traffic.
- **No endpoint at all** → the client has never successfully handshaked (not just "handshake expired").
  Check, in order: (1) client imported the *correct* config (compare the Interface public key shown in
  the client app against the expected pubkey for that peer — catches "scanned the wrong QR" cases,
  which has happened), (2) client-side network isn't blocking outbound UDP/51820 (try switching
  Wi-Fi↔cellular), (3) VPN is actually toggled on at the OS level, not just the app.
- **Connected with a handshake but "no data works"** → check DNS. A full-tunnel client with no
  reachable resolver will show as connected while being completely dead — see the DNS section above.
- `tcpdump -i eth0 -n udp port 51820` on the VPS can confirm whether packets are arriving at all;
  useful for telling apart "client never sent anything" from "server received but rejected it"
  (unrecognized keys silently drop with no server-side log).

## Pending

- [ ] Confirm all three current peers (tablet, toby's iphone, mel's iphone) hold a stable handshake
      after the DNS fix rollout (2026-07-06).
- [ ] Consider whether a fallback second DNS provider is worth adding if Cloudflare ever has an outage
      affecting a client mid-trip.
