---
name: terraform-refactoring
description: >-
  Safe Terraform refactors: moved and removed blocks, import blocks, splitting or merging modules,
  renaming resources without destroy, replacing resources with minimal downtime (create_before_destroy,
  blue/green patterns), addressing drift vs code, and communicating risky changes to stakeholders.
  Use before large module surgery or when plans show unexpected destroys.
---

# Refactoring Terraform without blowing up production

## Scope

Turns **architecture changes** into **reviewable, incremental** plans. Pair with `terraform-core` for
syntax of `moved`/`import` and `terraform-state` for CLI surgery fallback.

## Prefer configuration-driven migrations

### moved blocks

Communicate renames to Terraform without `state mv`:

```hcl
moved {
  from = aws_lambda_function.imu
  to   = module.compute.aws_lambda_function.imu
}
```

Order matters in complex chains; combine with PR notes explaining blast radius.

### removed blocks

Stop managing something but leave it running:

```hcl
removed {
  from = aws_s3_bucket.legacy_tmp

  lifecycle {
    destroy = false
  }
}
```

Document who owns lifecycle going forward - otherwise you accumulate orphans nobody understands.

## import blocks

Bring existing resources into code with review:

```hcl
import {
  to = aws_kinesis_stream.imu_ingest
  id = "arn:aws:kinesis:us-east-1:111111111111:stream/prod-imu-ingest"
}

resource "aws_kinesis_stream" "imu_ingest" {
  name            = "prod-imu-ingest"
  stream_mode_details {
    stream_mode = "ON_DEMAND"
  }
}
```

Validate **import ID** format per resource docs - typos cause confusing partial imports.

## Splitting modules

1. Extract module directory with **identical inputs** to previous root locals.
2. Add `moved` blocks mapping old addresses to `module.foo.*`.
3. `terraform plan` should show **no-op**; if not, fix addresses before apply.

## Merging modules

Usually harder - watch **for_each** key changes. Sometimes migrate in two phases: create parallel
resources, cut traffic, destroy old (expensive). Prefer **feature flags** in applications to tolerate
duplicate infra briefly.

## Renaming without destroy

Terraform considers **resource name changes** as destroy/create. Use **`moved`** or `state mv` to keep
identity.

## Zero-downtime replacements

- Use `lifecycle { create_before_destroy = true }` where supported.
- For **Lamba**, publish new version/image, shift aliases - not always Terraform’s job alone.
- For databases, use **replicas** or **blue/green** vendor features - Terraform declares supporting
  infra, apps handle cutover.

## replace_triggered_by

Force replacement when tangential resources change (e.g., launch template version) - see `terraform-core`.

## Drift remediation vs refactor

1. `plan -refresh-only` to align state with actual.
2. Decide whether **code** or **cloud** is authoritative.
3. If cloud is wrong, fix cloud; if code is wrong, edit Terraform; **never** blindly `state rm`.

## Communication template

For risky PRs, include **Risk**, **Rollback**, **Customer impact**, **Monitoring** - even infra-only
changes deserve this to align SRE.

## Rollback strategy

Terraform lacks auto rollbacks - keep **previous git tag** and `plan.bin` artifact for supervised
`terraform apply` of older revision when safe.

## Provider upgrades during refactors

Avoid mixing **provider major bumps** with big refactors - separate PRs reduce unknowns when plans explode.

## Testing refactors

Run **`terraform test`** with mocks + integration plan in staging workspace with **copied state** (where
permitted).

## Anti-pattern: huge -target fixes

Targeting masks missing dependencies - acceptable temporarily, not as a lifestyle.

## Layered refactors

Refactor **one axis** per PR: names OR module split OR provider bump - not all three simultaneously.

## Observability during cutovers

Add temporary dashboards measuring error rates; if Terraform swap correlates with spikes, pause and
rollback per playbook.

## Data copies

Some refactors need **`aws_db_instance` snapshot** before replace - Terraform orchestrates **snapshot
identifier** attributes where supported; otherwise use runbook-managed snapshots.

## State backups

Before apply, `terraform state pull` to timestamped object - store encrypted.

## Handling partial applies

If apply fails mid-way, **do not** immediately re-apply - read error, check partial resources, consult
`terraform plan` for remainder; sometimes `refresh` needed.

## Team coordination

When multiple squads touch shared modules, freeze **interface changes** during critical business
windows (Black Friday, quarter close).

## Documentation debt

Update **architecture diagrams** in the same PR as refactors - reviewers catch mistaken `moved` targets
faster with visuals.

## When not to use this skill

Mystery errors → `terraform-troubleshooting`. Secrets handling → `terraform-secrets`.

## Example: rename for_each key

If you must change map keys, use temporary `moved` chains or accept one-time recreation - plan cost with
owners.

## Using terraform console for diffs

Compute expression outcomes (`merge` of tags) in console to predict plan noise before editing dozens
of files.

## FinOps sign-off

Refactors that replace SKUs (RI commitments) need finance ack - Terraform changes have dollar sequelae.

## Post-refactor hygiene

Delete **dead variables/outputs** after migration completes; dead API surfaces confuse module users.

## Continuous migration mindset

Track remaining **`FIXME` moved blocks** in backlog; temporary scaffolding ossify into permanent risk.

## Epilogue

Disciplined refactors feel slow - **fast** merges that skip planning cause weekend outages that cost more
than patience ever would.

## Refactoring checklist (printable)

1. Snapshot state.
2. Ensure clean working tree.
3. Split provider bumps.
4. Add `moved`/`import` with comments.
5. Plan in CI; compare to expected.
6. Apply in maintenance window if needed.
7. Monitor SLOs 24–72h.
8. Remove temporary compatibility code.

## Communication with auditors

When refactors touch compliance controls, attach **plan files** (sanitized) to change tickets - auditors
prefer deterministic evidence over oral summaries.

## Handling terraform graph complexity

Run `terraform graph | dot -Tsvg` for large systems sparingly - visualizations help explain why certain
resources must move together; archive SVGs in design docs for onboarding.

## Cross-stack ordering

Sometimes module split requires **two-phase delivery**: deploy new remote state outputs before
switching consumers - coordinate PR merges across repos with a shared checklist.

## Data migration ownership

If refactor needs **ETL** (S3 object rewrite, Dynamo scans), assign explicit DBA/data engineer owners;
Terraform provisions buckets, scripts run elsewhere - blur causes half-migrated datasets.

## Contract drift in protobufs/events

Streaming systems (Kinesis/Pub/Sub) may need **schema registry** updates separate from Terraform - 
note dependencies in PR body so consumers deploy coordinated releases.

## Tagging refactors

When renaming resources risks billingAllocation tags shifting, notify FinOps before apply - they may need
to remap chargeback rules.

## Observability of Terraform metadata

After refactor, verify **CloudWatch log groups / alarms** still reference correct ARNs - grep for old
log group names in dashboards JSON.

## When to accept destroys

Occasionally **destroy/create** is cheaper/safer than clever `moved` gymnastics (ephemeral caches).
Document customer impact windows honestly.

## Closing word

Refactoring is **risk management** dressed as code shuffling - treat it with the seriousness of a prod
deploy, because it is one.

## Terraform Cloud run ordering

If using Terraform Cloud, **queue** refactors behind feature work when using same workspace - serialized
runs avoid conflicting state versions; communicate queue expectations in Slack.

## Local vs remote exec differences

Refactors tested locally with **local backend** may behave differently on remote exec due to env
vars - mirror CI variables in `.envrc` or documented shells.

## Large team codeowners

Set **CODEOWNERS** paths for `moved`/`import` heavy directories so platform seniors always review - junior
edits to migration blocks are high risk.

## Linting during refactors

Run `tflint --fix` cautiously - auto-fix altering comments or ordering can obscure subtle diffs; prefer
explicit human formatting commits.

## Measuring success

After refactor, **`terraform plan`** should be calmer (fewer tangled replaces). Track **plan line
count** metrics over weeks - spikes warrant retros.

## Canary workspaces

For uncertain migrations, clone **workspace** with copied state subset (where safe) to rehearse; destroy
canary after - some orgs forbid state copying; respect policy.

## Feature toggles in modules

Temporary `var.enable_legacy_resource` booleans help gradual migration - delete toggles once stable to
avoid permanent complexity.

## Recording decisions (ADR)

Store Architecture Decision Records beside code referencing WHY `moved` blocks exist - future teams won’t
reverse refactors safely without rationale.

## Training shadowing

Pair senior+junior on first refactor PR after onboarding - reading moves is easier with narration.

## Postmortems without blame

If refactor causes incident, capture **signals** that plans underplayed risk - improve checklists, not
individuals.

## Automation guardrails

Pre-commit hook blocking `terraform destroy` without `ALLOW_DESTROY=1` env var prevents accidents
during local experimentation - trivial to bypass intentionally, but stops muscle-memory mistakes.

## Enterprise timeframe realism

Multi-week refactors need **executive summaries** monthly - translate technical milestones into business
risk reduction language.

## Closing checklist part deux

Validate **dashboards**, **on-call playbooks**, and **customer status pages** reference new resource
names - human docs lag Terraform easily.

## Backwards compatibility for remote_state consumers

When splitting modules, publish **temporary duplicate outputs** (`legacy_alb_dns`, `alb_dns`) until
downstream stacks cut over - communicate deprecation timelines in Slack #infra-announce.

## Address churn in Terragrunt

Terragrunt `dependency` outputs may reorder applies - verify `mock_outputs` during refactor windows - see
`terraform-terragrunt`.

## Resource address grep

Before merging, `grep -R 'aws_lambda_function.imu' -n` to catch lingering documentation snippets
pointing to old addresses - confuses operators during incidents.

## Performance of plans after refactor

If plan time doubles due to expanded graphs, consider **splitting stacks** rather than micro-optimizing
expressions - time is money in CI.

## Verifying destroy plans

When expecting destroys, copy plan JSON to spreadsheet highlighting **each** destroy with owner sign-off - 
no mass “LGTM” on thousand-line destroys.

## Communicating to app teams

Give sample **CloudWatch queries / traces** showing successful behavior post-refactor - evidence calms
nervous partners faster than Terraform alone.

## Legal holds

Some datasets cannot be destroyed - `prevent_destroy` alone insufficient; legal may require retaining
specific buckets - note holds in module README to block naive destroys.

## Final sanity

Sleep on gigantic refactors; fresh eyes catch missing `moved` blocks better than caffeinated 2 AM
eyes - schedule reviews with timezone diversity when feasible.

Celebrate successful refactors briefly - teams that only spotlight outages under-invest in incremental
hygiene; a quick kudos message reinforces discipline.

Archive **plan JSON line counts** before/after refactors - even if imperfect, trending downward signals
simpler systems worth sustaining.

Capture **Terraform CLI and provider versions** in those archives - forensics without version context
misattributes regressions to the wrong commit.
