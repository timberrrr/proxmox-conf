# CT 101 — Ollama (STOPPED / DROPPED)

- **IP:** 192.168.1.10
- **Status:** STOPPED, `onboot=0` — kept intact, reversible
- **Specs:** unprivileged, 4c/4G/32G, NVIDIA GTX 1650 passthrough

## Why Dropped

Ollama (qwen2.5:3b) was rarely queried and outclassed by Claude/GPT, yet reserved ~2.2GB of the
4GB GPU 24/7 — forcing Frigate onto the weaker iGPU. Dropping it freed VRAM for higher-res Frigate
detection. HA Assist now uses the native Claude/Anthropic conversation agent instead.

Delete CT 101 later if never reverted.

## Model Notes (for reference if ever revived)

| Model | VRAM | Speed | Notes |
|-|-|-|-|
| qwen2.5:3b | 2.2GB | 47 tok/s | Best 3B tool-calling, was default |
| llama3.2:3b | ~2.2GB | 47 tok/s | A/B alternative |
| gemma3:4b | — | 0.6 tok/s | REJECTED — spills off GPU |
| phi4-mini | — | 2.6 tok/s | REJECTED — spills at 8k context |

Config (if revived): `OLLAMA_HOST=0.0.0.0:11434`, `FLASH_ATTENTION=1`, `KV_CACHE_TYPE=q8_0`,
`CONTEXT_LENGTH=8192`, `KEEP_ALIVE=15m` (yields VRAM when idle to allow Immich time-sharing).

**Note:** Frigate (always-on big model) + a usable LLM can't both be resident on 4GB.
Real fix if local LLM needed = bigger/second GPU (8–12GB).
