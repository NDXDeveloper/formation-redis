🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.6 Limitations du Cluster (multi-key operations, transactions)

## Introduction

Redis Cluster, malgré ses nombreux avantages en termes de scaling horizontal et de haute disponibilité, impose des contraintes architecturales significatives qui découlent directement de son modèle distribué Shared-Nothing. Ces limitations ne sont pas des défauts de conception, mais plutôt des compromis inévitables inhérents à tout système distribué qui cherche à maintenir performance, disponibilité et tolérance aux pannes.

Cette section explore en détail les contraintes imposées par Redis Cluster, leurs origines techniques, leurs implications pratiques, et les stratégies pour les contourner ou s'adapter à ces limitations lors de la conception d'applications.

## Origine des limitations : Le problème fondamental

### Le théorème CAP appliqué à Redis Cluster

```
┌─────────────────────────────────────────────────────────────┐
│                    Théorème CAP                             │
│              (Consistency, Availability, Partition)         │
└─────────────────────────────────────────────────────────────┘

Un système distribué ne peut garantir simultanément que 2 des 3 propriétés :

    [C] Consistency          [A] Availability       [P] Partition Tolerance
    ═══════════════          ════════════════       ═══════════════════════
    Tous les nœuds           Le système répond      Le système continue
    voient les mêmes         toujours (même         de fonctionner malgré
    données au même          partiellement)         les pannes réseau
    instant


Redis Cluster choisit : AP (Availability + Partition Tolerance)
═══════════════════════════════════════════════════════════════

    ✓ Availability : Le cluster continue de servir les requêtes
                     même si certains nœuds sont down

    ✓ Partition Tolerance : Le cluster fonctionne malgré
                            les partitions réseau

    ✗ Strong Consistency : Pas de garantie que tous les nœuds
                           voient la même valeur simultanément


Conséquences directes :
═══════════════════════

1. Pas de transactions distribuées (2PC impossible)
   └─> Les transactions sont limitées à un seul nœud

2. Cohérence éventuelle (eventual consistency)
   └─> Fenêtre de réplication asynchrone

3. Opérations multi-clés limitées
   └─> Seulement si toutes les clés sont sur le même nœud

4. Pas de coordination globale
   └─> Chaque nœud est autonome
```

### Partitionnement des données et indépendance des nœuds

```
Le problème fondamental des opérations distribuées :
════════════════════════════════════════════════════

Opération atomique sur 1 nœud (Redis standalone) :
──────────────────────────────────────────────────

    MULTI
    SET key1 "value1"    ┐
    SET key2 "value2"    │ Exécutées atomiquement
    INCR counter         │ sur le MÊME serveur
    EXEC                 ┘

    Garantie : Tout ou rien (atomicité) ✓


Opération sur plusieurs nœuds (Redis Cluster) :
────────────────────────────────────────────────

    MULTI
    SET key1 "value1"    → Nœud A (slot 1234)
    SET key2 "value2"    → Nœud B (slot 5678)  ← Slots différents !
    INCR counter         → Nœud C (slot 9012)
    EXEC

    Problème :
    ├─ Nœud A, B, C sont indépendants
    ├─ Pas de coordinateur centralisé
    ├─ Pas de protocole 2PC (Two-Phase Commit)
    └─> IMPOSSIBLE de garantir l'atomicité

    Résultat : ❌ CROSSSLOT Keys in request don't hash to the same slot
```

## Limitation 1 : Opérations multi-clés

### Commandes multi-clés affectées

```
┌─────────────────────────────────────────────────────────────┐
│        Commandes multi-clés avec limitations cluster        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ COMMANDES GET/SET MULTIPLES                                 │
│ ═══════════════════════════                                 │
│                                                             │
│ MGET key1 key2 key3                                         │
│ ├─ Fonctionne SI : hash(key1) = hash(key2) = hash(key3)     │
│ └─ Échoue SI : clés sur slots/nœuds différents              │
│                                                             │
│ MSET key1 val1 key2 val2 key3 val3                          │
│ ├─ Même contrainte que MGET                                 │
│ └─ Erreur : CROSSSLOT si slots différents                   │
│                                                             │
│ DEL key1 key2 key3                                          │
│ ├─ Peut échouer partiellement si multi-slots                │
│ └─ Retourne nombre de clés supprimées (peut être < 3)       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ OPÉRATIONS ENSEMBLISTES (SETS)                              │
│ ══════════════════════════════                              │
│                                                             │
│ SUNION set1 set2 set3                                       │
│ SINTER set1 set2 set3                                       │
│ SDIFF set1 set2                                             │
│ ├─ Nécessite que tous les sets soient sur le même slot      │
│ └─ Solution : Hash tags ou restructuration                  │
│                                                             │
│ SUNIONSTORE dest set1 set2                                  │
│ SINTERSTORE dest set1 set2                                  │
│ SDIFFSTORE dest set1 set2                                   │
│ ├─ dest + tous les sets doivent être sur même slot          │
│ └─ Sinon : CROSSSLOT                                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ OPÉRATIONS SUR SORTED SETS                                  │
│ ══════════════════════════                                  │
│                                                             │
│ ZUNIONSTORE dest numkeys key1 key2                          │
│ ZINTERSTORE dest numkeys key1 key2                          │
│ ├─ Toutes les clés doivent être co-localisées               │
│ └─ Erreur CROSSSLOT si non respecté                         │
│                                                             │
│ ZDIFF numkeys key1 key2 (Redis 6.2+)                        │
│ ZDIFFSTORE dest numkeys key1 key2                           │
│ ├─ Même contrainte                                          │
│ └─ Co-localisation obligatoire                              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ OPÉRATIONS BITWISE                                          │
│ ══════════════                                              │
│                                                             │
│ BITOP AND/OR/XOR dest key1 key2                             │
│ ├─ dest + toutes les clés sources → même slot               │
│ └─ Utilisé pour opérations sur bitmaps distribués           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ AUTRES COMMANDES AFFECTÉES                                  │
│ ══════════════════════════                                  │
│                                                             │
│ RENAME oldkey newkey                                        │
│ ├─ Fonctionne SI hash(oldkey) = hash(newkey)                │
│ └─ Sinon : ERR No such key (impossible de déplacer)         │
│                                                             │
│ RPOPLPUSH source dest                                       │
│ BRPOPLPUSH source dest timeout                              │
│ ├─ source et dest doivent être sur même slot                │
│ └─ Alternative Redis 6.2+ : LMOVE avec hash tags            │
│                                                             │
│ PFCOUNT key1 key2 key3 (HyperLogLog)                        │
│ PFMERGE dest key1 key2                                      │
│ ├─ Toutes les clés co-localisées                            │
│ └─ HyperLogLog distribué nécessite merge côté client        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Exemples d'échecs et de succès

```bash
# ═══════════════════════════════════════════════════════════
# DÉMONSTRATION DES LIMITATIONS MULTI-CLÉS
# ═══════════════════════════════════════════════════════════

# Se connecter au cluster
redis-cli -c -h 192.168.1.10 -p 6379


# EXEMPLE 1 : MGET avec clés sur slots différents (ÉCHEC)
# ────────────────────────────────────────────────────────

127.0.0.1:6379> SET user:1000 "Alice"
OK

127.0.0.1:6379> SET user:2000 "Bob"
-> Redirected to slot [8834] located at 192.168.1.11:6379
OK

127.0.0.1:6379> SET user:3000 "Charlie"
-> Redirected to slot [12456] located at 192.168.1.12:6379
OK

# Tentative de MGET sur les 3 clés
127.0.0.1:6379> MGET user:1000 user:2000 user:3000
(error) CROSSSLOT Keys in request don't hash to the same slot

# Calcul des slots (pour comprendre)
127.0.0.1:6379> CLUSTER KEYSLOT user:1000
(integer) 5798

127.0.0.1:6379> CLUSTER KEYSLOT user:2000
(integer) 8834

127.0.0.1:6379> CLUSTER KEYSLOT user:3000
(integer) 12456

# Slots différents → MGET impossible


# EXEMPLE 2 : MGET avec hash tags (SUCCÈS)
# ─────────────────────────────────────────

# Utiliser hash tags pour forcer même slot
127.0.0.1:6379> SET {users}:1000 "Alice"
OK

127.0.0.1:6379> SET {users}:2000 "Bob"
OK

127.0.0.1:6379> SET {users}:3000 "Charlie"
OK

# Vérifier les slots
127.0.0.1:6379> CLUSTER KEYSLOT {users}:1000
(integer) 4576

127.0.0.1:6379> CLUSTER KEYSLOT {users}:2000
(integer) 4576  ← Même slot !

127.0.0.1:6379> CLUSTER KEYSLOT {users}:3000
(integer) 4576  ← Même slot !

# MGET fonctionne maintenant
127.0.0.1:6379> MGET {users}:1000 {users}:2000 {users}:3000
1) "Alice"
2) "Bob"
3) "Charlie"


# EXEMPLE 3 : Opérations ensemblistes (ÉCHEC sans hash tag)
# ──────────────────────────────────────────────────────────

127.0.0.1:6379> SADD users:active "alice" "bob"
(integer) 2

127.0.0.1:6379> SADD users:premium "bob" "charlie"
-> Redirected to slot [9876] located at 192.168.1.11:6379
(integer) 2

# Tentative d'intersection
127.0.0.1:6379> SINTER users:active users:premium
(error) CROSSSLOT Keys in request don't hash to the same slot


# EXEMPLE 4 : Opérations ensemblistes (SUCCÈS avec hash tag)
# ───────────────────────────────────────────────────────────

127.0.0.1:6379> SADD {usersets}:active "alice" "bob"
(integer) 2

127.0.0.1:6379> SADD {usersets}:premium "bob" "charlie"
(integer) 2

# Intersection fonctionne
127.0.0.1:6379> SINTER {usersets}:active {usersets}:premium
1) "bob"

# Store fonctionne aussi
127.0.0.1:6379> SINTERSTORE {usersets}:active_premium {usersets}:active {usersets}:premium
(integer) 1


# EXEMPLE 5 : RENAME entre slots différents (ÉCHEC)
# ──────────────────────────────────────────────────

127.0.0.1:6379> SET old:key "value"
OK

# Calcul des slots
127.0.0.1:6379> CLUSTER KEYSLOT old:key
(integer) 1234

127.0.0.1:6379> CLUSTER KEYSLOT new:key
(integer) 5678  ← Slot différent

# RENAME échoue
127.0.0.1:6379> RENAME old:key new:key
(error) ERR No such key

# Solution : hash tags
127.0.0.1:6379> SET {keys}:old "value"
OK

127.0.0.1:6379> RENAME {keys}:old {keys}:new
OK  ✓


# EXEMPLE 6 : DEL multi-clés (succès partiel)
# ────────────────────────────────────────────

127.0.0.1:6379> SET key1 "a"
OK

127.0.0.1:6379> SET key2 "b"
-> Redirected to slot [4998] located at 192.168.1.10:6379
OK

127.0.0.1:6379> SET key3 "c"
-> Redirected to slot [9189] located at 192.168.1.11:6379
OK

# DEL ne retourne pas d'erreur mais peut ne supprimer qu'une partie
127.0.0.1:6379> DEL key1 key2 key3
(integer) 1  ← Seulement 1 clé supprimée (celle du nœud local)

# Pour supprimer toutes les clés, le client doit router vers chaque nœud
```

### Schéma explicatif des limitations multi-clés

```
┌─────────────────────────────────────────────────────────────┐
│          Pourquoi MGET échoue sur multi-slots               │
└─────────────────────────────────────────────────────────────┘

Client envoie : MGET user:1000 user:2000 user:3000
                      │         │         │
                      │         │         │
        Calcul CRC16 sur chaque clé
                      │         │         │
                      ▼         ▼         ▼
                  Slot 5798  Slot 8834  Slot 12456
                      │         │         │
        Mapping slot → nœud
                      │         │         │
                      ▼         ▼         ▼
                  Node A     Node B     Node C
                      │         │         │

Problème : Les 3 clés sont sur 3 nœuds différents
          ↓
Redis Cluster ne peut pas :
├─ Faire un broadcast vers tous les nœuds
├─ Coordonner une transaction multi-nœuds
├─ Garantir l'atomicité de la lecture
└─> ERREUR : CROSSSLOT


Solution : Hash Tags
════════════════════

MGET {users}:1000 {users}:2000 {users}:3000
         └───────────┘
         Hash uniquement "users"
                │
                ▼
            Slot 4576
                │
                ▼
            Node A (unique)
                │
                ▼
          Exécution atomique ✓
```

## Limitation 2 : Transactions (MULTI/EXEC)

### Scope des transactions limité à un nœud

```
┌─────────────────────────────────────────────────────────────┐
│            Transactions dans Redis Cluster                  │
└─────────────────────────────────────────────────────────────┘

Redis Standalone (pas de limitation) :
══════════════════════════════════════

    MULTI
    SET account:1:balance 1000      ┐
    SET account:2:balance 500       │
    INCRBY account:1:balance -100   │ Transaction atomique
    INCRBY account:2:balance +100   │ sur toutes les clés
    EXEC                            ┘

    Toutes les commandes s'exécutent sur le même serveur
    → Atomicité garantie ✓


Redis Cluster (limitation stricte) :
═════════════════════════════════════

    MULTI
    SET account:1:balance 1000      → Slot 1234 (Node A)
    SET account:2:balance 500       → Slot 5678 (Node B)  ← Différent !
    EXEC

    Résultat : ❌ CROSSSLOT

    Les transactions ne peuvent affecter que des clés du MÊME slot


Transaction valide dans un cluster :
═════════════════════════════════════

    MULTI
    SET {account:1}:balance 1000       ┐
    SET {account:1}:email "a@x.com"    │ Toutes les clés
    INCR {account:1}:login_count       │ hash "account:1"
    EXEC                               ┘ → Même slot

    Toutes les clés forcées sur le même slot via hash tag
    → Transaction exécutée atomiquement sur Node A ✓
```

### Exemples de transactions

```bash
# ═══════════════════════════════════════════════════════════
# TRANSACTIONS DANS REDIS CLUSTER
# ═══════════════════════════════════════════════════════════


# EXEMPLE 1 : Transaction cross-slot (ÉCHEC)
# ───────────────────────────────────────────

127.0.0.1:6379> MULTI
OK

127.0.0.1:6379> SET user:1000:name "Alice"
QUEUED

127.0.0.1:6379> SET user:2000:name "Bob"
QUEUED

127.0.0.1:6379> EXEC
(error) CROSSSLOT Keys in request don't hash to the same slot

# Les commandes ont été QUEUED mais pas exécutées


# EXEMPLE 2 : Transaction avec hash tags (SUCCÈS)
# ────────────────────────────────────────────────

127.0.0.1:6379> MULTI
OK

127.0.0.1:6379> SET {user:1000}:name "Alice"
QUEUED

127.0.0.1:6379> SET {user:1000}:email "alice@example.com"
QUEUED

127.0.0.1:6379> INCR {user:1000}:visit_count
QUEUED

127.0.0.1:6379> SADD {user:1000}:tags "premium" "active"
QUEUED

127.0.0.1:6379> EXEC
1) OK
2) OK
3) (integer) 1
4) (integer) 2

# Toutes les opérations exécutées atomiquement ✓


# EXEMPLE 3 : Transaction avec WATCH (Optimistic Locking)
# ────────────────────────────────────────────────────────

# Transférer de l'argent entre deux comptes (même utilisateur)

127.0.0.1:6379> SET {user:1000}:checking 1000
OK

127.0.0.1:6379> SET {user:1000}:savings 500
OK

# Observer les comptes pour optimistic locking
127.0.0.1:6379> WATCH {user:1000}:checking {user:1000}:savings
OK

# Vérifier les soldes
127.0.0.1:6379> GET {user:1000}:checking
"1000"

127.0.0.1:6379> GET {user:1000}:savings
"500"

# Démarrer la transaction
127.0.0.1:6379> MULTI
OK

127.0.0.1:6379> DECRBY {user:1000}:checking 100
QUEUED

127.0.0.1:6379> INCRBY {user:1000}:savings 100
QUEUED

127.0.0.1:6379> EXEC
1) (integer) 900
2) (integer) 600

# Transaction exécutée avec succès ✓


# EXEMPLE 4 : Transaction cross-users (IMPOSSIBLE)
# ─────────────────────────────────────────────────

# Transférer de l'argent entre deux utilisateurs différents
# IMPOSSIBLE avec une transaction atomique dans cluster

127.0.0.1:6379> SET {user:1000}:balance 1000
OK

127.0.0.1:6379> SET {user:2000}:balance 500
OK

# Vérifier les slots
127.0.0.1:6379> CLUSTER KEYSLOT {user:1000}:balance
(integer) 5649

127.0.0.1:6379> CLUSTER KEYSLOT {user:2000}:balance
(integer) 7598  ← Slots différents !

# Transaction impossible
127.0.0.1:6379> MULTI
OK

127.0.0.1:6379> DECRBY {user:1000}:balance 100
QUEUED

127.0.0.1:6379> INCRBY {user:2000}:balance 100
QUEUED

127.0.0.1:6379> EXEC
(error) CROSSSLOT Keys in request don't hash to the same slot


# WORKAROUND : Deux transactions séparées (SANS atomicité globale)
# ─────────────────────────────────────────────────────────────────

# Transaction 1 : Débit
127.0.0.1:6379> DECRBY {user:1000}:balance 100
(integer) 900

# Transaction 2 : Crédit
127.0.0.1:6379> INCRBY {user:2000}:balance 100
(integer) 600

# ATTENTION : Pas d'atomicité entre les deux opérations !
# Si le serveur crash entre les deux, incohérence possible
```

### Limitations du WATCH en cluster

```bash
# ═══════════════════════════════════════════════════════════
# WATCH ET OPTIMISTIC LOCKING EN CLUSTER
# ═══════════════════════════════════════════════════════════


# WATCH fonctionne uniquement pour clés sur le même nœud
# ──────────────────────────────────────────────────────

# Cas 1 : WATCH multi-slot (FONCTIONNE mais avec limitations)
127.0.0.1:6379> WATCH user:1000 user:2000
OK

# WATCH accepte les clés multi-slots MAIS :
# 1. Chaque WATCH est local au nœud
# 2. Pas de coordination entre nœuds
# 3. Race conditions possibles si accès concurrent sur différents nœuds


# Cas 2 : WATCH avec hash tags (RECOMMANDÉ)
127.0.0.1:6379> WATCH {user:1000}:balance {user:1000}:credit_limit
OK

# Garantit que WATCH surveille le même nœud
# → Optimistic locking fiable ✓


# WATCH cross-slot : Danger de race condition
# ────────────────────────────────────────────

# Thread 1 (Node A)                 # Thread 2 (Node B)
WATCH {user:1000}:balance           WATCH {user:2000}:balance
GET {user:1000}:balance             GET {user:2000}:balance
# balance = 1000                    # balance = 500

MULTI                                MULTI
DECRBY {user:1000}:balance 100      INCRBY {user:2000}:balance 100
EXEC ✓                              EXEC ✓

# Les deux transactions réussissent indépendamment
# Mais pas d'atomicité globale entre user:1000 et user:2000
# Possible : débit réussi mais crédit échoue (ou inversement)
```

## Limitation 3 : Scripts Lua

### Restrictions sur les scripts Lua

```
┌─────────────────────────────────────────────────────────────┐
│           Scripts Lua dans Redis Cluster                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ RESTRICTION 1 : Toutes les clés sur le même nœud            │
│ ═════════════════════════════════════════════               │
│                                                             │
│ • Toutes les clés accessibles par le script doivent être    │
│   sur le même nœud/slot                                     │
│ • Les clés doivent être passées explicitement en argument   │
│   (KEYS array)                                              │
│ • Redis vérifie les hash slots avant d'exécuter le script   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ RESTRICTION 2 : Pas de clés dynamiques                      │
│ ══════════════════════════════════════                      │
│                                                             │
│ Interdit :                                                  │
│ ```lua                                                      │
│ local key = "user:" .. ARGV[1]                              │
│ return redis.call("GET", key)  ← Clé construite dynamiquement
│ ```                                                         │
│                                                             │
│ Redis ne peut pas vérifier le slot au préalable             │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ RESTRICTION 3 : KEYS doit contenir toutes les clés          │
│ ═══════════════════════════════════════════════             │
│                                                             │
│ Correct :                                                   │
│ ```lua                                                      │
│ local balance = redis.call("GET", KEYS[1])                  │
│ redis.call("SET", KEYS[2], balance * 2)                     │
│ ```                                                         │
│                                                             │
│ Appel : EVAL script 2 {user:1000}:balance {user:1000}:double│
│                      ↑                                      │
│                      Nombre de clés (KEYS)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Exemples de scripts Lua

```lua
-- ═══════════════════════════════════════════════════════════
-- SCRIPTS LUA DANS REDIS CLUSTER
-- ═══════════════════════════════════════════════════════════


-- EXEMPLE 1 : Script valide avec hash tags
-- ─────────────────────────────────────────

-- Script : Transfert intra-utilisateur
local checking = redis.call("GET", KEYS[1])
local savings = redis.call("GET", KEYS[2])
local amount = tonumber(ARGV[1])

if tonumber(checking) < amount then
    return {err = "Insufficient funds"}
end

redis.call("DECRBY", KEYS[1], amount)
redis.call("INCRBY", KEYS[2], amount)

return {ok = "Transfer successful"}


-- Appel depuis redis-cli
```

```bash
redis-cli -c -h 192.168.1.10 --eval transfer.lua \
    {user:1000}:checking {user:1000}:savings , 100
#   └────────────────────────────────────┘   └──┘
#   KEYS (même hash tag)                     ARGV

# Résultat : {ok = "Transfer successful"} ✓
```

```lua
-- EXEMPLE 2 : Script invalide (clés non co-localisées)
-- ─────────────────────────────────────────────────────

-- Script : Transfert inter-utilisateurs (IMPOSSIBLE)
local balance1 = redis.call("GET", KEYS[1])
local balance2 = redis.call("GET", KEYS[2])
local amount = tonumber(ARGV[1])

redis.call("DECRBY", KEYS[1], amount)
redis.call("INCRBY", KEYS[2], amount)

return "OK"
```

```bash
# Appel
redis-cli --eval transfer_users.lua \
    {user:1000}:balance {user:2000}:balance , 100

# Résultat : (error) CROSSSLOT Keys in request don't hash to the same slot
```

```lua
-- EXEMPLE 3 : Script avec clés dynamiques (DANGEREUX)
-- ───────────────────────────────────────────────────

-- Ce script peut échouer en cluster
local user_id = ARGV[1]
local key = "user:" .. user_id .. ":balance"
return redis.call("GET", key)  -- ❌ Clé construite dynamiquement
```

```bash
# Appel
redis-cli EVAL "..." 0 1000

# Problème : Redis ne peut pas vérifier le slot avant exécution
# Peut fonctionner ou échouer selon où le script est exécuté
```

```lua
-- EXEMPLE 4 : Script correct avec toutes les clés en KEYS
-- ────────────────────────────────────────────────────────

-- Incrémenter un compteur avec limite
local current = tonumber(redis.call("GET", KEYS[1]) or "0")
local limit = tonumber(ARGV[1])

if current >= limit then
    return {err = "Limit reached"}
end

local new_value = redis.call("INCR", KEYS[1])
redis.call("EXPIRE", KEYS[1], ARGV[2])  -- TTL
return new_value
```

```bash
# Appel correct
redis-cli EVAL "..." 1 {ratelimit}:api:user:1000 100 3600
#                     └───────────────────────┘
#                     KEYS[1] avec hash tag

# Fonctionne ✓ car une seule clé, sur un seul slot
```

## Limitation 4 : Autres contraintes importantes

### SELECT database (toujours DB 0)

```
┌─────────────────────────────────────────────────────────────┐
│              Bases de données (DB) en cluster               │
└─────────────────────────────────────────────────────────────┘

Redis Standalone :
══════════════════
• 16 databases (DB 0 à DB 15) par défaut
• SELECT permet de basculer entre DB
• Isolation des données par DB

    127.0.0.1:6379> SELECT 1
    OK
    127.0.0.1:6379[1]> SET key "value in DB 1"
    OK


Redis Cluster :
═══════════════
• Une seule database : DB 0 (toujours)
• SELECT est ignoré ou retourne erreur

    127.0.0.1:6379> SELECT 1
    (error) ERR SELECT is not allowed in cluster mode

• Toutes les clés sont dans DB 0


Impact :
════════
Applications multi-tenant utilisant SELECT doivent :
├─ Préfixer les clés avec tenant ID
├─ Utiliser hash tags pour isolation
└─ Exemple : {tenant:acme}:users vs {tenant:corp}:users


Alternative :
═════════════
• Utiliser plusieurs clusters Redis séparés
• Un cluster par tenant ou environnement
```

### Pub/Sub classique (broadcast à tous les nœuds)

```
┌─────────────────────────────────────────────────────────────┐
│                Pub/Sub dans Redis Cluster                   │
└─────────────────────────────────────────────────────────────┘

Problème avec Pub/Sub classique :
══════════════════════════════════

• Les messages Pub/Sub sont BROADCAST à TOUS les nœuds
• Pas de sharding des channels
• Overhead réseau élevé si nombreux messages


Scénario :
──────────

    Publisher                      Subscribers
       │                         ┌────────────┐
       │  PUBLISH news "hello"   │ Client A   │
       └────────────────────────>│ (Node A)   │
                │                └────────────┘
                │                ┌────────────┐
                └───────────────>│ Client B   │
                │                │ (Node B)   │
                │                └────────────┘
                │                ┌────────────┐
                └───────────────>│ Client C   │
                                 │ (Node C)   │
                                 └────────────┘

• Le message est propagé à tous les nœuds du cluster
• Même si seul Client B a souscrit au channel "news"
• Overhead : 2 messages inutiles (vers Node A et C)


Solution Redis 7+ : Sharded Pub/Sub
════════════════════════════════════

• SSUBSCRIBE / SPUBLISH
• Les channels sont shardés (comme les clés)
• Messages envoyés uniquement au nœud responsable du channel

    SPUBLISH {news}:tech "hello"
              └──────┘
              Hash tag → Slot → Nœud spécifique

• Réduit l'overhead réseau
• Scalabilité améliorée


Recommandation :
════════════════
• Pub/Sub classique : OK pour faible volume
• Sharded Pub/Sub : Recommandé pour forte charge
• Kafka/RabbitMQ : Si besoins avancés (persistence, ordering, etc.)
```

### Scan vs Keys (comportement différent)

```bash
# ═══════════════════════════════════════════════════════════
# SCAN DANS REDIS CLUSTER
# ═══════════════════════════════════════════════════════════


# Redis Standalone : SCAN global
# ───────────────────────────────

127.0.0.1:6379> SCAN 0 MATCH user:* COUNT 100
1) "128"     # Cursor pour prochaine itération
2) 1) "user:1000"
   2) "user:1001"
   3) "user:1002"
   ...


# Redis Cluster : SCAN par nœud
# ──────────────────────────────

# SCAN ne scanne que le nœud LOCAL
127.0.0.1:6379> SCAN 0 MATCH user:* COUNT 100
1) "0"       # Fin du scan pour CE nœud
2) 1) "user:1234"
   2) "user:5678"

# Pour scanner tout le cluster, faire SCAN sur CHAQUE nœud

# Script pour scanner tout le cluster
#!/bin/bash

NODES=(
    "192.168.1.10:6379"
    "192.168.1.11:6379"
    "192.168.1.12:6379"
)

for node in "${NODES[@]}"; do
    echo "Scanning node $node..."
    redis-cli -h ${node%:*} -p ${node#*:} --scan --pattern "user:*"
done


# Alternative : Client intelligent
# ─────────────────────────────────

# Clients comme redis-py-cluster gèrent automatiquement
# le scan de tous les nœuds

from rediscluster import RedisCluster

cluster = RedisCluster(startup_nodes=[{"host": "192.168.1.10", "port": 6379}])

# Scan automatique de tous les nœuds
for key in cluster.scan_iter(match="user:*", count=100):
    print(key)
```

### INFO et commandes de monitoring

```bash
# ═══════════════════════════════════════════════════════════
# COMMANDES D'ADMINISTRATION EN CLUSTER
# ═══════════════════════════════════════════════════════════


# INFO : Par nœud uniquement
# ──────────────────────────

# INFO retourne stats du nœud LOCAL uniquement
redis-cli -h 192.168.1.10 INFO memory
# used_memory:1234567890  ← Seulement ce nœud

# Pour stats globales du cluster, agréger manuellement
total_memory=0
for node in 192.168.1.10 192.168.1.11 192.168.1.12; do
    mem=$(redis-cli -h $node INFO memory | grep "used_memory:" | cut -d: -f2)
    total_memory=$((total_memory + mem))
done
echo "Total cluster memory: $total_memory bytes"


# DBSIZE : Par nœud
# ─────────────────

redis-cli -h 192.168.1.10 DBSIZE
(integer) 10000  ← Clés sur ce nœud seulement

# Total cluster = somme de DBSIZE de tous les nœuds


# CLIENT LIST : Par nœud
# ──────────────────────

redis-cli -h 192.168.1.10 CLIENT LIST
# Liste uniquement les clients connectés à ce nœud


# SLOWLOG : Par nœud
# ──────────────────

redis-cli -h 192.168.1.10 SLOWLOG GET 10
# Slow queries sur ce nœud uniquement

# Pour analyse globale, collecter de tous les nœuds
```

### FLUSHDB / FLUSHALL

```bash
# ═══════════════════════════════════════════════════════════
# FLUSH EN CLUSTER (DANGER !)
# ═══════════════════════════════════════════════════════════


# FLUSHDB / FLUSHALL : Par nœud uniquement
# ─────────────────────────────────────────

# FLUSHALL sur un nœud vide SEULEMENT ce nœud
redis-cli -h 192.168.1.10 FLUSHALL
OK

# Les données des autres nœuds (B, C) restent intactes !


# Pour vider tout le cluster : Exécuter sur TOUS les nœuds
# ─────────────────────────────────────────────────────────

for node in 192.168.1.10 192.168.1.11 192.168.1.12; do
    redis-cli -h $node FLUSHALL
done

# ATTENTION : Aucune confirmation, données perdues immédiatement


# Bonne pratique : Renommer FLUSHALL
# ───────────────────────────────────

# Dans redis.conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""

# Empêche exécution accidentelle
```

## Workarounds et stratégies d'adaptation

### Stratégie 1 : Hash Tags systématiques

```
┌─────────────────────────────────────────────────────────────┐
│            Patterns de Hash Tags recommandés                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. PAR ENTITÉ MÉTIER                                        │
│    ═══════════════════                                      │
│    {user:1000}:profile                                      │
│    {user:1000}:settings                                     │
│    {user:1000}:sessions                                     │
│    {user:1000}:friends:list                                 │
│                                                             │
│    Avantage : Toutes les données d'un user sur même nœud    │
│    Inconvénient : Hotspots si certains users très actifs    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 2. PAR FEATURE/MODULE                                       │
│    ══════════════════════                                   │
│    {cart}:user:1000                                         │
│    {cart}:user:1001                                         │
│    {sessions}:user:1000                                     │
│    {sessions}:user:1001                                     │
│                                                             │
│    Avantage : Isolation par fonctionnalité                  │
│    Inconvénient : Données d'un user dispersées              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 3. PAR TENANT (Multi-tenancy)                               │
│    ═══════════════════════════                              │
│    {tenant:acme}:users                                      │
│    {tenant:acme}:products                                   │
│    {tenant:corp}:users                                      │
│    {tenant:corp}:products                                   │
│                                                             │
│    Avantage : Isolation complète par tenant                 │
│    Inconvénient : Distribution inégale si tenants de tailles variées
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 4. PAR FENÊTRE TEMPORELLE                                   │
│    ══════════════════════════                               │
│    {analytics:2024-12-11}:pageviews                         │
│    {analytics:2024-12-11}:clicks                            │
│    {analytics:2024-12-11}:conversions                       │
│                                                             │
│    Avantage : Agrégations temporelles efficaces             │
│    Inconvénient : Hotspots sur période courante             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Stratégie 2 : Denormalization et duplication

```
Accepter la duplication pour éviter opérations multi-clés
══════════════════════════════════════════════════════════

Exemple : Compteurs d'amis
──────────────────────────

Approche normalisée (problématique en cluster) :
─────────────────────────────────────────────────

SET user:1000:name "Alice"
SADD friendships:1000 "2000" "3000" "4000"
    ↓
Pour afficher profil avec nombre d'amis :
    GET user:1000:name
    SCARD friendships:1000
    ↓ Deux clés potentiellement sur slots différents


Approche dénormalisée (adaptée au cluster) :
─────────────────────────────────────────────

HSET {user:1000}:profile \
    name "Alice" \
    friend_count 3 \
    email "alice@example.com"

SADD {user:1000}:friends "2000" "3000" "4000"
    ↓
Toutes les données sur le même slot
Une transaction peut atomiquement :
    MULTI
    HSET {user:1000}:profile friend_count 4
    SADD {user:1000}:friends "5000"
    EXEC


Trade-off :
───────────
✓ Performance : Lecture en une requête
✓ Atomicité : Transaction possible
✗ Duplication : friend_count stocké 2 fois
✗ Cohérence : Doit maintenir manuellement
```

### Stratégie 3 : Application-Level Transactions

```javascript
// ═══════════════════════════════════════════════════════════
// TRANSACTIONS MULTI-NŒUDS CÔTÉ APPLICATION
// ═══════════════════════════════════════════════════════════

// Contexte : Transfert d'argent entre deux utilisateurs
// Impossible avec MULTI/EXEC (slots différents)


// Pattern 1 : Two-Phase sans rollback automatique
// ────────────────────────────────────────────────

async function transferMoney(fromUser, toUser, amount) {
    const fromKey = `{user:${fromUser}}:balance`;
    const toKey = `{user:${toUser}}:balance`;

    try {
        // Phase 1 : Débit
        const balanceFrom = await redis.get(fromKey);
        if (parseInt(balanceFrom) < amount) {
            throw new Error("Insufficient funds");
        }

        await redis.decrby(fromKey, amount);
        console.log(`Debited ${amount} from ${fromUser}`);

        // Phase 2 : Crédit
        await redis.incrby(toKey, amount);
        console.log(`Credited ${amount} to ${toUser}`);

        return { success: true };

    } catch (error) {
        // PROBLÈME : Pas de rollback automatique
        // Si crédit échoue, le débit est déjà effectué !
        console.error("Transfer failed:", error);

        // Compensation manuelle nécessaire
        // await redis.incrby(fromKey, amount); // Rollback manuel

        return { success: false, error: error.message };
    }
}


// Pattern 2 : Avec log de transaction (pour recovery)
// ────────────────────────────────────────────────────

async function transferMoneyWithLog(fromUser, toUser, amount) {
    const txId = generateTransactionId();
    const txKey = `transaction:${txId}`;

    try {
        // Créer log de transaction
        await redis.hset(txKey,
            'status', 'PENDING',
            'from', fromUser,
            'to', toUser,
            'amount', amount,
            'timestamp', Date.now()
        );

        // Phase 1 : Débit
        await redis.decrby(`{user:${fromUser}}:balance`, amount);
        await redis.hset(txKey, 'status', 'DEBITED');

        // Phase 2 : Crédit
        await redis.incrby(`{user:${toUser}}:balance`, amount);
        await redis.hset(txKey, 'status', 'COMPLETED');

        // Supprimer le log (transaction réussie)
        await redis.del(txKey);

        return { success: true, txId };

    } catch (error) {
        // Marquer comme échouée
        await redis.hset(txKey, 'status', 'FAILED', 'error', error.message);

        // Un job background peut nettoyer les transactions échouées
        return { success: false, txId, error: error.message };
    }
}


// Pattern 3 : Optimistic Locking multi-clés
// ──────────────────────────────────────────

async function transferMoneyOptimistic(fromUser, toUser, amount) {
    const fromKey = `{user:${fromUser}}:balance`;
    const toKey = `{user:${toUser}}:balance`;

    // Lecture initiale
    const balanceFrom = await redis.get(fromKey);
    const balanceTo = await redis.get(toKey);

    if (parseInt(balanceFrom) < amount) {
        throw new Error("Insufficient funds");
    }

    // Version-based optimistic locking
    const versionFrom = await redis.get(`${fromKey}:version`);
    const versionTo = await redis.get(`${toKey}:version`);

    // Tenter mise à jour avec vérification de version
    const pipeline = redis.pipeline();

    // Vérifier + modifier source
    pipeline.watch(`${fromKey}:version`);
    pipeline.get(`${fromKey}:version`);

    const result = await pipeline.exec();

    if (result[1][1] !== versionFrom) {
        throw new Error("Concurrent modification detected on source");
    }

    // Similaire pour destination...
    // (Code simplifié pour illustration)

    return { success: true };
}
```

### Stratégie 4 : Utiliser Redis Streams pour coordination

```bash
# ═══════════════════════════════════════════════════════════
# COORDINATION MULTI-NŒUDS VIA REDIS STREAMS
# ═══════════════════════════════════════════════════════════


# Pattern : Event Sourcing pour opérations cross-slot
# ────────────────────────────────────────────────────

# Au lieu d'une transaction atomique, émettre des events

# Event 1 : Demande de transfert
XADD transfer:requests * \
    from user:1000 \
    to user:2000 \
    amount 100 \
    status PENDING

# Consumer 1 traite le débit
XREADGROUP GROUP processors consumer1 COUNT 1 STREAMS transfer:requests >
# Process: DECRBY {user:1000}:balance 100
# Emit: XADD transfer:events * tx_id 12345 status DEBITED

# Consumer 2 traite le crédit
XREADGROUP GROUP processors consumer2 COUNT 1 STREAMS transfer:events >
# Process: INCRBY {user:2000}:balance 100
# Emit: XADD transfer:events * tx_id 12345 status COMPLETED


# Avantages :
# ├─ Pas de blocage entre nœuds
# ├─ Traçabilité complète (audit log)
# ├─ Retry automatique en cas d'échec
# └─ Scalabilité (consumer groups)

# Inconvénients :
# ├─ Cohérence éventuelle (pas immédiate)
# ├─ Complexité accrue
# └─ Nécessite gestion des compensations
```

## Comparaison avec d'autres systèmes

```
┌─────────────────────────────────────────────────────────────┐
│      Redis Cluster vs Autres Systèmes Distribués            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ REDIS CLUSTER                                               │
│ ═════════════                                               │
│ ✓ Performance : Ultra-rapide (in-memory)                    │
│ ✓ Disponibilité : Haute (failover automatique)              │
│ ✗ Transactions : Limitées à un nœud                         │
│ ✗ Cohérence : Éventuelle uniquement                         │
│ ✗ Operations multi-clés : Hash tags requis                  │
│                                                             │
│ Use case : Cache distribué, session store, leaderboards     │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ MONGODB (SHARDED)                                           │
│ ══════════════════                                          │
│ ✓ Transactions multi-documents : Oui (depuis 4.0)           │
│ ✓ Opérations multi-clés : Sans limitation                   │
│ ✓ Requêtes complexes : Agrégations puissantes               │
│ ✗ Performance : Plus lent que Redis (sur disque)            │
│ ✗ Complexité : Sharding plus complexe                       │
│                                                             │
│ Use case : Base de données documentaire avec transactions   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ CASSANDRA                                                   │
│ ═════════                                                   │
│ ✓ Scalabilité : Excellente (linear scaling)                 │
│ ✓ Disponibilité : Très haute (no SPOF)                      │
│ ✗ Transactions : Très limitées (LWT seulement)              │
│ ✗ Cohérence : Tunable mais souvent éventuelle               │
│ ✗ Latence : Plus élevée que Redis                           │
│                                                             │
│ Use case : Time-series, IoT, analytics à très grande échelle│
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ POSTGRESQL (avec Citus)                                     │
│ ═══════════════════════                                     │
│ ✓ Transactions : ACID complètes                             │
│ ✓ Opérations multi-clés : Toutes supportées                 │
│ ✓ Requêtes : SQL standard                                   │
│ ✗ Performance : Pas in-memory                               │
│ ✗ Scalabilité : Plus limitée que Redis Cluster              │
│                                                             │
│ Use case : Base transactionnelle avec sharding              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Checklist de décision architecturale

```
Avant d'utiliser Redis Cluster, se poser ces questions :
═════════════════════════════════════════════════════════

☐ Mon application nécessite-t-elle des transactions ACID strictes ?
  └─> Si OUI : Considérer PostgreSQL ou MongoDB

☐ Ai-je besoin d'opérations multi-clés fréquentes sur clés non liées ?
  └─> Si OUI : Restructurer données ou considérer autre système

☐ Puis-je modéliser mes données avec hash tags de co-localisation ?
  └─> Si OUI : Redis Cluster viable

☐ Puis-je accepter la cohérence éventuelle ?
  └─> Si NON : Considérer base traditionnelle avec ACID

☐ La performance sub-milliseconde est-elle critique ?
  └─> Si OUI : Redis Cluster excellent choix

☐ Ai-je besoin de SELECT multi-database ?
  └─> Si OUI : Redis standalone ou refactorer avec préfixes

☐ Mon cas d'usage est-il principalement lecture ?
  └─> Si OUI : Redis Cluster avec replicas idéal

☐ Ai-je besoin de Pub/Sub à haute fréquence ?
  └─> Si OUI : Utiliser Sharded Pub/Sub (Redis 7+) ou Kafka

☐ Puis-je tolérer une fenêtre de perte de données (~secondes) ?
  └─> Si NON : Configurer persistence stricte + WAIT
```

## Conclusion

Les limitations de Redis Cluster ne sont pas des bugs, mais des conséquences inévitables de choix architecturaux privilégiant performance, disponibilité et tolérance aux pannes. Comprendre ces limitations est essentiel pour :

1. **Concevoir correctement** : Modéliser données avec hash tags appropriés
2. **Éviter les pièges** : Ne pas tenter d'opérations impossibles
3. **Choisir judicieusement** : Utiliser Redis Cluster pour cas d'usage adaptés
4. **Implémenter proprement** : Utiliser workarounds quand nécessaire

Redis Cluster excelle pour :
- Cache distribué haute performance
- Session stores
- Leaderboards et compteurs
- Rate limiting
- Cas d'usage où données sont naturellement partitionnables

Redis Cluster n'est PAS adapté pour :
- Transactions complexes multi-entités
- Applications nécessitant ACID strict
- Opérations analytiques complexes nécessitant JOINs

---

**Points clés à retenir :**

- **CROSSSLOT** : Erreur quand clés sur slots différents
- **Hash tags `{...}`** : Solution pour co-localiser données liées
- **Transactions** : MULTI/EXEC limité à un seul nœud
- **Scripts Lua** : Toutes clés doivent être co-localisées
- **SELECT DB** : Pas supporté, toujours DB 0
- **Pub/Sub** : Broadcast à tous nœuds (utiliser Sharded Pub/Sub si possible)
- **SCAN** : Par nœud uniquement, scanner tous pour vue complète
- **Workarounds** : Hash tags, dénormalisation, transactions app-level

La prochaine section (11.7) explorera le routing des requêtes (client-side vs proxy-based).

⏭️ [Client-side routing vs Proxy-based routing](/11-architecture-distribuee-scaling/07-client-side-vs-proxy-routing.md)
