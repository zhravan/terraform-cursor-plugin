---
name: tf-state-push
description: Upload a local state file to the configured backend (overwrite remote state)
---

# terraform state push

## Risk class

**Catastrophic**  -  can **overwrite** remote state, causing loss of truth, recreation storms, or merge
of incompatible histories.

## Pre-flight

- Break-glass approval process documented.
- Verified **`-force`** implications (if used) per documentation.
- Backup current remote state to encrypted object beforehand (`state pull`).

## Confirmation prompts (mandatory)

Ask human to type exact **backend workspace/key** and provide **incident ticket**. Require verbal or
typed confirmation: “I understand this may **overwrite production state**.”

Abort if not an **actual recovery** situation.

## Example invocations

```bash
terraform state pull > pre-incident.json
# only after incident commander approval:
terraform state push tfstate.repaired.json
```

## Safe patterns

- Default stance: **do not run** outside recovery drills or documented procedures.
- After push, run **`terraform plan`** immediately - expect careful analysis.

## Related

- `tf-state-pull`
- Skills: `skills/terraform-state/SKILL.md`, `skills/terraform-troubleshooting/SKILL.md`
