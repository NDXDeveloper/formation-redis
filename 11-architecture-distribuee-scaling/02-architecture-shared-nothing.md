🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.2 Architecture "Shared-Nothing" du Cluster

## Introduction

L'architecture "Shared-Nothing" (ou "non partagée") constitue le paradigme fondamental sur lequel repose Redis Cluster. Ce modèle architectural, popularisé dans les années 1980 par Michael Stonebraker, représente une rupture radicale avec les architectures traditionnelles de bases de données qui s'appuyaient sur des ressources partagées (mémoire, disque, ou processeur).

Dans une architecture Shared-Nothing, chaque nœud du cluster est complètement autonome et indépendant. Il possède ses propres ressources (CPU, RAM, stockage) et ne partage rien avec les autres nœuds, communiquant uniquement via des messages réseau. Cette approche élimine les points de contention centralisés et permet un scaling horizontal quasi-illimité.

## Principes fondamentaux de l'architecture Shared-Nothing

### Définition formelle

```
┌────────────────────────────────────────────────────────────┐
│          Architecture Shared-Nothing : Définition          │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Propriétés caractéristiques :                             │
│                                                            │
│  1. AUTONOMIE COMPLÈTE                                     │
│     └─> Chaque nœud = unité indépendante                   │
│         ├─ Processus distincts                             │
│         ├─ Espace mémoire isolé                            │
│         └─ Stockage local dédié                            │
│                                                            │
│  2. PAS DE RESSOURCE PARTAGÉE                              │
│     └─> Aucun accès direct aux ressources d'autres nœuds   │
│         ├─ Pas de mémoire partagée (No Shared Memory)      │
│         ├─ Pas de disque partagé (No Shared Disk)          │
│         └─ Pas de bus système commun                       │
│                                                            │
│  3. COMMUNICATION PAR MESSAGES                             │
│     └─> Échange d'informations uniquement via réseau       │
│         ├─ Protocole de communication défini               │
│         ├─ Messages asynchrones                            │
│         └─ Pas de mémoire transactionnelle distribuée      │
│                                                            │
│  4. PARTITIONNEMENT DES DONNÉES                            │
│     └─> Chaque nœud possède un sous-ensemble des données   │
│         ├─ Pas de duplication (sauf réplication explicite) │
│         ├─ Partition par clé (sharding)                    │
│         └─ Localité des données                            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Comparaison avec les architectures alternatives

```
┌─────────────────────────────────────────────────────────────┐
│     Taxonomie des architectures de systèmes distribués      │
└─────────────────────────────────────────────────────────────┘

1. SHARED-MEMORY (Mémoire Partagée)
═══════════════════════════════════════
   ┌─────────────────────────────┐
   │    Mémoire Commune (RAM)    │
   └─────────────────────────────┘
            │         │         │
        ┌───▼───┐ ┌───▼───┐ ┌───▼───┐
        │ CPU 1 │ │ CPU 2 │ │ CPU 3 │
        └───────┘ └───────┘ └───────┘

   Exemples : SMP (Symmetric Multi-Processing), NUMA
   ✓ Latence très faible pour accès mémoire
   ✗ Scaling limité (~100 cores max)
   ✗ Coût très élevé
   ✗ Point de défaillance unique


2. SHARED-DISK (Disque Partagé)
═══════════════════════════════════════
        ┌──────┐    ┌──────┐    ┌──────┐
        │Node 1│    │Node 2│    │Node 3│
        │ RAM  │    │ RAM  │    │ RAM  │
        └───┬──┘    └───┬──┘    └───┬──┘
            │           │           │
            └───────────┼───────────┘
                        │
                   ┌────▼────┐
                   │   SAN   │
                   │ Storage │
                   └─────────┘

   Exemples : Oracle RAC, IBM DB2 pureScale
   ✓ Partage des données facilité
   ✓ Pas de repartitionnement nécessaire
   ✗ Contention sur le stockage partagé
   ✗ Réseau SAN = coût élevé + SPOF
   ✗ Scaling limité par bande passante SAN


3. SHARED-NOTHING (Rien de Partagé) ✓ Redis Cluster
═══════════════════════════════════════════════════
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  Node 1  │    │  Node 2  │    │  Node 3  │
    ├──────────┤    ├──────────┤    ├──────────┤
    │   RAM    │    │   RAM    │    │   RAM    │
    ├──────────┤    ├──────────┤    ├──────────┤
    │   Disk   │    │   Disk   │    │   Disk   │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
         └───────────────┼───────────────┘
                    Network Only

   Exemples : Redis Cluster, Cassandra, MongoDB sharded
   ✓ Scaling horizontal quasi-illimité
   ✓ Pas de SPOF ni de bottleneck centralisé
   ✓ Isolation des pannes par nœud
   ✓ Coût linéaire (commodity hardware)
   ✗ Complexité accrue (partitionnement, routing)
   ✗ Opérations multi-partitions limitées
   ✗ Cohérence éventuelle (trade-off CAP)
```

## Implémentation dans Redis Cluster

### Architecture physique d'un nœud

```
┌─────────────────────────────────────────────────────────────┐
│                Anatomie d'un nœud Redis Cluster             │
└─────────────────────────────────────────────────────────────┘

  Serveur Physique / VM / Container
  ┌───────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌──────────────────────────────────────────────────────┐ │
  │  │        Processus Redis (Instance unique)             │ │
  │  ├──────────────────────────────────────────────────────┤ │
  │  │                                                      │ │
  │  │  Port 6379 : Client Protocol                         │ │
  │  │  ├─ Accepte connexions clients                       │ │
  │  │  ├─ Gère commandes Redis (GET, SET, ...)             │ │
  │  │  └─ Retourne résultats ou redirections               │ │
  │  │                                                      │ │
  │  │  Port 16379 : Cluster Bus (Gossip)                   │ │
  │  │  ├─ Communication inter-nœuds                        │ │
  │  │  ├─ Heartbeats (PING/PONG)                           │ │
  │  │  └─ Propagation d'état                               │ │
  │  │                                                      │ │
  │  ├──────────────────────────────────────────────────────┤ │
  │  │           Espace Mémoire (RAM)                       │ │
  │  ├──────────────────────────────────────────────────────┤ │
  │  │  • Dataset local (hash slots assignés)               │ │
  │  │  • Table de routing (tous les slots → nœuds)         │ │
  │  │  • État du cluster (clusterState)                    │ │
  │  │  • Buffer réplication                                │ │
  │  │  • Buffer client output                              │ │
  │  └──────────────────────────────────────────────────────┘ │
  │                                                           │
  │  ┌──────────────────────────────────────────────────────┐ │
  │  │              Stockage Local (Disk)                   │ │
  │  ├──────────────────────────────────────────────────────┤ │
  │  │  • RDB snapshots (dump.rdb)                          │ │
  │  │  • AOF log (appendonly.aof)                          │ │
  │  │  • Cluster configuration (nodes.conf)                │ │
  │  └──────────────────────────────────────────────────────┘ │
  │                                                           │
  └───────────────────────────────────────────────────────────┘

Principe clé : Aucun accès direct aux ressources des autres nœuds
```

### Isolation mémoire complète

Dans Redis Cluster, chaque nœud gère sa propre mémoire de manière totalement autonome :

```
Cluster de 3 nœuds avec 64 GB RAM chacun :
─────────────────────────────────────────────

Node A (192.168.1.10)             Node B (192.168.1.11)
┌─────────────────────┐           ┌─────────────────────┐
│   RAM : 64 GB       │           │   RAM : 64 GB       │
├─────────────────────┤           ├─────────────────────┤
│ Slots : 0-5460      │           │ Slots : 5461-10922  │
│ Keys  : ~10M        │           │ Keys  : ~10M        │
│ Used  : 42 GB       │           │ Used  : 43 GB       │
│ Free  : 22 GB       │           │ Free  : 21 GB       │
└─────────────────────┘           └─────────────────────┘
         │                                   │
         │                                   │
         └────────────┬──────────────────────┘
                      │
           Network Communication Only
                      │
         ┌────────────▼──────────────┐
         │   Node C (192.168.1.12)   │
         ├───────────────────────────┤
         │   RAM : 64 GB             │
         ├───────────────────────────┤
         │ Slots : 10923-16383       │
         │ Keys  : ~10M              │
         │ Used  : 41 GB             │
         │ Free  : 23 GB             │
         └───────────────────────────┘

Caractéristiques :
══════════════════
• Chaque nœud alloue et libère sa mémoire indépendamment
• Pas de synchronisation des allocations mémoire
• Fragmentation locale à chaque nœud
• Politique d'éviction (maxmemory-policy) locale
• Capacité totale = Somme des capacités individuelles
  → 64 + 64 + 64 = 192 GB (vs 64 GB pour un seul nœud)
```

### Absence de transaction distribuée

Conséquence directe de l'architecture Shared-Nothing :

```
┌─────────────────────────────────────────────────────────────┐
│        Transactions : Scope limité au nœud local            │
└─────────────────────────────────────────────────────────────┘

SCÉNARIO SUPPORTÉ (Clés sur le même slot)
══════════════════════════════════════════

Client exécute :
────────────────
MULTI
SET {user:1000}:profile "John Doe"
SET {user:1000}:email "john@example.com"
INCR {user:1000}:login_count
EXEC

Calcul des slots :
──────────────────
{user:1000}:profile      → CRC16("user:1000") = Slot 5649
{user:1000}:email        → CRC16("user:1000") = Slot 5649  ✓
{user:1000}:login_count  → CRC16("user:1000") = Slot 5649  ✓

Toutes les clés → même slot → même nœud → Transaction OK


SCÉNARIO NON SUPPORTÉ (Clés sur slots différents)
══════════════════════════════════════════════════

Client exécute :
────────────────
MULTI
SET user:1000 "John"     → Slot 5798 (Node A)
SET user:2000 "Jane"     → Slot 8234 (Node B)
SET user:3000 "Bob"      → Slot 12456 (Node C)
EXEC

Résultat :
──────────
❌ CROSSSLOT Keys in request don't hash to the same slot

Raison :
────────
Redis Cluster ne peut pas garantir l'atomicité d'une transaction
qui touche plusieurs nœuds (pas de 2PC - Two-Phase Commit)

Solutions :
───────────
1. Utiliser les hash tags pour co-localiser
2. Restructurer le modèle de données
3. Accepter plusieurs transactions séparées (sans atomicité globale)
```

## Avantages de l'architecture Shared-Nothing

### 1. Scaling horizontal illimité (théorique)

```
Performance et Capacité = Fonction linéaire du nombre de nœuds
════════════════════════════════════════════════════════════════

Capacité mémoire :
──────────────────
1 nœud   →  64 GB
10 nœuds →  640 GB
100 nœuds → 6.4 TB
1000 nœuds → 64 TB

Formule : Capacité_totale = N × RAM_par_nœud


Throughput (ops/sec) :
──────────────────────
1 nœud   →  100,000 ops/sec
10 nœuds →  1,000,000 ops/sec
100 nœuds → 10,000,000 ops/sec

Formule : Throughput_total ≈ N × Throughput_par_nœud
(avec overhead minimal du routing et gossip)


Comparaison avec Shared-Disk :
───────────────────────────────
Shared-Disk : Performance plafonne à cause du storage bottleneck
Shared-Nothing : Performance scale linéairement jusqu'aux limites réseau

         Performance (ops/sec)
              │
              │                    ╱ Shared-Nothing
              │                  ╱
              │                ╱
  1,000,000 ──┤              ╱
              │            ╱
              │          ╱
    500,000 ──┤        ╱    ┌─────── Shared-Disk (plateau)
              │      ╱      │
              │    ╱────────┘
              │  ╱
              └──────────────────────────► Nombre de nœuds
                 1  10  20  30  40  50
```

### 2. Isolation complète des pannes

```
Défaillance d'un nœud = Impact localisé uniquement
═══════════════════════════════════════════════════

État nominal (3 nœuds) :
────────────────────────
Node A : Slots 0-5460      → 33.3% des données  ✓ UP
Node B : Slots 5461-10922  → 33.3% des données  ✓ UP
Node C : Slots 10923-16383 → 33.4% des données  ✓ UP

Capacité disponible : 100%
Ops/sec disponibles : 100%


Node B tombe en panne :
───────────────────────
Node A : Slots 0-5460      → 33.3% des données  ✓ UP
Node B : Slots 5461-10922  → 33.3% des données  ❌ DOWN
Node C : Slots 10923-16383 → 33.4% des données  ✓ UP

Impact :
├─ Capacité disponible : 66.7% (seules données de A et C)
├─ Ops/sec disponibles : 66.7%
├─ Clés slots 5461-10922 : INACCESSIBLES
└─ Clés autres slots : ACCESSIBLES sans dégradation

Si replicas configurées :
└─> Replica B1 promeut → failover automatique
    └─> Capacité : 100% restaurée en ~15-30 secondes


Contraste avec Shared-Disk :
─────────────────────────────
Si disque partagé tombe :
└─> 100% des données INACCESSIBLES
    └─> Tous les nœuds impactés
        └─> Cluster complètement DOWN

Shared-Nothing = Isolation des pannes
```

### 3. Absence de point de contention unique

```
┌─────────────────────────────────────────────────────────────┐
│          Comparaison : Contention et Bottlenecks            │
└─────────────────────────────────────────────────────────────┘

Architecture avec Proxy Central :
═════════════════════════════════

    Clients (1000+)
         │
         │  Toutes les requêtes passent par le proxy
         │
         ▼
    ┌─────────┐ ◄─── BOTTLENECK (CPU + Réseau)
    │  Proxy  │
    └────┬────┘
         │
    ┌────┼────┐
    │    │    │
    ▼    ▼    ▼
  Node  Node  Node
    A     B     C

Limites :
├─ Proxy = SPOF
├─ Saturation du proxy à forte charge
├─ Latence ajoutée (hop supplémentaire)
└─> Scaling limité par capacité du proxy


Redis Cluster (Shared-Nothing) :
═════════════════════════════════

    Clients (1000+)
         │
    ┌────┼────┐  Distribution intelligente
    │    │    │  (Client-side routing)
    ▼    ▼    ▼
  ┌────┐ ┌────┐ ┌────┐
  │Node│ │Node│ │Node│
  │ A  │ │ B  │ │ C  │
  └────┘ └────┘ └────┘

Avantages :
├─ Pas de SPOF
├─ Charge distribuée uniformément
├─ Latence minimale (1 hop direct)
├─ Chaque nœud = entry point valide
└─> Scaling illimité par ajout de nœuds
```

### 4. Coût linéaire et hardware commodity

```
Modèle économique : Shared-Nothing
═══════════════════════════════════

Configuration Shared-Disk (pour 600 GB RAM) :
──────────────────────────────────────────────
• 3x Serveurs haute-performance : 180,000 €
• 1x SAN enterprise (FC/iSCSI) : 150,000 €
• Switch SAN : 50,000 €
• Total : ~380,000 € + maintenance coûteuse

Configuration Redis Cluster (pour 600 GB RAM) :
────────────────────────────────────────────────
• 10x Serveurs commodity (64GB RAM) : 50,000 €
• Switch Ethernet 10Gb : 5,000 €
• Total : ~55,000 € (7x moins cher !)

Scaling incrémental :
─────────────────────
Besoin de +20% capacité ?
└─> Shared-Disk : Upgrade SAN (coût ++)
└─> Shared-Nothing : +2 nœuds (coût +)

Résilience :
────────────
Shared-Disk : Panne SAN = disaster
Shared-Nothing : Panne d'un nœud = impact 10%
```

## Trade-offs et limitations

### 1. Complexité opérationnelle accrue

```
┌─────────────────────────────────────────────────────────────┐
│    Complexité : Shared-Nothing vs Single Instance           │
└─────────────────────────────────────────────────────────────┘

Single Redis Instance (Simple) :
════════════════════════════════
• 1 processus à monitorer
• 1 fichier de config
• 1 dump RDB à backuper
• Latence : O(1) - accès direct
• Troubleshooting : Logs centralisés


Redis Cluster (Complexe) :
═══════════════════════════
• N processus à monitorer (ex: 6 nœuds)
• N fichiers de config à synchroniser
• N dumps RDB + 1 nodes.conf par nœud
• Latence : O(1) + overhead réseau + redirections
• Troubleshooting : Logs distribués, correlation nécessaire
• Resharding manuel ou automatisé
• Failover à tester et valider
• Gossip protocol à comprendre et monitorer

Compromis :
───────────
Scaling horizontal ⇔ Complexité opérationnelle
```

### 2. Opérations multi-clés limitées

Liste des limitations dues à l'absence de coordination globale :

```
OPÉRATIONS NON SUPPORTÉES (ou limitées) :
══════════════════════════════════════════

1. Transactions multi-slots :
   ❌ MULTI/EXEC sur clés de slots différents

2. Commandes multi-clés :
   ❌ MGET key1 key2 key3  (si slots différents)
   ❌ MSET key1 val1 key2 val2  (si slots différents)
   ❌ SUNION set1 set2 set3  (si slots différents)
   ❌ SDIFF set1 set2  (si slots différents)
   ❌ ZUNIONSTORE dest key1 key2  (si slots différents)

3. Scripts Lua :
   ❌ Accès à des clés de slots différents
   ⚠️  Toutes les clés doivent être passées en argument
       (pour vérification de co-localisation)

4. Rename entre slots :
   ❌ RENAME key1 key2  (si hash(key1) ≠ hash(key2))

5. Scan global :
   ⚠️  SCAN nécessite N appels (1 par nœud)

6. SELECT database :
   ❌ Pas de support multi-database (toujours DB 0)


SOLUTIONS DE CONTOURNEMENT :
═════════════════════════════

1. Hash Tags pour co-localisation forcée :
   ✓ {user:1000}:profile
   ✓ {user:1000}:friends
   └─> Garantit même slot

2. Restructuration du modèle de données :
   Au lieu de : user:1000, user:2000, user:3000
   Utiliser : {shard:0}:users (hash contenant tous les users)

3. Application-level aggregation :
   Implémenter l'agrégation côté client si multi-slots requis

4. Accepter la cohérence éventuelle :
   Plusieurs opérations séparées au lieu d'une transaction atomique
```

### 3. Overhead du réseau

```
Comparaison des latences :
══════════════════════════

Single Instance (In-Memory) :
─────────────────────────────
GET key → Latence : ~0.1 ms (access RAM local)


Redis Cluster (même datacenter) :
──────────────────────────────────
GET key → Latence : ~0.5-1 ms
├─ Calcul du slot : ~0.01 ms
├─ Routing vers bon nœud : 0 ms (si client intelligent)
├─ Network RTT : ~0.3-0.5 ms
└─ Access RAM : ~0.1 ms

Overhead réseau : +0.4-0.9 ms (~5-10x)


Redis Cluster (inter-datacenter) :
───────────────────────────────────
GET key → Latence : ~50-200 ms
└─> Network RTT domine (géographie)


MITIGATION :
════════════
• Utiliser clients intelligents (pas de redirections)
• Pipelining pour amortir le RTT
• Co-localiser clients et cluster (même datacenter/VPC)
• Caching côté client pour hot keys
```

### 4. Pas de cohérence forte garantie

```
Fenêtre de vulnérabilité :
══════════════════════════

Timeline d'une écriture avec réplication asynchrone :

T0 : Client écrit sur Master A
     SET user:1000 "John Doe"  ✓
     │
     │ Acknowledgement immédiat au client
     ▼

T0+1ms : Réplication vers Replica A1 (asynchrone)
         │
         │ Donnée en transit sur le réseau
         │
         ▼

T0+2ms : Donnée arrive sur Replica A1  ✓

Mais si Master A crash entre T0 et T0+2ms :
────────────────────────────────────────────
└─> Donnée perdue (pas encore répliquée)
    └─> Failover vers A1 → Donnée absente


GARANTIE DE COHÉRENCE :
═══════════════════════
Redis Cluster offre : "At-most-once delivery"
├─> Chaque écriture confirmée au client...
├─> ...PEUT être perdue lors d'un failover
└─> Cohérence ÉVENTUELLE, pas FORTE


MITIGATION (pour écritures critiques) :
════════════════════════════════════════
WAIT numreplicas timeout
└─> Force attente de la réplication avant ACK

Exemple :
─────────
SET critical:data "important"
WAIT 1 5000  ← Attendre au moins 1 replica (timeout 5s)
└─> Réduit la fenêtre de vulnérabilité
    └─> Mais augmente la latence d'écriture
```

## Gestion de la mémoire distribuée

### Stratégies d'allocation par nœud

```
Chaque nœud gère sa mémoire indépendamment :
════════════════════════════════════════════

Configuration par nœud (redis.conf) :
─────────────────────────────────────
maxmemory 60gb                    # Limite RAM
maxmemory-policy allkeys-lru      # Politique d'éviction

Scénario avec 3 nœuds (64 GB RAM each) :
─────────────────────────────────────────

Node A : maxmemory 60gb
├─ Utilisé : 58 GB (97%)  ← Proche de la limite
├─ Politique : allkeys-lru activée
└─> Évictions locales sur Node A uniquement

Node B : maxmemory 60gb
├─ Utilisé : 45 GB (75%)  ← Beaucoup d'espace libre
├─ Politique : allkeys-lru en attente
└─> Pas d'évictions

Node C : maxmemory 60gb
├─ Utilisé : 52 GB (87%)
├─ Politique : allkeys-lru en attente
└─> Pas d'évictions


Problématique : Hot Spots
══════════════════════════
Si un slot contient beaucoup plus de données :
└─> Le nœud responsable atteint maxmemory avant les autres
    └─> Évictions asymétriques
        └─> Distribution inégale de la capacité utilisée
```

### Monitoring de la mémoire distribuée

```bash
# Script de monitoring agrégé pour le cluster

#!/bin/bash
# cluster-memory-monitor.sh

NODES=(
    "192.168.1.10:6379"
    "192.168.1.11:6379"
    "192.168.1.12:6379"
)

echo "=== Cluster Memory Report ==="
echo ""

total_used=0
total_max=0

for node in "${NODES[@]}"; do
    IFS=':' read -r host port <<< "$node"

    used=$(redis-cli -h $host -p $port INFO memory | grep "used_memory:" | cut -d: -f2 | tr -d '\r')
    max=$(redis-cli -h $host -p $port CONFIG GET maxmemory | tail -1)

    used_gb=$(echo "scale=2; $used / 1024 / 1024 / 1024" | bc)
    max_gb=$(echo "scale=2; $max / 1024 / 1024 / 1024" | bc)
    usage=$(echo "scale=1; $used * 100 / $max" | bc)

    echo "Node $node:"
    echo "  Used: ${used_gb} GB / ${max_gb} GB (${usage}%)"

    total_used=$(echo "$total_used + $used" | bc)
    total_max=$(echo "$total_max + $max" | bc)
done

total_used_gb=$(echo "scale=2; $total_used / 1024 / 1024 / 1024" | bc)
total_max_gb=$(echo "scale=2; $total_max / 1024 / 1024 / 1024" | bc)
total_usage=$(echo "scale=1; $total_used * 100 / $total_max" | bc)

echo ""
echo "=== Cluster Totals ==="
echo "Total Used: ${total_used_gb} GB / ${total_max_gb} GB (${total_usage}%)"
```

### Rééquilibrage de la charge mémoire

Procédure pour corriger une distribution inégale :

```bash
# 1. Identifier les nœuds surchargés
redis-cli --cluster check 192.168.1.10:6379

# Sortie typique :
# Node A: 58GB (97%) ← Surchargé
# Node B: 45GB (75%)
# Node C: 52GB (87%)

# 2. Analyser la distribution des clés par slot
for node in 192.168.1.10 192.168.1.11 192.168.1.12; do
    echo "=== $node ==="
    redis-cli -h $node DBSIZE
    redis-cli -h $node INFO keyspace
done

# 3. Décision de resharding
# Si Node A est surchargé, déplacer des slots vers Node B

redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-from <node-a-id> \
    --cluster-to <node-b-id> \
    --cluster-slots 500 \
    --cluster-yes

# 4. Monitoring post-resharding
watch -n 10 './cluster-memory-monitor.sh'

# 5. Ajuster maxmemory si nécessaire
redis-cli -h 192.168.1.10 CONFIG SET maxmemory 65gb
```

## Procédures de maintenance avancées

### 1. Ajout d'un nœud avec rééquilibrage automatique

```bash
# Procédure complète d'ajout de nœud

# Étape 0 : Vérifier l'état actuel du cluster
redis-cli --cluster check 192.168.1.10:6379
redis-cli --cluster info 192.168.1.10:6379

# Sortie :
# 192.168.1.10:6379 -> 10240 keys | 5461 slots
# 192.168.1.11:6379 -> 10280 keys | 5462 slots
# 192.168.1.12:6379 -> 10244 keys | 5461 slots

# Étape 1 : Démarrer le nouveau nœud (Node D)
ssh 192.168.1.13
sudo systemctl start redis@6379

# Vérifier qu'il est seul (pas encore dans le cluster)
redis-cli -h 192.168.1.13 -p 6379 CLUSTER INFO
# cluster_state:fail (normal, pas encore de cluster)
# cluster_known_nodes:1 (lui-même seulement)

# Étape 2 : Ajouter le nœud au cluster (MEET)
redis-cli --cluster add-node 192.168.1.13:6379 192.168.1.10:6379

# Vérification : Node D connaît maintenant tous les autres
redis-cli -h 192.168.1.13 CLUSTER NODES
# Devrait lister les 4 nœuds (A, B, C, D)

# État : D est dans le cluster mais possède 0 slots

# Étape 3 : Rééquilibrage automatique
redis-cli --cluster rebalance 192.168.1.10:6379 \
    --cluster-threshold 1 \
    --cluster-use-empty-masters

# Explications des paramètres :
# --cluster-threshold 1 : Accepter déséquilibre de 1% max
# --cluster-use-empty-masters : Utiliser Node D (qui a 0 slots)

# Résultat attendu :
# 192.168.1.10:6379 -> 4096 slots (25%)
# 192.168.1.11:6379 -> 4096 slots (25%)
# 192.168.1.12:6379 -> 4096 slots (25%)
# 192.168.1.13:6379 -> 4096 slots (25%) ← Nouveau

# Étape 4 : Ajouter une replica pour Node D
redis-cli --cluster add-node 192.168.1.14:6379 192.168.1.10:6379 \
    --cluster-slave \
    --cluster-master-id <node-d-id>

# Étape 5 : Vérification finale
redis-cli --cluster check 192.168.1.10:6379
# [OK] All 16384 slots covered.
# 4 masters, 4 replicas

# Étape 6 : Monitoring post-ajout (24-48h)
# Vérifier :
# - Distribution uniforme de la charge
# - Pas de redirections excessives
# - Réplication stable
# - Gossip protocol healthy
```

### 2. Retrait d'un nœud avec redistribution

```bash
# Procédure sécurisée de retrait de nœud

# Étape 0 : Identifier le nœud à retirer
redis-cli --cluster check 192.168.1.10:6379

# Décision : Retirer Node C (192.168.1.12) et sa replica

# Étape 1 : Si le nœud a une replica, la supprimer d'abord
NODE_C_REPLICA_ID=$(redis-cli -h 192.168.1.15 CLUSTER MYID)
redis-cli --cluster del-node 192.168.1.10:6379 $NODE_C_REPLICA_ID

# Étape 2 : Déplacer TOUS les slots de Node C vers les autres nœuds
NODE_C_ID=$(redis-cli -h 192.168.1.12 CLUSTER MYID)

# Option A : Répartir équitablement sur tous les autres nœuds
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-from $NODE_C_ID \
    --cluster-to all \
    --cluster-slots 5461 \
    --cluster-yes \
    --cluster-timeout 60000 \
    --cluster-pipeline 10

# Option B : Cibler un nœud spécifique
# redis-cli --cluster reshard 192.168.1.10:6379 \
#     --cluster-from $NODE_C_ID \
#     --cluster-to <node-a-id> \
#     --cluster-slots 5461 \
#     --cluster-yes

# Étape 3 : Vérifier que Node C ne possède plus aucun slot
redis-cli -h 192.168.1.12 CLUSTER INFO | grep cluster_slots_assigned
# Doit retourner : cluster_slots_assigned:0

redis-cli --cluster check 192.168.1.10:6379
# 192.168.1.12:6379 -> 0 keys | 0 slots ← Prêt à être retiré

# Étape 4 : Retirer Node C du cluster
redis-cli --cluster del-node 192.168.1.10:6379 $NODE_C_ID

# Étape 5 : Arrêter le service Redis sur Node C
ssh 192.168.1.12
sudo systemctl stop redis@6379
sudo systemctl disable redis@6379

# Optionnel : Nettoyer les données
# sudo rm -rf /var/lib/redis/*

# Étape 6 : Vérification finale
redis-cli --cluster check 192.168.1.10:6379
# [OK] All 16384 slots covered.
# 3 masters, 3 replicas (Node C absent)

# Étape 7 : Mettre à jour la configuration côté client
# Retirer 192.168.1.12:6379 des seeds nodes
# Les clients intelligents découvriront automatiquement la nouvelle topologie
```

### 3. Maintenance avec zéro downtime

```bash
# Scénario : Mise à jour d'un nœud (OS patch, upgrade Redis, etc.)

# Principe : Basculer sur replica avant maintenance du master

# Étape 1 : Identifier le master à maintenir
TARGET_MASTER="192.168.1.11:6379"

# Étape 2 : Vérifier que le master a bien une replica
redis-cli -h 192.168.1.11 -p 6379 INFO replication
# role:master
# connected_slaves:1
# slave0:ip=192.168.1.14,port=6379,state=online

# Étape 3 : Déclencher un failover PLANIFIÉ sur la replica
# (Pas d'attente de timeout, basculement immédiat)
redis-cli -h 192.168.1.14 -p 6379 CLUSTER FAILOVER TAKEOVER

# La replica devient master SANS attendre le timeout
# L'ancien master devient replica automatiquement

# Étape 4 : Vérifier le basculement
redis-cli -h 192.168.1.14 CLUSTER NODES | grep myself
# Devrait afficher : myself,master

redis-cli -h 192.168.1.11 CLUSTER NODES | grep myself
# Devrait afficher : myself,slave

# Étape 5 : Maintenance sur l'ancien master (maintenant replica)
ssh 192.168.1.11

# Arrêter Redis
sudo systemctl stop redis@6379

# Effectuer la maintenance (OS patch, upgrade, etc.)
sudo apt update && sudo apt upgrade -y
# ou
sudo yum update -y
# ou upgrade Redis binary

# Redémarrer Redis
sudo systemctl start redis@6379

# Vérifier que la réplication reprend
redis-cli -h 192.168.1.11 INFO replication
# role:slave
# master_host:192.168.1.14
# master_link_status:up

# Étape 6 : Optionnel - Re-basculer pour restaurer la topologie initiale
# Si souhaité, faire l'inverse pour remettre 192.168.1.11 en master
redis-cli -h 192.168.1.11 CLUSTER FAILOVER TAKEOVER

# Étape 7 : Répéter pour tous les autres nœuds
# En parallèle ou séquentiellement selon politique de maintenance

# Downtime total : 0 seconde (basculement instantané)
```

### 4. Récupération après corruption du cluster

```bash
# Scénario catastrophe : Configuration cluster corrompue

# Symptômes :
# - cluster_state:fail persistant
# - Slots non assignés ou dupliqués
# - Nœuds ne se reconnaissent plus

# ATTENTION : Procédure de dernier recours

# Étape 1 : Backup complet AVANT toute intervention
for node in 192.168.1.10 192.168.1.11 192.168.1.12; do
    ssh $node "sudo cp /var/lib/redis/nodes.conf /var/lib/redis/nodes.conf.backup"
    ssh $node "sudo cp /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.backup"
done

# Étape 2 : Tentative de réparation automatique
redis-cli --cluster fix 192.168.1.10:6379 \
    --cluster-fix-with-unreachable-masters

# Si échec, passer aux étapes suivantes

# Étape 3 : Reset SOFT du cluster (préserve les données)
# À faire sur TOUS les nœuds
for node in 192.168.1.10 192.168.1.11 192.168.1.12; do
    redis-cli -h $node CLUSTER RESET SOFT
done

# Étape 4 : Recréer le cluster
redis-cli --cluster create \
    192.168.1.10:6379 \
    192.168.1.11:6379 \
    192.168.1.12:6379 \
    --cluster-replicas 0 \
    --cluster-yes

# Les données sont préservées car RESET SOFT n'a pas vidé le dataset

# Étape 5 : Ré-ajouter les replicas
redis-cli --cluster add-node 192.168.1.13:6379 192.168.1.10:6379 \
    --cluster-slave --cluster-master-id <node-a-id>

redis-cli --cluster add-node 192.168.1.14:6379 192.168.1.10:6379 \
    --cluster-slave --cluster-master-id <node-b-id>

redis-cli --cluster add-node 192.168.1.15:6379 192.168.1.10:6379 \
    --cluster-slave --cluster-master-id <node-c-id>

# Étape 6 : Vérification exhaustive
redis-cli --cluster check 192.168.1.10:6379
redis-cli --cluster info 192.168.1.10:6379

# Comparer le nombre de clés avant/après
# Vérifier qu'aucune donnée n'a été perdue
```

## Considérations pour la production

### Dimensionnement optimal

```
Règles de dimensionnement Shared-Nothing :
═══════════════════════════════════════════

1. NOMBRE DE NŒUDS
   ├─ Minimum : 3 masters (majorité = 2)
   ├─ Recommandé production : 6 nœuds (3M + 3R)
   ├─ Large scale : 10-50 nœuds
   └─ Maximum pratique : ~1000 nœuds
      (au-delà, overhead Gossip devient significatif)

2. MÉMOIRE PAR NŒUD
   ├─ Minimum : 8 GB
   ├─ Sweet spot : 16-64 GB
   ├─ Maximum : 256 GB
   └─> Au-delà de 256GB :
       ├─ Fork pour RDB/AOF devient lent
       ├─ Failover plus long (réplication volumineuse)
       └─ Considérer split en plusieurs nœuds

3. CPU PAR NŒUD
   ├─ Single-threaded (1 core pour Redis)
   ├─ Recommandé : 4-8 cores
   └─> Cores additionnels utiles pour :
       ├─ I/O threads (Redis 6+)
       ├─ Background saving (fork)
       └─ Monitoring / Logs

4. RÉSEAU
   ├─ Minimum : 1 Gbps
   ├─ Recommandé : 10 Gbps
   ├─ Latence inter-nœuds : < 1 ms (LAN) ou < 10 ms (WAN)
   └─> Bande passante critique pour :
       ├─ Gossip protocol
       ├─ Réplication master→replica
       └─ Resharding (migration de slots)

5. STOCKAGE
   ├─ Type : SSD recommandé (si persistence activée)
   ├─ Capacité : 2x la RAM (pour RDB + AOF)
   ├─ IOPS : > 10,000 IOPS
   └─> Redis est in-memory mais :
       ├─ RDB dump = burst write
       ├─ AOF = séquentiel mais continu
```

### Topologies recommandées

```
┌─────────────────────────────────────────────────────────────┐
│              Topologies Shared-Nothing Courantes            │
└─────────────────────────────────────────────────────────────┘

1. PETITE SCALE (< 100 GB, < 100k ops/sec)
   ═════════════════════════════════════════

   3 Masters + 3 Replicas (6 nœuds total)

   [M1] ─── [R1]
   [M2] ─── [R2]
   [M3] ─── [R3]

   Capacité : 3 × RAM_par_nœud
   HA : Tolère 1 panne de master


2. MOYENNE SCALE (100-500 GB, 100k-1M ops/sec)
   ════════════════════════════════════════════

   6 Masters + 6 Replicas (12 nœuds total)

   [M1]─[R1]   [M2]─[R2]   [M3]─[R3]
   [M4]─[R4]   [M5]─[R5]   [M6]─[R6]

   Capacité : 6 × RAM_par_nœud
   HA : Tolère 3 pannes simultanées (1 par master)


3. LARGE SCALE (> 500 GB, > 1M ops/sec)
   ══════════════════════════════════════

   12+ Masters avec replicas multiples

   Considérations :
   ├─ Gossip overhead : monitoring requis
   ├─ Latence réseau : critique
   └─ Compléxité opérationnelle élevée


4. MULTI-DC (Disaster Recovery)
   ══════════════════════════════

   DC1 (Primary):        DC2 (DR):
   [M1]─[R1]             [R1']
   [M2]─[R2]      ⟷     [R2']
   [M3]─[R3]             [R3']

   Réplication asynchrone inter-DC
   ⚠️  Latence élevée (10-100+ ms)
   ⚠️  Pas de failover automatique inter-DC
```

### Checklist de déploiement production

```
✅ AVANT LE DÉPLOIEMENT
   ├─ Architecture validée (nombre de nœuds, RAM, réseau)
   ├─ Cluster configuré et testé en staging
   ├─ Procédures de failover testées et documentées
   ├─ Procédures de resharding testées
   ├─ Plan de reprise d'activité (DRP) rédigé
   └─ Formation équipe ops sur Redis Cluster

✅ CONFIGURATION CLUSTER
   ├─ cluster-enabled yes
   ├─ cluster-node-timeout 15000 (ajuster selon réseau)
   ├─ cluster-replica-validity-factor 10
   ├─ cluster-require-full-coverage yes (ou no selon cas)
   └─ cluster-migration-barrier 1

✅ CONFIGURATION SYSTÈME
   ├─ Désactiver THP (Transparent Huge Pages)
   ├─ Configurer overcommit_memory = 1
   ├─ Ajuster somaxconn et tcp-backlog
   ├─ Configurer firewall (ports 6379 + 16379)
   └─ NTP synchronisé sur tous les nœuds

✅ MONITORING
   ├─ Dashboards : cluster_state, slots coverage, memory
   ├─ Alertes : nœud down, slot non couvert, mémoire haute
   ├─ Métriques Gossip : latency, message rate
   ├─ Logs centralisés (ELK, Splunk, etc.)
   └─ Health checks automatisés

✅ HAUTE DISPONIBILITÉ
   ├─ Au moins 1 replica par master
   ├─ Replicas sur différents racks/AZ
   ├─ Persistence configurée (RDB + AOF)
   ├─ Backups automatisés (quotidien minimum)
   └─ Procédure de restauration testée

✅ SÉCURITÉ
   ├─ ACLs configurées (requirepass dépréciée)
   ├─ TLS/SSL activé si sensible
   ├─ Bind sur interfaces privées uniquement
   ├─ Firewall : whitelist IP sources
   └─ Audit logging activé

✅ OPÉRATIONS
   ├─ Runbooks pour scénarios d'incident
   ├─ Contacts on-call définis
   ├─ Fenêtres de maintenance planifiées
   ├─ Processus de rollback documenté
   └─ Post-mortems après chaque incident
```

## Conclusion

L'architecture Shared-Nothing de Redis Cluster représente un choix architectural fondamental qui privilégie :

1. **Scaling horizontal** illimité sur la simplicité opérationnelle
2. **Performance distribuée** sur la cohérence forte
3. **Résilience par isolation** sur la coordination globale
4. **Coût linéaire** (commodity hardware) sur l'infrastructure spécialisée

Cette architecture impose des contraintes (opérations multi-clés limitées, cohérence éventuelle) mais offre en contrepartie des capacités de scaling et de résilience impossibles à atteindre avec des architectures traditionnelles.

La compréhension profonde de ce paradigme Shared-Nothing est essentielle pour :
- Concevoir des modèles de données adaptés (hash tags, co-localisation)
- Dimensionner correctement le cluster (nombre de nœuds, RAM par nœud)
- Opérer le cluster en production (resharding, maintenance, monitoring)
- Anticiper et gérer les cas limites (hot spots, pannes multiples)

---

**Points clés à retenir :**

- **Autonomie complète** : Chaque nœud = processus + RAM + disque indépendants
- **Pas de ressource partagée** : Communication uniquement via réseau
- **Scaling linéaire** : Capacité totale = Somme des capacités individuelles
- **Isolation des pannes** : Défaillance localisée (1/N des données)
- **Complexité opérationnelle** : Trade-off inévitable du Shared-Nothing
- **Pas de cohérence forte** : Accepter l'éventuelle cohérence
- **Maintenance par nœud** : Chaque nœud est une unité opérationnelle
- **Dimensionnement critique** : 16-64 GB RAM / nœud = sweet spot

La section suivante (11.3) détaillera les mécanismes de distribution des données via les hash slots.

⏭️ [Distribution des données : Hash Slots (0-16383)](/11-architecture-distribuee-scaling/03-distribution-donnees-hash-slots.md)
