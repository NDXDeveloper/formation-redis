🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 4 : Le cycle de vie de la donnée

## Introduction

Dans Redis, contrairement aux bases de données traditionnelles où les données persistent indéfiniment jusqu'à suppression explicite, **chaque donnée a un cycle de vie** qui peut être contrôlé, automatisé et optimisé. Ce module explore les mécanismes fondamentaux qui régissent l'existence, la persistance et la disparition des données dans Redis.

## Pourquoi le cycle de vie est-il critique ?

Redis est avant tout une base de données **en mémoire**. Cette caractéristique fondamentale implique deux contraintes majeures :

1. **La RAM est limitée** : Contrairement au stockage disque, vous ne pouvez pas stocker indéfiniment des données
2. **La RAM est coûteuse** : Chaque gigaoctet non utilisé efficacement représente un coût financier direct

La gestion du cycle de vie des données n'est pas un luxe dans Redis, **c'est une nécessité architecturale**.

## Les trois phases du cycle de vie

### 1. Création et insertion (Birth)

La donnée entre dans Redis via une commande d'écriture (SET, HSET, LPUSH, etc.). À ce moment :
- La clé est créée dans le keyspace
- La mémoire est allouée
- Un espace dans la table de hachage interne est réservé
- Optionnellement, un TTL peut être défini

### 2. Existence et accès (Life)

Durant cette phase, la donnée :
- Occupe de la mémoire RAM
- Peut être lue, modifiée, interrogée
- Peut avoir son TTL modifié ou supprimé
- Est candidate aux politiques d'éviction si la RAM devient saturée

### 3. Disparition (Death)

La donnée peut quitter Redis de plusieurs façons :
- **Expiration passive** : Vérification lors de l'accès (lazy)
- **Expiration active** : Vérification périodique par Redis (proactive)
- **Suppression explicite** : Commande DEL/UNLINK
- **Éviction** : Suppression forcée par manque de mémoire

## Architecture interne du cycle de vie

### Le dictionnaire principal (Main Dict)

Redis stocke toutes les clés dans une structure de données appelée **dictionnaire** (hash table). Chaque base de données Redis (DB 0-15 par défaut) possède son propre dictionnaire.

```c
// Structure simplifiée (code C interne de Redis)
typedef struct redisDb {
    dict *dict;                 /* Le dictionnaire principal */
    dict *expires;              /* Dictionnaire des expirations */
    dict *blocking_keys;        /* Clés bloquantes */
    dict *watched_keys;         /* Clés surveillées (WATCH) */
    int id;                     /* Database ID */
} redisDb;
```

### Le dictionnaire des expirations (Expires Dict)

Parallèlement au dictionnaire principal, Redis maintient un **second dictionnaire** qui ne contient que les clés ayant un TTL défini. Cette séparation est cruciale pour la performance.

**Pourquoi deux dictionnaires séparés ?**

```
Dictionnaire principal (dict):
┌──────────┬──────────────┐
│ "user:1" │ → Hash obj   │
│ "user:2" │ → Hash obj   │
│ "cache:x"│ → String obj │
│ "queue:a"│ → List obj   │
└──────────┴──────────────┘

Dictionnaire des expirations (expires):
┌─────────┬───────────────────────────┐
│ "user:1"│ → 1733756400 (timestamp)  │
│ "cache:x"│ → 1733753000 (timestamp) │
└─────────┴───────────────────────────┘
```

Cette architecture permet :
- **Performance optimale** : Les clés sans TTL n'occupent pas d'espace supplémentaire
- **Scan rapide des expirations** : Seules les clés avec TTL sont inspectées
- **Complexité O(1)** : Vérifier si une clé a un TTL est instantané

## Mécanismes d'expiration

Redis implémente un système hybride sophistiqué combinant deux approches complémentaires.

### 1. Expiration passive (Lazy)

**Principe** : La clé est vérifiée uniquement lorsqu'elle est accédée.

```
Client → GET user:123
         ↓
Redis vérifie le TTL dans expires dict
         ↓
   TTL expiré ?
    ↙        ↘
  OUI        NON
   ↓          ↓
Supprime   Retourne
la clé     la valeur
   ↓
Retourne
(nil)
```

**Avantages** :
- Pas de surcharge CPU pour les clés jamais accédées
- Réponse immédiate

**Inconvénients** :
- Les clés expirées mais jamais accédées restent en mémoire
- Fuite de mémoire potentielle

### 2. Expiration active (Proactive)

Pour éviter l'accumulation de clés expirées, Redis exécute un **algorithme probabiliste** en arrière-plan.

**Configuration** :

```conf
# redis.conf
hz 10  # Fréquence d'exécution du serverCron (10 fois/seconde par défaut)
```

**Algorithme (simplifié)** :

```python
# Pseudo-code de l'expiration active
def activeExpireCycle():
    EFFORT = 10  # Pourcentage d'effort CPU max

    for db in databases:
        while True:
            # 1. Échantillonnage aléatoire
            sample = random_sample(db.expires, 20)

            # 2. Vérification et suppression
            expired_count = 0
            for key in sample:
                if is_expired(key):
                    delete(key)
                    expired_count += 1

            # 3. Condition d'arrêt
            if expired_count < 5:  # Moins de 25% expirés
                break

            # 4. Protection CPU
            if cpu_time_used() > EFFORT:
                break
```

**Caractéristiques** :
- Exécution : 10 fois par seconde (configurable via `hz`)
- Échantillonnage : 20 clés aléatoires par itération
- Condition de continuation : Si >25% des clés sont expirées, continue
- Protection CPU : S'arrête si trop de temps CPU consommé

**Impact de la configuration `hz`** :

```conf
hz 10   # Default - Balance entre CPU et réactivité
        # 10 cycles/sec = vérification toutes les 100ms

hz 100  # Haute fréquence - Plus réactif mais plus de CPU
        # 100 cycles/sec = vérification toutes les 10ms
        # Utile pour des TTL très courts (< 1 seconde)

hz 1    # Économie de CPU - Moins réactif
        # 1 cycle/sec = vérification chaque seconde
        # Acceptable si TTL > 10 secondes
```

## Anatomie d'une clé Redis

Chaque clé Redis en mémoire contient plus que sa valeur. Voici la structure complète :

```
┌─────────────────────────────────────────┐
│           RedisObject (16 bytes)        │
├─────────────────────────────────────────┤
│ type: 4 bits    (STRING, LIST, etc.)    │
│ encoding: 4 bits (RAW, INT, ZIPLIST...) │
│ lru: 24 bits    (LRU clock ou LFU data) │
│ refcount: 32 bits (compteur références) │
│ ptr: 64 bits    (pointeur vers data)    │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│           Données réelles               │
│        (taille variable)                │
└─────────────────────────────────────────┘
```

### Le champ LRU (24 bits)

Ce champ est essentiel pour les politiques d'éviction :

**Mode LRU** (Least Recently Used) :
```
24 bits = timestamp du dernier accès
Format : secondes divisées par une résolution
Permet de couvrir ~194 jours
```

**Mode LFU** (Least Frequently Used) :
```
┌─────────────┬────────────────┐
│ 16 bits     │ 8 bits         │
│ Compteur    │ Decay time     │
│ d'accès     │                │
└─────────────┴────────────────┘
```

Configuration :

```conf
# redis.conf
maxmemory-policy allkeys-lru  # Utilise le champ LRU
# OU
maxmemory-policy allkeys-lfu  # Utilise le champ LFU

# Pour LFU :
lfu-log-factor 10      # Logarithme du compteur (0-255)
lfu-decay-time 1       # Minutes avant decay du compteur
```

## La saturation mémoire : Le moment critique

Lorsque Redis atteint sa limite `maxmemory`, un mécanisme crucial s'active : **l'éviction**.

### Configuration de la limite mémoire

```conf
# redis.conf
maxmemory 2gb           # Limite absolue
maxmemory-policy noeviction  # Politique par défaut (ERREUR si dépassement)
```

### Le processus d'éviction

```
Écriture cliente (SET, LPUSH, etc.)
         ↓
Memory usage > maxmemory ?
         ↓
       OUI ───→ Applique maxmemory-policy
         │              ↓
         │      Sélectionne clés candidates
         │              ↓
         │      Calcule score (LRU/LFU/TTL)
         │              ↓
         │      Supprime clé avec pire score
         │              ↓
         │      Memory usage OK ?
         │              ↓
         │            OUI ──→ Continue écriture
         │              │
         │            NON
         │              ↓
         │           Répète
         ↓
       NON
         ↓
   Continue écriture
```

### Échantillonnage pour l'éviction

Redis n'examine pas TOUTES les clés (trop coûteux), il utilise l'échantillonnage :

```conf
# redis.conf
maxmemory-samples 5  # Défaut : examine 5 clés aléatoires

maxmemory-samples 3  # Plus rapide, moins précis
maxmemory-samples 10 # Plus lent, plus précis (recommandé en prod)
```

**Impact de maxmemory-samples** :

```
samples=3  : ~70% de précision
samples=5  : ~85% de précision (défaut)
samples=10 : ~95% de précision
samples=20 : ~99% de précision (coût CPU élevé)
```

## Observation du cycle de vie

### Commandes de diagnostic

```bash
# Informations sur une clé spécifique
redis-cli OBJECT ENCODING user:123
# Retourne : "hashtable", "ziplist", "int", etc.

redis-cli OBJECT IDLETIME user:123
# Retourne : secondes depuis dernier accès

redis-cli OBJECT FREQ user:123
# Retourne : fréquence d'accès (si LFU activé)

redis-cli OBJECT REFCOUNT user:123
# Retourne : nombre de références

# Inspection du TTL
redis-cli TTL user:123
# Retourne : secondes restantes, -1 (pas de TTL), -2 (clé inexistante)

redis-cli PTTL user:123
# Retourne : millisecondes restantes
```

### Métriques globales

```bash
redis-cli INFO stats
# Observe :
# - expired_keys: Total de clés expirées
# - evicted_keys: Total de clés évictées
# - keyspace_hits: Accès réussis
# - keyspace_misses: Accès manqués

redis-cli INFO memory
# Observe :
# - used_memory: Mémoire utilisée
# - used_memory_rss: Mémoire RSS (système)
# - mem_fragmentation_ratio: Ratio de fragmentation
# - maxmemory: Limite configurée
```

### Suivi en temps réel

```bash
# Monitor les expirations
redis-cli --bigkeys
# Identifie les plus grandes clés

redis-cli --memkeys
# Analyse l'utilisation mémoire par pattern

redis-cli --latency
# Mesure la latence d'expiration
```

## Performance et optimisations

### Coût des opérations de cycle de vie

| Opération | Complexité | Impact |
|-----------|-----------|---------|
| SET avec TTL | O(1) | Insertion dans 2 dictionnaires |
| GET (clé expirée) | O(1) | Vérification + suppression |
| EXPIRE | O(1) | Mise à jour expires dict |
| Expiration active | O(N) | N = nombre d'échantillons (20) |
| Éviction | O(N) | N = maxmemory-samples |

### Bonnes pratiques de configuration

**Pour des TTL courts (< 10s)** :

```conf
hz 50  # Augmente la fréquence de vérification
maxmemory-samples 5  # Échantillonnage standard
```

**Pour des TTL longs (> 1h)** :

```conf
hz 10  # Fréquence standard suffit
maxmemory-samples 10  # Précision accrue pour éviction
```

**Pour haute charge CPU** :

```conf
hz 10  # Minimise les vérifications
maxmemory-samples 3  # Échantillonnage minimal
```

**Pour haute précision d'éviction** :

```conf
maxmemory-samples 20  # Maximum de précision
lfu-log-factor 10  # Comptage LFU précis
lfu-decay-time 1  # Decay rapide
```

## Pièges et considérations

### 1. Fuite de mémoire par clés expirées

**Problème** : Des millions de clés avec TTL qui ne sont jamais accédées.

```bash
# Symptôme
redis-cli DBSIZE
# 10000000

redis-cli INFO stats | grep expired
# expired_keys:0  # Aucune expiration !
```

**Solution** : Augmenter `hz` ou forcer le nettoyage.

### 2. Pic CPU lors d'expirations massives

**Problème** : Expiration simultanée de millions de clés.

```python
# Anti-pattern : Toutes les sessions expirent en même temps
for user_id in range(1000000):
    redis.setex(f"session:{user_id}", 3600, data)
```

**Solution** : Ajouter un jitter aléatoire.

```python
import random
ttl = 3600 + random.randint(-300, 300)  # ±5 minutes
redis.setex(f"session:{user_id}", ttl, data)
```

### 3. Éviction de données importantes

**Problème** : maxmemory-policy évicte des données critiques.

**Solution** : Utiliser `volatile-*` policies et définir TTL uniquement sur les données évictables.

```conf
maxmemory-policy volatile-lru  # N'évicte QUE les clés avec TTL
```

### 4. Fragmentation après évictions massives

**Problème** : Après éviction/expiration, la mémoire RSS ne diminue pas.

```bash
redis-cli INFO memory | grep mem_fragmentation_ratio
# mem_fragmentation_ratio:2.5  # Mauvais !
```

**Solution** : Activer la défragmentation active.

```conf
activedefrag yes
active-defrag-cycle-min 1
active-defrag-cycle-max 25
```

## Vue d'ensemble du module

Ce module vous guidera à travers :

1. **CRUD et gestion d'erreurs** : Les opérations fondamentales sur le cycle de vie
2. **TTL et expiration** : Maîtriser les stratégies d'expiration automatique
3. **Politiques d'éviction** : Comprendre le comportement en saturation mémoire
4. **LRU vs LFU vs Random** : Choisir l'algorithme adapté à votre cas d'usage
5. **Namespaces et nommage** : Organiser efficacement votre keyspace
6. **SCAN vs KEYS** : Naviguer sans bloquer la production

## Prérequis

Avant d'aborder ce module, vous devez maîtriser :
- ✅ Les structures de données Redis (Strings, Lists, Hashes, Sets, Sorted Sets)
- ✅ Les commandes de base (GET, SET, DEL)
- ✅ La notion de keyspace et de databases (0-15)
- ✅ Les concepts de mémoire et performance

## Objectifs d'apprentissage

À la fin de ce module, vous serez capable de :

- 🎯 Comprendre précisément comment Redis gère le cycle de vie des données
- 🎯 Configurer les paramètres d'expiration et d'éviction selon vos besoins
- 🎯 Diagnostiquer et résoudre les problèmes de mémoire
- 🎯 Choisir la bonne stratégie d'éviction pour votre application
- 🎯 Optimiser les performances liées au cycle de vie
- 🎯 Éviter les pièges courants qui mènent à la dégradation de performance

---

**Note importante** : La gestion du cycle de vie n'est pas qu'une question de configuration. C'est une **discipline architecturale** qui requiert une compréhension profonde des mécanismes internes de Redis et des patterns d'accès de votre application. Les mauvaises décisions à ce niveau peuvent transformer Redis en goulot d'étranglement plutôt qu'en accélérateur de performance.

⏭️ [Commandes CRUD fondamentales et gestion des erreurs](/04-cycle-vie-donnee/01-crud-fondamentaux-gestion-erreurs.md)
