🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.8 Redis sur Kubernetes : StatefulSets

## 🎯 Objectifs

- Comprendre l'architecture StatefulSets pour Redis
- Déployer Redis Primary-Replica avec StatefulSets
- Configurer la persistence avec PersistentVolumes
- Implémenter les probes et health checks
- Gérer le scaling et les mises à jour
- Monitorer Redis sur Kubernetes avec Prometheus
- Maîtriser les stratégies de backup et restore
- Comparer StatefulSets vs Operators

---

## 🏗️ StatefulSets vs Deployments

### Différences fondamentales

```
┌─────────────────────────────────────────────────────────────────┐
│              StatefulSets vs Deployments pour Redis             │
│                                                                 │
│  Deployments (stateless):                                       │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  pod-abc123                                                │ │
│  │  pod-def456                                                │ │
│  │  pod-ghi789                                                │ │
│  │                                                            │ │
│  │  • Noms aléatoires                                         │ │
│  │  • Ordre de création non garanti                           │ │
│  │  • Storage éphémère ou partagé                             │ │
│  │  • Parfait pour applications stateless                     │ │
│  │  • Pas adapté pour Redis (perte de données)                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  StatefulSets (stateful):                                       │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  redis-0  ←→ PVC-redis-0 (100Gi)                           │ │
│  │  redis-1  ←→ PVC-redis-1 (100Gi)                           │ │
│  │  redis-2  ←→ PVC-redis-2 (100Gi)                           │ │
│  │                                                            │ │
│  │  • Noms stables (redis-0, redis-1, ...)                    │ │
│  │  • Ordre de création garanti (0→1→2)                       │ │
│  │  • Storage dédié par pod (PVC)                             │ │
│  │  • Network identity stable (redis-0.redis-headless)        │ │
│  │  • Parfait pour bases de données                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Caractéristiques clés StatefulSets:                            │
│                                                                 │
│  1. Identité de pod stable                                      │
│     redis-0 restera toujours redis-0 après restart              │
│                                                                 │
│  2. DNS stable                                                  │
│     redis-0.redis-headless.default.svc.cluster.local            │
│                                                                 │
│  3. Storage dédié et persistant                                 │
│     Chaque pod a son propre PVC qui survit aux restarts         │
│                                                                 │
│  4. Ordre de déploiement/scaling                                │
│     Scale up: 0→1→2→3 (séquentiel)                              │
│     Scale down: 3→2→1→0 (ordre inverse)                         │
│                                                                 │
│  5. Garanties de rolling update                                 │
│     Update: redis-2 → redis-1 → redis-0 (ordre inverse)         │
└─────────────────────────────────────────────────────────────────┘
```

### Pourquoi StatefulSets pour Redis

```yaml
Raisons d'utiliser StatefulSets pour Redis:

1. Identity stable
   ├── redis-0 = toujours le primary
   ├── redis-1, redis-2 = replicas
   └── Configuration basée sur l'ordinal

2. Storage persistant
   ├── Chaque pod a son propre PVC
   ├── Les données survivent aux restarts
   └── Pas de perte de données lors du scaling

3. DNS prévisible
   ├── redis-0.redis-headless.default.svc.cluster.local
   ├── Permet la configuration primary/replica
   └── Sentinel peut découvrir les nœuds

4. Ordre de déploiement
   ├── Primary (redis-0) démarre en premier
   ├── Replicas ensuite
   └── Évite les split-brain scenarios

5. Rolling updates contrôlés
   ├── Update des replicas d'abord
   ├── Primary en dernier
   └── Minimise le downtime
```

---

## 📦 Configuration de base avec StatefulSet

### Namespace et labels

```yaml
# 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: redis
  labels:
    name: redis
    environment: production
```

### StorageClass

```yaml
# 01-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: redis-storage
  namespace: redis
provisioner: kubernetes.io/aws-ebs  # Change selon votre cloud
parameters:
  type: gp3
  iopsPerGB: "50"
  fsType: ext4
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Retain  # Important: Retain pour ne pas perdre les données
volumeBindingMode: WaitForFirstConsumer
---
# Alternative GCP
# apiVersion: storage.k8s.io/v1
# kind: StorageClass
# metadata:
#   name: redis-storage-gcp
# provisioner: kubernetes.io/gce-pd
# parameters:
#   type: pd-ssd
#   replication-type: regional-pd
# allowVolumeExpansion: true
# reclaimPolicy: Retain
# volumeBindingMode: WaitForFirstConsumer
---
# Alternative Azure
# apiVersion: storage.k8s.io/v1
# kind: StorageClass
# metadata:
#   name: redis-storage-azure
# provisioner: kubernetes.io/azure-disk
# parameters:
#   storageaccounttype: Premium_LRS
#   kind: Managed
# allowVolumeExpansion: true
# reclaimPolicy: Retain
# volumeBindingMode: WaitForFirstConsumer
```

### ConfigMap pour redis.conf

```yaml
# 02-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: redis
data:
  redis.conf: |
    # Network
    bind 0.0.0.0
    port 6379
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300

    # General
    daemonize no
    supervised no
    loglevel notice
    databases 16

    # Snapshotting (RDB)
    save 900 1
    save 300 10
    save 60 10000
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    dir /data

    # Replication
    replica-serve-stale-data yes
    replica-read-only yes
    repl-diskless-sync no
    repl-diskless-sync-delay 5
    repl-disable-tcp-nodelay no
    replica-priority 100

    # Security (password from env var)
    # requirepass will be set via environment variable

    # Limits
    maxclients 10000
    maxmemory 2gb
    maxmemory-policy allkeys-lru

    # Append Only File
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    aof-load-truncated yes
    aof-use-rdb-preamble yes

    # Lua scripting
    lua-time-limit 5000

    # Slow log
    slowlog-log-slower-than 10000
    slowlog-max-len 128

    # Latency monitor
    latency-monitor-threshold 100

    # Advanced config
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    activerehashing yes
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit replica 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
    hz 10
    dynamic-hz yes
    aof-rewrite-incremental-fsync yes
    rdb-save-incremental-fsync yes

  primary.conf: |
    # Additional config for primary
    min-replicas-to-write 1
    min-replicas-max-lag 10

  replica.conf: |
    # Additional config for replicas
    replica-read-only yes
```

### Secret pour credentials

```yaml
# 03-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: redis
type: Opaque
stringData:
  password: "changeme-in-production"
  # Generate secure password:
  # openssl rand -base64 32
```

### Headless Service

```yaml
# 04-service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: redis
  labels:
    app: redis
spec:
  clusterIP: None  # Headless service
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
      protocol: TCP
---
# 05-service-primary.yaml
# Service pour accéder au primary uniquement
apiVersion: v1
kind: Service
metadata:
  name: redis-primary
  namespace: redis
  labels:
    app: redis
    role: primary
spec:
  type: ClusterIP
  selector:
    app: redis
    statefulset.kubernetes.io/pod-name: redis-0  # Point vers redis-0 (primary)
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
      protocol: TCP
---
# 06-service-replicas.yaml
# Service pour load-balancer sur les replicas (read-only)
apiVersion: v1
kind: Service
metadata:
  name: redis-replicas
  namespace: redis
  labels:
    app: redis
    role: replica
spec:
  type: ClusterIP
  selector:
    app: redis
    # Exclut redis-0 en utilisant les labels (nécessite gestion manuelle)
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
      protocol: TCP
```

---

## 🚀 StatefulSet complet

### StatefulSet avec init containers

```yaml
# 07-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: redis
  labels:
    app: redis
spec:
  serviceName: redis-headless
  replicas: 3  # 1 primary + 2 replicas

  # Update strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods

  # Pod management policy
  podManagementPolicy: OrderedReady  # Démarre les pods séquentiellement

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
    spec:
      # Security context
      securityContext:
        fsGroup: 999  # redis group
        runAsUser: 999  # redis user
        runAsNonRoot: true

      # Init container pour configuration
      initContainers:
        - name: config
          image: redis:7.4-alpine
          command:
            - sh
            - -c
            - |
              set -ex
              # Copy base config
              cp /mnt/config-map/redis.conf /etc/redis/redis.conf

              # Get pod ordinal
              ordinal=${HOSTNAME##*-}

              # Configure primary or replica
              if [ "$ordinal" = "0" ]; then
                echo "Configuring as PRIMARY"
                cat /mnt/config-map/primary.conf >> /etc/redis/redis.conf
              else
                echo "Configuring as REPLICA of redis-0"
                cat /mnt/config-map/replica.conf >> /etc/redis/redis.conf
                echo "replicaof redis-0.redis-headless.redis.svc.cluster.local 6379" >> /etc/redis/redis.conf
              fi

              # Set password
              echo "requirepass ${REDIS_PASSWORD}" >> /etc/redis/redis.conf
              echo "masterauth ${REDIS_PASSWORD}" >> /etc/redis/redis.conf
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password
          volumeMounts:
            - name: config-map
              mountPath: /mnt/config-map
            - name: config
              mountPath: /etc/redis

      containers:
        # Redis container
        - name: redis
          image: redis:7.4-alpine

          command:
            - redis-server
            - /etc/redis/redis.conf

          ports:
            - name: redis
              containerPort: 6379
              protocol: TCP

          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password

          # Probes
          startupProbe:
            tcpSocket:
              port: redis
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 30

          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - redis-cli -a $REDIS_PASSWORD ping | grep PONG
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 3

          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - |
                  redis-cli -a $REDIS_PASSWORD ping | grep PONG
                  # Also check if replica is synced (if not redis-0)
                  ordinal=${HOSTNAME##*-}
                  if [ "$ordinal" != "0" ]; then
                    redis-cli -a $REDIS_PASSWORD INFO replication | grep master_link_status:up
                  fi
            initialDelaySeconds: 15
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3

          # Resources
          resources:
            requests:
              cpu: 500m
              memory: 2Gi
            limits:
              cpu: 2000m
              memory: 3Gi

          # Volume mounts
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /etc/redis

          # Security
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL

        # Redis Exporter (sidecar for Prometheus)
        - name: redis-exporter
          image: oliver006/redis_exporter:v1.58-alpine

          ports:
            - name: metrics
              containerPort: 9121
              protocol: TCP

          env:
            - name: REDIS_ADDR
              value: "localhost:6379"
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi

          livenessProbe:
            httpGet:
              path: /
              port: metrics
            initialDelaySeconds: 10
            periodSeconds: 10

          readinessProbe:
            httpGet:
              path: /
              port: metrics
            initialDelaySeconds: 5
            periodSeconds: 5

      # Volumes
      volumes:
        - name: config-map
          configMap:
            name: redis-config
        - name: config
          emptyDir: {}

      # Affinity (spread across nodes and zones)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - redis
                topologyKey: kubernetes.io/hostname
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - redis
                topologyKey: topology.kubernetes.io/zone

  # Volume claim templates
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: redis
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: redis-storage
        resources:
          requests:
            storage: 100Gi
```

---

## 🛡️ Sécurité et stabilité

### Pod Disruption Budget

```yaml
# 08-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
  namespace: redis
spec:
  minAvailable: 2  # Au moins 2 pods disponibles pendant maintenance
  selector:
    matchLabels:
      app: redis
```

### Network Policy

```yaml
# 09-networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-network-policy
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app: redis

  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow traffic from app namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: app
      ports:
        - protocol: TCP
          port: 6379

    # Allow traffic from monitoring namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9121  # Redis Exporter

    # Allow inter-pod communication (replication)
    - from:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53

    # Allow inter-pod communication
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
```

### Priority Class

```yaml
# 10-priorityclass.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: redis-priority
value: 1000000
globalDefault: false
description: "Priority class for Redis StatefulSet"
```

---

## 📊 Monitoring avec Prometheus

### ServiceMonitor (Prometheus Operator)

```yaml
# 11-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-metrics
  namespace: redis
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Règles d'alerting

```yaml
# 12-prometheusrule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: redis-alerts
  namespace: redis
  labels:
    prometheus: kube-prometheus
spec:
  groups:
    - name: redis
      interval: 30s
      rules:
        # Redis down
        - alert: RedisDown
          expr: redis_up == 0
          for: 1m
          labels:
            severity: critical
            component: redis
          annotations:
            summary: "Redis instance {{ $labels.pod }} is down"
            description: "Redis pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has been down for more than 1 minute."

        # High memory usage
        - alert: RedisHighMemoryUsage
          expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
          for: 5m
          labels:
            severity: warning
            component: redis
          annotations:
            summary: "Redis memory usage is high on {{ $labels.pod }}"
            description: "Redis pod {{ $labels.pod }} memory usage is above 90% (current: {{ $value | humanizePercentage }})."

        # Replication broken
        - alert: RedisReplicationBroken
          expr: redis_connected_slaves < 2
          for: 5m
          labels:
            severity: critical
            component: redis
          annotations:
            summary: "Redis primary has less than 2 replicas"
            description: "Redis primary {{ $labels.pod }} has only {{ $value }} connected replicas (expected: 2)."

        # Replication lag
        - alert: RedisReplicationLag
          expr: redis_slave_repl_offset - redis_master_repl_offset > 1000000
          for: 5m
          labels:
            severity: warning
            component: redis
          annotations:
            summary: "Redis replication lag is high"
            description: "Redis replica {{ $labels.pod }} is lagging behind primary by {{ $value | humanize }}B."

        # Too many connections
        - alert: RedisTooManyConnections
          expr: redis_connected_clients > 5000
          for: 5m
          labels:
            severity: warning
            component: redis
          annotations:
            summary: "Too many Redis connections on {{ $labels.pod }}"
            description: "Redis pod {{ $labels.pod }} has {{ $value }} connections (threshold: 5000)."

        # Rejected connections
        - alert: RedisRejectedConnections
          expr: increase(redis_rejected_connections_total[5m]) > 0
          for: 1m
          labels:
            severity: warning
            component: redis
          annotations:
            summary: "Redis is rejecting connections on {{ $labels.pod }}"
            description: "Redis pod {{ $labels.pod }} has rejected {{ $value }} connections in the last 5 minutes."

        # Low hit rate
        - alert: RedisLowHitRate
          expr: |
            rate(redis_keyspace_hits_total[5m])
            /
            (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
            < 0.8
          for: 10m
          labels:
            severity: info
            component: redis
          annotations:
            summary: "Redis cache hit rate is low on {{ $labels.pod }}"
            description: "Redis pod {{ $labels.pod }} hit rate is below 80% (current: {{ $value | humanizePercentage }})."

        # Slow commands
        - alert: RedisSlowCommands
          expr: redis_slowlog_length > 100
          for: 5m
          labels:
            severity: warning
            component: redis
          annotations:
            summary: "Redis has many slow commands on {{ $labels.pod }}"
            description: "Redis pod {{ $labels.pod }} has {{ $value }} commands in slow log."
```

---

## 🔄 Opérations courantes

### Déploiement complet

```bash
#!/bin/bash
# deploy-redis.sh - Deploy Redis StatefulSet

set -e

NAMESPACE="redis"

echo "Creating namespace..."
kubectl apply -f 00-namespace.yaml

echo "Creating storage class..."
kubectl apply -f 01-storageclass.yaml

echo "Creating ConfigMap..."
kubectl apply -f 02-configmap.yaml

echo "Creating Secret..."
kubectl apply -f 03-secret.yaml

echo "Creating Services..."
kubectl apply -f 04-service-headless.yaml
kubectl apply -f 05-service-primary.yaml
kubectl apply -f 06-service-replicas.yaml

echo "Creating StatefulSet..."
kubectl apply -f 07-statefulset.yaml

echo "Creating PodDisruptionBudget..."
kubectl apply -f 08-pdb.yaml

echo "Creating NetworkPolicy..."
kubectl apply -f 09-networkpolicy.yaml

echo "Creating PriorityClass..."
kubectl apply -f 10-priorityclass.yaml

# Wait for pods to be ready
echo ""
echo "Waiting for Redis pods to be ready..."
kubectl wait --for=condition=ready pod -l app=redis -n ${NAMESPACE} --timeout=300s

# Verify deployment
echo ""
echo "=== Deployment Status ==="
kubectl get statefulset -n ${NAMESPACE}
kubectl get pods -n ${NAMESPACE}
kubectl get pvc -n ${NAMESPACE}

# Check replication
echo ""
echo "=== Replication Status ==="
REDIS_PASSWORD=$(kubectl get secret redis-secret -n ${NAMESPACE} -o jsonpath='{.data.password}' | base64 -d)
kubectl exec -n ${NAMESPACE} redis-0 -- redis-cli -a ${REDIS_PASSWORD} --no-auth-warning INFO replication

echo ""
echo "Deployment completed successfully!"
echo "Primary: redis-primary.redis.svc.cluster.local:6379"
echo "Replicas: redis-replicas.redis.svc.cluster.local:6379"
```

### Scaling

```bash
# Scale up (add replicas)
kubectl scale statefulset redis -n redis --replicas=5

# Scale down (remove replicas)
kubectl scale statefulset redis -n redis --replicas=3

# Verify scaling
kubectl get pods -n redis -w

# Check new replica replication status
REDIS_PASSWORD=$(kubectl get secret redis-secret -n redis -o jsonpath='{.data.password}' | base64 -d)
kubectl exec -n redis redis-3 -- redis-cli -a ${REDIS_PASSWORD} --no-auth-warning INFO replication
```

### Rolling update

```bash
# Update Redis image version
kubectl set image statefulset/redis redis=redis:7.4.1-alpine -n redis

# Monitor rollout
kubectl rollout status statefulset/redis -n redis

# Check rollout history
kubectl rollout history statefulset/redis -n redis

# Rollback if needed
kubectl rollout undo statefulset/redis -n redis
```

### Backup

```bash
#!/bin/bash
# backup-redis.sh - Backup Redis data

set -e

NAMESPACE="redis"
BACKUP_DIR="/backups/redis-k8s"
DATE=$(date +%Y%m%d-%H%M%S)
REDIS_PASSWORD=$(kubectl get secret redis-secret -n ${NAMESPACE} -o jsonpath='{.data.password}' | base64 -d)

mkdir -p ${BACKUP_DIR}

echo "Starting Redis backup at $(date)"

# Backup each Redis pod
for pod in redis-0 redis-1 redis-2; do
  echo "Backing up ${pod}..."

  # Trigger BGSAVE
  kubectl exec -n ${NAMESPACE} ${pod} -- redis-cli -a ${REDIS_PASSWORD} --no-auth-warning BGSAVE

  # Wait for BGSAVE to complete
  sleep 5

  # Copy data from PVC
  kubectl exec -n ${NAMESPACE} ${pod} -- tar czf - /data > ${BACKUP_DIR}/${pod}-${DATE}.tar.gz

  echo "  Saved to ${BACKUP_DIR}/${pod}-${DATE}.tar.gz"
done

# Backup StatefulSet manifests
kubectl get statefulset redis -n ${NAMESPACE} -o yaml > ${BACKUP_DIR}/statefulset-${DATE}.yaml
kubectl get pvc -n ${NAMESPACE} -o yaml > ${BACKUP_DIR}/pvc-${DATE}.yaml

# Clean old backups (keep 30 days)
find ${BACKUP_DIR} -name "*.tar.gz" -mtime +30 -delete
find ${BACKUP_DIR} -name "*.yaml" -mtime +30 -delete

echo "Backup completed at $(date)"
```

### Restore

```bash
#!/bin/bash
# restore-redis.sh - Restore Redis from backup

set -e

if [ $# -ne 2 ]; then
  echo "Usage: $0 <pod-name> <backup-file>"
  echo "Example: $0 redis-0 /backups/redis-k8s/redis-0-20240101-120000.tar.gz"
  exit 1
fi

POD=$1
BACKUP_FILE=$2
NAMESPACE="redis"

if [ ! -f "${BACKUP_FILE}" ]; then
  echo "Error: Backup file not found: ${BACKUP_FILE}"
  exit 1
fi

echo "WARNING: This will overwrite data in ${POD}!"
read -p "Are you sure? (yes/no): " confirmation

if [ "$confirmation" != "yes" ]; then
  echo "Restore cancelled."
  exit 0
fi

echo "Stopping Redis on ${POD}..."
kubectl exec -n ${NAMESPACE} ${POD} -- redis-cli -a $(kubectl get secret redis-secret -n ${NAMESPACE} -o jsonpath='{.data.password}' | base64 -d) --no-auth-warning SHUTDOWN NOSAVE || true

sleep 5

echo "Restoring backup..."
kubectl exec -n ${NAMESPACE} ${POD} -i -- sh -c "cd / && tar xzf -" < ${BACKUP_FILE}

echo "Starting Redis..."
kubectl delete pod ${POD} -n ${NAMESPACE}

echo "Waiting for pod to restart..."
kubectl wait --for=condition=ready pod ${POD} -n ${NAMESPACE} --timeout=120s

echo "Restore completed successfully!"
```

---

## 🔍 Debugging et troubleshooting

### Scripts de diagnostic

```bash
#!/bin/bash
# diagnose-redis.sh - Diagnostic complet

NAMESPACE="redis"
REDIS_PASSWORD=$(kubectl get secret redis-secret -n ${NAMESPACE} -o jsonpath='{.data.password}' | base64 -d)

echo "=== Redis Cluster Status ==="
kubectl get statefulset,pods,pvc,svc -n ${NAMESPACE}

echo ""
echo "=== Redis Primary Replication Info ==="
kubectl exec -n ${NAMESPACE} redis-0 -- redis-cli -a ${REDIS_PASSWORD} --no-auth-warning INFO replication

echo ""
echo "=== Redis Replicas Sync Status ==="
for pod in redis-1 redis-2; do
  echo "--- ${pod} ---"
  kubectl exec -n ${NAMESPACE} ${pod} -- redis-cli -a ${REDIS_PASSWORD} --no-auth-warning INFO replication | grep -E "role|master_link_status|master_sync"
done

echo ""
echo "=== Memory Usage ==="
for pod in redis-0 redis-1 redis-2; do
  echo "--- ${pod} ---"
  kubectl exec -n ${NAMESPACE} ${pod} -- redis-cli -a ${REDIS_PASSWORD} --no-auth-warning INFO memory | grep -E "used_memory_human|maxmemory_human"
done

echo ""
echo "=== Connected Clients ==="
for pod in redis-0 redis-1 redis-2; do
  echo "--- ${pod} ---"
  kubectl exec -n ${NAMESPACE} ${pod} -- redis-cli -a ${REDIS_PASSWORD} --no-auth-warning CLIENT LIST | wc -l
done

echo ""
echo "=== Recent Events ==="
kubectl get events -n ${NAMESPACE} --sort-by='.lastTimestamp' | tail -20

echo ""
echo "=== Pod Resources ==="
kubectl top pods -n ${NAMESPACE}

echo ""
echo "=== PVC Usage ==="
for pod in redis-0 redis-1 redis-2; do
  echo "--- ${pod} ---"
  kubectl exec -n ${NAMESPACE} ${pod} -- df -h /data
done
```

### Commandes de debug courantes

```bash
# Check logs
kubectl logs -n redis redis-0 -c redis --tail=100 -f

# Interactive shell
kubectl exec -n redis redis-0 -it -- sh

# Redis CLI interactive
REDIS_PASSWORD=$(kubectl get secret redis-secret -n redis -o jsonpath='{.data.password}' | base64 -d)
kubectl exec -n redis redis-0 -it -- redis-cli -a ${REDIS_PASSWORD}

# Check config
kubectl exec -n redis redis-0 -- cat /etc/redis/redis.conf

# Test connectivity from another pod
kubectl run redis-test --rm -it --image=redis:7.4-alpine -- redis-cli -h redis-primary.redis.svc.cluster.local -a ${REDIS_PASSWORD}

# Port-forward to local machine
kubectl port-forward -n redis svc/redis-primary 6379:6379

# Check DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup redis-0.redis-headless.redis.svc.cluster.local

# Check network connectivity
kubectl run -it --rm debug --image=busybox --restart=Never -- telnet redis-primary.redis.svc.cluster.local 6379
```

---

## ⚖️ StatefulSets vs Operators

### Comparaison

```yaml
┌──────────────────────────────────────────────────────────────┐
│              StatefulSets vs Redis Operators                 │
├──────────────────────────────────────────────────────────────┤
│ Critère            │ StatefulSets   │ Operators              │
├──────────────────────────────────────────────────────────────┤
│ Setup complexity   │ Medium         │ High                   │
│ Maintenance        │ Manual         │ Automated              │
│ Scaling            │ Manual         │ Automatic              │
│ Failover           │ Manual         │ Automatic              │
│ Cluster support    │ ❌ Manual      │ ✅ Native              │
│ Sentinel support   │ ❌ Manual      │ ✅ Native              │
│ Backup/Restore     │ Manual scripts │ ✅ Built-in            │
│ Monitoring         │ Manual         │ ✅ Integrated          │
│ Rolling updates    │ ✅ Built-in    │ ✅ Enhanced            │
│ Multi-tenancy      │ ❌             │ ✅                     │
│ CRDs               │ ❌             │ ✅                     │
│ Learning curve     │ 1 semaine      │ 2-3 semaines           │
│ Community support  │ Large          │ Growing                │
│ Production ready   │ ✅             │ ✅                     │
│ Cost               │ $0             │ $0 (open source)       │
└──────────────────────────────────────────────────────────────┘

Utilisez StatefulSets si:
├── Besoin de contrôle total sur la configuration
├── Équipe confortable avec Kubernetes primitives
├── Réplication simple (primary-replica)
├── Pas besoin de Sentinel ou Cluster
├── Budget temps pour maintenance
└── Setup one-time acceptable

Utilisez Operators si:
├── Besoin de Redis Cluster (sharding)
├── Besoin de Redis Sentinel (HA)
├── Failover automatique requis
├── Backup/restore automatisé souhaité
├── Multi-tenancy nécessaire
├── Équipe petit ou moins expérimentée
└── Day-2 operations automatisées préférées

Operators populaires:
├── Redis Enterprise Operator (Redis Inc.)
│   └─> Commercial, full features, support officiel
├── Spotahome Redis Operator
│   └─> Open-source, Sentinel + Cluster, populaire
├── OT Redis Operator (OpsTree)
│   └─> Open-source, simple, bien documenté
└── Bitnami Redis Operator
    └─> Helm-based, facile à démarrer
```

### Exemple d'Operator (preview)

```yaml
# Preview: Redis Cluster avec OT Redis Operator
# (Sera couvert en détail dans la section 15.9)

apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisCluster
metadata:
  name: redis-cluster
  namespace: redis
spec:
  clusterSize: 3

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

  redisExporter:
    enabled: true
    image: oliver006/redis_exporter:v1.58-alpine

  storage:
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: redis-storage
        resources:
          requests:
            storage: 100Gi

  securityContext:
    runAsUser: 999
    fsGroup: 999
```

---

## ✅ Conclusion

### Points clés à retenir

1. **StatefulSets = idéal pour Redis stateful**
   - Identity stable (redis-0, redis-1, ...)
   - Storage persistant par pod
   - DNS prévisible
   - Ordre de déploiement garanti

2. **Configuration production-ready**
   - Init containers pour config dynamique
   - Probes (startup, liveness, readiness)
   - Resource limits appropriés
   - Pod anti-affinity pour HA

3. **High Availability**
   - 1 primary + 2+ replicas
   - PodDisruptionBudget (minAvailable: 2)
   - Replication automatique via init container
   - Monitoring avec Prometheus

4. **Operations**
   - Scaling horizontal facile
   - Rolling updates contrôlés
   - Backup/restore scriptés
   - Diagnostic et troubleshooting

5. **Limites de StatefulSets**
   - Pas de failover automatique
   - Pas de Redis Cluster natif
   - Pas de Sentinel natif
   - Maintenance manuelle

### Checklist production

```yaml
✅ Configuration:
├── ✓ StorageClass avec reclaimPolicy: Retain
├── ✓ PersistentVolumes avec backups
├── ✓ ConfigMap pour redis.conf
├── ✓ Secret pour credentials
├── ✓ Services (headless + primary + replicas)
└── ✓ Init containers pour configuration dynamique

✅ High Availability:
├── ✓ 3+ replicas (1 primary + 2 replicas minimum)
├── ✓ Pod anti-affinity (spread across nodes/zones)
├── ✓ PodDisruptionBudget (minAvailable: 2)
├── ✓ Replication automatique
└── ✓ Health checks complets (startup, liveness, readiness)

✅ Security:
├── ✓ Password dans Secret
├── ✓ NetworkPolicy (ingress + egress)
├── ✓ Non-root user (runAsUser: 999)
├── ✓ Drop capabilities
└── ✓ ReadOnlyRootFilesystem (où possible)

✅ Monitoring:
├── ✓ Redis Exporter sidecar
├── ✓ ServiceMonitor pour Prometheus
├── ✓ PrometheusRule avec alertes
└── ✓ Dashboards Grafana

✅ Operations:
├── ✓ Backup scripts automatisés
├── ✓ Restore procedure documentée
├── ✓ Diagnostic scripts
├── ✓ Rolling update strategy
└── ✓ Scaling procedure
```

### Quand passer aux Operators

```
Passez aux Operators si:
  (Besoin Redis Cluster OU Redis Sentinel)
  OU
  (Failover automatique requis)
  OU
  (Équipe <3 personnes Kubernetes)
  OU
  (>10 instances Redis à gérer)
  OU
  (Multi-tenancy nécessaire)

SINON
  StatefulSets sont suffisants
```

---

**🎯 Prochaine section :** Nous allons explorer les Redis Operators (Spotahome, OpsTree, Redis Enterprise) dans la section 15.9 pour automatiser encore plus les opérations.

⏭️ [Opérateurs Kubernetes pour Redis](/15-redis-cloud-conteneurs/09-operateurs-kubernetes-redis.md)
