🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.7 - Fragmentation mémoire : Detection et défragmentation

## 🎯 Objectifs de cette section

- Comprendre la fragmentation mémoire dans Redis
- Calculer et interpréter le ratio de fragmentation
- Identifier les causes de fragmentation excessive
- Maîtriser l'active defragmentation (Redis 4.0+)
- Choisir entre defrag et restart
- Mettre en place un monitoring proactif

---

## 📚 Introduction : La fragmentation mémoire

### Qu'est-ce que la fragmentation ?

La **fragmentation mémoire** survient quand la mémoire physique (RSS) utilisée est significativement supérieure à la mémoire logique (used_memory) des données.

```
┌─────────────────────────────────────────────────┐
│  MÉMOIRE SANS FRAGMENTATION                     │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌────────────────────────────────────┐         │
│  │  Data: 8GB                         │         │
│  │  ████████████████████████████      │         │
│  └────────────────────────────────────┘         │
│                                                 │
│  RSS (physical): 8GB                            │
│  Used (logical): 8GB                            │
│  Ratio: 1.0 ✅ Pas de fragmentation             │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  MÉMOIRE AVEC FRAGMENTATION                     │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌────────────────────────────────────┐         │
│  │  Data: 8GB                         │         │
│  │  ██░░██░░██░░██░░██░░██░░██░░      │         │
│  │  (avec des trous)                  │         │
│  └────────────────────────────────────┘         │
│                                                 │
│  RSS (physical): 12GB                           │
│  Used (logical): 8GB                            │
│  Ratio: 1.5 ⚠️ Fragmentation 50%                │
│  Gaspillage: 4GB                                │
└─────────────────────────────────────────────────┘
```

### Pourquoi la fragmentation se produit ?

**Mécanisme** :

```
1. Allocation initiale (défaut)
   ┌───────────────────────────────────┐
   │ A │ B │ C │ D │ E │ F │ G │ H │   │
   └───────────────────────────────────┘
   Ratio: 1.0

2. Suppression de clés (B, D, F)
   ┌───────────────────────────────────┐
   │ A │   │ C │   │ E │   │ G │ H │   │
   └───────────────────────────────────┘
   Ratio: 1.8 (mémoire non libérée)

3. Nouvelles allocations (plus petites)
   ┌───────────────────────────────────┐
   │ A │ i │ C │ j │ E │ k │ G │ H │   │
   └───────────────────────────────────┘
   Ratio: 1.5 (espaces perdus)
```

**Causes principales** :

| Cause | Description | Impact |
|-------|-------------|--------|
| **Churn élevé** | Beaucoup de SET/DEL | Fort |
| **Taille variable** | Petites puis grosses clés | Moyen |
| **Évictions** | maxmemory-policy active | Fort |
| **Big keys** | Suppressions de gros objets | Très fort |
| **Allocateur** | jemalloc vs libc | Variable |
| **Long running** | Instance sans restart | Progressif |

### Métriques de fragmentation

```bash
# Métriques clés
redis-cli INFO memory

# Valeurs importantes :
used_memory:8589934592           # 8GB (données logiques)
used_memory_rss:12884901888      # 12GB (mémoire physique)
mem_fragmentation_ratio:1.50     # 1.5 = 50% de fragmentation
mem_fragmentation_bytes:4294967296  # 4GB gaspillés
```

**Interprétation du ratio** :

```
Ratio < 1.0   : SWAP (DANGER!)
              └─ Redis utilise le swap

Ratio = 1.0-1.3 : Excellent
                └─ Fragmentation minimale

Ratio = 1.3-1.5 : Acceptable
                └─ Fragmentation modérée

Ratio = 1.5-2.0 : Préoccupant
                └─ Gaspillage significatif

Ratio > 2.0   : Critique
              └─ Action immédiate requise
```

---

## 🔍 Détection de la fragmentation

### Quick check

```bash
# Commande rapide
redis-cli INFO memory | grep -E "used_memory:|used_memory_rss:|mem_fragmentation_ratio:"

# Output :
# used_memory:8589934592
# used_memory_rss:12884901888
# mem_fragmentation_ratio:1.50
```

### Script de monitoring détaillé

```bash
#!/bin/bash
# check-fragmentation.sh

echo "=== REDIS MEMORY FRAGMENTATION CHECK ==="
echo ""

# Extraire les métriques
INFO=$(redis-cli INFO memory)

USED=$(echo "$INFO" | grep "^used_memory:" | cut -d: -f2 | tr -d '\r')
RSS=$(echo "$INFO" | grep "^used_memory_rss:" | cut -d: -f2 | tr -d '\r')
RATIO=$(echo "$INFO" | grep "^mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r')
FRAG_BYTES=$(echo "$INFO" | grep "^mem_fragmentation_bytes:" | cut -d: -f2 | tr -d '\r')

# Convertir en GB
USED_GB=$(echo "scale=2; $USED / 1024 / 1024 / 1024" | bc)
RSS_GB=$(echo "scale=2; $RSS / 1024 / 1024 / 1024" | bc)
FRAG_GB=$(echo "scale=2; $FRAG_BYTES / 1024 / 1024 / 1024" | bc)

# Afficher
echo "Memory Statistics:"
echo "  Used (logical):  ${USED_GB} GB"
echo "  RSS (physical):  ${RSS_GB} GB"
echo "  Fragmentation:   ${FRAG_GB} GB wasted"
echo "  Ratio:           ${RATIO}"
echo ""

# Évaluation
if (( $(echo "$RATIO < 1.0" | bc -l) )); then
    echo "🔴 CRITICAL: Ratio < 1.0 - Redis is swapping!"
    echo "   Action: Increase RAM or reduce maxmemory"
    exit 2
elif (( $(echo "$RATIO > 2.0" | bc -l) )); then
    echo "🔴 CRITICAL: Ratio > 2.0 - High fragmentation"
    echo "   Action: Restart Redis or enable active defrag"
    exit 2
elif (( $(echo "$RATIO > 1.5" | bc -l) )); then
    echo "⚠️  WARNING: Ratio > 1.5 - Moderate fragmentation"
    echo "   Consider: Active defragmentation"
    exit 1
elif (( $(echo "$RATIO > 1.3" | bc -l) )); then
    echo "ℹ️  INFO: Ratio = 1.3-1.5 - Acceptable"
    echo "   Monitor: Keep an eye on the trend"
    exit 0
else
    echo "✅ OK: Ratio = ${RATIO} - No fragmentation"
    exit 0
fi
```

### Monitoring historique

```python
#!/usr/bin/env python3
"""
Monitoring historique de la fragmentation
"""
import redis
import time
import json
from datetime import datetime
from collections import deque

class FragmentationMonitor:
    def __init__(self, host='localhost', port=6379, history_size=288):
        """
        Args:
            history_size: Nombre de points d'historique (288 = 24h à 5min)
        """
        self.r = redis.Redis(host=host, port=port, decode_responses=True)
        self.history = deque(maxlen=history_size)
        self.history_file = 'fragmentation_history.json'
        self.load_history()

    def get_fragmentation_stats(self):
        """Récupère les stats de fragmentation"""
        info = self.r.info('memory')

        return {
            'timestamp': datetime.now().isoformat(),
            'used_memory': info['used_memory'],
            'used_memory_rss': info['used_memory_rss'],
            'mem_fragmentation_ratio': info.get('mem_fragmentation_ratio', 0),
            'mem_fragmentation_bytes': info.get('mem_fragmentation_bytes', 0),
            'allocator_allocated': info.get('allocator_allocated', 0),
            'allocator_active': info.get('allocator_active', 0),
            'allocator_resident': info.get('allocator_resident', 0)
        }

    def analyze_trend(self):
        """Analyse la tendance de fragmentation"""
        if len(self.history) < 12:  # Besoin de 1h de données
            return None

        # Calculer la tendance sur les 12 derniers points (1 heure)
        recent = list(self.history)[-12:]
        ratios = [h['mem_fragmentation_ratio'] for h in recent]

        # Tendance simple : différence entre moyenne récente et ancienne
        mid = len(ratios) // 2
        old_avg = sum(ratios[:mid]) / mid
        new_avg = sum(ratios[mid:]) / (len(ratios) - mid)

        trend = new_avg - old_avg

        return {
            'trend': trend,
            'direction': 'increasing' if trend > 0.05 else 'decreasing' if trend < -0.05 else 'stable',
            'current_ratio': ratios[-1],
            'hour_ago_ratio': ratios[0]
        }

    def predict_critical(self):
        """Prédit quand le ratio atteindra 2.0"""
        trend_data = self.analyze_trend()

        if not trend_data or trend_data['direction'] != 'increasing':
            return None

        current_ratio = trend_data['current_ratio']
        trend = trend_data['trend']

        if current_ratio >= 2.0:
            return {'status': 'critical_now', 'time_to_critical': 0}

        if trend <= 0:
            return None

        # Extrapolation linéaire simple
        time_to_critical_hours = (2.0 - current_ratio) / trend

        return {
            'status': 'warning' if time_to_critical_hours < 24 else 'info',
            'time_to_critical_hours': time_to_critical_hours
        }

    def save_history(self):
        """Sauvegarde l'historique"""
        with open(self.history_file, 'w') as f:
            json.dump(list(self.history), f, indent=2)

    def load_history(self):
        """Charge l'historique"""
        try:
            with open(self.history_file, 'r') as f:
                data = json.load(f)
                self.history.extend(data)
        except FileNotFoundError:
            pass

    def record(self):
        """Enregistre un point de données"""
        stats = self.get_fragmentation_stats()
        self.history.append(stats)
        self.save_history()
        return stats

    def report(self):
        """Génère un rapport détaillé"""
        stats = self.get_fragmentation_stats()
        trend = self.analyze_trend()
        prediction = self.predict_critical()

        print("=" * 60)
        print("FRAGMENTATION REPORT")
        print("=" * 60)
        print(f"Timestamp: {stats['timestamp']}")
        print()

        # Stats actuelles
        used_gb = stats['used_memory'] / 1024 / 1024 / 1024
        rss_gb = stats['used_memory_rss'] / 1024 / 1024 / 1024
        frag_gb = stats['mem_fragmentation_bytes'] / 1024 / 1024 / 1024
        ratio = stats['mem_fragmentation_ratio']

        print("Current Status:")
        print(f"  Used (logical):     {used_gb:.2f} GB")
        print(f"  RSS (physical):     {rss_gb:.2f} GB")
        print(f"  Wasted:             {frag_gb:.2f} GB")
        print(f"  Fragmentation:      {ratio:.2f}")

        # État
        if ratio < 1.0:
            status = "🔴 CRITICAL (SWAPPING)"
        elif ratio > 2.0:
            status = "🔴 CRITICAL"
        elif ratio > 1.5:
            status = "⚠️  WARNING"
        elif ratio > 1.3:
            status = "ℹ️  INFO"
        else:
            status = "✅ OK"

        print(f"  Status:             {status}")
        print()

        # Tendance
        if trend:
            print("Trend Analysis (last hour):")
            print(f"  Direction:          {trend['direction']}")
            print(f"  Change:             {trend['trend']:+.3f}")
            print(f"  Current:            {trend['current_ratio']:.2f}")
            print(f"  1 hour ago:         {trend['hour_ago_ratio']:.2f}")
            print()

        # Prédiction
        if prediction:
            print("Prediction:")
            if prediction['status'] == 'critical_now':
                print("  ⚠️  RATIO ALREADY CRITICAL (>= 2.0)")
            else:
                hours = prediction['time_to_critical_hours']
                print(f"  Time to critical:   {hours:.1f} hours")
                if hours < 24:
                    print("  ⚠️  Will reach 2.0 in less than 24 hours!")
            print()

        # Recommandations
        print("Recommendations:")
        if ratio > 2.0:
            print("  • Immediate action required")
            print("  • Option 1: Restart Redis")
            print("  • Option 2: Enable active defragmentation")
        elif ratio > 1.5:
            print("  • Enable active defragmentation")
            print("  • Monitor closely")
        elif ratio > 1.3:
            print("  • Monitor the trend")
            print("  • Consider enabling active defrag if increasing")
        else:
            print("  • No action needed")

        print("=" * 60)

    def run(self, interval=300):
        """Boucle de monitoring"""
        print("Starting Fragmentation Monitor")
        print(f"Interval: {interval}s")
        print("-" * 60)

        while True:
            try:
                self.record()
                self.report()

                print(f"\nNext check in {interval}s")
                print("-" * 60)

                time.sleep(interval)

            except KeyboardInterrupt:
                print("\nMonitoring stopped")
                break
            except Exception as e:
                print(f"Error: {e}")
                time.sleep(interval)

if __name__ == "__main__":
    monitor = FragmentationMonitor()
    monitor.run(interval=300)  # Check every 5 minutes
```

---

## 🔧 Active Defragmentation (Redis 4.0+)

### Qu'est-ce que l'active defrag ?

**Active defragmentation** : Redis déplace activement les données en mémoire pour éliminer les trous.

```
AVANT active defrag :
┌────────────────────────────────────┐
│ A │   │ C │   │ E │   │ G │ H │    │
└────────────────────────────────────┘

APRÈS active defrag :
┌────────────────────────────────────┐
│ A │ C │ E │ G │ H │                │
└────────────────────────────────────┘
         ↑ Mémoire libérée au système
```

### Configuration de base

```conf
# redis.conf

# Activer active defrag
activedefrag yes

# Seuil minimal de fragmentation pour démarrer
active-defrag-threshold-lower 10
# → Démarre si fragmentation > 10%

# Seuil pour utiliser plus de CPU
active-defrag-threshold-upper 100
# → Mode agressif si fragmentation > 100%

# Mémoire minimum fragmentée pour démarrer (en MB)
active-defrag-ignore-bytes 100mb
# → Ignore si < 100MB fragmentés

# Effort CPU (% du temps)
active-defrag-cycle-min 1
active-defrag-cycle-max 25
# → Utilise 1-25% du CPU

# Effort par passe
active-defrag-max-scan-fields 1000
# → Scan max 1000 fields par passe
```

### Configuration avancée

#### Configuration conservatrice (production sensible)

```conf
# Pour production critique avec latence sensible
activedefrag yes
active-defrag-threshold-lower 20      # Commence à 20%
active-defrag-threshold-upper 100
active-defrag-ignore-bytes 200mb      # Ignore si < 200MB
active-defrag-cycle-min 1             # CPU min : 1%
active-defrag-cycle-max 10            # CPU max : 10%
active-defrag-max-scan-fields 500     # Moins agressif
```

#### Configuration agressive (off-peak)

```conf
# Pour défragmentation rapide (ex: la nuit)
activedefrag yes
active-defrag-threshold-lower 10      # Commence à 10%
active-defrag-threshold-upper 50
active-defrag-ignore-bytes 50mb
active-defrag-cycle-min 10            # CPU min : 10%
active-defrag-cycle-max 50            # CPU max : 50%
active-defrag-max-scan-fields 2000    # Plus agressif
```

### Activation dynamique

```bash
# Activer à la volée
redis-cli CONFIG SET activedefrag yes

# Configurer les seuils
redis-cli CONFIG SET active-defrag-threshold-lower 10
redis-cli CONFIG SET active-defrag-cycle-min 5
redis-cli CONFIG SET active-defrag-cycle-max 25

# Vérifier l'état
redis-cli INFO memory | grep -E "active_defrag"

# Output :
# active_defrag_running:1              # 1 = en cours
# active_defrag_hits:1532              # Réallocations réussies
# active_defrag_misses:45              # Échecs
# active_defrag_key_hits:234           # Clés défragmentées
# active_defrag_key_misses:12          # Échecs
```

### Monitoring de l'active defrag

```bash
#!/bin/bash
# monitor-active-defrag.sh

echo "=== ACTIVE DEFRAGMENTATION MONITOR ==="
echo ""

while true; do
    TIMESTAMP=$(date +"%H:%M:%S")

    # Métriques
    INFO=$(redis-cli INFO memory)

    RATIO=$(echo "$INFO" | grep "^mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r')
    RUNNING=$(echo "$INFO" | grep "^active_defrag_running:" | cut -d: -f2 | tr -d '\r')
    HITS=$(echo "$INFO" | grep "^active_defrag_hits:" | cut -d: -f2 | tr -d '\r')
    MISSES=$(echo "$INFO" | grep "^active_defrag_misses:" | cut -d: -f2 | tr -d '\r')

    # CPU
    CPU=$(redis-cli INFO cpu | grep "^used_cpu_sys:" | cut -d: -f2 | tr -d '\r')

    # Afficher
    printf "[%s] Ratio: %.2f | Running: %d | Hits: %d | Misses: %d | CPU: %.2f\n" \
           "$TIMESTAMP" "$RATIO" "$RUNNING" "$HITS" "$MISSES" "$CPU"

    sleep 5
done
```

### Impact sur les performances

```bash
#!/bin/bash
# test-defrag-impact.sh

echo "=== TESTING ACTIVE DEFRAG IMPACT ==="
echo ""

# 1. Mesurer la latence AVANT
echo "Measuring baseline latency (10s)..."
BEFORE=$(timeout 10 redis-cli --latency-history | tail -1)
echo "Baseline: $BEFORE"
echo ""

# 2. Activer active defrag avec effort modéré
echo "Enabling active defragmentation..."
redis-cli CONFIG SET activedefrag yes
redis-cli CONFIG SET active-defrag-cycle-min 10
redis-cli CONFIG SET active-defrag-cycle-max 25

# Attendre que ça démarre
sleep 5

# 3. Mesurer la latence PENDANT
echo "Measuring latency during defrag (10s)..."
DURING=$(timeout 10 redis-cli --latency-history | tail -1)
echo "During defrag: $DURING"
echo ""

# 4. Attendre la fin (ou 5 minutes max)
echo "Waiting for defrag to complete (max 5 min)..."
TIMEOUT=300
ELAPSED=0

while [ $ELAPSED -lt $TIMEOUT ]; do
    RUNNING=$(redis-cli INFO memory | grep "^active_defrag_running:" | cut -d: -f2 | tr -d '\r')

    if [ "$RUNNING" -eq 0 ]; then
        echo "Defragmentation completed after ${ELAPSED}s"
        break
    fi

    printf "."
    sleep 5
    ELAPSED=$((ELAPSED + 5))
done
echo ""

# 5. Mesurer la latence APRÈS
echo "Measuring latency after defrag (10s)..."
AFTER=$(timeout 10 redis-cli --latency-history | tail -1)
echo "After defrag: $AFTER"
echo ""

# 6. Résultats
echo "=== RESULTS ==="
redis-cli INFO memory | grep -E "mem_fragmentation_ratio|active_defrag_hits"
```

---

## 🔄 Restart vs Active Defrag

### Comparaison

| Critère | Restart | Active Defrag |
|---------|---------|---------------|
| **Downtime** | Quelques secondes | ✅ Aucun |
| **Efficacité** | ✅ 100% | 80-95% |
| **Impact CPU** | Aucun après restart | 5-25% pendant defrag |
| **Impact latence** | Pic au restart | Légère augmentation |
| **Durée** | < 1 minute | 10 minutes - 2 heures |
| **Complexité** | Simple | Configuration à ajuster |
| **Risque** | ⚠️ Clients déconnectés | ✅ Minimal |

### Quand choisir le restart ?

✅ **Utiliser le restart si** :
- Ratio > 2.5 (très haute fragmentation)
- Instance non-critique (dev/staging)
- Fenêtre de maintenance disponible
- Active defrag trop lent
- Réplication configurée (switch vers replica)

❌ **Éviter le restart si** :
- Production 24/7 sans replica
- Pas de fenêtre de maintenance
- Clients ne gèrent pas la reconnexion
- Données volumineuses (long reload)

### Quand choisir l'active defrag ?

✅ **Utiliser active defrag si** :
- Ratio = 1.5-2.0 (fragmentation modérée)
- Production sans downtime possible
- Redis 4.0+ disponible
- CPU disponible (< 80% d'utilisation)
- Patience (plusieurs heures OK)

❌ **Éviter active defrag si** :
- Redis < 4.0
- CPU déjà saturé (> 90%)
- Besoin de résultat immédiat
- Ratio > 2.5 (trop lent)

### Procédure de restart optimisée

```bash
#!/bin/bash
# restart-for-defrag.sh

echo "=== REDIS RESTART FOR DEFRAGMENTATION ==="
echo ""

# Vérifications pré-restart
echo "Pre-restart checks..."

# 1. Ratio actuel
RATIO=$(redis-cli INFO memory | grep "^mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r')
echo "Current fragmentation ratio: $RATIO"

if (( $(echo "$RATIO < 1.5" | bc -l) )); then
    echo "⚠️  Ratio < 1.5, restart not needed"
    read -p "Continue anyway? [y/N]: " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        exit 0
    fi
fi

# 2. Vérifier persistance
echo ""
echo "Checking persistence..."
AOF=$(redis-cli CONFIG GET appendonly | tail -1)

if [ "$AOF" = "yes" ]; then
    echo "AOF is enabled"

    # Force BGREWRITEAOF
    echo "Forcing AOF rewrite..."
    redis-cli BGREWRITEAOF

    # Attendre la fin
    while [ $(redis-cli INFO persistence | grep aof_rewrite_in_progress | cut -d: -f2 | tr -d '\r') -eq 1 ]; do
        echo -n "."
        sleep 1
    done
    echo " Done"
else
    echo "AOF disabled, forcing BGSAVE..."
    redis-cli BGSAVE

    while [ $(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2 | tr -d '\r') -eq 1 ]; do
        echo -n "."
        sleep 1
    done
    echo " Done"
fi

# 3. Métriques avant restart
echo ""
echo "Metrics before restart:"
redis-cli INFO memory | grep -E "used_memory_human|used_memory_rss_human|mem_fragmentation_ratio"

# 4. Confirm restart
echo ""
read -p "Restart Redis now? [y/N]: " -n 1 -r
echo

if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "Restart cancelled"
    exit 0
fi

# 5. Restart
echo ""
echo "Restarting Redis..."
START_TIME=$(date +%s)

systemctl restart redis

# Attendre que Redis soit prêt
sleep 2

# Test de disponibilité
MAX_WAIT=30
WAITED=0

while ! redis-cli PING > /dev/null 2>&1; do
    if [ $WAITED -ge $MAX_WAIT ]; then
        echo "❌ Redis did not start after ${MAX_WAIT}s"
        exit 1
    fi

    echo -n "."
    sleep 1
    WAITED=$((WAITED + 1))
done

END_TIME=$(date +%s)
DOWNTIME=$((END_TIME - START_TIME))

echo ""
echo "✅ Redis restarted (downtime: ${DOWNTIME}s)"

# 6. Métriques après restart
echo ""
echo "Metrics after restart:"
redis-cli INFO memory | grep -E "used_memory_human|used_memory_rss_human|mem_fragmentation_ratio"

# 7. Calculer le gain
NEW_RATIO=$(redis-cli INFO memory | grep "^mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r')

echo ""
echo "Results:"
echo "  Before: $RATIO"
echo "  After:  $NEW_RATIO"

IMPROVEMENT=$(echo "scale=2; ($RATIO - $NEW_RATIO) / $RATIO * 100" | bc)
echo "  Improvement: ${IMPROVEMENT}%"

echo ""
echo "=== RESTART COMPLETE ==="
```

---

## 🎯 Stratégies de prévention

### 1. Éviter les patterns problématiques

#### Pattern anti-fragmentation

```python
# ❌ MAUVAIS : Créer puis supprimer en boucle
for i in range(1000000):
    r.set(f'temp:{i}', 'x' * 10000)
    r.delete(f'temp:{i}')
# → Fragmentation maximale!

# ✅ BON : Réutiliser les mêmes clés
for i in range(1000000):
    key = f'temp:{i % 1000}'  # Cycle sur 1000 clés
    r.set(key, 'x' * 10000)

# ✅ BON : Utiliser des structures de données réutilisables
# Au lieu de SET/DEL constant, utiliser LIST/HASH
r.lpush('queue', data)
r.rpop('queue')
```

#### Taille constante des valeurs

```python
# ❌ MAUVAIS : Tailles très variables
r.set('key1', 'x' * 100)
r.set('key2', 'x' * 100000)
r.set('key3', 'x' * 10)

# ✅ BON : Tailles similaires
# Ou grouper par taille
r.set('small:key1', 'x' * 100)
r.set('small:key2', 'x' * 120)
r.set('large:key1', 'x' * 100000)
```

### 2. Configuration optimale

```conf
# redis.conf - Anti-fragmentation

# 1. Utiliser jemalloc (défaut)
# Bien meilleur que libc malloc pour Redis

# 2. Active defrag préventif
activedefrag yes
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
active-defrag-ignore-bytes 100mb
active-defrag-cycle-min 1
active-defrag-cycle-max 10

# 3. Limiter les évictions (cause de fragmentation)
maxmemory-policy allkeys-lru
# Si possible, augmenter maxmemory pour éviter évictions

# 4. Lazy freeing (libération asynchrone)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes

# 5. Restart programmé hebdomadaire (si acceptable)
# Via cron ou orchestrateur
```

### 3. Monitoring proactif

```yaml
# prometheus-fragmentation-rules.yml
groups:
  - name: redis_fragmentation
    rules:
      # Alerte : Fragmentation modérée
      - alert: RedisFragmentationModerate
        expr: redis_mem_fragmentation_ratio > 1.5
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Redis fragmentation is moderate"
          description: "{{ $labels.instance }} fragmentation: {{ $value }}"

      # Alerte : Fragmentation élevée
      - alert: RedisFragmentationHigh
        expr: redis_mem_fragmentation_ratio > 2.0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Redis fragmentation is HIGH"
          description: "{{ $labels.instance }} fragmentation: {{ $value }} - ACTION REQUIRED"

      # Alerte : Fragmentation en augmentation
      - alert: RedisFragmentationIncreasing
        expr: |
          (
            redis_mem_fragmentation_ratio
            - redis_mem_fragmentation_ratio offset 1h
          ) > 0.2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis fragmentation increasing rapidly"
          description: "{{ $labels.instance }} fragmentation increased by {{ $value }} in 1 hour"

      # Alerte : Gaspillage mémoire significatif
      - alert: RedisFragmentationWaste
        expr: redis_mem_fragmentation_bytes > 1073741824  # 1GB
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis wasting significant memory to fragmentation"
          description: "{{ $labels.instance }} wasting {{ $value | humanize }}B"
```

### 4. Restart planifié

```bash
#!/bin/bash
# weekly-restart.sh (à planifier via cron)

# Cron: 0 3 * * 0 /usr/local/bin/weekly-restart.sh

RATIO=$(redis-cli INFO memory | grep "^mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r')

# Ne restart que si fragmentation > 1.5
if (( $(echo "$RATIO > 1.5" | bc -l) )); then
    echo "Fragmentation at $RATIO, restarting..."

    # BGSAVE avant restart
    redis-cli BGSAVE
    while [ $(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2 | tr -d '\r') -eq 1 ]; do
        sleep 1
    done

    # Restart
    systemctl restart redis

    # Log
    echo "[$(date)] Redis restarted for defragmentation (ratio was $RATIO)" >> /var/log/redis/weekly-restart.log
else
    echo "Fragmentation at $RATIO, no restart needed"
fi
```

---

## 📊 Dashboard et reporting

### Script de rapport mensuel

```python
#!/usr/bin/env python3
"""
Rapport mensuel de fragmentation
"""
import json
from datetime import datetime, timedelta
from statistics import mean, stdev

def generate_monthly_report(history_file='fragmentation_history.json'):
    """Génère un rapport mensuel"""

    # Charger l'historique
    with open(history_file, 'r') as f:
        data = json.load(f)

    # Filtrer le dernier mois
    now = datetime.now()
    one_month_ago = now - timedelta(days=30)

    monthly_data = [
        d for d in data
        if datetime.fromisoformat(d['timestamp']) > one_month_ago
    ]

    if not monthly_data:
        print("No data for the last month")
        return

    # Calculer les statistiques
    ratios = [d['mem_fragmentation_ratio'] for d in monthly_data]

    print("=" * 60)
    print("REDIS FRAGMENTATION - MONTHLY REPORT")
    print("=" * 60)
    print(f"Period: {one_month_ago.date()} to {now.date()}")
    print(f"Data points: {len(monthly_data)}")
    print()

    print("Fragmentation Ratio Statistics:")
    print(f"  Average:    {mean(ratios):.3f}")
    print(f"  Min:        {min(ratios):.3f}")
    print(f"  Max:        {max(ratios):.3f}")
    if len(ratios) > 1:
        print(f"  Std Dev:    {stdev(ratios):.3f}")
    print()

    # Temps passé dans chaque zone
    excellent = sum(1 for r in ratios if r < 1.3)
    acceptable = sum(1 for r in ratios if 1.3 <= r < 1.5)
    warning = sum(1 for r in ratios if 1.5 <= r < 2.0)
    critical = sum(1 for r in ratios if r >= 2.0)

    total = len(ratios)

    print("Time in Each Zone:")
    print(f"  Excellent (<1.3):   {excellent/total*100:5.1f}% ({excellent} points)")
    print(f"  Acceptable (1.3-1.5): {acceptable/total*100:5.1f}% ({acceptable} points)")
    print(f"  Warning (1.5-2.0):  {warning/total*100:5.1f}% ({warning} points)")
    print(f"  Critical (>=2.0):   {critical/total*100:5.1f}% ({critical} points)")
    print()

    # Tendance générale
    if len(ratios) >= 7:
        first_week_avg = mean(ratios[:7])
        last_week_avg = mean(ratios[-7:])
        trend = last_week_avg - first_week_avg

        print("Trend Analysis:")
        print(f"  First week avg:  {first_week_avg:.3f}")
        print(f"  Last week avg:   {last_week_avg:.3f}")
        print(f"  Trend:           {trend:+.3f}")

        if abs(trend) < 0.05:
            trend_text = "Stable ✅"
        elif trend > 0:
            trend_text = "Increasing ⚠️"
        else:
            trend_text = "Decreasing ✅"

        print(f"  Direction:       {trend_text}")
        print()

    # Recommandations
    print("Recommendations:")
    if max(ratios) > 2.0:
        print("  ⚠️  Fragmentation reached critical levels")
        print("     - Review defragmentation strategy")
        print("     - Consider scheduled restarts")
    elif max(ratios) > 1.5:
        print("  ℹ️  Fragmentation occasionally high")
        print("     - Enable active defragmentation")
        print("     - Monitor closely")
    else:
        print("  ✅ Fragmentation under control")
        print("     - Current strategy is working")

    print("=" * 60)

if __name__ == "__main__":
    generate_monthly_report()
```

---

## 🎓 Best Practices

### Configuration recommandée

```conf
# redis.conf - Configuration optimale anti-fragmentation

# Active defrag toujours activé en production
activedefrag yes

# Paramètres conservateurs pour production
active-defrag-threshold-lower 15      # Démarre à 15%
active-defrag-threshold-upper 100
active-defrag-ignore-bytes 100mb      # Ignore si < 100MB gaspillés
active-defrag-cycle-min 1             # CPU min : 1%
active-defrag-cycle-max 15            # CPU max : 15% (conservateur)
active-defrag-max-scan-fields 1000

# Lazy freeing pour réduire la fragmentation
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes

# Allocateur : toujours utiliser jemalloc (défaut)
# Ne pas changer!
```

### Checklist de gestion

**Quotidien** :
- [ ] Vérifier le ratio de fragmentation
- [ ] S'assurer que active defrag fonctionne si besoin
- [ ] Vérifier les alertes

**Hebdomadaire** :
- [ ] Analyser l'historique de fragmentation
- [ ] Vérifier la tendance
- [ ] Considérer un restart si ratio > 1.8

**Mensuel** :
- [ ] Générer un rapport de fragmentation
- [ ] Analyser les patterns problématiques
- [ ] Ajuster la stratégie si nécessaire
- [ ] Revoir la configuration active defrag

### Décision tree

```
Fragmentation détectée
    ↓
Ratio < 1.3 ?
    ├─ OUI → ✅ Rien à faire
    └─ NON ↓

Ratio < 1.5 ?
    ├─ OUI → ℹ️ Monitor
    └─ NON ↓

Ratio < 2.0 ?
    ├─ OUI → Active defrag (si Redis 4.0+)
    │        OU restart si fenêtre maintenance
    └─ NON ↓

Ratio >= 2.0
    └─ 🔴 Action immédiate :
         Option 1: Restart (rapide, downtime)
         Option 2: Active defrag agressif
         Option 3: Failover vers replica + restart
```

---

## 🎯 Points clés à retenir

1. **Fragmentation normale < 1.5** → Au-delà, action nécessaire
2. **Active defrag (Redis 4.0+)** → Solution sans downtime
3. **Restart = 100% efficace** → Mais downtime
4. **Monitoring essentiel** → Détecter tôt, agir tôt
5. **Prévention > Cure** → Éviter les patterns problématiques
6. **jemalloc > libc** → Toujours utiliser jemalloc
7. **Lazy freeing** → Réduit la fragmentation
8. **Ratio > 2.0 = critique** → Action immédiate

---

**🚀 Section suivante** : [14.8 - Tuning et optimisation des commandes](./08-tuning-optimisation-commandes.md)

⏭️ [Tuning et optimisation des commandes](/14-performance-troubleshooting/08-tuning-optimisation-commandes.md)
