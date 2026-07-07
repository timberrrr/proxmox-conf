# /homeassistant/docs

Docs about *this* HA instance only. This is the subset that gets pushed to (and
should stay in sync with) `docs/homeassistant/` in the
[proxmox-conf](https://github.com/timberrrr/proxmox-conf) repo, which is the master
repo for the whole Proxmox homelab (host + every VM/CT).

- **[automations.md](automations.md)** — every HA automation, why it's built the way
  it is, and the reasoning behind the less-obvious design choices (sleep schedule
  science, the TV/casting saga, quiet hours). Read this before touching `automations.yaml`.
- **[integrations.md](integrations.md)** — HA integration inventory: what's connected,
  what changed recently, what was tried and abandoned (and why), what's deferred.
- **[haos-bbr-fix.md](haos-bbr-fix.md)** — HAOS core update failure diagnosis (TLS/BBR
  networking issue).

Cross-system docs that used to sit here (whole-homelab TODO, remote-access
architecture, an old migration-context dump, a raw terminal transcript) have been
removed — they belonged to the master repo, not this one, and in the case of
remote-access.md the copy here had gone stale relative to the master repo's version.
See `docs/README.md` and `docs/homeassistant/` in `proxmox-conf` for all of that.
