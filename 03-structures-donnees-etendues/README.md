🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 3 : Structures de données étendues (Redis Stack)

## Vue d'ensemble du module

Après avoir maîtrisé les structures de données natives de Redis Core (Strings, Lists, Hashes, Sets, Sorted Sets), il est temps d'explorer **Redis Stack** : une extension puissante qui transforme Redis d'un simple cache en une **base de données multi-modèle** capable de gérer des cas d'usage modernes et complexes.

Redis Stack n'est pas une alternative à Redis Core, mais une **surcouche de modules** qui enrichit considérablement les capacités de Redis tout en conservant ses performances exceptionnelles et son modèle de données en mémoire.

---

## Pourquoi Redis Stack ?

### Les limites de Redis Core pour les cas d'usage modernes

Redis Core excelle dans :
- ✅ Le caching simple (clé-valeur)
- ✅ Les compteurs et incréments atomiques
- ✅ Les leaderboards avec Sorted Sets
- ✅ Les files d'attente basiques avec Lists

Mais il montre ses limites face à :
- ❌ La recherche full-text sur des documents
- ❌ Le requêtage complexe sur des objets JSON
- ❌ La recherche sémantique et vectorielle (IA/ML)
- ❌ Les séries temporelles avec agrégations (IoT, monitoring)
- ❌ Les filtres probabilistes pour de gros volumes

### Redis Stack comble ces lacunes

Redis Stack ajoute des **modules natifs** qui étendent Redis sans compromettre ses performances :

| Module | Cas d'usage principal |
|--------|----------------------|
| **RedisJSON** | Documents JSON natifs avec requêtage JSONPath |
| **RediSearch** | Indexation, full-text search, agrégations, vector search |
| **RedisTimeSeries** | Données temporelles avec agrégations (IoT, métriques) |
| **RedisBloom** | Filtres probabilistes (Bloom, Cuckoo, Count-Min Sketch) |
| **RedisGraph** (déprécié) | Bases de données orientées graphes |

---

## Architecture Redis Stack

```
┌─────────────────────────────────────────────────────────┐
│                    Applications                         │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                   Redis Stack                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│  │RedisJSON │  │RediSearch│  │TimeSeries│  │  Bloom  │  │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └────┬────┘  │
│        │             │             │            │       │
│  ┌─────▼─────────────▼─────────────▼────────────▼─────┐ │
│  │              Redis Core Engine                     │ │
│  │   (Single-thread, I/O Multiplexing, In-Memory)     │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Points clés :**
- Redis Stack = Redis Core + Modules
- Les modules sont **natifs** et compilés en C
- Performance quasi-identique à Redis Core
- Utilise le même moteur de persistance (RDB/AOF)
- Compatible avec tous les outils Redis (redis-cli, Redis Insight)

---

## Vue d'ensemble des modules

### 1️⃣ RedisJSON : Documents JSON natifs

**Problème résolu :** Avec Redis Core, pour stocker un objet JSON, vous devez :
- Le sérialiser en String → perte de structure
- Récupérer tout l'objet pour modifier un champ → inefficace
- Implémenter la logique de mise à jour côté application

**Solution RedisJSON :**

```bash
# Stocker un document JSON directement
JSON.SET user:1001 $ '{"name":"Alice","age":30,"city":"Paris","skills":["Python","Redis"]}'

# Récupérer un chemin spécifique (JSONPath)
JSON.GET user:1001 $.name
# Résultat: ["Alice"]

# Modifier uniquement l'âge (opération atomique)
JSON.NUMINCRBY user:1001 $.age 1

# Ajouter une compétence au tableau
JSON.ARRAPPEND user:1001 $.skills '"Docker"'

# Récupérer la taille du tableau
JSON.ARRLEN user:1001 $.skills
# Résultat: 3
```

**Cas d'usage modernes :**
- 🛒 **E-commerce** : Panier avec articles complexes
- 👤 **Profils utilisateurs** : Données structurées évolutives
- ⚙️ **Configuration dynamique** : Settings d'application
- 📊 **API REST** : Cache de réponses JSON

---

### 2️⃣ RediSearch : Indexation et recherche avancée

**Problème résolu :** Redis Core ne permet pas de :
- Chercher par texte : "tous les produits contenant 'laptop'"
- Filtrer : "âge > 25 AND ville = 'Paris'"
- Trier : "par prix croissant"

**Solution RediSearch :**

```bash
# Créer un index sur des documents JSON
FT.CREATE idx:products
  ON JSON
  PREFIX 1 product:
  SCHEMA
    $.name AS name TEXT SORTABLE
    $.price AS price NUMERIC SORTABLE
    $.category AS category TAG

# Insérer des produits
JSON.SET product:1 $ '{"name":"Laptop Dell XPS","price":1299,"category":"electronics"}'
JSON.SET product:2 $ '{"name":"Laptop HP Pavilion","price":899,"category":"electronics"}'

# Recherche full-text
FT.SEARCH idx:products "laptop" SORTBY price ASC
# Retourne les produits triés par prix

# Recherche avec filtres
FT.SEARCH idx:products "@category:{electronics} @price:[500 1000]"
# Produits électroniques entre 500 et 1000€

# Agrégations
FT.AGGREGATE idx:products "*"
  GROUPBY 1 @category
  REDUCE AVG 1 @price AS avg_price
# Prix moyen par catégorie
```

**Cas d'usage modernes :**
- 🔍 **Moteurs de recherche** : E-commerce, marketplace
- 📱 **Autocomplete** : Suggestions de recherche en temps réel
- 📊 **Analytics** : Agrégations complexes sur données en mémoire
- 🧠 **Recommendation engines** : Recherche par similarité

---

### 3️⃣ RediSearch Vector Search : IA et RAG

**Problème résolu :** Pour l'IA moderne (LLMs, RAG), vous devez :
- Stocker des embeddings vectoriels (ex: 1536 dimensions)
- Effectuer des recherches par similarité sémantique (KNN/ANN)
- Combiner recherche vectorielle + filtres métadonnées

**Solution Vector Search :**

```bash
# Créer un index avec champ vectoriel
FT.CREATE idx:documents
  ON HASH
  PREFIX 1 doc:
  SCHEMA
    content TEXT
    embedding VECTOR FLAT 6 TYPE FLOAT32 DIM 384 DISTANCE_METRIC COSINE

# Stocker un document avec son embedding
HSET doc:1
  content "Redis is an in-memory database"
  embedding "\x00\x00\x80?\x00\x00\x00@..."  # 384 floats

# Recherche vectorielle (KNN)
FT.SEARCH idx:documents
  "*=>[KNN 5 @embedding $query_vector]"
  PARAMS 2 query_vector "\x00\x00\x80?..."  # embedding de la requête
  RETURN 2 content __vector_score
```

**Cas d'usage modernes :**
- 🤖 **RAG (Retrieval Augmented Generation)** : Contexte pour LLMs
- 🎯 **Recommendation** : Produits similaires
- 🖼️ **Recherche d'images** : Similarité visuelle
- 📚 **Semantic search** : "Trouver des documents similaires"

---

### 4️⃣ RedisTimeSeries : Données temporelles

**Problème résolu :** Avec Redis Core, pour des séries temporelles :
- Vous devez gérer manuellement les timestamps
- Pas d'agrégations natives (moyenne sur 1h, downsampling)
- Pas de compaction automatique

**Solution RedisTimeSeries :**

```bash
# Créer une série temporelle
TS.CREATE temperature:sensor1
  RETENTION 86400000  # 24h en millisecondes
  LABELS sensor_id 1 location "datacenter-a"

# Ajouter des mesures
TS.ADD temperature:sensor1 * 22.5  # * = timestamp actuel
TS.ADD temperature:sensor1 * 23.1
TS.ADD temperature:sensor1 * 22.8

# Créer une règle d'agrégation (moyenne sur 5 min)
TS.CREATERULE temperature:sensor1 temperature:sensor1:avg5min
  AGGREGATION avg 300000  # 5 min en ms

# Requête avec agrégation
TS.RANGE temperature:sensor1 1638360000000 1638363600000 AGGREGATION avg 60000
# Moyenne par minute sur 1 heure

# Requêtes multi-séries
TS.MRANGE - + FILTER location=datacenter-a
# Toutes les séries du datacenter-a
```

**Cas d'usage modernes :**
- 🌡️ **IoT** : Capteurs, télémétrie
- 📊 **Monitoring** : Métriques système (CPU, RAM)
- 💹 **Finance** : Prix des actifs, trading
- 🏭 **Industrie 4.0** : Données machines en temps réel

---

### 5️⃣ RedisBloom : Filtres probabilistes

**Problème résolu :** Pour vérifier l'existence dans de gros volumes :
- Redis Core nécessite de stocker toutes les clés (mémoire)
- Recherche dans un Set peut être lente pour des millions d'entrées

**Solution RedisBloom :**

```bash
# Bloom Filter : "Est-ce que X existe ?" (faux positifs possibles)
BF.ADD emails "alice@example.com"
BF.EXISTS emails "bob@example.com"  # 0 (n'existe pas, 100% sûr)
BF.EXISTS emails "alice@example.com"  # 1 (existe, ou faux positif ~0.01%)

# Cuckoo Filter : Permet la suppression
CF.ADD usernames "alice123"
CF.DEL usernames "alice123"  # OK, suppression possible

# Count-Min Sketch : Comptage d'événements fréquents
CMS.INCRBY page_views index.html 1
CMS.QUERY page_views index.html  # Estimation du nombre de vues

# Top-K : Les K éléments les plus fréquents
TOPK.ADD trending_keywords "redis" "kubernetes" "docker" "redis"
TOPK.LIST trending_keywords  # ["redis", "kubernetes", "docker"]
```

**Cas d'usage modernes :**
- 🚫 **Anti-spam** : "Cet email est-il dans la blacklist ?"
- 🔐 **Leak detection** : "Ce mot de passe a-t-il fuité ?"
- 📈 **Analytics** : Top produits, trending topics
- 🎯 **Deduplication** : URLs crawlées, IDs traités

---

## Comparaison : Redis Core vs Redis Stack

| Fonctionnalité | Redis Core | Redis Stack |
|----------------|------------|-------------|
| **Stockage JSON** | String sérialisé | RedisJSON (natif) |
| **Recherche full-text** | KEYS/SCAN (lent) | RediSearch (indexé) |
| **Filtrage complexe** | Logic applicative | RediSearch (requêtes) |
| **Vector search** | Impossible | RediSearch (KNN/ANN) |
| **Time-series** | Sorted Sets (manuel) | RedisTimeSeries (natif) |
| **Bloom filters** | Implémentation custom | RedisBloom (optimisé) |
| **Agrégations** | Logic applicative | RediSearch/TimeSeries |

---

## Quand utiliser Redis Stack ?

### ✅ Utilisez Redis Stack si :

- Vous manipulez des **documents JSON complexes**
- Vous avez besoin de **recherche full-text** performante
- Vous construisez un **moteur de recommandation** (vector search)
- Vous gérez des **séries temporelles** (IoT, monitoring, finance)
- Vous avez des **gros volumes** nécessitant des filtres probabilistes
- Vous voulez des **agrégations en temps réel**

### ❌ Restez sur Redis Core si :

- Vous faites uniquement du **caching simple** (clé-valeur)
- Vous utilisez Redis comme **message broker** (Pub/Sub, Streams)
- Vous gérez des **compteurs** ou **leaderboards simples**
- Vous privilégiez la **compatibilité maximale** (tous les forks Redis)
- Vous êtes limité en **mémoire** (Redis Stack consomme plus)

---

## Installation et vérification

### Vérifier si Redis Stack est disponible

```bash
# Vérifier les modules chargés
redis-cli MODULE LIST

# Devrait retourner (si Redis Stack installé) :
# 1) 1) "name"
#    2) "search"
# 2) 1) "name"
#    2) "ReJSON"
# 3) 1) "name"
#    2) "timeseries"
# 4) 1) "name"
#    2) "bf"
```

### Avec Docker (recommandé)

```bash
# Redis Stack complet (tous les modules)
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest

# Redis Stack Server only (sans Redis Insight)
docker run -d --name redis-stack-server -p 6379:6379 redis/redis-stack-server:latest
```

### Tester RedisJSON

```bash
redis-cli JSON.SET test $ '{"hello":"world"}'
redis-cli JSON.GET test
# Résultat: "{\"hello\":\"world\"}"
```

---

## Performance et considérations

### Impact mémoire

Redis Stack consomme **plus de mémoire** que Redis Core :

| Module | Overhead mémoire |
|--------|------------------|
| **RedisJSON** | ~20-30% vs String sérialisé (mais meilleure compression) |
| **RediSearch** | +10-50% pour les index (dépend des champs indexés) |
| **RedisTimeSeries** | Très efficace grâce à la compaction |
| **RedisBloom** | Très faible (c'est le but des filtres probabilistes) |

### Performance

- **RedisJSON** : ~90-95% des perfs de Redis Core pour GET/SET
- **RediSearch** : Recherches en **O(log N)** vs O(N) pour SCAN
- **Vector Search** : 1000-10000 requêtes/sec (dépend de la dimensionnalité)
- **RedisTimeSeries** : 100K+ écritures/sec par série

### Compatibilité

⚠️ **Attention** : Redis Stack n'est pas disponible sur tous les forks :

| Fork | Redis Stack |
|------|-------------|
| **Redis OSS** | ✅ Complet |
| **Redis Enterprise** | ✅ Complet + features avancées |
| **Valkey** | ❌ Non supporté (fork sans modules) |
| **KeyDB** | ⚠️ Partiel (certains modules compatibles) |
| **Dragonfly** | ⚠️ En cours de développement |

---

## Structure du module

Ce module est organisé en **7 sections progressives** :

1. **Introduction à Redis Stack** : Vision globale et adoption
2. **RedisJSON** : Documents JSON natifs et JSONPath
3. **RediSearch (Partie 1)** : Indexation et full-text search
4. **RediSearch (Partie 2)** : Agrégations et requêtes complexes
5. **RediSearch (Partie 3)** : Vector Search pour l'IA/RAG
6. **RedisTimeSeries** : Gestion de données temporelles
7. **RedisBloom** : Filtres probabilistes (Bloom, Cuckoo)

---

## Prérequis

Avant de commencer ce module, vous devez maîtriser :

✅ **Les structures natives de Redis Core** (Module 2) :
- Strings, Lists, Hashes, Sets, Sorted Sets
- Commandes CRUD de base
- Concept de TTL et expiration

✅ **Les concepts fondamentaux** :
- Modèle clé-valeur
- Sérialisation JSON (côté application)
- Requêtage de base

✅ **Environnement de travail** :
- Redis Stack installé (Docker recommandé)
- redis-cli ou Redis Insight configuré
- Familiarité avec JSONPath (bonus)

---

## À retenir

🔑 **Messages clés de ce module :**

1. **Redis Stack ≠ Redis remplacé** : C'est une extension, pas une alternative
2. **Multi-modèle** : Redis devient base documentaire, moteur de recherche, TSDB
3. **Performance préservée** : Modules natifs en C, optimisations poussées
4. **Cas d'usage modernes** : IA/RAG, IoT, e-commerce, analytics temps réel
5. **Trade-offs** : Plus de features = plus de mémoire + compatibilité réduite

---

## Ressources additionnelles

📚 **Documentation officielle :**
- [Redis Stack Documentation](https://redis.io/docs/stack/)
- [RedisJSON Commands](https://redis.io/docs/stack/json/)
- [RediSearch Reference](https://redis.io/docs/stack/search/)
- [RedisTimeSeries Guide](https://redis.io/docs/stack/timeseries/)

🎥 **Tutoriels vidéo :**
- [Redis Stack Quickstart](https://redis.io/docs/stack/get-started/)
- [Vector Search Tutorial](https://redis.io/docs/stack/search/reference/vectors/)

💻 **Exemples de code :**
- [Redis Stack Examples (GitHub)](https://github.com/redis-stack/redis-stack-examples)

---

**Prêt à explorer Redis Stack ?** Passons à la première section : [Introduction à Redis Stack - Pourquoi l'adopter ?](./01-introduction-redis-stack.md)

⏭️ [Introduction à Redis Stack : Pourquoi l'adopter ?](/03-structures-donnees-etendues/01-introduction-redis-stack.md)
