---
name: tf-state-rm
description: Remove resource instances from Terraform state without destroying cloud objects
---

# terraform state rm

## Risk class

**High** — causes Terraform to **forget** resources; next apply may attempt to **recreate** or
**adopt** incorrectly if config still exists.

## Pre-flight

- `terraform state pull` backup.
- Written rationale ticket: why orphaning state is correct.
- Understand whether **cloud resource** should continue to exist unmanaged.

## Confirmation prompts (mandatory)

Ask:

1. “Confirm address **`ADDR`** to remove from state only?”
2. “Confirm **cloud object will NOT be deleted** by this command—but **may** be deleted later if
   config still declares it—what is the follow-up?”
3. “Two-person review completed?” (prod)

## Example invocations

```bash
terraform state rm 'aws_security_group.temp'
terraform state rm 'module.legacy.aws_s3_bucket.a'
```

## Safe patterns

- Immediately run **`terraform plan`** to understand future actions.
- Consider **`removed` blocks** when Terraform version supports declarative workflow.

## Related

- `tf-state-mv`
- Skills: `skills/terraform-state/SKILL.md`, `skills/terraform-troubleshooting/SKILL.md`
