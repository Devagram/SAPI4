# GitHub Copilot project instructions

Concise project context for Copilot suggestions. For longer/Claude-specific guidance see `CLAUDE.md` at repo root.

## What this is

Web interface (D + vibe-d) wrapping Microsoft's legacy SAPI 4 TTS engine, with C++ helpers (`sapi4out.exe`, `sapi4limits.exe`) that shell out per request. Used for character-voice TTS, primarily L&H TruVoice voices. Fork of `tetyys/SAPI4`.

## Stack

- **C++** (MSVC, /MT, 32-bit) — `sapi4.cpp`, `sapi4out.cpp`, `sapi4limits.cpp`. Direct COM calls against SAPI4.
- **D** (ldc2, x86_64) — `SAPI4_web/source/app.d`. vibe-d HTTP server, port 23451.
- **Diet templates + vanilla JS** — `SAPI4_web/views/`, `SAPI4_web/public/`.
- **nginx** (production) — front-proxies `/SAPI4/` to localhost:23451.

## Suggest

- vibe-d 0.9.x idioms for new HTTP handlers.
- `std.process.execute` / `pipeProcess` patterns for spawning the C++ helpers (match `app.d`'s existing usage).
- Diet template extensions of `layout.dt` for new pages.
- Plain-DOM JS in `tts.js` style — no framework, no build step.

## Avoid

- Adding `web-config` back to `dub.json` — it pulls `std.xml` which modern Phobos no longer ships. Inline any small XML parsing instead.
- Adding `std.xml` imports anywhere.
- Hardcoding paths to `C:\Program Files\...\Enterprise\...` — use Community or make configurable.
- Calling SAPI4 COM directly from D — keep that surface in the C++ helpers and shell out.
- Streaming-STT engines for the `feat-stt` branch — SAPI4 has no streaming synth, so the pipeline is utterance-batched and streaming-partials buy nothing. Use batch Whisper (faster-whisper local) instead.

## Conventions

- Tabs for indent across all languages here.
- Frontend asset URLs are prefixed with `/SAPI4/` (matches nginx mount point).
- Commit subjects: short imperative, ≤ 70 chars.
- Don't suggest CRLF normalizations in unrelated PRs — line-ending churn is a known repo issue and needs its own dedicated PR.
