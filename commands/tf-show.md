---
name: tf-show
description: Show current state or a saved plan file in human or JSON form
---

# terraform show

## Risk class

**Data exposure**  -  plans and states may include sensitive attributes; **`-json`** especially risky to
paste publicly.

## Pre-flight

- Know whether you’re showing **state** default vs **`plan.bin`**.

## Confirmation prompts

Before uploading `show -json` anywhere: “Redacted for **secrets** and **customer data**?”

## Example invocations

```bash
terraform show
terraform show plan.bin
terraform show -json plan.bin > plan.json
```

## Safe patterns

- Pair **`plan.json`** with **Conftest/OPA** policies in CI (see `skills/terraform-testing/SKILL.md`).

## Related

- `tf-plan`
- `tf-output`
