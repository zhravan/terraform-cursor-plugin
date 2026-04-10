---
name: tf-providers-lock
description: Update the dependency lock file with checksums for provider releases
---

# terraform providers lock

## Risk class

**Medium repository impact**  -  updates `.terraform.lock.hcl`; incorrect platforms can break CI on
exotic architectures.

## Pre-flight

- Decide required **target platforms** (`linux_amd64`, `darwin_arm64`, …).
- Ensure network access to registry or mirrors.

## Confirmation prompts

Before committing lockfile changes: “Which **Terraform/provider versions** triggered this update, and
did we run **`terraform validate`** afterward?”

## Example invocations

```bash
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64
terraform init -upgrade
terraform providers lock
```

## Safe patterns

- In CI for application repos: **`terraform init -lockfile=readonly`** to prevent accidental drift.
- Record **why** lock changed in PR description (CVE, bugfix).

## Related

- `tf-init`
- Skill: `skills/terraform-providers/SKILL.md`, `skills/terraform-security/SKILL.md`
