🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4 Async/Await et programmation réactive

## Introduction

La programmation asynchrone est **cruciale** pour maximiser les performances des applications Redis. Contrairement aux opérations CPU-bound, les opérations Redis sont **I/O-bound** : le temps d'attente réseau représente souvent 95% du temps total d'une opération.

Pendant qu'une requête attend la réponse de Redis, une application **synchrone** bloque un thread/processus entier, alors qu'une application **asynchrone** peut traiter d'autres requêtes.

```
┌────────────────────────────────────────────────────────────┐
│           Synchrone vs Asynchrone                          │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  SYNCHRONE (bloquant)                                      │
│  Thread 1: [▓▓▓GET█████████████████] ← Bloqué              │
│  Thread 2: [▓▓▓GET█████████████████] ← Bloqué              │
│  Thread 3: [▓▓▓GET█████████████████] ← Bloqué              │
│            └─┬─┘└────────┬─────────┘                       │
│             CPU      Attente réseau                        │
│                                                            │
│  ASYNCHRONE (non-bloquant)                                 │
│  Event Loop: [▓GET▓GET▓GET████] ← Multiples ops            │
│               └─┬─┘                                        │
│                CPU (négligeable)                           │
│                                                            │
│  Résultat : 100x plus de requêtes/sec avec async !         │
└────────────────────────────────────────────────────────────┘
```

**Gain typique** : Une application async peut traiter **10-100x plus de requêtes** qu'une version synchrone avec les mêmes ressources.

---

## Concepts fondamentaux

### 1. Synchrone vs Asynchrone

**Synchrone (bloquant)**
```python
# Bloque pendant ~1ms (network RTT)
value = redis.get('key')
# Thread inactif pendant tout ce temps
print(value)
```

**Asynchrone (non-bloquant)**
```python
# Libère immédiatement le thread
value = await redis.get('key')  # Pas de blocage !
# Autres tâches peuvent s'exécuter pendant l'attente
print(value)
```

### 2. Event Loop (boucle d'événements)

L'event loop est le cœur de la programmation asynchrone :

```
┌─────────────────────────────────────────────────┐
│            Event Loop                           │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Envoyer requête Redis (GET key1)            │
│  2. Envoyer requête Redis (GET key2)            │
│  3. Envoyer requête Redis (GET key3)            │
│  4. Traiter autre tâche pendant attente         │
│  5. Recevoir réponse key1 → callback            │
│  6. Recevoir réponse key2 → callback            │
│  7. Recevoir réponse key3 → callback            │
│                                                 │
│  Tout sur un seul thread !                      │
└─────────────────────────────────────────────────┘
```

### 3. Promises, Futures, Coroutines

| Langage | Concept | Syntaxe |
|---------|---------|---------|
| **Python** | Coroutine | `async def`, `await` |
| **Node.js** | Promise | `async/await`, `.then()` |
| **Go** | Goroutine | `go func()`, channels |
| **Java** | CompletableFuture | `.thenApply()`, `.join()` |

---

## Python : asyncio et redis-py

### Configuration du client async

```python
import asyncio
import redis.asyncio as redis
from typing import List, Dict, Any

# Configuration du client async
async def create_redis_client():
    """Crée un client Redis asynchrone"""
    client = redis.Redis(
        host='localhost',
        port=6379,
        decode_responses=True,
        socket_connect_timeout=5,
        socket_timeout=5,
        max_connections=50,
    )

    # Test de connexion
    await client.ping()
    print("✅ Redis async client connected")

    return client

# Utilisation
async def main():
    client = await create_redis_client()

    # Opérations async
    await client.set('key', 'value')
    value = await client.get('key')
    print(f"Value: {value}")

    await client.close()

# Lancement
asyncio.run(main())
```

### Pattern : Opérations concurrentes

```python
import asyncio
import redis.asyncio as redis
import time

async def get_multiple_keys_sequential(client, keys: List[str]) -> List[str]:
    """
    ❌ LENT : Opérations séquentielles
    Temps total = n * latence_redis
    """
    results = []
    for key in keys:
        value = await client.get(key)
        results.append(value)
    return results

async def get_multiple_keys_concurrent(client, keys: List[str]) -> List[str]:
    """
    ✅ RAPIDE : Opérations concurrentes
    Temps total ≈ latence_redis (1x, pas n×)
    """
    # Créer toutes les coroutines
    tasks = [client.get(key) for key in keys]

    # Exécuter en parallèle
    results = await asyncio.gather(*tasks)

    return results

async def benchmark():
    client = await redis.Redis(decode_responses=True)

    # Préparer des clés
    keys = [f'user:{i}' for i in range(100)]
    for key in keys:
        await client.set(key, f'value_{key}')

    # Test séquentiel
    start = time.time()
    await get_multiple_keys_sequential(client, keys)
    sequential_time = time.time() - start
    print(f"⏱️  Séquentiel : {sequential_time:.2f}s")

    # Test concurrent
    start = time.time()
    await get_multiple_keys_concurrent(client, keys)
    concurrent_time = time.time() - start
    print(f"⚡ Concurrent : {concurrent_time:.2f}s")

    speedup = sequential_time / concurrent_time
    print(f"🚀 Speedup : {speedup:.1f}x plus rapide !")

    await client.close()

# Output typique :
# ⏱️  Séquentiel : 0.15s  (100 × 1.5ms)
# ⚡ Concurrent : 0.002s  (1 × 1.5ms + overhead)
# 🚀 Speedup : 75.0x plus rapide !

asyncio.run(benchmark())
```

### Pattern : Pipeline asynchrone

```python
import asyncio
import redis.asyncio as redis

async def pipeline_example(client):
    """
    Pipeline asynchrone : envoie N commandes en 1 RTT
    """
    # Créer le pipeline
    async with client.pipeline(transaction=True) as pipe:
        # Ajouter des commandes (ne les exécute pas encore)
        pipe.set('key1', 'value1')
        pipe.set('key2', 'value2')
        pipe.incr('counter')
        pipe.get('key1')
        pipe.get('key2')
        pipe.get('counter')

        # Exécuter toutes les commandes en une fois
        results = await pipe.execute()

    # results = [True, True, 1, 'value1', 'value2', '1']
    print(f"Results: {results}")

    return results

async def main():
    client = await redis.Redis(decode_responses=True)
    await pipeline_example(client)
    await client.close()

asyncio.run(main())
```

### Pattern : Limite de concurrence (Semaphore)

```python
import asyncio
import redis.asyncio as redis
from typing import List

class RateLimitedRedisClient:
    """Client Redis avec limite de concurrence"""

    def __init__(self, client, max_concurrent: int = 10):
        self.client = client
        self.semaphore = asyncio.Semaphore(max_concurrent)

    async def get(self, key: str):
        """GET avec limite de concurrence"""
        async with self.semaphore:
            return await self.client.get(key)

    async def set(self, key: str, value: str):
        """SET avec limite de concurrence"""
        async with self.semaphore:
            return await self.client.set(key, value)

    async def get_many(self, keys: List[str]) -> List[str]:
        """
        Récupère plusieurs clés avec limite de concurrence

        Sans limite : 1000 requêtes simultanées → surcharge
        Avec limite : 10 requêtes max simultanées → stable
        """
        tasks = [self.get(key) for key in keys]
        results = await asyncio.gather(*tasks)
        return results

async def main():
    redis_client = await redis.Redis(decode_responses=True)

    # Client avec max 10 requêtes concurrentes
    limited_client = RateLimitedRedisClient(redis_client, max_concurrent=10)

    # Récupérer 1000 clés (mais max 10 à la fois)
    keys = [f'key:{i}' for i in range(1000)]
    results = await limited_client.get_many(keys)

    print(f"✅ Retrieved {len(results)} keys with rate limiting")

    await redis_client.close()

asyncio.run(main())
```

### Pattern : Timeout et cancellation

```python
import asyncio
import redis.asyncio as redis

async def get_with_timeout(client, key: str, timeout: float = 1.0):
    """GET avec timeout personnalisé"""
    try:
        # Timeout après N secondes
        result = await asyncio.wait_for(
            client.get(key),
            timeout=timeout
        )
        return result
    except asyncio.TimeoutError:
        print(f"⏱️ Timeout après {timeout}s pour key '{key}'")
        return None

async def multiple_operations_with_cancellation():
    """Opérations concurrentes avec possibilité d'annulation"""
    client = await redis.Redis(decode_responses=True)

    async def long_operation():
        """Opération potentiellement longue"""
        for i in range(10):
            await client.set(f'key:{i}', f'value:{i}')
            await asyncio.sleep(0.1)  # Simule traitement
        return "Done"

    # Créer la tâche
    task = asyncio.create_task(long_operation())

    try:
        # Attendre max 0.5 secondes
        result = await asyncio.wait_for(task, timeout=0.5)
        print(f"Résultat: {result}")
    except asyncio.TimeoutError:
        # Annuler la tâche si timeout
        task.cancel()
        print("⚠️ Opération annulée (timeout)")
        try:
            await task
        except asyncio.CancelledError:
            print("✅ Tâche annulée proprement")

    await client.close()

# Test timeout
async def main():
    client = await redis.Redis(decode_responses=True)

    # GET normal
    value = await get_with_timeout(client, 'existing_key', timeout=1.0)

    # GET qui timeout (clé dans Redis très lent)
    value = await get_with_timeout(client, 'slow_key', timeout=0.001)

    await client.close()

asyncio.run(main())
```

### Application complète : API asynchrone avec cache

```python
import asyncio
import redis.asyncio as redis
from typing import Optional, Dict
import json
import time

class AsyncCacheManager:
    """Gestionnaire de cache asynchrone avec Redis"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.stats = {
            'hits': 0,
            'misses': 0,
            'errors': 0,
        }

    async def get_cached(
        self,
        key: str,
        fetch_fn,
        ttl: int = 3600
    ) -> Optional[Dict]:
        """
        Pattern Cache-Aside asynchrone

        1. Essayer de lire dans Redis
        2. Si cache miss → exécuter fetch_fn
        3. Stocker dans Redis pour la prochaine fois
        """
        try:
            # Tentative lecture cache
            cached = await self.redis.get(key)

            if cached:
                self.stats['hits'] += 1
                print(f"🎯 Cache HIT: {key}")
                return json.loads(cached)

            print(f"❌ Cache MISS: {key}")
            self.stats['misses'] += 1

        except Exception as e:
            self.stats['errors'] += 1
            print(f"⚠️ Cache error: {e}")

        # Cache miss ou erreur : récupérer depuis la source
        data = await fetch_fn()

        # Stocker dans le cache (fire-and-forget)
        try:
            await self.redis.setex(
                key,
                ttl,
                json.dumps(data)
            )
            print(f"💾 Cached: {key}")
        except Exception as e:
            print(f"⚠️ Failed to cache: {e}")

        return data

    async def get_many_cached(
        self,
        keys: list,
        fetch_fn,
        ttl: int = 3600
    ) -> Dict[str, Dict]:
        """
        Récupère plusieurs éléments avec cache
        Optimisé pour minimiser les requêtes
        """
        # 1. Récupérer toutes les clés en parallèle
        cache_tasks = [self.redis.get(key) for key in keys]
        cached_values = await asyncio.gather(*cache_tasks, return_exceptions=True)

        # 2. Identifier les cache misses
        results = {}
        missing_keys = []

        for key, cached in zip(keys, cached_values):
            if isinstance(cached, Exception):
                missing_keys.append(key)
            elif cached:
                results[key] = json.loads(cached)
                self.stats['hits'] += 1
            else:
                missing_keys.append(key)
                self.stats['misses'] += 1

        # 3. Récupérer les données manquantes
        if missing_keys:
            fresh_data = await fetch_fn(missing_keys)

            # 4. Stocker dans le cache (en parallèle)
            cache_tasks = [
                self.redis.setex(key, ttl, json.dumps(data))
                for key, data in fresh_data.items()
            ]
            await asyncio.gather(*cache_tasks, return_exceptions=True)

            results.update(fresh_data)

        return results

    def get_stats(self) -> Dict:
        """Statistiques du cache"""
        total = self.stats['hits'] + self.stats['misses']
        hit_rate = (self.stats['hits'] / total * 100) if total > 0 else 0

        return {
            **self.stats,
            'hit_rate': f"{hit_rate:.1f}%",
        }

# Exemple d'utilisation avec une API fictive
async def fetch_user_from_db(user_id: int) -> Dict:
    """Simule une requête base de données lente"""
    print(f"🔍 Fetching user {user_id} from database...")
    await asyncio.sleep(0.1)  # Simule latence DB
    return {
        'id': user_id,
        'name': f'User {user_id}',
        'email': f'user{user_id}@example.com'
    }

async def fetch_users_from_db(user_ids: list) -> Dict[str, Dict]:
    """Récupère plusieurs utilisateurs depuis la DB"""
    print(f"🔍 Fetching {len(user_ids)} users from database...")
    await asyncio.sleep(0.1)

    return {
        f'user:{uid}': {
            'id': uid,
            'name': f'User {uid}',
            'email': f'user{uid}@example.com'
        }
        for uid in [int(k.split(':')[1]) for k in user_ids]
    }

async def main():
    # Initialiser le cache
    redis_client = await redis.Redis(decode_responses=True)
    cache = AsyncCacheManager(redis_client)

    # Test 1 : Requête unique
    print("\n=== Test 1 : Single Request ===")

    # Premier appel : cache miss
    user = await cache.get_cached(
        'user:123',
        lambda: fetch_user_from_db(123),
        ttl=60
    )
    print(f"User: {user}")

    # Deuxième appel : cache hit
    user = await cache.get_cached(
        'user:123',
        lambda: fetch_user_from_db(123),
        ttl=60
    )
    print(f"User: {user}")

    # Test 2 : Requêtes multiples concurrentes
    print("\n=== Test 2 : Concurrent Requests ===")

    # Récupérer 100 utilisateurs en parallèle
    user_ids = [f'user:{i}' for i in range(1, 101)]

    start = time.time()
    users = await cache.get_many_cached(
        user_ids,
        fetch_users_from_db,
        ttl=60
    )
    duration = time.time() - start

    print(f"✅ Retrieved {len(users)} users in {duration:.3f}s")

    # Deuxième fois : tout depuis le cache
    start = time.time()
    users = await cache.get_many_cached(
        user_ids,
        fetch_users_from_db,
        ttl=60
    )
    duration = time.time() - start

    print(f"✅ Retrieved {len(users)} users in {duration:.3f}s (cached)")

    # Statistiques
    print("\n=== Cache Stats ===")
    stats = cache.get_stats()
    print(f"Hits: {stats['hits']}")
    print(f"Misses: {stats['misses']}")
    print(f"Hit Rate: {stats['hit_rate']}")

    await redis_client.close()

asyncio.run(main())
```

---

## Node.js : Promises et async/await

### Configuration du client async

```javascript
import Redis from 'ioredis';

// ioredis est async par défaut
const redis = new Redis({
    host: 'localhost',
    port: 6379,
    maxRetriesPerRequest: 3,
});

// Toutes les commandes retournent des Promises
async function basicOperations() {
    // async/await (recommandé)
    await redis.set('key', 'value');
    const value = await redis.get('key');
    console.log('Value:', value);

    // .then() (style ancien)
    redis.get('key')
        .then(value => console.log('Value:', value))
        .catch(err => console.error('Error:', err));
}

basicOperations();
```

### Pattern : Opérations concurrentes avec Promise.all

```javascript
import Redis from 'ioredis';

const redis = new Redis();

/**
 * ❌ LENT : Opérations séquentielles
 */
async function getMultipleSequential(keys) {
    const results = [];
    for (const key of keys) {
        const value = await redis.get(key);
        results.push(value);
    }
    return results;
}

/**
 * ✅ RAPIDE : Opérations concurrentes
 */
async function getMultipleConcurrent(keys) {
    // Créer toutes les Promises
    const promises = keys.map(key => redis.get(key));

    // Exécuter en parallèle
    const results = await Promise.all(promises);

    return results;
}

async function benchmark() {
    // Préparer données
    const keys = Array.from({length: 100}, (_, i) => `user:${i}`);
    await Promise.all(
        keys.map(key => redis.set(key, `value_${key}`))
    );

    // Test séquentiel
    console.time('Sequential');
    await getMultipleSequential(keys);
    console.timeEnd('Sequential');
    // Sequential: ~150ms

    // Test concurrent
    console.time('Concurrent');
    await getMultipleConcurrent(keys);
    console.timeEnd('Concurrent');
    // Concurrent: ~2ms

    await redis.quit();
}

benchmark();
```

### Pattern : Promise.allSettled pour gérer les erreurs

```javascript
import Redis from 'ioredis';

const redis = new Redis();

/**
 * Promise.allSettled : Continue même si certaines Promises échouent
 */
async function getMultipleWithErrorHandling(keys) {
    const promises = keys.map(key => redis.get(key));

    // allSettled ne rejette jamais
    const results = await Promise.allSettled(promises);

    // Traiter les résultats
    const successful = [];
    const failed = [];

    results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
            successful.push({
                key: keys[index],
                value: result.value
            });
        } else {
            failed.push({
                key: keys[index],
                error: result.reason
            });
        }
    });

    console.log(`✅ Success: ${successful.length}`);
    console.log(`❌ Failed: ${failed.length}`);

    return { successful, failed };
}

// Utilisation
const keys = ['user:1', 'user:2', 'invalid:key:format'];
const results = await getMultipleWithErrorHandling(keys);
```

### Pattern : Rate limiting avec p-limit

```javascript
import Redis from 'ioredis';
import pLimit from 'p-limit';

const redis = new Redis();

/**
 * Limite le nombre de Promises concurrentes
 */
async function getMultipleWithRateLimit(keys, concurrency = 10) {
    // Créer un limiteur
    const limit = pLimit(concurrency);

    // Wrapper chaque opération dans le limiteur
    const promises = keys.map(key =>
        limit(() => redis.get(key))
    );

    // Exécuter (max `concurrency` à la fois)
    const results = await Promise.all(promises);

    return results;
}

// Traiter 1000 clés, mais max 10 à la fois
const keys = Array.from({length: 1000}, (_, i) => `key:${i}`);
const results = await getMultipleWithRateLimit(keys, 10);
console.log(`✅ Processed ${results.length} keys with rate limiting`);
```

### Pattern : Retry avec async-retry

```javascript
import Redis from 'ioredis';
import retry from 'async-retry';

const redis = new Redis();

/**
 * Retry automatique avec exponential backoff
 */
async function getWithRetry(key) {
    return await retry(
        async (bail) => {
            try {
                const value = await redis.get(key);
                return value;
            } catch (err) {
                // Si erreur non-retriable, abandonner
                if (err.message.includes('WRONGTYPE')) {
                    bail(err);
                    return;
                }

                // Sinon, retry
                console.log(`⚠️ Retry for key '${key}': ${err.message}`);
                throw err;
            }
        },
        {
            retries: 5,           // Max 5 tentatives
            factor: 2,            // Exponential backoff
            minTimeout: 100,      // Min 100ms
            maxTimeout: 5000,     // Max 5s
            randomize: true,      // Jitter
        }
    );
}

// Utilisation
try {
    const value = await getWithRetry('my:key');
    console.log('Value:', value);
} catch (err) {
    console.error('Failed after all retries:', err);
}
```

### Application complète : Worker pool asynchrone

```javascript
import Redis from 'ioredis';
import { EventEmitter } from 'events';

class AsyncWorkerPool extends EventEmitter {
    constructor(redisConfig, options = {}) {
        super();

        this.redis = new Redis(redisConfig);
        this.concurrency = options.concurrency || 10;
        this.queueKey = options.queueKey || 'jobs:queue';
        this.processingKey = options.processingKey || 'jobs:processing';
        this.running = false;
        this.activeJobs = 0;
    }

    /**
     * Démarre le worker pool
     */
    async start() {
        this.running = true;
        console.log(`🚀 Worker pool started (concurrency: ${this.concurrency})`);

        // Créer N workers concurrents
        const workers = Array.from(
            {length: this.concurrency},
            (_, i) => this.worker(i)
        );

        // Attendre que tous les workers se terminent
        await Promise.all(workers);
    }

    /**
     * Worker individuel
     */
    async worker(workerId) {
        while (this.running) {
            try {
                // BRPOPLPUSH : atomique et bloquant
                const job = await this.redis.brpoplpush(
                    this.queueKey,
                    this.processingKey,
                    5 // timeout 5s
                );

                if (!job) {
                    // Timeout, continuer
                    continue;
                }

                this.activeJobs++;
                this.emit('job-started', { workerId, job });

                // Traiter le job
                const result = await this.processJob(JSON.parse(job));

                // Retirer de processing
                await this.redis.lrem(this.processingKey, 1, job);

                this.activeJobs--;
                this.emit('job-completed', { workerId, job, result });

            } catch (err) {
                this.emit('job-failed', { workerId, error: err });
                console.error(`❌ Worker ${workerId} error:`, err);

                // Attendre avant de continuer
                await new Promise(resolve => setTimeout(resolve, 1000));
            }
        }

        console.log(`🛑 Worker ${workerId} stopped`);
    }

    /**
     * Traite un job (à override)
     */
    async processJob(job) {
        console.log(`⚙️ Processing job:`, job);

        // Simule traitement
        await new Promise(resolve => setTimeout(resolve, 100));

        return { success: true, processed_at: Date.now() };
    }

    /**
     * Ajoute un job à la queue
     */
    async addJob(job) {
        await this.redis.lpush(this.queueKey, JSON.stringify(job));
        this.emit('job-queued', job);
    }

    /**
     * Arrête le worker pool
     */
    async stop() {
        console.log('🛑 Stopping worker pool...');
        this.running = false;

        // Attendre que les jobs actifs se terminent
        while (this.activeJobs > 0) {
            await new Promise(resolve => setTimeout(resolve, 100));
        }

        await this.redis.quit();
        console.log('✅ Worker pool stopped');
    }

    /**
     * Statistiques
     */
    async getStats() {
        const queueLength = await this.redis.llen(this.queueKey);
        const processingLength = await this.redis.llen(this.processingKey);

        return {
            queued: queueLength,
            processing: processingLength,
            active_workers: this.activeJobs,
            concurrency: this.concurrency,
        };
    }
}

// Utilisation
const pool = new AsyncWorkerPool(
    { host: 'localhost', port: 6379 },
    { concurrency: 5, queueKey: 'my:jobs' }
);

// Event listeners
pool.on('job-started', ({ workerId, job }) => {
    console.log(`▶️  Worker ${workerId} started job:`, job);
});

pool.on('job-completed', ({ workerId, result }) => {
    console.log(`✅ Worker ${workerId} completed`);
});

pool.on('job-failed', ({ workerId, error }) => {
    console.error(`❌ Worker ${workerId} failed:`, error);
});

// Ajouter des jobs
for (let i = 0; i < 20; i++) {
    await pool.addJob({
        id: i,
        type: 'process-data',
        data: { value: i }
    });
}

// Démarrer le pool
await pool.start();

// Arrêter proprement après quelques secondes
setTimeout(async () => {
    await pool.stop();
}, 10000);
```

---

## Go : Goroutines et Channels

### Pattern : Concurrence avec goroutines

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

/**
 * ❌ LENT : Opérations séquentielles
 */
func getMultipleSequential(rdb *redis.Client, keys []string) []string {
    results := make([]string, len(keys))

    for i, key := range keys {
        val, _ := rdb.Get(ctx, key).Result()
        results[i] = val
    }

    return results
}

/**
 * ✅ RAPIDE : Opérations concurrentes avec goroutines
 */
func getMultipleConcurrent(rdb *redis.Client, keys []string) []string {
    results := make([]string, len(keys))
    var wg sync.WaitGroup

    for i, key := range keys {
        wg.Add(1)
        go func(index int, k string) {
            defer wg.Done()
            val, _ := rdb.Get(ctx, k).Result()
            results[index] = val
        }(i, key)
    }

    wg.Wait()
    return results
}

func benchmark() {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Préparer données
    keys := make([]string, 100)
    for i := 0; i < 100; i++ {
        key := fmt.Sprintf("user:%d", i)
        keys[i] = key
        rdb.Set(ctx, key, fmt.Sprintf("value_%d", i), 0)
    }

    // Test séquentiel
    start := time.Now()
    getMultipleSequential(rdb, keys)
    fmt.Printf("Sequential: %v\n", time.Since(start))
    // Sequential: ~150ms

    // Test concurrent
    start = time.Now()
    getMultipleConcurrent(rdb, keys)
    fmt.Printf("Concurrent: %v\n", time.Since(start))
    // Concurrent: ~2ms
}

func main() {
    benchmark()
}
```

### Pattern : Worker pool avec channels

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
)

type Job struct {
    ID   int
    Key  string
    Data string
}

type Result struct {
    Job   Job
    Value string
    Error error
}

type WorkerPool struct {
    redis       *redis.Client
    jobs        chan Job
    results     chan Result
    numWorkers  int
    wg          sync.WaitGroup
}

func NewWorkerPool(rdb *redis.Client, numWorkers int) *WorkerPool {
    return &WorkerPool{
        redis:      rdb,
        jobs:       make(chan Job, 100),
        results:    make(chan Result, 100),
        numWorkers: numWorkers,
    }
}

func (wp *WorkerPool) Start(ctx context.Context) {
    // Démarrer N workers
    for i := 0; i < wp.numWorkers; i++ {
        wp.wg.Add(1)
        go wp.worker(ctx, i)
    }
}

func (wp *WorkerPool) worker(ctx context.Context, id int) {
    defer wp.wg.Done()

    log.Printf("🚀 Worker %d started", id)

    for job := range wp.jobs {
        // Traiter le job
        result := wp.processJob(ctx, job)

        // Envoyer le résultat
        wp.results <- result
    }

    log.Printf("🛑 Worker %d stopped", id)
}

func (wp *WorkerPool) processJob(ctx context.Context, job Job) Result {
    // Stocker dans Redis
    err := wp.redis.Set(ctx, job.Key, job.Data, time.Hour).Err()

    if err != nil {
        return Result{Job: job, Error: err}
    }

    // Récupérer pour vérification
    val, err := wp.redis.Get(ctx, job.Key).Result()

    return Result{
        Job:   job,
        Value: val,
        Error: err,
    }
}

func (wp *WorkerPool) Submit(job Job) {
    wp.jobs <- job
}

func (wp *WorkerPool) Close() {
    close(wp.jobs)
    wp.wg.Wait()
    close(wp.results)
}

func (wp *WorkerPool) Results() <-chan Result {
    return wp.results
}

func main() {
    ctx := context.Background()

    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Créer le pool avec 5 workers
    pool := NewWorkerPool(rdb, 5)
    pool.Start(ctx)

    // Soumettre 100 jobs
    go func() {
        for i := 0; i < 100; i++ {
            pool.Submit(Job{
                ID:   i,
                Key:  fmt.Sprintf("job:%d", i),
                Data: fmt.Sprintf("data_%d", i),
            })
        }
        pool.Close()
    }()

    // Collecter les résultats
    successCount := 0
    errorCount := 0

    for result := range pool.Results() {
        if result.Error != nil {
            errorCount++
            log.Printf("❌ Job %d failed: %v", result.Job.ID, result.Error)
        } else {
            successCount++
        }
    }

    fmt.Printf("\n=== Results ===\n")
    fmt.Printf("✅ Success: %d\n", successCount)
    fmt.Printf("❌ Errors: %d\n", errorCount)
}
```

### Pattern : Rate limiting avec semaphore

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
    "golang.org/x/sync/semaphore"
)

func getMultipleWithRateLimit(
    ctx context.Context,
    rdb *redis.Client,
    keys []string,
    maxConcurrent int64,
) ([]string, error) {
    // Créer un semaphore
    sem := semaphore.NewWeighted(maxConcurrent)

    results := make([]string, len(keys))
    var wg sync.WaitGroup
    var mu sync.Mutex
    var firstErr error

    for i, key := range keys {
        // Acquérir le semaphore (bloque si limite atteinte)
        if err := sem.Acquire(ctx, 1); err != nil {
            return nil, err
        }

        wg.Add(1)
        go func(index int, k string) {
            defer wg.Done()
            defer sem.Release(1) // Libérer le semaphore

            val, err := rdb.Get(ctx, k).Result()
            if err != nil && firstErr == nil {
                mu.Lock()
                firstErr = err
                mu.Unlock()
            }

            results[index] = val
        }(i, key)
    }

    wg.Wait()
    return results, firstErr
}

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Préparer 1000 clés
    keys := make([]string, 1000)
    for i := 0; i < 1000; i++ {
        keys[i] = fmt.Sprintf("key:%d", i)
    }

    // Traiter avec max 10 goroutines concurrentes
    start := time.Now()
    results, err := getMultipleWithRateLimit(ctx, rdb, keys, 10)
    duration := time.Since(start)

    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("✅ Processed %d keys in %v (max 10 concurrent)\n",
               len(results), duration)
}
```

---

## Programmation réactive avec Redis Streams

### Consumer réactif (Python)

```python
import asyncio
import redis.asyncio as redis
from typing import AsyncIterator

class ReactiveStreamConsumer:
    """Consumer réactif de Redis Streams"""

    def __init__(self, redis_client, stream_key: str, consumer_group: str):
        self.redis = redis_client
        self.stream_key = stream_key
        self.consumer_group = consumer_group
        self.consumer_name = f"consumer-{id(self)}"

    async def create_consumer_group(self):
        """Crée le consumer group s'il n'existe pas"""
        try:
            await self.redis.xgroup_create(
                self.stream_key,
                self.consumer_group,
                id='0',
                mkstream=True
            )
            print(f"✅ Consumer group '{self.consumer_group}' created")
        except redis.ResponseError as e:
            if 'BUSYGROUP' not in str(e):
                raise

    async def consume_stream(self) -> AsyncIterator:
        """
        Consomme le stream de manière réactive
        Yield les messages au fur et à mesure
        """
        await self.create_consumer_group()

        last_id = '>'

        while True:
            try:
                # XREADGROUP avec BLOCK
                messages = await self.redis.xreadgroup(
                    self.consumer_group,
                    self.consumer_name,
                    {self.stream_key: last_id},
                    count=10,
                    block=5000  # Bloque max 5s
                )

                if not messages:
                    continue

                for stream, message_list in messages:
                    for message_id, data in message_list:
                        yield {
                            'id': message_id,
                            'data': data,
                            'stream': stream
                        }

                        # ACK le message
                        await self.redis.xack(
                            self.stream_key,
                            self.consumer_group,
                            message_id
                        )

            except asyncio.CancelledError:
                print("Stream consumer cancelled")
                break
            except Exception as e:
                print(f"Error consuming stream: {e}")
                await asyncio.sleep(1)

# Utilisation
async def main():
    redis_client = await redis.Redis(decode_responses=True)

    # Producer : ajoute des messages
    async def producer():
        for i in range(100):
            await redis_client.xadd(
                'events',
                {'type': 'user_action', 'user_id': i, 'action': 'click'}
            )
            await asyncio.sleep(0.1)

    # Consumer : traite les messages de manière réactive
    consumer = ReactiveStreamConsumer(redis_client, 'events', 'my-group')

    async def consume():
        async for message in consumer.consume_stream():
            print(f"📨 Message reçu: {message['id']} - {message['data']}")
            # Traiter le message ici

    # Lancer producer et consumer en parallèle
    await asyncio.gather(
        producer(),
        consume(),
    )

    await redis_client.close()

asyncio.run(main())
```

---

## Comparaison des performances

### Benchmark synchrone vs asynchrone

```python
import asyncio
import redis
import redis.asyncio as aioredis
import time

def benchmark_sync(iterations=1000):
    """Benchmark synchrone"""
    client = redis.Redis(decode_responses=True)

    start = time.time()
    for i in range(iterations):
        client.set(f'key:{i}', f'value:{i}')
        client.get(f'key:{i}')
    duration = time.time() - start

    ops_per_sec = (iterations * 2) / duration
    print(f"Sync: {duration:.2f}s ({ops_per_sec:.0f} ops/sec)")

    client.close()
    return duration

async def benchmark_async(iterations=1000):
    """Benchmark asynchrone"""
    client = await aioredis.Redis(decode_responses=True)

    start = time.time()

    # Exécuter toutes les opérations en parallèle
    tasks = []
    for i in range(iterations):
        tasks.append(client.set(f'key:{i}', f'value:{i}'))
        tasks.append(client.get(f'key:{i}'))

    await asyncio.gather(*tasks)

    duration = time.time() - start
    ops_per_sec = (iterations * 2) / duration
    print(f"Async: {duration:.2f}s ({ops_per_sec:.0f} ops/sec)")

    await client.close()
    return duration

# Comparaison
sync_time = benchmark_sync(1000)
async_time = asyncio.run(benchmark_async(1000))

speedup = sync_time / async_time
print(f"\n🚀 Speedup: {speedup:.1f}x plus rapide avec async !")

# Output typique :
# Sync: 1.85s (1081 ops/sec)
# Async: 0.05s (40000 ops/sec)
# 🚀 Speedup: 37.0x plus rapide avec async !
```

---

## Bonnes pratiques

### ✅ 1. Utiliser async pour I/O-bound workloads

```python
# ✅ BON : I/O-bound (réseau) → async
async def fetch_users(user_ids):
    tasks = [redis.get(f'user:{uid}') for uid in user_ids]
    return await asyncio.gather(*tasks)

# ❌ MAUVAIS : CPU-bound → async n'aide pas
async def compute_fibonacci(n):
    # Calcul intensif CPU : async inutile
    return fib(n)
```

### ✅ 2. Limiter la concurrence

```javascript
// ✅ Limiter à N opérations concurrentes
import pLimit from 'p-limit';
const limit = pLimit(10);

const promises = keys.map(key =>
    limit(() => redis.get(key))
);
```

### ✅ 3. Gérer les timeouts

```python
# ✅ Toujours définir un timeout
try:
    result = await asyncio.wait_for(
        redis.get('key'),
        timeout=5.0
    )
except asyncio.TimeoutError:
    # Gérer le timeout
    pass
```

### ✅ 4. Utiliser le pipeline pour les opérations séquentielles

```javascript
// ❌ MAUVAIS : Multiple round-trips
await redis.set('key1', 'val1');
await redis.set('key2', 'val2');
await redis.set('key3', 'val3');

// ✅ BON : 1 seul round-trip
const pipeline = redis.pipeline();
pipeline.set('key1', 'val1');
pipeline.set('key2', 'val2');
pipeline.set('key3', 'val3');
await pipeline.exec();
```

### ✅ 5. Cleanup avec try/finally

```python
# ✅ Toujours fermer les ressources
client = await redis.Redis()
try:
    await client.get('key')
finally:
    await client.close()

# Ou utiliser context manager
async with redis.Redis() as client:
    await client.get('key')
```

---

## Anti-patterns à éviter

### ❌ 1. Oublier await

```javascript
// ❌ ERREUR : Oubli de await
const value = redis.get('key'); // Retourne une Promise !
console.log(value); // Promise { <pending> }

// ✅ CORRECT
const value = await redis.get('key');
console.log(value); // 'actual value'
```

### ❌ 2. Trop de concurrence

```python
# ❌ MAUVAIS : 10000 requêtes simultanées
tasks = [redis.get(f'key:{i}') for i in range(10000)]
await asyncio.gather(*tasks) # Surcharge !

# ✅ BON : Limiter avec semaphore
sem = asyncio.Semaphore(50)
async def limited_get(key):
    async with sem:
        return await redis.get(key)

tasks = [limited_get(f'key:{i}') for i in range(10000)]
await asyncio.gather(*tasks)
```

### ❌ 3. Mixing sync et async

```python
# ❌ ERREUR : Mélanger sync et async
import redis
import redis.asyncio as aioredis

sync_client = redis.Redis()
async_client = await aioredis.Redis()

# Ceci bloquera l'event loop !
sync_client.get('key')  # ❌ Dans une fonction async

# ✅ BON : Utiliser uniquement async
await async_client.get('key')
```

### ❌ 4. Ne pas gérer les erreurs

```javascript
// ❌ MAUVAIS : Pas de gestion d'erreurs
const promises = keys.map(key => redis.get(key));
const results = await Promise.all(promises);
// Si UNE Promise échoue, TOUTES échouent !

// ✅ BON : Utiliser Promise.allSettled
const promises = keys.map(key => redis.get(key));
const results = await Promise.allSettled(promises);
// Continue même si certaines échouent
```

---

## Points clés à retenir

🔑 **Async = I/O-bound** : Utilisez async pour opérations réseau/disque

🔑 **Concurrence ≠ Parallélisme** : Async ne crée pas de threads

🔑 **Toujours limiter** : N'exécutez pas 10000 requêtes simultanées

🔑 **Pipeline pour séquences** : Groupez les opérations séquentielles

🔑 **Timeouts obligatoires** : Définissez toujours des timeouts

🔑 **Promise.all vs allSettled** : Choisissez selon le cas d'usage

🔑 **Cleanup essentiel** : Fermez toujours les connexions

🔑 **Benchmark votre code** : Mesurez les gains réels

---

## Ressources complémentaires

- **Python asyncio** : https://docs.python.org/3/library/asyncio.html
- **redis-py async** : https://redis-py.readthedocs.io/en/stable/examples/asyncio_examples.html
- **Node.js async patterns** : https://nodejs.org/en/docs/guides/blocking-vs-non-blocking/
- **Go concurrency** : https://go.dev/tour/concurrency/1

---

## Prochaine section

➡️ **Section 9.5** : Bonnes pratiques de développement - Patterns et architecture

**Niveau** : Intermédiaire
**Durée estimée** : 60 minutes
**Prérequis** : Sections 9.1-9.4

⏭️ [Bonnes pratiques de développement](/09-integration-langages-programmation/05-bonnes-pratiques-developpement.md)
