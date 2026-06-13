# Joplin Windows ARM64 Build Procedure

## Contents

- Environment discovery
- Toolchain setup
- Dependency installation
- Electron native rebuild
- Bundling and packaging
- Architecture verification
- Smoke testing
- Known failure modes

## Environment Discovery

Run from the repository root:

```powershell
git status --short
Get-Content package.json
Get-Content packages\app-desktop\package.json
Get-Content packages\app-desktop\tools\electronRebuild.js
node -p "process.version + ' ' + process.platform + ' ' + process.arch"
```

Read the versions from the repository. Values verified during the original
build were Node 24.12.0, Yarn 4.16.0, Electron 42.3.0, and Electron ABI 146.
These are examples, not permanent constants.

The Node check must report `win32 arm64`.

## Toolchain Setup

Required components:

- Native Windows ARM64 Node.js
- Python 3
- Visual Studio 2022 Build Tools
- Visual C++ ARM64 compiler and libraries
- Windows 11 SDK
- Git

Configure `node-gyp` in the current shell:

<!-- cSpell:disable -->
```powershell
$env:PYTHON = (Get-Command python).Source
$env:npm_config_python = $env:PYTHON
$env:GYP_MSVS_VERSION = '2022'
```
<!-- cSpell:enable -->

When Node is not installed globally, prepend its directory to `PATH` and invoke
the repository Yarn release with Node:

```powershell
node .yarn\releases\yarn-4.16.0.cjs --version
```

Prefer the `yarnPath` declared in `.yarnrc.yml` if it changes.

## Dependency Installation

```powershell
node .yarn\releases\yarn-4.16.0.cjs install
```

Expected ARM64-specific behavior:

- `keytar` 7.9.0 can obtain a Windows ARM64 prebuilt module.
- `sqlite3` 5.1.6 has no Windows ARM64 prebuilt module and falls back to
  `node-gyp`.
- `wasm-pack` 0.13.1 rejects Windows ARM64. It belongs to the OneNote converter
  path, which the normal non-CI development build skips.

Confirm SQLite output:

```powershell
Get-Item packages\app-desktop\node_modules\sqlite3\lib\binding\napi-v6-win32-unknown-arm64\node_sqlite3.node
```

If install fails, inspect the Yarn `xfs-*\build.log` paths printed in the
failure output. Distinguish a missing compiler/Python error from an unsupported
optional package.

## Electron Native Rebuild

Derive the ABI for the current Electron release. For the verified Electron
42.3.0 build, use:

```powershell
node .yarn\releases\yarn-4.16.0.cjs workspace @joplin/app-desktop exec electron-rebuild --force-abi 146 --arch arm64
```

The repository's `electronRebuild.js` historically rebuilt Windows ia32 and
x64 modules. Inspect it before using `yarn electronRebuild`; do not assume it
handles ARM64.

## Bundling and Packaging

Bundle:

```powershell
node .yarn\releases\yarn-4.16.0.cjs workspace @joplin/app-desktop exec gulp before-dist
```

For unsigned local builds:

```powershell
$env:CSC_IDENTITY_AUTO_DISCOVERY = 'false'
```

Build a pure ARM64 portable package:

```powershell
node .yarn\releases\yarn-4.16.0.cjs workspace @joplin/app-desktop exec electron-builder --win portable --arm64 --publish never --config.portable.artifactName='JoplinPortable-arm64.${ext}'
```

Build a pure ARM64 installer:

<!-- cSpell:disable -->
```powershell
node .yarn\releases\yarn-4.16.0.cjs workspace @joplin/app-desktop exec electron-builder --win nsis --arm64 --publish never --config.win.artifactName='Joplin-Setup-${version}-arm64.${ext}'
```
<!-- cSpell:enable -->

Do not rely on:

```powershell
electron-builder --win --arm64
```

The target architecture arrays in `packages/app-desktop/package.json` can
still cause x64 and ia32 packages to be built. Explicit target commands produce
pure ARM64 artifacts.

Expected outputs:

```text
packages/app-desktop/dist/Joplin-Setup-<version>-arm64.exe
packages/app-desktop/dist/JoplinPortable-arm64.exe
packages/app-desktop/dist/win-arm64-unpacked/
```

## Architecture Verification

Use this helper for PE files:

```powershell
$path = 'packages\app-desktop\dist\win-arm64-unpacked\Joplin.exe'
$bytes = [System.IO.File]::ReadAllBytes((Resolve-Path $path))
$peOffset = [BitConverter]::ToInt32($bytes, 0x3c)
'{0:X4}' -f [BitConverter]::ToUInt16($bytes, $peOffset + 4)
```

`AA64` means ARM64. Check:

- `Joplin.exe`
- `node_sqlite3.node`
- `keytar.node`

<!-- cSpell:disable -->
Native modules may be stored inside `app.asar`. Use `@electron/asar` to list or
extract them when necessary.
<!-- cSpell:enable -->

Generate hashes:

```powershell
Get-FileHash packages\app-desktop\dist\Joplin-Setup-*-arm64.exe -Algorithm SHA256
Get-FileHash packages\app-desktop\dist\JoplinPortable-arm64.exe -Algorithm SHA256
```

## Smoke Testing

Some agent terminals define `ELECTRON_RUN_AS_NODE=1`, which makes Electron
behave like Node and reject Joplin arguments. Remove it only from the test
process environment:

```powershell
Remove-Item Env:ELECTRON_RUN_AS_NODE -ErrorAction SilentlyContinue
```

Start the unpacked build with an isolated workspace-local profile:

```powershell
packages\app-desktop\dist\win-arm64-unpacked\Joplin.exe --profile "$PWD\.arm64-test-profile" --no-welcome
```

Verify that it remains alive long enough to create profile files and logs.
Close only the test instance. Before recursively deleting the test profile,
resolve its absolute path and verify it remains inside the workspace.

## Known Failure Modes

### Python not found

Set `PYTHON` and `npm_config_python` to a working native Python executable.

### Visual Studio not found

<!-- cSpell:disable -->
Install Build Tools with the ARM64 C++ component and Windows SDK. Confirm the
installation with `vswhere`.
<!-- cSpell:enable -->

### SQLite prebuilt download returns 404

This is expected for `sqlite3` 5.1.6 on Windows ARM64. Ensure source compilation
continues successfully.

### wasm-pack reports unsupported Windows ARM64

Do not misdiagnose this as a desktop runtime failure. Confirm the desktop
dependencies and bundles were produced. OneNote converter development may need
a separate workaround or native Rust tooling.

### sqlite-vec is unavailable

`sqlite-vec` 0.1.9 does not publish a Windows ARM64 package. Joplin catches the
extension load failure and disables vector/semantic search. Normal keyword
search, FTS, note editing, synchronisation, encryption, attachments, and
plugins remain usable.

### App exits with code 9 and rejects Joplin flags

Check `ELECTRON_RUN_AS_NODE`; clear it for the smoke-test process.

### Windows warns about the package

Local artifacts are unsigned unless signing is explicitly configured.
SmartScreen warnings are expected for unsigned builds.
