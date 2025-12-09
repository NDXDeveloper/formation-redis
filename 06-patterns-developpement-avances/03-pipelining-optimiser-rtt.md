🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3 Pipelining : Optimiser le RTT (Round Trip Time)

## Introduction

Le pipelining est l'une des optimisations les plus efficaces pour améliorer les performances Redis. En réduisant drastiquement le nombre d'allers-retours réseau, on peut multiplier le débit par 10, voire 100 dans certains scénarios. Pourtant, cette technique reste sous-utilisée par manque de compréhension de son fonctionnement.

Cette section explore en profondeur le pipelining : comment il fonctionne, quand l'utiliser, et comment l'implémenter correctement.

## Le problème du RTT (Round Trip Time)

### Qu'est-ce que le RTT ?

Le **Round Trip Time** est le temps nécessaire pour qu'un paquet de données fasse l'aller-retour entre le client et le serveur Redis.

```text
Client                                    Redis Server
  │                                             │
  │────── SET user:1 "Alice" ─────────────────> │
  │                                             │ Process: 0.1ms
  │                                             │
  │<────────── OK ──────────────────────────────│
  │                                             │
  └─ Total Time = RTT + Processing Time
```

Typical values:
- Same machine:     0.1 - 0.5 ms
- Same datacenter:  1 - 5 ms
- Cross-region:     50 - 200 ms
- Cross-continent:  100 - 500 ms

### Impact du RTT sans pipelining

**Scenario: 1000 commandes SET à exécuter**

Sans pipelining (séquentiel):

```text
Command 1: SET key1 → [RTT: 5ms] → OK
Command 2: SET key2 → [RTT: 5ms] → OK
Command 3: SET key3 → [RTT: 5ms] → OK
...
Command 1000: SET key1000 → [RTT: 5ms] → OK

Total time = 1000 × 5ms = 5000ms = 5 seconds
Processing time = 1000 × 0.01ms = 10ms
Network overhead = 4990ms (99.8% du temps!)

Throughput: 1000 / 5 = 200 ops/sec
```

Avec pipelining:

```text
Batch 1: SET key1, key2, ..., key100 → [RTT: 5ms] → OK...
Batch 2: SET key101, key102, ..., key200 → [RTT: 5ms] → OK...
...
Batch 10: SET key901, ..., key1000 → [RTT: 5ms] → OK...

Total time = 10 × 5ms = 50ms
Processing time = 1000 × 0.01ms = 10ms
Network overhead = 40ms (44% du temps)

Throughput: 1000 / 0.05 = 20,000 ops/sec

Speedup: 100x faster! 🚀
```

### Visualisation du problème

WITHOUT PIPELINING (Sequential):

```text
Time →
     Request  Wait  Response  Request  Wait  Response
     ▼        ████  ▲         ▼        ████  ▲
Client  →  [Network]  →  Redis

     ├─RTT─┤        ├─RTT─┤        ├─RTT─┤

Total = N × RTT (N = number of commands)
```

WITH PIPELINING (Batch):

```text
Time →
     Requests (bulk)      Wait      Responses (bulk)
     ▼▼▼▼▼▼▼▼▼▼          ████      ▲▲▲▲▲▲▲▲▲▲
Client  ────────────→  [Network]  ←────────────  Redis

     ├─────────────RTT─────────────┤

Total = 1 × RTT (regardless of N!)
```

---

## Comment fonctionne le pipelining

### Architecture Redis et RESP Protocol

Redis utilise le protocole RESP (REdis Serialization Protocol) qui supporte nativement le pipelining.

**RESP Protocol Flow:**

```text
CLIENT SIDE:
1. Queue multiple commands in memory
2. Send all commands in one TCP packet
3. Flush the buffer to the network
4. Wait for all responses

SERVER SIDE:
1. Receive buffer with multiple commands
2. Parse commands sequentially
3. Execute commands (single-threaded)
4. Buffer all responses
5. Send all responses in one TCP packet

KEY POINTS:
- Commands still execute sequentially on server
- No parallelization (Redis is single-threaded)
- Benefit comes from reducing network RTT
- Server buffers responses until all are ready
```

### Diagramme détaillé

```text
┌─────────────────────────────────────────────────────────────┐
│                    PIPELINING FLOW                           │
└─────────────────────────────────────────────────────────────┘

CLIENT APPLICATION
     │
     │ 1. Enqueue commands
     ├── pipe.set("key1", "val1")
     ├── pipe.set("key2", "val2")
     ├── pipe.get("key1")
     ├── pipe.incr("counter")
     │
     │ 2. Execute pipeline
     ↓
CLIENT BUFFER
     │ Commands serialized to RESP:
     │ *3\r\n$3\r\nSET\r\n$4\r\nkey1\r\n$4\r\nval1\r\n
     │ *3\r\n$3\r\nSET\r\n$4\r\nkey2\r\n$4\r\nval2\r\n
     │ *2\r\n$3\r\nGET\r\n$4\r\nkey1\r\n
     │ *2\r\n$4\r\nINCR\r\n$7\r\ncounter\r\n
     │
     │ 3. Single TCP send
     ↓
═════════════════════════════════════════════════════════════
                       NETWORK (1 RTT)
═════════════════════════════════════════════════════════════
     │
     ↓
REDIS SERVER
     │ 4. Parse commands
     ├── Command 1: SET key1 val1  → Execute → Buffer: OK
     ├── Command 2: SET key2 val2  → Execute → Buffer: OK
     ├── Command 3: GET key1       → Execute → Buffer: "val1"
     └── Command 4: INCR counter   → Execute → Buffer: 1
     │
     │ 5. Send all responses
     ↓
RESPONSE BUFFER
     │ +OK\r\n
     │ +OK\r\n
     │ $4\r\nval1\r\n
     │ :1\r\n
     │
     │ 6. Single TCP send
     ↓
═════════════════════════════════════════════════════════════
                       NETWORK (1 RTT)
═════════════════════════════════════════════════════════════
     │
     ↓
CLIENT APPLICATION
     │ 7. Parse responses
     └── [OK, OK, "val1", 1]
```

---

## Implémentation Python

### Basic Pipelining

```python
import redis
import time

# Connexion Redis
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# ============================================================
# SANS PIPELINING (Baseline)
# ============================================================

def without_pipeline(n=1000):
    """Exécution séquentielle sans pipeline"""
    start = time.time()

    for i in range(n):
        r.set(f"user:{i}", f"User {i}")

    elapsed = time.time() - start
    ops_per_sec = n / elapsed

    print(f"WITHOUT Pipeline:")
    print(f"  Commands: {n}")
    print(f"  Time: {elapsed:.3f}s")
    print(f"  Throughput: {ops_per_sec:.0f} ops/sec\n")

    return elapsed


# ============================================================
# AVEC PIPELINING
# ============================================================

def with_pipeline(n=1000):
    """Exécution avec pipeline"""
    start = time.time()

    # Créer un pipeline
    pipe = r.pipeline()

    # Enqueuer toutes les commandes
    for i in range(n):
        pipe.set(f"user:{i}", f"User {i}")

    # Exécuter en une seule fois
    pipe.execute()

    elapsed = time.time() - start
    ops_per_sec = n / elapsed

    print(f"WITH Pipeline:")
    print(f"  Commands: {n}")
    print(f"  Time: {elapsed:.3f}s")
    print(f"  Throughput: {ops_per_sec:.0f} ops/sec\n")

    return elapsed


# ============================================================
# PIPELINING PAR BATCH
# ============================================================

def with_pipeline_batched(n=1000, batch_size=100):
    """
    Pipelining par batch pour limiter l'utilisation mémoire
    """
    start = time.time()

    for batch_start in range(0, n, batch_size):
        batch_end = min(batch_start + batch_size, n)

        pipe = r.pipeline()
        for i in range(batch_start, batch_end):
            pipe.set(f"user:{i}", f"User {i}")

        pipe.execute()

    elapsed = time.time() - start
    ops_per_sec = n / elapsed

    print(f"WITH Pipeline (Batched):")
    print(f"  Commands: {n}")
    print(f"  Batch size: {batch_size}")
    print(f"  Time: {elapsed:.3f}s")
    print(f"  Throughput: {ops_per_sec:.0f} ops/sec\n")

    return elapsed


# ============================================================
# EXEMPLE PRATIQUE : CHARGEMENT EN MASSE
# ============================================================

def bulk_load_users(users_data: list):
    """
    Charge un grand nombre d'utilisateurs efficacement
    """
    pipe = r.pipeline()

    for user in users_data:
        # Stocker le user dans un hash
        pipe.hset(
            f"user:{user['id']}",
            mapping={
                'name': user['name'],
                'email': user['email'],
                'created_at': user['created_at']
            }
        )

        # Ajouter à un index
        pipe.sadd('users:all', user['id'])

        # Indexer par email
        pipe.set(f"user:email:{user['email']}", user['id'])

    # Exécuter toutes les opérations
    results = pipe.execute()

    print(f"Loaded {len(users_data)} users")
    print(f"Total operations: {len(results)}")

    return results


# ============================================================
# RÉCUPÉRATION EN MASSE
# ============================================================

def bulk_get_users(user_ids: list):
    """
    Récupère plusieurs utilisateurs en une seule opération
    """
    pipe = r.pipeline()

    # Enqueuer toutes les lectures
    for user_id in user_ids:
        pipe.hgetall(f"user:{user_id}")

    # Exécuter et récupérer les résultats
    results = pipe.execute()

    # Parser les résultats
    users = []
    for i, user_data in enumerate(results):
        if user_data:
            users.append({
                'id': user_ids[i],
                **user_data
            })

    return users


# ============================================================
# OPÉRATIONS MIXTES
# ============================================================

def mixed_operations_example():
    """
    Exemple avec différents types d'opérations dans un pipeline
    """
    pipe = r.pipeline()

    # SET operations
    pipe.set("config:max_users", 1000)
    pipe.set("config:timeout", 30)

    # GET operations
    pipe.get("config:max_users")

    # INCR operations
    pipe.incr("stats:page_views")
    pipe.incr("stats:api_calls")

    # HASH operations
    pipe.hset("user:123", "name", "Alice")
    pipe.hget("user:123", "name")

    # LIST operations
    pipe.lpush("queue:jobs", "job1", "job2", "job3")
    pipe.llen("queue:jobs")

    # SET operations
    pipe.sadd("tags", "python", "redis", "database")
    pipe.scard("tags")

    # Exécuter tout
    results = pipe.execute()

    print("Mixed operations results:")
    for i, result in enumerate(results):
        print(f"  Operation {i+1}: {result}")

    return results


# ============================================================
# GESTION DES ERREURS
# ============================================================

def pipeline_with_error_handling():
    """
    Gestion des erreurs dans un pipeline
    """
    pipe = r.pipeline()

    # Commandes valides et invalides
    pipe.set("key1", "value1")
    pipe.incr("key1")  # ❌ Erreur : key1 est une string, pas un int
    pipe.set("key2", "value2")

    try:
        results = pipe.execute()
        print("Results:", results)
    except redis.exceptions.ResponseError as e:
        print(f"Pipeline error: {e}")
        # Le pipeline s'arrête à la première erreur
        # Les commandes suivantes ne sont pas exécutées


# ============================================================
# BENCHMARK COMPLET
# ============================================================

def run_benchmark():
    """
    Benchmark complet pour comparer les performances
    """
    print("=" * 60)
    print("REDIS PIPELINING BENCHMARK")
    print("=" * 60 + "\n")

    # Test avec différentes tailles
    for n in [100, 1000, 10000]:
        print(f"\n--- Test with {n} commands ---\n")

        # Sans pipeline
        time_without = without_pipeline(n)

        # Avec pipeline
        time_with = with_pipeline(n)

        # Avec pipeline batché
        time_batched = with_pipeline_batched(n, batch_size=100)

        # Calculer le speedup
        speedup = time_without / time_with
        print(f"Speedup: {speedup:.1f}x faster with pipelining\n")
        print("-" * 60)


# ============================================================
# CAS D'USAGE RÉEL : IMPORT CSV
# ============================================================

import csv
from datetime import datetime

def import_csv_to_redis(csv_file: str, batch_size: int = 1000):
    """
    Importe un fichier CSV dans Redis avec pipelining
    """
    total_imported = 0
    start = time.time()

    with open(csv_file, 'r') as f:
        reader = csv.DictReader(f)

        pipe = r.pipeline()
        count = 0

        for row in reader:
            # Créer l'entrée user
            user_id = row['id']

            pipe.hset(
                f"user:{user_id}",
                mapping={
                    'name': row['name'],
                    'email': row['email'],
                    'age': row['age'],
                    'city': row['city']
                }
            )

            # Indexation
            pipe.sadd('users:all', user_id)
            pipe.zadd('users:by_age', {user_id: int(row['age'])})

            count += 1

            # Exécuter par batch
            if count >= batch_size:
                pipe.execute()
                total_imported += count
                count = 0
                pipe = r.pipeline()

                print(f"Imported {total_imported} users...")

        # Exécuter le dernier batch
        if count > 0:
            pipe.execute()
            total_imported += count

    elapsed = time.time() - start

    print(f"\n✓ Import completed:")
    print(f"  Total users: {total_imported}")
    print(f"  Time: {elapsed:.2f}s")
    print(f"  Throughput: {total_imported / elapsed:.0f} users/sec")


# ============================================================
# MAIN
# ============================================================

if __name__ == '__main__':
    # Nettoyer Redis
    r.flushdb()

    # Lancer le benchmark
    run_benchmark()

    # Exemple d'opérations mixtes
    print("\n" + "=" * 60)
    print("MIXED OPERATIONS EXAMPLE")
    print("=" * 60 + "\n")
    mixed_operations_example()

    # Exemple de bulk load
    print("\n" + "=" * 60)
    print("BULK LOAD EXAMPLE")
    print("=" * 60 + "\n")

    users = [
        {
            'id': i,
            'name': f'User {i}',
            'email': f'user{i}@example.com',
            'created_at': datetime.now().isoformat()
        }
        for i in range(1, 101)
    ]

    bulk_load_users(users)

    # Exemple de bulk get
    print("\n" + "=" * 60)
    print("BULK GET EXAMPLE")
    print("=" * 60 + "\n")

    user_ids = [1, 5, 10, 20, 50]
    retrieved_users = bulk_get_users(user_ids)

    print(f"Retrieved {len(retrieved_users)} users:")
    for user in retrieved_users[:3]:
        print(f"  - {user['name']} ({user['email']})")
```

---

## Implémentation Node.js

### Basic Pipelining avec ioredis

```javascript
const Redis = require('ioredis');

// Connexion Redis
const redis = new Redis();

// ============================================================
// SANS PIPELINING
// ============================================================

async function withoutPipeline(n = 1000) {
    console.log('WITHOUT Pipeline:');
    const start = Date.now();

    for (let i = 0; i < n; i++) {
        await redis.set(`user:${i}`, `User ${i}`);
    }

    const elapsed = (Date.now() - start) / 1000;
    const opsPerSec = n / elapsed;

    console.log(`  Commands: ${n}`);
    console.log(`  Time: ${elapsed.toFixed(3)}s`);
    console.log(`  Throughput: ${opsPerSec.toFixed(0)} ops/sec\n`);

    return elapsed;
}

// ============================================================
// AVEC PIPELINING
// ============================================================

async function withPipeline(n = 1000) {
    console.log('WITH Pipeline:');
    const start = Date.now();

    // Créer un pipeline
    const pipeline = redis.pipeline();

    // Enqueuer toutes les commandes
    for (let i = 0; i < n; i++) {
        pipeline.set(`user:${i}`, `User ${i}`);
    }

    // Exécuter en une seule fois
    await pipeline.exec();

    const elapsed = (Date.now() - start) / 1000;
    const opsPerSec = n / elapsed;

    console.log(`  Commands: ${n}`);
    console.log(`  Time: ${elapsed.toFixed(3)}s`);
    console.log(`  Throughput: ${opsPerSec.toFixed(0)} ops/sec\n`);

    return elapsed;
}

// ============================================================
// PIPELINING PAR BATCH
// ============================================================

async function withPipelineBatched(n = 1000, batchSize = 100) {
    console.log('WITH Pipeline (Batched):');
    const start = Date.now();

    for (let batchStart = 0; batchStart < n; batchStart += batchSize) {
        const batchEnd = Math.min(batchStart + batchSize, n);

        const pipeline = redis.pipeline();
        for (let i = batchStart; i < batchEnd; i++) {
            pipeline.set(`user:${i}`, `User ${i}`);
        }

        await pipeline.exec();
    }

    const elapsed = (Date.now() - start) / 1000;
    const opsPerSec = n / elapsed;

    console.log(`  Commands: ${n}`);
    console.log(`  Batch size: ${batchSize}`);
    console.log(`  Time: ${elapsed.toFixed(3)}s`);
    console.log(`  Throughput: ${opsPerSec.toFixed(0)} ops/sec\n`);

    return elapsed;
}

// ============================================================
// BULK LOAD AVEC PROMISE.ALL (Anti-pattern!)
// ============================================================

async function bulkLoadWithPromiseAll(n = 1000) {
    console.log('WITH Promise.all (Anti-pattern):');
    const start = Date.now();

    // ❌ MAUVAIS : Crée N connexions/promesses
    const promises = [];
    for (let i = 0; i < n; i++) {
        promises.push(redis.set(`user:${i}`, `User ${i}`));
    }

    await Promise.all(promises);

    const elapsed = (Date.now() - start) / 1000;
    const opsPerSec = n / elapsed;

    console.log(`  Commands: ${n}`);
    console.log(`  Time: ${elapsed.toFixed(3)}s`);
    console.log(`  Throughput: ${opsPerSec.toFixed(0)} ops/sec`);
    console.log(`  ⚠️  Uses multiple connections - not recommended!\n`);

    return elapsed;
}

// ============================================================
// CHARGEMENT EN MASSE
// ============================================================

async function bulkLoadUsers(usersData) {
    const pipeline = redis.pipeline();

    for (const user of usersData) {
        // Stocker le user dans un hash
        pipeline.hset(
            `user:${user.id}`,
            'name', user.name,
            'email', user.email,
            'created_at', user.created_at
        );

        // Ajouter à un index
        pipeline.sadd('users:all', user.id);

        // Indexer par email
        pipeline.set(`user:email:${user.email}`, user.id);
    }

    const results = await pipeline.exec();

    console.log(`Loaded ${usersData.length} users`);
    console.log(`Total operations: ${results.length}`);

    return results;
}

// ============================================================
// RÉCUPÉRATION EN MASSE
// ============================================================

async function bulkGetUsers(userIds) {
    const pipeline = redis.pipeline();

    // Enqueuer toutes les lectures
    for (const userId of userIds) {
        pipeline.hgetall(`user:${userId}`);
    }

    // Exécuter et récupérer les résultats
    const results = await pipeline.exec();

    // Parser les résultats
    const users = [];
    for (let i = 0; i < results.length; i++) {
        const [err, userData] = results[i];
        if (!err && userData && Object.keys(userData).length > 0) {
            users.push({
                id: userIds[i],
                ...userData
            });
        }
    }

    return users;
}

// ============================================================
// OPÉRATIONS MIXTES
// ============================================================

async function mixedOperationsExample() {
    const pipeline = redis.pipeline();

    // SET operations
    pipeline.set('config:max_users', 1000);
    pipeline.set('config:timeout', 30);

    // GET operations
    pipeline.get('config:max_users');

    // INCR operations
    pipeline.incr('stats:page_views');
    pipeline.incr('stats:api_calls');

    // HASH operations
    pipeline.hset('user:123', 'name', 'Alice');
    pipeline.hget('user:123', 'name');

    // LIST operations
    pipeline.lpush('queue:jobs', 'job1', 'job2', 'job3');
    pipeline.llen('queue:jobs');

    // SET operations
    pipeline.sadd('tags', 'python', 'redis', 'database');
    pipeline.scard('tags');

    // Exécuter tout
    const results = await pipeline.exec();

    console.log('Mixed operations results:');
    results.forEach(([err, result], index) => {
        console.log(`  Operation ${index + 1}: ${result}`);
    });

    return results;
}

// ============================================================
// GESTION DES ERREURS
// ============================================================

async function pipelineWithErrorHandling() {
    const pipeline = redis.pipeline();

    // Commandes valides et invalides
    pipeline.set('key1', 'value1');
    pipeline.incr('key1');  // ❌ Erreur : key1 est une string
    pipeline.set('key2', 'value2');

    try {
        const results = await pipeline.exec();

        console.log('Pipeline executed. Results:');
        results.forEach(([err, result], index) => {
            if (err) {
                console.log(`  Command ${index + 1}: ERROR - ${err.message}`);
            } else {
                console.log(`  Command ${index + 1}: ${result}`);
            }
        });
    } catch (error) {
        console.error('Pipeline error:', error);
    }
}

// ============================================================
// PIPELINING AVEC TRANSACTIONS (MULTI/EXEC)
// ============================================================

async function pipelineWithTransaction() {
    /**
     * En Node.js (ioredis), pipeline() et multi() sont similaires
     * mais multi() garantit l'atomicité
     */

    const pipeline = redis.multi();  // ou redis.pipeline()

    pipeline.set('balance:user1', 1000);
    pipeline.decrby('balance:user1', 100);
    pipeline.incrby('balance:user2', 100);

    const results = await pipeline.exec();

    console.log('Transaction results:');
    results.forEach(([err, result], index) => {
        console.log(`  Command ${index + 1}: ${result}`);
    });

    return results;
}

// ============================================================
// PATTERN : STREAMING AVEC PIPELINE
// ============================================================

const { Readable } = require('stream');

class RedisPipelineStream extends Readable {
    constructor(dataGenerator, batchSize = 100) {
        super({ objectMode: true });
        this.dataGenerator = dataGenerator;
        this.batchSize = batchSize;
        this.buffer = [];
    }

    _read() {
        const batch = this.dataGenerator(this.batchSize);

        if (batch.length === 0) {
            this.push(null);  // End of stream
            return;
        }

        const pipeline = redis.pipeline();

        for (const item of batch) {
            pipeline.set(`item:${item.id}`, JSON.stringify(item));
        }

        pipeline.exec().then(results => {
            this.push({ batch: batch.length, results: results.length });
        });
    }
}

// ============================================================
// BENCHMARK COMPLET
// ============================================================

async function runBenchmark() {
    console.log('='.repeat(60));
    console.log('REDIS PIPELINING BENCHMARK (Node.js)');
    console.log('='.repeat(60) + '\n');

    for (const n of [100, 1000, 10000]) {
        console.log(`\n--- Test with ${n} commands ---\n`);

        // Sans pipeline
        const timeWithout = await withoutPipeline(n);

        // Avec pipeline
        const timeWith = await withPipeline(n);

        // Avec pipeline batché
        const timeBatched = await withPipelineBatched(n, 100);

        // Promise.all (anti-pattern)
        const timePromiseAll = await bulkLoadWithPromiseAll(n);

        // Calculer le speedup
        const speedup = timeWithout / timeWith;
        console.log(`Speedup: ${speedup.toFixed(1)}x faster with pipelining\n`);
        console.log('-'.repeat(60));
    }
}

// ============================================================
// CAS D'USAGE : ANALYTICS BATCH WRITE
// ============================================================

async function batchWriteAnalytics(events) {
    /**
     * Écrit un batch d'événements analytics dans Redis
     */
    const pipeline = redis.pipeline();

    for (const event of events) {
        const timestamp = Math.floor(event.timestamp / 1000);
        const hour = Math.floor(timestamp / 3600) * 3600;

        // Incrémenter les compteurs
        pipeline.hincrby(`analytics:${hour}`, event.type, 1);

        // Ajouter à la série temporelle
        pipeline.zadd(
            `events:${event.type}`,
            timestamp,
            JSON.stringify(event)
        );

        // Compter les users uniques (HyperLogLog)
        pipeline.pfadd(`users:${hour}`, event.userId);
    }

    const start = Date.now();
    await pipeline.exec();
    const elapsed = Date.now() - start;

    console.log(`Wrote ${events.length} analytics events in ${elapsed}ms`);
    console.log(`Throughput: ${(events.length / elapsed * 1000).toFixed(0)} events/sec`);
}

// ============================================================
// PATTERN : CACHE WARMING
// ============================================================

async function warmCache(keys) {
    /**
     * Réchauffe le cache pour plusieurs clés
     */
    console.log(`Warming cache for ${keys.length} keys...`);

    const batchSize = 100;
    let warmed = 0;

    for (let i = 0; i < keys.length; i += batchSize) {
        const batch = keys.slice(i, i + batchSize);
        const pipeline = redis.pipeline();

        for (const key of batch) {
            // Simuler le chargement de données
            const data = await fetchDataForKey(key);
            pipeline.setex(key, 3600, JSON.stringify(data));
        }

        await pipeline.exec();
        warmed += batch.length;

        console.log(`  Warmed ${warmed}/${keys.length} keys...`);
    }

    console.log('✓ Cache warming completed');
}

async function fetchDataForKey(key) {
    // Simuler une requête DB
    return { key, data: `Data for ${key}`, timestamp: Date.now() };
}

// ============================================================
// MAIN
// ============================================================

async function main() {
    try {
        // Nettoyer Redis
        await redis.flushdb();

        // Lancer le benchmark
        await runBenchmark();

        // Exemple d'opérations mixtes
        console.log('\n' + '='.repeat(60));
        console.log('MIXED OPERATIONS EXAMPLE');
        console.log('='.repeat(60) + '\n');
        await mixedOperationsExample();

        // Exemple de gestion d'erreurs
        console.log('\n' + '='.repeat(60));
        console.log('ERROR HANDLING EXAMPLE');
        console.log('='.repeat(60) + '\n');
        await pipelineWithErrorHandling();

        // Exemple de bulk load
        console.log('\n' + '='.repeat(60));
        console.log('BULK LOAD EXAMPLE');
        console.log('='.repeat(60) + '\n');

        const users = Array.from({ length: 100 }, (_, i) => ({
            id: i + 1,
            name: `User ${i + 1}`,
            email: `user${i + 1}@example.com`,
            created_at: new Date().toISOString()
        }));

        await bulkLoadUsers(users);

        // Exemple de bulk get
        console.log('\n' + '='.repeat(60));
        console.log('BULK GET EXAMPLE');
        console.log('='.repeat(60) + '\n');

        const userIds = [1, 5, 10, 20, 50];
        const retrievedUsers = await bulkGetUsers(userIds);

        console.log(`Retrieved ${retrievedUsers.length} users:`);
        retrievedUsers.slice(0, 3).forEach(user => {
            console.log(`  - ${user.name} (${user.email})`);
        });

        // Exemple d'analytics
        console.log('\n' + '='.repeat(60));
        console.log('ANALYTICS BATCH WRITE');
        console.log('='.repeat(60) + '\n');

        const events = Array.from({ length: 1000 }, (_, i) => ({
            type: ['page_view', 'click', 'purchase'][i % 3],
            userId: Math.floor(Math.random() * 1000),
            timestamp: Date.now() - Math.floor(Math.random() * 3600000)
        }));

        await batchWriteAnalytics(events);

    } catch (error) {
        console.error('Error:', error);
    } finally {
        await redis.quit();
    }
}

// Lancer le script
main();
```

---

## Pipelining vs Transactions

### Différences clés

| Feature | Pipeline | Transaction |
|---------|----------|-------------|
| **Atomicity** | ❌ No | ✅ Yes |
| **Execution order** | ✅ Guaranteed | ✅ Guaranteed |
| **Error handling** | Partial success | All-or-nothing |
| **Performance** | ⚡ Excellent | ⚡ Excellent |
| **Network RTT** | 1 RTT | 2 RTT* |
| **Other clients block** | ❌ No | ❌ No |
| **Use case** | Bulk operations | Critical updates |

\* MULTI + commands + EXEC = 2 round trips minimum

### Exemples comparatifs

```python
# ============================================================
# PIPELINE : Pas atomique
# ============================================================

def pipeline_example():
    pipe = r.pipeline()

    pipe.set('key1', 'value1')
    pipe.set('key2', 'value2')
    pipe.incr('key1')  # ❌ Erreur : key1 n'est pas un integer
    pipe.set('key3', 'value3')

    results = pipe.execute()

    # Résultat : key1 et key2 sont créés, key3 NON
    # Les commandes sont partiellement exécutées
    # ⚠️ État inconsistant possible


# ============================================================
# TRANSACTION : Atomique
# ============================================================

def transaction_example():
    pipe = r.pipeline()

    # Démarrer une transaction
    pipe.multi()

    pipe.set('key1', 'value1')
    pipe.set('key2', 'value2')
    pipe.incr('key1')  # ❌ Erreur
    pipe.set('key3', 'value3')

    try:
        results = pipe.execute()
    except redis.exceptions.ResponseError:
        # Toute la transaction est annulée
        # AUCUNE commande n'est exécutée
        # ✅ État consistant garanti
        pass


# ============================================================
# QUAND UTILISER QUOI ?
# ============================================================

# ✅ PIPELINE : Bulk operations sans besoin d'atomicité
def use_pipeline():
    pipe = r.pipeline()
    for i in range(1000):
        pipe.set(f"log:{i}", f"Log entry {i}")
    pipe.execute()
    # Si une échoue, ce n'est pas critique

# ✅ TRANSACTION : Opérations critiques
def use_transaction():
    pipe = r.pipeline()
    pipe.multi()

    # Transférer de l'argent (atomicité requise!)
    pipe.decrby('balance:user1', 100)
    pipe.incrby('balance:user2', 100)

    pipe.execute()
    # Les deux doivent réussir ou échouer ensemble
```

---

## Bonnes pratiques et optimisations

### 1. Choisir la bonne taille de batch

```python
import math

def optimal_batch_size(total_commands: int,
                       avg_command_size: int = 50,
                       max_memory_mb: int = 10) -> int:
    """
    Calcule la taille optimale de batch pour le pipelining

    Args:
        total_commands: Nombre total de commandes
        avg_command_size: Taille moyenne d'une commande (bytes)
        max_memory_mb: Mémoire max à utiliser pour le buffer

    Returns:
        Taille de batch recommandée
    """
    max_memory_bytes = max_memory_mb * 1024 * 1024

    # Calculer combien de commandes tiennent en mémoire
    max_commands_in_memory = max_memory_bytes // avg_command_size

    # Choisir un batch size raisonnable
    if total_commands <= max_commands_in_memory:
        # Tout tient en mémoire, un seul batch
        return total_commands
    else:
        # Découper en plusieurs batches
        # Viser 100-1000 commandes par batch pour équilibrer
        recommended = min(max_commands_in_memory, 1000)
        return recommended


# Exemple
batch_size = optimal_batch_size(
    total_commands=100000,
    avg_command_size=100,  # 100 bytes par commande
    max_memory_mb=10       # 10 MB de buffer max
)

print(f"Recommended batch size: {batch_size}")
```

### 2. Mesurer et monitorer

```python
import time
from contextlib import contextmanager

@contextmanager
def measure_pipeline(name: str):
    """Context manager pour mesurer les performances du pipeline"""
    start = time.time()
    commands_sent = 0

    def track_command():
        nonlocal commands_sent
        commands_sent += 1

    yield track_command

    elapsed = time.time() - start
    throughput = commands_sent / elapsed if elapsed > 0 else 0

    print(f"Pipeline '{name}':")
    print(f"  Commands: {commands_sent}")
    print(f"  Time: {elapsed:.3f}s")
    print(f"  Throughput: {throughput:.0f} ops/sec")


# Utilisation
with measure_pipeline("user_import") as track:
    pipe = r.pipeline()

    for i in range(1000):
        pipe.set(f"user:{i}", f"User {i}")
        track()

    pipe.execute()
```

### 3. Limiter la taille du pipeline

```python
class SmartPipeline:
    """
    Pipeline intelligent avec gestion automatique des batches
    """
    def __init__(self, redis_client, max_batch_size=1000):
        self.redis = redis_client
        self.max_batch_size = max_batch_size
        self.current_batch = []
        self.total_executed = 0

    def add_command(self, command, *args):
        """Ajoute une commande au pipeline"""
        self.current_batch.append((command, args))

        # Auto-flush si le batch est plein
        if len(self.current_batch) >= self.max_batch_size:
            self.flush()

    def flush(self):
        """Exécute toutes les commandes en attente"""
        if not self.current_batch:
            return []

        pipe = self.redis.pipeline()

        for command, args in self.current_batch:
            getattr(pipe, command)(*args)

        results = pipe.execute()

        self.total_executed += len(self.current_batch)
        self.current_batch = []

        return results

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # Flush automatique à la sortie du context
        self.flush()


# Utilisation
with SmartPipeline(r, max_batch_size=100) as pipe:
    for i in range(1000):
        pipe.add_command('set', f'key:{i}', f'value {i}')
        # Flush automatique tous les 100 commandes

print(f"Total executed: {pipe.total_executed}")
```

### 4. Gestion des erreurs avancée

```python
def robust_pipeline_execution(commands: list):
    """
    Exécute un pipeline avec gestion d'erreur robuste
    """
    pipe = r.pipeline()

    # Ajouter toutes les commandes
    for cmd, *args in commands:
        getattr(pipe, cmd)(*args)

    try:
        results = pipe.execute()

        # Vérifier les résultats individuels
        errors = []
        successes = []

        for i, result in enumerate(results):
            if isinstance(result, Exception):
                errors.append({
                    'index': i,
                    'command': commands[i],
                    'error': str(result)
                })
            else:
                successes.append({
                    'index': i,
                    'command': commands[i],
                    'result': result
                })

        return {
            'success_count': len(successes),
            'error_count': len(errors),
            'errors': errors
        }

    except Exception as e:
        print(f"Pipeline execution failed: {e}")
        return {
            'success_count': 0,
            'error_count': len(commands),
            'errors': [{'error': str(e)}]
        }


# Exemple
commands = [
    ('set', 'key1', 'value1'),
    ('incr', 'counter'),
    ('lpush', 'list', 'item1', 'item2'),
]

stats = robust_pipeline_execution(commands)
print(f"Successes: {stats['success_count']}")
print(f"Errors: {stats['error_count']}")
```

---

## Cas d'usage avancés

### 1. Déduplication avec pipeline

```python
def deduplicate_and_store(items: list):
    """
    Déduplique et stocke des items avec pipeline
    """
    # Phase 1: Vérifier l'existence en batch
    check_pipe = r.pipeline()
    for item in items:
        check_pipe.exists(f"item:{item['id']}")

    existence_results = check_pipe.execute()

    # Phase 2: Stocker seulement les nouveaux items
    store_pipe = r.pipeline()
    new_items = []

    for i, exists in enumerate(existence_results):
        if not exists:
            item = items[i]
            store_pipe.set(f"item:{item['id']}", json.dumps(item))
            new_items.append(item)

    if new_items:
        store_pipe.execute()

    print(f"Stored {len(new_items)} new items (skipped {len(items) - len(new_items)} duplicates)")
```

### 2. Pipeline avec retry logic

```python
import time
from typing import Callable

def pipeline_with_retry(
    execute_func: Callable,
    max_retries: int = 3,
    backoff: float = 0.1
) -> any:
    """
    Exécute un pipeline avec retry automatique
    """
    last_exception = None

    for attempt in range(max_retries):
        try:
            return execute_func()
        except redis.exceptions.ConnectionError as e:
            last_exception = e
            if attempt < max_retries - 1:
                wait_time = backoff * (2 ** attempt)
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)

    raise last_exception


# Utilisation
def my_pipeline_operation():
    pipe = r.pipeline()
    for i in range(100):
        pipe.set(f"key:{i}", f"value {i}")
    return pipe.execute()

results = pipeline_with_retry(my_pipeline_operation)
```

### 3. Pipeline distribué sur plusieurs instances Redis

```python
def distributed_pipeline(redis_instances: list, key_value_pairs: dict):
    """
    Distribue les écritures sur plusieurs instances Redis en parallèle
    """
    # Grouper par instance (hash-based partitioning)
    partitions = {i: [] for i in range(len(redis_instances))}

    for key, value in key_value_pairs.items():
        # Simple hash pour déterminer l'instance
        instance_idx = hash(key) % len(redis_instances)
        partitions[instance_idx].append((key, value))

    # Créer un pipeline par instance
    pipelines = []
    for instance_idx, instance in enumerate(redis_instances):
        pipe = instance.pipeline()

        for key, value in partitions[instance_idx]:
            pipe.set(key, value)

        pipelines.append(pipe)

    # Exécuter tous les pipelines en parallèle
    import concurrent.futures

    with concurrent.futures.ThreadPoolExecutor() as executor:
        futures = [executor.submit(pipe.execute) for pipe in pipelines]
        results = [f.result() for f in concurrent.futures.as_completed(futures)]

    return results
```

---

## Performance Tips

### Checklist d'optimisation

**Pipelining Checklist**

**Performance:**
- ✅ Utiliser le pipelining pour > 10 commandes
- ✅ Batch size entre 100-1000 pour équilibrer mémoire/réseau
- ✅ Éviter Promise.all en Node.js (utiliser pipeline)
- ✅ Mesurer le throughput (objectif: >10,000 ops/sec)

**Mémoire:**
- ✅ Limiter la taille du buffer côté client
- ✅ Flusher régulièrement pour libérer la mémoire
- ✅ Surveiller l'utilisation mémoire Redis (INFO memory)

**Réseau:**
- ✅ Pipeline = 1 RTT (vs N RTT sans pipeline)
- ✅ Considérer la latence réseau dans le batch size
- ✅ Utiliser la connexion locale si possible

**Fiabilité:**
- ✅ Gérer les erreurs individuelles
- ✅ Implémenter retry logic si nécessaire
- ✅ Utiliser MULTI/EXEC pour l'atomicité critique
- ✅ Monitorer les timeouts

**Monitoring:**
- ✅ Tracker le throughput (ops/sec)
- ✅ Mesurer la latence P50, P95, P99
- ✅ Alerter si throughput < seuil
- ✅ Logger les erreurs de pipeline

### Benchmarks de référence

**Typical Pipelining Performance (Local Redis):**

| Commands | Without Pipeline | With Pipeline | Speedup |
|----------|-----------------|---------------|---------|
| 100 | 50ms | 5ms | 10x |
| 1,000 | 500ms | 15ms | 33x |
| 10,000 | 5,000ms | 50ms | 100x |
| 100,000 | 50,000ms | 500ms | 100x |

**Network Impact (Same datacenter, 2ms RTT):**

| Commands | Without Pipeline | With Pipeline | Speedup |
|----------|-----------------|---------------|---------|
| 100 | 200ms | 10ms | 20x |
| 1,000 | 2,000ms | 25ms | 80x |
| 10,000 | 20,000ms | 100ms | 200x |

---

## Conclusion

Le pipelining est une technique essentielle pour optimiser les performances Redis. Les points clés à retenir :

**Avantages** :
- ⚡ Réduction drastique du temps total (10-100x plus rapide)
- 🚀 Augmentation massive du throughput
- 💾 Utilisation optimale de la bande passante
- 📊 Scalabilité améliorée

**Quand utiliser** :
- Bulk operations (import/export)
- Batch processing
- Analytics writes
- Cache warming
- Toute opération > 10 commandes

**Quand NE PAS utiliser** :
- Opérations nécessitant atomicité (utiliser MULTI/EXEC)
- Commandes interdépendantes
- Besoin de résultats intermédiaires
- Mémoire limitée côté client

**Best Practices** :
1. Toujours utiliser le pipelining pour > 10 commandes
2. Choisir un batch size adapté (100-1000)
3. Monitorer les performances
4. Gérer les erreurs correctement
5. Ne pas confondre pipeline et transaction

Le pipelining est gratuit en termes de complexité et apporte des gains énormes. Il n'y a aucune raison de ne pas l'utiliser dès que vous avez plusieurs commandes à exécuter!

---


⏭️ [Client-Side Caching (Tracking) : La fonctionnalité killer](/06-patterns-developpement-avances/04-client-side-caching.md)
