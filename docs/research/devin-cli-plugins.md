# Devin CLI Plugins — Research for Marketplace Development

> **Status:** Plugins are **in beta**; behavior subject to change.
> ([plugin-template README](https://github.com/CognitionAI/plugin-template))
>
> **Source note:** Doc pages are cited as `docs.devinenterprise.com` (the mirror the
> research agent could reach); the canonical URL is `https://docs.devin.ai/<same-path>`.
> The four core pages — `/product-guides/plugins`, `/product-guides/plugin-ecosystem`,
> `/cli/extensibility/plugins/overview`, `/cli/extensibility/plugins/quickstart` — were
> independently fetched from canonical `docs.devin.ai` and confirmed to carry the same
> content quoted here.
>
> **CLI verification:** `devin plugins` command syntax below was confirmed against the
> installed binary (`devin 3000.10.21`, Windows): subcommands are `install`, `list`,
> `info`, `update`, `remove`, `prune` — there is **no** `marketplace` subcommand — and
> `install` accepts `<SOURCE>` (`owner/repo`, git URL, or local path, with
> `#path/to/plugin` for subfolder plugins) plus `-y/--yes` and `--local`.

---

## 1. TL;DR for a marketplace builder

- A Devin plugin is a directory with `.devin-plugin/plugin.json`; everything else is optional.
  It can ship **skills** (`skills/<name>/SKILL.md`), **rules** (`AGENTS.md` + `rules/*.md`),
  **custom subagents** (`agents/`), **lifecycle hooks** (`hooks.json`), **MCP servers**
  (manifest `mcpServers` field or `.mcp.json` / `mcp_config.json`), and a **dependency/policy**
  triple (`requiredPlugins` / `optionalPlugins` / `forbiddenPlugins`).
- A "marketplace" is not a special format: **a marketplace repo is itself a plugin** — the
  *meta-plugin* pattern. Its root `.devin-plugin/plugin.json` lists other plugins in
  `requiredPlugins`/`optionalPlugins`. The official example is
  [`CognitionAI/devin-marketplace`](https://github.com/CognitionAI/devin-marketplace), whose root
  manifest lists ~160 entries in `optionalPlugins`, each either `"./plugins/<slug>"` (a plugin
  authored in the repo) or a pinned `{ "source": "git-subdir" | "url", "url", "path"?, "sha" }`
  object pointing at a third-party repo.
- There is **no separate "marketplace add" CLI command**. Users install plugins (including a
  meta-plugin repo) with `devin plugins install <source>`; org/enterprise distribution happens
  through managed manifests edited in the Devin web app (Customize → Plugins).
- Installed plugins are cached under `%APPDATA%\devin\cli\plugins\cache\<sanitized-source>\<version>\`
  with `lock.json` + `discovered.json` at `cli\plugins\` tracking specs, resolved SHAs, and versions.

---

## 2. Plugin anatomy

### Directory layout (official)

```
my-plugin/
├── .devin-plugin/
│   └── plugin.json     # The plugin manifest (only required file; only `name` required in it)
├── AGENTS.md           # Optional always-on rule
├── rules/              # Optional triggered rules (*.md with trigger frontmatter)
├── agents/
│   └── reviewer.md     # Optional custom subagent (agents/reviewer/AGENT.md also works)
├── hooks.json          # Optional lifecycle hooks
├── .mcp.json           # Optional MCP servers (see §5.4 — mcp_config.json also observed)
└── skills/
    └── review/
        └── SKILL.md    # An ordinary skill
```

Sources: [CLI plugins reference](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md),
[plugin-ecosystem guide](https://docs.devinenterprise.com/product-guides/plugin-ecosystem.md).
Verified against the installed `ponytail` plugin and the `kitchen-sink` template
([tree](https://github.com/CognitionAI/plugin-template/tree/main/plugins/kitchen-sink)).

"One repo (or one `git-subdir` subfolder) is one plugin. A single repo can host many plugins as
subfolders, each referenced with its own `git-subdir` source." —
[plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md)

### Manifest: `.devin-plugin/plugin.json`

Only `name` is required; it must be unique among installed plugins and becomes the
`/<name>:…` slash-command namespace. Names are lowercase alphanumeric with single `-` or `.`
separators (`review-tools`, `acme.tools`). —
[plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md)

Full documented field list:

| Field | Notes |
|---|---|
| `name` | **Required.** Unique; identity + `/<name>:` namespace. Regex per marketplace validator: `^[a-z0-9]+([.-][a-z0-9]+)*$`; the plugin-template validator uses stricter kebab-case `^[a-z0-9]+(-[a-z0-9]+)*$`. |
| `version` | Semver; plugin-template CI enforces `/^\d+\.\d+\.\d+([-+].*)?$/`. Used as the on-disk cache `version_dir` (see §7). |
| `description` | Shown by `devin plugins info` and in the Customize catalog. |
| `author` | Object `{ "name": "…", "email": "…" }` (docs). ponytail uses `{ "name", "url" }` — both shapes observed. |
| `homepage` | URL. |
| `repository` | URL. |
| `license` | e.g. `"MIT"`. |
| `keywords` | `string[]`; marketplace plugins use it for catalog categories (e.g. `"Essentials"`, `"Monitoring & Analytics"`, `"Development Tools"`). |
| `displayName` | Human title for cards, e.g. `"Context7"`. Used by all official marketplace plugins. |
| `logo` | Repo-relative path (`logo.svg`, or `png/jpg/jpeg/webp`, subpaths allowed) **or** an `https://` URL. Official guidance: prefer a checked-in file so cards never load from a third-party host. |
| `skills` | Path or `string[]` replacing the default `skills/` dir; `[]` disables skill loading. Paths must stay inside the plugin. |
| `mcpServers` | Inline server map, path(s) to declaration files, or `{ "paths": [...], "exclusive": true }` — see §5.4. |
| `requiredPlugins` | Auto-installed recursively on install. Failure of a required dep fails the whole install. |
| `optionalPlugins` | Allow-list (not auto-installed); carves exceptions out of `forbiddenPlugins`. |
| `forbiddenPlugins` | Deny-list of identities / globs (`acme/*`, `*/secrets`, `"*"` = lockdown). |

Complete real example — `kitchen-sink` plugin
([source](https://github.com/CognitionAI/plugin-template/blob/main/plugins/kitchen-sink/.devin-plugin/plugin.json)):

```json
{
  "name": "kitchen-sink",
  "version": "0.1.0",
  "description": "Every plugin capability in one place: rules, skills, a subagent, hooks, MCP, and governance",
  "author": { "name": "Your Org", "email": "plugins@example.com" },
  "license": "MIT",
  "keywords": ["devin", "plugin", "governance", "hooks", "mcp"],
  "requiredPlugins": [
    {
      "source": "git-subdir",
      "url": "https://github.com/usacognition/plugin-template.git",
      "path": "plugins/hello-world"
    }
  ],
  "optionalPlugins": ["acme/deploy-tools"],
  "forbiddenPlugins": ["sketchy-org/*"]
}
```

Minimal real example — `hello-world`
([source](https://github.com/CognitionAI/plugin-template/blob/main/plugins/hello-world/.devin-plugin/plugin.json)):

```json
{
  "name": "hello-world",
  "version": "0.1.0",
  "description": "The smallest useful Devin plugin: one always-on rule and one skill"
}
```

MCP-catalog example — `context7` from the official marketplace
([source](https://github.com/CognitionAI/devin-marketplace/blob/main/plugins/context7/.devin-plugin/plugin.json)):

```json
{
  "name": "context7",
  "displayName": "Context7",
  "description": "Access up-to-date documentation for 1000+ npm packages and frameworks",
  "homepage": "https://upstash.com",
  "repository": "https://github.com/upstash/context7",
  "logo": "logo.svg",
  "keywords": ["Development Tools"],
  "mcpServers": {
    "context7": { "url": "https://mcp.context7.com/mcp" }
  }
}
```

### Compatible (fallback) formats

If `.devin-plugin/plugin.json` is absent, Devin loads other ecosystems' plugin layouts.
Precedence: **`.devin-plugin/plugin.json` > `.claude-plugin/plugin.json` > root `plugin.json`**. —
[plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md)

- **Claude plugins**: root `.mcp.json` and manifest `mcpServers` honored;
  `${CLAUDE_PLUGIN_ROOT}` expands to the plugin root in server configs.
- **Agent Plugins 1.0.0** ([spec](https://github.com/agentplugins/agent-plugins-spec)):
  root `plugin.json`, MCP in root `mcp.json` (read after `.mcp.json`; `.mcp.json` wins on
  name collision — and legacy-layout plugins never read `mcp.json`), `skills/` dir.
  Extra runtime conventions *only* for this layout:
  - `type` field on MCP entries (`stdio` | `streamable-http` | `sse`) accepted instead of `transport`.
  - `${PLUGIN_ROOT}` expands to the plugin root (like `${CLAUDE_PLUGIN_ROOT}`).
  - `${PLUGIN_DATA}` in `args`/`env`/`cwd` → persistent per-plugin writable data dir, keyed by
    plugin identity (survives updates, deleted on uninstall).
  - stdio servers get `PLUGIN_ROOT` + `PLUGIN_DATA` env vars; `cwd` defaults to plugin root;
    `./`-prefixed `command` resolves inside the plugin root (validated for containment).
  - Unknown `$schema` version → warn, load best-effort.

Real-world proof: the installed `ponytail` plugin carries *both* `.devin-plugin/plugin.json`
and `.claude-plugin/plugin.json` (plus `.codex-plugin/`, `.qoder-plugin/`, `.github/plugin/`,
`.grok-plugin/`, `.agents/plugins/marketplace.json`, `gemini-extension.json`, root `plugin.json`
and `plugin.yaml` for other hosts) — one repo, many manifests.
Local path: `C:\Users\jahanson\AppData\Roaming\devin\cli\plugins\cache\github.com_DietrichGebert_ponytail-38c3408d\4.9.0\`

---

## 3. Install & management commands (CLI)

From the [plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md)
and [quickstart](https://docs.devinenterprise.com/cli/extensibility/plugins/quickstart.md):

```bash
# Sources
devin plugins install acme/review-tools                       # GitHub owner/repo
devin plugins install acme/plugins#plugins/review             # plugin in a repo subfolder (#fragment)
devin plugins install https://gitlab.com/acme/review-tools.git  # any git host
devin plugins install ./my-plugin                             # local folder (linked; for authoring)
devin plugins install --local ./my-plugin                     # this machine only, not synced to Cloud

devin plugins install <source> -y    # or --yes: skip the pre-install summary prompt

# Managing
devin plugins list                    # installed plugins, versions, policy-blocked status
devin plugins info review-tools       # skills, hooks, rules, required/optional/forbidden lists
devin plugins update review-tools     # re-fetch at latest version
devin plugins update                  # re-fetch all
devin plugins remove review-tools     # remove from personal plugins (auto-installed deps stay)
devin plugins remove review-tools --local   # machine-only removal
devin plugins remove review-tools --force   # remove even if still required by governance
devin plugins prune                   # drop requirements from deleted repos; GC unused content
```

Behavior details (all from the overview page):

- Before installing, Devin **shows what the plugin adds** (skills, auto-installed required
  plugins, policy like forbids); `-y`/`--yes` skips the prompt.
- Installs are **user-level**, synced to your personal manifest in Devin Cloud → they load on
  every signed-in machine and in cloud sessions. `--local` opts out.
- Requires sign-in (`devin auth login`). An enterprise can disable CLI plugins entirely.
- Local-folder installs are **linked**: edits apply next session, no `update` needed.
- `remove --force`: a governance-required plugin is **reinstalled next session** in the
  requiring scope; the CLI warns which level still requires it.
- A **name collision** with an installed plugin refuses the install.

Third-party confirmation — ponytail README documents `devin plugins install DietrichGebert/ponytail`
and `devin plugins remove ponytail`, with skills surfaced as `/ponytail:ponytail`, etc.
(`…\cache\github.com_DietrichGebert_ponytail-38c3408d\4.9.0\README.md`, "Devin CLI" section).

---

## 4. Dependency / policy lists — the marketplace language

Every manifest (managed manifest, repo `.devin/config.json`, or plugin manifest) uses the same
three lists. A dependency entry is a **source** — string shorthand or object:

| Form | Meaning |
|---|---|
| `"owner/repo"` | GitHub repository |
| `"https://…"`, `"git@…"`, `"ssh://…"` | any git URL |
| `{ "source": "github", "repo": "owner/repo" }` | GitHub, object form |
| `{ "source": "url", "url": "https://gitlab.com/team/plugin.git" }` | git URL, object form |
| `{ "source": "git-subdir", "url": "…", "path": "sub/dir" }` | plugin in a subfolder of a shared repo |
| `{ "source": "github", "repo": "owner/repo", "sha": "3f2a9c1…" }` | **pinned** to an exact commit |
| `{ "source": "github", "repo": "owner/repo", "ref": "v2" }` | **tracks** a branch/tag — re-resolved on every refresh |

`sha` and `ref` work with every object form and are mutually exclusive; neither → tracks the
repo's default branch. All GitHub spellings of a repo (`owner/repo`, HTTPS, `.git`, SSH) are
the **same plugin identity**. A local path is also a valid identity (for forbids). —
[plugins overview §Dependencies](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md)

Additional observed source type (local `lock.json`): `account-upload` — the uploaded-`.zip` flow:

```json
{ "source": "account-upload", "bundleId": "default", "bundleScope": "user" }
```

Source: `C:\Users\jahanson\AppData\Roaming\devin\cli\plugins\lock.json` (and `discovered.json`
nodes with `"resolved": { "kind": "accountUpload", "content_hash": "…" }`).

**`env` on dependency entries** (managed manifests, for cloud sessions): an entry may carry
`"env": { "NAME": "value" }` to configure command hooks in cloud sessions; values may be
`secret:org:NAME` / `secret:enterprise:NAME` / `secret:personal:NAME` / `secret:session:NAME` /
`secret:repo:owner/repo:NAME` (append `/ENTRY` for key-value secrets). —
[plugins guide §Plugin environment variables](https://docs.devinenterprise.com/product-guides/plugins.md)

### Governance semantics (deny-wins)

- **Deny wins** across all active manifests and installed plugins.
- **Self-override**: a manifest's own required/optional entries (and a plugin itself) are exempt
  from *its own* forbids → `"forbiddenPlugins": ["*"], "optionalPlugins": ["acme/approved"]`
  = "allow only what's listed". Covers only *directly listed* entries — transitive deps aren't
  exempt under lockdown.
- **No cross-scope re-permitting**: one level's allow-list cannot re-permit what another forbids.
- Enforcement: **install time** (blocked plugin / unsatisfiable required dep / name collision →
  refused) and **load time** (blocked-after-install stays on disk; skills skipped at session
  start with a warning naming the forbidder).
- **Managed manifest fetch failure fails open**: that level's plugins aren't installed and its
  forbids aren't enforced for that session.
- **Pin conflicts** (two manifests pinning different SHAs): resolve by aligning pins or deferring
  to the higher-authority level; Customize flags them.

### Authority levels (highest → lowest)

1. **Enterprise** — account-wide managed manifest.
2. **Org** — below its enterprise; can add but not overrule. Cloud sessions use the session's
   org manifest; CLI/Desktop use the user's *primary* org's manifest.
3. **Repo** — `requiredPlugins`/`optionalPlugins`/`forbiddenPlugins` in a checkout's
   `.devin/config.json`, discovered by walking up from the working directory.
4. **User** — personal installs (`devin plugins install`, synced via personal manifest) or
   `--local` on one machine.

Higher authority wins: a lower level can never re-permit a higher forbid, nor forbid a higher
require. Local `discovered.json` corroborates the scope model with `"scope": "managed"` origins
carrying `account_id`/`org_id`/`user_id` and `"scope": "repo"` origins carrying a `root` path.
(`C:\Users\jahanson\AppData\Roaming\devin\cli\plugins\discovered.json`)

---

## 5. What a plugin can contribute

### 5.1 Skills — `skills/<name>/SKILL.md`

Ordinary Devin skills; plugins introduce no new format. Directory name is the skill id;
exposed as `/<plugin>:<skill>` slash commands. The manifest `skills` field can point elsewhere
(string or array; `[]` disables). —
[creating-skills](https://docs.devinenterprise.com/cli/extensibility/skills/creating-skills.md),
[plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md)

**Frontmatter fields (documented):**

| Field | Type | Default | Purpose |
|---|---|---|---|
| `name` | string | directory name | Display name / invocation name |
| `description` | string | — | Shown in slash completions; also the model's trigger cue |
| `argument-hint` | string | — | e.g. `"[file] [options]"` |
| `model` | string | current model | Per-skill model override (same values as `--model`) |
| `subagent` | bool | `false` | Run the skill as a subagent (experimental) |
| `agent` | string | — | Run as subagent with a specific profile (wins over `subagent`) |
| `allowed-tools` | list | all tools | e.g. `read`, `grep`, `glob`, `exec`, `mcp__github__list_issues` |
| `permissions` | object | inherit | `{ allow: [], deny: [], ask: [] }` — additive, can't exceed higher-level denies |
| `triggers` | list | `[user, model]` | Invocation channels |

**Observed but undocumented field:** `disable-model-invocation: true` in the uploaded bundle's
`skills/handoff/SKILL.md`
(`C:\Users\jahanson\AppData\Roaming\devin\cli\plugins\cache\account-upload_user_default-eccf859c\0.1.0\skills\handoff\SKILL.md`) — likely mirrors the Claude-skills convention; treated as real by that bundle.

Real examples: `triggers: [user]` + `argument-hint` (account-upload `handoff`); `triggers: [user]`
(account-upload `show-me`); `description: >` multi-line + `argument-hint` + `license` (ponytail);
`name` + `description` only (context7-mcp in devin-marketplace).

**Subagent-skill semantics:** `subagent: true` → default `subagent_general` profile; `agent:
<profile>` → a named profile; skill body becomes the task. No nested subagents (inline instead);
orchestration is always one level deep.

### 5.2 Rules — `AGENTS.md` + `rules/*.md`

- `AGENTS.md` at the plugin root = always-on rule injected every session, alongside the user's
  own rules. Keep it short — it costs context for everyone.
- `rules/*.md` = triggered rules with the same frontmatter as Windsurf rules:
  `trigger: always_on | manual | model_decision | agent | glob` (+ `description`, `globs` for
  glob rules). Manual rules load only when explicitly referenced (e.g. @-mention).
- Verified: ponytail's `AGENTS.md` is in fact injected (it appeared as a session rule while
  researching). Template example: `rules/release-checklist.md` with `trigger: manual`
  ([source](https://github.com/CognitionAI/plugin-template/blob/main/plugins/kitchen-sink/rules/release-checklist.md)).
- Sources: [rules](https://docs.devinenterprise.com/cli/extensibility/rules.md),
  [plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md).

### 5.3 Custom subagents — `agents/`

- `agents/<name>.md` (flat) **or** `agents/<name>/AGENT.md` (directory; `AGENTS.md`,
  `agent.md`, `agents.md` also accepted; `AGENT.md` wins if several exist). `name:` in
  frontmatter overrides the path-derived id. Available as `<plugin>:<name>`.
- Frontmatter: `name`, `description`, `model` (default: the *default subagent model*, i.e.
  SWE-1.6 via router — **not** the parent's model), `allowed-tools` (alias `tools` accepted;
  `ask_user_question` always withheld), `max-nesting` (opt-in nested spawning).
- **CLI/Desktop only** — plugin subagents do not load in cloud sessions.
- Example: [kitchen-sink `agents/test-writer/AGENT.md`](https://github.com/CognitionAI/plugin-template/blob/main/plugins/kitchen-sink/agents/test-writer/AGENT.md)
  (`name` + `description` frontmatter, system-prompt body).
- Source: [subagents](https://docs.devinenterprise.com/cli/subagents.md).

### 5.4 MCP servers

Three declaration mechanisms (first wins on server-name collision):

1. **`mcpServers` field in `.devin-plugin/plugin.json`** — four shapes:
   ```jsonc
   { "mcpServers": "config/mcp.json" }                                  // one declaration file
   { "mcpServers": ["config/mcp.json", "config/extra.json"] }           // several, in order
   { "mcpServers": { "paths": ["config/mcp.json"], "exclusive": true } } // suppress root convention
   { "mcpServers": { "linear": { "command": "npx", "args": ["-y", "linear-mcp"] } } } // inline map
   ```
   Declared paths must stay inside the plugin; unsafe entries are dropped (unlike `skills`,
   an invalid `mcpServers` only disables MCP loading). Empty array/empty map leaves the root
   convention enabled.
2. **`.mcp.json` at plugin root** — the conventional file (Claude-plugin convention, honored
   for Devin plugins too).
3. **`mcp.json` at plugin root** — only for Agent-Plugins-layout plugins (root `plugin.json`).

**Filename conflict to note:** the docs' anatomy tree says `.mcp.json`, but the kitchen-sink
template ships **`mcp_config.json`** with `{"mcpServers": {"deepwiki": {"serverUrl": "https://mcp.deepwiki.com/mcp"}}}`
([source](https://github.com/CognitionAI/plugin-template/blob/main/plugins/kitchen-sink/mcp_config.json))
— `mcp_config.json` is the Devin CLI's own MCP config filename (`.devin/mcp_config.json` in
projects), so a plugin-root `mcp_config.json` is very likely also read. UNVERIFIED which exact
filename(s) are conventional for Devin-layout plugins beyond `.mcp.json`; also note the template
uses `serverUrl` (Windsurf-style key) rather than the documented `url`.

**Server config fields:**

- stdio: `command`, `args` (list), `env` (object). (Marketplace validator enforces exactly
  these + optional `description`.)
- HTTP: `url` (must be https), `headers` (object), `transport` (`"http"` default w/ SSE fallback
  on 4xx, or `"sse"`), `oauthClientId`, `oauthScopes`. Docs for user config also list
  `oauthClientSecret` and `oauthResource`, but **plugins may not carry a client secret — a config
  with one is rejected at activation**.
- Agent-Plugins layout may use `type` (`stdio`/`streamable-http`/`sse`) and `cwd`.

**Credentials/secrets:** literal secret values in the config are **stripped**; reference them as
`${NAME}` (resolved from credentials saved on the user's installation). `${NAME:-default}`
fallback syntax is supported. The `MCP_` prefix is retired (validator errors on it). Runtime
placeholders `${CLAUDE_PLUGIN_ROOT}` / `${PLUGIN_ROOT}` expand to the plugin root.
For stdio `env`, a saved credential replaces the env entry with the same key — so self-reference:
`"AWS_PROFILE": "${AWS_PROFILE}"`.
Sources: [devin-marketplace `.agents/skills/adding-a-plugin/SKILL.md`](https://github.com/CognitionAI/devin-marketplace/blob/main/.agents/skills/adding-a-plugin/SKILL.md),
[scripts/validate.py](https://github.com/CognitionAI/devin-marketplace/blob/main/scripts/validate.py),
[plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md),
[MCP configuration](https://docs.devinenterprise.com/cli/extensibility/mcp/configuration.md).

Real stdio example — `aws-core`
([source](https://github.com/CognitionAI/devin-marketplace/blob/main/plugins/aws-core/.devin-plugin/plugin.json)):

```json
"mcpServers": {
  "aws-core": {
    "command": "uvx",
    "args": ["awslabs.core-mcp-server@latest"],
    "env": { "AWS_PROFILE": "${AWS_PROFILE}", "AWS_REGION": "${AWS_REGION}" }
  }
}
```

OAuth: CLI users run `devin mcp login <server>` for a plugin's OAuth server; cloud sessions use
the connection made in the web app (Customize → MCPs). MCP-prompt servers contribute
`/mcp__<server>__<prompt>` slash commands. Plugin-served MCP loads in-session but doesn't yet
appear in the MCP settings UI (per plugin-ecosystem "current limitations").

### 5.5 Hooks — `hooks.json` at plugin root

Registers lifecycle hooks in every session where the plugin is installed. **Best effort, fail
open** — a broken hook doesn't stop the session. —
[plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md),
[hooks overview](https://docs.devinenterprise.com/cli/extensibility/hooks/overview.md)

Events: `PreToolUse`, `PostToolUse`, `PermissionRequest`, `UserPromptSubmit`, `Stop`,
`PostCompaction`, `SessionStart`, `SessionEnd`. (Same set observed in user
`%APPDATA%\devin\config.json`.)

Format (same JSON shape as `.devin/hooks.v1.json` and Claude `settings.json` `hooks`):

```json
{
  "PreToolUse": [
    {
      "matcher": "exec",
      "hooks": [{ "type": "command", "command": "./scripts/validate.sh", "timeout": 10 }]
    }
  ]
}
```

- `matcher`: regex on `tool_name` (tool events only); empty/omitted = all.
- `type`: `"command"` (JSON in on stdin, JSON control object on stdout, `DEVIN_PROJECT_DIR`
  env set) or `"prompt"` (LLM-evaluated — **CLI/local-only**, not cloud).
- `timeout` in seconds. (Claude-format hooks may also carry `statusMessage`, seen in ponytail's
  `hooks/claude-codex-hooks.json`.)
- Exit codes: `0` ok, `2` block, other = non-blocking error.
- stdout control object: `{"decision": "approve"|"block", "reason": …}`,
  `{"hookSpecificOutput": {"hookEventName": …, "additionalContext": …}}` (context injection for
  `UserPromptSubmit`, `SessionStart`, `PostToolUse`), `{"hookSpecificOutput":
  {"hookEventName": "PreToolUse", "updatedInput": {…}}}` (rewrite tool args).
- stdin payload: `hook_event_name`, `session_id` (stable per session), `prompt_id` (rotates per
  user turn; absent pre-first-prompt), plus per-event fields (`tool_name`, `tool_input`,
  `tool_response`, `prompt`, `stop_hook_active`, `summary`, `source`, `reason`).
- Cloud-session limits: `command` hooks run on the session machine and support every event
  **except `SessionStart`/`SessionEnd`**; `prompt` hooks are CLI/local-only.
- `/hooks` slash command lists registered hooks.
- Template example: [kitchen-sink `hooks.json`](https://github.com/CognitionAI/plugin-template/blob/main/plugins/kitchen-sink/hooks.json).

---

## 6. Marketplace mechanics

### The official marketplace repo — `CognitionAI/devin-marketplace`

The repo **is itself a plugin**; its root `.devin-plugin/plugin.json` is the catalog index —
"that file is the only thing consumers read"
([adding-a-plugin SKILL.md](https://github.com/CognitionAI/devin-marketplace/blob/main/.agents/skills/adding-a-plugin/SKILL.md)).

Root manifest (abridged; full file: [.devin-plugin/plugin.json](https://github.com/CognitionAI/devin-marketplace/blob/main/.devin-plugin/plugin.json)):

```json
{
  "name": "devin-marketplace",
  "description": "Devin's plugin marketplace: a plugin for every MCP server Devin hosts, plus third-party plugins pinned to reviewed commits.",
  "homepage": "https://github.com/CognitionAI/devin-marketplace",
  "repository": "https://github.com/CognitionAI/devin-marketplace",
  "skills": [],
  "optionalPlugins": [
    "./plugins/airtable",
    "./plugins/notion",
    "./plugins/context7",
    {
      "source": "git-subdir",
      "url": "https://github.com/databricks/databricks-agent-skills.git",
      "path": "plugins/databricks/claude",
      "sha": "f2e82274b10d0769016afdf0520f0010c6981198"
    }
  ]
}
```

Entry forms in `optionalPlugins` (enforced by `scripts/validate.py`):

- `"./plugins/<slug>"` — a plugin authored in-repo; must contain `.devin-plugin/plugin.json`
  whose `name` equals `<slug>`.
- `{ "source": "url", "url": "https://host/owner/repo.git", "sha": "<40-hex>" }` — whole upstream repo.
- `{ "source": "git-subdir", "url": "…", "path": "dir/in/repo", "sha": "<40-hex>" }` — plugin in
  an upstream subdir. `path` required for `git-subdir`, forbidden otherwise; no leading `/`, no `..`.
- Always pin a full 40-hex sha — never a branch/tag. No display metadata on upstream entries
  (the plugin's own manifest supplies it).
- Entries sorted by identity `(url, path)`, each identity once; `python3 scripts/validate.py
  --fix` canonicalizes, `--fetch` confirms upstream SHAs resolve; CI runs it.

In-repo plugin requirements (same validator): manifest keys restricted to `name`, `displayName`,
`description`, `homepage`, `repository`, `logo`, `keywords`, `mcpServers`; `name` must equal the
dir slug; `displayName` single-line, no stray whitespace; `logo` repo-relative file that exists
(or https URL); `mcpServers` must declare **exactly one server named `<slug>`**.

Repo layout: `plugins/<slug>/{.devin-plugin/plugin.json, logo.svg, skills/<slug>-mcp/SKILL.md}`,
plus `.agents/skills/adding-a-plugin/SKILL.md` (contributor guide) and `.github/workflows`
(validate on PR).

Consumers reference marketplace plugins pinned by commit, per the repo README:

```json
{
  "requiredPlugins": [
    {
      "source": "git-subdir",
      "url": "https://github.com/CognitionAI/devin-marketplace.git",
      "path": "plugins/notion",
      "sha": "<commit>"
    }
  ]
}
```

### The meta-plugin / team-marketplace pattern

`CognitionAI/team-marketplace-template` is the canonical team marketplace: root manifest
`team-starter-pack` with `requiredPlugins` (git-subdir into itself, unpinned), `optionalPlugins`
(including a third-party `git-subdir` into `anthropics/claude-code`), and `forbiddenPlugins`.
Distribution: an admin adds one `requiredPlugins` entry to the managed manifest in
**Customize → Plugins → Add plugin → From repository**. —
[team-marketplace-template](https://github.com/CognitionAI/team-marketplace-template),
[quickstart](https://docs.devinenterprise.com/cli/extensibility/plugins/quickstart.md)

### Web app (Customize) mechanics

- **Customize** page (`app.devin.ai/customize`, replaced Settings→Plugins/Marketplace):
  tabs = Plugins, Skills, MCPs, Hooks, Rules; scope tabs = Personal / Organization / Enterprise
  (each shows *effective* content after governance).
- **Browse marketplace** = official catalog + org/enterprise-added plugins; install menu offers
  every writable scope; first install shows a security notice; official plugins are marked.
- **Add plugin** supports: From repository (`owner/repo` or git URL + optional subdir, private
  repos via Git integration), Upload `.zip` (stored with the scope — this is the `account-upload`
  source in the CLI's lock file), Create plugin (in-app editor with version history).
- **Indexing**: Devin clones each plugin source, reads manifests, resolves deps, applies
  governance; auto-queued on changes; **Reindex plugins** in Plugin settings forces a refresh;
  private-repo content is *withheld* in orgs that can't reach the repo (forbids still apply);
  indexing inherits the org's session network policy.
- **Managed manifest** per scope is one JSON document with `requiredPlugins` / `optionalPlugins`
  / `forbiddenPlugins`; stored verbatim, validated at install time.
- Pinning UI: unpinned entries flagged **"Plugin source not pinned"**; `sha` pins exact content.
- Standalone accounts: one account manifest (labelled "Organization" in Customize) + personal.
- Enterprise admins can hide the official marketplace and control marketplace-MCP availability
  per org; enterprise can disable CLI plugins.
- Source: [plugins guide](https://docs.devinenterprise.com/product-guides/plugins.md).

### On-disk cache (this machine, verified)

`%APPDATA%\devin\cli\plugins\`:

- `cache\<sanitized-source>\<version_dir>\` — full plugin contents.
  e.g. `cache\github.com_DietrichGebert_ponytail-38c3408d\4.9.0\` (source =
  `host_owner_repo-<hash8>`, version dir = manifest `version`) and
  `cache\account-upload_user_default-eccf859c\0.1.0\`. `.tmpXXXXXX` dirs = in-progress fetches.
- `lock.json` — resolved state: `requirements[]` (`spec`, `origin`, `required`), `resolved[]`
  (`name`, `spec`, `identity` — e.g. `https://github.com/DietrichGebert/ponytail` or
  `account-upload:user:default`, `resolved: {"kind": "git", "sha"} or {"kind": "accountUpload",
  "content_hash"}`, `fetched_at`, `version_dir`, `refresh_failure` diagnostics).
- `discovered.json` — per-origin walk results: origins keyed by `{scope: "managed", account_id,
  org_id?, user_id?}` or `{scope: "repo", root}`; `declaration_hash`, `roots[]`, `reached[]`,
  `nodes[]` (identity/name/spec/resolved/fetched_at/assets_status).
- `discovered/` dir exists (empty on this machine).

### Update semantics

- Manifest changes apply to the **next session**; running sessions keep what they loaded.
- Content changes: cloud fetches at session start; CLI refreshes via `devin plugins update`;
  Customize shows new content after reindex. Pinned `sha` never changes until edited; `ref`
  re-resolves each refresh; unpinned tracks default branch ("merging to the default branch **is**
  the release").
- Uploaded plugins are always pinned to uploaded content; editable from their details sheet
  (version history per save); deleting removes files permanently.

---

## 7. Distribution & versioning

- **Sources:** GitHub `owner/repo`, any git URL (`https`, `git@`, `ssh://`), `#sub/dir` fragment
  or `git-subdir` source for monorepo plugins, local folder (linked), account upload (`.zip`).
- **Private repos:** work as-is — cloud fetches through the org's Git integration, CLI users
  fetch with their own git credentials (need repo access).
- **Monorepo vs one-per-repo:** both first-class. Official marketplace = monorepo
  (`plugins/<slug>/`); team-marketplace-template = meta-plugin monorepo; ponytail =
  one-plugin-per-repo (manifest at root).
- **Versioning:** `version` in the manifest is semver (template CI enforces
  `X.Y.Z[-+…]`) and becomes the cache directory name. Resolution identity for git sources is the
  commit SHA (stored in `lock.json` `resolved.sha`); `sha` pins, `ref` tracks, default = default
  branch. There is no registry-side version index — "latest" = whatever the tracked ref resolves
  to at fetch time. GitHub release tags are optional (`v*` tags are a common convention, e.g.
  ponytail's publish workflow).
- **Identity:** all GitHub URL spellings of one repo are the same identity; identity for
  `git-subdir` is `(url, path)`; uploads are `account-upload:<scope>:<bundleId>`.

---

## 8. CLI vs Desktop vs Cloud

Same plugin format everywhere; surface differences:

| Capability | Cloud sessions | Devin CLI | Devin Desktop |
|---|---|---|---|
| Skills, rules, MCP | ✔ | ✔ | ✔ (Devin Local) |
| Custom subagents (`agents/`) | ✘ | ✔ | ✔ |
| `command` hooks | all events **except** `SessionStart`/`SessionEnd` (run on session machine, only while up) | ✔ | ✔ |
| `prompt` hooks | ✘ | ✔ | ✔ |
| Personal installs | via synced personal manifest | `devin plugins install` (synced) / `--local` | via account |
| Org managed manifest reach | ✔ | only if it's the user's **primary** org | same |
| Enterprise/account manifest reach | ✔ | ✔ | ✔ |
| OAuth for plugin MCPs | connection made in web app | `devin mcp login` | — |

Also: plugins don't apply to the classic Cascade agent; enterprise can disable CLI plugins;
plugin MCPs don't yet appear in the MCP settings UI; governance fails open on fetch failure.
Sources: [plugins overview](https://docs.devinenterprise.com/cli/extensibility/plugins/overview.md),
[plugin-ecosystem](https://docs.devinenterprise.com/product-guides/plugin-ecosystem.md).

---

## 9. Best-practice patterns observed

- **Naming:** kebab-case slugs equal to the directory name (CI-enforced in both reference repos).
  Skill dirs kebab-case, `name` matching dir. Marketplace convention: one MCP server per plugin,
  server key = slug, companion skill named `<slug>-mcp`.
- **Manifests:** minimal keys; `displayName` only where a card title differs from slug;
  `description` written as the model-facing trigger ("Use when…" style in SKILL.md description,
  which doubles as the auto-invocation cue).
- **Portability pattern (ponytail):** keep shared content in `skills/`, `AGENTS.md`, `hooks/` and
  add a thin per-host manifest adapter (`.devin-plugin/`, `.claude-plugin/`, `.codex-plugin/`…)
  rather than forking content per host. Its `docs/agent-portability.md` maps host→files.
- **README conventions:** install command per host, command table, uninstall table, screenshots/
  logos under `assets/`; i18n variants optional.
- **Security hygiene:** never commit credentials; `${NAME}` placeholders only; `logo` as a
  checked-in relative file; pin third-party entries to 40-hex SHAs; review vendor + commit
  before endorsing.
- **Validation in CI:** every reference repo ships a validator run on PR —
  `scripts/validate.py` (marketplace; structural + canonical-form + upstream-fetch modes),
  `scripts/validate-plugins.mjs` (plugin-template), `scripts/validate-template.mjs`
  (team-marketplace-template).
- **Keep `AGENTS.md` short** — it's always-on context for every install.
- **Skill granularity:** the plugin is the install unit; to offer skills separately, split them
  into separate plugins.

---

## 10. Unverified / gaps

- ~~**`devin plugins --help` output**~~ — RESOLVED: verified against `devin 3000.10.21`
  on this machine. Subcommands: `install` (sources: `owner/repo`, git URL, local path,
  `#path/to/plugin` subfolder fragment; flags `-y/--yes`, `--local`), `list`, `info`
  (shows skills, hooks, rules, MCP servers, required/optional/forbidden lists),
  `update`, `remove`, `prune`. `devin plugins info` output also confirmed the
  `/<plugin>:<skill>` namespacing and always-on `AGENTS.md` rules on the installed
  ponytail plugin.
- ~~**`devin plugins marketplace` subcommand**~~ — RESOLVED: confirmed absent from the
  binary's subcommand list; marketplace UX is the web app's Customize page, and a
  "marketplace" repo is just a meta-plugin installed via `devin plugins install`.
- **Plugin-root MCP filename:** docs say `.mcp.json`; kitchen-sink ships `mcp_config.json` with a
  `serverUrl` key. Both likely load, but the complete set of conventional filenames (and whether
  `serverUrl` is officially supported vs. `url`) is UNVERIFIED.
- **`disable-model-invocation`** SKILL.md field: confirmed present in the installed
  uploaded bundle (`…\account-upload_user_default-eccf859c\0.1.0\skills\handoff\SKILL.md`,
  used with `triggers: [user]`), but absent from the documented field table — likely a
  Claude-skills-compatible escape hatch for "user-only" skills.
- Exact cache-dir sanitization rule (`github.com_DietrichGebert_ponytail-38c3408d` →
  `host_owner_repo-<8-hex>`) inferred from two examples; hash input UNVERIFIED.
- Whether `devin plugins install` accepts a bare git URL *with* `#sub/dir` for non-GitHub hosts:
  the docs show `#plugins/review` on `owner/repo` form and `.git#plugins/hello-world` in the
  plugin-template README; generalization to arbitrary hosts is inferred.
- docs.devin.ai vs docs.devinenterprise.com: content assumed identical (same site structure,
  `.md` endpoints); the enterprise mirror may carry enterprise-specific extras not in the
  consumer docs.

## Sources index

- Docs (canonical `docs.devin.ai`; fetched via `docs.devinenterprise.com` mirror):
  `/cli/extensibility/plugins/overview`, `/cli/extensibility/plugins/quickstart`,
  `/cli/extensibility/skills/creating-skills`, `/cli/extensibility/rules`,
  `/cli/extensibility/hooks/overview`, `/cli/extensibility/hooks/lifecycle-hooks`,
  `/cli/extensibility/mcp/configuration`, `/cli/subagents`, `/product-guides/plugins`,
  `/product-guides/plugin-ecosystem`, `/llms.txt`.
- Reference repos: [CognitionAI/devin-marketplace](https://github.com/CognitionAI/devin-marketplace)
  (root `plugin.json`, `scripts/validate.py`, `.agents/skills/adding-a-plugin/SKILL.md`,
  `plugins/{context7,aws-core,notion,sentry}/.devin-plugin/plugin.json`),
  [CognitionAI/plugin-template](https://github.com/CognitionAI/plugin-template)
  (`plugins/{hello-world,kitchen-sink}/…`, `scripts/validate-plugins.mjs`),
  [CognitionAI/team-marketplace-template](https://github.com/CognitionAI/team-marketplace-template)
  (root `plugin.json`).
- Local machine: `C:\Users\jahanson\AppData\Roaming\devin\cli\plugins\{lock.json,discovered.json}`,
  `…\plugins\cache\github.com_DietrichGebert_ponytail-38c3408d\4.9.0\` (full third-party plugin),
  `…\plugins\cache\account-upload_user_default-eccf859c\0.1.0\` (uploaded bundle),
  `C:\Users\jahanson\AppData\Roaming\devin\config.json`, `C:\Users\jahanson\AppData\Roaming\devin\skills\`.
