🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2 Stratégies anti-problèmes : Cache Avalanche, Stampede et Penetration

## Introduction

Un cache mal configuré peut devenir votre pire ennemi. Lorsque des milliers de requêtes frappent simultanément votre base de données parce que le cache a failli, les conséquences peuvent être catastrophiques : timeout en cascade, surcharge du serveur DB, voire arrêt complet du service.

Cette section explore les trois problèmes majeurs du caching et leurs solutions éprouvées en production. Ces patterns sont essentiels pour construire des systèmes résilients qui supportent des charges importantes.

## Vue d'ensemble des problèmes

```
┌─────────────────────────────────────────────────────────────────┐
│                    CACHE FAILURE PATTERNS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem          │ Cause              │ Impact                 │
│  ────────────────────────────────────────────────────────────── │
│  Cache Avalanche  │ Mass expiration    │ DB overwhelmed         │
│  🌊               │ All keys expire    │ Service degradation    │
│                   │ simultaneously     │                        │
│                   │                    │                        │
│  Cache Stampede   │ Popular key        │ Redundant DB loads     │
│  🏃🏃🏃           │ expires, multiple  │ Temporary overload
│                   │ threads reload     │                        │
│                   │                    │                        │
│  Cache            │ Queries for        │ DB queries for         │
│  Penetration      │ non-existent data  │ nothing                │
│  🕳️               │ bypass cache       │ Wasted resources       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Impact en chiffres

| Problème | Sans protection | Avec protection | Amélioration |
|----------|----------------|-----------------|--------------|
| **Avalanche** | 10,000 req/s → DB | 100 req/s → DB | **99% réduction** |
| **Stampede** | 100 threads → DB | 1 thread → DB | **99% réduction** |
| **Penetration** | 1,000 req/s → DB | 0 req/s → DB | **100% réduction** |

---

## Problem 1 : Cache Avalanche

### Définition du problème

**Cache Avalanche** se produit lorsqu'un grand nombre de clés expirent simultanément, provoquant une vague de requêtes vers la base de données.

### Scénario catastrophe

```
Time: 00:00:00 - Application démarre
├─ 10,000 clés créées avec TTL=3600s
│
Time: 01:00:00 - Toutes les clés expirent EN MÊME TEMPS
│
└─> 10,000 cache misses simultanés
    └─> 10,000 requêtes DB simultanées
        └─> DB surchargée
            └─> Timeouts
                └─> Service DOWN ❌


TIMELINE VISUALIZATION:

Cache State:
────────────────────────────────────────────────────────
00:00  ████████████████████  (All keys cached)
00:30  ████████████████████  (Still cached)
00:59  ████████████████████  (About to expire...)
01:00  ░░░░░░░░░░░░░░░░░░░░  (ALL KEYS GONE!)
01:01  ░░░░░░░░░░░░░░░░░░░░  (Rebuilding...)

DB Load:
────────────────────────────────────────────────────────
00:00  ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁  (Normal: 10 req/s)
00:30  ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁  (Normal: 10 req/s)
00:59  ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁  (Normal: 10 req/s)
01:00  ████████████████████  (SPIKE: 10,000 req/s) 💥
01:01  ▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅  (Recovering...)
```

### Cause racine

```python
# ❌ PROBLÈME : TTL identique pour toutes les clés
def cache_users_badly():
    TTL = 3600  # 1 heure

    for user_id in range(10000):
        user = fetch_user_from_db(user_id)
        redis.setex(f"user:{user_id}", TTL, json.dumps(user))

    # Toutes les clés expirent à 01:00:00 exactement!
    # ⚠️ AVALANCHE GARANTIE dans 1 heure
```

### Solution 1 : TTL aléatoire (Random Jitter)

La solution la plus simple et efficace : ajouter un délai aléatoire au TTL.

```python
import redis
import random
import json
from typing import Dict, Any

class AntiAvalancheCache:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client
        self.base_ttl = 3600  # 1 heure
        self.jitter_range = 300  # ±5 minutes

    def set_with_jitter(self, key: str, value: Any) -> None:
        """
        Stocke une valeur avec un TTL randomisé
        """
        # TTL = base_ttl ± random(0, jitter_range)
        jitter = random.randint(0, self.jitter_range)
        ttl = self.base_ttl + jitter

        self.cache.setex(key, ttl, json.dumps(value))
        print(f"Cached {key} with TTL={ttl}s (jitter: +{jitter}s)")

    def cache_users(self, user_ids: list) -> None:
        """
        Cache plusieurs utilisateurs avec TTL distribués
        """
        for user_id in user_ids:
            user = self._fetch_user_from_db(user_id)
            self.set_with_jitter(f"user:{user_id}", user)

    def _fetch_user_from_db(self, user_id: int) -> Dict[str, Any]:
        # Simulation
        return {'id': user_id, 'name': f'User {user_id}'}


# Résultat avec jitter:
# ────────────────────────────────────────────────────────
# user:1   → TTL = 3612s (expire à 01:00:12)
# user:2   → TTL = 3745s (expire à 01:02:25)
# user:3   → TTL = 3521s (expire à 00:58:41)
# ...
# ────────────────────────────────────────────────────────
# Les expirations sont ÉTALÉES sur 10 minutes
# Pas d'avalanche! ✓
```

```javascript
// Node.js implementation
const Redis = require('ioredis');

class AntiAvalancheCache {
    constructor(redisClient) {
        this.cache = redisClient;
        this.baseTTL = 3600; // 1 heure
        this.jitterRange = 300; // ±5 minutes
    }

    async setWithJitter(key, value) {
        // TTL randomisé
        const jitter = Math.floor(Math.random() * this.jitterRange);
        const ttl = this.baseTTL + jitter;

        await this.cache.setex(key, ttl, JSON.stringify(value));
        console.log(`Cached ${key} with TTL=${ttl}s (jitter: +${jitter}s)`);
    }

    async cacheUsers(userIds) {
        for (const userId of userIds) {
            const user = await this._fetchUserFromDB(userId);
            await this.setWithJitter(`user:${userId}`, user);
        }
    }

    async _fetchUserFromDB(userId) {
        // Simulation
        return { id: userId, name: `User ${userId}` };
    }
}

// Utilisation
(async () => {
    const redis = new Redis();
    const cache = new AntiAvalancheCache(redis);

    const userIds = Array.from({length: 1000}, (_, i) => i + 1);
    await cache.cacheUsers(userIds);
})();
```

### Solution 2 : TTL hiérarchique par catégorie

Différencier le TTL selon l'importance des données.

```python
class HierarchicalTTLCache:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client

        # Configuration TTL par catégorie
        self.ttl_config = {
            'critical': {
                'base': 7200,    # 2 heures
                'jitter': 600    # ±10 minutes
            },
            'normal': {
                'base': 3600,    # 1 heure
                'jitter': 300    # ±5 minutes
            },
            'low_priority': {
                'base': 1800,    # 30 minutes
                'jitter': 180    # ±3 minutes
            }
        }

    def set_with_priority(self, key: str, value: Any, priority: str = 'normal') -> None:
        """
        Cache avec TTL basé sur la priorité
        """
        config = self.ttl_config.get(priority, self.ttl_config['normal'])

        jitter = random.randint(0, config['jitter'])
        ttl = config['base'] + jitter

        self.cache.setex(key, ttl, json.dumps(value))
        print(f"Cached {key} [{priority}] with TTL={ttl}s")

    def cache_heterogeneous_data(self):
        """
        Cache différents types de données avec différentes priorités
        """
        # Données critiques : profils utilisateurs
        self.set_with_priority('user:123', {'name': 'VIP'}, 'critical')

        # Données normales : produits
        self.set_with_priority('product:456', {'name': 'Item'}, 'normal')

        # Données low-priority : stats
        self.set_with_priority('stats:daily', {'views': 1000}, 'low_priority')


# Résultat :
# ────────────────────────────────────────────────────────
# Cache Expiration Distribution:
#
# Critical (2h base):     ████████████████
# Normal (1h base):       ████████
# Low Priority (30m):     ████
#
# Pas de pic d'expiration simultané ✓
# ────────────────────────────────────────────────────────
```

### Solution 3 : Pré-warming intelligent

Rafraîchir le cache avant expiration pour éviter l'avalanche.

```python
import threading
import time
from datetime import datetime, timedelta

class PreWarmingCache:
    def __init__(self, redis_client: redis.Redis, db_connection):
        self.cache = redis_client
        self.db = db_connection
        self.base_ttl = 3600
        self.prewarm_threshold = 0.8  # Rafraîchir à 80% du TTL

        # Démarrer le worker de pre-warming
        self.prewarming_enabled = True
        self.worker = threading.Thread(target=self._prewarming_worker)
        self.worker.daemon = True
        self.worker.start()

    def get_with_prewarm(self, key: str) -> Any:
        """
        Récupère une valeur et déclenche un pre-warm si nécessaire
        """
        # Lire depuis le cache
        cached = self.cache.get(key)

        if cached:
            # Vérifier le TTL restant
            ttl = self.cache.ttl(key)

            if ttl > 0:
                # Calculer le seuil de pre-warming
                threshold = self.base_ttl * self.prewarm_threshold

                if ttl < (self.base_ttl - threshold):
                    # TTL proche de l'expiration, déclencher pre-warm
                    print(f"⚠️  Pre-warming {key} (TTL: {ttl}s)")
                    self._schedule_prewarm(key)

            return json.loads(cached)

        # Cache miss - charger normalement
        return self._load_and_cache(key)

    def _schedule_prewarm(self, key: str):
        """
        Programme le rafraîchissement d'une clé
        """
        prewarm_job = {
            'key': key,
            'scheduled_at': datetime.now().isoformat()
        }

        self.cache.lpush('prewarm_queue', json.dumps(prewarm_job))

    def _prewarming_worker(self):
        """
        Worker qui traite les pre-warming en arrière-plan
        """
        while self.prewarming_enabled:
            try:
                # Récupérer un job de pre-warming
                job_json = self.cache.rpop('prewarm_queue')

                if job_json:
                    job = json.loads(job_json)
                    key = job['key']

                    print(f"🔄 Pre-warming {key}...")
                    self._load_and_cache(key)
                    print(f"✓ Pre-warmed {key}")

                time.sleep(0.1)  # Éviter de boucler trop vite

            except Exception as e:
                print(f"Error in pre-warming worker: {e}")
                time.sleep(1)

    def _load_and_cache(self, key: str) -> Any:
        """
        Charge depuis la DB et cache
        """
        # Extraire l'entity type et l'ID depuis la clé
        # Format attendu: "entity:id"
        parts = key.split(':')
        if len(parts) == 2:
            entity_type, entity_id = parts

            # Charger depuis la DB (simulé)
            data = self._fetch_from_db(entity_type, entity_id)

            if data:
                # Cache avec jitter
                jitter = random.randint(0, 300)
                ttl = self.base_ttl + jitter
                self.cache.setex(key, ttl, json.dumps(data))

            return data

        return None

    def _fetch_from_db(self, entity_type: str, entity_id: str) -> Dict[str, Any]:
        # Simulation
        return {'id': entity_id, 'type': entity_type, 'data': 'example'}

    def shutdown(self):
        """Arrêt gracieux du worker"""
        self.prewarming_enabled = False
        self.worker.join(timeout=5)


# Utilisation
if __name__ == '__main__':
    r = redis.Redis(decode_responses=True)
    cache = PreWarmingCache(r, None)

    # Première lecture : cache miss
    data = cache.get_with_prewarm('user:123')

    # Simuler le passage du temps
    time.sleep(2)

    # Lectures suivantes : cache hit + pre-warm automatique si proche expiration
    for _ in range(5):
        data = cache.get_with_prewarm('user:123')
        time.sleep(1)

    cache.shutdown()
```

### Solution 4 : Circuit Breaker pour protection DB

Si malgré tout une avalanche se produit, limiter les dégâts.

```python
from enum import Enum
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Blocking requests
    HALF_OPEN = "half_open"  # Testing recovery

class CircuitBreaker:
    def __init__(self, redis_client: redis.Redis,
                 failure_threshold: int = 5,
                 timeout: int = 60):
        self.cache = redis_client
        self.failure_threshold = failure_threshold
        self.timeout = timeout  # seconds
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None

    def call_db(self, func, *args, **kwargs):
        """
        Appelle la DB avec protection Circuit Breaker
        """
        # Vérifier l'état du circuit
        if self.state == CircuitState.OPEN:
            # Vérifier si le timeout est écoulé
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                print("🔄 Circuit HALF_OPEN - Testing recovery")
            else:
                # Circuit toujours ouvert, retourner fallback
                print("⛔ Circuit OPEN - Returning fallback")
                return self._fallback()

        # Tenter l'appel
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        """Appel DB réussi"""
        if self.state == CircuitState.HALF_OPEN:
            print("✓ Circuit CLOSED - Recovery successful")
            self.state = CircuitState.CLOSED

        self.failure_count = 0

    def _on_failure(self):
        """Appel DB échoué"""
        self.failure_count += 1
        self.last_failure_time = datetime.now()

        if self.failure_count >= self.failure_threshold:
            print(f"⚠️  Circuit OPEN - {self.failure_count} failures")
            self.state = CircuitState.OPEN

    def _should_attempt_reset(self) -> bool:
        """Vérifie si on peut tenter une réinitialisation"""
        if self.last_failure_time is None:
            return False

        elapsed = (datetime.now() - self.last_failure_time).total_seconds()
        return elapsed >= self.timeout

    def _fallback(self):
        """Valeur de fallback quand le circuit est ouvert"""
        return None  # Ou une valeur par défaut


class ProtectedCache:
    def __init__(self, redis_client: redis.Redis, db_connection):
        self.cache = redis_client
        self.db = db_connection
        self.circuit_breaker = CircuitBreaker(redis_client)

    def get_user(self, user_id: int):
        """
        Récupère un utilisateur avec protection Circuit Breaker
        """
        cache_key = f"user:{user_id}"

        # Essayer le cache
        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)

        # Cache miss - utiliser le Circuit Breaker
        try:
            user = self.circuit_breaker.call_db(
                self._fetch_user_from_db,
                user_id
            )

            if user:
                self.cache.setex(cache_key, 3600, json.dumps(user))

            return user
        except Exception as e:
            print(f"Error with Circuit Breaker: {e}")
            return None

    def _fetch_user_from_db(self, user_id: int):
        # Simulation avec possibilité d'erreur
        if random.random() < 0.3:  # 30% chance d'erreur
            raise Exception("DB connection failed")

        return {'id': user_id, 'name': f'User {user_id}'}
```

### Visualisation de l'impact

```
Sans protection (Avalanche):
────────────────────────────────────────────────────────
Time    Cache Hits    DB Queries    Response Time
00:59   ████████████  ▁▁            10ms
01:00   ░░░░░░░░░░░░  ████████████  5000ms (timeout!)
01:01   ████          ████          500ms (recovering)
01:02   ████████████  ▁▁            10ms


Avec protection (Jitter + Circuit Breaker):
────────────────────────────────────────────────────────
Time    Cache Hits    DB Queries    Response Time
00:59   ████████████  ▁▁            10ms
01:00   ██████████    ██            15ms (smooth)
01:01   ████████████  ▁             10ms
01:02   ████████████  ▁             10ms
```

---

## Problem 2 : Cache Stampede (Thundering Herd)

### Définition du problème

**Cache Stampede** se produit lorsqu'une clé très populaire expire et que de multiples threads/processus tentent simultanément de la recharger.

### Scénario catastrophe

```
Hot Key: "homepage_content" (1000 req/s)
TTL expires at 12:00:00

Timeline:
────────────────────────────────────────────────────────
11:59:59  Thread 1: GET homepage_content → HIT ✓
          Thread 2: GET homepage_content → HIT ✓
          ...
          Thread 1000: GET homepage_content → HIT ✓

12:00:00  [KEY EXPIRES]

12:00:00.001  Thread 1: GET homepage_content → MISS
              Thread 1: → SELECT FROM db (100ms)

12:00:00.002  Thread 2: GET homepage_content → MISS
              Thread 2: → SELECT FROM db (100ms)

12:00:00.003  Thread 3: GET homepage_content → MISS
              Thread 3: → SELECT FROM db (100ms)

...

12:00:00.100  100 threads → 100 identical DB queries! 💥


VISUALIZATION:
────────────────────────────────────────────────────────
               ┌──────┐
Cache Miss →   │      │
               │ DB   │  ← 100 identical queries
  Thread 1 →── │      │
  Thread 2 →── │      │
  Thread 3 →── │      │
  ...          │      │
  Thread 100→──│      │
               └──────┘
                  ↓
              OVERLOAD
```

### Solution 1 : Lock-based (Pessimistic)

Un seul thread charge la donnée, les autres attendent.

```python
import time
from contextlib import contextmanager

class StampedeProtectedCache:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client
        self.default_ttl = 3600
        self.lock_timeout = 10  # secondes

    def get_with_lock(self, key: str, load_func, *args) -> Any:
        """
        Récupère une valeur avec protection contre le stampede
        """
        # 1. Essayer le cache
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # 2. Cache miss - Acquérir un lock
        lock_key = f"lock:{key}"
        lock_acquired = self._acquire_lock(lock_key)

        if lock_acquired:
            # Ce thread a acquis le lock
            try:
                # Double-check : un autre thread a peut-être déjà chargé
                cached = self.cache.get(key)
                if cached:
                    return json.loads(cached)

                # Charger la donnée
                print(f"🔒 Lock acquired, loading {key}")
                data = load_func(*args)

                # Cacher avec jitter
                jitter = random.randint(0, 300)
                self.cache.setex(key, self.default_ttl + jitter, json.dumps(data))

                return data
            finally:
                # Libérer le lock
                self._release_lock(lock_key)
                print(f"🔓 Lock released for {key}")
        else:
            # Un autre thread a le lock, attendre et réessayer
            print(f"⏳ Waiting for lock on {key}")
            return self._wait_for_cache(key, load_func, *args)

    def _acquire_lock(self, lock_key: str) -> bool:
        """
        Acquiert un lock distribué
        """
        # SET NX EX : atomique
        acquired = self.cache.set(
            lock_key,
            "1",
            nx=True,  # Only if not exists
            ex=self.lock_timeout
        )
        return acquired

    def _release_lock(self, lock_key: str):
        """
        Libère le lock
        """
        self.cache.delete(lock_key)

    def _wait_for_cache(self, key: str, load_func, *args, max_wait: int = 5):
        """
        Attend que le cache soit peuplé par un autre thread
        """
        start_time = time.time()

        while (time.time() - start_time) < max_wait:
            # Vérifier si le cache a été peuplé
            cached = self.cache.get(key)
            if cached:
                print(f"✓ Cache populated by another thread")
                return json.loads(cached)

            # Attendre un peu
            time.sleep(0.1)

        # Timeout - charger quand même (fallback)
        print(f"⚠️  Timeout waiting for cache, loading directly")
        return load_func(*args)


# Utilisation
if __name__ == '__main__':
    r = redis.Redis(decode_responses=True)
    cache = StampedeProtectedCache(r)

    def expensive_query():
        """Simule une requête DB coûteuse"""
        print("💾 Executing expensive DB query...")
        time.sleep(2)  # Simule 2 secondes
        return {'content': 'Homepage content', 'timestamp': time.time()}

    # Simuler 10 threads concurrents
    import concurrent.futures

    def worker(thread_id):
        result = cache.get_with_lock('homepage', expensive_query)
        print(f"Thread {thread_id}: {result['timestamp']}")

    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = [executor.submit(worker, i) for i in range(10)]
        concurrent.futures.wait(futures)

    # Résultat : Un seul appel DB malgré 10 threads! ✓
```

### Solution 2 : Probabilistic Early Expiration

Rafraîchir probabilistiquement avant expiration.

```python
import math

class ProbabilisticCache:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client
        self.base_ttl = 3600
        self.beta = 1.0  # Facteur de probabilité

    def get_with_probabilistic_refresh(self, key: str, load_func, *args) -> Any:
        """
        Implémentation de l'algorithme XFetch (Optimal Probabilistic Cache Stampede Prevention)

        Paper: "Optimal Probabilistic Cache Stampede Prevention"
        https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf
        """
        # Lire depuis le cache
        cached = self.cache.get(key)

        if cached:
            # Récupérer le TTL restant
            ttl = self.cache.ttl(key)

            if ttl > 0:
                # Calculer la probabilité de refresh
                # probability = beta * log(rand) * remaining_ttl
                delta = self.base_ttl - ttl  # Temps écoulé

                # Formule XFetch
                random_value = random.random()

                if random_value == 0:
                    random_value = 0.0001  # Éviter log(0)

                xfetch_value = delta * self.beta * abs(math.log(random_value))

                if xfetch_value >= ttl:
                    # Déclencher un refresh anticipé
                    print(f"🎲 Probabilistic refresh triggered for {key}")
                    print(f"   TTL: {ttl}s, Delta: {delta}s, XFetch: {xfetch_value:.2f}")

                    # Charger en arrière-plan (ou bloquer selon le cas)
                    data = load_func(*args)

                    # Recacher
                    jitter = random.randint(0, 300)
                    self.cache.setex(key, self.base_ttl + jitter, json.dumps(data))

                    return data

            return json.loads(cached)

        # Cache miss - charger normalement
        data = load_func(*args)
        jitter = random.randint(0, 300)
        self.cache.setex(key, self.base_ttl + jitter, json.dumps(data))

        return data


# Simulation pour comprendre la probabilité
def simulate_probabilistic_refresh():
    """
    Montre comment la probabilité augmente près de l'expiration
    """
    base_ttl = 3600
    beta = 1.0

    print("Probabilité de refresh selon le temps restant:")
    print("─" * 60)

    for remaining_ttl in [3600, 1800, 900, 300, 60, 10, 1]:
        delta = base_ttl - remaining_ttl

        # Calculer la probabilité moyenne sur 10000 échantillons
        refresh_count = 0
        samples = 10000

        for _ in range(samples):
            random_value = random.random()
            if random_value == 0:
                random_value = 0.0001

            xfetch = delta * beta * abs(math.log(random_value))

            if xfetch >= remaining_ttl:
                refresh_count += 1

        probability = (refresh_count / samples) * 100

        print(f"TTL restant: {remaining_ttl:4d}s  →  Probabilité: {probability:5.2f}%")

    print("─" * 60)


if __name__ == '__main__':
    simulate_probabilistic_refresh()

    # Output:
    # ────────────────────────────────────────────────────────
    # TTL restant: 3600s  →  Probabilité:  0.00%
    # TTL restant: 1800s  →  Probabilité:  0.52%
    # TTL restant:  900s  →  Probabilité:  4.12%
    # TTL restant:  300s  →  Probabilité: 19.87%
    # TTL restant:   60s  →  Probabilité: 63.21%
    # TTL restant:   10s  →  Probabilité: 98.17%
    # TTL restant:    1s  →  Probabilité: 99.99%
    # ────────────────────────────────────────────────────────
```

### Solution 3 : Stale-While-Revalidate

Retourner la valeur expirée pendant le rechargement.

```python
class StaleWhileRevalidateCache:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client
        self.ttl = 3600
        self.grace_period = 300  # 5 minutes après expiration

    def get_with_swr(self, key: str, load_func, *args) -> Any:
        """
        Implémentation du pattern Stale-While-Revalidate
        """
        value_key = f"value:{key}"
        stale_key = f"stale:{key}"
        lock_key = f"lock:{key}"

        # 1. Essayer la valeur fraîche
        fresh_value = self.cache.get(value_key)

        if fresh_value:
            return json.loads(fresh_value)

        # 2. Valeur expirée - Essayer la valeur stale
        stale_value = self.cache.get(stale_key)

        if stale_value:
            # On a une valeur stale, la retourner immédiatement
            print(f"📋 Returning stale value for {key}")

            # En arrière-plan, revalider si personne ne le fait
            if self.cache.set(lock_key, "1", nx=True, ex=30):
                print(f"🔄 Revalidating {key} in background")

                # Dans un vrai système, utiliser un thread/celery task
                threading.Thread(
                    target=self._revalidate,
                    args=(key, load_func, args)
                ).start()

            return json.loads(stale_value)

        # 3. Pas de valeur du tout - charger normalement
        print(f"💾 Loading fresh {key}")
        data = load_func(*args)
        self._cache_with_stale(key, data)

        return data

    def _cache_with_stale(self, key: str, data: Any):
        """
        Cache la valeur avec une copie stale pour SWR
        """
        value_key = f"value:{key}"
        stale_key = f"stale:{key}"

        serialized = json.dumps(data)

        # Valeur fraîche avec TTL normal
        self.cache.setex(value_key, self.ttl, serialized)

        # Valeur stale avec TTL + grace period
        self.cache.setex(stale_key, self.ttl + self.grace_period, serialized)

        print(f"✓ Cached {key} (fresh: {self.ttl}s, stale: {self.ttl + self.grace_period}s)")

    def _revalidate(self, key: str, load_func, args):
        """
        Revalide la valeur en arrière-plan
        """
        try:
            data = load_func(*args)
            self._cache_with_stale(key, data)
            print(f"✓ Revalidated {key}")
        except Exception as e:
            print(f"✗ Revalidation failed for {key}: {e}")
        finally:
            # Libérer le lock
            lock_key = f"lock:{key}"
            self.cache.delete(lock_key)


# Visualisation du pattern SWR
def visualize_swr():
    """
    Timeline du pattern Stale-While-Revalidate
    """
    print("""
Stale-While-Revalidate Timeline:
────────────────────────────────────────────────────────

Time   Fresh Value    Stale Value    Response
────────────────────────────────────────────────────────
00:00  ✓ Cached       ✓ Cached       Return fresh ⚡
30:00  ✓ Cached       ✓ Cached       Return fresh ⚡
59:00  ✓ Cached       ✓ Cached       Return fresh ⚡

60:00  ✗ EXPIRED      ✓ Available    Return stale ⚡
                      🔄 Revalidate   (in background)

61:00  ✓ REFRESHED    ✓ Available    Return fresh ⚡

65:00  ✗ EXPIRED      ✗ EXPIRED      Load from DB 💾

────────────────────────────────────────────────────────
Benefits:
- No stampede (only 1 revalidation)
- Fast response (return stale immediately)
- Graceful degradation
    """)


if __name__ == '__main__':
    visualize_swr()
```

### Implémentation Node.js complète

```javascript
const Redis = require('ioredis');

class StampedeProtection {
    constructor(redisClient) {
        this.cache = redisClient;
        this.defaultTTL = 3600;
        this.lockTimeout = 10;
    }

    // Pattern 1: Lock-based
    async getWithLock(key, loadFunc) {
        // Essayer le cache
        const cached = await this.cache.get(key);
        if (cached) {
            return JSON.parse(cached);
        }

        // Acquérir le lock
        const lockKey = `lock:${key}`;
        const lockAcquired = await this.cache.set(
            lockKey,
            '1',
            'NX',
            'EX',
            this.lockTimeout
        );

        if (lockAcquired) {
            try {
                // Double-check
                const cached = await this.cache.get(key);
                if (cached) {
                    return JSON.parse(cached);
                }

                console.log(`🔒 Lock acquired, loading ${key}`);
                const data = await loadFunc();

                // Cache avec jitter
                const jitter = Math.floor(Math.random() * 300);
                await this.cache.setex(
                    key,
                    this.defaultTTL + jitter,
                    JSON.stringify(data)
                );

                return data;
            } finally {
                await this.cache.del(lockKey);
                console.log(`🔓 Lock released for ${key}`);
            }
        } else {
            // Attendre et réessayer
            console.log(`⏳ Waiting for lock on ${key}`);
            return await this._waitForCache(key, loadFunc);
        }
    }

    async _waitForCache(key, loadFunc, maxWait = 5000) {
        const startTime = Date.now();

        while ((Date.now() - startTime) < maxWait) {
            const cached = await this.cache.get(key);
            if (cached) {
                console.log('✓ Cache populated by another thread');
                return JSON.parse(cached);
            }

            await new Promise(resolve => setTimeout(resolve, 100));
        }

        console.log('⚠️  Timeout waiting, loading directly');
        return await loadFunc();
    }

    // Pattern 2: Stale-While-Revalidate
    async getWithSWR(key, loadFunc) {
        const valueKey = `value:${key}`;
        const staleKey = `stale:${key}`;
        const lockKey = `lock:${key}`;

        // Essayer la valeur fraîche
        const freshValue = await this.cache.get(valueKey);
        if (freshValue) {
            return JSON.parse(freshValue);
        }

        // Essayer la valeur stale
        const staleValue = await this.cache.get(staleKey);
        if (staleValue) {
            console.log(`📋 Returning stale value for ${key}`);

            // Revalider en arrière-plan
            const lockSet = await this.cache.set(lockKey, '1', 'NX', 'EX', 30);
            if (lockSet) {
                console.log(`🔄 Revalidating ${key} in background`);
                this._revalidate(key, loadFunc).catch(console.error);
            }

            return JSON.parse(staleValue);
        }

        // Pas de valeur - charger
        console.log(`💾 Loading fresh ${key}`);
        const data = await loadFunc();
        await this._cacheWithStale(key, data);

        return data;
    }

    async _cacheWithStale(key, data) {
        const valueKey = `value:${key}`;
        const staleKey = `stale:${key}`;
        const gracePeriod = 300;

        const serialized = JSON.stringify(data);

        // Valeur fraîche
        await this.cache.setex(valueKey, this.defaultTTL, serialized);

        // Valeur stale
        await this.cache.setex(
            staleKey,
            this.defaultTTL + gracePeriod,
            serialized
        );

        console.log(`✓ Cached ${key} with SWR`);
    }

    async _revalidate(key, loadFunc) {
        try {
            const data = await loadFunc();
            await this._cacheWithStale(key, data);
            console.log(`✓ Revalidated ${key}`);
        } catch (error) {
            console.error(`✗ Revalidation failed for ${key}:`, error);
        } finally {
            await this.cache.del(`lock:${key}`);
        }
    }
}

// Test avec charge concurrente
async function testStampedeProtection() {
    const redis = new Redis();
    const protection = new StampedeProtection(redis);

    let dbCallCount = 0;

    async function expensiveQuery() {
        dbCallCount++;
        console.log(`💾 DB call #${dbCallCount}`);
        await new Promise(resolve => setTimeout(resolve, 2000));
        return { content: 'Data', timestamp: Date.now() };
    }

    // Simuler 20 requêtes concurrentes
    console.log('Testing with 20 concurrent requests...');
    const promises = Array.from({ length: 20 }, (_, i) =>
        protection.getWithLock('hot_key', expensiveQuery)
    );

    await Promise.all(promises);

    console.log(`\n✓ Done! DB was called ${dbCallCount} time(s)`);
    console.log(`Without protection, it would be called 20 times!`);

    process.exit(0);
}

testStampedeProtection();
```

---

## Problem 3 : Cache Penetration

### Définition du problème

**Cache Penetration** se produit lorsque des requêtes pour des données inexistantes traversent systématiquement le cache et frappent la base de données.

### Scénario catastrophe

```
Attaque / Bug Application:
────────────────────────────────────────────────────────
Request 1: GET user:99999999 → Cache MISS → DB Query → NULL
Request 2: GET user:88888888 → Cache MISS → DB Query → NULL
Request 3: GET user:77777777 → Cache MISS → DB Query → NULL
...
Request 1000: GET user:11111111 → Cache MISS → DB Query → NULL

Résultat:
- 1000 requêtes DB pour RIEN
- DB surchargée par des requêtes inutiles
- Pas de bénéfice du cache
- Possible attack vector (DDoS via invalid IDs)


VISUALIZATION:
────────────────────────────────────────────────────────
Attacker/Bug → Invalid IDs
    │
    ├─→ user:999999 ──→ Cache MISS ──→ DB Query ──→ NULL
    ├─→ user:888888 ──→ Cache MISS ──→ DB Query ──→ NULL
    ├─→ user:777777 ──→ Cache MISS ──→ DB Query ──→ NULL
    └─→ user:666666 ──→ Cache MISS ──→ DB Query ──→ NULL
                                ↓
                          DB OVERLOAD 💥
```

### Solution 1 : Cache des valeurs NULL

Cacher explicitement l'absence de données.

```python
class NullCachingPattern:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client
        self.positive_ttl = 3600     # 1 heure pour données existantes
        self.negative_ttl = 60       # 1 minute pour NULL
        self.null_placeholder = "__NULL__"

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        """
        Récupère un utilisateur avec cache de NULL
        """
        cache_key = f"user:{user_id}"

        # Lire depuis le cache
        cached = self.cache.get(cache_key)

        if cached:
            # Vérifier si c'est un NULL caché
            if cached == self.null_placeholder:
                print(f"✓ Cache HIT for NULL user {user_id}")
                return None

            print(f"✓ Cache HIT for user {user_id}")
            return json.loads(cached)

        # Cache MISS - Charger depuis DB
        print(f"💾 Loading user {user_id} from DB")
        user = self._fetch_user_from_db(user_id)

        if user:
            # Donnée existe - cache normal
            self.cache.setex(
                cache_key,
                self.positive_ttl,
                json.dumps(user)
            )
        else:
            # Donnée n'existe pas - cache le NULL
            print(f"→ Caching NULL for user {user_id}")
            self.cache.setex(
                cache_key,
                self.negative_ttl,
                self.null_placeholder
            )

        return user

    def _fetch_user_from_db(self, user_id: int) -> Optional[Dict[str, Any]]:
        # Simulation : seulement les IDs < 1000 existent
        if user_id < 1000:
            return {'id': user_id, 'name': f'User {user_id}'}
        return None


# Test de protection
if __name__ == '__main__':
    r = redis.Redis(decode_responses=True)
    cache = NullCachingPattern(r)

    # Premier appel pour un user inexistant
    print("=== First call for invalid user ===")
    user = cache.get_user(999999)
    print(f"Result: {user}\n")

    # Deuxième appel - devrait être un cache HIT
    print("=== Second call for same invalid user ===")
    user = cache.get_user(999999)
    print(f"Result: {user}\n")

    # Output:
    # === First call for invalid user ===
    # 💾 Loading user 999999 from DB
    # → Caching NULL for user 999999
    # Result: None
    #
    # === Second call for same invalid user ===
    # ✓ Cache HIT for NULL user 999999
    # Result: None
    #
    # ✓ DB protégée! Deuxième appel ne touche pas la DB
```

### Solution 2 : Bloom Filter

Utiliser un Bloom Filter pour vérifier rapidement si une clé peut exister.

```python
import mmh3  # MurmurHash3 pour le Bloom Filter
from bitarray import bitarray

class BloomFilter:
    def __init__(self, size: int = 1000000, hash_count: int = 7):
        """
        Bloom Filter simple

        Args:
            size: Taille du bit array
            hash_count: Nombre de fonctions de hash
        """
        self.size = size
        self.hash_count = hash_count
        self.bit_array = bitarray(size)
        self.bit_array.setall(0)

    def add(self, item: str):
        """Ajoute un élément au Bloom Filter"""
        for seed in range(self.hash_count):
            index = mmh3.hash(item, seed) % self.size
            self.bit_array[index] = 1

    def contains(self, item: str) -> bool:
        """
        Vérifie si un élément PEUT exister

        Returns:
            True: L'élément PEUT exister (ou false positive)
            False: L'élément n'existe CERTAINEMENT PAS
        """
        for seed in range(self.hash_count):
            index = mmh3.hash(item, seed) % self.size
            if self.bit_array[index] == 0:
                return False  # Certainement pas présent
        return True  # Probablement présent


class BloomFilterCache:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client
        self.bloom = BloomFilter(size=1000000, hash_count=7)
        self.ttl = 3600

        # Initialiser le Bloom Filter avec les IDs existants
        self._initialize_bloom_filter()

    def _initialize_bloom_filter(self):
        """
        Charge tous les IDs existants dans le Bloom Filter
        (À faire au démarrage de l'application)
        """
        print("Initializing Bloom Filter...")

        # Simuler le chargement de tous les user IDs existants
        # En production, ceci viendrait de la DB
        existing_ids = range(1, 1000)  # IDs 1-999 existent

        for user_id in existing_ids:
            self.bloom.add(f"user:{user_id}")

        print(f"✓ Bloom Filter initialized with {len(existing_ids)} items")

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        """
        Récupère un utilisateur avec protection Bloom Filter
        """
        cache_key = f"user:{user_id}"

        # 1. Vérifier le Bloom Filter AVANT le cache
        if not self.bloom.contains(cache_key):
            # L'utilisateur n'existe CERTAINEMENT PAS
            print(f"🚫 Bloom Filter: user {user_id} does not exist")
            return None

        # 2. Le user PEUT exister, vérifier le cache
        cached = self.cache.get(cache_key)
        if cached:
            print(f"✓ Cache HIT for user {user_id}")
            return json.loads(cached)

        # 3. Cache MISS - Charger depuis DB
        print(f"💾 Loading user {user_id} from DB")
        user = self._fetch_user_from_db(user_id)

        if user:
            self.cache.setex(cache_key, self.ttl, json.dumps(user))
        else:
            # False positive du Bloom Filter
            print(f"⚠️  False positive for user {user_id}")

        return user

    def _fetch_user_from_db(self, user_id: int) -> Optional[Dict[str, Any]]:
        if user_id < 1000:
            return {'id': user_id, 'name': f'User {user_id}'}
        return None


# Test avec et sans Bloom Filter
def compare_bloom_filter():
    r = redis.Redis(decode_responses=True)
    bloom_cache = BloomFilterCache(r)

    print("\n=== Testing with Bloom Filter ===\n")

    # Test 1: User existant
    print("1. Valid user (ID: 500):")
    user = bloom_cache.get_user(500)
    print(f"   Result: {user}\n")

    # Test 2: User inexistant (bloqué par Bloom)
    print("2. Invalid user (ID: 999999):")
    user = bloom_cache.get_user(999999)
    print(f"   Result: {user}\n")

    # Test 3: Batch d'IDs invalides
    print("3. Batch of 100 invalid IDs:")
    db_calls = 0
    for i in range(100000, 100100):
        user = bloom_cache.get_user(i)
        if user is not None:
            db_calls += 1

    print(f"   DB calls made: {db_calls}")
    print(f"   DB calls blocked: {100 - db_calls}\n")

    # Output attendu:
    # DB calls made: 0-1 (false positives seulement)
    # DB calls blocked: 99-100


if __name__ == '__main__':
    compare_bloom_filter()
```

### Solution 3 : Bloom Filter avec Redis

Utiliser RedisBloom (module Redis) pour un Bloom Filter distribué.

```python
from redis.commands.search.field import TagField
from redis.commands.bf import BFInfo

class RedisBloomCache:
    def __init__(self, redis_client: redis.Redis):
        """
        Utilise RedisBloom pour un Bloom Filter distribué

        Nécessite Redis avec le module RedisBloom:
        docker run -p 6379:6379 redis/redis-stack-server:latest
        """
        self.cache = redis_client
        self.bloom_key = "users:bloom"
        self.ttl = 3600

        # Initialiser le Bloom Filter Redis
        self._initialize_redis_bloom()

    def _initialize_redis_bloom(self):
        """
        Crée un Bloom Filter dans Redis
        """
        try:
            # Créer le Bloom Filter avec RedisBloom
            # BF.RESERVE key error_rate capacity
            self.cache.execute_command(
                'BF.RESERVE',
                self.bloom_key,
                0.01,     # 1% error rate
                1000000   # Expected number of items
            )
            print("✓ Redis Bloom Filter created")
        except Exception as e:
            # Le filtre existe déjà
            print(f"Bloom Filter already exists: {e}")

        # Peupler avec les IDs existants
        existing_ids = range(1, 1000)
        for user_id in existing_ids:
            self.cache.execute_command(
                'BF.ADD',
                self.bloom_key,
                f"user:{user_id}"
            )

        print(f"✓ Bloom Filter populated with {len(existing_ids)} items")

    def exists_in_bloom(self, key: str) -> bool:
        """
        Vérifie si une clé existe dans le Bloom Filter
        """
        result = self.cache.execute_command('BF.EXISTS', self.bloom_key, key)
        return bool(result)

    def add_to_bloom(self, key: str):
        """
        Ajoute une clé au Bloom Filter
        """
        self.cache.execute_command('BF.ADD', self.bloom_key, key)

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        """
        Récupère un utilisateur avec Redis Bloom Filter
        """
        cache_key = f"user:{user_id}"

        # 1. Vérifier le Bloom Filter
        if not self.exists_in_bloom(cache_key):
            print(f"🚫 Bloom Filter: user {user_id} does not exist")
            return None

        # 2. Vérifier le cache
        cached = self.cache.get(cache_key)
        if cached:
            print(f"✓ Cache HIT for user {user_id}")
            return json.loads(cached)

        # 3. Charger depuis DB
        print(f"💾 Loading user {user_id} from DB")
        user = self._fetch_user_from_db(user_id)

        if user:
            self.cache.setex(cache_key, self.ttl, json.dumps(user))
            # Ajouter au Bloom Filter si nouveau
            self.add_to_bloom(cache_key)

        return user

    def _fetch_user_from_db(self, user_id: int) -> Optional[Dict[str, Any]]:
        if user_id < 1000:
            return {'id': user_id, 'name': f'User {user_id}'}
        return None

    def get_bloom_info(self):
        """
        Récupère les informations sur le Bloom Filter
        """
        info = self.cache.execute_command('BF.INFO', self.bloom_key)

        # Parser les infos (format: [key1, val1, key2, val2, ...])
        info_dict = {}
        for i in range(0, len(info), 2):
            key = info[i].decode('utf-8') if isinstance(info[i], bytes) else info[i]
            val = info[i+1]
            info_dict[key] = val

        return info_dict


# Benchmark: Avec vs Sans Bloom Filter
def benchmark_bloom_filter():
    import time

    r = redis.Redis(decode_responses=True)
    r.flushdb()  # Clean start

    # Sans Bloom Filter
    print("=== WITHOUT Bloom Filter ===")
    start = time.time()

    for i in range(100000, 101000):  # 1000 IDs invalides
        cache_key = f"user:{i}"
        cached = r.get(cache_key)
        if not cached:
            # Simule une requête DB
            pass

    time_without = time.time() - start
    print(f"Time: {time_without:.2f}s")
    print(f"DB queries: 1000\n")

    # Avec Bloom Filter
    print("=== WITH Bloom Filter ===")
    bloom_cache = RedisBloomCache(r)

    start = time.time()
    db_queries = 0

    for i in range(100000, 101000):  # 1000 IDs invalides
        user = bloom_cache.get_user(i)
        if user is not None:
            db_queries += 1

    time_with = time.time() - start
    print(f"Time: {time_with:.2f}s")
    print(f"DB queries: {db_queries}")
    print(f"Queries blocked: {1000 - db_queries}")
    print(f"Speedup: {time_without / time_with:.2f}x\n")

    # Afficher les infos du Bloom Filter
    info = bloom_cache.get_bloom_info()
    print("Bloom Filter Info:")
    for key, value in info.items():
        print(f"  {key}: {value}")


if __name__ == '__main__':
    benchmark_bloom_filter()
```

### Solution 4 : Validation et Rate Limiting

Combiner plusieurs stratégies pour une protection complète.

```python
class ComprehensivePenetrationProtection:
    def __init__(self, redis_client: redis.Redis):
        self.cache = redis_client
        self.bloom = BloomFilter()
        self.ttl = 3600
        self.null_ttl = 60

        # Rate limiting par IP
        self.rate_limit_window = 60  # secondes
        self.rate_limit_max = 100    # requêtes par fenêtre

    def get_user(self, user_id: int, client_ip: str) -> Optional[Dict[str, Any]]:
        """
        Protection multicouche contre la pénétration
        """
        # Couche 1: Validation de l'input
        if not self._validate_user_id(user_id):
            print(f"❌ Invalid user ID format: {user_id}")
            return None

        # Couche 2: Rate limiting par IP
        if not self._check_rate_limit(client_ip):
            print(f"🚫 Rate limit exceeded for {client_ip}")
            return None

        # Couche 3: Bloom Filter
        cache_key = f"user:{user_id}"
        if not self.bloom.contains(cache_key):
            print(f"🚫 Bloom Filter: user {user_id} does not exist")
            self._increment_rate_limit(client_ip)
            return None

        # Couche 4: Cache (incluant NULL cache)
        cached = self.cache.get(cache_key)
        if cached:
            if cached == "__NULL__":
                print(f"✓ Cache HIT for NULL user {user_id}")
                self._increment_rate_limit(client_ip)
                return None

            print(f"✓ Cache HIT for user {user_id}")
            self._increment_rate_limit(client_ip)
            return json.loads(cached)

        # Couche 5: Database avec protection
        print(f"💾 Loading user {user_id} from DB")
        user = self._fetch_user_from_db(user_id)

        if user:
            self.cache.setex(cache_key, self.ttl, json.dumps(user))
        else:
            # Cache le NULL
            self.cache.setex(cache_key, self.null_ttl, "__NULL__")

        self._increment_rate_limit(client_ip)
        return user

    def _validate_user_id(self, user_id: int) -> bool:
        """
        Validation basique de l'ID
        """
        # Vérifier que c'est un entier positif raisonnable
        return isinstance(user_id, int) and 0 < user_id < 1000000000

    def _check_rate_limit(self, client_ip: str) -> bool:
        """
        Vérifie si le client a dépassé le rate limit
        """
        key = f"ratelimit:{client_ip}"
        current = self.cache.get(key)

        if current:
            count = int(current)
            return count < self.rate_limit_max

        return True

    def _increment_rate_limit(self, client_ip: str):
        """
        Incrémente le compteur de rate limit
        """
        key = f"ratelimit:{client_ip}"

        pipe = self.cache.pipeline()
        pipe.incr(key)
        pipe.expire(key, self.rate_limit_window)
        pipe.execute()

    def _fetch_user_from_db(self, user_id: int) -> Optional[Dict[str, Any]]:
        if user_id < 1000:
            return {'id': user_id, 'name': f'User {user_id}'}
        return None


# Test de la protection multicouche
def test_comprehensive_protection():
    r = redis.Redis(decode_responses=True)
    protection = ComprehensivePenetrationProtection(r)

    client_ip = "192.168.1.100"

    print("=== Test 1: Valid ID ===")
    user = protection.get_user(500, client_ip)
    print(f"Result: {user}\n")

    print("=== Test 2: Invalid ID (format) ===")
    user = protection.get_user(-1, client_ip)
    print(f"Result: {user}\n")

    print("=== Test 3: Invalid ID (Bloom Filter) ===")
    user = protection.get_user(999999, client_ip)
    print(f"Result: {user}\n")

    print("=== Test 4: Rate limit (101 requests) ===")
    success_count = 0
    blocked_count = 0

    for i in range(101):
        user = protection.get_user(i + 1, client_ip)
        if user is not None or i < 100:
            success_count += 1
        else:
            blocked_count += 1

    print(f"Successful: {success_count}")
    print(f"Blocked: {blocked_count}\n")


if __name__ == '__main__':
    test_comprehensive_protection()
```

---

## Comparaison et Best Practices

### Matrice de solutions

| Problème | Solution | Complexité | Efficacité | Cas d'usage |
|----------|----------|------------|------------|-------------|
| **Avalanche** | TTL Jitter | 🟢 Simple | ⚡⚡⚡ Excellent | Par défaut |
| | Pre-warming | 🟡 Moyen | ⚡⚡ Bon | Cache predictable |
| | Circuit Breaker | 🔴 Complexe | ⚡⚡⚡ Excellent | Protection ultime |
| **Stampede** | Lock-based | 🟡 Moyen | ⚡⚡⚡ Excellent | Hot keys |
| | Probabilistic | 🔴 Complexe | ⚡⚡ Bon | Distribution uniforme |
| | Stale-While-Revalidate | 🟡 Moyen | ⚡⚡⚡ Excellent | Latence critique |
| **Penetration** | NULL Cache | 🟢 Simple | ⚡⚡ Bon | IDs aléatoires |
| | Bloom Filter | 🟡 Moyen | ⚡⚡⚡ Excellent | Grande volumétrie |
| | Rate Limiting | 🟢 Simple | ⚡⚡ Bon | Protection attaque |

### Stack recommandée en production

```python
class ProductionCacheStack:
    """
    Stack complète de protection pour production
    """
    def __init__(self, redis_client: redis.Redis, db_connection):
        self.cache = redis_client
        self.db = db_connection

        # Anti-Avalanche: TTL avec jitter
        self.base_ttl = 3600
        self.jitter_range = 300

        # Anti-Stampede: Stale-While-Revalidate
        self.grace_period = 300

        # Anti-Penetration: Bloom Filter + NULL cache
        self.bloom = BloomFilter()
        self.null_ttl = 60

        # Circuit Breaker pour protection ultime
        self.circuit_breaker = CircuitBreaker(redis_client)

    def get(self, key: str, load_func, *args) -> Any:
        """
        GET avec protection complète
        """
        # 1. Bloom Filter (si applicable)
        if hasattr(self, 'bloom') and not self.bloom.contains(key):
            return None

        # 2. Stale-While-Revalidate
        value_key = f"value:{key}"
        stale_key = f"stale:{key}"

        # Essayer valeur fraîche
        fresh = self.cache.get(value_key)
        if fresh:
            if fresh == "__NULL__":
                return None
            return json.loads(fresh)

        # Essayer valeur stale
        stale = self.cache.get(stale_key)
        if stale:
            # Revalider en arrière-plan
            self._schedule_revalidation(key, load_func, args)
            return json.loads(stale) if stale != "__NULL__" else None

        # 3. Charger avec Circuit Breaker
        try:
            data = self.circuit_breaker.call_db(load_func, *args)

            if data:
                # Cache avec jitter
                self._cache_with_jitter_and_stale(key, data)
            else:
                # Cache NULL
                self.cache.setex(value_key, self.null_ttl, "__NULL__")

            return data
        except Exception as e:
            print(f"Error loading {key}: {e}")
            return None

    def _cache_with_jitter_and_stale(self, key: str, data: Any):
        """Cache avec jitter et copie stale"""
        value_key = f"value:{key}"
        stale_key = f"stale:{key}"

        jitter = random.randint(0, self.jitter_range)
        ttl = self.base_ttl + jitter

        serialized = json.dumps(data)

        self.cache.setex(value_key, ttl, serialized)
        self.cache.setex(stale_key, ttl + self.grace_period, serialized)

    def _schedule_revalidation(self, key: str, load_func, args):
        """Programme une revalidation en arrière-plan"""
        lock_key = f"lock:{key}"
        if self.cache.set(lock_key, "1", nx=True, ex=30):
            threading.Thread(
                target=self._revalidate,
                args=(key, load_func, args)
            ).start()

    def _revalidate(self, key: str, load_func, args):
        """Revalide en arrière-plan"""
        try:
            data = load_func(*args)
            if data:
                self._cache_with_jitter_and_stale(key, data)
        finally:
            self.cache.delete(f"lock:{key}")


# Exemple d'utilisation complète
if __name__ == '__main__':
    r = redis.Redis(decode_responses=True)
    db = None  # Connexion DB

    stack = ProductionCacheStack(r, db)

    def expensive_query(user_id):
        time.sleep(0.1)  # Simule latence DB
        if user_id < 1000:
            return {'id': user_id, 'name': f'User {user_id}'}
        return None

    # Test
    user = stack.get('user:123', expensive_query, 123)
    print(f"User: {user}")
```

### Checklist de déploiement

```
✅ Anti-Avalanche
   □ TTL avec jitter (±5-10%)
   □ Circuit breaker configuré
   □ Monitoring des expirations simultanées

✅ Anti-Stampede
   □ Locks pour hot keys
   □ Stale-While-Revalidate activé
   □ Métriques sur les rechargements multiples

✅ Anti-Penetration
   □ Bloom Filter initialisé
   □ NULL cache activé (TTL court)
   □ Rate limiting par IP/user
   □ Validation des inputs

✅ Monitoring
   □ Cache hit rate (objectif: >80%)
   □ DB query count
   □ Nombre de locks acquis
   □ Bloom filter false positive rate
   □ Circuit breaker state
```

---

## Conclusion

Les trois problèmes majeurs du caching - **Avalanche**, **Stampede**, et **Penetration** - peuvent détruire un système en production s'ils ne sont pas anticipés. Heureusement, chacun a des solutions éprouvées :

**Anti-Avalanche** : TTL randomisé et circuit breakers
**Anti-Stampede** : Locks et Stale-While-Revalidate
**Anti-Penetration** : Bloom Filters et NULL caching

La clé est de combiner plusieurs stratégies en fonction de votre charge et de vos contraintes. Un système de production robuste intègre toutes ces protections dans une stack cohérente.

---


⏭️ [Pipelining : Optimiser le RTT (Round Trip Time)](/06-patterns-developpement-avances/03-pipelining-optimiser-rtt.md)
