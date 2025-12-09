🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1 Caching Patterns : Cache-Aside, Write-Through, Write-Back

## Introduction

Le caching est la raison d'être principale de Redis dans la plupart des architectures modernes. Un cache bien conçu peut réduire la latence de 100ms à moins de 1ms et diviser la charge sur la base de données primaire par 10, voire 100. Cependant, tous les patterns de caching ne se valent pas : le choix du bon pattern dépend de vos besoins en cohérence, performance et complexité.

Cette section explore les trois patterns de caching fondamentaux et leurs variantes, avec leurs trade-offs respectifs.

## Vue d'ensemble des patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    CACHING PATTERNS COMPARISON                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pattern          │ Reads      │ Writes     │ Consistency       │
│  ────────────────────────────────────────────────────────────── │
│  Cache-Aside     │ ⚡⚡⚡     │ 💾         │ 🟡 Eventually
│  (Lazy Loading)  │ Fast       │ Direct DB  │    Consistent      │
│                  │            │            │                    │
│  Write-Through   │ ⚡⚡       │ 💾💾      │ 🟢 Strong
│  (Write Cache)   │ Fast       │ Slower     │    Consistent      │
│                  │            │            │                    │
│  Write-Back      │ ⚡⚡⚡     │ ⚡⚡⚡     │ 🔴 Weak
│  (Write Behind)  │ Fast       │ Very Fast  │    Risk of Loss    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Choix rapide selon votre cas d'usage

| Cas d'usage | Pattern recommandé | Raison |
|------------|-------------------|---------|
| Read-heavy, eventual consistency OK | Cache-Aside | Simple, performant, résilient |
| Write-heavy, strong consistency | Write-Through | Garantit la cohérence |
| Write-heavy, latency critique | Write-Back | Maximum de performance |
| API REST standard | Cache-Aside | Pattern le plus courant |
| E-commerce checkout | Write-Through | Pas de perte de données |
| Analytics real-time | Write-Back | Débit d'écriture élevé |

---

## Pattern 1 : Cache-Aside (Lazy Loading)

### Concept

**Cache-Aside** est le pattern le plus répandu. L'application gère explicitement le cache : elle vérifie d'abord Redis, et si la donnée n'existe pas (cache miss), elle la charge depuis la base de données et la met en cache.

### Flux de lecture

```
┌──────────────┐
│ APPLICATION  │
└──────────────┘
       │
       │ 1. GET key
       ↓
┌──────────────┐
│    REDIS     │  ← Cache
└──────────────┘
       │
       ├─→ Cache HIT ─────→ Return data ✓
       │
       └─→ Cache MISS
              │
              │ 2. SELECT * FROM ...
              ↓
       ┌──────────────┐
       │  DATABASE    │
       └──────────────┘
              │
              │ 3. Return data
              ↓
       ┌──────────────┐
       │    REDIS     │
       │ SET key data │  ← 4. Populate cache
       └──────────────┘
              │
              └─→ Return data ✓
```

### Flux d'écriture

```
┌──────────────┐
│ APPLICATION  │
└──────────────┘
       │
       │ 1. UPDATE/INSERT
       ↓
┌──────────────┐
│  DATABASE    │  ← Primary store
└──────────────┘
       │
       │ 2. Success
       ↓
┌──────────────┐
│    REDIS     │
│   DEL key    │  ← 3. Invalidate cache (optional)
└──────────────┘
       │
       └─→ Return success ✓
```

### Implémentation Python

```python
import redis
import psycopg2
import json
from typing import Optional, Dict, Any
from datetime import timedelta

class CacheAsidePattern:
    def __init__(self, redis_client: redis.Redis, db_connection):
        self.cache = redis_client
        self.db = db_connection
        self.default_ttl = 3600  # 1 heure

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        """
        Récupère un utilisateur avec Cache-Aside pattern
        """
        cache_key = f"user:{user_id}"

        # 1. Essayer de lire depuis le cache
        cached_data = self.cache.get(cache_key)

        if cached_data:
            # Cache HIT
            print(f"✓ Cache HIT for user {user_id}")
            return json.loads(cached_data)

        # 2. Cache MISS - Charger depuis la DB
        print(f"✗ Cache MISS for user {user_id}")
        user = self._load_user_from_db(user_id)

        if user:
            # 3. Peupler le cache avec TTL
            self.cache.setex(
                cache_key,
                self.default_ttl,
                json.dumps(user)
            )
            print(f"→ Cached user {user_id} for {self.default_ttl}s")

        return user

    def update_user(self, user_id: int, user_data: Dict[str, Any]) -> bool:
        """
        Met à jour un utilisateur et invalide le cache
        """
        cache_key = f"user:{user_id}"

        # 1. Écrire dans la base de données
        success = self._update_user_in_db(user_id, user_data)

        if success:
            # 2. Invalider le cache (stratégie d'invalidation)
            self.cache.delete(cache_key)
            print(f"→ Invalidated cache for user {user_id}")

            # Alternative : Update immédiat du cache (Write-Through hybrid)
            # self.cache.setex(cache_key, self.default_ttl, json.dumps(user_data))

        return success

    def _load_user_from_db(self, user_id: int) -> Optional[Dict[str, Any]]:
        """Charge depuis PostgreSQL"""
        cursor = self.db.cursor()
        cursor.execute(
            "SELECT id, name, email, created_at FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()

        if row:
            return {
                'id': row[0],
                'name': row[1],
                'email': row[2],
                'created_at': str(row[3])
            }
        return None

    def _update_user_in_db(self, user_id: int, data: Dict[str, Any]) -> bool:
        """Met à jour dans PostgreSQL"""
        cursor = self.db.cursor()
        try:
            cursor.execute(
                "UPDATE users SET name = %s, email = %s WHERE id = %s",
                (data.get('name'), data.get('email'), user_id)
            )
            self.db.commit()
            return True
        except Exception as e:
            self.db.rollback()
            print(f"Error updating user: {e}")
            return False


# ===== VARIANTE : Cache-Aside avec Refresh automatique =====

class CacheAsideWithRefresh(CacheAsidePattern):
    """
    Variante qui rafraîchit automatiquement le cache sur lecture
    (Sliding Window Expiration)
    """

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        cache_key = f"user:{user_id}"

        # Lire depuis le cache
        cached_data = self.cache.get(cache_key)

        if cached_data:
            # Cache HIT - Rafraîchir le TTL (Sliding Window)
            self.cache.expire(cache_key, self.default_ttl)
            print(f"✓ Cache HIT for user {user_id} (TTL refreshed)")
            return json.loads(cached_data)

        # Cache MISS - Comportement standard
        return super().get_user(user_id)


# ===== VARIANTE : Cache-Aside avec Probabilistic Early Expiration =====

import random
import time

class CacheAsideWithProbabilisticExpiration(CacheAsidePattern):
    """
    Évite les Cache Stampedes en rechargeant le cache avant expiration
    avec une probabilité croissante
    """

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        cache_key = f"user:{user_id}"

        cached_data = self.cache.get(cache_key)

        if cached_data:
            # Vérifier le TTL restant
            ttl = self.cache.ttl(cache_key)

            if ttl > 0:
                # Calculer la probabilité de refresh anticipé
                # Plus le TTL est proche de 0, plus la probabilité est élevée
                expiry_threshold = self.default_ttl * 0.2  # 20% du TTL

                if ttl < expiry_threshold:
                    # Probabilité = (1 - ttl/threshold)
                    probability = 1 - (ttl / expiry_threshold)

                    if random.random() < probability:
                        print(f"🔄 Probabilistic refresh for user {user_id} (TTL: {ttl}s)")
                        # Refresh en arrière-plan (async dans un vrai système)
                        self._refresh_cache_async(user_id)

            return json.loads(cached_data)

        return super().get_user(user_id)

    def _refresh_cache_async(self, user_id: int):
        """Rafraîchit le cache en arrière-plan"""
        # Dans un vrai système, utiliser une queue ou un thread pool
        user = self._load_user_from_db(user_id)
        if user:
            cache_key = f"user:{user_id}"
            self.cache.setex(cache_key, self.default_ttl, json.dumps(user))


# ===== Exemple d'utilisation =====

if __name__ == '__main__':
    # Setup
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    db = psycopg2.connect("dbname=myapp user=postgres")

    cache = CacheAsidePattern(r, db)

    # Premier appel : Cache MISS
    user = cache.get_user(123)
    print(f"User: {user}")

    # Deuxième appel : Cache HIT
    user = cache.get_user(123)
    print(f"User: {user}")

    # Mise à jour : Invalide le cache
    cache.update_user(123, {'name': 'John Doe', 'email': 'john@example.com'})

    # Troisième appel : Cache MISS à nouveau
    user = cache.get_user(123)
    print(f"User: {user}")
```

### Implémentation Node.js

```javascript
const Redis = require('ioredis');
const { Pool } = require('pg');

class CacheAsidePattern {
    constructor(redisClient, dbPool) {
        this.cache = redisClient;
        this.db = dbPool;
        this.defaultTTL = 3600; // 1 heure
    }

    async getUser(userId) {
        const cacheKey = `user:${userId}`;

        // 1. Essayer de lire depuis le cache
        const cachedData = await this.cache.get(cacheKey);

        if (cachedData) {
            // Cache HIT
            console.log(`✓ Cache HIT for user ${userId}`);
            return JSON.parse(cachedData);
        }

        // 2. Cache MISS - Charger depuis la DB
        console.log(`✗ Cache MISS for user ${userId}`);
        const user = await this._loadUserFromDB(userId);

        if (user) {
            // 3. Peupler le cache avec TTL
            await this.cache.setex(
                cacheKey,
                this.defaultTTL,
                JSON.stringify(user)
            );
            console.log(`→ Cached user ${userId} for ${this.defaultTTL}s`);
        }

        return user;
    }

    async updateUser(userId, userData) {
        const cacheKey = `user:${userId}`;

        // 1. Écrire dans la base de données
        const success = await this._updateUserInDB(userId, userData);

        if (success) {
            // 2. Invalider le cache
            await this.cache.del(cacheKey);
            console.log(`→ Invalidated cache for user ${userId}`);
        }

        return success;
    }

    async _loadUserFromDB(userId) {
        const result = await this.db.query(
            'SELECT id, name, email, created_at FROM users WHERE id = $1',
            [userId]
        );

        if (result.rows.length > 0) {
            const row = result.rows[0];
            return {
                id: row.id,
                name: row.name,
                email: row.email,
                created_at: row.created_at.toISOString()
            };
        }

        return null;
    }

    async _updateUserInDB(userId, data) {
        try {
            await this.db.query(
                'UPDATE users SET name = $1, email = $2 WHERE id = $3',
                [data.name, data.email, userId]
            );
            return true;
        } catch (error) {
            console.error('Error updating user:', error);
            return false;
        }
    }
}

// ===== VARIANTE : Cache-Aside avec batching =====

class CacheAsideBatched extends CacheAsidePattern {
    async getUsers(userIds) {
        const cacheKeys = userIds.map(id => `user:${id}`);

        // 1. Batch read depuis Redis avec MGET
        const cachedData = await this.cache.mget(...cacheKeys);

        const results = [];
        const missedIds = [];

        // 2. Identifier les hits et les misses
        userIds.forEach((userId, index) => {
            if (cachedData[index]) {
                // Cache HIT
                results[index] = JSON.parse(cachedData[index]);
                console.log(`✓ Cache HIT for user ${userId}`);
            } else {
                // Cache MISS
                missedIds.push(userId);
                console.log(`✗ Cache MISS for user ${userId}`);
            }
        });

        // 3. Charger les misses depuis la DB (batch query)
        if (missedIds.length > 0) {
            const users = await this._loadUsersFromDB(missedIds);

            // 4. Peupler le cache avec pipeline
            const pipeline = this.cache.pipeline();

            users.forEach(user => {
                const cacheKey = `user:${user.id}`;
                pipeline.setex(
                    cacheKey,
                    this.defaultTTL,
                    JSON.stringify(user)
                );

                // Ajouter au résultat
                const originalIndex = userIds.indexOf(user.id);
                results[originalIndex] = user;
            });

            await pipeline.exec();
            console.log(`→ Cached ${users.length} users`);
        }

        return results;
    }

    async _loadUsersFromDB(userIds) {
        const result = await this.db.query(
            'SELECT id, name, email, created_at FROM users WHERE id = ANY($1)',
            [userIds]
        );

        return result.rows.map(row => ({
            id: row.id,
            name: row.name,
            email: row.email,
            created_at: row.created_at.toISOString()
        }));
    }
}

// ===== Exemple d'utilisation =====

(async () => {
    const redis = new Redis();
    const db = new Pool({
        host: 'localhost',
        database: 'myapp',
        user: 'postgres'
    });

    const cache = new CacheAsidePattern(redis, db);

    // Premier appel : Cache MISS
    let user = await cache.getUser(123);
    console.log('User:', user);

    // Deuxième appel : Cache HIT
    user = await cache.getUser(123);
    console.log('User:', user);

    // Test du batching
    const batchedCache = new CacheAsideBatched(redis, db);
    const users = await batchedCache.getUsers([123, 456, 789]);
    console.log('Users:', users);
})();
```

### Avantages et inconvénients

**✅ Avantages**
- **Simplicité** : Pattern le plus simple à comprendre et implémenter
- **Résilience** : Si Redis tombe, l'application continue de fonctionner (en lisant la DB)
- **Lazy loading** : Ne cache que ce qui est réellement lu
- **Contrôle fin** : L'application décide quoi et quand cacher

**❌ Inconvénients**
- **Cache Stampede** : Si une clé populaire expire, plusieurs requêtes peuvent charger simultanément
- **Cohérence** : Fenêtre où le cache peut être stale après une mise à jour
- **Initial latency** : Premier accès toujours lent (cache miss)
- **Code duplication** : Logique de cache dispersée dans le code applicatif

### Bonnes pratiques

```python
# ✅ BON : TTL approprié selon la volatilité des données
TTL_CONFIG = {
    'user_profile': 3600,      # 1h - Peu volatile
    'product_price': 300,       # 5min - Volatile
    'user_cart': 1800,          # 30min - Session-based
    'static_content': 86400     # 24h - Très stable
}

# ✅ BON : Gestion des cache misses multiples (Stampede protection)
import threading

class CacheWithStampedeProtection:
    def __init__(self, redis_client, db):
        self.cache = redis_client
        self.db = db
        self.locks = {}  # En mémoire, ou utiliser Redis SET NX

    def get_with_lock(self, key, load_fn, ttl=3600):
        # Essayer le cache
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # Acquérir un lock local pour éviter les requêtes multiples
        lock = self.locks.setdefault(key, threading.Lock())

        with lock:
            # Double-check après acquisition du lock
            cached = self.cache.get(key)
            if cached:
                return json.loads(cached)

            # Charger la donnée
            data = load_fn()
            if data:
                self.cache.setex(key, ttl, json.dumps(data))

            return data

# ✅ BON : Namespace et versioning
def build_cache_key(entity, id, version='v1'):
    return f"{version}:{entity}:{id}"

# Si le schéma change, incrémenter la version
key = build_cache_key('user', 123, 'v2')  # Invalide automatiquement v1

# ✅ BON : Metrics et observabilité
class CacheWithMetrics:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db
        self.hits = 0
        self.misses = 0

    def get(self, key, load_fn, ttl=3600):
        cached = self.cache.get(key)

        if cached:
            self.hits += 1
            return json.loads(cached)

        self.misses += 1
        data = load_fn()
        if data:
            self.cache.setex(key, ttl, json.dumps(data))
        return data

    def hit_rate(self):
        total = self.hits + self.misses
        return (self.hits / total * 100) if total > 0 else 0
```

---

## Pattern 2 : Write-Through

### Concept

Dans le pattern **Write-Through**, toute écriture passe d'abord par le cache, qui se charge ensuite de persister dans la base de données. Le cache et la DB sont toujours synchronisés : aucune donnée ne peut exister dans la DB sans être dans le cache.

### Flux de lecture

```
┌──────────────┐
│ APPLICATION  │
└──────────────┘
       │
       │ 1. GET key
       ↓
┌──────────────┐
│    REDIS     │  ← Always in sync with DB
└──────────────┘
       │
       ├─→ Cache HIT ─────→ Return data ✓
       │
       └─→ Cache MISS
              │
              │ 2. SELECT * FROM ...
              ↓
       ┌──────────────┐
       │  DATABASE    │
       └──────────────┘
              │
              │ 3. Return data
              ↓
       ┌──────────────┐
       │    REDIS     │
       │ SET key data │  ← 4. Populate cache
       └──────────────┘
```

### Flux d'écriture

```
┌──────────────┐
│ APPLICATION  │
└──────────────┘
       │
       │ 1. UPDATE data
       ↓
┌──────────────┐
│    REDIS     │  ← Write to cache first
│ SET key data │
└──────────────┘
       │
       │ 2. Write to DB
       ↓
┌──────────────┐
│  DATABASE    │  ← Then persist
│    UPDATE    │
└──────────────┘
       │
       └─→ Return success ✓

    ⚠️ If DB write fails → Rollback cache
```

### Implémentation Python

```python
import redis
import psycopg2
import json
from typing import Optional, Dict, Any
from contextlib import contextmanager

class WriteThroughPattern:
    def __init__(self, redis_client: redis.Redis, db_connection):
        self.cache = redis_client
        self.db = db_connection
        self.default_ttl = 3600

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        """
        Lecture similaire à Cache-Aside
        """
        cache_key = f"user:{user_id}"

        # Essayer le cache
        cached_data = self.cache.get(cache_key)
        if cached_data:
            print(f"✓ Cache HIT for user {user_id}")
            return json.loads(cached_data)

        # Cache MISS - Charger depuis DB
        print(f"✗ Cache MISS for user {user_id}")
        user = self._load_user_from_db(user_id)

        if user:
            self.cache.setex(cache_key, self.default_ttl, json.dumps(user))

        return user

    def update_user(self, user_id: int, user_data: Dict[str, Any]) -> bool:
        """
        Write-Through : Cache d'abord, puis DB
        """
        cache_key = f"user:{user_id}"

        # 1. Écrire dans le cache d'abord
        try:
            self.cache.setex(
                cache_key,
                self.default_ttl,
                json.dumps(user_data)
            )
            print(f"→ Wrote to cache for user {user_id}")
        except Exception as e:
            print(f"✗ Cache write failed: {e}")
            return False

        # 2. Puis persister dans la DB
        try:
            success = self._update_user_in_db(user_id, user_data)

            if not success:
                # ROLLBACK : Supprimer du cache si DB échoue
                self.cache.delete(cache_key)
                print(f"⚠️  Rolled back cache for user {user_id}")
                return False

            print(f"✓ Wrote to DB for user {user_id}")
            return True

        except Exception as e:
            # ROLLBACK du cache
            self.cache.delete(cache_key)
            print(f"✗ DB write failed, rolled back cache: {e}")
            return False

    def create_user(self, user_data: Dict[str, Any]) -> Optional[int]:
        """
        Création avec Write-Through
        """
        # 1. Créer dans la DB pour obtenir l'ID
        user_id = self._create_user_in_db(user_data)

        if user_id:
            # 2. Peupler immédiatement le cache
            cache_key = f"user:{user_id}"
            user_data['id'] = user_id

            self.cache.setex(
                cache_key,
                self.default_ttl,
                json.dumps(user_data)
            )
            print(f"✓ Created user {user_id} in DB and cache")

        return user_id

    def _load_user_from_db(self, user_id: int) -> Optional[Dict[str, Any]]:
        cursor = self.db.cursor()
        cursor.execute(
            "SELECT id, name, email, created_at FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()

        if row:
            return {
                'id': row[0],
                'name': row[1],
                'email': row[2],
                'created_at': str(row[3])
            }
        return None

    def _update_user_in_db(self, user_id: int, data: Dict[str, Any]) -> bool:
        cursor = self.db.cursor()
        try:
            cursor.execute(
                "UPDATE users SET name = %s, email = %s WHERE id = %s",
                (data.get('name'), data.get('email'), user_id)
            )
            self.db.commit()
            return True
        except Exception as e:
            self.db.rollback()
            raise e

    def _create_user_in_db(self, data: Dict[str, Any]) -> Optional[int]:
        cursor = self.db.cursor()
        try:
            cursor.execute(
                "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
                (data.get('name'), data.get('email'))
            )
            user_id = cursor.fetchone()[0]
            self.db.commit()
            return user_id
        except Exception as e:
            self.db.rollback()
            print(f"Error creating user: {e}")
            return None


# ===== VARIANTE : Write-Through avec Transaction distribuée =====

class WriteThroughWithTransaction(WriteThroughPattern):
    """
    Utilise WATCH pour garantir la cohérence entre Redis et la DB
    """

    def update_user_atomic(self, user_id: int, user_data: Dict[str, Any]) -> bool:
        cache_key = f"user:{user_id}"

        # Démarrer une transaction Redis avec WATCH
        with self.cache.pipeline() as pipe:
            try:
                # WATCH la clé pour détecter les modifications concurrentes
                pipe.watch(cache_key)

                # Lire la valeur actuelle
                current_value = pipe.get(cache_key)

                # Écrire dans la DB
                success = self._update_user_in_db(user_id, user_data)

                if not success:
                    pipe.unwatch()
                    return False

                # Transaction MULTI/EXEC
                pipe.multi()
                pipe.setex(cache_key, self.default_ttl, json.dumps(user_data))
                pipe.execute()

                print(f"✓ Atomic write for user {user_id}")
                return True

            except redis.WatchError:
                # La clé a été modifiée pendant la transaction
                print(f"⚠️  Concurrent modification detected for user {user_id}")
                return False


# ===== VARIANTE : Write-Through avec Queue pour la DB =====

import queue
import threading

class WriteThroughWithQueue(WriteThroughPattern):
    """
    Écrit dans Redis immédiatement, mais utilise une queue pour la DB
    (Hybride entre Write-Through et Write-Back)
    """

    def __init__(self, redis_client, db_connection):
        super().__init__(redis_client, db_connection)
        self.write_queue = queue.Queue()
        self.worker_thread = threading.Thread(target=self._process_writes)
        self.worker_thread.daemon = True
        self.worker_thread.start()

    def update_user(self, user_id: int, user_data: Dict[str, Any]) -> bool:
        cache_key = f"user:{user_id}"

        # 1. Écriture immédiate dans Redis
        try:
            self.cache.setex(cache_key, self.default_ttl, json.dumps(user_data))
            print(f"→ Wrote to cache for user {user_id}")
        except Exception as e:
            print(f"✗ Cache write failed: {e}")
            return False

        # 2. Enqueuer pour écriture DB asynchrone
        self.write_queue.put(('update', user_id, user_data))
        print(f"→ Enqueued DB write for user {user_id}")

        return True

    def _process_writes(self):
        """Worker thread qui traite les écritures DB"""
        while True:
            try:
                operation, user_id, data = self.write_queue.get(timeout=1)

                if operation == 'update':
                    success = self._update_user_in_db(user_id, data)

                    if not success:
                        # En cas d'échec, on pourrait re-enqueuer ou alerter
                        print(f"⚠️  DB write failed for user {user_id}")
                    else:
                        print(f"✓ DB write completed for user {user_id}")

                self.write_queue.task_done()

            except queue.Empty:
                continue
            except Exception as e:
                print(f"Error in write worker: {e}")


# ===== Exemple d'utilisation =====

if __name__ == '__main__':
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    db = psycopg2.connect("dbname=myapp user=postgres")

    cache = WriteThroughPattern(r, db)

    # Créer un utilisateur (Write-Through)
    user_id = cache.create_user({
        'name': 'Alice',
        'email': 'alice@example.com'
    })
    print(f"Created user: {user_id}")

    # Lecture : déjà en cache
    user = cache.get_user(user_id)
    print(f"User: {user}")

    # Mise à jour (Write-Through)
    success = cache.update_user(user_id, {
        'name': 'Alice Smith',
        'email': 'alice.smith@example.com'
    })
    print(f"Update success: {success}")

    # Lecture : cache à jour
    user = cache.get_user(user_id)
    print(f"Updated user: {user}")
```

### Implémentation Node.js

```javascript
const Redis = require('ioredis');
const { Pool } = require('pg');

class WriteThroughPattern {
    constructor(redisClient, dbPool) {
        this.cache = redisClient;
        this.db = dbPool;
        this.defaultTTL = 3600;
    }

    async getUser(userId) {
        const cacheKey = `user:${userId}`;

        // Essayer le cache
        const cachedData = await this.cache.get(cacheKey);
        if (cachedData) {
            console.log(`✓ Cache HIT for user ${userId}`);
            return JSON.parse(cachedData);
        }

        // Cache MISS
        console.log(`✗ Cache MISS for user ${userId}`);
        const user = await this._loadUserFromDB(userId);

        if (user) {
            await this.cache.setex(
                cacheKey,
                this.defaultTTL,
                JSON.stringify(user)
            );
        }

        return user;
    }

    async updateUser(userId, userData) {
        const cacheKey = `user:${userId}`;

        // 1. Écrire dans le cache d'abord
        try {
            await this.cache.setex(
                cacheKey,
                this.defaultTTL,
                JSON.stringify(userData)
            );
            console.log(`→ Wrote to cache for user ${userId}`);
        } catch (error) {
            console.error(`✗ Cache write failed: ${error}`);
            return false;
        }

        // 2. Puis persister dans la DB
        try {
            const success = await this._updateUserInDB(userId, userData);

            if (!success) {
                // ROLLBACK : Supprimer du cache
                await this.cache.del(cacheKey);
                console.log(`⚠️  Rolled back cache for user ${userId}`);
                return false;
            }

            console.log(`✓ Wrote to DB for user ${userId}`);
            return true;

        } catch (error) {
            // ROLLBACK du cache
            await this.cache.del(cacheKey);
            console.error(`✗ DB write failed, rolled back cache: ${error}`);
            return false;
        }
    }

    async createUser(userData) {
        // 1. Créer dans la DB pour obtenir l'ID
        const userId = await this._createUserInDB(userData);

        if (userId) {
            // 2. Peupler immédiatement le cache
            const cacheKey = `user:${userId}`;
            userData.id = userId;

            await this.cache.setex(
                cacheKey,
                this.defaultTTL,
                JSON.stringify(userData)
            );
            console.log(`✓ Created user ${userId} in DB and cache`);
        }

        return userId;
    }

    async _loadUserFromDB(userId) {
        const result = await this.db.query(
            'SELECT id, name, email, created_at FROM users WHERE id = $1',
            [userId]
        );

        if (result.rows.length > 0) {
            const row = result.rows[0];
            return {
                id: row.id,
                name: row.name,
                email: row.email,
                created_at: row.created_at.toISOString()
            };
        }

        return null;
    }

    async _updateUserInDB(userId, data) {
        const client = await this.db.connect();
        try {
            await client.query(
                'UPDATE users SET name = $1, email = $2 WHERE id = $3',
                [data.name, data.email, userId]
            );
            return true;
        } catch (error) {
            throw error;
        } finally {
            client.release();
        }
    }

    async _createUserInDB(data) {
        const client = await this.db.connect();
        try {
            const result = await client.query(
                'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id',
                [data.name, data.email]
            );
            return result.rows[0].id;
        } catch (error) {
            console.error('Error creating user:', error);
            return null;
        } finally {
            client.release();
        }
    }
}

// ===== VARIANTE : Write-Through avec Retry Logic =====

class WriteThroughWithRetry extends WriteThroughPattern {
    async updateUserWithRetry(userId, userData, maxRetries = 3) {
        let attempts = 0;

        while (attempts < maxRetries) {
            try {
                return await this.updateUser(userId, userData);
            } catch (error) {
                attempts++;
                console.log(`Retry attempt ${attempts}/${maxRetries}`);

                if (attempts >= maxRetries) {
                    throw error;
                }

                // Exponential backoff
                await new Promise(resolve =>
                    setTimeout(resolve, Math.pow(2, attempts) * 100)
                );
            }
        }
    }
}

// ===== Exemple d'utilisation =====

(async () => {
    const redis = new Redis();
    const db = new Pool({
        host: 'localhost',
        database: 'myapp',
        user: 'postgres'
    });

    const cache = new WriteThroughPattern(redis, db);

    // Créer un utilisateur
    const userId = await cache.createUser({
        name: 'Bob',
        email: 'bob@example.com'
    });
    console.log(`Created user: ${userId}`);

    // Mise à jour Write-Through
    const success = await cache.updateUser(userId, {
        name: 'Bob Johnson',
        email: 'bob.johnson@example.com'
    });
    console.log(`Update success: ${success}`);

    // Lecture depuis le cache
    const user = await cache.getUser(userId);
    console.log('User:', user);
})();
```

### Avantages et inconvénients

**✅ Avantages**
- **Strong consistency** : Cache et DB toujours synchronisés
- **Simple reasoning** : Pas de période de staleness
- **Cache toujours chaud** : Toutes les données récentes sont en cache
- **Protection des reads** : Cache protège la DB des lectures

**❌ Inconvénients**
- **Write latency** : Chaque écriture attend la DB (plus lent)
- **Write amplification** : Toutes les écritures passent par 2 systèmes
- **Single point of failure** : Si Redis tombe, les écritures échouent
- **Complexité de rollback** : Gérer les échecs DB nécessite du code

### Bonnes pratiques

```python
# ✅ BON : Transaction pour garantir atomicité
def update_with_transaction(redis_client, db, user_id, data):
    cache_key = f"user:{user_id}"

    # Utiliser une transaction DB
    cursor = db.cursor()
    try:
        cursor.execute("BEGIN")

        # Écriture DB
        cursor.execute(
            "UPDATE users SET name = %s WHERE id = %s",
            (data['name'], user_id)
        )

        # Écriture cache seulement si DB réussit
        redis_client.setex(cache_key, 3600, json.dumps(data))

        cursor.execute("COMMIT")
        return True
    except Exception as e:
        cursor.execute("ROLLBACK")
        return False

# ✅ BON : Logging pour audit trail
import logging

class WriteThroughWithAudit(WriteThroughPattern):
    def __init__(self, redis_client, db):
        super().__init__(redis_client, db)
        self.logger = logging.getLogger(__name__)

    def update_user(self, user_id, data):
        self.logger.info(f"Updating user {user_id}: {data}")

        success = super().update_user(user_id, data)

        if success:
            self.logger.info(f"Successfully updated user {user_id}")
        else:
            self.logger.error(f"Failed to update user {user_id}")

        return success

# ✅ BON : Health check pour détecter les désynchronisations
def verify_consistency(redis_client, db, user_id):
    """Vérifie que cache et DB sont synchronisés"""
    cache_key = f"user:{user_id}"

    # Lire depuis cache
    cached = redis_client.get(cache_key)

    # Lire depuis DB
    cursor = db.cursor()
    cursor.execute("SELECT name, email FROM users WHERE id = %s", (user_id,))
    db_data = cursor.fetchone()

    if cached and db_data:
        cached_obj = json.loads(cached)
        return (cached_obj['name'] == db_data[0] and
                cached_obj['email'] == db_data[1])

    return cached is None and db_data is None
```

---

## Pattern 3 : Write-Back (Write-Behind)

### Concept

Le pattern **Write-Back** écrit d'abord dans Redis, puis persiste dans la base de données de manière **asynchrone** et **batched**. C'est le pattern le plus performant mais aussi le plus risqué : les données peuvent être perdues si Redis crash avant la persistance.

### Flux d'écriture

```
┌──────────────┐
│ APPLICATION  │
└──────────────┘
       │
       │ 1. UPDATE data (fast!)
       ↓
┌──────────────┐
│    REDIS     │  ← Immediate write
│ SET key data │
│ LPUSH queue  │  ← Add to write queue
└──────────────┘
       │
       └─→ Return success immediately ✓


   [Background Worker Thread/Process]
              │
              │ 2. RPOP queue (batch)
              ↓
       ┌──────────────┐
       │ Write Buffer │  ← Accumulate writes
       └──────────────┘
              │
              │ 3. Batch INSERT/UPDATE
              ↓
       ┌──────────────┐
       │  DATABASE    │  ← Async persistence
       └──────────────┘
```

### Architecture détaillée

```
┌───────────────────────────────────────────────────────────────┐
│                    WRITE-BACK ARCHITECTURE                    │
└───────────────────────────────────────────────────────────────┘

  [Client Requests]
         │
         ↓
  ┌─────────────┐
  │   Redis     │
  │  (Cache)    │ ← 1. Immediate write (< 1ms)
  └─────────────┘
         │
         ├─→ [Value] : user:123 → {"name": "Alice"}
         │
         └─→ [Queue] : write_queue → [
                 {"table": "users", "id": 123, "data": {...}},
                 {"table": "users", "id": 456, "data": {...}},
                 ...
             ]

  ┌─────────────────────────────────────────┐
  │      Background Worker (Async)          │
  │                                         │
  │  while True:                            │
  │    items = RPOP write_queue 100         │
  │    batch_write_to_db(items)             │
  │    sleep(1)                             │
  └─────────────────────────────────────────┘
         │
         ↓
  ┌─────────────┐
  │ PostgreSQL  │ ← 2. Batched write (periodic)
  │  (Primary)  │
  └─────────────┘
```

### Implémentation Python

```python
import redis
import psycopg2
import json
import time
import threading
from typing import Dict, Any, List
from datetime import datetime

class WriteBackPattern:
    def __init__(self, redis_client: redis.Redis, db_connection):
        self.cache = redis_client
        self.db = db_connection
        self.default_ttl = 3600
        self.write_queue_key = "write_queue"
        self.batch_size = 100
        self.flush_interval = 1  # secondes

        # Démarrer le worker en arrière-plan
        self.worker_thread = threading.Thread(target=self._background_writer)
        self.worker_thread.daemon = True
        self.worker_thread.start()

        print("✓ Write-Back worker started")

    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        """
        Lecture depuis le cache (Write-Back maintient le cache chaud)
        """
        cache_key = f"user:{user_id}"

        cached_data = self.cache.get(cache_key)
        if cached_data:
            print(f"✓ Cache HIT for user {user_id}")
            return json.loads(cached_data)

        # Si pas en cache, charger depuis DB
        print(f"✗ Cache MISS for user {user_id}")
        user = self._load_user_from_db(user_id)

        if user:
            self.cache.setex(cache_key, self.default_ttl, json.dumps(user))

        return user

    def update_user(self, user_id: int, user_data: Dict[str, Any]) -> bool:
        """
        Write-Back : Écriture immédiate dans Redis, async vers DB
        """
        cache_key = f"user:{user_id}"

        # 1. Écriture IMMÉDIATE dans Redis
        try:
            self.cache.setex(
                cache_key,
                self.default_ttl,
                json.dumps(user_data)
            )
            print(f"→ Wrote to cache for user {user_id}")
        except Exception as e:
            print(f"✗ Cache write failed: {e}")
            return False

        # 2. Enqueuer pour écriture DB asynchrone
        write_job = {
            'table': 'users',
            'operation': 'update',
            'id': user_id,
            'data': user_data,
            'timestamp': datetime.now().isoformat()
        }

        self.cache.lpush(self.write_queue_key, json.dumps(write_job))
        print(f"→ Enqueued DB write for user {user_id}")

        # Retourne immédiatement (sans attendre la DB)
        return True

    def create_user(self, user_data: Dict[str, Any]) -> int:
        """
        Création avec Write-Back
        Note : L'ID doit être généré côté application ou via Redis INCR
        """
        # Générer un ID avec Redis
        user_id = int(self.cache.incr("user_id_sequence"))
        user_data['id'] = user_id

        # Écriture immédiate dans Redis
        cache_key = f"user:{user_id}"
        self.cache.setex(cache_key, self.default_ttl, json.dumps(user_data))

        # Enqueuer pour DB
        write_job = {
            'table': 'users',
            'operation': 'insert',
            'id': user_id,
            'data': user_data,
            'timestamp': datetime.now().isoformat()
        }

        self.cache.lpush(self.write_queue_key, json.dumps(write_job))
        print(f"✓ Created user {user_id} (async to DB)")

        return user_id

    def _background_writer(self):
        """
        Worker thread qui traite la queue d'écriture
        """
        while True:
            try:
                # Récupérer un batch d'écritures
                jobs = self._pop_batch(self.batch_size)

                if jobs:
                    print(f"→ Processing batch of {len(jobs)} writes")
                    self._batch_write_to_db(jobs)
                    print(f"✓ Batch persisted to DB")

                # Attendre avant le prochain flush
                time.sleep(self.flush_interval)

            except Exception as e:
                print(f"✗ Error in background writer: {e}")
                time.sleep(5)  # Backoff en cas d'erreur

    def _pop_batch(self, size: int) -> List[Dict[str, Any]]:
        """
        Récupère un batch depuis la queue
        """
        jobs = []

        # Utiliser RPOP en boucle pour récupérer plusieurs éléments
        # (Redis 6.2+ supporte RPOP avec count)
        for _ in range(size):
            job_json = self.cache.rpop(self.write_queue_key)
            if not job_json:
                break
            jobs.append(json.loads(job_json))

        return jobs

    def _batch_write_to_db(self, jobs: List[Dict[str, Any]]):
        """
        Écrit un batch dans la DB (optimisé)
        """
        if not jobs:
            return

        cursor = self.db.cursor()

        try:
            # Grouper par type d'opération
            inserts = [j for j in jobs if j['operation'] == 'insert']
            updates = [j for j in jobs if j['operation'] == 'update']

            # Batch INSERT
            if inserts:
                insert_values = [
                    (j['id'], j['data']['name'], j['data']['email'])
                    for j in inserts
                ]

                cursor.executemany(
                    "INSERT INTO users (id, name, email) VALUES (%s, %s, %s) "
                    "ON CONFLICT (id) DO NOTHING",
                    insert_values
                )

            # Batch UPDATE
            if updates:
                update_values = [
                    (j['data']['name'], j['data']['email'], j['id'])
                    for j in updates
                ]

                cursor.executemany(
                    "UPDATE users SET name = %s, email = %s WHERE id = %s",
                    update_values
                )

            self.db.commit()

        except Exception as e:
            self.db.rollback()
            print(f"✗ Batch write failed: {e}")

            # Re-enqueuer les jobs échoués (avec retry logic)
            for job in jobs:
                self.cache.lpush(self.write_queue_key, json.dumps(job))

            raise e

    def _load_user_from_db(self, user_id: int) -> Optional[Dict[str, Any]]:
        cursor = self.db.cursor()
        cursor.execute(
            "SELECT id, name, email, created_at FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()

        if row:
            return {
                'id': row[0],
                'name': row[1],
                'email': row[2],
                'created_at': str(row[3])
            }
        return None

    def force_flush(self):
        """
        Force le flush de toutes les écritures en attente
        Utile lors d'un shutdown graceful
        """
        print("→ Forcing flush of pending writes...")

        while True:
            jobs = self._pop_batch(self.batch_size)
            if not jobs:
                break
            self._batch_write_to_db(jobs)

        print("✓ All pending writes flushed")


# ===== VARIANTE : Write-Back avec priorité =====

class WriteBackWithPriority(WriteBackPattern):
    """
    Supporte plusieurs queues avec différentes priorités
    """

    def __init__(self, redis_client, db):
        super().__init__(redis_client, db)
        self.high_priority_queue = "write_queue:high"
        self.low_priority_queue = "write_queue:low"

    def update_user(self, user_id: int, user_data: Dict[str, Any], priority='normal') -> bool:
        cache_key = f"user:{user_id}"

        # Écriture cache
        self.cache.setex(cache_key, self.default_ttl, json.dumps(user_data))

        # Choisir la queue selon priorité
        queue_key = (self.high_priority_queue if priority == 'high'
                    else self.write_queue_key)

        write_job = {
            'table': 'users',
            'operation': 'update',
            'id': user_id,
            'data': user_data,
            'priority': priority,
            'timestamp': datetime.now().isoformat()
        }

        self.cache.lpush(queue_key, json.dumps(write_job))
        print(f"→ Enqueued {priority} priority write for user {user_id}")

        return True

    def _background_writer(self):
        """Worker qui traite la high priority en premier"""
        while True:
            try:
                # Traiter high priority d'abord
                high_jobs = self._pop_batch_from_queue(
                    self.high_priority_queue,
                    self.batch_size
                )

                if high_jobs:
                    print(f"→ Processing HIGH priority batch: {len(high_jobs)}")
                    self._batch_write_to_db(high_jobs)

                # Puis normal priority
                normal_jobs = self._pop_batch(self.batch_size)

                if normal_jobs:
                    print(f"→ Processing NORMAL priority batch: {len(normal_jobs)}")
                    self._batch_write_to_db(normal_jobs)

                time.sleep(self.flush_interval)

            except Exception as e:
                print(f"✗ Error in background writer: {e}")
                time.sleep(5)

    def _pop_batch_from_queue(self, queue_key: str, size: int) -> List[Dict]:
        jobs = []
        for _ in range(size):
            job_json = self.cache.rpop(queue_key)
            if not job_json:
                break
            jobs.append(json.loads(job_json))
        return jobs


# ===== VARIANTE : Write-Back avec Dead Letter Queue =====

class WriteBackWithDLQ(WriteBackPattern):
    """
    Écritures qui échouent après X retries vont dans une Dead Letter Queue
    """

    def __init__(self, redis_client, db):
        super().__init__(redis_client, db)
        self.dlq_key = "write_queue:dlq"
        self.max_retries = 3

    def _batch_write_to_db(self, jobs: List[Dict[str, Any]]):
        cursor = self.db.cursor()

        for job in jobs:
            retry_count = job.get('retry_count', 0)

            try:
                # Tenter l'écriture
                if job['operation'] == 'update':
                    cursor.execute(
                        "UPDATE users SET name = %s, email = %s WHERE id = %s",
                        (job['data']['name'], job['data']['email'], job['id'])
                    )

                self.db.commit()

            except Exception as e:
                self.db.rollback()

                if retry_count < self.max_retries:
                    # Re-enqueuer avec retry_count incrémenté
                    job['retry_count'] = retry_count + 1
                    self.cache.lpush(self.write_queue_key, json.dumps(job))
                    print(f"⚠️  Retry {retry_count + 1}/{self.max_retries} for job {job['id']}")
                else:
                    # Déplacer vers DLQ
                    self.cache.lpush(self.dlq_key, json.dumps(job))
                    print(f"✗ Job {job['id']} moved to DLQ after {self.max_retries} retries")


# ===== Exemple d'utilisation =====

if __name__ == '__main__':
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    db = psycopg2.connect("dbname=myapp user=postgres")

    cache = WriteBackPattern(r, db)

    # Créer plusieurs utilisateurs (très rapide!)
    for i in range(5):
        user_id = cache.create_user({
            'name': f'User {i}',
            'email': f'user{i}@example.com'
        })
        print(f"Created user {user_id}")

    # Les écritures DB se font en arrière-plan
    print("\n→ Waiting for background writes...")
    time.sleep(3)

    # Lecture depuis le cache (toujours à jour)
    user = cache.get_user(1)
    print(f"\nUser from cache: {user}")

    # Mise à jour (retour immédiat)
    cache.update_user(1, {
        'name': 'Updated User',
        'email': 'updated@example.com'
    })

    # Force flush avant shutdown
    print("\n→ Shutting down...")
    cache.force_flush()
    print("✓ Shutdown complete")
```

### Implémentation Node.js

```javascript
const Redis = require('ioredis');
const { Pool } = require('pg');

class WriteBackPattern {
    constructor(redisClient, dbPool) {
        this.cache = redisClient;
        this.db = dbPool;
        this.defaultTTL = 3600;
        this.writeQueueKey = 'write_queue';
        this.batchSize = 100;
        this.flushInterval = 1000; // ms

        // Démarrer le worker
        this._startBackgroundWriter();
        console.log('✓ Write-Back worker started');
    }

    async getUser(userId) {
        const cacheKey = `user:${userId}`;

        const cachedData = await this.cache.get(cacheKey);
        if (cachedData) {
            console.log(`✓ Cache HIT for user ${userId}`);
            return JSON.parse(cachedData);
        }

        console.log(`✗ Cache MISS for user ${userId}`);
        const user = await this._loadUserFromDB(userId);

        if (user) {
            await this.cache.setex(
                cacheKey,
                this.defaultTTL,
                JSON.stringify(user)
            );
        }

        return user;
    }

    async updateUser(userId, userData) {
        const cacheKey = `user:${userId}`;

        // 1. Écriture IMMÉDIATE dans Redis
        try {
            await this.cache.setex(
                cacheKey,
                this.defaultTTL,
                JSON.stringify(userData)
            );
            console.log(`→ Wrote to cache for user ${userId}`);
        } catch (error) {
            console.error(`✗ Cache write failed: ${error}`);
            return false;
        }

        // 2. Enqueuer pour DB
        const writeJob = {
            table: 'users',
            operation: 'update',
            id: userId,
            data: userData,
            timestamp: new Date().toISOString()
        };

        await this.cache.lpush(
            this.writeQueueKey,
            JSON.stringify(writeJob)
        );
        console.log(`→ Enqueued DB write for user ${userId}`);

        return true; // Retour immédiat
    }

    async createUser(userData) {
        // Générer un ID avec Redis
        const userId = await this.cache.incr('user_id_sequence');
        userData.id = userId;

        // Écriture cache
        const cacheKey = `user:${userId}`;
        await this.cache.setex(
            cacheKey,
            this.defaultTTL,
            JSON.stringify(userData)
        );

        // Enqueuer
        const writeJob = {
            table: 'users',
            operation: 'insert',
            id: userId,
            data: userData,
            timestamp: new Date().toISOString()
        };

        await this.cache.lpush(
            this.writeQueueKey,
            JSON.stringify(writeJob)
        );
        console.log(`✓ Created user ${userId} (async to DB)`);

        return userId;
    }

    _startBackgroundWriter() {
        this.writerInterval = setInterval(
            async () => {
                try {
                    const jobs = await this._popBatch(this.batchSize);

                    if (jobs.length > 0) {
                        console.log(`→ Processing batch of ${jobs.length} writes`);
                        await this._batchWriteToDB(jobs);
                        console.log(`✓ Batch persisted to DB`);
                    }
                } catch (error) {
                    console.error(`✗ Error in background writer: ${error}`);
                }
            },
            this.flushInterval
        );
    }

    async _popBatch(size) {
        const jobs = [];

        for (let i = 0; i < size; i++) {
            const jobJson = await this.cache.rpop(this.writeQueueKey);
            if (!jobJson) break;
            jobs.push(JSON.parse(jobJson));
        }

        return jobs;
    }

    async _batchWriteToDB(jobs) {
        if (jobs.length === 0) return;

        const client = await this.db.connect();

        try {
            await client.query('BEGIN');

            // Grouper par opération
            const inserts = jobs.filter(j => j.operation === 'insert');
            const updates = jobs.filter(j => j.operation === 'update');

            // Batch INSERT
            if (inserts.length > 0) {
                const values = inserts.map(j =>
                    `(${j.id}, '${j.data.name}', '${j.data.email}')`
                ).join(',');

                await client.query(
                    `INSERT INTO users (id, name, email) VALUES ${values}
                     ON CONFLICT (id) DO NOTHING`
                );
            }

            // Batch UPDATE
            for (const job of updates) {
                await client.query(
                    'UPDATE users SET name = $1, email = $2 WHERE id = $3',
                    [job.data.name, job.data.email, job.id]
                );
            }

            await client.query('COMMIT');

        } catch (error) {
            await client.query('ROLLBACK');
            console.error(`✗ Batch write failed: ${error}`);

            // Re-enqueuer les jobs
            for (const job of jobs) {
                await this.cache.lpush(
                    this.writeQueueKey,
                    JSON.stringify(job)
                );
            }

            throw error;
        } finally {
            client.release();
        }
    }

    async _loadUserFromDB(userId) {
        const result = await this.db.query(
            'SELECT id, name, email, created_at FROM users WHERE id = $1',
            [userId]
        );

        if (result.rows.length > 0) {
            const row = result.rows[0];
            return {
                id: row.id,
                name: row.name,
                email: row.email,
                created_at: row.created_at.toISOString()
            };
        }

        return null;
    }

    async forceFlush() {
        console.log('→ Forcing flush of pending writes...');

        while (true) {
            const jobs = await this._popBatch(this.batchSize);
            if (jobs.length === 0) break;
            await this._batchWriteToDB(jobs);
        }

        console.log('✓ All pending writes flushed');
    }

    async shutdown() {
        clearInterval(this.writerInterval);
        await this.forceFlush();
    }
}

// ===== Exemple d'utilisation =====

(async () => {
    const redis = new Redis();
    const db = new Pool({
        host: 'localhost',
        database: 'myapp',
        user: 'postgres'
    });

    const cache = new WriteBackPattern(redis, db);

    // Créer plusieurs utilisateurs (très rapide!)
    for (let i = 0; i < 5; i++) {
        const userId = await cache.createUser({
            name: `User ${i}`,
            email: `user${i}@example.com`
        });
        console.log(`Created user ${userId}`);
    }

    // Les écritures DB se font en arrière-plan
    console.log('\n→ Waiting for background writes...');
    await new Promise(resolve => setTimeout(resolve, 3000));

    // Lecture depuis le cache
    const user = await cache.getUser(1);
    console.log('\nUser from cache:', user);

    // Shutdown graceful
    console.log('\n→ Shutting down...');
    await cache.shutdown();
    console.log('✓ Shutdown complete');

    process.exit(0);
})();
```

### Avantages et inconvénients

**✅ Avantages**
- **Performance maximale** : Écritures sub-milliseconde
- **Débit élevé** : Supporte des milliers d'écritures/sec
- **Batching** : Réduit la charge sur la DB (10x-100x moins de requêtes)
- **Smooth DB load** : Lisse les pics d'écriture

**❌ Inconvénients**
- **Risque de perte** : Si Redis crash, les écritures non persistées sont perdues
- **Complexité** : Worker threads, queues, retry logic
- **Cohérence faible** : Latence entre cache et DB
- **Debugging difficile** : Écritures asynchrones compliquent le troubleshooting

### Quand utiliser Write-Back

```python
# ✅ Cas d'usage idéaux pour Write-Back

# 1. Analytics / Metrics (perte acceptable)
def record_page_view(page_id, user_id):
    """Les vues de page peuvent tolérer une petite perte"""
    cache.incr(f"pageviews:{page_id}")
    enqueue_write('pageviews', page_id, user_id)

# 2. Logging haute fréquence
def log_event(event_type, data):
    """Logs peuvent être batched"""
    enqueue_write('events', event_type, data)

# 3. Leaderboards (eventual consistency OK)
def update_score(user_id, score):
    """Le leaderboard peut avoir quelques secondes de latence"""
    cache.zadd('leaderboard', {user_id: score})
    enqueue_write('scores', user_id, score)

# ❌ NE PAS utiliser Write-Back pour

# 1. Transactions financières (perte inacceptable)
def process_payment(amount):
    # Toujours écrire directement en DB
    db.insert('payments', amount)

# 2. Données critiques (compliance, légal)
def store_gdpr_consent(user_id, consent):
    # Write-Through ou direct DB
    write_through(user_id, consent)

# 3. Données de sécurité
def log_security_event(event):
    # Direct DB avec ACK
    db.insert_with_confirmation('security_log', event)
```

---

## Comparaison finale et guide de choix

### Matrice de décision

| Critère | Cache-Aside | Write-Through | Write-Back |
|---------|-------------|---------------|------------|
| **Read Performance** | ⚡⚡⚡ Excellent | ⚡⚡⚡ Excellent | ⚡⚡⚡ Excellent |
| **Write Performance** | ⚡⚡ Good | ⚡ Average | ⚡⚡⚡ Excellent |
| **Consistency** | 🟡 Eventual | 🟢 Strong | 🔴 Weak |
| **Durability** | 🟢 High | 🟢 High | 🔴 At Risk |
| **Complexity** | 🟢 Simple | 🟡 Medium | 🔴 Complex |
| **DB Protection** | 🟡 Partial | 🟢 Full | 🟢 Full |
| **Resilience** | 🟢 High | 🟡 Medium | 🔴 Low |

### Arbre de décision

```
                         Choisir un Caching Pattern
                                    │
                                    ↓
                    ┌───────────────────────────┐
                    │ Tolérance à la perte ?    │
                    └───────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                 Non                              Oui
                    │                               │
                    ↓                               ↓
        ┌──────────────────────┐       ┌──────────────────────┐
        │ Writes > 1000/sec ?  │       │    Write-Back        │
        └──────────────────────┘       │  (Performance Max)   │
                    │                  └──────────────────────┘
        ┌───────────┴───────────┐
        │                       │
       Non                     Oui
        │                       │
        ↓                       ↓
┌────────────────┐    ┌────────────────────┐
│ Cache-Aside    │    │  Write-Through     │
│ (Standard)     │    │ (High Consistency) │
└────────────────┘    └────────────────────┘
```

### Patterns hybrides recommandés

```python
# Pattern Hybride : Cache-Aside + Write-Through selon le type de donnée

class HybridCachingPattern:
    def __init__(self, redis_client, db):
        self.cache = redis_client
        self.db = db

        # Configurer le pattern par type d'entité
        self.patterns = {
            'user_profile': 'cache_aside',    # Peu d'écritures
            'product_inventory': 'write_through',  # Consistency critique
            'page_views': 'write_back',       # Haute fréquence
            'session': 'cache_aside'          # Lecture intensive
        }

    def get(self, entity_type, entity_id):
        pattern = self.patterns.get(entity_type, 'cache_aside')

        if pattern == 'cache_aside':
            return self._cache_aside_get(entity_type, entity_id)
        else:
            return self._standard_get(entity_type, entity_id)

    def update(self, entity_type, entity_id, data):
        pattern = self.patterns.get(entity_type, 'cache_aside')

        if pattern == 'cache_aside':
            return self._cache_aside_update(entity_type, entity_id, data)
        elif pattern == 'write_through':
            return self._write_through_update(entity_type, entity_id, data)
        elif pattern == 'write_back':
            return self._write_back_update(entity_type, entity_id, data)
```

### Checklist finale

Avant de choisir votre pattern, posez-vous ces questions :

**Cache-Aside**
- ✅ Ratio lecture/écriture > 10:1 ?
- ✅ Tolérance à des données légèrement stale ?
- ✅ Besoin de simplicité ?
- ✅ Résilience critique si Redis tombe ?

**Write-Through**
- ✅ Consistency forte requise ?
- ✅ Ratio lecture/écriture < 5:1 ?
- ✅ Budget latency d'écriture acceptable ?
- ✅ Toutes les données doivent être en cache ?

**Write-Back**
- ✅ Volume d'écritures > 1000/sec ?
- ✅ Latence d'écriture critique (< 1ms) ?
- ✅ Tolérance à une petite perte de données ?
- ✅ Infrastructure pour worker threads ?

---

## Métriques et monitoring

Pour tous les patterns, surveillez :

```python
# Métriques clés à tracker

class CacheMetrics:
    def __init__(self):
        self.reads = 0
        self.writes = 0
        self.cache_hits = 0
        self.cache_misses = 0
        self.write_latencies = []
        self.read_latencies = []

    def report(self):
        hit_rate = (self.cache_hits / (self.cache_hits + self.cache_misses)) * 100
        avg_read_latency = sum(self.read_latencies) / len(self.read_latencies)
        avg_write_latency = sum(self.write_latencies) / len(self.write_latencies)

        return {
            'hit_rate': f"{hit_rate:.2f}%",
            'read_latency_avg': f"{avg_read_latency:.2f}ms",
            'write_latency_avg': f"{avg_write_latency:.2f}ms",
            'total_operations': self.reads + self.writes
        }

# Alertes recommandées
ALERTS = {
    'hit_rate': {
        'threshold': 80,  # %
        'severity': 'warning'
    },
    'write_queue_length': {  # Pour Write-Back
        'threshold': 10000,
        'severity': 'critical'
    },
    'db_sync_lag': {  # Pour Write-Back
        'threshold': 60,  # secondes
        'severity': 'warning'
    }
}
```

---

## Conclusion

Les trois patterns de caching ont chacun leur place dans une architecture moderne :

- **Cache-Aside** : Le choix par défaut pour 80% des cas
- **Write-Through** : Quand la cohérence prime sur la performance
- **Write-Back** : Quand la performance est absolument critique

Le secret est souvent d'utiliser **plusieurs patterns dans la même application**, selon le type de données et les contraintes métier.

---


⏭️ [Stratégies anti-problèmes : Cache Avalanche, Stampede et Penetration](/06-patterns-developpement-avances/02-strategies-anti-problemes.md)
