🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.8 - Mises à jour sans downtime (Rolling upgrades)

## Introduction

Les mises à jour Redis en production doivent être effectuées **sans interruption de service**. Une mise à jour mal planifiée peut causer :

- 🔴 **Downtime complet** de l'application
- 🔴 **Perte de données** si mauvaise procédure
- 🔴 **Split-brain** en cas d'erreur de réplication
- 🔴 **Incompatibilités** entre versions

> **⚠️ Principe fondamental :** Toujours mettre à jour les replicas AVANT le master. Toujours avoir un plan de rollback testé.

---

## Stratégies de mise à jour

### 1. Vue d'ensemble des stratégies

```
┌─────────────────────────────────────────────────────────────────┐
│              STRATÉGIES DE MISE À JOUR REDIS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. BLUE-GREEN DEPLOYMENT                                       │
│     ├── Déployer nouveau cluster en parallèle                   │
│     ├── Migrer trafic progressivement                           │
│     ├── Rollback: Simple bascule                                │
│     └── Coût: 2x ressources temporairement                      │
│                                                                 │
│  2. ROLLING UPDATE (Master-Replica)                             │
│     ├── Update replicas un par un                               │
│     ├── Failover vers replica updated                           │
│     ├── Update ancien master                                    │
│     └── Downtime: ~5-30 secondes (failover)                     │
│                                                                 │
│  3. ROLLING UPDATE (Redis Cluster)                              │
│     ├── Update replicas de chaque shard                         │
│     ├── Failover shard par shard                                │
│     ├── Update anciens masters                                  │
│     └── Downtime: 0 (si cluster bien configuré)                 │
│                                                                 │
│  4. MAINTENANCE WINDOW                                          │
│     ├── Planifier downtime                                      │
│     ├── Mise à jour complète                                    │
│     ├── Tests exhaustifs                                        │
│     └── Acceptable pour dev/staging uniquement                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Matrice de compatibilité

```
┌─────────────────────────────────────────────────────────────────┐
│         COMPATIBILITÉ VERSIONS REDIS (REPLICATION)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Règle générale:                                                │
│  ├── Replica peut être N+1 version vs Master                    │
│  ├── Master NE PEUT PAS être N+1 version vs Replica             │
│  └── Différence max: 1 version majeure                          │
│                                                                 │
│  Exemples compatibles:                                          │
│  ✅ Master 6.2 → Replica 7.0 (OK temporairement)                │
│  ✅ Master 7.0 → Replica 7.2 (OK)                               │
│  ✅ Master 6.2 → Replica 6.2 (OK)                               │
│                                                                 │
│  Exemples incompatibles:                                        │
│  ❌ Master 7.0 → Replica 6.2 (NON!)                             │
│  ❌ Master 7.2 → Replica 6.0 (NON!)                             │
│                                                                 │
│  Procédure sûre:                                                │
│  1. Update TOUTES les replicas                                  │
│  2. Failover (replica devient master)                           │
│  3. Update ancien master                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Rolling Update : Master-Replica simple

### 1. Architecture et procédure

```
┌─────────────────────────────────────────────────────────────────┐
│         ROLLING UPDATE : MASTER-REPLICA ARCHITECTURE            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  État initial:                                                  │
│  ┌─────────────┐                                                │
│  │   Master    │  Version 6.2                                   │
│  │  (active)   │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ├──────────────┬──────────────┐                         │
│         │              │              │                         │
│  ┌──────▼──────┐┌──────▼──────┐┌──────▼──────┐                  │
│  │  Replica 1  ││  Replica 2  ││  Replica 3  │  Version 6.2     │
│  │  (standby)  ││  (standby)  ││  (standby)  │                  │
│  └─────────────┘└─────────────┘└─────────────┘                  │
│                                                                 │
│  Étape 1: Update Replicas                                       │
│  ┌─────────────┐                                                │
│  │   Master    │  Version 6.2 (inchangé)                        │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ├──────────────┬──────────────┐                         │
│  ┌──────▼──────┐┌──────▼──────┐┌──────▼──────┐                  │
│  │  Replica 1  ││  Replica 2  ││  Replica 3  │  Version 7.0     │
│  │   (7.0)     ││   (7.0)     ││   (7.0)     │  (updated!)      │
│  └─────────────┘└─────────────┘└─────────────┘                  │
│                                                                 │
│  Étape 2: Failover (Replica 1 devient Master)                   │
│  ┌─────────────┐                                                │
│  │  Master     │  Version 7.0 (nouveau!)                        │
│  │ (ex-Rep1)   │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ├──────────────┬──────────────┐                         │
│  ┌──────▼──────┐┌──────▼──────┐┌──────▼──────┐                  │
│  │ Ancien      ││  Replica 2  ││  Replica 3  │                  │
│  │ Master      ││   (7.0)     ││   (7.0)     │                  │
│  │ (6.2)       ││             ││             │                  │
│  └─────────────┘└─────────────┘└─────────────┘                  │
│                                                                 │
│  Étape 3: Update ancien Master                                  │
│  ┌─────────────┐                                                │
│  │   Master    │  Version 7.0                                   │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ├──────────────┬──────────────┐                         │
│  ┌──────▼──────┐┌──────▼──────┐┌──────▼──────┐                  │
│  │  Replica 1  ││  Replica 2  ││  Replica 3  │  Version 7.0     │
│  │ (ex-master) ││             ││             │  (tous à jour!)  │
│  └─────────────┘└─────────────┘└─────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Script de rolling update

```bash
#!/bin/bash
# rolling-update-master-replica.sh

set -e

# ============================================================================
# ROLLING UPDATE SCRIPT - MASTER-REPLICA ARCHITECTURE
# ============================================================================

# Configuration
MASTER_HOST="redis-master.example.com"
MASTER_PORT=6379
REPLICA_HOSTS=("redis-replica1.example.com" "redis-replica2.example.com" "redis-replica3.example.com")
REPLICA_PORT=6379
NEW_VERSION="7.2.3"
REDIS_PASSWORD="YourPassword123"

# Couleurs pour output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Logging
LOG_FILE="/var/log/redis-rolling-update-$(date +%Y%m%d-%H%M%S).log"
exec > >(tee -a "$LOG_FILE")
exec 2>&1

echo "=============================================="
echo "REDIS ROLLING UPDATE"
echo "Target version: $NEW_VERSION"
echo "Start time: $(date)"
echo "=============================================="

# ============================================================================
# FONCTIONS UTILITAIRES
# ============================================================================

redis_cli_cmd() {
    local host=$1
    local port=$2
    local cmd=$3
    redis-cli -h "$host" -p "$port" -a "$REDIS_PASSWORD" --no-auth-warning $cmd
}

get_redis_version() {
    local host=$1
    local port=$2
    redis_cli_cmd "$host" "$port" "INFO server" | grep "redis_version:" | cut -d: -f2 | tr -d '\r'
}

get_role() {
    local host=$1
    local port=$2
    redis_cli_cmd "$host" "$port" "INFO replication" | grep "role:" | cut -d: -f2 | tr -d '\r'
}

wait_for_sync() {
    local host=$1
    local port=$2
    local max_wait=300  # 5 minutes
    local elapsed=0

    echo "Waiting for $host to sync..."

    while [ $elapsed -lt $max_wait ]; do
        local state=$(redis_cli_cmd "$host" "$port" "INFO replication" | grep "master_link_status:" | cut -d: -f2 | tr -d '\r')

        if [ "$state" = "up" ]; then
            echo -e "${GREEN}✓${NC} $host is in sync"
            return 0
        fi

        echo "  Waiting for sync... ($elapsed/$max_wait seconds)"
        sleep 5
        elapsed=$((elapsed + 5))
    done

    echo -e "${RED}✗${NC} Timeout waiting for $host to sync"
    return 1
}

# ============================================================================
# PHASE 0: PRÉ-VÉRIFICATIONS
# ============================================================================

echo ""
echo "=============================================="
echo "PHASE 0: PRE-FLIGHT CHECKS"
echo "=============================================="

# Vérifier version actuelle master
MASTER_CURRENT_VERSION=$(get_redis_version "$MASTER_HOST" "$MASTER_PORT")
echo "Master current version: $MASTER_CURRENT_VERSION"

if [ -z "$MASTER_CURRENT_VERSION" ]; then
    echo -e "${RED}✗${NC} Cannot connect to master"
    exit 1
fi

# Vérifier rôle master
MASTER_ROLE=$(get_role "$MASTER_HOST" "$MASTER_PORT")
if [ "$MASTER_ROLE" != "master" ]; then
    echo -e "${RED}✗${NC} $MASTER_HOST is not master (role: $MASTER_ROLE)"
    exit 1
fi
echo -e "${GREEN}✓${NC} Master role confirmed"

# Vérifier toutes les replicas
echo ""
echo "Checking replicas..."
for replica in "${REPLICA_HOSTS[@]}"; do
    version=$(get_redis_version "$replica" "$REPLICA_PORT")
    role=$(get_role "$replica" "$REPLICA_PORT")

    if [ -z "$version" ]; then
        echo -e "${RED}✗${NC} Cannot connect to $replica"
        exit 1
    fi

    if [ "$role" != "slave" ]; then
        echo -e "${RED}✗${NC} $replica is not a replica (role: $role)"
        exit 1
    fi

    echo -e "${GREEN}✓${NC} $replica - version $version, role $role"
done

# Backup configuration
echo ""
echo "Backing up current configurations..."
for host in "$MASTER_HOST" "${REPLICA_HOSTS[@]}"; do
    redis_cli_cmd "$host" "$REPLICA_PORT" "CONFIG REWRITE"
    echo -e "${GREEN}✓${NC} Configuration backed up on $host"
done

# Confirmation
echo ""
echo -e "${YELLOW}WARNING:${NC} About to update Redis from $MASTER_CURRENT_VERSION to $NEW_VERSION"
echo "Master: $MASTER_HOST"
echo "Replicas: ${REPLICA_HOSTS[*]}"
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Aborted by user"
    exit 0
fi

# ============================================================================
# PHASE 1: UPDATE REPLICAS
# ============================================================================

echo ""
echo "=============================================="
echo "PHASE 1: UPDATING REPLICAS"
echo "=============================================="

for replica in "${REPLICA_HOSTS[@]}"; do
    echo ""
    echo "Updating $replica..."

    # Sur le serveur replica, exécuter:
    # 1. Stop Redis
    # 2. Upgrade binaries
    # 3. Start Redis
    # 4. Wait for sync

    ssh "$replica" << 'ENDSSH'
        # Stop Redis
        sudo systemctl stop redis-server

        # Backup old binary
        sudo cp /usr/bin/redis-server /usr/bin/redis-server.backup

        # Install new version (exemple avec apt)
        sudo apt-get update
        sudo apt-get install -y redis-server=$NEW_VERSION

        # Start Redis
        sudo systemctl start redis-server
ENDSSH

    # Attendre que la replica se synchronise
    sleep 5
    wait_for_sync "$replica" "$REPLICA_PORT"

    # Vérifier nouvelle version
    NEW_VER=$(get_redis_version "$replica" "$REPLICA_PORT")
    echo -e "${GREEN}✓${NC} $replica updated to version $NEW_VER"
done

echo ""
echo -e "${GREEN}✓${NC} All replicas updated successfully"

# ============================================================================
# PHASE 2: FAILOVER TO UPDATED REPLICA
# ============================================================================

echo ""
echo "=============================================="
echo "PHASE 2: FAILOVER TO UPDATED REPLICA"
echo "=============================================="

# Choisir première replica comme nouveau master
NEW_MASTER="${REPLICA_HOSTS[0]}"

echo "Promoting $NEW_MASTER to master..."

# Exécuter REPLICAOF NO ONE sur la replica choisie
redis_cli_cmd "$NEW_MASTER" "$REPLICA_PORT" "REPLICAOF NO ONE"

# Attendre quelques secondes
sleep 3

# Vérifier que c'est bien devenu master
NEW_ROLE=$(get_role "$NEW_MASTER" "$REPLICA_PORT")
if [ "$NEW_ROLE" != "master" ]; then
    echo -e "${RED}✗${NC} Failover failed! $NEW_MASTER is still $NEW_ROLE"
    exit 1
fi

echo -e "${GREEN}✓${NC} $NEW_MASTER is now master"

# Reconfigurer ancien master en replica
echo ""
echo "Reconfiguring old master as replica..."
redis_cli_cmd "$MASTER_HOST" "$MASTER_PORT" "REPLICAOF $NEW_MASTER $REPLICA_PORT"

sleep 5
wait_for_sync "$MASTER_HOST" "$MASTER_PORT"

echo -e "${GREEN}✓${NC} Old master is now replica of $NEW_MASTER"

# Reconfigurer autres replicas
echo ""
echo "Reconfiguring other replicas..."
for replica in "${REPLICA_HOSTS[@]}"; do
    if [ "$replica" != "$NEW_MASTER" ]; then
        echo "Reconfiguring $replica..."
        redis_cli_cmd "$replica" "$REPLICA_PORT" "REPLICAOF $NEW_MASTER $REPLICA_PORT"
        wait_for_sync "$replica" "$REPLICA_PORT"
        echo -e "${GREEN}✓${NC} $replica reconfigured"
    fi
done

# ============================================================================
# PHASE 3: UPDATE OLD MASTER
# ============================================================================

echo ""
echo "=============================================="
echo "PHASE 3: UPDATING OLD MASTER"
echo "=============================================="

echo "Updating $MASTER_HOST (now a replica)..."

ssh "$MASTER_HOST" << 'ENDSSH'
    sudo systemctl stop redis-server
    sudo cp /usr/bin/redis-server /usr/bin/redis-server.backup
    sudo apt-get update
    sudo apt-get install -y redis-server=$NEW_VERSION
    sudo systemctl start redis-server
ENDSSH

sleep 5
wait_for_sync "$MASTER_HOST" "$MASTER_PORT"

NEW_VER=$(get_redis_version "$MASTER_HOST" "$MASTER_PORT")
echo -e "${GREEN}✓${NC} Old master updated to version $NEW_VER"

# ============================================================================
# PHASE 4: VALIDATION
# ============================================================================

echo ""
echo "=============================================="
echo "PHASE 4: VALIDATION"
echo "=============================================="

echo "Current topology:"
echo "  Master: $NEW_MASTER (version $(get_redis_version "$NEW_MASTER" "$REPLICA_PORT"))"

echo "  Replicas:"
for host in "$MASTER_HOST" "${REPLICA_HOSTS[@]}"; do
    if [ "$host" != "$NEW_MASTER" ]; then
        version=$(get_redis_version "$host" "$REPLICA_PORT")
        echo "    - $host (version $version)"
    fi
done

# Test write/read
echo ""
echo "Testing write/read operations..."
TEST_KEY="rolling-update-test-$(date +%s)"
TEST_VALUE="success"

redis_cli_cmd "$NEW_MASTER" "$REPLICA_PORT" "SET $TEST_KEY $TEST_VALUE EX 60" > /dev/null

for host in "${REPLICA_HOSTS[@]}" "$MASTER_HOST"; do
    if [ "$host" != "$NEW_MASTER" ]; then
        sleep 1
        RESULT=$(redis_cli_cmd "$host" "$REPLICA_PORT" "GET $TEST_KEY")
        if [ "$RESULT" = "$TEST_VALUE" ]; then
            echo -e "${GREEN}✓${NC} Read test passed on $host"
        else
            echo -e "${RED}✗${NC} Read test failed on $host"
        fi
    fi
done

redis_cli_cmd "$NEW_MASTER" "$REPLICA_PORT" "DEL $TEST_KEY" > /dev/null

# ============================================================================
# SUMMARY
# ============================================================================

echo ""
echo "=============================================="
echo "ROLLING UPDATE COMPLETED SUCCESSFULLY"
echo "=============================================="
echo "End time: $(date)"
echo "New master: $NEW_MASTER"
echo "All nodes version: $NEW_VERSION"
echo "Log file: $LOG_FILE"
echo ""
echo -e "${YELLOW}IMPORTANT:${NC} Update your application configuration to point to new master: $NEW_MASTER"
echo "=============================================="
```

---

## Rolling Update : Redis Sentinel

### 1. Procédure avec Sentinel

```bash
#!/bin/bash
# rolling-update-sentinel.sh

set -e

# ============================================================================
# ROLLING UPDATE WITH SENTINEL
# ============================================================================

SENTINEL_HOSTS=("sentinel1.example.com" "sentinel2.example.com" "sentinel3.example.com")
SENTINEL_PORT=26379
MASTER_NAME="mymaster"
NEW_VERSION="7.2.3"

echo "=============================================="
echo "REDIS ROLLING UPDATE WITH SENTINEL"
echo "=============================================="

# ============================================================================
# PHASE 1: UPDATE REDIS REPLICAS
# ============================================================================

echo ""
echo "Phase 1: Updating replicas..."

# Obtenir master et replicas depuis Sentinel
MASTER=$(redis-cli -h "${SENTINEL_HOSTS[0]}" -p "$SENTINEL_PORT" \
    SENTINEL get-master-addr-by-name "$MASTER_NAME" | head -1)
MASTER_PORT=$(redis-cli -h "${SENTINEL_HOSTS[0]}" -p "$SENTINEL_PORT" \
    SENTINEL get-master-addr-by-name "$MASTER_NAME" | tail -1)

echo "Current master: $MASTER:$MASTER_PORT"

# Obtenir liste des replicas
REPLICAS=$(redis-cli -h "${SENTINEL_HOSTS[0]}" -p "$SENTINEL_PORT" \
    SENTINEL replicas "$MASTER_NAME")

# Pour chaque replica, update
echo "$REPLICAS" | grep -oP 'ip=\K[^,]+' | while read replica; do
    echo "Updating replica: $replica"

    # SSH et update
    ssh "$replica" << 'ENDSSH'
        sudo systemctl stop redis-server
        sudo apt-get update && sudo apt-get install -y redis-server
        sudo systemctl start redis-server
ENDSSH

    echo "✓ Replica $replica updated"
done

# ============================================================================
# PHASE 2: SENTINEL FAILOVER
# ============================================================================

echo ""
echo "Phase 2: Triggering Sentinel failover..."

# Déclencher failover automatique
redis-cli -h "${SENTINEL_HOSTS[0]}" -p "$SENTINEL_PORT" \
    SENTINEL failover "$MASTER_NAME"

# Attendre fin du failover
echo "Waiting for failover to complete..."
sleep 10

# Vérifier nouveau master
NEW_MASTER=$(redis-cli -h "${SENTINEL_HOSTS[0]}" -p "$SENTINEL_PORT" \
    SENTINEL get-master-addr-by-name "$MASTER_NAME" | head -1)

echo "New master: $NEW_MASTER"

if [ "$NEW_MASTER" = "$MASTER" ]; then
    echo "✗ Failover did not change master!"
    exit 1
fi

echo "✓ Failover completed successfully"

# ============================================================================
# PHASE 3: UPDATE OLD MASTER
# ============================================================================

echo ""
echo "Phase 3: Updating old master (now replica)..."

ssh "$MASTER" << 'ENDSSH'
    sudo systemctl stop redis-server
    sudo apt-get update && sudo apt-get install -y redis-server
    sudo systemctl start redis-server
ENDSSH

echo "✓ Old master updated"

# ============================================================================
# PHASE 4: UPDATE SENTINEL NODES
# ============================================================================

echo ""
echo "Phase 4: Updating Sentinel nodes..."

for sentinel in "${SENTINEL_HOSTS[@]}"; do
    echo "Updating Sentinel: $sentinel"

    ssh "$sentinel" << 'ENDSSH'
        sudo systemctl stop redis-sentinel
        sudo apt-get update && sudo apt-get install -y redis-sentinel
        sudo systemctl start redis-sentinel
ENDSSH

    # Attendre que Sentinel se reconnecte
    sleep 5

    echo "✓ Sentinel $sentinel updated"
done

echo ""
echo "=============================================="
echo "SENTINEL ROLLING UPDATE COMPLETED"
echo "=============================================="
```

---

## Rolling Update : Redis Cluster

### 1. Procédure Cluster (sans downtime)

```bash
#!/bin/bash
# rolling-update-cluster.sh

set -e

# ============================================================================
# ROLLING UPDATE - REDIS CLUSTER
# ============================================================================

# Liste de tous les nodes du cluster
CLUSTER_NODES=(
    "node1.example.com:6379"
    "node2.example.com:6379"
    "node3.example.com:6379"
    "node4.example.com:6379"
    "node5.example.com:6379"
    "node6.example.com:6379"
)

NEW_VERSION="7.2.3"

echo "=============================================="
echo "REDIS CLUSTER ROLLING UPDATE"
echo "Nodes: ${CLUSTER_NODES[@]}"
echo "Target version: $NEW_VERSION"
echo "=============================================="

# ============================================================================
# FONCTIONS
# ============================================================================

get_node_role() {
    local node=$1
    local host=$(echo $node | cut -d: -f1)
    local port=$(echo $node | cut -d: -f2)

    redis-cli -h "$host" -p "$port" CLUSTER NODES | grep myself | \
        grep -oP '(master|slave)'
}

get_master_id() {
    local node=$1
    local host=$(echo $node | cut -d: -f1)
    local port=$(echo $node | cut -d: -f2)

    redis-cli -h "$host" -p "$port" CLUSTER NODES | grep master | \
        grep -v myself | head -1 | awk '{print $1}'
}

# ============================================================================
# PHASE 1: IDENTIFIER MASTERS ET REPLICAS
# ============================================================================

echo ""
echo "Phase 1: Identifying topology..."

declare -a MASTERS
declare -a REPLICAS

for node in "${CLUSTER_NODES[@]}"; do
    role=$(get_node_role "$node")

    if [ "$role" = "master" ]; then
        MASTERS+=("$node")
        echo "  Master: $node"
    else
        REPLICAS+=("$node")
        echo "  Replica: $node"
    fi
done

echo ""
echo "Found ${#MASTERS[@]} masters and ${#REPLICAS[@]} replicas"

# ============================================================================
# PHASE 2: UPDATE REPLICAS
# ============================================================================

echo ""
echo "=============================================="
echo "Phase 2: Updating replicas..."
echo "=============================================="

for replica in "${REPLICAS[@]}"; do
    host=$(echo $replica | cut -d: -f1)
    port=$(echo $replica | cut -d: -f2)

    echo ""
    echo "Updating replica: $host:$port"

    # SSH et update
    ssh "$host" << ENDSSH
        echo "Stopping Redis on $host..."
        sudo systemctl stop redis-server

        echo "Upgrading to version $NEW_VERSION..."
        sudo apt-get update
        sudo apt-get install -y redis-server=$NEW_VERSION

        echo "Starting Redis on $host..."
        sudo systemctl start redis-server
ENDSSH

    # Attendre que le node rejoigne le cluster
    echo "Waiting for $host to rejoin cluster..."
    sleep 10

    # Vérifier état cluster
    STATE=$(redis-cli -h "$host" -p "$port" CLUSTER INFO | grep cluster_state | cut -d: -f2 | tr -d '\r')

    if [ "$STATE" = "ok" ]; then
        echo "✓ Replica $host:$port updated and rejoined cluster"
    else
        echo "✗ Replica $host:$port cluster state: $STATE"
        exit 1
    fi
done

echo ""
echo "✓ All replicas updated successfully"

# ============================================================================
# PHASE 3: FAILOVER POUR CHAQUE MASTER
# ============================================================================

echo ""
echo "=============================================="
echo "Phase 3: Failing over masters..."
echo "=============================================="

for master in "${MASTERS[@]}"; do
    host=$(echo $master | cut -d: -f1)
    port=$(echo $master | cut -d: -f2)

    echo ""
    echo "Processing master: $host:$port"

    # Trouver une replica de ce master
    MASTER_ID=$(redis-cli -h "$host" -p "$port" CLUSTER MYID)

    # Trouver replica correspondante (updated)
    REPLICA_NODE=""
    for replica in "${REPLICAS[@]}"; do
        r_host=$(echo $replica | cut -d: -f1)
        r_port=$(echo $replica | cut -d: -f2)

        REPLICA_MASTER=$(redis-cli -h "$r_host" -p "$r_port" CLUSTER NODES | \
            grep myself | awk '{print $4}')

        if [ "$REPLICA_MASTER" = "$MASTER_ID" ]; then
            REPLICA_NODE="$replica"
            break
        fi
    done

    if [ -z "$REPLICA_NODE" ]; then
        echo "✗ No replica found for master $host:$port"
        exit 1
    fi

    echo "Found replica: $REPLICA_NODE"

    # Déclencher failover
    r_host=$(echo $REPLICA_NODE | cut -d: -f1)
    r_port=$(echo $REPLICA_NODE | cut -d: -f2)

    echo "Triggering failover on $r_host:$r_port..."
    redis-cli -h "$r_host" -p "$r_port" CLUSTER FAILOVER

    # Attendre fin du failover
    sleep 15

    # Vérifier que replica est devenue master
    NEW_ROLE=$(get_node_role "$REPLICA_NODE")

    if [ "$NEW_ROLE" = "master" ]; then
        echo "✓ Failover successful: $REPLICA_NODE is now master"
    else
        echo "✗ Failover failed: $REPLICA_NODE is still $NEW_ROLE"
        exit 1
    fi

    echo "Old master $host:$port is now replica"
done

echo ""
echo "✓ All masters failed over successfully"

# ============================================================================
# PHASE 4: UPDATE OLD MASTERS (NOW REPLICAS)
# ============================================================================

echo ""
echo "=============================================="
echo "Phase 4: Updating old masters..."
echo "=============================================="

for old_master in "${MASTERS[@]}"; do
    host=$(echo $old_master | cut -d: -f1)
    port=$(echo $old_master | cut -d: -f2)

    echo ""
    echo "Updating $host:$port (now replica)..."

    ssh "$host" << ENDSSH
        sudo systemctl stop redis-server
        sudo apt-get update
        sudo apt-get install -y redis-server=$NEW_VERSION
        sudo systemctl start redis-server
ENDSSH

    sleep 10

    STATE=$(redis-cli -h "$host" -p "$port" CLUSTER INFO | grep cluster_state | cut -d: -f2 | tr -d '\r')

    if [ "$STATE" = "ok" ]; then
        echo "✓ $host:$port updated and rejoined cluster"
    else
        echo "✗ $host:$port cluster state: $STATE"
        exit 1
    fi
done

# ============================================================================
# PHASE 5: VALIDATION
# ============================================================================

echo ""
echo "=============================================="
echo "Phase 5: Final validation..."
echo "=============================================="

# Vérifier version de tous les nodes
echo ""
echo "Cluster versions:"
for node in "${CLUSTER_NODES[@]}"; do
    host=$(echo $node | cut -d: -f1)
    port=$(echo $node | cut -d: -f2)

    version=$(redis-cli -h "$host" -p "$port" INFO server | grep redis_version | cut -d: -f2 | tr -d '\r')
    role=$(get_node_role "$node")

    echo "  $host:$port - Version: $version, Role: $role"
done

# Vérifier santé du cluster
echo ""
echo "Cluster health check:"
FIRST_NODE="${CLUSTER_NODES[0]}"
host=$(echo $FIRST_NODE | cut -d: -f1)
port=$(echo $FIRST_NODE | cut -d: -f2)

redis-cli -h "$host" -p "$port" CLUSTER INFO

echo ""
echo "=============================================="
echo "CLUSTER ROLLING UPDATE COMPLETED"
echo "=============================================="
echo "All nodes updated to version $NEW_VERSION"
echo "Cluster is healthy and operational"
echo "=============================================="
```

---

## Tests et validation

### 1. Tests pré-upgrade

```bash
#!/bin/bash
# pre-upgrade-tests.sh

echo "=== PRE-UPGRADE TESTS ==="

# Test 1: Connexion Redis
echo "1. Testing Redis connectivity..."
if redis-cli PING | grep -q "PONG"; then
    echo "   ✓ Redis is reachable"
else
    echo "   ✗ Cannot connect to Redis"
    exit 1
fi

# Test 2: Réplication
echo "2. Checking replication status..."
REPL_INFO=$(redis-cli INFO replication)
echo "$REPL_INFO" | grep "role:"
echo "$REPL_INFO" | grep "connected_slaves:"

# Test 3: Persistance
echo "3. Checking persistence..."
redis-cli INFO persistence | grep -E "rdb_last_save_time|aof_enabled"

# Test 4: Charge actuelle
echo "4. Current load..."
redis-cli INFO stats | grep -E "instantaneous_ops_per_sec|total_commands_processed"

# Test 5: Clients connectés
echo "5. Connected clients..."
redis-cli INFO clients | grep "connected_clients"

# Test 6: Mémoire utilisée
echo "6. Memory usage..."
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"

# Test 7: Dataset snapshot
echo "7. Taking dataset snapshot..."
NUM_KEYS=$(redis-cli DBSIZE | cut -d' ' -f1)
echo "   Total keys: $NUM_KEYS"

# Sauvegarder quelques clés pour validation post-upgrade
echo "   Sampling keys for validation..."
redis-cli --scan --pattern '*' | head -100 > /tmp/redis-keys-sample.txt

# Test 8: Write test
echo "8. Write test..."
TEST_KEY="pre-upgrade-test-$(date +%s)"
redis-cli SET "$TEST_KEY" "test-value" EX 3600
if redis-cli GET "$TEST_KEY" | grep -q "test-value"; then
    echo "   ✓ Write test passed"
    redis-cli DEL "$TEST_KEY"
else
    echo "   ✗ Write test failed"
    exit 1
fi

echo ""
echo "✓ All pre-upgrade tests passed"
```

### 2. Tests post-upgrade

```bash
#!/bin/bash
# post-upgrade-tests.sh

echo "=== POST-UPGRADE TESTS ==="

# Test 1: Version
echo "1. Checking Redis version..."
VERSION=$(redis-cli INFO server | grep redis_version | cut -d: -f2 | tr -d '\r')
echo "   Current version: $VERSION"

# Test 2: Réplication
echo "2. Checking replication..."
REPL_STATE=$(redis-cli INFO replication | grep "master_link_status" | cut -d: -f2 | tr -d '\r')
if [ "$REPL_STATE" = "up" ]; then
    echo "   ✓ Replication is up"
else
    echo "   ✗ Replication state: $REPL_STATE"
    exit 1
fi

# Test 3: Dataset integrity
echo "3. Checking dataset integrity..."
NUM_KEYS=$(redis-cli DBSIZE | cut -d' ' -f1)
echo "   Total keys: $NUM_KEYS"

# Vérifier échantillon de clés
MISSING=0
while read key; do
    if ! redis-cli EXISTS "$key" | grep -q "1"; then
        echo "   ✗ Missing key: $key"
        MISSING=$((MISSING + 1))
    fi
done < /tmp/redis-keys-sample.txt

if [ $MISSING -eq 0 ]; then
    echo "   ✓ All sampled keys present"
else
    echo "   ✗ $MISSING keys missing"
    exit 1
fi

# Test 4: Write/Read test
echo "4. Write/Read test..."
TEST_KEY="post-upgrade-test-$(date +%s)"
TEST_VALUE="success-$(date +%s)"

redis-cli SET "$TEST_KEY" "$TEST_VALUE" EX 3600
RESULT=$(redis-cli GET "$TEST_KEY")

if [ "$RESULT" = "$TEST_VALUE" ]; then
    echo "   ✓ Write/Read test passed"
    redis-cli DEL "$TEST_KEY"
else
    echo "   ✗ Write/Read test failed"
    exit 1
fi

# Test 5: Performance baseline
echo "5. Performance test..."
redis-benchmark -t set,get -n 10000 -q

# Test 6: Commandes spécifiques à la version
echo "6. Testing new version features..."
# Exemple: tester nouvelles commandes Redis 7.x
# redis-cli ACL LIST > /dev/null && echo "   ✓ ACL commands work"

echo ""
echo "✓ All post-upgrade tests passed"
```

---

## Rollback procedures

### 1. Rollback complet

```bash
#!/bin/bash
# rollback-redis.sh

set -e

echo "=============================================="
echo "REDIS ROLLBACK PROCEDURE"
echo "=============================================="

# Configuration
BACKUP_VERSION="6.2.10"
REDIS_HOSTS=("redis1.example.com" "redis2.example.com" "redis3.example.com")

read -p "Are you sure you want to rollback to $BACKUP_VERSION? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Rollback cancelled"
    exit 0
fi

echo ""
echo "Starting rollback to version $BACKUP_VERSION..."

for host in "${REDIS_HOSTS[@]}"; do
    echo ""
    echo "Rolling back $host..."

    ssh "$host" << ENDSSH
        # Stop Redis
        sudo systemctl stop redis-server

        # Restore backup binary
        if [ -f /usr/bin/redis-server.backup ]; then
            sudo cp /usr/bin/redis-server.backup /usr/bin/redis-server
            echo "Binary restored from backup"
        else
            # Reinstall old version
            sudo apt-get install -y redis-server=$BACKUP_VERSION
            echo "Old version reinstalled"
        fi

        # Restore config if needed
        if [ -f /etc/redis/redis.conf.backup ]; then
            sudo cp /etc/redis/redis.conf.backup /etc/redis/redis.conf
        fi

        # Start Redis
        sudo systemctl start redis-server

        # Verify
        sleep 3
        redis-cli PING
ENDSSH

    if [ $? -eq 0 ]; then
        echo "✓ $host rolled back successfully"
    else
        echo "✗ Rollback failed on $host"
        exit 1
    fi
done

echo ""
echo "=============================================="
echo "ROLLBACK COMPLETED"
echo "=============================================="
echo "All nodes rolled back to version $BACKUP_VERSION"
```

---

## Cas particuliers et pièges

### 1. Incompatibilités de version

```
┌─────────────────────────────────────────────────────────────────┐
│           CAS PARTICULIERS ET PIÈGES À ÉVITER                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. CHANGEMENTS RDB/AOF FORMAT                                  │
│     ├── Redis 7.0: Nouveau format RDB v10                       │
│     ├── Downgrade impossible si RDB sauvegardé en v10           │
│     └── Solution: Sauvegarder RDB AVANT upgrade                 │
│                                                                 │
│  2. COMMANDES DÉPRÉCIÉES                                        │
│     ├── MIGRATE avec COPY (removed en 7.0)                      │
│     ├── GEORADIUS (deprecated en 6.2, use GEOSEARCH)            │
│     └── Solution: Auditer code application avant upgrade        │
│                                                                 │
│  3. ACLs PAR DÉFAUT PLUS RESTRICTIFS                            │
│     ├── Redis 7+: Default user plus restreint                   │
│     ├── Peut casser applications existantes                     │
│     └── Solution: Tester ACLs en staging                        │
│                                                                 │
│  4. CHANGEMENTS MODULE API                                      │
│     ├── RedisJSON, RediSearch peuvent être incompatibles        │
│     ├── Vérifier compatibilité module AVANT upgrade             │
│     └── Update modules en même temps que Redis                  │
│                                                                 │
│  5. MAXMEMORY-POLICY COMPORTEMENT                               │
│     ├── Changements subtils dans éviction                       │
│     ├── volatile-lru peut agir différemment                     │
│     └── Solution: Monitorer évictions après upgrade             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Script de vérification compatibilité

```bash
#!/bin/bash
# check-upgrade-compatibility.sh

echo "=== REDIS UPGRADE COMPATIBILITY CHECK ==="

TARGET_VERSION="7.2"

# 1. Vérifier format RDB
echo "1. Checking RDB format..."
RDB_VERSION=$(redis-cli CONFIG GET dir | tail -1)
if [ -f "$RDB_VERSION/dump.rdb" ]; then
    RDB_VER=$(od -An -t x1 -N 9 "$RDB_VERSION/dump.rdb" | tr -d ' ')
    echo "   RDB version: $RDB_VER"
else
    echo "   No RDB file found"
fi

# 2. Vérifier commandes dépréciées dans logs
echo "2. Scanning for deprecated commands..."
grep -E "MIGRATE.*COPY|GEORADIUS" /var/log/redis/redis.log | head -5

# 3. Vérifier modules
echo "3. Checking loaded modules..."
redis-cli MODULE LIST

# 4. Vérifier ACLs
echo "4. Checking ACL configuration..."
redis-cli ACL LIST

# 5. Vérifier config pour breaking changes
echo "5. Checking configuration..."
redis-cli CONFIG GET maxmemory-policy
redis-cli CONFIG GET save

echo ""
echo "Review above output for potential issues"
```

---

## Checklist de mise à jour

### Checklist Pré-upgrade

- [ ] **Backup complet**
  - [ ] RDB backup récent
  - [ ] AOF backup si activé
  - [ ] Configuration redis.conf sauvegardée

- [ ] **Tests en staging**
  - [ ] Mise à jour testée sur environnement identique
  - [ ] Performance validée
  - [ ] Pas de régression détectée

- [ ] **Documentation**
  - [ ] Release notes lues
  - [ ] Breaking changes identifiés
  - [ ] Dépendances application vérifiées

- [ ] **Plan de rollback**
  - [ ] Procédure documentée
  - [ ] Binaires backup disponibles
  - [ ] Délai maximum défini

- [ ] **Communication**
  - [ ] Équipe informée
  - [ ] Maintenance window planifiée (si nécessaire)
  - [ ] Monitoring renforcé prévu

- [ ] **Outils**
  - [ ] Scripts de mise à jour testés
  - [ ] Accès SSH vérifié
  - [ ] Outils monitoring actifs

### Checklist Pendant upgrade

- [ ] **Phase 1: Replicas**
  - [ ] Update replicas un par un
  - [ ] Vérifier sync après chaque update
  - [ ] Monitoring latence

- [ ] **Phase 2: Failover**
  - [ ] Failover vers replica updated
  - [ ] Vérifier nouveau master
  - [ ] Reconfigurer replicas

- [ ] **Phase 3: Ancien master**
  - [ ] Update ancien master
  - [ ] Vérifier sync
  - [ ] Valider réplication

- [ ] **Tests continus**
  - [ ] Monitoring actif
  - [ ] Tests écriture/lecture
  - [ ] Latence surveillance

### Checklist Post-upgrade

- [ ] **Validation**
  - [ ] Version vérifiée sur tous nodes
  - [ ] Réplication fonctionnelle
  - [ ] Dataset intact (sample check)
  - [ ] Performance baseline validée

- [ ] **Configuration**
  - [ ] Nouvelles features activées si nécessaire
  - [ ] Configuration optimisée
  - [ ] Changements documentés

- [ ] **Monitoring**
  - [ ] Métriques normales
  - [ ] Pas d'erreurs logs
  - [ ] Alertes désactivées après stabilisation

- [ ] **Documentation**
  - [ ] Procédure réelle documentée
  - [ ] Incidents notés
  - [ ] Leçons apprises

---

## Automatisation avec Ansible

```yaml
# ansible/rolling-update-redis.yml
---
- name: Redis Rolling Update
  hosts: redis_cluster
  become: yes
  serial: 1  # Un node à la fois

  vars:
    redis_version: "7.2.3"
    redis_port: 6379

  tasks:
    - name: Check current Redis version
      command: redis-cli INFO server
      register: redis_info

    - name: Display current version
      debug:
        msg: "Current version: {{ redis_info.stdout | regex_search('redis_version:(.+)', '\\1') }}"

    - name: Check node role
      command: redis-cli -p {{ redis_port }} INFO replication
      register: redis_role

    - name: Stop Redis
      systemd:
        name: redis-server
        state: stopped

    - name: Backup current binary
      copy:
        src: /usr/bin/redis-server
        dest: /usr/bin/redis-server.backup
        remote_src: yes

    - name: Install new Redis version
      apt:
        name: redis-server={{ redis_version }}
        state: present
        update_cache: yes

    - name: Start Redis
      systemd:
        name: redis-server
        state: started
        enabled: yes

    - name: Wait for Redis to be ready
      wait_for:
        port: "{{ redis_port }}"
        delay: 5
        timeout: 60

    - name: Verify Redis is responding
      command: redis-cli -p {{ redis_port }} PING
      register: ping_result
      failed_when: "'PONG' not in ping_result.stdout"

    - name: Check replication sync (if replica)
      command: redis-cli -p {{ redis_port }} INFO replication
      register: repl_check
      when: "'slave' in redis_role.stdout"

    - name: Verify sync is complete
      assert:
        that:
          - "'master_link_status:up' in repl_check.stdout"
        fail_msg: "Replication not in sync"
      when: "'slave' in redis_role.stdout"

    - name: Pause between nodes
      pause:
        seconds: 30
```

---

## 📚 Résumé et bonnes pratiques

### Règles d'or

1. **Toujours mettre à jour les replicas avant le master**
2. **Tester en staging avec données réelles**
3. **Avoir un plan de rollback testé**
4. **Monitorer intensivement pendant et après**
5. **Documenter chaque étape**

### Downtime attendu

- **Master-Replica simple:** 5-30 secondes (failover)
- **Sentinel:** 5-30 secondes (failover automatique)
- **Cluster:** 0 seconde (si bien configuré)
- **Blue-Green:** 0 seconde (bascule trafic)

### Quand faire une mise à jour

- **Sécurité critique:** Immédiatement
- **Bug majeur:** Dans les 48h
- **Nouvelle feature:** Planifié en maintenance window
- **Upgrade majeur:** Après tests exhaustifs en staging

---

**Fin du Module 12 - Redis en Production et Sécurité** ✅

⏭️ [Monitoring et Observabilité](/13-monitoring-observabilite/README.md)
