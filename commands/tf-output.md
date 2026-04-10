---
name: tf-output
description: Query root module outputs after apply
---

# terraform output

## Risk class

**Read-only** — output values may be marked **sensitive**; still appear depending on flags/version.

## Pre-flight

- State exists and resources applied (outputs evaluate from state/modules).

## Confirmation prompts

Before printing **sensitive** outputs to screens in shared meetings, ask if redacted form suffices.

## Example invocations

```bash
terraform output
terraform output -raw database_url
terraform output -json
```

## Safe patterns

- Prefer **automation** consuming `-json` with secret hygiene (CI secret store), not chat logs.

## Related

- `tf-show`
- Skill: `skills/terraform-core/SKILL.md` (sensitive outputs)
