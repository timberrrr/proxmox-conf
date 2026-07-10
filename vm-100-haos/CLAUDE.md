# Home Assistant Config Repo — Workflow Notes

This repo is a **live HA config directory**, not a conventional project repo
reviewed via PRs. Edits here take effect on the running Home Assistant
instance directly (after a reload/restart), so the normal "isolate in a
worktree, commit, open a draft PR" workflow does not apply.

## Backup / GitHub sync

This repo itself has **no git remote**. Backup to GitHub is handled by an
automatic hook instead:

- `.claude/settings.json` registers a `PostToolUse` hook on `Write|Edit` that
  runs `.claude/sync-to-proxmox-conf.sh`.
- That script copies any tracked file just edited under `/homeassistant/*`
  into `timberrrr/proxmox-conf` (under `vm-100-haos/`, or `docs/homeassistant/`
  for `docs/*`), commits, and pushes — automatically, on every edit.
- So: edit the live file directly, and the GitHub backup happens on its own.
  No manual `git push` here, no PR.

## Background sessions

Background jobs are sandboxed by default to worktree-only edits, which
breaks this workflow (HA won't see a worktree copy, and the sync hook only
watches `/homeassistant/*`). `.claude/settings.json` sets:

```json
"worktree": { "bgIsolation": "none" }
```

to disable that guard for this repo, so background sessions can edit the
live config directly like foreground sessions do. If this setting is ever
missing/reverted, edits will be blocked with a worktree-isolation error —
re-add it (or ask the user to, since fixing it from inside a blocked
session is a catch-22: editing settings.json is itself blocked until the
setting is already in place).

## Practical implications

- Just edit files under `/homeassistant` directly — no EnterWorktree, no
  branch, no PR.
- After changing automations/scripts, call `automation.reload` /
  `script.reload` (via the `homeassistant` MCP server) or restart HA so the
  change takes effect.
- Secrets and generated state (`secrets.yaml`, `.storage/`, `.cloud/`,
  `tesla_fleet.key`, `custom_components/`, etc.) are gitignored and the sync
  hook skips anything not tracked — don't worry about those leaking to
  `proxmox-conf`.
