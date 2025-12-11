🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.3 Distribution des données : Hash Slots (0-16383)

## Introduction

Le mécanisme de distribution des données dans Redis Cluster repose sur un concept élégant et déterministe : les **hash slots**. Contrairement à d'autres systèmes distribués qui utilisent du consistent hashing avec des anneaux virtuels, Redis implémente un modèle de partitionnement fixe basé sur 16384 slots prédéfinis. Cette approche offre une prévisibilité totale et simplifie considérablement les opérations de maintenance.

## Architecture des Hash Slots

### Le modèle de partitionnement

Redis Cluster divise l'espace de clés en **16384 hash slots** (numérotés de 0 à 16383). Chaque clé de la base de données est assignée de manière déterministe à l'un de ces slots via une fonction de hachage.

```
Espace total : 16384 slots (0-16383)
Fonction : HASH_SLOT = CRC16(key) mod 16384
```

#### Pourquoi 16384 slots ?

Ce nombre n'est pas arbitraire. Il représente un compromis optimal entre plusieurs contraintes techniques :

1. **Taille du bitmap de slots** : 16384 slots = 2048 octets (2KB)
   - Chaque nœud maintient un bitmap indiquant quels slots il possède
   - 2KB est suffisamment petit pour être transmis efficacement via le protocole Gossip
   - Permet des heartbeats rapides sans surcharge réseau

2. **Granularité de distribution** :
   - Avec 16384 slots, un cluster de 100 nœuds peut avoir ~164 slots par nœud
   - Granularité suffisante pour un rééquilibrage précis
   - Pas de surcharge computationnelle lors des calculs de slot

3. **Limite pratique** :
   - Redis recommande un maximum de ~1000 nœuds par cluster
   - 16384 slots permettent une distribution équitable même avec ce nombre de nœuds
   - Au-delà, le protocole Gossip devient inefficace

### Schéma de distribution basique

```
┌─────────────────────────────────────────────────────────────┐
│                    Espace de Hash Slots                     │
│                        (0 - 16383)                          │
└─────────────────────────────────────────────────────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Nœud A      │    │  Nœud B      │    │  Nœud C      │
│              │    │              │    │              │
│ Slots:       │    │ Slots:       │    │ Slots:       │
│ 0-5460       │    │ 5461-10922   │    │ 10923-16383  │
│              │    │              │    │              │
│ (~33.3%)     │    │ (~33.3%)     │    │ (~33.4%)     │
└──────────────┘    └──────────────┘    └──────────────┘
```

## Algorithme de calcul de Hash Slot

### Fonction de hachage CRC16

Redis utilise CRC16 (Cyclic Redundancy Check 16-bit) pour calculer le slot d'une clé :

```
HASH_SLOT = CRC16(key) & 16383
```

L'opération `& 16383` (équivalent à `mod 16384`) est une optimisation bitwise car 16384 = 2^14.

### Implémentation du calcul

Voici la logique interne simplifiée :

```c
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e;

    // Recherche des marqueurs {hashtag}
    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    if (s == keylen) {
        // Pas de hashtag, utiliser la clé complète
        return crc16(key, keylen) & 16383;
    }

    // Hashtag trouvé, rechercher la fermeture
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    if (e == keylen || e == s+1) {
        // Hashtag invalide, utiliser la clé complète
        return crc16(key, keylen) & 16383;
    }

    // Utiliser uniquement la partie entre { et }
    return crc16(key+s+1, e-s-1) & 16383;
}
```

### Exemples de calcul

```bash
# Clé simple
user:1000 → CRC16("user:1000") & 16383 = 5798

# Avec hashtag
user:{1000}:profile → CRC16("1000") & 16383 = 5649
user:{1000}:settings → CRC16("1000") & 16383 = 5649  # Même slot !

# Clés multiples pour transaction
{user:1000}:profile
{user:1000}:friends
{user:1000}:posts
# Toutes mappées au même slot via le hashtag "user:1000"
```

## Hash Tags : Contrôle de la co-localisation

### Principe des hash tags

Les hash tags permettent de forcer plusieurs clés à être assignées au même slot, essentiel pour :
- Les opérations multi-clés (MGET, MSET)
- Les transactions (MULTI/EXEC)
- Les scripts Lua
- Les opérations ensemblistes (SUNION, SINTER)

### Syntaxe et règles

```
Règle : Seule la partie entre { et } est utilisée pour le calcul du hash

Exemples valides :
{user1000}:profile         → hash sur "user1000"
{user1000}:friends         → hash sur "user1000"
user:{1000}:sessions       → hash sur "1000"
{invoice:2024:Q1}:total    → hash sur "invoice:2024:Q1"

Exemples invalides (pas de hashtag détecté) :
user{1000:profile          → hash sur toute la clé
user:1000}:profile         → hash sur toute la clé
user:{}:profile            → hash sur toute la clé (vide)
user::profile              → hash sur toute la clé
```

### Stratégies de naming avec hash tags

```
1. Par entité métier :
   {order:12345}:items
   {order:12345}:customer
   {order:12345}:shipping

2. Par tenant (multi-tenancy) :
   {tenant:acme}:users:list
   {tenant:acme}:config
   {tenant:acme}:sessions

3. Par fenêtre temporelle :
   {analytics:2024-12-11}:pageviews
   {analytics:2024-12-11}:conversions
   {analytics:2024-12-11}:revenue

4. Par shard manuel :
   {shard:0}:user:1000
   {shard:1}:user:1001
   {shard:2}:user:1002
```

## Distribution et assignation des slots

### Architecture d'un cluster minimal (3 masters)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Redis Cluster (3 nœuds)                     │
└─────────────────────────────────────────────────────────────────┘

Nœud Master A (192.168.1.10:6379)
├─ Node ID: a1b2c3d4...
├─ Slots assignés: 0-5460 (5461 slots)
└─ Replica: Nœud D (192.168.1.13:6379)

Nœud Master B (192.168.1.11:6379)
├─ Node ID: e5f6g7h8...
├─ Slots assignés: 5461-10922 (5462 slots)
└─ Replica: Nœud E (192.168.1.14:6379)

Nœud Master C (192.168.1.12:6379)
├─ Node ID: i9j0k1l2...
├─ Slots assignés: 10923-16383 (5461 slots)
└─ Replica: Nœud F (192.168.1.15:6379)

┌─────────────────────────────────────────────────────────────────┐
│  Principe : Chaque master possède une plage continue de slots   │
│  Total : 16384 slots répartis équitablement                     │
└─────────────────────────────────────────────────────────────────┘
```

### Mapping slots → nœuds

Chaque nœud du cluster maintient une table de correspondance :

```
Slot → Node mapping table (maintenue en mémoire)
─────────────────────────────────────────────────
Slot 0-5460      → Node A (a1b2c3d4...)
Slot 5461-10922  → Node B (e5f6g7h8...)
Slot 10923-16383 → Node C (i9j0k1l2...)

Cette table est synchronisée via le protocole Gossip
et mise à jour lors de :
- Ajout/suppression de nœuds
- Resharding
- Failover
```

### Vérification de la distribution

```bash
# Voir la distribution des slots sur tous les nœuds
redis-cli --cluster check 192.168.1.10:6379

# Sortie typique :
192.168.1.10:6379 (a1b2c3d4...) -> 5461 keys | 5461 slots | 1 slaves
192.168.1.11:6379 (e5f6g7h8...) -> 5462 keys | 5462 slots | 1 slaves
192.168.1.12:6379 (i9j0k1l2...) -> 5461 keys | 5461 slots | 1 slaves

[OK] All 16384 slots covered.
```

## Procédures de maintenance des slots

### 1. Visualisation de l'état des slots

#### Vérifier les slots d'un nœud spécifique

```bash
# Se connecter à un nœud
redis-cli -h 192.168.1.10 -p 6379

# Voir les slots assignés au nœud courant
127.0.0.1:6379> CLUSTER SLOTS

# Sortie : Liste des plages de slots avec leurs nœuds responsables
1) 1) (integer) 0           # Début de plage
   2) (integer) 5460        # Fin de plage
   3) 1) "192.168.1.10"     # IP du master
      2) (integer) 6379     # Port du master
      3) "a1b2c3d4..."      # Node ID
   4) 1) "192.168.1.13"     # IP de la replica
      2) (integer) 6379     # Port de la replica
      3) "d4e5f6g7..."      # Node ID replica
```

#### Obtenir les informations détaillées

```bash
# Info sur les slots du nœud local
127.0.0.1:6379> CLUSTER INFO
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1

# Voir tous les nœuds et leurs slots
127.0.0.1:6379> CLUSTER NODES
a1b2c3d4... 192.168.1.10:6379@16379 myself,master - 0 0 1 connected 0-5460
e5f6g7h8... 192.168.1.11:6379@16379 master - 0 1623456789 2 connected 5461-10922
i9j0k1l2... 192.168.1.12:6379@16379 master - 0 1623456790 3 connected 10923-16383
```

### 2. Resharding : Déplacement de slots

Le resharding est le processus de déplacement de slots d'un nœud à un autre, nécessaire lors de :
- Ajout de nouveaux nœuds
- Suppression de nœuds
- Rééquilibrage de la charge

#### Processus de resharding

```
┌────────────────────────────────────────────────────────────┐
│              Étapes du Resharding d'un Slot                │
└────────────────────────────────────────────────────────────┘

Phase 1 : PRÉPARATION
─────────────────────
Nœud Source              Nœud Destination
(Master A)               (Master D - nouveau)
    │                           │
    │  1. SETSLOT MIGRATING ──> │
    │                           │ 2. SETSLOT IMPORTING
    │                           │

Phase 2 : MIGRATION DES CLÉS
────────────────────────────
    │                           │
    │  3. Pour chaque clé :     │
    │     ├─ DUMP key           │
    │     ├─ RESTORE ─────────> │
    │     └─ DEL key            │
    │                           │

Phase 3 : FINALISATION
──────────────────────
    │                           │
    │  4. SETSLOT NODE <dest>   │
    │                           │
    │  5. Propagation Gossip    │
    │  ════════════════════════>│
    │         (tous nœuds)      │
    │                           │
```

#### Commandes de resharding manuel

```bash
# Étape 1 : Marquer le slot comme en cours de migration (sur source)
redis-cli -h 192.168.1.10 -p 6379
127.0.0.1:6379> CLUSTER SETSLOT 8000 MIGRATING i9j0k1l2-destination-node-id

# Étape 2 : Marquer le slot comme en cours d'import (sur destination)
redis-cli -h 192.168.1.12 -p 6379
127.0.0.1:6379> CLUSTER SETSLOT 8000 IMPORTING a1b2c3d4-source-node-id

# Étape 3 : Migration des clés une par une
redis-cli -h 192.168.1.10 -p 6379
127.0.0.1:6379> CLUSTER GETKEYSINSLOT 8000 1000
1) "user:5432"
2) "session:abc123"
# Pour chaque clé :
127.0.0.1:6379> MIGRATE 192.168.1.12 6379 "user:5432" 0 5000

# Étape 4 : Finaliser la migration (sur tous les nœuds)
127.0.0.1:6379> CLUSTER SETSLOT 8000 NODE i9j0k1l2-destination-node-id
```

#### Resharding automatisé avec redis-cli

```bash
# Resharding interactif
redis-cli --cluster reshard 192.168.1.10:6379

# Resharding automatique : déplacer 1000 slots vers un nœud
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-from a1b2c3d4-source-node-id \
    --cluster-to i9j0k1l2-dest-node-id \
    --cluster-slots 1000 \
    --cluster-yes

# Rééquilibrage automatique de tous les slots
redis-cli --cluster rebalance 192.168.1.10:6379 \
    --cluster-threshold 2 \
    --cluster-use-empty-masters
```

### 3. Ajout d'un nœud avec redistribution

#### Procédure complète

```bash
# Étape 1 : Ajouter le nouveau nœud au cluster
redis-cli --cluster add-node 192.168.1.15:6379 192.168.1.10:6379

# À ce stade, le nouveau nœud ne possède aucun slot

# Étape 2 : Vérifier l'état
redis-cli --cluster check 192.168.1.10:6379

# Sortie :
# 192.168.1.15:6379 (m4n5o6p7...) -> 0 keys | 0 slots | 0 slaves  ← Nouveau
# 192.168.1.10:6379 (a1b2c3d4...) -> 5461 keys | 5461 slots | 1 slaves
# 192.168.1.11:6379 (e5f6g7h8...) -> 5462 keys | 5462 slots | 1 slaves
# 192.168.1.12:6379 (i9j0k1l2...) -> 5461 keys | 5461 slots | 1 slaves

# Étape 3 : Redistribuer les slots équitablement
redis-cli --cluster rebalance 192.168.1.10:6379

# Résultat final (4 nœuds, ~4096 slots chacun) :
# 192.168.1.15:6379 -> 4096 slots (0-4095)
# 192.168.1.10:6379 -> 4096 slots (4096-8191)
# 192.168.1.11:6379 -> 4096 slots (8192-12287)
# 192.168.1.12:6379 -> 4096 slots (12288-16383)
```

### 4. Suppression d'un nœud avec redistribution

```bash
# Étape 1 : Déplacer tous les slots du nœud à supprimer
redis-cli --cluster reshard 192.168.1.10:6379 \
    --cluster-from i9j0k1l2-node-to-remove \
    --cluster-to all \
    --cluster-slots 5461 \
    --cluster-yes

# Étape 2 : Vérifier que le nœud n'a plus de slots
redis-cli --cluster check 192.168.1.10:6379
# Le nœud devrait afficher : 0 keys | 0 slots

# Étape 3 : Supprimer le nœud du cluster
redis-cli --cluster del-node 192.168.1.10:6379 i9j0k1l2-node-to-remove

# Étape 4 : Arrêter le service Redis sur le nœud supprimé
ssh 192.168.1.12
sudo systemctl stop redis
```

### 5. Gestion des slots orphelins

Des slots peuvent devenir orphelins lors de :
- Échec d'un resharding
- Crash d'un nœud pendant une migration
- Corruption de la configuration du cluster

#### Détection des slots orphelins

```bash
# Vérifier l'intégrité du cluster
redis-cli --cluster check 192.168.1.10:6379

# Sortie en cas de problème :
[WARNING] Node 192.168.1.11:6379 has slots in migrating state (8000).
[WARNING] Node 192.168.1.12:6379 has slots in importing state (8000).
[ERR] Not all 16384 slots are covered by nodes.

# Identifier les slots non assignés
redis-cli --cluster fix 192.168.1.10:6379 --cluster-fix-with-unreachable-masters
```

#### Correction manuelle

```bash
# Cas 1 : Slot en état MIGRATING bloqué
redis-cli -h 192.168.1.11 -p 6379
127.0.0.1:6379> CLUSTER SETSLOT 8000 STABLE

# Cas 2 : Slot en état IMPORTING bloqué
redis-cli -h 192.168.1.12 -p 6379
127.0.0.1:6379> CLUSTER SETSLOT 8000 STABLE

# Cas 3 : Réassigner un slot orphelin
127.0.0.1:6379> CLUSTER SETSLOT 8000 NODE i9j0k1l2-target-node-id

# Vérifier la correction
redis-cli --cluster check 192.168.1.10:6379
[OK] All 16384 slots covered.
```

### 6. Monitoring continu des slots

#### Script de surveillance

```bash
#!/bin/bash
# check-cluster-slots.sh

CLUSTER_NODE="192.168.1.10:6379"

echo "=== Vérification de la couverture des slots ==="
redis-cli -h ${CLUSTER_NODE%:*} -p ${CLUSTER_NODE#*:} --cluster check $CLUSTER_NODE | grep -E "slots|All 16384"

echo ""
echo "=== Distribution des slots par nœud ==="
redis-cli -h ${CLUSTER_NODE%:*} -p ${CLUSTER_NODE#*:} --cluster info $CLUSTER_NODE

echo ""
echo "=== Slots en migration ==="
redis-cli -h ${CLUSTER_NODE%:*} -p ${CLUSTER_NODE#*:} CLUSTER NODES | grep -E "MIGRATING|IMPORTING"

if [ $? -eq 0 ]; then
    echo "⚠️  ATTENTION : Des slots sont en cours de migration"
    exit 1
else
    echo "✅ Aucune migration en cours"
fi
```

## Optimisations et considérations avancées

### Granularité du resharding

```
Compromis entre vitesse et disponibilité :

Migration de 1000 slots en une fois :
├─ Avantages : Plus rapide (moins d'overhead)
└─ Inconvénients : Impact sur les performances

Migration de 10 slots à la fois :
├─ Avantages : Impact minimal sur les performances
└─ Inconvénients : Plus lent, plus d'opérations

Recommandation production :
└─ Migrer par lots de 100-500 slots
   avec pause de 1-2 secondes entre chaque lot
```

### Pipeline de migration

Pour optimiser les performances lors du resharding :

```bash
# Utiliser le pipeline pour réduire les RTT
redis-cli -h source-node --cluster reshard target-node \
    --cluster-slots 1000 \
    --cluster-pipeline 10 \
    --cluster-replace

# --cluster-pipeline 10 : migrer 10 clés en parallèle
# --cluster-replace : remplacer les clés existantes sur destination
```

### Calcul de la charge par slot

```bash
# Obtenir le nombre de clés dans une plage de slots
redis-cli -h 192.168.1.10 -p 6379

# Pour un slot spécifique
127.0.0.1:6379> CLUSTER COUNTKEYSINSLOT 5000
(integer) 42

# Pour une plage de slots (script)
for slot in {0..100}; do
    count=$(redis-cli CLUSTER COUNTKEYSINSLOT $slot)
    echo "Slot $slot: $count keys"
done | sort -t: -k2 -n -r | head -20  # Top 20 slots les plus chargés
```

### Impact sur les clients pendant resharding

```
Comportement client pendant la migration d'un slot :

Client envoie GET user:5432 (slot 8000)
         │
         ▼
    Nœud Source (A)
         │
         ├─ Clé présente localement ?
         │  └─> Oui : Retourner la valeur
         │
         └─ Clé déjà migrée ?
            └─> Oui : Retourner "-MOVED 8000 192.168.1.12:6379"

Client suit la redirection :
         │
         ▼
    Nœud Destination (D)
         └─> Retourner la valeur

Pendant la période de migration :
- Certaines clés sur source
- Certaines clés sur destination
- Redirections -MOVED automatiques
- Transparence pour l'application (client intelligent)
```

## Cas limites et situations exceptionnelles

### Cluster en mode dégradé

```
Scénario : Un master tombe sans replica disponible

État initial :
Node A: Slots 0-5460     ✅ UP
Node B: Slots 5461-10922 ✅ UP
Node C: Slots 10923-16383 ❌ DOWN (no replica)

Résultat :
cluster_state: fail
cluster_slots_fail: 5461

Les slots 10923-16383 sont INACCESSIBLES
└─> Toutes les opérations sur ces slots échouent
    avec "-CLUSTERDOWN Hash slot not served"
```

### Récupération après incident

```bash
# Option 1 : Forcer le cluster à accepter l'état dégradé
redis-cli -h 192.168.1.10 -p 6379
127.0.0.1:6379> CLUSTER SETSLOT 10923 STABLE
# Répéter pour chaque slot affecté...

# Option 2 : Réassigner les slots orphelins à un autre master
redis-cli --cluster fix 192.168.1.10:6379 \
    --cluster-fix-with-unreachable-masters

# Cette commande va :
# 1. Identifier les slots non servis
# 2. Les réassigner au master disponible le moins chargé
# 3. Propager la nouvelle configuration via Gossip
```

### Limite de resharding

```
Contrainte : On ne peut pas déplacer un slot qui contient
des clés en cours de modification (race condition)

Solution : Redis implémente un mécanisme de locking :
1. Les clés du slot en migration restent accessibles
2. Nouvelles écritures sur la destination
3. Lecture redirigée automatiquement
4. Migration atomique des clés restantes
```

## Checklist de maintenance des slots

```
✅ Avant toute opération de maintenance :
   ├─ Vérifier cluster_state:ok
   ├─ Confirmer que tous les slots sont couverts (16384)
   ├─ S'assurer qu'aucune migration n'est en cours
   └─ Backup de la configuration du cluster

✅ Pendant le resharding :
   ├─ Monitorer la latence des requêtes
   ├─ Surveiller l'utilisation mémoire (MIGRATE = copie temporaire)
   ├─ Vérifier les logs pour les erreurs de migration
   └─ Mesurer le taux de redirections -MOVED

✅ Après le resharding :
   ├─ Exécuter CLUSTER CHECK pour validation
   ├─ Vérifier la distribution des clés par nœud
   ├─ Confirmer que cluster_slots_assigned = 16384
   └─ Tester l'accès aux clés sur tous les slots migrés

✅ En cas de problème :
   ├─ Identifier l'état des slots (CLUSTER NODES)
   ├─ Vérifier les logs Redis de tous les nœuds
   ├─ Utiliser CLUSTER SETSLOT STABLE pour débloquer
   └─ En dernier recours : CLUSTER RESET + recréation
```

## Conclusion

Le système de hash slots de Redis Cluster offre un modèle de partitionnement déterministe et prévisible, essentiel pour construire des systèmes distribués fiables. La compréhension approfondie de ce mécanisme permet :

- **Planification précise** : Anticiper la distribution des données lors de l'ajout/suppression de nœuds
- **Maintenance contrôlée** : Effectuer des opérations de resharding sans interruption de service
- **Optimisation** : Utiliser les hash tags pour garantir la co-localisation des données liées
- **Troubleshooting** : Diagnostiquer et corriger rapidement les problèmes de slots orphelins ou bloqués

La maîtrise du resharding et des procédures de maintenance des slots est une compétence fondamentale pour tout architecte ou opérateur de Redis Cluster en environnement de production critique.

---

**Points clés à retenir :**

1. **16384 slots fixes** : Espace prédéterminé, pas de consistent hashing dynamique
2. **CRC16 & 16383** : Fonction de hachage déterministe et rapide
3. **Hash tags `{...}`** : Permettent la co-localisation pour opérations multi-clés
4. **Resharding = 4 phases** : MIGRATING → IMPORTING → Migration clés → SETSLOT NODE
5. **Monitoring continu** : Vérifier régulièrement avec `CLUSTER CHECK` et `CLUSTER SLOTS`
6. **Pipeline de migration** : Utiliser `--cluster-pipeline` pour optimiser les performances
7. **Gestion des erreurs** : Toujours avoir un plan de récupération pour les slots orphelins

⏭️ [Redis Cluster : Concepts (Sharding, Hash Slots, Gossip Protocol)](/11-architecture-distribuee-scaling/01-redis-cluster-concepts.md)
