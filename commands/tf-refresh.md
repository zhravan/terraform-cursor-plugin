---
name: tf-refresh
description: Update Terraform state with real infrastructure attributes (legacy command)
---

# terraform refresh

## Risk class

Historically **mutated state only**; in modern Terraform, prefer **`terraform apply -refresh-only`** /
**`terraform plan -refresh-only`** patterns - verify your version’s documentation.

## Pre-flight

- Understand you are reconciling **Terraform’s picture of reality** - may prelude larger plans.

## Confirmation prompts

Before refresh during incidents: “Are we confident this won’t hide the need for a broader `plan`?”

## Example invocations

```bash
terraform apply -refresh-only
terraform plan -refresh-only
```

## Safe patterns

- Capture **before/after** state snapshots when debugging mysterious diffs.

## Related

- `tf-plan`
- Skill: `skills/terraform-state/SKILL.md`
