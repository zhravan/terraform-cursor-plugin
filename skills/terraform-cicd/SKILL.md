---
name: terraform-cicd
description: >-
  CI/CD patterns for Terraform: GitHub Actions and GitLab CI examples with OIDC to clouds (avoid
  long-lived keys), plan-as-PR-comment flows, storing sanitized plan artifacts and gated apply jobs,
  approval gates for production, scheduled drift detection, and third-party runners (Atlantis,
  Terraform Cloud/Enterprise, Spacelift, env0). Use when designing pipelines or comparing orchestrators.
---

# Terraform CI/CD orchestration

## Scope

Covers **how** Terraform runs in automation—not individual resources. Pair with `terraform-testing`
for linters and `terraform-security` for IAM policies.

## GitHub Actions + AWS OIDC (no long-lived keys)

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/terraform-plan
          aws-region: us-east-1
      - run: terraform -chdir=infra init -input=false
      - run: terraform -chdir=infra plan -input=false -out=plan.bin
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: infra/plan.bin
```

Use **separate** `role-to-assume` for `apply` job with tighter trust + environment protection rules.

### Plan as PR comment

Use actions like `robburger/terraform-pr-commenter` or custom scripts parsing `terraform show` text.
Scrub secrets—plans can still leak resource names; post **sanitized** snippets.

### Apply job with approval

```yaml
apply:
  needs: plan
  environment: production
  runs-on: ubuntu-latest
  steps:
    - uses: actions/download-artifact@v4
      with:
        name: tfplan
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::111111111111:role/terraform-apply
        aws-region: us-east-1
    - run: terraform -chdir=infra apply -input=false plan.bin
```

GitHub **Environments** add required reviewers—use them for prod.

## GitLab CI OIDC (AWS sketch)

```yaml
plan:
  image: hashicorp/terraform:1.9.4
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://gitlab.com
  script:
    - terraform -chdir=infra init -input=false
    - terraform -chdir=infra plan -input=false -out=plan.bin
  artifacts:
    paths:
      - infra/plan.bin
```

Configure AWS `WebIdentity` trust for GitLab’s issuer URL and **subject** claims matching project +
branch.

## Drift detection cron

Nightly `terraform plan -detailed-exitcode` (or modern equivalent exit semantics) in read-only
mode—open issues when drift ≠ 0. Pair with **`terraform plan -refresh-only`** depending on playbook.

## Atlantis

**Pros:** PR comments native, project-level workflows. **Cons:** operate a long-lived service—secure
with SSO, network policies, and separate apply secrets per repo.

Key ideas:

- **`atlantis.yaml`** declares project roots.
- Use **repo-level** workflows to enforce `plan` on every PR, `apply` after merge or explicit comment
  based on policy.

## Terraform Cloud / Enterprise

Remote execution, cost estimation integrations, policy sets, VCS-driven runs. Good when you want
**SaaS orchestration** instead of self-hosted runners—evaluate data residency and RBAC.

## Spacelift

Policy-as-code, stack dependencies, and drift scheduling with managed workers—compare pricing vs
engineering cost of self-run Actions.

## env0

Similar managed orchestration with environment TTL for ephemeral stacks—great for preview envs.

## Artifact hygiene

- Store `plan.bin` encrypted with short retention.
- Never attach raw state to artifacts.
- Sign `plan.json` if compliance demands non-repudiation.

## Monorepo strategies

- `terraform -chdir` matrices based on `git diff` paths.
- **Terragrunt** `run-all` pipelines—see `terraform-terragrunt`.

## Pull request sizing

Large PRs obscure risk—enforce **stack-sized** changes with design docs when edits exceed N resources.

## Rollbacks

Terraform lacks automatic rollback—keep **previous plan artifacts** and git commits ready. Some teams
maintain **`terraform apply` from earlier commit** runbooks; test in staging.

## Canary stacks

For risky refactors, duplicate stack with **smaller scope** (new workspace) and shift traffic using
app-level controls—destroy old stack after validation.

## Windows runners

Avoid unless necessary—shell differences cause friction; prefer Linux containers.

## Secrets in CI

Inject via OIDC-derived roles and cloud secret managers—not repository variables for powerful keys.

## Self-hosted runners hardening

Isolate runners in **private subnets**, patch OS, rotate registration tokens, restrict which repos may
use each runner pool.

## Cost of concurrency

Prevent overlapping applies with **queueing** (Terraform Cloud) or **mutex** steps in Actions
(action-github-locks, etc.).

## Observability

Export Terraform logs to centralized logging; tag with commit SHA and workspace.

## When not to use this skill

Troubleshooting failed applies → `terraform-troubleshooting`. Writing modules → `terraform-modules`.

## Pipeline failure triage order

1. Init failure → credentials/provider mirror/backend.
2. Plan error → syntax, provider schema drift, var missing.
3. Apply error → API quotas, eventual consistency, dependency.

## Example: plan artifact verification

Before apply, compute `terraform show -json plan.bin | sha256sum` and compare to review comment hash
to prevent tampered plans.

## BONUS: Slack notifications

Post compact diff summaries to Slack **after** plan success; include VCS link and author—avoid raw
secrets.

## Enterprise change windows

Coordinate applies with **change advisory boards** when touching backbone networking—CI automation
should still respect blackout periods via calendars or manual gates.

## Documentation debt

Maintain **`RUNBOOK.md`** in infra repos with exact CI job names, emergency bypass steps (discouraged),
and who to page—future incidents move faster.

## Rate limits

Cloud control planes throttle; parallel matrix plans can exhaust APIs—throttle concurrency per account.

## Terraform version pinning

Pin **exact** Terraform patch in CI; upgrade deliberately after reading release notes—minor bumps have
broken providers historically.

## Final guidance

Pick orchestration based on **team size**, **compliance**, and ** Desire for self-hosting**:
GitLab/GitHub native for most, Terraform Cloud/Spacelift when governance features justify spend,
Atlantis when PR-centric flows dominate engineering culture.

## GitHub merge queue considerations

When using merge queues, ensure Terraform jobs re-run on the **merged SHA**, not stale PR head—
prevents applying plans generated before another merge landed.

## GitLab parent/child pipelines

Split **lint** child from **apply** parent to speed feedback; pass `plan.bin` via dotenv artifacts
carefully with size.

## Cloud-specific OIDC examples (pointers)

- AWS: IAM role trust policy with **`token.actions.githubusercontent.com`** audience + sub patterns.
- Azure: federated credentials on app registrations.
- GCP: workload identity pools binding provider attributes to service accounts.

Document subject claims in internal wiki—incorrect `sub` strings are the top misconfiguration.

## Ephemeral environment teardown

Schedule `terraform destroy` for preview envs using CI TTL jobs—tag resources for auto-cleanup; avoid
orphaned NAT gateways draining budgets.

## Build once, plan many

Cache `terraform init` plugin dir per lock hash across matrix jobs to reduce network calls—invalidate
cache when `.terraform.lock.hcl` changes.

## Human applies in emergencies

If CI unavailable, require screen recording + ticket + two approvals before local `apply`—feed logs
back into pipeline templates to close gaps later.

## Supply-chain inside CI

Pin third-party Actions to **commit SHAs** for high-risk workflows; dependabot can bump with review.

## Scaling review culture

When plan output noise grows, invest in **compact plans** (`-compact-warnings`) and module refactors—
humans ignore giant plans and miss real risk.

## Partial applies and targeting

Avoid `-target` in automated applies except documented break-glass—pipelines that routinely target
subgraphs create snowflake states diverging from declared full configuration.

## Blue/green infrastructure patterns

Some teams maintain parallel workspaces (`blue`/`green`) for large VPC replacements—CI selects active
workspace via pipeline input; Terraform still requires careful data migration between waves.

## Policy in CI vs in Terraform Cloud

Running **Sentinel/OPA** locally in CI duplicates Terraform Cloud policy but offers air-gapped options—
choose one source of truth to avoid conflicting verdicts.

## Windows of maintenance

When upgrading providers, open **maintenance mode** tickets for applications sensitive to replace
operations—CI timing alone cannot calm data-plane churn.

## Documentation for Atlantis webhook security

Atlantis webhooks must verify **HMAC secrets** per VCS vendor—rotate on engineer departures; log
illegal webhook attempts.

## Release trains

Align Terraform module releases with **application release trains** so consumers adopt semver tags on
predictable schedules—reduces „we forgot to bump module“ drift.

## Capacity for plan parallelism

Large orgs shard Terraform workloads by **business unit** CI pools to prevent noisy neighbor API
throttling—SRE tracks AWS `RequestLimitExceeded` spikes correlated with Monday morning plan storms.

## Closing operations mantra

Automate **plan**, gate **apply**, measure **drift**, rehearse **rollback**, narrate **changes**—five
verbs that keep Terraform CI mature.

## PR template checklist

Embed checkboxes: ``terraform fmt``, ``validate``, ``tflint``, ``checkov``, ``screenshot of plan
summary``, ``linked ticket``—cheap guard against human forgetfulness.

## Backport pipeline

For hotfixes on release branches, duplicate CI with stricter reviewers—cherry-pick infra commits need
the same gates as main to avoid emergency-induced regressions.

## Feature flags vs Terraform

Sometimes product feature flags gate app rollout while Terraform provisions dormant infra—document
which system owns risk during partially enabled phases.

## Terraform Cloud agents

Self-hosted agents in private networks bridge Terraform Cloud to internal endpoints—patch agents like
production workloads and monitor their IAM roles.

## env0 drift remediation integration

Managed platforms sometimes open **auto-remediation** PRs after drift scans—verify bot PRs with same
rigor as human PRs; attackers compromising bots becomes a supply-chain angle.

## Observability of CI time

Track **p50/p95 plan durations** per root; spikes highlight provider issues or new data sources—feed
to platform teams before engineers blame code.

## Secrets rotation alignment

When rotating CI OIDC thumbprints or audience strings, coordinate multi-VCS if using both GitHub and
GitLab with shared roles—document distinct trust policies.

## Final note

Great Terraform CI is **boring**: obvious stages, pinned versions, crisp approvals, loud drift alarms,
and practiced incident responses.

Add **explicit timeouts** to CI jobs running `terraform apply`—hung applies may hold state locks and
block teammates; failing fast triggers pager attention sooner than silent multi-hour hangs.

For **self-hosted GitHub runners**, pin Docker images digest-by-digest for Terraform CLI containers—
`latest` tags have caused surprise provider incompatibilities mid-week with no code changes.

Record **which Terraform workspaces** each pipeline touches in job names—operators scanning CI
dashboards spot accidental prod touches faster when naming is explicit.

Tag each pipeline run with the **Terraform root path** (`-chdir`) in telemetry so distributed monorepo
failures sort quickly in log aggregators.
