---
name: terraform-providers
description: >-
  Provider installation, versioning, and advanced workflows. Use for required_providers blocks,
  version constraints, provider aliases, passing configuration_aliases into modules, dependency lock
  file (.terraform.lock.hcl), provider mirroring and air-gapped installs, development overrides
  (.terraformrc), and building or consuming custom SDK-based providers. Complements terraform-core
  and terraform-modules.
---

# Terraform providers: versioning, aliases, and supply chain

## Scope

You will configure **which providers** a stack may download, how **versions** evolve over time,
how **multiple configurations** (regions, accounts, Kubernetes clusters) coexist via **aliases**, and
how **CI** reproduces provider binaries through the **lock file** and optional **mirrors**.

## required_providers in terraform {}

Declare every provider your module touches. This defines **source** (registry namespace) and
**version** constraints independent of `provider` blocks.

```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.40"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.30"
    }
  }
}
```

### Version constraint operators

Common patterns:

- `~> 5.0` allows `5.x` but not `6.0` (pessimistic).
- `>= 5.50, < 6.0.0` expresses a window explicitly.
- **Pin majors** aggressively for infra stacks; allow minor/patch drift for bug fixes.

After editing constraints, run:

```bash
terraform init -upgrade
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64
```

Commit **`.terraform.lock.hcl`** so every machine and CI worker resolves identical provider builds.

## Provider blocks and aliases

Default provider configurations pick up environment variables (`AWS_PROFILE`, `GOOGLE_CREDENTIALS`).
Explicit `provider` blocks name alternate setups; **`alias`** exposes them to resources.

```hcl
provider "aws" {
  region = var.primary_region
}

provider "aws" {
  alias  = "replica"
  region = var.replica_region
}

resource "aws_s3_bucket_replication_configuration" "telemetry" {
  provider = aws.replica
  bucket   = aws_s3_bucket.telemetry.id
  # ...
}
```

## Passing providers into modules (configuration_aliases)

Child modules that need **multiple accounts** or **regions** declare empty `configuration_aliases`
in `required_providers`, then the root passes concrete providers via the `providers` map.

### Child module

```hcl
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.log_archive]
    }
  }
}

resource "aws_s3_bucket" "audit" {
  provider = aws.log_archive
  bucket   = "central-audit-${data.aws_caller_identity.current.account_id}"
}
```

### Root module

```hcl
module "logging" {
  source = "./modules/central-log-bucket"

  providers = {
    aws.log_archive = aws.security
  }
}
```

Mis-wiring `providers = { ... }` is a top cause of **resources landing in the wrong account** - always
`terraform plan` with an **account identity data source** (`aws_caller_identity`, `azurerm_client_config`)
when validating new maps.

## Lock file: why it matters

**`.terraform.lock.hcl`** stores cryptographic hashes for each provider **package** per **platform**
triple. Benefits:

- **Reproducible CI** - no surprise provider upgrades mid-pipeline.
- **Air-gapped mirrors** - CI pulls exactly the bytes you hashed locally when populating a mirror.

Never **delete** the lock file casually; instead regenerate with `terraform providers lock` when you
intentionally adopt a new provider version.

## Provider mirrors (filesystem_mirror, network_mirror)

Organizations block registry.terraform.io from servers. Configure **`.terraformrc`** or
**`TF_CLI_CONFIG_FILE`** to redirect discovery:

```hcl
provider_installation {
  filesystem_mirror {
    path    = "/var/terraform/providers"
    include = ["registry.terraform.io/hashicorp/*"]
  }

  direct {
    exclude = ["registry.terraform.io/hashicorp/*"]
  }
}
```

Populate `/var/terraform/providers` with `terraform providers mirror` or vendor packaging jobs.

## Development overrides

While building a forked provider, use **development overrides** so Terraform loads your local binary
without publishing:

```hcl
provider_installation {
  dev_overrides {
    "registry.terraform.io/myorg/aws" = "/Users/me/go/bin/terraform-provider-aws"
  }

  direct {}
}
```

Remember to **remove** overrides before shipping modules to colleagues - otherwise their machines
cannot find your local path.

## Custom and forked providers

To ship an in-house API:

1. Implement the **Terraform plugin protocol** (Go using `terraform-plugin-framework` or SDKv2).
2. Publish to **your** registry namespace or distribute via mirror.
3. Reference with `source = "terraform.example.com/myorg/myapi"`.

Keep plugin **major versions** aligned with breaking schema changes; use **migration guides** for
teams upgrading.

## Provider caching in CI

Mount a persistent cache between pipeline steps:

```bash
export TF_PLUGIN_CACHE_DIR="$PWD/.terraform-plugin-cache"
terraform init -input=false
```

Speed matters when you run dozens of workspaces - just ensure the cache directory is **trusted** and
occasionally purged if corruption is suspected.

## Security notes

- Prefer **HashiCorp Terraform Registry** or your **Private Registry** over arbitrary HTTP downloads.
- Verify CI uses **`terraform init -lockfile=readonly`** in pipelines where developers should not
  mutate the lock file unintentionally.
- When a CVE ships for a provider, bump the **minimum** version in `required_providers`, run `lock`
  for all target platforms, and backport as needed.

## Example: dual-region with explicit providers

```hcl
provider "aws" {
  alias  = "use1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "usw2"
  region = "us-west-2"
}

module "kinesis_ingest_use1" {
  source = "./modules/kinesis-ingest"

  providers = {
    aws = aws.use1
  }

  stream_name = "imu-telemetry-use1"
}

module "lambda_dr_usw2" {
  source = "./modules/lambda-ecr"

  providers = {
    aws = aws.usw2
  }

  function_name = "imu-processor-dr"
}
```

## Troubleshooting provider resolution

- **`Failed to query available provider packages`**: network or mirror misconfiguration; confirm
  `provider_installation` stanzas.
- **`provider package does not match any of the checksums`**: regenerate lock with the missing
  **OS/ARCH** or supply `TF_PLUGIN_CACHE_MAY_BREAK_DEPENDENCY_LOCK_FILE` only as a temporary escape
  hatch in dev (never standard in prod CI).

## When not to use this skill

- Resource-level AWS design → `terraform-aws`.
- Secrets handling → `terraform-secrets`.
- Choosing OpenTofu vs Terraform → `terraform-opentofu`.

## Multi-org provider authentication

Large enterprises sometimes require **assumed roles** per provider block using dynamic `assume_role`
blocks (AWS) or **`provider_meta`** patterns (limited). Document credential flow on diagrams;
on-call engineers should not guess which OIDC role feeds which alias during incidents.

## Long-term maintainability

Schedule **quarterly** provider reviews: read changelogs for majors (`aws` v5 to v6), run speculative
`plan` in staging with `-refresh-only` first, then full plans. Capture diffs attributable to
**new defaults** (common on cloud providers) separately from intentional config edits - teams that skip
this review get noisy plans that hide real drift.

## Habit: pair init with fmt and validate

```bash
terraform fmt -recursive
terraform init -backend=false -input=false
terraform validate
```

Adding `-backend=false` lets you validate modules in isolation before credentials exist - useful for
**module CI** that only checks configuration grammar.

## Closing reminders

- **Declare** all providers in `required_providers`.
- **Commit** `.terraform.lock.hcl`.
- **Alias** when you fan out regions/accounts; **wire** `configuration_aliases` deliberately.
- **Mirror** or **cache** for speed and air gaps.
- **Dev overrides** are personal - never leak them to shared roots.

## Downgrade and hotfix policies

Sometimes you must **temporarily cap** a provider version after a regression:

```hcl
aws = {
  source  = "hashicorp/aws"
  version = ">= 5.50.0, != 5.63.0"
}
```

Document the ticket, expected timeline for removal of the exclusion, and who validates the next
patch. Leaving `!=` pins forever creates archaeology problems - set calendar reminders.

## terraform providers schema (inspecting locally)

Developers can inspect resource schemas while offline:

```bash
terraform providers schema -json > providers.schema.json
```

CI can diff schema outputs across branches to catch **unexpected removals** when upgrading majors.
This is advanced but valuable for platform teams publishing internal modules.

## google vs google-beta

GCP ships **`google`** and **`google-beta`** provider binaries. Beta resources belong in the beta
provider; keep both version constraints aligned to reduce odd coupling. Example:

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.40"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.40"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

## azurerm features block and subscription pinning

The AzureRM provider uses **`features {}`** to toggle destructive behaviors (key vault purge, VM
extensions). Set it once per alias and pair with **`subscription_id`** **`tenant_id`** arguments when
you must be explicit about landing zones.

## kubernetes provider: kubeconfig vs exec

The Kubernetes provider can load **`kubeconfig`** paths or use **`exec`** plugins (EKS, GKE, AKS).
For CI OIDC, **`exec`** is typical - ensure the pipeline task caches credentials minimally and rotates
tokens per job.

```hcl
provider "kubernetes" {
  host                   = var.cluster_endpoint
  cluster_ca_certificate = base64decode(var.cluster_ca)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks", "get-token",
      "--cluster-name", var.cluster_name,
      "--region", var.aws_region
    ]
  }
}
```

## Provider protocol upgrades (SDKv2 vs plugin/framework)

When consuming internally developed providers, ask maintainers whether they are on **plugin
framework** or legacy SDKv2 - error messages and debugging steps differ. Framework providers often
surface richer diagnostics; SDKv2 may still power popular community forks.

## Coordinating with Terragrunt

Terragrunt can **generate** provider blocks per environment. Treat generated snippets as **build
artifacts** in review (`terragrunt.hcl` stays canonical). Ensure `required_providers` versions still
live in modules so `validate` works when Terragrunt is not involved - see `terraform-terragrunt`.

## Supply chain attestation (advanced)

Some enterprises store **`terraform providers lock`** checksums alongside **SBOM** metadata.
While Terraform does not ship SBOMs for third-party providers automatically, recording **exact
ZIP** URLs from your mirror plus file hashes in an internal ledger gives auditors a chain of
custody if a registry artifact ever changes unexpectedly.

## Practical checklist before upgrading majors

1. Read provider **upgrade guide** (AWS/Azure/GCP maintain detailed pages).
2. Run `terraform plan` in **shadow** workspace with copied state (where legal) to preview churn.
3. Split changes: first bump provider with **no config edits**, then follow with resource refactors.
4. Watch for **removed attributes** - use `moved` blocks when resource types split.

## Mindshare for platform teams

Publish a **short internal RFC template** for provider bumps: blast radius, rollback, tests run
(`validate`, `plan`, integration), and on-call notification window. Provider changes are as risky as
service deploys - give them the same ceremony.

