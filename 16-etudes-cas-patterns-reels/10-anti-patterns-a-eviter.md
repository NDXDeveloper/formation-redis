🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Anti-Patterns à éviter absolument

## Vue d'ensemble

Cette section catalogue les **erreurs critiques** à ne jamais commettre avec Redis, observées dans des systèmes en production.

**Objectif** : Prévenir les incidents majeurs en identifiant et corrigeant ces anti-patterns avant le déploiement.

⚠️ **Chaque anti-pattern inclut** :
- Impact en production (outage, data loss, performance degradation)
- Exemple de code problématique
- Solution correcte
- Cas réel d'incident

---

## 1. Anti-Patterns de Design

### 1.1 ❌ Unbounded Collections (Memory Bomb)

**Problème** : Collections qui grandissent indéfiniment sans limite.

**Impact** :
- 🔥 **Critique** : OOM (Out of Memory) → Redis crash
- 💸 **Coût** : Scaling vertical coûteux et inefficace
- ⏱️ **Performance** : Opérations O(N) de plus en plus lentes

**Exemple problématique** :

```python
# ❌ BAD: List grows forever
def log_event(event):
    redis.lpush("application:logs", json.dumps(event))
    # Problem: After 1 million events = 100+ MB
    # After 1 year = tens of GB for logs alone!

# ❌ BAD: Set with all user IDs
def track_active_user(user_id):
    redis.sadd("active_users:all_time", user_id)
    # Problem: 50M users = 2+ GB memory forever

# ❌ BAD: Sorted Set leaderboard without cleanup
def update_score(user_id, score):
    redis.zadd("leaderboard:global", {user_id: score})
    # Problem: Users who quit still in leaderboard forever
```

**Impact réel** :

```
Incident: E-commerce platform
- Logs stored in Redis List without limit
- After 6 months: 500 GB of logs in Redis
- Cost: $5,000/month for RAM
- Resolution: Moved to Elasticsearch

Timeline:
Day 1-30: 10 GB (acceptable)
Day 90: 50 GB (warning)
Day 180: 200 GB (critical)
Day 200: 500 GB → OOM crash → 2h downtime
```

**✅ Solutions** :

```python
# Solution 1: Fixed-size collections with LTRIM
def log_event(event):
    redis.lpush("application:logs", json.dumps(event))
    redis.ltrim("application:logs", 0, 999)  # Keep last 1000
    # Alternative: Use LPUSH + LTRIM in Lua for atomicity

# Solution 2: Time-based rotation with TTL
def log_event(event):
    key = f"logs:{datetime.now().strftime('%Y-%m-%d-%H')}"  # Hourly rotation
    redis.lpush(key, json.dumps(event))
    redis.expire(key, 86400)  # 24h TTL
    # Old logs auto-deleted

# Solution 3: Periodic cleanup with SCAN
def cleanup_old_users():
    """Remove inactive users from leaderboard"""
    cursor = 0
    while True:
        cursor, members = redis.zscan("leaderboard:global", cursor, count=1000)

        for member, score in members:
            last_activity = get_user_last_activity(member)
            if now() - last_activity > timedelta(days=90):
                redis.zrem("leaderboard:global", member)

        if cursor == 0:
            break

# Solution 4: Use appropriate data structure
# Instead of storing all data in Redis, use:
# - Recent data: Redis (hot cache)
# - Historical data: PostgreSQL, Elasticsearch, S3
def log_event(event):
    # Hot: Last 1 hour in Redis
    redis.lpush(f"logs:recent:{hour}", json.dumps(event))
    redis.expire(f"logs:recent:{hour}", 3600)

    # Cold: Historical in database
    async_worker.queue(lambda: db.insert("logs", event))
```

**Guidelines** :

```yaml
Always ask when adding to collection:
  - What's the maximum size? (set LTRIM)
  - How long should data persist? (set TTL)
  - When should old data be removed? (implement cleanup)
  - Is Redis the right storage? (consider DB for historical)

Red flags:
  ⚠️ LPUSH/RPUSH without LTRIM
  ⚠️ SADD without cleanup strategy
  ⚠️ ZADD without pruning
  ⚠️ No TTL on time-based data
```

---

### 1.2 ❌ Large Keys (Megabyte Monster)

**Problème** : Stocker des valeurs > 1 MB dans une seule clé.

**Impact** :
- 🔥 **Performance** : Latence × 100-1000
- 🔥 **Blocking** : Redis single-threaded → blocking operations
- 💥 **Network** : Saturation bandwidth

**Exemple problématique** :

```python
# ❌ BAD: 100 MB JSON in one key
huge_data = {
    "users": [... 1 million users ...],
    "products": [... 500k products ...],
    "orders": [... 10 million orders ...]
}

redis.set("application:all_data", json.dumps(huge_data))  # 100 MB!

# Problems:
# 1. GET blocks Redis for 100-500ms
# 2. Network transfer: 100 MB over wire
# 3. JSON parsing: 500ms-2s
# 4. Memory fragmentation

# ❌ BAD: Large HTML page in cache
def cache_page(url, html):
    redis.setex(f"page:{url}", 3600, html)  # 5 MB HTML!
```

**Impact réel** :

```
Incident: News website

Problem:
- Cached full HTML pages (2-10 MB each)
- High traffic article = 1000 req/sec
- Each GET = 5 MB × 1000 = 5 GB/sec network
- Redis CPU: 100% (decompression/serialization)

Symptoms:
- p99 latency: 2 seconds (vs 5ms target)
- Redis blocking other operations
- Replication lag: 30 seconds

Resolution:
- Chunk large pages into 100KB pieces
- Latency: 2s → 20ms
```

**✅ Solutions** :

```python
# Solution 1: Chunk large data
def store_large_value(key, large_data, chunk_size=100_000):
    """Split large data into chunks"""
    serialized = json.dumps(large_data)
    chunks = [
        serialized[i:i+chunk_size]
        for i in range(0, len(serialized), chunk_size)
    ]

    # Store chunks
    pipe = redis.pipeline()
    for idx, chunk in enumerate(chunks):
        pipe.set(f"{key}:chunk:{idx}", chunk)
    pipe.set(f"{key}:chunks", len(chunks))
    pipe.execute()

def retrieve_large_value(key):
    """Retrieve and reassemble chunks"""
    num_chunks = int(redis.get(f"{key}:chunks"))

    pipe = redis.pipeline()
    for idx in range(num_chunks):
        pipe.get(f"{key}:chunk:{idx}")
    chunks = pipe.execute()

    return json.loads(''.join(chunks))

# Solution 2: Compression
import gzip

def store_compressed(key, data):
    serialized = json.dumps(data).encode('utf-8')
    compressed = gzip.compress(serialized, compresslevel=6)

    redis.setex(key, 3600, compressed)
    # Typical compression: 5 MB → 500 KB (÷10)

def retrieve_compressed(key):
    compressed = redis.get(key)
    decompressed = gzip.decompress(compressed)
    return json.loads(decompressed.decode('utf-8'))

# Solution 3: Store reference, not content
def cache_page_smart(url, html):
    # Don't store 5 MB HTML in Redis
    # Store in S3, cache reference
    s3_key = upload_to_s3(html)
    redis.setex(f"page:{url}:ref", 3600, s3_key)

    # Also cache metadata (small)
    redis.setex(f"page:{url}:meta", 3600, json.dumps({
        "size": len(html),
        "cached_at": time.time()
    }))

# Solution 4: Use Hash for partial access
# Instead of one huge JSON, use Hash
redis.hset("user:123", mapping={
    "name": "Alice",
    "email": "alice@example.com",
    "preferences": json.dumps(preferences),  # Small JSON
    # ... other fields
})

# Access only needed fields
name = redis.hget("user:123", "name")  # Fast, small
```

**Guidelines** :

```yaml
Size limits:
  - Recommended max: 100 KB per key
  - Warning threshold: 1 MB per key
  - Never: > 10 MB per key

Strategies:
  - < 100 KB: Store as-is
  - 100 KB - 1 MB: Consider compression
  - 1 MB - 10 MB: Chunk into smaller pieces
  - > 10 MB: Don't store in Redis (use S3, DB)

Detection:
  redis-cli --bigkeys  # Find large keys
  redis-cli --memkeys  # Memory usage per key
```

---

### 1.3 ❌ Using SELECT for Database Isolation

**Problème** : Utiliser `SELECT` pour isoler les données.

**Impact** :
- 🔥 **Thread-safety** : SELECT n'est pas thread-safe
- 💥 **Cluster** : Incompatible avec Redis Cluster
- 🐛 **Bugs** : Race conditions difficiles à debug

**Exemple problématique** :

```python
# ❌ BAD: SELECT is not thread-safe
redis = redis.Redis(host='localhost', port=6379)

def process_user_data(user_id):
    redis.select(1)  # Switch to DB 1
    user = redis.get(f"user:{user_id}")
    # BUG: Another thread might have called SELECT(2) here!
    redis.select(0)  # Switch back
    return user

# Problem in multi-threaded app:
# Thread 1: SELECT(1) → GET user → expects DB 1
# Thread 2: SELECT(2) → (executes between Thread 1's operations)
# Thread 1: GET actually reads from DB 2! → Wrong data

# ❌ BAD: Using SELECT for "environments"
redis.select(0)  # Production
redis.select(1)  # Staging
redis.select(2)  # Dev
# Problem: Not supported in Redis Cluster
```

**Impact réel** :

```
Incident: Multi-tenant SaaS platform

Design:
- DB 0: Tenant A
- DB 1: Tenant B
- DB 2: Tenant C
- ... (16 databases total)

Problem:
- Race condition between threads
- SELECT(1) followed by GET returned data from SELECT(2)
- Data leakage between tenants! (GDPR violation)

Resolution:
- Separate Redis instances per tenant
- OR: Key prefixing (tenant:A:user:123)
```

**✅ Solutions** :

```python
# Solution 1: Separate connection per database
class MultiDBRedis:
    def __init__(self, host, port):
        self.connections = {}

    def get_db(self, db_num):
        if db_num not in self.connections:
            self.connections[db_num] = redis.Redis(
                host=self.host,
                port=self.port,
                db=db_num
            )
        return self.connections[db_num]

# Usage
redis_manager = MultiDBRedis('localhost', 6379)

def process_user_data(user_id):
    db1 = redis_manager.get_db(1)  # Thread-safe
    user = db1.get(f"user:{user_id}")
    return user

# Solution 2: Key prefixing (recommended)
def get_tenant_key(tenant_id, key):
    return f"tenant:{tenant_id}:{key}"

# All operations use prefix
redis.set(get_tenant_key("tenantA", "user:123"), data)
redis.get(get_tenant_key("tenantA", "user:123"))

# Solution 3: Separate Redis instances
# Best for strong isolation
tenants = {
    "tenantA": redis.Redis(host='redis-tenantA', port=6379),
    "tenantB": redis.Redis(host='redis-tenantB', port=6379),
    "tenantC": redis.Redis(host='redis-tenantC', port=6379),
}
```

**Guidelines** :

```yaml
Never use SELECT in production for:
  ❌ Multi-tenant isolation
  ❌ Environment separation (prod/staging)
  ❌ Application module separation

Instead use:
  ✅ Key prefixing
  ✅ Separate Redis instances
  ✅ Separate connection pools per DB

SELECT is OK only for:
  ✅ Single-threaded scripts
  ✅ Development/testing
```

---

## 2. Anti-Patterns de Performance

### 2.1 ❌ KEYS in Production (The Nuclear Option)

**Problème** : Utiliser `KEYS *` ou `KEYS pattern` en production.

**Impact** :
- 🔥 **CRITICAL** : Bloque Redis complètement (O(N) operation)
- 💥 **Outage** : Toutes les autres opérations bloquées
- ⏱️ **Latency** : Peut prendre 10-60 secondes sur 10M keys

**Exemple problématique** :

```python
# ❌ BAD: KEYS blocks everything
def get_all_users():
    user_keys = redis.keys("user:*")  # 10 million keys = 30s blocked!
    return [redis.get(key) for key in user_keys]

# Impact:
# - During 30s, ALL operations blocked
# - Web requests timeout
# - Health checks fail
# - Monitoring thinks Redis is down

# ❌ BAD: Using KEYS in cron job
def cleanup_expired_sessions():
    """Runs every hour"""
    expired = redis.keys("session:*")
    for key in expired:
        if should_delete(key):
            redis.delete(key)
    # Blocks Redis for minutes every hour!
```

**Impact réel** :

```
Incident: E-commerce site during Black Friday

Timeline:
14:00: Marketing runs script to "count active sessions"
14:00: Script executes: redis.keys("session:*")
14:01: Redis blocked for 45 seconds (5M keys)
14:01: All web requests timeout
14:01: Checkout flow completely down
14:02: Revenue loss: $50,000 in 2 minutes

Root cause:
- KEYS("session:*") on 5 million keys
- Blocked all operations including checkout

Resolution:
- Killed the script
- Replaced KEYS with SCAN
- Added monitoring to detect KEYS commands
```

**✅ Solutions** :

```python
# Solution 1: Use SCAN (iterative, non-blocking)
def get_all_users_safe():
    """Non-blocking iteration over keys"""
    cursor = 0
    user_keys = []

    while True:
        cursor, keys = redis.scan(
            cursor,
            match="user:*",
            count=100  # Fetch 100 keys per iteration
        )
        user_keys.extend(keys)

        if cursor == 0:  # End of iteration
            break

    return user_keys

# SCAN benefits:
# - Non-blocking (yields control between iterations)
# - Consistent O(1) time per call
# - Can be paused/resumed

# Solution 2: SCAN with pipeline for batch processing
def cleanup_expired_sessions_safe():
    cursor = 0
    deleted = 0

    while True:
        cursor, keys = redis.scan(cursor, match="session:*", count=1000)

        if keys:
            pipe = redis.pipeline()
            for key in keys:
                if should_delete(key):
                    pipe.delete(key)

            results = pipe.execute()
            deleted += sum(results)

        if cursor == 0:
            break

    logger.info(f"Deleted {deleted} expired sessions")
    return deleted

# Solution 3: Maintain a Set for trackable keys
# Instead of scanning, maintain an index
def add_user(user_id, data):
    redis.set(f"user:{user_id}", data)
    redis.sadd("users:index", f"user:{user_id}")  # Track in Set

def get_all_users():
    user_keys = redis.smembers("users:index")  # O(N) but in-memory
    return [redis.get(key) for key in user_keys]

# Solution 4: Use RediSearch for complex queries
# Create index (once)
redis.ft("users_idx").create_index([
    TextField("name"),
    NumericField("age")
])

# Query (fast, indexed)
results = redis.ft("users_idx").search("@age:[25 35]")
```

**Guidelines** :

```yaml
Never use KEYS in production:
  ❌ KEYS *
  ❌ KEYS pattern
  ❌ Any O(N) operation without iteration

Always use instead:
  ✅ SCAN (with MATCH and COUNT)
  ✅ Maintain index Sets
  ✅ RediSearch for complex queries

Detection:
  # Monitor slow log
  redis-cli SLOWLOG GET 10

  # If you see KEYS → Fix immediately!
```

---

### 2.2 ❌ Missing Pipelining for Batch Operations

**Problème** : Exécuter N commandes avec N round-trips.

**Impact** :
- ⏱️ **Latency** : × 1000 plus lent
- 💸 **Network** : Saturation bandwidth
- 📉 **Throughput** : Gaspillage de capacité

**Exemple problématique** :

```python
# ❌ BAD: 1000 round-trips
def save_users(users):
    for user in users:  # 1000 users
        redis.set(f"user:{user.id}", json.dumps(user.dict()))

    # Cost: 1000 × 1ms RTT = 1000ms
    # vs Pipeline: 1ms RTT + processing

# ❌ BAD: N+1 query problem
def get_user_profiles(user_ids):
    profiles = []
    for user_id in user_ids:  # 100 user IDs
        profile = redis.get(f"user:{user_id}:profile")
        profiles.append(profile)

    # Cost: 100 × 1ms = 100ms
    # vs Pipeline: 1-2ms

# ❌ BAD: Individual cache checks
def get_products(product_ids):
    products = []
    for pid in product_ids:
        cached = redis.get(f"product:{pid}")
        if cached:
            products.append(cached)
        else:
            product = db.query(pid)
            redis.set(f"product:{pid}", product)
            products.append(product)

    # Problem: N round-trips for cache checks
    # then M round-trips for cache sets
```

**Impact réel** :

```
Incident: API endpoint slow

Endpoint: GET /api/users/bulk?ids=1,2,3,...,100
Expected: < 50ms
Actual: 150ms

Profiling:
- Redis operations: 120ms
- Database: 10ms
- Logic: 20ms

Root cause:
- 100 individual redis.get() calls
- 100 × 1ms RTT = 100ms
- Pipeline would be: 1-2ms

Fix:
- Use pipeline
- Latency: 150ms → 30ms (×5 improvement)
```

**✅ Solutions** :

```python
# Solution 1: Pipeline for batch writes
def save_users_fast(users):
    pipe = redis.pipeline()

    for user in users:
        pipe.set(f"user:{user.id}", json.dumps(user.dict()))

    pipe.execute()  # 1 round-trip

    # Speedup: ×1000 for 1000 users

# Solution 2: Pipeline for batch reads
def get_user_profiles_fast(user_ids):
    pipe = redis.pipeline()

    for user_id in user_ids:
        pipe.get(f"user:{user_id}:profile")

    profiles = pipe.execute()  # 1 round-trip
    return profiles

# Solution 3: Pipeline with batching
def save_users_batched(users, batch_size=100):
    """For very large datasets"""
    for i in range(0, len(users), batch_size):
        batch = users[i:i+batch_size]

        pipe = redis.pipeline()
        for user in batch:
            pipe.set(f"user:{user.id}", json.dumps(user.dict()))
        pipe.execute()

# Solution 4: MGET/MSET for simple cases
def get_products_fast(product_ids):
    keys = [f"product:{pid}" for pid in product_ids]

    # MGET: Get multiple keys in one command
    products = redis.mget(keys)

    return products
```

**Guidelines** :

```yaml
Always use pipeline when:
  ✅ Multiple independent operations
  ✅ Batch inserts/updates
  ✅ Bulk reads
  ✅ > 5 operations

Don't use pipeline when:
  ⚠️ Operations depend on each other
  ⚠️ Need immediate results per operation
  ⚠️ Using transactions (use MULTI/EXEC)

Optimal batch size:
  - Small datasets: 100-500 commands
  - Large datasets: Break into batches of 100-200
```

---

### 2.3 ❌ Not Using Connection Pooling

**Problème** : Créer nouvelle connexion pour chaque opération.

**Impact** :
- ⏱️ **Latency** : +5-10ms par connexion (TCP handshake + auth)
- 💥 **Overhead** : CPU wasted sur connection setup
- 📉 **Throughput** : Limite de connexions épuisée

**Exemple problématique** :

```python
# ❌ BAD: New connection every time
def get_user(user_id):
    client = redis.Redis(host='localhost', port=6379)
    user = client.get(f"user:{user_id}")
    client.close()
    return user

# Cost per call:
# - TCP handshake: 3-5ms
# - AUTH: 0.5ms
# - GET: 0.5ms
# - Total: 4-6ms (10× slower than necessary!)

# ❌ BAD: Connection per thread
def worker_thread():
    while True:
        task = queue.get()

        # New connection per task!
        client = redis.Redis(host='localhost', port=6379)
        process(task, client)
        client.close()
```

**✅ Solutions** :

```python
# Solution 1: Global connection pool
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,
    socket_timeout=5,
    socket_connect_timeout=3,
    socket_keepalive=True,
    health_check_interval=30
)

client = redis.Redis(connection_pool=pool)

def get_user(user_id):
    return client.get(f"user:{user_id}")
    # Reuses connection from pool

# Solution 2: Context manager for explicit pooling
class RedisPool:
    def __init__(self):
        self.pool = redis.ConnectionPool(
            host='localhost',
            port=6379,
            max_connections=100
        )

    def get_client(self):
        return redis.Redis(connection_pool=self.pool)

redis_pool = RedisPool()

def process_task(task):
    client = redis_pool.get_client()
    # Connection returned to pool automatically
    return client.get(task.key)

# Solution 3: Framework-specific pooling
# Django
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
                'retry_on_timeout': True
            }
        }
    }
}

# Flask
redis_client = redis.Redis(
    connection_pool=redis.ConnectionPool(
        host='localhost',
        max_connections=50
    )
)
```

**Guidelines** :

```yaml
Connection pool configuration:
  max_connections: (concurrent threads × 2) to (threads × 5)
  socket_timeout: 3-5 seconds
  socket_connect_timeout: 2-3 seconds
  health_check_interval: 30 seconds

Monitoring:
  - Pool exhaustion
  - Connection creation rate
  - Idle connections
```

---

## 3. Anti-Patterns de Scalabilité

### 3.1 ❌ Hot Keys (The Bottleneck)

**Problème** : Concentration du trafic sur quelques clés.

**Impact** :
- 🔥 **Bottleneck** : Une clé = un shard = limite de throughput
- 💥 **Cascading failure** : Hot shard down = tout down
- 📉 **Unbalanced** : 90% trafic sur 10% des shards

**Exemple problématique** :

```python
# ❌ BAD: Global counter (hot key)
def track_pageview():
    redis.incr("pageviews:global")
    # Problem: 100k req/sec on ONE key
    # Redis can handle ~100k ops/sec per instance
    # → Bottleneck!

# ❌ BAD: Celebrity user cache
def get_celebrity_profile(celebrity_id):
    return redis.get(f"profile:{celebrity_id}")

# Problem: If Taylor Swift tweets
# → 1 million fans hit same key in 1 second
# → Hot key bottleneck

# ❌ BAD: Trending article
def cache_article(article_id, content):
    redis.setex(f"article:{article_id}", 3600, content)

# Viral article → 50k req/sec on one key
```

**Impact réel** :

```
Incident: Social media platform

Event: Celebrity posts controversial tweet
Result: Profile page requests spike
- 500k req/sec on profile:celebrity_123
- Single Redis shard handling this key
- Shard CPU: 100%
- Other keys on same shard: slow/timeout
- Cascading failure to app servers

Fix:
- Replicate hot keys across multiple instances
- Load balance reads across replicas
```

**✅ Solutions** :

```python
# Solution 1: Local caching for hot keys
from cachetools import TTLCache

local_cache = TTLCache(maxsize=1000, ttl=60)  # 1-minute local cache

def get_hot_profile(user_id):
    # Check local cache first
    if user_id in local_cache:
        return local_cache[user_id]

    # Miss → Get from Redis
    profile = redis.get(f"profile:{user_id}")
    local_cache[user_id] = profile

    return profile

# Benefit: 90% requests served from local cache
# Only 10% hit Redis

# Solution 2: Replication with read distribution
class HotKeyManager:
    def __init__(self, master, replicas):
        self.master = master
        self.replicas = replicas
        self.replica_idx = 0

    def get_hot_key(self, key):
        # Round-robin across replicas
        replica = self.replicas[self.replica_idx % len(self.replicas)]
        self.replica_idx += 1

        return replica.get(key)

    def set_hot_key(self, key, value):
        # Write to master
        return self.master.set(key, value)

# Solution 3: Sharding hot keys
def track_pageview_sharded():
    # Shard counter across 10 keys
    shard = random.randint(0, 9)
    redis.incr(f"pageviews:global:shard:{shard}")

def get_total_pageviews():
    # Aggregate shards
    total = 0
    pipe = redis.pipeline()

    for shard in range(10):
        pipe.get(f"pageviews:global:shard:{shard}")

    results = pipe.execute()
    return sum(int(r or 0) for r in results)

# Solution 4: CDN for static hot content
def serve_hot_article(article_id):
    # Don't serve from Redis
    # Serve from CDN (Cloudflare, Fastly)
    cdn_url = f"https://cdn.example.com/articles/{article_id}"
    return redirect(cdn_url)
```

**Guidelines** :

```yaml
Detect hot keys:
  redis-cli --hotkeys

  # Monitor top keys by traffic

Prevention strategies:
  1. Local caching (L1 cache)
  2. Read replicas for hot data
  3. Shard hot counters
  4. CDN for static content
  5. Rate limiting per key

Threshold:
  ⚠️ > 10k ops/sec per key: Consider optimization
  🔥 > 50k ops/sec per key: Critical, must fix
```

---

### 3.2 ❌ Not Planning for Growth

**Problème** : Pas de stratégie pour croissance des données.

**Impact** :
- 💸 **Cost** : Scaling vertical coûteux
- 🔥 **Outage** : Memory full = éviction ou crash
- 📉 **Performance** : Degradation graduelle

**Exemple problématique** :

```python
# ❌ BAD: No eviction policy
# maxmemory: 8GB
# maxmemory-policy: noeviction

# Problem:
# Month 1: 2 GB used → OK
# Month 6: 7 GB used → Warning
# Month 8: 8 GB used → FULL
# New writes: ERROR: OOM command not allowed

# ❌ BAD: No TTL on temporary data
def cache_user_session(session_id, data):
    redis.set(f"session:{session_id}", data)
    # No TTL → Data never expires
    # Old sessions accumulate forever

# ❌ BAD: No monitoring
# No alerts on memory usage
# Discover problem when users complain
```

**✅ Solutions** :

```python
# Solution 1: Appropriate eviction policy
"""
redis.conf:

maxmemory 8gb
maxmemory-policy allkeys-lru

Policies:
- noeviction: Return errors (default - BAD!)
- allkeys-lru: Evict least recently used
- volatile-lru: Evict LRU with TTL only
- allkeys-random: Random eviction
- volatile-random: Random with TTL
- volatile-ttl: Evict soonest to expire

Recommendation: allkeys-lru or volatile-lru
"""

# Solution 2: Always set TTL
def cache_user_session(session_id, data):
    redis.setex(
        f"session:{session_id}",
        86400,  # 24h TTL
        data
    )

# Solution 3: Memory monitoring
def check_memory_usage():
    info = redis.info("memory")

    used_mb = info["used_memory"] / (1024 * 1024)
    max_mb = info["maxmemory"] / (1024 * 1024)

    usage_pct = (used_mb / max_mb) * 100

    if usage_pct > 90:
        alert("Redis memory > 90%")
    elif usage_pct > 80:
        warning("Redis memory > 80%")

    return usage_pct

# Solution 4: Growth projection
def project_growth():
    """Estimate when scaling needed"""
    current_keys = redis.dbsize()
    current_memory = get_memory_usage()

    # Historical growth rate
    keys_per_month = 1_000_000
    memory_per_million_keys = 100  # MB

    months_until_full = (
        (max_memory - current_memory)
        / (keys_per_month * memory_per_million_keys / 1_000_000)
    )

    logger.info(f"Months until Redis full: {months_until_full:.1f}")

    if months_until_full < 3:
        alert("Scale Redis within 3 months")
```

**Guidelines** :

```yaml
Capacity planning:
  1. Monitor memory usage weekly
  2. Track growth rate
  3. Project capacity needs 6 months ahead
  4. Plan scaling before 70% usage

Alerts:
  - 70%: Warning, plan scaling
  - 80%: Urgent, scale soon
  - 90%: Critical, scale now

TTL strategy:
  - Session data: 24h
  - Cache: 5min - 1h
  - Analytics: 7 days
  - Temporary data: Always set TTL
```

---

## 4. Anti-Patterns de Sécurité

### 4.1 ❌ No Authentication/Authorization

**Problème** : Redis accessible sans mot de passe.

**Impact** :
- 🔥 **CRITICAL** : Data breach, ransomware
- 💰 **Bitcoin mining** : Ressources détournées
- 💥 **Data loss** : Commande FLUSHALL par attacker

**Exemple problématique** :

```bash
# ❌ BAD: No password
redis.conf:
# requirepass commented out
bind 0.0.0.0  # Exposed to internet!

# Result: Anyone can connect:
redis-cli -h your-server.com
> FLUSHALL  # Delete everything!
> CONFIG SET dir /tmp
> CONFIG SET dbfilename shell.php
# Upload malicious code
```

**Impact réel** :

```
Incident: Ransomware attack (Multiple companies, 2019-2020)

Attack vector:
1. Scan internet for open Redis (port 6379)
2. Connect without password
3. FLUSHALL (delete all data)
4. SET ransom:message "Pay 1 BTC to recover"
5. Demand payment

Affected:
- Thousands of Redis instances
- Data loss, downtime
- Some paid ransom (no guarantee of recovery)

Prevention:
- requirepass
- bind 127.0.0.1
- Firewall rules
```

**✅ Solutions** :

```bash
# Solution 1: Require password
# redis.conf
requirepass "strong_random_password_here"

# Python client
redis_client = redis.Redis(
    host='localhost',
    port=6379,
    password='strong_random_password_here'
)

# Solution 2: Bind to localhost only
# redis.conf
bind 127.0.0.1

# Or specific private IP
bind 10.0.1.50

# Solution 3: Use ACL (Redis 6+)
# redis.conf
user default off  # Disable default user
user admin on >strong_password ~* +@all
user readonly on >read_only_password ~* +@read

# Python
redis_client = redis.Redis(
    host='localhost',
    port=6379,
    username='admin',
    password='strong_password'
)

# Solution 4: TLS/SSL
# redis.conf
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt

# Python
redis_client = redis.Redis(
    host='localhost',
    port=6380,
    ssl=True,
    ssl_cert_reqs='required',
    ssl_ca_certs='/path/to/ca.crt'
)
```

**Security checklist** :

```yaml
✅ Must have:
  - requirepass set
  - bind to private IP
  - Firewall rules (allow only app servers)
  - Regular password rotation

✅ Recommended:
  - ACL (Redis 6+) for fine-grained permissions
  - TLS/SSL for encryption in transit
  - Network isolation (VPC/private subnet)
  - Monitoring for suspicious activity

✅ Disable dangerous commands:
  rename-command FLUSHALL ""
  rename-command FLUSHDB ""
  rename-command CONFIG "CONFIG_abc123"
  rename-command SHUTDOWN "SHUTDOWN_xyz789"
```

---

## 5. Anti-Patterns Opérationnels

### 5.1 ❌ No Persistence Strategy

**Problème** : Pas de backup, pas de persistence.

**Impact** :
- 💥 **Data loss** : Redis crash = tout perdu
- 🔥 **Recovery** : Impossible de restore
- 💸 **Business** : Perte de sessions, cache warmup coûteux

**Exemple problématique** :

```bash
# ❌ BAD: No persistence
# redis.conf
save ""  # RDB disabled
appendonly no  # AOF disabled

# Problem:
# - Power outage → All data lost
# - Redis crash → All data lost
# - Server reboot → All data lost

# ❌ BAD: RDB only, infrequent saves
save 900 1     # Save if 1 key changed in 15 min
save 300 10    # Save if 10 keys changed in 5 min
save 60 10000  # Save if 10k keys changed in 1 min

# Problem: Can lose up to 15 minutes of data
```

**Impact réel** :

```
Incident: E-commerce platform

Configuration:
- appendonly no
- save 900 1

Event:
- Power outage at 14:35
- Last RDB save: 14:22
- Data loss: 13 minutes of orders
- Result: 500 orders lost
- Cost: $50,000 + customer trust

Should have:
- AOF with fsync everysec
- Would lose < 1 second of data
```

**✅ Solutions** :

```bash
# Solution 1: AOF with everysec (recommended)
# redis.conf
appendonly yes
appendfsync everysec  # Balance performance/durability

# Data loss: < 1 second maximum

# Solution 2: AOF + RDB (belt and suspenders)
appendonly yes
appendfsync everysec

save 900 1
save 300 10
save 60 10000

# Benefits:
# - AOF: Minimal data loss
# - RDB: Faster restart (compact file)

# Solution 3: Redis Sentinel for HA
# 3 Redis instances: 1 master + 2 replicas
# Automatic failover if master fails

# Solution 4: Backup strategy
#!/bin/bash
# Backup script (cron: daily at 2 AM)

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/backups/redis

# Backup RDB
cp /var/lib/redis/dump.rdb $BACKUP_DIR/dump_$TIMESTAMP.rdb

# Backup AOF
cp /var/lib/redis/appendonly.aof $BACKUP_DIR/appendonly_$TIMESTAMP.aof

# Compress
gzip $BACKUP_DIR/*_$TIMESTAMP.*

# Upload to S3
aws s3 cp $BACKUP_DIR/ s3://my-redis-backups/ --recursive

# Cleanup old backups (keep 30 days)
find $BACKUP_DIR -mtime +30 -delete
```

**Guidelines** :

```yaml
Persistence strategy:
  Use case: Pure cache → RDB or none OK
  Use case: Session store → AOF everysec
  Use case: Primary database → AOF always + RDB

Configuration:
  appendonly yes
  appendfsync everysec  # Good balance
  # OR
  appendfsync always    # Max durability, slower

Backup schedule:
  - Daily: Full backup
  - Hourly: Incremental (if critical)
  - Test restore: Monthly

Monitoring:
  - AOF rewrite status
  - RDB save failures
  - Disk space
```

---

### 5.2 ❌ No Monitoring/Alerting

**Problème** : Découvrir les problèmes quand users se plaignent.

**Impact** :
- ⏱️ **MTTR** : Mean Time To Repair élevé
- 💥 **Incidents** : Cascading failures
- 📉 **SLA** : Breach sans warning

**Exemple problématique** :

```python
# ❌ BAD: No monitoring
# Application using Redis, but:
# - No metrics collected
# - No alerts configured
# - No health checks

# Problems discovered too late:
# - Memory usage 99% → Already OOM
# - Latency spiked → Users already complaining
# - Replication lag 10 min → Data inconsistent
```

**✅ Solutions** :

```python
# Solution 1: Prometheus metrics
from prometheus_client import Counter, Histogram, Gauge

redis_operations = Counter(
    'redis_operations_total',
    'Total Redis operations',
    ['operation', 'status']
)

redis_latency = Histogram(
    'redis_operation_duration_seconds',
    'Redis operation latency',
    ['operation']
)

redis_memory = Gauge(
    'redis_memory_used_bytes',
    'Redis memory usage in bytes'
)

def monitored_get(key):
    with redis_latency.labels(operation='get').time():
        try:
            result = redis.get(key)
            redis_operations.labels(operation='get', status='success').inc()
            return result
        except Exception as e:
            redis_operations.labels(operation='get', status='error').inc()
            raise

# Solution 2: Health check endpoint
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/health')
def health_check():
    health = {
        'redis': check_redis_health(),
        'timestamp': time.time()
    }

    status_code = 200 if health['redis']['healthy'] else 503
    return jsonify(health), status_code

def check_redis_health():
    try:
        # Ping test
        start = time.time()
        redis.ping()
        latency = (time.time() - start) * 1000

        # Memory check
        info = redis.info('memory')
        memory_pct = (info['used_memory'] / info['maxmemory']) * 100

        # Replication check
        repl_info = redis.info('replication')

        healthy = (
            latency < 100 and  # < 100ms
            memory_pct < 90 and  # < 90%
            repl_info.get('master_link_status') == 'up'
        )

        return {
            'healthy': healthy,
            'latency_ms': latency,
            'memory_usage_pct': memory_pct,
            'replication_status': repl_info.get('master_link_status')
        }

    except Exception as e:
        return {'healthy': False, 'error': str(e)}

# Solution 3: Alerts (Prometheus AlertManager)
"""
# prometheus-rules.yml

groups:
  - name: redis_alerts
    rules:
      - alert: RedisDown
        expr: up{job="redis"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis instance down"

      - alert: RedisHighMemory
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage > 90%"

      - alert: RedisHighLatency
        expr: redis_operation_duration_seconds{quantile="0.99"} > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis p99 latency > 100ms"

      - alert: RedisReplicationLag
        expr: redis_replication_lag_seconds > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis replication lag > 10s"
"""
```

**Essential metrics to monitor** :

```yaml
Performance:
  - Operations per second
  - Latency (p50, p95, p99)
  - Hit ratio
  - Slow commands

Memory:
  - used_memory
  - used_memory_peak
  - mem_fragmentation_ratio
  - evicted_keys

Connections:
  - connected_clients
  - blocked_clients
  - rejected_connections

Persistence:
  - rdb_last_save_time
  - rdb_changes_since_last_save
  - aof_current_size
  - aof_rewrite_in_progress

Replication:
  - master_link_status
  - master_last_io_seconds_ago
  - slave_repl_offset
```

---

## 6. Checklist : Éviter les Anti-Patterns

### 🔍 Pre-Deployment Checklist

```yaml
Design:
  ✅ No unbounded collections (all have max size or TTL)
  ✅ No keys > 1 MB (chunked if needed)
  ✅ No SELECT usage (use key prefixing)
  ✅ Key naming convention consistent

Performance:
  ✅ No KEYS usage (replaced with SCAN)
  ✅ Pipeline used for batch operations
  ✅ Connection pooling configured
  ✅ Lua scripts for atomicity if needed

Scalability:
  ✅ Hot key strategy defined
  ✅ Eviction policy configured
  ✅ Growth projection done
  ✅ Capacity alerts set

Security:
  ✅ requirepass configured
  ✅ bind to private IP
  ✅ Firewall rules active
  ✅ Dangerous commands disabled

Operations:
  ✅ Persistence strategy defined (AOF/RDB)
  ✅ Backup schedule active
  ✅ Monitoring configured
  ✅ Alerts defined
  ✅ Health checks implemented
```

### 🚨 Red Flags to Watch

```yaml
Code review red flags:
  🚩 redis.keys()
  🚩 No TTL on temporary data
  🚩 redis.set() in loop (no pipeline)
  🚩 New Redis() in function (no pooling)
  🚩 redis.select() anywhere

Configuration red flags:
  🚩 maxmemory-policy noeviction
  🚩 save "" + appendonly no (no persistence)
  🚩 bind 0.0.0.0
  🚩 No requirepass

Monitoring red flags:
  🚩 Memory usage > 80%
  🚩 Evicted keys growing
  🚩 Slow commands increasing
  🚩 Replication lag > 5s
```

---

## Conclusion

### Impact Summary

| Anti-Pattern | Severity | Impact | Fix Difficulty |
|--------------|----------|--------|----------------|
| Unbounded collections | 🔥 Critical | OOM crash | Easy |
| Large keys | 🔥 Critical | Performance × 100 | Medium |
| SELECT usage | 💥 High | Data leakage | Easy |
| KEYS in prod | 🔥 Critical | Complete outage | Easy |
| No pipelining | 💥 High | Latency × 1000 | Easy |
| No pooling | 💥 High | Latency × 10 | Easy |
| Hot keys | 💥 High | Bottleneck | Medium |
| No growth plan | 💥 High | Surprise OOM | Medium |
| No auth | 🔥 Critical | Data breach | Easy |
| No persistence | 🔥 Critical | Data loss | Easy |
| No monitoring | 💥 High | Late detection | Medium |

### Prevention Strategy

```
1. Education
   - Train team on anti-patterns
   - Code review checklist
   - Share incident post-mortems

2. Automation
   - Linting rules (detect KEYS, SELECT)
   - Pre-commit hooks
   - CI/CD checks

3. Monitoring
   - Continuous monitoring
   - Proactive alerts
   - Regular audits

4. Testing
   - Load testing
   - Chaos engineering
   - Failure scenarios
```

### Further Reading

- [Design Patterns Recommandés](./09-design-patterns-recommandes.md) → Good practices
- [Redis Best Practices](https://redis.io/docs/management/optimization/)
- Incident post-mortems from previous cases

---

**🎯 Remember** : La meilleure façon d'éviter ces anti-patterns est de les connaître et de les détecter tôt. Ce document est votre checklist pour un Redis production-grade et résilient.

⏭️ [Gouvernance et Conformité](/17-gouvernance-conformite/README.md)
