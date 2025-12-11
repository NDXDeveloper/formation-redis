🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.1 Comparatif des solutions managées

## 🎯 Objectifs

- Comprendre l'offre Redis managée de chaque cloud provider
- Comparer les architectures, SLA, pricing et fonctionnalités
- Identifier les forces et faiblesses de chaque solution
- Choisir la solution optimale selon le contexte (workload, budget, contraintes)
- Maîtriser la configuration Infrastructure as Code pour chaque provider

---

## 📊 Vue d'ensemble du marché

### Les 4 acteurs principaux

```
┌─────────────────────────────────────────────────────────────┐
│                     Redis Managed Services                  │
└─────────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼──────┐ ┌──────▼────┐ ┌────────▼────┐ ┌──────────┐
    │     AWS     │ │   Azure   │ │     GCP     │ │  Redis   │
    │             │ │           │ │             │ │Enterprise│
    │ ElastiCache │ │   Cache   │ │ Memorystore │ │  Cloud   │
    │  MemoryDB   │ │ for Redis │ │             │ │          │
    └─────────────┘ └───────────┘ └─────────────┘ └──────────┘

    Parts de marché (2024)
    AWS: ~45%
    Azure: ~30%
    GCP: ~15%
    Redis Enterprise: ~10%
```

### Évolution historique

```
2011: Redis 2.0 released
      └─> Self-hosted uniquement

2013: AWS ElastiCache for Redis (première offre managée)
      └─> Redis 2.6, pas de cluster

2016: Azure Cache for Redis
      └─> Tiers Premium avec persistence

2017: Google Cloud Memorystore
      └─> Focus sur l'intégration GCP

2021: AWS MemoryDB for Redis
      └─> Durabilité + Redis compatible

2023: Redis Enterprise Cloud (dominance feature)
      └─> Active-Active, Multi-cloud

2024: Toutes les offres supportent Redis 7.2+
      └─> Convergence des features
```

---

## 🔍 Comparaison globale

### Tableau synthétique

| Critère | AWS ElastiCache | AWS MemoryDB | Azure Cache | GCP Memorystore | Redis Enterprise |
|---------|-----------------|--------------|-------------|-----------------|------------------|
| **Version Redis** | 7.1 | 7.0 | 7.2 | 7.2 | 7.2 |
| **SLA** | 99.9% (Multi-AZ) | 99.99% | 99.9% | 99.9% | 99.999% |
| **Max RAM/node** | 317 GB | 419 GB | 1.2 TB | 300 GB | 12 TB |
| **Max nodes/cluster** | 500 | 500 | 10 shards | 1 node | Illimité |
| **Persistence** | RDB/AOF | Durable (log) | RDB/AOF | RDB | RDB/AOF |
| **Multi-AZ** | ✅ | ✅ (built-in) | ✅ | ✅ | ✅ |
| **TLS** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Auth** | AUTH + RBAC | AUTH + RBAC | AUTH + RBAC | AUTH | RBAC avancé |
| **Backup auto** | ✅ | ✅ (continuous) | ✅ | ✅ (daily) | ✅ |
| **Auto-scaling** | ❌ | ❌ | ✅ (Premium) | ❌ | ✅ |
| **Active-Active** | ❌ | ❌ | ❌ (preview) | ❌ | ✅ |
| **Modules Redis** | ❌ | ❌ | ❌ | ❌ | ✅ (Stack) |
| **Pricing** | 💰💰 | 💰💰💰 | 💰💰 | 💰💰 | 💰💰💰💰 |

**Légende SLA :**
- 99.9% = ~8h downtime/an
- 99.99% = ~52min downtime/an
- 99.999% = ~5min downtime/an

---

## ☁️ AWS : Double offre

### Architecture comparative

```
┌────────────────────────────────────────────────────────────┐
│                        AWS VPC                             │
│                                                            │
│  ┌──────────────────────┐     ┌─────────────────────────┐  │
│  │  ElastiCache         │     │     MemoryDB            │  │
│  │  (Cache use-case)    │     │  (Primary DB use-case)  │  │
│  │                      │     │                         │  │
│  │  ┌────────────────┐  │     │  ┌──────────────────┐   │  │
│  │  │Primary Node    │  │     │  │ Primary Node     │   │  │
│  │  │  (AZ-1)        │  │     │  │   + WAL          │   │  │
│  │  └────────┬───────┘  │     │  └────────┬─────────┘   │  │
│  │           │          │     │           │             │  │
│  │  ┌────────▼───────┐  │     │  ┌────────▼─────────┐   │  │
│  │  │Replica (AZ-2)  │  │     │  │Replica + WAL     │   │  │
│  │  └────────────────┘  │     │  │(sync replication)│   │  │
│  │                      │     │  └──────────────────┘   │  │
│  │  • Async repl        │     │  • Sync repl (strong)   │  │
│  │  • RDB/AOF optional  │     │  • Multi-AZ WAL         │  │
│  │  • Failover ~2-3min  │     │  • Failover <1min       │  │
│  └──────────────────────┘     └─────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### AWS ElastiCache for Redis

#### Caractéristiques clés

**✅ Points forts :**
- Maturité (11 ans sur le marché)
- Intégration AWS complète (CloudWatch, IAM, VPC, etc.)
- Large gamme d'instance types (r7g, r6g, r5, m7g...)
- Support du cluster mode (sharding automatique)
- Global Datastore (réplication cross-region, read-only)
- Backup automatique vers S3

**⚠️ Limitations :**
- Pas de durabilité garantie (cache volatil)
- Pas de transactions multi-key en mode cluster
- Pas de modules Redis Stack
- Scaling vertical nécessite un basculement

#### Modes de déploiement

```yaml
# 1. Standalone (dev/test)
Mode: Single Node
├── Pas de réplication
├── Pas de failover
└── Downtime lors des maintenances

# 2. Cluster Mode Disabled (réplication simple)
Mode: Primary + Replicas
├── 1 primary + 0-5 replicas
├── Réplication asynchrone
├── Failover automatique
└── Toutes les clés sur un seul shard

# 3. Cluster Mode Enabled (sharding)
Mode: Multi-shard cluster
├── 1-500 shards
├── Chaque shard = 1 primary + 0-5 replicas
├── Distribution automatique via hash slots
└── Scaling horizontal
```

#### Configuration Terraform

```hcl
# ElastiCache Cluster Mode Enabled (production-ready)
resource "aws_elasticache_replication_group" "redis_cluster" {
  replication_group_id       = "prod-redis-cluster"
  replication_group_description = "Production Redis Cluster"

  # Engine configuration
  engine                     = "redis"
  engine_version            = "7.1"
  port                      = 6379
  parameter_group_name      = aws_elasticache_parameter_group.redis_params.name

  # Node configuration
  node_type                 = "cache.r7g.xlarge"  # 26.32 GB RAM, 4 vCPUs
  num_node_groups          = 3                     # 3 shards
  replicas_per_node_group  = 2                     # 2 replicas per shard

  # Multi-AZ with automatic failover
  automatic_failover_enabled = true
  multi_az_enabled          = true

  # Security
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token_enabled        = true
  auth_token                = random_password.redis_auth.result

  # Subnet and security
  subnet_group_name         = aws_elasticache_subnet_group.redis_subnet.name
  security_group_ids        = [aws_security_group.redis_sg.id]

  # Maintenance and backup
  maintenance_window        = "sun:05:00-sun:07:00"
  snapshot_retention_limit  = 7
  snapshot_window          = "03:00-05:00"
  final_snapshot_identifier = "prod-redis-final-snapshot"

  # Monitoring
  notification_topic_arn    = aws_sns_topic.redis_alerts.arn

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  # Log delivery configuration
  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis_slow_log.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "slow-log"
  }

  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis_engine_log.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "engine-log"
  }

  tags = {
    Environment = "production"
    Team        = "platform"
    ManagedBy   = "terraform"
  }
}

# Parameter group for optimization
resource "aws_elasticache_parameter_group" "redis_params" {
  name   = "prod-redis-params"
  family = "redis7"

  # Memory management
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  # Timeout for idle connections
  parameter {
    name  = "timeout"
    value = "300"
  }

  # TCP keepalive
  parameter {
    name  = "tcp-keepalive"
    value = "300"
  }

  # Slow log
  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"  # 10ms
  }

  parameter {
    name  = "slowlog-max-len"
    value = "128"
  }

  # Notify keyspace events
  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"  # Expired and Evicted events
  }
}

# Subnet group (Multi-AZ)
resource "aws_elasticache_subnet_group" "redis_subnet" {
  name       = "prod-redis-subnet-group"
  subnet_ids = [
    aws_subnet.private_a.id,
    aws_subnet.private_b.id,
    aws_subnet.private_c.id,
  ]

  tags = {
    Name = "Redis subnet group"
  }
}

# Security group
resource "aws_security_group" "redis_sg" {
  name        = "prod-redis-sg"
  description = "Security group for Redis cluster"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app_sg.id]  # Only from app tier
    description     = "Redis from application tier"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name = "prod-redis-sg"
  }
}

# CloudWatch alarms
resource "aws_cloudwatch_metric_alarm" "redis_cpu" {
  alarm_name          = "redis-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "EngineCPUUtilization"
  namespace           = "AWS/ElastiCache"
  period              = "300"
  statistic           = "Average"
  threshold           = "75"
  alarm_description   = "Redis CPU utilization too high"
  alarm_actions       = [aws_sns_topic.redis_alerts.arn]

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.redis_cluster.id
  }
}

resource "aws_cloudwatch_metric_alarm" "redis_memory" {
  alarm_name          = "redis-high-memory"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "DatabaseMemoryUsagePercentage"
  namespace           = "AWS/ElastiCache"
  period              = "300"
  statistic           = "Average"
  threshold           = "85"
  alarm_description   = "Redis memory usage too high"
  alarm_actions       = [aws_sns_topic.redis_alerts.arn]

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.redis_cluster.id
  }
}

# Outputs
output "redis_configuration_endpoint" {
  description = "Redis configuration endpoint (cluster mode)"
  value       = aws_elasticache_replication_group.redis_cluster.configuration_endpoint_address
}

output "redis_connection_string" {
  description = "Redis connection string"
  value       = "rediss://${aws_elasticache_replication_group.redis_cluster.configuration_endpoint_address}:6379"
  sensitive   = true
}
```

#### Pricing ElastiCache (us-east-1, 2024)

```yaml
Instance Types (on-demand, par heure)
├── cache.t4g.micro:    $0.016/h  (0.5 GB RAM)
├── cache.t4g.small:    $0.032/h  (1.37 GB RAM)
├── cache.t4g.medium:   $0.064/h  (3.09 GB RAM)
├── cache.r7g.large:    $0.226/h  (13.07 GB RAM)
├── cache.r7g.xlarge:   $0.453/h  (26.32 GB RAM)
├── cache.r7g.2xlarge:  $0.906/h  (52.82 GB RAM)
├── cache.r7g.4xlarge:  $1.812/h  (105.81 GB RAM)
├── cache.r7g.8xlarge:  $3.624/h  (211.80 GB RAM)
├── cache.r7g.12xlarge: $5.436/h  (317.77 GB RAM)
└── cache.r7g.16xlarge: $7.248/h  (423.65 GB RAM)

Reserved Instances (1 year, all upfront)
└── Économie: ~40% vs on-demand

Reserved Instances (3 years, all upfront)
└── Économie: ~60% vs on-demand

Coûts additionnels
├── Backup storage: $0.085/GB-mois (au-delà de la taille du cluster)
├── Data transfer (out): $0.09/GB (vers Internet)
└── Data transfer (in): Gratuit

Exemple cluster production (3 shards, 2 replicas, r7g.xlarge)
├── Nodes: 3 shards × (1 primary + 2 replicas) = 9 nodes
├── Coût horaire: 9 × $0.453 = $4.077/h
├── Coût mensuel: ~$2,976/mois (on-demand)
└── Coût mensuel: ~$1,190/mois (RI 3 ans)
```

---

### AWS MemoryDB for Redis

#### Différence fondamentale avec ElastiCache

```
┌─────────────────────────────────────────────────────────────┐
│                    ElastiCache vs MemoryDB                   │
└─────────────────────────────────────────────────────────────┘

ElastiCache (Cache tier)
├── Design: Cache volatile
├── Durabilité: Best-effort (RDB/AOF optional)
├── Réplication: Asynchrone
├── Failover: ~2-3 minutes
├── Use case: Cache applicatif
└── RTO/RPO: Minutes / Possible data loss

MemoryDB (Primary database)
├── Design: Base de données durable
├── Durabilité: Multi-AZ transaction log (like Aurora)
├── Réplication: Synchrone vers log + async vers replicas
├── Failover: <1 minute
├── Use case: Primary DB, session store, leaderboards
└── RTO/RPO: Seconds / Zero data loss
```

#### Architecture MemoryDB

```
┌────────────────────────────────────────────────────────────┐
│                    AWS MemoryDB Architecture               │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Primary Node (AZ-1)                     │  │
│  │                                                      │  │
│  │  ┌─────────────┐                                     │  │
│  │  │ Redis Engine│                                     │  │
│  │  │  (in-memory)│                                     │  │
│  │  └──────┬──────┘                                     │  │
│  │         │                                            │  │
│  │         │ (1) Write                                  │  │
│  │         ▼                                            │  │
│  │  ┌─────────────┐      ┌─────────────┐                │  │
│  │  │ Transaction │─────▶│ Transaction │                │  │
│  │  │ Log (AZ-1)  │      │ Log (AZ-2)  │ (Sync)         │  │
│  │  └─────────────┘      └─────────────┘                │  │
│  │         │                     │                      │  │
│  │         │ (2) ACK after       │                      │  │
│  │         │     multi-AZ write  │                      │  │
│  │         ▼                     ▼                      │  │
│  │  ┌─────────────┐       ┌─────────────┐               │  │
│  │  │   Replica   │       │   Replica   │               │  │
│  │  │   (AZ-2)    │       │   (AZ-3)    │ (Async)       │  │
│  │  └─────────────┘       └─────────────┘               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  • Strong consistency: Writes committed to multi-AZ log    │
│  • Zero data loss: Transaction log is durable              │
│  • Fast recovery: Replicas can be promoted instantly       │
└────────────────────────────────────────────────────────────┘
```

#### Configuration Terraform MemoryDB

```hcl
# MemoryDB Cluster (production-ready)
resource "aws_memorydb_cluster" "redis_primary" {
  name                   = "prod-memorydb-cluster"
  description           = "Production MemoryDB for primary data store"

  # Node configuration
  node_type             = "db.r7g.xlarge"  # 26.32 GB RAM
  num_shards            = 3
  num_replicas_per_shard = 2

  # ACL for authentication
  acl_name              = aws_memorydb_acl.redis_acl.name

  # Subnet and security
  subnet_group_name     = aws_memorydb_subnet_group.redis_subnet.name
  security_group_ids    = [aws_security_group.memorydb_sg.id]

  # Engine
  engine_version        = "7.0"
  port                  = 6379
  parameter_group_name  = aws_memorydb_parameter_group.redis_params.name

  # TLS
  tls_enabled           = true

  # Maintenance
  maintenance_window    = "sun:05:00-sun:07:00"

  # Snapshot configuration
  snapshot_retention_limit = 7
  snapshot_window         = "03:00-05:00"
  final_snapshot_name     = "prod-memorydb-final"

  # KMS encryption
  kms_key_arn           = aws_kms_key.memorydb.arn

  # Automatic snapshots to S3
  snapshot_arns = []

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  tags = {
    Environment = "production"
    Purpose     = "primary-datastore"
  }
}

# ACL (Access Control List) - fine-grained permissions
resource "aws_memorydb_acl" "redis_acl" {
  name = "prod-redis-acl"

  user_names = [
    aws_memorydb_user.admin.name,
    aws_memorydb_user.app_readwrite.name,
    aws_memorydb_user.app_readonly.name,
  ]

  tags = {
    Name = "Production Redis ACL"
  }
}

# Admin user (full access)
resource "aws_memorydb_user" "admin" {
  user_name     = "admin"
  access_string = "on ~* &* +@all"  # All keys, all channels, all commands

  authentication_mode {
    type      = "password"
    passwords = [random_password.admin_password.result]
  }

  tags = {
    Role = "admin"
  }
}

# Application user (read-write, no dangerous commands)
resource "aws_memorydb_user" "app_readwrite" {
  user_name     = "app-readwrite"
  access_string = "on ~* &* +@all -@dangerous -@admin"

  authentication_mode {
    type      = "password"
    passwords = [random_password.app_password.result]
  }

  tags = {
    Role = "application"
  }
}

# Read-only user
resource "aws_memorydb_user" "app_readonly" {
  user_name     = "app-readonly"
  access_string = "on ~* &* +@read"

  authentication_mode {
    type      = "password"
    passwords = [random_password.readonly_password.result]
  }

  tags = {
    Role = "readonly"
  }
}

# Parameter group
resource "aws_memorydb_parameter_group" "redis_params" {
  name   = "prod-memorydb-params"
  family = "memorydb_redis7"

  parameter {
    name  = "maxmemory-policy"
    value = "noeviction"  # MemoryDB = primary DB, don't evict
  }

  parameter {
    name  = "activedefrag"
    value = "yes"
  }

  parameter {
    name  = "timeout"
    value = "300"
  }
}

# Subnet group
resource "aws_memorydb_subnet_group" "redis_subnet" {
  name       = "prod-memorydb-subnet"
  subnet_ids = [
    aws_subnet.private_a.id,
    aws_subnet.private_b.id,
    aws_subnet.private_c.id,
  ]
}

# Outputs
output "memorydb_endpoint" {
  value = aws_memorydb_cluster.redis_primary.cluster_endpoint[0].address
}

output "memorydb_port" {
  value = aws_memorydb_cluster.redis_primary.cluster_endpoint[0].port
}
```

#### Pricing MemoryDB (us-east-1, 2024)

```yaml
Instance Types (on-demand, par heure)
├── db.t4g.small:     $0.055/h  (1.37 GB RAM)
├── db.t4g.medium:    $0.111/h  (3.09 GB RAM)
├── db.r7g.large:     $0.403/h  (13.07 GB RAM)
├── db.r7g.xlarge:    $0.806/h  (26.32 GB RAM)
├── db.r7g.2xlarge:   $1.613/h  (52.82 GB RAM)
├── db.r7g.4xlarge:   $3.226/h  (105.81 GB RAM)
├── db.r7g.8xlarge:   $6.452/h  (211.80 GB RAM)
├── db.r7g.12xlarge:  $9.677/h  (317.77 GB RAM)
└── db.r7g.16xlarge:  $12.903/h (419.69 GB RAM)

Note: MemoryDB ~78% plus cher que ElastiCache
      (justifié par la durabilité garantie)

Exemple cluster production (3 shards, 2 replicas, r7g.xlarge)
├── Nodes: 9 nodes
├── Coût horaire: 9 × $0.806 = $7.254/h
├── Coût mensuel: ~$5,295/mois (on-demand)
└── Économie possible avec Savings Plans

Coûts additionnels
├── Snapshot storage: $0.085/GB-mois
├── Data transfer (out): $0.09/GB
└── KMS encryption: Inclus
```

#### Quand utiliser MemoryDB vs ElastiCache ?

```yaml
Utiliser MemoryDB si:
✅ Redis est votre base de données primaire
✅ Zero data loss est critique
✅ Besoin de consistency forte
✅ Session store avec SLA strict
✅ Leaderboards avec données précieuses
✅ Budget permet un surcoût de ~78%

Utiliser ElastiCache si:
✅ Cache applicatif (pas primary DB)
✅ Perte de données acceptable
✅ Coût est une contrainte forte
✅ Cas d'usage: cache de requêtes SQL, API responses
✅ Donnée reconstituable depuis source primaire
```

---

## 🔷 Azure Cache for Redis

### Architecture et tiers

```
┌──────────────────────────────────────────────────────────────┐
│               Azure Cache for Redis - Tiers                  │
└──────────────────────────────────────────────────────────────┘

┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐
│   Basic     │  │  Standard   │  │   Premium   │  │Enterprise │
│             │  │             │  │             │  │           │
│ • 1 node    │  │ • 2 nodes   │  │ • Cluster   │  │• Redis    │
│ • No SLA    │  │ • 99.9% SLA │  │ • 99.9% SLA │  │ Enterprise│
│ • Dev/Test  │  │ • Replica   │  │ • Persist   │  │• 99.99%   │
│ • 250MB-53GB│  │ • 250MB-53GB│  │ • 6GB-1.2TB │  │• 12TB/n   │
│             │  │             │  │ • Geo-repl  │  │• Active-  │
│             │  │             │  │ • VNet      │  │  Active   │
└─────────────┘  └─────────────┘  └─────────────┘  └───────────┘
    Dev              Prod             Prod            Mission-
                    (simple)        (advanced)        Critical
```

### Azure Cache Premium (production standard)

#### Caractéristiques

**✅ Points forts :**
- Intégration Azure complète (Monitoring, RBAC, Private Link)
- Geo-replication (active-passive)
- VNet injection (isolation réseau complète)
- Zone redundancy (availability zones)
- Persistence RDB et AOF
- Import/Export vers Blob Storage
- Scheduled updates (patching contrôlé)

**⚠️ Limitations :**
- Maximum 10 shards par cluster
- Pas de modules Redis Stack
- Scaling vertical nécessite downtime (quelques minutes)
- Active-Active pas disponible (hors tier Enterprise)

#### Configuration ARM Template

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cacheName": {
      "type": "string",
      "defaultValue": "prod-redis-premium"
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "redisCacheCapacity": {
      "type": "int",
      "defaultValue": 3,
      "allowedValues": [1, 2, 3, 4],
      "metadata": {
        "description": "1=6GB, 2=13GB, 3=26GB, 4=53GB"
      }
    },
    "shardCount": {
      "type": "int",
      "defaultValue": 3,
      "minValue": 1,
      "maxValue": 10
    }
  },
  "variables": {
    "redisCacheName": "[parameters('cacheName')]"
  },
  "resources": [
    {
      "type": "Microsoft.Cache/redis",
      "apiVersion": "2023-04-01",
      "name": "[variables('redisCacheName')]",
      "location": "[parameters('location')]",
      "properties": {
        "sku": {
          "name": "Premium",
          "family": "P",
          "capacity": "[parameters('redisCacheCapacity')]"
        },
        "redisConfiguration": {
          "maxmemory-policy": "allkeys-lru",
          "maxmemory-reserved": "50",
          "maxfragmentationmemory-reserved": "50",
          "rdb-backup-enabled": "true",
          "rdb-backup-frequency": "60",
          "rdb-backup-max-snapshot-count": "1",
          "rdb-storage-connection-string": "[concat('DefaultEndpointsProtocol=https;AccountName=', parameters('storageAccountName'), ';AccountKey=', listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName')), '2021-09-01').keys[0].value)]"
        },
        "enableNonSslPort": false,
        "minimumTlsVersion": "1.2",
        "publicNetworkAccess": "Disabled",
        "redisVersion": "6",
        "shardCount": "[parameters('shardCount')]",
        "replicasPerMaster": 1,
        "replicasPerPrimary": 1,
        "zones": [
          "1",
          "2",
          "3"
        ],
        "subnetId": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]"
      ],
      "tags": {
        "Environment": "Production",
        "CostCenter": "Platform"
      }
    },
    {
      "type": "Microsoft.Cache/redis/firewallRules",
      "apiVersion": "2023-04-01",
      "name": "[concat(variables('redisCacheName'), '/AllowAppSubnet')]",
      "dependsOn": [
        "[resourceId('Microsoft.Cache/redis', variables('redisCacheName'))]"
      ],
      "properties": {
        "startIP": "10.0.1.0",
        "endIP": "10.0.1.255"
      }
    },
    {
      "type": "Microsoft.Insights/diagnosticSettings",
      "apiVersion": "2021-05-01-preview",
      "scope": "[format('Microsoft.Cache/redis/{0}', variables('redisCacheName'))]",
      "name": "redis-diagnostics",
      "dependsOn": [
        "[resourceId('Microsoft.Cache/redis', variables('redisCacheName'))]"
      ],
      "properties": {
        "workspaceId": "[parameters('logAnalyticsWorkspaceId')]",
        "logs": [
          {
            "category": "ConnectedClientList",
            "enabled": true,
            "retentionPolicy": {
              "enabled": true,
              "days": 30
            }
          }
        ],
        "metrics": [
          {
            "category": "AllMetrics",
            "enabled": true,
            "retentionPolicy": {
              "enabled": true,
              "days": 30
            }
          }
        ]
      }
    }
  ],
  "outputs": {
    "redisHostName": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.Cache/redis', variables('redisCacheName'))).hostName]"
    },
    "redisSslPort": {
      "type": "int",
      "value": "[reference(resourceId('Microsoft.Cache/redis', variables('redisCacheName'))).sslPort]"
    },
    "redisPrimaryKey": {
      "type": "securestring",
      "value": "[listKeys(resourceId('Microsoft.Cache/redis', variables('redisCacheName')), '2023-04-01').primaryKey]"
    }
  }
}
```

#### Configuration Terraform (Azure)

```hcl
# Azure Cache for Redis Premium
resource "azurerm_redis_cache" "premium" {
  name                = "prod-redis-premium"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  # SKU
  capacity            = 3  # P3 = 26GB
  family              = "P"
  sku_name            = "Premium"

  # Cluster configuration
  shard_count         = 3

  # Zone redundancy
  zones               = ["1", "2", "3"]

  # Network
  subnet_id           = azurerm_subnet.redis.id
  public_network_access_enabled = false

  # TLS
  minimum_tls_version = "1.2"
  enable_non_ssl_port = false

  # Redis configuration
  redis_configuration {
    maxmemory_policy                     = "allkeys-lru"
    maxmemory_reserved                   = 50
    maxfragmentationmemory_reserved      = 50

    # RDB persistence
    rdb_backup_enabled                   = true
    rdb_backup_frequency                 = 60  # minutes
    rdb_backup_max_snapshot_count        = 1
    rdb_storage_connection_string        = azurerm_storage_account.redis_backup.primary_blob_connection_string

    # AOF persistence (Premium only)
    aof_backup_enabled                   = true
    aof_storage_connection_string_0      = azurerm_storage_account.redis_backup.primary_blob_connection_string
    aof_storage_connection_string_1      = azurerm_storage_account.redis_backup.secondary_blob_connection_string

    # Notify keyspace events
    notify_keyspace_events               = "Ex"
  }

  # Patch schedule
  patch_schedule {
    day_of_week    = "Sunday"
    start_hour_utc = 5
  }

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}

# Geo-replication (active-passive)
resource "azurerm_redis_cache" "secondary" {
  name                = "prod-redis-secondary"
  location            = "westeurope"  # Different region
  resource_group_name = azurerm_resource_group.dr.name

  capacity            = 3
  family              = "P"
  sku_name            = "Premium"
  shard_count         = 3

  zones               = ["1", "2", "3"]
  minimum_tls_version = "1.2"

  redis_configuration {
    maxmemory_policy = "allkeys-lru"
  }
}

resource "azurerm_redis_linked_server" "geo_replication" {
  target_redis_cache_name     = azurerm_redis_cache.secondary.name
  resource_group_name         = azurerm_resource_group.main.name
  linked_redis_cache_id       = azurerm_redis_cache.premium.id
  linked_redis_cache_location = azurerm_redis_cache.premium.location
  server_role                 = "Primary"
}

# Private endpoint for private network access
resource "azurerm_private_endpoint" "redis" {
  name                = "redis-private-endpoint"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "redis-privateserviceconnection"
    private_connection_resource_id = azurerm_redis_cache.premium.id
    subresource_names              = ["redisCache"]
    is_manual_connection           = false
  }
}

# Monitoring alerts
resource "azurerm_monitor_metric_alert" "redis_cpu" {
  name                = "redis-high-cpu"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_redis_cache.premium.id]
  description         = "Alert when Redis CPU is high"

  criteria {
    metric_namespace = "Microsoft.Cache/redis"
    metric_name      = "percentProcessorTime"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = azurerm_monitor_action_group.platform.id
  }
}

resource "azurerm_monitor_metric_alert" "redis_memory" {
  name                = "redis-high-memory"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_redis_cache.premium.id]

  criteria {
    metric_namespace = "Microsoft.Cache/redis"
    metric_name      = "usedmemorypercentage"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 85
  }

  action {
    action_group_id = azurerm_monitor_action_group.platform.id
  }
}

# Outputs
output "redis_hostname" {
  value = azurerm_redis_cache.premium.hostname
}

output "redis_ssl_port" {
  value = azurerm_redis_cache.premium.ssl_port
}

output "redis_primary_key" {
  value     = azurerm_redis_cache.premium.primary_access_key
  sensitive = true
}
```

#### Pricing Azure Cache (2024, West Europe)

```yaml
Basic Tier (dev/test only)
├── C0 (250 MB):   €0.016/h
├── C1 (1 GB):     €0.041/h
├── C2 (2.5 GB):   €0.083/h
├── C3 (6 GB):     €0.167/h
├── C4 (13 GB):    €0.333/h
├── C5 (26 GB):    €0.667/h
└── C6 (53 GB):    €1.333/h

Standard Tier (with replication)
├── Même pricing que Basic
└── Mais avec 1 replica (2x le prix)

Premium Tier (production)
├── P1 (6 GB):     €0.342/h  → ~€250/mois
├── P2 (13 GB):    €0.684/h  → ~€500/mois
├── P3 (26 GB):    €1.368/h  → ~€1,000/mois
├── P4 (53 GB):    €2.736/h  → ~€2,000/mois
└── P5 (120 GB):   €6.148/h  → ~€4,500/mois

Cluster Premium (avec sharding)
└── Prix/shard = prix base × nombre de shards

Exemple: P3 avec 3 shards
├── €1.368/h × 3 = €4.104/h
└── ~€3,000/mois

Enterprise Tiers (Redis Enterprise)
├── E10 (12 GB):   €3.186/h  → ~€2,300/mois
├── E20 (25 GB):   €6.373/h  → ~€4,600/mois
├── E50 (50 GB):   €12.75/h  → ~€9,300/mois
└── E100 (100 GB): €25.49/h  → ~€18,600/mois

Coûts additionnels
├── Geo-replication: Prix du cache secondaire
├── Storage (persistence): ~€0.02/GB-mois
├── Bandwidth out: €0.087/GB
└── Reserved Instances: -35% (1 an) / -55% (3 ans)
```

---

## 🔴 Google Cloud Memorystore

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│               Google Cloud Memorystore                     │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               VPC Network                            │  │
│  │                                                      │  │
│  │  ┌─────────────────────────────────────────────┐     │  │
│  │  │  Redis Instance (Zone A)                    │     │  │
│  │  │  ┌────────────┐      ┌────────────┐         │     │  │
│  │  │  │  Primary   │─────▶│  Replica   │         │     │  │
│  │  │  │   Node     │      │   (Zone B) │         │     │  │
│  │  │  └────────────┘      └────────────┘         │     │  │
│  │  │                                             │     │  │
│  │  │  • High availability (auto failover)        │     │  │
│  │  │  • Read replicas (up to 5)                  │     │  │
│  │  │  • Persistence: RDB snapshots               │     │  │
│  │  └─────────────────────────────────────────────┘     │  │
│  │                                                      │  │
│  │  Private Service Connection                          │  │
│  │  (VPC Peering)                                       │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Caractéristiques

**✅ Points forts :**
- Intégration GCP native (Cloud Monitoring, IAM, VPC)
- Simplicité d'usage (moins de configuration qu'AWS/Azure)
- High availability automatique
- Read replicas (jusqu'à 5)
- Maintenance window contrôlée
- Export vers GCS (Google Cloud Storage)
- Connexion via Private Service Connect (pas d'IP publique)

**⚠️ Limitations :**
- Pas de sharding/clustering (single instance jusqu'à 300 GB)
- Pas de cross-region replication
- Pas de modules Redis Stack
- Moins de features avancées qu'AWS/Azure

### Configuration Terraform (GCP)

```hcl
# Google Cloud Memorystore for Redis
resource "google_redis_instance" "production" {
  name           = "prod-redis-instance"
  tier           = "STANDARD_HA"  # BASIC or STANDARD_HA
  memory_size_gb = 50
  region         = "europe-west1"

  # Redis version
  redis_version = "REDIS_7_2"

  # Display name
  display_name = "Production Redis Instance"

  # Network
  authorized_network      = google_compute_network.main.id
  connect_mode            = "PRIVATE_SERVICE_ACCESS"
  reserved_ip_range       = "10.0.2.0/29"  # /29 provides 8 IPs

  # Location (zone)
  location_id             = "europe-west1-b"
  alternative_location_id = "europe-west1-c"  # For HA

  # Redis configuration
  redis_configs = {
    maxmemory-policy            = "allkeys-lru"
    notify-keyspace-events      = "Ex"
    timeout                     = "300"
    activedefrag                = "yes"
    lfu-log-factor              = "10"
    lfu-decay-time              = "1"
  }

  # Maintenance policy
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

  # Persistence configuration
  persistence_config {
    persistence_mode    = "RDB"
    rdb_snapshot_period = "ONE_HOUR"  # TWELVE_HOURS, ONE_HOUR, SIX_HOURS
    rdb_snapshot_start_time = "2024-01-01T03:00:00Z"
  }

  # Auth
  auth_enabled = true

  # TLS
  transit_encryption_mode = "SERVER_AUTHENTICATION"

  # Replica count (read replicas)
  replica_count = 2
  read_replicas_mode = "READ_REPLICAS_ENABLED"

  labels = {
    environment = "production"
    team        = "platform"
    managed_by  = "terraform"
  }
}

# VPC Network
resource "google_compute_network" "main" {
  name                    = "prod-vpc"
  auto_create_subnetworks = false
}

# Subnet for applications
resource "google_compute_subnetwork" "apps" {
  name          = "apps-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = "europe-west1"
  network       = google_compute_network.main.id

  private_ip_google_access = true
}

# Private Service Connection for Memorystore
resource "google_compute_global_address" "redis_private_ip" {
  name          = "redis-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.main.id
}

resource "google_service_networking_connection" "redis_private_vpc" {
  network                 = google_compute_network.main.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.redis_private_ip.name]
}

# Firewall rule (allow Redis from app subnet)
resource "google_compute_firewall" "allow_redis" {
  name    = "allow-redis-from-apps"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["6379"]
  }

  source_ranges = ["10.0.1.0/24"]  # App subnet
  target_tags   = ["redis"]
}

# Cloud Monitoring alert policies
resource "google_monitoring_alert_policy" "redis_cpu" {
  display_name = "Redis High CPU"
  combiner     = "OR"

  conditions {
    display_name = "Redis CPU > 80%"

    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND resource.labels.instance_id = \"${google_redis_instance.production.id}\" AND metric.type = \"redis.googleapis.com/stats/cpu_utilization\""
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
    auto_close = "1800s"
  }
}

resource "google_monitoring_alert_policy" "redis_memory" {
  display_name = "Redis High Memory"
  combiner     = "OR"

  conditions {
    display_name = "Redis Memory > 85%"

    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND resource.labels.instance_id = \"${google_redis_instance.production.id}\" AND metric.type = \"redis.googleapis.com/stats/memory/usage_ratio\""
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
}

# Notification channel
resource "google_monitoring_notification_channel" "email" {
  display_name = "Platform Team Email"
  type         = "email"

  labels = {
    email_address = "platform-team@example.com"
  }
}

# Outputs
output "redis_host" {
  description = "Redis instance host"
  value       = google_redis_instance.production.host
}

output "redis_port" {
  description = "Redis instance port"
  value       = google_redis_instance.production.port
}

output "redis_connection_string" {
  description = "Redis connection string"
  value       = "redis://${google_redis_instance.production.host}:${google_redis_instance.production.port}"
  sensitive   = true
}

output "redis_auth_string" {
  description = "Redis AUTH string"
  value       = google_redis_instance.production.auth_string
  sensitive   = true
}
```

### Pricing Memorystore (europe-west1, 2024)

```yaml
Basic Tier (single node, no HA)
├── M1 (1 GB):    $0.049/h   → ~$36/mois
├── M2 (2.5 GB):  $0.123/h   → ~$90/mois
├── M3 (5 GB):    $0.245/h   → ~$179/mois
├── M4 (10 GB):   $0.490/h   → $358/mois
└── M5 (20 GB):   $0.980/h   → ~$715/mois

Standard Tier (HA with automatic failover)
├── M1 (5 GB):    $0.145/h   → ~$106/mois
├── M2 (10 GB):   $0.290/h   → ~$212/mois
├── M3 (20 GB):   $0.580/h   → ~$423/mois
├── M4 (50 GB):   $1.450/h   → ~$1,058/mois
├── M5 (100 GB):  $2.900/h   → ~$2,117/mois
└── M6 (300 GB):  $8.700/h   → ~$6,351/mois

Read Replicas (Standard tier only)
└── +$0.029/GB-h par replica

Exemple: Standard 50GB + 2 read replicas
├── Instance principale: $1.450/h
├── Replica 1: $1.450/h
├── Replica 2: $1.450/h
└── Total: $4.350/h (~$3,175/mois)

Coûts additionnels
├── Network egress (internet): $0.12/GB
├── Network egress (same region): Gratuit
├── Snapshots: Inclus (automated daily)
└── Pas de Reserved Instances (prix fixes)
```

---

## 🏢 Redis Enterprise Cloud

### Positionnement

Redis Enterprise Cloud est la solution **premium** directement de Redis Ltd (créateurs de Redis) offrant :
- Les features les plus avancées
- Support commercial direct de Redis
- Portabilité multi-cloud (AWS, Azure, GCP)
- Active-Active geo-distribution avec CRDTs
- Tous les modules Redis Stack inclus

```
┌─────────────────────────────────────────────────────────────────┐
│              Redis Enterprise Cloud Architecture                │
│                                                                 │
│  ┌────────────────────┐  ┌────────────────────┐  ┌───────────┐  │
│  │   AWS Region       │  │   Azure Region     │  │GCP Region │  │
│  │                    │  │                    │  │           │  │
│  │ ┌───────────────┐  │  │ ┌───────────────┐  │  │┌─────────┐│  │
│  │ │Active Database│◄─┼──┼─┤Active Database│◄─┼──┤│Active DB││  │
│  │ │               │  │  │ │               │  │  ││         ││  │
│  │ │ • Read/Write  │  │  │ │ • Read/Write  │  │  ││R/W      ││  │
│  │ │ • CRDT merge  │  │  │ │ • CRDT merge  │  │  ││CRDT     ││  │
│  │ └───────────────┘  │  │ └───────────────┘  │  │└─────────┘│  │
│  └────────────────────┘  └────────────────────┘  └───────────┘  │
│                                                                 │
│  • Conflict-free Replicated Data Types (CRDTs)                  │
│  • Bi-directional replication                                   │
│  • Last-Write-Wins + custom resolution                          │
│  • Local read/write latency (<1ms)                              │
└─────────────────────────────────────────────────────────────────┘
```

### Tiers et fonctionnalités

```yaml
Essentials (basic)
├── Single availability zone
├── Daily backup
├── Standard support
├── Pas de Active-Active
└── Modules: RediSearch, RedisJSON, RedisBloom

Professional (production standard)
├── Multi-AZ
├── Continuous backup
├── Enhanced support (24/7)
├── Active-Active: ❌
└── Tous les modules Redis Stack

Enterprise (mission-critical)
├── Multi-AZ + Multi-Region
├── Active-Active: ✅
├── Premium support (SLA 99.999%)
├── Custom replication topology
├── Dedicated account manager
└── Tous les modules + optimisations propriétaires
```

### Avantages uniques

**Active-Active Geo-Distribution :**
```
┌──────────────────────────────────────────────────────┐
│        Active-Active avec CRDTs                      │
│                                                      │
│  Region US-East           Region EU-West             │
│  ┌──────────┐            ┌──────────┐                │
│  │ Database │◄──────────▶│ Database │                │
│  │          │            │          │                │
│  │ Write:   │            │ Write:   │                │
│  │ SET k v1 │            │ SET k v2 │                │
│  │ (t=100)  │            │ (t=101)  │                │
│  └────┬─────┘            └─────┬────┘                │
│       │                        │                     │
│       │  Replication bidirectionnelle                │
│       └────────────────────────┘                     │
│                                                      │
│  Résolution CRDT automatique:                        │
│  • Counter: Addition (v1 + v2)                       │
│  • String: Last-Write-Wins (t=101 gagne)             │
│  • Set: Union (v1 ∪ v2)                              │
│  • Sorted Set: Max score wins                        │
└──────────────────────────────────────────────────────┘
```

**Redis Stack complet :**
- ✅ RediSearch (full-text, vector search)
- ✅ RedisJSON (documents JSON natifs)
- ✅ RedisTimeSeries (time-series data)
- ✅ RedisBloom (filtres probabilistes)
- ✅ RedisGraph (graph database) - déprécié en 2024
- ✅ RedisGears (programmation réactive) - déprécié

### Configuration via Terraform

```hcl
# Redis Enterprise Cloud via Terraform
# Provider: redislabs/rediscloud
terraform {
  required_providers {
    rediscloud = {
      source  = "RedisLabs/rediscloud"
      version = "~> 1.3"
    }
  }
}

provider "rediscloud" {
  api_key    = var.redis_cloud_api_key
  secret_key = var.redis_cloud_secret_key
}

# Subscription (managed infrastructure)
resource "rediscloud_subscription" "production" {
  name           = "production-subscription"
  payment_method = "credit-card"

  cloud_provider {
    provider         = "AWS"
    region {
      region                       = "us-east-1"
      multiple_availability_zones  = true
      preferred_availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
    }
  }

  # Subscription sizing
  memory_storage = "ram"  # or "ram-and-flash" for tiering

  # Throughput
  throughput_measurement_by    = "operations-per-second"
  throughput_measurement_value = 25000
}

# Active-Active database (multi-region)
resource "rediscloud_active_active_subscription" "global" {
  name           = "global-active-active"
  payment_method = "credit-card"

  # Region 1: US-East
  cloud_provider {
    provider = "AWS"
    region {
      region                       = "us-east-1"
      multiple_availability_zones  = true
      networking_deployment_cidr   = "10.0.0.0/24"
    }
  }

  # Region 2: EU-West
  cloud_provider {
    provider = "AWS"
    region {
      region                       = "eu-west-1"
      multiple_availability_zones  = true
      networking_deployment_cidr   = "10.1.0.0/24"
    }
  }

  # Region 3: Asia-Pacific
  cloud_provider {
    provider = "AWS"
    region {
      region                       = "ap-southeast-1"
      multiple_availability_zones  = true
      networking_deployment_cidr   = "10.2.0.0/24"
    }
  }
}

# Database within subscription
resource "rediscloud_subscription_database" "main" {
  subscription_id              = rediscloud_subscription.production.id
  name                         = "production-db"
  protocol                     = "redis"
  memory_limit_in_gb          = 50
  data_persistence            = "aof-every-1-second"
  replication                 = true
  throughput_measurement_by   = "operations-per-second"
  throughput_measurement_value = 25000

  # Modules
  modules = [
    {
      name = "RedisJSON"
    },
    {
      name = "RediSearch"
    },
    {
      name = "RedisBloom"
    },
    {
      name = "RedisTimeSeries"
    }
  ]

  # High availability
  replication               = true
  replica_of               = []
  periodic_backup_path     = "s3://my-bucket/redis-backups"

  # Alerts
  alert {
    name  = "dataset-size"
    value = 40  # Alert at 80% (40/50 GB)
  }

  alert {
    name  = "throughput-higher-than"
    value = 20000
  }

  # Password
  password = random_password.redis_password.result

  # Client SSL certificate
  client_ssl_certificate = file("${path.module}/certs/client.crt")
}

# Active-Active database instance
resource "rediscloud_active_active_subscription_database" "global_db" {
  subscription_id = rediscloud_active_active_subscription.global.id
  name            = "global-database"
  memory_limit_in_gb = 10

  # Global configuration
  global_data_persistence      = "aof-every-1-second"
  global_password             = random_password.global_redis_password.result
  global_source_ips           = ["0.0.0.0/0"]
  global_alert {
    name  = "dataset-size"
    value = 8  # 80% of 10GB
  }

  # Override per region if needed
  override_region {
    name                  = "us-east-1"
    override_global_password = false
  }

  override_region {
    name                  = "eu-west-1"
    override_global_password = false
  }
}

# Outputs
output "redis_endpoint" {
  value = rediscloud_subscription_database.main.public_endpoint
}

output "redis_port" {
  value = rediscloud_subscription_database.main.public_port
}

output "redis_password" {
  value     = rediscloud_subscription_database.main.password
  sensitive = true
}
```

### Pricing Redis Enterprise Cloud (2024)

```yaml
Modèle de pricing (pay-as-you-go)
├── RAM: $0.119/GB-h (~$87/GB-mois)
├── RAM+Flash (tiering): $0.048/GB-h RAM + $0.012/GB-h Flash
├── Throughput: $0.016/1000 ops/sec-h
├── Active-Active: +100% surcharge sur RAM
└── Modules: Inclus

Exemples:

1. Simple cache (50GB RAM, 10K ops/sec)
├── RAM: 50 × $87 = $4,350/mois
├── Throughput: 10 × $0.016 × 730 = $117/mois
└── Total: ~$4,467/mois

2. Active-Active global (3 regions, 50GB each, 25K ops/sec)
├── RAM: 50 × 3 × $87 × 2 (Active-Active) = $26,100/mois
├── Throughput: 25 × $0.016 × 730 × 3 = $876/mois
└── Total: ~$26,976/mois

3. Tiering (200GB dataset, 50GB RAM + 150GB Flash)
├── RAM: 50 × $35 = $1,750/mois
├── Flash: 150 × $9 = $1,350/mois
├── Throughput: 10K ops/sec = $117/mois
└── Total: ~$3,217/mois
   (vs $17,400 pour 200GB full RAM)

Entreprise (tarification personnalisée)
├── Volume discounts (>500GB)
├── Committed use discounts
├── Premium support inclus
└── Architecture review & consulting
```

---

## 📊 Synthèse comparative

### Par critère technique

```yaml
Durabilité maximale:
1. AWS MemoryDB (transaction log multi-AZ)
2. Redis Enterprise Cloud (AOF + backups continus)
3. Azure Cache Premium (RDB + AOF)
4. GCP Memorystore (RDB snapshots)
5. AWS ElastiCache (cache volatil)

Fonctionnalités avancées:
1. Redis Enterprise Cloud (Active-Active, Stack, CRDTs)
2. AWS ElastiCache (Global Datastore, large scale)
3. Azure Cache Enterprise (Redis Stack)
4. Azure Cache Premium (geo-replication)
5. GCP Memorystore (simplicité)

Scalabilité:
1. Redis Enterprise Cloud (illimité)
2. AWS ElastiCache/MemoryDB (500 shards)
3. Azure Cache Premium (10 shards)
4. GCP Memorystore (300GB single instance)

SLA:
1. Redis Enterprise Cloud (99.999% = 5min/an)
2. AWS MemoryDB (99.99% = 52min/an)
3. AWS ElastiCache Multi-AZ (99.9% = 8h/an)
4. Azure Cache Premium (99.9%)
5. GCP Memorystore Standard (99.9%)

Rapport qualité/prix:
1. GCP Memorystore (simple, efficace)
2. AWS ElastiCache (mature, économique)
3. Azure Cache Premium (bon équilibre)
4. AWS MemoryDB (cher mais durable)
5. Redis Enterprise Cloud (premium pricing)
```

### Matrice de décision

| Use Case | Recommandation | Justification |
|----------|----------------|---------------|
| **Cache applicatif simple** | AWS ElastiCache / Azure Cache Standard | Coût optimal, features suffisantes |
| **Session store critique** | AWS MemoryDB / Redis Enterprise | Zero data loss requis |
| **Leaderboards/Gaming** | AWS ElastiCache / GCP Memorystore | Latence faible, coût raisonnable |
| **E-commerce global** | Redis Enterprise Cloud (Active-Active) | Multi-région, low latency partout |
| **Search avec RediSearch** | Redis Enterprise Cloud | Seul à offrir les modules Stack |
| **IoT/TimeSeries** | Redis Enterprise Cloud | RedisTimeSeries natif |
| **Startup MVP** | GCP Memorystore / Azure Cache Basic | Simple, rapide à déployer |
| **Migration depuis on-prem** | AWS MemoryDB / Redis Enterprise | Compatibilité maximale |
| **Environnement multi-cloud** | Redis Enterprise Cloud | Portable, abstraction du cloud |
| **Budget très contraint** | AWS ElastiCache (RI 3 ans) | Meilleurs discounts long-terme |

---

## 🎯 Guide de sélection

### Arbre de décision

```
Avez-vous besoin de Redis Stack (Search, JSON, etc.) ?
├─ OUI → Redis Enterprise Cloud (seule option)
└─ NON → Continuez ↓

Avez-vous besoin d'Active-Active multi-région ?
├─ OUI → Redis Enterprise Cloud
└─ NON → Continuez ↓

Redis est-il votre base de données primaire ?
├─ OUI → AWS MemoryDB ou Redis Enterprise Cloud
└─ NON (cache) → Continuez ↓

Quel est votre cloud provider principal ?
├─ AWS → ElastiCache
├─ Azure → Azure Cache Premium
├─ GCP → Memorystore
└─ Multi-cloud → Redis Enterprise Cloud

Quel est votre budget ?
├─ Très contraint → ElastiCache avec RI, Memorystore
├─ Moyen → Azure Cache, ElastiCache standard
├─ Élevé → MemoryDB
└─ Premium → Redis Enterprise Cloud
```

### Checklist de critères

```yaml
Critères techniques:
☐ Version Redis requise (7.x ?)
☐ Modules Redis Stack nécessaires ?
☐ Taille max du dataset
☐ Throughput requis (ops/sec)
☐ Latence P99 acceptable
☐ Sharding nécessaire ?
☐ Cross-region replication ?
☐ Active-Active requis ?

Critères opérationnels:
☐ SLA requis (99.9%, 99.99%, 99.999% ?)
☐ RPO acceptable (0 vs quelques secondes ?)
☐ RTO acceptable (1min vs 5min ?)
☐ Backup automatique requis ?
☐ Point-in-time recovery ?
☐ Encryption at-rest/in-transit ?
☐ Compliance (GDPR, HIPAA, SOC2) ?

Critères organisationnels:
☐ Expertise interne (AWS/Azure/GCP) ?
☐ Multi-cloud strategy ?
☐ Préférence vendor-neutral ?
☐ Support commercial requis ?
☐ Budget total disponible ?
☐ Contraintes de coût (OpEx vs CapEx) ?
```

---

## 💡 Recommandations finales

### Pour la plupart des cas (80%)

**Si vous êtes sur AWS :**
```yaml
Development: ElastiCache (cluster mode disabled, t4g.medium)
Staging: ElastiCache (cluster mode enabled, r7g.large, 2 shards)
Production: ElastiCache (cluster mode enabled, r7g.xlarge, 3 shards, Multi-AZ)
ou MemoryDB si zero data loss critique
```

**Si vous êtes sur Azure :**
```yaml
Development: Azure Cache Basic C2
Staging: Azure Cache Standard P1
Production: Azure Cache Premium P3 (3 shards, Zone redundancy)
```

**Si vous êtes sur GCP :**
```yaml
Development: Memorystore Basic M1
Staging: Memorystore Standard M2
Production: Memorystore Standard M4 (50GB) + 2 read replicas
```

### Pour les cas avancés (20%)

**E-commerce global, fintech, gaming :**
→ **Redis Enterprise Cloud** avec Active-Active

**IoT, analytics temps réel, RAG/AI :**
→ **Redis Enterprise Cloud** pour les modules Stack

**Budget illimité, mission-critical :**
→ **Redis Enterprise Cloud** avec support premium

---

## 📈 Tendances et futur

### 2024-2025

```yaml
Convergence des features:
├── Tous les providers tendent vers Redis 7.2+
├── TLS et auth deviennent standard
├── Multi-AZ est la norme
└── Backup automatique inclus partout

Différenciateurs clés:
├── Active-Active (Redis Enterprise uniquement)
├── Redis Stack modules (Redis Enterprise uniquement)
├── Tiering RAM+Flash (Azure, Redis Enterprise)
└── Durabilité garantie (MemoryDB unique sur AWS)

Évolutions attendues:
├── Auto-scaling enfin disponible (Azure l'a déjà)
├── Serverless Redis (pay-per-request)
├── Meilleure intégration Kubernetes (operators managés)
└── AI/ML features (vector search) chez tous les providers
```

### Ce qui ne change pas

- **Redis reste le standard** pour les données en mémoire
- **Managed services gagnent** face au self-hosted (sauf K8s)
- **Coût du cloud** reste élevé (vs on-premise)
- **Vendor lock-in** est réel (architecture specific)

---

## ✅ Checklist avant de choisir

```yaml
☐ Avez-vous testé avec votre workload réel ?
☐ Avez-vous calculé le TCO sur 3 ans ?
☐ Avez-vous vérifié les quotas/limites ?
☐ Avez-vous testé le failover ?
☐ Avez-vous validé la latence depuis vos régions ?
☐ Avez-vous un plan de disaster recovery ?
☐ Avez-vous configuré le monitoring ?
☐ Avez-vous vérifié la compliance ?
☐ Avez-vous un plan de migration (si changement) ?
☐ Avez-vous formé vos équipes ?
```

---

**🎯 Section suivante :** Nous allons maintenant approfondir chaque solution cloud, en commençant par **AWS ElastiCache vs MemoryDB** dans la section 15.2.

⏭️ [AWS : ElastiCache vs MemoryDB (durabilité)](/15-redis-cloud-conteneurs/02-aws-elasticache-memorydb.md)
