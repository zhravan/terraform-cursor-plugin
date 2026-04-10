# Terraform Cursor plugin

Agent **Skills**, **command guides** (terraform workflows + risk notes), and optional **MCP** wiring for Terraform, OpenTofu, Terragrunt, and major clouds. Manifest version: **`1.0.0`** in [`.cursor-plugin/plugin.json`](.cursor-plugin/plugin.json).

| | Count | Start here |
|--|------:|------------|
| Skills | 17 | [`skills/terraform-core/SKILL.md`](skills/terraform-core/SKILL.md) |
| Commands | 24 | [`commands/`](commands/tf-init.md) |
| MCP | 1 | [`.mcp.json`](.mcp.json) |

**Needs:** Cursor with plugin support. **Optional:** Docker for MCP (`hashicorp/terraform-mcp-server` per `.mcp.json`; not on npm - see [HashiCorp: deploy MCP server](https://developer.hashicorp.com/terraform/docs/tools/mcp-server/deploy)). Set `TFE_TOKEN` / `TFE_HOSTNAME` for HCP Terraform / TFE.

## Install

1. **Clone** (or fork) and point Cursor at this folder as a local plugin, then reload.
2. **Zip:** download from [Releases](https://github.com/zhravan/terraform-cursor-plugin/releases) (`terraform-cursor-plugin.zip` after a `v*` tag), **or** **Actions → Package →** latest run → artifact (manual runs do **not** create a Release). This repo does **not** use GitHub **Packages**.
3. Extract if needed; layout matches the tree below.

**Release tip:** Pushing a tag creates the Release, e.g. `git tag v1.0.1 && git push origin v1.0.1`. Workflows must be on the default branch; enable Actions under repo **Settings** if needed.

## Layout

`.cursor-plugin/plugin.json` (authoritative paths), `.mcp.json`, `commands/`, `skills/<name>/SKILL.md`, `CHANGELOG.md`, `cliff.toml` (release notes from [Conventional Commits](https://www.conventionalcommits.org/) via [git-cliff](https://git-cliff.org/)).

## CI

| Workflow | When |
|----------|------|
| [`ci.yml`](.github/workflows/ci.yml) | PR / `main` - JSON + manifest parity |
| [`package.yml`](.github/workflows/package.yml) | Manual or `v*` tag - zip + Release on tags |

Local check: `jq empty .cursor-plugin/plugin.json .mcp.json`

## Contributing

PRs welcome. Add or rename skills/commands in `plugin.json`. Use Conventional Commits (`feat:`, `fix:`, …) for release notes.

## License

[MIT](LICENSE).
