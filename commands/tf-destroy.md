---
name: tf-destroy
description: Destroy all resources managed by the current Terraform state and configuration
---

# terraform destroy

## Risk class

**Critical / irreversible** for many resource types. Assumes **correct workspace** - disasters have
destroyed production due to wrong `TF_WORKSPACE`.

## Pre-flight

- **Extra** peer review (two-person rule for regulated environments).
- Confirm `terraform workspace show` output.
- Confirm backend key / remote workspace name matches intention.
- Capture **`terraform state list`** and **`terraform plan -destroy`** artifacts.
- Ensure backups for stateful resources exist outside Terraform when required (DB snapshots, etc.).

## Confirmation prompts (mandatory)

Ask:

1. “Confirm **environment name** and **cloud account/project/subscription ID**.”
2. “Type **the full workspace name** you expect to destroy.”
3. “List **business owner** who approved destroy and ticket id.”
4. “Confirm **no shared** resources (DNS, transit, logging) will strand other teams.”

If human hesitates, **abort**.

## Example invocations

```bash
terraform plan -destroy -out=destroy.bin
terraform apply destroy.bin
terraform destroy                  # interactive - avoid in CI without strong guards
terraform destroy -target=...      # dangerous partial destroy - last resort with design doc
```

## Safe patterns

- Prefer **`plan -destroy` → review → `apply destroy.bin`** over ad-hoc `destroy`.
- After destroy verification, consider **`terraform workspace delete`** only when appropriate.

## Related

- `tf-plan`
- `tf-apply`
- Skills: `skills/terraform-core/SKILL.md` (`prevent_destroy`), `skills/terraform-security/SKILL.md`
