🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #2 : Moteur de recherche e-commerce avec RediSearch

## Vue d'ensemble

**Niveau** : ⭐⭐⭐ Avancé
**Complexité technique** : Élevée
**Impact production** : Critique (conversion et revenus)
**Technologies** : Redis Stack (RediSearch + RedisJSON)

---

## 1. Contexte et problématique

### Scénario business

Une plateforme e-commerce multi-vendeurs (type Amazon, Etsy, ou Cdiscount) avec les caractéristiques suivantes :

**Chiffres clés** :
- 50 millions de produits actifs
- 100 000 nouveaux produits par jour
- 5 millions de recherches par heure (peak)
- 2 000 catégories et 10 000 marques
- 15 000 vendeurs
- SLA : Latence < 50ms (p95) pour recherche
- Conversion : 3.5% baseline → objectif 5%

**Besoins métier prioritaires** :

1. **Recherche full-text ultra-rapide**
   - Recherche multi-langue (FR, EN, ES, DE)
   - Tolérance aux fautes de frappe
   - Synonymes et stemming
   - Recherche dans titre, description, marque, catégorie

2. **Filtres facettés (Faceted Search)**
   - Prix (ranges dynamiques)
   - Marque (avec compteurs)
   - Catégorie hiérarchique
   - Couleur, taille, matériau
   - Disponibilité en stock
   - Note client (rating)
   - Livraison gratuite

3. **Tri et pertinence**
   - Tri par pertinence (BM25 scoring)
   - Tri par prix (asc/desc)
   - Tri par popularité (ventes)
   - Tri par note client
   - Boost des produits sponsorisés

4. **Autocomplete et suggestions**
   - Suggestions de requêtes (as-you-type)
   - Correction orthographique
   - Suggestions de produits populaires

5. **Personnalisation**
   - Boost basé sur historique utilisateur
   - Filtrage géographique (disponibilité locale)
   - Prix personnalisés (promo membre)

### Problèmes à résoudre

#### 1. **Latence de recherche inadmissible**

```
❌ État actuel : PostgreSQL Full-Text Search
SELECT p.* FROM products p
WHERE to_tsvector('french', p.title || ' ' || p.description)
      @@ plainto_tsquery('french', 'chaussures running')
AND p.price BETWEEN 50 AND 150
AND p.stock > 0
ORDER BY ts_rank(...) DESC
LIMIT 20;

Problèmes :
- Latence : 500-2000ms (inacceptable)
- Pas de facettes en temps réel
- Scaling vertical uniquement
- Pas d'autocomplete natif
- Index GiST/GIN volumineux et lents
```

#### 2. **Facettes dynamiques coûteuses**

```sql
-- Compter les produits par marque (pour afficher filtres)
SELECT brand, COUNT(*)
FROM products
WHERE [...conditions de recherche...]
GROUP BY brand;

-- Requête séparée pour chaque facette !
-- 5 facettes = 6 requêtes SQL minimum
-- Latence totale : 2-3 secondes
```

#### 3. **Indexation et mises à jour**

```
Contraintes :
- 100k nouveaux produits/jour
- Prix changent toutes les 5 minutes (promo flash)
- Stock mis à jour à chaque vente
- Réindexation complète PostgreSQL : 8 heures !

Problème : Index stale = mauvaise expérience utilisateur
```

#### 4. **Scaling et coûts**

```
Infrastructure actuelle :
- PostgreSQL Primary : 96 CPU, 768GB RAM
- 8 Read Replicas pour distribuer les recherches
- Coût mensuel : $8,000

Problème : Scaling linéaire avec volume de recherches
```

---

## 2. Analyse des alternatives

### Option 1 : Elasticsearch

```json
POST /products/_search
{
  "query": {
    "multi_match": {
      "query": "chaussures running",
      "fields": ["title^3", "description", "brand^2"]
    }
  },
  "aggs": {
    "brands": { "terms": { "field": "brand.keyword" } },
    "price_ranges": { "range": { "field": "price", "ranges": [...] } }
  }
}
```

**Avantages** :
- ✅ Recherche full-text puissante (Lucene)
- ✅ Agrégations (facettes) natives
- ✅ Scaling horizontal (sharding)
- ✅ Écosystème mature (Kibana, Logstash)

**Inconvénients** :
- ❌ Latence : 20-100ms (JVM overhead)
- ❌ Consommation mémoire élevée (Java heap)
- ❌ Complexité opérationnelle (GC tuning, cluster management)
- ❌ Coût : Cluster 5 nodes = $2,000/mois minimum
- ❌ Near real-time indexing (1s refresh interval)

**Verdict** : ⚠️ **Solution robuste mais coûteuse**. Overkill pour nos besoins si budget limité.

---

### Option 2 : Algolia (SaaS)

```javascript
index.search('chaussures running', {
  filters: 'price:50 TO 150 AND stock > 0',
  facets: ['brand', 'category', 'color']
});
```

**Avantages** :
- ✅ Latence < 10ms (CDN global)
- ✅ Typo-tolerance et synonymes out-of-the-box
- ✅ Zero ops (fully managed)
- ✅ Excellent autocomplete

**Inconvénients** :
- ❌ Coût élevé : $1/1000 recherches + $0.50/1000 records
  - 5M recherches/h × 24h × 30j = 3.6B recherches/mois
  - Coût : **$3.6M/mois** (prohibitif !)
- ❌ Vendor lock-in
- ❌ Personnalisation limitée

**Verdict** : ❌ **Trop coûteux à notre échelle**.

---

### Option 3 : RediSearch ✅

```bash
FT.SEARCH idx:products "chaussures running"
  FILTER price 50 150
  FILTER stock 1 +inf
  RETURN 3 title price brand
  SORTBY _score DESC
  LIMIT 0 20
```

**Avantages** :
- ✅ Latence : **< 5ms** (p95) grâce à in-memory
- ✅ Indexation temps réel (< 100ms après write)
- ✅ Agrégations natives (FT.AGGREGATE)
- ✅ Typo-tolerance, stemming, phonétique
- ✅ Autocomplete via SUGGET
- ✅ Coût : **$500-1000/mois** (vs $8k PostgreSQL)
- ✅ Simple à opérer (Redis standard)

**Inconvénients** :
- ⚠️ Limité à RAM disponible (mais acceptable : 50M produits × 5KB = 250GB)
- ⚠️ Fonctionnalités moins riches qu'Elasticsearch (pas de ML, graph queries)
- ⚠️ Dépendance Redis Stack

**Trade-offs assumés** :
- ➕ Performance × 100 vs PostgreSQL
- ➕ Coût ÷ 8 vs Infrastructure actuelle
- ➕ Simplicité vs Elasticsearch
- ➖ Besoin de RAM (mais mémoire pas chère)

**Verdict** : ✅ **Solution optimale** pour rapport performance/coût/simplicité.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │  Web Server  │   │ Mobile API   │   │  Admin API   │      │
│  │  (Next.js)   │   │  (GraphQL)   │   │  (Django)    │      │
│  └───────┬──────┘   └───────┬──────┘   └───────┬──────┘      │
└──────────┼──────────────────┼──────────────────┼─────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌──────────────────────────────────────────────────────────────┐
│              Search Service (Load Balanced)                  │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  - Query parsing & validation                       │     │
│  │  - User context enrichment (location, history)      │     │
│  │  - Cache warming & precomputation                   │     │
│  │  - Result ranking & personalization                 │     │
│  └─────────────────────────────────────────────────────┘     │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│                  RediSearch Cluster                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │  Shard 1   │  │  Shard 2   │  │  Shard 3   │              │
│  │ Hash 0-5k  │  │ Hash 5k-11k│  │Hash 11k-16k│              │
│  │  + Replica │  │  + Replica │  │  + Replica │              │
│  └────────────┘  └────────────┘  └────────────┘              │
│                                                              │
│  Index: idx:products (50M documents)                         │
│  - Full-text fields: title, description                      │
│  - TAG fields: brand, category, color                        │
│  - NUMERIC fields: price, stock, rating                      │
│  - GEO field: seller_location                                │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Product Database (Source of Truth)             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  PostgreSQL (OLTP)                                   │   │
│  │  - Transactional writes (orders, inventory)          │   │
│  │  - CDC (Change Data Capture) → Redis                 │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
           ▲
           │ CDC Stream (Debezium)
           │
┌──────────┴──────────────────────────────────────────────────┐
│              Data Pipeline (Real-time Sync)                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Kafka / Redis Streams                               │   │
│  │  - product.created                                   │   │
│  │  - product.updated                                   │   │
│  │  - product.deleted                                   │   │
│  │  - inventory.changed                                 │   │
│  └───────────┬──────────────────────────────────────────┘   │
│              ▼                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Indexer Service                                     │   │
│  │  - Transform data for RediSearch                     │   │
│  │  - Batch updates (1000/sec)                          │   │
│  │  - Handle failures & retries                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Flux de données

#### **Flux de recherche (Read Path)**

```
1. User types "chaussures running nike"
   ├─ Frontend debounce (300ms)
   └─ POST /api/search

2. Search Service receives request
   ├─ Parse query (tokenization, normalization)
   ├─ Enrich with user context:
   │  ├─ User location (geo filter)
   │  ├─ User history (personalization boost)
   │  └─ User segment (price sensitivity)
   ├─ Check cache (query → results)
   │  └─ If hit → return (latency ~1ms)
   └─ If miss → proceed

3. Build RediSearch query
   ├─ Full-text: "(chaussures|shoes) running nike"
   ├─ Filters: stock > 0, location < 50km
   ├─ Facets: brand, price_range, rating
   └─ Sort: _score DESC

4. FT.SEARCH idx:products [query]
   ├─ Distributed across shards
   ├─ Each shard returns top 20 local results
   └─ Coordinator merges & sorts

5. Post-processing
   ├─ Apply business rules (hide out-of-stock)
   ├─ Inject sponsored products (positions 1, 5, 10)
   ├─ Personalization boost (+20% if in wishlist)
   └─ Cache result (TTL 5 minutes)

6. Return JSON to client
   └─ Latency: 3-8ms (p95)

Total latency budget:
- Network: 1ms
- Search Service: 1ms
- RediSearch: 3-5ms
- Post-process: 1-2ms
= 6-9ms (p95)
```

#### **Flux d'indexation (Write Path)**

```
1. Product updated in PostgreSQL
   ├─ UPDATE products SET price = 99.99 WHERE id = 12345
   └─ Trigger: pg_notify('product_update', '{"id": 12345, ...}')

2. CDC Pipeline (Debezium / Custom)
   ├─ Capture change log
   ├─ Transform to RediSearch format
   └─ Publish to Kafka / Redis Stream

3. Indexer Service consumes event
   ├─ Batch: collect 1000 events or 100ms timeout
   ├─ Transform to RedisJSON format
   └─ Bulk index via pipeline

4. Redis Pipeline execution
   ├─ JSON.SET product:12345 $ {...}
   ├─ JSON.SET product:12346 $ {...}
   ├─ ... (1000 operations)
   └─ Pipeline executes (latency ~50ms)

5. RediSearch auto-indexes
   ├─ Background indexing thread
   ├─ Index built in < 100ms
   └─ Searchable immediately

6. Invalidate cache (if needed)
   └─ DEL cache:query:* (selective invalidation)

Throughput: 100k products indexed/sec
Lag: < 200ms (write → searchable)
```

---

## 4. Modélisation des données

### 4.1 Schéma RedisJSON

**Clé** : `product:{product_id}`
**Type** : RedisJSON

```json
{
  "product_id": "prod_7k3m9p2x",
  "sku": "NIKE-AIR-MAX-270-BLK-42",

  "title": "Nike Air Max 270 Chaussures de Running Homme",
  "description": "Découvrez le confort ultime avec la Nike Air Max 270. Unité Max Air visible pour un amorti exceptionnel. Tige en mesh respirant pour une ventilation optimale. Semelle en caoutchouc pour une adhérence maximale.",

  "brand": "Nike",
  "category": "Chaussures > Running > Homme",
  "category_id": "cat_running_homme",
  "category_path": ["Chaussures", "Running", "Homme"],

  "price": 149.99,
  "currency": "EUR",
  "original_price": 189.99,
  "discount_percent": 21,

  "stock": 47,
  "stock_status": "in_stock",
  "low_stock_threshold": 10,

  "rating": 4.7,
  "review_count": 1834,
  "rating_distribution": {
    "5": 1245,
    "4": 412,
    "3": 98,
    "2": 45,
    "1": 34
  },

  "attributes": {
    "color": "Noir",
    "size": "42",
    "material": "Mesh synthétique",
    "weight": "310g",
    "gender": "Homme",
    "age_group": "Adulte"
  },

  "variants": [
    {
      "variant_id": "var_abc123",
      "color": "Noir",
      "size": "42",
      "sku": "NIKE-AIR-MAX-270-BLK-42",
      "stock": 47,
      "price": 149.99
    },
    {
      "variant_id": "var_abc124",
      "color": "Blanc",
      "size": "42",
      "sku": "NIKE-AIR-MAX-270-WHT-42",
      "stock": 23,
      "price": 149.99
    }
  ],

  "seller": {
    "seller_id": "seller_xyz789",
    "name": "Nike Store Paris",
    "rating": 4.9,
    "location": {
      "lat": 48.8566,
      "lon": 2.3522,
      "city": "Paris",
      "country": "FR"
    },
    "shipping": {
      "free_shipping": true,
      "delivery_time_days": 2
    }
  },

  "images": [
    "https://cdn.example.com/products/prod_7k3m9p2x/main.jpg",
    "https://cdn.example.com/products/prod_7k3m9p2x/side.jpg",
    "https://cdn.example.com/products/prod_7k3m9p2x/detail.jpg"
  ],

  "seo": {
    "slug": "nike-air-max-270-running-homme-noir",
    "keywords": ["nike", "air max", "running", "chaussures sport"]
  },

  "metrics": {
    "views_30d": 12450,
    "sales_30d": 247,
    "conversion_rate": 0.0198,
    "popularity_score": 87.5
  },

  "flags": {
    "is_new": false,
    "is_bestseller": true,
    "is_featured": false,
    "is_sponsored": false,
    "is_eco_friendly": true
  },

  "metadata": {
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-12-11T14:25:00Z",
    "indexed_at": "2024-12-11T14:25:12Z",
    "version": 47
  }
}
```

### 4.2 Définition de l'index RediSearch

```bash
FT.CREATE idx:products
  ON JSON
  PREFIX 1 product:
  SCHEMA
    # Full-text search fields
    $.title AS title TEXT WEIGHT 5.0 PHONETIC dm:fr
    $.description AS description TEXT WEIGHT 1.0 PHONETIC dm:fr
    $.brand AS brand_text TEXT WEIGHT 3.0
    $.seo.keywords[*] AS keywords TEXT

    # TAG fields (exact match, facets)
    $.brand AS brand TAG SORTABLE
    $.category_id AS category TAG SORTABLE
    $.attributes.color AS color TAG
    $.attributes.size AS size TAG
    $.attributes.gender AS gender TAG
    $.stock_status AS stock_status TAG

    # NUMERIC fields (range queries, sorting)
    $.price AS price NUMERIC SORTABLE
    $.stock AS stock NUMERIC SORTABLE
    $.rating AS rating NUMERIC SORTABLE
    $.review_count AS review_count NUMERIC SORTABLE
    $.discount_percent AS discount NUMERIC SORTABLE
    $.metrics.popularity_score AS popularity NUMERIC SORTABLE

    # GEO field (location-based search)
    $.seller.location AS location GEO

    # Boolean flags (filtering)
    $.flags.is_new AS is_new TAG
    $.flags.is_bestseller AS is_bestseller TAG
    $.seller.shipping.free_shipping AS free_shipping TAG
```

**Paramètres clés** :

- `WEIGHT` : Pondération pour le scoring (titre = 5× plus important que description)
- `PHONETIC dm:fr` : Tolérance phonétique en français (Double Metaphone)
- `SORTABLE` : Permet le tri sur ce champ
- `GEO` : Support des requêtes géospatiales

### 4.3 Index secondaires et optimisations

```bash
# Autocomplete (suggestions)
FT.SUGADD autocomplete:brands "Nike" 1000
FT.SUGADD autocomplete:brands "Adidas" 950
FT.SUGADD autocomplete:brands "New Balance" 800

FT.SUGADD autocomplete:categories "Running" 5000
FT.SUGADD autocomplete:categories "Basketball" 3500

# Synonymes pour améliorer la recherche
FT.SYNUPDATE idx:products synonym_group1
  "chaussures" "souliers" "baskets" "sneakers"

FT.SYNUPDATE idx:products synonym_group2
  "running" "course" "jogging"
```

---

## 5. Implémentation technique

### 5.1 Search Service - Python (Production-Ready)

```python
"""
E-commerce Search Service avec RediSearch
Implémentation production-ready avec caching, facets, personnalisation
"""

import json
import time
import hashlib
from typing import List, Dict, Any, Optional, Tuple
from dataclasses import dataclass, asdict
from enum import Enum
import logging

import redis
from redis.commands.search.query import Query
from redis.commands.search.aggregation import AggregateRequest, Asc, Desc
from redis.commands.search import reducers

# Configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'decode_responses': True,
    'socket_timeout': 3,
    'socket_connect_timeout': 3,
    'retry_on_timeout': True,
    'max_connections': 200
}

# Index name
INDEX_NAME = "idx:products"

# Cache TTL
SEARCH_CACHE_TTL = 300  # 5 minutes


# ============================================================================
# Data Classes
# ============================================================================

class SortBy(Enum):
    RELEVANCE = "_score"
    PRICE_ASC = "price"
    PRICE_DESC = "-price"
    RATING = "-rating"
    POPULARITY = "-popularity"
    NEWEST = "-created_at"


@dataclass
class SearchFilters:
    """Filtres de recherche"""
    min_price: Optional[float] = None
    max_price: Optional[float] = None
    brands: Optional[List[str]] = None
    categories: Optional[List[str]] = None
    colors: Optional[List[str]] = None
    sizes: Optional[List[str]] = None
    min_rating: Optional[float] = None
    in_stock_only: bool = True
    free_shipping_only: bool = False
    is_new: Optional[bool] = None
    is_bestseller: Optional[bool] = None

    # Geo filter
    user_lat: Optional[float] = None
    user_lon: Optional[float] = None
    max_distance_km: Optional[float] = None


@dataclass
class SearchParams:
    """Paramètres de recherche"""
    query: str
    filters: SearchFilters
    sort_by: SortBy = SortBy.RELEVANCE
    page: int = 1
    page_size: int = 20
    include_facets: bool = True
    user_id: Optional[str] = None  # Pour personnalisation


@dataclass
class FacetValue:
    """Valeur d'une facette avec compteur"""
    value: str
    count: int


@dataclass
class Facets:
    """Facettes (filtres) avec compteurs"""
    brands: List[FacetValue]
    categories: List[FacetValue]
    colors: List[FacetValue]
    price_ranges: List[FacetValue]
    rating_ranges: List[FacetValue]


@dataclass
class SearchResult:
    """Résultat de recherche"""
    product_id: str
    title: str
    brand: str
    price: float
    original_price: Optional[float]
    rating: float
    review_count: int
    image_url: str
    stock_status: str
    free_shipping: bool
    is_bestseller: bool
    relevance_score: float


@dataclass
class SearchResponse:
    """Réponse complète de recherche"""
    results: List[SearchResult]
    total_count: int
    page: int
    page_size: int
    total_pages: int
    facets: Optional[Facets]
    search_time_ms: float
    from_cache: bool


# ============================================================================
# Search Service
# ============================================================================

class ProductSearchService:
    """
    Service de recherche produits avec RediSearch

    Features:
    - Full-text search avec phonétique
    - Filtres facettés avec compteurs
    - Tri multi-critères
    - Personnalisation basée sur user
    - Cache intelligent des résultats
    - Géolocalisation
    """

    def __init__(self, redis_config: Dict = None):
        config = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**config)

        # Vérifier que l'index existe
        try:
            self.redis.ft(INDEX_NAME).info()
            logger.info(f"Index {INDEX_NAME} found and ready")
        except redis.ResponseError:
            logger.error(f"Index {INDEX_NAME} not found!")
            raise

    def _generate_cache_key(self, params: SearchParams) -> str:
        """Générer clé de cache basée sur params de recherche"""
        # Exclure user_id pour mutualiser le cache
        cache_params = {
            'query': params.query,
            'filters': asdict(params.filters),
            'sort_by': params.sort_by.value,
            'page': params.page,
            'page_size': params.page_size
        }

        # Hash SHA256 pour clé compacte
        params_str = json.dumps(cache_params, sort_keys=True)
        cache_hash = hashlib.sha256(params_str.encode()).hexdigest()[:16]

        return f"cache:search:{cache_hash}"

    def _build_query_string(self, params: SearchParams) -> str:
        """Construire la query string RediSearch"""
        filters = params.filters

        # Base query (full-text)
        if params.query.strip():
            # Escape special characters
            query_escaped = params.query.replace('-', ' ')
            base_query = f"({query_escaped})"
        else:
            base_query = "*"  # Match all

        # Filters
        filter_parts = []

        # Price range
        if filters.min_price is not None or filters.max_price is not None:
            min_p = filters.min_price or 0
            max_p = filters.max_price or "+inf"
            filter_parts.append(f"@price:[{min_p} {max_p}]")

        # Brands (OR logic)
        if filters.brands:
            brands_query = " | ".join(f"{{{b}}}" for b in filters.brands)
            filter_parts.append(f"@brand:{{{brands_query}}}")

        # Categories
        if filters.categories:
            cats_query = " | ".join(f"{{{c}}}" for c in filters.categories)
            filter_parts.append(f"@category:{{{cats_query}}}")

        # Colors
        if filters.colors:
            colors_query = " | ".join(f"{{{c}}}" for c in filters.colors)
            filter_parts.append(f"@color:{{{colors_query}}}")

        # Sizes
        if filters.sizes:
            sizes_query = " | ".join(f"{{{s}}}" for s in filters.sizes)
            filter_parts.append(f"@size:{{{sizes_query}}}")

        # Rating
        if filters.min_rating:
            filter_parts.append(f"@rating:[{filters.min_rating} +inf]")

        # Stock
        if filters.in_stock_only:
            filter_parts.append("@stock:[1 +inf]")

        # Free shipping
        if filters.free_shipping_only:
            filter_parts.append("@free_shipping:{true}")

        # Flags
        if filters.is_new is not None:
            filter_parts.append(f"@is_new:{{{str(filters.is_new).lower()}}}")

        if filters.is_bestseller is not None:
            filter_parts.append(f"@is_bestseller:{{{str(filters.is_bestseller).lower()}}}")

        # Combine
        if filter_parts:
            full_query = f"{base_query} {' '.join(filter_parts)}"
        else:
            full_query = base_query

        return full_query

    def _build_redis_query(self, params: SearchParams) -> Query:
        """Construire l'objet Query RediSearch"""
        query_str = self._build_query_string(params)

        # Pagination
        offset = (params.page - 1) * params.page_size

        query = Query(query_str) \
            .paging(offset, params.page_size) \
            .no_content()  # Ne pas retourner le document complet (on le fetch après)

        # Sorting
        if params.sort_by != SortBy.RELEVANCE:
            sort_field = params.sort_by.value
            if sort_field.startswith('-'):
                query.sort_by(sort_field[1:], asc=False)
            else:
                query.sort_by(sort_field, asc=True)

        # Geo filter (si présent)
        filters = params.filters
        if all([filters.user_lat, filters.user_lon, filters.max_distance_km]):
            radius_m = filters.max_distance_km * 1000
            query.add_filter(
                f"@location:[{filters.user_lon} {filters.user_lat} {radius_m} m]"
            )

        # Return specific fields pour optimisation
        query.return_fields(
            'product_id', 'title', 'brand', 'price', 'original_price',
            'rating', 'review_count', 'images', 'stock_status',
            'free_shipping', 'is_bestseller'
        )

        return query

    def search(self, params: SearchParams) -> SearchResponse:
        """
        Recherche produits avec RediSearch

        Returns:
            SearchResponse avec résultats et facettes
        """
        start_time = time.time()

        # Check cache
        cache_key = self._generate_cache_key(params)
        cached = self.redis.get(cache_key)

        if cached:
            logger.info(f"Cache hit for query: {params.query}")
            response_dict = json.loads(cached)
            response = SearchResponse(**response_dict)
            response.from_cache = True
            return response

        # Build query
        query = self._build_redis_query(params)

        try:
            # Execute search
            result = self.redis.ft(INDEX_NAME).search(query)

            # Parse results
            results = []
            for doc in result.docs:
                # Parse images (stored as JSON string)
                images = json.loads(doc.images) if hasattr(doc, 'images') else []

                results.append(SearchResult(
                    product_id=doc.product_id,
                    title=doc.title,
                    brand=doc.brand,
                    price=float(doc.price),
                    original_price=float(doc.original_price) if hasattr(doc, 'original_price') else None,
                    rating=float(doc.rating),
                    review_count=int(doc.review_count),
                    image_url=images[0] if images else "",
                    stock_status=doc.stock_status,
                    free_shipping=doc.free_shipping == 'true',
                    is_bestseller=doc.is_bestseller == 'true',
                    relevance_score=1.0  # TODO: extract from RediSearch
                ))

            # Get facets (si demandé)
            facets = None
            if params.include_facets:
                facets = self._compute_facets(params)

            # Build response
            total_pages = (result.total + params.page_size - 1) // params.page_size

            response = SearchResponse(
                results=results,
                total_count=result.total,
                page=params.page,
                page_size=params.page_size,
                total_pages=total_pages,
                facets=facets,
                search_time_ms=(time.time() - start_time) * 1000,
                from_cache=False
            )

            # Cache result
            self.redis.setex(
                cache_key,
                SEARCH_CACHE_TTL,
                json.dumps(asdict(response))
            )

            logger.info(
                f"Search completed: query='{params.query}', "
                f"results={result.total}, time={response.search_time_ms:.2f}ms"
            )

            return response

        except redis.ResponseError as e:
            logger.error(f"RediSearch error: {e}")
            raise

    def _compute_facets(self, params: SearchParams) -> Facets:
        """
        Calculer les facettes avec compteurs via FT.AGGREGATE
        """
        base_query = self._build_query_string(params)

        # Aggregation pour chaque facette
        facets_data = {}

        # Brands facet
        agg_brands = AggregateRequest(base_query) \
            .group_by('@brand', reducers.count().alias('count')) \
            .sort_by(Desc('@count')) \
            .limit(0, 20)

        try:
            result_brands = self.redis.ft(INDEX_NAME).aggregate(agg_brands).rows
            facets_data['brands'] = [
                FacetValue(value=row[1], count=int(row[3]))
                for row in result_brands
            ]
        except Exception as e:
            logger.warning(f"Failed to compute brands facet: {e}")
            facets_data['brands'] = []

        # Categories facet
        agg_categories = AggregateRequest(base_query) \
            .group_by('@category', reducers.count().alias('count')) \
            .sort_by(Desc('@count')) \
            .limit(0, 20)

        try:
            result_categories = self.redis.ft(INDEX_NAME).aggregate(agg_categories).rows
            facets_data['categories'] = [
                FacetValue(value=row[1], count=int(row[3]))
                for row in result_categories
            ]
        except Exception as e:
            logger.warning(f"Failed to compute categories facet: {e}")
            facets_data['categories'] = []

        # Colors facet
        agg_colors = AggregateRequest(base_query) \
            .group_by('@color', reducers.count().alias('count')) \
            .sort_by(Desc('@count')) \
            .limit(0, 20)

        try:
            result_colors = self.redis.ft(INDEX_NAME).aggregate(agg_colors).rows
            facets_data['colors'] = [
                FacetValue(value=row[1], count=int(row[3]))
                for row in result_colors
            ]
        except Exception as e:
            logger.warning(f"Failed to compute colors facet: {e}")
            facets_data['colors'] = []

        # Price ranges (predefined buckets)
        price_buckets = [
            (0, 50, "0-50€"),
            (50, 100, "50-100€"),
            (100, 200, "100-200€"),
            (200, 500, "200-500€"),
            (500, float('inf'), "500€+")
        ]

        price_facets = []
        for min_p, max_p, label in price_buckets:
            # Count products in this range
            count_query = f"{base_query} @price:[{min_p} {max_p}]"
            try:
                count_result = self.redis.ft(INDEX_NAME).search(
                    Query(count_query).no_content().paging(0, 0)
                )
                if count_result.total > 0:
                    price_facets.append(FacetValue(value=label, count=count_result.total))
            except Exception as e:
                logger.warning(f"Failed to count price bucket {label}: {e}")

        facets_data['price_ranges'] = price_facets

        # Rating ranges
        rating_buckets = [
            (4.5, 5.0, "4.5★+"),
            (4.0, 4.5, "4★+"),
            (3.0, 4.0, "3★+"),
            (0, 3.0, "<3★")
        ]

        rating_facets = []
        for min_r, max_r, label in rating_buckets:
            count_query = f"{base_query} @rating:[{min_r} {max_r}]"
            try:
                count_result = self.redis.ft(INDEX_NAME).search(
                    Query(count_query).no_content().paging(0, 0)
                )
                if count_result.total > 0:
                    rating_facets.append(FacetValue(value=label, count=count_result.total))
            except Exception as e:
                logger.warning(f"Failed to count rating bucket {label}: {e}")

        facets_data['rating_ranges'] = rating_facets

        return Facets(**facets_data)

    def autocomplete(self, prefix: str, max_suggestions: int = 10) -> List[str]:
        """
        Autocomplete pour suggestions de recherche

        Args:
            prefix: Début de la requête (ex: "chau")
            max_suggestions: Nombre max de suggestions

        Returns:
            Liste de suggestions
        """
        try:
            # Recherche dans suggestions dictionary
            suggestions = self.redis.ft(INDEX_NAME).sugget(
                f"autocomplete:products",
                prefix,
                fuzzy=True,
                num=max_suggestions
            )

            return [s.string for s in suggestions]

        except redis.ResponseError:
            # Fallback: recherche dans l'index
            query = Query(f"{prefix}*").no_content().paging(0, max_suggestions)
            results = self.redis.ft(INDEX_NAME).search(query)

            return [doc.title for doc in results.docs]

    def invalidate_cache(self, pattern: str = "cache:search:*"):
        """Invalider le cache de recherche"""
        cursor = 0
        invalidated = 0

        while True:
            cursor, keys = self.redis.scan(cursor, match=pattern, count=1000)
            if keys:
                self.redis.delete(*keys)
                invalidated += len(keys)

            if cursor == 0:
                break

        logger.info(f"Invalidated {invalidated} cache entries")
        return invalidated


# ============================================================================
# Indexer Service
# ============================================================================

class ProductIndexer:
    """
    Service d'indexation des produits dans RediSearch
    """

    def __init__(self, redis_config: Dict = None):
        config = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**config)

    def index_product(self, product_data: Dict) -> bool:
        """
        Indexer un seul produit

        Args:
            product_data: Données produit (format JSON)

        Returns:
            True si succès
        """
        product_id = product_data.get('product_id')
        if not product_id:
            logger.error("Product ID missing")
            return False

        key = f"product:{product_id}"

        try:
            # Store as RedisJSON
            self.redis.json().set(key, '$', product_data)

            logger.info(f"Indexed product: {product_id}")
            return True

        except redis.RedisError as e:
            logger.error(f"Failed to index product {product_id}: {e}")
            return False

    def bulk_index(self, products: List[Dict], batch_size: int = 1000) -> Tuple[int, int]:
        """
        Indexer plusieurs produits en batch

        Returns:
            (success_count, failure_count)
        """
        success_count = 0
        failure_count = 0

        # Process in batches
        for i in range(0, len(products), batch_size):
            batch = products[i:i + batch_size]

            # Pipeline pour performance
            pipe = self.redis.pipeline(transaction=False)

            for product in batch:
                product_id = product.get('product_id')
                if product_id:
                    key = f"product:{product_id}"
                    pipe.json().set(key, '$', product)

            try:
                results = pipe.execute()
                success_count += sum(1 for r in results if r)
                failure_count += sum(1 for r in results if not r)

            except redis.RedisError as e:
                logger.error(f"Batch indexing failed: {e}")
                failure_count += len(batch)

        logger.info(
            f"Bulk indexing completed: {success_count} success, {failure_count} failures"
        )

        return success_count, failure_count

    def delete_product(self, product_id: str) -> bool:
        """Supprimer un produit de l'index"""
        key = f"product:{product_id}"

        try:
            deleted = self.redis.delete(key)
            logger.info(f"Deleted product: {product_id}")
            return deleted > 0

        except redis.RedisError as e:
            logger.error(f"Failed to delete product {product_id}: {e}")
            return False


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    # Initialize services
    search_service = ProductSearchService()

    # Exemple de recherche
    filters = SearchFilters(
        min_price=50.0,
        max_price=200.0,
        brands=["Nike", "Adidas"],
        min_rating=4.0,
        in_stock_only=True,
        free_shipping_only=True
    )

    params = SearchParams(
        query="chaussures running",
        filters=filters,
        sort_by=SortBy.RELEVANCE,
        page=1,
        page_size=20,
        include_facets=True
    )

    # Execute search
    response = search_service.search(params)

    print(f"\n✅ Search Results:")
    print(f"   Query: '{params.query}'")
    print(f"   Total: {response.total_count} products")
    print(f"   Time: {response.search_time_ms:.2f}ms")
    print(f"   From cache: {response.from_cache}")

    print(f"\n📦 Products:")
    for i, product in enumerate(response.results[:5], 1):
        print(f"   {i}. {product.title}")
        print(f"      Brand: {product.brand} | Price: {product.price}€ | Rating: {product.rating}★")

    if response.facets:
        print(f"\n🔍 Facets:")
        print(f"   Brands: {len(response.facets.brands)} options")
        for facet in response.facets.brands[:5]:
            print(f"      - {facet.value} ({facet.count})")

        print(f"   Price ranges:")
        for facet in response.facets.price_ranges:
            print(f"      - {facet.value} ({facet.count})")
```

### 5.2 Code Node.js (Frontend Integration)

```javascript
/**
 * E-commerce Search Client - Node.js
 * Frontend-ready avec debouncing et caching
 */

const Redis = require('ioredis');
const crypto = require('crypto');

// Configuration
const REDIS_CONFIG = {
  host: 'localhost',
  port: 6379,
  retryStrategy: (times) => Math.min(times * 50, 2000),
  maxRetriesPerRequest: 3,
};

const INDEX_NAME = 'idx:products';
const SEARCH_CACHE_TTL = 300;

// ============================================================================
// Search Client
// ============================================================================

class ProductSearchClient {
  constructor(config = REDIS_CONFIG) {
    this.redis = new Redis(config);
  }

  /**
   * Recherche produits
   */
  async search({
    query = '',
    filters = {},
    sortBy = 'relevance',
    page = 1,
    pageSize = 20,
    includeFacets = true,
  }) {
    const startTime = Date.now();

    // Generate cache key
    const cacheKey = this._generateCacheKey({
      query,
      filters,
      sortBy,
      page,
      pageSize,
    });

    // Check cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      const response = JSON.parse(cached);
      response.fromCache = true;
      response.searchTimeMs = Date.now() - startTime;
      return response;
    }

    // Build RediSearch query
    const queryString = this._buildQueryString(query, filters);
    const offset = (page - 1) * pageSize;

    // Execute search
    const args = [
      INDEX_NAME,
      queryString,
      'LIMIT',
      offset,
      pageSize,
    ];

    // Add sorting
    if (sortBy !== 'relevance') {
      const sortField = this._getSortField(sortBy);
      args.push('SORTBY', sortField.field, sortField.order);
    }

    // Return fields
    args.push(
      'RETURN', '10',
      'product_id', 'title', 'brand', 'price', 'rating',
      'review_count', 'images', 'stock_status', 'free_shipping', 'is_bestseller'
    );

    try {
      const result = await this.redis.call('FT.SEARCH', ...args);

      // Parse result
      const totalCount = result[0];
      const results = this._parseSearchResults(result.slice(1));

      // Get facets if requested
      let facets = null;
      if (includeFacets) {
        facets = await this._computeFacets(queryString);
      }

      const response = {
        results,
        totalCount,
        page,
        pageSize,
        totalPages: Math.ceil(totalCount / pageSize),
        facets,
        searchTimeMs: Date.now() - startTime,
        fromCache: false,
      };

      // Cache result
      await this.redis.setex(cacheKey, SEARCH_CACHE_TTL, JSON.stringify(response));

      return response;

    } catch (err) {
      console.error('[Search] Error:', err);
      throw err;
    }
  }

  /**
   * Autocomplete suggestions
   */
  async autocomplete(prefix, maxSuggestions = 10) {
    try {
      // Use RediSearch prefix query
      const query = `${prefix}*`;
      const result = await this.redis.call(
        'FT.SEARCH',
        INDEX_NAME,
        query,
        'LIMIT', 0, maxSuggestions,
        'RETURN', 1, 'title'
      );

      const suggestions = [];
      for (let i = 1; i < result.length; i += 2) {
        const fields = result[i + 1];
        for (let j = 0; j < fields.length; j += 2) {
          if (fields[j] === 'title') {
            suggestions.push(fields[j + 1]);
          }
        }
      }

      return suggestions;

    } catch (err) {
      console.error('[Autocomplete] Error:', err);
      return [];
    }
  }

  // ==========================================================================
  // Helper Methods
  // ==========================================================================

  _generateCacheKey(params) {
    const paramsStr = JSON.stringify(params);
    const hash = crypto.createHash('sha256').update(paramsStr).digest('hex').slice(0, 16);
    return `cache:search:${hash}`;
  }

  _buildQueryString(query, filters) {
    let queryParts = [];

    // Base query
    if (query.trim()) {
      queryParts.push(`(${query})`);
    } else {
      queryParts.push('*');
    }

    // Price filter
    if (filters.minPrice !== undefined || filters.maxPrice !== undefined) {
      const min = filters.minPrice || 0;
      const max = filters.maxPrice || '+inf';
      queryParts.push(`@price:[${min} ${max}]`);
    }

    // Brand filter
    if (filters.brands && filters.brands.length > 0) {
      const brandsQuery = filters.brands.map(b => `{${b}}`).join(' | ');
      queryParts.push(`@brand:{${brandsQuery}}`);
    }

    // Category filter
    if (filters.categories && filters.categories.length > 0) {
      const catsQuery = filters.categories.map(c => `{${c}}`).join(' | ');
      queryParts.push(`@category:{${catsQuery}}`);
    }

    // Stock filter
    if (filters.inStockOnly) {
      queryParts.push('@stock:[1 +inf]');
    }

    // Rating filter
    if (filters.minRating) {
      queryParts.push(`@rating:[${filters.minRating} +inf]`);
    }

    // Free shipping
    if (filters.freeShippingOnly) {
      queryParts.push('@free_shipping:{true}');
    }

    return queryParts.join(' ');
  }

  _getSortField(sortBy) {
    const sortMap = {
      'price_asc': { field: 'price', order: 'ASC' },
      'price_desc': { field: 'price', order: 'DESC' },
      'rating': { field: 'rating', order: 'DESC' },
      'popularity': { field: 'popularity', order: 'DESC' },
    };

    return sortMap[sortBy] || { field: '_score', order: 'DESC' };
  }

  _parseSearchResults(rawResults) {
    const results = [];

    for (let i = 0; i < rawResults.length; i += 2) {
      const key = rawResults[i];
      const fields = rawResults[i + 1];

      const product = { product_id: key.replace('product:', '') };

      for (let j = 0; j < fields.length; j += 2) {
        const fieldName = fields[j];
        const fieldValue = fields[j + 1];

        if (fieldName === 'images') {
          product.imageUrl = JSON.parse(fieldValue)[0] || '';
        } else if (fieldName === 'price' || fieldName === 'rating') {
          product[fieldName] = parseFloat(fieldValue);
        } else if (fieldName === 'review_count') {
          product.reviewCount = parseInt(fieldValue, 10);
        } else if (fieldName === 'free_shipping' || fieldName === 'is_bestseller') {
          product[fieldName] = fieldValue === 'true';
        } else {
          product[fieldName] = fieldValue;
        }
      }

      results.push(product);
    }

    return results;
  }

  async _computeFacets(baseQuery) {
    try {
      // Brand facet
      const brandResult = await this.redis.call(
        'FT.AGGREGATE',
        INDEX_NAME,
        baseQuery,
        'GROUPBY', '1', '@brand',
        'REDUCE', 'COUNT', '0', 'AS', 'count',
        'SORTBY', '2', '@count', 'DESC',
        'LIMIT', '0', '20'
      );

      const brands = this._parseFacetResult(brandResult);

      // Category facet
      const categoryResult = await this.redis.call(
        'FT.AGGREGATE',
        INDEX_NAME,
        baseQuery,
        'GROUPBY', '1', '@category',
        'REDUCE', 'COUNT', '0', 'AS', 'count',
        'SORTBY', '2', '@count', 'DESC',
        'LIMIT', '0', '20'
      );

      const categories = this._parseFacetResult(categoryResult);

      return {
        brands,
        categories,
        priceRanges: [], // Simplified for brevity
        ratingRanges: [],
      };

    } catch (err) {
      console.error('[Facets] Error:', err);
      return null;
    }
  }

  _parseFacetResult(rawResult) {
    const facets = [];

    // Skip count at index 0
    for (let i = 1; i < rawResult.length; i++) {
      const row = rawResult[i];
      facets.push({
        value: row[1],
        count: parseInt(row[3], 10),
      });
    }

    return facets;
  }

  async close() {
    await this.redis.quit();
  }
}

// ============================================================================
// Exemple d'utilisation (Express.js API)
// ============================================================================

const express = require('express');
const app = express();
const searchClient = new ProductSearchClient();

app.get('/api/search', async (req, res) => {
  try {
    const {
      q = '',
      minPrice,
      maxPrice,
      brands,
      sortBy = 'relevance',
      page = 1,
      pageSize = 20,
    } = req.query;

    const filters = {
      minPrice: minPrice ? parseFloat(minPrice) : undefined,
      maxPrice: maxPrice ? parseFloat(maxPrice) : undefined,
      brands: brands ? brands.split(',') : undefined,
      inStockOnly: true,
    };

    const response = await searchClient.search({
      query: q,
      filters,
      sortBy,
      page: parseInt(page, 10),
      pageSize: parseInt(pageSize, 10),
      includeFacets: true,
    });

    res.json(response);

  } catch (err) {
    console.error('Search error:', err);
    res.status(500).json({ error: 'Search failed' });
  }
});

app.get('/api/autocomplete', async (req, res) => {
  try {
    const { q } = req.query;

    if (!q || q.length < 2) {
      return res.json({ suggestions: [] });
    }

    const suggestions = await searchClient.autocomplete(q, 10);
    res.json({ suggestions });

  } catch (err) {
    console.error('Autocomplete error:', err);
    res.status(500).json({ error: 'Autocomplete failed' });
  }
});

// app.listen(3000, () => {
//   console.log('Search API listening on port 3000');
// });

module.exports = ProductSearchClient;
```

---

## 6. Monitoring et métriques

### 6.1 KPIs à surveiller

```yaml
# Dashboard Grafana - Métriques critiques

# 1. Latence de recherche (p50, p95, p99)
search_latency_ms:
  p50: < 5ms
  p95: < 20ms
  p99: < 50ms

# 2. Throughput
searches_per_second:
  normal: 500-1000/s
  peak: 5000/s
  max_capacity: 10000/s

# 3. Cache hit ratio
cache_hit_ratio:
  target: > 60%
  excellent: > 80%

# 4. Index size et mémoire
index_memory_mb:
  current: ~250GB (50M products)
  growth_rate: +5GB/day

# 5. Facet computation time
facet_latency_ms:
  target: < 20ms
  max: < 100ms

# 6. Indexing lag
indexing_lag_seconds:
  target: < 1s
  max: < 5s

# 7. Error rate
search_error_rate:
  target: < 0.1%
  critical: > 1%
```

### 6.2 Alertes critiques

```yaml
# Prometheus Alerting Rules

groups:
  - name: search_alerts
    interval: 30s
    rules:

      # Latence excessive
      - alert: SearchHighLatency
        expr: histogram_quantile(0.95, search_latency_seconds) > 0.050
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Search p95 latency > 50ms"
          description: "{{ $value }}s detected"

      # Cache hit ratio faible
      - alert: SearchLowCacheHitRatio
        expr: rate(search_cache_hits[5m]) / rate(search_total[5m]) < 0.50
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Search cache hit ratio < 50%"

      # Index memory critique
      - alert: SearchIndexMemoryHigh
        expr: redis_memory_used_bytes{index="idx:products"} > 300000000000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Search index memory > 300GB"

      # Indexing lag élevé
      - alert: SearchIndexingLagHigh
        expr: search_indexing_lag_seconds > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Indexing lag > 10 seconds"

      # Facets timeout
      - alert: SearchFacetsTimeout
        expr: rate(search_facets_timeout_total[5m]) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Facets computation timing out"
```

---

## 7. Scalabilité et évolutions

### 7.1 Scaling vertical (Phase 1)

```
Single Redis Instance
├─ 256GB RAM
├─ 32 CPU cores
├─ RediSearch index: 250GB
├─ Capacity: 50M products, 10k searches/sec
└─ Coût: ~$800/mois

Avantages:
✅ Simple à opérer
✅ Latence minimale (pas de sharding)

Limites:
❌ Single point of failure
❌ Limité à RAM d'une machine
```

### 7.2 Scaling horizontal (Phase 2)

```
Redis Cluster (6 shards)
├─ 6 Masters: 128GB RAM each
├─ 6 Replicas: 128GB RAM each
├─ Total capacity: 768GB effective
├─ Hash slots distribution (16384 total)
├─ Capacity: 150M products, 50k searches/sec
└─ Coût: ~$4000/mois

Avantages:
✅ Haute disponibilité (replicas)
✅ Scaling linéaire
✅ Pas de SPOF

Limites:
⚠️ Facets doivent agréger résultats de tous shards
⚠️ Complexité client-side routing
```

**Implémentation facets cross-shard** :

```python
def compute_facets_cluster(query_string):
    """Calculer facets sur Redis Cluster"""
    # Get all master nodes
    cluster_nodes = redis_cluster.cluster_nodes()
    master_nodes = [n for n in cluster_nodes if n['role'] == 'master']

    # Execute aggregation on each shard in parallel
    with ThreadPoolExecutor(max_workers=len(master_nodes)) as executor:
        futures = []
        for node in master_nodes:
            future = executor.submit(
                compute_facets_single_shard,
                node['host'],
                node['port'],
                query_string
            )
            futures.append(future)

        # Merge results
        all_facets = [f.result() for f in futures]
        merged_facets = merge_facet_counts(all_facets)

    return merged_facets
```

### 7.3 Optimisations avancées

#### **Tiering de mémoire (Redis Enterprise)**

```
Hot tier (RAM): Produits populaires (10M)
├─ 95% des recherches
├─ Latence < 5ms
└─ Coût: 100GB × $5/GB = $500/mois

Cold tier (Flash SSD): Produits rares (40M)
├─ 5% des recherches
├─ Latence < 20ms
└─ Coût: 200GB × $0.50/GB = $100/mois

Total: $600/mois vs $1500/mois (all-RAM)
Économie: 60%
```

#### **Sharding intelligent par catégorie**

```python
# Plutôt que hash-based sharding, utiliser category-based
shard_mapping = {
    'Electronics': 'shard1',
    'Fashion': 'shard2',
    'Home': 'shard3',
    # ...
}

# Avantages:
# - Recherches dans une catégorie = 1 seul shard (pas de fan-out)
# - Facets calculés localement
# - Latence divisée par N (nombre de shards)
```

---

## 8. Cas limites et optimisations

### 8.1 Typo tolerance et synonymes

```python
# Configuration typo tolerance
FT.CREATE idx:products
  ...
  SCHEMA
    $.title AS title TEXT PHONETIC dm:fr  # Double Metaphone français

# Recherche "chosures" → trouve "chaussures"
# Recherche "naik" → trouve "nike"
```

```bash
# Synonymes
FT.SYNUPDATE idx:products group1 "running" "course" "jogging" "trail"
FT.SYNUPDATE idx:products group2 "sneakers" "baskets" "chaussures" "tennis"
FT.SYNUPDATE idx:products group3 "noir" "black" "schwarz" "negro"

# Recherche multi-langue automatique
```

### 8.2 Gestion des requêtes vides ("*")

```python
def search_with_defaults(params):
    """Gérer les recherches sans query (browse)"""
    if not params.query or params.query.strip() == "":
        # Fallback: trier par popularité ou nouveautés
        params.sort_by = SortBy.POPULARITY
        params.query = "*"

    return search_service.search(params)
```

### 8.3 Hot queries (cache warming)

```python
# Identifier les 100 requêtes les plus fréquentes
top_queries = [
    "chaussures running",
    "iphone 15",
    "machine à café",
    # ...
]

# Warmer le cache au démarrage / périodiquement
def warm_cache():
    for query in top_queries:
        params = SearchParams(query=query, ...)
        search_service.search(params)  # Cache pour 5 min
```

### 8.4 Pagination profonde

```python
# ❌ Problème: page 1000 = offset 20000 (coûteux)
params = SearchParams(query="shoes", page=1000, page_size=20)

# ✅ Solution: Cursor-based pagination
def search_with_cursor(query, cursor=None, page_size=20):
    # Utiliser SORTBY + FILTER pour pagination
    if cursor:
        # Reprendre après le dernier item
        query += f" @product_id:>{cursor}"

    results = search(query, limit=page_size)
    next_cursor = results[-1].product_id if results else None

    return results, next_cursor
```

---

## 9. Sécurité

### 9.1 Injection de requêtes

```python
# ❌ Vulnérable
def search_unsafe(user_query):
    query = f"@title:{user_query}"
    return redis.ft(INDEX_NAME).search(query)

# Attaque: user_query = "* @price:[0 0]"
# → Recherche tous les produits gratuits !

# ✅ Sécurisé: Escape special characters
def search_safe(user_query):
    # Escape: , . < > { } [ ] " ' : ; ! @ # $ % ^ & * ( ) - + = ~ |
    escaped = re.sub(r'([,.<>{}[\]"\':;!@#$%^&*()\-+=~|])', r'\\\1', user_query)
    query = f"@title:({escaped})"
    return redis.ft(INDEX_NAME).search(query)
```

### 9.2 Rate limiting

```python
from redis import Redis
from time import time

def rate_limit_search(user_id: str, max_requests: int = 100, window: int = 60):
    """
    Rate limit: 100 recherches / minute par utilisateur
    """
    key = f"ratelimit:search:{user_id}"
    current = int(time())

    pipe = redis.pipeline()
    pipe.zadd(key, {current: current})
    pipe.zremrangebyscore(key, 0, current - window)
    pipe.zcard(key)
    pipe.expire(key, window)

    _, _, count, _ = pipe.execute()

    if count > max_requests:
        raise Exception(f"Rate limit exceeded: {count}/{max_requests} requests")
```

---

## 10. Conclusion

### Points clés à retenir

- ✅ **RediSearch = Latence sub-50ms** pour recherche full-text complexe
- ✅ **Facets natives** via FT.AGGREGATE (pas besoin de requêtes séparées)
- ✅ **Indexation temps réel** (< 1s lag entre write et searchable)
- ✅ **Coût optimisé** : 80% moins cher qu'Elasticsearch pour même performance
- ✅ **Autocomplete natif** avec SUGGET
- ✅ **Scaling horizontal** via Redis Cluster (avec trade-offs)
- ✅ **Typo tolerance** et synonymes out-of-the-box

### Quand NE PAS utiliser RediSearch

- ❌ **Requêtes analytiques complexes** : Elasticsearch + Kibana plus adapté
- ❌ **Machine Learning intégré** : Elasticsearch ML plus mature
- ❌ **Graph queries** : Neo4j ou Elasticsearch plus appropriés
- ❌ **Audit trail complet** : Base relationnelle (PostgreSQL) nécessaire
- ❌ **Budget RAM limité** : Solution on-disk (Elasticsearch, Solr) plus viable

### Comparaison finale

| Critère | PostgreSQL FTS | Elasticsearch | Algolia | RediSearch |
|---------|----------------|---------------|---------|------------|
| Latence (p95) | 500-2000ms | 20-100ms | 5-10ms | **3-8ms** |
| Coût (mensuel) | $8,000 | $2,000 | $3.6M | **$500-1000** |
| Indexing lag | 8 heures | 1 seconde | < 100ms | **< 1s** |
| Facets | Manuel | Natif | Natif | **Natif** |
| Ops complexity | Faible | Élevée | Nulle (SaaS) | **Faible** |

### Prochaines lectures

- [Cas #3 : Leaderboard de jeu vidéo](./03-cas-leaderboard-jeu-video.md) → Sorted Sets à grande échelle
- [RediSearch Advanced](../03-structures-donnees-etendues/03-redisearch-indexation-fulltext.md) → Fonctionnalités avancées
- [Redis Cluster Sharding](../11-architecture-distribuee-scaling/03-distribution-donnees-hash-slots.md) → Scaling horizontal

---

**📚 Ressources complémentaires** :
- [RediSearch Documentation](https://redis.io/docs/stack/search/)
- [RediSearch Query Syntax](https://redis.io/docs/stack/search/reference/query_syntax/)
- [E-commerce Search Best Practices](https://www.algolia.com/doc/guides/solutions/ecommerce/)

⏭️ [Cas #3 : Leaderboard de jeu vidéo temps réel](/16-etudes-cas-patterns-reels/03-cas-leaderboard-jeu-video.md)
