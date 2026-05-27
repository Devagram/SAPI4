---
name: sapi4-smoke-test
description: Verify a running SAPI4 deployment end-to-end. Lists voices, hits the index page, synthesizes a test phrase, and confirms the returned WAV is valid PCM audio.
---

# sapi4-smoke-test

Confirms the full pipeline works after a build or config change. Assumes `sapi4-run` has been invoked and the server is listening.

## 1. Voice enumeration (no server needed)

```bash
cd /mnt/c/temp/SAPI4/run && cmd.exe /c "cd C:\temp\SAPI4\run && sapi4limits.exe" 2>&1 | tr -d '\r' | head -20
```

Expected: list of voice names, e.g. `Adult Male #1, American English (TruVoice)`. Empty output means no TTS engine is registered.

## 2. Server reachable (via nginx)

```bash
cd /mnt/c && powershell.exe -NoProfile -Command "(Invoke-WebRequest -Uri 'http://localhost/SAPI4/' -UseBasicParsing).StatusCode"
```

Should print `200`.

## 3. Voice limits API

```bash
cd /mnt/c && powershell.exe -NoProfile -Command "\$v = 'Adult Male #1, American English (TruVoice)'; \$u = 'http://localhost/SAPI4/VoiceLimitations?voice=' + [Uri]::EscapeDataString(\$v); (Invoke-WebRequest -Uri \$u -UseBasicParsing).Content"
```

Should print a JSON object with `defPitch`, `minPitch`, `maxPitch`, `defSpeed`, `minSpeed`, `maxSpeed`.

## 4. Synthesize a WAV

```bash
cd /mnt/c && powershell.exe -NoProfile -Command "\$v = 'Adult Male #1, American English (TruVoice)'; \$u = 'http://localhost/SAPI4/SAPI4?text=this+is+a+smoke+test&voice=' + [Uri]::EscapeDataString(\$v); Invoke-WebRequest -Uri \$u -OutFile C:\temp\smoke.wav -UseBasicParsing; (Get-Item C:\temp\smoke.wav).Length"
```

Expected: a byte count (typically 20–80 KB for a short phrase). Then confirm format:

```bash
file /mnt/c/temp/smoke.wav
```

Should print: `RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 11025 Hz`.

## Diagnostic checklist when it fails

- 200 fails → server not running. Check `netstat`, restart via `sapi4-run`.
- 200 succeeds, voice limits 404 → voice name URL-encoded wrong. The `#` in `Adult Male #1` must become `%23`.
- Voice limits OK, synth returns 400 "Please reformat your text" → `sapi4out.exe` failed. Run it directly with the same args to see stderr.
- Synth returns 0-byte WAV → temp file written but deleted before stream finished. Investigate `app.d`'s `openFile`/`pipe`/`removeFile` ordering.
