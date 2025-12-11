🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #4 : Analytics temps réel (HyperLogLog + TimeSeries)

## Vue d'ensemble

**Niveau** : ⭐⭐⭐ Avancé
**Complexité technique** : Élevée
**Impact production** : Critique (décisions business data-driven)
**Technologies** : Redis Stack (RedisTimeSeries) + Redis Core (HyperLogLog)

---

## 1. Contexte et problématique

### Scénario business

Une plateforme SaaS B2B (type Datadog, New Relic, ou Mixpanel) fournissant des analytics en temps réel pour applications web et mobiles :

**Chiffres clés** :
- 10 000 clients (entreprises)
- 500 millions d'utilisateurs finaux trackés
- 10 milliards d'événements par jour
- 100 000 événements par seconde (peak)
- SLA : Métriques visibles en < 5 secondes
- Rétention : 1 an de données détaillées
- Dashboard refresh : 10 secondes

**Besoins métier critiques** :

1. **Métriques de trafic en temps réel**
   - Utilisateurs actifs maintenant (last 1 min, 5 min, 1h)
   - Utilisateurs uniques par jour/semaine/mois
   - Pages vues par seconde
   - Sessions actives
   - Taux de rebond

2. **Métriques de performance**
   - Latence moyenne/p95/p99 des requêtes
   - Taux d'erreur (4xx, 5xx)
   - Throughput (requests/sec)
   - Distribution géographique

3. **Métriques business**
   - Conversions en temps réel
   - Revenue par heure/jour
   - Nouveaux utilisateurs vs returning
   - Funnel analytics

4. **Agrégations temporelles multi-niveaux**
   - Raw data : 1 seconde
   - Niveau 1 : 1 minute (downsampling)
   - Niveau 2 : 5 minutes
   - Niveau 3 : 1 heure
   - Niveau 4 : 1 jour

### Problèmes à résoudre

#### 1. **Cardinalité unique à grande échelle**

```
❌ Problème : Compter les utilisateurs uniques sur 1 milliard d'événements

Approche naïve avec Set :
SET daily_users:2024-12-11 → {user1, user2, ..., userN}
Memory : N × 50 bytes (UUID) = 50GB pour 1 milliard d'users !

Impossible à l'échelle :
- 1 mois de données = 1.5TB de RAM
- Coût prohibitif
```

#### 2. **Stockage et agrégation de séries temporelles**

```
Problème : Stocker 100k events/sec avec rétention 1 an
- 100k × 60 × 60 × 24 × 365 = 3.15 trillions de points
- Format naïf : timestamp (8 bytes) + value (8 bytes) = 16 bytes
- Total : 50TB de données !

Agrégations :
- "Donner moi le trafic moyen des 7 derniers jours par heure"
- Scan de 7 × 24 = 168 heures de données
- Latence : plusieurs secondes (inacceptable)
```

#### 3. **Fenêtres glissantes (Sliding Windows)**

```
Besoin : "Utilisateurs uniques des dernières 24 heures"

Problème :
- Fenêtre glisse en continu
- À 14:30, fenêtre = [hier 14:30 → aujourd'hui 14:30]
- À 14:31, fenêtre = [hier 14:31 → aujourd'hui 14:31]

Solution naïve : Recalculer toutes les heures
→ Coûteux en CPU
```

#### 4. **Hot partitions (concentration temporelle)**

```
Problème : Tous les événements "now" écrivent dans la même clé
- Key : metrics:2024-12-11T14:30:00
- 100k writes/sec sur cette clé
- Risque de saturation d'un shard Redis
```

---

## 2. Analyse des alternatives

### Option 1 : Base de données relationnelle (PostgreSQL)

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NOT NULL,
    event_type VARCHAR(50),
    timestamp TIMESTAMP NOT NULL,
    properties JSONB
);

CREATE INDEX idx_timestamp ON events(timestamp);
CREATE INDEX idx_user_timestamp ON events(user_id, timestamp);

-- Compter utilisateurs uniques (jour)
SELECT COUNT(DISTINCT user_id)
FROM events
WHERE timestamp >= '2024-12-11' AND timestamp < '2024-12-12';

-- Trafic par heure
SELECT
    date_trunc('hour', timestamp) AS hour,
    COUNT(*) AS events
FROM events
WHERE timestamp >= NOW() - INTERVAL '24 hours'
GROUP BY hour
ORDER BY hour;
```

**Avantages** :
- ✅ ACID complet
- ✅ Queries SQL puissantes
- ✅ Données persistantes

**Inconvénients** :
- ❌ `COUNT(DISTINCT user_id)` : Full table scan → 30-60 secondes
- ❌ Indexes sur timestamp : Space × 2
- ❌ Insertion 100k rows/sec : Impossible sans partitioning
- ❌ Aggregations temps réel : Trop lentes (5-30s)
- ❌ Scaling : Sharding complexe

**Benchmark** (PostgreSQL 14, 1 milliard de rows) :
```
Requête                           Latence
-----------------------------------------
COUNT(DISTINCT user_id) 1 jour    45s
Aggregation 24h par heure         12s
Insertion 1000 events             200ms
```

**Verdict** : ❌ **Inadapté** pour analytics temps réel haute fréquence.

---

### Option 2 : Time-Series Database (InfluxDB)

```
# InfluxDB Line Protocol
events,user_id=user123,event=page_view value=1 1702300800000000000

# Query: Unique users (utilise COUNT(DISTINCT))
SELECT COUNT(DISTINCT(user_id)) FROM events
WHERE time >= now() - 24h

# Query: Events per minute
SELECT COUNT(value) FROM events
WHERE time >= now() - 1h
GROUP BY time(1m)
```

**Avantages** :
- ✅ Optimisé pour time-series
- ✅ Compression TSM (Time-Structured Merge)
- ✅ Downsampling natif (Continuous Queries)
- ✅ Rétention policies automatiques

**Inconvénients** :
- ❌ `COUNT(DISTINCT)` reste coûteux (scan)
- ❌ Latence : 1-5 secondes pour queries complexes
- ❌ Consommation mémoire élevée (Go runtime)
- ❌ Scaling horizontal limité (InfluxDB OSS = single node)

**Verdict** : ⚠️ **Bon pour time-series, mais pas optimal pour cardinality**.

---

### Option 3 : Clickhouse (OLAP)

```sql
CREATE TABLE events (
    user_id UUID,
    event_type String,
    timestamp DateTime,
    properties String
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (timestamp, user_id);

-- Unique users avec uniq() (approximatif)
SELECT uniq(user_id) FROM events
WHERE timestamp >= today() - INTERVAL 1 DAY;

-- Events par heure
SELECT
    toStartOfHour(timestamp) AS hour,
    count() AS events
FROM events
WHERE timestamp >= now() - INTERVAL 24 HOUR
GROUP BY hour;
```

**Avantages** :
- ✅ Très performant pour agrégations (columnar)
- ✅ `uniq()` utilise HyperLogLog (approximatif mais rapide)
- ✅ Compression excellente
- ✅ Scaling horizontal natif

**Inconvénients** :
- ❌ Latence : 500ms - 5s (acceptable mais pas < 100ms)
- ❌ Complexité opérationnelle élevée
- ❌ Pas de "temps réel" strict (batch inserts)
- ❌ Coût infrastructure significatif

**Verdict** : ⚠️ **Excellent pour analytics, mais overkill si Redis suffit**.

---

### Option 4 : Redis (HyperLogLog + TimeSeries) ✅

```bash
# HyperLogLog : Compter unique users
PFADD daily_users:2024-12-11 user123 user456 user789
PFCOUNT daily_users:2024-12-11
# → 3 users uniques (erreur < 0.81%)
# Memory : 12KB fixe !

# TimeSeries : Stocker events/sec
TS.ADD events:count:1s * 1523 RETENTION 3600
TS.RANGE events:count:1s - + AGGREGATION avg 60000
# → Moyenne par minute
```

**Avantages** :
- ✅ Latence : **< 10ms** pour queries
- ✅ HyperLogLog : **12KB fixe** pour cardinality (vs 50GB Set)
- ✅ TimeSeries : Compression automatique + downsampling
- ✅ In-memory : Queries ultra-rapides
- ✅ Simple à opérer (Redis standard)
- ✅ Coût : **10× moins cher** que Clickhouse

**Inconvénients** :
- ⚠️ HyperLogLog : Approximation (erreur 0.81%)
- ⚠️ TimeSeries : RAM limitée (mais tiering possible)
- ⚠️ Pas de queries SQL complexes

**Trade-offs assumés** :
- ➕ Performance × 100 vs alternatives
- ➕ Coût ÷ 10
- ➕ Simplicité maximale
- ➖ Approximation acceptable (0.81% erreur sur cardinality)
- ➖ Mémoire nécessaire (mais optimisée)

**Verdict** : ✅ **Solution optimale** pour analytics temps réel avec contraintes de latence.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────┐
│              Event Sources (Applications)                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │   Web    │  │  Mobile  │  │   API    │                   │
│  │   App    │  │   App    │  │ Backend  │                   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                   │
└───────┼─────────────┼─────────────┼─────────────────────────┘
        │ HTTP POST   │             │
        │ /track      │             │
        ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────┐
│            Event Collection API (Load Balanced)             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Event validation                                 │    │
│  │  - Deduplication (if needed)                        │    │
│  │  - Enrichment (geo, user-agent)                     │    │
│  │  - Batching (100 events → 1 write)                  │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Message Queue (Optional)                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Kafka / Redis Streams                              │    │
│  │  - Buffer spikes                                    │    │
│  │  - Replay capability                                │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│           Analytics Processing Service                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  For each event:                                    │    │
│  │  1. Update HyperLogLog (unique users)               │    │
│  │  2. Update TimeSeries (events/sec, latency, etc.)   │    │
│  │  3. Update Counters (total events)                  │    │
│  │  4. Pipeline for atomicity                          │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Redis Stack Cluster                      │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │  HyperLogLog Keys (Cardinality Estimation)         │     │
│  │  ───────────────────────────────────────           │     │
│  │  unique_users:minute:2024-12-11T14:30     12KB     │     │
│  │  unique_users:hour:2024-12-11T14          12KB     │     │
│  │  unique_users:day:2024-12-11              12KB     │     │
│  │  unique_users:week:2024-W50               12KB     │     │
│  │  unique_users:month:2024-12               12KB     │     │
│  │                                                    │     │
│  │  Total: ~60KB pour tous les niveaux temporels      │     │
│  │  (vs 50GB avec Set!)                               │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │  TimeSeries Keys (Metrics over Time)               │     │
│  │  ───────────────────────────────────────           │     │
│  │  ts:events:count                                   │     │
│  │    └─ Samples: 1/sec, retention: 1 hour            │     │
│  │    └─ Aggregation: 1min avg, retention: 1 day      │     │
│  │    └─ Aggregation: 1h avg, retention: 1 month      │     │
│  │                                                    │     │
│  │  ts:latency:p95                                    │     │
│  │  ts:errors:count                                   │     │
│  │  ts:revenue:sum                                    │     │
│  │  ...                                               │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  Master-Replica Setup (HA)                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ Master   │──│ Replica1 │  │ Replica2 │                   │
│  │ (Writes) │  │ (Reads)  │  │ (Reads)  │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Analytics API Service                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  REST Endpoints:                                    │    │
│  │  - GET /metrics/unique-users?window=24h             │    │
│  │  - GET /metrics/events-per-sec?from=X&to=Y          │    │
│  │  - GET /metrics/latency?aggregation=avg&bucket=1m   │    │
│  │  - GET /dashboard/realtime                          │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│                 Dashboard (Frontend)                        │
│  - Real-time charts (Chart.js, D3.js)                       │
│  - WebSocket updates (every 10s)                            │
│  - Filters (date range, dimensions)                         │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Flux de données

#### **Flux d'ingestion (Write Path)**

```
1. Application sends event
   POST /track
   {
     "event": "page_view",
     "user_id": "user123",
     "timestamp": 1702300800,
     "properties": {
       "page": "/products/item-42",
       "referrer": "google.com",
       "country": "US"
     }
   }

2. Event Collection API
   ├─ Validation (schema, required fields)
   ├─ Enrichment (add server_timestamp, ip_to_geo)
   ├─ Batching (collect 100 events or 100ms timeout)
   └─ Send to Processing Service

3. Analytics Processing Service
   For each event:

   # A. Update HyperLogLog (unique users)
   current_minute = "2024-12-11T14:30"
   PFADD unique_users:minute:{current_minute} user123
   PFADD unique_users:hour:2024-12-11T14 user123
   PFADD unique_users:day:2024-12-11 user123

   # B. Update TimeSeries (events count)
   TS.ADD ts:events:count {timestamp} 1

   # C. Update TimeSeries (per event type)
   TS.ADD ts:events:page_view:count {timestamp} 1

   # D. Update latency metrics (if applicable)
   if event.latency:
       TS.ADD ts:latency:p95 {timestamp} {event.latency}

   # E. Pipeline execution (atomic)
   MULTI
   ... (all commands above)
   EXEC

4. TimeSeries auto-compaction
   ├─ Raw (1s samples) → 1min aggregation (automatic)
   ├─ 1min aggregation → 1h aggregation (automatic)
   └─ Old data expires per retention policy

Throughput: 100k events/sec processed
Latency: < 50ms (event received → queryable)
```

#### **Flux de requête (Read Path)**

```
1. Dashboard requests: "Unique users last 24h"
   GET /metrics/unique-users?window=24h

2. Analytics API Service
   ├─ Calculate time range
   │  from = now() - 24h
   │  to = now()
   │
   ├─ Determine granularity
   │  24h → use hourly HyperLogLog (24 keys)
   │
   └─ Query Redis

3. Redis Query (HyperLogLog merge)
   # Get all hourly HLLs for last 24h
   keys = [
     "unique_users:hour:2024-12-10T15",
     "unique_users:hour:2024-12-10T16",
     ...,
     "unique_users:hour:2024-12-11T14"
   ]

   # Merge HyperLogLogs
   PFMERGE temp:merged {keys...}
   PFCOUNT temp:merged
   # → 1,247,892 unique users

   DEL temp:merged  # Cleanup

4. Return JSON
   {
     "unique_users": 1247892,
     "window": "24h",
     "accuracy": "~0.81%",
     "cached": false
   }

Query latency: 5-15ms
Cache (optional): 30s TTL
```

### 3.3 Décisions architecturales clés

#### **Choix 1 : HyperLogLog vs Set pour unique counting**

**Comparaison mémoire** :

```python
# ❌ Set (exact counting)
users_set = set()
for i in range(1_000_000_000):  # 1 billion users
    users_set.add(f"user{i}")

# Memory: 1B × 50 bytes (UUID average) = 50GB

# ✅ HyperLogLog (approximate counting)
redis.pfadd("unique_users:day", *[f"user{i}" for i in range(1_000_000_000)])

# Memory: 12KB fixed!
# Error: ±0.81% (standard error)
# For 1B users: error ≈ ±8.1M users (acceptable)
```

**Trade-off assumé** :
- ➕ Mémoire ÷ 4,000,000 (50GB → 12KB)
- ➕ Queries O(1) au lieu de O(N)
- ➖ Approximation (mais 99.2% de précision acceptable)

---

#### **Choix 2 : TimeSeries multi-niveaux (Downsampling)**

**Problème** : Requête "trafic des 30 derniers jours par heure"

```python
# ❌ Approche naïve : Query raw data
# 30 jours × 24 heures × 3600 secondes = 2,592,000 points
# Scan + aggregation = 5-10 secondes

# ✅ Approche avec downsampling
TS.CREATE ts:events:count:1s RETENTION 3600  # 1 hour raw data

TS.CREATE ts:events:count:1m RETENTION 86400  # 1 day (1min avg)
TS.CREATERULE ts:events:count:1s ts:events:count:1m AGGREGATION avg 60000

TS.CREATE ts:events:count:1h RETENTION 2592000  # 30 days (1h avg)
TS.CREATERULE ts:events:count:1m ts:events:count:1h AGGREGATION avg 3600000

# Query 30 jours par heure → 720 points au lieu de 2.5M
TS.RANGE ts:events:count:1h (now-30d) now
# Latency: ~10ms au lieu de 5s
```

**Trade-off assumé** :
- ➕ Queries × 500 plus rapides
- ➕ Mémoire divisée par 60-3600
- ➖ Perte de granularité (mais souvent pas nécessaire)

---

#### **Choix 3 : HyperLogLog par fenêtre temporelle**

**Alternative envisagée** : 1 seul HLL global + timestamp filtering

```python
# ❌ Single HLL (impossible de filter par date)
redis.pfadd("all_users", "user123", "user456", ...)
# Problème : Comment obtenir "unique users yesterday" ?
# → Impossible sans replay des événements

# ✅ Multiple HLLs par granularité
redis.pfadd("unique_users:day:2024-12-11", "user123")
redis.pfadd("unique_users:hour:2024-12-11T14", "user123")
redis.pfadd("unique_users:minute:2024-12-11T14:30", "user123")

# Queries temporelles faciles
yesterday = redis.pfcount("unique_users:day:2024-12-10")
last_hour = redis.pfcount("unique_users:hour:2024-12-11T14")
```

**Trade-off assumé** :
- ➕ Flexibility temporelle maximale
- ➕ Queries par fenêtre O(1)
- ➖ Write amplification (3-5 HLLs par event)
- ➖ Mémoire ×5 (mais toujours minime : 60KB total)

---

## 4. Modélisation des données

### 4.1 Schéma HyperLogLog

**Pattern de clés** : `unique_users:{granularity}:{timestamp}`

```
# Par minute (sliding window très récent)
unique_users:minute:2024-12-11T14:30
unique_users:minute:2024-12-11T14:31
unique_users:minute:2024-12-11T14:32
...

# Par heure (queries 24h)
unique_users:hour:2024-12-11T00
unique_users:hour:2024-12-11T01
unique_users:hour:2024-12-11T02
...

# Par jour (queries 30j, 90j)
unique_users:day:2024-12-01
unique_users:day:2024-12-02
...
unique_users:day:2024-12-11

# Par semaine (queries annuelles)
unique_users:week:2024-W01
unique_users:week:2024-W02
...

# Par mois
unique_users:month:2024-01
unique_users:month:2024-02
...
```

**Rétention** :
```
Minute : 60 minutes (1 hour)
Hour   : 168 hours (7 days)
Day    : 365 days (1 year)
Week   : 104 weeks (2 years)
Month  : 36 months (3 years)

Total memory: ~500 × 12KB = 6MB
```

### 4.2 Schéma RedisTimeSeries

**TimeSeries avec auto-compaction** :

```bash
# Events count (raw → 1min → 1h → 1day)
TS.CREATE ts:events:count:1s
  RETENTION 3600
  LABELS metric events granularity 1s

TS.CREATE ts:events:count:1m
  RETENTION 86400
  LABELS metric events granularity 1m

TS.CREATERULE ts:events:count:1s ts:events:count:1m
  AGGREGATION sum 60000

TS.CREATE ts:events:count:1h
  RETENTION 2592000
  LABELS metric events granularity 1h

TS.CREATERULE ts:events:count:1m ts:events:count:1h
  AGGREGATION sum 3600000

TS.CREATE ts:events:count:1d
  RETENTION 31536000
  LABELS metric events granularity 1d

TS.CREATERULE ts:events:count:1h ts:events:count:1d
  AGGREGATION sum 86400000
```

**Autres métriques** :

```bash
# Latency (p50, p95, p99)
ts:latency:p50:1s
ts:latency:p95:1s
ts:latency:p99:1s

# Error rates
ts:errors:4xx:count:1s
ts:errors:5xx:count:1s

# Revenue
ts:revenue:sum:1s

# Per-dimension metrics
ts:events:country:US:count:1s
ts:events:device:mobile:count:1s
ts:events:event_type:purchase:count:1s
```

### 4.3 Index et métadonnées

**Metadata de configuration** :

```
config:metrics → Hash {
  "retention_days": "365",
  "downsampling_enabled": "true",
  "hll_precision": "14"
}
```

---

## 5. Implémentation technique

### 5.1 Code Python (Production-Ready)

```python
"""
Analytics Service avec HyperLogLog et TimeSeries
Implémentation production-ready avec multi-level aggregations
"""

import time
import hashlib
from datetime import datetime, timedelta
from typing import List, Dict, Any, Optional, Tuple
from dataclasses import dataclass
from enum import Enum
import logging

import redis
from redis.commands.timeseries.info import TSInfo

# Configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'db': 0,
    'decode_responses': True,
    'socket_timeout': 5,
    'max_connections': 200
}

# Granularités temporelles
class TimeGranularity(Enum):
    MINUTE = ("minute", 60, 3600)        # 1h retention
    HOUR = ("hour", 3600, 604800)        # 7 days retention
    DAY = ("day", 86400, 31536000)       # 1 year retention
    WEEK = ("week", 604800, 63072000)    # 2 years retention
    MONTH = ("month", 2592000, 94608000) # 3 years retention


# ============================================================================
# Data Classes
# ============================================================================

@dataclass
class Event:
    """Événement analytics"""
    user_id: str
    event_type: str
    timestamp: float
    properties: Dict[str, Any]

    def to_dict(self) -> Dict:
        return {
            'user_id': self.user_id,
            'event_type': self.event_type,
            'timestamp': self.timestamp,
            'properties': self.properties
        }


@dataclass
class MetricValue:
    """Valeur d'une métrique à un instant donné"""
    timestamp: float
    value: float


@dataclass
class UniqueUsersResult:
    """Résultat de comptage d'utilisateurs uniques"""
    count: int
    window: str
    granularity: str
    accuracy: str = "~0.81%"


# ============================================================================
# Analytics Service
# ============================================================================

class AnalyticsService:
    """
    Service d'analytics temps réel

    Features:
    - HyperLogLog pour cardinality unique
    - TimeSeries pour métriques temporelles
    - Multi-level aggregations automatiques
    - Sliding windows
    - High-throughput event processing
    """

    def __init__(self, redis_config: Dict = None):
        config = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**config)

        # Test connexion
        try:
            self.redis.ping()
            logger.info("AnalyticsService initialized")
        except redis.RedisError as e:
            logger.error(f"Redis connection failed: {e}")
            raise

        # Initialize TimeSeries structure
        self._setup_timeseries()

    def _setup_timeseries(self):
        """Créer les TimeSeries avec règles de compaction"""
        try:
            # Events count (4 niveaux)
            self._create_ts_hierarchy(
                "events:count",
                [
                    ("1s", 3600),      # 1h raw
                    ("1m", 86400),     # 1 day
                    ("1h", 2592000),   # 30 days
                    ("1d", 31536000)   # 1 year
                ],
                "sum"
            )

            # Latency p95
            self._create_ts_hierarchy(
                "latency:p95",
                [
                    ("1s", 3600),
                    ("1m", 86400),
                    ("1h", 2592000),
                    ("1d", 31536000)
                ],
                "avg"  # Average of p95s
            )

            # Error counts
            self._create_ts_hierarchy(
                "errors:count",
                [
                    ("1s", 3600),
                    ("1m", 86400),
                    ("1h", 2592000),
                    ("1d", 31536000)
                ],
                "sum"
            )

            logger.info("TimeSeries structure initialized")

        except redis.ResponseError as e:
            # TimeSeries déjà créées (normal)
            logger.debug(f"TimeSeries already exist: {e}")

    def _create_ts_hierarchy(
        self,
        base_name: str,
        levels: List[Tuple[str, int]],
        aggregation: str
    ):
        """
        Créer hiérarchie de TimeSeries avec compaction automatique

        Args:
            base_name: Nom de base (ex: "events:count")
            levels: Liste de (granularity, retention)
            aggregation: Type d'agrégation (sum, avg, min, max)
        """
        prev_key = None

        for granularity, retention in levels:
            key = f"ts:{base_name}:{granularity}"

            try:
                # Créer TimeSeries
                self.redis.ts().create(
                    key,
                    retention_msecs=retention * 1000,
                    labels={"metric": base_name, "granularity": granularity}
                )

                # Créer règle de compaction depuis niveau précédent
                if prev_key:
                    # Calculer bucket_size en ms
                    if granularity == "1m":
                        bucket_ms = 60000
                    elif granularity == "1h":
                        bucket_ms = 3600000
                    elif granularity == "1d":
                        bucket_ms = 86400000
                    else:
                        bucket_ms = 1000

                    self.redis.ts().createrule(
                        prev_key,
                        key,
                        aggregation,
                        bucket_ms
                    )

                prev_key = key

            except redis.ResponseError:
                # Déjà créé
                pass

    # ========================================================================
    # Event Ingestion
    # ========================================================================

    def track_event(self, event: Event) -> bool:
        """
        Tracker un événement (mise à jour HLL + TimeSeries)

        Args:
            event: Événement à tracker

        Returns:
            True si succès
        """
        try:
            timestamp_ms = int(event.timestamp * 1000)

            # Pipeline pour atomicité
            pipe = self.redis.pipeline()

            # 1. Update HyperLogLog (unique users) à tous les niveaux
            dt = datetime.fromtimestamp(event.timestamp)

            # Minute
            minute_key = f"unique_users:minute:{dt.strftime('%Y-%m-%dT%H:%M')}"
            pipe.pfadd(minute_key, event.user_id)
            pipe.expire(minute_key, 3600)  # 1h TTL

            # Hour
            hour_key = f"unique_users:hour:{dt.strftime('%Y-%m-%dT%H')}"
            pipe.pfadd(hour_key, event.user_id)
            pipe.expire(hour_key, 604800)  # 7 days TTL

            # Day
            day_key = f"unique_users:day:{dt.strftime('%Y-%m-%d')}"
            pipe.pfadd(day_key, event.user_id)
            pipe.expire(day_key, 31536000)  # 1 year TTL

            # 2. Update TimeSeries (events count)
            pipe.ts().add("ts:events:count:1s", timestamp_ms, 1)

            # 3. Update per-event-type TimeSeries
            event_type_key = f"ts:events:{event.event_type}:count:1s"
            pipe.ts().add(event_type_key, timestamp_ms, 1)

            # 4. Update latency si présent
            if 'latency' in event.properties:
                latency_ms = event.properties['latency']
                pipe.ts().add("ts:latency:p95:1s", timestamp_ms, latency_ms)

            # Execute pipeline
            pipe.execute()

            return True

        except redis.RedisError as e:
            logger.error(f"Failed to track event: {e}")
            return False

    def track_events_batch(self, events: List[Event]) -> int:
        """
        Tracker un batch d'événements (optimisé)

        Returns:
            Nombre d'événements trackés avec succès
        """
        success_count = 0

        # Grouper par timestamp pour optimiser
        events_by_ts = {}
        for event in events:
            ts_key = int(event.timestamp)
            if ts_key not in events_by_ts:
                events_by_ts[ts_key] = []
            events_by_ts[ts_key].append(event)

        # Pipeline global
        pipe = self.redis.pipeline()

        for ts, events_group in events_by_ts.items():
            for event in events_group:
                dt = datetime.fromtimestamp(event.timestamp)
                timestamp_ms = int(event.timestamp * 1000)

                # HyperLogLog updates
                minute_key = f"unique_users:minute:{dt.strftime('%Y-%m-%dT%H:%M')}"
                pipe.pfadd(minute_key, event.user_id)

                hour_key = f"unique_users:hour:{dt.strftime('%Y-%m-%dT%H')}"
                pipe.pfadd(hour_key, event.user_id)

                day_key = f"unique_users:day:{dt.strftime('%Y-%m-%d')}"
                pipe.pfadd(day_key, event.user_id)

                # TimeSeries
                pipe.ts().add("ts:events:count:1s", timestamp_ms, 1)

        try:
            results = pipe.execute()
            success_count = len([r for r in results if r])

            logger.info(f"Batch tracked: {success_count}/{len(events) * 4} operations")
            return success_count

        except redis.RedisError as e:
            logger.error(f"Batch tracking failed: {e}")
            return 0

    # ========================================================================
    # Unique Users Queries (HyperLogLog)
    # ========================================================================

    def get_unique_users_count(
        self,
        window: str = "24h",
        granularity: Optional[str] = None
    ) -> UniqueUsersResult:
        """
        Compter utilisateurs uniques sur une fenêtre temporelle

        Args:
            window: Fenêtre (1h, 24h, 7d, 30d, etc.)
            granularity: Forcer une granularité (hour, day)

        Returns:
            UniqueUsersResult
        """
        # Parse window
        amount, unit = self._parse_window(window)

        # Déterminer granularité optimale si non spécifiée
        if granularity is None:
            if amount <= 1 and unit in ['h', 'hour']:
                granularity = "minute"
            elif amount <= 48 and unit in ['h', 'hour']:
                granularity = "hour"
            else:
                granularity = "day"

        # Générer liste de clés
        keys = self._generate_hll_keys(amount, unit, granularity)

        if not keys:
            return UniqueUsersResult(count=0, window=window, granularity=granularity)

        try:
            if len(keys) == 1:
                # Single key
                count = self.redis.pfcount(keys[0])
            else:
                # Merge multiple HLLs
                temp_key = f"temp:merged:{int(time.time()*1000)}"
                self.redis.pfmerge(temp_key, *keys)
                count = self.redis.pfcount(temp_key)
                self.redis.delete(temp_key)

            return UniqueUsersResult(
                count=count,
                window=window,
                granularity=granularity
            )

        except redis.RedisError as e:
            logger.error(f"Failed to get unique users count: {e}")
            return UniqueUsersResult(count=0, window=window, granularity=granularity)

    def _parse_window(self, window: str) -> Tuple[int, str]:
        """Parse window string (ex: '24h' → (24, 'h'))"""
        import re
        match = re.match(r'(\d+)([hdwm])', window.lower())
        if not match:
            raise ValueError(f"Invalid window format: {window}")

        amount = int(match.group(1))
        unit = match.group(2)
        return amount, unit

    def _generate_hll_keys(
        self,
        amount: int,
        unit: str,
        granularity: str
    ) -> List[str]:
        """Générer liste de clés HLL pour fenêtre temporelle"""
        keys = []
        now = datetime.now()

        if granularity == "minute":
            for i in range(amount):
                dt = now - timedelta(minutes=i)
                key = f"unique_users:minute:{dt.strftime('%Y-%m-%dT%H:%M')}"
                keys.append(key)

        elif granularity == "hour":
            hours = amount if unit == 'h' else amount * 24
            for i in range(hours):
                dt = now - timedelta(hours=i)
                key = f"unique_users:hour:{dt.strftime('%Y-%m-%dT%H')}"
                keys.append(key)

        elif granularity == "day":
            days = amount if unit == 'd' else amount * 7 if unit == 'w' else amount * 30
            for i in range(days):
                dt = now - timedelta(days=i)
                key = f"unique_users:day:{dt.strftime('%Y-%m-%d')}"
                keys.append(key)

        return keys

    # ========================================================================
    # TimeSeries Queries
    # ========================================================================

    def get_metric_range(
        self,
        metric: str,
        from_timestamp: float,
        to_timestamp: float,
        aggregation: Optional[str] = None,
        bucket_size: Optional[int] = None
    ) -> List[MetricValue]:
        """
        Récupérer une métrique sur une période

        Args:
            metric: Nom de la métrique (ex: "events:count")
            from_timestamp: Timestamp début
            to_timestamp: Timestamp fin
            aggregation: Type d'agrégation (avg, sum, min, max)
            bucket_size: Taille du bucket en secondes

        Returns:
            Liste de MetricValue
        """
        # Déterminer granularité optimale
        duration = to_timestamp - from_timestamp

        if duration <= 3600:  # 1h
            key = f"ts:{metric}:1s"
        elif duration <= 86400:  # 1 day
            key = f"ts:{metric}:1m"
        elif duration <= 2592000:  # 30 days
            key = f"ts:{metric}:1h"
        else:
            key = f"ts:{metric}:1d"

        from_ms = int(from_timestamp * 1000)
        to_ms = int(to_timestamp * 1000)

        try:
            if aggregation and bucket_size:
                # Agrégation custom
                bucket_ms = bucket_size * 1000
                results = self.redis.ts().range(
                    key,
                    from_ms,
                    to_ms,
                    aggregation_type=aggregation,
                    bucket_size_msec=bucket_ms
                )
            else:
                # Raw data
                results = self.redis.ts().range(key, from_ms, to_ms)

            return [
                MetricValue(timestamp=ts/1000, value=value)
                for ts, value in results
            ]

        except redis.ResponseError as e:
            logger.error(f"Failed to get metric range: {e}")
            return []

    def get_metric_latest(self, metric: str, granularity: str = "1s") -> Optional[MetricValue]:
        """Récupérer la dernière valeur d'une métrique"""
        key = f"ts:{metric}:{granularity}"

        try:
            result = self.redis.ts().get(key)
            if result:
                ts, value = result
                return MetricValue(timestamp=ts/1000, value=value)
            return None

        except redis.ResponseError as e:
            logger.error(f"Failed to get latest metric: {e}")
            return None

    # ========================================================================
    # Dashboard Queries (Combined)
    # ========================================================================

    def get_realtime_dashboard(self) -> Dict[str, Any]:
        """
        Récupérer toutes les métriques pour dashboard temps réel

        Returns:
            Dict avec toutes les métriques
        """
        now = time.time()
        one_hour_ago = now - 3600

        # Parallel queries
        unique_users_1h = self.get_unique_users_count("1h")
        unique_users_24h = self.get_unique_users_count("24h")
        unique_users_7d = self.get_unique_users_count("7d")

        events_last_minute = self.get_metric_range(
            "events:count",
            now - 60,
            now
        )

        latency_p95 = self.get_metric_latest("latency:p95", "1m")
        errors_count = self.get_metric_latest("errors:count", "1m")

        # Calculer events/sec
        total_events = sum(m.value for m in events_last_minute)
        events_per_sec = total_events / 60 if events_last_minute else 0

        return {
            "timestamp": now,
            "unique_users": {
                "last_hour": unique_users_1h.count,
                "last_24h": unique_users_24h.count,
                "last_7d": unique_users_7d.count
            },
            "throughput": {
                "events_per_second": round(events_per_sec, 2),
                "total_last_minute": int(total_events)
            },
            "performance": {
                "latency_p95_ms": latency_p95.value if latency_p95 else 0,
            },
            "errors": {
                "count_last_minute": errors_count.value if errors_count else 0
            }
        }

    # ========================================================================
    # Maintenance
    # ========================================================================

    def cleanup_expired_hll(self):
        """Nettoyer les HyperLogLog expirés (optionnel, TTL gère déjà)"""
        # Redis TTL gère automatiquement
        pass

    def get_memory_stats(self) -> Dict:
        """Statistiques mémoire des structures"""
        try:
            info = self.redis.info("memory")

            # Compter les HLL keys
            cursor = 0
            hll_count = 0
            while True:
                cursor, keys = self.redis.scan(cursor, match="unique_users:*", count=1000)
                hll_count += len(keys)
                if cursor == 0:
                    break

            # Compter les TS keys
            cursor = 0
            ts_count = 0
            while True:
                cursor, keys = self.redis.scan(cursor, match="ts:*", count=1000)
                ts_count += len(keys)
                if cursor == 0:
                    break

            return {
                "total_memory_mb": info['used_memory'] / (1024 * 1024),
                "hll_keys_count": hll_count,
                "timeseries_keys_count": ts_count,
                "estimated_hll_memory_kb": hll_count * 12,  # 12KB per HLL
            }

        except redis.RedisError as e:
            logger.error(f"Failed to get memory stats: {e}")
            return {}


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    import random

    # Initialize service
    service = AnalyticsService()

    print("\n📊 Simulating analytics events...")

    # Simuler 1000 événements
    events = []
    now = time.time()

    for i in range(1000):
        event = Event(
            user_id=f"user{random.randint(1, 500)}",  # 500 users uniques
            event_type=random.choice(["page_view", "click", "purchase"]),
            timestamp=now - random.randint(0, 3600),  # Last hour
            properties={
                "latency": random.randint(10, 500)
            }
        )
        events.append(event)

    # Track batch
    service.track_events_batch(events)
    print(f"✅ Tracked {len(events)} events")

    # Queries
    print("\n📈 Unique Users:")
    result_1h = service.get_unique_users_count("1h")
    print(f"   Last hour: {result_1h.count} users (±{result_1h.accuracy})")

    result_24h = service.get_unique_users_count("24h")
    print(f"   Last 24h: {result_24h.count} users")

    # TimeSeries
    print("\n⏱️  Events over time:")
    events_range = service.get_metric_range(
        "events:count",
        now - 300,  # Last 5 minutes
        now,
        aggregation="sum",
        bucket_size=60  # 1 minute buckets
    )

    for metric in events_range[:5]:
        dt = datetime.fromtimestamp(metric.timestamp)
        print(f"   {dt.strftime('%H:%M')}: {int(metric.value)} events")

    # Dashboard
    print("\n🎯 Real-time Dashboard:")
    dashboard = service.get_realtime_dashboard()
    print(f"   Unique users (1h): {dashboard['unique_users']['last_hour']}")
    print(f"   Events/sec: {dashboard['throughput']['events_per_second']}")
    print(f"   Latency p95: {dashboard['performance']['latency_p95_ms']:.2f}ms")

    # Memory stats
    print("\n💾 Memory Stats:")
    stats = service.get_memory_stats()
    print(f"   Total memory: {stats['total_memory_mb']:.2f} MB")
    print(f"   HLL keys: {stats['hll_keys_count']} (~{stats['estimated_hll_memory_kb']} KB)")
    print(f"   TimeSeries keys: {stats['timeseries_keys_count']}")
```

### 5.2 Code Node.js (API REST)

```javascript
/**
 * Analytics API avec Express.js
 * Endpoints pour dashboard temps réel
 */

const express = require('express');
const Redis = require('ioredis');
const app = express();

// Configuration
const REDIS_CONFIG = {
  host: 'localhost',
  port: 6379,
  retryStrategy: (times) => Math.min(times * 50, 2000),
};

const redis = new Redis(REDIS_CONFIG);

// Middleware
app.use(express.json());

// ============================================================================
// Helper Functions
// ============================================================================

function parseWindow(window) {
  const match = window.match(/^(\d+)([hdwm])$/);
  if (!match) throw new Error(`Invalid window: ${window}`);

  return {
    amount: parseInt(match[1], 10),
    unit: match[2],
  };
}

function generateHLLKeys(amount, unit, granularity) {
  const keys = [];
  const now = new Date();

  if (granularity === 'hour') {
    const hours = unit === 'h' ? amount : amount * 24;
    for (let i = 0; i < hours; i++) {
      const dt = new Date(now.getTime() - i * 3600 * 1000);
      const key = `unique_users:hour:${dt.toISOString().slice(0, 13)}`;
      keys.push(key);
    }
  } else if (granularity === 'day') {
    let days = amount;
    if (unit === 'w') days = amount * 7;
    if (unit === 'm') days = amount * 30;

    for (let i = 0; i < days; i++) {
      const dt = new Date(now.getTime() - i * 86400 * 1000);
      const key = `unique_users:day:${dt.toISOString().slice(0, 10)}`;
      keys.push(key);
    }
  }

  return keys;
}

// ============================================================================
// API Endpoints
// ============================================================================

/**
 * POST /track
 * Track an event
 */
app.post('/track', async (req, res) => {
  try {
    const { user_id, event_type, properties = {} } = req.body;

    if (!user_id || !event_type) {
      return res.status(400).json({ error: 'Missing user_id or event_type' });
    }

    const now = Date.now();
    const dt = new Date(now);

    // Pipeline
    const pipeline = redis.pipeline();

    // HyperLogLog updates
    const minuteKey = `unique_users:minute:${dt.toISOString().slice(0, 16)}`;
    pipeline.pfadd(minuteKey, user_id);
    pipeline.expire(minuteKey, 3600);

    const hourKey = `unique_users:hour:${dt.toISOString().slice(0, 13)}`;
    pipeline.pfadd(hourKey, user_id);
    pipeline.expire(hourKey, 604800);

    const dayKey = `unique_users:day:${dt.toISOString().slice(0, 10)}`;
    pipeline.pfadd(dayKey, user_id);
    pipeline.expire(dayKey, 31536000);

    // TimeSeries
    pipeline.call('TS.ADD', 'ts:events:count:1s', now, 1);
    pipeline.call('TS.ADD', `ts:events:${event_type}:count:1s`, now, 1);

    if (properties.latency) {
      pipeline.call('TS.ADD', 'ts:latency:p95:1s', now, properties.latency);
    }

    await pipeline.exec();

    res.json({ success: true, timestamp: now });

  } catch (err) {
    console.error('Track error:', err);
    res.status(500).json({ error: 'Failed to track event' });
  }
});

/**
 * GET /metrics/unique-users
 * Get unique users count
 *
 * Query params:
 * - window: 1h, 24h, 7d, 30d (default: 24h)
 */
app.get('/metrics/unique-users', async (req, res) => {
  try {
    const window = req.query.window || '24h';
    const { amount, unit } = parseWindow(window);

    // Determine granularity
    let granularity;
    if (amount <= 48 && unit === 'h') {
      granularity = 'hour';
    } else {
      granularity = 'day';
    }

    // Generate keys
    const keys = generateHLLKeys(amount, unit, granularity);

    if (keys.length === 0) {
      return res.json({ count: 0, window, granularity });
    }

    let count;
    if (keys.length === 1) {
      count = await redis.pfcount(keys[0]);
    } else {
      const tempKey = `temp:merged:${Date.now()}`;
      await redis.pfmerge(tempKey, ...keys);
      count = await redis.pfcount(tempKey);
      await redis.del(tempKey);
    }

    res.json({
      count,
      window,
      granularity,
      accuracy: '~0.81%',
    });

  } catch (err) {
    console.error('Unique users error:', err);
    res.status(500).json({ error: 'Failed to get unique users' });
  }
});

/**
 * GET /metrics/timeseries
 * Get time series data
 *
 * Query params:
 * - metric: events:count, latency:p95, etc.
 * - from: Unix timestamp (default: now - 1h)
 * - to: Unix timestamp (default: now)
 * - aggregation: avg, sum, min, max (optional)
 * - bucket: Bucket size in seconds (optional)
 */
app.get('/metrics/timeseries', async (req, res) => {
  try {
    const metric = req.query.metric || 'events:count';
    const now = Date.now();
    const from = parseInt(req.query.from) || (now - 3600000);
    const to = parseInt(req.query.to) || now;

    // Determine granularity
    const duration = (to - from) / 1000;
    let granularity;
    if (duration <= 3600) granularity = '1s';
    else if (duration <= 86400) granularity = '1m';
    else if (duration <= 2592000) granularity = '1h';
    else granularity = '1d';

    const key = `ts:${metric}:${granularity}`;

    // Build command
    const args = [key, from, to];

    if (req.query.aggregation && req.query.bucket) {
      args.push('AGGREGATION', req.query.aggregation, parseInt(req.query.bucket) * 1000);
    }

    const results = await redis.call('TS.RANGE', ...args);

    // Parse results
    const data = results.map(([timestamp, value]) => ({
      timestamp: timestamp / 1000,
      value: parseFloat(value),
    }));

    res.json({ data, metric, granularity });

  } catch (err) {
    console.error('TimeSeries error:', err);
    res.status(500).json({ error: 'Failed to get timeseries data' });
  }
});

/**
 * GET /dashboard/realtime
 * Get all metrics for real-time dashboard
 */
app.get('/dashboard/realtime', async (req, res) => {
  try {
    const now = Date.now();

    // Parallel queries
    const [
      uniqueUsers1h,
      uniqueUsers24h,
      eventsLastMinute,
      latestLatency,
    ] = await Promise.all([
      // Unique users 1h
      (async () => {
        const keys = generateHLLKeys(1, 'h', 'hour');
        const tempKey = `temp:merged:${now}:1h`;
        await redis.pfmerge(tempKey, ...keys);
        const count = await redis.pfcount(tempKey);
        await redis.del(tempKey);
        return count;
      })(),

      // Unique users 24h
      (async () => {
        const keys = generateHLLKeys(24, 'h', 'hour');
        const tempKey = `temp:merged:${now}:24h`;
        await redis.pfmerge(tempKey, ...keys);
        const count = await redis.pfcount(tempKey);
        await redis.del(tempKey);
        return count;
      })(),

      // Events last minute
      redis.call('TS.RANGE', 'ts:events:count:1s', now - 60000, now),

      // Latest latency
      redis.call('TS.GET', 'ts:latency:p95:1m'),
    ]);

    // Calculate events/sec
    const totalEvents = eventsLastMinute.reduce((sum, [, val]) => sum + parseFloat(val), 0);
    const eventsPerSec = totalEvents / 60;

    const latencyP95 = latestLatency ? parseFloat(latestLatency[1]) : 0;

    res.json({
      timestamp: now / 1000,
      unique_users: {
        last_hour: uniqueUsers1h,
        last_24h: uniqueUsers24h,
      },
      throughput: {
        events_per_second: Math.round(eventsPerSec * 100) / 100,
        total_last_minute: Math.round(totalEvents),
      },
      performance: {
        latency_p95_ms: latencyP95,
      },
    });

  } catch (err) {
    console.error('Dashboard error:', err);
    res.status(500).json({ error: 'Failed to get dashboard data' });
  }
});

/**
 * GET /stats/memory
 * Get memory statistics
 */
app.get('/stats/memory', async (req, res) => {
  try {
    const info = await redis.info('memory');
    const lines = info.split('\r\n');

    const getMemoryValue = (key) => {
      const line = lines.find(l => l.startsWith(key));
      return line ? parseFloat(line.split(':')[1]) : 0;
    };

    // Count keys
    let hllCount = 0;
    let tsCount = 0;
    let cursor = '0';

    do {
      const [newCursor, keys] = await redis.scan(cursor, 'MATCH', 'unique_users:*', 'COUNT', 1000);
      hllCount += keys.length;
      cursor = newCursor;
    } while (cursor !== '0');

    cursor = '0';
    do {
      const [newCursor, keys] = await redis.scan(cursor, 'MATCH', 'ts:*', 'COUNT', 1000);
      tsCount += keys.length;
      cursor = newCursor;
    } while (cursor !== '0');

    res.json({
      total_memory_mb: getMemoryValue('used_memory') / (1024 * 1024),
      hll_keys_count: hllCount,
      timeseries_keys_count: tsCount,
      estimated_hll_memory_kb: hllCount * 12,
    });

  } catch (err) {
    console.error('Memory stats error:', err);
    res.status(500).json({ error: 'Failed to get memory stats' });
  }
});

// ============================================================================
// Server Start
// ============================================================================

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`🚀 Analytics API listening on port ${PORT}`);
  console.log(`   POST /track`);
  console.log(`   GET  /metrics/unique-users?window=24h`);
  console.log(`   GET  /metrics/timeseries?metric=events:count`);
  console.log(`   GET  /dashboard/realtime`);
  console.log(`   GET  /stats/memory`);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, closing connections...');
  await redis.quit();
  process.exit(0);
});

module.exports = app;
```

---

## 6. Monitoring et métriques

### 6.1 KPIs critiques

```yaml
# Dashboard Grafana

# 1. Event ingestion rate
event_ingestion_rate:
  current: 50k/sec
  peak: 100k/sec
  target: > 80% capacity utilization

# 2. HyperLogLog accuracy
hll_error_rate:
  theoretical: 0.81%
  monitored: < 1.0%

# 3. Query latency
query_latency_ms:
  unique_users: < 10ms (p95)
  timeseries_range: < 50ms (p95)
  dashboard: < 100ms (p95)

# 4. Memory usage
memory_usage:
  hll_per_key: 12KB
  total_hll: ~6MB (500 keys)
  timeseries: ~2GB (estimated)

# 5. TimeSeries compaction lag
compaction_lag_seconds:
  1s_to_1m: < 60s
  1m_to_1h: < 3600s
  warning: > 300s
```

### 6.2 Alertes

```yaml
# Prometheus Alerts

groups:
  - name: analytics_alerts
    rules:

      # Event ingestion latency
      - alert: AnalyticsHighIngestionLatency
        expr: histogram_quantile(0.95, rate(event_track_duration_seconds[5m])) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Event tracking p95 latency > 100ms"

      # HyperLogLog memory spike
      - alert: AnalyticsHLLMemorySpike
        expr: redis_memory_used_bytes{key_pattern="unique_users:*"} > 100000000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "HLL keys using > 100MB (expected ~6MB)"

      # TimeSeries data gap
      - alert: AnalyticsTimeSeriesDataGap
        expr: time() - redis_timeseries_last_timestamp{metric="events:count:1s"} > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "No events received for 5+ minutes"

      # Query failure rate
      - alert: AnalyticsHighQueryFailureRate
        expr: rate(analytics_query_errors_total[5m]) / rate(analytics_queries_total[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Analytics query error rate > 1%"
```

---

## 7. Conclusion

### Points clés à retenir

- ✅ **HyperLogLog = Cardinality à coût fixe** (12KB pour 1 milliard d'éléments)
- ✅ **TimeSeries = Agrégations automatiques** multi-niveaux avec downsampling
- ✅ **Latence < 10ms** pour queries analytics temps réel
- ✅ **Mémoire optimisée** : 50GB Set → 12KB HLL (÷4,000,000)
- ✅ **Sliding windows** naturels avec clés temporelles
- ✅ **Précision acceptable** : ±0.81% erreur vs exact counting
- ✅ **Simplicité d'implémentation** vs Clickhouse/InfluxDB

### Quand NE PAS utiliser cette solution

- ❌ **Précision absolue requise** : Comptabilité, transactions financières
- ❌ **Queries SQL complexes** : Clickhouse plus approprié
- ❌ **Rétention > 5 ans** : Warehouses (BigQuery, Snowflake)
- ❌ **Machine Learning avancé** : Nécessite Spark, Databricks
- ❌ **Audit trail complet** : Base relationnelle nécessaire

### Comparaison finale

| Critère | PostgreSQL | InfluxDB | Clickhouse | Redis HLL+TS |
|---------|------------|----------|------------|--------------|
| Latence (p95) | 10-30s | 1-5s | 0.5-5s | **< 10ms** |
| Cardinality memory | 50GB | 10GB | 1GB | **12KB** |
| Downsampling | Manuel | Natif | Manuel | **Natif** |
| Ops complexity | Faible | Moyenne | Élevée | **Faible** |
| Coût (mensuel) | $500 | $2000 | $3000 | **$500** |

### Prochaines lectures

- [Cas #5 : Rate Limiting](./05-cas-rate-limiting-api-gateway.md) → Sliding Window avec Sorted Sets
- [HyperLogLog Deep Dive](../02-structures-donnees-natives/07-hyperloglog-comptage-unique.md) → Algorithme détaillé
- [RedisTimeSeries](../03-structures-donnees-etendues/06-redistimeseries-donnees-temporelles.md) → Features avancées

---

**📚 Ressources complémentaires** :
- [HyperLogLog Algorithm Explained](https://en.wikipedia.org/wiki/HyperLogLog)
- [RedisTimeSeries Documentation](https://redis.io/docs/stack/timeseries/)
- [Real-time Analytics at Scale (Redis Labs)](https://redis.com/solutions/use-cases/real-time-analytics/)

⏭️ [Cas #5 : Rate Limiting pour API Gateway](/16-etudes-cas-patterns-reels/05-cas-rate-limiting-api-gateway.md)
