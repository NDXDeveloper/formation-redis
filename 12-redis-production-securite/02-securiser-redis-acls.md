🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.2 - Sécuriser Redis : ACLs (Access Control Lists) granulaires

## Introduction

Les **Access Control Lists (ACLs)** introduites dans Redis 6.0 représentent une révolution dans la sécurisation de Redis. Elles permettent un contrôle granulaire des permissions au niveau :

- 🔐 **Utilisateur** - Authentification multi-utilisateurs
- 🔑 **Commandes** - Restriction par commande ou catégorie
- 🗝️ **Clés** - Accès limité par pattern de clés
- 📡 **Channels** - Restriction Pub/Sub par pattern

> **⚠️ Avant Redis 6.0 :** Seulement `requirepass` (un seul mot de passe global)
> **✅ Depuis Redis 6.0 :** Système ACL complet avec utilisateurs, rôles et permissions granulaires

---

## Évolution de la sécurité Redis

```
┌─────────────────────────────────────────────────────────────┐
│         REDIS < 6.0 : Sécurité basique                      │
│                                                             │
│  • requirepass : 1 mot de passe global                      │
│  • Tous les utilisateurs = même permissions                 │
│  • Pas de traçabilité par utilisateur                       │
│  • rename-command : solution hacky                          │
└─────────────────────────────────────────────────────────────┘
                            ↓ Upgrade
┌─────────────────────────────────────────────────────────────┐
│         REDIS 6.0+ : ACLs granulaires                       │
│                                                             │
│  ✅ Multi-utilisateurs avec mots de passe séparés           │
│  ✅ Permissions par commande (+@read, -FLUSHDB, etc.)       │
│  ✅ Permissions par pattern de clés (~user:*, ~cache:*)     │
│  ✅ Permissions Pub/Sub par channel (&notifications:*)      │
│  ✅ Audit trail par utilisateur                             │
│  ✅ Gestion dynamique (pas besoin de restart)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Syntaxe ACL complète

### Structure d'une règle ACL

```
user <username> <flags> <passwords> <keys> <channels> <commands>
```

### 1. Flags utilisateur

| Flag | Description | Exemple |
|------|-------------|---------|
| `on` | Utilisateur activé | `user alice on` |
| `off` | Utilisateur désactivé | `user bob off` |
| `nopass` | Pas de mot de passe requis (DANGEREUX) | `user dev nopass` |
| `resetpass` | Supprimer tous les mots de passe | `user alice resetpass` |
| `reset` | Reset complet des permissions | `user alice reset` |

### 2. Définition des mots de passe

| Syntaxe | Description | Exemple |
|---------|-------------|---------|
| `>password` | Ajouter mot de passe en clair | `>MyP@ssw0rd` |
| `#<hash>` | Ajouter hash SHA256 | `#<sha256-hash>` |
| `<password` | Supprimer mot de passe | `<OldPassword` |
| `nopass` | Aucun mot de passe | `nopass` |
| `resetpass` | Supprimer tous les mots de passe | `resetpass` |

```bash
# Générer hash SHA256 pour mot de passe
echo -n "MySecurePassword" | sha256sum
# Output: 3c9909afec25354d551dae21590bb26e38d53f2173b8d3dc3eee4c047e7ab1c1

# Utiliser dans ACL
user alice on #3c9909afec25354d551dae21590bb26e38d53f2173b8d3dc3eee4c047e7ab1c1 ...
```

### 3. Patterns de clés

| Syntaxe | Description | Exemple | Clés matchées |
|---------|-------------|---------|---------------|
| `~*` | Toutes les clés | `~*` | Toutes |
| `~key` | Clé exacte | `~user:123` | `user:123` uniquement |
| `~prefix:*` | Pattern wildcard | `~session:*` | `session:abc`, `session:123` |
| `~prefix:*:suffix` | Pattern complexe | `~user:*:profile` | `user:1:profile`, `user:2:profile` |
| `%R~pattern` | Read-only sur pattern | `%R~cache:*` | Lecture seule sur `cache:*` |
| `%W~pattern` | Write-only sur pattern | `%W~logs:*` | Écriture seule sur `logs:*` |
| `%RW~pattern` | Read-Write sur pattern | `%RW~data:*` | Lecture-écriture sur `data:*` |

### 4. Patterns de channels (Pub/Sub)

| Syntaxe | Description | Exemple |
|---------|-------------|---------|
| `&*` | Tous les channels | `&*` |
| `&channel` | Channel exact | `&notifications` |
| `&prefix:*` | Pattern wildcard | `&events:*` |

### 5. Permissions de commandes

#### Syntaxe de base

| Syntaxe | Description | Exemple |
|---------|-------------|---------|
| `+<command>` | Autoriser commande | `+get +set +hget` |
| `-<command>` | Interdire commande | `-del -flushdb` |
| `+@<category>` | Autoriser catégorie | `+@read +@write` |
| `-@<category>` | Interdire catégorie | `-@admin -@dangerous` |
| `+<cmd>\|<subcommand>` | Autoriser sous-commande | `+config\|get` |
| `allcommands` | Toutes les commandes | `allcommands` ou `+@all` |
| `nocommands` | Aucune commande | `nocommands` ou `-@all` |

#### Catégories de commandes principales

| Catégorie | Description | Commandes incluses |
|-----------|-------------|-------------------|
| `@read` | Lectures seules | GET, MGET, HGET, LRANGE, SMEMBERS, etc. |
| `@write` | Écritures | SET, HSET, LPUSH, SADD, ZADD, etc. |
| `@admin` | Administration | SAVE, BGSAVE, SHUTDOWN, CONFIG, etc. |
| `@dangerous` | Dangereuses | FLUSHALL, FLUSHDB, KEYS, etc. |
| `@fast` | Rapides O(1) | GET, SET, HGET, HSET, etc. |
| `@slow` | Potentiellement lentes | KEYS, SMEMBERS (grands sets), etc. |
| `@keyspace` | Modifient keyspace | DEL, RENAME, EXPIRE, etc. |
| `@string` | Opérations strings | GET, SET, APPEND, INCR, etc. |
| `@list` | Opérations lists | LPUSH, RPOP, LRANGE, etc. |
| `@set` | Opérations sets | SADD, SMEMBERS, SINTER, etc. |
| `@sortedset` | Opérations sorted sets | ZADD, ZRANGE, ZINCRBY, etc. |
| `@hash` | Opérations hashes | HGET, HSET, HMGET, HINCRBY, etc. |
| `@pubsub` | Pub/Sub | PUBLISH, SUBSCRIBE, PSUBSCRIBE, etc. |
| `@stream` | Streams | XADD, XREAD, XREADGROUP, etc. |
| `@scripting` | Scripts Lua | EVAL, EVALSHA, SCRIPT, etc. |
| `@transaction` | Transactions | MULTI, EXEC, DISCARD, WATCH, etc. |
| `@connection` | Connexion | AUTH, SELECT, CLIENT, PING, etc. |
| `@server` | Serveur | INFO, CONFIG, DEBUG, etc. |

### Liste complète des catégories

```bash
# Obtenir toutes les catégories disponibles
redis-cli ACL CAT

# Obtenir les commandes d'une catégorie
redis-cli ACL CAT read
redis-cli ACL CAT dangerous
redis-cli ACL CAT admin
```

---

## 🎯 Exemples d'ACLs par rôle

### 1. Utilisateur Admin - Accès complet

```acl
# Administrateur système - Tous les droits
user admin on >AdminSecureP@ss2024! ~* &* +@all

# Ou de manière plus explicite:
user admin on >AdminSecureP@ss2024! allkeys allchannels allcommands

# Avec hash SHA256 (recommandé):
user admin on #<sha256-hash-du-password> ~* &* +@all
```

**Usage :**
- Gestion du serveur Redis
- Configuration
- Troubleshooting
- Opérations de maintenance

### 2. Application Read-Write - Accès données applicatives

```acl
# Application avec lecture/écriture sur ses données uniquement
user app_backend on >AppBackendP@ss2024! ~app:* ~cache:* +@read +@write +@hash +@string +@list +@set +@sortedset -@dangerous -@admin

# Détail des permissions:
# ~app:*     : Accès à toutes les clés commençant par "app:"
# ~cache:*   : Accès à toutes les clés commençant par "cache:"
# +@read     : Toutes les commandes de lecture
# +@write    : Toutes les commandes d'écriture
# +@hash     : Opérations sur hashes
# +@string   : Opérations sur strings
# +@list     : Opérations sur lists
# +@set      : Opérations sur sets
# +@sortedset: Opérations sur sorted sets
# -@dangerous: Bloquer commandes dangereuses (KEYS, FLUSHDB, etc.)
# -@admin    : Bloquer commandes admin
```

**Usage :**
- Backend applicatif
- Services métier
- APIs

### 3. Application Read-Only - Lecture seule

```acl
# Application avec lecture seule (ex: reporting, analytics)
user app_readonly on >ReadOnlyP@ss2024! ~app:* ~cache:* ~analytics:* +@read +get +mget +hget +hmget +lrange +smembers +zrange +exists +ttl +type -@write -@admin -@dangerous

# Alternative plus stricte (read-only marker):
user app_readonly on >ReadOnlyP@ss2024! %R~app:* %R~cache:* %R~analytics:* +@read -@write -@admin -@dangerous
```

**Usage :**
- Dashboards analytics
- Services de reporting
- Read replicas

### 4. Cache Manager - Gestion du cache uniquement

```acl
# Service gérant uniquement le cache avec TTL
user cache_manager on >CacheManagerP@ss2024! ~cache:* +@read +@write +@string +get +set +mget +mset +del +expire +ttl +exists -@admin -@dangerous -scan -keys

# Détail:
# ~cache:*   : Uniquement clés de cache
# +get, +set : Lecture/écriture basique
# +del       : Suppression de cache
# +expire    : Gestion TTL
# -scan, -keys: Pas de scan (utiliser SCAN côté app si nécessaire)
```

**Usage :**
- Service de cache distribué
- Invalidation de cache
- Warm-up de cache

### 5. Session Store - Gestion des sessions

```acl
# Service gérant les sessions utilisateur
user session_manager on >SessionP@ss2024! ~session:* +@read +@write +@hash +@string +get +set +hget +hset +hmget +hmset +hdel +del +expire +ttl +exists -@admin -@dangerous

# Alternative avec RedisJSON (si Redis Stack):
user session_manager on >SessionP@ss2024! ~session:* +@read +@write +json.get +json.set +json.del +expire +del -@admin -@dangerous
```

**Usage :**
- Stockage sessions web
- Paniers e-commerce
- État d'authentification

### 6. Queue Worker - Traitement de queues

```acl
# Worker consommant des jobs depuis une queue
user queue_worker on >QueueWorkerP@ss2024! ~queue:jobs ~queue:jobs:processing ~queue:jobs:failed +@read +@write +@list +brpop +rpoplpush +lpush +llen +lrange +del -@admin -@dangerous

# Worker producer (création de jobs uniquement):
user queue_producer on >QueueProducerP@ss2024! ~queue:jobs +@write +@list +lpush +llen -@admin -@dangerous

# Dead letter queue handler:
user dlq_handler on >DLQHandlerP@ss2024! ~queue:jobs:failed ~queue:jobs:dlq +@read +@write +@list -@admin -@dangerous
```

**Usage :**
- Workers asynchrones
- Job processing
- Task queues

### 7. Pub/Sub Publisher - Publication uniquement

```acl
# Service publiant des événements
user event_publisher on >PublisherP@ss2024! &events:* &notifications:* +@pubsub +publish -subscribe -psubscribe -@admin -@dangerous

# Avec accès clés pour metadata:
user event_publisher on >PublisherP@ss2024! ~metadata:* &events:* &notifications:* +@read +@pubsub +publish +get +hget -@admin -@dangerous
```

**Usage :**
- Event sourcing
- Notifications temps réel
- Event bus

### 8. Pub/Sub Subscriber - Souscription uniquement

```acl
# Service consommant des événements
user event_subscriber on >SubscriberP@ss2024! &events:* &notifications:* +@pubsub +subscribe +psubscribe +unsubscribe +punsubscribe -publish -@admin -@dangerous

# Avec accès lecture pour traitement:
user event_subscriber on >SubscriberP@ss2024! ~data:* &events:* +@read +@pubsub +subscribe +psubscribe +get +hget -publish -@admin -@dangerous
```

**Usage :**
- Event listeners
- Consumers temps réel
- Websocket backends

### 9. Monitoring User - Métriques uniquement

```acl
# Utilisateur pour monitoring (Prometheus, Grafana, etc.)
user monitoring on >MonitoringP@ss2024! ~* +@read +info +ping +client|list +slowlog +latency +memory|stats +config|get -@write -@admin -@dangerous

# Alternative plus stricte (commandes explicites):
user monitoring on >MonitoringP@ss2024! ~* +info +ping +client|list +client|info +slowlog|get +slowlog|len +latency|latest +latency|history +latency|doctor +memory|stats +memory|doctor +config|get -@write -@admin
```

**Usage :**
- Prometheus exporters
- Monitoring tools
- Health checks

### 10. Backup User - Sauvegarde uniquement

```acl
# Utilisateur pour backups
user backup on >BackupP@ss2024! ~* +bgsave +save +lastsave +info|persistence -@write -@admin -@dangerous -shutdown -debug

# Avec accès RDB inspection:
user backup on >BackupP@ss2024! ~* +bgsave +lastsave +info +config|get -@write -@admin
```

**Usage :**
- Scripts de backup
- Disaster recovery
- Snapshots automatiques

### 11. Replication User - Réplication master-replica

```acl
# Utilisateur pour réplication (sur le master)
user replicator on >ReplicatorP@ss2024! ~* +psync +replconf +ping +info +@read -@write -@admin -@dangerous

# Alternative (Redis gère automatiquement):
# Le plus souvent, utiliser masteruser et masterauth dans redis.conf
```

**Usage :**
- Réplication master → replica
- Configuration automatique Sentinel

### 12. Developer - Développement local uniquement

```acl
# Développeur avec accès étendu mais pas destructeur
user developer on >DevP@ss2024! ~dev:* ~test:* +@all -flushdb -flushall -shutdown -config|set -config|rewrite -debug -@dangerous

# Plus permissif (dev local):
user developer on >DevP@ss2024! ~* +@all -flushdb -flushall -shutdown
```

**Usage :**
- Environnement développement
- Tests locaux
- Debugging

### 13. Rate Limiter - Compteurs uniquement

```acl
# Service de rate limiting
user rate_limiter on >RateLimitP@ss2024! ~ratelimit:* +@read +@write +@string +get +set +incr +incrby +decr +decrby +expire +ttl +del -@admin -@dangerous

# Avec sorted sets pour sliding window:
user rate_limiter on >RateLimitP@ss2024! ~ratelimit:* +@read +@write +@string +@sortedset +get +set +incr +zadd +zremrangebyscore +zcard +expire -@admin -@dangerous
```

**Usage :**
- API rate limiting
- Throttling
- Quota management

### 14. Analytics - Écritures compteurs uniquement

```acl
# Service d'analytics avec HyperLogLog et compteurs
user analytics on >AnalyticsP@ss2024! ~analytics:* ~stats:* +@read +@write +@hyperloglog +pfadd +pfcount +pfmerge +incr +incrby +hincrby +zincrby +get +hget +zrange -@admin -@dangerous

# Avec TimeSeries (Redis Stack):
user analytics on >AnalyticsP@ss2024! ~analytics:* ~timeseries:* +@read +@write +ts.add +ts.get +ts.range +ts.mrange +incr +incrby -@admin -@dangerous
```

**Usage :**
- Compteurs de visite
- Métriques applicatives
- Analytics temps réel

### 15. Search User - RediSearch uniquement

```acl
# Utilisateur pour recherche full-text (RediSearch)
user search_user on >SearchP@ss2024! ~* +@read +ft.search +ft.aggregate +ft.info +ft.explain +ft.spellcheck +get +hget +hmget -@write -ft.create -ft.dropindex -@admin -@dangerous

# Avec indexation (write):
user search_indexer on >IndexerP@ss2024! ~* +@read +@write +ft.create +ft.dropindex +ft.alter +ft.sugadd +ft.sugdel +hset +hmset +del -@admin -@dangerous
```

**Usage :**
- Moteur de recherche
- Indexation documents
- Full-text search

---

## 🔐 Fichier ACL complet pour production

### users.acl - Configuration production complète

```acl
# ============================================================================
# REDIS ACL CONFIGURATION - PRODUCTION
# ============================================================================
# Date: 2024
# Environment: Production
# Redis Version: 7.2+
# ============================================================================

# ----------------------------------------------------------------------------
# DÉSACTIVER UTILISATEUR DEFAULT - SÉCURITÉ CRITIQUE
# ----------------------------------------------------------------------------
# L'utilisateur "default" existe toujours et doit être désactivé en production
user default off resetpass -@all

# ----------------------------------------------------------------------------
# ADMINISTRATEURS
# ----------------------------------------------------------------------------

# Admin principal - Accès complet
user admin on #5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8 ~* &* +@all resetchannels resetkeys
# Password hash pour: "AdminSecureP@ss2024!"

# Admin secondaire - Backup admin
user admin_backup on #a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3 ~* &* +@all
# Password hash pour: "BackupAdminP@ss2024!"

# SRE avec restrictions - Pas de FLUSHALL/FLUSHDB
user sre on #8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92 ~* &* +@all -flushall -flushdb -shutdown
# Password hash pour: "SRE_P@ss2024!"

# ----------------------------------------------------------------------------
# APPLICATIONS BACKEND
# ----------------------------------------------------------------------------

# Application principale - Read/Write sur données app
user app_backend on #c7d5b4f4c437f5c9d8e9a3b1f6e8d2c4a5b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1 ~app:* ~cache:* +@read +@write +@hash +@string +@list +@set +@sortedset +@hyperloglog +expire +ttl +exists +del -@admin -@dangerous -keys -scan
# Password hash: Généré pour app_backend

# Application analytics - Read-only + écritures compteurs
user app_analytics on #d8e6c5b3a4f6c8d7e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2 ~analytics:* ~stats:* +@read +@hyperloglog +pfadd +pfcount +incr +incrby +hincrby +zincrby -@admin -@dangerous
# Password hash: Généré pour app_analytics

# Microservice User Management
user svc_users on #e9f7d6c4b5a7d9e8f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3 ~user:* ~profile:* +@read +@write +@hash +@string +get +set +hget +hset +hmget +hmset +hdel +del +expire +exists -@admin -@dangerous

# Microservice Orders
user svc_orders on #f0a8e7d5c6b8e0f9a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4 ~order:* ~cart:* +@read +@write +@hash +@list +@string -@admin -@dangerous

# ----------------------------------------------------------------------------
# CACHE MANAGEMENT
# ----------------------------------------------------------------------------

# Cache service - Gestion cache HTTP/API
user cache_manager on #a1b9f8e6d7c9f0a8b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4 ~cache:* +@read +@write +@string +get +set +mget +mset +setex +del +expire +ttl -@admin -@dangerous -keys

# CDN cache invalidator
user cdn_invalidator on #b2c0a9f7e8d0a9b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5 ~cache:cdn:* +del +expire -@all +del +expire

# ----------------------------------------------------------------------------
# SESSION MANAGEMENT
# ----------------------------------------------------------------------------

# Session store principal
user session_store on #c3d1b0a8f9e1b0c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6 ~session:* +@read +@write +@hash +@string +get +set +hget +hset +hmget +hmset +hdel +del +expire +ttl -@admin -@dangerous

# Session cleaner (TTL management)
user session_cleaner on #d4e2c1b9a0f2c1d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7 ~session:* +@read +expire +del +ttl +scan +exists -@admin

# ----------------------------------------------------------------------------
# QUEUE WORKERS
# ----------------------------------------------------------------------------

# Job producer - Création de jobs
user queue_producer on #e5f3d2c0b1a3d2e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8 ~queue:jobs ~queue:jobs:priority +@write +@list +lpush +rpush +llen -@admin -@dangerous

# Job consumer - Traitement de jobs
user queue_worker on #f6a4e3d1c2b4e3f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9 ~queue:jobs ~queue:jobs:processing ~queue:jobs:failed +@read +@write +@list +brpop +rpoplpush +lpush +lrange +llen +del -@admin -@dangerous

# Dead letter queue handler
user dlq_handler on #a7b5f4e2d3c5f4a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0 ~queue:jobs:failed ~queue:jobs:dlq +@read +@write +@list +@string -@admin -@dangerous

# ----------------------------------------------------------------------------
# PUB/SUB
# ----------------------------------------------------------------------------

# Event publisher
user event_publisher on #b8c6a5f3e4d6a5b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1 &events:* &notifications:* +@pubsub +publish -subscribe -psubscribe -@admin

# Event subscriber
user event_subscriber on #c9d7b6a4f5e7b6c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2 ~data:* &events:* &notifications:* +@read +@pubsub +subscribe +psubscribe +unsubscribe +get +hget -publish -@admin

# Websocket backend
user websocket_backend on #d0e8c7b5a6f8c7d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3 ~ws:* &websocket:* +@read +@write +@pubsub +@string +@hash -@admin -@dangerous

# ----------------------------------------------------------------------------
# RATE LIMITING
# ----------------------------------------------------------------------------

# Rate limiter service
user rate_limiter on #e1f9d8c6b7a9d8e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4 ~ratelimit:* +@read +@write +@string +@sortedset +get +set +incr +incrby +zadd +zcount +zremrangebyscore +expire +del -@admin -@dangerous

# ----------------------------------------------------------------------------
# SEARCH (RediSearch)
# ----------------------------------------------------------------------------

# Search query user
user search_user on #f2a0e9d7c8b0e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5 ~* +@read +ft.search +ft.aggregate +ft.info +ft.explain +get +hget +hmget -@write -@admin

# Search indexer
user search_indexer on #a3b1f0e8d9c1f0a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6 ~* +@read +@write +ft.create +ft.alter +ft.dropindex +ft.sugadd +hset +hmset +del -@admin -@dangerous

# ----------------------------------------------------------------------------
# MONITORING & OPERATIONS
# ----------------------------------------------------------------------------

# Monitoring (Prometheus, Grafana)
user monitoring on #b4c2a1f9e0d2a1b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7 ~* +@read +info +ping +client|list +client|info +slowlog|get +latency|latest +memory|stats +config|get -@write -@admin

# Backup user
user backup on #c5d3b2a0f1e3b2c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8 ~* +bgsave +lastsave +info|persistence +config|get -@write -@admin -shutdown

# Healthcheck user (minimal permissions)
user healthcheck on #d6e4c3b1a2f4c3d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9 ~* +ping +info|server +info|replication -@all +ping +info

# ----------------------------------------------------------------------------
# DÉVELOPPEMENT (Non-production)
# ----------------------------------------------------------------------------

# Developer (uniquement dev/staging)
# user developer on >DevP@ss2024! ~dev:* ~test:* +@all -flushdb -flushall -shutdown -config|set -debug -@dangerous

# ============================================================================
# NOTES:
# - Tous les passwords sont hashés en SHA256
# - L'utilisateur "default" DOIT être désactivé
# - Adapter les patterns de clés selon votre nomenclature
# - Tester les ACLs avant déploiement production
# - Auditer régulièrement les permissions
# ============================================================================
```

---

## 🔄 Migration depuis requirepass vers ACLs

### Étape 1 : Audit de l'existant

```bash
# 1. Identifier tous les clients connectés
redis-cli CLIENT LIST

# 2. Analyser les commandes utilisées
redis-cli INFO commandstats | head -30

# 3. Identifier les patterns de clés
redis-cli --scan --pattern '*' | head -100 | sed 's/:[^:]*$//' | sort | uniq
```

### Étape 2 : Créer users.acl avec migration

```acl
# users.acl - Phase de migration

# PHASE 1: Garder default actif temporairement
user default on >OldRequirePass123! ~* &* +@all

# PHASE 2: Créer nouveaux utilisateurs
user admin on >NewAdminP@ss2024! ~* &* +@all
user app_backend on >AppP@ss2024! ~app:* ~cache:* +@read +@write +@hash +@string -@admin -@dangerous
user monitoring on >MonitorP@ss2024! ~* +@read +info +ping -@write -@admin

# PHASE 3: Déploiement progressif
# Les anciens clients continuent avec "default"
# Nouveaux déploiements utilisent utilisateurs spécifiques
```

### Étape 3 : Configuration redis.conf

```conf
# redis.conf pendant migration

# Charger ACLs
aclfile /etc/redis/users.acl

# Garder requirepass temporairement (synchronisé avec user default)
requirepass OldRequirePass123!

# Après migration complète:
# Supprimer requirepass, désactiver default dans users.acl
```

### Étape 4 : Migration progressive des clients

```python
# Exemple de migration côté application (Python)

# AVANT (avec requirepass)
import redis
r = redis.Redis(host='localhost', port=6379, password='OldRequirePass123!')

# APRÈS (avec ACLs)
r = redis.Redis(
    host='localhost',
    port=6379,
    username='app_backend',  # Nouveau paramètre
    password='AppP@ss2024!'
)

# Test de connexion
try:
    r.ping()
    print("✅ Connexion ACL réussie")
except redis.AuthenticationError as e:
    print(f"❌ Erreur auth: {e}")
```

### Étape 5 : Validation et désactivation default

```bash
# 1. Vérifier que tous les clients utilisent ACLs
redis-cli ACL GETUSER default
# Si "on" et des connexions actives → migration pas finie

# 2. Identifier les clients utilisant encore default
redis-cli CLIENT LIST | grep 'user=default'

# 3. Une fois migration complète, désactiver default
redis-cli ACL SETUSER default off
# Ou dans users.acl:
# user default off resetpass -@all

# 4. Redémarrer Redis
systemctl restart redis

# 5. Vérifier
redis-cli ACL GETUSER default
# Devrait afficher: "flags: off -@all"
```

---

## 🛠️ Gestion des ACLs en production

### Commandes essentielles

```bash
# ============================================================================
# GESTION DES ACLs - COMMANDES REDIS
# ============================================================================

# Lister tous les utilisateurs
redis-cli ACL LIST

# Détails d'un utilisateur spécifique
redis-cli ACL GETUSER username

# Vérifier l'utilisateur actuellement connecté
redis-cli ACL WHOAMI

# ============================================================================
# CRÉATION D'UTILISATEURS
# ============================================================================

# Créer utilisateur via CLI
redis-cli ACL SETUSER newuser on >password ~prefix:* +@read +@write -@dangerous

# Créer utilisateur avec hash
echo -n "SecurePassword" | sha256sum
redis-cli ACL SETUSER newuser on #<hash> ~* +@all

# ============================================================================
# MODIFICATION D'UTILISATEURS
# ============================================================================

# Ajouter permission
redis-cli ACL SETUSER alice +get +set

# Retirer permission
redis-cli ACL SETUSER alice -del -flushdb

# Ajouter pattern de clés
redis-cli ACL SETUSER alice ~newprefix:*

# Ajouter catégorie de commandes
redis-cli ACL SETUSER alice +@hash

# Changer mot de passe
redis-cli ACL SETUSER alice >NewPassword123!
redis-cli ACL SETUSER alice resetpass >NewPassword123!  # Reset puis nouveau

# ============================================================================
# DÉSACTIVATION / SUPPRESSION
# ============================================================================

# Désactiver utilisateur (garde config)
redis-cli ACL SETUSER bob off

# Réactiver
redis-cli ACL SETUSER bob on

# Supprimer utilisateur complètement
redis-cli ACL DELUSER bob

# ============================================================================
# SAUVEGARDE ACLs
# ============================================================================

# Sauvegarder ACLs actuelles dans aclfile
redis-cli ACL SAVE

# Recharger depuis aclfile
redis-cli ACL LOAD

# ============================================================================
# DEBUGGING
# ============================================================================

# Voir toutes les catégories
redis-cli ACL CAT

# Voir commandes d'une catégorie
redis-cli ACL CAT read
redis-cli ACL CAT dangerous

# Tester permissions utilisateur (Redis 7.0+)
redis-cli ACL DRYRUN username get key
redis-cli ACL DRYRUN username flushdb

# Log des authentifications échouées
redis-cli ACL LOG
redis-cli ACL LOG 10  # 10 derniers

# Reset log
redis-cli ACL LOG RESET

# ============================================================================
# GÉNÉRATION DE PASSWORDS
# ============================================================================

# Générer password aléatoire (Redis 6.2+)
redis-cli ACL GENPASS
redis-cli ACL GENPASS 32  # 32 bytes = 256 bits

# Exemple d'utilisation:
NEW_PASS=$(redis-cli ACL GENPASS 32)
redis-cli ACL SETUSER alice >$NEW_PASS
```

### Script de gestion des utilisateurs

```bash
#!/bin/bash
# manage-redis-users.sh - Gestion ACLs Redis

REDIS_CLI="redis-cli -a $REDIS_ADMIN_PASSWORD"

function list_users() {
    echo "=== REDIS USERS ==="
    $REDIS_CLI ACL LIST
}

function create_user() {
    local username=$1
    local password=$2
    local keys=$3
    local commands=$4

    echo "Creating user: $username"
    $REDIS_CLI ACL SETUSER "$username" on ">$password" "$keys" $commands

    if [ $? -eq 0 ]; then
        echo "✅ User $username created"
        $REDIS_CLI ACL SAVE
    else
        echo "❌ Failed to create user $username"
    fi
}

function delete_user() {
    local username=$1

    echo "Deleting user: $username"
    $REDIS_CLI ACL DELUSER "$username"

    if [ $? -eq 0 ]; then
        echo "✅ User $username deleted"
        $REDIS_CLI ACL SAVE
    else
        echo "❌ Failed to delete user $username"
    fi
}

function disable_user() {
    local username=$1

    echo "Disabling user: $username"
    $REDIS_CLI ACL SETUSER "$username" off

    if [ $? -eq 0 ]; then
        echo "✅ User $username disabled"
        $REDIS_CLI ACL SAVE
    fi
}

function show_user() {
    local username=$1

    echo "=== USER DETAILS: $username ==="
    $REDIS_CLI ACL GETUSER "$username"
}

function rotate_password() {
    local username=$1
    local new_password=$2

    echo "Rotating password for: $username"
    $REDIS_CLI ACL SETUSER "$username" resetpass ">$new_password"

    if [ $? -eq 0 ]; then
        echo "✅ Password rotated for $username"
        $REDIS_CLI ACL SAVE
        echo "⚠️  Update application configuration with new password"
    fi
}

function audit_permissions() {
    echo "=== ACL AUDIT ==="

    # Utilisateur default encore actif?
    DEFAULT_STATUS=$($REDIS_CLI ACL GETUSER default | grep "flags")
    if [[ $DEFAULT_STATUS == *"on"* ]]; then
        echo "⚠️  WARNING: user 'default' is still ACTIVE"
    else
        echo "✅ User 'default' is disabled"
    fi

    # Utilisateurs avec +@all?
    echo ""
    echo "Users with full permissions (+@all):"
    $REDIS_CLI ACL LIST | grep "+@all"

    # Utilisateurs sans password?
    echo ""
    echo "Users without password (nopass):"
    $REDIS_CLI ACL LIST | grep "nopass"
}

# Menu principal
case "$1" in
    list)
        list_users
        ;;
    create)
        create_user "$2" "$3" "$4" "$5"
        ;;
    delete)
        delete_user "$2"
        ;;
    disable)
        disable_user "$2"
        ;;
    show)
        show_user "$2"
        ;;
    rotate)
        rotate_password "$2" "$3"
        ;;
    audit)
        audit_permissions
        ;;
    *)
        echo "Usage: $0 {list|create|delete|disable|show|rotate|audit}"
        echo ""
        echo "Examples:"
        echo "  $0 list"
        echo "  $0 create myuser 'Pass123!' '~app:*' '+@read +@write -@dangerous'"
        echo "  $0 show myuser"
        echo "  $0 rotate myuser 'NewPass456!'"
        echo "  $0 disable myuser"
        echo "  $0 delete myuser"
        echo "  $0 audit"
        exit 1
        ;;
esac
```

---

## 🔍 Audit et monitoring des ACLs

### 1. Script d'audit de sécurité

```bash
#!/bin/bash
# audit-redis-acls.sh

echo "=== REDIS ACL SECURITY AUDIT ==="
echo "Date: $(date)"
echo ""

# 1. Utilisateur default
echo "1. Checking default user status..."
DEFAULT_FLAGS=$(redis-cli ACL GETUSER default | grep "flags" | cut -d' ' -f2)
if [[ $DEFAULT_FLAGS == *"on"* ]]; then
    echo "❌ CRITICAL: User 'default' is ACTIVE - MUST BE DISABLED"
else
    echo "✅ User 'default' is disabled"
fi

# 2. Utilisateurs sans mot de passe
echo ""
echo "2. Checking for users without password..."
NOPASS_USERS=$(redis-cli ACL LIST | grep "nopass")
if [ -n "$NOPASS_USERS" ]; then
    echo "❌ WARNING: Users with nopass found:"
    echo "$NOPASS_USERS"
else
    echo "✅ All users have passwords"
fi

# 3. Utilisateurs avec permissions complètes
echo ""
echo "3. Checking for users with full permissions..."
FULL_PERM_USERS=$(redis-cli ACL LIST | grep -E "\+@all|allcommands")
if [ -n "$FULL_PERM_USERS" ]; then
    echo "⚠️  Users with full permissions:"
    echo "$FULL_PERM_USERS"
else
    echo "✅ No users with unrestricted access (except expected admins)"
fi

# 4. Utilisateurs avec accès à commandes dangereuses
echo ""
echo "4. Checking for dangerous command access..."
DANGEROUS_USERS=$(redis-cli ACL LIST | grep -v "\-@dangerous")
echo "Users with access to dangerous commands:"
echo "$DANGEROUS_USERS"

# 5. Historique des échecs d'authentification
echo ""
echo "5. Recent authentication failures..."
redis-cli ACL LOG 10

# 6. Clients connectés par utilisateur
echo ""
echo "6. Connected clients by user..."
redis-cli CLIENT LIST | awk '{for(i=1;i<=NF;i++){if($i~/^user=/){print $i}}}' | sort | uniq -c

# 7. Permissions par utilisateur
echo ""
echo "7. Detailed permissions per user..."
USERS=$(redis-cli ACL LIST | awk '{print $2}')
for user in $USERS; do
    echo ""
    echo "--- User: $user ---"
    redis-cli ACL GETUSER $user | head -20
done

echo ""
echo "=== AUDIT COMPLETE ==="
```

### 2. Monitoring continu

```python
#!/usr/bin/env python3
# monitor_acl_violations.py

import redis
import time
from datetime import datetime

# Connexion Redis avec user monitoring
r = redis.Redis(
    host='localhost',
    port=6379,
    username='monitoring',
    password='MonitorP@ss2024!',
    decode_responses=True
)

def check_acl_violations():
    """Monitor ACL log for authentication failures"""

    acl_log = r.execute_command('ACL', 'LOG', '100')

    violations = []
    for entry in acl_log:
        if entry['reason'] in ['auth', 'command']:
            violations.append({
                'timestamp': datetime.fromtimestamp(entry['age-seconds']),
                'username': entry.get('username', 'unknown'),
                'reason': entry['reason'],
                'context': entry.get('context', ''),
                'object': entry.get('object', '')
            })

    return violations

def alert_security_team(violations):
    """Send alert if violations detected"""
    if not violations:
        return

    print(f"⚠️  {len(violations)} ACL violation(s) detected:")
    for v in violations:
        print(f"  - [{v['timestamp']}] User: {v['username']}, "
              f"Reason: {v['reason']}, Object: {v['object']}")

def monitor_user_activity():
    """Monitor active users"""
    clients = r.execute_command('CLIENT', 'LIST')

    user_counts = {}
    for client_info in clients.split('\n'):
        if 'user=' in client_info:
            user = [p for p in client_info.split() if 'user=' in p][0].split('=')[1]
            user_counts[user] = user_counts.get(user, 0) + 1

    print(f"\n📊 Active connections by user:")
    for user, count in user_counts.items():
        print(f"  - {user}: {count} connection(s)")

def check_dangerous_commands():
    """Check if dangerous commands are being used"""
    info = r.info('commandstats')

    dangerous = ['flushall', 'flushdb', 'keys', 'config_set', 'shutdown']
    found_dangerous = {}

    for key, value in info.items():
        cmd = key.replace('cmdstat_', '')
        if cmd in dangerous:
            found_dangerous[cmd] = value['calls']

    if found_dangerous:
        print(f"\n⚠️  Dangerous commands executed:")
        for cmd, calls in found_dangerous.items():
            print(f"  - {cmd.upper()}: {calls} time(s)")

if __name__ == '__main__':
    print("=== Starting ACL Monitoring ===")

    while True:
        try:
            violations = check_acl_violations()
            alert_security_team(violations)

            monitor_user_activity()
            check_dangerous_commands()

            print(f"\n[{datetime.now()}] Monitoring... (Ctrl+C to stop)")
            time.sleep(60)  # Check every minute

        except KeyboardInterrupt:
            print("\n\n=== Monitoring stopped ===")
            break
        except Exception as e:
            print(f"Error: {e}")
            time.sleep(10)
```

---

## 🧪 Tests de validation des ACLs

### Test Suite complète

```bash
#!/bin/bash
# test-acls.sh - Validation des ACLs

REDIS_HOST="localhost"
REDIS_PORT="6379"

# Couleurs pour output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

function test_user() {
    local username=$1
    local password=$2
    local test_cmd=$3
    local expected=$4  # "success" ou "failure"

    result=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT --user $username --pass $password $test_cmd 2>&1)

    if [[ $expected == "success" ]]; then
        if [[ $result != *"NOPERM"* ]] && [[ $result != *"NOAUTH"* ]]; then
            echo -e "${GREEN}✅ PASS${NC}: $username can execute: $test_cmd"
            return 0
        else
            echo -e "${RED}❌ FAIL${NC}: $username should be able to execute: $test_cmd"
            echo "   Result: $result"
            return 1
        fi
    else
        if [[ $result == *"NOPERM"* ]] || [[ $result == *"NOAUTH"* ]]; then
            echo -e "${GREEN}✅ PASS${NC}: $username correctly blocked from: $test_cmd"
            return 0
        else
            echo -e "${RED}❌ FAIL${NC}: $username should be blocked from: $test_cmd"
            echo "   Result: $result"
            return 1
        fi
    fi
}

echo "=== REDIS ACL TEST SUITE ==="
echo ""

# Test 1: Default user doit être désactivé
echo "Test 1: Default user disabled"
test_user "default" "" "PING" "failure"
echo ""

# Test 2: Admin a accès complet
echo "Test 2: Admin full access"
test_user "admin" "AdminSecureP@ss2024!" "PING" "success"
test_user "admin" "AdminSecureP@ss2024!" "SET test:key value" "success"
test_user "admin" "AdminSecureP@ss2024!" "GET test:key" "success"
test_user "admin" "AdminSecureP@ss2024!" "INFO" "success"
echo ""

# Test 3: App backend - accès limité
echo "Test 3: App backend limited access"
test_user "app_backend" "AppBackendP@ss2024!" "PING" "success"
test_user "app_backend" "AppBackendP@ss2024!" "SET app:test value" "success"
test_user "app_backend" "AppBackendP@ss2024!" "GET app:test" "success"
test_user "app_backend" "AppBackendP@ss2024!" "SET other:test value" "failure"  # Pattern mismatch
test_user "app_backend" "AppBackendP@ss2024!" "FLUSHDB" "failure"  # Dangerous command
test_user "app_backend" "AppBackendP@ss2024!" "CONFIG GET *" "failure"  # Admin command
echo ""

# Test 4: Cache manager - seulement cache
echo "Test 4: Cache manager cache-only access"
test_user "cache_manager" "CacheManagerP@ss2024!" "SET cache:page:1 html" "success"
test_user "cache_manager" "CacheManagerP@ss2024!" "GET cache:page:1" "success"
test_user "cache_manager" "CacheManagerP@ss2024!" "DEL cache:page:1" "success"
test_user "cache_manager" "CacheManagerP@ss2024!" "SET app:data value" "failure"  # Wrong prefix
test_user "cache_manager" "CacheManagerP@ss2024!" "KEYS *" "failure"  # Blocked command
echo ""

# Test 5: Monitoring - read only
echo "Test 5: Monitoring read-only"
test_user "monitoring" "MonitoringP@ss2024!" "INFO" "success"
test_user "monitoring" "MonitoringP@ss2024!" "PING" "success"
test_user "monitoring" "MonitoringP@ss2024!" "GET app:test" "success"
test_user "monitoring" "MonitoringP@ss2024!" "SET test value" "failure"  # No write
test_user "monitoring" "MonitoringP@ss2024!" "FLUSHDB" "failure"
echo ""

# Test 6: Queue worker
echo "Test 6: Queue worker"
test_user "queue_worker" "QueueWorkerP@ss2024!" "LPUSH queue:jobs job1" "success"
test_user "queue_worker" "QueueWorkerP@ss2024!" "BRPOP queue:jobs 1" "success"
test_user "queue_worker" "QueueWorkerP@ss2024!" "SET other:key value" "failure"  # Wrong prefix
echo ""

# Test 7: Session store
echo "Test 7: Session store"
test_user "session_store" "SessionP@ss2024!" "HSET session:user:123 name John" "success"
test_user "session_store" "SessionP@ss2024!" "HGET session:user:123 name" "success"
test_user "session_store" "SessionP@ss2024!" "EXPIRE session:user:123 3600" "success"
test_user "session_store" "SessionP@ss2024!" "SET app:data value" "failure"  # Wrong prefix
echo ""

# Test 8: Rate limiter
echo "Test 8: Rate limiter"
test_user "rate_limiter" "RateLimitP@ss2024!" "INCR ratelimit:user:123" "success"
test_user "rate_limiter" "RateLimitP@ss2024!" "EXPIRE ratelimit:user:123 60" "success"
test_user "rate_limiter" "RateLimitP@ss2024!" "SET other:key value" "failure"
echo ""

echo "=== TEST SUITE COMPLETE ==="
```

---

## 📋 Checklists de sécurité ACL

### Checklist déploiement initial

- [ ] **Utilisateur default désactivé**
  ```bash
  redis-cli ACL GETUSER default | grep "flags: off"
  ```

- [ ] **Tous les utilisateurs ont des mots de passe forts**
  - Minimum 16 caractères
  - Mix majuscules, minuscules, chiffres, symboles
  - Pas de mots du dictionnaire

- [ ] **Mots de passe hashés dans users.acl**
  ```bash
  grep -c "#" /etc/redis/users.acl  # Doit être > 0
  ```

- [ ] **Permissions minimales par utilisateur**
  - Principe du moindre privilège
  - Pas de +@all sauf admins
  - Patterns de clés restrictifs

- [ ] **Commandes dangereuses bloquées**
  ```bash
  redis-cli ACL LIST | grep -E "flushall|flushdb|keys|config|shutdown"
  ```

- [ ] **ACL LOG activé et surveillé**
  ```bash
  redis-cli ACL LOG
  ```

- [ ] **Backup users.acl automatisé**
  ```bash
  cp /etc/redis/users.acl /backup/users.acl.$(date +%Y%m%d)
  ```

### Checklist audit mensuel

- [ ] **Revoir tous les utilisateurs actifs**
  ```bash
  redis-cli ACL LIST | grep "on"
  ```

- [ ] **Supprimer utilisateurs inutilisés**
  ```bash
  redis-cli CLIENT LIST | grep user=
  # Comparer avec ACL LIST
  ```

- [ ] **Vérifier échecs d'authentification**
  ```bash
  redis-cli ACL LOG 100 | grep -c "auth"
  ```

- [ ] **Rotation des mots de passe applicatifs**
  - Tous les 90 jours recommandé
  - Coordination avec équipes dev

- [ ] **Audit des permissions**
  ```bash
  ./audit-redis-acls.sh > audit_$(date +%Y%m%d).log
  ```

- [ ] **Vérifier users.acl vs ACLs actives**
  ```bash
  diff <(cat /etc/redis/users.acl) <(redis-cli ACL LIST)
  ```

### Checklist incident de sécurité

- [ ] **Identifier l'utilisateur compromis**
  ```bash
  redis-cli ACL LOG | grep -A5 "auth"
  redis-cli CLIENT LIST | grep user=<username>
  ```

- [ ] **Désactiver immédiatement l'utilisateur**
  ```bash
  redis-cli ACL SETUSER <username> off
  redis-cli ACL SAVE
  ```

- [ ] **Killer connexions actives**
  ```bash
  redis-cli CLIENT KILL USER <username>
  ```

- [ ] **Analyser commandes exécutées**
  ```bash
  redis-cli SLOWLOG GET 100
  redis-cli MONITOR  # Temporairement
  ```

- [ ] **Rotation password ou recréation utilisateur**
  ```bash
  redis-cli ACL SETUSER <username> resetpass ><new-password>
  ```

- [ ] **Notifier équipes concernées**
  - Équipe sécurité
  - Équipes dev utilisant cet utilisateur
  - Management si données sensibles

- [ ] **Post-mortem et amélioration ACLs**
  - Documenter l'incident
  - Renforcer les permissions si nécessaire

---

## 🚨 Troubleshooting ACL

### Problèmes courants et solutions

#### 1. "NOAUTH Authentication required"

```bash
# Cause: Pas authentifié ou mauvais password
# Solution:
redis-cli --user username --pass password PING

# Vérifier que l'utilisateur existe:
redis-cli --user admin --pass adminpass ACL GETUSER username
```

#### 2. "NOPERM this user has no permissions to run the 'command' command"

```bash
# Cause: Utilisateur n'a pas la permission pour cette commande
# Solution: Vérifier et ajouter permission

# Voir permissions actuelles:
redis-cli ACL GETUSER username

# Ajouter permission:
redis-cli ACL SETUSER username +command
redis-cli ACL SAVE
```

#### 3. "NOPERM this user has no permissions to access one of the keys"

```bash
# Cause: Pattern de clés ne matche pas
# Solution: Vérifier et ajuster pattern

# Voir patterns actuels:
redis-cli ACL GETUSER username | grep "keys"

# Ajouter pattern:
redis-cli ACL SETUSER username ~newpattern:*
redis-cli ACL SAVE
```

#### 4. Client refuse de se connecter après migration ACL

```bash
# Vérifier configuration client:
# - Username correct?
# - Password correct?
# - Client supporte AUTH avec username?

# Test manuel:
redis-cli --user username --pass password PING

# Vérifier logs Redis:
tail -f /var/log/redis/redis.log | grep "AUTH"
```

#### 5. ACL SAVE échoue

```bash
# Cause: Permissions fichier ou aclfile non défini
# Solution:

# Vérifier aclfile dans redis.conf:
redis-cli CONFIG GET aclfile

# Vérifier permissions:
ls -la /etc/redis/users.acl

# Corriger permissions:
chown redis:redis /etc/redis/users.acl
chmod 640 /etc/redis/users.acl
```

#### 6. ACL LOAD échoue

```bash
# Cause: Syntaxe invalide dans users.acl
# Solution:

# Valider syntaxe:
redis-server --test-memory 1 --aclfile /etc/redis/users.acl

# Backup et correction:
cp /etc/redis/users.acl /etc/redis/users.acl.backup
nano /etc/redis/users.acl  # Corriger erreurs
redis-cli ACL LOAD
```

---

## 📚 Ressources et bonnes pratiques

### Bonnes pratiques finales

1. **🔐 Principe du moindre privilège**
   - Donner uniquement les permissions nécessaires
   - Préférer patterns restrictifs (~app:*) vs wildcards (~*)

2. **🔑 Gestion des mots de passe**
   - Mots de passe forts (16+ caractères)
   - Hasher avec SHA256 dans users.acl
   - Rotation régulière (90 jours)
   - Ne JAMAIS commiter passwords en clair

3. **📝 Documentation**
   - Documenter chaque utilisateur et son usage
   - Maintenir matrice permissions
   - Procédures rotation passwords

4. **🔍 Audit et monitoring**
   - Surveiller ACL LOG quotidiennement
   - Audit mensuel des permissions
   - Alertes sur échecs authentification

5. **🔄 Automatisation**
   - Scripts de provisioning utilisateurs
   - Tests automatisés des ACLs
   - CI/CD pour déploiement users.acl

6. **💾 Backup**
   - Backup users.acl avant modifications
   - Versioning (git) de users.acl
   - Plan de recovery

7. **🎯 Séparation des rôles**
   - Un utilisateur = un service/application
   - Pas de partage de credentials
   - Traçabilité par utilisateur

### Documentation officielle

- [Redis ACL Documentation](https://redis.io/docs/management/security/acl/)
- [Redis Security Guide](https://redis.io/docs/management/security/)
- [ACL Best Practices](https://redis.io/docs/management/security/acl/#acl-rules)

---

**Section suivante :** [12.3 - Authentification et gestion des utilisateurs](./03-authentification-gestion-utilisateurs.md)

⏭️ [Authentification et gestion des utilisateurs](/12-redis-production-securite/03-authentification-gestion-utilisateurs.md)
