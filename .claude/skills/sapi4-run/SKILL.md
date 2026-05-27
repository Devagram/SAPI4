---
name: sapi4-run
description: Start the SAPI4 web server (sapi4.exe on 127.0.0.1:23451) and nginx (port 80) reverse proxy so the UI is reachable at http://localhost/SAPI4/. Assumes binaries are already built and in C:\temp\SAPI4\run.
---

# sapi4-run

Starts the SAPI4 stack locally. Run `sapi4-build` first if binaries are missing or stale.

## Preconditions

- `C:\temp\SAPI4\run\sapi4.exe` exists (build done).
- SAPI4 runtime installed: `C:\Windows\Speech\speech.dll` present.
- A TTS engine registered: `C:\Windows\lhsp\tv\tv_enua.dll` (TruVoice) at minimum.
- nginx installed (e.g. via scoop: `scoop install nginx`).
- nginx config has `location ^~ /SAPI4/ { proxy_pass http://127.0.0.1:23451/; }` in the default server block.

## Start the D server

```bash
cmd.exe /c "cd C:\temp\SAPI4\run && sapi4.exe" 2>&1 | tr -d '\r' | tee /tmp/sapi4_server.log
```

Run in background via the harness's `run_in_background` parameter. Confirm with:

```bash
cd /mnt/c && cmd.exe /c "netstat -ano | findstr 23451" 2>&1 | tr -d '\r'
```

Should show `TCP 127.0.0.1:23451 ... LISTENING <pid>`.

## Start nginx (elevated, for port 80)

```bash
cd /mnt/c && powershell.exe -NoProfile -Command "Start-Process -FilePath 'C:\Users\tomp4\scoop\apps\nginx\current\nginx.exe' -ArgumentList '-p','C:\Users\tomp4\scoop\apps\nginx\current' -Verb RunAs -WindowStyle Hidden"
```

(UAC prompt will appear on Windows desktop; user confirms.)

Confirm:

```bash
cd /mnt/c && cmd.exe /c "netstat -ano | findstr :80 " 2>&1 | tr -d '\r' | grep LISTENING
```

## Verify end-to-end

```bash
cd /mnt/c && powershell.exe -NoProfile -Command "(Invoke-WebRequest -Uri 'http://localhost/SAPI4/' -UseBasicParsing).StatusCode"
```

Should print `200`. Browser visit: **http://localhost/SAPI4/**.

For a synthesis smoke test, invoke `sapi4-smoke-test`.

## Common failures

- **`sapi4.exe` exits with `sapi4limits fail`** — TruVoice (or any TTS engine) not registered. Reinstall `tv_enua.exe` elevated; confirm `C:\Windows\Speech\Engines\TTS\` has entries (or registry under `HKLM\Software\Microsoft\Speech\Voices`).
- **port 23451 already listening** — old `sapi4.exe` running. Run `sapi4-stop` first.
- **nginx says address already in use** — IIS or another web server on :80. Stop it or change nginx's listen directive.
- **404 on `/SAPI4/scripts/tts.js`** through nginx — proxy_pass path is missing the trailing `/`. Must be `proxy_pass http://127.0.0.1:23451/;` (slash matters).
