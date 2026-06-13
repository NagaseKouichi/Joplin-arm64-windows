# Joplin Windows ARM64 Build Skill

This repository provides instructions and an AI Agent Skill for building the
Joplin desktop application as native Windows ARM64 installer and portable
packages.

It is intended primarily for AI coding agents such as Codex. The Skill gives
an agent the build sequence, native dependency workarounds, packaging commands,
verification steps, and safety constraints needed to complete the build.

This repository is not a fork or copy of the Joplin source code. The agent
builds from the upstream [Joplin repository](https://github.com/laurent22/joplin).

## Use With An AI Agent

Give the agent access to:

```text
skills/build-joplin-windows-arm64/
```

For agents that support Skills, invoke:

```text
$build-joplin-windows-arm64
```

Example prompt:

```text
Use $build-joplin-windows-arm64 to build and verify native Joplin desktop
installer and portable packages on this Windows ARM64 machine.
```

The agent should:

1. Inspect the current Joplin versions and build configuration.
2. Confirm the host and Node.js runtime are native ARM64.
3. Prepare Python, Visual C++ ARM64 tools, and the Windows SDK.
4. Install dependencies and compile missing native ARM64 modules.
5. Rebuild Electron dependencies specifically for ARM64.
6. Create pure ARM64 installer and portable artifacts.
7. Verify PE architectures and run an isolated-profile smoke test.
8. Report hashes, signing state, test results, and remaining limitations.

The Skill intentionally tells the agent to derive current versions from the
Joplin checkout instead of permanently relying on the versions used during the
original successful build.

## Repository Contents

- [`SKILL.md`](skills/build-joplin-windows-arm64/SKILL.md): Main AI Agent
  workflow, guardrails, and completion criteria.
- [`build-procedure.md`](skills/build-joplin-windows-arm64/references/build-procedure.md):
  Detailed commands, verification steps, and failure handling.
- [`build_desktop_windows_arm64.md`](readme/dev/build_desktop_windows_arm64.md):
  Human-readable Windows ARM64 build guide.
- [`BUILD.md`](readme/dev/BUILD.md): Upstream Joplin build documentation with
  a link to the ARM64 guide.

## Prebuilt Preview

An unofficial ARM64 Preview build is available from
[GitHub Releases](https://github.com/NagaseKouichi/Joplin-arm64-windows/releases).

The release contains:

- Native Windows ARM64 installer
- Native Windows ARM64 portable executable
- SHA-256 checksum file

## Known Limitations

- Preview binaries are not code-signed. Windows may display an unknown
  publisher or SmartScreen warning.
- `sqlite-vec` 0.1.9 does not provide a Windows ARM64 binary. AI
  vector/semantic search is disabled.
- Normal keyword search, FTS, note editing, synchronisation, encryption,
  attachments, and plugins remain available.
- The Preview build has not completed the full official Joplin release test
  matrix.

## Status

The documented procedure was successfully used to build Joplin 3.7.1 with
Electron 42.3.0 on Windows 11 ARM64. Future Joplin versions may change Node.js,
Electron ABI, dependencies, or packaging configuration, so the agent must
inspect the checkout before applying version-specific commands.

## Disclaimer

This is an unofficial community project. It is not produced, signed, or
supported by the official Joplin project.
