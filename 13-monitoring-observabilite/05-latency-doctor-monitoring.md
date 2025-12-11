🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.5 Latency Doctor et Latency Monitoring

## Introduction

La latence est le KPI le plus critique pour Redis. En tant que système in-memory conçu pour des réponses sub-millisecondes, toute dégradation de latence a un impact immédiat sur l'expérience utilisateur. Cette section explore les outils natifs Redis et les stratégies de monitoring avancées pour détecter, diagnostiquer et résoudre les problèmes de latence.

### Pourquoi la latence est critique

**Redis = Speed** :
```
Latence attendue : 0.1 - 1ms (P99)
Latence problématique : > 10ms
Latence catastrophique : > 100ms
```

**Impact business** :
```
+100ms latence = -1% conversion (Amazon)
+1s latence = -7% conversion (Walmart)
Redis timeout (2s) = Incident P1
```

### Anatomie de la latence Redis

```
┌─────────────────────────────────────────────────────┐
│                Total Latency                        │
│                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │ Network  │→│ Redis    │→│ Command  │→│Network │  │
│  │ (client  │ │ Queue    │ │ Execution│ │(reply) │  │
│  │ →server) │ │ Wait     │ │          │ │        │  │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘  │
│                                                     │
│  Mesurable    Opaque      Mesurable    Mesurable    │
└─────────────────────────────────────────────────────┘
```

**Composantes** :
1. **Network RTT** : Round-trip réseau (client → Redis)
2. **Queue Wait** : Attente dans la queue Redis (single-thread)
3. **Command Execution** : Temps d'exécution de la commande
4. **Reply Network** : Envoi de la réponse au client

## 1. Latency Monitoring Subsystem de Redis

### 1.1 Activation du monitoring

**Configuration** :
```conf
# redis.conf
latency-monitor-threshold 100  # En millisecondes

# Si latency > 100ms → Enregistré dans le subsystem
```

**Activation à chaud** :
```bash
# Activer (seuil 100ms)
redis-cli CONFIG SET latency-monitor-threshold 100

# Vérifier
redis-cli CONFIG GET latency-monitor-threshold

# Désactiver
redis-cli CONFIG SET latency-monitor-threshold 0
```

**Recommandations par environnement** :

| Environnement | Seuil | Rationale |
|---------------|-------|-----------|
| **Production** | 50-100ms | Détecter anomalies sans bruit |
| **Staging** | 25-50ms | Testing plus strict |
| **Dev** | 10-25ms | Maximum sensibilité |
| **Cache critique** | 10ms | SLA strict |

### 1.2 Événements monitorés

**Types d'événements** :
```
command          : Exécution de commande lente
fast-command     : Commande "rapide" anormalement lente
fork             : Fork pour BGSAVE/AOF rewrite
aof-write        : Écriture AOF bloquée
aof-fsync-always : fsync AOF synchrone
aof-fstat        : fstat() sur fichier AOF
aof-rename       : Renommage fichier AOF
aof-rewrite-diff-write : Écriture différentiel AOF
active-defrag-cycle : Cycle de défragmentation active
expire-cycle     : Cycle d'expiration de clés
eviction-cycle   : Cycle d'éviction
```

### 1.3 Commandes LATENCY

#### LATENCY LATEST

**Objectif** : Voir les dernières latences enregistrées

```bash
redis-cli LATENCY LATEST
```

**Output** :
```
1) 1) "command"
   2) (integer) 1702310000    # Timestamp Unix
   3) (integer) 156            # Latence (ms)
   4) (integer) 1247           # Latence max depuis reset

2) 1) "fork"
   2) (integer) 1702309500
   3) (integer) 2345
   4) (integer) 3456
```

**Interprétation** :
- Événement `command` : Dernière commande lente 156ms
- Événement `fork` : Dernier fork a pris 2.3 secondes

#### LATENCY HISTORY

**Objectif** : Historique d'un événement spécifique

```bash
redis-cli LATENCY HISTORY command
```

**Output** :
```
1) 1) (integer) 1702310000    # Timestamp
   2) (integer) 156            # Latence (ms)

2) 1) (integer) 1702309800
   2) (integer) 234

3) 1) (integer) 1702309600
   2) (integer) 189
```

**Limite** : Garde maximum 160 entrées par événement

#### LATENCY GRAPH

**Objectif** : Graphique ASCII de la latence

```bash
redis-cli LATENCY GRAPH command
```

**Output** :
```
command - high 234 ms, low 45 ms (all time high 456 ms)
--------------------------------------------------------------------------------
   #_
  _|
 _|
_|
||
||    _
||   ||
||   ||    #
||   ||   _|
```

**Usage** : Diagnostic rapide en SSH sans Grafana

#### LATENCY DOCTOR

**🔥 Commande la plus puissante 🔥**

```bash
redis-cli LATENCY DOCTOR
```

**Output exemple** :
```
Dave, I have observed latency spikes in this Redis instance.
You don't mind talking about it, do you Dave?

1. command: 12 latency spikes (average 156ms, mean deviation 45ms, period 120.00 sec).
   Worst all time event 234ms.

2. fork: 3 latency spikes (average 2345ms, mean deviation 567ms, period 3600.00 sec).
   Worst all time event 3456ms.

I have a few advices for you:

- Your current Transparent Huge Pages (THP) support seems to be enabled.
  Latency due to forks can be reduced disabling THP.

- The fork() system call took 2345 milliseconds in the last BGSAVE.
  The data set is 8GB, you may want to increase the machine RAM.

- I detected a slow command taking 234ms: KEYS *
  Slow commands can block Redis. Use SCAN instead of KEYS.

- AOF fsync is taking 156ms on average.
  Your disk may be too slow. Consider using faster SSD.
```

**Analyse automatique** :
- Corrélation des événements
- Identification des patterns
- Recommandations actionnables
- Références aux métriques système

#### LATENCY RESET

**Objectif** : Réinitialiser les données de latence

```bash
# Reset tous les événements
redis-cli LATENCY RESET

# Reset un événement spécifique
redis-cli LATENCY RESET command

# Retour
(integer) 1  # Nombre d'événements reset
```

**Use case** : Après correction d'un problème, reset pour confirmer

#### LATENCY HELP

```bash
redis-cli LATENCY HELP
```

**Output** :
```
LATENCY DOCTOR                     -- Return a human-readable latency analysis report.
LATENCY GRAPH <event>              -- Return an ASCII-art graph of event latency.
LATENCY HISTORY <event>            -- Return time-latency samples for event.
LATENCY LATEST                     -- Return latest latency samples for all events.
LATENCY RESET [event]              -- Reset latency data of one or all events.
LATENCY HELP                       -- This help.
```

## 2. Causes de Latence et Diagnostic

### 2.1 Fork Latency (BGSAVE/AOF Rewrite)

**Symptôme** :
```bash
redis-cli LATENCY LATEST
# fork: 2500ms
```

**Cause** : Le fork() duplique l'espace mémoire → latence proportionnelle à la RAM utilisée

**Facteurs aggravants** :
1. **Dataset large** : Plus de RAM = fork plus long
2. **THP activé** : Transparent Huge Pages augmente la latence de fork
3. **Fragmentation élevée** : Plus de pages à copier
4. **Swap actif** : Fork extrêmement lent

**Diagnostic** :
```bash
# 1. Vérifier le temps de fork récent
redis-cli INFO stats | grep latest_fork_usec
# latest_fork_usec:2500000  (2.5 secondes)

# 2. Vérifier THP
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never  ← Problème si [always]

# 3. Vérifier la RAM et fragmentation
redis-cli INFO memory | grep -E "used_memory:|mem_fragmentation"
# used_memory:8589934592  (8GB)
# mem_fragmentation_ratio:1.75
```

**Solutions** :

#### Solution 1 : Désactiver THP
```bash
# Temporaire
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Permanent (systemd)
sudo tee /etc/systemd/system/disable-thp.service > /dev/null <<EOF
[Unit]
Description=Disable Transparent Huge Pages (THP)
Before=redis.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable disable-thp
sudo systemctl start disable-thp
```

**Gain attendu** : Fork latency réduite de 50-70%

#### Solution 2 : Augmenter la RAM
```
Dataset : 8GB
Fork : 2.5s
→ Passer à 16GB RAM
→ Fork estimé : 1.2s
```

#### Solution 3 : Réduire la fréquence de BGSAVE
```conf
# redis.conf - Moins fréquent
save 900 1       # ← Défaut
save 3600 1      # ← Moins fréquent (1h)

# Ou désactiver si RDB non critique
save ""
```

#### Solution 4 : Défragmentation
```conf
# Réduire la fragmentation
activedefrag yes
active-defrag-threshold-lower 10
```

### 2.2 Slow Commands

**Symptôme** :
```bash
redis-cli LATENCY LATEST
# command: 234ms
```

**Cause** : Commandes O(N) sur grandes collections

**Commandes dangereuses** :

| Commande | Complexité | Danger | Alternative |
|----------|-----------|--------|-------------|
| `KEYS *` | O(N) | 🔴 Extrême | `SCAN` |
| `HGETALL` | O(N) | 🟠 Élevé | `HSCAN` |
| `SMEMBERS` | O(N) | 🟠 Élevé | `SSCAN` |
| `ZRANGE 0 -1` | O(N) | 🟠 Élevé | Limiter range |
| `SORT` | O(N log N) | 🔴 Élevé | Éviter ou LIMIT |
| `SUNIONSTORE` | O(N) | 🟠 Moyen | Dépend de N |
| `FLUSHDB` | O(N) | 🟠 Moyen | `FLUSHDB ASYNC` |

**Diagnostic** :

#### SLOWLOG

**Configuration** :
```conf
# redis.conf
slowlog-log-slower-than 10000  # 10ms en microsecondes
slowlog-max-len 128            # Garder 128 entrées
```

**Consultation** :
```bash
# Voir les commandes lentes
redis-cli SLOWLOG GET 10
```

**Output** :
```
1) 1) (integer) 127              # ID unique
   2) (integer) 1702310000       # Timestamp Unix
   3) (integer) 234567           # Durée (microsecondes = 234ms)
   4) 1) "KEYS"                  # Commande
      2) "*"                     # Arguments
   5) "10.0.1.50:54321"          # IP:Port client
   6) ""                         # Client name

2) 1) (integer) 126
   2) (integer) 1702309800
   3) (integer) 156789           # 156ms
   4) 1) "HGETALL"
      2) "user:12345"
   5) "10.0.1.51:54322"
   6) "app-worker-3"
```

**Analyse** :
```bash
# Nombre d'entrées dans le slowlog
redis-cli SLOWLOG LEN
# (integer) 45

# Reset du slowlog
redis-cli SLOWLOG RESET
```

**Monitoring avec script** :
```bash
#!/bin/bash
# slowlog-monitor.sh

while true; do
  redis-cli SLOWLOG GET 10 | grep -A 5 "KEYS\|HGETALL\|SMEMBERS" | \
  logger -t redis-slowlog
  sleep 60
done
```

**Solutions** :

#### Solution 1 : Remplacer KEYS par SCAN
```python
# ❌ Mauvais : Bloquant
keys = redis.keys('user:*')

# ✅ Bon : Non-bloquant
cursor = 0
keys = []
while True:
    cursor, partial_keys = redis.scan(cursor, match='user:*', count=100)
    keys.extend(partial_keys)
    if cursor == 0:
        break
```

#### Solution 2 : Limiter HGETALL
```python
# ❌ Mauvais : Tout le hash
data = redis.hgetall('large_hash')

# ✅ Bon : Champs spécifiques
data = redis.hmget('large_hash', 'field1', 'field2', 'field3')

# ✅ Ou HSCAN si besoin de tout
cursor = 0
data = {}
while True:
    cursor, items = redis.hscan('large_hash', cursor, count=100)
    data.update(dict(zip(items[::2], items[1::2])))
    if cursor == 0:
        break
```

#### Solution 3 : Pagination ZRANGE
```python
# ❌ Mauvais : Tout le sorted set
items = redis.zrange('leaderboard', 0, -1)

# ✅ Bon : Top N seulement
items = redis.zrange('leaderboard', 0, 99)  # Top 100
```

### 2.3 AOF Latency

**Symptôme** :
```bash
redis-cli LATENCY LATEST
# aof-fsync-always: 156ms
# aof-write: 89ms
```

**Cause** : I/O disque bloquant pendant fsync

**Diagnostic** :
```bash
# 1. Vérifier la politique AOF
redis-cli CONFIG GET appendfsync
# appendfsync: everysec  (ou always, no)

# 2. Vérifier AOF delayed fsync
redis-cli INFO persistence | grep aof_delayed_fsync
# aof_delayed_fsync: 12  ← Problème si > 0

# 3. Tester I/O disque
sudo iostat -x 1 10
# Surveiller %util, await, svctm
```

**Impact des politiques** :

| appendfsync | Durabilité | Latence | Use Case |
|-------------|-----------|---------|----------|
| `always` | Maximale | Élevée (100-500ms) | Finance, critique |
| `everysec` | Haute | Faible (< 1ms) | **Recommandé** |
| `no` | Minimale | Minimale | Cache éphémère |

**Solutions** :

#### Solution 1 : Changer la politique
```bash
# Si currently "always" et acceptable de perdre 1s
redis-cli CONFIG SET appendfsync everysec
```

**Trade-off** :
- `always` → `everysec` : Perte max 1s de données, gain latence énorme
- `everysec` → `no` : Perte possible plusieurs secondes, gain latence marginal

#### Solution 2 : Upgrade disque (HDD → SSD)
```
HDD (7200 RPM)  : 100-200 IOPS, latence 10-20ms
SATA SSD        : 50k-100k IOPS, latence 0.1-1ms
NVMe SSD        : 500k-1M IOPS, latence 0.01-0.1ms

Impact sur AOF fsync:
HDD → SSD : Latence ÷ 100
SATA SSD → NVMe : Latence ÷ 10
```

#### Solution 3 : no-appendfsync-on-rewrite
```conf
# redis.conf
no-appendfsync-on-rewrite yes
```

**Effet** : Pendant AOF rewrite, désactive fsync (moins de durabilité, mais pas de spike)

**Recommandé** : `yes` pour éviter les spikes lors des rewrites

### 2.4 Expiration/Eviction Cycles

**Symptôme** :
```bash
redis-cli LATENCY LATEST
# expire-cycle: 45ms
# eviction-cycle: 78ms
```

**Cause** : Trop de clés à expirer/évincer en un cycle

**Diagnostic** :
```bash
# 1. Nombre de clés avec TTL
redis-cli INFO keyspace
# db0:keys=10000000,expires=9500000,avg_ttl=300000

# Ratio : 95% des clés ont un TTL → Potentiellement problématique

# 2. Taux d'expiration
redis-cli INFO stats | grep expired_keys
# expired_keys:8547123

# 3. Évictions
redis-cli INFO stats | grep evicted_keys
# evicted_keys:154789  ← Problème si > 0
```

**Solutions** :

#### Solution 1 : Répartir les expirations
```python
# ❌ Mauvais : Tous les TTL identiques
import time
for i in range(1000000):
    redis.setex(f'key:{i}', 3600, 'value')  # Expiration à la même seconde

# ✅ Bon : TTL avec jitter
import random
for i in range(1000000):
    ttl = 3600 + random.randint(-300, 300)  # ±5 min jitter
    redis.setex(f'key:{i}', ttl, 'value')
```

**Effet** : Répartit les expirations dans le temps → pas de spike

#### Solution 2 : Lazy expiration + Active
```conf
# redis.conf
# Redis expire les clés en deux modes:
# 1. Lazy : À l'accès
# 2. Active : Cycles périodiques (10x/sec par défaut)

# Tuning du cycle actif
hz 10  # Fréquence des tâches de fond (défaut)
```

**Trade-off** :
- `hz` plus élevé (50-100) : Expirations plus rapides, mais plus de CPU
- `hz` plus bas (5) : Moins de CPU, mais expirations plus lentes

#### Solution 3 : Augmenter maxmemory (éviter évictions)
```bash
# Si évictions actives
redis-cli CONFIG SET maxmemory 8gb  # Augmenter
```

### 2.5 Network Latency

**Symptôme** : Latence côté client élevée mais pas dans Redis

**Diagnostic** :

#### Ping RTT
```bash
# Ping depuis le client
redis-cli -h redis-server --latency
# min: 0.12, max: 45.67, avg: 0.89 (1000 samples)

# Ping étendu
redis-cli -h redis-server --latency-history
# Affiche l'historique avec graph
```

**Outils réseau** :
```bash
# 1. Ping ICMP
ping redis-server
# rtt min/avg/max = 0.2/0.5/1.2 ms

# 2. MTR (My Traceroute)
mtr redis-server
# Identifie les hops avec perte de paquets

# 3. iperf (bandwidth test)
# Server
iperf3 -s

# Client
iperf3 -c redis-server
# 9.8 Gbits/sec → Bon
# 100 Mbits/sec → Goulot d'étranglement
```

**Solutions** :

#### Solution 1 : Co-localisation
```
Même datacenter : < 1ms RTT
Cross-datacenter (même région) : 5-15ms RTT
Cross-datacenter (différentes régions) : 50-200ms RTT
Cross-continent : 150-300ms RTT

Recommandation : Client et Redis dans le même subnet/AZ
```

#### Solution 2 : Pipelining
```python
# ❌ Sans pipeline : N × RTT
for i in range(1000):
    redis.get(f'key:{i}')  # 1000 roundtrips

# ✅ Avec pipeline : 1 × RTT
pipe = redis.pipeline()
for i in range(1000):
    pipe.get(f'key:{i}')
results = pipe.execute()  # 1 seul roundtrip
```

**Gain** :
```
Sans pipeline : 1000 × 0.5ms = 500ms
Avec pipeline : 1 × 0.5ms = 0.5ms
Gain : 1000×
```

#### Solution 3 : Connection pooling
```python
# ❌ Nouvelle connexion à chaque requête
def get_user(user_id):
    r = redis.Redis(host='redis-server')  # 3-way handshake
    return r.get(f'user:{user_id}')

# ✅ Connection pool
pool = redis.ConnectionPool(
    host='redis-server',
    max_connections=50,
    decode_responses=True
)
redis_client = redis.Redis(connection_pool=pool)

def get_user(user_id):
    return redis_client.get(f'user:{user_id}')
```

### 2.6 Memory Swap

**Symptôme** : Latence erratique, pics extrêmes (> 1s)

**Diagnostic** :
```bash
# 1. Vérifier si Redis est en swap
redis-cli INFO memory | grep used_memory_rss
# used_memory_rss:10000000000  (10GB)

redis-cli INFO memory | grep used_memory:
# used_memory:8000000000  (8GB)

# RSS > used_memory → Pas de swap (OK)
# RSS < used_memory → SWAP ACTIF (PROBLÈME GRAVE)

# 2. Vérifier le swap système
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi       12Gi       500Mi       100Mi        2.5Gi        2.8Gi
# Swap:          8.0Gi      3.2Gi      4.8Gi  ← 3.2GB utilisés (PROBLÈME)

# 3. Voir quel process utilise le swap
sudo smem -t -k
```

**Solutions** :

#### Solution 1 : Désactiver le swap (production)
```bash
# Temporaire
sudo swapoff -a

# Permanent
sudo sed -i '/swap/d' /etc/fstab
```

**Important** : Serveurs production dédiés à Redis ne devraient JAMAIS swapper

#### Solution 2 : Augmenter la RAM
```
Redis used_memory : 8GB
RAM système : 16GB
→ Pas assez de marge

Recommandation : RAM ≥ 2 × used_memory_peak
Exemple : 8GB dataset → 16GB+ RAM
```

#### Solution 3 : vm.overcommit_memory
```bash
# Vérifier
sysctl vm.overcommit_memory
# vm.overcommit_memory = 0  ← Défaut (pas optimal)

# Configurer pour Redis
sudo sysctl vm.overcommit_memory=1

# Permanent
echo "vm.overcommit_memory=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Explication** :
- `0` : Kernel décide (peut refuser fork)
- `1` : Toujours overcommit (recommandé pour Redis)
- `2` : Ne jamais overcommit au-delà de swap+RAM

## 3. Monitoring de Latence avec Prometheus

### 3.1 Métriques de latence exposées

**Par Redis Exporter** :
```
# Latence moyenne des commandes
redis_command_call_duration_seconds_sum
redis_command_call_duration_seconds_count

# Latence par percentile (si histograms activés)
redis_command_call_duration_seconds_bucket{le="0.001"}  # 1ms
redis_command_call_duration_seconds_bucket{le="0.005"}  # 5ms
redis_command_call_duration_seconds_bucket{le="0.01"}   # 10ms
redis_command_call_duration_seconds_bucket{le="0.05"}   # 50ms
redis_command_call_duration_seconds_bucket{le="0.1"}    # 100ms
redis_command_call_duration_seconds_bucket{le="0.5"}    # 500ms
redis_command_call_duration_seconds_bucket{le="1.0"}    # 1s
redis_command_call_duration_seconds_bucket{le="+Inf"}

# Temps de fork
redis_latest_fork_seconds

# Slowlog count
redis_slowlog_length
```

### 3.2 Requêtes PromQL pour latence

#### Latence moyenne
```promql
# Latence moyenne sur 5 minutes (en ms)
rate(redis_command_call_duration_seconds_sum[5m]) /
rate(redis_command_call_duration_seconds_count[5m]) * 1000
```

#### Latence P50, P95, P99
```promql
# P50 (médiane)
histogram_quantile(0.50,
  rate(redis_command_call_duration_seconds_bucket[5m])
) * 1000

# P95
histogram_quantile(0.95,
  rate(redis_command_call_duration_seconds_bucket[5m])
) * 1000

# P99
histogram_quantile(0.99,
  rate(redis_command_call_duration_seconds_bucket[5m])
) * 1000

# P99.9
histogram_quantile(0.999,
  rate(redis_command_call_duration_seconds_bucket[5m])
) * 1000
```

#### Latence par commande
```promql
# Top 10 commandes par latence moyenne
topk(10,
  rate(redis_command_call_duration_seconds_sum{cmd!=""}[5m]) /
  rate(redis_command_call_duration_seconds_count{cmd!=""}[5m])
) * 1000
```

#### Fork latency
```promql
# Dernière latence de fork (en secondes)
redis_latest_fork_seconds

# Taux de fork
rate(redis_rdb_changes_since_last_save[5m]) > 0
```

#### Slowlog growth
```promql
# Croissance du slowlog
rate(redis_slowlog_length[5m])
```

### 3.3 Recording Rules pour latence

```yaml
# /etc/prometheus/rules/redis_latency.yml
groups:
  - name: redis_latency
    interval: 30s
    rules:
      # Latence moyenne par instance
      - record: redis:latency_avg:ms
        expr: |
          rate(redis_command_call_duration_seconds_sum[5m]) /
          rate(redis_command_call_duration_seconds_count[5m]) * 1000

      # P50
      - record: redis:latency_p50:ms
        expr: |
          histogram_quantile(0.50,
            rate(redis_command_call_duration_seconds_bucket[5m])
          ) * 1000

      # P95
      - record: redis:latency_p95:ms
        expr: |
          histogram_quantile(0.95,
            rate(redis_command_call_duration_seconds_bucket[5m])
          ) * 1000

      # P99
      - record: redis:latency_p99:ms
        expr: |
          histogram_quantile(0.99,
            rate(redis_command_call_duration_seconds_bucket[5m])
          ) * 1000

      # Fork latency élevé
      - record: redis:fork_latency:high
        expr: |
          redis_latest_fork_seconds > 1
```

### 3.4 Alertes latence

```yaml
# /etc/prometheus/rules/redis_latency_alerts.yml
groups:
  - name: redis_latency_alerts
    interval: 30s
    rules:
      # Latence P99 élevée
      - alert: RedisLatencyP99High
        expr: |
          histogram_quantile(0.99,
            rate(redis_command_call_duration_seconds_bucket[5m])
          ) * 1000 > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis latency P99 élevée sur {{ $labels.instance }}"
          description: "P99: {{ $value | humanize }}ms (> 10ms)"

      # Latence P99 critique
      - alert: RedisLatencyP99Critical
        expr: |
          histogram_quantile(0.99,
            rate(redis_command_call_duration_seconds_bucket[5m])
          ) * 1000 > 50
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis latency P99 CRITIQUE sur {{ $labels.instance }}"
          description: "P99: {{ $value | humanize }}ms (> 50ms)"

      # Fork latency élevé
      - alert: RedisForkLatencyHigh
        expr: redis_latest_fork_seconds > 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Fork latency élevé sur {{ $labels.instance }}"
          description: "Fork: {{ $value }}s - Considérer désactivation THP"

      # Slowlog croissant
      - alert: RedisSlowlogGrowing
        expr: rate(redis_slowlog_length[5m]) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Slowlog croissant sur {{ $labels.instance }}"
          description: "{{ $value }} commandes lentes/min - Vérifier SLOWLOG"

      # Baseline deviation
      - alert: RedisLatencyAnomaly
        expr: |
          abs(
            redis:latency_p99:ms -
            avg_over_time(redis:latency_p99:ms[7d])
          ) > (3 * stddev_over_time(redis:latency_p99:ms[7d]))
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Anomalie de latence détectée"
          description: "Latence dévie de +3σ de la moyenne 7j"
```

## 4. Panel Grafana pour Latency

### 4.1 Panel : Latency Percentiles

```json
{
  "title": "Command Latency Percentiles",
  "type": "timeseries",
  "targets": [
    {
      "expr": "histogram_quantile(0.50, rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) * 1000",
      "legendFormat": "P50 - {{instance}}",
      "refId": "A"
    },
    {
      "expr": "histogram_quantile(0.95, rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) * 1000",
      "legendFormat": "P95 - {{instance}}",
      "refId": "B"
    },
    {
      "expr": "histogram_quantile(0.99, rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) * 1000",
      "legendFormat": "P99 - {{instance}}",
      "refId": "C"
    },
    {
      "expr": "histogram_quantile(0.999, rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) * 1000",
      "legendFormat": "P99.9 - {{instance}}",
      "refId": "D"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "ms",
      "min": 0,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"value": 0, "color": "green"},
          {"value": 5, "color": "yellow"},
          {"value": 10, "color": "orange"},
          {"value": 50, "color": "red"}
        ]
      }
    }
  }
}
```

### 4.2 Panel : Latency Heatmap

```json
{
  "title": "Latency Distribution (Heatmap)",
  "type": "heatmap",
  "targets": [
    {
      "expr": "sum(rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) by (le)",
      "format": "heatmap",
      "legendFormat": "{{le}}",
      "refId": "A"
    }
  ],
  "options": {
    "calculate": false,
    "cellGap": 2,
    "color": {
      "mode": "scheme",
      "scheme": "Spectral",
      "steps": 128
    },
    "yAxis": {
      "unit": "s",
      "decimals": 0
    }
  }
}
```

### 4.3 Panel : Fork Latency

```json
{
  "title": "Fork Latency (seconds)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "redis_latest_fork_seconds{instance=~\"$instance\"}",
      "legendFormat": "{{instance}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "s",
      "min": 0,
      "thresholds": {
        "steps": [
          {"value": 0, "color": "green"},
          {"value": 1, "color": "yellow"},
          {"value": 2, "color": "red"}
        ]
      }
    }
  }
}
```

### 4.4 Panel : Slowlog Table

```json
{
  "title": "Slow Commands",
  "type": "table",
  "targets": [
    {
      "expr": "topk(10, sum(rate(redis_command_call_duration_seconds_count{instance=~\"$instance\"}[5m])) by (cmd))",
      "format": "table",
      "instant": true,
      "refId": "A"
    },
    {
      "expr": "topk(10, rate(redis_command_call_duration_seconds_sum{instance=~\"$instance\"}[5m]) / rate(redis_command_call_duration_seconds_count{instance=~\"$instance\"}[5m])) by (cmd) * 1000",
      "format": "table",
      "instant": true,
      "refId": "B"
    }
  ],
  "transformations": [
    {
      "id": "merge"
    },
    {
      "id": "organize",
      "options": {
        "renameByName": {
          "cmd": "Command",
          "Value #A": "Calls/sec",
          "Value #B": "Avg Latency (ms)"
        }
      }
    }
  ]
}
```

## 5. Latency SLI/SLO

### 5.1 Définir des SLI (Service Level Indicators)

**Métriques de latence** :
```
SLI_latency_p50 = P50 latency < 1ms
SLI_latency_p95 = P95 latency < 5ms
SLI_latency_p99 = P99 latency < 10ms
SLI_latency_p999 = P99.9 latency < 50ms
```

### 5.2 Définir des SLO (Service Level Objectives)

**Objectifs de disponibilité** :
```
SLO: 99.9% des requêtes P99 < 10ms

Calcul :
Total requêtes sur 30j : 2.592 milliards (1000 req/s × 30j)
Budget erreur : 0.1% = 2.592 millions de requêtes
→ Maximum 2.592M requêtes > 10ms toléré
```

**Requête PromQL pour SLO compliance** :
```promql
# % de temps où P99 < 10ms
100 * (
  count_over_time(
    (histogram_quantile(0.99,
      rate(redis_command_call_duration_seconds_bucket[5m])
    ) * 1000 < 10)[30d:]
  ) /
  count_over_time(
    histogram_quantile(0.99,
      rate(redis_command_call_duration_seconds_bucket[5m])
    )[30d:]
  )
)
```

**Alerte SLO burn rate** :
```yaml
- alert: RedisLatencySLOBurnRate
  expr: |
    (
      1 -
      (
        sum(rate(redis_command_call_duration_seconds_bucket{le="0.01"}[1h]))
        /
        sum(rate(redis_command_call_duration_seconds_count[1h]))
      )
    ) > 0.001 * 14.4  # 14.4× burn rate
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "SLO budget burning too fast"
    description: "À ce rythme, budget SLO épuisé en 2 jours"
```

## 6. Troubleshooting Playbook

### 6.1 Checklist diagnostic rapide

**Étape 1 : Identifier le type de latence** (2 min)
```bash
# 1. Latency Doctor
redis-cli --latency-doctor

# 2. Latest events
redis-cli LATENCY LATEST

# 3. Slowlog
redis-cli SLOWLOG GET 10

# 4. Fork recent
redis-cli INFO stats | grep latest_fork_usec
```

**Étape 2 : Vérifier les suspects habituels** (5 min)
```bash
# 1. THP
cat /sys/kernel/mm/transparent_hugepage/enabled

# 2. Swap
free -h | grep Swap

# 3. Disk I/O
iostat -x 1 5

# 4. Memory
redis-cli INFO memory | grep -E "used_memory|fragmentation"

# 5. Network
redis-cli --latency
```

**Étape 3 : Corréler avec métriques** (3 min)
```bash
# Grafana dashboard
# Vérifier les panels :
# - Memory usage spike ?
# - Ops/sec spike ?
# - Clients spike ?
# - Évictions actives ?
```

### 6.2 Arbre de décision

```
Latence détectée
    │
    ├─ Événement "fork" ?
    │   ├─ Oui → THP activé ?
    │   │   ├─ Oui → Désactiver THP
    │   │   └─ Non → RAM insuffisante, augmenter
    │   └─ Non → Continuer
    │
    ├─ Événement "command" ?
    │   ├─ Oui → Vérifier SLOWLOG
    │   │   └─ KEYS, HGETALL, etc. ?
    │   │       └─ Remplacer par SCAN, limiter
    │   └─ Non → Continuer
    │
    ├─ Événement "aof-*" ?
    │   ├─ Oui → appendfsync = always ?
    │   │   ├─ Oui → Changer en everysec
    │   │   └─ Non → Disque lent, upgrade SSD
    │   └─ Non → Continuer
    │
    ├─ Événement "expire/eviction" ?
    │   ├─ Oui → Trop de clés expirant simultanément ?
    │   │   └─ Ajouter jitter aux TTL
    │   └─ Non → Continuer
    │
    └─ Network latency client ?
        └─ Oui → Co-location, pipelining, pooling
```

### 6.3 Scripts d'automatisation

**Script de diagnostic complet** :
```bash
#!/bin/bash
# redis-latency-diagnosis.sh

echo "=== Redis Latency Diagnosis ==="
echo ""

echo "1. Latency Doctor:"
redis-cli LATENCY DOCTOR
echo ""

echo "2. Latest Latency Events:"
redis-cli LATENCY LATEST
echo ""

echo "3. Top 5 Slow Commands:"
redis-cli SLOWLOG GET 5
echo ""

echo "4. Fork Latency:"
redis-cli INFO stats | grep latest_fork_usec
echo ""

echo "5. THP Status:"
cat /sys/kernel/mm/transparent_hugepage/enabled
echo ""

echo "6. Swap Usage:"
free -h | grep -E "Mem:|Swap:"
echo ""

echo "7. Disk I/O:"
iostat -x 1 3
echo ""

echo "8. Memory Stats:"
redis-cli INFO memory | grep -E "used_memory:|fragmentation|maxmemory"
echo ""

echo "9. Client Latency (10 pings):"
redis-cli --latency-history -i 1 -c 10
echo ""

echo "=== Diagnosis Complete ==="
```

## 7. Best Practices Récapitulatives

### 7.1 Configuration

- ✅ `latency-monitor-threshold 100` (ou moins en fonction du SLA)
- ✅ `slowlog-log-slower-than 10000` (10ms)
- ✅ THP désactivé (`echo never`)
- ✅ `vm.overcommit_memory=1`
- ✅ Pas de swap sur serveurs dédiés
- ✅ `appendfsync everysec` (pas `always` sauf besoin critique)
- ✅ `no-appendfsync-on-rewrite yes`

### 7.2 Monitoring

- ✅ Alertes sur P99 latency (> 10ms warning, > 50ms critical)
- ✅ Alertes sur fork latency (> 1s)
- ✅ Monitoring du slowlog growth
- ✅ Dashboard avec percentiles (P50, P95, P99, P99.9)
- ✅ SLI/SLO définis et trackés

### 7.3 Code applicatif

- ✅ Utiliser SCAN au lieu de KEYS
- ✅ Limiter HGETALL, SMEMBERS, ZRANGE
- ✅ Pipelining pour requêtes multiples
- ✅ Connection pooling
- ✅ TTL avec jitter
- ✅ Timeouts configurés (2-5s)

### 7.4 Infrastructure

- ✅ SSD NVMe pour persistence
- ✅ RAM ≥ 2× peak memory
- ✅ Client et Redis co-localisés (même AZ/subnet)
- ✅ 10 Gbps+ réseau si possible
- ✅ CPU moderne (latence fork dépend du CPU)

## Conclusion

La latence Redis est un indicateur précoce de problèmes systémiques. Un monitoring proactif avec LATENCY DOCTOR, SLOWLOG, et Prometheus permet de :

1. **Détecter** : Identifier les anomalies avant impact utilisateur
2. **Diagnostiquer** : Comprendre la cause racine (fork, slowlog, I/O, réseau)
3. **Résoudre** : Appliquer les corrections appropriées
4. **Prévenir** : Ajuster configuration et code pour éviter récurrence

**Checklist opérationnelle** :
- [ ] `latency-monitor-threshold` configuré
- [ ] THP désactivé
- [ ] Alertes Prometheus actives
- [ ] Dashboard Grafana avec percentiles
- [ ] SLOWLOG régulièrement consulté
- [ ] SLI/SLO définis
- [ ] Playbook de troubleshooting documenté
- [ ] Tests de latence automatisés (CI/CD)

Une latence maîtrisée = Redis performant = Utilisateurs satisfaits.

---

**Prochaine section** : 13.6 - Alerting : Quand et comment alerter (stratégies avancées)

⏭️ [Alerting : Quand et comment alerter](/13-monitoring-observabilite/06-alerting-quand-comment.md)
