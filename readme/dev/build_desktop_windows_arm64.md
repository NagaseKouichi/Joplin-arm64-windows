# Building the desktop application for Windows ARM64

This document describes how to build native Windows ARM64 installer and
portable packages for the Joplin desktop application.

The procedure was verified on Windows 11 ARM64 with Joplin 3.7.1 and Electron
42.3.0.

## Requirements

Run all commands in a native Windows PowerShell or Command Prompt. Do not use
WSL, and keep the repository in a path without spaces.

Install the following ARM64 tools:

- Node.js 24.12.0
- Python 3
- Visual Studio 2022 Build Tools
- Visual C++ ARM64 build tools
- Windows 11 SDK
- Git

The repository contains its required Yarn release at
`.yarn/releases/yarn-4.16.0.cjs`, so a global Yarn installation is not
required.

Verify that Node is running natively:

```powershell
node -p "process.version + ' ' + process.platform + ' ' + process.arch"
```

The output should end with:

```text
win32 arm64
```

Configure Python and Visual Studio for `node-gyp`:

<!-- cSpell:disable -->
```powershell
$env:PYTHON = (Get-Command python).Source
$env:npm_config_python = $env:PYTHON
$env:GYP_MSVS_VERSION = '2022'
```
<!-- cSpell:enable -->

## Install dependencies

From the repository root, run:

```powershell
node .yarn\releases\yarn-4.16.0.cjs install
```

The `sqlite3` 5.1.6 package does not publish a Windows ARM64 prebuilt binary.
Yarn therefore builds it locally with `node-gyp`, Visual C++, and the Windows
SDK.

The `wasm-pack` 0.13.1 post-install script may fail with:

```text
Unsupported platform: Windows_NT arm64
```

This package belongs to the OneNote converter build. The normal development
build skips that converter, and the failure does not prevent the desktop
application from being bundled once the remaining dependencies have been
installed.

Confirm that the desktop ARM64 SQLite binary exists:

```powershell
Get-Item packages\app-desktop\node_modules\sqlite3\lib\binding\napi-v6-win32-unknown-arm64\node_sqlite3.node
```

## Rebuild Electron native modules

The existing `yarn electronRebuild` task rebuilds the configured Windows x86
architectures. Invoke Electron Rebuild directly to target ARM64:

```powershell
node .yarn\releases\yarn-4.16.0.cjs workspace @joplin/app-desktop exec electron-rebuild --force-abi 146 --arch arm64
```

ABI 146 corresponds to the Electron 42 version currently used by the desktop
package. Recheck this value when Electron is upgraded.

## Bundle the desktop application

```powershell
node .yarn\releases\yarn-4.16.0.cjs workspace @joplin/app-desktop exec gulp before-dist
```

## Build ARM64 packages

Disable automatic certificate discovery when producing an unsigned local
build:

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

Specifying the portable or installer target explicitly is important. Running
only `electron-builder --win --arm64` does not override the architecture
arrays in `packages/app-desktop/package.json`; it can also build the configured
x64 and ia32 packages.

The output files are written to:

```text
packages/app-desktop/dist/Joplin-Setup-<version>-arm64.exe
packages/app-desktop/dist/JoplinPortable-arm64.exe
packages/app-desktop/dist/win-arm64-unpacked/
```

Local builds are unsigned unless `SIGN_APPLICATION=1` and the required signing
credentials are configured. Windows SmartScreen may warn when opening an
unsigned package.

## Verify the build

The following PowerShell snippet prints the PE machine type of the unpacked
application:

```powershell
$path = 'packages\app-desktop\dist\win-arm64-unpacked\Joplin.exe'
$bytes = [System.IO.File]::ReadAllBytes((Resolve-Path $path))
$peOffset = [BitConverter]::ToInt32($bytes, 0x3c)
'{0:X4}' -f [BitConverter]::ToUInt16($bytes, $peOffset + 4)
```

An ARM64 executable reports:

```text
AA64
```

For a smoke test, use a separate profile:

```powershell
Remove-Item Env:ELECTRON_RUN_AS_NODE -ErrorAction SilentlyContinue
packages\app-desktop\dist\win-arm64-unpacked\Joplin.exe --profile "$PWD\.arm64-test-profile" --no-welcome
```

Some automated terminal environments set `ELECTRON_RUN_AS_NODE=1`. Remove it
before testing, otherwise Electron runs as a Node.js command-line process
instead of starting Joplin.

Generate checksums with:

```powershell
Get-FileHash packages\app-desktop\dist\Joplin-Setup-*-arm64.exe -Algorithm SHA256
Get-FileHash packages\app-desktop\dist\JoplinPortable-arm64.exe -Algorithm SHA256
```

## Known limitation

`sqlite-vec` 0.1.9 publishes Windows x64 binaries but no Windows ARM64 binary.
Joplin handles failure to load this extension by disabling vector search.
Regular note editing, storage, search, synchronisation, and encryption remain
available.
