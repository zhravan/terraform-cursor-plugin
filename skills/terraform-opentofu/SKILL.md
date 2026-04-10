---
name: terraform-opentofu
description: >-
  OpenTofu vs HashiCorp Terraform: governance differences, migration steps, registry and tooling
  compatibility, encrypted state options, early evaluation features, provider iteration nuances, and
  decision guidance for selecting a toolchain in 2026. Use when evaluating forks, planning migrations,
  or explaining legal/licensing implications to stakeholders - not for vendor FUD.
---

# OpenTofu and Terraform: pragmatic comparison

## Scope

**OpenTofu** is a community-driven fork intent on keeping core IaC tooling open. **Terraform** remains
the HashiCorp trademark product with its own license evolution. This skill helps engineers make
**technical** decisions - legal review belongs to counsel.

## Divergence areas (high level)

Over time forks may adopt features at different cadences: language keywords, testing subsystems,
**encryption**, provider discovery, and compatibility with **private registries**. Always read release
notes for **your pinned versions** - blog summaries go stale.

## Migration steps (Terraform → OpenTofu)

1. Pin current Terraform version and capture **`terraform version`** output.
2. Run **`tofu version`** in CI alongside (read-only) to compare.
3. Replace CLI in pipelines (`tofu init/plan/apply`) after `tofu init -upgrade` in non-prod.
4. Validate **`.terraform.lock.hcl`** compatibility - regenerate if maintainers recommend.
5. Re-run full **policy** suite (Checkov/Conftest) since JSON schemas might differ slightly.
6. Update documentation and **provider mirrors** if internal.

Expect **human process changes** (binary names, Docker images).

## Encrypted state (conceptual)

Forks may offer improved **customer-managed encryption** stories for state files; combine with backend
encryption (S3 SSE-KMS, GCS CMEK) regardless. Understand **key rotation** implications: Terraform/OpenTofu
both need decrypt access during plan/apply.

## Early evaluation / experimental language

Track which features are **experimental** or **preview** in each toolchain - promote experiments to
staging only with feature flags documented in module README.

## Provider iteration

`required_providers` blocks remain central. **Registry namespace** compatibility is generally aligned
but confirm private registry tokens for each CLI in automation.

## When to pick Terraform (HashiCorp)

- **Enterprise support contracts** and bundled HCP services are mandated.
- **Organization standard** already standardized on Terraform images everywhere.
- **Third-party tools** (some scanners, vendors) lag OpenTofu testing - verify before committing.

## When to pick OpenTofu

- **Open-source governance** priorities outweigh vendor relationship considerations**.
- Desire to stay on a **community-driven** release model for core CLI while still using standard HCL.
- Internal platform willing to **own** compatibility testing across providers.

Many teams remain on Terraform simply because **muscle memory and images** exist - not irrational.

## Migration risk register

- CI caching paths referencing `terraform` binary names.
- Custom **pre-commit hooks** hard-coding terraform.
- **`local-exec`** scripts embedding terraform.
- **Training materials** and **on-call runbooks** outdated.

## Compatibility testing matrix

Build a matrix: {AWS, Azure, GCP} × {modules X, Y} × {Terraform vs OpenTofu} at pinned versions - record
plan diffs. Even zero diff builds confidence.

## Registry credentials

`~/.terraformrc` / `~/.tofurc` paths may differ - standardize with `TF_CLI_CONFIG_FILE`.

## Terraform Cloud / Enterprise

OpenTufu may not target TFC APIs identically - if you depend on **remote execution**, validate integration
or continue Terraform for orchestrated runs while using OpenTofu locally (generally discouraged due to
split-brain).

## Documentation clarity

When publishing modules internally, state **supported CLIs** explicitly in README to set consumer
expectations.

## Licensing literacy

Licenses evolved - schedule annual **legal refresh** with counsel; engineers should avoid definitive
legal statements in Slack.

## Testing parity

Ensure **`terraform test`** vs **`tofu test`** commands behave as expected in your pipelines - wrappers
may abstract this.

## Backwards compatibility promises

OpenTofu aims for compatibility but **bug-for-bug** parity is not guaranteed - retain ability to roll back
to Terraform during evaluation windows.

## Observability parity

Update **`TF_LOG`** playbooks to mention `TOFU_LOG` or equivalent environment variables if diverging.

## Supply chain

Use **official** release artifacts and verify checksums - same as Terraform.

## Community engagement

When adopting OpenTofu, contribute minimal reproducible bug reports upstream - improves everyone.

## When not to use this skill

Day-to-day AWS resources → `terraform-aws`.

## Myth busting

“Forks are automatically slower” - **measure** plan durations; anecdote misleads.

## Migration communications

Tell stakeholders what changes in **risk**, not politics: testing burden, rollback, training.

## Closing

Pick a toolchain deliberately; **pin versions**; **test**; **document**. You can change later with
discipline - panic switching during incidents rarely ends well.

## Feature tracking spreadsheet

Maintain internal sheet of **language features** (checks, imports, ephemerals) vs toolchain versions - 
onboarding reads one table instead of archaeology across blogs.

## Docker images

If using `hashicorp/terraform` Docker tags, evaluate **`ghcr.io/opentofu/opentofu`** tags similarly - 
pin by digest in CI.

## Homebrew vs apt

Developer laptops diverge quickly - standardize on **asdf** or **mise** plugins supporting both CLIs for
harmony.

## Windows support

Validate CLIs on Windows runners if applicable - path quoting differences lurk.

## Provider bug triage

When providers crash, reproduce with **both** CLIs before filing provider bugs - maintainers ask.

## Intellectual honesty

Benchmarks should include **cold** and **warm** plans, multiple roots, and realistic provider counts.

## Enterprise politics

Some procurement teams prefer single-vendor; others prefer OSS - provide **objective** compatibility
matrices to procurement, not editorials.

## Backporting features

Occasionally cherry-pick internal patches - maintain private **fork** only when absolutely necessary;
upstream fixes beat long forks.

## Summary sentence

OpenTofu vs Terraform is not a moral choice - it is **governance, support, and compatibility** under your
organization’s constraints; test accordingly.

## State encryption dual-controls

If using customer-managed KMS with dual control (HSM), ensure break-glass runbooks cover Terraform and
fork CLIs equally - operators shouldn’t discover incompatibility during drills.

## Air-gapped registries

Replicate provider zips to mirrors regardless of CLI - verify both CLIs honor `provider_installation`
blocks.

## CI naming

Rename jobs `iac-plan` not `terraform-plan` to reduce thrash when switching CLIs - small ergonomics
matter at scale.

## Training slide one-liner

“Same language, different runtime - validate plans like any major provider upgrade.”

## Final encouragement

Healthy competition between toolchains raises quality - contribute feedback constructively whichever side
you ship.

## Upgrade cadence alignment

If org runs **Terraform 1.8** while evaluating OpenTofu **1.8-equivalent**, misalignment on minor versions
causes noisy comparisons - sync patch families before performance benchmarking.

## Provider mirrors and checksum mismatches

Switching CLIs may recompute provider cache paths - run `terraform providers lock` / OpenTofu equivalent
on all worker OS targets to prevent “checksum mismatch” flakes during first rollout week.

## Module publishing contracts

Public modules should declare **`required_version`** constraints permissive enough for either toolchain if
you intend broad consumption - or split module branches explicitly to avoid unsatisfiable constraints.

## Editor integrations

VS Code Terraform extension behavior might differ subtly - document recommended settings.json for teams
using diagnostics heavily.

## Incident confusion

During incidents, on-call might mix CLIs accidentally - standardize shell prompts (`PS1`) to display
`terraform` vs `tofu` clearly.

## Backport security patches

Track security advisories for **both** CLIs when evaluation window straddles announcements - don’t assume
patches land same day.

## Migration rollback plan

Keep previous CI container image tags for **instant** rollback - store last-known-good digests in runbook.

## Vendor neutrality in RFPs

When responding to RFPs, phrase support as **“OpenTofu-compatible”** only after actually testing - 
marketing language without matrices invites audit pain.

## Observability dashboards

Tag applies with **`toolchain=tofu`** dimension in metrics to spot behavioral differences quickly.

## Community modules

Popular GitHub modules may mention only Terraform - open issues politely requesting documented OpenTofu
support when you validate compatibility.

## Academic honesty in PoCs

Two-week PoCs should include **apply/destroy cycle** on non-prod, not just plans - apply reveals provider
edge cases.

## Cross-functional sign-offs

Security, legal, FinOps each care about different facets - prepare **FAQ** addressing encryption, license,
support SLOs.

## End-of-life planning

Track announcements for old CLI versions - migrations happen twice if you postpone upgrades indefinitely.

## Extended comparison table (illustrative)

| Topic | Evaluate by |
|-------|------------|
| Language features | Release notes diff |
| Registry auth | Integration test |
| State encryption | Security review |
| CI time | Benchmark |
| Support | Procurement |

## FAQ snippet for internal wiki

**Q: Can we mix Terraform Cloud runs with OpenTofu local?** **A:** Only after vendor confirmation - 
default answer is no until proven.

## Final disclaimer

This skill avoids legal advice - consult counsel for license compliance questions in regulated
industries.

## Practical migration sprint (5 days)

Day 1 inventory binaries; Day 2 non-prod `init`; Day 3 policy harness; Day 4 perf benchmark; Day 5
go/no-go review with leads. Adjust for org size.

## Keeping empathy

Engineers may feel toolchain discussions distract from shipping - tie decisions to **measurable** CI
minutes saved or risk reduced, not ideology.

## Long-term ownership

Assign **toolchain owner** role rotating quarterly - prevents orphaned knowledge if one champion leaves.

## Closing reprise

Whether you run Terraform, OpenTofu, or both during transition, **disciplined testing** matters more
than the logo in the binary name.

## Binary verification

Teach contributors to **`shasum -a 256`** downloaded archives against published checksum files - 
supply-chain hygiene applies equally after a toolchain switch.

## Docs site mirroring

Mirror official docs offline if air-gapped - update snapshots when bumping minor versions; stale docs
cause misconfiguration more often than code bugs.

## Coexistence timeout

Set a calendar deadline ending “either/or evaluation” to avoid months of dual maintenance draining
focus - decide and commit resources.

## Executive summary template

One paragraph: current pain, proposed change, test evidence, roll-back, licensing note, decision date.
If you cannot fill each slot, you are not ready to switch.

Add a **`COMPATIBILITY.md`** file in your platform repo summarizing verification dates - stale claims
hurt trust more than admitting “last tested 90 days ago, rerun before adoption.”

When executives ask “are we safe?”, answer with **test matrices and rollback time**, not ideology - 
numbers align cross-functional teams faster than slogans.
