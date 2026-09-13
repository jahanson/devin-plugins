# Installing Deno via npm — Research for `deno-installer` skill

> **Status:** Facts below are drawn from primary sources fetched 2026 (this session):
> the Deno installation docs page, the live npm registry metadata and the
> `deno@2.9.6` tarball contents, and per-platform `@deno/*` package metadata.
> Inferences and unknowns are labeled as such.

**Primary sources:**

- Deno installation docs: <https://docs.deno.com/runtime/getting_started/installation/>
- npm registry, `deno` package metadata: <https://registry.npmjs.org/deno>
- `deno@2.9.6` tarball (bin.cjs, install.cjs, install_api.cjs, package.json):
  <https://registry.npmjs.org/deno/-/deno-2.9.6.tgz>
- `@deno/win32-x64` metadata (representative platform package):
  <https://registry.npmjs.org/@deno%2Fwin32-x64>
- GitHub releases (referenced by docs for `deno upgrade` and manual zips):
  <https://github.com/denoland/deno/releases>

---

## 1. Is `npm install -g deno` official?

**Yes — fact.** The official installation page lists `npm install -g deno` as a
first-class install method on all three OS tabs (Linux, macOS, Windows), alongside
the shell/PowerShell scripts, Nix, Homebrew, Scoop, Chocolatey, and Winget. The
`deno` package's `repository` field points at `github.com/denoland/deno`, author is
"the Deno authors", license MIT.

## 2. Where do the binaries come from? Does anything contact GitHub or deno.land?

**Fact (from reading `install_api.cjs` and `package.json` in the 2.9.6 tarball):**

- `deno` on npm is a thin wrapper package (`bin.cjs`, `install.cjs`,
  `install_api.cjs`, ~no other code). The actual runtime binaries ship as six
  **npm optional dependencies**, all on the npm registry:
  - `@deno/win32-x64`, `@deno/win32-arm64`
  - `@deno/darwin-x64`, `@deno/darwin-arm64`
  - `@deno/linux-x64-glibc`, `@deno/linux-arm64-glibc`
- The `postinstall` script (`install.cjs` → `install_api.cjs`) resolves the
  platform package already downloaded by npm (`require.resolve("@deno/<target>/package.json")`),
  then **hard-links or copies the executable into the `deno` package folder** and
  chmods it on POSIX. It performs **no network requests** — the binary arrives
  entirely via the npm registry during the normal dependency download.
- `bin.cjs` is a Node wrapper that `spawnSync`s the resolved `deno`/`deno.exe`;
  if the binary is missing it lazily re-runs `runInstall()` — still no network.
- **Conclusion: `npm install -g deno` contacts only the configured npm registry
  (plus whatever registry mirror is configured). It never touches deno.land or
  github.com.** This is the property that matters for allowlisted-npm networks.

## 3. Supported OS/architectures

**Fact:** six platform packages: Windows x64, Windows ARM64, macOS x64, macOS
ARM64, Linux x64 glibc, Linux ARM64 glibc. The docs state Deno runs on macOS,
Linux, Windows on x64 and arm64.

**Fact:** `install_api.cjs#getLinuxFamily()` **hard-fails on musl** (Alpine etc.):
it throws `"Musl is not supported. It's one of our priorities."` pointing at
<https://github.com/denoland/deno/issues/3711>. There is no musl npm package.

**Fact:** other architectures (x86, armv7, riscv64, etc.) throw
`"Unsupported architecture … Only x64 and aarch64 binaries are available."`

**Fact (docs):** Windows requires Windows 10 version 1709 / Server 2016 1709+.

## 4. Package names and current tags

**Fact (live registry):**

- `deno` dist-tags: `latest: 2.9.6`, `lts: 2.2.15`.
- Platform packages (`@deno/<target>`) are version-locked: `deno@2.9.6` pins each
  optional dep to exactly `2.9.6`. They carry the same `latest`/`lts` dist-tags.
- Version pinning works as normal npm: `npm install -g deno@2.9.5`,
  `deno@lts`, `deno@latest`.

## 5. Node/npm requirements

**Fact:** `deno@2.9.6` declares **no `engines` field** and no `os`/`cpu` on the
wrapper package. The platform packages declare `os`/`cpu`, and the Linux packages
additionally declare `libc: ["glibc"]` (confirmed in the registry metadata for
`@deno/linux-x64-glibc@2.9.6` and in `tools/release/npm/build.ts` in
denoland/deno). No package in the set declares `engines`. npm therefore imposes
no Node-version gate, and npm itself filters out the glibc packages on musl
systems.

**Inference:** any Node capable of running CommonJS and `process.report`
(used for musl detection on Linux only) suffices — i.e. effectively any modern
Node. The host needs a working Node+npm before this route is usable at all.

## 6. Global install permissions and PATH

**Fact (install_api.cjs `findBinDir()`):** on global install the postinstall uses
`npm_config_global` + `npm_config_prefix` to locate the real bin dir
(`{prefix}/bin` on POSIX, `{prefix}` itself on Windows) and rewrites the
`deno` shim there. Permissions are therefore exactly npm's global-install
permissions — no sudo beyond what `npm install -g` already needs, no elevation
inside the package.

**Fact (docs, "command not found" section):** for npm installs the docs direct
users to `npm config get prefix` to find the global bin directory. npm's global
bin dir is normally already on PATH wherever npm itself is.

**Fact (install_api.cjs):** postinstall's `replaceBinEntry()` **rewrites the
`node_modules/.bin` (or global bin dir) `deno` shim to invoke the native binary
directly** — `.cmd`/`.ps1` rewrite on Windows, symlink re-point on POSIX — to
avoid Node startup overhead. If it can't verify the shim belongs to this package
it silently falls back to the `bin.cjs` wrapper.

## 7. Performance and behavioral caveats vs the native installer

**Fact (docs):** *"The startup time of the Deno command gets affected if it's
installed via npm. We recommend the official install script (shell or PowerShell)
for better performance."*

**Fact (install.cjs):** this caveat is partially mitigated by design: postinstall
attempts to bypass the Node wrapper entirely by rewriting the bin shim to the
native binary. When that succeeds, per-invocation Node overhead disappears; when
it fails (read-only FS, unverifiable shim, nonstandard layout), every `deno`
invocation pays a Node spawn first.

**Fact:** `install_api.cjs` tolerates read-only filesystems — if copying the exe
into the `deno` package dir fails it falls back to running the binary in place
inside `@deno/<target>`.

**Fact (docs + inference):** `deno upgrade` is documented as fetching the latest
release from `github.com/denoland/deno/releases`. For an npm-installed Deno the
correct upgrade path is `npm install -g deno@<version>` — *inference:* `deno
upgrade` on an npm install would still try GitHub (the binary doesn't know how it
was installed), which fails or is wrong in a GitHub-blocked network. Unknown:
whether recent Deno detects npm provenance and refuses; not verified in source.

**Unknown:** whether `deno install -g` (Deno's own script installer) or the VS
Code extension behaves differently under an npm-managed binary. Not verified.

## 8. Does `npx deno` work?

**Fact:** yes. `bin` is `bin.cjs`, so `npx deno …` runs the wrapper; on first run
with a missing exe it lazily runs `runInstall()` (still npm-only). This gives a
no-global-install usage mode, subject to the Node-startup overhead since the
`.bin` rewrite path is global/local-install oriented.

## 9. Upgrades and version pinning

- **Fact:** upgrade = `npm install -g deno@latest` (or a pinned version, or
  `deno@lts` for the LTS line). Downgrade = same with an older version.
- **Fact:** the `deno` package pins its `@deno/*` optional deps to the identical
  version, so a `deno` version fully determines the binary.
- **Caveat:** `deno upgrade` (the self-updater) pulls from GitHub releases, not
  npm — do not recommend it on npm-installed setups or GitHub-blocked networks.

## 10. Value for npm-allowlisted networks

**Fact + inference:** the entire install path (wrapper + native binary) is
satisfied by the npm registry; no deno.land or github.com access is required at
install or runtime. In environments where `registry.npmjs.org` (or an internal
mirror/proxy such as Artifactory/Nexus) is allowlisted but `deno.land` and
`github.com` are blocked, `npm install -g deno` is the only documented install
route that works without a network exception. Caveats that still break there:
musl distros, non-x64/arm64 arches, and `deno upgrade`.

## Recommended decision order for the `install-deno` skill

1. `deno --version` — already installed? report and stop (unchanged).
2. Prefer the official native installer (`install.ps1` / `install.sh`) per current
   docs, **if** the network can reach deno.land/GitHub — it avoids the npm startup
   caveat and needs no Node.
3. Otherwise, or when Node/npm is already present or deno.land/GitHub are
   blocked: `npm install -g deno` (optionally `deno@<version>`/`@lts` for pinning).
   Note the prerequisites: working npm, x64/arm64, glibc Linux only, Windows
   10 1709+.
4. For a no-global-install probe: `npx deno --version`.
5. Upgrades on npm installs: `npm install -g deno@latest`, never `deno upgrade`.
6. Verify with `deno --version`; if not found, `npm config get prefix` for the
   bin dir (docs' own guidance for npm installs).
