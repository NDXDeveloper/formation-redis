🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4 RediSearch : Agrégations et requêtes complexes

## Introduction

La section précédente vous a montré comment effectuer des recherches avec **FT.SEARCH** pour trouver des documents individuels. Mais que faire si vous voulez calculer des **statistiques**, créer des **rapports**, ou construire des **facettes de recherche** ?

**FT.AGGREGATE** transforme RediSearch en un moteur d'**agrégation temps réel**, comparable à Elasticsearch Aggregations ou aux requêtes GROUP BY de SQL, mais avec des performances in-memory exceptionnelles.

Avec FT.AGGREGATE, vous pouvez :
- 📊 **Calculer des statistiques** : Somme, moyenne, min, max, comptage
- 📈 **Grouper des données** : Par catégorie, date, plage de prix
- 🔄 **Transformer les données** : Calculs, formatage, expressions
- 📉 **Créer des facettes** : "5 produits en catégorie laptop, 3 en tablet"
- ⚡ **Performance** : Millions d'agrégations par seconde

---

## FT.SEARCH vs FT.AGGREGATE

### Différence fondamentale

| Critère | FT.SEARCH | FT.AGGREGATE |
|---------|-----------|--------------|
| **Objectif** | Trouver des documents | Calculer des statistiques |
| **Résultat** | Liste de documents | Données agrégées |
| **Exemple SQL** | `SELECT * FROM products WHERE...` | `SELECT category, COUNT(*), AVG(price) FROM products GROUP BY category` |
| **Usage** | Liste de produits, recherche | Dashboard, analytics, facettes |

### Exemple comparatif

```bash
# FT.SEARCH : Trouver tous les laptops
FT.SEARCH idx:products "@category:{laptop}" LIMIT 0 10
# Résultat : 10 produits individuels

# FT.AGGREGATE : Combien de laptops par marque ?
FT.AGGREGATE idx:products "@category:{laptop}"
  GROUPBY 1 @brand
  REDUCE COUNT 0 AS count
# Résultat : Dell: 15, HP: 12, Apple: 8, etc.
```

---

## Syntaxe de base : FT.AGGREGATE

```bash
FT.AGGREGATE {index_name} {query}
  [LOAD {count} {field}...]
  [GROUPBY {count} {field}...
    REDUCE {function} {nargs} {arg}... [AS {name}]
  ]...
  [SORTBY {count} {field} [ASC|DESC]...]
  [APPLY {expression} AS {name}]...
  [LIMIT {offset} {count}]
  [FILTER {expression}]
```

**Structure d'un pipeline d'agrégation** :
1. **Query** : Filtrer les documents (comme FT.SEARCH)
2. **LOAD** : Charger des champs spécifiques
3. **GROUPBY** : Grouper par un ou plusieurs champs
4. **REDUCE** : Calculer des agrégations (COUNT, SUM, AVG, etc.)
5. **APPLY** : Appliquer des transformations
6. **FILTER** : Filtrer les résultats agrégés
7. **SORTBY** : Trier les résultats
8. **LIMIT** : Paginer

---

## Fonctions d'agrégation (REDUCE)

### COUNT : Compter les éléments

```bash
# Compter le nombre de produits par catégorie
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count

# Résultat :
# 1) (integer) 3
# 2) 1) "category"
#    2) "laptop"
#    3) "count"
#    4) "25"
# 3) 1) "category"
#    2) "tablet"
#    3) "count"
#    4) "12"
# 4) 1) "category"
#    2) "accessory"
#    3) "count"
#    4) "43"
```

---

### SUM : Somme

```bash
# Revenu total par catégorie
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE SUM 1 @price AS total_revenue

# Résultat :
# 1) (integer) 3
# 2) 1) "category"
#    2) "laptop"
#    3) "total_revenue"
#    4) "32499.75"  # Somme des prix de tous les laptops
```

---

### AVG : Moyenne

```bash
# Prix moyen par marque
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @brand
  REDUCE AVG 1 @price AS avg_price

# Résultat :
# 1) (integer) 3
# 2) 1) "brand"
#    2) "Dell"
#    3) "avg_price"
#    4) "1149.99"
# 3) 1) "brand"
#    2) "HP"
#    3) "avg_price"
#    4) "899.50"
```

---

### MIN / MAX : Minimum et Maximum

```bash
# Prix min et max par catégorie
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE MIN 1 @price AS min_price
  REDUCE MAX 1 @price AS max_price

# Résultat :
# 1) (integer) 2
# 2) 1) "category"
#    2) "laptop"
#    3) "min_price"
#    4) "699.99"
#    5) "max_price"
#    6) "2499.99"
```

---

### COUNT_DISTINCT : Comptage unique

```bash
# Nombre de marques distinctes par catégorie
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT_DISTINCT 1 @brand AS unique_brands

# Résultat :
# 1) (integer) 2
# 2) 1) "category"
#    2) "laptop"
#    3) "unique_brands"
#    4) "5"  # 5 marques différentes
```

---

### STDDEV : Écart-type

```bash
# Écart-type des prix par catégorie
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE STDDEV 1 @price AS price_stddev

# Utile pour analyser la dispersion des prix
```

---

### QUANTILE : Percentiles

```bash
# Médiane (50e percentile) et 95e percentile des prix
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE QUANTILE 2 @price 0.5 AS median_price
  REDUCE QUANTILE 2 @price 0.95 AS p95_price
```

---

### TOLIST : Collecter les valeurs

```bash
# Liste des prix de tous les produits d'une catégorie
FT.AGGREGATE idx:products "@category:{laptop}"
  GROUPBY 1 @category
  REDUCE TOLIST 1 @price AS all_prices

# Résultat :
# ...
# "all_prices"
# ["1299.99", "899.99", "1499.99", "2499.99", ...]
```

---

### FIRST_VALUE / RANDOM_SAMPLE : Échantillonnage

```bash
# Récupérer le premier produit de chaque catégorie
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE FIRST_VALUE 1 @name AS sample_product

# Échantillon aléatoire
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE RANDOM_SAMPLE 2 @name 3 AS sample_products
# Récupère 3 produits aléatoires par catégorie
```

---

## GROUPBY : Groupement de données

### Groupement simple (1 champ)

```bash
# Nombre de produits par catégorie
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
```

---

### Groupement multiple (plusieurs champs)

```bash
# Nombre de produits par catégorie ET par marque
FT.AGGREGATE idx:products "*"
  GROUPBY 2 @category @brand
  REDUCE COUNT 0 AS count

# Résultat :
# 1) (integer) 6
# 2) 1) "category"
#    2) "laptop"
#    3) "brand"
#    4) "Dell"
#    5) "count"
#    6) "15"
# 3) 1) "category"
#    2) "laptop"
#    3) "brand"
#    4) "HP"
#    5) "count"
#    6) "12"
# 4) 1) "category"
#    2) "laptop"
#    3) "brand"
#    4) "Apple"
#    5) "count"
#    6) "8"
# ...
```

---

### GROUPBY sans champs : Agrégation globale

```bash
# Statistiques globales sur tous les produits
FT.AGGREGATE idx:products "*"
  GROUPBY 0
  REDUCE COUNT 0 AS total_products
  REDUCE SUM 1 @price AS total_value
  REDUCE AVG 1 @price AS avg_price

# Résultat :
# 1) (integer) 1
# 2) 1) "total_products"
#    2) "80"
#    3) "total_value"
#    4) "89999.20"
#    5) "avg_price"
#    6) "1124.99"
```

---

## APPLY : Transformation de données

### Calculs arithmétiques

```bash
# Calculer le revenu total (price × stock)
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE SUM 1 @price AS total_price
  REDUCE SUM 1 @stock AS total_stock
  APPLY "@total_price * @total_stock" AS potential_revenue

# Calculer une remise de 20%
FT.AGGREGATE idx:products "*"
  LOAD 1 @price
  APPLY "@price * 0.8" AS discounted_price
```

---

### Formatage de chaînes

```bash
# Formater une chaîne
FT.AGGREGATE idx:products "*"
  LOAD 2 @name @price
  APPLY "format('%s - $%.2f', @name, @price)" AS formatted_name

# Concaténation
FT.AGGREGATE idx:products "*"
  LOAD 2 @brand @name
  APPLY "@brand . ' ' . @name" AS full_name
```

---

### Fonctions de date/heure

```bash
# Extraire l'année d'un timestamp
FT.AGGREGATE idx:orders "*"
  LOAD 1 @created_at
  APPLY "timefmt(@created_at, '%Y')" AS year

# Grouper par année
FT.AGGREGATE idx:orders "*"
  LOAD 1 @created_at
  APPLY "timefmt(@created_at, '%Y-%m')" AS month
  GROUPBY 1 @month
  REDUCE COUNT 0 AS orders_count
```

---

### Conditions (if/else)

```bash
# Classifier les produits par prix
FT.AGGREGATE idx:products "*"
  LOAD 1 @price
  APPLY "if(@price < 100, 'Budget', if(@price < 500, 'Mid-range', 'Premium'))" AS price_tier
  GROUPBY 1 @price_tier
  REDUCE COUNT 0 AS count
```

---

## SORTBY : Tri des résultats

```bash
# Trier par count décroissant
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  SORTBY 2 @count DESC

# Trier par plusieurs champs
FT.AGGREGATE idx:products "*"
  GROUPBY 2 @category @brand
  REDUCE AVG 1 @price AS avg_price
  SORTBY 4 @category ASC @avg_price DESC
```

**Note** : `SORTBY 2` signifie 2 arguments (champ + direction).

---

## FILTER : Filtrage post-agrégation

```bash
# Ne garder que les catégories avec >10 produits
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  FILTER "@count > 10"

# Filtrer par plusieurs conditions
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @brand
  REDUCE AVG 1 @price AS avg_price
  REDUCE COUNT 0 AS count
  FILTER "@count > 5 && @avg_price < 1000"
```

---

## LIMIT : Pagination

```bash
# Les 10 premières catégories
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  SORTBY 2 @count DESC
  LIMIT 0 10

# Résultats 11-20 (page 2)
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  SORTBY 2 @count DESC
  LIMIT 10 10
```

---

## Cas d'usage modernes

### 1️⃣ Dashboard e-commerce : Analytics en temps réel

**Contexte** : Dashboard affichant les KPIs de vente

#### Données de test

```bash
# Créer l'index orders
FT.CREATE idx:orders
  ON JSON
  PREFIX 1 order:
  SCHEMA
    $.order_id AS order_id NUMERIC
    $.user_id AS user_id NUMERIC
    $.total AS total NUMERIC
    $.status AS status TAG
    $.items_count AS items_count NUMERIC
    $.created_at AS created_at NUMERIC SORTABLE
    $.category AS category TAG

# Insérer des commandes
JSON.SET order:1 $ '{"order_id":1,"user_id":1001,"total":1359.97,"status":"completed","items_count":3,"created_at":1702123200,"category":"electronics"}'
JSON.SET order:2 $ '{"order_id":2,"user_id":1002,"total":299.99,"status":"completed","items_count":1,"created_at":1702126800,"category":"books"}'
JSON.SET order:3 $ '{"order_id":3,"user_id":1001,"total":899.99,"status":"pending","items_count":2,"created_at":1702130400,"category":"electronics"}'
JSON.SET order:4 $ '{"order_id":4,"user_id":1003,"total":1599.99,"status":"completed","items_count":4,"created_at":1702134000,"category":"electronics"}'
JSON.SET order:5 $ '{"order_id":5,"user_id":1004,"total":149.99,"status":"cancelled","items_count":1,"created_at":1702137600,"category":"clothing"}'
```

#### KPI 1 : Statistiques globales

```bash
# Nombre total de commandes, revenu total, panier moyen
FT.AGGREGATE idx:orders "*"
  GROUPBY 0
  REDUCE COUNT 0 AS total_orders
  REDUCE SUM 1 @total AS total_revenue
  REDUCE AVG 1 @total AS avg_order_value
  REDUCE AVG 1 @items_count AS avg_items_per_order

# Résultat :
# 1) (integer) 1
# 2) 1) "total_orders"
#    2) "5"
#    3) "total_revenue"
#    4) "4309.93"
#    5) "avg_order_value"
#    6) "861.99"
#    7) "avg_items_per_order"
#    8) "2.2"
```

#### KPI 2 : Revenu par catégorie

```bash
# Revenu et nombre de commandes par catégorie
FT.AGGREGATE idx:orders "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS orders_count
  REDUCE SUM 1 @total AS revenue
  REDUCE AVG 1 @total AS avg_order_value
  SORTBY 2 @revenue DESC

# Résultat :
# 1) (integer) 3
# 2) 1) "category"
#    2) "electronics"
#    3) "orders_count"
#    4) "3"
#    5) "revenue"
#    6) "3859.95"
#    7) "avg_order_value"
#    8) "1286.65"
# 3) 1) "category"
#    2) "books"
#    3) "orders_count"
#    4) "1"
#    5) "revenue"
#    6) "299.99"
#    7) "avg_order_value"
#    8) "299.99"
# 4) 1) "category"
#    2) "clothing"
#    3) "orders_count"
#    4) "1"
#    5) "revenue"
#    6) "149.99"
#    7) "avg_order_value"
#    8) "149.99"
```

#### KPI 3 : Commandes par statut

```bash
# Distribution des statuts
FT.AGGREGATE idx:orders "*"
  GROUPBY 1 @status
  REDUCE COUNT 0 AS count
  APPLY "@count / 5 * 100" AS percentage
  SORTBY 2 @count DESC

# Résultat :
# 1) (integer) 3
# 2) 1) "status"
#    2) "completed"
#    3) "count"
#    4) "3"
#    5) "percentage"
#    6) "60"
# 3) 1) "status"
#    2) "pending"
#    3) "count"
#    4) "1"
#    5) "percentage"
#    6) "20"
# 4) 1) "status"
#    2) "cancelled"
#    3) "count"
#    4) "1"
#    5) "percentage"
#    6) "20"
```

#### KPI 4 : Analyse temporelle (commandes par jour)

```bash
# Commandes par jour
FT.AGGREGATE idx:orders "*"
  LOAD 1 @created_at
  APPLY "timefmt(@created_at, '%Y-%m-%d')" AS date
  GROUPBY 1 @date
  REDUCE COUNT 0 AS orders_count
  REDUCE SUM 1 @total AS daily_revenue
  SORTBY 2 @date ASC
```

---

### 2️⃣ Facettes de recherche (Filters dynamiques)

**Contexte** : Interface de recherche avec filtres à compteurs

```bash
# Créer l'index produits (si pas déjà fait)
FT.CREATE idx:products
  ON JSON
  PREFIX 1 product:
  SCHEMA
    $.name AS name TEXT
    $.price AS price NUMERIC
    $.category AS category TAG
    $.brand AS brand TAG
    $.stock AS stock NUMERIC
    $.rating AS rating NUMERIC

# Recherche "laptop" avec facettes
# Facette 1 : Nombre de produits par marque
FT.AGGREGATE idx:products "laptop"
  GROUPBY 1 @brand
  REDUCE COUNT 0 AS count
  SORTBY 2 @count DESC

# Résultat :
# Dell: 15, HP: 12, Apple: 8, Lenovo: 5

# Facette 2 : Distribution par plage de prix
FT.AGGREGATE idx:products "laptop"
  LOAD 1 @price
  APPLY "floor(@price / 500) * 500" AS price_range
  GROUPBY 1 @price_range
  REDUCE COUNT 0 AS count
  SORTBY 2 @price_range ASC

# Résultat :
# 500-999€: 12 produits
# 1000-1499€: 18 produits
# 1500-1999€: 8 produits
# 2000+: 2 produits

# Facette 3 : En stock vs Rupture
FT.AGGREGATE idx:products "laptop"
  LOAD 1 @stock
  APPLY "if(@stock > 0, 'In Stock', 'Out of Stock')" AS availability
  GROUPBY 1 @availability
  REDUCE COUNT 0 AS count
```

**Usage côté application** :

```javascript
// Pseudo-code pour une interface web
async function getSearchFacets(query) {
  // Facettes de marques
  const brands = await redis.ft.aggregate('idx:products', query, {
    GROUPBY: '@brand',
    REDUCE: [{ function: 'COUNT', args: [], AS: 'count' }],
    SORTBY: { BY: '@count', DIRECTION: 'DESC' }
  });

  // Facettes de prix
  const prices = await redis.ft.aggregate('idx:products', query, {
    LOAD: ['@price'],
    APPLY: 'floor(@price / 500) * 500 AS price_range',
    GROUPBY: '@price_range',
    REDUCE: [{ function: 'COUNT', args: [], AS: 'count' }]
  });

  return { brands, prices };
}
```

---

### 3️⃣ Analytics de logs : Détection d'anomalies

**Contexte** : Analyser les logs d'erreurs par service

```bash
# Index logs (déjà créé en section 3.3)
FT.CREATE idx:logs
  ON HASH
  PREFIX 1 log:
  SCHEMA
    message TEXT
    level TAG
    service TAG
    timestamp NUMERIC SORTABLE
    response_time NUMERIC

# Insérer des logs
HSET log:1 message "Request successful" level "INFO" service "api" timestamp 1702123200 response_time 45
HSET log:2 message "Database timeout" level "ERROR" service "api" timestamp 1702123205 response_time 5000
HSET log:3 message "Request successful" level "INFO" service "web" timestamp 1702123210 response_time 32
HSET log:4 message "Connection failed" level "ERROR" service "database" timestamp 1702123215 response_time 0
HSET log:5 message "Request successful" level "INFO" service "api" timestamp 1702123220 response_time 38

# Analyse 1 : Nombre d'erreurs par service
FT.AGGREGATE idx:logs "@level:{ERROR}"
  GROUPBY 1 @service
  REDUCE COUNT 0 AS error_count
  SORTBY 2 @error_count DESC

# Analyse 2 : Temps de réponse moyen par service et niveau
FT.AGGREGATE idx:logs "*"
  GROUPBY 2 @service @level
  REDUCE AVG 1 @response_time AS avg_response_time
  REDUCE COUNT 0 AS count
  SORTBY 2 @avg_response_time DESC

# Analyse 3 : Taux d'erreur par service
FT.AGGREGATE idx:logs "*"
  GROUPBY 1 @service
  REDUCE COUNT 0 AS total_logs
  REDUCE COUNT_DISTINCT 1 @level AS distinct_levels
  SORTBY 2 @total_logs DESC

# Analyse 4 : Logs par tranche horaire
FT.AGGREGATE idx:logs "*"
  LOAD 1 @timestamp
  APPLY "timefmt(@timestamp, '%Y-%m-%d %H:00')" AS hour
  GROUPBY 1 @hour
  REDUCE COUNT 0 AS logs_count
  SORTBY 2 @hour ASC
```

---

### 4️⃣ Analyse de performance produit (A/B testing)

**Contexte** : Comparer les performances de variantes de produits

```bash
# Index avec données d'expérimentation
FT.CREATE idx:product_views
  ON HASH
  PREFIX 1 view:
  SCHEMA
    product_id NUMERIC
    variant TAG
    conversion TAG
    revenue NUMERIC
    timestamp NUMERIC

# Insérer des vues
HSET view:1 product_id 101 variant "A" conversion "true" revenue 1299.99 timestamp 1702123200
HSET view:2 product_id 101 variant "B" conversion "false" revenue 0 timestamp 1702123205
HSET view:3 product_id 101 variant "A" conversion "true" revenue 1299.99 timestamp 1702123210
HSET view:4 product_id 101 variant "B" conversion "true" revenue 1299.99 timestamp 1702123215

# Analyse : Taux de conversion par variante
FT.AGGREGATE idx:product_views "*"
  GROUPBY 1 @variant
  REDUCE COUNT 0 AS total_views
  REDUCE COUNT_DISTINCT 1 @conversion AS conversions
  REDUCE SUM 1 @revenue AS total_revenue
  APPLY "@total_revenue / @total_views" AS revenue_per_view
  SORTBY 2 @revenue_per_view DESC

# Résultat :
# Variante A: 2 vues, 2 conversions (100%), 2599.98€
# Variante B: 2 vues, 1 conversion (50%), 1299.99€
```

---

### 5️⃣ Analyse de cohortes utilisateurs

**Contexte** : Analyser le comportement des utilisateurs par cohorte

```bash
# Index utilisateurs avec date d'inscription
FT.CREATE idx:users
  ON JSON
  PREFIX 1 user:
  SCHEMA
    $.user_id AS user_id NUMERIC
    $.signup_date AS signup_date NUMERIC
    $.total_spent AS total_spent NUMERIC
    $.orders_count AS orders_count NUMERIC
    $.country AS country TAG

# Analyse : Statistiques par pays
FT.AGGREGATE idx:users "*"
  GROUPBY 1 @country
  REDUCE COUNT 0 AS users_count
  REDUCE AVG 1 @total_spent AS avg_lifetime_value
  REDUCE AVG 1 @orders_count AS avg_orders
  REDUCE SUM 1 @total_spent AS total_revenue
  SORTBY 2 @total_revenue DESC

# Analyse : Cohortes par mois d'inscription
FT.AGGREGATE idx:users "*"
  LOAD 1 @signup_date
  APPLY "timefmt(@signup_date, '%Y-%m')" AS signup_month
  GROUPBY 1 @signup_month
  REDUCE COUNT 0 AS new_users
  REDUCE AVG 1 @total_spent AS avg_spent
  SORTBY 2 @signup_month ASC
```

---

## Pipelines d'agrégation avancés

### Pipeline multi-étapes

```bash
# Pipeline complexe : Classifier les utilisateurs, puis agréger
FT.AGGREGATE idx:users "*"
  # Étape 1 : Charger les données
  LOAD 2 @total_spent @orders_count
  # Étape 2 : Classifier les utilisateurs
  APPLY "if(@total_spent > 10000, 'VIP', if(@total_spent > 1000, 'Regular', 'Casual'))" AS user_tier
  # Étape 3 : Calculer AOV
  APPLY "@total_spent / @orders_count" AS avg_order_value
  # Étape 4 : Grouper par tier
  GROUPBY 1 @user_tier
  # Étape 5 : Calculer des statistiques
  REDUCE COUNT 0 AS users_count
  REDUCE AVG 1 @avg_order_value AS avg_aov
  REDUCE SUM 1 @total_spent AS total_revenue
  # Étape 6 : Calculer le pourcentage
  APPLY "@total_revenue / 1000000 * 100" AS revenue_percentage
  # Étape 7 : Trier
  SORTBY 2 @total_revenue DESC
```

---

### Nested aggregations (approximation)

RediSearch ne supporte pas directement les agrégations imbriquées, mais on peut les simuler :

```bash
# Objectif : Top 3 produits par catégorie

# Étape 1 : Agréger par catégorie et produit
FT.AGGREGATE idx:orders "*"
  GROUPBY 2 @category @product_id
  REDUCE COUNT 0 AS sales_count
  REDUCE SUM 1 @total AS revenue
  SORTBY 2 @revenue DESC

# Étape 2 : Filtrer côté application pour obtenir le top 3 par catégorie
# (Nécessite une logique applicative)
```

---

## Performance et optimisation

### Benchmark : FT.AGGREGATE vs logique applicative

**Scénario** : Calculer la somme des ventes par catégorie (100 000 commandes)

```bash
# Approche 1 : Logique applicative
# 1. FT.SEARCH idx:orders "*" LIMIT 0 100000
# 2. Pour chaque commande : category → sum
# 3. Temps : ~500-800ms (transfert réseau + calculs)

# Approche 2 : FT.AGGREGATE
FT.AGGREGATE idx:orders "*"
  GROUPBY 1 @category
  REDUCE SUM 1 @total AS revenue
# Temps : ~3-8ms

# Gain : 100-250x plus rapide
```

---

### Impact mémoire des agrégations

```bash
# Les agrégations sont calculées en mémoire
# Pas de stockage persistant des résultats

# Exemple : Agréger 1 million de documents
# Mémoire temporaire : ~50-200 MB pendant le calcul
# Après le calcul : mémoire libérée immédiatement
```

---

### Optimisation des requêtes

#### ✅ Utiliser des filtres avant GROUPBY

```bash
# ✅ Bon : Filtrer avant d'agréger
FT.AGGREGATE idx:orders
  "@status:{completed} @created_at:[1702000000 +inf]"
  GROUPBY 1 @category
  REDUCE SUM 1 @total AS revenue

# ❌ Mauvais : Agréger puis filtrer
FT.AGGREGATE idx:orders "*"
  GROUPBY 1 @category
  REDUCE SUM 1 @total AS revenue
  FILTER "@revenue > 1000"
# (FILTER post-agrégation est OK, mais le filtre de base devrait être dans la query)
```

#### ✅ Limiter les REDUCE

```bash
# ❌ Mauvais : Trop de REDUCE inutiles
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  REDUCE SUM 1 @price AS sum_price
  REDUCE AVG 1 @price AS avg_price
  REDUCE MIN 1 @price AS min_price
  REDUCE MAX 1 @price AS max_price
  REDUCE STDDEV 1 @price AS stddev_price
  REDUCE TOLIST 1 @name AS all_names  # Très coûteux

# ✅ Bon : Seulement les métriques nécessaires
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  REDUCE AVG 1 @price AS avg_price
```

#### ✅ Utiliser LIMIT

```bash
# ✅ Bon : Limiter les résultats
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  SORTBY 2 @count DESC
  LIMIT 0 10  # Top 10 seulement

# ❌ Mauvais : Pas de LIMIT (retourne toutes les catégories)
```

---

## Comparaison : RediSearch vs Elasticsearch vs SQL

### Agrégations équivalentes

**SQL** :
```sql
SELECT category, COUNT(*) as count, AVG(price) as avg_price
FROM products
WHERE stock > 0
GROUP BY category
HAVING COUNT(*) > 10
ORDER BY count DESC
LIMIT 10;
```

**RediSearch** :
```bash
FT.AGGREGATE idx:products "@stock:[1 +inf]"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  REDUCE AVG 1 @price AS avg_price
  FILTER "@count > 10"
  SORTBY 2 @count DESC
  LIMIT 0 10
```

**Elasticsearch** :
```json
{
  "query": { "range": { "stock": { "gt": 0 } } },
  "aggs": {
    "categories": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }
  }
}
```

---

### Performance comparée

| Critère | RediSearch | Elasticsearch | PostgreSQL |
|---------|------------|---------------|------------|
| **Latence** | < 5ms | 20-100ms | 50-500ms |
| **Throughput** | 50K+ agg/sec | 5-10K agg/sec | 1-5K queries/sec |
| **Mémoire** | Élevée (RAM) | Moyenne (cache) | Faible (disque) |
| **Agrégations complexes** | Limitées | Avancées | Avancées (SQL) |
| **Cas d'usage** | Analytics temps réel | Analytics + recherche | OLTP + analytics |

---

## Limitations

### 1. Pas de joins

```bash
# ❌ Impossible de joindre users et orders
# SELECT users.name, SUM(orders.total)
# FROM users JOIN orders ON users.id = orders.user_id
# GROUP BY users.name

# ✅ Workaround : Dénormaliser
JSON.SET order:1 $ '{
  "order_id": 1,
  "user_name": "Alice",  # Dénormalisé
  "total": 1359.97
}'
```

---

### 2. Agrégations imbriquées limitées

```bash
# ❌ Pas de nested aggregations natives
# (Top 3 produits par catégorie nécessite logique applicative)

# ✅ Alternative : Plusieurs requêtes
# 1. FT.AGGREGATE pour lister les catégories
# 2. FT.AGGREGATE par catégorie pour le top 3
```

---

### 3. Pas de window functions

```bash
# ❌ Pas de ROW_NUMBER, RANK, LEAD, LAG comme en SQL

# ✅ Alternative : Implémenter côté application
```

---

## Bonnes pratiques

### ✅ 1. Utiliser FT.AGGREGATE pour les statistiques

```bash
# ✅ Bon : Calculer côté Redis
FT.AGGREGATE idx:orders "*"
  GROUPBY 0
  REDUCE SUM 1 @total AS total_revenue

# ❌ Mauvais : Transférer toutes les données
FT.SEARCH idx:orders "*" RETURN 1 total
# Puis calculer côté application
```

---

### ✅ 2. Combiner avec FT.SEARCH pour la pagination

```bash
# FT.SEARCH pour lister les documents
FT.SEARCH idx:products "laptop" LIMIT 0 20

# FT.AGGREGATE pour les facettes/compteurs
FT.AGGREGATE idx:products "laptop"
  GROUPBY 1 @brand
  REDUCE COUNT 0 AS count
```

---

### ✅ 3. Cacher les résultats d'agrégations coûteuses

```javascript
// Pseudo-code
async function getDashboardStats() {
  const cacheKey = 'dashboard:stats';

  // Vérifier le cache
  let stats = await redis.get(cacheKey);

  if (!stats) {
    // Calculer les stats avec FT.AGGREGATE
    stats = await redis.ft.aggregate('idx:orders', '*', {
      GROUPBY: 0,
      REDUCE: [
        { function: 'COUNT', AS: 'total_orders' },
        { function: 'SUM', args: ['@total'], AS: 'revenue' }
      ]
    });

    // Mettre en cache 5 minutes
    await redis.set(cacheKey, JSON.stringify(stats), 'EX', 300);
  }

  return stats;
}
```

---

### ✅ 4. Préférer TAG pour les GROUP BY

```bash
# ✅ Bon : Grouper par TAG (rapide)
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA category TAG

FT.AGGREGATE idx:products "*" GROUPBY 1 @category

# ⚠️ Attention : Grouper par TEXT (plus lent, tokenisé)
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA category TEXT

# Le groupement fonctionnera mais sera moins efficace
```

---

## Intégration avec les langages

### Python (redis-py)

```python
from redis import Redis
from redis.commands.search.aggregation import AggregateRequest, reducer

r = Redis(decode_responses=True)

# Créer une requête d'agrégation
req = AggregateRequest("*") \
    .group_by('@category', reducer.count().alias('count')) \
    .sort_by('@count', asc=False) \
    .limit(0, 10)

# Exécuter
result = r.ft('idx:products').aggregate(req)

for row in result.rows:
    print(f"{row['category']}: {row['count']}")
```

---

### Node.js (node-redis)

```javascript
const result = await client.ft.aggregate('idx:products', '*', {
  STEPS: [
    {
      type: AggregateSteps.GROUPBY,
      properties: '@category',
      REDUCE: [{
        type: AggregateGroupByReducers.COUNT,
        AS: 'count'
      }]
    },
    {
      type: AggregateSteps.SORTBY,
      BY: {
        property: '@count',
        direction: 'DESC'
      }
    },
    {
      type: AggregateSteps.LIMIT,
      from: 0,
      size: 10
    }
  ]
});

console.log(result);
```

---

## Résumé

**FT.AGGREGATE permet de** :
- ✅ Calculer des statistiques en temps réel (COUNT, SUM, AVG, MIN, MAX)
- ✅ Grouper des données par un ou plusieurs champs
- ✅ Transformer les données avec APPLY
- ✅ Filtrer les résultats agrégés avec FILTER
- ✅ Trier et paginer avec SORTBY et LIMIT
- ✅ Créer des dashboards et facettes de recherche

**Fonctions principales** :
- `COUNT`, `COUNT_DISTINCT` : Comptage
- `SUM`, `AVG`, `MIN`, `MAX` : Statistiques
- `STDDEV`, `QUANTILE` : Statistiques avancées
- `TOLIST`, `FIRST_VALUE` : Échantillonnage

**Cas d'usage idéaux** :
- 📊 Dashboards analytics temps réel
- 🔍 Facettes de recherche dynamiques
- 📈 Reporting et KPIs
- 🔬 Analyse de logs et événements
- 👥 Analyse de cohortes utilisateurs

**Performance** :
- 100-250x plus rapide que logique applicative
- Latence < 5ms pour millions de documents
- Calculs in-memory (pas de stockage persistant)

---

**Prêt pour l'IA ?** Passons à la section suivante : [3.5 RediSearch - Vector Search et cas d'usage IA/RAG](./05-redisearch-vector-search-ia-rag.md)

⏭️ [RediSearch : Vector Search et cas d'usage IA/RAG](/03-structures-donnees-etendues/05-redisearch-vector-search-ia-rag.md)
