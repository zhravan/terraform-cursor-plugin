---
name: tf-fmt
description: Format Terraform configuration files to canonical style
---

# terraform fmt

## Risk class

**Low** — rewrites `.tf` / `.tfvars` files in working tree (review diff in VCS).

## Pre-flight

- Commit or stash unrelated edits if you need a clean diff.
- Ensure CI uses **`terraform fmt -check`** on PRs.

## Confirmation prompts

None required for routine formatting. If running against a huge legacy repo, warn the human that a
large diff is upcoming.

## Example invocations

```bash
terraform fmt
terraform fmt -recursive
terraform fmt -check -recursive
terraform fmt -diff
```

## Safe patterns

- Prefer **`-recursive`** in modules/monorepos.
- Pair with **pre-commit** hooks to avoid style debates in PR comments.

## Related

- `tf-validate`
