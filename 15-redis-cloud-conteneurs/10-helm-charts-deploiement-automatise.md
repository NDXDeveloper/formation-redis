🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.10 Helm charts et déploiement automatisé

## 🎯 Objectifs

- Comprendre Helm et son rôle pour le déploiement Redis
- Maîtriser le Bitnami Redis Helm chart (le plus populaire)
- Déployer Redis en différentes topologies avec Helm
- Configurer les values.yaml pour production
- Intégrer Helm dans les pipelines CI/CD
- Implémenter GitOps avec ArgoCD et Flux
- Gérer les secrets de manière sécurisée
- Comparer Helm vs Operators vs manifests bruts
- Automatiser les déploiements multi-environnements

---

## 📦 Introduction à Helm

### Concept et architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Helm: Package Manager for Kubernetes         │
│                                                                 │
│  Without Helm (manual manifests):                               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  redis-namespace.yaml                                      │ │
│  │  redis-configmap.yaml                                      │ │
│  │  redis-secret.yaml                                         │ │
│  │  redis-storageclass.yaml                                   │ │
│  │  redis-service-headless.yaml                               │ │
│  │  redis-service-primary.yaml                                │ │
│  │  redis-statefulset.yaml                                    │ │
│  │  redis-pdb.yaml                                            │ │
│  │  redis-networkpolicy.yaml                                  │ │
│  │  redis-servicemonitor.yaml                                 │ │
│  │                                                            │ │
│  │  10+ files × 3 environments = 30+ files to manage          │ │
│  │  Copy/paste errors, inconsistencies, hard to maintain      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  With Helm (templated package):                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Chart.yaml (metadata)                                     │ │
│  │  values.yaml (default configuration)                       │ │
│  │  templates/ (Kubernetes manifest templates)                │ │
│  │    ├─ statefulset.yaml                                     │ │
│  │    ├─ service.yaml                                         │ │
│  │    ├─ configmap.yaml                                       │ │
│  │    └─ ... (all resources templated)                        │ │
│  │                                                            │ │
│  │  values-dev.yaml (dev overrides)                           │ │
│  │  values-staging.yaml (staging overrides)                   │ │
│  │  values-production.yaml (production overrides)             │ │
│  │                                                            │ │
│  │  Deploy: helm install redis ./redis-chart -f values-prod.yaml│
│  │  1 command = entire stack deployed correctly               │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Helm Benefits:                                                 │
│  ✅ Packaging: Bundle related K8s resources                     │
│  ✅ Templating: Reuse configurations across environments        │
│  ✅ Versioning: Track chart versions + app versions             │
│  ✅ Rollback: Easy rollback to previous release                 │
│  ✅ Dependencies: Manage chart dependencies                     │
│  ✅ Hooks: Execute actions at specific lifecycle stages         │
│  ✅ Distribution: Share charts via repositories                 │
│                                                                 │
│  Helm Architecture:                                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Helm CLI (client)                                         │ │
│  │       ↓                                                    │ │
│  │  Chart + values.yaml                                       │ │
│  │       ↓                                                    │ │
│  │  Template Engine (Go templates)                            │ │
│  │       ↓                                                    │ │
│  │  Rendered Kubernetes manifests                             │ │
│  │       ↓                                                    │ │
│  │  kubectl apply (via Kubernetes API)                        │ │
│  │       ↓                                                    │ │
│  │  Kubernetes Cluster                                        │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Helm vs alternatives

```yaml
┌───────────────────────────────────────────────────────────────┐
│        Helm vs Alternatives for Redis Deployment              │
├───────────────────────────────────────────────────────────────┤
│ Method        │ Complexity │ Maintenance │ Features │ Best For│
├───────────────────────────────────────────────────────────────┤
│ Raw manifests │ Medium     │ High        │ Full     │ Learning│
│ Kustomize     │ Low        │ Medium      │ Limited  │ Simple  │
│ Helm          │ Low        │ Low         │ Good     │ General │
│ Operators     │ High       │ Very Low    │ Best     │ Complex │
└───────────────────────────────────────────────────────────────┘

Use Helm when:
✅ Need to deploy Redis across multiple environments
✅ Want standardized, tested configurations
✅ Team not ready for Operators complexity
✅ Need quick deployment (< 5 minutes)
✅ Want community-maintained charts

Use Operators when:
✅ Need Redis Cluster or Sentinel
✅ Want auto-failover and self-healing
✅ Need day-2 operations automation
✅ Have complex requirements

Use raw manifests when:
✅ Learning Kubernetes
✅ Need full control over every detail
✅ Very specific/custom requirements
```

---

## 📚 Helm charts populaires pour Redis

### Vue d'ensemble

```yaml
Popular Redis Helm Charts:

1. Bitnami Redis
   ├── Repository: https://charts.bitnami.com/bitnami
   ├── Chart: redis
   ├── Downloads: ~50M+ (most popular!)
   ├── Maintainer: VMware (Bitnami)
   ├── Supported modes:
   │   ├── ✅ Standalone
   │   ├── ✅ Primary-Replica
   │   └── ✅ Sentinel
   ├── Features:
   │   ├── ✅ Multiple architectures
   │   ├── ✅ Metrics (Prometheus)
   │   ├── ✅ NetworkPolicy
   │   ├── ✅ PodSecurityPolicy
   │   ├── ✅ TLS support
   │   └── ✅ Well documented
   └── Best for: General use, production

2. Bitnami Redis Cluster
   ├── Repository: https://charts.bitnami.com/bitnami
   ├── Chart: redis-cluster
   ├── Downloads: ~5M+
   ├── Supported modes:
   │   └── ✅ Redis Cluster (sharding)
   └── Best for: Cluster mode with sharding

3. Spotahome Redis Operator (Helm)
   ├── Repository: https://spotahome.github.io/redis-operator
   ├── Chart: redis-operator
   ├── Includes operator + CRDs
   └── Best for: Sentinel with operator

4. OpsTree Redis Operator (Helm)
   ├── Repository: https://ot-container-kit.github.io/helm-charts/
   ├── Chart: redis-operator
   ├── Includes operator + CRDs
   └── Best for: Cluster with operator

Recommendation:
├── Start with: Bitnami Redis (most popular, well-tested)
├── Need Cluster: Bitnami Redis Cluster
└── Advanced: Operators (covered in 15.9)
```

---

## 🚀 Bitnami Redis Helm Chart

### Installation de base

```bash
# Add Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for Redis chart
helm search repo bitnami/redis

# Show chart info
helm show chart bitnami/redis
helm show values bitnami/redis > values.yaml

# Install Redis (development)
helm install redis bitnami/redis \
  --namespace redis \
  --create-namespace

# Get admin password
export REDIS_PASSWORD=$(kubectl get secret --namespace redis redis -o jsonpath="{.data.redis-password}" | base64 -d)

# Connect to Redis
kubectl run --namespace redis redis-client --rm --tty -i --restart='Never' \
  --env REDIS_PASSWORD=$REDIS_PASSWORD \
  --image docker.io/bitnami/redis:7.4-debian-12 -- bash

redis-cli -h redis-master -a $REDIS_PASSWORD
```

### Architecture Standalone (simple)

```yaml
# values-standalone.yaml
# Simple Redis standalone for development

global:
  redis:
    password: "changeme"

# Architecture: standalone (single instance)
architecture: standalone

# Image configuration
image:
  registry: docker.io
  repository: bitnami/redis
  tag: 7.4-debian-12
  pullPolicy: IfNotPresent

# Authentication
auth:
  enabled: true
  password: "changeme"  # Change in production!
  # existingSecret: "redis-secret"
  # existingSecretPasswordKey: "redis-password"

# Master (standalone) configuration
master:
  # Replica count (1 for standalone)
  count: 1

  # Resources
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

  # Persistence
  persistence:
    enabled: true
    storageClass: ""  # Use default
    size: 10Gi

  # Service
  service:
    type: ClusterIP
    port: 6379

# Redis configuration
commonConfiguration: |-
  # Enable AOF persistence
  appendonly yes
  appendfsync everysec

  # Memory policy
  maxmemory-policy allkeys-lru

  # Disable dangerous commands
  rename-command FLUSHDB ""
  rename-command FLUSHALL ""

# Metrics (Prometheus)
metrics:
  enabled: true

  image:
    registry: docker.io
    repository: bitnami/redis-exporter
    tag: 1.58-debian-12

  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 100m
      memory: 128Mi

  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: 30s
```

### Architecture Primary-Replica (HA)

```yaml
# values-primary-replica.yaml
# Redis with replication for production

global:
  storageClass: "gp3"  # AWS EBS gp3

# Architecture: replication (1 primary + N replicas)
architecture: replication

# Image configuration
image:
  registry: docker.io
  repository: bitnami/redis
  tag: 7.4-debian-12
  pullPolicy: IfNotPresent

# Authentication
auth:
  enabled: true
  existingSecret: "redis-secret"
  existingSecretPasswordKey: "redis-password"

# Master/Primary configuration
master:
  count: 1

  # Resources (production-sized)
  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  # Persistence
  persistence:
    enabled: true
    storageClass: "gp3"
    size: 100Gi

  # Pod anti-affinity (spread across nodes)
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: redis
                app.kubernetes.io/component: master
            topologyKey: kubernetes.io/hostname

  # Service
  service:
    type: ClusterIP
    port: 6379

# Replica configuration
replica:
  # Number of replicas
  replicaCount: 2

  # Resources (same as master)
  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  # Persistence
  persistence:
    enabled: true
    storageClass: "gp3"
    size: 100Gi

  # Pod anti-affinity
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: redis
                app.kubernetes.io/component: replica
            topologyKey: kubernetes.io/hostname

  # Service (read-only)
  service:
    type: ClusterIP
    port: 6379

# Redis configuration (both primary and replicas)
commonConfiguration: |-
  # Snapshotting
  save 900 1
  save 300 10
  save 60 10000

  # AOF persistence
  appendonly yes
  appendfsync everysec
  auto-aof-rewrite-percentage 100
  auto-aof-rewrite-min-size 64mb

  # Memory
  maxmemory-policy allkeys-lru

  # Slow log
  slowlog-log-slower-than 10000
  slowlog-max-len 128

  # Disable dangerous commands
  rename-command FLUSHDB ""
  rename-command FLUSHALL ""
  rename-command CONFIG "CONFIG_ADMIN"

# Security
securityContext:
  enabled: true
  fsGroup: 1001
  runAsUser: 1001
  runAsNonRoot: true

containerSecurityContext:
  enabled: true
  runAsUser: 1001
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL

# Pod Disruption Budget
pdb:
  create: true
  minAvailable: 1

# Network Policy
networkPolicy:
  enabled: true
  allowExternal: true
  ingressNSMatchLabels:
    name: app
  ingressNSPodMatchLabels:
    app: backend

# Metrics
metrics:
  enabled: true

  image:
    registry: docker.io
    repository: bitnami/redis-exporter
    tag: 1.58-debian-12

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi

  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: 30s
    scrapeTimeout: 10s
    labels:
      prometheus: kube-prometheus

  prometheusRule:
    enabled: true
    namespace: monitoring
    labels:
      prometheus: kube-prometheus
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis instance is down"
          description: "Redis instance {{ $labels.instance }} has been down for more than 1 minute."

      - alert: RedisHighMemoryUsage
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage is high"
          description: "Redis instance {{ $labels.instance }} memory usage is above 90%."

      - alert: RedisTooManyConnections
        expr: redis_connected_clients > 5000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Too many Redis connections"
          description: "Redis instance {{ $labels.instance }} has {{ $value }} connections."

# Volume Permissions (if needed for non-root)
volumePermissions:
  enabled: false
```

### Architecture Sentinel (HA avec failover automatique)

```yaml
# values-sentinel.yaml
# Redis Sentinel for automatic failover

global:
  storageClass: "gp3"

# Architecture: replication with Sentinel
architecture: replication

# Image
image:
  registry: docker.io
  repository: bitnami/redis
  tag: 7.4-debian-12

# Auth
auth:
  enabled: true
  existingSecret: "redis-secret"
  existingSecretPasswordKey: "redis-password"

# Sentinel configuration
sentinel:
  enabled: true

  # Number of Sentinel nodes (3 or 5 recommended)
  quorum: 2

  # Sentinel image
  image:
    registry: docker.io
    repository: bitnami/redis-sentinel
    tag: 7.4-debian-12

  # Master set name
  masterSet: mymaster

  # Resources
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 400m
      memory: 512Mi

  # Persistence for Sentinel
  persistence:
    enabled: false  # Sentinel doesn't need persistence

  # Configuration
  configuration: |-
    sentinel down-after-milliseconds mymaster 5000
    sentinel failover-timeout mymaster 10000
    sentinel parallel-syncs mymaster 1

  # Service
  service:
    type: ClusterIP
    ports:
      sentinel: 26379

# Master configuration
master:
  count: 1

  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  persistence:
    enabled: true
    storageClass: "gp3"
    size: 100Gi

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: redis
          topologyKey: kubernetes.io/hostname

# Replica configuration
replica:
  replicaCount: 2

  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  persistence:
    enabled: true
    storageClass: "gp3"
    size: 100Gi

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: redis
          topologyKey: kubernetes.io/hostname

# Redis configuration
commonConfiguration: |-
  save 900 1
  save 300 10
  save 60 10000
  appendonly yes
  appendfsync everysec
  maxmemory-policy allkeys-lru
  slowlog-log-slower-than 10000
  rename-command FLUSHDB ""
  rename-command FLUSHALL ""

# Security
securityContext:
  enabled: true
  fsGroup: 1001
  runAsUser: 1001

containerSecurityContext:
  enabled: true
  runAsUser: 1001
  runAsNonRoot: true
  allowPrivilegeEscalation: false

# PDB
pdb:
  create: true
  minAvailable: 2  # At least 2 pods (1 master + 1 replica)

# Metrics
metrics:
  enabled: true

  sentinel:
    enabled: true

  serviceMonitor:
    enabled: true
    namespace: monitoring
```

---

## 🔧 Configuration avancée

### TLS/SSL configuration

```yaml
# values-tls.yaml
# Redis with TLS encryption

# ... (base configuration) ...

# TLS configuration
tls:
  enabled: true

  # Certificate authority
  authClients: true

  # Certificate files (from existing secret)
  certificatesSecret: "redis-tls-certs"
  certFilename: "tls.crt"
  certKeyFilename: "tls.key"
  certCAFilename: "ca.crt"

  # DH params
  dhParamsFilename: ""

# Metrics with TLS
metrics:
  enabled: true

  # Redis Exporter needs to connect via TLS
  extraArgs:
    skip-tls-verification: "false"
---
# Create TLS secret
# apiVersion: v1
# kind: Secret
# metadata:
#   name: redis-tls-certs
#   namespace: redis
# type: Opaque
# data:
#   tls.crt: <base64-encoded-cert>
#   tls.key: <base64-encoded-key>
#   ca.crt: <base64-encoded-ca>
```

### External access (LoadBalancer)

```yaml
# values-external.yaml
# Redis accessible from outside cluster

# ... (base configuration) ...

# Master service with LoadBalancer
master:
  service:
    type: LoadBalancer
    port: 6379

    # Annotations for cloud provider
    annotations:
      # AWS
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

      # GCP
      # cloud.google.com/load-balancer-type: "Internal"

      # Azure
      # service.beta.kubernetes.io/azure-load-balancer-internal: "true"

    # Load balancer source ranges (whitelist IPs)
    loadBalancerSourceRanges:
      - "10.0.0.0/8"
      - "172.16.0.0/12"
      - "192.168.0.0/16"

# Replica service (read-only external)
replica:
  service:
    type: LoadBalancer
    port: 6379
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

### Backup configuration avec sidecars

```yaml
# values-backup.yaml
# Redis with automated backup sidecar

# ... (base configuration) ...

master:
  # Sidecar for backup
  sidecars:
    - name: backup
      image: amazon/aws-cli:latest
      command:
        - /bin/sh
        - -c
        - |
          while true; do
            echo "Starting backup at $(date)"

            # Trigger BGSAVE
            redis-cli -h localhost -a $REDIS_PASSWORD --no-auth-warning BGSAVE

            # Wait for BGSAVE to complete
            sleep 30

            # Upload to S3
            aws s3 cp /data/dump.rdb s3://my-bucket/redis-backups/dump-$(date +%Y%m%d-%H%M%S).rdb

            # Sleep 24 hours
            sleep 86400
          done
      env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: redis-password
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: secret-access-key
        - name: AWS_DEFAULT_REGION
          value: "us-east-1"
      volumeMounts:
        - name: redis-data
          mountPath: /data
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 200m
          memory: 256Mi
```

---

## 🔐 Gestion des secrets

### Création du secret Redis

```bash
#!/bin/bash
# create-redis-secret.sh

# Generate secure password
REDIS_PASSWORD=$(openssl rand -base64 32)

# Create Kubernetes secret
kubectl create secret generic redis-secret \
  --namespace redis \
  --from-literal=redis-password="${REDIS_PASSWORD}"

# Save password securely (use vault in production!)
echo "Redis password: ${REDIS_PASSWORD}" > redis-password.txt
chmod 600 redis-password.txt

echo "Secret created: redis-secret"
echo "Password saved to: redis-password.txt (keep secure!)"
```

### Sealed Secrets (pour GitOps)

```yaml
# sealed-secret.yaml
# Encrypted secret safe for Git

apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: redis-secret
  namespace: redis
spec:
  encryptedData:
    redis-password: AgBhY3R1YWxseS1lbmNyeXB0ZWQtZGF0YS1oZXJlLi4u
  template:
    metadata:
      name: redis-secret
      namespace: redis
    type: Opaque

# To create:
# 1. Install kubeseal CLI
# 2. Create normal secret
# 3. Encrypt: kubectl create secret generic redis-secret \
#      --dry-run=client -o yaml \
#      --from-literal=redis-password=mypassword \
#      | kubeseal -o yaml > sealed-secret.yaml
# 4. Commit to Git (safe!)
```

### External Secrets Operator

```yaml
# external-secret.yaml
# Fetch secret from AWS Secrets Manager, Azure Key Vault, etc.

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: redis-secret
  namespace: redis
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore

  target:
    name: redis-secret
    creationPolicy: Owner

  data:
    - secretKey: redis-password
      remoteRef:
        key: production/redis/password
---
# SecretStore configuration
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: redis
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

---

## 🔄 CI/CD et GitOps

### Pipeline GitLab CI

```yaml
# .gitlab-ci.yml
# GitLab CI pipeline for Redis deployment

stages:
  - validate
  - deploy-dev
  - deploy-staging
  - deploy-production

variables:
  HELM_VERSION: "3.14.0"
  CHART_VERSION: "18.6.1"

# Validate Helm chart
validate:
  stage: validate
  image: alpine/helm:${HELM_VERSION}
  script:
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
    - helm lint -f values-dev.yaml
    - helm template redis bitnami/redis -f values-dev.yaml --validate
  only:
    - merge_requests
    - main

# Deploy to dev
deploy:dev:
  stage: deploy-dev
  image: alpine/helm:${HELM_VERSION}
  script:
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
    - |
      helm upgrade --install redis bitnami/redis \
        --version ${CHART_VERSION} \
        --namespace redis-dev \
        --create-namespace \
        -f values-dev.yaml \
        --wait \
        --timeout 10m
  environment:
    name: dev
    kubernetes:
      namespace: redis-dev
  only:
    - main

# Deploy to staging (manual)
deploy:staging:
  stage: deploy-staging
  image: alpine/helm:${HELM_VERSION}
  script:
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
    - |
      helm upgrade --install redis bitnami/redis \
        --version ${CHART_VERSION} \
        --namespace redis-staging \
        --create-namespace \
        -f values-staging.yaml \
        --wait \
        --timeout 10m
  environment:
    name: staging
    kubernetes:
      namespace: redis-staging
  when: manual
  only:
    - main

# Deploy to production (manual + approval)
deploy:production:
  stage: deploy-production
  image: alpine/helm:${HELM_VERSION}
  script:
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
    - |
      helm upgrade --install redis bitnami/redis \
        --version ${CHART_VERSION} \
        --namespace redis \
        --create-namespace \
        -f values-production.yaml \
        --wait \
        --timeout 15m \
        --atomic \
        --cleanup-on-fail
  environment:
    name: production
    kubernetes:
      namespace: redis
  when: manual
  only:
    - main
```

### ArgoCD Application

```yaml
# argocd-application.yaml
# GitOps deployment with ArgoCD

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-production
  namespace: argocd

  # Finalizer for cascade deletion
  finalizers:
    - resources-finalizer.argocd.argoproj.io

spec:
  # Project
  project: default

  # Source (Git repository)
  source:
    repoURL: https://github.com/myorg/infrastructure
    targetRevision: main
    path: helm/redis

    # Helm configuration
    helm:
      # Release name
      releaseName: redis

      # Values files (in order of precedence)
      valueFiles:
        - values.yaml
        - values-production.yaml

      # Additional parameters (override values)
      parameters:
        - name: master.resources.limits.memory
          value: "4Gi"
        - name: replica.replicaCount
          value: "3"

  # Destination cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: redis

  # Sync policy
  syncPolicy:
    # Automated sync
    automated:
      # Auto-create namespace
      prune: true
      selfHeal: true
      allowEmpty: false

    # Sync options
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - RespectIgnoreDifferences=true

    # Retry strategy
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Ignore differences (prevent drift detection on these fields)
  ignoreDifferences:
    - group: apps
      kind: StatefulSet
      jsonPointers:
        - /spec/replicas  # Ignore if autoscaling enabled
        - /spec/template/spec/containers/0/image  # Ignore image tag updates
```

### Flux CD HelmRelease

```yaml
# flux-helmrelease.yaml
# GitOps deployment with Flux CD

apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: redis
  namespace: redis
spec:
  # Release interval
  interval: 10m

  # Timeout
  timeout: 15m

  # Chart
  chart:
    spec:
      chart: redis
      version: "18.6.1"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 1h

  # Values
  valuesFrom:
    - kind: ConfigMap
      name: redis-values-common
    - kind: Secret
      name: redis-values-secret
      valuesKey: values.yaml

  values:
    # Inline values (lower precedence)
    architecture: replication

    master:
      count: 1
      persistence:
        enabled: true
        size: 100Gi

    replica:
      replicaCount: 2
      persistence:
        enabled: true
        size: 100Gi

  # Install/Upgrade configuration
  install:
    createNamespace: true
    remediation:
      retries: 3

  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true

  # Rollback configuration
  rollback:
    recreate: true
    cleanupOnFail: true

  # Post-install/upgrade tests
  test:
    enable: true

  # Drift detection
  driftDetection:
    mode: enabled
    ignore:
      - paths: ["/spec/replicas"]
        target:
          kind: StatefulSet
---
# HelmRepository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
```

---

## 🛠️ Commandes Helm essentielles

### Installation et gestion

```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search charts
helm search repo redis
helm search repo bitnami/redis --versions

# Show chart info
helm show chart bitnami/redis
helm show values bitnami/redis
helm show readme bitnami/redis

# Install with custom values
helm install redis bitnami/redis \
  --namespace redis \
  --create-namespace \
  -f values-production.yaml \
  --wait \
  --timeout 10m

# Install with inline values
helm install redis bitnami/redis \
  --namespace redis \
  --set auth.password=mypassword \
  --set master.persistence.size=100Gi \
  --set replica.replicaCount=3

# Dry run (simulate)
helm install redis bitnami/redis \
  -f values-production.yaml \
  --dry-run \
  --debug

# Template (render manifests without installing)
helm template redis bitnami/redis \
  -f values-production.yaml \
  --output-dir ./rendered-manifests

# Upgrade
helm upgrade redis bitnami/redis \
  -f values-production.yaml \
  --wait \
  --timeout 10m \
  --atomic  # Rollback on failure

# Upgrade with new values
helm upgrade redis bitnami/redis \
  --reuse-values \
  --set replica.replicaCount=5

# List releases
helm list -n redis
helm list --all-namespaces

# Get release info
helm get values redis -n redis
helm get manifest redis -n redis
helm get notes redis -n redis
helm get all redis -n redis

# History
helm history redis -n redis

# Rollback
helm rollback redis 1 -n redis  # Rollback to revision 1
helm rollback redis -n redis    # Rollback to previous revision

# Uninstall
helm uninstall redis -n redis

# Uninstall and keep history
helm uninstall redis -n redis --keep-history

# Test
helm test redis -n redis
```

### Debugging

```bash
# Debug installation
helm install redis bitnami/redis \
  -f values.yaml \
  --debug \
  --dry-run

# Check rendered values
helm get values redis -n redis
helm get values redis -n redis --all

# Check manifests
helm get manifest redis -n redis | less

# Diff before upgrade (requires helm-diff plugin)
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade redis bitnami/redis -f values-production-new.yaml

# Lint chart
helm lint -f values.yaml

# Validate rendered manifests
helm template redis bitnami/redis -f values.yaml | kubectl apply --dry-run=client -f -
```

---

## 📊 Monitoring et observabilité

### Grafana Dashboard provisioning

```yaml
# grafana-dashboard-configmap.yaml
# Auto-provision Redis dashboard in Grafana

apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  redis-dashboard.json: |
    {
      "dashboard": {
        "title": "Redis (Helm)",
        "tags": ["redis", "helm"],
        "timezone": "browser",
        "panels": [
          {
            "title": "Connected Clients",
            "targets": [
              {
                "expr": "redis_connected_clients{namespace=\"redis\"}",
                "legendFormat": "{{ pod }}"
              }
            ],
            "type": "graph"
          },
          {
            "title": "Memory Usage",
            "targets": [
              {
                "expr": "redis_memory_used_bytes{namespace=\"redis\"} / redis_memory_max_bytes{namespace=\"redis\"} * 100",
                "legendFormat": "{{ pod }}"
              }
            ],
            "type": "gauge"
          },
          {
            "title": "Commands Per Second",
            "targets": [
              {
                "expr": "rate(redis_commands_processed_total{namespace=\"redis\"}[1m])",
                "legendFormat": "{{ pod }}"
              }
            ],
            "type": "graph"
          }
        ]
      }
    }
```

### Alerting avec PrometheusRule

```yaml
# Created automatically by Bitnami chart when:
# metrics.prometheusRule.enabled: true

# Or create custom PrometheusRule:
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: redis-custom-alerts
  namespace: redis
  labels:
    prometheus: kube-prometheus
spec:
  groups:
    - name: redis.custom
      interval: 30s
      rules:
        - alert: RedisSlowCommands
          expr: redis_slowlog_length > 50
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Redis has many slow commands"
            description: "Redis {{ $labels.pod }} has {{ $value }} commands in slow log."

        - alert: RedisEvictedKeys
          expr: increase(redis_evicted_keys_total[5m]) > 100
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Redis is evicting keys"
            description: "Redis {{ $labels.pod }} evicted {{ $value }} keys in last 5 minutes."
```

---

## 🔍 Comparaison finale

### Helm vs Operators vs Raw manifests

```yaml
┌────────────────────────────────────────────────────────────────┐
│         Deployment Method Comparison for Redis                 │
├────────────────────────────────────────────────────────────────┤
│ Aspect          │ Raw       │ Helm      │ Operators │ Managed  │
│                 │ Manifests │           │           │ Service  │
├────────────────────────────────────────────────────────────────┤
│ Setup time      │ 2-4h      │ 5-10min   │ 30-60min  │ 10min    │
│ Maintenance     │ High      │ Low       │ Very Low  │ None     │
│ Flexibility     │ ⭐⭐⭐⭐⭐│ ⭐⭐⭐⭐ │ ⭐⭐⭐    │ ⭐⭐     │
│ Automation      │ ❌        │ ⚠️ Basic  │ ✅        │ ✅       │
│ Multi-env       │ Complex   │ Easy      │ Easy      │ N/A      │
│ Learning curve  │ 1 week    │ 1-2 days  │ 3-5 days  │ 1 hour   │
│ Cost            │ $0        │ $0        │ $0        │ $$$      │
│ Community       │ Large     │ Large     │ Medium    │ Vendor   │
│ Upgrades        │ Manual    │ helm up.  │ Automated │ Managed  │
│ Failover        │ Manual    │ Manual    │ Auto      │ Auto     │
│ Backup          │ Scripts   │ Scripts   │ Built-in  │ Built-in │
│ Cluster mode    │ Complex   │ Chart     │ Native    │ Native   │
│ Best for        │ Learning  │ Most      │ Complex   │ Simplest │
└────────────────────────────────────────────────────────────────┘

Decision Matrix:

Choose Raw Manifests if:
├── Learning Kubernetes
├── Need absolute control
├── Very specific/unique requirements
└── Have time for manual operations

Choose Helm if:
├── Multi-environment deployments
├── Standard Redis use cases
├── Want quick setup
├── Team comfortable with K8s basics
└── Don't need advanced automation

Choose Operators if:
├── Need Redis Cluster or Sentinel
├── Want automatic failover
├── Need day-2 operations automation
├── Have complex requirements
└── Team has K8s expertise

Choose Managed Service if:
├── Want zero operational overhead
├── Budget allows ($$$)
├── Don't need K8s deployment
└── Prefer vendor support
```

---

## ✅ Best practices production

### Checklist complète

```yaml
✅ Chart Configuration:
├── ✓ Use specific chart version (not "latest")
├── ✓ Pin image tags (not "latest")
├── ✓ Enable authentication (auth.enabled: true)
├── ✓ Use existingSecret for passwords
├── ✓ Enable persistence (both master and replicas)
├── ✓ Set appropriate resource limits
├── ✓ Configure PodDisruptionBudget
└── ✓ Enable NetworkPolicy

✅ High Availability:
├── ✓ Use replication architecture (1 master + 2+ replicas)
├── ✓ Enable Sentinel for auto-failover (if needed)
├── ✓ Configure pod anti-affinity (spread across nodes)
├── ✓ Set minAvailable in PDB
└── ✓ Use separate storage for master and replicas

✅ Security:
├── ✓ Store passwords in Secrets (not values.yaml)
├── ✓ Enable TLS if external access needed
├── ✓ Use NetworkPolicy to restrict access
├── ✓ Run as non-root user
├── ✓ Drop all capabilities
├── ✓ Disable dangerous commands (FLUSHDB, FLUSHALL)
└── ✓ Use RBAC for service accounts

✅ Monitoring:
├── ✓ Enable metrics.enabled: true
├── ✓ Configure ServiceMonitor for Prometheus
├── ✓ Set up PrometheusRule for alerts
├── ✓ Create Grafana dashboards
└── ✓ Monitor slow log

✅ Backup:
├── ✓ Enable both RDB and AOF
├── ✓ Configure backup schedule (sidecar or CronJob)
├── ✓ Test restore procedure quarterly
├── ✓ Store backups off-cluster (S3/GCS/Azure)
└── ✓ Document restore procedure

✅ Operations:
├── ✓ Use GitOps (ArgoCD/Flux)
├── ✓ Version control values files
├── ✓ Test in staging before production
├── ✓ Use --atomic flag for production upgrades
├── ✓ Document runbooks
└── ✓ Set up on-call rotation

✅ Cost Optimization:
├── ✓ Right-size resources (don't over-provision)
├── ✓ Use node selectors for spot instances (dev/staging)
├── ✓ Enable vertical pod autoscaler (optional)
└── ✓ Clean up unused PVCs
```

### Structure de repository recommandée

```
infrastructure/
├── helm/
│   └── redis/
│       ├── Chart.yaml              # Chart metadata (if custom chart)
│       ├── values.yaml             # Common values (all envs)
│       ├── values-dev.yaml         # Dev overrides
│       ├── values-staging.yaml     # Staging overrides
│       ├── values-production.yaml  # Production overrides
│       ├── templates/              # Custom templates (if needed)
│       │   └── custom-resource.yaml
│       └── README.md               # Documentation
│
├── secrets/
│   ├── redis-secret-dev.enc.yaml       # Sealed secret (dev)
│   ├── redis-secret-staging.enc.yaml   # Sealed secret (staging)
│   └── redis-secret-production.enc.yaml # Sealed secret (prod)
│
├── argocd/
│   ├── redis-dev.yaml              # ArgoCD app (dev)
│   ├── redis-staging.yaml          # ArgoCD app (staging)
│   └── redis-production.yaml       # ArgoCD app (prod)
│
├── scripts/
│   ├── backup-redis.sh             # Backup script
│   ├── restore-redis.sh            # Restore script
│   └── test-redis.sh               # Smoke tests
│
└── docs/
    ├── runbooks/
    │   ├── deployment.md
    │   ├── failover.md
    │   ├── backup-restore.md
    │   └── troubleshooting.md
    └── architecture.md
```

---

## ✅ Conclusion

### Points clés à retenir

1. **Helm = Package manager pour Kubernetes**
   - Templating puissant
   - Gestion de versions
   - Rollback facile
   - Distribution via repositories

2. **Bitnami Redis = Chart le plus populaire**
   - 50M+ downloads
   - Bien maintenu (VMware)
   - 3 architectures (Standalone, Replication, Sentinel)
   - Production-ready

3. **Configuration par environnement**
   - values.yaml (base)
   - values-{env}.yaml (overrides)
   - Secrets séparés (Sealed Secrets, External Secrets)

4. **GitOps recommandé**
   - ArgoCD ou Flux CD
   - Git = source of truth
   - Déploiements automatisés
   - Drift detection

5. **Monitoring essentiel**
   - Redis Exporter intégré
   - Prometheus + Grafana
   - Alerting configuré
   - Dashboards pré-faits

### Recommandations finales

```yaml
Pour commencer:
└─> Utilisez Bitnami Redis Helm chart
    ├── Déploiement en 5 minutes
    ├── Architecture replication (1 master + 2 replicas)
    ├── Metrics activées
    └── PDB configuré

Pour production:
└─> Ajoutez GitOps
    ├── ArgoCD ou Flux CD
    ├── Secrets management (Sealed Secrets / External Secrets)
    ├── Multi-environment (dev, staging, prod)
    └── Monitoring complet (Prometheus + Grafana)

Pour besoins avancés:
└─> Considérez les Operators (section 15.9)
    ├── Si besoin Redis Cluster
    ├── Si besoin Sentinel avec automation
    ├── Si backup/restore automatique requis
    └── Si failover automatique critique
```

### Quand ne PAS utiliser Helm

```
N'utilisez PAS Helm si:
├── Vous apprenez Kubernetes (commencez par raw manifests)
├── Vous avez besoin de contrôle total sur chaque ligne
├── Votre setup est très spécifique/unique
├── L'équipe n'est pas à l'aise avec le templating
└── Vous préférez les Operators pour l'automation

Dans ces cas:
├── Utilisez raw manifests (section 15.8) ou
└── Utilisez Operators (section 15.9)
```

---

**🎯 Module 15 COMPLÉTÉ !** Nous avons maintenant couvert l'intégralité du spectre Redis dans le cloud et conteneurs : solutions managées (AWS, Azure, GCP, Redis Enterprise), conteneurisation (Docker, Docker Compose), orchestration (Kubernetes StatefulSets, Operators), et déploiement automatisé (Helm, GitOps). Vous disposez de tous les outils et connaissances pour déployer et gérer Redis en production dans n'importe quel environnement.

⏭️ [Études de cas et patterns réels](/16-etudes-cas-patterns-reels/README.md)
