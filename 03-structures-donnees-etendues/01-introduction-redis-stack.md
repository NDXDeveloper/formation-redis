🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1 Introduction à Redis Stack : Pourquoi l'adopter ?

## Introduction

Vous maîtrisez Redis Core : vous savez utiliser les Strings pour du caching, les Sorted Sets pour des leaderboards, les Hashes pour représenter des objets, et les Lists pour des files d'attente. **Pourquoi alors adopter Redis Stack ?**

La réponse est simple : **Redis Core a été conçu pour la vitesse et la simplicité, pas pour la complexité fonctionnelle**. Lorsque vos besoins dépassent le simple stockage clé-valeur, vous vous retrouvez à implémenter côté application ce que Redis Stack fournit nativement.

Redis Stack transforme Redis d'un **cache ultra-rapide** en une **base de données multi-modèle** capable de rivaliser avec des solutions spécialisées (Elasticsearch, MongoDB, InfluxDB) tout en conservant les performances légendaires de Redis.

---

## Le problème : Les limites de Redis Core

### Scénario 1 : Recherche dans des documents JSON

**Avec Redis Core** : Vous stockez des profils utilisateurs

```bash
# Stocker un profil utilisateur en JSON sérialisé
SET user:1001 '{"id":1001,"name":"Alice Dubois","age":28,"city":"Paris","skills":["Python","Redis","Docker"],"premium":true}'

# 🔴 Problème : Comment trouver tous les utilisateurs de Paris ?
# Solution : SCAN + désérialisation côté application
SCAN 0 MATCH user:* COUNT 100
# Puis pour chaque clé :
# 1. GET user:xxxx
# 2. Désérialiser le JSON
# 3. Vérifier si city == "Paris"
# 4. Filtrer côté application
```

**Problèmes rencontrés :**
- ❌ `SCAN` bloque Redis (opération O(N))
- ❌ Transfert de **toutes les données** sur le réseau
- ❌ Désérialisation CPU-intensive côté client
- ❌ Pas de tri, pas de pagination efficace
- ❌ Temps de réponse proportionnel au nombre d'utilisateurs

**Avec Redis Stack (RediSearch)** :

```bash
# 1. Créer un index sur les documents JSON
FT.CREATE idx:users
  ON JSON
  PREFIX 1 user:
  SCHEMA
    $.name AS name TEXT SORTABLE
    $.age AS age NUMERIC SORTABLE
    $.city AS city TAG SORTABLE
    $.premium AS premium TAG

# 2. Stocker les documents (RedisJSON)
JSON.SET user:1001 $ '{"id":1001,"name":"Alice Dubois","age":28,"city":"Paris","skills":["Python","Redis","Docker"],"premium":true}'
JSON.SET user:1002 $ '{"id":1002,"name":"Bob Martin","age":35,"city":"Lyon","skills":["Java","Kubernetes"],"premium":false}'
JSON.SET user:1003 $ '{"id":1003,"name":"Claire Leroy","age":24,"city":"Paris","skills":["JavaScript","React"],"premium":true}'

# 3. Rechercher tous les utilisateurs de Paris, triés par âge
FT.SEARCH idx:users "@city:{Paris}" SORTBY age ASC RETURN 3 $.name $.age $.premium

# Résultat instantané :
# 1) "2"  # Nombre de résultats
# 2) "user:1003"
# 3) 1) "$.name"
#    2) "\"Claire Leroy\""
#    3) "$.age"
#    4) "24"
# 4) "user:1001"
# 5) 1) "$.name"
#    2) "\"Alice Dubois\""
#    3) "$.age"
#    4) "28"
```

**Avantages :**
- ✅ Recherche **indexée** en O(log N)
- ✅ Filtrage côté Redis (pas de transfert inutile)
- ✅ Tri et pagination natifs
- ✅ Temps de réponse **constant**, indépendant du volume

**Gain de performance mesuré** : **50-100x plus rapide** sur 10K+ documents.

---

### Scénario 2 : Modification partielle d'un objet JSON

**Avec Redis Core** :

```bash
# Stocker un panier e-commerce
SET cart:5001 '{"user_id":1001,"items":[{"id":101,"name":"Laptop","qty":1,"price":1299.99},{"id":202,"name":"Mouse","qty":2,"price":29.99}],"total":1359.97}'

# 🔴 Problème : L'utilisateur ajoute un article
# Solution : GET + Désérialisation + Modification + Sérialisation + SET
# Côté application (pseudo-code) :
cart = JSON.parse(redis.get("cart:5001"))
cart.items.push({id: 303, name: "Keyboard", qty: 1, price: 79.99})
cart.total += 79.99
redis.set("cart:5001", JSON.stringify(cart))
```

**Problèmes :**
- ❌ **Race condition** : 2 utilisateurs modifient simultanément
- ❌ Transfert du JSON complet (aller-retour réseau)
- ❌ Logic applicative complexe
- ❌ Besoin de transactions (WATCH/MULTI/EXEC) pour l'atomicité

**Avec Redis Stack (RedisJSON)** :

```bash
# Ajouter un article atomiquement
JSON.ARRAPPEND cart:5001 $.items '{"id":303,"name":"Keyboard","qty":1,"price":79.99}'

# Incrémenter le total atomiquement
JSON.NUMINCRBY cart:5001 $.total 79.99

# Résultat : 2 commandes atomiques, pas de transfert du JSON complet
```

**Avantages :**
- ✅ **Opérations atomiques** sur des sous-parties du JSON
- ✅ Pas de transfert réseau du document complet
- ✅ Pas de race condition (opérations atomiques natives)
- ✅ Syntaxe JSONPath standard

**Gain mesuré** : Réduction de **70-90% de la bande passante** réseau.

---

### Scénario 3 : Recherche sémantique (IA/RAG)

**Contexte** : Vous construisez un chatbot avec GPT-4 et voulez implémenter du RAG (Retrieval Augmented Generation) pour fournir du contexte depuis votre documentation.

**Approche traditionnelle (sans Redis Stack)** :

```
1. Stocker les embeddings dans PostgreSQL avec pgvector
2. Pour chaque requête utilisateur :
   - Générer l'embedding de la question (OpenAI API)
   - Requête SQL pour KNN : SELECT * FROM docs ORDER BY embedding <=> $1 LIMIT 5
   - Latence : 50-200ms (I/O disque)
3. Concaténer les résultats et envoyer à GPT-4
```

**Avec Redis Stack (Vector Search)** :

```bash
# 1. Créer un index avec champ vectoriel (HNSW pour rapidité)
FT.CREATE idx:docs
  ON HASH
  PREFIX 1 doc:
  SCHEMA
    title TEXT
    content TEXT
    category TAG
    embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 1536 DISTANCE_METRIC COSINE

# 2. Insérer des documents avec leurs embeddings
HSET doc:1
  title "Guide Redis Cluster"
  content "Redis Cluster permet le sharding automatique..."
  category "architecture"
  embedding "<binary_embedding_1536_floats>"

# 3. Recherche vectorielle (KNN) avec filtres métadonnées
FT.SEARCH idx:docs
  "(@category:{architecture})=>[KNN 5 @embedding $query_vector AS score]"
  PARAMS 2 query_vector "<query_embedding>"
  RETURN 3 title content score
  SORTBY score ASC
  DIALECT 2

# Résultat en < 5ms (mémoire) :
# 1) "5"
# 2) "doc:1"
# 3) 1) "title"
#    2) "Guide Redis Cluster"
#    3) "score"
#    4) "0.92"  # Similarité cosine
```

**Avantages :**
- ✅ Latence **< 5ms** vs 50-200ms (disque)
- ✅ Combinaison filtres + vector search
- ✅ Algorithme HNSW optimisé pour la vitesse
- ✅ Tout en mémoire, pas de sérialisation

**Impact métier** : Réponses chatbot **10-40x plus rapides**.

---

## Les cas d'usage qui justifient Redis Stack

### 1️⃣ E-commerce et Marketplaces

**Problématique** : Recherche produits avec filtres multiples, tri, pagination

```bash
# Créer un index produits
FT.CREATE idx:products
  ON JSON
  PREFIX 1 product:
  SCHEMA
    $.name AS name TEXT WEIGHT 5.0 SORTABLE
    $.description AS description TEXT
    $.price AS price NUMERIC SORTABLE
    $.brand AS brand TAG SORTABLE
    $.category AS category TAG
    $.stock AS stock NUMERIC
    $.rating AS rating NUMERIC SORTABLE

# Recherche : "laptop gamer" + filtres prix et marque + tri par rating
FT.SEARCH idx:products
  "(@name|description:laptop gamer) @price:[800 2000] @brand:{ASUS|MSI|Dell}"
  SORTBY rating DESC
  LIMIT 0 20
  RETURN 5 $.name $.price $.brand $.rating $.stock
```

**Sans Redis Stack** : Elasticsearch (infrastructure complexe) ou base SQL lente.

---

### 2️⃣ IoT et Monitoring

**Problématique** : Milliers de capteurs envoyant des données chaque seconde

```bash
# Créer une série temporelle par capteur
TS.CREATE sensor:temp:dc1:server42
  RETENTION 604800000  # 7 jours
  DUPLICATE_POLICY LAST
  LABELS sensor_type temperature location dc1 server server42

# Ingestion continue (100K+ writes/sec possibles)
TS.ADD sensor:temp:dc1:server42 * 72.5
TS.ADD sensor:temp:dc1:server42 * 73.1
TS.ADD sensor:temp:dc1:server42 * 71.8

# Créer une agrégation automatique (moyenne sur 5 min)
TS.CREATERULE sensor:temp:dc1:server42
  sensor:temp:dc1:server42:avg_5min
  AGGREGATION avg 300000

# Query : Température moyenne par datacenter sur la dernière heure
TS.MRANGE - +
  AGGREGATION avg 300000
  FILTER sensor_type=temperature
  GROUPBY location
  REDUCE avg
```

**Sans Redis Stack** : InfluxDB, TimescaleDB (plus complexes, moins rapides).

---

### 3️⃣ Recommendation Engines

**Problématique** : Trouver des produits similaires en temps réel

```bash
# Index avec embeddings produits (générés par ML)
FT.CREATE idx:products_ml
  ON HASH
  PREFIX 1 prod:
  SCHEMA
    name TEXT
    embedding VECTOR FLAT 6 TYPE FLOAT32 DIM 128 DISTANCE_METRIC COSINE

# Lors de la consultation d'un produit
# → Récupérer son embedding
# → Trouver les 10 produits les plus similaires
FT.SEARCH idx:products_ml
  "*=>[KNN 10 @embedding $prod_embedding]"
  PARAMS 2 prod_embedding "<current_product_embedding>"
```

**Sans Redis Stack** : Milvus, Pinecone (coûteux, infrastructure séparée).

---

### 4️⃣ Real-time Analytics

**Problématique** : Dashboard avec métriques agrégées en temps réel

```bash
# Index pour analytics
FT.CREATE idx:events
  ON HASH
  PREFIX 1 event:
  SCHEMA
    user_id NUMERIC
    event_type TAG
    revenue NUMERIC
    timestamp NUMERIC SORTABLE
    country TAG

# Requête d'agrégation : Revenu total par pays, dernières 24h
FT.AGGREGATE idx:events
  "@timestamp:[$(now-86400) $(now)]"
  GROUPBY 1 @country
  REDUCE SUM 1 @revenue AS total_revenue
  REDUCE COUNT 0 AS event_count
  SORTBY 2 @total_revenue DESC
```

**Sans Redis Stack** : Elasticsearch (lent sur agrégations temps réel).

---

## Comparaison avec les alternatives

| Cas d'usage | Sans Redis Stack | Avec Redis Stack | Gain |
|-------------|------------------|------------------|------|
| **Recherche full-text** | Elasticsearch | RediSearch | Latence ÷10, Infra ÷3 |
| **Vector search (IA)** | Pinecone/Milvus | RediSearch | Latence ÷20, Coût ÷5 |
| **Time-series (IoT)** | InfluxDB | RedisTimeSeries | Throughput ×3 |
| **JSON partiel** | MongoDB | RedisJSON | Latence ÷5 |
| **Filtres probabilistes** | Implémentation custom | RedisBloom | Dev time ÷10 |

---

## Architecture : Comment Redis Stack fonctionne

### Modules vs Base Redis séparée

```
┌─────────────────────────────────────────────────────┐
│              Application                            │
└───────────────┬─────────────────────────────────────┘
                │
        ┌───────▼───────┐
        │  Redis Client │
        └───────┬───────┘
                │
┌───────────────▼─────────────────────────────────────┐
│                Redis Stack Server                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │RedisJSON │  │RediSearch│  │TimeSeries│   (Modules)
│  └─────┬────┘  └─────┬────┘  └─────┬────┘           │
│        │             │             │                │
│  ┌─────▼─────────────▼─────────────▼──────────────┐ │
│  │         Redis Core Engine                      │ │
│  │  • Single-threaded event loop                  │ │
│  │  • I/O Multiplexing (epoll/kqueue)             │ │
│  │  • In-memory data structures                   │ │
│  │  • RDB/AOF persistence                         │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Points clés :**
- Les modules sont des **shared libraries** chargées dans Redis
- Ils **étendent les commandes** disponibles (JSON.SET, FT.SEARCH, etc.)
- Ils utilisent la **même infrastructure** (networking, persistence, réplication)
- **Single-threaded** : pas de complexité de concurrence

---

## Performance : Redis Stack vs Redis Core

### Benchmark : GET/SET simples

```bash
# Redis Core
redis-benchmark -t set,get -n 1000000 -q
SET: 120,000 requests per second
GET: 130,000 requests per second

# Redis Stack (avec modules chargés mais non utilisés)
SET: 115,000 requests per second  # -4%
GET: 125,000 requests per second  # -4%
```

**Conclusion** : Impact négligeable si vous n'utilisez pas les modules.

### Benchmark : Recherche full-text (10K documents)

```bash
# Redis Core (SCAN + filtre applicatif)
Temps moyen : 450ms

# RediSearch (index)
Temps moyen : 4ms

# Gain : 112x plus rapide
```

### Benchmark : Vector Search (100K vecteurs, dim 768)

```bash
# PostgreSQL + pgvector
Latence p50 : 85ms
Latence p99 : 320ms

# Redis Stack (HNSW)
Latence p50 : 3ms
Latence p99 : 12ms

# Gain : 28x plus rapide (p50)
```

---

## Migration : Faut-il tout réécrire ?

**Non !** Redis Stack est **100% rétrocompatible** avec Redis Core.

### Approche progressive

```bash
# 1. Installer Redis Stack (remplace Redis Core)
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest

# 2. Votre code existant continue de fonctionner
SET session:abc123 "{\"user_id\":1001}"
GET session:abc123
# Pas de changement

# 3. Migrer progressivement vers RedisJSON
JSON.SET session:abc123 $ '{"user_id":1001,"cart":[]}'
JSON.GET session:abc123
# Cohabitation possible
```

### Stratégie de migration

**Phase 1** : Nouvelles features utilisent Redis Stack
- Nouvelles recherches → RediSearch
- Nouvelles séries temporelles → RedisTimeSeries

**Phase 2** : Migration opportuniste
- Lors de refactoring, migrer vers RedisJSON
- Ajouter des index progressivement

**Phase 3** : Optimisation
- Analyser les hotspots (SLOWLOG)
- Migrer les requêtes lentes vers les modules adaptés

---

## Considérations de déploiement

### Mémoire

Redis Stack consomme **plus de RAM** :

```bash
# Exemple : 1 million de documents JSON (2KB chacun)

# Redis Core (String sérialisé) :
# 1M × 2KB = 2GB + overhead = ~2.3GB

# RedisJSON :
# 1M × 2KB = 2GB + overhead RedisJSON = ~2.5GB (+10%)

# RediSearch (avec index full-text) :
# 2.5GB + index (500MB) = ~3GB (+30%)
```

**Trade-off** : +20-30% mémoire pour des performances 50-100x meilleures.

### Compatibilité avec les forks

| Fork | Support Redis Stack |
|------|---------------------|
| **Redis OSS** | ✅ Complet |
| **Redis Enterprise** | ✅ Complet + features additionnelles |
| **Valkey** | ❌ Non (fork sans modules) |
| **KeyDB** | ⚠️ Partiel (modules à recompiler) |
| **Dragonfly** | ⚠️ En développement |

**Attention** : Si vous utilisez Valkey/KeyDB, Redis Stack n'est **pas disponible**.

---

## Cas d'usage : Quand NE PAS utiliser Redis Stack

### ❌ Caching simple clé-valeur

```bash
# Si vous faites ça :
SET cache:user:1001:profile '{"name":"Alice"}'
GET cache:user:1001:profile

# Redis Core suffit largement
# Redis Stack est un overkill
```

### ❌ Files d'attente simples

```bash
# Si vous utilisez uniquement Lists :
LPUSH queue:jobs '{"task":"send_email"}'
BRPOP queue:jobs 0

# Redis Core est parfait
# Redis Streams serait mieux, mais pas Redis Stack
```

### ❌ Compteurs et leaderboards

```bash
# Redis Core est optimal :
INCR counter:page_views
ZADD leaderboard 1500 "player:alice"
ZRANGE leaderboard 0 9 WITHSCORES REV
```

### ❌ Contraintes d'infrastructure

- **Mémoire limitée** : Redis Stack consomme 20-30% de plus
- **Compatibilité requise** : Si vous devez supporter Valkey
- **Simplicité maximale** : Redis Core = moins de complexité

---

## Le verdict : Devriez-vous adopter Redis Stack ?

### ✅ Adoptez Redis Stack si :

1. **Recherche complexe** : Filtres, tri, agrégations
2. **Documents JSON** : Manipulation partielle fréquente
3. **IA/ML** : Vector search, recommendation engines
4. **Time-series** : IoT, monitoring, métriques
5. **Analytics temps réel** : Dashboards avec agrégations
6. **Bloom filters** : Gros volumes, deduplication

### ⚠️ Évaluez l'impact si :

- Mémoire limitée (calculez +20-30%)
- Besoin de compatibilité avec Valkey/KeyDB
- Équipe peu familière avec Redis (courbe d'apprentissage)

### ❌ Restez sur Redis Core si :

- Caching simple sans recherche
- Compteurs, leaderboards basiques
- Files d'attente simples (Lists/Streams)
- Infrastructure très contrainte

---

## Exemple complet : Avant/Après Redis Stack

### Cas d'usage : Moteur de recherche e-commerce

**Avant (Redis Core + Elasticsearch)** :

```
┌────────────┐         ┌──────────────┐
│ Application│────────▶│    Redis     │ (Cache session, panier)
│            │         │     Core     │
└──────┬─────┘         └──────────────┘
       │
       │               ┌──────────────┐
       └──────────────▶│ Elasticsearch│ (Recherche produits)
                       └──────────────┘
```

**Infrastructure** :
- 3 nœuds Redis (cache)
- 3 nœuds Elasticsearch (recherche)
- **Coût** : ~$800/mois

**Latence recherche** : 30-80ms

---

**Après (Redis Stack)** :

```
┌────────────┐         ┌──────────────┐
│ Application│────────▶│ Redis Stack  │
│            │         │  (All-in-one)│
└────────────┘         └──────────────┘
```

**Infrastructure** :
- 3 nœuds Redis Stack
- **Coût** : ~$300/mois (-62%)

**Latence recherche** : 3-8ms (-80%)

**Code côté application** :

```javascript
// Avant (2 appels réseau) :
const session = await redis.get(`session:${userId}`);
const products = await elasticsearch.search({
  query: { match: { name: searchQuery } }
});

// Après (1 appel réseau) :
const [session, products] = await Promise.all([
  redis.json.get(`session:${userId}`),
  redis.ft.search('idx:products', searchQuery)
]);
```

---

## Checklist de décision

Utilisez cette checklist pour décider si Redis Stack vous convient :

```
[ ] J'ai besoin de recherche full-text sur mes données
[ ] Je manipule des documents JSON complexes
[ ] Je construis un système avec IA/ML (embeddings)
[ ] Je gère des séries temporelles (IoT, métriques)
[ ] J'ai des agrégations en temps réel à calculer
[ ] Je peux allouer 20-30% de mémoire supplémentaire
[ ] Mon équipe peut monter en compétence sur les nouveaux modules
[ ] Je ne suis pas contraint à Valkey/KeyDB
[ ] Je cherche à simplifier mon architecture (moins de composants)
[ ] Je veux réduire la latence de mes requêtes complexes

Score :
- 7+ ✅ : Redis Stack est un excellent choix
- 4-6 ⚠️ : Évaluez projet par projet
- 0-3 ❌ : Redis Core suffit probablement
```

---

## Prochaines étapes

Maintenant que vous comprenez **pourquoi** adopter Redis Stack, les sections suivantes vont vous montrer **comment** :

- **Section 3.2** : RedisJSON - Maîtriser les documents JSON natifs
- **Section 3.3** : RediSearch - Indexation et recherche full-text
- **Section 3.4** : RediSearch - Agrégations et requêtes complexes
- **Section 3.5** : RediSearch - Vector Search pour l'IA/RAG
- **Section 3.6** : RedisTimeSeries - Séries temporelles et IoT
- **Section 3.7** : RedisBloom - Filtres probabilistes avancés

---

## Ressources

### Documentation officielle
- [Redis Stack Overview](https://redis.io/docs/stack/)
- [Redis Stack vs Redis OSS](https://redis.io/docs/stack/about/)

### Benchmarks
- [RediSearch Performance](https://redis.io/docs/stack/search/design/performance/)
- [Vector Search Benchmarks](https://redis.io/blog/benchmarking-results-for-vector-databases/)

### Guides de migration
- [Migrating from Redis OSS to Stack](https://redis.io/docs/stack/get-started/migration/)
- [Best Practices](https://redis.io/docs/stack/get-started/best-practices/)

---

**Prêt à explorer RedisJSON ?** Passons à la section suivante : [3.2 RedisJSON - Documents JSON natifs](./02-redisjson-documents-json.md)

⏭️ [RedisJSON : Stocker et manipuler des documents JSON natifs](/03-structures-donnees-etendues/02-redisjson-documents-json.md)
