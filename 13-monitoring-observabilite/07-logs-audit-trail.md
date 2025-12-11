🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.7 Logs et Audit Trail

## Introduction

Les logs Redis sont souvent négligés au profit des métriques, pourtant ils contiennent des informations critiques pour le troubleshooting, la sécurité et la conformité. Cette section couvre la configuration, la centralisation, l'analyse et l'audit des logs Redis en production.

### Pourquoi les logs sont essentiels

**Métriques vs Logs** :
```
Métriques (What?)          Logs (Why?)
─────────────────          ───────────
Memory: 95%         →      "maxmemory reached, evicting keys"
Latency: 250ms      →      "Fork: 2345ms for BGSAVE"
Clients: 1000       →      "Client X.X.X.X connected"
```

**Les logs répondent à** :
- **Pourquoi** l'incident s'est produit
- **Qui** a fait quoi et quand
- **Comment** le système a réagi
- **Quand** exactement l'événement est survenu

### Les 3 types de logs Redis

```
┌────────────────────────────────────────────────┐
│  1. Server Logs (redis.log)                    │
│     - Démarrage/arrêt                          │
│     - Warnings/Errors                          │
│     - Configuration changes                    │
│     - Replication events                       │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│  2. Slowlog (in-memory)                        │
│     - Commandes lentes                         │
│     - Latence par commande                     │
│     - Client IP                                │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│  3. Audit Logs (via modules ou ACL)            │
│     - Qui a exécuté quelle commande            │
│     - Tentatives d'authentification            │
│     - Violations ACL                           │
│     - Commandes sensibles (FLUSHDB, CONFIG)    │
└────────────────────────────────────────────────┘
```

## 1. Configuration du Logging Redis

### 1.1 Niveaux de log

**redis.conf** :
```conf
# Niveau de log
loglevel notice

# Niveaux disponibles:
# debug    - Très verbeux, uniquement dev/debug
# verbose  - Beaucoup d'infos, debug approfondi
# notice   - Modérément verbeux, recommandé production
# warning  - Uniquement warnings et errors
```

**Recommandations par environnement** :

| Environnement | loglevel | Justification |
|---------------|----------|---------------|
| **Production** | `notice` | Balance signal/bruit optimal |
| **Staging** | `verbose` | Plus de détails pour tests |
| **Dev** | `debug` | Maximum de détails |
| **Investigation** | `debug` temporaire | Troubleshooting approfondi |

### 1.2 Destination des logs

```conf
# Fichier de log
logfile /var/log/redis/redis-server.log

# Syslog (alternatif)
# syslog-enabled yes
# syslog-ident redis
# syslog-facility local0

# Stdout (conteneurs Docker)
# logfile ""
```

**Choix par infrastructure** :

#### Option 1 : Fichier local (VMs traditionnelles)
```conf
logfile /var/log/redis/redis-server.log
```

**Avantages** :
- Simple, fiable
- Pas de dépendance externe
- Rotation avec logrotate

**Configuration logrotate** (`/etc/logrotate.d/redis`) :
```
/var/log/redis/redis-server.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    missingok
    copytruncate
    postrotate
        /bin/kill -SIGUSR1 $(cat /var/run/redis/redis-server.pid) 2>/dev/null || true
    endscript
}
```

#### Option 2 : Syslog (centralisation)
```conf
syslog-enabled yes
syslog-ident redis-prod-master
syslog-facility local0
```

**Avantages** :
- Centralisation native
- Intégration avec infrastructure logging existante
- Pas de gestion de rotation

**Rsyslog config** (`/etc/rsyslog.d/redis.conf`) :
```
# Envoyer logs Redis vers serveur central
local0.* @@logserver.company.com:514

# Ou fichier local dédié
local0.* /var/log/redis/redis-syslog.log
```

#### Option 3 : Stdout (Containers)
```conf
logfile ""
```

**Pour Docker/Kubernetes** :
```yaml
# Kubernetes
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: redis
    args: ["--logfile", ""]
    # Logs visibles via kubectl logs
```

**Avantages** :
- Pas de gestion de fichiers
- Compatible avec systèmes de log containers (Fluentd, Fluent Bit)
- 12-factor app compliance

### 1.3 Commandes sensibles à logger

**Configuration pour audit** :
```conf
# Activer AOF pour tracer toutes les write commands (side-effect)
appendonly yes
appendfilename "appendonly.aof"

# Le AOF peut servir d'audit trail partiel
# Contient toutes les commandes qui modifient les données
```

**Limites** : AOF ne contient que les writes, pas les reads ni les commandes admin

## 2. Événements Importants dans les Logs

### 2.1 Événements de cycle de vie

#### Démarrage
```
# Pattern
[PID] DD MMM HH:MM:SS.mmm * Redis <version> starting
[PID] DD MMM HH:MM:SS.mmm * Configuration loaded
[PID] DD MMM HH:MM:SS.mmm * Server initialized
[PID] DD MMM HH:MM:SS.mmm * Ready to accept connections

# Exemple
[1234] 15 Jan 10:30:00.123 * Redis 7.2.3 64 bit starting
[1234] 15 Jan 10:30:00.234 * Configuration loaded
[1234] 15 Jan 10:30:00.456 * Server initialized
[1234] 15 Jan 10:30:00.789 * Ready to accept connections
```

**Analyser** : Temps entre "starting" et "Ready" (should be < 10s)

#### Arrêt gracieux
```
[PID] DD MMM HH:MM:SS.mmm # User requested shutdown...
[PID] DD MMM HH:MM:SS.mmm * Saving the final RDB snapshot before exiting.
[PID] DD MMM HH:MM:SS.mmm * DB saved on disk
[PID] DD MMM HH:MM:SS.mmm # Redis is now ready to exit, bye bye...
```

#### Crash/Arrêt brutal
```
# Pas de log "bye bye"
# Dernière ligne abrupte
# Vérifier dmesg pour OOM killer
```

### 2.2 Événements de mémoire

#### Approche maxmemory
```
[PID] DD MMM HH:MM:SS.mmm # WARNING: memory usage is close to maxmemory
```

**Action** : Augmenter maxmemory ou activer éviction

#### OOM (Out of Memory)
```
[PID] DD MMM HH:MM:SS.mmm # Can't save in background: fork: Cannot allocate memory
[PID] DD MMM HH:MM:SS.mmm # MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commands that may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails
```

**Impact** : Redis passe en mode read-only

**Recherche dans logs** :
```bash
grep -i "cannot allocate memory\|MISCONF" /var/log/redis/redis.log
```

### 2.3 Événements de persistence

#### RDB Snapshot
```
# Début
[PID] DD MMM HH:MM:SS.mmm * Background saving started by pid 5678

# Succès
[PID] DD MMM HH:MM:SS.mmm * Background saving terminated with success

# Échec
[PID] DD MMM HH:MM:SS.mmm # Background saving error
```

#### AOF Rewrite
```
[PID] DD MMM HH:MM:SS.mmm * Background AOF rewrite started by pid 5678
[PID] DD MMM HH:MM:SS.mmm * Background AOF rewrite terminated with success
[PID] DD MMM HH:MM:SS.mmm * Residual parent diff successfully flushed to the rewritten AOF
[PID] DD MMM HH:MM:SS.mmm * Background AOF rewrite finished successfully
```

**Latence de fork** :
```
[PID] DD MMM HH:MM:SS.mmm * Background AOF rewrite started by pid 5678
[PID] DD MMM HH:MM:SS.mmm * Fork took 2345 milliseconds to complete
```

**Analyser** : Fork > 1000ms = problème (THP, RAM insuffisante)

### 2.4 Événements de réplication

#### Replica se connecte au master
```
# Sur le master
[PID] DD MMM HH:MM:SS.mmm * Replica 10.0.1.10:6379 asks for synchronization
[PID] DD MMM HH:MM:SS.mmm * Full resync requested by replica 10.0.1.10:6379
[PID] DD MMM HH:MM:SS.mmm * Starting BGSAVE for SYNC with target: disk
[PID] DD MMM HH:MM:SS.mmm * Background saving started by pid 5678
[PID] DD MMM HH:MM:SS.mmm * Background saving terminated with success
[PID] DD MMM HH:MM:SS.mmm * Synchronization with replica 10.0.1.10:6379 succeeded
```

#### Replica perd la connexion
```
# Sur le master
[PID] DD MMM HH:MM:SS.mmm - Connection with replica 10.0.1.10:6379 lost.

# Sur le replica
[PID] DD MMM HH:MM:SS.mmm # Connection with master lost.
[PID] DD MMM HH:MM:SS.mmm * Connecting to MASTER 10.0.1.5:6379
[PID] DD MMM HH:MM:SS.mmm * MASTER <-> REPLICA sync started
```

### 2.5 Événements de sécurité

#### Tentatives d'authentification
```
# Succès (loglevel debug uniquement)
[PID] DD MMM HH:MM:SS.mmm - Accepted 10.0.1.50:54321

# Échec d'authentification
[PID] DD MMM HH:MM:SS.mmm - Client sent AUTH, but no password is set
# Ou
[PID] DD MMM HH:MM:SS.mmm # Warning: AUTH failed for 10.0.1.99:12345
```

#### Violations ACL (Redis 6+)
```
[PID] DD MMM HH:MM:SS.mmm # ACL violation: User 'readonly_user' has no permissions to run the 'set' command
```

**Recherche attaques** :
```bash
grep -i "AUTH failed\|ACL violation" /var/log/redis/redis.log | tail -100
```

### 2.6 Événements de configuration

#### Changement de config à chaud
```
[PID] DD MMM HH:MM:SS.mmm * CONFIG SET 'maxmemory' '8589934592' by client 10.0.1.50:54321
```

**Audit** : Tracer qui change quoi
```bash
grep "CONFIG SET" /var/log/redis/redis.log
```

### 2.7 Warnings importants

#### Transparent Huge Pages
```
[PID] DD MMM HH:MM:SS.mmm # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root
```

**Action immédiate** : Désactiver THP

#### Overcommit memory
```
[PID] DD MMM HH:MM:SS.mmm # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf
```

#### Limits trop bas
```
[PID] DD MMM HH:MM:SS.mmm # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
```

**Configuration** :
```bash
sudo sysctl -w net.core.somaxconn=65535
```

## 3. Centralisation des Logs

### 3.1 Architecture de centralisation

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Redis 1    │  │  Redis 2    │  │  Redis 3    │
│  (logs)     │  │  (logs)     │  │  (logs)     │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
                ┌───────▼────────┐
                │  Log Shipper   │
                │ (Filebeat/     │
                │  Fluentd)      │
                └───────┬────────┘
                        │
                ┌───────▼────────┐
                │  Log Agregator │
                │ (Logstash/     │
                │  Fluentd)      │
                └───────┬────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌───────▼───────┐  ┌────▼───────┐ ┌─────▼──────┐
│ Elasticsearch │  │   Loki     │ │   S3/GCS   │
│ (Recherche)   │  │(Time-series│ │  (Archive) │
└───────┬───────┘  └────┬───────┘ └────────────┘
        │               │
        └───────┬───────┘
                │
        ┌───────▼────────┐
        │    Kibana/     │
        │    Grafana     │
        │ (Visualisation)│
        └────────────────┘
```

### 3.2 Stack ELK (Elasticsearch, Logstash, Kibana)

#### Filebeat configuration
**filebeat.yml** :
```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/redis/redis-server.log

    # Parser les logs Redis
    multiline.pattern: '^\[\d+\]'
    multiline.negate: true
    multiline.match: after

    # Tags pour filtrage
    tags: ["redis", "production"]

    # Champs custom
    fields:
      env: production
      service: redis
      instance: redis-master-1
      region: us-east-1
    fields_under_root: true

# Output vers Logstash
output.logstash:
  hosts: ["logstash:5044"]

  # Ou directement vers Elasticsearch
  # output.elasticsearch:
  #   hosts: ["elasticsearch:9200"]
  #   index: "redis-logs-%{+yyyy.MM.dd}"
```

#### Logstash configuration
**logstash.conf** :
```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  if "redis" in [tags] {
    # Parser les logs Redis
    grok {
      match => {
        "message" => "\[%{NUMBER:pid}\] %{MONTHDAY:day} %{MONTH:month} %{TIME:time} %{LOGLEVEL:loglevel} %{GREEDYDATA:msg}"
      }
    }

    # Parser timestamp
    date {
      match => ["day month time", "dd MMM HH:mm:ss.SSS"]
      target => "@timestamp"
    }

    # Extraire événements spécifiques
    if [msg] =~ /Background saving/ {
      mutate {
        add_field => { "event_type" => "bgsave" }
      }
    }

    if [msg] =~ /MASTER.*REPLICA/ {
      mutate {
        add_field => { "event_type" => "replication" }
      }
    }

    if [msg] =~ /AUTH failed|ACL violation/ {
      mutate {
        add_field => { "event_type" => "security" }
        add_tag => ["security_alert"]
      }
    }

    # Extraire IP client des logs
    if [msg] =~ /client/ {
      grok {
        match => {
          "msg" => "%{IP:client_ip}:%{NUMBER:client_port}"
        }
      }

      # GeoIP
      geoip {
        source => "client_ip"
        target => "geoip"
      }
    }
  }
}

output {
  if "redis" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "redis-logs-%{+YYYY.MM.dd}"

      # Template pour mapping
      template_name => "redis-logs"
      template => "/etc/logstash/templates/redis-logs.json"
    }

    # Alerting sur événements sécurité
    if "security_alert" in [tags] {
      http {
        url => "https://alerting-webhook.company.com/security"
        http_method => "post"
        format => "json"
        content_type => "application/json"
        message => '{"severity":"high","source":"redis","message":"%{msg}","client":"%{client_ip}"}'
      }
    }
  }
}
```

#### Elasticsearch index template
**redis-logs.json** :
```json
{
  "index_patterns": ["redis-logs-*"],
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.lifecycle.name": "redis-logs-policy",
    "index.lifecycle.rollover_alias": "redis-logs"
  },
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "pid": {"type": "integer"},
      "loglevel": {"type": "keyword"},
      "msg": {"type": "text"},
      "event_type": {"type": "keyword"},
      "instance": {"type": "keyword"},
      "env": {"type": "keyword"},
      "region": {"type": "keyword"},
      "client_ip": {"type": "ip"},
      "client_port": {"type": "integer"},
      "geoip": {
        "properties": {
          "location": {"type": "geo_point"}
        }
      }
    }
  }
}
```

#### Kibana dashboard queries

**Requêtes utiles** :
```
# Tous les warnings/errors
loglevel:(WARNING OR ERROR)

# Événements de réplication
event_type:replication

# Tentatives d'authentification échouées
msg:"AUTH failed"

# Fork latency > 1s
msg:"Fork took" AND msg:/[1-9]\d{3,} milliseconds/

# Logs par instance
instance:"redis-master-1"

# Logs dernière heure
@timestamp:[now-1h TO now]
```

### 3.3 Stack Grafana Loki

**Promtail configuration** :
```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: redis
    static_configs:
      - targets:
          - localhost
        labels:
          job: redis
          env: production
          instance: redis-master-1
          __path__: /var/log/redis/redis-server.log

    # Pipeline de parsing
    pipeline_stages:
      # Parse timestamp et niveau
      - regex:
          expression: '^\[(?P<pid>\d+)\] (?P<day>\d+) (?P<month>\w+) (?P<time>[\d:.]+) (?P<level>[*#-]) (?P<msg>.*)$'

      # Extraire labels
      - labels:
          level:
          pid:

      # Convertir symboles en niveaux
      - template:
          source: level_name
          template: '{{ if eq .level "*" }}INFO{{ else if eq .level "#" }}WARNING{{ else }}DEBUG{{ end }}'

      - labels:
          level_name:

      # Timestamp
      - timestamp:
          source: time
          format: "02 Jan 15:04:05.000"
```

**Requêtes LogQL** :
```logql
# Tous les logs Redis
{job="redis"}

# Warnings et errors uniquement
{job="redis"} |~ "# WARNING|# ERROR"

# Fork latency
{job="redis"} |~ "Fork took"

# Événements de réplication
{job="redis"} |~ "MASTER.*REPLICA|Synchronization"

# Rate de logs warnings sur 5m
rate({job="redis", level_name="WARNING"}[5m])

# Top 10 messages par fréquence
topk(10, count_over_time({job="redis"}[1h]))
```

### 3.4 Solutions cloud natives

#### AWS CloudWatch Logs
```bash
# Installation agent
sudo yum install amazon-cloudwatch-agent

# Configuration
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/redis/redis-server.log",
            "log_group_name": "/aws/redis/production",
            "log_stream_name": "{instance_id}-redis",
            "timestamp_format": "%d %b %H:%M:%S"
          }
        ]
      }
    }
  }
}
```

#### GCP Cloud Logging
```yaml
# Fluentd config pour GCP
<source>
  @type tail
  path /var/log/redis/redis-server.log
  pos_file /var/log/td-agent/redis.pos
  tag redis
  <parse>
    @type regexp
    expression /^\[(?<pid>\d+)\] (?<time>\d+ \w+ [\d:.]+) (?<level>[*#-]) (?<message>.*)$/
    time_format %d %b %H:%M:%S.%L
  </parse>
</source>

<match redis>
  @type google_cloud
  use_metadata_service true

  # Labels
  labels {
    "service": "redis",
    "env": "production"
  }
</match>
```

## 4. Audit Trail et Conformité

### 4.1 Exigences de conformité

**Réglementations communes** :

| Norme | Exigence Logging | Redis Impact |
|-------|-----------------|--------------|
| **RGPD** | Tracer accès données personnelles | Logger GET/SET sur clés users |
| **PCI-DSS** | Tracer accès données cartes | Logger toutes commandes sensibles |
| **SOX** | Audit trail modifications données financières | Logger writes sur clés financières |
| **HIPAA** | Tracer accès données santé | Logger accès PHI |
| **SOC 2** | Monitoring accès et changements | Logs + ACLs + alerting |

### 4.2 Audit avec ACLs (Redis 6+)

**Configuration ACL avec logging** :
```bash
# Créer utilisateurs avec permissions limitées
redis-cli ACL SETUSER audit_reader \
  on \
  >password \
  ~* \
  +@read \
  -@write \
  -@dangerous

redis-cli ACL SETUSER app_writer \
  on \
  >password \
  ~app:* \
  +@read \
  +@write \
  -@dangerous \
  -FLUSHDB \
  -FLUSHALL \
  -KEYS

# User admin complet
redis-cli ACL SETUSER admin \
  on \
  >admin_password \
  ~* \
  +@all
```

**Logs ACL** :
```
# Tentative d'accès refusé
[PID] DD MMM HH:MM:SS.mmm # ACL violation: User 'audit_reader' has no permissions to run the 'set' command

# Commande sensible exécutée
[PID] DD MMM HH:MM:SS.mmm # User 'admin' executed 'FLUSHDB' from client 10.0.1.50:54321
```

### 4.3 Module Redis Audit

**Redis Audit Module** (commercial ou custom) :
```c
// Pseudo-code d'un module d'audit
RedisModuleCommandFilter *filter = RedisModule_RegisterCommandFilter(ctx, AuditFilter, REDISMODULE_CMDFILTER_NOSELF);

int AuditFilter(RedisModuleCommandFilterCtx *filter) {
    // Capturer toutes les commandes
    const char *cmd = RedisModule_CommandFilterArgGet(filter, 0);
    const char *client_ip = RedisModule_GetClientInfoById(ctx, client_id)->addr;

    // Logger dans un fichier ou base externe
    AuditLog("%s - User: %s, IP: %s, Command: %s, Args: %s\n",
             timestamp, username, client_ip, cmd, args);

    return REDISMODULE_OK;
}
```

**Format de log audit** :
```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "user": "app_writer",
  "client_ip": "10.0.1.50",
  "client_port": 54321,
  "command": "SET",
  "args": ["user:12345:email", "john@example.com"],
  "database": 0,
  "result": "OK",
  "duration_ms": 0.12
}
```

### 4.4 Command Logging via Proxy

**Twemproxy avec logging** :
```yaml
# nutcracker.yml
redis_pool:
  listen: 127.0.0.1:6380
  hash: fnv1a_64
  distribution: ketama
  auto_eject_hosts: true
  redis: true

  # Log toutes les commandes
  log_level: 11  # LOG_DEBUG

  servers:
    - 10.0.1.5:6379:1
    - 10.0.1.6:6379:1
```

**Alternative : Redis Proxy custom** :
```python
# redis_audit_proxy.py
import redis
import logging
import json
from datetime import datetime

logging.basicConfig(filename='redis_audit.log', level=logging.INFO)

class AuditRedisProxy:
    def __init__(self, redis_host='localhost', redis_port=6379):
        self.redis = redis.Redis(host=redis_host, port=redis_port)

    def execute_command(self, user, client_ip, command, *args):
        audit_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "user": user,
            "client_ip": client_ip,
            "command": command,
            "args": args,
        }

        try:
            start = datetime.utcnow()
            result = self.redis.execute_command(command, *args)
            duration = (datetime.utcnow() - start).total_seconds() * 1000

            audit_entry["result"] = "OK"
            audit_entry["duration_ms"] = duration

            # Log seulement commandes sensibles
            if command.upper() in ['SET', 'DEL', 'FLUSHDB', 'FLUSHALL', 'CONFIG']:
                logging.info(json.dumps(audit_entry))

            return result

        except Exception as e:
            audit_entry["result"] = "ERROR"
            audit_entry["error"] = str(e)
            logging.error(json.dumps(audit_entry))
            raise
```

### 4.5 Rétention et archivage

**Politique de rétention recommandée** :

| Type de log | Hot storage | Warm storage | Cold storage | Total |
|-------------|-------------|--------------|--------------|-------|
| **Production** | 7 jours (ES/Loki) | 30 jours (S3 Standard) | 1-7 ans (S3 Glacier) | Conformité |
| **Staging** | 3 jours | 7 jours | - | 10 jours |
| **Dev** | 1 jour | - | - | 1 jour |

**Elasticsearch ILM Policy** :
```json
{
  "policy": "redis-logs-policy",
  "phases": {
    "hot": {
      "min_age": "0ms",
      "actions": {
        "rollover": {
          "max_size": "50GB",
          "max_age": "1d"
        },
        "set_priority": {
          "priority": 100
        }
      }
    },
    "warm": {
      "min_age": "7d",
      "actions": {
        "allocate": {
          "number_of_replicas": 1
        },
        "set_priority": {
          "priority": 50
        }
      }
    },
    "cold": {
      "min_age": "30d",
      "actions": {
        "allocate": {
          "number_of_replicas": 0
        },
        "freeze": {},
        "set_priority": {
          "priority": 0
        }
      }
    },
    "delete": {
      "min_age": "90d",
      "actions": {
        "delete": {}
      }
    }
  }
}
```

**Script d'archivage vers S3** :
```bash
#!/bin/bash
# archive-redis-logs.sh

DATE=$(date -d '7 days ago' +%Y-%m-%d)
LOG_FILE="/var/log/redis/redis-server.log-${DATE}.gz"

if [ -f "$LOG_FILE" ]; then
    # Upload vers S3
    aws s3 cp "$LOG_FILE" \
        s3://company-logs-archive/redis/$(date +%Y)/$(date +%m)/ \
        --storage-class GLACIER

    # Supprimer local après confirmation
    if [ $? -eq 0 ]; then
        rm "$LOG_FILE"
        echo "Archived and deleted: $LOG_FILE"
    fi
fi
```

## 5. Analyse et Troubleshooting

### 5.1 Patterns de recherche courants

#### Identifier redémarrages
```bash
# Comptage des redémarrages
grep "Redis.*starting" /var/log/redis/redis.log | wc -l

# Derniers redémarrages avec timestamps
grep "Redis.*starting" /var/log/redis/redis.log | tail -10
```

#### Fork latency analysis
```bash
# Extraire toutes les latences de fork
grep "Fork took" /var/log/redis/redis.log | \
  awk '{print $(NF-2)}' | \
  sort -n | \
  awk '{sum+=$1; count++} END {print "Avg:", sum/count, "ms", "Max:", $NF, "ms"}'
```

#### Événements de réplication
```bash
# Timeline de réplication
grep -E "MASTER.*REPLICA|Synchronization|replica.*lost" /var/log/redis/redis.log | \
  tail -50
```

#### Erreurs par période
```bash
# Erreurs par heure (dernières 24h)
grep "# " /var/log/redis/redis.log | \
  awk '{print $2, $3, $4}' | \
  cut -d: -f1 | \
  sort | uniq -c | \
  sort -rn
```

#### Top erreurs
```bash
# Fréquence des messages d'erreur
grep "# " /var/log/redis/redis.log | \
  awk -F'# ' '{print $2}' | \
  cut -d' ' -f1-10 | \
  sort | uniq -c | \
  sort -rn | \
  head -20
```

### 5.2 Corrélation Logs + Métriques

**Workflow d'investigation** :
```
1. Alerte Prometheus → Latence spike à 10:30
   ↓
2. Dashboard Grafana → Memory spike simultané
   ↓
3. Logs (10:29-10:31) → Recherche événements
   ↓
4. Trouve: "Fork took 2345ms" à 10:30
   ↓
5. Root cause: BGSAVE pendant pic de trafic
```

**Annotation Grafana depuis logs** :
```python
# Script pour créer annotations Grafana depuis logs
import re
import requests
from datetime import datetime

GRAFANA_URL = "http://grafana:3000"
API_KEY = "your-api-key"

def parse_redis_log(log_file):
    with open(log_file, 'r') as f:
        for line in f:
            # Détecter événements importants
            if "Fork took" in line:
                match = re.search(r'\[(\d+)\] (\d+ \w+ [\d:.]+).*Fork took (\d+) milliseconds', line)
                if match:
                    timestamp = datetime.strptime(match.group(2), "%d %b %H:%M:%S.%f")
                    duration = int(match.group(3))

                    # Créer annotation Grafana
                    annotation = {
                        "dashboardId": 1,
                        "time": int(timestamp.timestamp() * 1000),
                        "tags": ["redis", "fork", "latency"],
                        "text": f"Fork latency: {duration}ms"
                    }

                    requests.post(
                        f"{GRAFANA_URL}/api/annotations",
                        json=annotation,
                        headers={"Authorization": f"Bearer {API_KEY}"}
                    )

parse_redis_log('/var/log/redis/redis.log')
```

### 5.3 Alerting sur patterns de logs

**Prometheus avec mtail** :
```perl
# redis.mtail - Exposer métriques depuis logs

counter redis_errors_total by level, msg

/^\[(?P<pid>\d+)\] (?P<timestamp>\d+ \w+ [\d:.]+) # (?P<level>\w+) (?P<msg>.*)$/ {
  redis_errors_total[$level][$msg]++
}

# Compter forks longs
counter redis_slow_forks_total
gauge redis_last_fork_duration_ms

/Fork took (?P<duration>\d+) milliseconds/ {
  redis_last_fork_duration_ms = $duration

  $duration > 1000 {
    redis_slow_forks_total++
  }
}
```

**Alerte Prometheus** :
```yaml
- alert: RedisSlowForksIncreasing
  expr: rate(redis_slow_forks_total[1h]) > 0.1
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Forks lents fréquents sur Redis"
    description: "{{ $value }} forks > 1s par heure"
```

## 6. Sécurité et Protection des Logs

### 6.1 Données sensibles dans les logs

**⚠️ Problème** : Les logs peuvent contenir des données sensibles

**Exemple** :
```bash
# Slowlog peut exposer des données
redis-cli SLOWLOG GET 1
# 1) 1) (integer) 127
#    2) (integer) 1702310000
#    3) (integer) 234567
#    4) 1) "SET"
#       2) "user:12345:email"
#       3) "john.doe@example.com"  ← Donnée personnelle !
```

**Solutions** :

#### 1. Redaction dans Logstash
```ruby
filter {
  # Masquer emails
  mutate {
    gsub => [
      "msg", "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}", "[REDACTED_EMAIL]"
    ]
  }

  # Masquer numéros de carte
  mutate {
    gsub => [
      "msg", "\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b", "[REDACTED_CC]"
    ]
  }

  # Masquer IPs privées (optionnel)
  mutate {
    gsub => [
      "msg", "\b10\.\d{1,3}\.\d{1,3}\.\d{1,3}\b", "[REDACTED_IP]"
    ]
  }
}
```

#### 2. Hashing des valeurs sensibles
```python
import hashlib

def hash_sensitive_data(value):
    """Hash one-way pour audit trail sans exposer données"""
    return hashlib.sha256(value.encode()).hexdigest()[:16]

# Exemple
email = "john@example.com"
audit_log = {
    "command": "GET",
    "key": f"user:email:{hash_sensitive_data(email)}"
}
# Au lieu de: "key": "user:email:john@example.com"
```

#### 3. Chiffrement des logs
```bash
# Chiffrer logs avant archivage
gpg --encrypt --recipient logs@company.com /var/log/redis/redis.log

# Ou avec AWS KMS
aws kms encrypt \
  --key-id alias/logs-encryption-key \
  --plaintext fileb:///var/log/redis/redis.log \
  --output text \
  --query CiphertextBlob | base64 -d > redis.log.encrypted
```

### 6.2 Contrôle d'accès aux logs

**Permissions fichiers** :
```bash
# Logs lisibles uniquement par redis et root
sudo chown redis:redis /var/log/redis/redis-server.log
sudo chmod 640 /var/log/redis/redis-server.log
```

**Elasticsearch security** :
```json
PUT _security/role/redis_logs_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["redis-logs-*"],
      "privileges": ["read", "view_index_metadata"],
      "query": {
        "term": {
          "env": "production"
        }
      }
    }
  ]
}
```

### 6.3 Tamper-proof logging

**Write-once storage (WORM)** :
- S3 Object Lock (compliance mode)
- Azure Immutable Blob Storage
- GCS retention policies

**S3 Object Lock** :
```bash
# Activer Object Lock sur le bucket
aws s3api put-object-lock-configuration \
  --bucket redis-audit-logs \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 2555
      }
    }
  }'
```

**Blockchain-based audit trail** (advanced) :
```python
# Pseudo-code : Hash chain pour intégrité
import hashlib
import json

class AuditChain:
    def __init__(self):
        self.chain = []
        self.current_hash = "0"

    def add_entry(self, log_entry):
        entry = {
            "previous_hash": self.current_hash,
            "data": log_entry,
            "timestamp": time.time()
        }
        entry_hash = hashlib.sha256(json.dumps(entry).encode()).hexdigest()
        entry["hash"] = entry_hash

        self.chain.append(entry)
        self.current_hash = entry_hash

    def verify_integrity(self):
        for i, entry in enumerate(self.chain):
            if i > 0:
                if entry["previous_hash"] != self.chain[i-1]["hash"]:
                    return False, f"Tamper detected at entry {i}"
        return True, "Chain intact"
```

## 7. Best Practices et Checklist

### 7.1 Configuration de base

```bash
# ✅ Configuration production recommandée
loglevel notice
logfile /var/log/redis/redis-server.log

# ✅ Ou syslog pour centralisation
# syslog-enabled yes
# syslog-ident redis-prod-1
# syslog-facility local0

# ✅ Rotation configurée
# Voir /etc/logrotate.d/redis

# ✅ Slowlog activé
slowlog-log-slower-than 10000
slowlog-max-len 128
```

### 7.2 Centralisation

- ✅ Logs envoyés vers système central (ELK, Loki, CloudWatch)
- ✅ Parsing structuré (Logstash, Promtail)
- ✅ Indexation avec labels appropriés (env, region, instance)
- ✅ Dashboards de visualisation créés
- ✅ Alerting sur patterns critiques configuré

### 7.3 Conformité

- ✅ Politique de rétention définie et appliquée
- ✅ Données sensibles redacted ou hashed
- ✅ Contrôle d'accès strict (RBAC)
- ✅ Audit trail activé pour commandes sensibles
- ✅ Logs archivés en immutable storage si requis
- ✅ Documentation conformité à jour

### 7.4 Opérations

- ✅ Monitoring du système de logging lui-même
- ✅ Alertes sur échecs d'envoi de logs
- ✅ Capacité disque surveillée
- ✅ Scripts de recherche/analyse documentés
- ✅ Runbooks incluent analyse de logs
- ✅ Corrélation logs + métriques fonctionnelle

### 7.5 Checklist complète

**Configuration** :
- [ ] Loglevel approprié (notice en prod)
- [ ] Destination configurée (fichier/syslog/stdout)
- [ ] Rotation mise en place
- [ ] Slowlog activé et configuré
- [ ] ACLs configurées (Redis 6+)

**Centralisation** :
- [ ] Log shipper installé (Filebeat/Promtail)
- [ ] Parsing configuré
- [ ] Destination accessible (ES/Loki)
- [ ] Indexation testée
- [ ] Dashboard créé

**Sécurité** :
- [ ] Permissions fichiers correctes
- [ ] Données sensibles protégées
- [ ] Accès contrôlé (RBAC)
- [ ] Chiffrement si requis
- [ ] Audit trail activé

**Conformité** :
- [ ] Politique de rétention documentée
- [ ] Archivage automatisé
- [ ] Immutabilité configurée si requis
- [ ] Processus de purge défini
- [ ] Audits réguliers planifiés

**Opérations** :
- [ ] Monitoring logs pipeline
- [ ] Alertes configurées
- [ ] Scripts d'analyse disponibles
- [ ] Documentation à jour
- [ ] Tests réguliers

## 8. Troubleshooting Checklist

### Problème : Logs non visibles

```bash
# 1. Vérifier configuration
redis-cli CONFIG GET loglevel
redis-cli CONFIG GET logfile

# 2. Vérifier permissions
ls -la $(redis-cli CONFIG GET dir | tail -1)/../redis-server.log

# 3. Vérifier espace disque
df -h /var/log/redis

# 4. Vérifier rotation
cat /etc/logrotate.d/redis
```

### Problème : Logs trop volumineux

```bash
# 1. Réduire loglevel
redis-cli CONFIG SET loglevel warning

# 2. Augmenter fréquence rotation
# Modifier /etc/logrotate.d/redis : daily → hourly

# 3. Compresser plus agressivement
# Ajouter dans logrotate: compress, delaycompress
```

### Problème : Logs ne sont pas centralisés

```bash
# 1. Vérifier log shipper
systemctl status filebeat

# 2. Vérifier connectivity
curl -X GET "elasticsearch:9200/_cluster/health?pretty"

# 3. Vérifier parsing
journalctl -u filebeat -n 100

# 4. Test manuel
filebeat test output
```

## Conclusion

Les logs Redis sont essentiels pour :

1. **Troubleshooting** : Comprendre les incidents post-mortem
2. **Sécurité** : Audit trail et détection d'intrusion
3. **Conformité** : Répondre aux exigences réglementaires
4. **Optimisation** : Identifier patterns et bottlenecks

**Points clés** :
- Centraliser les logs pour analyse cross-instances
- Structurer et parser pour faciliter recherche
- Protéger les données sensibles
- Corréler avec les métriques
- Maintenir conformité réglementaire

Un système de logging bien conçu est invisible en temps normal mais invaluable lors des incidents.

---

**Fin du Module 13 : Monitoring et Observabilité**

**Récapitulatif complet** :
- 13.1 : Redis INFO - Toutes les métriques décryptées
- 13.2 : Métriques clés (Hit ratio, Fragmentation, Évictions)
- 13.3 : Redis Exporter et Prometheus
- 13.4 : Dashboards Grafana
- 13.5 : Latency Doctor et monitoring latence
- 13.6 : Alerting stratégique
- 13.7 : Logs et audit trail

Vous disposez maintenant d'une stack de monitoring complète et production-ready pour Redis !

⏭️ [Performance et Troubleshooting](/14-performance-troubleshooting/README.md)
