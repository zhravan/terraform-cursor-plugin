---
name: tf-apply
description: Apply a saved plan or compute-and-apply infrastructure changes
---

# terraform apply

## Risk class

**High**  -  mutates real infrastructure. Mistakes can cause outages, data loss, or security exposure.

## Pre-flight

- Clean `git status` (or intentional hotfix branch) and peer review merged for the change.
- Fresh `terraform plan` reviewed; prefer **`apply plan.bin`** over speculative apply-without-plan in
  production.
- Maintenance window communication if service-impacting.
- Break-glass credentials **not** the default - use CI OIDC roles.

## Confirmation prompts (mandatory)

Ask the human, in explicit language:

1. “This apply targets **workspace X** in directory **Y** with cloud account **Z** - confirm.”
2. “**Destroy count** = N, **replace count** = M - confirm acceptable.”
3. “Post-apply verification: **which metrics/dashboards** will we watch for N minutes?”

Abort if any answer is unclear.

## Example invocations

```bash
terraform apply
terraform apply plan.bin
terraform apply -auto-approve   # CI only, with gated approvals upstream
terraform apply -target=module.network.aws_vpc.this  # break-glass; document why
terraform apply -refresh-only
```

## Safe patterns

- In production CI: **no** `-auto-approve` without required reviewers + environment protection rules.
- After partial failure, **`terraform plan`** again before retry - do not blindly re-run apply.

## Related

- `tf-plan`
- `tf-destroy`  -  even higher risk destructive path
- Skills: `skills/terraform-cicd/SKILL.md`, `skills/terraform-refactoring/SKILL.md`
