🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.7 Tests de basculement et scénarios de failure

## Introduction

Tester régulièrement les mécanismes de failover est **CRITIQUE** pour garantir que votre infrastructure Redis Sentinel fonctionnera réellement lors d'une panne en production. Un failover non testé est un failover qui échouera probablement au pire moment.

Cette section couvre tous les aspects des tests de basculement : scénarios, scripts d'automatisation, validation, et best practices pour le chaos engineering.

**Principe fondamental** : *"Ce qui n'est pas testé ne fonctionne pas"*

---

## 🎯 Objectifs des tests de failover

### Pourquoi tester ?

```
┌─────────────────────────────────────────────────────────────────┐
│  OBJECTIFS DES TESTS DE FAILOVER                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. VALIDATION TECHNIQUE                                        │
│  ══════════════════════════════════════════                     │
│  • Vérifier que le failover fonctionne (temps, processus)       │
│  • Valider la configuration Sentinel (quorum, timeouts)         │
│  • Confirmer la détection de pannes                             │
│  • Tester la promotion replica → master                         │
│  • Vérifier la reconfiguration automatique                      │
│                                                                 │
│  2. VALIDATION APPLICATIVE                                      │
│  ══════════════════════════════════════════                     │
│  • Clients reconnectent automatiquement                         │
│  • Pas de perte de données (min-replicas-to-write)              │
│  • Performance maintenue post-failover                          │
│  • Monitoring et alerting fonctionnent                          │
│  • Scripts de notification s'exécutent                          │
│                                                                 │
│  3. VALIDATION OPÉRATIONNELLE                                   │
│  ══════════════════════════════════════════                     │
│  • Équipe connaît les procédures                                │
│  • Runbooks sont à jour                                         │
│  • Communication fonctionne (alertes, escalation)               │
│  • Post-mortem template préparé                                 │
│  • Temps de réponse mesurés                                     │
│                                                                 │
│  4. CONFORMITÉ ET AUDIT                                         │
│  ══════════════════════════════════════════                     │
│  • Démontrer capacité de disaster recovery                      │
│  • Respecter SLA/SLO                                            │
│  • Documenter RTO/RPO réels                                     │
│  • Prouver résilience aux auditors                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Métriques à mesurer

```
┌─────────────────────────────────────────────────────────────────┐
│  MÉTRIQUES CLÉS DU FAILOVER                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RTO (Recovery Time Objective)                                  │
│  ══════════════════════════════════════════════════             │
│  Temps maximal acceptable avant restauration du service         │
│                                                                 │
│  Composantes:                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ T1: Détection (SDOWN)           │ 30s (down-after-ms)      │ │
│  │ T2: Consensus (ODOWN)           │ 5s (votes Sentinel)      │ │
│  │ T3: Leader election             │ 2s (vote failover)       │ │
│  │ T4: Promotion replica           │ 5s (REPLICAOF NO ONE)    │ │
│  │ T5: Reconfiguration replicas    │ 10s (REPLICAOF new)      │ │
│  │ T6: Notification clients        │ Variable (Pub/Sub)       │ │
│  │ T7: Reconnexion clients         │ Variable (retry)         │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │ TOTAL RTO                       │ ~50-70s (typique)        │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  RPO (Recovery Point Objective)                                 │
│  ══════════════════════════════════════════════════             │
│  Perte de données maximale acceptable                           │
│                                                                 │
│  Avec min-replicas-to-write = 1:                                │
│  • RPO ≈ 0 (aucune écriture confirmée perdue)                   │
│  • Master isolé refuse writes immédiatement                     │
│                                                                 │
│  Sans min-replicas-to-write:                                    │
│  • RPO = données dans replication backlog non répliqué          │
│  • Potentiellement plusieurs secondes de données                │
│                                                                 │
│  Disponibilité (Uptime %)                                       │
│  ══════════════════════════════════════════════════             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ SLA Target   │ Downtime/an  │ Downtime/mois │ RTO requis│    │
│  ├──────────────┼──────────────┼───────────────┼───────────┤    │
│  │ 99%          │ 3.65 days    │ 7.2 hours     │ Minutes   │    │
│  │ 99.9%        │ 8.76 hours   │ 43.2 minutes  │ Minutes   │    │
│  │ 99.99%       │ 52.56 min    │ 4.32 minutes  │ 60s       │    │
│  │ 99.999%      │ 5.26 min     │ 25.9 seconds  │ 30s       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Autres métriques:                                              │
│  • Latence pendant failover (pic)                               │
│  • Erreurs clients pendant failover (%)                         │
│  • Temps pour notification complète                             │
│  • Temps de récupération ancien master                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🧪 Scénarios de test essentiels

### Scénario 1 : Panne master (clean shutdown)

**Description** : Arrêt propre du master (SIGTERM)

```bash
#!/bin/bash
# test-scenario-1-clean-master-shutdown.sh

echo "════════════════════════════════════════════════════════════"
echo "SCENARIO 1: Clean Master Shutdown"
echo "════════════════════════════════════════════════════════════"
echo ""

# Configuration
MASTER_HOST="10.0.1.10"
SENTINEL_HOST="10.0.1.20"
SENTINEL_PORT="26379"
MASTER_NAME="mymaster"

# Timestamp functions
timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

# Couleurs
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[$(timestamp)] INFO${NC}: $1"
}

log_warn() {
    echo -e "${YELLOW}[$(timestamp)] WARN${NC}: $1"
}

log_error() {
    echo -e "${RED}[$(timestamp)] ERROR${NC}: $1"
}

# ════════════════════════════════════════════════════════════════
# PHASE 0: PRÉ-TEST - ÉTAT INITIAL
# ════════════════════════════════════════════════════════════════

log_info "Phase 0: Capturing initial state"

INITIAL_MASTER=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
    SENTINEL get-master-addr-by-name $MASTER_NAME | head -n1)

log_info "Initial master: $INITIAL_MASTER"

# Vérifier nombre de replicas
REPLICA_COUNT=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
    SENTINEL master $MASTER_NAME | grep -A1 "num-slaves" | tail -n1)

log_info "Connected replicas: $REPLICA_COUNT"

if [ "$REPLICA_COUNT" -lt 1 ]; then
    log_error "No replicas connected, aborting test"
    exit 1
fi

# Capturer métriques baseline
log_info "Capturing baseline metrics..."
BASELINE_LATENCY=$(redis-cli -h $MASTER_HOST --latency-history -i 1 -c 5 2>/dev/null | tail -n1)
log_info "Baseline latency: $BASELINE_LATENCY"

# ════════════════════════════════════════════════════════════════
# PHASE 1: DÉCLENCHER PANNE
# ════════════════════════════════════════════════════════════════

log_warn "Phase 1: Triggering master shutdown"
START_TIME=$(date +%s)

# Arrêt propre (SIGTERM)
ssh root@$MASTER_HOST "systemctl stop redis-server"

if [ $? -eq 0 ]; then
    log_info "Master shutdown command sent"
else
    log_error "Failed to shutdown master"
    exit 1
fi

# ════════════════════════════════════════════════════════════════
# PHASE 2: OBSERVER DÉTECTION
# ════════════════════════════════════════════════════════════════

log_info "Phase 2: Observing failure detection..."

# Attendre SDOWN
log_info "Waiting for SDOWN..."
SDOWN_DETECTED=false
for i in {1..60}; do
    FLAGS=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
        SENTINEL master $MASTER_NAME | grep -A1 "^flags$" | tail -n1)

    if [[ "$FLAGS" == *"s_down"* ]]; then
        SDOWN_TIME=$(date +%s)
        SDOWN_ELAPSED=$((SDOWN_TIME - START_TIME))
        log_info "SDOWN detected after ${SDOWN_ELAPSED}s"
        SDOWN_DETECTED=true
        break
    fi
    sleep 1
done

if [ "$SDOWN_DETECTED" = false ]; then
    log_error "SDOWN not detected within 60s"
    exit 1
fi

# Attendre ODOWN
log_info "Waiting for ODOWN..."
ODOWN_DETECTED=false
for i in {1..30}; do
    FLAGS=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
        SENTINEL master $MASTER_NAME | grep -A1 "^flags$" | tail -n1)

    if [[ "$FLAGS" == *"o_down"* ]]; then
        ODOWN_TIME=$(date +%s)
        ODOWN_ELAPSED=$((ODOWN_TIME - START_TIME))
        log_info "ODOWN detected after ${ODOWN_ELAPSED}s"
        ODOWN_DETECTED=true
        break
    fi
    sleep 1
done

if [ "$ODOWN_DETECTED" = false ]; then
    log_error "ODOWN not detected within 30s"
    exit 1
fi

# ════════════════════════════════════════════════════════════════
# PHASE 3: OBSERVER FAILOVER
# ════════════════════════════════════════════════════════════════

log_info "Phase 3: Observing failover process..."

FAILOVER_DETECTED=false
for i in {1..120}; do
    FLAGS=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
        SENTINEL master $MASTER_NAME | grep -A1 "^flags$" | tail -n1)

    if [[ "$FLAGS" == *"failover"* ]]; then
        log_info "Failover in progress..."
    fi

    # Vérifier si nouveau master
    CURRENT_MASTER=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
        SENTINEL get-master-addr-by-name $MASTER_NAME | head -n1)

    if [ "$CURRENT_MASTER" != "$INITIAL_MASTER" ]; then
        FAILOVER_TIME=$(date +%s)
        FAILOVER_ELAPSED=$((FAILOVER_TIME - START_TIME))
        log_info "Failover completed after ${FAILOVER_ELAPSED}s"
        log_info "New master: $CURRENT_MASTER"
        FAILOVER_DETECTED=true
        break
    fi

    sleep 1
done

if [ "$FAILOVER_DETECTED" = false ]; then
    log_error "Failover not completed within 120s"
    exit 1
fi

# ════════════════════════════════════════════════════════════════
# PHASE 4: VALIDATION POST-FAILOVER
# ════════════════════════════════════════════════════════════════

log_info "Phase 4: Validating post-failover state..."

# Attendre stabilisation
sleep 10

# Vérifier nouveau master accessible
if redis-cli -h $CURRENT_MASTER PING >/dev/null 2>&1; then
    log_info "✅ New master is accessible"
else
    log_error "❌ New master is NOT accessible"
fi

# Vérifier mode master
ROLE=$(redis-cli -h $CURRENT_MASTER ROLE | head -n1)
if [ "$ROLE" = "master" ]; then
    log_info "✅ New master has role 'master'"
else
    log_error "❌ New master role is '$ROLE' (expected 'master')"
fi

# Vérifier replicas reconnectés
NEW_REPLICA_COUNT=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
    SENTINEL master $MASTER_NAME | grep -A1 "num-slaves" | tail -n1)
log_info "Replicas after failover: $NEW_REPLICA_COUNT"

# Test write
TEST_KEY="failover_test:$(date +%s)"
if redis-cli -h $CURRENT_MASTER SET $TEST_KEY "success" EX 60 >/dev/null 2>&1; then
    log_info "✅ Write test successful"
else
    log_error "❌ Write test failed"
fi

# Test read
TEST_VALUE=$(redis-cli -h $CURRENT_MASTER GET $TEST_KEY)
if [ "$TEST_VALUE" = "success" ]; then
    log_info "✅ Read test successful"
else
    log_error "❌ Read test failed"
fi

# Cleanup
redis-cli -h $CURRENT_MASTER DEL $TEST_KEY >/dev/null 2>&1

# ════════════════════════════════════════════════════════════════
# PHASE 5: RAPPORT FINAL
# ════════════════════════════════════════════════════════════════

END_TIME=$(date +%s)
TOTAL_DURATION=$((END_TIME - START_TIME))

echo ""
echo "════════════════════════════════════════════════════════════"
echo "TEST REPORT - SCENARIO 1"
echo "════════════════════════════════════════════════════════════"
echo ""
echo "Initial master:    $INITIAL_MASTER"
echo "New master:        $CURRENT_MASTER"
echo ""
echo "SDOWN detected:    ${SDOWN_ELAPSED}s"
echo "ODOWN detected:    ${ODOWN_ELAPSED}s"
echo "Failover complete: ${FAILOVER_ELAPSED}s"
echo "Total duration:    ${TOTAL_DURATION}s"
echo ""
echo "RTO (actual):      ${FAILOVER_ELAPSED}s"
echo "RPO (estimated):   0s (clean shutdown with min-replicas-to-write)"
echo ""
echo "════════════════════════════════════════════════════════════"

# Retourner l'ancien master en ligne (manuel)
log_warn "Remember to bring back old master:"
log_warn "  ssh root@$INITIAL_MASTER 'systemctl start redis-server'"
```

### Scénario 2 : Panne brutale master (kill -9)

```bash
#!/bin/bash
# test-scenario-2-master-kill.sh

echo "════════════════════════════════════════════════════════════"
echo "SCENARIO 2: Brutal Master Kill (SIGKILL)"
echo "════════════════════════════════════════════════════════════"
echo ""

MASTER_HOST="10.0.1.10"
SENTINEL_HOST="10.0.1.20"
SENTINEL_PORT="26379"
MASTER_NAME="mymaster"

timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

log_info() {
    echo "[$(timestamp)] INFO: $1"
}

log_warn() {
    echo "[$(timestamp)] WARN: $1"
}

# Phase 0: État initial
log_info "Capturing initial state..."
INITIAL_MASTER=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
    SENTINEL get-master-addr-by-name $MASTER_NAME | head -n1)
log_info "Initial master: $INITIAL_MASTER"

# Phase 1: Kill brutal
log_warn "Sending SIGKILL to master..."
START_TIME=$(date +%s)

# Trouver PID et kill -9
MASTER_PID=$(ssh root@$MASTER_HOST "pgrep -f 'redis-server.*6379'")
if [ -n "$MASTER_PID" ]; then
    ssh root@$MASTER_HOST "kill -9 $MASTER_PID"
    log_info "Sent SIGKILL to PID $MASTER_PID"
else
    log_warn "Master PID not found"
    exit 1
fi

# Phase 2: Observer failover
log_info "Observing failover..."
FAILOVER_DETECTED=false
for i in {1..120}; do
    CURRENT_MASTER=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
        SENTINEL get-master-addr-by-name $MASTER_NAME 2>/dev/null | head -n1)

    if [ "$CURRENT_MASTER" != "$INITIAL_MASTER" ] && [ -n "$CURRENT_MASTER" ]; then
        FAILOVER_TIME=$(date +%s)
        FAILOVER_ELAPSED=$((FAILOVER_TIME - START_TIME))
        log_info "Failover completed after ${FAILOVER_ELAPSED}s"
        log_info "New master: $CURRENT_MASTER"
        FAILOVER_DETECTED=true
        break
    fi
    sleep 1
done

if [ "$FAILOVER_DETECTED" = false ]; then
    log_warn "Failover not completed within 120s"
    exit 1
fi

# Phase 3: Validation
sleep 5
log_info "Validating new master..."

if redis-cli -h $CURRENT_MASTER PING >/dev/null 2>&1; then
    log_info "✅ New master responsive"
else
    log_warn "❌ New master not responsive"
fi

# Test write
if redis-cli -h $CURRENT_MASTER SET "test:kill" "recovered" EX 60 >/dev/null 2>&1; then
    log_info "✅ Write test passed"
else
    log_warn "❌ Write test failed"
fi

# Rapport
echo ""
echo "════════════════════════════════════════════════════════════"
echo "TEST REPORT - SCENARIO 2 (BRUTAL KILL)"
echo "════════════════════════════════════════════════════════════"
echo "Initial master: $INITIAL_MASTER"
echo "New master:     $CURRENT_MASTER"
echo "RTO:            ${FAILOVER_ELAPSED}s"
echo ""
echo "Note: SIGKILL simule crash serveur, panne hardware, kernel panic"
echo "════════════════════════════════════════════════════════════"

# Rappel recovery
log_warn "To recover old master:"
log_warn "  ssh root@$INITIAL_MASTER 'systemctl start redis-server'"
log_warn "  # Il deviendra automatiquement replica du nouveau master"
```

### Scénario 3 : Partition réseau

```bash
#!/bin/bash
# test-scenario-3-network-partition.sh

echo "════════════════════════════════════════════════════════════"
echo "SCENARIO 3: Network Partition"
echo "════════════════════════════════════════════════════════════"
echo ""

MASTER_HOST="10.0.1.10"
SENTINEL_HOST="10.0.1.20"
SENTINEL_PORT="26379"
MASTER_NAME="mymaster"

# Vérifier si iptables disponible
if ! command -v iptables &> /dev/null; then
    echo "ERROR: iptables not available"
    exit 1
fi

timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

log_info() {
    echo "[$(timestamp)] INFO: $1"
}

log_warn() {
    echo "[$(timestamp)] WARN: $1"
}

# Phase 0: État initial
log_info "Initial state check..."
INITIAL_MASTER=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
    SENTINEL get-master-addr-by-name $MASTER_NAME | head -n1)
log_info "Initial master: $INITIAL_MASTER"

# Phase 1: Créer partition réseau
log_warn "Creating network partition..."
log_info "Blocking traffic to/from master..."

START_TIME=$(date +%s)

# Bloquer tout traffic vers master (sur le master lui-même)
ssh root@$MASTER_HOST <<'EOF'
# Sauvegarder règles actuelles
iptables-save > /tmp/iptables-backup.rules

# Bloquer tout INPUT (sauf localhost)
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -j DROP

# Bloquer tout OUTPUT (sauf localhost)
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -j DROP

echo "Network partition created"
EOF

log_info "Master is now isolated"

# Phase 2: Observer failover
log_info "Observing failover due to partition..."
FAILOVER_DETECTED=false

for i in {1..120}; do
    CURRENT_MASTER=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
        SENTINEL get-master-addr-by-name $MASTER_NAME 2>/dev/null | head -n1)

    if [ "$CURRENT_MASTER" != "$INITIAL_MASTER" ] && [ -n "$CURRENT_MASTER" ]; then
        FAILOVER_TIME=$(date +%s)
        FAILOVER_ELAPSED=$((FAILOVER_TIME - START_TIME))
        log_info "Failover completed after ${FAILOVER_ELAPSED}s"
        log_info "New master: $CURRENT_MASTER"
        FAILOVER_DETECTED=true
        break
    fi

    sleep 1
done

if [ "$FAILOVER_DETECTED" = false ]; then
    log_warn "Failover not detected within 120s"
fi

# Phase 3: Valider nouveau master
log_info "Validating new master..."
sleep 5

if redis-cli -h $CURRENT_MASTER PING >/dev/null 2>&1; then
    log_info "✅ New master is accessible"
fi

# Test min-replicas-to-write sur ancien master
log_info "Testing min-replicas-to-write protection on isolated master..."
# (On ne peut pas tester directement car master est isolé)
log_info "Old master should be rejecting writes (if min-replicas-to-write configured)"

# Phase 4: Restaurer réseau
log_warn "Restoring network..."
RESTORE_TIME=$(date +%s)

ssh root@$MASTER_HOST <<'EOF'
# Restaurer règles iptables
if [ -f /tmp/iptables-backup.rules ]; then
    iptables-restore < /tmp/iptables-backup.rules
    rm /tmp/iptables-backup.rules
else
    # Si pas de backup, flush tout
    iptables -F
    iptables -X
    iptables -P INPUT ACCEPT
    iptables -P FORWARD ACCEPT
    iptables -P OUTPUT ACCEPT
fi

echo "Network restored"
EOF

log_info "Network partition removed"

# Phase 5: Observer réconciliation
log_info "Observing reconciliation..."
sleep 10

# Vérifier que ancien master devient replica
OLD_MASTER_ROLE=$(redis-cli -h $INITIAL_MASTER ROLE 2>/dev/null | head -n1)
if [ "$OLD_MASTER_ROLE" = "slave" ]; then
    log_info "✅ Old master converted to replica"
else
    log_warn "⚠️  Old master role: $OLD_MASTER_ROLE (expected 'slave')"
fi

# Vérifier nombre de replicas sur nouveau master
REPLICA_COUNT=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT \
    SENTINEL master $MASTER_NAME | grep -A1 "num-slaves" | tail -n1)
log_info "Replicas after reconciliation: $REPLICA_COUNT"

# Rapport
RESTORE_ELAPSED=$((RESTORE_TIME - START_TIME))
TOTAL_DURATION=$(( $(date +%s) - START_TIME ))

echo ""
echo "════════════════════════════════════════════════════════════"
echo "TEST REPORT - SCENARIO 3 (NETWORK PARTITION)"
echo "════════════════════════════════════════════════════════════"
echo "Initial master:           $INITIAL_MASTER"
echo "New master:               $CURRENT_MASTER"
echo "Old master role after:    $OLD_MASTER_ROLE"
echo ""
echo "Failover time:            ${FAILOVER_ELAPSED}s"
echo "Partition duration:       ${RESTORE_ELAPSED}s"
echo "Total test duration:      ${TOTAL_DURATION}s"
echo ""
echo "Split-brain protection:   $([ -n "$MIN_REPLICAS" ] && echo "Enabled" || echo "Check manually")"
echo ""
echo "════════════════════════════════════════════════════════════"
```

### Scénario 4 : Panne Sentinel

```bash
#!/bin/bash
# test-scenario-4-sentinel-failure.sh

echo "════════════════════════════════════════════════════════════"
echo "SCENARIO 4: Sentinel Failure"
echo "════════════════════════════════════════════════════════════"
echo ""

SENTINELS=("10.0.1.20" "10.0.1.21" "10.0.1.22")
SENTINEL_PORT="26379"
MASTER_NAME="mymaster"

timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

log_info() {
    echo "[$(timestamp)] INFO: $1"
}

# Phase 0: État initial
log_info "Checking initial Sentinel state..."

ACTIVE_SENTINELS=0
for sentinel in "${SENTINELS[@]}"; do
    if redis-cli -h $sentinel -p $SENTINEL_PORT PING >/dev/null 2>&1; then
        ACTIVE_SENTINELS=$((ACTIVE_SENTINELS + 1))
        log_info "Sentinel $sentinel: ACTIVE"
    else
        log_info "Sentinel $sentinel: DOWN"
    fi
done

log_info "Active Sentinels: $ACTIVE_SENTINELS"

if [ $ACTIVE_SENTINELS -lt 3 ]; then
    log_info "WARNING: Not all Sentinels active initially"
fi

# Phase 1: Arrêter 1 Sentinel
TARGET_SENTINEL="${SENTINELS[0]}"
log_info "Stopping Sentinel: $TARGET_SENTINEL"

ssh root@$TARGET_SENTINEL "systemctl stop redis-sentinel"

if [ $? -eq 0 ]; then
    log_info "✅ Sentinel stopped"
else
    log_info "❌ Failed to stop Sentinel"
    exit 1
fi

sleep 5

# Phase 2: Vérifier quorum
log_info "Checking if quorum still reachable..."

REMAINING_SENTINEL="${SENTINELS[1]}"
QUORUM=$(redis-cli -h $REMAINING_SENTINEL -p $SENTINEL_PORT \
    SENTINEL master $MASTER_NAME | grep -A1 "^quorum$" | tail -n1)

log_info "Configured quorum: $QUORUM"

CKQUORUM=$(redis-cli -h $REMAINING_SENTINEL -p $SENTINEL_PORT \
    SENTINEL ckquorum $MASTER_NAME)

if echo "$CKQUORUM" | grep -q "OK"; then
    log_info "✅ Quorum still reachable with 2 Sentinels"
else
    log_info "❌ Quorum NOT reachable"
fi

# Phase 3: Tester failover avec Sentinel down
log_info "Testing if failover still possible..."
log_info "(This requires manually triggering master failure)"
log_info "Command: redis-cli -h $REMAINING_SENTINEL -p $SENTINEL_PORT SENTINEL failover $MASTER_NAME"

# Phase 4: Redémarrer Sentinel
log_info "Restarting Sentinel: $TARGET_SENTINEL"
ssh root@$TARGET_SENTINEL "systemctl start redis-sentinel"

sleep 5

# Vérifier reconnexion
if redis-cli -h $TARGET_SENTINEL -p $SENTINEL_PORT PING >/dev/null 2>&1; then
    log_info "✅ Sentinel restarted and accessible"
else
    log_info "❌ Sentinel not accessible after restart"
fi

# Rapport
echo ""
echo "════════════════════════════════════════════════════════════"
echo "TEST REPORT - SCENARIO 4 (SENTINEL FAILURE)"
echo "════════════════════════════════════════════════════════════"
echo "Total Sentinels:          3"
echo "Configured quorum:        $QUORUM"
echo "Quorum after 1 down:      $(echo "$CKQUORUM" | grep -q "OK" && echo "OK" || echo "FAIL")"
echo ""
echo "Conclusion:"
echo "• With 3 Sentinels and quorum=2:"
echo "  → Can tolerate 1 Sentinel failure"
echo "  → Failover still possible with 2 remaining"
echo ""
echo "════════════════════════════════════════════════════════════"
```

---

## 🤖 Suite de tests automatisée

### Test runner complet

```bash
#!/bin/bash
# redis-sentinel-test-suite.sh

set -e

# Configuration
TEST_SUITE_VERSION="1.0.0"
RESULTS_DIR="./test-results-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RESULTS_DIR"

# Couleurs
GREEN='\033[0;32m'
RED='\033[0;31m'
BLUE='\033[0;34m'
NC='\033[0m'

# ════════════════════════════════════════════════════════════════
# FONCTIONS UTILITAIRES
# ════════════════════════════════════════════════════════════════

banner() {
    echo ""
    echo "════════════════════════════════════════════════════════════"
    echo "$1"
    echo "════════════════════════════════════════════════════════════"
    echo ""
}

test_header() {
    echo ""
    echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${BLUE}  TEST: $1${NC}"
    echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo ""
}

test_pass() {
    echo -e "${GREEN}✅ PASS${NC}: $1"
}

test_fail() {
    echo -e "${RED}❌ FAIL${NC}: $1"
}

# ════════════════════════════════════════════════════════════════
# TESTS DE VALIDATION PRÉ-FAILOVER
# ════════════════════════════════════════════════════════════════

run_pre_checks() {
    test_header "PRE-FAILOVER VALIDATION"

    local test_log="$RESULTS_DIR/pre-checks.log"

    # Test 1: Sentinel connectivity
    echo "Test 1: Sentinel Connectivity" | tee -a "$test_log"
    ACTIVE_SENTINELS=0
    for sentinel in "${SENTINELS[@]}"; do
        if redis-cli -h "$sentinel" -p 26379 PING &>/dev/null; then
            ACTIVE_SENTINELS=$((ACTIVE_SENTINELS + 1))
            test_pass "Sentinel $sentinel reachable" | tee -a "$test_log"
        else
            test_fail "Sentinel $sentinel unreachable" | tee -a "$test_log"
        fi
    done

    if [ "$ACTIVE_SENTINELS" -ge 3 ]; then
        test_pass "All Sentinels active ($ACTIVE_SENTINELS/3)" | tee -a "$test_log"
    else
        test_fail "Not all Sentinels active ($ACTIVE_SENTINELS/3)" | tee -a "$test_log"
    fi

    # Test 2: Quorum configuration
    echo ""
    echo "Test 2: Quorum Configuration" | tee -a "$test_log"
    QUORUM=$(redis-cli -h "${SENTINELS[0]}" -p 26379 \
        SENTINEL master "$MASTER_NAME" | grep -A1 "^quorum$" | tail -n1)

    EXPECTED_QUORUM=$(( (${#SENTINELS[@]} / 2) + 1 ))

    if [ "$QUORUM" -eq "$EXPECTED_QUORUM" ]; then
        test_pass "Quorum optimal: $QUORUM" | tee -a "$test_log"
    else
        test_fail "Quorum suboptimal: $QUORUM (expected $EXPECTED_QUORUM)" | tee -a "$test_log"
    fi

    # Test 3: Replica connectivity
    echo ""
    echo "Test 3: Replica Count" | tee -a "$test_log"
    REPLICA_COUNT=$(redis-cli -h "${SENTINELS[0]}" -p 26379 \
        SENTINEL master "$MASTER_NAME" | grep -A1 "num-slaves" | tail -n1)

    if [ "$REPLICA_COUNT" -ge 2 ]; then
        test_pass "Sufficient replicas: $REPLICA_COUNT" | tee -a "$test_log"
    else
        test_fail "Insufficient replicas: $REPLICA_COUNT (need ≥2)" | tee -a "$test_log"
    fi

    # Test 4: Master protection (min-replicas-to-write)
    echo ""
    echo "Test 4: Split-Brain Protection" | tee -a "$test_log"
    MASTER_ADDR=$(redis-cli -h "${SENTINELS[0]}" -p 26379 \
        SENTINEL get-master-addr-by-name "$MASTER_NAME" | head -n1)

    MIN_REPLICAS=$(redis-cli -h "$MASTER_ADDR" CONFIG GET min-replicas-to-write | tail -n1)

    if [ "$MIN_REPLICAS" -ge 1 ]; then
        test_pass "min-replicas-to-write: $MIN_REPLICAS" | tee -a "$test_log"
    else
        test_fail "min-replicas-to-write: $MIN_REPLICAS (should be ≥1)" | tee -a "$test_log"
    fi

    echo ""
}

# ════════════════════════════════════════════════════════════════
# TEST FAILOVER
# ════════════════════════════════════════════════════════════════

run_failover_test() {
    test_header "FAILOVER TEST"

    local test_log="$RESULTS_DIR/failover.log"

    echo "Triggering manual failover..." | tee -a "$test_log"
    START_TIME=$(date +%s)

    redis-cli -h "${SENTINELS[0]}" -p 26379 \
        SENTINEL failover "$MASTER_NAME" | tee -a "$test_log"

    # Observer failover
    echo "Observing failover progress..." | tee -a "$test_log"

    INITIAL_MASTER=$(redis-cli -h "${SENTINELS[0]}" -p 26379 \
        SENTINEL get-master-addr-by-name "$MASTER_NAME" | head -n1)

    FAILOVER_COMPLETED=false
    for i in {1..60}; do
        sleep 1
        CURRENT_MASTER=$(redis-cli -h "${SENTINELS[0]}" -p 26379 \
            SENTINEL get-master-addr-by-name "$MASTER_NAME" | head -n1)

        if [ "$CURRENT_MASTER" != "$INITIAL_MASTER" ]; then
            FAILOVER_TIME=$(($(date +%s) - START_TIME))
            echo "Failover completed in ${FAILOVER_TIME}s" | tee -a "$test_log"
            echo "New master: $CURRENT_MASTER" | tee -a "$test_log"
            FAILOVER_COMPLETED=true
            break
        fi
    done

    if [ "$FAILOVER_COMPLETED" = true ]; then
        test_pass "Failover successful (${FAILOVER_TIME}s)" | tee -a "$test_log"
    else
        test_fail "Failover did not complete within 60s" | tee -a "$test_log"
    fi

    echo ""
}

# ════════════════════════════════════════════════════════════════
# TESTS POST-FAILOVER
# ════════════════════════════════════════════════════════════════

run_post_checks() {
    test_header "POST-FAILOVER VALIDATION"

    local test_log="$RESULTS_DIR/post-checks.log"

    # Attendre stabilisation
    sleep 10

    # Test 1: Master accessibility
    echo "Test 1: New Master Accessibility" | tee -a "$test_log"
    NEW_MASTER=$(redis-cli -h "${SENTINELS[0]}" -p 26379 \
        SENTINEL get-master-addr-by-name "$MASTER_NAME" | head -n1)

    if redis-cli -h "$NEW_MASTER" PING &>/dev/null; then
        test_pass "New master $NEW_MASTER accessible" | tee -a "$test_log"
    else
        test_fail "New master $NEW_MASTER NOT accessible" | tee -a "$test_log"
    fi

    # Test 2: Write capability
    echo ""
    echo "Test 2: Write Capability" | tee -a "$test_log"
    TEST_KEY="test:failover:$(date +%s)"

    if redis-cli -h "$NEW_MASTER" SET "$TEST_KEY" "success" EX 60 &>/dev/null; then
        test_pass "Write successful" | tee -a "$test_log"
    else
        test_fail "Write failed" | tee -a "$test_log"
    fi

    # Test 3: Read capability
    echo ""
    echo "Test 3: Read Capability" | tee -a "$test_log"
    TEST_VALUE=$(redis-cli -h "$NEW_MASTER" GET "$TEST_KEY" 2>/dev/null)

    if [ "$TEST_VALUE" = "success" ]; then
        test_pass "Read successful" | tee -a "$test_log"
    else
        test_fail "Read failed" | tee -a "$test_log"
    fi

    # Cleanup
    redis-cli -h "$NEW_MASTER" DEL "$TEST_KEY" &>/dev/null

    # Test 4: Replication reconstitution
    echo ""
    echo "Test 4: Replication Status" | tee -a "$test_log"
    NEW_REPLICA_COUNT=$(redis-cli -h "${SENTINELS[0]}" -p 26379 \
        SENTINEL master "$MASTER_NAME" | grep -A1 "num-slaves" | tail -n1)

    if [ "$NEW_REPLICA_COUNT" -ge 1 ]; then
        test_pass "Replicas reconnected: $NEW_REPLICA_COUNT" | tee -a "$test_log"
    else
        test_fail "No replicas connected" | tee -a "$test_log"
    fi

    echo ""
}

# ════════════════════════════════════════════════════════════════
# RAPPORT FINAL
# ════════════════════════════════════════════════════════════════

generate_report() {
    local report_file="$RESULTS_DIR/test-report.md"

    cat > "$report_file" <<EOF
# Redis Sentinel Failover Test Report

**Date:** $(date '+%Y-%m-%d %H:%M:%S')
**Test Suite Version:** $TEST_SUITE_VERSION

## Summary

- **Total Tests:** $(grep -r "PASS\|FAIL" "$RESULTS_DIR"/*.log | wc -l)
- **Passed:** $(grep -r "PASS" "$RESULTS_DIR"/*.log | wc -l)
- **Failed:** $(grep -r "FAIL" "$RESULTS_DIR"/*.log | wc -l)

## Pre-Failover Checks

\`\`\`
$(cat "$RESULTS_DIR/pre-checks.log")
\`\`\`

## Failover Test

\`\`\`
$(cat "$RESULTS_DIR/failover.log")
\`\`\`

## Post-Failover Checks

\`\`\`
$(cat "$RESULTS_DIR/post-checks.log")
\`\`\`

## Metrics

- **RTO (observed):** $(grep "Failover completed" "$RESULTS_DIR/failover.log" | awk '{print $NF}')
- **RPO:** 0s (manual failover, no data loss)

## Conclusion

$(if [ "$(grep -r "FAIL" "$RESULTS_DIR"/*.log | wc -l)" -eq 0 ]; then
    echo "✅ All tests passed. Sentinel failover working as expected."
else
    echo "⚠️ Some tests failed. Review logs for details."
fi)

## Files

- Pre-checks: \`pre-checks.log\`
- Failover: \`failover.log\`
- Post-checks: \`post-checks.log\`
EOF

    echo ""
    echo "Report generated: $report_file"
}

# ════════════════════════════════════════════════════════════════
# MAIN
# ════════════════════════════════════════════════════════════════

main() {
    banner "REDIS SENTINEL FAILOVER TEST SUITE v${TEST_SUITE_VERSION}"

    # Configuration
    SENTINELS=("10.0.1.20" "10.0.1.21" "10.0.1.22")
    MASTER_NAME="mymaster"

    echo "Configuration:"
    echo "  Sentinels: ${SENTINELS[*]}"
    echo "  Master name: $MASTER_NAME"
    echo "  Results dir: $RESULTS_DIR"
    echo ""

    read -p "Press Enter to start tests..." </dev/tty

    # Exécuter tests
    run_pre_checks
    run_failover_test
    run_post_checks

    # Générer rapport
    generate_report

    banner "TEST SUITE COMPLETE"
    echo "Results saved in: $RESULTS_DIR"
}

# Exécuter
main "$@"
```

---

## 🎓 Points clés à retenir

1. **Tests réguliers obligatoires** : Minimum mensuel en production
2. **Environnement dédié** : Tester en pre-prod d'abord
3. **Documentation** : Chaque test doit être documenté
4. **Métriques** : Mesurer RTO/RPO réels à chaque test
5. **Automation** : Scripts pour répétabilité
6. **Scénarios variés** : Clean shutdown, kill, partition réseau
7. **Validation complète** : Pre-check, failover, post-check
8. **Équipe formée** : Tous doivent connaître les procédures
9. **Chaos engineering** : Tests imprévisibles en production
10. **Amélioration continue** : Ajuster config selon résultats

---

## 🔗 Références

- [Redis Sentinel Testing](https://redis.io/docs/management/sentinel/)
- [Chaos Engineering Principles](https://principlesofchaos.org/)
- [Site Reliability Engineering (Google)](https://sre.google/books/)

---


⏭️ [Architecture Distribuée et Scaling Horizontal](/11-architecture-distribuee-scaling/README.md)
