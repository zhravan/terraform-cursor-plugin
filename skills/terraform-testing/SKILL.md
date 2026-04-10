---
name: terraform-testing
description: >-
  Quality gates for Terraform: native terraform test (.tftest.hcl with run blocks, command=plan vs
  apply, expect failures, assertions, variables, mock_provider), Terratest in Go for integration,
  tflint with community rulesets (including tflint-ruleset-aws), Checkov, Trivy config scanning,
  OPA/Conftest for policy-as-code, and terrascan. Use when designing CI stages or module test harnesses.
---

# Testing and static analysis for Terraform

## Scope

This skill stitches together **fast local feedback**, **CI policy enforcement**, and **optional
integration** tests hitting real clouds. It complements `terraform-cicd` (pipelines) and
`terraform-security` (threat modeling).

## Native `terraform test` (1.6+)

Place tests under `tests/` or alongside modules using **`*.tftest.hcl`** files. Tests orchestrate
**`run`** blocks that execute `plan` or `apply` against **`command = plan`** (default) or
**`command = apply`**.

```hcl
# tests/defaults.tftest.hcl
variables {
  env = "test"
}

mock_provider "aws" {
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "111111111111"
      arn        = "arn:aws:iam::111111111111:root"
    }
  }
}

run "plan_succeeds" {
  command = plan
}

run "apply_critical_path" {
  command = apply

  assert {
    condition     = aws_s3_bucket.telemetry.bucket != ""
    error_message = "Bucket not created"
  }
}
```

**`mock_provider`** prevents real API calls for data sources you stub—critical for fast unit-style
runs. Match mocked types to actual data source names used in modules.

### Assertions

`assert` blocks accept arbitrary boolean expressions referencing resource attributes after **`apply`**
runs (or computed values after `plan` when allowed). Keep assertions **stable**—avoid timestamps.

### Expect failures

Use **`command = plan`** with `expect_failures = [var.bad_input]` (Terraform 1.6+ patterns evolve—
consult docs for your exact version) to assert validation rejects bad variables.

## Terratest (Go)

Terratest wraps Terraform with Go tests—useful for **real AWS/GCP/Azure** smoke tests.

```go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestIMUStackBootstrap(t *testing.T) {
  t.Parallel()

  opts := &terraform.Options{
    TerraformDir: "../examples/default",
    Vars: map[string]interface{}{
      "env": "terratest",
    },
  }

  defer terraform.Destroy(t, opts)
  terraform.InitAndApply(t, opts)

  arn := terraform.Output(t, opts, "stream_arn")
  if arn == "" {
    t.Fatalf("missing stream arn")
  }
}
```

Keep integration tests **short** and **isolated**—dedicated accounts/projects, aggressive cleanup.

## tflint

```bash
tflint --init
tflint --recursive
```

Add **AWS ruleset**:

```hcl
# .tflint.hcl
plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "aws_resource_missing_tags" {
  enabled = true
}
```

Tune severity; fail CI on **errors**, warn on **warnings** initially to avoid blocking all PRs.

## Checkov

```bash
checkov -d . --framework terraform --download-external-modules true
```

Suppress findings with **inline** `checkov:skip` comments sparingly—prefer module fixes. Map failures
to security tickets.

## Trivy (`trivy config`)

```bash
trivy config --severity CRITICAL,HIGH .
```

Good for **misconfiguration** passes parallel to Checkov—correlate duplicates to avoid noise fatigue.

## OPA / Conftest

Write Rego policies asserting tag presence, encryption flags, and banned public resources.

```rego
package main

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  not resource.change.after.versioning
  msg := sprintf("Bucket %v must enable versioning", [resource.name])
}
```

```bash
conftest test plan.json
```

Generate `plan.json` via `terraform plan -out=plan.bin && terraform show -json plan.bin > plan.json`.

## terrascan

```bash
terrascan scan -i terraform -d .
```

Useful in multi-cloud repositories; tune policies to match organizational exceptions.

## terraform fmt and validate (baseline)

CI should always run:

```bash
terraform fmt -check -recursive
terraform init -backend=false -input=false
terraform validate
```

For stacks needing credentials even at validate time, split **modules** (validate without creds) from
**roots**.

## Module examples as tests

Every module should ship **`examples/default`** runnable in CI with **`terraform init -backend=false`**
when possible, or ephemeral backend.

## Performance testing plans

Large monorepos benefit from **directory targeting**:

```bash
terraform -chdir=modules/vpc init -backend=false
terraform -chdir=modules/vpc validate
```

## Test data factories

Keep **`tests/fixtures`** for JSON policies and **small zipped lambdas** used in tests—avoid pulling
multi-GB assets.

## Flaky integration tests

Cloud APIs are eventually consistent—use **retries** in Terratest or mark tests **serial** (`t.Parallel`
carefully). For IoT/Kinesis stacks, consider **localstack** only when representational—not feature
complete.

## Contract tests between modules

**`terraform providers`** and **`terraform graph`** can be snapshotted in CI to catch accidental new
edges—but avoid brittle compares; prefer targeted **`depends_on` reviews** in PR templates.

## When to skip integration tests

Branches that only touch **markdown** should skip slow suites—use path filters in CI (`paths-ignore`).

## Reporting

Publish junit from linters when supported; store **`plan.json`** artifacts securely (they may hint
at resource names).

## Closing

Layer defenses: **fmt/validate/tflint** (minutes), **policy engines** (Checkov/Trivy/Conftest),
**terraform test** (unit-ish), **Terratest** (integration, expensive). Calibrate spend vs risk—pci/sox
orgs invest more in Conftest + signed plans.

## Sample GitHub Actions matrix snippet

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: terraform-linters/setup-tflint@v4
      - run: tflint --init && tflint --recursive
      - uses: bridgecrewio/checkov-action@master
        with:
          directory: .
```

Pair with OIDC cloud auth in separate jobs—see `terraform-cicd`.

## mock_provider pitfalls

Mocks must return **shapes** matching provider schemas—upgrade mocks when provider SDKs change field
types. When mocks drift, tests pass while production plans fail: regenerate mocks after provider bumps.

## terraform test directory layout

Colocate `tests/*.tftest.hcl` for modules; for monorepos, names like `tests/network.tftest.hcl` clarify
scope. Document how developers run a single test file if tooling supports targeting in your version.

## Benchmarking plan duration

Track **`terraform plan` duration** in CI metrics—regressions often trace to new data sources scanning
huge inventories. Optimize with `-refresh=false` in PR plans only when safe (discuss trade-offs with
drift).

## Static registries

Mirror providers in CI using `TF_PLUGIN_CACHE_DIR` to avoid download flakiness—align cache keys with
lock file hashes.

## Documentation tests

If README embeds `terraform-docs` output, CI should fail when `terraform-docs` would rewrite—keeps
examples honest.

## Security scanners overlap

Checkov and Trivy overlap; choose primary ownership per org to reduce duplicate toil. Route exceptions
through the same **risk acceptance** workflow.

## Noise reduction

Start with **HIGH/CRITICAL** only; expand severities as backlog shrinks. Teams that enable all rules
day one often disable entire tools—gradual rollout wins.

## Assertions on destroy

Rarely, test `terraform destroy` in ephemeral envs to catch dependency ordering bugs; run during low
traffic and parallelize cautiously.

## Flake analytics

Log flake rate for integration tests; quarantine consistently flaky tests instead of retry spamming
main—flaky tests erode trust in green builds.

## Terraform Cloud policy (Sentinel) vs Conftest

Organizations on HCP Terraform may implement **Sentinel** policies (vendor-specific language).
`terraform plan` JSON output is similarly consumable—pick **one primary** general policy tool for
OSS-style repos (usually Conftest) and use Sentinel only when mandated by the platform contract.

## Kitchen-Terraform (legacy)

Some enterprises still maintain **Kitchen-Terraform** harnesses wrapping InSpec. If you encounter
them, budget time to migrate to **`terraform test`** or Terratest for simpler contributor UX—
document existing Ruby dependencies clearly until retirement.

## Provider caching in test runners

CI images should pre-install common Terrafrom versions and warm plugin caches. Ephemeral runners pay
cold-start costs on every job unless you layer a slim cache volume.

## Pull request sizing vs tests

Large PRs that touch many roots should **shard** test jobs—matrix by directory using a script that
emits changed Terraform roots from `git diff`.

## Mutation testing for modules

Advanced teams occasionally **perturb** variables in property-style tests (fuzz CIDR inputs) to catch
validation gaps—keep fuzzing offline to avoid API rate limits.

## Load testing infrastructure

After Terraform provisions load envs, hand off to **k6** or **Locust** pipelines—still "testing" but
outside Terraform; mention handoffs in module README so SRE knows ownership boundaries.

## Sample `terraform test` command

```bash
terraform test -filter=tests/defaults.tftest.hcl
```

Flags evolve—consult `terraform test -help` for your version; some releases add **verbose** logging or
**junit** reporting experiments.

## Evaluating new scanners

Pilot scanners on **one repo** first; normalize rule IDs and map them to **CIS** controls when
auditors ask for traceability—see `terraform-security`.

## Infracost and cost regression tests

Optional **Infracost** steps compare PR costs to main—treat cost JSON as another artifact requiring
review. Pair with **FinOps-approved instance types** enforced via tflint or Conftest to catch expensive
skus before they merge.

## Terratest retry helpers

Terratest provides **retry** helpers for intermittent AWS errors—use exponential backoff caps to
avoid burning minutes waiting on misconfiguration that will never succeed.

## terraform console in tests

For complex `locals`, developers can script **`terraform console`** with heredoc inputs in CI to
assert expression outputs—lighter than full plans when validating pure math.

## Splitting unit vs integration jobs

GitHub Actions: `needs: [lint]` gates integration jobs; require **manual approval** for integration in
forked PRs to prevent crypto-mining abuse. Document this security posture for open-source Terraform
modules you publish.

## Verifying provider lock files in PRs

Add a job `terraform providers lock -check` (orPlatform list) ensuring drift cannot merge silently.
When upgrading providers, require explicit **`BREAKING`** label in PR templates.

## Test naming conventions

Prefix run blocks with `test_` or environment names consistently—future engineers should understand
failure context from logs without reading HCL.

## Handling large plan files

Compress `plan.json` with `gzip` before uploading artifacts; parsers in policy steps should stream
parse to reduce memory spikes on CI workers.

## Accessibility of tests

Prefer **`terraform test`** for contributors who do not know Go; maintain at least one Go integration
test only where native testing cannot cover cloud behavior.

## Closing checklist for module authors

- `examples/` init in CI
- `terraform test` with mocks for expensive data sources
- `tflint` + `checkov` clean at agreed severities
- Document how to run tests locally in CONTRIBUTING
