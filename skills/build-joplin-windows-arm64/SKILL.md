---
name: build-joplin-windows-arm64
description: Build, diagnose, package, and verify the Joplin Electron desktop application as native Windows ARM64 installer and portable artifacts. Use when an agent is asked to compile Joplin on Windows ARM64, repair ARM64 native dependency failures, produce ARM64 Electron packages, validate PE architectures, or document/reproduce the Windows ARM64 build.
---

# Build Joplin Windows ARM64

Build from a native Windows ARM64 PowerShell or Command Prompt. Do not use WSL.

## Workflow

1. Read [references/build-procedure.md](references/build-procedure.md).
2. Inspect `package.json`, `packages/app-desktop/package.json`, and
   `packages/app-desktop/tools/electronRebuild.js` before running commands.
   Derive current Node, Yarn, Electron, Electron ABI, and package versions
   rather than assuming the reference values are still current.
3. Check `git status --short` before building. Preserve unrelated user changes.
4. Confirm `process.platform` is `win32` and `process.arch` is `arm64`.
5. Ensure Python, Visual Studio C++ ARM64 tools, and a Windows SDK are
   available before allowing `sqlite3` to fall back to source compilation.
6. Install dependencies. Treat a missing Windows ARM64 `sqlite3` prebuild as
   expected and compile it locally.
7. Run Electron Rebuild directly with `--arch arm64`. Do not use the existing
   Windows rebuild wrapper without checking its architecture behavior.
8. Bundle with `gulp before-dist`.
9. Build the installer and portable targets separately with explicit ARM64
   arguments and distinct ARM64 artifact names.
10. Verify the unpacked executable and native modules are ARM64 PE files.
11. Smoke-test with an isolated profile after removing
   `ELECTRON_RUN_AS_NODE` from the test process environment.
12. Report artifact paths, sizes, hashes, signing state, smoke-test result,
   and unsupported optional features.

## Guardrails

- Never change or delete the user's normal Joplin profile.
- Never sign or publish artifacts unless explicitly requested and credentials
  are available.
- Do not assume `electron-builder --win --arm64` overrides architecture arrays
  in package configuration. Use an explicit target.
- Do not claim full feature parity when `sqlite-vec` has no Windows ARM64
  binary. State that vector/semantic search is disabled while normal FTS
  search and core note features remain available.
- Keep downloaded tools and generated packages untracked unless the user asks
  to commit them.
- Do not use `--no-verify` to bypass repository hooks when committing build
  documentation or support changes.

## Completion Criteria

Finish only after:

- `Joplin.exe` reports PE machine `AA64`.
- Packaged `sqlite3` and `keytar` modules are ARM64 when present.
- A pure ARM64 installer or portable artifact exists as requested.
- The unpacked app survives a short isolated-profile startup test, or the
  exact startup blocker is captured.
- Remaining limitations and unsigned-build warnings are clearly reported.
