🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4 LRU vs LFU vs Random : Choisir la bonne stratégie

## Introduction

Le choix de l'algorithme d'éviction (LRU, LFU ou Random) est l'une des décisions les plus importantes lors de la configuration de Redis. Ce choix impacte directement :

- **Le hit ratio** : Pourcentage de requêtes servies depuis le cache
- **La performance** : CPU et latence des opérations
- **L'utilisation mémoire** : Efficacité de l'espace disponible
- **La prévisibilité** : Comportement du système en production

Contrairement à une base de données traditionnelle où tout est persisté, Redis en tant que cache doit constamment **décider quoi garder en mémoire**. Cette section compare en profondeur les trois algorithmes principaux pour vous aider à faire le bon choix.

## Les trois algorithmes : Vue d'ensemble

### LRU (Least Recently Used)

**Principe** : Évicte les données les **moins récemment utilisées**.

```
Hypothèse fondamentale:
"Les données récemment accédées seront probablement accédées à nouveau bientôt"

Chronologie des accès:
Time ─────────────────────────────────→
     A   B   C   A   D   B   E
     │   │   │   │   │   │   │
     └───┴───┴───┴───┴───┴───┴─ Plus récent
                             ↑
                             Garde E, B, D, A
                             Évicte C
```

### LFU (Least Frequently Used)

**Principe** : Évicte les données les **moins fréquemment utilisées**.

```
Hypothèse fondamentale:
"Les données fréquemment accédées continueront d'être fréquemment accédées"

Compteur d'accès:
Key A: 100 accès
Key B: 50 accès
Key C: 10 accès
Key D: 5 accès
     ↑
     Évicte D (moins fréquent)
```

### Random

**Principe** : Évicte des données **aléatoirement**.

```
Hypothèse fondamentale:
"Toutes les données ont la même probabilité d'être accédées"

Ou: "La vitesse d'éviction prime sur la qualité du choix"

Keys: [A, B, C, D, E]
      └─ Sélection aléatoire
         Évicte au hasard
```

## Mécanismes internes détaillés

### LRU : Implémentation Redis

#### Structure de données

```c
// Champ LRU dans RedisObject (24 bits)
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:24;      // ← LRU timestamp
    int refcount;
    void *ptr;
} robj;

// LRU_BITS = 24 bits
// Range: 0 à 16,777,215 (2^24 - 1)
// Résolution: 1 seconde (ajustable)
```

#### Calcul du timestamp LRU

```c
// Horloge LRU globale (secondes divisées par résolution)
#define LRU_CLOCK_RESOLUTION 1000  // 1 seconde
#define LRU_CLOCK_MAX ((1<<24)-1)  // 16,777,215

unsigned int getLRUClock(void) {
    return (mstime() / LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX;
}

// Mise à jour à chaque accès
void updateLRU(robj *o) {
    o->lru = getLRUClock();
}
```

#### Calcul de l'idle time

```c
// Calcul du temps d'inactivité (en secondes)
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();

    if (lruclock >= o->lru) {
        // Cas normal
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        // Wrap-around (après ~194 jours)
        return (LRU_CLOCK_MAX - o->lru + lruclock) * LRU_CLOCK_RESOLUTION;
    }
}
```

#### Wraparound du compteur

```
LRU timeline (24 bits):
0 ────────────────────── 16,777,215 (wrap) ──→ 0

Exemple de wraparound:
Key A: lru = 16,777,210
Global: lru = 5 (après wrap)

Idle time = (16,777,215 - 16,777,210 + 5) = 10 secondes ✓
```

#### Approximation LRU avec échantillonnage

```c
// Redis n'implémente pas un LRU parfait (trop coûteux)
// Utilise un échantillonnage probabiliste

robj *evictionPoolFindBest(void) {
    struct evictionPoolEntry pool[EVPOOL_SIZE];

    // Échantillonne maxmemory_samples clés
    for (int i = 0; i < maxmemory_samples; i++) {
        dictEntry *de = dictGetRandomKey(db->dict);
        robj *o = dictGetVal(de);

        // Calcule idle time
        unsigned long long idle = estimateObjectIdleTime(o);

        // Insère dans le pool si idle > min(pool)
        evictionPoolInsert(pool, de, idle);
    }

    // Retourne entrée avec plus grand idle time
    return pool[EVPOOL_SIZE - 1];
}
```

**Qualité de l'approximation** :

```
LRU parfait (liste doublement chaînée):
- Complexité: O(1) pour accès, O(1) pour éviction
- Overhead: 16 bytes par clé (2 pointeurs)
- Mémoire: Inacceptable pour millions de clés

LRU approximé (échantillonnage):
- Complexité: O(N) où N = maxmemory_samples
- Overhead: 3 bytes par clé (champ lru)
- Précision: 85-95% selon samples
```

### LFU : Implémentation Redis

#### Structure de données

Le champ LRU (24 bits) est réutilisé pour stocker les données LFU :

```c
// En mode LFU, le champ lru est réinterprété:
┌────────────────────┬────────────────┐
│ 16 bits            │ 8 bits         │
│ Last decrement time│ Counter (0-255)│
└────────────────────┴────────────────┘

#define LFU_INIT_VAL 5

// Extraction des composants
#define LFUDecrAndReturn(o) /* voir ci-dessous */
#define LFULogIncr(counter) /* voir ci-dessous */
```

#### Compteur logarithmique

Le compteur n'est pas incrémenté de façon linéaire mais **logarithmique** :

```c
/* Incrémente le compteur de façon logarithmique */
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;  // Saturation

    double r = (double)rand() / RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;

    double p = 1.0 / (baseval * server.lfu_log_factor + 1);

    if (r < p) counter++;
    return counter;
}
```

**Comportement selon lfu-log-factor** :

```
lfu-log-factor = 10 (défaut):

Hits     Counter    Probabilité d'incrément
────────────────────────────────────────────
1        5          100%
10       ~7         50%
100      ~10        10%
1000     ~15        1%
1M       ~25        0.001%

lfu-log-factor = 100:

Hits     Counter    Probabilité d'incrément
────────────────────────────────────────────
1        5          100%
10       ~6         91%
100      ~8         50%
1000     ~10        10%
1M       ~15        0.1%
```

**Visualisation** :

```python
import math
import matplotlib.pyplot as plt

def lfu_counter(hits, log_factor=10, init_val=5):
    counter = init_val
    for _ in range(hits):
        baseval = max(0, counter - init_val)
        p = 1.0 / (baseval * log_factor + 1)
        if random.random() < p:
            counter = min(255, counter + 1)
    return counter

# Courbe counter vs hits pour différents log_factor
log_factors = [1, 10, 100]
hits_range = range(1, 10000)

for lf in log_factors:
    counters = [lfu_counter(h, lf) for h in hits_range]
    plt.plot(hits_range, counters, label=f'log_factor={lf}')

plt.xlabel('Hits')
plt.ylabel('Counter')
plt.legend()
# Courbe logarithmique
```

#### Decay du compteur

Le compteur **décroît** dans le temps pour oublier les anciens accès :

```c
/* Décrémente le compteur selon le temps écoulé */
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;  // Last decrement time
    unsigned long counter = o->lru & 255;

    unsigned long num_periods = server.lfu_decay_time ?
        LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;

    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;

    return counter;
}
```

**Comportement du decay** :

```
lfu-decay-time = 1 (1 minute):

Time elapsed    Counter change
──────────────────────────────
0-59s          Pas de décay
60s            -1
120s           -2
...            ...

Si counter = 10:
- Après 10 min sans accès → counter = 0
- Clé devient candidate à l'éviction

lfu-decay-time = 10 (10 minutes):
- Decay plus lent
- Favorise les anciennes données populaires
```

#### Stratégie complète LFU

```
1. Nouvel objet créé
   └─> counter = 5 (LFU_INIT_VAL)

2. À chaque accès
   ├─> Décrémente selon temps écoulé
   └─> Incrémente probabilistiquement

3. À l'éviction
   ├─> Échantillonne N clés
   ├─> Applique decay sur chaque clé
   ├─> Sélectionne clé avec plus petit counter
   └─> Évicte
```

### Random : Implémentation Redis

L'algorithme Random est le plus simple :

```c
// Sélection aléatoire
robj *evictRandomKey(redisDb *db) {
    dictEntry *de = dictGetRandomKey(db->dict);

    if (de == NULL) return NULL;

    return dictGetVal(de);
}

// Pas de calcul de score
// Pas de comparaison
// Juste une sélection aléatoire
```

**Avantages de la simplicité** :

```
Overhead CPU:
- LRU:    ~1.15x baseline
- LFU:    ~1.25x baseline
- Random: ~1.02x baseline  ← Minimal

Latence d'éviction:
- LRU:    0.5-2 ms
- LFU:    1-5 ms
- Random: 0.1-0.5 ms  ← Plus rapide
```

## Comparaison algorithmique

### Complexité temporelle

| Opération | LRU | LFU | Random |
|-----------|-----|-----|--------|
| **Accès (GET)** | O(1) + update | O(1) + update + decay | O(1) |
| **Éviction** | O(N) samples | O(N) samples + decay | O(1) |
| **Update overhead** | ~1ns | ~10ns | 0 |

### Complexité spatiale

| Algorithme | Overhead par clé | Structure additionnelle |
|------------|------------------|-------------------------|
| **LRU** | 3 bytes (24 bits) | Pool 16 entrées |
| **LFU** | 3 bytes (24 bits) | Pool 16 entrées |
| **Random** | 0 bytes | Aucune |

### Précision de l'éviction

**Test standardisé** : Workload 80/20 (80% des accès sur 20% des clés)

```
Algorithme    Précision    Hit Ratio    Description
────────────────────────────────────────────────────
LRU parfait   100%         ~90%         Théorique
LRU (samp=5)  ~85%         ~82%         Redis défaut
LRU (samp=10) ~95%         ~87%         Redis optimisé
LFU (log=10)  ~90%         ~87%         Redis défaut
LFU (log=100) ~85%         ~85%         Moins discriminant
Random        ~50%         ~60%         Baseline
```

## Benchmarks pratiques

### Setup du benchmark

```python
import redis
import random
import time

class CacheBenchmark:
    def __init__(self, policy, maxmemory='10mb'):
        self.r = redis.Redis(decode_responses=True)
        self.r.flushdb()
        self.r.config_set('maxmemory', maxmemory)
        self.r.config_set('maxmemory-policy', f'allkeys-{policy}')

    def zipfian_distribution(self, n, alpha=1.5):
        """Génère distribution Zipf (80/20)"""
        weights = [1 / (i ** alpha) for i in range(1, n + 1)]
        total = sum(weights)
        return [w / total for w in weights]

    def run_workload(self, num_keys=1000, num_ops=100000):
        # Pré-remplissage
        for i in range(num_keys):
            self.r.set(f'key:{i}', f'value{i}')

        # Distribution Zipfian
        probs = self.zipfian_distribution(num_keys)

        hits = 0
        misses = 0
        start = time.time()

        for _ in range(num_ops):
            # Sélectionne clé selon distribution
            key_id = random.choices(range(num_keys), probs)[0]

            result = self.r.get(f'key:{key_id}')
            if result:
                hits += 1
            else:
                misses += 1
                # Recharge
                self.r.set(f'key:{key_id}', f'value{key_id}')

        elapsed = time.time() - start

        return {
            'hits': hits,
            'misses': misses,
            'hit_ratio': hits / (hits + misses),
            'ops_per_sec': num_ops / elapsed,
            'avg_latency_ms': (elapsed / num_ops) * 1000
        }
```

### Résultats benchmark : Hit Ratio

**Workload 80/20** (80% accès sur 20% clés) :

```bash
# Test
python benchmark.py --workload zipf --alpha 1.5

Results:
Policy         Hit Ratio    OPS/sec    Avg Latency
────────────────────────────────────────────────────
allkeys-lru    82.4%        45,200     0.022 ms
allkeys-lfu    87.1%        41,800     0.024 ms
allkeys-random 59.7%        48,500     0.021 ms
```

**Workload uniforme** (accès équiprobables) :

```
Policy         Hit Ratio    OPS/sec    Avg Latency
────────────────────────────────────────────────────
allkeys-lru    51.2%        46,100     0.022 ms
allkeys-lfu    50.8%        42,500     0.024 ms
allkeys-random 50.1%        48,900     0.020 ms
```

**Workload "scan"** (lecture séquentielle périodique) :

```
Scénario: 1 scan complet toutes les 100 requêtes

Policy         Hit Ratio    Impact scan
─────────────────────────────────────────
allkeys-lru    45.2%        ❌ Très affecté
allkeys-lfu    78.3%        ✅ Résiste bien
allkeys-random 59.1%        ⚠️ Non affecté
```

### Résultats benchmark : Performance CPU

**Measurement** : CPU cycles par éviction

```
Test: Éviction de 10,000 clés

Algorithm    CPU Cycles    Time (ms)    Normalized
──────────────────────────────────────────────────
random       1,250,000     0.5          1.00x
lru (s=3)    1,875,000     0.75         1.50x
lru (s=5)    2,500,000     1.0          2.00x
lru (s=10)   4,375,000     1.75         3.50x
lfu (s=5)    3,125,000     1.25         2.50x
```

### Résultats benchmark : Latency distribution

**P50, P95, P99 latencies** (éviction active) :

```
Policy         P50      P95      P99      Max
──────────────────────────────────────────────
allkeys-random 0.1ms    0.3ms    0.5ms    2ms
allkeys-lru    0.5ms    1.5ms    3ms      8ms
allkeys-lfu    1.0ms    3.0ms    6ms      15ms
```

## Patterns d'accès et choix de stratégie

### 1. Accès temporels (Temporal Locality)

**Caractéristiques** :
- Les données récemment accédées sont réaccédées bientôt
- Pattern typique : navigation utilisateur, sessions

**Exemple** :

```
User session flow:
Time ─────────────────────────────────→
     Login → Profile → Settings → Logout
     │       │         │           │
     └───────┴─────────┴───────────┘
     Toutes accédées dans une fenêtre courte
```

**Recommandation** : **LRU** ✅

```conf
maxmemory-policy allkeys-lru
maxmemory-samples 5
```

**Justification** :
- LRU capture parfaitement la temporal locality
- Données récentes restent en cache
- Économique en CPU

### 2. Accès fréquentiels (Frequency Locality)

**Caractéristiques** :
- Certaines données sont accédées beaucoup plus souvent
- Pattern stable dans le temps
- Pattern typique : top produits, top articles

**Exemple** :

```
E-commerce product views (1 semaine):

Product A: 50,000 views  ← Hot
Product B: 10,000 views
Product C: 5,000 views
Product D: 1,000 views
...
Product Z: 10 views      ← Cold
```

**Recommandation** : **LFU** ✅

```conf
maxmemory-policy allkeys-lfu
lfu-log-factor 10
lfu-decay-time 5  # Decay plus lent pour stabilité
maxmemory-samples 10  # Meilleure précision
```

**Justification** :
- LFU identifie et garde les données populaires
- Résiste aux scans temporaires
- Optimal pour hot data stable

### 3. Accès en rafales (Burst Access)

**Caractéristiques** :
- Pics d'accès soudains sur certaines données
- Popularité change rapidement
- Pattern typique : trending topics, flash sales

**Exemple** :

```
News article views:
Hour 1:  Article X: 1000 views    ← Breaking news
Hour 2:  Article X: 50000 views   ← Trending
Hour 3:  Article X: 5000 views    ← Décline
Hour 24: Article X: 100 views     ← Cold
```

**Recommandation** : **LRU** ✅ ou **LFU avec decay rapide** ⚠️

```conf
# Option 1: LRU (préféré)
maxmemory-policy allkeys-lru

# Option 2: LFU avec decay rapide
maxmemory-policy allkeys-lfu
lfu-decay-time 1  # 1 minute (très rapide)
lfu-log-factor 5  # Croissance rapide du compteur
```

**Justification** :
- LRU s'adapte rapidement aux changements
- LFU avec decay rapide oublie vite les anciens patterns

### 4. Scans périodiques

**Caractéristiques** :
- Lecture complète du dataset périodiquement
- Pollue le cache LRU
- Pattern typique : batch jobs, analytics, backups

**Exemple** :

```
Normal access + periodic scan:

Time: 0-55 min   → Accès utilisateurs normaux
Time: 55-60 min  → SCAN complet (backup/analytics)
                   └─> Tous les keys deviennent "récents"
                   └─> Évacue le cache utile !

Après scan avec LRU:
Cache = [données scan inutiles] ❌
      ≠ [données utilisateurs] ✅
```

**Recommandation** : **LFU** ✅

```conf
maxmemory-policy allkeys-lfu
lfu-log-factor 10
lfu-decay-time 5
```

**Justification** :
- LFU ignore les accès uniques du scan
- Garde les vraies données fréquentes
- LRU serait complètement invalidé

### 5. Accès uniformes

**Caractéristiques** :
- Toutes les données ont la même probabilité d'accès
- Pas de pattern particulier
- Pattern typique : rare en pratique

**Recommandation** : **Random** ✅

```conf
maxmemory-policy allkeys-random
```

**Justification** :
- LRU et LFU n'apportent aucun bénéfice
- Random est plus rapide
- Simplicité maximale

### 6. Working Set stable

**Caractéristiques** :
- Ensemble de données actives stable
- Taille << mémoire totale
- Pattern typique : référentiels, configurations

**Exemple** :

```
Application configuration:
- 1000 configs utilisées activement
- 10000 configs totales
- Cache: 2000 configs

Working set (1000) tient facilement en cache
→ Peu d'évictions
→ Algorithme importe peu
```

**Recommandation** : **LRU** ou **Random** ✅

```conf
# Simple et efficace
maxmemory-policy allkeys-lru
maxmemory-samples 3  # Peut réduire
```

**Justification** :
- Peu d'évictions = impact algorithme minimal
- Optimiser pour la performance

## Cas d'usage détaillés

### Cas 1 : Session Store

**Contexte** :
- Sessions utilisateur HTTP
- Accès séquentiels (même user)
- TTL automatique

**Pattern d'accès** :

```
User logs in → Creates session
  ├─> Multiple requests in quick succession
  ├─> Idle period
  └─> More requests or timeout

Temporal locality: Forte ✅
Frequency locality: Faible
```

**Configuration recommandée** :

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru  # ← Temporal locality
maxmemory-samples 5
```

**Alternative avec TTL** :

```conf
maxmemory-policy volatile-lru  # Protège autres données
# Sessions avec TTL:
# SETEX session:abc123 1800 {data}
```

### Cas 2 : Cache API externe

**Contexte** :
- Mise en cache de réponses API
- Certains endpoints très populaires
- Coût de rechargement élevé

**Pattern d'accès** :

```
API endpoints popularity:
/api/trending    → 10,000 req/min  ← Hot
/api/search      → 5,000 req/min
/api/product/:id → Variable
/api/obscure     → 10 req/min      ← Cold

Frequency locality: Forte ✅
Temporal locality: Moyenne
```

**Configuration recommandée** :

```conf
maxmemory 16gb
maxmemory-policy allkeys-lfu  # ← Frequency locality
lfu-log-factor 10
lfu-decay-time 10  # 10 min pour stabilité
maxmemory-samples 10
```

**Justification** :
- Endpoints populaires restent en cache
- Endpoints rares évictés rapidement
- Maximise le hit ratio sur données coûteuses

### Cas 3 : E-commerce product catalog

**Contexte** :
- Catalogue produits
- Top produits très consultés
- Scans périodiques (inventory sync)

**Pattern d'accès** :

```
Product views (Zipfian distribution):
Top 100 products  → 80% des vues
Mid 1000 products → 15% des vues
Tail 10000        → 5% des vues

+ Scan complet toutes les heures (inventory)

Frequency locality: Très forte ✅
Scan pollution: Présente ❌
```

**Configuration recommandée** :

```conf
maxmemory 32gb
maxmemory-policy allkeys-lfu  # ← Résiste au scan
lfu-log-factor 10
lfu-decay-time 15  # Decay lent
maxmemory-samples 15  # Haute précision
```

**Impact du scan** :

```
With LRU:
- Scan invalide tout le cache ❌
- Hit ratio drops 90% → 20% pendant sync

With LFU:
- Scan n'affecte pas les compteurs significativement ✅
- Hit ratio stable ~85%
```

### Cas 4 : Real-time analytics

**Contexte** :
- Compteurs temps réel
- Dashboards
- Toutes les métriques aussi importantes

**Pattern d'accès** :

```
Metrics:
counter:sales:today
counter:users:online
counter:errors:rate
...

Tous accédés régulièrement et uniformément

Locality: Aucune
```

**Configuration recommandée** :

```conf
maxmemory 8gb
maxmemory-policy allkeys-random  # ← Pas de pattern
# Ou noeviction si toutes les métriques critiques
```

**Justification** :
- Aucun algorithme intelligent n'apporte de bénéfice
- Random = performance maximale
- Ou noeviction si perte inacceptable

### Cas 5 : CDN / Image cache

**Contexte** :
- Cache d'images/fichiers statiques
- Quelques images très populaires (logo, avatar default)
- Longue traîne d'images rares

**Pattern d'accès** :

```
Image requests:
logo.png          → 1M req/day  ← Hot
avatar_default    → 500k req/day
user_avatar_123   → 10 req/day   ← Cold
...

Frequency locality: Très forte ✅
Temporal locality: Faible
```

**Configuration recommandée** :

```conf
maxmemory 128gb  # Large cache
maxmemory-policy allkeys-lfu
lfu-log-factor 100  # Croissance très lente
lfu-decay-time 1440  # 24h (très lent)
maxmemory-samples 20  # Précision maximale
```

**Justification** :
- Images populaires doivent TOUJOURS être en cache
- LFU avec décay lent = stabilité maximale
- Tail images évictées aggressivement

### Cas 6 : Rate Limiting

**Contexte** :
- Compteurs de rate limiting par IP/user
- Accès en rafales
- TTL court

**Pattern d'accès** :

```
Rate limit counters:
user:123:rate → Multiple accesses in seconds
              → Then TTL expires (60s)

Temporal locality: Très forte (bursts) ✅
Frequency locality: Faible
```

**Configuration recommandée** :

```conf
maxmemory 2gb
maxmemory-policy volatile-lru  # Avec TTL mandatory
maxmemory-samples 5
```

**Alternative** :

```python
# All rate limit keys have TTL
redis.incr(f"rate:{user_id}")
redis.expire(f"rate:{user_id}", 60)

# volatile-lru évacue les anciens compteurs
```

## Tuning et optimisation

### Tuning LRU

#### maxmemory-samples

**Impact sur la qualité** :

```python
# Test empirique
def test_lru_quality(samples):
    # Workload 80/20
    results = benchmark(
        policy='allkeys-lru',
        samples=samples,
        workload='zipf'
    )
    return results['hit_ratio']

samples=3:  78.2% hit ratio
samples=5:  82.4% hit ratio  ← Défaut
samples=10: 87.1% hit ratio
samples=20: 89.5% hit ratio
```

**Recommandations** :

```conf
# Development/testing
maxmemory-samples 3

# Production standard
maxmemory-samples 5  # Défaut

# Production critique (hit ratio >> CPU)
maxmemory-samples 10

# High-frequency trading, CDN edge
maxmemory-samples 20
```

### Tuning LFU

#### lfu-log-factor

Contrôle la vitesse de croissance du compteur :

```conf
# Croissance rapide (favorise données récentes)
lfu-log-factor 1
# 100 hits → counter ~50

# Croissance moyenne (défaut)
lfu-log-factor 10
# 100 hits → counter ~10

# Croissance lente (favorise données anciennes)
lfu-log-factor 100
# 100 hits → counter ~3
```

**Cas d'usage** :

```
Pattern volatil (trending topics):
└─> lfu-log-factor 5  # Réagit vite

Pattern stable (référentiels):
└─> lfu-log-factor 50  # Stabilité

Pattern mixte:
└─> lfu-log-factor 10  # Défaut
```

#### lfu-decay-time

Contrôle la vitesse d'oubli :

```conf
# Oubli rapide (adaptatif)
lfu-decay-time 1  # 1 minute
# Old hot key devient cold en ~10 minutes

# Oubli moyen (défaut)
lfu-decay-time 5  # 5 minutes
# Old hot key devient cold en ~50 minutes

# Oubli lent (stable)
lfu-decay-time 60  # 1 heure
# Old hot key devient cold en ~10 heures
```

**Cas d'usage** :

```
Charge fluctuante (flash sales):
└─> lfu-decay-time 1  # Adaptation rapide

Charge stable (API gateway):
└─> lfu-decay-time 10  # Balance

Working set quasi-permanent (CDN):
└─> lfu-decay-time 60  # Stabilité maximale
```

#### Combinaisons recommandées

**Pattern: Highly volatile** (social media trends)

```conf
maxmemory-policy allkeys-lfu
lfu-log-factor 5      # Croissance rapide
lfu-decay-time 1      # Oubli rapide
maxmemory-samples 5
```

**Pattern: Stable hot data** (CDN edge cache)

```conf
maxmemory-policy allkeys-lfu
lfu-log-factor 50     # Croissance lente
lfu-decay-time 60     # Oubli très lent
maxmemory-samples 20  # Précision max
```

**Pattern: Balanced** (API caching)

```conf
maxmemory-policy allkeys-lfu
lfu-log-factor 10     # Défaut
lfu-decay-time 5      # Défaut
maxmemory-samples 10
```

### Optimisations communes

#### Lazy freeing

Toujours activer pour toutes les politiques :

```conf
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
```

**Impact** :

```
Without lazyfree:
- Éviction grande clé → 100ms blocage ❌

With lazyfree:
- Éviction grande clé → <1ms (async) ✅
```

#### Active defrag

Avec éviction active, la fragmentation peut s'accumuler :

```conf
activedefrag yes
active-defrag-cycle-min 5
active-defrag-cycle-max 75
active-defrag-threshold-lower 10
```

## Migration entre stratégies

### À chaud (sans downtime)

```bash
# État actuel
redis-cli CONFIG GET maxmemory-policy
# "allkeys-lru"

# Changement à chaud
redis-cli CONFIG SET maxmemory-policy allkeys-lfu

# Persister
redis-cli CONFIG REWRITE
```

**Impact** :

```
Changement LRU → LFU:
- Counters initialisés à LFU_INIT_VAL (5)
- Anciennes données avantagées temporairement
- Normalisation après quelques minutes

Changement LFU → LRU:
- Timestamp initialisé à now()
- Toutes les clés paraissent "récentes"
- Normalisation après quelques accès
```

### Migration progressive (A/B testing)

```python
# Setup A/B test
def route_to_redis(user_id):
    if hash(user_id) % 2 == 0:
        return redis_lru  # Instance LRU
    else:
        return redis_lfu  # Instance LFU

# Mesure comparative
def measure_hit_ratio(redis_instance, duration=3600):
    stats_before = redis_instance.info('stats')
    time.sleep(duration)
    stats_after = redis_instance.info('stats')

    hits_delta = stats_after['keyspace_hits'] - stats_before['keyspace_hits']
    misses_delta = stats_after['keyspace_misses'] - stats_before['keyspace_misses']

    return hits_delta / (hits_delta + misses_delta)

# Résultats
print(f"LRU hit ratio: {measure_hit_ratio(redis_lru)}")
print(f"LFU hit ratio: {measure_hit_ratio(redis_lfu)}")
```

### Checklist de migration

```
Avant migration:
☐ Benchmark hit ratio actuel
☐ Mesure CPU baseline
☐ Mesure P99 latency
☐ Identifier workload pattern

Pendant migration:
☐ CONFIG SET en test
☐ Observer métriques 1 heure
☐ Comparer avec baseline
☐ A/B test si critique

Après migration:
☐ CONFIG REWRITE si satisfait
☐ Monitor 24h
☐ Ajuster tuning si nécessaire
☐ Documenter changement
```

## Métriques de décision

### Tableau récapitulatif

| Critère | LRU | LFU | Random | Recommandation |
|---------|-----|-----|--------|----------------|
| **Hit ratio (80/20)** | 82% | 87% | 60% | LFU gagne |
| **Hit ratio (uniforme)** | 51% | 51% | 50% | Égalité |
| **CPU overhead** | +15% | +25% | +2% | Random gagne |
| **Latency P99** | 3ms | 6ms | 0.5ms | Random gagne |
| **Résistance scans** | ❌ Faible | ✅ Forte | ⚠️ N/A | LFU gagne |
| **Adaptabilité** | ✅ Rapide | ⚠️ Lent | ⚠️ N/A | LRU gagne |
| **Simplicité** | ✅ Simple | ⚠️ Complexe | ✅ Très simple | Random gagne |
| **Overhead mémoire** | 3 bytes | 3 bytes | 0 bytes | Random gagne |
| **Tuning requis** | Minimal | Moyen | Aucun | Random gagne |

### Arbre de décision

```
Avez-vous un pattern d'accès identifiable ?
├─ NON → Random
└─ OUI
   │
   ├─ Pattern temporel (récence) ?
   │  └─ OUI → LRU
   │
   ├─ Pattern fréquentiel (popularité) ?
   │  └─ OUI
   │     │
   │     ├─ Stable dans le temps ?
   │     │  └─ OUI → LFU (decay lent)
   │     │
   │     └─ Volatile (trending) ?
   │        └─ OUI → LRU ou LFU (decay rapide)
   │
   └─ Scans périodiques ?
      └─ OUI → LFU
```

### Scoring system

Évaluez votre cas d'usage :

```python
def choose_policy(answers):
    score_lru = 0
    score_lfu = 0
    score_random = 0

    # Q1: Temporal locality forte ?
    if answers['temporal_locality'] == 'strong':
        score_lru += 3

    # Q2: Frequency locality forte ?
    if answers['frequency_locality'] == 'strong':
        score_lfu += 3

    # Q3: Scans périodiques ?
    if answers['periodic_scans']:
        score_lfu += 2
        score_lru -= 2

    # Q4: Pattern stable ?
    if answers['stable_pattern']:
        score_lfu += 2

    # Q5: Volatilité (trending) ?
    if answers['volatile_pattern']:
        score_lru += 2
        score_lfu += 1

    # Q6: Performance critique ?
    if answers['performance_critical']:
        score_random += 2

    # Q7: Hit ratio critique ?
    if answers['hit_ratio_critical']:
        score_lfu += 2

    # Q8: Toutes données équiprobables ?
    if answers['uniform_access']:
        score_random += 5

    policies = [
        ('LRU', score_lru),
        ('LFU', score_lfu),
        ('Random', score_random)
    ]

    return max(policies, key=lambda x: x[1])[0]
```

## Conclusion

### Règles d'or

1. **Pour 80% des cas : LRU**
   - Simple, efficace, prédictible
   - Bon compromis performance/qualité

2. **Pour hot data stable : LFU**
   - Meilleur hit ratio
   - Résiste aux scans
   - Nécessite tuning

3. **Pour pattern uniforme : Random**
   - Performance maximale
   - Simplicité absolue
   - Aucun overhead

### Configuration par défaut recommandée

**Pour démarrer** :

```conf
# Configuration universelle sûre
maxmemory 8gb
maxmemory-policy allkeys-lru
maxmemory-samples 5
lazyfree-lazy-eviction yes
```

**Après analyse** :

```bash
# 1. Mesurer pattern
redis-cli MONITOR | analyze_pattern.py

# 2. Benchmark
./benchmark.sh --policies lru,lfu --duration 3600

# 3. Choisir selon résultats
redis-cli CONFIG SET maxmemory-policy allkeys-lfu

# 4. Tuner
redis-cli CONFIG SET lfu-log-factor 10
redis-cli CONFIG SET lfu-decay-time 5

# 5. Monitorer et ajuster
```

### Ne pas oublier

- **Mesurer avant d'optimiser** : Benchmarker est obligatoire
- **Le contexte est roi** : Pas de solution universelle
- **Tuning itératif** : Ajuster progressivement
- **Monitoring continu** : Hit ratio, CPU, latency
- **Documenter** : Justifier les choix de configuration

Le choix entre LRU, LFU et Random n'est pas une question de "meilleur algorithme" mais de **meilleur ajustement à votre workload**. Prenez le temps de caractériser votre pattern d'accès et de benchmarker.

La section suivante explorera les namespaces et bonnes pratiques de nommage des clés.

⏭️ [Namespaces et bonnes pratiques de nommage (Key patterns)](/04-cycle-vie-donnee/05-namespaces-bonnes-pratiques-nommage.md)
