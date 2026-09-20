# SETUP — Lars HUD on Hermes (Local-Voice Build)

Master agent/platform name: **Lars**. Core setup doc for the customized Hermes UI project. Full execution spec: see the build spec (Phase 1–5) referenced in the repo docs; this file is the stable source of truth for environment, constraints, and architecture.

---

## 1. Machine (fixed — do not re-detect)

| Item | Value |
|---|---|
| OS | Windows 10 (build 19045), bare metal — **not dockerised**, no container isolation |
| CPU | Intel i7-6600U — 2 cores / 4 threads, 2.60GHz |
| GPU | Intel HD 520 (integrated) — **CPU-only, no CUDA, no ML inference on GPU** |
| RAM | 16GB |
| Disk | 238GB SSD |
| Shell | Git Bash (MSYS2) — use POSIX syntax in scripts |
| Python | 3.11+ (venv per project) |
| Host user | `Admin` (hostname `Dave-new-folio`) |

## 2. Existing Hermes install (do NOT modify core)

- Hermes lives at `C:\Users\Admin\AppData\Local\hermes\` (Hermes home / `$HERMES_HOME` equivalent).
- Runs bare-metal with full machine access; protection model is **policy-based only**: `approvals.mode` (smart/manual) gates destructive shell commands, secret redaction filters tool output. There is **no OS-level sandbox** — treat agent commands accordingly.
- Integration surface: Hermes API server only.
  - `API_SERVER_ENABLED=true`, port **8642**, loopback-bound, in `AppData\Local\hermes\.env`
  - Generate `API_SERVER_KEY` + `JARVIS_HUD_TOKEN` at setup (never hardcode, never commit)
- Profiles: independent agent instances under `AppData\Local\hermes\profiles\<name>\` — each has isolated skills, memories, cron, plugins. One active at a time.
- CLI reference: `hermes config set KEY VAL` for settings; **never hand-edit `config.yaml`** (stray indent can corrupt the live gateway).

## 3. Base project

- Fork/clone: `https://github.com/eadmin2/jarvis_ai` (MIT) → local path `C:\Users\Admin\jarvis-hermes-hud`
- Fork on GitHub first so we can push our customizations; clone the fork.
- Reuse as-is: `server/` (FastAPI voice pipeline + HUD host), `server/hud/` (single-file vanilla JS — no build step), `hermes-plugin/` (agent summons HUD panels), `client/` (push-to-talk).
- Upstream tested on macOS/Apple Silicon; Windows needs the documented launch/systemd → Task Scheduler / service adjustments.

## 4. Component decisions (final — updated after Phase 3)

| Component | Choice | Rationale |
|---|---|---|
| STT | `faster-whisper` `tiny.en`, `int8`, `device=cpu` — **fully local, verified** (~1.4–2.4s per utterance) | Real-time on dual-core CPU; never leaves the machine |
| TTS cloud (default toggle) | **Groq Orpheus** `canopylabs/orpheus-v1-english`, voice `troy` — **1.36s to first audio** (measured) | Fastest; requires one-time model-terms acceptance in Groq console + `GROQ_API_KEY` |
| TTS local (fallback / privacy mode) | Kokoro v1.0 ONNX, voice **`bm_lewis`** (lowest male, median F0 92 Hz measured across all 12 male voices) | ~9.2s to first audio on i7-6600U — acceptable in privacy mode; auto-fallback target |
| Voice host | `jarvis_ai/server` (FastAPI) | Streaming STT → Hermes → TTS; sentence-level streaming |
| Architecture | **5 isolated Hermes profiles (Divs)**, one active at a time | CPU can't run concurrent agents; isolation keeps memory/context/skills per module |

**Hybrid voice (decided, supersedes original zero-cloud rule #1):** the HUD has a runtime **LOCAL↔CLOUD** voice toggle (`/api/voice/settings`, no restart). Cloud failure auto-falls-back to local Kokoro mid-turn (proven live). STT always stays local.

**API keys:** voice needs **no keys in local mode** (models are one-time public downloads from Hugging Face / kokoro-onnx GitHub releases). Cloud mode uses `GROQ_API_KEY` (also supports ElevenLabs). The brain uses OpenRouter + DeepInfra. All keys live only in the Hermes `.env`.

**Kokoro on local disk (gitignored, do not commit):** `server/models/kokoro/` — `kokoro-v1.0.onnx` (311MB), `voices-v1.0.bin` (27MB, from kokoro-onnx GitHub releases `model-files-v1.0`; the per-voice `.bin` files on HF are raw arrays kokoro-onnx cannot load), `config.json`, `tokenizer.json`. Quantized variants (`model_quantized`, `model_q8f16`) tested **slower** on this CPU — do not switch.


## 5. The 5 Divs (profiles/agents/modules)

The 5 profiles are organized as **Divs** with fixed numbers and HUD theme colors. Div 7 is the master — "Lars" himself.

| Div | Color theme | Role |
|---|---|---|
| **Div 7 — Master (Lars)** | dark blue | Master coordinator + voice mic module. His knowledge base — always answers verbally about it. Dashboard: consolidated stats of all other Divs; social media posts/comments going out; GI Gross Income (manual weekly entry); truncated crucial-comms module (messages/email); stats of a few n8n flows; custom stock prices. Can pull a HUD browser panel when there's web content to see/discuss. Dashboards use full width. |
| **Div 1 — Comms** | deep gold | WhatsApp, FB Messenger, other social DMs, emails (filtered to crucial), mobile voicemails. **Slack** is the master mobile-app communicator to/from Hermes/Lars (Slack mini-apps + dashboards). |
| **Div 3 — Records** | pink | Central files; working address-book DB of all known active connections (people & entities); active client records; invoices; Treasury. |
| **Div 4 — Coding Production** | green | **2 coding production agents:** (a) straight Python/JS — React+Vite / Vue frontend app building; (b) n8n specialist with integration to frontends and CRM functions into Div 6. All important MCPs + tools for AI app production. Native graph-DB app production. |
| **Div 6 — Public CRM** | yellow | Actual n8n flows doing marketing; any marketing DB; graph DBs. Connects into Div 1. |

Profile directory names: `div7` (master, effectively the default "Lars" profile), `div1`, `div3`, `div4`, `div6`. Exact theme hexes for each Div color are a Phase 5 styling-pass decision. Verify current profile-creation CLI syntax against https://hermes-agent.nousresearch.com/docs/ before implementing.

## 6. Hard rules

1. **Voice privacy by default, speed by choice:** STT is always local. TTS boots local (Kokoro); cloud (Groq) is opt-in via the HUD toggle and auto-falls-back to local on failure. Spoken text goes to Groq's servers only while CLOUD is selected.
2. No edits to Hermes core source; integration only via API server + profile mechanism.
3. Secrets go in `.env` only; settings in `config.yaml` only; both never committed.
4. HUD customizations happen in `server/hud/` in place — no build step introduced.
5. Profile switching from the HUD must work without full server restart; if the backend only supports one profile per process, that's a flagged scope change, not a silent fallback.
6. This repo (and the whole setup) must stay **portable**: everything exportable lives in `AppData\Local\hermes\` (skills, memories, cron, platforms, profiles, SOUL.md) plus this repo. When exporting or sharing: **exclude `.env`, `auth.json`, `pairing/`, `sessions/`, `state.db*`** — secrets and private session history.

## 7. Addendum — layout & wake word (revises Phase 5)

**Layout is state-driven, not fixed:**
- Idle / first load: voice orb is the dominant, large element — primary surface.
- On activity start (first message / tool-call / render event from the active profile): orb auto-relegates to a small persistent corner module. Trigger off the **real event**, never a fixed timer (timers clip fast exchanges).
- Once relegated, the active profile's native view owns the freed space at near-full width — **not** a jarvis_ai embedded viewer panel. Do not route module content through jarvis_ai's viewers.
- Reverts to full-orb on idle (threshold: N seconds with no activity, or explicit close/done action).
- Orb stays a live control surface at small size (mic active), never decorative.
- **Input parity at every size:** anything reachable by voice must also be reachable by click/type, in both large and relegated states. Nothing voice-only.
- Phase 5 menu visuals are provisional — expect a styling pass after first render; do not hard-code positions/sizes assuming the jarvis_ai skin is final.

**Gate:** relegate/expand transitions fire reliably off real activity events across all 5 profiles, not just the first one tested.

**jarvis_ui viewer may be dropped entirely** — it wastes space; we may tweak or replace it. Decision deferred until first render.

**Wake word:** local background listener triggers on **"Hey Lars"** while Hermes is alive in the background — wakes the voice surface (replaces/augments jarvis_ai's push-to-talk ring as the primary activation path). Local VAD/keyword-spotting only — no cloud wake-word service.

## 8. Install sequence (summary)

```bash
# 0. Fork eadmin2/jarvis_ai on GitHub, then:
git clone https://github.com/<YOU>/jarvis-hermes-hud.git
cd jarvis-hermes-hud/server
python -m venv .venv
.venv/Scripts/pip install fastapi uvicorn requests pyyaml numpy anthropic \
    RealtimeSTT faster-whisper silero-vad websockets psutil

# 1. Enable Hermes API server (loopback) — edit AppData\Local\hermes\.env:
API_SERVER_ENABLED=true
API_SERVER_KEY=<secrets.token_urlsafe(32)>
JARVIS_HUD_TOKEN=jarvis-<random hex>
# ELEVENLABS_API_KEY must NOT be needed after Phase 3

# 2. Start: hermes gateway  →  verify 127.0.0.1:8642 reachable
# 3. Boot server → HUD loads, typed chat works first (gate before Phase 2)
# 4. Pin STT (tiny.en/int8/cpu) → 10s utterance transcribes <2s, no outbound calls
# 5. Swap TTS provider to local Kokoro ONNX (model + voice pack downloaded locally at setup)
# 6. Create 5 profiles → HUD menu switching, no restart
```

## 9. Migration — porting this whole setup to another machine (e.g. stronger desktop / VPS)

Everything splits into three bundles:

1. **This repo** (`git clone AxiomLC/lars-hermes`) — code + docs. On the target:
   ```bash
   cd server && python -m venv .venv
   .venv/Scripts/pip install fastapi uvicorn requests pyyaml numpy scipy anthropic \
       RealtimeSTT faster-whisper silero-vad websockets psutil kokoro-onnx soundfile onnxruntime
   bash scripts/make-certs.sh      # regenerates TLS certs for the NEW machine's LAN IP
   ```
2. **Model downloads** (gitignored — re-fetch or copy `server/models/kokoro/`):
   ```bash
   curl -L -o server/models/kokoro/kokoro-v1.0.onnx  https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX/resolve/main/onnx/model.onnx
   curl -L -o server/models/kokoro/voices-v1.0.bin   https://github.com/thewh1teagle/kokoro-onnx/releases/download/model-files-v1.0/voices-v1.0.bin
   curl -L -o server/models/kokoro/config.json       https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX/resolve/main/config.json
   curl -L -o server/models/kokoro/tokenizer.json    https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX/resolve/main/tokenizer.json
   ```
   (whisper `tiny.en` auto-downloads to the HF cache on first run)
3. **Hermes state** — zip `%LOCALAPPDATA%\hermes\` from the source machine → `~/.hermes/` (Linux/mac) or `%LOCALAPPDATA%\hermes\` (Windows) on the target, **EXCLUDING**: `.env`, `auth.json`, `pairing/`, `sessions/`, `state.db*`, `logs/`, caches. Re-create `.env` entries fresh on the target: `API_SERVER_ENABLED=true`, `API_SERVER_KEY`, `JARVIS_HUD_TOKEN`, `OPENROUTER_API_KEY`, `DEEPINFRA_API_KEY`, `GROQ_API_KEY`.

**Post-migration checklist:** `hermes gateway install` (login autostart) → `/health` on 8642 → boot `server/server.py` → run `scripts/ws_e2e_test.py` with a 16k WAV → toggle voice LOCAL↔CLOUD in HUD → Groq terms already accepted per org (no re-accept needed unless new org).

**On stronger hardware, revisit:** Kokoro local mode will get much faster (worth re-benchmarking — the local-privacy mode may become the daily default); whisper could step up to `small.en`/`base.en` for accuracy; multiple Divs may even run concurrently if RAM/CPU allow.

## 10. Acceptance checklist

- [ ] No outbound network calls for STT/TTS in default operation (verified by inspection)
- [x] Voice round trip works on CPU-only hardware (verified: full turn incl. agent tool call)
- [x] Hybrid TTS: cloud Groq (1.36s first audio) with proven local Kokoro fallback + HUD toggle
- [ ] All 5 profiles exist, isolated, individually addressable, no shared memory/context
- [ ] HUD menu switches active profile without full server restart (or limitation documented + flagged)
- [ ] Orb relegate/expand fires off real activity events across all 5 profiles; input parity at every size
- [ ] "Hey Lars" wake word works with Hermes running in background; zero cloud wake-word calls
- [ ] Hermes core unmodified
- [ ] Export bundle reproducible: zip of config/SOUL/skills/memories/cron/platforms/profiles minus secrets
