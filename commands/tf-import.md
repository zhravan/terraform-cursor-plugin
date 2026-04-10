---
name: tf-import
description: Import existing infrastructure into Terraform state (CLI workflow)
---

# terraform import

## Risk class

**Medium** — mutates **state only**, but wrong ID causes **wrong** mapping and future destructive plans.

Prefer modern **`import` blocks** in `.tf` files when workflow allows (see skill `terraform-core`).

## Pre-flight

- Resource configuration block exists and matches reality **closely enough** for provider import
  semantics.
- You have correct **import ID** format from provider documentation.
- Snapshot state: `terraform state pull > backup-$(date +%s).json`.

## Confirmation prompts

Ask human:

1. “Confirm **resource address** `ADDR`?”
2. “Confirm **import id** / ARN / name?”
3. “What **follow-up plan** will reconcile configuration drift after import?”

## Example invocations

```bash
terraform import 'aws_s3_bucket.logs' my-log-bucket-name
terraform import 'module.vpc.aws_vpc.this' vpc-0abc123
terraform import 'aws_lambda_function.imu' imu-processor
```

## Safe patterns

- Immediately run **`terraform plan`** — expect diffs; adjust config until plan is safe.
- Avoid mass imports without **runbook**—batch per resource type.

## Related

- Skills: `skills/terraform-core/SKILL.md` (import blocks), `skills/terraform-refactoring/SKILL.md`
