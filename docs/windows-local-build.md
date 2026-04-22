# Building the Windows Desktop App Locally

Guide for producing a local (unsigned) Windows NSIS installer from a development machine.

## Prerequisites

| Tool                           | Required Version | Check                                                                 |
| ------------------------------ | ---------------- | --------------------------------------------------------------------- |
| Node.js                        | ^24.13.1         | `node --version`                                                      |
| Bun                            | ^1.3.9           | `bun --version`                                                       |
| Python                         | 3.10+            | `python --version`                                                    |
| Visual Studio 2022 Build Tools | MSVC v143        | `ls "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools"` |

### Node version management

The project requires Node 24. Use `fnm` to install and switch:

```powershell
fnm install 24
fnm use 24
# Optional: make it the default for new shells
fnm default 24
```

### Python for node-gyp

The `node-pty` native module uses node-gyp, which requires Python. If Python is not on PATH (e.g. installed via `uv`), set the environment variable explicitly:

```powershell
$env:PYTHON = "C:\Users\<you>\AppData\Roaming\uv\python\cpython-3.13.1-windows-x86_64-none\python.exe"
```

Adjust the path to match your Python install. The build script auto-discovers Python from common locations, but the env var is the most reliable method.

### Electron trusted dependency

Bun skips lifecycle scripts for packages not listed in `trustedDependencies`. The `electron` package must be listed there so its postinstall script downloads the Electron binary:

```jsonc
// package.json (root)
"trustedDependencies": [
  "electron",
  "node-pty"
]
```

If `electron` is missing from this list, the desktop app will fail at runtime with:

```
Error: Electron failed to install correctly, please delete node_modules/electron and try installing again
```

## Build Command

```powershell
fnm use 24
$env:PYTHON = "<path-to-python.exe>"
bun run dist:desktop:win
```

This runs `scripts/build-desktop-artifact.ts --platform win --target nsis --arch x64` which:

1. Builds all packages (`bun run build:desktop`)
2. Creates a temporary staging directory
3. Copies build artifacts and the Windows icon (`assets/prod/t3-black-windows.ico`)
4. Installs production-only dependencies in the staging area
5. Runs `electron-builder --win --x64 --publish never`
6. Copies the final `.exe` installer to `release/`

### Useful flags

| Flag                    | Purpose                                                            |
| ----------------------- | ------------------------------------------------------------------ |
| `--verbose`             | Stream full subprocess output (recommended for debugging)          |
| `--skip-build`          | Skip `bun run build:desktop`, reuse existing `dist/` artifacts     |
| `--keep-stage`          | Keep the temp staging directory after build (useful for debugging) |
| `--build-version X.Y.Z` | Override the version number in the installer                       |

Example with all debug flags:

```powershell
bun run dist:desktop:win -- --verbose --keep-stage
```

## Known Issue: node-pty native rebuild failures

The `@electron/rebuild` step recompiles `node-pty` (and its `winpty` dependency) against Electron's Node headers. On local Windows machines this can fail in two ways.

### Problem 1: `.bat` files not found in temp directories

```
'GetCommitHash.bat' is not recognized as an internal or external command
gyp: Call to 'cmd /c "cd shared && GetCommitHash.bat"' returned exit status 1
```

**Cause:** `winpty.gyp` calls `cmd /c "cd shared && GetCommitHash.bat"` to resolve a git commit hash. On some Windows configurations, `cmd.exe` does not find `.bat` files in the current directory without an explicit `.\` prefix when running inside temp paths.

**Fix:** Patch the two batch file references in the staging area's `winpty.gyp`:

```
# In: <stage>/node_modules/node-pty/deps/winpty/src/winpty.gyp

# Line ~13 — change:
'WINPTY_COMMIT_HASH%': '<!(cmd /c "cd shared && GetCommitHash.bat")',
# to:
'WINPTY_COMMIT_HASH%': '<!(cmd /c "cd shared && .\\GetCommitHash.bat")',

# Line ~25 — change:
'<!(cmd /c "cd shared && UpdateGenVersion.bat <(WINPTY_COMMIT_HASH)")',
# to:
'<!(cmd /c "cd shared && .\\UpdateGenVersion.bat <(WINPTY_COMMIT_HASH)")',
```

> **Note:** This only affects the ephemeral staging directory. The CI builds on GitHub Actions (`windows-2022` runner) do not hit this issue because the runner's `cmd.exe` configuration resolves `.bat` files without the prefix.

### Problem 2: Spectre-mitigated MSVC libraries not installed

```
error MSB8040: Spectre-mitigated libraries are required for this project.
Install them from the Visual Studio installer (Individual components tab)
```

**Cause:** Both `binding.gyp` (node-pty) and `winpty.gyp` set `'SpectreMitigation': 'Spectre'`, which requires the Spectre-mitigated MSVC libraries that are not installed by default with VS Build Tools.

**Fix (recommended):** Install the missing component via Visual Studio Installer:

1. Open Visual Studio Installer
2. Click **Modify** on "Build Tools 2022"
3. Go to the **Individual components** tab
4. Search for **"Spectre"**
5. Check **"MSVC v143 - VS 2022 C++ x64/x86 Spectre-mitigated libs (Latest)"**
6. Click **Modify**

**Fix (quick, local builds only):** Patch the staging area to disable Spectre mitigation:

```
# In: <stage>/node_modules/node-pty/binding.gyp
# and: <stage>/node_modules/node-pty/deps/winpty/src/winpty.gyp
#
# Replace all occurrences of:
'SpectreMitigation': 'Spectre'
# with:
'SpectreMitigation': 'false'
```

Then delete the stale `build/` directory so node-gyp regenerates the `.vcxproj` files:

```powershell
Remove-Item -Recurse -Force <stage>\node_modules\node-pty\build
```

> **Warning:** Disabling Spectre mitigation removes a security hardening feature. This is acceptable for local development builds but should not be used for production releases.

### Applying patches with --keep-stage

If the automated `bun run dist:desktop:win` fails at the `electron-builder` step, use `--keep-stage` to preserve the staging directory, apply the patches above, then manually run `electron-builder`:

```powershell
# 1. Run the build (will fail but staging dir is preserved)
bun run dist:desktop:win -- --keep-stage --verbose

# 2. Note the staging path from the output, e.g.:
#    C:\Users\<you>\AppData\Local\Temp\t3code-desktop-win-stage-XXXXXX\app

# 3. Apply patches to winpty.gyp and binding.gyp (see above)

# 4. Delete stale build dir
Remove-Item -Recurse -Force <stage>\app\node_modules\node-pty\build

# 5. Re-run electron-builder from the staging directory
cd <stage>\app
$env:PYTHON = "<path-to-python.exe>"
$env:npm_config_msvs_version = "2022"
$env:GYP_MSVS_VERSION = "2022"
$env:CSC_IDENTITY_AUTO_DISCOVERY = "false"
bunx electron-builder --win --x64 --publish never

# 6. Copy the installer out
Copy-Item dist\T3-Code-*.exe ..\..\..\..\release\
```

## Output

The installer is written to `release/T3-Code-<version>-x64.exe` (~137 MB, unsigned).
