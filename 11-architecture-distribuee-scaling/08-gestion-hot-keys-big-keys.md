🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.8 Gestion des Hot Keys et Big Keys dans un cluster

## Introduction

Les Hot Keys et Big Keys constituent deux catégories de problèmes de performance particulièrement critiques dans Redis Cluster. Alors qu'ils peuvent déjà causer des problèmes dans une instance Redis standalone, leur impact est amplifié dans un environnement distribué où ils peuvent créer des déséquilibres sévères, saturer des nœuds spécifiques, et compromettre la disponibilité globale du cluster.

Ces problèmes sont insidieux car ils peuvent apparaître progressivement, passer inaperçus dans les environnements de développement avec des charges limitées, puis causer des pannes catastrophiques en production lors de pics de trafic. Leur gestion nécessite une compréhension approfondie de l'architecture distribuée de Redis Cluster et une approche proactive de monitoring et d'optimisation.

Cette section explore en détail la nature de ces problèmes, leurs impacts spécifiques dans un cluster distribué, les méthodes de détection, et les stratégies d'atténuation et de résolution.

## Hot Keys : Définition et problématique

### Qu'est-ce qu'une Hot Key ?

```
┌─────────────────────────────────────────────────────────────┐
│                    Définition : Hot Key                     │
└─────────────────────────────────────────────────────────────┘

Hot Key = Clé accédée de manière disproportionnée par rapport
          aux autres clés du cluster

Caractéristiques :
══════════════════
• Taux d'accès très élevé (10x-1000x la moyenne)
• Concentration du trafic sur un seul slot/nœud
• Pattern typique : lecture intensive
• Peut être temporaire (trending topic) ou permanent (config globale)


Exemples typiques de Hot Keys :
════════════════════════════════

1. Configuration globale
   └─> "config:app:version" lue par chaque requête

2. Trending topic (réseau social)
   └─> "post:viral:12345" avec millions de vues

3. Compteur global
   └─> "stats:global:visitors" incrémenté en continu

4. Session d'utilisateur populaire
   └─> "session:celebrity:user" accédée massivement

5. Cache d'API externe
   └─> "cache:weather:paris" lu par tous les utilisateurs


Différence avec charge uniforme :
══════════════════════════════════

Charge UNIFORME (saine) :
─────────────────────────
    Node A      Node B      Node C
    1000 r/s    1000 r/s    1000 r/s
    ════════    ════════    ════════
    Équilibré ✓


Charge avec HOT KEY :
─────────────────────
    Node A      Node B      Node C
    10000 r/s   1000 r/s    1000 r/s
    ═════════   ════════    ════════
    Déséquilibré ✗

    Node A saturé par hot key sur ses slots
```

### Impact dans Redis Cluster

```
┌─────────────────────────────────────────────────────────────┐
│              Impact des Hot Keys en Cluster                 │
└─────────────────────────────────────────────────────────────┘

1. SATURATION D'UN NŒUD SPÉCIFIQUE
═══════════════════════════════════

Hot key "trending:post:viral" → Slot 8754 → Node B

    Client 1 ─┐
    Client 2 ─┤
    Client 3 ─┤
    Client 4 ─┼──> GET trending:post:viral ──> Node B ⚠️ SATURÉ
    Client 5 ─┤                                 │
    ...       ─┤                                 ├─ CPU: 95%
    Client N ─┘                                  ├─ Bandwidth: MAX
                                                 └─ Latency: 100ms+

    Node A: 10% CPU (idle)
    Node C: 10% CPU (idle)

Conséquence : Gaspillage de ressources (A, C sous-utilisés)


2. AUGMENTATION DE LA LATENCE
══════════════════════════════

Sans hot key :
    GET regular:key → 0.5ms (temps de réponse)

Avec hot key sur même nœud :
    GET regular:key → 50ms (×100 !)

Cause : File d'attente (queuing) des requêtes
        Node B traite d'abord les milliers de GET sur hot key


3. RISQUE DE TIMEOUTS
══════════════════════

Clients configurés avec timeout 1s
    │
    └─> Node B saturé, répond après 2s
        │
        └─> Client timeout ✗
            └─> Retry
                └─> Aggrave la charge (retry storm)


4. DÉSÉQUILIBRE DE LA CHARGE
═════════════════════════════

Distribution théorique : 33% / 33% / 33%
Distribution réelle avec hot key : 85% / 7.5% / 7.5%

Impossible d'ajouter plus de nœuds pour résoudre
(hot key restera sur le même nœud)


5. IMPACT SUR LE FAILOVER
══════════════════════════

Si Node B (avec hot key) tombe :
    │
    ├─> Failover vers replica
    ├─> Replica devient master
    └─> Replica AUSSI saturée par hot key ✗

Cycle vicieux : Master crash → Replica promue → Replica crash


6. EFFET CASCADE
════════════════

Node B saturé
    │
    ├─> Clients timeout et retry
    ├─> Augmentation connexions
    ├─> OOM sur Node B
    └─> Node B crash
        │
        └─> Failover vers replica
            └─> Replica crash (même problème)
                └─> Slot 8754 non disponible
                    └─> Application down ✗✗✗
```

### Pattern de Hot Keys par cas d'usage

```
┌─────────────────────────────────────────────────────────────┐
│              Patterns typiques de Hot Keys                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. CONFIGURATION GLOBALE (Permanent)                        │
│    ═══════════════════════════                              │
│    Pattern : config:global:*                                │
│    Accès : Lecture à chaque requête                         │
│    Volume : 10k-100k req/sec                                │
│    Solution : Réplication côté client, cache local          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 2. TRENDING CONTENT (Temporaire)                            │
│    ══════════════════════════                               │
│    Pattern : post:trending:*, video:viral:*                 │
│    Accès : Burst intense pendant quelques heures/jours      │
│    Volume : 100k-1M req/sec au pic                          │
│    Solution : CDN, réplication, éclatement                  │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 3. RATE LIMITING GLOBAL (Semi-permanent)                    │
│    ══════════════════════════                               │
│    Pattern : ratelimit:global:api                           │
│    Accès : Incrémentation à chaque appel API                │
│    Volume : Suit le trafic global                           │
│    Solution : Rate limiting local, agrégation différée      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 4. LOCK DISTRIBUÉ (Intermittent)                            │
│    ═══════════════════════                                  │
│    Pattern : lock:resource:shared                           │
│    Accès : Contention élevée sur ressource partagée         │
│    Volume : Pics lors de contention                         │
│    Solution : Redesign (éliminer bottleneck)                │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 5. COMPTEUR PARTAGÉ (Continu)                               │
│    ════════════════════════                                 │
│    Pattern : counter:global:views                           │
│    Accès : INCR continu                                     │
│    Volume : Proportionnel au trafic                         │
│    Solution : Sharding du compteur, agrégation batch        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Big Keys : Définition et problématique

### Qu'est-ce qu'une Big Key ?

```
┌─────────────────────────────────────────────────────────────┐
│                    Définition : Big Key                     │
└─────────────────────────────────────────────────────────────┘

Big Key = Clé dont la valeur consomme une quantité excessive
          de mémoire ou dont l'opération prend un temps significatif

Critères de taille :
════════════════════

Type            Seuil "Big"         Impact critique
────────────────────────────────────────────────────────
String          > 10 MB             > 100 MB
List            > 10,000 éléments   > 100,000 éléments
Set             > 10,000 membres    > 100,000 membres
Sorted Set      > 10,000 membres    > 100,000 membres
Hash            > 10,000 champs     > 100,000 champs

Note : Ces seuils sont indicatifs et dépendent du contexte


Exemples typiques de Big Keys :
════════════════════════════════

1. Liste de tous les utilisateurs
   └─> "users:all" LIST avec millions d'éléments

2. Session utilisateur surchargée
   └─> "session:user:123" HASH avec milliers de champs

3. Cache d'une page HTML complète
   └─> "cache:page:homepage" STRING de 50 MB

4. Historique complet d'événements
   └─> "events:user:123:history" LIST sans limite

5. Leaderboard géant
   └─> "leaderboard:global" ZSET avec millions de joueurs


Différence avec clés normales :
════════════════════════════════

Clé normale :
    GET user:1000:profile (1 KB)
    └─> Temps : 0.1ms
    └─> Réseau : négligeable
    └─> Mémoire : négligeable


Big Key :
    GET cache:homepage (50 MB)
    └─> Temps : 100ms (blocking)
    └─> Réseau : saturé (400 Mbps pour transférer)
    └─> Mémoire : 50 MB par réplication
```

### Impact dans Redis Cluster

```
┌─────────────────────────────────────────────────────────────┐
│              Impact des Big Keys en Cluster                 │
└─────────────────────────────────────────────────────────────┘

1. BLOCAGE DU THREAD PRINCIPAL
═══════════════════════════════

Redis est single-threaded (par nœud)

GET big_key (50 MB)
    │
    ├─> Sérialisation : 50ms
    ├─> Transfert réseau : 100ms
    └─> Total : 150ms de blocage ⚠️

Pendant ces 150ms :
    ├─ Aucune autre commande n'est traitée
    ├─ Toutes les requêtes vers ce nœud sont en attente
    └─> Timeouts généralisés sur les clients


2. SATURATION RÉSEAU
════════════════════

Transfert d'une big key de 100 MB :
    └─> Sur lien 1 Gbps = 800ms
    └─> Bande passante monopolisée

Impact sur autres clés du même nœud :
    └─> Latence réseau dégradée
    └─> Paquets perdus
    └─> Retransmissions TCP


3. PROBLÈMES DE RÉPLICATION
════════════════════════════

Big key sur Master A
    │
    ├─> Réplication vers Replica A1
    │   └─> Transfert de 100 MB
    │       ├─> Réseau saturé
    │       ├─> Replica en retard (lag)
    │       └─> Risque de resync complet
    │
    └─> Lors d'une écriture :
        ├─> Master envoie à replica
        ├─> Replica met 5s à appliquer (big SET/LIST/HASH)
        └─> Lag de 5s entre master et replica


4. MIGRATION LORS DU RESHARDING
════════════════════════════════

Resharding du slot contenant une big key :

MIGRATE host port key 0 timeout
    │
    ├─> Sérialisation de 500 MB
    ├─> Transfert réseau de 500 MB
    ├─> Timeout par défaut : 5s
    └─> Big key : 60s ✗
        └─> Migration échoue
            └─> Slot reste en MIGRATING
                └─> Cluster instable


5. IMPACT MÉMOIRE DISPROPORTIONNÉ
══════════════════════════════════

Cluster 3 nœuds, 16384 slots
Chaque nœud devrait avoir : ~33% de la mémoire

Avec une big key de 5 GB sur Node A :
    Node A : 8 GB (47% du total)  ← Déséquilibré
    Node B : 4.5 GB (26%)
    Node C : 4.5 GB (27%)

Conséquences :
    ├─> Node A proche de maxmemory
    ├─> Évictions précoces sur Node A
    └─> Autres nœuds sous-utilisés


6. PROBLÈMES DE PERSISTENCE
════════════════════════════

RDB (snapshot) :
    └─> Écriture de la big key bloque fork()
    └─> Copy-on-write penalty élevé

AOF (append only) :
    └─> Écriture de 100 MB dans l'AOF
    └─> fsync bloque pendant l'écriture
    └─> Latence spike généralisée


7. OPÉRATIONS DANGEREUSES
══════════════════════════

DEL big_key
    └─> Redis doit libérer toute la mémoire
    └─> Si big key = 1 GB
        └─> DEL bloque pendant 1-2 secondes ⚠️
        └─> Toutes les requêtes en attente

EXPIRE big_key 0
    └─> Même problème que DEL

Solution : UNLINK (non-blocking depuis Redis 4.0)
    └─> Marque pour suppression asynchrone
    └─> Thread background libère la mémoire
    └─> Pas de blocage ✓
```

### Types de structures et leur comportement

```
┌─────────────────────────────────────────────────────────────┐
│         Comportement des Big Keys par type                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ STRING (blob)                                               │
│ ═════════════                                               │
│ Problème principal : Transfert réseau                       │
│                                                             │
│ GET bigstring (100 MB)                                      │
│ └─> Temps O(1) mais transfert long                          │
│ └─> Monopolise bande passante                               │
│                                                             │
│ SET bigstring (100 MB)                                      │
│ └─> Allocation mémoire instantanée                          │
│ └─> Réplication coûteuse                                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ LIST                                                        │
│ ════                                                        │
│ Problème : Opérations sur tous les éléments                 │
│                                                             │
│ LRANGE biglist 0 -1  (1M éléments)                          │
│ └─> O(N) = très lent                                        │
│ └─> Bloque le serveur                                       │
│                                                             │
│ LPUSH biglist value  (ajout)                                │
│ └─> O(1) = rapide ✓                                         │
│                                                             │
│ DEL biglist                                                 │
│ └─> O(N) = libération de 1M éléments                        │
│ └─> Bloque pendant plusieurs secondes                       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ SET / HASH / SORTED SET                                     │
│ ═══════════════════════                                     │
│ Problème : Complexité O(N) sur opérations complètes         │
│                                                             │
│ SMEMBERS bigset  (1M membres)                               │
│ └─> O(N) = retourne tous les membres                        │
│ └─> Transfert de millions de valeurs                        │
│                                                             │
│ HGETALL bighash  (1M champs)                                │
│ └─> O(N) = retourne tous les champs                         │
│ └─> Sérialisation très coûteuse                             │
│                                                             │
│ ZRANGE bigsortedset 0 -1  (1M éléments)                     │
│ └─> O(log(N)+M) mais M=1M                                   │
│ └─> Très lent                                               │
│                                                             │
│ Alternative : Utiliser SCAN, HSCAN, SSCAN, ZSCAN            │
│ └─> Itération par curseur                                   │
│ └─> Batches de 100-1000 éléments                            │
│ └─> N'affecte pas les autres clients                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Détection et monitoring

### Détection des Hot Keys

```bash
# ═══════════════════════════════════════════════════════════
# MÉTHODES DE DÉTECTION DES HOT KEYS
# ═══════════════════════════════════════════════════════════


# MÉTHODE 1 : --hotkeys (Redis 4.0+)
# ───────────────────────────────────

# Analyse statistique avec sampling
redis-cli --hotkeys -h 192.168.1.10

# Output :
# Sampled 10000 keys in the keyspace
# hot key found with frequency: 8765 keyname: trending:post:viral
# hot key found with frequency: 5432 keyname: config:app:version
# hot key found with frequency: 3210 keyname: counter:global:hits

# Fonctionnement :
# ├─ Sample aléatoire de clés
# ├─ Mesure de la fréquence d'accès via LFU (Least Frequently Used)
# └─ Nécessite maxmemory-policy allkeys-lfu ou volatile-lfu

# Configuration requise dans redis.conf :
maxmemory-policy allkeys-lfu
lfu-log-factor 10
lfu-decay-time 1


# MÉTHODE 2 : MONITOR (temps réel, attention à l'impact)
# ───────────────────────────────────────────────────────

# MONITOR retourne TOUTES les commandes en temps réel
redis-cli -h 192.168.1.10 MONITOR | head -1000 > commands.log

# Analyser les logs
cat commands.log | grep -oP '"[^"]+"' | sort | uniq -c | sort -nr | head -20

# Output :
# 8765 "trending:post:viral"
# 5432 "config:app:version"
# 3210 "counter:global:hits"

# ⚠️ ATTENTION : MONITOR est TRÈS coûteux
# ├─ Duplique toutes les commandes vers le client
# ├─ Impacte significativement les performances
# └─ À utiliser UNIQUEMENT en debug court terme (<1 minute)


# MÉTHODE 3 : Analyse des logs avec pattern matching
# ───────────────────────────────────────────────────

# Activer slowlog pour capturer commandes lentes
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms

# Récupérer slowlog
redis-cli SLOWLOG GET 100

# Analyser les clés fréquentes dans slowlog
redis-cli SLOWLOG GET 1000 | grep '"GET"' | grep -oP 'key:[^ ]+' | sort | uniq -c | sort -nr


# MÉTHODE 4 : Client-side instrumentation
# ────────────────────────────────────────

# Instrumenter l'application pour tracker les clés accédées
# Exemple avec Python :

from collections import Counter
import redis

class InstrumentedRedis:
    def __init__(self, *args, **kwargs):
        self.client = redis.Redis(*args, **kwargs)
        self.key_counter = Counter()

    def get(self, key):
        self.key_counter[key] += 1
        return self.client.get(key)

    def get_hot_keys(self, top_n=10):
        return self.key_counter.most_common(top_n)

# Utilisation
r = InstrumentedRedis(host='localhost')
# ... utiliser r.get() dans l'application ...

# Périodiquement, logger les hot keys
print(r.get_hot_keys(10))


# MÉTHODE 5 : Monitoring externe (Prometheus + Exporter)
# ───────────────────────────────────────────────────────

# Utiliser redis_exporter avec key sampling
# https://github.com/oliver006/redis_exporter

# redis_exporter avec option -check-keys
redis_exporter \
    -redis.addr redis://192.168.1.10:6379 \
    -check-keys "trending:*,config:*,counter:*"

# Métriques exportées :
# redis_key_size{key="trending:post:viral"} 1234
# redis_key_access_count{key="trending:post:viral"} 89765

# Alertes Prometheus pour hot keys :
# Règle d'alerte si une clé dépasse 10k req/min
groups:
  - name: redis_hotkeys
    rules:
      - alert: HotKeyDetected
        expr: rate(redis_key_access_count[1m]) > 10000
        annotations:
          summary: "Hot key detected: {{ $labels.key }}"


# MÉTHODE 6 : Script Lua pour échantillonnage
# ────────────────────────────────────────────

# Script qui sample les clés et retourne statistiques
local cursor = "0"
local keys_sampled = {}
local sample_size = 10000

repeat
    local result = redis.call("SCAN", cursor, "COUNT", 100)
    cursor = result[1]
    local keys = result[2]

    for _, key in ipairs(keys) do
        local freq = redis.call("OBJECT", "FREQ", key)
        if freq then
            table.insert(keys_sampled, {key, freq})
        end
    end
until cursor == "0" or #keys_sampled >= sample_size

-- Trier par fréquence
table.sort(keys_sampled, function(a, b) return a[2] > b[2] end)

-- Retourner top 10
local result = {}
for i = 1, math.min(10, #keys_sampled) do
    table.insert(result, keys_sampled[i])
end

return result
```

### Détection des Big Keys

```bash
# ═══════════════════════════════════════════════════════════
# MÉTHODES DE DÉTECTION DES BIG KEYS
# ═══════════════════════════════════════════════════════════


# MÉTHODE 1 : --bigkeys (natif Redis)
# ────────────────────────────────────

# Scan toutes les clés et identifie les plus grandes par type
redis-cli --bigkeys -h 192.168.1.10

# Output :
# -------- summary -------
#
# Biggest string found: cache:homepage (50.2 MB)
# Biggest list found: events:user:123:history (1,234,567 items)
# Biggest set found: users:all (500,000 members)
# Biggest hash found: session:user:456 (100,000 fields)
# Biggest zset found: leaderboard:global (2,000,000 members)
#
# 12345678 keys scanned
# 150 big keys found

# Options utiles :
redis-cli --bigkeys -i 0.01  # Pause 10ms entre clés (moins impactant)


# MÉTHODE 2 : MEMORY USAGE (Redis 4.0+)
# ──────────────────────────────────────

# Obtenir taille exacte d'une clé spécifique
redis-cli MEMORY USAGE cache:homepage
# (integer) 52428800  # 50 MB

redis-cli MEMORY USAGE events:user:123:history
# (integer) 10485760  # 10 MB

# Script pour scanner toutes les clés et mesurer taille
redis-cli --scan | while read key; do
    size=$(redis-cli MEMORY USAGE "$key")
    echo "$size $key"
done | sort -rn | head -20

# Top 20 biggest keys avec leur taille


# MÉTHODE 3 : Script d'analyse complet
# ─────────────────────────────────────

#!/bin/bash
# find-big-keys.sh
# Analyse complète des big keys avec détails

REDIS_HOST="192.168.1.10"
REDIS_PORT="6379"
THRESHOLD_MB=10

echo "Scanning for big keys (threshold: ${THRESHOLD_MB} MB)..."

redis-cli -h $REDIS_HOST -p $REDIS_PORT --scan | while read key; do
    # Obtenir le type
    type=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT TYPE "$key")

    # Obtenir la taille en mémoire
    size=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT MEMORY USAGE "$key")
    size_mb=$(awk "BEGIN {print $size/1024/1024}")

    # Si > threshold, afficher détails
    if (( $(echo "$size_mb > $THRESHOLD_MB" | bc -l) )); then

        # Détails selon le type
        case $type in
            "string")
                strlen=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT STRLEN "$key")
                echo "BIG KEY: $key | Type: $type | Size: ${size_mb} MB | Length: $strlen bytes"
                ;;
            "list")
                llen=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT LLEN "$key")
                echo "BIG KEY: $key | Type: $type | Size: ${size_mb} MB | Elements: $llen"
                ;;
            "set")
                scard=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT SCARD "$key")
                echo "BIG KEY: $key | Type: $type | Size: ${size_mb} MB | Members: $scard"
                ;;
            "zset")
                zcard=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT ZCARD "$key")
                echo "BIG KEY: $key | Type: $type | Size: ${size_mb} MB | Members: $zcard"
                ;;
            "hash")
                hlen=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT HLEN "$key")
                echo "BIG KEY: $key | Type: $type | Size: ${size_mb} MB | Fields: $hlen"
                ;;
        esac
    fi
done


# MÉTHODE 4 : Analyse via RDB Tools
# ──────────────────────────────────

# Installer rdb-tools (Python)
pip install rdbtools python-lzf

# Analyser un dump RDB
rdb --command memory dump.rdb > memory-report.csv

# Format CSV :
# database,type,key,size_in_bytes,encoding,num_elements,len_largest_element

# Filtrer les big keys (> 10 MB)
awk -F',' '$4 > 10485760 {print $3,$4/1024/1024 " MB"}' memory-report.csv | sort -k2 -rn

# Générer rapport HTML
rdb --command memory dump.rdb --bytes 10485760 --largest 100 > big-keys.html


# MÉTHODE 5 : Monitoring continu (Python script)
# ───────────────────────────────────────────────

import redis
import time
from collections import defaultdict

BIG_KEY_THRESHOLD_MB = 10
SCAN_INTERVAL_SECONDS = 300  # Toutes les 5 minutes

def scan_big_keys(host, port):
    r = redis.Redis(host=host, port=port, decode_responses=True)
    big_keys = []

    cursor = '0'
    while True:
        cursor, keys = r.scan(cursor=cursor, count=100)

        for key in keys:
            try:
                # Obtenir taille en bytes
                size_bytes = r.memory_usage(key)

                if size_bytes and size_bytes > (BIG_KEY_THRESHOLD_MB * 1024 * 1024):
                    key_type = r.type(key)
                    size_mb = size_bytes / 1024 / 1024

                    big_keys.append({
                        'key': key,
                        'type': key_type,
                        'size_mb': round(size_mb, 2)
                    })
            except Exception as e:
                print(f"Error checking key {key}: {e}")

        if cursor == '0':
            break

    return big_keys

def monitor_big_keys():
    while True:
        print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Scanning for big keys...")

        big_keys = scan_big_keys('192.168.1.10', 6379)

        if big_keys:
            print(f"Found {len(big_keys)} big keys:")
            for bk in sorted(big_keys, key=lambda x: x['size_mb'], reverse=True):
                print(f"  - {bk['key']} ({bk['type']}): {bk['size_mb']} MB")

            # Envoyer alerte si nécessaire
            # send_alert(big_keys)
        else:
            print("No big keys found")

        time.sleep(SCAN_INTERVAL_SECONDS)

if __name__ == "__main__":
    monitor_big_keys()


# MÉTHODE 6 : INFO MEMORY pour vue globale
# ─────────────────────────────────────────

redis-cli INFO MEMORY

# Métriques importantes :
# used_memory_human:15.23G
# used_memory_peak_human:18.45G
# mem_fragmentation_ratio:1.15

# Si ratio > 1.5 : fragmentation importante
# Peut indiquer présence de big keys supprimées/modifiées


# MÉTHODE 7 : DEBUG OBJECT (détails d'une clé)
# ─────────────────────────────────────────────

redis-cli DEBUG OBJECT cache:homepage

# Output :
# Value at:0x7f8a9c000000 refcount:1 encoding:raw serializedlength:52428800 lru:12345678 lru_seconds_idle:42

# serializedlength = taille sérialisée (approximative de la mémoire)
```

### Monitoring en continu

```yaml
# ═══════════════════════════════════════════════════════════
# CONFIGURATION PROMETHEUS + GRAFANA POUR HOT/BIG KEYS
# ═══════════════════════════════════════════════════════════

# prometheus.yml
# ──────────────

scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets:
          - '192.168.1.10:9121'  # redis_exporter
          - '192.168.1.11:9121'
          - '192.168.1.12:9121'

# Alertes Prometheus
# ──────────────────

# prometheus-alerts.yml

groups:
  - name: redis_keys
    rules:
      # Alerte : Hot Key détectée
      - alert: RedisHotKeyDetected
        expr: |
          rate(redis_command_calls_total{cmd="get"}[1m])
          / on(instance)
          redis_commands_processed_total > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Potential hot key on {{ $labels.instance }}"
          description: "GET commands represent >50% of traffic"

      # Alerte : Big Key détectée
      - alert: RedisBigKeyDetected
        expr: redis_key_size > 10485760  # 10 MB
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Big key detected: {{ $labels.key }}"
          description: "Key size: {{ $value | humanize }}B"

      # Alerte : Déséquilibre de charge
      - alert: RedisNodeImbalance
        expr: |
          max(rate(redis_commands_processed_total[5m]))
          / min(rate(redis_commands_processed_total[5m])) > 3
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis cluster load imbalance"
          description: "One node handling 3x more traffic than others"


# Dashboard Grafana
# ─────────────────

# grafana-dashboard.json (extrait)
{
  "panels": [
    {
      "title": "Commands per Second by Node",
      "targets": [
        {
          "expr": "rate(redis_commands_processed_total[1m])"
        }
      ]
    },
    {
      "title": "Top Keys by Size",
      "targets": [
        {
          "expr": "topk(10, redis_key_size)"
        }
      ]
    },
    {
      "title": "Memory Usage per Node",
      "targets": [
        {
          "expr": "redis_memory_used_bytes"
        }
      ]
    },
    {
      "title": "Network I/O per Node",
      "targets": [
        {
          "expr": "rate(redis_net_input_bytes_total[1m])"
        }
      ]
    }
  ]
}
```

## Solutions et stratégies d'atténuation

### Solutions pour Hot Keys

```
┌─────────────────────────────────────────────────────────────┐
│              Stratégies pour atténuer Hot Keys              │
└─────────────────────────────────────────────────────────────┘


STRATÉGIE 1 : RÉPLICATION DE LA HOT KEY
════════════════════════════════════════

Principe : Dupliquer la hot key sur plusieurs slots


Avant :
    trending:post:viral → Slot 8754 → Node B (saturé)


Après :
    trending:post:viral:1 → Slot 1234 → Node A
    trending:post:viral:2 → Slot 5678 → Node B
    trending:post:viral:3 → Slot 9012 → Node C

Client choisit aléatoirement : trending:post:viral:{1,2,3}


Implémentation (Python) :
─────────────────────────

import random
import redis

def get_hot_key_replicated(key, num_replicas=3):
    """Lire hot key avec load balancing"""
    replica_id = random.randint(1, num_replicas)
    replicated_key = f"{key}:{replica_id}"
    return redis_client.get(replicated_key)

def set_hot_key_replicated(key, value, num_replicas=3):
    """Écrire hot key sur toutes les replicas"""
    for i in range(1, num_replicas + 1):
        replicated_key = f"{key}:{i}"
        redis_client.set(replicated_key, value)

# Utilisation
set_hot_key_replicated("trending:post:viral", post_data, num_replicas=5)
data = get_hot_key_replicated("trending:post:viral", num_replicas=5)


Avantages :
├─ Distribution de la charge sur plusieurs nœuds
├─ Simple à implémenter
└─ Transparent pour la logique métier

Inconvénients :
├─ Cohérence éventuelle entre replicas
├─ Consommation mémoire × N
└─ Complexité de mise à jour (écrire sur toutes les replicas)


STRATÉGIE 2 : CACHE LOCAL CÔTÉ CLIENT
══════════════════════════════════════

Principe : Cache en mémoire de l'application


Application
    │
    ├─> Cache local (in-memory)
    │   ├─ config:app:version → cached 30s
    │   └─ trending:post:viral → cached 10s
    │
    └─> Redis (fallback si cache miss)


Implémentation (Python avec cachetools) :
─────────────────────────────────────────

from cachetools import TTLCache
import redis

# Cache local avec TTL
local_cache = TTLCache(maxsize=1000, ttl=30)  # 30 secondes

def get_with_local_cache(key):
    """GET avec cache local"""

    # Vérifier cache local
    if key in local_cache:
        print(f"Cache HIT (local): {key}")
        return local_cache[key]

    # Cache miss → Redis
    value = redis_client.get(key)

    # Stocker dans cache local
    if value:
        local_cache[key] = value
        print(f"Cache MISS (Redis): {key}")

    return value

# Utilisation
config = get_with_local_cache("config:app:version")


Avantages :
├─ Latence ultra-faible (in-memory local)
├─ Réduit drastiquement charge sur Redis
├─ Pas de modification de Redis
└─ Scalabilité parfaite

Inconvénients :
├─ Staleness (données peuvent être périmées pendant TTL)
├─ Consommation mémoire sur chaque instance d'application
└─ Cohérence éventuelle


STRATÉGIE 3 : LECTURE SUR REPLICAS
═══════════════════════════════════

Principe : Rediriger les lectures vers replicas (Redis 7+)


Configuration :
──────────────

# redis.conf sur chaque nœud
replica-read-only yes

# Client configure route de lecture
# (certaines bibliothèques le supportent nativement)


Load balancing manuel :
───────────────────────

# Python exemple
from rediscluster import RedisCluster

startup_nodes = [
    {"host": "192.168.1.10", "port": 6379},  # Master A
    {"host": "192.168.1.13", "port": 6379},  # Replica A
    {"host": "192.168.1.11", "port": 6379},  # Master B
    {"host": "192.168.1.14", "port": 6379},  # Replica B
]

# Configurer pour lire sur replicas
cluster = RedisCluster(
    startup_nodes=startup_nodes,
    decode_responses=True,
    read_from_replicas=True  # Active lectures sur replicas
)

value = cluster.get("trending:post:viral")
# Peut être routé vers replica automatiquement


Avantages :
├─ Offload des masters
├─ Double la capacité de lecture
└─ Pas de modification de code métier

Inconvénients :
├─ Cohérence éventuelle (replica lag)
├─ Complexité de configuration client
└─ Pas adapté si données doivent être à jour


STRATÉGIE 4 : SHARDING DE LA HOT KEY
═════════════════════════════════════

Principe : Décomposer une hot key en multiples sous-clés


Exemple : Compteur global
─────────────────────────

Avant :
    counter:global:views → INCR 100k/sec → Node A saturé


Après :
    counter:global:views:shard:0 → Node A
    counter:global:views:shard:1 → Node B
    counter:global:views:shard:2 → Node C
    ...
    counter:global:views:shard:9 → Node C


Implémentation :
───────────────

import hashlib

NUM_SHARDS = 10

def incr_sharded_counter(base_key, client_id):
    """Incrémenter compteur shardé"""
    # Déterminer shard basé sur client_id
    shard = int(hashlib.md5(client_id.encode()).hexdigest(), 16) % NUM_SHARDS
    sharded_key = f"{base_key}:shard:{shard}"

    return redis_client.incr(sharded_key)

def get_total_counter(base_key):
    """Obtenir total en agrégeant tous les shards"""
    total = 0
    for shard in range(NUM_SHARDS):
        sharded_key = f"{base_key}:shard:{shard}"
        value = redis_client.get(sharded_key)
        total += int(value) if value else 0
    return total

# Utilisation
incr_sharded_counter("counter:global:views", "user123")
incr_sharded_counter("counter:global:views", "user456")

total = get_total_counter("counter:global:views")
print(f"Total views: {total}")


Avantages :
├─ Distribution parfaite de la charge
├─ Scalabilité linéaire
└─ Chaque shard sur nœud différent

Inconvénients :
├─ Obtenir total nécessite N lectures
├─ Complexité accrue
└─ Pas adapté à tous les types de données


STRATÉGIE 5 : CDN / EDGE CACHING
═════════════════════════════════

Pour contenu web statique ou semi-statique


    Clients
       │
       ▼
    CDN (Cloudflare, Fastly)
       │
       ├─> Cache hit (99%) → Retourne directement
       │
       └─> Cache miss (1%) → Redis Cluster
                              └─> Mise en cache CDN


Avantages :
├─ Réduit charge Redis de 99%
├─ Latence ultra-faible (POP CDN proche du client)
├─ Absorption pics de trafic
└─ Protection DDoS

Inconvénients :
├─ Coût supplémentaire (CDN)
├─ Staleness (cache CDN peut être périmé)
└─ Uniquement pour données publiques
```

### Solutions pour Big Keys

```
┌─────────────────────────────────────────────────────────────┐
│              Stratégies pour atténuer Big Keys              │
└─────────────────────────────────────────────────────────────┘


STRATÉGIE 1 : FRAGMENTATION DE LA BIG KEY
══════════════════════════════════════════

Principe : Diviser une grande structure en plusieurs petites


Exemple : Liste de tous les utilisateurs
─────────────────────────────────────────

Avant :
    users:all LIST [user1, user2, ..., user1000000]  # 1M éléments


Après :
    users:batch:0 LIST [user1, ..., user1000]        # 1k éléments
    users:batch:1 LIST [user1001, ..., user2000]
    ...
    users:batch:999 LIST [user999001, ..., user1000000]


Implémentation :
───────────────

BATCH_SIZE = 1000

def add_user_fragmented(user_id):
    """Ajouter user dans batch approprié"""
    batch_id = user_id // BATCH_SIZE
    batch_key = f"users:batch:{batch_id}"
    redis_client.rpush(batch_key, user_id)

def get_all_users_fragmented():
    """Récupérer tous les users en itérant sur batches"""
    all_users = []
    batch_id = 0

    while True:
        batch_key = f"users:batch:{batch_id}"
        users = redis_client.lrange(batch_key, 0, -1)

        if not users:
            break

        all_users.extend(users)
        batch_id += 1

    return all_users


Exemple : Session utilisateur avec hash gigantesque
────────────────────────────────────────────────────

Avant :
    session:user:123 HASH
    ├─ field1: value1
    ├─ field2: value2
    ...
    └─ field100000: value100000  # 100k champs


Après :
    session:user:123:profile HASH (10 champs)
    session:user:123:settings HASH (50 champs)
    session:user:123:history HASH (dynamic, paginated)


Avantages :
├─ Opérations individuelles plus rapides
├─ Flexibilité (expiration différente par fragment)
└─ Limite impact d'une opération

Inconvénients :
├─ Complexité accrue
├─ Requêtes multiples si besoin de tout
└─ Fragmentation à gérer


STRATÉGIE 2 : PAGINATION ET CURSEURS
═════════════════════════════════════

Utiliser SCAN, HSCAN, SSCAN, ZSCAN au lieu de KEYS, HGETALL, etc.


Mauvais :
────────

# Retourne TOUS les membres (bloquant)
all_members = redis_client.smembers("users:all")  # 1M membres, bloque 5s


Bon :
────

# Itération par curseur (non-bloquant)
cursor = 0
members = []

while True:
    cursor, batch = redis_client.sscan("users:all", cursor, count=1000)
    members.extend(batch)

    if cursor == 0:
        break


Encore mieux : Générateur Python
─────────────────────────────────

def scan_set_members(key, batch_size=1000):
    """Générateur qui yield members par batch"""
    cursor = 0

    while True:
        cursor, batch = redis_client.sscan(key, cursor, count=batch_size)

        for member in batch:
            yield member

        if cursor == 0:
            break

# Utilisation
for member in scan_set_members("users:all"):
    process(member)  # Traiter un par un, pas de blocage


Avantages :
├─ Pas de blocage du serveur
├─ Mémoire constante côté client
└─ Autres clients non impactés

Inconvénients :
├─ Plus lent si besoin de tout
└─ Pas de snapshot atomique


STRATÉGIE 3 : COMPRESSION DES DONNÉES
══════════════════════════════════════

Compresser les grandes valeurs avant stockage


import gzip
import redis

def set_compressed(key, value):
    """Stocker valeur compressée"""
    compressed = gzip.compress(value.encode())
    redis_client.set(key, compressed)

def get_compressed(key):
    """Récupérer et décompresser"""
    compressed = redis_client.get(key)
    if compressed:
        return gzip.decompress(compressed).decode()
    return None

# Exemple
large_html = "<html>..." * 1000  # 1 MB
set_compressed("cache:page:home", large_html)

# Stocké comme ~100 KB (ratio 10:1)


Avantages :
├─ Réduction mémoire significative (50-90%)
├─ Réduction bande passante réseau
└─ Plus de clés dans la mémoire disponible

Inconvénients :
├─ CPU pour compression/décompression
├─ Latence légèrement accrue
└─ Pas adapté à toutes les données


STRATÉGIE 4 : EXTERNALISATION
══════════════════════════════

Stocker grandes valeurs en dehors de Redis


Avant :
    cache:video:123 STRING [50 MB de données binaires]


Après :
    cache:video:123 STRING "s3://bucket/videos/123.mp4"
    └─> Pointeur vers S3


Implémentation :
───────────────

import boto3
import redis

s3 = boto3.client('s3')
BUCKET = 'video-cache'

def set_large_value(key, value, threshold_mb=10):
    """Stocker en Redis si petit, S3 si gros"""
    size_mb = len(value) / 1024 / 1024

    if size_mb > threshold_mb:
        # Uploader vers S3
        s3_key = f"redis-overflow/{key}"
        s3.put_object(Bucket=BUCKET, Key=s3_key, Body=value)

        # Stocker pointeur dans Redis
        pointer = f"s3://{BUCKET}/{s3_key}"
        redis_client.set(key, pointer)
    else:
        # Stocker directement dans Redis
        redis_client.set(key, value)

def get_large_value(key):
    """Récupérer de Redis ou S3"""
    value = redis_client.get(key)

    if value.startswith(b's3://'):
        # C'est un pointeur S3
        bucket, s3_key = value.decode().replace('s3://', '').split('/', 1)
        response = s3.get_object(Bucket=bucket, Key=s3_key)
        return response['Body'].read()
    else:
        # Valeur directe
        return value


Avantages :
├─ Redis garde taille raisonnable
├─ S3 optimisé pour gros objets
├─ Coût de stockage réduit (S3 moins cher)
└─ Scalabilité illimitée

Inconvénients :
├─ Latence accrue (appel S3)
├─ Complexité
├─ Coût transfert S3
└─ Dépendance externe


STRATÉGIE 5 : LAZY DELETION (UNLINK)
═════════════════════════════════════

Utiliser UNLINK au lieu de DEL pour big keys


# Mauvais (bloque le serveur)
redis_client.delete("big_list")  # Peut bloquer 5s


# Bon (asynchrone)
redis_client.unlink("big_list")  # Retourne immédiatement


Redis 4.0+ : Background thread libère la mémoire


Implémentation dans application :
─────────────────────────────────

def safe_delete_key(key):
    """Supprimer clé de manière sûre"""
    # Vérifier taille
    size_mb = redis_client.memory_usage(key) / 1024 / 1024

    if size_mb > 10:
        # Big key → UNLINK
        print(f"Using UNLINK for big key {key} ({size_mb} MB)")
        return redis_client.unlink(key)
    else:
        # Small key → DEL standard
        return redis_client.delete(key)


Avantages :
├─ Pas de blocage
├─ Simple à utiliser
└─ Natif Redis (pas de dépendance)

Limitations :
├─ Redis 4.0+ uniquement
└─ N'empêche pas création de big keys


STRATÉGIE 6 : TTL ET EXPIRATION
════════════════════════════════

Limiter la croissance avec expiration automatique


# Mauvais : Liste sans limite
redis_client.rpush("events:user:123", event)
# Liste grandit indéfiniment


# Bon : LTRIM + TTL
redis_client.rpush("events:user:123", event)
redis_client.ltrim("events:user:123", -1000, -1)  # Garder 1000 derniers
redis_client.expire("events:user:123", 86400)  # Expire après 24h


# Alternative : ZADD avec score timestamp + ZREMRANGEBYSCORE
now = time.time()
redis_client.zadd("events:user:123", {event: now})

# Supprimer events > 7 jours
week_ago = now - 7 * 86400
redis_client.zremrangebyscore("events:user:123", 0, week_ago)


Avantages :
├─ Limite automatique de taille
├─ Pas de maintenance manuelle
└─ Prévient croissance infinie

Inconvénients :
├─ Perte de données anciennes
└─ Logique de trim à implémenter
```

## Procédures de résolution en production

### Procédure d'urgence : Hot Key critique

```bash
# ═══════════════════════════════════════════════════════════
# PROCÉDURE D'URGENCE : HOT KEY SATURANT UN NŒUD
# ═══════════════════════════════════════════════════════════

# SYMPTÔMES
# ─────────
# • Node B à 95% CPU, autres nœuds à 10%
# • Latence élevée sur certaines requêtes (100ms+)
# • Timeouts clients
# • Logs montrent GET répété sur même clé


# ÉTAPE 1 : IDENTIFIER LA HOT KEY (5 minutes)
# ────────────────────────────────────────────

# Activer monitoring sur le nœud saturé
redis-cli -h 192.168.1.11 --hotkeys

# Ou MONITOR (très court, <30 secondes)
redis-cli -h 192.168.1.11 MONITOR | head -1000 > /tmp/commands.log
grep -oP '"[^"]+"' /tmp/commands.log | sort | uniq -c | sort -nr | head -5

# Supposons identification : trending:post:viral


# ÉTAPE 2 : MITIGATION IMMÉDIATE (2 minutes)
# ───────────────────────────────────────────

# Option A : Activer cache local côté application
# Déployer configuration d'urgence avec TTL court

# Exemple config (injecter via feature flag)
ENABLE_LOCAL_CACHE=true
LOCAL_CACHE_TTL_SECONDS=10
LOCAL_CACHE_KEYS="trending:post:viral"

# Option B : Répliquer la hot key manuellement

# Obtenir valeur
VALUE=$(redis-cli -h 192.168.1.11 GET trending:post:viral)

# Créer replicas sur autres nœuds (avec hash tags pour forcer slots)
redis-cli -h 192.168.1.10 SET "trending:post:viral:1" "$VALUE"
redis-cli -h 192.168.1.11 SET "trending:post:viral:2" "$VALUE"
redis-cli -h 192.168.1.12 SET "trending:post:viral:3" "$VALUE"

# Mettre à jour application pour load balance
# (nécessite déploiement rapide ou feature flag)


# ÉTAPE 3 : VÉRIFICATION (1 minute)
# ──────────────────────────────────

# Surveiller charge CPU du nœud
watch -n 1 'redis-cli -h 192.168.1.11 INFO CPU | grep used_cpu_sys'

# Surveiller latence
redis-cli -h 192.168.1.11 --latency-history

# Après mitigation, charge devrait diminuer


# ÉTAPE 4 : SOLUTION PERMANENTE (post-incident)
# ──────────────────────────────────────────────

# 1. Analyser pourquoi cette clé est devenue hot
#    - Événement externe (trending topic) ?
#    - Bug applicatif (polling agressif) ?
#    - Attaque DDoS ?

# 2. Implémenter solution appropriée
#    - Cache local avec TTL
#    - CDN pour contenu public
#    - Rate limiting côté application
#    - Réplication permanente

# 3. Ajouter monitoring proactif
#    - Alertes sur déséquilibre de charge
#    - Dashboard visualisant top keys par trafic

# 4. Post-mortem
#    - Documenter incident
#    - Partager learnings
#    - Améliorer runbook


# ÉTAPE 5 : COMMUNICATION (durant incident)
# ──────────────────────────────────────────

# Template de communication
cat <<EOF
INCIDENT: Hot key causing Redis node saturation

Status: INVESTIGATING / MITIGATING / RESOLVED
Start: $(date)
Impact: Increased latency on requests to Node B

Actions taken:
- [12:00] Hot key identified: trending:post:viral
- [12:03] Local cache enabled on application servers
- [12:05] Load balanced, Node B CPU back to normal

Next steps:
- Monitor for recurrence
- Implement permanent solution
- Schedule post-mortem

Updates: Every 15 minutes or when status changes
EOF
```

### Procédure d'urgence : Big Key bloquant

```bash
# ═══════════════════════════════════════════════════════════
# PROCÉDURE D'URGENCE : BIG KEY BLOQUANT LE SERVEUR
# ═══════════════════════════════════════════════════════════

# SYMPTÔMES
# ─────────
# • Timeouts généralisés sur un nœud
# • Slowlog montre commandes prenant 1s+
# • Network bandwidth saturé périodiquement
# • Une commande GET/HGETALL/LRANGE prend énormément de temps


# ÉTAPE 1 : IDENTIFIER LA BIG KEY (3 minutes)
# ────────────────────────────────────────────

# Vérifier slowlog
redis-cli -h 192.168.1.10 SLOWLOG GET 10

# Example output :
# 1) "GET cache:homepage 12345678"  # 5000000 microseconds (5 seconds)

# Confirmer que c'est une big key
redis-cli -h 192.168.1.10 MEMORY USAGE cache:homepage
# (integer) 104857600  # 100 MB !


# ÉTAPE 2 : ÉVALUER L'IMPACT (1 minute)
# ──────────────────────────────────────

# Fréquence d'accès
redis-cli -h 192.168.1.10 OBJECT FREQ cache:homepage
# Élevé = problème critique

# Vérifier si d'autres big keys
redis-cli -h 192.168.1.10 --bigkeys -i 0.1

# Type de la clé
TYPE=$(redis-cli -h 192.168.1.10 TYPE cache:homepage)
echo "Type: $TYPE"


# ÉTAPE 3 : MITIGATION IMMÉDIATE
# ───────────────────────────────

# Option A : Si la clé peut être supprimée (cache invalide)
# ──────────────────────────────────────────────────────────

# ATTENTION : NE PAS utiliser DEL (va bloquer encore plus)
# Utiliser UNLINK (asynchrone)
redis-cli -h 192.168.1.10 UNLINK cache:homepage

# Vérifier suppression
redis-cli -h 192.168.1.10 EXISTS cache:homepage
# (integer) 0  ✓


# Option B : Si la clé est importante (ne peut pas être supprimée)
# ─────────────────────────────────────────────────────────────────

# Temporairement, désactiver l'application qui accède à cette clé
# Ou implémenter circuit breaker pour cette clé spécifique

# Config d'urgence dans l'application :
BLOCKED_KEYS="cache:homepage"
# Application skip l'accès à cette clé et utilise fallback


# Option C : Migration vers stockage externe
# ───────────────────────────────────────────

# Si temps le permet, migrer vers S3/blob storage
aws s3 cp /tmp/homepage.html s3://cache-bucket/homepage.html

# Remplacer dans Redis par un pointeur
redis-cli -h 192.168.1.10 SET cache:homepage:pointer "s3://cache-bucket/homepage.html"

# Mettre à jour application pour fetch depuis S3


# ÉTAPE 4 : SOLUTION PERMANENTE
# ──────────────────────────────

# Selon le type de big key :

# Pour STRING (blob) :
# ├─ Compression (gzip)
# ├─ CDN pour contenu statique
# └─ S3 pour très gros objets

# Pour LIST/SET/HASH :
# ├─ Fragmentation (batching)
# ├─ Pagination avec SCAN/HSCAN
# └─ TTL + LTRIM pour limiter croissance

# Pour ZSET (leaderboard) :
# ├─ ZREMRANGEBYRANK pour limiter taille
# └─ Fragmentation temporelle (daily/weekly boards)


# ÉTAPE 5 : PRÉVENTION FUTURE
# ────────────────────────────

# 1. Monitoring proactif
cat > /usr/local/bin/check-big-keys.sh <<'EOF'
#!/bin/bash
THRESHOLD_MB=10

redis-cli --bigkeys -i 0.01 | grep -E "string|list|set|hash|zset" | while read line; do
    key=$(echo "$line" | awk '{print $4}')
    size=$(redis-cli MEMORY USAGE "$key")
    size_mb=$(awk "BEGIN {print $size/1024/1024}")

    if (( $(echo "$size_mb > $THRESHOLD_MB" | bc -l) )); then
        echo "ALERT: Big key detected: $key ($size_mb MB)"
        # Envoyer alerte
    fi
done
EOF

chmod +x /usr/local/bin/check-big-keys.sh

# Cron toutes les 5 minutes
echo "*/5 * * * * /usr/local/bin/check-big-keys.sh" | crontab -


# 2. Limites au niveau application
# Rejeter écritures qui créeraient big keys

def safe_set(key, value, max_size_mb=10):
    size_mb = len(value) / 1024 / 1024

    if size_mb > max_size_mb:
        raise ValueError(f"Value too large: {size_mb} MB > {max_size_mb} MB limit")

    return redis_client.set(key, value)


# 3. Documentation et training
# Former l'équipe sur :
# ├─ Dangers des big keys
# ├─ Utilisation de SCAN vs KEYS
# ├─ UNLINK vs DEL
# └─ Patterns de fragmentation
```

### Migration d'une big key vers fragmentation

```python
# ═══════════════════════════════════════════════════════════
# MIGRATION : BIG KEY → FRAGMENTATION (ZERO DOWNTIME)
# ═══════════════════════════════════════════════════════════

import redis
import time

# Configuration
REDIS_HOST = "192.168.1.10"
OLD_KEY = "users:all"  # Big LIST avec 1M éléments
NEW_KEY_PREFIX = "users:batch"
BATCH_SIZE = 1000

r = redis.Redis(host=REDIS_HOST, decode_responses=True)


def migrate_big_list_to_batches():
    """
    Migrer une grande liste vers plusieurs petites listes
    Sans bloquer le serveur
    """

    print("=== Migration : Big List → Batched Lists ===")

    # ÉTAPE 1 : Vérifier la taille
    # ─────────────────────────────
    total_elements = r.llen(OLD_KEY)
    print(f"Total elements in {OLD_KEY}: {total_elements}")

    if total_elements == 0:
        print("List is empty, nothing to migrate")
        return

    # ÉTAPE 2 : Créer les batches progressivement
    # ────────────────────────────────────────────
    batch_id = 0
    migrated_count = 0

    while True:
        # Lire un batch (non-destructif)
        start_idx = batch_id * BATCH_SIZE
        end_idx = start_idx + BATCH_SIZE - 1

        batch = r.lrange(OLD_KEY, start_idx, end_idx)

        if not batch:
            break  # Plus d'éléments

        # Écrire le batch
        new_key = f"{NEW_KEY_PREFIX}:{batch_id}"
        r.rpush(new_key, *batch)

        migrated_count += len(batch)
        batch_id += 1

        print(f"Migrated batch {batch_id}: {len(batch)} elements ({migrated_count}/{total_elements})")

        # Pause pour ne pas saturer (5ms)
        time.sleep(0.005)

    print(f"✓ Migration complete: {batch_id} batches created")

    # ÉTAPE 3 : Double écriture (application écrit dans les deux)
    # ────────────────────────────────────────────────────────────
    print("\n⚠️  Action required:")
    print("1. Deploy application update to write to BOTH old and new keys")
    print("2. Wait for deployment (ensure no data loss)")
    print("3. Run verification (see verify_migration)")
    print("4. Switch reads to new keys")
    print("5. Stop writing to old key")
    print("6. Delete old key")


def verify_migration():
    """
    Vérifier que la migration est complète et cohérente
    """
    print("\n=== Verification ===")

    # Compter éléments dans old key
    old_count = r.llen(OLD_KEY)
    print(f"Old key ({OLD_KEY}): {old_count} elements")

    # Compter éléments dans new batches
    batch_id = 0
    new_count = 0

    while True:
        new_key = f"{NEW_KEY_PREFIX}:{batch_id}"
        batch_len = r.llen(new_key)

        if batch_len == 0:
            break

        new_count += batch_len
        batch_id += 1

    print(f"New keys ({NEW_KEY_PREFIX}:*): {new_count} elements across {batch_id} batches")

    # Vérifier cohérence
    if old_count == new_count:
        print("✓ Migration verified: counts match")
        return True
    else:
        print(f"✗ Mismatch: old={old_count}, new={new_count}")
        return False


def delete_old_key_safely():
    """
    Supprimer l'ancienne big key de manière sûre
    """
    print("\n=== Deleting old key ===")

    # Vérifier une dernière fois
    if not verify_migration():
        print("✗ Verification failed, aborting deletion")
        return

    # Utiliser UNLINK (asynchrone)
    result = r.unlink(OLD_KEY)

    if result:
        print(f"✓ Old key {OLD_KEY} deleted (unlinked)")
    else:
        print(f"Key {OLD_KEY} not found or already deleted")


# EXÉCUTION
# ─────────

if __name__ == "__main__":
    # Phase 1 : Migration
    migrate_big_list_to_batches()

    # Phase 2 : Attendre déploiement application avec double écriture
    input("\nPress Enter after deploying application with dual writes...")

    # Phase 3 : Vérification
    if verify_migration():
        # Phase 4 : Basculer lectures vers new keys
        input("\nPress Enter after switching reads to new keys...")

        # Phase 5 : Supprimer old key
        delete_old_key_safely()

        print("\n=== Migration Complete ===")
        print("Next steps:")
        print("1. Monitor for any issues")
        print("2. Remove dual-write code after 24-48h")
        print("3. Update documentation")
```

## Best practices et recommandations

```
┌─────────────────────────────────────────────────────────────┐
│              Best Practices - Hot Keys & Big Keys           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ DESIGN PRÉVENTIF                                            │
│ ════════════════                                            │
│                                                             │
│ ✓ Modéliser données pour éviter hot/big keys dès la conception
│ ✓ Utiliser hash tags strategiquement pour fragmentation     │
│ ✓ Limiter taille des collections (LTRIM, ZREMRANGEBYRANK)   │
│ ✓ TTL systématique sur données temporaires                  │
│ ✓ Compression pour grandes valeurs (gzip)                   │
│ ✓ Cache local pour config globale / données read-heavy      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ MONITORING PROACTIF                                         │
│ ═══════════════════                                         │
│                                                             │
│ ✓ Scan quotidien avec --bigkeys                             │
│ ✓ Alertes sur déséquilibre de charge entre nœuds            │
│ ✓ Métriques : p99 latency, network I/O par nœud             │
│ ✓ Dashboard Grafana avec top keys par taille/accès          │
│ ✓ Slowlog monitoring continu                                │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ OPÉRATIONS SÉCURISÉES                                       │
│ ══════════════════════                                      │
│                                                             │
│ ✓ UNLINK au lieu de DEL pour big keys                       │
│ ✓ SCAN au lieu de KEYS / SMEMBERS / HGETALL                 │
│ ✓ Pagination (HSCAN, SSCAN, ZSCAN) pour itérations          │
│ ✓ Pipeline pour réduire RTT sur opérations multiples        │
│ ✓ Éviter MONITOR en production (très coûteux)               │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ARCHITECTURE RÉSILIENTE                                     │
│ ═══════════════════════                                     │
│                                                             │
│ ✓ Cache local (application-side) pour hot keys              │
│ ✓ CDN pour contenu public statique/semi-statique            │
│ ✓ Circuit breaker pour clés problématiques                  │
│ ✓ Fallback mechanisms (degraded mode)                       │
│ ✓ Rate limiting côté application si nécessaire              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ DOCUMENTATION ET FORMATION                                  │
│ ══════════════════════════════                              │
│                                                             │
│ ✓ Runbooks pour incidents hot/big keys                      │
│ ✓ Formation équipe sur patterns anti-patterns               │
│ ✓ Code reviews pour identifier risques                      │
│ ✓ Post-mortems après chaque incident                        │
│ ✓ Partage de learnings inter-équipes                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Conclusion

La gestion des Hot Keys et Big Keys dans Redis Cluster est un défi permanent qui nécessite une approche multidimensionnelle combinant prévention, détection et réaction. Ces problèmes sont particulièrement critiques dans un environnement distribué car ils créent des déséquilibres qui ne peuvent être résolus simplement en ajoutant des nœuds.

Les points essentiels à retenir :

1. **Prévention** : Conception applicative évitant création de hot/big keys
2. **Détection précoce** : Monitoring continu avec alertes
3. **Réaction rapide** : Runbooks et procédures d'urgence testées
4. **Solutions durables** : Fragmentation, cache local, CDN selon contexte
5. **Amélioration continue** : Post-mortems et partage de connaissances

Une stratégie efficace combine plusieurs techniques :
- Cache local pour hot keys de lecture
- Fragmentation pour big keys
- Monitoring proactif avec alertes
- Runbooks pour incidents
- Architecture résiliente avec fallbacks

En maîtrisant ces aspects, il devient possible de maintenir un cluster Redis performant et stable même face à des patterns d'accès non uniformes ou des données volumineuses.

---

**Points clés à retenir :**

- **Hot Key** : Taux d'accès disproportionné → saturation d'un nœud
- **Big Key** : Taille excessive → blocage du serveur (single-threaded)
- **Détection** : --hotkeys, --bigkeys, MONITOR, slowlog, instrumentation
- **Impact cluster** : Déséquilibre, impossible à résoudre par ajout de nœuds
- **Solutions hot keys** : Cache local, réplication, CDN, sharding
- **Solutions big keys** : Fragmentation, UNLINK, pagination (SCAN), compression
- **UNLINK vs DEL** : UNLINK est asynchrone, crucial pour big keys
- **Monitoring** : Prometheus, Grafana, alertes sur déséquilibre

La prochaine section (11.9) explorera la réplication cross-datacenter pour la haute disponibilité géographique.

⏭️ [Cross-datacenter replication (Active-Active vs Active-Passive)](/11-architecture-distribuee-scaling/09-cross-datacenter-replication.md)
