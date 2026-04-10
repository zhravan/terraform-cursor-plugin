---
name: tf-graph
description: Emit a dependency graph in Graphviz DOT format
---

# terraform graph

## Risk class

**Read-only**  -  exposes resource addresses (architecture hints).

## Pre-flight

- `terraform init` for the configuration graph should reflect.

## Confirmation prompts

None, unless exporting diagrams to external parties - review for sensitive names.

## Example invocations

```bash
terraform graph | dot -Tsvg > graph.svg
terraform graph -type=plan
```

## Safe patterns

- Great for **cycle** debugging - pair with `tf-plan` errors.

## Related

- Skill: `skills/terraform-troubleshooting/SKILL.md`
