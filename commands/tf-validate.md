---
name: tf-validate
description: Validate Terraform configuration syntax and internal consistency
---

# terraform validate

## Risk class

**Read-only** — does not touch cloud APIs directly (still may load provider schemas).

## Pre-flight

Run **`terraform init`** first (often with **`-backend=false`** in module CI) so providers are installed.

## Confirmation prompts

None.

## Example invocations

```bash
terraform init -backend=false -input=false
terraform validate
terraform -chdir=modules/vpc validate
```

## Safe patterns

- Add to **every** CI pipeline before `plan`.
- When validate fails after provider upgrade, read the **first** error block—subsequent errors may be
  cascades.

## Related

- `tf-init`
- `tf-fmt`
