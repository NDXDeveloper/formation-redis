🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.9 Complexité algorithmique (Big O) des commandes

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ La notation Big O et son importance pour Redis
- ✅ Les différentes classes de complexité (O(1), O(log N), O(N), etc.)
- ✅ Quelles commandes éviter en production
- ✅ Comment choisir la bonne commande selon le contexte
- ✅ Les techniques d'optimisation et les alternatives sûres

---

## 📘 La notation Big O : Comprendre la performance

### Qu'est-ce que Big O ?

La notation **Big O** décrit comment le **temps d'exécution** d'une commande évolue en fonction de la **taille des données** (N).

**Analogie simple** :
```
Imaginez chercher un nom dans un annuaire téléphonique :

O(1) : Vous connaissez la page exacte → Accès direct
O(log N) : Vous divisez par 2 à chaque fois → Recherche dichotomique
O(N) : Vous lisez page par page → Recherche linéaire
O(N²) : Vous comparez chaque page avec toutes les autres
```

### Pourquoi c'est critique pour Redis ?

Redis est **single-threaded** : une seule commande s'exécute à la fois. Une commande lente **bloque toutes les autres**.

```bash
# Scénario catastrophe
Client 1 : KEYS *           # O(N) avec N = 10 millions de clés
           # Bloque Redis pendant 2 secondes ! 😱

Client 2 : GET cache:key    # Doit attendre 2 secondes
Client 3 : INCR counter     # Doit attendre 2 secondes
Client 4 : SADD set item    # Doit attendre 2 secondes

# Résultat : TOUTE l'application est bloquée !
```

---

## 📊 Les classes de complexité

### O(1) - Temps constant ⚡ SAFE

**Définition** : Le temps d'exécution ne dépend **pas** de la taille des données.

```bash
# Exemples O(1) : TOUJOURS rapides
127.0.0.1:6379> SET mykey "value"     # O(1)
127.0.0.1:6379> GET mykey             # O(1)
127.0.0.1:6379> INCR counter          # O(1)
127.0.0.1:6379> HGET user:123 name    # O(1)
127.0.0.1:6379> LPUSH queue item      # O(1)
127.0.0.1:6379> RPOP queue            # O(1)
127.0.0.1:6379> SADD myset member     # O(1)
127.0.0.1:6379> SISMEMBER myset val   # O(1)
127.0.0.1:6379> ZADD zset 10 member   # O(log N) mais souvent considéré "rapide"

# Performance
# 1 élément : ~0.01 ms
# 1 million d'éléments : ~0.01 ms ← Identique !
```

**✅ Utilisez en production sans crainte.**

### O(log N) - Temps logarithmique ⚡ SAFE

**Définition** : Le temps double quand les données sont multipliées par leur carré.

```bash
# Exemples O(log N) : Très rapides même avec beaucoup de données
127.0.0.1:6379> ZADD leaderboard 1500 "player"   # O(log N)
127.0.0.1:6379> ZRANK leaderboard "player"       # O(log N)
127.0.0.1:6379> ZREM leaderboard "player"        # O(log N)

# Performance approximative
# 1 000 éléments : ~0.01 ms (10 comparaisons)
# 1 000 000 éléments : ~0.02 ms (20 comparaisons)
# 1 000 000 000 éléments : ~0.03 ms (30 comparaisons)

# Croissance très lente !
```

**✅ Utilisez en production sans crainte.**

### O(N) - Temps linéaire ⚠️ DANGER en production

**Définition** : Le temps est proportionnel au nombre d'éléments.

```bash
# Exemples O(N) : Dangereux sur de gros volumes !
127.0.0.1:6379> KEYS *                    # O(N) - N = total de clés
127.0.0.1:6379> SMEMBERS huge:set         # O(N) - N = membres du Set
127.0.0.1:6379> HGETALL huge:hash         # O(N) - N = champs du Hash
127.0.0.1:6379> LRANGE list 0 -1          # O(N) - N = taille de la liste
127.0.0.1:6379> ZRANGE zset 0 -1          # O(log N + N)
127.0.0.1:6379> SUNION set1 set2 set3     # O(N) - N = total membres

# Performance
# 100 éléments : 0.1 ms
# 10 000 éléments : 10 ms
# 1 000 000 éléments : 1000 ms = 1 seconde !! 💥
```

**⚠️ Évitez en production ou limitez N.**

### O(N·M) - Temps polynomial ❌ ÉVITEZ

**Définition** : Le temps dépend du produit de plusieurs tailles.

```bash
# Exemples O(N·M) : Très dangereux !
127.0.0.1:6379> SINTER set1 set2          # O(N·M)
# N = taille du plus petit Set
# M = nombre de Sets

127.0.0.1:6379> ZINTERSTORE dest 5 zset1 zset2 zset3 zset4 zset5
# O(N·K) + O(M·log M)
# Peut bloquer Redis plusieurs secondes

# Performance
# 2 Sets de 1000 éléments : ~10 ms
# 2 Sets de 100 000 éléments : ~10 secondes !! 💥💥
```

**❌ Évitez absolument en production.**

---

## 🔍 Commandes par structure de données

### Strings

| Commande | Complexité | Notes | Production |
|----------|------------|-------|------------|
| `GET` | O(1) | | ✅ Safe |
| `SET` | O(1) | | ✅ Safe |
| `MGET` | O(N) | N = nombre de clés | ✅ Safe si N petit |
| `MSET` | O(N) | N = nombre de clés | ✅ Safe si N petit |
| `INCR/DECR` | O(1) | | ✅ Safe |
| `APPEND` | O(1) | Amortized | ✅ Safe |
| `STRLEN` | O(1) | | ✅ Safe |
| `GETRANGE` | O(N) | N = longueur extraite | ⚠️ Attention |
| `SETRANGE` | O(1) | Si pas de réallocation | ✅ Safe |

**Conseil** :
```bash
# ✅ BON : MGET avec peu de clés
MGET key1 key2 key3 key4 key5  # O(5) = O(1) en pratique

# ⚠️ ATTENTION : MGET avec beaucoup de clés
MGET key1 key2 ... key10000  # O(10000) = peut être lent
```

### Lists

| Commande | Complexité | Notes | Production |
|----------|------------|-------|------------|
| `LPUSH/RPUSH` | O(1) | Par élément | ✅ Safe |
| `LPOP/RPOP` | O(1) | | ✅ Safe |
| `LLEN` | O(1) | | ✅ Safe |
| `LINDEX` | O(N) | N = index | ⚠️ Évitez milieu liste |
| `LRANGE` | O(S+N) | S = offset, N = range | ⚠️ Limitez N |
| `LSET` | O(N) | N = index | ⚠️ Évitez milieu liste |
| `LINSERT` | O(N) | Recherche du pivot | ⚠️ Évitez |
| `LREM` | O(N+M) | N = longueur, M = suppressions | ⚠️ Évitez |
| `LTRIM` | O(N) | N = éléments supprimés | ⚠️ Attention |
| `RPOPLPUSH` | O(1) | | ✅ Safe |

**Exemples** :
```bash
# ✅ SAFE : Opérations aux extrémités
LPUSH mylist "item"    # O(1)
RPOP mylist            # O(1)
LLEN mylist            # O(1)

# ⚠️ DANGER : Accès au milieu
LINDEX mylist 500000   # O(500000) - LENT !
LSET mylist 500000 "value"  # O(500000) - LENT !

# ⚠️ DANGER : Range trop large
LRANGE mylist 0 -1     # Retourne TOUTE la liste !
# ✅ SAFE : Range limitée
LRANGE mylist 0 99     # Seulement 100 éléments
```

### Hashes

| Commande | Complexité | Notes | Production |
|----------|------------|-------|------------|
| `HSET` | O(1) | Par champ | ✅ Safe |
| `HGET` | O(1) | | ✅ Safe |
| `HMGET` | O(N) | N = champs demandés | ✅ Safe si N petit |
| `HGETALL` | O(N) | N = tous les champs | ⚠️ Évitez gros Hash |
| `HDEL` | O(N) | N = champs à supprimer | ✅ Safe si N petit |
| `HEXISTS` | O(1) | | ✅ Safe |
| `HLEN` | O(1) | | ✅ Safe |
| `HKEYS/HVALS` | O(N) | N = nombre de champs | ⚠️ Évitez gros Hash |
| `HINCRBY` | O(1) | | ✅ Safe |

**Exemples** :
```bash
# ✅ SAFE : Opérations sur champs individuels
HGET user:123 email              # O(1)
HSET user:123 city "Paris"       # O(1)
HMGET user:123 name email age    # O(3) = O(1) en pratique

# ⚠️ DANGER : Hash avec 100 000 champs
HGETALL huge:hash  # O(100000) - Bloque Redis !

# ✅ ALTERNATIVE : HSCAN
HSCAN huge:hash 0 COUNT 100
```

### Sets

| Commande | Complexité | Notes | Production |
|----------|------------|-------|------------|
| `SADD` | O(1) | Par membre | ✅ Safe |
| `SREM` | O(N) | N = membres à supprimer | ✅ Safe si N petit |
| `SISMEMBER` | O(1) | | ✅ Safe |
| `SMISMEMBER` | O(N) | N = membres à vérifier | ✅ Safe si N petit |
| `SCARD` | O(1) | | ✅ Safe |
| `SMEMBERS` | O(N) | N = tous les membres | ⚠️ Évitez gros Set |
| `SRANDMEMBER` | O(1) | Sans COUNT | ✅ Safe |
| `SRANDMEMBER` | O(N) | Avec COUNT | ⚠️ Attention |
| `SPOP` | O(1) | Sans COUNT | ✅ Safe |
| `SUNION` | O(N) | N = total membres | ⚠️ Évitez gros Sets |
| `SINTER` | O(N·M) | Produit des tailles | ❌ Évitez |
| `SDIFF` | O(N) | N = total membres | ⚠️ Évitez gros Sets |

**Exemples** :
```bash
# ✅ SAFE
SADD myset "member"         # O(1)
SISMEMBER myset "member"    # O(1)
SCARD myset                 # O(1)

# ⚠️ DANGER : Set avec 1 million de membres
SMEMBERS huge:set  # O(1000000) - Bloque Redis !

# ✅ ALTERNATIVE : SSCAN
SSCAN huge:set 0 COUNT 100

# ❌ DANGER EXTRÊME : Intersection de gros Sets
SINTER set1:100k set2:200k set3:150k  # O(N·M) - CATASTROPHE !

# ✅ ALTERNATIVE : Pré-calculer et mettre en cache
SINTERSTORE cache:result set1 set2 set3
EXPIRE cache:result 300
```

### Sorted Sets

| Commande | Complexité | Notes | Production |
|----------|------------|-------|------------|
| `ZADD` | O(log N) | Par membre | ✅ Safe |
| `ZREM` | O(M·log N) | M = membres à supprimer | ✅ Safe si M petit |
| `ZRANK/ZREVRANK` | O(log N) | | ✅ Safe |
| `ZSCORE` | O(1) | | ✅ Safe |
| `ZCARD` | O(1) | | ✅ Safe |
| `ZRANGE` | O(log N + M) | M = éléments retournés | ⚠️ Limitez M |
| `ZRANGEBYSCORE` | O(log N + M) | M = éléments retournés | ⚠️ Limitez M |
| `ZCOUNT` | O(log N) | | ✅ Safe |
| `ZINCRBY` | O(log N) | | ✅ Safe |
| `ZPOPMIN/ZPOPMAX` | O(log N · M) | M = nombre d'éléments | ✅ Safe si M petit |

**Exemples** :
```bash
# ✅ SAFE : Opérations sur membres individuels
ZADD leaderboard 1500 "player"     # O(log N)
ZSCORE leaderboard "player"         # O(1)
ZRANK leaderboard "player"          # O(log N)

# ✅ SAFE : Range limité
ZRANGE leaderboard 0 9              # Top 10 - O(log N + 10)

# ⚠️ DANGER : Range trop large
ZRANGE leaderboard 0 -1             # TOUS les membres !
# ✅ ALTERNATIVE : ZSCAN ou limiter
ZRANGE leaderboard 0 999            # Max 1000 éléments
```

### HyperLogLog

| Commande | Complexité | Notes | Production |
|----------|------------|-------|------------|
| `PFADD` | O(1) | | ✅ Safe |
| `PFCOUNT` | O(1) | 1 HLL | ✅ Safe |
| `PFCOUNT` | O(N) | N HLLs | ⚠️ Attention si N grand |
| `PFMERGE` | O(N) | N = nombre de HLLs | ⚠️ Attention si N grand |

**Exemples** :
```bash
# ✅ SAFE
PFADD visitors "user:123"    # O(1)
PFCOUNT visitors             # O(1)

# ⚠️ ATTENTION : Union de beaucoup de HLLs
PFCOUNT day1 day2 day3 ... day365  # O(365)
# Acceptable mais peut prendre quelques ms
```

### Bitmaps

| Commande | Complexité | Notes | Production |
|----------|------------|-------|------------|
| `SETBIT` | O(1) | | ✅ Safe |
| `GETBIT` | O(1) | | ✅ Safe |
| `BITCOUNT` | O(N) | N = taille en octets | ⚠️ Attention |
| `BITPOS` | O(N) | N = taille en octets | ⚠️ Attention |
| `BITOP` | O(N) | N = plus grand bitmap | ⚠️ Attention |

**Exemples** :
```bash
# ✅ SAFE
SETBIT active:users 123 1    # O(1)
GETBIT active:users 123      # O(1)

# ⚠️ ATTENTION : Bitmap de 10 MB
BITCOUNT huge:bitmap         # O(10000000 bytes) - Peut être lent

# ✅ ALTERNATIVE : BITCOUNT avec range
BITCOUNT huge:bitmap 0 999   # Seulement 1000 bytes
```

---

## 🚨 Les commandes à ÉVITER en production

### KEYS : L'ennemi public n°1

```bash
# ❌ INTERDIT en production
KEYS *                 # Scanne TOUTES les clés
KEYS user:*            # Scanne TOUTES les clés pour filtrer

# Pourquoi c'est catastrophique :
# - O(N) où N = nombre TOTAL de clés dans la DB
# - Bloque Redis pendant l'exécution
# - Sur 10M de clés : peut prendre 1-2 secondes !

# ✅ ALTERNATIVE : SCAN
SCAN 0 MATCH user:* COUNT 100
# - O(1) par appel
# - Ne bloque pas Redis
# - Itération progressive
```

**Exemple de catastrophe** :
```python
# ❌ Code qui a planté un site e-commerce
def get_all_sessions():
    keys = redis.keys("session:*")  # 500 000 sessions
    # Redis bloqué pendant 800ms
    # 10 000 requêtes en attente
    # Site down !
    return keys

# ✅ Code corrigé
def get_all_sessions():
    cursor = 0
    sessions = []
    while True:
        cursor, keys = redis.scan(cursor, match="session:*", count=100)
        sessions.extend(keys)
        if cursor == 0:
            break
    return sessions
```

### SMEMBERS, HGETALL, LRANGE 0 -1 : Les "bombes à retardement"

```bash
# ⚠️ DANGER : Tout récupérer d'un coup

# Set avec 500 000 membres
SMEMBERS huge:set         # O(500000) - 300ms de blocage

# Hash avec 100 000 champs
HGETALL huge:hash         # O(100000) - 150ms de blocage

# List avec 1 million d'éléments
LRANGE huge:list 0 -1     # O(1000000) - 500ms de blocage

# ✅ ALTERNATIVES : SCAN ou RANGE limité

# Pour Sets
SSCAN huge:set 0 COUNT 1000

# Pour Hashes
HSCAN huge:hash 0 COUNT 1000

# Pour Lists
LRANGE huge:list 0 999    # Seulement 1000 éléments
```

### FLUSHDB, FLUSHALL : Les "boutons rouges"

```bash
# ❌ ATTENTION : Suppression massive
FLUSHDB      # Supprime TOUTE la base courante
FLUSHALL     # Supprime TOUTES les bases (0-15)

# Sur une DB de 10M de clés : peut bloquer 1-2 secondes

# ✅ ALTERNATIVE : Version asynchrone
FLUSHDB ASYNC
FLUSHALL ASYNC
# Redis supprime en arrière-plan, pas de blocage
```

### SINTER avec de gros Sets : Le "tueur silencieux"

```bash
# ❌ CATASTROPHE : Intersection de gros Sets
127.0.0.1:6379> SINTER tags:redis tags:nosql tags:database
# Si chaque Set a 100 000 membres → O(100k × 100k) = 💥

# Symptômes :
# - Redis freeze pendant plusieurs secondes
# - Toutes les autres commandes en attente
# - Timeouts côté client

# ✅ ALTERNATIVES :
# 1. Pré-calculer et cacher
SINTERSTORE cache:result tags:redis tags:nosql tags:database
EXPIRE cache:result 3600

# 2. Utiliser RediSearch pour les requêtes complexes
FT.SEARCH idx "@tags:{redis|nosql|database}"
```

---

## 📏 Guide de décision : Quelle commande utiliser ?

### Scénario 1 : Lister toutes les clés

```bash
# ❌ NE JAMAIS FAIRE
KEYS *

# ✅ FAIRE
cursor = 0
all_keys = []
while True:
    cursor, keys = SCAN(cursor, COUNT=1000)
    all_keys.extend(keys)
    if cursor == 0:
        break
```

### Scénario 2 : Compter les éléments d'une collection

```bash
# ✅ TOUJOURS O(1) : Utilisez les commandes de comptage
SCARD myset        # O(1)
HLEN myhash        # O(1)
LLEN mylist        # O(1)
ZCARD myzset       # O(1)

# ❌ N'utilisez JAMAIS
len(SMEMBERS myset)     # O(N) + transfert réseau
len(HGETALL myhash)     # O(N) + transfert réseau
len(LRANGE list 0 -1)   # O(N) + transfert réseau
```

### Scénario 3 : Vérifier l'existence

```bash
# ✅ O(1) pour les éléments
SISMEMBER myset "member"      # O(1)
HEXISTS myhash "field"        # O(1)

# ⚠️ O(N) pour les clés globales
EXISTS key1 key2 key3         # O(N) mais N petit = OK

# ❌ Ne faites JAMAIS
"key" in KEYS("*")            # O(N) sur toutes les clés !
```

### Scénario 4 : Obtenir une partie des données

```bash
# ✅ Toujours limiter la plage
LRANGE mylist 0 99           # Maximum 100 éléments
ZRANGE myzset 0 9            # Top 10
SSCAN myset 0 COUNT 100      # Par batch

# ❌ Ne jamais tout récupérer
LRANGE mylist 0 -1           # Toute la liste
ZRANGE myzset 0 -1           # Tout le Sorted Set
SMEMBERS myset               # Tout le Set
```

---

## ⏱️ Règles d'or pour la production

### Règle 1 : Commandes < 1ms

**Objectif** : Chaque commande doit s'exécuter en **moins de 1 milliseconde**.

```bash
# ✅ RAPIDES (< 1ms)
GET key                # ~0.01ms
SET key value          # ~0.01ms
HGET hash field        # ~0.01ms
ZADD zset 10 member    # ~0.02ms
SISMEMBER set member   # ~0.01ms

# ⚠️ LENTES (> 1ms) - À surveiller
LRANGE list 0 10000    # ~10ms
ZRANGE zset 0 10000    # ~15ms
HSCAN hash 0 COUNT 10k # ~20ms

# ❌ TRÈS LENTES (> 100ms) - INTERDIT
KEYS *                 # 100-1000ms
SMEMBERS huge:set      # 50-500ms
SINTER big1 big2 big3  # 100-2000ms
```

### Règle 2 : Limiter N dans les commandes O(N)

```bash
# ✅ BON : N limité et connu
LRANGE mylist 0 99                # N = 100
ZRANGE leaderboard 0 9            # N = 10
HSCAN myhash 0 COUNT 100          # N = 100

# ❌ MAUVAIS : N illimité ou inconnu
LRANGE mylist 0 -1                # N = ???
ZRANGE zset 0 -1 WITHSCORES       # N = ???
HGETALL myhash                    # N = ???
```

### Règle 3 : Toujours utiliser SCAN au lieu de KEYS

```bash
# ❌ INTERDIT
KEYS pattern:*
HGETALL huge:hash
SMEMBERS huge:set

# ✅ AUTORISÉ
SCAN cursor MATCH pattern:* COUNT 100
HSCAN hash cursor COUNT 100
SSCAN set cursor COUNT 100
```

### Règle 4 : Pré-calculer les résultats coûteux

```bash
# Si une opération O(N·M) est nécessaire, pré-calculez et cachez

# ❌ Calculer à chaque requête
SINTER tags:redis tags:nosql tags:cache  # O(N·M) à chaque fois

# ✅ Pré-calculer toutes les heures
# Job cron ou task planifiée
SINTERSTORE cache:redis-nosql-cache tags:redis tags:nosql tags:cache
EXPIRE cache:redis-nosql-cache 3600

# Lecture ultra-rapide
SMEMBERS cache:redis-nosql-cache  # O(N) mais result mis en cache
```

### Règle 5 : Monitorer les commandes lentes

```bash
# Activer le slowlog
CONFIG SET slowlog-log-slower-than 10000  # 10ms
CONFIG SET slowlog-max-len 128

# Consulter régulièrement
SLOWLOG GET 10

# Exemple de sortie
1) 1) (integer) 3           # ID
   2) (integer) 1733749200  # Timestamp
   3) (integer) 150000      # Durée en microsecondes (150ms)
   4) 1) "KEYS"
      2) "user:*"
   5) "127.0.0.1:54321"     # Client
   6) ""                    # Client name

# Identifier et corriger les commandes problématiques
```

---

## 🛠️ Optimisations et alternatives

### Optimisation 1 : Pagination

```bash
# ❌ Récupérer tout d'un coup
LRANGE notifications:user:123 0 -1  # Peut-être 10 000 notifications

# ✅ Paginer
def get_notifications(user_id, page=0, page_size=20):
    start = page * page_size
    end = start + page_size - 1
    return redis.lrange(f"notifications:{user_id}", start, end)

# Page 0 : éléments 0-19
# Page 1 : éléments 20-39
# etc.
```

### Optimisation 2 : TTL automatique

```bash
# Limiter la croissance des structures

# ⚠️ Sans TTL : croissance infinie
LPUSH logs:user:123 "log entry"
# Après 1 an : des millions d'entrées !

# ✅ Avec TTL
LPUSH logs:user:123 "log entry"
EXPIRE logs:user:123 86400  # 24 heures

# ✅ Ou LTRIM pour limiter la taille
LPUSH logs:user:123 "log entry"
LTRIM logs:user:123 0 999   # Garder max 1000 logs
```

### Optimisation 3 : Sharding applicatif

```bash
# Si un Set devient trop gros (> 100k membres)

# ❌ Un seul gros Set
SADD users:active user:123
# ... 1 million d'utilisateurs
SMEMBERS users:active  # O(1M) - LENT !

# ✅ Sharding en plusieurs petits Sets
def get_shard(user_id):
    return user_id % 100  # 100 shards

def add_active_user(user_id):
    shard = get_shard(user_id)
    redis.sadd(f"users:active:{shard}", user_id)

# Maintenant chaque Set a ~10 000 membres
# SMEMBERS users:active:42  # O(10k) - Rapide !

# Pour compter le total
def count_all_active():
    total = 0
    for shard in range(100):
        total += redis.scard(f"users:active:{shard}")
    return total
```

### Optimisation 4 : Redis Modules pour requêtes complexes

```bash
# Pour des recherches et agrégations complexes

# ❌ Avec Redis Core : très lent
# Filtrer manuellement avec SCAN + HGETALL + logique application

# ✅ Avec RediSearch (Redis Stack)
FT.CREATE idx ON HASH PREFIX 1 product: SCHEMA
  name TEXT
  price NUMERIC
  category TAG

FT.SEARCH idx "@category:{electronics} @price:[100 500]"
# Index natif + requêtes optimisées
```

---

## 📊 Tableau récapitulatif

### Commandes SAFE pour la production ✅

| Commande | Complexité | Note |
|----------|------------|------|
| GET, SET, INCR | O(1) | Toujours rapide |
| HGET, HSET, HEXISTS | O(1) | Toujours rapide |
| LPUSH, RPOP, LLEN | O(1) | Toujours rapide |
| SADD, SISMEMBER, SCARD | O(1) | Toujours rapide |
| ZADD, ZSCORE, ZCARD | O(1) ou O(log N) | Très rapide |
| SETBIT, GETBIT | O(1) | Toujours rapide |
| PFADD, PFCOUNT (1 HLL) | O(1) | Toujours rapide |
| EXISTS (peu de clés) | O(N) | N petit = OK |
| MGET, MSET (peu de clés) | O(N) | N petit = OK |

### Commandes ATTENTION en production ⚠️

| Commande | Complexité | Limite recommandée |
|----------|------------|--------------------|
| LRANGE | O(N) | N < 1000 |
| ZRANGE, ZREVRANGE | O(log N + M) | M < 1000 |
| HMGET | O(N) | N < 100 |
| SMISMEMBER | O(N) | N < 100 |
| BITCOUNT | O(N) | Avec range |
| SUNION | O(N) | N < 10 000 |
| ZRANGEBYSCORE | O(log N + M) | M < 1000 |

### Commandes INTERDITES en production ❌

| Commande | Complexité | Alternative |
|----------|------------|-------------|
| KEYS | O(N) | SCAN |
| SMEMBERS | O(N) | SSCAN |
| HGETALL | O(N) | HSCAN |
| HKEYS, HVALS | O(N) | HSCAN |
| LRANGE 0 -1 | O(N) | LRANGE avec limite |
| SINTER (gros Sets) | O(N·M) | Pré-calculer |
| FLUSHDB/FLUSHALL | O(N) | Utiliser ASYNC |

---

## 🎓 Points clés à retenir

1. ✅ **Redis est single-threaded** : une commande lente bloque tout
2. ✅ **O(1) et O(log N) sont safe** : utilisez sans crainte
3. ✅ **O(N) en production** : limitez N ou utilisez SCAN
4. ❌ **KEYS est interdit** : utilisez toujours SCAN
5. ❌ **SMEMBERS, HGETALL sur gros volumes** : utilisez SSCAN, HSCAN
6. ⚠️ **LRANGE, ZRANGE** : limitez toujours la plage
7. ⚠️ **SINTER avec gros Sets** : pré-calculez et cachez
8. 📏 **Règle d'or** : chaque commande < 1ms
9. 🔍 **Monitorer** : activez slowlog et surveillez
10. 🎯 **Paginer, limiter, cacher** : les 3 piliers de l'optimisation

---

## 🚀 Conclusion du module

Félicitations ! Vous avez complété le module sur les **structures de données natives** de Redis. Vous maîtrisez maintenant :

- ✅ Les 8 structures principales (Strings, Lists, Hashes, Sets, Sorted Sets, HyperLogLog, Bitmaps)
- ✅ Leurs commandes essentielles et cas d'usage
- ✅ Les implications de performance avec Big O
- ✅ Les bonnes pratiques pour la production

**Prochaine étape** : Explorez les structures étendues de Redis Stack (RedisJSON, RediSearch, etc.) ou plongez dans l'architecture et la persistance.

➡️ **Module suivant** : Module 3 : Structures de données étendues (Redis Stack)

---

**Durée estimée** : 1h30
**Niveau** : Intermédiaire
**Prérequis** : Sections 2.1 à 2.8 complétées

⏭️ [Structures de données étendues (Redis Stack)](/03-structures-donnees-etendues/README.md)
