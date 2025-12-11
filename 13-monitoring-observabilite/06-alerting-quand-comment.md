🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.6 Alerting : Quand et comment alerter

## Introduction

L'alerting est un art délicat : trop peu d'alertes = incidents non détectés, trop d'alertes = désensibilisation et alert fatigue. Cette section couvre les stratégies d'alerting production-ready pour Redis, de la conception des règles à l'intégration avec l'astreinte.

### Le paradoxe de l'alerting

```
┌─────────────────────────────────────────────┐
│         Trop d'alertes                      │
│    (Alert Fatigue)                          │
│                                             │
│  • Désensibilisation de l'équipe            │
│  • Alertes ignorées                         │
│  • Vrais incidents manqués                  │
│  • Burnout on-call                          │
└─────────────────────────────────────────────┘
                    ▲
                    │
              Sweet Spot
                    │
                    ▼
┌─────────────────────────────────────────────┐
│         Pas assez d'alertes                 │
│    (Blind Spots)                            │
│                                             │
│  • Incidents découverts par les users       │
│  • Temps de réponse élevé                   │
│  • Impact business majeur                   │
│  • Perte de confiance                       │
└─────────────────────────────────────────────┘
```

### Principes fondamentaux

**Règle d'or** : Chaque alerte doit être **actionnaire** et **urgente**

```
Alerte = Quelque chose est cassé OU va casser bientôt
       + Action humaine immédiate requise
       + Impact utilisateur si non traité
```

## 1. Philosophie de l'Alerting

### 1.1 Symptom-Based vs Cause-Based

#### Symptom-Based (Recommandé)

**Définition** : Alerter sur l'impact utilisateur, pas sur la cause technique

**Exemple Redis** :
```yaml
# ✅ BON : Symptom-based
- alert: RedisHighLatency
  expr: redis:latency_p99:ms > 50
  annotations:
    summary: "Les utilisateurs subissent des lenteurs"

# ❌ MAUVAIS : Cause-based
- alert: RedisCPUHigh
  expr: redis_cpu_usage > 80
  annotations:
    summary: "CPU élevé"
```

**Pourquoi ?**
- CPU 80% peut être normal si latence OK
- CPU 30% est problématique si latence élevée
- **Alerter sur ce qui impacte les users, pas sur les métriques intermédiaires**

#### Quand utiliser Cause-Based ?

**Acceptable pour** :
- Prédiction (avant impact) : `memory > 90%` → OOM imminent
- Debugging (combiné avec symptom) : Latence élevée + CPU 100%

### 1.2 Niveaux de Sévérité

**Architecture à 3 niveaux** :

| Niveau | Définition | Action | Notification | Exemple |
|--------|-----------|--------|--------------|---------|
| **Warning** | Tendance dégradée, pas d'impact immédiat | Enquête en heures ouvrées | Slack | Memory > 80% |
| **Critical** | Impact utilisateur actuel ou imminent | Action immédiate | PagerDuty | Latence > 100ms |
| **Info** | Événement notable, pas d'action | Logging/dashboard | Slack (optionnel) | Deployment |

**Erreur courante** : Trop de niveaux (low, medium, high, critical, urgent, emergency)
→ Confusion et désensibilisation

### 1.3 Alert Fatigue : Comment l'éviter

**Symptômes d'alert fatigue** :
- Alertes ignorées systématiquement
- "Acknowledge without checking"
- Burnout on-call
- Vraies alertes manquées

**Solutions** :

#### 1. Tuning agressif des seuils
```yaml
# ❌ Trop sensible (alerte chaque nuit)
- alert: RedisMemoryHigh
  expr: redis_memory_used_pct > 70
  for: 5m

# ✅ Seuil réaliste
- alert: RedisMemoryHigh
  expr: redis_memory_used_pct > 85
  for: 10m
```

#### 2. Période de "for" appropriée
```yaml
# ❌ Trop court (faux positifs sur spikes temporaires)
- alert: RedisLatencyHigh
  expr: redis:latency_p99:ms > 10
  for: 30s

# ✅ Filtre les spikes
- alert: RedisLatencyHigh
  expr: redis:latency_p99:ms > 10
  for: 5m
```

#### 3. Alertes basées sur tendance, pas valeur absolue
```promql
# ❌ Valeur fixe
redis_connected_clients > 900

# ✅ Déviation de la baseline
abs(redis_connected_clients - avg_over_time(redis_connected_clients[7d])) >
(3 * stddev_over_time(redis_connected_clients[7d]))
```

#### 4. Grouping et deduplication
```yaml
# Grouper les alertes similaires
route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

## 2. Alertes Essentielles Redis

### 2.1 Tier 1 : Alertes Critiques (PagerDuty)

Ces alertes DOIVENT réveiller quelqu'un à 3h du matin.

#### Alerte 1 : Redis Down

```yaml
- alert: RedisDown
  expr: redis_up == 0
  for: 1m
  labels:
    severity: critical
    tier: 1
  annotations:
    summary: "Redis instance {{ $labels.instance }} is DOWN"
    description: |
      Redis ne répond plus depuis 1 minute.

      Impact: Tous les services dépendants sont affectés.

      Action immédiate:
      1. Vérifier l'état du process: systemctl status redis
      2. Vérifier les logs: journalctl -u redis -n 100
      3. Si crash: vérifier dmesg (OOM killer?)
      4. Redémarrer si nécessaire

      Runbook: https://wiki.company.com/redis-down
```

**Justification** : Redis indisponible = impact immédiat 100% des users

#### Alerte 2 : Latence Critique

```yaml
- alert: RedisLatencyCritical
  expr: |
    histogram_quantile(0.99,
      rate(redis_command_call_duration_seconds_bucket[5m])
    ) * 1000 > 100
  for: 3m
  labels:
    severity: critical
    tier: 1
  annotations:
    summary: "Redis latency P99 critique sur {{ $labels.instance }}"
    description: |
      Latence P99: {{ $value | humanize }}ms (seuil: 100ms)

      Impact: Timeouts applicatifs, UX dégradée.

      Diagnostic immédiat:
      1. redis-cli LATENCY DOCTOR
      2. redis-cli SLOWLOG GET 10
      3. redis-cli INFO stats | grep latest_fork_usec
      4. Vérifier dashboard Grafana pour cause (spike ops? memory?)

      Causes fréquentes:
      - Fork (BGSAVE) en cours
      - Commandes lentes (KEYS, HGETALL)
      - Swap actif
      - Saturation réseau

      Runbook: https://wiki.company.com/redis-latency
```

**Justification** : Latence > 100ms = timeouts, impact direct users

#### Alerte 3 : Memory Critical (OOM Imminent)

```yaml
- alert: RedisMemoryCritical
  expr: |
    (redis_memory_used_bytes / redis_memory_maxmemory_bytes) * 100 > 95
  for: 2m
  labels:
    severity: critical
    tier: 1
  annotations:
    summary: "Redis mémoire CRITIQUE sur {{ $labels.instance }}"
    description: |
      Utilisation: {{ $value | humanize1024 }}% (seuil: 95%)

      Impact: OOM imminent, évictions ou crashes dans les minutes qui suivent.

      Action URGENTE:
      1. Vérifier politique éviction: redis-cli CONFIG GET maxmemory-policy
      2. Si noeviction: basculer temporairement sur allkeys-lru
         redis-cli CONFIG SET maxmemory-policy allkeys-lru
      3. Augmenter maxmemory si RAM disponible:
         redis-cli CONFIG SET maxmemory 8gb
      4. Identifier les gros consommateurs:
         redis-cli --bigkeys
      5. Planifier scaling (+ RAM ou cluster)

      Runbook: https://wiki.company.com/redis-oom
```

**Justification** : OOM = crash imminent ou évictions massives

#### Alerte 4 : Replication Broken (Master-Replica)

```yaml
- alert: RedisReplicationBroken
  expr: |
    redis_master_link_up{redis_role="replica"} == 0
  for: 2m
  labels:
    severity: critical
    tier: 1
  annotations:
    summary: "Replica {{ $labels.instance }} déconnecté du master"
    description: |
      Le replica ne peut plus se synchroniser avec le master.

      Impact:
      - Perte de redondance
      - Risque de split-brain si failover
      - Backup compromis

      Diagnostic:
      1. Depuis le replica: redis-cli INFO replication
      2. Vérifier réseau entre master et replica
      3. Vérifier authentification (password/ACL)
      4. Vérifier logs replica: grep "MASTER" /var/log/redis/redis.log

      Si prolongé > 15 min:
      - Envisager resync manuel
      - Ou provisionner nouveau replica

      Runbook: https://wiki.company.com/redis-replication
```

**Justification** : Réplication cassée = perte de HA, risque de data loss

#### Alerte 5 : Cluster State FAIL

```yaml
- alert: RedisClusterFail
  expr: redis_cluster_state == 0
  for: 1m
  labels:
    severity: critical
    tier: 1
  annotations:
    summary: "Redis Cluster en état FAIL"
    description: |
      Le cluster Redis est en état FAIL (slots non couverts ou nodes down).

      Impact:
      - Certaines clés inaccessibles
      - Requêtes échouent avec CLUSTERDOWN
      - Perte partielle ou totale de service

      Action immédiate:
      1. Identifier les nodes en fail:
         redis-cli CLUSTER NODES | grep fail
      2. Vérifier les slots non couverts:
         redis-cli CLUSTER INFO | grep cluster_slots
      3. Tenter de restaurer les nodes en fail
      4. Si node irrémédiablement down:
         - Fail over vers replica
         - Ou réassigner les slots

      Runbook: https://wiki.company.com/redis-cluster-fail
```

**Justification** : Cluster FAIL = service potentiellement down

### 2.2 Tier 2 : Alertes Warning (Slack)

Ces alertes nécessitent attention mais pas réveil nocturne.

#### Alerte 6 : Memory High (Trend)

```yaml
- alert: RedisMemoryHigh
  expr: |
    (redis_memory_used_bytes / redis_memory_maxmemory_bytes) * 100 > 80
  for: 15m
  labels:
    severity: warning
    tier: 2
  annotations:
    summary: "Memory élevée sur {{ $labels.instance }}"
    description: |
      Utilisation: {{ $value | humanize }}%

      Pas d'urgence immédiate mais surveillance nécessaire.

      Actions à planifier (heures ouvrées):
      1. Analyser la tendance sur 7j (dashboard Grafana)
      2. Si croissance continue: planifier scaling
      3. Vérifier évictions: redis-cli INFO stats | grep evicted_keys
      4. Optimiser si possible (compression, TTL, structures)

      Projection OOM:
      {{ query "predict_linear(redis_memory_used_bytes[7d], 86400*7)" | humanize }} dans 7j
```

#### Alerte 7 : Hit Ratio Low

```yaml
- alert: RedisHitRatioLow
  expr: |
    100 * (
      rate(redis_keyspace_hits_total[10m]) /
      (rate(redis_keyspace_hits_total[10m]) + rate(redis_keyspace_misses_total[10m]))
    ) < 80
  for: 30m
  labels:
    severity: warning
    tier: 2
  annotations:
    summary: "Hit ratio faible sur {{ $labels.instance }}"
    description: |
      Hit ratio: {{ $value | humanizePercentage }}

      Impact: Charge accrue sur la base de données backend.

      Investigations:
      1. Comparer avec baseline 7j
      2. Vérifier évictions récentes
      3. Analyser pattern d'accès (nouveau trafic?)
      4. Vérifier TTL (trop courts?)

      Si dégradation continue:
      - Considérer cache warming
      - Augmenter TTL si approprié
      - Augmenter capacité mémoire
```

#### Alerte 8 : Fragmentation High

```yaml
- alert: RedisFragmentationHigh
  expr: redis_mem_fragmentation_ratio > 1.5
  for: 1h
  labels:
    severity: warning
    tier: 2
  annotations:
    summary: "Fragmentation élevée sur {{ $labels.instance }}"
    description: |
      Fragmentation ratio: {{ $value }}
      Mémoire gaspillée: {{ query "redis_mem_fragmentation_bytes{instance='$instance'}" | humanize1024 }}

      Impact: Surconsommation mémoire ({{ $value | humanize }}× l'usage réel)

      Actions:
      1. Activer active defrag si pas déjà fait:
         redis-cli CONFIG SET activedefrag yes
      2. Monitorer évolution sur 24h
      3. Si pas d'amélioration: planifier redémarrage maintenance

      Note: Fragmentation > 2.0 justifie un redémarrage
```

#### Alerte 9 : Evictions Active

```yaml
- alert: RedisEvictingKeys
  expr: rate(redis_evicted_keys_total[5m]) > 10
  for: 10m
  labels:
    severity: warning
    tier: 2
  annotations:
    summary: "Évictions actives sur {{ $labels.instance }}"
    description: |
      Taux d'éviction: {{ $value }} clés/sec

      Impact: Perte de données chaudes, hit ratio dégradé.

      Causes:
      - Maxmemory trop petit pour le workload
      - Dataset croissant sans scaling

      Actions:
      1. Confirmer politique éviction (allkeys-lru recommandé)
      2. Estimer besoin mémoire réel
      3. Planifier augmentation maxmemory ou scaling
```

#### Alerte 10 : Fork Latency High

```yaml
- alert: RedisForkLatencyHigh
  expr: redis_latest_fork_seconds > 2
  for: 5m
  labels:
    severity: warning
    tier: 2
  annotations:
    summary: "Fork latency élevé sur {{ $labels.instance }}"
    description: |
      Dernière latence fork: {{ $value }}s

      Impact: Spike de latence lors des BGSAVE/AOF rewrite

      Causes probables:
      1. THP activé (Transparent Huge Pages)
      2. Dataset large avec RAM limite
      3. Fragmentation élevée

      Actions:
      1. Vérifier THP: cat /sys/kernel/mm/transparent_hugepage/enabled
      2. Si [always]: désactiver (echo never)
      3. Considérer augmentation RAM si dataset > 50% RAM totale
```

#### Alerte 11 : Slowlog Growing

```yaml
- alert: RedisSlowlogGrowing
  expr: rate(redis_slowlog_length[10m]) > 1
  for: 15m
  labels:
    severity: warning
    tier: 2
  annotations:
    summary: "Slowlog en croissance sur {{ $labels.instance }}"
    description: |
      {{ $value }} commandes lentes/min enregistrées

      Impact potentiel: Commandes bloquantes dégradant la performance.

      Investigation:
      1. redis-cli SLOWLOG GET 10
      2. Identifier patterns (KEYS, HGETALL, etc.)
      3. Tracer dans le code applicatif
      4. Remplacer par alternatives (SCAN, HSCAN)

      Dashboard: [lien vers Grafana slowlog panel]
```

#### Alerte 12 : Connected Clients High

```yaml
- alert: RedisClientsHigh
  expr: |
    redis_connected_clients / redis_config_maxclients > 0.8
  for: 10m
  labels:
    severity: warning
    tier: 2
  annotations:
    summary: "Nombre de clients élevé sur {{ $labels.instance }}"
    description: |
      Clients: {{ query "redis_connected_clients{instance='$instance'}" }}
      Maxclients: {{ query "redis_config_maxclients{instance='$instance'}" }}
      Utilisation: {{ $value | humanizePercentage }}

      Risque: Refus de nouvelles connexions si maxclients atteint.

      Investigation:
      1. Vérifier si normal (scaling app?)
      2. Suspecter connection leak si croissance anormale
      3. redis-cli CLIENT LIST | wc -l

      Si anormal:
      - Vérifier connection pooling applicatif
      - Augmenter maxclients si légitime
```

### 2.3 Tier 3 : Alertes Info (Dashboard)

Ces alertes sont informatives, pas d'action requise.

```yaml
- alert: RedisRestarted
  expr: changes(redis_uptime_in_seconds[5m]) > 0
  labels:
    severity: info
  annotations:
    summary: "Redis {{ $labels.instance }} a redémarré"
    description: |
      L'instance a été redémarrée il y a {{ query "redis_uptime_in_seconds{instance='$instance'}" }}s

      Vérifier:
      - Redémarrage planifié? (maintenance)
      - Ou crash? (vérifier logs)

- alert: RedisBGSaveInProgress
  expr: redis_rdb_bgsave_in_progress == 1
  labels:
    severity: info
  annotations:
    summary: "BGSAVE en cours sur {{ $labels.instance }}"
    description: |
      Backup RDB en cours. Possible spike de latence.
      Durée typique: {{ query "redis_rdb_last_bgsave_time_sec{instance='$instance'}" }}s
```

## 3. Configuration Prometheus Alertmanager

### 3.1 Architecture Alertmanager

```
┌──────────────────────────────────────────────┐
│           Prometheus Server                  │
│  (Évaluation des règles d'alerte)            │
└────────────────┬─────────────────────────────┘
                 │
                 │ Alertes actives
                 ▼
┌───────────────────────────────────────────┐
│           Alertmanager                    │
│                                           │
│  ┌────────────────────────────────────┐   │
│  │  Grouping & Deduplication          │   │
│  └────────────┬───────────────────────┘   │
│               │                           │
│  ┌────────────▼───────────────────────┐   │
│  │  Silencing & Inhibition            │   │
│  └────────────┬───────────────────────┘   │
│               │                           │
│  ┌────────────▼───────────────────────┐   │
│  │  Routing                           │   │
│  └────────────┬───────────────────────┘   │
└───────────────┼───────────────────────────┘
                │
      ┌─────────┼──────────┐
      │         │          │
      ▼         ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│  Slack  │ │PagerDuty│ │  Email  │
└─────────┘ └─────────┘ └─────────┘
```

### 3.2 Configuration Alertmanager (alertmanager.yml)

```yaml
global:
  # Résolution URL Prometheus (pour liens dans alertes)
  resolve_timeout: 5m

  # Configuration Slack globale
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

  # Configuration PagerDuty globale
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates custom
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing des alertes
route:
  # Labels de regroupement
  group_by: ['alertname', 'instance', 'severity']

  # Attendre 30s pour grouper alertes similaires
  group_wait: 30s

  # Intervalle entre groupes
  group_interval: 5m

  # Ne pas répéter avant 4h
  repeat_interval: 4h

  # Receiver par défaut
  receiver: 'slack-warnings'

  # Routes spécifiques
  routes:
    # Critiques → PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      group_wait: 10s
      repeat_interval: 1h
      continue: true  # Continuer vers autres routes

    # Critiques → Slack aussi
    - match:
        severity: critical
      receiver: 'slack-critical'
      group_wait: 10s

    # Warnings → Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'

    # Info → Slack optionnel
    - match:
        severity: info
      receiver: 'slack-info'
      group_interval: 30m
      repeat_interval: 24h

    # Alertes Redis spécifiques → Channel dédié
    - match_re:
        alertname: 'Redis.*'
      receiver: 'slack-redis'
      continue: true

# Inhibition (suppression conditionnelle)
inhibit_rules:
  # Si Redis down, supprimer toutes les autres alertes Redis
  - source_match:
      alertname: 'RedisDown'
    target_match_re:
      alertname: 'Redis.*'
    equal: ['instance']

  # Si cluster fail, supprimer alertes nodes individuels
  - source_match:
      alertname: 'RedisClusterFail'
    target_match_re:
      alertname: 'Redis.*'
    equal: ['cluster']

# Receivers (destinations)
receivers:
  # PagerDuty Critical
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        severity: 'critical'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
          num_firing: '{{ .Alerts.Firing | len }}'
          num_resolved: '{{ .Alerts.Resolved | len }}'

  # Slack Critical
  - name: 'slack-critical'
    slack_configs:
      - channel: '#redis-alerts-critical'
        color: 'danger'
        title: ':rotating_light: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Summary:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Instance:* {{ .Labels.instance }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
        actions:
          - type: button
            text: 'Runbook :book:'
            url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
          - type: button
            text: 'Dashboard :chart_with_upwards_trend:'
            url: 'https://grafana.company.com/d/redis-overview'
          - type: button
            text: 'Silence :mute:'
            url: '{{ .ExternalURL }}/#/silences/new?filter=%7B{{ .GroupLabels.SortedPairs.Values | join "," }}%7D'

  # Slack Warnings
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#redis-alerts'
        color: 'warning'
        title: ':warning: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Summary:* {{ .Annotations.summary }}
          *Instance:* {{ .Labels.instance }}
          {{ end }}

  # Slack Redis dédié
  - name: 'slack-redis'
    slack_configs:
      - channel: '#redis-team'
        color: '{{ if eq .Status "firing" }}warning{{ else }}good{{ end }}'
        title: '{{ .GroupLabels.alertname }}'
        text: |
          {{ .CommonAnnotations.summary }}

          Affected instances: {{ .Alerts.Firing | len }}

  # Slack Info
  - name: 'slack-info'
    slack_configs:
      - channel: '#redis-info'
        color: 'good'
        title: 'ℹ️ {{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.summary }}'
```

### 3.3 Templates personnalisés

**Template Slack enrichi** (`/etc/alertmanager/templates/slack.tmpl`) :
```go
{{ define "slack.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }}
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
*Alert:* {{ .Labels.alertname }}
*Severity:* {{ .Labels.severity }}
*Instance:* {{ .Labels.instance }}
*Summary:* {{ .Annotations.summary }}

*Description:*
{{ .Annotations.description }}

*Value:* {{ .Value }}
*Started at:* {{ .StartsAt.Format "2006-01-02 15:04:05 MST" }}
{{ if ne .Status "firing" }}*Resolved at:* {{ .EndsAt.Format "2006-01-02 15:04:05 MST" }}{{ end }}

---
{{ end }}

*Runbook:* {{ .CommonAnnotations.runbook_url }}
*Dashboard:* https://grafana.company.com/d/redis-overview
*Silence:* {{ .ExternalURL }}/#/silences/new
{{ end }}
```

### 3.4 Silences programmatiques

**API pour créer un silence** :
```bash
# Silence pendant maintenance (2h)
START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
END=$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ)

curl -X POST http://alertmanager:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {
        "name": "instance",
        "value": "redis-prod-1:9121",
        "isRegex": false
      }
    ],
    "startsAt": "'$START'",
    "endsAt": "'$END'",
    "createdBy": "ops-team",
    "comment": "Maintenance planifiée: défragmentation mémoire"
  }'
```

**Script pour maintenance** :
```bash
#!/bin/bash
# silence-redis.sh

INSTANCE="$1"
DURATION_HOURS="$2"
REASON="$3"

if [ -z "$INSTANCE" ] || [ -z "$DURATION_HOURS" ]; then
  echo "Usage: $0 <instance> <duration_hours> <reason>"
  exit 1
fi

START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
END=$(date -u -d "+${DURATION_HOURS} hours" +%Y-%m-%dT%H:%M:%SZ)

curl -X POST http://alertmanager:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d "{
    \"matchers\": [
      {\"name\": \"instance\", \"value\": \"${INSTANCE}\", \"isRegex\": false}
    ],
    \"startsAt\": \"${START}\",
    \"endsAt\": \"${END}\",
    \"createdBy\": \"$(whoami)\",
    \"comment\": \"${REASON}\"
  }"

echo "Silence créé pour ${INSTANCE} pendant ${DURATION_HOURS}h"
echo "Raison: ${REASON}"
```

**Utilisation** :
```bash
./silence-redis.sh redis-prod-1:9121 2 "Redémarrage planifié pour défragmentation"
```

## 4. Notification Channels

### 4.1 PagerDuty (On-call)

**Configuration du service** :
```yaml
pagerduty_configs:
  - service_key: 'YOUR_SERVICE_KEY'

    # Routing key (PagerDuty Events API v2)
    routing_key: 'YOUR_ROUTING_KEY'

    # URL custom (si on-premise)
    url: 'https://events.pagerduty.com/v2/enqueue'

    # Severity mapping
    severity: '{{ if eq .Labels.severity "critical" }}critical{{ else }}warning{{ end }}'

    # Détails de l'alerte
    description: '{{ .CommonAnnotations.summary }}'

    client: 'Prometheus Alertmanager'
    client_url: '{{ .ExternalURL }}'

    details:
      firing: '{{ .Alerts.Firing | len }}'
      resolved: '{{ .Alerts.Resolved | len }}'
      alertname: '{{ .GroupLabels.alertname }}'
      instance: '{{ .GroupLabels.instance }}'
      severity: '{{ .GroupLabels.severity }}'
      description: '{{ .CommonAnnotations.description }}'
      runbook: '{{ .CommonAnnotations.runbook_url }}'
      dashboard: 'https://grafana.company.com/d/redis-overview'
```

**Escalation policy PagerDuty** :
```
Level 1: Primary on-call (immédiat)
  ↓ (15 min sans ACK)
Level 2: Secondary on-call
  ↓ (15 min sans ACK)
Level 3: Engineering manager
  ↓ (30 min sans ACK)
Level 4: CTO (incidents majeurs)
```

### 4.2 Slack

**Intégration webhook** :
```yaml
slack_configs:
  - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXX'
    channel: '#redis-alerts'

    # Username et icon
    username: 'Alertmanager'
    icon_emoji: ':prometheus:'

    # Titre dynamique
    title: |
      {{ if eq .Status "firing" }}:fire:{{ else }}:white_check_mark:{{ end }}
      {{ .GroupLabels.alertname }}

    # URL du titre
    title_link: 'https://grafana.company.com/d/redis-overview'

    # Couleur
    color: |
      {{ if eq .Status "resolved" }}good
      {{ else if eq .CommonLabels.severity "warning" }}warning
      {{ else }}danger{{ end }}

    # Corps du message
    text: |
      {{ range .Alerts }}
      *Instance:* {{ .Labels.instance }}
      *Severity:* {{ .Labels.severity }}
      *Summary:* {{ .Annotations.summary }}
      {{ if .Annotations.description }}
      *Details:*
      {{ .Annotations.description }}
      {{ end }}
      {{ end }}

    # Actions (boutons)
    actions:
      - type: button
        text: 'View Dashboard'
        url: 'https://grafana.company.com/d/redis-overview'
      - type: button
        text: 'Runbook'
        url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
      - type: button
        text: 'Silence 1h'
        url: '{{ .ExternalURL }}/#/silences/new'

    # Champs additionnels
    fields:
      - title: 'Environment'
        value: '{{ .CommonLabels.env }}'
        short: true
      - title: 'Region'
        value: '{{ .CommonLabels.region }}'
        short: true
      - title: 'Firing'
        value: '{{ .Alerts.Firing | len }}'
        short: true
      - title: 'Resolved'
        value: '{{ .Alerts.Resolved | len }}'
        short: true
```

### 4.3 Email

**Configuration SMTP** :
```yaml
# global config
global:
  smtp_smarthost: 'smtp.company.com:587'
  smtp_from: 'alertmanager@company.com'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'secure_password'
  smtp_require_tls: true

# receiver
receivers:
  - name: 'email-oncall'
    email_configs:
      - to: 'oncall-redis@company.com'
        headers:
          Subject: |
            [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
        html: |
          <!DOCTYPE html>
          <html>
          <head>
            <style>
              .alert { padding: 10px; margin: 10px 0; border-radius: 4px; }
              .critical { background-color: #ffebee; border-left: 4px solid #f44336; }
              .warning { background-color: #fff3e0; border-left: 4px solid #ff9800; }
              .info { background-color: #e3f2fd; border-left: 4px solid #2196f3; }
            </style>
          </head>
          <body>
            <h2>{{ .GroupLabels.alertname }}</h2>
            {{ range .Alerts }}
            <div class="alert {{ .Labels.severity }}">
              <h3>{{ .Labels.instance }}</h3>
              <p><strong>Summary:</strong> {{ .Annotations.summary }}</p>
              <p><strong>Description:</strong></p>
              <pre>{{ .Annotations.description }}</pre>
              <p><strong>Started:</strong> {{ .StartsAt.Format "2006-01-02 15:04:05 MST" }}</p>
              {{ if .Annotations.runbook_url }}
              <p><a href="{{ .Annotations.runbook_url }}">View Runbook</a></p>
              {{ end }}
            </div>
            {{ end }}
          </body>
          </html>
```

### 4.4 Webhook Custom

**Intégration avec système de ticketing** :
```yaml
receivers:
  - name: 'jira-tickets'
    webhook_configs:
      - url: 'https://jira-webhook.company.com/create-ticket'
        send_resolved: true
        http_config:
          bearer_token: 'YOUR_API_TOKEN'

        # Max alerts dans une notification
        max_alerts: 10
```

**Webhook receiver (Python Flask)** :
```python
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

JIRA_URL = "https://jira.company.com/rest/api/2/issue"
JIRA_TOKEN = "your_jira_token"

@app.route('/create-ticket', methods=['POST'])
def create_ticket():
    alert_data = request.json

    for alert in alert_data.get('alerts', []):
        if alert['status'] == 'firing':
            # Créer ticket JIRA
            ticket = {
                "fields": {
                    "project": {"key": "REDIS"},
                    "summary": f"{alert['labels']['alertname']} on {alert['labels']['instance']}",
                    "description": alert['annotations']['description'],
                    "issuetype": {"name": "Incident"},
                    "priority": {"name": "High" if alert['labels']['severity'] == 'critical' else "Medium"},
                    "labels": ["redis", "alertmanager", alert['labels']['severity']]
                }
            }

            response = requests.post(
                JIRA_URL,
                json=ticket,
                headers={
                    "Authorization": f"Bearer {JIRA_TOKEN}",
                    "Content-Type": "application/json"
                }
            )

            if response.status_code == 201:
                print(f"Ticket créé: {response.json()['key']}")

        elif alert['status'] == 'resolved':
            # Résoudre le ticket correspondant
            pass  # Logique de résolution

    return jsonify({"status": "ok"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## 5. Runbooks et Documentation

### 5.1 Structure d'un runbook

**Template** :
```markdown
# Runbook: [Nom de l'alerte]

## Métadonnées
- **Alerte:** RedisDown
- **Severity:** Critical
- **SLA:** Résolution < 15 minutes
- **Dernière mise à jour:** 2024-01-15
- **Responsable:** Équipe Infrastructure

## Symptômes
- Redis instance ne répond plus
- Applications en erreur avec timeout Redis
- Métriques Prometheus: `redis_up == 0`

## Impact
- **Utilisateurs:** 100% des fonctionnalités dépendantes indisponibles
- **Business:** Perte de revenue, impossibilité de traiter les commandes
- **SLA:** Violation du SLA de disponibilité 99.9%

## Diagnostic rapide (< 2 min)
```bash
# 1. Vérifier le process
ssh redis-server
sudo systemctl status redis

# 2. Vérifier les logs récents
sudo journalctl -u redis -n 100 --no-pager

# 3. Vérifier dmesg (OOM killer?)
sudo dmesg | tail -50 | grep -i redis

# 4. Tenter connexion
redis-cli PING
```

## Causes fréquentes
1. **OOM Killer** (70%)
   - Symptôme: Dans dmesg: "Out of memory: Kill process"
   - Cause: Mémoire insuffisante, maxmemory trop élevé

2. **Crash/Segfault** (15%)
   - Symptôme: Core dump dans /var/crash
   - Cause: Bug Redis, corruption mémoire

3. **Kill manuel** (10%)
   - Symptôme: Dans logs: "Received SIGTERM"
   - Cause: Intervention humaine, automation

4. **Problème disque** (5%)
   - Symptôme: "Can't save in background: fork: Cannot allocate memory"
   - Cause: Disque plein, I/O error

## Résolution

### Étape 1: Redémarrage d'urgence
```bash
# Redémarrer Redis
sudo systemctl restart redis

# Vérifier démarrage
sudo systemctl status redis
redis-cli PING
```

### Étape 2: Si échec redémarrage
```bash
# Vérifier config
sudo redis-server /etc/redis/redis.conf --test-memory 1024

# Vérifier permissions
ls -la /var/lib/redis
ls -la /var/log/redis

# Tenter démarrage manuel (debug)
sudo -u redis redis-server /etc/redis/redis.conf
```

### Étape 3: Si OOM détecté
```bash
# 1. Réduire maxmemory temporairement
sudo sed -i 's/maxmemory .*/maxmemory 4gb/' /etc/redis/redis.conf

# 2. Redémarrer
sudo systemctl restart redis

# 3. Planifier scaling urgent
```

### Étape 4: Restauration depuis backup (si nécessaire)
```bash
# 1. Arrêter Redis
sudo systemctl stop redis

# 2. Restaurer dump.rdb
sudo cp /backup/dump.rdb /var/lib/redis/

# 3. Permissions
sudo chown redis:redis /var/lib/redis/dump.rdb

# 4. Démarrer
sudo systemctl start redis
```

## Post-mortem
- [ ] Documenter la cause racine
- [ ] Mettre à jour les runbooks si nécessaire
- [ ] Créer ticket pour actions préventives
- [ ] Communiquer aux stakeholders

## Escalation
Si non résolu en 15 min:
1. Appeler Secondary on-call
2. Créer incident Statuspage
3. Escalader au manager

## Liens utiles
- Dashboard: https://grafana.company.com/d/redis-overview
- Logs: https://kibana.company.com/app/redis
- Wiki: https://wiki.company.com/redis
```

### 5.2 Génération automatique de runbooks

**Script pour générer runbooks depuis annotations** :
```python
#!/usr/bin/env python3
# generate-runbooks.py

import yaml
import os

def generate_runbook(alert):
    alertname = alert['alert']
    annotations = alert['annotations']
    labels = alert['labels']

    runbook = f"""# Runbook: {alertname}

## Métadonnées
- **Alerte:** {alertname}
- **Severity:** {labels.get('severity', 'N/A')}
- **Tier:** {labels.get('tier', 'N/A')}

## Symptômes
{annotations.get('summary', 'N/A')}

## Description
{annotations.get('description', 'N/A')}

## Impact
{annotations.get('impact', 'À documenter')}

## Diagnostic
{annotations.get('diagnostic', 'À documenter')}

## Résolution
{annotations.get('resolution', 'À documenter')}

## Liens
- Dashboard: {annotations.get('dashboard_url', 'N/A')}
- Logs: {annotations.get('logs_url', 'N/A')}
"""

    filename = f"runbooks/{alertname}.md"
    os.makedirs('runbooks', exist_ok=True)

    with open(filename, 'w') as f:
        f.write(runbook)

    print(f"Generated: {filename}")

def main():
    # Charger les règles d'alerte
    with open('prometheus-rules.yml', 'r') as f:
        rules = yaml.safe_load(f)

    for group in rules.get('groups', []):
        for rule in group.get('rules', []):
            if 'alert' in rule:
                generate_runbook(rule)

if __name__ == '__main__':
    main()
```

## 6. Testing et Validation

### 6.1 Tests unitaires des alertes

**Outil: promtool** :
```bash
# Valider syntaxe des règles
promtool check rules /etc/prometheus/rules/*.yml

# Tester une règle avec données fictives
promtool test rules test-alerts.yml
```

**Fichier de test** (`test-alerts.yml`) :
```yaml
# Test RedisDown alert
rule_files:
  - ../prometheus-rules.yml

evaluation_interval: 1m

tests:
  # Test 1: Redis up → pas d'alerte
  - interval: 1m
    input_series:
      - series: 'redis_up{instance="redis-1:9121"}'
        values: '1+0x10'  # Valeur 1 pendant 10 minutes

    alert_rule_test:
      - eval_time: 5m
        alertname: RedisDown
        exp_alerts: []  # Pas d'alerte attendue

  # Test 2: Redis down → alerte après 1 min
  - interval: 1m
    input_series:
      - series: 'redis_up{instance="redis-1:9121"}'
        values: '1 1 0+0x10'  # Down après 2 min

    alert_rule_test:
      - eval_time: 2m
        alertname: RedisDown
        exp_alerts: []  # Pas encore (for: 1m)

      - eval_time: 3m
        alertname: RedisDown
        exp_alerts:
          - exp_labels:
              severity: critical
              instance: redis-1:9121
            exp_annotations:
              summary: "Redis instance redis-1:9121 is DOWN"
```

**Exécution** :
```bash
promtool test rules test-alerts.yml
# Unit Testing:  test-alerts.yml
#   SUCCESS
```

### 6.2 Tests d'intégration (chaos engineering)

**Simuler une panne Redis** :
```bash
#!/bin/bash
# test-redis-down-alert.sh

echo "=== Test: Redis Down Alert ==="

# 1. Arrêter Redis
echo "Stopping Redis..."
sudo systemctl stop redis
echo "Redis stopped at $(date)"

# 2. Attendre l'alerte (1 min for + 30s group_wait)
echo "Waiting 90 seconds for alert..."
sleep 90

# 3. Vérifier Alertmanager
ALERTS=$(curl -s http://alertmanager:9093/api/v2/alerts | jq '.[] | select(.labels.alertname=="RedisDown")')

if [ -n "$ALERTS" ]; then
    echo "✅ SUCCESS: Alert triggered"
    echo "$ALERTS" | jq .
else
    echo "❌ FAILURE: Alert not triggered"
    exit 1
fi

# 4. Redémarrer Redis
echo "Restarting Redis..."
sudo systemctl start redis

# 5. Attendre résolution
echo "Waiting for resolution..."
sleep 60

# 6. Vérifier résolution
ALERTS=$(curl -s http://alertmanager:9093/api/v2/alerts | jq '.[] | select(.labels.alertname=="RedisDown" and .status.state=="active")')

if [ -z "$ALERTS" ]; then
    echo "✅ SUCCESS: Alert resolved"
else
    echo "⚠️  Alert still active"
fi

echo "=== Test Complete ==="
```

### 6.3 Tests de notification

**Script de test notification** :
```bash
#!/bin/bash
# test-notifications.sh

# Générer une alerte test via Alertmanager
curl -X POST http://alertmanager:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[
    {
      "labels": {
        "alertname": "TestAlert",
        "severity": "warning",
        "instance": "test-instance",
        "job": "test"
      },
      "annotations": {
        "summary": "Test alert - please ignore",
        "description": "This is a test alert to validate notifications"
      },
      "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
      "endsAt": "'$(date -u -d '+5 minutes' +%Y-%m-%dT%H:%M:%SZ)'"
    }
  ]'

echo "Test alert sent. Check Slack/PagerDuty/Email."
```

## 7. Métriques sur les Alertes

### 7.1 Self-monitoring des alertes

**Métriques Alertmanager** :
```promql
# Nombre d'alertes actives
alertmanager_alerts{state="active"}

# Nombre d'alertes silencées
alertmanager_silences{state="active"}

# Notifications envoyées
rate(alertmanager_notifications_total[5m])

# Taux d'échec des notifications
rate(alertmanager_notifications_failed_total[5m]) /
rate(alertmanager_notifications_total[5m])

# Latence des notifications
alertmanager_notification_latency_seconds
```

**Dashboard Alertmanager** :
```json
{
  "panels": [
    {
      "title": "Active Alerts",
      "targets": [{
        "expr": "sum(alertmanager_alerts{state='active'}) by (alertname)"
      }]
    },
    {
      "title": "Notification Success Rate",
      "targets": [{
        "expr": "1 - (rate(alertmanager_notifications_failed_total[5m]) / rate(alertmanager_notifications_total[5m]))"
      }]
    }
  ]
}
```

### 7.2 Alertes sur les alertes

```yaml
# Meta-alerting
- alert: AlertmanagerDown
  expr: up{job="alertmanager"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Alertmanager is down"

- alert: TooManyAlerts
  expr: sum(alertmanager_alerts{state="active"}) > 50
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Too many active alerts ({{ $value }})"
    description: "Possible alert storm or systemic issue"

- alert: AlertNotificationFailing
  expr: |
    rate(alertmanager_notifications_failed_total[5m]) /
    rate(alertmanager_notifications_total[5m]) > 0.1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Alert notifications failing"
    description: "{{ $value | humanizePercentage }} of notifications are failing"
```

## 8. Best Practices Récapitulatives

### 8.1 Do's ✅

- **Alerter sur les symptômes**, pas les causes
- **Une alerte = une action** claire et documentée
- **Runbooks** obligatoires pour alertes critiques
- **Tester** régulièrement les alertes (chaos engineering)
- **Tuner** agressivement les seuils (éviter faux positifs)
- **Grouper** les alertes similaires
- **Documenter** toutes les résolutions d'alertes
- **Review** mensuelle des alertes (ajouter/supprimer/ajuster)
- **Silencer** pendant les maintenances
- **Monitorer** les métriques d'alerting elles-mêmes

### 8.2 Don'ts ❌

- ❌ Alerter sur des métriques non actionnables
- ❌ Créer des alertes "pour information"
- ❌ Avoir plus de 3 niveaux de sévérité
- ❌ Alertes sans runbook (critiques)
- ❌ Seuils arbitraires non basés sur l'expérience
- ❌ Période "for" trop courte (< 1 min)
- ❌ Répéter les alertes trop fréquemment
- ❌ Ignorer les alertes sans les corriger
- ❌ Laisser des alertes en mode "flapping"
- ❌ Alerter sur des événements planifiés

### 8.3 Checklist de production

- [ ] Alertes Tier 1 (critiques) définies et testées
- [ ] Alertes Tier 2 (warnings) configurées
- [ ] Runbooks créés pour toutes les alertes critiques
- [ ] Alertmanager configuré avec HA
- [ ] Notifications PagerDuty/Slack fonctionnelles
- [ ] Routing correct par sévérité
- [ ] Inhibition rules configurées
- [ ] Silences programmables (API)
- [ ] Tests automatisés des alertes
- [ ] Dashboard de monitoring des alertes
- [ ] Escalation policy définie
- [ ] Documentation à jour
- [ ] Review mensuelle planifiée

## 9. Métriques de Qualité des Alertes

### 9.1 KPIs à tracker

```promql
# Ratio signal/bruit
(count(ALERTS{alertstate="firing", severity="critical", resolved="false"})) /
(count(ALERTS{alertstate="firing"}))

# Temps moyen de résolution (MTTR)
avg(ALERTS_resolved_timestamp - ALERTS_firing_timestamp) by (alertname)

# Taux de faux positifs
count(ALERTS{resolved="true", duration="<5m"}) /
count(ALERTS{resolved="true"})

# Alert fatigue indicator
rate(alertmanager_alerts{state="active"}[24h]) >
avg_over_time(alertmanager_alerts{state="active"}[7d]) * 2
```

### 9.2 Objectifs

| Métrique | Cible | Excellent | Critique |
|----------|-------|-----------|----------|
| **Faux positifs** | < 5% | < 2% | > 20% |
| **MTTR** | < 15 min | < 5 min | > 60 min |
| **Alertes/jour** | < 10 | < 5 | > 50 |
| **Couverture incidents** | 100% | 100% | < 90% |

## Conclusion

Un système d'alerting efficace est l'équilibre entre :

1. **Sensibilité** : Détecter tous les vrais problèmes
2. **Spécificité** : Éviter les faux positifs
3. **Actionnabilité** : Chaque alerte a une action claire
4. **Maintenabilité** : Review et amélioration continue

**Principes finaux** :
- Moins d'alertes = Mieux (si couverture complète)
- Alerte = Symptôme utilisateur + Action requise
- Runbooks = Obligatoires pour critiques
- Testing = Garant de la fiabilité
- Review = Amélioration continue

Un bon système d'alerting Redis permet de dormir tranquille tout en ayant la certitude d'être réveillé si nécessaire.

---

**Prochaine section** : 13.7 - Logs et audit trail (logging, conformité, forensics)

⏭️ [Logs et audit trail](/13-monitoring-observabilite/07-logs-audit-trail.md)
