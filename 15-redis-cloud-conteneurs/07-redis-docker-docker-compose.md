🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.7 Redis avec Docker et Docker Compose

## 🎯 Objectifs

- Maîtriser les déploiements Redis avec Docker
- Configurer Redis en haute disponibilité avec Docker Compose
- Implémenter la persistence avec volumes Docker
- Sécuriser les déploiements conteneurisés
- Déployer Redis Sentinel et Cluster avec Compose
- Monitorer et debugger les conteneurs Redis
- Comprendre les limites vs Kubernetes

---

## 🐳 Redis avec Docker

### Image officielle et variantes

```yaml
Images Redis disponibles (Docker Hub):

redis:latest
├── Base: Debian Bookworm
├── Redis: Latest stable version
├── Size: ~138MB
└── Use: Development, testing

redis:7.4-alpine
├── Base: Alpine Linux 3.20
├── Redis: 7.4.x
├── Size: ~42MB (3x smaller!)
└── Use: Production (lightweight)

redis:7.4-bookworm
├── Base: Debian Bookworm
├── Redis: 7.4.x
├── Size: ~138MB
└── Use: Production (full tooling)

redis/redis-stack:latest
├── Includes: RediSearch, RedisJSON, RedisTimeSeries, RedisBloom
├── Size: ~500MB
├── Use: Development with modules
└── Not recommended for production (use Redis Enterprise)

Versions recommandées production:
├── redis:7.4-alpine → Lightweight, secure
├── redis:7.2-alpine → LTS stability
└── redis:6.2-alpine → Legacy compatibility
```

### Configuration de base

```dockerfile
# Dockerfile optimisé pour production
FROM redis:7.4-alpine

# Metadata
LABEL maintainer="platform-team@company.com"
LABEL version="1.0"
LABEL description="Redis 7.4 production-ready image"

# Install additional tools (optional)
RUN apk add --no-cache \
    bash \
    curl \
    ca-certificates

# Create redis user (already exists in base image)
# Create directories with proper permissions
RUN mkdir -p /data /etc/redis && \
    chown -R redis:redis /data /etc/redis

# Copy custom redis.conf
COPY redis.conf /etc/redis/redis.conf

# Fix permissions
RUN chown redis:redis /etc/redis/redis.conf && \
    chmod 640 /etc/redis/redis.conf

# Switch to redis user (non-root)
USER redis

# Expose port
EXPOSE 6379

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD redis-cli ping || exit 1

# Volume for persistence
VOLUME ["/data"]

# Entry point
ENTRYPOINT ["redis-server"]
CMD ["/etc/redis/redis.conf"]
```

### Configuration redis.conf pour Docker

```conf
# redis.conf - Production configuration for Docker

# Network
bind 0.0.0.0
port 6379
tcp-backlog 511
timeout 300
tcp-keepalive 300

# General
daemonize no
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile ""
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

# Security
# Password should be set via environment variable
requirepass ${REDIS_PASSWORD}
# Disable dangerous commands
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG_ADMIN_ONLY"
rename-command SHUTDOWN "SHUTDOWN_ADMIN_ONLY"

# Limits
maxclients 10000
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 5

# Append Only File (AOF)
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

# Latency monitoring
latency-monitor-threshold 100

# Event notification
notify-keyspace-events ""

# Advanced config
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

# TLS/SSL (if enabled)
# tls-port 6379
# port 0
# tls-cert-file /etc/redis/certs/redis.crt
# tls-key-file /etc/redis/certs/redis.key
# tls-ca-cert-file /etc/redis/certs/ca.crt
# tls-auth-clients no
```

### Commandes Docker de base

```bash
# Run Redis en développement (données éphémères)
docker run -d \
  --name redis-dev \
  -p 6379:6379 \
  redis:7.4-alpine

# Run Redis avec password
docker run -d \
  --name redis-secure \
  -p 6379:6379 \
  -e REDIS_PASSWORD=my-secret-password \
  redis:7.4-alpine \
  redis-server --requirepass my-secret-password

# Run Redis avec volume persistant
docker run -d \
  --name redis-persistent \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7.4-alpine \
  redis-server --appendonly yes

# Run Redis avec configuration custom
docker run -d \
  --name redis-custom \
  -p 6379:6379 \
  -v $(pwd)/redis.conf:/etc/redis/redis.conf \
  -v redis-data:/data \
  redis:7.4-alpine \
  redis-server /etc/redis/redis.conf

# Run Redis avec resource limits
docker run -d \
  --name redis-limited \
  -p 6379:6379 \
  -m 2g \
  --cpus="1.5" \
  --memory-swap 2g \
  -v redis-data:/data \
  redis:7.4-alpine \
  redis-server --maxmemory 1900mb --maxmemory-policy allkeys-lru

# Connect to Redis
docker exec -it redis-dev redis-cli

# View logs
docker logs -f redis-dev

# Stats
docker stats redis-dev

# Inspect
docker inspect redis-dev

# Backup data
docker run --rm \
  -v redis-data:/data \
  -v $(pwd)/backups:/backup \
  alpine \
  tar czf /backup/redis-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .

# Restore data
docker run --rm \
  -v redis-data:/data \
  -v $(pwd)/backups:/backup \
  alpine \
  sh -c "cd /data && tar xzf /backup/redis-backup-20240101-120000.tar.gz"
```

---

## 🔄 Redis avec Docker Compose

### Architecture simple (standalone)

```yaml
# docker-compose.yml - Simple Redis standalone

version: '3.9'

services:
  redis:
    image: redis:7.4-alpine
    container_name: redis-standalone
    restart: unless-stopped

    # Ports
    ports:
      - "6379:6379"

    # Environment
    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD:-changeme}

    # Command with password
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-changeme}
      --appendonly yes
      --appendfsync everysec
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru

    # Volumes
    volumes:
      - redis-data:/data
      - ./redis.conf:/etc/redis/redis.conf:ro

    # Health check
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

    # Networks
    networks:
      - redis-network

volumes:
  redis-data:
    driver: local

networks:
  redis-network:
    driver: bridge
```

### Architecture Primary-Replica

```yaml
# docker-compose-replication.yml - Redis avec réplication

version: '3.9'

services:
  # Primary (Master)
  redis-primary:
    image: redis:7.4-alpine
    container_name: redis-primary
    restart: unless-stopped

    ports:
      - "6379:6379"

    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD:-changeme}

    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-changeme}
      --masterauth ${REDIS_PASSWORD:-changeme}
      --appendonly yes
      --appendfsync everysec
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
      --save 60 10000

    volumes:
      - redis-primary-data:/data

    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

    networks:
      - redis-network

  # Replica 1
  redis-replica-1:
    image: redis:7.4-alpine
    container_name: redis-replica-1
    restart: unless-stopped

    ports:
      - "6380:6379"

    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD:-changeme}

    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-changeme}
      --masterauth ${REDIS_PASSWORD:-changeme}
      --replicaof redis-primary 6379
      --replica-read-only yes
      --appendonly yes
      --appendfsync everysec
      --maxmemory 2gb

    volumes:
      - redis-replica-1-data:/data

    depends_on:
      redis-primary:
        condition: service_healthy

    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

    networks:
      - redis-network

  # Replica 2
  redis-replica-2:
    image: redis:7.4-alpine
    container_name: redis-replica-2
    restart: unless-stopped

    ports:
      - "6381:6379"

    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD:-changeme}

    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-changeme}
      --masterauth ${REDIS_PASSWORD:-changeme}
      --replicaof redis-primary 6379
      --replica-read-only yes
      --appendonly yes
      --appendfsync everysec
      --maxmemory 2gb

    volumes:
      - redis-replica-2-data:/data

    depends_on:
      redis-primary:
        condition: service_healthy

    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

    networks:
      - redis-network

volumes:
  redis-primary-data:
    driver: local
  redis-replica-1-data:
    driver: local
  redis-replica-2-data:
    driver: local

networks:
  redis-network:
    driver: bridge
```

### Architecture avec Redis Sentinel (HA)

```yaml
# docker-compose-sentinel.yml - Redis avec Sentinel pour HA

version: '3.9'

services:
  # Redis Primary
  redis-primary:
    image: redis:7.4-alpine
    container_name: redis-primary
    restart: unless-stopped

    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD:-changeme}

    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-changeme}
      --masterauth ${REDIS_PASSWORD:-changeme}
      --appendonly yes
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru

    volumes:
      - redis-primary-data:/data

    networks:
      - redis-sentinel-network

    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Replica 1
  redis-replica-1:
    image: redis:7.4-alpine
    container_name: redis-replica-1
    restart: unless-stopped

    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD:-changeme}

    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-changeme}
      --masterauth ${REDIS_PASSWORD:-changeme}
      --replicaof redis-primary 6379
      --replica-read-only yes
      --appendonly yes
      --maxmemory 2gb

    volumes:
      - redis-replica-1-data:/data

    depends_on:
      - redis-primary

    networks:
      - redis-sentinel-network

    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Replica 2
  redis-replica-2:
    image: redis:7.4-alpine
    container_name: redis-replica-2
    restart: unless-stopped

    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD:-changeme}

    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-changeme}
      --masterauth ${REDIS_PASSWORD:-changeme}
      --replicaof redis-primary 6379
      --replica-read-only yes
      --appendonly yes
      --maxmemory 2gb

    volumes:
      - redis-replica-2-data:/data

    depends_on:
      - redis-primary

    networks:
      - redis-sentinel-network

    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Sentinel 1
  redis-sentinel-1:
    image: redis:7.4-alpine
    container_name: redis-sentinel-1
    restart: unless-stopped

    ports:
      - "26379:26379"

    command: >
      redis-sentinel /etc/redis/sentinel.conf
      --sentinel

    volumes:
      - ./sentinel-1.conf:/etc/redis/sentinel.conf
      - redis-sentinel-1-data:/data

    depends_on:
      - redis-primary
      - redis-replica-1
      - redis-replica-2

    networks:
      - redis-sentinel-network

    healthcheck:
      test: ["CMD", "redis-cli", "-p", "26379", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Sentinel 2
  redis-sentinel-2:
    image: redis:7.4-alpine
    container_name: redis-sentinel-2
    restart: unless-stopped

    ports:
      - "26380:26379"

    command: >
      redis-sentinel /etc/redis/sentinel.conf
      --sentinel

    volumes:
      - ./sentinel-2.conf:/etc/redis/sentinel.conf
      - redis-sentinel-2-data:/data

    depends_on:
      - redis-primary
      - redis-replica-1
      - redis-replica-2

    networks:
      - redis-sentinel-network

    healthcheck:
      test: ["CMD", "redis-cli", "-p", "26379", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Sentinel 3
  redis-sentinel-3:
    image: redis:7.4-alpine
    container_name: redis-sentinel-3
    restart: unless-stopped

    ports:
      - "26381:26379"

    command: >
      redis-sentinel /etc/redis/sentinel.conf
      --sentinel

    volumes:
      - ./sentinel-3.conf:/etc/redis/sentinel.conf
      - redis-sentinel-3-data:/data

    depends_on:
      - redis-primary
      - redis-replica-1
      - redis-replica-2

    networks:
      - redis-sentinel-network

    healthcheck:
      test: ["CMD", "redis-cli", "-p", "26379", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis-primary-data:
  redis-replica-1-data:
  redis-replica-2-data:
  redis-sentinel-1-data:
  redis-sentinel-2-data:
  redis-sentinel-3-data:

networks:
  redis-sentinel-network:
    driver: bridge
```

### Configuration Sentinel

```conf
# sentinel-1.conf (identique pour sentinel-2.conf et sentinel-3.conf)

# Sentinel port
port 26379

# Sentinel dir
dir /data

# Sentinel monitor (automatically updated by Sentinel)
sentinel monitor mymaster redis-primary 6379 2

# Auth
sentinel auth-pass mymaster changeme

# Timeouts
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1

# Notification scripts (optional)
# sentinel notification-script mymaster /etc/redis/notify.sh
# sentinel client-reconfig-script mymaster /etc/redis/reconfig.sh

# Logging
loglevel notice
logfile ""

# Quorum: 2 sentinels must agree before failover
# With 3 sentinels, 2 is the minimum for majority
```

### Script de démarrage Sentinel

```bash
#!/bin/bash
# start-sentinel-stack.sh

# Set password
export REDIS_PASSWORD="your-secure-password-here"

# Create sentinel configs if not exist
for i in {1..3}; do
  if [ ! -f "sentinel-$i.conf" ]; then
    cat > "sentinel-$i.conf" <<EOF
port 26379
dir /data
sentinel monitor mymaster redis-primary 6379 2
sentinel auth-pass mymaster ${REDIS_PASSWORD}
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
loglevel notice
logfile ""
EOF
    chmod 644 "sentinel-$i.conf"
  fi
done

# Start stack
docker-compose -f docker-compose-sentinel.yml up -d

# Wait for services
echo "Waiting for Redis and Sentinel to start..."
sleep 10

# Check status
echo ""
echo "=== Redis Primary Info ==="
docker exec redis-primary redis-cli -a ${REDIS_PASSWORD} --no-auth-warning INFO replication

echo ""
echo "=== Sentinel 1 Info ==="
docker exec redis-sentinel-1 redis-cli -p 26379 SENTINEL masters

echo ""
echo "Stack started successfully!"
echo "Redis Primary: localhost:6379"
echo "Sentinel 1: localhost:26379"
echo "Sentinel 2: localhost:26380"
echo "Sentinel 3: localhost:26381"
```

---

## 🔐 Sécurité avec Docker

### Secrets management

```yaml
# docker-compose-secrets.yml - Using Docker secrets

version: '3.9'

services:
  redis:
    image: redis:7.4-alpine
    restart: unless-stopped

    secrets:
      - redis_password

    # Use secret for password
    command: >
      sh -c '
      redis-server
      --requirepass "$$(cat /run/secrets/redis_password)"
      --appendonly yes
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
      '

    volumes:
      - redis-data:/data

    networks:
      - redis-network

    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '2'
          memory: 2G

secrets:
  redis_password:
    file: ./secrets/redis_password.txt

volumes:
  redis-data:

networks:
  redis-network:
```

### TLS/SSL avec certificats

```yaml
# docker-compose-tls.yml - Redis with TLS

version: '3.9'

services:
  redis-tls:
    image: redis:7.4-alpine
    container_name: redis-tls
    restart: unless-stopped

    ports:
      - "6379:6379"

    volumes:
      - redis-tls-data:/data
      - ./certs:/etc/redis/certs:ro
      - ./redis-tls.conf:/etc/redis/redis.conf:ro

    command: redis-server /etc/redis/redis.conf

    networks:
      - redis-tls-network

    healthcheck:
      test: >
        redis-cli
        --tls
        --cert /etc/redis/certs/redis.crt
        --key /etc/redis/certs/redis.key
        --cacert /etc/redis/certs/ca.crt
        ping
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  redis-tls-data:

networks:
  redis-tls-network:
    driver: bridge
```

```conf
# redis-tls.conf

# Disable non-TLS port
port 0

# Enable TLS
tls-port 6379
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt

# TLS options
tls-auth-clients optional
tls-replication yes
tls-cluster yes

# Security
requirepass changeme
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG ""

# Persistence
appendonly yes
appendfsync everysec
dir /data

# Limits
maxmemory 2gb
maxmemory-policy allkeys-lru
```

### Génération de certificats TLS

```bash
#!/bin/bash
# generate-tls-certs.sh

CERTS_DIR="./certs"
mkdir -p $CERTS_DIR

# Generate CA key and certificate
openssl genrsa -out $CERTS_DIR/ca.key 4096
openssl req -x509 -new -nodes \
  -key $CERTS_DIR/ca.key \
  -sha256 -days 3650 \
  -out $CERTS_DIR/ca.crt \
  -subj "/C=US/ST=State/L=City/O=Company/OU=IT/CN=Redis-CA"

# Generate Redis server key and CSR
openssl genrsa -out $CERTS_DIR/redis.key 2048
openssl req -new \
  -key $CERTS_DIR/redis.key \
  -out $CERTS_DIR/redis.csr \
  -subj "/C=US/ST=State/L=City/O=Company/OU=IT/CN=redis"

# Sign Redis certificate with CA
openssl x509 -req \
  -in $CERTS_DIR/redis.csr \
  -CA $CERTS_DIR/ca.crt \
  -CAkey $CERTS_DIR/ca.key \
  -CAcreateserial \
  -out $CERTS_DIR/redis.crt \
  -days 365 \
  -sha256

# Generate client key and certificate (optional)
openssl genrsa -out $CERTS_DIR/client.key 2048
openssl req -new \
  -key $CERTS_DIR/client.key \
  -out $CERTS_DIR/client.csr \
  -subj "/C=US/ST=State/L=City/O=Company/OU=IT/CN=redis-client"

openssl x509 -req \
  -in $CERTS_DIR/client.csr \
  -CA $CERTS_DIR/ca.crt \
  -CAkey $CERTS_DIR/ca.key \
  -CAcreateserial \
  -out $CERTS_DIR/client.crt \
  -days 365 \
  -sha256

# Set permissions
chmod 600 $CERTS_DIR/*.key
chmod 644 $CERTS_DIR/*.crt

echo "TLS certificates generated in $CERTS_DIR"
```

---

## 📊 Monitoring et observabilité

### Stack de monitoring complet

```yaml
# docker-compose-monitoring.yml - Redis avec monitoring complet

version: '3.9'

services:
  # Redis
  redis:
    image: redis:7.4-alpine
    container_name: redis
    restart: unless-stopped

    ports:
      - "6379:6379"

    command: >
      redis-server
      --requirepass changeme
      --appendonly yes
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru

    volumes:
      - redis-data:/data

    networks:
      - monitoring

    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "changeme", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  # Redis Exporter (for Prometheus)
  redis-exporter:
    image: oliver006/redis_exporter:v1.58-alpine
    container_name: redis-exporter
    restart: unless-stopped

    ports:
      - "9121:9121"

    environment:
      - REDIS_ADDR=redis://redis:6379
      - REDIS_PASSWORD=changeme

    depends_on:
      - redis

    networks:
      - monitoring

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped

    ports:
      - "9090:9090"

    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus

    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'

    depends_on:
      - redis-exporter

    networks:
      - monitoring

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped

    ports:
      - "3000:3000"

    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false

    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro

    depends_on:
      - prometheus

    networks:
      - monitoring

  # AlertManager (optional)
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped

    ports:
      - "9093:9093"

    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager-data:/alertmanager

    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'

    networks:
      - monitoring

volumes:
  redis-data:
  prometheus-data:
  grafana-data:
  alertmanager-data:

networks:
  monitoring:
    driver: bridge
```

### Configuration Prometheus

```yaml
# prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'docker-redis'
    environment: 'production'

# Alerting configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load rules
rule_files:
  - '/etc/prometheus/alerts/*.yml'

# Scrape configurations
scrape_configs:
  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets:
          - redis-exporter:9121
        labels:
          instance: 'redis-primary'
          role: 'cache'

  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets:
          - localhost:9090
```

### Règles d'alerting

```yaml
# alerts/redis-alerts.yml

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
        annotations:
          summary: "Redis instance is down"
          description: "Redis instance {{ $labels.instance }} has been down for more than 1 minute."

      # High memory usage
      - alert: RedisHighMemoryUsage
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage is high"
          description: "Redis instance {{ $labels.instance }} memory usage is above 90% (current: {{ $value | humanizePercentage }})."

      # Too many connections
      - alert: RedisTooManyConnections
        expr: redis_connected_clients > 5000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Too many Redis connections"
          description: "Redis instance {{ $labels.instance }} has {{ $value }} connections (threshold: 5000)."

      # Replication lag
      - alert: RedisReplicationLag
        expr: redis_replication_lag_bytes > 1000000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis replication lag is high"
          description: "Redis replica {{ $labels.instance }} is lagging behind master by {{ $value | humanize }}B."

      # Rejected connections
      - alert: RedisRejectedConnections
        expr: increase(redis_rejected_connections_total[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Redis is rejecting connections"
          description: "Redis instance {{ $labels.instance }} has rejected {{ $value }} connections in the last 5 minutes."

      # High CPU usage
      - alert: RedisHighCPU
        expr: rate(redis_cpu_sys_seconds_total[5m]) > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis CPU usage is high"
          description: "Redis instance {{ $labels.instance }} CPU usage is above 80%."
```

### Configuration Grafana datasource

```yaml
# grafana/datasources/prometheus.yml

apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

### Dashboard Grafana (JSON excerpt)

```json
{
  "dashboard": {
    "title": "Redis Monitoring",
    "panels": [
      {
        "title": "Connected Clients",
        "targets": [
          {
            "expr": "redis_connected_clients",
            "legendFormat": "{{ instance }}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Memory Usage",
        "targets": [
          {
            "expr": "redis_memory_used_bytes / redis_memory_max_bytes * 100",
            "legendFormat": "{{ instance }}"
          }
        ],
        "type": "gauge",
        "thresholds": "80,90"
      },
      {
        "title": "Commands Per Second",
        "targets": [
          {
            "expr": "rate(redis_commands_processed_total[1m])",
            "legendFormat": "{{ instance }}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Cache Hit Rate",
        "targets": [
          {
            "expr": "rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) * 100",
            "legendFormat": "Hit Rate %"
          }
        ],
        "type": "stat"
      }
    ]
  }
}
```

---

## 🧪 Redis Cluster avec Docker Compose

### Configuration cluster (6 nœuds)

```yaml
# docker-compose-cluster.yml - Redis Cluster (3 primaries + 3 replicas)

version: '3.9'

services:
  # Primary nodes
  redis-cluster-1:
    image: redis:7.4-alpine
    container_name: redis-cluster-1
    restart: unless-stopped
    ports:
      - "7001:6379"
      - "17001:16379"
    volumes:
      - redis-cluster-1-data:/data
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass changeme
      --masterauth changeme
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.11

  redis-cluster-2:
    image: redis:7.4-alpine
    container_name: redis-cluster-2
    restart: unless-stopped
    ports:
      - "7002:6379"
      - "17002:16379"
    volumes:
      - redis-cluster-2-data:/data
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass changeme
      --masterauth changeme
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.12

  redis-cluster-3:
    image: redis:7.4-alpine
    container_name: redis-cluster-3
    restart: unless-stopped
    ports:
      - "7003:6379"
      - "17003:16379"
    volumes:
      - redis-cluster-3-data:/data
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass changeme
      --masterauth changeme
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.13

  # Replica nodes
  redis-cluster-4:
    image: redis:7.4-alpine
    container_name: redis-cluster-4
    restart: unless-stopped
    ports:
      - "7004:6379"
      - "17004:16379"
    volumes:
      - redis-cluster-4-data:/data
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass changeme
      --masterauth changeme
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.14

  redis-cluster-5:
    image: redis:7.4-alpine
    container_name: redis-cluster-5
    restart: unless-stopped
    ports:
      - "7005:6379"
      - "17005:16379"
    volumes:
      - redis-cluster-5-data:/data
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass changeme
      --masterauth changeme
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.15

  redis-cluster-6:
    image: redis:7.4-alpine
    container_name: redis-cluster-6
    restart: unless-stopped
    ports:
      - "7006:6379"
      - "17006:16379"
    volumes:
      - redis-cluster-6-data:/data
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass changeme
      --masterauth changeme
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.16

volumes:
  redis-cluster-1-data:
  redis-cluster-2-data:
  redis-cluster-3-data:
  redis-cluster-4-data:
  redis-cluster-5-data:
  redis-cluster-6-data:

networks:
  redis-cluster-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Script d'initialisation du cluster

```bash
#!/bin/bash
# init-redis-cluster.sh

echo "Starting Redis Cluster nodes..."
docker-compose -f docker-compose-cluster.yml up -d

echo "Waiting for nodes to start..."
sleep 10

echo "Creating Redis Cluster..."
docker exec -it redis-cluster-1 redis-cli \
  --cluster create \
  172.20.0.11:6379 \
  172.20.0.12:6379 \
  172.20.0.13:6379 \
  172.20.0.14:6379 \
  172.20.0.15:6379 \
  172.20.0.16:6379 \
  --cluster-replicas 1 \
  -a changeme \
  --cluster-yes

echo ""
echo "Cluster created successfully!"
echo ""
echo "Check cluster status:"
echo "docker exec -it redis-cluster-1 redis-cli -c -a changeme cluster info"
echo "docker exec -it redis-cluster-1 redis-cli -c -a changeme cluster nodes"
```

---

## 🔧 Best practices et production readiness

### Checklist production

```yaml
✅ Configuration:
├── ✓ Password authentication (requirepass)
├── ✓ Dangerous commands disabled (FLUSHDB, FLUSHALL, CONFIG)
├── ✓ Persistence enabled (AOF + RDB)
├── ✓ Maxmemory policy configured
├── ✓ TLS/SSL enabled for production
└── ✓ Proper logging configuration

✅ High Availability:
├── ✓ Replication configured (min 1 replica)
├── ✓ Sentinel for automatic failover (min 3 sentinels)
├── ✓ Health checks configured
├── ✓ Restart policy: unless-stopped
└── ✓ Multiple availability zones (if possible)

✅ Performance:
├── ✓ Resource limits set (CPU, memory)
├── ✓ Memory reservations configured
├── ✓ Appropriate maxmemory setting
├── ✓ Connection limits configured
└── ✓ Kernel tuning (if possible)

✅ Monitoring:
├── ✓ Prometheus + Grafana configured
├── ✓ Redis Exporter deployed
├── ✓ Alerting rules configured
├── ✓ Log aggregation setup
└── ✓ Dashboards configured

✅ Backup & Recovery:
├── ✓ Automated backup script
├── ✓ Backup retention policy (30 days)
├── ✓ Restore procedure documented
├── ✓ Backup verification (monthly)
└── ✓ Disaster recovery plan

✅ Security:
├── ✓ Non-root user in container
├── ✓ Read-only filesystem (where possible)
├── ✓ Secrets management (Docker secrets)
├── ✓ Network isolation
├── ✓ TLS encryption
└── ✓ Regular security updates

✅ Networking:
├── ✓ Custom bridge network
├── ✓ Internal DNS resolution
├── ✓ No host network mode
├── ✓ Firewall rules (if needed)
└── ✓ IPv6 disabled (if not needed)

✅ Volumes:
├── ✓ Named volumes (not bind mounts)
├── ✓ Proper permissions
├── ✓ Backup strategy
└── ✓ Volume driver appropriate (local/NFS/etc)
```

### Script de backup automatisé

```bash
#!/bin/bash
# redis-backup.sh - Automated Redis backup

set -e

# Configuration
REDIS_CONTAINER="redis-primary"
BACKUP_DIR="/backups/redis"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="redis-backup-${DATE}.tar.gz"

# Create backup directory
mkdir -p ${BACKUP_DIR}

echo "Starting Redis backup at $(date)"

# Trigger BGSAVE
docker exec ${REDIS_CONTAINER} redis-cli -a changeme --no-auth-warning BGSAVE

# Wait for BGSAVE to complete
echo "Waiting for BGSAVE to complete..."
while [ $(docker exec ${REDIS_CONTAINER} redis-cli -a changeme --no-auth-warning LASTSAVE) == $(docker exec ${REDIS_CONTAINER} redis-cli -a changeme --no-auth-warning LASTSAVE) ]; do
  sleep 1
done

# Create backup
docker run --rm \
  --volumes-from ${REDIS_CONTAINER} \
  -v ${BACKUP_DIR}:/backup \
  alpine \
  tar czf /backup/${BACKUP_FILE} -C /data .

# Calculate size
BACKUP_SIZE=$(du -h ${BACKUP_DIR}/${BACKUP_FILE} | cut -f1)
echo "Backup created: ${BACKUP_FILE} (${BACKUP_SIZE})"

# Clean old backups
echo "Cleaning backups older than ${RETENTION_DAYS} days..."
find ${BACKUP_DIR} -name "redis-backup-*.tar.gz" -mtime +${RETENTION_DAYS} -delete

# Upload to S3 (optional)
# aws s3 cp ${BACKUP_DIR}/${BACKUP_FILE} s3://my-bucket/redis-backups/

echo "Backup completed at $(date)"
```

### Script de restore

```bash
#!/bin/bash
# redis-restore.sh - Restore Redis from backup

set -e

if [ $# -ne 1 ]; then
  echo "Usage: $0 <backup-file>"
  echo "Example: $0 /backups/redis/redis-backup-20240101-120000.tar.gz"
  exit 1
fi

BACKUP_FILE=$1
REDIS_CONTAINER="redis-primary"

if [ ! -f "${BACKUP_FILE}" ]; then
  echo "Error: Backup file not found: ${BACKUP_FILE}"
  exit 1
fi

echo "WARNING: This will overwrite all data in ${REDIS_CONTAINER}!"
read -p "Are you sure? (yes/no): " confirmation

if [ "$confirmation" != "yes" ]; then
  echo "Restore cancelled."
  exit 0
fi

echo "Stopping Redis..."
docker-compose stop ${REDIS_CONTAINER}

echo "Restoring backup..."
docker run --rm \
  --volumes-from ${REDIS_CONTAINER} \
  -v $(dirname ${BACKUP_FILE}):/backup \
  alpine \
  sh -c "cd /data && rm -rf * && tar xzf /backup/$(basename ${BACKUP_FILE})"

echo "Starting Redis..."
docker-compose start ${REDIS_CONTAINER}

echo "Waiting for Redis to start..."
sleep 5

echo "Verifying restore..."
docker exec ${REDIS_CONTAINER} redis-cli -a changeme --no-auth-warning PING

echo "Restore completed successfully!"
```

---

## 📊 Comparaison Docker vs Kubernetes

### Avantages de Docker Compose

```yaml
✅ Avantages Docker Compose:

Simplicité:
├── Configuration YAML simple
├── Courbe d'apprentissage faible
├── Setup rapide (minutes)
└── Parfait pour dev/staging

Coût:
├── Aucun overhead d'orchestration
├── Fonctionne sur une seule machine
├── Pas besoin de cluster K8s
└── Idéal pour PME/startups

Debugging:
├── Logs faciles (docker logs)
├── Shell direct (docker exec)
├── Networking simple
└── Moins d'abstraction

Portabilité:
├── Fonctionne partout (Docker Desktop, Linux, cloud)
├── Pas de dépendance cloud
└── Migration facile

Use cases appropriés:
├── Environnements dev/test
├── Petites productions (<10 services)
├── Single-host deployments
├── Prototypage rapide
└── Budget contraint
```

### Limitations de Docker Compose

```yaml
❌ Limitations Docker Compose:

Scalabilité:
├── Limité à un seul host
├── Pas de scaling automatique
├── Pas de rolling updates
└── Pas de self-healing

High Availability:
├── Single point of failure (host)
├── Pas de multi-datacenter
├── Failover manuel
└── Pas de load balancing intégré

Orchestration:
├── Pas de scheduler
├── Pas de resource quotas
├── Pas de pod affinity/anti-affinity
└── Gestion manuelle des dépendances

Production limitations:
├── Pas adapté pour >5 hosts
├── Pas de zero-downtime deployments
├── Pas de canary deployments
├── Monitoring basique
└── Pas de service mesh

Quand passer à Kubernetes:
├── >3 hosts nécessaires
├── HA critique
├── Multi-région requis
├── >20 services
├── Scaling automatique nécessaire
└── Compliance/audit avancé
```

### Matrice de décision

```yaml
┌──────────────────────────────────────────────────────────────┐
│              Docker Compose vs Kubernetes                    │
├──────────────────────────────────────────────────────────────┤
│ Critère            │ Docker Compose │ Kubernetes             │
├──────────────────────────────────────────────────────────────┤
│ Complexité         │ ⭐⭐⭐⭐⭐     │ ⭐⭐
│ Setup time         │ 5 minutes      │ 1-3 hours              │
│ Learning curve     │ 1 jour         │ 2-4 semaines           │
│ Coût infra         │ $50-200/mois   │ $500-5000/mois         │
│ Max hosts          │ 1              │ 1000+                  │
│ HA native          │ ❌             │ ✅                     │
│ Auto-scaling       │ ❌             │ ✅                     │
│ Rolling updates    │ ❌             │ ✅                     │
│ Self-healing       │ ❌             │ ✅                     │
│ Multi-region       │ ❌             │ ✅                     │
│ Service mesh       │ ❌             │ ✅ (Istio/Linkerd)     │
│ Secrets mgmt       │ Basique        │ Avancé                 │
│ Monitoring         │ Manuel         │ Natif                  │
│ Cost @ 10 services │ ~$100/mois     │ ~$1000/mois            │
└──────────────────────────────────────────────────────────────┘

Recommandation par taille:
├── Startup (<5 services): Docker Compose
├── PME (5-20 services): Docker Compose ou Kubernetes
├── Entreprise (>20 services): Kubernetes
└── Multi-région: Kubernetes obligatoire
```

---

## ✅ Conclusion

### Points clés à retenir

1. **Docker pour développement et petite production**
   - Setup simple et rapide
   - Coût minimal
   - Parfait pour <5 services sur 1 host

2. **Docker Compose pour orchestration simple**
   - Multi-conteneurs sur un host
   - Configuration déclarative
   - Adapté pour staging et petites productions

3. **Sentinel pour HA avec Docker**
   - Failover automatique
   - 3 Sentinels minimum
   - Quorum de 2 pour 3 Sentinels

4. **Cluster pour sharding avec Docker**
   - 6 nœuds minimum (3 primaries + 3 replicas)
   - IPs statiques requises
   - Complexe à maintenir

5. **Monitoring essentiel**
   - Redis Exporter + Prometheus + Grafana
   - Alerting configuré
   - Logs centralisés

### Recommandations par environnement

```yaml
Développement:
└─> Docker simple (docker run)
    Cost: $0
    Setup: 5 minutes

Staging:
└─> Docker Compose avec réplication
    Cost: ~$100/mois
    Setup: 30 minutes

Production (petite):
└─> Docker Compose + Sentinel + Monitoring
    Cost: ~$200-500/mois
    Setup: 2-4 heures

Production (moyenne):
└─> Considérer Kubernetes
    Cost: ~$1000/mois
    Setup: 1-2 jours

Production (grande):
└─> Kubernetes obligatoire
    Cost: $2000+/mois
    Setup: 1 semaine
```

### Quand migrer vers Kubernetes

```
Migrez vers Kubernetes si:
  (Nombre de hosts > 1)
  OU
  (HA critique ET budget > $1000/mois)
  OU
  (Scaling automatique requis)
  OU
  (Multi-région nécessaire)
  OU
  (>20 microservices)

SINON
  Restez avec Docker Compose
```

---

**🎯 Prochaine section :** Nous allons maintenant explorer Redis sur Kubernetes avec StatefulSets dans la section 15.8.

⏭️ [Redis sur Kubernetes : StatefulSets](/15-redis-cloud-conteneurs/08-redis-kubernetes-statefulsets.md)
