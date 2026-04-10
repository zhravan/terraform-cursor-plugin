---
name: terraform-troubleshooting
description: >-
  Operational debugging for Terraform: interpreting common errors, using TF_LOG=DEBUG effectively,
  narrowing noisy plan diffs, breaking dependency cycles, recovering from state locks, safe
  force-unlock procedures, resolving “objects in state but not in config,” provider/throttling issues,
  and decide when to stop using -target. Use during incidents and frustrating plans - not as a substitute
  for refactoring discipline.
---

# Terraform troubleshooting playbook

## Scope

Turns **cryptic failures** into **actionable next steps**. Pair with `terraform-refactoring` when
the fix is structural and `terraform-state` when the fix is state surgery.

## Enable debug logging (carefully)

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=/tmp/terraform.debug.log
terraform plan
```

**Warning:** DEBUG may log sensitive values - scrub logs before sharing externally. Revert to `INFO`
afterwards.

## Common error: “Error acquiring the state lock”

Someone (or CI) holds the lock.

1. Identify **LockID** message (DynamoDB lock ID or cloud console entry).
2. Confirm no active applies (Slack CI, Terraform Cloud runs list).
3. If confirmed stale, coordinate **`terraform force-unlock <lockid>`** per runbook - never solo in prod
   without evidence.

Document lock holder identity in ticket.

## force-unlock guardrails

- Two-person rule for production.
- Capture **`terraform plan` proving** no concurrent mutation expected.
- Take backend screenshot/log.

## Cycle errors: dependency graph issues

Symptoms: “Cycle:” in plan with resource loop.

Mitigations:

- Remove unnecessary `depends_on`.
- Stop referencing attributes that create implicit cycles - use `locals` to break?
- Split stacks to remove hard cycle - sometimes the architecture must change.

`terraform graph` helps visualize - keep DOT viewers handy.

## “Resource is tainted” / replace storms

**Taint** is largely legacy - prefer `terraform apply -replace=ADDRESS` on modern versions. Untaint only
when you understand why resource marked tainted.

## Provider errors and throttling

AWS `RequestLimitExceeded` - reduce parallelism:

```bash
terraform apply -parallelism=5
```

Also backoff in CI; consider **account-level API quotas** increases.

## “objects are in state but not in configuration”

Happened after removing code without `removed` block/`state rm`. Options:

- Re-import with `import` blocks.
- **`terraform state rm 'ADDR'`** if resource intentionally abandoned.
- **`removed` block** to formalize no-management story.

Never leave mystery orphans - document decisions.

## Wrong workspace

Symptoms: plan wants to delete everything or create duplicates.

Actions:

```bash
terraform workspace list
terraform workspace select prod
```

Ensure `TF_WORKSPACE` env not surprising you in CI.

## Backend misconfiguration

`terraform init` fails with 403/404:

- Validate bucket/key/region.
- IAM policy includes `s3:ListBucket` on bucket + prefix.
- KMS key policy allows encrypt/decrypt for role.

## “Provider produced inconsistent result”

Often provider bug or eventual consistency. Steps:

1. `terraform apply -refresh-only` then re-plan.
2. Upgrade/downgrade provider per issue tracker.
3. Escalate with **minimal reproduction**.

## Data sources failing in CI

Missing credentials or network path to metadata endpoints - add **OIDC env** or VPC endpoints for
`sts`.

## Large plan diffs from refresh

Use **`terraform plan -refresh-only`** first to separate drift from config changes - then edit code or
fix cloud.

## Secrets appearing in diff

If `TF_VAR_*` leaked incorrectly, rotate secrets - even if incident seems small.

## “Invalid for_each argument”

Collection unknown during plan - depends on resource attributes not computed yet.

Fix patterns:

- Pass known keys via variables.
- `try`/`length()` restructure - sometimes `-target` temporary apply (use caution) seeds data.
- Split modules so dependencies are explicit.

## Helm/Kubernetes provider auth flake

Tokens expired mid-plan - refresh kubeconfig or reduce plan duration; CI may need `aws eks update-kubeconfig`
each job.

## S3 state versioning rescue

If bad push occurred, **restore previous object version** with AWS CLI after revoking CI access - treat as
incident; update runbooks.

## When to use -target

Rare, tactical debugging or partial recovery - not standard operating procedure. If `-target` needed
often, architecture or state is unhealthy.

## Terraform Cloud run stuck

Cancel run via UI; verify agent not crashed; check **private agent** logs if used.

## “Cannot assume role”

ExternalId wrong, session duration too long, or trust policy mis-scoped - use AWS CLI `sts assume-role`
outside Terraform to reproduce.

## Module source 404

Typo in `source`, wrong `ref` tag, or missing credentials to private Git - run `git ls-remote` manually.

## Terraform version mismatch

`.terraform-version` / `required_version` conflicts - align local/CI.

## Windows-specific path issues

Backslashes, locked `.terraform` dirs - run terminal as admin? Prefer Linux CI.

## “No changes” but engineers expect changes**

Perhaps wrong directory, stale `TF_DATA_DIR`, or `-refresh=false` earlier masked drift - verify commands.

## Handling partial state corruption

Restore backup `state pull`, compare JSON, escalate to experts - avoid manual JSON edit unless you enjoy
outages.

## Paging policy

Define when Terraform incidents page SRE vs platform vs security - state locking might wait business
hours, credential leaks page immediately.

## Post-incident

Save **`TF_LOG`** excerpts sanitized, plan files, applied commit SHAs - future you needs timeline.

## Education snippets

Teach engineers **`terraform state list | sort`** muscle memory - sorting reveals duplicates quickly.

## Observability

Emit OpenTelemetry?? Not native - wrap Terraform in scripts logging duration per stage to your metrics
stack if valuable.

## When to stop for sleep

Repeated `apply` attempts worsening partial applies - pause, snapshot state, escalate rested engineers.

## Closing mindset

Terraform errors look like walls - usually they are **puzzles** with clues in logs, IAM, and graphs.
Systematic wins over random flag toggling.

## Bonus: checksum mismatch on init

Re-delete `.terraform` carefully (not state!) and `init` again - verify corporate proxy not rewriting
bytes.

## Bonus: long path issues on Windows

Relocate repo closer to `C:\src` - Terraform + providers + long module names exceed `MAX_PATH`.

## Bonus: Apple Silicon providers

Ensure lock file includes `darwin_arm64` - `terraform providers lock` with explicit platforms.

## Bonus: dockerized terraform vs local mismatch

Version skew between dev Docker and CI - pin same digest everywhere.

## Bonus: locale/UTF-8

Rare parsing issues when files saved wrong encoding - enforce UTF-8 in editorconfigs.

## Interpersonal: calm communications

Incident threads get tense - prefer factual bullet updates: **impact**, **hypothesis**, **next action**.

## Architecture signal

If troubleshooting repeats weekly, schedule **refactor sprint** - tactical fixes accumulate debt.

## Knowledge base

Tag resolved incidents Confluence page with **error string substring** for searchability.

## Automation guardrails

Pre-commit blocking `terraform apply` without `*_ALLOW=1` locally for dangerous roots - prevents habit of
“just applying” during debugging.

## Vendor support tickets

When opening HashiCorp/provider tickets, attach minimal reproduction **and** redacted DEBUG excerpt - 
speeds triage.

## Final checklist

1. Confirm directory/workspace/backend.
2. Confirm credentials + clock skew.
3. Refresh state perspective.
4. Inspect DEBUG logs locally (sanitized).
5. Graph cycles if needed.
6. Involve second reviewer before unlock/state rm.
7. Document outcome.

## Endnote

The best troubleshooting is **boring**: tight versioning, clean modules, peer reviewed state ops, and
dashboards that show drift before users do.

## “Error Message: context deadline exceeded”

Often provider HTTP client timeouts hitting slow endpoints - check corporate proxies, VPC endpoints, or
increase provider timeout settings if supported; verify not systemic network outage.

## CloudFront / global service eventual consistency

Creates sometimes lag - `terraform apply` fails then succeeds on retry - wrap CI with limited retry for
known benign cases, but don’t hide real errors - log retries.

## “Invalid index” on empty tuples

`count = 0` resources referenced without `[0]` guards - use `one()` / `try()` patterns in modern Terraform
or conditional expressions carefully.

## Templatefile missing variables

`templatefile` errors show line numbers - run `terraform console` with smaller template inputs to bisect.

## Remote exec provisioner failures

Provisioner stderr easy to miss - grep logs; prefer replacing provisioners with native resources when
possible.

## Data source eventual consistency after create

Some APIs return 404 briefly - `depends_on` alone may be insufficient; use short `time_sleep` resource
(sparingly) when documented necessary.

## Azure AD replication lag

Role assignments may fail immediately after SP creation - retry or insert wait resource - coordinate with
Azure known issues list.

## GCP API not enabled

Clear error with `googleapi: Error 403: ... API not enabled` - `google_project_service` ordering must
precede dependents; separate apply waves sometimes needed.

## EKS OIDC thumbprint issues

`tls_certificate` data source must reach `sts` endpoints - corporate proxies break - document offline
workaround if any.

## atlantis/terragrunt double wrapping errors

Read **innermost** Terraform stderr - wrappers may swallow detail - run plain `terraform` in same dir with
generated files to compare.

## “value is unknown” in console asserts during test

`terraform test` limitations - ensure values known at plan time or switch assertion timing to post-apply
runs.

## Disk full mid-apply

Large providers fill `/tmp` - monitor runner disk; set `TMPDIR` to larger volume in CI.

## Zombie lock files (local)

Rare stale `*.lock.info` on local backends - delete cautiously only when sure no concurrent process - prefer
remote backends to avoid this class entirely.

## IPv6-only environments

Some provider endpoints may misbehave - verify dual-stack or set correct `TF_HTTP_PROXY` settings.

## Clock skew breaks SigV4

NTP drift on VMs breaks AWS signing - ensure hypervisors synced.

## Corporate SSL inspection

Custom CAs must be trusted in Terraform Go TLS - install certs into container images running Terraform.

## Rate of change vs humans

If troubleshooting tickets exceed team capacity, automation for drift detection and self-service FAQs
reduces repeated questions - measure ticket volume weekly.

## Psychological safety

Blameless postmortems encourage reporting near-misses - those reports prevent future full incidents.

## Closing encouragement

You will still get surprised - that is normal - make surprises **rare** and **recoverable** through backups,
locks discipline, and calm runbooks.

## Appendix: quick command cheat sheet

```bash
terraform version
terraform providers
terraform workspace show
terraform state list
terraform state pull > state.json
terraform force-unlock ID   # caution
terraform console
```

Memorize less, bookmark more - consistent URLs in runbooks beat heroic memory.

## Multiline error parsing

Copy entire error blocks into tickets - partial screenshots hide causal lines Terraform prints earlier.

## HCL formatter masquerading as errors

Sometimes syntax errors point to previous line missing comma - run `terraform fmt` to reveal alignment
issues.

## Cloud vendor status pages

Check AWS/Azure/GCP status before deep debugging - hours wasted otherwise.

## Final sentence

**Stop, snapshot, share logs, involve a peer** - the four S’s save production more than frantic clicking.
