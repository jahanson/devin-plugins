# Repository guidance

## Purpose

This repository is a curated Devin CLI plugin marketplace. The root is a meta-plugin; its source of truth is `.devin-plugin/plugin.json`.

Use `docs/research/devin-cli-plugins.md` for the researched Devin plugin format, capability, distribution, and governance details. Prefer current official Devin documentation when it conflicts with the research snapshot.

## Marketplace structure

- Keep the root meta-plugin content-free: retain `"skills": []` and do not add root rules, hooks, agents, MCP servers, or `AGENTS.md` behavior intended for consumers.
- Add installable entries to `optionalPlugins` unless the user explicitly requests automatic installation through `requiredPlugins`.
- A plugin hosted below a repository root must use a `git-subdir` source whose `path` points to the directory containing that plugin's `.devin-plugin/plugin.json`.
- Treat the selected plugin directory as a security and capability boundary. Files above it are not plugin content. Do not widen the boundary merely to make discovery work.
- For an in-repository plugin, use `plugins/<plugin-name>/` with its own `.devin-plugin/plugin.json`.
- Plugin names use lowercase letters and numbers separated by single hyphens or periods. Keep names unique across the marketplace.

## External plugins

- Review the exact upstream content before adding it.
- Pin external sources to a full commit SHA for stable marketplace releases. A branch `ref` is acceptable only while developing a source change that has not yet been committed and pushed.
- Use the canonical clone URL and the narrowest valid `path`.
- Never place credentials or literal secrets in plugin files. MCP credentials use `${NAME}` references.

## Skills-only plugins

- A skills-only plugin may set `skills` to one or more plugin-relative collection directories.
- Each configured collection directory contains named skill directories, each with a `SKILL.md`, for example `engineering/tdd/SKILL.md`.
- Do not point a skills-only plugin at a repository root that also contains `AGENTS.md`, `hooks.json`, `agents/`, `rules/`, or MCP configuration. Put the plugin manifest at a narrower boundary instead.
- Do not copy or flatten skills solely to satisfy the conventional `skills/<name>/SKILL.md` layout; use the manifest's `skills` paths.

## Verification

After changing a manifest:

1. Parse every changed JSON file.
2. Confirm each `git-subdir.path` contains `.devin-plugin/plugin.json` at the referenced revision or clearly report when an unpushed source change prevents remote verification.
3. Enumerate the configured skill roots and verify that every intended skill has `SKILL.md` and excluded buckets are unreachable.
4. Confirm the plugin boundary contains only the requested capability types.
5. Review `git diff` and `git status`.

Do not install plugins globally, mutate the user's Devin plugin state, commit, or push unless explicitly requested.
