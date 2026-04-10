---
name: tf-untaint
description: Remove the tainted flag from a resource instance
---

# terraform untaint

## Risk class

**Low**  -  removes **replace marker** from state; does not change cloud resource directly.

## Pre-flight

- Confirm resource truly should **not** be replaced anymore.

## Confirmation prompts

Ask: “Confirm **ADDR** should no longer be replaced on next apply?”

## Example invocations

```bash
terraform untaint 'aws_instance.worker[0]'
```

## Related

- `tf-taint`
- `tf-plan`
