🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #8 : IoT et Time-Series avec RedisTimeSeries

## Vue d'ensemble

**Niveau** : ⭐⭐⭐ Avancé
**Complexité technique** : Élevée
**Impact production** : Critique (operational intelligence)
**Technologies** : Redis Stack (RedisTimeSeries) + IoT protocols

---

## 1. Contexte et problématique

### Scénario business

Une plateforme de monitoring industriel IoT (type ThingWorx, PTC, ou Siemens MindSphere) pour Smart Factory :

**Chiffres clés** :
- 100 000 capteurs IoT déployés (température, pression, vibration, énergie)
- 1 million de data points par seconde
- 86 milliards de data points par jour
- 50 usines connectées à travers le monde
- SLA : Latence ingestion < 100ms, Query < 500ms
- Retention : 1s granularity (1 semaine), 1min (1 an), 1h (5 ans)
- Coût actuel : $50k/mois → Objectif : $15k/mois

**Besoins métier critiques** :

1. **Ingestion haute fréquence**
   - 1M data points/sec en continu
   - Pics à 5M data points/sec
   - Latence < 100ms (sensor → database)
   - Pas de perte de données

2. **Downsampling automatique**
   - Raw data (1s) : 1 semaine
   - Minute aggregates (avg, max, min) : 1 an
   - Hour aggregates : 5 ans
   - Day aggregates : illimité
   - Automatic compaction en background

3. **Requêtes analytics temps réel**
   - Dashboard : 100 métriques en < 500ms
   - Time range queries : "Last 24h", "Last 30 days"
   - Aggregations : avg, sum, min, max, stddev, percentiles
   - Multi-sensor queries : "All temp sensors in zone A"

4. **Alerting et anomaly detection**
   - Seuils statiques : temp > 80°C
   - Seuils dynamiques : +20% vs moyenne 1h
   - Anomaly detection : ML-based (isolation forest)
   - Alert latency < 5 secondes

### Problèmes à résoudre

#### 1. **Volume et vélocité des données**

```
Challenge : 1M data points/sec = 86B/jour = 31 Trillion/an

Storage naïf (PostgreSQL) :
- Size: 31T × 50 bytes = 1.5 PB/an (prohibitif!)
- Write throughput: 1M inserts/sec (impossible sans sharding massif)
- Query: Full table scan sur milliards de rows (minutes)

Solution : Time-series optimized DB
- Compression: 10:1 ratio → 150 TB/an
- Write optimized: Sequential writes, WAL
- Query optimized: Indexé par temps
```

#### 2. **Retention policies et storage tiers**

```
Problème : Balance entre précision et coût

Stratégie multi-tier :
┌─────────────┬────────────┬──────────┬─────────┐
│ Granularity │ Retention  │ Storage  │ Cost    │
├─────────────┼────────────┼──────────┼─────────┤
│ 1 second    │ 7 days     │ Hot SSD  │ High    │
│ 1 minute    │ 1 year     │ Warm HDD │ Medium  │
│ 1 hour      │ 5 years    │ Cold S3  │ Low     │
│ 1 day       │ Forever    │ Archive  │ Minimal │
└─────────────┴────────────┴──────────┴─────────┘

Auto-downsampling :
raw_1s → minute_avg → hour_avg → day_avg
         (automatic compaction rules)
```

#### 3. **Query performance sur large time ranges**

```
Problème : Query "average temperature last 30 days"

Approche naïve :
SELECT AVG(value) FROM sensor_data
WHERE sensor_id = 'temp_001'
    AND timestamp >= NOW() - INTERVAL '30 days'
→ Scan 30 × 86,400 = 2,592,000 rows
→ Latency : 5-30 seconds

Solution : Pre-aggregated data
Query minute_avg (43,200 rows au lieu de 2.6M)
→ Latency : < 100ms
```

#### 4. **Multi-sensor aggregations**

```
Besoin : "Average temperature across all 500 sensors in zone A"

Challenge :
- 500 sensors × 86,400 points/day = 43.2M data points
- Cross-sensor aggregation

Solution : Hierarchical aggregation
sensor_001 → avg → ┐
sensor_002 → avg → ├─→ zone_A_avg
    ...            │
sensor_500 → avg → ┘
```

---

## 2. Analyse des alternatives

### Option 1 : PostgreSQL avec TimescaleDB

```sql
-- Extension TimescaleDB
CREATE EXTENSION timescaledb;

-- Hypertable (automatic partitioning)
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    sensor_id TEXT NOT NULL,
    value DOUBLE PRECISION,
    metadata JSONB
);

SELECT create_hypertable('sensor_data', 'time');

-- Continuous aggregates (auto-downsampling)
CREATE MATERIALIZED VIEW sensor_data_1min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    sensor_id,
    AVG(value) AS avg_value,
    MAX(value) AS max_value,
    MIN(value) AS min_value
FROM sensor_data
GROUP BY bucket, sensor_id;

-- Retention policy
SELECT add_retention_policy('sensor_data', INTERVAL '7 days');
```

**Avantages** :
- ✅ SQL familier
- ✅ Continuous aggregates (auto-downsampling)
- ✅ Retention policies automatiques
- ✅ ACID compliance

**Inconvénients** :
- ❌ Latence : 50-500ms queries (index overhead)
- ❌ Write throughput : 100k-500k/sec (limite)
- ❌ Scaling horizontal complexe
- ❌ Coût : $500-2000/mois pour 1M points/sec

**Benchmark** (TimescaleDB 2.11, 1M data points/sec) :
```
Opération                    Latence
----------------------------------------
Write (batch 1000)          50-200ms
Query (1h range)            100-300ms
Query (30d range)           500-2000ms
Continuous aggregate        Background (5-30 min lag)
```

**Verdict** : ⚠️ **Solide mais coûteux** à grande échelle.

---

### Option 2 : InfluxDB

```python
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

client = InfluxDBClient(url="http://localhost:8086", token="...")
write_api = client.write_api(write_options=SYNCHRONOUS)

# Write
point = Point("temperature") \
    .tag("sensor_id", "temp_001") \
    .tag("zone", "A") \
    .field("value", 25.5)

write_api.write(bucket="sensors", record=point)

# Query (Flux language)
query = '''
from(bucket: "sensors")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "temperature")
  |> aggregateWindow(every: 1m, fn: mean)
'''

result = client.query_api().query(query)
```

**Avantages** :
- ✅ Time-series native
- ✅ High write throughput (500k-1M/sec)
- ✅ Automatic downsampling (tasks)
- ✅ Flux query language (puissant)

**Inconvénients** :
- ⚠️ Latence : 10-100ms queries
- ❌ Coût : $500-5000/mois (InfluxDB Cloud)
- ❌ Learning curve (Flux)
- ⚠️ Scaling : Cluster setup complexe

**Benchmark** (InfluxDB 2.7, 1M points/sec) :
```
Opération                    Latence
----------------------------------------
Write (batch 5000)          20-100ms
Query (1h range, 1 sensor)  10-50ms
Query (24h range, 100 sensors) 100-500ms
Downsampling                Background (1-10 min lag)
```

**Verdict** : ✅ **Excellent mais coûteux** pour très grande échelle.

---

### Option 3 : RedisTimeSeries ✅

```bash
# Create time-series
TS.CREATE sensor:temp_001:raw
    RETENTION 604800000  # 7 days in ms
    DUPLICATE_POLICY LAST
    LABELS sensor_id temp_001 zone A type temperature

# Create downsampling rules (automatic)
TS.CREATERULE sensor:temp_001:raw sensor:temp_001:1min
    AGGREGATION avg 60000  # 1 min in ms

TS.CREATERULE sensor:temp_001:1min sensor:temp_001:1hour
    AGGREGATION avg 3600000  # 1 hour in ms

# Add data point
TS.ADD sensor:temp_001:raw * 25.5

# Query
TS.RANGE sensor:temp_001:raw - + AGGREGATION avg 60000
```

**Avantages** :
- ✅ Latence : **< 5ms** queries (in-memory)
- ✅ Write throughput : **1M-5M/sec** per instance
- ✅ Automatic downsampling (real-time)
- ✅ Native aggregations (avg, sum, min, max, etc.)
- ✅ Coût : **$200-500/mois** pour 1M points/sec
- ✅ Simple à opérer

**Inconvénients** :
- ⚠️ In-memory : RAM required (mais tiering disponible)
- ⚠️ Query language moins riche que Flux
- ⚠️ Pas d'ACID (mais acceptable pour IoT)

**Benchmark** (Redis Stack 7.2, 1M points/sec) :
```
Opération                         Latency
--------------------------------------------
Write (TS.ADD)                   0.1-1ms
Write (batch pipeline)           0.05ms per point
Query (1h range, 1 sensor)       2-10ms
Query (24h range, 100 sensors)   50-200ms
Multi-sensor aggregation         10-100ms
Downsampling                     Real-time (< 1s lag)
Memory                           ~50 bytes per data point
```

**Trade-off assumé** :
- ➕ Performance × 100 (vs TimescaleDB)
- ➕ Coût ÷ 10 (vs InfluxDB Cloud)
- ➕ Latence ultra-faible
- ➕ Downsampling en temps réel
- ➖ RAM required (mais gérable avec tiering)
- ➖ Query language moins avancé

**Verdict** : ✅ **Solution optimale** pour IoT haute fréquence.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────┐
│                    IoT Devices / Sensors                    │
│  100,000 sensors (temp, pressure, vibration, energy)        │
│  - Sending data every 1 second                              │
│  - Protocol: MQTT, HTTP, CoAP                               │
└──────────┬──────────────────────────────────────────────────┘
           │ MQTT / HTTP
           ▼
┌─────────────────────────────────────────────────────────────┐
│              IoT Gateway / Edge Processing                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Data validation                                  │    │
│  │  - Preprocessing (unit conversion, filtering)       │    │
│  │  - Batching (100 points per batch)                  │    │
│  │  - Compression                                      │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │ Batch HTTP
           ▼
┌─────────────────────────────────────────────────────────────┐
│           Ingestion API (Load Balanced)                     │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│  │  Ingest 1  │   │  Ingest 2  │   │  Ingest N  │           │
│  │  Service   │   │  Service   │   │  Service   │           │
│  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘           │
└────────┼────────────────┼────────────────┼──────────────────┘
         │                │                │
         └────────────────┼────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│           RedisTimeSeries Cluster                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Time-Series Structure (per sensor):                 │   │
│  │                                                      │   │
│  │  sensor:temp_001:raw                                 │   │
│  │  ├─ Retention: 7 days                                │   │
│  │  ├─ Labels: {sensor_id, zone, type}                  │   │
│  │  └─ Compaction rules:                                │   │
│  │      ├─> sensor:temp_001:1min (avg, 1 year)          │   │
│  │      │    └─> sensor:temp_001:1hour (avg, 5 years)   │   │
│  │      │         └─> sensor:temp_001:1day (avg, ∞)     │   │
│  │      ├─> sensor:temp_001:1min_max (max, 1 year)      │   │
│  │      └─> sensor:temp_001:1min_min (min, 1 year)      │   │
│  │                                                      │   │
│  │  Multi-level aggregation (automatic):                │   │
│  │  raw (1s) → 1min → 1hour → 1day                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Cluster Configuration:                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ Shard 1  │  │ Shard 2  │  │ Shard N  │                   │
│  │ Master   │  │ Master   │  │ Master   │                   │
│  │ Replica  │  │ Replica  │  │ Replica  │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
│                                                             │
│  Memory tiering (optional):                                 │
│  - Hot data (7 days): RAM                                   │
│  - Warm data (1 year): Redis on Flash                       │
│  - Cold data (5 years): S3 export                           │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Analytics & Visualization Layer                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Real-time dashboards (Grafana)                   │    │
│  │  - Alerting engine                                  │    │
│  │  - Anomaly detection (ML)                           │    │
│  │  - Predictive maintenance                           │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Flux d'ingestion

#### **Write Path (1M points/sec)**

```
1. Sensor sends data point
   {
       "sensor_id": "temp_001",
       "timestamp": 1702300800000,
       "value": 25.5,
       "unit": "celsius"
   }

2. IoT Gateway batches data
   Batch size: 100 points
   Frequency: Every 100ms or when full
   → 10,000 batches/sec

3. Ingestion API receives batch
   POST /api/ingest/batch
   Body: [100 data points]

4. Write to Redis (pipeline)
   pipe = redis.pipeline()
   for point in batch:
       pipe.ts().add(
           f"sensor:{point.sensor_id}:raw",
           point.timestamp,
           point.value
       )
   pipe.execute()

   Latency per batch: ~5ms
   → 5ms / 100 = 0.05ms per point

5. Automatic compaction (background)
   RedisTimeSeries auto-compacts to:
   ├─ sensor:temp_001:1min (avg)
   ├─ sensor:temp_001:1min_max (max)
   ├─ sensor:temp_001:1min_min (min)
   └─ sensor:temp_001:1hour → 1day

   Lag: < 1 second (real-time)

Total latency: Sensor → Queryable = ~100ms
```

#### **Query Path (Dashboard)**

```
User requests: "Temperature trend last 24 hours for sensor temp_001"

1. API receives request
   GET /api/sensors/temp_001/data?range=24h&aggregation=1min

2. Determine optimal granularity
   Range: 24h = 1,440 minutes
   → Use 1min aggregates (1,440 points)

   Alternative: Use raw data (86,400 points) = 60× more data

3. Query RedisTimeSeries
   TS.RANGE sensor:temp_001:1min
       (now - 24h) now
       AGGREGATION avg 60000

4. Response
   [
       [1702300800000, 25.5],
       [1702300860000, 25.6],
       ...
   ]

   Data size: 1,440 points × 16 bytes = 23 KB
   Latency: ~10ms

5. Optional: Multi-sensor query
   TS.MRANGE (now - 24h) now
       AGGREGATION avg 60000
       FILTER zone=A type=temperature

   Returns: 100 sensors × 1,440 points
   Latency: ~100ms
```

### 3.3 Décisions architecturales clés

#### **Choix 1 : Multi-level compaction strategy**

```
Strategy: 4-level hierarchy

Level 1 (raw): 1 second granularity
├─ Retention: 7 days
├─ Storage: In-memory (hot)
└─ Use case: Real-time monitoring, debugging

Level 2 (1min aggregates): avg, max, min
├─ Retention: 1 year
├─ Storage: In-memory or Redis on Flash
├─ Automatic from raw (TS.CREATERULE)
└─ Use case: Dashboard trends, alerts

Level 3 (1hour aggregates): avg, max, min
├─ Retention: 5 years
├─ Storage: Redis on Flash or periodic export to S3
├─ Automatic from 1min
└─ Use case: Historical analysis, reporting

Level 4 (1day aggregates): avg, max, min, sum
├─ Retention: Forever
├─ Storage: PostgreSQL or S3
├─ Manual export (daily cron)
└─ Use case: Long-term trends, compliance

Compaction latency: < 1 second (real-time)
```

**Trade-off assumé** :
- ➕ Query speed × 60-86,400 (vs raw)
- ➕ Storage ÷ 60-86,400
- ➖ Approximation (mais avg/max/min préservés)

---

#### **Choix 2 : Labeling strategy pour multi-sensor queries**

```bash
# Hierarchical labels
TS.CREATE sensor:temp_001:raw
    LABELS
        sensor_id temp_001
        zone A
        building Factory_1
        type temperature
        unit celsius
        criticality high

# Permet queries flexibles:

# 1. All sensors in zone A
TS.MRANGE - + FILTER zone=A

# 2. All temperature sensors
TS.MRANGE - + FILTER type=temperature

# 3. High criticality in building Factory_1
TS.MRANGE - + FILTER building=Factory_1 criticality=high

# 4. Combination
TS.MRANGE - + FILTER zone=A type=temperature
```

**Trade-off assumé** :
- ➕ Flexibilité maximale pour analytics
- ➕ No schema changes needed
- ➖ Label storage overhead (~50 bytes per series)

---

#### **Choix 3 : Batching strategy**

**Naïf (1 point à la fois)** :
```python
# ❌ 1M round-trips/sec (impossible)
for point in data_points:
    redis.ts().add(key, timestamp, value)

Latency: 1M × 1ms = 1000 seconds!
```

**Optimal (batching avec pipeline)** :
```python
# ✅ 10k round-trips/sec (feasible)
def ingest_batch(points, batch_size=100):
    pipe = redis.pipeline()

    for point in points:
        pipe.ts().add(
            f"sensor:{point.sensor_id}:raw",
            point.timestamp,
            point.value
        )

        if len(pipe) >= batch_size:
            pipe.execute()
            pipe = redis.pipeline()

    if len(pipe) > 0:
        pipe.execute()

# 1M points / 100 per batch = 10k batches
# 10k × 5ms = 50 seconds (vs 1000s)
```

**Trade-off assumé** :
- ➕ Throughput × 100
- ➕ Latency acceptable (~100ms end-to-end)
- ➖ Slightly delayed ingestion (batching delay)

---

## 4. Implémentation technique

### 4.1 Code Python (Production-Ready)

```python
"""
IoT Time-Series Platform avec RedisTimeSeries
Implémentation production-ready
"""

import time
from typing import List, Dict, Any, Optional, Tuple
from dataclasses import dataclass, field
from datetime import datetime, timedelta
import logging
from enum import Enum

import redis
from redis.commands.timeseries.commands import TimeSeriesCommands

# Configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'db': 0,
    'decode_responses': False
}

# Constants
BATCH_SIZE = 100
RETENTION_POLICIES = {
    'raw': 7 * 24 * 3600 * 1000,      # 7 days in ms
    '1min': 365 * 24 * 3600 * 1000,   # 1 year in ms
    '1hour': 5 * 365 * 24 * 3600 * 1000,  # 5 years in ms
}


# ============================================================================
# Data Classes
# ============================================================================

class AggregationType(Enum):
    AVG = "avg"
    SUM = "sum"
    MIN = "min"
    MAX = "max"
    RANGE = "range"
    COUNT = "count"
    FIRST = "first"
    LAST = "last"
    STD_P = "std.p"  # Standard deviation (population)
    STD_S = "std.s"  # Standard deviation (sample)


@dataclass
class SensorDataPoint:
    """Point de données d'un capteur"""
    sensor_id: str
    timestamp: int  # Unix timestamp in milliseconds
    value: float
    unit: Optional[str] = None
    metadata: Dict[str, Any] = field(default_factory=dict)


@dataclass
class TimeSeriesConfig:
    """Configuration d'une time-series"""
    sensor_id: str
    retention_ms: int
    labels: Dict[str, str]
    duplicate_policy: str = "LAST"  # LAST, FIRST, MIN, MAX, SUM


@dataclass
class CompactionRule:
    """Règle de compaction"""
    source_key: str
    dest_key: str
    aggregation: AggregationType
    bucket_size_ms: int


# ============================================================================
# IoT Time-Series Manager
# ============================================================================

class IoTTimeSeriesManager:
    """
    Gestionnaire de time-series IoT avec RedisTimeSeries

    Features:
    - High-frequency ingestion (1M points/sec)
    - Automatic downsampling avec compaction rules
    - Multi-level aggregations
    - Flexible querying
    - Alerting support
    """

    def __init__(self, redis_config: Dict = None):
        redis_conf = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**redis_conf)
        self.ts: TimeSeriesCommands = self.redis.ts()

        # Test connexion
        try:
            self.redis.ping()
            logger.info("IoTTimeSeriesManager initialized")
        except redis.RedisError as e:
            logger.error(f"Redis connection failed: {e}")
            raise

    # ========================================================================
    # Time-Series Creation
    # ========================================================================

    def create_sensor_timeseries(self, config: TimeSeriesConfig) -> bool:
        """
        Créer une time-series pour un capteur

        Args:
            config: Configuration de la time-series

        Returns:
            True si créée, False si existe déjà
        """
        key = f"sensor:{config.sensor_id}:raw"

        try:
            # Check if exists
            try:
                self.ts.info(key)
                logger.debug(f"Time-series already exists: {key}")
                return False
            except:
                pass

            # Create time-series
            self.ts.create(
                key,
                retention_msecs=config.retention_ms,
                duplicate_policy=config.duplicate_policy,
                labels=config.labels
            )

            logger.info(f"Time-series created: {key}")

            # Create compaction rules
            self._create_compaction_rules(config.sensor_id)

            return True

        except redis.RedisError as e:
            logger.error(f"Failed to create time-series: {e}")
            return False

    def _create_compaction_rules(self, sensor_id: str):
        """
        Créer règles de compaction automatique

        Hierarchy:
        raw (1s) → 1min (avg/max/min) → 1hour (avg/max/min) → 1day (avg/max/min)
        """
        raw_key = f"sensor:{sensor_id}:raw"

        # Level 1: raw → 1min aggregates
        aggregations_1min = [
            (AggregationType.AVG, "1min"),
            (AggregationType.MAX, "1min_max"),
            (AggregationType.MIN, "1min_min"),
            (AggregationType.SUM, "1min_sum"),
        ]

        for agg_type, suffix in aggregations_1min:
            dest_key = f"sensor:{sensor_id}:{suffix}"

            # Create destination time-series
            try:
                self.ts.create(
                    dest_key,
                    retention_msecs=RETENTION_POLICIES['1min'],
                    labels={"sensor_id": sensor_id, "aggregation": agg_type.value, "interval": "1min"}
                )
            except:
                pass  # Already exists

            # Create compaction rule
            try:
                self.ts.createrule(
                    raw_key,
                    dest_key,
                    aggregation_type=agg_type.value,
                    bucket_size_msec=60000  # 1 minute
                )
                logger.debug(f"Compaction rule created: {raw_key} → {dest_key} ({agg_type.value})")
            except Exception as e:
                logger.warning(f"Failed to create rule: {e}")

        # Level 2: 1min → 1hour aggregates
        min_key = f"sensor:{sensor_id}:1min"
        aggregations_1hour = [
            (AggregationType.AVG, "1hour"),
            (AggregationType.MAX, "1hour_max"),
            (AggregationType.MIN, "1hour_min"),
        ]

        for agg_type, suffix in aggregations_1hour:
            dest_key = f"sensor:{sensor_id}:{suffix}"

            try:
                self.ts.create(
                    dest_key,
                    retention_msecs=RETENTION_POLICIES['1hour'],
                    labels={"sensor_id": sensor_id, "aggregation": agg_type.value, "interval": "1hour"}
                )
            except:
                pass

            try:
                self.ts.createrule(
                    min_key,
                    dest_key,
                    aggregation_type=agg_type.value,
                    bucket_size_msec=3600000  # 1 hour
                )
            except Exception as e:
                logger.warning(f"Failed to create rule: {e}")

        # Level 3: 1hour → 1day aggregates
        hour_key = f"sensor:{sensor_id}:1hour"

        day_key = f"sensor:{sensor_id}:1day"
        try:
            self.ts.create(
                day_key,
                retention_msecs=0,  # Forever
                labels={"sensor_id": sensor_id, "aggregation": "avg", "interval": "1day"}
            )
        except:
            pass

        try:
            self.ts.createrule(
                hour_key,
                day_key,
                aggregation_type="avg",
                bucket_size_msec=86400000  # 1 day
            )
        except Exception as e:
            logger.warning(f"Failed to create rule: {e}")

    # ========================================================================
    # Data Ingestion
    # ========================================================================

    def add_datapoint(
        self,
        sensor_id: str,
        value: float,
        timestamp: Optional[int] = None
    ) -> int:
        """
        Ajouter un point de données (usage simple)

        Args:
            sensor_id: ID du capteur
            value: Valeur mesurée
            timestamp: Timestamp en ms (None = now)

        Returns:
            Timestamp du point ajouté
        """
        key = f"sensor:{sensor_id}:raw"

        try:
            ts = self.ts.add(key, timestamp or "*", value)
            return ts
        except redis.RedisError as e:
            logger.error(f"Failed to add datapoint: {e}")
            raise

    def ingest_batch(
        self,
        datapoints: List[SensorDataPoint],
        batch_size: int = BATCH_SIZE
    ) -> Dict[str, int]:
        """
        Ingestion batch optimisée (usage production)

        Args:
            datapoints: Liste de points à ingérer
            batch_size: Taille des batches pour pipeline

        Returns:
            Stats: {"success": N, "failed": M, "duration_ms": X}
        """
        start_time = time.time()
        success = 0
        failed = 0

        pipe = self.redis.pipeline()
        batch_count = 0

        for dp in datapoints:
            key = f"sensor:{dp.sensor_id}:raw"

            try:
                pipe.ts().add(key, dp.timestamp, dp.value)
                batch_count += 1

                # Execute batch when full
                if batch_count >= batch_size:
                    results = pipe.execute()
                    success += len([r for r in results if r])
                    failed += len([r for r in results if not r])

                    pipe = self.redis.pipeline()
                    batch_count = 0

            except Exception as e:
                logger.error(f"Failed to add to pipeline: {e}")
                failed += 1

        # Execute remaining
        if batch_count > 0:
            results = pipe.execute()
            success += len([r for r in results if r])
            failed += len([r for r in results if not r])

        duration_ms = (time.time() - start_time) * 1000

        logger.info(
            f"Batch ingested: {success} success, {failed} failed, "
            f"{duration_ms:.2f}ms ({success/duration_ms*1000:.0f} points/sec)"
        )

        return {
            "success": success,
            "failed": failed,
            "duration_ms": duration_ms
        }

    # ========================================================================
    # Querying
    # ========================================================================

    def get_range(
        self,
        sensor_id: str,
        from_time: int,
        to_time: int,
        aggregation: Optional[AggregationType] = None,
        bucket_size_ms: Optional[int] = None
    ) -> List[Tuple[int, float]]:
        """
        Récupérer données pour une plage temporelle

        Args:
            sensor_id: ID du capteur
            from_time: Timestamp début (ms)
            to_time: Timestamp fin (ms)
            aggregation: Type d'agrégation (None = raw)
            bucket_size_ms: Taille bucket pour aggregation

        Returns:
            Liste de (timestamp, value)
        """
        # Auto-select optimal granularity
        key = self._select_optimal_key(sensor_id, from_time, to_time)

        try:
            if aggregation and bucket_size_ms:
                result = self.ts.range(
                    key,
                    from_time,
                    to_time,
                    aggregation_type=aggregation.value,
                    bucket_size_msec=bucket_size_ms
                )
            else:
                result = self.ts.range(key, from_time, to_time)

            return [(int(ts), float(val)) for ts, val in result]

        except redis.RedisError as e:
            logger.error(f"Query failed: {e}")
            return []

    def _select_optimal_key(
        self,
        sensor_id: str,
        from_time: int,
        to_time: int
    ) -> str:
        """
        Sélectionner clé optimale selon la plage temporelle

        Strategy:
        - < 1 hour: Use raw (1s)
        - 1 hour - 7 days: Use 1min aggregates
        - 7 days - 1 year: Use 1hour aggregates
        - > 1 year: Use 1day aggregates
        """
        duration_ms = to_time - from_time
        duration_hours = duration_ms / (3600 * 1000)

        if duration_hours < 1:
            return f"sensor:{sensor_id}:raw"
        elif duration_hours <= 168:  # 7 days
            return f"sensor:{sensor_id}:1min"
        elif duration_hours <= 8760:  # 1 year
            return f"sensor:{sensor_id}:1hour"
        else:
            return f"sensor:{sensor_id}:1day"

    def get_multi_range(
        self,
        filters: Dict[str, str],
        from_time: int,
        to_time: int,
        aggregation: Optional[AggregationType] = None,
        bucket_size_ms: Optional[int] = None
    ) -> Dict[str, List[Tuple[int, float]]]:
        """
        Query multiple sensors avec filtres

        Args:
            filters: Labels filters {"zone": "A", "type": "temperature"}
            from_time: Start timestamp
            to_time: End timestamp
            aggregation: Aggregation type
            bucket_size_ms: Bucket size

        Returns:
            Dict {sensor_id: [(timestamp, value), ...]}
        """
        try:
            if aggregation and bucket_size_ms:
                results = self.ts.mrange(
                    from_time,
                    to_time,
                    filters=filters,
                    aggregation_type=aggregation.value,
                    bucket_size_msec=bucket_size_ms
                )
            else:
                results = self.ts.mrange(from_time, to_time, filters=filters)

            # Parse results
            parsed = {}
            for key, labels, data in results:
                sensor_id = key.decode().split(":")[1] if isinstance(key, bytes) else key.split(":")[1]
                parsed[sensor_id] = [(int(ts), float(val)) for ts, val in data]

            return parsed

        except redis.RedisError as e:
            logger.error(f"Multi-range query failed: {e}")
            return {}

    def get_latest(self, sensor_id: str) -> Optional[Tuple[int, float]]:
        """Récupérer dernière valeur"""
        key = f"sensor:{sensor_id}:raw"

        try:
            result = self.ts.get(key)
            if result:
                return (int(result[0]), float(result[1]))
            return None
        except redis.RedisError as e:
            logger.error(f"Failed to get latest: {e}")
            return None

    # ========================================================================
    # Analytics
    # ========================================================================

    def calculate_statistics(
        self,
        sensor_id: str,
        from_time: int,
        to_time: int
    ) -> Dict[str, float]:
        """
        Calculer statistiques sur une période

        Returns:
            {"avg": X, "min": Y, "max": Z, "count": N, "stddev": S}
        """
        # Use pre-aggregated data if available
        key_min = f"sensor:{sensor_id}:1min"

        try:
            # Get all aggregations
            avg_data = self.ts.range(key_min, from_time, to_time)

            if not avg_data:
                return {}

            values = [float(val) for _, val in avg_data]

            # Calculate statistics
            import statistics

            return {
                "avg": statistics.mean(values),
                "min": min(values),
                "max": max(values),
                "count": len(values),
                "stddev": statistics.stdev(values) if len(values) > 1 else 0.0
            }

        except Exception as e:
            logger.error(f"Statistics calculation failed: {e}")
            return {}

    # ========================================================================
    # Alerting
    # ========================================================================

    def check_threshold_alert(
        self,
        sensor_id: str,
        threshold: float,
        operator: str = ">"
    ) -> bool:
        """
        Vérifier seuil d'alerte

        Args:
            sensor_id: ID du capteur
            threshold: Seuil
            operator: Opérateur (>, <, >=, <=, ==)

        Returns:
            True si alerte déclenchée
        """
        latest = self.get_latest(sensor_id)

        if not latest:
            return False

        _, value = latest

        operators = {
            ">": value > threshold,
            "<": value < threshold,
            ">=": value >= threshold,
            "<=": value <= threshold,
            "==": value == threshold
        }

        alert = operators.get(operator, False)

        if alert:
            logger.warning(
                f"🚨 Alert triggered: {sensor_id} = {value:.2f} "
                f"(threshold: {operator} {threshold})"
            )

        return alert

    def check_anomaly_detection(
        self,
        sensor_id: str,
        window_minutes: int = 60,
        std_threshold: float = 3.0
    ) -> bool:
        """
        Détection d'anomalie simple (Z-score)

        Alert si valeur actuelle > mean + (std_threshold × stddev)
        """
        now = int(time.time() * 1000)
        from_time = now - (window_minutes * 60 * 1000)

        # Get statistics
        stats = self.calculate_statistics(sensor_id, from_time, now)

        if not stats:
            return False

        # Get latest value
        latest = self.get_latest(sensor_id)
        if not latest:
            return False

        _, current_value = latest

        # Z-score
        mean = stats['avg']
        stddev = stats['stddev']

        if stddev == 0:
            return False

        z_score = abs((current_value - mean) / stddev)

        anomaly = z_score > std_threshold

        if anomaly:
            logger.warning(
                f"🚨 Anomaly detected: {sensor_id} = {current_value:.2f} "
                f"(z-score: {z_score:.2f}, threshold: {std_threshold})"
            )

        return anomaly

    # ========================================================================
    # Monitoring
    # ========================================================================

    def get_sensor_info(self, sensor_id: str) -> Dict:
        """Info complète sur un capteur"""
        key = f"sensor:{sensor_id}:raw"

        try:
            info = self.ts.info(key)

            return {
                "total_samples": info.total_samples,
                "memory_bytes": info.memory_usage,
                "first_timestamp": info.first_timestamp,
                "last_timestamp": info.last_timestamp,
                "retention_msecs": info.retention_msecs,
                "chunk_count": info.chunk_count,
                "labels": info.labels
            }
        except redis.RedisError as e:
            logger.error(f"Failed to get sensor info: {e}")
            return {}


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    # Initialize
    manager = IoTTimeSeriesManager()

    print("\n🏭 IoT Time-Series Platform Demo\n")

    # Create sensor
    config = TimeSeriesConfig(
        sensor_id="temp_001",
        retention_ms=RETENTION_POLICIES['raw'],
        labels={
            "sensor_id": "temp_001",
            "zone": "A",
            "type": "temperature",
            "unit": "celsius",
            "building": "Factory_1"
        }
    )

    print("📡 Creating sensor time-series...")
    manager.create_sensor_timeseries(config)

    # Simulate ingestion
    print("\n📊 Simulating data ingestion (1000 points)...")

    now = int(time.time() * 1000)
    datapoints = []

    for i in range(1000):
        datapoints.append(SensorDataPoint(
            sensor_id="temp_001",
            timestamp=now - (1000 - i) * 1000,  # 1 point per second
            value=25.0 + (i % 10) * 0.5  # Simulate temperature variation
        ))

    stats = manager.ingest_batch(datapoints)
    print(f"   Success: {stats['success']}, Duration: {stats['duration_ms']:.2f}ms")
    print(f"   Throughput: {stats['success']/stats['duration_ms']*1000:.0f} points/sec")

    # Wait for compaction
    print("\n⏳ Waiting for automatic compaction...")
    time.sleep(2)

    # Query data
    print("\n🔍 Querying last 10 minutes (raw data)...")
    from_time = now - (10 * 60 * 1000)
    data = manager.get_range("temp_001", from_time, now)
    print(f"   Retrieved {len(data)} data points")
    print(f"   Sample: {data[:3] if len(data) >= 3 else data}")

    # Query aggregated
    print("\n📈 Querying with 1-minute aggregation...")
    agg_data = manager.get_range(
        "temp_001",
        from_time,
        now,
        aggregation=AggregationType.AVG,
        bucket_size_ms=60000
    )
    print(f"   Retrieved {len(agg_data)} aggregated points")

    # Statistics
    print("\n📊 Statistics (last 10 minutes):")
    stats_data = manager.calculate_statistics("temp_001", from_time, now)
    for key, value in stats_data.items():
        print(f"   {key}: {value:.2f}")

    # Latest value
    print("\n📌 Latest value:")
    latest = manager.get_latest("temp_001")
    if latest:
        ts, val = latest
        print(f"   Timestamp: {ts}, Value: {val:.2f}")

    # Threshold alert
    print("\n🚨 Checking threshold alert (> 30°C):")
    alert = manager.check_threshold_alert("temp_001", 30.0, ">")
    print(f"   Alert triggered: {alert}")

    # Anomaly detection
    print("\n🔍 Anomaly detection (Z-score):")
    anomaly = manager.check_anomaly_detection("temp_001", window_minutes=10, std_threshold=3.0)
    print(f"   Anomaly detected: {anomaly}")

    # Sensor info
    print("\n💾 Sensor Info:")
    info = manager.get_sensor_info("temp_001")
    print(f"   Total samples: {info.get('total_samples', 0)}")
    print(f"   Memory: {info.get('memory_bytes', 0)} bytes")
    print(f"   Chunks: {info.get('chunk_count', 0)}")
```

---

## 5. Monitoring et métriques

### 5.1 KPIs critiques

```yaml
# 1. Ingestion performance
ingestion_throughput:
  target: > 1M points/sec
  current: 800k-2M points/sec

ingestion_latency_ms:
  p50: < 50ms
  p95: < 100ms
  p99: < 200ms

# 2. Query performance
query_latency_ms:
  raw_1h: < 10ms
  aggregated_24h: < 50ms
  multi_sensor_100: < 200ms

# 3. Storage efficiency
memory_per_datapoint:
  raw: ~50 bytes
  1min_agg: ~30 bytes
  compression_ratio: 10:1

# 4. Data quality
data_loss_rate:
  target: < 0.01%

compaction_lag:
  target: < 1 second
```

---

## 6. Conclusion

### Points clés

- ✅ **RedisTimeSeries** : Ingestion 1M-5M points/sec
- ✅ **Auto-compaction** : Downsampling temps réel (< 1s lag)
- ✅ **Multi-level** : raw → 1min → 1hour → 1day automatique
- ✅ **Query < 10ms** : In-memory ultra-rapide
- ✅ **Coût ÷ 10** : vs InfluxDB Cloud

### Prochaines lectures

- [RedisTimeSeries Docs](https://redis.io/docs/stack/timeseries/)
- [Cas #4 : Analytics temps réel](./04-cas-analytics-temps-reel.md)

---

**📚 Ressources** :
- [Time-Series Best Practices](https://redis.io/docs/stack/timeseries/design/)
- [IoT Architecture Patterns](https://aws.amazon.com/iot/solutions/)

⏭️ [Design patterns recommandés](/16-etudes-cas-patterns-reels/09-design-patterns-recommandes.md)
