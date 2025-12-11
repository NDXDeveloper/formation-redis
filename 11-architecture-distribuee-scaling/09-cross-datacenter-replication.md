🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.9 Cross-datacenter replication (Active-Active vs Active-Passive)

## Introduction

La réplication cross-datacenter (XDR - Cross-Datacenter Replication) constitue l'ultime niveau de résilience et de disponibilité pour Redis Cluster. Alors que la réplication intra-cluster assure la haute disponibilité face aux défaillances individuelles de nœuds, la réplication géographique protège contre des pannes de datacenters entiers (catastrophes naturelles, pannes régionales cloud, coupures réseau majeures) tout en permettant de servir les utilisateurs depuis des emplacements géographiquement proches pour réduire la latence.

Cette section explore les architectures Active-Passive et Active-Active, leurs implémentations techniques, leurs compromis en termes de cohérence et de disponibilité, ainsi que les défis spécifiques à la réplication géographique distribuée.

## Contexte et motivations

### Pourquoi la réplication cross-datacenter ?

```
┌──────────────────────────────────────────────────────────────┐
│          Motivations pour XDR (Cross-Datacenter Replication) │
└──────────────────────────────────────────────────────────────┘

1. DISASTER RECOVERY (DR)
═════════════════════════
Protection contre perte complète d'un datacenter

Scénarios :
├─ Catastrophe naturelle (ouragan, tremblement de terre)
├─ Panne électrique régionale prolongée
├─ Incendie datacenter
├─ Attaque physique / terrorisme
└─ Erreur opérationnelle majeure (suppression zone AWS)

Objectifs :
├─ RPO (Recovery Point Objective) : < 1 seconde de perte de données
└─ RTO (Recovery Time Objective) : < 5 minutes de downtime


2. RÉDUCTION DE LATENCE GÉOGRAPHIQUE
═════════════════════════════════════
Servir utilisateurs depuis DC proche

Sans XDR (tous utilisateurs → DC unique) :
    User US East → DC Europe → 150ms latency
    User Asia → DC Europe → 300ms latency

Avec XDR (utilisateurs → DC proche) :
    User US East → DC US East → 10ms latency
    User Europe → DC Europe → 10ms latency
    User Asia → DC Asia → 10ms latency

Amélioration : 10-30x réduction de latence


3. CONFORMITÉ RÉGLEMENTAIRE
════════════════════════════
Respect des lois de résidence des données

RGPD (Europe) : Données citoyens EU doivent rester en EU
Lois chinoises : Données citoyens chinois en Chine
CLOUD Act (US) : Implications juridiques données hors US

Solution : Données répliquées localement selon géolocalisation


4. HAUTE DISPONIBILITÉ GÉOGRAPHIQUE
════════════════════════════════════
Tolérance aux pannes régionales

Single DC :
    Availability = 99.9% (8.76h downtime/an)

Multi-DC avec failover :
    Availability = 99.99% (52.6min downtime/an)

Multi-DC Active-Active :
    Availability = 99.999% (5.26min downtime/an)


5. SCALABILITÉ GÉOGRAPHIQUE
════════════════════════════
Capacité lecture distribuée globalement

Single DC : 100k req/sec maximum
Multi-DC : N × 100k req/sec (N datacenters)

Exemple : 3 DCs → 300k req/sec capacité totale


6. ISOLATION DES PANNES
════════════════════════
Blast radius limité

Panne dans DC Asia :
├─ Sans XDR : Service global down
└─ Avec XDR : Seulement users Asia affectés temporairement
              Users US/Europe continuent normalement
```

### Challenges spécifiques à Redis Cluster XDR

```
┌─────────────────────────────────────────────────────────────┐
│              Défis de la réplication géographique           │
└─────────────────────────────────────────────────────────────┘

1. LATENCE WAN
══════════════
Liaison inter-DC : 50-300ms (vs <1ms LAN)

Impact sur réplication synchrone :
    WRITE → Réplication vers DC distant → 300ms
    └─> Chaque écriture prend 300ms minimum
        └─> Throughput limité, latence inacceptable

Solution : Réplication ASYNCHRONE obligatoire


2. THÉORÈME CAP À GRANDE ÉCHELLE
═════════════════════════════════
Partition réseau WAN plus fréquente qu'en LAN

Network partition entre DCs :
├─ Réplication impossible temporairement
├─ Divergence des données entre DCs
└─ Nécessité de gestion de conflits

Redis Cluster XDR choisit : AP (Availability + Partition tolerance)
    └─> Cohérence éventuelle (eventual consistency)


3. BANDE PASSANTE LIMITÉE
══════════════════════════
WAN moins performant que LAN

Bande passante typique :
├─ LAN : 10-100 Gbps
└─ WAN : 1-10 Gbps

Impact :
├─ Coût élevé du transfert (facturé au volume)
├─ Saturation possible lors de pics
└─> Compression et optimisation nécessaires


4. ARCHITECTURE REDIS CLUSTER NON CONÇUE POUR XDR
══════════════════════════════════════════════════
Redis Cluster open-source :
├─ Pas de réplication cross-cluster native
├─ Chaque cluster est indépendant
└─> Solutions tierces ou Redis Enterprise nécessaires


5. GESTION DES CONFLITS D'ÉCRITURE
═══════════════════════════════════
Active-Active : Écritures concurrentes sur même clé

DC US : SET user:1000:name "Alice"  (t=0)
DC EU : SET user:1000:name "Alicia" (t=0)

Après réplication :
    DC US : name = "Alicia" (dernière écriture)
    DC EU : name = "Alice"  (dernière écriture)

    ✗ Inconsistance !

Solution : CRDT (Conflict-free Replicated Data Types)
           ou Last-Write-Wins (LWW) avec timestamps


6. COÛT FINANCIER
═════════════════
Infrastructure multi-DC très coûteuse

Coûts :
├─ N × Infrastructure (chaque DC = cluster complet)
├─ Bande passante WAN facturée au volume
├─ Licences Redis Enterprise (si utilisé)
└─> 3-5x le coût d'un déploiement single-DC
```

## Architecture Active-Passive

### Principe et topologie

```
┌─────────────────────────────────────────────────────────────┐
│                   Active-Passive (Master-Slave)             │
└─────────────────────────────────────────────────────────────┘

Principe : Un DC actif (écritures + lectures), un DC passif (standby)


    ┌──────────────────────────────────────────────────────┐
    │                   Datacenter ACTIF                   │
    │                    (Primary)                         │
    │                                                      │
    │   ┌──────────┐   ┌──────────┐   ┌──────────┐         │
    │   │  Node A  │   │  Node B  │   │  Node C  │         │
    │   │ Master   │   │ Master   │   │ Master   │         │
    │   └────┬─────┘   └────┬─────┘   └────┬─────┘         │
    │        │              │              │               │
    │        │    Redis Cluster (Primary)  │               │
    │        │              │              │               │
    └────────┼──────────────┼──────────────┼───────────────┘
             │              │              │
             │              │              │
             │   Réplication WAN (async)   │
             │              │              │
             ▼              ▼              ▼
    ┌────────────────────────────────────────────────────────┐
    │                 Datacenter PASSIF                      │
    │                   (Secondary/Standby)                  │
    │                                                        │
    │   ┌──────────┐   ┌──────────┐   ┌──────────┐           │
    │   │  Node D  │   │  Node E  │   │  Node F  │           │
    │   │  Replica │   │  Replica │   │  Replica │           │
    │   └──────────┘   └──────────┘   └──────────┘           │
    │                                                        │
    │             Redis Cluster (Secondary)                  │
    │             Receive only (pas d'écritures)             │
    │                                                        │
    └────────────────────────────────────────────────────────┘


Flux de données :
═════════════════

1. Toutes les écritures → DC Actif
2. DC Actif réplique vers DC Passif (asynchrone)
3. DC Passif applique les changements
4. DC Passif ne sert PAS les lectures (option : peut servir lectures)


Avantages :
═══════════
✓ Simple à comprendre et implémenter
✓ Pas de conflits d'écriture (un seul master)
✓ Cohérence plus forte (single source of truth)
✓ Coût modéré (DC passif peut être plus petit)
✓ Compatible avec Redis Cluster open-source (via outils tiers)


Inconvénients :
═══════════════
✗ DC passif sous-utilisé (gaspillage ressources)
✗ Pas de réduction de latence pour utilisateurs distants
✗ Failover manuel ou automatique nécessaire
✗ RPO > 0 (perte possible de données récentes)
✗ Capacité lecture limitée au DC actif
```

### Implémentation avec Redis Cluster

```bash
# ═══════════════════════════════════════════════════════════
# IMPLÉMENTATION ACTIVE-PASSIVE AVEC REDIS CLUSTER
# ═══════════════════════════════════════════════════════════

# ARCHITECTURE
# ────────────
# DC Primary (US-EAST) : Cluster 3 nodes (active)
# DC Secondary (EU-WEST) : Cluster 3 nodes (passive)


# MÉTHODE 1 : Réplication au niveau RDB/AOF
# ──────────────────────────────────────────

# Sur chaque nœud du DC Primary, activer persistence
# redis.conf
appendonly yes
save 900 1
save 300 10

# Script de synchronisation continue
#!/bin/bash
# sync-to-secondary.sh

PRIMARY_NODES=("us-east-1a" "us-east-1b" "us-east-1c")
SECONDARY_NODES=("eu-west-1a" "eu-west-1b" "eu-west-1c")

while true; do
    for i in "${!PRIMARY_NODES[@]}"; do
        primary="${PRIMARY_NODES[$i]}"
        secondary="${SECONDARY_NODES[$i]}"

        # RDB sync
        scp "$primary:/var/lib/redis/dump.rdb" "/tmp/dump-$i.rdb"
        scp "/tmp/dump-$i.rdb" "$secondary:/var/lib/redis/dump.rdb"

        # Redémarrer secondary pour charger nouveau RDB
        ssh "$secondary" "redis-cli shutdown nosave && systemctl start redis"
    done

    sleep 300  # Toutes les 5 minutes
done

# ⚠️ Cette méthode est basique et a un RPO de ~5 minutes


# MÉTHODE 2 : Redis REPLICAOF avec tunneling
# ───────────────────────────────────────────

# Créer un tunnel SSH/VPN entre DCs

# Sur chaque nœud Secondary, configurer comme replica du Primary

# Secondary Node D (EU-WEST-1A)
redis-cli -h eu-west-1a -p 6379 REPLICAOF us-east-1a 6379

# Problème : Redis Cluster n'est pas conçu pour réplication cross-cluster
# Chaque nœud se réplique individuellement, pas de cohérence cluster


# MÉTHODE 3 : RIOT (Redis Input/Output Tools)
# ────────────────────────────────────────────

# RIOT : Outil open-source pour réplication Redis
# https://github.com/redis-developer/riot

# Installation
wget https://github.com/redis-developer/riot/releases/download/v3.1.0/riot-3.1.0.zip
unzip riot-3.1.0.zip

# Réplication live Primary → Secondary
./riot replicate \
    redis://us-east-1a:6379,us-east-1b:6379,us-east-1c:6379 \
    redis://eu-west-1a:6379,eu-west-1b:6379,eu-west-1c:6379 \
    --mode live \
    --batch-size 500

# Options :
# --mode live : Réplication continue (vs snapshot)
# --mode snapshot : Réplication ponctuelle
# --batch-size : Nombre de commandes par batch


# Avantages RIOT :
✓ Fonctionne avec Redis Cluster
✓ Réplication continue (streaming)
✓ Open-source et gratuit
✓ Support RDB + AOF + Live streaming

# Inconvénients RIOT :
✗ Point de défaillance (si RIOT crash, réplication stop)
✗ Latence additionnelle
✗ Pas de gestion de failover automatique


# MÉTHODE 4 : RedisRaft (expérimental)
# ─────────────────────────────────────

# RedisRaft : Module Redis implémentant Raft consensus
# Permet réplication multi-DC avec consensus

# ⚠️ Expérimental, non recommandé pour production


# MÉTHODE 5 : Redis Enterprise (solution commerciale)
# ────────────────────────────────────────────────────

# Redis Enterprise supporte nativement Active-Passive XDR

# Configuration via UI ou API
curl -X POST https://redis-enterprise-api/v1/crdb \
    -d '{
        "name": "my-database",
        "replication": "active-passive",
        "participating_clusters": [
            {"url": "redis://us-east-cluster"},
            {"url": "redis://eu-west-cluster", "passive": true}
        ]
    }'

# Avantages Redis Enterprise :
✓ Réplication native et optimisée
✓ Failover automatique
✓ Compression WAN intégrée
✓ Monitoring et observabilité
✓ Support commercial

# Inconvénients :
✗ Coût élevé (licence)
✗ Vendor lock-in
```

### Procédure de failover

```bash
# ═══════════════════════════════════════════════════════════
# PROCÉDURE DE FAILOVER : ACTIF → PASSIF
# ═══════════════════════════════════════════════════════════

# SCÉNARIO : DC Primary (US-EAST) est tombé
#            Basculer vers DC Secondary (EU-WEST)


# ÉTAPE 1 : DÉTECTER LA PANNE (automatique ou manuel)
# ────────────────────────────────────────────────────

# Health check automatique
#!/bin/bash
# health-check.sh

PRIMARY_ENDPOINT="us-east-cluster.example.com:6379"

while true; do
    if ! redis-cli -h $PRIMARY_ENDPOINT PING > /dev/null 2>&1; then
        echo "PRIMARY DOWN - Initiating failover"
        /usr/local/bin/trigger-failover.sh
        break
    fi
    sleep 10
done


# ÉTAPE 2 : ARRÊTER LA RÉPLICATION (éviter corruption)
# ─────────────────────────────────────────────────────

# Si Primary est partiellement accessible, arrêter réplication
# Pour éviter réplication de données corrompues

# Sur Secondary nodes
for node in eu-west-1a eu-west-1b eu-west-1c; do
    redis-cli -h $node REPLICAOF NO ONE
done


# ÉTAPE 3 : PROMOUVOIR SECONDARY EN PRIMARY
# ──────────────────────────────────────────

# Activer écritures sur Secondary cluster
# Si Redis Enterprise, via API :
curl -X POST https://redis-api/v1/crdb/promote \
    -d '{"cluster": "eu-west-cluster"}'

# Si setup manuel, simplement autoriser écritures :
# (pas de changement nécessaire si Secondary lisait déjà)


# ÉTAPE 4 : BASCULER LE TRAFIC (DNS ou Load Balancer)
# ─────────────────────────────────────────────────────

# Option A : DNS failover
# Changer enregistrement DNS pour pointer vers Secondary

aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890ABC \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "redis.example.com",
                "Type": "CNAME",
                "TTL": 60,
                "ResourceRecords": [{"Value": "eu-west-cluster.example.com"}]
            }
        }]
    }'

# Attention : TTL DNS peut retarder basculement (60s ici)


# Option B : Global Load Balancer (ex: AWS Global Accelerator)
# Désactiver endpoint Primary, activer endpoint Secondary

aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn arn:aws:... \
    --endpoint-configurations '[
        {
            "EndpointId": "eu-west-cluster",
            "Weight": 100
        }
    ]'

# Basculement instantané (pas de TTL DNS)


# ÉTAPE 5 : VÉRIFIER LE FAILOVER
# ───────────────────────────────

# Test écriture/lecture depuis nouveau Primary
redis-cli -h eu-west-cluster.example.com SET test:failover "success"
redis-cli -h eu-west-cluster.example.com GET test:failover

# Vérifier métriques
redis-cli -h eu-west-1a INFO replication
# role:master (devrait être master maintenant)

redis-cli -h eu-west-1a INFO stats
# total_commands_processed (devrait augmenter)


# ÉTAPE 6 : ÉVALUER LA PERTE DE DONNÉES
# ──────────────────────────────────────

# RPO dépend de la latence de réplication avant panne

# Vérifier AOF/RDB timestamp du Secondary
ls -lh /var/lib/redis/dump.rdb
# Si réplication était à 10s près, RPO ≈ 10 secondes


# ÉTAPE 7 : COMMUNICATION
# ────────────────────────

# Informer stakeholders du failover

cat <<EOF
INCIDENT: Primary datacenter failure - Failover executed

Timeline:
- 14:00 UTC: Primary DC (US-EAST) unresponsive
- 14:02 UTC: Automatic failover triggered
- 14:05 UTC: Traffic routed to Secondary DC (EU-WEST)
- 14:06 UTC: Service restored

Impact:
- Downtime: 6 minutes
- Data loss: ~10 seconds (RPO met)
- Geographic shift: Users now served from EU-WEST

Next steps:
- Root cause analysis of Primary failure
- Plan reconstruction of Primary DC
- Monitor Secondary for stability
EOF


# ÉTAPE 8 : POST-FAILOVER (après incident)
# ─────────────────────────────────────────

# Une fois Primary DC récupéré :

# Option A : Reconstruire Primary comme nouveau Secondary
# 1. Configurer Primary nodes en replica de EU-WEST
# 2. Laisser sync complet
# 3. Une fois à jour, décider si re-failover vers Primary

# Option B : Garder EU-WEST comme nouveau Primary permanent
# (si latence acceptable pour utilisateurs US)
```

### Monitoring et métriques

```yaml
# ═══════════════════════════════════════════════════════════
# MONITORING ACTIVE-PASSIVE
# ═══════════════════════════════════════════════════════════

# Métriques Prometheus
# ────────────────────

# prometheus-rules-xdr.yml

groups:
  - name: redis_xdr_active_passive
    interval: 30s
    rules:

      # Replication lag (écart entre Primary et Secondary)
      - record: redis:replication:lag_seconds
        expr: |
          (redis_replication_last_io_seconds_ago{role="slave"})

      # Alerte : Lag élevé
      - alert: RedisXDRHighLag
        expr: redis:replication:lag_seconds > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High replication lag between datacenters"
          description: "{{ $labels.instance }} is {{ $value }}s behind"

      # Alerte : Réplication cassée
      - alert: RedisXDRReplicationDown
        expr: redis_master_link_status{role="slave"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "XDR replication link down"
          description: "{{ $labels.instance }} lost connection to master"

      # Alerte : Primary datacenter down
      - alert: RedisXDRPrimaryDown
        expr: up{job="redis-primary"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Primary datacenter unreachable"
          description: "Failover may be required"

      # Bande passante WAN
      - record: redis:xdr:bandwidth_mbps
        expr: |
          rate(redis_net_output_bytes_total{role="master"}[1m]) * 8 / 1000000

      # Alerte : Bande passante saturée
      - alert: RedisXDRBandwidthSaturated
        expr: redis:xdr:bandwidth_mbps > 800  # 80% de 1 Gbps
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "WAN bandwidth near saturation"


# Dashboard Grafana
# ─────────────────

# grafana-xdr-dashboard.json (extrait)

{
  "panels": [
    {
      "title": "Replication Lag (Primary → Secondary)",
      "targets": [{
        "expr": "redis:replication:lag_seconds"
      }],
      "yaxis": {
        "label": "Seconds"
      }
    },
    {
      "title": "WAN Bandwidth Usage",
      "targets": [{
        "expr": "redis:xdr:bandwidth_mbps"
      }],
      "yaxis": {
        "label": "Mbps"
      }
    },
    {
      "title": "Commands/sec by Datacenter",
      "targets": [
        {
          "expr": "rate(redis_commands_processed_total{dc='us-east'}[1m])",
          "legendFormat": "Primary (US-EAST)"
        },
        {
          "expr": "rate(redis_commands_processed_total{dc='eu-west'}[1m])",
          "legendFormat": "Secondary (EU-WEST)"
        }
      ]
    },
    {
      "title": "Replication Status",
      "targets": [{
        "expr": "redis_master_link_status{role='slave'}"
      }],
      "valueMaps": [
        {"value": "0", "text": "DOWN"},
        {"value": "1", "text": "UP"}
      ]
    }
  ]
}


# Vérifications manuelles
# ───────────────────────

# Vérifier réplication lag
redis-cli -h eu-west-1a INFO replication | grep master_repl_offset

# Sur Primary :
# master_repl_offset:123456789

# Sur Secondary :
# master_repl_offset:123456500

# Lag = 123456789 - 123456500 = 289 bytes de retard


# Vérifier vitesse de réplication
redis-cli -h eu-west-1a INFO stats | grep instantaneous_input_kbps
# instantaneous_input_kbps:850.42  # Réplication à 850 KB/s
```

## Architecture Active-Active

### Principe et défis

```
┌─────────────────────────────────────────────────────────────┐
│                   Active-Active (Multi-Master)              │
└─────────────────────────────────────────────────────────────┘

Principe : Chaque DC accepte lectures ET écritures
          Réplication bidirectionnelle entre tous les DCs


    ┌────────────────────────┐       ┌────────────────────────┐
    │   Datacenter US-EAST   │       │   Datacenter EU-WEST   │
    │       (Active)         │       │       (Active)         │
    │                        │       │                        │
    │   ┌──────────────┐     │       │     ┌──────────────┐   │
    │   │  Cluster A   │     │       │     │  Cluster B   │   │
    │   │  (R/W)       │◄────┼───────┼────►│  (R/W)       │   │
    │   └──────────────┘     │       │     └──────────────┘   │
    │                        │       │                        │
    └────────────┬───────────┘       └───────────┬────────────┘
                 │                               │
                 │         Réplication           │
                 │        bidirectionnelle       │
                 │                               │
                 │       ┌────────────────────┐  │
                 │       │  Datacenter ASIA   │  │
                 └───────┤     (Active)       ├──┘
                         │                    │
                         │   ┌──────────────┐ │
                         │   │  Cluster C   │ │
                         │   │  (R/W)       │ │
                         │   └──────────────┘ │
                         │                    │
                         └────────────────────┘


Réplication mesh (tous vers tous) :
════════════════════════════════════

US-EAST ──────────────┐
   │                  │
   │                  ▼
   │              EU-WEST
   │                  │
   │                  │
   ▼                  ▼
 ASIA ────────────> EU-WEST

Chaque changement est répliqué vers tous les autres DCs


Avantages :
═══════════
✓ Latence minimale (users → DC proche)
✓ Haute disponibilité (perte 1 DC = service continue)
✓ Capacité lecture ET écriture distribuée globalement
✓ Pas de failover nécessaire (tous actifs)
✓ Scalabilité géographique maximale


Inconvénients :
═══════════════
✗ COMPLEXITÉ TRÈS ÉLEVÉE
✗ Gestion des conflits d'écriture
✗ Cohérence éventuelle (eventual consistency)
✗ Coût maximum (N × infrastructure complète)
✗ Debugging difficile (état distribué)
✗ Nécessite CRDT ou résolution de conflits


PROBLÈME CENTRAL : CONFLITS D'ÉCRITURE
═══════════════════════════════════════

Temps T=0 :
    US-EAST : SET user:1000:score 100
    EU-WEST : SET user:1000:score 200

Temps T=1 (après réplication) :
    US-EAST reçoit : user:1000:score = 200 (de EU-WEST)
    EU-WEST reçoit : user:1000:score = 100 (de US-EAST)

Quelle valeur garder ? 100 ou 200 ?
    └─> Stratégie de résolution nécessaire !
```

### Stratégies de résolution de conflits

```
┌─────────────────────────────────────────────────────────────┐
│              Stratégies de résolution de conflits           │
└─────────────────────────────────────────────────────────────┘


STRATÉGIE 1 : LAST-WRITE-WINS (LWW)
════════════════════════════════════

Principe : L'écriture avec le timestamp le plus récent gagne


DC1 (t=100) : SET key "value1"
DC2 (t=105) : SET key "value2"

Après réplication :
    Les deux DCs convergent vers "value2" (t=105 > t=100)


Implémentation :
───────────────

Chaque écriture inclut timestamp :

    SET key "value"
    └─> Stocké comme : {value: "value", timestamp: 1638360000.123}

Lors de réplication :
    IF incoming_timestamp > local_timestamp:
        Accepter nouvelle valeur
    ELSE:
        Ignorer (garder valeur locale)


Avantages :
├─ Simple à implémenter
├─ Convergence garantie
└─ Pas d'intervention manuelle

Inconvénients :
├─ Perte de données possible (écriture "perdante" ignorée)
├─ Dépend de la synchronisation des horloges
└─> Dérive d'horloge = incohérence


STRATÉGIE 2 : CRDT (Conflict-free Replicated Data Types)
═════════════════════════════════════════════════════════

Principe : Types de données mathématiquement conçus pour
          converger sans conflits


Types CRDT courants :
─────────────────────

1. COUNTER (G-Counter, PN-Counter)
   ├─ Chaque DC maintient son propre compteur
   └─ Total = somme de tous les compteurs

   DC1 : counter_dc1 = +10
   DC2 : counter_dc2 = +5
   Total : 10 + 5 = 15 ✓

   Même après réplication dans n'importe quel ordre !


2. SET (OR-Set)
   ├─ Observe-Remove Set
   └─ Éléments ont des IDs uniques

   DC1 : ADD("alice", uid=1)
   DC2 : ADD("bob", uid=2)

   Résultat convergent : {"alice", "bob"}


3. REGISTER (LWW-Register)
   ├─ Last-Write-Wins avec timestamp
   └─> Similaire à stratégie 1


4. MAP (OR-Map)
   ├─ Map avec résolution par champ
   └─ Chaque champ suit sa propre stratégie CRDT


Exemple Redis Enterprise CRDB :
────────────────────────────────

Redis Enterprise implémente CRDT nativement

SET key "value"  →  LWW-Register automatiquement
INCR counter     →  PN-Counter automatiquement
SADD set "item"  →  OR-Set automatiquement


Avantages :
├─ Pas de perte de données
├─ Convergence mathématiquement garantie
├─ Pas de synchronisation d'horloge stricte nécessaire
└─ Idéal pour Active-Active

Inconvénients :
├─ Complexité d'implémentation élevée
├─ Overhead mémoire (métadonnées CRDT)
├─ Pas tous les types de données supportés
└─> Redis Enterprise requis (ou implémentation custom)


STRATÉGIE 3 : VECTOR CLOCKS / VERSION VECTORS
══════════════════════════════════════════════

Principe : Chaque DC maintient un vecteur de versions


DC1 : [DC1:1, DC2:0, DC3:0]
DC2 : [DC1:0, DC2:1, DC3:0]

Écriture sur DC1 → [DC1:2, DC2:0, DC3:0]
Écriture sur DC2 → [DC1:0, DC2:2, DC3:0]

Après réplication :
    Détection de conflit si vecteurs incomparables
    └─> Résolution manuelle ou automatique nécessaire


Avantages :
├─ Détection précise des conflits
└─ Pas de dépendance horloge

Inconvénients :
├─ Overhead mémoire élevé
├─ Complexité de résolution
└─> Rarement utilisé avec Redis


STRATÉGIE 4 : APPLICATION-LEVEL CONFLICT RESOLUTION
════════════════════════════════════════════════════

Principe : L'application décide de la résolution


Exemple : Score de jeu
──────────────────────

DC1 : user:1000:score = 100 (t=1)
DC2 : user:1000:score = 150 (t=2)

Stratégie métier : Toujours prendre le score le plus élevé

Résolution custom :
    IF incoming_score > local_score:
        Accept
    ELSE:
        Reject

Résultat : score = 150 sur tous les DCs ✓


Avantages :
├─ Logique métier précise
└─ Pas de perte de données métier importantes

Inconvénients :
├─ Complexité applicative
├─ Chaque type de données nécessite sa logique
└─> Maintenance difficile


STRATÉGIE 5 : PARTITIONING (Éviter conflits)
═════════════════════════════════════════════

Principe : Chaque DC possède des clés exclusives


Shard par géographie :
    US users → Clés US-EAST uniquement
    EU users → Clés EU-WEST uniquement

    user:US:1000 → US-EAST (jamais écrit depuis EU)
    user:EU:2000 → EU-WEST (jamais écrit depuis US)

Pas de conflits car pas d'écritures concurrentes !


Avantages :
├─ Pas de conflits = pas de résolution
├─ Simple
└─ Conformité RGPD facilité

Inconvénients :
├─ Utilisateur "bloqué" à son DC d'origine
├─ Pas de vraie Active-Active
└─> Plus proche d'Active-Passive géographiquement partitionné
```

### Implémentation avec Redis Enterprise

```yaml
# ═══════════════════════════════════════════════════════════
# REDIS ENTERPRISE : ACTIVE-ACTIVE AVEC CRDB
# ═══════════════════════════════════════════════════════════

# Redis Enterprise supporte nativement Active-Active via CRDB
# (Conflict-free Replicated DataBase)


# Configuration CRDB via API
# ──────────────────────────

curl -X POST https://redis-api.example.com/v1/crdbs \
    -H "Content-Type: application/json" \
    -d '{
        "name": "global-database",
        "memory_size": 10737418240,
        "replication": true,
        "participating_clusters": [
            {
                "cluster": {
                    "url": "https://us-east.redis.example.com",
                    "name": "us-east-cluster"
                }
            },
            {
                "cluster": {
                    "url": "https://eu-west.redis.example.com",
                    "name": "eu-west-cluster"
                }
            },
            {
                "cluster": {
                    "url": "https://asia-pacific.redis.example.com",
                    "name": "asia-cluster"
                }
            }
        ],
        "crdt_config": {
            "causal_consistency": true,
            "sync_sources": ["all"],
            "featureset_version": 1
        },
        "data_persistence": "aof-every-1-sec",
        "eviction_policy": "allkeys-lru"
    }'


# Comportement CRDT automatique
# ──────────────────────────────

# Les types Redis standard deviennent CRDT transparently :

# STRING → LWW-Register
redis-cli -h us-east SET user:1000:name "Alice"
redis-cli -h eu-west SET user:1000:name "Alicia"
# Résolution automatique avec timestamp


# COUNTER → PN-Counter (Positive-Negative Counter)
redis-cli -h us-east INCRBY global:views 100
redis-cli -h eu-west INCRBY global:views 50
# Après réplication : global:views = 150 (somme)


# SET → OR-Set
redis-cli -h us-east SADD tags "redis" "cluster"
redis-cli -h eu-west SADD tags "active-active" "crdt"
# Après réplication : tags = {"redis", "cluster", "active-active", "crdt"}


# SORTED SET → LWW-Sorted-Set avec scores
redis-cli -h us-east ZADD leaderboard 100 "player1"
redis-cli -h eu-west ZADD leaderboard 150 "player1"
# Résolution : score le plus récent (150)


# Monitoring CRDB
# ───────────────

# Métriques spécifiques CRDB

# Lag de réplication entre DCs
curl https://redis-api.example.com/v1/crdbs/global-database/stats | jq '.replication_lag'

# Output :
{
  "us-east → eu-west": "120ms",
  "us-east → asia": "250ms",
  "eu-west → us-east": "125ms",
  "eu-west → asia": "200ms",
  "asia → us-east": "245ms",
  "asia → eu-west": "195ms"
}


# Bandwidth entre DCs
curl https://redis-api.example.com/v1/crdbs/global-database/stats | jq '.bandwidth'

# Output :
{
  "us-east → eu-west": "45 Mbps",
  "us-east → asia": "38 Mbps",
  ...
}


# Conflits détectés et résolus
curl https://redis-api.example.com/v1/crdbs/global-database/stats | jq '.conflicts'

# Output :
{
  "total_conflicts": 12456,
  "conflicts_resolved": 12456,
  "conflicts_pending": 0
}


# Configuration avancée
# ─────────────────────

# Activer causal consistency (cohérence causale)
# Garantit que les lectures respectent l'ordre causal des écritures

curl -X PUT https://redis-api.example.com/v1/crdbs/global-database \
    -d '{"causal_consistency": true}'

# Avec causal consistency :
#   DC1: SET x "1"; SET y "2"  (y dépend de x)
#   DC2: GET y → "2"  THEN GET x → "1"  ✓
#
# Sans causal consistency :
#   DC2: GET y → "2"  THEN GET x → null (pas encore répliqué) ✗


# Compression WAN
curl -X PUT https://redis-api.example.com/v1/crdbs/global-database \
    -d '{"compression": 6}'  # Niveau 0-9 (6 = bon compromis)

# Réduit bandwidth de 50-70% typiquement
```

### Patterns de déploiement géographique

```
┌─────────────────────────────────────────────────────────────┐
│              Patterns de déploiement multi-DC               │
└─────────────────────────────────────────────────────────────┘


PATTERN 1 : GLOBAL MESH (3+ DCs, tous Active)
══════════════════════════════════════════════

Topologie complète : Chaque DC connecté à tous les autres


       US-EAST ←──────────────→ EU-WEST
          ↕                         ↕
          ↕                         ↕
          ↕                         ↕
       ASIA-PAC ←──────────────────┘


Latence réplication :
    US-EAST ↔ EU-WEST : 80ms
    US-EAST ↔ ASIA-PAC : 200ms
    EU-WEST ↔ ASIA-PAC : 150ms

Cas d'usage :
├─ Application globale (gaming, social network)
├─ Utilisateurs répartis mondialement
└─ Besoin latence minimale partout

Coût : ÉLEVÉ (3 × infra complète + bandwidth mesh)


PATTERN 2 : HUB-AND-SPOKE (1 hub central, N spokes)
════════════════════════════════════════════════════

Hub central réplique vers spokes
Spokes répliquent vers hub (pas entre eux)


    SPOKE-1 ────────┐
                    │
    SPOKE-2 ────────┼────→ HUB (central)
                    │
    SPOKE-3 ────────┘


Avantages :
├─ Réduit connexions (N au lieu de N×(N-1)/2)
├─ Simplifie monitoring
└─ Coût bandwidth réduit

Inconvénients :
├─ Latence 2× entre spokes (via hub)
└─ Hub = point de contention

Cas d'usage :
├─ Hub = DC principal (US)
├─ Spokes = DC régionaux (EU, ASIA)
└─ Trafic inter-régional faible


PATTERN 3 : REGIONAL CLUSTERS
══════════════════════════════

Chaque région = cluster indépendant
Réplication sélective de certaines clés


    US-CLUSTER          EU-CLUSTER          ASIA-CLUSTER
    ├─ user:US:*       ├─ user:EU:*        ├─ user:ASIA:*
    ├─ config:global   ├─ config:global    ├─ config:global
    └─ (no EU data)    └─ (no ASIA data)   └─ (no US data)


Réplication :
    config:global → tous les clusters
    user:* → seulement cluster d'origine


Avantages :
├─ Isolation géographique (RGPD)
├─ Bandwidth optimisé
└─ Scalabilité indépendante par région

Inconvénients :
├─ Complexité applicative (routing par région)
└─ Utilisateur fixé à sa région

Cas d'usage :
├─ SaaS multi-tenant
├─ Conformité réglementaire stricte
└─ Données régionalisées


PATTERN 4 : CASCADING REPLICATION
══════════════════════════════════

Réplication en cascade pour réduire charge hub


    PRIMARY ──→ SECONDARY-1 ──→ SECONDARY-2
                     │
                     └──────────→ SECONDARY-3


Avantages :
├─ Réduit charge réseau sur Primary
└─ Scalabilité réplication

Inconvénients :
├─ Latence accrue (multi-hop)
├─ RPO augmenté
└─ Complexité

Cas d'usage :
├─ 5+ DCs
└─ Primary avec bandwidth limité
```

## Latence et performance

### Analyse de latence WAN

```
┌─────────────────────────────────────────────────────────────┐
│                  Latence inter-datacenter                   │
└─────────────────────────────────────────────────────────────┘

Latences typiques (round-trip) :
════════════════════════════════

Même région (ex: US-EAST-1A ↔ US-EAST-1B)
    Latence : 1-3ms
    Bande passante : 100 Gbps+

Régions proches (ex: US-EAST ↔ US-WEST)
    Latence : 60-80ms
    Bande passante : 10-100 Gbps

Continents (ex: US ↔ Europe)
    Latence : 80-120ms
    Bande passante : 1-10 Gbps

Antipodes (ex: US ↔ Australie)
    Latence : 180-250ms
    Bande passante : 1-5 Gbps


Impact sur réplication :
════════════════════════

Réplication asynchrone :
    └─> Latence WAN n'impacte PAS les écritures locales ✓
    └─> Mais crée lag de réplication

Exemple :
    Write sur US-EAST à t=0
    └─> Répliqué vers EU-WEST à t=100ms
        └─> Fenêtre de 100ms où EU-WEST a ancienne valeur


Calcul du lag :
═══════════════

Lag = Latency_WAN + Processing_time + Queue_time

Latency_WAN : 100ms (US ↔ EU)
Processing : 5ms (sérialisation/désérialisation)
Queue : 10ms (buffer réplication)
────────────────────────
Total lag : 115ms


Sous forte charge :
    Queue_time peut atteindre plusieurs secondes
    └─> Lag total = 100ms + 5ms + 5000ms = 5.1 secondes !


Optimisations :
═══════════════

1. COMPRESSION
   ├─ Réduire volume → moins de temps transfert
   └─> Gain 50-70% bandwidth

2. BATCHING
   ├─ Regrouper plusieurs ops en un batch
   └─> Amortir overhead réseau

3. PIPELINING
   ├─ Envoyer batch suivant avant ACK du précédent
   └─> Utiliser pleine bande passante

4. DEDICATED LINKS (VPN, Direct Connect)
   ├─ Éviter internet public
   └─> Latence plus stable, bandwidth garanti
```

### Optimisation de la bande passante

```bash
# ═══════════════════════════════════════════════════════════
# OPTIMISATIONS BANDWIDTH POUR XDR
# ═══════════════════════════════════════════════════════════


# TECHNIQUE 1 : COMPRESSION
# ──────────────────────────

# Redis Enterprise : Compression native
# Niveau 6 (défaut) : Bon compromis CPU/compression

# Comparaison :
Sans compression : 1 GB/heure → 1 GB transfert
Avec compression (ratio 70%) : 1 GB/heure → 300 MB transfert
Économie : 700 MB/heure = 16.8 GB/jour


# TECHNIQUE 2 : DELTA SYNC
# ─────────────────────────

# N'envoyer que les changements, pas les valeurs complètes

# Sans delta :
SET large_object "{...5MB...}"  → Réplication 5 MB

# Avec delta (Redis Enterprise) :
SET large_object "{...5MB...}"
HSET large_object field1 "new"  → Réplication 10 bytes seulement


# TECHNIQUE 3 : SELECTIVE REPLICATION
# ────────────────────────────────────

# Répliquer seulement certaines clés vers certains DCs

# Configuration Redis Enterprise :
curl -X POST .../crdbs/config \
    -d '{
        "replication_rules": [
            {
                "pattern": "user:EU:*",
                "replicate_to": ["eu-west"],
                "exclude_from": ["us-east", "asia"]
            },
            {
                "pattern": "config:*",
                "replicate_to": ["all"]
            }
        ]
    }'


# TECHNIQUE 4 : THROTTLING / RATE LIMITING
# ─────────────────────────────────────────

# Limiter débit réplication pour ne pas saturer WAN

# Redis Enterprise :
curl -X PUT .../crdbs/config \
    -d '{
        "replication_bandwidth_limit": "100MB/s"
    }'

# Utile si :
├─ Bandwidth partagé avec autres applications
├─ Coût facturé au volume
└─ Éviter saturation lors de catch-up après panne


# TECHNIQUE 5 : ASYNC WRITES (Write-Behind)
# ──────────────────────────────────────────

# Buffer écritures localement, flush par batch

# Sans buffering :
1000 writes/sec × 1 KB/write = 1 MB/sec bandwidth

# Avec buffering (flush toutes les 100ms) :
100 writes buffered × 10 flushes/sec = même throughput
Mais moins de packets, moins d'overhead TCP


# TECHNIQUE 6 : COALESCING
# ─────────────────────────

# Fusionner écritures multiples sur même clé

# Sans coalescing :
SET key "v1"  → Réplication
SET key "v2"  → Réplication
SET key "v3"  → Réplication
Total : 3 réplications

# Avec coalescing :
SET key "v1"
SET key "v2"
SET key "v3"
└─> Réplication : SET key "v3" seulement
Total : 1 réplication


# MONITORING BANDWIDTH
# ────────────────────

# Prometheus query pour bandwidth WAN
rate(redis_net_output_bytes_total{role="master"}[1m]) * 8 / 1000000

# Alerte si > 80% capacité lien
```

## Disaster Recovery et procédures

### Procédure DR pour Active-Active

```bash
# ═══════════════════════════════════════════════════════════
# DISASTER RECOVERY : PERTE COMPLÈTE D'UN DC (ACTIVE-ACTIVE)
# ═══════════════════════════════════════════════════════════

# SCÉNARIO : DC ASIA-PACIFIC complètement détruit
#            DCs US-EAST et EU-WEST toujours actifs


# ÉTAPE 1 : DÉTECTION (automatique)
# ──────────────────────────────────

# Health checks détectent perte de connectivité vers ASIA

# Logs :
[2024-01-15 14:00:00] ERROR: ASIA-PAC unreachable
[2024-01-15 14:00:05] ERROR: Replication to ASIA-PAC failed
[2024-01-15 14:00:10] CRITICAL: ASIA-PAC datacenter DOWN


# ÉTAPE 2 : ÉVALUATION IMMÉDIATE
# ───────────────────────────────

# Impact sur service :
├─ Utilisateurs ASIA : Affectés (DC local down)
├─ Utilisateurs US/EU : Non affectés (DCs actifs)

# État des données :
├─ US-EAST : OK, contient toutes données répliquées avant panne
├─ EU-WEST : OK, contient toutes données répliquées avant panne
└─ ASIA-PAC : PERDU (assume hardware destroyed)


# ÉTAPE 3 : ROUTER TRAFIC ASIA VERS AUTRE DC
# ───────────────────────────────────────────

# Option A : Route vers DC géographiquement proche

# Via DNS (GeoDNS)
aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890ABC \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "redis-asia.example.com",
                "Type": "CNAME",
                "TTL": 60,
                "GeoLocation": {
                    "ContinentCode": "AS"
                },
                "ResourceRecords": [
                    {"Value": "redis-eu-west.example.com"}
                ]
            }
        }]
    }'

# Utilisateurs ASIA maintenant routés vers EU-WEST
# Latence augmentée (300ms au lieu de 10ms) mais service maintenu


# Option B : Global Load Balancer avec health checks

# AWS Global Accelerator / Cloudflare Load Balancer
# Détecte automatiquement ASIA down, route vers backup


# ÉTAPE 4 : DÉSACTIVER RÉPLICATION VERS ASIA
# ───────────────────────────────────────────

# Éviter tentatives de réplication vers DC mort

# Redis Enterprise API :
curl -X DELETE https://redis-api/v1/crdbs/global-db/participants/asia-cluster


# ÉTAPE 5 : ÉVALUER PERTE DE DONNÉES
# ───────────────────────────────────

# RPO dépend du lag de réplication avant panne

# Vérifier dernière réplication réussie
curl https://redis-api/v1/crdbs/global-db/replication_log | jq

# Output :
{
  "last_successful_sync": {
    "asia_to_us": "2024-01-15 13:59:58",
    "asia_to_eu": "2024-01-15 13:59:59"
  },
  "failure_detected": "2024-01-15 14:00:00"
}

# Perte estimée : 1-2 secondes de données écrites sur ASIA
# avant destruction


# ÉTAPE 6 : RECONSTRUIRE DC ASIA (après incident)
# ────────────────────────────────────────────────

# Une fois nouveau hardware disponible :

# 1. Provisionner nouveau cluster Redis sur ASIA-NEW
#    (machines, réseau, Redis Enterprise)

# 2. Réintégrer ASIA-NEW dans CRDB
curl -X POST https://redis-api/v1/crdbs/global-db/participants \
    -d '{
        "cluster": {
            "url": "https://asia-new.redis.example.com",
            "name": "asia-new-cluster"
        }
    }'

# 3. Initial sync (peut prendre plusieurs heures)
#    CRDB copie état complet de US ou EU vers ASIA-NEW

# 4. Une fois sync terminé, ASIA-NEW devient actif
#    Trafic automatiquement routé vers ASIA-NEW

# 5. Nettoyer l'ancien ASIA (si nécessaire)


# ÉTAPE 7 : POST-MORTEM
# ──────────────────────

cat <<EOF > /docs/postmortem-asia-dc-loss.md
# Post-Mortem : Perte DC ASIA-PACIFIC

## Incident Summary
Date: 2024-01-15 14:00 UTC
Duration: 6 hours (until traffic fully restored)
Impact: 10% users (ASIA region) experienced degraded service

## Timeline
- 13:59:58 : Last successful replication from ASIA
- 14:00:00 : ASIA DC unresponsive (power failure confirmed)
- 14:02:30 : Traffic rerouted to EU-WEST for ASIA users
- 14:05:00 : Replication to ASIA disabled in CRDB
- 20:00:00 : New ASIA-NEW cluster provisioned
- 23:30:00 : Initial sync completed, ASIA-NEW active

## Data Loss
Estimated: 1-2 seconds of writes on ASIA before failure
RPO met: < 5 seconds (SLA: < 60 seconds)

## What Went Well
✓ Active-Active design prevented global outage
✓ Automatic health checks detected failure quickly
✓ Runbook followed successfully
✓ 90% of users unaffected

## What Could Be Improved
✗ Manual traffic rerouting took 2.5 minutes
  → Action: Implement automatic GeoDNS failover
✗ Initial sync to ASIA-NEW took 3.5 hours
  → Action: Pre-warm standby DC with periodic snapshots

## Action Items
1. [P0] Implement automated failover (owner: SRE team, due: 2024-02-01)
2. [P1] Setup standby ASIA DC with daily snapshots (owner: Ops, due: 2024-02-15)
3. [P2] Improve monitoring alerts for WAN replication lag (owner: DevOps, due: 2024-02-28)
EOF
```

### Tests de DR (Chaos Engineering)

```bash
# ═══════════════════════════════════════════════════════════
# TESTS DISASTER RECOVERY (GAME DAYS)
# ═══════════════════════════════════════════════════════════

# Objectif : Valider que les procédures DR fonctionnent
#            Entraîner l'équipe
#            Identifier faiblesses avant incident réel


# TEST 1 : FAILOVER SIMPLE (ACTIVE-PASSIVE)
# ──────────────────────────────────────────

#!/bin/bash
# test-failover-active-passive.sh

echo "=== DR Test : Active-Passive Failover ==="

# 1. Vérifier état initial
echo "Step 1: Verify initial state"
redis-cli -h primary.example.com PING || exit 1
redis-cli -h secondary.example.com PING || exit 1

# 2. Injecter panne sur Primary (simulé)
echo "Step 2: Simulate Primary failure"
ssh primary.example.com "sudo systemctl stop redis"

# 3. Vérifier détection panne
echo "Step 3: Wait for failure detection (30s)"
sleep 30

# 4. Déclencher failover
echo "Step 4: Trigger failover to Secondary"
/usr/local/bin/failover-to-secondary.sh

# 5. Vérifier traffic sur Secondary
echo "Step 5: Verify traffic on Secondary"
for i in {1..10}; do
    redis-cli -h redis.example.com INCR test:failover:counter
done

count=$(redis-cli -h secondary.example.com GET test:failover:counter)
if [ "$count" -eq 10 ]; then
    echo "✓ Failover successful"
else
    echo "✗ Failover failed"
    exit 1
fi

# 6. Restaurer Primary
echo "Step 6: Restore Primary"
ssh primary.example.com "sudo systemctl start redis"

# 7. Rapport
echo "=== Test Complete ==="
echo "Downtime: ~60 seconds"
echo "Data loss: 0 keys"


# TEST 2 : PERTE D'UN DC (ACTIVE-ACTIVE)
# ───────────────────────────────────────

#!/bin/bash
# test-dc-loss-active-active.sh

echo "=== DR Test : Active-Active DC Loss ==="

TARGET_DC="asia"

# 1. Baseline metrics
echo "Collecting baseline metrics..."
BASELINE_QPS=$(curl -s http://prometheus:9090/api/v1/query?query=redis_commands_per_sec | jq '.data.result[0].value[1]')

# 2. Simuler perte DC ASIA (network partition)
echo "Simulating loss of DC: $TARGET_DC"
ssh asia-firewall "iptables -A OUTPUT -j DROP"

# 3. Attendre routing automatique (ou manuel)
echo "Waiting for traffic rerouting..."
sleep 60

# 4. Vérifier service toujours up
echo "Verifying service availability..."
for dc in us-east eu-west; do
    redis-cli -h $dc.redis.example.com PING || echo "✗ $dc unreachable"
done

# 5. Vérifier QPS total
CURRENT_QPS=$(curl -s http://prometheus:9090/api/v1/query?query=redis_commands_per_sec | jq '.data.result[0].value[1]')
echo "QPS: Baseline=$BASELINE_QPS, Current=$CURRENT_QPS"

# 6. Restaurer ASIA
echo "Restoring $TARGET_DC..."
ssh asia-firewall "iptables -F"

# 7. Attendre resync
echo "Waiting for resync..."
sleep 300

# 8. Rapport
echo "=== Test Complete ==="
echo "Service remained available: ✓"
echo "Estimated downtime for ASIA users: 60s"


# TEST 3 : SPLIT-BRAIN (cas critique)
# ────────────────────────────────────

#!/bin/bash
# test-split-brain.sh

echo "=== DR Test : Split-Brain Scenario ==="

# Simuler partition réseau isolant EU-WEST
echo "Creating network partition..."
ssh eu-west-firewall "iptables -A INPUT -s us-east-subnet -j DROP"
ssh eu-west-firewall "iptables -A INPUT -s asia-subnet -j DROP"

# EU-WEST ne peut plus communiquer avec US/ASIA
# Mais EU-WEST continue de servir clients locaux

# Écriture concurrente sur même clé
redis-cli -h us-east SET critical:key "value_from_us"
redis-cli -h eu-west SET critical:key "value_from_eu"

echo "Split-brain created. Monitoring for 60s..."
sleep 60

# Restaurer réseau
ssh eu-west-firewall "iptables -F"

echo "Network restored. Checking conflict resolution..."
sleep 30

# Vérifier convergence
us_value=$(redis-cli -h us-east GET critical:key)
eu_value=$(redis-cli -h eu-west GET critical:key)

if [ "$us_value" == "$eu_value" ]; then
    echo "✓ Conflict resolved, values converged: $us_value"
else
    echo "✗ Split-brain persists! US=$us_value, EU=$eu_value"
fi
```

## Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│         Best Practices : Cross-Datacenter Replication       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ARCHITECTURE                                                │
│ ════════════                                                │
│                                                             │
│ ✓ Choisir pattern adapté au use case                        │
│   ├─ Active-Passive : DR simple, pas de conflits            │
│   └─ Active-Active : Latence minimale, HA maximale          │
│                                                             │
│ ✓ Dimensionner bande passante WAN généreusement             │
│   └─> 2-3x charge moyenne pour absorber pics                │
│                                                             │
│ ✓ Utiliser liens dédiés (VPN, Direct Connect)               │
│   └─> Éviter internet public pour latence stable            │
│                                                             │
│ ✓ Prévoir compression (50-70% gain bandwidth)               │
│                                                             │
│ ✓ Tester régulièrement failover (game days)                 │
│   └─> Au moins trimestriellement                            │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ RÉPLICATION                                                 │
│ ═══════════                                                 │
│                                                             │
│ ✓ Toujours asynchrone pour WAN (latence inacceptable si sync)
│                                                             │
│ ✓ Monitorer lag de réplication en continu                   │
│   └─> Alertes si lag > seuil (ex: 5s)                       │
│                                                             │
│ ✓ Buffer/Batch pour optimiser bandwidth                     │
│                                                             │
│ ✓ Réplication sélective si possible                         │
│   └─> Ne répliquer que données nécessaires par région       │
│                                                             │
│ ✓ Prévoir retry automatique en cas d'échec réseau           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ COHÉRENCE                                                   │
│ ══════════                                                  │
│                                                             │
│ ✓ Comprendre et accepter eventual consistency               │
│                                                             │
│ ✓ Utiliser CRDT si Active-Active (Redis Enterprise)         │
│                                                             │
│ ✓ Implémenter idempotence dans l'application                │
│   └─> Gérer réception multiple de même event                │
│                                                             │
│ ✓ Timestamping précis (NTP synchronisé)                     │
│   └─> Essentiel pour LWW                                    │
│                                                             │
│ ✓ Éviter opérations non-commutatives sur Active-Active      │
│   └─> Ex: SET OK, APPEND risqué                             │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ MONITORING                                                  │
│ ══════════                                                  │
│                                                             │
│ ✓ Métriques par DC ET globales                              │
│                                                             │
│ ✓ Dashboards séparés par région                             │
│                                                             │
│ ✓ Alertes multi-niveaux :                                   │
│   ├─ P1: DC complètement down                               │
│   ├─ P2: Lag > 10s                                          │
│   └─ P3: Bandwidth > 80%                                    │
│                                                             │
│ ✓ Distributed tracing (OpenTelemetry)                       │
│   └─> Suivre requête cross-DC                               │
│                                                             │
│ ✓ Runbooks à jour et testés                                 │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ COÛTS                                                       │
│ ═════                                                       │
│                                                             │
│ ✓ Budgeter correctement :                                   │
│   ├─ Infrastructure : N × coût single-DC                    │
│   ├─ Bandwidth : Facturé au volume                          │
│   └─ Licences : Redis Enterprise si Active-Active           │
│                                                             │
│ ✓ Optimiser bandwidth pour réduire coûts                    │
│   └─> Compression, batching, selective replication          │
│                                                             │
│ ✓ Dimensionner DCs selon charge régionale                   │
│   └─> Ex: US 50%, EU 30%, ASIA 20%                          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ SÉCURITÉ                                                    │
│ ════════                                                    │
│                                                             │
│ ✓ Chiffrer réplication WAN (TLS)                            │
│                                                             │
│ ✓ Authentification mutuelle entre DCs                       │
│                                                             │
│ ✓ Firewall : Whitelist IPs uniquement                       │
│                                                             │
│ ✓ Conformité réglementaire (RGPD, etc.)                     │
│   └─> Residence des données par région                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Conclusion

La réplication cross-datacenter représente le sommet de la complexité architecturale pour Redis Cluster, offrant une résilience maximale au prix d'une complexité opérationnelle significative. Le choix entre Active-Passive et Active-Active doit être guidé par les besoins réels :

**Active-Passive** convient pour :
- Disaster Recovery simple
- Applications tolérant latence géographique
- Budgets limités
- Équipes de taille modeste

**Active-Active** convient pour :
- Applications globales nécessitant latence minimale
- Haute disponibilité critique (99.999%+)
- Capacité à investir dans infrastructure et licences
- Équipes matures avec expertise systèmes distribués

Dans tous les cas, une approche rigoureuse de testing (game days), monitoring continu, et documentation exhaustive sont essentiels pour maintenir un système XDR stable et performant.

---

**Points clés à retenir :**

- **Active-Passive** : Simple, un seul master, pas de conflits
- **Active-Active** : Latence minimale, HA maximale, gestion conflits nécessaire
- **Réplication WAN** : Toujours asynchrone (latence inacceptable si sync)
- **CRDT** : Solution mathématique pour convergence sans conflits (Redis Enterprise)
- **LWW** : Last-Write-Wins, simple mais peut perdre données
- **Lag** : Latency + Processing + Queue, monitorer en continu
- **Compression** : 50-70% gain bandwidth, essentiel pour XDR
- **Tests DR** : Game days réguliers, chaos engineering
- **Coût** : 3-5x infrastructure single-DC, budgéter appropriément

Le module 11 "Architecture Distribuée et Scaling Horizontal" est maintenant complet avec cette section finale sur la réplication géographique.

⏭️ [Redis en Production et Sécurité](/12-redis-production-securite/README.md)
