🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.5 Gestion des nœuds (ajout, suppression, resharding)

## Introduction

La gestion dynamique des nœuds constitue l'une des fonctionnalités clés de Redis Cluster, permettant l'adaptation du cluster aux besoins évolutifs de capacité et de performance. Contrairement aux systèmes distribués traditionnels nécessitant un arrêt complet pour modification de topologie, Redis Cluster permet d'ajouter, de supprimer et de redistribuer les données entre nœuds sans interruption de service.

Cette section détaille les opérations de gestion du cycle de vie des nœuds, du resharding stratégique, et des procédures de maintenance permettant d'assurer la disponibilité continue du cluster tout en optimisant la distribution des données.

## Concepts préliminaires

### État d'un nœud dans le cluster

```
┌─────────────────────────────────────────────────────────────┐
│                Cycle de vie d'un nœud                       │
└─────────────────────────────────────────────────────────────┘

    ┌─────────────┐
    │   ISOLÉ     │  Nœud démarré mais pas dans le cluster
    │ (isolated)  │  cluster_known_nodes: 1 (lui-même)
    └──────┬──────┘
           │
           │ CLUSTER MEET <ip> <port>
           │
           ▼
    ┌─────────────┐
    │   MEMBRE    │  Nœud intégré, mais sans slot
    │  (member)   │  cluster_known_nodes: N
    └──────┬──────┘  cluster_slots_assigned: 0
           │
           │ Assignation de slots (ADDSLOTS/SETSLOT)
           │ ou Failover (si replica)
           │
           ▼
    ┌─────────────┐
    │   ACTIF     │  Master avec slots ou Replica fonctionnelle
    │  (active)   │  Sert des requêtes
    └──────┬──────┘
           │
           │ Détection de panne ou CLUSTER FORGET
           │
           ▼
    ┌─────────────┐
    │    FAIL     │  Nœud détecté comme en panne
    │   (failed)  │  Plus de communication possible
    └──────┬──────┘
           │
           │ Nœud redémarre ou est retiré
           │
           ▼
    [Retour à ISOLÉ ou SUPPRIMÉ]
```

### Rôles des nœuds

```
MASTER (Maître)
═══════════════
• Possède des hash slots (portion du keyspace)
• Accepte les lectures et écritures
• Réplique vers ses replicas
• Peut participer au vote lors d'élections
• Doit avoir au moins 1 replica pour HA

REPLICA (Esclave)
═════════════════
• Ne possède pas de slots directement
• Réplique de façon asynchrone un master
• Lecture seule (sauf si cluster-replica-no-failover no)
• Peut devenir master lors d'un failover
• Participe au protocole Gossip

ORPHAN MASTER (Maître orphelin)
════════════════════════════════
• Master sans replica
• Point de défaillance unique pour ses slots
• État à éviter en production
• Redis peut migrer des replicas automatiquement

ORPHAN REPLICA (Replica orpheline)
═══════════════════════════════════
• Replica dont le master est tombé définitivement
• Ne sert à rien tant qu'elle n'est pas promue ou réassignée
• À réassigner manuellement à un nouveau master
```

## Ajout de nœuds au cluster

### Ajout d'un master (avec redistribution de slots)

```bash
# ═══════════════════════════════════════════════════════════
# PROCÉDURE COMPLÈTE : AJOUT D'UN MASTER
# ═══════════════════════════════════════════════════════════

# CONTEXTE INITIAL
# ────────────────
# Cluster existant : 3 masters (A, B, C) + 3 replicas
# Objectif : Ajouter un 4ème master (D) et redistribuer les slots

# Topologie initiale :
# Master A: slots 0-5460     (33.3%)
# Master B: slots 5461-10922 (33.3%)
# Master C: slots 10923-16383 (33.4%)
# Total: 16384 slots

# Topologie cible :
# Master A: slots 0-4095     (25%)
# Master B: slots 4096-8191  (25%)
# Master C: slots 8192-12287 (25%)
# Master D: slots 12288-16383 (25%) ← Nouveau
# Total: 16384 slots


# ÉTAPE 1 : Préparer le nouveau nœud D
# ─────────────────────────────────────

# Sur le serveur du nœud D (192.168.1.16)
ssh 192.168.1.16

# Installer et configurer Redis (si pas déjà fait)
sudo apt install redis-server

# Configuration minimale
cat <<EOF | sudo tee /etc/redis/redis.conf
port 6379
bind 192.168.1.16
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 15000
appendonly yes
dir /var/lib/redis
EOF

# Démarrer Redis
sudo systemctl restart redis
sudo systemctl enable redis

# Vérifier que le nœud est isolé
redis-cli -h 192.168.1.16 CLUSTER INFO
# cluster_state:fail (normal, pas encore dans le cluster)
# cluster_known_nodes:1


# ÉTAPE 2 : Ajouter le nœud au cluster (MEET)
# ────────────────────────────────────────────

# Depuis n'importe quelle machine (ou un nœud existant)
redis-cli --cluster add-node 192.168.1.16:6379 192.168.1.10:6379

# Syntaxe : add-node <new-node> <existing-node>
# Le nouveau nœud contacte le nœud existant et rejoint le cluster

# Output attendu :
# >>> Adding node 192.168.1.16:6379 to cluster 192.168.1.10:6379
# >>> Performing Cluster Check
# [OK] All nodes agree about slots configuration.
# >>> Send CLUSTER MEET to node 192.168.1.16:6379
# [OK] New node added correctly.


# ÉTAPE 3 : Vérifier l'intégration
# ─────────────────────────────────

redis-cli -h 192.168.1.16 CLUSTER INFO
# cluster_state:ok (maintenant dans le cluster)
# cluster_known_nodes:4 (A, B, C, D)
# cluster_slots_assigned:16384 (tous assignés aux autres)

redis-cli -h 192.168.1.16 CLUSTER NODES
# Devrait lister les 4 nœuds
# Node D apparaît comme "master" mais avec 0 slots

redis-cli --cluster check 192.168.1.10:6379
# 192.168.1.10:6379 -> 5461 slots
# 192.168.1.11:6379 -> 5462 slots
# 192.168.1.12:6379 -> 5461 slots
# 192.168.1.16:6379 -> 0 slots      ← Nouveau nœud sans slots


# ÉTAPE 4 : Calculer la nouvelle distribution
# ────────────────────────────────────────────

# Objectif : Distribution équitable à 25% chacun (4096 slots)
#
# Transferts nécessaires :
# - De A (5461 slots) → A garde 4096, transfère 1365 à D
# - De B (5462 slots) → B garde 4096, transfère 1366 à D
# - De C (5461 slots) → C garde 4096, transfère 1365 à D
#
# Total vers D : 1365 + 1366 + 1365 = 4096 slots ✓


# ÉTAPE 5 : Resharding automatique
# ─────────────────────────────────

# Option A : Redistribution automatique équilibrée
redis-cli --cluster rebalance 192.168.1.10:6379 \
    --cluster-threshold 1 \
    --cluster-use-empty-masters

# Cette commande :
# 1. Calcule la distribution optimale
# 2. Identifie les transferts nécessaires
# 3. Exécute le resharding automatiquement
# 4. Équilibre pour que chaque nœud ait ~25% des slots

# Output :
# >>> Performing Cluster Check
# >>> Rebalancing across 4 nodes.
# Moving 1365 slots from 192.168.1.10:6379 to 192.168.1.16:6379
# Moving 1366 slots from 192.168.1.11:6379 to 192.168.1.16:6379
# Moving 1365 slots from 192.168.1.12:6379 to 192.168.1.16:6379


# Option B : Resharding manuel avec contrôle fin
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-from <node-a-id>,<node-b-id>,<node-c-id> \
    --cluster-to <node-d-id> \
    --cluster-slots 4096 \
    --cluster-yes \
    --cluster-pipeline 10

# Paramètres :
# --cluster-from : Liste des nœuds sources (séparés par virgule)
# --cluster-to : Nœud destination
# --cluster-slots : Nombre total de slots à transférer
# --cluster-yes : Pas de confirmation interactive
# --cluster-pipeline : Nombre de clés migrées en parallèle


# ÉTAPE 6 : Monitoring du resharding
# ───────────────────────────────────

# Dans un terminal séparé, surveiller la progression
watch -n 2 'redis-cli --cluster check 192.168.1.10:6379'

# Métriques à surveiller :
# - Nombre de slots par nœud
# - Nombre de clés en migration
# - État du cluster (cluster_state:ok)

# Vérifier les redirections pendant le resharding
redis-cli -h 192.168.1.10 -c GET some:key
# Peut retourner des -MOVED pendant la migration


# ÉTAPE 7 : Validation post-resharding
# ─────────────────────────────────────

# Vérifier la distribution finale
redis-cli --cluster check 192.168.1.10:6379

# Output attendu :
# 192.168.1.10:6379 -> 4096 slots (25%)
# 192.168.1.11:6379 -> 4096 slots (25%)
# 192.168.1.12:6379 -> 4096 slots (25%)
# 192.168.1.16:6379 -> 4096 slots (25%)
# [OK] All 16384 slots covered.

# Vérifier l'état du cluster
redis-cli -h 192.168.1.16 CLUSTER INFO | grep cluster_state
# cluster_state:ok

# Tester l'accès aux données
redis-cli -c -h 192.168.1.16 GET test:key
# Devrait fonctionner avec redirection automatique si nécessaire


# ÉTAPE 8 : Ajouter une replica pour le nouveau master
# ─────────────────────────────────────────────────────

# Préparer le nœud replica (192.168.1.17)
ssh 192.168.1.17
# [Configuration identique à l'étape 1]

# Ajouter comme replica du nouveau master D
redis-cli --cluster add-node 192.168.1.17:6379 192.168.1.10:6379 \
    --cluster-slave \
    --cluster-master-id <node-d-id>

# Récupérer le node-d-id :
NODE_D_ID=$(redis-cli -h 192.168.1.16 CLUSTER MYID)
echo $NODE_D_ID

redis-cli --cluster add-node 192.168.1.17:6379 192.168.1.10:6379 \
    --cluster-slave \
    --cluster-master-id $NODE_D_ID

# Output :
# >>> Adding node 192.168.1.17:6379 to cluster 192.168.1.10:6379
# >>> Performing Cluster Check
# >>> Check if node 192.168.1.16:6379 is a master...
# >>> Configure node as replica of 192.168.1.16:6379
# [OK] New node added correctly.


# ÉTAPE 9 : Validation finale
# ────────────────────────────

redis-cli --cluster check 192.168.1.10:6379

# Output final attendu :
# Master A (192.168.1.10) -> 4096 slots | 1 replica
# Master B (192.168.1.11) -> 4096 slots | 1 replica
# Master C (192.168.1.12) -> 4096 slots | 1 replica
# Master D (192.168.1.16) -> 4096 slots | 1 replica ← Nouveau
# [OK] All 16384 slots covered.
```

### Schéma du processus d'ajout

```
┌─────────────────────────────────────────────────────────────┐
│         Processus d'ajout d'un master avec slots            │
└─────────────────────────────────────────────────────────────┘

État initial (3 masters)
────────────────────────

    Master A        Master B        Master C
    [0-5460]       [5461-10922]   [10923-16383]
       │               │               │
    Replica A1     Replica B1      Replica C1


Étape 1 : Ajout du nœud D
──────────────────────────

    Master A        Master B        Master C      Master D
    [0-5460]       [5461-10922]   [10923-16383]    [VIDE]
       │               │               │              │
    Replica A1     Replica B1      Replica C1        X


Étape 2 : Resharding (migration de slots)
──────────────────────────────────────────

    Master A        Master B        Master C      Master D
    [0-5460]       [5461-10922]   [10923-16383]    [VIDE]
       │               │               │              │
       │               │               │              │
       └───────> 1365 slots ─────────────────────────>│
       │               └───────> 1366 slots ─────────>│
       │               │               └───> 1365 ───>│


Étape 3 : État après resharding
────────────────────────────────

    Master A        Master B        Master C      Master D
    [0-4095]       [4096-8191]    [8192-12287]  [12288-16383]
       │               │               │              │
    Replica A1     Replica B1      Replica C1     Replica D1


Distribution : 4 × 4096 = 16384 slots (25% chacun) ✓
```

### Ajout d'une replica (sans resharding)

```bash
# ═══════════════════════════════════════════════════════════
# AJOUT D'UNE REPLICA À UN MASTER EXISTANT
# ═══════════════════════════════════════════════════════════

# Contexte : Master sans replica ou ajout d'une 2ème replica
# Pas de resharding nécessaire (les replicas ne possèdent pas de slots)


# ÉTAPE 1 : Préparer le nœud replica
# ───────────────────────────────────

# Configuration identique aux masters (cluster-enabled yes)
ssh 192.168.1.20
sudo systemctl start redis


# ÉTAPE 2 : Identifier le master cible
# ─────────────────────────────────────

# Lister les masters
redis-cli -h 192.168.1.10 CLUSTER NODES | grep master

# Output :
# a1b2c3d4... 192.168.1.10:6379 master - 0 1234567890 1 connected 0-5460
# e5f6g7h8... 192.168.1.11:6379 master - 0 1234567891 2 connected 5461-10922
# ...

# Choisir le master (ex: 192.168.1.10)
# Récupérer son Node ID
MASTER_ID=$(redis-cli -h 192.168.1.10 CLUSTER MYID)
echo "Master ID: $MASTER_ID"


# ÉTAPE 3 : Ajouter la replica
# ─────────────────────────────

redis-cli --cluster add-node 192.168.1.20:6379 192.168.1.10:6379 \
    --cluster-slave \
    --cluster-master-id $MASTER_ID

# Alternative : Laisser Redis choisir automatiquement le master le moins répliqué
redis-cli --cluster add-node 192.168.1.20:6379 192.168.1.10:6379 \
    --cluster-slave
# Redis assignera la replica au master ayant le moins de replicas


# ÉTAPE 4 : Vérification
# ──────────────────────

# Vérifier le rôle
redis-cli -h 192.168.1.20 INFO replication
# role:slave
# master_host:192.168.1.10
# master_port:6379
# master_link_status:up

# Vérifier depuis le cluster
redis-cli -h 192.168.1.10 CLUSTER NODES | grep 192.168.1.20
# Devrait montrer : slave, master_id = $MASTER_ID

# Vérifier la réplication
redis-cli -h 192.168.1.10 INFO replication
# connected_slaves:2 (si c'était la 2ème replica)
```

## Suppression de nœuds du cluster

### Suppression d'une replica (simple)

```bash
# ═══════════════════════════════════════════════════════════
# SUPPRESSION D'UNE REPLICA (PAS DE RESHARDING)
# ═══════════════════════════════════════════════════════════

# Les replicas ne possèdent pas de slots, donc suppression simple


# ÉTAPE 1 : Identifier la replica à supprimer
# ────────────────────────────────────────────

redis-cli -h 192.168.1.10 CLUSTER NODES | grep slave

# Output :
# r1s2t3u4... 192.168.1.13:6379 slave a1b2c3d4... 0 1234567890 1 connected
# r5s6t7u8... 192.168.1.20:6379 slave a1b2c3d4... 0 1234567891 1 connected

# Décision : Supprimer la replica 192.168.1.20
REPLICA_ID=$(redis-cli -h 192.168.1.20 CLUSTER MYID)
echo "Replica ID to remove: $REPLICA_ID"


# ÉTAPE 2 : Supprimer du cluster
# ───────────────────────────────

redis-cli --cluster del-node 192.168.1.10:6379 $REPLICA_ID

# Syntaxe : del-node <any-cluster-node> <node-id-to-remove>

# Output :
# >>> Removing node r5s6t7u8... from cluster 192.168.1.10:6379
# >>> Sending CLUSTER FORGET messages to the cluster...
# >>> SHUTDOWN the node.


# ÉTAPE 3 : Arrêter le service Redis sur le nœud supprimé
# ────────────────────────────────────────────────────────

ssh 192.168.1.20
sudo systemctl stop redis
sudo systemctl disable redis

# Optionnel : Nettoyer les données
sudo rm -rf /var/lib/redis/*


# ÉTAPE 4 : Validation
# ────────────────────

redis-cli -h 192.168.1.10 CLUSTER NODES | grep 192.168.1.20
# Ne devrait rien retourner (nœud absent)

redis-cli --cluster check 192.168.1.10:6379
# [OK] All 16384 slots covered.
# Replica 192.168.1.20 ne devrait plus apparaître
```

### Suppression d'un master (avec redistribution)

```bash
# ═══════════════════════════════════════════════════════════
# SUPPRESSION D'UN MASTER (RESHARDING OBLIGATOIRE)
# ═══════════════════════════════════════════════════════════

# Contexte : Retirer Master D et redistribuer ses slots
# Topologie actuelle : 4 masters (A, B, C, D) avec 4096 slots chacun
# Topologie cible : 3 masters (A, B, C) avec ~5461 slots chacun


# ÉTAPE 1 : Vérifier l'état du master à supprimer
# ────────────────────────────────────────────────

redis-cli -h 192.168.1.16 CLUSTER INFO
# cluster_slots_assigned: devrait être > 0

redis-cli --cluster check 192.168.1.10:6379 | grep 192.168.1.16
# 192.168.1.16:6379 -> 4096 slots | 1 replica

MASTER_D_ID=$(redis-cli -h 192.168.1.16 CLUSTER MYID)
echo "Master D ID: $MASTER_D_ID"


# ÉTAPE 2 : Supprimer d'abord la replica du master D
# ───────────────────────────────────────────────────

# Identifier la replica de D
redis-cli -h 192.168.1.10 CLUSTER NODES | grep "slave $MASTER_D_ID"
# r9s0t1u2... 192.168.1.17:6379 slave <master-d-id>...

REPLICA_D_ID=$(redis-cli -h 192.168.1.17 CLUSTER MYID)

# Supprimer la replica
redis-cli --cluster del-node 192.168.1.10:6379 $REPLICA_D_ID

# Arrêter le service
ssh 192.168.1.17
sudo systemctl stop redis


# ÉTAPE 3 : Transférer TOUS les slots du master D
# ────────────────────────────────────────────────

# Option A : Redistribuer vers tous les autres masters
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-from $MASTER_D_ID \
    --cluster-to all \
    --cluster-slots 4096 \
    --cluster-yes \
    --cluster-pipeline 10

# Cette commande va :
# 1. Distribuer les 4096 slots de D vers A, B, C
# 2. A, B, C recevront chacun ~1365 slots
# 3. Nouvelle répartition : A=5461, B=5461, C=5462 slots

# Option B : Redistribuer vers un master spécifique
# NODE_A_ID=$(redis-cli -h 192.168.1.10 CLUSTER MYID)
# redis-cli --cluster reshard 192.168.1.10:6379 \
#     --cluster-from $MASTER_D_ID \
#     --cluster-to $NODE_A_ID \
#     --cluster-slots 4096 \
#     --cluster-yes


# ÉTAPE 4 : Vérifier que le master D n'a plus de slots
# ─────────────────────────────────────────────────────

redis-cli --cluster check 192.168.1.10:6379 | grep 192.168.1.16
# 192.168.1.16:6379 -> 0 keys | 0 slots | 0 replicas

# Vérifier localement
redis-cli -h 192.168.1.16 CLUSTER INFO | grep cluster_slots_assigned
# cluster_slots_assigned:0 ✓


# ÉTAPE 5 : Supprimer le master D du cluster
# ───────────────────────────────────────────

redis-cli --cluster del-node 192.168.1.10:6379 $MASTER_D_ID

# Output :
# >>> Removing node <master-d-id> from cluster 192.168.1.10:6379
# >>> Sending CLUSTER FORGET messages to the cluster...
# >>> SHUTDOWN the node.


# ÉTAPE 6 : Arrêter le service
# ─────────────────────────────

ssh 192.168.1.16
sudo systemctl stop redis
sudo systemctl disable redis

# Optionnel : Nettoyer
sudo rm -rf /var/lib/redis/*


# ÉTAPE 7 : Validation finale
# ────────────────────────────

redis-cli --cluster check 192.168.1.10:6379

# Output attendu :
# 192.168.1.10:6379 -> 5461 slots (33.3%)
# 192.168.1.11:6379 -> 5461 slots (33.3%)
# 192.168.1.12:6379 -> 5462 slots (33.4%)
# [OK] All 16384 slots covered.
# ← Master D absent

# Tester l'accès aux données
redis-cli -c -h 192.168.1.10 GET test:key
# Devrait fonctionner normalement
```

### Schéma du processus de suppression

```
┌─────────────────────────────────────────────────────────────┐
│        Processus de suppression d'un master                 │
└─────────────────────────────────────────────────────────────┘

État initial (4 masters)
────────────────────────

    Master A      Master B      Master C      Master D
    [0-4095]     [4096-8191]   [8192-12287]  [12288-16383]
       │             │             │              │
    Replica A1   Replica B1    Replica C1     Replica D1


Étape 1 : Supprimer Replica D1
───────────────────────────────

    Master A      Master B      Master C      Master D
    [0-4095]     [4096-8191]   [8192-12287]  [12288-16383]
       │             │             │              │
    Replica A1   Replica B1    Replica C1        X (supprimée)


Étape 2 : Transférer les slots de D
────────────────────────────────────

    Master A      Master B      Master C      Master D
    [0-4095]     [4096-8191]   [8192-12287]  [12288-16383]
       │             │             │              │
       │<────── 1365 slots ───────────────────────┘
       │             │<────── 1366 slots ─────────┘
       │             │             │<─ 1365 slots ┘


Étape 3 : Master D vidé, prêt à être supprimé
──────────────────────────────────────────────

    Master A      Master B      Master C      Master D
    [0-5460]     [5461-10922]  [10923-16383]   [VIDE]
       │             │             │              X
    Replica A1   Replica B1    Replica C1


Étape 4 : Suppression de Master D
──────────────────────────────────

    Master A      Master B      Master C
    [0-5460]     [5461-10922]  [10923-16383]
       │             │             │
    Replica A1   Replica B1    Replica C1

Retour à 3 masters (33.3% chacun) ✓
```

## Resharding avancé

### Stratégies de resharding

```
┌─────────────────────────────────────────────────────────────┐
│            Stratégies de Resharding                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. RESHARDING ÉQUILIBRÉ (Rebalance)                         │
│    ════════════════════════════════                         │
│    Objectif : Distribution uniforme sur tous les nœuds      │
│    Cas d'usage : Après ajout/suppression de nœuds           │
│                                                             │
│    redis-cli --cluster rebalance <node>                     │
│        --cluster-threshold 2                                │
│        --cluster-use-empty-masters                          │
│                                                             │
│    Résultat : Chaque master a ~(16384 / N) slots            │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 2. RESHARDING CIBLÉ (Targeted)                              │
│    ═══════════════════════════                              │
│    Objectif : Déplacer des slots spécifiques                │
│    Cas d'usage : Corriger un hot spot, décharger un nœud    │
│                                                             │
│    redis-cli --cluster reshard <node>                       │
│        --cluster-from <source-node-id>                      │
│        --cluster-to <target-node-id>                        │
│        --cluster-slots <number>                             │
│                                                             │
│    Résultat : Transfert précis d'un nombre de slots         │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 3. RESHARDING PAR HOT SPOTS                                 │
│    ════════════════════════════                             │
│    Objectif : Redistribuer les slots les plus chargés       │
│    Cas d'usage : Nœud saturé par quelques slots             │
│                                                             │
│    Analyse préalable :                                      │
│    for i in {0..16383}; do                                  │
│        count=$(redis-cli CLUSTER COUNTKEYSINSLOT $i)        │
│        echo "$i:$count"                                     │
│    done | sort -t: -k2 -n -r | head -20                     │
│                                                             │
│    Puis resharding manuel des slots identifiés              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 4. RESHARDING PROGRESSIF (Incremental)                      │
│    ═══════════════════════════════════                      │
│    Objectif : Migration par petits lots pour limiter impact │
│    Cas d'usage : Production avec charge élevée              │
│                                                             │
│    for batch in {1..10}; do                                 │
│        redis-cli --cluster reshard <node>                   │
│            --cluster-slots 100                              │
│            --cluster-pipeline 5                             │
│        sleep 60  # Pause entre les lots                     │
│    done                                                     │
│                                                             │
│    Résultat : Migration de 1000 slots en 10 lots de 100     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Resharding manuel avancé

```bash
# ═══════════════════════════════════════════════════════════
# RESHARDING MANUEL AVEC CONTRÔLE TOTAL
# ═══════════════════════════════════════════════════════════

# Scénario : Déplacer les slots 8000-8099 de Node B vers Node A


# ÉTAPE 1 : Identifier les nœuds source et destination
# ─────────────────────────────────────────────────────

SOURCE_NODE="192.168.1.11"  # Node B
TARGET_NODE="192.168.1.10"  # Node A

SOURCE_ID=$(redis-cli -h $SOURCE_NODE CLUSTER MYID)
TARGET_ID=$(redis-cli -h $TARGET_NODE CLUSTER MYID)

echo "Source: $SOURCE_NODE ($SOURCE_ID)"
echo "Target: $TARGET_NODE ($TARGET_ID)"


# ÉTAPE 2 : Marquer les slots en migration
# ─────────────────────────────────────────

# Pour chaque slot à migrer
for slot in {8000..8099}; do
    # Sur le nœud source : marquer comme MIGRATING
    redis-cli -h $SOURCE_NODE CLUSTER SETSLOT $slot MIGRATING $TARGET_ID

    # Sur le nœud destination : marquer comme IMPORTING
    redis-cli -h $TARGET_NODE CLUSTER SETSLOT $slot IMPORTING $SOURCE_ID
done

echo "Slots 8000-8099 marked for migration"


# ÉTAPE 3 : Migrer les clés slot par slot
# ────────────────────────────────────────

for slot in {8000..8099}; do
    echo "Migrating slot $slot..."

    # Obtenir toutes les clés du slot
    keys=$(redis-cli -h $SOURCE_NODE CLUSTER GETKEYSINSLOT $slot 1000)

    # Si le slot contient des clés, les migrer
    if [ ! -z "$keys" ]; then
        for key in $keys; do
            redis-cli -h $SOURCE_NODE MIGRATE \
                $TARGET_NODE 6379 "$key" 0 5000
        done
    fi

    echo "Slot $slot migrated (or was empty)"
done


# ÉTAPE 4 : Finaliser la migration (changer ownership)
# ─────────────────────────────────────────────────────

for slot in {8000..8099}; do
    # Sur TOUS les nœuds du cluster, déclarer le nouveau propriétaire
    redis-cli -h $SOURCE_NODE CLUSTER SETSLOT $slot NODE $TARGET_ID
    redis-cli -h $TARGET_NODE CLUSTER SETSLOT $slot NODE $TARGET_ID

    # Optionnel : sur les autres nœuds aussi
    redis-cli -h 192.168.1.12 CLUSTER SETSLOT $slot NODE $TARGET_ID
done

echo "Migration finalized"


# ÉTAPE 5 : Vérification
# ──────────────────────

redis-cli --cluster check 192.168.1.10:6379 | grep -E "192.168.1.10|192.168.1.11"

# Vérifier que les slots ont changé de propriétaire
redis-cli -h 192.168.1.10 CLUSTER NODES | grep "myself" | grep "8000-8099"
# Devrait montrer que Node A possède maintenant 8000-8099
```

### Optimisation du resharding

```bash
# ═══════════════════════════════════════════════════════════
# TECHNIQUES D'OPTIMISATION DU RESHARDING
# ═══════════════════════════════════════════════════════════


# 1. PIPELINE : Migrer plusieurs clés en parallèle
# ─────────────────────────────────────────────────

# Sans pipeline (lent)
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-slots 1000

# Avec pipeline (plus rapide)
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-slots 1000 \
    --cluster-pipeline 20  # Migrer 20 clés en parallèle

# Impact :
# - Sans pipeline : ~10 clés/sec (RTT × nombre de clés)
# - Avec pipeline=20 : ~200 clés/sec (RTT amortisé)


# 2. TIMEOUT : Ajuster le timeout de migration
# ─────────────────────────────────────────────

redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-timeout 60000  # 60 secondes (défaut: 15s)

# Utiliser un timeout plus long pour :
# - Grosses clés (>1MB)
# - Réseau lent ou latence élevée
# - Charge du cluster élevée


# 3. REPLACE : Écraser les clés existantes
# ─────────────────────────────────────────

redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-slots 1000 \
    --cluster-replace  # Remplacer si clé existe déjà

# Attention : Peut causer des pertes de données si utilisé incorrectement


# 4. COPY : Mode non-destructif (Redis 7+)
# ─────────────────────────────────────────

# MIGRATE avec option COPY
redis-cli -h 192.168.1.10 MIGRATE 192.168.1.11 6379 "mykey" 0 5000 COPY

# La clé est copiée mais pas supprimée de la source
# Utile pour tester avant de finaliser


# 5. Resharding par lots avec monitoring
# ───────────────────────────────────────

#!/bin/bash
# reshard-progressive.sh

TOTAL_SLOTS=1000
BATCH_SIZE=100
PAUSE_SECONDS=30

for ((i=0; i<$TOTAL_SLOTS; i+=$BATCH_SIZE)); do
    echo "Batch $((i/$BATCH_SIZE + 1)): Migrating $BATCH_SIZE slots..."

    # Migrer un lot
    redis-cli --cluster reshard 192.168.1.10:6379 \
        --cluster-from <source-id> \
        --cluster-to <target-id> \
        --cluster-slots $BATCH_SIZE \
        --cluster-yes \
        --cluster-pipeline 10

    # Pause pour laisser le cluster se stabiliser
    echo "Pausing for $PAUSE_SECONDS seconds..."
    sleep $PAUSE_SECONDS

    # Vérifier l'état
    redis-cli -h 192.168.1.10 CLUSTER INFO | grep cluster_state

    echo "Batch completed. Progress: $((i+BATCH_SIZE))/$TOTAL_SLOTS slots"
    echo "─────────────────────────────────────"
done

echo "Resharding completed!"
```

## Opérations de maintenance avancées

### Réassignation de replicas

```bash
# ═══════════════════════════════════════════════════════════
# CHANGER LE MASTER D'UNE REPLICA
# ═══════════════════════════════════════════════════════════

# Scénario : Replica R1 réplique Master A
#            Objectif : R1 doit répliquer Master B


# ÉTAPE 1 : Identifier la replica et les masters
# ───────────────────────────────────────────────

REPLICA_NODE="192.168.1.13"
OLD_MASTER="192.168.1.10"  # Master A
NEW_MASTER="192.168.1.11"  # Master B

NEW_MASTER_ID=$(redis-cli -h $NEW_MASTER CLUSTER MYID)


# ÉTAPE 2 : Changer le master de la replica
# ──────────────────────────────────────────

redis-cli -h $REPLICA_NODE CLUSTER REPLICATE $NEW_MASTER_ID

# Cette commande :
# 1. Arrête la réplication avec l'ancien master
# 2. Vide les données actuelles de la replica
# 3. Démarre la réplication complète avec le nouveau master
# 4. Synchronise toutes les données du nouveau master


# ÉTAPE 3 : Vérification
# ──────────────────────

redis-cli -h $REPLICA_NODE INFO replication
# role:slave
# master_host:192.168.1.11  ← Nouveau master
# master_port:6379
# master_link_status:up

redis-cli -h $NEW_MASTER INFO replication
# connected_slaves:2  ← +1 replica

redis-cli -h $OLD_MASTER INFO replication
# connected_slaves:0  ← -1 replica


# ÉTAPE 4 : Vérifier la santé du cluster
# ───────────────────────────────────────

redis-cli --cluster check 192.168.1.10:6379
# Devrait montrer la nouvelle topologie
```

### Équilibrage automatique des replicas

```bash
# ═══════════════════════════════════════════════════════════
# REDISTRIBUTION AUTOMATIQUE DES REPLICAS
# ═══════════════════════════════════════════════════════════

# Redis peut automatiquement migrer des replicas vers masters orphelins


# Configuration (redis.conf)
cluster-migration-barrier 1
# Minimum de replicas à garder avant migration automatique

cluster-allow-replica-migration yes
# Autoriser la migration automatique (défaut: yes)


# Scénario déclenchant la migration automatique :
# ───────────────────────────────────────────────
#
# État initial :
# Master A : 2 replicas (R1, R2)
# Master B : 0 replicas (orphan)
# Master C : 1 replica (R3)
#
# Action automatique de Redis :
# Master A : 1 replica (R1) ← R2 migrée automatiquement
# Master B : 1 replica (R2) ← Récupère une replica
# Master C : 1 replica (R3)


# Forcer une migration manuelle de replica
# ─────────────────────────────────────────

# Identifier une replica "en trop"
redis-cli -h 192.168.1.10 INFO replication
# connected_slaves:2 (R1, R2)

# Identifier un master orphelin
redis-cli --cluster check 192.168.1.10:6379 | grep "0 slaves"
# Master B : 0 slaves

# Déplacer R2 vers Master B
MASTER_B_ID=$(redis-cli -h 192.168.1.11 CLUSTER MYID)
redis-cli -h 192.168.1.14 CLUSTER REPLICATE $MASTER_B_ID
```

### Maintenance d'un nœud sans downtime

```bash
# ═══════════════════════════════════════════════════════════
# MAINTENANCE D'UN MASTER SANS INTERRUPTION
# ═══════════════════════════════════════════════════════════

# Objectif : Mettre à jour/réparer un master sans impacter le service


# ÉTAPE 1 : Identifier le master à maintenir
# ───────────────────────────────────────────

MASTER_TO_MAINTAIN="192.168.1.10"
MASTER_ID=$(redis-cli -h $MASTER_TO_MAINTAIN CLUSTER MYID)


# ÉTAPE 2 : Vérifier qu'il a une replica
# ───────────────────────────────────────

redis-cli -h $MASTER_TO_MAINTAIN INFO replication | grep connected_slaves
# connected_slaves:1 ✓

# Identifier la replica
REPLICA=$(redis-cli -h 192.168.1.10 CLUSTER NODES | grep "slave $MASTER_ID" | awk '{print $2}')
REPLICA_HOST=$(echo $REPLICA | cut -d: -f1)
echo "Replica: $REPLICA_HOST"


# ÉTAPE 3 : Failover manuel PLANIFIÉ (pas d'attente)
# ───────────────────────────────────────────────────

# Sur la replica, déclencher un failover sans attendre
redis-cli -h $REPLICA_HOST CLUSTER FAILOVER TAKEOVER

# Cette commande :
# 1. La replica devient immédiatement master
# 2. L'ancien master devient replica automatiquement
# 3. Pas d'interruption de service (basculement instantané)
# 4. Pas d'attente du cluster-node-timeout


# ÉTAPE 4 : Vérifier le basculement
# ──────────────────────────────────

sleep 5  # Laisser le temps au basculement

redis-cli -h $REPLICA_HOST CLUSTER NODES | grep myself
# Devrait montrer : myself,master

redis-cli -h $MASTER_TO_MAINTAIN CLUSTER NODES | grep myself
# Devrait montrer : myself,slave


# ÉTAPE 5 : Maintenance sur l'ancien master (maintenant replica)
# ───────────────────────────────────────────────────────────────

ssh $MASTER_TO_MAINTAIN

# Arrêter Redis
sudo systemctl stop redis

# Effectuer la maintenance
# - Mise à jour OS : apt upgrade / yum update
# - Upgrade Redis : installer nouvelle version
# - Réparation disque
# - Changement de configuration
# - etc.

# Redémarrer Redis
sudo systemctl start redis

# Vérifier que la réplication reprend
redis-cli -h $MASTER_TO_MAINTAIN INFO replication
# role:slave
# master_link_status:up


# ÉTAPE 6 : Optionnel - Restaurer la topologie initiale
# ──────────────────────────────────────────────────────

# Si souhaité, refaire basculer pour remettre comme master
redis-cli -h $MASTER_TO_MAINTAIN CLUSTER FAILOVER TAKEOVER

# Résultat : Topologie restaurée, maintenance terminée


# DOWNTIME TOTAL : 0 secondes ✓
```

### Remplacement d'un nœud défaillant

```bash
# ═══════════════════════════════════════════════════════════
# REMPLACER UN NŒUD MATÉRIELLEMENT DÉFAILLANT
# ═══════════════════════════════════════════════════════════

# Scénario : Un serveur physique est mort et ne redémarrera pas
# Objectif : Remplacer par un nouveau serveur


# CONTEXTE :
# Master A (192.168.1.10) : MORT (hardware failure)
# Replica A1 (192.168.1.13) : UP et promue en master automatiquement


# ÉTAPE 1 : Vérifier l'état après failover automatique
# ─────────────────────────────────────────────────────

redis-cli -h 192.168.1.11 CLUSTER NODES

# Output :
# a1b2c3d4... 192.168.1.10:6379 master,fail - 1234567890 ...
# r1s2t3u4... 192.168.1.13:6379 master - 1234567900 ...  ← Promue

# L'ancien master A est marqué comme "fail"
# La replica A1 est devenue master


# ÉTAPE 2 : Retirer définitivement l'ancien nœud défaillant
# ──────────────────────────────────────────────────────────

FAILED_NODE_ID="a1b2c3d4..."  # ID de l'ancien master

# Depuis n'importe quel nœud actif
redis-cli -h 192.168.1.11 CLUSTER FORGET $FAILED_NODE_ID

# Répéter sur tous les nœuds pour propager
for node in 192.168.1.11 192.168.1.12 192.168.1.13 192.168.1.14 192.168.1.15; do
    redis-cli -h $node CLUSTER FORGET $FAILED_NODE_ID
done

# Après 60 secondes, le nœud sera complètement oublié du cluster


# ÉTAPE 3 : Préparer le nouveau serveur de remplacement
# ──────────────────────────────────────────────────────

NEW_SERVER="192.168.1.20"  # Nouveau serveur physique

ssh $NEW_SERVER
# Installer et configurer Redis (même config que l'ancien)
sudo apt install redis-server
# ... configuration ...
sudo systemctl start redis


# ÉTAPE 4 : Ajouter le nouveau nœud comme replica
# ────────────────────────────────────────────────

# Le nouveau master (ex-replica A1) n'a plus de replica
# On ajoute le nouveau serveur comme replica de ce master

NEW_MASTER_ID=$(redis-cli -h 192.168.1.13 CLUSTER MYID)

redis-cli --cluster add-node $NEW_SERVER:6379 192.168.1.11:6379 \
    --cluster-slave \
    --cluster-master-id $NEW_MASTER_ID


# ÉTAPE 5 : Validation
# ────────────────────

redis-cli --cluster check 192.168.1.11:6379

# Devrait montrer :
# Master A1 (192.168.1.13) : X slots | 1 replica (192.168.1.20)
# ← Nouveau serveur opérationnel


# RÉSULTAT :
# Ancien serveur mort → Oublié
# Nouveau serveur → Intégré comme replica
# Haute disponibilité restaurée ✓
```

## Monitoring et troubleshooting des opérations

### Scripts de monitoring pendant les opérations

```bash
#!/bin/bash
# monitor-cluster-operations.sh
# Monitoring en temps réel des opérations de gestion de nœuds

CLUSTER_NODE="192.168.1.10:6379"
REFRESH_INTERVAL=2

while true; do
    clear
    echo "=========================================="
    echo "Redis Cluster Operation Monitor"
    echo "Time: $(date +'%Y-%m-%d %H:%M:%S')"
    echo "=========================================="
    echo ""

    # État global du cluster
    echo "=== Cluster State ==="
    redis-cli -h ${CLUSTER_NODE%:*} CLUSTER INFO | grep -E "cluster_state|cluster_slots_assigned|cluster_slots_ok|cluster_known_nodes|cluster_size"
    echo ""

    # Distribution des slots par nœud
    echo "=== Slots Distribution ==="
    redis-cli --cluster check $CLUSTER_NODE | grep -E "slots|OK"
    echo ""

    # Slots en migration
    echo "=== Migrations in Progress ==="
    migrations=$(redis-cli -h ${CLUSTER_NODE%:*} CLUSTER NODES | grep -E "MIGRATING|IMPORTING")
    if [ -z "$migrations" ]; then
        echo "No migrations in progress"
    else
        echo "$migrations"
    fi
    echo ""

    # Nœuds en échec
    echo "=== Failed Nodes ==="
    failed=$(redis-cli -h ${CLUSTER_NODE%:*} CLUSTER NODES | grep fail)
    if [ -z "$failed" ]; then
        echo "No failed nodes"
    else
        echo "$failed"
    fi
    echo ""

    # Métriques de performance
    echo "=== Performance Metrics ==="
    redis-cli -h ${CLUSTER_NODE%:*} INFO stats | grep -E "instantaneous_ops_per_sec|total_commands_processed"
    echo ""

    sleep $REFRESH_INTERVAL
done
```

### Commandes de diagnostic

```bash
# ═══════════════════════════════════════════════════════════
# DIAGNOSTIC COMPLET D'OPÉRATIONS DE GESTION
# ═══════════════════════════════════════════════════════════


# 1. Vérifier l'état des migrations
# ──────────────────────────────────

# Voir tous les slots en cours de migration
redis-cli -h 192.168.1.10 CLUSTER NODES | grep -E "importing|migrating"

# Compter les slots en migration
redis-cli -h 192.168.1.10 CLUSTER NODES | grep -c migrating

# Détail d'un slot spécifique
redis-cli -h 192.168.1.10 CLUSTER SLOTS | grep -A 2 "8000"


# 2. Analyser la performance pendant resharding
# ──────────────────────────────────────────────

# Latence en temps réel
redis-cli -h 192.168.1.10 --latency-history

# Taux de redirections
redis-cli -h 192.168.1.10 INFO stats | grep keyspace_misses

# Charge du serveur
redis-cli -h 192.168.1.10 INFO cpu


# 3. Vérifier la cohérence du cluster
# ────────────────────────────────────

# Check complet avec détails
redis-cli --cluster check 192.168.1.10:6379 --cluster-search-multiple-owners

# Fix automatique si problèmes détectés
redis-cli --cluster fix 192.168.1.10:6379


# 4. Analyser les logs pendant les opérations
# ────────────────────────────────────────────

# Suivre les logs en temps réel
tail -f /var/log/redis/redis.log | grep -E "CLUSTER|MIGRATE|SLOT"

# Rechercher les erreurs
grep -i error /var/log/redis/redis.log | tail -20


# 5. Tester l'accès aux données pendant migration
# ────────────────────────────────────────────────

# Écrire et lire continuellement
while true; do
    redis-cli -c -h 192.168.1.10 SET test:$(date +%s) "$(date)"
    redis-cli -c -h 192.168.1.10 GET test:$(date +%s)
    sleep 1
done


# 6. Vérifier la distribution des clés après resharding
# ──────────────────────────────────────────────────────

for node in 192.168.1.10 192.168.1.11 192.168.1.12; do
    keys=$(redis-cli -h $node DBSIZE)
    echo "$node: $keys keys"
done


# 7. Analyser les clés par slot
# ──────────────────────────────

#!/bin/bash
# count-keys-per-slot.sh

for slot in {0..16383}; do
    count=$(redis-cli -h 192.168.1.10 CLUSTER COUNTKEYSINSLOT $slot)
    if [ $count -gt 0 ]; then
        echo "Slot $slot: $count keys"
    fi
done | sort -t: -k2 -n -r | head -50


# 8. Vérifier la santé de la réplication
# ───────────────────────────────────────

for node in 192.168.1.10 192.168.1.11 192.168.1.12; do
    echo "=== $node ==="
    redis-cli -h $node INFO replication | grep -E "role|connected_slaves|master_link_status"
    echo ""
done
```

### Troubleshooting des problèmes courants

```
┌─────────────────────────────────────────────────────────────┐
│         Problèmes courants lors de la gestion de nœuds      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 1 : Resharding bloqué                              │
│ ═══════════════════════════                                 │
│ Symptôme : Migration ne progresse pas                       │
│                                                             │
│ Causes :                                                    │
│ ├─ Slots en état MIGRATING/IMPORTING non finalisés          │
│ ├─ Clé très volumineuse bloque la migration                 │
│ └─ Timeout réseau trop court                                │
│                                                             │
│ Diagnostic :                                                │
│ redis-cli CLUSTER NODES | grep -E "importing|migrating"     │
│ redis-cli CLUSTER SLOTS                                     │
│                                                             │
│ Solutions :                                                 │
│ ├─ Réinitialiser l'état : CLUSTER SETSLOT <slot> STABLE     │
│ ├─ Augmenter timeout : --cluster-timeout 60000              │
│ └─ Migration manuelle des grosses clés                      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 2 : Impossible de supprimer un nœud                │
│ ═══════════════════════════════════════════                 │
│ Symptôme : del-node échoue                                  │
│                                                             │
│ Causes :                                                    │
│ ├─ Le nœud possède encore des slots                         │
│ ├─ Le nœud est un master avec replicas                      │
│ └─ Le nœud n'est pas dans le cluster                        │
│                                                             │
│ Solutions :                                                 │
│ ├─ Vérifier : redis-cli CLUSTER INFO                        │
│ ├─ Transférer tous les slots avant suppression              │
│ ├─ Supprimer les replicas d'abord                           │
│ └─ Utiliser CLUSTER FORGET si définitivement mort           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 3 : Slots orphelins après opération                │
│ ═════════════════════════════════════════════               │
│ Symptôme : cluster_slots_ok < 16384                         │
│                                                             │
│ Causes :                                                    │
│ ├─ Opération interrompue brutalement                        │
│ ├─ Nœud crash pendant migration                             │
│ └─ Erreur réseau pendant resharding                         │
│                                                             │
│ Solutions :                                                 │
│ redis-cli --cluster fix 192.168.1.10:6379                   │
│                                                             │
│ Ou manuellement :                                           │
│ redis-cli CLUSTER SETSLOT <slot> STABLE                     │
│ redis-cli CLUSTER SETSLOT <slot> NODE <node-id>             │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 4 : Performance dégradée pendant resharding        │
│ ════════════════════════════════════════════════            │
│ Symptôme : Latence élevée, timeouts                         │
│                                                             │
│ Causes :                                                    │
│ ├─ Trop de clés migrées simultanément                       │
│ ├─ Grosses clés (>1MB)                                      │
│ └─ Charge CPU/réseau élevée                                 │
│                                                             │
│ Solutions :                                                 │
│ ├─ Réduire pipeline : --cluster-pipeline 5                  │
│ ├─ Migration par petits lots (100 slots à la fois)          │
│ ├─ Pause entre les lots : sleep 60                          │
│ └─ Planifier pendant heures creuses                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Best practices opérationnelles

```
┌─────────────────────────────────────────────────────────────┐
│        Best Practices pour la gestion de nœuds              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ✓ PLANIFICATION                                             │
│   ├─ Documenter l'architecture avant et après               │
│   ├─ Calculer la nouvelle distribution de slots             │
│   ├─ Estimer la durée de resharding                         │
│   ├─ Planifier pendant période de faible charge             │
│   └─ Prévoir un rollback si possible                        │
│                                                             │
│ ✓ EXÉCUTION                                                 │
│   ├─ Toujours traiter les replicas avant les masters        │
│   ├─ Utiliser --cluster-yes pour éviter prompts             │
│   ├─ Monitorer en temps réel (latence, slots, erreurs)      │
│   ├─ Migration progressive (petits lots)                    │
│   └─ Garder les logs de toutes les opérations               │
│                                                             │
│ ✓ VALIDATION                                                │
│   ├─ Vérifier : cluster_state:ok                            │
│   ├─ Vérifier : cluster_slots_ok:16384                      │
│   ├─ Tester lecture/écriture sur tous les nœuds             │
│   ├─ Comparer nombre de clés avant/après                    │
│   └─ Valider la distribution (rebalance)                    │
│                                                             │
│ ✓ SÉCURITÉ                                                  │
│   ├─ Backup complet avant opérations majeures               │
│   ├─ Tester en staging d'abord                              │
│   ├─ Avoir un plan de rollback                              │
│   ├─ Communication avec l'équipe                            │
│   └─ Monitoring des alertes                                 │
│                                                             │
│ ✓ PERFORMANCE                                               │
│   ├─ Utiliser --cluster-pipeline (10-20)                    │
│   ├─ Éviter les opérations pendant pics de charge           │
│   ├─ Surveiller la latence réseau                           │
│   ├─ Pause entre lots si charge élevée                      │
│   └─ Optimiser TCP (buffer size, keepalive)                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Conclusion

La gestion dynamique des nœuds est une capacité fondamentale de Redis Cluster qui permet l'adaptation continue aux besoins évolutifs. Les opérations d'ajout, de suppression et de resharding, bien que complexes, peuvent être exécutées sans interruption de service lorsqu'elles sont planifiées et exécutées méthodiquement.

Les points essentiels à retenir :

1. **Ordre des opérations** : Toujours traiter replicas avant masters
2. **Resharding progressif** : Privilégier petits lots avec monitoring
3. **Validation rigoureuse** : Vérifier l'état après chaque opération majeure
4. **Monitoring continu** : Surveiller métriques pendant toute l'opération
5. **Documentation** : Tracer toutes les modifications d'architecture

Une maîtrise approfondie de ces opérations est indispensable pour maintenir un cluster Redis performant, équilibré et hautement disponible en production.

---

**Points clés à retenir :**

- **Ajout de master** : MEET → Rebalance → Ajouter replica
- **Suppression de master** : Supprimer replica → Vider slots → del-node
- **Resharding** : Utiliser pipeline, lots progressifs, monitoring continu
- **Failover manuel** : CLUSTER FAILOVER TAKEOVER pour maintenance sans downtime
- **Validation** : cluster_state:ok + cluster_slots_ok:16384 obligatoire
- **Troubleshooting** : CLUSTER FIX pour corriger slots orphelins
- **Performance** : --cluster-pipeline pour optimiser migrations
- **Sécurité** : Backup avant opérations, test en staging

La section suivante (11.6) explorera les limitations du cluster et les contraintes architecturales à prendre en compte.

⏭️ [Limitations du Cluster (multi-key operations, transactions)](/11-architecture-distribuee-scaling/06-limitations-cluster.md)
