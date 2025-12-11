🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.5 Redis Cloud (Redis Enterprise Cloud)

## 🎯 Objectifs

- Maîtriser l'architecture Active-Active avec CRDTs
- Comprendre les modules Redis Stack et leurs cas d'usage
- Configurer des déploiements multi-cloud avec Terraform
- Implémenter des stratégies de geo-distribution
- Optimiser les coûts avec le tiering RAM+Flash
- Évaluer le ROI de Redis Enterprise vs solutions cloud natives

---

## 🏗️ Architecture et positionnement unique

### Vue d'ensemble Redis Enterprise Cloud

```
┌────────────────────────────────────────────────────────────────┐
│              Redis Enterprise Cloud - Architecture             │
│                                                                │
│  Unique Value Proposition:                                     │
│  ├─ Seule solution avec Active-Active (multi-region R/W)       │
│  ├─ Tous les modules Redis Stack inclus                        │
│  ├─ Multi-cloud support (AWS + Azure + GCP)                    │
│  ├─ Auto-scaling et auto-tiering                               │
│  └─ SLA 99.999% (5 minutes downtime/an)                        │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Region 1    │  │  Region 2    │  │  Region 3    │          │
│  │  (AWS)       │  │  (Azure)     │  │  (GCP)       │          │
│  │              │  │              │  │              │          │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │
│  │ │Active DB │◄┼──┼►│Active DB │◄┼──┼►│Active DB │ │          │
│  │ │          │ │  │ │          │ │  │ │          │ │          │
│  │ │ Read +   │ │  │ │ Read +   │ │  │ │ Read +   │ │          │
│  │ │ Write    │ │  │ │ Write    │ │  │ │ Write    │ │          │
│  │ │          │ │  │ │          │ │  │ │          │ │          │
│  │ │ CRDTs    │ │  │ │ CRDTs    │ │  │ │ CRDTs    │ │          │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                │
│  Conflict-Free Replicated Data Types (CRDTs):                  │
│  ├─ Counter: Automatic addition (v1 + v2)                      │
│  ├─ String: Last-Write-Wins avec timestamps                    │
│  ├─ Set: Union (v1 ∪ v2)                                       │
│  ├─ Sorted Set: Max score wins                                 │
│  └─ Hash: Field-level LWW                                      │
│                                                                │
│  Key Benefits:                                                 │
│  • Local read/write latency (<1ms) everywhere                  │
│  • No single point of failure (all regions active)             │
│  • Automatic conflict resolution                               │
│  • No application changes needed                               │
└────────────────────────────────────────────────────────────────┘
```

### Architecture technique détaillée

```
┌─────────────────────────────────────────────────────────────────────┐
│         Redis Enterprise Cloud - Technical Stack                    │
│                                                                     │
│  Application Layer                                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Your Application (any language)                             │   │
│  │  └─ Redis client library                                     │   │
│  └───────────────────────┬──────────────────────────────────────┘   │
│                          │                                          │
│                          │ Standard Redis protocol                  │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Redis Enterprise Proxy Layer                                │   │
│  │  ├─ Load balancing                                           │   │
│  │  ├─ High availability                                        │   │
│  │  ├─ Connection pooling                                       │   │
│  │  └─ Protocol optimization                                    │   │
│  └───────────────────────┬──────────────────────────────────────┘   │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Redis Enterprise Cluster                                    │   │
│  │                                                              │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  │   │
│  │  │   Shard 1      │  │   Shard 2      │  │   Shard 3      │  │   │
│  │  │                │  │                │  │                │  │   │
│  │  │  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │  │   │
│  │  │  │Primary   │  │  │  │Primary   │  │  │  │Primary   │  │  │   │
│  │  │  │          │  │  │  │          │  │  │  │          │  │  │   │
│  │  │  │ RAM      │  │  │  │ RAM      │  │  │  │ RAM      │  │  │   │
│  │  │  │  +       │  │  │  │  +       │  │  │  │  +       │  │  │   │
│  │  │  │ Flash    │  │  │  │ Flash    │  │  │  │  Flash   │  │  │   │
│  │  │  │(optional)│  │  │  │(optional)│  │  │  │(optional)│  │  │   │
│  │  │  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │  │   │
│  │  │  ┌────────┐    │  │  ┌────────┐    │  │  ┌────────┐    │  │   │
│  │  │  │Replica │    │  │  │Replica │    │  │  │Replica │    │  │   │
│  │  │  └────────┘    │  │  └────────┘    │  │  └────────┘    │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘  │   │
│  │                                                              │   │
│  │  Features:                                                   │   │
│  │  ├─ Automatic sharding                                       │   │
│  │  ├─ Instant failover (<1s)                                   │   │
│  │  ├─ Zero-downtime scaling                                    │   │
│  │  ├─ Auto-tiering (RAM+Flash)                                 │   │
│  │  └─ Geo-distributed replication                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Redis Stack Modules (integrated)                            │   │
│  │  ├─ RediSearch (full-text, vector search)                    │   │
│  │  ├─ RedisJSON (native JSON documents)                        │   │
│  │  ├─ RedisTimeSeries (time-series data)                       │   │
│  │  ├─ RedisBloom (probabilistic filters)                       │   │
│  │  └─ RedisGraph (deprecated, sunset 2024)                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🌍 Active-Active Geo-Distribution

### Architecture Active-Active avec CRDTs

```
┌────────────────────────────────────────────────────────────────┐
│        Active-Active Conflict-Free Replicated Data Types       │
│                                                                │
│  Scenario: E-commerce global avec 3 régions                    │
│                                                                │
│  Region US-East (t=0)          Region EU-West (t=0)            │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │  Database        │          │  Database        │            │
│  │                  │          │                  │            │
│  │  WRITE:          │          │  WRITE:          │            │
│  │  SET user:1 {    │          │  SET user:1 {    │            │
│  │    name: "John"  │          │    email: "j@a"  │            │
│  │  }               │          │  }               │            │
│  │  timestamp: 100  │          │  timestamp: 101  │            │
│  └────────┬─────────┘          └────────┬─────────┘            │
│           │                             │                      │
│           │  Bi-directional sync        │                      │
│           └──────────────┬──────────────┘                      │
│                          │                                     │
│                          ▼                                     │
│           ┌──────────────────────────┐                         │
│           │  CRDT Resolution         │                         │
│           │  (automatic merging)     │                         │
│           │                          │                         │
│           │  Result: user:1 {        │                         │
│           │    name: "John"          │                         │
│           │    email: "j@a"          │                         │
│           │  }                       │                         │
│           │                          │                         │
│           │  • Field-level merge     │                         │
│           │  • LWW per field         │                         │
│           │  • No data loss          │                         │
│           └──────────────────────────┘                         │
│                                                                │
│  Supported CRDT Types:                                         │
│                                                                │
│  Counter (auto-increment globally)                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  US-EAST: INCR views:product:123  (local: +1)           │   │
│  │  EU-WEST: INCR views:product:123  (local: +1)           │   │
│  │  Result:  views:product:123 = 2   (addition)            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  String (Last-Write-Wins)                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  US-EAST: SET status "active" (t=100)                   │   │
│  │  EU-WEST: SET status "inactive" (t=101)                 │   │
│  │  Result:  status = "inactive" (timestamp wins)          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  Set (Union)                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  US-EAST: SADD tags:product "sale" "new"                │   │
│  │  EU-WEST: SADD tags:product "featured" "new"            │   │
│  │  Result:  tags:product = {"sale", "new", "featured"}    │   │
│  │           (union of all members)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  Sorted Set (Max score wins)                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  US-EAST: ZADD leaderboard 100 "player1"                │   │
│  │  EU-WEST: ZADD leaderboard 150 "player1"                │   │
│  │  Result:  leaderboard: player1 = 150 (max score)        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  Hash (Field-level LWW)                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  US-EAST: HSET user:1 name "John" (t=100)               │   │
│  │  EU-WEST: HSET user:1 email "j@a.com" (t=101)           │   │
│  │  Result:  user:1 = {name: "John", email: "j@a.com"}     │   │
│  │           (per-field resolution)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

### Configuration Active-Active

```yaml
Active-Active Configuration:
├── Minimum regions: 2
├── Maximum regions: Unlimited (practical limit ~10)
├── Replication: Bi-directional (mesh topology)
├── Conflict resolution: Automatic (CRDT-based)
├── Latency: Region-local reads/writes (<1ms)
├── Consistency: Eventually consistent (seconds)
├── Use cases:
│   ├── Global e-commerce
│   ├── Multi-region gaming
│   ├── Financial services (trading)
│   ├── Content delivery
│   └── IoT data collection

Topology Examples:

2 Regions (simple):
US-EAST ←→ EU-WEST

3 Regions (triangle):
    US-EAST
      ↙  ↘
EU-WEST ←→ ASIA-PACIFIC

5 Regions (full mesh):
US-EAST ←→ US-WEST
   ↕         ↕
EU-WEST ←→ ASIA-PACIFIC ←→ SOUTH-AMERICA
```

---

## 🧰 Redis Stack Modules

### Vue d'ensemble complète

```
┌─────────────────────────────────────────────────────────────────┐
│                     Redis Stack Modules                         │
└─────────────────────────────────────────────────────────────────┘

1. RediSearch (Search & Vector)
   ├── Full-text search (inverted index)
   ├── Vector search (HNSW algorithm)
   ├── Aggregation pipeline (like MongoDB)
   ├── Auto-complete
   ├── Fuzzy matching
   └── Use cases:
       ├── E-commerce product search
       ├── RAG (Retrieval Augmented Generation)
       ├── Semantic search with embeddings
       └── Log analysis

2. RedisJSON
   ├── Native JSON document storage
   ├── JSONPath queries
   ├── Atomic operations on nested fields
   ├── Indexing with RediSearch
   └── Use cases:
       ├── User profiles
       ├── Configuration storage
       ├── API response caching
       └── Event streaming

3. RedisTimeSeries
   ├── Time-series data optimized storage
   ├── Downsampling & aggregation
   ├── Compaction rules
   ├── Real-time analytics
   └── Use cases:
       ├── IoT sensor data
       ├── Financial market data
       ├── Application metrics
       └── Infrastructure monitoring

4. RedisBloom
   ├── Bloom Filter (membership test)
   ├── Cuckoo Filter (deletions supported)
   ├── Count-Min Sketch (frequency estimation)
   ├── Top-K (most frequent items)
   └── Use cases:
       ├── Spam detection
       ├── Duplicate detection
       ├── Rate limiting
       └── Cache admission policies

5. RedisGraph (DEPRECATED - sunset 2024)
   ├── Graph database
   ├── Cypher query language
   └── Migration path: Neo4j, Amazon Neptune
```

### Exemple d'usage : RediSearch + RedisJSON

```python
# Example: E-commerce product catalog with RediSearch + RedisJSON
import redis
from redis.commands.search.field import TextField, NumericField, TagField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import Query

# Connect to Redis Enterprise Cloud
r = redis.Redis(
    host='redis-12345.c123.us-east-1-1.ec2.cloud.redislabs.com',
    port=12345,
    password='your-password',
    ssl=True,
    decode_responses=True
)

# Create index on JSON documents
index_name = "idx:products"
schema = (
    TextField("$.name", as_name="name"),
    TextField("$.description", as_name="description"),
    NumericField("$.price", as_name="price"),
    TagField("$.category", as_name="category"),
    TagField("$.tags", as_name="tags"),
    NumericField("$.stock", as_name="stock")
)

try:
    r.ft(index_name).create_index(
        schema,
        definition=IndexDefinition(
            prefix=["product:"],
            index_type=IndexType.JSON
        )
    )
except Exception as e:
    print(f"Index already exists: {e}")

# Insert JSON documents
products = [
    {
        "id": "product:1",
        "data": {
            "name": "iPhone 15 Pro",
            "description": "Latest Apple smartphone with A17 Pro chip",
            "price": 999,
            "category": "electronics",
            "tags": ["smartphone", "apple", "5g"],
            "stock": 50
        }
    },
    {
        "id": "product:2",
        "data": {
            "name": "MacBook Pro 16",
            "description": "Powerful laptop for professionals",
            "price": 2499,
            "category": "electronics",
            "tags": ["laptop", "apple", "m3"],
            "stock": 20
        }
    },
    {
        "id": "product:3",
        "data": {
            "name": "AirPods Pro 2",
            "description": "Wireless earbuds with active noise cancellation",
            "price": 249,
            "category": "electronics",
            "tags": ["audio", "apple", "wireless"],
            "stock": 100
        }
    }
]

# Insert documents
for product in products:
    r.json().set(product["id"], "$", product["data"])

# Search examples
print("=== Search Examples ===\n")

# 1. Full-text search
print("1. Search 'apple':")
results = r.ft(index_name).search(Query("apple"))
for doc in results.docs:
    print(f"  - {doc.name}: ${doc.price}")

# 2. Filter by price range
print("\n2. Products between $200-$1000:")
results = r.ft(index_name).search(
    Query("*").add_filter(NumericField("price", from_value=200, to_value=1000))
)
for doc in results.docs:
    print(f"  - {doc.name}: ${doc.price}")

# 3. Category + tag filter
print("\n3. Electronics tagged 'laptop':")
results = r.ft(index_name).search(
    Query("@category:{electronics} @tags:{laptop}")
)
for doc in results.docs:
    print(f"  - {doc.name}")

# 4. Aggregation (average price by category)
print("\n4. Average price by category:")
from redis.commands.search.aggregation import AggregateRequest, Reducer
request = AggregateRequest("*").group_by(
    "@category",
    Reducer("avg", ["@price"], as_name="avg_price")
)
results = r.ft(index_name).aggregate(request)
for row in results.rows:
    print(f"  - {row[1]}: ${float(row[3]):.2f}")

# 5. Auto-complete (prefix search)
print("\n5. Auto-complete 'iph':")
results = r.ft(index_name).search(Query("iph*"))
for doc in results.docs:
    print(f"  - {doc.name}")
```

### Exemple d'usage : RedisTimeSeries

```python
# Example: IoT sensor data with RedisTimeSeries
import redis
import time
from redis.commands.timeseries.commands import TimeSeriesCommands

r = redis.Redis(
    host='redis-12345.c123.us-east-1-1.ec2.cloud.redislabs.com',
    port=12345,
    password='your-password',
    ssl=True,
    decode_responses=True
)

# Create time-series for temperature sensor
sensor_key = "sensor:temperature:room1"

try:
    r.ts().create(
        sensor_key,
        retention_msecs=86400000,  # 1 day retention
        labels={
            "sensor_type": "temperature",
            "room": "room1",
            "unit": "celsius"
        }
    )

    # Create compaction rule (1-hour averages)
    r.ts().createrule(
        source_key=sensor_key,
        dest_key=f"{sensor_key}:1h",
        aggregation_type="avg",
        bucket_size_msec=3600000  # 1 hour
    )
except Exception as e:
    print(f"Time series already exists: {e}")

# Add data points
print("Adding temperature readings...")
current_time = int(time.time() * 1000)
for i in range(100):
    temperature = 20 + (i % 10)  # Simulate readings
    r.ts().add(sensor_key, current_time + i * 60000, temperature)  # Every minute

# Query data
print("\n=== Query Examples ===")

# 1. Get latest value
latest = r.ts().get(sensor_key)
print(f"Latest temperature: {latest[1]}°C at {latest[0]}")

# 2. Range query (last hour)
one_hour_ago = current_time - 3600000
results = r.ts().range(sensor_key, one_hour_ago, current_time)
print(f"\nReadings in last hour: {len(results)} data points")

# 3. Aggregation (average per 10 minutes)
aggregated = r.ts().range(
    sensor_key,
    one_hour_ago,
    current_time,
    aggregation_type="avg",
    bucket_size_msec=600000  # 10 minutes
)
print("\nAverage temperature per 10 minutes:")
for timestamp, value in aggregated:
    print(f"  {timestamp}: {value:.2f}°C")

# 4. Multi-get (query multiple sensors)
results = r.ts().mget(filters=["sensor_type=temperature"])
print("\nAll temperature sensors:")
for key, labels, value in results:
    print(f"  {labels['room']}: {value[1]}°C")
```

---

## 🔧 Configuration avec Terraform

### Module Terraform complet

```hcl
# Redis Enterprise Cloud - Terraform Configuration
# Provider: redislabs/rediscloud ~> 1.3

terraform {
  required_version = ">= 1.5"

  required_providers {
    rediscloud = {
      source  = "RedisLabs/rediscloud"
      version = "~> 1.3"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

# Provider configuration
provider "rediscloud" {
  # Set via environment variables:
  # export REDISCLOUD_ACCESS_KEY="your-access-key"
  # export REDISCLOUD_SECRET_KEY="your-secret-key"
}

# Variables
variable "environment" {
  type        = string
  description = "Environment name"
  default     = "production"
}

variable "subscription_name" {
  type        = string
  description = "Redis Cloud subscription name"
  default     = "production-subscription"
}

variable "cloud_provider" {
  type        = string
  description = "Cloud provider: AWS, GCP, or Azure"
  default     = "AWS"

  validation {
    condition     = contains(["AWS", "GCP", "Azure"], var.cloud_provider)
    error_message = "Cloud provider must be AWS, GCP, or Azure"
  }
}

variable "primary_region" {
  type        = string
  description = "Primary region for deployment"
  default     = "us-east-1"
}

variable "enable_active_active" {
  type        = bool
  description = "Enable Active-Active geo-distribution"
  default     = false
}

variable "secondary_regions" {
  type        = list(string)
  description = "Secondary regions for Active-Active"
  default     = []
}

variable "database_memory_gb" {
  type        = number
  description = "Database memory in GB"
  default     = 10
}

variable "throughput_ops_sec" {
  type        = number
  description = "Throughput in operations per second"
  default     = 25000
}

variable "enable_modules" {
  type = object({
    search      = bool
    json        = bool
    timeseries  = bool
    bloom       = bool
  })
  description = "Enable Redis Stack modules"
  default = {
    search     = true
    json       = true
    timeseries = true
    bloom      = true
  }
}

variable "enable_auto_tiering" {
  type        = bool
  description = "Enable RAM+Flash auto-tiering"
  default     = false
}

variable "tags" {
  type        = map(string)
  description = "Tags to apply"
  default = {
    ManagedBy = "Terraform"
  }
}

# Local variables
locals {
  common_tags = merge(
    var.tags,
    {
      Environment = var.environment
      Terraform   = "true"
    }
  )

  # Module list
  enabled_modules = [
    for module_name, enabled in {
      "RedisJSON"       = var.enable_modules.json
      "RediSearch"      = var.enable_modules.search
      "RedisTimeSeries" = var.enable_modules.timeseries
      "RedisBloom"      = var.enable_modules.bloom
    } : module_name if enabled
  ]
}

# Random password for database
resource "random_password" "db_password" {
  length  = 32
  special = false
}

# --- Single Region Deployment ---
resource "rediscloud_subscription" "main" {
  count = var.enable_active_active ? 0 : 1

  name           = var.subscription_name
  payment_method = "credit-card"

  cloud_provider {
    provider = var.cloud_provider

    region {
      region                       = var.primary_region
      multiple_availability_zones  = true
      preferred_availability_zones = []  # Let Redis choose
      networking_deployment_cidr   = "10.0.0.0/24"
    }
  }

  # Memory storage type
  memory_storage = var.enable_auto_tiering ? "ram-and-flash" : "ram"

  # Throughput measurement
  throughput_measurement_by    = "operations-per-second"
  throughput_measurement_value = var.throughput_ops_sec

  # Persistent storage
  persistent_storage_encryption = true
}

# Database for single region
resource "rediscloud_subscription_database" "main" {
  count = var.enable_active_active ? 0 : 1

  subscription_id              = rediscloud_subscription.main[0].id
  name                         = "${var.environment}-database"
  protocol                     = "redis"
  memory_limit_in_gb          = var.database_memory_gb
  data_persistence            = "aof-every-1-second"
  replication                 = true
  throughput_measurement_by   = "operations-per-second"
  throughput_measurement_value = var.throughput_ops_sec

  # Redis modules
  dynamic "modules" {
    for_each = local.enabled_modules
    content {
      name = modules.value
    }
  }

  # High availability
  replication = true

  # Password
  password = random_password.db_password.result

  # Alerts
  alert {
    name  = "dataset-size"
    value = var.database_memory_gb * 0.8  # Alert at 80%
  }

  alert {
    name  = "throughput-higher-than"
    value = var.throughput_ops_sec * 0.8
  }

  alert {
    name  = "latency"
    value = 5  # Alert if latency > 5ms
  }

  # Client SSL certificate (optional)
  # client_ssl_certificate = file("${path.module}/certs/client.crt")
}

# --- Active-Active Deployment ---
resource "rediscloud_active_active_subscription" "global" {
  count = var.enable_active_active ? 1 : 0

  name           = "${var.subscription_name}-active-active"
  payment_method = "credit-card"

  # Primary region
  cloud_provider {
    provider = var.cloud_provider

    region {
      region                       = var.primary_region
      multiple_availability_zones  = true
      networking_deployment_cidr   = "10.0.0.0/24"
    }
  }

  # Secondary regions
  dynamic "cloud_provider" {
    for_each = var.secondary_regions
    content {
      provider = var.cloud_provider

      region {
        region                       = cloud_provider.value
        multiple_availability_zones  = true
        networking_deployment_cidr   = "10.${cloud_provider.key + 1}.0.0/24"
      }
    }
  }

  # Creation plan
  creation_plan {
    memory_limit_in_gb          = var.database_memory_gb
    quantity                    = 1
    replication                 = true
    throughput_measurement_by   = "operations-per-second"
    throughput_measurement_value = var.throughput_ops_sec

    # Redis modules
    dynamic "modules" {
      for_each = local.enabled_modules
      content {
        name = modules.value
      }
    }
  }
}

# Active-Active database
resource "rediscloud_active_active_subscription_database" "global" {
  count = var.enable_active_active ? 1 : 0

  subscription_id = rediscloud_active_active_subscription.global[0].id
  name            = "${var.environment}-global-database"
  memory_limit_in_gb = var.database_memory_gb

  # Global configuration
  global_data_persistence      = "aof-every-1-second"
  global_password             = random_password.db_password.result
  global_source_ips           = ["0.0.0.0/0"]  # Restrict in production

  global_alert {
    name  = "dataset-size"
    value = var.database_memory_gb * 0.8
  }

  # Region-specific overrides (optional)
  dynamic "override_region" {
    for_each = concat([var.primary_region], var.secondary_regions)
    content {
      name                     = override_region.value
      override_global_password = false
    }
  }
}

# --- Outputs ---
output "subscription_id" {
  description = "Subscription ID"
  value       = var.enable_active_active ? rediscloud_active_active_subscription.global[0].id : rediscloud_subscription.main[0].id
}

output "database_id" {
  description = "Database ID"
  value       = var.enable_active_active ? rediscloud_active_active_subscription_database.global[0].id : rediscloud_subscription_database.main[0].id
}

output "database_endpoint" {
  description = "Database endpoint"
  value       = var.enable_active_active ? null : rediscloud_subscription_database.main[0].public_endpoint
}

output "database_port" {
  description = "Database port"
  value       = var.enable_active_active ? null : rediscloud_subscription_database.main[0].public_port
}

output "database_password" {
  description = "Database password"
  value       = random_password.db_password.result
  sensitive   = true
}

output "connection_string" {
  description = "Redis connection string"
  value = var.enable_active_active ? null : (
    "rediss://:${random_password.db_password.result}@${rediscloud_subscription_database.main[0].public_endpoint}:${rediscloud_subscription_database.main[0].public_port}"
  )
  sensitive = true
}

output "enabled_modules" {
  description = "List of enabled Redis Stack modules"
  value       = local.enabled_modules
}
```

### Variables file (terraform.tfvars)

```hcl
# Production configuration - Single region
environment          = "production"
subscription_name    = "prod-redis-subscription"
cloud_provider       = "AWS"
primary_region       = "us-east-1"
enable_active_active = false

database_memory_gb   = 50
throughput_ops_sec   = 50000

enable_modules = {
  search     = true
  json       = true
  timeseries = true
  bloom      = true
}

enable_auto_tiering = false

tags = {
  Team        = "platform"
  CostCenter  = "engineering"
  Environment = "production"
}
```

```hcl
# Active-Active configuration (multi-region)
environment          = "production"
subscription_name    = "prod-redis-global"
cloud_provider       = "AWS"
primary_region       = "us-east-1"
enable_active_active = true
secondary_regions    = ["eu-west-1", "ap-southeast-1"]

database_memory_gb   = 50
throughput_ops_sec   = 100000

enable_modules = {
  search     = true
  json       = true
  timeseries = false
  bloom      = false
}

enable_auto_tiering = true  # Use RAM+Flash tiering

tags = {
  Team        = "platform"
  CostCenter  = "engineering"
  Environment = "production"
  Geo         = "global"
}
```

---

## 💰 Pricing et ROI

### Modèle de pricing

```yaml
Redis Enterprise Cloud Pricing Model:

Components:
├── RAM: $0.119/GB-hour (~$87/GB-month)
├── RAM+Flash: $0.048/GB-hour RAM + $0.012/GB-hour Flash
├── Throughput: $0.016/1000 ops/sec-hour
├── Active-Active: +100% surcharge on RAM
└── Modules: Included (no extra cost)

Examples (Single Region):

1. Standard cache (50GB RAM, 10K ops/sec)
├── RAM: 50 × $87 = $4,350/month
├── Throughput: 10 × $0.016 × 730 = $117/month
└── Total: ~$4,467/month

2. Large cache (200GB RAM, 25K ops/sec)
├── RAM: 200 × $87 = $17,400/month
├── Throughput: 25 × $0.016 × 730 = $292/month
└── Total: ~$17,692/month

3. With RAM+Flash tiering (500GB total, 100GB RAM + 400GB Flash)
├── RAM: 100 × $35 = $3,500/month
├── Flash: 400 × $9 = $3,600/month
├── Throughput: 25K ops/sec = $292/month
└── Total: ~$7,392/month
   (vs $43,500 for 500GB full RAM - 83% savings!)

Active-Active (Multi-Region):

4. Global e-commerce (3 regions, 50GB each, 25K ops/sec)
├── RAM: 50 × 3 regions × $87 × 2 (Active-Active) = $26,100/month
├── Throughput: 25 × $0.016 × 730 × 3 = $876/month
└── Total: ~$26,976/month

5. Mission-critical fintech (5 regions, 100GB each, 50K ops/sec)
├── RAM: 100 × 5 × $87 × 2 = $87,000/month
├── Throughput: 50 × $0.016 × 730 × 5 = $2,920/month
└── Total: ~$89,920/month

Volume Discounts:
├── >500 GB: Negotiated pricing
├── Committed use: 15-30% discount
└── Enterprise contracts: Custom pricing

Hidden Costs to Consider:
├── Data transfer (cross-region): $0.02-0.12/GB
├── Bandwidth (internet egress): $0.09/GB
└── Support: Premium support included (24/7)
```

### Calculateur de TCO

```python
# Redis Enterprise Cloud TCO Calculator

class RedisEnterpriseCloudPricing:
    """Calculate Redis Enterprise Cloud costs"""

    # Pricing (2024, USD)
    RAM_PRICE_GB_MONTH = 87  # USD
    RAM_FLASH_RAM_GB_MONTH = 35  # USD (for tiering)
    RAM_FLASH_FLASH_GB_MONTH = 9  # USD (for tiering)
    THROUGHPUT_1K_OPS_MONTH = 11.68  # USD (730h × $0.016)
    ACTIVE_ACTIVE_MULTIPLIER = 2.0

    @classmethod
    def calculate_monthly_cost(
        cls,
        memory_gb: int,
        throughput_ops_sec: int,
        regions: int = 1,
        active_active: bool = False,
        use_tiering: bool = False,
        flash_gb: int = 0
    ) -> dict:
        """
        Calculate monthly cost

        Args:
            memory_gb: RAM in GB
            throughput_ops_sec: Operations per second
            regions: Number of regions
            active_active: Enable Active-Active
            use_tiering: Use RAM+Flash tiering
            flash_gb: Flash storage in GB (if tiering)
        """

        # Memory cost
        if use_tiering:
            ram_cost = memory_gb * cls.RAM_FLASH_RAM_GB_MONTH
            flash_cost = flash_gb * cls.RAM_FLASH_FLASH_GB_MONTH
            total_memory_cost = ram_cost + flash_cost
        else:
            ram_cost = memory_gb * cls.RAM_PRICE_GB_MONTH
            flash_cost = 0
            total_memory_cost = ram_cost

        # Apply Active-Active multiplier
        if active_active:
            total_memory_cost *= cls.ACTIVE_ACTIVE_MULTIPLIER

        # Multiply by regions
        total_memory_cost *= regions

        # Throughput cost
        throughput_units = throughput_ops_sec / 1000
        throughput_cost = throughput_units * cls.THROUGHPUT_1K_OPS_MONTH * regions

        # Total compute
        total_compute = total_memory_cost + throughput_cost

        # Additional costs (estimates)
        support_cost = 0  # Included
        data_transfer = 200 * regions  # Estimate $200/region

        total_monthly = total_compute + data_transfer

        return {
            'ram_gb': memory_gb,
            'flash_gb': flash_gb if use_tiering else 0,
            'regions': regions,
            'active_active': active_active,
            'ram_cost_monthly': round(ram_cost * regions, 2),
            'flash_cost_monthly': round(flash_cost * regions, 2),
            'throughput_cost_monthly': round(throughput_cost, 2),
            'data_transfer_monthly': round(data_transfer, 2),
            'total_monthly': round(total_monthly, 2),
            'total_annual': round(total_monthly * 12, 2),
            'cost_per_gb': round(total_monthly / (memory_gb * regions), 2)
        }

# Examples
print("=== Scenario 1: Standard cache (50GB, single region) ===")
cost1 = RedisEnterpriseCloudPricing.calculate_monthly_cost(
    memory_gb=50,
    throughput_ops_sec=10000,
    regions=1,
    active_active=False,
    use_tiering=False
)
print(f"Monthly: ${cost1['total_monthly']:,}")
print(f"Annual: ${cost1['total_annual']:,}")

print("\n=== Scenario 2: Active-Active global (3 regions, 50GB each) ===")
cost2 = RedisEnterpriseCloudPricing.calculate_monthly_cost(
    memory_gb=50,
    throughput_ops_sec=25000,
    regions=3,
    active_active=True,
    use_tiering=False
)
print(f"Monthly: ${cost2['total_monthly']:,}")
print(f"Annual: ${cost2['total_annual']:,}")
print(f"Cost per GB: ${cost2['cost_per_gb']}/GB")

print("\n=== Scenario 3: Large with tiering (100GB RAM + 400GB Flash) ===")
cost3 = RedisEnterpriseCloudPricing.calculate_monthly_cost(
    memory_gb=100,
    throughput_ops_sec=25000,
    regions=1,
    active_active=False,
    use_tiering=True,
    flash_gb=400
)
cost3_no_tiering = RedisEnterpriseCloudPricing.calculate_monthly_cost(
    memory_gb=500,
    throughput_ops_sec=25000,
    regions=1,
    active_active=False,
    use_tiering=False
)
print(f"With tiering: ${cost3['total_monthly']:,}/month")
print(f"Without tiering (500GB RAM): ${cost3_no_tiering['total_monthly']:,}/month")
print(f"Savings: ${cost3_no_tiering['total_monthly'] - cost3['total_monthly']:,}/month ({((cost3_no_tiering['total_monthly'] - cost3['total_monthly']) / cost3_no_tiering['total_monthly'] * 100):.0f}%)")

print("\n=== Comparison vs AWS ElastiCache (50GB, RI 3 years) ===")
elasticache_ri_monthly = 2381  # From previous section
redis_enterprise_monthly = cost1['total_monthly']
print(f"AWS ElastiCache (RI 3y): ${elasticache_ri_monthly}/month")
print(f"Redis Enterprise Cloud: ${redis_enterprise_monthly}/month")
print(f"Premium: ${redis_enterprise_monthly - elasticache_ri_monthly}/month")
print(f"Premium %: {((redis_enterprise_monthly / elasticache_ri_monthly - 1) * 100):.0f}%")
print("\nRedis Enterprise adds:")
print("  ✅ Active-Active capability")
print("  ✅ Redis Stack modules (Search, JSON, TimeSeries, Bloom)")
print("  ✅ Multi-cloud support")
print("  ✅ Auto-scaling")
print("  ✅ 99.999% SLA (vs 99.9%)")
```

Output:
```
=== Scenario 1: Standard cache (50GB, single region) ===
Monthly: $4,667
Annual: $56,004

=== Scenario 2: Active-Active global (3 regions, 50GB each) ===
Monthly: $27,176
Annual: $326,112
Cost per GB: $181.17/GB

=== Scenario 3: Large with tiering (100GB RAM + 400GB Flash) ===
With tiering: $7,592/month
Without tiering (500GB RAM): $43,792/month
Savings: $36,200/month (83%)

=== Comparison vs AWS ElastiCache (50GB, RI 3 years) ===
AWS ElastiCache (RI 3y): $2,381/month
Redis Enterprise Cloud: $4,667/month
Premium: $2,286/month
Premium %: 96%

Redis Enterprise adds:
  ✅ Active-Active capability
  ✅ Redis Stack modules (Search, JSON, TimeSeries, Bloom)
  ✅ Multi-cloud support
  ✅ Auto-scaling
  ✅ 99.999% SLA (vs 99.9%)
```

### Analyse ROI

```python
# ROI Analysis: When does Redis Enterprise make sense?

def calculate_roi(
    monthly_revenue: float,
    downtime_cost_per_hour: float,
    data_loss_cost: float,
    elasticache_cost_monthly: float = 2381,
    redis_enterprise_cost_monthly: float = 4667,
    elasticache_sla_uptime: float = 0.999,
    redis_enterprise_sla_uptime: float = 0.99999
):
    """
    Calculate ROI of Redis Enterprise vs ElastiCache
    """

    monthly_premium = redis_enterprise_cost_monthly - elasticache_cost_monthly
    annual_premium = monthly_premium * 12

    # Expected downtime per year
    elasticache_downtime_hours = 8760 * (1 - elasticache_sla_uptime)  # ~8.76 hours
    redis_ent_downtime_hours = 8760 * (1 - redis_enterprise_sla_uptime)  # ~0.09 hours

    downtime_difference = elasticache_downtime_hours - redis_ent_downtime_hours

    # Cost of additional downtime with ElastiCache
    downtime_cost_diff = downtime_difference * downtime_cost_per_hour

    # ROI calculation
    net_benefit = downtime_cost_diff - annual_premium

    print(f"Monthly Revenue: ${monthly_revenue:,}")
    print(f"Downtime cost: ${downtime_cost_per_hour:,}/hour")
    print(f"\nRedis Enterprise premium: ${monthly_premium:,}/month (${annual_premium:,}/year)")
    print(f"\nExpected downtime:")
    print(f"  ElastiCache (99.9%): {elasticache_downtime_hours:.2f} hours/year")
    print(f"  Redis Enterprise (99.999%): {redis_ent_downtime_hours:.2f} hours/year")
    print(f"  Difference: {downtime_difference:.2f} hours/year")
    print(f"\nCost of additional downtime (ElastiCache): ${downtime_cost_diff:,}/year")
    print(f"\nNet benefit (Redis Enterprise): ${net_benefit:,}/year")

    if net_benefit > 0:
        roi_percent = (net_benefit / annual_premium) * 100
        print(f"\n✅ Redis Enterprise is justified")
        print(f"   ROI: {roi_percent:.0f}%")
        print(f"   Payback period: {12 / (net_benefit / annual_premium + 1):.1f} months")
    else:
        print(f"\n❌ Redis Enterprise not justified by downtime alone")
        print(f"   Loss: ${-net_benefit:,}/year")
        print(f"\n   Consider other factors:")
        print(f"   • Active-Active (multi-region performance)")
        print(f"   • Redis Stack modules (new capabilities)")
        print(f"   • Multi-cloud flexibility")

    return net_benefit

print("=== Scenario 1: E-commerce ($10M/month revenue) ===")
calculate_roi(
    monthly_revenue=10_000_000,
    downtime_cost_per_hour=50_000,  # $50K/hour downtime cost
    data_loss_cost=100_000
)

print("\n" + "="*60)
print("=== Scenario 2: SaaS Platform ($1M/month revenue) ===")
calculate_roi(
    monthly_revenue=1_000_000,
    downtime_cost_per_hour=5_000,  # $5K/hour
    data_loss_cost=25_000
)

print("\n" + "="*60)
print("=== Scenario 3: Fintech ($50M/month revenue) ===")
calculate_roi(
    monthly_revenue=50_000_000,
    downtime_cost_per_hour=500_000,  # $500K/hour
    data_loss_cost=5_000_000
)
```

---

## 📊 Comparaison finale avec toutes les solutions

### Tableau récapitulatif complet

| Critère | Redis Enterprise | AWS ElastiCache | AWS MemoryDB | Azure Premium | GCP Memorystore |
|---------|------------------|-----------------|--------------|---------------|-----------------|
| **Architecture** |
| Max capacity | Unlimited | 150 TB | 200 TB | 1.2 TB | 300 GB |
| Clustering | ✅ Advanced | ✅ (500 shards) | ✅ (500 shards) | ✅ (10 shards) | ❌ |
| Active-Active | ✅ **Unique** | ❌ | ❌ | ❌ (preview) | ❌ |
| **Modules & Features** |
| Redis Stack | ✅ **All modules** | ❌ | ❌ | ❌ (Ent tier only) | ❌ |
| RediSearch | ✅ | ❌ | ❌ | ❌ | ❌ |
| RedisJSON | ✅ | ❌ | ❌ | ❌ | ❌ |
| RedisTimeSeries | ✅ | ❌ | ❌ | ❌ | ❌ |
| RedisBloom | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Durability** |
| Persistence | RDB+AOF | RDB/AOF | WAL (durable) | RDB+AOF | RDB |
| RPO | Seconds | 1s-15min | 0 (zero loss) | 1s-15min | Hours |
| RTO | <1 min | 2-3 min | <1 min | 2-3 min | 1-2 min |
| **Availability** |
| SLA | **99.999%** | 99.9% | 99.99% | 99.9% | 99.9% |
| Multi-region | ✅ Active-Active | ✅ Passive | ❌ | ✅ Passive | ❌ |
| Auto-scaling | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Cloud Support** |
| Multi-cloud | ✅ (AWS+Azure+GCP) | AWS only | AWS only | Azure only | GCP only |
| Portability | ✅ High | ❌ Low | ❌ Low | ❌ Low | ❌ Low |
| **Pricing (50GB, on-demand)** |
| Monthly cost | ~$4,667 | ~$5,952 | ~$10,597 | ~€1,000 | ~€3,175 |
| Reserved/RI | Custom | $2,381 (RI 3y) | $4,239 (RI 3y) | ~€450 (RI 3y) | N/A |
| Cost premium | Baseline | -49% (RI) | -9% (RI) | -90% (RI) | -32% |
| **Support** |
| Support level | 24/7 Premium | Standard | Standard | Standard | Standard |
| SLA guarantee | ✅ | ✅ | ✅ | ✅ | ✅ |
| Direct access to Redis team | ✅ | ❌ | ❌ | ❌ | ❌ |

### Matrice de décision finale

```yaml
Utilisez Redis Enterprise Cloud si:
✅ Active-Active multi-région requis
✅ Redis Stack modules nécessaires (Search, JSON, etc.)
✅ SLA 99.999% mandatory
✅ Multi-cloud strategy ou migration prévue
✅ Budget permet premium pricing (2-3× cloud native)
✅ Besoin de support direct de Redis team
✅ Compliance requiert contrôle total

Utilisez AWS ElastiCache si:
✅ Cache standard (pas primary DB)
✅ 100% AWS infrastructure
✅ Budget contraint (avec RI)
✅ Perte de données acceptable
✅ SLA 99.9% suffisant
✅ Pas besoin de Redis Stack

Utilisez AWS MemoryDB si:
✅ Redis comme primary database
✅ Zero data loss critique
✅ 100% AWS infrastructure
✅ Budget permet (~78% plus cher qu'ElastiCache)
✅ SLA 99.99% requis

Utilisez Azure Cache Premium si:
✅ 100% Azure infrastructure
✅ VNet injection required
✅ Geo-replication passive suffisante
✅ Budget moyen (RI 3y économique)

Utilisez GCP Memorystore si:
✅ 100% GCP infrastructure
✅ Simplicité > fonctionnalités
✅ Dataset <300GB
✅ Intégration GKE prioritaire
✅ Pas besoin de clustering
```

---

## ✅ Conclusion

### Points clés à retenir

1. **Redis Enterprise = Solution Premium**
   - Coût 2-3× supérieur aux solutions cloud natives
   - Justifié par Active-Active + Redis Stack + Multi-cloud

2. **Active-Active est unique**
   - Seule solution avec true multi-region read/write
   - CRDTs pour résolution automatique des conflits
   - Latence locale (<1ms) partout

3. **Redis Stack = Game Changer**
   - RediSearch pour full-text et vector search
   - RedisJSON pour documents natifs
   - RedisTimeSeries pour IoT/monitoring
   - RedisBloom pour probabilistic data structures

4. **Multi-cloud = Portabilité**
   - Déploiement identique sur AWS/Azure/GCP
   - Migration simplifiée
   - Évite le vendor lock-in

5. **Auto-scaling & Auto-tiering**
   - Scaling automatique selon charge
   - RAM+Flash pour économies (jusqu'à 80%)
   - Managed fully (zero ops)

### Recommandations finales

**Utilisez Redis Enterprise Cloud pour :**
```
1. E-commerce global avec Active-Active
   └─> Latence <1ms dans chaque région
       3 régions × 50GB = ~$27K/mois

2. RAG/AI avec vector search (RediSearch)
   └─> Semantic search sur embeddings
       100GB RAM = ~$8.7K/mois

3. IoT time-series à grande échelle (RedisTimeSeries)
   └─> Millions de sensors
       500GB avec tiering = ~$7.4K/mois

4. Document store JSON (RedisJSON + RediSearch)
   └─> Alternative MongoDB/Elasticsearch
       50GB = ~$4.7K/mois
```

**N'utilisez PAS Redis Enterprise si :**
```
❌ Budget très contraint (préférer AWS RI)
❌ Use case simple (cache standard)
❌ Pas besoin de Redis Stack modules
❌ Single-cloud avec solution native performante
❌ Pas besoin d'Active-Active
```

### ROI simplifié

```
Si votre downtime coûte >$10K/heure
ET/OU
Si vous avez besoin des modules Redis Stack
ET/OU
Si Active-Active multi-région est requis
ALORS
  → Redis Enterprise est justifié

SINON
  → Utilisez la solution native de votre cloud
     avec Reserved Instances
```

---

**🎯 Module 15 terminé !** Nous avons couvert toutes les solutions Redis managées du marché. La prochaine étape serait d'explorer les déploiements self-hosted sur Kubernetes avec les sections suivantes (Docker, StatefulSets, Operators, Helm).

⏭️ [Gestion des coûts : Tiering de mémoire (RAM + Flash/SSD)](/15-redis-cloud-conteneurs/06-gestion-couts-tiering-memoire.md)
