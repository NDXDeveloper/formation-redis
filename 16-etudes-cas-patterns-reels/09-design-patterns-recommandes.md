🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Design Patterns Recommandés

## Vue d'ensemble

Cette section synthétise les **design patterns éprouvés** observés dans les 8 cas d'étude précédents, avec des guidelines pour leur application.

**Objectif** : Fournir un catalogue de patterns réutilisables pour architecturer des systèmes Redis production-grade.

---

## 1. Cache Patterns

### 1.1 Cache-Aside (Lazy Loading)

**Cas observé** : Cas #6 (Cache SQL)

**Principe** : L'application gère le cache explicitement.

```
┌───────────────────────────────────────────────┐
│ Application Flow                              │
└───────────────────────────────────────────────┘

1. Check cache
   ├─ HIT  → Return cached data
   └─ MISS → Query database
              ├─ Store in cache
              └─ Return data

Characteristics:
- Application controls caching logic
- Data only loaded when needed (lazy)
- Cache can be inconsistent with DB
```

**Implémentation type** :

```python
def get_data(key):
    # 1. Try cache
    cached = redis.get(key)
    if cached:
        return deserialize(cached)  # Cache hit

    # 2. Cache miss → Query DB
    data = db.query(...)

    # 3. Store in cache
    redis.setex(key, TTL, serialize(data))

    return data
```

**Quand utiliser** :

✅ **Utiliser si** :
- Read-heavy workload (lecture >> écriture)
- Données pas toujours nécessaires (lazy loading OK)
- Tolérance à staleness (données légèrement obsolètes OK)
- Contrôle explicite de l'invalidation souhaité

❌ **Éviter si** :
- Write-heavy workload
- Consistance stricte requise
- Warm-up critique au démarrage

**Trade-offs** :

| Aspect | Avantage | Inconvénient |
|--------|----------|--------------|
| Performance | ✅ Cache hit ultra-rapide | ⚠️ Cache miss = 2× latency (cache + DB) |
| Consistance | ⚠️ Eventual consistency | ❌ Peut servir données stales |
| Complexité | ✅ Simple à implémenter | ⚠️ Application gère invalidation |
| Resilience | ✅ Fail-safe (fallback DB) | - |

**Variante : Cache-Aside avec Stampede Protection** :

```python
def get_data_with_lock(key):
    # Try cache
    cached = redis.get(key)
    if cached:
        return cached

    # Acquire lock to prevent stampede
    lock_key = f"{key}:lock"
    lock = redis.set(lock_key, "1", nx=True, ex=30)

    if lock:
        # This thread queries DB
        try:
            data = db.query(...)
            redis.setex(key, TTL, data)
            return data
        finally:
            redis.delete(lock_key)
    else:
        # Wait for other thread
        time.sleep(0.1)
        return get_data_with_lock(key)  # Retry
```

---

### 1.2 Write-Through Cache

**Principe** : Écriture synchrone dans cache ET database.

```
┌───────────────────────────────────────────────┐
│ Write Flow                                    │
└───────────────────────────────────────────────┘

Application
   ├─> Write to Cache
   └─> Write to Database (sync)

Read Flow
   └─> Always read from Cache (guaranteed fresh)

Characteristics:
- Cache always consistent with DB
- Write latency = cache + DB
- No cache misses on reads
```

**Implémentation type** :

```python
def save_data(key, data):
    # 1. Write to cache (fast)
    redis.setex(key, TTL, serialize(data))

    # 2. Write to DB (slower)
    db.save(data)

    # Cache is always fresh

def get_data(key):
    # Simple read from cache (no fallback needed)
    return redis.get(key)
```

**Quand utiliser** :

✅ **Utiliser si** :
- Consistance cache-DB critique
- Read-heavy avec writes modérés
- Cache misses inacceptables

❌ **Éviter si** :
- Write-heavy (latency d'écriture × 2)
- Writes non critiques

---

### 1.3 Write-Behind (Write-Back)

**Cas observé** : Cas #1 (Session Store)

**Principe** : Écriture asynchrone vers database.

```
┌───────────────────────────────────────────────┐
│ Write Flow                                    │
└───────────────────────────────────────────────┘

Application
   └─> Write to Cache (fast)
        └─> [Async] Write to Database (background)

Benefits:
- Write latency = cache only
- Batch writes to DB (throughput ×10-100)
- DB can lag behind cache
```

**Implémentation type** :

```python
import queue
import threading

write_queue = queue.Queue()

def save_data(key, data):
    # 1. Write to cache (fast, synchronous)
    redis.setex(key, TTL, serialize(data))

    # 2. Queue for async DB write
    write_queue.put((key, data))

    # Return immediately (low latency)

# Background worker
def db_writer_worker():
    while True:
        batch = []

        # Collect batch
        for _ in range(100):
            try:
                batch.append(write_queue.get(timeout=1))
            except queue.Empty:
                break

        if batch:
            # Batch write to DB
            db.bulk_insert(batch)

# Start worker thread
threading.Thread(target=db_writer_worker, daemon=True).start()
```

**Quand utiliser** :

✅ **Utiliser si** :
- Write-heavy workload
- Latence d'écriture critique
- Batch writes possibles
- Eventual consistency acceptable

❌ **Éviter si** :
- Durabilité immédiate requise (risque perte données si crash)
- ACID transactions nécessaires

**⚠️ Attention** : Risque de data loss si Redis crash avant flush vers DB.

---

### 1.4 Refresh-Ahead

**Principe** : Rafraîchir cache avant expiration.

```
┌───────────────────────────────────────────────┐
│ Refresh Strategy                              │
└───────────────────────────────────────────────┘

Timeline:
0s ──────────── 60s ──────────── 120s
    Data cached   Pre-fetch      Expire
                  at 50s         (would be)

Benefit: No cache miss ever
```

**Implémentation type** :

```python
def get_data_with_refresh(key, ttl=120):
    cached = redis.get(key)

    if cached:
        # Check TTL
        remaining_ttl = redis.ttl(key)

        # If close to expiration, refresh in background
        if remaining_ttl < ttl * 0.2:  # 20% threshold
            threading.Thread(
                target=_refresh_cache,
                args=(key,),
                daemon=True
            ).start()

        return cached

    # Cache miss (should be rare)
    return _load_and_cache(key, ttl)

def _refresh_cache(key):
    data = db.query(...)
    redis.setex(key, TTL, serialize(data))
```

**Quand utiliser** :

✅ **Utiliser si** :
- Cache misses très coûteuses
- Latence prédictible requise
- Données rafraîchissables de manière asynchrone

---

## 2. Data Modeling Patterns

### 2.1 Key Naming Convention

**Observé dans tous les cas**

**Pattern recommandé** :

```
Format: {namespace}:{entity}:{id}:{attribute}

Examples:
- user:profile:123                    (Hash)
- session:usr_abc:data                (JSON)
- product:prod_456:embedding          (Vector)
- sensor:temp_001:raw                 (TimeSeries)
- rate_limit:user:usr_123:1702300800  (String)

Benefits:
✅ Hierarchical organization
✅ Easy pattern matching (SCAN)
✅ Collision avoidance
✅ Self-documenting
```

**Guidelines** :

```yaml
Structure:
  - Use colons (:) as separators
  - Start with namespace (app/service name)
  - Include entity type
  - Include unique identifier
  - Optional: Add timestamp or version

Bad examples:
  ❌ "user123"              # No structure
  ❌ "user-123-profile"     # Inconsistent separator
  ❌ "123"                  # Not self-documenting
  ❌ "userprofile123data"   # No separation

Good examples:
  ✅ "myapp:user:123:profile"
  ✅ "cache:query:abc123"
  ✅ "session:usr_456:2024-12-11"
```

---

### 2.2 Hash vs JSON

**Cas observé** : Cas #1 (RedisJSON vs Hash)

**Decision Tree** :

```
Data structure needed?
├─ Simple key-value pairs
│  └─> Use HASH
│      - Pros: Native, HGETALL, HINCRBY
│      - Cons: Flat structure only
│
└─ Nested objects/arrays
   └─> Use JSON
       - Pros: JSONPath queries, nested updates
       - Cons: Requires Redis Stack
```

**Exemples comparatifs** :

```python
# HASH (flat data)
redis.hset("user:123", mapping={
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30
})

# Retrieve field
name = redis.hget("user:123", "name")

# Increment
redis.hincrby("user:123", "age", 1)

# ────────────────────────────────────────

# JSON (nested data)
redis.json().set("user:123", "$", {
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30,
    "address": {
        "city": "Paris",
        "country": "France"
    },
    "preferences": ["email", "sms"]
})

# Retrieve nested field
city = redis.json().get("user:123", "$.address.city")

# Update nested field
redis.json().set("user:123", "$.address.city", "Lyon")

# Append to array
redis.json().arrappend("user:123", "$.preferences", "push")
```

**Quand utiliser quoi** :

| Critère | HASH | JSON |
|---------|------|------|
| Structure | ✅ Flat (1 niveau) | ✅ Nested (N niveaux) |
| Performance | ✅ Légèrement plus rapide | ⚠️ Overhead parsing |
| Queries | ⚠️ Limité (get/set fields) | ✅ JSONPath puissant |
| Memory | ✅ Légèrement plus compact | ⚠️ JSON overhead |
| Compatibility | ✅ Redis core | ⚠️ Requires Redis Stack |
| Atomic ops | ✅ HINCRBY, HINCRBYFLOAT | ✅ JSON.NUMINCRBY |

---

### 2.3 Secondary Indexing Pattern

**Cas observé** : Cas #2 (RediSearch), Cas #7 (Vector Search)

**Sans RediSearch (manual)** :

```python
# Primary data
redis.hset("user:123", mapping={"name": "Alice", "city": "Paris"})

# Secondary index (Set)
redis.sadd("idx:city:Paris", "user:123")
redis.sadd("idx:city:Paris", "user:456")

# Query by city
user_ids = redis.smembers("idx:city:Paris")
users = [redis.hgetall(f"user:{uid}") for uid in user_ids]

# Problem: Manual maintenance
# - Must update indexes on every write
# - Risk of inconsistency
```

**Avec RediSearch (automatic)** :

```python
# Create index (once)
redis.ft("users_idx").create_index([
    TextField("name"),
    TagField("city"),
    NumericField("age")
])

# Add data (indexes automatically)
redis.hset("user:123", mapping={
    "name": "Alice",
    "city": "Paris",
    "age": 30
})

# Query (automatic index usage)
results = redis.ft("users_idx").search(
    "@city:{Paris} @age:[25 35]"
)

# Benefits:
# ✅ Automatic index maintenance
# ✅ Complex queries (AND, OR, NOT)
# ✅ Full-text search
# ✅ Aggregations
```

**Recommendation** : Utiliser RediSearch dès que indexing requis (éviter manual indexing).

---

### 2.4 Time-Series Data Modeling

**Cas observé** : Cas #4 (Analytics), Cas #8 (IoT)

**Pattern : Multi-level aggregations**

```
┌──────────────────────────────────────────────┐
│ Hierarchical Time-Series                     │
└──────────────────────────────────────────────┘

sensor:temp_001:raw          (1s granularity, 7 days)
   ├─> sensor:temp_001:1min  (1min, 1 year)
   │    ├─> sensor:temp_001:1hour (1h, 5 years)
   │    │    └─> sensor:temp_001:1day (1d, forever)
   │    │
   │    ├─> sensor:temp_001:1min_max (max values)
   │    └─> sensor:temp_001:1min_min (min values)
   │
   └─> Automatic compaction rules

Benefits:
✅ Query speed (less data points)
✅ Storage efficiency
✅ Automatic downsampling
```

**Implémentation** :

```python
# Create hierarchy
redis.ts().create("sensor:temp_001:raw", retention_msecs=7*24*3600*1000)

# Auto-compaction rules
redis.ts().createrule(
    "sensor:temp_001:raw",
    "sensor:temp_001:1min",
    aggregation_type="avg",
    bucket_size_msec=60000  # 1 minute
)

redis.ts().createrule(
    "sensor:temp_001:1min",
    "sensor:temp_001:1hour",
    aggregation_type="avg",
    bucket_size_msec=3600000  # 1 hour
)

# Query selection strategy
def get_optimal_key(time_range_hours):
    if time_range_hours < 1:
        return "sensor:temp_001:raw"
    elif time_range_hours <= 168:  # 1 week
        return "sensor:temp_001:1min"
    else:
        return "sensor:temp_001:1hour"
```

---

## 3. Performance Patterns

### 3.1 Pipeline Pattern

**Observé dans tous les cas**

**Principe** : Batch multiple commands en 1 round-trip.

```
┌──────────────────────────────────────────────┐
│ Without Pipeline (N round-trips)             │
└──────────────────────────────────────────────┘

for i in range(1000):
    redis.set(f"key:{i}", value)  # 1 RTT each

Total time: 1000 × 1ms = 1000ms

┌──────────────────────────────────────────────┐
│ With Pipeline (1 round-trip)                 │
└──────────────────────────────────────────────┘

pipe = redis.pipeline()
for i in range(1000):
    pipe.set(f"key:{i}", value)
pipe.execute()  # 1 RTT for all

Total time: 1ms + processing
Speedup: ×1000
```

**Implémentation type** :

```python
def bulk_insert(data_list, batch_size=100):
    pipe = redis.pipeline()

    for i, data in enumerate(data_list):
        pipe.set(f"key:{data.id}", data.value)

        # Execute every batch_size commands
        if (i + 1) % batch_size == 0:
            pipe.execute()
            pipe = redis.pipeline()

    # Execute remaining
    if len(pipe) > 0:
        pipe.execute()
```

**Quand utiliser** :

✅ **Toujours utiliser** pour operations multiples :
- Bulk inserts
- Batch updates
- Multiple independent reads

❌ **Ne pas utiliser** si :
- Operations interdépendantes (résultat de cmd1 nécessaire pour cmd2)
- Transactions ACID requises (utiliser MULTI/EXEC à la place)

**Guidelines** :

```python
# ❌ Bad: No pipeline
for user_id in user_ids:
    redis.get(f"user:{user_id}")

# ✅ Good: Pipeline
pipe = redis.pipeline()
for user_id in user_ids:
    pipe.get(f"user:{user_id}")
results = pipe.execute()

# ✅ Better: Optimal batch size
def batch_get(keys, batch_size=100):
    results = []

    for i in range(0, len(keys), batch_size):
        batch = keys[i:i+batch_size]
        pipe = redis.pipeline()
        for key in batch:
            pipe.get(key)
        results.extend(pipe.execute())

    return results
```

**Batch size recommendation** : 50-500 commands per pipeline.

---

### 3.2 Lua Scripting for Atomicity

**Cas observé** : Cas #3 (Leaderboard), Cas #5 (Rate Limiting)

**Principe** : Operations complexes atomiques en 1 RTT.

**Exemple : Rate Limiting atomique**

```lua
-- Without Lua (3 commands = race condition)
local count = redis.call('GET', key)
if count < limit then
    redis.call('INCR', key)
    return 1
end
return 0

-- Problem: Another thread can increment between GET and INCR
```

```lua
-- With Lua (atomic)
local key = KEYS[1]
local limit = tonumber(ARGV[1])

local current = redis.call('GET', key)

if current == false then
    redis.call('SETEX', key, 3600, 1)
    return 1
end

current = tonumber(current)

if current >= limit then
    return 0  -- Rate limited
end

redis.call('INCR', key)
return 1  -- Allowed
```

**Quand utiliser Lua** :

✅ **Utiliser si** :
- Atomicité requise (check-and-set)
- Réduire RTT (multiple commands → 1)
- Logic complexe côté serveur

❌ **Éviter si** :
- Logic simple (un seul command suffit)
- Debugging critique (Lua = black box)
- Frequent changes (déploiement script complexe)

**Best practices Lua** :

```python
# Pre-load script (once)
RATE_LIMIT_SCRIPT_SHA = redis.script_load(RATE_LIMIT_SCRIPT)

# Execute with EVALSHA (faster)
def check_rate_limit(user_id, limit):
    return redis.evalsha(
        RATE_LIMIT_SCRIPT_SHA,
        1,  # Number of keys
        f"rate_limit:{user_id}",  # KEYS[1]
        limit  # ARGV[1]
    )

# Fallback to EVAL if script not loaded
try:
    result = redis.evalsha(sha, ...)
except redis.exceptions.NoScriptError:
    result = redis.eval(script, ...)
```

---

### 3.3 Connection Pooling

**Principe** : Réutiliser connexions TCP pour éviter overhead.

```python
# ❌ Bad: New connection per operation
def get_user(user_id):
    client = redis.Redis(host='localhost', port=6379)
    user = client.get(f"user:{user_id}")
    client.close()  # Expensive!
    return user

# Cost: TCP handshake + auth = 5-10ms per call

# ✅ Good: Connection pool (shared)
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,
    decode_responses=True
)

client = redis.Redis(connection_pool=pool)

def get_user(user_id):
    return client.get(f"user:{user_id}")

# Cost: ~0.5ms per call (×10-20 faster)
```

**Configuration optimale** :

```python
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,  # Threads × 2-3
    socket_timeout=5,    # Prevent hanging
    socket_connect_timeout=3,
    socket_keepalive=True,
    health_check_interval=30  # Check stale connections
)
```

---

## 4. Scalability Patterns

### 4.1 Sharding Pattern

**Cas observé** : Cas #3 (Leaderboard sharding)

**Principe** : Distribuer données sur multiple instances.

```
┌──────────────────────────────────────────────┐
│ Hash-based Sharding                          │
└──────────────────────────────────────────────┘

user_id → hash(user_id) % num_shards → Shard N

Example:
- user:123 → hash(123) % 4 = 1 → Shard 1
- user:456 → hash(456) % 4 = 3 → Shard 3

Benefits:
✅ Distribute load
✅ Horizontal scaling
✅ Avoid hot keys
```

**Implémentation** :

```python
class ShardedRedis:
    def __init__(self, shard_configs):
        self.shards = [
            redis.Redis(**config)
            for config in shard_configs
        ]
        self.num_shards = len(self.shards)

    def _get_shard(self, key):
        # Consistent hashing
        shard_idx = hash(key) % self.num_shards
        return self.shards[shard_idx]

    def set(self, key, value):
        shard = self._get_shard(key)
        return shard.set(key, value)

    def get(self, key):
        shard = self._get_shard(key)
        return shard.get(key)

    def get_multi(self, keys):
        # Group by shard
        shard_keys = {}
        for key in keys:
            shard = self._get_shard(key)
            shard_keys.setdefault(shard, []).append(key)

        # Parallel fetch
        results = {}
        for shard, shard_key_list in shard_keys.items():
            pipe = shard.pipeline()
            for key in shard_key_list:
                pipe.get(key)
            shard_results = pipe.execute()

            for key, value in zip(shard_key_list, shard_results):
                results[key] = value

        return results
```

**Trade-offs** :

| Aspect | Avantage | Inconvénient |
|--------|----------|--------------|
| Scalability | ✅ Horizontal scaling | ⚠️ Rebalancing complexe |
| Load distribution | ✅ No hot keys | - |
| Operations | ⚠️ Single-key OK | ❌ Multi-key difficile (cross-shard) |
| Maintenance | ⚠️ More instances | ⚠️ More operational complexity |

**Recommendation** : Utiliser Redis Cluster plutôt que sharding manuel.

---

### 4.2 Master-Replica Pattern

**Observé dans tous les cas**

**Principe** : Réplication pour high availability et read scaling.

```
┌──────────────────────────────────────────────┐
│ Master-Replica Setup                         │
└──────────────────────────────────────────────┘

        ┌─────────┐
        │ Master  │ (Writes)
        └────┬────┘
             │ Replication
        ┌────┴────┬────────┐
        │         │        │
    ┌───▼───┐ ┌──▼────┐ ┌──▼────┐
    │Replica│ │Replica│ │Replica│ (Reads)
    └───────┘ └───────┘ └───────┘

Benefits:
✅ Read scaling (N replicas)
✅ High availability (failover)
✅ Geographic distribution
```

**Configuration** :

```python
# Master (writes)
master = redis.Redis(host='redis-master', port=6379)

# Replicas (reads)
replicas = [
    redis.Redis(host='redis-replica-1', port=6379),
    redis.Redis(host='redis-replica-2', port=6379),
    redis.Redis(host='redis-replica-3', port=6379),
]

# Write to master
def write_data(key, value):
    return master.set(key, value)

# Read from replica (round-robin)
replica_idx = 0

def read_data(key):
    global replica_idx
    replica = replicas[replica_idx % len(replicas)]
    replica_idx += 1
    return replica.get(key)
```

**Avec Sentinel (automatic failover)** :

```python
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('sentinel-1', 26379),
    ('sentinel-2', 26379),
    ('sentinel-3', 26379)
], socket_timeout=0.1)

# Get master (automatic failover)
master = sentinel.master_for('mymaster', socket_timeout=0.1)

# Get replica (load balancing)
replica = sentinel.slave_for('mymaster', socket_timeout=0.1)

# Use
master.set('key', 'value')  # Write
value = replica.get('key')  # Read
```

---

## 5. Reliability Patterns

### 5.1 Circuit Breaker Pattern

**Principe** : Éviter d'overload un service défaillant.

```
┌──────────────────────────────────────────────┐
│ Circuit Breaker States                       │
└──────────────────────────────────────────────┘

CLOSED (normal)
   │ Failures > threshold
   ▼
OPEN (reject all)
   │ After timeout
   ▼
HALF_OPEN (test)
   │ Success → CLOSED
   │ Failure → OPEN
```

**Implémentation** :

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.failures = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()

        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Usage
breaker = CircuitBreaker(failure_threshold=3, timeout=30)

def get_data(key):
    return breaker.call(redis.get, key)
```

---

### 5.2 Retry with Exponential Backoff

**Principe** : Retry avec délai croissant.

```python
import time
import random

def retry_with_backoff(
    func,
    max_retries=3,
    base_delay=0.1,
    max_delay=10,
    exponential_base=2
):
    """
    Retry avec exponential backoff

    Delays: 0.1s, 0.2s, 0.4s, 0.8s, ...
    """
    for attempt in range(max_retries):
        try:
            return func()
        except redis.RedisError as e:
            if attempt == max_retries - 1:
                raise  # Last attempt

            # Calculate delay
            delay = min(
                base_delay * (exponential_base ** attempt),
                max_delay
            )

            # Add jitter to avoid thundering herd
            jitter = random.uniform(0, delay * 0.1)

            time.sleep(delay + jitter)

    raise Exception("Max retries exceeded")

# Usage
def get_user(user_id):
    return retry_with_backoff(
        lambda: redis.get(f"user:{user_id}"),
        max_retries=3
    )
```

---

## 6. Monitoring Patterns

### 6.1 Metrics Pattern

**Observé dans tous les cas**

**Métriques essentielles** :

```python
from dataclasses import dataclass
from typing import Dict
import time

@dataclass
class RedisMetrics:
    """Métriques Redis à tracker"""

    # Performance
    operation_latency_ms: Dict[str, float]  # {operation: latency}
    throughput_ops_per_sec: float

    # Cache
    hit_ratio: float
    hits: int
    misses: int

    # Connections
    connected_clients: int
    blocked_clients: int

    # Memory
    used_memory_mb: float
    used_memory_peak_mb: float
    fragmentation_ratio: float

    # Persistence
    rdb_last_save_time: int
    rdb_changes_since_last_save: int

class RedisMonitor:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.ops_count = {}
        self.ops_latency = {}

    def track_operation(self, operation: str):
        """Decorator to track operations"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                start = time.time()

                try:
                    result = func(*args, **kwargs)

                    # Track success
                    latency = (time.time() - start) * 1000
                    self._record_latency(operation, latency)

                    return result
                except Exception as e:
                    # Track error
                    self._record_error(operation)
                    raise

            return wrapper
        return decorator

    def _record_latency(self, operation, latency):
        if operation not in self.ops_latency:
            self.ops_latency[operation] = []

        self.ops_latency[operation].append(latency)

        # Keep last 1000 operations
        if len(self.ops_latency[operation]) > 1000:
            self.ops_latency[operation].pop(0)

    def get_metrics(self) -> RedisMetrics:
        """Collect current metrics"""
        info = self.redis.info()
        stats = self.redis.info("stats")

        # Calculate hit ratio
        hits = stats.get("keyspace_hits", 0)
        misses = stats.get("keyspace_misses", 0)
        total = hits + misses
        hit_ratio = hits / total if total > 0 else 0

        return RedisMetrics(
            operation_latency_ms=self._calculate_avg_latency(),
            throughput_ops_per_sec=stats.get("instantaneous_ops_per_sec", 0),
            hit_ratio=hit_ratio,
            hits=hits,
            misses=misses,
            connected_clients=info.get("connected_clients", 0),
            blocked_clients=info.get("blocked_clients", 0),
            used_memory_mb=info.get("used_memory", 0) / (1024 * 1024),
            used_memory_peak_mb=info.get("used_memory_peak", 0) / (1024 * 1024),
            fragmentation_ratio=info.get("mem_fragmentation_ratio", 0),
            rdb_last_save_time=info.get("rdb_last_save_time", 0),
            rdb_changes_since_last_save=info.get("rdb_changes_since_last_save", 0)
        )

    def _calculate_avg_latency(self):
        return {
            op: sum(latencies) / len(latencies)
            for op, latencies in self.ops_latency.items()
            if latencies
        }

# Usage
monitor = RedisMonitor(redis)

@monitor.track_operation("get_user")
def get_user(user_id):
    return redis.get(f"user:{user_id}")

# Collect metrics
metrics = monitor.get_metrics()
print(f"Hit ratio: {metrics.hit_ratio:.2%}")
print(f"Avg latency: {metrics.operation_latency_ms}")
```

---

### 6.2 Health Check Pattern

```python
def redis_health_check(redis_client, timeout=2):
    """
    Health check complet

    Returns: {"healthy": bool, "details": dict}
    """
    health = {
        "healthy": True,
        "details": {}
    }

    try:
        # 1. Ping test
        start = time.time()
        redis_client.ping()
        ping_latency = (time.time() - start) * 1000

        health["details"]["ping_latency_ms"] = ping_latency

        if ping_latency > timeout * 1000:
            health["healthy"] = False
            health["details"]["error"] = "High latency"

        # 2. Memory check
        info = redis_client.info("memory")
        used_memory_pct = info.get("used_memory", 0) / info.get("maxmemory", 1)

        health["details"]["memory_usage_pct"] = used_memory_pct * 100

        if used_memory_pct > 0.9:
            health["healthy"] = False
            health["details"]["warning"] = "High memory usage"

        # 3. Replication check (if replica)
        replication = redis_client.info("replication")
        if replication.get("role") == "slave":
            master_link = replication.get("master_link_status")
            health["details"]["replication_status"] = master_link

            if master_link != "up":
                health["healthy"] = False
                health["details"]["error"] = "Replication down"

        return health

    except redis.RedisError as e:
        return {
            "healthy": False,
            "details": {"error": str(e)}
        }
```

---

## 7. Decision Matrix

### 7.1 Choisir le bon pattern de cache

```
┌────────────────────────────────────────────────────────────────┐
│ Cache Pattern Selection                                        │
├──────────────┬──────────────┬────────────┬─────────────────────┤
│ Pattern      │ Write Heavy  │ Read Heavy │ Consistency         │
├──────────────┼──────────────┼────────────┼─────────────────────┤
│ Cache-Aside  │ ⚠️           │ ✅         │ Eventual            │
│ Write-Through│ ⚠️           │ ✅         │ Strong              │
│ Write-Behind │ ✅           │ ✅         │ Eventual (risk)     │
│ Refresh-Ahead│ ⚠️           │ ✅         │ Eventual            │
└──────────────┴──────────────┴────────────┴─────────────────────┘
```

### 7.2 Choisir la structure de données

```
Use case → Structure recommandée

Session data → RedisJSON (nested) ou Hash (flat)
Full-text search → RediSearch
Leaderboard → Sorted Set
Rate limiting → String + TTL + Lua
Time-series → RedisTimeSeries
Recommendations → RediSearch Vector
Analytics → HyperLogLog + TimeSeries
Queue → List ou Stream
Pub/Sub → Pub/Sub ou Stream
Locks → String (SET NX EX)
```

---

## 8. Anti-Patterns à éviter

### ❌ Anti-Pattern 1 : Unbounded Collections

```python
# ❌ BAD: List grows forever
redis.lpush("logs", log_entry)  # No size limit!

# ✅ GOOD: Trim to max size
redis.lpush("logs", log_entry)
redis.ltrim("logs", 0, 999)  # Keep last 1000

# ✅ BETTER: Use TTL
redis.lpush("logs:2024-12-11", log_entry)
redis.expire("logs:2024-12-11", 86400)  # 1 day
```

### ❌ Anti-Pattern 2 : Large Keys

```python
# ❌ BAD: Store 100 MB JSON in one key
redis.set("huge_data", json.dumps(huge_dict))

# ✅ GOOD: Split into smaller chunks
for i, chunk in enumerate(chunks(huge_dict, size=1000)):
    redis.set(f"data:chunk:{i}", json.dumps(chunk))
```

### ❌ Anti-Pattern 3 : SELECT Database

```python
# ❌ BAD: Use SELECT to switch DBs
redis.select(1)  # NOT thread-safe!
redis.set("key", "value")

# ✅ GOOD: Separate connection per DB
db0 = redis.Redis(host='localhost', port=6379, db=0)
db1 = redis.Redis(host='localhost', port=6379, db=1)

db1.set("key", "value")
```

### ❌ Anti-Pattern 4 : KEYS in Production

```python
# ❌ BAD: KEYS blocks Redis
all_keys = redis.keys("user:*")  # O(N) - blocks everything!

# ✅ GOOD: Use SCAN (iterative)
cursor = 0
keys = []
while True:
    cursor, batch = redis.scan(cursor, match="user:*", count=100)
    keys.extend(batch)
    if cursor == 0:
        break
```

---

## Conclusion

### Checklist : Design Production-Ready Redis

```yaml
Architecture:
  ✅ Cache pattern choisi (Cache-Aside, Write-Through, etc.)
  ✅ Master-Replica pour HA
  ✅ Connection pooling configuré
  ✅ Sharding strategy si > 100 GB

Performance:
  ✅ Pipeline pour batch operations
  ✅ Lua pour atomicité si nécessaire
  ✅ Indexes (RediSearch) si queries complexes
  ✅ Multi-level aggregations (TimeSeries)

Reliability:
  ✅ Circuit breaker implémenté
  ✅ Retry with exponential backoff
  ✅ Health checks automatiques
  ✅ Failover configuré (Sentinel/Cluster)

Monitoring:
  ✅ Metrics collectées (latency, hit ratio, memory)
  ✅ Alerting configuré (Prometheus/Grafana)
  ✅ Logging approprié
  ✅ Tracing distribué (si microservices)

Data Modeling:
  ✅ Key naming convention cohérente
  ✅ TTL configuré (pas de memory leak)
  ✅ Structure de données optimale
  ✅ Pas d'anti-patterns
```

### Patterns par use case

| Use Case | Patterns recommandés |
|----------|---------------------|
| Web cache | Cache-Aside + Pipeline + Master-Replica |
| Session store | Write-Behind + RedisJSON + TTL |
| Real-time analytics | TimeSeries + HyperLogLog + Aggregations |
| Search engine | RediSearch + Secondary indexes |
| Leaderboard | Sorted Set + Composite scores |
| Rate limiting | Lua scripting + TTL |
| Recommendations | Vector search + Hybrid queries |
| IoT monitoring | TimeSeries + Auto-compaction |

---

**Prochaines lectures recommandées** :

- [Anti-patterns à éviter](./10-anti-patterns-eviter.md) → Erreurs courantes
- Modules techniques spécifiques pour approfondissement
- Documentation Redis officielle pour features avancées

⏭️ [Anti-patterns à éviter absolument](/16-etudes-cas-patterns-reels/10-anti-patterns-a-eviter.md)
