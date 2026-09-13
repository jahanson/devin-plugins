---
name: install-deno
description: Install the latest stable or LTS Deno runtime, including through npm when deno.land or GitHub is blocked. Use only when the user explicitly asks to install Deno.
argument-hint: "[latest|lts|version]"
triggers:
  - user
---

Install Deno, defaulting to the latest stable release unless the user requests `lts` or an exact version.

When you need the concrete latest stable tag, read [reference/fetch-latest-deno-release-version.md](reference/fetch-latest-deno-release-version.md). It documents cross-platform lookups through `gh release view`, the GitHub releases API, and Deno's `release-latest.txt` endpoint, including PowerShell commands. Each returns a tag shaped `v<major>.<minor>.<patch>`, suitable for comparing an installed version or supplying an exact release. Choose a lookup compatible with the environment's network policy; skip it when the selected package manager can resolve `latest` directly and no comparison is needed.

1. Run `deno --version`. If it succeeds, report the installed version and executable path, then stop unless the user explicitly requested a different version.
2. Choose an installation route in this order:
   - Prefer Deno's official PowerShell or shell installer for the latest stable release.
   - Use npm when the user requests LTS or an exact version, prefers npm, or mentions a restricted provider, network blocklist, npm allowlist, or inaccessible deno.land/GitHub. The official `deno` npm package includes its native binaries as platform-specific `@deno/*` packages, so installation uses the configured npm registry without downloading from deno.land or GitHub.
   - Also support an available platform or version manager when the user requests it. Do not switch package managers silently.
3. Select the exact command, keeping the native scripts preferred:
   - Windows PowerShell: `irm https://deno.land/install.ps1 | iex`
   - macOS or Linux POSIX shell: `curl -fsSL https://deno.land/install.sh | sh`
   - npm, latest: `npm install --global deno@latest`
   - npm, LTS: `npm install --global deno@lts`
   - npm, exact version: `npm install --global deno@<version>`
   - Windows alternatives: `winget install DenoLand.Deno`, `scoop install deno`, or `choco install deno`
   - macOS alternatives: `brew install deno` or `sudo port install deno`
   - Cross-platform version managers: asdf or vfox, using the commands documented in the primary installation reference below
4. Before execution, show the exact command and ask for explicit approval because it installs software and may download executable code. Stop if approval is declined.
5. Run only the approved command. Do not elevate privileges, modify shell profiles, change package-manager configuration, or choose another package manager without separate approval. The npm route supports Windows and macOS on x64/arm64 and glibc Linux on x64/arm64; do not use it on musl Linux or another architecture.
6. Verify with `deno --version` and report the installed version and executable path. For an npm install that is not found on `PATH`, run `npm config get prefix` and report the bin location. A new shell may be required.
7. For future npm-managed upgrades, use `npm install --global deno@latest` or `deno@lts`; do not recommend `deno upgrade`, which may require GitHub access.

Primary installation reference: https://docs.deno.com/runtime/getting_started/installation/
