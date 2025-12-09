🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.7 Structures probabilistes : HyperLogLog

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Le concept de comptage probabiliste et son utilité
- ✅ Comment HyperLogLog compte des millions d'éléments uniques avec seulement 12 KB
- ✅ Les commandes PFADD, PFCOUNT et PFMERGE
- ✅ La précision et la marge d'erreur (~0.81%)
- ✅ Les cas d'usage réels (analytics, métriques, visiteurs uniques)

---

## 📘 HyperLogLog : Le compteur magique

### Qu'est-ce que HyperLogLog ?

**HyperLogLog** (HLL) est une structure de données **probabiliste** qui permet de compter des éléments **uniques** avec une **mémoire constante** de seulement **12 KB**, quelle que soit la quantité de données.

**Le problème classique** :
```bash
# Compter les visiteurs uniques avec un Set
SADD visitors:2024-12-09 "user:1"
SADD visitors:2024-12-09 "user:2"
SADD visitors:2024-12-09 "user:3"
# ... 10 millions d'utilisateurs

SCARD visitors:2024-12-09
(integer) 10000000

# Mémoire utilisée : ~1 GB 😱
```

**La solution HyperLogLog** :
```bash
# Compter les visiteurs uniques avec HyperLogLog
PFADD visitors:2024-12-09 "user:1"
PFADD visitors:2024-12-09 "user:2"
PFADD visitors:2024-12-09 "user:3"
# ... 10 millions d'utilisateurs

PFCOUNT visitors:2024-12-09
(integer) 9998234  # ≈10M (précision ~99%)

# Mémoire utilisée : 12 KB 🚀
```

**Économie de mémoire : 83 000× moins !**

### Le compromis : Précision vs Mémoire

HyperLogLog offre :
- ✅ **Mémoire constante** : toujours 12 KB maximum
- ✅ **Performance O(1)** : ajout et comptage ultra-rapides
- ⚠️ **Approximation** : erreur standard de **0.81%**

**Exemple d'erreur** :
- Comptage réel : 10 000 000
- HyperLogLog : 9 919 000 à 10 081 000
- Erreur : ±81 000 (0.81%)

Pour la plupart des cas d'usage analytics, c'est **largement acceptable** !

---

## 🔧 Les commandes HyperLogLog

### PFADD : Ajouter des éléments

```bash
# Ajouter un élément
127.0.0.1:6379> PFADD unique:visitors "user:123"
(integer) 1  # 1 = cardinalité estimée a changé

# Ajouter le même élément (doublon)
127.0.0.1:6379> PFADD unique:visitors "user:123"
(integer) 0  # 0 = cardinalité n'a pas changé

# Ajouter plusieurs éléments
127.0.0.1:6379> PFADD unique:visitors "user:456" "user:789" "user:101"
(integer) 1  # Cardinalité a changé

# ⚠️ Note : La valeur de retour n'est PAS le nouveau comptage !
# Elle indique seulement si la cardinalité estimée a changé
```

**Point important** : Vous ne pouvez **pas récupérer** les éléments individuels d'un HyperLogLog. Il ne stocke que des statistiques, pas les données elles-mêmes.

```bash
# ❌ Impossible de faire :
PFMEMBERS unique:visitors  # N'existe pas !

# ✅ Vous pouvez seulement compter
PFCOUNT unique:visitors
```

### PFCOUNT : Compter les éléments uniques

```bash
# Compter les éléments uniques
127.0.0.1:6379> PFCOUNT unique:visitors
(integer) 4  # Estimation : 4 éléments uniques

# Compter sur plusieurs HyperLogLogs (union)
127.0.0.1:6379> PFADD visitors:monday "user:1" "user:2" "user:3"
(integer) 1

127.0.0.1:6379> PFADD visitors:tuesday "user:2" "user:3" "user:4"
(integer) 1

# Comptage union de lundi + mardi
127.0.0.1:6379> PFCOUNT visitors:monday visitors:tuesday
(integer) 4  # user:1, user:2, user:3, user:4 (user:2 et user:3 comptés une seule fois)

# Vous pouvez compter jusqu'à plusieurs HyperLogLogs
127.0.0.1:6379> PFCOUNT hll1 hll2 hll3 hll4 hll5
# Retourne l'union de tous
```

### PFMERGE : Fusionner des HyperLogLogs

```bash
# Créer plusieurs HyperLogLogs
127.0.0.1:6379> PFADD page:A:visitors "user:1" "user:2" "user:3"
(integer) 1

127.0.0.1:6379> PFADD page:B:visitors "user:3" "user:4" "user:5"
(integer) 1

127.0.0.1:6379> PFADD page:C:visitors "user:5" "user:6" "user:7"
(integer) 1

# Fusionner dans un nouveau HyperLogLog
127.0.0.1:6379> PFMERGE total:visitors page:A:visitors page:B:visitors page:C:visitors
OK

# Compter le total
127.0.0.1:6379> PFCOUNT total:visitors
(integer) 7  # user:1 à user:7 (dédupliqués)

# Vérifier les comptages individuels
127.0.0.1:6379> PFCOUNT page:A:visitors
(integer) 3

127.0.0.1:6379> PFCOUNT page:B:visitors
(integer) 3

127.0.0.1:6379> PFCOUNT page:C:visitors
(integer) 3
```

**Note** : PFMERGE est destructif pour la clé de destination. Elle écrase la clé existante.

```bash
# Si la destination existe, elle est écrasée
127.0.0.1:6379> PFADD destination "existing:data"
(integer) 1

127.0.0.1:6379> PFMERGE destination source1 source2
OK  # destination est maintenant l'union de source1 et source2
```

---

## 📊 Cas d'usage #1 : Visiteurs uniques par jour

### Tracking quotidien

```bash
# Lundi 9 décembre 2024
127.0.0.1:6379> PFADD visitors:2024-12-09 "user:123"
(integer) 1

127.0.0.1:6379> PFADD visitors:2024-12-09 "user:456"
(integer) 1

127.0.0.1:6379> PFADD visitors:2024-12-09 "user:123"
(integer) 0  # Doublon, pas compté

127.0.0.1:6379> PFADD visitors:2024-12-09 "user:789"
(integer) 1

# Comptage du jour
127.0.0.1:6379> PFCOUNT visitors:2024-12-09
(integer) 3  # 3 visiteurs uniques aujourd'hui

# Mardi 10 décembre 2024
127.0.0.1:6379> PFADD visitors:2024-12-10 "user:456"
(integer) 1

127.0.0.1:6379> PFADD visitors:2024-12-10 "user:789"
(integer) 1

127.0.0.1:6379> PFADD visitors:2024-12-10 "user:999"
(integer) 1

# Comptage du jour
127.0.0.1:6379> PFCOUNT visitors:2024-12-10
(integer) 3

# Visiteurs uniques sur 2 jours (union)
127.0.0.1:6379> PFCOUNT visitors:2024-12-09 visitors:2024-12-10
(integer) 4  # user:123, user:456, user:789, user:999

# Ajouter un TTL pour nettoyer automatiquement
127.0.0.1:6379> EXPIRE visitors:2024-12-09 2592000
(integer) 1  # Expire dans 30 jours
```

### Visiteurs uniques de la semaine/du mois

```bash
# Créer un HyperLogLog pour la semaine
127.0.0.1:6379> PFMERGE visitors:2024-week-49 \
  visitors:2024-12-09 \
  visitors:2024-12-10 \
  visitors:2024-12-11 \
  visitors:2024-12-12 \
  visitors:2024-12-13 \
  visitors:2024-12-14 \
  visitors:2024-12-15
OK

# Comptage hebdomadaire
127.0.0.1:6379> PFCOUNT visitors:2024-week-49
(integer) 15234  # Visiteurs uniques de la semaine

# Mensuel (union de toutes les semaines)
127.0.0.1:6379> PFMERGE visitors:2024-12 \
  visitors:2024-week-49 \
  visitors:2024-week-50 \
  visitors:2024-week-51 \
  visitors:2024-week-52
OK

127.0.0.1:6379> PFCOUNT visitors:2024-12
(integer) 47821  # Visiteurs uniques du mois
```

**Code application** :
```python
def track_visitor(user_id, date):
    """Tracker un visiteur pour une date donnée"""
    key = f"visitors:{date}"
    redis.pfadd(key, f"user:{user_id}")

    # Définir un TTL de 90 jours
    redis.expire(key, 7776000)

def get_daily_unique_visitors(date):
    """Obtenir le nombre de visiteurs uniques du jour"""
    key = f"visitors:{date}"
    return redis.pfcount(key)

def get_weekly_unique_visitors(year, week):
    """Obtenir le nombre de visiteurs uniques de la semaine"""
    keys = [f"visitors:{year}-{month:02d}-{day:02d}"
            for day in range(7)]  # Simplification
    return redis.pfcount(*keys)
```

---

## 📈 Cas d'usage #2 : Analytics d'articles

```bash
# Article 1 : Tracking des lectures uniques
127.0.0.1:6379> PFADD article:42:unique_views "user:123"
(integer) 1

127.0.0.1:6379> PFADD article:42:unique_views "user:456"
(integer) 1

127.0.0.1:6379> PFADD article:42:unique_views "user:123"
(integer) 0  # Déjà lu par user:123

# Vues totales (avec doublons) : utiliser INCR
127.0.0.1:6379> INCR article:42:total_views
(integer) 1

127.0.0.1:6379> INCR article:42:total_views
(integer) 2

127.0.0.1:6379> INCR article:42:total_views
(integer) 3

# Statistiques de l'article
127.0.0.1:6379> PFCOUNT article:42:unique_views
(integer) 2  # 2 lecteurs uniques

127.0.0.1:6379> GET article:42:total_views
"3"  # 3 vues au total

# Calcul du taux de relecture
# (total_views - unique_views) / total_views
# (3 - 2) / 3 = 33% de relectures
```

**Tracking par source de trafic** :
```bash
# Vues depuis Google
127.0.0.1:6379> PFADD article:42:source:google "user:123" "user:456"
(integer) 1

# Vues depuis Twitter
127.0.0.1:6379> PFADD article:42:source:twitter "user:789" "user:999"
(integer) 1

# Vues depuis email
127.0.0.1:6379> PFADD article:42:source:email "user:456" "user:999"
(integer) 1

# Total de lecteurs uniques (tous canaux confondus)
127.0.0.1:6379> PFCOUNT article:42:source:google article:42:source:twitter article:42:source:email
(integer) 4  # user:456 et user:999 dédupliqués

# Lecteurs uniques par canal
127.0.0.1:6379> PFCOUNT article:42:source:google
(integer) 2

127.0.0.1:6379> PFCOUNT article:42:source:twitter
(integer) 2

127.0.0.1:6379> PFCOUNT article:42:source:email
(integer) 2
```

---

## 🌐 Cas d'usage #3 : Métriques d'API

```bash
# Tracker les IP uniques appelant l'API
127.0.0.1:6379> PFADD api:endpoint:users:unique_ips "192.168.1.1"
(integer) 1

127.0.0.1:6379> PFADD api:endpoint:users:unique_ips "192.168.1.2"
(integer) 1

127.0.0.1:6379> PFADD api:endpoint:users:unique_ips "192.168.1.1"
(integer) 0  # Déjà compté

# Nombre d'IP uniques
127.0.0.1:6379> PFCOUNT api:endpoint:users:unique_ips
(integer) 2

# Tracker les user agents uniques
127.0.0.1:6379> PFADD api:endpoint:users:unique_agents "Mozilla/5.0" "curl/7.68.0"
(integer) 1

127.0.0.1:6379> PFCOUNT api:endpoint:users:unique_agents
(integer) 2

# Tracker les utilisateurs authentifiés uniques
127.0.0.1:6379> PFADD api:endpoint:users:unique_users "user:123" "user:456"
(integer) 1

127.0.0.1:6379> PFCOUNT api:endpoint:users:unique_users
(integer) 2
```

**Dashboard d'API** :
```python
def track_api_call(endpoint, ip, user_agent, user_id=None):
    """Tracker un appel API"""
    date = datetime.now().strftime("%Y-%m-%d")

    # IP uniques
    redis.pfadd(f"api:{endpoint}:{date}:ips", ip)

    # User agents uniques
    redis.pfadd(f"api:{endpoint}:{date}:agents", user_agent)

    # Utilisateurs authentifiés uniques
    if user_id:
        redis.pfadd(f"api:{endpoint}:{date}:users", user_id)

    # Total des appels (avec doublons)
    redis.incr(f"api:{endpoint}:{date}:total")

def get_api_metrics(endpoint, date):
    """Obtenir les métriques d'un endpoint"""
    return {
        "unique_ips": redis.pfcount(f"api:{endpoint}:{date}:ips"),
        "unique_agents": redis.pfcount(f"api:{endpoint}:{date}:agents"),
        "unique_users": redis.pfcount(f"api:{endpoint}:{date}:users"),
        "total_calls": int(redis.get(f"api:{endpoint}:{date}:total") or 0)
    }
```

---

## 🔬 Cas d'usage #4 : Dédoublonnage de recherches

```bash
# Tracker les requêtes uniques sur un site de recherche
127.0.0.1:6379> PFADD search:queries:2024-12-09 "redis tutorial"
(integer) 1

127.0.0.1:6379> PFADD search:queries:2024-12-09 "python redis"
(integer) 1

127.0.0.1:6379> PFADD search:queries:2024-12-09 "redis tutorial"
(integer) 0  # Doublon

127.0.0.1:6379> PFADD search:queries:2024-12-09 "nodejs redis"
(integer) 1

# Nombre de requêtes uniques
127.0.0.1:6379> PFCOUNT search:queries:2024-12-09
(integer) 3

# Tracker par catégorie
127.0.0.1:6379> PFADD search:category:tech:queries "redis" "python" "docker"
(integer) 1

127.0.0.1:6379> PFADD search:category:sports:queries "football" "basketball"
(integer) 1

# Requêtes uniques dans tech
127.0.0.1:6379> PFCOUNT search:category:tech:queries
(integer) 3

# Requêtes uniques dans sports
127.0.0.1:6379> PFCOUNT search:category:sports:queries
(integer) 2

# Total de requêtes uniques (tous domaines)
127.0.0.1:6379> PFCOUNT search:category:tech:queries search:category:sports:queries
(integer) 5
```

---

## 💰 Cas d'usage #5 : E-commerce - Produits vus

```bash
# Tracker les produits vus par chaque utilisateur
127.0.0.1:6379> PFADD user:123:products_viewed "product:1" "product:2" "product:3"
(integer) 1

127.0.0.1:6379> PFADD user:123:products_viewed "product:2"
(integer) 0  # Déjà vu

127.0.0.1:6379> PFADD user:123:products_viewed "product:4"
(integer) 1

# Nombre de produits uniques vus par l'utilisateur
127.0.0.1:6379> PFCOUNT user:123:products_viewed
(integer) 4

# Inverser : utilisateurs uniques ayant vu un produit
127.0.0.1:6379> PFADD product:1:unique_viewers "user:123" "user:456" "user:789"
(integer) 1

127.0.0.1:6379> PFCOUNT product:1:unique_viewers
(integer) 3

# Produits vus par catégorie
127.0.0.1:6379> PFADD user:123:viewed:electronics "product:1" "product:2"
(integer) 1

127.0.0.1:6379> PFADD user:123:viewed:books "product:10" "product:11"
(integer) 1

# Combien de produits électroniques vus ?
127.0.0.1:6379> PFCOUNT user:123:viewed:electronics
(integer) 2
```

---

## 🆚 HyperLogLog vs Set : Quand utiliser quoi ?

### Comparaison : 1 million d'utilisateurs uniques

| Critère | Set | HyperLogLog |
|---------|-----|-------------|
| **Mémoire** | ~100 MB | 12 KB |
| **Précision** | 100% (exacte) | ~99.19% (±0.81%) |
| **Peut lister les membres** | ✅ Oui (SMEMBERS) | ❌ Non |
| **Opérations ensemblistes** | ✅ SUNION, SINTER, SDIFF | ⚠️ PFCOUNT (union seulement) |
| **Vérifier appartenance** | ✅ SISMEMBER | ❌ Impossible |
| **Complexité ajout** | O(1) | O(1) |
| **Complexité comptage** | O(1) | O(1) |
| **Cas d'usage** | < 10 000 membres, besoin de précision | > 100 000 membres, approximation OK |

### Tableau de décision

```bash
# ✅ Utilisez un Set si :
# - Vous avez besoin de précision exacte
# - Vous devez lister les membres individuels
# - Vous avez < 10 000 éléments uniques
# - Vous avez besoin d'opérations ensemblistes complexes
# - Vous devez vérifier l'appartenance d'un élément

SADD active:users "user:123"
SISMEMBER active:users "user:123"  # Vérification possible
SMEMBERS active:users              # Listage possible

# ✅ Utilisez HyperLogLog si :
# - Vous comptez > 100 000 éléments uniques
# - L'approximation (~1% d'erreur) est acceptable
# - La mémoire est une contrainte
# - Vous ne devez jamais lister les membres
# - Vous faites de l'analytics, des métriques

PFADD visitors "user:123"
PFCOUNT visitors               # Comptage uniquement
# Pas de PFISMEMBER ou PFMEMBERS !
```

### Exemple de migration Set → HyperLogLog

```python
# Scénario : Vous commencez avec un Set, mais ça grandit trop

# Phase 1 : Petit volume (< 10 000 utilisateurs) → Set
def track_visitor_v1(user_id):
    redis.sadd("visitors", user_id)

def get_count_v1():
    return redis.scard("visitors")

# Phase 2 : Volume croissant → Migration vers HyperLogLog
def migrate_to_hll():
    """Migrer les données existantes"""
    members = redis.smembers("visitors")

    for member in members:
        redis.pfadd("visitors:hll", member)

    # Vérifier
    set_count = redis.scard("visitors")
    hll_count = redis.pfcount("visitors:hll")

    print(f"Set: {set_count}, HLL: {hll_count}")
    # Set: 10000, HLL: 9982 (erreur de 0.18%, acceptable)

    # Renommer et supprimer l'ancien Set
    redis.rename("visitors", "visitors:old:set")
    redis.rename("visitors:hll", "visitors")
    redis.delete("visitors:old:set")

# Phase 3 : Nouveau code avec HyperLogLog
def track_visitor_v2(user_id):
    redis.pfadd("visitors", user_id)

def get_count_v2():
    return redis.pfcount("visitors")
```

---

## 🧮 Comprendre la précision

### L'erreur standard : 0.81%

HyperLogLog a une erreur standard de **0.81%** (ou 1/√m où m = 16384 registres).

**Exemples de précision** :

| Comptage réel | Plage HyperLogLog (99% confiance) | Erreur absolue max |
|---------------|-----------------------------------|---------------------|
| 1 000 | 992 à 1 008 | ±8 |
| 10 000 | 9 919 à 10 081 | ±81 |
| 100 000 | 99 190 à 100 810 | ±810 |
| 1 000 000 | 991 900 à 1 008 100 | ±8 100 |
| 10 000 000 | 9 919 000 à 10 081 000 | ±81 000 |
| 100 000 000 | 99 190 000 à 100 810 000 | ±810 000 |

**Observation** : L'erreur **absolue** augmente avec le nombre d'éléments, mais l'erreur **relative** reste à ~0.81%.

### Test de précision

```bash
# Ajouter beaucoup d'éléments
127.0.0.1:6379> PFADD test:precision "elem:1"
(integer) 1

# Ajouter 10 000 éléments uniques (script)
# for i in range(1, 10001):
#     redis.pfadd("test:precision", f"elem:{i}")

127.0.0.1:6379> PFCOUNT test:precision
(integer) 10043  # Réel : 10 000, Estimé : 10 043 (erreur de 0.43%)

# Ajouter 100 000 éléments
# for i in range(1, 100001):
#     redis.pfadd("test:precision2", f"elem:{i}")

127.0.0.1:6379> PFCOUNT test:precision2
(integer) 99754  # Réel : 100 000, Estimé : 99 754 (erreur de 0.25%)

# L'erreur est aléatoire et respecte la distribution statistique
```

---

## ⚡ Performance et mémoire

### Utilisation de mémoire

```bash
# Créer un HyperLogLog vide
127.0.0.1:6379> PFADD empty:hll "dummy"
(integer) 1

127.0.0.1:6379> MEMORY USAGE empty:hll
(integer) 15280  # ~15 KB (proche de 12 KB théorique + overhead Redis)

# Ajouter 1 million d'éléments
# for i in range(1, 1000001):
#     redis.pfadd("huge:hll", f"user:{i}")

127.0.0.1:6379> MEMORY USAGE huge:hll
(integer) 15280  # Toujours ~15 KB ! 🚀

# Comparaison avec un Set
127.0.0.1:6379> SADD huge:set "user:1" "user:2" "user:3"
# ... (ajouter 1000 éléments)

127.0.0.1:6379> MEMORY USAGE huge:set
(integer) 87340  # ~87 KB pour 1000 éléments seulement

# Pour 1 million : ~87 MB vs 15 KB (5800× moins de mémoire !)
```

### Performance des opérations

| Opération | Complexité | Notes |
|-----------|------------|-------|
| `PFADD` | O(1) | Très rapide |
| `PFCOUNT` | O(1) | Avec 1 HLL |
| `PFCOUNT` | O(N) | Avec N HLLs (union) |
| `PFMERGE` | O(N) | N = nombre de HLLs à fusionner |

**Benchmark** (sur machine standard) :
```bash
# 1 million de PFADD
Temps : ~2 secondes
Throughput : ~500 000 ops/sec

# 1 million de SADD (pour comparaison)
Temps : ~2 secondes
Throughput : ~500 000 ops/sec

# La différence n'est pas dans la vitesse, mais dans la mémoire !
```

---

## 🚨 Limitations et pièges

### 1. Impossible de récupérer les éléments

```bash
# ❌ Vous NE POUVEZ PAS faire :
PFMEMBERS visitors      # N'existe pas
PFISMEMBER visitors "user:123"  # N'existe pas

# HyperLogLog ne stocke pas les valeurs individuelles !
# Seulement des statistiques pour le comptage
```

### 2. Pas de suppression d'éléments

```bash
# ❌ Impossible de retirer un élément d'un HyperLogLog
PFREM visitors "user:123"  # N'existe pas !

# Une fois ajouté, un élément influence le comptage pour toujours
# La seule option : supprimer tout le HyperLogLog
DEL visitors
```

### 3. L'approximation peut surprendre

```bash
# Ajouter 10 éléments
127.0.0.1:6379> PFADD small:hll "1" "2" "3" "4" "5" "6" "7" "8" "9" "10"
(integer) 1

127.0.0.1:6379> PFCOUNT small:hll
(integer) 10  # Peut retourner 9 ou 11 !

# Sur de petits échantillons (< 100), l'erreur relative peut être plus grande
# HyperLogLog est conçu pour de GRANDS volumes
```

### 4. PFMERGE écrase la destination

```bash
127.0.0.1:6379> PFADD dest "existing"
(integer) 1

127.0.0.1:6379> PFCOUNT dest
(integer) 1

127.0.0.1:6379> PFMERGE dest source1 source2
OK

127.0.0.1:6379> PFCOUNT dest
(integer) 5  # Les anciennes données de "dest" sont perdues !
```

### 5. Confusion avec les préfixes "PF"

```bash
# Pourquoi "PF" ?
# En hommage à Philippe Flajolet, mathématicien français
# qui a contribué aux algorithmes de comptage probabiliste

# Les commandes :
PFADD    # Probabilistic Flajolet ADD
PFCOUNT  # Probabilistic Flajolet COUNT
PFMERGE  # Probabilistic Flajolet MERGE
```

---

## 📋 Checklist : Quand utiliser HyperLogLog

### ✅ Utilisez HyperLogLog pour :
- Compter des **visiteurs uniques** (analytics web)
- Métriques d'**API** (IP uniques, users uniques)
- **Recherches uniques** sur un moteur de recherche
- Tout comptage **> 100 000 éléments uniques**
- Quand l'**approximation (±1%)** est acceptable
- Quand la **mémoire** est une contrainte forte
- **Métriques temps réel** avec agrégations (jour/semaine/mois)

### ❌ N'utilisez PAS HyperLogLog pour :
- Comptage de **< 10 000 éléments** → Set est mieux
- Besoin de **précision exacte** (transactions financières) → Set
- Besoin de **lister les membres** → Set ou Sorted Set
- Besoin de **supprimer des éléments** → Set
- **Vérifier l'appartenance** d'un élément → Set (SISMEMBER)
- Opérations ensemblistes **complexes** (intersection, différence) → Set

---

## 🎓 Points clés à retenir

1. ✅ **HyperLogLog = compteur magique** : 12 KB pour compter des milliards d'éléments
2. ✅ **Erreur ~0.81%** : acceptable pour analytics, métriques, statistiques
3. ✅ **Mémoire constante** : 12 KB quel que soit le volume
4. ✅ **O(1) pour PFADD et PFCOUNT** : ultra-rapide
5. ✅ **PFMERGE** : fusionner plusieurs HLLs pour agrégations
6. ⚠️ **Probabiliste** : pas de garantie de précision exacte
7. ⚠️ **Pas de récupération** : impossible de lister ou vérifier les membres
8. ⚠️ **Pas de suppression** : impossible de retirer un élément
9. 🎯 Idéal pour : visiteurs uniques, analytics, métriques à grande échelle

---

## 🚀 Prochaine étape

Vous connaissez maintenant HyperLogLog pour le comptage unique probabiliste. Explorons une autre structure spécialisée : les **Bitmaps** pour gérer efficacement des états binaires !

➡️ **Section suivante** : [2.8 Bitmaps : Gestion efficace d'états binaires](./08-bitmaps-etats-binaires.md)

---

**Durée estimée** : 1h
**Niveau** : Intermédiaire
**Prérequis** : Sections 2.1 à 2.6 complétées

⏭️ [Bitmaps : Gestion efficace d'états binaires](/02-structures-donnees-natives/08-bitmaps-etats-binaires.md)
