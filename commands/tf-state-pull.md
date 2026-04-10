---
name: tf-state-pull
description: Download and print the current remote Terraform state (JSON) to stdout
---

# terraform state pull

## Risk class

**Critical data exposure** — output contains resource attributes; may include sensitive values. Does not
mutate state on its own.

## Pre-flight

- Ensure output is redirected to **encrypted** storage if saved (`> file`).
- Confirm you are authorized to read full state.

## Confirmation prompts

Before saving to disk: “Where will this file be stored, who can access it, and when will it be
deleted?”

## Example invocations

```bash
terraform state pull > tfstate.backup.$(date +%s).json
terraform state pull | jq '.resources[] | select(.type=="aws_s3_bucket")'
```

## Safe patterns

- Never attach raw state to public tickets.
- Prefer **`terraform state list/show`** for targeted inspection when sufficient.

## Related

- `tf-state-push` — inverse operation, extremely dangerous
- Skills: `skills/terraform-state/SKILL.md`
