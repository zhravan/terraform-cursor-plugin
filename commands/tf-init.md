---
name: tf-init
description: Initialize a working directory, configure backends, and install providers/modules
---

# terraform init

## Risk class

**Low** (local metadata). Still touches **provider/module downloads** and may **migrate state** when
backend config changes.

## Pre-flight

- You are in the correct directory or will use `-chdir`.
- Network access to registry and backend is available (unless using air-gapped mirrors).
- Terraform version matches `required_version` in root module.

## When reconfiguration is needed

After changing **backend**, **provider source**, or **module sources**, rerun `init` with appropriate
flags.

## Confirmation prompts (agent-facing)

- Before `terraform init -migrate-state`, ask the human to confirm **source** and **destination** backends and that a maintenance window is acceptable.
- Before `terraform init -reconfigure`, confirm this will not unintentionally abandon prior backend metadata on shared workstations.

## Example invocations

```bash
terraform init
terraform -chdir=infra/live/prod init -input=false
terraform init -upgrade
terraform init -migrate-state
terraform init -backend-config=prod.backend.hcl
terraform init -lockfile=readonly
```

## Safe patterns

- Use **`-input=false`** in CI once variables are wired.
- Prefer **partial backend config files** for secrets (`bucket`, `key`) instead of committing secrets.
- Run **`terraform providers lock`** after intentional provider upgrades (see `tf-providers-lock`).

## Related

- `tf-plan`  -  after successful init
- `tf-providers-lock`  -  pin checksums across platforms
- Skill: `skills/terraform-providers/SKILL.md`, `skills/terraform-state/SKILL.md`
