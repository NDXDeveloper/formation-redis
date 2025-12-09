🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5 Stratégies hybrides pour la production

## Introduction

La **stratégie hybride** consiste à activer simultanément RDB et AOF pour bénéficier des avantages des deux mécanismes de persistance. Cette approche est devenue la **configuration standard recommandée** pour la production Redis depuis plusieurs années.

### Le meilleur des deux mondes

```
RDB seul          AOF seul          Hybride (RDB + AOF)
┌────────┐        ┌────────┐        ┌────────────────┐
│ Rapide │        │ Durable│        │ Rapide         │
│ Compact│        │ Fiable │        │ Durable        │
│ Simple │        │ Audit  │        │ Flexible       │
├────────┤        ├────────┤        │ Résilient      │
│ Perte  │        │ Lent   │        │ Production-    │
│ données│        │ Gros   │        │ ready          │
└────────┘        └────────┘        └────────────────┘
```

**Principe :** Redis écrit à la fois des snapshots RDB ET un journal AOF, permettant de choisir le mécanisme optimal selon les circonstances.

## Pourquoi combiner RDB et AOF ?

### Les 7 avantages de la stratégie hybride

#### 1. Double protection contre la perte de données

```
Scénario : Crash Redis brutal
├─ RDB : Dernier snapshot il y a 10 minutes → Perte 10 min
├─ AOF : Dernier fsync il y a 0.5 secondes → Perte 0.5s
└─ Décision au restart : Redis charge AOF (plus récent)
    → Perte réelle : 0.5 seconde ✅
```

#### 2. Flexibilité de récupération

```
Cas 1 : Démarrage normal
Redis charge automatiquement AOF (plus complet)

Cas 2 : AOF corrompu
redis-server --appendonly no  # Charge depuis RDB
Puis : réparer AOF et réactiver

Cas 3 : Démarrage rapide nécessaire
# Désactiver temporairement AOF pour charger RDB rapide
# Puis réactiver AOF
```

#### 3. Backups optimisés

```
RDB : Backups rapides et compacts
- Fichier unique
- Compression native
- Transfert réseau rapide
- Idéal pour archivage

AOF : Audit et récupération fine
- Journal complet des opérations
- Point-in-time recovery possible
- Debugging et analyse
```

#### 4. Optimisation des ressources

```
RDB : Utilisé pour
- Snapshots périodiques (toutes les 15 min)
- Backups quotidiens
- Réplication initiale

AOF : Utilisé pour
- Durabilité continue (fsync everysec)
- Récupération primaire
- Audit trail
```

#### 5. Résilience accrue

| Type de problème | RDB seul | AOF seul | Hybride |
|------------------|----------|----------|---------|
| Corruption RDB | ❌ Perte totale | ✅ AOF prend le relais | ✅ AOF prend le relais |
| Corruption AOF | ✅ RDB prend le relais | ⚠️ Réparation requise | ✅ RDB prend le relais |
| Disque plein | ⚠️ Snapshot échoue | ❌ Bloque tout | ⚠️ Snapshot échoue, AOF continue |
| OOM (fork) | ❌ Pas de snapshot | ✅ AOF continue | ⚠️ Pas de snapshot, AOF continue |

#### 6. Performance équilibrée

```
Performance :
RDB seul    : 100% (baseline)
AOF everysec: 70-75%
Hybride     : 65-70%

Durabilité :
RDB seul    : Faible (5-15 min perte)
AOF everysec: Excellente (1s perte)
Hybride     : Excellente (1s perte)

Verdict : -30% perf pour +1000% durabilité
```

#### 7. Simplicité opérationnelle

```bash
# Configuration simple
save 900 1
save 300 10
appendonly yes
appendfsync everysec

# Monitoring unifié
redis-cli INFO persistence  # Voir RDB + AOF

# Backup simple
cp dump.rdb backup/
cp appendonly.aof backup/

# Restauration automatique (Redis choisit AOF)
systemctl restart redis
```

## Configurations hybrides recommandées

### Configuration 1 : Standard (90% des cas)

**Cas d'usage :** Session store, job queues, leaderboards, compteurs métiers

```conf
# === RDB : Snapshots périodiques ===
save 900 1        # 15 minutes si 1+ changement
save 300 10       # 5 minutes si 10+ changements
save 60 10000     # 1 minute si 10K+ changements

dir /var/lib/redis
dbfilename dump.rdb
rdbcompression yes
rdbchecksum yes
stop-writes-on-bgsave-error yes

# Redis 7+
rdb-save-incremental-fsync yes

# === AOF : Journal continu ===
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec           # ← Compromis optimal

# Redis 7+ : Format hybride
aof-use-rdb-preamble yes      # RDB + delta AOF
appenddirname "appendonlydir"

# Réécriture automatique
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
no-appendfsync-on-rewrite no
aof-load-truncated yes
```

**Caractéristiques :**
- Perte max : ~1 seconde
- Performance : 65-70% du max
- Démarrage : Rapide (format hybride)
- Complexité : Moyenne

**Métriques attendues :**
```
Ops/sec: 55,000-65,000
Latence P50: 0.4ms
Latence P99: 5ms
CPU: 30-40%
RAM: 2x dataset (fork)
Disque: 4-5x dataset
```

### Configuration 2 : Performance prioritaire

**Cas d'usage :** Cache critique avec warm-up long, read-heavy workload

```conf
# === RDB : Snapshots plus espacés ===
save 3600 1       # 1 heure si 1+ changement
save 900 10       # 15 minutes si 10+ changements
save 300 100      # 5 minutes si 100+ changements

# RDB optimisé
rdbcompression yes
rdbchecksum yes
rdb-save-incremental-fsync yes

# === AOF : Durabilité réduite ===
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# Réécritures moins fréquentes
auto-aof-rewrite-percentage 150    # Tolérer 2.5x la taille base
auto-aof-rewrite-min-size 128mb

# Optimisation performance
no-appendfsync-on-rewrite yes      # Pas de fsync pendant rewrite
```

**Caractéristiques :**
- Perte max : 1-5 minutes (selon activity)
- Performance : 75-80% du max
- Priorité : Performance > Durabilité
- Trade-off : Acceptable pour cache

**Métriques attendues :**
```
Ops/sec: 65,000-75,000
Latence P50: 0.3ms
Latence P99: 3ms
CPU: 25-35%
Disque I/O: Réduit
```

### Configuration 3 : Durabilité maximale

**Cas d'usage :** Données critiques, compliance, finance

```conf
# === RDB : Backups fréquents ===
save 300 1        # 5 minutes si 1+ changement
save 60 10        # 1 minute si 10+ changements
save 30 100       # 30 secondes si 100+ changements

rdbcompression yes
rdbchecksum yes
stop-writes-on-bgsave-error yes
rdb-save-incremental-fsync yes

# === AOF : Durabilité maximale ===
appendonly yes
appendfsync always             # ← Chaque commande fsync
aof-use-rdb-preamble yes

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
no-appendfsync-on-rewrite no   # Maintenir durabilité TOUJOURS
aof-load-truncated no          # Stricte sur corruption
```

**Caractéristiques :**
- Perte max : Aucune (sauf crash matériel)
- Performance : 10-20% du max
- Priorité : Durabilité absolue
- Coût : Élevé (scaling horizontal requis)

**Métriques attendues :**
```
Ops/sec: 400-800 (par instance)
Latence P50: 10-20ms
Latence P99: 50-100ms
CPU: 20-30% (bloqué par I/O)
I/O disque: Maximum
```

**Architecture recommandée :**
```
Load Balancer
    ├─ Redis 1 (master) + 2 replicas
    ├─ Redis 2 (master) + 2 replicas
    └─ Redis 3 (master) + 2 replicas

Débit total: 1,200-2,400 ops/s
Coût: x5-10 vs config standard
```

### Configuration 4 : Write-heavy workload

**Cas d'usage :** IoT, analytics temps réel, logs

```conf
# === RDB : Snapshots fréquents mais tolérés ===
save 600 1
save 300 100
save 60 1000

rdbcompression yes
rdb-save-incremental-fsync yes

# === AOF : Optimisé pour volume ===
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# Réécritures agressives (volume élevé)
auto-aof-rewrite-percentage 50     # Réécrire dès 1.5x
auto-aof-rewrite-min-size 32mb
no-appendfsync-on-rewrite yes      # Optimisation (avec réplication)

# Buffers plus grands
aof-rewrite-incremental-fsync yes  # Redis 7+
```

**Caractéristiques :**
- Gestion active de la taille AOF
- Réécritures fréquentes
- Trade-off durabilité/volume

**Métriques attendues :**
```
Ops/sec: 50,000-60,000
AOF rewrites: 20-50/jour
Taille AOF: Contrôlée (<2x dataset)
I/O disque: Élevé
```

### Configuration 5 : Read-heavy avec écritures occasionnelles

**Cas d'usage :** Catalogue produits, configuration, référentiels

```conf
# === RDB : Priorité (peu d'écritures) ===
save 3600 1
save 1800 10
save 900 100

rdbcompression yes
rdbchecksum yes

# === AOF : Backup de sécurité ===
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# Réécritures espacées (peu d'activité)
auto-aof-rewrite-percentage 200
auto-aof-rewrite-min-size 128mb
```

**Caractéristiques :**
- RDB suffit presque
- AOF comme safety net
- Overhead minimal

**Métriques attendues :**
```
Ops/sec: 80,000+ (majoritairement GET)
Taille AOF: Stable
Réécritures: Rares (1-2/jour)
Overhead: Minimal
```

## Tableau comparatif des stratégies hybrides

| Critère | Standard | Performance | Durabilité max | Write-heavy | Read-heavy |
|---------|----------|-------------|----------------|-------------|------------|
| **Perte données** | ~1s | 1-5min | Aucune | ~1s | ~1s |
| **Ops/sec** | 55-65K | 65-75K | 400-800 | 50-60K | 80K+ |
| **Latence P99** | 5ms | 3ms | 50-100ms | 8ms | 2ms |
| **Complexité** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **Coût** | 1x | 1x | 5-10x | 1.5x | 0.8x |
| **Use cases** | 90% | Cache | Finance | IoT/Logs | Catalogue |

## Gestion du chargement au démarrage

### Ordre de priorité Redis

Lors du démarrage, Redis suit cet ordre de priorité pour charger les données :

```
Démarrage Redis
    ↓
1. AOF activé ?
    ├─ OUI → Chercher fichiers AOF
    │         ├─ AOF trouvé ET valide → Charger AOF
    │         └─ AOF manquant/corrompu → Erreur (ou charger RDB si config)
    └─ NON → Chercher fichier RDB
              ├─ RDB trouvé ET valide → Charger RDB
              └─ RDB manquant → Démarrer vide
```

### Tableau de décision au chargement

| AOF actif | Fichier AOF | Fichier RDB | Action Redis | Données chargées |
|-----------|-------------|-------------|--------------|------------------|
| ✅ | ✅ Valide | ✅ Valide | Charge AOF | Depuis AOF (plus récent) |
| ✅ | ✅ Valide | ❌ Absent | Charge AOF | Depuis AOF |
| ✅ | ❌ Absent | ✅ Valide | Erreur* | Aucune (ou RDB si config) |
| ✅ | ⚠️ Corrompu | ✅ Valide | Erreur* | Aucune (ou RDB si config) |
| ❌ | N/A | ✅ Valide | Charge RDB | Depuis RDB |
| ❌ | N/A | ❌ Absent | Démarre vide | Aucune |

*Avec `aof-load-truncated yes`, Redis tronque l'AOF corrompu et charge ce qui est valide.

### Configuration du comportement au chargement

```conf
# Tolérer un AOF tronqué (recommandé)
aof-load-truncated yes

# Si activé : Redis charge la partie valide de l'AOF
# Si désactivé : Redis refuse de démarrer si AOF corrompu
```

**Scénario de récupération :**

```bash
# AOF corrompu détecté
redis-server
# [WARNING] Short read while loading AOF. Truncating.

# Avec aof-load-truncated yes :
# Redis charge jusqu'au point de corruption
# Perte : Dernières commandes après corruption

# Sans aof-load-truncated (ou corruption sévère) :
# 1. Backup l'AOF corrompu
cp appendonly.aof appendonly.aof.broken

# 2. Réparer ou charger depuis RDB
redis-check-aof --fix appendonly.aof
# OU
redis-server --appendonly no  # Charge RDB

# 3. Réactiver AOF
redis-cli CONFIG SET appendonly yes
redis-cli BGREWRITEAOF
```

## Format hybride AOF (Redis 7+)

### Le format RDB-Preamble

Redis 7+ introduit un format AOF hybride qui combine RDB et AOF dans le même fichier :

```
Structure du fichier AOF hybride :
┌─────────────────────────────────────┐
│ === PRÉAMBULE RDB ===               │
│ REDIS0011                           │
│ [Snapshot complet en format RDB]    │
│ ...                                 │
│ (Compact, rapide à charger)         │
├─────────────────────────────────────┤
│ === DELTA AOF ===                   │
│ *3                                  │
│ $3                                  │
│ SET                                 │
│ ...                                 │
│ (Commandes depuis le snapshot)      │
└─────────────────────────────────────┘
```

### Avantages du format hybride

| Critère | AOF pur | AOF hybride | Amélioration |
|---------|---------|-------------|--------------|
| **Taille fichier** | 15 GB | 5 GB | -67% |
| **Temps chargement** | 8 min | 1 min | -88% |
| **Fréquence rewrite** | Élevée | Réduite | -50% |
| **Compatibilité** | Ancienne | Redis 7+ | - |
| **Lisibilité** | Totale | Partielle | Préambule binaire |

### Configuration du format hybride

```conf
# Activer le format hybride (Redis 7+)
aof-use-rdb-preamble yes

# Après activation, forcer une réécriture
redis-cli BGREWRITEAOF
```

**Impact immédiat :**
```bash
# Avant
-rw-r--r-- 1 redis redis 15G appendonly.aof

# Après BGREWRITEAOF avec aof-use-rdb-preamble yes
-rw-r--r-- 1 redis redis 5.2G appendonly.aof

Gain : -65% taille
Temps chargement : 8min → 1min20s
```

### Évolution du format avec multi-part (Redis 7.0+)

```
Redis 6 (AOF unique) :
appendonly.aof  (format hybride possible)

Redis 7+ (Multi-part) :
appendonlydir/
  ├── appendonly.aof.1.base.rdb      ← Préambule RDB
  ├── appendonly.aof.1.incr.aof      ← Incrément 1
  ├── appendonly.aof.2.incr.aof      ← Incrément 2
  └── appendonly.aof.manifest        ← Manifeste
```

**Avantages multi-part :**
- Réécritures non-bloquantes améliorées
- Corruption isolée (seul un segment affecté)
- Gestion simplifiée des gros AOF

## Optimisation des ressources

### Gestion de la mémoire

#### Dimensionnement avec stratégie hybride

```
Calcul mémoire requise :

RAM serveur = Dataset + Overhead fork + Buffers système

Détail :
- Dataset : 20 GB
- Overhead fork RDB : 20 GB × 50% (COW) = 10 GB
- Buffers AOF : 500 MB
- Système : 2 GB

Total : 32.5 GB → Provisionner 40 GB (marge)
```

#### Tableau de dimensionnement

| Dataset | RAM minimal | RAM recommandée | Justification |
|---------|-------------|-----------------|---------------|
| 1 GB | 3 GB | 4 GB | Fork + marge |
| 5 GB | 12 GB | 16 GB | 2.5x dataset |
| 10 GB | 22 GB | 32 GB | 2x dataset + overhead |
| 20 GB | 42 GB | 64 GB | 2x dataset + confort |
| 50 GB | 105 GB | 128 GB | 2x dataset + buffers |
| 100 GB | 210 GB | 256 GB | 2x dataset + marge |

**Note :** Avec AOF seul (sans RDB), RAM requise = Dataset × 1.1-1.2 uniquement.

### Gestion du disque

#### Estimation de l'espace disque

```
Espace disque = RDB + AOF + Buffers + Backups

Exemple (dataset 20 GB) :
- RDB : 6 GB (30% compression)
- AOF hybride : 8 GB (40% dataset)
- Buffers/tmp : 2 GB
- Backups locaux (7 jours) : 6 GB × 7 = 42 GB

Total : 58 GB → Provisionner 100 GB (marge)
```

#### Tableau d'estimation disque

| Dataset RAM | RDB | AOF hybride | Backups (7j) | Total | Recommandé |
|-------------|-----|-------------|--------------|-------|------------|
| 5 GB | 1.5 GB | 2.5 GB | 10 GB | 14 GB | 30 GB |
| 10 GB | 3 GB | 5 GB | 20 GB | 28 GB | 50 GB |
| 20 GB | 6 GB | 10 GB | 42 GB | 58 GB | 100 GB |
| 50 GB | 15 GB | 25 GB | 105 GB | 145 GB | 250 GB |
| 100 GB | 30 GB | 50 GB | 210 GB | 290 GB | 500 GB |

### Optimisation de l'I/O

#### Répartition des fichiers

**Configuration optimale :**
```conf
# Redis principal sur SSD rapide
dir /mnt/ssd-fast/redis
dbfilename dump.rdb
appendonly yes
appenddirname "appendonlydir"

# Backups sur stockage moins coûteux
# (via script externe)
/mnt/storage/redis-backups/
```

**Architecture multi-disques (avancé) :**
```
Disque 1 (NVMe) : Redis workdir + AOF
├── /var/lib/redis/
│   ├── appendonlydir/
│   └── (fichiers actifs)

Disque 2 (SSD) : Snapshots RDB
├── /mnt/ssd/redis-snapshots/
│   └── dump.rdb

Disque 3 (HDD RAID) : Backups
├── /mnt/backups/redis/
│   ├── dump-YYYYMMDD.rdb
│   └── aof-YYYYMMDD.tar.gz
```

**Gains :**
- NVMe pour AOF → Latence fsync réduite
- SSD séparé pour RDB → Pas de contention I/O
- HDD pour backups → Coût optimisé

#### Stratégies selon le workload

| Workload | Configuration I/O | Justification |
|----------|-------------------|---------------|
| **Read-heavy** | RDB + AOF sur même disque | I/O faible |
| **Balanced** | Tout sur SSD | Bon compromis |
| **Write-heavy** | AOF sur NVMe, RDB sur SSD | Isoler AOF critique |
| **Très write-heavy** | AOF sur RAID-10 SSD | Débit maximum |

## Monitoring de la stratégie hybride

### Métriques combinées essentielles

```bash
# Script de monitoring complet
redis-cli INFO persistence

# Métriques RDB
rdb_changes_since_last_save:1247
rdb_bgsave_in_progress:0
rdb_last_save_time:1701878400
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:3

# Métriques AOF
aof_enabled:1
aof_rewrite_in_progress:0
aof_last_rewrite_time_sec:5
aof_current_size:104857600
aof_base_size:52428800
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_delayed_fsync:0
```

### Dashboard recommandé

**Métriques à grapher (Prometheus + Grafana) :**

```promql
# 1. Durée depuis dernière sauvegarde
(time() - redis_rdb_last_save_timestamp_seconds) / 60

# 2. Ratio croissance AOF
redis_aof_current_size_bytes / redis_aof_base_size_bytes

# 3. Fsync retardés (problème I/O)
rate(redis_aof_delayed_fsync[5m])

# 4. Statut des sauvegardes
redis_rdb_last_bgsave_status == 0  # 0 = erreur

# 5. Temps des réécritures
redis_aof_last_rewrite_duration_seconds

# 6. Espace disque utilisé
node_filesystem_avail_bytes{mountpoint="/var/lib/redis"}
```

### Alertes recommandées

| Alerte | Condition | Sévérité | Action |
|--------|-----------|----------|--------|
| RDB save failed | `rdb_last_bgsave_status != ok` | 🔴 Critique | Investiguer logs immédiatement |
| AOF write failed | `aof_last_write_status != ok` | 🔴 Critique | Vérifier disque, Redis bloqué |
| AOF rewrite failed | `aof_last_bgrewrite_status != ok` | 🟡 Warning | Vérifier espace disque |
| Pas de save depuis 2h | `time() - rdb_last_save_time > 7200` | 🟡 Warning | Vérifier snapshots actifs |
| AOF trop gros | `aof_current_size / aof_base_size > 5` | 🟡 Warning | Forcer réécriture |
| Fsync retardés | `aof_delayed_fsync > 0` (croissant) | 🟡 Warning | Problème I/O disque |
| Disque presque plein | `disk_usage > 85%` | 🔴 Critique | Libérer espace |

### Script de health-check

```bash
#!/bin/bash
# health-check-redis-persistence.sh

# Couleurs
RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
NC='\033[0m'

echo "=== Redis Persistence Health Check ==="

# Check RDB
RDB_STATUS=$(redis-cli INFO persistence | grep rdb_last_bgsave_status | cut -d: -f2 | tr -d '\r')
RDB_TIME=$(redis-cli INFO persistence | grep rdb_last_save_time | cut -d: -f2 | tr -d '\r')
NOW=$(date +%s)
RDB_AGE=$((NOW - RDB_TIME))

echo -e "\n--- RDB Status ---"
if [ "$RDB_STATUS" == "ok" ]; then
    echo -e "${GREEN}✓${NC} Last RDB save: OK"
else
    echo -e "${RED}✗${NC} Last RDB save: FAILED"
fi
echo "Last save: $((RDB_AGE / 60)) minutes ago"

if [ $RDB_AGE -gt 3600 ]; then
    echo -e "${YELLOW}⚠${NC} Warning: No RDB save for >1 hour"
fi

# Check AOF
AOF_ENABLED=$(redis-cli INFO persistence | grep aof_enabled | cut -d: -f2 | tr -d '\r')
AOF_WRITE_STATUS=$(redis-cli INFO persistence | grep aof_last_write_status | cut -d: -f2 | tr -d '\r')
AOF_REWRITE_STATUS=$(redis-cli INFO persistence | grep aof_last_bgrewrite_status | cut -d: -f2 | tr -d '\r')
AOF_SIZE=$(redis-cli INFO persistence | grep aof_current_size | cut -d: -f2 | tr -d '\r')
AOF_BASE=$(redis-cli INFO persistence | grep aof_base_size | cut -d: -f2 | tr -d '\r')
AOF_DELAYED=$(redis-cli INFO persistence | grep aof_delayed_fsync | cut -d: -f2 | tr -d '\r')

echo -e "\n--- AOF Status ---"
if [ "$AOF_ENABLED" == "1" ]; then
    echo -e "${GREEN}✓${NC} AOF enabled"

    if [ "$AOF_WRITE_STATUS" == "ok" ]; then
        echo -e "${GREEN}✓${NC} AOF writes: OK"
    else
        echo -e "${RED}✗${NC} AOF writes: FAILED"
    fi

    if [ "$AOF_REWRITE_STATUS" == "ok" ]; then
        echo -e "${GREEN}✓${NC} Last AOF rewrite: OK"
    else
        echo -e "${YELLOW}⚠${NC} Last AOF rewrite: FAILED"
    fi

    # Ratio de croissance
    if [ "$AOF_BASE" -gt 0 ]; then
        AOF_RATIO=$((AOF_SIZE / AOF_BASE))
        echo "AOF size ratio: ${AOF_RATIO}x base size"

        if [ $AOF_RATIO -gt 5 ]; then
            echo -e "${YELLOW}⚠${NC} Warning: AOF very large, consider rewrite"
        fi
    fi

    # Fsync retardés
    if [ "$AOF_DELAYED" -gt 0 ]; then
        echo -e "${YELLOW}⚠${NC} Warning: $AOF_DELAYED delayed fsyncs (I/O issue)"
    else
        echo -e "${GREEN}✓${NC} No delayed fsyncs"
    fi
else
    echo -e "${YELLOW}⚠${NC} AOF disabled"
fi

# Check disk space
DISK_USAGE=$(df /var/lib/redis | tail -1 | awk '{print $5}' | sed 's/%//')
echo -e "\n--- Disk Space ---"
echo "Usage: ${DISK_USAGE}%"
if [ "$DISK_USAGE" -gt 85 ]; then
    echo -e "${RED}✗${NC} Critical: Disk usage >85%"
elif [ "$DISK_USAGE" -gt 70 ]; then
    echo -e "${YELLOW}⚠${NC} Warning: Disk usage >70%"
else
    echo -e "${GREEN}✓${NC} Disk space OK"
fi

echo -e "\n=== End of Health Check ===\n"
```

## Troubleshooting stratégie hybride

### Problème 1 : Redémarrage très lent

**Symptôme :**
```bash
systemctl restart redis
# Redis prend 10+ minutes à démarrer
```

**Diagnostic :**
```bash
# Vérifier taille AOF
ls -lh /var/lib/redis/appendonly.aof
-rw-r--r-- 1 redis redis 25G appendonly.aof  # Très gros !

# Vérifier si format hybride activé
redis-cli CONFIG GET aof-use-rdb-preamble
1) "aof-use-rdb-preamble"
2) "no"  # ← Problème : format AOF pur
```

**Solutions :**

1. **Activer format hybride (Redis 7+) :**
```bash
redis-cli CONFIG SET aof-use-rdb-preamble yes
redis-cli BGREWRITEAOF

# Attendre la fin du rewrite
redis-cli INFO persistence | grep aof_rewrite_in_progress
```

2. **Forcer chargement depuis RDB (temporaire) :**
```bash
# Démarrer sans AOF pour charger RDB rapide
redis-server --appendonly no

# Une fois démarré, réactiver AOF
redis-cli CONFIG SET appendonly yes
redis-cli BGREWRITEAOF
```

3. **Réduire la taille AOF avant restart :**
```bash
redis-cli BGREWRITEAOF
# Attendre la fin
redis-cli INFO persistence | grep aof_rewrite_in_progress:0

# Puis restart
systemctl restart redis
```

### Problème 2 : AOF et RDB désynchronisés

**Symptôme :**
```bash
# RDB très ancien
redis-cli LASTSAVE
1701878400  # Il y a 24 heures !

# AOF OK
redis-cli INFO persistence | grep aof_last_write_status
aof_last_write_status:ok
```

**Causes possibles :**
- Snapshots RDB qui échouent silencieusement
- `stop-writes-on-bgsave-error no` configuré
- OOM lors des forks RDB

**Diagnostic :**
```bash
# Vérifier statut RDB
redis-cli INFO persistence | grep rdb_last_bgsave_status
rdb_last_bgsave_status:err  # ← Problème !

# Vérifier les logs
tail -f /var/log/redis/redis-server.log
# Background saving error: Cannot allocate memory
```

**Solutions :**

1. **Augmenter la RAM disponible**
2. **Activer overcommit :**
```bash
sysctl vm.overcommit_memory=1
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
```

3. **Réduire la fréquence des snapshots :**
```conf
# Moins agressif
save 3600 1
save 1800 10
```

4. **Forcer un snapshot manuel :**
```bash
redis-cli BGSAVE
```

### Problème 3 : Disque plein avec stratégie hybride

**Symptôme :**
```bash
redis-cli SET key value
(error) MISCONF Redis is configured to save RDB/AOF but is currently not able to persist on disk.

df -h /var/lib/redis
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       50G   50G     0 100% /var  # Disque plein !
```

**Actions immédiates :**

1. **Identifier les fichiers volumineux :**
```bash
du -sh /var/lib/redis/*
25G     /var/lib/redis/appendonly.aof
15G     /var/lib/redis/dump.rdb
8G      /var/lib/redis/appendonlydir/
```

2. **Libérer de l'espace (backups anciens) :**
```bash
# Supprimer anciens backups
find /backup/redis -name "*.rdb" -mtime +7 -delete
find /backup/redis -name "*.aof.gz" -mtime +7 -delete
```

3. **Forcer réécriture AOF :**
```bash
redis-cli BGREWRITEAOF
# Cela compactera l'AOF (peut libérer 50-80%)
```

4. **Temporairement désactiver RDB :**
```bash
redis-cli CONFIG SET save ""
# Cela évite les nouveaux snapshots RDB
```

5. **Augmenter le disque (solution permanente)**

### Problème 4 : Réécritures AOF trop fréquentes

**Symptôme :**
```bash
# Logs Redis
[1234] Background AOF rewrite started
[1234] Background AOF rewrite finished successfully
[1234] Background AOF rewrite started  # 10 minutes plus tard !
```

**Causes :**
- Write-heavy workload
- Seuil de réécriture trop bas
- AOF qui grandit très vite

**Solutions :**

1. **Ajuster les seuils :**
```conf
# Plus tolérant
auto-aof-rewrite-percentage 150  # Au lieu de 100
auto-aof-rewrite-min-size 128mb  # Au lieu de 64mb
```

2. **Optimiser pendant réécriture :**
```conf
# Désactiver fsync pendant rewrite (si réplication)
no-appendfsync-on-rewrite yes
```

3. **Considérer le sharding :**
```
1 Redis (write-heavy) → 3 Redis (sharding)
Charge divisée par 3
Réécritures moins fréquentes par instance
```

## Stratégies avancées

### Stratégie 1 : Déporter la persistance sur replicas

**Architecture :**
```
Master (écritures)
├── RDB : désactivé (save "")
├── AOF : désactivé (appendonly no)
└── Performance maximale

Replica 1 (persistence)
├── RDB : activé (save 900 1)
├── AOF : activé (appendfsync everysec)
└── Assure la durabilité

Replica 2 (persistence)
├── RDB : activé
├── AOF : activé
└── Redondance
```

**Configuration master :**
```conf
# Master : Performance maximale
save ""
appendonly no
```

**Configuration replicas :**
```conf
# Replicas : Persistance complète
replicaof <master-ip> 6379

save 900 1
save 300 10
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
```

**Avantages :**
- ✅ Master : Performance maximale (100%)
- ✅ Durabilité : Assurée par replicas
- ✅ Backups : Depuis replicas (pas d'impact master)

**Inconvénients :**
- ⚠️ Latence de réplication (ms)
- ⚠️ Si master crash, basculer sur replica
- ⚠️ Complexité opérationnelle accrue

**Quand l'utiliser :**
- Workload extrêmement write-heavy
- Performance absolument critique
- Infrastructure mature avec Sentinel/Cluster

### Stratégie 2 : Persistance tiered

**Concept :** Différencier persistance selon criticité des données

```
Keys critiques (user:*, order:*)
├── Master : RDB + AOF always
└── Protection maximale

Keys non-critiques (cache:*, tmp:*)
├── Master : RDB seul
└── Performance optimale
```

**Implémentation :** Utiliser plusieurs instances Redis

```
Redis instance 1 (critical-data)
Port: 6379
Config: RDB + AOF everysec

Redis instance 2 (cache-data)
Port: 6380
Config: RDB espacé

Application
├── Critical data → Redis :6379
└── Cache data → Redis :6380
```

### Stratégie 3 : Snapshot coordonnés (Multi-instance)

**Problème :** Plusieurs instances Redis, snapshots non coordonnés

**Solution :** Script de snapshot coordonné

```bash
#!/bin/bash
# coordinated-snapshot.sh

INSTANCES=(
    "localhost:6379"
    "localhost:6380"
    "localhost:6381"
)

echo "Starting coordinated snapshot..."

# 1. Déclencher BGSAVE sur toutes les instances
for instance in "${INSTANCES[@]}"; do
    HOST=$(echo $instance | cut -d: -f1)
    PORT=$(echo $instance | cut -d: -f2)
    redis-cli -h $HOST -p $PORT BGSAVE &
done

# 2. Attendre la fin de tous les snapshots
for instance in "${INSTANCES[@]}"; do
    HOST=$(echo $instance | cut -d: -f1)
    PORT=$(echo $instance | cut -d: -f2)

    while [ $(redis-cli -h $HOST -p $PORT INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2) -eq 1 ]; do
        sleep 1
    done

    echo "Snapshot completed on $instance"
done

echo "All snapshots completed"

# 3. Copier tous les RDB avec timestamp identique
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
for instance in "${INSTANCES[@]}"; do
    PORT=$(echo $instance | cut -d: -f2)
    cp /var/lib/redis-$PORT/dump.rdb /backup/redis/dump-$PORT-$TIMESTAMP.rdb
done
```

## Cas d'usage réels

### Cas 1 : Plateforme SaaS - Multi-tenant

**Contexte :**
- 10,000 tenants
- Sessions + cache + jobs
- SLA 99.9%

**Architecture :**
```
Load Balancer
    ├── Redis Cluster (3 masters + 3 replicas)
    │   ├── Master 1 : RDB + AOF everysec
    │   ├── Master 2 : RDB + AOF everysec
    │   └── Master 3 : RDB + AOF everysec
    │
    └── Redis Sentinel (3 instances)
        └── Monitoring + Failover auto
```

**Configuration :**
```conf
# Hybride standard
save 900 1
save 300 10
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Résultats :**
- Durabilité : Perte max 1 seconde
- Disponibilité : 99.95% (dépasse SLA)
- Performance : 45,000 ops/s par master
- Coût : Modéré (6 instances)

### Cas 2 : E-commerce - Black Friday

**Contexte :**
- Traffic normal : 5,000 ops/s
- Traffic Black Friday : 50,000 ops/s
- Performance critique

**Stratégie :**

**Pré Black Friday (T-7 jours) :**
```conf
# Optimiser pour performance
save 3600 1
save 900 10

appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite yes  # Optimisation

# Scaling horizontal
# 1 instance → 5 instances (sharding)
```

**Pendant Black Friday :**
```bash
# Monitoring intensif
watch -n 5 'redis-cli INFO stats | grep instantaneous_ops_per_sec'

# Si besoin, désactiver temporairement AOF fsync
redis-cli CONFIG SET appendfsync no
# (Durée limitée, avec surveillance)
```

**Post Black Friday (T+1) :**
```bash
# Restaurer config normale
redis-cli CONFIG SET appendfsync everysec

# Forcer réécriture AOF
redis-cli BGREWRITEAOF

# Backup complet
redis-cli BGSAVE
```

### Cas 3 : Finance - Trading platform

**Contexte :**
- Transactions financières
- Aucune perte tolérée
- Latence critique (<10ms)

**Architecture :**
```
Region 1 (Primary)
├── Redis 1 : AOF always + RDB
├── Redis 2 : AOF always + RDB
└── Redis 3 : AOF always + RDB
    └── Load Balancer (round-robin)

Region 2 (DR)
├── Redis 4 : Réplication depuis Région 1
├── Redis 5 : Réplication depuis Région 1
└── Redis 6 : Réplication depuis Région 1
```

**Configuration :**
```conf
# Durabilité maximale
save 300 1
appendonly yes
appendfsync always
aof-use-rdb-preamble yes
no-appendfsync-on-rewrite no
stop-writes-on-bgsave-error yes
```

**Résultats :**
- Durabilité : Maximale (aucune perte)
- Performance : 1,200 ops/s total (3 × 400)
- Latence P99 : 35ms
- Coût : Élevé (6 instances + DR)

**Trade-off accepté :** Performance sacrifiée pour durabilité absolue

## Checklist de production

### Configuration

- [ ] RDB activé avec snapshots réguliers
- [ ] AOF activé avec `appendfsync everysec`
- [ ] Format hybride activé (`aof-use-rdb-preamble yes` sur Redis 7+)
- [ ] Réécritures automatiques configurées
- [ ] `stop-writes-on-bgsave-error yes`
- [ ] `aof-load-truncated yes`

### Infrastructure

- [ ] RAM >= 2x dataset (pour fork RDB)
- [ ] Disque >= 5x dataset (RDB + AOF + backups)
- [ ] SSD pour workdir (NVMe si write-heavy)
- [ ] Overcommit memory activé (`vm.overcommit_memory=1`)
- [ ] Transparent Huge Pages désactivé

### Haute disponibilité

- [ ] Au minimum 1 replica (idéalement 2+)
- [ ] Redis Sentinel configuré (3 instances minimum)
- [ ] Tests de failover réguliers (mensuel)
- [ ] Procédures de recovery documentées

### Backups

- [ ] Backups automatisés quotidiens
- [ ] Backups RDB ET AOF
- [ ] Stockage hors-site (S3, autre datacenter)
- [ ] Rétention : 7 jours + 4 semaines + 6 mois
- [ ] Tests de restauration réguliers (mensuel)

### Monitoring

- [ ] Métriques RDB dans Prometheus
- [ ] Métriques AOF dans Prometheus
- [ ] Dashboard Grafana configuré
- [ ] Alertes sur échecs de persistance
- [ ] Alertes sur croissance anormale AOF
- [ ] Alertes sur espace disque

### Tests

- [ ] Test crash Redis (kill -9)
- [ ] Test crash système
- [ ] Test corruption AOF
- [ ] Test disque plein
- [ ] Test restauration depuis backup
- [ ] Mesure temps de chargement

## Conclusion

La stratégie hybride (RDB + AOF) représente le **standard de facto** pour Redis en production. Elle offre :

- ✅ **Durabilité excellente** : Perte maximale de 1 seconde
- ✅ **Performance acceptable** : 60-70% du maximum théorique
- ✅ **Flexibilité** : Multiple options de récupération
- ✅ **Résilience** : Tolérance aux corruptions
- ✅ **Production-ready** : Testé et éprouvé

### Recommandation universelle

**Pour 90% des applications :**
```conf
save 900 1
save 300 10
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
```

### Adaptations selon le contexte

- **Cache pur** → RDB seul suffit
- **Performance critique** → Réduire fréquence RDB
- **Durabilité critique** → AOF always + Réplication
- **Write-heavy** → Réécritures agressives + Sharding
- **Read-heavy** → Configuration standard

**Règle d'or** : Toujours tester vos configurations en environnement de staging avant la production, et valider régulièrement vos procédures de restauration.

---

**Points clés à retenir :**
- La stratégie hybride combine les avantages de RDB et AOF
- Format hybride (Redis 7+) : -60% taille, -70% temps chargement
- Dimensionner : RAM = 2x dataset, Disque = 5x dataset
- Monitoring actif : RDB + AOF + espace disque
- Toujours avoir des backups testés

---


⏭️ [Backup et restauration : Bonnes pratiques](/05-persistance-fiabilite/06-backup-restauration-bonnes-pratiques.md)
