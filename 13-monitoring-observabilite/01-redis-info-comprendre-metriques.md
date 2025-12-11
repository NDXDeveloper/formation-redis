🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.1 Redis INFO : Comprendre toutes les métriques

## Introduction

La commande `INFO` est la pierre angulaire du monitoring Redis. Elle expose plus de 100 métriques organisées en sections thématiques. Comprendre ces métriques en profondeur est essentiel pour diagnostiquer les problèmes, optimiser les performances et prendre des décisions d'architecture éclairées.

## Anatomie de la commande INFO

### Utilisation de base

```bash
# Toutes les sections
redis-cli INFO

# Section spécifique
redis-cli INFO server
redis-cli INFO memory
redis-cli INFO stats

# Plusieurs sections
redis-cli INFO server memory

# Format pour parsing (sans commentaires)
redis-cli --raw INFO | grep -v "^#"
```

### Sections disponibles

```
server          # Informations sur le serveur Redis
clients         # Connexions clients
memory          # Utilisation et gestion de la mémoire
persistence     # État RDB/AOF
stats           # Statistiques générales
replication     # État de la réplication
cpu             # Consommation CPU
commandstats    # Statistiques par commande
cluster         # État du cluster (si activé)
keyspace        # Statistiques par base de données
modules         # Modules chargés (Redis Stack)
errorstats      # Statistiques d'erreurs (Redis 6.2+)
```

## Section SERVER : Identité et configuration

### Exemple de sortie

```
# Server
redis_version:7.2.3
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:1234567890abcdef
redis_mode:standalone
os:Linux 5.15.0-91-generic x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:c11-builtin
gcc_version:11.4.0
process_id:1234
process_supervised:no
run_id:a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
tcp_port:6379
server_time_usec:1702308123456789
uptime_in_seconds:864000
uptime_in_days:10
hz:10
configured_hz:10
lru_clock:12345678
executable:/usr/local/bin/redis-server
config_file:/etc/redis/redis.conf
io_threads_active:0
```

### Métriques critiques

#### redis_version
**Format** : `X.Y.Z`
**Exemple** : `7.2.3`
**Utilisation** :
- Identifier la version exacte pour compatibilité features
- Vérifier si patchs de sécurité appliqués
- Planifier les upgrades

**À surveiller** :
- Version < 6.0 → Pas d'ACLs, pas de SSL/TLS natif
- Version < 7.0 → Pas de Functions, pas de Sharded Pub/Sub

#### redis_mode
**Valeurs possibles** :
- `standalone` : Instance unique
- `sentinel` : Instance Sentinel (monitoring)
- `cluster` : Nœud dans un cluster

**Importance** : Détermine les métriques disponibles et le comportement

#### multiplexing_api
**Valeurs courantes** :
- `epoll` (Linux) : Optimal
- `kqueue` (BSD/macOS) : Optimal
- `select` : Sous-optimal, limité à 1024 connexions

**Red flag** : `select` en production Linux = problème de compilation

#### process_id (PID)
**Utilisation** :
- Surveillance de la stabilité (PID change = redémarrage)
- Monitoring système (top, htop avec PID spécifique)
- Signaux Unix (kill -USR1 pour logs, etc.)

#### run_id
**Format** : Hash 40 caractères
**Utilisation** :
- Identifier l'instance actuelle
- Détecter les redémarrages (run_id change)
- Suivi de la réplication (replica utilise le run_id du master)

**Cas d'usage** : Corrélation logs multi-instances

#### uptime_in_seconds
**Signification** : Temps depuis le dernier démarrage
**Surveillance** :
- Uptime < 300s : Redémarrage récent (enquêter)
- Uptime réinitialisé régulièrement : Instabilité
- Uptime très élevé : Bon signe, mais prévoir maintenance

#### hz
**Valeur par défaut** : 10
**Signification** : Fréquence (en Hz) des tâches de fond Redis
- Expiration des clés
- Évictions
- Gestion des clients
- Replication
- AOF rewrite

**Impact** :
- `hz: 10` → Tâches toutes les 100ms
- `hz: 100` → Tâches toutes les 10ms (plus réactif mais + CPU)

**Recommandation** : Laisser à 10 sauf workload spécifique (real-time gaming)

#### io_threads_active
**Valeurs** :
- `0` : I/O threading désactivé (par défaut)
- `1` : I/O threading activé

**Redis 6+** : Permet le multi-threading pour I/O réseau
**Configuration** : `io-threads 4` + `io-threads-do-reads yes`

## Section CLIENTS : Gestion des connexions

### Exemple de sortie

```
# Clients
connected_clients:247
cluster_connections:0
maxclients:10000
client_recent_max_input_buffer:16
client_recent_max_output_buffer:0
blocked_clients:5
tracking_clients:0
clients_in_timeout_table:0
total_blocking_keys:12
total_blocking_keys_on_nokey:0
```

### Métriques critiques

#### connected_clients
**Signification** : Nombre de connexions clients actives
**Valeurs typiques** :
- Application avec pool : 50-200 par serveur app
- Sans pool : Peut monter à plusieurs milliers

**À surveiller** :
```
connected_clients > 80% de maxclients → Warning
connected_clients == maxclients → Critical (nouvelles connexions refusées)
```

**Croissance soudaine** :
- Connection leak applicatif
- Absence de connection pooling
- Attaque DDoS

**Chute brutale** :
- Timeout réseau
- Redémarrage applicatif
- Problème firewall/load balancer

#### maxclients
**Défaut** : 10000
**Configuration** : `maxclients 10000` dans redis.conf

**Calcul du dimensionnement** :
```
maxclients = (nb_serveurs_app × pool_size × 1.5) + marge
```

**Exemple** :
- 10 serveurs app
- Pool de 20 connexions chacun
- maxclients = (10 × 20 × 1.5) + 100 = 400

**Limite système** :
```
maxclients ≤ (ulimit -n) - 32
```
Les 32 connexions réservées : internal, monitoring, replication

#### blocked_clients
**Signification** : Clients en attente de commandes bloquantes

**Commandes bloquantes** :
- `BLPOP`, `BRPOP`, `BRPOPLPUSH` (listes)
- `BLMOVE`, `BLMPOP` (Redis 7+)
- `BZPOPMIN`, `BZPOPMAX` (sorted sets)
- `XREAD BLOCK`, `XREADGROUP BLOCK` (streams)

**Interprétation** :
- **Normal** : Si vous utilisez des queues avec blocking pop
- **Problème** : Croissance continue sans décroissance
  - Workers morts qui ne consomment pas
  - Queue producteur trop lent
  - Deadlock applicatif

**Exemple de monitoring** :
```promql
# Alerter si blocked_clients > 50% de connected_clients
(redis_blocked_clients / redis_connected_clients) > 0.5
```

#### tracking_clients
**Redis 6+** : Client-side caching
**Signification** : Nombre de clients utilisant le tracking (RESP3)

**Utilisation** :
- Invalidation cache côté client
- Réduction drastique du trafic réseau
- Feature killer de Redis 6+

#### client_recent_max_input_buffer
**Signification** : Taille max du buffer d'entrée récent (bytes)
**Défaut** : 16 bytes (très petit)

**Problème si élevé** :
- Client envoyant des commandes très longues
- Possible tentative d'exploitation (buffer overflow)
- Requête malformée

**Surveillance** :
```
client_recent_max_input_buffer > 1MB → Enquêter
```

#### client_recent_max_output_buffer
**Signification** : Taille max du buffer de sortie récent
**Problème si élevé** :
- Client lent à lire les réponses
- Réponse très volumineuse (`KEYS *`, `HGETALL` sur gros hash)
- Risque de déconnexion client (timeout)

**Configuration** :
```
# redis.conf
client-output-buffer-limit normal 0 0 0      # Pas de limite (défaut)
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

## Section MEMORY : Le cœur du monitoring Redis

### Exemple de sortie

```
# Memory
used_memory:2147483648
used_memory_human:2.00G
used_memory_rss:2415919104
used_memory_rss_human:2.25G
used_memory_peak:3221225472
used_memory_peak_human:3.00G
used_memory_peak_perc:66.67%
used_memory_overhead:104857600
used_memory_startup:52428800
used_memory_dataset:2042626048
used_memory_dataset_perc:95.12%
allocator_allocated:2147483648
allocator_active:2281701376
allocator_resident:2415919104
total_system_memory:16777216000
total_system_memory_human:15.62G
used_memory_lua:49152
used_memory_vm_eval:49152
used_memory_lua_human:48.00K
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
number_of_functions:0
number_of_libraries:0
used_memory_vm_functions:32768
used_memory_vm_total:81920
used_memory_vm_total_human:80.00K
used_memory_functions:312
used_memory_scripts_eval:0
maxmemory:4294967296
maxmemory_human:4.00G
maxmemory_policy:allkeys-lru
allocator_frag_ratio:1.06
allocator_frag_bytes:134217728
allocator_rss_ratio:1.06
allocator_rss_bytes:134217728
rss_overhead_ratio:1.00
rss_overhead_bytes:0
mem_fragmentation_ratio:1.12
mem_fragmentation_bytes:268435456
mem_not_counted_for_evict:0
mem_replication_backlog:1048576
mem_total_replication_buffers:2097152
mem_clients_slaves:0
mem_clients_normal:20971520
mem_cluster_links:0
mem_aof_buffer:0
mem_allocator:jemalloc-5.3.0
active_defrag_running:0
lazyfree_pending_objects:0
lazyfreed_objects:0
```

### Métriques fondamentales

#### used_memory
**Définition** : Mémoire allouée par Redis depuis l'allocateur
**Inclut** :
- Données utilisateur (clés + valeurs)
- Overhead interne Redis
- Buffers de réplication
- Clients connectés

**N'inclut pas** :
- Fragmentation mémoire
- Mémoire système non-allouée

**Formule de calcul de l'utilisation** :
```
utilisation_pct = (used_memory / maxmemory) × 100
```

**Seuils opérationnels** :
- **< 70%** : Vert - Capacité confortable
- **70-85%** : Jaune - Surveillance accrue
- **85-95%** : Orange - Planifier upgrade ou éviction active
- **> 95%** : Rouge - Risque OOM imminent

#### used_memory_rss
**Définition** : Resident Set Size - Mémoire physique réellement occupée
**Mesure système** : Ce que voit le système d'exploitation

**Important** :
```
used_memory_rss = used_memory + fragmentation + autres
```

**Surveillance** : Si `used_memory_rss >> used_memory` → fragmentation

#### mem_fragmentation_ratio
**🔥 MÉTRIQUE CRITIQUE 🔥**

**Calcul** :
```
mem_fragmentation_ratio = used_memory_rss / used_memory
```

**Interprétation détaillée** :

| Ratio | État | Signification | Action |
|-------|------|---------------|--------|
| < 1.0 | 🔴 Danger | Swap ou overcommit | Urgent : Augmenter RAM ou limiter maxmemory |
| 1.0 - 1.3 | 🟢 Optimal | Fragmentation négligeable | RAS |
| 1.3 - 1.5 | 🟡 Acceptable | Fragmentation modérée | Surveiller la tendance |
| 1.5 - 2.0 | 🟠 Problématique | Fragmentation notable | Planifier redémarrage |
| > 2.0 | 🔴 Critique | Perte mémoire importante | Redémarrage recommandé |
| > 3.0 | 🔴 Sévère | 66%+ de mémoire gaspillée | Redémarrage urgent |

**Causes de fragmentation élevée** :
1. **Workload volatile** : Créations/suppressions fréquentes de clés de tailles variées
2. **Évictions massives** : Politique d'éviction agressive
3. **Longue durée d'uptime** : Fragmentation s'accumule naturellement
4. **Datasets hétérogènes** : Mix de petites et très grosses clés

**Solution Redis 4.0+** : Active Defragmentation
```conf
# redis.conf
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
active-defrag-cycle-min 1
active-defrag-cycle-max 25
```

#### mem_fragmentation_bytes
**Définition** : Quantité absolue de mémoire fragmentée (bytes)

**Calcul** :
```
mem_fragmentation_bytes = used_memory_rss - used_memory
```

**Exemple** :
- `used_memory`: 2GB
- `used_memory_rss`: 2.5GB
- `mem_fragmentation_bytes`: 512MB de mémoire gaspillée

**Surveillance** : Si > 500MB → Considérer defragmentation ou redémarrage

#### used_memory_peak
**Définition** : Pic de mémoire atteint depuis le dernier démarrage

**Utilisation** :
- Planification de capacité
- Dimensionnement `maxmemory`
- Identifier les pics de charge

**Ratio actuel/peak** :
```
used_memory_peak_perc = (used_memory / used_memory_peak) × 100
```

**Si proche de 100%** : Mémoire actuellement à son maximum historique

#### used_memory_overhead
**Définition** : Mémoire non liée aux données utilisateur

**Inclut** :
- Structures internes Redis
- Dictionnaires
- Expire tables
- Buffers clients
- Backlog de réplication
- AOF buffer

**Calcul de l'efficacité** :
```
efficacite_memoire = used_memory_dataset / used_memory × 100
```

**Objectif** : > 90% pour un usage efficace

#### used_memory_dataset
**Définition** : Mémoire réellement utilisée par les données utilisateur

**Formule** :
```
used_memory_dataset = used_memory - used_memory_overhead
```

#### maxmemory
**Définition** : Limite maximale configurable

**Configuration** :
```conf
# redis.conf
maxmemory 4gb

# Ou pourcentage de RAM système (Redis 7+)
maxmemory 75%
```

**Recommandation de dimensionnement** :
```
maxmemory = RAM_totale × 0.75

# Exemple : Serveur 16GB RAM
maxmemory = 16GB × 0.75 = 12GB
```

**Pourquoi 75% et pas 100%** :
- 15% : OS, buffers système
- 10% : Fragmentation, overhead Redis, pics

#### maxmemory_policy
**Valeurs possibles** :

**Éviction basée sur les données** :
- `allkeys-lru` : LRU sur toutes les clés (recommandé pour cache)
- `allkeys-lfu` : LFU sur toutes les clés (Redis 4.0+)
- `allkeys-random` : Aléatoire sur toutes les clés

**Éviction basée sur TTL** :
- `volatile-lru` : LRU sur clés avec expire
- `volatile-lfu` : LFU sur clés avec expire
- `volatile-random` : Aléatoire sur clés avec expire
- `volatile-ttl` : Clés avec TTL le plus court

**Pas d'éviction** :
- `noeviction` : Retourne erreur si mémoire pleine (défaut)

**Choix selon use case** :

| Use Case | Politique recommandée |
|----------|----------------------|
| Cache pur | `allkeys-lru` ou `allkeys-lfu` |
| Session store | `volatile-lru` |
| Queue/Stream | `noeviction` |
| Mixed workload | `allkeys-lru` |

#### allocator_frag_ratio
**Définition** : Fragmentation au niveau de l'allocateur mémoire (jemalloc)

**Calcul** :
```
allocator_frag_ratio = allocator_active / allocator_allocated
```

**Complément de mem_fragmentation_ratio** : Plus granulaire

#### mem_allocator
**Valeurs courantes** :
- `jemalloc-5.x.x` : Par défaut, optimal (recommandé)
- `libc` : Système, moins performant
- `tcmalloc` : Google, bon aussi

**Red flag** : `libc` en production = fragmentation potentiellement plus élevée

### Métriques de réplication (mémoire)

#### mem_replication_backlog
**Définition** : Taille du backlog de réplication

**Rôle** : Buffer circulaire stockant les commandes write pour les replicas

**Configuration** :
```conf
repl-backlog-size 1mb  # Défaut
```

**Dimensionnement** :
```
repl-backlog-size = write_throughput_per_sec × expected_disconnect_time

# Exemple :
# 1000 writes/sec × 100 bytes/write = 100KB/s
# Tolérance 60s de déconnexion
repl-backlog-size = 100KB × 60 = 6MB
```

#### mem_clients_slaves
**Définition** : Mémoire utilisée par les buffers des replicas connectés

**Calcul approximatif** :
```
mem_clients_slaves ≈ nb_replicas × 32KB (par défaut)
```

**Peut exploser si** : Replica trop lent, accumulation de writes

#### mem_clients_normal
**Définition** : Mémoire des buffers clients normaux

**Calcul** :
```
mem_clients_normal ≈ connected_clients × buffer_moyen
```

**Surveillance** : Croissance anormale = clients lents ou requêtes volumineuses

### Métriques de scripts

#### used_memory_lua (Redis < 7)
#### used_memory_vm_eval (Redis 7+)
**Définition** : Mémoire utilisée par les scripts Lua/EVAL

**Inclut** :
- Scripts en cache
- Contexte d'exécution
- Variables globales

**Limite** : 5MB par défaut (protections DoS)

#### number_of_cached_scripts
**Définition** : Nombre de scripts Lua en cache

**Gestion** :
- Scripts chargés via `SCRIPT LOAD`
- Persistent jusqu'à `SCRIPT FLUSH` ou redémarrage

**Surveillance** : Croissance = possibles scripts non-optimisés

## Section PERSISTENCE : RDB & AOF

### Exemple de sortie

```
# Persistence
loading:0
async_loading:0
current_cow_peak:0
current_cow_size:0
current_cow_size_age:0
current_fork_perc:0.00
current_save_keys_processed:0
current_save_keys_total:0
rdb_changes_since_last_save:12847
rdb_bgsave_in_progress:0
rdb_last_save_time:1702305600
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:2
rdb_current_bgsave_time_sec:-1
rdb_saves:127
rdb_last_cow_size:4194304
rdb_last_load_keys_expired:0
rdb_last_load_keys_loaded:1523478
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:18
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_rewrites:42
aof_rewrites_consecutive_failures:0
aof_last_write_status:ok
aof_last_cow_size:8388608
aof_current_size:524288000
aof_base_size:104857600
aof_pending_rewrite:0
aof_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0
```

### Métriques RDB

#### rdb_changes_since_last_save
**Définition** : Nombre de write operations depuis le dernier snapshot

**Utilisation** :
- Estimation de la perte de données potentielle
- Déclenchement manuel de `BGSAVE` si nécessaire

**Exemple** :
```
rdb_changes_since_last_save: 150000
→ Si crash maintenant, 150k opérations perdues
```

**Surveillance** :
```promql
# Alerter si > 1 million de changements non-sauvegardés
redis_rdb_changes_since_last_save > 1000000
```

#### rdb_bgsave_in_progress
**Valeurs** :
- `0` : Pas de BGSAVE en cours
- `1` : BGSAVE en cours (fork + écriture disque)

**Impact pendant BGSAVE** :
- Latency spike au moment du fork
- I/O disque élevé
- Doublement potentiel de la mémoire (COW - Copy-On-Write)

#### rdb_last_save_time
**Définition** : Timestamp Unix du dernier snapshot réussi

**Calcul du délai** :
```python
import time
time_since_last_save = time.time() - rdb_last_save_time
```

**Alerte recommandée** :
```
time() - rdb_last_save_time > 3600  # 1 heure sans backup
```

#### rdb_last_bgsave_status
**Valeurs** :
- `ok` : Dernier BGSAVE réussi
- `err` : Dernier BGSAVE échoué

**Si `err`** :
1. Vérifier les logs Redis
2. Causes courantes :
   - Disque plein
   - Permissions insuffisantes
   - Problème fork (ENOMEM)

#### rdb_last_bgsave_time_sec
**Définition** : Durée du dernier BGSAVE (secondes)

**Valeurs typiques** :
- < 10s : Dataset petit (< 1GB)
- 10-60s : Dataset moyen (1-10GB)
- > 60s : Dataset large (> 10GB) ou disque lent

**Surveillance** : Augmentation progressive = dataset qui grandit

#### rdb_last_cow_size
**Définition** : Mémoire COW utilisée lors du dernier BGSAVE

**COW (Copy-On-Write)** :
- Lors du fork, pages mémoire partagées
- Modification → copie de la page
- Peak mémoire = used_memory + cow_size

**Exemple** :
```
used_memory: 8GB
rdb_last_cow_size: 2GB
→ Pic à 10GB pendant BGSAVE
```

**Dimensionnement serveur** :
```
RAM_nécessaire = used_memory × 1.5
# Pour absorber le COW
```

### Métriques AOF

#### aof_enabled
**Valeurs** :
- `0` : AOF désactivé
- `1` : AOF activé

**Configuration** :
```conf
appendonly yes
```

#### aof_rewrite_in_progress
**Valeurs** :
- `0` : Pas de rewrite en cours
- `1` : Rewrite en cours

**AOF Rewrite** : Compacte l'AOF en rejouant le dataset

**Impact** :
- Fork (spike latency)
- I/O disque élevé
- COW mémoire

#### aof_current_size
**Définition** : Taille actuelle du fichier AOF (bytes)

**Surveillance** :
```
aof_current_size > 10 × aof_base_size
→ AOF non-compacté, rewrite nécessaire
```

#### aof_base_size
**Définition** : Taille de l'AOF après le dernier rewrite

**Ratio de croissance** :
```
growth_ratio = aof_current_size / aof_base_size
```

**Déclenchement auto-rewrite** :
```conf
auto-aof-rewrite-percentage 100  # Rewrite si 2× la taille base
auto-aof-rewrite-min-size 64mb   # Taille min pour rewrite
```

#### aof_last_rewrite_time_sec
**Définition** : Durée du dernier rewrite

**Valeurs typiques** : Similaires à BGSAVE

#### aof_delayed_fsync
**🔥 MÉTRIQUE CRITIQUE 🔥**

**Définition** : Nombre de fsync retardés à cause de l'I/O disque

**Si > 0** :
- Disque saturé
- AOF fsync bloqué > 2 secondes
- Redis retarde les fsync pour ne pas bloquer

**Conséquence** :
- Perte de données potentielle même avec `appendfsync always`
- Dégradation des performances

**Action** :
1. Vérifier l'I/O disque (`iostat -x`)
2. Considérer SSD NVMe
3. Revoir `appendfsync` policy

## Section STATS : Statistiques globales

### Exemple de sortie

```
# Stats
total_connections_received:1523847
total_commands_processed:98745632
instantaneous_ops_per_sec:2547
total_net_input_bytes:15728640000
total_net_output_bytes:31457280000
instantaneous_input_kbps:256.32
instantaneous_output_kbps:512.64
rejected_connections:0
sync_full:3
sync_partial_ok:127
sync_partial_err:2
expired_keys:45123
expired_stale_perc:0.05
expired_time_cap_reached_count:0
expire_cycle_cpu_milliseconds:1234
evicted_keys:0
evicted_clients:0
total_eviction_exceeded_time:0
current_eviction_exceeded_time:0
keyspace_hits:8547123
keyspace_misses:1245632
pubsub_channels:12
pubsub_patterns:3
pubsubshard_channels:0
latest_fork_usec:2347
total_forks:169
migrate_cached_sockets:0
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0
total_active_defrag_time:0
current_active_defrag_time:0
tracking_total_keys:0
tracking_total_items:0
tracking_total_prefixes:0
unexpected_error_replies:0
total_error_replies:12
dump_payload_sanitizations:0
total_reads_processed:98745632
total_writes_processed:98745632
io_threaded_reads_processed:0
io_threaded_writes_processed:0
reply_buffer_shrinks:0
reply_buffer_expands:0
eventloop_cycles:86400000
eventloop_duration_sum:259200000
instantaneous_eventloop_cycles_per_sec:10
instantaneous_eventloop_duration_usec:3
acl_access_denied_auth:0
acl_access_denied_cmd:0
acl_access_denied_key:0
acl_access_denied_channel:0
```

### Métriques de trafic

#### total_commands_processed
**Définition** : Nombre total de commandes traitées depuis le démarrage

**Calcul du throughput** :
```
avg_ops_per_sec = total_commands_processed / uptime_in_seconds
```

**Croissance linéaire** = Charge stable

#### instantaneous_ops_per_sec
**🔥 MÉTRIQUE CRITIQUE 🔥**

**Définition** : Débit instantané (fenêtre ~1 seconde)

**Valeurs de référence** :
- **< 1000 ops/s** : Charge faible
- **1000-10000 ops/s** : Charge moyenne
- **10000-50000 ops/s** : Charge élevée
- **> 100000 ops/s** : Charge très élevée (nécessite tuning)

**Capacité Redis** :
- Single instance : 100k-500k ops/s (dépend des commandes)
- Cluster : Linéaire avec le nombre de shards

**Surveillance** :
- Établir une baseline
- Alerter sur déviations >200% de la baseline

#### total_net_input_bytes / total_net_output_bytes
**Définition** : Trafic réseau cumulé

**Calcul de la bande passante** :
```python
input_mbps = (total_net_input_bytes / uptime_in_seconds) / 1048576
output_mbps = (total_net_output_bytes / uptime_in_seconds) / 1048576
```

#### rejected_connections
**Définition** : Connexions refusées (maxclients atteint)

**Si > 0** :
1. Augmenter `maxclients`
2. Vérifier connection leak applicatif
3. Considérer connection pooling

### Métriques de cache

#### keyspace_hits / keyspace_misses
**🔥 MÉTRIQUES CRITIQUES 🔥**

**Définition** :
- `keyspace_hits` : Clés trouvées (GET, HGET, etc.)
- `keyspace_misses` : Clés non trouvées

**Calcul du hit ratio** :
```
hit_ratio = keyspace_hits / (keyspace_hits + keyspace_misses)
```

**Objectifs** :

| Use Case | Hit Ratio cible |
|----------|----------------|
| Cache applicatif | > 90% |
| Session store | > 95% |
| Full-page cache | > 80% |
| API cache | > 85% |

**Si hit ratio faible** :
1. TTL trop courts
2. Cache trop petit (évictions)
3. Mauvaise stratégie de clés
4. Pattern d'accès imprévisible

**Monitoring** :
```promql
# Hit ratio sur 5 minutes
rate(redis_keyspace_hits_total[5m]) /
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
```

#### expired_keys
**Définition** : Clés expirées par TTL

**Différence avec evicted_keys** :
- `expired_keys` : TTL naturel ✅ (normal)
- `evicted_keys` : Suppression forcée par manque de mémoire ⚠️ (problème)

**Taux d'expiration** :
```
expiration_rate = expired_keys / uptime_in_seconds
```

#### evicted_keys
**🚨 MÉTRIQUE CRITIQUE 🚨**

**Définition** : Clés supprimées par politique d'éviction

**Si > 0** :
- Mémoire insuffisante
- `maxmemory` trop petit
- Dataset croissant

**Action** :
1. Augmenter `maxmemory` ou RAM
2. Optimiser les données (compression, structures)
3. Revoir les TTL
4. Considérer sharding (Cluster)

**Évictions régulières** = Sous-dimensionnement chronique

### Métriques de réplication

#### sync_full
**Définition** : Nombre de synchronisations complètes

**Déclenchement** :
- Premier sync d'un nouveau replica
- Replica trop désynchronisé (replication backlog débordé)
- Redémarrage master avec nouveau run_id

**Coûteux** : Transfert complet du dataset + BGSAVE

#### sync_partial_ok
**Définition** : Synchronisations partielles réussies

**Optimal** : Ratio sync_partial_ok >> sync_full

#### sync_partial_err
**Définition** : Synchronisations partielles échouées

**Causes** :
- Backlog trop petit
- Replica déconnecté trop longtemps
- Réseau instable

**Si > 0** : Augmenter `repl-backlog-size`

### Métriques de fork

#### latest_fork_usec
**Définition** : Durée du dernier fork (microsecondes)

**Conversion** :
```
latest_fork_ms = latest_fork_usec / 1000
```

**Valeurs typiques** :
- **< 100ms** : Excellent
- **100-500ms** : Bon
- **500-1000ms** : Acceptable
- **> 1000ms** : Problématique (spike de latency)

**Facteurs d'impact** :
1. Taille du dataset
2. Fragmentation mémoire
3. Configuration THP (Transparent Huge Pages)

**Optimisation** :
```bash
# Désactiver THP (recommandé)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

#### total_forks
**Définition** : Nombre total de forks

**Inclut** :
- BGSAVE
- AOF rewrite
- Module forks

### Métriques Pub/Sub

#### pubsub_channels
**Définition** : Nombre de channels Pub/Sub actifs

#### pubsub_patterns
**Définition** : Nombre de patterns Pub/Sub actifs

**Pattern matching** : `PSUBSCRIBE news.*` compte comme 1 pattern

#### pubsubshard_channels (Redis 7+)
**Définition** : Channels du Sharded Pub/Sub

**Avantage** : Scalabilité dans les clusters

## Section REPLICATION : Master-Replica

### Exemple de sortie (Master)

```
# Replication
role:master
connected_slaves:2
slave0:ip=10.0.1.10,port=6379,state=online,offset=87452369,lag=0
slave1:ip=10.0.1.11,port=6379,state=online,offset=87452369,lag=1
master_failover_state:no-failover
master_replid:a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:87452369
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:86403794
repl_backlog_histlen:1048576
```

### Exemple de sortie (Replica)

```
# Replication
role:slave
master_host:10.0.1.5
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_read_repl_offset:87452369
slave_repl_offset:87452369
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:87452369
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:86403794
repl_backlog_histlen:1048576
```

### Métriques Master

#### role
**Valeurs** :
- `master` : Instance principale (writes)
- `slave` : Instance réplica (read-only)

#### connected_slaves
**Définition** : Nombre de replicas connectés

**Surveillance** :
```
connected_slaves < nombre_attendu → Alerte
```

#### slave0, slave1, ... (par replica)
**Format** :
```
ip=X.X.X.X,port=6379,state=online,offset=123456,lag=1
```

**Champs critiques** :
- `state` : `online` (bon) ou `send_bulk` / `wait_bgsave` (sync en cours)
- `offset` : Position dans le replication stream
- `lag` : Secondes depuis le dernier ACK du replica

**Lag** :
- **lag=0** : Synchrone (excellent)
- **lag=1-5** : Normal
- **lag>10** : Problématique
- **lag>60** : Critique

#### master_repl_offset
**Définition** : Position actuelle dans le replication log

**Usage** : Calcul du replication lag

### Métriques Replica

#### master_link_status
**🔥 MÉTRIQUE CRITIQUE 🔥**

**Valeurs** :
- `up` : Connexion au master OK
- `down` : Déconnecté du master

**Si `down`** :
1. Vérifier réseau
2. Master accessible ?
3. Credentials corrects ?
4. Firewall bloquant ?

#### master_last_io_seconds_ago
**Définition** : Secondes depuis la dernière communication

**Surveillance** :
```
master_last_io_seconds_ago > 30 → Warning
master_last_io_seconds_ago > 60 → Critical
```

#### master_sync_in_progress
**Valeurs** :
- `0` : Pas de sync
- `1` : Full sync en cours

**Impact** : Performance dégradée pendant le sync

#### slave_repl_offset
**Définition** : Position du replica dans le replication log

**Calcul du lag** :
```
replication_lag_bytes = master_repl_offset - slave_repl_offset
```

**Lag temporel** :
```
# Approximatif, dépend du throughput
lag_seconds = replication_lag_bytes / avg_bytes_per_sec
```

## Section CPU

### Exemple de sortie

```
# CPU
used_cpu_sys:1234.56
used_cpu_user:5678.90
used_cpu_sys_children:123.45
used_cpu_user_children:234.56
used_cpu_sys_main_thread:1111.11
used_cpu_user_main_thread:4444.44
```

### Métriques

#### used_cpu_sys / used_cpu_user
**Définition** :
- `used_cpu_sys` : Temps CPU système (kernel, I/O)
- `used_cpu_user` : Temps CPU utilisateur (Redis logic)

**Unité** : Secondes cumulées

**Calcul du % CPU** :
```
cpu_usage = ((used_cpu_sys + used_cpu_user) / uptime_in_seconds) × 100
```

**Interprétation** :
- Ratio `sys/user` élevé → Beaucoup d'I/O ou syscalls
- Ratio `user/sys` élevé → CPU-intensive (normal)

#### used_cpu_sys_children / used_cpu_user_children
**Définition** : CPU consommé par les processus forkés (BGSAVE, AOF rewrite)

**Usage** : Mesurer l'impact de la persistence

**Si élevé** : Persistence fréquente ou datasets volumineux

## Section COMMANDSTATS : Performance par commande

### Exemple de sortie

```
# Commandstats
cmdstat_get:calls=12547896,usec=25478963,usec_per_call=2.03,rejected_calls=0,failed_calls=0
cmdstat_set:calls=8745632,usec=17854236,usec_per_call=2.04,rejected_calls=0,failed_calls=0
cmdstat_hgetall:calls=154789,usec=15478963,usec_per_call=100.00,rejected_calls=0,failed_calls=0
cmdstat_keys:calls=42,usec=4200000,usec_per_call=100000.00,rejected_calls=0,failed_calls=0
cmdstat_del:calls=5478963,usec=5478963,usec_per_call=1.00,rejected_calls=0,failed_calls=0
```

### Analyse

**Format** :
```
cmdstat_<COMMAND>:calls=N,usec=T,usec_per_call=A,rejected_calls=R,failed_calls=F
```

#### calls
**Définition** : Nombre d'exécutions de la commande

**Top commands** : Identifier les commandes les plus fréquentes

#### usec_per_call
**🔥 MÉTRIQUE CRITIQUE 🔥**

**Définition** : Latence moyenne par appel (microsecondes)

**Objectifs** :
- **< 100 µs** : Excellent (commandes simples)
- **< 1000 µs (1ms)** : Bon
- **> 10000 µs (10ms)** : Problématique
- **> 100000 µs (100ms)** : Très problématique (commande bloquante)

**Commandes à surveiller** :
- `KEYS` : Toujours O(N), interdit en production
- `HGETALL` : O(N), acceptable si hash petit
- `SMEMBERS` : O(N), idem
- `SORT` : O(N log N), peut être très lent

**Action si latence élevée** :
1. Revoir l'algorithme applicatif
2. Utiliser `SCAN` au lieu de `KEYS`
3. Limiter la taille des structures
4. Considérer `HSCAN`, `SSCAN`

#### rejected_calls
**Redis 7+** : Commandes rejetées par ACLs ou autre

#### failed_calls
**Redis 7+** : Commandes échouées (erreur d'exécution)

## Section ERRORSTATS (Redis 6.2+)

### Exemple de sortie

```
# Errorstats
errorstat_ERR:count=127
errorstat_WRONGTYPE:count=42
errorstat_NOAUTH:count=18
errorstat_READONLY:count=5
```

### Utilisation

**Format** : `errorstat_<TYPE>:count=N`

**Types d'erreurs courants** :
- `ERR` : Erreur générique
- `WRONGTYPE` : Mauvais type de structure
- `NOAUTH` : Authentification manquante
- `READONLY` : Écriture tentée sur replica read-only
- `NOPERM` : ACL violation

**Surveillance** :
- `WRONGTYPE` croissant → Bug applicatif
- `NOAUTH` croissant → Attaque ou configuration incorrecte
- `READONLY` → Application écrit sur replica par erreur

## Section KEYSPACE : Statistiques par DB

### Exemple de sortie

```
# Keyspace
db0:keys=1523478,expires=458963,avg_ttl=3600000
db1:keys=45632,expires=12456,avg_ttl=7200000
db15:keys=1,expires=0,avg_ttl=0
```

### Métriques

#### keys
**Définition** : Nombre de clés dans la DB

**Surveillance** :
- Croissance continue non maîtrisée
- Calcul de la densité mémoire : `used_memory / total_keys`

#### expires
**Définition** : Nombre de clés avec TTL

**Ratio** :
```
ttl_coverage = expires / keys
```

**Interprétation** :
- **100%** : Toutes les clés expirent (cache pur)
- **50%** : Mix cache + données permanentes
- **0%** : Aucune expiration (risque OOM)

#### avg_ttl
**Définition** : TTL moyen des clés (millisecondes)

**Conversion** :
```
avg_ttl_hours = avg_ttl / 3600000
```

## Conclusion : Dashboard opérationnel

### Métriques vitales à afficher

**Tier 1 - Alerting immédiat** :
1. `redis_up`
2. `mem_fragmentation_ratio`
3. `used_memory / maxmemory`
4. `master_link_status` (replicas)
5. `evicted_keys`

**Tier 2 - Surveillance quotidienne** :
6. `hit_ratio`
7. `instantaneous_ops_per_sec`
8. `connected_clients`
9. `replication_lag`
10. `rdb_last_save_time`

**Tier 3 - Analyse performance** :
11. `commandstats` (latence par commande)
12. `latest_fork_usec`
13. `aof_delayed_fsync`
14. `blocked_clients`
15. `mem_clients_slaves`

### Commande recap pour export

```bash
# Export toutes les métriques en format parsable
redis-cli --raw INFO all | awk -F: '/^[a-z]/ {print $1 "=" $2}'

# Ou JSON (avec redis-cli récent)
redis-cli --json INFO all

# Métriques spécifiques en one-liner
redis-cli INFO memory | grep -E "used_memory:|maxmemory:|mem_fragmentation"
```

---

**Prochaine section** : 13.2 - Métriques clés : Hit ratio, Fragmentation, Évictions (deep dive et stratégies)

⏭️ [Métriques clés : Hit ratio, Fragmentation, Évictions](/13-monitoring-observabilite/02-metriques-cles-hit-ratio-fragmentation.md)
