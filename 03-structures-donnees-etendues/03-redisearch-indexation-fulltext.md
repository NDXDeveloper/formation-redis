🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3 RediSearch : Indexation et Full-text search

## Introduction

Avec Redis Core, rechercher des données nécessite de connaître la clé exacte ou d'utiliser `SCAN` pour parcourir toutes les clés. Avec RedisJSON, vous pouvez manipuler des documents JSON, mais sans capacité de recherche.

**RediSearch** comble cette lacune en ajoutant des **capacités d'indexation et de recherche** à Redis, transformant votre base en mémoire en un **moteur de recherche full-text ultra-rapide**.

RediSearch permet de :
- 🔍 **Recherche full-text** : Trouver des documents par leur contenu textuel
- 🎯 **Filtrage avancé** : Combiner plusieurs critères (prix, catégorie, stock)
- 📊 **Tri et pagination** : Ordonner les résultats par pertinence ou autre champ
- 🚀 **Performance** : Recherches indexées en O(log N) vs O(N) pour SCAN
- 🔗 **Intégration** : Fonctionne avec Hashes et RedisJSON

---

## Pourquoi RediSearch ?

### Le problème sans RediSearch

**Scénario** : Vous avez 100 000 produits et voulez trouver tous les laptops Dell entre 800€ et 1500€, triés par prix.

```bash
# Avec Redis Core : SCAN + Filtrage applicatif
SCAN 0 MATCH product:* COUNT 1000
# → Récupérer chaque clé
# → Désérialiser chaque document
# → Filtrer côté application (marque, prix, catégorie)
# → Trier côté application
# Temps : 2-5 secondes pour 100K produits
```

**Avec RediSearch** :

```bash
# Recherche indexée
FT.SEARCH idx:products
  "@category:{laptop} @brand:{Dell} @price:[800 1500]"
  SORTBY price ASC
  LIMIT 0 20

# Temps : 2-5 millisecondes
# Gain : 1000x plus rapide
```

---

## Architecture RediSearch

```
┌─────────────────────────────────────────────────────────┐
│                    Application                          │
└───────────────┬─────────────────────────────────────────┘
                │
        ┌───────▼──────────┐
        │   FT.SEARCH      │ (Query)
        │   FT.CREATE      │ (Index definition)
        └───────┬──────────┘
                │
┌───────────────▼─────────────────────────────────────────┐
│                   RediSearch Module                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │            Inverted Index                        │   │
│  │  "laptop" → [doc:1, doc:5, doc:12, ...]          │   │
│  │  "dell"   → [doc:1, doc:3, doc:8, ...]           │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │         Secondary Indexes                        │   │
│  │  price: BTree index                              │   │
│  │  category: Tag index                             │   │
│  └──────────────────────────────────────────────────┘   │
└───────────────┬─────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────┐
│              Redis Core Data Structures                 │
│  product:1 → HASH or JSON                               │
│  product:2 → HASH or JSON                               │
│  ...                                                    │
└─────────────────────────────────────────────────────────┘
```

**Points clés** :
- RediSearch crée des **index secondaires** sur vos données existantes
- Les données restent dans Hashes ou RedisJSON
- L'index est maintenu **automatiquement** lors des mises à jour

---

## Installation et vérification

### Vérifier que RediSearch est disponible

```bash
# Vérifier les modules chargés
redis-cli MODULE LIST

# Devrait contenir :
# 1) 1) "name"
#    2) "search"
#    3) "ver"
#    4) 20800  # Version 2.8.0
```

### Avec Docker (Redis Stack)

```bash
# Démarrer Redis Stack avec RediSearch
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest

# Tester RediSearch
redis-cli FT._LIST
# Résultat : (empty array) si aucun index n'existe
```

---

## Créer un index : FT.CREATE

### Syntaxe de base

```bash
FT.CREATE {index_name}
  ON {HASH | JSON}
  PREFIX {count} {prefix}...
  SCHEMA {field_name} {field_type} [options]...
```

### Exemple simple : Index sur des Hashes

```bash
# 1. Créer des produits (Hashes)
HSET product:101 name "Laptop Dell XPS 13" price 1299.99 category "laptop" brand "Dell" stock 45
HSET product:102 name "Mouse Logitech MX Master" price 99.99 category "accessory" brand "Logitech" stock 150
HSET product:103 name "Laptop HP Pavilion 15" price 899.99 category "laptop" brand "HP" stock 30

# 2. Créer un index
FT.CREATE idx:products
  ON HASH
  PREFIX 1 product:
  SCHEMA
    name TEXT WEIGHT 5.0 SORTABLE
    price NUMERIC SORTABLE
    category TAG SORTABLE
    brand TAG
    stock NUMERIC
```

**Explication** :
- `ON HASH` : Indexe des Hashes (alternatif : `ON JSON`)
- `PREFIX 1 product:` : Indexe toutes les clés commençant par `product:`
- `SCHEMA` : Définition des champs indexés
  - `name TEXT` : Recherche full-text sur ce champ
  - `WEIGHT 5.0` : Poids pour le scoring de pertinence (défaut : 1.0)
  - `SORTABLE` : Permet de trier par ce champ
  - `price NUMERIC` : Recherche numérique (ranges)
  - `category TAG` : Recherche exacte (pas de tokenisation)

---

### Exemple avec RedisJSON

```bash
# 1. Créer des produits JSON
JSON.SET product:201 $ '{"name":"MacBook Pro 16","price":2499.99,"category":"laptop","brand":"Apple","specs":{"ram":"16GB","cpu":"M3 Pro"},"stock":20}'
JSON.SET product:202 $ '{"name":"iPad Air","price":599.99,"category":"tablet","brand":"Apple","specs":{"ram":"8GB","cpu":"M2"},"stock":80}'

# 2. Créer un index sur JSON
FT.CREATE idx:products_json
  ON JSON
  PREFIX 1 product:
  SCHEMA
    $.name AS name TEXT WEIGHT 5.0 SORTABLE
    $.price AS price NUMERIC SORTABLE
    $.category AS category TAG SORTABLE
    $.brand AS brand TAG
    $.stock AS stock NUMERIC
    $.specs.ram AS ram TAG
```

**Note JSONPath** : Utilisez `$.path` pour accéder aux champs JSON, puis `AS alias` pour nommer le champ dans l'index.

---

## Types de champs

### 1️⃣ TEXT : Recherche full-text

**Usage** : Descriptions, titres, contenu textuel

```bash
# Création
SCHEMA description TEXT WEIGHT 2.0 SORTABLE

# Caractéristiques :
# - Tokenisation automatique (split par espaces, ponctuation)
# - Stemming (optionnel) : "running" → "run"
# - Support de la pertinence (scoring)
# - Support des langues (anglais, français, etc.)
```

**Options** :
- `WEIGHT` : Influence sur le score de pertinence (défaut : 1.0)
- `SORTABLE` : Permet le tri par ce champ (coût mémoire +30%)
- `NOSTEM` : Désactive le stemming
- `PHONETIC` : Recherche phonétique (ex: "Jon" trouve "John")

---

### 2️⃣ TAG : Recherche exacte

**Usage** : Catégories, tags, énumérations

```bash
# Création
SCHEMA category TAG SEPARATOR ","

# Caractéristiques :
# - Pas de tokenisation (valeur exacte)
# - Idéal pour les filtres (categories, statuts, pays)
# - Supporte plusieurs valeurs séparées
```

**Exemple** :

```bash
HSET product:101 tags "premium,featured,bestseller"

# Créer l'index avec séparateur
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA tags TAG SEPARATOR ","

# Rechercher les produits "premium"
FT.SEARCH idx:products "@tags:{premium}"
```

---

### 3️⃣ NUMERIC : Recherche numérique

**Usage** : Prix, quantités, scores, timestamps

```bash
# Création
SCHEMA price NUMERIC SORTABLE

# Caractéristiques :
# - Recherche par range [min max]
# - Comparaisons (<, >, <=, >=)
# - Tri efficace
```

**Exemples** :

```bash
# Produits entre 100€ et 500€
FT.SEARCH idx:products "@price:[100 500]"

# Produits > 1000€
FT.SEARCH idx:products "@price:[1000 +inf]"

# Produits < 50€
FT.SEARCH idx:products "@price:[-inf 50]"
```

---

### 4️⃣ GEO : Recherche géospatiale

**Usage** : Coordonnées GPS, localisation

```bash
# Création
SCHEMA location GEO

# Format de données : "longitude,latitude"
HSET store:1 name "Store Paris" location "2.3522,48.8566"

# Recherche dans un rayon de 5km autour d'un point
FT.SEARCH idx:stores "@location:[2.3522 48.8566 5 km]"
```

---

## Recherche : FT.SEARCH

### Syntaxe de base

```bash
FT.SEARCH {index_name} {query}
  [LIMIT {offset} {count}]
  [SORTBY {field} [ASC|DESC]]
  [RETURN {count} {field}...]
```

### Exemple simple

```bash
# Recherche simple : tous les produits contenant "laptop"
FT.SEARCH idx:products "laptop"

# Résultat :
# 1) (integer) 2  # Nombre de résultats
# 2) "product:101"
# 3) 1) "name"
#    2) "Laptop Dell XPS 13"
#    3) "price"
#    4) "1299.99"
#    ...
# 4) "product:103"
# 5) 1) "name"
#    2) "Laptop HP Pavilion 15"
#    ...
```

---

## Syntaxe de requête avancée

### Recherche dans un champ spécifique

```bash
# Recherche "dell" uniquement dans le champ "brand"
FT.SEARCH idx:products "@brand:dell"

# Recherche "laptop" dans le champ "name"
FT.SEARCH idx:products "@name:laptop"
```

---

### Opérateurs booléens

#### AND (implicite ou explicite)

```bash
# Recherche "laptop dell" (AND implicite)
FT.SEARCH idx:products "laptop dell"

# Recherche explicite avec AND
FT.SEARCH idx:products "laptop & dell"
```

#### OR

```bash
# Recherche "dell" OU "hp"
FT.SEARCH idx:products "dell | hp"

# Dans un champ spécifique
FT.SEARCH idx:products "@brand:(dell | hp)"
```

#### NOT

```bash
# Recherche "laptop" mais PAS "dell"
FT.SEARCH idx:products "laptop -dell"

# Avec @field
FT.SEARCH idx:products "@category:{laptop} -@brand:{dell}"
```

---

### Recherche par range (NUMERIC)

```bash
# Prix entre 500 et 1500
FT.SEARCH idx:products "@price:[500 1500]"

# Prix >= 1000 (infini supérieur)
FT.SEARCH idx:products "@price:[1000 +inf]"

# Prix < 100
FT.SEARCH idx:products "@price:[-inf (100]"
# Note: ( signifie "exclusif"

# Prix exactement 99.99
FT.SEARCH idx:products "@price:[99.99 99.99]"
```

---

### Recherche par TAG (exacte)

```bash
# Une seule valeur
FT.SEARCH idx:products "@category:{laptop}"

# Plusieurs valeurs (OR)
FT.SEARCH idx:products "@category:{laptop | tablet}"

# Avec échappement (espaces)
FT.SEARCH idx:products "@brand:{Dell\\ Inc}"
```

---

### Wildcards et préfixes

```bash
# Préfixe : tous les produits commençant par "lap"
FT.SEARCH idx:products "lap*"

# Suffixe (non supporté directement, utiliser NOSTEM)
# Workaround : Indexer à l'envers

# Infix (non supporté directement)
```

---

### Phrases exactes

```bash
# Recherche la phrase exacte "dell xps"
FT.SEARCH idx:products '"dell xps"'

# Avec tolérance (1 mot manquant)
FT.SEARCH idx:products '"dell xps"~1'
```

---

### Proximité

```bash
# Recherche "dell" et "laptop" distants de max 3 mots
FT.SEARCH idx:products "dell laptop"~3
```

---

## Options de recherche

### LIMIT : Pagination

```bash
# Les 10 premiers résultats
FT.SEARCH idx:products "laptop" LIMIT 0 10

# Résultats 11-20 (page 2)
FT.SEARCH idx:products "laptop" LIMIT 10 10

# Résultats 21-30 (page 3)
FT.SEARCH idx:products "laptop" LIMIT 20 10
```

---

### SORTBY : Tri des résultats

```bash
# Trier par prix croissant
FT.SEARCH idx:products "laptop" SORTBY price ASC

# Trier par prix décroissant
FT.SEARCH idx:products "laptop" SORTBY price DESC

# Par défaut : tri par pertinence (score)
FT.SEARCH idx:products "laptop"
```

**Attention** : Le champ doit être `SORTABLE` dans le schéma.

---

### RETURN : Sélection des champs retournés

```bash
# Retourner uniquement name et price
FT.SEARCH idx:products "laptop" RETURN 2 name price

# Résultat :
# 1) (integer) 2
# 2) "product:101"
# 3) 1) "name"
#    2) "Laptop Dell XPS 13"
#    3) "price"
#    4) "1299.99"
# ...

# Retourner tous les champs (comportement par défaut)
FT.SEARCH idx:products "laptop"

# Ne rien retourner (seulement les IDs et count)
FT.SEARCH idx:products "laptop" NOCONTENT
```

---

### HIGHLIGHT : Mise en évidence des termes

```bash
# Entourer les termes de recherche avec des tags
FT.SEARCH idx:products "laptop dell"
  HIGHLIGHT FIELDS 1 name
  TAGS "<b>" "</b>"

# Résultat :
# ...
# "name"
# "<b>Laptop</b> <b>Dell</b> XPS 13"
```

---

### SUMMARIZE : Extraits de texte

```bash
# Créer un résumé du champ "description"
FT.SEARCH idx:products "redis"
  SUMMARIZE FIELDS 1 description
  FRAGS 3
  LEN 20

# Résultat : 3 fragments de 20 mots max contenant "redis"
```

---

## Cas d'usage modernes

### 1️⃣ Moteur de recherche e-commerce

**Contexte** : Marketplace avec recherche avancée

```bash
# Créer l'index produits
FT.CREATE idx:products
  ON JSON
  PREFIX 1 product:
  SCHEMA
    $.name AS name TEXT WEIGHT 5.0 SORTABLE
    $.description AS description TEXT WEIGHT 2.0
    $.price AS price NUMERIC SORTABLE
    $.category AS category TAG SORTABLE
    $.brand AS brand TAG SORTABLE
    $.stock AS stock NUMERIC
    $.rating AS rating NUMERIC SORTABLE
    $.created_at AS created_at NUMERIC SORTABLE

# Insérer des produits
JSON.SET product:101 $ '{
  "name": "Laptop Dell XPS 13",
  "description": "Ultrabook professionnel avec écran 4K et processeur Intel Core i7",
  "price": 1299.99,
  "category": "laptop",
  "brand": "Dell",
  "stock": 45,
  "rating": 4.7,
  "created_at": 1702123200
}'

JSON.SET product:102 $ '{
  "name": "Laptop HP Pavilion 15",
  "description": "Ordinateur portable polyvalent pour le quotidien",
  "price": 899.99,
  "category": "laptop",
  "brand": "HP",
  "stock": 30,
  "rating": 4.2,
  "created_at": 1702209600
}'

JSON.SET product:103 $ '{
  "name": "Mouse Logitech MX Master 3",
  "description": "Souris ergonomique sans fil pour professionnels",
  "price": 99.99,
  "category": "accessory",
  "brand": "Logitech",
  "stock": 150,
  "rating": 4.8,
  "created_at": 1702296000
}'
```

#### Scénario 1 : Recherche simple

```bash
# Utilisateur cherche "laptop"
FT.SEARCH idx:products "laptop"
  SORTBY rating DESC
  LIMIT 0 20
  RETURN 4 name price rating stock
```

#### Scénario 2 : Recherche avec filtres

```bash
# Utilisateur cherche "laptop" entre 800€ et 1500€, marque Dell ou HP
FT.SEARCH idx:products
  "laptop @price:[800 1500] @brand:{Dell|HP}"
  SORTBY price ASC
  LIMIT 0 20
  RETURN 4 name price brand rating
```

#### Scénario 3 : Recherche avancée avec facettes

```bash
# Recherche avec statistiques par catégorie (voir section 3.4 pour FT.AGGREGATE)
# Pour l'instant, recherche simple avec filtres multiples

# Laptops Dell, en stock, bien notés (>4.0)
FT.SEARCH idx:products
  "@category:{laptop} @brand:{Dell} @stock:[1 +inf] @rating:[4.0 +inf]"
  SORTBY rating DESC
  LIMIT 0 10
```

#### Scénario 4 : Autocomplete

```bash
# Utilisateur tape "lap" → Suggestions
FT.SEARCH idx:products "lap*"
  LIMIT 0 5
  RETURN 1 name

# Résultat :
# 1) (integer) 2
# 2) "product:101"
# 3) 1) "name"
#    2) "Laptop Dell XPS 13"
# 4) "product:102"
# 5) 1) "name"
#    2) "Laptop HP Pavilion 15"
```

---

### 2️⃣ Base de connaissances / Documentation

**Contexte** : Recherche dans des articles de documentation

```bash
# Créer l'index articles
FT.CREATE idx:articles
  ON JSON
  PREFIX 1 article:
  SCHEMA
    $.title AS title TEXT WEIGHT 10.0 SORTABLE
    $.content AS content TEXT WEIGHT 2.0
    $.tags AS tags TAG SEPARATOR ","
    $.author AS author TAG
    $.category AS category TAG
    $.published_at AS published_at NUMERIC SORTABLE
    $.views AS views NUMERIC SORTABLE

# Insérer des articles
JSON.SET article:1 $ '{
  "title": "Introduction à Redis Stack",
  "content": "Redis Stack étend Redis Core avec des modules puissants comme RediSearch, RedisJSON...",
  "tags": "redis,tutorial,beginner",
  "author": "alice",
  "category": "tutorial",
  "published_at": 1702123200,
  "views": 1250
}'

JSON.SET article:2 $ '{
  "title": "RediSearch : Indexation et recherche",
  "content": "RediSearch permet de créer des index secondaires pour effectuer des recherches full-text...",
  "tags": "redis,redisearch,advanced",
  "author": "bob",
  "category": "guide",
  "published_at": 1702209600,
  "views": 890
}'
```

#### Recherche d'articles

```bash
# Recherche par contenu
FT.SEARCH idx:articles "indexation recherche"
  HIGHLIGHT FIELDS 2 title content
  TAGS "<mark>" "</mark>"
  LIMIT 0 10

# Recherche par tags
FT.SEARCH idx:articles "@tags:{redis}"
  SORTBY views DESC
  LIMIT 0 10

# Recherche d'articles récents d'un auteur
FT.SEARCH idx:articles "@author:{alice}"
  SORTBY published_at DESC
  LIMIT 0 10

# Recherche full-text avec filtres
FT.SEARCH idx:articles
  "redis @category:{tutorial} @published_at:[1702000000 +inf]"
  SORTBY published_at DESC
```

---

### 3️⃣ Recherche d'utilisateurs (réseau social)

**Contexte** : Trouver des utilisateurs par nom, bio, skills

```bash
# Créer l'index utilisateurs
FT.CREATE idx:users
  ON JSON
  PREFIX 1 user:
  SCHEMA
    $.username AS username TEXT SORTABLE
    $.full_name AS full_name TEXT WEIGHT 3.0 SORTABLE
    $.bio AS bio TEXT
    $.skills AS skills TAG SEPARATOR ","
    $.location AS location TAG
    $.followers_count AS followers_count NUMERIC SORTABLE
    $.verified AS verified TAG

# Insérer des utilisateurs
JSON.SET user:1001 $ '{
  "username": "alice_dev",
  "full_name": "Alice Dubois",
  "bio": "Full-stack developer | Redis enthusiast | Paris",
  "skills": "python,redis,docker,kubernetes",
  "location": "Paris",
  "followers_count": 1250,
  "verified": "true"
}'

JSON.SET user:1002 $ '{
  "username": "bob_data",
  "full_name": "Bob Martin",
  "bio": "Data engineer specializing in real-time analytics",
  "skills": "kafka,spark,redis,python",
  "location": "Lyon",
  "followers_count": 890,
  "verified": "false"
}'
```

#### Recherche d'utilisateurs

```bash
# Par nom
FT.SEARCH idx:users "@full_name:alice"

# Par compétence
FT.SEARCH idx:users "@skills:{redis}"

# Par localisation
FT.SEARCH idx:users "@location:{Paris}"

# Utilisateurs vérifiés avec Redis dans leurs skills
FT.SEARCH idx:users "@skills:{redis} @verified:{true}"
  SORTBY followers_count DESC
  LIMIT 0 10

# Recherche dans la bio
FT.SEARCH idx:users "@bio:(developer | engineer)"
```

---

### 4️⃣ Recherche de logs et événements

**Contexte** : Analyser des logs applicatifs

```bash
# Créer l'index logs
FT.CREATE idx:logs
  ON HASH
  PREFIX 1 log:
  SCHEMA
    message TEXT
    level TAG
    service TAG
    user_id NUMERIC
    timestamp NUMERIC SORTABLE
    ip TAG

# Insérer des logs
HSET log:1 message "User login successful" level "INFO" service "auth" user_id 1001 timestamp 1702123200 ip "192.168.1.100"
HSET log:2 message "Database connection failed" level "ERROR" service "api" user_id 0 timestamp 1702123205 ip "192.168.1.101"
HSET log:3 message "User login failed: invalid password" level "WARN" service "auth" user_id 1002 timestamp 1702123210 ip "192.168.1.102"
```

#### Recherche de logs

```bash
# Tous les logs ERROR
FT.SEARCH idx:logs "@level:{ERROR}"

# Logs du service auth dans la dernière heure
FT.SEARCH idx:logs
  "@service:{auth} @timestamp:[1702120000 +inf]"
  SORTBY timestamp DESC

# Logs contenant "failed" ou "error"
FT.SEARCH idx:logs "failed | error"

# Logs d'un utilisateur spécifique
FT.SEARCH idx:logs "@user_id:[1001 1001]"

# Logs WARN ou ERROR des dernières 24h
FT.SEARCH idx:logs
  "@level:{WARN|ERROR} @timestamp:[1702036800 +inf]"
  SORTBY timestamp DESC
  LIMIT 0 100
```

---

### 5️⃣ Recherche géospatiale (magasins)

**Contexte** : Trouver des magasins à proximité

```bash
# Créer l'index magasins
FT.CREATE idx:stores
  ON HASH
  PREFIX 1 store:
  SCHEMA
    name TEXT SORTABLE
    address TEXT
    city TAG
    location GEO
    rating NUMERIC SORTABLE
    open TAG

# Insérer des magasins (format : "longitude,latitude")
HSET store:1 name "Store Paris Centre" address "123 Rue de Rivoli" city "Paris" location "2.3522,48.8566" rating 4.5 open "true"
HSET store:2 name "Store Paris Nord" address "456 Boulevard de la Villette" city "Paris" location "2.3686,48.8811" rating 4.2 open "true"
HSET store:3 name "Store Lyon" address "789 Rue de la République" city "Lyon" location "4.8357,45.7640" rating 4.7 open "false"
```

#### Recherche géospatiale

```bash
# Magasins dans un rayon de 5km autour de la Tour Eiffel (2.2945, 48.8584)
FT.SEARCH idx:stores "@location:[2.2945 48.8584 5 km]"

# Magasins ouverts dans un rayon de 10km, triés par rating
FT.SEARCH idx:stores
  "@location:[2.2945 48.8584 10 km] @open:{true}"
  SORTBY rating DESC

# Magasins d'une ville spécifique
FT.SEARCH idx:stores "@city:{Paris}"

# Combiner recherche textuelle et géo
FT.SEARCH idx:stores
  "centre @location:[2.3522 48.8566 3 km]"
  SORTBY rating DESC
```

---

## Performance et optimisation

### Benchmark : SCAN vs RediSearch

**Scénario** : Recherche dans 100 000 produits

```bash
# SCAN (Redis Core)
# Temps moyen : 1500-3000ms (dépend du nombre de clés)

# RediSearch (indexé)
# Temps moyen : 2-8ms

# Gain : 200-1000x plus rapide
```

---

### Impact mémoire des index

```bash
# Exemple : 100 000 produits

# Données seules (Hashes) : 200 MB

# Avec index RediSearch :
# - Index TEXT (name, description) : +30 MB
# - Index NUMERIC (price, stock) : +5 MB
# - Index TAG (category, brand) : +8 MB
# Total : 200 MB + 43 MB = 243 MB

# Overhead : +21%
```

**Conseil** : N'indexez que les champs nécessaires pour réduire l'overhead.

---

### Options SORTABLE

```bash
# Sans SORTABLE
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA name TEXT

# Avec SORTABLE (+30% mémoire pour ce champ)
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA name TEXT SORTABLE
```

**Trade-off** : `SORTABLE` consomme plus de mémoire mais permet le tri rapide.

---

## Maintenance des index

### Lister les index

```bash
FT._LIST

# Résultat :
# 1) "idx:products"
# 2) "idx:users"
# 3) "idx:articles"
```

---

### Informations sur un index

```bash
FT.INFO idx:products

# Résultat :
# 1) "index_name"
# 2) "idx:products"
# 3) "index_options"
# 4) (empty array)
# 5) "index_definition"
# 6) 1) "key_type"
#    2) "HASH"
#    3) "prefixes"
#    4) 1) "product:"
# 7) "attributes"
# 8) 1) 1) "identifier"
#       2) "name"
#       3) "attribute"
#       4) "name"
#       5) "type"
#       6) "TEXT"
#       ...
# 9) "num_docs"
# 10) "3"  # Nombre de documents indexés
# 11) "max_doc_id"
# 12) "3"
# 13) "num_terms"
# 14) "15"  # Nombre de termes dans l'index inversé
# 15) "num_records"
# 16) "30"
# ...
```

---

### Supprimer un index

```bash
# Supprimer l'index (les données restent)
FT.DROPINDEX idx:products

# Supprimer l'index ET les données
FT.DROPINDEX idx:products DD
```

**Attention** : `DD` (Drop Documents) supprime les clés Redis !

---

### Reconstruire un index

```bash
# Supprimer et recréer
FT.DROPINDEX idx:products
FT.CREATE idx:products ON HASH PREFIX 1 product: SCHEMA...

# RediSearch réindexe automatiquement les clés existantes
```

---

## Limitations

### 1. Wildcards limités

```bash
# ✅ Préfixe supporté
FT.SEARCH idx:products "lap*"

# ❌ Suffixe non supporté directement
# FT.SEARCH idx:products "*top"

# ❌ Infix non supporté
# FT.SEARCH idx:products "*apt*"
```

**Workaround** : Utiliser des trigrams ou stocker les données inversées.

---

### 2. Multi-key operations limitées

RediSearch indexe clé par clé. Pas de "joins" entre index.

```bash
# ❌ Impossible de joindre users et orders
# SELECT users.name, orders.total FROM users JOIN orders...

# ✅ Dénormaliser les données
JSON.SET order:12345 $ '{
  "order_id": 12345,
  "user_name": "Alice",  # Dénormalisé
  "total": 1359.97
}'
```

---

### 3. Pas de mises à jour partielles d'index

Si vous modifiez un document, l'index entier du document est recalculé.

```bash
# Modifier le prix
HSET product:101 price 999.99

# RediSearch réindexe automatiquement product:101 (tous les champs)
```

---

## Bonnes pratiques

### ✅ 1. Nommer les index de manière cohérente

```bash
# Convention recommandée : idx:{entity_type}
FT.CREATE idx:products ...
FT.CREATE idx:users ...
FT.CREATE idx:orders ...
```

---

### ✅ 2. Utiliser les préfixes pour isoler les données

```bash
# Un index par type d'entité
FT.CREATE idx:products ON HASH PREFIX 1 product: ...
FT.CREATE idx:users ON HASH PREFIX 1 user: ...

# Éviter de mélanger dans un même index
```

---

### ✅ 3. N'indexer que les champs nécessaires

```bash
# ❌ Mauvais : Indexer tous les champs
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA name TEXT description TEXT price NUMERIC ... internal_id NUMERIC

# ✅ Bon : Indexer uniquement les champs recherchés
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA name TEXT price NUMERIC category TAG
```

---

### ✅ 4. Utiliser SORTABLE seulement si nécessaire

```bash
# ✅ Si vous triez souvent par price
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA price NUMERIC SORTABLE

# ❌ Si vous ne triez jamais par description
# SCHEMA description TEXT SORTABLE  # Gaspillage de mémoire
```

---

### ✅ 5. Limiter les résultats avec LIMIT

```bash
# ✅ Bon : Pagination
FT.SEARCH idx:products "laptop" LIMIT 0 20

# ❌ Mauvais : Tous les résultats
FT.SEARCH idx:products "laptop"  # Peut retourner 10 000 résultats
```

---

### ✅ 6. Utiliser TAG pour les filtres exacts

```bash
# ✅ Bon : Utiliser TAG pour category
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA category TAG

FT.SEARCH idx:products "@category:{laptop}"

# ❌ Mauvais : Utiliser TEXT pour category (moins efficace)
FT.CREATE idx:products ON HASH PREFIX 1 product:
  SCHEMA category TEXT
```

---

## Troubleshooting

### Erreur : "Unknown index name"

```bash
FT.SEARCH idx:products "laptop"
# (error) Unknown index name

# Solution : Créer l'index
FT.CREATE idx:products ON HASH PREFIX 1 product: SCHEMA...
```

---

### Erreur : "Syntax error"

```bash
FT.SEARCH idx:products "@price:[100 500"
# (error) Syntax error

# Solution : Fermer le bracket
FT.SEARCH idx:products "@price:[100 500]"
```

---

### Aucun résultat alors que les données existent

```bash
# Vérifier que les clés correspondent au préfixe
FT.INFO idx:products
# Regarder "prefixes": "product:"

# Si vos clés sont "prod:101", l'index ne les trouve pas
# Solution : Recréer l'index avec le bon préfixe
FT.DROPINDEX idx:products
FT.CREATE idx:products ON HASH PREFIX 1 prod: SCHEMA...
```

---

### Performance dégradée

```bash
# Vérifier le nombre de documents indexés
FT.INFO idx:products
# "num_docs": "1000000"  # Trop de documents ?

# Solution 1 : Partitionner l'index (plusieurs préfixes)
# Solution 2 : Limiter la taille des champs TEXT
# Solution 3 : Utiliser Redis Cluster pour distribuer la charge
```

---

## Comparaison : RediSearch vs Elasticsearch

| Critère | RediSearch | Elasticsearch |
|---------|------------|---------------|
| **Latence** | < 5ms (mémoire) | 20-200ms (disque + cache) |
| **Throughput** | 100K+ requêtes/sec | 5-20K requêtes/sec |
| **Indexation** | Temps réel (instantané) | Near real-time (~1s) |
| **Requêtes complexes** | Limité (pas de joins) | Avancé (aggregations, joins) |
| **Infrastructure** | Simple (Redis) | Complexe (cluster ES) |
| **Coût** | Faible (mémoire) | Élevé (CPU + disque + mémoire) |
| **Cas d'usage** | Cache + recherche rapide | Recherche analytique approfondie |

**Conclusion** : RediSearch pour **vitesse et simplicité**, Elasticsearch pour **requêtes complexes** et analyses approfondies.

---

## Résumé

**RediSearch permet de** :
- ✅ Créer des index secondaires sur Hashes ou JSON
- ✅ Effectuer des recherches full-text ultra-rapides (< 5ms)
- ✅ Filtrer, trier, paginer les résultats
- ✅ Combiner recherche textuelle, numérique, géospatiale
- ✅ Remplacer Elasticsearch pour des cas d'usage simples

**Types de champs** :
- `TEXT` : Recherche full-text
- `TAG` : Recherche exacte (catégories, tags)
- `NUMERIC` : Recherche par range
- `GEO` : Recherche géospatiale

**Cas d'usage idéaux** :
- 🛒 E-commerce (recherche produits)
- 📚 Documentation (recherche articles)
- 👥 Réseaux sociaux (recherche utilisateurs)
- 📊 Logs et événements (analyse temps réel)
- 📍 Géolocalisation (magasins, points d'intérêt)

**Limitations** :
- Pas de joins entre index
- Wildcards limités (préfixe uniquement)
- +20-40% mémoire pour les index

---

**Prêt pour les agrégations ?** Passons à la section suivante : [3.4 RediSearch - Agrégations et requêtes complexes](./04-redisearch-agregations-requetes.md)

⏭️ [RediSearch : Agrégations et requêtes complexes](/03-structures-donnees-etendues/04-redisearch-agregations-requetes.md)
