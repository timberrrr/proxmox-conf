# HAOS (VM 100) — Missing BBR, likely cause of Core update pull failures

## Symptom

HA Supervisor repeatedly failed to download the Core 2026.7.0 update image from
`ghcr.io/home-assistant/qemux86-64-homeassistant`:

```
ERROR (MainThread) [supervisor.docker.interface] Can't install ghcr.io/home-assistant/qemux86-64-homeassistant:2026.7.0:
[0] failed to copy: local error: tls: bad record MAC
WARNING (MainThread) [supervisor.homeassistant.core] Downloading Home Assistant version 2026.7.0 failed
```

Failed identically on 3 consecutive attempts (2026-07-04 19:22, 19:30, 19:36; retried again
2026-07-05 13:43 — same error).

## Diagnosis (done from inside the HA Supervisor add-on container)

Manually pulled the same image layers with `curl` (bypassing Docker) to isolate the fault:

- Small layers (11KB–13MB): downloaded fine, correct checksum, fast.
- The large layer (`sha256:a48da49a...`, 381MB): first attempt returned **HTTP 200 but silently
  truncated at 49MB** (checksum mismatch, no error surfaced by curl). A later retry completed
  successfully but took **500 seconds at ~760KB/s** — very slow and marginal.

This is the same signature already root-caused in `remote-access.md` / `ct-105-jellyfin.md`:
**TCP cubic congestion control collapsing on longer-duration / lossy transfers**, while short
requests are unaffected. That was fixed for CT105 (Jellyfin) and the Proxmox host itself by
switching to **BBR** on 2026-07-04 (see `remote-access.md` § Throughput — BBR congestion control).

Checked directly on this HAOS instance:

```
$ sysctl net.ipv4.tcp_congestion_control net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_congestion_control = cubic
net.ipv4.tcp_available_congestion_control = reno cubic
```

`bbr` is not even in the available list — the module isn't loaded. This confirms **HAOS (VM 100)
never got the BBR fix**.

## Why HAOS was missed

The 2026-07-04 BBR fix was applied via `/etc/sysctl.d/99-bbr.conf` +
`/etc/modules-load.d/bbr.conf` **on the Proxmox host**. That propagates automatically to
**LXC containers** (CT105, CT107, etc.) because they share the host kernel/netns defaults.

**HAOS (VM 100) is a full VM with its own independent kernel** — it does not inherit host
sysctl/module state at all. It needs the same fix applied independently, inside the VM.

This is the pending handoff item already noted in `remote-access.md`:
> "HANDOFF to the HA-device Claude instance (VM 100)" — but the Claude Code add-on runs in an
> unprivileged Supervisor container with no `/lib/modules` and no host netns access, so it
> **cannot apply this fix itself**. It has to be done from a true host shell inside the HAOS VM
> (e.g. via the Proxmox console for VM 100, or an SSH/terminal add-on running in *host* mode,
> not the sandboxed add-on container).

## Fix to apply (inside the HAOS VM's host shell — not the add-on container)

```bash
modprobe tcp_bbr
echo tcp_bbr > /etc/modules-load.d/bbr.conf
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
printf 'net.core.default_qdisc=fq\nnet.ipv4.tcp_congestion_control=bbr\n' > /etc/sysctl.d/99-bbr.conf
```

**Caveat:** HAOS is an immutable/overlay-based OS (unlike the Debian LXCs), so file persistence
rules for `/etc` may differ from a normal Debian host. Verify `/etc/sysctl.d/99-bbr.conf` and
`/etc/modules-load.d/bbr.conf` actually survive a HAOS reboot/update before considering this
done — if HAOS resets `/etc` on boot, the sysctl/module load will need to be applied via a HAOS
supported persistence mechanism instead (e.g. supervisor-managed config, or a systemd unit dropped
somewhere HAOS preserves across boots).

## Verification after applying

```bash
sysctl net.ipv4.tcp_congestion_control    # should read: bbr
```

Then retry the HA Core update (`ha core update --version 2026.7.0` from the Supervisor side) and
confirm it completes without the `tls: bad record MAC` error, and faster than the ~760KB/s seen
under cubic.
