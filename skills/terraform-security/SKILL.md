---
name: terraform-security
description: >-
  Secure Terraform operations: least-privilege IAM for CI and humans, KMS encryption for state and
  data planes, avoiding secret material in VCS and state, network segmentation patterns, scanning for
  unintended public exposure, CIS benchmark alignment, and supply-chain hardening including lock file
  verification, private module/provider mirrors, and SBOM-adjacent provenance practices. Use when
  threat modeling infrastructure-as-code or responding to auditor questions.
---

# Security for Terraform-driven infrastructure

## Scope

Link **identity**, **secrets**, **network**, **state**, and **supply chain** controls. Pair with
`terraform-secrets` for operational handling of credentials and `terraform-testing` for automated
policy enforcement.

## Least-privilege IAM for automation

### Pattern: short-lived OIDC for CI

Prefer **OIDC trust** (GitHub Actions, GitLab CI) to long-lived `AWS_ACCESS_KEY_ID` secrets. Attach
policies scoped to **`terraform plan`** (read-only) vs **`apply`** (write) roles. Use **different**
roles per environment - never one mega-admin role.

### Human break-glass

Require **MFA** and **time-bound** assumption for admin roles used with Terraform. Log sessions to a
central SIEM; correlate `terraform` CLI user agents if available.

## KMS encryption everywhere

- **State buckets** (S3/GCS/Azure Blob) should use **customer-managed keys** where regulations demand.
- **RDS / SQL** with CMK and **rotation**.
- **Kinesis / Kafka / Pub/Sub** encryption at rest - Terraform should declare keys and grants together.

Document **KMS key policies** alongside Terraform - reviewers must understand which service principals
can decrypt.

## Secrets avoidance in state

Terraform stores **resource attributes** in state - even `sensitive` only hides CLI output. Mitigations:

- Reference **Secrets Manager / SSM / Key Vault / Secret Manager** by ARN/name, not raw strings.
- Use **`write_only`** / **ephemeral** patterns where supported (modern Terraform) - see
  `terraform-secrets`.

## Network segmentation

- Private subnets for compute; **no public SG rules** unless justified.
- **Private endpoints** for control planes (S3 interface endpoints in AWS, Private Service Connect in
  GCP, private endpoints in Azure).
- **Hub/spoke** or **shared VPC** patterns with Terraform documenting **which** CIDRs may talk to data
  stores.

## Public resource scanning

Run **Checkov/Trivy/terrascan** rules that ban `0.0.0.0/0` ingress on admin ports, public S3 ACLs,
and overly broad `Principal = *` in IAM/Lake policies. For gray areas (public CloudFront origins),
require security approval with architectural diagrams.

## CIS benchmarks

Map modules to **CIS AWS Foundations / Azure CIS / GKE CIS** controls:

- CloudTrail / Azure Activity Log / Cloud Audit Logs enabled (via Terraform where possible).
- Encryption defaults on storage + databases.
- MFA enforced at organization root.

Use **`conftest`** libraries maintaining CIS-oriented Rego packs - tune false positives carefully.

## Supply chain: lock files and mirrors

- Commit **`.terraform.lock.hcl`**; in CI run `terraform init -lockfile=readonly`.
- Host **provider mirrors** with internal artifact storage; verify hashes.
- Pin **module versions** (`source` refs); forbid `main` in prod roots via policy.

## SBOM and provenance (emerging)

Track provider ZIP URLs and versions in internal manifests; some enterprises attach **SLSA** style
metadata to internal modules (signature on commit tags). Terraform ecosystem lags container SBOM
maturity - document known gaps to auditors.

## Terraform Cloud tokens

`TF_TOKEN_*` must be **scoped** (org vs team) and **short-lived** when vendor supports - never log in CI
outputs. Rotate on engineer offboarding.

## State file access control

Restrict **`s3:GetObject`/`PutObject`** on `terraform.tfstate` to CI roles and named humans. Enable
**object lock / immutability** where legal allows for forensic recovery.

## blast radius reduction

Split monolithic stacks at account/project boundaries - SCPs / org policies limit what Terraform can ever
do even if policies regress.

## Drift vs attackers

Detect **unexpected security group rules** via drift jobs (see `terraform-cicd`). Attackers outside
Terraform may open paths - pair IaC with **periodic scans**.

## Reviewer checklist

- Are new data sources pulling **sensitive** fields into state unnecessarily?
- Does KMS policy grant **`kms:Decrypt`** too broadly?
- Are **`local-exec`** provisioners introducing unaudited command execution?
- Are **`publicly_accessible = true`** databases justified?

## Incident response

If state bucket exfiltration suspected:

1. Rotate cloud credentials tied to state access.
2. Assess whether secrets exist plaintext in captured state snapshots.
3. Re-key KMS if unauthorized decrypt possible.
4. Rebuild backend with new keys if integrity uncertain.

## When not to use this skill

Operational debugging of plan errors → `terraform-troubleshooting`. Multi-account routing →
`terraform-multi-account`.

## Defense in depth diagram (conceptual)

Identity (OIDC/MFA) → least-privilege roles → private networks → encrypted data + state → policy scans
in CI → drift detection in prod → centralized logging.

## Minimum viable security baseline

Even small teams should implement: remote state + locking + OIDC CI + Checkov on PRs + encrypted RDS +
no public admin SGs + secrets in managed stores.

## Custom rule examples (Conftest)

Deny S3 without versioning; deny IAM `*:*` actions without exception tag; deny unencrypted ELBs where
alternatives exist.

## Education

Run quarterly **secure Terraform** workshops covering `terraform state` risks and demonstration of
**Terraform stealing** scenarios in lab accounts - developers remember stories.

## Compliance mapping

Maintain spreadsheet mapping modules to SOC2/ISO controls; update on major refactors - auditors prefer
living documents over one-off screenshots.

## Supply-chain attacks on modules

Vendor modules as you would libraries: verify publisher, prefer signed Git tags, review releases, and
diff major upgrades - malicious modules are rare but devastating.

## Security champions in PR review

Require one **infra security** reviewer for changes touching IAM or network backbone resources - even
if code owners are generic platform teams.

## Logging terraform operations

Wrap `terraform apply` in CI with structured logs (who/when/which commit/workspace). Ship to SIEM;
alert on applies outside pipelines.

## Closing

Terraform security is **IAM + state + network + MR workflow**. Tools help, but culture - reviews,
least privilege, and environment isolation - does the heavy lifting.

## Cloud-specific encryption helpers

- AWS: **`aws_kms_key`**, **`aws_kms_alias`**, **`aws_kms_grant`** for cross-account decrypt scenarios
  with tight grants.
- Azure: **`azurerm_key_vault_key`** + CMK disk encryption for managed disks and storage.
- GCP: **`google_kms_crypto_key_iam_member`** for service agent usage - watch for default service
  agents requiring grants.

## Zero Trust undertones

Even “internal” Terraform runs should assume **zero trust** between CI agents and control planes - use
private connectivity for state and provider mirrors when exfiltration is a concern.

## Data residency

Encryption keys must live in **approved regions**; Terraform should not casually create cross-region
KMS usage without legal review - encode allowed regions in Conftest.

## Penetration test preparations

Before pen tests, snapshot Terraform and **tag** test resources distinctly; ensure pen testers
understand automation may revert changes during drift jobs - coordinate freeze windows.

## Post-quantum and algorithms

Some organizations mandate **future-facing algorithms** on certificates and KMS; Terraform rarely
sets PQ algorithms today - track provider features and policy exceptions.

## Key rotation runbooks

Terraform can re-key resources, but **data plane** implications (dual-encrypt periods) need runbooks.
Practice rotation in staging with application teams before production calendar events.

## Security group hygiene

Prefer **`security_group_rule`** resources or **separate rules** over inline `ingress` blocks in large
SGs - Terraform plans become clearer and imports cleaner.

## IAM policy JSON vs data sources

Prefer **`*_policy_document`** data sources for lint-like composition vs raw heredocs - reduces subtle
JSON errors that open unintended access.

## Public documentation leaks

Ensure module README examples use **obvious dummy** account IDs (`111111111111`) not real IDs - some
teams scrub accidentally committed ARNs from history using BFG; prevention beats cleanup.

## Final reminder

Security tooling flags **known bads**; architectural reviews catch **unknown bads** - keep both.

## Terraform metadata leakage

`terraform show -json` and provider schemas may reveal **resource addresses** that hint architecture.
Store artifacts in **private** buckets with short TTLs; redact customer-specific names when sharing
outside trust boundaries.

## Dynamic credentials TTL

For CI using **vault/agent** style dynamic AWS creds, ensure TTL exceeds longest `apply` but remains
short enough to limit blast radius - monitor for mid-apply expiry failures that leave partial stacks.

## heredoc command injection

Avoid interpolating untrusted PR titles or issue bodies into `local-exec` commands - treat CI inputs as
hostile. Prefer structured environment variables with allowlists.

## Provider mirrors integrity

When hosting mirrors, sign manifests or store BLAKE2 hashes in append-only logs - detect mirror
tampering before workers pull malware disguised as provider zips.

## Break-glass applies

Require **second approver** and **recorded screen** or **paired session** for production applies
outside normal pipelines - human applies bypass guardrails; monitor heavily.

## Encryption for CI caches

If caching `.terraform/` directories, encrypt caches at rest in artifact storage - caches can embed
downloaded modules with sensitive names.

## Red team exercises

Simulate **stolen laptop** scenarios: revoke credentials, verify state bucket policies block exfil,
confirm Terraform Cloud tokens deleted centrally.

## Regulatory addenda

HIPAA/PCI may mandate **FIM** (file integrity monitoring) on CI runners executing `terraform` - coordinate
with SecOps before claiming Terraform-only controls satisfy logging requirements.

## Secure-by-default module templates

Provide starter modules with **encryption on**, **public access blocked**, and **strict IAM**
patterns - teams copy/paste less insecurely when defaults are safe.

## Closing operations tip

After security incidents, run targeted **`terraform plan`** in all workspaces to detect attacker
resources outside modules - automation might still be innocent while manual changes lurk.

Pair Terraform controls with **cloud-native guardrails** (AWS Organizations SCPs, Azure Policy at MG
scope, GCP org constraints) - IaC cannot revoke capabilities that org policies never granted.

Document **version pinning** for security scanners in CI - silent auto-upgrades can open merge queues
to sudden policy failures unrelated to application changes; treat scanner upgrades like provider
upgrades with comms windows.

Maintain an **`exceptions.yml`** with owners and expiry dates for suppressed findings - permanent ignores
rot into unmeasured risk.

Quarterly, rotate **`terraform` CI roles** and **`state`** bucket policies through peer review - these
quiet controls silently rot when people change teams without paperwork.
