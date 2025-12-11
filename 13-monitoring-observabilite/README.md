🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 13 : Monitoring et Observabilité

## Introduction

Le monitoring Redis ne se limite pas à vérifier que le service répond. En production, une observabilité efficace est la différence entre détecter un problème avant qu'il n'impacte les utilisateurs et subir un incident majeur. Ce module explore les stratégies, métriques et outils essentiels pour maintenir Redis en conditions optimales.

### Pourquoi l'observabilité Redis est critique

Redis, de par sa nature in-memory et son rôle souvent central dans l'architecture (cache, session store, queue), présente des caractéristiques particulières :

- **Effet multiplicateur** : Une dégradation Redis impacte généralement plusieurs services downstream
- **Volatilité des données** : La nature éphémère des données nécessite un monitoring proactif de la mémoire
- **Single-thread** : Un seul thread bloquant peut paralyser l'ensemble du service
- **Pas de "buffer"** : Contrairement aux bases de données sur disque, une saturation mémoire Redis est immédiate et critique

### Les trois piliers de l'observabilité Redis

#### 1. **Métriques (Metrics)**
Les indicateurs quantitatifs qui révèlent l'état et la performance du système :
- Utilisation mémoire et fragmentation
- Hit/Miss ratio du cache
- Latence des commandes
- Throughput (ops/sec)
- Évictions et expirations
- Réplication lag
- Connexions clients

#### 2. **Logs**
Les événements discrets qui racontent l'histoire du système :
- Démarrage/arrêt du service
- Changements de configuration
- Warnings et erreurs
- Événements de réplication
- Commandes lentes (slowlog)
- Failover et élections Sentinel/Cluster

#### 3. **Traces**
Le parcours des requêtes à travers le système :
- Latence applicative vs latence Redis
- Corrélation entre services
- Identification des hot paths
- Analyse des patterns d'accès

## Architecture de monitoring Redis en production

### Stack de monitoring recommandé

```
┌─────────────────────────────────────────────────────┐
│                   Grafana                           │
│            (Visualisation & Alerting)               │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────┴───────────────────────────────────┐
│                 Prometheus                          │
│          (Métriques Time-Series DB)                 │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────┴───────────────────────────────────┐
│              Redis Exporter                         │
│         (Collecteur de métriques)                   │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────┴───────────────────────────────────┐
│            Redis Instance(s)                        │
│     (Master, Replicas, Sentinel, Cluster)           │
└─────────────────────────────────────────────────────┘
```

### Composants clés

#### Redis Exporter
- **Rôle** : Interroge périodiquement Redis via `INFO` et expose les métriques au format Prometheus
- **Installation** : Container Docker ou binaire sur le même host
- **Port par défaut** : 9121
- **Overhead** : Minimal (~1-2% CPU), exécute `INFO` toutes les N secondes

#### Prometheus
- **Rôle** : Collecte, stocke et agrège les métriques time-series
- **Scraping** : Pull-based, interroge les exporters à intervalle régulier (15-60s)
- **Rétention** : Configurable (15j-90j typiquement)
- **Alerting** : Évalue les règles d'alerte et déclenche Alertmanager

#### Grafana
- **Rôle** : Visualisation, dashboards, alerting complémentaire
- **Dashboards pré-construits** : Communauté active avec dashboards Redis prêts à l'emploi
- **Intégration** : Prometheus comme data source

## Métriques critiques à monitorer

### 1. Métriques de santé système

#### Disponibilité
```
redis_up
```
- **Signification** : Le service Redis est-il accessible ?
- **Valeur** : 1 (up) ou 0 (down)
- **Alerte** : Immédiate si `redis_up == 0`

#### Uptime
```
redis_uptime_in_seconds
```
- **Signification** : Temps depuis le dernier redémarrage
- **Usage** : Détection de redémarrages inattendus
- **Alerte** : Si uptime < 300 secondes (redémarrage récent)

### 2. Métriques mémoire (CRITIQUES)

#### Utilisation mémoire
```
redis_memory_used_bytes
redis_memory_max_bytes
redis_memory_used_rss_bytes
```

**Calcul du taux d'utilisation** :
```
(redis_memory_used_bytes / redis_memory_max_bytes) * 100
```

**Seuils recommandés** :
- **Warning** : > 70%
- **Critical** : > 85%
- **Emergency** : > 95%

#### Fragmentation mémoire
```
redis_mem_fragmentation_ratio
```

**Interprétation** :
- **< 1.0** : Swap ou overcommit (DANGER)
- **1.0 - 1.5** : Optimal
- **1.5 - 2.0** : Fragmentation modérée
- **> 2.0** : Fragmentation sévère (perte mémoire importante)

**Formule** :
```
fragmentation_ratio = used_memory_rss / used_memory
```

### 3. Métriques de performance

#### Hit Ratio (Cache)
```
rate(redis_keyspace_hits_total[5m]) /
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
```

**Objectifs** :
- **Cache** : > 90% souhaitable
- **Session Store** : > 95% attendu
- **Queue** : N/A (métrique non pertinente)

#### Évictions
```
rate(redis_evicted_keys_total[5m])
```

**Signification** : Clés supprimées par manque de mémoire (politique d'éviction)
**Alerte** :
- Évictions > 0 avec `maxmemory-policy noeviction` → Problème grave
- Évictions croissantes → Dimensionnement insuffisant

#### Expirations
```
rate(redis_expired_keys_total[5m])
```

**Différence avec évictions** : Expirations = TTL naturel (normal), Évictions = manque de RAM (problème)

### 4. Métriques de latence

#### Commandes instantanées
```
redis_instantaneous_ops_per_sec
```

**Usage** : Détecter les pics de charge
**Baseline** : Établir une baseline en production pour détecter les anomalies

#### Latency moyenne
```
rate(redis_commands_duration_seconds_total[5m]) /
rate(redis_commands_processed_total[5m])
```

**Objectifs** :
- **P50** : < 1ms
- **P95** : < 5ms
- **P99** : < 10ms

### 5. Métriques de connexions

#### Clients connectés
```
redis_connected_clients
```

**Surveillance** :
- Pic soudain → Possible connection leak applicatif
- Approche de `maxclients` → Risque de refus de connexion
- Chute brutale → Possibles timeouts réseau

#### Clients bloqués
```
redis_blocked_clients
```

**Causes** : Commandes bloquantes (`BLPOP`, `BRPOP`, `BRPOPLPUSH`, `XREAD BLOCK`)
**Normal** : Si vous utilisez des queues avec blocking
**Alerte** : Croissance continue = possibles workers morts

### 6. Métriques de persistance

#### Dernière sauvegarde RDB
```
redis_rdb_last_save_timestamp_seconds
```

**Alerte** : Si `time() - redis_rdb_last_save_timestamp_seconds > 7200` (2h sans backup)

#### AOF en cours
```
redis_aof_rewrite_in_progress
```

**Impact** : Performance dégradée pendant le rewrite
**Usage** : Corréler avec des pics de latence

### 7. Métriques de réplication

#### Replication lag
```
redis_replication_lag_seconds
```

**Criticalité** : HAUTE pour les architectures Master-Replica
**Objectifs** :
- **< 1s** : Excellent
- **1-5s** : Acceptable
- **> 10s** : Problématique
- **> 60s** : Critique

#### Replicas connectés
```
redis_connected_slaves
```

**Alerte** : Si nombre attendu ≠ nombre réel

### 8. Métriques Cluster (si applicable)

#### Cluster state
```
redis_cluster_state
```

**Valeurs** :
- **1** : OK
- **0** : FAIL (slots non couverts ou nœuds down)

#### Slots migrés
```
redis_cluster_slots_migrating
redis_cluster_slots_importing
```

**Usage** : Monitoring pendant resharding

## Configuration Prometheus

### Exemple de configuration (prometheus.yml)

```yaml
global:
  scrape_interval: 15s      # Fréquence de collecte
  evaluation_interval: 15s   # Fréquence d'évaluation des règles

scrape_configs:
  # Redis instances
  - job_name: 'redis'
    static_configs:
      - targets:
          - 'redis-exporter-01:9121'  # Production Master
          - 'redis-exporter-02:9121'  # Production Replica 1
          - 'redis-exporter-03:9121'  # Production Replica 2
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
      - source_labels: [__address__]
        regex: 'redis-exporter-01:.*'
        target_label: redis_role
        replacement: 'master'
      - source_labels: [__address__]
        regex: 'redis-exporter-0[23]:.*'
        target_label: redis_role
        replacement: 'replica'

  # Redis Sentinel
  - job_name: 'redis-sentinel'
    static_configs:
      - targets:
          - 'sentinel-exporter-01:9355'
          - 'sentinel-exporter-02:9355'
          - 'sentinel-exporter-03:9355'

  # Redis Cluster (si applicable)
  - job_name: 'redis-cluster'
    static_configs:
      - targets:
          - 'cluster-node-01:9121'
          - 'cluster-node-02:9121'
          - 'cluster-node-03:9121'
          - 'cluster-node-04:9121'
          - 'cluster-node-05:9121'
          - 'cluster-node-06:9121'
    relabel_configs:
      - source_labels: [__address__]
        target_label: cluster_node
```

### Règles d'alerte Prometheus (alert.rules.yml)

```yaml
groups:
  - name: redis_alerts
    interval: 30s
    rules:
      # Disponibilité
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis instance {{ $labels.instance }} est down"
          description: "L'instance Redis {{ $labels.instance }} ne répond plus depuis > 1 minute."

      # Mémoire
      - alert: RedisMemoryHigh
        expr: (redis_memory_used_bytes / redis_memory_max_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Utilisation mémoire Redis élevée sur {{ $labels.instance }}"
          description: "Utilisation mémoire à {{ $value | humanize }}% sur {{ $labels.instance }}"

      - alert: RedisMemoryCritical
        expr: (redis_memory_used_bytes / redis_memory_max_bytes) * 100 > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Utilisation mémoire Redis CRITIQUE sur {{ $labels.instance }}"
          description: "Utilisation mémoire à {{ $value | humanize }}% - Risque OOM imminent!"

      - alert: RedisFragmentationHigh
        expr: redis_mem_fragmentation_ratio > 2.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Fragmentation mémoire élevée sur {{ $labels.instance }}"
          description: "Fragmentation ratio: {{ $value }} (> 2.0) - Considérer un restart"

      # Évictions
      - alert: RedisEvictingKeys
        expr: rate(redis_evicted_keys_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis éviction de clés sur {{ $labels.instance }}"
          description: "{{ $value }} clés/sec évictées - Mémoire insuffisante"

      # Hit Ratio
      - alert: RedisCacheHitRateLow
        expr: |
          rate(redis_keyspace_hits_total[5m]) /
          (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) < 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio faible sur {{ $labels.instance }}"
          description: "Hit ratio: {{ $value | humanizePercentage }} (< 80%)"

      # Réplication
      - alert: RedisReplicationLagHigh
        expr: redis_replication_lag_seconds > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag élevé sur {{ $labels.instance }}"
          description: "Lag: {{ $value }}s (> 10s) - Replica en retard"

      - alert: RedisReplicaDisconnected
        expr: redis_connected_slaves < 2  # Si on attend 2 replicas
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Replica Redis déconnecté sur {{ $labels.instance }}"
          description: "Seulement {{ $value }} replica(s) connecté(s)"

      # Persistance
      - alert: RedisRDBSaveDelayed
        expr: time() - redis_rdb_last_save_timestamp_seconds > 7200
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pas de sauvegarde RDB récente sur {{ $labels.instance }}"
          description: "Dernière sauvegarde: {{ $value | humanizeDuration }} ago"

      # Connexions
      - alert: RedisConnectionsHigh
        expr: redis_connected_clients > 900  # Si maxclients = 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Nombre de connexions élevé sur {{ $labels.instance }}"
          description: "{{ $value }} connexions actives (proche du max)"

      # Cluster
      - alert: RedisClusterStateFailure
        expr: redis_cluster_state == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis Cluster en état FAIL sur {{ $labels.instance }}"
          description: "Le cluster n'est pas opérationnel - Vérifier les slots"
```

## Configuration Grafana

### Dashboards communautaires recommandés

**1. Redis Dashboard for Prometheus Redis Exporter (ID: 763)**
- Dashboard le plus populaire
- Couverture complète des métriques
- Visualisations bien conçues

**2. Redis Overview (ID: 11835)**
- Vue consolidée multi-instances
- Bon pour les architectures Master-Replica

**3. Redis Cluster Dashboard (ID: 11692)**
- Spécialisé pour Redis Cluster
- Visualisation des slots et resharding

### Variables Grafana recommandées

```
# Instance Redis (multi-sélection)
instance = label_values(redis_up, instance)

# Environnement
env = label_values(redis_up, env)

# Rôle Redis
role = label_values(redis_up, redis_role)

# Cluster (si applicable)
cluster = label_values(redis_up, cluster_name)
```

### Panels critiques à inclure

#### Panel 1 : Vue d'ensemble (Single Stat)
```
# Instances UP
count(redis_up == 1)

# Mémoire moyenne utilisée
avg(redis_memory_used_bytes / redis_memory_max_bytes) * 100

# Hit Ratio moyen
avg(
  rate(redis_keyspace_hits_total[5m]) /
  (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
)

# OPS/sec total
sum(rate(redis_commands_processed_total[1m]))
```

#### Panel 2 : Utilisation mémoire (Graph)
```
# Par instance
(redis_memory_used_bytes{instance=~"$instance"} / redis_memory_max_bytes) * 100

# Avec seuils (thresholds) :
- > 70% : Jaune
- > 85% : Orange
- > 95% : Rouge
```

#### Panel 3 : Hit/Miss ratio (Graph)
```
# Hit ratio
rate(redis_keyspace_hits_total{instance=~"$instance"}[5m]) /
(rate(redis_keyspace_hits_total{instance=~"$instance"}[5m]) +
 rate(redis_keyspace_misses_total{instance=~"$instance"}[5m]))

# Hits bruts
rate(redis_keyspace_hits_total{instance=~"$instance"}[5m])

# Misses bruts
rate(redis_keyspace_misses_total{instance=~"$instance"}[5m])
```

#### Panel 4 : Latency & Throughput (Graph)
```
# Ops/sec
rate(redis_commands_processed_total{instance=~"$instance"}[1m])

# Latency moyenne (ms)
rate(redis_commands_duration_seconds_total{instance=~"$instance"}[5m]) /
rate(redis_commands_processed_total{instance=~"$instance"}[5m]) * 1000
```

#### Panel 5 : Connexions clients (Graph)
```
redis_connected_clients{instance=~"$instance"}

# Avec annotation si maxclients configuré
```

#### Panel 6 : Évictions & Expirations (Graph)
```
# Évictions (problème)
rate(redis_evicted_keys_total{instance=~"$instance"}[5m])

# Expirations (normal)
rate(redis_expired_keys_total{instance=~"$instance"}[5m])
```

#### Panel 7 : Réplication (Graph - si Master-Replica)
```
# Lag en secondes
redis_replication_lag_seconds{instance=~"$instance"}

# Replicas connectés
redis_connected_slaves{instance=~"$instance"}
```

#### Panel 8 : Fragmentation (Graph)
```
redis_mem_fragmentation_ratio{instance=~"$instance"}

# Ligne de référence à 1.5
```

## Stratégies de monitoring par environnement

### Production
- **Scrape interval** : 15s
- **Rétention** : 30-90 jours
- **Alerting** : Actif avec astreinte
- **Dashboards** : Temps réel + historique
- **Logs** : Centralisés (ELK, Loki)
- **SLO/SLA** : Définis et trackés

### Staging
- **Scrape interval** : 30s
- **Rétention** : 15-30 jours
- **Alerting** : Notifications Slack/Teams
- **Dashboards** : Simplifiés
- **Logs** : Rétention courte

### Développement
- **Scrape interval** : 60s
- **Rétention** : 7 jours
- **Alerting** : Optionnel
- **Dashboards** : Basiques
- **Logs** : Local

## Bonnes pratiques opérationnelles

### 1. Définir des baselines
- Collecter les métriques pendant 2-4 semaines
- Identifier les patterns normaux (jour/nuit, semaine/weekend)
- Établir des seuils basés sur la réalité, pas sur des valeurs arbitraires

### 2. Monitoring proactif vs réactif
- **Proactif** : Détecter les tendances avant le problème (mémoire croissante)
- **Réactif** : Alerter sur les problèmes immédiats (service down)
- **Équilibre** : 70% proactif, 30% réactif

### 3. Alert fatigue
- **Éviter** : Trop d'alertes = désensibilisation
- **Regrouper** : Alertes similaires en une seule
- **Contextualiser** : Inclure les actions recommandées dans l'annotation
- **Escalade** : Warning → Critical → Page

### 4. Corrélation des métriques
Ne jamais analyser une métrique isolément :
- Évictions ↑ + Mémoire ↑ = Sous-dimensionnement
- Latency ↑ + Blocked clients ↑ = Commandes lentes bloquantes
- Hit ratio ↓ + Évictions ↑ = TTL trop courts ou cache trop petit

### 5. Documentation des incidents
Pour chaque alerte déclenchée :
- Contexte : Qu'observait-on ?
- Impact : Quels services affectés ?
- Cause : Root cause identifiée ?
- Action : Qu'avons-nous fait ?
- Prévention : Comment éviter la répétition ?

## Outils complémentaires

### Redis Insight
- **Usage** : Exploration visuelle, debugging
- **Avantages** : GUI native Redis, support Redis Stack
- **Limites** : Pas de time-series, pas d'alerting

### Datadog / New Relic / Dynatrace
- **Usage** : Monitoring APM complet
- **Avantages** : Corrélation applicative, traces distribuées
- **Limites** : Coût élevé

### Custom scripts
- **Usage** : Monitoring spécifique métier
- **Exemple** : Vérifier l'âge des messages dans une queue
- **Implémentation** : Cron + script + exposition Prometheus

## Checklist de monitoring Redis

- [ ] Redis Exporter déployé et fonctionnel
- [ ] Prometheus scrape correctement configuré
- [ ] Dashboard Grafana avec métriques critiques
- [ ] Alertes définies (mémoire, disponibilité, réplication)
- [ ] Baseline établie pour les métriques clés
- [ ] Runbooks documentés pour chaque alerte
- [ ] Logs centralisés et corrélés avec métriques
- [ ] Tests réguliers des alertes (chaos engineering)
- [ ] SLO/SLA définis et mesurés
- [ ] Revue mensuelle des alertes et métriques

## Conclusion

Un monitoring efficace de Redis repose sur :

1. **Visibilité** : Collecter les bonnes métriques aux bons intervalles
2. **Contexte** : Comprendre ce qui est normal pour votre workload
3. **Réactivité** : Alerter avant que l'impact utilisateur ne survienne
4. **Amélioration continue** : Ajuster les seuils et alertes basés sur l'expérience

Le monitoring n'est pas une configuration "fire and forget" : c'est un processus vivant qui évolue avec votre infrastructure et vos besoins métier.

---

**Prochaines sections du module :**
- 13.1 : Analyse détaillée de `Redis INFO`
- 13.2 : Deep dive sur les métriques critiques
- 13.3 : Configuration avancée Prometheus
- 13.4 : Dashboards Grafana sur-mesure
- 13.5 : Latency Doctor et troubleshooting
- 13.6 : Stratégies d'alerting intelligentes
- 13.7 : Logs, audit trail et conformité

⏭️ [Redis INFO : Comprendre toutes les métriques](/13-monitoring-observabilite/01-redis-info-comprendre-metriques.md)
