# Terraform Cursor plugin (repo rules)

Use this repo as a **local Cursor plugin** (preferred) or as a source of **Agent Skills**.

## Defaults

- Prefer **Terraform/OpenTofu CLI** commands that are already documented under `commands/`.
- Use **read-only / plan-first** workflows unless the user explicitly asks to apply changes.
- Never suggest committing secrets; follow the guidance in `skills/terraform-secrets/SKILL.md`.

## If MCP is enabled

If `.mcp.json` is configured in the user’s Cursor setup, prefer using the Terraform MCP server for:

- Reading module/provider docs and registry metadata
- Inspecting state safely (when available)
- Producing structured summaries of plans/state outputs

When MCP is not available, fall back to CLI guidance and the command guides in `commands/`.

## Skill routing

- Terraform fundamentals and workflows: `skills/terraform-core/SKILL.md`
- Troubleshooting: `skills/terraform-troubleshooting/SKILL.md`
- Security and secrets handling: `skills/terraform-security/SKILL.md`, `skills/terraform-secrets/SKILL.md`
- Cloud-specific: `skills/terraform-aws/SKILL.md`, `skills/terraform-azure/SKILL.md`, `skills/terraform-gcp/SKILL.md`
