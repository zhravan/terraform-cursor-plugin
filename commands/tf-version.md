---
name: tf-version
description: Print Terraform CLI version metadata
---

# terraform version

## Risk class

**None** — informational.

## Pre-flight

None.

## Confirmation prompts

None.

## Example invocations

```bash
terraform version
terraform version -json
```

## Safe patterns

- CI should print version at job start for **forensics**.
- Compare with **`required_version`** constraints in root module.

## Related

- Skills: `skills/terraform-opentofu/SKILL.md` (toolchain context)
