# Home Assistant (VM 100) — HA-instance docs

Docs specific to the running HA instance itself (automations, integrations), as
opposed to the rest of this repo which covers the Proxmox host and other VMs/CTs.
See [vm-100-haos.md](../vm-100-haos.md) for the VM/infra side (BBR, IP ban, etc.).

- **[automations.md](automations.md)** — every HA automation and the reasoning
  behind the less-obvious design choices (sleep schedule science, the TV/casting
  saga, quiet hours).
- **[integrations.md](integrations.md)** — HA integration inventory: what's
  connected, what changed recently, what was tried and abandoned (and why).
- **[haos-bbr-fix.md](haos-bbr-fix.md)** — HAOS core update failure diagnosis
  (TLS/BBR networking issue specific to the HAOS VM).

Maintained from the HA-device Claude instance; pushed here as the source of truth
for anything HA-instance-specific. Cross-system docs (Jellyfin, remote access,
Frigate, etc.) live at the repo root under `docs/`, not here.
