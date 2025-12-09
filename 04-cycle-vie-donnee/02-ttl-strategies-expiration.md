🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2 TTL (Time To Live) et stratégies d'expiration

## Introduction

Le TTL (Time To Live) est l'un des mécanismes les plus puissants de Redis. Il permet de définir une **durée de vie automatique** pour les données, transformant Redis en un système de cache auto-nettoyant. Contrairement aux bases de données traditionnelles où la suppression est toujours explicite, Redis peut gérer lui-même l'expiration des données selon des règles temporelles.

## Architecture du système d'expiration

### Le dictionnaire des expirations

Comme vu dans l'introduction du module, Redis maintient deux structures parallèles :

```
Base de données Redis (redisDb):
┌─────────────────────────────────────────┐
│ dict *dict;           ← Toutes les clés │
│ dict *expires;        ← Clés avec TTL   │
│ dict *blocking_keys;                    │
│ dict *watched_keys;                     │
└─────────────────────────────────────────┘
```

**Implémentation interne** :

```c
// Structure simplifiée du dictionnaire d'expirations
typedef struct redisDb {
    dict *dict;           // Keyspace principal
    dict *expires;        // Hash table: key → timestamp (millisecondes)
    // ...
} redisDb;

// Exemple de données dans expires:
// "session:abc123" → 1733847600000 (timestamp Unix en ms)
// "cache:result"   → 1733844000000
```

### Représentation temporelle

Redis stocke les expirations en **millisecondes depuis l'epoch Unix** :

```
Epoch Unix (1er janvier 1970 00:00:00 UTC)
         ↓
    [Time axis]
         ↓
1733847600000 ms (9 décembre 2024 15:00:00)
         ↓
    Expiration
```

**Pourquoi les millisecondes ?**
- Précision suffisante pour la plupart des cas d'usage
- Représentable en 64 bits : supporte jusqu'en ~292 millions d'années
- Compatible avec les timestamps système standard

## Commandes de gestion du TTL

### EXPIRE : Définir un TTL relatif (secondes)

**Signature** : `EXPIRE key seconds [NX|XX|GT|LT]`

**Options (Redis 7.0+)** :
- `NX` : Ne définit le TTL que si la clé n'a PAS déjà de TTL
- `XX` : Ne définit le TTL que si la clé a DÉJÀ un TTL
- `GT` : Ne définit le TTL que si le nouveau > actuel
- `LT` : Ne définit le TTL que si le nouveau < actuel

**Exemples** :

```bash
# TTL simple
SET user:session "data"
EXPIRE user:session 3600
# (integer) 1 - Succès

# Vérification
TTL user:session
# (integer) 3599

# Échec sur clé inexistante
EXPIRE nonexistent 3600
# (integer) 0

# Options conditionnelles (Redis 7+)
EXPIRE user:session 7200 NX
# (integer) 0 - Échec car a déjà un TTL

EXPIRE user:session 7200 XX
# (integer) 1 - Succès car a déjà un TTL

EXPIRE user:session 1800 GT
# (integer) 0 - Échec car 1800 < 7200

EXPIRE user:session 10800 GT
# (integer) 1 - Succès car 10800 > 7200
```

**Mécanisme interne** :

```
1. Vérifie que la clé existe dans dict
2. Si options NX/XX/GT/LT : Vérifie conditions
3. Calcule timestamp = now() + seconds * 1000
4. Insère/met à jour dans expires dict
5. Retourne 1 (succès) ou 0 (échec)
```

### PEXPIRE : TTL en millisecondes

**Signature** : `PEXPIRE key milliseconds [NX|XX|GT|LT]`

```bash
# Précision milliseconde
PEXPIRE cache:result 500
# (integer) 1

# Expire dans 500ms
PTTL cache:result
# (integer) 498

# Après 500ms
PTTL cache:result
# (integer) -2 - Clé expirée et supprimée
```

**Cas d'usage** :
- Rate limiting très précis
- Verrous distribués courts
- Cache de calculs rapides
- Coordination temps réel

### EXPIREAT : TTL absolu (timestamp Unix)

**Signature** : `EXPIREAT key unix-timestamp [NX|XX|GT|LT]`

```bash
# Expire à une date précise (timestamp secondes)
EXPIREAT user:promo 1735689600
# (integer) 1
# Expire le 1er janvier 2025 00:00:00 UTC

# Vérification avec date humaine
# date -d @1735689600
# Wed Jan  1 00:00:00 UTC 2025

# Calcul dynamique en bash
EXPIREAT user:temp $(date -d "+1 day" +%s)
```

**Avantages** :
- Expiration précise à une date/heure spécifique
- Idéal pour promotions, contenus temporaires, embargos
- Pas de dérive due aux délais réseau

### PEXPIREAT : TTL absolu en millisecondes

**Signature** : `PEXPIREAT key unix-timestamp-milliseconds [NX|XX|GT|LT]`

```bash
# Précision milliseconde
PEXPIREAT event:live 1733847600000
# (integer) 1

# Calcul en Python
# import time
# timestamp_ms = int((time.time() + 3600) * 1000)
```

### EXPIRETIME : Obtenir le timestamp d'expiration

**Signature** : `EXPIRETIME key` (Redis 7.0+)

```bash
SET key "value"
EXPIRE key 3600

EXPIRETIME key
# (integer) 1733847600 - Timestamp Unix en secondes

# Si pas de TTL
EXPIRETIME persistent_key
# (integer) -1

# Si clé inexistante
EXPIRETIME nonexistent
# (integer) -2
```

### PEXPIRETIME : Timestamp en millisecondes

**Signature** : `PEXPIRETIME key` (Redis 7.0+)

```bash
PEXPIRETIME key
# (integer) 1733847600000
```

### TTL : Temps restant en secondes

**Signature** : `TTL key`

**Valeurs de retour** :
- `N > 0` : Secondes restantes
- `-1` : Clé existe sans TTL (persistante)
- `-2` : Clé n'existe pas

```bash
SET permanent "data"
TTL permanent
# (integer) -1

SET temporary "data"
EXPIRE temporary 3600
TTL temporary
# (integer) 3599

TTL nonexistent
# (integer) -2
```

### PTTL : Temps restant en millisecondes

**Signature** : `PTTL key`

```bash
PTTL temporary
# (integer) 3598947

# Permet un polling précis
while [ $(redis-cli PTTL key) -gt 0 ]; do
    echo "Waiting..."
    sleep 0.1
done
```

### PERSIST : Supprimer le TTL

**Signature** : `PERSIST key`

```bash
SET key "value"
EXPIRE key 3600
TTL key
# (integer) 3599

PERSIST key
# (integer) 1 - Succès

TTL key
# (integer) -1 - Plus de TTL

PERSIST key
# (integer) 0 - Échec, pas de TTL à supprimer
```

**Mécanisme interne** :

```
1. Vérifie que la clé existe
2. Vérifie qu'elle a un TTL (présence dans expires)
3. Supprime l'entrée du dictionnaire expires
4. La clé reste dans dict principal
5. Retourne 1 (succès) ou 0 (échec)
```

## Les deux mécanismes d'expiration

### 1. Expiration passive (Lazy)

**Principe** : Vérification lors de l'accès à la clé.

**Flux d'exécution** :

```
Client envoie : GET key
       ↓
Redis reçoit la commande
       ↓
Lookup clé dans dict
       ↓
   Clé trouvée ?
    ↙        ↘
  NON        OUI
   ↓          ↓
Retourne   Lookup dans expires
  nil          ↓
           A un TTL ?
            ↙      ↘
          NON      OUI
           ↓        ↓
        Retourne  Timestamp < now() ?
        valeur     ↙            ↘
                 NON           OUI
                  ↓             ↓
               Retourne      Supprime clé
               valeur        (dict + expires)
                               ↓
                           Retourne nil
```

**Implémentation simplifiée** :

```c
// Pseudo-code de l'expiration passive
robj *lookupKeyRead(redisDb *db, robj *key) {
    dictEntry *de = dictFind(db->dict, key->ptr);
    if (de == NULL) return NULL;

    // Vérifie expiration
    if (dictFind(db->expires, key->ptr)) {
        long long when = dictGetSignedIntegerVal(de);
        if (mstime() > when) {
            // Clé expirée
            dbDelete(db, key);
            server.stat_expiredkeys++;
            return NULL;
        }
    }

    return dictGetVal(de);
}
```

**Avantages** :
- ✅ Coût CPU minimal
- ✅ Aucune surcharge pour clés jamais accédées
- ✅ Découverte immédiate à l'accès

**Inconvénients** :
- ❌ Clés expirées jamais accédées restent en mémoire
- ❌ Fuite de mémoire potentielle
- ❌ Utilisation mémoire imprévisible

### 2. Expiration active (Proactive)

**Principe** : Vérification périodique en arrière-plan.

**Configuration** :

```conf
# redis.conf
hz 10  # Fréquence du serverCron (10 Hz = 10 fois/seconde)
```

**Algorithme détaillé** :

```python
# Pseudo-code de l'expiration active
def activeExpireCycle():
    # Constantes
    ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP = 20
    ACTIVE_EXPIRE_CYCLE_FAST_DURATION = 1000  # microsecondes
    ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC = 25   # % CPU max

    start_time = monotonic_us()
    timelimit = 1000000 * SLOW_TIME_PERC / server.hz / 100

    for db_id in range(server.dbnum):
        db = server.db[db_id]

        # Répète tant qu'on trouve beaucoup de clés expirées
        while True:
            # 1. Échantillonnage aléatoire
            sampled = 0
            expired = 0

            for _ in range(KEYS_PER_LOOP):
                entry = random_dict_entry(db.expires)
                if entry is None:
                    break

                sampled += 1

                # 2. Vérifie expiration
                if entry.timestamp < now():
                    delete_key(db, entry.key)
                    expired += 1

            # 3. Conditions d'arrêt
            if sampled == 0:
                break  # Plus de clés avec TTL

            if expired < sampled / 4:
                break  # Moins de 25% expirées

            if monotonic_us() - start_time > timelimit:
                break  # Limite CPU atteinte
```

**Comportement en détail** :

```
Cycle d'expiration active (chaque 100ms avec hz=10):
┌─────────────────────────────────────────┐
│ 1. Sélectionne DB 0                     │
│    ├─ Sample 20 clés aléatoires         │
│    ├─ Trouve 8 expirées (40%)           │
│    └─ Continue (>25% expirées)          │
│                                         │
│ 2. Sélectionne DB 0 (2e itération)      │
│    ├─ Sample 20 clés                    │
│    ├─ Trouve 3 expirées (15%)           │
│    └─ Stop (<25% expirées)              │
│                                         │
│ 3. Passe à DB 1                         │
│    └─ ...                               │
└─────────────────────────────────────────┘
```

**Impact du paramètre `hz`** :

| hz | Fréquence | Intervalle | Usage CPU | Réactivité | Use Case |
|----|-----------|------------|-----------|------------|----------|
| 1 | 1 Hz | 1000ms | Très faible | Faible | TTL > 10s |
| 10 | 10 Hz | 100ms | Faible | Normale | Défaut |
| 50 | 50 Hz | 20ms | Moyen | Haute | TTL < 5s |
| 100 | 100 Hz | 10ms | Élevé | Très haute | TTL < 1s |

**Configuration recommandée** :

```conf
# Pour TTL courts (rate limiting, locks)
hz 50

# Pour TTL moyens (cache, sessions)
hz 10  # Défaut

# Pour TTL longs (> 1 heure)
hz 5

# Pour économiser CPU (IoT, embedded)
hz 1
```

### Impact des expirations sur la latence

**Mesure** :

```bash
# Activer le monitoring de latence
CONFIG SET latency-monitor-threshold 100

# Insérer beaucoup de clés avec TTL court
for i in {1..100000}; do
    redis-cli SETEX key:$i 1 value
done

# Attendre l'expiration
sleep 2

# Observer les pics de latence
redis-cli LATENCY DOCTOR
```

**Résultat typique** :

```
I have observed latency spikes in this Redis instance.
The following peak latencies have been detected:

- 15 msec - 2024-12-09 15:30:42 UTC
  Command causing latency: EXPIRE cycle
```

**Mitigation** :

```conf
# Étaler les expirations avec jitter
# Dans l'application:
ttl = base_ttl + random.randint(-jitter, jitter)

# Exemple Python:
import random
base_ttl = 3600
jitter = 300  # ±5 minutes
ttl = base_ttl + random.randint(-jitter, jitter)
redis.setex(key, ttl, value)
```

## Stratégies d'expiration

### 1. TTL fixe

**Pattern** : Toutes les données expirent après la même durée.

```bash
# Cache de requêtes API
SET cache:weather:paris "22°C" EX 1800  # 30 minutes
SET cache:weather:london "18°C" EX 1800
SET cache:weather:berlin "20°C" EX 1800
```

**Avantages** :
- ✅ Simplicité
- ✅ Prévisibilité

**Inconvénients** :
- ❌ Pic d'expiration si beaucoup de clés créées en même temps
- ❌ Pas adapté aux données de valeur variable

**Use cases** :
- Cache d'API externes
- Sessions utilisateur
- Rate limiting par IP

### 2. TTL variable selon contexte

**Pattern** : TTL adapté à l'importance/fréquence d'accès.

```bash
# Cache stratifié
SET cache:hot:user:123 "data" EX 3600    # 1h - données fréquentes
SET cache:warm:user:456 "data" EX 7200   # 2h - données moyennes
SET cache:cold:user:789 "data" EX 14400  # 4h - données rares
```

**Implémentation** :

```python
def cache_with_adaptive_ttl(key, value, access_count):
    if access_count > 1000:
        ttl = 3600  # Hot
    elif access_count > 100:
        ttl = 7200  # Warm
    else:
        ttl = 14400  # Cold

    redis.setex(key, ttl, value)
```

### 3. TTL avec jitter (anti-stampede)

**Pattern** : Ajouter de l'aléatoire pour éviter les expirations simultanées.

```python
import random

def set_with_jitter(redis, key, value, base_ttl, jitter_percent=10):
    jitter = int(base_ttl * jitter_percent / 100)
    ttl = base_ttl + random.randint(-jitter, jitter)
    redis.setex(key, ttl, value)

# Exemple
set_with_jitter(redis, "cache:data", "value", 3600, 20)
# TTL entre 2880 et 4320 secondes (±20%)
```

**Visualisation** :

```
Sans jitter:
Time ─────────────────────────────────────────────→
     t=0s                            t=3600s
     │ 1000 clés créées              │ 1000 clés expirent
     │                               │ → Pic CPU
     └───────────────────────────────┘

Avec jitter (±10%):
Time ─────────────────────────────────────────────→
     t=0s          t=3240s ─── t=3600s ─── t=3960s
     │              │           │           │
     │              └─ 150 exp  │           │
     │                     └─ 700 exp       │
     │                                └─ 150 exp
     └──────────────────────────────────────┘
     → Charge CPU étalée
```

### 4. TTL incrémental (warming)

**Pattern** : Augmenter progressivement le TTL selon la popularité.

```python
def cache_with_warming(redis, key, value):
    # Première insertion : TTL court
    if not redis.exists(key):
        redis.setex(key, 300, value)  # 5 minutes
    else:
        # Augmente le TTL si réaccédée
        current_ttl = redis.ttl(key)
        if current_ttl < 3600:  # Max 1 heure
            new_ttl = min(current_ttl * 2, 3600)
            redis.expire(key, new_ttl)
```

### 5. TTL glissant (sliding window)

**Pattern** : Réinitialiser le TTL à chaque accès.

```python
def get_with_sliding_ttl(redis, key, base_ttl=3600):
    value = redis.get(key)
    if value:
        # Réinitialise le TTL
        redis.expire(key, base_ttl)
    return value
```

**Use case** : Sessions actives, locks avec renouvellement

**Alternative optimisée** :

```python
# Évite le double round-trip
def get_and_reset_ttl(redis, key, ttl):
    pipe = redis.pipeline()
    pipe.get(key)
    pipe.expire(key, ttl)
    result = pipe.execute()
    return result[0]
```

### 6. TTL absolu (deadline)

**Pattern** : Expiration à une date/heure précise.

```python
from datetime import datetime, timedelta

def set_with_deadline(redis, key, value, deadline):
    timestamp = int(deadline.timestamp())
    redis.set(key, value)
    redis.expireat(key, timestamp)

# Exemple: Promotion jusqu'au 31 décembre
deadline = datetime(2024, 12, 31, 23, 59, 59)
set_with_deadline(redis, "promo:xmas", "50% OFF", deadline)
```

**Use cases** :
- Promotions temporaires
- Contenu embarqué temporaire
- Droits d'accès temporaires
- Événements planifiés

### 7. Multi-tier TTL (cascade)

**Pattern** : Plusieurs niveaux de cache avec TTL différents.

```python
def cache_cascade(redis, key, value):
    # L1: Très rapide, TTL court
    redis.setex(f"l1:{key}", 60, value)

    # L2: Rapide, TTL moyen
    redis.setex(f"l2:{key}", 300, value)

    # L3: Moins rapide, TTL long
    redis.setex(f"l3:{key}", 3600, value)

def get_cascade(redis, key):
    # Cherche L1 d'abord
    value = redis.get(f"l1:{key}")
    if value:
        return value

    # Cherche L2
    value = redis.get(f"l2:{key}")
    if value:
        # Repopule L1
        redis.setex(f"l1:{key}", 60, value)
        return value

    # Cherche L3
    value = redis.get(f"l3:{key}")
    if value:
        # Repopule L1 et L2
        redis.setex(f"l1:{key}", 60, value)
        redis.setex(f"l2:{key}", 300, value)
        return value

    return None
```

## Interactions TTL avec les commandes

### Commandes qui préservent le TTL

```bash
SET key "value" EX 3600
TTL key
# (integer) 3599

# APPEND préserve le TTL
APPEND key " more"
TTL key
# (integer) 3595 - TTL continue

# INCR préserve le TTL
SET counter 0 EX 3600
INCR counter
TTL counter
# (integer) 3598 - TTL préservé

# LPUSH, RPUSH, SADD, etc. préservent le TTL
```

### Commandes qui suppriment le TTL

```bash
SET key "value" EX 3600
TTL key
# (integer) 3599

# SET sans KEEPTTL supprime le TTL
SET key "new value"
TTL key
# (integer) -1 - Plus de TTL !

# GETSET supprime le TTL (legacy)
GETSET key "newer"
TTL key
# (integer) -1
```

### Option KEEPTTL (Redis 6.0+)

```bash
SET key "value" EX 3600
TTL key
# (integer) 3599

# Préserve le TTL existant
SET key "new value" KEEPTTL
TTL key
# (integer) 3595 - TTL préservé !
```

### RENAME et TTL

```bash
SET oldkey "value" EX 3600
RENAME oldkey newkey

# Le TTL est transféré
TTL newkey
# (integer) 3598

# RENAMENX aussi
RENAMENX oldkey2 newkey2
TTL newkey2
# (integer) XXX - TTL transféré
```

### Transactions et TTL

```bash
MULTI
SET key "value"
EXPIRE key 3600
EXEC
# 1) OK
# 2) (integer) 1

# Le TTL est bien appliqué atomiquement
TTL key
# (integer) 3599
```

## Patterns avancés

### 1. Cache avec pré-expiration (probabilistic early expiration)

**Problème** : Éviter le cache stampede quand beaucoup de clients accèdent à une clé qui vient d'expirer.

**Solution** : Expire "probablement" avant le TTL réel.

```python
import random
import time

def get_with_early_expiration(redis, key, beta=1.0):
    # Récupère valeur et TTL
    pipe = redis.pipeline()
    pipe.get(key)
    pipe.ttl(key)
    value, ttl = pipe.execute()

    if value is None:
        return None

    # Calcule probabilité d'expiration anticipée
    # xfetch = -beta * ttl * log(random())
    if ttl > 0:
        delta = beta * ttl * abs(random.random())
        if delta < 1.0:
            # Expire anticipativement
            return None

    return value
```

**Formule** :

```
P(expire) = beta * TTL * ln(rand())

beta = 1.0 : expire ~37% du temps quand TTL → 0
beta = 2.0 : expire ~86% du temps quand TTL → 0
```

### 2. Distributed lock avec TTL

```python
import uuid

def acquire_lock(redis, lock_name, ttl=10):
    identifier = str(uuid.uuid4())

    # Acquiert le lock avec TTL
    if redis.set(lock_name, identifier, nx=True, ex=ttl):
        return identifier
    return None

def release_lock(redis, lock_name, identifier):
    # Script Lua pour release atomique
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    return redis.eval(script, 1, lock_name, identifier)

def refresh_lock(redis, lock_name, identifier, ttl=10):
    # Script Lua pour refresh atomique
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("expire", KEYS[1], ARGV[2])
    else
        return 0
    end
    """
    return redis.eval(script, 1, lock_name, identifier, ttl)
```

### 3. Rate limiting avec TTL

**Fixed Window** :

```python
def rate_limit_fixed(redis, user_id, limit=100, window=60):
    key = f"rate:{user_id}:{int(time.time() // window)}"

    pipe = redis.pipeline()
    pipe.incr(key)
    pipe.expire(key, window * 2)  # Safety margin
    count, _ = pipe.execute()

    return count <= limit
```

**Sliding Window Log** :

```python
import time

def rate_limit_sliding(redis, user_id, limit=100, window=60):
    key = f"rate:{user_id}"
    now = time.time()

    pipe = redis.pipeline()
    # Supprime événements expirés
    pipe.zremrangebyscore(key, 0, now - window)
    # Ajoute nouvel événement
    pipe.zadd(key, {str(now): now})
    # Compte événements dans la fenêtre
    pipe.zcard(key)
    # Définit TTL
    pipe.expire(key, window)

    _, _, count, _ = pipe.execute()

    return count <= limit
```

### 4. Session store avec renouvellement automatique

```python
def save_session(redis, session_id, data, ttl=1800):
    key = f"session:{session_id}"
    redis.setex(key, ttl, json.dumps(data))

def get_session(redis, session_id, auto_renew=True):
    key = f"session:{session_id}"
    data = redis.get(key)

    if data and auto_renew:
        # Renouvelle le TTL à chaque accès
        redis.expire(key, 1800)

    return json.loads(data) if data else None

def touch_session(redis, session_id):
    # Renouvelle sans récupérer les données
    key = f"session:{session_id}"
    redis.expire(key, 1800)
```

### 5. Cache avec grace period

**Pattern** : Servir du cache expiré temporairement pendant le rechargement.

```python
def get_with_grace(redis, key, grace_period=30):
    # Clés primaire et de grace
    primary_key = f"cache:{key}"
    grace_key = f"grace:{key}"

    # Essaie cache primaire
    value = redis.get(primary_key)
    if value:
        return value

    # Cache expiré, essaie grace
    value = redis.get(grace_key)
    if value:
        # Retourne valeur expirée pendant rechargement
        return value

    return None

def set_with_grace(redis, key, value, ttl=3600, grace_period=30):
    primary_key = f"cache:{key}"
    grace_key = f"grace:{key}"

    # Cache primaire
    redis.setex(primary_key, ttl, value)

    # Cache de grace (TTL plus long)
    redis.setex(grace_key, ttl + grace_period, value)
```

## Monitoring et observabilité

### Métriques clés

```bash
# Nombre de clés expirées
redis-cli INFO stats | grep expired_keys
# expired_keys:12543

# Nombre total de clés avec TTL
redis-cli INFO keyspace
# db0:keys=50000,expires=12000,avg_ttl=3600000

# Taux d'expiration par seconde
redis-cli INFO stats | grep expired_keys
# Attendre 1 seconde
redis-cli INFO stats | grep expired_keys
# Calculer la différence
```

### Commandes de diagnostic

```bash
# Trouver toutes les clés avec TTL < 60s
redis-cli --scan --pattern '*' | while read key; do
    ttl=$(redis-cli TTL "$key")
    if [ "$ttl" -gt 0 ] && [ "$ttl" -lt 60 ]; then
        echo "$key: $ttl seconds"
    fi
done

# Distribution des TTL
redis-cli --eval ttl_distribution.lua

# Script Lua ttl_distribution.lua:
local keys = redis.call('KEYS', '*')
local buckets = {0, 0, 0, 0, 0}  -- <60s, <5m, <1h, <1d, >1d

for i, key in ipairs(keys) do
    local ttl = redis.call('TTL', key)
    if ttl > 0 then
        if ttl < 60 then
            buckets[1] = buckets[1] + 1
        elseif ttl < 300 then
            buckets[2] = buckets[2] + 1
        elseif ttl < 3600 then
            buckets[3] = buckets[3] + 1
        elseif ttl < 86400 then
            buckets[4] = buckets[4] + 1
        else
            buckets[5] = buckets[5] + 1
        end
    end
end

return buckets
```

### Alerting sur les expirations

```python
# Monitoring script
def monitor_expiration_rate(redis, threshold=1000):
    stats1 = redis.info('stats')
    expired1 = stats1['expired_keys']

    time.sleep(60)

    stats2 = redis.info('stats')
    expired2 = stats2['expired_keys']

    rate = expired2 - expired1

    if rate > threshold:
        alert(f"High expiration rate: {rate} keys/min")
```

## Pièges et considérations

### 1. Expiration massive simultanée

**Problème** :

```python
# ❌ Anti-pattern
for i in range(1000000):
    redis.setex(f"key:{i}", 3600, "value")

# Toutes expirent en même temps 1h plus tard
# → Pic CPU massif
```

**Solution** :

```python
# ✅ Avec jitter
import random
for i in range(1000000):
    ttl = 3600 + random.randint(-300, 300)  # ±5 min
    redis.setex(f"key:{i}", ttl, "value")
```

### 2. TTL et réplication

**Comportement** :
- L'expiration active se produit sur le master
- Les replicas reçoivent des commandes DEL explicites
- Les replicas ne font PAS d'expiration active autonome

**Implications** :

```
Master:                    Replica:
┌────────────────┐        ┌────────────────┐
│ Expire active  │  DEL   │                │
│ Détecte key:123│───────→│ Reçoit DEL     │
│ Envoie DEL     │        │ Supprime key   │
└────────────────┘        └────────────────┘
```

**Problème potentiel** : Lag de réplication peut causer une disponibilité temporaire de clés expirées sur replicas.

### 3. Transactions et expiration

```bash
# La clé peut expirer PENDANT la transaction
MULTI
GET key  # key existe
# ... 10 secondes passent, key expire
SET key "new"
EXEC

# Résultat:
# 1) (nil)  - key avait expiré
# 2) OK
```

**Solution** : Utiliser WATCH ou vérifier TTL avant MULTI.

### 4. TTL négatifs ou zéro

```bash
# TTL négatif
EXPIRE key -1
# (integer) 1 - Accepté !

GET key
# (nil) - Supprimé immédiatement

# TTL zéro
EXPIRE key 0
# (integer) 1

GET key
# (nil) - Supprimé immédiatement
```

### 5. Impact sur la persistence

**RDB** : Les clés expirées ne sont pas sauvées dans le snapshot.

**AOF** : Les expirations génèrent des commandes DEL dans le log.

```
# AOF file
*2
$3
SET
$4
key1
$5
value
*2
$6
EXPIRE
$4
key1
$4
3600
# ... 1 heure plus tard (expiration active)
*2
$3
DEL
$4
key1
```

**Configuration** :

```conf
# redis.conf

# Ne pas persister les clés avec TTL très court
# (elles seront déjà expirées au reload)
# → Utiliser une base DB dédiée sans AOF/RDB
```

### 6. Overhead mémoire du dictionnaire expires

```
Impact mémoire par clé avec TTL:
┌────────────────────────────────┐
│ Entry dans dict principal:     │
│ - Pointeur clé: 8 bytes        │
│ - Pointeur valeur: 8 bytes     │
│                                │
│ Entry dans dict expires:       │
│ - Pointeur clé: 8 bytes        │
│ - Timestamp: 8 bytes           │
│                                │
│ Total overhead: ~32 bytes      │
└────────────────────────────────┘
```

**Exemple** : 10M clés avec TTL = ~320MB d'overhead

## Configuration optimale

```conf
# redis.conf - Configuration production

# Fréquence d'expiration active
hz 10  # 10 Hz = toutes les 100ms (défaut)
# Augmenter si TTL courts (<10s): hz 50
# Diminuer si CPU limité ou TTL longs: hz 5

# Activation du lazy free pour expirations
lazyfree-lazy-expire yes  # Expire en background

# Limite CPU pour expiration active (implicite via hz)
# Avec hz=10: max ~2.5% CPU pour expiration

# AOF et expirations
appendfsync everysec  # Balance performance/durabilité

# RDB et expirations
save 900 1      # Snapshot après 15 min si 1 write
save 300 10     # Snapshot après 5 min si 10 writes
save 60 10000   # Snapshot après 1 min si 10k writes

# Monitoring
latency-monitor-threshold 100  # 100ms
```

## Résumé : Cheat sheet TTL

| Commande | Précision | Type | Use Case |
|----------|-----------|------|----------|
| EXPIRE | Secondes | Relatif | Cache général |
| PEXPIRE | Millisecondes | Relatif | Rate limiting précis |
| EXPIREAT | Secondes | Absolu | Promotions/deadlines |
| PEXPIREAT | Millisecondes | Absolu | Événements précis |
| TTL | Secondes | Query | Monitoring |
| PTTL | Millisecondes | Query | Polling précis |
| PERSIST | - | Suppression | Rendre permanent |
| EXPIRETIME | Secondes | Query | Debug (Redis 7+) |
| PEXPIRETIME | Millisecondes | Query | Debug (Redis 7+) |

**Valeurs de retour TTL/PTTL** :
- `N > 0` : Temps restant
- `-1` : Pas de TTL (persistant)
- `-2` : Clé inexistante

**Options EXPIRE (Redis 7+)** :
- `NX` : Définit TTL si n'en a pas
- `XX` : Définit TTL si en a déjà un
- `GT` : Définit si nouveau > actuel
- `LT` : Définit si nouveau < actuel

## Conclusion

Le TTL est un outil puissant mais qui demande une compréhension fine des mécanismes internes :

- **Deux modes d'expiration** : Passive (à l'accès) et active (périodique)
- **Configuration critique** : Le paramètre `hz` contrôle la balance CPU/réactivité
- **Stratégies variées** : Du simple TTL fixe aux patterns complexes avec jitter et grace periods
- **Overhead** : ~32 bytes par clé avec TTL
- **Réplication** : Les replicas reçoivent des DEL, ne font pas d'expiration autonome

La maîtrise du TTL permet de construire des caches auto-nettoyants, des systèmes de rate limiting précis, et d'optimiser l'utilisation mémoire sans intervention manuelle.

La section suivante abordera les politiques d'éviction lorsque la mémoire est saturée.

⏭️ [Politiques d'éviction : Que se passe-t-il quand la RAM est pleine ?](/04-cycle-vie-donnee/03-politiques-eviction-ram-pleine.md)
