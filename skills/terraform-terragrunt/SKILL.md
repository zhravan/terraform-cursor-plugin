---
name: terraform-terragrunt
description: >-
  Terragrunt workflows for DRY Terraform: terragrunt.hcl units, generate blocks for provider/backend,
  dependency wiring with mock outputs, remote_state coordination, run-all batching, include chains,
  versioning considerations, and guidance on when the added complexity pays off versus plain Terraform
  modules. Use for multi-environment fleet management—not as default for tiny stacks.
---

# Terragrunt: DRY orchestration around Terraform

## Scope

**Terragrunt** is a thin wrapper that keeps **Terraform code DRY** across many environments by
generating `.tf` fragments, orchestrating **remote state**, and coordinating **`run-all`** applies.
It is **not** a replacement for well-factored modules.

## terragrunt.hcl anatomy (typical)

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "github.com/myorg/terraform-modules.git//app?ref=v3.2.0"
}

inputs = {
  env        = "prod"
  vpc_id     = dependency.network.outputs.vpc_id
  stream_arn = dependency.ingest.outputs.stream_arn
}

dependency "network" {
  config_path = "../network"

  mock_outputs = {
    vpc_id = "vpc-mock"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "ingest" {
  config_path = "../ingest"

  mock_outputs = {
    stream_arn = "arn:aws:kinesis:us-east-1:111111111111:stream/mock"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "us-east-1"
}
EOF
}
```

## generate blocks

Use **`generate`** for boilerplate `backend.tf` / `provider.tf` files to avoid copy/paste across env
folders—keep generated files **.gitignored** when policy demands (many teams commit them for
transparency—pick one approach and document).

## remote_state block

Terragrunt can configure **remote_state** once in `root.hcl`, injecting backend settings per unit—
ensures uniform bucket/key naming (`${path_relative_to_include()}` patterns).

## run-all

```bash
terragrunt run-all plan --terragrunt-non-interactive
terragrunt run-all apply --terragrunt-non-interactive
```

Understand **parallelism** and graph ordering—`dependency` links define hints; failures mid-run require
manual triage across stacks.

## When Terragrunt is worth it

**Yes** when:

- Dozens of similar env folders need shared `backend`/`provider` snippets.
- You want enforced ordering hints between stacks (`dependency`).
- Monorepo with **clear directory per env**.

**No** when:

- Single root module with a few workspaces—native Terraform workspaces or separate stacks suffice.
- Team lacks bandwidth to debug wrapper issues—pure Terraform reduces moving parts.

## Version coupling

Pin **Terragrunt** versions in CI like Terraform—upgrades can change parsing behavior.

## include chains

`find_in_parent_folders` enables **hierarchical** configuration—great power, great confusion if depth
changes; add comments explaining folder layout in `README`.

## Mock outputs pitfalls

Mock outputs allow `validate` without live `apply` of dependencies—misconfigured mocks hide missing
outputs until integration—periodically run **full ordered apply** in staging.

## generate + fmt

Generated files might not pass `terraform fmt` if templates sloppy—lint generation templates.

## Hooks

Terragrunt **before_hook / after_hook** can run `tflint`, `checkov`—keep hooks fast to preserve dev UX.

## CI integration

Wrap `terragrunt run-all plan` with **IAM role** assuming per account; matrix by account folder.

## State key conventions

Encode **account/env** in state keys consistently—Terragrunt’s relative path pattern helps until teams
move folders—mind refactors.

## When not to use this skill

Terraform-only module authoring → `terraform-modules`.

## Common errors

- **Cyclic dependency** between terragrunt units—break by introducing third `bootstrap` unit.
- **Provider version conflicts** when generated `required_providers` mismatches module—test `init`.

## Interop with Terraform Cloud

Remote backend integration differs—validate whether Terragrunt generation fights Terraform Cloud
workspaces—some teams skip Terragrunt in TFC-only flows.

## Local development ergonomia

`terragrunt hclfmt` formats HCL consistently—pre-commit friendly.

## Testing strategy

Smoke test **single unit** before `run-all` in CI to isolate failures faster.

## Documentation

Draw **folder tree** diagrams in repo root—new hires struggle without visual maps.

## Anti-pattern: business logic in terragrunt

Keep complex transforms in **Terraform modules**, not in `inputs = { ... massive ternary ... }`—
Terragrunt should orchestrate, not implement algorithms.

## Secrets in terragrunt.hcl

Avoid embedding secrets directly—use **env vars** / **SOPS** loaders; Terragrunt can `read_terragrunt_config`
across includes—watch for accidental secret commit in generated files.

## Performance

`run-all` may download modules repeatedly—enable **download directories caching** options (per
Terragrunt version docs) in CI.

## Migration off terragrunt

Document exit path—some teams graduate to pure Terraform with reusable modules once fleet complexity
drops; keep migration notes to avoid lock-in fear.

## Closing

Terragrunt rewards **discipline**—without folder conventions and CI guardrails, it amplifies chaos.
Use it when repetition pain clearly exceeds wrapper overhead.

## Advanced: read_terragrunt_config

Compose configs dynamically—powerful for multi-region loops—beware readability; complex Terragrunt can
be harder to debug than HCL in `.tf` modules.

## Windows paths

`find_in_parent_folders` behaves differently under Git bash vs PowerShell—standardize dev environment
or document gotchas.

## include path limits

Very deep repos may hit path length limits on Windows—another reason CI uses Linux.

## run-all graph exports

Some teams export Terragrunt dependency graphs into documentation during releases—helps auditors see
blast radius ordering.

## Terragrunt vs workspaces

Workspaces split state within one config; Terragrunt typically splits **directories**—choose based on
desired PR granularity—monolithic workspace PRs scare reviewers.

## State encryption alignment

Even with Terragrunt, backend encryption remains paramount—`remote_state` config must still set KMS keys.

## Provider generation caution

Generating multiple provider aliases requires careful template testing—one typo duplicates providers block
wide.

## run-all apply approvals

Use manual gates between `plan-all` and `apply-all` stages—full automation without human review is rare
for prod.

## Troubleshooting pointers

If `dependency.outputs` empty, ensure dependency stack applied—Terragrunt doesn’t auto apply unless
configured— see `terraform-troubleshooting` for underlying Terraform errors masked by wrapper.

## Community plugins

Third-party Terragrunt extensions exist—evaluate support burden before adopting—plain hooks often suffice.

## Benchmark anecdote

Teams report faster developer onboarding with Terragrunt when folder patterns consistent—measure your
own p50 `terragrunt plan` times quarterly.

## Final mantra

Terragrunt should **delete repetition**, not **add mystery**—if new engineers need a two-hour lecture
to run `plan`, simplify.

## Upgrade testing matrix

Test new Terragrunt versions against **oldest supported Terraform** in your estate—compat boundaries
shift.

## Observability

Wrap `run-all` logs with **timestamps** and **unit names**—parallel logs interleave confusingly in
naive CI captures.

## Secrets injection pattern

Use `aws ssm get-parameter` in `before_hook` sparingly—failures stall plans; prefer CI to export env.

## Coordination with monorepo scanners

Point Checkov/Trivy at **generated** `.tf` directories when committed—or scan only source modules if
generated files excluded; align policy to avoid blind spots.

## Retrospective prompts

After Terragrunt incidents, ask: “Would pure Terraform have been clearer?” Answer honestly quarterly.

## Summary

Terragrunt is **glue**—valuable glue, but still glue—don’t let it become the place where unclear
architecture hides.

## Unit test analogy

Treat each Terragrunt directory as a **unit** with contract tests in CI—`terragrunt render-json` can
snapshot expected inputs for diffs during refactors.

## IAM role chaining

When `run-all` spans accounts, ensure **session naming** identifies pipeline job IDs in CloudTrail—
Terragrunt doesn’t add magic; roles must still be least privilege.

## Partial applies recovery

If `run-all apply` fails halfway, rerun only failed units after fixing root cause—document command line
snippets in runbook to avoid accidental full reruns costing time.

## Caching downloaded modules

Configure **`TERRAGRUNT_DOWNLOAD`** paths in CI to reuse module downloads—invalidate when lock files
change—speed matters on large fleets.

## CLI autocomplete

Ship **shell autocomplete** instructions for engineers—small quality-of-life reduces typos in long
`run-all` invocations.

## Disaster recovery

Back up Terragrunt **include trees** as zealously as Terraform modules—losing `root.hcl` hurts as much
as losing `backend.tf`.

## Roadmap awareness

Follow Terragrunt release notes for Terraform compatibility claims—pin pair (`TG x.y`, `TF a.b`) in
internal standards doc.

## Visualizing dependencies

Use `terragrunt graph-dependencies` (if available in your version) and render to PNG in docs during
major topology changes—picture clarifies ordering arguments in PR reviews.

## Pairing with Atlantis

Atlantis project definitions can call Terragrunt per directory—standardize comment commands (`atlantis
plan -p network`) mapping cleanly to Terragrunt units—confusion here spawns mis-applies.

## Avoiding god includes

`root.hcl` growing unbounded hints missing Terraform module abstraction—refactor repeated `inputs`
constructs into modules instead of mega includes.

## SOPS integration example sketch

```
read_terragrunt_config(find_in_parent_folders("encrypted.hcl"))
```

Keep examples version-controlled without plaintext—pair with age recipients file in repo doc.

## Culture

Senior engineers should occasionally **`terragrunt run-all plan` locally** on laptops mirroring CI,
catching parallelism issues early instead of only in pipelines.

## Final checklist

- Folder conventions documented?
- Mock outputs tested?
- CI pins Terragrunt version?
- Backend generation reviewed for KMS?
- Secrets not in HCL?

## Closing line

Terragrunt’s payoff is **scale**: many similar stacks with shared bones—without scale, prefer simpler tools.

## Multi-region duplication

Symlink or template `terragrunt.hcl` per region carefully—accidental symlink loops confuse Git on
Windows; prefer explicit copies with shared `root.hcl` if symlinks painful.

## Cost ownership

`run-all` encourages many stacks—ensure **FinOps** reviews folder sprawl; inactive stacks still cost money
even if Terragrunt config elegant.

## Audit exports

For regulated customers, export rendered Terragrunt JSON inputs per release tag—proves what values were
seen during Terraform execution.

## Developer laptop resource usage

Parallelism on laptops can peg CPU—teach `TERRAGRUNT_PARALLELISM` tuning for humane local dev
experience.

## Naming collision between units

Prefix Terragrunt-managed DynamoDB lock entries (if any) uniquely—some teams share locking infra across
mono repo units; misconfiguration causes spooky cross-talk.

## Editor support

VS Code Terragrunt extension maturity fluctuates—fallback to HCL syntax highlighting if specialized
features buggy; don’t block adoption on IDE polish.

## Layered environments (dev/stage/prod)

Promote modules by **bumping `terraform.source` ref** sequentially through env folders—git diff of bumps
should be trivially reviewable.

## Bootstrap ordering

First-time org setup requires **S3+Dynamo** stack applied before others—Terragrunt `dependency` cannot
fabricate buckets that don’t exist—human bootstrap still real.

## Long path names on CI

Shorten `path_relative_to_include()` segments if S3 object key length approaches limits—especially for
deep monorepos.

## Post-incident note

When debugging, grab **`terragrunt dag`**/`graph` outputs plus underlying **`terraform plan`** logs—
wrappers obscure root causes if you only keep Terragrunt stdout.

## Encouraging modular thinking

Terragrunt is not license to skip **module semver** hygiene—keep modules small; Terragrunt composes,
not replaces, module quality.

## Final reassurance

Used well, Terragrunt **reduces** copy/paste defects; used poorly, it hides them behind indirection—pick
your team’s maturity level honestly.

