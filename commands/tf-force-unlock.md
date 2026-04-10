---
name: tf-force-unlock
description: Manually release a remote Terraform state lock
---

# terraform force-unlock

## Risk class

**Critical** — can allow **concurrent writes** to state, corrupting it, if a live operation still
holds the lock legitimately.

## Pre-flight

- Verify **no** running `apply` / CI job / Terraform Cloud run owns the lock.
- Capture lock **ID** precisely from error message.
- Incident ticket + two-person review for production.

## Confirmation prompts (mandatory)

Ask human:

1. “Show me the **`terraform plan/apply` output** proving the lock holder crashed or was cancelled.”
2. “Paste **Lock ID** you intend to unlock—read it back aloud.”
3. “Who is on-call if this goes wrong?”

If uncertain, **wait** and investigate remote backend lock table/ UI.

## Example invocations

```bash
terraform force-unlock 1234567890abcdef
```

## Safe patterns

- After unlock, run **`terraform plan`** before **apply** to ensure consistent view.

## Related

- Skill: `skills/terraform-troubleshooting/SKILL.md`, `skills/terraform-state/SKILL.md`
