🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #6 : Cache de résultats de requêtes SQL complexes

## Vue d'ensemble

**Niveau** : ⭐⭐ Intermédiaire
**Complexité technique** : Moyenne
**Impact production** : Critique (performance et coûts)
**Technologies** : Redis Core + Pattern Cache-Aside

---

## 1. Contexte et problématique

### Scénario business

Une application SaaS de business intelligence (type Tableau, Looker, ou Metabase) avec tableau de bord analytics :

**Chiffres clés** :
- 5 000 entreprises clientes
- 100 000 utilisateurs actifs
- 500 dashboards différents
- Base de données : 2 TB de données transactionnelles
- 10 000 requêtes SQL par minute (peak)
- Requêtes complexes : 5-30 secondes d'exécution
- SLA : Dashboard load < 3 secondes

**Besoins métier critiques** :

1. **Performance des dashboards**
   - Requêtes SQL avec JOINs multiples (5-8 tables)
   - Agrégations sur millions de lignes
   - Window functions complexes
   - Latence actuelle : 5-30s → Cible : < 500ms

2. **Réduction de charge database**
   - CPU database : 80-95% en peak
   - Slow queries : 40% des requêtes > 5s
   - Connection pool saturé (200/200)
   - Coût infrastructure : $5,000/mois

3. **Expérience utilisateur**
   - Dashboard refresh toutes les 30s
   - Même requête exécutée 100× par minute
   - Users multiples sur même dashboard
   - Timeouts fréquents (> 30s)

4. **Contraintes métier**
   - Données "near real-time" acceptables (5 min de latence OK)
   - Certaines métriques mises à jour toutes les heures
   - Invalidation sélective nécessaire (nouvelles commandes → refresh)

### Problèmes à résoudre

#### 1. **Requêtes SQL lentes et répétitives**

```sql
-- Exemple: Revenue dashboard (15s execution)
SELECT
    DATE_TRUNC('day', o.created_at) AS day,
    p.category,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    COUNT(o.id) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    AVG(oi.quantity * oi.unit_price) AS avg_order_value,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY oi.quantity * oi.unit_price) AS p95_order_value
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at >= NOW() - INTERVAL '30 days'
    AND o.status = 'completed'
    AND c.country IN ('US', 'CA', 'MX')
GROUP BY DATE_TRUNC('day', o.created_at), p.category
ORDER BY day DESC, revenue DESC;
```

**Problème** :
- Exécutée 100× par minute (100 users consultent le dashboard)
- 15 secondes × 100 = 1,500 secondes CPU gaspillées
- Résultats identiques (données changent toutes les minutes seulement)

#### 2. **Cache Stampede (Thundering Herd)**

```
Scénario : Cache expire pour une requête populaire

T=0: Cache expires
T=1: Request 1 → Cache MISS → Query DB (15s)
T=2: Request 2 → Cache MISS → Query DB (15s)
T=3: Request 3 → Cache MISS → Query DB (15s)
...
T=100: Request 100 → Cache MISS → Query DB (15s)

Résultat : 100 requêtes identiques en parallèle → DB overload
```

#### 3. **Invalidation sélective**

```
Problème : Comment invalider cache quand données changent ?

Approches naïves :
❌ Invalider tout → Perte de tous les caches (même non affectés)
❌ TTL court → Trop de cache misses
❌ Jamais invalider → Données stales

Solution : Tag-based invalidation + TTL raisonnable
```

#### 4. **Sérialisation et taille des résultats**

```python
# Résultat SQL : 10,000 rows × 10 columns
result = execute_query(sql)  # List[Dict]

# Sérialisation naïve
import json
cached = json.dumps(result)  # 5 MB string

# Problème :
# - Network transfer : 5 MB × 100 requests = 500 MB/min
# - Memory : 5 MB × 1000 queries cached = 5 GB
# - Parsing : json.loads() coûteux
```

---

## 2. Analyse des alternatives

### Option 1 : Materialized Views (PostgreSQL)

```sql
-- Materialized View
CREATE MATERIALIZED VIEW revenue_dashboard AS
SELECT
    DATE_TRUNC('day', o.created_at) AS day,
    p.category,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    -- ...
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
-- ...
GROUP BY DATE_TRUNC('day', o.created_at), p.category;

-- Refresh (manual ou scheduled)
REFRESH MATERIALIZED VIEW revenue_dashboard;

-- Query (fast)
SELECT * FROM revenue_dashboard WHERE day >= NOW() - INTERVAL '7 days';
```

**Avantages** :
- ✅ Intégré à PostgreSQL
- ✅ Query simple et rapide (< 100ms)
- ✅ Index possibles

**Inconvénients** :
- ❌ Refresh bloque la view (CONCURRENTLY = plus lent)
- ❌ Pas de granularité fine (refresh complète)
- ❌ Scaling : 500 dashboards = 500 materialized views !
- ❌ Storage : Duplication des données
- ❌ Paramètres dynamiques difficiles (ex: WHERE user_id = ?)

**Verdict** : ⚠️ **Bon pour views statiques**, inadapté pour dashboards dynamiques.

---

### Option 2 : Application-Level Caching (In-Memory)

```python
# Cache in-memory (par instance)
query_cache = {}  # {query_hash: (result, expiration)}

def get_query_result(sql):
    cache_key = hashlib.md5(sql.encode()).hexdigest()

    if cache_key in query_cache:
        result, expires_at = query_cache[cache_key]
        if time.time() < expires_at:
            return result  # Cache hit

    # Cache miss
    result = execute_sql(sql)
    query_cache[cache_key] = (result, time.time() + 300)  # 5 min TTL
    return result
```

**Avantages** :
- ✅ Latence ultra-faible (< 1ms)
- ✅ Simple à implémenter
- ✅ Pas de dépendance externe

**Inconvénients** :
- ❌ **Pas de partage entre instances** (critical flaw)
- ❌ Cache hit ratio faible (divisé par N instances)
- ❌ Memory leak sans eviction policy
- ❌ Perdu au restart

**Verdict** : ❌ **Inadapté** pour architecture multi-instances (99% des cas).

---

### Option 3 : Redis Cache-Aside ✅

```python
def get_query_result(sql, params):
    # 1. Generate cache key
    cache_key = f"query:{hash_query(sql, params)}"

    # 2. Try cache (Redis)
    cached = redis.get(cache_key)
    if cached:
        return deserialize(cached)  # Cache hit

    # 3. Cache miss → Query DB
    result = execute_sql(sql, params)

    # 4. Store in cache
    redis.setex(cache_key, 300, serialize(result))  # 5 min TTL

    return result
```

**Avantages** :
- ✅ Partagé entre toutes les instances
- ✅ Latence : < 5ms pour cache hit (vs 15s DB)
- ✅ Hit ratio élevé (cache global)
- ✅ TTL automatique (cleanup)
- ✅ Eviction policy (LRU)
- ✅ Invalidation sélective possible

**Inconvénients** :
- ⚠️ Dépendance Redis (mais acceptable)
- ⚠️ Sérialisation overhead (mais gérable)
- ⚠️ Cache stampede si non géré (mais solutions existent)

**Trade-off assumé** :
- ➕ Performance × 3000 (15s → 5ms)
- ➕ Charge DB ÷ 100
- ➕ Coûts ÷ 5
- ➖ Complexité légèrement accrue (gérable)

**Verdict** : ✅ **Solution optimale** pour cache de requêtes SQL distribué.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────┐
│                   Client (Browser/Mobile)                   │
│  - Dashboard refresh every 30s                              │
└──────────┬──────────────────────────────────────────────────┘
           │ HTTPS
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Load Balancer                                  │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│         Application Servers (Stateless)                     │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│  │  App 1     │   │  App 2     │   │  App N     │           │
│  │            │   │            │   │            │           │
│  │  Query     │   │  Query     │   │  Query     │           │
│  │  Cache     │   │  Cache     │   │  Cache     │           │
│  │  Layer     │   │  Layer     │   │  Layer     │           │
│  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘           │
└────────┼────────────────┼────────────────┼──────────────────┘
         │                │                │
         └────────────────┼────────────────┘
                          ▼
         ┌────────────────────────────────┐
         │   Check Redis Cache            │
         │   ├─ Hit  → Return (5ms)       │
         │   └─ Miss → Query DB + Cache   │
         └────────────────┬───────────────┘
                          │
         ┌────────────────┴───────────────┐
         │                                │
         ▼                                ▼
┌──────────────────┐          ┌──────────────────────┐
│  Redis Cache     │          │  PostgreSQL          │
│  ┌────────────┐  │          │  ┌────────────────┐  │
│  │ Key Format │  │          │  │ Transactional  │  │
│  │            │  │          │  │ Data (2TB)     │  │
│  │ query:hash │  │          │  │                │  │
│  │ → result   │  │          │  │ Complex Queries│  │
│  │            │  │          │  │ (5-30s)        │  │
│  │ TTL: 5min  │  │          │  └────────────────┘  │
│  └────────────┘  │          │                      │
│                  │          │  Master-Replica      │
│  LRU Eviction    │          │  ┌──────┐  ┌──────┐  │
│  Max Memory: 8GB │          │  │Master│──│Replica  │
│                  │          │  └──────┘  └──────┘  │
└──────────────────┘          └──────────────────────┘
         │
         │ Invalidation Events
         ▼
┌──────────────────────────────────────────────────────────────┐
│              Event Stream (Optional)                         │
│  - order.created → Invalidate revenue_* queries              │
│  - product.updated → Invalidate product_* queries            │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Flux de requête (Cache-Aside Pattern)

#### **Cache Hit (95% des cas après warm-up)**

```
1. Client requests dashboard
   GET /api/dashboard/revenue?period=30d

2. Application generates SQL
   sql = "SELECT ... WHERE created_at >= NOW() - INTERVAL '30 days' ..."
   params = {"period": "30d"}

3. Generate cache key
   cache_key = "query:md5(sql + params)"
   → "query:a3f5b9c..."

4. Check Redis
   cached_result = redis.get(cache_key)

   IF cached_result:
       result = deserialize(cached_result)
       RETURN result  ✅ Cache HIT (latency: ~5ms)

5. Add cache headers
   Cache-Control: max-age=300
   X-Cache-Status: HIT

Total latency: ~10ms (vs 15s DB query)
```

#### **Cache Miss (5% des cas, cold start)**

```
1-3. [Same as above]

4. Check Redis
   cached_result = redis.get(cache_key)

   IF NOT cached_result:  ❌ Cache MISS

5. Query Database
   result = db.execute(sql, params)
   Latency: ~15 seconds

6. Serialize result
   serialized = json.dumps(result)
   Compressed: gzip(serialized)  # Optional: reduce size 70%

7. Store in Redis
   redis.setex(cache_key, 300, serialized)  # TTL = 5 min

8. Return result
   RETURN result

9. Add cache headers
   Cache-Control: max-age=300
   X-Cache-Status: MISS

Total latency: ~15s (first time)
Subsequent requests: ~5ms (cache hit)
```

### 3.3 Décisions architecturales clés

#### **Choix 1 : Cache-Aside vs Cache-Through**

**Cache-Aside (choisi)** :
```python
# Application contrôle cache explicitement
def get_data(query):
    cached = redis.get(key)
    if cached:
        return cached

    result = db.query(query)
    redis.set(key, result, ex=300)
    return result
```

**Cache-Through (alternative)** :
```python
# Cache gère automatiquement
def get_data(query):
    # Cache intercepte et gère
    return cache.get_or_load(key, lambda: db.query(query))
```

**Trade-off assumé** :
- ➕ Contrôle explicite de l'invalidation
- ➕ Flexibilité (TTL par query)
- ➕ Failure mode : Fail to DB (safe)
- ➖ Code légèrement plus verbeux

---

#### **Choix 2 : Hashing de requêtes pour clés**

**Approche naïve** :
```python
# ❌ SQL comme clé directe (problème : trop long)
key = f"query:{sql}"
# Limite Redis : 512 MB par key, mais clé elle-même < 1KB recommandé
```

**Approche choisie** :
```python
# ✅ Hash de SQL + params
import hashlib
import json

def generate_cache_key(sql: str, params: dict) -> str:
    # Normaliser SQL (whitespace, case)
    normalized_sql = ' '.join(sql.split()).lower()

    # Inclure params pour unicité
    key_data = f"{normalized_sql}:{json.dumps(params, sort_keys=True)}"

    # Hash SHA256 (collision impossible en pratique)
    hash_hex = hashlib.sha256(key_data.encode()).hexdigest()

    return f"query:{hash_hex[:16]}"  # 16 chars suffisent

# Exemple
sql = "SELECT * FROM orders WHERE created_at > %(date)s"
params = {"date": "2024-01-01"}
key = generate_cache_key(sql, params)
# → "query:a3f5b9c2d8e1f4a7"
```

**Trade-off assumé** :
- ➕ Clés courtes et consistantes
- ➕ Collision impossible (SHA256)
- ➖ Debugging légèrement plus difficile (pas de SQL visible)

---

#### **Choix 3 : Sérialisation avec compression**

**Benchmark sérialisation** :

```python
import json
import pickle
import msgpack
import gzip

result = [{"id": i, "name": f"Product {i}", "price": 99.99} for i in range(10000)]

# JSON (baseline)
json_size = len(json.dumps(result))  # 450 KB
json_time = 50 ms  # encode + decode

# Pickle (Python-specific)
pickle_size = len(pickle.dumps(result))  # 420 KB
pickle_time = 30 ms

# MessagePack (binary, cross-language)
msgpack_size = len(msgpack.packb(result))  # 380 KB
msgpack_time = 25 ms

# JSON + gzip
compressed = gzip.compress(json.dumps(result).encode())
compressed_size = len(compressed)  # 135 KB (70% reduction!)
compressed_time = 80 ms  # encode + compress + decompress + decode
```

**Choix : JSON + gzip pour résultats > 100KB**

```python
def serialize(data):
    json_str = json.dumps(data)

    if len(json_str) > 102400:  # 100 KB threshold
        compressed = gzip.compress(json_str.encode())
        return b"gzip:" + compressed  # Prefix pour identifier

    return json_str.encode()

def deserialize(data):
    if data.startswith(b"gzip:"):
        decompressed = gzip.decompress(data[5:])
        return json.loads(decompressed.decode())

    return json.loads(data.decode())
```

**Trade-off assumé** :
- ➕ Taille ÷ 3 (réduit network + memory)
- ➕ JSON = human-readable debug
- ➖ CPU +60% pour compression (acceptable)

---

## 4. Implémentation technique

### 4.1 Code Python (Production-Ready)

```python
"""
SQL Query Cache avec Redis
Implémentation production-ready avec cache-aside pattern
"""

import time
import hashlib
import json
import gzip
from typing import Any, Dict, List, Optional, Callable
from dataclasses import dataclass
from functools import wraps
import logging

import redis
import psycopg2
from psycopg2.extras import RealDictCursor

# Configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'db': 0,
    'decode_responses': False,  # Binary data (gzip)
    'socket_timeout': 3,
    'max_connections': 50
}

DB_CONFIG = {
    'host': 'localhost',
    'port': 5432,
    'database': 'analytics',
    'user': 'app_user',
    'password': 'secure_password',
    'cursor_factory': RealDictCursor
}

# Constants
DEFAULT_TTL = 300  # 5 minutes
COMPRESSION_THRESHOLD = 102400  # 100 KB
CACHE_STAMPEDE_LOCK_TTL = 30  # 30 seconds


# ============================================================================
# Data Classes
# ============================================================================

@dataclass
class CacheStats:
    """Statistiques de cache"""
    hits: int
    misses: int
    hit_ratio: float
    avg_query_time_ms: float
    avg_cache_time_ms: float


# ============================================================================
# Query Cache
# ============================================================================

class QueryCache:
    """
    Cache de requêtes SQL avec Redis

    Features:
    - Cache-aside pattern
    - Compression automatique (gzip)
    - Cache stampede protection
    - Tag-based invalidation
    - TTL per-query configurable
    - Metrics et monitoring
    """

    def __init__(self, redis_config: Dict = None, db_config: Dict = None):
        # Redis connexion
        redis_conf = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**redis_conf)

        # Database connexion pool
        self.db_config = db_config or DB_CONFIG

        # Metrics
        self.stats = {
            'hits': 0,
            'misses': 0,
            'total_query_time': 0.0,
            'total_cache_time': 0.0,
            'query_count': 0
        }

        # Test connexions
        try:
            self.redis.ping()
            logger.info("QueryCache initialized")
        except redis.RedisError as e:
            logger.error(f"Redis connection failed: {e}")
            raise

    # ========================================================================
    # Core Cache Operations
    # ========================================================================

    def generate_cache_key(
        self,
        sql: str,
        params: Optional[Dict] = None,
        prefix: str = "query"
    ) -> str:
        """
        Générer clé de cache unique pour requête

        Args:
            sql: Requête SQL
            params: Paramètres de la requête
            prefix: Préfixe de la clé

        Returns:
            Cache key (ex: "query:a3f5b9c2d8e1f4a7")
        """
        # Normaliser SQL
        normalized_sql = ' '.join(sql.split()).lower().strip()

        # Inclure params
        if params:
            params_str = json.dumps(params, sort_keys=True)
            key_data = f"{normalized_sql}:{params_str}"
        else:
            key_data = normalized_sql

        # Hash SHA256
        hash_hex = hashlib.sha256(key_data.encode()).hexdigest()

        return f"{prefix}:{hash_hex[:16]}"

    def _serialize(self, data: Any) -> bytes:
        """Sérialiser données avec compression si nécessaire"""
        json_str = json.dumps(data, default=str)  # default=str pour dates
        json_bytes = json_str.encode('utf-8')

        # Compression si > 100KB
        if len(json_bytes) > COMPRESSION_THRESHOLD:
            compressed = gzip.compress(json_bytes, compresslevel=6)
            logger.debug(
                f"Compressed: {len(json_bytes)} → {len(compressed)} bytes "
                f"({100 * (1 - len(compressed)/len(json_bytes)):.1f}% reduction)"
            )
            return b"gzip:" + compressed

        return json_bytes

    def _deserialize(self, data: bytes) -> Any:
        """Désérialiser données avec décompression si nécessaire"""
        if data.startswith(b"gzip:"):
            decompressed = gzip.decompress(data[5:])
            return json.loads(decompressed.decode('utf-8'))

        return json.loads(data.decode('utf-8'))

    def get(self, cache_key: str) -> Optional[Any]:
        """Récupérer résultat du cache"""
        start_time = time.time()

        try:
            cached = self.redis.get(cache_key)

            elapsed = (time.time() - start_time) * 1000

            if cached:
                self.stats['hits'] += 1
                self.stats['total_cache_time'] += elapsed
                logger.debug(f"Cache HIT: {cache_key} ({elapsed:.2f}ms)")
                return self._deserialize(cached)
            else:
                self.stats['misses'] += 1
                logger.debug(f"Cache MISS: {cache_key}")
                return None

        except redis.RedisError as e:
            logger.error(f"Cache get error: {e}")
            return None

    def set(
        self,
        cache_key: str,
        value: Any,
        ttl: int = DEFAULT_TTL,
        tags: Optional[List[str]] = None
    ):
        """
        Stocker résultat dans le cache

        Args:
            cache_key: Clé de cache
            value: Valeur à cacher
            ttl: Time-to-live en secondes
            tags: Tags pour invalidation sélective
        """
        try:
            serialized = self._serialize(value)

            # Store main cache
            self.redis.setex(cache_key, ttl, serialized)

            # Store tags (if provided)
            if tags:
                for tag in tags:
                    tag_key = f"tag:{tag}"
                    self.redis.sadd(tag_key, cache_key)
                    self.redis.expire(tag_key, ttl)

            logger.debug(f"Cached: {cache_key} (TTL: {ttl}s, Size: {len(serialized)} bytes)")

        except redis.RedisError as e:
            logger.error(f"Cache set error: {e}")

    # ========================================================================
    # Query Execution with Caching
    # ========================================================================

    def query(
        self,
        sql: str,
        params: Optional[Dict] = None,
        ttl: int = DEFAULT_TTL,
        tags: Optional[List[str]] = None,
        force_refresh: bool = False
    ) -> List[Dict]:
        """
        Exécuter requête avec cache

        Args:
            sql: Requête SQL
            params: Paramètres
            ttl: TTL du cache
            tags: Tags pour invalidation
            force_refresh: Bypass cache

        Returns:
            Liste de résultats (list of dict)
        """
        cache_key = self.generate_cache_key(sql, params)

        # Check cache (sauf si force_refresh)
        if not force_refresh:
            cached_result = self.get(cache_key)
            if cached_result is not None:
                return cached_result

        # Cache miss → Query database
        start_time = time.time()

        try:
            result = self._execute_sql(sql, params)

            query_time = (time.time() - start_time) * 1000
            self.stats['total_query_time'] += query_time
            self.stats['query_count'] += 1

            logger.info(f"Query executed: {query_time:.2f}ms ({len(result)} rows)")

            # Store in cache
            self.set(cache_key, result, ttl, tags)

            return result

        except Exception as e:
            logger.error(f"Query execution error: {e}")
            raise

    def query_with_stampede_protection(
        self,
        sql: str,
        params: Optional[Dict] = None,
        ttl: int = DEFAULT_TTL,
        tags: Optional[List[str]] = None
    ) -> List[Dict]:
        """
        Exécuter requête avec protection cache stampede

        Utilise un lock Redis pour éviter N requêtes simultanées
        """
        cache_key = self.generate_cache_key(sql, params)

        # Check cache
        cached_result = self.get(cache_key)
        if cached_result is not None:
            return cached_result

        # Lock pour éviter stampede
        lock_key = f"{cache_key}:lock"
        lock_acquired = self.redis.set(
            lock_key,
            "1",
            ex=CACHE_STAMPEDE_LOCK_TTL,
            nx=True  # Set if not exists
        )

        if lock_acquired:
            # Ce thread exécute la requête
            try:
                result = self.query(sql, params, ttl, tags, force_refresh=True)
                return result
            finally:
                self.redis.delete(lock_key)
        else:
            # Attendre que le lock soit libéré (autre thread query DB)
            logger.debug(f"Waiting for lock: {lock_key}")

            for _ in range(10):  # Max 3 secondes d'attente
                time.sleep(0.3)

                # Re-check cache
                cached_result = self.get(cache_key)
                if cached_result is not None:
                    return cached_result

            # Timeout → Execute anyway
            logger.warning("Lock timeout, executing query anyway")
            return self.query(sql, params, ttl, tags, force_refresh=True)

    def _execute_sql(self, sql: str, params: Optional[Dict] = None) -> List[Dict]:
        """Exécuter SQL sur PostgreSQL"""
        conn = psycopg2.connect(**self.db_config)

        try:
            with conn.cursor() as cursor:
                cursor.execute(sql, params)

                # Fetch all as list of dicts
                rows = cursor.fetchall()
                return [dict(row) for row in rows]
        finally:
            conn.close()

    # ========================================================================
    # Invalidation
    # ========================================================================

    def invalidate(self, cache_key: str) -> bool:
        """Invalider une clé spécifique"""
        try:
            deleted = self.redis.delete(cache_key)
            logger.info(f"Invalidated: {cache_key}")
            return deleted > 0
        except redis.RedisError as e:
            logger.error(f"Invalidation error: {e}")
            return False

    def invalidate_by_tag(self, tag: str) -> int:
        """
        Invalider tous les caches avec un tag spécifique

        Args:
            tag: Tag à invalider (ex: "revenue", "orders")

        Returns:
            Nombre de clés invalidées
        """
        tag_key = f"tag:{tag}"

        try:
            # Récupérer toutes les clés avec ce tag
            cache_keys = self.redis.smembers(tag_key)

            if not cache_keys:
                logger.debug(f"No keys found for tag: {tag}")
                return 0

            # Delete toutes les clés
            deleted = self.redis.delete(*cache_keys)

            # Delete tag set
            self.redis.delete(tag_key)

            logger.info(f"Invalidated {deleted} keys for tag: {tag}")
            return deleted

        except redis.RedisError as e:
            logger.error(f"Tag invalidation error: {e}")
            return 0

    def invalidate_pattern(self, pattern: str) -> int:
        """
        Invalider toutes les clés matching un pattern

        Args:
            pattern: Pattern Redis (ex: "query:revenue:*")

        Returns:
            Nombre de clés invalidées
        """
        cursor = 0
        deleted = 0

        try:
            while True:
                cursor, keys = self.redis.scan(cursor, match=pattern, count=100)

                if keys:
                    deleted += self.redis.delete(*keys)

                if cursor == 0:
                    break

            logger.info(f"Invalidated {deleted} keys matching: {pattern}")
            return deleted

        except redis.RedisError as e:
            logger.error(f"Pattern invalidation error: {e}")
            return 0

    # ========================================================================
    # Cache Warming
    # ========================================================================

    def warm_up(self, queries: List[Dict]):
        """
        Pre-populate cache avec requêtes courantes

        Args:
            queries: Liste de {"sql": str, "params": dict, "ttl": int, "tags": list}
        """
        logger.info(f"Warming up cache with {len(queries)} queries...")

        warmed = 0
        for query_def in queries:
            try:
                self.query(
                    sql=query_def['sql'],
                    params=query_def.get('params'),
                    ttl=query_def.get('ttl', DEFAULT_TTL),
                    tags=query_def.get('tags'),
                    force_refresh=True
                )
                warmed += 1
            except Exception as e:
                logger.error(f"Warm-up failed for query: {e}")

        logger.info(f"Cache warmed: {warmed}/{len(queries)} queries")
        return warmed

    # ========================================================================
    # Monitoring
    # ========================================================================

    def get_stats(self) -> CacheStats:
        """Récupérer statistiques de cache"""
        total_requests = self.stats['hits'] + self.stats['misses']
        hit_ratio = self.stats['hits'] / total_requests if total_requests > 0 else 0.0

        avg_query_time = (
            self.stats['total_query_time'] / self.stats['query_count']
            if self.stats['query_count'] > 0 else 0.0
        )

        avg_cache_time = (
            self.stats['total_cache_time'] / self.stats['hits']
            if self.stats['hits'] > 0 else 0.0
        )

        return CacheStats(
            hits=self.stats['hits'],
            misses=self.stats['misses'],
            hit_ratio=hit_ratio,
            avg_query_time_ms=avg_query_time,
            avg_cache_time_ms=avg_cache_time
        )

    def reset_stats(self):
        """Reset des statistiques"""
        self.stats = {
            'hits': 0,
            'misses': 0,
            'total_query_time': 0.0,
            'total_cache_time': 0.0,
            'query_count': 0
        }


# ============================================================================
# Decorator
# ============================================================================

def cached_query(ttl: int = DEFAULT_TTL, tags: Optional[List[str]] = None):
    """
    Decorator pour cacher automatiquement une fonction de requête

    Usage:
        @cached_query(ttl=600, tags=["revenue"])
        def get_revenue_report(start_date, end_date):
            sql = "SELECT ... WHERE date BETWEEN %(start)s AND %(end)s"
            return execute_sql(sql, {"start": start_date, "end": end_date})
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            cache = QueryCache()

            # Generate cache key from function name + args
            func_signature = f"{func.__name__}:{args}:{sorted(kwargs.items())}"
            cache_key = cache.generate_cache_key(func_signature, prefix="func")

            # Check cache
            cached = cache.get(cache_key)
            if cached is not None:
                return cached

            # Execute function
            result = func(*args, **kwargs)

            # Store in cache
            cache.set(cache_key, result, ttl, tags)

            return result

        return wrapper
    return decorator


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    # Initialize cache
    cache = QueryCache()

    print("\n💾 SQL Query Cache Demo\n")

    # Exemple de requête
    sql_revenue = """
    SELECT
        DATE(created_at) AS date,
        SUM(total_amount) AS revenue,
        COUNT(*) AS order_count
    FROM orders
    WHERE created_at >= %(start_date)s
        AND status = 'completed'
    GROUP BY DATE(created_at)
    ORDER BY date DESC
    LIMIT 30
    """

    params = {"start_date": "2024-11-01"}

    # First call (cache miss)
    print("🔍 First call (cache miss)...")
    start = time.time()
    result1 = cache.query(
        sql_revenue,
        params,
        ttl=300,
        tags=["revenue", "orders"]
    )
    time1 = (time.time() - start) * 1000
    print(f"   ⏱️  Time: {time1:.2f}ms")
    print(f"   📊 Rows: {len(result1)}")

    # Second call (cache hit)
    print("\n🔍 Second call (cache hit)...")
    start = time.time()
    result2 = cache.query(sql_revenue, params)
    time2 = (time.time() - start) * 1000
    print(f"   ⏱️  Time: {time2:.2f}ms")
    print(f"   ⚡ Speedup: {time1/time2:.1f}x faster")

    # Stats
    print("\n📈 Cache Statistics:")
    stats = cache.get_stats()
    print(f"   Hit ratio: {stats.hit_ratio:.2%}")
    print(f"   Hits: {stats.hits}")
    print(f"   Misses: {stats.misses}")
    print(f"   Avg query time: {stats.avg_query_time_ms:.2f}ms")
    print(f"   Avg cache time: {stats.avg_cache_time_ms:.2f}ms")

    # Invalidation by tag
    print("\n🗑️  Invalidating 'revenue' tag...")
    deleted = cache.invalidate_by_tag("revenue")
    print(f"   Deleted {deleted} keys")

    # Decorator example
    print("\n🎨 Decorator Example:")

    @cached_query(ttl=600, tags=["products"])
    def get_top_products(limit=10):
        return [
            {"name": f"Product {i}", "sales": 1000 - i*10}
            for i in range(limit)
        ]

    # First call
    start = time.time()
    products1 = get_top_products(5)
    time1 = (time.time() - start) * 1000
    print(f"   First call: {time1:.2f}ms")

    # Second call (cached)
    start = time.time()
    products2 = get_top_products(5)
    time2 = (time.time() - start) * 1000
    print(f"   Second call: {time2:.2f}ms (cached)")
    print(f"   Speedup: {time1/time2:.1f}x")
```

---

## 5. Cas avancés et optimisations

### 5.1 Cache Warm-up automatique

```python
class CacheWarmer:
    """Warmer automatique de cache au démarrage"""

    def __init__(self, cache: QueryCache):
        self.cache = cache

    def warm_popular_queries(self):
        """Pre-load les requêtes les plus populaires"""
        popular_queries = [
            {
                "sql": "SELECT * FROM dashboard_revenue WHERE period = %(period)s",
                "params": {"period": "30d"},
                "ttl": 300,
                "tags": ["revenue"]
            },
            {
                "sql": "SELECT * FROM dashboard_users WHERE active = true",
                "params": None,
                "ttl": 600,
                "tags": ["users"]
            },
            # ... autres queries populaires
        ]

        return self.cache.warm_up(popular_queries)

    def schedule_warming(self, interval_minutes: int = 30):
        """Scheduler warm-up périodique"""
        import schedule

        schedule.every(interval_minutes).minutes.do(self.warm_popular_queries)

        while True:
            schedule.run_pending()
            time.sleep(60)
```

### 5.2 Invalidation event-driven

```python
# Event handlers pour invalidation automatique
def on_order_created(order_id):
    """Event: Nouvelle commande créée"""
    cache.invalidate_by_tag("revenue")
    cache.invalidate_by_tag("orders")
    logger.info(f"Cache invalidated after order creation: {order_id}")

def on_product_updated(product_id):
    """Event: Produit modifié"""
    cache.invalidate_by_tag("products")
    cache.invalidate_pattern(f"query:product:{product_id}:*")
```

### 5.3 Adaptive TTL basé sur query cost

```python
def calculate_adaptive_ttl(query_time_ms: float) -> int:
    """
    TTL adaptatif basé sur le coût de la requête

    Query lente = TTL long (moins de recomputation)
    Query rapide = TTL court (données plus fresh)
    """
    if query_time_ms > 10000:  # > 10s
        return 1800  # 30 min
    elif query_time_ms > 5000:  # > 5s
        return 600   # 10 min
    elif query_time_ms > 1000:  # > 1s
        return 300   # 5 min
    else:
        return 60    # 1 min

# Usage
start = time.time()
result = execute_query(sql)
query_time = (time.time() - start) * 1000

ttl = calculate_adaptive_ttl(query_time)
cache.set(cache_key, result, ttl)
```

---

## 6. Monitoring et métriques

### 6.1 KPIs critiques

```yaml
# Dashboard Grafana

# 1. Cache hit ratio
cache_hit_ratio:
  target: > 80%
  warning: < 70%
  critical: < 50%

# 2. Cache vs DB latency
cache_latency_ms:
  p50: < 2ms
  p95: < 5ms
  p99: < 10ms

db_query_latency_ms:
  p50: 2-10s
  p95: 10-30s

# 3. Cache size
cache_memory_mb:
  current: 2-5 GB
  max: 8 GB

cache_keys_count:
  typical: 10k-50k keys

# 4. Invalidation rate
cache_invalidations_per_min:
  normal: 10-100
  high: > 500

# 5. Database load reduction
db_queries_per_sec:
  before_cache: 1000/s
  after_cache: 50/s
  reduction: 95%
```

### 6.2 Alertes

```yaml
# Prometheus Alerts

groups:
  - name: query_cache_alerts
    rules:

      # Hit ratio faible
      - alert: QueryCacheLowHitRatio
        expr: cache_hits / (cache_hits + cache_misses) < 0.70
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Query cache hit ratio < 70%"

      # Cache size critique
      - alert: QueryCacheSizeCritical
        expr: redis_memory_used_bytes{instance="query-cache"} > 7500000000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Query cache using > 7.5GB (max 8GB)"

      # Stampede détecté
      - alert: QueryCacheStampedeDetected
        expr: rate(cache_lock_acquired[1m]) > 100
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Cache stampede: > 100 locks/min"
```

---

## 7. Conclusion

### Points clés à retenir

- ✅ **Cache-Aside = Pattern le plus flexible** pour queries SQL
- ✅ **Performance × 3000** (15s → 5ms après cache)
- ✅ **Charge DB ÷ 100** (95% de reduction)
- ✅ **Compression** : Taille ÷ 3 avec gzip (> 100KB)
- ✅ **Stampede protection** : Lock Redis évite N queries simultanées
- ✅ **Tag-based invalidation** : Invalidation sélective et précise
- ✅ **TTL automatique** : Pas de memory leak
- ✅ **Adaptive TTL** : Query lente = cache plus longtemps

### Quand NE PAS cacher

- ❌ **Données transactionnelles critiques** : Solde bancaire, inventory
- ❌ **Données change en temps réel** : Stock exchange, betting
- ❌ **Requêtes ultra-rapides** (< 10ms) : Overhead cache > gain
- ❌ **Données hautement personnalisées** : Chaque user unique
- ❌ **Audit trail requis** : Logs complets nécessaires

### Comparaison finale

| Critère | No Cache | Materialized View | App Memory | Redis Cache |
|---------|----------|-------------------|------------|-------------|
| Latence | 5-30s | 50-500ms | < 1ms | **5-10ms** |
| Distribution | N/A | DB-level | ❌ Per-instance | ✅ Global |
| Invalidation | N/A | Manual | Manual | **Flexible** |
| Dynamic params | ✅ Yes | ❌ Limited | ✅ Yes | ✅ Yes |
| Memory cost | N/A | DB storage | High | **Moderate** |

### Prochaines lectures

- [Cache Patterns](/06-patterns-developpement-avances/01-caching-patterns.md) → Write-Through, Write-Back
- [Cache Stampede](/06-patterns-developpement-avances/02-strategies-anti-problemes.md) → Solutions avancées
- [Client-Side Caching](/06-patterns-developpement-avances/04-client-side-caching.md) → Niveau suivant

---

**📚 Ressources complémentaires** :
- [Cache-Aside Pattern (Microsoft)](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [Redis Best Practices](https://redis.io/docs/manual/patterns/)
- [Query Result Caching](https://www.percona.com/blog/mysql-query-cache-performance/)

⏭️ [Cas #7 : Recommendation Engine avec Vector Search](/16-etudes-cas-patterns-reels/07-cas-recommendation-engine-vector-search.md)
