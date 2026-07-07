# Proxmox Homelab

## Start here
Read `docs/README.md` first. It has connection params, inventory, current status, and next actions.
Per-host detail is in `docs/ct-<id>-<name>.md`.

## Security constraint
**OneDrive credentials must NEVER be stored on the Proxmox host or any LXC container.**
The OneDrive account is too sensitive. Music sync and photo backup use Windows-side tooling
(robocopy over SMB) — not rclone on the server. Do not suggest or implement any approach
that stores OneDrive tokens, credentials, or rclone remotes on the server.
