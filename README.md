# Devin Plugins

A curated marketplace of plugins for the Devin CLI.

## Available plugins

| Plugin | Source | Contents |
| --- | --- | --- |
| `mattpocock-skills` | [`jahanson/matt-skills`](https://github.com/jahanson/matt-skills) | 25 promoted engineering and productivity skills |

The Matt Pocock plugin uses `skills/` as its plugin boundary. It loads only the `engineering/` and `productivity/` collections, excluding repository-level rules, hooks, agents, and the `misc/`, `in-progress/`, and `deprecated/` skill buckets.

## Install

Install the marketplace meta-plugin:

```sh
devin plugins install jahanson/devin-plugins
```

The marketplace currently lists plugins under `optionalPlugins`. Optional plugins are endorsed but are not installed automatically. Install Matt Pocock's skills separately:

```sh
devin plugins install jahanson/matt-skills#skills
```

Verify the installation:

```sh
devin plugins list
devin plugins info mattpocock-skills
```

Skills are exposed using the plugin namespace, for example:

```text
/mattpocock-skills:tdd
/mattpocock-skills:code-review
```

## One-command installation

To make marketplace installation pull in every included plugin automatically, move its entries from `optionalPlugins` to `requiredPlugins` in [`.devin-plugin/plugin.json`](.devin-plugin/plugin.json). Then this command installs the meta-plugin and all required plugins recursively:

```sh
devin plugins install jahanson/devin-plugins
```

Use `optionalPlugins` for a catalog of separately selected plugins. Use `requiredPlugins` for a baseline bundle installed as one unit.

## Update or remove

```sh
devin plugins update mattpocock-skills
devin plugins update hsn-marketplace

devin plugins remove mattpocock-skills
devin plugins remove hsn-marketplace
```

## Local development

Install the marketplace checkout on this machine only:

```sh
devin plugins install --local .
```

Install the local Matt skills plugin directly while editing it:

```sh
devin plugins install --local V:/matt-skills/skills
```

Local-folder plugins are linked, so edits apply in the next Devin session without running `devin plugins update`.

## Repository layout

```text
.devin-plugin/plugin.json     Marketplace meta-plugin manifest
docs/research/                Devin CLI plugin research
```

The marketplace is an ordinary Devin plugin whose dependency lists act as its catalog. Devin CLI does not have a separate `marketplace add` command.

## References

- [Devin CLI plugins](https://docs.devin.ai/cli/extensibility/plugins/overview)
- [Team marketplace quickstart](https://docs.devin.ai/cli/extensibility/plugins/quickstart)
- [Plugin ecosystem guide](https://docs.devin.ai/product-guides/plugin-ecosystem)
- [Local research](docs/research/devin-cli-plugins.md)

## Maintaining the marketplace

- Treat each plugin directory as its capability boundary. A `git-subdir` entry must point to the directory containing that plugin's `.devin-plugin/plugin.json`.
- Put plugins maintained in this repository under `plugins/<plugin-name>/`, each with its own manifest.
- Review external plugin content before listing it. Track a branch only during development, then pin releases to a full commit SHA.
- Keep plugin names unique and use lowercase letters and numbers separated by single hyphens or periods.
- Never store credentials in plugin files. Use `${NAME}` references for MCP credentials.
- After changing a manifest, parse its JSON, verify referenced subdirectories and configured capability roots, then review `git diff` and `git status`.
