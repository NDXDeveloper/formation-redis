🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3 Politiques d'éviction : Que se passe-t-il quand la RAM est pleine ?

## Introduction

Redis est une base de données **en mémoire**. Cette caractéristique fondamentale implique une contrainte incontournable : la RAM est **finie**. Lorsque cette limite est atteinte, Redis doit prendre une décision critique : **que faire des nouvelles écritures ?**

Les politiques d'éviction (eviction policies) définissent le comportement de Redis face à la saturation mémoire. C'est un mécanisme sophistiqué qui peut faire la différence entre une application résiliente et un système qui s'effondre en production.

## Le problème de la saturation mémoire

### Scénario typique

```
Application → Redis (2GB maxmemory)
         ↓
   Écrit progressivement
         ↓
   1.0 GB utilisé ✅
   1.5 GB utilisé ✅
   1.9 GB utilisé ✅
   2.0 GB utilisé ⚠️
         ↓
   Nouvelle écriture ?
         ↓
    Que faire ?
```

### Options possibles

1. **Refuser l'écriture** : Retourner une erreur OOM
2. **Supprimer des données** : Libérer de l'espace automatiquement
3. **Bloquer** : Attendre qu'un processus libère de la mémoire (❌ jamais dans Redis)

## Configuration de la limite mémoire

### maxmemory : La limite absolue

```conf
# redis.conf

# Définir une limite de mémoire
maxmemory 2gb

# Ou en bytes
maxmemory 2147483648

# Ou avec suffixes
maxmemory 2000mb
maxmemory 2048000kb
```

**Sans maxmemory** :

```conf
# Pas de limite (comportement par défaut)
maxmemory 0

# ⚠️ Dangereux en production !
# Redis peut consommer toute la RAM disponible
# → OOM killer du système peut tuer Redis
```

**Calcul recommandé** :

```
Mémoire système: 8 GB
├─ Système + autres services: 2 GB
├─ Buffer/cache kernel: 1 GB
└─ Redis maxmemory: 5 GB (safe)

maxmemory = (RAM totale - 2GB) * 0.75
```

**Vérification runtime** :

```bash
# Voir la limite actuelle
redis-cli CONFIG GET maxmemory
# 1) "maxmemory"
# 2) "2147483648"

# Modifier à chaud (non persistant)
redis-cli CONFIG SET maxmemory 4gb

# Sauvegarder la config
redis-cli CONFIG REWRITE
```

### maxmemory-policy : Choisir le comportement

```conf
# redis.conf
maxmemory-policy noeviction  # Défaut (refuse les écritures)
```

## Les 8 politiques d'éviction

### Vue d'ensemble

| Politique | Cible | Algorithme | Description |
|-----------|-------|------------|-------------|
| `noeviction` | - | - | Refuse les écritures (erreur) |
| `allkeys-lru` | Toutes | LRU | Évicte les moins récemment utilisées |
| `allkeys-lfu` | Toutes | LFU | Évicte les moins fréquemment utilisées |
| `allkeys-random` | Toutes | Random | Évicte aléatoirement |
| `volatile-lru` | Avec TTL | LRU | Évicte parmi clés avec TTL (LRU) |
| `volatile-lfu` | Avec TTL | LFU | Évicte parmi clés avec TTL (LFU) |
| `volatile-random` | Avec TTL | Random | Évicte parmi clés avec TTL (aléatoire) |
| `volatile-ttl` | Avec TTL | TTL court | Évicte clés avec TTL le plus court |

### 1. noeviction (défaut)

**Comportement** : Redis **refuse** toute écriture qui nécessiterait plus de mémoire.

**Mécanisme** :

```
Client → SET newkey "value"
         ↓
Memory usage > maxmemory ?
         ↓
       OUI
         ↓
    Retourne erreur OOM
    "OOM command not allowed when used memory > 'maxmemory'."
         ↓
    Pas d'écriture
```

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy noeviction
```

**Test** :

```bash
# Remplir Redis jusqu'à la limite
redis-cli CONFIG SET maxmemory 10mb
for i in {1..10000}; do
    redis-cli SET key:$i "$(head -c 1000 /dev/urandom | base64)"
done

# Tentative d'écriture supplémentaire
redis-cli SET overflow "data"
# (error) OOM command not allowed when used memory > 'maxmemory'.

# Les lectures continuent de fonctionner
redis-cli GET key:1
# "..." - OK

# INCR, APPEND peuvent encore fonctionner si pas d'allocation nouvelle
redis-cli SET counter 0
redis-cli INCR counter
# (integer) 1 - OK (pas d'allocation nouvelle, reuse du buffer)
```

**Avantages** :
- ✅ Aucune perte de données
- ✅ Comportement prévisible
- ✅ Application contrôle les erreurs

**Inconvénients** :
- ❌ Application doit gérer l'erreur OOM
- ❌ Redis devient read-only
- ❌ Peut bloquer des fonctionnalités critiques

**Use cases** :
- Cache critique où aucune donnée ne doit être perdue
- Bases de données où l'intégrité prime
- Environnements où l'application gère elle-même le nettoyage

### 2. allkeys-lru (Least Recently Used)

**Comportement** : Évicte les clés les **moins récemment utilisées** parmi TOUTES les clés.

**Mécanisme** :

```
Client → SET newkey "value"
         ↓
Memory usage > maxmemory ?
         ↓
       OUI
         ↓
    Échantillonne N clés (maxmemory-samples)
         ↓
    Calcule score LRU pour chaque clé
         ↓
    Sélectionne clé avec plus vieux LRU
         ↓
    Supprime la clé (UNLINK)
         ↓
    Répète jusqu'à memory < maxmemory
         ↓
    Continue l'écriture
```

**Algorithme LRU simplifié** :

```c
// Structure RedisObject (simplifié)
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:24;  // LRU clock: secondes / résolution
    int refcount;
    void *ptr;
} robj;

// Calcul du score LRU
long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        // Wrap around
        return (LRU_CLOCK_MAX - o->lru + lruclock) * LRU_CLOCK_RESOLUTION;
    }
}

// LRU_CLOCK_RESOLUTION = 1000 ms (1 seconde)
// LRU_CLOCK_MAX = 2^24 - 1 = 16777215
// Range: ~194 jours
```

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 5  # Nombre de clés échantillonnées
```

**Exemple de fonctionnement** :

```bash
# Configuration
redis-cli CONFIG SET maxmemory 10mb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Remplir avec des clés
redis-cli SET key:1 "value"
redis-cli SET key:2 "value"
redis-cli SET key:3 "value"

# Accéder à certaines clés
redis-cli GET key:1  # Met à jour LRU de key:1
sleep 2
redis-cli GET key:2  # Met à jour LRU de key:2

# Observer les idle times
redis-cli OBJECT IDLETIME key:1  # ~2 secondes
redis-cli OBJECT IDLETIME key:2  # ~0 secondes
redis-cli OBJECT IDLETIME key:3  # ~5 secondes

# Quand maxmemory est atteinte, key:3 sera évictée en premier
```

**Avantages** :
- ✅ Garde les données fréquemment accédées
- ✅ Bon compromis performance/simplicité
- ✅ Fonctionne bien pour la plupart des caches

**Inconvénients** :
- ❌ Approximation (échantillonnage, pas vrai LRU)
- ❌ Les "scans" peuvent invalider le cache
- ❌ Peut évacuer des données sans TTL

**Use cases** :
- Cache général
- Sessions utilisateur
- Données fréquemment accédées

### 3. allkeys-lfu (Least Frequently Used)

**Comportement** : Évicte les clés les **moins fréquemment utilisées** parmi TOUTES les clés.

**Mécanisme LFU** :

Redis implémente un LFU probabiliste avec decay :

```c
// Structure du champ LRU en mode LFU (24 bits)
┌────────────────┬────────────┐
│ 16 bits        │ 8 bits     │
│ Compteur accès │ Decay time │
└────────────────┴────────────┘

// Compteur : 0-255 (logarithmique)
// Decay : Timestamp du dernier accès
```

**Algorithme de comptage** :

```c
// Incrémentation du compteur (probabiliste)
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;

    double r = (double)rand() / RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;

    double p = 1.0 / (baseval * server.lfu_log_factor + 1);

    if (r < p) counter++;
    return counter;
}

// Avec lfu_log_factor=10:
// 1 hit  → counter=1 (probabilité 100%)
// 10 hits → counter=5 (probabilité ~50% au 10e hit)
// 100 hits → counter=10
// 1000 hits → counter=15
```

**Decay du compteur** :

```c
// Réduction du compteur dans le temps
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;  // Last decrement time
    unsigned long counter = o->lru & 255;

    if (LFUTimeElapsed(ldt) >= server.lfu_decay_time) {
        if (counter > 0) {
            counter--;
        }
    }

    return counter;
}
```

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy allkeys-lfu

# Facteur logarithmique (0-255)
lfu-log-factor 10   # Défaut
# Plus élevé = compteur augmente plus lentement
# Plus bas = compteur augmente plus rapidement

# Temps de decay en minutes
lfu-decay-time 1    # Défaut: 1 minute
# Temps avant que le compteur décroisse
```

**Impact de lfu-log-factor** :

```
lfu-log-factor=1:
100 hits → counter ≈ 50
1000 hits → counter ≈ 100

lfu-log-factor=10 (défaut):
100 hits → counter ≈ 10
1000 hits → counter ≈ 15

lfu-log-factor=100:
100 hits → counter ≈ 3
1000 hits → counter ≈ 5
```

**Exemple** :

```bash
redis-cli CONFIG SET maxmemory-policy allkeys-lfu
redis-cli CONFIG SET lfu-log-factor 10
redis-cli CONFIG SET lfu-decay-time 1

# Créer des clés avec différentes fréquences
for i in {1..100}; do redis-cli GET key:hot; done      # 100 accès
for i in {1..10}; do redis-cli GET key:warm; done      # 10 accès
redis-cli GET key:cold                                 # 1 accès

# Observer les fréquences
redis-cli OBJECT FREQ key:hot   # ~15
redis-cli OBJECT FREQ key:warm  # ~5
redis-cli OBJECT FREQ key:cold  # ~1

# key:cold sera évictée en premier
```

**Avantages** :
- ✅ Meilleur que LRU pour patterns d'accès répétitifs
- ✅ Garde les données vraiment populaires
- ✅ Résiste aux scans qui invalident le LRU

**Inconvénients** :
- ❌ Plus complexe à comprendre
- ❌ Nécessite tuning (log_factor, decay_time)
- ❌ Peut favoriser les vieilles données populaires

**Use cases** :
- Cache avec accès très répétitifs
- Données avec popularité stable
- Protection contre cache pollution

### 4. allkeys-random

**Comportement** : Évicte des clés **aléatoirement** parmi TOUTES les clés.

**Mécanisme** :

```
Client → SET newkey "value"
         ↓
Memory usage > maxmemory ?
         ↓
       OUI
         ↓
    Sélectionne une clé aléatoire
         ↓
    Supprime la clé
         ↓
    Répète jusqu'à memory < maxmemory
         ↓
    Continue l'écriture
```

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy allkeys-random
```

**Avantages** :
- ✅ Très rapide (pas de calcul de score)
- ✅ Prévisible en performance
- ✅ Simple à comprendre

**Inconvénients** :
- ❌ Aucune intelligence
- ❌ Peut évacuer des données importantes
- ❌ Imprévisible en résultat

**Use cases** :
- Tests/développement
- Cas où toutes les données ont la même importance
- Performance absolue requise

### 5. volatile-lru

**Comportement** : Évicte parmi les clés **avec TTL**, selon l'algorithme LRU.

**Mécanisme** :

```
Client → SET newkey "value"
         ↓
Memory usage > maxmemory ?
         ↓
       OUI
         ↓
    Échantillonne N clés AVEC TTL
         ↓
    Si aucune clé avec TTL trouvée
         ↓
    Retourne erreur OOM
         ↓
    Sinon, calcule score LRU
         ↓
    Évicte clé avec plus vieux LRU
         ↓
    Continue l'écriture
```

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy volatile-lru
maxmemory-samples 5
```

**Exemple** :

```bash
redis-cli CONFIG SET maxmemory 10mb
redis-cli CONFIG SET maxmemory-policy volatile-lru

# Clés sans TTL (ne seront PAS évictées)
redis-cli SET important:1 "critical data"
redis-cli SET important:2 "critical data"

# Clés avec TTL (candidates à l'éviction)
redis-cli SET cache:1 "data" EX 3600
redis-cli SET cache:2 "data" EX 3600
redis-cli SET cache:3 "data" EX 3600

# Accès à cache:1 et cache:2
redis-cli GET cache:1
redis-cli GET cache:2

# Quand maxmemory atteinte, cache:3 sera évictée (LRU le plus vieux)
# important:1 et important:2 ne seront JAMAIS évictées
```

**Avantages** :
- ✅ Protège les données sans TTL
- ✅ Bon contrôle sur ce qui est évictable
- ✅ Combine éviction et expiration

**Inconvénients** :
- ❌ Erreur OOM si aucune clé avec TTL
- ❌ Nécessite discipline de mettre TTL sur cache
- ❌ Performance dépend du ratio clés avec/sans TTL

**Use cases** :
- Mix de données permanentes et cache
- Applications critiques avec cache secondaire
- Contrôle fin de l'éviction

### 6. volatile-lfu

**Comportement** : Évicte parmi les clés **avec TTL**, selon l'algorithme LFU.

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy volatile-lfu
lfu-log-factor 10
lfu-decay-time 1
```

**Avantages/Inconvénients** : Identiques à volatile-lru, mais avec LFU.

### 7. volatile-random

**Comportement** : Évicte **aléatoirement** parmi les clés avec TTL.

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy volatile-random
```

**Use cases** :
- Performance maximale avec protection des données sans TTL
- Éviction équitable parmi le cache

### 8. volatile-ttl

**Comportement** : Évicte les clés avec le **TTL le plus court** parmi les clés avec TTL.

**Mécanisme** :

```
Client → SET newkey "value"
         ↓
Memory usage > maxmemory ?
         ↓
       OUI
         ↓
    Échantillonne N clés AVEC TTL
         ↓
    Pour chaque clé, lit TTL dans expires dict
         ↓
    Sélectionne clé avec TTL le plus court
         ↓
    Supprime la clé
         ↓
    Continue l'écriture
```

**Configuration** :

```conf
maxmemory 2gb
maxmemory-policy volatile-ttl
```

**Exemple** :

```bash
redis-cli CONFIG SET maxmemory-policy volatile-ttl

# Clés avec différents TTL
redis-cli SET temp:1 "data" EX 60      # 1 minute
redis-cli SET temp:2 "data" EX 300     # 5 minutes
redis-cli SET temp:3 "data" EX 3600    # 1 heure

# Quand maxmemory atteinte, temp:1 sera évictée en premier
```

**Avantages** :
- ✅ Logique intuitive (évacue ce qui expire bientôt)
- ✅ Complémentaire à l'expiration automatique
- ✅ Prioritise les données à durée de vie longue

**Inconvénients** :
- ❌ Peut évacuer des données fréquemment accédées
- ❌ Ignore la popularité
- ❌ Erreur OOM si aucune clé avec TTL

**Use cases** :
- Cache avec différentes priorités via TTL
- Données temporaires avec urgence variable

## L'échantillonnage : maxmemory-samples

### Le problème de performance

Calculer le score LRU/LFU de **toutes** les clés serait trop coûteux :

```
1 million de clés:
├─ LRU parfait : Parcourir 1M clés → ~100ms
└─ Échantillonnage : Parcourir 5 clés → <0.1ms
```

### L'échantillonnage probabiliste

Redis utilise un échantillonnage aléatoire :

```python
# Pseudo-code de l'éviction
def evict_key():
    best_key = None
    best_score = 0

    # Échantillonne N clés aléatoires
    for _ in range(maxmemory_samples):
        key = random_key()
        score = calculate_score(key)  # LRU, LFU, TTL, etc.

        if score > best_score:
            best_score = score
            best_key = key

    delete(best_key)
```

### Configuration

```conf
# redis.conf
maxmemory-samples 5  # Défaut

# Plus de samples = meilleure précision, plus de CPU
maxmemory-samples 3   # Rapide, moins précis
maxmemory-samples 10  # Lent, plus précis
maxmemory-samples 20  # Très lent, très précis
```

### Impact sur la précision

**Test de précision LRU** (source: Redis doc) :

```
Échantillonnage vs LRU parfait:

maxmemory-samples 3:  ~70% de précision
maxmemory-samples 5:  ~85% de précision (défaut)
maxmemory-samples 10: ~95% de précision
maxmemory-samples 20: ~99% de précision

Coût CPU:
samples=3  → 1x
samples=5  → 1.5x
samples=10 → 3x
samples=20 → 6x
```

**Visualisation** :

```
LRU parfait (théorique):
Clés évictées: [oldest] ████████████████████ [newest]

samples=3:
Clés évictées: [oldest] ██░░██░░████░░██░░░█ [newest]
                           ↑ Quelques "jeunes" clés évictées par erreur

samples=10:
Clés évictées: [oldest] ███████████████░░░░░ [newest]
                                     ↑ Très proche du LRU parfait
```

### Recommandations

```conf
# Environnement de développement
maxmemory-samples 3  # Vitesse > précision

# Production standard
maxmemory-samples 5  # Bon équilibre (défaut)

# Production critique
maxmemory-samples 10  # Précision importante

# Haute charge, CPU limité
maxmemory-samples 3

# Faible charge, précision maximale
maxmemory-samples 20
```

## Le processus d'éviction en détail

### Flow d'exécution complet

```
Client → SET key "value"
    ↓
1. Parse command
    ↓
2. Vérifie type de commande (write ?)
    ↓
3. Vérifie mémoire AVANT exécution
    ↓
used_memory + estimated_size > maxmemory ?
    ↓
  OUI ───→ 4. Applique éviction
    │           ↓
    │       Policy = noeviction ?
    │           ↓
    │         OUI ───→ Retourne erreur OOM
    │           │
    │         NON
    │           ↓
    │       5. Pool d'éviction
    │           ↓
    │       Échantillonne maxmemory_samples clés
    │           ↓
    │       Filtre selon policy (all vs volatile)
    │           ↓
    │       Calcule score (LRU/LFU/TTL/Random)
    │           ↓
    │       Trie par score
    │           ↓
    │       6. Supprime clé(s)
    │           ↓
    │       UNLINK (async) ou DEL (sync)
    │           ↓
    │       Répète si encore > maxmemory
    │           ↓
    │  NON  7. Exécute commande
    └────────→
```

### Estimation de la taille

Redis estime la taille de la nouvelle allocation avant d'exécuter la commande :

```c
// Estimation simplifiée
size_t estimated_size = 0;

switch (command) {
    case SET:
        estimated_size = sizeof(robj) + strlen(key) + strlen(value);
        break;
    case LPUSH:
        estimated_size = sizeof(listNode) + strlen(value);
        break;
    case SADD:
        estimated_size = sizeof(dictEntry) + strlen(member);
        break;
    // ...
}

if (used_memory + estimated_size > maxmemory) {
    evict();
}
```

### Pool d'éviction

Redis 3.0+ utilise un **pool** pour améliorer la qualité de l'éviction :

```c
#define EVPOOL_SIZE 16

struct evictionPoolEntry {
    unsigned long long idle;  // Score (idle time, freq, ttl)
    sds key;                  // Clé candidate
    sds cached;               // SDS cached pour performance
    int dbid;                 // Database ID
};

static struct evictionPoolEntry EvictionPoolLRU[EVPOOL_SIZE];
```

**Mécanisme** :

```
1. Pool initialement vide (16 slots)
2. Échantillonne maxmemory_samples clés
3. Calcule score pour chaque clé
4. Insère dans le pool si score > min(pool)
5. Répète jusqu'à pool plein
6. Évicte clé avec plus haut score du pool
7. Continue si nécessaire
```

**Avantage** : Améliore la qualité sans augmenter maxmemory-samples.

## Comparaison des politiques

### Tableau décisionnel

| Critère | noeviction | allkeys-lru | allkeys-lfu | volatile-lru | volatile-ttl |
|---------|-----------|-------------|-------------|--------------|--------------|
| **Perte de données** | ❌ Jamais | ✅ Cache | ✅ Cache | ⚠️ Cache | ⚠️ Cache |
| **Erreur OOM** | ✅ Oui | ❌ Jamais | ❌ Jamais | ⚠️ Si pas TTL | ⚠️ Si pas TTL |
| **CPU** | Minimal | Faible | Moyen | Faible | Faible |
| **Précision** | N/A | Bonne | Meilleure | Bonne | Moyenne |
| **Complexité** | Simple | Simple | Moyenne | Moyenne | Simple |

### Matrice de décision

```
Votre cas d'usage:

1. "J'ai uniquement du cache" ?
   └─> allkeys-lru (ou allkeys-lfu si accès répétitifs)

2. "J'ai données permanentes + cache" ?
   └─> volatile-lru (avec TTL sur cache)

3. "Toutes mes données ont la même importance" ?
   └─> allkeys-random

4. "Mes données sont critiques" ?
   └─> noeviction (+ monitoring + alerting)

5. "Mon cache a différentes priorités" ?
   └─> volatile-ttl (TTL court = basse priorité)

6. "J'ai beaucoup de scans/parcours" ?
   └─> allkeys-lfu (résiste mieux)

7. "Je veux la performance maximale" ?
   └─> allkeys-random
```

### Benchmarks comparatifs

**Hit ratio avec workload réel** :

```
Workload: 80/20 (80% accès sur 20% des données)

allkeys-lru:    ~82% hit ratio
allkeys-lfu:    ~87% hit ratio  ← Meilleur
allkeys-random: ~60% hit ratio
volatile-lru:   ~80% hit ratio (avec 50% des clés avec TTL)
```

**CPU overhead** :

```
Baseline (noeviction): 1.0x

allkeys-random:  1.02x
allkeys-lru:     1.15x
allkeys-lfu:     1.25x
volatile-lru:    1.18x
volatile-ttl:    1.20x
```

## Configuration par cas d'usage

### 1. Cache pur (type Memcached)

```conf
# Configuration recommandée
maxmemory 8gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# Ou si accès très répétitifs
maxmemory-policy allkeys-lfu
lfu-log-factor 10
lfu-decay-time 1
```

**Justification** :
- Toutes les données sont du cache → `allkeys-*`
- LRU fonctionne bien pour la plupart des patterns
- LFU si vous avez des "hits" très concentrés

### 2. Session store

```conf
maxmemory 4gb
maxmemory-policy volatile-lru
maxmemory-samples 5

# Sessions avec TTL
# SET session:abc123 {data} EX 1800  # 30 minutes
```

**Justification** :
- Sessions ont toutes un TTL → `volatile-*`
- LRU évacue les sessions inactives
- Protection contre OOM si TTL oubliés

### 3. Mix données permanentes + cache

```conf
maxmemory 16gb
maxmemory-policy volatile-lru
maxmemory-samples 10

# Données permanentes: PAS de TTL
# SET user:123:profile {data}

# Cache: AVEC TTL
# SET cache:api:result {data} EX 3600
```

**Justification** :
- Sépare clairement permanent vs cache via TTL
- Aucun risque d'éviction de données importantes
- Flexible et sécurisé

### 4. Application critique (pas de perte)

```conf
maxmemory 8gb
maxmemory-policy noeviction

# Monitoring et alerting obligatoires
# Alert si used_memory > 80% maxmemory
```

**Justification** :
- Aucune perte de données acceptable
- Application gère l'erreur OOM
- Nettoyage manuel ou automatique via application

### 5. Cache de résultats coûteux

```conf
maxmemory 32gb
maxmemory-policy allkeys-lfu
lfu-log-factor 10
lfu-decay-time 5

maxmemory-samples 10
```

**Justification** :
- Calculs coûteux → maximiser réutilisation
- LFU garde les résultats vraiment populaires
- decay-time plus long pour favoriser stabilité

### 6. Multi-tenant avec priorités

```conf
maxmemory 64gb
maxmemory-policy volatile-ttl
maxmemory-samples 10

# Tenant premium: TTL long
# SET tenant:premium:123 {data} EX 86400  # 24h

# Tenant standard: TTL court
# SET tenant:std:456 {data} EX 3600  # 1h
```

**Justification** :
- TTL comme mécanisme de priorité
- Évacue d'abord les données moins prioritaires
- QoS naturel via TTL

## Monitoring de l'éviction

### Métriques clés

```bash
# Nombre total d'évictions
redis-cli INFO stats | grep evicted_keys
# evicted_keys:12543

# Taux d'éviction (calculé)
# evictions_per_sec = delta(evicted_keys) / delta(time)

# Mémoire utilisée
redis-cli INFO memory | grep used_memory:
# used_memory:2147483648

# Ratio mémoire utilisée
redis-cli INFO memory | grep used_memory_peak:
# used_memory_peak:2200000000

# Hit ratio
redis-cli INFO stats | grep keyspace
# keyspace_hits:1000000
# keyspace_misses:100000
# Hit ratio = hits / (hits + misses) = 90.9%
```

### Commandes de diagnostic

```bash
# Vérifier la politique actuelle
redis-cli CONFIG GET maxmemory-policy
# 1) "maxmemory-policy"
# 2) "allkeys-lru"

# Observer les clés candidates à l'éviction
redis-cli --bigkeys
# Identifie les plus grosses clés

# Analyser la distribution des idle times
redis-cli --scan --pattern 'cache:*' | head -100 | while read key; do
    idle=$(redis-cli OBJECT IDLETIME "$key")
    echo "$key: $idle seconds"
done | sort -t: -k2 -n

# Observer en temps réel
redis-cli --stat
# Shows ops/sec, hit%, eviction%, etc.
```

### Script de monitoring avancé

```python
import redis
import time

def monitor_eviction(r, interval=60):
    stats_before = r.info('stats')
    mem_before = r.info('memory')

    time.sleep(interval)

    stats_after = r.info('stats')
    mem_after = r.info('memory')

    # Calculs
    evictions = stats_after['evicted_keys'] - stats_before['evicted_keys']
    eviction_rate = evictions / interval

    used_mem = mem_after['used_memory']
    max_mem = r.config_get('maxmemory')['maxmemory']
    mem_ratio = (used_mem / int(max_mem)) * 100 if max_mem != '0' else 0

    hits = stats_after['keyspace_hits'] - stats_before['keyspace_hits']
    misses = stats_after['keyspace_misses'] - stats_before['keyspace_misses']
    hit_ratio = (hits / (hits + misses) * 100) if (hits + misses) > 0 else 0

    print(f"""
Évictions: {evictions} ({eviction_rate:.2f}/sec)
Mémoire: {used_mem / 1024 / 1024:.2f} MB / {int(max_mem) / 1024 / 1024:.2f} MB ({mem_ratio:.1f}%)
Hit ratio: {hit_ratio:.1f}%
    """)

    # Alerting
    if mem_ratio > 90:
        alert("CRITICAL: Memory usage > 90%")
    if eviction_rate > 100:
        alert("WARNING: High eviction rate")
    if hit_ratio < 70:
        alert("WARNING: Low hit ratio")
```

### Alertes Prometheus

```yaml
# prometheus/alerts.yml
groups:
  - name: redis_eviction
    rules:
      - alert: RedisHighMemoryUsage
        expr: redis_memory_used_bytes / redis_config_maxmemory * 100 > 90
        for: 5m
        annotations:
          summary: "Redis memory usage > 90%"

      - alert: RedisHighEvictionRate
        expr: rate(redis_evicted_keys_total[5m]) > 100
        for: 5m
        annotations:
          summary: "High eviction rate (>100/sec)"

      - alert: RedisLowHitRatio
        expr: redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.7
        for: 10m
        annotations:
          summary: "Hit ratio < 70%"
```

## Impact sur la performance

### Latence d'éviction

**Mesure** :

```bash
redis-cli CONFIG SET latency-monitor-threshold 100

# Remplir jusqu'à maxmemory
# ... attendre les évictions

redis-cli LATENCY LATEST
# 1) 1) "eviction-cycle"
#    2) (integer) 1733847600
#    3) (integer) 15
#    4) (integer) 50
# 15ms de latency max, 50ms au 99e percentile
```

**Impact selon la politique** :

```
allkeys-random:  0.1-0.5 ms   (très rapide)
allkeys-lru:     0.5-2 ms     (rapide)
allkeys-lfu:     1-5 ms       (moyen)
volatile-lru:    0.5-3 ms     (rapide, dépend du ratio)
volatile-ttl:    0.5-3 ms     (rapide)
```

### Throughput

**Impact sur le débit** :

```
Baseline (pas d'éviction): 100k ops/sec

Avec éviction active:
- allkeys-random:  95k ops/sec (-5%)
- allkeys-lru:     90k ops/sec (-10%)
- allkeys-lfu:     85k ops/sec (-15%)
- volatile-lru:    92k ops/sec (-8%)
```

## Pièges et considérations

### 1. Oublier maxmemory en production

```conf
# ❌ Dangereux
maxmemory 0  # Pas de limite

# ✅ Toujours définir
maxmemory 8gb
maxmemory-policy allkeys-lru
```

### 2. noeviction sans monitoring

```conf
# ❌ Bombe à retardement
maxmemory 2gb
maxmemory-policy noeviction
# ... aucun monitoring

# ✅ Avec monitoring
maxmemory 2gb
maxmemory-policy noeviction
# + Alerting si used_memory > 80%
# + Nettoyage automatique ou manuel
```

### 3. volatile-* sans discipline TTL

```python
# ❌ Erreur OOM en production
redis.set('cache:data', value)  # Pas de TTL !
redis.set('important:data', value)  # Pas de TTL !

# Configuration: volatile-lru
# → Erreur OOM car aucune clé évictable

# ✅ Discipline stricte
redis.setex('cache:data', 3600, value)  # Cache avec TTL
redis.set('important:data', value)      # Permanent sans TTL
```

### 4. samples trop faible

```conf
# ❌ Mauvaise précision
maxmemory-samples 1  # ~50% précision

# ✅ Au moins 5
maxmemory-samples 5  # ~85% précision
```

### 5. Confusion LRU vs LFU

```
Cas d'usage          Recommandation
────────────────────────────────────
Accès uniformes      → LRU
Accès répétitifs     → LFU
Scans fréquents      → LFU
Cache de requêtes    → LRU
Hot data stable      → LFU
```

### 6. Éviction et réplication

**Important** : L'éviction se produit sur le master ET les replicas :

```
Master:
├─ Détecte maxmemory atteinte
├─ Applique éviction
├─ Supprime clé (DEL)
└─ Réplique DEL vers replicas

Replica:
├─ Reçoit DEL du master
└─ Supprime clé

Important: Replica peut AUSSI atteindre maxmemory localement
→ Configure maxmemory sur les replicas aussi !
```

### 7. Grande clés et éviction

```bash
# Problème: Éviction d'une grande clé peut bloquer
SADD bigset $(seq 1 10000000)  # 10M membres

# Si bigset est évictée:
# DEL bigset → ~100ms de blocage !

# Solution: Activer lazyfree
CONFIG SET lazyfree-lazy-eviction yes
# Éviction devient asynchrone (UNLINK)
```

## Configuration production recommandée

```conf
# redis.conf

################################ MEMORY ################################

# Limite mémoire (ajuster selon votre RAM)
maxmemory 8gb

# Politique d'éviction (choisir selon cas d'usage)
maxmemory-policy allkeys-lru

# Précision de l'éviction
maxmemory-samples 10

# Éviction asynchrone (recommandé)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes

# Si allkeys-lfu ou volatile-lfu
lfu-log-factor 10
lfu-decay-time 1

# Seuil latence monitoring
latency-monitor-threshold 100

################################ MONITORING ################################

# Stats détaillées
info-print-stats yes

# Slowlog
slowlog-log-slower-than 10000
slowlog-max-len 128
```

## Conclusion

Les politiques d'éviction sont un mécanisme crucial pour gérer Redis en production. Points clés :

- **noeviction** : Sûr mais nécessite gestion applicative
- **allkeys-lru** : Meilleur choix par défaut pour du cache pur
- **allkeys-lfu** : Meilleur pour patterns d'accès répétitifs
- **volatile-*** : Protection des données permanentes
- **maxmemory-samples** : Balance entre précision et CPU (5-10 recommandé)
- **lazyfree-lazy-eviction** : Toujours activer en production

Le choix de la politique dépend de votre cas d'usage, mais la règle d'or est : **toujours définir maxmemory et une politique adaptée**.

La section suivante approfondira la comparaison LRU vs LFU vs Random avec des benchmarks détaillés.

⏭️ [LRU vs LFU vs Random : Choisir la bonne stratégie](/04-cycle-vie-donnee/04-lru-vs-lfu-vs-random.md)
