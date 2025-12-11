🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.2 - Memory Analysis : --bigkeys, --memkeys et memory doctor

## 🎯 Objectifs de cette section

- Maîtriser les outils natifs d'analyse mémoire Redis
- Identifier et résoudre les problèmes de consommation mémoire
- Diagnostiquer les big keys et leur impact sur les performances
- Optimiser l'utilisation mémoire en production
- Mettre en place une stratégie de monitoring mémoire proactive

---

## 📚 Introduction : La mémoire dans Redis

### Pourquoi l'analyse mémoire est critique

Redis est une base de données **in-memory** :
- Toutes les données sont en RAM
- La mémoire est la ressource la plus précieuse
- Une mauvaise gestion → OOM (Out Of Memory) → crash

```
┌───────────────────────────────────────────────┐
│  REDIS MEMORY ARCHITECTURE                    │
├───────────────────────────────────────────────┤
│                                               │
│  ┌───────────────────────────────────────┐    │
│  │  Data Memory (clés + valeurs)         │    │
│  │  - Strings, Lists, Sets, Hashes...    │    │
│  │  - 80-95% de la mémoire totale        │    │
│  └───────────────────────────────────────┘    │
│                ↓                              │
│  ┌───────────────────────────────────────┐    │
│  │  Overhead Memory                      │    │
│  │  - Structures internes                │    │
│  │  - Dictionnaires, pointeurs           │    │
│  │  - 5-15% de la mémoire                │    │
│  └───────────────────────────────────────┘    │
│                ↓                              │
│  ┌───────────────────────────────────────┐    │
│  │  Buffer Memory                        │    │
│  │  - Client buffers                     │    │
│  │  - Replication buffer                 │    │
│  │  - AOF buffer                         │    │
│  └───────────────────────────────────────┘    │
│                ↓                              │
│  ┌───────────────────────────────────────┐    │
│  │  Fragmentation                        │    │
│  │  - Mémoire inutilisée mais allouée    │    │
│  │  - Variable selon l'allocateur        │    │
│  └───────────────────────────────────────┘    │
└───────────────────────────────────────────────┘
```

### Les symptômes de problèmes mémoire

| Symptôme | Cause probable | Impact |
|----------|----------------|--------|
| **OOM (Out Of Memory)** | Mémoire saturée | Crash Redis |
| **Évictions fréquentes** | Proche de maxmemory | Perte de données |
| **Fragmentation élevée** | Allocation/désallocation | Gaspillage RAM |
| **Latence élevée** | Big keys | Blocage des commandes |
| **Réplication lente** | Buffer de réplication plein | Lag master/replica |
| **Swap utilisé** | RAM insuffisante | Performances catastrophiques |

### Métriques mémoire essentielles

```bash
# Commande rapide pour snapshot mémoire
redis-cli INFO memory

# Métriques clés à surveiller :
used_memory_human        # Mémoire utilisée par les données
used_memory_rss_human    # Mémoire réelle (RSS) du process
used_memory_peak_human   # Pic historique
mem_fragmentation_ratio  # Ratio de fragmentation
maxmemory               # Limite configurée
evicted_keys            # Nombre de clés évictées
```

---

## 🔧 Tool 1 : redis-cli --bigkeys

### Qu'est-ce que --bigkeys ?

Un outil de **scan et analyse** qui parcourt tout le keyspace pour identifier les plus grosses clés.

**Fonctionnement** :
- Utilise SCAN en interne (non-bloquant)
- Échantillonne et mesure chaque type de données
- Génère un rapport statistique

### Utilisation de base

```bash
# Scan complet du keyspace
redis-cli --bigkeys

# Avec authentification
redis-cli -a password --bigkeys

# Spécifier la base de données
redis-cli -n 1 --bigkeys

# Avec intervalle entre les scans (pour réduire l'impact)
redis-cli --bigkeys -i 0.1  # 100ms entre chaque scan
```

### Format de sortie

```bash
$ redis-cli --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Sampled 1000 keys so far
[00.50%] Sampled 5000 keys so far
[01.00%] Sampled 10000 keys so far
...

-------- summary -------

Sampled 1000000 keys in the keyspace!
Total key length in bytes is 25000000 (avg len 25.00)

Biggest string found 'cache:homepage:en' has 5242880 bytes
Biggest list   found 'queue:tasks' has 100000 items
Biggest set    found 'tags:all' has 50000 members
Biggest hash   found 'user:12345:profile' has 10000 fields
Biggest zset   found 'leaderboard:global' has 1000000 members
Biggest stream found 'events:log' has 500000 entries

0 strings with 0 bytes (00.00% of keys, avg size 0.00)
450000 lists with 45000000 items (45.00% of keys, avg size 100.00)
300000 sets with 15000000 members (30.00% of keys, avg size 50.00)
200000 hashs with 40000000 fields (20.00% of keys, avg size 200.00)
50000 zsets with 25000000 members (05.00% of keys, avg size 500.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
```

### Interprétation des résultats

#### 1. Identifier les big keys problématiques

**Seuils problématiques** :

```
Type      | Taille normale | Big Key    | Très problématique
----------|----------------|------------|-------------------
String    | < 100 KB       | > 1 MB     | > 10 MB
List      | < 1000 items   | > 10K      | > 100K items
Set       | < 1000 members | > 10K      | > 100K members
Hash      | < 1000 fields  | > 10K      | > 100K fields
Sorted Set| < 1000 members | > 10K      | > 100K members
Stream    | < 10K entries  | > 100K     | > 1M entries
```

#### 2. Analyser la distribution

```bash
# Si vous avez beaucoup de big keys :
# → Problème de modèle de données

# Distribution inégale (Pareto 80/20) :
# 80% de la mémoire utilisée par 20% des clés
# → Optimiser ces 20%

# Moyenne élevée mais pas de grosses clés :
# → Problème de volume global
```

### Limites de --bigkeys

❌ **Ce que --bigkeys NE fait PAS** :
- Ne donne pas la mémoire réelle utilisée (seulement le nombre d'éléments)
- N'analyse pas le contenu des clés
- Ne détecte pas les "hot keys" (clés très sollicitées)
- Échantillonne seulement (pas exhaustif pour tous les types)

✅ **Ce que --bigkeys FAIT bien** :
- Scan rapide et sécurisé (non-bloquant)
- Vue d'ensemble statistique
- Identification des outliers

---

## 🔍 Tool 2 : redis-cli --memkeys (Redis 4.0+)

### Qu'est-ce que --memkeys ?

Une version **améliorée de --bigkeys** qui mesure la **mémoire réelle** consommée, pas seulement le nombre d'éléments.

⚠️ **Note** : Fonctionnalité disponible à partir de Redis 4.0+

### Utilisation

```bash
# Analyse mémoire complète
redis-cli --memkeys

# Avec sampling (plus rapide)
redis-cli --memkeys --memkeys-samples 1000

# Avec intervalle pour réduire l'impact
redis-cli --memkeys -i 0.1
```

### Format de sortie

```bash
$ redis-cli --memkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.

[00.00%] Sampled 1000 keys so far
...

-------- summary -------

Sampled 100000 keys in the keyspace!

Biggest string found 'cache:homepage:en' has 5242880 bytes
Biggest list   found 'queue:tasks' uses 15728640 bytes
Biggest set    found 'tags:all' uses 2097152 bytes
Biggest hash   found 'user:12345:profile' uses 8388608 bytes
Biggest zset   found 'leaderboard:global' uses 52428800 bytes

10000 strings with 104857600 bytes (10.00% of keys, avg size 10485.76)
30000 lists with 471859200 bytes (30.00% of keys, avg size 15728.64)
...
```

### Avantages de --memkeys vs --bigkeys

| Critère | --bigkeys | --memkeys |
|---------|-----------|-----------|
| **Métrique** | Nombre d'éléments | Bytes réels |
| **Précision** | Approximative | Exacte |
| **Performance** | Rapide | Plus lent |
| **Version Redis** | Toutes | 4.0+ |
| **Usage** | Première analyse | Deep dive |

### Cas d'usage optimal

```bash
# 1. First pass : Vue d'ensemble rapide
redis-cli --bigkeys -i 0.1

# 2. Second pass : Analyse mémoire détaillée
redis-cli --memkeys --memkeys-samples 10000

# 3. Focus : Analyse spécifique d'une clé
redis-cli MEMORY USAGE user:12345:profile SAMPLES 0
```

---

## 🩺 Tool 3 : MEMORY DOCTOR

### Qu'est-ce que MEMORY DOCTOR ?

Un **diagnostic automatisé** qui analyse l'état de la mémoire et fournit des recommandations.

### Utilisation

```bash
# Diagnostic complet
redis-cli MEMORY DOCTOR
```

### Exemples de sorties

#### Cas 1 : Fragmentation élevée

```
Sam, I detected a few issues in this Redis instance memory implants:

 * High fragmentation: This instance has a memory fragmentation
   greater than 1.4 (it is 1.52). This fragmentation is mainly
   due to changes in the data you are storing in Redis over time.

   Suggestion: Try to run 'MEMORY PURGE' or restart the instance
   to solve this issue. If the problem persists consider
   'activedefrag' configuration option.
```

#### Cas 2 : Mémoire quasi-saturée

```
Sam, I detected a few issues in this Redis instance memory implants:

 * Peak memory: The memory usage of this instance is very close
   to the peak memory usage. Current: 9.8 GB, Peak: 9.9 GB.

   Suggestion: You are near the memory limit. Consider increasing
   the maxmemory limit, enabling eviction policies, or archiving
   old data.
```

#### Cas 3 : Big keys détectés

```
Sam, I detected a few issues in this Redis instance memory implants:

 * Big keys: Your instance contains keys that are using a lot of
   memory. Run 'redis-cli --bigkeys' to find them.

   Suggestion: Consider splitting large keys into smaller ones or
   using different data structures.
```

#### Cas 4 : Tout va bien

```
Sam, this instance memory is OK.
```

### Analyse des recommandations

| Message | Signification | Action |
|---------|---------------|--------|
| **High fragmentation** | mem_frag_ratio > 1.4 | MEMORY PURGE ou restart |
| **Peak memory** | Proche de maxmemory | Augmenter limite ou éviction |
| **Big keys** | Grosses clés présentes | Identifier et refactorer |
| **High clients count** | Trop de clients connectés | Vérifier les connection pools |
| **AOF buffer high** | Buffer AOF volumineux | Optimiser AOF ou augmenter RAM |

---

## 🔬 Commandes MEMORY avancées (Redis 4.0+)

### MEMORY USAGE <key> [SAMPLES <count>]

Mesure la mémoire exacte utilisée par une clé spécifique.

```bash
# Mémoire utilisée par une clé
redis-cli MEMORY USAGE user:12345:profile

# Résultat : (integer) 8388736  # 8 MB

# Avec échantillonnage (plus précis)
redis-cli MEMORY USAGE user:12345:profile SAMPLES 5

# Pour tous les types
redis-cli MEMORY USAGE mystring
redis-cli MEMORY USAGE mylist
redis-cli MEMORY USAGE myset
redis-cli MEMORY USAGE myhash
redis-cli MEMORY USAGE myzset
redis-cli MEMORY USAGE mystream
```

**Calcul de la mémoire** :
- Inclut la valeur + overhead Redis (pointeurs, structures)
- Précision dépend du paramètre SAMPLES (0 = rapide, 5 = précis)

### MEMORY STATS

Statistiques détaillées sur l'utilisation mémoire.

```bash
redis-cli MEMORY STATS
```

**Sortie détaillée** :

```
 1) "peak.allocated"
 2) (integer) 10737418240    # 10 GB pic historique
 3) "total.allocated"
 4) (integer) 8589934592     # 8 GB actuellement alloués
 5) "startup.allocated"
 6) (integer) 1048576        # 1 MB au démarrage
 7) "replication.backlog"
 8) (integer) 1048576        # 1 MB backlog réplication
 9) "clients.slaves"
10) (integer) 0              # 0 bytes pour les replicas
11) "clients.normal"
12) (integer) 16384          # 16 KB pour les clients normaux
13) "aof.buffer"
14) (integer) 0              # AOF buffer vide
15) "db.0"
16) 1) "overhead.hashtable.main"
    2) (integer) 1048576     # Overhead des tables de hash
    3) "overhead.hashtable.expires"
    4) (integer) 524288      # Overhead des expirations
17) "overhead.total"
18) (integer) 2621440        # Total overhead
19) "keys.count"
20) (integer) 1000000        # 1M clés
21) "keys.bytes-per-key"
22) (integer) 128            # 128 bytes overhead par clé
23) "dataset.bytes"
24) (integer) 8587313152     # 8 GB de données pures
25) "dataset.percentage"
26) "99.97"                  # 99.97% de la mémoire = données
27) "peak.percentage"
28) "79.99"                  # 80% du pic historique
29) "fragmentation"
30) "1.25"                   # Ratio de fragmentation
```

### MEMORY PURGE

Force la libération de mémoire fragmentée (utilise l'allocateur système).

```bash
redis-cli MEMORY PURGE
# Résultat : OK

# Vérifier l'impact
redis-cli INFO memory | grep mem_fragmentation_ratio
```

⚠️ **Attention** : Peut bloquer Redis pendant quelques millisecondes.

### MEMORY MALLOC-STATS

Statistiques de l'allocateur mémoire (jemalloc par défaut).

```bash
redis-cli MEMORY MALLOC-STATS
```

**Sortie complexe** (extrait) :

```
___ Begin jemalloc statistics ___
Version: 5.2.1
...
Allocated: 8589934592, active: 8724152320, metadata: 159383552,
resident: 9112182784, mapped: 10737418240
...
```

**Utilité** : Diagnostic avancé de fragmentation et leak mémoire.

---

## 📊 Méthodologie d'analyse complète

### Phase 1 : Vue d'ensemble (5 minutes)

```bash
#!/bin/bash
# quick-memory-check.sh

echo "=== REDIS MEMORY QUICK CHECK ==="
echo ""

# 1. Métriques de base
echo "--- Basic Metrics ---"
redis-cli INFO memory | grep -E "used_memory_human|used_memory_rss_human|used_memory_peak_human|mem_fragmentation_ratio|maxmemory_human|evicted_keys"

echo ""

# 2. MEMORY DOCTOR
echo "--- MEMORY DOCTOR ---"
redis-cli MEMORY DOCTOR

echo ""

# 3. Distribution par base de données
echo "--- Database Sizes ---"
for db in {0..15}; do
    size=$(redis-cli -n $db DBSIZE 2>/dev/null)
    if [ "$size" != "0" ]; then
        echo "DB $db: $size keys"
    fi
done
```

### Phase 2 : Identification des big keys (15 minutes)

```bash
#!/bin/bash
# analyze-bigkeys.sh

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT_DIR="memory_analysis_${TIMESTAMP}"
mkdir -p $OUTPUT_DIR

echo "=== BIG KEYS ANALYSIS ==="
echo ""

# 1. Scan avec --bigkeys
echo "Running --bigkeys scan..."
redis-cli --bigkeys -i 0.1 > ${OUTPUT_DIR}/bigkeys_report.txt 2>&1

# 2. Extraire les plus grosses clés
echo "Top 10 biggest keys:"
grep "Biggest" ${OUTPUT_DIR}/bigkeys_report.txt

# 3. Mesurer la mémoire de chaque big key
echo ""
echo "Measuring memory usage..."

# Extraire les noms de clés
grep "Biggest" ${OUTPUT_DIR}/bigkeys_report.txt | \
  awk '{print $(NF-2)}' | \
  sed "s/'//g" | \
  while read key; do
    memory=$(redis-cli MEMORY USAGE "$key" 2>/dev/null)
    type=$(redis-cli TYPE "$key" 2>/dev/null)
    ttl=$(redis-cli TTL "$key" 2>/dev/null)
    echo "Key: $key | Type: $type | Memory: $memory bytes | TTL: $ttl"
  done > ${OUTPUT_DIR}/bigkeys_memory.txt

cat ${OUTPUT_DIR}/bigkeys_memory.txt

echo ""
echo "Reports saved in: $OUTPUT_DIR/"
```

### Phase 3 : Analyse détaillée (30 minutes)

```python
#!/usr/bin/env python3
"""
Advanced memory analysis script
"""
import redis
import sys
from collections import defaultdict

def analyze_memory_distribution(host='localhost', port=6379, samples=10000):
    """Analyse la distribution mémoire par pattern de clés"""

    r = redis.Redis(host=host, port=port, decode_responses=True)

    # Collecter les clés par pattern
    patterns = defaultdict(list)
    cursor = 0
    scanned = 0

    print("Scanning keyspace...")

    while True:
        cursor, keys = r.scan(cursor, count=100)

        for key in keys:
            # Extraire le pattern (préfixe avant le premier ':')
            if ':' in key:
                pattern = key.split(':')[0] + ':*'
            else:
                pattern = 'no-prefix'

            # Mesurer la mémoire (sampling)
            if len(patterns[pattern]) < 100:  # Limiter à 100 échantillons par pattern
                try:
                    memory = r.memory_usage(key)
                    if memory:
                        patterns[pattern].append({
                            'key': key,
                            'memory': memory,
                            'type': r.type(key)
                        })
                except:
                    pass

        scanned += len(keys)
        if scanned >= samples or cursor == 0:
            break

        if scanned % 1000 == 0:
            print(f"Scanned {scanned} keys...")

    # Analyse par pattern
    print("\n=== MEMORY DISTRIBUTION BY PATTERN ===\n")

    pattern_stats = []

    for pattern, keys in patterns.items():
        if not keys:
            continue

        total_memory = sum(k['memory'] for k in keys)
        avg_memory = total_memory / len(keys)
        max_memory = max(k['memory'] for k in keys)

        pattern_stats.append({
            'pattern': pattern,
            'samples': len(keys),
            'total_mb': total_memory / 1024 / 1024,
            'avg_bytes': avg_memory,
            'max_bytes': max_memory
        })

    # Trier par mémoire totale
    pattern_stats.sort(key=lambda x: x['total_mb'], reverse=True)

    # Afficher les résultats
    print(f"{'Pattern':<30} {'Samples':<10} {'Total (MB)':<15} {'Avg (bytes)':<15} {'Max (bytes)':<15}")
    print("-" * 90)

    for stat in pattern_stats[:20]:  # Top 20
        print(f"{stat['pattern']:<30} {stat['samples']:<10} {stat['total_mb']:<15.2f} {stat['avg_bytes']:<15.0f} {stat['max_bytes']:<15.0f}")

    # Calculer le total
    total_sampled_mb = sum(s['total_mb'] for s in pattern_stats)
    print("-" * 90)
    print(f"{'TOTAL (sampled)':<30} {scanned:<10} {total_sampled_mb:<15.2f}")

    # Identifier les problèmes
    print("\n=== ISSUES DETECTED ===\n")

    for stat in pattern_stats:
        issues = []

        if stat['avg_bytes'] > 1024 * 1024:  # Moyenne > 1MB
            issues.append(f"High average size: {stat['avg_bytes']/1024/1024:.2f} MB")

        if stat['max_bytes'] > 10 * 1024 * 1024:  # Max > 10MB
            issues.append(f"Very large key found: {stat['max_bytes']/1024/1024:.2f} MB")

        if issues:
            print(f"Pattern: {stat['pattern']}")
            for issue in issues:
                print(f"  ⚠️  {issue}")
            print()

if __name__ == "__main__":
    analyze_memory_distribution()
```

---

## 🎭 Patterns de problèmes mémoire classiques

### Pattern 1 : Big String Keys

**Signature** :
```bash
redis-cli --bigkeys
# Biggest string found 'cache:page:homepage' has 52428800 bytes (50MB)
```

**Diagnostic** :

```bash
# Vérifier le contenu
redis-cli --raw GET cache:page:homepage | head -c 1000

# Type d'encoding
redis-cli OBJECT ENCODING cache:page:homepage
# Résultat : "raw" (indique un string volumineux)

# Mémoire exacte
redis-cli MEMORY USAGE cache:page:homepage
# Résultat : 52428800
```

**Solutions** :

```bash
# Option 1 : Compression côté application
# Avant : stocker HTML brut (50MB)
# Après : stocker HTML gzippé (5MB)

# Option 2 : Fragmenter en plusieurs clés
# Au lieu de : cache:page:homepage (50MB)
# Faire :
#   cache:page:homepage:header (5MB)
#   cache:page:homepage:body (40MB)
#   cache:page:homepage:footer (5MB)

# Option 3 : Utiliser RedisJSON pour structure
# Permet d'accéder à des parties sans tout charger
JSON.SET cache:page:homepage $ '{"header": "...", "body": "...", "footer": "..."}'
JSON.GET cache:page:homepage $.header
```

### Pattern 2 : Large Hash avec beaucoup de champs

**Signature** :
```bash
redis-cli --bigkeys
# Biggest hash found 'user:12345:profile' has 100000 fields
```

**Diagnostic** :

```bash
# Nombre de champs
redis-cli HLEN user:12345:profile
# Résultat : 100000

# Mémoire totale
redis-cli MEMORY USAGE user:12345:profile
# Résultat : 15728640 (15MB)

# Échantillonner quelques champs
redis-cli HSCAN user:12345:profile 0 COUNT 10
```

**Causes courantes** :
- Modèle de données inadapté
- Accumulation sans nettoyage
- Utilisation comme "table" SQL

**Solutions** :

```bash
# ❌ UN SEUL gros hash
HSET user:12345:profile field1 value1 field2 value2 ... field100000 value100000

# ✅ SPLIT en plusieurs hash par catégorie
HSET user:12345:basic name "John" email "john@example.com"
HSET user:12345:preferences theme "dark" language "en"
HSET user:12345:stats loginCount "150" lastLogin "2024-12-11"

# ✅ Ou utiliser une structure différente
# Redis Streams ou Sorted Sets selon le cas d'usage
```

### Pattern 3 : Sorted Sets massifs

**Signature** :
```bash
redis-cli --bigkeys
# Biggest zset found 'leaderboard:global' has 10000000 members
```

**Diagnostic** :

```bash
# Taille du sorted set
redis-cli ZCARD leaderboard:global
# Résultat : 10000000

# Mémoire
redis-cli MEMORY USAGE leaderboard:global
# Résultat : 800000000 (800MB!)

# Vérifier la distribution des scores
redis-cli ZRANGE leaderboard:global 0 10 WITHSCORES
redis-cli ZREVRANGE leaderboard:global 0 10 WITHSCORES
```

**Solutions** :

```bash
# Option 1 : Limiter la taille (top N)
# Garder seulement le top 100K
ZREMRANGEBYRANK leaderboard:global 0 -100001

# Option 2 : Archiver les vieux scores
# Déplacer les anciens dans une autre structure
ZRANGEBYSCORE leaderboard:global -inf (timestamp-30days) | xargs ZREM

# Option 3 : Partitionner par période
# Au lieu de : leaderboard:global (10M membres)
# Faire :
#   leaderboard:2024-12 (100K membres)
#   leaderboard:2024-11 (100K membres)
#   ...
```

### Pattern 4 : Lists comme queues non-drainées

**Signature** :
```bash
redis-cli --bigkeys
# Biggest list found 'queue:pending' has 5000000 items
```

**Diagnostic** :

```bash
# Taille de la liste
redis-cli LLEN queue:pending
# Résultat : 5000000

# Vérifier si elle grossit
redis-cli LLEN queue:pending
# Attendre 10 secondes
redis-cli LLEN queue:pending
# Si augmente → problème de consumer

# Échantillon du début et de la fin
redis-cli LRANGE queue:pending 0 5
redis-cli LRANGE queue:pending -5 -1
```

**Causes** :
- Producers trop rapides
- Consumers en panne
- Logique de traitement trop lente

**Solutions** :

```bash
# Option 1 : Augmenter les consumers
# Scaler horizontalement le traitement

# Option 2 : Rate limiting sur les producers
# Limiter l'injection dans la queue

# Option 3 : Migrer vers Redis Streams
# Meilleure gestion des consumers groups
XADD mystream * field1 value1 field2 value2
XREADGROUP GROUP mygroup consumer1 COUNT 100 STREAMS mystream >

# Option 4 : TTL sur les messages (avec Streams)
# Supprimer automatiquement les vieux messages
XTRIM mystream MAXLEN ~ 100000
```

### Pattern 5 : Clés temporaires non-nettoyées

**Signature** :
```bash
redis-cli --bigkeys
# Beaucoup de clés avec pattern "temp:*" ou "lock:*"
```

**Diagnostic** :

```bash
# Compter les clés par pattern
redis-cli --scan --pattern "temp:*" | wc -l

# Vérifier les TTL
redis-cli --scan --pattern "temp:*" | head -20 | \
  while read key; do
    ttl=$(redis-cli TTL "$key")
    echo "$key : TTL=$ttl"
  done
```

**Causes** :
- TTL non défini
- Crashes avant nettoyage
- Logique de cleanup défaillante

**Solutions** :

```bash
# ✅ TOUJOURS définir un TTL sur les clés temporaires
SET temp:session:abc123 value EX 3600  # 1 heure

# ✅ Utiliser SETEX pour atomicité
SETEX temp:lock:resource 30 "locked"

# ✅ Script de nettoyage périodique
# Cleanup job toutes les heures
0 * * * * /usr/local/bin/redis-cleanup-temp-keys.sh

# ✅ Monitoring des clés sans TTL
redis-cli --scan --pattern "temp:*" | \
  while read key; do
    ttl=$(redis-cli TTL "$key")
    if [ "$ttl" = "-1" ]; then
      echo "WARNING: $key has no TTL"
    fi
  done
```

---

## 🚨 Guide de résolution : Out Of Memory imminent

### Scénario : Redis proche de maxmemory

**Alertes** :
- `used_memory` > 90% de `maxmemory`
- Évictions commencent
- Applications signalent des erreurs

### Investigation immédiate (< 5 minutes)

```bash
# 1. État actuel
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|evicted_keys"

# 2. Vérifier les big keys
redis-cli --bigkeys -i 0.01 | grep "Biggest"

# 3. MEMORY DOCTOR
redis-cli MEMORY DOCTOR
```

### Actions d'urgence

#### Option 1 : Augmenter temporairement maxmemory

```bash
# ⚠️ Solution temporaire uniquement
redis-cli CONFIG SET maxmemory 16gb

# Vérifier RAM disponible sur le serveur
free -h

# Attention : ne pas dépasser 80% de la RAM totale
```

#### Option 2 : Activer/ajuster la politique d'éviction

```bash
# Si pas d'éviction configurée
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Ou politique plus agressive
redis-cli CONFIG SET maxmemory-policy volatile-lru
```

#### Option 3 : Nettoyage manuel urgent

```bash
# Identifier et supprimer des big keys non-critiques
redis-cli --bigkeys | grep "Biggest"

# Supprimer des clés temporaires obsolètes
redis-cli --scan --pattern "temp:*" | \
  while read key; do
    ttl=$(redis-cli TTL "$key")
    if [ "$ttl" = "-1" ]; then
      redis-cli DEL "$key"
    fi
  done

# Nettoyer des caches expirés
redis-cli --scan --pattern "cache:*" | \
  head -1000 | \
  xargs redis-cli DEL
```

### Analyse post-crise (< 1 heure)

```python
#!/usr/bin/env python3
"""
Post-mortem memory analysis
"""
import redis
import json
from datetime import datetime

def post_mortem_analysis(host='localhost', port=6379):
    r = redis.Redis(host=host, port=port, decode_responses=True)

    report = {
        'timestamp': datetime.now().isoformat(),
        'memory_info': {},
        'top_memory_consumers': [],
        'recommendations': []
    }

    # Métriques mémoire
    info = r.info('memory')
    report['memory_info'] = {
        'used_memory_mb': info['used_memory'] / 1024 / 1024,
        'used_memory_peak_mb': info['used_memory_peak'] / 1024 / 1024,
        'fragmentation_ratio': info['mem_fragmentation_ratio'],
        'evicted_keys': info.get('evicted_keys', 0)
    }

    # Scanner les big keys
    print("Scanning for big keys...")
    cursor = 0
    big_keys = []

    while True:
        cursor, keys = r.scan(cursor, count=100)

        for key in keys:
            try:
                memory = r.memory_usage(key)
                if memory and memory > 1024 * 1024:  # > 1MB
                    big_keys.append({
                        'key': key,
                        'type': r.type(key),
                        'memory_mb': memory / 1024 / 1024,
                        'ttl': r.ttl(key)
                    })
            except:
                pass

        if cursor == 0:
            break

    # Top 20
    big_keys.sort(key=lambda x: x['memory_mb'], reverse=True)
    report['top_memory_consumers'] = big_keys[:20]

    # Recommandations
    if report['memory_info']['fragmentation_ratio'] > 1.5:
        report['recommendations'].append(
            "High fragmentation detected. Consider restarting Redis or enabling active defragmentation."
        )

    if report['memory_info']['evicted_keys'] > 0:
        report['recommendations'].append(
            f"{report['memory_info']['evicted_keys']} keys were evicted. Increase maxmemory or optimize data model."
        )

    if len(big_keys) > 10:
        report['recommendations'].append(
            f"{len(big_keys)} keys are using more than 1MB each. Review data model and consider splitting large keys."
        )

    # Sauvegarder le rapport
    filename = f"memory_postmortem_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    with open(filename, 'w') as f:
        json.dump(report, f, indent=2)

    print(f"\nPost-mortem report saved to: {filename}")
    print("\n=== SUMMARY ===")
    print(f"Used memory: {report['memory_info']['used_memory_mb']:.2f} MB")
    print(f"Peak memory: {report['memory_info']['used_memory_peak_mb']:.2f} MB")
    print(f"Fragmentation: {report['memory_info']['fragmentation_ratio']:.2f}")
    print(f"Evicted keys: {report['memory_info']['evicted_keys']}")
    print(f"\nTop 5 memory consumers:")
    for i, key in enumerate(report['top_memory_consumers'][:5], 1):
        print(f"  {i}. {key['key']} ({key['type']}): {key['memory_mb']:.2f} MB")

if __name__ == "__main__":
    post_mortem_analysis()
```

---

## 📈 Monitoring continu et préventif

### Métriques clés à monitorer

```yaml
# Prometheus metrics to track
- redis_memory_used_bytes
- redis_memory_max_bytes
- redis_memory_fragmentation_ratio
- redis_evicted_keys_total
- redis_memory_used_peak_bytes
- redis_mem_clients_slaves
- redis_mem_clients_normal
```

### Alertes recommandées

```yaml
# alerting-rules.yml

groups:
  - name: redis_memory
    rules:
      # Alerte : Mémoire > 80%
      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage is high"
          description: "{{ $labels.instance }} is using {{ $value | humanizePercentage }} of maxmemory"

      # Alerte : Mémoire > 95%
      - alert: RedisMemoryCritical
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis memory usage is critical"
          description: "{{ $labels.instance }} is using {{ $value | humanizePercentage }} of maxmemory - OOM imminent"

      # Alerte : Fragmentation élevée
      - alert: RedisHighFragmentation
        expr: redis_memory_fragmentation_ratio > 1.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory fragmentation is high"
          description: "{{ $labels.instance }} fragmentation ratio is {{ $value }}"

      # Alerte : Évictions actives
      - alert: RedisEvictionsOccurring
        expr: rate(redis_evicted_keys_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis is evicting keys"
          description: "{{ $labels.instance }} is evicting {{ $value }} keys/sec"
```

### Dashboard Grafana

**Panels essentiels** :

```
1. Memory Usage (gauge)
   - used_memory vs maxmemory
   - Seuils : 80% warning, 95% critical

2. Memory Fragmentation (time series)
   - Ratio de fragmentation dans le temps
   - Ligne de référence à 1.5

3. Evictions (counter)
   - Nombre de clés évictées
   - Rate par seconde

4. Big Keys Alert (table)
   - Liste des plus grosses clés
   - Mise à jour quotidienne

5. Memory by Component (pie chart)
   - Dataset vs overhead vs buffers
```

---

## 🎓 Best Practices

### Configuration

✅ **DO**
- Définir `maxmemory` à 80% de la RAM disponible
- Activer `maxmemory-policy` appropriée
- Configurer `activedefrag yes` si fragmentation récurrente
- Monitorer en continu la mémoire

❌ **DON'T**
- Ne jamais laisser `maxmemory 0` (illimité) en production
- Ne pas ignorer les alertes de mémoire
- Ne pas stocker de gros objets binaires dans Redis sans compression

### Modèle de données

✅ **DO**
- Fragmenter les big keys en plusieurs petites clés
- Utiliser les structures de données appropriées
- Définir des TTL sur les données temporaires
- Compresser les gros strings côté application

❌ **DON'T**
- Ne pas utiliser Redis comme stockage de fichiers
- Ne pas stocker des documents XML/JSON > 1MB sans structure
- Ne pas laisser des listes/sets grossir indéfiniment

### Maintenance

✅ **DO**
- Scanner régulièrement avec `--bigkeys` (hebdomadaire)
- Analyser `MEMORY STATS` mensuellement
- Auditer les patterns de clés
- Nettoyer les clés obsolètes

❌ **DON'T**
- Ne jamais ignorer `MEMORY DOCTOR`
- Ne pas négliger la fragmentation > 1.5
- Ne pas reporter les nettoyages de clés obsolètes

---

## 🔗 Checklist d'analyse mémoire

### Analyse rapide (quotidienne)

- [ ] Vérifier `used_memory` vs `maxmemory`
- [ ] Consulter `mem_fragmentation_ratio`
- [ ] Vérifier `evicted_keys`
- [ ] Exécuter `MEMORY DOCTOR`

### Analyse approfondie (hebdomadaire)

- [ ] Exécuter `redis-cli --bigkeys`
- [ ] Analyser la distribution par pattern
- [ ] Identifier les croissances anormales
- [ ] Vérifier les clés sans TTL

### Audit complet (mensuel)

- [ ] Exécuter `redis-cli --memkeys`
- [ ] Analyser `MEMORY STATS` en détail
- [ ] Auditer le modèle de données
- [ ] Planifier les optimisations nécessaires
- [ ] Documenter les changements

---

## 📚 Ressources complémentaires

### Documentation officielle
- [Redis MEMORY commands](https://redis.io/commands/?group=server)
- [Memory Optimization](https://redis.io/docs/management/optimization/memory-optimization/)
- [Understanding Redis memory](https://redis.io/docs/management/optimization/memory-optimization/)

### Outils
- **Redis Insight** : GUI avec analyseur mémoire intégré
- **redis-rdb-tools** : Analyse offline des fichiers RDB
- **redis-memory-analyzer** : Outil tiers d'analyse

### Scripts utiles
- Scripts d'analyse dans cette section (Python, Bash)
- Dashboards Grafana pré-configurés
- Alertes Prometheus

---

## 🎯 Points clés à retenir

1. **--bigkeys pour vue d'ensemble** → Rapide mais approximatif
2. **--memkeys pour précision** → Plus lent mais exact (Redis 4.0+)
3. **MEMORY DOCTOR** → Diagnostic automatisé, toujours l'écouter
4. **MEMORY USAGE** → Mesure exacte par clé
5. **Big keys = goulot d'étranglement** → Fragmenter ou optimiser
6. **Monitoring continu** → Prévenir vaut mieux que guérir
7. **Fragmentation > 1.5** → Action nécessaire
8. **Évictions = signal d'alarme** → Augmenter RAM ou optimiser

---

**🚀 Section suivante** : [14.3 - Debugging avancé : MONITOR, CLIENT LIST, CLIENT KILL](./03-debugging-avance-monitor-client.md)

⏭️ [Debugging avancé : MONITOR, CLIENT LIST, CLIENT KILL](/14-performance-troubleshooting/03-debugging-avance-monitor-client.md)
