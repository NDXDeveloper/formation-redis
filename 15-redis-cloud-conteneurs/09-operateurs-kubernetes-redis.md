🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.9 Opérateurs Kubernetes pour Redis

## 🎯 Objectifs

- Comprendre le pattern Operator et ses avantages
- Maîtriser les principaux Redis Operators
- Déployer Redis Cluster avec Operators
- Déployer Redis Sentinel avec Operators
- Comparer Spotahome, OpsTree, et Redis Enterprise Operators
- Automatiser les opérations day-2 (backup, failover, scaling)
- Monitorer et troubleshooter les operators
- Choisir le bon operator selon le use case

---

## 🤖 Le pattern Operator

### Concept et architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    Kubernetes Operator Pattern                 │
│                                                                │
│  Traditional Approach (StatefulSets):                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Human Operator (SRE/DevOps)                              │ │
│  │       ↓                                                   │ │
│  │  1. Deploy StatefulSet                                    │ │
│  │  2. Monitor manually                                      │ │
│  │  3. Scale manually (kubectl scale)                        │ │
│  │  4. Failover manually (promote replica)                   │ │
│  │  5. Backup manually (scripts)                             │ │
│  │  6. Upgrade manually (rolling update)                     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                │
│  Operator Approach (Automated):                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Custom Resource Definition (CRD)                         │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ apiVersion: redis.io/v1                              │ │ │
│  │  │ kind: RedisCluster                                   │ │ │
│  │  │ metadata:                                            │ │ │
│  │  │   name: my-redis-cluster                             │ │ │
│  │  │ spec:                                                │ │ │
│  │  │   size: 6                                            │ │ │
│  │  │   storage: 100Gi                                     │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │       ↓                                                   │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │  Operator Controller (Software)                      │ │ │
│  │  │                                                      │ │ │
│  │  │  Reconciliation Loop:                                │ │ │
│  │  │  1. Watch CRD changes                                │ │ │
│  │  │  2. Compare desired state vs actual state            │ │ │
│  │  │  3. Take actions to converge                         │ │ │
│  │  │     ├─> Create StatefulSet                           │ │ │
│  │  │     ├─> Create Services                              │ │ │
│  │  │     ├─> Configure replication                        │ │ │
│  │  │     ├─> Monitor health                               │ │ │
│  │  │     ├─> Auto-failover if needed                      │ │ │
│  │  │     ├─> Auto-backup on schedule                      │ │ │
│  │  │     └─> Auto-scale if configured                     │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │       ↓                                                   │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │  Kubernetes Resources (managed)                      │ │ │
│  │  │  ├─> StatefulSets                                    │ │ │
│  │  │  ├─> Services                                        │ │ │
│  │  │  ├─> ConfigMaps                                      │ │ │
│  │  │  ├─> PersistentVolumeClaims                          │ │ │
│  │  │  └─> Pods                                            │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                │
│  Avantages des Operators:                                      │
│  ✅ Infrastructure as Code (declarative)                       │
│  ✅ Automation des opérations récurrentes                      │
│  ✅ Domain-specific knowledge encodé                           │
│  ✅ Self-healing automatique                                   │
│  ✅ Réduction de la charge opérationnelle                      │
│  ✅ Consistance entre environnements                           │
└────────────────────────────────────────────────────────────────┘
```

### Operator Maturity Model

```yaml
Operator Capability Levels (Operator Framework):

Level 1: Basic Install
├── Automated application provisioning
├── Configuration via CRD
└── Example: Deploy Redis Standalone

Level 2: Seamless Upgrades
├── Automated patch and minor version upgrades
├── No downtime for upgrades
└── Example: Rolling update Redis 7.2 → 7.4

Level 3: Full Lifecycle
├── App lifecycle (backup, restore, failover)
├── Automated scaling
└── Example: Auto-backup daily, failover on primary failure

Level 4: Deep Insights
├── Metrics, alerts, log processing
├── Workload analysis
└── Example: Prometheus integration, slowlog alerts

Level 5: Auto Pilot
├── Horizontal/vertical auto-scaling
├── Auto-tuning based on metrics
└── Example: Auto-scale based on memory pressure

Redis Operators Maturity:
├── Redis Enterprise Operator: Level 5
├── Spotahome Redis Operator: Level 3-4
├── OpsTree Redis Operator: Level 3
└── Bitnami (Helm): Level 2
```

---

## 🔧 Spotahome Redis Operator

### Vue d'ensemble

```yaml
Spotahome Redis Operator:
├── Repository: https://github.com/spotahome/redis-operator
├── Stars: ~1.5k
├── Maintainer: Spotahome (active)
├── License: Apache 2.0
├── Supported modes:
│   ├── ✅ Redis Sentinel (HA)
│   ├── ❌ Redis Cluster (not supported)
│   └── ✅ Standalone
├── Features:
│   ├── ✅ Automatic failover (via Sentinel)
│   ├── ✅ Sentinel topology management
│   ├── ✅ Redis exporter integration
│   ├── ✅ PodDisruptionBudget
│   ├── ✅ Affinity rules
│   └── ✅ Custom configurations
├── Maturity: Level 3
└── Best for: Redis Sentinel deployments

Strengths:
✅ Mature and stable (5+ years)
✅ Excellent Sentinel support
✅ Simple CRD
✅ Good documentation
✅ Active community

Weaknesses:
❌ No Redis Cluster support
❌ No backup/restore built-in
❌ Limited auto-scaling
```

### Installation

```bash
# Install via kubectl
kubectl apply -f https://raw.githubusercontent.com/spotahome/redis-operator/master/manifests/databases.spotahome.com_redisfailovers.yaml

kubectl apply -f https://raw.githubusercontent.com/spotahome/redis-operator/master/example/operator/all-redis-operator-resources.yaml

# Verify installation
kubectl get deployment redis-operator -n redis-operator
kubectl logs -n redis-operator deployment/redis-operator
```

### CRD : RedisFailover (Sentinel)

```yaml
# spotahome-redis-sentinel.yaml
apiVersion: databases.spotahome.com/v1
kind: RedisFailover
metadata:
  name: redis-sentinel
  namespace: redis
spec:
  # Sentinel configuration
  sentinel:
    replicas: 3

    image: redis:7.4-alpine
    imagePullPolicy: IfNotPresent

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 400m
        memory: 512Mi

    # Custom Sentinel config
    customConfig:
      - "down-after-milliseconds 5000"
      - "failover-timeout 10000"
      - "parallel-syncs 1"

    # Affinity
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: redis-sentinel
                  app.kubernetes.io/component: sentinel
              topologyKey: kubernetes.io/hostname

    # Security context
    securityContext:
      runAsUser: 999
      runAsNonRoot: true
      fsGroup: 999

  # Redis configuration
  redis:
    replicas: 3  # 1 primary + 2 replicas

    image: redis:7.4-alpine
    imagePullPolicy: IfNotPresent

    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 3Gi

    # Storage
    storage:
      persistentVolumeClaim:
        metadata:
          name: redis-data
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: redis-storage
          resources:
            requests:
              storage: 100Gi

    # Custom Redis config
    customConfig:
      - "maxmemory 2gb"
      - "maxmemory-policy allkeys-lru"
      - "appendonly yes"
      - "appendfsync everysec"
      - "save 900 1"
      - "save 300 10"
      - "save 60 10000"

    # Exporter (for Prometheus)
    exporter:
      enabled: true
      image: oliver006/redis_exporter:v1.58-alpine
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi

    # Affinity
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: redis-sentinel
                  app.kubernetes.io/component: redis
              topologyKey: kubernetes.io/hostname

    # Security context
    securityContext:
      runAsUser: 999
      runAsNonRoot: true
      fsGroup: 999

  # Auth (password)
  auth:
    secretPath: redis-password  # Key in secret
```

### Secret pour Spotahome

```yaml
# spotahome-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-sentinel
  namespace: redis
type: Opaque
stringData:
  redis-password: "changeme-in-production"
```

### Services créés automatiquement

```yaml
# Services created by Spotahome operator:

1. redis-sentinel (Sentinel service)
   ├── Type: ClusterIP
   ├── Port: 26379
   └── Purpose: Access to Sentinels

2. redis-sentinel-rw (Read-Write service)
   ├── Type: ClusterIP
   ├── Port: 6379
   └── Purpose: Access to current primary (writes)

3. redis-sentinel-ro (Read-Only service)
   ├── Type: ClusterIP
   ├── Port: 6379
   └── Purpose: Access to replicas (reads)

# Connect from application:
redis://redis-sentinel-rw.redis.svc.cluster.local:6379  # Writes
redis://redis-sentinel-ro.redis.svc.cluster.local:6379  # Reads
```

---

## 🛠️ OpsTree Redis Operator

### Vue d'ensemble

```yaml
OpsTree Redis Operator:
├── Repository: https://github.com/OT-CONTAINER-KIT/redis-operator
├── Stars: ~700
├── Maintainer: OpsTree Solutions (active)
├── License: Apache 2.0
├── Supported modes:
│   ├── ✅ Redis Standalone
│   ├── ✅ Redis Cluster (6+ nodes)
│   ├── ✅ Redis Sentinel
│   └── ✅ Redis Replication (without Sentinel)
├── Features:
│   ├── ✅ Cluster sharding and replication
│   ├── ✅ Sentinel for HA
│   ├── ✅ Password and TLS support
│   ├── ✅ Redis exporter
│   ├── ✅ Backup to S3/Azure/GCS
│   ├── ✅ Restore from backup
│   ├── ✅ Custom configurations
│   └── ✅ Monitoring integration
├── Maturity: Level 3
└── Best for: Redis Cluster or multi-mode deployments

Strengths:
✅ Supports all Redis modes (Standalone, Cluster, Sentinel)
✅ Best Cluster support
✅ Backup/restore to cloud storage
✅ Good documentation
✅ Active development

Weaknesses:
❌ Less mature than Spotahome
❌ Smaller community
❌ Some bugs in edge cases
```

### Installation

```bash
# Install via Helm
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update

# Install operator
helm install redis-operator ot-helm/redis-operator \
  --namespace redis-operator \
  --create-namespace

# Verify
kubectl get deployment redis-operator -n redis-operator
```

### CRD : Redis Cluster

```yaml
# opstree-redis-cluster.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisCluster
metadata:
  name: redis-cluster
  namespace: redis
spec:
  # Cluster size (must be multiple of 2, minimum 6)
  clusterSize: 6  # 3 primaries + 3 replicas

  # Kubernetes configuration
  kubernetesConfig:
    image: redis:7.4-alpine
    imagePullPolicy: IfNotPresent

    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 3Gi

    # Security context
    securityContext:
      runAsUser: 999
      fsGroup: 999
      runAsNonRoot: true

    # Service type
    redisServiceType: ClusterIP

  # Redis configuration
  redisConfig:
    additionalRedisConfig: |
      maxmemory 2gb
      maxmemory-policy allkeys-lru
      appendonly yes
      appendfsync everysec
      cluster-require-full-coverage no
      cluster-node-timeout 5000

  # Storage
  storage:
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: redis-storage
        resources:
          requests:
            storage: 100Gi

    # Node-local storage (optional, for performance)
    # nodeSelector:
    #   storage-type: ssd

  # Redis Exporter
  redisExporter:
    enabled: true
    image: oliver006/redis_exporter:v1.58-alpine
    imagePullPolicy: IfNotPresent

    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi

    # Prometheus service monitor
    serviceMonitor:
      enabled: true
      namespace: monitoring
      labels:
        prometheus: kube-prometheus
      interval: 30s
      scrapeTimeout: 10s

  # Affinity
  affinity:
    podAntiAffinity: soft  # soft or hard
    topologyKey: kubernetes.io/hostname

  # TLS configuration (optional)
  # TLS:
  #   ca: tls-secret
  #   cert: tls-secret
  #   key: tls-secret
  #   secret:
  #     secretName: redis-tls-cert

  # Priority class
  priorityClassName: redis-priority
```

### CRD : Redis Sentinel

```yaml
# opstree-redis-sentinel.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: Redis
metadata:
  name: redis-sentinel
  namespace: redis
spec:
  # Mode
  mode: sentinel

  # Sentinel configuration
  redisSentinelConfig:
    redisReplicationName: redis-replication
    sentinelSize: 3

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 400m
        memory: 512Mi

    redisConfig:
      additionalRedisConfig: |
        down-after-milliseconds 5000
        failover-timeout 10000
        parallel-syncs 1

  # Kubernetes config
  kubernetesConfig:
    image: redis:7.4-alpine
    imagePullPolicy: IfNotPresent

  # Storage
  storage:
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: redis-storage
        resources:
          requests:
            storage: 100Gi
---
# Redis Replication (monitored by Sentinel)
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisReplication
metadata:
  name: redis-replication
  namespace: redis
spec:
  # Replication size (1 primary + N replicas)
  clusterSize: 3  # 1 primary + 2 replicas

  # Kubernetes configuration
  kubernetesConfig:
    image: redis:7.4-alpine
    imagePullPolicy: IfNotPresent

    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 3Gi

    securityContext:
      runAsUser: 999
      fsGroup: 999

  # Redis config
  redisConfig:
    additionalRedisConfig: |
      maxmemory 2gb
      maxmemory-policy allkeys-lru
      appendonly yes
      appendfsync everysec

  # Storage
  storage:
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: redis-storage
        resources:
          requests:
            storage: 100Gi

  # Redis Exporter
  redisExporter:
    enabled: true
    image: oliver006/redis_exporter:v1.58-alpine
```

### Backup configuration (OpsTree)

```yaml
# opstree-backup.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisBackup
metadata:
  name: redis-cluster-backup
  namespace: redis
spec:
  # Redis cluster to backup
  redisClusterName: redis-cluster

  # Schedule (cron format)
  schedule: "0 2 * * *"  # Daily at 2 AM

  # Storage backend
  storage:
    # S3 backend
    s3:
      bucket: my-redis-backups
      region: us-east-1
      endpoint: https://s3.amazonaws.com

      # Credentials from secret
      accessKeyIdSecret:
        name: aws-credentials
        key: access-key-id
      secretAccessKeySecret:
        name: aws-credentials
        key: secret-access-key

    # Alternative: Azure Blob Storage
    # azureBlob:
    #   container: redis-backups
    #   storageAccount: myaccount
    #   storageAccountKeySecret:
    #     name: azure-credentials
    #     key: storage-account-key

    # Alternative: GCS
    # gcs:
    #   bucket: my-redis-backups
    #   credentialsSecret:
    #     name: gcs-credentials
    #     key: service-account.json

  # Retention policy
  retention:
    keepDaily: 7
    keepWeekly: 4
    keepMonthly: 6
---
# AWS credentials secret
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: redis
type: Opaque
stringData:
  access-key-id: "YOUR_ACCESS_KEY_ID"
  secret-access-key: "YOUR_SECRET_ACCESS_KEY"
```

---

## 🏢 Redis Enterprise Operator

### Vue d'ensemble

```yaml
Redis Enterprise Operator:
├── Repository: https://github.com/RedisLabs/redis-enterprise-k8s-docs
├── Maintainer: Redis Inc. (official)
├── License: Proprietary (commercial)
├── Supported modes:
│   ├── ✅ Redis Enterprise Cluster
│   ├── ✅ Active-Active (CRDT)
│   ├── ✅ Redis Stack modules
│   ├── ✅ Auto-tiering (RAM+Flash)
│   └── ✅ All enterprise features
├── Features:
│   ├── ✅ Active-Active geo-distribution
│   ├── ✅ Redis Stack (Search, JSON, TimeSeries, Bloom)
│   ├── ✅ Auto-scaling
│   ├── ✅ Multi-tenancy
│   ├── ✅ 99.999% SLA
│   ├── ✅ Backup/restore to S3/GCS/Azure
│   ├── ✅ Enterprise support
│   └── ✅ Advanced monitoring
├── Maturity: Level 5
├── Cost: Commercial (contact Redis Inc.)
└── Best for: Enterprise production with budget

Strengths:
✅ Full Redis Enterprise features
✅ Active-Active (unique)
✅ Redis Stack modules
✅ Enterprise support 24/7
✅ Most mature operator
✅ Auto-tiering for cost savings
✅ Multi-tenancy

Weaknesses:
❌ Commercial license required
❌ Expensive ($$$)
❌ Proprietary
❌ Vendor lock-in
```

### Installation

```bash
# Requires Redis Enterprise license
# Contact Redis Inc. for license

# Install operator
kubectl create namespace redis-enterprise

# Apply bundle (includes CRDs, RBAC, Operator)
kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/master/bundle.yaml

# Verify
kubectl get deployment redis-enterprise-operator -n redis-enterprise
```

### CRD : RedisEnterpriseCluster

```yaml
# redis-enterprise-cluster.yaml
apiVersion: app.redislabs.com/v1
kind: RedisEnterpriseCluster
metadata:
  name: rec
  namespace: redis-enterprise
spec:
  # Cluster nodes (3-9 recommended)
  nodes: 3

  # Redis Enterprise version
  redisEnterpriseImageSpec:
    imagePullPolicy: IfNotPresent
    repository: redislabs/redis
    versionTag: 7.4.2-92

  # Redis Enterprise admin credentials
  username: admin@example.com

  # Persistent storage
  persistentSpec:
    enabled: true
    storageClassName: redis-storage
    volumeSize: 100Gi

  # Services
  redisEnterpriseServicesRiggerImageSpec:
    imagePullPolicy: IfNotPresent
    repository: redislabs/k8s-controller
    versionTag: 7.4.2-2

  # Resource requests/limits
  redisEnterpriseNodeResources:
    limits:
      cpu: 4000m
      memory: 16Gi
    requests:
      cpu: 2000m
      memory: 8Gi

  # Services configuration
  servicesRiggerSpec:
    databaseServiceType: ClusterIP
    servicesRiggerImageSpec:
      repository: redislabs/services-manager
      versionTag: 7.4.2-2

  # TLS
  enforceIPv4: false
  createServiceAccount: true

  # License (from secret)
  licenseSecretName: redis-enterprise-license
---
# License secret
apiVersion: v1
kind: Secret
metadata:
  name: redis-enterprise-license
  namespace: redis-enterprise
type: Opaque
stringData:
  license: |
    -----BEGIN LICENSE-----
    YOUR_REDIS_ENTERPRISE_LICENSE_HERE
    -----END LICENSE-----
```

### CRD : RedisEnterpriseDatabase

```yaml
# redis-enterprise-database.yaml
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseDatabase
metadata:
  name: redis-db
  namespace: redis-enterprise
spec:
  # Memory size
  memorySize: 10GB

  # Replication
  replication: true

  # Sharding
  shardCount: 3

  # Persistence
  persistence: aofEverySecond

  # Modules (Redis Stack)
  modules:
    - name: search
      version: 2.8.4
    - name: ReJSON
      version: 2.6.6
    - name: timeseries
      version: 1.10.5
    - name: bf
      version: 2.6.3

  # Eviction policy
  evictionPolicy: allkeys-lru

  # OSS Cluster API (for compatibility)
  ossCluster: false

  # TLS
  tlsMode: enabled

  # Backup
  backup:
    enabled: true
    interval: 24h
    s3:
      awsAccessKeyId: YOUR_ACCESS_KEY
      awsSecretAccessKey: YOUR_SECRET_KEY
      bucketName: redis-backups
      subdir: redis-db
```

---

## 📊 Comparaison des Operators

### Matrice de décision complète

```yaml
┌────────────────────────────────────────────────────────────────────────┐
│                   Redis Operators Comparison Matrix                    │
├────────────────────────────────────────────────────────────────────────┤
│ Feature              │ Spotahome │ OpsTree  │ Redis Ent │ StatefulSet  │
├────────────────────────────────────────────────────────────────────────┤
│ DEPLOYMENT MODES
│ Standalone           │ ✅        │ ✅       │ ✅        │ ✅
│ Replication          │ ✅        │ ✅       │ ✅        │ ✅
│ Sentinel             │ ✅ Best   │ ✅       │ ❌        │ Manual
│ Cluster              │ ❌        │ ✅ Best  │ ✅        │ Manual
│ Active-Active        │ ❌        │ ❌       │ ✅ Unique │ ❌
├────────────────────────────────────────────────────────────────────────┤
│ OPERATIONS
│ Auto-failover        │ ✅        │ ✅       │ ✅        │ ❌
│ Auto-scaling         │ ❌        │ ⚠️ Basic │ ✅        │ ❌
│ Backup/Restore       │ ❌        │ ✅       │ ✅        │ Scripts
│ Rolling updates      │ ✅        │ ✅       │ ✅        │ ✅
│ Config hot-reload    │ ⚠️ Partial│ ✅       │ ✅        │ ❌
├────────────────────────────────────────────────────────────────────────┤
│ FEATURES
│ Redis Stack modules  │ ❌        │ ❌       │ ✅        │ Manual
│ TLS support          │ ⚠️ Basic │ ✅       │ ✅        │ Manual
│ Multi-tenancy        │ ❌        │ ❌       │ ✅        │ ❌
│ Auto-tiering         │ ❌        │ ❌       │ ✅        │ ❌
│ Monitoring built-in  │ ✅        │ ✅       │ ✅        │ Manual
├────────────────────────────────────────────────────────────────────────┤
│ MATURITY
│ Stability            │ ⭐⭐⭐⭐⭐ │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐
│ Community size       │ Large    │ Medium   │ Small     │ N/A
│ Active development   │ ✅        │ ✅       │ ✅        │ N/A
│ Production ready     │ ✅        │ ✅       │ ✅        │ ✅
│ Documentation        │ ⭐⭐⭐⭐   │ ⭐⭐⭐    │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐
├────────────────────────────────────────────────────────────────────────┤
│ COST                                                                   │
│ License              │ Free     │ Free     │ $$$$      │ Free          │
│ Support              │ Community│ Community│ Enterprise│ Community     │
│ Learning curve       │ 2-3 days │ 3-5 days │ 1 week    │ 1 week        │
├────────────────────────────────────────────────────────────────────────┤
│ BEST FOR                                                               │
│ Use case             │ Sentinel │ Cluster  │ Enterprise│ Full control  │
│ Team size            │ 2-5      │ 2-5      │ Any       │ 3+            │
│ Budget               │ Small    │ Small    │ Large     │ Any           │
└────────────────────────────────────────────────────────────────────────┘
```

### Arbre de décision

```
Avez-vous un budget pour Redis Enterprise ($10K+/an) ?
├─ OUI → Redis Enterprise Operator
│   └─> Si besoin Active-Active, Redis Stack, ou support 24/7
│
└─ NON → Quel mode Redis ?
    ├─ Sentinel (HA simple) → Spotahome Redis Operator
    │   └─> Mature, stable, excellent pour Sentinel
    │
    ├─ Cluster (sharding) → OpsTree Redis Operator
    │   └─> Meilleur support Cluster open-source
    │
    ├─ Standalone/Replication → OpsTree ou StatefulSets
    │   ├─> OpsTree si besoin backup/restore automatique
    │   └─> StatefulSets si besoin contrôle total
    │
    └─ Besoin de contrôle total → StatefulSets
        └─> Pas d'abstraction, configuration manuelle
```

---

## 🔄 Opérations Day-2

### Scaling avec Operators

```bash
# Spotahome (Sentinel) - Scale replicas
kubectl patch redisfailover redis-sentinel -n redis \
  --type='merge' \
  -p '{"spec":{"redis":{"replicas":5}}}'

# OpsTree (Cluster) - Scale cluster
kubectl patch rediscluster redis-cluster -n redis \
  --type='merge' \
  -p '{"spec":{"clusterSize":10}}'  # Must be even number

# Redis Enterprise - Scale nodes
kubectl patch redisenterprisecluster rec -n redis-enterprise \
  --type='merge' \
  -p '{"spec":{"nodes":5}}'

# Verify scaling
kubectl get pods -n redis -w
```

### Failover testing

```bash
# Test failover by deleting primary pod

# Spotahome - Find and delete primary
kubectl get pods -n redis -l app.kubernetes.io/component=redis
kubectl delete pod rfr-redis-sentinel-0 -n redis

# Watch Sentinel perform failover
kubectl logs -n redis -l app.kubernetes.io/component=sentinel -f

# Verify new primary
kubectl exec -n redis rfr-redis-sentinel-0 -- \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# OpsTree - Delete cluster primary
kubectl get rediscluster redis-cluster -n redis -o yaml | grep -A5 masterNodes
kubectl delete pod redis-cluster-0 -n redis

# Cluster will automatically elect new primary
kubectl exec -n redis redis-cluster-1 -- redis-cli CLUSTER NODES
```

### Upgrade Redis version

```bash
# Spotahome - Update image
kubectl patch redisfailover redis-sentinel -n redis \
  --type='merge' \
  -p '{"spec":{"redis":{"image":"redis:7.4-alpine"}}}'

# OpsTree - Update image
kubectl patch rediscluster redis-cluster -n redis \
  --type='merge' \
  -p '{"spec":{"kubernetesConfig":{"image":"redis:7.4-alpine"}}}'

# Monitor rolling update
kubectl rollout status statefulset/redis-cluster -n redis

# Verify version
kubectl exec -n redis redis-cluster-0 -- redis-cli INFO server | grep redis_version
```

### Backup and restore (OpsTree)

```bash
# Trigger manual backup
kubectl create -f - <<EOF
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisBackup
metadata:
  name: redis-manual-backup
  namespace: redis
spec:
  redisClusterName: redis-cluster
  storage:
    s3:
      bucket: my-redis-backups
      region: us-east-1
      accessKeyIdSecret:
        name: aws-credentials
        key: access-key-id
      secretAccessKeySecret:
        name: aws-credentials
        key: secret-access-key
EOF

# Check backup status
kubectl get redisbackup redis-manual-backup -n redis -o yaml

# Restore from backup
kubectl create -f - <<EOF
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisRestore
metadata:
  name: redis-restore-20240101
  namespace: redis
spec:
  redisClusterName: redis-cluster
  storage:
    s3:
      bucket: my-redis-backups
      region: us-east-1
      key: backup-20240101-120000.tar.gz
      accessKeyIdSecret:
        name: aws-credentials
        key: access-key-id
      secretAccessKeySecret:
        name: aws-credentials
        key: secret-access-key
EOF
```

---

## 📊 Monitoring des Operators

### ServiceMonitor pour Prometheus

```yaml
# All operators expose metrics via Redis Exporter
# ServiceMonitor is created automatically by operators
# Manual ServiceMonitor example:

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-operator-metrics
  namespace: redis
  labels:
    prometheus: kube-prometheus
spec:
  selector:
    matchLabels:
      app: redis
      # Adjust based on operator
  endpoints:
    - port: redis-exporter
      interval: 30s
      path: /metrics
```

### Grafana Dashboard

```yaml
# Import community dashboards:

Spotahome:
├── Dashboard ID: 11835 (Redis Sentinel)
└── Source: https://grafana.com/grafana/dashboards/11835

OpsTree:
├── Dashboard ID: 14615 (Redis Cluster)
└── Source: https://grafana.com/grafana/dashboards/14615

Redis Enterprise:
├── Built-in dashboards
└── Access via Redis Enterprise UI

Key metrics to monitor:
├── Cluster health (nodes up/down)
├── Primary/replica status
├── Replication lag
├── Memory usage
├── Commands per second
├── Cache hit rate
├── Connected clients
└── Slow log entries
```

---

## 🐛 Troubleshooting

### Diagnostic commands

```bash
# Check operator logs
kubectl logs -n redis-operator deployment/redis-operator -f

# Check CRD status
kubectl get redisfailover -n redis -o yaml  # Spotahome
kubectl get rediscluster -n redis -o yaml   # OpsTree
kubectl get redisenterprisecluster -n redis-enterprise -o yaml  # Redis Ent

# Check events
kubectl get events -n redis --sort-by='.lastTimestamp'

# Check pod logs
kubectl logs -n redis redis-0 -c redis -f

# Describe resources
kubectl describe redisfailover redis-sentinel -n redis

# Check operator-managed resources
kubectl get statefulset,svc,pvc,pods -n redis -l app=redis

# Debug operator reconciliation
kubectl logs -n redis-operator deployment/redis-operator --tail=100 | grep -i error
```

### Common issues

```yaml
Issue: Operator not creating pods
Solution:
├── Check operator logs for errors
├── Verify CRD installed: kubectl get crd
├── Check RBAC permissions: kubectl auth can-i create statefulset --as=system:serviceaccount:redis-operator:redis-operator
└── Verify storage class exists: kubectl get sc

Issue: Pods stuck in Pending
Solution:
├── Check PVC status: kubectl get pvc -n redis
├── Check storage class: kubectl describe sc redis-storage
├── Check node resources: kubectl top nodes
└── Check pod events: kubectl describe pod redis-0 -n redis

Issue: Replication not working
Solution:
├── Check network connectivity between pods
├── Verify redis password in secret
├── Check Redis logs: kubectl logs redis-0 -n redis -c redis
├── Check replication status: kubectl exec redis-0 -n redis -- redis-cli INFO replication
└── Verify NetworkPolicy not blocking traffic

Issue: Sentinel not detecting failure
Solution:
├── Check Sentinel logs: kubectl logs sentinel-0 -n redis
├── Verify down-after-milliseconds setting
├── Check network latency between Sentinel and Redis pods
├── Verify quorum configuration (majority required)
└── Check if Sentinel can reach Redis: kubectl exec sentinel-0 -- redis-cli -p 26379 SENTINEL masters

Issue: Cluster slots not assigned
Solution:
├── Check cluster status: kubectl exec redis-cluster-0 -- redis-cli CLUSTER INFO
├── Verify cluster meet command executed
├── Check cluster nodes: kubectl exec redis-cluster-0 -- redis-cli CLUSTER NODES
└── Manually fix slots if needed: redis-cli --cluster fix <node-ip>:6379

Issue: Operator CRD updates not applied
Solution:
├── Delete and recreate CRD (careful - destructive!)
├── Check operator version compatibility
├── Verify operator has watched namespace
└── Check operator logs for reconciliation errors
```

---

## ✅ Conclusion et recommandations

### Points clés à retenir

1. **Operators = Automation**
   - Encodent les best practices opérationnelles
   - Réduisent la charge de maintenance
   - Self-healing automatique

2. **Choisir le bon operator**
   - Spotahome → Redis Sentinel (mature, stable)
   - OpsTree → Redis Cluster (meilleur support open-source)
   - Redis Enterprise → Features avancées + support (coût élevé)
   - StatefulSets → Contrôle total (plus de travail)

3. **Maturity matters**
   - Spotahome : 5+ ans, très stable
   - OpsTree : 2+ ans, croissance rapide
   - Redis Enterprise : Production-grade, support 24/7

4. **Features vs Complexity**
   - Plus de features = plus de complexité
   - Évaluer les besoins réels vs "nice to have"
   - Commencer simple, ajouter features au besoin

5. **Day-2 Operations**
   - Backup/restore critiques en production
   - Monitoring et alerting essentiels
   - Tester failover régulièrement

### Recommandations par use case

```yaml
Startup (<10 services):
├── Budget: Limité
├── Équipe: 1-2 personnes
└── Solution: Spotahome (Sentinel) ou StatefulSets
    Rationale: Simple, gratuit, suffisant

PME (10-50 services):
├── Budget: Moyen ($1K-5K/mois infra)
├── Équipe: 2-5 personnes
└── Solution: OpsTree (Cluster) ou Spotahome (Sentinel)
    Rationale: Features avancées, support communauté

Entreprise (>50 services):
├── Budget: Important (>$10K/mois infra)
├── Équipe: 5+ personnes
├── SLA: 99.99%+
└── Solution: Redis Enterprise Operator
    Rationale: Support 24/7, SLA garantie, features enterprise

Use cases spécifiques:
├── E-commerce global → Redis Enterprise (Active-Active)
├── Analytics → OpsTree Cluster (sharding)
├── Session store → Spotahome Sentinel (HA simple)
├── Cache distribué → OpsTree Cluster ou Sentinel
└── RAG/Vector search → Redis Enterprise (Redis Stack)
```

### Checklist de décision finale

```yaml
☐ Évaluer le mode Redis nécessaire (Standalone/Sentinel/Cluster)

☐ Définir les exigences de disponibilité (SLA)
   └─> 99.9%: Sentinel suffisant
   └─> 99.99%+: Considérer Redis Enterprise

☐ Évaluer le budget
   └─> <$1K/mois: Open-source (Spotahome/OpsTree)
   └─> >$10K/mois: Redis Enterprise envisageable

☐ Évaluer la taille de l'équipe
   └─> 1-2 personnes: Spotahome (simple)
   └─> 3-5 personnes: OpsTree (features)
   └─> 5+ personnes: Redis Enterprise (complexité ok)

☐ Évaluer les features nécessaires
   └─> Redis Stack modules: Redis Enterprise only
   └─> Active-Active: Redis Enterprise only
   └─> Cluster: OpsTree best open-source

☐ Tester l'operator en staging (2-4 semaines)

☐ Valider backup/restore procedures

☐ Configurer monitoring et alerting

☐ Documenter runbooks pour opérations courantes

☐ Former l'équipe sur l'operator choisi
```

### Migration path

```yaml
StatefulSets → Operator:

Étape 1: Déployer operator en parallèle
├── Install operator
├── Deploy new cluster avec operator
├── Sync data (Redis replication)
└── Duration: 1 jour

Étape 2: Basculer traffic progressivement
├── 10% traffic → operator cluster
├── 50% traffic
├── 100% traffic
└── Duration: 1 semaine

Étape 3: Decommission StatefulSets
├── Verify all traffic on operator cluster
├── Backup StatefulSets data
├── Delete StatefulSets
└── Duration: 1 jour

Total: 2 semaines avec validation complète
```

---

**🎯 Fin du module 15 !** Nous avons couvert l'intégralité des solutions Redis dans le cloud (AWS, Azure, GCP, Redis Enterprise) et les déploiements conteneurisés (Docker, Kubernetes avec StatefulSets et Operators). Vous disposez maintenant de tous les outils pour déployer et gérer Redis en production dans n'importe quel environnement cloud ou conteneurisé.

⏭️ [Helm charts et déploiement automatisé](/15-redis-cloud-conteneurs/10-helm-charts-deploiement-automatise.md)
