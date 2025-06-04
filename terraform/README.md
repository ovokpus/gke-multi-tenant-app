# GKE Multi-Tenant Cluster Management Playbook - Terraform Implementation

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Terraform Infrastructure Setup](#terraform-infrastructure-setup)
4. [Namespace Management](#namespace-management)
5. [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
6. [Resource Quotas and Limits](#resource-quotas-and-limits)
7. [Monitoring and Cost Optimization](#monitoring-and-cost-optimization)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

## Overview

Multi-tenant GKE clusters enable multiple teams or applications to share cluster resources while maintaining isolation and fair resource distribution. This playbook provides a complete Terraform-based approach to provisioning and managing multi-tenant GKE clusters with proper namespace segregation, RBAC, and resource management.

### Key Benefits

- **Cost Optimization**: Shared infrastructure reduces per-team overhead
- **Resource Efficiency**: Better utilization through shared node pools
- **Centralized Management**: Unified cluster administration
- **Isolation**: Logical separation through namespaces and RBAC

## Prerequisites

### Required Tools

```bash
# Install required tools
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# Install Google Cloud SDK
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### GCP Setup

```bash
# Set project and enable APIs
export PROJECT_ID="your-project-id"
export REGION="us-central1"
export ZONE="us-central1-a"

gcloud config set project $PROJECT_ID
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable bigquery.googleapis.com
gcloud services enable monitoring.googleapis.com
```

## Terraform Infrastructure Setup

### Directory Structure

```
gke-multi-tenant/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── terraform.tfvars
├── modules/
│   ├── gke-cluster/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── namespaces/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── rbac/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── manifests/
    ├── resource-quotas/
    ├── network-policies/
    └── monitoring/
```

### Core Terraform Configuration

#### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = "~> 1.14"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}

data "google_client_config" "default" {}

provider "kubernetes" {
  host                   = "https://${module.gke_cluster.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(module.gke_cluster.ca_certificate)
}

provider "kubectl" {
  host                   = "https://${module.gke_cluster.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(module.gke_cluster.ca_certificate)
  load_config_file       = false
}
```

#### `variables.tf`

```hcl
variable "project_id" {
  description = "The GCP project ID"
  type        = string
}

variable "region" {
  description = "The GCP region"
  type        = string
  default     = "us-central1"
}

variable "zone" {
  description = "The GCP zone"
  type        = string
  default     = "us-central1-a"
}

variable "cluster_name" {
  description = "The name of the GKE cluster"
  type        = string
  default     = "multi-tenant-cluster"
}

variable "network_name" {
  description = "The name of the VPC network"
  type        = string
  default     = "gke-multi-tenant-network"
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
  default     = "gke-multi-tenant-subnet"
}

variable "teams" {
  description = "List of teams for namespace creation"
  type = list(object({
    name        = string
    description = string
    cpu_limit   = string
    memory_limit = string
    cpu_request = string
    memory_request = string
    pod_limit   = number
  }))
  default = [
    {
      name           = "team-a"
      description    = "Development Team A"
      cpu_limit      = "4"
      memory_limit   = "8Gi"
      cpu_request    = "2"
      memory_request = "4Gi"
      pod_limit      = 10
    },
    {
      name           = "team-b"
      description    = "Development Team B"
      cpu_limit      = "4"
      memory_limit   = "8Gi"
      cpu_request    = "2"
      memory_request = "4Gi"
      pod_limit      = 10
    }
  ]
}

variable "node_pool_config" {
  description = "Node pool configuration"
  type = object({
    machine_type   = string
    min_count      = number
    max_count      = number
    disk_size_gb   = number
    disk_type      = string
    preemptible    = bool
  })
  default = {
    machine_type = "e2-standard-4"
    min_count    = 1
    max_count    = 10
    disk_size_gb = 100
    disk_type    = "pd-standard"
    preemptible  = false
  }
}
```

#### `main.tf`

```hcl
# Network Configuration
resource "google_compute_network" "vpc" {
  name                    = var.network_name
  auto_create_subnetworks = false
  routing_mode           = "GLOBAL"
}

resource "google_compute_subnetwork" "subnet" {
  name          = var.subnet_name
  ip_cidr_range = "10.0.0.0/16"
  network       = google_compute_network.vpc.id
  region        = var.region

  secondary_ip_range {
    range_name    = "k8s-pod-range"
    ip_cidr_range = "10.1.0.0/16"
  }

  secondary_ip_range {
    range_name    = "k8s-service-range"
    ip_cidr_range = "10.2.0.0/16"
  }
}

# GKE Cluster Module
module "gke_cluster" {
  source = "./modules/gke-cluster"

  project_id                = var.project_id
  cluster_name             = var.cluster_name
  region                   = var.region
  zone                     = var.zone
  network                  = google_compute_network.vpc.name
  subnetwork              = google_compute_subnetwork.subnet.name
  secondary_range_pods    = google_compute_subnetwork.subnet.secondary_ip_range[0].range_name
  secondary_range_services = google_compute_subnetwork.subnet.secondary_ip_range[1].range_name
  node_pool_config        = var.node_pool_config
}

# BigQuery Dataset for Usage Metering
resource "google_bigquery_dataset" "usage_metering" {
  dataset_id    = "gke_usage_metering"
  friendly_name = "GKE Usage Metering Dataset"
  description   = "Dataset for GKE cluster usage metrics"
  location      = var.region

  access {
    role          = "OWNER"
    user_by_email = data.google_client_config.default.account_id
  }

  access {
    role           = "READER"
    special_group  = "projectReaders"
  }

  access {
    role           = "WRITER"
    special_group  = "projectWriters"
  }
}

# Service Accounts for Teams
resource "google_service_account" "team_service_accounts" {
  for_each = { for team in var.teams : team.name => team }

  account_id   = "${each.value.name}-dev"
  display_name = "${each.value.description} Service Account"
  description  = "Service account for ${each.value.description}"
}

resource "google_project_iam_member" "team_cluster_viewer" {
  for_each = { for team in var.teams : team.name => team }

  project = var.project_id
  role    = "roles/container.clusterViewer"
  member  = "serviceAccount:${google_service_account.team_service_accounts[each.key].email}"
}

# Namespaces Module
module "namespaces" {
  source = "./modules/namespaces"

  teams = var.teams

  depends_on = [module.gke_cluster]
}

# RBAC Module
module "rbac" {
  source = "./modules/rbac"

  teams               = var.teams
  service_accounts    = google_service_account.team_service_accounts

  depends_on = [module.namespaces]
}

# Enable GKE Usage Metering
resource "null_resource" "enable_usage_metering" {
  provisioner "local-exec" {
    command = <<EOF
gcloud container clusters update ${var.cluster_name} \
  --zone ${var.zone} \
  --resource-usage-bigquery-dataset ${google_bigquery_dataset.usage_metering.dataset_id} \
  --project ${var.project_id}
EOF
  }

  depends_on = [module.gke_cluster, google_bigquery_dataset.usage_metering]
}
```

### GKE Cluster Module

#### `modules/gke-cluster/main.tf`

```hcl
resource "google_container_cluster" "primary" {
  name     = var.cluster_name
  location = var.zone

  # We can't create a cluster with no node pool defined, but we want to only use
  # separately managed node pools. So we create the smallest possible default
  # node pool and immediately delete it.
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = var.network
  subnetwork = var.subnetwork

  # Enable IP aliasing
  ip_allocation_policy {
    cluster_secondary_range_name  = var.secondary_range_pods
    services_secondary_range_name = var.secondary_range_services
  }

  # Enable network policy
  network_policy {
    enabled = true
  }

  # Enable workload identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Enable resource usage export
  resource_usage_export_config {
    enable_network_egress_metering       = true
    enable_resource_consumption_metering = true
    bigquery_destination {
      dataset_id = "gke_usage_metering"
    }
  }

  # Security and compliance
  master_auth {
    client_certificate_config {
      issue_client_certificate = false
    }
  }

  # Enable shielded nodes
  node_config {
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }

  # Enable binary authorization
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  # Maintenance policy
  maintenance_policy {
    recurring_window {
      start_time = "2023-01-01T02:00:00Z"
      end_time   = "2023-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }

  # Addons
  addons_config {
    horizontal_pod_autoscaling {
      disabled = false
    }
    network_policy_config {
      disabled = false
    }
    gce_persistent_disk_csi_driver_config {
      enabled = true
    }
  }

  # Logging and monitoring
  logging_service    = "logging.googleapis.com/kubernetes"
  monitoring_service = "monitoring.googleapis.com/kubernetes"
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "${var.cluster_name}-node-pool"
  location   = var.zone
  cluster    = google_container_cluster.primary.name
  node_count = var.node_pool_config.min_count

  autoscaling {
    min_node_count = var.node_pool_config.min_count
    max_node_count = var.node_pool_config.max_count
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  node_config {
    preemptible  = var.node_pool_config.preemptible
    machine_type = var.node_pool_config.machine_type

    disk_size_gb = var.node_pool_config.disk_size_gb
    disk_type    = var.node_pool_config.disk_type

    # Google recommends custom service accounts that have cloud-platform scope and permissions granted via IAM Roles.
    service_account = google_service_account.gke_node_sa.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    labels = {
      env = "production"
    }

    # Enable workload identity
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    # Shielded instance config
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    tags = ["gke-node", "${var.cluster_name}-node"]
  }
}

resource "google_service_account" "gke_node_sa" {
  account_id   = "${var.cluster_name}-node-sa"
  display_name = "GKE Node Service Account"
}

resource "google_project_iam_member" "gke_node_sa_roles" {
  for_each = toset([
    "roles/logging.logWriter",
    "roles/monitoring.metricWriter",
    "roles/monitoring.viewer",
    "roles/stackdriver.resourceMetadata.writer"
  ])

  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.gke_node_sa.email}"
}
```

#### `modules/gke-cluster/variables.tf`

```hcl
variable "project_id" {
  description = "The GCP project ID"
  type        = string
}

variable "cluster_name" {
  description = "The name of the GKE cluster"
  type        = string
}

variable "region" {
  description = "The GCP region"
  type        = string
}

variable "zone" {
  description = "The GCP zone"
  type        = string
}

variable "network" {
  description = "The name of the VPC network"
  type        = string
}

variable "subnetwork" {
  description = "The name of the subnet"
  type        = string
}

variable "secondary_range_pods" {
  description = "The secondary range for pods"
  type        = string
}

variable "secondary_range_services" {
  description = "The secondary range for services"
  type        = string
}

variable "node_pool_config" {
  description = "Node pool configuration"
  type = object({
    machine_type   = string
    min_count      = number
    max_count      = number
    disk_size_gb   = number
    disk_type      = string
    preemptible    = bool
  })
}
```

#### `modules/gke-cluster/outputs.tf`

```hcl
output "cluster_name" {
  description = "GKE cluster name"
  value       = google_container_cluster.primary.name
}

output "cluster_endpoint" {
  description = "GKE cluster endpoint"
  value       = google_container_cluster.primary.endpoint
}

output "cluster_ca_certificate" {
  description = "GKE cluster CA certificate"
  value       = google_container_cluster.primary.master_auth[0].cluster_ca_certificate
}

output "endpoint" {
  value = google_container_cluster.primary.endpoint
}

output "ca_certificate" {
  value = google_container_cluster.primary.master_auth[0].cluster_ca_certificate
}
```

## Namespace Management

### Namespaces Module

#### `modules/namespaces/main.tf`

```hcl
resource "kubernetes_namespace" "team_namespaces" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    name = each.value.name
    labels = {
      "team"                                 = each.value.name
      "istio-injection"                     = "enabled"
      "pod-security.kubernetes.io/enforce" = "restricted"
      "pod-security.kubernetes.io/audit"   = "restricted"
      "pod-security.kubernetes.io/warn"    = "restricted"
    }
    annotations = {
      "description" = each.value.description
    }
  }
}

# Resource Quotas
resource "kubernetes_resource_quota" "team_quotas" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    name      = "${each.value.name}-quota"
    namespace = kubernetes_namespace.team_namespaces[each.key].metadata[0].name
  }

  spec {
    hard = {
      "requests.cpu"                    = each.value.cpu_request
      "requests.memory"                 = each.value.memory_request
      "limits.cpu"                      = each.value.cpu_limit
      "limits.memory"                   = each.value.memory_limit
      "count/pods"                      = each.value.pod_limit
      "count/services"                  = "5"
      "count/secrets"                   = "10"
      "count/configmaps"                = "10"
      "count/persistentvolumeclaims"    = "4"
      "count/services.loadbalancers"    = "2"
      "count/services.nodeports"        = "0"
    }
  }
}

# Limit Ranges
resource "kubernetes_limit_range" "team_limit_ranges" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    name      = "${each.value.name}-limit-range"
    namespace = kubernetes_namespace.team_namespaces[each.key].metadata[0].name
  }

  spec {
    limit {
      type = "Container"
      default = {
        cpu    = "500m"
        memory = "512Mi"
      }
      default_request = {
        cpu    = "100m"
        memory = "128Mi"
      }
      max = {
        cpu    = "2"
        memory = "4Gi"
      }
      min = {
        cpu    = "50m"
        memory = "64Mi"
      }
    }
  }
}

# Network Policies
resource "kubernetes_network_policy" "team_network_policies" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    name      = "${each.value.name}-network-policy"
    namespace = kubernetes_namespace.team_namespaces[each.key].metadata[0].name
  }

  spec {
    pod_selector {}

    policy_types = ["Ingress", "Egress"]

    # Allow ingress within the same namespace
    ingress {
      from {
        namespace_selector {
          match_labels = {
            name = each.value.name
          }
        }
      }
    }

    # Allow egress to DNS, same namespace, and internet
    egress {
      to {
        namespace_selector {
          match_labels = {
            name = "kube-system"
          }
        }
      }
      ports {
        protocol = "UDP"
        port     = "53"
      }
    }

    egress {
      to {
        namespace_selector {
          match_labels = {
            name = each.value.name
          }
        }
      }
    }

    egress {
      to {}
    }
  }
}
```

#### `modules/namespaces/variables.tf`

```hcl
variable "teams" {
  description = "List of teams for namespace creation"
  type = list(object({
    name        = string
    description = string
    cpu_limit   = string
    memory_limit = string
    cpu_request = string
    memory_request = string
    pod_limit   = number
  }))
}
```

## Role-Based Access Control (RBAC)

### RBAC Module

#### `modules/rbac/main.tf`

```hcl
# Developer Role for each namespace
resource "kubernetes_role" "developer" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    namespace = each.value.name
    name      = "developer"
  }

  rule {
    api_groups = [""]
    resources  = ["pods", "services", "serviceaccounts", "endpoints", "persistentvolumeclaims", "configmaps", "secrets"]
    verbs      = ["create", "get", "list", "update", "patch", "watch", "delete"]
  }

  rule {
    api_groups = ["apps"]
    resources  = ["deployments", "replicasets", "daemonsets", "statefulsets"]
    verbs      = ["create", "get", "list", "update", "patch", "watch", "delete"]
  }

  rule {
    api_groups = ["batch"]
    resources  = ["jobs", "cronjobs"]
    verbs      = ["create", "get", "list", "update", "patch", "watch", "delete"]
  }

  rule {
    api_groups = ["networking.k8s.io"]
    resources  = ["networkpolicies", "ingresses"]
    verbs      = ["create", "get", "list", "update", "patch", "watch", "delete"]
  }

  rule {
    api_groups = ["autoscaling"]
    resources  = ["horizontalpodautoscalers"]
    verbs      = ["create", "get", "list", "update", "patch", "watch", "delete"]
  }

  rule {
    api_groups = [""]
    resources  = ["events"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = [""]
    resources  = ["pods/log", "pods/exec", "pods/portforward"]
    verbs      = ["get", "list", "create"]
  }
}

# Role Bindings for Service Accounts
resource "kubernetes_role_binding" "team_developers" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    name      = "${each.value.name}-developers"
    namespace = each.value.name
  }

  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "Role"
    name      = kubernetes_role.developer[each.key].metadata[0].name
  }

  subject {
    kind      = "User"
    name      = var.service_accounts[each.key].email
    api_group = "rbac.authorization.k8s.io"
  }
}

# Read-only role for monitoring and observability
resource "kubernetes_role" "viewer" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    namespace = each.value.name
    name      = "viewer"
  }

  rule {
    api_groups = [""]
    resources  = ["pods", "services", "endpoints", "persistentvolumeclaims", "configmaps"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = ["apps"]
    resources  = ["deployments", "replicasets", "daemonsets", "statefulsets"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = ["batch"]
    resources  = ["jobs", "cronjobs"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = [""]
    resources  = ["events"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = [""]
    resources  = ["pods/log"]
    verbs      = ["get", "list"]
  }
}

# Cluster-level view permissions for teams
resource "kubernetes_cluster_role" "namespace_viewer" {
  metadata {
    name = "namespace-viewer"
  }

  rule {
    api_groups = [""]
    resources  = ["namespaces"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = [""]
    resources  = ["nodes"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = ["metrics.k8s.io"]
    resources  = ["nodes", "pods"]
    verbs      = ["get", "list"]
  }
}

resource "kubernetes_cluster_role_binding" "team_namespace_viewers" {
  for_each = { for team in var.teams : team.name => team }

  metadata {
    name = "${each.value.name}-namespace-viewer"
  }

  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = kubernetes_cluster_role.namespace_viewer.metadata[0].name
  }

  subject {
    kind      = "User"
    name      = var.service_accounts[each.key].email
    api_group = "rbac.authorization.k8s.io"
  }
}
```

#### `modules/rbac/variables.tf`

```hcl
variable "teams" {
  description = "List of teams"
  type = list(object({
    name        = string
    description = string
    cpu_limit   = string
    memory_limit = string
    cpu_request = string
    memory_request = string
    pod_limit   = number
  }))
}

variable "service_accounts" {
  description = "Map of service accounts for teams"
  type        = map(any)
}
```

## Resource Quotas and Limits

### Advanced Resource Management

#### `manifests/resource-quotas/priority-classes.yaml`

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority class for critical workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 500
globalDefault: true
description: "Medium priority class for regular workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority class for batch workloads"
```

#### Pod Disruption Budgets Template

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ${TEAM_NAME}-pdb
  namespace: ${TEAM_NAME}
spec:
  minAvailable: 1
  selector:
    matchLabels:
      team: ${TEAM_NAME}
```

## Monitoring and Cost Optimization

### Monitoring Setup

#### `manifests/monitoring/prometheus-config.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        target_label: __address__
        replacement: '${1}:9100'
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```

### Cost Monitoring Queries

#### BigQuery Cost Analysis Queries

```sql
-- Daily cost breakdown by namespace
SELECT
  DATE(usage_start_time) as date,
  namespace,
  resource_name,
  SUM(cost) as daily_cost
FROM `${PROJECT_ID}.gke_usage_metering.usage_metering_cost_breakdown`
WHERE namespace NOT IN ('kube-system', 'kube-public', 'kube-node-lease')
GROUP BY date, namespace, resource_name
ORDER BY date DESC, daily_cost DESC;

-- Resource utilization efficiency
SELECT
  namespace,
  AVG(cpu_utilization) as avg_cpu_utilization,
  AVG(memory_utilization) as avg_memory_utilization,
  SUM(cost) as total_cost,
  SUM(cost) / NULLIF(AVG(cpu_utilization), 0) as cost_per_cpu_utilization
FROM `${PROJECT_ID}.gke_usage_metering.gke_cluster_resource_usage`
WHERE namespace NOT IN ('kube-system', 'kube-public', 'kube-node-lease')
GROUP BY namespace
ORDER BY cost_per_cpu_utilization DESC;

-- Namespace resource quota utilization
SELECT
  namespace,
  DATE(usage_start_time) as date,
  resource_name,
  SUM(request) as total_requests,
  MAX(limit) as quota_limit,
  (SUM(request) / MAX(limit)) * 100 as utilization_percentage
FROM `${PROJECT_ID}.gke_usage_metering.gke_cluster_resource_consumption`
WHERE namespace NOT IN ('kube-system', 'kube-public', 'kube-node-lease')
  AND limit > 0
GROUP BY namespace, date, resource_name
ORDER BY utilization_percentage DESC;
```

### Grafana Dashboard Configuration

#### `manifests/monitoring/grafana-dashboard.json`

```json
{
  "dashboard": {
    "id": null,
    "title": "GKE Multi-Tenant Resource Usage",
    "tags": ["gke", "multi-tenant"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "CPU Usage by Namespace",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total{namespace!~\"kube-.*\"}[5m])) by (namespace)",
            "legendFormat": "{{namespace}}"
          }
        ]
      },
      {
        "id": 2,
        "title": "Memory Usage by Namespace",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(container_memory_usage_bytes{namespace!~\"kube-.*\"}) by (namespace)",
            "legendFormat": "{{namespace}}"
          }
        ]
      },
      {
        "id": 3,
        "title": "Pod Count by Namespace",
        "type": "graph",
        "targets": [
          {
            "expr": "count by (namespace) (kube_pod_info{namespace!~\"kube-.*\"})",
            "legendFormat": "{{namespace}}"
          }
        ]
      }
    ]
  }
}
```

## Best Practices

### Security Best Practices

1. **Pod Security Standards**

   - Implement Pod Security Standards at the namespace level
   - Use restricted profile for production workloads
   - Regularly audit and update security policies

2. **Network Segmentation**

   - Implement Network Policies for each namespace
   - Use Istio service mesh for advanced traffic management
   - Regular network policy testing and validation

3. **RBAC Principle of Least Privilege**
   - Grant minimal required permissions
   - Regular RBAC audits and reviews
   - Use service accounts instead of user accounts for applications

### Resource Management Best Practices

1. **Resource Quotas**

   - Set both requests and limits quotas
   - Monitor quota utilization regularly
   - Implement alerts for quota violations

2. **Quality of Service Classes**

   - Use Guaranteed QoS for critical workloads
   - Implement proper resource requests and limits
   - Use Burstable QoS for variable workloads

3. **Horizontal Pod Autoscaling**
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: app-hpa
     namespace: team-a
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: app-deployment
     minReplicas: 2
     maxReplicas: 10
     metrics:
       - type: Resource
         resource:
           name: cpu
           target:
             type: Utilization
             averageUtilization: 70
       - type: Resource
         resource:
           name: memory
           target:
             type: Utilization
             averageUtilization: 80
   ```

### Cost Optimization Strategies

1. **Node Pool Optimization**

   - Use appropriate machine types for workloads
   - Implement cluster autoscaling
   - Consider spot/preemptible instances for non-critical workloads

2. **Storage Optimization**

   - Use appropriate storage classes
   - Implement storage quotas
   - Regular cleanup of unused PVCs

3. **Resource Right-sizing**
   - Regular resource usage analysis
   - Implement Vertical Pod Autoscaling recommendations
   - Use tools like GKE Autopilot for automatic optimization

### Deployment Management

#### `terraform.tfvars.example`

```hcl
project_id   = "your-gcp-project-id"
region       = "us-central1"
zone         = "us-central1-a"
cluster_name = "production-multi-tenant"

teams = [
  {
    name           = "frontend"
    description    = "Frontend Development Team"
    cpu_limit      = "8"
    memory_limit   = "16Gi"
    cpu_request    = "4"
    memory_request = "8Gi"
    pod_limit      = 20
  },
  {
    name           = "backend"
    description    = "Backend Development Team"
    cpu_limit      = "12"
    memory_limit   = "24Gi"
    cpu_request    = "6"
    memory_request = "12Gi"
    pod_limit      = 30
  },
  {
    name           = "data"
    description    = "Data Engineering Team"
    cpu_limit      = "16"
    memory_limit   = "32Gi"
    cpu_request    = "8"
    memory_request = "16Gi"
    pod_limit      = 25
  }
]

node_pool_config = {
  machine_type = "e2-standard-8"
  min_count    = 2
  max_count    = 20
  disk_size_gb = 100
  disk_type    = "pd-ssd"
  preemptible  = false
}
```

## Troubleshooting

### Common Issues and Solutions

#### Resource Quota Exceeded

```bash
# Check resource quota status
kubectl describe quota -n <namespace>

# Check current resource usage
kubectl top pods -n <namespace>
kubectl top nodes

# Increase quota if justified
kubectl patch resourcequota <quota-name> -n <namespace> --patch '{"spec":{"hard":{"cpu":"8"}}}'
```

#### RBAC Permission Denied

```bash
# Check user permissions
kubectl auth can-i create pods --namespace=<namespace> --as=<user>

# Verify role bindings
kubectl get rolebindings -n <namespace>
kubectl describe rolebinding <binding-name> -n <namespace>

# Check service account permissions
kubectl get serviceaccounts -n <namespace>
kubectl describe sa <service-account> -n <namespace>
```

#### Network Policy Issues

```bash
# Test network connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Check network policies
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>

# Verify DNS resolution
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

#### Resource Monitoring

```bash
# Monitor resource usage
kubectl top pods --all-namespaces
kubectl top nodes

# Check cluster events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Analyze resource consumption
kubectl describe node <node-name>
```

### Useful Commands

#### Deployment Commands

```bash
# Initialize Terraform
terraform init

# Plan deployment
terraform plan -var-file="terraform.tfvars"

# Apply configuration
terraform apply -var-file="terraform.tfvars"

# Get cluster credentials
gcloud container clusters get-credentials <cluster-name> --zone <zone>

# Verify namespace creation
kubectl get namespaces

# Check resource quotas
kubectl get resourcequotas --all-namespaces

# Verify RBAC setup
kubectl get roles,rolebindings --all-namespaces
```

#### Monitoring Commands

```bash
# Check BigQuery datasets
bq ls

# Query usage data
bq query --use_legacy_sql=false 'SELECT * FROM `project.dataset.table` LIMIT 10'

# Monitor cluster metrics
kubectl top pods --all-namespaces
kubectl top nodes

# Check resource usage
kubectl describe quota -n <namespace>
```

This playbook provides a comprehensive foundation for managing multi-tenant GKE clusters using Terraform with proper namespace isolation, RBAC, resource management, and cost optimization strategies.
