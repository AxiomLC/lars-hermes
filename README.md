# Lars — Voice Server Module

**Lars** is AxiomLC's master agent platform on [Hermes Agent](https://github.com/NousResearch/hermes-agent) (NousResearch). This repo now serves ONE role in that platform:

> **The voice server** — a FastAPI pipeline: local Whisper STT → Hermes Agent brain → TTS (cloud Groq or local Kokoro, runtime-toggleable), streaming over WebSocket to any UI client.

**The Lars UI lives elsewhere:** [`hermes-workspace`](https://github.com/outsourc-e/hermes-workspace) (React + TS + Tailwind) is the base being themed into the Lars interface — 5 Div pages, glass cards, sharp edges, deep-blue futuristic theme. That repo is cloned separately at `C:\Users\Admin\hermes-workspace`; its UI work does not live here.

Full architecture, measured latencies, 5-Div spec, and machine-migration guide: **[SETUP.md](SETUP.md)**.

## What this repo provides

- `server/server.py` — voice pipeline (STT/TTS/brain wiring, sentence streaming, approval cards, secret redaction before cloud TTS)
- `server/hud/` — legacy single-file HUD (superseded by hermes-workspace for Lars; kept as reference/fallback)
- `hermes-plugin/` — tool plugin letting the agent summon HUD panels
- `client/`, `worker/` — optional push-to-talk client and GPU sidecars (unused in the Lars build)

## Quick facts (verified on the build machine)

| Leg | Implementation | Measured |
|---|---|---|
| STT | faster-whisper `tiny.en` int8 cpu — **fully local** | ~1.4–2.4s per utterance |
| TTS cloud | Groq Orpheus (`canopylabs/orpheus-v1-english`) | 1.36s to first audio |
| TTS local | Kokoro v1.0 ONNX, voice `bm_lewis` (lowest male, 92 Hz F0) | ~9.2s to first audio |
| Fallback | Cloud TTS failure → local Kokoro mid-turn, automatically | proven live |
| Toggle | Runtime switch, no restart | `/api/voice/settings` |

## Run

```bash
cd server
.venv/Scripts/python server.py        # Windows; use .venv/bin/python elsewhere
# Legacy HUD: https://localhost/hud/   ·  WS: ws://127.0.0.1:8765/ws
```

Requires the Hermes API server enabled (`API_SERVER_ENABLED=true`, port 8642) — see SETUP.md §2.

## Security model

- The Hermes API key never reaches the browser: UI clients talk through a strict allowlist proxy on the voice server.
- All endpoints + browser WebSockets are token-gated.
- Hermes' API binds to loopback only.
- Secret-shaped strings are redacted before text leaves for cloud TTS.
- LAN-only by design — do not port-forward to the internet.

## Credits & license

Built on [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research. Voice pipeline originally built on the MIT-licensed `jarvis_ai` project (voice HUD for Hermes); STT by [faster-whisper](https://github.com/SYSTRAN/faster-whisper) / [RealtimeSTT](https://github.com/KoljaB/RealtimeSTT); TTS by [Kokoro](https://github.com/thewh1teagle/kokoro-onnx) (local) and [Groq](https://groq.com) (cloud).

MIT — see [LICENSE](LICENSE).
