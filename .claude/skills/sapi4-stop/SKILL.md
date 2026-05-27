---
name: sapi4-stop
description: Stop the running SAPI4 web server and nginx reverse proxy. Use before rebuilding (binaries can't be replaced while loaded) or to free ports.
---

# sapi4-stop

Terminates `sapi4.exe` and `nginx.exe` on the Windows host. Safe to run when nothing is running (taskkill will report no matching process; that's fine).

## Stop both

```bash
cmd.exe /c "taskkill /F /IM sapi4.exe /IM nginx.exe" 2>&1 | tr -d '\r'
```

## Verify

```bash
cd /mnt/c && cmd.exe /c "netstat -ano | findstr \"23451 :80\"" 2>&1 | tr -d '\r'
```

Should print nothing (or only non-LISTENING entries).

## Notes

- nginx spawns a master + worker; `/IM nginx.exe` catches both.
- If `sapi4.exe` was launched via `cmd.exe /c "... && sapi4.exe"` in the background through the harness, also use `TaskStop <task_id>` to clean up the harness-side wrapper.
- Don't `rm` `C:\temp\SAPI4` until after stopping — `sapi4.dll` is loaded by `sapi4.exe` and deletion will fail (or the .dll will be tombstoned and the process kept alive by Windows).
