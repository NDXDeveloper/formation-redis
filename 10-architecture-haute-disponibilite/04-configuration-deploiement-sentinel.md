🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.4 Configuration et déploiement de Sentinel

## Introduction

Le déploiement de Redis Sentinel en production nécessite une planification minutieuse et une configuration adaptée à l'infrastructure cible. Cette section couvre tous les aspects pratiques du déploiement, de l'installation initiale aux configurations avancées pour différents environnements.

**Objectifs** :
- Déployer Sentinel sur différentes infrastructures (bare metal, VMs, containers, cloud)
- Configurer correctement la topologie et le réseau
- Automatiser le déploiement et la gestion
- Valider le fonctionnement post-déploiement

---

## 🏗️ Topologies de déploiement

### Topologie 1 : Single Datacenter (3 AZ)

Configuration standard pour haute disponibilité dans une seule région.

```
┌───────────────────────────────────────────────────────────────┐
│  DÉPLOIEMENT SINGLE-DC / MULTI-AZ                             │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Region: eu-west-1                                            │
│                                                               │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐   │
│  │  AZ-A          │  │  AZ-B          │  │  AZ-C          │   │
│  │  (Paris-1)     │  │  (Paris-2)     │  │  (Paris-3)     │   │
│  ├────────────────┤  ├────────────────┤  ├────────────────┤   │
│  │                │  │                │  │                │   │
│  │  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │   │
│  │  │  Master  │  │  │  │ Replica1 │  │  │  │ Replica2 │  │   │
│  │  │  :6379   │──┼──┼─▶│  :6379   │  │  │  │  :6379   │  │   │
│  │  │10.0.1.10 │  │  │  │10.0.1.11 │◀─┼──┼──│10.0.1.12 │  │   │
│  │  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │   │
│  │       ▲        │  │       ▲        │  │       ▲        │   │
│  │       │        │  │       │        │  │       │        │   │
│  │  ┌────┴─────┐  │  │  ┌────┴─────┐  │  │  ┌────┴─────┐  │   │
│  │  │Sentinel1 │  │  │  │Sentinel2 │  │  │  │Sentinel3 │  │   │
│  │  │ :26379   │◀─┼──┼─▶│ :26379   │◀─┼──┼─▶│ :26379   │  │   │
│  │  │10.0.1.20 │  │  │  │10.0.1.21 │  │  │  │10.0.1.22 │  │   │
│  │  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │   │
│  │                │  │                │  │                │   │
│  └────────────────┘  └────────────────┘  └────────────────┘   │
│                                                               │
│  Caractéristiques:                                            │
│  • Latence: <5ms entre AZs                                    │
│  • Tolérance: 1 AZ complète peut tomber                       │
│  • Quorum: 2 (sur 3 Sentinels)                                │
│  • Réseau: VPC privé avec security groups                     │
│                                                               │
│  Adressage recommandé:                                        │
│  • Redis: 10.0.1.x (subnet application)                       │
│  • Sentinel: 10.0.1.y (même subnet ou dédié)                  │
│  • Clients: Accès via Sentinel discovery                      │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Topologie 2 : Multi-Datacenter (Active-Passive)

```
┌────────────────────────────────────────────────────────────────┐
│  DÉPLOIEMENT MULTI-DC (ACTIVE-PASSIVE)                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Primary DC (Paris)            Disaster Recovery DC (Frankfurt)│
│  ┌──────────────────────┐       ┌──────────────────────┐       │
│  │                      │       │                      │       │
│  │  ┌────────────┐      │       │   ┌────────────┐     │       │
│  │  │   Master   │      │       │   │  Replica   │     │       │
│  │  │   :6379    │──────┼───────┼──▶│   :6379    │     │       │
│  │  │ 10.1.0.10  │      │ WAN   │   │ 10.2.0.10  │     │       │
│  │  └─────┬──────┘      │ 50ms  │   └─────┬──────┘     │       │
│  │        │             │       │         │            │       │
│  │  ┌─────┴──────┐      │       │   ┌─────┴──────┐     │       │
│  │  │ Replica    │      │       │   │ Replica    │     │       │
│  │  │  :6379     │      │       │   │  :6379     │     │       │
│  │  │ 10.1.0.11  │      │       │   │ 10.2.0.11  │     │       │
│  │  └────────────┘      │       │   └────────────┘     │       │
│  │                      │       │                      │       │
│  │  Sentinels (3):      │       │   Sentinels (2):     │       │
│  │  10.1.0.20-22        │       │   10.2.0.20-21       │       │
│  │                      │       │                      │       │
│  └──────────────────────┘       └──────────────────────┘       │
│                                                                │
│  Configuration Sentinels:                                      │
│  • 3 Sentinels à Paris (quorum = 2)                            │
│  • 2 Sentinels à Frankfurt (observation)                       │
│  • Total: 5 Sentinels, Quorum = 3                              │
│                                                                │
│  Comportement:                                                 │
│  • Normal: Master à Paris, Replicas partout                    │
│  • Panne Paris: Quorum impossible (2 Sentinels Frankfurt < 3)  │
│  • → Failover manuel requis (promotion Frankfurt)              │
│                                                                │
│  Note: Pour failover auto cross-DC, utiliser quorum = 3 sur 5  │
│  mais risque de failover WAN (latence élevée)                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Topologie 3 : Sentinels co-localisés vs dédiés

```
┌───────────────────────────────────────────────────────────────┐
│  CO-LOCALISÉS vs DÉDIÉS                                       │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Option A: Co-localisés (même serveur)                        │
│  ┌────────────────────────────────────────────┐               │
│  │  Server 1                                  │               │
│  │  ┌──────────────────┐                      │               │
│  │  │ Redis Master     │ :6379                │               │
│  │  └──────────────────┘                      │               │
│  │  ┌──────────────────┐                      │               │
│  │  │ Sentinel         │ :26379               │               │
│  │  └──────────────────┘                      │               │
│  │                                            │               │
│  │  Avantages:                                │               │
│  │  ✅ Moins de serveurs (coût)               │               │
│  │  ✅ Latence monitoring minimale            │               │
│  │  ❌ Panne serveur = perte Redis + Sentinel │               │
│  │  ❌ Contention ressources CPU/RAM          │               │
│  └────────────────────────────────────────────┘               │
│                                                               │
│  Option B: Dédiés (serveurs séparés)                          │
│  ┌────────────────────┐    ┌────────────────────┐             │
│  │  Server 1 (Redis)  │    │ Server 4 (Sentinel)│             │
│  │  ┌──────────────┐  │    │  ┌──────────────┐  │             │
│  │  │ Redis Master │  │    │  │ Sentinel 1   │  │             │
│  │  │    :6379     │  │    │  │   :26379     │  │             │
│  │  └──────────────┘  │    │  └──────────────┘  │             │
│  └────────────────────┘    └────────────────────┘             │
│                                                               │
│  ┌────────────────────┐    ┌────────────────────┐             │
│  │  Server 2 (Redis)  │    │ Server 5 (Sentinel)│             │
│  │  ┌──────────────┐  │    │  ┌──────────────┐  │             │
│  │  │ Replica 1    │  │    │  │ Sentinel 2   │  │             │
│  │  │    :6379     │  │    │  │   :26379     │  │             │
│  │  └──────────────┘  │    │  └──────────────┘  │             │
│  └────────────────────┘    └────────────────────┘             │
│                                                               │
│  ┌────────────────────┐    ┌────────────────────┐             │
│  │  Server 3 (Redis)  │    │ Server 6 (Sentinel)│             │
│  │  ┌──────────────┐  │    │  ┌──────────────┐  │             │
│  │  │ Replica 2    │  │    │  │ Sentinel 3   │  │             │
│  │  │    :6379     │  │    │  │   :26379     │  │             │
│  │  └──────────────┘  │    │  └──────────────┘  │             │
│  └────────────────────┘    └────────────────────┘             │
│                                                               │
│  Avantages:                                                   │
│  ✅ Isolation complète (panne indépendante)                   │
│  ✅ Pas de contention ressources                              │
│  ✅ Scaling indépendant                                       │
│  ❌ Plus de serveurs (coût)                                   │
│  ❌ Latence monitoring légèrement supérieure                  │
│                                                               │
│  Recommandation Production:                                   │
│  • PME: Co-localisés (optimiser coûts)                        │
│  • Entreprise: Dédiés (fiabilité maximale)                    │
│  • Hybride: Sentinels sur app servers (charge légère)         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## 📦 Installation

### Méthode 1 : Installation depuis les sources

```bash
#!/bin/bash
# install-redis-sentinel-from-source.sh

# Variables
REDIS_VERSION="7.2.4"
INSTALL_DIR="/opt/redis"
DATA_DIR="/var/lib/redis/sentinel"
LOG_DIR="/var/log/redis"

# Installer dépendances
apt-get update
apt-get install -y build-essential tcl pkg-config libssl-dev

# Télécharger Redis
cd /tmp
wget http://download.redis.io/releases/redis-${REDIS_VERSION}.tar.gz
tar xzf redis-${REDIS_VERSION}.tar.gz
cd redis-${REDIS_VERSION}

# Compiler avec TLS support
make BUILD_TLS=yes
make test  # Optionnel mais recommandé

# Installer
make install PREFIX=${INSTALL_DIR}

# Créer utilisateur et groupes
useradd -r -s /bin/false redis

# Créer répertoires
mkdir -p ${DATA_DIR}
mkdir -p ${LOG_DIR}
mkdir -p /etc/redis

# Permissions
chown -R redis:redis ${DATA_DIR}
chown -R redis:redis ${LOG_DIR}

# Vérifier installation
${INSTALL_DIR}/bin/redis-sentinel --version

echo "Redis Sentinel installed successfully!"
echo "Version: $(${INSTALL_DIR}/bin/redis-sentinel --version)"
```

### Méthode 2 : Installation via package manager

```bash
# Ubuntu/Debian
apt-get update
apt-get install -y redis-sentinel

# CentOS/RHEL
yum install -y redis

# Fedora
dnf install -y redis

# Vérifier version
redis-sentinel --version
```

### Méthode 3 : Docker (développement/testing)

```bash
# Pull image officielle
docker pull redis:7.2-alpine

# Redis Sentinel est inclus dans l'image redis
docker run -d --name redis-sentinel \
  -p 26379:26379 \
  -v $(pwd)/sentinel.conf:/etc/redis/sentinel.conf \
  redis:7.2-alpine \
  redis-sentinel /etc/redis/sentinel.conf
```

---

## ⚙️ Configurations complètes par environnement

### Configuration 1 : Production Single-DC

**Fichier : `/etc/redis/sentinel-production.conf`**

```ini
# ════════════════════════════════════════════════════════════════
# REDIS SENTINEL - PRODUCTION CONFIGURATION
# ════════════════════════════════════════════════════════════════
# Environnement: Production Single-DC Multi-AZ
# Infrastructure: AWS / Azure / GCP
# Date: 2024-12-11
# ════════════════════════════════════════════════════════════════

################################# GENERAL #####################################

# Port d'écoute
port 26379

# Bind sur IP privée spécifique (sécurité)
bind 10.0.1.20 127.0.0.1

# Protection mode (si bind 0.0.0.0, mettre yes + requirepass)
protected-mode yes

# Mot de passe Sentinel (Redis 6+)
# requirepass "SentinelSecurePassword2024!"

# Daemonize (no si systemd, yes sinon)
daemonize no

# PID file
pidfile /var/run/redis/sentinel.pid

# Répertoire de travail
dir /var/lib/redis/sentinel

# Log file et level
logfile "/var/log/redis/sentinel.log"
loglevel notice

################################ MONITORING ###################################

# Monitorer le master principal
# Format: sentinel monitor <master-name> <ip> <port> <quorum>
#
# master-name: Nom logique (utilisé par clients)
# ip: IP du master actuel
# port: Port Redis
# quorum: Nombre de Sentinels pour déclarer ODOWN (2 sur 3 ici)
sentinel monitor mymaster 10.0.1.10 6379 2

# Authentification master
sentinel auth-pass mymaster MasterStrongPassword2024!

# ACL user (Redis 6+ si configuré)
# sentinel auth-user mymaster sentinel_user

################################# TIMEOUTS ####################################

# Temps sans réponse avant SDOWN (milliseconds)
# Valeur production typique: 30000 (30s)
# Augmenter si réseau instable ou charge élevée
sentinel down-after-milliseconds mymaster 30000

# Timeout total failover process
# Formule: 3-5× down-after-milliseconds
# Si dépassé: failover abandonné, retry plus tard
sentinel failover-timeout mymaster 180000

################################# FAILOVER ####################################

# Nombre de replicas à resync en parallèle après failover
# 1 = conservateur (pas de charge, plus lent)
# 2-3 = aggressif (plus rapide, plus de charge)
# Recommandation: 1 en production
sentinel parallel-syncs mymaster 1

# Délai avant qu'un replica soit éligible pour promotion (Redis 6.2+)
# Évite de promouvoir un replica fraîchement démarré
# 180000 = 3 minutes
sentinel replica-validity-time mymaster 180000

# Priority pour sélection replica
# Plus bas = plus prioritaire
# Configuré sur chaque Redis replica:
#   replica-priority 100 (master primaire)
#   replica-priority 90  (backup preferred)
#   replica-priority 80  (backup secondary)

################################# SCRIPTS #####################################

# Script de notification (événements majeurs)
sentinel notification-script mymaster /etc/redis/scripts/sentinel-notify.sh

# Script de reconfiguration client (post-failover)
sentinel client-reconfig-script mymaster /etc/redis/scripts/sentinel-reconfig.sh

# Timeout exécution scripts (60 secondes)
sentinel script-max-execution-time 60000

################################## SECURITY ###################################

# Empêcher reconfiguration dynamique via commandes
sentinel deny-scripts-reconfig yes

# Ne pas résoudre hostnames (performance + stabilité)
sentinel resolve-hostnames no
sentinel announce-hostnames no

# Pour Docker/NAT: annoncer IP/port publiques
# sentinel announce-ip 203.0.113.20
# sentinel announce-port 26379

################################# ADVANCED ####################################

# TLS (si Redis compilé avec TLS)
# port 0
# tls-port 26379
# tls-cert-file /etc/redis/certs/sentinel.crt
# tls-key-file /etc/redis/certs/sentinel.key
# tls-ca-cert-file /etc/redis/certs/ca.crt
# tls-replication yes

# ACL (Redis 6+)
# aclfile /etc/redis/sentinel-users.acl

################################ AUTO-MANAGED #################################
# Les lignes suivantes sont gérées automatiquement par Sentinel
# NE PAS MODIFIER MANUELLEMENT - Sentinel les met à jour

# sentinel myid <unique-id-generated-by-sentinel>
# sentinel config-epoch mymaster <epoch-number>
# sentinel leader-epoch mymaster <epoch-number>
# sentinel current-epoch <epoch-number>

# sentinel known-replica mymaster 10.0.1.11 6379
# sentinel known-replica mymaster 10.0.1.12 6379
# sentinel known-sentinel mymaster 10.0.1.21 26379 <sentinel2-id>
# sentinel known-sentinel mymaster 10.0.1.22 26379 <sentinel3-id>

# ════════════════════════════════════════════════════════════════
# END OF CONFIGURATION
# ════════════════════════════════════════════════════════════════
```

### Configuration 2 : Multi-Master (plusieurs masters surveillés)

```ini
# ════════════════════════════════════════════════════════════════
# SENTINEL - MULTI-MASTER CONFIGURATION
# ════════════════════════════════════════════════════════════════

port 26379
bind 10.0.1.20
protected-mode yes
daemonize no
pidfile /var/run/redis/sentinel.pid
dir /var/lib/redis/sentinel
logfile "/var/log/redis/sentinel.log"
loglevel notice

# ────────────────────────────────────────────────────────────────
# MASTER 1: Application principale
# ────────────────────────────────────────────────────────────────

sentinel monitor app-master 10.0.1.10 6379 2
sentinel auth-pass app-master AppMasterPass123!
sentinel down-after-milliseconds app-master 30000
sentinel failover-timeout app-master 180000
sentinel parallel-syncs app-master 1
sentinel notification-script app-master /etc/redis/scripts/notify-app.sh
sentinel client-reconfig-script app-master /etc/redis/scripts/reconfig-app.sh

# ────────────────────────────────────────────────────────────────
# MASTER 2: Cache sessions
# ────────────────────────────────────────────────────────────────

sentinel monitor session-master 10.0.2.10 6379 2
sentinel auth-pass session-master SessionMasterPass456!
sentinel down-after-milliseconds session-master 30000
sentinel failover-timeout session-master 180000
sentinel parallel-syncs session-master 1
sentinel notification-script session-master /etc/redis/scripts/notify-session.sh
sentinel client-reconfig-script session-master /etc/redis/scripts/reconfig-session.sh

# ────────────────────────────────────────────────────────────────
# MASTER 3: File d'attente jobs
# ────────────────────────────────────────────────────────────────

sentinel monitor queue-master 10.0.3.10 6379 2
sentinel auth-pass queue-master QueueMasterPass789!
sentinel down-after-milliseconds queue-master 30000
sentinel failover-timeout queue-master 180000
sentinel parallel-syncs queue-master 1
sentinel notification-script queue-master /etc/redis/scripts/notify-queue.sh
sentinel client-reconfig-script queue-master /etc/redis/scripts/reconfig-queue.sh

# ────────────────────────────────────────────────────────────────
# SECURITY & ADVANCED
# ────────────────────────────────────────────────────────────────

sentinel deny-scripts-reconfig yes
sentinel resolve-hostnames no
sentinel announce-hostnames no
sentinel script-max-execution-time 60000

# Note: Un Sentinel peut surveiller plusieurs masters
# Chaque master a sa propre config indépendante
# Coût ressources: ~10-20MB RAM par master surveillé
```

---

## 🐳 Déploiement avec Docker

### Docker Compose : Setup complet (development/testing)

```yaml
# docker-compose.yml - Redis + Sentinel stack

version: '3.8'

networks:
  redis-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

services:
  # ═══════════════════════════════════════════════════════════
  # REDIS INSTANCES
  # ═══════════════════════════════════════════════════════════

  redis-master:
    image: redis:7.2-alpine
    container_name: redis-master
    command: >
      redis-server
      --port 6379
      --requirepass "MasterPassword123"
      --masterauth "MasterPassword123"
      --appendonly yes
      --appendfsync everysec
      --save ""
    ports:
      - "6379:6379"
    volumes:
      - redis-master-data:/data
    networks:
      redis-net:
        ipv4_address: 172.20.0.10
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis-replica1:
    image: redis:7.2-alpine
    container_name: redis-replica1
    command: >
      redis-server
      --port 6379
      --replicaof redis-master 6379
      --requirepass "ReplicaPassword123"
      --masterauth "MasterPassword123"
      --replica-read-only yes
      --appendonly yes
    ports:
      - "6380:6379"
    volumes:
      - redis-replica1-data:/data
    networks:
      redis-net:
        ipv4_address: 172.20.0.11
    depends_on:
      - redis-master

  redis-replica2:
    image: redis:7.2-alpine
    container_name: redis-replica2
    command: >
      redis-server
      --port 6379
      --replicaof redis-master 6379
      --requirepass "ReplicaPassword123"
      --masterauth "MasterPassword123"
      --replica-read-only yes
      --appendonly yes
    ports:
      - "6381:6379"
    volumes:
      - redis-replica2-data:/data
    networks:
      redis-net:
        ipv4_address: 172.20.0.12
    depends_on:
      - redis-master

  # ═══════════════════════════════════════════════════════════
  # SENTINEL INSTANCES
  # ═══════════════════════════════════════════════════════════

  sentinel1:
    image: redis:7.2-alpine
    container_name: sentinel1
    command: redis-sentinel /etc/redis/sentinel.conf
    ports:
      - "26379:26379"
    volumes:
      - ./sentinel1.conf:/etc/redis/sentinel.conf
      - sentinel1-data:/data
    networks:
      redis-net:
        ipv4_address: 172.20.0.20
    depends_on:
      - redis-master
      - redis-replica1
      - redis-replica2

  sentinel2:
    image: redis:7.2-alpine
    container_name: sentinel2
    command: redis-sentinel /etc/redis/sentinel.conf
    ports:
      - "26380:26379"
    volumes:
      - ./sentinel2.conf:/etc/redis/sentinel.conf
      - sentinel2-data:/data
    networks:
      redis-net:
        ipv4_address: 172.20.0.21
    depends_on:
      - redis-master
      - redis-replica1
      - redis-replica2

  sentinel3:
    image: redis:7.2-alpine
    container_name: sentinel3
    command: redis-sentinel /etc/redis/sentinel.conf
    ports:
      - "26381:26379"
    volumes:
      - ./sentinel3.conf:/etc/redis/sentinel.conf
      - sentinel3-data:/data
    networks:
      redis-net:
        ipv4_address: 172.20.0.22
    depends_on:
      - redis-master
      - redis-replica1
      - redis-replica2

volumes:
  redis-master-data:
  redis-replica1-data:
  redis-replica2-data:
  sentinel1-data:
  sentinel2-data:
  sentinel3-data:
```

**Fichier Sentinel pour Docker** (`sentinel1.conf`, `sentinel2.conf`, `sentinel3.conf`) :

```ini
# sentinel.conf pour Docker

port 26379
bind 0.0.0.0
protected-mode no
dir /data

# Monitor master (utiliser nom de container ou IP fixe)
sentinel monitor mymaster redis-master 6379 2
# OU avec IP fixe:
# sentinel monitor mymaster 172.20.0.10 6379 2

sentinel auth-pass mymaster MasterPassword123

sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1

# Pour Docker: annoncer IP accessible de l'extérieur
# sentinel announce-ip <host-ip>
# sentinel announce-port <host-port-for-this-sentinel>

loglevel notice
```

**Commandes de gestion** :

```bash
# Démarrer le stack
docker-compose up -d

# Vérifier les logs
docker-compose logs -f sentinel1

# Tester la connexion
redis-cli -h localhost -p 26379 SENTINEL masters

# Obtenir le master actuel
redis-cli -h localhost -p 26379 SENTINEL get-master-addr-by-name mymaster

# Forcer un failover (test)
redis-cli -h localhost -p 26379 SENTINEL failover mymaster

# Arrêter le master (simuler panne)
docker-compose stop redis-master

# Observer le failover dans les logs
docker-compose logs -f sentinel1 sentinel2 sentinel3

# Nettoyer
docker-compose down -v
```

---

## ☸️ Déploiement sur Kubernetes

### StatefulSet complet avec Sentinels

```yaml
# redis-sentinel-k8s.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-sentinel-config
  namespace: redis
data:
  redis-master.conf: |
    port 6379
    bind 0.0.0.0
    protected-mode no
    requirepass "K8sMasterPass123!"
    masterauth "K8sMasterPass123!"
    appendonly yes
    appendfsync everysec
    save ""

  redis-replica.conf: |
    port 6379
    bind 0.0.0.0
    protected-mode no
    requirepass "K8sReplicaPass123!"
    masterauth "K8sMasterPass123!"
    replicaof redis-0.redis-service.redis.svc.cluster.local 6379
    replica-read-only yes
    appendonly yes

  sentinel.conf: |
    port 26379
    bind 0.0.0.0
    protected-mode no
    dir /data
    sentinel monitor mymaster redis-0.redis-service.redis.svc.cluster.local 6379 2
    sentinel auth-pass mymaster K8sMasterPass123!
    sentinel down-after-milliseconds mymaster 30000
    sentinel failover-timeout mymaster 180000
    sentinel parallel-syncs mymaster 1

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: redis
spec:
  clusterIP: None  # Headless service
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
  selector:
    app: redis

---
apiVersion: v1
kind: Service
metadata:
  name: sentinel-service
  namespace: redis
spec:
  clusterIP: None  # Headless service
  ports:
    - name: sentinel
      port: 26379
      targetPort: 26379
  selector:
    app: redis-sentinel

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: redis
spec:
  serviceName: redis-service
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.2-alpine
        ports:
        - containerPort: 6379
          name: redis
        command:
          - sh
          - -c
          - |
            if [ "$(hostname)" = "redis-0" ]; then
              redis-server /etc/redis/redis-master.conf
            else
              redis-server /etc/redis/redis-replica.conf
            fi
        volumeMounts:
        - name: redis-data
          mountPath: /data
        - name: config
          mountPath: /etc/redis
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: redis-sentinel-config
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sentinel
  namespace: redis
spec:
  serviceName: sentinel-service
  replicas: 3
  selector:
    matchLabels:
      app: redis-sentinel
  template:
    metadata:
      labels:
        app: redis-sentinel
    spec:
      containers:
      - name: sentinel
        image: redis:7.2-alpine
        ports:
        - containerPort: 26379
          name: sentinel
        command:
          - redis-sentinel
          - /etc/redis/sentinel.conf
        volumeMounts:
        - name: sentinel-data
          mountPath: /data
        - name: config
          mountPath: /etc/redis
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
      volumes:
      - name: config
        configMap:
          name: redis-sentinel-config
  volumeClaimTemplates:
  - metadata:
      name: sentinel-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi

---
# Service LoadBalancer pour accès externe (optionnel)
apiVersion: v1
kind: Service
metadata:
  name: sentinel-lb
  namespace: redis
spec:
  type: LoadBalancer
  ports:
    - name: sentinel
      port: 26379
      targetPort: 26379
  selector:
    app: redis-sentinel
```

**Déploiement** :

```bash
# Créer namespace
kubectl create namespace redis

# Appliquer configuration
kubectl apply -f redis-sentinel-k8s.yaml

# Vérifier déploiement
kubectl get pods -n redis -w

# Tester connexion Sentinel
kubectl exec -it sentinel-0 -n redis -- redis-cli -p 26379 SENTINEL masters

# Accéder au service depuis un pod
kubectl run -it --rm redis-client --image=redis:7.2-alpine --restart=Never -- sh
# redis-cli -h sentinel-service.redis.svc.cluster.local -p 26379 SENTINEL get-master-addr-by-name mymaster
```

---

## 🔧 Intégration systemd

### Service systemd pour Sentinel

```ini
# /etc/systemd/system/redis-sentinel.service

[Unit]
Description=Redis Sentinel
After=network.target
Documentation=https://redis.io/docs/management/sentinel/

[Service]
Type=notify
User=redis
Group=redis

# Exécutable et configuration
ExecStart=/usr/local/bin/redis-sentinel /etc/redis/sentinel.conf --supervised systemd
ExecStop=/bin/kill -s TERM $MAINPID

# Répertoire de travail
WorkingDirectory=/var/lib/redis/sentinel

# Limites de ressources
LimitNOFILE=65535
MemoryLimit=256M

# Sécurité
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true
ReadWritePaths=/var/lib/redis/sentinel /var/log/redis

# Redémarrage automatique
Restart=on-failure
RestartSec=5s
TimeoutStartSec=30s
TimeoutStopSec=30s

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=redis-sentinel

[Install]
WantedBy=multi-user.target
```

**Commandes systemd** :

```bash
# Recharger systemd
systemctl daemon-reload

# Activer au démarrage
systemctl enable redis-sentinel

# Démarrer
systemctl start redis-sentinel

# Vérifier status
systemctl status redis-sentinel

# Logs
journalctl -u redis-sentinel -f

# Redémarrer
systemctl restart redis-sentinel

# Arrêter
systemctl stop redis-sentinel
```

---

## 🚀 Scripts de déploiement automatisé

### Script de déploiement complet

```bash
#!/bin/bash
# deploy-redis-sentinel.sh - Déploiement automatisé

set -e  # Exit on error

# ════════════════════════════════════════════════════════════════
# CONFIGURATION
# ════════════════════════════════════════════════════════════════

ENVIRONMENT=${1:-production}  # production, staging, development
REDIS_VERSION="7.2.4"

# IPs des serveurs (à adapter)
MASTER_IP="10.0.1.10"
REPLICA1_IP="10.0.1.11"
REPLICA2_IP="10.0.1.12"
SENTINEL1_IP="10.0.1.20"
SENTINEL2_IP="10.0.1.21"
SENTINEL3_IP="10.0.1.22"

MASTER_PASSWORD="GenerateSecurePassword123!"
QUORUM=2

# Répertoires
INSTALL_DIR="/opt/redis"
DATA_DIR="/var/lib/redis"
LOG_DIR="/var/log/redis"
CONFIG_DIR="/etc/redis"

# ════════════════════════════════════════════════════════════════
# FONCTIONS
# ════════════════════════════════════════════════════════════════

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

error() {
    echo "[ERROR] $1" >&2
    exit 1
}

install_redis() {
    local host=$1
    log "Installing Redis on $host..."

    ssh root@$host bash <<EOF
        set -e
        apt-get update
        apt-get install -y build-essential tcl wget

        cd /tmp
        wget http://download.redis.io/releases/redis-${REDIS_VERSION}.tar.gz
        tar xzf redis-${REDIS_VERSION}.tar.gz
        cd redis-${REDIS_VERSION}

        make
        make install PREFIX=${INSTALL_DIR}

        # Créer utilisateur
        useradd -r -s /bin/false redis || true

        # Créer répertoires
        mkdir -p ${DATA_DIR}/{master,replica,sentinel}
        mkdir -p ${LOG_DIR}
        mkdir -p ${CONFIG_DIR}

        chown -R redis:redis ${DATA_DIR}
        chown -R redis:redis ${LOG_DIR}

        echo "Redis installed on $host"
EOF
}

configure_master() {
    log "Configuring Redis Master on $MASTER_IP..."

    ssh root@$MASTER_IP bash <<EOF
        cat > ${CONFIG_DIR}/redis-master.conf <<CONFIG
port 6379
bind $MASTER_IP 127.0.0.1
protected-mode yes
requirepass "$MASTER_PASSWORD"
masterauth "$MASTER_PASSWORD"
dir ${DATA_DIR}/master
logfile "${LOG_DIR}/redis-master.log"
appendonly yes
appendfsync everysec
save ""
maxmemory 4gb
maxmemory-policy allkeys-lru
CONFIG

        chown redis:redis ${CONFIG_DIR}/redis-master.conf
        echo "Master configured"
EOF
}

configure_replica() {
    local replica_ip=$1
    local replica_num=$2

    log "Configuring Redis Replica $replica_num on $replica_ip..."

    ssh root@$replica_ip bash <<EOF
        cat > ${CONFIG_DIR}/redis-replica.conf <<CONFIG
port 6379
bind $replica_ip 127.0.0.1
protected-mode yes
requirepass "$MASTER_PASSWORD"
masterauth "$MASTER_PASSWORD"
replicaof $MASTER_IP 6379
replica-read-only yes
replica-priority $((100 - replica_num * 10))
dir ${DATA_DIR}/replica
logfile "${LOG_DIR}/redis-replica${replica_num}.log"
appendonly yes
appendfsync everysec
maxmemory 4gb
maxmemory-policy allkeys-lru
CONFIG

        chown redis:redis ${CONFIG_DIR}/redis-replica.conf
        echo "Replica $replica_num configured"
EOF
}

configure_sentinel() {
    local sentinel_ip=$1
    local sentinel_num=$2

    log "Configuring Sentinel $sentinel_num on $sentinel_ip..."

    ssh root@$sentinel_ip bash <<EOF
        cat > ${CONFIG_DIR}/sentinel.conf <<CONFIG
port 26379
bind $sentinel_ip 127.0.0.1
protected-mode yes
daemonize no
pidfile ${DATA_DIR}/sentinel/sentinel.pid
dir ${DATA_DIR}/sentinel
logfile "${LOG_DIR}/sentinel${sentinel_num}.log"
loglevel notice

sentinel monitor mymaster $MASTER_IP 6379 $QUORUM
sentinel auth-pass mymaster $MASTER_PASSWORD
sentinel down-after-milliseconds mymaster 30000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel deny-scripts-reconfig yes
sentinel resolve-hostnames no
sentinel announce-hostnames no
CONFIG

        chown redis:redis ${CONFIG_DIR}/sentinel.conf
        echo "Sentinel $sentinel_num configured"
EOF
}

deploy_systemd_service() {
    local host=$1
    local service_type=$2  # master, replica, sentinel

    log "Deploying systemd service for $service_type on $host..."

    local config_file=""
    local service_name=""

    case $service_type in
        master)
            config_file="${CONFIG_DIR}/redis-master.conf"
            service_name="redis-master"
            ;;
        replica)
            config_file="${CONFIG_DIR}/redis-replica.conf"
            service_name="redis-replica"
            ;;
        sentinel)
            config_file="${CONFIG_DIR}/sentinel.conf"
            service_name="redis-sentinel"
            ;;
    esac

    ssh root@$host bash <<EOF
        cat > /etc/systemd/system/${service_name}.service <<SERVICE
[Unit]
Description=Redis ${service_type}
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=${INSTALL_DIR}/bin/redis-server $config_file --supervised systemd
ExecStop=/bin/kill -s TERM \$MAINPID
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
SERVICE

        systemctl daemon-reload
        systemctl enable ${service_name}
        echo "Systemd service deployed for $service_type"
EOF
}

start_services() {
    log "Starting services..."

    # Démarrer master
    ssh root@$MASTER_IP "systemctl start redis-master"
    sleep 5

    # Démarrer replicas
    ssh root@$REPLICA1_IP "systemctl start redis-replica"
    ssh root@$REPLICA2_IP "systemctl start redis-replica"
    sleep 5

    # Démarrer sentinels
    ssh root@$SENTINEL1_IP "systemctl start redis-sentinel"
    ssh root@$SENTINEL2_IP "systemctl start redis-sentinel"
    ssh root@$SENTINEL3_IP "systemctl start redis-sentinel"

    log "All services started"
}

verify_deployment() {
    log "Verifying deployment..."

    # Vérifier master
    if ssh root@$MASTER_IP "redis-cli -a $MASTER_PASSWORD PING" | grep -q "PONG"; then
        log "✅ Master is responding"
    else
        error "Master is not responding"
    fi

    # Vérifier replicas
    local replica_count=$(ssh root@$MASTER_IP "redis-cli -a $MASTER_PASSWORD INFO replication | grep connected_slaves" | cut -d: -f2 | tr -d '\r')
    if [ "$replica_count" -eq 2 ]; then
        log "✅ 2 replicas connected"
    else
        log "⚠️  Only $replica_count replicas connected (expected 2)"
    fi

    # Vérifier sentinels
    if ssh root@$SENTINEL1_IP "redis-cli -p 26379 SENTINEL master mymaster" | grep -q "ip"; then
        log "✅ Sentinel is monitoring master"
    else
        error "Sentinel is not monitoring master"
    fi

    log "Deployment verification complete"
}

# ════════════════════════════════════════════════════════════════
# MAIN
# ════════════════════════════════════════════════════════════════

main() {
    log "Starting Redis Sentinel deployment ($ENVIRONMENT)..."

    # Installer Redis sur tous les serveurs
    install_redis $MASTER_IP
    install_redis $REPLICA1_IP
    install_redis $REPLICA2_IP
    install_redis $SENTINEL1_IP
    install_redis $SENTINEL2_IP
    install_redis $SENTINEL3_IP

    # Configurer instances
    configure_master
    configure_replica $REPLICA1_IP 1
    configure_replica $REPLICA2_IP 2
    configure_sentinel $SENTINEL1_IP 1
    configure_sentinel $SENTINEL2_IP 2
    configure_sentinel $SENTINEL3_IP 3

    # Déployer services systemd
    deploy_systemd_service $MASTER_IP master
    deploy_systemd_service $REPLICA1_IP replica
    deploy_systemd_service $REPLICA2_IP replica
    deploy_systemd_service $SENTINEL1_IP sentinel
    deploy_systemd_service $SENTINEL2_IP sentinel
    deploy_systemd_service $SENTINEL3_IP sentinel

    # Démarrer services
    start_services

    # Vérifier déploiement
    sleep 10
    verify_deployment

    log "✅ Deployment complete!"
    log "Master: $MASTER_IP:6379"
    log "Sentinels: $SENTINEL1_IP:26379, $SENTINEL2_IP:26379, $SENTINEL3_IP:26379"
}

# Exécuter
main "$@"
```

---

## ✅ Vérification post-déploiement

### Checklist complète

```bash
#!/bin/bash
# verify-sentinel-deployment.sh

echo "═══════════════════════════════════════════════"
echo "Redis Sentinel Deployment Verification"
echo "═══════════════════════════════════════════════"
echo ""

SENTINEL_HOST="localhost"
SENTINEL_PORT="26379"
MASTER_NAME="mymaster"

# Couleurs
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

check_pass() {
    echo -e "${GREEN}✅ PASS${NC}: $1"
}

check_fail() {
    echo -e "${RED}❌ FAIL${NC}: $1"
}

check_warn() {
    echo -e "${YELLOW}⚠️  WARN${NC}: $1"
}

# 1. Vérifier connectivité Sentinel
echo "1. Sentinel Connectivity"
if redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT PING >/dev/null 2>&1; then
    check_pass "Sentinel is reachable"
else
    check_fail "Cannot connect to Sentinel"
    exit 1
fi
echo ""

# 2. Vérifier master discovery
echo "2. Master Discovery"
MASTER_ADDR=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL get-master-addr-by-name $MASTER_NAME)
if [ -n "$MASTER_ADDR" ]; then
    MASTER_IP=$(echo "$MASTER_ADDR" | head -n1)
    MASTER_PORT=$(echo "$MASTER_ADDR" | tail -n1)
    check_pass "Master found: $MASTER_IP:$MASTER_PORT"

    # Vérifier connectivité master
    if redis-cli -h $MASTER_IP -p $MASTER_PORT PING >/dev/null 2>&1; then
        check_pass "Master is accessible"
    else
        check_fail "Master is not accessible"
    fi
else
    check_fail "No master found"
fi
echo ""

# 3. Vérifier nombre de replicas
echo "3. Replica Count"
REPLICA_COUNT=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL master $MASTER_NAME | grep -A1 "num-slaves" | tail -n1)
if [ "$REPLICA_COUNT" -ge 2 ]; then
    check_pass "$REPLICA_COUNT replicas configured (≥2)"
else
    check_warn "Only $REPLICA_COUNT replica(s) configured"
fi
echo ""

# 4. Vérifier nombre de Sentinels
echo "4. Sentinel Count"
SENTINEL_COUNT=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL master $MASTER_NAME | grep -A1 "num-other-sentinels" | tail -n1)
TOTAL_SENTINELS=$((SENTINEL_COUNT + 1))
if [ "$TOTAL_SENTINELS" -ge 3 ]; then
    check_pass "$TOTAL_SENTINELS Sentinels configured (≥3)"
else
    check_fail "Only $TOTAL_SENTINELS Sentinel(s) configured (need ≥3)"
fi
echo ""

# 5. Vérifier quorum
echo "5. Quorum Configuration"
QUORUM=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL master $MASTER_NAME | grep -A1 "^quorum$" | tail -n1)
EXPECTED_QUORUM=$(( ($TOTAL_SENTINELS / 2) + 1 ))
if [ "$QUORUM" -eq "$EXPECTED_QUORUM" ]; then
    check_pass "Quorum is $QUORUM (optimal for $TOTAL_SENTINELS Sentinels)"
else
    check_warn "Quorum is $QUORUM (expected $EXPECTED_QUORUM for $TOTAL_SENTINELS Sentinels)"
fi

# Vérifier que quorum est atteignable
CKQUORUM=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL ckquorum $MASTER_NAME)
if echo "$CKQUORUM" | grep -q "OK"; then
    check_pass "Quorum is satisfied and reachable"
else
    check_fail "Quorum cannot be reached!"
fi
echo ""

# 6. Vérifier état des replicas
echo "6. Replica Status"
redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL replicas $MASTER_NAME | \
    awk '/^ip$/{getline ip; getline; getline port_line; getline port; getline; getline flags_line; getline flags; print ip":"port" - "flags}' | \
while read replica_info; do
    if echo "$replica_info" | grep -q "slave"; then
        check_pass "Replica: $replica_info"
    else
        check_fail "Replica issue: $replica_info"
    fi
done
echo ""

# 7. Vérifier configuration down-after-milliseconds
echo "7. Timeouts Configuration"
DOWN_AFTER=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL master $MASTER_NAME | grep -A1 "down-after-milliseconds" | tail -n1)
if [ "$DOWN_AFTER" -ge 30000 ]; then
    check_pass "down-after-milliseconds: ${DOWN_AFTER}ms (≥30s)"
else
    check_warn "down-after-milliseconds: ${DOWN_AFTER}ms (<30s, may cause false positives)"
fi

FAILOVER_TIMEOUT=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL master $MASTER_NAME | grep -A1 "failover-timeout" | tail -n1)
check_pass "failover-timeout: ${FAILOVER_TIMEOUT}ms"
echo ""

# 8. Vérifier protection split-brain
echo "8. Split-Brain Protection"
if [ -n "$MASTER_IP" ]; then
    MIN_REPLICAS=$(redis-cli -h $MASTER_IP -p $MASTER_PORT CONFIG GET min-replicas-to-write 2>/dev/null | tail -n1)
    if [ "$MIN_REPLICAS" -ge 1 ]; then
        check_pass "min-replicas-to-write: $MIN_REPLICAS (split-brain protection active)"
    else
        check_warn "min-replicas-to-write: $MIN_REPLICAS (no split-brain protection)"
    fi
fi
echo ""

# 9. Tester failover (optionnel, commenter si non désiré)
# echo "9. Failover Test (commented out)"
# echo "To test failover manually, run:"
# echo "  redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL failover $MASTER_NAME"
# echo ""

echo "═══════════════════════════════════════════════"
echo "Verification Complete"
echo "═══════════════════════════════════════════════"
```

---

## 🎓 Points clés à retenir

1. **Topologie standard** : 3 Sentinels, 1 master, 2+ replicas, Multi-AZ
2. **Quorum = (N/2) + 1** : Toujours nombre impair de Sentinels
3. **Co-location acceptable** : Pour PME, sinon préférer dédié
4. **Docker viable** : Pour dev/test, attention aux IPs/networking
5. **Kubernetes nécessite StatefulSets** : Pas de Deployments
6. **Systemd recommandé** : Pour production bare metal/VMs
7. **Scripts automatisés** : Essentiels pour déploiements répétables
8. **Vérification post-deploy** : Obligatoire, automatiser avec scripts
9. **Documentation** : Maintenir runbooks à jour
10. **Testing régulier** : Simuler pannes en pre-prod

---

## 🔗 Références

- [Redis Sentinel Documentation](https://redis.io/docs/management/sentinel/)
- [Docker Official Redis Image](https://hub.docker.com/_/redis)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Systemd Service Files](https://www.freedesktop.org/software/systemd/man/systemd.service.html)

---

**Section suivante** : [10.5 Split-brain et Quorum : Comprendre les risques](./05-split-brain-quorum-risques.md)

⏭️ [Split-brain et Quorum : Comprendre les risques](/10-architecture-haute-disponibilite/05-split-brain-quorum-risques.md)
