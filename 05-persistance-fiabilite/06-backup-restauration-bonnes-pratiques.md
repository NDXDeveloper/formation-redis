🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.6 Backup et restauration : Bonnes pratiques

## Introduction

Les mécanismes de persistance (RDB et AOF) protègent contre les crashs et redémarrages, mais ils **ne suffisent pas** pour une protection complète des données. Les backups sont essentiels pour se protéger contre :

- 🔥 **Corruption de données** (bug applicatif, commande destructive)
- 💾 **Défaillance matérielle** (disque, serveur, datacenter)
- 🏢 **Sinistres majeurs** (incendie, inondation, catastrophe naturelle)
- 👤 **Erreurs humaines** (suppression accidentelle, mauvaise configuration)
- 🦠 **Attaques malveillantes** (ransomware, sabotage)
- 📋 **Conformité légale** (RGPD, archivage à long terme)

### La règle 3-2-1

La stratégie de backup universellement reconnue :

```
3 copies des données
├─ 1 copy : Données en production (Redis actif)
├─ 2 copies : Backups
│   ├─ Backup 1 : Local (récupération rapide)
│   └─ Backup 2 : Distant (protection contre sinistre)

2 types de médias différents
├─ Média 1 : SSD local
└─ Média 2 : Cloud storage (S3, Azure Blob, GCS)

1 copie hors-site (off-site)
└─ Cloud ou datacenter distant
```

### RPO vs RTO : Comprendre les objectifs

| Métrique | Définition | Exemple | Impact |
|----------|------------|---------|--------|
| **RPO** (Recovery Point Objective) | Quelle perte de données maximum ? | 1 heure, 1 jour | Fréquence des backups |
| **RTO** (Recovery Time Objective) | Combien de temps pour restaurer ? | 5 min, 1 heure | Type de stockage, automatisation |

**Exemple :**
```
E-commerce :
- RPO : 1 heure (backup toutes les heures)
- RTO : 15 minutes (restauration rapide depuis backup local)
- Coût : Modéré

Finance :
- RPO : 0 (réplication continue)
- RTO : 5 minutes (failover automatique)
- Coût : Élevé
```

## Stratégies de backup par source

### Backup RDB : La méthode standard

**Avantages :**
- ✅ Fichier unique, compact (30-40% du dataset)
- ✅ Restauration très rapide (quelques secondes à minutes)
- ✅ Facile à copier, transférer, archiver
- ✅ Compatible entre versions Redis
- ✅ Pas besoin d'arrêter Redis

**Inconvénients :**
- ❌ Point-in-time du dernier snapshot uniquement
- ❌ Pas de granularité fine (tout ou rien)
- ❌ Si corrompu, tout est perdu

**Configuration pour backups optimaux :**
```conf
# Snapshots réguliers
save 3600 1       # Toutes les heures minimum
save 900 10       # Backup de sécurité
save 300 100

# Optimisations
rdbcompression yes         # Fichiers plus petits
rdbchecksum yes           # Vérification intégrité
rdb-save-incremental-fsync yes  # I/O lissés

# Nom de fichier simple
dbfilename dump.rdb
```

### Backup AOF : Le journal complet

**Avantages :**
- ✅ Granularité fine (commande par commande)
- ✅ Point-in-time recovery possible
- ✅ Réparable en cas de corruption partielle
- ✅ Format texte auditable

**Inconvénients :**
- ❌ Fichiers volumineux (3-5x la taille RDB)
- ❌ Restauration plus lente
- ❌ Nécessite plus d'espace de stockage
- ❌ Manipulation plus complexe

**Configuration pour backups :**
```conf
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes  # Format hybride (Redis 7+)

# Réécritures pour taille raisonnable
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### Tableau comparatif : RDB vs AOF pour backups

| Critère | RDB | AOF | AOF hybride |
|---------|-----|-----|-------------|
| **Taille backup** | ⭐⭐⭐⭐⭐ Compact | ⭐⭐ Volumineux | ⭐⭐⭐⭐ Compact |
| **Vitesse backup** | ⭐⭐⭐⭐⭐ Rapide | ⭐⭐⭐ Moyen | ⭐⭐⭐⭐ Rapide |
| **Vitesse restauration** | ⭐⭐⭐⭐⭐ Très rapide | ⭐⭐⭐ Moyen | ⭐⭐⭐⭐ Rapide |
| **Point-in-time recovery** | ❌ Non | ✅ Oui | ⚠️ Limité |
| **Granularité** | Snapshot complet | Par commande | Snapshot + delta |
| **Facilité transfert** | ⭐⭐⭐⭐⭐ Simple | ⭐⭐ Complexe | ⭐⭐⭐⭐ Simple |
| **Archivage long terme** | ⭐⭐⭐⭐⭐ Idéal | ⭐⭐ Moyen | ⭐⭐⭐⭐ Bon |
| **Recommandation** | ✅ Backups standard | ⚠️ Cas spécifiques | ✅ Redis 7+ |

### Recommandation : Backup combiné RDB + AOF

**Meilleure pratique :**
```bash
#!/bin/bash
# Backup combiné (optimal)

# 1. Forcer snapshot RDB (compact, rapide)
redis-cli BGSAVE
while [ $(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2) -eq 1 ]; do
    sleep 1
done

# 2. Copier RDB (backup principal)
cp /var/lib/redis/dump.rdb /backup/rdb/dump-$(date +%Y%m%d_%H%M%S).rdb

# 3. Copier AOF (point-in-time recovery)
cp /var/lib/redis/appendonly.aof /backup/aof/aof-$(date +%Y%m%d_%H%M%S).aof

# Alternative Redis 7+ (multi-part AOF)
tar czf /backup/aof/aof-$(date +%Y%m%d_%H%M%S).tar.gz \
    -C /var/lib/redis appendonlydir/
```

**Avantages du backup combiné :**
- RDB pour restauration rapide (usage principal)
- AOF pour point-in-time recovery si nécessaire
- Double protection contre corruption

## Fréquence et rétention des backups

### Matrice de fréquence par criticité

| Criticité | Fréquence backup | Rétention | RPO | Coût stockage |
|-----------|-----------------|-----------|-----|---------------|
| **Cache non-critique** | 1x/jour | 3 jours | 24h | Très faible |
| **Cache important** | 4x/jour (6h) | 7 jours | 6h | Faible |
| **Données standard** | 1x/heure | 7j + 4 semaines | 1h | Moyen |
| **Données importantes** | 1x/15min | 7j + 4w + 6m | 15min | Élevé |
| **Données critiques** | Continu (réplication) | 7j + 4w + 1an | 0-1s | Très élevé |

### Politique de rétention recommandée (GFS)

**GFS = Grandfather-Father-Son (annuel-mensuel-quotidien)**

```
Backups quotidiens (Son)
├─ Fréquence : Toutes les heures
├─ Rétention : 7 jours
├─ Stockage : Local + S3 Standard
└─ Usage : Récupération récente (<7j)

Backups hebdomadaires (Father)
├─ Fréquence : 1x/semaine (dimanche)
├─ Rétention : 4 semaines
├─ Stockage : S3 Standard-IA
└─ Usage : Récupération intermédiaire (1-4 semaines)

Backups mensuels (Grandfather)
├─ Fréquence : 1x/mois (1er du mois)
├─ Rétention : 12 mois
├─ Stockage : S3 Glacier
└─ Usage : Archivage, compliance

Backups annuels (optionnel)
├─ Fréquence : 1x/an (1er janvier)
├─ Rétention : 7 ans
├─ Stockage : S3 Deep Archive
└─ Usage : Conformité légale, audit
```

### Tableau de coût par stratégie (AWS S3)

| Stratégie | Volume/mois | Classe stockage | Coût/mois | Usage |
|-----------|-------------|-----------------|-----------|-------|
| **Quotidiens (7j)** | 50 GB × 7 = 350 GB | S3 Standard | $8 | Récupération rapide |
| **Hebdo (4 sem)** | 50 GB × 4 = 200 GB | S3 Standard-IA | $2.50 | Récupération moyenne |
| **Mensuels (12m)** | 50 GB × 12 = 600 GB | S3 Glacier Instant | $2.40 | Archivage court terme |
| **Annuels (7 ans)** | 50 GB × 7 = 350 GB | S3 Glacier Deep | $0.35 | Archivage long terme |
| **Total** | ~1.5 TB | Mixte | **~$13/mois** | - |

*Prix AWS S3 US-East-1 (indicatifs)*

## Automatisation des backups

### Script de backup complet (production-ready)

```bash
#!/bin/bash
# redis-backup.sh - Script de backup production
# Usage: ./redis-backup.sh [hourly|daily|weekly|monthly]

set -euo pipefail

# === CONFIGURATION ===
REDIS_HOST="localhost"
REDIS_PORT="6379"
REDIS_PASSWORD=""  # Optionnel

REDIS_DIR="/var/lib/redis"
BACKUP_BASE="/backup/redis"
LOG_FILE="/var/log/redis-backup.log"

# AWS S3 (optionnel)
S3_BUCKET="s3://my-company-backups/redis"
ENABLE_S3=true

# Rétention
HOURLY_RETENTION_DAYS=7
DAILY_RETENTION_DAYS=30
WEEKLY_RETENTION_DAYS=90
MONTHLY_RETENTION_DAYS=365

# === FONCTIONS ===
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

check_redis() {
    if ! redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" PING >/dev/null 2>&1; then
        log "ERROR: Redis not responding"
        exit 1
    fi
}

trigger_bgsave() {
    log "Triggering BGSAVE..."
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" BGSAVE >/dev/null

    # Attendre la fin du BGSAVE
    while [ "$(redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2 | tr -d '\r')" -eq 1 ]; do
        sleep 1
    done

    # Vérifier le statut
    STATUS=$(redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" INFO persistence | grep rdb_last_bgsave_status | cut -d: -f2 | tr -d '\r')
    if [ "$STATUS" != "ok" ]; then
        log "ERROR: BGSAVE failed"
        exit 1
    fi

    log "BGSAVE completed successfully"
}

backup_rdb() {
    local type=$1
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_dir="$BACKUP_BASE/$type"

    mkdir -p "$backup_dir"

    # Copier RDB
    local rdb_file="$REDIS_DIR/dump.rdb"
    if [ -f "$rdb_file" ]; then
        local backup_file="$backup_dir/dump-$timestamp.rdb"
        cp "$rdb_file" "$backup_file"
        gzip "$backup_file"
        log "RDB backed up: $backup_file.gz"

        # Upload vers S3 si activé
        if [ "$ENABLE_S3" = true ]; then
            aws s3 cp "$backup_file.gz" "$S3_BUCKET/$type/" \
                --storage-class STANDARD_IA \
                --metadata backup-date="$timestamp",type="$type"
            log "Uploaded to S3: $S3_BUCKET/$type/"
        fi
    else
        log "WARNING: RDB file not found"
    fi
}

backup_aof() {
    local type=$1
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_dir="$BACKUP_BASE/$type"

    mkdir -p "$backup_dir"

    # Redis 7+ (multi-part AOF)
    if [ -d "$REDIS_DIR/appendonlydir" ]; then
        local backup_file="$backup_dir/aof-$timestamp.tar.gz"
        tar czf "$backup_file" -C "$REDIS_DIR" appendonlydir/
        log "AOF (multi-part) backed up: $backup_file"

        if [ "$ENABLE_S3" = true ]; then
            aws s3 cp "$backup_file" "$S3_BUCKET/$type/"
            log "AOF uploaded to S3"
        fi
    # Redis 6 (single AOF)
    elif [ -f "$REDIS_DIR/appendonly.aof" ]; then
        local backup_file="$backup_dir/aof-$timestamp.aof"
        cp "$REDIS_DIR/appendonly.aof" "$backup_file"
        gzip "$backup_file"
        log "AOF backed up: $backup_file.gz"

        if [ "$ENABLE_S3" = true ]; then
            aws s3 cp "$backup_file.gz" "$S3_BUCKET/$type/"
        fi
    fi
}

cleanup_old_backups() {
    local type=$1
    local retention=$2
    local backup_dir="$BACKUP_BASE/$type"

    if [ -d "$backup_dir" ]; then
        find "$backup_dir" -name "*.gz" -mtime +$retention -delete
        log "Cleaned up old $type backups (>$retention days)"
    fi
}

send_notification() {
    local status=$1
    local message=$2

    # Slack webhook (exemple)
    # curl -X POST -H 'Content-type: application/json' \
    #     --data "{\"text\":\"Redis Backup [$status]: $message\"}" \
    #     https://hooks.slack.com/services/YOUR/WEBHOOK/URL

    # Email (exemple avec sendmail)
    # echo "$message" | mail -s "Redis Backup [$status]" ops@company.com

    log "Notification sent: $status - $message"
}

# === MAIN ===
BACKUP_TYPE=${1:-hourly}

log "===== Starting Redis backup ($BACKUP_TYPE) ====="

# Vérifier Redis
check_redis

# Déclencher BGSAVE
trigger_bgsave

# Backup RDB (toujours)
backup_rdb "$BACKUP_TYPE"

# Backup AOF (optionnel selon type)
if [ "$BACKUP_TYPE" = "daily" ] || [ "$BACKUP_TYPE" = "weekly" ] || [ "$BACKUP_TYPE" = "monthly" ]; then
    backup_aof "$BACKUP_TYPE"
fi

# Nettoyage
case "$BACKUP_TYPE" in
    hourly)
        cleanup_old_backups "hourly" $HOURLY_RETENTION_DAYS
        ;;
    daily)
        cleanup_old_backups "daily" $DAILY_RETENTION_DAYS
        ;;
    weekly)
        cleanup_old_backups "weekly" $WEEKLY_RETENTION_DAYS
        ;;
    monthly)
        cleanup_old_backups "monthly" $MONTHLY_RETENTION_DAYS
        ;;
esac

# Notification succès
send_notification "SUCCESS" "Backup $BACKUP_TYPE completed successfully"

log "===== Backup completed successfully ====="
```

### Configuration Cron

```bash
# /etc/cron.d/redis-backup

# Backups horaires (toutes les heures)
0 * * * * redis /opt/scripts/redis-backup.sh hourly

# Backups quotidiens (2h du matin)
0 2 * * * redis /opt/scripts/redis-backup.sh daily

# Backups hebdomadaires (dimanche 3h)
0 3 * * 0 redis /opt/scripts/redis-backup.sh weekly

# Backups mensuels (1er du mois 4h)
0 4 1 * * redis /opt/scripts/redis-backup.sh monthly
```

### Monitoring des backups

```bash
#!/bin/bash
# check-backup-health.sh

BACKUP_DIR="/backup/redis/hourly"
MAX_AGE_HOURS=2

LATEST_BACKUP=$(find "$BACKUP_DIR" -name "*.gz" -type f -printf '%T@ %p\n' | sort -n | tail -1 | cut -d' ' -f2)

if [ -z "$LATEST_BACKUP" ]; then
    echo "CRITICAL: No backup found in $BACKUP_DIR"
    exit 2
fi

BACKUP_AGE_SECONDS=$(( $(date +%s) - $(stat -c %Y "$LATEST_BACKUP") ))
BACKUP_AGE_HOURS=$(( BACKUP_AGE_SECONDS / 3600 ))

if [ $BACKUP_AGE_HOURS -gt $MAX_AGE_HOURS ]; then
    echo "WARNING: Latest backup is $BACKUP_AGE_HOURS hours old (max: $MAX_AGE_HOURS)"
    exit 1
else
    echo "OK: Latest backup is $BACKUP_AGE_HOURS hour(s) old"
    exit 0
fi
```

## Stockage des backups

### Tableau comparatif des options de stockage

| Type de stockage | RPO | RTO | Coût | Résilience | Usage |
|-----------------|-----|-----|------|------------|-------|
| **Disque local (même serveur)** | 1h | 1 min | Faible | ⭐ | ❌ Non recommandé seul |
| **NAS local** | 1h | 5 min | Moyen | ⭐⭐ | ⚠️ Protection partielle |
| **Disque serveur distant** | 1h | 10 min | Moyen | ⭐⭐⭐ | ✅ Bon |
| **S3 / Blob Storage** | 1h | 30 min | Faible | ⭐⭐⭐⭐⭐ | ✅ **Recommandé** |
| **Multi-région cloud** | 15 min | 1h | Élevé | ⭐⭐⭐⭐⭐ | ✅ Haute disponibilité |
| **Glacier / Archive** | - | 1-12h | Très faible | ⭐⭐⭐⭐⭐ | ✅ Archivage |

### Architecture de stockage recommandée

```
Production Redis
    ↓ (backup toutes les heures)
    ├─ Disque local (SSD)
    │   └─ Backups 24 dernières heures
    │       └─ Usage : Récupération ultra-rapide
    │
    ├─ S3 Standard (même région)
    │   └─ Backups 7 derniers jours
    │       └─ Usage : Récupération rapide
    │
    ├─ S3 Standard-IA (multi-région)
    │   └─ Backups 30 derniers jours
    │       └─ Usage : Protection régionale
    │
    └─ S3 Glacier
        └─ Backups >30 jours (12 mois)
            └─ Usage : Archivage, compliance
```

### Configuration multi-région AWS

```bash
#!/bin/bash
# Backup multi-région

PRIMARY_REGION="us-east-1"
DR_REGION="eu-west-1"

PRIMARY_BUCKET="s3://company-redis-backup-use1"
DR_BUCKET="s3://company-redis-backup-euw1"

# 1. Backup vers région primaire
aws s3 cp dump.rdb.gz "$PRIMARY_BUCKET/$(date +%Y%m%d)/" \
    --region $PRIMARY_REGION

# 2. Réplication vers région DR
aws s3 cp dump.rdb.gz "$DR_BUCKET/$(date +%Y%m%d)/" \
    --region $DR_REGION

# Alternative : S3 Cross-Region Replication (automatique)
# Configuration S3 bucket avec règle de réplication
```

### Chiffrement des backups

```bash
#!/bin/bash
# Backup avec chiffrement GPG

# 1. Backup RDB
redis-cli BGSAVE
# ... attendre fin BGSAVE ...

# 2. Chiffrement
gpg --batch --yes \
    --passphrase-file /etc/redis/backup.key \
    --symmetric \
    --cipher-algo AES256 \
    --output dump-$(date +%Y%m%d).rdb.gpg \
    /var/lib/redis/dump.rdb

# 3. Upload chiffré
aws s3 cp dump-$(date +%Y%m%d).rdb.gpg \
    s3://my-secure-backups/redis/ \
    --server-side-encryption AES256

# Restauration :
# gpg --decrypt dump-20231215.rdb.gpg > dump.rdb
```

## Procédures de restauration

### Restauration standard depuis RDB

```bash
#!/bin/bash
# restore-from-rdb.sh

set -e

BACKUP_FILE=$1
REDIS_DIR="/var/lib/redis"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.rdb.gz>"
    exit 1
fi

echo "=== Redis Restauration depuis RDB ==="

# 1. Vérifier que le backup existe
if [ ! -f "$BACKUP_FILE" ]; then
    echo "ERROR: Backup file not found: $BACKUP_FILE"
    exit 1
fi

# 2. Arrêter Redis
echo "Stopping Redis..."
systemctl stop redis

# 3. Backup de l'état actuel (sécurité)
echo "Backing up current state..."
if [ -f "$REDIS_DIR/dump.rdb" ]; then
    mv "$REDIS_DIR/dump.rdb" "$REDIS_DIR/dump.rdb.before-restore"
fi

# 4. Décompresser et copier le backup
echo "Restoring backup..."
if [[ "$BACKUP_FILE" == *.gz ]]; then
    gunzip -c "$BACKUP_FILE" > "$REDIS_DIR/dump.rdb"
else
    cp "$BACKUP_FILE" "$REDIS_DIR/dump.rdb"
fi

# 5. Ajuster les permissions
chown redis:redis "$REDIS_DIR/dump.rdb"
chmod 640 "$REDIS_DIR/dump.rdb"

# 6. Vérifier l'intégrité du RDB
echo "Checking RDB integrity..."
if ! redis-check-rdb "$REDIS_DIR/dump.rdb"; then
    echo "ERROR: RDB file is corrupted"
    echo "Restoring previous state..."
    mv "$REDIS_DIR/dump.rdb.before-restore" "$REDIS_DIR/dump.rdb"
    exit 1
fi

# 7. Démarrer Redis
echo "Starting Redis..."
systemctl start redis

# 8. Attendre que Redis soit prêt
echo "Waiting for Redis to be ready..."
for i in {1..30}; do
    if redis-cli PING >/dev/null 2>&1; then
        break
    fi
    sleep 1
done

# 9. Vérifier
if redis-cli PING >/dev/null 2>&1; then
    KEYS=$(redis-cli DBSIZE | cut -d: -f2)
    echo "SUCCESS: Redis restored with $KEYS keys"

    # Nettoyer le backup temporaire
    rm -f "$REDIS_DIR/dump.rdb.before-restore"
else
    echo "ERROR: Redis failed to start"
    exit 1
fi
```

### Restauration depuis AOF

```bash
#!/bin/bash
# restore-from-aof.sh

BACKUP_AOF=$1
REDIS_DIR="/var/lib/redis"

echo "=== Redis Restauration depuis AOF ==="

# 1. Arrêter Redis
systemctl stop redis

# 2. Backup de l'état actuel
if [ -f "$REDIS_DIR/appendonly.aof" ]; then
    mv "$REDIS_DIR/appendonly.aof" "$REDIS_DIR/appendonly.aof.before-restore"
fi

# 3. Restaurer AOF
if [[ "$BACKUP_AOF" == *.tar.gz ]]; then
    # Redis 7+ (multi-part)
    tar xzf "$BACKUP_AOF" -C "$REDIS_DIR"
else
    # Redis 6 (single file)
    gunzip -c "$BACKUP_AOF" > "$REDIS_DIR/appendonly.aof"
fi

# 4. Vérifier intégrité AOF
echo "Checking AOF integrity..."
if [ -f "$REDIS_DIR/appendonly.aof" ]; then
    redis-check-aof --fix "$REDIS_DIR/appendonly.aof"
fi

# 5. Ajuster permissions
chown -R redis:redis "$REDIS_DIR"

# 6. Configuration Redis pour charger AOF
cat > /tmp/redis-restore.conf <<EOF
dir $REDIS_DIR
appendonly yes
aof-load-truncated yes
EOF

# 7. Démarrer Redis
redis-server /tmp/redis-restore.conf --daemonize yes

# 8. Vérifier
sleep 5
if redis-cli PING >/dev/null 2>&1; then
    echo "SUCCESS: Redis restored from AOF"
else
    echo "ERROR: Failed to restore from AOF"
    exit 1
fi
```

### Tableau de temps de restauration

| Taille dataset | Source | Temps (SSD) | Temps (HDD) | RTO visé |
|----------------|--------|-------------|-------------|----------|
| **1 GB** | RDB local | 5-10s | 15-30s | <1 min |
| **10 GB** | RDB local | 30-60s | 2-5 min | <5 min |
| **50 GB** | RDB local | 3-5 min | 10-20 min | <15 min |
| **100 GB** | RDB local | 6-10 min | 20-40 min | <30 min |
| **1 GB** | RDB S3 | 1-2 min | 2-3 min | <5 min |
| **10 GB** | RDB S3 | 5-10 min | 10-15 min | <15 min |
| **50 GB** | RDB S3 | 20-30 min | 40-60 min | <1h |

**Facteurs d'influence :**
- Bande passante réseau (si S3/remote)
- Type de disque (SSD vs HDD)
- Charge serveur
- Compression activée ou non

## Point-in-time recovery (PITR)

### Principe du PITR avec AOF

```
Scénario : Suppression accidentelle à 14h30

Timeline :
09:00 - Backup RDB quotidien
14:29 - État OK
14:30 - FLUSHALL accidentel (💥 toutes les données supprimées)
14:31 - Détection du problème

Recovery :
1. Charger backup RDB 09:00 (état du matin)
2. Rejouer AOF de 09:00 à 14:29 (avant le FLUSHALL)
3. Résultat : Données restaurées à 14:29
```

### Script de PITR

```bash
#!/bin/bash
# point-in-time-recovery.sh
# Restaurer jusqu'à une date/heure spécifique

TARGET_TIME=$1  # Format: "2023-12-15 14:29:00"
RDB_BACKUP="/backup/redis/daily/dump-20231215_090000.rdb.gz"
AOF_BACKUP="/backup/redis/daily/aof-20231215_090000.aof.gz"
REDIS_DIR="/var/lib/redis"

echo "=== Point-in-Time Recovery ==="
echo "Target time: $TARGET_TIME"

# 1. Arrêter Redis
systemctl stop redis

# 2. Restaurer RDB (état de base)
echo "Restoring base RDB..."
gunzip -c "$RDB_BACKUP" > "$REDIS_DIR/dump.rdb"

# 3. Filtrer AOF jusqu'au point dans le temps
echo "Filtering AOF commands until $TARGET_TIME..."

# Convertir TARGET_TIME en timestamp
TARGET_TS=$(date -d "$TARGET_TIME" +%s)

# Extraire commandes AOF jusqu'au timestamp
gunzip -c "$AOF_BACKUP" | awk -v target="$TARGET_TS" '
/^#TS:/ {
    timestamp = substr($0, 5)
    if (timestamp > target) {
        exit
    }
}
{
    print
}
' > "$REDIS_DIR/appendonly.aof"

# 4. Configurer Redis pour charger AOF
cat > /tmp/redis-pitr.conf <<EOF
dir $REDIS_DIR
appendonly yes
aof-load-truncated yes
EOF

# 5. Démarrer Redis
redis-server /tmp/redis-pitr.conf --daemonize yes

# 6. Vérifier
sleep 5
if redis-cli PING >/dev/null 2>&1; then
    KEYS=$(redis-cli DBSIZE | cut -d: -f2)
    echo "SUCCESS: Restored $KEYS keys to $TARGET_TIME"
else
    echo "ERROR: Recovery failed"
    exit 1
fi
```

**Note :** Le PITR nécessite que l'AOF contienne des timestamps. Dans Redis 7+, activer :
```conf
aof-timestamp-enabled yes
```

## Disaster Recovery (DR)

### Plan de reprise d'activité

#### Niveau 1 : Perte d'un serveur Redis

**RTO : 5-15 minutes**

```
Scénario : Serveur Redis principal down

Action immédiate :
1. Sentinel détecte la panne (30s)
2. Failover automatique vers replica (1-2 min)
3. Applications redirigées automatiquement
4. Durée totale : 2-3 minutes

Backup : Pas nécessaire (haute disponibilité)
```

#### Niveau 2 : Perte d'un datacenter

**RTO : 30-60 minutes**

```
Scénario : Datacenter entier indisponible

Action :
1. Basculer vers datacenter DR (manuel ou auto)
2. Récupérer derniers backups (S3 multi-région)
3. Restaurer Redis dans DC secondaire
4. Rediriger applications
5. Durée : 30-60 minutes

Configuration requise :
- Backups répliqués multi-région
- Infrastructure DR pré-provisionnée
- Runbooks documentés et testés
```

#### Niveau 3 : Corruption de données

**RTO : Variable (15 min - 2 heures)**

```
Scénario : Bug applicatif, mauvaise commande, ransomware

Action :
1. Identifier quand la corruption a eu lieu
2. Sélectionner backup approprié (avant corruption)
3. Restaurer depuis backup
4. Si PITR nécessaire, utiliser AOF
5. Valider l'intégrité des données
6. Remettre en service

Durée : Dépend de la taille et de la méthode
```

### Architecture DR recommandée

```
Région Primaire (US-East)
├── Redis Cluster (3 masters + 3 replicas)
│   └── Backups toutes les heures → S3 US-East
│       └── Réplication S3 → S3 EU-West (async)
│
└── Sentinel (3 instances)

Région DR (EU-West)
├── Redis Cluster (3 masters + 3 replicas)
│   └── Réplication depuis Région Primaire (optional)
│       └── Ou : Restauration depuis S3 EU-West
│
└── Sentinel (3 instances)

Basculement DR :
1. Détection panne Région Primaire
2. Promotion Région DR en Primary
3. Update DNS / Load Balancer
4. Applications reconnectent automatiquement
```

### Tableau de stratégies DR

| Stratégie | RTO | RPO | Coût | Complexité | Usage |
|-----------|-----|-----|------|------------|-------|
| **Backup + restore** | 30-60 min | 1h | Faible | ⭐⭐ | Startup, non-critique |
| **Warm standby** | 15-30 min | 1 min | Moyen | ⭐⭐⭐ | PME, standard |
| **Hot standby** | 5-15 min | 0-1s | Élevé | ⭐⭐⭐⭐ | Entreprise, critique |
| **Active-Active** | 0 (transparent) | 0 | Très élevé | ⭐⭐⭐⭐⭐ | Finance, gaming |

## Tests de restauration

### Pourquoi tester ?

**Statistique réelle :** 30-40% des backups échouent lors de la première restauration réelle.

**Causes courantes :**
- Fichier corrompu non détecté
- Permissions incorrectes
- Dépendances manquantes
- Procédure obsolète
- Erreur humaine

### Planning de tests recommandé

| Type de test | Fréquence | Durée | Qui ? |
|--------------|-----------|-------|-------|
| **Test basique** | Hebdomadaire | 15 min | Automatisé |
| **Test complet** | Mensuel | 1-2h | Équipe ops |
| **Drill DR complet** | Trimestriel | 4-8h | Toute l'équipe |
| **Drill catastrophe** | Annuel | 1 jour | Toute l'organisation |

### Script de test automatisé

```bash
#!/bin/bash
# test-restore-weekly.sh

set -e

echo "=== Weekly Restore Test ==="
DATE=$(date +%Y%m%d)
LOG_FILE="/var/log/redis-restore-tests/test-$DATE.log"

{
    echo "Test started at $(date)"

    # 1. Récupérer dernier backup
    LATEST_BACKUP=$(find /backup/redis/daily -name "*.rdb.gz" | sort | tail -1)
    echo "Testing backup: $LATEST_BACKUP"

    # 2. Restaurer dans Redis de test (port 6380)
    TEST_DIR="/tmp/redis-test-$$"
    mkdir -p "$TEST_DIR"

    gunzip -c "$LATEST_BACKUP" > "$TEST_DIR/dump.rdb"

    # 3. Démarrer Redis test
    redis-server --port 6380 \
                 --dir "$TEST_DIR" \
                 --dbfilename dump.rdb \
                 --daemonize yes \
                 --pidfile "$TEST_DIR/redis-test.pid"

    # 4. Attendre démarrage
    sleep 5

    # 5. Tests de validation
    if redis-cli -p 6380 PING >/dev/null 2>&1; then
        KEYS=$(redis-cli -p 6380 DBSIZE | cut -d: -f2)
        echo "✓ Restore successful: $KEYS keys loaded"

        # Test d'intégrité (quelques clés au hasard)
        SAMPLE_KEYS=$(redis-cli -p 6380 --scan --count 10 | head -5)
        for key in $SAMPLE_KEYS; do
            redis-cli -p 6380 GET "$key" >/dev/null
        done
        echo "✓ Integrity check passed"

        TEST_STATUS="SUCCESS"
    else
        echo "✗ Restore failed"
        TEST_STATUS="FAILED"
    fi

    # 6. Nettoyage
    redis-cli -p 6380 SHUTDOWN NOSAVE
    rm -rf "$TEST_DIR"

    echo "Test completed at $(date)"
    echo "Status: $TEST_STATUS"

} | tee "$LOG_FILE"

# 7. Notification
if [ "$TEST_STATUS" = "FAILED" ]; then
    # Alerter l'équipe
    echo "Restore test FAILED" | mail -s "ALERT: Redis Restore Test Failed" ops@company.com
fi
```

### Checklist de test de restauration

#### Test basique (15 min)

- [ ] Récupérer dernier backup
- [ ] Vérifier intégrité fichier (checksum)
- [ ] Restaurer dans environnement de test
- [ ] Vérifier nombre de clés
- [ ] Tester quelques opérations (GET/SET)
- [ ] Documenter résultat

#### Test complet (1-2h)

- [ ] Test basique
- [ ] Tester tous les types de données
- [ ] Mesurer temps de restauration
- [ ] Tester depuis plusieurs sources (local, S3, DR)
- [ ] Tester point-in-time recovery
- [ ] Vérifier logs d'erreurs
- [ ] Valider performances post-restauration
- [ ] Documenter écarts ou problèmes

#### Drill DR complet (4-8h)

- [ ] Simulation panne datacenter primaire
- [ ] Récupération backups depuis région DR
- [ ] Restauration complète dans DC secondaire
- [ ] Tests de charge
- [ ] Basculement applications
- [ ] Vérification intégrité données
- [ ] Mesure RTO/RPO réels
- [ ] Retour en arrière (rollback)
- [ ] Rapport post-mortem
- [ ] Mise à jour runbooks

## Backup de configurations

### Ne pas oublier les configs !

```bash
#!/bin/bash
# backup-redis-config.sh

CONFIG_DIR="/etc/redis"
BACKUP_DIR="/backup/redis-configs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# Backup de toutes les configs
tar czf "$BACKUP_DIR/redis-configs-$TIMESTAMP.tar.gz" \
    "$CONFIG_DIR" \
    /etc/systemd/system/redis*.service \
    /etc/security/limits.d/*redis* \
    /etc/sysctl.d/*redis*

# Upload vers S3
aws s3 cp "$BACKUP_DIR/redis-configs-$TIMESTAMP.tar.gz" \
    s3://my-backups/redis-configs/

# Rétention 90 jours
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +90 -delete
```

### Versioning des configurations

```bash
# Utiliser Git pour versionner les configs
cd /etc/redis
git init
git add .
git commit -m "Initial Redis configuration"

# À chaque modification
git commit -am "Update maxmemory to 16GB"
git push origin main

# Récupération d'une ancienne version
git checkout HEAD~5 -- redis.conf
```

## Conformité et réglementations

### Exigences par réglementation

| Réglementation | Rétention min | Chiffrement | Géo-restrictions | Audit |
|----------------|---------------|-------------|------------------|-------|
| **RGPD** | 30j-5ans (selon contexte) | ✅ Obligatoire | ✅ UE uniquement | ✅ Logs accès |
| **SOX** | 7 ans | ✅ Obligatoire | ⚠️ Selon données | ✅ Immutabilité |
| **HIPAA** | 6 ans | ✅ Obligatoire | ✅ US uniquement | ✅ Complet |
| **PCI-DSS** | 1 an min | ✅ Obligatoire | ⚠️ Selon région | ✅ Quarterly |

### Configuration pour conformité RGPD

```bash
#!/bin/bash
# Backup conforme RGPD

# 1. Chiffrement obligatoire
gpg --encrypt --recipient ops@company.eu dump.rdb

# 2. Stockage dans UE uniquement
aws s3 cp dump.rdb.gpg s3://eu-backup/ \
    --region eu-west-1 \
    --metadata data-classification=personal,retention=5years

# 3. Logs d'accès
echo "$(date) - Backup created by $USER" >> /var/log/redis-backup-audit.log

# 4. Suppression automatique après rétention
# (S3 Lifecycle Policy configurée pour 5 ans)
```

## Checklist de production

### Configuration des backups

- [ ] RDB activé avec snapshots réguliers
- [ ] AOF activé (pour PITR si nécessaire)
- [ ] Script de backup automatisé et testé
- [ ] Cron jobs configurés (hourly/daily/weekly/monthly)
- [ ] Politique de rétention définie et appliquée
- [ ] Chiffrement activé (GPG ou S3 SSE)

### Stockage

- [ ] Backups locaux (24h) sur SSD
- [ ] Backups S3/Cloud (7j minimum)
- [ ] Backups multi-région (DR)
- [ ] Archivage long terme (Glacier)
- [ ] Espace disque suffisant (5x dataset)
- [ ] Monitoring espace disque

### Restauration

- [ ] Procédure de restauration documentée
- [ ] Scripts de restauration testés
- [ ] RTO/RPO mesurés et validés
- [ ] Point-in-time recovery testé
- [ ] Plan DR documenté et testé
- [ ] Runbooks à jour

### Tests

- [ ] Tests automatisés hebdomadaires
- [ ] Tests manuels mensuels
- [ ] Drill DR trimestriel
- [ ] Résultats documentés
- [ ] Améliorations identifiées et appliquées

### Monitoring

- [ ] Monitoring succès/échecs backups
- [ ] Alertes sur backups manquants (>2h)
- [ ] Monitoring taille backups
- [ ] Monitoring temps backups
- [ ] Logs centralisés
- [ ] Dashboard Grafana

### Sécurité et conformité

- [ ] Backups chiffrés
- [ ] Accès restreints (RBAC)
- [ ] Audit logs activés
- [ ] Conformité réglementaire validée
- [ ] Rétention selon obligations légales
- [ ] Procédure suppression sécurisée

## Conclusion

Les backups sont la **dernière ligne de défense** pour vos données Redis. Même avec RDB + AOF + Réplication, les backups sont indispensables pour protéger contre :

- Corruptions de données
- Erreurs humaines
- Défaillances matérielles majeures
- Sinistres et catastrophes

### Règles d'or des backups Redis

1. **La règle 3-2-1** : 3 copies, 2 médias, 1 hors-site
2. **Automatiser** : Les backups manuels sont oubliés
3. **Tester régulièrement** : Un backup non testé est un backup qui n'existe pas
4. **Monitorer** : Alertes sur échecs, backups manquants
5. **Documenter** : Procédures claires, runbooks à jour
6. **Chiffrer** : Protection des données sensibles
7. **Retenir selon la loi** : Conformité réglementaire

### Configuration minimale viable (MVP)

```conf
# Redis
save 900 1
appendonly yes
appendfsync everysec

# Cron
0 * * * * /opt/scripts/redis-backup.sh hourly
0 2 * * * /opt/scripts/redis-backup.sh daily

# Stockage
- Local : 24 heures
- S3 : 30 jours
- Glacier : 12 mois

# Tests
- Hebdomadaire : Automatisé
- Mensuel : Manuel complet
```

**Rappel** : Le meilleur backup est celui que vous n'aurez jamais besoin d'utiliser, mais le pire cauchemar est d'en avoir besoin sans en avoir.

---

**Points clés à retenir :**
- Backups ≠ Persistance (RDB/AOF protègent contre crashs, backups contre catastrophes)
- La règle 3-2-1 est incontournable
- Tester régulièrement est aussi important que faire les backups
- RDB pour backups (compact, rapide), AOF pour PITR si nécessaire
- Cloud storage (S3) recommandé pour résilience et coût
- Toujours mesurer et valider RTO/RPO réels

---


⏭️ [Patterns de développement avancés](/06-patterns-developpement-avances/README.md)
