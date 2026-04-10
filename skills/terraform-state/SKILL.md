---
name: terraform-state
description: >-
  Remote state backends, state sharing, and safe operations. Use for S3 (+ DynamoDB vs native
  locking in Terraform 1.10+), GCS, azurerm, Terraform Cloud/Enterprise, terraform_remote_state,
  state surgery commands (mv, rm, pull, push, replace-provider), drift detection, refresh-only plans,
  and state encryption options. Pair with terraform-secrets for sensitive attributes in state.
---

# Terraform state and backends

## Scope

This skill is about **where Terraform stores state**, how teams **coordinate locks**, how stacks
**read each other's outputs**, and how to **repair** state without corrupting reality. It does not
duplicate provider resource docs (use cloud skills) except where resources implement encryption
(e.g. S3 bucket policies, KMS keys).

## Local vs remote state

**Local** `terraform.tfstate` is fine for solo experiments. Production teams use **remote backends**
so CI and multiple operators share one canonical state with **locking** to prevent concurrent writes.

## Amazon S3 backend (classic pattern)

Historically, **S3** stored the object and **DynamoDB** held a lock item. You create a bucket with
versioning, encryption, and tight bucket policy; a small DynamoDB table acts as the lock table.

```hcl
terraform {
  backend "s3" {
    bucket         = "myorg-terraform-state-us-east-1"
    key            = "apps/imu-pipeline/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:111111111111:key/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"

    # workspace_key_prefix optional when using workspaces
  }
}
```

Operational checklist:

- **Versioning ON** for accidental overwrite recovery.
- **Deny non-TLS** and **enforce SSE-KMS** with bucket policy.
- Restrict `s3:PutObject`/`DeleteObject` to CI roles and break-glass admin roles.

## Terraform 1.10+ S3 native state locking

Terraform **1.10 and newer** can use **S3 native locking** instead of DynamoDB in some
configurations (consult current AWS provider and Terraform release notes for your exact combo).
When migrating, plan a maintenance window: update backend config, re-init with `-migrate-state`, and
verify lock behavior under load before deleting DynamoDB if no longer required.

## Google Cloud Storage backend

GCS supports **state locking** via optional `use_lockfile`. Always enable **uniform bucket-level
access** and **CMEK** for regulated workloads.

```hcl
terraform {
  backend "gcs" {
    bucket = "myorg-tf-state"
    prefix = "apps/imu-pipeline/prod"

    # encryption optional via bucket default or Google-managed keys
  }
}
```

## AzureRM backend

Use a **Storage Account** container with **blob lease** locking (Terraform handles leases when
properly configured). Prefer **customer-managed keys** on the storage account for encryption at
rest and align network rules with private endpoints when mandated.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "tfstatemyorg01"
    container_name       = "tfstate"
    key                  = "imu-pipeline.prod.tfstate"
  }
}
```

## Terraform Cloud / Enterprise remote backend

Remote operations move **plan/apply** execution to HCP Terraform or Terraform Enterprise. Teams gain
**RBAC**, **policy sets**, run history, and optional **local** execution modes when required.

```hcl
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      name = "imu-pipeline-prod"
    }
  }
}
```

Set `credentials` via `terraform login` or environment tokens per vendor documentation.

## terraform_remote_state (read-only sharing)

Consumers read another stack’s **outputs** without importing its resources. Only **exposed outputs**
are visible; state remains in the producer backend.

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "myorg-terraform-state-us-east-1"
    key    = "network/prod/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_security_group_rule" "allow_from_vpc" {
  type              = "ingress"
  security_group_id = aws_security_group.lambda.id
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = [data.terraform_remote_state.network.outputs.vpc_cidr]
}
```

Hard anti-patterns:

- Reading **remote state** to infer **private resource attributes** that are not outputs—export
  explicit outputs instead of reaching into foreign state blobs.

## State surgery CLI patterns

Always **pull a backup** before `push`, `mv`, `rm`, or `replace-provider`.

```bash
terraform state pull >"tfstate.backup.$(date +%s).json"
```

### mv (rename)

```bash
terraform state mv 'aws_lambda_function.old' 'aws_lambda_function.new'
```

Use `moved` blocks when you want the rename **documented in Git**—preferred for teams.

### rm (forget)

```bash
terraform state rm 'module.app.aws_s3_bucket_public_access_block.leaky'
```

Run only after confirming the address is wrong or you intentionally orphan an object.

### replace-provider (registry moves or forks)

```bash
terraform state replace-provider 'registry.terraform.io/-/aws' 'registry.terraform.io/hashicorp/aws'
```

Run `terraform plan` immediately after; unexpected mass changes mean you used the wrong mapping.

### push (dangerous)

`terraform state push` can **overwrite** remote state. Require **peer review**, **backup**, and
maintenance communications before using.

## Drift detection and refresh-only plans

Drift occurs when operators change resources outside Terraform. **`terraform plan -refresh-only`**
(or modern `plan` subcommands, depending on version) focuses Terraform on reconciling state with
reality without proposing attribute edits—useful to see drift before a normal plan.

## State encryption

**Encryption at rest** is a property of the backend:

- **S3**: SSE-S3 or **SSE-KMS** with tight key policies.
- **GCS**: bucket default encryption with **CMEK** where required.
- **Azure**: storage encryption + CMK.
- **Terraform Cloud**: vendor-managed encryption; some tiers support customer-managed controls.

**Encryption in Terraform 1.10+** introduced optional **state file encryption** features—consult
current docs for enablement flags and key management; combine with backend-level KMS for defense in
depth.

## Workspace discipline

Workspaces **split state** by key in the same backend. Prefer **separate state keys** per
environment (`key = ".../prod"`) or fully separate workspaces/stacks when blast radius demands it.

## Troubleshooting pointers

- **Lock errors**: see `terraform-troubleshooting` for `force-unlock` policies; never unlock
  concurrent applies without verifying the holder crashed.
- **Missing outputs in remote_state**: add outputs in the producer; consumers cannot synthesize
  them.

## When not to use this skill

- How to declare AWS resources → `terraform-aws`.
- Secret ingestion patterns → `terraform-secrets`.
- Testing drift in CI → `terraform-testing` and `terraform-cicd`.

## Example: backend bootstrap (minimal IAM)

Keep bootstrap stacks tiny and **manually protected**—they create the bucket and lock table for
everyone else.

```hcl
resource "aws_s3_bucket" "tf_state" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "tf_locks" {
  name         = var.lock_table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

After bootstrap, flip all application stacks to use `backend "s3"` pointing at these resources.

## Backend initialization and migration

Switching backends is explicit and should be rehearsed in non-prod:

```bash
terraform init -migrate-state
```

Terraform compares the previous and new backend; answer prompts carefully. For CI, prefer **partial
configuration** (`-backend-config=...` files or HCL fragments) so secrets never live in tracked
Terraform files.

```bash
terraform init \
  -backend-config="bucket=myorg-terraform-state-us-east-1" \
  -backend-config="key=apps/imu-pipeline/prod/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-locks"
```

## Consistency with remote operations

When using Terraform Cloud with **remote execution**, runs occur on the vendor compute pool; local
`state` subcommands still interact with the same remote record but may require authentication and
organization permissions. Document who may run `state rm` in runbooks—treat it like database admin
access.

## Inspecting state safely

These commands are read-only with respect to infrastructure but may expose secrets if outputs are
not redacted:

```bash
terraform state list
terraform state show 'module.compute.aws_lambda_function.imu_processor'
```

Pair with `terraform show` on saved plan files for audit evidence without mutating state.

## Drift remediation workflow

1. `terraform plan -refresh-only` to let Terraform learn reality.
2. If changes are **acceptable**, follow with a normal `plan` to codify the changes in configuration
   files rather than mutating state alone.
3. If drift is **not acceptable**, fix cloud resources to match Terraform and re-plan.

Avoid editing `terraform.tfstate` by hand unless you are following a documented recovery and have
**checksum awareness** for remote backends.

## Cross-account remote state on AWS

When the state bucket lives in a **logging/security account**, consumers assume a role in `init` via
provider credentials or OIDC. Keep **KMS key policies** aligned so `s3:GetObject` decrypt works
from CI roles in workload accounts. Document the **exact IAM** statements in your landing zone—see
`terraform-multi-account`.

## Performance and blast radius

Large states slow planning. Split stacks along **natural service boundaries** (network, data,
compute) and connect via `terraform_remote_state` or data sources. Very large single stacks also
complicate **`terraform state mv`** operations—migrate incrementally with `moved` blocks and small
PRs.

## Operational calendar for state buckets

Schedule **access reviews** for who can `PutObject` to `terraform.tfstate` prefixes. Enable **S3
access logging** or CloudTrail data events where price allows; alert on deletes and encryption
configuration changes.

## Summary rules

- Prefer **remote backends** with **locking** and **encryption**.
- Treat `state push` and `force-unlock` as **break-glass** with approvals.
- Use **`moved`/`import` blocks** in version control whenever possible so changes are reviewable.
- Export explicit **outputs** for sharing; never rely on reading foreign state for unpublished data.

## Parity checks after incidents

After provider upgrades or emergency hand edits, run:

```bash
terraform providers
terraform state pull | terraform show -json > /tmp/state.snapshot.json
```

Store snapshots according to retention policy. Comparing **`terraform show -json`** outputs before
and after a controversial `state rm` gives auditors a machine-readable trail. Always pair CLI
operations with tickets that reference **addresses** acted upon—future you will thank present you.

When teaching new operators, have them practice **`state list`** and **`state show`** in a sandbox
workspace before granting production `-backend` credentials. Muscle memory for addresses reduces
panic-induced typos during incidents.

Document the **rollback** story for backend migrations: who approves, how long you retain the
previous state key, and which monitors prove the new backend receives fresh writes after cutover.
A rollback without a plan often reintroduces split-brain updates—worse than the original incident.

Label backup objects with **`terraform workspace` + git SHA** when possible—untagged `state.pull`
files become orphan puzzles months later during audits.
