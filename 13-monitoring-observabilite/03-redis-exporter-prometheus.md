🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.3 Redis Exporter et Prometheus

## Introduction

Redis Exporter est le pont entre Redis et Prometheus, transformant les métriques Redis internes en format time-series exploitable. Cette section couvre l'architecture, la configuration, et les optimisations nécessaires pour un monitoring production-ready.

### Pourquoi Redis Exporter ?

**Le problème** : Redis expose ses métriques via `INFO`, mais :
- Format texte non-structuré
- Pas de stockage historique
- Pas d'agrégation multi-instances
- Pas d'alerting natif

**La solution** : Redis Exporter + Prometheus
```
┌─────────────┐
│   Redis     │ ←─── INFO command
│ (instance)  │
└──────┬──────┘
       │
       │ Parse & Transform
       ▼
┌─────────────┐
│   Redis     │ ←─── HTTP /metrics
│  Exporter   │      (Prometheus format)
└──────┬──────┘
       │
       │ Scrape (Pull)
       ▼
┌─────────────┐
│ Prometheus  │
│ (TSDB)      │
└──────┬──────┘
       │
       │ Query (PromQL)
       ▼
┌─────────────┐
│   Grafana   │
│ (Dashboard) │
└─────────────┘
```

## 1. Architecture et Composants

### 1.1 Redis Exporter

**Projet officiel** : [oliver006/redis_exporter](https://github.com/oliver006/redis_exporter)

**Caractéristiques** :
- Écrit en Go (performant, single binary)
- Support Redis 2.x → 7.x
- Support Sentinel et Cluster
- Export de métriques custom via Lua scripts
- Support TLS/SSL
- Multi-tenant (plusieurs instances Redis)

**Métriques exposées** : 100+ métriques natives + custom

### 1.2 Modes de déploiement

#### Mode 1 : Exporter par instance (Recommandé)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Redis Master │    │ Redis Replica│    │ Redis Replica│
│   :6379      │    │   :6379      │    │   :6379      │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       │                   │                   │
┌──────▼───────┐    ┌──────▼───────┐    ┌──────▼───────┐
│  Exporter    │    │  Exporter    │    │  Exporter    │
│   :9121      │    │   :9121      │    │   :9121      │
└──────────────┘    └──────────────┘    └──────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    ┌──────▼───────┐
                    │  Prometheus  │
                    │   :9090      │
                    └──────────────┘
```

**Avantages** :
- Isolation des pannes (un exporter down n'affecte pas les autres)
- Simplicité de déploiement (sidecar pattern)
- Faible latence (localhost)

**Inconvénients** :
- Plus de ressources (1 exporter par Redis)
- Gestion multi-instances

#### Mode 2 : Exporter centralisé

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Redis Master │    │ Redis Replica│    │ Redis Replica│
│   :6379      │    │   :6379      │    │   :6379      │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    ┌──────▼───────┐
                    │   Exporter   │
                    │  (multi-     │
                    │   target)    │
                    │   :9121      │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Prometheus  │
                    └──────────────┘
```

**Avantages** :
- Moins de ressources (1 seul exporter)
- Configuration centralisée

**Inconvénients** :
- SPOF (Single Point of Failure)
- Latence réseau vers chaque Redis
- Complexité de configuration

### 1.3 Choix du mode de déploiement

| Critère | Per-instance | Centralisé |
|---------|--------------|------------|
| **Nb instances** | < 20 | > 20 |
| **Infrastructure** | Kubernetes, Docker | VMs, Bare metal |
| **Criticité** | Haute (HA) | Moyenne |
| **Ressources dispo** | Suffisantes | Limitées |
| **Recommandation** | ✅ Préféré | Acceptable |

## 2. Installation de Redis Exporter

### 2.1 Installation via Docker

**Image officielle** :
```bash
docker pull oliver006/redis_exporter:latest
```

**Démarrage simple** :
```bash
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  oliver006/redis_exporter:latest \
  --redis.addr=redis://redis-server:6379
```

**Avec authentification** :
```bash
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  oliver006/redis_exporter:latest \
  --redis.addr=redis://redis-server:6379 \
  --redis.password=my_secure_password
```

**Avec TLS** :
```bash
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  -v /path/to/certs:/certs \
  oliver006/redis_exporter:latest \
  --redis.addr=rediss://redis-server:6379 \
  --redis.password=my_password \
  --tls-client-key-file=/certs/client-key.pem \
  --tls-client-cert-file=/certs/client-cert.pem \
  --tls-ca-cert-file=/certs/ca-cert.pem \
  --skip-tls-verification=false
```

### 2.2 Installation via Docker Compose

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  redis:
    image: redis:7.2-alpine
    container_name: redis-server
    ports:
      - "6379:6379"
    command: redis-server --requirepass my_password
    volumes:
      - redis-data:/data
    networks:
      - redis-net

  redis-exporter:
    image: oliver006/redis_exporter:latest
    container_name: redis-exporter
    ports:
      - "9121:9121"
    environment:
      REDIS_ADDR: "redis://redis-server:6379"
      REDIS_PASSWORD: "my_password"
    depends_on:
      - redis
    networks:
      - redis-net
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - redis-net
    restart: unless-stopped

volumes:
  redis-data:
  prometheus-data:

networks:
  redis-net:
    driver: bridge
```

### 2.3 Installation sur Kubernetes

**Déploiement avec Sidecar Pattern** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-with-exporter
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      # Redis container
      - name: redis
        image: redis:7.2-alpine
        ports:
        - containerPort: 6379
          name: redis
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        resources:
          requests:
            memory: "4Gi"
            cpu: "1000m"
          limits:
            memory: "8Gi"
            cpu: "2000m"
        volumeMounts:
        - name: redis-data
          mountPath: /data
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

      # Redis Exporter sidecar
      - name: redis-exporter
        image: oliver006/redis_exporter:v1.55.0
        ports:
        - containerPort: 9121
          name: metrics
        env:
        - name: REDIS_ADDR
          value: "redis://localhost:6379"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /health
            port: 9121
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 9121
          initialDelaySeconds: 5
          periodSeconds: 5

      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: production
  labels:
    app: redis
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: redis
  - port: 9121
    targetPort: 9121
    name: metrics
  selector:
    app: redis

---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: production
type: Opaque
stringData:
  password: "my_secure_redis_password_change_me"
```

### 2.4 Installation binaire (Linux)

```bash
# Télécharger la dernière release
EXPORTER_VERSION="v1.55.0"
wget https://github.com/oliver006/redis_exporter/releases/download/${EXPORTER_VERSION}/redis_exporter-${EXPORTER_VERSION}.linux-amd64.tar.gz

# Extraire
tar xvzf redis_exporter-${EXPORTER_VERSION}.linux-amd64.tar.gz

# Déplacer le binaire
sudo mv redis_exporter-${EXPORTER_VERSION}.linux-amd64/redis_exporter /usr/local/bin/

# Rendre exécutable
sudo chmod +x /usr/local/bin/redis_exporter

# Créer un utilisateur système
sudo useradd --no-create-home --shell /bin/false redis_exporter

# Créer le fichier systemd
sudo tee /etc/systemd/system/redis_exporter.service > /dev/null <<EOF
[Unit]
Description=Redis Exporter
After=network.target

[Service]
Type=simple
User=redis_exporter
Group=redis_exporter
ExecStart=/usr/local/bin/redis_exporter \\
  --redis.addr=redis://localhost:6379 \\
  --redis.password=${REDIS_PASSWORD} \\
  --web.listen-address=:9121 \\
  --web.telemetry-path=/metrics
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Démarrer le service
sudo systemctl daemon-reload
sudo systemctl enable redis_exporter
sudo systemctl start redis_exporter

# Vérifier le statut
sudo systemctl status redis_exporter
```

## 3. Configuration avancée de Redis Exporter

### 3.1 Options de ligne de commande

**Options essentielles** :
```bash
redis_exporter \
  # Connexion Redis
  --redis.addr=redis://redis-host:6379 \
  --redis.password=secret \
  --redis.user=monitoring_user \          # ACL (Redis 6+)

  # Connexion multiple (multi-target)
  --redis.addr=redis://master:6379,redis://replica1:6379,redis://replica2:6379 \

  # TLS/SSL
  --redis.addr=rediss://redis-host:6379 \
  --tls-client-key-file=/path/to/key.pem \
  --tls-client-cert-file=/path/to/cert.pem \
  --tls-ca-cert-file=/path/to/ca.pem \
  --skip-tls-verification=false \

  # Exporter HTTP
  --web.listen-address=:9121 \
  --web.telemetry-path=/metrics \

  # Namespace des métriques
  --namespace=redis \

  # Métriques spécifiques
  --check-keys=user:*,session:* \          # Pattern de clés à monitorer
  --check-single-keys=important_key1,key2 \ # Clés spécifiques
  --check-streams=stream1,stream2 \        # Streams à monitorer
  --check-single-streams=critical_stream \ # Stream critique

  # Performance
  --connection-timeout=15s \
  --redis-metrics-only=false \             # Inclure Go runtime metrics
  --include-system-metrics=true \

  # Scripts Lua custom
  --script=/path/to/custom_metrics.lua \

  # Logging
  --log-format=txt \                       # txt ou json
  --debug=false \
  --verbose=false
```

### 3.2 Variables d'environnement

```bash
# Préférer les variables d'environnement pour les secrets
export REDIS_ADDR="redis://redis-host:6379"
export REDIS_PASSWORD="my_secret_password"
export REDIS_USER="exporter_user"

# Pour multi-target
export REDIS_ADDR="redis://master:6379,redis://replica:6379"
export REDIS_PASSWORD="pass1,pass2"  # Correspondance 1:1

# TLS
export REDIS_EXPORTER_TLS_CLIENT_KEY_FILE="/certs/key.pem"
export REDIS_EXPORTER_TLS_CLIENT_CERT_FILE="/certs/cert.pem"
export REDIS_EXPORTER_TLS_CA_CERT_FILE="/certs/ca.pem"

redis_exporter
```

### 3.3 Configuration multi-instance

**Fichier de configuration** (redis-instances.yml) :
```yaml
redis_instances:
  - name: "master-prod"
    addr: "redis://redis-master.prod:6379"
    password: "${REDIS_MASTER_PASSWORD}"
    labels:
      env: "production"
      role: "master"
      region: "us-east-1"

  - name: "replica1-prod"
    addr: "redis://redis-replica1.prod:6379"
    password: "${REDIS_REPLICA_PASSWORD}"
    labels:
      env: "production"
      role: "replica"
      region: "us-east-1"

  - name: "replica2-prod"
    addr: "redis://redis-replica2.prod:6379"
    password: "${REDIS_REPLICA_PASSWORD}"
    labels:
      env: "production"
      role: "replica"
      region: "us-west-2"

  - name: "master-staging"
    addr: "redis://redis-master.staging:6379"
    password: "${REDIS_STAGING_PASSWORD}"
    labels:
      env: "staging"
      role: "master"
      region: "us-east-1"
```

**Lancement** :
```bash
redis_exporter --redis-file=/path/to/redis-instances.yml
```

### 3.4 Monitoring de clés spécifiques

**Use case** : Monitorer des clés business-critical

```bash
redis_exporter \
  --redis.addr=redis://localhost:6379 \
  --check-keys='user:*' \                    # Toutes les clés user:*
  --check-keys='session:*' \                 # Toutes les sessions
  --check-keys='cache:product:*' \           # Cache produits
  --check-single-keys='stats:daily_revenue' \# Clé métrique business
  --check-single-keys='config:feature_flags'
```

**Métriques exposées** :
```
# Nombre de clés matchant le pattern
redis_key_count{db="0",key="user:*"} 15478

# Taille en mémoire d'une clé spécifique
redis_key_size{db="0",key="stats:daily_revenue"} 245

# TTL de la clé
redis_key_ttl{db="0",key="session:abc123"} 3540
```

**Requête Prometheus** :
```promql
# Alerter si clé critique absente
absent(redis_key_size{key="config:feature_flags"}) == 1
```

### 3.5 Monitoring de Streams

```bash
redis_exporter \
  --redis.addr=redis://localhost:6379 \
  --check-streams='events:*' \
  --check-single-streams='orders:pending'
```

**Métriques exposées** :
```
# Longueur du stream
redis_stream_length{db="0",stream="orders:pending"} 1247

# Nombre de groupes de consommateurs
redis_stream_groups{db="0",stream="orders:pending"} 3

# Messages en attente par groupe
redis_stream_group_pending{db="0",stream="orders:pending",group="workers"} 42
```

### 3.6 Scripts Lua personnalisés

**Cas d'usage** : Exposer des métriques business spécifiques

**Script custom : active_users.lua** :
```lua
-- Retourner le nombre d'utilisateurs actifs (exemple)
local active_count = redis.call('SCARD', 'users:active')
local premium_count = redis.call('SCARD', 'users:premium')

return {
  {
    metric = "redis_active_users",
    value = active_count,
    labels = {user_type = "standard"}
  },
  {
    metric = "redis_active_users",
    value = premium_count,
    labels = {user_type = "premium"}
  }
}
```

**Lancement** :
```bash
redis_exporter \
  --redis.addr=redis://localhost:6379 \
  --script=/path/to/active_users.lua
```

**Métrique Prometheus** :
```
redis_active_users{user_type="standard"} 8547
redis_active_users{user_type="premium"} 1245
```

## 4. Configuration Prometheus

### 4.1 Configuration de base (prometheus.yml)

```yaml
global:
  scrape_interval: 15s      # Fréquence de collecte par défaut
  evaluation_interval: 15s   # Fréquence d'évaluation des règles
  external_labels:
    cluster: 'production'
    datacenter: 'us-east-1'

# Configuration des alertes
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'

# Chargement des règles
rule_files:
  - '/etc/prometheus/rules/*.yml'

# Configuration du scraping
scrape_configs:
  # Redis instances
  - job_name: 'redis'
    static_configs:
      - targets:
          - 'redis-exporter-master:9121'
          - 'redis-exporter-replica1:9121'
          - 'redis-exporter-replica2:9121'

    # Ajouter des labels custom
    relabel_configs:
      # Extraire l'instance depuis le target
      - source_labels: [__address__]
        target_label: instance

      # Identifier le rôle (master/replica) depuis le hostname
      - source_labels: [__address__]
        regex: '.*master.*'
        target_label: redis_role
        replacement: 'master'

      - source_labels: [__address__]
        regex: '.*replica.*'
        target_label: redis_role
        replacement: 'replica'

    # Timeouts
    scrape_timeout: 10s

    # Intervalle spécifique (override global)
    scrape_interval: 15s
```

### 4.2 Service Discovery Kubernetes

**Annotations-based discovery** :
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - production
            - staging

    relabel_configs:
      # Scraper uniquement les pods avec l'annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      # Extraire le port depuis l'annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: $1:9121

      # Extraire le path depuis l'annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Ajouter des labels du pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace

      - source_labels: [__meta_kubernetes_pod_name]
        target_label: kubernetes_pod_name

      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app

      - source_labels: [__meta_kubernetes_pod_label_env]
        target_label: env
```

**Pod annotations** (voir section 2.3) :
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9121"
  prometheus.io/path: "/metrics"
```

### 4.3 Service Discovery Consul

```yaml
scrape_configs:
  - job_name: 'redis-consul'
    consul_sd_configs:
      - server: 'consul.service.consul:8500'
        services:
          - 'redis-exporter'

    relabel_configs:
      # Extraire les métadonnées Consul
      - source_labels: [__meta_consul_service_metadata_env]
        target_label: env

      - source_labels: [__meta_consul_service_metadata_role]
        target_label: redis_role

      - source_labels: [__meta_consul_service_metadata_region]
        target_label: region
```

**Enregistrement Consul** :
```json
{
  "service": {
    "name": "redis-exporter",
    "port": 9121,
    "tags": ["metrics", "redis"],
    "meta": {
      "env": "production",
      "role": "master",
      "region": "us-east-1"
    },
    "checks": [
      {
        "http": "http://localhost:9121/health",
        "interval": "10s",
        "timeout": "2s"
      }
    ]
  }
}
```

### 4.4 Configuration multi-datacenter

```yaml
scrape_configs:
  # Datacenter US-EAST
  - job_name: 'redis-us-east'
    static_configs:
      - targets:
          - 'redis-exporter-master-use1:9121'
          - 'redis-exporter-replica1-use1:9121'
        labels:
          datacenter: 'us-east-1'
          cluster: 'production'

  # Datacenter US-WEST
  - job_name: 'redis-us-west'
    static_configs:
      - targets:
          - 'redis-exporter-master-usw2:9121'
          - 'redis-exporter-replica1-usw2:9121'
        labels:
          datacenter: 'us-west-2'
          cluster: 'production'

  # Datacenter EU
  - job_name: 'redis-eu'
    static_configs:
      - targets:
          - 'redis-exporter-master-eu1:9121'
          - 'redis-exporter-replica1-eu1:9121'
        labels:
          datacenter: 'eu-central-1'
          cluster: 'production'
```

### 4.5 Sentinel monitoring

```yaml
scrape_configs:
  # Redis Sentinel instances
  - job_name: 'redis-sentinel'
    static_configs:
      - targets:
          - 'sentinel-exporter-1:9355'  # Port différent pour Sentinel
          - 'sentinel-exporter-2:9355'
          - 'sentinel-exporter-3:9355'

    relabel_configs:
      - source_labels: [__address__]
        target_label: sentinel_instance
```

**Exporter Sentinel** :
```bash
# Sentinel utilise un exporter différent
redis_exporter \
  --redis.addr=redis://sentinel-host:26379 \
  --is-sentinel=true \
  --web.listen-address=:9355
```

### 4.6 Cluster monitoring

```yaml
scrape_configs:
  # Redis Cluster nodes
  - job_name: 'redis-cluster'
    static_configs:
      - targets:
          - 'cluster-node-1:9121'
          - 'cluster-node-2:9121'
          - 'cluster-node-3:9121'
          - 'cluster-node-4:9121'
          - 'cluster-node-5:9121'
          - 'cluster-node-6:9121'

    relabel_configs:
      # Identifier les masters vs replicas
      - source_labels: [__address__]
        regex: 'cluster-node-[135].*'
        target_label: cluster_role
        replacement: 'master'

      - source_labels: [__address__]
        regex: 'cluster-node-[246].*'
        target_label: cluster_role
        replacement: 'replica'

      # Ajouter l'ID du nœud
      - source_labels: [__address__]
        regex: 'cluster-node-([0-9]).*'
        target_label: node_id
        replacement: '$1'
```

## 5. Métriques exposées par Redis Exporter

### 5.1 Catégories de métriques

#### Métriques de base (toujours présentes)

```
# Disponibilité
redis_up{addr="redis://localhost:6379"} 1

# Métriques INFO
redis_uptime_in_seconds 864000
redis_connected_clients 247
redis_blocked_clients 5
redis_used_memory_bytes 2147483648
redis_used_memory_rss_bytes 2415919104
redis_used_memory_peak_bytes 3221225472
redis_mem_fragmentation_ratio 1.12
redis_maxmemory_bytes 4294967296

# Statistiques
redis_commands_processed_total 98745632
redis_keyspace_hits_total 8547123
redis_keyspace_misses_total 1245632
redis_evicted_keys_total 0
redis_expired_keys_total 45123
redis_rejected_connections_total 0

# Réplication
redis_connected_slaves 2
redis_repl_backlog_size 1048576
redis_master_repl_offset 87452369
redis_slave_repl_offset 87452369  # Si replica
redis_master_link_up 1            # Si replica
redis_master_last_io_seconds 0    # Si replica

# Persistance
redis_rdb_changes_since_last_save 12847
redis_rdb_last_save_timestamp_seconds 1702305600
redis_aof_enabled 1
redis_aof_current_size_bytes 524288000
redis_aof_base_size_bytes 104857600

# Keyspace (par DB)
redis_db_keys{db="db0"} 1523478
redis_db_keys_expiring{db="db0"} 458963
redis_db_avg_ttl_seconds{db="db0"} 3600
```

#### Métriques de commandes

```
# Par commande
redis_commands_duration_seconds_total{cmd="get"} 25.478963
redis_commands_total{cmd="get"} 12547896
redis_command_call_duration_seconds{cmd="get",quantile="0.5"} 0.000002
redis_command_call_duration_seconds{cmd="get",quantile="0.99"} 0.000010
```

#### Métriques Cluster (si applicable)

```
redis_cluster_enabled 1
redis_cluster_state 1  # 1=OK, 0=FAIL
redis_cluster_slots_assigned 16384
redis_cluster_slots_ok 16384
redis_cluster_slots_pfail 0
redis_cluster_slots_fail 0
redis_cluster_known_nodes 6
redis_cluster_size 3
redis_cluster_current_epoch 127
```

#### Métriques custom (--check-keys)

```
redis_key_size{db="db0",key="user:12345"} 1247
redis_key_ttl_seconds{db="db0",key="user:12345"} 3540
redis_key_count{db="db0",key="user:*"} 15478
```

### 5.2 Mapping INFO → Métriques Prometheus

| Redis INFO | Prometheus Metric | Type |
|------------|-------------------|------|
| `connected_clients` | `redis_connected_clients` | Gauge |
| `used_memory` | `redis_used_memory_bytes` | Gauge |
| `mem_fragmentation_ratio` | `redis_mem_fragmentation_ratio` | Gauge |
| `total_commands_processed` | `redis_commands_processed_total` | Counter |
| `keyspace_hits` | `redis_keyspace_hits_total` | Counter |
| `keyspace_misses` | `redis_keyspace_misses_total` | Counter |
| `evicted_keys` | `redis_evicted_keys_total` | Counter |
| `expired_keys` | `redis_expired_keys_total` | Counter |
| `rdb_last_save_time` | `redis_rdb_last_save_timestamp_seconds` | Gauge |

### 5.3 Labels ajoutés par l'exporter

```
redis_up{
  addr="redis://redis-master:6379",
  alias="master-prod",
  instance="redis-exporter:9121",
  job="redis"
} 1
```

**Labels standards** :
- `addr` : Adresse Redis complète
- `alias` : Nom logique (si configuré)
- `instance` : Instance de l'exporter
- `job` : Job Prometheus

**Labels custom** (via relabel_configs) :
- `env` : production/staging/dev
- `role` : master/replica
- `region` : us-east-1, eu-west-1, etc.
- `datacenter` : DC physique

## 6. Optimisation et Tuning

### 6.1 Tuning de la fréquence de scraping

**Compromise latence vs précision** :

| Scrape Interval | Précision | Charge | Use Case |
|-----------------|-----------|--------|----------|
| 5s | Très haute | Élevée | Critical systems |
| 15s | Haute | Moyenne | Production (recommandé) |
| 30s | Moyenne | Faible | Staging |
| 60s | Basse | Très faible | Dev/Test |

**Configuration adaptative** :
```yaml
scrape_configs:
  # Production : 15s
  - job_name: 'redis-production'
    scrape_interval: 15s
    static_configs:
      - targets: ['redis-prod:9121']

  # Staging : 30s
  - job_name: 'redis-staging'
    scrape_interval: 30s
    static_configs:
      - targets: ['redis-staging:9121']

  # Dev : 60s
  - job_name: 'redis-dev'
    scrape_interval: 60s
    static_configs:
      - targets: ['redis-dev:9121']
```

### 6.2 Réduction de la cardinalité

**Problème** : Trop de labels uniques → Explosion de séries temporelles

**Mauvaise pratique** :
```yaml
# ❌ Ajouter des IDs dynamiques comme labels
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_uid]
    target_label: pod_uid  # Mauvais : UID unique par pod
```

**Bonne pratique** :
```yaml
# ✅ Limiter aux labels nécessaires
relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace

  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app

  # Pas d'UID, pas de timestamp, pas de hash
```

**Limiter les métriques de commandes** :
```bash
# Collecter uniquement les commandes fréquentes
redis_exporter \
  --redis.addr=redis://localhost:6379 \
  --export-client-list=false \        # Désactiver client list
  --skip-tls-verification=true        # Si TLS non requis
```

### 6.3 Optimisation mémoire de Prometheus

**Configuration TSDB** :
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

# Ligne de commande
storage:
  tsdb:
    path: /prometheus
    retention.time: 30d          # Rétention 30 jours
    retention.size: 50GB         # Limite taille
    min-block-duration: 2h       # Durée min d'un bloc
    max-block-duration: 36h      # Durée max d'un bloc
```

**Estimation de la taille** :
```
Calcul approximatif :
- 1 métrique × 1 label × scrape_interval 15s = ~2 bytes/sample
- Redis Exporter expose ~150 métriques de base
- 150 metrics × 2 bytes × (86400s / 15s) = 1.7 MB/jour/instance

Pour 10 instances × 30 jours rétention :
10 × 30 × 1.7 MB = 510 MB
+ overhead (indexation, compression) ≈ 1 GB
```

### 6.4 Compression et downsampling

**Recording rules** pour pré-agréger :
```yaml
# /etc/prometheus/rules/redis_recording_rules.yml
groups:
  - name: redis_aggregations
    interval: 60s
    rules:
      # Hit ratio pré-calculé (5m window)
      - record: redis:hit_ratio:rate5m
        expr: |
          rate(redis_keyspace_hits_total[5m]) /
          (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

      # Utilisation mémoire en %
      - record: redis:memory_used:percent
        expr: |
          (redis_used_memory_bytes / redis_maxmemory_bytes) * 100

      # OPS/sec par rôle (master/replica)
      - record: redis:ops_per_sec:by_role
        expr: |
          sum(rate(redis_commands_processed_total[5m])) by (redis_role)

      # Évictions rate agrégé
      - record: redis:evictions:rate5m
        expr: |
          sum(rate(redis_evicted_keys_total[5m]))
```

**Avantages** :
- Requêtes Grafana plus rapides (métriques pré-calculées)
- Moins de CPU Prometheus lors des requêtes
- Historique agrégé plus compact

## 7. Haute Disponibilité

### 7.1 HA de l'exporter

**Option 1 : Multiple exporters (recommandé)** :
```
┌──────────────┐    ┌──────────────┐
│ Exporter 1   │    │ Exporter 2   │
│ (node A)     │    │ (node B)     │
└──────┬───────┘    └──────┬───────┘
       │                   │
       └───────┬───────────┘
               │
        ┌──────▼───────┐
        │ Prometheus   │
        │ (déduplique) │
        └──────────────┘
```

**Configuration Prometheus** :
```yaml
scrape_configs:
  - job_name: 'redis-ha'
    static_configs:
      # Scraper les deux exporters
      - targets:
          - 'redis-exporter-1:9121'
          - 'redis-exporter-2:9121'
        labels:
          redis_addr: 'redis://redis-master:6379'

    # Prometheus déduplique automatiquement
    honor_labels: true
```

**Option 2 : Exporter derrière load balancer** :
```
┌──────────────┐    ┌──────────────┐
│ Exporter 1   │    │ Exporter 2   │
└──────┬───────┘    └──────┬───────┘
       │                   │
       └───────┬───────────┘
               │
        ┌──────▼───────┐
        │Load Balancer │
        │  (HAProxy)   │
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │ Prometheus   │
        └──────────────┘
```

**HAProxy config** :
```
frontend redis_exporter_front
    bind *:9121
    mode http
    default_backend redis_exporter_back

backend redis_exporter_back
    mode http
    balance roundrobin
    option httpchk GET /health
    server exporter1 10.0.1.10:9121 check
    server exporter2 10.0.1.11:9121 check
```

### 7.2 HA de Prometheus

**Prometheus en réplication** :
```
┌─────────────────┐    ┌─────────────────┐
│  Prometheus 1   │    │  Prometheus 2   │
│  (replica A)    │    │  (replica B)    │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    │
             ┌──────▼───────┐
             │   Grafana    │
             └──────────────┘
```

**Grafana avec multiple datasources** :
```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus-1
    type: prometheus
    access: proxy
    url: http://prometheus-1:9090
    isDefault: true
    jsonData:
      timeInterval: "15s"

  - name: Prometheus-2
    type: prometheus
    access: proxy
    url: http://prometheus-2:9090
    isDefault: false
    jsonData:
      timeInterval: "15s"
```

### 7.3 Monitoring du monitoring

**Métriques Prometheus** :
```promql
# Prometheus est-il UP ?
up{job="prometheus"}

# Scrape duration
scrape_duration_seconds{job="redis"}

# Samples ingérés
prometheus_tsdb_head_samples_appended_total

# Erreurs de scraping
up{job="redis"} == 0
```

**Métriques Exporter** :
```promql
# Exporter health
up{job="redis"} == 1

# Durée de scraping Redis
redis_exporter_scrape_duration_seconds

# Erreurs de connexion
redis_up{addr=~".*"} == 0
```

**Alertes meta-monitoring** :
```yaml
- alert: PrometheusDown
  expr: up{job="prometheus"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Prometheus instance down"

- alert: RedisExporterDown
  expr: up{job="redis"} == 0
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Redis Exporter {{ $labels.instance }} is down"

- alert: RedisScrapeSlow
  expr: redis_exporter_scrape_duration_seconds > 10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Redis scraping slow ({{ $value }}s)"
```

## 8. Troubleshooting

### 8.1 Problèmes courants

#### Problème 1 : Exporter ne démarre pas

**Symptôme** :
```
Error connecting to Redis: dial tcp: lookup redis-host: no such host
```

**Solutions** :
```bash
# Vérifier la résolution DNS
nslookup redis-host

# Tester la connexion Redis
redis-cli -h redis-host -p 6379 PING

# Vérifier les credentials
redis-cli -h redis-host -p 6379 -a password PING

# Vérifier les ACLs (Redis 6+)
redis-cli -h redis-host -p 6379 --user exporter_user --pass password PING
```

#### Problème 2 : Métriques manquantes

**Diagnostic** :
```bash
# Vérifier l'endpoint de l'exporter
curl http://redis-exporter:9121/metrics

# Vérifier Prometheus targets
curl http://prometheus:9090/api/v1/targets | jq
```

**Causes courantes** :
- Exporter non accessible (firewall, réseau)
- Job mal configuré dans prometheus.yml
- Labels relabel_configs qui filtrent tout

#### Problème 3 : Latence de scraping élevée

**Diagnostic** :
```promql
redis_exporter_scrape_duration_seconds > 5
```

**Causes** :
- Redis instance très chargée (slowlog)
- Connexion réseau lente
- Trop de métriques custom (--check-keys avec patterns larges)

**Solutions** :
```bash
# Réduire les patterns de clés
redis_exporter \
  --redis.addr=redis://localhost:6379 \
  --check-keys='important:*' \  # Au lieu de '*'

# Augmenter le timeout
redis_exporter \
  --redis.addr=redis://localhost:6379 \
  --connection-timeout=30s \
  --scrape-timeout=25s
```

#### Problème 4 : Cardinalité excessive

**Symptôme** :
```
Prometheus TSDB out of memory
Too many time series
```

**Diagnostic** :
```promql
# Top séries par métrique
topk(10, count by (__name__)({__name__=~".+"}))
```

**Solution** :
```bash
# Limiter les labels
# ❌ Éviter
--check-keys='*'  # Toutes les clés → cardinalité explosive

# ✅ Préférer
--check-keys='critical:*'  # Subset contrôlé
--check-single-keys='key1,key2,key3'  # Clés spécifiques
```

### 8.2 Debug mode

```bash
# Activer le debug
redis_exporter \
  --redis.addr=redis://localhost:6379 \
  --debug=true \
  --verbose=true \
  --log-format=json
```

**Output** :
```json
{
  "level": "debug",
  "msg": "Scraping Redis",
  "addr": "redis://localhost:6379",
  "duration_ms": 245
}
```

### 8.3 Health check

```bash
# Endpoint health
curl http://redis-exporter:9121/health

# Si healthy
{"status":"ok"}

# Si unhealthy
{"status":"error","message":"Cannot connect to Redis"}
```

**Intégration Kubernetes** :
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 9121
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: 9121
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

## 9. Best Practices de Production

### 9.1 Checklist de déploiement

- [ ] Exporter par instance Redis (sidecar pattern)
- [ ] Authentification configurée (password/ACL)
- [ ] TLS activé si Redis distant
- [ ] Labels appropriés (env, role, region)
- [ ] Health checks configurés
- [ ] Resource limits définis (CPU, RAM)
- [ ] Logging activé (format JSON)
- [ ] Prometheus scraping fonctionnel
- [ ] Alertes de base configurées
- [ ] Dashboard Grafana créé
- [ ] Documentation des métriques custom

### 9.2 Sécurité

**Pas d'exposition publique** :
```yaml
# ❌ Mauvais
--web.listen-address=0.0.0.0:9121

# ✅ Bon (internal only)
--web.listen-address=127.0.0.1:9121
--web.listen-address=10.0.0.5:9121  # IP interne
```

**ACLs Redis (Redis 6+)** :
```bash
# Créer un utilisateur exporter en read-only
redis-cli ACL SETUSER exporter_user \
  on \
  >secure_password \
  +INFO +PING +CLIENT +SLOWLOG +LATENCY \
  ~* \
  &*

# Tester
redis-cli --user exporter_user --pass secure_password INFO
```

**Secrets management** :
```bash
# ❌ Éviter
--redis.password=plaintext_password

# ✅ Préférer
export REDIS_PASSWORD=$(kubectl get secret redis-password -o jsonpath='{.data.password}' | base64 -d)
redis_exporter --redis.addr=...
```

### 9.3 Resource limits

**Kubernetes** :
```yaml
resources:
  requests:
    memory: "64Mi"    # Minimum
    cpu: "50m"        # 0.05 core
  limits:
    memory: "128Mi"   # Maximum
    cpu: "100m"       # 0.1 core
```

**Consommation typique** :
- **CPU** : 0.02-0.05 core (idle), 0.1-0.2 core (busy)
- **RAM** : 50-80 MB (baseline), jusqu'à 150 MB si beaucoup de métriques

### 9.4 Monitoring en production

**KPIs à suivre** :
```promql
# 1. Scrape success rate
avg_over_time(up{job="redis"}[5m]) * 100

# 2. Scrape duration
histogram_quantile(0.99, rate(redis_exporter_scrape_duration_seconds_bucket[5m]))

# 3. Exporter uptime
time() - process_start_time_seconds{job="redis"}

# 4. Memory usage
process_resident_memory_bytes{job="redis"} / 1024 / 1024
```

## Conclusion

Redis Exporter et Prometheus forment le cœur du monitoring Redis moderne. Une configuration appropriée permet :

1. **Visibilité complète** : 100+ métriques natives + custom
2. **Scalabilité** : Multi-instance, multi-datacenter
3. **Fiabilité** : HA, health checks, alerting
4. **Performance** : Faible overhead, scraping optimisé

**Points clés à retenir** :
- Déployer un exporter par instance Redis (sidecar)
- Scraping interval : 15s en production
- Toujours configurer l'authentification et TLS
- Limiter la cardinalité (labels, patterns de clés)
- Monitorer le monitoring (meta-alertes)

---

**Prochaine section** : 13.4 - Dashboards Grafana pour Redis (configurations avancées)

⏭️ [Dashboards Grafana pour Redis](/13-monitoring-observabilite/04-dashboards-grafana-redis.md)
