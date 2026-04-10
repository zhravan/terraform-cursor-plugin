---
name: tf-plan
description: Compute and optionally save an execution plan without applying infrastructure changes
---

# terraform plan

## Risk class

**Read-mostly** to cloud APIs (refresh) but can **expose sensitive values** in logs if misconfigured.
Does **not** mutate infrastructure by default.

## Pre-flight

- `terraform init` completed successfully.
- Correct workspace selected (`terraform workspace show`).
- Credentials scoped for **read** if using least-privilege plan roles.

## Confirmation prompts

- Before sharing plan output externally, confirm **no secrets** leaked in resource attributes.
- If plan proposes **unexpected destroys**, stop and escalate—do not immediately proceed to apply.

## Example invocations

```bash
terraform plan
terraform plan -out=plan.bin
terraform plan -var-file=prod.tfvars
terraform plan -destroy
terraform plan -refresh-only
terraform plan -replace='module.compute.aws_instance.broken[0]'
```

## Safe patterns

- Always archive **`plan.bin`** for audited applies where required.
- Use **`-refresh-only`** to understand drift before normal plans.
- Avoid habitual **`-target`** — it hides dependency issues (break-glass only).

## Related

- `tf-apply` — consumes `plan.bin`
- `tf-show` — human/JSON view of saved plan
- Skill: `skills/terraform-core/SKILL.md`, `skills/terraform-troubleshooting/SKILL.md`
