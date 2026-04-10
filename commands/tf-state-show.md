---
name: tf-state-show
description: Show attributes for one resource instance from Terraform state
---

# terraform state show

## Risk class

**Read-only** for infrastructure. Output can include **sensitive attributes** depending on provider and
`-json` usage.

## Pre-flight

- Known good address from `terraform state list`.

## Confirmation prompts

Before sharing output externally, confirm scrubbing of secrets.

## Example invocations

```bash
terraform state show 'aws_kinesis_stream.imu_ingest'
terraform state show -json 'module.compute.aws_lambda_function.this'
```

## Safe patterns

- Prefer **narrow** addresses—avoid dumping entire state; use `state pull` + jq with care.

## Related

- `tf-state-list`
- `tf-show` — for saved plans / human-readable state snapshots
