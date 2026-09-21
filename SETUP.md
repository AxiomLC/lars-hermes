# SETUP — Lars (Hermes Agent Platform)

Master agent/platform name: **Lars**. This is the stable source of truth for environment, constraints, architecture, and the user's UI requirements.

**Architecture pivot (decided):** the Lars UI is built on **`hermes-workspace`** (React 19 + TS + Tailwind 4, zero-fork on vanilla Hermes) — NOT the legacy single-file HUD in this repo. This repo (`lars-hermes`) is now the **voice server module** only. See §10.

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
| Python | 3.11+ (venv per project) · Node 22 + pnpm (workspace UI) |
| Host user | `Admin` (hostname `Dave-new-folio`) |

## 2. Existing Hermes install (do NOT modify core)

- Hermes lives at `C:\Users\Admin\AppData\Local\hermes\` (Hermes home / `$HERMES_HOME` equivalent).
- Runs bare-metal with full machine access; protection model is **policy-based only**: `approvals.mode` (smart/manual) gates destructive shell commands, secret redaction filters tool output. There is **no OS-level sandbox** — treat agent commands accordingly.
- Integration surfaces (all documented, no core edits):
  - **Gateway API**: `API_SERVER_ENABLED=true`, port **8642**, loopback, in `AppData\Local\hermes\.env`
  - **Dashboard API**: port **9119** (sessions/skills/jobs/config) — `hermes dashboard --port 9119 --host 127.0.0.1 --no-open`
  - Secrets: `API_SERVER_KEY` + `JARVIS_HUD_TOKEN` (never hardcode, never commit)
- Profiles: independent agent instances under `AppData\Local\hermes\profiles\<name>\` — each has isolated skills, memories, cron, plugins. One active at a time.
- CLI: `hermes config set KEY VAL` for settings; **never hand-edit `config.yaml`**.
- Note: the Hermes **desktop app** ships its own wired mic/TTS in chat (confirmed v0.21.3). That voice belongs to the desktop client, not the 8642 API surface — third-party UIs still need this repo's voice server.

## 3. The two repos

| Repo | Role | Local path |
|---|---|---|
| `AxiomLC/lars-hermes` | **Voice server module** (FastAPI: whisper STT → Hermes → Groq/Kokoro TTS). Derivative of the MIT-licensed `jarvis_ai` voice-HUD project; upstream remote removed; legacy HUD superseded. | `C:\Users\Admin\lars-hermes` |
| `outsourc-e/hermes-workspace` | **UI base** (React 19 + TS + Tailwind v4, zero-fork on vanilla Hermes). Talks to gateway :8642 + dashboard :9119 only. | `C:\Users\Admin\hermes-workspace` |

## 4. Component decisions (final — verified on this machine)

| Component | Choice | Measured |
|---|---|---|
| STT | faster-whisper `tiny.en`, `int8`, `device=cpu` — **fully local** | ~1.4–2.4s per utterance |
| TTS cloud (opt-in) | **Groq Orpheus** `canopylabs/orpheus-v1-english`, voice `troy` | 1.36s to first audio |
| TTS local (default/privacy) | Kokoro v1.0 ONNX, voice **`bm_lewis`** (lowest male, median F0 92 Hz — all 12 male voices measured) | ~9.2s to first audio |
| Fallback | Cloud TTS failure → local Kokoro **mid-turn, automatically** (proven live) | — |
| Toggle | Runtime, no restart: `/api/voice/settings` | — |
| LLM brain | Hermes default OpenRouter GLM → fallback chain: OpenRouter DeepSeek v4 Flash → DeepInfra DeepSeek v4 Flash | — |

**Hybrid voice policy:** STT always local. TTS boots local; cloud is opt-in; cloud failure auto-falls-back to local. Spoken text reaches Groq only while CLOUD is selected.

**API keys:** voice needs **no keys in local mode** (one-time public model downloads: HF + kokoro-onnx GitHub releases). Cloud TTS uses `GROQ_API_KEY`. Brain uses OpenRouter + DeepInfra. All keys in Hermes `.env` only.

**Kokoro on local disk (gitignored, never commit):** `server/models/kokoro/` — `kokoro-v1.0.onnx` (311MB), `voices-v1.0.bin` (27MB, from kokoro-onnx GitHub releases `model-files-v1.0`; per-voice `.bin` files on HF are raw arrays kokoro-onnx cannot load), `config.json`, `tokenizer.json`. Quantized variants tested **slower** on this CPU — do not switch.

## 5. The 5 Divs (profiles/agents/modules)

The 5 profiles are organized as **Divs** with fixed numbers and UI highlight colors. Div 7 is the master — "Lars" himself.

| Div | Color theme | Role |
|---|---|---|
| **Div 7 — Master (Lars)** | dark blue | Master coordinator + voice mic module (lower-left corner). His knowledge base — always answers verbally about it. Dashboard: consolidated stats of all other Divs; social media posts/comments going out; GI Gross Income (manual weekly entry); truncated crucial-comms module (messages/email); stats of a few n8n flows; custom stock prices. Can pull a browser panel when there's web content to see/discuss. Dashboards use full width. |
| **Div 1 — Comms** | deep gold | WhatsApp, FB Messenger, other social DMs, emails (filtered to crucial), mobile voicemails. **Slack** is the master mobile-app communicator to/from Hermes/Lars (Slack mini-apps + dashboards). |
| **Div 3 — Records** | pink | Central files; working address-book DB of all known active connections (people & entities); active client records; invoices; Treasury. |
| **Div 4 — Coding Production** | green | **2 coding production agents:** (a) straight Python/JS — React+Vite / Vue frontend app building; (b) n8n specialist with integration to frontends and CRM functions into Div 6. All important MCPs + tools for AI app production. Native graph-DB app production. |
| **Div 6 — Public CRM** | yellow | Actual n8n flows doing marketing; any marketing DB; graph DBs. Connects into Div 1. |

Profile directory names: `div7` (master = "Lars" himself), `div1`, `div3`, `div4`, `div6`.

## 6. Hard rules

1. **Voice privacy by default, speed by choice:** STT always local. TTS boots local (Kokoro); cloud (Groq) opt-in, auto-fallback on failure.
2. No edits to Hermes core; integration only via documented surfaces (gateway 8642 + dashboard 9119) + profile mechanism. hermes-workspace stays **zero-fork** where feasible.
3. Secrets in `.env` only; settings in `config.yaml` only; both never committed. Model binaries never committed.
4. Profile switching from the UI must work without full restart; flag limitations — no silent fallback.
5. **Portability:** exportable state lives in `AppData\Local\hermes\` + the two repos. Export/share **excludes** `.env`, `auth.json`, `pairing/`, `sessions/`, `state.db*`.
6. **All future UI modules/plugins adopt the Lars theme** — graph DB viewers, comms summaries, CRM modules. No per-module styling.

## 7. UI requirements (user's stated desires — authoritative)

**Aesthetic (sitewide, incl. all future plugin pages):**
- **Glass look** for cards and stats displays — translucent panels (`--theme-glass` + `backdrop-filter: blur`)
- **Sharp edges everywhere** — border radius → 0
- **Deep blue futuristic base color**; edge highlights per Div (§5 colors) around Div pages
- **Futuristic font** — one font-stack swap in the theme block
- hermes-workspace ships `src/scifi-theme.css` (cyberpunk HUD starter) — fork its variable block into a "Lars" theme rather than starting from scratch. **Theme mechanism verified in source:** `[data-theme='id'] { --theme-*: ... }` blocks in `src/styles.css`; switching via `src/lib/theme.ts` (`root.setAttribute('data-theme', ...)`); 10 themes ship (Nous/Matrix/Hermes/Bronze/Slate/Mono × light/dark); `--theme-glass` is already a first-class token.

**Layout & behavior:**
- **Do NOT over-engineer voice.** Default Hermes desktop already has voice wired in its chat (mic + TTS buttons — user-confirmed). What Lars adds: a **dynamic mic module in the lower-left corner**, backed by this repo's voice server.
- The custom menu (left sidebar) stays. One menu item opens the **core Hermes sidebar/menu**; other core Hermes controls may sit top/bottom, restyled to match.
- **Div pages are basically full width.** The legacy orb layout addendum is **superseded** by this simpler model.
- **Input parity:** anything reachable by voice must also be reachable by click/type.
- **Wake word:** local background listener on **"Hey Lars"** wakes the voice surface (no cloud wake-word service).
- **All future modules adopt the Lars theme** — graph DB viewers, messaging summaries, CRM modules. No per-module custom styling.

## 8. Voice server — build state (this repo)

- Phase 1 ✅ Hermes API server on 8642; gateway autostarts at login
- Phase 2 ✅ STT pinned tiny.en/int8/cpu; <2s verified; no cloud STT paths
- Phase 3 ✅ Hybrid TTS (Groq 1.36s ↔ Kokoro 9.2s), runtime toggle, auto-fallback, Windows `.env` path fix, full round trip verified incl. agent tool calls
- Legacy HUD + orb layout: superseded (kept in `server/hud/` for reference)

## 9. Machine migration (porting to a stronger desktop / VPS)

1. **Repos:** clone `AxiomLC/lars-hermes` + the UI repo. Voice server venv:
   ```bash
   cd server && python -m venv .venv
   .venv/Scripts/pip install fastapi uvicorn requests pyyaml numpy scipy anthropic \
       RealtimeSTT faster-whisper silero-vad websockets psutil kokoro-onnx soundfile onnxruntime
   ```
2. **Models** (gitignored): re-fetch or copy `server/models/kokoro/` (URLs: HF `onnx-community/Kokoro-82M-v1.0-ONNX` + kokoro-onnx GitHub release `model-files-v1.0`); whisper tiny.en auto-downloads on first run.
3. **Hermes state:** zip `%LOCALAPPDATA%\hermes\` → target Hermes home, **EXCLUDING** `.env`, `auth.json`, `pairing/`, `sessions/`, `state.db*`, `logs/`, caches. Re-create `.env` keys fresh: `API_SERVER_ENABLED`, `API_SERVER_KEY`, `JARVIS_HUD_TOKEN`, `OPENROUTER_API_KEY`, `DEEPINFRA_API_KEY`, `GROQ_API_KEY`.
4. **Post-migration checklist:** `hermes gateway install` → `/health` on 8642 → dashboard on 9119 → voice server up → e2e voice test → toggle LOCAL↔CLOUD.

**On stronger hardware, revisit:** Kokoro local may become the daily default (re-benchmark); whisper could step up to `base.en`/`small.en`; multiple Divs may run concurrently.

## 10. The pivot — hermes-workspace as UI base (current plan)

**Why:** the user's clarified scope is ~90% UI (Div pages, glass theme, full-width layouts, corner mic) and hermes-workspace already ships: runtime profile switching, Operations/Dashboard/Kanban pages, sessions/skills/MCP/files/terminal as themed pages, a 10-theme CSS-variable system, and a scifi theme starter aligned with the futuristic aesthetic. Component architecture scales to the planned plugin pages (graph DB, CRM, comms summaries) far better than a single HTML file.

**Build order:**
1. Fork `outsourc-e/hermes-workspace` → `AxiomLC`, clone locally
2. Add the **"Lars" theme** (11th theme): deep-blue futuristic, glass cards, sharp edges, Div highlight scheme, futuristic font — new `[data-theme='lars']` block + registration in `src/lib/theme.ts`
3. Wire the 5 Divs to Hermes profiles via the workspace's runtime profile switching
4. Integrate this repo's voice server as the **corner mic module** (lower-left), restyled to theme
5. Verify profile switching works without restart; flag any backend limitation

**Windows stack notes (from the repo's AGENTS.md):** three services — gateway `:8642`, dashboard `:9119` (`hermes dashboard --port 9119 --host 127.0.0.1 --no-open`), workspace `:3000` (`pnpm dev`). Node 22+, pnpm required. Optional Electron app (`pnpm electron:dev`).

## 11. Acceptance checklist

- [x] Voice round trip works on CPU-only hardware (verified: full turn incl. agent tool call)
- [x] Hybrid TTS: cloud Groq (1.36s first audio) with proven local Kokoro fallback + runtime toggle
- [ ] Lars theme added to hermes-workspace (glass, sharp edges, deep blue, Div colors, futuristic font)
- [ ] All 5 Divs exist as isolated Hermes profiles, individually addressable, no shared memory/context
- [ ] UI profile switching works without full restart
- [ ] Corner mic module integrated and themed
- [ ] "Hey Lars" wake word works with Hermes running in background; zero cloud wake-word calls
- [ ] Hermes core unmodified; hermes-workspace stays zero-fork where feasible
- [ ] Export bundle reproducible: repos + Hermes state minus secrets
