# Terraform Cursor plugin

Cursor plugin providing **Agent Skills** and **commands** for production HashiCorp Terraform and related workflows: language semantics, remote state, providers and modules, major clouds (AWS, Azure, GCP), Kubernetes, testing, security, CI/CD, secrets, multi-account, refactoring, OpenTofu, Terragrunt, and operational troubleshooting.

Structure mirrors [appwrite/cursor-plugin](https://github.com/appwrite/cursor-plugin): `.cursor-plugin/plugin.json`, `skills/<name>/SKILL.md`, `commands/<name>.md`, `.mcp.json`, plus `README.md`, `LICENSE`, and `CHANGELOG.md`.

## Install (local plugin)

1. Clone or copy this directory to a location Cursor loads plugins from (for example `~/.cursor/plugins/local/terraform-cursor-plugin`—follow current Cursor documentation for plugin discovery paths).
2. Restart Cursor or reload plugins so skills and commands appear.
3. Optional: enable MCP (see below).

## Model Context Protocol (Terraform Registry)

The official **HashiCorp Terraform MCP server** exposes up-to-date Registry provider and module information. It is **not** published to npm as `@hashicorp/terraform-mcp-server` (that scoped package does not exist on the public npm registry). Supported install paths per HashiCorp documentation include:

- **Docker (recommended default in `.mcp.json`)**: image `hashicorp/terraform-mcp-server`, `stdio` via `docker run -i --rm`.
- **Binary**: download from [HashiCorp releases](https://releases.hashicorp.com/terraform-mcp-server).
- **Go**: `go install github.com/hashicorp/terraform-mcp-server/cmd/terraform-mcp-server@latest`

See [Deploy the Terraform MCP server](https://developer.hashicorp.com/terraform/docs/tools/mcp-server/deploy).

### Authentication

- Without an API token, the server can still answer many Registry-oriented prompts (behavior depends on server version).
- For HCP Terraform / Terraform Enterprise integration, set `TFE_TOKEN` and optionally `TFE_HOSTNAME` (or platform-specific equivalents documented by HashiCorp). Treat tokens as secrets; prefer short-lived tokens and least privilege.

### Docker requirement

The bundled `.mcp.json` assumes **Docker** is installed and on `PATH`. If you cannot run Docker, replace the `command`/`args` in `.mcp.json` with the path to a locally installed `terraform-mcp-server` binary.

## Manifest and inventory

`.cursor-plugin/plugin.json` lists every skill and command path. Paths are relative to this plugin root and must match files on disk.

## Skills

| Skill | Path |
|-------|------|
| Core language and configuration | `skills/terraform-core/SKILL.md` |
| State, backends, remote state, encryption | `skills/terraform-state/SKILL.md` |
| Provider configuration and tooling | `skills/terraform-providers/SKILL.md` |
| Module design and registries | `skills/terraform-modules/SKILL.md` |
| AWS (provider 5+, IMU/Kinesis/Lambda/IoT anchor) | `skills/terraform-aws/SKILL.md` |
| Azure (azurerm) | `skills/terraform-azure/SKILL.md` |
| GCP (google / google-beta) | `skills/terraform-gcp/SKILL.md` |
| Kubernetes and Helm | `skills/terraform-kubernetes/SKILL.md` |
| Testing (terraform test, Terratest, linters, policy) | `skills/terraform-testing/SKILL.md` |
| Security and compliance | `skills/terraform-security/SKILL.md` |
| CI/CD patterns | `skills/terraform-cicd/SKILL.md` |
| Secrets and sensitive data | `skills/terraform-secrets/SKILL.md` |
| Multi-account / organizations | `skills/terraform-multi-account/SKILL.md` |
| Refactoring and migration | `skills/terraform-refactoring/SKILL.md` |
| OpenTofu vs Terraform | `skills/terraform-opentofu/SKILL.md` |
| Terragrunt | `skills/terraform-terragrunt/SKILL.md` |
| Troubleshooting | `skills/terraform-troubleshooting/SKILL.md` |

## Commands

| Command doc | Purpose |
|-------------|---------|
| `commands/tf-init.md` | Initialize backend and modules |
| `commands/tf-plan.md` | Planning |
| `commands/tf-apply.md` | Applying changes |
| `commands/tf-destroy.md` | Destroy (high risk) |
| `commands/tf-fmt.md` | Format |
| `commands/tf-validate.md` | Validate configuration |
| `commands/tf-import.md` | Import existing resources |
| `commands/tf-state-list.md` | List state addresses |
| `commands/tf-state-show.md` | Show one instance |
| `commands/tf-state-mv.md` | Move addresses in state |
| `commands/tf-state-rm.md` | Remove from state |
| `commands/tf-state-pull.md` | Pull remote state JSON |
| `commands/tf-state-push.md` | Push state (dangerous) |
| `commands/tf-taint.md` | Taint (legacy) |
| `commands/tf-untaint.md` | Untaint |
| `commands/tf-output.md` | Root module outputs |
| `commands/tf-workspace.md` | Workspace management |
| `commands/tf-refresh.md` | Refresh-only |
| `commands/tf-console.md` | Expression console |
| `commands/tf-graph.md` | Dependency graph |
| `commands/tf-providers-lock.md` | Dependency lock file |
| `commands/tf-force-unlock.md` | Force unlock remote state |
| `commands/tf-version.md` | Print version |
| `commands/tf-show.md` | Show saved plan or state |

## Terraform vs OpenTofu

OpenTofu is a community fork with its own release cadence and feature set. For divergence, migration, and toolchain choice, see `skills/terraform-opentofu/SKILL.md`.

## License

MIT `LICENSE`.
