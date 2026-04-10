---
name: tf-workspace
description: Manage Terraform workspaces for a given backend configuration
---

# terraform workspace

## Risk class

**High potential for human error**  -  selecting wrong workspace can **plan against prod state** while
believing you’re in dev.

## Pre-flight

- Understand whether your team standardizes on **directories per env** instead of workspaces.

## Confirmation prompts

Before `select` on shared laptop:

“Read aloud the output of **`terraform workspace show`** after selection.”

## Example invocations

```bash
terraform workspace list
terraform workspace select prod
terraform workspace new staging
terraform workspace delete staging
```

## Safe patterns

- **Never** rely on workspace alone for hard prod/stage isolation in regulated environments - separate
  state backends/keys are safer.
- Pin **`TF_WORKSPACE`** in CI explicitly.

## Related

- Skills: `skills/terraform-state/SKILL.md`, `skills/terraform-troubleshooting/SKILL.md`
