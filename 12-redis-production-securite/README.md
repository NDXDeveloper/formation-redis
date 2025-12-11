🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 12 : Redis en Production et Sécurité

## Vue d'ensemble du module

La mise en production de Redis est une étape critique qui nécessite une approche rigoureuse et méthodique. Ce module s'adresse aux **DevOps, SRE et Architectes** qui doivent déployer, sécuriser et maintenir des instances Redis en environnement de production.

> **⚠️ Point critique :** Redis, par défaut, est configuré pour la rapidité et la facilité de développement, **pas pour la sécurité en production**. Un Redis mal configuré est une porte d'entrée majeure pour les attaquants.

## Objectifs du module

À l'issue de ce module, vous serez capable de :

1. ✅ Configurer Redis de manière optimale pour la production
2. 🔒 Sécuriser Redis contre les vecteurs d'attaque courants
3. 🛡️ Implémenter une stratégie de défense en profondeur
4. 📊 Dimensionner correctement les ressources
5. 🔄 Effectuer des mises à jour sans interruption de service
6. 📋 Suivre les bonnes pratiques Linux pour Redis

---

## Architecture de sécurité multicouche

Redis en production nécessite une approche de sécurité par couches :

```
┌─────────────────────────────────────────────────────────────┐
│                    COUCHE RÉSEAU                            │
│  • VPC/VLAN isolation                                       │
│  • Firewall rules (iptables/Security Groups)                │
│  • Bind to private IP only                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                 COUCHE TRANSPORT                            │
│  • TLS/SSL encryption                                       │
│  • Certificate validation                                   │
│  • Mutual TLS (mTLS)                                        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              COUCHE AUTHENTIFICATION                        │
│  • ACLs granulaires (Redis 6+)                              │
│  • User management                                          │
│  • Password policies                                        │
│  • requirepass (legacy)                                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│               COUCHE AUTORISATION                           │
│  • Command filtering                                        │
│  • Key pattern restrictions                                 │
│  • Channel restrictions (Pub/Sub)                           │
│  • Read-only users                                          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                 COUCHE SYSTÈME                              │
│  • Kernel parameters tuning                                 │
│  • Resource limits (ulimit)                                 │
│  • Disable Transparent Huge Pages                           │
│  • Memory overcommit settings                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              COUCHE MONITORING & AUDIT                      │
│  • Audit logging                                            │
│  • Security event monitoring                                │
│  • Alerting on suspicious activity                          │
│  • Regular security audits                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚠️ Les 10 erreurs fatales en production

### 1. **Exposition publique sur Internet**
```bash
# ❌ DANGEREUX - Écoute sur toutes les interfaces
bind 0.0.0.0

# ✅ CORRECT - Écoute uniquement sur localhost ou IP privée
bind 127.0.0.1 10.0.1.50
```

### 2. **Absence d'authentification**
```bash
# ❌ DANGEREUX - Pas de mot de passe
# protected-mode yes # Insuffisant seul

# ✅ CORRECT - ACLs ou au minimum requirepass
requirepass "VotreMot2P@sseC0mplexe!2024"
# Ou mieux : ACLs (Redis 6+)
```

### 3. **Commandes dangereuses non désactivées**
```bash
# ❌ DANGEREUX - Toutes les commandes activées
# (par défaut)

# ✅ CORRECT - Désactiver les commandes dangereuses
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG ""
rename-command SHUTDOWN ""
rename-command DEBUG ""
```

### 4. **Pas de limite de mémoire**
```bash
# ❌ DANGEREUX - Pas de limite
# maxmemory non défini

# ✅ CORRECT - Limite stricte avec politique d'éviction
maxmemory 8gb
maxmemory-policy allkeys-lru
```

### 5. **THP (Transparent Huge Pages) activé**
```bash
# ❌ DANGEREUX - THP activé (défaut Linux)
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# ✅ CORRECT - THP désactivé
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### 6. **Overcommit memory non configuré**
```bash
# ❌ DANGEREUX - Overcommit désactivé
vm.overcommit_memory = 0

# ✅ CORRECT - Overcommit activé pour Redis
vm.overcommit_memory = 1
```

### 7. **Pas de monitoring de la mémoire**
```bash
# ❌ DANGEREUX - Pas d'alertes OOM
# Aucune surveillance

# ✅ CORRECT - Monitoring avec alertes
# Prometheus + Grafana avec alertes à 80% et 90%
```

### 8. **Utilisation de KEYS en production**
```bash
# ❌ DANGEREUX - Bloque Redis
KEYS user:*

# ✅ CORRECT - Utiliser SCAN
SCAN 0 MATCH user:* COUNT 100
```

### 9. **Backup non testé**
```bash
# ❌ DANGEREUX - Backup configuré mais jamais testé
save 900 1

# ✅ CORRECT - Backup testé régulièrement avec restauration
# Tests de DR (Disaster Recovery) mensuels
```

### 10. **Pas de plan de mise à jour**
```bash
# ❌ DANGEREUX - Version obsolète avec failles de sécurité
# Redis 4.x ou 5.x en production en 2024

# ✅ CORRECT - Version récente avec plan de mise à jour
# Redis 7.2+ avec rolling upgrades planifiés
```

---

## 📋 Checklist de sécurité pré-production

### Phase 1 : Configuration réseau

- [ ] **Isolation réseau**
  - [ ] Redis déployé dans un VPC/VLAN privé
  - [ ] Pas d'IP publique assignée à Redis
  - [ ] Bind uniquement sur interface privée
  - [ ] Firewall configuré (Security Groups/iptables)

- [ ] **Règles firewall**
  ```bash
  # Autoriser uniquement les serveurs applicatifs
  iptables -A INPUT -p tcp --dport 6379 -s 10.0.1.0/24 -j ACCEPT
  iptables -A INPUT -p tcp --dport 6379 -j DROP
  ```

- [ ] **TLS/SSL activé**
  - [ ] Certificats générés et valides
  - [ ] TLS 1.2 minimum
  - [ ] Ciphers forts uniquement
  - [ ] Mutual TLS (optionnel mais recommandé)

### Phase 2 : Authentification et autorisation

- [ ] **ACLs configurées (Redis 6+)**
  - [ ] Utilisateur `default` désactivé ou restreint
  - [ ] Utilisateurs applicatifs avec permissions minimales
  - [ ] Utilisateur admin séparé
  - [ ] Mots de passe forts (16+ caractères)

- [ ] **Commandes dangereuses**
  - [ ] `FLUSHDB` désactivé
  - [ ] `FLUSHALL` désactivé
  - [ ] `KEYS` désactivé
  - [ ] `CONFIG` restreint
  - [ ] `SHUTDOWN` restreint
  - [ ] `DEBUG` désactivé
  - [ ] `SCRIPT FLUSH` restreint

### Phase 3 : Configuration système Linux

- [ ] **Kernel parameters**
  ```bash
  # /etc/sysctl.conf
  vm.overcommit_memory = 1
  net.core.somaxconn = 65535
  net.ipv4.tcp_max_syn_backlog = 65535
  vm.swappiness = 0
  ```

- [ ] **Transparent Huge Pages désactivé**
  ```bash
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
  ```

- [ ] **Limites système**
  ```bash
  # /etc/security/limits.conf
  redis soft nofile 65535
  redis hard nofile 65535
  redis soft nproc 65535
  redis hard nproc 65535
  ```

- [ ] **Désactivation du swap**
  ```bash
  swapoff -a
  # Ou configuration vm.swappiness = 0
  ```

### Phase 4 : Configuration Redis

- [ ] **Mémoire et éviction**
  - [ ] `maxmemory` défini à 80% de la RAM disponible
  - [ ] `maxmemory-policy` adapté au use case
  - [ ] `maxmemory-samples` configuré (default: 5)

- [ ] **Persistance**
  - [ ] Mode de persistance choisi (RDB/AOF/Hybrid)
  - [ ] Fréquence de sauvegarde adaptée
  - [ ] Chemin de sauvegarde avec permissions correctes
  - [ ] Espace disque suffisant

- [ ] **Réseau et connexions**
  - [ ] `timeout` configuré (300 secondes recommandé)
  - [ ] `tcp-keepalive` activé (300 secondes)
  - [ ] `maxclients` défini selon besoin

- [ ] **Logging**
  - [ ] Niveau de log approprié (`notice` en production)
  - [ ] Rotation des logs configurée
  - [ ] Logs envoyés vers système centralisé

### Phase 5 : Haute disponibilité

- [ ] **Réplication**
  - [ ] Au moins 1 replica configuré
  - [ ] `repl-diskless-sync` évalué
  - [ ] `min-replicas-to-write` configuré si nécessaire

- [ ] **Sentinel ou Cluster**
  - [ ] Quorum configuré correctement
  - [ ] Tests de failover effectués
  - [ ] Clients configurés pour découverte automatique

### Phase 6 : Monitoring et alerting

- [ ] **Métriques collectées**
  - [ ] Mémoire utilisée / maxmemory
  - [ ] Évictions
  - [ ] Hit ratio
  - [ ] Latency
  - [ ] Connexions actives
  - [ ] Réplication lag

- [ ] **Alertes configurées**
  - [ ] Mémoire > 80%
  - [ ] Évictions > seuil
  - [ ] Hit ratio < seuil
  - [ ] Latency anormale
  - [ ] Réplication cassée
  - [ ] Master down

### Phase 7 : Backup et disaster recovery

- [ ] **Stratégie de backup**
  - [ ] Backups automatiques configurés
  - [ ] Rétention définie
  - [ ] Backups stockés hors site
  - [ ] Encryption des backups

- [ ] **Tests de restauration**
  - [ ] Procédure de restauration documentée
  - [ ] Test de restauration effectué
  - [ ] RTO/RPO définis et validés

### Phase 8 : Documentation et procédures

- [ ] **Documentation**
  - [ ] Architecture Redis documentée
  - [ ] Configuration commentée
  - [ ] Runbook de production créé
  - [ ] Procédures d'incident documentées

- [ ] **Formation équipe**
  - [ ] Équipe formée sur Redis
  - [ ] On-call rotation définie
  - [ ] Escalation path documenté

---

## 🔧 Configuration redis.conf de référence pour production

### Configuration minimale sécurisée

```conf
# ============================================================================
# REDIS PRODUCTION CONFIGURATION - SECURE BASELINE
# ============================================================================
# Version: Redis 7.2+
# Environnement: Production
# Niveau: Sécurité renforcée
# ============================================================================

# ----------------------------------------------------------------------------
# RÉSEAU
# ----------------------------------------------------------------------------

# Bind uniquement sur interface privée (JAMAIS 0.0.0.0 en production)
bind 127.0.0.1 10.0.1.50

# Port par défaut (changer en production pour réduire le scanning)
port 6379

# Protection automatique si pas d'auth (doit rester yes)
protected-mode yes

# Timeout des connexions inactives (5 minutes)
timeout 300

# TCP keepalive (détection connexions mortes)
tcp-keepalive 300

# Backlog des connexions TCP
tcp-backlog 511

# ----------------------------------------------------------------------------
# TLS/SSL (Redis 6+)
# ----------------------------------------------------------------------------

# Activer TLS sur le port standard
tls-port 6379
port 0  # Désactiver port non-TLS

# Certificats
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt

# Modes d'authentification TLS
tls-auth-clients optional  # ou 'yes' pour mutual TLS obligatoire

# Protocoles et ciphers
tls-protocols "TLSv1.2 TLSv1.3"
tls-ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
tls-prefer-server-ciphers yes

# ----------------------------------------------------------------------------
# AUTHENTIFICATION ET ACLs
# ----------------------------------------------------------------------------

# Charger les ACLs depuis un fichier externe
aclfile /etc/redis/users.acl

# Authentification legacy (si ACLs non utilisées)
# requirepass VotreMotDePasseTresComplexe123!

# Désactiver l'utilisateur default (après configuration ACLs)
# Dans users.acl:
# user default off

# ----------------------------------------------------------------------------
# MÉMOIRE
# ----------------------------------------------------------------------------

# Limite mémoire (80% de la RAM disponible recommandé)
maxmemory 8gb

# Politique d'éviction
maxmemory-policy allkeys-lru

# Échantillonnage pour LRU/LFU
maxmemory-samples 5

# ----------------------------------------------------------------------------
# PERSISTANCE - HYBRIDE (RDB + AOF)
# ----------------------------------------------------------------------------

# RDB - Snapshots
save 900 1      # Après 900 sec (15 min) si au moins 1 clé changée
save 300 10     # Après 300 sec (5 min) si au moins 10 clés changées
save 60 10000   # Après 60 sec si au moins 10000 clés changées

# Ne pas arrêter si échec écriture
stop-writes-on-bgsave-error yes

# Compression RDB
rdbcompression yes
rdbchecksum yes

# Nom du fichier RDB
dbfilename dump.rdb

# AOF - Append Only File
appendonly yes
appendfilename "appendonly.aof"

# Stratégie de sync AOF (everysec = bon compromis)
appendfsync everysec

# Réecriture automatique AOF
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Ne pas sync pendant rewrite
no-appendfsync-on-rewrite no

# Répertoire de travail
dir /var/lib/redis

# ----------------------------------------------------------------------------
# RÉPLICATION
# ----------------------------------------------------------------------------

# Si ce serveur est un replica
# replicaof <masterip> <masterport>

# Auth vers le master
# masterauth <master-password>

# Lecture seule sur replica
replica-read-only yes

# Sync sans disque (si réseau rapide)
repl-diskless-sync no
repl-diskless-sync-delay 5

# Backlog de réplication (augmenter si réplication lente)
repl-backlog-size 256mb
repl-backlog-ttl 3600

# Ping entre master et replica
repl-ping-replica-period 10
repl-timeout 60

# Désactiver TCP_NODELAY (meilleur débit, latence légèrement plus haute)
repl-disable-tcp-nodelay no

# Priorité du replica pour promotion (100 = défaut, 0 = jamais promu)
replica-priority 100

# Minimum de replicas pour accepter écritures (optionnel mais recommandé)
min-replicas-to-write 1
min-replicas-max-lag 10

# ----------------------------------------------------------------------------
# SÉCURITÉ - COMMANDES DANGEREUSES
# ----------------------------------------------------------------------------

# Désactiver les commandes dangereuses
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG ""
rename-command SHUTDOWN ""
rename-command DEBUG ""
rename-command BGSAVE ""
rename-command BGREWRITEAOF ""
rename-command SAVE ""
rename-command SPOP ""
rename-command SREM ""
rename-command MIGRATE ""

# Ou les renommer (moins sécurisé mais permet usage admin)
# rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
# rename-command FLUSHALL my_flushall_2024

# ----------------------------------------------------------------------------
# LIMITES
# ----------------------------------------------------------------------------

# Nombre maximum de clients
maxclients 10000

# ----------------------------------------------------------------------------
# LAZY FREEING
# ----------------------------------------------------------------------------

# Libération mémoire asynchrone (réduit les spikes de latence)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes

# Suppression lazy des DBs
lazyfree-lazy-user-del yes

# ----------------------------------------------------------------------------
# LOGGING
# ----------------------------------------------------------------------------

# Niveau de log en production
loglevel notice

# Fichier de log (ou "" pour stdout)
logfile "/var/log/redis/redis-server.log"

# Syslog (optionnel)
# syslog-enabled yes
# syslog-ident redis
# syslog-facility local0

# ----------------------------------------------------------------------------
# SLOW LOG
# ----------------------------------------------------------------------------

# Temps d'exécution pour considérer une commande lente (microsecondes)
slowlog-log-slower-than 10000  # 10ms

# Taille du slow log
slowlog-max-len 128

# ----------------------------------------------------------------------------
# LATENCY MONITORING
# ----------------------------------------------------------------------------

# Seuil de latency monitoring (millisecondes)
latency-monitor-threshold 100

# ----------------------------------------------------------------------------
# DEFRAGMENTATION ACTIVE
# ----------------------------------------------------------------------------

# Activer la défragmentation active
activedefrag yes

# Seuils de déclenchement
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100

# Effort CPU (1-10, 10 = max)
active-defrag-cycle-min 1
active-defrag-cycle-max 25

# ----------------------------------------------------------------------------
# THREADS I/O (Redis 6+)
# ----------------------------------------------------------------------------

# Threads pour I/O réseau (1 thread = désactivé, 2-4 recommandé)
io-threads 4

# I/O threads pour écriture aussi (pas juste lecture)
io-threads-do-reads yes

# ----------------------------------------------------------------------------
# ADVANCED CONFIG
# ----------------------------------------------------------------------------

# Hash max entries avant conversion en hash table
hash-max-listpack-entries 512
hash-max-listpack-value 64

# List max size
list-max-listpack-size -2

# Set max entries
set-max-intset-entries 512

# Sorted set max entries
zset-max-listpack-entries 128
zset-max-listpack-value 64

# HyperLogLog sparse max bytes
hll-sparse-max-bytes 3000

# Stream node max entries
stream-node-max-bytes 4096
stream-node-max-entries 100

# Active rehashing
activerehashing yes

# Client output buffer limits
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# Fréquence du serveur (Hz)
hz 10

# Adaptive HZ
dynamic-hz yes

# AOF rewrite incremental fsync
aof-rewrite-incremental-fsync yes

# RDB save incremental fsync
rdb-save-incremental-fsync yes

# Jemalloc background thread
jemalloc-bg-thread yes

# ============================================================================
# FIN DE CONFIGURATION
# ============================================================================
```

---

## 📝 Exemple de fichier ACL (users.acl)

```acl
# ============================================================================
# REDIS ACL CONFIGURATION
# ============================================================================
# Format: user <username> <flags> <permissions>
# ============================================================================

# Désactiver l'utilisateur default (après migration)
user default off

# ------------------------------------------------------------------------------
# UTILISATEUR ADMIN - Accès complet
# ------------------------------------------------------------------------------
user admin on >AdminP@ssw0rd!2024 ~* &* +@all

# ------------------------------------------------------------------------------
# UTILISATEUR APPLICATION - Lecture/Écriture limitée
# ------------------------------------------------------------------------------
# Peut accéder uniquement aux clés commençant par "app:"
# Peut publier sur channels "notifications:*"
# Commandes autorisées : lecture, écriture, listes, hash
user app_user on >AppUserP@ss2024! ~app:* &notifications:* +@read +@write +@list +@hash +@string -@dangerous

# ------------------------------------------------------------------------------
# UTILISATEUR CACHE - Lecture seule
# ------------------------------------------------------------------------------
user cache_reader on >CacheRead2024! ~cache:* +@read +get +mget +exists +ttl +type

# ------------------------------------------------------------------------------
# UTILISATEUR MONITORING - Lecture seule + commandes INFO
# ------------------------------------------------------------------------------
user monitoring on >MonitorP@ss2024! ~* +@read +info +config|get +client|list +slowlog +latency +memory

# ------------------------------------------------------------------------------
# UTILISATEUR BACKUP - Sauvegarde uniquement
# ------------------------------------------------------------------------------
user backup on >BackupP@ss2024! ~* +bgsave +lastsave +save +info

# ------------------------------------------------------------------------------
# UTILISATEUR QUEUE - Files d'attente uniquement
# ------------------------------------------------------------------------------
user queue_worker on >QueueW0rk2024! ~queue:* ~queue:*:processing +@list +@read +@write -@dangerous

# ------------------------------------------------------------------------------
# UTILISATEUR PUBSUB - Publication uniquement
# ------------------------------------------------------------------------------
user publisher on >PublishP@ss2024! &events:* +publish

# ------------------------------------------------------------------------------
# UTILISATEUR SUBSCRIBER - Souscription uniquement
# ------------------------------------------------------------------------------
user subscriber on >SubP@ss2024! &events:* +subscribe +psubscribe +unsubscribe +punsubscribe

# ============================================================================
# CATÉGORIES DE COMMANDES
# ============================================================================
# @read      - Commandes de lecture
# @write     - Commandes d'écriture
# @admin     - Commandes administratives
# @dangerous - Commandes dangereuses (FLUSHALL, KEYS, etc.)
# @fast      - Commandes O(1) ou O(log N)
# @slow      - Commandes potentiellement lentes
# @keyspace  - Commandes affectant le keyspace
# @string    - Commandes sur strings
# @list      - Commandes sur lists
# @set       - Commandes sur sets
# @sortedset - Commandes sur sorted sets
# @hash      - Commandes sur hashes
# @pubsub    - Commandes pub/sub
# @stream    - Commandes sur streams
# ============================================================================
```

---

## 🔐 Configuration système Linux pour Redis

### 1. Fichier /etc/sysctl.conf

```bash
# ============================================================================
# CONFIGURATION KERNEL POUR REDIS
# ============================================================================

# Memory Overcommit - CRITIQUE pour Redis
# 0 = Heuristique (défaut, MAUVAIS pour Redis)
# 1 = Toujours accepter (REQUIS pour Redis)
# 2 = Jamais accepter plus que swap + ratio
vm.overcommit_memory = 1

# Ratio overcommit (si mode = 2)
vm.overcommit_ratio = 100

# Swappiness - Minimiser l'usage du swap
# 0-1 recommandé pour Redis
vm.swappiness = 0

# TCP backlog - Augmenter pour haute concurrence
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TCP keepalive settings
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 30

# Réutilisation rapide des sockets TIME_WAIT
net.ipv4.tcp_tw_reuse = 1

# TCP Fast Open
net.ipv4.tcp_fastopen = 3

# Buffer sizes
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# File handles
fs.file-max = 2097152

# Appliquer : sysctl -p
```

### 2. Fichier /etc/security/limits.conf

```bash
# ============================================================================
# LIMITES SYSTÈME POUR L'UTILISATEUR REDIS
# ============================================================================

# Nombre maximum de fichiers ouverts
redis soft nofile 65535
redis hard nofile 65535

# Nombre maximum de processus
redis soft nproc 65535
redis hard nproc 65535

# Taille maximale du core dump (optionnel)
redis soft core unlimited
redis hard core unlimited

# Taille maximale de la stack
redis soft stack 10240
redis hard stack 10240
```

### 3. Script de désactivation THP

```bash
#!/bin/bash
# ============================================================================
# DÉSACTIVATION TRANSPARENT HUGE PAGES (THP)
# ============================================================================
# Fichier: /etc/rc.local ou service systemd
# THP cause des latences imprévisibles avec Redis

# Désactiver THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Vérifier
cat /sys/kernel/mm/transparent_hugepage/enabled
# Doit afficher: always madvise [never]

# Rendre permanent avec systemd
# Créer /etc/systemd/system/disable-thp.service:
```

### 4. Service systemd pour désactivation THP

```ini
# /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages (THP)
Before=redis.service
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=multi-user.target
```

Activation :
```bash
systemctl daemon-reload
systemctl enable disable-thp.service
systemctl start disable-thp.service
```

---

## 🎯 Matrice de décision : Configuration selon l'environnement

| Aspect | Développement | Staging | Production |
|--------|---------------|---------|------------|
| **Bind** | 127.0.0.1 | IP privée | IP privée |
| **Protected mode** | yes | yes | yes |
| **Authentification** | Optionnel | Requis | Requis (ACLs) |
| **TLS** | Non | Oui | Oui (obligatoire) |
| **Maxmemory** | 512mb | 2-4gb | 8gb+ |
| **Persistance** | RDB seul | RDB + AOF | Hybride |
| **Réplication** | Non | 1 replica | 2+ replicas |
| **Monitoring** | Basique | Complet | Complet + alerting |
| **Backup** | Manuel | Automatique | Automatique + off-site |
| **Commandes dangereuses** | Actives | Renommées | Désactivées |
| **Logging** | debug | notice | notice/warning |
| **Lazy freeing** | Non | Oui | Oui |
| **Active defrag** | Non | Oui | Oui |
| **IO threads** | 1 | 2-4 | 4 |

---

## 📊 Checklist de validation post-déploiement

### Validation immédiate (J0)

```bash
# 1. Vérifier que Redis démarre
systemctl status redis

# 2. Vérifier le binding
ss -tlnp | grep 6379

# 3. Tester la connexion authentifiée
redis-cli -a yourpassword PING

# 4. Vérifier les ACLs
redis-cli -a adminpassword ACL LIST

# 5. Vérifier la configuration chargée
redis-cli -a adminpassword CONFIG GET maxmemory
redis-cli -a adminpassword CONFIG GET appendonly

# 6. Vérifier THP
cat /sys/kernel/mm/transparent_hugepage/enabled  # doit être [never]

# 7. Vérifier overcommit
sysctl vm.overcommit_memory  # doit être 1

# 8. Vérifier les logs
tail -f /var/log/redis/redis-server.log

# 9. Tester les commandes désactivées
redis-cli -a password KEYS *  # doit retourner une erreur

# 10. Vérifier la réplication (si configurée)
redis-cli -a password INFO replication
```

### Validation J+1

- [ ] Aucun warning dans les logs
- [ ] Utilisation mémoire stable
- [ ] Pas de latence anormale
- [ ] Réplication synchronisée
- [ ] Backups effectués
- [ ] Monitoring fonctionnel
- [ ] Alertes de test reçues

### Validation J+7

- [ ] Aucune éviction prématurée
- [ ] Pas de fragmentation excessive (< 1.5)
- [ ] Hit ratio conforme aux attentes
- [ ] Pas de slow queries répétées
- [ ] Backups restaurables validés
- [ ] Procédures d'incident testées

---

## 🚨 Vecteurs d'attaque courants et mitigations

### 1. **Port scan et découverte**
```
Attaque: Scan des ports 6379-6380 sur Internet
Mitigation:
  - Firewall avec whitelist IP
  - Changement du port par défaut
  - Bind sur IP privée uniquement
```

### 2. **Authentification faible**
```
Attaque: Brute force sur requirepass
Mitigation:
  - Mot de passe fort (16+ caractères)
  - ACLs avec verrouillage après échecs
  - Monitoring des tentatives de connexion
```

### 3. **Injection de commandes**
```
Attaque: EVAL avec code Lua malveillant
Mitigation:
  - Validation stricte des inputs
  - ACLs restreignant EVAL
  - Sandbox Lua activé
```

### 4. **Déni de service (DoS)**
```
Attaque: KEYS * sur grosse DB
Mitigation:
  - KEYS renommé ou désactivé
  - Utilisation de SCAN uniquement
  - Rate limiting au niveau applicatif
```

### 5. **Exfiltration de données**
```
Attaque: Dump complet via SYNC (réplication)
Mitigation:
  - masterauth configuré
  - Monitoring des nouvelles connexions replica
  - Network isolation
```

---

## 📚 Contenu du module

Ce module se compose des sections suivantes :

1. **Configuration optimale pour la production** - Configuration redis.conf détaillée et commentée
2. **Sécuriser Redis avec ACLs** - Gestion granulaire des permissions
3. **Authentification et gestion des utilisateurs** - Stratégies d'authentification
4. **Chiffrement TLS/SSL** - Configuration et impact sur les performances
5. **Protection réseau** - Firewall, VPC, et isolation
6. **Bonnes pratiques Linux** - THP, Swap, Overcommit et tuning kernel
7. **Dimensionnement et planification de capacité** - Calculer les ressources nécessaires
8. **Mises à jour sans downtime** - Stratégies de rolling upgrades

---

## 🎓 Prérequis

Avant d'aborder ce module, vous devez maîtriser :

- ✅ Les structures de données Redis (Modules 1-3)
- ✅ Le cycle de vie de la donnée et la persistance (Modules 4-5)
- ✅ L'administration Linux de base
- ✅ Les concepts réseau (TCP/IP, TLS, Firewall)
- ✅ Les principes de haute disponibilité

---

## ⚠️ Points d'attention critiques

> **DANGER : Commandes à ne JAMAIS exécuter en production**
> ```bash
> FLUSHALL  # Efface TOUTES les données de TOUTES les DB
> FLUSHDB   # Efface toutes les données de la DB courante
> KEYS *    # Bloque Redis sur de grosses instances
> CONFIG SET  # Peut casser Redis en live
> SHUTDOWN  # Arrête Redis
> DEBUG SEGFAULT  # Crashe Redis volontairement
> ```

> **INFO : Redis n'est PAS sécurisé par défaut**
> Redis privilégie la simplicité et la performance. La sécurité est de VOTRE responsabilité. Un Redis exposé sans authentification sera compromis en quelques minutes.

> **CRITIQUE : Sauvegardez avant TOUTE manipulation**
> Avant toute modification de configuration ou mise à jour, effectuez un backup complet. Les données perdues ne sont jamais récupérables.

---

## 🔗 Ressources complémentaires

- [Redis Security Guide officiel](https://redis.io/docs/management/security/)
- [Redis Admin Guide](https://redis.io/docs/management/admin/)
- [Linux Performance Tuning for Redis](https://redis.io/docs/management/optimization/)
- [OWASP - Securing Redis](https://cheatsheetseries.owasp.org/cheatsheets/Redis_Security_Cheat_Sheet.html)

---

**Prochaine section :** [12.1 - Configuration optimale pour la production](./01-configuration-optimale-production.md)

⏭️ [Configuration optimale pour la production](/12-redis-production-securite/01-configuration-optimale-production.md)
