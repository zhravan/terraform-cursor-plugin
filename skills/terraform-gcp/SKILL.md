---
name: terraform-gcp
description: >-
  Production use of hashicorp/google and hashicorp/google-beta. Use for IAM and service accounts,
  VPC networking, Cloud NAT and PSC, Google Kubernetes Engine, Cloud Run and Cloud Functions v2,
  Pub/Sub and BigQuery ingestion patterns, Cloud Storage buckets with CMEK and uniform access,
  org/folder/project hierarchy, and cross-project service identities. Depth comparable to other cloud
  skills; combine with terraform-multi-account for org-level patterns.
---

# Terraform on Google Cloud Platform

## Scope

Apply this skill when provisioning **projects**, **networks**, **GKE**, **serverless runtimes**,
**Pub/Sub**, and **GCS**. Use **`google-beta`** when resources require preview features. Keep
provider versions **synchronized** between `google` and `google-beta` to avoid odd coupling.

## Provider configuration

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.40"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.40"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
```

Authentication flows:

- Local: `gcloud auth application-default login`
- CI: **Workload Identity Federation** (preferred) or encrypted service account keys (discourage)

## IAM: service accounts and bindings

```hcl
resource "google_service_account" "imu_processor" {
  account_id   = "imu-processor"
  display_name = "IMU stream processor"
}

data "google_iam_policy" "telemetry_bucket" {
  binding {
    role = "roles/storage.objectViewer"
    members = [
      "serviceAccount:${google_service_account.imu_processor.email}",
    ]
  }
}

resource "google_storage_bucket_iam_policy" "telemetry" {
  bucket      = google_storage_bucket.telemetry.name
  policy_data = data.google_iam_policy.telemetry_bucket.policy_data
}
```

Prefer **resource-level IAM** modules over giant monolithic `google_project_iam_binding` blocks that
erase sibling bindings—use **`google_project_iam_member`** for additive grants when possible.

## Networking

```hcl
resource "google_compute_network" "this" {
  name                    = "${var.env}-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "private" {
  name                     = "${var.env}-private"
  ip_cidr_range            = var.private_cidr
  region                   = var.region
  network                  = google_compute_network.this.id
  private_ip_google_access = true
}

resource "google_compute_router" "nat" {
  name    = "${var.env}-router"
  region  = var.region
  network = google_compute_network.this.id
}

resource "google_compute_router_nat" "this" {
  name                               = "${var.env}-nat"
  router                             = google_compute_router.nat.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}
```

## GKE (private cluster sketch)

```hcl
resource "google_container_cluster" "this" {
  provider = google-beta

  name     = "${var.env}-gke"
  location = var.region

  network    = google_compute_network.this.name
  subnetwork = google_compute_subnetwork.private.name

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  ip_allocation_policy {}

  remove_default_node_pool = true
  initial_node_count       = 1
}

resource "google_container_node_pool" "default" {
  name       = "default-pool"
  location   = var.region
  cluster    = google_container_cluster.this.name
  node_count = 3

  node_config {
    preemptible  = false
    machine_type = "e2-standard-4"

    service_account = google_service_account.gke_nodes.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

Harden with **shielded nodes**, **Workload Identity**, **release channels**, and **authorized
networks** or fully private endpoints per security mandates.

## Cloud Run (HTTP service)

```hcl
resource "google_cloud_run_v2_service" "processor" {
  provider = google-beta
  name     = "${var.env}-imu-processor"
  location = var.region

  template {
    service_account = google_service_account.imu_processor.email

    scaling {
      min_instance_count = 1
      max_instance_count = 50
    }

    containers {
      image = var.container_image
      resources {
        limits = {
          cpu    = "2"
          memory = "1Gi"
        }
      }
    }

    vpc_access {
      connector = google_vpc_access_connector.serverless.id
      egress    = "PRIVATE_RANGES_ONLY"
    }
  }
}

resource "google_cloud_run_v2_service_iam_member" "invoke" {
  provider = google-beta
  location = google_cloud_run_v2_service.processor.location
  name     = google_cloud_run_v2_service.processor.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.events.email}"
}
```

## Cloud Functions (2nd gen)

```hcl
resource "google_cloudfunctions2_function" "handler" {
  provider = google-beta
  name     = "${var.env}-imu-fn"
  location = var.region

  build_config {
    runtime     = "python312"
    entry_point = "handler"
    source {
      storage_source {
        bucket = google_storage_bucket.functions_src.name
        object = google_storage_bucket_object.fn_zip.name
      }
    }
  }

  service_config {
    max_instance_count    = 50
    available_memory      = "512M"
    service_account_email = google_service_account.imu_processor.email
  }

  event_trigger {
    trigger_region = var.region
    event_type     = "google.cloud.pubsub.topic.v1.messagePublished"
    pubsub_topic   = google_pubsub_topic.imu_raw.id
    retry_policy   = "RETRY_POLICY_RETRY"
  }
}
```

## Pub/Sub and BigQuery (analytics path)

```hcl
resource "google_pubsub_topic" "imu_raw" {
  name = "${var.env}-imu-raw"
}

resource "google_pubsub_subscription" "imu_bq" {
  name  = "${var.env}-imu-to-bq"
  topic = google_pubsub_topic.imu_raw.name

  ack_deadline_seconds = 60
}

resource "google_bigquery_dataset" "imu" {
  dataset_id                 = "imu_${replace(var.env, "-", "_")}"
  location                   = var.bq_location
  default_table_expiration_ms = 3600000 * 24 * 400 # optional TTL story
}
```

Wire **BigQuery subscriptions** or **Dataflow templates** in higher environments—patterns align with
AWS Kinesis analytics, though implementations differ.

## Cloud Storage (hardened)

```hcl
resource "google_storage_bucket" "telemetry" {
  name                        = "${var.project_id}-${var.env}-imu-telemetry"
  uniform_bucket_level_access = true
  location                    = var.region

  encryption {
    default_kms_key_name = google_kms_crypto_key.telemetry.id
  }

  versioning {
    enabled = true
  }

  autoclass {
    enabled = true
  }
}
```

Add **org policy** constraints denying `allUsers` grants; scan for public buckets with **Security
Command Center**.

## KMS

```hcl
resource "google_kms_key_ring" "this" {
  name     = "${var.env}-kr"
  location = var.region
}

resource "google_kms_crypto_key" "telemetry" {
  name            = "telemetry"
  key_ring        = google_kms_key_ring.this.id
  rotation_period = "7776000s"

  version_template {
    algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
    protection_level = "SOFTWARE"
  }
}
```

## Org / folder / project facts

Use **`google_folder`**, **`google_project`**, **`google_organization_iam_member`** for landing zones.
Always separate **billing** attachment (`google_project` `billing_account`) from application stacks.

## Serverless VPC connector

```hcl
resource "google_vpc_access_connector" "serverless" {
  name          = "${var.env}-vpc-conn"
  region        = var.region
  network       = google_compute_network.this.name
  ip_cidr_range = "10.8.0.0/28"
}
```

## When not to use this skill

Azure/AWS-first topics live in sibling skills. Deep Kubernetes **in-cluster** resources may prefer
`terraform-kubernetes` after cluster creation.

## State and org constraints

Terraform state should live in a **central seed project** bucket with **uniform access**, **CMEK**,
and **object versioning**. Locking is a GCS concern (Terraform handles lock objects)—review
`terraform-state`.

## Testing hooks

Use **`terraform test`** against modules with `mock_provider` for Pub/Sub naming; for integration,
Terratest with a disposable project—see `terraform-testing`.

## FinOps defaults

Enable **labels** (`labels = { env = var.env }`) on GKE, buckets, and functions—export a `local.mandatory_labels`
map merged everywhere.

## Cross-project IAM

Service agents (`service-${PROJECT_NUMBER}@...`) need **`roles/artifactregistry.reader`** when Cloud Run pulls images cross-project. Document service agent emails in module outputs to avoid guesswork.

## Operational escalations

When **`google-beta`** resources disappear from plans unexpectedly, verify you didn’t drop the provider
on the resource—some GA promotions move resources to `google` only in newer provider releases; read
changelogs before major bumps.

## Bonus: Cloud Monitoring dashboards-as-code

Optional `google_monitoring_dashboard` JSON payloads can encode dashboards—keep JSON in `templatefile`
for readability. Pair with alerting policies for Pub/Sub dead-letter topics.

## Developer experience

Standardize **`TF_VAR_project_id`** in dotenv files for local apply; never commit SA JSON—use **gcloud**
ADC or short-lived WIF tokens in CI.

## Summary

GCP Terraform quality hinges on **IAM membership style**, **private networking +NAT/Serverless
connectors**, and **dual provider (`google`/`google-beta`) hygiene**. Treat service-agent emails and
org policies as first-class inputs in modules—surprises arise when defaults assume a blank-slate
project but land in a heavily constrained folder.

## Shared VPC attachments

Spoke projects attach to Shared VPC host subnets via **`google_compute_shared_vpc_service_project`**
and **`google_compute_subnetwork_iam_member`**. Document which **team owns the host project**—
Terraform mistakes here show up as routes existing but instances lacking `PRIVATE_GOOGLE_ACCESS` to
reach APIs.

## Private Service Connect

Expose managed services privately with **PSC endpoints** and **Cloud DNS** private zones. Because APIs
evolve quickly, confirm whether your resource still requires **`google-beta`**; add comments in code
when pinning provider features to preview tracks.

## Data residency

Set **`location`**/`region` thoughtfully for Pub/Sub, BigQuery, Storage, and KMS. Multi-regional
choices (`US`, `EU`) differ from single regions—cost and compliance trade-offs belong in README.

## Hierarchical firewall policies

Org policies + **`google_compute_firewall_policy`** (VPC firewall policies tied to folders) centralize
guardrails. Coordinate with network teams so application stacks don’t fight hierarchical policies with
overly broad `google_compute_firewall` rules.

## Artifact Registry

Store container images in **`google_artifact_registry_repository`** with **cleanup policies** and
**vulnerability scanning** enabled. Cloud Run and GKE pulls should use **service account** auth, not
long-lived JSON keys.

## Secret Manager

```hcl
resource "google_secret_manager_secret" "api_key" {
  secret_id = "${var.env}-vendor-api-key"

  replication {
    automatic = true
  }
}
```

Consume via **environment variables sourced from Secret Manager** in Cloud Run/Functions where
supported—see `terraform-secrets` for plaintext-in-state caveats.

## Cloud KMS IAM for service agents

Many services need **`roles/cloudkms.cryptoKeyEncrypterDecrypter`** on CMEK keys. When plans show
cryptic encrypt/decrypt failures at runtime but Terraform succeeded, verify **IAM eventual
consistency** and **service agent** grants.

## Quotas

Large GKE clusters or high-throughput Pub/Sub workloads may require **quota increases**. Track
ticket IDs in your platform backlog; Terraform cannot fix `InsufficientQuota` errors without human API
requests.

## Backups and DR

For GKE stateful workloads, combine **persistent disks**, **Backup for GKE**, or vendor backups—
declare only what Terraform should own (e.g., **backup plans**) and document operational restores.

## Naming collisions

Project IDs are **globally unique**; buckets too. Use **`random_id`** suffixes sparingly—FinOps teams
prefer deterministic patterns. Validate lengths (`length <= 30` for many identifiers).

## Collaboration tips

Use **`terraform plan -generate-config-out`** (Terraform 1.8+) cautiously when importing
hand-built projects—treat generated files as drafts. Pair imports with `terraform import` blocks per
`terraform-core`.

## Lessons from other clouds

From AWS IoT/Kinesis patterns: on GCP, **MQTT to Pub/Sub** often uses **IoT Core equivalent services**
or partner gateways—Terraform wiring focuses on **topics/subscriptions/functions** more than device
firmware. Reuse the **batch/partial failure** discipline even when SDKs differ.

## Org policy diagnostics

When `google_project` creation fails with policy violations, capture the **constraint name** and
route to the organization admin; keep Terraform modules **idempotent** so retries succeed after policy
exceptions or folder moves.

Document **`google_project_service`** activations required by downstream resources—mysterious API
errors often mean the **Service Usage API** has not been enabled yet for that project.

Prefer **`google_project_service`** with **`disable_on_destroy = false`** in shared platform stacks—
accidental service disables during destroy devastate dependent environments.

Snapshot **`gcloud services list --enabled`** before major refactors—Terraform may not yet know about
hand-enabled APIs others set in the console.


