🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.8 - Tuning et optimisation des commandes

## 🎯 Objectifs de cette section

- Maîtriser la complexité algorithmique des commandes Redis
- Identifier et remplacer les commandes inefficaces
- Optimiser les patterns d'accès aux données
- Utiliser les techniques avancées (pipelining, transactions)
- Mesurer et améliorer les performances
- Adopter les best practices d'optimisation

---

## 📚 Introduction : L'importance de l'optimisation

### Pourquoi optimiser les commandes ?

Redis est rapide, mais **une commande mal choisie peut bloquer toute l'instance**.

```
┌─────────────────────────────────────────────────┐
│  REDIS = SINGLE-THREADED                        │
├─────────────────────────────────────────────────┤
│                                                 │
│  Commande O(1)  : 0.05ms  ✅                    │
│  Commande O(N)  : 50ms    ⚠️                    │
│  Commande O(N²) : 5000ms  ❌ CATASTROPHE        │
│                                                 │
│  Une seule commande lente bloque TOUTES         │
│  les autres requêtes en attente                 │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Impact d'une commande lente** :

```
Exemple : KEYS * sur 10M de clés

Temps d'exécution : 5 secondes
Pendant ce temps :
  - 0 autres commandes exécutées
  - 50,000 requêtes en attente
  - Timeout côté clients
  - Dégradation complète du service
```

### Les 3 piliers de l'optimisation

```
1. COMPLEXITÉ ALGORITHMIQUE
   └─ Choisir des commandes O(1) ou O(log N)

2. VOLUME DE DONNÉES
   └─ Limiter la quantité de données transférées

3. NOMBRE D'APPELS
   └─ Réduire les round-trips réseau
```

---

## 📊 Complexité algorithmique des commandes

### Notation Big O : Rappel

```
O(1)      : Temps constant (idéal)
O(log N)  : Logarithmique (très bon)
O(N)      : Linéaire (acceptable si N petit)
O(N log N): N log N (coûteux)
O(N²)     : Quadratique (ÉVITER)
O(N*M)    : Produit (ÉVITER)
```

### Tableau de complexité par structure

#### Strings

| Commande | Complexité | Notes |
|----------|------------|-------|
| GET | O(1) | ✅ Idéal |
| SET | O(1) | ✅ Idéal |
| INCR/DECR | O(1) | ✅ Atomique |
| GETRANGE | O(N) | N = longueur substring |
| SETRANGE | O(N) | N = longueur à écrire |
| APPEND | O(1) | ✅ Amorti |
| STRLEN | O(1) | ✅ Idéal |

#### Lists

| Commande | Complexité | Notes |
|----------|------------|-------|
| LPUSH/RPUSH | O(1) | ✅ Idéal |
| LPOP/RPOP | O(1) | ✅ Idéal |
| LLEN | O(1) | ✅ Idéal |
| LINDEX | O(N) | N = index (lent au milieu) |
| LRANGE | O(S+N) | S = offset, N = count |
| LINSERT | O(N) | ⚠️ Éviter si possible |
| LSET | O(N) | ⚠️ Éviter si possible |
| LTRIM | O(N) | N = éléments supprimés |

#### Sets

| Commande | Complexité | Notes |
|----------|------------|-------|
| SADD | O(1) | ✅ Par élément |
| SREM | O(N) | N = nombre d'éléments |
| SISMEMBER | O(1) | ✅ Idéal |
| SCARD | O(1) | ✅ Idéal |
| SMEMBERS | O(N) | ❌ ÉVITER (retourne tout) |
| SSCAN | O(1) | ✅ Par itération |
| SINTER | O(N*M) | N = plus petit set |
| SUNION | O(N) | N = somme tous sets |
| SDIFF | O(N) | N = somme tous sets |

#### Hashes

| Commande | Complexité | Notes |
|----------|------------|-------|
| HGET | O(1) | ✅ Idéal |
| HSET | O(1) | ✅ Idéal |
| HMGET | O(N) | N = nombre de fields |
| HMSET | O(N) | N = nombre de fields |
| HLEN | O(1) | ✅ Idéal |
| HGETALL | O(N) | ❌ ÉVITER (retourne tout) |
| HSCAN | O(1) | ✅ Par itération |
| HKEYS/HVALS | O(N) | ⚠️ Retourne tout |
| HINCRBY | O(1) | ✅ Atomique |

#### Sorted Sets

| Commande | Complexité | Notes |
|----------|------------|-------|
| ZADD | O(log N) | ✅ Très bon |
| ZREM | O(M log N) | M = éléments à supprimer |
| ZSCORE | O(1) | ✅ Idéal |
| ZRANK | O(log N) | ✅ Très bon |
| ZRANGE | O(log N + M) | M = éléments retournés |
| ZRANGEBYSCORE | O(log N + M) | M = éléments retournés |
| ZCARD | O(1) | ✅ Idéal |
| ZCOUNT | O(log N) | ✅ Très bon |
| ZINCRBY | O(log N) | ✅ Très bon |
| ZUNIONSTORE | O(N + M log M) | Coûteux |
| ZINTERSTORE | O(N*M log M) | Très coûteux |

#### Commandes générales

| Commande | Complexité | Notes |
|----------|------------|-------|
| KEYS | O(N) | ❌ JAMAIS en production |
| SCAN | O(1) | ✅ Par itération |
| EXISTS | O(N) | N = nombre de clés |
| DEL | O(N) | N = nombre de clés |
| UNLINK | O(1) | ✅ Async (mieux que DEL) |
| RENAME | O(1) | ✅ Idéal |
| TYPE | O(1) | ✅ Idéal |
| EXPIRE | O(1) | ✅ Idéal |
| TTL | O(1) | ✅ Idéal |

---

## 🚫 Commandes à éviter en production

### La liste noire

#### 1. KEYS *

```bash
# ❌ JAMAIS CELA
KEYS *
KEYS user:*
KEYS cache:*

# Pourquoi c'est catastrophique :
# - O(N) où N = total de clés dans la DB
# - Bloque Redis pendant toute l'exécution
# - Peut prendre plusieurs secondes sur 10M+ clés
# - Aucune possibilité d'interrompre

# ✅ TOUJOURS CELA
SCAN 0 MATCH user:* COUNT 100
# - O(1) par itération
# - Non-bloquant
# - Peut être pausé/repris
```

#### 2. SMEMBERS sur gros sets

```bash
# ❌ Éviter
SMEMBERS large_set
# Si le set a 100K membres → 100K éléments retournés d'un coup

# ✅ Utiliser SSCAN
SSCAN large_set 0 COUNT 100
# Retourne 100 éléments à la fois
```

#### 3. HGETALL sur gros hashes

```bash
# ❌ Éviter
HGETALL user:12345:data
# Si le hash a 50K fields → 50K fields retournés

# ✅ Alternative 1 : Récupérer seulement les champs nécessaires
HMGET user:12345:data field1 field2 field3

# ✅ Alternative 2 : Utiliser HSCAN
HSCAN user:12345:data 0 COUNT 100
```

#### 4. LRANGE 0 -1 sur grosses listes

```bash
# ❌ Éviter
LRANGE queue:tasks 0 -1
# Récupère toute la liste

# ✅ Paginer
LRANGE queue:tasks 0 99    # Premiers 100
LRANGE queue:tasks 100 199 # 100 suivants
```

#### 5. SORT avec patterns complexes

```bash
# ❌ Très coûteux
SORT mylist BY weight:* GET object:* GET name:*
# O(N log N) + accès à d'autres clés

# ✅ Pré-calculer et utiliser Sorted Sets
ZADD sorted_list score1 "element1" score2 "element2"
ZRANGE sorted_list 0 -1 WITHSCORES
```

#### 6. Opérations ensemblistes sur gros sets

```bash
# ❌ Coûteux
SUNION set1 set2 set3 set4 set5
# O(N) où N = somme de tous les sets

# ❌ Très coûteux
SINTER large_set1 large_set2 large_set3
# O(N*M) où N = plus petit set, M = nombre de sets

# ✅ Limiter le nombre de sets
# ✅ Précalculer avec SUNIONSTORE si résultat utilisé multiple fois
SUNIONSTORE result set1 set2 set3
# Puis récupérer par parties avec SSCAN
```

---

## ✅ Alternatives optimisées

### Pattern 1 : Remplacer KEYS par SCAN

```python
#!/usr/bin/env python3
"""
Scanner efficacement le keyspace
"""
import redis

def scan_keys_efficient(r, pattern='*', count=100):
    """
    Alternative à KEYS - utilise SCAN
    """
    cursor = 0
    keys_found = []

    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=count)
        keys_found.extend(keys)

        if cursor == 0:
            break

    return keys_found

# Usage
r = redis.Redis()
user_keys = scan_keys_efficient(r, pattern='user:*', count=1000)
print(f"Found {len(user_keys)} user keys")
```

### Pattern 2 : Remplacer SMEMBERS par SSCAN

```python
def get_set_members_efficient(r, key, batch_size=100):
    """
    Alternative à SMEMBERS - utilise SSCAN
    """
    cursor = 0
    members = []

    while True:
        cursor, batch = r.sscan(key, cursor, count=batch_size)
        members.extend(batch)

        if cursor == 0:
            break

    return members

# Usage
members = get_set_members_efficient(r, 'large_set', batch_size=500)
```

### Pattern 3 : Remplacer HGETALL par HMGET

```python
# ❌ Récupérer tout
data = r.hgetall('user:12345')

# ✅ Récupérer seulement ce qui est nécessaire
data = r.hmget('user:12345', ['name', 'email', 'age', 'city'])

# ✅ Si vraiment besoin de tout, utiliser HSCAN
def get_hash_efficient(r, key, batch_size=100):
    cursor = 0
    hash_data = {}

    while True:
        cursor, data = r.hscan(key, cursor, count=batch_size)
        hash_data.update(data)

        if cursor == 0:
            break

    return hash_data
```

### Pattern 4 : Pré-calcul avec Sorted Sets

```python
# Exemple : Leaderboard avec scores calculés

# ❌ Mauvais : Calculer à chaque fois
def get_leaderboard_slow(r):
    users = r.smembers('users')
    leaderboard = []

    for user in users:
        score = calculate_complex_score(r, user)  # Coûteux!
        leaderboard.append((user, score))

    leaderboard.sort(key=lambda x: x[1], reverse=True)
    return leaderboard[:10]

# ✅ Bon : Pré-calculer dans un Sorted Set
def update_leaderboard(r, user):
    score = calculate_complex_score(r, user)
    r.zadd('leaderboard', {user: score})

def get_leaderboard_fast(r):
    return r.zrevrange('leaderboard', 0, 9, withscores=True)
```

### Pattern 5 : Batch operations

```python
# ❌ Appels individuels
for i in range(1000):
    r.set(f'key:{i}', f'value:{i}')
# 1000 round-trips réseau!

# ✅ Utiliser pipeline
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
# 1 seul round-trip!

# ✅ Ou MSET pour les strings
mapping = {f'key:{i}': f'value:{i}' for i in range(1000)}
r.mset(mapping)
```

---

## 🚀 Techniques d'optimisation avancées

### 1. Pipelining

**Principe** : Envoyer plusieurs commandes d'un coup sans attendre les réponses individuelles.

```python
import redis
import time

r = redis.Redis()

# Sans pipeline
start = time.time()
for i in range(10000):
    r.set(f'key:{i}', f'value:{i}')
    r.get(f'key:{i}')
duration_no_pipeline = time.time() - start

# Avec pipeline
start = time.time()
pipe = r.pipeline()
for i in range(10000):
    pipe.set(f'key:{i}', f'value:{i}')
    pipe.get(f'key:{i}')
pipe.execute()
duration_pipeline = time.time() - start

print(f"Sans pipeline: {duration_no_pipeline:.2f}s")
print(f"Avec pipeline: {duration_pipeline:.2f}s")
print(f"Speedup: {duration_no_pipeline/duration_pipeline:.1f}x")

# Résultat typique :
# Sans pipeline: 12.34s
# Avec pipeline: 0.45s
# Speedup: 27.4x
```

**Quand utiliser** :
- ✅ Batch d'écritures
- ✅ Batch de lectures indépendantes
- ✅ Mix lectures/écritures sans dépendances
- ❌ Commandes dépendantes (utiliser transactions)

### 2. Transactions avec MULTI/EXEC

```python
# Pour des opérations atomiques dépendantes

def transfer_points(r, from_user, to_user, points):
    """
    Transfert atomique de points entre utilisateurs
    """
    pipe = r.pipeline()

    # WATCH pour optimistic locking
    pipe.watch(f'user:{from_user}:points')

    # Vérifier le solde
    balance = int(pipe.get(f'user:{from_user}:points') or 0)

    if balance < points:
        pipe.unwatch()
        return False

    # Transaction atomique
    pipe.multi()
    pipe.decrby(f'user:{from_user}:points', points)
    pipe.incrby(f'user:{to_user}:points', points)

    try:
        pipe.execute()
        return True
    except redis.WatchError:
        # Retry si conflit
        return transfer_points(r, from_user, to_user, points)
```

### 3. Lua Scripts pour atomicité complexe

```python
# Script Lua pour opération atomique complexe
lua_script = """
local key = KEYS[1]
local value = redis.call('GET', key)

if not value then
    return 0
end

local num = tonumber(value)
if num < 100 then
    redis.call('INCR', key)
    return 1
else
    return -1
end
"""

# Charger le script
increment_if_less_than_100 = r.register_script(lua_script)

# Utiliser
result = increment_if_less_than_100(keys=['counter'])
```

**Avantages Lua** :
- Atomicité garantie
- Pas de round-trips réseau
- Logic complexe côté serveur
- Performance optimale

### 4. Compression des données

```python
import json
import gzip
import base64

def set_compressed(r, key, data, ttl=None):
    """
    Stocker des données compressées
    """
    # Sérialiser
    json_data = json.dumps(data)

    # Compresser
    compressed = gzip.compress(json_data.encode())

    # Encoder en base64 pour stockage
    encoded = base64.b64encode(compressed)

    # Stocker
    if ttl:
        r.setex(key, ttl, encoded)
    else:
        r.set(key, encoded)

def get_compressed(r, key):
    """
    Récupérer des données compressées
    """
    encoded = r.get(key)
    if not encoded:
        return None

    # Décoder
    compressed = base64.b64decode(encoded)

    # Décompresser
    json_data = gzip.decompress(compressed).decode()

    # Désérialiser
    return json.loads(json_data)

# Usage
large_data = {'users': [{'name': f'user{i}', 'data': 'x'*1000} for i in range(100)]}

# Comparaison taille
import sys
normal_size = sys.getsizeof(json.dumps(large_data))
compressed_size = sys.getsizeof(gzip.compress(json.dumps(large_data).encode()))

print(f"Normal: {normal_size} bytes")
print(f"Compressed: {compressed_size} bytes")
print(f"Ratio: {normal_size/compressed_size:.1f}x")
```

### 5. Client-side caching (Redis 6+)

```python
import redis

# Activer le client-side caching
r = redis.Redis(decode_responses=True)

# Tracking activé
r.client_tracking_on()

# Les GET suivants seront cachés côté client
value = r.get('frequently_accessed_key')

# Redis notifiera automatiquement si la clé change
# Le cache client sera invalidé

# Désactiver si nécessaire
r.client_tracking_off()
```

---

## 📈 Profiling et mesure des performances

### Script de profiling complet

```python
#!/usr/bin/env python3
"""
Profiling avancé des commandes Redis
"""
import redis
import time
import statistics
from collections import defaultdict

class RedisCommandProfiler:
    def __init__(self, host='localhost', port=6379):
        self.r = redis.Redis(host=host, port=port, decode_responses=True)
        self.results = defaultdict(list)

    def profile_command(self, name, func, iterations=1000):
        """
        Profile une commande spécifique
        """
        print(f"Profiling {name}...")

        latencies = []

        for i in range(iterations):
            start = time.perf_counter()
            try:
                func(i)
            except Exception as e:
                print(f"  Error: {e}")
                continue
            end = time.perf_counter()

            latencies.append((end - start) * 1000)  # ms

        self.results[name] = latencies

    def report(self):
        """
        Génère un rapport détaillé
        """
        print("\n" + "=" * 80)
        print("REDIS COMMAND PROFILING REPORT")
        print("=" * 80)

        # Tableau de résultats
        print(f"\n{'Command':<30} {'Avg (ms)':<12} {'P50':<12} {'P95':<12} {'P99':<12} {'Max':<12}")
        print("-" * 80)

        # Trier par latence moyenne
        sorted_results = sorted(
            self.results.items(),
            key=lambda x: statistics.mean(x[1]),
            reverse=True
        )

        for name, latencies in sorted_results:
            avg = statistics.mean(latencies)
            p50 = statistics.median(latencies)
            p95 = sorted(latencies)[int(len(latencies) * 0.95)]
            p99 = sorted(latencies)[int(len(latencies) * 0.99)]
            max_lat = max(latencies)

            # Indicateur de performance
            if avg < 0.1:
                indicator = "🟢"
            elif avg < 1:
                indicator = "🟡"
            else:
                indicator = "🔴"

            print(f"{name:<30} {avg:<12.3f} {p50:<12.3f} {p95:<12.3f} {p99:<12.3f} {max_lat:<12.3f} {indicator}")

        print("-" * 80)

        # Recommandations
        print("\n" + "=" * 80)
        print("RECOMMENDATIONS")
        print("=" * 80)

        for name, latencies in sorted_results:
            avg = statistics.mean(latencies)

            if avg > 10:
                print(f"\n⚠️  {name}")
                print(f"   Average latency: {avg:.2f}ms")
                print(f"   Action: This command is very slow. Consider:")
                print(f"   - Using alternative commands (SCAN instead of KEYS)")
                print(f"   - Reducing data size")
                print(f"   - Using pipelining if possible")

            elif avg > 1:
                print(f"\nℹ️  {name}")
                print(f"   Average latency: {avg:.2f}ms")
                print(f"   Suggestion: Monitor this command, consider optimization")

# Exemple d'utilisation
def run_profiling():
    profiler = RedisCommandProfiler()

    # Setup : créer des données de test
    r = profiler.r

    # Créer une list
    r.delete('test:list')
    for i in range(10000):
        r.rpush('test:list', f'item{i}')

    # Créer un hash
    r.delete('test:hash')
    for i in range(10000):
        r.hset('test:hash', f'field{i}', f'value{i}')

    # Créer un set
    r.delete('test:set')
    for i in range(10000):
        r.sadd('test:set', f'member{i}')

    # Créer un sorted set
    r.delete('test:zset')
    for i in range(10000):
        r.zadd('test:zset', {f'member{i}': i})

    # Profile différentes commandes

    # Strings
    profiler.profile_command(
        'GET (simple)',
        lambda i: r.get(f'test:key:{i}')
    )

    profiler.profile_command(
        'SET (simple)',
        lambda i: r.set(f'test:key:{i}', f'value{i}')
    )

    # Lists
    profiler.profile_command(
        'LPUSH',
        lambda i: r.lpush('test:list:temp', f'item{i}')
    )

    profiler.profile_command(
        'LRANGE (first 10)',
        lambda i: r.lrange('test:list', 0, 9)
    )

    profiler.profile_command(
        'LRANGE (first 100)',
        lambda i: r.lrange('test:list', 0, 99)
    )

    profiler.profile_command(
        'LRANGE (all 10K)',
        lambda i: r.lrange('test:list', 0, -1),
        iterations=100  # Moins d'itérations car lent
    )

    # Hashes
    profiler.profile_command(
        'HGET',
        lambda i: r.hget('test:hash', f'field{i%10000}')
    )

    profiler.profile_command(
        'HMGET (10 fields)',
        lambda i: r.hmget('test:hash', [f'field{j}' for j in range(10)])
    )

    profiler.profile_command(
        'HGETALL (10K fields)',
        lambda i: r.hgetall('test:hash'),
        iterations=100
    )

    # Sets
    profiler.profile_command(
        'SISMEMBER',
        lambda i: r.sismember('test:set', f'member{i%10000}')
    )

    profiler.profile_command(
        'SMEMBERS (10K members)',
        lambda i: r.smembers('test:set'),
        iterations=100
    )

    # Sorted Sets
    profiler.profile_command(
        'ZSCORE',
        lambda i: r.zscore('test:zset', f'member{i%10000}')
    )

    profiler.profile_command(
        'ZRANGE (first 10)',
        lambda i: r.zrange('test:zset', 0, 9)
    )

    profiler.profile_command(
        'ZRANGE (first 100)',
        lambda i: r.zrange('test:zset', 0, 99)
    )

    # Scan operations
    profiler.profile_command(
        'SSCAN (100 at a time)',
        lambda i: r.sscan('test:set', 0, count=100)
    )

    profiler.profile_command(
        'HSCAN (100 at a time)',
        lambda i: r.hscan('test:hash', 0, count=100)
    )

    # Pipeline
    def pipeline_test(i):
        pipe = r.pipeline()
        for j in range(10):
            pipe.get(f'test:key:{j}')
        pipe.execute()

    profiler.profile_command(
        'Pipeline (10 GET)',
        pipeline_test
    )

    # Rapport
    profiler.report()

    # Cleanup
    r.delete('test:list', 'test:list:temp', 'test:hash', 'test:set', 'test:zset')

if __name__ == "__main__":
    run_profiling()
```

---

## 🎯 Optimisation par use case

### Use Case 1 : Cache applicatif

```python
# ❌ Anti-pattern : Cache inefficace
def get_user_slow(user_id):
    user = r.get(f'user:{user_id}')
    if not user:
        user = db.query_user(user_id)  # DB query
        r.set(f'user:{user_id}', json.dumps(user))  # Cache sans TTL!
    return json.loads(user)

# ✅ Pattern optimisé
def get_user_optimized(user_id):
    # 1. Cache avec TTL
    user = r.get(f'user:{user_id}')

    if not user:
        user = db.query_user(user_id)
        # TTL pour éviter stale data
        r.setex(f'user:{user_id}', 3600, json.dumps(user))
    else:
        user = json.loads(user)

    return user

# ✅✅ Pattern avancé avec compression
def get_user_advanced(user_id):
    # Cache L1 : mémoire locale (très rapide)
    if user_id in local_cache:
        if time.time() - local_cache[user_id]['time'] < 60:  # 1 min
            return local_cache[user_id]['data']

    # Cache L2 : Redis (rapide)
    user_compressed = r.get(f'user:{user_id}')

    if user_compressed:
        user = decompress(user_compressed)
    else:
        # Cache miss : DB
        user = db.query_user(user_id)
        r.setex(f'user:{user_id}', 3600, compress(user))

    # Mettre en cache local
    local_cache[user_id] = {'data': user, 'time': time.time()}

    return user
```

### Use Case 2 : Compteurs temps réel

```python
# ❌ Anti-pattern : Incréments individuels
def track_pageview_slow(page_id):
    r.incr(f'pageviews:{page_id}')
    r.incr(f'pageviews:{page_id}:today')
    r.incr(f'pageviews:total')
# 3 round-trips réseau!

# ✅ Pattern optimisé : Pipeline
def track_pageview_optimized(page_id):
    pipe = r.pipeline()
    pipe.incr(f'pageviews:{page_id}')
    pipe.incr(f'pageviews:{page_id}:today')
    pipe.incr(f'pageviews:total')
    pipe.execute()
# 1 seul round-trip!

# ✅✅ Pattern avancé : Batch local puis flush
class PageviewTracker:
    def __init__(self, r, flush_interval=1.0):
        self.r = r
        self.buffer = defaultdict(int)
        self.flush_interval = flush_interval
        self.last_flush = time.time()

    def track(self, page_id):
        # Incrémenter en mémoire
        self.buffer[f'pageviews:{page_id}'] += 1
        self.buffer[f'pageviews:total'] += 1

        # Flush si nécessaire
        if time.time() - self.last_flush > self.flush_interval:
            self.flush()

    def flush(self):
        if not self.buffer:
            return

        # Flush en batch vers Redis
        pipe = self.r.pipeline()
        for key, count in self.buffer.items():
            pipe.incrby(key, count)
        pipe.execute()

        # Reset
        self.buffer.clear()
        self.last_flush = time.time()

# Usage
tracker = PageviewTracker(r)
for i in range(10000):
    tracker.track(f'page_{i % 100}')
tracker.flush()  # Flush final
```

### Use Case 3 : Leaderboard

```python
# ❌ Anti-pattern : Calcul à chaque lecture
def get_leaderboard_slow():
    users = r.smembers('users')
    scores = []

    for user in users:
        # Calcul du score (coûteux!)
        score = calculate_score(r, user)
        scores.append((user, score))

    scores.sort(key=lambda x: x[1], reverse=True)
    return scores[:10]

# ✅ Pattern optimisé : Sorted Set
def update_user_score(user_id, score):
    r.zadd('leaderboard', {user_id: score})

def get_leaderboard_optimized():
    # O(log N + M) où M = 10
    return r.zrevrange('leaderboard', 0, 9, withscores=True)

# ✅✅ Pattern avancé : Multiple leaderboards
def update_user_score_advanced(user_id, score, country):
    pipe = r.pipeline()

    # Global leaderboard
    pipe.zadd('leaderboard:global', {user_id: score})

    # Country leaderboard
    pipe.zadd(f'leaderboard:country:{country}', {user_id: score})

    # Time-based leaderboards
    today = datetime.now().strftime('%Y-%m-%d')
    pipe.zadd(f'leaderboard:daily:{today}', {user_id: score})

    pipe.execute()

def get_leaderboard_with_rank(user_id):
    """Récupère top 10 + rang de l'utilisateur"""
    pipe = r.pipeline()

    # Top 10
    pipe.zrevrange('leaderboard:global', 0, 9, withscores=True)

    # Rang de l'utilisateur
    pipe.zrevrank('leaderboard:global', user_id)

    # Score de l'utilisateur
    pipe.zscore('leaderboard:global', user_id)

    results = pipe.execute()

    return {
        'top_10': results[0],
        'user_rank': results[1] + 1 if results[1] is not None else None,
        'user_score': results[2]
    }
```

---

## 📋 Best Practices et checklist

### Checklist d'optimisation

**Avant le déploiement** :
- [ ] Toutes les commandes ont une complexité ≤ O(log N)
- [ ] Pas de KEYS, SMEMBERS, HGETALL en production
- [ ] Pipelining utilisé pour les batch operations
- [ ] TTL définis sur toutes les données temporaires
- [ ] Compression activée pour les gros objets
- [ ] Monitoring des commandes lentes (SLOWLOG)

**Choix de structures** :
- [ ] Strings pour les valeurs simples (< 100KB)
- [ ] Hashes pour les objets (au lieu de JSON strings)
- [ ] Lists pour les queues (pas pour l'accès aléatoire)
- [ ] Sets pour l'unicité
- [ ] Sorted Sets pour le classement
- [ ] Éviter les big keys (fragmenter si > 10K éléments)

**Patterns d'accès** :
- [ ] SCAN au lieu de KEYS
- [ ] SSCAN/HSCAN au lieu de SMEMBERS/HGETALL
- [ ] HMGET au lieu de HGETALL quand possible
- [ ] Pipelining pour les batch operations
- [ ] Lua scripts pour la logique complexe atomique

### Règles d'or

```
1. Privilégier O(1) et O(log N)
   └─ Éviter O(N), O(N²), O(N*M)

2. Minimiser les round-trips
   └─ Pipeline, MGET, HMGET

3. Ne jamais bloquer Redis
   └─ Pas de KEYS, utiliser SCAN

4. Fragmenter les big keys
   └─ Taille max recommandée : 10K éléments

5. Toujours définir des TTL
   └─ Éviter les memory leaks

6. Mesurer avant d'optimiser
   └─ SLOWLOG, profiling, benchmarks

7. Cache intelligemment
   └─ TTL appropriés, compression si gros

8. Penser atomicité
   └─ Transactions, Lua scripts quand nécessaire
```

### Anti-patterns à éviter

```
❌ KEYS * en production
❌ SMEMBERS sur gros sets
❌ HGETALL sur gros hashes
❌ Boucles de commandes individuelles
❌ Big keys (> 10K éléments)
❌ Pas de TTL sur données temporaires
❌ JSON strings au lieu de hashes
❌ LRANGE 0 -1 sur grosses listes
❌ SORT avec patterns complexes
❌ Multiples SINTER/SUNION sans cache
```

---

## 🎯 Points clés à retenir

1. **Complexité algorithmique** → Toujours vérifier le Big O
2. **SCAN > KEYS** → Jamais KEYS en production
3. **Pipeline** → Réduire les round-trips réseau
4. **Lua scripts** → Atomicité et performance
5. **Fragmenter big keys** → Max 10K éléments
6. **TTL partout** → Éviter memory leaks
7. **Profiler** → SLOWLOG et benchmarks
8. **Structures appropriées** → Hash > JSON string

---

**🚀 Section suivante** : [14.9 - Benchmarking avec redis-benchmark](./09-benchmarking-redis-benchmark.md)

⏭️ [Benchmarking avec redis-benchmark](/14-performance-troubleshooting/09-benchmarking-redis-benchmark.md)
