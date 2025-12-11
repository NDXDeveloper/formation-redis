🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.4 - Problèmes de latence : Causes et solutions

## 🎯 Objectifs de cette section

- Comprendre les différentes sources de latence dans Redis
- Maîtriser les outils natifs de diagnostic de latence
- Identifier et résoudre les problèmes de latence système
- Optimiser la latence réseau et applicative
- Mettre en place un monitoring proactif de la latence

---

## 📚 Introduction : La latence dans Redis

### Qu'est-ce que la latence ?

La **latence** est le temps entre l'envoi d'une commande et la réception de la réponse.

```
┌──────────────────────────────────────────────────┐
│  LATENCE TOTALE = RTT + Processing Time          │
├──────────────────────────────────────────────────┤
│                                                  │
│  Client ─────────────────────────> Redis         │
│         ↑                                ↓       │
│         │    Network RTT (Round Trip)    │       │
│         │                                ↓       │
│         │                         Processing     │
│         │                         (Commands)     │
│         │                                ↓       │
│         └────────────────────────────────┘       │
│                                                  │
└──────────────────────────────────────────────────┘

Latence normale Redis : < 1ms (souvent < 0.1ms)
Latence préoccupante : 1-10ms
Latence critique : > 10ms
```

### Les 4 catégories de latence

| Catégorie | Cause | Impact typique | Fréquence |
|-----------|-------|----------------|-----------|
| **Intrinsèque** | Redis lui-même | 0.05-0.5ms | Constant |
| **Système** | OS, fork, swap | 10-1000ms | Sporadique |
| **Réseau** | RTT, bande passante | 0.1-10ms | Variable |
| **Applicative** | Design, big keys | 1-100ms+ | Dépend du code |

### Pourquoi Redis est rapide (normalement)

**Architecture optimisée** :
- Single-threaded → Pas de context switching
- In-memory → Pas d'I/O disque (sauf persistance)
- Multiplexage I/O (epoll/kqueue) → Gestion efficace
- Structures de données optimisées

**Latence attendue** :
```
Commande O(1) locale : 0.05-0.1ms
Commande O(1) réseau : 0.5-1ms
Commande O(N) petite : 1-5ms
Commande O(N) grosse : 10-100ms+
```

---

## 🔧 Outils natifs de diagnostic de latence

### LATENCY DOCTOR : Le diagnostic automatisé

**Commande** :
```bash
redis-cli --latency-doctor
```

**Fonctionnement** :
- Exécute une série de tests de latence
- Analyse les événements de latence enregistrés
- Fournit un diagnostic et des recommandations

**Sortie exemple** :

```
Dave, I have observed latency spikes in this Redis instance.
You don't mind talking about it, do you Dave?

1. command: 5 latency spikes (average 300ms, mean deviation 120ms, period 21.00 sec).
   Worst all time event 500ms.

I have a few advices for you:

- Your current Transparent Huge Pages (THP) setting seem to be problematic.
  Check CONFIG GET slowlog-log-slower-than and CONFIG SET slowlog-log-slower-than.

- I detected a high latency in the background SAVE. This may be related to
  slow disk or huge databases. Use smaller RDB snapshots intervals.

- The system is configured to use swap. This may cause latency spikes.
  Check your swap usage with 'free' command.
```

### LATENCY MONITOR : Enregistrement des pics

#### Configuration

```bash
# Activer le monitoring de latence (seuil en millisecondes)
CONFIG SET latency-monitor-threshold 100

# Dans redis.conf
latency-monitor-threshold 100
```

**Seuils recommandés** :

| Environnement | Seuil | Justification |
|---------------|-------|---------------|
| **Production critique** | 10ms | Détection précoce |
| **Production standard** | 50ms | Équilibre |
| **Développement** | 5ms | Profiling agressif |

#### Commandes LATENCY

##### 1. LATENCY LATEST

Affiche les derniers événements de latence par type.

```bash
redis-cli LATENCY LATEST
```

**Sortie** :
```
1) 1) "command"           # Type d'événement
   2) (integer) 1702293847 # Timestamp
   3) (integer) 250        # Latence en ms
   4) (integer) 500        # Latence max

2) 1) "fast-command"
   2) (integer) 1702293850
   3) (integer) 150
   4) (integer) 300

3) 1) "fork"
   2) (integer) 1702293845
   3) (integer) 1200
   4) (integer) 2500
```

##### 2. LATENCY HISTORY <event>

Historique détaillé d'un type d'événement.

```bash
redis-cli LATENCY HISTORY command
```

**Sortie** :
```
1) 1) (integer) 1702293847 # Timestamp
   2) (integer) 250         # Latence

2) 1) (integer) 1702293850
   2) (integer) 180

3) 1) (integer) 1702293855
   2) (integer) 320
```

##### 3. LATENCY GRAPH <event>

Graphique ASCII de la latence dans le temps.

```bash
redis-cli LATENCY GRAPH command
```

**Sortie** :
```
command - high 500 ms, low 100 ms (all time high 500 ms)
--------------------------------------------------------------------------------
   #_
  _| |
 _|  |_
|     ||
+---------------------------------------------------------------------+ 300ms
```

##### 4. LATENCY RESET [event]

Réinitialiser les données de latence.

```bash
# Reset tous les événements
redis-cli LATENCY RESET

# Reset un événement spécifique
redis-cli LATENCY RESET command
```

### Types d'événements de latence

| Événement | Description | Cause typique |
|-----------|-------------|---------------|
| **command** | Commandes lentes | O(N), big keys |
| **fast-command** | Commandes rapides lentes | Système surchargé |
| **fork** | Fork pour RDB/AOF | Copy-on-write, huge DB |
| **rdb-unlink-temp-file** | Suppression fichier RDB | I/O disque lent |
| **aof-write** | Écriture AOF | Disque lent, fsync |
| **aof-fsync-always** | fsync synchrone AOF | Disque lent |
| **aof-rewrite-diff-write** | Écriture diff AOF rewrite | I/O disque |
| **aof-rename** | Rename fichier AOF | Système de fichiers |

---

## 🔍 Tests de latence avec redis-cli

### --latency : Test simple

```bash
redis-cli --latency
```

**Sortie** :
```
min: 0, max: 2, avg: 0.15 (1254 samples)
```

Envoie des PING en continu et mesure le temps de réponse.

### --latency-history : Historique avec intervalles

```bash
redis-cli --latency-history -i 1
```

**Sortie** :
```
min: 0, max: 1, avg: 0.12 (1420 samples) -- 1.00 seconds range
min: 0, max: 2, avg: 0.14 (1398 samples) -- 2.00 seconds range
min: 0, max: 15, avg: 0.18 (1385 samples) -- 3.00 seconds range  ⚠️ Spike!
min: 0, max: 1, avg: 0.13 (1410 samples) -- 4.00 seconds range
```

**Usage** : Détecter les pics de latence sporadiques.

### --latency-dist : Distribution de latence

```bash
redis-cli --latency-dist
```

**Sortie** (distribution interactive en temps réel) :
```
      (ms)  |    ====================
    < 0.1   |    ████████████████████ 95.2%
    0.1-0.5 |    ███ 4.3%
    0.5-1   |    █ 0.4%
    1-10    |    ▏0.1%
    > 10    |    ▏0.0%
```

### --intrinsic-latency : Latence intrinsèque système

Mesure la latence **minimale** imposée par le système (sans Redis).

```bash
redis-cli --intrinsic-latency 60
```

**Sortie** :
```
Max latency so far: 1 microseconds.
Max latency so far: 15 microseconds.
Max latency so far: 1247 microseconds.  ⚠️ Problème système!

1428327840 total runs (avg latency: 0.0420 microseconds / 42.02 nanoseconds per run).
Worst run took 29667x longer than the average latency.
```

**Interprétation** :
- < 100μs : Excellent système
- 100-500μs : Acceptable
- > 1000μs (1ms) : Problème système (CPU throttling, hyperviseur, etc.)

---

## 🎭 Les 7 causes principales de latence

### Cause 1 : Fork pour persistance (RDB/AOF)

#### Symptôme

```bash
redis-cli LATENCY LATEST
# 1) "fork"
#    2) (integer) 1702293847
#    3) (integer) 2500    # 2.5 secondes!
#    4) (integer) 5000    # Max: 5 secondes
```

#### Explication technique

**Pourquoi le fork est coûteux** :

```
┌─────────────────────────────────────────────────┐
│  BGSAVE/BGREWRITEAOF Process                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Redis appelle fork()                        │
│     ↓                                           │
│  2. Le kernel copie la page table               │
│     (Pour 10GB de data = ~50MB de page table)   │
│     ↓                                           │
│  3. Pendant ce temps : TOUTES les commandes     │
│     sont BLOQUÉES                               │
│     ↓                                           │
│  4. Fork terminé → Processing reprend           │
│                                                 │
│  Temps de fork : 10ms par GB de RAM             │
│  (peut être bien pire avec THP)                 │
└─────────────────────────────────────────────────┘
```

**Calcul du temps de fork** :
```
DB size : 10GB → Fork time : ~100ms (sans THP)
DB size : 50GB → Fork time : ~500ms (sans THP)
DB size : 100GB → Fork time : ~1s (sans THP)

Avec THP activé : peut multiplier par 10-20x !
```

#### Diagnostic

```bash
# Vérifier la fréquence des BGSAVE
redis-cli INFO persistence | grep rdb_changes_since_last_save

# Vérifier l'historique des fork
redis-cli LATENCY HISTORY fork

# Vérifier THP (Transparent Huge Pages)
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never  ← Problématique
# always [madvise] never  ← OK
# always madvise [never]  ← Meilleur pour Redis
```

#### Solutions

**Solution 1 : Désactiver THP (recommandé)**

```bash
# Temporaire
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Permanent (dans /etc/rc.local ou systemd)
cat << 'EOF' > /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=redis.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF

systemctl daemon-reload
systemctl enable disable-thp
systemctl start disable-thp
```

**Solution 2 : Ajuster la fréquence de BGSAVE**

```conf
# redis.conf
# Au lieu de snapshots fréquents :
# save 900 1
# save 300 10
# save 60 10000

# Réduire la fréquence :
save 3600 1
save 1800 100
save 300 10000
```

**Solution 3 : Utiliser un replica pour la persistance**

```
┌──────────────┐          ┌──────────────┐
│   Master     │  sync    │   Replica    │
│ (no persist) ├─────────>│ (RDB/AOF)    │
└──────────────┘          └──────────────┘
     ↑
     │ Writes
     │
  Clients
```

**Configuration master** :
```conf
# redis-master.conf
save ""              # Pas de RDB
appendonly no        # Pas de AOF
```

**Configuration replica** :
```conf
# redis-replica.conf
replicaof master-ip 6379
save 900 1
appendonly yes
```

### Cause 2 : Swap (Catastrophique!)

#### Symptôme

```bash
# Latence extrême et variable
redis-cli --latency-history
# min: 0, max: 5000, avg: 250.00  ← 5 secondes!
```

#### Diagnostic

```bash
# Vérifier l'utilisation du swap
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi       14Gi       500Mi        10Mi       1.0Gi       800Mi
# Swap:         8.0Gi       2.0Gi      6.0Gi  ← PROBLÈME!

# Vérifier si Redis utilise le swap
cat /proc/$(pgrep redis-server)/smaps | grep -A 10 "Swap:"

# Vérifier vm.swappiness
cat /proc/sys/vm/swappiness
# 60  ← Trop élevé pour Redis
```

#### Explication

**Pourquoi le swap tue Redis** :

```
RAM access  : 0.0001ms (100 nanoseconds)
SSD access  : 0.1ms    (100 microseconds)  → 1000x plus lent
HDD access  : 10ms     (10 milliseconds)   → 100,000x plus lent!

Si une clé accédée est swappée → blocage de 10-100ms
```

#### Solutions

**Solution 1 : Désactiver complètement le swap**

```bash
# Temporaire
swapoff -a

# Permanent
# Commenter les lignes swap dans /etc/fstab
sudo sed -i '/swap/s/^/#/' /etc/fstab

# Vérifier
free -h | grep Swap
# Swap:            0B          0B          0B  ✅
```

**Solution 2 : Réduire vm.swappiness**

```bash
# Temporaire
sysctl vm.swappiness=1

# Permanent
echo "vm.swappiness=1" >> /etc/sysctl.conf
sysctl -p
```

**Solution 3 : Augmenter la RAM ou réduire maxmemory**

```bash
# Option A : Plus de RAM physique
# Option B : Réduire maxmemory
redis-cli CONFIG SET maxmemory 8gb

# Option C : Partitionner (sharding)
```

### Cause 3 : Overcommit Memory désactivé

#### Symptôme

Lors d'un BGSAVE :
```
# Log Redis :
# Background saving error: Cannot allocate memory
# Latence élevée car BGSAVE échoue et retry
```

#### Diagnostic

```bash
cat /proc/sys/vm/overcommit_memory
# 0  ← Problématique (heuristic)
# 1  ← Bon pour Redis (always)
# 2  ← Jamais overcommit
```

#### Explication

**Pourquoi Redis a besoin d'overcommit** :

```
Fork nécessite théoriquement 2x la RAM :
- Processus parent : 10GB
- Processus child (fork) : 10GB
→ Total : 20GB

Mais grâce au Copy-on-Write (CoW) :
- En réalité : 10GB + quelques MB seulement

Sans overcommit_memory=1 :
- Le kernel refuse le fork si RAM < 20GB
- Même si CoW n'utilisera que 10GB + overhead
```

#### Solution

```bash
# Temporaire
sysctl vm.overcommit_memory=1

# Permanent
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
sysctl -p

# Vérifier
cat /proc/sys/vm/overcommit_memory
# 1  ✅
```

### Cause 4 : Latence réseau

#### Symptôme

```bash
redis-cli --latency
# min: 2, max: 10, avg: 5.00 (1254 samples)

# Mais LATENCY LATEST montre des commandes rapides :
redis-cli LATENCY LATEST
# Pas d'événements enregistrés

# → La latence est ailleurs (réseau)
```

#### Diagnostic

**Test 1 : Ping réseau**

```bash
# Ping l'hôte Redis
ping -c 100 redis-host

# Résultats attendus :
# LAN : 0.1-0.5ms
# VPC cloud : 0.5-2ms
# Cross-AZ : 2-5ms
# Cross-region : 50-200ms
```

**Test 2 : Comparer local vs distant**

```bash
# Test local (sur le serveur Redis)
redis-cli --latency
# min: 0, max: 1, avg: 0.10

# Test distant (depuis l'application)
redis-cli -h redis-host --latency
# min: 2, max: 10, avg: 5.00

# Différence = latence réseau (RTT)
```

**Test 3 : MTU et fragmentation**

```bash
# Tester différentes tailles de paquets
ping -c 10 -M do -s 1472 redis-host  # MTU 1500
ping -c 10 -M do -s 8972 redis-host  # MTU 9000 (jumbo frames)

# Si échec avec gros paquets → problème MTU
```

#### Solutions

**Solution 1 : Optimiser le RTT**

```bash
# Options réseau pour Redis (redis.conf)
tcp-backlog 511           # File d'attente TCP
tcp-keepalive 300         # Keepalive en secondes

# Options TCP système (sysctl)
net.core.somaxconn=65535
net.ipv4.tcp_max_syn_backlog=65535
net.core.netdev_max_backlog=65535
```

**Solution 2 : Pipelining**

Au lieu de :
```python
# ❌ Sans pipelining : 3 RTT
r.set('key1', 'val1')  # RTT
r.set('key2', 'val2')  # RTT
r.set('key3', 'val3')  # RTT
```

Faire :
```python
# ✅ Avec pipelining : 1 RTT
pipe = r.pipeline()
pipe.set('key1', 'val1')
pipe.set('key2', 'val2')
pipe.set('key3', 'val3')
pipe.execute()
```

**Gain** : 3x moins de RTT → 3x plus rapide sur réseau lent.

**Solution 3 : Jumbo Frames (si possible)**

```bash
# Augmenter MTU à 9000 (datacenter uniquement)
ip link set eth0 mtu 9000

# Vérifier
ip link show eth0 | grep mtu
# mtu 9000  ✅
```

**Solution 4 : Colocation application/Redis**

```
❌ Mauvais : App et Redis dans différents AZ
   Latence : 5-10ms

✅ Bon : App et Redis dans le même AZ
   Latence : 0.5-2ms

✅ Meilleur : App et Redis sur le même hôte (sidecar)
   Latence : 0.1-0.5ms
```

### Cause 5 : Big Keys

#### Symptôme

```bash
redis-cli LATENCY LATEST
# 1) "command"
#    2) (integer) 1702293847
#    3) (integer) 50     # Certaines commandes prennent 50ms
#    4) (integer) 200    # Max : 200ms

redis-cli SLOWLOG GET 5
# Montre HGETALL, SMEMBERS, LRANGE sur de grosses clés
```

#### Diagnostic détaillé

Voir section 14.2 pour l'analyse complète des big keys.

**Quick check** :
```bash
redis-cli --bigkeys -i 0.1

# Si trouve des clés > 1MB ou > 10K éléments :
# → Probable cause de latence
```

#### Solutions

**Solution 1 : Fragmenter les big keys**

```bash
# ❌ Une grosse hash
HSET user:12345 field1 val1 field2 val2 ... field100000 val100000

# ✅ Plusieurs petites hash
HSET user:12345:profile name "John" email "john@example.com"
HSET user:12345:prefs theme "dark" lang "en"
HSET user:12345:stats visits 1500 lastLogin "2024-12-11"
```

**Solution 2 : Utiliser SCAN au lieu de XGETALL**

```bash
# ❌ HGETALL sur grosse hash → bloque 50ms
HGETALL user:12345:all

# ✅ HSCAN avec pagination
HSCAN user:12345:all 0 COUNT 100
```

**Solution 3 : Lazy deletion**

```bash
# ❌ DEL sur grosse clé → bloque
DEL huge_list

# ✅ UNLINK (async deletion)
UNLINK huge_list
```

### Cause 6 : AOF fsync

#### Symptôme

```bash
redis-cli LATENCY LATEST
# 1) "aof-fsync-always"
#    2) (integer) 1702293847
#    3) (integer) 15
#    4) (integer) 50

redis-cli INFO persistence | grep aof
# aof_enabled:1
# aof_fsync:always  ← Cause de latence
```

#### Explication

**Les 3 modes AOF fsync** :

```
appendfsync no
├─ Fsync géré par l'OS (tous les 30s typiquement)
├─ Performances : EXCELLENTES (pas de blocage)
└─ Durabilité : FAIBLE (perte possible de 30s de données)

appendfsync everysec  ← RECOMMANDÉ
├─ Fsync toutes les secondes (thread séparé)
├─ Performances : BONNES (impact minimal)
└─ Durabilité : BONNE (perte max 1s)

appendfsync always
├─ Fsync après CHAQUE commande (synchrone)
├─ Performances : MAUVAISES (bloque à chaque write)
└─ Durabilité : MAXIMALE (aucune perte)
```

**Impact de `fsync always`** :

```
Sans fsync     : 100,000 ops/sec, latence 0.1ms
Avec fsync     : 500 ops/sec,     latence 20ms

Réduction : 200x le throughput !
```

#### Solution

**Changer le mode AOF** :

```bash
# Temporaire
redis-cli CONFIG SET appendfsync everysec

# Permanent (redis.conf)
appendfsync everysec

# Vérifier
redis-cli CONFIG GET appendfsync
# 1) "appendfsync"
# 2) "everysec"  ✅
```

**Ou désactiver AOF si RDB suffit** :

```bash
# Si vous acceptez la perte de données depuis le dernier snapshot
redis-cli CONFIG SET appendonly no
```

### Cause 7 : Slow I/O (disque lent)

#### Symptôme

```bash
redis-cli LATENCY LATEST
# 1) "aof-write"
#    2) (integer) 1702293847
#    3) (integer) 100
#    4) (integer) 500

# ou

# 1) "rdb-unlink-temp-file"
#    2) (integer) 1702293847
#    3) (integer) 300
#    4) (integer) 1000
```

#### Diagnostic

**Tester les performances du disque** :

```bash
# Test écriture séquentielle
dd if=/dev/zero of=/var/lib/redis/testfile bs=1M count=1000 oflag=direct
# Résultats attendus :
# SSD  : 200-500 MB/s
# HDD  : 50-100 MB/s
# Network storage : variable

# Test avec fsync (plus réaliste)
dd if=/dev/zero of=/var/lib/redis/testfile bs=4k count=10000 oflag=dsync
# Mesure les performances avec fsync après chaque écriture

# Vérifier les I/O wait
iostat -x 1
# Si %iowait > 10% → disque saturé
```

**Identifier le processus consommant l'I/O** :

```bash
iotop -o
# Affiche les processus triant par I/O

# Si redis-server en top avec writes constants :
# → AOF ou RDB trop fréquents
```

#### Solutions

**Solution 1 : Utiliser un disque plus rapide**

```
HDD (7200 RPM)  : 100 IOPS, 100 MB/s
SSD SATA        : 50K IOPS, 500 MB/s
NVMe SSD        : 500K IOPS, 3000 MB/s
```

**Solution 2 : Optimiser la configuration AOF**

```conf
# redis.conf

# Réduire le rewrite du AOF
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 128mb

# Si disque vraiment lent, considérer :
appendfsync no
# Ou
appendfsync everysec
no-appendfsync-on-rewrite yes  # Pas de fsync pendant rewrite
```

**Solution 3 : Séparer les I/O**

```bash
# RDB et AOF sur des disques différents
mkdir /mnt/fast-disk/redis-aof
mkdir /mnt/cheap-disk/redis-rdb

# redis.conf
dir /mnt/cheap-disk/redis-rdb
appenddirname /mnt/fast-disk/redis-aof
```

---

## 📊 Méthodologie de diagnostic systématique

### Framework LATENCY-CHECK

```
L - Look at LATENCY DOCTOR
A - Analyze system metrics (CPU, RAM, I/O)
T - Test with redis-cli --latency tools
E - Examine SLOWLOG
N - Network RTT measurement
C - Check client-side code
Y - Your persistence configuration

C - Correlate events
H - History of latency spikes
E - Environment (VM, container, cloud)
C - Configuration audit
K - Kill problematic clients if needed
```

### Processus pas à pas

#### Étape 1 : Collecte initiale (< 2 minutes)

```bash
#!/bin/bash
# quick-latency-check.sh

echo "=== QUICK LATENCY CHECK ==="
echo ""

# 1. LATENCY DOCTOR
echo "--- LATENCY DOCTOR ---"
redis-cli LATENCY DOCTOR
echo ""

# 2. LATENCY LATEST
echo "--- LATENCY LATEST ---"
redis-cli LATENCY LATEST
echo ""

# 3. Test de latence local
echo "--- LOCAL LATENCY TEST (10s) ---"
timeout 10 redis-cli --latency
echo ""

# 4. Métriques système
echo "--- SYSTEM METRICS ---"
echo "CPU:"
top -bn1 | grep "Cpu(s)" | awk '{print "  User: "$2", System: "$4", IOWait: "$10}'

echo "Memory:"
free -h | grep -E "Mem:|Swap:"

echo "Disk I/O:"
iostat -x 1 2 | tail -n +4 | awk 'NR>1 {print "  "$1": "$4" r/s, "$5" w/s, "$10"% util"}'

# 5. SLOWLOG
echo ""
echo "--- SLOWLOG (last 5) ---"
redis-cli SLOWLOG GET 5 | grep -E "integer|^[0-9]"

# 6. Config suspicion
echo ""
echo "--- CONFIGURATION CHECK ---"
echo "THP: $(cat /sys/kernel/mm/transparent_hugepage/enabled)"
echo "Swappiness: $(cat /proc/sys/vm/swappiness)"
echo "Overcommit: $(cat /proc/sys/vm/overcommit_memory)"
echo "AOF fsync: $(redis-cli CONFIG GET appendfsync | tail -1)"
```

#### Étape 2 : Analyse approfondie (< 10 minutes)

```python
#!/usr/bin/env python3
"""
Analyse approfondie de latence Redis
"""
import redis
import time
import statistics
from datetime import datetime

class LatencyAnalyzer:
    def __init__(self, host='localhost', port=6379):
        self.r = redis.Redis(host=host, port=port, decode_responses=True)

    def measure_command_latency(self, iterations=1000):
        """Mesure la latence de différentes commandes"""

        commands = {
            'PING': lambda: self.r.ping(),
            'GET': lambda: self.r.get('test_key'),
            'SET': lambda: self.r.set('test_key', 'value'),
            'INCR': lambda: self.r.incr('counter'),
            'LPUSH': lambda: self.r.lpush('list', 'item'),
        }

        results = {}

        for cmd_name, cmd_func in commands.items():
            latencies = []

            for _ in range(iterations):
                start = time.perf_counter()
                try:
                    cmd_func()
                except:
                    pass
                end = time.perf_counter()

                latencies.append((end - start) * 1000)  # ms

            results[cmd_name] = {
                'min': min(latencies),
                'max': max(latencies),
                'avg': statistics.mean(latencies),
                'p50': statistics.median(latencies),
                'p95': sorted(latencies)[int(len(latencies) * 0.95)],
                'p99': sorted(latencies)[int(len(latencies) * 0.99)]
            }

        return results

    def analyze_latency_events(self):
        """Analyse les événements de latence enregistrés"""

        try:
            latest = self.r.execute_command('LATENCY', 'LATEST')
        except:
            return "Latency monitoring not enabled"

        if not latest:
            return "No latency events recorded"

        events = {}
        for event in latest:
            event_name = event[0]
            timestamp = event[1]
            latency = event[2]
            max_latency = event[3]

            events[event_name] = {
                'timestamp': datetime.fromtimestamp(timestamp).isoformat(),
                'last_latency_ms': latency,
                'max_latency_ms': max_latency
            }

        return events

    def check_system_impact(self):
        """Vérifie l'impact système"""

        info = self.r.info()

        issues = []

        # Vérifier la mémoire
        used_memory = info['used_memory']
        maxmemory = info.get('maxmemory', 0)

        if maxmemory > 0 and used_memory > maxmemory * 0.9:
            issues.append({
                'severity': 'warning',
                'category': 'memory',
                'message': f"Memory usage high: {used_memory/1024/1024:.0f}MB / {maxmemory/1024/1024:.0f}MB"
            })

        # Vérifier les évictions
        if info.get('evicted_keys', 0) > 0:
            issues.append({
                'severity': 'warning',
                'category': 'memory',
                'message': f"Keys are being evicted: {info['evicted_keys']}"
            })

        # Vérifier les rejected connections
        if info.get('rejected_connections', 0) > 0:
            issues.append({
                'severity': 'critical',
                'category': 'connections',
                'message': f"Connections rejected: {info['rejected_connections']}"
            })

        # Vérifier le ratio de fragmentation
        frag_ratio = info.get('mem_fragmentation_ratio', 1.0)
        if frag_ratio > 1.5:
            issues.append({
                'severity': 'warning',
                'category': 'memory',
                'message': f"High fragmentation: {frag_ratio:.2f}"
            })

        return issues

    def run_complete_analysis(self):
        """Analyse complète"""

        print("=== LATENCY ANALYSIS ===\n")

        # 1. Command latency
        print("--- Command Latency (1000 iterations) ---")
        latencies = self.measure_command_latency()

        print(f"{'Command':<10} {'Min':<8} {'Avg':<8} {'P95':<8} {'P99':<8} {'Max':<8}")
        print("-" * 60)
        for cmd, stats in latencies.items():
            print(f"{cmd:<10} {stats['min']:<8.3f} {stats['avg']:<8.3f} {stats['p95']:<8.3f} {stats['p99']:<8.3f} {stats['max']:<8.3f}")

        # 2. Latency events
        print("\n--- Recorded Latency Events ---")
        events = self.analyze_latency_events()
        if isinstance(events, str):
            print(events)
        else:
            for event_name, data in events.items():
                print(f"{event_name}:")
                print(f"  Last: {data['last_latency_ms']}ms at {data['timestamp']}")
                print(f"  Max:  {data['max_latency_ms']}ms")

        # 3. System impact
        print("\n--- System Issues ---")
        issues = self.check_system_impact()
        if not issues:
            print("No issues detected ✅")
        else:
            for issue in issues:
                severity = issue['severity'].upper()
                print(f"[{severity}] {issue['message']}")

        # 4. Recommendations
        print("\n--- Recommendations ---")

        if any(stat['p99'] > 1 for stat in latencies.values()):
            print("⚠️  P99 latency > 1ms. Check network or system load.")

        if any(stat['max'] > 10 for stat in latencies.values()):
            print("⚠️  Max latency > 10ms. Run LATENCY DOCTOR for details.")

        if events and isinstance(events, dict):
            if 'fork' in events:
                print("⚠️  Fork latency detected. Consider disabling THP or reducing save frequency.")
            if 'aof-fsync-always' in events:
                print("⚠️  AOF fsync latency. Consider changing to 'everysec'.")

if __name__ == "__main__":
    analyzer = LatencyAnalyzer()
    analyzer.run_complete_analysis()
```

#### Étape 3 : Résolution ciblée (< 30 minutes)

Basé sur l'analyse, appliquer la solution appropriée :

```bash
#!/bin/bash
# latency-fix.sh

echo "Select the issue to fix:"
echo "1) THP (Transparent Huge Pages)"
echo "2) Swap"
echo "3) Overcommit memory"
echo "4) AOF fsync"
echo "5) Save frequency"
echo "6) All system optimizations"

read -p "Choice: " choice

case $choice in
    1)
        echo "Disabling THP..."
        echo never > /sys/kernel/mm/transparent_hugepage/enabled
        echo never > /sys/kernel/mm/transparent_hugepage/defrag
        echo "✅ THP disabled"
        ;;
    2)
        echo "Disabling swap..."
        swapoff -a
        sysctl vm.swappiness=1
        echo "✅ Swap disabled"
        ;;
    3)
        echo "Setting overcommit_memory=1..."
        sysctl vm.overcommit_memory=1
        echo "✅ Overcommit enabled"
        ;;
    4)
        echo "Changing AOF fsync to everysec..."
        redis-cli CONFIG SET appendfsync everysec
        echo "✅ AOF fsync changed"
        ;;
    5)
        echo "Reducing save frequency..."
        redis-cli CONFIG SET save "3600 1 1800 100 300 10000"
        echo "✅ Save frequency reduced"
        ;;
    6)
        echo "Applying all optimizations..."

        # THP
        echo never > /sys/kernel/mm/transparent_hugepage/enabled
        echo never > /sys/kernel/mm/transparent_hugepage/defrag

        # Swap
        sysctl vm.swappiness=1

        # Overcommit
        sysctl vm.overcommit_memory=1

        # TCP
        sysctl net.core.somaxconn=65535

        # AOF
        redis-cli CONFIG SET appendfsync everysec

        # Save
        redis-cli CONFIG SET save "3600 1 1800 100 300 10000"

        echo "✅ All optimizations applied"
        ;;
    *)
        echo "Invalid choice"
        exit 1
        ;;
esac

echo ""
echo "Testing latency after fix..."
timeout 5 redis-cli --latency
```

---

## 📈 Monitoring et alertes

### Métriques Prometheus

```yaml
# prometheus-rules.yml
groups:
  - name: redis_latency
    rules:
      # Alerte : Latence P99 élevée
      - alert: RedisHighLatency
        expr: redis_command_duration_seconds{quantile="0.99"} > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis P99 latency is high"
          description: "{{ $labels.instance }} P99 latency is {{ $value }}s"

      # Alerte : Fork latency
      - alert: RedisForkLatency
        expr: redis_latency_last{event="fork"} > 1000
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Redis fork latency is high"
          description: "{{ $labels.instance }} fork latency: {{ $value }}ms"

      # Alerte : AOF latency
      - alert: RedisAOFLatency
        expr: redis_latency_last{event="aof-write"} > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis AOF write latency is high"
          description: "{{ $labels.instance }} AOF latency: {{ $value }}ms"
```

### Dashboard Grafana

Panels essentiels :

```
1. Command Latency (time series)
   - P50, P95, P99
   - Par type de commande

2. Latency Events (table)
   - Fork, AOF, RDB
   - Timestamp et durée

3. System Impact (gauges)
   - CPU iowait
   - Memory fragmentation
   - Swap usage

4. Network Latency (time series)
   - RTT measurements
   - Bandwidth usage
```

---

## 🎓 Best Practices

### Configuration optimale

```conf
# redis.conf - Configuration anti-latence

# 1. Désactiver THP (via système, pas Redis)

# 2. Persistence optimisée
save 3600 1
save 1800 100
save 300 10000
appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 128mb

# 3. Réseau optimisé
tcp-backlog 511
tcp-keepalive 300
timeout 300

# 4. Memory
maxmemory 8gb
maxmemory-policy allkeys-lru

# 5. Clients
maxclients 10000

# 6. Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128

# 7. Latency monitoring
latency-monitor-threshold 100
```

### Configuration système

```bash
# /etc/sysctl.conf

# Memory
vm.overcommit_memory=1
vm.swappiness=1

# Network
net.core.somaxconn=65535
net.ipv4.tcp_max_syn_backlog=65535
net.core.netdev_max_backlog=65535

# File descriptors
fs.file-max=100000
```

### Checklist de prévention

- [ ] THP désactivé
- [ ] Swap désactivé ou swappiness=1
- [ ] Overcommit_memory=1
- [ ] AOF en mode everysec
- [ ] Save frequency raisonnable
- [ ] Latency monitoring activé
- [ ] Alertes configurées
- [ ] Big keys éliminées
- [ ] Pipelining utilisé côté client
- [ ] Réseau optimisé (colocation si possible)

---

## 🎯 Points clés à retenir

1. **Latence normale < 1ms** → Au-delà, investiguer
2. **THP = ennemi #1** → Toujours désactiver
3. **Swap = catastrophe** → Désactiver ou réduire
4. **Fork peut bloquer** → Optimiser ou utiliser replica pour persist
5. **Réseau compte** → Pipelining et colocation
6. **Big keys = bloquant** → Fragmenter ou SCAN
7. **LATENCY DOCTOR** → Premier outil de diagnostic
8. **Monitoring proactif** → Détecter avant la crise

---

**🚀 Section suivante** : [14.5 - Out of Memory (OOM) : Diagnostic et résolution](./05-out-of-memory-diagnostic-resolution.md)

⏭️ [Out of Memory (OOM) : Diagnostic et résolution](/14-performance-troubleshooting/05-out-of-memory-diagnostic-resolution.md)
