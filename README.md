# terraform-cursor-plugin

Terraform skills (`skills/*/SKILL.md`), CLI stubs (`commands/*.md`), manifest (`.cursor-plugin/plugin.json`), MCP (`.mcp.json` - needs Docker + `hashicorp/terraform-mcp-server`).

Install: put this folder where Cursor loads plugins, reload. Details: paths and versions live in `plugin.json` and HashiCorp MCP [deploy docs](https://developer.hashicorp.com/terraform/docs/tools/mcp-server/deploy).

CI validates the manifest on PRs. **Actions → Package → Run workflow** downloads a zip; pushing tag `v*` also uploads that zip to **Releases**.

MIT - see `LICENSE`.
