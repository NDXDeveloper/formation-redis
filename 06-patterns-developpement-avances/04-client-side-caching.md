🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4 Client-Side Caching (Tracking) : La fonctionnalité killer

## Introduction

Le **Client-Side Caching** est l'une des fonctionnalités les plus puissantes introduites dans Redis 6.0. Elle permet au client de maintenir un cache local en mémoire et de recevoir des notifications d'invalidation lorsque les données changent sur le serveur. Cette technique peut réduire la latence à pratiquement zéro et diviser la charge réseau par 100.

Malgré son potentiel énorme, cette fonctionnalité reste méconnue. Cette section explore en profondeur son fonctionnement et comment l'exploiter.

## Le problème résolu par le Client-Side Caching

### Scénario classique avec Redis

```
Architecture traditionnelle (même avec Redis):
────────────────────────────────────────────────────────

User Request → Application Server
                     ↓
                 Check Redis  ← [Network RTT: 1-5ms]
                     ↓
                 Return data

Même avec Redis, chaque lecture nécessite un RTT réseau!

Performance:
- Same datacenter: 1-5ms par requête
- Cross-region: 50-200ms par requête
- 1000 req/s = 1000-5000ms de latence réseau cumulée
```

### Avec Client-Side Caching

```
Architecture avec Client-Side Caching:
────────────────────────────────────────────────────────

User Request → Application Server
                     ↓
                 Check LOCAL cache  ← [Memory: 0.001ms]
                     ↓
                 Return data

Seule la PREMIÈRE requête touche Redis.
Les suivantes lisent depuis la mémoire locale!

Performance:
- Local memory: 0.001-0.01ms (1000x plus rapide!)
- 1000 req/s = 1ms de latence totale (vs 5000ms)
- Redis notifie si les données changent → cache invalidé

Speedup: 100-1000x sur les lectures répétées! 🚀
```

### Visualisation de l'impact

```
WITHOUT Client-Side Caching:
────────────────────────────────────────────────────────
Request 1: App → Redis [5ms] → user:123 data
Request 2: App → Redis [5ms] → user:123 data (same!)
Request 3: App → Redis [5ms] → user:123 data (same!)
Request 4: App → Redis [5ms] → user:123 data (same!)
...
Request 100: App → Redis [5ms] → user:123 data

Total time: 100 × 5ms = 500ms
Network overhead: 100%


WITH Client-Side Caching:
────────────────────────────────────────────────────────
Request 1: App → Redis [5ms] → user:123 data ✓ Cache it locally
Request 2: App → Local [0.01ms] → user:123 data (from cache)
Request 3: App → Local [0.01ms] → user:123 data (from cache)
...
Request 100: App → Local [0.01ms] → user:123 data (from cache)

[Meanwhile on Redis: user:123 updated]
Redis → App: INVALIDATE "user:123" ← Async notification

Request 101: App → Redis [5ms] → user:123 NEW data ✓ Cache it

Total time: 1 × 5ms + 99 × 0.01ms = 6ms
Network overhead: 0.2%

Speedup: 83x faster! 🚀
```

---

## Comment fonctionne le Client-Side Caching

### Architecture RESP3

Redis 6.0 a introduit le protocole RESP3 qui supporte les **Push Messages** : le serveur peut envoyer des messages au client sans que celui-ci ne les demande.

```
┌─────────────────────────────────────────────────────────────┐
│              CLIENT-SIDE CACHING ARCHITECTURE               │
└─────────────────────────────────────────────────────────────┘

CLIENT APPLICATION
     │
     ├── Local Cache (dict/map)
     │   ├─ "user:123" → {"name": "Alice", ...}
     │   ├─ "product:456" → {"price": 99.99, ...}
     │   └─ "config:settings" → {...}
     │
     ↓
┌────────────────────────────────────────────────────────────┐
│                     REDIS CLIENT                           │
│                                                            │
│  ┌─────────────────┐              ┌────────────────────┐   │
│  │ Main Connection │              │ Push Connection    │   │
│  │ (RESP3)         │              │ (Subscribes to     │   │
│  │                 │              │  invalidations)    │   │
│  └─────────────────┘              └────────────────────┘   │
│         │                                    ↑             │
│         │                                    │             │
└─────────┼────────────────────────────────────┼─────────────┘
          │                                    │
          ↓                                    │
═════════════════════════════════════════════════════════════
                       NETWORK
═════════════════════════════════════════════════════════════
          │                                    │
          ↓                                    │
┌────────────────────────────────────────────────────────────┐
│                     REDIS SERVER                           │
│                                                            │
│  ┌─────────────────┐              ┌────────────────────┐   │
│  │ Command Handler │              │ Tracking Table     │   │
│  │                 │              │                    │   │
│  │ GET user:123 ──→│─────────────→│ Track: client X    │   │
│  │                 │              │   watching user:123│   │
│  └─────────────────┘              └────────────────────┘   │
│         │                                    │             │
│         │                                    │             │
│         ↓                                    ↓             │
│  ┌─────────────────┐              ┌────────────────────┐   │
│  │ SET user:123 ──→│─────────────→│ INVALIDATE         │   │
│  │                 │              │ → Notify client X  │   │
│  └─────────────────┘              └────────────────────┘   │
│                                           │                │
└───────────────────────────────────────────┼────────────────┘
                                            │
                                            ↓
                                    PUSH: invalidate user:123
                                            │
                                            ↓
                            CLIENT: Clear local cache for user:123
```

### Flux détaillé

```
STEP 1: ENABLE TRACKING
────────────────────────────────────────────────────────
Client → Redis: CLIENT TRACKING ON REDIRECT <client-id>
Redis → Client: OK

Redis commence à tracker les clés lues par ce client


STEP 2: FIRST READ (Cache MISS)
────────────────────────────────────────────────────────
Client → Redis: GET user:123
Redis:
  1. Exécute GET user:123
  2. Note: "client X is tracking user:123"
  3. Retourne: {"name": "Alice"}

Client:
  1. Reçoit {"name": "Alice"}
  2. Stocke en cache local: cache["user:123"] = data


STEP 3: SUBSEQUENT READS (Cache HIT)
────────────────────────────────────────────────────────
Client:
  1. Check local cache for "user:123"
  2. HIT! Return cached data (no network call)

Latency: 0.001ms (vs 5ms with Redis)


STEP 4: DATA CHANGES ON REDIS
────────────────────────────────────────────────────────
Another Client → Redis: SET user:123 {"name": "Bob"}
Redis:
  1. Exécute SET
  2. Check tracking table: "client X is watching user:123"
  3. Send PUSH message to client X: INVALIDATE ["user:123"]

Client X (Push Connection):
  1. Receives PUSH: invalidate user:123
  2. Removes "user:123" from local cache
  3. Next GET will be a cache MISS → fetch from Redis


STEP 5: NEXT READ AFTER INVALIDATION (Cache MISS)
────────────────────────────────────────────────────────
Client → Redis: GET user:123
Redis → Client: {"name": "Bob"}  (new data)
Client: Store in local cache again

The cycle repeats!
```

---

## Implémentation Python

### Configuration de base

```python
import redis
import json
import time
from typing import Dict, Any, Optional

class ClientSideCacheRedis:
    """
    Redis client avec Client-Side Caching
    """

    def __init__(self, host='localhost', port=6379):
        # Connexion principale (RESP3)
        self.redis = redis.Redis(
            host=host,
            port=port,
            protocol=3,  # RESP3 requis!
            decode_responses=True
        )

        # Cache local en mémoire
        self.local_cache: Dict[str, Any] = {}

        # Connexion pour les push messages
        self.pubsub = None

        # Statistiques
        self.stats = {
            'local_hits': 0,
            'local_misses': 0,
            'invalidations': 0
        }

        # Activer le tracking
        self._enable_tracking()

    def _enable_tracking(self):
        """
        Active le Client-Side Caching avec tracking
        """
        try:
            # CLIENT TRACKING ON: Active le tracking
            # BCAST: Mode broadcast (optionnel)
            # NOLOOP: Ne pas recevoir nos propres invalidations
            result = self.redis.execute_command(
                'CLIENT', 'TRACKING', 'ON',
                'NOLOOP'
            )

            print(f"✓ Client tracking enabled: {result}")

            # Démarrer le listener d'invalidations
            self._start_invalidation_listener()

        except redis.exceptions.ResponseError as e:
            print(f"⚠️  Tracking not available: {e}")
            print("Make sure you're using Redis 6.0+ with RESP3")

    def _start_invalidation_listener(self):
        """
        Démarre un thread pour écouter les invalidations
        """
        import threading

        def listen_for_invalidations():
            # Utiliser une connexion pub/sub pour les push messages
            pubsub = self.redis.pubsub()

            # En mode RESP3, les invalidations arrivent comme push messages
            # On utilise listen() pour les recevoir
            for message in pubsub.listen():
                if message['type'] == 'invalidate':
                    # Message d'invalidation reçu
                    keys = message.get('data', [])
                    self._handle_invalidation(keys)

        # Lancer dans un thread daemon
        listener_thread = threading.Thread(
            target=listen_for_invalidations,
            daemon=True
        )
        listener_thread.start()

        print("✓ Invalidation listener started")

    def _handle_invalidation(self, keys):
        """
        Gère les messages d'invalidation
        """
        if keys:
            for key in keys:
                if key in self.local_cache:
                    del self.local_cache[key]
                    self.stats['invalidations'] += 1
                    print(f"🔄 Invalidated cache for: {key}")

    def get(self, key: str) -> Optional[Any]:
        """
        GET avec Client-Side Caching
        """
        # 1. Vérifier le cache local
        if key in self.local_cache:
            self.stats['local_hits'] += 1
            print(f"✓ LOCAL cache hit: {key}")
            return self.local_cache[key]

        # 2. Cache MISS - Aller sur Redis
        self.stats['local_misses'] += 1
        print(f"✗ LOCAL cache miss: {key}")

        value = self.redis.get(key)

        if value:
            # Stocker en cache local
            try:
                parsed_value = json.loads(value)
                self.local_cache[key] = parsed_value
            except json.JSONDecodeError:
                self.local_cache[key] = value

        return self.local_cache.get(key)

    def set(self, key: str, value: Any) -> bool:
        """
        SET classique (invalidera le cache si trackée)
        """
        if isinstance(value, (dict, list)):
            value = json.dumps(value)

        result = self.redis.set(key, value)

        # Supprimer du cache local si présent
        # (L'invalidation viendra aussi du serveur)
        if key in self.local_cache:
            del self.local_cache[key]

        return result

    def get_stats(self) -> Dict[str, Any]:
        """
        Retourne les statistiques du cache
        """
        total = self.stats['local_hits'] + self.stats['local_misses']
        hit_rate = (self.stats['local_hits'] / total * 100) if total > 0 else 0

        return {
            'local_hits': self.stats['local_hits'],
            'local_misses': self.stats['local_misses'],
            'invalidations': self.stats['invalidations'],
            'hit_rate': f"{hit_rate:.2f}%",
            'cache_size': len(self.local_cache)
        }


# ============================================================
# IMPLÉMENTATION AVANCÉE AVEC redis-py (v5+)
# ============================================================

class AdvancedClientSideCache:
    """
    Implémentation avancée avec gestion complète des invalidations
    """

    def __init__(self, host='localhost', port=6379):
        # Connexion principale RESP3
        self.client = redis.Redis(
            host=host,
            port=port,
            protocol=3,
            decode_responses=True
        )

        # Cache local avec métadonnées
        self.cache = {}
        self.cache_metadata = {}

        # Configuration
        self.max_cache_size = 10000
        self.ttl = 300  # TTL local du cache (5 minutes)

        # Stats
        self.stats = {
            'gets': 0,
            'local_hits': 0,
            'local_misses': 0,
            'redis_hits': 0,
            'redis_misses': 0,
            'invalidations': 0,
            'evictions': 0
        }

        self._setup_tracking()

    def _setup_tracking(self):
        """
        Configure le tracking avec options avancées
        """
        try:
            # CLIENT TRACKING ON REDIRECT <client-id> PREFIX <prefix> BCAST NOLOOP

            # Version simple (recommandée)
            self.client.execute_command(
                'CLIENT', 'TRACKING', 'ON',
                'NOLOOP'
            )

            print("✓ Advanced tracking enabled")

        except Exception as e:
            print(f"⚠️  Could not enable tracking: {e}")

    def get(self, key: str) -> Optional[Any]:
        """
        GET avec cache local et TTL
        """
        self.stats['gets'] += 1

        # 1. Vérifier le cache local
        if key in self.cache:
            # Vérifier le TTL local
            cached_at = self.cache_metadata[key]['cached_at']
            age = time.time() - cached_at

            if age < self.ttl:
                self.stats['local_hits'] += 1
                return self.cache[key]
            else:
                # Expiré localement
                del self.cache[key]
                del self.cache_metadata[key]

        # 2. Cache local MISS - Aller sur Redis
        self.stats['local_misses'] += 1

        value = self.client.get(key)

        if value:
            self.stats['redis_hits'] += 1

            # Parser si JSON
            try:
                parsed = json.loads(value)
            except (json.JSONDecodeError, TypeError):
                parsed = value

            # Stocker en cache local
            self._add_to_cache(key, parsed)

            return parsed
        else:
            self.stats['redis_misses'] += 1
            return None

    def _add_to_cache(self, key: str, value: Any):
        """
        Ajoute une valeur au cache local avec gestion de la taille max
        """
        # Vérifier la taille max
        if len(self.cache) >= self.max_cache_size:
            self._evict_lru()

        self.cache[key] = value
        self.cache_metadata[key] = {
            'cached_at': time.time(),
            'access_count': 0
        }

    def _evict_lru(self):
        """
        Éviction LRU si le cache est plein
        """
        # Trouver la clé la plus ancienne
        oldest_key = min(
            self.cache_metadata.keys(),
            key=lambda k: self.cache_metadata[k]['cached_at']
        )

        del self.cache[oldest_key]
        del self.cache_metadata[oldest_key]
        self.stats['evictions'] += 1

    def mget(self, keys: list) -> list:
        """
        MGET avec cache local
        """
        results = []
        keys_to_fetch = []
        key_indices = {}

        # Vérifier le cache local pour chaque clé
        for i, key in enumerate(keys):
            if key in self.cache:
                results.append(self.cache[key])
                self.stats['local_hits'] += 1
            else:
                results.append(None)
                keys_to_fetch.append(key)
                key_indices[key] = i
                self.stats['local_misses'] += 1

        # Fetch depuis Redis si nécessaire
        if keys_to_fetch:
            redis_values = self.client.mget(keys_to_fetch)

            for key, value in zip(keys_to_fetch, redis_values):
                if value:
                    try:
                        parsed = json.loads(value)
                    except (json.JSONDecodeError, TypeError):
                        parsed = value

                    self._add_to_cache(key, parsed)
                    results[key_indices[key]] = parsed
                    self.stats['redis_hits'] += 1
                else:
                    self.stats['redis_misses'] += 1

        return results

    def clear_cache(self):
        """
        Vide le cache local
        """
        cache_size = len(self.cache)
        self.cache.clear()
        self.cache_metadata.clear()
        print(f"✓ Cleared {cache_size} items from local cache")


# ============================================================
# EXEMPLE D'UTILISATION
# ============================================================

def demo_client_side_caching():
    """
    Démo du Client-Side Caching
    """
    print("=" * 60)
    print("CLIENT-SIDE CACHING DEMO")
    print("=" * 60 + "\n")

    # Créer le client avec CSC
    cache = AdvancedClientSideCache()

    # Préparer des données
    user_data = {
        'id': 123,
        'name': 'Alice',
        'email': 'alice@example.com'
    }

    # SET initial
    print("1. Setting user:123 in Redis...")
    cache.client.set('user:123', json.dumps(user_data))

    # Premier GET - Cache MISS
    print("\n2. First GET (should be a local cache MISS):")
    result = cache.get('user:123')
    print(f"   Result: {result}")

    # Deuxième GET - Cache HIT
    print("\n3. Second GET (should be a local cache HIT):")
    result = cache.get('user:123')
    print(f"   Result: {result}")

    # Plusieurs GETs pour tester la performance
    print("\n4. Running 100 GETs...")
    start = time.time()
    for _ in range(100):
        cache.get('user:123')
    elapsed = time.time() - start
    print(f"   Time for 100 GETs: {elapsed*1000:.2f}ms")
    print(f"   Avg per GET: {elapsed*10:.2f}ms")

    # Statistiques
    print("\n5. Cache Statistics:")
    stats = cache.stats
    for key, value in stats.items():
        print(f"   {key}: {value}")

    total_gets = stats['gets']
    local_hit_rate = (stats['local_hits'] / total_gets * 100) if total_gets > 0 else 0
    print(f"\n   Local Hit Rate: {local_hit_rate:.2f}%")


# ============================================================
# BENCHMARK AVEC ET SANS CSC
# ============================================================

def benchmark_csc():
    """
    Compare les performances avec et sans Client-Side Caching
    """
    print("\n" + "=" * 60)
    print("BENCHMARK: WITH vs WITHOUT Client-Side Caching")
    print("=" * 60 + "\n")

    # Setup
    regular_redis = redis.Redis(decode_responses=True)
    csc_redis = AdvancedClientSideCache()

    # Préparer des données
    test_keys = [f"benchmark:key:{i}" for i in range(100)]
    for key in test_keys:
        regular_redis.set(key, json.dumps({'value': key}))

    # Test 1: Sans CSC (Redis direct)
    print("Test 1: Regular Redis (no CSC)")
    start = time.time()
    for _ in range(10):  # 10 itérations
        for key in test_keys:
            regular_redis.get(key)
    elapsed_no_csc = time.time() - start
    ops_per_sec_no_csc = (100 * 10) / elapsed_no_csc

    print(f"  Time: {elapsed_no_csc*1000:.2f}ms")
    print(f"  Throughput: {ops_per_sec_no_csc:.0f} ops/sec\n")

    # Test 2: Avec CSC
    print("Test 2: With Client-Side Caching")

    # Premier passage pour peupler le cache
    for key in test_keys:
        csc_redis.get(key)

    # Deuxième passage (cache chaud)
    start = time.time()
    for _ in range(10):  # 10 itérations
        for key in test_keys:
            csc_redis.get(key)
    elapsed_with_csc = time.time() - start
    ops_per_sec_with_csc = (100 * 10) / elapsed_with_csc

    print(f"  Time: {elapsed_with_csc*1000:.2f}ms")
    print(f"  Throughput: {ops_per_sec_with_csc:.0f} ops/sec\n")

    # Comparaison
    speedup = elapsed_no_csc / elapsed_with_csc
    print(f"Speedup: {speedup:.1f}x faster with CSC! 🚀")
    print(f"Throughput increase: {(speedup - 1) * 100:.0f}%")


if __name__ == '__main__':
    # Vérifier la version Redis
    r = redis.Redis()
    info = r.info('server')
    version = info['redis_version']

    print(f"Redis version: {version}")

    if version < '6.0.0':
        print("⚠️  Client-Side Caching requires Redis 6.0+")
        print("Please upgrade your Redis server")
    else:
        # Lancer la démo
        demo_client_side_caching()

        # Lancer le benchmark
        benchmark_csc()
```

---

## Implémentation Node.js

### Configuration avec ioredis

```javascript
const Redis = require('ioredis');

class ClientSideCacheRedis {
    constructor(options = {}) {
        this.host = options.host || 'localhost';
        this.port = options.port || 6379;

        // Cache local
        this.cache = new Map();
        this.cacheMetadata = new Map();

        // Configuration
        this.maxCacheSize = options.maxCacheSize || 10000;
        this.ttl = options.ttl || 300000; // 5 minutes en ms

        // Stats
        this.stats = {
            gets: 0,
            localHits: 0,
            localMisses: 0,
            redisHits: 0,
            redisMisses: 0,
            invalidations: 0,
            evictions: 0
        };

        // Initialiser les connexions
        this._setupConnections();
    }

    _setupConnections() {
        // Connexion principale (RESP3)
        this.client = new Redis({
            host: this.host,
            port: this.port,
            enableReadyCheck: true,
            // ioredis supporte RESP3 automatiquement
        });

        // Connexion pour les invalidations
        this.invalidationClient = new Redis({
            host: this.host,
            port: this.port
        });

        this._enableTracking();
    }

    async _enableTracking() {
        try {
            // Activer le tracking avec CLIENT TRACKING ON
            await this.client.call('CLIENT', 'TRACKING', 'ON', 'NOLOOP');

            console.log('✓ Client tracking enabled');

            // Écouter les invalidations
            this._setupInvalidationListener();

        } catch (error) {
            console.error('⚠️  Could not enable tracking:', error.message);
            console.log('Make sure you are using Redis 6.0+');
        }
    }

    _setupInvalidationListener() {
        /**
         * En RESP3, les invalidations arrivent comme messages push
         * ioredis les expose via l'événement 'message'
         */

        // Utiliser un subscriber pour les invalidations
        this.invalidationClient.on('message', (channel, message) => {
            if (channel === '__redis__:invalidate') {
                this._handleInvalidation(JSON.parse(message));
            }
        });

        // S'abonner au channel d'invalidation
        this.invalidationClient.subscribe('__redis__:invalidate');

        console.log('✓ Invalidation listener started');
    }

    _handleInvalidation(keys) {
        if (Array.isArray(keys)) {
            for (const key of keys) {
                if (this.cache.has(key)) {
                    this.cache.delete(key);
                    this.cacheMetadata.delete(key);
                    this.stats.invalidations++;
                    console.log(`🔄 Invalidated cache for: ${key}`);
                }
            }
        }
    }

    async get(key) {
        this.stats.gets++;

        // 1. Vérifier le cache local
        if (this.cache.has(key)) {
            const metadata = this.cacheMetadata.get(key);
            const age = Date.now() - metadata.cachedAt;

            if (age < this.ttl) {
                this.stats.localHits++;
                console.log(`✓ LOCAL cache hit: ${key}`);
                return this.cache.get(key);
            } else {
                // Expiré localement
                this.cache.delete(key);
                this.cacheMetadata.delete(key);
            }
        }

        // 2. Cache local MISS - Aller sur Redis
        this.stats.localMisses++;
        console.log(`✗ LOCAL cache miss: ${key}`);

        const value = await this.client.get(key);

        if (value !== null) {
            this.stats.redisHits++;

            // Parser si JSON
            let parsed;
            try {
                parsed = JSON.parse(value);
            } catch (e) {
                parsed = value;
            }

            // Stocker en cache local
            this._addToCache(key, parsed);

            return parsed;
        } else {
            this.stats.redisMisses++;
            return null;
        }
    }

    _addToCache(key, value) {
        // Vérifier la taille max
        if (this.cache.size >= this.maxCacheSize) {
            this._evictLRU();
        }

        this.cache.set(key, value);
        this.cacheMetadata.set(key, {
            cachedAt: Date.now(),
            accessCount: 0
        });
    }

    _evictLRU() {
        // Trouver la clé la plus ancienne
        let oldestKey = null;
        let oldestTime = Infinity;

        for (const [key, metadata] of this.cacheMetadata.entries()) {
            if (metadata.cachedAt < oldestTime) {
                oldestTime = metadata.cachedAt;
                oldestKey = key;
            }
        }

        if (oldestKey) {
            this.cache.delete(oldestKey);
            this.cacheMetadata.delete(oldestKey);
            this.stats.evictions++;
        }
    }

    async set(key, value) {
        if (typeof value === 'object') {
            value = JSON.stringify(value);
        }

        const result = await this.client.set(key, value);

        // Supprimer du cache local
        if (this.cache.has(key)) {
            this.cache.delete(key);
            this.cacheMetadata.delete(key);
        }

        return result;
    }

    async mget(keys) {
        const results = [];
        const keysToFetch = [];
        const keyIndices = {};

        // Vérifier le cache local
        for (let i = 0; i < keys.length; i++) {
            const key = keys[i];

            if (this.cache.has(key)) {
                results.push(this.cache.get(key));
                this.stats.localHits++;
            } else {
                results.push(null);
                keysToFetch.push(key);
                keyIndices[key] = i;
                this.stats.localMisses++;
            }
        }

        // Fetch depuis Redis si nécessaire
        if (keysToFetch.length > 0) {
            const redisValues = await this.client.mget(...keysToFetch);

            for (let i = 0; i < keysToFetch.length; i++) {
                const key = keysToFetch[i];
                const value = redisValues[i];

                if (value !== null) {
                    let parsed;
                    try {
                        parsed = JSON.parse(value);
                    } catch (e) {
                        parsed = value;
                    }

                    this._addToCache(key, parsed);
                    results[keyIndices[key]] = parsed;
                    this.stats.redisHits++;
                } else {
                    this.stats.redisMisses++;
                }
            }
        }

        return results;
    }

    getStats() {
        const total = this.stats.gets;
        const localHitRate = total > 0
            ? (this.stats.localHits / total * 100).toFixed(2)
            : 0;

        return {
            ...this.stats,
            localHitRate: `${localHitRate}%`,
            cacheSize: this.cache.size
        };
    }

    clearCache() {
        const size = this.cache.size;
        this.cache.clear();
        this.cacheMetadata.clear();
        console.log(`✓ Cleared ${size} items from local cache`);
    }

    async close() {
        await this.client.quit();
        await this.invalidationClient.quit();
    }
}

// ============================================================
// DÉMO
// ============================================================

async function demo() {
    console.log('='.repeat(60));
    console.log('CLIENT-SIDE CACHING DEMO (Node.js)');
    console.log('='.repeat(60) + '\n');

    const cache = new ClientSideCacheRedis();

    // Préparer des données
    const userData = {
        id: 123,
        name: 'Alice',
        email: 'alice@example.com'
    };

    // SET initial
    console.log('1. Setting user:123 in Redis...');
    await cache.set('user:123', userData);

    // Premier GET - Cache MISS
    console.log('\n2. First GET (local cache MISS):');
    let result = await cache.get('user:123');
    console.log('   Result:', result);

    // Deuxième GET - Cache HIT
    console.log('\n3. Second GET (local cache HIT):');
    result = await cache.get('user:123');
    console.log('   Result:', result);

    // Plusieurs GETs
    console.log('\n4. Running 100 GETs...');
    const start = Date.now();
    for (let i = 0; i < 100; i++) {
        await cache.get('user:123');
    }
    const elapsed = Date.now() - start;
    console.log(`   Time for 100 GETs: ${elapsed}ms`);
    console.log(`   Avg per GET: ${(elapsed / 100).toFixed(2)}ms`);

    // Statistiques
    console.log('\n5. Cache Statistics:');
    const stats = cache.getStats();
    Object.entries(stats).forEach(([key, value]) => {
        console.log(`   ${key}: ${value}`);
    });

    await cache.close();
}

// ============================================================
// BENCHMARK
// ============================================================

async function benchmark() {
    console.log('\n' + '='.repeat(60));
    console.log('BENCHMARK: WITH vs WITHOUT Client-Side Caching');
    console.log('='.repeat(60) + '\n');

    // Setup
    const regularRedis = new Redis();
    const cscRedis = new ClientSideCacheRedis();

    // Préparer des données
    const testKeys = Array.from(
        { length: 100 },
        (_, i) => `benchmark:key:${i}`
    );

    for (const key of testKeys) {
        await regularRedis.set(key, JSON.stringify({ value: key }));
    }

    // Test 1: Sans CSC
    console.log('Test 1: Regular Redis (no CSC)');
    let start = Date.now();
    for (let iter = 0; iter < 10; iter++) {
        for (const key of testKeys) {
            await regularRedis.get(key);
        }
    }
    const elapsedNoCsc = Date.now() - start;
    const opsPerSecNoCsc = (100 * 10) / (elapsedNoCsc / 1000);

    console.log(`  Time: ${elapsedNoCsc}ms`);
    console.log(`  Throughput: ${opsPerSecNoCsc.toFixed(0)} ops/sec\n`);

    // Test 2: Avec CSC
    console.log('Test 2: With Client-Side Caching');

    // Premier passage pour peupler le cache
    for (const key of testKeys) {
        await cscRedis.get(key);
    }

    // Deuxième passage (cache chaud)
    start = Date.now();
    for (let iter = 0; iter < 10; iter++) {
        for (const key of testKeys) {
            await cscRedis.get(key);
        }
    }
    const elapsedWithCsc = Date.now() - start;
    const opsPerSecWithCsc = (100 * 10) / (elapsedWithCsc / 1000);

    console.log(`  Time: ${elapsedWithCsc}ms`);
    console.log(`  Throughput: ${opsPerSecWithCsc.toFixed(0)} ops/sec\n`);

    // Comparaison
    const speedup = elapsedNoCsc / elapsedWithCsc;
    console.log(`Speedup: ${speedup.toFixed(1)}x faster with CSC! 🚀`);
    console.log(`Throughput increase: ${((speedup - 1) * 100).toFixed(0)}%`);

    await regularRedis.quit();
    await cscRedis.close();
}

// ============================================================
// PATTERN : CSC avec Express.js
// ============================================================

const express = require('express');

function createExpressAppWithCSC() {
    const app = express();
    const cache = new ClientSideCacheRedis();

    app.get('/user/:id', async (req, res) => {
        const userId = req.params.id;
        const cacheKey = `user:${userId}`;

        try {
            // Utiliser le CSC
            const user = await cache.get(cacheKey);

            if (user) {
                res.json({
                    user,
                    cached: true,
                    stats: cache.getStats()
                });
            } else {
                res.status(404).json({ error: 'User not found' });
            }
        } catch (error) {
            res.status(500).json({ error: error.message });
        }
    });

    app.post('/user/:id', async (req, res) => {
        const userId = req.params.id;
        const cacheKey = `user:${userId}`;
        const userData = req.body;

        try {
            // Écrire dans Redis (invalidera le cache)
            await cache.set(cacheKey, userData);

            res.json({
                success: true,
                user: userData
            });
        } catch (error) {
            res.status(500).json({ error: error.message });
        }
    });

    app.get('/stats', (req, res) => {
        res.json(cache.getStats());
    });

    return { app, cache };
}

// Lancer le serveur
async function startServer() {
    const { app, cache } = createExpressAppWithCSC();

    const PORT = 3000;
    app.listen(PORT, () => {
        console.log(`Server running on http://localhost:${PORT}`);
        console.log('Try:');
        console.log(`  GET  http://localhost:${PORT}/user/123`);
        console.log(`  POST http://localhost:${PORT}/user/123`);
        console.log(`  GET  http://localhost:${PORT}/stats`);
    });
}

// Main
async function main() {
    try {
        await demo();
        await benchmark();

        // Optionnel: Démarrer le serveur Express
        // await startServer();
    } catch (error) {
        console.error('Error:', error);
    }
}

main();
```

---

## Modes de tracking avancés

### 1. Tracking normal vs Broadcasting

```
┌─────────────────────────────────────────────────────────────┐
│                  TRACKING MODES COMPARISON                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Mode         │ Memory    │ Latency   │ Use Case            │
│  ────────────────────────────────────────────────────────── │
│  Normal       │ O(keys)   │ Low       │ Few clients         │
│  (Default)    │ per       │           │ Tracking specific   │
│               │ client    │           │ keys                │
│                                                             │
│  Broadcasting │ O(1)      │ Slightly  │ Many clients        │
│  (BCAST)      │           │ higher    │ Same key patterns   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Exemples de configuration

```python
# ============================================================
# MODE 1: TRACKING NORMAL (par défaut)
# ============================================================

def setup_normal_tracking(redis_client):
    """
    Tracking normal : Redis track exactement quelles clés
    chaque client lit
    """
    redis_client.execute_command(
        'CLIENT', 'TRACKING', 'ON',
        'NOLOOP'  # Ne pas recevoir nos propres invalidations
    )

    # Avantages:
    # - Précis : seules les clés lues sont invalidées
    # - Pas de false positives

    # Inconvénients:
    # - Mémoire O(nombre de clés tracées par client)
    # - Plus de mémoire serveur si beaucoup de clients


# ============================================================
# MODE 2: BROADCASTING (BCAST)
# ============================================================

def setup_broadcast_tracking(redis_client, prefixes=['user:', 'product:']):
    """
    Tracking broadcast : Redis envoie toutes les invalidations
    pour certains préfixes
    """
    # Construction de la commande
    cmd = ['CLIENT', 'TRACKING', 'ON', 'BCAST', 'NOLOOP']

    # Ajouter les préfixes
    for prefix in prefixes:
        cmd.extend(['PREFIX', prefix])

    redis_client.execute_command(*cmd)

    # Avantages:
    # - Mémoire O(1) côté serveur
    # - Scalable pour beaucoup de clients

    # Inconvénients:
    # - False positives possibles
    # - Plus de trafic réseau (toutes les invalidations pour le préfixe)


# ============================================================
# MODE 3: OPTIN (manuel)
# ============================================================

def setup_optin_tracking(redis_client):
    """
    Mode OPTIN : Le client décide explicitement quelles
    clés tracker
    """
    redis_client.execute_command(
        'CLIENT', 'TRACKING', 'ON',
        'OPTIN',
        'NOLOOP'
    )

    # Avant chaque GET, activer le tracking pour cette clé
    redis_client.execute_command('CLIENT', 'CACHING', 'YES')
    value = redis_client.get('user:123')

    # La clé user:123 est maintenant trackée

    # Avantages:
    # - Contrôle total sur ce qui est tracké
    # - Optimisation fine de la mémoire

    # Inconvénients:
    # - Plus complexe à gérer
    # - Nécessite du code supplémentaire


# ============================================================
# MODE 4: OPTOUT (inverse de OPTIN)
# ============================================================

def setup_optout_tracking(redis_client):
    """
    Mode OPTOUT : Tout est tracké sauf si explicitement désactivé
    """
    redis_client.execute_command(
        'CLIENT', 'TRACKING', 'ON',
        'OPTOUT',
        'NOLOOP'
    )

    # Par défaut, tout est tracké
    value = redis_client.get('user:123')  # Tracké

    # Désactiver temporairement pour une commande
    redis_client.execute_command('CLIENT', 'CACHING', 'NO')
    value = redis_client.get('temp:data')  # Non tracké
```

---

## Cas d'usage réels

### 1. E-commerce : Cache produits avec CSC

```python
class ProductCacheWithCSC:
    """
    Cache de produits e-commerce avec Client-Side Caching
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.local_cache = {}

        # Activer le tracking BCAST pour les produits
        self.redis.execute_command(
            'CLIENT', 'TRACKING', 'ON',
            'BCAST',
            'PREFIX', 'product:',
            'NOLOOP'
        )

    def get_product(self, product_id: int) -> dict:
        """
        Récupère un produit avec CSC
        """
        cache_key = f"product:{product_id}"

        # Check local cache
        if cache_key in self.local_cache:
            return self.local_cache[cache_key]

        # Fetch from Redis
        product_json = self.redis.get(cache_key)

        if product_json:
            product = json.loads(product_json)
            self.local_cache[cache_key] = product
            return product

        # Fetch from DB (fallback)
        product = self._fetch_from_db(product_id)

        if product:
            # Cache in Redis
            self.redis.setex(
                cache_key,
                3600,  # 1 hour
                json.dumps(product)
            )
            self.local_cache[cache_key] = product

        return product

    def update_product_price(self, product_id: int, new_price: float):
        """
        Met à jour le prix d'un produit
        Redis enverra automatiquement une invalidation
        """
        cache_key = f"product:{product_id}"

        # Charger le produit
        product = self.get_product(product_id)

        if product:
            # Mettre à jour le prix
            product['price'] = new_price
            product['updated_at'] = time.time()

            # Sauvegarder dans Redis
            # → Déclenche l'invalidation du cache local
            self.redis.setex(
                cache_key,
                3600,
                json.dumps(product)
            )

            # Supprimer du cache local
            if cache_key in self.local_cache:
                del self.local_cache[cache_key]

    def _fetch_from_db(self, product_id: int) -> dict:
        # Simuler une requête DB
        return {
            'id': product_id,
            'name': f'Product {product_id}',
            'price': 99.99,
            'stock': 100
        }


# Utilisation dans une API
def handle_product_request(product_id: int):
    cache = ProductCacheWithCSC(redis_client)

    # Premier appel : Redis + DB
    product = cache.get_product(product_id)  # 10ms

    # Appels suivants : Local cache
    product = cache.get_product(product_id)  # 0.01ms
    product = cache.get_product(product_id)  # 0.01ms

    # Si le prix change ailleurs :
    # → Redis envoie invalidation
    # → Prochain appel ira sur Redis
```

### 2. Session store avec CSC

```javascript
class SessionStoreWithCSC {
    constructor(redis) {
        this.redis = redis;
        this.localCache = new Map();

        // Tracking pour les sessions
        this.redis.call(
            'CLIENT', 'TRACKING', 'ON',
            'BCAST',
            'PREFIX', 'session:',
            'NOLOOP'
        );
    }

    async getSession(sessionId) {
        const key = `session:${sessionId}`;

        // Check local cache
        if (this.localCache.has(key)) {
            return this.localCache.get(key);
        }

        // Fetch from Redis
        const sessionData = await this.redis.get(key);

        if (sessionData) {
            const session = JSON.parse(sessionData);
            this.localCache.set(key, session);
            return session;
        }

        return null;
    }

    async updateSession(sessionId, updates) {
        const key = `session:${sessionId}`;

        // Charger la session
        const session = await this.getSession(sessionId) || {};

        // Appliquer les mises à jour
        Object.assign(session, updates);

        // Sauvegarder dans Redis (TTL 30 min)
        await this.redis.setex(
            key,
            1800,
            JSON.stringify(session)
        );

        // Invalider le cache local
        this.localCache.delete(key);
    }

    async destroySession(sessionId) {
        const key = `session:${sessionId}`;

        await this.redis.del(key);
        this.localCache.delete(key);
    }
}
```

---

## Métriques et monitoring

### Monitoring du CSC

```python
class MonitoredClientSideCache:
    """
    CSC avec monitoring intégré
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.cache = {}

        # Métriques détaillées
        self.metrics = {
            'gets': 0,
            'local_hits': 0,
            'local_misses': 0,
            'redis_hits': 0,
            'redis_misses': 0,
            'invalidations': 0,
            'cache_size_samples': [],
            'latency_samples': {
                'local': [],
                'redis': []
            }
        }

        # Activer tracking
        self._enable_tracking()

    def get(self, key):
        import time

        self.metrics['gets'] += 1

        # Local cache check
        if key in self.cache:
            start = time.perf_counter()
            value = self.cache[key]
            latency = (time.perf_counter() - start) * 1000

            self.metrics['local_hits'] += 1
            self.metrics['latency_samples']['local'].append(latency)

            return value

        # Redis fetch
        self.metrics['local_misses'] += 1

        start = time.perf_counter()
        value = self.redis.get(key)
        latency = (time.perf_counter() - start) * 1000

        self.metrics['latency_samples']['redis'].append(latency)

        if value:
            self.metrics['redis_hits'] += 1
            self.cache[key] = value
        else:
            self.metrics['redis_misses'] += 1

        # Sample cache size
        self.metrics['cache_size_samples'].append(len(self.cache))

        return value

    def get_report(self):
        """
        Génère un rapport de performance
        """
        total = self.metrics['gets']
        if total == 0:
            return "No metrics yet"

        local_hit_rate = (self.metrics['local_hits'] / total) * 100

        # Calculer latences moyennes
        local_latencies = self.metrics['latency_samples']['local']
        redis_latencies = self.metrics['latency_samples']['redis']

        avg_local = sum(local_latencies) / len(local_latencies) if local_latencies else 0
        avg_redis = sum(redis_latencies) / len(redis_latencies) if redis_latencies else 0

        # Calculer taille moyenne du cache
        cache_sizes = self.metrics['cache_size_samples']
        avg_cache_size = sum(cache_sizes) / len(cache_sizes) if cache_sizes else 0

        return f"""
Client-Side Cache Performance Report:
─────────────────────────────────────────
Total requests:       {total}
Local cache hits:     {self.metrics['local_hits']} ({local_hit_rate:.2f}%)
Local cache misses:   {self.metrics['local_misses']}
Redis hits:           {self.metrics['redis_hits']}
Redis misses:         {self.metrics['redis_misses']}
Invalidations:        {self.metrics['invalidations']}

Latency:
  Local cache:        {avg_local:.3f}ms (avg)
  Redis:              {avg_redis:.3f}ms (avg)
  Speedup:            {avg_redis / avg_local if avg_local > 0 else 0:.1f}x

Cache size:           {len(self.cache)} (current)
                      {avg_cache_size:.1f} (average)
        """
```

### Dashboard Prometheus/Grafana

```python
from prometheus_client import Counter, Histogram, Gauge

# Métriques Prometheus
csc_requests = Counter(
    'redis_csc_requests_total',
    'Total requests via CSC',
    ['cache_type']  # 'local' ou 'redis'
)

csc_latency = Histogram(
    'redis_csc_latency_seconds',
    'Latency of CSC requests',
    ['cache_type']
)

csc_cache_size = Gauge(
    'redis_csc_cache_size',
    'Number of items in local cache'
)

csc_invalidations = Counter(
    'redis_csc_invalidations_total',
    'Total cache invalidations received'
)


class PrometheusMonitoredCSC:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.cache = {}
        self._enable_tracking()

    def get(self, key):
        import time

        # Local cache
        if key in self.cache:
            start = time.time()
            value = self.cache[key]
            latency = time.time() - start

            csc_requests.labels(cache_type='local').inc()
            csc_latency.labels(cache_type='local').observe(latency)

            return value

        # Redis
        start = time.time()
        value = self.redis.get(key)
        latency = time.time() - start

        csc_requests.labels(cache_type='redis').inc()
        csc_latency.labels(cache_type='redis').observe(latency)

        if value:
            self.cache[key] = value
            csc_cache_size.set(len(self.cache))

        return value

    def _handle_invalidation(self, keys):
        for key in keys:
            if key in self.cache:
                del self.cache[key]
                csc_invalidations.inc()
                csc_cache_size.set(len(self.cache))
```

---

## Limitations et considérations

### Limitations connues

```
┌──────────────────────────────────────────────────────────────┐
│              CLIENT-SIDE CACHING LIMITATIONS                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Redis 6.0+ requis                                        │
│     - RESP3 protocol obligatoire                             │
│     - Ne fonctionne pas avec Redis < 6.0                     │
│                                                              │
│  2. Mémoire client                                           │
│     - Cache local consomme de la RAM                         │
│     - Besoin de limiter la taille du cache                   │
│     - Risque de memory leak si mal géré                      │
│                                                              │
│  3. Cohérence                                                │
│     - Eventual consistency seulement                         │
│     - Délai possible entre update et invalidation            │
│     - Pas adapté pour strong consistency                     │
│                                                              │
│  4. Complexité                                               │
│     - Setup plus complexe                                    │
│     - Debugging plus difficile                               │
│     - Nécessite monitoring                                   │
│                                                              │
│  5. Overhead serveur                                         │
│     - Redis doit tracker les clés par client                 │
│     - Mémoire serveur augmentée (mode normal)                │
│     - CPU pour envoyer invalidations                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Quand utiliser CSC

```python
# ✅ BON CAS D'USAGE

# 1. Lecture très intensive (95%+ lectures)
def analytics_dashboard():
    # Dashboard lit les mêmes stats des milliers de fois
    stats = cache.get('dashboard:stats')
    # CSC parfait ici!

# 2. Données rarement mises à jour
def get_config():
    # Config change rarement
    config = cache.get('app:config')
    # CSC excellent!

# 3. Many clients, same data
def get_product_catalog():
    # Tous les clients lisent le même catalogue
    catalog = cache.get('products:catalog')
    # CSC avec BCAST mode!


# ❌ MAUVAIS CAS D'USAGE

# 1. Écritures fréquentes
def update_counter():
    # Compteur mis à jour en permanence
    counter = cache.get('counter')
    # CSC inutile, trop d'invalidations

# 2. Strong consistency requise
def bank_balance():
    # Solde bancaire doit être exact
    balance = cache.get(f'balance:{user_id}')
    # Ne PAS utiliser CSC!

# 3. Données volatiles
def real_time_stock_price():
    # Prix change chaque seconde
    price = cache.get('stock:AAPL')
    # CSC contre-productif

# 4. Données uniques par client
def user_specific_data(user_id):
    # Chaque user a des données différentes
    data = cache.get(f'user:{user_id}:temp')
    # CSC pas optimal si beaucoup d'users différents
```

---

## Conclusion

Le **Client-Side Caching** est une fonctionnalité révolutionnaire de Redis 6.0+ qui peut multiplier les performances par 100-1000 pour les workloads read-heavy. Les points clés :

**Avantages** :
- ⚡ Latence proche de zéro (0.001ms vs 5ms)
- 🚀 Throughput multiplié par 100-1000
- 📉 Réduction drastique de la charge réseau
- 🔄 Invalidations automatiques (cohérence garantie)

**Quand utiliser** :
- Read-heavy workloads (95%+ lectures)
- Données rarement mises à jour
- Many clients reading same data
- Configuration, catalogues, metadata

**Quand NE PAS utiliser** :
- Strong consistency requise
- Écritures très fréquentes
- Données volatiles
- Système critique sans Redis 6.0+

**Best Practices** :
1. Limiter la taille du cache local (max 10,000 items)
2. Monitorer le hit rate (objectif: >80%)
3. Utiliser BCAST mode pour many clients
4. Gérer la mémoire client (LRU eviction)
5. Avoir un fallback si tracking échoue

Le Client-Side Caching est LA fonctionnalité killer pour les applications modernes avec Redis. Si vous avez Redis 6.0+ et des lectures intensives, vous devez l'utiliser!

---


⏭️ [Distributed Locking : Le pattern Redlock](/06-patterns-developpement-avances/05-distributed-locking-redlock.md)
