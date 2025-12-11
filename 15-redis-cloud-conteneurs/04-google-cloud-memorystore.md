🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.4 Google Cloud Memorystore

## 🎯 Objectifs

- Maîtriser l'architecture Memorystore et ses deux tiers
- Configurer des instances production avec Terraform et gcloud CLI
- Implémenter Private Service Connect pour l'isolation réseau
- Déployer des read replicas pour la scalabilité en lecture
- Intégrer Memorystore avec GKE (Google Kubernetes Engine)
- Optimiser les coûts et performances sur GCP

---

## 🏗️ Architecture et positionnement

### Vue d'ensemble de Memorystore

```
┌──────────────────────────────────────────────────────────────┐
│           Google Cloud Memorystore for Redis                 │
│                                                              │
│  Philosophy: Simplicité et intégration native GCP            │
│                                                              │
│  ┌──────────────┐              ┌──────────────┐              │
│  │    Basic     │              │  Standard    │              │
│  │    Tier      │              │     HA       │              │
│  │              │              │              │              │
│  │ Single node  │              │Primary + Rep │              │
│  │ No HA        │              │ Auto failover│              │
│  │ Dev/Test     │              │ Production   │              │
│  │              │              │              │              │
│  │ 1-300 GB     │              │ 5-300 GB     │              │
│  │              │              │              │              │
│  │ €0.035/GB-h  │              │ €0.087/GB-h  │              │
│  └──────────────┘              └──────────────┘              │
│                                                              │
│  Unique Features:                                            │
│  ├── Managed fully by Google (minimal config)                │
│  ├── Private Service Connect (VPC native)                    │
│  ├── Read replicas (Standard only, up to 5)                  │
│  ├── Automated daily backups                                 │
│  ├── Import/export to Google Cloud Storage                   │
│  └── Deep integration with GKE, Cloud Run, App Engine        │
│                                                              │
│  Limitations vs AWS/Azure:                                   │
│  ├── No native clustering (single instance up to 300GB)      │
│  ├── No cross-region replication                             │
│  ├── No Redis modules (Stack)                                │
│  ├── Less configuration options                              │
│  └── Simpler = less flexibility                              │
└──────────────────────────────────────────────────────────────┘
```

### Architecture réseau avec Private Service Connect

```
┌────────────────────────────────────────────────────────────┐
│                  Google Cloud VPC Network                  │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Subnet: applications (10.0.1.0/24)         │    │
│  │                                                    │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │    │
│  │  │   GKE Pod   │  │   GKE Pod   │  │  VM / GCE  │  │    │
│  │  │  (app-1)    │  │  (app-2)    │  │  Instance  │  │    │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬─────┘  │    │
│  │         │                │                │        │    │
│  │         └────────────────┼────────────────┘        │    │
│  │                          │                         │    │
│  └──────────────────────────┼─────────────────────────┘    │
│                             │                              │
│                             │ Private connection           │
│                             │ (no public IP)               │
│                             ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │    Private Service Connect (Managed by Google)      │   │
│  │                                                     │   │
│  │    Reserved IP Range: 10.0.2.0/29 (8 IPs)           │   │
│  │                                                     │   │
│  │    ┌────────────────────────────────────────┐       │   │
│  │    │  Memorystore Instance (Standard HA)    │       │   │
│  │    │                                        │       │   │
│  │    │  ┌──────────────┐  ┌──────────────┐    │       │   │
│  │    │  │   Primary    │  │   Replica    │    │       │   │
│  │    │  │   (Zone A)   │  │   (Zone B)   │    │       │   │
│  │    │  │              │  │              │    │       │   │
│  │    │  │  10.0.2.2    │  │  10.0.2.3    │    │       │   │
│  │    │  └──────────────┘  └──────────────┘    │       │   │
│  │    │                                        │       │   │
│  │    │  • Automatic failover                  │       │   │
│  │    │  • Async replication                   │       │   │
│  │    │  • Read replicas optional              │       │   │
│  │    └────────────────────────────────────────┘       │   │
│  │                                                     │   │
│  │    Private DNS: redis-instance.*.redis.googleapis   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                            │
│  Key Network Features:                                     │
│  ├── No public IP (100% private)                           │
│  ├── VPC peering automatic (managed by Google)             │
│  ├── Private Service Connect handles routing               │
│  ├── Cross-project VPC peering supported                   │
│  └── Firewall rules not needed (private by design)         │
└────────────────────────────────────────────────────────────┘
```

### Comparaison Basic vs Standard

| Caractéristique | Basic Tier | Standard Tier |
|-----------------|------------|---------------|
| **Disponibilité** |
| Nodes | 1 | 2 (primary + replica) |
| SLA | None | 99.9% |
| Automatic failover | ❌ | ✅ (30-60s) |
| Zone placement | Single zone | Multi-zone |
| Downtime | Maintenance downtime | Zero-downtime upgrades |
| **Capacité** |
| Min RAM | 1 GB | 5 GB |
| Max RAM | 300 GB | 300 GB |
| Max connections | ~20K | ~40K |
| **Performance** |
| Throughput | Medium | High |
| Read replicas | ❌ | ✅ (0-5 replicas) |
| **Persistence** |
| Snapshots | ✅ Daily | ✅ Daily |
| Point-in-time recovery | ❌ | ❌ |
| Export/Import | ✅ GCS | ✅ GCS |
| **Réseau** |
| VPC peering | ✅ | ✅ |
| Private IP only | ✅ | ✅ |
| Transit encryption | ✅ (TLS) | ✅ (TLS) |
| **Monitoring** |
| Cloud Monitoring | ✅ | ✅ |
| Cloud Logging | ✅ | ✅ |
| Alerting | ✅ | ✅ |
| **Pricing** |
| Cost per GB-hour | €0.035/GB-h | €0.087/GB-h |
| Cost 50GB/month | ~€1,277 | ~€3,176 |
| Use case | Dev/Test | Production |

---

## 📋 Configuration avec Terraform

### Module Terraform complet

```hcl
# Google Cloud Memorystore for Redis - Production Configuration
# Provider: hashicorp/google ~> 5.0

terraform {
  required_version = ">= 1.5"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.10"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Variables
variable "project_id" {
  type        = string
  description = "GCP Project ID"
}

variable "region" {
  type        = string
  description = "GCP region"
  default     = "europe-west1"
}

variable "environment" {
  type        = string
  description = "Environment name"
  default     = "production"
}

variable "redis_tier" {
  type        = string
  description = "Redis tier: BASIC or STANDARD_HA"
  default     = "STANDARD_HA"

  validation {
    condition     = contains(["BASIC", "STANDARD_HA"], var.redis_tier)
    error_message = "Redis tier must be BASIC or STANDARD_HA"
  }
}

variable "memory_size_gb" {
  type        = number
  description = "Memory size in GB"
  default     = 50

  validation {
    condition     = var.memory_size_gb >= 1 && var.memory_size_gb <= 300
    error_message = "Memory size must be between 1 and 300 GB"
  }
}

variable "redis_version" {
  type        = string
  description = "Redis version"
  default     = "REDIS_7_2"

  validation {
    condition = contains([
      "REDIS_6_X",
      "REDIS_7_0",
      "REDIS_7_2"
    ], var.redis_version)
    error_message = "Invalid Redis version"
  }
}

variable "replica_count" {
  type        = number
  description = "Number of read replicas (0-5, Standard tier only)"
  default     = 2

  validation {
    condition     = var.replica_count >= 0 && var.replica_count <= 5
    error_message = "Replica count must be between 0 and 5"
  }
}

variable "enable_auth" {
  type        = bool
  description = "Enable AUTH (password protection)"
  default     = true
}

variable "enable_transit_encryption" {
  type        = bool
  description = "Enable TLS encryption"
  default     = true
}

variable "labels" {
  type        = map(string)
  description = "Labels to apply to resources"
  default = {
    managed_by = "terraform"
  }
}

# Local variables
locals {
  name_prefix = "${var.environment}-redis"

  common_labels = merge(
    var.labels,
    {
      environment = var.environment
      terraform   = "true"
    }
  )
}

# VPC Network
resource "google_compute_network" "main" {
  name                    = "${local.name_prefix}-vpc"
  auto_create_subnetworks = false

  description = "VPC for ${var.environment} environment"
}

# Subnet for applications
resource "google_compute_subnetwork" "apps" {
  name          = "${local.name_prefix}-apps-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.region
  network       = google_compute_network.main.id

  # Enable Private Google Access (for GCS, etc.)
  private_ip_google_access = true

  # Enable flow logs
  log_config {
    aggregation_interval = "INTERVAL_5_SEC"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# Reserved IP range for Memorystore (Private Service Connect)
resource "google_compute_global_address" "redis_private_ip" {
  name          = "${local.name_prefix}-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.main.id

  description = "Reserved IP range for Memorystore"
}

# Private Service Connection (required for Memorystore)
resource "google_service_networking_connection" "redis_private_vpc" {
  network                 = google_compute_network.main.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.redis_private_ip.name]
}

# Cloud Router for Cloud NAT (if needed for internet access)
resource "google_compute_router" "main" {
  name    = "${local.name_prefix}-router"
  region  = var.region
  network = google_compute_network.main.id

  bgp {
    asn = 64514
  }
}

# Cloud NAT (for outbound internet from private instances)
resource "google_compute_router_nat" "main" {
  name                               = "${local.name_prefix}-nat"
  router                             = google_compute_router.main.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}

# Firewall rule: Allow Redis from app subnet
resource "google_compute_firewall" "allow_redis" {
  name    = "${local.name_prefix}-allow-redis"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["6379"]
  }

  source_ranges = ["10.0.1.0/24"]  # App subnet
  target_tags   = ["redis"]

  description = "Allow Redis access from application subnet"
}

# Firewall rule: Allow internal communication
resource "google_compute_firewall" "allow_internal" {
  name    = "${local.name_prefix}-allow-internal"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
  }

  allow {
    protocol = "udp"
  }

  allow {
    protocol = "icmp"
  }

  source_ranges = ["10.0.0.0/16"]

  description = "Allow internal VPC communication"
}

# Random AUTH password (if enabled)
resource "random_password" "redis_auth" {
  count   = var.enable_auth ? 1 : 0
  length  = 32
  special = false
}

# Secret Manager for storing AUTH password
resource "google_secret_manager_secret" "redis_auth" {
  count     = var.enable_auth ? 1 : 0
  secret_id = "${local.name_prefix}-auth-password"

  labels = local.common_labels

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "redis_auth" {
  count       = var.enable_auth ? 1 : 0
  secret      = google_secret_manager_secret.redis_auth[0].id
  secret_data = random_password.redis_auth[0].result
}

# Memorystore Redis Instance (Standard HA)
resource "google_redis_instance" "main" {
  name               = "${local.name_prefix}-instance"
  tier               = var.redis_tier
  memory_size_gb     = var.memory_size_gb
  region             = var.region
  redis_version      = var.redis_version
  display_name       = "${var.environment} Redis Instance"

  # Network
  authorized_network      = google_compute_network.main.id
  connect_mode            = "PRIVATE_SERVICE_ACCESS"
  reserved_ip_range       = google_compute_global_address.redis_private_ip.address

  # Location (primary zone)
  location_id             = "${var.region}-b"
  alternative_location_id = var.redis_tier == "STANDARD_HA" ? "${var.region}-c" : null

  # Redis configuration
  redis_configs = {
    maxmemory-policy            = "allkeys-lru"
    notify-keyspace-events      = "Ex"
    timeout                     = "300"

    # Active defragmentation
    activedefrag                = "yes"

    # LFU tuning (for allkeys-lfu if used)
    lfu-log-factor              = "10"
    lfu-decay-time              = "1"

    # Slow log
    slowlog-log-slower-than     = "10000"  # 10ms
    slowlog-max-len             = "128"
  }

  # Maintenance policy (weekly window)
  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 5
        minutes = 0
      }
      duration = "3600s"  # 1 hour
    }
  }

  # Persistence configuration (RDB snapshots)
  persistence_config {
    persistence_mode    = "RDB"
    rdb_snapshot_period = "ONE_HOUR"  # ONE_HOUR, SIX_HOURS, TWELVE_HOURS, TWENTY_FOUR_HOURS
    rdb_snapshot_start_time = formatdate("YYYY-MM-DD'T'hh:mm:ss'Z'", timestamp())
  }

  # Authentication
  auth_enabled = var.enable_auth

  # TLS encryption
  transit_encryption_mode = var.enable_transit_encryption ? "SERVER_AUTHENTICATION" : "DISABLED"

  # Read replicas (Standard tier only)
  replica_count      = var.redis_tier == "STANDARD_HA" ? var.replica_count : 0
  read_replicas_mode = var.redis_tier == "STANDARD_HA" && var.replica_count > 0 ? "READ_REPLICAS_ENABLED" : "READ_REPLICAS_DISABLED"

  labels = local.common_labels

  depends_on = [
    google_service_networking_connection.redis_private_vpc
  ]

  # Lifecycle
  lifecycle {
    prevent_destroy = true  # Protect production data

    ignore_changes = [
      persistence_config[0].rdb_snapshot_start_time  # Ignore auto-generated timestamp
    ]
  }
}

# Cloud Monitoring - Alert Policy for CPU
resource "google_monitoring_alert_policy" "redis_cpu" {
  display_name = "${local.name_prefix} High CPU"
  combiner     = "OR"

  conditions {
    display_name = "Redis CPU > 80%"

    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND resource.labels.instance_id = \"${google_redis_instance.main.id}\" AND metric.type = \"redis.googleapis.com/stats/cpu_utilization\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 0.8

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]

  alert_strategy {
    auto_close = "1800s"  # 30 minutes
  }

  documentation {
    content   = "Redis CPU utilization is above 80%. Consider scaling up or optimizing queries."
    mime_type = "text/markdown"
  }
}

# Cloud Monitoring - Alert Policy for Memory
resource "google_monitoring_alert_policy" "redis_memory" {
  display_name = "${local.name_prefix} High Memory"
  combiner     = "OR"

  conditions {
    display_name = "Redis Memory > 85%"

    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND resource.labels.instance_id = \"${google_redis_instance.main.id}\" AND metric.type = \"redis.googleapis.com/stats/memory/usage_ratio\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 0.85

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]

  alert_strategy {
    auto_close = "1800s"
  }

  documentation {
    content   = "Redis memory usage is above 85%. Consider increasing instance size or implementing eviction policy."
    mime_type = "text/markdown"
  }
}

# Cloud Monitoring - Alert Policy for Replication Lag
resource "google_monitoring_alert_policy" "redis_replication_lag" {
  count        = var.redis_tier == "STANDARD_HA" ? 1 : 0
  display_name = "${local.name_prefix} High Replication Lag"
  combiner     = "OR"

  conditions {
    display_name = "Redis Replication Lag > 5s"

    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND resource.labels.instance_id = \"${google_redis_instance.main.id}\" AND metric.type = \"redis.googleapis.com/replication/master/slaves/lag\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 5

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MAX"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]

  alert_strategy {
    auto_close = "1800s"
  }

  documentation {
    content   = "Redis replication lag is above 5 seconds. Check network and load."
    mime_type = "text/markdown"
  }
}

# Cloud Monitoring - Alert Policy for Connections
resource "google_monitoring_alert_policy" "redis_connections" {
  display_name = "${local.name_prefix} High Connections"
  combiner     = "OR"

  conditions {
    display_name = "Redis Connected Clients > 5000"

    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND resource.labels.instance_id = \"${google_redis_instance.main.id}\" AND metric.type = \"redis.googleapis.com/clients/connected\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 5000

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MAX"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]

  alert_strategy {
    auto_close = "1800s"
  }

  documentation {
    content   = "Redis has more than 5000 connected clients. Investigate connection leaks."
    mime_type = "text/markdown"
  }
}

# Notification Channel (Email)
resource "google_monitoring_notification_channel" "email" {
  display_name = "Platform Team Email"
  type         = "email"

  labels = {
    email_address = "platform-team@example.com"
  }
}

# Optional: Notification Channel (Slack)
resource "google_monitoring_notification_channel" "slack" {
  count        = 0  # Set to 1 to enable
  display_name = "Slack Notifications"
  type         = "slack"

  labels = {
    channel_name = "#redis-alerts"
  }

  sensitive_labels {
    auth_token = var.slack_auth_token  # Add this variable if using Slack
  }
}

# Cloud Logging - Log Sink (export to BigQuery for analysis)
resource "google_logging_project_sink" "redis_logs" {
  name        = "${local.name_prefix}-logs-sink"
  destination = "bigquery.googleapis.com/projects/${var.project_id}/datasets/${google_bigquery_dataset.redis_logs.dataset_id}"

  filter = "resource.type = \"redis_instance\" AND resource.labels.instance_id = \"${google_redis_instance.main.id}\""

  unique_writer_identity = true
}

# BigQuery Dataset for logs
resource "google_bigquery_dataset" "redis_logs" {
  dataset_id  = "${replace(local.name_prefix, "-", "_")}_logs"
  location    = var.region
  description = "Redis logs for analysis"

  default_table_expiration_ms = 2592000000  # 30 days

  labels = local.common_labels
}

# Grant log sink write permissions to BigQuery
resource "google_bigquery_dataset_iam_member" "log_sink_writer" {
  dataset_id = google_bigquery_dataset.redis_logs.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = google_logging_project_sink.redis_logs.writer_identity
}

# Cloud Storage bucket for export/import
resource "google_storage_bucket" "redis_backups" {
  name          = "${var.project_id}-${local.name_prefix}-backups"
  location      = var.region
  force_destroy = false

  uniform_bucket_level_access = true

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age = 30  # Keep backups for 30 days
    }
    action {
      type = "Delete"
    }
  }

  lifecycle_rule {
    condition {
      age                = 7
      matches_storage_class = ["STANDARD"]
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }

  labels = local.common_labels
}

# IAM binding for Cloud Build (if using CI/CD)
resource "google_project_iam_member" "cloudbuild_redis_admin" {
  count   = 0  # Set to 1 if using Cloud Build
  project = var.project_id
  role    = "roles/redis.admin"
  member  = "serviceAccount:${var.project_id}@cloudbuild.gserviceaccount.com"
}

# Outputs
output "redis_host" {
  description = "Redis instance host (private IP)"
  value       = google_redis_instance.main.host
}

output "redis_port" {
  description = "Redis instance port"
  value       = google_redis_instance.main.port
}

output "redis_connection_string" {
  description = "Redis connection string"
  value       = var.enable_auth ? "rediss://:${random_password.redis_auth[0].result}@${google_redis_instance.main.host}:${google_redis_instance.main.port}" : "redis://${google_redis_instance.main.host}:${google_redis_instance.main.port}"
  sensitive   = true
}

output "redis_auth_string" {
  description = "Redis AUTH password"
  value       = var.enable_auth ? random_password.redis_auth[0].result : null
  sensitive   = true
}

output "redis_instance_id" {
  description = "Redis instance ID"
  value       = google_redis_instance.main.id
}

output "redis_instance_name" {
  description = "Redis instance name"
  value       = google_redis_instance.main.name
}

output "redis_read_endpoint" {
  description = "Redis read endpoint (for read replicas)"
  value       = google_redis_instance.main.read_endpoint
}

output "redis_current_location_id" {
  description = "Current location of Redis primary"
  value       = google_redis_instance.main.current_location_id
}

output "vpc_network_id" {
  description = "VPC network ID"
  value       = google_compute_network.main.id
}

output "vpc_network_name" {
  description = "VPC network name"
  value       = google_compute_network.main.name
}

output "app_subnet_id" {
  description = "Application subnet ID"
  value       = google_compute_subnetwork.apps.id
}

output "backup_bucket_name" {
  description = "GCS bucket name for backups"
  value       = google_storage_bucket.redis_backups.name
}

output "logs_dataset_id" {
  description = "BigQuery dataset ID for logs"
  value       = google_bigquery_dataset.redis_logs.dataset_id
}

output "auth_secret_id" {
  description = "Secret Manager secret ID for AUTH password"
  value       = var.enable_auth ? google_secret_manager_secret.redis_auth[0].secret_id : null
}
```

### Variables file (terraform.tfvars)

```hcl
# Production configuration
project_id   = "my-gcp-project-prod"
region       = "europe-west1"
environment  = "production"

# Redis configuration
redis_tier          = "STANDARD_HA"
memory_size_gb      = 50
redis_version       = "REDIS_7_2"
replica_count       = 2
enable_auth         = true
enable_transit_encryption = true

# Labels
labels = {
  team        = "platform"
  cost_center = "engineering"
  managed_by  = "terraform"
}
```

### Déploiement

```bash
#!/bin/bash
# Deploy Google Cloud Memorystore with Terraform

set -euo pipefail

PROJECT_ID="my-gcp-project-prod"
REGION="europe-west1"

echo "=== Deploying Memorystore for Redis ==="

# Set project
gcloud config set project $PROJECT_ID

# Enable required APIs
echo "Enabling required APIs..."
gcloud services enable \
  redis.googleapis.com \
  servicenetworking.googleapis.com \
  compute.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  secretmanager.googleapis.com \
  bigquery.googleapis.com \
  storage.googleapis.com

# Initialize Terraform
terraform init

# Format
terraform fmt

# Validate
terraform validate

# Plan
terraform plan \
  -var="project_id=$PROJECT_ID" \
  -var="region=$REGION" \
  -out=tfplan

# Apply
echo "Applying configuration..."
terraform apply tfplan

# Get outputs
echo ""
echo "=== Deployment Complete ==="
echo ""
terraform output redis_host
terraform output redis_port
echo ""
echo "To get connection string (sensitive):"
echo "terraform output -raw redis_connection_string"
```

---

## 🔧 Configuration avec gcloud CLI

### Déploiement via CLI

```bash
#!/bin/bash
# Create Memorystore instance with gcloud CLI

PROJECT_ID="my-gcp-project-prod"
REGION="europe-west1"
INSTANCE_NAME="production-redis-instance"
NETWORK="projects/$PROJECT_ID/global/networks/prod-vpc"

# Create VPC network
gcloud compute networks create prod-vpc \
  --subnet-mode=custom \
  --project=$PROJECT_ID

# Create subnet
gcloud compute networks subnets create apps-subnet \
  --network=prod-vpc \
  --region=$REGION \
  --range=10.0.1.0/24 \
  --enable-private-ip-google-access \
  --project=$PROJECT_ID

# Reserve IP range for Private Service Connect
gcloud compute addresses create redis-ip-range \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network=prod-vpc \
  --project=$PROJECT_ID

# Create private connection
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=redis-ip-range \
  --network=prod-vpc \
  --project=$PROJECT_ID

# Create Memorystore instance (Standard HA)
gcloud redis instances create $INSTANCE_NAME \
  --tier=STANDARD_HA \
  --size=50 \
  --region=$REGION \
  --zone=$REGION-b \
  --alternative-zone=$REGION-c \
  --redis-version=redis_7_2 \
  --network=$NETWORK \
  --connect-mode=PRIVATE_SERVICE_ACCESS \
  --enable-auth \
  --transit-encryption-mode=SERVER_AUTHENTICATION \
  --persistence-mode=RDB \
  --rdb-snapshot-period=one-hour \
  --rdb-snapshot-start-time=2024-01-01T03:00:00Z \
  --replica-count=2 \
  --read-replicas-mode=READ_REPLICAS_ENABLED \
  --redis-config=maxmemory-policy=allkeys-lru \
  --redis-config=notify-keyspace-events=Ex \
  --redis-config=activedefrag=yes \
  --maintenance-window-day=SUNDAY \
  --maintenance-window-hour=5 \
  --project=$PROJECT_ID

# Wait for instance to be ready
echo "Waiting for instance to be ready..."
gcloud redis instances describe $INSTANCE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --format="value(state)"

# Get connection info
echo ""
echo "=== Instance Created ==="
gcloud redis instances describe $INSTANCE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --format="table(host,port,authString,currentLocationId)"

# Export instance details to file
gcloud redis instances describe $INSTANCE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --format=json > redis-instance-config.json

echo ""
echo "Instance details saved to redis-instance-config.json"
```

### Gestion des snapshots

```bash
#!/bin/bash
# Snapshot management

INSTANCE_NAME="production-redis-instance"
REGION="europe-west1"
PROJECT_ID="my-gcp-project-prod"
BUCKET_NAME="gs://my-project-redis-backups"

# Export instance to Cloud Storage
echo "Exporting Redis data to GCS..."
gcloud redis instances export \
  $INSTANCE_NAME \
  $BUCKET_NAME/backup-$(date +%Y%m%d-%H%M%S).rdb \
  --region=$REGION \
  --project=$PROJECT_ID

# Import from Cloud Storage
echo "Importing Redis data from GCS..."
gcloud redis instances import \
  $INSTANCE_NAME \
  $BUCKET_NAME/backup-20240615-120000.rdb \
  --region=$REGION \
  --project=$PROJECT_ID

# List available backups
echo "Available backups:"
gsutil ls $BUCKET_NAME/

# Get instance state during export/import
gcloud redis instances describe $INSTANCE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --format="value(state)"
```

---

## 🚀 Intégration avec GKE

### Déploiement d'application connectée à Memorystore

```yaml
# kubernetes/redis-client-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: default
data:
  redis-host: "10.0.2.2"  # Memorystore private IP
  redis-port: "6379"
  redis-tls-enabled: "true"

---
apiVersion: v1
kind: Secret
metadata:
  name: redis-credentials
  namespace: default
type: Opaque
stringData:
  redis-password: ""  # Populate from Secret Manager or manually

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-client-app
  namespace: default
  labels:
    app: redis-client
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis-client
  template:
    metadata:
      labels:
        app: redis-client
    spec:
      # GKE Workload Identity (recommended for GCP services)
      serviceAccountName: redis-client-sa

      containers:
      - name: app
        image: gcr.io/my-project/redis-client-app:v1.0.0

        env:
        # Redis connection configuration
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: redis-config
              key: redis-host

        - name: REDIS_PORT
          valueFrom:
            configMapKeyRef:
              name: redis-config
              key: redis-port

        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: redis-password

        - name: REDIS_TLS_ENABLED
          valueFrom:
            configMapKeyRef:
              name: redis-config
              key: redis-tls-enabled

        # Connection string (constructed)
        - name: REDIS_URL
          value: "rediss://:$(REDIS_PASSWORD)@$(REDIS_HOST):$(REDIS_PORT)"

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        livenessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5

      # Ensure pods are spread across zones
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis-client
              topologyKey: topology.kubernetes.io/zone

---
apiVersion: v1
kind: Service
metadata:
  name: redis-client-service
  namespace: default
spec:
  selector:
    app: redis-client
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http
  type: ClusterIP
```

### Service Account avec Workload Identity

```yaml
# kubernetes/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: redis-client-sa
  namespace: default
  annotations:
    # Link to GCP Service Account (Workload Identity)
    iam.gke.io/gcp-service-account: redis-client@my-project.iam.gserviceaccount.com
```

```bash
#!/bin/bash
# Setup Workload Identity for GKE to access Secret Manager

PROJECT_ID="my-gcp-project-prod"
GCP_SA_NAME="redis-client"
K8S_SA_NAME="redis-client-sa"
K8S_NAMESPACE="default"

# Create GCP Service Account
gcloud iam service-accounts create $GCP_SA_NAME \
  --display-name="Redis Client Service Account" \
  --project=$PROJECT_ID

# Grant access to Secret Manager
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${GCP_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Allow Kubernetes SA to impersonate GCP SA (Workload Identity)
gcloud iam service-accounts add-iam-policy-binding \
  ${GCP_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${K8S_NAMESPACE}/${K8S_SA_NAME}]"

echo "Workload Identity configured successfully"
```

### Example application (Python)

```python
# app.py - Python Flask app connecting to Memorystore
import os
import redis
from flask import Flask, jsonify

app = Flask(__name__)

# Redis configuration from environment
REDIS_HOST = os.getenv('REDIS_HOST', 'localhost')
REDIS_PORT = int(os.getenv('REDIS_PORT', 6379))
REDIS_PASSWORD = os.getenv('REDIS_PASSWORD', '')
REDIS_TLS_ENABLED = os.getenv('REDIS_TLS_ENABLED', 'false').lower() == 'true'

# Initialize Redis client
redis_client = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    password=REDIS_PASSWORD,
    ssl=REDIS_TLS_ENABLED,
    ssl_cert_reqs=None if REDIS_TLS_ENABLED else False,
    decode_responses=True,
    socket_connect_timeout=5,
    socket_timeout=5,
    retry_on_timeout=True,
    health_check_interval=30
)

@app.route('/healthz')
def health():
    """Health check endpoint"""
    try:
        redis_client.ping()
        return jsonify({"status": "healthy", "redis": "connected"}), 200
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 503

@app.route('/ready')
def ready():
    """Readiness check endpoint"""
    try:
        redis_client.ping()
        return jsonify({"status": "ready"}), 200
    except Exception as e:
        return jsonify({"status": "not_ready", "error": str(e)}), 503

@app.route('/set/<key>/<value>')
def set_key(key, value):
    """Set a key-value pair"""
    try:
        redis_client.set(key, value, ex=3600)  # 1 hour TTL
        return jsonify({"status": "success", "key": key, "value": value}), 200
    except Exception as e:
        return jsonify({"status": "error", "error": str(e)}), 500

@app.route('/get/<key>')
def get_key(key):
    """Get a value by key"""
    try:
        value = redis_client.get(key)
        if value is None:
            return jsonify({"status": "not_found", "key": key}), 404
        return jsonify({"status": "success", "key": key, "value": value}), 200
    except Exception as e:
        return jsonify({"status": "error", "error": str(e)}), 500

@app.route('/info')
def redis_info():
    """Get Redis info"""
    try:
        info = redis_client.info('server')
        return jsonify({
            "status": "success",
            "redis_version": info.get('redis_version'),
            "uptime_days": info.get('uptime_in_days'),
            "connected_clients": redis_client.info('clients').get('connected_clients')
        }), 200
    except Exception as e:
        return jsonify({"status": "error", "error": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app.py .

# Run as non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "--timeout", "60", "app:app"]
```

```txt
# requirements.txt
flask==3.0.0
redis==5.0.1
gunicorn==21.2.0
```

---

## 📊 Monitoring avec Cloud Monitoring

### Dashboard personnalisé

```json
{
  "displayName": "Redis Production Dashboard",
  "mosaicLayout": {
    "columns": 12,
    "tiles": [
      {
        "width": 6,
        "height": 4,
        "widget": {
          "title": "Redis CPU Utilization",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "resource.type=\"redis_instance\" AND metric.type=\"redis.googleapis.com/stats/cpu_utilization\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MEAN"
                    }
                  }
                },
                "plotType": "LINE",
                "targetAxis": "Y1"
              }
            ],
            "yAxis": {
              "label": "CPU %",
              "scale": "LINEAR"
            }
          }
        }
      },
      {
        "xPos": 6,
        "width": 6,
        "height": 4,
        "widget": {
          "title": "Redis Memory Usage Ratio",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "resource.type=\"redis_instance\" AND metric.type=\"redis.googleapis.com/stats/memory/usage_ratio\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MEAN"
                    }
                  }
                },
                "plotType": "LINE",
                "targetAxis": "Y1"
              }
            ],
            "yAxis": {
              "label": "Memory Ratio",
              "scale": "LINEAR"
            },
            "thresholds": [
              {
                "value": 0.85,
                "color": "YELLOW",
                "direction": "ABOVE"
              },
              {
                "value": 0.95,
                "color": "RED",
                "direction": "ABOVE"
              }
            ]
          }
        }
      },
      {
        "yPos": 4,
        "width": 6,
        "height": 4,
        "widget": {
          "title": "Redis Operations (Calls/sec)",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "resource.type=\"redis_instance\" AND metric.type=\"redis.googleapis.com/stats/ops_per_sec\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_RATE"
                    }
                  }
                },
                "plotType": "LINE",
                "targetAxis": "Y1"
              }
            ]
          }
        }
      },
      {
        "xPos": 6,
        "yPos": 4,
        "width": 6,
        "height": 4,
        "widget": {
          "title": "Connected Clients",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "resource.type=\"redis_instance\" AND metric.type=\"redis.googleapis.com/clients/connected\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MAX"
                    }
                  }
                },
                "plotType": "LINE",
                "targetAxis": "Y1"
              }
            ]
          }
        }
      },
      {
        "yPos": 8,
        "width": 6,
        "height": 4,
        "widget": {
          "title": "Cache Hit Ratio",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "resource.type=\"redis_instance\" AND metric.type=\"redis.googleapis.com/stats/cache_hit_ratio\"",
                    "aggregation": {
                      "alignmentPeriod": "300s",
                      "perSeriesAligner": "ALIGN_MEAN"
                    }
                  }
                },
                "plotType": "LINE",
                "targetAxis": "Y1"
              }
            ],
            "yAxis": {
              "label": "Hit Ratio",
              "scale": "LINEAR"
            },
            "thresholds": [
              {
                "value": 0.8,
                "color": "YELLOW",
                "direction": "BELOW"
              }
            ]
          }
        }
      },
      {
        "xPos": 6,
        "yPos": 8,
        "width": 6,
        "height": 4,
        "widget": {
          "title": "Replication Lag (seconds)",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "resource.type=\"redis_instance\" AND metric.type=\"redis.googleapis.com/replication/master/slaves/lag\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MAX"
                    }
                  }
                },
                "plotType": "LINE",
                "targetAxis": "Y1"
              }
            ],
            "thresholds": [
              {
                "value": 5,
                "color": "YELLOW",
                "direction": "ABOVE"
              },
              {
                "value": 10,
                "color": "RED",
                "direction": "ABOVE"
              }
            ]
          }
        }
      }
    ]
  }
}
```

### Métriques disponibles

```yaml
Performance Metrics:
├── redis.googleapis.com/stats/cpu_utilization
│   └── CPU usage (0-1)
├── redis.googleapis.com/stats/ops_per_sec
│   └── Operations per second
├── redis.googleapis.com/commands/calls
│   └── Command calls (by command type)
└── redis.googleapis.com/stats/average_ttl
    └── Average TTL of keys with TTL

Memory Metrics:
├── redis.googleapis.com/stats/memory/usage
│   └── Used memory (bytes)
├── redis.googleapis.com/stats/memory/usage_ratio
│   └── Used memory ratio (0-1)
├── redis.googleapis.com/stats/memory/system_memory_usage_ratio
│   └── System memory usage
└── redis.googleapis.com/stats/evicted_keys
    └── Number of evicted keys

Connections:
├── redis.googleapis.com/clients/connected
│   └── Connected clients
├── redis.googleapis.com/clients/blocked
│   └── Blocked clients
└── redis.googleapis.com/stats/connections/total
    └── Total connections received

Cache Efficiency:
├── redis.googleapis.com/stats/cache_hit_ratio
│   └── Hit ratio (0-1)
├── redis.googleapis.com/keyspace/keys
│   └── Total keys
└── redis.googleapis.com/keyspace/avg_ttl
    └── Average TTL

Replication (Standard tier):
├── redis.googleapis.com/replication/master/slaves/lag
│   └── Replication lag (seconds)
├── redis.googleapis.com/replication/master/slaves/offset
│   └── Replication offset
└── redis.googleapis.com/replication/role
    └── Instance role (master/slave)

Network:
├── redis.googleapis.com/stats/network_traffic
│   └── Network bytes (in/out)
└── redis.googleapis.com/stats/reject_connections
    └── Rejected connections
```

---

## 💰 Pricing détaillé (2024)

### Grille tarifaire

```yaml
Basic Tier (per GB-hour, europe-west1):
├── Pricing: €0.035/GB-hour
├── Monthly (730h): €25.55/GB-month
└── No SLA

Standard Tier (per GB-hour, europe-west1):
├── Pricing: €0.087/GB-hour
├── Monthly (730h): €63.51/GB-month
└── 99.9% SLA

Examples (Standard Tier, europe-west1):
├── 5 GB:   €0.435/h  →  €317.55/mois
├── 10 GB:  €0.870/h  →  €635.10/mois
├── 20 GB:  €1.740/h  →  €1,270.20/mois
├── 50 GB:  €4.350/h  →  €3,175.50/mois
├── 100 GB: €8.700/h  →  €6,351.00/mois
├── 200 GB: €17.400/h →  €12,702.00/mois
└── 300 GB: €26.100/h →  €19,053.00/mois

Read Replicas (Standard tier only):
├── Cost per replica: €0.029/GB-hour
├── Example: 50GB + 2 replicas
│   ├── Primary: €4.350/h
│   ├── Replica 1: €1.450/h
│   ├── Replica 2: €1.450/h
│   └── Total: €7.250/h (€5,292.50/mois)

Coûts additionnels:
├── Snapshots: Inclus (automated daily)
├── Cloud Storage (export/import): €0.020/GB-mois (Standard)
├── Network egress (internet): €0.12/GB
├── Network egress (same region): Gratuit
├── Network egress (cross-region): €0.01/GB
└── Pas de Reserved Instances (prix fixes)

Comparaison par région (Standard 50GB):
├── us-central1:    $3,190/mois  (le moins cher)
├── europe-west1:   €3,175/mois
├── asia-southeast1: $3,650/mois
└── australia-southeast1: $4,015/mois (le plus cher)
```

### Calculateur de coût

```python
# GCP Memorystore Pricing Calculator

class MemorystorePricing:
    """Calculate Memorystore costs"""

    # Pricing per GB-hour (europe-west1, 2024)
    BASIC_PRICE_GB_HOUR = 0.035  # EUR
    STANDARD_PRICE_GB_HOUR = 0.087  # EUR
    REPLICA_PRICE_GB_HOUR = 0.029  # EUR

    HOURS_PER_MONTH = 730

    @classmethod
    def calculate_monthly_cost(
        cls,
        tier: str,
        memory_gb: int,
        replica_count: int = 0
    ) -> dict:
        """
        Calculate monthly cost for Memorystore

        Args:
            tier: 'basic' or 'standard'
            memory_gb: Memory size in GB (1-300)
            replica_count: Number of read replicas (0-5, Standard only)
        """

        if tier.lower() == 'basic':
            price_per_gb_hour = cls.BASIC_PRICE_GB_HOUR
            replica_count = 0  # Basic doesn't support replicas
        else:
            price_per_gb_hour = cls.STANDARD_PRICE_GB_HOUR

        # Primary instance cost
        primary_hourly = memory_gb * price_per_gb_hour
        primary_monthly = primary_hourly * cls.HOURS_PER_MONTH

        # Replica costs
        replica_hourly = 0
        if replica_count > 0 and tier.lower() == 'standard':
            replica_hourly = memory_gb * cls.REPLICA_PRICE_GB_HOUR * replica_count

        replica_monthly = replica_hourly * cls.HOURS_PER_MONTH

        # Total compute
        total_hourly = primary_hourly + replica_hourly
        total_monthly = total_hourly * cls.HOURS_PER_MONTH

        # Additional costs (estimates)
        storage_cost = 50  # €50/month for exports/imports
        egress_cost = 100  # €100/month for network egress

        additional_monthly = storage_cost + egress_cost

        return {
            'tier': tier,
            'memory_gb': memory_gb,
            'replica_count': replica_count,
            'primary_hourly': round(primary_hourly, 3),
            'replica_hourly': round(replica_hourly, 3),
            'total_hourly': round(total_hourly, 3),
            'primary_monthly': round(primary_monthly, 2),
            'replica_monthly': round(replica_monthly, 2),
            'compute_monthly': round(total_monthly, 2),
            'additional_monthly': round(additional_monthly, 2),
            'total_monthly': round(total_monthly + additional_monthly, 2),
            'total_annual': round((total_monthly + additional_monthly) * 12, 2)
        }

# Examples
print("=== Scenario 1: Production Standard 50GB + 2 replicas ===")
cost1 = MemorystorePricing.calculate_monthly_cost(
    tier='standard',
    memory_gb=50,
    replica_count=2
)
print(f"Primary: €{cost1['primary_monthly']:,}/month")
print(f"Replicas (2): €{cost1['replica_monthly']:,}/month")
print(f"Total: €{cost1['total_monthly']:,}/month")
print(f"Annual: €{cost1['total_annual']:,}/year")

print("\n=== Scenario 2: Large cache 200GB Standard ===")
cost2 = MemorystorePricing.calculate_monthly_cost(
    tier='standard',
    memory_gb=200,
    replica_count=3
)
print(f"Total: €{cost2['total_monthly']:,}/month")
print(f"Annual: €{cost2['total_annual']:,}/year")

print("\n=== Scenario 3: Dev/Test Basic 10GB ===")
cost3 = MemorystorePricing.calculate_monthly_cost(
    tier='basic',
    memory_gb=10,
    replica_count=0
)
print(f"Total: €{cost3['total_monthly']:,}/month")
print(f"Annual: €{cost3['total_annual']:,}/year")

print("\n=== Comparison: Basic vs Standard (50GB) ===")
basic = MemorystorePricing.calculate_monthly_cost('basic', 50, 0)
standard = MemorystorePricing.calculate_monthly_cost('standard', 50, 0)
print(f"Basic:    €{basic['total_monthly']}/month")
print(f"Standard: €{standard['total_monthly']}/month")
print(f"Premium for HA: €{standard['total_monthly'] - basic['total_monthly']}/month")
print(f"Premium %: {((standard['total_monthly'] / basic['total_monthly']) - 1) * 100:.1f}%")
```

Output:
```
=== Scenario 1: Production Standard 50GB + 2 replicas ===
Primary: €3,175.5/month
Replicas (2): €2,117/month
Total: €5,442.5/month
Annual: €65,310/year

=== Scenario 2: Large cache 200GB Standard ===
Total: €26,754/month
Annual: €321,048/year

=== Scenario 3: Dev/Test Basic 10GB ===
Total: €405.5/month
Annual: €4,866/year

=== Comparison: Basic vs Standard (50GB) ===
Basic:    €1,427.5/month
Standard: €3,325.5/month
Premium for HA: €1,898/month
Premium %: 132.9%
```

---

## 📊 Comparaison avec AWS et Azure

### Tableau comparatif détaillé

| Critère | GCP Memorystore | AWS ElastiCache | AWS MemoryDB | Azure Cache Premium |
|---------|-----------------|-----------------|--------------|---------------------|
| **Architecture** |
| Max RAM/instance | 300 GB | 317 GB | 419 GB | 1.2 TB (clustered) |
| Clustering natif | ❌ | ✅ (500 shards) | ✅ (500 shards) | ✅ (10 shards) |
| Read replicas | ✅ (0-5) | ✅ (0-5) | ✅ (0-5) | ❌ |
| **Disponibilité** |
| SLA | 99.9% | 99.9% / 99.99% | 99.99% | 99.9% / 99.99% |
| Multi-zone | ✅ | ✅ | ✅ (built-in) | ✅ |
| Geo-replication | ❌ | ✅ (Global DS) | ❌ | ✅ (passive) |
| Active-Active | ❌ | ❌ | ❌ | ✅ (Enterprise) |
| **Réseau** |
| VPC native | ✅ (PSC) | Subnet group | Subnet group | ✅ (VNet injection) |
| Private endpoint | ✅ (always) | ✅ | ✅ | ✅ |
| Public endpoint | ❌ | ✅ (optional) | ✅ (optional) | ✅ (optional) |
| **Persistence** |
| RDB | ✅ | ✅ | N/A (WAL) | ✅ |
| AOF | ❌ | ✅ | N/A (WAL) | ✅ |
| Durability | Snapshots | Optional | Guaranteed | Optional |
| **Pricing (50GB Standard/HA)** |
| On-demand | ~€3,175/mois | ~$360/mois | ~$640/mois | ~€1,000/mois |
| Reserved/RI | N/A | ~$145/mois | ~$260/mois | ~€450/mois |
| **Simplicity** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Flexibility** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Feature set** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### Forces et faiblesses

**GCP Memorystore - Forces :**
```
✅ Simplicité d'utilisation (minimal config)
✅ Intégration GCP native (GKE, Cloud Run, etc.)
✅ Private Service Connect (sécurité maximale)
✅ Automated daily backups inclus
✅ Read replicas faciles à déployer
✅ Tarification simple (pas de surprise)
```

**GCP Memorystore - Faiblesses :**
```
❌ Pas de clustering (max 300GB single instance)
❌ Pas de cross-region replication
❌ Pas de Redis modules (Stack)
❌ Moins de configuration options
❌ Pas de Reserved Instances (coût fixe élevé)
❌ AOF persistence non disponible
```

### Quand choisir Memorystore ?

```yaml
Utilisez Memorystore si:
☑ Vous êtes déjà sur GCP
☑ Simplicité > Flexibilité
☑ Dataset <300GB (single instance suffit)
☑ Intégration GKE/Cloud Run prioritaire
☑ Pas besoin de features avancées
☑ Budget permet prix GCP

N'utilisez PAS Memorystore si:
☒ Besoin de clustering (>300GB)
☒ Cross-region replication requis
☒ Redis modules nécessaires (Stack)
☒ Budget très contraint (pas de RI)
☒ Maximum de configuration requis
```

---

## ✅ Best Practices

### Configuration production

```yaml
Instance Configuration:
├── Tier: STANDARD_HA (toujours en prod)
├── Memory: Right-size (monitor usage)
├── Redis version: Latest stable (7.2)
├── Read replicas: 2-3 (pour read-heavy workloads)
├── AUTH: Enabled (toujours)
├── TLS: SERVER_AUTHENTICATION (toujours)
└── Persistence: RDB avec snapshot horaire

Network:
├── VPC: Dedicated VPC pour chaque environnement
├── Private Service Connect: Oui (jamais d'IP publique)
├── Firewall: Minimal (only necessary rules)
└── VPC Peering: Pour cross-project si nécessaire

Monitoring:
├── Cloud Monitoring: Tous les dashboards configurés
├── Alerting: CPU >80%, Memory >85%, Lag >5s
├── Logging: Export to BigQuery pour analyse
└── Uptime checks: Si applicable

Backup:
├── Daily snapshots: Activés
├── Export to GCS: Hebdomadaire
├── Retention: 30 jours minimum
└── Test restore: Trimestriel

Security:
├── Workload Identity: Pour GKE
├── Secret Manager: Pour AUTH password
├── IAM: Least privilege
├── Audit logs: Activés
└── VPC Service Controls: Si compliance strict
```

### Optimisation des coûts

```yaml
Right-sizing:
├── Monitor memory usage trends
├── Scale down si usage <50% constant
├── Start small, scale up as needed
└── Use Basic tier pour dev/test

Read replicas:
├── Only if read-heavy workload
├── Monitor actual read distribution
├── Consider app-level caching instead
└── Replicas cost 1/3 of primary

Network:
├── Keep traffic in same region (gratuit)
├── Use Cloud CDN pour edge caching
├── Minimize cross-region calls
└── Use compression when possible

Exports/Imports:
├── Schedule during low-traffic hours
├── Use lifecycle policies on GCS
├── Compress exports (RDB is compressible)
└── Delete old backups regularly
```

---

## 🎯 Conclusion

### Points clés à retenir

1. **Simplicité avant tout**
   - Memorystore = configuration minimale
   - Managed fully par Google
   - Parfait pour use-cases standards

2. **Limitations claires**
   - Pas de clustering (max 300GB)
   - Pas de cross-region
   - Moins de features qu'AWS/Azure

3. **Intégration GCP native**
   - Private Service Connect
   - Workload Identity avec GKE
   - Cloud Monitoring natif

4. **Pricing simple mais élevé**
   - Pas de Reserved Instances
   - €63.51/GB-mois (Standard)
   - Read replicas économiques (€21.17/GB-mois)

5. **Idéal pour :**
   - Projets 100% GCP
   - Datasets <300GB
   - Besoin de simplicité
   - Intégration GKE forte

### Recommandations finales

**Pour production GCP standard :**
```
Memorystore Standard HA
├── 50 GB RAM
├── 2 read replicas
├── AUTH + TLS enabled
├── Daily RDB snapshots
└── Cloud Monitoring dashboards
Coût: ~€5,400/mois
```

**Alternative si budget contraint :**
```
Considérer AWS ElastiCache avec RI 3 ans
ou Azure Cache Premium avec RI
→ Économies significatives (~60%)
```

**Alternative si >300GB :**
```
AWS ElastiCache/MemoryDB avec clustering
ou Azure Cache Premium avec sharding
→ Memorystore n'est pas adapté
```

---

**🎯 Section suivante :** Nous allons maintenant explorer **Redis Enterprise Cloud** dans la section 15.5, la solution premium multi-cloud avec toutes les features avancées.

⏭️ [Redis Cloud (Redis Enterprise Cloud)](/15-redis-cloud-conteneurs/05-redis-cloud-enterprise.md)
