---
name: terraform-multi-account
description: >-
  AWS Organizations, account vending, and landing zones expressed as Terraform (or platform layers
  wrapping Terraform). Covers organizational units, SCP guardrails, centralized logging accounts,
  cross-account IAM roles (assume_role provider aliases), provider and backend patterns per account,
  remote state federation layout, workload vs security vs log-archive accounts, and environment
  isolation strategies. Use when expanding beyond a single account or subscription.
---

# Multi-account Terraform on AWS (and parallels)

## Scope

Focuses on **AWS Organizations** idioms - the concepts translate to Azure Management Groups + subs and
GCP folders/projects with adjusted resource names.

## Account taxonomy

Typical production estate:

- **Security / tooling** – centralized CI runners, artifact buckets, Terraform remote state.
- **Log archive** – immutable CloudTrail, Config snapshots.
- **Shared network** – transit gateway, inspection VPCs.
- **Workloads** – per env (`dev`, `stage`, `prod`) or per business unit.

## Organizational units (OUs)

Terraform (often a **bootstrap/root** stack) declares OUs and attaches accounts:

```hcl
resource "aws_organizations_organizational_unit" "workloads" {
  name      = "Workloads"
  parent_id = aws_organizations_organization.this.roots[0].id
}

resource "aws_organizations_account" "app_prod" {
  name                       = "app-prod"
  email                      = "aws-app-prod@example.com"
  parent_id                  = aws_organizations_organizational_unit.workloads.id
  iam_user_access_to_billing = "ALLOW"
  role_name                  = "OrganizationAccountAccessRole"
}
```

Human process: **email uniqueness** per account, finance ownership defined ahead of time.

## Service Control Policies (SCPs)

Attach guardrails denying risky APIs (`s3:PutBucketPublicAccess`, etc.). Terraform can manage
**`aws_organizations_policy`** attachments - coordinate with security to avoid locking out emergency
break-glass roles.

## Centralized logging account

Bucket policies allow **org trail** delivery; Terraform in the log account declares bucket + KMS;
org-level resources reference central bucket ARN.

## Cross-account assume_role (providers)

```hcl
provider "aws" {
  alias  = "workload_prod"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::222222222222:role/OrganizationDeploymentRole"
    session_name = "terraform"
    external_id  = var.external_id # if required
  }
}

module "app_prod" {
  source = "./modules/app"

  providers = {
    aws = aws.workload_prod
  }
}
```

Pair with **OIDC** in CI so humans rarely hold long-lived keys.

## Remote state per account

Each workload account stack should point **state** to buckets in the security/tooling account with
KMS keys trusting the workload role.

`terraform_remote_state` consumers assume **read roles** in producer accounts - encode IAM explicitly.

## Account vending automation

Platforms often wrap Terraform with **ServiceNow** or **Backstage** to trigger account creation
pipelines. Guard concurrency - Organizations API throttles; queue requests.

## Blast radius

Never give **`AdministratorAccess`** to CI roles - scope narrowly (`ec2:*` on specific ARN patterns,
`s3:*` limited to known prefixes). Use permission boundaries for humans.

## Environment isolation

**Hard isolation**: separate accounts per env. **Soft isolation**: single account with RBAC - hard is
preferred for regulated industries.

## IPAM and networking

Use **transit gateway** attachment modules; keep **RFC1918** diagrams in repo README to guide
reviewers when CIDR plans change.

## Provider default_tags per account

Different accounts may require **different mandatory tags** - parameterize tag maps as variables per
account stack rather than one global locals file hidden in a monorepo.

## When not to use this skill

Single-project GCP/Azure nuances → cloud skills. Troubleshooting assume role missing →
`terraform-troubleshooting`.

## Organization root protections

Enable **MFA delete** where available; restrict **root user** usage - Terraform should rarely touch root
only resources; if it must, peer review intensely.

## RAM shared resources

AWS RAM shares **`resource_share`** constructs (subnets, TGW) cross-account - ensure **invites accepted**
via automation or runbooks; Terraform may need `null_resource` local-exec for acceptance in some
legacy setups - prefer native resources when available.

## Parameter Store federation

Use **hierarchical SSM paths** with account-specific roots (`/accountid/app/...`) to avoid collisions
when aggregating metrics.

## Closing

Healthy multi-account Terraform treats **platform stacks** as products: documented, tested, and
versioned - application stacks assume their contracts rather than re-solving org-wide concerns weekly.

## Quarantine accounts

Spin up **sandbox** accounts per developer team with aggressive SCP deny lists - Terraform practice
there prevents surprise bills on prod credentials.

## Orphan account cleanup

Automate **`aws_organizations_account` deletion** guardrails - false triggers destroy financial audit
trails; use `prevent_destroy` and manual approval workflows.

## FinOps visibility

Enable **Cost and Usage Reports** to Athena/Glue via Terraform in master payer delegates - but confirm
who owns **CUR bucket** encryption keys centrally.

## ExternalId discipline

Rotate **`external_id`** on third-party vendor roles periodically; store in Secrets Manager, not in
plain `tfvars`.

## Inter-account KMS

KMS key policies granting cross-account `kms:Decrypt` for state buckets must include **least privilege**
via `Condition` on `kms:ViaService`.

## Detective controls

Enable **GuardDuty**/`Security Hub` centrally with delegated admin accounts - Terraform modules should idempotently
enable standards without thrashing enable/disable nightly.

## Naming standards

Enforce **`account_name`** conventions matching CMDB - automation looks unglamorous but prevents CMDB
rot.

## Transit secrets

Never reuse the **same GPG key** for all SOPS files across business units - partition keys per OU.

## Post-merger integrations

Merging two AWS orgs is rare and painful - Terraform can declare new OUs but **account migration** is
mostly manual API work; document human steps prominently.

## Break-glass routing

Keep **routing tables** in network accounts manageable - Terraform changes there are high risk; require
senior reviewer and pre-flight traceroutes in staging.

## Final mantra

**Many accounts, many states, few surprises** - that is the goal of disciplined multi-account IaC.

## Landing zone modules

Vendored **landing zone blueprints** (Control Tower, custom) should pin **module semver** and track
upstream advisories - platform drift is worse than app drift because footguns scale org-wide.

## AWS SSO / IAM Identity Center users

Terraform can manage **permission sets** and account assignments - great for reproducibility, but
coordinate with HR offboarding workflows; Terraform is not the HR system of record.

## Resource Sharing pitfalls

Some RAM shares require **principal Organization** vs explicit account lists - SCP denials on RAM APIs
surface as vague `AccessDenied` during applies; keep explicit deny lists documented.

## Cloud WAN / global networks

Advanced connectivity stacks belong in dedicated **network platform** repos - avoid every app team
redeclaring global resources; expose **attachment IDs** via SSM outputs.

## Backup centralization

`AWS Backup` vaults often live in log-archive accounts - Terraform must manage vault policies allowing
source accounts membership; test restores quarterly.

## Dedicated security-tooling accounts

Third-party scanners (Prisma, Wiz) integrate via **cross-account IAM** - Terraform wires roles; ensure
**externalId** secret rotation is automated.

## Data perimeter projects

Some orgs implement **data perimeter** SCPs isolating sensitive data services - Terraform applies SCPs in waves;
run **what-if** simulations in audit mode before enforcement mode toggles.

## Azure parallel: management groups

`azurerm_management_group` hierarchies mirror AWS OUs; policies attach at MG scope - see `terraform-azure`.

## GCP parallel: folders

`google_folder` + org policies enforce constraints; service projects attach to Shared VPC host via
**`google_compute_shared_vpc_service_project`**.

## Hybrid cloud routing

When VPN/Direct Connect/Interconnect land in network accounts, Terraform should declare **BGP ASN**
parameters once - duplicate ASN configs cause painful outages.

## Cost allocation tags org-wide

Implement **tag policies** at org level requiring `Environment`, `Owner` - Terraform modules should
merge local tags with mandatory org tags injected via vars.

## Change velocity concerns

Too many platform applies per day risks overlapping runs - use **mutex** queues for org-wide stacks.

## Cross-account CodeBuild runners

Some teams run Terraform in **CodeBuild** in tooling accounts assuming into targets - Codestar
connections replace long-lived GitHub PATs for repo access.

## Observability of assumptions

Maintain **`ASSUMPTIONS.md`** describing which accounts exist before running root module
`terraform apply` - new hires shouldn’t guess preconditions.

## Tabletop exercises

Simulate **`deny:organizations:LeaveOrganization`** mis-SCP - ensure you can still fix via break-glass.

## Perimeter egress

Egress VPCs with **network firewalls** should have Terraform modules separating **rule groups**
from attachment - PRs remain reviewable.

## Global IAM alias collisions

`account_alias` must be globally unique - validate via data source or pre-check script before apply.

## Closing operations

After expanding from one to many accounts, revisit **CloudWatch dashboards** - split per account to
avoid throttling giant metric queries.

## Additional example: read-only audit role

```hcl
data "aws_iam_policy_document" "audit_trust" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::111111111111:root"]
    }
  }
}

resource "aws_iam_role" "audit_readonly" {
  name                 = "SecurityAuditReadOnly"
  assume_role_policy   = data.aws_iam_policy_document.audit_trust.json
  max_session_duration = 3600
}

resource "aws_iam_role_policy_attachment" "audit" {
  role       = aws_iam_role.audit_readonly.name
  policy_arn = "arn:aws:iam::aws:policy/SecurityAudit"
}
```

Use distinct audit roles per tool (SIEM vs humans) to trace access patterns.

## Drift between Organizations and CMDB

Schedule reconciliation jobs comparing **Organizations API** vs internal CMDB; Terraform may succeed
while business records stale - finance audits care about the CMDB truth.

## Elevated access windows

Implement **JIT** access products - Terraform declares baseline roles, not standing admin - pair with
vendor tooling (e.g., AWS IAM Identity Center session duration policies).

## Sustainability

Document **carbon-aware** region choices if org commitments exist - platform teams may constrain allowed
`region` variables per account class.

## Final reminder

Multi-account Terraform is **socio-technical** as much as code: OUs without ownership become puzzles;
pair every automated account with a named business owner in your CMDB fields.

## Partner-facing accounts

Isolate vendor connectivity in **dedicated partner accounts** with narrow routing - commit allowlisted
CIDRs next to Terraform so network reviewers see scope instantly.

## Data residency classes

Stamp accounts with **`DataClass`** tags and reject disallowed `var.region` values via variable
validation - prevent accidental multi-region replication of regulated datasets.

## Org-wide enablement calendars

Roll out new services (Config conformance packs, Inspector) using **staged OUs** - Terraform apply once
per wave with monitoring; big bang applies across hundreds of accounts invite throttling and partial
failures that are tedious to reconcile.

Track **Organizations API error rates** in platform metrics - sudden spikes often correlate with
misconfigured SCP experiments; pause automation when error budget burns.

Publish a **network egress map** showing which accounts own NAT, inspection, and egress security
profiles - Terraform changes without that context invite asymmetric routing mistakes during incidents.
