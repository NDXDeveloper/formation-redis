🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1 Commandes CRUD fondamentales et gestion des erreurs

## Introduction

Les opérations CRUD (Create, Read, Update, Delete) constituent la base de toute interaction avec Redis. Contrairement aux bases de données relationnelles où ces opérations sont exprimées en SQL, Redis expose des **commandes atomiques** spécifiques à chaque structure de données. Comprendre leurs mécanismes internes, leurs garanties et leurs modes de défaillance est essentiel pour construire des applications robustes.

## Architecture des commandes Redis

### Le pipeline de traitement d'une commande

Chaque commande Redis traverse plusieurs étapes avant son exécution :

```
Client → Connexion TCP → Command Buffer → Parser → Command Table Lookup
                                                            ↓
                                                    Validation args
                                                            ↓
                                                    Execution function
                                                            ↓
                                                    Response encoding
                                                            ↓
                                                    Output Buffer → Client
```

### La table des commandes (Command Table)

Redis maintient une table interne qui définit chaque commande :

```c
// Structure simplifiée (code C de Redis)
struct redisCommand {
    char *name;              // Nom de la commande
    redisCommandProc *proc;  // Fonction d'exécution
    int arity;               // Nombre d'arguments (-N = au moins N)
    char *sflags;            // Flags (write, readonly, fast, etc.)
    uint64_t flags;          // Flags compilés
    int firstkey;            // Position du premier arg clé
    int lastkey;             // Position du dernier arg clé
    int keystep;             // Pas entre les clés
    long long microseconds;  // Temps d'exécution cumulé
    long long calls;         // Nombre d'appels
};
```

**Exemple de définition** :

```c
{"get", getCommand, 2, "rF", 0, NULL, 1, 1, 1, 0, 0, 0}
// name: "get"
// arity: 2 (commande + 1 argument)
// flags: "r" (readonly), "F" (fast)
// firstkey: 1, lastkey: 1, keystep: 1
```

### Flags importants

Les flags déterminent le comportement de la commande :

| Flag | Signification | Impact |
|------|---------------|--------|
| `w` | Write | Modifie les données, répliqué, AOF |
| `r` | Readonly | Lecture seule, peut s'exécuter sur replica |
| `F` | Fast | Complexité O(1), très rapide |
| `s` | Slow | Peut prendre du temps (KEYS, FLUSHDB) |
| `R` | Random | Retour non déterministe |
| `S` | Sort | Trie les résultats |
| `m` | May-replicate | Peut répliquer conditionnellement |
| `M` | Movable-keys | Position des clés variable |

## Opérations CREATE : Création de données

### SET : La commande universelle

**Signature** :
```
SET key value [NX|XX] [GET] [EX seconds|PX milliseconds|EXAT unix-time-seconds|PXAT unix-time-milliseconds|KEEPTTL]
```

**Mécanisme interne** :

```
1. Parser valide les arguments
2. Vérifie NX/XX si présent
   ├─ NX : Vérifie que la clé n'existe pas
   └─ XX : Vérifie que la clé existe
3. Si GET : Sauvegarde ancienne valeur
4. Alloue mémoire pour la nouvelle valeur
5. Encode la valeur (INT, EMBSTR ou RAW)
6. Insère/met à jour dans le dictionnaire
7. Si TTL : Insère dans expires dict
8. Libère ancienne valeur si remplacement
9. Propage aux replicas
10. Écrit dans AOF si activé
```

**Exemples d'utilisation** :

```bash
# Création simple
SET user:1:name "Alice"
# OK

# Création conditionnelle (clé n'existe pas)
SET user:2:name "Bob" NX
# OK

SET user:2:name "Bobby" NX
# (nil) - Échec car existe déjà

# Mise à jour conditionnelle (clé existe)
SET user:2:name "Bobby" XX
# OK

# Set et Get atomique
SET user:2:name "Robert" GET
# "Bobby" - Retourne l'ancienne valeur

# Avec TTL
SET session:abc123 "data" EX 3600
# OK - Expire dans 1 heure

# TTL en millisecondes
SET cache:result "value" PX 500
# OK - Expire dans 500ms

# TTL absolu (timestamp UNIX)
SET temp:data "value" EXAT 1735689600
# OK - Expire à une date précise

# Préserver le TTL existant
SET user:1:name "Alice Updated" KEEPTTL
# OK - Garde le TTL original
```

### Encodage automatique des valeurs

Redis optimise automatiquement l'encodage selon la valeur :

```bash
# Encodage INT (8 bytes)
SET counter:views 42
OBJECT ENCODING counter:views
# "int"

# Encodage EMBSTR (≤44 bytes, Redis 7+)
SET user:short "Alice"
OBJECT ENCODING user:short
# "embstr"

# Encodage RAW (>44 bytes)
SET user:long "This is a very long string that exceeds the embstr limit..."
OBJECT ENCODING user:long
# "raw"
```

**Structure mémoire selon l'encodage** :

```
INT (entier -9223372036854775808 à 9223372036854775807):
┌──────────────────┐
│ RedisObject      │ 16 bytes
│ type: STRING     │
│ encoding: INT    │
│ ptr: valeur      │ ← Valeur directement dans le pointeur
└──────────────────┘
Total: 16 bytes

EMBSTR (≤44 bytes):
┌──────────────────┬──────────────┐
│ RedisObject      │ SDS String   │
│ type: STRING     │ len: 5       │
│ encoding: EMBSTR │ "Alice\0"    │
└──────────────────┴──────────────┘
Total: 16 + 8 + len bytes (allocation unique)

RAW (>44 bytes):
┌──────────────────┐      ┌──────────────┐
│ RedisObject      │ ───→ │ SDS String   │
│ type: STRING     │      │ len: 50      │
│ encoding: RAW    │      │ "Long..."    │
└──────────────────┘      └──────────────┘
Total: 16 + 8 + len bytes (allocations séparées)
```

### SETNX, SETEX, PSETEX : Commandes legacy

Ces commandes existent toujours mais sont **obsolètes** depuis Redis 2.6.12 :

```bash
# Legacy (déprécié)
SETNX key value
SETEX key seconds value
PSETEX key milliseconds value

# Moderne (recommandé)
SET key value NX
SET key value EX seconds
SET key value PX milliseconds
```

**Pourquoi préférer SET avec options ?**
- Moins de commandes à maintenir
- Atomicité garantie (1 seule commande)
- Options combinables (NX + EX en une fois)
- Meilleure lisibilité

### Commandes spécifiques aux structures

**Hashes** :

```bash
# Création d'un hash complet
HSET user:1 name "Alice" age 30 city "Paris"
# (integer) 3

# Création conditionnelle d'un champ
HSETNX user:1 email "alice@example.com"
# (integer) 1

HSETNX user:1 email "bob@example.com"
# (integer) 0 - Échec, existe déjà
```

**Lists** :

```bash
# Création par insertion gauche
LPUSH queue:tasks "task1" "task2" "task3"
# (integer) 3

# Création par insertion droite
RPUSH events "event1" "event2"
# (integer) 2

# Insertion conditionnelle (clé doit exister)
LPUSHX queue:tasks "task0"
# (integer) 4 - Succès

LPUSHX queue:new "task"
# (integer) 0 - Échec, liste n'existe pas
```

**Sets** :

```bash
# Création d'un set
SADD tags:article:1 "redis" "database" "nosql"
# (integer) 3

# Ajout de doublons ignoré
SADD tags:article:1 "redis"
# (integer) 0
```

**Sorted Sets** :

```bash
# Création avec scores
ZADD leaderboard 100 "player1" 200 "player2" 150 "player3"
# (integer) 3

# Avec options
ZADD leaderboard NX 250 "player4"  # N'ajoute que si n'existe pas
ZADD leaderboard XX 180 "player3"  # Met à jour si existe
ZADD leaderboard GT 210 "player2"  # Met à jour si score > actuel
ZADD leaderboard LT 90 "player1"   # Met à jour si score < actuel
```

## Opérations READ : Lecture de données

### GET : Lecture simple

**Mécanisme interne** :

```
1. Lookup dans le dictionnaire principal
2. Si clé trouvée :
   ├─ Vérifie TTL dans expires dict
   │  ├─ Si expiré : Supprime et retourne nil
   │  └─ Si valide : Continue
   ├─ Met à jour LRU/LFU
   └─ Retourne la valeur
3. Si clé non trouvée : Retourne nil
```

**Exemples** :

```bash
GET user:1:name
# "Alice"

GET user:999:name
# (nil)

# GET ne fonctionne que sur les strings
HSET obj:1 field value
GET obj:1
# (error) WRONGTYPE Operation against a key holding the wrong kind of value
```

### MGET : Lecture multiple atomique

**Avantages** :
- 1 seul round-trip réseau au lieu de N
- Atomique (snapshot cohérent)
- Jusqu'à 10x plus rapide que N GET

```bash
MGET user:1:name user:2:name user:3:name
# 1) "Alice"
# 2) "Bob"
# 3) (nil)

# Même avec des clés inexistantes
MGET key1 key2 key3 key4 key5
# Retourne toujours un tableau de 5 éléments
```

**Complexité** : O(N) où N = nombre de clés

### EXISTS : Vérification d'existence

```bash
EXISTS user:1
# (integer) 1 - Existe

EXISTS user:999
# (integer) 0 - N'existe pas

# Multiple keys (Redis 3.0.3+)
EXISTS user:1 user:2 user:999
# (integer) 2 - Retourne le nombre de clés existantes
```

**Mécanisme interne** :
- Simple lookup dans le dictionnaire
- Vérifie le TTL si la clé a une expiration
- Ne charge pas la valeur en mémoire
- Très performant : O(1) par clé

### TYPE : Identification du type

```bash
SET key1 "value"
TYPE key1
# string

LPUSH key2 "item"
TYPE key2
# list

HSET key3 field value
TYPE key3
# hash

TYPE nonexistent
# none
```

**Types retournés** :
- `string`
- `list`
- `set`
- `zset` (sorted set)
- `hash`
- `stream`
- `none` (clé inexistante)

### Commandes de lecture par structure

**Hashes** :

```bash
# Lire un champ
HGET user:1 name
# "Alice"

# Lire plusieurs champs
HMGET user:1 name age city
# 1) "Alice"
# 2) "30"
# 3) "Paris"

# Lire tous les champs
HGETALL user:1
# 1) "name"
# 2) "Alice"
# 3) "age"
# 4) "30"
# 5) "city"
# 6) "Paris"

# Nombre de champs
HLEN user:1
# (integer) 3

# Vérifier existence d'un champ
HEXISTS user:1 email
# (integer) 0
```

**Lists** :

```bash
# Lire par index
LINDEX queue:tasks 0
# "task3"

# Lire range
LRANGE queue:tasks 0 -1
# 1) "task3"
# 2) "task2"
# 3) "task1"

# Longueur
LLEN queue:tasks
# (integer) 3
```

**Sets** :

```bash
# Lire tous les membres
SMEMBERS tags:article:1
# 1) "redis"
# 2) "database"
# 3) "nosql"

# Vérifier appartenance
SISMEMBER tags:article:1 "redis"
# (integer) 1

# Cardinalité
SCARD tags:article:1
# (integer) 3
```

**Sorted Sets** :

```bash
# Par rang
ZRANGE leaderboard 0 2
# 1) "player1"
# 2) "player3"
# 3) "player2"

# Par score
ZRANGEBYSCORE leaderboard 100 200
# 1) "player1"
# 2) "player3"
# 3) "player2"

# Avec scores
ZRANGE leaderboard 0 -1 WITHSCORES
# 1) "player1"
# 2) "100"
# 3) "player3"
# 4) "150"
```

## Opérations UPDATE : Mise à jour de données

### SET avec XX : Update conditionnel

```bash
# Met à jour uniquement si existe
SET user:1:name "Alice Updated" XX
# OK

SET user:999:name "Nobody" XX
# (nil) - Échec, clé n'existe pas
```

### Opérations atomiques d'incrémentation

**INCR, INCRBY, DECR, DECRBY** :

```bash
# Initialise à 0 si n'existe pas, puis incrémente
INCR counter:views
# (integer) 1

INCR counter:views
# (integer) 2

INCRBY counter:views 10
# (integer) 12

DECR counter:views
# (integer) 11

DECRBY counter:views 5
# (integer) 6
```

**Mécanisme interne** :

```
1. Lookup clé dans dictionnaire
2. Si n'existe pas : Crée avec valeur 0
3. Si existe :
   ├─ Vérifie type STRING
   ├─ Vérifie que la valeur est un entier valide
   └─ Parse la valeur
4. Effectue l'opération arithmétique
5. Vérifie overflow (64 bits signé)
6. Stocke nouvelle valeur
7. Réplique et écrit AOF
```

**Limites** :

```bash
SET counter "not_a_number"
INCR counter
# (error) ERR value is not an integer or out of range

# Limite supérieure
SET counter 9223372036854775807
INCR counter
# (error) ERR increment or decrement would overflow

# Limite inférieure
SET counter -9223372036854775808
DECR counter
# (error) ERR increment or decrement would overflow
```

**INCRBYFLOAT** : Arithmétique flottante

```bash
SET temperature 23.5
INCRBYFLOAT temperature 1.3
# "24.8"

INCRBYFLOAT temperature -0.5
# "24.3"
```

### APPEND : Concaténation

```bash
SET message "Hello"
APPEND message " World"
# (integer) 11 - Retourne la nouvelle longueur

GET message
# "Hello World"

# Si clé n'existe pas, équivalent à SET
APPEND newkey "value"
# (integer) 5
```

**Cas d'usage** :
- Construction progressive de strings
- Logs simples
- Concaténation sans lecture préalable

### GETSET : Mise à jour atomique avec lecture

**Note** : Déprécié depuis Redis 6.2, utiliser `SET key value GET`

```bash
# Ancien (déprécié)
GETSET counter:total 0
# "156" - Retourne ancienne valeur

# Nouveau (recommandé)
SET counter:total 0 GET
# "156"
```

### Mises à jour sur structures complexes

**Hashes** :

```bash
# Mise à jour d'un champ
HSET user:1 age 31
# (integer) 0 - 0 car champ existait

# Incrément atomique
HINCRBY user:1 age 1
# (integer) 32

HINCRBYFLOAT user:1 rating 0.5
# "4.5"
```

**Lists** :

```bash
# Mise à jour par index
LSET queue:tasks 0 "updated_task"
# OK

# Erreur si index invalide
LSET queue:tasks 100 "task"
# (error) ERR index out of range
```

**Sorted Sets** :

```bash
# Incrément de score
ZINCRBY leaderboard 50 "player1"
# "150" - Nouveau score

# Avec options conditionnelles (Redis 6.2+)
ZADD leaderboard GT 160 "player1"
# (integer) 0 - Pas mis à jour car 160 > 150 est faux
```

## Opérations DELETE : Suppression de données

### DEL : Suppression synchrone

**Signature** : `DEL key [key ...]`

**Mécanisme interne** :

```
1. Pour chaque clé :
   ├─ Lookup dans dictionnaire
   ├─ Si existe :
   │  ├─ Supprime du dictionnaire principal
   │  ├─ Supprime du dictionnaire expires si TTL
   │  ├─ Appelle destructeur spécifique au type
   │  ├─ Libère mémoire IMMÉDIATEMENT
   │  └─ Incrémente compteur deleted
   └─ Si n'existe pas : Continue
2. Réplique aux replicas
3. Écrit dans AOF
4. Retourne nombre de clés supprimées
```

```bash
DEL user:1
# (integer) 1

DEL user:1 user:2 user:999
# (integer) 2 - user:1 et user:2 supprimés, user:999 n'existait pas

# Suppression de grande structure (BLOQUANT !)
SADD bigset $(seq 1 10000000)
DEL bigset
# ... peut prendre plusieurs secondes et bloquer Redis !
```

**Complexité** :
- O(1) pour chaque clé de type simple (string, small hash)
- O(N) pour structures complexes où N = nombre d'éléments
  - Set de 1M membres : ~100ms de blocage
  - List de 1M éléments : ~100ms de blocage

### UNLINK : Suppression asynchrone (recommandé)

**Signature** : `UNLINK key [key ...]`

**Différence avec DEL** :

```
DEL:
Client → Commande → Suppression → Libération mémoire → Réponse
                    └────────────────────┘
                         BLOQUANT

UNLINK:
Client → Commande → Déréférencement → Réponse
                                ↓
                    Thread background → Libération mémoire
                                        (NON BLOQUANT)
```

**Mécanisme** :

```
1. Pour chaque clé :
   ├─ Lookup dans dictionnaire
   ├─ Si existe :
   │  ├─ Calcule coût de libération
   │  ├─ Si coût < seuil (64 éléments) :
   │  │  └─ Libère directement (comme DEL)
   │  └─ Sinon :
   │     ├─ Supprime du dictionnaire
   │     ├─ Ajoute à la queue de libération
   │     └─ Thread bio_lazy_free libère en background
   └─ Si n'existe pas : Continue
2. Retourne immédiatement nombre de clés supprimées
```

```bash
# Petite structure : comportement identique à DEL
UNLINK user:1
# (integer) 1

# Grande structure : non bloquant
SADD bigset $(seq 1 10000000)
UNLINK bigset
# (integer) 1 - Retour immédiat !
```

**Configuration du thread background** :

```conf
# redis.conf
lazyfree-lazy-user-del no  # Par défaut DEL est synchrone
# Pour rendre DEL asynchrone automatiquement :
lazyfree-lazy-user-del yes  # DEL se comporte comme UNLINK
```

**Quand utiliser UNLINK ?**
- ✅ Suppression de grandes structures (>10k éléments)
- ✅ Suppression fréquente de clés
- ✅ Application sensible à la latence
- ✅ Toujours en production !

**Quand utiliser DEL ?**
- ⚠️ Test/développement uniquement
- ⚠️ Suppression garantie immédiate requise (rare)

### Suppression de champs spécifiques

**Hashes** :

```bash
HDEL user:1 temporary_field
# (integer) 1

HDEL user:1 field1 field2 field3
# (integer) 2 - Supprime plusieurs champs
```

**Sets** :

```bash
SREM tags:article:1 "obsolete_tag"
# (integer) 1

SREM tags:article:1 "tag1" "tag2" "tag3"
# (integer) 2
```

**Sorted Sets** :

```bash
ZREM leaderboard "player_inactive"
# (integer) 1

# Suppression par rang
ZREMRANGEBYRANK leaderboard 0 9
# (integer) 10 - Supprime top 10

# Suppression par score
ZREMRANGEBYSCORE leaderboard 0 50
# (integer) 5 - Supprime scores 0-50
```

**Lists** :

```bash
# Suppression par valeur
LREM queue:tasks 0 "task_to_remove"
# (integer) 1 - Supprime toutes les occurrences

LREM queue:tasks 2 "duplicate_task"
# (integer) 2 - Supprime 2 premières occurrences

# Trim (garde uniquement un range)
LTRIM queue:tasks 0 99
# OK - Garde seulement les 100 premiers éléments
```

## Gestion des erreurs : Typologie et stratégies

### Catégories d'erreurs Redis

#### 1. Erreurs de syntaxe

```bash
SET key
# (error) ERR wrong number of arguments for 'set' command

GET key1 key2
# (error) ERR wrong number of arguments for 'get' command
```

**Code d'erreur** : `ERR`
**Cause** : Arguments invalides
**Action** : Corriger la commande côté client

#### 2. Erreurs de type

```bash
LPUSH mystring "value"
# OK

GET mystring
# (error) WRONGTYPE Operation against a key holding the wrong kind of value
```

**Code d'erreur** : `WRONGTYPE`
**Cause** : Opération incompatible avec le type de données
**Action** : Vérifier le type avec TYPE avant opération

#### 3. Erreurs de valeur

```bash
SET counter "abc"
INCR counter
# (error) ERR value is not an integer or out of range
```

**Code d'erreur** : `ERR`
**Cause** : Valeur invalide pour l'opération
**Action** : Valider les données avant insertion

#### 4. Erreurs de mémoire

```bash
# Avec maxmemory atteint et noeviction
SET newkey "value"
# (error) OOM command not allowed when used memory > 'maxmemory'.
```

**Code d'erreur** : `OOM` (Out Of Memory)
**Cause** : Limite maxmemory atteinte
**Action** :
- Augmenter maxmemory
- Changer politique d'éviction
- Nettoyer les données

#### 5. Erreurs de réplication

```bash
# Sur un replica avec writes désactivés
SET key "value"
# (error) READONLY You can't write against a read only replica.
```

**Code d'erreur** : `READONLY`
**Cause** : Tentative d'écriture sur un replica
**Action** : Diriger les writes vers le master

#### 6. Erreurs transactionnelles

```bash
MULTI
SET key1 "value"
INCR key1  # Sera en erreur car "value" n'est pas un int
EXEC
# 1) OK
# 2) (error) ERR value is not an integer or out of range
```

**Comportement** : La transaction s'exécute partiellement !
**Action** : Valider les données avant MULTI

#### 7. Erreurs de script Lua

```bash
EVAL "return redis.call('INCR', 'nonint')" 0
# (error) ERR Error running script (call to f_...): @user_script:1: ERR value is not an integer or out of range
```

**Code d'erreur** : `ERR Error running script`
**Cause** : Erreur dans le script ou commande appelée
**Action** : Gérer les erreurs dans le script Lua

### Codes de retour selon le protocole RESP

Redis utilise RESP (REdis Serialization Protocol) :

```
Type de réponse :
+ Simple String    : +OK\r\n
- Error           : -ERR message\r\n
: Integer         : :42\r\n
$ Bulk String     : $5\r\nHello\r\n
* Array           : *2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

**Exemples d'erreurs** :

```
-ERR wrong number of arguments for 'set' command
-WRONGTYPE Operation against a key holding the wrong kind of value
-OOM command not allowed when used memory > 'maxmemory'
-READONLY You can't write against a read only replica
-NOSCRIPT No matching script. Please use EVAL.
-BUSYKEY Target key name already exists.
-MOVED 3999 127.0.0.1:6381 (Redis Cluster)
-ASK 3999 127.0.0.1:6381 (Redis Cluster)
```

### Stratégies de gestion d'erreurs côté application

#### 1. Retry avec backoff exponentiel

```python
import redis
import time

def redis_operation_with_retry(redis_client, max_retries=3):
    for attempt in range(max_retries):
        try:
            return redis_client.get('key')
        except redis.ConnectionError as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt  # 1s, 2s, 4s
            time.sleep(wait_time)
        except redis.TimeoutError:
            # Timeout : retry immédiat
            continue
```

#### 2. Gestion par type d'erreur

```python
import redis

def safe_incr(redis_client, key):
    try:
        return redis_client.incr(key)
    except redis.ResponseError as e:
        error_msg = str(e)

        if "WRONGTYPE" in error_msg:
            # Type invalide : réinitialiser
            redis_client.delete(key)
            return redis_client.incr(key)

        elif "not an integer" in error_msg:
            # Valeur corrompue : réinitialiser
            redis_client.set(key, 0)
            return redis_client.incr(key)

        elif "OOM" in error_msg:
            # Mémoire pleine : attendre et retry
            time.sleep(1)
            return redis_client.incr(key)

        else:
            # Erreur inconnue : propager
            raise
```

#### 3. Validation préventive

```python
def safe_set_json(redis_client, key, value):
    # Validation avant envoi
    if not isinstance(value, (dict, list)):
        raise ValueError("Value must be dict or list")

    try:
        json_str = json.dumps(value)

        # Vérification taille
        if len(json_str) > 512 * 1024 * 1024:  # 512MB
            raise ValueError("Value too large")

        return redis_client.set(key, json_str)

    except json.JSONEncodeError:
        raise ValueError("Value is not JSON serializable")
```

#### 4. Circuit breaker pattern

```python
class RedisCircuitBreaker:
    def __init__(self, redis_client, failure_threshold=5, timeout=60):
        self.redis_client = redis_client
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN

    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            if self.state == 'HALF_OPEN':
                self.state = 'CLOSED'
                self.failures = 0
            return result

        except redis.ConnectionError:
            self.failures += 1
            self.last_failure_time = time.time()

            if self.failures >= self.failure_threshold:
                self.state = 'OPEN'

            raise
```

### Configuration pour la résilience

```conf
# redis.conf

# Timeout des connexions inactives
timeout 300

# Nombre max de clients
maxclients 10000

# Gestion OOM
maxmemory 2gb
maxmemory-policy allkeys-lru

# Replica read-only (sécurité)
replica-read-only yes

# Protection contre FLUSHALL accidentel
rename-command FLUSHDB ""
rename-command FLUSHALL ""

# Limite requêtes lentes
slowlog-log-slower-than 10000  # 10ms
slowlog-max-len 128

# Protection CPU
lua-time-limit 5000  # 5s max pour scripts Lua
```

## Bonnes pratiques CRUD

### 1. Toujours préférer les opérations atomiques

```bash
# ❌ Mauvais : Race condition
GET counter
# "42"
# ... autre client incrémente entre temps
SET counter 43

# ✅ Bon : Atomique
INCR counter
```

### 2. Utiliser les commandes modernes

```bash
# ❌ Déprécié
SETNX key value
SETEX key 60 value
GETSET key newvalue

# ✅ Moderne
SET key value NX
SET key value EX 60
SET key newvalue GET
```

### 3. Préférer UNLINK à DEL

```bash
# ❌ Peut bloquer
DEL large:set

# ✅ Non bloquant
UNLINK large:set
```

### 4. Utiliser MGET pour lectures multiples

```bash
# ❌ N round-trips
for key in keys:
    value = GET key

# ✅ 1 round-trip
values = MGET key1 key2 key3 ... keyN
```

### 5. Vérifier le type avant opération

```bash
# ❌ Peut échouer
INCR unknown_key

# ✅ Vérifie d'abord
TYPE unknown_key
# Si "string" ou "none" : OK pour INCR
# Si autre type : Gérer l'erreur
```

### 6. Gérer les erreurs de manière granulaire

```python
# ✅ Gestion fine
try:
    redis_client.set(key, value)
except redis.ConnectionError:
    # Problème réseau : retry
    pass
except redis.TimeoutError:
    # Timeout : augmenter timeout ou retry
    pass
except redis.ResponseError as e:
    if "OOM" in str(e):
        # Mémoire pleine : nettoyer ou alerter
        pass
    elif "READONLY" in str(e):
        # Sur replica : router vers master
        pass
```

### 7. Limiter la taille des valeurs

```python
# ✅ Validation côté application
MAX_VALUE_SIZE = 10 * 1024 * 1024  # 10MB

if len(value) > MAX_VALUE_SIZE:
    raise ValueError("Value too large")

redis_client.set(key, value)
```

### 8. Utiliser des namespaces

```bash
# ❌ Collision potentielle
SET user "data"
SET session "data"

# ✅ Namespace clair
SET user:123:profile "data"
SET session:abc:data "data"
```

## Résumé : Matrice CRUD par structure

| Structure | Create | Read | Update | Delete |
|-----------|--------|------|--------|--------|
| **String** | SET, SETNX | GET, MGET | SET, APPEND, INCR | DEL, UNLINK |
| **Hash** | HSET, HSETNX | HGET, HMGET, HGETALL | HSET, HINCRBY | HDEL, DEL, UNLINK |
| **List** | LPUSH, RPUSH | LRANGE, LINDEX | LSET | LREM, LPOP, RPOP, DEL |
| **Set** | SADD | SMEMBERS, SISMEMBER | SADD (upsert) | SREM, SPOP, DEL |
| **Sorted Set** | ZADD | ZRANGE, ZSCORE | ZADD, ZINCRBY | ZREM, ZPOPMIN, DEL |

## Conclusion

La maîtrise des commandes CRUD et de la gestion d'erreurs est fondamentale pour construire des applications Redis robustes. Les points clés à retenir :

- **Atomicité** : Redis garantit l'atomicité de chaque commande individuelle
- **Performance** : Préférer les commandes groupées (MGET, MSET) et asynchrones (UNLINK)
- **Résilience** : Implémenter retry, circuit breaker et validation côté application
- **Modernité** : Utiliser les commandes récentes (SET avec options, UNLINK)
- **Type safety** : Toujours vérifier le type avant opération critique

La section suivante abordera le TTL (Time To Live) et les stratégies d'expiration automatique des données.

⏭️ [TTL (Time To Live) et stratégies d'expiration](/04-cycle-vie-donnee/02-ttl-strategies-expiration.md)
