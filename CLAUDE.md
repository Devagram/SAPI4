# CLAUDE.md

Project-level instructions for Claude Code (and similar AI assistants) working in this repo. Read this before exploring or making changes — it captures load-bearing conventions that aren't obvious from the code alone.

## What this project is

A web interface around Microsoft's legacy **SAPI 4** Text-to-Speech engine, serving the L&H TruVoice voices (the "Adult Male #N / Adult Female #N" voices, plus Microsoft Sam-style if a separate engine is installed). Originally `tetyys/SAPI4`; this fork is `Devagram/SAPI4`. Used for character-voice content / streaming bits.

- **C++ helpers** (`sapi4.cpp/.hpp`, `sapi4out.cpp`, `sapi4limits.cpp`) — thin wrappers over the SAPI4 COM API. Build to `sapi4.dll`, `sapi4out.exe`, `sapi4limits.exe`. **Windows-only** (32-bit COM).
- **D web server** (`SAPI4_web/source/app.d`) — vibe-d HTTP server. Spawns the C++ helpers per request, streams resulting WAV back. Builds to `sapi4.exe`.
- **Frontend** (`SAPI4_web/views/`, `SAPI4_web/public/`) — Diet templates + plain JS. Assumes mounted under `/SAPI4/` (asset paths hard-code the prefix); use nginx `proxy_pass http://127.0.0.1:23451/` to map `/SAPI4/` → server root.

Live deployment is Linux + Wine + Xvfb per upstream README. Local dev on this machine is Windows-native (see SETUP_NOTES.md).

## Architecture at a glance

See `SETUP_NOTES.md` for the full mermaid diagram. TL;DR of request flow:

```
Browser → nginx :80 (/SAPI4/) → sapi4.exe :23451 → sapi4out.exe → temp.wav → stream back
                                              ↓
                                      sapi4limits.exe at startup (enumerate voices)
```

`sapi4out.exe` and `sapi4limits.exe` use COM to load `C:\Windows\Speech\speech.dll` (SAPI4 runtime) which loads the registered TTS engine DLL (e.g. `C:\Windows\lhsp\tv\tv_enua.dll` for TruVoice).

## Branches

- **`master`** — production / mainline.
- **`feat-stt`** — work-in-progress speech-to-text → SAPI4-TTS pipeline for live-streaming as a character voice. See `FEAT_STT_PLAN.md` on that branch for the design + checklist. Critical architectural decision: **the pipeline is utterance-batched, not streaming-with-partials**, because SAPI4 has no streaming synth — gating on VAD silence is structurally forced and partials are wasted. Don't propose streaming STT engines (Deepgram streaming, Vosk, etc.) for this pipeline.

## Build & run

Local dev environment: **WSL2 driving the Windows host via interop**. Source of truth lives in WSL (`/home/tomp4/SAPI4`); a transient build copy lives at `C:\temp\SAPI4` (cmd.exe can't use UNC paths as CWD, so a copy onto NTFS is unavoidable for builds).

Full build/run instructions: `SETUP_NOTES.md`. Quick reference:

- C++ build: copy source to `C:\temp\SAPI4`, run `build.bat` (uses VS 2022 Community vcvars32).
- D build: `dub --compiler=ldc2 --arch=x86_64 --build=release` in `SAPI4_web/`. Note: x86_64, not x86 as the upstream README says — the installed LDC is x64-only and the server only `execute()`s the SAPI4 .exes (doesn't link against them), so the parent process arch doesn't have to match.
- Run: `sapi4.exe` from a directory containing `sapi4.dll`, `sapi4out.exe`, `sapi4limits.exe`, and `public/`. Bind: `127.0.0.1:23451`.
- nginx in front: `location ^~ /SAPI4/ { proxy_pass http://127.0.0.1:23451/; }`.

Project skills under `.claude/skills/` wrap these — prefer invoking them over re-typing the commands.

## Critical conventions

- **`web-config` dependency was removed** from `SAPI4_web/dub.json` — it pulled in `std.xml`, which Phobos has dropped. Its only callsite in `app.d` was a dead `if (false) { readOption(...) }` block guarding the long-retired OPTLINK linker, also removed. Don't re-add web-config.
- **`build.bat` uses VS 2022 Community path**, not Enterprise as the upstream README assumed. If contributors have a different VS edition, they'll need to adjust locally (don't commit their path).
- **Frontend asset paths hard-code `/SAPI4/`** in `views/layout.dt` and `public/scripts/tts.js`. Hitting the server directly at `:23451` shows a broken page (404 on assets). Either use the nginx proxy or rewrite those paths.
- **CRLF / line-ending churn** when files are copied across `/home/` ↔ `/mnt/c/`. Always diff with `-w` when comparing across the boundary. NTFS-side clones perpetually show ~10 files as "modified" due to git's autocrlf — these are phantom diffs, ignore them; real changes show up alongside as additional `M` entries with actual content deltas.
- **Single source of truth: the WSL clone.** Do not maintain parallel clones on `/mnt/c/...` for source editing. The only legitimate Windows-side copy is the transient build directory at `C:\temp\SAPI4`.

## Files worth knowing

| Path | What |
|---|---|
| `sapi4.cpp` / `.hpp` | C++ SAPI4 COM wrapper, compiled to `sapi4.dll` |
| `sapi4out.cpp` | CLI: `sapi4out <voice> <pitch> <speed> <text>` → writes WAV, prints path |
| `sapi4limits.cpp` | CLI: lists voices, or `sapi4limits <voice>` for pitch/speed limits |
| `build.bat` | MSVC build script (Windows-only) |
| `SAPI4_web/source/app.d` | D web server — URLRouter, voice cache, request handlers |
| `SAPI4_web/dub.json` | D dependencies |
| `SAPI4_web/views/index.dt`, `layout.dt` | Diet templates for the web UI |
| `SAPI4_web/public/scripts/tts.js` | Frontend JS (calls `/SAPI4/SAPI4` and `/SAPI4/VoiceLimitations`) |
| `SETUP_NOTES.md` | Bring-up notes for this machine (Windows-native build path) |
| `FEAT_STT_PLAN.md` (feat-stt branch) | STT pipeline design + checklist |

## Editing guidance

- Don't add Windows-specific tooling suggestions (PowerShell scripts, GH Desktop workflows) when a WSL-native equivalent works. SAPI4 itself requires Windows; everything else should prefer Linux.
- Never edit files inside `C:\temp\SAPI4` and treat that as source — always edit in `/home/tomp4/SAPI4` and re-copy to build.
- Don't commit `.env`, `.vs/`, `dub.selections.json`, or build artifacts.
- Match commit message style: short imperative subject (≤ 70 chars), optional body explaining *why*, end with `Co-Authored-By:` if AI-assisted.
