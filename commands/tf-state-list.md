---
name: tf-state-list
description: List resource addresses tracked in Terraform state (read-only for infrastructure)
---

# terraform state list

## Risk class

**Read-only** for infrastructure. **State data is sensitive**  -  addresses may reveal architecture; some
outputs may reference secret metadata.

## Pre-flight

- Correct backend credentials; verify workspace.

## Confirmation prompts

None for listing. Warn before piping output to public pastebins.

## Example invocations

```bash
terraform state list
terraform state list | grep aws_lambda
terraform state list -state=custom.tfstate   # rare; local dev only
```

## Safe patterns

- Use to scope **`state show` / `mv` / `rm`** operations precisely.

## Related

- `tf-state-show`
- Skills: `skills/terraform-state/SKILL.md`
