🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.6 SCAN vs KEYS : Ne jamais bloquer la production

## Introduction

**KEYS est l'une des commandes les plus dangereuses de Redis en production.** Elle peut transformer votre système haute performance en goulot d'étranglement complet en une seule requête. Pourtant, elle reste tentante par sa simplicité apparente : `KEYS user:*` retourne instantanément toutes les clés correspondantes... ou du moins c'est ce qu'on croit.

Cette section explique pourquoi KEYS est dangereuse, comment SCAN résout le problème, et comment l'utiliser correctement dans tous les contextes de production.

## Le problème de KEYS

### Comportement bloquant

```bash
# Commande simple en apparence
KEYS user:*
```

**Ce qui se passe réellement** :

```
Client → KEYS user:*
    ↓
Redis reçoit commande
    ↓
BLOQUE le serveur complet
    ↓
Parcourt TOUT le keyspace (dict principal)
    ↓
Pour chaque clé:
  ├─ Vérifie pattern
  ├─ Ajoute à résultat si match
  └─ Continue
    ↓
Toutes les clés parcourues (10M clés = ~1 seconde)
    ↓
Retourne résultat complet
    ↓
DÉBLOQUE le serveur
```

### Impact en production

**Scénario réel** :

```
Production Redis:
├─ 10 millions de clés
├─ 50,000 requêtes/seconde
└─ Quelqu'un exécute: KEYS *

Résultat:
Time 0ms:    KEYS démarre
Time 10ms:   100,000 requêtes en attente
Time 50ms:   500,000 requêtes en attente (timeout commence)
Time 100ms:  1,000,000 requêtes en attente
Time 500ms:  Clients timeout en masse
Time 1000ms: KEYS termine, retourne 10M clés
Time 1001ms: Avalanche de retries
Time 1005ms: Redis overload, crash possible
```

**Métriques observées** :

```bash
# Avant KEYS
redis-cli INFO stats | grep instantaneous_ops_per_sec
instantaneous_ops_per_sec:50000

# Pendant KEYS (1 seconde)
instantaneous_ops_per_sec:0  # ZERO !

# Après KEYS
instantaneous_ops_per_sec:150000  # Pic de retries
```

### Pourquoi KEYS est O(N)

**Code interne simplifié** :

```c
// redis/src/db.c (simplifié)
void keysCommand(client *c) {
    dictIterator *di;
    dictEntry *de;
    sds pattern = c->argv[1]->ptr;
    int allkeys = (pattern[0] == '*' && patternlen == 1);

    // Itérateur sur TOUT le dictionnaire
    di = dictGetIterator(c->db->dict);

    // Parcourt TOUTES les clés
    while((de = dictNext(di)) != NULL) {
        sds key = dictGetKey(de);
        robj *keyobj;

        // Vérifie pattern
        if (allkeys || stringmatchlen(pattern, patternlen,
                                       key, sdslen(key), 0)) {
            keyobj = createStringObject(key, sdslen(key));

            // Vérifie expiration
            if (!keyIsExpired(c->db, keyobj)) {
                addReplyBulk(c, keyobj);
                numkeys++;
            }
            decrRefCount(keyobj);
        }
    }
    dictReleaseIterator(di);
}
```

**Complexité** : `O(N)` où N = nombre total de clés dans la DB

**Temps d'exécution estimé** :

```
1,000 clés      → ~0.1 ms   ✅ Acceptable (dev)
10,000 clés     → ~1 ms     ⚠️ Limite
100,000 clés    → ~10 ms    ❌ Dangereux
1,000,000 clés  → ~100 ms   ❌ Inacceptable
10,000,000 clés → ~1000 ms  ❌ Catastrophique
```

### Architecture single-threaded

Redis est **single-threaded** pour les commandes :

```
Thread principal Redis:
┌─────────────────────────────┐
│ Boucle événements           │
│                             │
│ while(1) {                  │
│   cmd = getNextCommand()    │
│   execute(cmd)              │ ← BLOQUE ici pendant KEYS
│   sendResponse()            │
│ }                           │
└─────────────────────────────┘

Pendant KEYS:
- Aucune autre commande n'est traitée
- GET, SET, INCR, etc. tous bloqués
- Monitoring bloqué
- Réplication bloquée
```

**Démonstration** :

```bash
# Terminal 1
redis-cli KEYS *
# ... attend 1 seconde

# Terminal 2 (pendant KEYS)
redis-cli PING
# ... timeout après 1 seconde
# (nil)
```

## La solution : SCAN

### Principe fondamental

SCAN est un **itérateur incrémental** qui parcourt le keyspace par petits morceaux :

```
KEYS:
Client → KEYS * → [ATTEND 1s] → [10M clés]

SCAN:
Client → SCAN 0 → [10ms] → [100 clés + cursor]
       → SCAN cursor → [10ms] → [100 clés + cursor]
       → SCAN cursor → [10ms] → [100 clés + cursor]
       ...
       (100,000 itérations, mais non-bloquant)
```

### Garanties de SCAN

SCAN offre les garanties suivantes :

1. **Non bloquant** : Chaque appel est O(1) moyen
2. **Complet** : Toutes les clés présentes du début à la fin sont retournées
3. **Sans état côté serveur** : Le cursor est opaque mais sans état serveur
4. **Pas de duplicatas garantis** : Sauf si clés modifiées pendant scan

**Garanties formelles** :

```
Clé présente de t0 à tN → Retournée AU MOINS 1 fois
Clé ajoutée pendant scan → Peut être retournée ou non
Clé supprimée pendant scan → Peut être retournée ou non
```

### Signature de SCAN

```bash
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
```

**Paramètres** :

- `cursor` : Position dans l'itération (0 pour démarrer)
- `MATCH` : Pattern de filtrage (optionnel)
- `COUNT` : Hint du nombre d'éléments par itération (défaut: 10)
- `TYPE` : Filtre par type de données (Redis 6.0+)

**Retour** :

```bash
SCAN 0
# 1) "17"        ← Nouveau cursor
# 2) 1) "key:1"  ← Clés de cette itération
#    2) "key:2"
#    3) "key:3"
```

## Mécanisme interne de SCAN

### Structure du dictionnaire Redis

Redis utilise une **hash table** avec rehashing incrémental :

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];          // Deux hash tables pour rehashing
    long rehashidx;        // -1 si pas de rehashing en cours
    int iterators;
} dict;

typedef struct dictht {
    dictEntry **table;     // Array de buckets
    unsigned long size;    // Taille de la table (puissance de 2)
    unsigned long sizemask;// size - 1 (pour masquage rapide)
    unsigned long used;    // Nombre d'entrées
} dictht;
```

**Visualisation** :

```
Hash Table (size = 8):
Index   Bucket
  0  →  [key:1] → [key:9] → NULL
  1  →  [key:2] → NULL
  2  →  NULL
  3  →  [key:3] → [key:11] → NULL
  4  →  [key:4] → NULL
  5  →  NULL
  6  →  [key:6] → NULL
  7  →  [key:7] → [key:15] → NULL
```

### Le cursor : Bits inversés

Le cursor de SCAN n'est **pas séquentiel** mais utilise les **bits inversés** :

```
Taille table = 8 (2^3)
Bits nécessaires = 3

Ordre de parcours (bits inversés):
Binary    Decimal   Bucket
000    →  0      →  0
100    →  4      →  4
010    →  2      →  2
110    →  6      →  6
001    →  1      →  1
101    →  5      →  5
011    →  3      →  3
111    →  7      →  7
```

**Pourquoi ?** Permet de gérer le rehashing sans re-scanner :

```
Rehashing de 8 → 16 buckets:

Bucket 0 (000) → se split en:
  - Bucket 0 (0000)
  - Bucket 8 (1000)

Si on a déjà visité 0 avec cursor=000:
  - Prochain cursor = 100 (bucket 4)
  - Bucket 8 (1000) sera visité car 100 < 1000

Pas de duplication ni de miss !
```

### Algorithme de SCAN

**Code simplifié** :

```c
void scanGenericCommand(client *c, robj *o, unsigned long cursor) {
    dictIterator *di;
    dictEntry *de;
    list *keys = listCreate();
    long maxiterations = COUNT_DEFAULT;

    // Itère sur les buckets
    do {
        // Calcule bucket actuel
        unsigned long bucket = cursor & dictht->sizemask;

        // Parcourt la chaîne du bucket
        de = dictht->table[bucket];
        while (de) {
            sds key = dictGetKey(de);

            // Filtre par pattern si nécessaire
            if (pattern == NULL ||
                stringmatchlen(pattern, patternlen, key, sdslen(key), 0)) {
                listAddNodeTail(keys, createStringObject(key, sdslen(key)));
            }

            de = de->next;
            maxiterations--;

            if (maxiterations == 0) break;
        }

        // Prochain cursor (bits inversés)
        cursor = (cursor | (cursor >> 1)) + 1;

    } while (cursor != 0 && maxiterations > 0);

    // Retourne nouveau cursor + clés
    addReplyArrayLen(c, 2);
    addReplyBulkLongLong(c, cursor);
    addReplyArrayLen(c, listLength(keys));
    // ... retourne clés
}
```

### COUNT : Un hint, pas une garantie

Le paramètre `COUNT` est un **hint** :

```bash
SCAN 0 COUNT 100
# Peut retourner:
# - 0 clés (bucket vide)
# - 50 clés (moins que COUNT)
# - 100 clés (COUNT exact)
# - 150 clés (plus que COUNT si longue chaîne)
```

**Comportement** :

```
COUNT contrôle le nombre d'itérations, PAS le nombre de clés:

COUNT 10:
- Parcourt ~10 buckets
- Retourne toutes les clés de ces buckets
- Si bucket a 20 clés → retourne 20 clés

COUNT 100:
- Parcourt ~100 buckets
- Plus de clés retournées en moyenne
- Plus de CPU par appel
- Moins d'appels totaux
```

**Recommandations** :

```bash
# Défaut (petits volumes)
SCAN cursor COUNT 10

# Production (équilibré)
SCAN cursor COUNT 100

# Batch processing (performance)
SCAN cursor COUNT 1000

# Système très chargé (latence min)
SCAN cursor COUNT 5
```

## Utilisation pratique de SCAN

### Itération complète basique

```python
import redis

def scan_all_keys(redis_client):
    """Itère sur toutes les clés"""
    cursor = 0
    keys = []

    while True:
        # SCAN retourne (new_cursor, keys_list)
        cursor, batch = redis_client.scan(cursor, count=100)
        keys.extend(batch)

        # cursor == 0 signifie fin d'itération
        if cursor == 0:
            break

    return keys

# Utilisation
r = redis.Redis()
all_keys = scan_all_keys(r)
print(f"Total keys: {len(all_keys)}")
```

### Avec pattern matching

```python
def scan_pattern(redis_client, pattern, count=100):
    """Scanne clés avec pattern"""
    cursor = 0
    matched_keys = []

    while True:
        cursor, keys = redis_client.scan(cursor, match=pattern, count=count)
        matched_keys.extend(keys)

        if cursor == 0:
            break

    return matched_keys

# Utilisation
user_keys = scan_pattern(r, "user:*")
cache_keys = scan_pattern(r, "cache:api:*")
```

### Generator pattern (économie mémoire)

```python
def scan_keys_generator(redis_client, pattern=None, count=100):
    """Generator pour itération mémoire-efficace"""
    cursor = 0

    while True:
        cursor, keys = redis_client.scan(cursor, match=pattern, count=count)

        # Yield chaque clé individuellement
        for key in keys:
            yield key

        if cursor == 0:
            break

# Utilisation
for key in scan_keys_generator(r, "user:*:profile"):
    profile = r.get(key)
    process(profile)

# Ou avec limite
from itertools import islice
first_1000 = list(islice(scan_keys_generator(r, "cache:*"), 1000))
```

### Traitement par batch

```python
def scan_and_process_batch(redis_client, pattern, processor, batch_size=100):
    """Traite clés par batch"""
    cursor = 0
    total_processed = 0

    while True:
        cursor, keys = redis_client.scan(cursor, match=pattern, count=batch_size)

        if keys:
            # Traite batch
            processor(keys)
            total_processed += len(keys)

        if cursor == 0:
            break

    return total_processed

# Utilisation
def delete_batch(keys):
    if keys:
        r.delete(*keys)

deleted = scan_and_process_batch(r, "temp:*", delete_batch)
print(f"Deleted {deleted} keys")
```

### Avec timeout (protection)

```python
import time

def scan_with_timeout(redis_client, pattern, timeout_seconds=10, count=100):
    """SCAN avec timeout pour éviter boucles infinies"""
    cursor = 0
    keys = []
    start_time = time.time()

    while True:
        # Vérifie timeout
        if time.time() - start_time > timeout_seconds:
            raise TimeoutError(f"SCAN exceeded {timeout_seconds}s timeout")

        cursor, batch = redis_client.scan(cursor, match=pattern, count=count)
        keys.extend(batch)

        if cursor == 0:
            break

    return keys
```

### Parallélisation (avec précautions)

```python
from concurrent.futures import ThreadPoolExecutor

def scan_parallel(redis_client, pattern, num_workers=4):
    """SCAN parallèle (expérimental)"""
    # Note: SCAN n'est pas vraiment parallélisable
    # Mais on peut traiter les résultats en parallèle

    def worker(keys_batch):
        results = []
        for key in keys_batch:
            value = redis_client.get(key)
            results.append((key, value))
        return results

    # SCAN séquentiel (pas d'autre choix)
    cursor = 0
    all_results = []

    with ThreadPoolExecutor(max_workers=num_workers) as executor:
        while True:
            cursor, keys = redis_client.scan(cursor, match=pattern, count=100)

            if keys:
                # Traitement parallèle des clés
                future = executor.submit(worker, keys)
                all_results.append(future)

            if cursor == 0:
                break

    # Collecte résultats
    results = []
    for future in all_results:
        results.extend(future.result())

    return results
```

## SCAN avec MATCH vs filtrage client

### Où se fait le filtrage ?

**MATCH côté serveur** :

```bash
SCAN 0 MATCH user:* COUNT 100
```

**Processus** :

```
1. Redis scanne ~100 buckets
2. Pour chaque clé:
   ├─ Vérifie pattern "user:*"
   ├─ Si match: Ajoute au résultat
   └─ Si non-match: Ignore
3. Retourne uniquement les clés matchées
```

**Filtrage côté client** :

```python
# Récupère toutes les clés
cursor, keys = r.scan(cursor, count=100)

# Filtre côté client
user_keys = [k for k in keys if k.startswith('user:')]
```

### Performance comparée

**Benchmark** :

```python
import time

# Setup: 100k clés, 10k matchent "user:*"
for i in range(100000):
    if i < 10000:
        r.set(f"user:{i}", "data")
    else:
        r.set(f"other:{i}", "data")

# Test 1: MATCH côté serveur
start = time.time()
keys = []
cursor = 0
while True:
    cursor, batch = r.scan(cursor, match="user:*", count=100)
    keys.extend(batch)
    if cursor == 0:
        break
elapsed_server = time.time() - start

# Test 2: Filtrage client
start = time.time()
keys = []
cursor = 0
while True:
    cursor, batch = r.scan(cursor, count=100)
    keys.extend([k for k in batch if k.startswith(b'user:')])
    if cursor == 0:
        break
elapsed_client = time.time() - start

print(f"Server MATCH: {elapsed_server:.2f}s")
print(f"Client filter: {elapsed_client:.2f}s")
```

**Résultats typiques** :

```
100k clés totales, 10k matchent:

Server MATCH: 0.45s
Client filter: 0.52s

Différence: ~15% plus lent côté client

Raison:
- Transfert réseau de 100k clés vs 10k clés
- Parsing 100k réponses vs 10k réponses
```

**Recommandation** : Toujours utiliser `MATCH` côté serveur si possible.

### Limitations de MATCH

MATCH supporte les wildcards simples :

```bash
# Wildcards supportés
SCAN 0 MATCH user:*          # ✅ Préfixe
SCAN 0 MATCH *:profile       # ✅ Suffixe
SCAN 0 MATCH user:*:profile  # ✅ Milieu
SCAN 0 MATCH user:?00:*      # ✅ ? = 1 caractère
SCAN 0 MATCH user:[0-9]*     # ✅ Range

# Limitations
SCAN 0 MATCH user:{123|456}:*  # ❌ Pas de regex
SCAN 0 MATCH ^user.*$          # ❌ Pas d'ancres
```

**Pour patterns complexes** : Filtrer côté client

```python
import re

def scan_with_regex(redis_client, regex_pattern, count=100):
    """SCAN avec regex côté client"""
    pattern = re.compile(regex_pattern)
    cursor = 0
    matched = []

    while True:
        cursor, keys = redis_client.scan(cursor, count=count)

        # Filtre regex côté client
        for key in keys:
            if pattern.match(key.decode() if isinstance(key, bytes) else key):
                matched.append(key)

        if cursor == 0:
            break

    return matched

# Utilisation
# Trouve users avec ID à 3 chiffres exactement
users = scan_with_regex(r, r'^user:\d{3}:profile$')
```

## TYPE filter (Redis 6.0+)

### Filtrage par type de données

```bash
# Scanne uniquement les strings
SCAN 0 TYPE string

# Scanne uniquement les hashes
SCAN 0 TYPE hash

# Scanne uniquement les lists
SCAN 0 TYPE list

# Combine avec MATCH
SCAN 0 MATCH user:* TYPE hash COUNT 100
```

**Types supportés** :

```bash
string
list
set
zset
hash
stream
```

**Exemple pratique** :

```python
def scan_by_type(redis_client, data_type, pattern=None, count=100):
    """SCAN filtré par type"""
    cursor = 0
    keys = []

    while True:
        args = {'cursor': cursor, 'count': count, '_type': data_type}
        if pattern:
            args['match'] = pattern

        cursor, batch = redis_client.scan(**args)
        keys.extend(batch)

        if cursor == 0:
            break

    return keys

# Utilisation
hash_keys = scan_by_type(r, 'hash', 'user:*')
string_keys = scan_by_type(r, 'string', 'cache:*')
```

**Performance** :

```
Sans TYPE:
- Scanne toutes clés
- Application vérifie type avec TYPE command
- 2 requêtes par clé

Avec TYPE:
- Filtre côté serveur
- Retourne uniquement type demandé
- 1 requête par batch
```

## Variantes de SCAN

### SSCAN : Scan de Set

```bash
# SSCAN <key> <cursor> [MATCH pattern] [COUNT count]
SSCAN myset 0 MATCH member:* COUNT 100
```

**Exemple** :

```python
def sscan_all(redis_client, key, pattern=None, count=100):
    """Itère sur tous membres d'un Set"""
    cursor = 0
    members = []

    while True:
        cursor, batch = redis_client.sscan(key, cursor, match=pattern, count=count)
        members.extend(batch)

        if cursor == 0:
            break

    return members

# Utilisation
# Set avec 10M membres
r.sadd('users:active', *range(10000000))

# SSCAN au lieu de SMEMBERS (bloquant)
active_users = sscan_all(r, 'users:active')
```

### HSCAN : Scan de Hash

```bash
# HSCAN <key> <cursor> [MATCH pattern] [COUNT count]
HSCAN myhash 0 MATCH field:* COUNT 100
```

**Retour** : Alternance field/value

```bash
HSCAN user:123 0
# 1) "0"
# 2) 1) "name"
#    2) "Alice"
#    3) "email"
#    4) "alice@example.com"
```

**Exemple** :

```python
def hscan_all(redis_client, key, pattern=None, count=100):
    """Itère sur tous champs d'un Hash"""
    cursor = 0
    items = {}

    while True:
        cursor, data = redis_client.hscan(key, cursor, match=pattern, count=count)
        items.update(data)

        if cursor == 0:
            break

    return items

# Utilisation
# Hash avec 1M champs
for i in range(1000000):
    r.hset('bigdata', f'field:{i}', f'value{i}')

# HSCAN au lieu de HGETALL (bloquant)
data = hscan_all(r, 'bigdata', match='field:100*')
```

### ZSCAN : Scan de Sorted Set

```bash
# ZSCAN <key> <cursor> [MATCH pattern] [COUNT count]
ZSCAN myzset 0 MATCH member:* COUNT 100
```

**Retour** : Alternance member/score

```bash
ZSCAN leaderboard 0
# 1) "0"
# 2) 1) "player1"
#    2) "1000"
#    3) "player2"
#    4) "950"
```

**Exemple** :

```python
def zscan_all(redis_client, key, pattern=None, count=100):
    """Itère sur tous membres d'un Sorted Set"""
    cursor = 0
    items = {}

    while True:
        cursor, data = redis_client.zscan(key, cursor, match=pattern, count=count)
        items.update(data)

        if cursor == 0:
            break

    return items

# Utilisation
zset_data = zscan_all(r, 'leaderboard')
# {b'player1': 1000.0, b'player2': 950.0, ...}
```

## Migration KEYS → SCAN

### Code legacy avec KEYS

```python
# ❌ Code legacy dangereux
def delete_cache_old(redis_client):
    keys = redis_client.keys('cache:*')  # BLOQUE !
    if keys:
        redis_client.delete(*keys)
    return len(keys)
```

### Migration progressive

**Étape 1** : Wrapper avec switch

```python
USE_SCAN = os.getenv('REDIS_USE_SCAN', 'false').lower() == 'true'

def get_keys_by_pattern(redis_client, pattern):
    if USE_SCAN:
        # Nouveau comportement
        cursor = 0
        keys = []
        while True:
            cursor, batch = redis_client.scan(cursor, match=pattern, count=100)
            keys.extend(batch)
            if cursor == 0:
                break
        return keys
    else:
        # Legacy
        return redis_client.keys(pattern)
```

**Étape 2** : Déployer avec flag désactivé

```bash
# Production
REDIS_USE_SCAN=false

# Monitor, pas d'erreur
```

**Étape 3** : Activer progressivement

```bash
# Activer 10% trafic
if hash(user_id) % 10 == 0:
    USE_SCAN = True
else:
    USE_SCAN = False
```

**Étape 4** : Full migration

```bash
# Production
REDIS_USE_SCAN=true

# Supprimer ancien code après stabilisation
```

### Pattern de migration complet

```python
class RedisKeyScanner:
    """Abstraction KEYS/SCAN avec migration progressive"""

    def __init__(self, redis_client, force_scan=None):
        self.redis = redis_client

        if force_scan is None:
            # Lecture config
            force_scan = os.getenv('REDIS_FORCE_SCAN', 'true').lower() == 'true'

        self.use_scan = force_scan

    def get_keys(self, pattern, count=100):
        """Récupère clés par pattern"""
        if self.use_scan:
            return self._scan_keys(pattern, count)
        else:
            return self._keys_legacy(pattern)

    def _scan_keys(self, pattern, count):
        """Implémentation SCAN"""
        cursor = 0
        keys = []

        while True:
            cursor, batch = self.redis.scan(cursor, match=pattern, count=count)
            keys.extend(batch)
            if cursor == 0:
                break

        return keys

    def _keys_legacy(self, pattern):
        """Implémentation legacy KEYS"""
        import warnings
        warnings.warn(
            f"Using deprecated KEYS command for pattern: {pattern}",
            DeprecationWarning
        )
        return self.redis.keys(pattern)

    def delete_by_pattern(self, pattern, batch_size=1000):
        """Supprime clés par pattern (safe)"""
        total_deleted = 0
        cursor = 0

        while True:
            cursor, keys = self.redis.scan(cursor, match=pattern, count=batch_size)

            if keys:
                # Delete batch
                self.redis.delete(*keys)
                total_deleted += len(keys)

            if cursor == 0:
                break

        return total_deleted

# Utilisation
scanner = RedisKeyScanner(r)
cache_keys = scanner.get_keys('cache:*')
deleted = scanner.delete_by_pattern('temp:*')
```

## Performance et benchmarking

### Benchmark KEYS vs SCAN

```python
import time
import redis

def benchmark_keys_vs_scan():
    r = redis.Redis()

    # Setup: Différentes tailles
    sizes = [1000, 10000, 100000, 1000000]

    for size in sizes:
        # Populate
        r.flushdb()
        for i in range(size):
            r.set(f'key:{i}', f'value{i}')

        # Benchmark KEYS
        start = time.time()
        keys_result = r.keys('key:*')
        keys_time = time.time() - start

        # Benchmark SCAN
        start = time.time()
        cursor = 0
        scan_result = []
        while True:
            cursor, batch = r.scan(cursor, match='key:*', count=100)
            scan_result.extend(batch)
            if cursor == 0:
                break
        scan_time = time.time() - start

        print(f"\nSize: {size:,} keys")
        print(f"KEYS:  {keys_time:.4f}s")
        print(f"SCAN:  {scan_time:.4f}s")
        print(f"Ratio: {scan_time/keys_time:.2f}x")
        print(f"Results match: {len(keys_result) == len(scan_result)}")

benchmark_keys_vs_scan()
```

**Résultats typiques** :

```
Size: 1,000 keys
KEYS:  0.0002s
SCAN:  0.0015s  (7.5x plus lent)
Results match: True

Size: 10,000 keys
KEYS:  0.0018s
SCAN:  0.0142s  (7.9x plus lent)
Results match: True

Size: 100,000 keys
KEYS:  0.0167s  ← Commence à être notable
SCAN:  0.1301s  (7.8x plus lent)
Results match: True

Size: 1,000,000 keys
KEYS:  0.1823s  ← BLOQUE 180ms !
SCAN:  1.4156s  (7.8x plus lent)
Results match: True
```

**Analyse** :

- SCAN est ~8x plus lent en temps total
- Mais SCAN ne bloque jamais Redis
- KEYS bloque pendant TOUT le temps
- SCAN permet d'intercaler autres requêtes

### Impact sur throughput

**Test de charge** :

```python
import threading
import time

def load_generator(redis_client, duration=10):
    """Génère charge constante"""
    start = time.time()
    ops = 0

    while time.time() - start < duration:
        redis_client.get('test')
        ops += 1

    return ops

def test_keys_impact():
    r = redis.Redis()

    # Populate 1M keys
    for i in range(1000000):
        r.set(f'key:{i}', f'value{i}')

    # Test baseline (sans KEYS)
    print("Baseline (no KEYS):")
    ops = load_generator(r, duration=10)
    print(f"  {ops:,} ops in 10s = {ops/10:,.0f} ops/sec")

    # Test avec KEYS concurrents
    print("\nWith KEYS every 100ms:")

    def keys_runner():
        for _ in range(100):  # 100x KEYS pendant 10s
            r.keys('key:*')
            time.sleep(0.1)

    keys_thread = threading.Thread(target=keys_runner)
    keys_thread.start()

    ops = load_generator(r, duration=10)
    keys_thread.join()

    print(f"  {ops:,} ops in 10s = {ops/10:,.0f} ops/sec")
    print(f"  Impact: {((ops_baseline - ops) / ops_baseline * 100):.1f}% drop")

test_keys_impact()
```

**Résultats attendus** :

```
Baseline (no KEYS):
  5,000,000 ops in 10s = 500,000 ops/sec

With KEYS every 100ms:
  2,000,000 ops in 10s = 200,000 ops/sec
  Impact: 60% drop
```

### Optimisation de COUNT

**Test de différentes valeurs COUNT** :

```python
def benchmark_count_values():
    r = redis.Redis()

    # 100k clés
    r.flushdb()
    for i in range(100000):
        r.set(f'key:{i}', f'value{i}')

    counts = [10, 50, 100, 500, 1000, 5000]

    for count in counts:
        start = time.time()
        cursor = 0
        total_keys = 0
        iterations = 0

        while True:
            cursor, batch = r.scan(cursor, count=count)
            total_keys += len(batch)
            iterations += 1
            if cursor == 0:
                break

        elapsed = time.time() - start

        print(f"COUNT={count:4d}: {elapsed:.4f}s, "
              f"{iterations:5d} iterations, "
              f"{total_keys:6d} keys")

benchmark_count_values()
```

**Résultats typiques** :

```
COUNT=  10: 2.1456s, 10847 iterations, 100000 keys
COUNT=  50: 0.5234s,  2156 iterations, 100000 keys
COUNT= 100: 0.2891s,  1087 iterations, 100000 keys
COUNT= 500: 0.1234s,   217 iterations, 100000 keys
COUNT=1000: 0.0892s,   109 iterations, 100000 keys
COUNT=5000: 0.0567s,    22 iterations, 100000 keys
```

**Analyse** :

```
COUNT plus élevé:
✅ Moins d'itérations
✅ Moins de round-trips réseau
✅ Plus rapide au total

❌ Plus de travail par itération
❌ Latence légèrement plus élevée par call
❌ Plus de mémoire par batch

Recommandation production: COUNT=100-1000
```

## Cas d'usage en production

### 1. Nettoyage de cache expiré

```python
def cleanup_expired_cache(redis_client, namespace='cache:',
                          batch_size=1000, max_time=60):
    """Nettoie cache en respectant timeout"""
    pattern = f"{namespace}*"
    start_time = time.time()
    total_deleted = 0
    cursor = 0

    while True:
        # Check timeout
        if time.time() - start_time > max_time:
            print(f"Timeout reached, deleted {total_deleted} keys")
            break

        # Scan batch
        cursor, keys = redis_client.scan(cursor, match=pattern, count=batch_size)

        if keys:
            # Vérifie TTL et supprime si expiré ou pas de TTL
            to_delete = []
            for key in keys:
                ttl = redis_client.ttl(key)
                if ttl == -1:  # Pas de TTL
                    to_delete.append(key)

            if to_delete:
                redis_client.delete(*to_delete)
                total_deleted += len(to_delete)

        if cursor == 0:
            break

    return total_deleted
```

### 2. Migration de données

```python
def migrate_keys_to_new_format(redis_client, old_pattern, transformer):
    """Migre clés vers nouveau format"""
    cursor = 0
    migrated = 0
    errors = []

    while True:
        cursor, keys = redis_client.scan(cursor, match=old_pattern, count=100)

        for old_key in keys:
            try:
                # Récupère ancienne valeur
                value = redis_client.get(old_key)
                if value is None:
                    continue

                # Transforme clé et valeur
                new_key, new_value = transformer(old_key, value)

                # Écrit nouveau format
                redis_client.set(new_key, new_value)

                # Optionnel: Supprime ancien
                # redis_client.delete(old_key)

                migrated += 1

            except Exception as e:
                errors.append((old_key, str(e)))

        if cursor == 0:
            break

    return migrated, errors

# Utilisation
def transform_user_key(old_key, old_value):
    # user:123 → user:v2:123
    user_id = old_key.split(':')[1]
    new_key = f"user:v2:{user_id}"

    # Parse et transforme valeur
    data = json.loads(old_value)
    data['version'] = 2
    new_value = json.dumps(data)

    return new_key, new_value

migrated, errors = migrate_keys_to_new_format(r, "user:*", transform_user_key)
print(f"Migrated: {migrated}, Errors: {len(errors)}")
```

### 3. Monitoring et statistiques

```python
def analyze_keyspace(redis_client):
    """Analyse le keyspace"""
    stats = {
        'total_keys': 0,
        'by_type': {},
        'by_namespace': {},
        'with_ttl': 0,
        'without_ttl': 0,
        'total_memory': 0
    }

    cursor = 0
    sample_size = 10000  # Échantillon pour performance
    sampled = 0

    while sampled < sample_size:
        cursor, keys = redis_client.scan(cursor, count=100)

        for key in keys:
            # Type
            key_type = redis_client.type(key)
            stats['by_type'][key_type] = stats['by_type'].get(key_type, 0) + 1

            # Namespace (premier segment)
            namespace = key.split(b':')[0] if b':' in key else b'root'
            stats['by_namespace'][namespace] = stats['by_namespace'].get(namespace, 0) + 1

            # TTL
            ttl = redis_client.ttl(key)
            if ttl > 0:
                stats['with_ttl'] += 1
            else:
                stats['without_ttl'] += 1

            # Memory (échantillon)
            try:
                mem = redis_client.memory_usage(key)
                if mem:
                    stats['total_memory'] += mem
            except:
                pass

            sampled += 1
            if sampled >= sample_size:
                break

        if cursor == 0:
            break

    stats['total_keys'] = sampled
    return stats

# Utilisation
stats = analyze_keyspace(r)
print(f"Total keys sampled: {stats['total_keys']:,}")
print(f"By type: {stats['by_type']}")
print(f"By namespace: {stats['by_namespace']}")
print(f"With TTL: {stats['with_ttl']}, Without: {stats['without_ttl']}")
print(f"Avg memory per key: {stats['total_memory'] / stats['total_keys']:.0f} bytes")
```

### 4. Backup sélectif

```python
def backup_keys_to_file(redis_client, pattern, output_file, format='json'):
    """Backup clés vers fichier"""
    import json

    with open(output_file, 'w') as f:
        cursor = 0
        total = 0

        while True:
            cursor, keys = redis_client.scan(cursor, match=pattern, count=100)

            for key in keys:
                # Récupère type et valeur
                key_type = redis_client.type(key)
                ttl = redis_client.ttl(key)

                if key_type == b'string':
                    value = redis_client.get(key)
                elif key_type == b'hash':
                    value = redis_client.hgetall(key)
                elif key_type == b'list':
                    value = redis_client.lrange(key, 0, -1)
                elif key_type == b'set':
                    value = list(redis_client.smembers(key))
                elif key_type == b'zset':
                    value = redis_client.zrange(key, 0, -1, withscores=True)
                else:
                    continue

                # Sérialise
                entry = {
                    'key': key.decode(),
                    'type': key_type.decode(),
                    'ttl': ttl,
                    'value': value
                }

                json.dump(entry, f)
                f.write('\n')
                total += 1

            if cursor == 0:
                break

        print(f"Backed up {total} keys to {output_file}")

# Utilisation
backup_keys_to_file(r, 'user:*', 'users_backup.jsonl')
```

## Bonnes pratiques

### DO ✅

```python
# ✅ Utiliser SCAN en production
cursor = 0
while True:
    cursor, keys = redis.scan(cursor, match='user:*', count=100)
    process(keys)
    if cursor == 0:
        break

# ✅ Pattern generator pour économie mémoire
for key in scan_keys_generator(r, 'cache:*'):
    process(key)

# ✅ COUNT adapté à la charge
# Charge élevée: COUNT=50
# Charge normale: COUNT=100
# Batch job: COUNT=1000

# ✅ Timeout pour éviter boucles infinies
scan_with_timeout(r, 'temp:*', timeout_seconds=30)

# ✅ MATCH côté serveur
redis.scan(cursor, match='user:*')  # Filtrage serveur

# ✅ TYPE filter si Redis 6.0+
redis.scan(cursor, match='cache:*', _type='string')

# ✅ Suppression par batch
def delete_by_pattern(redis, pattern, batch_size=1000):
    cursor = 0
    while True:
        cursor, keys = redis.scan(cursor, match=pattern, count=batch_size)
        if keys:
            redis.delete(*keys)
        if cursor == 0:
            break
```

### DON'T ❌

```python
# ❌ KEYS en production
keys = redis.keys('user:*')  # BLOQUE !

# ❌ Collecte tous résultats en mémoire si énorme
all_keys = []
cursor = 0
while True:
    cursor, batch = redis.scan(cursor)
    all_keys.extend(batch)  # 10M clés = OOM
    if cursor == 0:
        break

# ❌ Filtrage complexe sans MATCH
cursor, keys = redis.scan(cursor)  # Récupère tout
filtered = [k for k in keys if complex_filter(k)]  # Filtre client

# ❌ COUNT trop petit
redis.scan(cursor, count=1)  # Trop de round-trips

# ❌ COUNT trop grand sur système chargé
redis.scan(cursor, count=100000)  # Latence spike

# ❌ Pas de protection timeout
cursor = 0
while True:  # Peut boucler infiniment si bug
    cursor, keys = redis.scan(cursor)
    if cursor == 0:
        break

# ❌ Ignorer cursor != 0
cursor, keys = redis.scan(0)
# Process keys
# FIN - Incomplet si cursor != 0 !
```

## Monitoring SCAN

### Métriques à surveiller

```python
import time

class ScanMonitor:
    """Monitore performance SCAN"""

    def __init__(self):
        self.metrics = {
            'total_scans': 0,
            'total_iterations': 0,
            'total_keys': 0,
            'total_time': 0,
            'by_pattern': {}
        }

    def monitored_scan(self, redis_client, pattern, count=100):
        """SCAN avec monitoring"""
        start = time.time()
        cursor = 0
        keys = []
        iterations = 0

        while True:
            iter_start = time.time()
            cursor, batch = redis_client.scan(cursor, match=pattern, count=count)
            iter_time = time.time() - iter_start

            keys.extend(batch)
            iterations += 1

            # Log si itération lente
            if iter_time > 0.1:
                print(f"Slow SCAN iteration: {iter_time:.3f}s")

            if cursor == 0:
                break

        elapsed = time.time() - start

        # Update metrics
        self.metrics['total_scans'] += 1
        self.metrics['total_iterations'] += iterations
        self.metrics['total_keys'] += len(keys)
        self.metrics['total_time'] += elapsed

        if pattern not in self.metrics['by_pattern']:
            self.metrics['by_pattern'][pattern] = {
                'count': 0,
                'avg_time': 0,
                'avg_keys': 0
            }

        pattern_stats = self.metrics['by_pattern'][pattern]
        pattern_stats['count'] += 1
        pattern_stats['avg_time'] = (
            (pattern_stats['avg_time'] * (pattern_stats['count'] - 1) + elapsed)
            / pattern_stats['count']
        )
        pattern_stats['avg_keys'] = (
            (pattern_stats['avg_keys'] * (pattern_stats['count'] - 1) + len(keys))
            / pattern_stats['count']
        )

        return keys

    def report(self):
        """Génère rapport"""
        print("SCAN Metrics:")
        print(f"  Total scans: {self.metrics['total_scans']}")
        print(f"  Total iterations: {self.metrics['total_iterations']}")
        print(f"  Total keys: {self.metrics['total_keys']}")
        print(f"  Total time: {self.metrics['total_time']:.2f}s")
        print(f"  Avg time per scan: {self.metrics['total_time'] / max(1, self.metrics['total_scans']):.3f}s")
        print("\nBy pattern:")
        for pattern, stats in self.metrics['by_pattern'].items():
            print(f"  {pattern}:")
            print(f"    Count: {stats['count']}")
            print(f"    Avg time: {stats['avg_time']:.3f}s")
            print(f"    Avg keys: {stats['avg_keys']:.0f}")

# Utilisation
monitor = ScanMonitor()
keys1 = monitor.monitored_scan(r, 'user:*')
keys2 = monitor.monitored_scan(r, 'cache:*')
monitor.report()
```

## Conclusion

### Règles absolues

1. **JAMAIS KEYS en production** : Sans exception
2. **TOUJOURS SCAN** : Pour toute itération
3. **COUNT entre 100-1000** : Balance performance/latence
4. **MATCH côté serveur** : Réduit transfert réseau
5. **Generator pattern** : Économise mémoire
6. **Timeout protection** : Évite boucles infinies
7. **Monitoring** : Mesure performance SCAN

### Récapitulatif SCAN

```bash
# Template production-ready
cursor = 0
total_processed = 0

while True:
    # SCAN avec MATCH et COUNT adapté
    cursor, keys = redis.scan(
        cursor,
        match='namespace:*',
        count=100  # Ajuster selon charge
    )

    # Traite batch
    if keys:
        process_batch(keys)
        total_processed += len(keys)

    # Fin d'itération
    if cursor == 0:
        break

print(f"Processed {total_processed} keys")
```

### Migration checklist

```
☐ Identifier tous usages de KEYS
☐ Remplacer par SCAN avec generator
☐ Ajouter monitoring
☐ Tester en staging
☐ Déployer avec feature flag
☐ Activer progressivement
☐ Supprimer ancien code
☐ Documenter changement
```

SCAN n'est pas qu'une alternative à KEYS : c'est la **seule façon sûre** d'itérer sur le keyspace Redis en production. Le coût de développement supplémentaire est négligeable comparé au risque d'un incident production causé par KEYS.

Cette section conclut le module sur le cycle de vie de la donnée. Vous avez maintenant toutes les clés pour gérer efficacement les données dans Redis, de leur création à leur suppression, en passant par l'expiration, l'éviction et l'exploration du keyspace.

⏭️ [Persistance et fiabilité des données](/05-persistance-fiabilite/README.md)
