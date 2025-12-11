🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.2 Topologie de réplication (chaînée, en étoile)

## Introduction

La topologie de réplication définit comment les instances Redis sont organisées et interconnectées pour former un système de haute disponibilité. Le choix de la topologie impacte directement la latence, la scalabilité, la résilience et la complexité opérationnelle de votre infrastructure Redis.

Cette section explore les différentes topologies possibles, leurs avantages, inconvénients et cas d'usage appropriés.

---

## 🌟 Topologie 1 : Étoile (Star / Fan-out)

### Architecture

La topologie en étoile est la plus simple et la plus courante : un master central réplique vers plusieurs replicas de même niveau.

```
┌─────────────────────────────────────────────────────────────────┐
│  TOPOLOGIE EN ÉTOILE (STAR)                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     ┌────────────────┐                          │
│                     │                │                          │
│                     │  MASTER (M)    │                          │
│                     │  10.0.1.10     │                          │
│                     │  Zone A        │                          │
│                     │                │                          │
│                     └────────┬───────┘                          │
│                              │                                  │
│                              │ Replication                      │
│                              │ (tous au même niveau)            │
│              ┌───────────────┼───────────────┐                  │
│              │               │               │                  │
│              ▼               ▼               ▼                  │
│        ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│        │REPLICA 1 │    │REPLICA 2 │    │REPLICA 3 │             │
│        │10.0.1.11 │    │10.0.1.12 │    │10.0.1.13 │             │
│        │ Zone A   │    │ Zone B   │    │ Zone C   │             │
│        └──────────┘    └──────────┘    └──────────┘             │
│             │               │               │                   │
│             │               │               │                   │
│             └───────────────┴───────────────┘                   │
│                            │                                    │
│                     Read Operations                             │
│                            │                                    │
│                            ▼                                    │
│                    ┌──────────────┐                             │
│                    │Load Balancer │                             │
│                    │  (Optional)  │                             │
│                    └──────────────┘                             │
│                                                                 │
│  Caractéristiques:                                              │
│  • Latence uniforme: tous les replicas ont la même latence      │
│  • Scalabilité reads: ajout de replicas = plus de capacité      │
│  • Bande passante: N replicas = N × bandwidth master            │
│  • SPOF: Le master est un point de défaillance unique           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

**Master (10.0.1.10)** :

```ini
# redis-master-star.conf

################################# NETWORK ####################################
bind 0.0.0.0
port 6379
protected-mode yes
requirepass "MasterPass2024!"
masterauth "MasterPass2024!"

################################ REPLICATION #################################
# Configuration pour supporter N replicas
repl-diskless-sync yes
repl-diskless-sync-delay 5  # Attendre pour batcher les syncs
repl-diskless-sync-max-replicas 0  # Pas de limite

# Backlog dimensionné pour gérer déconnexions
repl-backlog-size 256mb
repl-backlog-ttl 3600

# Protection: Minimum 2 replicas connectés
min-replicas-to-write 2
min-replicas-max-lag 10

# Client output buffer: CRITIQUE pour topologie étoile
# Avec N replicas, le buffer doit supporter N × traffic
client-output-buffer-limit replica 1gb 256mb 60
# Format: <hard> <soft> <soft_seconds>
# Hard: 1GB → déconnexion immédiate si dépassé
# Soft: 256MB pendant 60s → déconnexion

################################ PERSISTENCE #################################
appendonly yes
appendfsync everysec
save ""  # Désactiver RDB auto-save

################################# MEMORY #####################################
maxmemory 8gb
maxmemory-policy allkeys-lru

################################### LOGS #####################################
loglevel notice
logfile "/var/log/redis/master-star.log"

# Monitoring des replicas
# Ajouter dans monitoring:
# - connected_slaves (doit être 3)
# - Lag de chaque replica (<10s)
```

**Replica 1, 2, 3** (configurations identiques sauf IPs) :

```ini
# redis-replica-star.conf

################################# NETWORK ####################################
bind 0.0.0.0
port 6379
protected-mode yes
requirepass "ReplicaPass2024!"
masterauth "MasterPass2024!"

################################ REPLICATION #################################
# Pointer vers le master
replicaof 10.0.1.10 6379

replica-read-only yes
replica-priority 100  # Ajuster pour chaque replica (100, 90, 80)

# Timeouts adaptés à la topologie
repl-timeout 60
repl-ping-replica-period 10

# Servir données obsolètes si master down?
replica-serve-stale-data yes  # "no" si cohérence stricte nécessaire

# Pour Docker/NAT (optionnel)
replica-announce-ip 10.0.1.11  # IP publique de ce replica
replica-announce-port 6379

################################ PERSISTENCE #################################
appendonly yes
appendfsync everysec

################################# MEMORY #####################################
maxmemory 8gb
maxmemory-policy allkeys-lru

################################### LOGS #####################################
loglevel notice
logfile "/var/log/redis/replica1-star.log"
```

### Analyse de performance

**Bande passante master** :

```
┌─────────────────────────────────────────────────────────────────┐
│  CONSOMMATION BANDE PASSANTE - TOPOLOGIE ÉTOILE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Formule: Bandwidth_total = Write_throughput × N_replicas       │
│                                                                 │
│  Exemple:                                                       │
│  • Write throughput: 100 MB/s                                   │
│  • Nombre de replicas: 3                                        │
│  • Bandwidth nécessaire: 100 × 3 = 300 MB/s                     │
│                                                                 │
│  Network saturation check:                                      │
│  ┌─────────────────────────────────────────────┐                │
│  │ Master                                      │                │
│  │                                             │                │
│  │  NIC: 1 Gbps (125 MB/s)                     │                │
│  │                                             │                │
│  │  Traffic:                                   │                │
│  │  ├─ Client writes: 100 MB/s                 │                │
│  │  ├─ Replica 1: 100 MB/s                     │                │
│  │  ├─ Replica 2: 100 MB/s                     │                │
│  │  └─ Replica 3: 100 MB/s                     │                │
│  │                                             │                │
│  │  Total: 400 MB/s  ❌ SATURATION!            │                │
│  └─────────────────────────────────────────────┘                │
│                                                                 │
│  Solution:                                                      │
│  • Upgrade NIC à 10 Gbps                                        │
│  • Réduire nombre de replicas                                   │
│  • Utiliser topologie en cascade                                │
│  • Compression (Redis Enterprise)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Avantages et inconvénients

**✅ Avantages** :

1. **Simplicité opérationnelle** : Configuration et maintenance aisées
2. **Latence uniforme** : Tous les replicas ont la même latence depuis master
3. **Promotion rapide** : N'importe quel replica peut devenir master
4. **Debugging facile** : Flux de données linéaire et prévisible

**❌ Inconvénients** :

1. **Scalabilité limitée** : Bande passante master = goulot d'étranglement
2. **SPOF** : Master est un point de défaillance unique
3. **Coût réseau** : N replicas = N × bande passante
4. **Pas de géo-distribution optimale** : Tous les replicas subissent la même latence WAN

### Cas d'usage recommandés

```yaml
Scénarios adaptés:
  - Nombre de replicas: 2-5 replicas maximum
  - Même datacenter / région: Latence <10ms
  - Write throughput: <50 MB/s
  - Read scaling: Modéré (quelques replicas suffisent)

Exemples:
  - Application web standard avec 2-3 replicas
  - Service API avec load balancing lecture
  - Cache distribué avec HA basique
```

---

## 🔗 Topologie 2 : Chaînée (Cascading / Chain)

### Architecture

La topologie chaînée organise les replicas en plusieurs niveaux hiérarchiques, réduisant la charge sur le master.

```
┌─────────────────────────────────────────────────────────────────┐
│  TOPOLOGIE CHAÎNÉE (CASCADING)                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Level 0 (Master):                                              │
│                                                                 │
│                     ┌────────────────┐                          │
│                     │   MASTER (M)   │                          │
│                     │   10.0.1.10    │                          │
│                     │   Write: 100%  │                          │
│                     └────────┬───────┘                          │
│                              │                                  │
│                              │ Bandwidth: 100 MB/s              │
│                              │                                  │
│  Level 1 (Primary Replicas): │                                  │
│                              │                                  │
│              ┌───────────────┴───────────────┐                  │
│              │                               │                  │
│              ▼                               ▼                  │
│        ┌──────────┐                    ┌──────────┐             │
│        │REPLICA L1│                    │REPLICA L1│             │
│        │   (R1)   │                    │   (R2)   │             │
│        │10.0.1.11 │                    │10.0.1.12 │             │
│        │Zone A    │                    │Zone B    │             │
│        └────┬─────┘                    └────┬─────┘             │
│             │                               │                   │
│             │ Bandwidth: 100 MB/s           │ Bandwidth: 100MB  │
│             │                               │                   │
│  Level 2:   │                               │                   │
│             │                               │                   │
│      ┌──────┴──────┐                 ┌──────┴──────┐            │
│      │             │                 │             │            │
│      ▼             ▼                 ▼             ▼            │
│  ┌───────┐    ┌───────┐         ┌───────┐    ┌───────┐          │
│  │R L2-A1│    │R L2-A2│         │R L2-B1│    │R L2-B2│          │
│  │.13    │    │.14    │         │.15    │    │.16    │          │
│  │Zone A │    │Zone A'│         │Zone B │    │Zone B'│          │
│  └───────┘    └───────┘         └───────┘    └───────┘          │
│                                                                 │
│  Caractéristiques:                                              │
│  • Master bandwidth: Réduit de 50% (2 replicas vs 6)            │
│  • Latence cumulée: L0→L1 (~5ms) + L1→L2 (~5ms) = ~10ms         │
│  • Scalabilité: Meilleure qu'étoile (ajout Level 3 possible)    │
│  • Complexité: Plus élevée (gestion multi-niveaux)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

**Master (Level 0)** :

```ini
# redis-master-cascade.conf

################################# NETWORK ####################################
bind 0.0.0.0
port 6379
protected-mode yes
requirepass "MasterPass2024!"
masterauth "MasterPass2024!"

################################ REPLICATION #################################
# Seulement 2 replicas directs (L1)
repl-diskless-sync yes
repl-diskless-sync-delay 5

# Backlog standard (charge réduite)
repl-backlog-size 128mb
repl-backlog-ttl 3600

# Protection: Minimum 1 replica L1
min-replicas-to-write 1
min-replicas-max-lag 10

# Output buffer adapté (seulement 2 connexions)
client-output-buffer-limit replica 512mb 128mb 60

################################ PERSISTENCE #################################
appendonly yes
appendfsync everysec

################################# MEMORY #####################################
maxmemory 8gb
maxmemory-policy allkeys-lru

################################### LOGS #####################################
loglevel notice
logfile "/var/log/redis/master-cascade.log"
```

**Replica Level 1 (R1, R2)** :

```ini
# redis-replica-l1.conf

################################# NETWORK ####################################
bind 0.0.0.0
port 6379
protected-mode yes
requirepass "ReplicaL1Pass2024!"
masterauth "MasterPass2024!"

################################ REPLICATION #################################
# Connexion au Master
replicaof 10.0.1.10 6379

replica-read-only yes
replica-priority 90  # Plus prioritaire que L2 pour promotion

# IMPORTANT: Accepter connexions de replicas L2
# (pas de config spéciale, juste s'assurer que bind 0.0.0.0)

repl-timeout 60
repl-ping-replica-period 10

# Servir données même si master down (pour L2)
replica-serve-stale-data yes

################################ PERSISTENCE #################################
appendonly yes
appendfsync everysec

################################# MEMORY #####################################
maxmemory 8gb
maxmemory-policy allkeys-lru

# Output buffer pour replicas L2 connectés
client-output-buffer-limit replica 512mb 128mb 60

################################### LOGS #####################################
loglevel notice
logfile "/var/log/redis/replica-l1-r1.log"

# NOTE CRITIQUE:
# Ce replica L1 agit à la fois comme:
# - Replica du Master (L0)
# - Master pour replicas L2
# Monitoring: Vérifier connected_slaves sur cette instance
```

**Replica Level 2 (R L2-A1, R L2-A2, etc.)** :

```ini
# redis-replica-l2.conf

################################# NETWORK ####################################
bind 0.0.0.0
port 6379
protected-mode yes
requirepass "ReplicaL2Pass2024!"
masterauth "ReplicaL1Pass2024!"  # Password du replica L1!

################################ REPLICATION #################################
# IMPORTANT: Pointer vers Replica L1, pas Master!
replicaof 10.0.1.11 6379  # IP du Replica L1 (R1)

replica-read-only yes
replica-priority 80  # Moins prioritaire que L1 pour promotion

repl-timeout 90  # Plus long (cumul latence L0→L1→L2)
repl-ping-replica-period 10

replica-serve-stale-data yes

################################ PERSISTENCE #################################
appendonly yes
appendfsync everysec

################################# MEMORY #####################################
maxmemory 8gb
maxmemory-policy allkeys-lru

################################### LOGS #####################################
loglevel notice
logfile "/var/log/redis/replica-l2-a1.log"

# ATTENTION:
# - Latence cumulée: lag(M→L1) + lag(L1→L2)
# - Si L1 down, L2 devient orphelin
# - Monitoring: Vérifier master_link_status ET upstream health
```

### Calcul de latence cumulée

```
┌─────────────────────────────────────────────────────────────────┐
│  LATENCE CUMULÉE - TOPOLOGIE CHAÎNÉE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Latence_L2 = Latence(M→L1) + Latence(L1→L2)                    │
│                                                                 │
│  Scénario 1: Même datacenter                                    │
│  ┌───────────────────────────────────────┐                      │
│  │ M → L1: 2ms (Same AZ)                 │                      │
│  │ L1 → L2: 3ms (Different AZ)           │                      │
│  │ Total L2: 5ms                         │  ✅ Acceptable       │
│  └───────────────────────────────────────┘                      │
│                                                                 │
│  Scénario 2: Multi-région                                       │
│  ┌───────────────────────────────────────┐                      │
│  │ M (Paris) → L1 (Frankfurt): 20ms      │                      │
│  │ L1 (Frankfurt) → L2 (London): 15ms    │                      │
│  │ Total L2: 35ms                        │  ⚠️  Élevé           │
│  └───────────────────────────────────────┘                      │
│                                                                 │
│  Scénario 3: Multi-niveau profond (3 niveaux)                   │
│  ┌───────────────────────────────────────┐                      │
│  │ M → L1: 5ms                           │                      │
│  │ L1 → L2: 5ms                          │                      │
│  │ L2 → L3: 5ms                          │                      │
│  │ Total L3: 15ms                        │  ⚠️  Limite          │
│  └───────────────────────────────────────┘                      │
│                                                                 │
│  Recommandation:                                                │
│  • Max 2 niveaux de cascade (L1, L2)                            │
│  • Total latency < 50ms                                         │
│  • Au-delà: considérer topologie étoile ou cluster              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Gestion des pannes en cascade

```
┌─────────────────────────────────────────────────────────────────┐
│  SCÉNARIOS DE PANNE - TOPOLOGIE CHAÎNÉE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Scénario 1: Panne Master (M)                                   │
│  ──────────────────────────────────────────                     │
│                                                                 │
│      ┌─────┐                                                    │
│      │  M  │ ✗ DOWN                                             │
│      └─────┘                                                    │
│         │                                                       │
│    ┌────┴────┐                                                  │
│    ▼         ▼                                                  │
│  ┌───┐     ┌───┐                                                │
│  │ L1│     │ L1│  ← Sentinel promeut L1 comme master            │
│  │ R1│     │ R2│                                                │
│  └─┬─┘     └─┬─┘                                                │
│    │         │                                                  │
│    ▼         ▼                                                  │
│   L2 OK    L2 OK  ← L2 restent connectés                        │
│                                                                 │
│  Impact: Minimal (failover standard)                            │
│  Action: Sentinel gère automatiquement                          │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Scénario 2: Panne Replica L1 (R1)                              │
│  ──────────────────────────────────────────                     │
│                                                                 │
│      ┌─────┐                                                    │
│      │  M  │ OK                                                 │
│      └──┬──┘                                                    │
│         │                                                       │
│    ┌────┴────┐                                                  │
│    │         ▼                                                  │
│  ┌───┐     ┌───┐                                                │
│  │ L1│✗    │ L1│ OK                                             │
│  │ R1│     │ R2│                                                │
│  └─┬─┘     └─┬─┘                                                │
│    │         │                                                  │
│    ▼         ▼                                                  │
│ L2 (R1)    L2 (R2) OK                                           │
│ ORPHELINS!                                                      │
│                                                                 │
│  Impact: Replicas L2 de R1 deviennent orphelins                 │
│  Action:                                                        │
│  1. Sentinel détecte panne R1                                   │
│  2. Replicas L2 de R1 doivent être reconfigurés:                │
│     redis-cli -h L2-A1 REPLICAOF 10.0.1.12 6379  (pointer R2)   │
│  3. Ou attendre que R1 revienne                                 │
│                                                                 │
│  ⚠️  Automation recommandée (script monitoring)                 │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Scénario 3: Partition réseau M↔L1                              │
│  ──────────────────────────────────────────                     │
│                                                                 │
│      ┌─────┐                                                    │
│      │  M  │ Isolated                                           │
│      └─────┘                                                    │
│         ✗ Network partition                                     │
│    ┌────┴────┐                                                  │
│    │         │                                                  │
│  ┌───┐     ┌───┐                                                │
│  │ L1│     │ L1│  ← Peuvent encore communiquer entre eux        │
│  │ R1│◄───▶│ R2│                                                │
│  └─┬─┘     └─┬─┘                                                │
│    │         │                                                  │
│    ▼         ▼                                                  │
│   L2        L2  ← Continuent à répliquer depuis L1              │
│                                                                 │
│  Impact: Split-brain possible                                   │
│  Protection: min-replicas-to-write sur Master                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Script d'auto-healing pour replicas L2 orphelins** :

```bash
#!/bin/bash
# auto-heal-l2-replicas.sh
# À exécuter sur un serveur de monitoring

MASTER="10.0.1.10"
L1_R1="10.0.1.11"
L1_R2="10.0.1.12"
L2_REPLICAS=("10.0.1.13" "10.0.1.14" "10.0.1.15" "10.0.1.16")

# Vérifier santé L1
check_l1_health() {
    local l1_host=$1
    redis-cli -h $l1_host PING >/dev/null 2>&1
    return $?
}

# Vérifier connexion d'un L2
check_l2_connection() {
    local l2_host=$1
    local status=$(redis-cli -h $l2_host INFO replication | grep master_link_status | cut -d: -f2 | tr -d '\r\n')
    [[ "$status" == "up" ]]
    return $?
}

# Reconfigurer un L2 vers un nouveau L1
repoint_l2() {
    local l2_host=$1
    local new_l1=$2
    echo "Reconfiguring $l2_host to point to $new_l1"
    redis-cli -h $l2_host REPLICAOF $new_l1 6379
}

# Boucle de monitoring
while true; do
    # Check L1 health
    L1_R1_UP=$(check_l1_health $L1_R1 && echo "yes" || echo "no")
    L1_R2_UP=$(check_l1_health $L1_R2 && echo "yes" || echo "no")

    echo "$(date): L1-R1=$L1_R1_UP, L1-R2=$L1_R2_UP"

    # Si L1-R1 down, pointer ses L2 vers L1-R2
    if [[ "$L1_R1_UP" == "no" && "$L1_R2_UP" == "yes" ]]; then
        for l2 in "${L2_REPLICAS[@]:0:2}"; do  # Premier 2 L2 appartiennent à R1
            if ! check_l2_connection $l2; then
                echo "⚠️  $l2 disconnected, repointing to $L1_R2"
                repoint_l2 $l2 $L1_R2
            fi
        done
    fi

    # Si L1-R2 down, pointer ses L2 vers L1-R1
    if [[ "$L1_R2_UP" == "no" && "$L1_R1_UP" == "yes" ]]; then
        for l2 in "${L2_REPLICAS[@]:2:2}"; do  # Dernier 2 L2 appartiennent à R2
            if ! check_l2_connection $l2; then
                echo "⚠️  $l2 disconnected, repointing to $L1_R1"
                repoint_l2 $l2 $L1_R1
            fi
        done
    fi

    sleep 10
done
```

### Avantages et inconvénients

**✅ Avantages** :

1. **Scalabilité améliorée** : Bande passante master réduite (N/2 si 2 branches)
2. **Répartition de charge** : Replicas L1 distribuent la charge
3. **Géo-distribution** : L1 dans régions principales, L2 dans zones secondaires
4. **Coût réseau optimisé** : Réplication locale dans chaque région

**❌ Inconvénients** :

1. **Latence cumulée** : L2+ subissent latence additionnelle
2. **Complexité accrue** : Configuration et monitoring plus complexes
3. **Orphelins potentiels** : L2 deviennent orphelins si L1 down
4. **Promotion compliquée** : Hiérarchie doit être reconfigurée lors failover

### Cas d'usage recommandés

```yaml
Scénarios adaptés:
  - Nombre de replicas: >5 replicas
  - Géo-distribution: Multi-régions avec réplication locale
  - Write throughput élevé: >50 MB/s
  - Bande passante limitée: Optimisation des coûts réseau

Exemples:
  - Application globale avec L1 par continent
  - CDN-like architecture avec edge replicas (L2)
  - Multi-datacenter avec hiérarchie régionale

À éviter:
  - Applications nécessitant latence uniforme
  - Petites déploiements (<5 replicas)
  - Équipe sans expertise DevOps avancée
```

---

## 🌐 Topologie 3 : Hybride (Étoile + Chaîne)

### Architecture

Combinaison des deux topologies pour optimiser différents cas d'usage.

```
┌────────────────────────────────────────────────────────────────┐
│  TOPOLOGIE HYBRIDE                                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│                     ┌────────────────┐                         │
│                     │   MASTER (M)   │                         │
│                     │   Paris        │                         │
│                     │   10.0.1.10    │                         │
│                     └────────┬───────┘                         │
│                              │                                 │
│          ┌───────────────────┼───────────────────┐             │
│          │                   │                   │             │
│          │                   │                   │             │
│  Étoile  │            Étoile │            Cascade│             │
│  (local) │           (backup)│          (distant)│             │
│          │                   │                   │             │
│          ▼                   ▼                   ▼             │
│    ┌──────────┐        ┌──────────┐        ┌──────────┐        │
│    │REPLICA L1│        │REPLICA L1│        │REPLICA L1│        │
│    │  (Local) │        │ (Backup) │        │(Regional)│        │
│    │  Paris   │        │  Paris   │        │Frankfurt │        │
│    │ 10.0.1.11│        │ 10.0.1.12│        │10.0.2.10 │        │
│    └──────────┘        └──────────┘        └────┬─────┘        │
│         │                    │                  │              │
│         │                    │                  │ Cascade      │
│    Read traffic         Read traffic            │ (local)      │
│     (Local)             (Backup)                │              │
│                                          ┌──────┴──────┐       │
│                                          │             │       │
│                                          ▼             ▼       │
│                                    ┌──────────┐ ┌──────────┐   │
│                                    │REPLICA L2│ │REPLICA L2│   │
│                                    │Frankfurt │ │Frankfurt │   │
│                                    │10.0.2.11 │ │10.0.2.12 │   │
│                                    └──────────┘ └──────────┘   │
│                                          │             │       │
│                                          └──────┬──────┘       │
│                                                 │              │
│                                          Read traffic          │
│                                         (Regional)             │
│                                                                │
│  Caractéristiques:                                             │
│  • Paris: 2 replicas étoile (latence minimale)                 │
│  • Frankfurt: 1 L1 + 2 L2 cascade (optimisation bandwidth)     │
│  • Best of both worlds                                         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Configuration

La configuration hybride utilise les mêmes fichiers que précédemment, avec une planification réseau appropriée :

**Planification des connexions** :

```yaml
# topology-map.yaml

master:
  host: 10.0.1.10
  location: Paris
  role: master
  direct_replicas:
    - 10.0.1.11  # L1 Local Paris (étoile)
    - 10.0.1.12  # L1 Backup Paris (étoile)
    - 10.0.2.10  # L1 Regional Frankfurt (tête de cascade)

replicas_l1:
  - host: 10.0.1.11
    location: Paris
    type: star  # Connexion directe au master
    purpose: Local read traffic

  - host: 10.0.1.12
    location: Paris
    type: star
    purpose: Backup / DR

  - host: 10.0.2.10
    location: Frankfurt
    type: cascade_head  # Tête de cascade pour région
    purpose: Regional distribution
    sub_replicas:
      - 10.0.2.11  # L2 Frankfurt 1
      - 10.0.2.12  # L2 Frankfurt 2

replicas_l2:
  - host: 10.0.2.11
    location: Frankfurt
    master: 10.0.2.10  # Pointe vers L1 régional

  - host: 10.0.2.12
    location: Frankfurt
    master: 10.0.2.10
```

### Cas d'usage recommandés

```yaml
Scénarios adaptés:
  - Multi-région avec asymétrie:
      • Région principale: Étoile (latence minimale)
      • Régions secondaires: Cascade (économie bandwidth)

  - Besoins mixtes:
      • Production: Étoile pour performance
      • Analytics: Cascade pour isolation
      • DR: Étoile pour failover rapide

  - Optimisation coûts:
      • Replicas locaux: Étoile (faible coût réseau)
      • Replicas WAN: Cascade (réduction 50% bandwidth)

Exemples:
  - E-commerce global: EU (étoile) + US/ASIA (cascade)
  - SaaS multi-tenant: Tenant principal (étoile) + autres (cascade)
  - Media streaming: CDN architecture hybride
```

---

## 🌍 Topologie 4 : Géo-distribuée

### Architecture

Optimisation pour applications globales avec latence locale.

```
┌─────────────────────────────────────────────────────────────────┐
│  TOPOLOGIE GÉO-DISTRIBUÉE                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    MASTER (Active)                         │ │
│  │                      EU-WEST-1                             │ │
│  │                      Paris                                 │ │
│  │                      10.1.0.10                             │ │
│  └──────────────────────┬─────────────────────────────────────┘ │
│                         │                                       │
│                         │ WAN Replication                       │
│         ┌───────────────┼───────────────┐                       │
│         │               │               │                       │
│  ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼─────┐                 │
│  │   EU-WEST   │ │  US-EAST    │ │  ASIA-PAC  │                 │
│  │  (Regional) │ │ (Regional)  │ │ (Regional) │                 │
│  └─────────────┘ └─────────────┘ └────────────┘                 │
│                                                                 │
│  Région 1: EU-WEST                                              │
│  ┌──────────────────────────────────────────┐                   │
│  │  Master: 10.1.0.10 (Paris)               │                   │
│  │    ↓                                     │                   │
│  │  Replica L1: 10.1.0.11 (Paris AZ-B)      │ ~2ms              │
│  │  Replica L1: 10.1.0.12 (London)          │ ~10ms             │
│  │    ↓                                     │                   │
│  │  Replica L2: 10.1.0.13 (Amsterdam)       │ ~5ms              │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  Région 2: US-EAST                                              │
│  ┌──────────────────────────────────────────┐                   │
│  │  Replica L1: 10.2.0.10 (Virginia)        │ ~80ms from M      │
│  │    ↓                                     │                   │
│  │  Replica L2: 10.2.0.11 (Virginia AZ-B)   │ ~2ms from L1      │
│  │  Replica L2: 10.2.0.12 (Ohio)            │ ~5ms from L1      │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  Région 3: ASIA-PAC                                             │
│  ┌──────────────────────────────────────────┐                   │
│  │  Replica L1: 10.3.0.10 (Singapore)       │ ~150ms from M     │
│  │    ↓                                     │                   │
│  │  Replica L2: 10.3.0.11 (Tokyo)           │ ~60ms from L1     │
│  │  Replica L2: 10.3.0.12 (Sydney)          │ ~80ms from L1     │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  Caractéristiques:                                              │
│  • Read locality: <10ms dans chaque région                      │
│  • Write latency: Variable selon région client                  │
│  • Failover régional possible (L1 → Master)                     │
│  • Bandwidth optimisé (1 connexion WAN par région)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration avancée avec DNS

**Stratégie de connexion client** :

```yaml
# dns-routing.yaml

# GeoDNS ou Latency-based routing (AWS Route53, Cloudflare)

endpoints:
  # Endpoint global (write-through)
  write:
    domain: redis-master.global.company.com
    target: 10.1.0.10  # Master Paris

  # Endpoints régionaux (read-local)
  read:
    eu-west:
      domain: redis-read.eu.company.com
      targets:
        - 10.1.0.11  # Paris AZ-B
        - 10.1.0.12  # London
      routing: round-robin

    us-east:
      domain: redis-read.us.company.com
      targets:
        - 10.2.0.11  # Virginia AZ-B
        - 10.2.0.12  # Ohio
      routing: round-robin

    asia-pac:
      domain: redis-read.asia.company.com
      targets:
        - 10.3.0.11  # Tokyo
        - 10.3.0.12  # Sydney
      routing: round-robin

# Application configuration
client_config:
  write_endpoint: redis-master.global.company.com:6379
  read_endpoint: redis-read.${AWS_REGION}.company.com:6379
  # Automatiquement résolu selon la région du client
```

**Code application (exemple Node.js)** :

```javascript
// redis-geo-client.js

const Redis = require('ioredis');

// Configuration géo-distribuée
const REGION = process.env.AWS_REGION || 'eu-west';
const WRITE_ENDPOINT = 'redis-master.global.company.com';
const READ_ENDPOINT = `redis-read.${REGION}.company.com`;

// Client pour writes (toujours vers master)
const writeClient = new Redis({
  host: WRITE_ENDPOINT,
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (times) => Math.min(times * 50, 2000),
});

// Client pour reads (local à la région)
const readClient = new Redis({
  host: READ_ENDPOINT,
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (times) => Math.min(times * 50, 2000),
});

// Helper functions
async function write(key, value, ttl = null) {
  if (ttl) {
    await writeClient.setex(key, ttl, value);
  } else {
    await writeClient.set(key, value);
  }
}

async function read(key) {
  return await readClient.get(key);
}

// Usage
(async () => {
  // Write to master (Paris)
  await write('user:1000', JSON.stringify({ name: 'Alice' }), 3600);

  // Read from local replica (fast!)
  const userData = await read('user:1000');
  console.log('User data:', userData);

  // Latence typique:
  // EU: read ~2ms, write ~5ms
  // US: read ~2ms, write ~85ms (WAN to Paris)
  // ASIA: read ~5ms, write ~155ms (WAN to Paris)
})();
```

### Monitoring géo-distribué

```bash
#!/bin/bash
# monitor-geo-replication.sh

MASTER="10.1.0.10"
declare -A REGIONS=(
  ["eu"]="10.1.0.11 10.1.0.12 10.1.0.13"
  ["us"]="10.2.0.10 10.2.0.11 10.2.0.12"
  ["asia"]="10.3.0.10 10.3.0.11 10.3.0.12"
)

echo "=== Geo-Distributed Replication Status ==="
echo "Master: $MASTER"
echo ""

# Get master offset
MASTER_OFFSET=$(redis-cli -h $MASTER INFO replication | grep master_repl_offset | cut -d: -f2 | tr -d '\r')
echo "Master offset: $MASTER_OFFSET"
echo ""

# Check each region
for region in "${!REGIONS[@]}"; do
  echo "Region: ${region^^}"
  echo "─────────────────────────────────"

  for replica in ${REGIONS[$region]}; do
    # Get replica info
    ROLE=$(redis-cli -h $replica INFO replication 2>/dev/null | grep "role:" | cut -d: -f2 | tr -d '\r')

    if [[ "$ROLE" == "slave" ]]; then
      OFFSET=$(redis-cli -h $replica INFO replication | grep master_repl_offset | cut -d: -f2 | tr -d '\r')
      LINK_STATUS=$(redis-cli -h $replica INFO replication | grep master_link_status | cut -d: -f2 | tr -d '\r')
      LAG_BYTES=$((MASTER_OFFSET - OFFSET))

      printf "  %-15s: offset=%d lag=%d bytes status=%s\n" $replica $OFFSET $LAG_BYTES $LINK_STATUS

      # Alert if lag > 10MB
      if [ $LAG_BYTES -gt 10485760 ]; then
        echo "    ⚠️  WARNING: High lag!"
      fi
    else
      echo "  $replica: UNREACHABLE or WRONG ROLE"
    fi
  done
  echo ""
done
```

---

## 📊 Matrice comparative des topologies

| Critère | Étoile | Chaînée | Hybride | Géo-distribuée |
|---------|--------|---------|---------|----------------|
| **Complexité** | ⭐ Simple | ⭐⭐⭐ Élevée | ⭐⭐⭐ Élevée | ⭐⭐⭐⭐ Très élevée |
| **Latence max** | 5-10ms | 10-50ms | 5-50ms | 5-200ms |
| **Scalabilité reads** | Moyenne | Élevée | Élevée | Très élevée |
| **Bandwidth master** | Élevé (N×) | Faible (2×) | Moyen | Faible (3×) |
| **Résilience panne** | Bonne | Moyenne | Bonne | Excellente |
| **Coût réseau** | Moyen-Élevé | Faible | Moyen | Faible (WAN) |
| **Failover** | Simple | Complexe | Moyen | Complexe |
| **Nombre replicas** | 2-5 | 5-20+ | 5-15 | 10-50+ |
| **Cas d'usage** | Standard | High-scale | Production | Global |

---

## 🎯 Recommandations de déploiement

### Règles de décision

```yaml
Choisir ÉTOILE si:
  - Replicas: ≤ 5
  - Localisation: Même datacenter/région
  - Latence critique: <10ms uniform
  - Équipe: DevOps junior
  - Exemple: Startup, PME, application régionale

Choisir CHAÎNÉE si:
  - Replicas: > 5
  - Write throughput: Élevé (>50 MB/s)
  - Bandwidth: Limité ou coûteux
  - Latency tolerance: 10-50ms acceptable
  - Exemple: High-traffic app, analytics replicas

Choisir HYBRIDE si:
  - Besoins mixtes: Performance + Scale
  - Multi-objectifs: Production + DR + Analytics
  - Budget: Moyen-élevé
  - Équipe: DevOps expérimenté
  - Exemple: E-commerce mature, SaaS B2B

Choisir GÉO-DISTRIBUÉE si:
  - Global: Multi-continents
  - Read locality: Critique (<10ms local)
  - Write latency: Toléré (50-200ms acceptable)
  - Budget: Élevé
  - Équipe: SRE team
  - Exemple: CDN, social media, global SaaS
```

### Checklist de déploiement

**Avant déploiement** :

- [ ] **Topologie documentée** : Schéma avec IPs, ports, rôles
- [ ] **Calcul latence** : Mesuré entre chaque hop
- [ ] **Calcul bandwidth** : Dimensionné selon throughput
- [ ] **Monitoring configuré** : Métriques par niveau
- [ ] **Runbook créé** : Procédures failover et healing
- [ ] **Tests réalisés** : Simulation pannes sur chaque niveau
- [ ] **DNS/Load balancing** : Configuré pour read routing
- [ ] **Alerting** : Seuils définis (lag, disconnect)
- [ ] **Backup strategy** : RDB/AOF sur replicas appropriés
- [ ] **Disaster Recovery** : Plan de restauration documenté

---

## 🔧 Outils de gestion de topologie

### Outil 1 : Visualisation automatique

```python
#!/usr/bin/env python3
# redis-topology-visualizer.py

import redis
import graphviz
from typing import Dict, List

def get_replication_info(host: str, port: int = 6379, password: str = None) -> Dict:
    """Récupère les infos de réplication d'une instance"""
    r = redis.Redis(host=host, port=port, password=password, decode_responses=True)
    info = r.info('replication')
    return info

def build_topology_graph(master_host: str, password: str = None) -> graphviz.Digraph:
    """Construit le graphe de la topologie"""
    dot = graphviz.Digraph(comment='Redis Topology')
    dot.attr(rankdir='TB')

    visited = set()

    def visit_node(host: str, level: int = 0, parent: str = None):
        if host in visited:
            return
        visited.add(host)

        try:
            info = get_replication_info(host, password=password)
            role = info['role']

            # Node styling
            if role == 'master':
                dot.node(host, f"MASTER\n{host}", shape='box', style='filled', fillcolor='lightblue')
            else:
                dot.node(host, f"REPLICA L{level}\n{host}", shape='ellipse', style='filled', fillcolor='lightgreen')

            # Edge from parent
            if parent:
                lag = info.get('master_last_io_seconds_ago', 'N/A')
                dot.edge(parent, host, label=f"{lag}s lag")

            # Visit slaves
            connected_slaves = info.get('connected_slaves', 0)
            for i in range(connected_slaves):
                slave_info = info[f'slave{i}']
                slave_ip = slave_info.split(',')[0].split('=')[1]
                visit_node(slave_ip, level + 1, host)

        except Exception as e:
            dot.node(host, f"ERROR\n{host}", shape='box', style='filled', fillcolor='red')
            print(f"Error visiting {host}: {e}")

    visit_node(master_host)
    return dot

# Usage
if __name__ == "__main__":
    import sys

    if len(sys.argv) < 2:
        print("Usage: python redis-topology-visualizer.py <master_ip> [password]")
        sys.exit(1)

    master_ip = sys.argv[1]
    password = sys.argv[2] if len(sys.argv) > 2 else None

    graph = build_topology_graph(master_ip, password)
    graph.render('redis-topology', view=True, format='png')
    print("Topology graph saved to redis-topology.png")
```

### Outil 2 : Validation de configuration

```python
#!/usr/bin/env python3
# validate-redis-topology.py

import redis
from typing import List, Tuple
import sys

class TopologyValidator:
    def __init__(self, nodes: List[Tuple[str, int, str]]):
        """
        nodes: List of (host, port, role) tuples
        role: 'master', 'l1', 'l2', etc.
        """
        self.nodes = nodes
        self.results = []

    def validate_connectivity(self):
        """Vérifie la connectivité de chaque nœud"""
        print("\n=== Connectivity Validation ===")
        for host, port, role in self.nodes:
            try:
                r = redis.Redis(host=host, port=port, socket_connect_timeout=5)
                r.ping()
                print(f"✅ {role:8s} {host}:{port} - REACHABLE")
            except Exception as e:
                print(f"❌ {role:8s} {host}:{port} - UNREACHABLE: {e}")
                self.results.append(('CONNECTIVITY', False, host, str(e)))

    def validate_replication(self):
        """Vérifie l'état de réplication"""
        print("\n=== Replication Validation ===")
        for host, port, role in self.nodes:
            try:
                r = redis.Redis(host=host, port=port, decode_responses=True)
                info = r.info('replication')

                actual_role = info['role']
                link_status = info.get('master_link_status', 'N/A')

                if role == 'master' and actual_role != 'master':
                    print(f"❌ {host}:{port} - Expected master, got {actual_role}")
                    self.results.append(('ROLE', False, host, f"Expected master, got {actual_role}"))
                elif role.startswith('l') and actual_role != 'slave':
                    print(f"❌ {host}:{port} - Expected slave, got {actual_role}")
                    self.results.append(('ROLE', False, host, f"Expected slave, got {actual_role}"))
                elif actual_role == 'slave' and link_status != 'up':
                    print(f"⚠️  {host}:{port} - Link status: {link_status}")
                    self.results.append(('LINK', False, host, f"Link status: {link_status}"))
                else:
                    print(f"✅ {role:8s} {host}:{port} - OK (role={actual_role}, link={link_status})")

            except Exception as e:
                print(f"❌ {host}:{port} - Error: {e}")
                self.results.append(('REPLICATION', False, host, str(e)))

    def validate_lag(self, max_lag: int = 10):
        """Vérifie le lag de réplication"""
        print(f"\n=== Lag Validation (max: {max_lag}s) ===")
        for host, port, role in self.nodes:
            if role == 'master':
                continue

            try:
                r = redis.Redis(host=host, port=port, decode_responses=True)
                info = r.info('replication')

                if info['role'] == 'slave':
                    lag = info.get('master_last_io_seconds_ago', 999)
                    if lag > max_lag:
                        print(f"⚠️  {host}:{port} - High lag: {lag}s")
                        self.results.append(('LAG', False, host, f"Lag: {lag}s"))
                    else:
                        print(f"✅ {host}:{port} - Lag: {lag}s")

            except Exception as e:
                print(f"❌ {host}:{port} - Error: {e}")

    def report(self):
        """Affiche le rapport final"""
        print("\n" + "="*60)
        print("VALIDATION REPORT")
        print("="*60)

        errors = [r for r in self.results if not r[1]]
        if not errors:
            print("✅ All validations passed!")
            return 0
        else:
            print(f"❌ {len(errors)} error(s) found:")
            for check_type, _, host, message in errors:
                print(f"  [{check_type}] {host}: {message}")
            return 1

# Usage example
if __name__ == "__main__":
    # Définir la topologie attendue
    topology = [
        ("10.0.1.10", 6379, "master"),
        ("10.0.1.11", 6379, "l1"),
        ("10.0.1.12", 6379, "l1"),
        ("10.0.1.13", 6379, "l2"),
        ("10.0.1.14", 6379, "l2"),
    ]

    validator = TopologyValidator(topology)
    validator.validate_connectivity()
    validator.validate_replication()
    validator.validate_lag(max_lag=10)

    exit_code = validator.report()
    sys.exit(exit_code)
```

---

## 🎓 Points clés à retenir

1. **Étoile = Simplicité** : Default choice pour <5 replicas, même région
2. **Chaîne = Scalabilité** : Optimal pour >5 replicas, réduction bandwidth
3. **Hybride = Flexibilité** : Best of both worlds, production réelle
4. **Géo = Global** : Lecture locale, écriture centrale, complexe
5. **Latence cumulée** : Chaque niveau ajoute 5-20ms
6. **Bandwidth master** : Étoile coûte N×, chaîne coûte 2-3×
7. **Orphelins** : Surveiller et auto-heal replicas L2 si L1 down
8. **Testing essentiel** : Simuler pannes à chaque niveau
9. **Documentation** : Schémas et runbooks impératifs
10. **Monitoring granulaire** : Métriques par niveau et par région

---

## 🔗 Références

- [Redis Replication Documentation](https://redis.io/docs/management/replication/)
- [Architecting for Scale (Redis Patterns)](https://redis.io/topics/patterns)
- [Multi-DC Replication Best Practices](https://redis.com/redis-best-practices/communication-patterns/multi-dc-replication/)

---

**Section suivante** : [10.3 Redis Sentinel : Monitoring et Failover automatique](./03-redis-sentinel-monitoring-failover.md)

⏭️ [Redis Sentinel : Monitoring et Failover automatique](/10-architecture-haute-disponibilite/03-redis-sentinel-monitoring-failover.md)
