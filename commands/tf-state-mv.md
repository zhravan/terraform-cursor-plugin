---
name: tf-state-mv
description: Move resources in Terraform state to new addresses (rename / module moves)
---

# terraform state mv

## Risk class

**High** — can corrupt mapping between state and config if used carelessly. Prefer **`moved` blocks**
when possible for reviewability.

## Pre-flight

- `terraform state pull > backup-$(date +%s).json`
- Peer review / ticket linked.
- Plan understanding of **FROM** and **TO** addresses (`terraform state list`).

## Confirmation prompts

Ask human:

1. “Confirm **FROM** and **TO** addresses verbatim—read aloud.”
2. “Why not use a **`moved` block** in this situation?”
3. “Confirm **no concurrent applies** are running.”

## Example invocations

```bash
terraform state mv 'aws_lambda_function.old' 'aws_lambda_function.new'
terraform state mv 'aws_instance.i' 'module.compute.aws_instance.i'
```

## Safe patterns

- Run **`terraform plan`** immediately after—expect no unexpected destroys for a pure rename.
- Document CLI move in ticket when configuration cannot yet include `moved` blocks.

## Related

- Skills: `skills/terraform-refactoring/SKILL.md`, `skills/terraform-core/SKILL.md`
