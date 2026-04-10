---
name: terraform-kubernetes
description: >-
  Use hashicorp/kubernetes and hashicorp/helm providers effectively. Covers kubeconfig vs exec auth,
  server-side apply considerations, CRDs (CustomResourceDefinition) management trade-offs, Helm
  releases (values, atomic upgrades, wait), the kubernetes_manifest resource for dynamic APIs, and
  when to prefer GitOps (Argo CD, Flux) over Terraform for in-cluster drift. Pair with cloud skills
  for cluster provisioning.
---

# Terraform with Kubernetes and Helm

## Scope

You already have a **cluster** (GKE/EKS/AKS) from cloud Terraform. This skill addresses **in-cluster**
objects: namespaces, RBAC, Helm charts, CRDs, and advanced **`kubernetes_manifest`**. It does not
replace **GitOps** - it helps you choose boundaries.

## Provider auth patterns

### kubeconfig path (simple dev)

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

### exec plugins (EKS example)

```hcl
provider "kubernetes" {
  host                   = var.cluster_endpoint
  cluster_ca_certificate = base64decode(var.cluster_ca)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks", "get-token",
      "--cluster-name", var.cluster_name,
      "--region", var.region
    ]
  }
}
```

Rotate tokens per CI job - do not store long-lived kubeconfigs in Terraform state.

## Basic objects

```hcl
resource "kubernetes_namespace" "imu" {
  metadata {
    name = "imu"
    labels = {
      environment = var.env
    }
  }
}

resource "kubernetes_service_account" "processor" {
  metadata {
    name      = "imu-processor"
    namespace = kubernetes_namespace.imu.metadata[0].name
  }
}

resource "kubernetes_role" "processor" {
  metadata {
    name      = "imu-processor"
    namespace = kubernetes_namespace.imu.metadata[0].name
  }

  rule {
    api_groups = [""]
    resources  = ["configmaps"]
    verbs      = ["get", "list", "watch"]
  }
}

resource "kubernetes_role_binding" "processor" {
  metadata {
    name      = "imu-processor"
    namespace = kubernetes_namespace.imu.metadata[0].name
  }

  subject {
    kind      = "ServiceAccount"
    name      = kubernetes_service_account.processor.metadata[0].name
    namespace = kubernetes_namespace.imu.metadata[0].name
  }

  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "Role"
    name      = kubernetes_role.processor.metadata[0].name
  }
}
```

## Helm provider

```hcl
provider "helm" {
  kubernetes {
    host                   = var.cluster_endpoint
    cluster_ca_certificate = base64decode(var.cluster_ca)
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args = [
        "eks", "get-token",
        "--cluster-name",
        var.cluster_name,
        "--region",
        var.region
      ]
    }
  }
}

resource "helm_release" "prometheus" {
  name       = "kube-prom"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  version    = "58.0.0"
  namespace  = "monitoring"
  create_namespace = true

  atomic    = true
  timeout   = 600
  wait      = true
  skip_crds = false

  values = [
    templatefile("${path.module}/values/prometheus.yaml.tftpl", {
      retention = "15d"
    })
  ]
}
```

**`atomic`** rolls back failed upgrades; **`skip_crds`** matters when CRDs are owned elsewhere.

## kubernetes_manifest (server-side apply friendly)

For APIs not modeled as first-class resources, use **`kubernetes_manifest`** with careful field
management:

```hcl
resource "kubernetes_manifest" "imm_imu" {
  manifest = {
    apiVersion = "autoscaling/v2"
    kind       = "HorizontalPodAutoscaler"
    metadata = {
      name      = "imu-processor"
      namespace = kubernetes_namespace.imu.metadata[0].name
    }
    spec = {
      scaleTargetRef = {
        apiVersion = "apps/v1"
        kind       = "Deployment"
        name       = "imu-processor"
      }
      minReplicas = 3
      maxReplicas = 50
      metrics = [{
        type     = "Resource"
        resource = {
          name = "cpu"
          target = {
            type               = "Utilization"
            averageUtilization = 70
          }
        }
      }]
    }
  }
}
```

Watch for **field manager conflicts** when mixing Terraform with controllers that also server-side
apply the same objects.

## CRDs: should Terraform own them?

**Pros of Terraform-managed CRDs:** single rollout with cluster baseline; versioned alongside cluster.

**Cons:** CRDs are cluster-scoped and impact many tenants; operators (Argo, Flux) often manage CRDs
with their own charts to avoid Terraform holding dangerous delete power.

Rule: **platform team** owns CRDs for shared controllers; **application teams** consume CRDs via
GitOps or Helm dependencies.

## Terraform vs Argo CD / Flux (GitOps)

| Concern | Terraform shines | GitOps shines |
|---------|------------------|---------------|
| Cluster + cloud IAM | Yes | Partial (needs glue) |
| App deploy velocity | Slower PR cycle | Fast reconcilers |
| Drift detection | Plan/apply | Automated sync/diff |
| Secrets | Wrap with external secrets operators | Native patterns (SOPS, ES) |

Hybrid is common: Terraform provisions **cluster + core addons**; GitOps manages **application
manifests** and **promotion** across namespaces.

## kubernetes_manifest provider caveats

- Large YAML blobs can explode plan output - keep manifests modular.
- Some fields are **server-defaulted**; expect perpetual drift if you omit them - use `computed_fields`
  or accept drift via `lifecycle` where appropriate (use sparingly).

## Namespaces as blast-radius boundaries

Always scope namespaced RBAC; avoid cluster-admin bindings in application stacks. Use **`kubernetes_limit_range`** and **`kubernetes_resource_quota`** in shared clusters.

## CSI / storage classes

```hcl
resource "kubernetes_storage_class" "ssd" {
  metadata {
    name = "ssd"
  }
  storage_provisioner = "ebs.csi.aws.com"
  volume_binding_mode = "WaitForFirstConsumer"
  parameters = {
    type = "gp3"
  }
}
```

Provisioner names differ per cloud - parameterize in modules.

## NetworkPolicies

Declare **`kubernetes_network_policy`** alongside app deployments when service meshes are absent - 
Terraform is excellent at codifying deny-by-default posture for regulated namespaces.

## Admission and policy

`ValidatingWebhookConfiguration` can be Terraform-managed, but **webhook TLS certificates** rotate - 
pair with **cert-manager** or cloud PKI. Many teams prefer **OPA Gatekeeper/Kyverno** installed once via
Helm, policies promoted through GitOps.

## Upgrades and CRD deletion danger

Never casually `terraform destroy` CRD owners - this can cascade-delete all CR instances. Use
`prevent_destroy` in production modules or isolate CRDs in dedicated stacks with strict approvals.

## Testing

Use `terraform test` with mocked kubernetes provider for fast feedback; integration tests belong in
Kind/k3d pipelines with ephemeral clusters - see `terraform-testing`.

## Helm release state

Terraform stores Helm release metadata in its state - not identical to Helm’s own secret storage - document this for operators who expect `helm` CLI to be authoritative.

## Blue/green and canaries

Advanced delivery (Argo Rollouts, Flagger) usually lives in GitOps controllers; Terraform provisions
cluster-level CRDs and baseline RBAC, not per-release traffic shaping.

## Service mesh notes

Installing Istio/Linkerd via Helm from Terraform is fine; **incremental config** (VirtualServices) may
flip to GitOps for developer velocity.

## When not to use Terraform

High-churn microservice YAML and frequent config tweaks often rot in Terraform PR latency - GitOps
wins. Keep Terraform for **durable infra** and **shared security baselines**.

## Incident tips

If `kubernetes` provider fails with **x509** errors, verify **`cluster_ca_certificate`** matches the
API server, especially after certificate rotation - refresh data from cloud APIs in the same apply.

## Summary

Treat Kubernetes Terraform as **platform glue**: cluster access, core charts, baseline CRDs/RBAC, and
integrations. Delegate **application lifecycle** to GitOps when teams need rapid iteration and strong
drift reconciliation - combine both deliberately rather than forcing a single tool to own everything.

## Workload Identity (cloud to cluster)

On GKE/EKS/AKS, bind **Kubernetes service accounts** to **cloud IAM** via workload identity - Terraform
declares the Kubernetes `ServiceAccount` annotations and the cloud-side IAM trusts. Double-check
**identity namespace** defaults (GKE) and **OIDC issuer URLs** (EKS) when rebuilding clusters - silent
breakage shows up at runtime as `AccessDenied`, not plan time.

## Ingress controllers

Installing **`ingress-nginx`**, **`aws-load-balancer-controller`**, or **Gateway API** implementations
via Helm from Terraform is standard. Split **DNS** (`external-dns`) ownership carefully between
platform and application pipelines to avoid TXT record fights.

## Operators and CR instance lifecycle

Operators (Prometheus, Strimzi, etc.) create CR instances. Terraform should usually **not** manage
those instances unless you accept Terraform as the **sole** mutator - prefer GitOps for instance-level
changes and let Terraform pin the operator **Helm release** only.

## Pod security

Adopt **Pod Security Standards** via namespace labels (`pod-security.kubernetes.io/enforce`) alongside
Terraform-managed namespaces. Document exceptions; security teams should grep plans for `privileged:
true` lifts.

## Secrets management

Avoid placing raw kube secrets in Terraform - prefer **External Secrets Operator** or cloud secret
integrations. When you must template a `kubernetes_secret`, mark outputs sensitive and rotate after
apply - see `terraform-secrets`.

## Provider alias patterns

Multiple clusters (prod vs DR) use **`provider "kubernetes" { alias = "dr" ... }`** and explicit
`provider = kubernetes.dr` on resources - identical to cloud provider alias stories.

## API version churn

Kubernetes APIs graduate (`extensions/v1beta1` removals). Pin GKE/EKS versions in Terraform cluster
resources and schedule API migration sprints - `kubectl convert` and provider errors guide refactors.

## Observability stack ownership

If Terraform installs Prometheus/Grafana/Loki, also Terraform **alerting** endpoints (`ServiceMonitor`
CRs) or accept GitOps ownership post-install. Mixed ownership causes Terraform to plan-delete
resources the operator created.

## Capacity and autoscaling

Declare **`cluster_autoscaler`** node pools / managed node groups via cloud providers; pair with
**HPA/VPA** objects. Terraform is best for **baselines**; autoscaling handles spikes.

## Disaster recovery

Back up **etcd** (managed by cloud) according to vendor guidance; for **in-cluster** stateful apps,
coordinate Velero/Snapshot CRDs. Terraform should declare backup CRDs only if DR is code-reviewed
infrastructure.

## Human factors

Platform teams should run **office hours** for Terraform+Helm because mis-clicked `helm uninstall`
bypasses Terraform - document reconciliation (`terraform apply`) procedures to restore desired state.

## Compatibility matrix

Maintain a table (README) mapping **Kubernetes version** ↔ **Helm chart versions** ↔ **provider
versions**. Upgrading one without the others is a common outage vector during minor Kubernetes bumps.

## CI tips for Helm + Terraform

Run `helm repo update` in CI before `terraform plan` when using remote chart repositories; pin
`version` on `helm_release` to avoid surprise upgrades. Cache chart downloads where possible but
validate checksums or version pins to prevent supply-chain drift between runners.

Document **`postrender`** or **`set`** usage in modules - teams often need to inject cluster-specific
URLs. Keeping those injections in `templatefile` makes reviews easier than opaque `--set` strings in
wrapper scripts.

For **multi-cluster Helm releases**, duplicate `helm_release` resources with distinct providers
rather than overloading `count` - clear resource addresses simplify targeted destroys during cluster
decommissioning.

Add **`lint`** steps for rendered manifests (`helm template` + `kubeconform`) in CI even when
Terraform plans succeed - schema errors sometimes surface only after API servers validate applied
objects.

Document **`kubernetes_manifest` wait** behavior if used - some fields require server reconciliation
before a second apply converges; rushing apply loops creates thrash.

