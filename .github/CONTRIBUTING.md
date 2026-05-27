# Contributing

Thanks for taking a look. This is a small project — the contribution process is intentionally informal.

## Setting up dev environment

See [`SETUP_NOTES.md`](../SETUP_NOTES.md) at the repo root for the full bring-up walkthrough. TL;DR:

- **Build host**: any Windows machine with Visual Studio 2019+ (Community or higher) and the Microsoft Speech SDK 4.0 (`SAPI4SDK.exe`) installed.
- **Runtime**: Windows with SAPI4 runtime (`spchapi.exe`) and the L&H TruVoice TTS engine (`tv_enua.exe`) installed — both ship in this repo. The upstream README also covers running on Linux + Wine.
- **D toolchain**: `ldc2` + `dub`. Modern LDC is x64-only; build the D server with `--arch=x86_64`.

## Branches & workflow

- Branch from `master`. Name branches with a short prefix indicating intent: `feat-*`, `fix-*`, `docs-*`, `refactor-*`.
- Keep PRs small and focused. One bug fix per PR; if you find more than one, open multiple.
- Rebase rather than merge when bringing your branch up to date with `master`.

## Commit messages

- Short imperative subject (≤ 70 characters). Examples in this repo's history: `Fix speed error message and x255 note`, `Call memset() to zero out VOICE_INFO`.
- Optional body explaining *why* the change was made, wrapped at ~72 chars.
- If you used an AI assistant, include a `Co-Authored-By:` trailer for transparency.

## Code style

No formal linter / formatter is configured. Match the surrounding style:

- **C++**: K&R-ish braces, tabs for indent.
- **D**: tabs, vibe-d idioms (see `app.d` for examples).
- **Diet templates / JS**: tabs.

## Heads-up: line endings

The repo mixes contributions from Windows and Linux developers and does not currently enforce a `.gitattributes` line-ending policy. If you see ~10 unrelated files appearing as modified after a fresh checkout, those are almost certainly CRLF/LF flips, not content changes. `git diff -w` ignores them. Don't include those phantom diffs in your PR — stage only files you actually edited.

If this becomes a recurring issue, a single PR adding `* text=auto` to `.gitattributes` and renormalizing everything would be welcome.

## Pull requests

- Open against `master` of `Devagram/SAPI4`.
- Include in the description: what the change does, why, and how you tested it.
- If the change affects the SAPI4 web API or alters the build process, note it explicitly so users can update their setups.

## Reporting bugs

Open an issue with: what you ran, what you expected, what you got. If it's a TTS audio output bug, include the request URL (with the `voice`/`pitch`/`speed` params) and, if you can, the resulting WAV.
