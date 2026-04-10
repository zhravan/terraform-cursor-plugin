---
name: terraform-azure
description: >-
  Production patterns for hashicorp/azurerm (3.x/4.x - pin in root stacks). Use for resource groups,
  management groups and policies (via azurerm or azapi where needed), networking (vnet, subnets, private
  endpoints), Azure Kubernetes Service, Linux/Windows Functions, Storage Accounts with immutability
  and encryption, Key Vault access policies vs RBAC, Monitor diagnostics, and landing zone integration.
  Depth comparable to terraform-aws for Azure-first workloads.
---

# Terraform on Microsoft Azure (azurerm)

## Scope

Use this skill for **`azurerm`** resources in landing zones, application hosting, and data planes. For
**preview** or gap-fill APIs, teams sometimes pair **`azapi`**; keep that escape hatch documented.
Cross-cloud concerns live in sibling skills.

## Provider baseline

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.100.0"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    key_vault {
      purge_soft_delete_on_destroy               = false
      recover_soft_deleted_key_vaults            = true
      purge_soft_deleted_secrets_on_destroy      = false
      purge_soft_deleted_certificates_on_destroy = false
    }
  }

  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
}
```

**`features {}`** toggles destructive behaviors - review upgrades carefully; breaking changes appear in
provider changelogs.

## Resource group and tagging

```hcl
resource "azurerm_resource_group" "app" {
  name     = "${var.env}-imu-rg"
  location = var.location

  tags = {
    Project     = var.project
    Environment = var.env
    CostCenter  = var.cost_center
  }
}
```

Landing zones may forbid arbitrary RGs - use centrally issued names.

## Networking (hub-spoke friendly)

```hcl
resource "azurerm_virtual_network" "app" {
  name                = "${var.env}-vnet"
  location            = azurerm_resource_group.app.location
  resource_group_name = azurerm_resource_group.app.name
  address_space       = [var.vnet_cidr]
}

resource "azurerm_subnet" "aks" {
  name                 = "aks"
  resource_group_name  = azurerm_resource_group.app.name
  virtual_network_name = azurerm_virtual_network.app.name
  address_prefixes     = [cidrsubnet(var.vnet_cidr, 8, 1)]
}

resource "azurerm_subnet" "private_endpoints" {
  name                 = "private-endpoints"
  resource_group_name  = azurerm_resource_group.app.name
  virtual_network_name = azurerm_virtual_network.app.name
  address_prefixes     = [cidrsubnet(var.vnet_cidr, 8, 2)]
}
```

### Private endpoints (Storage)

```hcl
resource "azurerm_storage_account" "telemetry" {
  name                     = replace("${var.env}imutelemetry", "-", "")
  resource_group_name      = azurerm_resource_group.app.name
  location                 = azurerm_resource_group.app.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  public_network_access_enabled = false

  blob_properties {
    delete_retention_policy {
      days = 30
    }
  }
}

resource "azurerm_private_endpoint" "blob" {
  name                = "${var.env}-sa-blob-pe"
  location            = azurerm_resource_group.app.location
  resource_group_name = azurerm_resource_group.app.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "blob"
    private_connection_resource_id = azurerm_storage_account.telemetry.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
}
```

## Azure Kubernetes Service (AKS)

```hcl
resource "azurerm_kubernetes_cluster" "this" {
  name                = "${var.env}-aks"
  location            = azurerm_resource_group.app.location
  resource_group_name = azurerm_resource_group.app.name
  dns_prefix          = "${var.env}aks"

  default_node_pool {
    name       = "default"
    vm_size    = "Standard_DS3_v2"
    vnet_subnet_id = azurerm_subnet.aks.id
    zones      = ["1", "2", "3"]
    min_count  = 2
    max_count  = 6

    upgrade_settings {
      max_surge = "10%"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"
    network_policy = "azure"
  }
}
```

Integrate **Azure Monitor**, **workload identity**, and **private clusters** per security baseline - 
flags vary by API tier; consult current provider docs when upgrading majors.

## Azure Functions (consumption vs dedicated)

### Linux container on App Service Plan (illustrative)

```hcl
resource "azurerm_service_plan" "func" {
  name                = "${var.env}-func-plan"
  location            = azurerm_resource_group.app.location
  resource_group_name = azurerm_resource_group.app.name
  os_type             = "Linux"
  sku_name            = "EP1"
}

resource "azurerm_linux_function_app" "processor" {
  name                       = "${var.env}-imu-fn"
  location                   = azurerm_resource_group.app.location
  resource_group_name        = azurerm_resource_group.app.name
  service_plan_id            = azurerm_service_plan.func.id
  storage_account_name       = azurerm_storage_account.func.name
  storage_account_access_key = azurerm_storage_account.func.primary_access_key

  site_config {
    application_stack {
      python_version = "3.11"
    }
  }

  functions_extension_version = "~4"

  app_settings = {
    FUNCTIONS_WORKER_RUNTIME           = "python"
    WEBSITES_ENABLE_APP_SERVICE_STORAGE = "false"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

Wire **Key Vault references** via `azurerm_key_vault_secret` + RBAC instead of pasting secrets in
`app_settings` when possible.

## Key Vault (RBAC model)

```hcl
resource "azurerm_key_vault" "app" {
  name                       = "${var.env}-imu-kv"
  location                   = azurerm_resource_group.app.location
  resource_group_name        = azurerm_resource_group.app.name
  tenant_id                  = var.tenant_id
  sku_name                   = "standard"
  purge_protection_enabled   = true
  soft_delete_retention_days = 90

  rbac_authorization_enabled = true
}

resource "azurerm_role_assignment" "func_to_kv_secrets_user" {
  scope                = azurerm_key_vault.app.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_linux_function_app.processor.identity[0].principal_id
}
```

Prefer **private endpoints** for Key Vault in regulated networks.

## Diagnostics

```hcl
resource "azurerm_monitor_diagnostic_setting" "aks" {
  name                       = "${var.env}-aks-diag"
  target_resource_id        = azurerm_kubernetes_cluster.this.id
  log_analytics_workspace_id = var.law_id

  enabled_log {
    category = "kube-apiserver"
  }

  enabled_log {
    category = "kube-audit"
  }

  metric {
    category = "AllMetrics"
  }
}
```

## Multi-subscription patterns

Use **aliases** or **multiple provider blocks** (`alias = "hub"`) when deploying spoke peers referencing
hub virtual network IDs - mirror AWS multi-account mental model; see `terraform-multi-account`.

## State backend on Azure

Use `azurerm` remote backend (storage + container). Enable **versioning** and **CMK**; restrict
`storage_blob_data_contributor` to CI OIDC identities.

## Testing and policy

- `terraform validate` + `azurerm` provider upgrade previews in sandbox subs.
- **`checkov`** and **`tflint`** rulesets exist for Azure - see `terraform-testing`.

## Compared to AWS reference

Azure favors **managed identities** and **RBAC** over long IAM JSON documents, but the **shape** of
Terraform modules remains: networks, compute, functions, data, observability. Port lessons from the
Kinesis/Lambda story to **Event Hubs + Azure Functions** when building telemetry pipelines - patterns,
not resource names, transfer.

## Upgrade discipline

`azurerm` major upgrades rename attributes frequently. Maintain a **`upgrade guide.md`** in your
module repo; run `terraform plan` in **non-prod subscription** with identical code before prod.

## When not to use this skill

GCP resources → `terraform-gcp`. Raw Kubernetes YAML → `terraform-kubernetes`.

## Extra: budget + sentinel hooks

Wire **consumption budgets** (`azurerm_consumption_budget_resource_group`) to action groups for early
spend alarms. If using **Terraform Cloud policies** (Sentinel/OPA), keep module interfaces stable - 
policy denies on noisy tags are easier than retrofitting 200 services.

## Resilience choices

For mission-critical AKS, enable **zone redundancy**, **maintenance windows**, and **API server
authorized IP ranges** or private clusters. Pair Terraform changes with **blue/green node image**
rollouts documented in runbooks - some knobs remain operator-driven even when Terraform declares baseline
clusters.

## Naming limits and collisions

Azure names are often **globally unique** (`storage_account`, `key_vault`). Centralize `locals.name
= lower(replace(...))` patterns and validate lengths (`length <= 24` for storage) in variable
validation to fail fast.

## Governance via management groups

Higher-level governance (deny public IPs, enforce locations) might use **`azurerm_management_group Policy_assignment`** or **`azapi`** for newest APIs. Document ownership: platform IaC vs application IaC.

## Closing

Azure Terraform succeeds when **features blocks**, **identity models**, and **private networking** are
decided up front - retrofitting private endpoints or KV RBAC later is doable but expensive.

## Azure SQL (illustrative hardened pattern)

```hcl
resource "azurerm_mssql_server" "app" {
  name                         = "${var.env}-imu-sql"
  resource_group_name          = azurerm_resource_group.app.name
  location                     = azurerm_resource_group.app.location
  version                      = "12.0"
  administrator_login          = var.sql_admin_user
  administrator_login_password = random_password.sql_admin.result

  azuread_administrator {
    login_username = "breakglass-spn"
    object_id      = var.breakglass_object_id
  }

  minimum_tls_version = "1.2"
}

resource "azurerm_mssql_database" "app" {
  name      = "imudb"
  server_id = azurerm_mssql_server.app.id
  sku_name  = "S2"

  long_term_retention_policy {
    weekly_retention = "P4W"
  }
}

resource "azurerm_mssql_firewall_rule" "deny_public" {
  count            = var.allow_azure_services ? 1 : 0
  name             = "allow-azure-services"
  server_id        = azurerm_mssql_server.app.id
  start_ip_address = "0.0.0.0"
  end_ip_address   = "0.0.0.0"
}
```

Prefer **private endpoints** over public firewall rules; treat the above as a template to tighten.

## Application Gateway + WAF (edge sketch)

```hcl
resource "azurerm_public_ip" "agw" {
  name                = "${var.env}-agw-pip"
  location            = azurerm_resource_group.app.location
  resource_group_name = azurerm_resource_group.app.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_application_gateway" "this" {
  name                = "${var.env}-agw"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location

  sku {
    name     = "WAF_v2"
    tier     = "WAF_v2"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "gw-ip"
    subnet_id = azurerm_subnet.private_endpoints.id
  }

  frontend_port {
    name = "https"
    port = 443
  }

  frontend_ip_configuration {
    name                 = "public"
    public_ip_address_id = azurerm_public_ip.agw.id
  }

  # backend_http_settings, http_listeners, request_routing_rule blocks follow...
}
```

Complete AGW configs are verbose - keep them in dedicated modules with **examples**.

## User-assigned managed identities

Some teams prefer **user-assigned identities** reused across slots:

```hcl
resource "azurerm_user_assigned_identity" "app" {
  name                = "${var.env}-app-identity"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
}

resource "azurerm_role_assignment" "identity_storage" {
  scope                = azurerm_storage_account.telemetry.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_user_assigned_identity.app.principal_id
}
```

## Event Hub + Functions streaming analogy

Where AWS uses **Kinesis + Lambda**, Azure often pairs **Event Hubs** with **Functions** or **Stream
Analytics**. Terraform layout stays similar: stream namespace, authorization rules with least privilege,
consumer Function/Event Processor host, and **dead-letter** handling via Service Bus or storage
queues - mirror the partial failure tuning from the AWS reference skill when SDKs support batching.

## Observability conventions

Create **Log Analytics** workspace IDs once in platform stacks; pass as **`var.law_id`** to app stacks.
Standardize **diagnostic categories** in a module so every new resource inherits the same observability
contract without copy/paste.

## Runbooks vs Terraform

Some operations (AKS upgrades, node image rotation) remain semi-manual; document what Terraform
**does not** control to avoid false confidence during incidents.

## Training new contributors

Pair Azure Portal **read-only** roles with Terraform plans in sandboxes - engineers who only click Ops
will propose unsafe modules until they see how `features {}` interacts with deletes.

## Bastion and just-in-time access

For break-glass admin, prefer **Azure Bastion** + **PIM** rather than permanent public jump hosts.
Terraform can declare **`azurerm_bastion_host`** and supporting subnets; guard approvals carefully
because bastion subnets have design constraints (dedicated `/26` recommendation).

## Cost management tags

Enforce **`InheritTag`** policies from management groups so `azurerm_resource_group` tags cascade
consistently. Applications adding ad-hoc tag keys should register them with FinOps - random tag keys
break chargeback reports and waste cross-team time reconciling CSV exports.

## Reference: random password helper

```hcl
resource "random_password" "sql_admin" {
  length  = 24
  special = true
}
```

Store generated secrets in **Key Vault** via pipeline - not in Terraform outputs - see `terraform-secrets`.

Keep **`provider "random" {}`** registration (`required_providers`) alongside other providers when
using the random resource in shared modules.

Keep a **provider upgrade calendar** that links AzureRM release notes to your module milestones - Azure
API sunsets appear in provider changelogs before they surprise production applies.

