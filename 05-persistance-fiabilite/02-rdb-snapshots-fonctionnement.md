🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2 RDB (Redis Database) : Snapshots et fonctionnement

## Introduction

RDB (Redis Database) est le mécanisme de persistance le plus ancien et le plus simple de Redis. Il fonctionne sur le principe des **snapshots** : à intervalles réguliers ou sur demande, Redis crée une **copie complète** de toutes les données en mémoire et l'enregistre dans un fichier binaire compact sur disque.

### Principes fondamentaux

- **Format** : Fichier binaire compact (`.rdb`)
- **Contenu** : Snapshot complet de l'état de la base à un instant T
- **Déclenchement** : Automatique (règles) ou manuel (commande)
- **Mécanisme** : Fork du processus + Copy-on-Write (COW)
- **Blocage** : Non-bloquant pour les lectures/écritures

## Architecture et fonctionnement interne

### Le processus de création d'un snapshot RDB

```
1. Déclenchement (règle save ou commande BGSAVE)
         ↓
2. Redis fork() le processus principal
         ↓
3. Processus enfant créé (copie mémoire virtuelle)
         ↓
4. Processus parent continue à servir les requêtes
         ↓
5. Processus enfant écrit le snapshot sur disque
         ↓
6. Fichier .rdb.tmp créé
         ↓
7. Rename atomique : .rdb.tmp → .rdb
         ↓
8. Processus enfant termine et libère la mémoire
         ↓
9. Redis continue normalement
```

### Le mécanisme Copy-on-Write (COW)

Le **Copy-on-Write** est la technique clé qui permet à RDB d'être non-bloquant :

```
Avant le fork :
┌─────────────────────────┐
│   Redis Process         │
│   RAM: 10 GB            │
│   PID: 1234             │
└─────────────────────────┘

Après le fork (immédiat) :
┌─────────────────────────┐     ┌─────────────────────────┐
│   Redis Parent          │     │   Redis Child           │
│   PID: 1234             │     │   PID: 5678             │
│   Continue les requêtes │     │   Écrit le snapshot     │
└─────────────────────────┘     └─────────────────────────┘
         │                               │
         └───────── MÊME MÉMOIRE ────────┘
              (pages virtuelles partagées)

Quand le parent modifie une page :
┌─────────────────────────┐     ┌─────────────────────────┐
│   Redis Parent          │     │   Redis Child           │
│   Page A' (nouvelle)    │     │   Page A (originale)    │
│   Page B (partagée)     │     │   Page B (partagée)     │
└─────────────────────────┘     └─────────────────────────┘
```

**Avantage** : Le fork est instantané (quelques ms), seules les pages modifiées sont copiées.

**Coût mémoire** : Dans le pire cas (toutes les données modifiées pendant le snapshot), jusqu'à **2x la RAM** peut être nécessaire.

### Chronologie détaillée d'un BGSAVE

| Étape | Durée | Impact | Description |
|-------|-------|--------|-------------|
| **1. Déclenchement** | <1ms | Aucun | Règle save déclenchée ou commande BGSAVE |
| **2. Fork** | 1-100ms | ⚠️ Blocage très court | Création du processus enfant (dépend de la RAM) |
| **3. Setup** | 1-10ms | Aucun | Préparation du fichier .rdb.tmp |
| **4. Écriture** | 10s-5min | CPU + I/O | Sérialisation et écriture sur disque |
| **5. Fsync** | 100ms-5s | I/O | Synchronisation disque (selon config) |
| **6. Rename** | <1ms | Aucun | Rename atomique du fichier |
| **7. Cleanup** | 1-100ms | Aucun | Libération du processus enfant |

**Total** : 15 secondes à 5 minutes selon la taille du dataset et la performance du disque.

## Configuration RDB : Les règles de déclenchement

### Syntaxe de configuration

```conf
# redis.conf

# Format : save <secondes> <nombre de modifications>
save 900 1        # Snapshot si au moins 1 modification en 15 minutes
save 300 10       # Snapshot si au moins 10 modifications en 5 minutes
save 60 10000     # Snapshot si au moins 10000 modifications en 1 minute
```

### Tableau des configurations standards

| Configuration | Fréquence snapshot | Perte max | Overhead | Cas d'usage |
|---------------|-------------------|-----------|----------|-------------|
| `save ""` | Jamais | Toutes données | Aucun | Cache pur |
| `save 3600 1` | Toutes les heures | 1 heure | Très faible | Cache peu critique |
| `save 900 1`<br>`save 300 10` | 5-15 minutes | 5-15 min | Faible | Cache important |
| `save 300 10`<br>`save 60 100` | 1-5 minutes | 1-5 min | Moyen | Données semi-critiques |
| `save 60 1` | Chaque minute | 1 minute | Élevé | Données critiques (+ AOF recommandé) |

### Calcul de la fréquence réelle

Les règles RDB sont évaluées en **OU logique** : le snapshot est déclenché dès qu'**UNE** condition est remplie.

**Exemple 1 : Configuration standard**
```conf
save 900 1      # Règle A
save 300 10     # Règle B
save 60 10000   # Règle C
```

**Scénarios :**

| Activité | Règle déclenchée | Délai snapshot |
|----------|------------------|----------------|
| 1 SET puis silence | Règle A | 15 minutes |
| 50 SET en 3 minutes | Règle B | 5 minutes |
| Burst de 15000 SET | Règle C | 1 minute |
| 5 SET/minute constants | Règle B | 5 minutes max |

**Exemple 2 : Configuration agressive**
```conf
save 60 1
save 30 10
save 10 100
```

**Impact :**
- Snapshot minimum toutes les minutes
- Overhead élevé (CPU + I/O)
- Perte de données maximum : 1 minute
- Réservé aux cas où AOF n'est pas disponible

### Configuration avancée

```conf
# Fichier de sortie
dir /var/lib/redis                    # Répertoire de travail
dbfilename dump.rdb                   # Nom du fichier

# Compression
rdbcompression yes                    # Compression LZF (recommandé)
rdbchecksum yes                       # Checksum CRC64 (recommandé)

# Comportement en cas d'erreur
stop-writes-on-bgsave-error yes       # Arrêter les écritures si snapshot échoue

# Optimisation I/O (Redis 7+)
rdb-save-incremental-fsync yes        # Fsync progressif (réduit les pics I/O)
```

### Comparaison des paramètres

| Paramètre | Valeur | Impact performance | Impact fiabilité | Recommandation |
|-----------|--------|-------------------|------------------|----------------|
| `rdbcompression` | yes | -10% CPU snapshot | +50% gain espace | ✅ Activer |
| `rdbcompression` | no | Snapshot plus rapide | Fichiers 2-3x plus gros | ❌ Désactiver uniquement si CPU critique |
| `rdbchecksum` | yes | -5% snapshot | Détection corruption | ✅ Activer |
| `stop-writes-on-bgsave-error` | yes | Blocage écritures | Protection données | ✅ Activer (production) |
| `stop-writes-on-bgsave-error` | no | Continue malgré erreur | Risque perte silencieuse | ❌ Dangereux |
| `rdb-save-incremental-fsync` | yes | I/O lissés | Légère augmentation durée | ✅ Activer (Redis 7+) |

## Commandes RDB : SAVE vs BGSAVE

### SAVE : Snapshot bloquant (synchrone)

```bash
redis-cli SAVE
```

**Comportement :**
- ✅ Snapshot créé immédiatement
- ❌ **BLOQUE COMPLÈTEMENT Redis** pendant toute l'opération
- ❌ Aucune requête ne peut être servie (lectures et écritures)
- ✅ Garantie de cohérence absolue
- ⏱️ Durée : 5 secondes à plusieurs minutes

**Quand l'utiliser :**
- ✅ Arrêt planifié de Redis (maintenance)
- ✅ Backup manuel avant opération critique
- ✅ Scripts de sauvegarde hors-ligne
- ❌ **JAMAIS en production avec traffic actif**

**Exemple d'utilisation :**
```bash
# Shutdown gracieux avec sauvegarde
redis-cli SAVE
redis-cli SHUTDOWN

# Backup avant migration
redis-cli SAVE
cp /var/lib/redis/dump.rdb /backup/dump-$(date +%Y%m%d).rdb
```

### BGSAVE : Snapshot non-bloquant (asynchrone)

```bash
redis-cli BGSAVE
```

**Comportement :**
- ✅ Snapshot en arrière-plan (fork + child process)
- ✅ Redis continue de servir les requêtes
- ⚠️ Blocage très bref lors du fork (1-100ms)
- ✅ Production-safe
- ⏱️ Durée totale : 10 secondes à 5 minutes (invisible pour les clients)

**Quand l'utiliser :**
- ✅ **Production avec traffic** (utilisation standard)
- ✅ Backup manuel pendant le service
- ✅ Déclenchement par cron/script
- ✅ Snapshot avant opération risquée

**Vérifier l'état du BGSAVE :**
```bash
# Statut actuel
redis-cli INFO persistence | grep rdb_bgsave_in_progress
rdb_bgsave_in_progress:0

# Dernier snapshot
redis-cli LASTSAVE
1701878400  # Timestamp Unix

# Dernier statut
redis-cli INFO persistence | grep rdb_last_bgsave_status
rdb_last_bgsave_status:ok
```

### BGSAVE vs SAVE : Tableau comparatif

| Critère | SAVE | BGSAVE |
|---------|------|--------|
| **Blocage** | ❌ Total (toute la durée) | ✅ Très bref (<100ms au fork) |
| **Durée blocage** | 5s-5min | 1-100ms |
| **CPU** | 1 core | 1 core (process enfant) |
| **Mémoire** | 1x dataset | Jusqu'à 2x dataset (COW) |
| **Production** | ❌ Dangereux | ✅ Recommandé |
| **Cohérence** | ✅ Absolue | ✅ Point-in-time |
| **Commandes concurrentes** | ❌ Impossibles | ✅ Possibles |

### BGSAVE avec scheduler : Déclenchement automatique

La configuration `save` déclenche automatiquement des `BGSAVE` :

```conf
# Configuration
save 900 1
save 300 10
```

**Équivalent à :**
```bash
# Redis évalue en continu :
if (nb_modifications >= 1 AND elapsed_time >= 900) {
    BGSAVE
} else if (nb_modifications >= 10 AND elapsed_time >= 300) {
    BGSAVE
}
```

**Vérification du scheduler :**
```bash
redis-cli INFO persistence
# Sortie :
rdb_changes_since_last_save:1247
rdb_bgsave_in_progress:0
rdb_last_save_time:1701878400
rdb_last_bgsave_time_sec:3
rdb_last_bgsave_status:ok
```

## Format du fichier RDB

### Structure interne

Le fichier RDB est un **format binaire propriétaire** optimisé pour :
- Compacité (compression LZF intégrée)
- Vitesse de chargement
- Intégrité (checksum CRC64)

**Structure générale :**
```
┌──────────────────────────────────────┐
│ REDIS0011 (magic string + version)   │  9 bytes
├──────────────────────────────────────┤
│ Metadata (redis-ver, ctime, etc)     │  Variable
├──────────────────────────────────────┤
│ Database selector (DB 0)             │  Variable
├──────────────────────────────────────┤
│ Resize DB hash table                 │  Variable
├──────────────────────────────────────┤
│ Key-value pairs (type + key + value) │  Majorité du fichier
│   - Type: 1 byte                     │
│   - Key: Length-prefixed string      │
│   - Value: Encodé selon le type      │
│   - TTL: Optionnel (ms ou sec)       │
├──────────────────────────────────────┤
│ Database selector (DB 1)             │  Si plusieurs DB
│ ...                                  │
├──────────────────────────────────────┤
│ EOF marker (0xFF)                    │  1 byte
├──────────────────────────────────────┤
│ CRC64 checksum                       │  8 bytes
└──────────────────────────────────────┘
```

### Encodages optimisés

Redis utilise plusieurs encodages pour minimiser la taille du fichier :

| Type de donnée | Encodage | Gain typique |
|----------------|----------|--------------|
| **Entiers** | Varient selon taille (8/16/32 bits) | 50-75% |
| **Petites strings** | Length-prefixed sans compression | Minimal |
| **Grandes strings** | Compression LZF | 40-60% |
| **Lists courtes** | Ziplist (encodage compact) | 60-80% |
| **Hashes courts** | Ziplist | 60-80% |
| **Sets petits** | Intset (entiers) | 70-85% |
| **Sorted Sets courts** | Ziplist | 60-80% |

### Taille du fichier RDB

**Estimation approximative :**

```
Taille RDB ≈ (Taille RAM utilisée) × 0.3 à 0.7
```

Le ratio dépend de :
- Type de données (strings vs structures complexes)
- Compressibilité des données
- Nombre de clés (overhead par clé)

**Exemples réels :**

| Dataset | RAM utilisée | Taille RDB | Ratio | Type dominant |
|---------|--------------|------------|-------|---------------|
| Cache web | 10 GB | 3.2 GB | 32% | Strings compressibles (HTML/JSON) |
| Session store | 5 GB | 2.8 GB | 56% | Hashes avec JSON |
| Compteurs | 2 GB | 0.4 GB | 20% | Entiers (très compressibles) |
| Données binaires | 8 GB | 7.1 GB | 89% | Blobs non compressibles |

### Vérifier l'intégrité d'un fichier RDB

```bash
# Vérifier avec redis-check-rdb
redis-check-rdb /var/lib/redis/dump.rdb

# Sortie si OK :
[offset 0] Checking RDB file dump.rdb
[offset 27] AUX field = 'redis-ver' ('7.0.5')
[offset 45] AUX field = 'redis-bits' ('64')
...
[offset 1048576] Checksum OK
[offset 1048584] RDB looks OK!
```

**Réparation impossible** : Si le fichier RDB est corrompu, il n'existe pas de mécanisme de réparation automatique (contrairement à AOF). C'est pourquoi les backups réguliers sont critiques.

## Gestion de la mémoire et du fork

### Calcul de la mémoire nécessaire

**Règle générale :**
```
Mémoire totale requise = RAM Redis + (RAM Redis × taux de modification)
```

**Scénarios :**

| Taux de modification | RAM Redis | Mémoire COW supplémentaire | Total requis |
|---------------------|-----------|---------------------------|--------------|
| **Faible** (cache read-heavy) | 10 GB | 0.5-1 GB (5-10%) | 10.5-11 GB |
| **Moyen** (équilibré) | 10 GB | 2-4 GB (20-40%) | 12-14 GB |
| **Élevé** (write-heavy) | 10 GB | 5-8 GB (50-80%) | 15-18 GB |
| **Très élevé** (pire cas) | 10 GB | 10 GB (100%) | 20 GB |

### Optimiser la consommation mémoire pendant le fork

**1. Limiter maxmemory**
```conf
# Forcer l'éviction avant d'atteindre la limite système
maxmemory 8gb
maxmemory-policy allkeys-lru
```

**2. Augmenter la fréquence des snapshots**
```conf
# Snapshots plus fréquents = moins de modifications entre deux snapshots
save 300 10
save 60 100
```

**3. Utiliser Transparent Huge Pages (avec précaution)**
```bash
# Vérifier THP (Transparent Huge Pages)
cat /sys/kernel/mm/transparent_hugepage/enabled

# Désactiver THP (recommandé pour Redis)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

**Pourquoi désactiver THP ?**
- THP utilise des pages de 2 MB au lieu de 4 KB
- Copy-on-write copie des pages entières
- Avec THP : 2 MB copiés même pour 1 byte modifié
- Sans THP : seulement 4 KB copiés

### Durée du fork selon la taille du dataset

| RAM utilisée | Durée fork typique (SSD) | Durée fork (HDD) | Blocage perçu |
|--------------|-------------------------|------------------|---------------|
| 100 MB | <1 ms | <1 ms | Imperceptible |
| 1 GB | 1-5 ms | 5-10 ms | Imperceptible |
| 10 GB | 10-50 ms | 50-100 ms | Négligeable |
| 50 GB | 50-200 ms | 200-500 ms | Perceptible (P99) |
| 100 GB | 100-500 ms | 500ms-2s | ⚠️ Impact visible |
| 200 GB+ | 500ms-2s | 2-10s | ❌ Problématique |

**Recommandation** : Pour des datasets >100 GB, considérer Redis Cluster pour distribuer la charge.

## Gestion des erreurs RDB

### Types d'erreurs courantes

| Erreur | Cause | Solution |
|--------|-------|----------|
| **Can't save in background: fork: Cannot allocate memory** | RAM insuffisante pour fork | Augmenter RAM ou activer `vm.overcommit_memory=1` |
| **Background save terminated by signal** | OOM Killer a tué le processus | Vérifier logs système, augmenter RAM |
| **Write error saving DB on disk** | Disque plein | Libérer espace disque |
| **Bad signature trying to load DB** | Fichier RDB corrompu | Restaurer depuis backup |
| **Short read while loading DB** | Fichier RDB tronqué | Restaurer depuis backup |

### Configuration du comportement d'erreur

```conf
# Arrêter les écritures si le snapshot échoue (recommandé)
stop-writes-on-bgsave-error yes
```

**Impact de `stop-writes-on-bgsave-error yes` :**

```bash
# Si BGSAVE échoue :
redis-cli SET key value
(error) MISCONF Redis is configured to save RDB snapshots, but it is currently
not able to persist on disk. Commands that may modify the data set are disabled.

# Redis passe en read-only jusqu'à ce que le problème soit résolu
```

**Avantage** : Évite la perte de données silencieuse (données acceptées mais non persistées).

**Désavantage** : Service dégradé (lecture seule).

### Monitoring des snapshots RDB

**Commandes de diagnostic :**
```bash
# État actuel
redis-cli INFO persistence

# Métriques clés :
rdb_bgsave_in_progress:0          # 0 = pas de snapshot en cours
rdb_last_save_time:1701878400     # Timestamp du dernier snapshot
rdb_last_bgsave_status:ok         # ok ou err
rdb_last_bgsave_time_sec:3        # Durée du dernier snapshot
rdb_current_bgsave_time_sec:-1    # Durée snapshot actuel (-1 si aucun)
rdb_changes_since_last_save:1247  # Modifications depuis dernier save
```

**Alertes recommandées :**

| Métrique | Condition d'alerte | Priorité | Action |
|----------|-------------------|----------|--------|
| `rdb_last_bgsave_status` | `== err` | 🔴 Critique | Investiguer logs immédiatement |
| Temps depuis `rdb_last_save_time` | `> 2x save interval` | 🟡 Warning | Vérifier si snapshots déclenchés |
| `rdb_last_bgsave_time_sec` | `> 300s` | 🟡 Warning | Performance disque ou dataset trop gros |
| `rdb_changes_since_last_save` | Croissance continue | 🟡 Warning | Snapshots peut-être pas déclenchés |

## Stratégies de backup RDB

### Backup local

```bash
#!/bin/bash
# Script de backup quotidien

REDIS_DIR="/var/lib/redis"
BACKUP_DIR="/backup/redis"
DATE=$(date +%Y%m%d_%H%M%S)

# Déclencher un snapshot
redis-cli BGSAVE

# Attendre la fin du snapshot
while [ $(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2) -eq 1 ]; do
    sleep 1
done

# Copier le fichier RDB
cp $REDIS_DIR/dump.rdb $BACKUP_DIR/dump-$DATE.rdb

# Compresser
gzip $BACKUP_DIR/dump-$DATE.rdb

# Conserver seulement les 7 derniers jours
find $BACKUP_DIR -name "dump-*.rdb.gz" -mtime +7 -delete

echo "Backup completed: dump-$DATE.rdb.gz"
```

### Backup vers le cloud (AWS S3)

```bash
#!/bin/bash
# Backup Redis vers S3

REDIS_DIR="/var/lib/redis"
S3_BUCKET="s3://my-backups/redis"
DATE=$(date +%Y%m%d_%H%M%S)

# BGSAVE et attente
redis-cli BGSAVE
while [ $(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2) -eq 1 ]; do
    sleep 1
done

# Upload vers S3 avec compression
aws s3 cp $REDIS_DIR/dump.rdb $S3_BUCKET/dump-$DATE.rdb \
    --storage-class STANDARD_IA \
    --metadata backup-date=$DATE

echo "Backup uploaded to S3: dump-$DATE.rdb"
```

### Politique de rétention recommandée

| Type de backup | Fréquence | Rétention | Localisation | Coût typique |
|----------------|-----------|-----------|--------------|--------------|
| **Snapshot local** | Toutes les heures | 24 heures | Disque local | Faible (disque) |
| **Backup quotidien** | 1x par jour | 7 jours | NAS ou S3 Standard | Moyen |
| **Backup hebdomadaire** | 1x par semaine | 4 semaines | S3 Standard-IA | Faible |
| **Backup mensuel** | 1x par mois | 6-12 mois | S3 Glacier | Très faible |
| **Backup annuel** | 1x par an | 7 ans | S3 Deep Archive | Négligeable |

**Stratégie 3-2-1** (recommandée) :
- **3** copies des données (original + 2 backups)
- **2** types de média différents (disque local + cloud)
- **1** copie hors-site (cloud ou datacenter distant)

## Performance et optimisations

### Benchmark des performances RDB

| Dataset | Compressi on | Durée BGSAVE | Taille fichier | Débit écriture |
|---------|--------------|--------------|----------------|----------------|
| 1 GB strings | yes | 3-5s | 350 MB | 70-100 MB/s |
| 1 GB strings | no | 2-3s | 950 MB | 300-450 MB/s |
| 10 GB mixed | yes | 25-40s | 3.5 GB | 100-140 MB/s |
| 10 GB mixed | no | 15-25s | 9.2 GB | 370-600 MB/s |
| 50 GB mixed | yes | 120-200s | 18 GB | 90-150 MB/s |

**Facteurs d'influence :**
- Type de disque (SSD >> HDD)
- CPU disponible (compression)
- Taux de modification (COW)
- Fragmentation mémoire

### Optimisations recommandées

**1. Utiliser un SSD pour le répertoire de travail**
```conf
dir /mnt/ssd/redis
```

**Impact** : 3-5x plus rapide que HDD.

**2. Activer le fsync progressif (Redis 7+)**
```conf
rdb-save-incremental-fsync yes
```

**Avantage** : Lisse les pics d'I/O disque.

**3. Optimiser le système Linux**
```bash
# Activer overcommit memory (permet fork même si RAM "pleine")
sysctl vm.overcommit_memory=1
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf

# Désactiver THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Augmenter les limites de fichiers
ulimit -n 65535
```

**4. Dimensionner correctement la RAM**
```
RAM serveur >= (Dataset × 2) + 2 GB (overhead système)
```

**Exemple :**
- Dataset : 20 GB
- RAM requise : 20 × 2 + 2 = **42 GB minimum**

## Restauration depuis RDB

### Procédure de restauration

```bash
# 1. Arrêter Redis
systemctl stop redis

# 2. Remplacer le fichier RDB
cp /backup/dump-20231201.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# 3. Redémarrer Redis
systemctl start redis

# 4. Vérifier les logs
tail -f /var/log/redis/redis-server.log

# 5. Vérifier le chargement
redis-cli INFO server | grep redis_version
redis-cli DBSIZE
```

### Temps de chargement RDB

| Taille fichier RDB | Temps de chargement (SSD) | Temps de chargement (HDD) |
|-------------------|--------------------------|--------------------------|
| 100 MB | 1-2s | 2-5s |
| 1 GB | 5-10s | 15-30s |
| 10 GB | 30-60s | 2-5 min |
| 50 GB | 3-5 min | 10-20 min |
| 100 GB | 5-10 min | 20-40 min |

**Optimisations de chargement :**
- Disque SSD (3-5x plus rapide)
- Désactiver les clients pendant le chargement
- Augmenter `hz` temporairement : `CONFIG SET hz 100`

## Cas d'usage idéaux pour RDB

### ✅ Quand utiliser RDB seul

| Cas d'usage | Justification | Configuration |
|-------------|---------------|---------------|
| **Cache pur** | Reconstruction possible | `save 3600 1` |
| **Cache avec warm-up long** | Performance critique | `save 900 1` `save 300 10` |
| **Read-heavy workload** | Peu de modifications | `save 900 1` `save 300 10` |
| **Backups uniquement** | Utilisé avec AOF primary | `save 3600 1` |
| **Environnement test/dev** | Données non critiques | `save 300 10` |

### ❌ Quand NE PAS utiliser RDB seul

| Cas d'usage | Problème | Solution |
|-------------|----------|----------|
| **Données financières** | Perte 5-15min inacceptable | RDB + AOF always |
| **Job queues critiques** | Perte de tâches | RDB + AOF everysec |
| **Write-heavy workload** | Snapshots trop fréquents | RDB + AOF everysec |
| **Dataset >100 GB** | Fork trop coûteux | Redis Cluster + AOF |
| **Conformité stricte** | Audit trail requis | AOF + Réplication |

## Troubleshooting RDB

### Problème 1 : "Cannot allocate memory" lors du fork

**Symptôme :**
```
[1234] Background saving terminated by signal 11
```

**Diagnostic :**
```bash
# Vérifier la mémoire disponible
free -h
              total        used        free      shared  buff/cache   available
Mem:           15Gi        14Gi       100Mi       10Mi       1.0Gi       500Mi
                          ↑ Problème : seulement 500 MB disponibles
```

**Solutions :**

1. **Activer overcommit_memory :**
```bash
sysctl vm.overcommit_memory=1
```

2. **Augmenter la RAM :**
```
RAM requise = Dataset × 2
```

3. **Réduire le dataset :**
```conf
maxmemory 8gb
maxmemory-policy allkeys-lru
```

### Problème 2 : Snapshots qui prennent trop de temps

**Symptôme :**
```bash
redis-cli INFO persistence | grep rdb_last_bgsave_time_sec
rdb_last_bgsave_time_sec:245  # Plus de 4 minutes !
```

**Causes possibles :**
- Disque lent (HDD)
- Forte fragmentation
- Dataset très gros
- I/O disk saturés

**Solutions :**

1. **Migration vers SSD :**
```bash
# Avant (HDD) : 245 secondes
# Après (SSD) : 45 secondes
```

2. **Augmenter la fréquence :**
```conf
# Snapshots plus fréquents = moins de données à écrire
save 300 10
save 60 100
```

3. **Activer fsync progressif :**
```conf
rdb-save-incremental-fsync yes
```

### Problème 3 : Fichier RDB corrompu

**Symptôme :**
```bash
redis-server
# Fatal error loading the DB: Short read or OOM loading DB. Unrecoverable error, aborting now.
```

**Diagnostic :**
```bash
redis-check-rdb /var/lib/redis/dump.rdb
# [offset 1234567] Bad data format
```

**Solutions :**

1. **Restaurer depuis backup :**
```bash
cp /backup/dump-yesterday.rdb /var/lib/redis/dump.rdb
```

2. **Si pas de backup, tenter récupération partielle :**
```bash
# Démarrer Redis sans charger le RDB
redis-server --appendonly no --save ""

# Dans un autre terminal
redis-cli
> FLUSHALL  # Vider la base
> CONFIG SET appendonly yes
```

3. **Prévention :**
```conf
# Toujours activer le checksum
rdbchecksum yes

# Backups réguliers automatisés
# (voir section Backup)
```

## Checklist de production RDB

### Configuration optimale

```conf
# Snapshots
save 900 1
save 300 10
save 60 10000

# Fichiers
dir /var/lib/redis
dbfilename dump.rdb

# Optimisations
rdbcompression yes
rdbchecksum yes
rdb-save-incremental-fsync yes  # Redis 7+

# Sécurité
stop-writes-on-bgsave-error yes
```

### Checklist de déploiement

- [ ] RAM serveur >= Dataset × 2 + 2 GB
- [ ] Utilisation de SSD pour le répertoire de travail
- [ ] `vm.overcommit_memory=1` configuré
- [ ] Transparent Huge Pages désactivé
- [ ] Monitoring des métriques RDB (Prometheus/Grafana)
- [ ] Alertes configurées sur `rdb_last_bgsave_status`
- [ ] Script de backup automatisé et testé
- [ ] Politique de rétention définie (7j/4w/6m)
- [ ] Backups stockés hors-site (S3, etc.)
- [ ] Procédure de restauration documentée et testée

### Métriques à monitorer

| Métrique | Source | Alerte si | Priorité |
|----------|--------|-----------|----------|
| `rdb_last_bgsave_status` | INFO persistence | `!= ok` | 🔴 Critique |
| `rdb_last_save_time` | INFO persistence | `> 2x interval` | 🟡 Warning |
| `rdb_last_bgsave_time_sec` | INFO persistence | `> 300s` | 🟡 Warning |
| `rdb_changes_since_last_save` | INFO persistence | Croissance constante | 🟢 Info |
| Taille fichier RDB | Système | Croissance >50%/semaine | 🟡 Warning |
| Espace disque libre | Système | < 20% | 🔴 Critique |

## Conclusion

RDB est un mécanisme de persistance **simple, fiable et performant** pour la majorité des cas d'usage Redis. Ses points forts sont :

- ✅ **Performance** : Impact minimal sur les opérations (sauf fork)
- ✅ **Simplicité** : Un seul fichier, facile à sauvegarder
- ✅ **Compacité** : Format binaire optimisé
- ✅ **Rapidité** : Chargement très rapide au démarrage

Cependant, il présente des **limitations importantes** :

- ❌ **Perte de données** : Entre deux snapshots (5-15 minutes typiquement)
- ❌ **Consommation mémoire** : Fork peut nécessiter jusqu'à 2x la RAM
- ❌ **Coût du fork** : Sur de très gros datasets (>100 GB), peut être problématique

**Recommandation finale :** RDB seul convient pour les caches et données non-critiques. Pour la production avec des données importantes, **toujours combiner RDB + AOF** (section suivante).

---

**Points clés à retenir :**
- BGSAVE est non-bloquant grâce au Copy-on-Write
- Configurer `save` selon votre tolérance à la perte de données
- Toujours activer `rdbchecksum` et `rdbcompression`
- Dimensionner la RAM pour 2x le dataset
- Automatiser les backups et tester la restauration

---


⏭️ [AOF (Append Only File) : Log et sécurité maximale](/05-persistance-fiabilite/03-aof-log-securite-maximale.md)
