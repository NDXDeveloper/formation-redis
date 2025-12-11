🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.6 Gestion des coûts : Tiering de mémoire (RAM + Flash/SSD)

## 🎯 Objectifs

- Comprendre l'architecture du tiering RAM+Flash/SSD
- Maîtriser les algorithmes de promotion/dégradation des données
- Analyser l'impact performance vs coût
- Configurer le tiering sur Redis Enterprise et Azure
- Calculer les économies potentielles selon les workloads
- Identifier les cas d'usage appropriés pour le tiering

---

## 🏗️ Architecture du tiering de mémoire

### Concept fondamental

```
┌────────────────────────────────────────────────────────────────┐
│              RAM + Flash Tiering Architecture                  │
│                                                                │
│  Principe: Hot data in RAM, Cold data on Flash                 │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Application Layer                      │  │
│  │   (No changes needed - transparent to app)               │  │
│  └─────────────────────────┬────────────────────────────────┘  │
│                            │                                   │
│                            │ Redis Protocol                    │
│                            ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               Redis Enterprise Engine                    │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │        RAM Tier (Hot Data)                         │  │  │
│  │  │                                                    │  │  │
│  │  │  • Frequently accessed keys                        │  │  │
│  │  │  • Recently written data                           │  │  │
│  │  │  • High-priority keys (configurable)               │  │  │
│  │  │  • Latency: <1ms                                   │  │  │
│  │  │                                                    │  │  │
│  │  │  Example: 20% of dataset (80% of requests)         │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                            │                             │  │
│  │                            │ Auto-tiering engine         │  │
│  │                            ▼                             │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │        Flash/SSD Tier (Cold Data)                  │  │  │
│  │  │                                                    │  │  │
│  │  │  • Infrequently accessed keys                      │  │  │
│  │  │  • Older data                                      │  │  │
│  │  │  • Archive data                                    │  │  │
│  │  │  • Latency: 1-5ms (on cache miss)                  │  │  │
│  │  │                                                    │  │  │
│  │  │  Example: 80% of dataset (20% of requests)         │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                                                          │  │
│  │  Auto-Tiering Algorithm:                                 │  │
│  │  ├─ Promotion: Flash → RAM (on frequent access)          │  │
│  │  ├─ Demotion: RAM → Flash (on infrequent access)         │  │
│  │  ├─ Algorithm: LRU-based with thresholds                 │  │
│  │  └─ Transparent: No app changes needed                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  Cost Comparison (per GB-month):                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  RAM:   $87/GB-month   (baseline)                       │   │
│  │  Flash: $9/GB-month    (90% cheaper!)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  Example: 500GB dataset with 80/20 hot/cold ratio              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Full RAM:    500GB × $87 = $43,500/month               │   │
│  │  With Tiering:                                          │   │
│  │    - RAM:  100GB × $35 = $3,500/month                   │   │
│  │    - Flash: 400GB × $9 = $3,600/month                   │   │
│  │    - Total: $7,100/month                                │   │
│  │  Savings: $36,400/month (84%!)                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

### Algorithme de tiering détaillé

```
┌────────────────────────────────────────────────────────────────┐
│            Auto-Tiering Algorithm (LRU-based)                  │
│                                                                │
│  Promotion (Flash → RAM):                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Trigger: Key accessed on Flash tier                     │  │
│  │  Decision:                                               │  │
│  │    IF access_count >= threshold (e.g., 3 in 1 hour)      │  │
│  │    THEN promote to RAM                                   │  │
│  │    ELSE stay on Flash                                    │  │
│  │                                                          │  │
│  │  Process:                                                │  │
│  │  1. Read key from Flash                                  │  │
│  │  2. Increment access counter                             │  │
│  │  3. If counter >= threshold:                             │  │
│  │     a. Move key to RAM                                   │  │
│  │     b. If RAM full, demote LRU key to Flash              │  │
│  │     c. Update access metadata                            │  │
│  │  4. Return value to client                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  Demotion (RAM → Flash):                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Trigger: RAM utilization >= threshold (e.g., 95%)       │  │
│  │  Decision:                                               │  │
│  │    Select LRU keys from RAM                              │  │
│  │    IF key not accessed in last N seconds (e.g., 3600)    │  │
│  │    THEN demote to Flash                                  │  │
│  │                                                          │  │
│  │  Process:                                                │  │
│  │  1. Identify LRU keys in RAM                             │  │
│  │  2. Check access timestamp                               │  │
│  │  3. Write key to Flash (asynchronously)                  │  │
│  │  4. After write ACK, remove from RAM                     │  │
│  │  5. Update metadata                                      │  │
│  │  6. Free RAM space                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  Cache Miss Path (Flash read):                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  1. Client GET request                                   │  │
│  │  2. Check RAM (miss)                                     │  │
│  │  3. Check Flash metadata index (in RAM)                  │  │
│  │  4. Read from Flash/SSD                                  │  │
│  │     └─> Latency: 1-5ms (vs <1ms for RAM)                 │  │
│  │  5. Optionally promote to RAM                            │  │
│  │  6. Return value to client                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  Optimization: Flash Index in RAM                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • Metadata (key location, size) kept in RAM             │  │
│  │  • Bloom filters for fast negative lookups               │  │
│  │  • Compressed keys index                                 │  │
│  │  • Overhead: ~1-2% of Flash size                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## 📊 Impact sur les performances

### Latence comparative

```yaml
Performance Characteristics:

RAM (baseline):
├── Read latency P50: 0.5ms
├── Read latency P99: 1ms
├── Write latency P50: 0.7ms
├── Write latency P99: 1.5ms
└── Throughput: 100K+ ops/sec per shard

Flash/SSD (on cache miss):
├── Read latency P50: 2ms (4x slower)
├── Read latency P99: 5ms (5x slower)
├── Write latency P50: 1ms (same - writes go to RAM first)
├── Write latency P99: 2ms
└── Throughput: 50K ops/sec per shard (limited by I/O)

RAM + Flash Tiered (realistic workload):
├── Read latency P50: 0.8ms (20% hit Flash)
├── Read latency P99: 3ms
├── Write latency P50: 0.7ms (writes to RAM)
├── Write latency P99: 1.5ms
└── Throughput: 80K ops/sec per shard

Impact by Access Pattern:
┌────────────────────────────────────────────────────────────┐
│ Access Pattern    │ RAM Hit Rate │ Avg Latency │ Impact    │
├────────────────────────────────────────────────────────────┤
│ Very Hot (90/10)  │     90%      │   0.6ms     │ Minimal   │
│ Hot (80/20)       │     80%      │   0.8ms     │ Low       │
│ Warm (70/30)      │     70%      │   1.1ms     │ Moderate  │
│ Cool (60/40)      │     60%      │   1.5ms     │ High      │
│ Cold (50/50)      │     50%      │   2.0ms     │ Severe    │
└────────────────────────────────────────────────────────────┘

Recommendation: Use tiering only if >70% RAM hit rate expected
```

### Benchmark comparatif

```python
# Performance benchmark: RAM vs RAM+Flash

import redis
import time
import statistics

def benchmark_redis(host, port, password, iterations=10000):
    """Benchmark Redis operations"""
    r = redis.Redis(
        host=host,
        port=port,
        password=password,
        ssl=True,
        decode_responses=True
    )

    # Warm up
    for i in range(100):
        r.set(f"warmup:{i}", "value")

    # Write benchmark
    write_latencies = []
    for i in range(iterations):
        start = time.perf_counter()
        r.set(f"key:{i}", f"value_{i}")
        end = time.perf_counter()
        write_latencies.append((end - start) * 1000)  # ms

    # Read benchmark (hot keys - should be in RAM)
    read_hot_latencies = []
    for i in range(iterations):
        start = time.perf_counter()
        r.get(f"key:{i}")
        end = time.perf_counter()
        read_hot_latencies.append((end - start) * 1000)

    # Read benchmark (cold keys - may be on Flash)
    # Generate keys not in cache
    for i in range(iterations, iterations * 2):
        r.set(f"cold:{i}", f"value_{i}")

    # Wait for potential demotion to Flash
    time.sleep(60)

    read_cold_latencies = []
    for i in range(iterations, iterations * 2):
        start = time.perf_counter()
        r.get(f"cold:{i}")
        end = time.perf_counter()
        read_cold_latencies.append((end - start) * 1000)

    return {
        'write_p50': statistics.median(write_latencies),
        'write_p99': statistics.quantiles(write_latencies, n=100)[98],
        'read_hot_p50': statistics.median(read_hot_latencies),
        'read_hot_p99': statistics.quantiles(read_hot_latencies, n=100)[98],
        'read_cold_p50': statistics.median(read_cold_latencies),
        'read_cold_p99': statistics.quantiles(read_cold_latencies, n=100)[98]
    }

# Example results:
# Full RAM instance:
#   write_p50: 0.52ms, write_p99: 1.1ms
#   read_hot_p50: 0.48ms, read_hot_p99: 0.95ms
#   read_cold_p50: 0.51ms, read_cold_p99: 0.98ms

# RAM+Flash instance:
#   write_p50: 0.54ms, write_p99: 1.2ms
#   read_hot_p50: 0.50ms, read_hot_p99: 1.0ms
#   read_cold_p50: 2.1ms, read_cold_p99: 4.8ms  ← Flash reads
```

---

## 💰 Analyse des coûts

### Modèle de coût détaillé

```python
# Cost Analysis: Full RAM vs RAM+Flash Tiering

class TieringCostAnalysis:
    """Analyze cost savings with RAM+Flash tiering"""

    # Pricing (Redis Enterprise Cloud, 2024)
    RAM_ONLY_PRICE_GB_MONTH = 87  # USD
    RAM_TIERED_PRICE_GB_MONTH = 35  # USD (when using tiering)
    FLASH_PRICE_GB_MONTH = 9  # USD

    @classmethod
    def calculate_cost(
        cls,
        total_dataset_gb: int,
        ram_percent: float,  # 0.0 to 1.0
        use_tiering: bool = True
    ) -> dict:
        """
        Calculate cost for different configurations

        Args:
            total_dataset_gb: Total dataset size in GB
            ram_percent: Percentage of data to keep in RAM (0.0-1.0)
            use_tiering: Use RAM+Flash tiering vs full RAM
        """

        if not use_tiering:
            # Full RAM configuration
            total_cost = total_dataset_gb * cls.RAM_ONLY_PRICE_GB_MONTH
            ram_cost = total_cost
            flash_cost = 0
            ram_gb = total_dataset_gb
            flash_gb = 0
        else:
            # RAM+Flash tiering
            ram_gb = total_dataset_gb * ram_percent
            flash_gb = total_dataset_gb * (1 - ram_percent)

            ram_cost = ram_gb * cls.RAM_TIERED_PRICE_GB_MONTH
            flash_cost = flash_gb * cls.FLASH_PRICE_GB_MONTH
            total_cost = ram_cost + flash_cost

        # Calculate savings vs full RAM
        full_ram_cost = total_dataset_gb * cls.RAM_ONLY_PRICE_GB_MONTH
        savings = full_ram_cost - total_cost
        savings_percent = (savings / full_ram_cost) * 100 if full_ram_cost > 0 else 0

        return {
            'total_dataset_gb': total_dataset_gb,
            'ram_gb': ram_gb,
            'flash_gb': flash_gb,
            'ram_percent': ram_percent * 100,
            'ram_cost_monthly': round(ram_cost, 2),
            'flash_cost_monthly': round(flash_cost, 2),
            'total_cost_monthly': round(total_cost, 2),
            'full_ram_cost_monthly': round(full_ram_cost, 2),
            'savings_monthly': round(savings, 2),
            'savings_percent': round(savings_percent, 1),
            'cost_per_gb': round(total_cost / total_dataset_gb, 2)
        }

# Examples
print("=== Scenario 1: 100GB dataset ===")
for ram_pct in [1.0, 0.5, 0.3, 0.2]:
    result = TieringCostAnalysis.calculate_cost(100, ram_pct, use_tiering=True)
    print(f"\nRAM: {int(result['ram_percent'])}% ({result['ram_gb']}GB)")
    print(f"  Total cost: ${result['total_cost_monthly']:,}/month")
    print(f"  Savings: ${result['savings_monthly']:,}/month ({result['savings_percent']}%)")
    print(f"  Cost/GB: ${result['cost_per_gb']}/GB")

print("\n" + "="*60)
print("=== Scenario 2: 500GB dataset ===")
for ram_pct in [1.0, 0.3, 0.2, 0.15]:
    result = TieringCostAnalysis.calculate_cost(500, ram_pct, use_tiering=True)
    print(f"\nRAM: {int(result['ram_percent'])}% ({result['ram_gb']}GB)")
    print(f"  Total cost: ${result['total_cost_monthly']:,}/month")
    print(f"  Savings: ${result['savings_monthly']:,}/month ({result['savings_percent']}%)")

print("\n" + "="*60)
print("=== Scenario 3: 2TB dataset ===")
for ram_pct in [1.0, 0.2, 0.15, 0.1]:
    result = TieringCostAnalysis.calculate_cost(2000, ram_pct, use_tiering=True)
    print(f"\nRAM: {int(result['ram_percent'])}% ({result['ram_gb']}GB)")
    print(f"  Total cost: ${result['total_cost_monthly']:,}/month")
    print(f"  Savings: ${result['savings_monthly']:,}/month ({result['savings_percent']}%)")
```

Output:
```
=== Scenario 1: 100GB dataset ===

RAM: 100% (100GB)
  Total cost: $8,700/month
  Savings: $0/month (0.0%)
  Cost/GB: $87/GB

RAM: 50% (50GB)
  Total cost: $2,200/month
  Savings: $6,500/month (74.7%)
  Cost/GB: $22/GB

RAM: 30% (30GB)
  Total cost: $1,680/month
  Savings: $7,020/month (80.7%)
  Cost/GB: $16.8/GB

RAM: 20% (20GB)
  Total cost: $1,420/month
  Savings: $7,280/month (83.7%)
  Cost/GB: $14.2/GB

=== Scenario 2: 500GB dataset ===

RAM: 100% (500GB)
  Total cost: $43,500/month
  Savings: $0/month (0.0%)

RAM: 30% (150GB)
  Total cost: $8,400/month
  Savings: $35,100/month (80.7%)

RAM: 20% (100GB)
  Total cost: $7,100/month
  Savings: $36,400/month (83.7%)

RAM: 15% (75GB)
  Total cost: $6,487/month
  Savings: $37,012/month (85.1%)

=== Scenario 3: 2TB dataset ===

RAM: 100% (2000GB)
  Total cost: $174,000/month
  Savings: $0/month (0.0%)

RAM: 20% (400GB)
  Total cost: $28,400/month
  Savings: $145,600/month (83.7%)

RAM: 15% (300GB)
  Total cost: $25,950/month
  Savings: $148,050/month (85.1%)

RAM: 10% (200GB)
  Total cost: $23,200/month
  Savings: $150,800/month (86.7%)
```

### Break-even analysis

```python
def calculate_breakeven_ram_ratio(
    dataset_gb: int,
    performance_degradation_acceptable: float = 0.15  # 15% latency increase
) -> float:
    """
    Calculate optimal RAM ratio for cost vs performance

    Assumptions:
    - Each 10% of data on Flash adds ~5% to average latency
    - Application can tolerate up to X% latency increase
    """

    # Model: avg_latency = base_latency * (1 + flash_ratio * 0.5)
    # Example: 50% on Flash → 25% latency increase

    max_flash_ratio = performance_degradation_acceptable / 0.5
    optimal_ram_ratio = 1.0 - max_flash_ratio

    # Ensure between 10% and 100%
    optimal_ram_ratio = max(0.1, min(1.0, optimal_ram_ratio))

    # Calculate costs
    full_ram = TieringCostAnalysis.calculate_cost(dataset_gb, 1.0, False)
    with_tiering = TieringCostAnalysis.calculate_cost(dataset_gb, optimal_ram_ratio, True)

    print(f"Dataset: {dataset_gb}GB")
    print(f"Performance tolerance: {performance_degradation_acceptable*100:.0f}% latency increase")
    print(f"\nOptimal configuration:")
    print(f"  RAM: {optimal_ram_ratio*100:.0f}% ({dataset_gb*optimal_ram_ratio:.0f}GB)")
    print(f"  Flash: {(1-optimal_ram_ratio)*100:.0f}% ({dataset_gb*(1-optimal_ram_ratio):.0f}GB)")
    print(f"\nCost comparison:")
    print(f"  Full RAM: ${full_ram['total_cost_monthly']:,}/month")
    print(f"  With tiering: ${with_tiering['total_cost_monthly']:,}/month")
    print(f"  Savings: ${with_tiering['savings_monthly']:,}/month ({with_tiering['savings_percent']:.1f}%)")
    print(f"\nExpected latency impact: +{performance_degradation_acceptable*100:.0f}%")

    return optimal_ram_ratio

# Examples
print("=== Example 1: Latency-sensitive app (10% tolerance) ===")
calculate_breakeven_ram_ratio(500, performance_degradation_acceptable=0.10)

print("\n" + "="*60)
print("=== Example 2: Cost-sensitive app (20% tolerance) ===")
calculate_breakeven_ram_ratio(500, performance_degradation_acceptable=0.20)

print("\n" + "="*60)
print("=== Example 3: Archive/analytics (30% tolerance) ===")
calculate_breakeven_ram_ratio(2000, performance_degradation_acceptable=0.30)
```

---

## 🔧 Configuration et déploiement

### Redis Enterprise Cloud avec Terraform

```hcl
# Redis Enterprise Cloud with RAM+Flash Tiering
# Already covered in section 15.5, here's the specific tiering config

resource "rediscloud_subscription" "tiered" {
  name           = "production-tiered-subscription"
  payment_method = "credit-card"

  cloud_provider {
    provider = "AWS"
    region {
      region                       = "us-east-1"
      multiple_availability_zones  = true
      networking_deployment_cidr   = "10.0.0.0/24"
    }
  }

  # Enable RAM+Flash tiering
  memory_storage = "ram-and-flash"  # ← Key setting

  throughput_measurement_by    = "operations-per-second"
  throughput_measurement_value = 25000
}

resource "rediscloud_subscription_database" "main" {
  subscription_id              = rediscloud_subscription.tiered.id
  name                         = "production-tiered-db"
  protocol                     = "redis"

  # Total memory: 500GB
  # Redis Enterprise will automatically manage RAM/Flash ratio
  memory_limit_in_gb          = 500

  data_persistence            = "aof-every-1-second"
  replication                 = true
  throughput_measurement_by   = "operations-per-second"
  throughput_measurement_value = 25000

  # Tiering is automatic - no manual configuration needed
  # Redis Enterprise manages hot/cold data based on access patterns
}
```

### Azure Cache for Redis Premium (Flash preview)

```hcl
# Azure Cache for Redis with Flash (preview feature)
# Note: As of 2024, this is in preview for Enterprise tiers only

resource "azurerm_redis_enterprise_cache" "tiered" {
  name                = "prod-redis-enterprise"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  sku_name = "Enterprise_E100"  # 100GB RAM

  # Enable Flash (preview)
  # This adds SSD capacity for cold data
  capacity = 2  # Multiplier for Flash capacity

  tags = {
    Environment = "production"
    Tiering     = "enabled"
  }
}

resource "azurerm_redis_enterprise_database" "main" {
  name                = "default"
  resource_group_name = azurerm_resource_group.main.name

  cluster_id = azurerm_redis_enterprise_cache.tiered.id

  # Configure modules
  module {
    name = "RedisJSON"
  }

  module {
    name = "RediSearch"
  }

  # Clustering configuration
  clustering_policy = "EnterpriseCluster"
  eviction_policy   = "NoEviction"  # Rely on Flash instead of eviction

  # Note: Flash configuration is managed at cluster level
}
```

### Monitoring du tiering

```python
# Monitor RAM+Flash tiering effectiveness

import redis
import time
from datetime import datetime

def monitor_tiering_stats(host, port, password, interval_sec=60):
    """
    Monitor tiering statistics

    Key metrics to watch:
    - RAM hit rate (should be >70%)
    - Flash read rate
    - Promotion/demotion rate
    """

    r = redis.Redis(
        host=host,
        port=port,
        password=password,
        ssl=True,
        decode_responses=True
    )

    print("Monitoring RAM+Flash tiering...")
    print("="*70)

    while True:
        try:
            info = r.info()

            # Calculate RAM hit rate
            total_commands = info.get('total_commands_processed', 0)
            flash_reads = info.get('flash_reads', 0)  # Redis Enterprise specific
            ram_hits = total_commands - flash_reads
            ram_hit_rate = (ram_hits / total_commands * 100) if total_commands > 0 else 0

            # Memory usage
            used_memory_ram = info.get('used_memory', 0)
            used_memory_flash = info.get('used_memory_flash', 0)  # Redis Enterprise
            total_memory = used_memory_ram + used_memory_flash

            ram_percent = (used_memory_ram / total_memory * 100) if total_memory > 0 else 0
            flash_percent = (used_memory_flash / total_memory * 100) if total_memory > 0 else 0

            # Tiering operations
            promotions = info.get('flash_to_ram_promotions', 0)
            demotions = info.get('ram_to_flash_demotions', 0)

            print(f"\n[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}]")
            print(f"RAM Hit Rate: {ram_hit_rate:.1f}%")
            print(f"Memory Distribution:")
            print(f"  RAM:   {used_memory_ram / (1024**3):.2f} GB ({ram_percent:.1f}%)")
            print(f"  Flash: {used_memory_flash / (1024**3):.2f} GB ({flash_percent:.1f}%)")
            print(f"  Total: {total_memory / (1024**3):.2f} GB")
            print(f"Tiering Operations:")
            print(f"  Promotions (Flash→RAM): {promotions}")
            print(f"  Demotions (RAM→Flash):  {demotions}")
            print(f"Commands Processed: {total_commands:,}")
            print(f"Flash Reads: {flash_reads:,}")

            # Alert if RAM hit rate too low
            if ram_hit_rate < 70:
                print(f"\n⚠️  WARNING: RAM hit rate below 70% ({ram_hit_rate:.1f}%)")
                print(f"   Consider increasing RAM allocation")

            time.sleep(interval_sec)

        except KeyboardInterrupt:
            print("\n\nMonitoring stopped.")
            break
        except Exception as e:
            print(f"Error: {e}")
            time.sleep(interval_sec)

# Example usage:
# monitor_tiering_stats(
#     host='redis-12345.c123.us-east-1-1.ec2.cloud.redislabs.com',
#     port=12345,
#     password='your-password',
#     interval_sec=60
# )
```

---

## 📈 Cas d'usage et patterns

### Workloads appropriés pour le tiering

```yaml
✅ Excellents cas d'usage pour RAM+Flash:

1. User Session Store (large scale)
   ├── Caractéristiques:
   │   ├── Millions de sessions
   │   ├── 90% des sessions sont actives (hot)
   │   ├── 10% sont dormantes (cold)
   │   └── Pattern d'accès: 95/5
   ├── Configuration recommandée:
   │   ├── Total: 500GB
   │   ├── RAM: 100GB (20%)
   │   └── Flash: 400GB (80%)
   └── Économies: 84% vs full RAM

2. Product Catalog (e-commerce)
   ├── Caractéristiques:
   │   ├── Millions de SKUs
   │   ├── 20% des produits = 80% des vues (Pareto)
   │   ├── Catalogue complet doit être queryable
   │   └── Pattern: 80/20
   ├── Configuration recommandée:
   │   ├── Total: 1TB
   │   ├── RAM: 250GB (25%)
   │   └── Flash: 750GB (75%)
   └── Économies: 81% vs full RAM

3. Time-Series Analytics (historical data)
   ├── Caractéristiques:
   │   ├── Recent data (last 7 days) = hot
   │   ├── Historical data (>7 days) = cold
   │   ├── 95% queries on recent data
   │   └── Pattern: 90/10
   ├── Configuration recommandée:
   │   ├── Total: 2TB
   │   ├── RAM: 300GB (15%)
   │   └── Flash: 1.7TB (85%)
   └── Économies: 86% vs full RAM

4. Content Delivery (CDN origin)
   ├── Caractéristiques:
   │   ├── Popular content = 5% of total
   │   ├── Long tail content = 95%
   │   ├── 90% requests for popular content
   │   └── Pattern: 95/5
   ├── Configuration recommandée:
   │   ├── Total: 500GB
   │   ├── RAM: 75GB (15%)
   │   └── Flash: 425GB (85%)
   └── Économies: 85% vs full RAM

5. User Profiles (social network)
   ├── Caractéristiques:
   │   ├── Active users = 30% daily
   │   ├── Dormant users = 70%
   │   ├── Access follows power law
   │   └── Pattern: 70/30
   ├── Configuration recommandée:
   │   ├── Total: 1TB
   │   ├── RAM: 350GB (35%)
   │   └── Flash: 650GB (65%)
   └── Économies: 77% vs full RAM
```

### Workloads inappropriés

```yaml
❌ Mauvais cas d'usage pour RAM+Flash:

1. Real-time Trading/Bidding
   ├── Raison: Latence P99 critique (<1ms required)
   ├── Flash reads (1-5ms) unacceptable
   └── Solution: Full RAM only

2. High-Frequency Gaming Leaderboards
   ├── Raison: Toutes les données accessed fréquemment
   ├── Pattern: 100% hot data (no cold data)
   └── Solution: Full RAM + more shards

3. Rate Limiting (high-throughput)
   ├── Raison: Compteurs accessed every request
   ├── Flash would be constant cache miss
   └── Solution: Full RAM, small dataset

4. Cache de requêtes SQL (uniform access)
   ├── Raison: Pas de hot/cold pattern clair
   ├── LRU eviction plus adapté
   └── Solution: Full RAM with eviction policy

5. Streaming Analytics (real-time)
   ├── Raison: All recent data is hot
   ├── Flash tiering n'apporte rien
   └── Solution: Full RAM + sliding window
```

### Pattern de migration vers le tiering

```yaml
Migration Strategy: Full RAM → RAM+Flash

Phase 1: Analyse (2 semaines)
├── Monitor access patterns
│   ├── Identify hot vs cold data ratio
│   ├── Measure P50/P99 latencies baseline
│   └── Track memory usage trends
├── Calculate potential savings
└── Estimate performance impact

Phase 2: Test en staging (2 semaines)
├── Deploy tiered instance
├── Replicate production traffic (shadowing)
├── Compare latencies
│   ├── Acceptable if P99 increase <20%
│   └── RAM hit rate should be >70%
└── Validate cost savings

Phase 3: Gradual migration (4 semaines)
├── Week 1: 25% traffic to tiered instance
│   └── Monitor closely
├── Week 2: 50% traffic
│   └── Compare metrics
├── Week 3: 75% traffic
│   └── Verify cost savings
└── Week 4: 100% traffic
    └── Decommission old instance

Phase 4: Optimization (ongoing)
├── Adjust RAM/Flash ratio based on hit rate
├── Monitor performance regressions
└── Fine-tune access patterns
```

---

## 📊 Comparaison avec alternatives

### Tiering vs Eviction policies

```yaml
┌────────────────────────────────────────────────────────────────┐
│         RAM+Flash Tiering vs Eviction Policies                 │
└────────────────────────────────────────────────────────────────┘

Scenario: 500GB dataset, budget for 100GB RAM

Option 1: Full RAM (100GB) + Eviction (allkeys-lru)
├── Cost: 100GB × $87 = $8,700/month
├── Pros:
│   ├── Consistent low latency (<1ms)
│   ├── Simple configuration
│   └── Predictable performance
├── Cons:
│   ├── Only 20% of data available
│   ├── 80% of data lost (cache misses → DB)
│   ├── High cache miss rate (if uniform access)
│   └── Cannot query cold data
└── Use when: Hot set is well-defined and small

Option 2: RAM+Flash Tiering (100GB RAM + 400GB Flash)
├── Cost: (100×$35)+(400×$9) = $7,100/month
├── Pros:
│   ├── 100% of data queryable
│   ├── Hot data in RAM (<1ms)
│   ├── Cold data accessible (2-5ms)
│   └── 18% cheaper than full RAM
├── Cons:
│   ├── Variable latency (depends on tier)
│   ├── Requires access pattern analysis
│   └── Slightly more complex
└── Use when: Need full dataset with cost optimization

Option 3: Full RAM (500GB)
├── Cost: 500GB × $87 = $43,500/month
├── Pros:
│   ├── Consistently low latency
│   ├── Simple, predictable
│   └── Best performance
├── Cons:
│   └── 6× more expensive than tiering
└── Use when: Budget is not a constraint

Recommendation by use case:
├── Latency-critical (trading): Full RAM
├── Cost-critical (startup): RAM+eviction or tiering
├── Full dataset needed (analytics): Tiering
└── Hot set defined (sessions): Either, analyze cost/perf
```

### Tiering vs Sharding

```yaml
Another approach: Horizontal scaling with more shards

Scenario: 500GB dataset, need better performance

Option A: Tiering (single cluster)
├── 100GB RAM + 400GB Flash
├── Cost: $7,100/month
├── Latency: 0.8ms avg (2ms P99)
└── Throughput: 80K ops/sec

Option B: More shards (full RAM, distributed)
├── 5 shards × 100GB RAM = 500GB total
├── Cost: 5 × $8,700 = $43,500/month
├── Latency: 0.5ms avg (1ms P99)
└── Throughput: 500K ops/sec (5× better)

Decision Matrix:
┌──────────────────────────────────────────────────────────┐
│ Priority     │ Best Solution        │ Trade-off          │
├──────────────────────────────────────────────────────────┤
│ Cost         │ Tiering              │ +latency           │
│ Latency      │ More shards          │ +cost              │
│ Throughput   │ More shards          │ +cost              │
│ Simplicity   │ Single shard + tier  │ +latency           │
│ Full dataset │ Tiering              │ +latency for cold  │
└──────────────────────────────────────────────────────────┘

Hybrid approach (best of both):
├── 3 shards with tiering
│   ├── Each: 33GB RAM + 133GB Flash = 166GB total
│   ├── Total: 100GB RAM + 400GB Flash = 500GB
│   └── Cost: 3 × $2,367 = $7,100/month (same as single)
├── Benefits:
│   ├── 3× throughput vs single shard
│   ├── Better fault tolerance
│   └── Same cost as single tiered shard
└── Use for: High-throughput + large datasets
```

---

## 🎯 Recommandations et décision

### Arbre de décision

```
Est-ce que votre dataset est >200GB ?
├─ NON → Utilisez full RAM (coût acceptable)
└─ OUI → Continuez ↓

Avez-vous un pattern hot/cold clair (>70/30) ?
├─ NON → Utilisez full RAM + sharding
└─ OUI → Continuez ↓

Votre application tolère-t-elle +1-2ms de latence sur 20-30% des requêtes ?
├─ NON → Utilisez full RAM
└─ OUI → Continuez ↓

Votre budget est-il contraint ?
├─ OUI → Utilisez RAM+Flash tiering ✅
└─ NON → Utilisez full RAM pour performance maximale

Cas spécial: Dataset >1TB avec pattern hot/cold
└─ RAM+Flash tiering est fortement recommandé
   (économies de 80-85% vs full RAM)
```

### Checklist de décision

```yaml
☐ Analysez votre pattern d'accès (hot/cold ratio)
   └─ Méthode: Redis OBJECT IDLETIME pour échantillon de clés

☐ Mesurez votre latence P99 actuelle (baseline)
   └─ Acceptable degradation: <20% increase

☐ Calculez les économies potentielles
   └─ Use case typique: 80-85% savings vs full RAM

☐ Validez que votre provider supporte le tiering
   ├─ Redis Enterprise Cloud: ✅ Oui
   ├─ Azure Cache Enterprise: ✅ Oui (preview)
   ├─ AWS ElastiCache: ❌ Non
   ├─ AWS MemoryDB: ❌ Non
   └─ GCP Memorystore: ❌ Non

☐ Estimez la RAM allocation optimale
   └─ Target: 70-80% RAM hit rate minimum

☐ Testez en staging avec traffic production
   └─ Durée minimum: 2 semaines

☐ Configurez le monitoring du tiering
   ├─ RAM hit rate
   ├─ Flash read rate
   ├─ Promotion/demotion rate
   └─ P99 latency

☐ Planifiez la migration progressive (25%→50%→75%→100%)

☐ Documentez les économies et impact performance
```

### Calcul rapide

```python
def quick_tiering_decision(dataset_gb: int, budget_monthly: int) -> dict:
    """Quick decision: Should you use tiering?"""

    full_ram_cost = dataset_gb * 87  # $87/GB-month

    if budget_monthly >= full_ram_cost:
        return {
            'recommendation': 'Full RAM',
            'reason': 'Budget permits full RAM - best performance',
            'cost': full_ram_cost,
            'savings': 0
        }

    # Calculate required RAM ratio for budget
    # total_cost = (ram_gb × $35) + (flash_gb × $9)
    # Solve for ram_gb given budget

    # Try 20% RAM / 80% Flash (typical)
    ram_gb = dataset_gb * 0.2
    flash_gb = dataset_gb * 0.8
    tiering_cost = (ram_gb * 35) + (flash_gb * 9)

    if budget_monthly >= tiering_cost:
        savings = full_ram_cost - tiering_cost
        savings_pct = (savings / full_ram_cost) * 100

        return {
            'recommendation': 'RAM+Flash Tiering',
            'reason': f'Budget-appropriate, {savings_pct:.0f}% savings',
            'ram_gb': ram_gb,
            'flash_gb': flash_gb,
            'cost': tiering_cost,
            'savings': savings,
            'ram_percent': 20
        }
    else:
        # Budget too low even for tiering
        affordable_ram = budget_monthly / 35

        return {
            'recommendation': 'Reduce dataset or increase budget',
            'reason': 'Budget insufficient for dataset size',
            'affordable_ram_gb': affordable_ram,
            'required_budget_tiering': tiering_cost,
            'required_budget_full_ram': full_ram_cost
        }

# Examples
print("=== Dataset: 500GB, Budget: $50,000/month ===")
result = quick_tiering_decision(500, 50000)
print(f"Recommendation: {result['recommendation']}")
print(f"Reason: {result['reason']}")

print("\n=== Dataset: 500GB, Budget: $10,000/month ===")
result = quick_tiering_decision(500, 10000)
print(f"Recommendation: {result['recommendation']}")
if 'savings' in result:
    print(f"Cost: ${result['cost']:,}/month")
    print(f"Savings: ${result['savings']:,}/month")

print("\n=== Dataset: 500GB, Budget: $5,000/month ===")
result = quick_tiering_decision(500, 5000)
print(f"Recommendation: {result['recommendation']}")
print(f"Reason: {result['reason']}")
```

---

## ✅ Conclusion

### Points clés à retenir

1. **Économies massives possibles**
   - 80-85% de réduction de coût vs full RAM
   - Viable uniquement avec pattern hot/cold clair (>70/30)

2. **Trade-off performance accepté**
   - Latence Flash: +1-4ms vs RAM
   - Impact global: +20-30% sur P99 (acceptable pour beaucoup d'apps)
   - RAM hit rate >70% essentiel

3. **Cas d'usage idéaux**
   - Large datasets (>500GB)
   - Pattern Pareto 80/20
   - Budget contraint
   - Latency requirements: P99 <5ms acceptable

4. **Disponibilité limitée**
   - Redis Enterprise Cloud: ✅
   - Azure Cache Enterprise: ✅ (preview)
   - AWS/GCP: ❌

5. **Migration nécessite analyse**
   - 2-4 semaines de tests staging
   - Monitoring continu du RAM hit rate
   - Ajustements itératifs

### Formule de décision rapide

```
Utilisez RAM+Flash Tiering si:
  (Dataset > 200GB)
  ET
  (Hot/Cold ratio > 70/30)
  ET
  (P99 latency requirement < 5ms)
  ET
  (Budget savings > 70% désirés)

SINON
  Utilisez Full RAM avec sharding
```

### ROI minimum

```yaml
Pour que le tiering soit rentable:
├── Dataset minimum: 200GB
│   └── En dessous: coût full RAM acceptable
├── Économies minimum: $5,000/month
│   └── Justifie la complexité additionnelle
└── Pattern d'accès: >70% hot
    └── Sinon RAM hit rate trop faible
```

---

**🎯 Section suivante :** Nous allons maintenant explorer les déploiements Docker et Docker Compose dans la section 15.7.

⏭️ [Redis avec Docker et Docker Compose](/15-redis-cloud-conteneurs/07-redis-docker-docker-compose.md)
