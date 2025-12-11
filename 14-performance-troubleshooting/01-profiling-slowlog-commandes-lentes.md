🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.1 - Profiling : Slowlog et analyse des commandes lentes

## 🎯 Objectifs de cette section

- Maîtriser le SLOWLOG pour identifier les goulots d'étranglement
- Analyser et interpréter les patterns de commandes lentes
- Mettre en place une méthodologie systématique de profiling
- Identifier les optimisations critiques en production
- Prévenir les problèmes de performance avant qu'ils n'impactent les utilisateurs

---

## 📚 Introduction : Pourquoi le profiling est critique

### Le paradoxe de Redis

Redis est conçu pour être **extrêmement rapide** :
- Ops/sec : 100,000+ par cœur CPU
- Latence moyenne : < 1ms (souvent < 0.1ms)
- Architecture single-threaded optimisée

**Mais** : Une seule commande lente peut bloquer toutes les autres requêtes.

```
┌─────────────────────────────────────────────────┐
│  Temps d'exécution des commandes                │
├─────────────────────────────────────────────────┤
│  GET key          →  0.05ms  ✅                 │
│  SET key value    →  0.06ms  ✅                 │
│  INCR counter     →  0.04ms  ✅                 │
│  KEYS *           → 500.00ms  ❌ CATASTROPHE    │
│  GET key2         →  0.05ms (attend 500ms!)     │
└─────────────────────────────────────────────────┘
```

**Impact** : Pendant que KEYS * s'exécute, toutes les autres commandes sont en attente.

### Les 3 niveaux de "lenteur"

| Niveau | Durée | Impact | Priorité |
|--------|-------|--------|----------|
| **Acceptable** | < 1ms | Aucun | Monitoring |
| **Préoccupant** | 1-10ms | Latence perceptible | Investigation |
| **Critique** | > 10ms | Timeouts, erreurs | Action immédiate |

---

## 🔧 Le SLOWLOG : Votre meilleur allié

### Qu'est-ce que le SLOWLOG ?

Le SLOWLOG est un **enregistreur de commandes lentes** intégré à Redis :
- Enregistre automatiquement les commandes dépassant un seuil
- Stocké en mémoire (pas d'impact I/O)
- FIFO avec taille configurable
- Impact minimal sur les performances

### Architecture du SLOWLOG

```
┌────────────────────────────────────────────────┐
│          REDIS INSTANCE                        │
│                                                │
│  ┌────────────────────────────────────────┐    │
│  │  Command Queue                         │    │
│  │  ┌──────┬──────┬──────┬──────┐         │    │
│  │  │ GET  │ SET  │ KEYS │ GET  │         │    │
│  │  └──────┴──────┴──────┴──────┘         │    │
│  └────────────────────────────────────────┘    │
│               ↓                                │
│  ┌────────────────────────────────────────┐    │
│  │  Execution Timer                       │    │
│  │  (mesure le temps d'exécution)         │    │
│  └────────────────────────────────────────┘    │
│               ↓                                │
│  ┌────────────────────────────────────────┐    │
│  │  Slowlog Threshold Check               │    │
│  │  Si durée > slowlog-log-slower-than    │    │
│  └────────────────────────────────────────┘    │
│               ↓                                │
│  ┌────────────────────────────────────────┐    │
│  │  SLOWLOG (ring buffer in-memory)       │    │
│  │  ┌──────────────────────────────┐      │    │
│  │  │ Entry 1: KEYS * (500ms)      │      │    │
│  │  │ Entry 2: SMEMBERS big (50ms) │      │    │
│  │  │ Entry 3: HGETALL huge (20ms) │      │    │
│  │  │ ...                          │      │    │
│  │  └──────────────────────────────┘      │    │
│  └────────────────────────────────────────┘    │
└────────────────────────────────────────────────┘
```

---

## ⚙️ Configuration du SLOWLOG

### Paramètres de configuration

#### 1. `slowlog-log-slower-than` (seuil en microsecondes)

Définit à partir de quelle durée une commande est considérée comme "lente".

```bash
# Valeur par défaut : 10000 (10ms)
CONFIG GET slowlog-log-slower-than
# Résultat : "10000"

# Modifier le seuil (runtime)
CONFIG SET slowlog-log-slower-than 1000  # 1ms

# Modifier dans redis.conf (permanent)
slowlog-log-slower-than 1000
```

**Valeurs recommandées** :

| Environnement | Seuil | Justification |
|---------------|-------|---------------|
| **Production critique** | 1000 μs (1ms) | Détection précoce |
| **Production standard** | 10000 μs (10ms) | Équilibre performance/détection |
| **Développement** | 100 μs (0.1ms) | Profiling agressif |
| **Debugging** | 0 μs | Log TOUTES les commandes |

⚠️ **Attention** : `slowlog-log-slower-than 0` log TOUTES les commandes → Impact mémoire !

#### 2. `slowlog-max-len` (nombre d'entrées)

Définit le nombre maximum d'entrées dans le SLOWLOG.

```bash
# Valeur par défaut : 128
CONFIG GET slowlog-max-len
# Résultat : "128"

# Augmenter la taille
CONFIG SET slowlog-max-len 1000

# Dans redis.conf
slowlog-max-len 1000
```

**Calcul de la mémoire utilisée** :

```
Chaque entrée ≈ 250-500 bytes (selon la longueur de la commande)

128 entrées   ≈  32-64 KB   ✅ Minimal
1000 entrées  ≈ 250-500 KB  ✅ Raisonnable
10000 entrées ≈ 2.5-5 MB    ⚠️ Attention en production
```

### Configuration recommandée pour la production

```conf
# redis.conf

# Seuil agressif pour détecter les problèmes tôt
slowlog-log-slower-than 1000  # 1ms

# Historique suffisant pour analyser les patterns
slowlog-max-len 500

# Alternative : seuil plus tolérant
# slowlog-log-slower-than 5000  # 5ms
# slowlog-max-len 1000
```

---

## 🔍 Utilisation du SLOWLOG

### Commandes principales

#### 1. SLOWLOG GET [count]

Récupère les dernières entrées du SLOWLOG.

```bash
# Récupérer les 10 dernières entrées
redis-cli SLOWLOG GET 10
```

**Format de sortie** :

```bash
1) 1) (integer) 14           # ID unique de l'entrée
   2) (integer) 1702293847   # Timestamp Unix
   3) (integer) 52431        # Durée en microsecondes (52.4ms)
   4) 1) "KEYS"              # Commande
      2) "*"                 # Arguments
   5) "127.0.0.1:43501"      # Client IP:port
   6) "client-name"          # Nom du client (si défini)

2) 1) (integer) 13
   2) (integer) 1702293845
   3) (integer) 15234
   4) 1) "SMEMBERS"
      2) "large_set"
   5) "127.0.0.1:43502"
   6) ""
```

#### 2. SLOWLOG LEN

Retourne le nombre d'entrées actuelles dans le SLOWLOG.

```bash
redis-cli SLOWLOG LEN
# Résultat : (integer) 127
```

#### 3. SLOWLOG RESET

Vide complètement le SLOWLOG.

```bash
redis-cli SLOWLOG RESET
# Résultat : OK
```

**Utilisation** : Nettoyer avant un test de charge spécifique.

---

## 📊 Méthodologie d'analyse : Le framework SLOW-CHECK

### Étape 1 : Capture initiale (SNAPSHOT)

```bash
#!/bin/bash
# Script de capture du SLOWLOG

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT_FILE="slowlog_${TIMESTAMP}.txt"

echo "=== SLOWLOG SNAPSHOT ===" > $OUTPUT_FILE
echo "Date: $(date)" >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

# Informations contextuelles
echo "--- Redis Info ---" >> $OUTPUT_FILE
redis-cli INFO stats | grep -E "total_commands|instantaneous_ops" >> $OUTPUT_FILE
redis-cli INFO memory | grep -E "used_memory_human|mem_fragmentation" >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

# SLOWLOG complet
echo "--- SLOWLOG Entries ---" >> $OUTPUT_FILE
redis-cli SLOWLOG GET 200 >> $OUTPUT_FILE

# Résumé
echo "" >> $OUTPUT_FILE
echo "--- Summary ---" >> $OUTPUT_FILE
redis-cli SLOWLOG LEN >> $OUTPUT_FILE

echo "Snapshot saved to: $OUTPUT_FILE"
```

### Étape 2 : Analyse des patterns (PATTERN DETECTION)

#### Extraction et agrégation

```bash
# Extraire uniquement les commandes
redis-cli SLOWLOG GET 100 | grep -A 2 "integer" | grep -v "integer" | grep -v "^--$"

# Compter les occurrences par type de commande
redis-cli SLOWLOG GET 100 | \
  grep -E "^\s+[0-9]+\)" | \
  awk '{print $1}' | \
  sort | uniq -c | sort -rn
```

**Script d'analyse Python** :

```python
#!/usr/bin/env python3
import redis
from collections import Counter
import statistics

def analyze_slowlog(host='localhost', port=6379):
    r = redis.Redis(host=host, port=port, decode_responses=True)

    slowlog = r.slowlog_get(500)

    if not slowlog:
        print("No slow queries found.")
        return

    # Analyse par type de commande
    commands = Counter()
    durations = []
    command_durations = {}

    for entry in slowlog:
        cmd = entry['command']
        duration = entry['duration']  # en microsecondes

        commands[cmd] += 1
        durations.append(duration)

        if cmd not in command_durations:
            command_durations[cmd] = []
        command_durations[cmd].append(duration)

    # Résultats
    print("=== SLOWLOG ANALYSIS ===\n")

    print(f"Total slow queries: {len(slowlog)}")
    print(f"Unique commands: {len(commands)}\n")

    print("--- Top 10 slowest commands ---")
    for cmd, count in commands.most_common(10):
        avg_duration = statistics.mean(command_durations[cmd])
        max_duration = max(command_durations[cmd])
        print(f"{cmd:20} | Count: {count:4} | Avg: {avg_duration/1000:7.2f}ms | Max: {max_duration/1000:7.2f}ms")

    print("\n--- Duration statistics ---")
    print(f"Average: {statistics.mean(durations)/1000:.2f}ms")
    print(f"Median:  {statistics.median(durations)/1000:.2f}ms")
    print(f"P95:     {sorted(durations)[int(len(durations)*0.95)]/1000:.2f}ms")
    print(f"P99:     {sorted(durations)[int(len(durations)*0.99)]/1000:.2f}ms")
    print(f"Max:     {max(durations)/1000:.2f}ms")

if __name__ == "__main__":
    analyze_slowlog()
```

### Étape 3 : Identification des causes (ROOT CAUSE)

Pour chaque commande lente identifiée :

#### Questions à poser :

1. **Quelle est la commande ?**
   - Commande O(N) ? (KEYS, SMEMBERS, HGETALL...)
   - Commande O(1) mais sur une big key ?
   - Commande avec arguments problématiques ?

2. **Quelle est la fréquence ?**
   - Sporadique → Problème ponctuel
   - Régulière → Pattern d'accès problématique
   - Continue → Défaut de conception

3. **Quel est le client ?**
   - Quelle application ?
   - Quelle version du code ?
   - Y a-t-il un pattern temporel ?

4. **Quel est le contexte système ?**
   - CPU élevé au moment de la commande ?
   - Mémoire saturée ?
   - I/O disque (persistance) ?

---

## 🎭 Les patterns de commandes lentes classiques

### Pattern 1 : KEYS * (Le tueur silencieux)

**Signature SLOWLOG** :
```
1) "KEYS"
2) "*"
Duration: 500-5000ms (selon la taille du keyspace)
```

**Pourquoi c'est lent** :
- O(N) où N = nombre total de clés
- Bloque Redis pendant toute la durée
- Aucune possibilité d'interruption

**Impact** :
```
1M clés → 1-2 secondes de blocage total
10M clés → 10-20 secondes de blocage total
```

**Solution** :

```bash
# ❌ JAMAIS
KEYS *

# ✅ TOUJOURS
SCAN 0 MATCH pattern* COUNT 100

# Alternative avec SCAN complet
cursor=0
while true; do
    result=$(redis-cli SCAN $cursor MATCH "user:*" COUNT 1000)
    cursor=$(echo "$result" | head -1)
    keys=$(echo "$result" | tail -n +2)

    # Traiter les clés ici
    echo "$keys"

    # Si cursor = 0, on a fini
    [ "$cursor" -eq 0 ] && break
done
```

### Pattern 2 : SMEMBERS sur un gros Set

**Signature SLOWLOG** :
```
1) "SMEMBERS"
2) "large_set"
Duration: 50-500ms
```

**Pourquoi c'est lent** :
- O(N) où N = nombre d'éléments dans le set
- Retourne TOUS les membres en une seule fois
- Consommation mémoire sur le client

**Diagnostic** :

```bash
# Vérifier la taille du set
redis-cli SCARD large_set
# Résultat : 500000

# Estimer la mémoire
redis-cli MEMORY USAGE large_set
# Résultat : 50000000 (50MB)
```

**Solution** :

```bash
# ❌ Éviter
SMEMBERS large_set

# ✅ Utiliser SSCAN
SSCAN large_set 0 COUNT 100

# ✅ Ou limiter avec SRANDMEMBER
SRANDMEMBER large_set 100  # Récupère 100 membres aléatoires
```

### Pattern 3 : HGETALL sur un gros Hash

**Signature SLOWLOG** :
```
1) "HGETALL"
2) "user:12345:profile"
Duration: 20-200ms
```

**Pourquoi c'est lent** :
- O(N) où N = nombre de champs
- Retourne tous les champs d'un coup

**Diagnostic** :

```bash
# Nombre de champs
redis-cli HLEN user:12345:profile
# Résultat : 10000

# Mémoire utilisée
redis-cli MEMORY USAGE user:12345:profile
# Résultat : 5000000 (5MB)
```

**Solution** :

```bash
# ❌ Éviter
HGETALL user:12345:profile

# ✅ Récupérer uniquement les champs nécessaires
HMGET user:12345:profile name email age

# ✅ Si vraiment besoin de tout, utiliser HSCAN
HSCAN user:12345:profile 0 COUNT 100

# ✅ Refactorer le modèle de données
# Au lieu d'un gros hash, utiliser plusieurs petits hash
HMGET user:12345:basic name email
HMGET user:12345:preferences theme language
HMGET user:12345:stats loginCount lastLogin
```

### Pattern 4 : LRANGE avec gros range

**Signature SLOWLOG** :
```
1) "LRANGE"
2) "queue:tasks"
3) "0"
4) "-1"
Duration: 30-300ms
```

**Problème** :
```bash
# ❌ Récupérer toute la liste
LRANGE queue:tasks 0 -1

# La liste contient 100,000 éléments !
LLEN queue:tasks
# Résultat : 100000
```

**Solution** :

```bash
# ✅ Pagination
LRANGE queue:tasks 0 99    # Premiers 100 éléments
LRANGE queue:tasks 100 199 # 100 suivants

# ✅ Pour une queue, utiliser LPOP/RPOP
LPOP queue:tasks 100  # Redis 6.2+ : pop multiple

# ✅ Ou utiliser Redis Streams (meilleure solution)
XREAD COUNT 100 STREAMS mystream 0-0
```

### Pattern 5 : SORT (Tri complexe)

**Signature SLOWLOG** :
```
1) "SORT"
2) "list:items"
3) "BY"
4) "item:*->price"
5) "GET"
6) "item:*->name"
Duration: 100-1000ms
```

**Pourquoi c'est lent** :
- O(N log N) pour le tri
- Accès multiples à d'autres clés
- Pas de cache du résultat

**Solution** :

```bash
# ❌ Éviter SORT complexe
SORT list:items BY item:*->price GET item:*->name

# ✅ Utiliser un Sorted Set à la place
# Insérer les items déjà triés par prix
ZADD items:by_price 19.99 "item:1" 29.99 "item:2"

# Récupérer triés instantanément (O(log N))
ZRANGE items:by_price 0 9 WITHSCORES

# ✅ Pré-calculer et cacher le résultat
# Sortir le tri de Redis si possible
```

### Pattern 6 : SUNION/SINTER/SDIFF sur gros Sets

**Signature SLOWLOG** :
```
1) "SUNION"
2) "set:tags:1"
3) "set:tags:2"
4) "set:tags:3"
Duration: 50-500ms
```

**Problème** :
- O(N) où N = somme des éléments de tous les sets
- Peut créer un très gros résultat temporaire

**Solution** :

```bash
# ❌ Éviter sur de gros sets
SUNION set:tags:1 set:tags:2 set:tags:3

# ✅ Alternative 1 : Limiter le nombre de sets
SUNION set:tags:1 set:tags:2  # Max 2-3 sets

# ✅ Alternative 2 : Utiliser SUNIONSTORE + pagination
SUNIONSTORE result:temp set:tags:1 set:tags:2
SSCAN result:temp 0 COUNT 100
DEL result:temp

# ✅ Alternative 3 : Revoir le modèle de données
# Utiliser des Sorted Sets avec timestamps
ZADD tags:all timestamp1 "tag1" timestamp2 "tag2"
```

---

## 🔬 Techniques avancées de profiling

### 1. Profiling en temps réel avec MONITOR (DANGER)

⚠️ **ATTENTION** : MONITOR a un impact MAJEUR sur les performances.

**Utilisation sécurisée** :

```bash
# ✅ Limiter la durée à quelques secondes maximum
timeout 5 redis-cli MONITOR > monitor_output.txt

# ✅ Rediriger vers un fichier pour analyse offline
redis-cli MONITOR | head -n 10000 > commands.log

# ✅ Filtrer en temps réel
redis-cli MONITOR | grep "KEYS"
```

**Analyse de la sortie MONITOR** :

```bash
# Format de sortie :
# timestamp [database client] "command" "arg1" "arg2"
1702293847.123456 [0 127.0.0.1:43501] "SET" "user:123" "value"
1702293847.234567 [0 127.0.0.1:43502] "GET" "user:123"

# Analyser les patterns
cat monitor_output.txt | awk '{print $4}' | sort | uniq -c | sort -rn | head -20
```

### 2. Analyse avec redis-cli --latency

```bash
# Latency en temps réel
redis-cli --latency

# Latency avec historique
redis-cli --latency-history

# Latency distribution
redis-cli --latency-dist

# Monitoring de latence intrinsèque
redis-cli --intrinsic-latency 60  # Test pendant 60 secondes
```

### 3. Corrélation SLOWLOG + Métriques système

**Script de corrélation** :

```bash
#!/bin/bash
# Capture synchronisée SLOWLOG + métriques système

while true; do
    TIMESTAMP=$(date +%s)

    # SLOWLOG
    SLOW_COUNT=$(redis-cli SLOWLOG LEN)

    # Métriques système
    CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}')
    MEM=$(free | grep Mem | awk '{print ($3/$2) * 100.0}')

    # Redis stats
    OPS=$(redis-cli INFO stats | grep instantaneous_ops_per_sec | cut -d: -f2)

    echo "$TIMESTAMP,$SLOW_COUNT,$CPU,$MEM,$OPS"

    sleep 5
done > metrics_correlation.csv
```

**Analyse** :

```python
import pandas as pd
import matplotlib.pyplot as plt

# Charger les données
df = pd.read_csv('metrics_correlation.csv',
                 names=['timestamp', 'slow_count', 'cpu', 'memory', 'ops'])

# Calculer les corrélations
correlations = df.corr()
print("Correlation avec slow_count:")
print(correlations['slow_count'].sort_values(ascending=False))

# Visualiser
fig, axes = plt.subplots(3, 1, figsize=(12, 10))
axes[0].plot(df['timestamp'], df['slow_count'])
axes[0].set_title('Slow Queries over Time')

axes[1].plot(df['timestamp'], df['cpu'])
axes[1].set_title('CPU Usage')

axes[2].plot(df['timestamp'], df['ops'])
axes[2].set_title('Operations per Second')

plt.tight_layout()
plt.savefig('slowlog_correlation.png')
```

---

## 🎯 Guide de résolution pas à pas

### Scénario 1 : Pic soudain de commandes lentes

**Symptômes** :
- Alertes de latence
- SLOWLOG LEN augmente rapidement
- Applications signalent des timeouts

**Investigation** :

```bash
# Étape 1 : Quick check
redis-cli SLOWLOG GET 10

# Étape 2 : Identifier le pattern
redis-cli SLOWLOG GET 100 | grep -E "^\s+[0-9]+\)" | head -20

# Étape 3 : Vérifier la charge système
redis-cli INFO stats | grep instantaneous_ops_per_sec
top -p $(pgrep redis-server)

# Étape 4 : Identifier le client problématique
redis-cli CLIENT LIST | grep -v "idle=0"
```

**Actions** :

```bash
# Si un client exécute une commande bloquante
redis-cli CLIENT LIST | grep "cmd=KEYS"
# Killer le client si nécessaire
redis-cli CLIENT KILL ID <client-id>

# Si c'est un pattern d'accès
# → Contacter l'équipe de dev pour optimiser
# → Mettre en place un rate limiting temporaire
```

### Scénario 2 : Dégradation progressive des performances

**Symptômes** :
- SLOWLOG se remplit de plus en plus
- Commandes auparavant rapides deviennent lentes
- Fragmentation mémoire élevée

**Investigation** :

```bash
# Étape 1 : Historique du SLOWLOG
redis-cli SLOWLOG GET 500 > slowlog_history.txt

# Étape 2 : Analyse temporelle
# Extraire les timestamps
awk '/integer/ {if (NR % 6 == 2) print $2}' slowlog_history.txt | sort | uniq -c

# Étape 3 : Vérifier les big keys
redis-cli --bigkeys

# Étape 4 : Memory analysis
redis-cli INFO memory
redis-cli MEMORY DOCTOR
```

**Causes possibles** :

1. **Croissance des données**
   ```bash
   # Comparer la taille actuelle avec le baseline
   redis-cli DBSIZE
   redis-cli INFO memory | grep used_memory_human
   ```

2. **Fragmentation mémoire**
   ```bash
   redis-cli INFO memory | grep mem_fragmentation_ratio
   # Si > 1.5 → Considérer un restart ou active defrag
   ```

3. **Big keys accumulées**
   ```bash
   redis-cli --bigkeys --bigkeys-output bigkeys.txt
   # Analyser et nettoyer
   ```

### Scénario 3 : Commande spécifique toujours lente

**Investigation approfondie** :

```bash
# 1. Identifier la commande exacte
redis-cli SLOWLOG GET 100 | grep -A 5 "COMMAND_NAME"

# 2. Tester la commande manuellement
redis-cli --latency-history <<EOF
COMMAND_NAME args
EOF

# 3. Analyser la complexité
redis-cli TYPE key_name
redis-cli OBJECT ENCODING key_name

# 4. Mesurer la taille
redis-cli MEMORY USAGE key_name
redis-cli STRLEN key_name  # Pour strings
redis-cli LLEN key_name    # Pour lists
redis-cli HLEN key_name    # Pour hashes
redis-cli SCARD key_name   # Pour sets
redis-cli ZCARD key_name   # Pour sorted sets
```

**Plan d'action** :

```bash
# Option 1 : Refactoring
# - Diviser les big keys
# - Changer le modèle de données

# Option 2 : Optimisation de la commande
# - SMEMBERS → SSCAN
# - HGETALL → HMGET
# - KEYS → SCAN

# Option 3 : Cache côté application
# - Cacher le résultat si la donnée ne change pas souvent
# - Implémenter un cache local avec TTL
```

---

## 📈 Dashboard et monitoring continu

### Métriques clés à monitorer

```bash
# Script de collecte pour Prometheus/Grafana
#!/bin/bash

# Nombre d'entrées dans le SLOWLOG
SLOWLOG_LEN=$(redis-cli SLOWLOG LEN)

# Durée max de la dernière commande lente
LAST_SLOW_DURATION=$(redis-cli SLOWLOG GET 1 | grep -A 1 "integer) 3" | tail -1 | awk '{print $2}')

# Taux de commandes lentes par seconde
# (nécessite de tracker le delta)

echo "redis_slowlog_length{instance=\"redis1\"} $SLOWLOG_LEN"
echo "redis_slowlog_last_duration_us{instance=\"redis1\"} $LAST_SLOW_DURATION"
```

### Alertes recommandées

```yaml
# Prometheus alerting rules
groups:
  - name: redis_slowlog
    rules:
      - alert: RedisSlowlogGrowing
        expr: increase(redis_slowlog_length[5m]) > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis SLOWLOG growing rapidly"
          description: "{{ $labels.instance }} has logged {{ $value }} slow queries in 5min"

      - alert: RedisSlowlogCritical
        expr: redis_slowlog_last_duration_us > 50000
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis extremely slow query detected"
          description: "{{ $labels.instance }} had a query taking {{ $value }}μs ({{ humanizeDuration $value }})"

      - alert: RedisSlowlogFull
        expr: redis_slowlog_length >= 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis SLOWLOG is full"
          description: "Consider increasing slowlog-max-len or investigating the slow queries"
```

---

## 🎓 Best Practices

### Configuration

✅ **DO**
- Configurer un seuil bas en production (1-5ms) pour détection précoce
- Maintenir un historique suffisant (500-1000 entrées)
- Monitorer le SLOWLOG en continu via Prometheus
- Exporter régulièrement le SLOWLOG pour analyse historique

❌ **DON'T**
- Ne pas configurer un seuil trop bas (< 100μs) sans raison
- Ne pas ignorer le SLOWLOG pendant des semaines
- Ne jamais utiliser `slowlog-log-slower-than -1` (désactivé) en production

### Analyse

✅ **DO**
- Analyser le SLOWLOG quotidiennement
- Corréler avec les métriques système et applicatives
- Documenter les patterns récurrents
- Créer des runbooks pour les problèmes fréquents

❌ **DON'T**
- Ne pas se fier uniquement au SLOWLOG (analyser aussi le contexte)
- Ne pas ignorer les commandes "modérément lentes" (5-10ms)
- Ne pas supposer que toutes les commandes lentes sont problématiques

### Résolution

✅ **DO**
- Tester les solutions en dev/staging avant la production
- Documenter chaque intervention
- Vérifier l'impact des changements
- Impliquer les développeurs dans l'optimisation

❌ **DON'T**
- Ne jamais redémarrer Redis sans comprendre la cause
- Ne pas modifier la configuration sans backup
- Ne pas optimiser prématurément (profiling d'abord!)

---

## 🔗 Checklist de profiling

### Avant de déployer en production

- [ ] SLOWLOG configuré avec un seuil approprié
- [ ] Monitoring du SLOWLOG en place (Grafana)
- [ ] Alertes configurées pour commandes critiques
- [ ] Script d'export du SLOWLOG automatisé
- [ ] Runbooks créés pour les patterns connus
- [ ] Équipe formée à l'interprétation du SLOWLOG

### En cas de problème de performance

- [ ] Consulter le SLOWLOG immédiatement
- [ ] Identifier le pattern de commandes lentes
- [ ] Vérifier les métriques système corrélées
- [ ] Identifier le client/application responsable
- [ ] Tester la reproduction en environnement contrôlé
- [ ] Appliquer le correctif et valider
- [ ] Documenter l'incident (post-mortem)

---

## 📚 Ressources complémentaires

### Documentation officielle
- [Redis SLOWLOG](https://redis.io/commands/slowlog/)
- [Redis Latency Monitoring](https://redis.io/docs/management/optimization/latency/)

### Outils
- **Redis Insight** : Visualisation du SLOWLOG dans l'interface graphique
- **redis-cli --latency** : Tests de latence intégrés
- **Prometheus + Grafana** : Monitoring continu

### Lectures avancées
- "Redis in Action" - Chapter on Performance
- Blog Antirez (créateur de Redis) sur l'optimisation
- Redis Labs blogs sur le profiling

---

## 🎯 Points clés à retenir

1. **Le SLOWLOG est votre premier outil de profiling** - Configurez-le dès le départ
2. **Une commande lente bloque toutes les autres** - L'architecture single-threaded ne pardonne pas
3. **O(N) est l'ennemi** - Évitez KEYS, SMEMBERS, HGETALL sur de grosses données
4. **Profilez avant d'optimiser** - Ne devinez pas, mesurez !
5. **Corrélation = clé du diagnostic** - SLOWLOG + métriques système + logs applicatifs
6. **Monitoring continu > réaction** - Détectez les problèmes avant qu'ils n'impactent les utilisateurs

---

**🚀 Section suivante** : [14.2 - Memory Analysis : --bigkeys, --memkeys et memory doctor](./02-memory-analysis-bigkeys-memkeys.md)

⏭️ [Memory Analysis : --bigkeys, --memkeys et memory doctor](/14-performance-troubleshooting/02-memory-analysis-bigkeys-memkeys.md)
