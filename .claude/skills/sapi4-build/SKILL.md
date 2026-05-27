---
name: sapi4-build
description: Build the SAPI4 C++ binaries (sapi4.dll, sapi4out.exe, sapi4limits.exe) and the D web server (sapi4.exe). Stages source from the WSL repo to C:\temp\SAPI4 first because cmd.exe can't use UNC paths as CWD.
---

# sapi4-build

Builds the project's native binaries. Run after editing any of: `sapi4.cpp/.hpp`, `sapi4out.cpp`, `sapi4limits.cpp`, `build.bat`, `SAPI4_web/source/app.d`, or `SAPI4_web/dub.json`.

## Why staging is needed

cmd.exe (used to run MSVC's `vcvars32.bat` and `dub.exe`) cannot use a UNC path (`\\wsl.localhost\Ubuntu\...`) as its current directory. So we copy the WSL source tree to NTFS first, build there, and treat the result as throwaway. Source of truth stays in WSL.

## Steps

1. **Stage source.** Mirror the WSL tree to `C:\temp\SAPI4`, excluding `.git/` and build outputs:

   ```bash
   rsync -a --delete \
     --exclude '.git/' --exclude '.dub/' --exclude 'dub.selections.json' \
     --exclude '*.exe' --exclude '*.dll' --exclude '*.lib' --exclude '*.obj' --exclude '*.exp' \
     /home/tomp4/SAPI4/ /mnt/c/temp/SAPI4/
   ```

2. **Build C++.** Invoke MSVC via cmd.exe (uses VS 2022 Community vcvars32):

   ```bash
   cd /mnt/c/temp && cmd.exe /c "cd C:\temp\SAPI4 && build.bat" 2>&1 | tr -d '\r' | tail -40
   ```

   Expected output: three `cl` invocations, each producing an `.exe` or `.dll`. Should end without errors.

3. **Build D server.** From `C:\temp\SAPI4\SAPI4_web`:

   ```bash
   cd /mnt/c/temp && cmd.exe /c "cd C:\temp\SAPI4\SAPI4_web && dub --compiler=ldc2 --arch=x86_64 --build=release" 2>&1 | tr -d '\r' | tail -20
   ```

   Note: `--arch=x86_64`, NOT `--arch=x86` (installed LDC is x64-only; the D server only spawns the C++ .exes, doesn't link them, so parent process arch is independent).

4. **Assemble run dir.** Collect outputs into `C:\temp\SAPI4\run\`:

   ```bash
   cp /mnt/c/temp/SAPI4/sapi4.dll /mnt/c/temp/SAPI4/sapi4out.exe /mnt/c/temp/SAPI4/sapi4limits.exe \
      /mnt/c/temp/SAPI4/SAPI4_web/sapi4.exe \
      /mnt/c/temp/SAPI4/SAPI4_web/libcrypto-1_1-x64.dll /mnt/c/temp/SAPI4/SAPI4_web/libssl-1_1-x64.dll \
      /mnt/c/temp/SAPI4/run/
   cp -r /mnt/c/temp/SAPI4/SAPI4_web/public /mnt/c/temp/SAPI4/SAPI4_web/views /mnt/c/temp/SAPI4/run/
   ```

## If MSVC isn't found

`build.bat` calls a hardcoded vcvars32 path. If `cl` errors with "not recognized," the VS edition path is wrong. Update `build.bat` (then re-stage) to point at the installed edition: Community / Professional / Enterprise / BuildTools all live under `C:\Program Files\Microsoft Visual Studio\2022\<edition>\VC\Auxiliary\Build\vcvars32.bat`.

## Common failures

- **"unable to read module `xml`"** from web-config — means `web-config` is back in `dub.json`. Remove it; this repo doesn't use it (the optlink-workaround callsite was deleted long ago).
- **LNK4272 architecture mismatch** — you used `--arch=x86`. Use `x86_64`.
- **dub builds web-config anyway despite `dub.json` not listing it** — stale `dub.selections.json`. `rm SAPI4_web/dub.selections.json` and rebuild.
