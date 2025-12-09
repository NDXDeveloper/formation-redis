🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 6 : Patterns de développement avancés

## Introduction

Redis n'est pas simplement une base de données clé-valeur rapide. C'est une plateforme puissante qui permet d'implémenter des patterns architecturaux sophistiqués pour résoudre des problèmes complexes de développement d'applications distribuées modernes.

Ce module explore les patterns de développement avancés qui exploitent les capacités uniques de Redis : sa rapidité, son modèle de données riche, ses opérations atomiques et ses primitives de synchronisation. Ces patterns sont éprouvés en production par des entreprises gérant des millions de requêtes par seconde.

## Pourquoi des patterns avancés ?

### Les défis des systèmes modernes

Les applications distribuées modernes font face à des défis complexes :

```
┌─────────────────────────────────────────────────────────────┐
│                   DÉFIS ARCHITECTURAUX                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  🔄 Scalabilité Horizontale                                 │
│     └─ Gérer des milliers de serveurs simultanément         │
│                                                             │
│  ⚡ Performance Sub-milliseconde                            │
│     └─ Répondre en moins de 1ms à 99.9% des requêtes        │
│                                                             │
│  🔒 Cohérence des Données                                   │
│     └─ Éviter les race conditions dans un système distribué │
│                                                             │
│  🛡️  Protection contre les Abus                             │
│     └─ Rate limiting, anti-spam, anti-fraude                │
│                                                             │
│  📊 Gestion d'État Distribué                                │
│     └─ Sessions, locks, compteurs partagés                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Redis comme solution

Redis offre des primitives uniques qui le rendent idéal pour ces patterns :

| Caractéristique | Bénéfice pour les patterns |
|----------------|----------------------------|
| **Opérations atomiques** | Évite les race conditions sans locking complexe |
| **Single-threaded** | Garantit l'ordre d'exécution des commandes |
| **Sub-millisecond latency** | Permet des décisions en temps réel |
| **Structures de données riches** | Modélise directement les problèmes métier |
| **Persistence optionnelle** | Balance entre vitesse et durabilité |
| **Pub/Sub & Streams** | Communication temps réel entre services |

## Architecture des patterns avancés

### Vue d'ensemble

```
┌────────────────────────────────────────────────────────────────┐
│                    PATTERNS REDIS AVANCÉS                      │
└────────────────────────────────────────────────────────────────┘
           │
           ├─── 🗄️  CACHING PATTERNS
           │    ├─ Cache-Aside (Lazy Loading)
           │    ├─ Write-Through (Eager Writing)
           │    ├─ Write-Back (Delayed Writing)
           │    └─ Cache-First (Read-Heavy)
           │
           ├─── 🛡️  RESILIENCE PATTERNS
           │    ├─ Anti-Avalanche (Circuit Breaker)
           │    ├─ Anti-Stampede (Lock + Queue)
           │    └─ Anti-Penetration (Bloom Filter)
           │
           ├─── ⚡ PERFORMANCE PATTERNS
           │    ├─ Pipelining (Batch Operations)
           │    ├─ Client-Side Caching (Local Cache)
           │    └─ Lazy Expiration (TTL Strategy)
           │
           ├─── 🔒 SYNCHRONIZATION PATTERNS
           │    ├─ Distributed Lock (Redlock)
           │    ├─ Semaphore (Resource Limiting)
           │    └─ Leader Election
           │
           ├─── 🚦 RATE LIMITING PATTERNS
           │    ├─ Fixed Window Counter
           │    ├─ Sliding Window Log
           │    ├─ Sliding Window Counter
           │    └─ Token Bucket
           │
           ├─── 💾 STATE MANAGEMENT PATTERNS
           │    ├─ Session Store (User State)
           │    ├─ Shopping Cart (E-commerce)
           │    └─ Real-time Presence (Online Users)
           │
           └─── 📬 MESSAGING PATTERNS
                ├─ Job Queue (Background Tasks)
                ├─ Priority Queue (Urgent Tasks)
                └─ Delayed Queue (Scheduled Tasks)
```

## Concepts fondamentaux

### 1. Atomicité et opérations composées

Redis garantit que chaque commande s'exécute atomiquement. Pour des opérations composées, trois approches existent :

```python
# ❌ MAUVAIS : Non-atomique (race condition possible)
import redis

r = redis.Redis()

# Thread 1 et Thread 2 peuvent exécuter ceci simultanément
current_value = int(r.get('counter') or 0)
new_value = current_value + 1
r.set('counter', new_value)
# ⚠️ Risque de perte de comptage !
```

```python
# ✅ BON : Opération atomique native
r.incr('counter')  # Atomique, thread-safe
```

```python
# ✅ BON : Transaction MULTI/EXEC
pipe = r.pipeline()
pipe.multi()
pipe.incr('counter')
pipe.incr('total_requests')
pipe.execute()  # Les deux opérations sont exécutées atomiquement
```

```python
# ✅ BON : Script Lua (opération complexe atomique)
lua_script = """
local current = redis.call('GET', KEYS[1])
if not current then
    current = 0
else
    current = tonumber(current)
end

if current < tonumber(ARGV[1]) then
    redis.call('INCR', KEYS[1])
    return 1
else
    return 0
end
"""

# Incrémente seulement si < max_value
result = r.eval(lua_script, 1, 'counter', '100')
```

### 2. Time-To-Live (TTL) stratégique

Le TTL est au cœur de nombreux patterns Redis :

```javascript
// Node.js avec ioredis
const Redis = require('ioredis');
const redis = new Redis();

// Pattern 1: TTL dès la création
async function cacheWithTTL(key, value, ttlSeconds) {
    await redis.setex(key, ttlSeconds, JSON.stringify(value));
}

// Pattern 2: TTL conditionnel (refresh sur lecture)
async function getWithRefresh(key, ttlSeconds) {
    const value = await redis.get(key);
    if (value) {
        // Rafraîchir le TTL si accédé (Sliding Window)
        await redis.expire(key, ttlSeconds);
    }
    return value ? JSON.parse(value) : null;
}

// Pattern 3: TTL différencié par priorité
async function cacheByPriority(key, value, priority) {
    const ttlMap = {
        'high': 3600,    // 1 heure
        'medium': 1800,  // 30 minutes
        'low': 900       // 15 minutes
    };
    await redis.setex(key, ttlMap[priority] || 600, JSON.stringify(value));
}
```

### 3. Patterns de nommage

Une convention de nommage cohérente est critique :

```python
# Structure de clé recommandée: {service}:{entity}:{id}:{attribute}

class RedisKeyBuilder:
    @staticmethod
    def user_session(user_id):
        return f"auth:session:{user_id}"

    @staticmethod
    def user_cart(user_id):
        return f"ecommerce:cart:{user_id}"

    @staticmethod
    def api_rate_limit(user_id, endpoint):
        return f"ratelimit:user:{user_id}:{endpoint}"

    @staticmethod
    def cache_query(query_hash):
        return f"cache:query:{query_hash}"

    @staticmethod
    def lock(resource_name):
        return f"lock:{resource_name}"

# Utilisation
session_key = RedisKeyBuilder.user_session("user_12345")
# Résultat: "auth:session:user_12345"

# Avantages:
# 1. Facilite le debugging (redis-cli KEYS auth:session:*)
# 2. Permet des TTL par namespace
# 3. Évite les collisions de clés
# 4. Facilite le monitoring par service
```

### 4. Gestion d'erreurs et résilience

```javascript
// Pattern de connexion robuste avec retry
const Redis = require('ioredis');

const redis = new Redis({
    host: 'localhost',
    port: 6379,
    retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
    },
    maxRetriesPerRequest: 3,
    enableReadyCheck: true,
    enableOfflineQueue: true
});

// Pattern de fallback
async function safeGet(key, defaultValue = null) {
    try {
        const value = await redis.get(key);
        return value ? JSON.parse(value) : defaultValue;
    } catch (error) {
        console.error('Redis error:', error);
        return defaultValue; // Graceful degradation
    }
}

// Pattern de timeout
async function getWithTimeout(key, timeoutMs = 100) {
    return Promise.race([
        redis.get(key),
        new Promise((_, reject) =>
            setTimeout(() => reject(new Error('Timeout')), timeoutMs)
        )
    ]);
}
```

## Métriques et observabilité

### Instrumenter les patterns

```python
import time
from functools import wraps

class RedisMetrics:
    def __init__(self):
        self.hits = 0
        self.misses = 0
        self.errors = 0
        self.total_latency = 0
        self.operations = 0

    def record_hit(self):
        self.hits += 1

    def record_miss(self):
        self.misses += 1

    def record_error(self):
        self.errors += 1

    def record_latency(self, latency_ms):
        self.total_latency += latency_ms
        self.operations += 1

    def get_stats(self):
        hit_rate = (self.hits / (self.hits + self.misses) * 100) if (self.hits + self.misses) > 0 else 0
        avg_latency = (self.total_latency / self.operations) if self.operations > 0 else 0

        return {
            'hit_rate': f"{hit_rate:.2f}%",
            'total_hits': self.hits,
            'total_misses': self.misses,
            'total_errors': self.errors,
            'avg_latency_ms': f"{avg_latency:.2f}",
            'total_operations': self.operations
        }

metrics = RedisMetrics()

def measure_redis_operation(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        try:
            result = func(*args, **kwargs)
            latency_ms = (time.time() - start_time) * 1000
            metrics.record_latency(latency_ms)

            # Détecter hit/miss selon le contexte
            if result is None:
                metrics.record_miss()
            else:
                metrics.record_hit()

            return result
        except Exception as e:
            metrics.record_error()
            raise e
    return wrapper

@measure_redis_operation
def get_from_cache(key):
    return r.get(key)

# Utilisation
value = get_from_cache('user:123')
print(metrics.get_stats())
# {'hit_rate': '75.00%', 'total_hits': 3, 'total_misses': 1, ...}
```

## Architecture de référence

### Application type avec patterns Redis

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT APPLICATIONS                    │
│         (Mobile Apps, Web Frontend, API Consumers)          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY / LB                       │
│                    (Rate Limiting Layer)                    │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Service A  │   │   Service B  │   │   Service C  │
│              │   │              │   │              │
│ • Sessions   │   │ • Caching    │   │ • Job Queue  │
│ • User State │   │ • Query Cache│   │ • Tasks      │
└──────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ↓
        ┌─────────────────────────────────────┐
        │           REDIS LAYER               │
        ├─────────────────────────────────────┤
        │                                     │
        │  ┌─────────────────────────────┐    │
        │  │   Master Node               │    │
        │  │   • All Write Operations    │    │
        │  │   • High Availability       │    │
        │  └─────────────────────────────┘    │
        │          │           │              │
        │          ↓           ↓              │
        │  ┌───────────┐ ┌───────────┐        │
        │  │ Replica 1 │ │ Replica 2 │        │
        │  │ (Read)    │ │ (Read)    │        │
        │  └───────────┘ └───────────┘        │
        │                                     │
        └─────────────────────────────────────┘
                            │
                            ↓
        ┌───────────────────────────────────────┐
        │      PRIMARY DATA STORES              │
        │   (PostgreSQL, MongoDB, etc.)         │
        └───────────────────────────────────────┘
```

## Patterns Coverage Matrix

Ce module couvre les patterns suivants avec leur complexité et cas d'usage :

| Pattern | Complexité | Structures Redis | Cas d'usage principal |
|---------|-----------|------------------|----------------------|
| **Cache-Aside** | 🟢 Facile | Strings | API responses, DB queries |
| **Write-Through** | 🟡 Moyen | Strings, Hashes | Strong consistency |
| **Write-Back** | 🔴 Difficile | Lists, Lua | Write-heavy workloads |
| **Anti-Avalanche** | 🟡 Moyen | Strings, TTL | Cache warming, fallback |
| **Anti-Stampede** | 🔴 Difficile | Locks, Lua | Cache regeneration |
| **Anti-Penetration** | 🟡 Moyen | Bloom Filter | Invalid queries |
| **Pipelining** | 🟢 Facile | All | Batch operations |
| **Client-Side Caching** | 🟡 Moyen | RESP3 Protocol | Read-heavy apps |
| **Distributed Lock** | 🔴 Difficile | Strings, Lua | Resource coordination |
| **Rate Limiting** | 🟡 Moyen | Strings, Sorted Sets | API protection |
| **Session Store** | 🟢 Facile | Hashes, Strings | User authentication |
| **Job Queue** | 🟡 Moyen | Lists, Streams | Background processing |

## Exemple complet : Architecture multi-patterns

Voici une architecture réaliste combinant plusieurs patterns :

```python
import redis
import hashlib
import json
import time
from datetime import datetime, timedelta

class AdvancedRedisApp:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.metrics = RedisMetrics()

    # Pattern 1: Cache-Aside avec Anti-Penetration
    def get_user(self, user_id):
        """Récupère un utilisateur avec cache et protection"""
        cache_key = f"user:{user_id}"
        bloom_key = "bloom:valid_users"

        # 1. Vérifier le Bloom Filter (Anti-Penetration)
        if not self.redis.bf.exists(bloom_key, user_id):
            self.metrics.record_miss()
            return None  # User n'existe certainement pas

        # 2. Cache-Aside
        cached = self.redis.get(cache_key)
        if cached:
            self.metrics.record_hit()
            return json.loads(cached)

        # 3. Fallback vers DB (simulé)
        user_data = self._fetch_from_db(user_id)

        if user_data:
            # Cacher pour 1 heure
            self.redis.setex(cache_key, 3600, json.dumps(user_data))
            self.metrics.record_miss()

        return user_data

    # Pattern 2: Rate Limiting (Sliding Window)
    def check_rate_limit(self, user_id, limit=100, window=60):
        """Rate limit avec fenêtre glissante"""
        key = f"ratelimit:{user_id}"
        now = time.time()

        # Pipeline pour atomicité
        pipe = self.redis.pipeline()

        # Supprimer les entrées expirées
        pipe.zremrangebyscore(key, 0, now - window)

        # Compter les requêtes dans la fenêtre
        pipe.zcard(key)

        # Ajouter la nouvelle requête
        pipe.zadd(key, {str(now): now})

        # Expiration de la clé
        pipe.expire(key, window)

        results = pipe.execute()
        request_count = results[1]

        return request_count < limit

    # Pattern 3: Distributed Lock
    def acquire_lock(self, resource, ttl=10):
        """Acquiert un lock distribué"""
        lock_key = f"lock:{resource}"
        identifier = str(time.time())

        # SET NX EX : atomique
        acquired = self.redis.set(
            lock_key,
            identifier,
            nx=True,  # Only if not exists
            ex=ttl    # Expiration
        )

        return identifier if acquired else None

    def release_lock(self, resource, identifier):
        """Libère un lock de manière sûre"""
        lock_key = f"lock:{resource}"

        # Script Lua pour garantir qu'on libère notre propre lock
        lua_script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """

        return self.redis.eval(lua_script, 1, lock_key, identifier)

    # Pattern 4: Session Store
    def create_session(self, user_id, session_data, ttl=3600):
        """Crée une session utilisateur"""
        session_id = hashlib.sha256(
            f"{user_id}:{time.time()}".encode()
        ).hexdigest()

        session_key = f"session:{session_id}"

        # Stocker avec TTL automatique
        session = {
            'user_id': user_id,
            'created_at': datetime.now().isoformat(),
            **session_data
        }

        self.redis.setex(
            session_key,
            ttl,
            json.dumps(session)
        )

        return session_id

    # Pattern 5: Job Queue
    def enqueue_job(self, queue_name, job_data, priority=0):
        """Ajoute un job à la queue avec priorité"""
        job_id = hashlib.md5(
            f"{queue_name}:{time.time()}".encode()
        ).hexdigest()

        job = {
            'id': job_id,
            'data': job_data,
            'enqueued_at': time.time()
        }

        # Sorted Set pour priorité
        queue_key = f"queue:{queue_name}"
        self.redis.zadd(queue_key, {json.dumps(job): priority})

        return job_id

    def dequeue_job(self, queue_name, count=1):
        """Récupère des jobs par ordre de priorité"""
        queue_key = f"queue:{queue_name}"

        # ZPOPMIN : atomique, retourne le score le plus bas
        jobs = self.redis.zpopmin(queue_key, count)

        return [json.loads(job[0]) for job in jobs]

    def _fetch_from_db(self, user_id):
        """Simule une requête DB"""
        time.sleep(0.05)  # Simule latence DB
        return {'id': user_id, 'name': f'User {user_id}'}

# Utilisation
if __name__ == '__main__':
    r = redis.Redis(decode_responses=True)
    app = AdvancedRedisApp(r)

    # Test des patterns
    print("1. Cache-Aside:")
    user = app.get_user(123)
    print(f"   User: {user}")

    print("\n2. Rate Limiting:")
    for i in range(5):
        allowed = app.check_rate_limit('user_456', limit=3, window=10)
        print(f"   Request {i+1}: {'✓ Allowed' if allowed else '✗ Blocked'}")

    print("\n3. Distributed Lock:")
    lock_id = app.acquire_lock('critical_resource')
    if lock_id:
        print(f"   Lock acquired: {lock_id}")
        time.sleep(1)
        app.release_lock('critical_resource', lock_id)
        print("   Lock released")

    print("\n4. Session Management:")
    session_id = app.create_session('user_789', {'role': 'admin'})
    print(f"   Session created: {session_id}")

    print("\n5. Job Queue:")
    job_id = app.enqueue_job('email', {'to': 'user@example.com'}, priority=1)
    print(f"   Job enqueued: {job_id}")
    jobs = app.dequeue_job('email')
    print(f"   Jobs dequeued: {jobs}")
```

## Diagramme de décision : Quel pattern utiliser ?

```
                    START: Besoin Redis
                            │
                            ↓
                    ┌───────────────┐
                    │ Type de besoin│
                    └───────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
    [Données]          [Contrôle]         [Communication]
        │                   │                   │
        ↓                   ↓                   ↓
    ┌────────┐        ┌─────────┐        ┌──────────┐
    │Caching?│        │Locking? │        │Messaging?│
    └────────┘        └─────────┘        └──────────┘
        │                   │                   │
        ↓                   ↓                   ↓
    [Read-Heavy?]     [Multi-Node?]      [Real-time?]
        │                   │                   │
        ↓                   ↓                   ↓
    Yes → Cache-Aside  Yes → Redlock      Yes → Pub/Sub
    No → Write-Through No → Local Lock    No → Job Queue


    ┌──────────────────────────────────────────┐
    │     Questions à se poser :               │
    ├──────────────────────────────────────────┤
    │                                          │
    │ 1. Les données sont-elles lues souvent?  │
    │    → Pattern de Caching                  │
    │                                          │
    │ 2. Y a-t-il un risque de surcharge?      │
    │    → Pattern Anti-Problèmes              │
    │                                          │
    │ 3. Besoin de coordination distribuée?    │
    │    → Distributed Locking                 │
    │                                          │
    │ 4. Limiter les accès par utilisateur?    │
    │    → Rate Limiting                       │
    │                                          │
    │ 5. Traitement asynchrone nécessaire?     │
    │    → Job Queue Pattern                   │
    │                                          │
    └──────────────────────────────────────────┘
```

## Prérequis pour ce module

Avant d'aborder les patterns avancés, assurez-vous de maîtriser :

### Connaissances techniques
- ✅ Structures de données Redis natives (Strings, Hashes, Lists, Sets, Sorted Sets)
- ✅ Commandes Redis de base (GET, SET, INCR, EXPIRE, etc.)
- ✅ Concepts de TTL et expiration
- ✅ Transactions MULTI/EXEC
- ✅ Scripting Lua basique

### Connaissances architecturales
- ✅ Systèmes distribués et leurs challenges
- ✅ Race conditions et atomicité
- ✅ Patterns de cache traditionnels
- ✅ Concepts de haute disponibilité

### Outils
- ✅ Client Redis pour Python (`redis-py`) ou Node.js (`ioredis`)
- ✅ Redis CLI pour les tests
- ✅ Environnement de développement local avec Redis

## Structure du module

Les sections suivantes approfondissent chaque famille de patterns :

1. **Caching Patterns** : Stratégies pour optimiser les accès aux données
2. **Resilience Patterns** : Protection contre les défaillances en cascade
3. **Performance Patterns** : Optimisation des performances et du débit
4. **Synchronization Patterns** : Coordination dans les systèmes distribués
5. **Rate Limiting Patterns** : Protection et fair-use des ressources
6. **State Management** : Gestion d'état distribué
7. **Messaging Patterns** : Communication asynchrone entre services

Chaque section présente :
- 📊 Les cas d'usage concrets
- 💻 Implémentations complètes en Python et Node.js
- 🎯 Les pièges à éviter
- ⚡ Les optimisations possibles
- 📈 Les métriques à surveiller

## Prochaine section

La section suivante explore en détail les **Caching Patterns**, le fondement de l'utilisation de Redis dans la plupart des applications modernes.

---


⏭️ [Caching Patterns : Cache-Aside, Write-Through, Write-Back](/06-patterns-developpement-avances/01-caching-patterns.md)
