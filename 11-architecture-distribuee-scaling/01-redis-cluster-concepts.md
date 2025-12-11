🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.1 Redis Cluster : Concepts (Sharding, Hash Slots, Gossip Protocol)

## Introduction

Redis Cluster représente la solution native de Redis pour le scaling horizontal et la haute disponibilité sans point de défaillance unique (Single Point of Failure - SPOF). Contrairement aux architectures traditionnelles basées sur un proxy central ou une configuration master-replica simple, Redis Cluster implémente une architecture décentralisée où chaque nœud est autonome et communique directement avec les autres via un protocole de gossip.

Cette section explore les trois piliers conceptuels qui fondent Redis Cluster :
1. **Le sharding** : Comment les données sont partitionnées
2. **Les hash slots** : Le mécanisme de distribution déterministe
3. **Le protocole Gossip** : Comment les nœuds maintiennent une vue cohérente du cluster

## Qu'est-ce que Redis Cluster ?

### Définition et objectifs

Redis Cluster est une implémentation distribuée de Redis qui permet :

```
┌─────────────────────────────────────────────────────────────┐
│              Objectifs de Redis Cluster                     │
├─────────────────────────────────────────────────────────────┤
│ ✓ Partitionnement automatique des données (Sharding)        │
│ ✓ Scaling horizontal : Ajouter de la capacité en ajoutant   │
│   des nœuds                                                 │
│ ✓ Haute disponibilité : Failover automatique via replicas   │
│ ✓ Performance linéaire : O(n) avec n nœuds                  │
│ ✓ Architecture décentralisée : Pas de SPOF                  │
│ ✓ Tolérance aux pannes : Continue à fonctionner avec        │
│   défaillance partielle                                     │
└─────────────────────────────────────────────────────────────┘
```

### Architecture conceptuelle

```
                    Redis Cluster Architecture

┌────────────────────────────────────────────────────────────┐
│                     Application Layer                      │
│              (Smart Client - Cluster Aware)                │
└────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │   Master A   │ │   Master B   │ │   Master C   │
    │              │ │              │ │              │
    │ Slots:       │ │ Slots:       │ │ Slots:       │
    │ 0-5460       │ │ 5461-10922   │ │ 10923-16383  │
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
           │ Replication    │ Replication    │ Replication
           ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Replica A1  │ │  Replica B1  │ │  Replica C1  │
    │              │ │              │ │              │
    │ (Failover)   │ │ (Failover)   │ │ (Failover)   │
    └──────────────┘ └──────────────┘ └──────────────┘

    ◄─────────────────────────────────────────────────►
              Gossip Protocol (Bus Cluster)
         Tous les nœuds communiquent entre eux
```

### Caractéristiques fondamentales

**Décentralisation totale :**
- Aucun coordinateur central
- Chaque nœud connaît l'état complet du cluster
- Détection de pannes par consensus distribué
- Pas de proxy requis (client-side routing)

**Garanties de cohérence :**
- Pas de garantie de cohérence forte (Strong Consistency)
- Cohérence éventuelle (Eventual Consistency)
- Possibilité de perte de données lors de partitionnement réseau
- Mode synchrone optionnel avec `WAIT` pour les écritures critiques

## Le Sharding : Partitionnement horizontal des données

### Principe du sharding

Le sharding (ou partitionnement) consiste à diviser un dataset en sous-ensembles distribués sur plusieurs serveurs. Chaque serveur (ou nœud) ne stocke qu'une fraction des données totales.

```
                    Données complètes (100%)
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
    ┌────────┐          ┌────────┐          ┌────────┐
    │ Shard 1│          │ Shard 2│          │ Shard 3│
    │  33%   │          │  33%   │          │  34%   │
    │ Nœud A │          │ Nœud B │          │ Nœud C │
    └────────┘          └────────┘          └────────┘

Avantages :
├─ Capacité totale = Somme des capacités individuelles
├─ Performance = Agrégation des performances des nœuds
└─ Parallélisation des lectures/écritures
```

### Types de sharding

Redis Cluster utilise un **sharding basé sur une fonction de hachage** :

```
┌─────────────────────────────────────────────────────────────┐
│              Types de Sharding (Comparaison)                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. RANGE SHARDING                                           │
│    user:0001-5000  → Shard 1                                │
│    user:5001-10000 → Shard 2                                │
│    └─> Problème : Risque de hot spots (répartition inégale) │
│                                                             │
│ 2. HASH SHARDING (Redis Cluster) ✓                          │
│    CRC16(key) mod 16384 → Hash Slot → Shard                 │
│    └─> Avantage : Distribution uniforme et prévisible       │
│                                                             │
│ 3. DIRECTORY-BASED SHARDING                                 │
│    Lookup table : key → Shard                               │
│    └─> Problème : Lookup table devient un bottleneck        │
│                                                             │
│ 4. GEO SHARDING                                             │
│    Données par région géographique                          │
│    └─> Cas spécifique, non implémenté nativement            │
└─────────────────────────────────────────────────────────────┘
```

### Avantages du sharding dans Redis Cluster

**1. Scaling horizontal illimité (théorique)**
```
1 nœud   = 256 GB RAM max
10 nœuds = 2.5 TB RAM disponible
100 nœuds = 25 TB RAM disponible

Formule : Capacité totale = N × Capacité_par_nœud
```

**2. Performance linéaire**
```
Ops/sec = N × Ops_par_nœud

Exemple :
1 nœud  → 100,000 ops/sec
3 nœuds → 300,000 ops/sec
10 nœuds → 1,000,000 ops/sec

Note : En pratique, légère overhead due au routing et gossip
```

**3. Isolation des pannes**
```
Si un nœud tombe :
└─> Seules les clés de ce nœud sont impactées
    (1/N des données avec N nœuds)

└─> Les autres nœuds continuent à servir leurs données

└─> Si replica disponible : Failover automatique
```

## Hash Slots : Le mécanisme de distribution

### Concept des hash slots

Redis Cluster divise l'espace de clés en **16384 hash slots** numérotés de 0 à 16383. Chaque clé est assignée à un slot via une fonction de hachage déterministe.

```
┌─────────────────────────────────────────────────────────────┐
│           Architecture des Hash Slots                       │
└─────────────────────────────────────────────────────────────┘

           Espace complet : 16384 slots (0-16383)
                              │
                              │ Fonction : CRC16(key) & 16383
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   Slot 0-5460          Slot 5461-10922       Slot 10923-16383
        │                     │                     │
        │                     │                     │
        ▼                     ▼                     ▼
    Master A              Master B              Master C
    (Nœud 1)             (Nœud 2)              (Nœud 3)
```

### Calcul du hash slot

```
Pour une clé donnée, l'algorithme est :

HASH_SLOT = CRC16(key) mod 16384

Exemples :
─────────
user:1000        → CRC16("user:1000") & 16383 = 5798  → Nœud A
session:abc123   → CRC16("session:abc123") & 16383 = 12456 → Nœud C
product:9999     → CRC16("product:9999") & 16383 = 7834 → Nœud B

Propriété importante : DÉTERMINISTE
└─> La même clé sera toujours mappée au même slot
    peu importe le nœud qui effectue le calcul
```

### Distribution des slots aux nœuds

Lors de la création d'un cluster, les 16384 slots sont distribués équitablement :

```
Configuration initiale (3 masters) :
────────────────────────────────────

Master A : 192.168.1.10:6379
├─ Slots assignés : 0-5460 (5461 slots)
└─ Pourcentage : ~33.3%

Master B : 192.168.1.11:6379
├─ Slots assignés : 5461-10922 (5462 slots)
└─ Pourcentage : ~33.3%

Master C : 192.168.1.12:6379
├─ Slots assignés : 10923-16383 (5461 slots)
└─ Pourcentage : ~33.4%

Total : 16384 slots = 100% du keyspace
```

### Avantages des hash slots

**1. Prévisibilité**
```
Contrairement au consistent hashing :
└─> Pas de virtual nodes
└─> Pas de redistribution massive lors d'ajout/suppression
└─> Calcul instantané du slot d'une clé
```

**2. Granularité fine pour le resharding**
```
16384 slots permettent :
└─> Déplacement progressif lors d'ajout de nœuds
└─> Migration par petits lots (ex: 100 slots à la fois)
└─> Contrôle précis de la répartition
```

**3. Taille optimale pour le gossip**
```
16384 slots = 2048 octets (2 KB) en bitmap
└─> Chaque nœud maintient un bitmap de tous les slots
└─> Transmission rapide via le bus cluster
└─> Faible overhead mémoire
```

### Hash tags : Contrôle de la localisation

Pour forcer plusieurs clés à être dans le même slot (nécessaire pour les transactions et opérations multi-clés) :

```
Syntaxe : {tag}

Exemples :
──────────
{user:1000}:profile    → Hash sur "user:1000"
{user:1000}:friends    → Hash sur "user:1000"  ← Même slot !
{user:1000}:sessions   → Hash sur "user:1000"  ← Même slot !

Cela permet :
└─> MGET {user:1000}:profile {user:1000}:friends
└─> MULTI/EXEC sur plusieurs clés liées
└─> Scripts Lua accédant à plusieurs clés
```

## Le Gossip Protocol : Communication décentralisée

### Principe du protocole Gossip

Le protocole Gossip (ou "epidemic protocol") est un mécanisme de communication peer-to-peer où chaque nœud :
1. Maintient une vue de l'état du cluster
2. Échange régulièrement des informations avec d'autres nœuds aléatoires
3. Propage les changements de manière virale

```
┌─────────────────────────────────────────────────────────────┐
│                  Gossip Protocol Flow                       │
└─────────────────────────────────────────────────────────────┘

    Nœud A                  Nœud B                  Nœud C
       │                       │                       │
       │                       │                       │
  ┌────▼─────┐            ┌────▼─────┐            ┌────▼─────┐
  │  État    │            │  État    │            │  État    │
  │ Cluster  │            │ Cluster  │            │ Cluster  │
  └────┬─────┘            └────┬─────┘            └────┬─────┘
       │                       │                       │
       │  Heartbeat (PING)     │                       │
       ├──────────────────────►│                       │
       │                       │                       │
       │  Response (PONG)      │                       │
       ◄───────────────────────┤                       │
       │                       │                       │
       │                       │  Heartbeat (PING)     │
       │                       ├──────────────────────►│
       │                       │                       │
       │                       │  Response (PONG)      │
       │                       ◄───────────────────────┤
       │                       │                       │
       │  Update propagation   │                       │
       ├──────────────────────►│                       │
       │                       ├──────────────────────►│
       │                       │                       │

Chaque nœud "gossipe" avec quelques nœuds aléatoires
└─> Convergence exponentielle : O(log N) rounds
```

### Architecture du Bus Cluster

Redis Cluster utilise un second port (port + 10000) pour le bus cluster :

```
Configuration typique :
──────────────────────

Port 6379 : Client connections (Redis Protocol)
├─ Gère les requêtes clients
├─ GET, SET, MGET, etc.
└─ Communication synchrone

Port 16379 : Cluster Bus (Binary Protocol)
├─ Gossip entre nœuds
├─ Détection de pannes
├─ Resharding
├─ Failover
└─ Communication asynchrone


┌──────────────────────────────────────────────────────┐
│         Nœud Redis A (192.168.1.10)                  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Port 6379  ◄─── Clients (Applications)              │
│     │                                                │
│     └─► Redis Core (Data Operations)                 │
│                                                      │
│  Port 16379 ◄─── Cluster Bus                         │
│     │                                                │
│     └─► Gossip Handler                               │
│         ├─ Heartbeats                                │
│         ├─ Configuration sync                        │
│         └─ Failure detection                         │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Messages échangés via Gossip

```
┌─────────────────────────────────────────────────────────────┐
│            Types de messages Gossip                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PING                                                        │
│ └─> Heartbeat régulier envoyé à un nœud aléatoire           │
│     Contient : Node ID, slots, epoch, état                  │
│                                                             │
│ PONG                                                        │
│ └─> Réponse au PING                                         │
│     Contient : Confirmation de réception, état local        │
│                                                             │
│ MEET                                                        │
│ └─> Intégration d'un nouveau nœud au cluster                │
│     Envoyé via : CLUSTER MEET <ip> <port>                   │
│                                                             │
│ FAIL                                                        │
│ └─> Déclaration qu'un nœud est en panne                     │
│     Propagé à tous après consensus                          │
│                                                             │
│ PUBLISH                                                     │
│ └─> Propagation d'un message pub/sub                        │
│                                                             │
│ UPDATE                                                      │
│ └─> Changement de configuration (resharding, failover)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Fréquence et temporisation du Gossip

```bash
# Paramètres par défaut du protocole Gossip

cluster-node-timeout 15000    # 15 secondes
├─> Délai avant de considérer un nœud comme potentiellement en panne
└─> Les nœuds envoient des PING toutes les (timeout / 2) = 7.5 sec

cluster-replica-validity-factor 10
├─> Facteur de multiplication du timeout pour les replicas
└─> Replica valide si dernier contact < timeout × factor

cluster-migration-barrier 1
├─> Nombre minimum de replicas à garder avant migration
└─> Empêche un master de perdre toutes ses replicas

# Exemple de calcul :
# Cluster de 10 nœuds, timeout = 15s
# Chaque nœud envoie ~1 PING toutes les 7.5s
# Total : 10 nœuds × 1 PING/7.5s = ~1.3 messages/sec
# Plus les PONG, MEET, UPDATE → ~10-20 messages/sec pour le cluster
```

### Détection de pannes avec Gossip

```
┌─────────────────────────────────────────────────────────────┐
│          Processus de détection de panne                    │
└─────────────────────────────────────────────────────────────┘

Étape 1 : PFAIL (Probable Fail)
────────────────────────────────
Un nœud A ne reçoit plus de réponse du nœud C
└─> Marque C comme PFAIL localement
    (n'affecte pas encore le cluster)

    Nœud A: "C est peut-être down"
            (opinion subjective)


Étape 2 : Propagation via Gossip
─────────────────────────────────
A informe les autres nœuds de son opinion sur C

    A → B : "Je pense que C est PFAIL"
    A → D : "Je pense que C est PFAIL"

    B et D font leur propre vérification


Étape 3 : FAIL (Consensus atteint)
───────────────────────────────────
Si MAJORITÉ des masters marquent C comme PFAIL :

    Condition : (N/2 + 1) masters marquent C comme PFAIL

    └─> C est marqué FAIL par tout le cluster
        (changement d'état global)


Étape 4 : Propagation FAIL
───────────────────────────
Le message FAIL est propagé immédiatement à tous les nœuds

    A → BROADCAST : "C est FAIL (confirmé)"

    └─> Tous les nœuds marquent C comme FAIL
        └─> Déclenchement du failover si C est master


Étape 5 : Failover (si master)
───────────────────────────────
Si C était un master avec replicas :

    Replica C1 s'élit comme nouveau master
    ├─ Prend possession des slots de C
    ├─ Propage le changement via Gossip
    └─> Cluster opérationnel après ~15-30 secondes
```

### Vue cohérente du cluster

Chaque nœud maintient une structure de données complète du cluster :

```
Structure interne (simplifiée) :
─────────────────────────────────

struct clusterNode {
    char name[40];              // Node ID (SHA1)
    char ip[46];                // Adresse IP
    int port;                   // Port client
    int cport;                  // Port cluster
    int flags;                  // MASTER, SLAVE, PFAIL, FAIL
    mstime_t ping_sent;         // Timestamp dernier PING
    mstime_t pong_received;     // Timestamp dernier PONG
    unsigned char slots[16384/8]; // Bitmap des slots (2KB)
    int numslots;               // Nombre de slots
    struct clusterNode *slaveof; // Pointeur vers master (si replica)
    int numslaves;              // Nombre de replicas (si master)
    struct clusterNode **slaves; // Liste des replicas
    ...
};

Configuration complète du cluster :
────────────────────────────────────
struct clusterState {
    clusterNode *nodes[CLUSTER_SLOTS]; // Mapping slot → node
    clusterNode *myself;                // Ce nœud
    uint64_t currentEpoch;              // Epoch actuel
    dict *nodes_black_list;             // Nœuds exclus temporairement
    ...
};
```

### Convergence et cohérence éventuelle

```
Propriétés du Gossip dans Redis Cluster :
──────────────────────────────────────────

✓ Convergence rapide : O(log N) rounds
├─> Avec 10 nœuds : ~3-4 rounds
├─> Avec 100 nœuds : ~7-8 rounds
└─> Avec 1000 nœuds : ~10-11 rounds

✓ Résilience aux partitions
├─> Le cluster continue à fonctionner si majorité accessible
└─> Détection automatique du split-brain

✗ Pas de cohérence forte
├─> Fenêtre de propagation (quelques secondes)
├─> Possibilité de vues divergentes temporaires
└─> Résolution via epoch (vecteur d'horloge)

✓ Overhead réseau faible
├─> Messages compacts (2KB par heartbeat)
├─> Fréquence contrôlée (timeout/2)
└─> Pas de broadcast (seulement gossip)
```

## Interaction entre Sharding, Slots et Gossip

### Scénario complet : Ajout d'un nœud

```
┌─────────────────────────────────────────────────────────────┐
│    Ajout d'un nœud : Orchestration complète                 │
└─────────────────────────────────────────────────────────────┘

Étape 1 : MEET (Gossip)
────────────────────────
redis-cli --cluster add-node 192.168.1.13:6379 192.168.1.10:6379

└─> Nœud A envoie MEET à Nœud D
    └─> D rejoint le cluster
        └─> Propagation via Gossip : tous les nœuds connaissent D

État : D est dans le cluster mais n'a aucun slot


Étape 2 : Resharding (Slots)
─────────────────────────────
redis-cli --cluster reshard 192.168.1.10:6379

└─> Déplacement de slots de A, B, C vers D
    ├─ A transfert slots 0-1365 à D
    ├─ B transfert slots 5461-6826 à D
    └─ C transfert slots 10923-12288 à D

État : D possède maintenant ~4096 slots (25% du keyspace)


Étape 3 : Propagation (Gossip)
───────────────────────────────
└─> Chaque changement de slot ownership est propagé
    ├─> A → Gossip : "J'ai transféré slots 0-1365 à D"
    ├─> B → Gossip : "J'ai transféré slots 5461-6826 à D"
    └─> C → Gossip : "J'ai transféré slots 10923-12288 à D"

└─> Tous les nœuds mettent à jour leur table de routing

État : Cluster cohérent, D est opérationnel


Étape 4 : Réplication (Slots + Gossip)
───────────────────────────────────────
redis-cli --cluster add-node 192.168.1.14:6379 192.168.1.10:6379 \
    --cluster-slave --cluster-master-id <D-node-id>

└─> E rejoint comme replica de D
    └─> Gossip propage : "E est replica de D"
        └─> E réplique les slots de D en arrière-plan

État final : D (master avec 4096 slots) + E (replica)
```

### Scénario : Requête client avec redirection

```
┌─────────────────────────────────────────────────────────────┐
│         Client Request Flow avec Slot Routing               │
└─────────────────────────────────────────────────────────────┘

Client exécute : GET user:5000

Étape 1 : Calcul du slot (Client-side ou Server-side)
──────────────────────────────────────────────────────
CRC16("user:5000") & 16383 = 8754

Étape 2 : Client contacte un nœud aléatoire (ex: Nœud A)
─────────────────────────────────────────────────────────
Client → Nœud A : GET user:5000

Étape 3 : Nœud A vérifie s'il possède le slot 8754
───────────────────────────────────────────────────
Nœud A consulte sa table :
├─ Slot 8754 appartient à Nœud B
└─> Nœud A ne peut pas servir cette requête

Étape 4 : Redirection -MOVED
─────────────────────────────
Nœud A → Client : -MOVED 8754 192.168.1.11:6379

Étape 5 : Client suit la redirection
─────────────────────────────────────
Client → Nœud B : GET user:5000
Nœud B → Client : "John Doe"

Étape 6 : Client met en cache le mapping
─────────────────────────────────────────
Client mémorise : Slot 8754 → Nœud B
└─> Prochaine requête sur user:5000 ira directement à B


Optimisation (Smart Client) :
──────────────────────────────
Un client intelligent télécharge la table complète de slots :

CLUSTER SLOTS → Récupère tous les mappings
└─> Client route directement vers le bon nœud
    └─> Pas de redirection = latence réduite
```

### Scénario : Failover automatique

```
┌─────────────────────────────────────────────────────────────┐
│              Automatic Failover Process                     │
└─────────────────────────────────────────────────────────────┘

État initial :
──────────────
Master B (slots 5461-10922) + Replica B1

Timeline :
──────────

T0 : Master B crash
     │
     │  Gossip: Tous les nœuds envoient PING à B
     ▼

T0+7.5s : Premiers timeouts
          │
          │  Plusieurs nœuds marquent B comme PFAIL
          ▼

T0+15s : Consensus PFAIL → FAIL
         │
         │  Majorité des masters confirment : B est FAIL
         │  Message FAIL propagé instantanément
         ▼

T0+16s : Replica B1 détecte la panne du master
         │
         │  B1 démarre l'élection de failover
         │  B1 : "Je candidate pour devenir master"
         ▼

T0+17s : Élection
         │
         │  B1 demande des votes aux autres masters
         │  Condition : Majorité des masters vote pour B1
         ▼

T0+18s : B1 élu nouveau master
         │
         │  B1 exécute CLUSTER FAILOVER
         │  ├─ Prend possession des slots 5461-10922
         │  ├─ Se déclare master (epoch++)
         │  └─> Propage via Gossip
         ▼

T0+20s : Cluster stabilisé
         │
         │  Tous les nœuds savent :
         │  ├─ B est FAIL
         │  ├─ B1 est le nouveau master
         │  └─ Slots 5461-10922 → B1
         │
         │  Clients sont redirigés vers B1
         ▼

Résultat : Downtime de ~15-20 secondes pour les slots de B
```

## Architecture décentralisée vs centralisée

### Comparaison avec architectures alternatives

```
┌─────────────────────────────────────────────────────────────┐
│         Redis Cluster vs Proxy-based Architecture           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROXY-BASED (ex: Twemproxy, Codis)                          │
│ ════════════════════════════════════════                    │
│                                                             │
│     Client → Proxy → Redis Instances                        │
│              │                                              │
│              └─> SPOF + Bottleneck                          │
│                                                             │
│  ✓ Simple à configurer                                      │
│  ✗ Proxy = point de défaillance unique                      │
│  ✗ Latence ajoutée (hop supplémentaire)                     │
│  ✗ Scaling limité par la capacité du proxy                  │
│  ✗ Pas de failover automatique natif                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ REDIS CLUSTER (Décentralisé)                                │
│ ═══════════════════════════════                             │
│                                                             │
│     Client → Nœud Redis (routing intelligent)               │
│              │                                              │
│              └─> Tous les nœuds sont égaux                  │
│                                                             │
│  ✓ Pas de SPOF                                              │
│  ✓ Latence minimale (1 hop si client intelligent)           │
│  ✓ Scaling linéaire                                         │
│  ✓ Failover automatique intégré                             │
│  ✗ Plus complexe (client doit être cluster-aware)           │
│  ✗ Overhead du protocole Gossip                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Garanties et limitations

### Garanties offertes par Redis Cluster

```
1. DISPONIBILITÉ
   ├─ Le cluster reste opérationnel si majorité des masters accessibles
   ├─ Failover automatique en cas de panne d'un master (si replica existe)
   └─ Temps de basculement : ~15-30 secondes

2. PERFORMANCE
   ├─ Scaling quasi-linéaire en lecture et écriture
   ├─ Parallélisation des opérations sur différents slots
   └─ O(1) pour la plupart des opérations (GET, SET, HGET, etc.)

3. TOLÉRANCE AUX PANNES
   ├─ Résiste à la panne de (N/2 - 1) masters sans perte de données
   ├─ Détection automatique des pannes via consensus Gossip
   └─ Récupération automatique après réparation

4. CONSISTANCE (limitée)
   ├─ Cohérence éventuelle (Eventual Consistency)
   ├─ Pas de transactions distribuées (sauf si clés sur même slot)
   └─ Option WAIT pour attendre réplication synchrone
```

### Limitations importantes

```
1. OPÉRATIONS MULTI-CLÉS
   └─> Limitées aux clés du même slot (hash tag requis)
       Exemples non supportés :
       ├─ MGET user:1 user:2 user:3  (si slots différents)
       ├─ SUNION set:a set:b set:c   (si slots différents)
       └─ Transactions sur clés de slots différents

2. DATABASE UNIQUE
   └─> Pas de support de SELECT (toujours DB 0)
   └─> Incompatible avec applications multi-tenant via DB

3. SCRIPTS LUA
   └─> Les clés accessibles doivent être passées en argument
   └─> Toutes les clés doivent être dans le même slot

4. PUB/SUB
   └─> Les messages sont broadcast à tous les nœuds
   └─> Pas de sharding des channels (overhead réseau)
   └─> Solution : Sharded Pub/Sub (Redis 7+)

5. COHÉRENCE
   └─> Pas de cohérence forte garantie
   └─> Possibilité de perte de données lors de partitionnement
   └─> Fenêtre de vulnérabilité pendant le failover
```

## Procédures de maintenance courantes

### Vérification de l'état du cluster

```bash
# Commande de base pour vérifier la santé du cluster
redis-cli --cluster check 192.168.1.10:6379

# Sortie attendue :
192.168.1.10:6379 (a1b2c3d4...) -> 10240 keys | 5461 slots | 1 slaves.
192.168.1.11:6379 (e5f6g7h8...) -> 10280 keys | 5462 slots | 1 slaves.
192.168.1.12:6379 (i9j0k1l2...) -> 10244 keys | 5461 slots | 1 slaves.
[OK] All 16384 slots covered.

# Obtenir des informations détaillées
redis-cli --cluster info 192.168.1.10:6379

# Vérifier la propagation du Gossip
redis-cli -h 192.168.1.10 -p 6379 CLUSTER NODES
```

### Monitoring du protocole Gossip

```bash
# Voir les statistiques Gossip
redis-cli -h 192.168.1.10 -p 6379 CLUSTER INFO

# Métriques importantes :
cluster_stats_messages_sent:1234567
cluster_stats_messages_received:1234560
cluster_stats_messages_ping_sent:45000
cluster_stats_messages_pong_sent:45000
cluster_stats_messages_meet_sent:3
cluster_stats_messages_fail_sent:0

# Analyser les latences Gossip
redis-cli -h 192.168.1.10 -p 6379 --latency-history

# Identifier les nœuds lents dans le Gossip
redis-cli -h 192.168.1.10 -p 6379 CLUSTER NODES | grep -v connected
```

### Ajustement des paramètres Gossip

```bash
# Modifier le timeout de détection (redis.conf)
cluster-node-timeout 15000

# Réduire pour failover plus rapide (attention aux faux positifs) :
cluster-node-timeout 5000

# Augmenter pour éviter les faux positifs sur réseau lent :
cluster-node-timeout 30000

# Appliquer dynamiquement (sans redémarrage)
redis-cli CONFIG SET cluster-node-timeout 20000

# Vérifier la configuration actuelle
redis-cli CONFIG GET cluster-node-timeout
```

### Diagnostic des problèmes de sharding

```bash
# Identifier les hot spots (slots surchargés)
for i in {0..16383}; do
    count=$(redis-cli -h 192.168.1.10 CLUSTER COUNTKEYSINSLOT $i)
    echo "$i:$count"
done | sort -t: -k2 -n -r | head -20

# Vérifier la répartition des slots
redis-cli --cluster check 192.168.1.10:6379 | grep "slots"

# Rééquilibrer si nécessaire
redis-cli --cluster rebalance 192.168.1.10:6379 \
    --cluster-threshold 5 \
    --cluster-use-empty-masters
```

### Procédure de resharding planifié

```bash
# 1. Analyser l'état actuel
redis-cli --cluster check 192.168.1.10:6379

# 2. Calculer la nouvelle distribution souhaitée
# Exemple : 4 nœuds → 4096 slots chacun

# 3. Exécuter le resharding avec confirmation
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-from a1b2c3d4-source \
    --cluster-to m4n5o6p7-target \
    --cluster-slots 500 \
    --cluster-yes \
    --cluster-timeout 60000 \
    --cluster-pipeline 10

# 4. Monitorer la progression
watch -n 5 'redis-cli --cluster check 192.168.1.10:6379'

# 5. Vérifier l'intégrité post-resharding
redis-cli --cluster fix 192.168.1.10:6379
```

### Récupération après panne de Gossip

```bash
# Scénario : Cluster en état "fail" suite à problème Gossip

# 1. Identifier les nœuds problématiques
redis-cli -h 192.168.1.10 CLUSTER NODES | grep fail

# 2. Tenter une récupération automatique
redis-cli --cluster fix 192.168.1.10:6379

# 3. Si échec, reset d'un nœud (ATTENTION : DESTRUCTIF)
redis-cli -h 192.168.1.11 CLUSTER RESET SOFT

# 4. Réintégrer le nœud
redis-cli -h 192.168.1.10 CLUSTER MEET 192.168.1.11 6379

# 5. Réassigner les slots si nécessaire
redis-cli --cluster reshard 192.168.1.10:6379
```

## Bonnes pratiques opérationnelles

### Configuration réseau optimale

```bash
# Firewall rules pour Cluster
# Autoriser ports client (6379) et cluster bus (16379)

iptables -A INPUT -p tcp --dport 6379 -j ACCEPT
iptables -A INPUT -p tcp --dport 16379 -j ACCEPT

# S'assurer que le bus cluster est accessible entre tous les nœuds
# Test de connectivité :
nc -zv 192.168.1.11 16379

# Latence réseau acceptable : < 1ms (LAN), < 10ms (WAN)
ping -c 10 192.168.1.11
```

### Dimensionnement du cluster

```
Règles de dimensionnement :
───────────────────────────

Nombre de nœuds :
├─ Minimum : 3 masters (pour avoir majorité = 2)
├─ Recommandé : 6 nœuds (3 masters + 3 replicas)
├─ Maximum pratique : ~1000 nœuds
└─> Au-delà, overhead Gossip devient significatif

Nombre de slots par nœud :
├─ Idéal : ~2000-5000 slots par nœud
├─ Minimum : ~500 slots (granularité suffisante)
└─> 16384 / N nœuds = slots par nœud

Mémoire par nœud :
├─ Standard : 8-64 GB par nœud
├─ Large : 64-256 GB (attention à la fragmentation)
└─> Éviter >256GB (temps de fork élevé)

Replicas :
├─ Minimum : 1 replica par master
├─ Recommandé : 1-2 replicas par master
└─> Plus de 3 replicas rarement utile
```

### Checklist de mise en production

```
✅ AVANT LE DÉPLOIEMENT
   ├─ Vérifier que tous les nœuds peuvent communiquer (ports 6379 + 16379)
   ├─ Configurer cluster-node-timeout approprié (15000-30000 ms)
   ├─ Activer persistence (AOF + RDB) sur tous les nœuds
   ├─ Configurer maxmemory et maxmemory-policy
   └─ Tester le failover en environnement de staging

✅ CONFIGURATION RÉSEAU
   ├─ Latence inter-nœuds < 10ms
   ├─ Bande passante suffisante pour Gossip + réplication
   ├─ Pas de NAT entre les nœuds du cluster
   └─ DNS ou IPs fixes pour chaque nœud

✅ MONITORING
   ├─ Alertes sur cluster_state != ok
   ├─ Alertes sur cluster_slots_ok < 16384
   ├─ Monitoring de la latence Gossip
   ├─ Tracking des redirections -MOVED (taux élevé = problème)
   └─ Dashboard avec état de tous les nœuds

✅ OPÉRATIONS
   ├─ Procédure de resharding documentée
   ├─ Procédure de failover manuel documentée
   ├─ Plan de reprise d'activité (DRP)
   └─ Runbook pour les scénarios d'incident courants
```

## Conclusion

Redis Cluster représente une implémentation élégante et performante du scaling horizontal pour Redis, basée sur trois piliers fondamentaux :

1. **Le sharding via hash slots** offre un partitionnement déterministe et efficace des données
2. **Le protocole Gossip** assure une coordination décentralisée sans point de défaillance unique
3. **L'architecture distribuée** permet un scaling linéaire et une haute disponibilité native

Ces mécanismes, bien que sophistiqués, sont conçus pour fonctionner de manière transparente tout en offrant aux opérateurs un contrôle précis via des procédures de maintenance bien définies. La compréhension approfondie de ces concepts est essentielle pour déployer et maintenir un cluster Redis robuste en production.

---

**Points clés à retenir :**

- **16384 hash slots** : Unité atomique de distribution, calculés via CRC16
- **Gossip = O(log N)** : Convergence rapide avec faible overhead
- **Consensus PFAIL → FAIL** : Détection de pannes par majorité
- **Client-side routing** : Clients intelligents pour minimiser les redirections
- **Failover automatique** : ~15-30 secondes avec replicas
- **Pas de cohérence forte** : Accepter l'éventuelle cohérence
- **Limitation multi-clés** : Utiliser hash tags pour grouper
- **Monitoring crucial** : `cluster_state:ok` et `cluster_slots_ok:16384`

La section suivante (11.2) explorera l'architecture "Shared-Nothing" qui sous-tend ces concepts.

⏭️ [Architecture "Shared-Nothing" du Cluster](/11-architecture-distribuee-scaling/02-architecture-shared-nothing.md)
