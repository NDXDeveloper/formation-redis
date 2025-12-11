🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.2 Métriques clés : Hit ratio, Fragmentation, Évictions

## Introduction

En production Redis, trois métriques se distinguent par leur impact direct sur la performance, la disponibilité et les coûts d'infrastructure. Ces métriques sont les **indicateurs de santé** qui, lorsqu'elles dégradent, précèdent généralement un incident majeur.

### Pourquoi ces trois métriques sont critiques

| Métrique | Impact direct | Symptôme visible | Coût business |
|----------|---------------|------------------|---------------|
| **Hit Ratio** | Performance applicative | Latence ↑, timeouts | Perte d'utilisateurs |
| **Fragmentation** | Gaspillage mémoire | OOM, crashes | Surcoût infra 2-3× |
| **Évictions** | Perte de données | Cache misses ↑, bugs | Charge DB ↑, incidents |

### La triade de la santé Redis

```
┌─────────────────────────────────────────────────┐
│              Application Layer                  │
│         (latence, availability, UX)             │
└────────────────┬────────────────────────────────┘
                 │
        ┌────────┴────────┐
        │   Hit Ratio     │ ← Efficacité du cache
        │   (Cache hit %) │
        └────────┬────────┘
                 │
    ┌────────────┴────────────┐
    │                         │
┌───┴──────────┐     ┌────────┴────────┐
│ Fragmentation│     │   Évictions     │
│ (Mémoire)    │     │   (Capacité)    │
└──────────────┘     └─────────────────┘
     │                      │
     └──────────┬───────────┘
                │
       ┌────────┴────────┐
       │  Infrastructure │
       │  (RAM, Coûts)   │
       └─────────────────┘
```

## 1. Hit Ratio : L'efficacité du cache

### 1.1 Définition et calcul

Le **hit ratio** (ou taux de succès du cache) mesure le pourcentage de requêtes de lecture trouvant la donnée dans Redis.

#### Formule de base

```
Hit Ratio = Hits / (Hits + Misses) × 100
```

#### Métriques Redis impliquées

```bash
# Redis INFO stats
keyspace_hits:8547123      # Clés trouvées
keyspace_misses:1245632    # Clés non trouvées

# Calcul
hit_ratio = 8547123 / (8547123 + 1245632) = 87.3%
```

#### Requête Prometheus

```promql
# Hit ratio sur 5 minutes
100 * (
  rate(redis_keyspace_hits_total{instance="$instance"}[5m]) /
  (
    rate(redis_keyspace_hits_total{instance="$instance"}[5m]) +
    rate(redis_keyspace_misses_total{instance="$instance"}[5m])
  )
)
```

### 1.2 Objectifs par use case

| Use Case | Hit Ratio cible | Justification |
|----------|----------------|---------------|
| **Cache HTTP/API** | > 85% | Réduire la charge backend |
| **Session Store** | > 95% | Sessions actives toujours présentes |
| **Full-page cache** | > 80% | Pages populaires en cache |
| **Query cache (SQL)** | > 90% | Requêtes répétitives |
| **CDN backend cache** | > 75% | Contenu statique |
| **Rate limiting** | N/A | Pas de notion de hit/miss |
| **Job Queue** | N/A | Métrique non pertinente |

### 1.3 Anatomie d'un hit et d'un miss

#### Qu'est-ce qu'un HIT ?

Commandes qui incrémentent `keyspace_hits` :
```
GET key         → Clé existe → HIT
HGET hash f     → Hash et field existent → HIT
LINDEX list 0   → Liste existe et index valide → HIT
SISMEMBER s m   → Set existe et membre présent → HIT
ZRANK ss m      → Sorted set existe et membre présent → HIT
EXISTS key      → Clé existe → HIT
```

#### Qu'est-ce qu'un MISS ?

```
GET key         → Clé n'existe pas → MISS
HGET hash f     → Hash n'existe pas → MISS
EXISTS key      → Clé n'existe pas → MISS
```

#### Commandes qui n'affectent PAS hit/miss

```
SET key value   → Write operation (ni hit ni miss)
DEL key         → Write operation
INCR key        → Write operation
LPUSH list val  → Write operation
KEYS *          → Scan operation
INFO            → Admin operation
```

### 1.4 Facteurs impactant le hit ratio

#### 1. TTL (Time To Live)

**TTL trop court** :
```
TTL: 60 secondes
Fréquence d'accès: toutes les 90 secondes
→ Hit ratio catastrophique (~0%)
```

**TTL optimal** :
```
TTL ≥ 2 × Intervalle_moyen_entre_requêtes

Exemple :
Accès moyen toutes les 30s → TTL ≥ 60s
```

**Cas réel** :
```python
# ❌ Mauvais : TTL trop court
redis.setex(f"user:{user_id}", 300, user_data)  # 5 min
# Si le user est actif (requêtes toutes les 10s),
# la clé expire alors qu'elle est chaude

# ✅ Bon : TTL adapté
redis.setex(f"user:{user_id}", 1800, user_data)  # 30 min
# Couvre une session utilisateur typique
```

#### 2. Capacité mémoire insuffisante

**Scénario** : Évictions forcées
```
Maxmemory: 4GB
Dataset: 6GB de données chaudes
→ Évictions permanentes → Hit ratio effondré
```

**Calcul du dimensionnement** :
```
RAM_nécessaire = Working_set_size × 1.5

Working set = Données accédées dans la fenêtre de TTL
```

**Exemple** :
```
10M clés actives
Taille moyenne : 500 bytes
Working set : 10M × 500 = 5GB
RAM nécessaire : 5GB × 1.5 = 7.5GB
```

#### 3. Pattern d'accès imprévisible

**Cache inutile si** :
```
# Pattern aléatoire (ex: UUID sans pattern)
GET user:a1b2c3d4-5678-90ef-ghij-klmnopqrstuv
GET user:b2c3d4e5-6789-01fg-hijk-lmnopqrstuvw
GET user:c3d4e5f6-7890-12gh-ijkl-mnopqrstuvwx
→ Chaque requête est unique → Hit ratio ~0%
```

**Cache efficace si** :
```
# Pattern avec hotspots
GET product:12345  (100 requêtes/sec)
GET product:67890  (80 requêtes/sec)
GET product:11111  (60 requêtes/sec)
→ 20% des clés = 80% du trafic → Hit ratio élevé
```

#### 4. Politique d'éviction inadaptée

```
# ❌ Mauvaise config pour cache
maxmemory-policy volatile-lru
# Si 50% des clés n'ont pas de TTL → Jamais évictées → Saturation

# ✅ Bonne config pour cache
maxmemory-policy allkeys-lru
# Toutes les clés peuvent être évictées
```

### 1.5 Diagnostic d'un hit ratio dégradé

#### Étape 1 : Établir une baseline

```promql
# Hit ratio moyen sur 7 jours
avg_over_time(
  (
    rate(redis_keyspace_hits_total[5m]) /
    (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
  )[7d:]
)
```

**Baseline typique** :
```
Heure   | Hit Ratio
--------|----------
02:00   | 92%      ← Nuit, cache chaud
08:00   | 85%      ← Pic matin, cache froid
14:00   | 91%      ← Après-midi, cache stable
20:00   | 88%      ← Soirée, nouveau trafic
```

#### Étape 2 : Identifier la dégradation

**Dégradation progressive** :
```
Jour 1: 90%
Jour 2: 88%
Jour 3: 85%
Jour 4: 82%
→ Dataset qui croît + mémoire fixe → Évictions
```

**Dégradation brutale** :
```
12:00 → 91%
12:15 → 47%  ← Drop soudain
→ Déploiement applicatif ? Changement de pattern ?
```

#### Étape 3 : Corréler avec d'autres métriques

```promql
# Graphique de corrélation
{
  hit_ratio: rate(redis_keyspace_hits_total[5m]) /
             (rate(redis_keyspace_hits_total[5m]) +
              rate(redis_keyspace_misses_total[5m])),
  evictions: rate(redis_evicted_keys_total[5m]),
  memory_pct: redis_memory_used_bytes / redis_memory_max_bytes * 100
}
```

**Pattern typique** :
```
Memory_pct ↑ → Évictions ↑ → Hit_ratio ↓
```

#### Étape 4 : Analyser les patterns de clés

```bash
# Identifier les clés les plus "missées"
redis-cli MONITOR | grep -i "get" | awk '{print $4}' | sort | uniq -c | sort -rn | head -20

# Exemple de sortie
1547 "GET" "product:99999"    ← Produit inexistant ?
892  "GET" "user:deleted_123" ← Utilisateur supprimé ?
456  "GET" "session:expired"  ← Session expirée ?
```

### 1.6 Stratégies d'amélioration du hit ratio

#### Stratégie 1 : Augmentation intelligente des TTL

**Analyse** :
```sql
-- Analyser les patterns d'accès
SELECT
  key_prefix,
  AVG(time_between_access) as avg_interval,
  RECOMMENDED_TTL = AVG(time_between_access) * 2
FROM access_logs
GROUP BY key_prefix;
```

**Implémentation** :
```python
# Cache adaptatif basé sur la fréquence
def cache_with_adaptive_ttl(key, fetcher):
    # Récupérer historique d'accès
    access_count = redis.get(f"{key}:access_count") or 0

    # TTL adaptatif
    if access_count > 100:      # Très populaire
        ttl = 3600              # 1 heure
    elif access_count > 10:     # Populaire
        ttl = 1800              # 30 min
    else:                       # Peu accédé
        ttl = 300               # 5 min

    data = redis.get(key)
    if not data:
        data = fetcher()
        redis.setex(key, ttl, data)
        redis.incr(f"{key}:access_count")

    return data
```

#### Stratégie 2 : Cache warming (pré-chauffage)

**Cas d'usage** : Après un redémarrage ou un déploiement

```python
# Script de warming
def warm_cache():
    # Top 1000 produits les plus consultés
    hot_products = db.query("""
        SELECT product_id
        FROM product_views
        WHERE created_at > NOW() - INTERVAL '7 days'
        GROUP BY product_id
        ORDER BY COUNT(*) DESC
        LIMIT 1000
    """)

    for product_id in hot_products:
        product_data = db.get_product(product_id)
        redis.setex(
            f"product:{product_id}",
            3600,
            json.dumps(product_data)
        )

    logging.info(f"Cache warmed with {len(hot_products)} products")
```

**Scheduler** :
```yaml
# Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: redis-cache-warmer
spec:
  schedule: "0 2 * * *"  # 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: warmer
            image: app:latest
            command: ["python", "warm_cache.py"]
```

#### Stratégie 3 : Cache-aside avec fallback

```python
def get_user(user_id):
    # Tentative 1 : Redis
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)

    if user:
        # HIT
        return json.loads(user)

    # MISS → Fallback DB
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")

    if user:
        # Cacher pour la prochaine fois
        redis.setex(cache_key, 1800, json.dumps(user))
    else:
        # Cacher les "not found" pour éviter les requêtes répétées
        # (Cache Penetration protection)
        redis.setex(cache_key, 60, json.dumps({"_not_found": True}))

    return user
```

#### Stratégie 4 : Augmentation de la capacité mémoire

**Calcul du ROI** :
```
Coût actuel :
- RAM : 8GB @ $50/mois
- Hit ratio : 75%
- 25% requêtes → DB (100ms latence)
- 1000 req/s × 25% = 250 req/s vers DB
- Surcoût DB : CPU élevé, slowdowns

Coût après upgrade :
- RAM : 16GB @ $100/mois (+$50)
- Hit ratio : 95%
- 5% requêtes → DB (100ms latence)
- 1000 req/s × 5% = 50 req/s vers DB
- Économie : Réduction DB scaling (-$200/mois)

ROI : -$50 (Redis) + $200 (DB) = +$150/mois
```

### 1.7 Alerting sur le hit ratio

#### Règle Prometheus de base

```yaml
- alert: RedisHitRatioLow
  expr: |
    (
      rate(redis_keyspace_hits_total[5m]) /
      (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
    ) < 0.80
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Hit ratio faible sur {{ $labels.instance }}"
    description: "Hit ratio: {{ $value | humanizePercentage }} (< 80%)"
```

#### Règle avancée avec baseline

```yaml
- alert: RedisHitRatioDegradation
  expr: |
    (
      (
        rate(redis_keyspace_hits_total[5m]) /
        (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
      )
      /
      avg_over_time(
        (
          rate(redis_keyspace_hits_total[5m]) /
          (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
        )[7d:]
      )
    ) < 0.90
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Dégradation du hit ratio sur {{ $labels.instance }}"
    description: "Hit ratio actuel: {{ $value | humanizePercentage }} de la baseline 7j"
```

#### Alerting par segment

```yaml
# Différencier par use case
- alert: SessionStoreHitRatioLow
  expr: |
    (
      rate(redis_keyspace_hits_total{redis_role="session"}[5m]) /
      (rate(redis_keyspace_hits_total{redis_role="session"}[5m]) +
       rate(redis_keyspace_misses_total{redis_role="session"}[5m]))
    ) < 0.95
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Session store hit ratio critique"
    description: "Sessions non trouvées → Déconnexions utilisateurs"
```

## 2. Fragmentation mémoire : L'optimisation cachée

### 2.1 Comprendre la fragmentation

#### Définition

La **fragmentation mémoire** survient quand la mémoire allouée par l'allocateur (jemalloc) ne correspond plus à la mémoire réellement utilisée par Redis.

```
┌─────────────────────────────────────────────┐
│  Mémoire physique (used_memory_rss)         │
│                                             │
│  ┌────────────────────────────────────┐     │
│  │ Mémoire utilisée (used_memory)     │     │
│  │                                    │     │
│  │  [Données Redis]                   │     │
│  │                                    │     │
│  └────────────────────────────────────┘     │
│                                             │
│  [Trous] [Trous] [Trous] [Trous]            │ ← Fragmentation
│                                             │
└─────────────────────────────────────────────┘
```

#### Calcul

```
mem_fragmentation_ratio = used_memory_rss / used_memory
```

**Métriques Redis** :
```bash
# Redis INFO memory
used_memory:2147483648          # 2GB (données logiques)
used_memory_rss:2952790016      # 2.75GB (mémoire physique)
mem_fragmentation_ratio:1.37    # 37% de fragmentation
mem_fragmentation_bytes:805306368  # 768MB gaspillés
```

### 2.2 Échelle d'interprétation détaillée

| Ratio | État | Signification | Impact mémoire | Action |
|-------|------|---------------|----------------|--------|
| < 1.0 | 🔴 DANGER | Swap actif | Variable | **URGENT** : RAM insuffisante |
| 1.0 - 1.1 | 🟢 EXCELLENT | Fragmentation minimale | < 10% overhead | Monitoring normal |
| 1.1 - 1.3 | 🟢 OPTIMAL | Fragmentation acceptable | 10-30% overhead | RAS |
| 1.3 - 1.5 | 🟡 ACCEPTABLE | Fragmentation modérée | 30-50% overhead | Surveiller tendance |
| 1.5 - 2.0 | 🟠 PROBLÉMATIQUE | Fragmentation élevée | 50-100% overhead | Planifier action |
| 2.0 - 3.0 | 🔴 CRITIQUE | Perte importante | 100-200% overhead | Redémarrage recommandé |
| > 3.0 | 🔴 SÉVÈRE | Gaspillage majeur | > 200% overhead | Redémarrage urgent |

### 2.3 Causes de la fragmentation

#### Cause 1 : Workload volatil

**Scénario** : Créations/suppressions fréquentes de clés de tailles variées

```python
# Pattern générant de la fragmentation
for i in range(1000000):
    # Créer une clé de taille aléatoire
    size = random.randint(100, 10000)
    redis.set(f"data:{i}", "x" * size)

    # Supprimer une clé aléatoire
    if random.random() > 0.5:
        redis.delete(f"data:{random.randint(0, i)}")

# Résultat : Trous mémoire partout
```

**Visualisation** :
```
Temps T0: [100B][200B][150B][300B][250B]
Temps T1: [100B][____][150B][____][250B]  ← Suppressions
Temps T2: [100B][80B_][150B][____][250B]  ← Nouvelle clé 80B ne remplit pas le trou 200B
```

#### Cause 2 : Mix de petites et grosses clés

```python
# Fragmentation par hétérogénéité
redis.set("tiny", "x")                    # 1 byte
redis.set("small", "x" * 100)             # 100 bytes
redis.set("medium", "x" * 10000)          # 10KB
redis.set("large", "x" * 1000000)         # 1MB

# L'allocateur alloue par "classes de taille"
# → Gaspillage dans chaque allocation
```

#### Cause 3 : Évictions massives

```python
# Scénario : Atteinte de maxmemory
# → Redis évicte 10000 clés d'un coup
# → 10000 trous mémoire
# → Fragmentation soudaine
```

#### Cause 4 : Longue durée d'uptime

```
Jour 1:   fragmentation_ratio = 1.05
Jour 30:  fragmentation_ratio = 1.35
Jour 90:  fragmentation_ratio = 1.58
Jour 180: fragmentation_ratio = 1.87
→ Accumulation naturelle
```

### 2.4 Impact de la fragmentation

#### Impact 1 : Surconsommation mémoire

**Exemple réel** :
```
Configuration serveur : 16GB RAM
Maxmemory Redis : 12GB
Fragmentation : 1.8

Mémoire réellement consommée :
12GB × 1.8 = 21.6GB
→ Serveur en OOM !
```

#### Impact 2 : Déclenchement prématuré des évictions

```
Maxmemory : 4GB
Used_memory : 3.5GB (87.5%, pas encore d'éviction)
Fragmentation : 1.6
Used_memory_rss : 5.6GB

→ Système en swap
→ Redis ralentit
→ Clients timeout
```

#### Impact 3 : Coût infrastructure

```
Scénario A : Sans fragmentation
- Dataset : 10GB
- RAM nécessaire : 15GB (marge 50%)
- Coût : $150/mois

Scénario B : Avec fragmentation (ratio 2.0)
- Dataset : 10GB
- Fragmentation : 10GB supplémentaires
- RAM nécessaire : 30GB (20GB + marge)
- Coût : $300/mois

Surcoût : $150/mois soit 100%
```

### 2.5 Monitoring de la fragmentation

#### Dashboard Grafana

**Panel 1 : Fragmentation Ratio (Gauge)**
```promql
redis_mem_fragmentation_ratio{instance="$instance"}
```

**Seuils** :
- Vert : < 1.3
- Jaune : 1.3 - 1.5
- Orange : 1.5 - 2.0
- Rouge : > 2.0

**Panel 2 : Mémoire gaspillée (Graph)**
```promql
# Mémoire fragmentée en bytes
redis_mem_fragmentation_bytes{instance="$instance"}

# Ou calculée
redis_memory_used_rss_bytes - redis_memory_used_bytes
```

**Panel 3 : Tendance fragmentation (Graph)**
```promql
# Fragmentation sur 30 jours
redis_mem_fragmentation_ratio{instance="$instance"}[30d]
```

**Panel 4 : Corrélation fragmentation vs évictions**
```promql
{
  fragmentation: redis_mem_fragmentation_ratio,
  evictions: rate(redis_evicted_keys_total[5m])
}
```

#### Alertes Prometheus

**Alerte de base** :
```yaml
- alert: RedisFragmentationHigh
  expr: redis_mem_fragmentation_ratio > 1.5
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Fragmentation mémoire élevée sur {{ $labels.instance }}"
    description: "Ratio: {{ $value }} (> 1.5) - {{ $value | humanize }}% de mémoire gaspillée"
```

**Alerte critique** :
```yaml
- alert: RedisFragmentationCritical
  expr: redis_mem_fragmentation_ratio > 2.0
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Fragmentation mémoire CRITIQUE sur {{ $labels.instance }}"
    description: |
      Ratio: {{ $value }}
      Mémoire gaspillée: {{ query "redis_mem_fragmentation_bytes{instance='{{ $labels.instance }}'}" | humanize1024 }}
      Action: Redémarrage recommandé
```

**Alerte tendance** :
```yaml
- alert: RedisFragmentationIncreasing
  expr: |
    (
      redis_mem_fragmentation_ratio -
      redis_mem_fragmentation_ratio offset 7d
    ) > 0.3
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Fragmentation en augmentation sur {{ $labels.instance }}"
    description: "Augmentation de {{ $value }} sur 7 jours"
```

### 2.6 Solutions à la fragmentation

#### Solution 1 : Active Defragmentation (Redis 4.0+)

**Configuration optimale** :
```conf
# redis.conf

# Activer la défragmentation active
activedefrag yes

# Ne démarrer que si fragmentation significative
active-defrag-ignore-bytes 100mb        # Ignorer si < 100MB fragmentés
active-defrag-threshold-lower 10        # Démarrer à 10% fragmentation
active-defrag-threshold-upper 100       # Mode agressif à 100%

# Contrôle de la charge CPU
active-defrag-cycle-min 1               # Min 1% CPU
active-defrag-cycle-max 25              # Max 25% CPU (ajuster selon charge)

# Agressivité du scanning
active-defrag-max-scan-fields 1000      # Scan 1000 fields par cycle
```

**Monitoring de la défragmentation** :
```bash
# Redis INFO memory
active_defrag_running:1                  # Défrag en cours
active_defrag_hits:1547896              # Succès de défrag
active_defrag_misses:245678             # Échecs
active_defrag_key_hits:15478            # Clés défragmentées
active_defrag_key_misses:2456           # Clés non-défragmentables
```

**Efficacité** :
```
Avant : fragmentation_ratio = 1.8
Après 24h de defrag : fragmentation_ratio = 1.3
Gain : 27% de mémoire récupérée
```

**Limitations** :
- Augmente légèrement la latence (scanning actif)
- Pas efficace si workload trop volatil
- CPU overhead 1-25%

#### Solution 2 : Redémarrage planifié

**Quand redémarrer** :
- Fragmentation > 2.0 pendant > 7 jours
- Active defrag inefficace
- Maintenance planifiée

**Stratégie de redémarrage** :

**Option A : Redémarrage avec réplication (Zero Downtime)**
```bash
# 1. Promouvoir un replica
redis-cli -h replica1 REPLICAOF NO ONE

# 2. Rediriger le trafic applicatif vers replica1

# 3. Redémarrer l'ancien master
systemctl restart redis

# 4. Attendre chargement complet

# 5. Reconfigurer en replica
redis-cli -h old-master REPLICAOF replica1 6379

# 6. Une fois sync, repromouvoir en master
redis-cli -h old-master REPLICAOF NO ONE
redis-cli -h replica1 REPLICAOF old-master 6379
```

**Option B : Redémarrage avec Sentinel (Automatique)**
```bash
# Sentinel gère le failover automatiquement
systemctl restart redis

# Sentinel détecte la panne et promouvoit un replica
# Après redémarrage, l'instance rejoint comme replica
# Puis peut être repromue manuellement si nécessaire
```

**Option C : Redémarrage Cluster (Rolling restart)**
```bash
# Redémarrer les slaves d'abord, puis les masters
for node in slave1 slave2 slave3 master1 master2 master3; do
  echo "Redémarrage de $node..."
  redis-cli -h $node SHUTDOWN NOSAVE
  sleep 60  # Attendre le redémarrage
  # Vérifier que le node est revenu
  redis-cli -h $node PING
  sleep 300  # Attendre stabilisation réplication
done
```

#### Solution 3 : Tuning de l'allocateur

**jemalloc tuning** (avancé) :
```bash
# Variables d'environnement au démarrage Redis
export JEMALLOC_CONF="narenas:4,lg_tcache_max:15"
redis-server /etc/redis/redis.conf
```

**Paramètres** :
- `narenas:4` : Nombre d'arènes (réduire si mono-thread)
- `lg_tcache_max:15` : Taille max du thread cache

**Attention** : Réglages très spécifiques, tester avant production

#### Solution 4 : Optimisation du workload

**Stratégie 1 : Réduire la volatilité**
```python
# ❌ Mauvais : Volatilité élevée
def cache_user(user_id):
    redis.setex(f"user:{user_id}", 300, data)  # TTL court
    # → Créations/suppressions fréquentes

# ✅ Meilleur : Stabilité
def cache_user(user_id):
    redis.setex(f"user:{user_id}", 3600, data)  # TTL plus long
    # → Moins de churn
```

**Stratégie 2 : Homogénéiser les tailles**
```python
# ❌ Mauvais : Tailles hétérogènes
redis.set("small", "x" * 10)
redis.set("huge", "x" * 1000000)

# ✅ Meilleur : Tailles similaires ou séparation
# Option A : Compression pour homogénéiser
import zlib
data_compressed = zlib.compress(data.encode())
redis.set(key, data_compressed)

# Option B : Instances séparées
redis_small.set("small", data)   # Instance pour petites clés
redis_large.set("large", data)   # Instance pour grosses clés
```

### 2.7 Cas d'étude : Fragmentation en production

**Contexte** :
- E-commerce, 50M de sessions actives
- Redis 16GB RAM
- Fragmentation passée de 1.2 à 2.4 en 3 mois

**Analyse** :
```bash
# Pattern identifié
INFO commandstats | grep set
cmdstat_setex:calls=15478963,usec_per_call=45.2

INFO keyspace
db0:keys=50000000,expires=50000000,avg_ttl=900000  # 15 min TTL

# Calcul
50M clés × 4 renouvellements/heure = 200M SET/heure
→ Churn massif → Fragmentation
```

**Solution appliquée** :
```python
# Avant : TTL court, churn élevé
redis.setex(f"session:{sid}", 900, data)  # 15 min

# Après : TTL long, lazy cleanup
redis.setex(f"session:{sid}", 3600, data)  # 60 min

# Background job de cleanup basé sur activité
def cleanup_inactive_sessions():
    for session_id in get_inactive_sessions():
        redis.delete(f"session:{session_id}")
```

**Résultats** :
- Fragmentation : 2.4 → 1.4 en 2 semaines
- RAM économisée : 6GB (37%)
- Coût : -$500/mois

## 3. Évictions : Quand le cache déborde

### 3.1 Comprendre les évictions

#### Définition

Une **éviction** survient quand Redis supprime une clé pour libérer de la mémoire, car `maxmemory` est atteint.

```
┌────────────────────────────────────────┐
│  RAM: [████████████████████] 100%      │
│                                        │
│  Nouvelle clé arrive →                 │
│  ┌─────────────────────────────┐       │
│  │ Redis doit supprimer une    │       │
│  │ clé existante (éviction)    │       │
│  └─────────────────────────────┘       │
│                                        │
│  Politique: maxmemory-policy           │
└────────────────────────────────────────┘
```

#### Différence éviction vs expiration

| Aspect | Éviction | Expiration |
|--------|----------|------------|
| **Déclencheur** | Mémoire pleine | TTL expiré |
| **Intentionnel** | Non (problème) | Oui (normal) |
| **Métrique** | `evicted_keys` | `expired_keys` |
| **Impact** | Perte de données chaudes | Nettoyage prévu |
| **Action** | Augmenter RAM | RAS |

### 3.2 Métriques d'éviction

```bash
# Redis INFO stats
evicted_keys:154789              # Nombre total d'évictions
expired_keys:8547123             # Expirations (comparaison)

# Taux d'éviction
rate(redis_evicted_keys_total[5m])
```

**Objectif** : `evicted_keys` devrait rester à **0** dans un système sain

### 3.3 Politiques d'éviction

#### Vue d'ensemble

```
┌─────────────────────────────────────────────────┐
│            maxmemory-policy                     │
├─────────────────────────────────────────────────┤
│                                                 │
│  Basées sur LRU/LFU (toutes clés)               │
│  ├─ allkeys-lru    : LRU global                 │
│  ├─ allkeys-lfu    : LFU global (Redis 4+)      │
│  └─ allkeys-random : Aléatoire global           │
│                                                 │
│  Basées sur LRU/LFU (clés avec TTL)             │
│  ├─ volatile-lru   : LRU sur expires            │
│  ├─ volatile-lfu   : LFU sur expires            │
│  ├─ volatile-random: Aléatoire sur expires      │
│  └─ volatile-ttl   : Plus court TTL             │
│                                                 │
│  Pas d'éviction                                 │
│  └─ noeviction     : Erreur OOM (défaut)        │
│                                                 │
└─────────────────────────────────────────────────┘
```

#### Comparaison LRU vs LFU

**LRU (Least Recently Used)** :
```
Clé A : Accédée il y a 10 secondes  (récente)
Clé B : Accédée il y a 60 secondes  (ancienne)
→ Éviction de B (moins récente)

Problème : Une clé accédée massivement hier
mais pas aujourd'hui sera conservée
```

**LFU (Least Frequently Used)** :
```
Clé A : 1000 accès sur 24h  (fréquente)
Clé B : 5 accès sur 24h     (peu fréquente)
→ Éviction de B (moins fréquente)

Avantage : Conserve les vraies clés chaudes
```

#### Choix de la politique

| Use Case | Politique recommandée | Justification |
|----------|----------------------|---------------|
| Cache pur | `allkeys-lru` ou `allkeys-lfu` | Tout est évictable |
| Session store | `volatile-lru` | Sessions avec TTL |
| Mixed (cache + permanent) | `allkeys-lru` | Balance optimal |
| Queue/Stream | `noeviction` | Perte de données inacceptable |
| Rate limiting | `allkeys-lru` | Entrées anciennes moins importantes |

### 3.4 Impact des évictions

#### Impact 1 : Dégradation du hit ratio

```
Scénario :
- 1000 req/s
- Hit ratio sans éviction : 95%
- Évictions : 100 clés/s

Calcul :
Nouvelles clés évictées = 100/s
Requêtes affectées = 100/s (maintenant en miss)
Hit ratio dégradé = (950 - 100) / 1000 = 85%

Perte : 10 points de hit ratio
```

#### Impact 2 : Charge sur la base de données

```
Avant évictions :
- 1000 req/s vers Redis
- 50 req/s vers DB (5% miss)
- DB CPU : 20%

Avec évictions (100/s) :
- 1000 req/s vers Redis
- 150 req/s vers DB (15% miss)
- DB CPU : 60%

Impact : DB CPU × 3
```

#### Impact 3 : Latence applicative

```python
# Sans éviction
def get_user(user_id):
    user = redis.get(f"user:{user_id}")  # 1ms (hit)
    return user

# Avec éviction
def get_user(user_id):
    user = redis.get(f"user:{user_id}")  # 1ms (miss)
    if not user:
        user = db.query(...)             # +100ms (DB query)
        redis.setex(...)                 # +1ms
    return user

Latence : 1ms → 102ms (×100 pour les requêtes évictées)
```

### 3.5 Diagnostic des évictions

#### Étape 1 : Confirmer les évictions

```bash
# Évictions actuelles
redis-cli INFO stats | grep evicted_keys
evicted_keys:154789

# Taux d'éviction sur 5 minutes
redis-cli --stat | grep evicted
# Ou via Prometheus
rate(redis_evicted_keys_total[5m])
```

#### Étape 2 : Analyser la cause

**Requête Grafana multi-métriques** :
```promql
{
  memory_used_pct: (redis_memory_used_bytes / redis_memory_max_bytes) * 100,
  evictions_rate: rate(redis_evicted_keys_total[5m]),
  keys_total: redis_db_keys,
  hit_ratio: rate(redis_keyspace_hits_total[5m]) /
             (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
}
```

**Patterns à identifier** :

**Pattern 1 : Saturation chronique**
```
memory_used_pct: 100% permanent
evictions_rate: 50-100/s constant
→ Sous-dimensionnement
```

**Pattern 2 : Pics d'éviction**
```
08:00-09:00 : evictions_rate = 200/s
Rest of day : evictions_rate = 0
→ Pic de charge matinal
```

**Pattern 3 : Évictions croissantes**
```
Semaine 1 : 10 évictions/s
Semaine 2 : 25 évictions/s
Semaine 3 : 50 évictions/s
→ Dataset qui croît
```

#### Étape 3 : Identifier les clés évictées

```bash
# Monitoring en temps réel
redis-cli MONITOR | grep -i "del"

# Ou via logs (si log-level debug)
tail -f /var/log/redis/redis.log | grep "evict"
```

**Impossible de voir précisément quelles clés** : Redis ne log pas les évictions individuelles

**Workaround** : Instrumenter l'application
```python
# Wrapper Redis avec logging
class InstrumentedRedis:
    def __init__(self, redis_client):
        self.redis = redis_client

    def get(self, key):
        value = self.redis.get(key)
        if value is None:
            # Log potential eviction
            logger.warning(f"Cache miss for key: {key} - Possible eviction")
        return value
```

### 3.6 Solutions aux évictions

#### Solution 1 : Augmentation de maxmemory

**Calcul du dimensionnement optimal** :
```
# Méthode 1 : Basé sur le working set
working_set_size = nb_clés_actives × taille_moyenne_clé

# Méthode 2 : Basé sur le taux d'accès
données_accédées_dans_TTL = (requêtes/s × TTL) × taille_moyenne

# Exemple :
# 1000 req/s, TTL 3600s, taille moyenne 1KB
working_set = 1000 × 3600 × 1024 = 3.6GB

# Ajout marge 50%
maxmemory_optimal = 3.6GB × 1.5 = 5.4GB
```

**Implémentation** :
```bash
# Mise à jour dynamique (sans redémarrage)
redis-cli CONFIG SET maxmemory 5gb

# Persisté dans redis.conf
echo "maxmemory 5gb" >> /etc/redis/redis.conf
```

#### Solution 2 : Optimisation des TTL

**Stratégie** : Réduire le working set en diminuant les TTL des données froides

```python
# Analyse : Identifier les patterns d'accès
def analyze_key_access():
    for key in redis.scan_iter():
        last_access = redis.object("idletime", key)  # Secondes depuis dernier accès
        ttl = redis.ttl(key)

        if last_access > ttl * 0.8:
            # Clé rarement accédée mais avec TTL long
            print(f"Candidate for shorter TTL: {key}")

# Ajustement
# Avant
redis.setex("cold_data", 3600, data)  # 1h

# Après
redis.setex("cold_data", 600, data)   # 10 min
```

#### Solution 3 : Compression des données

```python
import zlib
import json

# Avant compression
data = {"user": "john", "email": "john@example.com", ...}
redis.set("user:123", json.dumps(data))
# Taille : ~500 bytes

# Avec compression
data_json = json.dumps(data)
data_compressed = zlib.compress(data_json.encode(), level=6)
redis.set("user:123", data_compressed)
# Taille : ~150 bytes (70% économie)

# Lecture
data_compressed = redis.get("user:123")
data_json = zlib.decompress(data_compressed).decode()
data = json.loads(data_json)
```

**Gain** :
```
Avant : 10M clés × 500 bytes = 5GB
Après : 10M clés × 150 bytes = 1.5GB
Économie : 3.5GB (70%)
```

#### Solution 4 : Sharding (Redis Cluster)

**Quand sharder** :
- Dataset > 50GB
- Évictions permanentes malgré max RAM
- Besoin de scalabilité horizontale

```
┌──────────────────────────────────────────────┐
│         Application                          │
└─────────────┬────────────────────────────────┘
              │
    ┌─────────┴─────────┐
    │  Redis Cluster    │
    │  (Hash Slots)     │
    └─────────┬─────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
┌───▼───┐ ┌───▼───┐ ┌───▼───┐
│ Node1 │ │ Node2 │ │ Node3 │
│ 16GB  │ │ 16GB  │ │ 16GB  │
└───────┘ └───────┘ └───────┘

Total : 48GB de cache distribué
```

### 3.7 Alerting sur les évictions

#### Alerte de base

```yaml
- alert: RedisEvictingKeys
  expr: rate(redis_evicted_keys_total[5m]) > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Redis éviction de clés sur {{ $labels.instance }}"
    description: |
      Taux d'éviction: {{ $value }} clés/sec
      Cause probable: Mémoire insuffisante
```

#### Alerte conditionnelle (selon la politique)

```yaml
- alert: RedisEvictionWithNoEvictionPolicy
  expr: |
    rate(redis_evicted_keys_total[5m]) > 0
    and
    redis_config_maxmemory_policy == "noeviction"
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "CRITIQUE: Évictions avec politique noeviction"
    description: |
      Des évictions se produisent alors que la politique est noeviction
      Cela ne devrait JAMAIS arriver - Bug ou config incorrecte
```

#### Alerte de tendance

```yaml
- alert: RedisEvictionRateIncreasing
  expr: |
    rate(redis_evicted_keys_total[5m]) >
    rate(redis_evicted_keys_total[5m] offset 1h) * 2
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Taux d'éviction en augmentation rapide"
    description: "Évictions doublées en 1 heure - Dataset croissant ?"
```

## 4. Corrélation des trois métriques

### 4.1 Le triangle de la performance

```
         Hit Ratio ↓
              │
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
Fragmentation ↑    Évictions ↑
    │                   │
    └─────────┬─────────┘
              │
              ▼
        Problème Mémoire
```

### 4.2 Patterns de corrélation courants

#### Pattern 1 : Cascade de dégradation

```
Étape 1 : Fragmentation ↑ (1.2 → 1.8)
         ↓
Étape 2 : Mémoire effective ↓ (moins de place pour données)
         ↓
Étape 3 : Évictions ↑ (manque de place)
         ↓
Étape 4 : Hit ratio ↓ (clés évictées = misses)
```

**Requête Prometheus pour détecter** :
```promql
# Alerte si les 3 métriques dégradées simultanément
(
  redis_mem_fragmentation_ratio > 1.5
  and
  rate(redis_evicted_keys_total[5m]) > 10
  and
  (
    rate(redis_keyspace_hits_total[5m]) /
    (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
  ) < 0.85
)
```

#### Pattern 2 : Évictions sans fragmentation

```
Fragmentation : 1.2 (bon)
Évictions : 100/s (élevé)
Hit ratio : 75% (bas)

→ Diagnostic : Maxmemory trop petit, pas de fragmentation
→ Action : Augmenter RAM
```

#### Pattern 3 : Fragmentation sans éviction

```
Fragmentation : 2.0 (élevé)
Évictions : 0
Hit ratio : 92% (bon)

→ Diagnostic : Gaspillage mémoire mais capacité suffisante
→ Action : Active defrag ou redémarrage planifié (moins urgent)
```

### 4.3 Dashboard de corrélation Grafana

**Vue unifiée recommandée** :

```
┌─────────────────────────────────────────────────────┐
│  Panel 1 : Métriques actuelles (Single Stat)        │
│  ┌─────────────┬──────────────┬─────────────┐       │
│  │  Hit Ratio  │ Fragmentation│  Évictions  │       │
│  │    89.2%    │     1.42     │   12.5/s    │       │
│  └─────────────┴──────────────┴─────────────┘       │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Panel 2 : Tendances sur 24h (Graph)                │
│                                                     │
│  Hit Ratio ──────                                   │
│  Fragmentation ─ ─ ─                                │
│  Évictions ·······                                  │
│                                                     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Panel 3 : Corrélation (Heatmap)                    │
│                                                     │
│  [Matrice de corrélation entre les 3 métriques]     │
│                                                     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Panel 4 : Actions recommandées (Table)             │
│  ┌────────────────────────────────────────────┐     │
│  │ État         │ Action                      │     │
│  ├────────────────────────────────────────────┤     │
│  │ ⚠️ Évictions  │ Augmenter maxmemory        │     │
│  │ ⚠️ Fragm. 1.5 │ Activer active defrag      │     │
│  └────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

**Code JSON du dashboard** (extrait) :
```json
{
  "panels": [
    {
      "title": "Métriques Critiques",
      "targets": [
        {
          "expr": "100 * (rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])))",
          "legendFormat": "Hit Ratio %"
        },
        {
          "expr": "redis_mem_fragmentation_ratio * 100",
          "legendFormat": "Fragmentation ×100"
        },
        {
          "expr": "rate(redis_evicted_keys_total[5m])",
          "legendFormat": "Évictions/s"
        }
      ]
    }
  ]
}
```

## 5. Stratégies opérationnelles avancées

### 5.1 Matrice de décision

| Métriques | Diagnostic | Actions | Urgence |
|-----------|-----------|---------|---------|
| Évictions > 0, Fragm < 1.5, Hit ratio < 90% | Sous-dimensionnement | Augmenter RAM | 🔴 Haute |
| Évictions > 0, Fragm > 1.8, Hit ratio < 85% | Sous-dim + Fragmentation | Redémarrage + RAM ↑ | 🔴 Haute |
| Évictions = 0, Fragm > 2.0, Hit ratio > 90% | Fragmentation isolée | Active defrag ou redémarrage planifié | 🟡 Moyenne |
| Évictions = 0, Fragm < 1.5, Hit ratio < 85% | TTL inadaptés | Ajuster TTL, warming | 🟡 Moyenne |
| Évictions = 0, Fragm < 1.5, Hit ratio > 95% | Optimal | Monitoring continu | 🟢 Basse |

### 5.2 Playbook d'intervention

#### Scénario 1 : Évictions actives

```bash
# 1. Confirmer les évictions
redis-cli INFO stats | grep evicted_keys

# 2. Vérifier la mémoire
redis-cli INFO memory | grep -E "used_memory:|maxmemory:|used_memory_peak"

# 3. Analyser la politique
redis-cli CONFIG GET maxmemory-policy

# 4. Solution immédiate : Augmenter maxmemory (si RAM dispo)
redis-cli CONFIG SET maxmemory 8gb

# 5. Solution permanente : Provisionner plus de RAM ou sharder
```

#### Scénario 2 : Fragmentation critique

```bash
# 1. Mesurer la fragmentation
redis-cli INFO memory | grep mem_fragmentation_ratio

# 2. Activer active defrag si pas déjà fait
redis-cli CONFIG SET activedefrag yes

# 3. Monitorer l'évolution
watch -n 60 'redis-cli INFO memory | grep -E "mem_fragmentation|active_defrag"'

# 4. Si pas d'amélioration après 24h : Planifier redémarrage
```

#### Scénario 3 : Hit ratio dégradé

```bash
# 1. Vérifier les évictions
redis-cli INFO stats | grep evicted_keys

# 2. Analyser les TTL
redis-cli INFO keyspace

# 3. Identifier les patterns
redis-cli --bigkeys

# 4. Ajuster selon la cause
# - Évictions → Augmenter RAM
# - TTL courts → Allonger TTL
# - Pattern imprévisible → Revoir stratégie caching
```

### 5.3 Automation via Runbooks

**Exemple de runbook Ansible** :
```yaml
---
- name: Redis Health Check and Remediation
  hosts: redis_servers
  tasks:
    - name: Get Redis metrics
      shell: redis-cli INFO stats
      register: redis_info

    - name: Check for evictions
      set_fact:
        evictions: "{{ redis_info.stdout | regex_search('evicted_keys:(\\d+)', '\\1') | first }}"

    - name: Alert if evictions detected
      debug:
        msg: "WARNING: {{ evictions }} evictions detected"
      when: evictions | int > 0

    - name: Get fragmentation ratio
      shell: redis-cli INFO memory | grep mem_fragmentation_ratio | cut -d: -f2
      register: fragmentation

    - name: Enable active defrag if fragmentation > 1.5
      command: redis-cli CONFIG SET activedefrag yes
      when: fragmentation.stdout | float > 1.5
```

## Conclusion

Les trois métriques clés — **hit ratio**, **fragmentation**, et **évictions** — forment le triangle de la santé Redis. Une surveillance proactive de ces métriques permet :

1. **Prévention** : Détecter les problèmes avant impact utilisateur
2. **Optimisation** : Dimensionner correctement l'infrastructure
3. **Économies** : Éviter le surprovisionnement ou les incidents coûteux

**Points à retenir** :
- Hit ratio > 90% = cache efficace
- Fragmentation < 1.5 = mémoire utilisée efficacement
- Évictions = 0 = capacité suffisante

**Prochaine action** : Mettre en place les dashboards et alertes recommandés dans cette section.

---

**Prochaine section** : 13.3 - Redis Exporter et Prometheus (configuration avancée)

⏭️ [Redis Exporter et Prometheus](/13-monitoring-observabilite/03-redis-exporter-prometheus.md)
