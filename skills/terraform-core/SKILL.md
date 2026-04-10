---
name: terraform-core
description: >-
  Terraform core language and root-module patterns. Use for HCL syntax, types and type constraints,
  expressions and built-in functions, meta-arguments (for_each, count, dynamic), lifecycle rules,
  moved/removed/import configuration blocks, depends_on, locals, variable validation, sensitive values,
  and terraform blocks including required_version and required_providers. Prefer this skill before
  cloud-specific skills when the question is about language semantics or portable configuration.
---

# Terraform core language

## Scope

This skill covers the **Terraform configuration language** (HCL): how you declare resources, wire
data between them, and constrain Terraform and provider versions. It does **not** replace
provider-specific resources (see cloud skills) or state backends (see `terraform-state`).

## Terraform block and versioning

Every root module should declare which **Terraform CLI** versions are supported and pin **provider**
sources. Use `required_version` to prevent accidental upgrades on developer laptops and CI runners.

```hcl
terraform {
  required_version = ">= 1.9.0, < 1.12.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Rules of thumb:

- Prefer **lower+upper** bounds (`>= 1.9, < 1.12`) over a single equality so patch releases flow.
- Couple language features to versions: e.g. certain `ephemeral` resource behaviors need modern
  Terraform; `check` blocks and `test` have minimum versions—consult release notes when adopting.

## Types and variable constraints

Variables accept **primitive** types (`string`, `number`, `bool`), **collection** types (`list`,
`set`, `map`), and **structural** types (`object`, `tuple`, `any`). Use **object** types for
structured inputs instead of untyped maps when you want IDE clarity and validation.

```hcl
variable "service" {
  type = object({
    name = string
    port = number
    tags = optional(map(string), {})
  })
  description = "Service descriptor used by compute and LB resources."
}

variable "cidr_allow" {
  type        = list(string)
  description = "CIDR blocks allowed to reach the ALB."

  validation {
    condition = alltrue([
      for c in var.cidr_allow : can(cidrhost(c, 0))
    ])
    error_message = "Each entry in cidr_allow must be a valid CIDR string."
  }
}
```

`optional()` inside object types (Terraform 1.3+) lets callers omit keys while you supply defaults
with the second argument to `optional`.

## Locals and expression composition

**Locals** deduplicate expressions and name intermediate values. Prefer locals over repeating the
same `merge()` or string templates across many resources.

```hcl
locals {
  base_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }

  name_prefix = "${var.project}-${var.environment}"

  # Example: normalize AZs to subnet maps elsewhere
  az_to_index = { for i, az in var.availability_zones : az => i }
}
```

Use **splat** (`[*]`) and **`for` expressions** to reshape collections. `merge(flatten([...]))` is
common for building tag maps across modules.

## Sensitive values and outputs

Mark variables and outputs **sensitive** to reduce accidental logging. This does **not** encrypt
state—it only affects CLI redaction in human-oriented output.

```hcl
variable "api_key" {
  type        = string
  sensitive   = true
  description = "External API key; never commit."
}

output "database_url" {
  value       = aws_db_instance.app.address
  sensitive   = true
  description = "DB hostname; still stored in state."
}
```

Secrets still land in **state**. See `terraform-secrets` for external secret stores, ephemeral
patterns, and write-only attributes where applicable.

## Meta-arguments: count vs for_each

**`count`** indexes resources with `count.index`. It forces replacement when indices shift (e.g.
shortening a list breaks alignment). **`for_each`** maps resources to keys (set or map) and is the
default choice for **address-stable** fan-out.

```hcl
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, index(var.availability_zones, each.key))
  availability_zone = each.key
}
```

Use **`count`** when you truly need a numeric index or a simple boolean toggle (`count =
var.enable_nat ? 1 : 0`). When in doubt, pick **`for_each`**.

## Dynamic blocks

**`dynamic`** generates nested blocks from collections. The label is the **block type name** inside
the resource (e.g. `ingress` on a security group).

```hcl
resource "aws_security_group" "app" {
  name   = "app"
  vpc_id = aws_vpc.this.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      description = ingress.value.description
      from_port   = ingress.value.from
      to_port     = ingress.value.to
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidrs
    }
  }
}
```

Avoid `dynamic` when a resource already accepts a **list** attribute—prefer native list attributes
when available for readability.

## Lifecycle meta-argument

### prevent_destroy

Blocks Terraform from planning a destroy (useful for stateful data stores). Combine with IAM to
stop console deletes when needed.

```hcl
resource "aws_db_instance" "app" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

### ignore_changes

Stops drift reconciliation for listed attributes—use sparingly when an external system mutates
tags or auto-scaling shapes and you accept Terraform ignoring those diffs.

```hcl
resource "aws_instance" "bastion" {
  # ...
  lifecycle {
    ignore_changes = [ami]
  }
}
```

### create_before_destroy

Orders replacement so the new object is created first when the provider supports it—critical for
zero-downtime when type-specific constraints allow.

```hcl
resource "aws_lb_target_group" "api" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```

### replace_triggered_by (Terraform 1.2+)

Forces replace when **other resource instances** change, even if argument diffs alone would not.

```hcl
resource "aws_instance" "worker" {
  # ...
  lifecycle {
    replace_triggered_by = [
      aws_launch_template.worker.latest_version
    ]
  }
}
```

## depends_on

Terraform infers dependencies from references. Add **`depends_on`** only when you need an **ordering**
constraint without referencing attributes—common with IAM attachments or eventual consistency edges.

```hcl
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"

  depends_on = [aws_iam_role.lambda_exec]
}
```

## moved and removed blocks (configuration-driven state migration)

**`moved`** renames addresses in state without destroy/create. Prefer this over `terraform state mv`
when you can encode history in config for reviewers.

```hcl
moved {
  from = aws_security_group.old
  to   = module.network.aws_security_group.app
}
```

**`removed`** tells Terraform to stop managing an object while optionally leaving it running
(`destroy = false`).

```hcl
removed {
  from = aws_s3_bucket.legacy_uploads

  lifecycle {
    destroy = false
  }
}
```

## import blocks (Terraform 1.5+)

Declare **import** in configuration to bring existing infrastructure under management with code
review. Pair with a matching `to` resource block.

```hcl
import {
  to = aws_iam_role.imported_ops
  id = "ops-readonly"
}

resource "aws_iam_role" "imported_ops" {
  name = "ops-readonly"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::111111111111:root" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

## Built-in functions (patterns)

Common families:

- **Strings**: `format`, `formatlist`, `replace`, `regex`, `split`, `join`
- **Collections**: `length`, `lookup`, `merge`, `concat`, `distinct`, `flatten`, `tolist`, `toset`
- **Encoding**: `jsonencode`, `jsondecode`, `yamlencode`, `base64encode`
- **Network**: `cidrsubnet`, `cidrhost`
- **Filesystem** (use sparingly in root modules): `file`, `templatefile`, `filebase64`

Example combining `templatefile` with clean separation of policy JSON:

```hcl
data "aws_iam_policy_document" "lambda_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda" {
  name               = "imu-processor-lambda"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume.json
}
```

## Practical module interfaces

Expose **small**, **typed** variable sets; return **outputs** that downstream stacks need (IDs, ARNs,
DNS names). Keep provider configuration at the **root**; children receive `configuration_aliases`
when multiple regions or accounts are required—see `terraform-providers` and `terraform-modules`.

## When not to use this skill

- Backend or state encryption details → `terraform-state`.
- AWS resource specifics → `terraform-aws`.
- Testing (`terraform test`, policies) → `terraform-testing`.

## Failure modes

- **Type errors** on `for_each`: ensure the collection is a **map** or a **set of strings**, not a
  list of objects (wrap with `{ for x in var.items : x.name => x }`).
- **Cycle errors**: break cycles with `-target` only as a last resort; usually reshape dependencies
  or split stacks—see `terraform-troubleshooting`.

## More expressions: try, can, and null handling

Use **`try`** for graceful fallbacks when a deep key might be absent; prefer typed **`object`**
variables so absence is caught earlier at plan time.

```hcl
locals {
  # Safe nested lookup with default
  log_level = try(var.observability.log_level, "INFO")
}
```

**`can`** pairs with `for`/`if` for validation and conditional logic without tripping evaluation:

```hcl
locals {
  valid_ports = [
    for p in var.ports : p if can(tonumber(p)) && tonumber(p) > 0 && tonumber(p) <= 65535
  ]
}
```

Treat **`null`** as “unset” in attributes that accept it; merging maps often uses `compact()` or
`{ for k, v in var.m : k => v if v != null }` to drop empty values.

## String templates and heredocs

**Interpolation** `${}` embeds expressions; **directives** `%{if}` enable inline conditionals inside
strings. For multi-line blobs (JSON policies, startup scripts), prefer `<<-EOF` heredocs and
indentation stripping.

```hcl
locals {
  user_data = <<-EOT
    #!/bin/bash
    set -euo pipefail
    echo "BOOTSTRAP_ENV=${var.environment}" >/etc/instance.env
  EOT
}
```

`templatefile(path, vars)` keeps large templates out of `.tf` files and enables reuse across
modules.

## Provider and resource identity recap

Each **resource** has a type and a logical name: `aws_s3_bucket.audit_logs`. **Data sources** use
`data.` prefix in references. **Provisioners** (`local-exec`, `remote-exec`) are a last resort—prefer
native APIs, user-data, or pipeline jobs.

## Provider meta-arguments on resources

Besides `for_each`, `count`, and `lifecycle`, resources can set **provider aliases** to target
alternate configurations. Always pass provider mappings explicitly in modules that need multi-region
or multi-account parallelism—see `terraform-providers` for `configuration_aliases`.

Example pattern:

```hcl
resource "aws_security_group" "peer" {
  provider  = aws.us_west_2
  name      = "peer-link"
  vpc_id    = aws_vpc.west.id
}
```

## Module version constraints

Inside `module` blocks, `version = "..."` pins **Terraform Registry** modules. For local paths, omit
version and use `source = "../modules/foo"`. Tag Git modules with **semver** and reference
`ref` pins—details in `terraform-modules`.

## Formatting and readability

Run `terraform fmt` on every change. For large `object` variables, align keys vertically in teams
that adopt style guides—Terraform accepts either style, but consistency beats cleverness.
