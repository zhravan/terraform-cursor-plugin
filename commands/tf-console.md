---
name: tf-console
description: Interactive console for evaluating Terraform expressions in module context
---

# terraform console

## Risk class

**Low** directly — but expressions may **read sensitive variables** from loaded tfvars; history files
may leak on shared machines.

## Pre-flight

- `terraform init` completed for the module you want to evaluate.

## Confirmation prompts

If pasting production tfvars into console, warn about **shell history** and **terminal logging**.

## Example invocations

```bash
terraform console
terraform console <<< 'cidrsubnet("10.0.0.0/16", 4, 2)'
```

## Safe patterns

- Use for **pure locals** debugging; avoid production secrets in interactive sessions.

## Related

- Skill: `skills/terraform-core/SKILL.md`
