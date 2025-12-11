🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 10 : Architecture Haute Disponibilité (DevOps)

## Vue d'ensemble

L'architecture haute disponibilité (HA - High Availability) est un pilier fondamental pour tout système Redis déployé en production. Ce module explore les mécanismes, patterns et stratégies permettant de garantir la continuité de service face aux défaillances matérielles, réseau ou logicielles.

**Niveau** : Avancé
**Public cible** : DevOps, Architectes, SRE
**Prérequis** : Connaissance solide de Redis Core, Linux, networking

---

## Objectifs du module

À l'issue de ce module, vous maîtriserez :

1. **Les mécanismes de réplication** Master-Replica et leurs implications
2. **Redis Sentinel** pour le monitoring et le failover automatique
3. **Les topologies de haute disponibilité** et leurs trade-offs
4. **La gestion des situations critiques** (split-brain, quorum, network partitions)
5. **L'intégration applicative** via Service Discovery
6. **Les tests de résilience** et procédures de basculement

---

## 🎯 Contexte et enjeux de la haute disponibilité

### Le défi de la disponibilité avec Redis

Redis, de par sa nature in-memory et single-threaded, présente des caractéristiques uniques en matière de haute disponibilité :

- **Vitesse vs Durabilité** : La performance exceptionnelle de Redis repose sur l'absence de disque dans le chemin critique
- **Single Point of Failure** : Une instance Redis standalone est un SPOF par définition
- **Latence réseau** : La réplication asynchrone introduit un délai incompressible
- **Cohérence vs Disponibilité** : Le théorème CAP s'applique pleinement

### Les 9's de disponibilité

| Disponibilité | Downtime annuel | Downtime mensuel | Cas d'usage |
|---------------|-----------------|------------------|-------------|
| 99% (2 nines) | 3.65 jours | 7.2 heures | Non acceptable en production |
| 99.9% (3 nines) | 8.76 heures | 43.2 minutes | Applications standard |
| 99.95% | 4.38 heures | 21.6 minutes | E-commerce, services B2B |
| 99.99% (4 nines) | 52.56 minutes | 4.32 minutes | Services critiques |
| 99.999% (5 nines) | 5.26 minutes | 25.9 secondes | Services financiers, santé |

**Pour Redis** : Atteindre 99.9% est réaliste avec Sentinel, 99.99% nécessite Redis Cluster + multi-AZ.

---

## 📊 Les trois piliers de la HA

### 1. Redondance (Redundancy)

Éliminer les points de défaillance uniques :

```
┌─────────────────────────────────────────────────┐
│          SANS REDONDANCE (SPOF)                 │
├─────────────────────────────────────────────────┤
│                                                 │
│         Application                             │
│              │                                  │
│              ▼                                  │
│         ┌─────────┐                             │
│         │  Redis  │  ◄── Single Point of Failure│
│         │ Master  │                             │
│         └─────────┘                             │
│                                                 │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│          AVEC REDONDANCE                        │
├─────────────────────────────────────────────────┤
│                                                 │
│         Application                             │
│              │                                  │
│              ▼                                  │
│         ┌─────────┐                             │
│         │  Redis  │                             │
│         │ Master  │─────────┐                   │
│         └─────────┘         │ Replication       │
│                             ▼                   │
│                      ┌─────────┐                │
│                      │  Redis  │                │
│                      │ Replica │                │
│                      └─────────┘                │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 2. Détection de pannes (Failure Detection)

Identifier rapidement les défaillances via :

- **Health checks actifs** : Ping/PONG, INFO replication
- **Monitoring passif** : Métriques système, logs
- **Consensus distribué** : Quorum entre Sentinels
- **Délais configurables** : down-after-milliseconds

### 3. Basculement (Failover)

Promouvoir automatiquement un replica en master :

```
┌────────────────────────────────────────────────┐
│         PROCESSUS DE FAILOVER                  │
├────────────────────────────────────────────────┤
│                                                │
│  Étape 1: Détection de panne                   │
│  ┌────────┐                                    │
│  │ Master │ ✗ DOWN                             │
│  └────────┘                                    │
│       │                                        │
│       │                                        │
│  ┌────────┐  ┌────────┐  ┌────────┐            │
│  │Sentinel│  │Sentinel│  │Sentinel│ ◄─ Quorum  │
│  │   1    │  │   2    │  │   3    │            │
│  └────────┘  └────────┘  └────────┘            │
│                                                │
│  Étape 2: Élection du nouveau master           │
│  ┌────────┐  ┌────────┐                        │
│  │Replica1│  │Replica2│                        │
│  └────────┘  └────────┘                        │
│       ▲           │                            │
│       │ Élu       │                            │
│       │           ▼                            │
│  ┌────────┐  ┌────────┐                        │
│  │  NEW   │  │Replica │                        │
│  │ MASTER │  │  of    │                        │
│  │        │──│  NEW   │                        │
│  └────────┘  │ MASTER │                        │
│              └────────┘                        │
│                                                │
│  Étape 3: Reconfiguration clients              │
│  Applications pointent vers nouveau master     │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 🏗️ Architectures de haute disponibilité

### Architecture 1 : Master-Replica simple

**Configuration minimale** pour la lecture scalable et backup à chaud.

```
┌─────────────────────────────────────────────────┐
│  ARCHITECTURE MASTER-REPLICA SIMPLE             │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────┐                               │
│  │ Application  │                               │
│  └───────┬──────┘                               │
│          │                                      │
│          │ Writes + Reads                       │
│          ▼                                      │
│    ┌──────────┐                                 │
│    │  Master  │ :6379                           │
│    │  (RW)    │                                 │
│    └─────┬────┘                                 │
│          │ Replication                          │
│          │ (async)                              │
│          ▼                                      │
│    ┌──────────┐         ┌──────────┐            │
│    │ Replica1 │         │ Replica2 │            │
│    │  (R)     │:6380    │  (R)     │:6381       │
│    └──────────┘         └──────────┘            │
│          ▲                    ▲                 │
│          │                    │                 │
│          └────────┬───────────┘                 │
│                   │ Read queries                │
│          ┌────────┴────────┐                    │
│          │  Load Balancer  │                    │
│          │   (Optional)    │                    │
│          └─────────────────┘                    │
│                                                 │
└─────────────────────────────────────────────────┘

✅ Avantages:
- Simple à configurer
- Scalabilité en lecture
- Backup à chaud

❌ Inconvénients:
- Pas de failover automatique
- SPOF sur le master
- Intervention manuelle nécessaire
```

**Configuration Master** (`redis-master.conf`) :

```ini
# Binding et port
bind 0.0.0.0
port 6379
protected-mode yes

# Authentification
requirepass "MyStrongMasterPassword123!"
masterauth "MyStrongMasterPassword123!"  # Pour re-sync après failover

# Réplication
replica-read-only yes
repl-diskless-sync yes
repl-diskless-sync-delay 5

# Persistance (RDB + AOF recommandé)
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec

# Mémoire
maxmemory 4gb
maxmemory-policy allkeys-lru

# Sécurité
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_a8f5f167f44f4964e6c998dee827110c"

# Logs
loglevel notice
logfile "/var/log/redis/redis-master.log"

# Performance
tcp-backlog 511
timeout 300
tcp-keepalive 60
```

**Configuration Replica** (`redis-replica.conf`) :

```ini
# Binding et port
bind 0.0.0.0
port 6380  # Port différent pour Replica1, 6381 pour Replica2

# Authentification
requirepass "MyStrongReplicaPassword123!"
masterauth "MyStrongMasterPassword123!"

# Réplication - CRUCIAL
replicaof 10.0.1.10 6379  # IP du master
replica-read-only yes
replica-priority 100  # Plus bas = plus prioritaire pour promotion

# Paramètres de réplication
repl-diskless-sync yes
repl-diskless-sync-delay 5
repl-backlog-size 256mb
repl-backlog-ttl 3600
min-replicas-to-write 1  # Master refuse writes si <1 replica connecté
min-replicas-max-lag 10   # Max 10 sec de lag

# Persistance (même config que master)
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec

# Mémoire
maxmemory 4gb
maxmemory-policy allkeys-lru

# Logs
loglevel notice
logfile "/var/log/redis/redis-replica1.log"
```

---

### Architecture 2 : Master-Replica + Sentinel (Recommandé)

**Configuration production standard** avec failover automatique.

```
┌───────────────────────────────────────────────────────────┐
│  ARCHITECTURE MASTER-REPLICA + SENTINEL                   │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐                                         │
│  │ Application  │                                         │
│  └───────┬──────┘                                         │
│          │                                                │
│          │ Sentinel Discovery                             │
│          │ (ask for current master)                       │
│          ▼                                                │
│    ┌──────────┐     ┌──────────┐     ┌──────────┐         │
│    │Sentinel 1│     │Sentinel 2│     │Sentinel 3│         │
│    │:26379    │────▶│:26379    │◀────│:26379    │         │
│    └────┬─────┘     └────┬─────┘     └────┬─────┘         │
│         │                │                │               │
│         │ Monitor        │ Monitor        │ Monitor       │
│         ▼                ▼                ▼               │
│    ┌─────────────────────────────────────────┐            │
│    │                                         │            │
│    │    ┌──────────┐                         │            │
│    │    │  Master  │ :6379                   │            │
│    │    │  (RW)    │                         │            │
│    │    └─────┬────┘                         │            │
│    │          │ Replication                  │            │
│    │          │                              │            │
│    │          ▼                              │            │
│    │    ┌──────────┐         ┌──────────┐    │            │
│    │    │ Replica1 │         │ Replica2 │    │            │
│    │    │  (R)     │:6380    │  (R)     │:6381            │
│    │    └──────────┘         └──────────┘    │            │
│    │                                         │            │
│    └─────────────────────────────────────────┘            │
│             Redis Replication Set                         │
│                                                           │
│  Caractéristiques:                                        │
│  • Quorum = 2 (minimum de Sentinels pour failover)        │
│  • Failover automatique en 30-60 secondes                 │
│  • Service Discovery intégré                              │
│  • Split-brain protection via quorum                      │
│                                                           │
└───────────────────────────────────────────────────────────┘

✅ Avantages:
- Failover automatique (RTO ~30-60s)
- Service Discovery
- Monitoring intégré
- Protection split-brain

❌ Inconvénients:
- Complexité accrue (3+ sentinels)
- Pas de scaling horizontal (writes)
- Réplication asynchrone (possible perte)
```

**Configuration Sentinel** (`sentinel.conf`) :

```ini
# Port Sentinel
port 26379
bind 0.0.0.0

# Persistance de la configuration (Sentinel réécrit ce fichier)
dir /var/lib/redis/sentinel

# Monitoring du master
# sentinel monitor <master-name> <ip> <port> <quorum>
sentinel monitor mymaster 10.0.1.10 6379 2

# Authentification
sentinel auth-pass mymaster MyStrongMasterPassword123!
sentinel auth-user mymaster default

# Détection de panne
sentinel down-after-milliseconds mymaster 30000  # 30 secondes
sentinel parallel-syncs mymaster 1  # 1 replica à la fois pour re-sync
sentinel failover-timeout mymaster 180000  # 3 minutes

# Notifications (optionnel)
sentinel notification-script mymaster /var/redis/notify.sh
sentinel client-reconfig-script mymaster /var/redis/reconfig.sh

# Sentinel configuration rewrite (ne pas modifier)
# Les lignes suivantes sont automatiquement gérées par Sentinel

# ACL (Redis 6+)
# sentinel sentinel-user <username>
# sentinel sentinel-pass <password>

# Logs
loglevel notice
logfile "/var/log/redis/sentinel.log"

# Configuration avancée
sentinel deny-scripts-reconfig yes  # Sécurité
sentinel resolve-hostnames no  # Utiliser IPs
sentinel announce-hostnames no
```

**Topologie de déploiement recommandée** :

```
┌────────────────────────────────────────────────────────────────┐
│  DÉPLOIEMENT MULTI-AZ POUR HAUTE DISPONIBILITÉ                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌───────────────────┐  ┌───────────────────┐  ┌────────────┐  │
│  │   AZ-1 (Zone A)   │  │   AZ-2 (Zone B)   │  │ AZ-3       │  │
│  ├───────────────────┤  ├───────────────────┤  ├────────────┤  │
│  │                   │  │                   │  │            │  │
│  │  ┌────────────┐   │  │  ┌────────────┐   │  │ ┌────────┐ │  │
│  │  │   Master   │   │  │  │  Replica 1 │   │  │ │Replica │ │
│  │  │   :6379    │───┼──┼─▶│   :6379    │   │  │ │  2     │ │
│  │  └────────────┘   │  │  └────────────┘   │  │ │:6379   │ │
│  │         ▲         │  │         ▲         │  │ └────────┘ │  │
│  │         │         │  │         │         │  │     ▲      │  │
│  │  ┌──────┴─────┐   │  │  ┌──────┴─────┐   │  │ ┌───┴────┐ │  │
│  │  │ Sentinel 1 │   │  │  │ Sentinel 2 │   │  │ │Sentinel│ │
│  │  │  :26379    │◀──┼──┼─▶│  :26379    │◀──┼──┼▶│   3    │ │
│  │  └────────────┘   │  │  └────────────┘   │  │ │:26379  │ │
│  │                   │  │                   │  │ └────────┘ │  │
│  └───────────────────┘  └───────────────────┘  └────────────┘  │
│                                                                │
│  Configuration:                                                │
│  • 3 zones de disponibilité (AWS AZ / Azure zones)             │
│  • 1 Master + 2 Replicas (1 par AZ)                            │
│  • 3 Sentinels (1 par AZ)                                      │
│  • Quorum = 2                                                  │
│  • Résistance à la panne d'une AZ complète                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

### Architecture 3 : Multi-Master (Active-Active)

⚠️ **Redis Core ne supporte pas le multi-master natif**. Solutions :
- Redis Enterprise (commercial)
- External tools : CRDT-based replication
- Application-level conflict resolution

```
┌────────────────────────────────────────────────────────────┐
│  ARCHITECTURE ACTIVE-ACTIVE (Redis Enterprise uniquement)  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│   Datacenter 1 (Paris)         Datacenter 2 (Frankfurt)    │
│  ┌─────────────────────┐      ┌─────────────────────┐      │
│  │                     │      │                     │      │
│  │  ┌────────────┐     │      │    ┌────────────┐   │      │
│  │  │  Master A  │     │      │    │  Master B  │   │      │
│  │  │   (RW)     │◀────┼──────┼───▶│   (RW)     │   │      │
│  │  └────────────┘     │      │    └────────────┘   │      │
│  │         ▲           │ CRDT │         ▲           │      │
│  │         │           │ sync │         │           │      │
│  │         │           │      │         │           │      │
│  │  ┌──────┴──────┐    │      │  ┌──────┴──────┐    │      │
│  │  │  Replica A  │    │      │  │  Replica B  │    │      │
│  │  └─────────────┘    │      │  └─────────────┘    │      │
│  │                     │      │                     │      │
│  └─────────────────────┘      └─────────────────────┘      │
│           ▲                            ▲                   │
│           │                            │                   │
│      Applications                 Applications             │
│       (Zone 1)                     (Zone 2)                │
│                                                            │
│  Caractéristiques:                                         │
│  • Writes dans les 2 datacenters                           │
│  • Résolution de conflits via CRDT                         │
│  • Latence locale pour les reads/writes                    │
│  • Haute résilience géographique                           │
│                                                            │
│  ⚠️  Nécessite Redis Enterprise ou solutions tierces       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 📐 Matrice de décision architecturale

| Critère | Master-Replica | Sentinel | Cluster | Active-Active |
|---------|---------------|----------|---------|---------------|
| **Failover auto** | ❌ Manuel | ✅ Oui (~30-60s) | ✅ Oui (~10-30s) | ✅ Oui (instantané) |
| **Scalabilité writes** | ❌ Non | ❌ Non | ✅ Oui | ✅ Oui |
| **Scalabilité reads** | ✅ Oui | ✅ Oui | ✅ Oui | ✅ Oui |
| **Complexité setup** | ⭐ Simple | ⭐⭐ Moyenne | ⭐⭐⭐ Élevée | ⭐⭐⭐⭐ Très élevée |
| **Cohérence** | Éventuelle | Éventuelle | Éventuelle | Faible (CRDT) |
| **Coût** | Faible | Moyen | Moyen-Élevé | Élevé |
| **Perte de données** | Possible | Possible (async) | Possible (async) | Minimale |
| **Cross-DC** | ✅ Possible | ✅ Possible | ⚠️ Latence | ✅ Optimal |
| **Multi-key ops** | ✅ Oui | ✅ Oui | ⚠️ Limité | ⚠️ Limité |
| **RTO (Recovery)** | Heures | 30-60s | 10-30s | <1s |
| **RPO (Data loss)** | Dernière sauvegarde | Secondes | Secondes | <1s |

**Légende** :
- RTO : Recovery Time Objective (temps de rétablissement)
- RPO : Recovery Point Objective (perte de données acceptable)

---

## 🎯 Recommandations par cas d'usage

### 1. Application Web Standard (99.9% uptime)
**Architecture** : Master + 2 Replicas + 3 Sentinels

```
Configuration:
- 1 Master (writes)
- 2 Replicas (reads + failover)
- 3 Sentinels (quorum = 2)
- Multi-AZ deployment
- RDB + AOF persistence

Coût: Moyen
Complexité: Moyenne
RTO: 30-60s
RPO: Quelques secondes
```

### 2. Service Critique (99.99% uptime)
**Architecture** : Cluster Multi-AZ

```
Configuration:
- Redis Cluster (3+ masters)
- 1 replica par master minimum
- Multi-AZ deployment
- Monitoring avancé
- AOF persistence

Coût: Élevé
Complexité: Élevée
RTO: 10-30s
RPO: Secondes
```

### 3. Application Globale Multi-Région
**Architecture** : Active-Active Cross-Region

```
Configuration:
- Redis Enterprise ou solutions custom
- 2+ datacenters géographiques
- CRDT-based replication
- Global load balancing

Coût: Très élevé
Complexité: Très élevée
RTO: <1s
RPO: <1s
```

### 4. Cache Non-Critique (99% uptime)
**Architecture** : Master + 1 Replica (sans Sentinel)

```
Configuration:
- 1 Master
- 1 Replica (backup uniquement)
- Failover manuel acceptable
- RDB snapshots

Coût: Faible
Complexité: Faible
RTO: Variable (manuel)
RPO: Minutes/heures
```

---

## 🔍 Concepts clés à maîtriser

### 1. Réplication asynchrone

La réplication dans Redis est **asynchrone par défaut** :

```
Time ────────────────────────────────────────────▶

Master:  WRITE ──▶ ACK Client
           │
           │ (délai réseau)
           ▼
Replica:     WRITE Applied ──▶ Replica ACK
                                     │
                                     │ (peut être perdu)
                                     ▼
                              Si failover ici:
                              perte de données
```

**Implications** :
- Possible perte de données entre master et replica
- Pas de garantie de cohérence forte
- Trade-off performance vs durabilité

**Solution partielle** : `WAIT` command

```bash
# Attendre que N replicas aient reçu les writes
WAIT 2 1000  # 2 replicas, timeout 1000ms
```

### 2. Quorum et Consensus

Le quorum est le **nombre minimum de Sentinels** qui doivent s'accorder pour prendre une décision :

```
┌───────────────────────────────────────────────┐
│  CALCUL DU QUORUM                             │
├───────────────────────────────────────────────┤
│                                               │
│  Formule: Quorum = (N / 2) + 1                │
│                                               │
│  3 Sentinels → Quorum = 2                     │
│  5 Sentinels → Quorum = 3                     │
│  7 Sentinels → Quorum = 4                     │
│                                               │
│  Résistance aux pannes:                       │
│  • 3 Sentinels: tolère 1 panne                │
│  • 5 Sentinels: tolère 2 pannes               │
│  • 7 Sentinels: tolère 3 pannes               │
│                                               │
│  ⚠️  Toujours utiliser un nombre impair       │
│      pour éviter les situations d'égalité     │
│                                               │
└───────────────────────────────────────────────┘
```

### 3. Split-Brain

Le **split-brain** survient lors d'une partition réseau :

```
┌──────────────────────────────────────────────────────┐
│  SCÉNARIO SPLIT-BRAIN                                │
├──────────────────────────────────────────────────────┤
│                                                      │
│  État initial:                                       │
│  ┌────────┐      ┌────────┐      ┌────────┐          │
│  │Sentinel│──────│Sentinel│──────│Sentinel│          │
│  │   1    │      │   2    │      │   3    │          │
│  └────┬───┘      └────┬───┘      └────┬───┘          │
│       │               │               │              │
│       ▼               ▼               ▼              │
│  ┌────────┐      ┌────────┐                          │
│  │ Master │      │Replica │                          │
│  └────────┘      └────────┘                          │
│                                                      │
│  Partition réseau:                                   │
│  ┌────────┐            │ ┌────────┐      ┌────────┐  │
│  │Sentinel│            │ │Sentinel│──────│Sentinel│  │
│  │   1    │ ✗ ISOLATED │ │  2     │      │   3    │  │
│  └────┬───┘            │ └────┬───┘      └────┬───┘  │
│       │                │      │               │      │
│       ▼                │      ▼               ▼      │
│  ┌────────┐            │ ┌────────┐◀─ Promoted       │
│  │OLD     │ ✗          │ │  NEW   │   as Master      │
│  │Master  │            │ │ Master │                  │
│  └────────┘            │ └────────┘                  │
│       ▲                │                             │
│       │                │  ⚠️  Deux Masters actifs!   │
│  Client writes         │      Risque de divergence   │
│  (perdus)              │                             │
│                                                      │
│  Protection: min-replicas-to-write                   │
│              + quorum suffisant                      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Prévention** :

```ini
# Configuration Master
min-replicas-to-write 1
min-replicas-max-lag 10

# Si <1 replica avec lag <10s: Master refuse les writes
# Évite d'écrire sur un master isolé
```

---

## 📊 Métriques de haute disponibilité

### Indicateurs clés à monitorer

| Métrique | Commande | Seuil alerte | Impact |
|----------|----------|--------------|--------|
| **Replication lag** | `INFO replication` | >5 secondes | Perte données potentielle |
| **Connected replicas** | `INFO replication` | <1 | Risque failover |
| **Master link status** | `INFO replication` (replica) | down | Réplication cassée |
| **Sentinel status** | `SENTINEL masters` | <quorum | Pas de failover possible |
| **Last failover** | `SENTINEL masters` | - | Historique stabilité |
| **Pending commands** | `INFO replication` | >1000 | Backlog saturé |

### Commandes de diagnostic

```bash
# Vérifier l'état de réplication (sur Master)
redis-cli INFO replication

# Output:
# role:master
# connected_slaves:2
# slave0:ip=10.0.1.11,port=6379,state=online,offset=12345,lag=0
# slave1:ip=10.0.1.12,port=6379,state=online,offset=12345,lag=1

# Vérifier l'état Sentinel
redis-cli -p 26379 SENTINEL masters

# Obtenir l'adresse du master actuel
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Lister les sentinels
redis-cli -p 26379 SENTINEL sentinels mymaster

# Forcer un failover (testing)
redis-cli -p 26379 SENTINEL failover mymaster
```

---

## 🚀 Checklist de mise en production HA

### Avant le déploiement

- [ ] **Architecture validée** selon le cas d'usage
- [ ] **Quorum correctement dimensionné** (nombre impair de Sentinels)
- [ ] **Multi-AZ deployment** configuré
- [ ] **Authentification forte** sur tous les composants
- [ ] **Persistance adaptée** (RDB + AOF pour critique)
- [ ] **Réseau sécurisé** (VPC, security groups, firewalls)
- [ ] **Monitoring configuré** (métriques HA)
- [ ] **Alerting en place** (lag, replica down, etc.)
- [ ] **Documentation runbook** (procédures d'incident)
- [ ] **Tests de failover réalisés** en pre-production

### Configuration réseau

```bash
# Règles firewall minimales (exemple iptables)

# Redis Master/Replicas (6379)
iptables -A INPUT -p tcp -s 10.0.0.0/16 --dport 6379 -j ACCEPT

# Sentinel (26379)
iptables -A INPUT -p tcp -s 10.0.0.0/16 --dport 26379 -j ACCEPT

# Gossip Cluster (16379) - si Cluster mode
iptables -A INPUT -p tcp -s 10.0.0.0/16 --dport 16379 -j ACCEPT

# Deny all other
iptables -A INPUT -p tcp --dport 6379 -j DROP
iptables -A INPUT -p tcp --dport 26379 -j DROP
```

### Paramètres systèmes Linux optimaux

```bash
# /etc/sysctl.conf
net.core.somaxconn = 65535
vm.overcommit_memory = 1
net.ipv4.tcp_max_syn_backlog = 65535

# Désactiver Transparent Huge Pages
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Limites de fichiers
# /etc/security/limits.conf
redis soft nofile 65535
redis hard nofile 65535
```

---

## 📚 Plan du module

Ce module se décompose en 7 sections détaillées :

1. **Réplication Master-Replica** : Mécanismes internes, latence, topologies
2. **Topologies de réplication** : Chainée, en étoile, hybride
3. **Redis Sentinel** : Architecture, monitoring, failover
4. **Configuration Sentinel** : Déploiement, tuning, best practices
5. **Split-brain et Quorum** : Comprendre et prévenir les risques
6. **Connexion clients** : Service Discovery, failover transparent
7. **Tests de basculement** : Chaos engineering, scenarios de failure

---

## 🎓 Points clés à retenir

1. **La HA n'est pas gratuite** : Complexité, coût, compromis CAP
2. **Sentinel ≠ Cluster** : Sentinel pour HA simple, Cluster pour scaling
3. **Réplication asynchrone** : Possible perte de données lors du failover
4. **Quorum impair** : Toujours 3, 5 ou 7 Sentinels (jamais 2, 4, 6)
5. **Multi-AZ obligatoire** : Pour résister aux pannes de datacenter
6. **Monitoring critique** : Replication lag, connected replicas, Sentinel health
7. **Tester le failover** : Régulièrement, en pre-production
8. **Documentation essentielle** : Runbooks, procédures d'incident

---

## 🔗 Références

- [Redis Replication Official Docs](https://redis.io/docs/management/replication/)
- [Redis Sentinel Official Docs](https://redis.io/docs/management/sentinel/)
- [High Availability with Redis Sentinel](https://redis.io/topics/sentinel)
- [Redis Cluster Specification](https://redis.io/topics/cluster-spec)

---

**Prochaine section** : [10.1 La réplication Master-Replica : Principes et latence](./01-replication-master-replica.md)

⏭️ [La réplication Master-Replica : Principes et latence](/10-architecture-haute-disponibilite/01-replication-master-replica.md)
