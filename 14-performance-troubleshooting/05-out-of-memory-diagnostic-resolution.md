🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.5 - Out of Memory (OOM) : Diagnostic et résolution

## 🎯 Objectifs de cette section

- Comprendre les mécanismes d'OOM dans Redis
- Détecter les situations pré-OOM avant la catastrophe
- Diagnostiquer les causes racines de consommation mémoire
- Résoudre les crises OOM en production
- Mettre en place une stratégie de prévention efficace

---

## 📚 Introduction : L'OOM dans Redis

### Qu'est-ce qu'un OOM ?

**Out Of Memory (OOM)** survient quand Redis ne peut plus allouer de mémoire pour stocker des données ou exécuter des opérations.

```
┌─────────────────────────────────────────────────┐
│  REDIS MEMORY SATURATION                        │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌────────────────────────────────────┐         │
│  │  used_memory (données + overhead)  │ 95%     │
│  ├────────────────────────────────────┤         │
│  │  Buffer disponible                 │ 5%      │
│  └────────────────────────────────────┘         │
│           maxmemory = 10GB                      │
│                                                 │
│  Nouvelle allocation → ÉCHEC                    │
│  ↓                                              │
│  • Évictions (si politique configurée)          │
│  • Erreur OOM (si pas de politique)             │
│  • Crash possible                               │
└─────────────────────────────────────────────────┘
```

### Les 3 types d'OOM dans Redis

| Type | Description | Cause | Impact |
|------|-------------|-------|--------|
| **Soft OOM** | maxmemory atteint | Données croissent | Évictions, performance |
| **Hard OOM** | RAM système épuisée | Pas de limite configurée | Crash Redis |
| **Fork OOM** | Échec du fork | Pas assez de RAM pour CoW | Échec persist |

### Symptômes d'un OOM imminent

**Indicateurs précoces** :
```bash
# 1. Mémoire > 90% de maxmemory
used_memory: 9.5GB
maxmemory: 10GB
→ Ratio: 95% ⚠️

# 2. Évictions qui commencent
evicted_keys: 15420
→ Augmente rapidement ⚠️

# 3. Rejets de connexions
rejected_connections: 150
→ Mémoire insuffisante pour clients ⚠️

# 4. Fork qui échoue
latest_fork_usec: 0
rdb_last_bgsave_status: err
→ BGSAVE impossible 🔴
```

---

## 🔍 Détection précoce : Avant la catastrophe

### Monitoring en temps réel

#### Script de surveillance continue

```python
#!/usr/bin/env python3
"""
Monitoring OOM avec alertes précoces
"""
import redis
import time
import smtplib
from email.mime.text import MIMEText
from datetime import datetime

class OOMMonitor:
    def __init__(self, host='localhost', port=6379,
                 warning_threshold=0.85, critical_threshold=0.95):
        self.r = redis.Redis(host=host, port=port, decode_responses=True)
        self.warning_threshold = warning_threshold
        self.critical_threshold = critical_threshold
        self.alerts_sent = {
            'warning': 0,
            'critical': 0
        }

    def get_memory_stats(self):
        """Récupère les stats mémoire"""
        info = self.r.info('memory')

        used_memory = info['used_memory']
        maxmemory = info.get('maxmemory', 0)

        if maxmemory == 0:
            # Pas de limite configurée → utiliser la RAM système
            # Attention : DANGEREUX en production!
            return None

        ratio = used_memory / maxmemory

        return {
            'used_memory': used_memory,
            'used_memory_human': info['used_memory_human'],
            'maxmemory': maxmemory,
            'maxmemory_human': info.get('maxmemory_human', 'unlimited'),
            'ratio': ratio,
            'evicted_keys': info.get('evicted_keys', 0),
            'mem_fragmentation_ratio': info.get('mem_fragmentation_ratio', 1.0),
            'used_memory_rss': info.get('used_memory_rss', 0)
        }

    def check_thresholds(self, stats):
        """Vérifie les seuils et retourne le niveau d'alerte"""
        if stats is None:
            return 'error', "maxmemory not configured - DANGEROUS!"

        ratio = stats['ratio']

        if ratio >= self.critical_threshold:
            return 'critical', f"Memory usage CRITICAL: {ratio*100:.1f}%"
        elif ratio >= self.warning_threshold:
            return 'warning', f"Memory usage HIGH: {ratio*100:.1f}%"
        else:
            return 'ok', f"Memory usage OK: {ratio*100:.1f}%"

    def get_growth_rate(self, interval=60):
        """Calcule le taux de croissance mémoire"""
        stats1 = self.get_memory_stats()
        if stats1 is None:
            return None

        mem1 = stats1['used_memory']

        time.sleep(interval)

        stats2 = self.get_memory_stats()
        if stats2 is None:
            return None

        mem2 = stats2['used_memory']

        growth = mem2 - mem1
        growth_rate = (growth / interval) * 60  # bytes per minute

        # Estimation temps avant saturation
        remaining = stats2['maxmemory'] - mem2
        if growth_rate > 0:
            time_to_full = remaining / (growth_rate / 60)  # minutes
        else:
            time_to_full = float('inf')

        return {
            'growth_bytes_per_min': growth_rate,
            'growth_mb_per_min': growth_rate / 1024 / 1024,
            'time_to_full_minutes': time_to_full
        }

    def send_alert(self, level, message, details):
        """Envoie une alerte (email, Slack, etc.)"""
        # Implémentation simplifiée
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')

        alert_msg = f"""
[{level.upper()}] Redis OOM Alert
Time: {timestamp}
Message: {message}

Details:
{details}
"""
        print(alert_msg)

        # TODO: Envoyer par email/Slack/PagerDuty
        # self.send_email(alert_msg)
        # self.send_slack(alert_msg)

    def run(self, check_interval=60):
        """Boucle de monitoring"""
        print("Starting OOM Monitor...")
        print(f"Warning threshold: {self.warning_threshold*100}%")
        print(f"Critical threshold: {self.critical_threshold*100}%")
        print(f"Check interval: {check_interval}s")
        print("-" * 60)

        while True:
            try:
                stats = self.get_memory_stats()

                if stats is None:
                    self.send_alert('error',
                                   'maxmemory not configured',
                                   'Redis is running without memory limit - DANGEROUS!')
                    time.sleep(check_interval)
                    continue

                level, message = self.check_thresholds(stats)

                # Afficher les stats
                timestamp = datetime.now().strftime('%H:%M:%S')
                print(f"[{timestamp}] Memory: {stats['used_memory_human']} / "
                      f"{stats['maxmemory_human']} ({stats['ratio']*100:.1f}%) - {level.upper()}")

                # Alerter si nécessaire
                if level == 'critical':
                    if self.alerts_sent['critical'] == 0:
                        details = f"""
Used: {stats['used_memory_human']}
Max: {stats['maxmemory_human']}
Ratio: {stats['ratio']*100:.1f}%
Evicted keys: {stats['evicted_keys']}
Fragmentation: {stats['mem_fragmentation_ratio']:.2f}
"""
                        self.send_alert('critical', message, details)
                        self.alerts_sent['critical'] += 1

                elif level == 'warning':
                    if self.alerts_sent['warning'] == 0:
                        # Calculer le taux de croissance
                        print("Calculating growth rate...")
                        growth = self.get_growth_rate(interval=60)

                        details = f"""
Used: {stats['used_memory_human']}
Max: {stats['maxmemory_human']}
Ratio: {stats['ratio']*100:.1f}%
Evicted keys: {stats['evicted_keys']}
"""
                        if growth:
                            details += f"""
Growth rate: {growth['growth_mb_per_min']:.2f} MB/min
Time to saturation: {growth['time_to_full_minutes']:.0f} minutes
"""

                        self.send_alert('warning', message, details)
                        self.alerts_sent['warning'] += 1

                else:  # OK
                    # Reset alert counters
                    self.alerts_sent = {'warning': 0, 'critical': 0}

                time.sleep(check_interval)

            except KeyboardInterrupt:
                print("\nMonitoring stopped.")
                break
            except Exception as e:
                print(f"Error: {e}")
                time.sleep(check_interval)

if __name__ == "__main__":
    monitor = OOMMonitor(
        warning_threshold=0.85,   # Alerte à 85%
        critical_threshold=0.95   # Critique à 95%
    )
    monitor.run(check_interval=60)
```

### Métriques à surveiller

```bash
# Script de vérification rapide
#!/bin/bash
# check-oom-risk.sh

echo "=== OOM RISK CHECK ==="
echo ""

# 1. Ratio mémoire
USED=$(redis-cli INFO memory | grep "^used_memory:" | cut -d: -f2 | tr -d '\r')
MAX=$(redis-cli INFO memory | grep "^maxmemory:" | cut -d: -f2 | tr -d '\r')

if [ "$MAX" -eq 0 ]; then
    echo "❌ CRITICAL: maxmemory not configured!"
    echo "   Redis can consume ALL system memory"
    echo ""
else
    RATIO=$(echo "scale=2; $USED / $MAX * 100" | bc)
    echo "Memory usage: $RATIO%"

    if (( $(echo "$RATIO > 95" | bc -l) )); then
        echo "🔴 CRITICAL: Memory > 95%"
    elif (( $(echo "$RATIO > 85" | bc -l) )); then
        echo "⚠️  WARNING: Memory > 85%"
    else
        echo "✅ OK: Memory < 85%"
    fi
    echo ""
fi

# 2. Évictions
EVICTED=$(redis-cli INFO stats | grep "^evicted_keys:" | cut -d: -f2 | tr -d '\r')
echo "Evicted keys: $EVICTED"

if [ "$EVICTED" -gt 0 ]; then
    echo "⚠️  Keys are being evicted - memory pressure"
else
    echo "✅ No evictions"
fi
echo ""

# 3. Rejected connections
REJECTED=$(redis-cli INFO stats | grep "^rejected_connections:" | cut -d: -f2 | tr -d '\r')
echo "Rejected connections: $REJECTED"

if [ "$REJECTED" -gt 0 ]; then
    echo "🔴 CRITICAL: Connections being rejected"
else
    echo "✅ No rejected connections"
fi
echo ""

# 4. Fork status
FORK_STATUS=$(redis-cli INFO persistence | grep "^rdb_last_bgsave_status:" | cut -d: -f2 | tr -d '\r')
echo "Last BGSAVE: $FORK_STATUS"

if [ "$FORK_STATUS" = "ok" ]; then
    echo "✅ Persistence working"
else
    echo "🔴 CRITICAL: BGSAVE failing - may be OOM related"
fi
echo ""

# 5. Fragmentation
FRAG=$(redis-cli INFO memory | grep "^mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r')
echo "Fragmentation ratio: $FRAG"

if (( $(echo "$FRAG > 1.5" | bc -l) )); then
    echo "⚠️  High fragmentation - wasting memory"
else
    echo "✅ Fragmentation OK"
fi
```

---

## 🚨 Diagnostic d'une crise OOM

### Étape 1 : Identifier le type d'OOM

```bash
#!/bin/bash
# diagnose-oom.sh

echo "=== OOM DIAGNOSTIC ==="
echo ""

# Vérifier les logs Redis
echo "--- Redis Logs (last 50 lines) ---"
tail -50 /var/log/redis/redis-server.log | grep -i "oom\|memory\|malloc\|fork"
echo ""

# Vérifier les logs système (OOM killer)
echo "--- System OOM Killer ---"
dmesg | grep -i "redis" | grep -i "killed\|oom" | tail -20
echo ""

# État actuel de la mémoire
echo "--- Current Memory State ---"
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio|evicted_keys"
echo ""

# Dernière erreur BGSAVE
echo "--- BGSAVE Status ---"
redis-cli INFO persistence | grep -E "rdb_last_bgsave_status|rdb_last_bgsave_time_sec|aof_last_bgrewrite_status"
echo ""

# Clients connectés
echo "--- Connected Clients ---"
CLIENTS=$(redis-cli CLIENT LIST | wc -l)
echo "Total clients: $CLIENTS"
echo ""

# Mémoire système
echo "--- System Memory ---"
free -h
echo ""
```

### Étape 2 : Analyse des causes

#### Cause 1 : Croissance naturelle des données

**Diagnostic** :
```bash
# Vérifier l'historique de croissance
redis-cli INFO memory | grep used_memory_peak_human

# Comparer avec la limite
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"
```

**Solution** : Augmenter maxmemory ou optimiser le modèle de données.

#### Cause 2 : Memory leak applicatif

**Diagnostic** :
```bash
# Vérifier le nombre de clés dans le temps
redis-cli DBSIZE

# Si DBSIZE n'augmente pas mais used_memory oui → leak possible

# Identifier les patterns de clés qui grossissent
redis-cli --bigkeys -i 0.1
```

**Solution** : Identifier et corriger le leak dans l'application.

#### Cause 3 : Big keys accumulées

**Diagnostic** :
```bash
# Scanner les big keys
redis-cli --bigkeys > bigkeys_report.txt

# Analyser les plus grosses
grep "Biggest" bigkeys_report.txt
```

**Solution** : Fragmenter ou supprimer les big keys.

#### Cause 4 : Fragmentation mémoire excessive

**Diagnostic** :
```bash
redis-cli INFO memory | grep mem_fragmentation_ratio
# Si ratio > 1.5 → fragmentation problématique

# Calculer la mémoire gaspillée
USED=$(redis-cli INFO memory | grep "^used_memory:" | cut -d: -f2)
RSS=$(redis-cli INFO memory | grep "^used_memory_rss:" | cut -d: -f2)
WASTED=$((RSS - USED))
echo "Memory wasted by fragmentation: $((WASTED / 1024 / 1024)) MB"
```

**Solution** : Redémarrer Redis ou activer active defrag.

#### Cause 5 : Pas de politique d'éviction configurée

**Diagnostic** :
```bash
redis-cli CONFIG GET maxmemory-policy
# 1) "maxmemory-policy"
# 2) "noeviction"  ← PROBLÈME si données croissent

redis-cli INFO stats | grep evicted_keys
# evicted_keys:0  ← Confirme qu'aucune éviction
```

**Solution** : Configurer une politique d'éviction appropriée.

---

## 🛠️ Résolution immédiate (mode crise)

### Scénario 1 : Redis encore opérationnel (Soft OOM)

#### Actions immédiates (< 5 minutes)

```bash
#!/bin/bash
# emergency-oom-response.sh

echo "=== EMERGENCY OOM RESPONSE ==="
echo ""

# 1. Augmenter temporairement maxmemory
CURRENT_MAX=$(redis-cli CONFIG GET maxmemory | tail -1)
echo "Current maxmemory: $CURRENT_MAX"

if [ "$CURRENT_MAX" != "0" ]; then
    # Augmenter de 20%
    NEW_MAX=$(echo "$CURRENT_MAX * 1.2 / 1" | bc)
    echo "Increasing maxmemory to: $NEW_MAX bytes"
    redis-cli CONFIG SET maxmemory $NEW_MAX

    # Vérifier la RAM système disponible
    echo ""
    echo "System memory:"
    free -h | grep Mem
    echo ""
    echo "⚠️  WARNING: Check if system has enough free RAM!"
fi

# 2. Activer/ajuster la politique d'éviction
POLICY=$(redis-cli CONFIG GET maxmemory-policy | tail -1)
echo "Current policy: $POLICY"

if [ "$POLICY" = "noeviction" ]; then
    echo "Enabling LRU eviction policy..."
    redis-cli CONFIG SET maxmemory-policy allkeys-lru
    echo "✅ Eviction enabled"
fi

# 3. Identifier et supprimer les big keys temporaires
echo ""
echo "Scanning for big temporary keys..."
redis-cli --scan --pattern "temp:*" | head -100 | while read key; do
    SIZE=$(redis-cli MEMORY USAGE "$key" 2>/dev/null)
    if [ -n "$SIZE" ] && [ "$SIZE" -gt 1048576 ]; then  # > 1MB
        echo "Deleting: $key ($((SIZE / 1024 / 1024)) MB)"
        redis-cli UNLINK "$key"
    fi
done

echo ""
echo "Scanning for big cache keys..."
redis-cli --scan --pattern "cache:*" | head -100 | while read key; do
    # Supprimer les caches sans TTL
    TTL=$(redis-cli TTL "$key")
    if [ "$TTL" -eq -1 ]; then  # Pas de TTL
        SIZE=$(redis-cli MEMORY USAGE "$key" 2>/dev/null)
        if [ -n "$SIZE" ] && [ "$SIZE" -gt 1048576 ]; then
            echo "Deleting cache without TTL: $key"
            redis-cli UNLINK "$key"
        fi
    fi
done

# 4. Forcer les évictions si nécessaire
echo ""
echo "Forcing immediate evictions by setting keys..."
for i in {1..100}; do
    redis-cli SET "eviction_trigger_$i" "dummy" EX 1 > /dev/null 2>&1
done
redis-cli DEL eviction_trigger_* > /dev/null 2>&1

echo ""
echo "=== RESPONSE COMPLETE ==="
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|evicted_keys"
```

#### Validation post-action

```bash
# Vérifier l'état après intervention
redis-cli INFO memory | grep -E "used_memory:|maxmemory:|mem_fragmentation_ratio"

# Vérifier les évictions
redis-cli INFO stats | grep evicted_keys

# Tester qu'on peut écrire
redis-cli SET test_write "ok" EX 10
redis-cli GET test_write
redis-cli DEL test_write
```

### Scénario 2 : Redis crashé (Hard OOM)

#### Diagnostic post-mortem

```bash
#!/bin/bash
# postmortem-oom.sh

echo "=== OOM POST-MORTEM ANALYSIS ==="
echo ""

# 1. Vérifier si OOM killer a agi
echo "--- OOM Killer Events ---"
dmesg | grep -i "redis.*killed" | tail -20
journalctl -u redis | grep -i "killed\|oom" | tail -20

# 2. Vérifier les derniers logs Redis
echo ""
echo "--- Last Redis Logs ---"
tail -100 /var/log/redis/redis-server.log | grep -A 5 -B 5 -i "oom\|memory\|crash"

# 3. Analyser le dump RDB (si disponible)
echo ""
echo "--- RDB File Info ---"
if [ -f /var/lib/redis/dump.rdb ]; then
    ls -lh /var/lib/redis/dump.rdb

    # Utiliser redis-rdb-tools si disponible
    if command -v rdb &> /dev/null; then
        echo "Analyzing RDB content..."
        rdb -c memory /var/lib/redis/dump.rdb | head -50
    fi
else
    echo "No RDB file found"
fi

# 4. Configuration au moment du crash
echo ""
echo "--- Redis Configuration ---"
grep -E "maxmemory|save|appendonly" /etc/redis/redis.conf | grep -v "^#"
```

#### Récupération

```bash
#!/bin/bash
# recover-from-oom.sh

echo "=== RECOVERING FROM OOM CRASH ==="
echo ""

# 1. Vérifier la RAM système disponible
echo "--- System Memory ---"
free -h

AVAILABLE_MB=$(free -m | awk 'NR==2{print $7}')
echo "Available: ${AVAILABLE_MB}MB"

if [ "$AVAILABLE_MB" -lt 1024 ]; then
    echo "❌ ERROR: Less than 1GB available - cannot safely start Redis"
    exit 1
fi

# 2. Ajuster maxmemory avant de redémarrer
echo ""
echo "Configuring safe maxmemory limit..."

# Calculer un maxmemory sécurisé (80% de la RAM disponible)
SAFE_MEMORY=$((AVAILABLE_MB * 1024 * 1024 * 80 / 100))

# Éditer redis.conf
cp /etc/redis/redis.conf /etc/redis/redis.conf.backup
sed -i "s/^maxmemory .*/maxmemory ${SAFE_MEMORY}/" /etc/redis/redis.conf

# Activer l'éviction
if ! grep -q "^maxmemory-policy" /etc/redis/redis.conf; then
    echo "maxmemory-policy allkeys-lru" >> /etc/redis/redis.conf
else
    sed -i "s/^maxmemory-policy .*/maxmemory-policy allkeys-lru/" /etc/redis/redis.conf
fi

echo "maxmemory set to: $((SAFE_MEMORY / 1024 / 1024))MB"
echo "maxmemory-policy: allkeys-lru"

# 3. Désactiver THP
echo ""
echo "Disabling THP..."
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 4. Configurer overcommit
echo "Setting overcommit_memory=1..."
sysctl vm.overcommit_memory=1

# 5. Tenter de démarrer Redis
echo ""
echo "Starting Redis..."
systemctl start redis

# Attendre le démarrage
sleep 3

# Vérifier l'état
if systemctl is-active --quiet redis; then
    echo "✅ Redis started successfully"

    # Vérifier la mémoire
    redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"
else
    echo "❌ Failed to start Redis"
    echo "Check logs: journalctl -u redis -n 50"
    exit 1
fi

echo ""
echo "=== RECOVERY COMPLETE ==="
```

---

## 🔧 Solutions long terme

### Solution 1 : Dimensionnement approprié

#### Calculer le maxmemory optimal

```python
#!/usr/bin/env python3
"""
Calculateur de maxmemory optimal
"""

def calculate_maxmemory(total_ram_gb,
                        use_case='cache',
                        has_persistence=True,
                        replica_count=0):
    """
    Calcule le maxmemory recommandé

    Args:
        total_ram_gb: RAM totale du serveur (GB)
        use_case: 'cache', 'database', 'mixed'
        has_persistence: True si RDB/AOF activé
        replica_count: Nombre de replicas (pour le master)
    """

    # Réserver pour l'OS et autres processus
    os_reserved_gb = min(2, total_ram_gb * 0.1)  # 10% ou 2GB max

    # Overhead de persistance (fork nécessite potentiellement 2x)
    if has_persistence:
        persistence_overhead = 1.0  # 100% supplémentaire théorique
    else:
        persistence_overhead = 0.0

    # Overhead de réplication
    replication_overhead = replica_count * 0.1  # 10% par replica

    # Calcul selon le use case
    if use_case == 'cache':
        # Cache : on peut utiliser plus de RAM (pas critique si perte)
        base_ratio = 0.75
    elif use_case == 'database':
        # Database : plus conservateur pour la persistance
        base_ratio = 0.60
    else:  # mixed
        base_ratio = 0.70

    # Ajuster pour la persistance
    if has_persistence:
        # Réduire pour laisser de la place au fork
        available_for_redis = (total_ram_gb - os_reserved_gb) / (1 + persistence_overhead)
    else:
        available_for_redis = total_ram_gb - os_reserved_gb

    # Maxmemory final
    maxmemory_gb = available_for_redis * base_ratio * (1 - replication_overhead)

    # Rapport
    print("=" * 60)
    print("MAXMEMORY CALCULATION")
    print("=" * 60)
    print(f"Total RAM: {total_ram_gb} GB")
    print(f"OS Reserved: {os_reserved_gb} GB")
    print(f"Use case: {use_case}")
    print(f"Persistence: {'Yes' if has_persistence else 'No'}")
    print(f"Replicas: {replica_count}")
    print("-" * 60)
    print(f"Recommended maxmemory: {maxmemory_gb:.2f} GB ({maxmemory_gb*1024:.0f} MB)")
    print(f"  = {int(maxmemory_gb * 1024 * 1024 * 1024)} bytes")
    print("-" * 60)

    # Warnings
    if maxmemory_gb / total_ram_gb > 0.8:
        print("⚠️  WARNING: maxmemory is > 80% of total RAM")
        print("   Consider increasing RAM or reducing maxmemory")

    if has_persistence and persistence_overhead > 0:
        print(f"ℹ️  With persistence, fork may need up to {available_for_redis * 2:.2f} GB")

    print("=" * 60)

    return maxmemory_gb * 1024 * 1024 * 1024  # Return en bytes

# Exemples
print("\nExample 1: 16GB server, cache use case, no persistence")
calculate_maxmemory(16, use_case='cache', has_persistence=False)

print("\nExample 2: 32GB server, database use case, with persistence and 2 replicas")
calculate_maxmemory(32, use_case='database', has_persistence=True, replica_count=2)

print("\nExample 3: 64GB server, mixed use case, with persistence")
calculate_maxmemory(64, use_case='mixed', has_persistence=True)
```

### Solution 2 : Politique d'éviction appropriée

#### Choisir la bonne politique

```
┌──────────────────────────────────────────────────┐
│  ÉVICTION POLICIES                               │
├──────────────────────────────────────────────────┤
│                                                  │
│  noeviction ❌                                   │
│  ├─ Pas d'éviction                               │
│  ├─ Erreur OOM si maxmemory atteint              │
│  └─ Use case : AUCUN (dangereux)                 │
│                                                  │
│  allkeys-lru ✅ (Cache général)                  │
│  ├─ Éviction LRU sur TOUTES les clés             │
│  ├─ Les moins récemment utilisées partent        │
│  └─ Use case : Cache pur                         │
│                                                  │
│  allkeys-lfu ✅ (Cache intelligent)              │
│  ├─ Éviction LFU (Least Frequently Used)         │
│  ├─ Les moins fréquemment utilisées partent      │
│  └─ Use case : Cache avec hot keys               │
│                                                  │
│  volatile-lru ✅ (Mix cache/data)                │
│  ├─ Éviction LRU seulement sur clés avec TTL     │
│  ├─ Préserve les données sans TTL                │
│  └─ Use case : Mix cache temporaire + données    │
│                                                  │
│  volatile-lfu ⚠️                                 │
│  ├─ Éviction LFU seulement sur clés avec TTL     │
│  └─ Use case : Rare                              │
│                                                  │
│  allkeys-random ⚠️                               │
│  ├─ Éviction aléatoire sur toutes les clés       │
│  └─ Use case : Quand LRU/LFU inapplicable        │
│                                                  │
│  volatile-random ⚠️                              │
│  ├─ Éviction aléatoire sur clés avec TTL         │
│  └─ Use case : Rare                              │
│                                                  │
│  volatile-ttl ✅ (Expiration prioritaire)        │
│  ├─ Éviction des clés avec TTL le plus court     │
│  ├─ Accélère les expirations naturelles          │
│  └─ Use case : Sessions, données temporaires     │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Configuration** :

```bash
# Pour un cache pur
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Pour un mix cache + données persistantes
redis-cli CONFIG SET maxmemory-policy volatile-lru

# Pour des sessions temporaires
redis-cli CONFIG SET maxmemory-policy volatile-ttl

# Dans redis.conf (permanent)
maxmemory-policy allkeys-lru
```

#### Test de la politique

```python
#!/usr/bin/env python3
"""
Tester le comportement de la politique d'éviction
"""
import redis
import time

def test_eviction_policy(r, policy='allkeys-lru'):
    """Teste une politique d'éviction"""

    print(f"\n=== Testing policy: {policy} ===\n")

    # Configurer
    r.config_set('maxmemory', '10mb')
    r.config_set('maxmemory-policy', policy)

    # Vider
    r.flushdb()

    # Insérer des données jusqu'à atteindre la limite
    print("Inserting data...")
    i = 0
    evictions_before = int(r.info('stats')['evicted_keys'])

    try:
        while True:
            # Données avec et sans TTL
            if i % 2 == 0:
                # Avec TTL
                r.setex(f'key_ttl_{i}', 3600, 'x' * 1000)
            else:
                # Sans TTL
                r.set(f'key_notl_{i}', 'x' * 1000)

            i += 1

            # Vérifier toutes les 100 insertions
            if i % 100 == 0:
                evictions = int(r.info('stats')['evicted_keys'])
                if evictions > evictions_before:
                    print(f"Evictions started after {i} keys")
                    evictions_before = evictions
                    break
    except redis.exceptions.ResponseError as e:
        print(f"Error after {i} keys: {e}")

    # Statistiques
    info = r.info('stats')
    memory = r.info('memory')

    print(f"\nInserted: {i} keys")
    print(f"Current keys: {r.dbsize()}")
    print(f"Evicted: {info['evicted_keys']}")
    print(f"Memory: {memory['used_memory_human']}")

    # Vérifier quel type de clés a été évicté
    keys_ttl = len(r.keys('key_ttl_*'))
    keys_notl = len(r.keys('key_notl_*'))

    print(f"\nRemaining keys with TTL: {keys_ttl}")
    print(f"Remaining keys without TTL: {keys_notl}")

    if keys_ttl == 0 and keys_notl > 0:
        print("→ Policy evicted TTL keys first")
    elif keys_notl == 0 and keys_ttl > 0:
        print("→ Policy evicted non-TTL keys first")
    else:
        print("→ Policy evicted both types")

# Tests
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

test_eviction_policy(r, 'allkeys-lru')
test_eviction_policy(r, 'volatile-lru')
test_eviction_policy(r, 'volatile-ttl')
```

### Solution 3 : Optimisation du modèle de données

#### Audit et optimisation

```python
#!/usr/bin/env python3
"""
Audit du modèle de données pour optimisation mémoire
"""
import redis
from collections import defaultdict

def audit_data_model(r):
    """Audit complet du modèle de données"""

    print("=== DATA MODEL AUDIT ===\n")

    # 1. Distribution par type de données
    print("--- Data Types Distribution ---")
    types = defaultdict(lambda: {'count': 0, 'total_memory': 0})

    cursor = 0
    scanned = 0

    while True:
        cursor, keys = r.scan(cursor, count=100)

        for key in keys:
            key_type = r.type(key)
            memory = r.memory_usage(key) or 0

            types[key_type]['count'] += 1
            types[key_type]['total_memory'] += memory

        scanned += len(keys)
        if scanned % 10000 == 0:
            print(f"Scanned {scanned} keys...")

        if cursor == 0:
            break

    print(f"\nTotal keys scanned: {scanned}\n")
    print(f"{'Type':<15} {'Count':<10} {'Total Memory':<20} {'Avg Memory'}")
    print("-" * 70)

    for dtype, stats in sorted(types.items(), key=lambda x: x[1]['total_memory'], reverse=True):
        count = stats['count']
        total_mb = stats['total_memory'] / 1024 / 1024
        avg_bytes = stats['total_memory'] / count if count > 0 else 0
        print(f"{dtype:<15} {count:<10} {total_mb:>10.2f} MB        {avg_bytes:>10.0f} bytes")

    # 2. Distribution par pattern de clés
    print("\n--- Key Patterns Distribution ---")
    patterns = defaultdict(lambda: {'count': 0, 'total_memory': 0})

    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, count=100)

        for key in keys:
            # Extraire le pattern (préfixe avant ':')
            if ':' in key:
                pattern = key.split(':')[0] + ':*'
            else:
                pattern = 'no-prefix'

            memory = r.memory_usage(key) or 0
            patterns[pattern]['count'] += 1
            patterns[pattern]['total_memory'] += memory

        if cursor == 0:
            break

    print(f"\n{'Pattern':<30} {'Count':<10} {'Total Memory':<20} {'Avg Memory'}")
    print("-" * 80)

    for pattern, stats in sorted(patterns.items(), key=lambda x: x[1]['total_memory'], reverse=True)[:20]:
        count = stats['count']
        total_mb = stats['total_memory'] / 1024 / 1024
        avg_bytes = stats['total_memory'] / count if count > 0 else 0
        print(f"{pattern:<30} {count:<10} {total_mb:>10.2f} MB        {avg_bytes:>10.0f} bytes")

    # 3. Clés sans TTL
    print("\n--- Keys without TTL ---")
    no_ttl_count = 0
    no_ttl_memory = 0

    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, count=100)

        for key in keys:
            if r.ttl(key) == -1:  # Pas de TTL
                no_ttl_count += 1
                no_ttl_memory += r.memory_usage(key) or 0

        if cursor == 0:
            break

    print(f"Keys without TTL: {no_ttl_count}")
    print(f"Memory used: {no_ttl_memory / 1024 / 1024:.2f} MB")
    print(f"Percentage: {no_ttl_count / scanned * 100:.1f}% of total keys")

    # 4. Recommandations
    print("\n--- RECOMMENDATIONS ---\n")

    # Check types inefficaces
    if types['string']['count'] > 0:
        avg_string_size = types['string']['total_memory'] / types['string']['count']
        if avg_string_size > 10240:  # > 10KB
            print("⚠️  String keys are large on average (>10KB)")
            print("   Consider using hashes for structured data")

    # Check clés sans TTL
    if no_ttl_count / scanned > 0.5:
        print("⚠️  More than 50% of keys have no TTL")
        print("   Consider setting TTL on temporary/cache data")

    # Check patterns avec beaucoup de clés
    largest_pattern = max(patterns.items(), key=lambda x: x[1]['count'])
    if largest_pattern[1]['count'] / scanned > 0.3:
        print(f"⚠️  Pattern '{largest_pattern[0]}' represents {largest_pattern[1]['count'] / scanned * 100:.1f}% of keys")
        print("   Consider if this data structure is optimal")

if __name__ == "__main__":
    r = redis.Redis(host='localhost', port=6379)
    audit_data_model(r)
```

### Solution 4 : Sharding (partitionnement)

Quand une instance ne suffit plus :

```
┌────────────────────────────────────────────────┐
│  SINGLE INSTANCE (before)                      │
│  ┌──────────────────────────────────────┐      │
│  │  Redis                               │      │
│  │  maxmemory: 100GB                    │      │
│  │  50M keys                            │      │
│  └──────────────────────────────────────┘      │
└────────────────────────────────────────────────┘
                    ↓
         SHARDING (after)
┌────────────────────────────────────────────────┐
│  ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │ Redis #1 │ │ Redis #2 │ │ Redis #3 │        │
│  │ 35GB     │ │ 35GB     │ │ 35GB     │        │
│  │ 17M keys │ │ 16M keys │ │ 17M keys │        │
│  └──────────┘ └──────────┘ └──────────┘        │
│                                                │
│  Total: 105GB capacity, 50M keys               │
│  + Room for growth in each instance            │
└────────────────────────────────────────────────┘
```

**Options de sharding** :
1. **Application-side sharding** : Logique dans l'app
2. **Proxy-based** : Twemproxy, Envoy
3. **Redis Cluster** : Sharding natif Redis

---

## 📊 Monitoring et alertes

### Métriques Prometheus

```yaml
# prometheus-oom-rules.yml
groups:
  - name: redis_oom
    rules:
      # Alerte : Mémoire > 85%
      - alert: RedisMemoryHigh
        expr: (redis_memory_used_bytes / redis_memory_max_bytes) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage is high"
          description: "{{ $labels.instance }} is using {{ $value | humanizePercentage }} of maxmemory"

      # Alerte critique : Mémoire > 95%
      - alert: RedisMemoryCritical
        expr: (redis_memory_used_bytes / redis_memory_max_bytes) > 0.95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis memory usage is CRITICAL"
          description: "{{ $labels.instance }} is using {{ $value | humanizePercentage }} - OOM imminent!"

      # Alerte : Évictions actives
      - alert: RedisEvictingKeys
        expr: rate(redis_evicted_keys_total[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis is evicting keys"
          description: "{{ $labels.instance }} is evicting {{ $value }} keys/sec - memory pressure"

      # Alerte : Maxmemory non configuré
      - alert: RedisNoMaxmemory
        expr: redis_memory_max_bytes == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Redis maxmemory not configured"
          description: "{{ $labels.instance }} has no memory limit - DANGEROUS!"

      # Alerte : BGSAVE échoue
      - alert: RedisBGSaveFailing
        expr: redis_rdb_last_bgsave_status == 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Redis BGSAVE failing"
          description: "{{ $labels.instance }} cannot save - may be OOM related"

      # Alerte : Croissance rapide
      - alert: RedisMemoryGrowthFast
        expr: rate(redis_memory_used_bytes[1h]) > 104857600  # 100MB/hour
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "Redis memory growing rapidly"
          description: "{{ $labels.instance }} is growing at {{ $value | humanize }}B/sec"
```

### Dashboard Grafana

Panels essentiels :

```
1. Memory Usage Gauge
   - used_memory / maxmemory
   - Seuils: 85% warning, 95% critical

2. Memory Timeline
   - used_memory over time
   - maxmemory line
   - Évictions overlay

3. Eviction Rate
   - evicted_keys rate per second
   - Alerte si > 0

4. Memory by Type
   - Pie chart: data vs overhead vs buffers

5. Growth Rate
   - Memory increase rate (MB/hour)
   - Projection to maxmemory

6. Time to Full
   - Estimation based on current growth
   - "X hours until OOM"
```

---

## 🎓 Best Practices

### Configuration production

```conf
# redis.conf - Configuration anti-OOM

# 1. TOUJOURS définir maxmemory
maxmemory 8gb

# 2. TOUJOURS avoir une politique d'éviction
maxmemory-policy allkeys-lru

# 3. Samples pour LRU (plus = plus précis)
maxmemory-samples 10

# 4. Politique pendant éviction
maxmemory-eviction-tenacity 10

# 5. Lazy freeing (évite les blocages)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes

# 6. Active defrag (si fragmentation)
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
```

### Checklist de prévention

- [ ] maxmemory configuré à 80% de la RAM disponible
- [ ] maxmemory-policy configurée (pas noeviction)
- [ ] Monitoring actif avec alertes (85%, 95%)
- [ ] Audit régulier du modèle de données
- [ ] TTL sur toutes les données temporaires
- [ ] Big keys fragmentées ou éliminées
- [ ] Fragmentation surveillée (< 1.5)
- [ ] BGSAVE/BGREWRITEAOF fonctionnent
- [ ] Plan de scaling défini (quand sharding nécessaire)
- [ ] Procédure d'urgence OOM documentée

### Procédure d'urgence

```
1. DÉTECTER (< 1 min)
   → Alertes monitoring
   → Logs Redis/système

2. DIAGNOSTIQUER (< 2 min)
   → Type d'OOM (soft/hard/fork)
   → Cause probable
   → État actuel

3. AGIR (< 5 min)
   → Augmenter maxmemory temporairement
   → Activer éviction si désactivée
   → Supprimer big keys temporaires
   → Forcer évictions

4. STABILISER (< 30 min)
   → Vérifier état post-action
   → Monitorer tendance
   → Communiquer aux équipes

5. RÉSOUDRE (< 24h)
   → Identifier cause racine
   → Implémenter solution long terme
   → Ajuster configuration
   → Documenter incident

6. PRÉVENIR (continu)
   → Améliorer monitoring
   → Optimiser modèle données
   → Planifier scaling
   → Former équipes
```

---

## 🎯 Points clés à retenir

1. **maxmemory = OBLIGATOIRE** → Jamais en production sans limite
2. **Politique d'éviction = ESSENTIELLE** → Sinon OOM garanti
3. **Monitoring à 85%/95%** → Agir avant la crise
4. **TTL sur données temporaires** → Libération automatique
5. **Big keys = danger** → Fragmenter ou optimiser
6. **Fragmentation < 1.5** → Redémarrer ou defrag si > 1.5
7. **Fork peut doubler la mémoire** → Prévoir 2x pour persistence
8. **Sharding quand nécessaire** → Single instance a des limites

---

**🚀 Section suivante** : [14.6 - Corruption de données et recovery](./06-corruption-donnees-recovery.md)

⏭️ [Corruption de données et recovery](/14-performance-troubleshooting/06-corruption-donnees-recovery.md)
