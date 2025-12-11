🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.4 Configuration et déploiement d'un cluster

## Introduction

Le déploiement d'un cluster Redis nécessite une configuration minutieuse et une procédure méthodique pour garantir la stabilité et les performances en production. Contrairement à une instance Redis autonome, un cluster implique la coordination de multiples nœuds, la configuration du protocole Gossip, et l'assignation stratégique des hash slots.

Cette section détaille les aspects pratiques du déploiement, depuis la configuration initiale jusqu'à la validation complète du cluster, en passant par les différentes méthodes de déploiement (manuelle, automatisée, containerisée) et les optimisations système nécessaires.

## Prérequis système

### Configuration matérielle minimale

```
┌─────────────────────────────────────────────────────────────┐
│           Spécifications matérielles par nœud               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ENVIRONNEMENT DE DÉVELOPPEMENT                              │
│ ════════════════════════════════                            │
│ CPU     : 2 cores                                           │
│ RAM     : 4 GB                                              │
│ Disque  : 20 GB SSD                                         │
│ Réseau  : 1 Gbps                                            │
│                                                             │
│ STAGING / PRE-PRODUCTION                                    │
│ ════════════════════════════                                │
│ CPU     : 4 cores                                           │
│ RAM     : 16 GB                                             │
│ Disque  : 100 GB SSD                                        │
│ Réseau  : 1-10 Gbps                                         │
│                                                             │
│ PRODUCTION                                                  │
│ ══════════                                                  │
│ CPU     : 8+ cores                                          │
│ RAM     : 32-256 GB                                         │
│ Disque  : 500 GB+ NVMe SSD                                  │
│ Réseau  : 10 Gbps minimum                                   │
│ IOPS    : 10,000+ (si persistence activée)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Configuration système Linux

```bash
# Configuration optimale pour Redis Cluster sur Linux

# 1. Désactiver Transparent Huge Pages (THP)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Rendre permanent (ajouter à /etc/rc.local ou systemd)
cat >> /etc/rc.local <<EOF
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF

# 2. Configurer overcommit memory
sysctl vm.overcommit_memory=1
echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf

# 3. Augmenter les limites de connexions
sysctl net.core.somaxconn=65535
echo "net.core.somaxconn = 65535" >> /etc/sysctl.conf

# 4. Configurer TCP backlog
sysctl net.ipv4.tcp_max_syn_backlog=65535
echo "net.ipv4.tcp_max_syn_backlog = 65535" >> /etc/sysctl.conf

# 5. Limites de fichiers ouverts
cat >> /etc/security/limits.conf <<EOF
redis soft nofile 65536
redis hard nofile 65536
EOF

# 6. Swappiness (réduire l'utilisation du swap)
sysctl vm.swappiness=1
echo "vm.swappiness = 1" >> /etc/sysctl.conf

# 7. Activer les changements
sysctl -p

# Vérification
cat /sys/kernel/mm/transparent_hugepage/enabled
# Doit afficher : always madvise [never]

sysctl vm.overcommit_memory
# Doit afficher : vm.overcommit_memory = 1
```

### Configuration réseau et firewall

```bash
# Ports requis pour Redis Cluster :
# - 6379 : Port client (configurable)
# - 16379 : Port cluster bus (port client + 10000)

# UFW (Ubuntu/Debian)
sudo ufw allow 6379/tcp
sudo ufw allow 16379/tcp

# FirewallD (RHEL/CentOS)
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --permanent --add-port=16379/tcp
sudo firewall-cmd --reload

# iptables (méthode manuelle)
iptables -A INPUT -p tcp --dport 6379 -j ACCEPT
iptables -A INPUT -p tcp --dport 16379 -j ACCEPT
iptables-save > /etc/iptables/rules.v4

# Vérifier la connectivité entre nœuds
# Depuis chaque nœud, tester les autres :
nc -zv 192.168.1.11 6379
nc -zv 192.168.1.11 16379
nc -zv 192.168.1.12 6379
nc -zv 192.168.1.12 16379

# Test de latence réseau (doit être < 1ms en LAN)
ping -c 10 192.168.1.11
```

## Configuration Redis pour le mode Cluster

### Fichier redis.conf minimal pour le cluster

```bash
# /etc/redis/redis-6379.conf
# Configuration minimale pour Redis Cluster

# ═══════════════════════════════════════════════════════════
# CONFIGURATION DE BASE
# ═══════════════════════════════════════════════════════════

# Port d'écoute
port 6379

# Interface réseau (IMPORTANT : ne pas utiliser 127.0.0.1 en cluster)
bind 0.0.0.0
# En production, limiter aux interfaces privées :
# bind 192.168.1.10

# Mode protégé (désactiver si bind != 127.0.0.1)
protected-mode no

# Nombre de databases (toujours 1 en mode cluster)
databases 1

# ═══════════════════════════════════════════════════════════
# CLUSTER
# ═══════════════════════════════════════════════════════════

# Activer le mode cluster (OBLIGATOIRE)
cluster-enabled yes

# Fichier de configuration du cluster (créé automatiquement)
cluster-config-file /var/lib/redis/nodes-6379.conf

# Timeout de détection de panne (en millisecondes)
# Valeur par défaut : 15000 (15 secondes)
# Production : 15000-30000
cluster-node-timeout 15000

# Facteur de validité des replicas
# Une replica est considérée valide si son dernier contact avec le master
# est inférieur à : (node-timeout × replica-validity-factor) + repl-ping-period
cluster-replica-validity-factor 10

# Nombre minimum de replicas à conserver avant migration automatique
cluster-migration-barrier 1

# Nécessite que tous les slots soient couverts pour accepter les requêtes
# yes : cluster refuse les requêtes si slots non couverts (recommandé)
# no : cluster accepte les requêtes même si certains slots ne sont pas couverts
cluster-require-full-coverage yes

# Autoriser les lectures sur les replicas (Redis 7+)
# cluster-allow-reads-when-down no

# Permettre aux replicas de servir les requêtes en lecture
cluster-replica-no-failover no

# ═══════════════════════════════════════════════════════════
# MÉMOIRE
# ═══════════════════════════════════════════════════════════

# Limite de mémoire (adapter selon RAM disponible)
maxmemory 30gb

# Politique d'éviction quand maxmemory est atteinte
maxmemory-policy allkeys-lru

# Échantillonnage pour LRU/LFU (3-10, défaut: 5)
maxmemory-samples 5

# ═══════════════════════════════════════════════════════════
# PERSISTENCE
# ═══════════════════════════════════════════════════════════

# RDB (snapshots)
save 900 1
save 300 10
save 60 10000

# Nom du fichier RDB
dbfilename dump-6379.rdb

# Répertoire de travail
dir /var/lib/redis

# Compression RDB
rdbcompression yes
rdbchecksum yes

# AOF (Append Only File)
appendonly yes
appendfilename "appendonly-6379.aof"

# Politique de fsync AOF
# everysec : bon compromis performance/durabilité
appendfsync everysec

# Réécriture automatique de l'AOF
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# ═══════════════════════════════════════════════════════════
# SÉCURITÉ
# ═══════════════════════════════════════════════════════════

# Mot de passe (utiliser ACLs en production)
# requirepass VotreMotDePasseSecurise

# ACLs (Redis 6+, recommandé)
# aclfile /etc/redis/users.acl

# Renommer les commandes dangereuses
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG-9a8b7c6d5e4f3a2b1"

# ═══════════════════════════════════════════════════════════
# RÉPLICATION
# ═══════════════════════════════════════════════════════════

# Réplication sans disque (streaming)
repl-diskless-sync yes
repl-diskless-sync-delay 5

# Backlog de réplication (important pour clusters)
repl-backlog-size 100mb
repl-backlog-ttl 3600

# ═══════════════════════════════════════════════════════════
# PERFORMANCE
# ═══════════════════════════════════════════════════════════

# TCP backlog
tcp-backlog 511

# TCP keepalive
tcp-keepalive 300

# Nombre de threads I/O (Redis 6+)
io-threads 4
io-threads-do-reads yes

# ═══════════════════════════════════════════════════════════
# LOGGING
# ═══════════════════════════════════════════════════════════

loglevel notice
logfile /var/log/redis/redis-6379.log

# Slowlog
slowlog-log-slower-than 10000
slowlog-max-len 128

# ═══════════════════════════════════════════════════════════
# LATENCY MONITORING
# ═══════════════════════════════════════════════════════════

latency-monitor-threshold 100
```

### Configuration différenciée par environnement

```bash
# Production : redis-production.conf
# ═══════════════════════════════════
port 6379
bind 10.0.1.10                    # IP privée du nœud
protected-mode yes
cluster-enabled yes
cluster-node-timeout 30000        # Plus élevé en production
maxmemory 128gb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
requirepass "ProductionPassword123!"
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG ""
io-threads 8                      # Plus de threads en production


# Staging : redis-staging.conf
# ═════════════════════════════
port 6379
bind 10.0.2.10
protected-mode yes
cluster-enabled yes
cluster-node-timeout 15000
maxmemory 32gb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
requirepass "StagingPassword456!"
io-threads 4


# Development : redis-dev.conf
# ════════════════════════════
port 6379
bind 0.0.0.0                      # Accessible partout (dev seulement!)
protected-mode no
cluster-enabled yes
cluster-node-timeout 5000         # Failover plus rapide en dev
maxmemory 4gb
maxmemory-policy noeviction       # Pas d'éviction en dev
appendonly no                     # Pas de persistence en dev
save ""                           # Désactiver snapshots RDB
io-threads 2
```

## Méthodes de déploiement

### Méthode 1 : Déploiement manuel (Bare Metal / VM)

```bash
# ═══════════════════════════════════════════════════════════
# DÉPLOIEMENT MANUEL - 6 NŒUDS (3 MASTERS + 3 REPLICAS)
# ═══════════════════════════════════════════════════════════

# ÉTAPE 1 : Installation de Redis sur chaque nœud
# ─────────────────────────────────────────────────

# Ubuntu/Debian
sudo apt update
sudo apt install -y redis-server redis-tools

# RHEL/CentOS
sudo yum install -y redis

# Ou compilation depuis les sources (pour version spécifique)
cd /tmp
wget https://download.redis.io/releases/redis-7.2.3.tar.gz
tar xzf redis-7.2.3.tar.gz
cd redis-7.2.3
make
sudo make install


# ÉTAPE 2 : Configuration sur chaque nœud
# ────────────────────────────────────────

# Nœuds : 192.168.1.10, 192.168.1.11, 192.168.1.12,
#         192.168.1.13, 192.168.1.14, 192.168.1.15

# Sur CHAQUE nœud, créer le fichier de configuration
sudo mkdir -p /etc/redis /var/lib/redis /var/log/redis
sudo chown redis:redis /var/lib/redis /var/log/redis

# Copier le fichier redis.conf préparé précédemment
sudo cp redis-cluster.conf /etc/redis/redis.conf

# Adapter le bind à l'IP du nœud (exemple pour node 1)
sudo sed -i 's/bind 0.0.0.0/bind 192.168.1.10/' /etc/redis/redis.conf


# ÉTAPE 3 : Créer le service systemd (si pas déjà existant)
# ──────────────────────────────────────────────────────────

cat <<EOF | sudo tee /etc/systemd/system/redis.service
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always
RestartSec=5s

# Limites
LimitNOFILE=65536

# Sécurité
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ReadWritePaths=/var/lib/redis /var/log/redis

[Install]
WantedBy=multi-user.target
EOF


# ÉTAPE 4 : Démarrer Redis sur chaque nœud
# ─────────────────────────────────────────

# Sur CHAQUE nœud :
sudo systemctl daemon-reload
sudo systemctl enable redis
sudo systemctl start redis
sudo systemctl status redis

# Vérifier que Redis écoute
sudo netstat -tlnp | grep redis
# Doit montrer : 6379 et 16379


# ÉTAPE 5 : Vérifier la configuration cluster sur chaque nœud
# ────────────────────────────────────────────────────────────

# Sur CHAQUE nœud, vérifier :
redis-cli -h 192.168.1.10 CONFIG GET cluster-enabled
# Doit retourner : 1) "cluster-enabled" 2) "yes"

redis-cli -h 192.168.1.10 CLUSTER INFO
# Doit montrer : cluster_state:fail (normal, pas encore de cluster)


# ÉTAPE 6 : Créer le cluster
# ───────────────────────────

# Depuis n'importe quel nœud ou machine de gestion :
redis-cli --cluster create \
    192.168.1.10:6379 \
    192.168.1.11:6379 \
    192.168.1.12:6379 \
    192.168.1.13:6379 \
    192.168.1.14:6379 \
    192.168.1.15:6379 \
    --cluster-replicas 1 \
    --cluster-yes

# Paramètres :
# --cluster-replicas 1 : 1 replica par master
# --cluster-yes : Confirmer automatiquement (sinon demande interactivement)

# Le cluster sera créé avec :
# - 3 masters (10, 11, 12)
# - 3 replicas (13, 14, 15) assignées automatiquement

# Output attendu :
# >>> Performing hash slots allocation on 6 nodes...
# Master[0] -> Slots 0 - 5460
# Master[1] -> Slots 5461 - 10922
# Master[2] -> Slots 10923 - 16383
# Adding replica 192.168.1.13:6379 to 192.168.1.10:6379
# Adding replica 192.168.1.14:6379 to 192.168.1.11:6379
# Adding replica 192.168.1.15:6379 to 192.168.1.12:6379
# >>> Nodes configuration updated
# [OK] All 16384 slots covered.


# ÉTAPE 7 : Validation du cluster
# ────────────────────────────────

redis-cli --cluster check 192.168.1.10:6379

# Output attendu :
# 192.168.1.10:6379 (abc12345...) -> 0 keys | 5461 slots | 1 slaves.
# 192.168.1.11:6379 (def67890...) -> 0 keys | 5462 slots | 1 slaves.
# 192.168.1.12:6379 (ghi24680...) -> 0 keys | 5461 slots | 1 slaves.
# [OK] All 16384 slots covered.

# Vérifier l'état sur chaque nœud
redis-cli -c -h 192.168.1.10 CLUSTER INFO
# cluster_state:ok
# cluster_slots_assigned:16384
# cluster_slots_ok:16384
# cluster_known_nodes:6
# cluster_size:3
```

### Méthode 2 : Déploiement avec Docker Compose

```yaml
# docker-compose.yml
# Déploiement d'un cluster Redis 6 nœuds avec Docker

version: '3.8'

services:
  redis-node-1:
    image: redis:7.2-alpine
    container_name: redis-node-1
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 6379
    ports:
      - "6379:6379"
      - "16379:16379"
    volumes:
      - redis-node-1-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.2

  redis-node-2:
    image: redis:7.2-alpine
    container_name: redis-node-2
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 6379
    ports:
      - "6380:6379"
      - "16380:16379"
    volumes:
      - redis-node-2-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.3

  redis-node-3:
    image: redis:7.2-alpine
    container_name: redis-node-3
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 6379
    ports:
      - "6381:6379"
      - "16381:16379"
    volumes:
      - redis-node-3-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.4

  redis-node-4:
    image: redis:7.2-alpine
    container_name: redis-node-4
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 6379
    ports:
      - "6382:6379"
      - "16382:16379"
    volumes:
      - redis-node-4-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.5

  redis-node-5:
    image: redis:7.2-alpine
    container_name: redis-node-5
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 6379
    ports:
      - "6383:6379"
      - "16383:16379"
    volumes:
      - redis-node-5-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.6

  redis-node-6:
    image: redis:7.2-alpine
    container_name: redis-node-6
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 6379
    ports:
      - "6384:6379"
      - "16384:16379"
    volumes:
      - redis-node-6-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.7

  # Service d'initialisation du cluster
  redis-cluster-init:
    image: redis:7.2-alpine
    container_name: redis-cluster-init
    depends_on:
      - redis-node-1
      - redis-node-2
      - redis-node-3
      - redis-node-4
      - redis-node-5
      - redis-node-6
    networks:
      - redis-cluster
    command: >
      sh -c "
      echo 'Waiting for Redis nodes to start...' &&
      sleep 10 &&
      echo 'Creating cluster...' &&
      redis-cli --cluster create
        172.20.0.2:6379
        172.20.0.3:6379
        172.20.0.4:6379
        172.20.0.5:6379
        172.20.0.6:6379
        172.20.0.7:6379
        --cluster-replicas 1
        --cluster-yes &&
      echo 'Cluster created successfully!' &&
      redis-cli -h 172.20.0.2 CLUSTER INFO
      "

volumes:
  redis-node-1-data:
  redis-node-2-data:
  redis-node-3-data:
  redis-node-4-data:
  redis-node-5-data:
  redis-node-6-data:

networks:
  redis-cluster:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

```bash
# Déploiement du cluster Docker

# Lancer le cluster
docker-compose up -d

# Vérifier les logs de création
docker-compose logs redis-cluster-init

# Vérifier l'état du cluster
docker exec -it redis-node-1 redis-cli CLUSTER INFO
docker exec -it redis-node-1 redis-cli CLUSTER NODES

# Tester le cluster
docker exec -it redis-node-1 redis-cli -c SET test "Hello Cluster"
docker exec -it redis-node-2 redis-cli -c GET test

# Arrêter le cluster
docker-compose down

# Arrêter et supprimer les volumes (ATTENTION : perte de données)
docker-compose down -v
```

### Méthode 3 : Script d'automatisation Bash

```bash
#!/bin/bash
# deploy-redis-cluster.sh
# Script d'automatisation du déploiement d'un cluster Redis

set -e  # Arrêt en cas d'erreur

# ═══════════════════════════════════════════════════════════
# CONFIGURATION
# ═══════════════════════════════════════════════════════════

REDIS_VERSION="7.2.3"
CLUSTER_NODES=(
    "192.168.1.10:6379"
    "192.168.1.11:6379"
    "192.168.1.12:6379"
    "192.168.1.13:6379"
    "192.168.1.14:6379"
    "192.168.1.15:6379"
)
REPLICAS=1
REDIS_PASSWORD="YourSecurePassword"
SSH_USER="ubuntu"
SSH_KEY="/home/user/.ssh/id_rsa"

# ═══════════════════════════════════════════════════════════
# FONCTIONS
# ═══════════════════════════════════════════════════════════

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

error() {
    echo "[ERROR] $1" >&2
    exit 1
}

# Installer Redis sur un nœud distant
install_redis_on_node() {
    local host=$1
    log "Installing Redis ${REDIS_VERSION} on ${host}..."

    ssh -i ${SSH_KEY} ${SSH_USER}@${host} << 'ENDSSH'
        set -e

        # Dépendances
        sudo apt-get update
        sudo apt-get install -y build-essential tcl wget

        # Télécharger et compiler Redis
        cd /tmp
        wget https://download.redis.io/releases/redis-7.2.3.tar.gz
        tar xzf redis-7.2.3.tar.gz
        cd redis-7.2.3
        make
        sudo make install

        # Créer utilisateur et répertoires
        sudo useradd --system --user-group --no-create-home redis || true
        sudo mkdir -p /etc/redis /var/lib/redis /var/log/redis
        sudo chown redis:redis /var/lib/redis /var/log/redis

        # Nettoyage
        cd /tmp
        rm -rf redis-7.2.3 redis-7.2.3.tar.gz
ENDSSH

    log "Redis installed on ${host}"
}

# Configurer un nœud
configure_node() {
    local host=$1
    log "Configuring node ${host}..."

    # Créer le fichier de configuration
    cat > /tmp/redis-${host}.conf << EOF
port 6379
bind ${host}
protected-mode yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 15000
appendonly yes
requirepass ${REDIS_PASSWORD}
masterauth ${REDIS_PASSWORD}
maxmemory 30gb
maxmemory-policy allkeys-lru
dir /var/lib/redis
logfile /var/log/redis/redis.log
loglevel notice
EOF

    # Copier la configuration sur le nœud
    scp -i ${SSH_KEY} /tmp/redis-${host}.conf ${SSH_USER}@${host}:/tmp/redis.conf

    ssh -i ${SSH_KEY} ${SSH_USER}@${host} << 'ENDSSH'
        sudo mv /tmp/redis.conf /etc/redis/redis.conf
        sudo chown redis:redis /etc/redis/redis.conf
        sudo chmod 640 /etc/redis/redis.conf
ENDSSH

    rm /tmp/redis-${host}.conf
    log "Node ${host} configured"
}

# Créer le service systemd
create_systemd_service() {
    local host=$1
    log "Creating systemd service on ${host}..."

    ssh -i ${SSH_KEY} ${SSH_USER}@${host} << 'ENDSSH'
        cat <<EOF | sudo tee /etc/systemd/system/redis.service
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli -a ${REDIS_PASSWORD} shutdown
Restart=always
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

        sudo systemctl daemon-reload
        sudo systemctl enable redis
        sudo systemctl start redis
ENDSSH

    log "Systemd service created and started on ${host}"
}

# Optimisation système
optimize_system() {
    local host=$1
    log "Optimizing system on ${host}..."

    ssh -i ${SSH_KEY} ${SSH_USER}@${host} << 'ENDSSH'
        # THP
        echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
        echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

        # Sysctl
        sudo sysctl vm.overcommit_memory=1
        sudo sysctl net.core.somaxconn=65535
        sudo sysctl vm.swappiness=1

        # Rendre permanent
        cat <<EOF | sudo tee -a /etc/sysctl.conf
vm.overcommit_memory = 1
net.core.somaxconn = 65535
vm.swappiness = 1
EOF
ENDSSH

    log "System optimized on ${host}"
}

# Créer le cluster
create_cluster() {
    log "Creating Redis cluster..."

    local nodes_string=""
    for node in "${CLUSTER_NODES[@]}"; do
        nodes_string="${nodes_string} ${node}"
    done

    redis-cli -a ${REDIS_PASSWORD} --cluster create \
        ${nodes_string} \
        --cluster-replicas ${REPLICAS} \
        --cluster-yes

    log "Cluster created successfully!"
}

# Validation du cluster
validate_cluster() {
    log "Validating cluster..."

    local first_node=${CLUSTER_NODES[0]}
    local host=${first_node%:*}
    local port=${first_node#*:}

    redis-cli -h ${host} -p ${port} -a ${REDIS_PASSWORD} --cluster check ${first_node}

    log "Cluster validation complete!"
}

# ═══════════════════════════════════════════════════════════
# MAIN
# ═══════════════════════════════════════════════════════════

main() {
    log "Starting Redis Cluster deployment..."
    log "Nodes: ${CLUSTER_NODES[@]}"
    log "Replicas per master: ${REPLICAS}"

    # Déploiement sur chaque nœud
    for node in "${CLUSTER_NODES[@]}"; do
        host=${node%:*}

        log "Processing node ${host}..."

        optimize_system ${host}
        install_redis_on_node ${host}
        configure_node ${host}
        create_systemd_service ${host}

        log "Node ${host} ready!"
    done

    # Attendre que tous les nœuds soient prêts
    log "Waiting for all nodes to be ready..."
    sleep 10

    # Créer le cluster
    create_cluster

    # Validation
    validate_cluster

    log "=========================================="
    log "Redis Cluster deployment complete!"
    log "=========================================="
}

# Exécution
main
```

```bash
# Utilisation du script

# 1. Rendre le script exécutable
chmod +x deploy-redis-cluster.sh

# 2. Éditer les variables de configuration dans le script
# - CLUSTER_NODES : IPs des nœuds
# - REDIS_PASSWORD : Mot de passe
# - SSH_USER et SSH_KEY : Accès SSH

# 3. Exécuter le déploiement
./deploy-redis-cluster.sh

# 4. Vérifier le déploiement
redis-cli -h 192.168.1.10 -a YourSecurePassword CLUSTER INFO
```

## Validation post-déploiement

### Checklist de validation complète

```bash
#!/bin/bash
# validate-redis-cluster.sh
# Script de validation complète d'un cluster Redis

CLUSTER_NODE="192.168.1.10:6379"
PASSWORD="YourPassword"

echo "=========================================="
echo "Redis Cluster Validation"
echo "=========================================="
echo ""

# Test 1 : État du cluster
echo "Test 1: Cluster State"
echo "───────────────────────"
redis-cli -h ${CLUSTER_NODE%:*} -p ${CLUSTER_NODE#*:} -a ${PASSWORD} CLUSTER INFO | grep cluster_state
if [ $? -eq 0 ]; then
    echo "✓ Cluster state check passed"
else
    echo "✗ Cluster state check FAILED"
    exit 1
fi
echo ""

# Test 2 : Couverture des slots
echo "Test 2: Slots Coverage"
echo "──────────────────────"
redis-cli -h ${CLUSTER_NODE%:*} -a ${PASSWORD} --cluster check ${CLUSTER_NODE} | grep "All 16384 slots covered"
if [ $? -eq 0 ]; then
    echo "✓ All slots covered"
else
    echo "✗ Slots coverage FAILED"
    exit 1
fi
echo ""

# Test 3 : Nombre de nœuds
echo "Test 3: Node Count"
echo "──────────────────"
node_count=$(redis-cli -h ${CLUSTER_NODE%:*} -a ${PASSWORD} CLUSTER NODES | wc -l)
echo "Nodes in cluster: ${node_count}"
if [ ${node_count} -ge 3 ]; then
    echo "✓ Sufficient nodes"
else
    echo "✗ Insufficient nodes (minimum 3 required)"
    exit 1
fi
echo ""

# Test 4 : Masters et replicas
echo "Test 4: Masters and Replicas"
echo "─────────────────────────────"
masters=$(redis-cli -h ${CLUSTER_NODE%:*} -a ${PASSWORD} CLUSTER NODES | grep master | wc -l)
slaves=$(redis-cli -h ${CLUSTER_NODE%:*} -a ${PASSWORD} CLUSTER NODES | grep slave | wc -l)
echo "Masters: ${masters}"
echo "Replicas: ${slaves}"
if [ ${masters} -ge 3 ] && [ ${slaves} -ge 3 ]; then
    echo "✓ Proper master/replica distribution"
else
    echo "⚠ Check master/replica distribution"
fi
echo ""

# Test 5 : Test d'écriture et lecture
echo "Test 5: Write/Read Operations"
echo "──────────────────────────────"
redis-cli -c -h ${CLUSTER_NODE%:*} -a ${PASSWORD} SET test:validation "$(date)" > /dev/null
value=$(redis-cli -c -h ${CLUSTER_NODE%:*} -a ${PASSWORD} GET test:validation)
if [ ! -z "$value" ]; then
    echo "✓ Write/Read test passed"
    echo "  Value: ${value}"
else
    echo "✗ Write/Read test FAILED"
    exit 1
fi
echo ""

# Test 6 : Test de redirection
echo "Test 6: Redirection Test"
echo "────────────────────────"
# Écrire une clé qui sera probablement sur un autre slot
redis-cli -c -h ${CLUSTER_NODE%:*} -a ${PASSWORD} SET test:redirect:12345 "redirect-test" > /dev/null
if [ $? -eq 0 ]; then
    echo "✓ Redirection working"
else
    echo "✗ Redirection FAILED"
    exit 1
fi
echo ""

# Test 7 : Latence
echo "Test 7: Latency Check"
echo "─────────────────────"
latency=$(redis-cli -h ${CLUSTER_NODE%:*} -a ${PASSWORD} --latency-history -i 1 -c 10 | grep avg | awk '{print $5}')
echo "Average latency: ${latency}"
echo ""

# Test 8 : Info replication
echo "Test 8: Replication Status"
echo "──────────────────────────"
redis-cli -h ${CLUSTER_NODE%:*} -a ${PASSWORD} INFO replication | grep -E "role|connected_slaves|master_link_status"
echo ""

# Résumé
echo "=========================================="
echo "✓ Validation completed successfully!"
echo "=========================================="
```

### Tests fonctionnels

```bash
# Test de la fonctionnalité cluster

# 1. Test multi-clés avec hash tags
redis-cli -c -h 192.168.1.10 -a Password
127.0.0.1:6379> MSET {user:1000}:name "John" {user:1000}:email "john@example.com"
OK

127.0.0.1:6379> MGET {user:1000}:name {user:1000}:email
1) "John"
2) "john@example.com"

# 2. Test de redirection
127.0.0.1:6379> SET user:2000 "Jane"
-> Redirected to slot [8834] located at 192.168.1.11:6379
OK

# 3. Test de failover
# Sur un master
127.0.0.1:6379> DEBUG SLEEP 30

# Observer le failover automatique
redis-cli -h 192.168.1.10 CLUSTER NODES
# La replica devrait devenir master

# 4. Test de performance
redis-benchmark -h 192.168.1.10 -a Password -c 50 -n 100000 -t get,set --cluster
```

## Troubleshooting du déploiement

### Problèmes courants et solutions

```
┌─────────────────────────────────────────────────────────────┐
│              Problèmes de déploiement courants              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 1 : "Cluster state: fail"                          │
│ ═══════════════════════════════════                         │
│ Symptôme : cluster_state:fail après création                │
│                                                             │
│ Causes possibles :                                          │
│ ├─ Tous les slots ne sont pas assignés                      │
│ ├─ Un ou plusieurs nœuds sont inaccessibles                 │
│ └─ Configuration réseau incorrecte                          │
│                                                             │
│ Diagnostic :                                                │
│ redis-cli --cluster check <node>                            │
│                                                             │
│ Solution :                                                  │
│ redis-cli --cluster fix <node>                              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 2 : "ERR Slot already covered"                     │
│ ═════════════════════════════════════════                   │
│ Symptôme : Erreur lors de la création du cluster            │
│                                                             │
│ Cause : Un nodes.conf existe déjà avec une configuration    │
│                                                             │
│ Solution :                                                  │
│ Sur CHAQUE nœud :                                           │
│ 1. Arrêter Redis : systemctl stop redis                     │
│ 2. Supprimer : rm /var/lib/redis/nodes.conf                 │
│ 3. Redémarrer : systemctl start redis                       │
│ 4. Recréer le cluster                                       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 3 : Nœuds ne se voient pas (Gossip)                │
│ ═════════════════════════════════════════                   │
│ Symptôme : CLUSTER NODES montre seulement le nœud local     │
│                                                             │
│ Cause : Firewall bloque port 16379 (cluster bus)            │
│                                                             │
│ Diagnostic :                                                │
│ nc -zv <other-node-ip> 16379                                │
│                                                             │
│ Solution :                                                  │
│ sudo ufw allow 16379/tcp                                    │
│ sudo firewall-cmd --add-port=16379/tcp --permanent          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PROBLÈME 4 : "Connection refused" en mode cluster           │
│ ═══════════════════════════════════════════════             │
│ Symptôme : Impossible de se connecter au nœud               │
│                                                             │
│ Cause : bind 127.0.0.1 (localhost only)                     │
│                                                             │
│ Solution :                                                  │
│ Dans redis.conf :                                           │
│ bind 0.0.0.0  # ou bind <IP-publique>                       │
│ protected-mode no  # si bind != 127.0.0.1                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Commandes de diagnostic

```bash
# Diagnostic complet d'un cluster Redis

# 1. Vérifier l'état détaillé
redis-cli --cluster check 192.168.1.10:6379

# 2. Voir la configuration effective
redis-cli -h 192.168.1.10 CONFIG GET "*cluster*"

# 3. Logs système
sudo journalctl -u redis -f

# Ou
tail -f /var/log/redis/redis.log

# 4. Tester la connectivité réseau entre tous les nœuds
for node in 192.168.1.11 192.168.1.12 192.168.1.13; do
    echo "Testing $node..."
    nc -zv $node 6379
    nc -zv $node 16379
done

# 5. Analyser le fichier nodes.conf
cat /var/lib/redis/nodes.conf

# 6. Vérifier les processus
ps aux | grep redis

# 7. Vérifier les ports en écoute
sudo netstat -tlnp | grep redis

# 8. Métriques du cluster
redis-cli -h 192.168.1.10 INFO cluster
redis-cli -h 192.168.1.10 CLUSTER INFO
redis-cli -h 192.168.1.10 CLUSTER NODES

# 9. Test de latence
redis-cli -h 192.168.1.10 --latency
redis-cli -h 192.168.1.10 --latency-history

# 10. Vérifier la réplication
redis-cli -h 192.168.1.10 INFO replication
```

## Best Practices de déploiement

```
┌─────────────────────────────────────────────────────────────┐
│         Best Practices pour déploiement Production          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ✓ PLANIFICATION                                             │
│   ├─ Dimensionner correctement (RAM, CPU, réseau)           │
│   ├─ Prévoir les replicas (1 minimum, 2 recommandé)         │
│   ├─ Documenter l'architecture                              │
│   └─ Définir les RPO/RTO                                    │
│                                                             │
│ ✓ SÉCURITÉ                                                  │
│   ├─ Utiliser ACLs (Redis 6+)                               │
│   ├─ Activer TLS/SSL si données sensibles                   │
│   ├─ Bind sur interfaces privées uniquement                 │
│   ├─ Firewall : whitelist des IPs                           │
│   └─ Renommer/désactiver commandes dangereuses              │
│                                                             │
│ ✓ HAUTE DISPONIBILITÉ                                       │
│   ├─ Minimum 3 masters pour majorité                        │
│   ├─ Replicas sur des racks/AZ différents                   │
│   ├─ Configurer cluster-node-timeout approprié              │
│   └─ Tester les procédures de failover                      │
│                                                             │
│ ✓ PERSISTENCE                                               │
│   ├─ Activer AOF pour durabilité                            │
│   ├─ Configurer RDB pour snapshots                          │
│   ├─ Backups automatisés quotidiens                         │
│   └─ Tester la restauration régulièrement                   │
│                                                             │
│ ✓ MONITORING                                                │
│   ├─ Métriques : cluster_state, memory, latency             │
│   ├─ Alertes : nœud down, slots non couverts                │
│   ├─ Logs centralisés                                       │
│   └─ Dashboards (Grafana)                                   │
│                                                             │
│ ✓ DOCUMENTATION                                             │
│   ├─ Runbooks opérationnels                                 │
│   ├─ Diagrammes d'architecture                              │
│   ├─ Procédures de disaster recovery                        │
│   └─ Contacts on-call                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Conclusion

Le déploiement d'un cluster Redis requiert une préparation minutieuse et une attention particulière aux détails de configuration. Les points critiques à retenir :

1. **Préparation système** : Optimisations Linux, firewall, connectivité réseau
2. **Configuration cluster** : Paramètres appropriés pour l'environnement cible
3. **Méthode de déploiement** : Choisir entre manuel, automatisé, ou containerisé selon contexte
4. **Validation rigoureuse** : Tests complets avant mise en production
5. **Documentation** : Procédures et runbooks pour les opérations courantes

Un déploiement réussi pose les fondations d'un cluster stable, performant et maintenable en production.

---

**Points clés à retenir :**

- **Configuration système** : THP, overcommit, somaxconn indispensables
- **Ports cluster** : 6379 (client) + 16379 (bus) doivent être accessibles
- **cluster-enabled yes** : Directive obligatoire dans redis.conf
- **Minimum 3 masters** : Pour avoir une majorité fonctionnelle
- **Validation complète** : Tester tous les aspects avant production
- **Documentation** : Runbooks et procédures essentiels
- **Monitoring** : Mettre en place dès le déploiement initial
- **Backups** : Configurer immédiatement la persistence et backups

La section suivante (11.5) détaillera la gestion opérationnelle des nœuds (ajout, suppression, resharding).

⏭️ [Gestion des nœuds (ajout, suppression, resharding)](/11-architecture-distribuee-scaling/05-gestion-noeuds-resharding.md)
