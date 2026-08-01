---
name: build-joplin-windows-arm64
description: Build, diagnose, package, and verify the Joplin Electron desktop application as native Windows ARM64 installer and portable artifacts. Use when an agent is asked to compile Joplin on Windows ARM64, repair ARM64 native dependency failures, produce ARM64 Electron packages, validate PE architectures, or document/reproduce the Windows ARM64 build.
---

# Build Joplin Windows ARM64

<!-- cSpell:words MSVC asar -->

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
7. Check whether the installed `sqlite-vec` package provides a `win32-arm64`
   loadable extension. If it does not, compile `vec0.dll` from the official
   sqlite-vec amalgamation with MSVC ARM64, verify it exports
   `sqlite3_vec_init`, and include the DLL as an Electron extra resource.
8. Run Electron Rebuild directly with `--arch arm64`. Do not use the existing
   Windows rebuild wrapper without checking its architecture behavior.
9. Bundle with `gulp before-dist`.
10. Build the installer and portable targets separately with explicit ARM64
   arguments and distinct ARM64 artifact names.
11. Verify the unpacked executable and native modules are ARM64 PE files.
12. Smoke-test with an isolated profile after removing
   `ELECTRON_RUN_AS_NODE` from the test process environment.
13. Report artifact paths, sizes, hashes, signing state, smoke-test result,
   and unsupported optional features.

## Guardrails

- Never change or delete the user's normal Joplin profile.
- Never sign or publish artifacts unless explicitly requested and credentials
  are available.
- Do not assume `electron-builder --win --arm64` overrides architecture arrays
  in package configuration. Use an explicit target.
- Do not compile sqlite-vec manually if the current `sqlite-vec` npm release
  already ships a supported Windows ARM64 package. The custom compile step is
  only needed because some releases, such as `sqlite-vec` 0.1.9, do not publish
  `win32-arm64` binaries.
- When a manual sqlite-vec build is needed, do not load the DLL from inside
  `app.asar`. `sqlite3.loadExtension()` needs a real filesystem path, so place
  `vec0.dll` under a packaged resource path such as
  `resources/build/sqlite-vec/vec0.dll`.
- Keep downloaded tools and generated packages untracked unless the user asks
  to commit them.
- Do not use `--no-verify` to bypass repository hooks when committing build
  documentation or support changes.

## Completion Criteria

Finish only after:

- `Joplin.exe` reports PE machine `AA64`.
- Packaged `sqlite3` and `keytar` modules are ARM64 when present.
- If sqlite-vec is bundled manually, packaged `vec0.dll` reports PE machine
  `AA64`, exports `sqlite3_vec_init`, and can answer `select vec_version()`.
- A pure ARM64 installer or portable artifact exists as requested.
- The unpacked app survives a short isolated-profile startup test, or the
  exact startup blocker is captured.
- Remaining limitations and unsigned-build warnings are clearly reported.
