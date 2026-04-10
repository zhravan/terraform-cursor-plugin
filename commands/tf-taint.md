---
name: tf-taint
description: Mark a resource instance for forced replacement on next apply (legacy workflow)
---

# terraform taint / untaint

## Risk class

**Medium** — forces **replace** on next apply; can cause downtime if misunderstood.

Prefer **`terraform apply -replace=ADDR`** on modern Terraform versions.

## Pre-flight

- Understand **why** replacement is required (config drift vs bug vs AMI refresh).
- `terraform plan` to see current proposals before adding taint.

## Confirmation prompts

Ask human to confirm **ADDR** and expected **customer impact**.

## Example invocations

```bash
terraform taint 'aws_instance.worker[0]'
terraform apply -replace='aws_instance.worker[0]'   # preferred modern replacement
```

## Safe patterns

- Document taint usage in ticket—`terraform show`/state may reveal tainted bits historically.
- Remove taint mistakes with **`tf-untaint`**.

## Related

- `tf-untaint`
- `tf-apply`
