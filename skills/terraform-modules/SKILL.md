---
name: terraform-modules
description: >-
  Module design, composition, and publication. Use for standard module layout (README, main.tf,
  variables.tf, outputs.tf), public/private registry usage, semantic versioning and Git refs,
  composing shared infrastructure, avoiding provider blocks inside reusable modules (no-provider-blocks
  rule) with documented exceptions, terraform-docs, examples/ directories, and module testing
  patterns. Pair with terraform-providers for configuration_aliases.
---

# Terraform modules: layout, composition, and hygiene

## Scope

Modules are **packaged configurations** you call with `module` blocks. This skill targets **reusable**
modules shared across teams—not one-off folder splits. State, CI, and cloud specifics are covered in
sibling skills.

## Standard layout

A maintainable module repository contains:

```
modules/vpc/
├── README.md           # generated or hand-written design & usage
├── main.tf             # primary resources
├── variables.tf        # inputs
├── outputs.tf          # contract surface
├── versions.tf         # terraform + required_providers
├── providers.tf        # ONLY for root wrappers, usually absent in child libs
├── examples/
│   └── default/
│       ├── main.tf
│       └── README.md
└── tests/              # optional terraform test or external harness
```

**`examples/`** should `terraform init && validate` in CI—the fastest signal that your interface still
works.

## No provider blocks inside reusable modules

**Rule:** Library modules should **not** pin `provider` blocks (except `terraform` constraints via
`required_providers`). Callers pass providers through explicit `providers = { ... }` maps when
aliases are needed.

**Why:** Hidden defaults cause resources to deploy into whichever account/region the caller’s
ambient credentials happen to target—silent data exfiltration or cross-env corruption.

**Exception:** Thin **root wrappers** that exist solely to instantiate a composition may set provider
defaults. Mark them clearly as **not** library-grade.

### Correct pattern

```hcl
# modules/lambda-ecr/versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# modules/lambda-ecr/main.tf
resource "aws_lambda_function" "this" {
  function_name = var.name
  package_type  = "Image"
  image_uri       = var.image_uri
  role            = aws_iam_role.execution.arn
  architectures   = ["x86_64"]
  # ...
}
```

### Root wires providers

```hcl
module "imu_lambda" {
  source = "git::https://example.com/terraform-modules.git//lambda-ecr?ref=v2.3.0"

  name      = "imu-processor"
  image_uri = "${data.aws_ecr_repository.imu.repository_url}:v1.4.2"

  providers = {
    aws = aws
  }
}
```

## Semantic versioning and Git refs

Registry modules use **semver** tags. Private Git modules typically pin:

```hcl
source = "git::https://github.com/myorg/tf-modules.git//vpc?ref=v1.8.0"
```

Avoid **floating** `ref=main` for anything beyond developer sandboxes. For hotfixes, cut **`v1.8.1`**
rather than retagging—Git tags should be immutable.

## Public vs private registry

**Public Registry** (`registry.terraform.io`) provides verified namespaces and signing expectations
(set org policies for which namespaces are allowed). **Private Registry** (Terraform Cloud/Enterprise
or self-hosted) mirrors tarball distribution while enforcing **SSO**.

Document in your README whether consumers should use:

```hcl
source  = "app.terraform.io/my-org/lambda-ecr/aws"
version = "2.3.0"
```

## Composing modules without mega-stacks

Prefer **narrow contracts**:

- **`network`** exports `vpc_id`, **private_route_table_ids**, **nat_gateway_ids`.
- **`data-plane`** accepts those outputs plus **Kinesis ARN**.
- **`observability`** attaches alarms based on **known CloudWatch namespaces**.

Keep modules **< ~300 lines** of Terraform where practical; split nested modules when files sprawl.

## terraform-docs

Use `terraform-docs` to keep **README inputs/outputs** synchronized:

```bash
terraform-docs markdown table --output-file README.md --output-mode inject .
```

CI should fail when generated docs drift (`terraform-docs` in CI diff check).

## Variables and outputs as API design

Treat `variables.tf` like a **function signature**:

- Defaults belong on **safe** booleans and tags—not on CIDRs that gate production traffic.
- Use `validation` blocks for quick feedback.
- Outputs should expose **ARNs/IDs** other stacks need—not entire resource objects unless justified.

```hcl
output "lambda_function_arn" {
  value       = aws_lambda_function.this.arn
  description = "Processor function ARN for event source mappings."
}
```

## Testing modules

Options:

1. **`terraform test`** with `tests/*.tftest.hcl` inside the module—fast, native—see `terraform-testing`.
2. **Terratest** in Go for integration with real clouds (slow, powerful).
3. **Kitchen-Terraform** (legacy but still found in enterprises).

At minimum, run `terraform fmt`, `validate`, and **`examples/*` init** on every PR.

## Anti-patterns

- **Data sources inside modules** that reach “anywhere” (`aws_vpc` by ambiguous tag) create fragile
  coupling—pass IDs explicitly.
- **`count` with index-based naming** breaks stability—prefer `for_each` keys derived from AZ or
  subnet name.
- **Over-broad `depends_on`**—use references instead.

## Example: examples/default

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

module "demo" {
  source = "../../"

  name      = "example-imu-processor"
  image_uri = "123456789012.dkr.ecr us-east-1.amazonaws.com/imu:v1.0.0"
}
```

Ship a **one-command** README snippet so new users succeed quickly.

## Module versioning strategy across environments

Many teams mirror Git tags to **Terraform Cloud module publishing**. Keep **identical semver** across
mirrors; if Enterprise registry lags public Git, automate promotion with signed pipelines.

## Dealing with breaking changes

1. **Major version bump** (`v2.0.0`).
2. **`moved` blocks** inside the module to preserve state addresses when possible.
3. **CHANGELOG entry** with explicit upgrade commands (`terraform state mv` fallback).
4. **Deprecation outputs**—temporary outputs warning consumers to rename inputs before removal.

## When not to use modules

If you only wrap **one resource** without abstraction benefit, keep it inline—modules carry mental
overhead. Likewise, don’t nest seven layers deep; flatten for readability.

## Contract with platform security

Platform teams should supply **hardened baselines** (KMS CMKs, SCP-aware naming, mandatory tags).
Expose **`var.tags`** and merge with hardened locals:

```hcl
locals {
  merged_tags = merge(var.tags, {
    Module      = basename(path.module)
    Environment = var.environment
  })
}
```

## Workspace/module confusion

**Workspaces** split **state**; **modules** split **configuration**. Don't use workspaces where
separate state files per business unit are safer—workspace mistakes can apply prod code to staging
state if human discipline slips.

## Large-object handling

When packaging **Lambda zip** assets, modules sometimes include `aws_s3_object` uploads. Document
maximum sizes and CI steps to build artefacts **before** `terraform apply` to avoid slow plans.

## Team review checklist

- Does README show **minimal** example?
- Are providers **only** in versions.tf/`required_providers`?
- Are **examples/** and **`terraform test`** present?
- Did **`terraform-docs`** update?
- Is **`ref` pinned** for Git modules?

## Growing a module library

Start with **three** stable modules (`network`, `eks/aks/gke`, `data-stores`) before abstracting
everything. Mature libraries add **observability** and **IAM** packs that accept **ARNs** rather
than reaching into sibling stacks unexpectedly.

## Interaction with OpenTofu registry

If you standardize on **OpenTofu**, verify private registry tokens and download URLs—see
`terraform-opentofu`. Module source strings remain similar; tooling around test may differ slightly.

## References across modules

Prefer **`terraform_remote_state`** or explicit **SSM Parameter Store / Secrets Manager** hand-offs
for cross-stack values. Never rely on “just query tags” unless tags are guaranteed unique by org policy.

## Finishing thoughts

Good modules behave like **libraries**: small surface, strong typing, explicit dependencies, docs,
and tests. Poor modules behave like **black boxes** that surprise operators at 2 AM—avoid surprises.

## Meta-data with terraform.workspace (use carefully)

`terraform.workspace` can suffix resource names in examples, but **relying on it inside shared
modules** couples your library to workspace naming conventions. Prefer an explicit `var.environment`
string so callers using **separate directories** per env behave identically to workspace users.

## Wrappers vs leaf modules

**Wrapper stacks** (env-live/prod, env-live/staging) compose leaf modules and typically own backend
`terraform` blocks, provider configurations, and remote state. Keep wrappers **thin**: glue code,
`locals` for tagging standards, and `module` blocks. Leaf modules should remain portable enough for
**integration tests** and open-source style reuse.

## Nested modules on disk

```
modules/network/
  main.tf
  modules/subnets/
    main.tf
```

Reference nested folders with relative sources:

```hcl
module "subnets" {
  source = "./modules/subnets"
}
```

Document nested boundaries in README so contributors do not accidentally promote **internal-only**
helpers to public API.

## Internal-only variables

Prefix experimental inputs with `experimental_` or mark them `nullable` and **undocumented** until
stable. Some teams reject any variable lacking README documentation in CI—automate that check.

## Cost estimation hooks

Consider pairing modules with **Infracost** or cloud vendor calculators in PR comments—module
authors should expose **instance sizes** and **replica counts** as variables to make cost models
honest. This is not strictly Terraform language but significantly affects module design reviews.

## Disaster recovery ownership

Modules that create **stateful** resources should README the **backup expectation**: who snapshots,
which alarms exist, how restores interact with Terraform (`prevent_destroy`, `lifecycle` hooks). A
module without DR notes often becomes production-critical silently.

## Changelog discipline

Keep `CHANGELOG.md` following **Keep a Changelog** with sections **Added/Changed/Fixed/Removed**.
Link to migration PRs. Consumers upgrade faster when they trust your release notes more than
reverse-engineering plans.

## Code reviews: what to grep for

Reviewers should search PR diffs for:

- `provider "aws"` inside `modules/` (reject unless justified)
- `~> 999` style nonsense constraints (typos happen)
- `count = length(var.items)` without `lifecycle` thought when lists reorder
- hard-coded account IDs (prefer data sources or variables)

## Student exercise

Take a **monolithic root module** and extract **one bounded submodule** (`sns_topics`, `iam_roles`)
end-to-end: write example, docs, upgrade guide for teammates. The exercise reveals which outputs were
implicitly “known” only to the original author—make them explicit.

## Version constraint duplication

It is normal—and desirable—for **root modules** and **child modules** both to declare
`required_providers`. The child declares minimum capabilities; the root may tighten or broaden
depending on how it composes providers. Conflicts appear as `init` errors early; fix them in the
child if the library should be more permissive.

Publish a short **“upgrade playbook”** wiki page listing commands (`init -upgrade`, `providers lock`,
example `plan`) so consumers don’t each invent divergent procedures when semver tags advance.

