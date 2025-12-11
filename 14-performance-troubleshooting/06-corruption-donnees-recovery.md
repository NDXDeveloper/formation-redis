🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.6 - Corruption de données et recovery

## 🎯 Objectifs de cette section

- Comprendre les différents types de corruption dans Redis
- Détecter rapidement une corruption de données
- Utiliser les outils natifs de diagnostic et réparation
- Mettre en place des procédures de recovery efficaces
- Prévenir les corruptions par une stratégie robuste

---

## 📚 Introduction : La corruption dans Redis

### Qu'est-ce qu'une corruption ?

Une **corruption de données** survient quand les données stockées deviennent incohérentes, illisibles ou invalides.

```
┌─────────────────────────────────────────────────┐
│  TYPES DE CORRUPTION                            │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Corruption MÉMOIRE (runtime)                │
│     ├─ Données en RAM corrompues                │
│     ├─ Structures internes invalides            │
│     └─ Impact : Perte de données en cours       │
│                                                 │
│  2. Corruption RDB (snapshot)                   │
│     ├─ Fichier dump.rdb invalide                │
│     ├─ Checksum incorrect                       │
│     └─ Impact : Restauration impossible         │
│                                                 │
│  3. Corruption AOF (log)                        │
│     ├─ Fichier appendonly.aof tronqué           │
│     ├─ Commandes invalides/incomplètes          │
│     └─ Impact : Replay échoue                   │
│                                                 │
│  4. Corruption RÉPLICATION                      │
│     ├─ Désynchronisation master/replica         │
│     ├─ Données différentes                      │
│     └─ Impact : Inconsistance du cluster        │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Causes de corruption

| Cause | Type | Fréquence | Gravité |
|-------|------|-----------|---------|
| **Crash système** | RDB/AOF | Moyenne | Élevée |
| **Disque plein** | AOF | Élevée | Moyenne |
| **Bug Redis** | Mémoire | Très rare | Critique |
| **Matériel défaillant** | Tous | Faible | Critique |
| **OOM killer** | Mémoire/Fichiers | Moyenne | Élevée |
| **Modification manuelle** | RDB/AOF | Rare | Critique |
| **Corruption secteur disque** | RDB/AOF | Rare | Élevée |
| **Coupure électrique** | RDB/AOF | Rare | Élevée |

### Impact et symptômes

**Symptômes de corruption** :

```bash
# 1. Redis refuse de démarrer
$ systemctl start redis
# Job for redis.service failed...

$ journalctl -u redis | tail
# "Bad file format reading the append only file"
# "RDB checksum error"
# "Short read or OOM loading DB"

# 2. Erreurs durant l'exécution
redis-cli GET key
# (error) MISCONF Redis is configured to save RDB snapshots

# 3. Commandes qui crashent Redis
redis-cli BGSAVE
# Redis server went away

# 4. Réplication qui échoue
redis-cli INFO replication
# master_link_status:down
# master_sync_in_progress:0
```

---

## 🔍 Détection de corruption

### Vérification au démarrage

Redis effectue automatiquement des vérifications au démarrage :

```bash
# Logs de démarrage normaux
$ tail -f /var/log/redis/redis-server.log

# ✅ Démarrage sain
* DB loaded from disk: 0.532 seconds
* Ready to accept connections

# ❌ Corruption RDB détectée
# Short read or OOM loading DB. Unrecoverable error, aborting now.
# Bad file format reading the append only file: make a backup of your AOF file

# ❌ Checksum invalide
# RDB file was saved with checksum disabled: no check performed.
# Wrong signature trying to load DB from file
```

### Script de vérification manuelle

```bash
#!/bin/bash
# check-redis-integrity.sh

echo "=== REDIS DATA INTEGRITY CHECK ==="
echo ""

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

ISSUES=0

# 1. Vérifier si Redis tourne
echo "--- Redis Service Status ---"
if systemctl is-active --quiet redis; then
    echo -e "${GREEN}✅ Redis is running${NC}"
else
    echo -e "${RED}❌ Redis is NOT running${NC}"
    ISSUES=$((ISSUES + 1))
fi
echo ""

# 2. Vérifier le fichier RDB
echo "--- RDB File Check ---"
RDB_FILE="/var/lib/redis/dump.rdb"

if [ -f "$RDB_FILE" ]; then
    echo "RDB file exists: $RDB_FILE"
    ls -lh "$RDB_FILE"

    # Vérifier avec redis-check-rdb
    if command -v redis-check-rdb &> /dev/null; then
        echo ""
        echo "Running redis-check-rdb..."
        redis-check-rdb "$RDB_FILE"

        if [ $? -eq 0 ]; then
            echo -e "${GREEN}✅ RDB file is valid${NC}"
        else
            echo -e "${RED}❌ RDB file is CORRUPTED${NC}"
            ISSUES=$((ISSUES + 1))
        fi
    else
        echo -e "${YELLOW}⚠️  redis-check-rdb not found${NC}"
    fi
else
    echo -e "${YELLOW}⚠️  No RDB file found${NC}"
fi
echo ""

# 3. Vérifier le fichier AOF
echo "--- AOF File Check ---"
AOF_FILE="/var/lib/redis/appendonly.aof"

if [ -f "$AOF_FILE" ]; then
    echo "AOF file exists: $AOF_FILE"
    ls -lh "$AOF_FILE"

    # Vérifier avec redis-check-aof
    if command -v redis-check-aof &> /dev/null; then
        echo ""
        echo "Running redis-check-aof..."
        redis-check-aof "$AOF_FILE"

        if [ $? -eq 0 ]; then
            echo -e "${GREEN}✅ AOF file is valid${NC}"
        else
            echo -e "${RED}❌ AOF file is CORRUPTED${NC}"
            ISSUES=$((ISSUES + 1))
        fi
    else
        echo -e "${YELLOW}⚠️  redis-check-aof not found${NC}"
    fi
else
    echo -e "${YELLOW}⚠️  No AOF file found (may be disabled)${NC}"
fi
echo ""

# 4. Vérifier la cohérence des données en mémoire
if systemctl is-active --quiet redis; then
    echo "--- In-Memory Data Check ---"

    # Tester des commandes basiques
    if redis-cli PING > /dev/null 2>&1; then
        echo -e "${GREEN}✅ Redis responds to PING${NC}"
    else
        echo -e "${RED}❌ Redis does NOT respond${NC}"
        ISSUES=$((ISSUES + 1))
    fi

    # Vérifier DBSIZE
    DBSIZE=$(redis-cli DBSIZE 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}✅ DBSIZE: $DBSIZE keys${NC}"
    else
        echo -e "${RED}❌ Cannot get DBSIZE${NC}"
        ISSUES=$((ISSUES + 1))
    fi

    # Vérifier INFO
    if redis-cli INFO server > /dev/null 2>&1; then
        echo -e "${GREEN}✅ INFO command works${NC}"
    else
        echo -e "${RED}❌ INFO command fails${NC}"
        ISSUES=$((ISSUES + 1))
    fi
fi
echo ""

# 5. Vérifier l'espace disque
echo "--- Disk Space Check ---"
DISK_USAGE=$(df -h /var/lib/redis | awk 'NR==2 {print $5}' | sed 's/%//')

if [ "$DISK_USAGE" -lt 90 ]; then
    echo -e "${GREEN}✅ Disk usage: ${DISK_USAGE}%${NC}"
else
    echo -e "${RED}❌ Disk usage CRITICAL: ${DISK_USAGE}%${NC}"
    ISSUES=$((ISSUES + 1))
fi
echo ""

# Résumé
echo "==================================="
if [ $ISSUES -eq 0 ]; then
    echo -e "${GREEN}✅ All checks passed!${NC}"
    exit 0
else
    echo -e "${RED}❌ Found $ISSUES issue(s)${NC}"
    exit 1
fi
```

---

## 🔧 Outils de diagnostic et réparation

### redis-check-rdb : Vérification et réparation RDB

#### Utilisation

```bash
# Vérification simple
redis-check-rdb /var/lib/redis/dump.rdb

# Output normal :
# [offset 0] Checking RDB file /var/lib/redis/dump.rdb
# [offset 26] AUX FIELD redis-ver = '7.0.0'
# [offset 40] AUX FIELD redis-bits = '64'
# ...
# [offset 12345] Checksum OK
# [offset 12345] \o/ RDB looks OK! \o/

# Output avec corruption :
# [offset 5432] RDB file was saved with checksum disabled: no check performed
# [offset 5678] Short read or OOM loading DB
```

#### Réparation automatique (limitée)

```bash
# redis-check-rdb ne répare PAS automatiquement
# Il identifie seulement les erreurs

# Pour récupérer partiellement :
redis-check-rdb --fix /var/lib/redis/dump.rdb
```

⚠️ **Attention** : `--fix` n'existe pas dans redis-check-rdb. La réparation est manuelle.

#### Extraction partielle d'un RDB corrompu

```python
#!/usr/bin/env python3
"""
Extraction partielle d'un fichier RDB corrompu
Utilise redis-rdb-tools
"""
import subprocess
import sys

def extract_partial_rdb(rdb_file, output_file):
    """
    Extrait autant de données que possible d'un RDB corrompu
    """
    print(f"Extracting data from: {rdb_file}")

    # Utiliser rdb pour extraire en JSON (ignore les erreurs)
    cmd = [
        'rdb',
        '--command', 'json',
        '--file', output_file,
        rdb_file
    ]

    try:
        result = subprocess.run(cmd, capture_output=True, text=True)

        if result.returncode == 0:
            print(f"✅ Successfully extracted to: {output_file}")
        else:
            print(f"⚠️  Partial extraction completed with errors")
            print(f"Output: {output_file}")
            if result.stderr:
                print(f"Errors: {result.stderr}")

    except FileNotFoundError:
        print("❌ redis-rdb-tools not installed")
        print("Install with: pip install rdbtools")
        sys.exit(1)

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: python3 extract_rdb.py <input.rdb> <output.json>")
        sys.exit(1)

    extract_partial_rdb(sys.argv[1], sys.argv[2])
```

### redis-check-aof : Vérification et réparation AOF

#### Utilisation basique

```bash
# Vérification
redis-check-aof /var/lib/redis/appendonly.aof

# Output normal :
# AOF analyzed: size=104857600, ok_up_to=104857600, ok_up_to_line=5000000, diff=0
# AOF is valid

# Output avec corruption :
# AOF analyzed: size=104857600, ok_up_to=104805432, ok_up_to_line=4998765, diff=52168
# AOF is not valid. Use the --fix option to try fixing it.
```

#### Réparation automatique

```bash
# Réparer l'AOF (FAIT UN BACKUP AUTOMATIQUE)
redis-check-aof --fix /var/lib/redis/appendonly.aof

# Output :
# AOF analyzed: size=104857600, ok_up_to=104805432, diff=52168
# This will shrink the AOF from 104857600 bytes, with 52168 bytes, to 104805432 bytes
# Continue? [y/N]: y
# Successfully truncated AOF
```

**Ce que fait `--fix`** :
1. Crée un backup : `appendonly.aof.bak`
2. Tronque le fichier à la dernière position valide
3. Supprime les commandes corrompues

⚠️ **Perte de données** : Les commandes après la corruption sont perdues !

#### Script automatisé de réparation AOF

```bash
#!/bin/bash
# repair-aof.sh

AOF_FILE="/var/lib/redis/appendonly.aof"
BACKUP_DIR="/var/backups/redis"

echo "=== AOF REPAIR PROCEDURE ==="
echo ""

# 1. Vérifier que Redis est arrêté
if systemctl is-active --quiet redis; then
    echo "❌ Redis is still running!"
    echo "Stop Redis first: systemctl stop redis"
    exit 1
fi

# 2. Créer un backup manuel supplémentaire
echo "Creating backup..."
mkdir -p "$BACKUP_DIR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
cp "$AOF_FILE" "$BACKUP_DIR/appendonly.aof.$TIMESTAMP"
echo "✅ Backup created: $BACKUP_DIR/appendonly.aof.$TIMESTAMP"
echo ""

# 3. Vérifier l'AOF
echo "Checking AOF file..."
redis-check-aof "$AOF_FILE"

if [ $? -eq 0 ]; then
    echo "✅ AOF is valid, no repair needed"
    exit 0
fi

echo ""
echo "⚠️  AOF file is corrupted"
echo ""

# 4. Proposer la réparation
read -p "Repair AOF? This will TRUNCATE corrupted data. [y/N]: " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "Repair cancelled"
    exit 0
fi

# 5. Réparer
echo ""
echo "Repairing AOF..."
echo "y" | redis-check-aof --fix "$AOF_FILE"

if [ $? -eq 0 ]; then
    echo ""
    echo "✅ AOF repaired successfully"

    # 6. Vérifier à nouveau
    echo ""
    echo "Verifying repaired AOF..."
    redis-check-aof "$AOF_FILE"

    if [ $? -eq 0 ]; then
        echo ""
        echo "✅ Repaired AOF is valid"
        echo ""
        echo "You can now restart Redis:"
        echo "  systemctl start redis"
    else
        echo ""
        echo "❌ Repaired AOF is still invalid"
        echo "Consider restoring from backup or RDB"
    fi
else
    echo ""
    echo "❌ AOF repair failed"
    echo "Restore from backup: cp $BACKUP_DIR/appendonly.aof.$TIMESTAMP $AOF_FILE"
fi
```

---

## 🚨 Procédures de recovery

### Scénario 1 : Corruption RDB détectée au démarrage

#### Symptôme

```bash
# Redis ne démarre pas
$ systemctl start redis
Job for redis.service failed

$ journalctl -u redis | tail
# Short read or OOM loading DB. Unrecoverable error, aborting now.
```

#### Procédure de recovery

```bash
#!/bin/bash
# recover-corrupted-rdb.sh

RDB_FILE="/var/lib/redis/dump.rdb"
RDB_BACKUP="/var/backups/redis/dump.rdb.backup"

echo "=== RDB CORRUPTION RECOVERY ==="
echo ""

# 1. Vérifier la corruption
echo "Step 1: Verifying RDB corruption..."
redis-check-rdb "$RDB_FILE"

if [ $? -eq 0 ]; then
    echo "✅ RDB is valid, no recovery needed"
    exit 0
fi

echo ""
echo "❌ RDB is corrupted"
echo ""

# 2. Options de recovery
echo "Recovery options:"
echo "  1) Restore from latest backup"
echo "  2) Start Redis without data (EMPTY)"
echo "  3) Use AOF if available"
echo "  4) Cancel"
echo ""
read -p "Choose option [1-4]: " option

case $option in
    1)
        # Restaurer depuis backup
        if [ -f "$RDB_BACKUP" ]; then
            echo ""
            echo "Restoring from backup: $RDB_BACKUP"
            cp "$RDB_FILE" "${RDB_FILE}.corrupted"
            cp "$RDB_BACKUP" "$RDB_FILE"

            echo "✅ Backup restored"
            echo ""
            echo "Starting Redis..."
            systemctl start redis

            if systemctl is-active --quiet redis; then
                echo "✅ Redis started successfully"
                redis-cli DBSIZE
            else
                echo "❌ Redis failed to start"
            fi
        else
            echo "❌ No backup found at: $RDB_BACKUP"
        fi
        ;;

    2)
        # Démarrer vide
        echo ""
        echo "⚠️  WARNING: Starting Redis EMPTY will LOSE ALL DATA"
        read -p "Are you sure? [y/N]: " -n 1 -r
        echo

        if [[ $REPLY =~ ^[Yy]$ ]]; then
            mv "$RDB_FILE" "${RDB_FILE}.corrupted"

            echo "Starting Redis with empty database..."
            systemctl start redis

            if systemctl is-active --quiet redis; then
                echo "✅ Redis started successfully (EMPTY)"
            else
                echo "❌ Redis failed to start"
            fi
        fi
        ;;

    3)
        # Utiliser AOF
        AOF_FILE="/var/lib/redis/appendonly.aof"

        if [ -f "$AOF_FILE" ]; then
            echo ""
            echo "AOF file found, checking..."
            redis-check-aof "$AOF_FILE"

            if [ $? -eq 0 ]; then
                # AOF valide, désactiver RDB
                mv "$RDB_FILE" "${RDB_FILE}.corrupted"

                # Modifier la config pour charger AOF
                sed -i 's/^appendonly no/appendonly yes/' /etc/redis/redis.conf

                echo "Starting Redis from AOF..."
                systemctl start redis

                if systemctl is-active --quiet redis; then
                    echo "✅ Redis started from AOF"
                    redis-cli DBSIZE
                else
                    echo "❌ Redis failed to start"
                fi
            else
                echo "❌ AOF is also corrupted"
                echo "Try repairing AOF with: redis-check-aof --fix"
            fi
        else
            echo "❌ No AOF file found"
        fi
        ;;

    4)
        echo "Recovery cancelled"
        exit 0
        ;;

    *)
        echo "Invalid option"
        exit 1
        ;;
esac
```

### Scénario 2 : Corruption AOF en production

#### Symptôme

```bash
# Redis tourne mais ne peut plus persister
redis-cli BGSAVE
# (error) MISCONF Redis is configured to save RDB snapshots...

# Logs montrent des erreurs AOF
$ tail -f /var/log/redis/redis-server.log
# Error writing to the AOF file: No space left on device
# Background AOF rewrite terminated with error
```

#### Recovery à chaud (sans arrêt)

```bash
#!/bin/bash
# hot-fix-aof.sh

echo "=== HOT FIX AOF (NO DOWNTIME) ==="
echo ""

# 1. Identifier le problème
echo "Step 1: Identifying issue..."

# Vérifier l'espace disque
DISK_USAGE=$(df /var/lib/redis | awk 'NR==2 {print $5}' | sed 's/%//')
echo "Disk usage: ${DISK_USAGE}%"

if [ "$DISK_USAGE" -gt 95 ]; then
    echo "❌ Disk is full!"
    echo ""
    echo "Actions to free space:"
    echo "  1) Clean old logs"
    echo "  2) Remove old backups"
    echo "  3) Move AOF to larger disk"

    # Nettoyage automatique
    read -p "Clean old Redis logs? [y/N]: " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        find /var/log/redis -name "*.log.*" -mtime +7 -delete
        echo "✅ Old logs deleted"
    fi
fi

echo ""
echo "Step 2: Checking AOF status..."

# État AOF actuel
redis-cli INFO persistence | grep -E "aof_enabled|aof_rewrite_in_progress|aof_last_bgrewrite_status"

# Si le rewrite AOF est bloqué
AOF_STATUS=$(redis-cli INFO persistence | grep aof_last_bgrewrite_status | cut -d: -f2 | tr -d '\r')

if [ "$AOF_STATUS" != "ok" ]; then
    echo "❌ AOF rewrite is failing"
    echo ""
    echo "Step 3: Attempting recovery..."

    # Option 1 : Désactiver temporairement AOF
    read -p "Temporarily disable AOF? Data since last save may be lost. [y/N]: " -n 1 -r
    echo

    if [[ $REPLY =~ ^[Yy]$ ]]; then
        echo "Disabling AOF..."
        redis-cli CONFIG SET appendonly no

        echo "Forcing RDB save..."
        redis-cli BGSAVE

        echo ""
        echo "✅ AOF disabled, RDB save initiated"
        echo ""
        echo "Monitor BGSAVE:"
        echo "  redis-cli INFO persistence | grep rdb_bgsave_in_progress"
        echo ""
        echo "Once BGSAVE complete, you can:"
        echo "  1) Fix AOF file offline"
        echo "  2) Re-enable AOF: redis-cli CONFIG SET appendonly yes"
    fi
fi
```

### Scénario 3 : Corruption mémoire en runtime

#### Symptôme

```bash
# Commandes crashent Redis
redis-cli GET key
# Connection closed by foreign host

# Redis redémarre constamment
$ systemctl status redis
# Active: active (running) ... Redis Server
# [puis crash et restart]
```

#### Diagnostic et recovery

```bash
#!/bin/bash
# diagnose-memory-corruption.sh

echo "=== MEMORY CORRUPTION DIAGNOSTIC ==="
echo ""

# 1. Vérifier les coredumps
echo "Step 1: Checking for coredumps..."

COREDUMP=$(find /var/lib/redis -name "core*" -o -name "dump.core*" | head -1)

if [ -n "$COREDUMP" ]; then
    echo "⚠️  Coredump found: $COREDUMP"

    # Analyser avec gdb si disponible
    if command -v gdb &> /dev/null; then
        echo ""
        echo "Analyzing coredump..."
        gdb -batch -ex "bt" /usr/bin/redis-server "$COREDUMP" 2>/dev/null | head -50
    fi
else
    echo "No coredump found"
fi

echo ""
echo "Step 2: Checking Redis logs..."

# Rechercher des patterns de crash
tail -100 /var/log/redis/redis-server.log | grep -i "assertion\|segfault\|panic\|bug\|crashed"

echo ""
echo "Step 3: Checking system logs..."

# OOM killer?
dmesg | grep -i redis | grep -i "killed\|oom" | tail -10

echo ""
echo "=== RECOVERY ACTIONS ==="
echo ""

# Actions recommandées
echo "If memory corruption detected:"
echo ""
echo "1. UPDATE REDIS (might be a known bug)"
echo "   Current version:"
redis-cli INFO server | grep redis_version
echo ""

echo "2. DISABLE PROBLEMATIC FEATURES"
echo "   - Disable AOF temporarily"
echo "   - Disable active defrag"
echo "   redis-cli CONFIG SET appendonly no"
echo "   redis-cli CONFIG SET activedefrag no"
echo ""

echo "3. RESTART WITH CLEAN STATE"
echo "   systemctl stop redis"
echo "   mv /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.backup"
echo "   systemctl start redis"
echo ""

echo "4. RESTORE FROM BACKUP"
echo "   (Use verified backup from before corruption)"
```

---

## 💾 Stratégies de backup robustes

### Backup automatisé multi-niveaux

```bash
#!/bin/bash
# redis-backup.sh - Stratégie de backup complète

REDIS_DIR="/var/lib/redis"
BACKUP_ROOT="/var/backups/redis"
RETENTION_DAYS=30

# Créer la structure de backup
mkdir -p "$BACKUP_ROOT"/{hourly,daily,weekly}

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
HOUR=$(date +%H)
DAY=$(date +%u)  # 1=Monday, 7=Sunday

echo "=== REDIS BACKUP - $TIMESTAMP ==="
echo ""

# 1. Déclencher un BGSAVE
echo "Triggering BGSAVE..."
redis-cli BGSAVE

# Attendre la fin du BGSAVE
while [ $(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2 | tr -d '\r') -eq 1 ]; do
    echo -n "."
    sleep 1
done
echo ""
echo "✅ BGSAVE completed"

# 2. Copier RDB
if [ -f "$REDIS_DIR/dump.rdb" ]; then
    # Backup horaire
    cp "$REDIS_DIR/dump.rdb" "$BACKUP_ROOT/hourly/dump_${TIMESTAMP}.rdb"
    echo "✅ Hourly backup created"

    # Backup quotidien (à minuit)
    if [ "$HOUR" == "00" ]; then
        cp "$REDIS_DIR/dump.rdb" "$BACKUP_ROOT/daily/dump_${TIMESTAMP}.rdb"
        echo "✅ Daily backup created"
    fi

    # Backup hebdomadaire (dimanche à minuit)
    if [ "$DAY" == "7" ] && [ "$HOUR" == "00" ]; then
        cp "$REDIS_DIR/dump.rdb" "$BACKUP_ROOT/weekly/dump_${TIMESTAMP}.rdb"
        echo "✅ Weekly backup created"
    fi
fi

# 3. Copier AOF si activé
if [ -f "$REDIS_DIR/appendonly.aof" ]; then
    cp "$REDIS_DIR/appendonly.aof" "$BACKUP_ROOT/hourly/appendonly_${TIMESTAMP}.aof"
    echo "✅ AOF backup created"
fi

# 4. Vérifier l'intégrité des backups
echo ""
echo "Verifying backup integrity..."

LATEST_RDB="$BACKUP_ROOT/hourly/dump_${TIMESTAMP}.rdb"
redis-check-rdb "$LATEST_RDB" > /dev/null 2>&1

if [ $? -eq 0 ]; then
    echo "✅ RDB backup is valid"
else
    echo "❌ RDB backup is CORRUPTED!"
    exit 1
fi

# 5. Nettoyer les vieux backups
echo ""
echo "Cleaning old backups..."

# Hourly : garder 24h
find "$BACKUP_ROOT/hourly" -name "dump_*.rdb" -mtime +1 -delete
find "$BACKUP_ROOT/hourly" -name "appendonly_*.aof" -mtime +1 -delete

# Daily : garder 30 jours
find "$BACKUP_ROOT/daily" -name "dump_*.rdb" -mtime +$RETENTION_DAYS -delete

# Weekly : garder 1 an
find "$BACKUP_ROOT/weekly" -name "dump_*.rdb" -mtime +365 -delete

echo "✅ Old backups cleaned"

# 6. Statistiques
echo ""
echo "Backup statistics:"
echo "  Hourly backups: $(ls -1 $BACKUP_ROOT/hourly/*.rdb 2>/dev/null | wc -l)"
echo "  Daily backups:  $(ls -1 $BACKUP_ROOT/daily/*.rdb 2>/dev/null | wc -l)"
echo "  Weekly backups: $(ls -1 $BACKUP_ROOT/weekly/*.rdb 2>/dev/null | wc -l)"

# 7. Taille totale
TOTAL_SIZE=$(du -sh "$BACKUP_ROOT" | cut -f1)
echo "  Total size: $TOTAL_SIZE"

echo ""
echo "=== BACKUP COMPLETE ==="
```

### Backup off-site automatisé

```bash
#!/bin/bash
# redis-backup-offsite.sh

BACKUP_SOURCE="/var/backups/redis"
S3_BUCKET="s3://my-redis-backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=== OFF-SITE BACKUP ==="
echo ""

# 1. Créer une archive des backups récents
ARCHIVE="/tmp/redis-backup-${TIMESTAMP}.tar.gz"

echo "Creating archive..."
tar -czf "$ARCHIVE" \
    "$BACKUP_SOURCE/daily" \
    "$BACKUP_SOURCE/weekly"

if [ $? -eq 0 ]; then
    echo "✅ Archive created: $ARCHIVE"
else
    echo "❌ Archive creation failed"
    exit 1
fi

# 2. Uploader vers S3 (ou autre)
echo ""
echo "Uploading to S3..."

if command -v aws &> /dev/null; then
    aws s3 cp "$ARCHIVE" "$S3_BUCKET/redis-backup-${TIMESTAMP}.tar.gz"

    if [ $? -eq 0 ]; then
        echo "✅ Uploaded to S3"
        rm "$ARCHIVE"
    else
        echo "❌ S3 upload failed"
        exit 1
    fi
else
    echo "❌ AWS CLI not installed"
    echo "Archive saved locally: $ARCHIVE"
fi

# 3. Nettoyer les vieux backups S3 (garder 90 jours)
echo ""
echo "Cleaning old S3 backups..."

if command -v aws &> /dev/null; then
    CUTOFF_DATE=$(date -d '90 days ago' +%Y%m%d)

    aws s3 ls "$S3_BUCKET/" | while read -r line; do
        FILE=$(echo $line | awk '{print $4}')
        FILE_DATE=$(echo $FILE | grep -oP '\d{8}' | head -1)

        if [ -n "$FILE_DATE" ] && [ "$FILE_DATE" -lt "$CUTOFF_DATE" ]; then
            echo "Deleting old backup: $FILE"
            aws s3 rm "$S3_BUCKET/$FILE"
        fi
    done
fi

echo ""
echo "=== OFF-SITE BACKUP COMPLETE ==="
```

---

## 🛡️ Prévention de la corruption

### Configuration robuste

```conf
# redis.conf - Configuration anti-corruption

# 1. Checksum activé (détecte la corruption)
rdbchecksum yes

# 2. Stop sur erreur d'écriture
stop-writes-on-bgsave-error yes

# 3. Compression RDB (réduit corruption réseau)
rdbcompression yes

# 4. AOF avec fsync régulier (mais pas always)
appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite yes

# 5. Réparation automatique AOF si possible
aof-load-truncated yes

# 6. Utiliser RDB+AOF hybride
aof-use-rdb-preamble yes

# 7. Éviter les big keys (fragmentent moins)
# (à gérer au niveau applicatif)
```

### Monitoring de la santé des fichiers

```python
#!/usr/bin/env python3
"""
Monitoring continu de la santé des fichiers Redis
"""
import subprocess
import time
import hashlib
from datetime import datetime

class RedisFileHealthMonitor:
    def __init__(self, rdb_file, aof_file, check_interval=300):
        self.rdb_file = rdb_file
        self.aof_file = aof_file
        self.check_interval = check_interval
        self.rdb_hash = None
        self.aof_hash = None

    def calculate_hash(self, filepath):
        """Calcule le hash d'un fichier"""
        try:
            with open(filepath, 'rb') as f:
                return hashlib.sha256(f.read()).hexdigest()
        except:
            return None

    def check_rdb(self):
        """Vérifie le RDB"""
        print(f"[{datetime.now()}] Checking RDB...")

        try:
            result = subprocess.run(
                ['redis-check-rdb', self.rdb_file],
                capture_output=True,
                text=True,
                timeout=60
            )

            if result.returncode == 0:
                # Calculer le hash
                current_hash = self.calculate_hash(self.rdb_file)

                if self.rdb_hash and current_hash != self.rdb_hash:
                    print(f"  ℹ️  RDB file changed")

                self.rdb_hash = current_hash
                print(f"  ✅ RDB is valid")
                return True
            else:
                print(f"  ❌ RDB is CORRUPTED!")
                print(f"  {result.stderr}")
                self.send_alert("RDB corruption detected!")
                return False

        except subprocess.TimeoutExpired:
            print(f"  ⚠️  RDB check timeout")
            return None
        except Exception as e:
            print(f"  ❌ Error checking RDB: {e}")
            return None

    def check_aof(self):
        """Vérifie l'AOF"""
        print(f"[{datetime.now()}] Checking AOF...")

        try:
            result = subprocess.run(
                ['redis-check-aof', self.aof_file],
                capture_output=True,
                text=True,
                timeout=60
            )

            if result.returncode == 0:
                current_hash = self.calculate_hash(self.aof_file)

                if self.aof_hash and current_hash != self.aof_hash:
                    print(f"  ℹ️  AOF file changed")

                self.aof_hash = current_hash
                print(f"  ✅ AOF is valid")
                return True
            else:
                print(f"  ❌ AOF is CORRUPTED!")
                print(f"  {result.stderr}")
                self.send_alert("AOF corruption detected!")
                return False

        except subprocess.TimeoutExpired:
            print(f"  ⚠️  AOF check timeout")
            return None
        except Exception as e:
            print(f"  ❌ Error checking AOF: {e}")
            return None

    def send_alert(self, message):
        """Envoie une alerte"""
        # TODO: Implémenter l'envoi (email, Slack, etc.)
        print(f"\n🚨 ALERT: {message}\n")

    def run(self):
        """Boucle de monitoring"""
        print("Starting Redis File Health Monitor")
        print(f"Check interval: {self.check_interval}s")
        print("-" * 60)

        while True:
            try:
                # Check RDB
                self.check_rdb()

                # Check AOF
                self.check_aof()

                print(f"Next check in {self.check_interval}s")
                print("-" * 60)

                time.sleep(self.check_interval)

            except KeyboardInterrupt:
                print("\nMonitoring stopped")
                break
            except Exception as e:
                print(f"Error in monitoring loop: {e}")
                time.sleep(self.check_interval)

if __name__ == "__main__":
    monitor = RedisFileHealthMonitor(
        rdb_file='/var/lib/redis/dump.rdb',
        aof_file='/var/lib/redis/appendonly.aof',
        check_interval=300  # 5 minutes
    )
    monitor.run()
```

### Tests de recovery réguliers

```bash
#!/bin/bash
# test-recovery.sh - Tester la recovery régulièrement

TEST_DIR="/tmp/redis-recovery-test"
BACKUP_DIR="/var/backups/redis"

echo "=== REDIS RECOVERY TEST ==="
echo ""

# 1. Préparer l'environnement de test
echo "Step 1: Preparing test environment..."
rm -rf "$TEST_DIR"
mkdir -p "$TEST_DIR"

# 2. Copier le dernier backup
echo "Step 2: Copying latest backup..."
LATEST_RDB=$(ls -t "$BACKUP_DIR"/*/dump_*.rdb 2>/dev/null | head -1)

if [ -z "$LATEST_RDB" ]; then
    echo "❌ No backup found"
    exit 1
fi

cp "$LATEST_RDB" "$TEST_DIR/dump.rdb"
echo "Using backup: $LATEST_RDB"

# 3. Vérifier le RDB
echo ""
echo "Step 3: Verifying RDB..."
redis-check-rdb "$TEST_DIR/dump.rdb"

if [ $? -ne 0 ]; then
    echo "❌ Backup RDB is corrupted!"
    exit 1
fi

echo "✅ RDB is valid"

# 4. Démarrer Redis de test
echo ""
echo "Step 4: Starting test Redis instance..."

# Config temporaire
cat > "$TEST_DIR/redis-test.conf" << EOF
port 6380
dir $TEST_DIR
dbfilename dump.rdb
daemonize yes
pidfile $TEST_DIR/redis-test.pid
logfile $TEST_DIR/redis-test.log
save ""
appendonly no
EOF

redis-server "$TEST_DIR/redis-test.conf"
sleep 2

# 5. Tester les commandes
echo "Step 5: Testing commands..."

DBSIZE=$(redis-cli -p 6380 DBSIZE)
echo "Database size: $DBSIZE keys"

if redis-cli -p 6380 PING > /dev/null 2>&1; then
    echo "✅ PING successful"
else
    echo "❌ PING failed"
    exit 1
fi

# Tester quelques GET
echo "Testing random GET commands..."
redis-cli -p 6380 --scan | head -10 | while read key; do
    redis-cli -p 6380 GET "$key" > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo "  ✅ GET $key"
    else
        echo "  ❌ GET $key FAILED"
    fi
done

# 6. Nettoyer
echo ""
echo "Step 6: Cleaning up..."
redis-cli -p 6380 SHUTDOWN NOSAVE
rm -rf "$TEST_DIR"

echo ""
echo "=== RECOVERY TEST COMPLETE ✅ ==="
```

---

## 📊 Checklist de prévention

### Configuration

- [ ] `rdbchecksum yes` activé
- [ ] `stop-writes-on-bgsave-error yes`
- [ ] `aof-load-truncated yes`
- [ ] `aof-use-rdb-preamble yes`
- [ ] Persistence configurée (RDB + AOF)

### Monitoring

- [ ] Vérification intégrité fichiers (quotidien)
- [ ] Alertes sur erreurs BGSAVE/AOF
- [ ] Monitoring espace disque
- [ ] Logs centralisés et analysés

### Backups

- [ ] Backups automatiques horaires/quotidiens/hebdomadaires
- [ ] Backups off-site (S3, autre DC)
- [ ] Vérification intégrité backups
- [ ] Tests de recovery mensuels
- [ ] Documentation procédures de recovery

### Infrastructure

- [ ] Disques en RAID (redondance)
- [ ] Monitoring matériel (SMART)
- [ ] UPS pour coupures électriques
- [ ] Filesystem moderne (ext4, xfs)

---

## 🎯 Points clés à retenir

1. **La corruption est rare mais possible** → Toujours avoir des backups
2. **redis-check-rdb et redis-check-aof** → Outils essentiels de diagnostic
3. **AOF réparable, RDB non** → AOF --fix existe, RDB nécessite backup
4. **Backups multi-niveaux** → Hourly/Daily/Weekly + off-site
5. **Tester les recovery** → Un backup non testé est inutile
6. **rdbchecksum yes** → Détection automatique de corruption
7. **Monitoring continu** → Détecter avant la catastrophe
8. **Documentation** → Procédures claires et accessibles

---

**🚀 Section suivante** : [14.7 - Fragmentation mémoire : Detection et défragmentation](./07-fragmentation-memoire-defragmentation.md)

⏭️ [Fragmentation mémoire : Detection et défragmentation](/14-performance-troubleshooting/07-fragmentation-memoire-defragmentation.md)
