# CT 102 — fbmp-watcher (Docker host)

- **IP:** 192.168.1.11
- **Status:** running, healthy
- **Specs:** unprivileged, nesting, 2c/4G/20G, Debian 13

## Setup

Docker-in-LXC (unprivileged + nesting). Runs `fbmp-agent` (FB Marketplace Watcher) via Docker
Compose at `/opt/fbmp-agent`.

**Model:** `claude-haiku-4-5` on Anthropic API.
Ollama repoint is a later task — vision re-score uses Sonnet + Batches API (no Ollama equivalent);
3B local = quality drop on valuation. Hybrid is the eventual plan.

## State Migration

Migrated from dev workstation: `data/listings.sqlite`, `auth_state.json`, `config.yaml`, `.env`.

**IMPORTANT:** Do NOT restart the old local instance on the dev PC — double Telegram alerts +
FB session conflict. CT 102 is now the source of truth.

## Geo-drift guard (incident 2026-07-01, fix 2026-07-04)

FB Marketplace **ignores the search URL's `latitude`/`longitude`/`radius`** and serves
results from an **account-level "current location"**. That location drifts whenever the
*shared* FB account browses Marketplace anywhere — including the phone app centring on
another city. On 2026-07-01 the app was used to browse **Perth**, and the watcher (same
account) served ~229 Perth (WA) listings in a ~2-hour window, then self-corrected.

The existing drift guard (`pipeline.py::_check_region_sanity`) only alerted on the
US-fallback case (0 AU + ≥8 US-state listings) and was **blind to intra-Australia drift**:
WA was excluded from its regexes (Washington/Western-Australia ambiguity), so the Perth
flood fired **no alert** — silent failure.

**Fix:** recast the guard from "AU vs US" to **home-region vs not-home**. Home =
QLD/NSW/VIC/ACT/TAS/SA (states the widest Brisbane +1500 km searches can reach); WA/NT
(~2500 km+, unreachable) and US state codes = drift. A future drift now pings Telegram on
the first cycle with 0 home + ≥8 out-of-region listings. **Detection/alert only** — no
attempt to fight FB's location (it self-corrects in a cycle or two). Original file backed
up on the container as `src/fbmp/pipeline.py.bak.<stamp>`.

## Orphaned classify batch (incident 2026-07-03, fix 2026-07-04)

Adding a search introduced a new pin category, which changed the category hash and made
**all ~320 saved items "unclassified"**, kicking a bulk title→category classify via the
**Anthropic Batches API** (`classify_titles_batch`). Anthropic batches are **asynchronous**:
this one sat at `0/320` for its first few minutes and didn't finish until **8.5 min** in
(created 10:27:34, ended 10:36:00 UTC; `succeeded=320, errored=0`, cost ≈ $0.17).

The old progress message only emitted interim `N/320` lines with no hint that 0 is normal
early on, so `0/320` repeated looked **stuck**. The user restarted the host — which **killed
the poll-and-apply loop and orphaned the batch**: it completed and billed server-side, but
nothing was left alive to collect the results, so 320 classifications were **thrown away**
(evidenced by *zero* `batch=1` rows ever in `api_usage`). **Low credits were NOT the cause.**

**Fixes (all three):**
1. **Message (causal):** on submit, say it runs in the background and can take minutes with
   the count sitting at 0 ("no need to restart"); suppress the `0/N` spam (only ping once the
   count advances); always send an explicit done/failed line. `pinboard.py::classify_titles_batch`.
2. **Durability self-heal:** new `saved_pipeline.py::classify_unclassified_saved`, called as a
   background task at startup (`__main__.py`). Re-classifies any saved items whose stored
   category-hash ≠ current — so a batch orphaned by *any* restart/crash/redeploy (incl.
   `docker compose --build`) auto-recovers on the next boot. Idempotent → no-op once all items
   carry the current hash.
3. **One-off:** the post-deploy boot's self-heal re-classified the ~518 then-unclassified items.

Backups on the container: `src/fbmp/{pinboard,saved_pipeline,__main__}.py.bak.<stamp>`.

## Future Use

CT 102 is the designated host for other trivial Docker apps (single Compose). HA is NOT here — HA
is its own VM 100.
