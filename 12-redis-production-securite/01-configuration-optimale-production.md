🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.1 - Configuration optimale pour la production

## Introduction

La configuration de Redis en production est un exercice d'équilibre entre **performance**, **fiabilité** et **sécurité**. Une configuration inadaptée peut entraîner :

- 🔴 **Pertes de données** (persistance mal configurée)
- 🔴 **Dégradation des performances** (swapping, fragmentation)
- 🔴 **Instabilité** (OOM, évictions excessives)
- 🔴 **Vulnérabilités de sécurité** (exposition, commandes dangereuses)

> **Principe fondamental :** Il n'existe pas de configuration universelle. Chaque use case nécessite un tuning spécifique basé sur vos contraintes de **latence**, **débit**, **durabilité** et **volumétrie**.

---

## Architecture décisionnelle de configuration

```
                    ┌─────────────────────┐
                    │   Use Case Redis    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Type d'utilisation │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼────────┐  ┌────▼───┐   ┌────────▼───────┐
    │  Cache volatile  │  │  Queue │   │ Session Store  │
    │  (pas de persist)│  │ (FIFO) │   │  (durabilité)  │
    └─────────┬────────┘  └───┬────┘   └───────┬────────┘
              │               │                │
    ┌─────────▼────────────────────────────────▼───────┐
    │           Contraintes techniques                 │
    │  • Latence cible (p50, p99, p999)                │
    │  • Débit (ops/sec)                               │
    │  • Volume de données (GB)                        │
    │  • Pattern d'accès (read-heavy, write-heavy)     │
    │  • Tolérance à la perte (RTO/RPO)                │
    └─────────┬──────────────────────────────┬─────────┘
              │                              │
    ┌─────────▼────────┐         ┌───────────▼───────┐
    │  Choix Mémoire   │         │ Choix Persistance │
    │  • maxmemory     │         │ • RDB / AOF       │
    │  • eviction      │         │ • Fréquence sync  │
    └──────────────────┘         └───────────────────┘
```

---

## 📊 Tableau comparatif : Configurations par use case

| Paramètre | Cache Volatile | Session Store | Queue (Job) | Base de données | Analytics |
|-----------|---------------|---------------|-------------|-----------------|-----------|
| **maxmemory-policy** | allkeys-lru | volatile-lru | noeviction | allkeys-lru | noeviction |
| **appendonly** | no | yes | yes | yes | yes |
| **appendfsync** | - | everysec | always | everysec | everysec |
| **save (RDB)** | disabled | 900 1 | 300 10 | 900 1 | 900 1 |
| **maxmemory** | 80% RAM | 70% RAM | 60% RAM | 80% RAM | 90% RAM |
| **timeout** | 300 | 0 | 0 | 300 | 0 |
| **replica** | optionnel | requis | requis | requis | optionnel |
| **persistence loss** | acceptable | inacceptable | inacceptable | inacceptable | tolérable |
| **Latency p99** | < 1ms | < 5ms | < 10ms | < 5ms | < 50ms |

---

## 🎯 Configuration #1 : Cache volatile haute performance

**Use case :** Cache de résultats de requêtes, calculs coûteux, fragments HTML

**Caractéristiques :**
- Perte de données acceptable au redémarrage
- Priorité absolue à la latence
- Éviction automatique selon LRU
- Pas de persistance (ou snapshots occasionnels)

```conf
# ============================================================================
# REDIS PRODUCTION CONFIG - CACHE VOLATILE HAUTE PERFORMANCE
# ============================================================================
# Use case: Cache de résultats, fragments HTML, calculs
# Priorité: Latence < 1ms (p99)
# Durabilité: Acceptable de perdre le cache au redémarrage
# ============================================================================

# ----------------------------------------------------------------------------
# RÉSEAU - Performance maximale
# ----------------------------------------------------------------------------
bind 10.0.1.50
port 6379
protected-mode yes

# Timeout agressif pour libérer les connexions inactives
timeout 300

# TCP keepalive - détection rapide des connexions mortes
tcp-keepalive 60

# Backlog TCP élevé pour haute concurrence
tcp-backlog 2048

# Désactiver TCP_NODELAY pour meilleur débit (légère latence acceptable)
# tcp-nodelay no  # Commenté : on privilégie latence

# ----------------------------------------------------------------------------
# TLS (si requis)
# ----------------------------------------------------------------------------
# tls-port 6379
# port 0
# tls-cert-file /etc/redis/certs/redis.crt
# tls-key-file /etc/redis/certs/redis.key
# tls-ca-cert-file /etc/redis/certs/ca.crt

# ----------------------------------------------------------------------------
# AUTHENTIFICATION
# ----------------------------------------------------------------------------
aclfile /etc/redis/users.acl
# Ou: requirepass VotreMotDePasse2024!

# ----------------------------------------------------------------------------
# MÉMOIRE - Configuration cache
# ----------------------------------------------------------------------------

# 80% de la RAM disponible (ex: 8GB sur machine 10GB)
maxmemory 8gb

# LRU sur toutes les clés (pas de distinction volatile/persistent)
maxmemory-policy allkeys-lru

# Échantillonnage LRU (5 = défaut, bon compromis)
maxmemory-samples 5

# ----------------------------------------------------------------------------
# PERSISTANCE - Minimale ou désactivée
# ----------------------------------------------------------------------------

# Option 1: Pas de persistance (performance maximale)
save ""
appendonly no

# Option 2: Snapshot occasionnel pour warm restart
# save 3600 1    # 1 fois par heure si au moins 1 changement
# save 86400 100 # 1 fois par jour si au moins 100 changements
# stop-writes-on-bgsave-error no  # Continue même si backup échoue
# rdbcompression yes
# rdbchecksum yes
# dbfilename dump.rdb

dir /var/lib/redis

# ----------------------------------------------------------------------------
# RÉPLICATION - Optionnelle pour cache
# ----------------------------------------------------------------------------
# Pour cache distribué géographiquement ou haute disponibilité

# Si replica configuré:
# replicaof <masterip> <masterport>
# masterauth <password>
# replica-read-only yes
# repl-diskless-sync yes  # Pas besoin de RDB pour sync

# Pas de limite de replicas pour écriture (cache tolérant)
# min-replicas-to-write 0

# ----------------------------------------------------------------------------
# SÉCURITÉ
# ----------------------------------------------------------------------------
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG ""
rename-command SHUTDOWN ""
rename-command DEBUG ""
rename-command BGSAVE ""
rename-command BGREWRITEAOF ""

# ----------------------------------------------------------------------------
# CLIENTS
# ----------------------------------------------------------------------------
# Haute limite car cache partagé par beaucoup d'apps
maxclients 50000

# Buffers clients - cache génère peu de données sortantes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# ----------------------------------------------------------------------------
# LAZY FREEING - CRITIQUE pour performance cache
# ----------------------------------------------------------------------------
# Libération asynchrone évite les spikes de latence
lazyfree-lazy-eviction yes     # Évictions en background
lazyfree-lazy-expire yes       # Expirations en background
lazyfree-lazy-server-del yes   # DEL en background
replica-lazy-flush yes         # Flush replica en background
lazyfree-lazy-user-del yes     # DEL utilisateur en background

# ----------------------------------------------------------------------------
# THREADS I/O - Performance élevée
# ----------------------------------------------------------------------------
# 4 threads pour I/O réseau (CPU avec 4+ cores)
io-threads 4
io-threads-do-reads yes

# ----------------------------------------------------------------------------
# DEFRAGMENTATION ACTIVE
# ----------------------------------------------------------------------------
# Important pour cache avec beaucoup d'évictions
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100

# Effort modéré (ne pas impacter latence)
active-defrag-cycle-min 1
active-defrag-cycle-max 15

# ----------------------------------------------------------------------------
# LOGGING
# ----------------------------------------------------------------------------
loglevel notice
logfile "/var/log/redis/cache.log"

# Slow log - seuil bas car cache doit être rapide
slowlog-log-slower-than 5000  # 5ms
slowlog-max-len 256

# ----------------------------------------------------------------------------
# LATENCY MONITORING
# ----------------------------------------------------------------------------
latency-monitor-threshold 50  # 50ms

# ----------------------------------------------------------------------------
# ADVANCED - Optimisation structures
# ----------------------------------------------------------------------------
# Optimisations mémoire pour petites valeurs (cache fragments)
hash-max-listpack-entries 512
hash-max-listpack-value 64
list-max-listpack-size -2
set-max-intset-entries 512
zset-max-listpack-entries 128
zset-max-listpack-value 64

# Active rehashing pour performance
activerehashing yes

# Fréquence serveur
hz 10
dynamic-hz yes

# Jemalloc background thread
jemalloc-bg-thread yes

# ============================================================================
```

**Métriques clés à surveiller :**
```bash
# Hit ratio (doit être > 90% pour cache efficace)
redis-cli INFO stats | grep keyspace_hits
redis-cli INFO stats | grep keyspace_misses

# Évictions (normal pour cache)
redis-cli INFO stats | grep evicted_keys

# Latency (doit rester < 1ms p99)
redis-cli --latency-history

# Fragmentation (doit être < 1.5)
redis-cli INFO memory | grep mem_fragmentation_ratio
```

---

## 🎯 Configuration #2 : Session Store avec durabilité

**Use case :** Stockage de sessions utilisateur, panier e-commerce, état d'authentification

**Caractéristiques :**
- Perte de données **inacceptable** (mauvaise UX)
- Durabilité via AOF + RDB
- TTL automatique sur les sessions
- Latence acceptable (< 5ms p99)

```conf
# ============================================================================
# REDIS PRODUCTION CONFIG - SESSION STORE DURABLE
# ============================================================================
# Use case: Sessions utilisateur, paniers, état auth
# Priorité: Durabilité > Performance
# Latence p99: < 5ms (acceptable pour session)
# ============================================================================

# ----------------------------------------------------------------------------
# RÉSEAU
# ----------------------------------------------------------------------------
bind 10.0.1.50
port 6379
protected-mode yes
timeout 0  # Pas de timeout (connexions persistantes)
tcp-keepalive 300
tcp-backlog 1024

# ----------------------------------------------------------------------------
# TLS - OBLIGATOIRE pour sessions (données sensibles)
# ----------------------------------------------------------------------------
tls-port 6379
port 0  # Désactiver port non-chiffré

tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt
tls-auth-clients yes  # Mutual TLS recommandé

tls-protocols "TLSv1.2 TLSv1.3"
tls-ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
tls-prefer-server-ciphers yes

# ----------------------------------------------------------------------------
# AUTHENTIFICATION - ACLs strictes
# ----------------------------------------------------------------------------
aclfile /etc/redis/users.acl

# ----------------------------------------------------------------------------
# MÉMOIRE - Conservation sessions actives
# ----------------------------------------------------------------------------

# 70% de RAM (marge pour pic de sessions)
maxmemory 7gb

# Éviction uniquement des clés avec TTL (sessions expirées)
maxmemory-policy volatile-lru

# Si aucune clé TTL, refuser nouvelles écritures (éviter perte session)
# maxmemory-policy noeviction  # Alternative stricte

maxmemory-samples 10  # Précision LRU élevée

# ----------------------------------------------------------------------------
# PERSISTANCE - HYBRIDE (RDB + AOF)
# ----------------------------------------------------------------------------

# RDB - Snapshots pour backup rapide
save 900 1      # 15 min, 1 changement
save 300 100    # 5 min, 100 changements
save 60 10000   # 1 min, 10000 changements

stop-writes-on-bgsave-error yes  # Sécurité: arrêt si backup fail
rdbcompression yes
rdbchecksum yes
dbfilename sessions-dump.rdb

# AOF - Log pour durabilité maximale
appendonly yes
appendfilename "sessions-appendonly.aof"

# everysec = bon compromis (durabilité vs performance)
appendfsync everysec

# Réécriture AOF automatique
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
no-appendfsync-on-rewrite no  # Sync même pendant rewrite

# AOF avec timestamps (Redis 7+) pour debugging
aof-use-rdb-preamble yes
aof-timestamp-enabled yes

dir /var/lib/redis/sessions

# ----------------------------------------------------------------------------
# RÉPLICATION - OBLIGATOIRE pour haute dispo
# ----------------------------------------------------------------------------

# Configuration sur replica:
# replicaof <master-ip> <master-port>
# masterauth <password>
# masteruser <username>

replica-read-only yes

# Réplication sans disque (sessions < volumétrie massive)
repl-diskless-sync no
repl-diskless-sync-delay 5

# Backlog généreux pour déconnexions réseau
repl-backlog-size 256mb
repl-backlog-ttl 3600

# Configuration master:
# Minimum 1 replica synchronisé pour écriture
min-replicas-to-write 1
min-replicas-max-lag 10

replica-priority 100

# ----------------------------------------------------------------------------
# SÉCURITÉ
# ----------------------------------------------------------------------------
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG "CONFIG_a8f5f167f44f4964e6c998dee827110c"
rename-command SHUTDOWN ""
rename-command DEBUG ""

# ----------------------------------------------------------------------------
# CLIENTS
# ----------------------------------------------------------------------------
maxclients 10000

client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 512mb 128mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# ----------------------------------------------------------------------------
# LAZY FREEING
# ----------------------------------------------------------------------------
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes      # Important: beaucoup de TTL
lazyfree-lazy-server-del yes
replica-lazy-flush yes
lazyfree-lazy-user-del yes

# ----------------------------------------------------------------------------
# THREADS I/O
# ----------------------------------------------------------------------------
io-threads 4
io-threads-do-reads yes

# ----------------------------------------------------------------------------
# DEFRAGMENTATION
# ----------------------------------------------------------------------------
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
active-defrag-cycle-min 1
active-defrag-cycle-max 25

# ----------------------------------------------------------------------------
# LOGGING
# ----------------------------------------------------------------------------
loglevel notice
logfile "/var/log/redis/sessions.log"

# Slow log
slowlog-log-slower-than 10000  # 10ms
slowlog-max-len 128

# ----------------------------------------------------------------------------
# LATENCY MONITORING
# ----------------------------------------------------------------------------
latency-monitor-threshold 100

# ----------------------------------------------------------------------------
# ADVANCED
# ----------------------------------------------------------------------------
# Sessions souvent stockées en JSON ou Hash
hash-max-listpack-entries 512
hash-max-listpack-value 256  # Augmenté pour JSON sessions

activerehashing yes
hz 10
dynamic-hz yes

# AOF fsync incremental
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

jemalloc-bg-thread yes

# ============================================================================
```

**ACL pour sessions :**
```acl
# users.acl pour session store

user default off

# Admin
user admin on >AdminP@ss2024! ~* &* +@all

# Application - accès sessions uniquement
user session_app on >SessionApp2024! ~session:* +@read +@write +@hash +@string +get +set +hget +hset +hmget +hmset +expire +ttl +del -@dangerous

# Monitoring
user monitoring on >Monitor2024! ~* +@read +info +config|get +slowlog

# Backup
user backup on >Backup2024! ~* +bgsave +lastsave +info
```

**Validation de la durabilité :**
```bash
# Test de durabilité des sessions
redis-cli SET session:test:123 "user_data" EX 3600

# Simuler crash
redis-cli DEBUG SLEEP 30 &
kill -9 $(pgrep redis-server)

# Redémarrer
systemctl start redis

# Vérifier récupération
redis-cli GET session:test:123
# Doit retourner "user_data"
```

---

## 🎯 Configuration #3 : Queue de jobs (Message Broker)

**Use case :** Files d'attente de jobs, workers asynchrones, task queue

**Caractéristiques :**
- Aucune perte de job acceptable
- Ordre FIFO respecté
- Persistance AOF avec sync agressive
- Politique noeviction (refuser si plein)

```conf
# ============================================================================
# REDIS PRODUCTION CONFIG - JOB QUEUE RELIABLE
# ============================================================================
# Use case: Files d'attente jobs, task queue, workers
# Priorité: Fiabilité absolue (0 perte)
# Pattern: LPUSH/BRPOP, RPOPLPUSH
# ============================================================================

# ----------------------------------------------------------------------------
# RÉSEAU
# ----------------------------------------------------------------------------
bind 10.0.1.50
port 6379
protected-mode yes
timeout 0  # Connexions persistantes pour BRPOP
tcp-keepalive 300
tcp-backlog 1024

# ----------------------------------------------------------------------------
# TLS
# ----------------------------------------------------------------------------
tls-port 6379
port 0
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt

# ----------------------------------------------------------------------------
# AUTHENTIFICATION
# ----------------------------------------------------------------------------
aclfile /etc/redis/users.acl

# ----------------------------------------------------------------------------
# MÉMOIRE - Refus si plein
# ----------------------------------------------------------------------------

# 60% de RAM (jobs peuvent s'accumuler pendant pic)
maxmemory 6gb

# CRITIQUE: noeviction pour garantir 0 perte
maxmemory-policy noeviction

# Application doit gérer l'erreur OOM et retry ou scaler

# ----------------------------------------------------------------------------
# PERSISTANCE - DURABILITÉ MAXIMALE
# ----------------------------------------------------------------------------

# RDB - Backup régulier
save 300 10      # 5 min, 10 jobs
save 60 1000     # 1 min, 1000 jobs
save 10 10000    # 10 sec, 10000 jobs (burst)

stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename queue-dump.rdb

# AOF - Mode always pour garantie absolue
appendonly yes
appendfilename "queue-appendonly.aof"

# CRITIQUE: appendfsync always = sync chaque commande
# Impact performance mais 0 perte garantie
appendfsync always

# Alternative plus performante mais risque mini:
# appendfsync everysec  # Perte max 1 seconde de jobs

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# IMPORTANT: pas de sync pendant rewrite AOF
# (durabilité > performance)
no-appendfsync-on-rewrite no

aof-use-rdb-preamble yes
aof-timestamp-enabled yes

dir /var/lib/redis/queue

# ----------------------------------------------------------------------------
# RÉPLICATION - Recommandée
# ----------------------------------------------------------------------------

# Sur replica:
# replicaof <master-ip> <master-port>

replica-read-only yes

repl-diskless-sync yes  # Queues = writes intensives
repl-diskless-sync-delay 5

repl-backlog-size 512mb  # Large backlog pour bursts
repl-backlog-ttl 3600

# Config master: minimum 1 replica avant écriture
min-replicas-to-write 1
min-replicas-max-lag 10

# ----------------------------------------------------------------------------
# SÉCURITÉ
# ----------------------------------------------------------------------------
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG ""
rename-command SHUTDOWN ""
rename-command DEBUG ""

# ----------------------------------------------------------------------------
# CLIENTS
# ----------------------------------------------------------------------------
maxclients 5000  # Workers + producers

# Buffer clients élevé pour BRPOP long-polling
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 512mb 128mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# ----------------------------------------------------------------------------
# LAZY FREEING - Attention avec queues
# ----------------------------------------------------------------------------
# Lazy freeing peut retarder libération mémoire
lazyfree-lazy-eviction no   # N/A car noeviction
lazyfree-lazy-expire yes    # Si jobs avec TTL
lazyfree-lazy-server-del no # Jobs doivent être supprimés sync
replica-lazy-flush yes
lazyfree-lazy-user-del no   # Workers suppriment jobs sync

# ----------------------------------------------------------------------------
# THREADS I/O
# ----------------------------------------------------------------------------
io-threads 2  # Moins de threads (writes intensives)
io-threads-do-reads yes

# ----------------------------------------------------------------------------
# DEFRAGMENTATION
# ----------------------------------------------------------------------------
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100

# Effort faible pendant heures de pointe
active-defrag-cycle-min 1
active-defrag-cycle-max 10

# ----------------------------------------------------------------------------
# LOGGING
# ----------------------------------------------------------------------------
loglevel notice
logfile "/var/log/redis/queue.log"

slowlog-log-slower-than 10000  # 10ms
slowlog-max-len 256

# ----------------------------------------------------------------------------
# LATENCY MONITORING
# ----------------------------------------------------------------------------
latency-monitor-threshold 100

# ----------------------------------------------------------------------------
# ADVANCED
# ----------------------------------------------------------------------------
# Optimisation Lists pour queues
list-max-listpack-size -2  # Lists jusqu'à 8kb en listpack
list-compress-depth 0      # Pas de compression (performance)

activerehashing yes
hz 10
dynamic-hz yes

aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

jemalloc-bg-thread yes

# ============================================================================
```

**ACL pour queue :**
```acl
# users.acl pour job queue

user default off

# Producer - peut ajouter jobs
user producer on >Producer2024! ~queue:* ~queue:*:processing +@write +@list +lpush +rpush +llen -@dangerous

# Worker - peut consommer jobs
user worker on >Worker2024! ~queue:* ~queue:*:processing +@read +@write +@list +brpop +rpoplpush +llen +del -@dangerous

# Dead letter queue handler
user dlq_handler on >DLQ2024! ~queue:*:dlq ~queue:*:failed +@read +@write +@list

# Monitoring
user monitor on >Monitor2024! ~* +@read +info +llen +@keyspace

# Admin
user admin on >Admin2024! ~* +@all
```

**Patterns de queue fiable :**
```bash
# Pattern RPOPLPUSH pour atomicité
# Producer
LPUSH queue:jobs '{"id":1,"task":"process"}'

# Worker (atomic move to processing)
RPOPLPUSH queue:jobs queue:jobs:processing

# Si succès
LREM queue:jobs:processing 1 '{"id":1,"task":"process"}'

# Si échec
RPOPLPUSH queue:jobs:processing queue:jobs:dlq
```

---

## 🎯 Configuration #4 : Base de données principale

**Use case :** Redis comme datastore principal (pas cache)

**Caractéristiques :**
- Durabilité critique
- Mix read/write
- RediSearch ou RedisJSON activés
- Haute disponibilité obligatoire

```conf
# ============================================================================
# REDIS PRODUCTION CONFIG - PRIMARY DATABASE
# ============================================================================
# Use case: Redis comme base de données principale
# Priorité: Durabilité = Performance
# Modules: RedisJSON, RediSearch
# ============================================================================

# ----------------------------------------------------------------------------
# MODULES (Redis Stack)
# ----------------------------------------------------------------------------
loadmodule /opt/redis-stack/lib/redisearch.so
loadmodule /opt/redis-stack/lib/rejson.so
# loadmodule /opt/redis-stack/lib/redistimeseries.so
# loadmodule /opt/redis-stack/lib/redisbloom.so

# ----------------------------------------------------------------------------
# RÉSEAU
# ----------------------------------------------------------------------------
bind 10.0.1.50
port 6379
protected-mode yes
timeout 300
tcp-keepalive 300
tcp-backlog 1024

# ----------------------------------------------------------------------------
# TLS
# ----------------------------------------------------------------------------
tls-port 6379
port 0
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt
tls-auth-clients yes

# ----------------------------------------------------------------------------
# AUTHENTIFICATION
# ----------------------------------------------------------------------------
aclfile /etc/redis/users.acl

# ----------------------------------------------------------------------------
# MÉMOIRE
# ----------------------------------------------------------------------------

# 80% RAM
maxmemory 8gb

# LRU sur toutes clés
maxmemory-policy allkeys-lru

# Haute précision LRU
maxmemory-samples 10

# ----------------------------------------------------------------------------
# PERSISTANCE - HYBRIDE OPTIMISÉ
# ----------------------------------------------------------------------------

# RDB
save 900 1
save 300 10
save 60 10000

stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename database-dump.rdb

# AOF
appendonly yes
appendfilename "database-appendonly.aof"

# everysec = compromis optimal
appendfsync everysec

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 128mb
no-appendfsync-on-rewrite no

aof-use-rdb-preamble yes
aof-timestamp-enabled yes

dir /var/lib/redis/database

# ----------------------------------------------------------------------------
# RÉPLICATION - OBLIGATOIRE
# ----------------------------------------------------------------------------

# Sur replicas:
# replicaof <master-ip> <master-port>

replica-read-only yes

# Diskless pour performance
repl-diskless-sync yes
repl-diskless-sync-delay 5

repl-backlog-size 512mb
repl-backlog-ttl 3600

# Config master
min-replicas-to-write 1
min-replicas-max-lag 10

replica-priority 100

# ----------------------------------------------------------------------------
# SÉCURITÉ
# ----------------------------------------------------------------------------
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG "CONFIG_db_9a8f5f167f44"
rename-command SHUTDOWN ""
rename-command DEBUG ""
rename-command MODULE ""

# ----------------------------------------------------------------------------
# CLIENTS
# ----------------------------------------------------------------------------
maxclients 10000

client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 512mb 128mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# ----------------------------------------------------------------------------
# LAZY FREEING
# ----------------------------------------------------------------------------
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes
lazyfree-lazy-user-del yes

# ----------------------------------------------------------------------------
# THREADS I/O
# ----------------------------------------------------------------------------
io-threads 4
io-threads-do-reads yes

# ----------------------------------------------------------------------------
# DEFRAGMENTATION
# ----------------------------------------------------------------------------
activedefrag yes
active-defrag-ignore-bytes 200mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
active-defrag-cycle-min 5
active-defrag-cycle-max 25

# ----------------------------------------------------------------------------
# LOGGING
# ----------------------------------------------------------------------------
loglevel notice
logfile "/var/log/redis/database.log"

slowlog-log-slower-than 10000
slowlog-max-len 256

latency-monitor-threshold 100

# ----------------------------------------------------------------------------
# ADVANCED
# ----------------------------------------------------------------------------
# Structures optimisées pour mix use cases
hash-max-listpack-entries 512
hash-max-listpack-value 128
list-max-listpack-size -2
set-max-intset-entries 512
zset-max-listpack-entries 128
zset-max-listpack-value 64

# JSON documents peuvent être volumineux
# Augmenter limits si JSON > 1MB
# proto-max-bulk-len 512mb

activerehashing yes
hz 10
dynamic-hz yes

aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

jemalloc-bg-thread yes

# ============================================================================
```

---

## 📏 Guide de dimensionnement : Calcul maxmemory

### Formule de calcul

```
maxmemory = (RAM_TOTALE × 0.75) - (OVERHEAD_SYSTÈME)

Où:
- RAM_TOTALE : Mémoire physique disponible
- 0.75 : Facteur de sécurité (75%)
- OVERHEAD_SYSTÈME : Mémoire OS + monitoring (500MB-1GB)
```

### Exemples concrets

| RAM Serveur | maxmemory recommandé | Justification |
|-------------|---------------------|---------------|
| 2 GB | 1.2 GB | Petit cache, dev/staging |
| 4 GB | 2.5 GB | Cache applicatif |
| 8 GB | 5.5 GB | Session store standard |
| 16 GB | 12 GB | Database ou gros cache |
| 32 GB | 24 GB | High-volume |
| 64 GB | 48 GB | Very high-volume |
| 128 GB | 100 GB | Enterprise |

### Calcul selon volumétrie de données

```bash
# Estimer la taille mémoire de vos données

# 1. Moyenne par clé (exemple: hash user)
redis-cli DEBUG OBJECT user:12345
# Value at: ... serializedlength:523 ...

# 2. Nombre de clés
redis-cli DBSIZE
# (integer) 1500000

# 3. Calcul mémoire données
TAILLE_DONNÉES = DBSIZE × serializedlength_moyen

# 4. Overhead Redis (~20-25% selon structures)
OVERHEAD_REDIS = TAILLE_DONNÉES × 0.25

# 5. Mémoire totale nécessaire
MÉMOIRE_NÉCESSAIRE = TAILLE_DONNÉES + OVERHEAD_REDIS

# 6. maxmemory avec marge 30%
maxmemory = MÉMOIRE_NÉCESSAIRE × 1.3
```

### Exemple de calcul

```
Scénario: Session store e-commerce
- 1 million d'utilisateurs actifs
- Taille moyenne session: 2 KB
- Structure: Hash

Calcul:
1. Données brutes: 1,000,000 × 2 KB = 2 GB
2. Overhead Redis (25%): 2 GB × 0.25 = 0.5 GB
3. Total: 2.5 GB
4. Marge 30%: 2.5 GB × 1.3 = 3.25 GB
5. Overhead OS: +0.5 GB

Résultat: Serveur 4-5 GB RAM minimum
          maxmemory 3.25 GB
```

---

## ⚙️ Tuning avancé selon profil de charge

### Profil READ-HEAVY (90% read, 10% write)

```conf
# Optimisations pour lectures intensives

# Plus de threads I/O
io-threads 6
io-threads-do-reads yes

# Cache des lookups
# (aucun paramètre spécifique, Redis optimisé par défaut)

# Réplication read replicas
# Diriger les reads vers replicas
replica-read-only yes

# Défragmentation moins agressive (peu de writes)
active-defrag-cycle-min 1
active-defrag-cycle-max 10

# Latency optimisée
hz 10
slowlog-log-slower-than 5000  # 5ms
```

### Profil WRITE-HEAVY (30% read, 70% write)

```conf
# Optimisations pour écritures intensives

# AOF avec everysec (pas always, trop lent)
appendfsync everysec

# Moins de threads I/O (GIL lua, transactions)
io-threads 2

# Défragmentation active (beaucoup de mutations)
activedefrag yes
active-defrag-cycle-min 5
active-defrag-cycle-max 50

# Background saves moins fréquents
save 900 1
save 300 100
# Supprimer: save 60 10000

# Backlog réplication large
repl-backlog-size 1gb

# Lazy freeing agressif
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes
```

### Profil SMALL OBJECTS (valeurs < 1KB)

```conf
# Optimisations pour petites valeurs

# Structures compactées
hash-max-listpack-entries 512
hash-max-listpack-value 64
list-max-listpack-size -1  # 4kb
zset-max-listpack-entries 256
zset-max-listpack-value 64

# Défragmentation moins prioritaire
activedefrag no
# Ou:
active-defrag-ignore-bytes 50mb

# Jemalloc optimisé pour petites allocs
# (compilation flag: --with-jemalloc)
```

### Profil LARGE OBJECTS (valeurs > 100KB)

```conf
# Optimisations pour grandes valeurs

# Augmenter proto max bulk
proto-max-bulk-len 512mb

# Client buffer plus large
client-output-buffer-limit normal 256mb 128mb 60

# Structures non compactées
hash-max-listpack-entries 64
hash-max-listpack-value 16
list-max-listpack-size -5  # 64kb

# Défragmentation prudente (éviter copy gros objets)
activedefrag yes
active-defrag-cycle-min 1
active-defrag-cycle-max 5

# AOF rewrite incremental (gros fichiers)
aof-rewrite-incremental-fsync yes
auto-aof-rewrite-min-size 256mb
```

---

## 🔍 Validation de configuration

### 1. Checklist pré-démarrage

```bash
#!/bin/bash
# validate-redis-config.sh

echo "=== REDIS CONFIGURATION VALIDATION ==="

# Vérifier syntaxe redis.conf
redis-server /etc/redis/redis.conf --test-memory 1

# Vérifier THP
THP=$(cat /sys/kernel/mm/transparent_hugepage/enabled)
if [[ $THP == *"[never]"* ]]; then
    echo "✅ THP disabled"
else
    echo "❌ THP enabled - MUST DISABLE"
fi

# Vérifier overcommit
OVERCOMMIT=$(sysctl vm.overcommit_memory | awk '{print $3}')
if [ "$OVERCOMMIT" == "1" ]; then
    echo "✅ Overcommit enabled"
else
    echo "❌ Overcommit disabled - MUST ENABLE"
fi

# Vérifier somaxconn
SOMAXCONN=$(sysctl net.core.somaxconn | awk '{print $3}')
if [ "$SOMAXCONN" -ge "1024" ]; then
    echo "✅ somaxconn sufficient: $SOMAXCONN"
else
    echo "⚠️  somaxconn low: $SOMAXCONN (recommend 65535)"
fi

# Vérifier file limits
NOFILE=$(ulimit -n)
if [ "$NOFILE" -ge "10000" ]; then
    echo "✅ File limits sufficient: $NOFILE"
else
    echo "❌ File limits too low: $NOFILE (recommend 65535)"
fi

# Vérifier permissions dir
REDIS_DIR=$(grep "^dir " /etc/redis/redis.conf | awk '{print $2}')
if [ -d "$REDIS_DIR" ] && [ -w "$REDIS_DIR" ]; then
    echo "✅ Redis dir writable: $REDIS_DIR"
else
    echo "❌ Redis dir not writable: $REDIS_DIR"
fi

# Vérifier ACL file
ACL_FILE=$(grep "^aclfile " /etc/redis/redis.conf | awk '{print $2}')
if [ -f "$ACL_FILE" ]; then
    echo "✅ ACL file exists: $ACL_FILE"
else
    echo "⚠️  ACL file not found: $ACL_FILE"
fi

# Vérifier swap
SWAP=$(free | grep Swap | awk '{print $2}')
if [ "$SWAP" == "0" ]; then
    echo "✅ Swap disabled"
else
    SWAPPINESS=$(sysctl vm.swappiness | awk '{print $3}')
    echo "⚠️  Swap enabled (swappiness: $SWAPPINESS)"
fi

echo "=== END VALIDATION ==="
```

### 2. Tests de charge post-démarrage

```bash
#!/bin/bash
# benchmark-redis.sh

echo "=== REDIS BENCHMARKING ==="

# 1. Latency baseline
echo "1. Measuring latency..."
redis-cli --latency-history -i 1 | head -20

# 2. Throughput test
echo ""
echo "2. Testing throughput..."
redis-benchmark -t set,get -n 1000000 -q -c 50 -d 100

# 3. Pipeline performance
echo ""
echo "3. Testing pipeline..."
redis-benchmark -t set,get -n 1000000 -q -c 50 -d 100 -P 16

# 4. Real-world mix
echo ""
echo "4. Testing mixed workload..."
redis-benchmark -t set,get,incr,lpush,rpush,lpop,rpop,sadd,hset,spop,zadd,zpopmin,lrange,mset -n 100000 -q

# 5. Large values
echo ""
echo "5. Testing large values (10KB)..."
redis-benchmark -t set,get -n 100000 -q -d 10240

# 6. Memory fragmentation check
echo ""
echo "6. Checking memory fragmentation..."
redis-cli INFO memory | grep fragmentation_ratio
```

### 3. Monitoring première heure

```bash
#!/bin/bash
# monitor-first-hour.sh

INTERVAL=60  # 1 minute
DURATION=3600  # 1 hour
ITERATIONS=$((DURATION / INTERVAL))

echo "=== MONITORING REDIS - FIRST HOUR ==="
echo "Timestamp,Memory_Used_MB,Fragmentation,Evictions,Hit_Rate,Connected_Clients,Ops_Per_Sec" > redis_monitoring.csv

for i in $(seq 1 $ITERATIONS); do
    TIMESTAMP=$(date +%s)

    # Collecter métriques
    MEMORY=$(redis-cli INFO memory | grep "used_memory_human" | cut -d: -f2 | tr -d '\r' | sed 's/M//')
    FRAG=$(redis-cli INFO memory | grep "mem_fragmentation_ratio" | cut -d: -f2 | tr -d '\r')
    EVICTIONS=$(redis-cli INFO stats | grep "evicted_keys" | cut -d: -f2 | tr -d '\r')

    HITS=$(redis-cli INFO stats | grep "keyspace_hits" | cut -d: -f2 | tr -d '\r')
    MISSES=$(redis-cli INFO stats | grep "keyspace_misses" | cut -d: -f2 | tr -d '\r')
    HIT_RATE=$(echo "scale=2; $HITS / ($HITS + $MISSES + 1) * 100" | bc)

    CLIENTS=$(redis-cli INFO clients | grep "connected_clients" | cut -d: -f2 | tr -d '\r')
    OPS=$(redis-cli INFO stats | grep "instantaneous_ops_per_sec" | cut -d: -f2 | tr -d '\r')

    # Écrire CSV
    echo "$TIMESTAMP,$MEMORY,$FRAG,$EVICTIONS,$HIT_RATE,$CLIENTS,$OPS" >> redis_monitoring.csv

    echo "[$i/$ITERATIONS] Mem: ${MEMORY}MB, Frag: $FRAG, Hit: ${HIT_RATE}%, Clients: $CLIENTS, Ops/s: $OPS"

    sleep $INTERVAL
done

echo "=== MONITORING COMPLETE ==="
echo "Data saved to redis_monitoring.csv"
```

---

## 🚨 Détection des problèmes de configuration

### Symptômes et diagnostics

| Symptôme | Cause probable | Configuration à vérifier |
|----------|---------------|-------------------------|
| **Latence élevée soudaine** | Swap activé | `vm.swappiness`, `maxmemory` |
| **OOM kills** | maxmemory trop élevé | `maxmemory` < 80% RAM |
| **Évictions excessives** | maxmemory trop bas | Augmenter `maxmemory` ou scaler |
| **Fragmentation > 2.0** | Beaucoup de mutations | Activer `activedefrag` |
| **Réplication lag** | Réplication synchrone | `repl-diskless-sync yes` |
| **Connexions refusées** | maxclients atteint | Augmenter `maxclients` |
| **Lenteur après BGSAVE** | I/O disque saturé | SSD, `rdbcompression no` |
| **Crash réguliers** | THP activé | Désactiver THP |
| **Perte de données** | AOF mal configuré | `appendfsync everysec/always` |

### Commandes de diagnostic

```bash
# 1. Vérifier configuration active
redis-cli CONFIG GET '*' | grep -A 1 "maxmemory\|appendonly\|save"

# 2. Identifier problèmes mémoire
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio|evicted_keys"

# 3. Analyser latence
redis-cli --latency
redis-cli --latency-history
redis-cli --latency-dist

# 4. Identifier slow queries
redis-cli SLOWLOG GET 10

# 5. Vérifier état réplication
redis-cli INFO replication

# 6. Analyser clients connectés
redis-cli CLIENT LIST

# 7. Statistiques commandes
redis-cli INFO commandstats | head -20

# 8. État persistance
redis-cli INFO persistence

# 9. Vérifier erreurs
redis-cli INFO stats | grep rejected_connections
redis-cli INFO stats | grep sync_
```

---

## 📋 Checklist finale avant production

### Checklist niveau 1 : Basique

- [ ] `bind` configuré sur IP privée (pas 0.0.0.0)
- [ ] `protected-mode yes`
- [ ] Authentification activée (ACL ou requirepass)
- [ ] `maxmemory` défini (80% RAM)
- [ ] `maxmemory-policy` adapté au use case
- [ ] Persistance configurée (RDB ou AOF)
- [ ] `dir` avec permissions correctes
- [ ] Commandes dangereuses désactivées
- [ ] THP désactivé système
- [ ] `vm.overcommit_memory = 1`

### Checklist niveau 2 : Sécurité

- [ ] TLS activé et configuré
- [ ] Certificats valides et non-expirés
- [ ] ACLs avec principe least privilege
- [ ] Firewall rules configurées
- [ ] Isolation réseau (VPC/VLAN)
- [ ] Logs audit activés
- [ ] Monitoring configuré
- [ ] Alertes définies
- [ ] Backup automatisé
- [ ] Procédure restore testée

### Checklist niveau 3 : Haute disponibilité

- [ ] Minimum 1 replica configuré
- [ ] `min-replicas-to-write` configuré
- [ ] Sentinel ou Cluster configuré
- [ ] Quorum correct (N/2 + 1)
- [ ] Tests failover effectués
- [ ] Clients configurés pour auto-discovery
- [ ] Load balancer configuré (si applicable)
- [ ] Monitoring réplication lag
- [ ] Runbook failover documenté

### Checklist niveau 4 : Performance

- [ ] `io-threads` configuré (2-6)
- [ ] `lazyfree-lazy-*` activé
- [ ] `activedefrag` configuré
- [ ] Benchmarks effectués
- [ ] Baselines de latence établies
- [ ] Hot keys identifiées et mitigées
- [ ] Slow log configuré et surveillé
- [ ] Client pools configurés correctement
- [ ] Indexes RediSearch optimisés (si applicable)

---

## 🎯 Templates de configuration rapides

### Template : Cache simple

```bash
# Configuration minimaliste cache
maxmemory 2gb
maxmemory-policy allkeys-lru
save ""
appendonly no
bind 127.0.0.1
requirepass YourPassword123!
```

### Template : Production minimale

```bash
# Production minimal viable
bind <private-ip>
port 6379
requirepass <strong-password>
maxmemory 8gb
maxmemory-policy allkeys-lru
save 900 1
appendonly yes
appendfsync everysec
```

### Template : Maximum sécurité

```bash
# Sécurité maximale
bind <private-ip>
tls-port 6379
port 0
tls-cert-file /path/to/cert.crt
tls-key-file /path/to/key.key
aclfile /etc/redis/users.acl
maxmemory 8gb
maxmemory-policy noeviction
appendonly yes
appendfsync always
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
min-replicas-to-write 1
```

---

## 📚 Ressources et références

### Documentation officielle
- [Redis Configuration](https://redis.io/docs/management/config/)
- [Redis Admin Guide](https://redis.io/docs/management/admin/)
- [Redis Persistence](https://redis.io/docs/management/persistence/)

### Outils de validation
- [redis-check-rdb](https://redis.io/docs/management/persistence/#redis-check-rdb) - Vérifier fichiers RDB
- [redis-check-aof](https://redis.io/docs/management/persistence/#redis-check-aof) - Réparer fichiers AOF
- [Redis Doctor](https://redis.io/docs/management/optimization/) - Diagnostics automatiques

### Calculateurs en ligne
- [Redis Memory Calculator](https://redis.io/redis-enterprise/technology/redis-enterprise-flash/)
- [Sizing Calculator](https://docs.redis.com/latest/rs/databases/memory-performance/memory-limit/)

---

**Section suivante :** [12.2 - Sécuriser Redis avec ACLs](./02-securiser-redis-acls.md)

⏭️ [Sécuriser Redis : ACLs (Access Control Lists) granulaires](/12-redis-production-securite/02-securiser-redis-acls.md)
