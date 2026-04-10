---
name: terraform-secrets
description: >-
  Secret management with Terraform: Mozilla SOPS for encrypted gitops files, AWS Secrets Manager and
  SSM Parameter Store patterns, HashiCorp Vault provider flows, Terraform 1.10+ ephemeral resources,
  write-only attributes that reduce plaintext persistence in state, external data handoffs, and
  practical guidance for avoiding secrets in VCS and minimizing exposure in state snapshots. Use
  when designing secret injection or reviewing risky resource attributes.
---

# Secrets and sensitive data in Terraform workflows

## Scope

Focuses on **where secrets live**, how Terraform **references** them, and how to **limit** exposure in
state and logs. Complements `terraform-security` (IAM and governance).

## Plain facts about state

**Terraform state is not a secret store.** Anything needed to call cloud APIs (IDs, connection strings,
ARNs) lands in state. `sensitive = true` only affects human-readable output formatting.

## Pattern A: SOPS (git-encrypted secrets)

Store encrypted YAML/JSON in Git; decrypt in CI locally or via `age`/`pgp` keys. Terraform reads
values using `sops_decrypt_file` (external/datasource patterns) or pre-rendered `TF_VAR_*`.

Workflow:

1. Engineers edit secrets with `sops secrets.enc.yaml`.
2. CI decrypts to env vars before `terraform plan`.
3. Never commit plaintext.

## Pattern B: AWS Secrets Manager

Reference **secret ARN**; optionally use **`aws_secretsmanager_secret_version`** data source with
**`version_stage`**. For rotation, ensure consumers handle **staging labels**.

```hcl
data "aws_secretsmanager_secret_version" "vendor_api" {
  secret_id = var.vendor_secret_arn
}

resource "kubernetes_secret" "vendor" {
  metadata {
    name      = "vendor-api"
    namespace = var.namespace
  }

  data = {
    api_key = data.aws_secretsmanager_secret_version.vendor_api.secret_string
  }
}
```

Above duplicates secret material into **Kubernetes** - ensure etcd encryption + RBAC; better: use
**External Secrets Operator** and keep Terraform wiring minimal.

## Pattern C: SSM Parameter Store (StringSecureString)

Good for hierarchical keys `/app/prod/db/password`. Terraform can read **`aws_ssm_parameter` with
`with_decryption = true`** - again, value hits state.

Prefer **reading parameters inside applications** via IAM rather than piping through Terraform when
possible.

```hcl
data "aws_ssm_parameter" "db_url" {
  name            = "/app/prod/db/url"
  with_decryption = true
}
```

## Vault provider

Use when Vault is your **system of record**:

```hcl
provider "vault" {
  address = var.vault_addr
  auth {
    method = "jwt"
    path   = "auth/github/login" # illustrative
    # Configure per your auth backend
  }
}

data "vault_kv_secret_v2" "db" {
  mount = "secret"
  name  = "data/prod/db"
}
```

Ensure CI uses **short-lived** Vault tokens; avoid long-lived `VAULT_TOKEN` in plaintext GH secrets.

## Ephemeral resources (Terraform 1.10+ trend)

**Ephemeral** blocks/resources (consult current docs for your exact Terraform version) model secrets
that should not persist in state the same way as normal resources - useful for dynamic credentials.
Adopt cautiously: providers must support ephemeral patterns; verify behavior with `terraform show`.

## Write-only attributes

Some provider fields are marked **write-only**: Terraform tracks that a value was set but may not
store returned secret material - still validate with traces whether your provider version honors this as
expected.

## External secret operators (preferred for k8s)

Let Terraform install the operator; applications or ops teams sync secrets from cloud/Vault into
clusters - Terraform avoids holding plaintext beyond bootstrap.

## Avoiding secrets in VCS

- No `.tfvars` with real secrets in Git - use `*.tfvars` gitignored or SOPS.
- Ban `'password ='` in `git grep` pre-commit except examples.
- Use **`terraform fmt`** + **`gitleaks`** in CI.

## Lambda/environment variables trap

Putting secrets in **`environment { variables { ... } }`** exposes them to console viewers with Lambda
permissions - prefer **Secrets Manager** retrieval inside code with tiny IAM policies.

## RDS master passwords

Use **`manage_master_user_password`** (AWS) where available so AWS rotates secrets; otherwise inject
via Secrets Manager and **never** output password.

## CI logging

Redact `TF_LOG` in CI unless debugging - debug logs can leak sensitive values despite redaction efforts.

## Rotation runbooks

1. Update secret upstream.
2. If Terraform manages consumers, `terraform apply` to propagate references (not literal values).
3. Applications hot-reload or restart per platform capability.

## Multi-account secret sharing

Use **RAM/shared Secrets** patterns cautiously - often **copy-on-write** per account is clearer than
cross-account `kms:Decrypt` spaghetti - see `terraform-multi-account`.

## When not to use this skill

Backend KMS for state buckets → `terraform-state`. IAM for CI → `terraform-cicd`.

## HSM and compliance

Some orgs require **CloudHSM**-backed keys - Terraform declares KMS aliases referencing HSM-backed keys;
latency and cost implications belong in architecture reviews.

## SOPS + Terraform alternative

Run **`sops exec-env`** wrapper to export env vars only for the Terraform process lifetime - reduces
accidental shell history leakage vs long-lived exports.

## write-only + destroy ordering

Ensure destroy graphs don’t strand dependent secrets - use **`prevent_destroy`** on critical secret
versions only when business rules require (often prefer allow destroy with backups).

## Incident: leaked tfstate

Assume compromise if state bucket exfiltrated - rotate all credentials whose material appeared in state,
including derived Kubernetes secrets Terraform applied.

## Culture

Reward engineers for **removing** secrets from Terraform over clever hiding - simple architectures audit
easier.

## Vault dynamic secrets

Database creds minted per apply are powerful but complicate **drift**: decide who owns active leases
when Terraform reruns frequently - often better handled purely in runtime, not Terraform.

## GCP Secret Manager + IAM

Bind **`roles/secretmanager.secretAccessor`** narrowly to service accounts; avoid project-wide bindings
“for speed.”

## Azure Key Vault references

App Service **`key_vault_reference_identity_id`** patterns keep Terraform from ever seeing raw secret
strings - favor those integrations for PaaS.

## Final checklist

- Prefer **runtime fetch** over Terraform fetch when feasible.
- If Terraform must fetch, accept **state risk** and mitigate with **KMS + bucket policies + no
  public CIs**.
- Adopt **ephemeral/write-only** features as providers mature - retest after upgrades.

## SSM vs Secrets Manager cost

High-churn secrets: Secrets Manager charges per secret/month and API calls - SSM SecureString may fit
non-rotating configs; align FinOps before Terraforming thousands of parameters blindly.

## Encryption helpers

Wrap `random_password` resources with **`lifecycle { ignore_changes }`** cautiously - often better to
treat passwords as immutable and rotate deliberately via Secrets Manager rotation Lambda rather than
Terraform randomness.

## Teaching junior contributors

Run lunch sessions decoding **`terraform state show`** for a `kubernetes_secret` to viscerally teach
why operators frown on storing customer PII via Terraform.

## Documentation templates

Module README should list **which secrets the module expects** (`var.secret_arn` vs inline) - no
surprise data sources decrypting prod SSM paths without code review visibility.

## Closing

Terraform should **orchestrate** trust boundaries, not become a password vault. Push sensitive values
to systems designed for rotation, auditing, and fine access control - and shrink Terraform’s
responsibility whenever you can.

## PEM and TLS private keys

Avoid checking **TLS private keys** into modules; use **`tls_private_key`** only for dev sandboxes.
Production should integrate with **ACM**, **private CAs**, or **`cert-manager`** + ACME - Terraform
declares consumers, not long-lived key material.

## Cloudflare / third-party API tokens

Tokens in `TF_VAR_cloudflare_api_token` should be **scoped** to single zones and rotated via API - 
record rotation playbooks because tokens often lack org-wide IdP integration.

## GitHub PAT anti-pattern

Do not embed GitHub personal access tokens in Terraform to download modules - use **SSH deploy keys**
or **VCS-packaged modules** instead; PATs become organizational debt quickly.

## OCI registry credentials

For **`helm` / `docker` pulls**, prefer **IAM/instance roles** over static user passwords - Terraform
should wire roles, not write `.docker/config.json` secrets into stateful resources.

## Auditing secret reads

Enable **CloudTrail data events** (selective) on Secrets Manager / SSM calls when regulations
require - Terraform applies themselves generate API trails useful in investigations.

## Partial masking in plans

Terraform’s masking is not cryptographically secure - treat **plan artifacts** as confidential; store in
private buckets with encryption and short retention.

## Emergency break-glass secrets

Keep offline **hardware tokens** or **split knowledge** procedures for root keys - Terraform cannot
replace organizational rituals for highest sensitivity credentials.

## KMS grants cleanup

When using **`aws_kms_grant`**, ensure retiring resources revoke grants - stale grants can linger and
confuse audits; add integration tests in non-prod when feasible.

## Dynamic terraform variables from 1Password

Some teams integrate 1Password CLI (`op run -- terraform plan`) - document required CLI versions and
timeouts; failures mid-plan leave partial applies risk - practice abort procedures.

## Secret scanning exceptions

When Checkov flags **data sources decrypting parameters**, annotate with risk acceptance linking
architecture decision records - silence without context rots.

## Team rotation drills

Quarterly, pick a random **non-prod** secret and execute full rotation via runbook while Terraform
observes - uncover missing app reload hooks before prod fire drills.

## Terraform Cloud variable sets

Sensitive **variable sets** in Terraform Cloud still persist encrypted at vendor - understand retention
and export policies before storing regulated data; classify variables in governance spreadsheets.

## In-memory only credentials

For ultra-sensitive bootstrap, some teams run Terraform exclusively on **ephemeral CI jobs** with
**tmpfs** and no artifact uploads - still assume compromise if provider tokens hit process memory
unexpectedly logged.

## Cross-plane secret duplication

Avoid duplicating the same secret into **five** Terraform-managed stores - each duplication expands
blast radius; pick **one authoritative store** and reference narrowly.

## Kubernetes bootstrap chicken-and-egg

Bootstrapping clusters often needs an initial **admin kubeconfig** - store bootstrap tokens in
break-glass vaults, rotate after `kube-system` stabilizes, and remove temporary `kubernetes_secret`
resources from code.

## Differential secret handling in modules

Expose **`sensitive = true` outputs** only when unavoidable; consumers should prefer referencing
opaque ARNs rather than raw secret payloads crossing module boundaries.

## Future-looking interfaces

Follow HashiCorp RFCs on **ephemeral values** - they may change how modules declare “never persist this.”
Pin docs links in module README so upgrades happen intentionally.

## Summary sentence

Secrets are a **shared responsibility**: Terraform declares **edges** and **permissions**, while
specialized secret services own **material**, **rotation**, and **audit** - blur that line at your peril.
