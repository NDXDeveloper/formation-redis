🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.4 Dashboards Grafana pour Redis

## Introduction

Un dashboard Grafana efficace transforme des centaines de métriques brutes en insights actionnables. Cette section couvre la conception, la configuration et l'optimisation de dashboards Redis production-ready.

### Philosophie d'un bon dashboard

**Les 4 règles d'or** :
1. **Hiérarchie** : Du général au spécifique (top-down)
2. **Actionnable** : Chaque panel doit suggérer une action
3. **Contextualisé** : Comparaison historique et seuils
4. **Performant** : Requêtes optimisées, pas de surcharge

**Anti-patterns à éviter** :
- ❌ Trop de panels (> 20) → Confusion
- ❌ Métriques vanity (inutiles) → Bruit
- ❌ Pas de contexte temporel → Impossible d'identifier les anomalies
- ❌ Pas de seuils → Interprétation difficile

## 1. Architecture de dashboards Redis

### 1.1 Hiérarchie recommandée

```
┌─────────────────────────────────────────────────┐
│  Level 0 : OVERVIEW (Landing Page)              │
│  Vue multi-instances, métriques critiques       │
│  ↓ Drill-down si anomalie                       │
└─────────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
┌───────▼─────┐ ┌───▼───────┐ ┌─▼─────────┐
│ Level 1:    │ │ Level 1:  │ │ Level 1:  │
│ PERFORMANCE │ │ MEMORY    │ │ CLUSTER   │
│ (latence,   │ │ (frag,    │ │ (slots,   │
│  throughput)│ │  usage)   │ │  nodes)   │
└─────────────┘ └───────────┘ └───────────┘
        │           │           │
        ▼           ▼           ▼
┌─────────────────────────────────────────────────┐
│  Level 2 : DETAILED (Per-instance deep dive)    │
│  Analyse approfondie d'une instance spécifique  │
└─────────────────────────────────────────────────┘
```

### 1.2 Dashboards par persona

| Persona | Dashboard | Métriques clés | Refresh |
|---------|-----------|----------------|---------|
| **SRE/Ops** | Overview + Alerting | Uptime, Mémoire, Évictions | 30s |
| **Dev** | Performance | Hit ratio, Latency, Commands | 1m |
| **Architect** | Capacity Planning | Trends, Growth, Projections | 5m |
| **Business** | Business Metrics | Custom metrics, KPIs | 5m |

## 2. Dashboards Communautaires

### 2.1 Top dashboards recommandés

#### Dashboard 1 : Redis Dashboard for Prometheus Redis Exporter (ID: 763)

**URL** : https://grafana.com/grafana/dashboards/763

**Caractéristiques** :
- ⭐ 4.8/5 (1000+ téléchargements)
- Coverage complète : 15+ panels
- Multi-instance support
- Alerting intégré

**Panels inclus** :
- Total items per DB
- Commands executed/sec
- Hits/Misses per sec
- Total memory usage
- Connected clients
- Network I/O
- Replication lag

**Installation** :
```bash
# Via CLI
grafana-cli plugins install redis-datasource

# Import dashboard
curl -X POST http://admin:admin@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"dashboard": {...}, "overwrite": true}'
```

#### Dashboard 2 : Redis Overview (ID: 11835)

**URL** : https://grafana.com/grafana/dashboards/11835

**Spécificités** :
- Vue consolidée multi-instances
- Focus sur la santé globale
- Comparaisons master vs replicas
- Idéal pour architectures HA

#### Dashboard 3 : Redis Cluster Dashboard (ID: 11692)

**URL** : https://grafana.com/grafana/dashboards/11692

**Pour** : Redis Cluster uniquement
**Panels spécifiques** :
- Slots distribution
- Node states
- Cluster-wide operations
- Resharding progress

### 2.2 Import de dashboards communautaires

**Méthode 1 : UI Grafana**
```
1. Grafana UI → Dashboards → Import
2. Saisir l'ID (763, 11835, etc.)
3. Sélectionner le datasource Prometheus
4. Import
```

**Méthode 2 : API**
```bash
DASHBOARD_ID=763
GRAFANA_URL="http://localhost:3000"
API_KEY="your-api-key"

curl -X POST "${GRAFANA_URL}/api/dashboards/import" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "dashboard": {
      "id": null,
      "uid": null,
      "title": "Redis Overview"
    },
    "overwrite": false,
    "inputs": [{
      "name": "DS_PROMETHEUS",
      "type": "datasource",
      "pluginId": "prometheus",
      "value": "Prometheus"
    }]
  }'
```

**Méthode 3 : Provisioning (GitOps)**
```yaml
# /etc/grafana/provisioning/dashboards/redis.yaml
apiVersion: 1

providers:
  - name: 'Redis Dashboards'
    orgId: 1
    folder: 'Redis'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards/redis
```

### 2.3 Limites des dashboards communautaires

**Pourquoi créer des dashboards custom ?**

1. **Métriques spécifiques** : Dashboards génériques ne couvrent pas vos métriques business
2. **Labels custom** : Vos labels (env, region, etc.) nécessitent des adaptations
3. **Use case unique** : Cache vs Queue vs Session Store → besoins différents
4. **Branding** : Logos, couleurs, organisation custom

## 3. Conception de Dashboard Custom

### 3.1 Dashboard Overview Multi-instances

**Structure** :
```
┌────────────────────────────────────────────────────────────────┐
│ Row 1 : Status Global (Single Stats)                           │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│ │Instances │ │ Uptime   │ │Avg Memory│ │Avg Hit   │            │
│ │  UP      │ │  Avg     │ │  Usage   │ │  Ratio   │            │
│ │   12     │ │  45d     │ │   67%    │ │   92%    │            │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘            │
└────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────┐
│ Row 2 : Memory & Performance (Graphs)                          │
│ ┌─────────────────────────┐ ┌─────────────────────────┐        │
│ │ Memory Usage by Instance│ │ Ops/sec by Instance     │        │
│ │                         │ │                         │        │
│ │   [Time Series Graph]   │ │   [Time Series Graph]   │        │
│ └─────────────────────────┘ └─────────────────────────┘        │
└────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────┐
│ Row 3 : Health Indicators (Gauges + Table)                     │
│ ┌──────────┐ ┌──────────┐ ┌─────────────────────────┐          │
│ │Fragm Avg │ │Evictions │ │ Instances Table         │          │
│ │  Gauge   │ │  Gauge   │ │ (Status, Role, Memory)  │          │
│ └──────────┘ └──────────┘ └─────────────────────────┘          │
└────────────────────────────────────────────────────────────────┘
```

**JSON Dashboard** (extrait) :
```json
{
  "dashboard": {
    "title": "Redis Overview - Production",
    "tags": ["redis", "overview", "production"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-6h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Instances UP",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "count(redis_up == 1)",
            "legendFormat": "UP",
            "refId": "A"
          }
        ],
        "options": {
          "colorMode": "value",
          "graphMode": "none",
          "orientation": "auto"
        },
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"value": null, "color": "red"},
                {"value": 10, "color": "yellow"},
                {"value": 12, "color": "green"}
              ]
            }
          }
        }
      }
    ]
  }
}
```

### 3.2 Variables et Templating

**Variables essentielles** :

#### Variable 1 : Instance
```json
{
  "name": "instance",
  "type": "query",
  "datasource": "Prometheus",
  "query": "label_values(redis_up, instance)",
  "refresh": 1,
  "multi": true,
  "includeAll": true,
  "allValue": ".*"
}
```

#### Variable 2 : Environment
```json
{
  "name": "env",
  "type": "query",
  "datasource": "Prometheus",
  "query": "label_values(redis_up, env)",
  "refresh": 1,
  "multi": false,
  "includeAll": true
}
```

#### Variable 3 : Redis Role
```json
{
  "name": "redis_role",
  "type": "custom",
  "options": [
    {"text": "All", "value": ".*"},
    {"text": "Master", "value": "master"},
    {"text": "Replica", "value": "replica"}
  ],
  "current": {
    "text": "All",
    "value": ".*"
  }
}
```

#### Variable 4 : Time Range (Interval)
```json
{
  "name": "interval",
  "type": "interval",
  "auto": true,
  "auto_count": 30,
  "auto_min": "10s",
  "options": [
    {"text": "1m", "value": "1m"},
    {"text": "5m", "value": "5m"},
    {"text": "10m", "value": "10m"},
    {"text": "30m", "value": "30m"},
    {"text": "1h", "value": "1h"}
  ]
}
```

**Utilisation dans les requêtes** :
```promql
# Filtrage par instance et env
redis_memory_used_bytes{instance=~"$instance", env="$env"}

# Filtrage par rôle
redis_commands_processed_total{redis_role=~"$redis_role"}

# Intervalle dynamique
rate(redis_keyspace_hits_total{instance=~"$instance"}[$interval])
```

## 4. Panels Détaillés avec PromQL

### 4.1 Panel : Utilisation Mémoire (%)

**Type** : Time Series Graph
**Objectif** : Surveiller l'utilisation mémoire vs maxmemory

**Requête PromQL** :
```promql
(redis_memory_used_bytes{instance=~"$instance"} /
 redis_memory_maxmemory_bytes{instance=~"$instance"}) * 100
```

**Configuration JSON** :
```json
{
  "title": "Memory Usage (%)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "(redis_memory_used_bytes{instance=~\"$instance\"} / redis_memory_maxmemory_bytes{instance=~\"$instance\"}) * 100",
      "legendFormat": "{{instance}} - {{redis_role}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "min": 0,
      "max": 100,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"value": 0, "color": "green"},
          {"value": 70, "color": "yellow"},
          {"value": 85, "color": "orange"},
          {"value": 95, "color": "red"}
        ]
      }
    },
    "overrides": []
  },
  "options": {
    "tooltip": {"mode": "multi"},
    "legend": {"displayMode": "table", "placement": "bottom"}
  }
}
```

### 4.2 Panel : Hit Ratio (%)

**Type** : Time Series Graph
**Objectif** : Efficacité du cache

**Requête PromQL** :
```promql
100 * (
  rate(redis_keyspace_hits_total{instance=~"$instance"}[5m]) /
  (
    rate(redis_keyspace_hits_total{instance=~"$instance"}[5m]) +
    rate(redis_keyspace_misses_total{instance=~"$instance"}[5m])
  )
)
```

**Avec baseline (7 jours)** :
```promql
# Hit ratio actuel
100 * (
  rate(redis_keyspace_hits_total{instance=~"$instance"}[5m]) /
  (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
)

# Baseline 7 jours
avg_over_time(
  (
    100 * (
      rate(redis_keyspace_hits_total{instance=~"$instance"}[5m]) /
      (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
    )
  )[7d:5m]
)
```

**Configuration JSON avec baseline** :
```json
{
  "title": "Cache Hit Ratio (%)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "100 * (rate(redis_keyspace_hits_total{instance=~\"$instance\"}[5m]) / (rate(redis_keyspace_hits_total{instance=~\"$instance\"}[5m]) + rate(redis_keyspace_misses_total{instance=~\"$instance\"}[5m])))",
      "legendFormat": "{{instance}} - Current",
      "refId": "A"
    },
    {
      "expr": "avg_over_time((100 * (rate(redis_keyspace_hits_total{instance=~\"$instance\"}[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))))[7d:5m])",
      "legendFormat": "{{instance}} - 7d Baseline",
      "refId": "B"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "min": 0,
      "max": 100,
      "thresholds": {
        "steps": [
          {"value": 0, "color": "red"},
          {"value": 80, "color": "yellow"},
          {"value": 90, "color": "green"}
        ]
      }
    }
  }
}
```

### 4.3 Panel : Fragmentation Ratio

**Type** : Gauge
**Objectif** : Visualiser la fragmentation mémoire

**Requête PromQL** :
```promql
redis_mem_fragmentation_ratio{instance=~"$instance"}
```

**Configuration JSON** :
```json
{
  "title": "Memory Fragmentation Ratio",
  "type": "gauge",
  "targets": [
    {
      "expr": "redis_mem_fragmentation_ratio{instance=~\"$instance\"}",
      "legendFormat": "{{instance}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "min": 0,
      "max": 3,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"value": 0, "color": "red"},
          {"value": 1.0, "color": "green"},
          {"value": 1.3, "color": "yellow"},
          {"value": 1.5, "color": "orange"},
          {"value": 2.0, "color": "red"}
        ]
      },
      "mappings": [
        {
          "type": "range",
          "options": {
            "from": 0,
            "to": 1,
            "result": {"text": "SWAP!", "color": "dark-red"}
          }
        }
      ]
    }
  },
  "options": {
    "showThresholdLabels": true,
    "showThresholdMarkers": true
  }
}
```

### 4.4 Panel : Évictions Rate

**Type** : Time Series Graph
**Objectif** : Détecter les évictions (problème de capacité)

**Requête PromQL** :
```promql
rate(redis_evicted_keys_total{instance=~"$instance"}[5m])
```

**Avec alerte visuelle si > 0** :
```json
{
  "title": "Evictions Rate (keys/sec)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "rate(redis_evicted_keys_total{instance=~\"$instance\"}[5m])",
      "legendFormat": "{{instance}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "ops",
      "min": 0,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"value": 0, "color": "green"},
          {"value": 0.1, "color": "red"}
        ]
      }
    }
  },
  "alert": {
    "name": "Redis Evictions Detected",
    "conditions": [
      {
        "evaluator": {
          "type": "gt",
          "params": [0]
        },
        "operator": {"type": "and"},
        "query": {"params": ["A", "5m", "now"]},
        "reducer": {"type": "avg"}
      }
    ],
    "frequency": "1m",
    "handler": 1,
    "message": "Redis is evicting keys - Memory insufficient",
    "noDataState": "ok",
    "executionErrorState": "alerting"
  }
}
```

### 4.5 Panel : Throughput (Ops/sec)

**Type** : Time Series Graph
**Objectif** : Charge Redis en opérations par seconde

**Requête PromQL** :
```promql
# Ops/sec par instance
rate(redis_commands_processed_total{instance=~"$instance"}[1m])

# Ops/sec agrégé par rôle
sum(rate(redis_commands_processed_total{redis_role=~"$redis_role"}[1m])) by (redis_role)

# Ops/sec total
sum(rate(redis_commands_processed_total[1m]))
```

**Configuration JSON avec multiple targets** :
```json
{
  "title": "Redis Throughput",
  "type": "timeseries",
  "targets": [
    {
      "expr": "rate(redis_commands_processed_total{instance=~\"$instance\"}[1m])",
      "legendFormat": "{{instance}}",
      "refId": "A"
    },
    {
      "expr": "sum(rate(redis_commands_processed_total{instance=~\"$instance\"}[1m]))",
      "legendFormat": "Total",
      "refId": "B"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "ops",
      "custom": {
        "fillOpacity": 10,
        "lineWidth": 2
      }
    },
    "overrides": [
      {
        "matcher": {"id": "byName", "options": "Total"},
        "properties": [
          {
            "id": "custom.lineWidth",
            "value": 3
          },
          {
            "id": "color",
            "value": {"mode": "fixed", "fixedColor": "blue"}
          }
        ]
      }
    ]
  }
}
```

### 4.6 Panel : Latency (Percentiles)

**Type** : Time Series Graph
**Objectif** : Latence des commandes (P50, P95, P99)

**Requête PromQL** :
```promql
# P50 (médiane)
histogram_quantile(0.50,
  rate(redis_command_call_duration_seconds_bucket{instance=~"$instance"}[5m])
) * 1000

# P95
histogram_quantile(0.95,
  rate(redis_command_call_duration_seconds_bucket{instance=~"$instance"}[5m])
) * 1000

# P99
histogram_quantile(0.99,
  rate(redis_command_call_duration_seconds_bucket{instance=~"$instance"}[5m])
) * 1000
```

**Configuration JSON** :
```json
{
  "title": "Command Latency (ms)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "histogram_quantile(0.50, rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) * 1000",
      "legendFormat": "{{instance}} - P50",
      "refId": "A"
    },
    {
      "expr": "histogram_quantile(0.95, rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) * 1000",
      "legendFormat": "{{instance}} - P95",
      "refId": "B"
    },
    {
      "expr": "histogram_quantile(0.99, rate(redis_command_call_duration_seconds_bucket{instance=~\"$instance\"}[5m])) * 1000",
      "legendFormat": "{{instance}} - P99",
      "refId": "C"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "ms",
      "min": 0,
      "thresholds": {
        "steps": [
          {"value": 0, "color": "green"},
          {"value": 5, "color": "yellow"},
          {"value": 10, "color": "red"}
        ]
      }
    }
  }
}
```

### 4.7 Panel : Réplication Lag

**Type** : Time Series Graph
**Objectif** : Surveiller le retard des replicas

**Requête PromQL** :
```promql
# Lag en secondes (pour replicas)
redis_master_last_io_seconds_ago{instance=~"$instance", redis_role="replica"}

# Ou via repl_offset
(redis_master_repl_offset - redis_slave_repl_offset) /
avg_over_time(rate(redis_master_repl_offset[5m])[1m:])
```

**Configuration JSON** :
```json
{
  "title": "Replication Lag (seconds)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "redis_master_last_io_seconds_ago{instance=~\"$instance\", redis_role=\"replica\"}",
      "legendFormat": "{{instance}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "s",
      "min": 0,
      "thresholds": {
        "steps": [
          {"value": 0, "color": "green"},
          {"value": 5, "color": "yellow"},
          {"value": 10, "color": "red"}
        ]
      }
    }
  }
}
```

### 4.8 Panel : Connexions Clients

**Type** : Time Series Graph
**Objectif** : Surveiller le nombre de clients connectés

**Requête PromQL** :
```promql
redis_connected_clients{instance=~"$instance"}

# Avec pourcentage de maxclients
(redis_connected_clients{instance=~"$instance"} /
 redis_config_maxclients{instance=~"$instance"}) * 100
```

**Configuration JSON avec threshold dynamique** :
```json
{
  "title": "Connected Clients",
  "type": "timeseries",
  "targets": [
    {
      "expr": "redis_connected_clients{instance=~\"$instance\"}",
      "legendFormat": "{{instance}}",
      "refId": "A"
    },
    {
      "expr": "redis_config_maxclients{instance=~\"$instance\"} * 0.8",
      "legendFormat": "{{instance}} - 80% threshold",
      "refId": "B"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "min": 0
    },
    "overrides": [
      {
        "matcher": {"id": "byRegex", "options": "/threshold/"},
        "properties": [
          {
            "id": "custom.lineStyle",
            "value": {"fill": "dash"}
          },
          {
            "id": "color",
            "value": {"mode": "fixed", "fixedColor": "red"}
          }
        ]
      }
    ]
  }
}
```

### 4.9 Panel : Top Commandes (Table)

**Type** : Table
**Objectif** : Identifier les commandes les plus fréquentes

**Requête PromQL** :
```promql
topk(10,
  sum(rate(redis_command_call_duration_seconds_count{instance=~"$instance"}[5m])) by (cmd)
)
```

**Configuration JSON** :
```json
{
  "title": "Top 10 Commands by Frequency",
  "type": "table",
  "targets": [
    {
      "expr": "topk(10, sum(rate(redis_command_call_duration_seconds_count{instance=~\"$instance\"}[5m])) by (cmd))",
      "format": "table",
      "instant": true,
      "refId": "A"
    }
  ],
  "transformations": [
    {
      "id": "organize",
      "options": {
        "excludeByName": {"Time": true},
        "indexByName": {},
        "renameByName": {
          "cmd": "Command",
          "Value": "Calls/sec"
        }
      }
    }
  ],
  "fieldConfig": {
    "overrides": [
      {
        "matcher": {"id": "byName", "options": "Calls/sec"},
        "properties": [
          {
            "id": "custom.displayMode",
            "value": "gradient-gauge"
          },
          {
            "id": "unit",
            "value": "ops"
          }
        ]
      }
    ]
  }
}
```

### 4.10 Panel : Instances Status (Table)

**Type** : Table
**Objectif** : Vue d'ensemble de toutes les instances

**Requête PromQL** :
```promql
# Up/Down
redis_up{instance=~"$instance"}

# Memory %
(redis_memory_used_bytes / redis_memory_maxmemory_bytes) * 100

# Hit Ratio
rate(redis_keyspace_hits_total[5m]) /
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) * 100

# Ops/sec
rate(redis_commands_processed_total[1m])
```

**Configuration JSON multi-target** :
```json
{
  "title": "Redis Instances Overview",
  "type": "table",
  "targets": [
    {
      "expr": "redis_up{instance=~\"$instance\"}",
      "format": "table",
      "instant": true,
      "refId": "A"
    },
    {
      "expr": "(redis_memory_used_bytes{instance=~\"$instance\"} / redis_memory_maxmemory_bytes{instance=~\"$instance\"}) * 100",
      "format": "table",
      "instant": true,
      "refId": "B"
    },
    {
      "expr": "100 * (rate(redis_keyspace_hits_total{instance=~\"$instance\"}[5m]) / (rate(redis_keyspace_hits_total{instance=~\"$instance\"}[5m]) + rate(redis_keyspace_misses_total{instance=~\"$instance\"}[5m])))",
      "format": "table",
      "instant": true,
      "refId": "C"
    }
  ],
  "transformations": [
    {
      "id": "merge",
      "options": {}
    },
    {
      "id": "organize",
      "options": {
        "excludeByName": {"Time": true, "job": true},
        "renameByName": {
          "instance": "Instance",
          "redis_role": "Role",
          "Value #A": "Status",
          "Value #B": "Memory %",
          "Value #C": "Hit Ratio %"
        }
      }
    }
  ],
  "fieldConfig": {
    "overrides": [
      {
        "matcher": {"id": "byName", "options": "Status"},
        "properties": [
          {
            "id": "mappings",
            "value": [
              {
                "type": "value",
                "options": {
                  "0": {"text": "DOWN", "color": "red"},
                  "1": {"text": "UP", "color": "green"}
                }
              }
            ]
          },
          {
            "id": "custom.displayMode",
            "value": "color-background"
          }
        ]
      }
    ]
  }
}
```

## 5. Dashboards par Use Case

### 5.1 Dashboard : Cache Performance

**Focus** : Hit ratio, Latency, TTL effectiveness

**Panels spécifiques** :
1. **Hit/Miss Ratio** (Time Series)
2. **Cache Size vs Capacity** (Gauge)
3. **Average TTL** (Stat)
4. **Top Missed Keys** (Table - si --check-keys activé)
5. **Evictions Rate** (Time Series avec alerte)

**Requêtes clés** :
```promql
# Hit ratio
100 * rate(redis_keyspace_hits_total[5m]) /
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

# TTL moyen
redis_db_avg_ttl_seconds

# Keys avec TTL vs sans TTL
redis_db_keys_expiring / redis_db_keys
```

### 5.2 Dashboard : Session Store

**Focus** : Disponibilité, Latency, Session count

**Panels spécifiques** :
1. **Uptime** (Stat avec 99.9% SLA)
2. **Active Sessions** (Time Series - redis_db_keys)
3. **Session Creation Rate** (Time Series)
4. **Read Latency P99** (Time Series)
5. **Expired Sessions** (Time Series)

**Requêtes clés** :
```promql
# Uptime %
avg_over_time(redis_up[24h]) * 100

# Sessions actives
redis_db_keys{db="0"}

# Taux de création (approximatif via commands)
rate(redis_commands_total{cmd="setex"}[5m])
```

### 5.3 Dashboard : Queue/Stream Monitoring

**Focus** : Queue depth, Processing rate, Consumer health

**Panels spécifiques** :
1. **Queue Depth** (Time Series - si LIST)
2. **Stream Length** (Time Series - si STREAM)
3. **Consumer Groups** (Stat)
4. **Pending Messages** (Time Series)
5. **Processing Rate** (Time Series)

**Requêtes clés** :
```promql
# Stream length (si --check-streams)
redis_stream_length{stream="orders"}

# Pending messages par groupe
redis_stream_group_pending{stream="orders", group="workers"}

# Processing rate (via XACK)
rate(redis_commands_total{cmd="xack"}[5m])
```

### 5.4 Dashboard : Cluster Monitoring

**Focus** : Slots distribution, Node health, Resharding

**Panels spécifiques** :
1. **Cluster State** (Stat - OK/FAIL)
2. **Slots Distribution** (Bar Gauge par node)
3. **Cluster-wide Operations** (Time Series)
4. **Failing Nodes** (Stat)
5. **Migration Status** (Stat si resharding)

**Requêtes clés** :
```promql
# Cluster state
redis_cluster_state

# Slots par node
sum(redis_cluster_slots_ok) by (instance)

# Nodes en fail
count(redis_cluster_state == 0)

# Slots en migration
redis_cluster_slots_migrating + redis_cluster_slots_importing
```

### 5.5 Dashboard : Capacity Planning

**Focus** : Trends, Growth, Projections

**Panels spécifiques** :
1. **Memory Growth (30d)** (Time Series)
2. **Keys Growth (30d)** (Time Series)
3. **Projected OOM Date** (Stat - via predict_linear)
4. **Daily Ops Growth** (Time Series)
5. **Cost Projection** (Stat - si métriques custom)

**Requêtes clés** :
```promql
# Memory growth trend
predict_linear(redis_memory_used_bytes[30d], 86400 * 30)

# Nombre de jours avant OOM (si tendance linéaire)
(redis_memory_maxmemory_bytes - redis_memory_used_bytes) /
(deriv(redis_memory_used_bytes[7d]) * 86400)

# Keys growth rate
deriv(redis_db_keys[7d]) * 86400 * 30
```

## 6. Annotations et Événements

### 6.1 Annotations automatiques

**Événements Redis** :
```json
{
  "annotations": {
    "list": [
      {
        "name": "Redis Restarts",
        "datasource": "Prometheus",
        "enable": true,
        "expr": "changes(redis_uptime_in_seconds[5m]) > 0",
        "step": "5m",
        "titleFormat": "Redis Restarted",
        "textFormat": "Instance {{instance}} restarted",
        "tagKeys": "instance,redis_role",
        "iconColor": "red"
      }
    ]
  }
}
```

**Déploiements** :
```json
{
  "name": "Deployments",
  "datasource": "Prometheus",
  "enable": true,
  "expr": "changes(kube_deployment_status_observed_generation[5m]) > 0",
  "titleFormat": "Deployment",
  "textFormat": "App deployed",
  "iconColor": "blue"
}
```

### 6.2 Annotations manuelles

**Via API** :
```bash
curl -X POST http://localhost:3000/api/annotations \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "dashboardId": 1,
    "panelId": 1,
    "time": '$(date +%s000)',
    "timeEnd": '$(date +%s000)',
    "tags": ["maintenance", "redis"],
    "text": "Redis maintenance: Memory defragmentation"
  }'
```

## 7. Alerting dans Grafana

### 7.1 Configuration d'alertes

**Alerte : Memory High** :
```json
{
  "alert": {
    "name": "Redis Memory High",
    "message": "Redis memory usage above 85% on {{instance}}",
    "conditions": [
      {
        "evaluator": {
          "type": "gt",
          "params": [85]
        },
        "operator": {"type": "and"},
        "query": {
          "params": ["A", "5m", "now"]
        },
        "reducer": {
          "type": "avg"
        },
        "type": "query"
      }
    ],
    "executionErrorState": "alerting",
    "frequency": "1m",
    "handler": 1,
    "noDataState": "no_data",
    "notifications": [
      {"uid": "slack-notifications"},
      {"uid": "pagerduty-critical"}
    ]
  }
}
```

### 7.2 Notification Channels

**Slack** :
```json
{
  "name": "Slack Alerts",
  "type": "slack",
  "settings": {
    "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
    "username": "Grafana",
    "iconEmoji": ":chart_with_upwards_trend:",
    "mentionChannel": "here",
    "recipient": "#redis-alerts"
  }
}
```

**PagerDuty** :
```json
{
  "name": "PagerDuty Critical",
  "type": "pagerduty",
  "settings": {
    "integrationKey": "YOUR_INTEGRATION_KEY",
    "severity": "critical",
    "autoResolve": true
  }
}
```

### 7.3 Alert Rules Best Practices

**Règle 1 : Conditions multiples** :
```
IF Memory > 85%
AND Evictions > 0
FOR 5 minutes
→ ALERT
```

**Règle 2 : Taux de changement** :
```promql
# Alerter si memory augmente de > 10% en 15 min
(redis_memory_used_bytes - redis_memory_used_bytes offset 15m) /
redis_memory_used_bytes offset 15m > 0.10
```

**Règle 3 : Anomaly detection** :
```promql
# Alerter si metric dévie de > 3σ de la moyenne 7j
abs(redis_memory_used_bytes - avg_over_time(redis_memory_used_bytes[7d])) >
(3 * stddev_over_time(redis_memory_used_bytes[7d]))
```

## 8. Optimisation des Dashboards

### 8.1 Performance des requêtes

**❌ Requête lente** :
```promql
# Trop de labels, trop de séries
redis_memory_used_bytes{instance=~".*"}[30d]
```

**✅ Requête optimisée** :
```promql
# Limiter le time range, utiliser recording rules
redis:memory_used:avg1h{instance=~"$instance"}[7d]
```

**Recording rule** :
```yaml
groups:
  - name: redis_optimizations
    interval: 60s
    rules:
      - record: redis:memory_used:avg1h
        expr: avg_over_time(redis_memory_used_bytes[1h])
```

### 8.2 Caching dans Grafana

**Configuration** :
```ini
# grafana.ini
[dataproxy]
timeout = 30
keep_alive_seconds = 30

[caching]
enabled = true
ttl = 300  # 5 minutes
```

### 8.3 Limitation des séries temporelles

**Utiliser topk/bottomk** :
```promql
# ❌ Toutes les instances
redis_memory_used_bytes

# ✅ Top 10 seulement
topk(10, redis_memory_used_bytes)
```

## 9. Export, Import et Versioning

### 9.1 Export JSON

**Via UI** :
```
Dashboard → Settings → JSON Model → Copy to clipboard
```

**Via API** :
```bash
DASHBOARD_UID="redis-overview"
curl -H "Authorization: Bearer ${API_KEY}" \
  http://localhost:3000/api/dashboards/uid/${DASHBOARD_UID} | \
  jq '.dashboard' > redis-dashboard.json
```

### 9.2 Versioning avec Git

**Structure recommandée** :
```
grafana-dashboards/
├── README.md
├── provisioning/
│   └── dashboards.yml
└── dashboards/
    ├── redis-overview.json
    ├── redis-performance.json
    ├── redis-cluster.json
    └── redis-capacity.json
```

**Provisioning automatique** :
```yaml
# provisioning/dashboards.yml
apiVersion: 1

providers:
  - name: 'Redis Dashboards'
    orgId: 1
    folder: 'Redis'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /etc/grafana/dashboards
      foldersFromFilesStructure: true
```

### 9.3 CI/CD pour Dashboards

**GitHub Actions** :
```yaml
name: Deploy Grafana Dashboards

on:
  push:
    branches: [main]
    paths:
      - 'dashboards/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Deploy to Grafana
        env:
          GRAFANA_URL: ${{ secrets.GRAFANA_URL }}
          GRAFANA_API_KEY: ${{ secrets.GRAFANA_API_KEY }}
        run: |
          for dashboard in dashboards/*.json; do
            curl -X POST "${GRAFANA_URL}/api/dashboards/db" \
              -H "Authorization: Bearer ${GRAFANA_API_KEY}" \
              -H "Content-Type: application/json" \
              -d @${dashboard}
          done
```

## 10. Best Practices Récapitulatives

### 10.1 Design

- ✅ Hiérarchie claire (Overview → Détail)
- ✅ Maximum 15-20 panels par dashboard
- ✅ Variables pour filtrage dynamique
- ✅ Couleurs cohérentes (rouge=danger, vert=OK)
- ✅ Seuils basés sur SLO/SLA
- ❌ Éviter les vanity metrics
- ❌ Pas de panels sans action claire

### 10.2 Performance

- ✅ Recording rules pour requêtes complexes
- ✅ Limiter les time ranges (< 7j par défaut)
- ✅ Utiliser topk/bottomk au lieu de *
- ✅ Cache des résultats (5-15 min)
- ❌ Éviter les regex complexes
- ❌ Pas de requêtes > 30s

### 10.3 Maintenance

- ✅ Versioning Git des dashboards
- ✅ Provisioning automatique
- ✅ Documentation inline (descriptions)
- ✅ Review périodique (trimestielle)
- ❌ Pas de dashboards orphelins
- ❌ Pas de credentials hardcodés

### 10.4 Alerting

- ✅ Alertes actionnables uniquement
- ✅ Runbooks documentés
- ✅ Escalation claire (Warning → Critical)
- ✅ Auto-résolution quand possible
- ❌ Éviter l'alert fatigue
- ❌ Pas d'alertes sur des dérivées

## Conclusion

Un dashboard Grafana efficace pour Redis doit :

1. **Informer** : Métriques critiques visibles immédiatement
2. **Contextualiser** : Historique, baselines, seuils
3. **Guider** : Drill-down intuitif vers les détails
4. **Alerter** : Problèmes détectés avant impact

**Checklist finale** :
- [ ] Variables configurées (instance, env, role)
- [ ] Panels critiques présents (memory, hit ratio, évictions)
- [ ] Seuils basés sur des valeurs réelles
- [ ] Alertes configurées avec notifications
- [ ] Dashboard versionné dans Git
- [ ] Documentation des panels
- [ ] Tests de performance (< 5s load time)
- [ ] Review avec l'équipe

Un bon dashboard Redis se construit par itérations successives, en ajustant les panels et seuils basés sur l'expérience opérationnelle réelle.

---

**Prochaine section** : 13.5 - Latency Doctor et Latency Monitoring (diagnostic avancé)

⏭️ [Latency Doctor et Latency Monitoring](/13-monitoring-observabilite/05-latency-doctor-monitoring.md)
