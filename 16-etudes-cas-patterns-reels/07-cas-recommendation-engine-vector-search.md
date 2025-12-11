🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #7 : Recommendation Engine avec Vector Search

## Vue d'ensemble

**Niveau** : ⭐⭐⭐ Avancé
**Complexité technique** : Élevée
**Impact production** : Critique (engagement et revenue)
**Technologies** : Redis Stack (RediSearch + Vector Similarity) + ML

---

## 1. Contexte et problématique

### Scénario business

Une plateforme e-commerce (type Amazon, Alibaba) ou streaming (Netflix, Spotify) avec système de recommandations personnalisées :

**Chiffres clés** :
- 10 millions de produits/contenus
- 50 millions d'utilisateurs actifs
- 100 millions d'interactions par jour
- 500 000 requêtes de recommandations par seconde (peak)
- SLA : Latence < 50ms (p95)
- Conversion baseline : 3.5% → Objectif : 5.5%
- Revenue impact : +$50M/an avec meilleures recommendations

**Besoins métier critiques** :

1. **Recommandations personnalisées en temps réel**
   - "Produits similaires" (content-based)
   - "Clients qui ont acheté X ont aussi acheté Y" (collaborative)
   - "Recommandé pour vous" (hybrid)
   - "Continuer à regarder" (session-based)

2. **Recherche sémantique**
   - "robe rouge élégante" → Trouver produits même sans exact match
   - Comprendre intent : "cadeau anniversaire maman"
   - Recherche par image (reverse image search)

3. **Contraintes de performance**
   - Latence < 50ms pour ne pas impacter UX
   - Scalabilité : 500k requests/sec
   - Freshness : Nouveaux produits indexés en < 1 min
   - Cold start : Recommandations sans historique

4. **Business rules**
   - Filtres : prix, disponibilité, catégorie, localisation
   - Boost : promotions, nouveautés, marge haute
   - Diversité : Pas 10 recommandations identiques
   - A/B testing : Multiple strategies

### Problèmes à résoudre

#### 1. **Vector Similarity Search à grande échelle**

```
Problème : Trouver les K produits les plus similaires parmi 10M

Approche naïve (brute force) :
for each product in catalog (10M):
    similarity = cosine_similarity(query_vector, product_vector)

Complexité : O(N × D) où N = 10M produits, D = 768 dimensions
→ 10M × 768 = 7.68 milliards d'opérations
→ Latence : 10-30 secondes (inacceptable!)

Solution : Approximate Nearest Neighbors (ANN)
→ HNSW, IVF, LSH
→ Latence : < 50ms avec 95-99% accuracy
```

#### 2. **Embeddings generation**

```
Problème : Comment transformer produits en vecteurs ?

Options :
1. Pre-trained models (sentence-transformers)
   - Avantage : Prêt à l'emploi
   - Inconvénient : Générique, pas optimisé pour domaine

2. Fine-tuned models (domain-specific)
   - Avantage : Meilleure qualité
   - Inconvénient : Coût d'entraînement

3. Multimodal (texte + images)
   - Avantage : Richesse sémantique
   - Inconvénient : Complexité

Vector dimensions trade-off :
- 384 dims : Rapide mais moins précis
- 768 dims : Standard (BERT)
- 1536 dims : Très précis mais lent
```

#### 3. **Hybrid Search (Vector + Filters)**

```
Besoin : "Produits similaires à X, prix < $100, en stock, catégorie Électronique"

Approche naïve :
1. Vector search → Top 1000 produits similaires
2. Filter → Garder seulement ceux qui matchent
3. Return top 10

Problème : Si seulement 5/1000 matchent les filtres → Pas assez de résultats

Solution : Pre-filtering ou Post-filtering avec sur-sampling
```

#### 4. **Cold Start Problem**

```
Problème : Nouveaux utilisateurs sans historique

Solutions :
1. Content-based : Recommander selon attributs (catégorie, prix)
2. Popularité : Trending items
3. Onboarding quiz : Demander préférences explicites
4. Contextual : Géolocalisation, device, heure
```

---

## 2. Analyse des alternatives

### Option 1 : PostgreSQL avec pgvector

```sql
-- Extension pgvector
CREATE EXTENSION vector;

-- Table produits avec embeddings
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    embedding vector(768)
);

-- Index pour Vector Search
CREATE INDEX ON products USING ivfflat (embedding vector_cosine_ops);

-- Query: Trouver produits similaires
SELECT id, name, embedding <=> '[0.1, 0.2, ...]' AS distance
FROM products
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 10;
```

**Avantages** :
- ✅ Intégré à PostgreSQL (pas de nouvelle infra)
- ✅ ACID compliance
- ✅ Filters avec WHERE clause

**Inconvénients** :
- ❌ Latence : 100-500ms pour 1M vectors
- ❌ Index ivfflat moins performant que HNSW
- ❌ Scaling horizontal difficile
- ❌ Pas optimisé pour high-throughput reads

**Benchmark** (PostgreSQL 15 + pgvector, 1M vectors 768-dim) :
```
Query                     Latency
--------------------------------
Top 10 similar (no filter)  200ms
Top 10 + WHERE filter       350ms
Throughput                  100 req/s per core
```

**Verdict** : ⚠️ **Acceptable pour < 1M vectors**, sous-optimal pour échelle.

---

### Option 2 : Pinecone (SaaS)

```python
import pinecone

pinecone.init(api_key="...")
index = pinecone.Index("products")

# Upsert vectors
index.upsert(vectors=[
    ("prod1", [0.1, 0.2, ...], {"category": "electronics", "price": 99}),
    ("prod2", [0.3, 0.4, ...], {"category": "clothing", "price": 49})
])

# Query
results = index.query(
    vector=[0.1, 0.2, ...],
    top_k=10,
    filter={"category": {"$eq": "electronics"}, "price": {"$lte": 100}}
)
```

**Avantages** :
- ✅ Managed service (zero ops)
- ✅ Latence : < 50ms
- ✅ Scaling automatique
- ✅ HNSW algorithm

**Inconvénients** :
- ❌ Coût : $70-1000/mois (10M vectors)
- ❌ Vendor lock-in
- ❌ Data egress costs
- ❌ Cold start latency (serverless)

**Verdict** : ⚠️ **Excellent mais coûteux** à grande échelle.

---

### Option 3 : RediSearch Vector Similarity ✅

```bash
# Create index with vector field
FT.CREATE products_idx
    ON JSON
    PREFIX 1 product:
    SCHEMA
        name TEXT
        category TAG
        price NUMERIC
        embedding VECTOR HNSW 6
            TYPE FLOAT32
            DIM 768
            DISTANCE_METRIC COSINE

# Add product
JSON.SET product:1 $ '{
    "name": "Laptop",
    "category": "electronics",
    "price": 999,
    "embedding": [0.1, 0.2, ...]
}'

# Vector search with filters
FT.SEARCH products_idx
    "@category:{electronics} @price:[0 1000]"
    RETURN 3 name category price
    SORTBY __embedding_score
    DIALECT 2
    PARAMS 2 embedding_vector "0.1,0.2,..."
    LIMIT 0 10
```

**Avantages** :
- ✅ Latence : **< 10ms** pour 1M vectors
- ✅ HNSW algorithm (state-of-the-art)
- ✅ Hybrid search natif (vector + filters)
- ✅ In-memory : Ultra-rapide
- ✅ Throughput : 100k req/s per instance
- ✅ Self-hosted : Contrôle complet

**Inconvénients** :
- ⚠️ In-memory : Limite de RAM (mais tiering possible)
- ⚠️ Pas de GPU acceleration (CPU only)

**Benchmark** (Redis Stack 7.2, 1M vectors 768-dim) :
```
Query                           Latency
----------------------------------------
Top 10 similar (HNSW)          8ms
Top 10 + filters               12ms
Throughput                     50k req/s
Index build time               2 min
Memory                         ~4GB (1M × 768 × 4 bytes)
```

**Trade-off assumé** :
- ➕ Performance optimale (< 10ms)
- ➕ Coût ÷ 10 vs Pinecone
- ➕ Full control
- ➖ RAM required (mais acceptable avec tiering)

**Verdict** : ✅ **Solution optimale** pour vector search haute performance.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                   Client Applications                        │
│  - Web App                                                   │
│  - Mobile App                                                │
│  - Recommendation Widgets                                    │
└──────────┬───────────────────────────────────────────────────┘
           │ HTTPS
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Recommendation API                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Endpoints:                                         │    │
│  │  - POST /recommend/similar-items                    │    │
│  │  - POST /recommend/personalized                     │    │
│  │  - POST /search/semantic                            │    │
│  │  - GET /recommend/trending                          │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│           Embedding Service (Inference)                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Model: sentence-transformers/all-MiniLM-L6-v2      │    │
│  │  - Input: Text query                                │    │
│  │  - Output: Vector[384]                              │    │
│  │  - Latency: ~20ms CPU / ~5ms GPU                    │    │
│  │  - Batch size: 32 for throughput                    │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Redis Stack (Vector Store)                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  RediSearch with Vector Similarity (HNSW)           │    │
│  │                                                     │    │
│  │  Index: products_idx                                │    │
│  │  ├─ 10M products                                    │    │
│  │  ├─ Vector field: embedding (768-dim)               │    │
│  │  ├─ Metadata: name, category, price, rating, etc.   │    │
│  │  └─ HNSW params: M=16, EF=200                       │    │
│  │                                                     │    │
│  │  Memory: ~8GB (10M × 768 × 4 bytes + overhead)      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Master-Replica Setup                                       │
│  ┌──────────┐  ┌──────────┐                                 │
│  │ Master   │──│ Replica  │                                 │
│  │ (Write)  │  │ (Read)   │                                 │
│  └──────────┘  └──────────┘                                 │
└──────────┬──────────────────────────────────────────────────┘
           │
           │ (Optional) Offline Pipeline
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Embedding Generation Pipeline                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Batch processing:                                  │    │
│  │  1. Fetch new/updated products from DB              │    │
│  │  2. Generate embeddings (batch of 1000)             │    │
│  │  3. Update Redis vector index                       │    │
│  │  4. Run every 5 minutes                             │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL (Source of Truth)                   │
│  - Products metadata                                        │
│  - User interactions (views, clicks, purchases)             │
│  - A/B test configs                                         │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Flux de recommandation

#### **Similar Items (Content-Based)**

```
1. User clicks on product
   Product ID: prod_12345

2. Fetch product embedding from Redis
   embedding = redis.json.get("product:12345", "$.embedding")

3. Vector search for similar products
   FT.SEARCH products_idx
       "*"
       RETURN 3 name price rating
       SORTBY __embedding_score
       PARAMS 2 embedding_vector {embedding}
       LIMIT 0 10

4. Post-processing
   ├─ Remove original product
   ├─ Apply business rules (in stock, price range)
   ├─ Diversify (max 2 per category)
   └─ Add metadata (images, reviews)

5. Return recommendations
   [
       {"id": "prod_456", "score": 0.92, "name": "..."},
       {"id": "prod_789", "score": 0.89, "name": "..."},
       ...
   ]

Total latency: ~30ms
```

#### **Personalized Recommendations (Hybrid)**

```
1. User profile
   user_id: user_abc123
   history: [prod_1, prod_2, prod_3, ...]
   preferences: {"category": "electronics", "price_max": 500}

2. Generate user embedding (average of viewed products)
   embeddings = [redis.json.get(f"product:{id}", "$.embedding") for id in history]
   user_embedding = np.mean(embeddings, axis=0)

3. Hybrid search (vector + filters)
   FT.SEARCH products_idx
       "@category:{electronics} @price:[0 500]"
       RETURN 3 name price
       SORTBY __embedding_score
       PARAMS 2 embedding_vector {user_embedding}
       LIMIT 0 20

4. Re-ranking with business logic
   ├─ Boost: New products (+10%), Promotions (+20%)
   ├─ Penalize: Already viewed (-50%), Low rating (-30%)
   ├─ Diversity: Spread across subcategories
   └─ Exploration: 20% random (avoid filter bubble)

5. Return top 10
   Latency: ~50ms
```

### 3.3 Décisions architecturales clés

#### **Choix 1 : HNSW vs Flat Index**

**HNSW (Hierarchical Navigable Small World)** - Choisi :
```
Complexité :
- Build: O(N × log N × M)
- Search: O(log N × EF)

Paramètres :
- M: Nombre de connections par node (16-48)
  → M=16 : Balance speed/accuracy
- EF_construction: Taille de la candidate list (100-200)
  → Plus élevé = meilleure accuracy mais plus lent
- EF_runtime: Candidate list pour search (10-500)
  → Ajustable par query

Performance :
- 1M vectors : ~10ms latency
- 10M vectors : ~30ms latency
- Recall@10 : 95-99%
```

**Flat (Brute Force)** :
```
Complexité : O(N)
Performance : 100-1000ms pour 1M vectors
Recall : 100% (exact)

Usage : Seulement si < 100k vectors
```

**Trade-off assumé** :
- ➕ Latence × 100 meilleure (10ms vs 1000ms)
- ➕ Scalable à des dizaines de millions
- ➖ Approximation (95-99% recall vs 100%)
- ➖ Memory overhead (HNSW graph)

---

#### **Choix 2 : Vector Dimensions (384 vs 768)**

**Benchmark** :

```python
Model                        Dimensions  Latency  Accuracy
-----------------------------------------------------------
all-MiniLM-L6-v2            384         5ms      Good
all-mpnet-base-v2           768         20ms     Better
text-embedding-ada-002      1536        50ms     Best (OpenAI)

Memory usage (1M vectors):
- 384 dims: 1.5 GB (1M × 384 × 4 bytes)
- 768 dims: 3 GB
- 1536 dims: 6 GB
```

**Choix : 384 dimensions (all-MiniLM-L6-v2)**

**Trade-off assumé** :
- ➕ Latence ÷ 4 (5ms vs 20ms)
- ➕ Memory ÷ 2
- ➕ Throughput × 4
- ➖ Accuracy légèrement inférieure (mais acceptable)

---

#### **Choix 3 : Pre-filtering vs Post-filtering**

**Pre-filtering** (appliqué avant vector search) :
```bash
# Filters en query string
FT.SEARCH products_idx
    "@category:{electronics} @price:[0 100] @in_stock:{true}"
    SORTBY __embedding_score
    PARAMS 2 embedding_vector "..."
```

**Avantages** :
- ✅ Pas de compute gaspillé sur items non-éligibles
- ✅ Résultats garantis de matcher filters

**Inconvénients** :
- ⚠️ Si filters trop restrictifs → Peu de résultats

**Post-filtering** (appliqué après vector search) :
```python
# Fetch top 1000
results = vector_search(query_vector, top_k=1000)

# Filter
filtered = [r for r in results if r.price <= 100 and r.in_stock]

# Return top 10
return filtered[:10]
```

**Avantages** :
- ✅ Toujours des résultats (relaxer filters si nécessaire)

**Inconvénients** :
- ⚠️ Compute gaspillé sur items non-éligibles

**Choix : Pre-filtering avec over-fetching**

```python
# Fetch more than needed (top_k = 100 au lieu de 10)
# Pour compenser le filtering
FT.SEARCH products_idx
    "@category:{electronics}"
    LIMIT 0 100  # Over-fetch
```

---

## 4. Implémentation technique

### 4.1 Code Python (Production-Ready)

```python
"""
Recommendation Engine avec RediSearch Vector Similarity
Implémentation production-ready
"""

import time
import numpy as np
from typing import List, Dict, Any, Optional, Tuple
from dataclasses import dataclass
import logging

import redis
from redis.commands.search.field import TextField, NumericField, TagField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import Query

from sentence_transformers import SentenceTransformer

# Configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'db': 0,
    'decode_responses': False
}

# Model configuration
MODEL_NAME = "sentence-transformers/all-MiniLM-L6-v2"
VECTOR_DIM = 384

# HNSW parameters
HNSW_M = 16
HNSW_EF_CONSTRUCTION = 200
HNSW_EF_RUNTIME = 10


# ============================================================================
# Data Classes
# ============================================================================

@dataclass
class Product:
    """Produit avec embedding"""
    id: str
    name: str
    category: str
    price: float
    rating: float
    description: str
    embedding: Optional[List[float]] = None

    def to_redis_json(self) -> Dict:
        """Convertir en format Redis JSON"""
        return {
            "id": self.id,
            "name": self.name,
            "category": self.category,
            "price": self.price,
            "rating": self.rating,
            "description": self.description,
            "embedding": self.embedding if self.embedding else []
        }


@dataclass
class Recommendation:
    """Résultat de recommandation"""
    product_id: str
    score: float
    name: str
    category: str
    price: float


# ============================================================================
# Embedding Service
# ============================================================================

class EmbeddingService:
    """
    Service de génération d'embeddings

    Utilise sentence-transformers pour encoder texte en vecteurs
    """

    def __init__(self, model_name: str = MODEL_NAME):
        logger.info(f"Loading model: {model_name}")
        self.model = SentenceTransformer(model_name)
        self.dimension = self.model.get_sentence_embedding_dimension()
        logger.info(f"Model loaded, dimension: {self.dimension}")

    def encode(self, text: str) -> List[float]:
        """Encoder texte en vecteur"""
        embedding = self.model.encode(text, convert_to_numpy=True)
        return embedding.tolist()

    def encode_batch(self, texts: List[str], batch_size: int = 32) -> List[List[float]]:
        """Encoder batch de textes (optimisé)"""
        embeddings = self.model.encode(
            texts,
            batch_size=batch_size,
            show_progress_bar=False,
            convert_to_numpy=True
        )
        return embeddings.tolist()

    def generate_product_embedding(self, product: Product) -> List[float]:
        """Générer embedding pour un produit"""
        # Combiner plusieurs champs avec pondération
        text = f"{product.name} {product.name} {product.category} {product.description}"
        return self.encode(text)


# ============================================================================
# Recommendation Engine
# ============================================================================

class RecommendationEngine:
    """
    Moteur de recommandations avec RediSearch

    Features:
    - Vector similarity search (HNSW)
    - Hybrid search (vector + filters)
    - Content-based recommendations
    - Personalized recommendations
    """

    def __init__(
        self,
        redis_config: Dict = None,
        embedding_service: Optional[EmbeddingService] = None
    ):
        # Redis connexion
        redis_conf = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**redis_conf)

        # Embedding service
        self.embedding_service = embedding_service or EmbeddingService()

        # Index name
        self.index_name = "products_idx"

        # Initialize index
        self._create_index()

        logger.info("RecommendationEngine initialized")

    # ========================================================================
    # Index Management
    # ========================================================================

    def _create_index(self):
        """Créer index RediSearch avec champ vectoriel"""
        try:
            # Check if index exists
            self.redis.ft(self.index_name).info()
            logger.info(f"Index {self.index_name} already exists")
            return
        except:
            pass

        # Create index schema
        schema = (
            TextField("name", weight=2.0),
            TextField("description"),
            TagField("category"),
            NumericField("price", sortable=True),
            NumericField("rating", sortable=True),
            VectorField(
                "embedding",
                "HNSW",
                {
                    "TYPE": "FLOAT32",
                    "DIM": VECTOR_DIM,
                    "DISTANCE_METRIC": "COSINE",
                    "INITIAL_CAP": 10000,
                    "M": HNSW_M,
                    "EF_CONSTRUCTION": HNSW_EF_CONSTRUCTION
                }
            )
        )

        # Create index
        self.redis.ft(self.index_name).create_index(
            schema,
            definition=IndexDefinition(
                prefix=["product:"],
                index_type=IndexType.JSON
            )
        )

        logger.info(f"Index {self.index_name} created successfully")

    # ========================================================================
    # Product Management
    # ========================================================================

    def add_product(self, product: Product, generate_embedding: bool = True):
        """Ajouter un produit à l'index"""
        # Generate embedding if needed
        if generate_embedding and not product.embedding:
            product.embedding = self.embedding_service.generate_product_embedding(product)

        # Store in Redis as JSON
        key = f"product:{product.id}"
        self.redis.json().set(key, "$", product.to_redis_json())

        logger.debug(f"Product added: {product.id}")

    def add_products_batch(self, products: List[Product], batch_size: int = 100):
        """Ajouter batch de produits (optimisé)"""
        logger.info(f"Adding {len(products)} products...")

        # Generate embeddings in batch
        texts = []
        for product in products:
            if not product.embedding:
                text = f"{product.name} {product.name} {product.category} {product.description}"
                texts.append(text)

        if texts:
            logger.info("Generating embeddings...")
            embeddings = self.embedding_service.encode_batch(texts, batch_size)

            # Assign embeddings
            idx = 0
            for product in products:
                if not product.embedding:
                    product.embedding = embeddings[idx]
                    idx += 1

        # Store in Redis
        logger.info("Storing in Redis...")
        pipe = self.redis.pipeline()

        for product in products:
            key = f"product:{product.id}"
            pipe.json().set(key, "$", product.to_redis_json())

        pipe.execute()
        logger.info(f"Batch added: {len(products)} products")

    # ========================================================================
    # Vector Search
    # ========================================================================

    def _vector_to_bytes(self, vector: List[float]) -> bytes:
        """Convertir vecteur en bytes pour Redis"""
        return np.array(vector, dtype=np.float32).tobytes()

    def similar_products(
        self,
        product_id: str,
        top_k: int = 10,
        filters: Optional[Dict] = None
    ) -> List[Recommendation]:
        """
        Trouver produits similaires (content-based)

        Args:
            product_id: ID du produit de référence
            top_k: Nombre de recommandations
            filters: Filtres optionnels {"category": "electronics", "price_max": 100}

        Returns:
            Liste de Recommendation
        """
        # Fetch product embedding
        key = f"product:{product_id}"
        product_data = self.redis.json().get(key)

        if not product_data or "embedding" not in product_data:
            logger.warning(f"Product not found or no embedding: {product_id}")
            return []

        embedding = product_data["embedding"]

        # Search similar products
        return self.search_by_vector(embedding, top_k + 1, filters, exclude_id=product_id)

    def search_by_vector(
        self,
        query_vector: List[float],
        top_k: int = 10,
        filters: Optional[Dict] = None,
        exclude_id: Optional[str] = None
    ) -> List[Recommendation]:
        """
        Recherche vectorielle avec filtres optionnels

        Args:
            query_vector: Vecteur de requête
            top_k: Nombre de résultats
            filters: Filtres {"category": str, "price_min": float, "price_max": float}
            exclude_id: ID à exclure des résultats

        Returns:
            Liste de Recommendation
        """
        # Build filter query
        filter_parts = []

        if filters:
            if "category" in filters:
                filter_parts.append(f"@category:{{{filters['category']}}}")

            if "price_min" in filters and "price_max" in filters:
                filter_parts.append(
                    f"@price:[{filters['price_min']} {filters['price_max']}]"
                )
            elif "price_max" in filters:
                filter_parts.append(f"@price:[0 {filters['price_max']}]")

        base_query = " ".join(filter_parts) if filter_parts else "*"

        # Vector search query
        query = (
            Query(base_query)
            .return_fields("id", "name", "category", "price", "rating")
            .sort_by("__embedding_score")
            .paging(0, top_k * 2)  # Over-fetch for filtering
            .dialect(2)
        )

        # Execute search
        vector_bytes = self._vector_to_bytes(query_vector)

        try:
            results = self.redis.ft(self.index_name).search(
                query,
                query_params={
                    "embedding_vector": vector_bytes,
                    "EF_RUNTIME": HNSW_EF_RUNTIME
                }
            )

            # Parse results
            recommendations = []

            for doc in results.docs:
                product_id = doc.id.replace("product:", "")

                # Exclude if needed
                if exclude_id and product_id == exclude_id:
                    continue

                # Parse score (distance → similarity)
                score = 1.0 - float(doc.__embedding_score)

                recommendations.append(Recommendation(
                    product_id=product_id,
                    score=score,
                    name=doc.name,
                    category=doc.category,
                    price=float(doc.price)
                ))

                if len(recommendations) >= top_k:
                    break

            return recommendations

        except Exception as e:
            logger.error(f"Vector search error: {e}")
            return []

    def search_by_text(
        self,
        query_text: str,
        top_k: int = 10,
        filters: Optional[Dict] = None
    ) -> List[Recommendation]:
        """
        Recherche sémantique par texte

        Args:
            query_text: Texte de recherche (ex: "red elegant dress")
            top_k: Nombre de résultats
            filters: Filtres optionnels

        Returns:
            Liste de Recommendation
        """
        # Generate embedding from text
        query_vector = self.embedding_service.encode(query_text)

        # Vector search
        return self.search_by_vector(query_vector, top_k, filters)

    # ========================================================================
    # Personalized Recommendations
    # ========================================================================

    def personalized_recommendations(
        self,
        user_history: List[str],
        top_k: int = 10,
        filters: Optional[Dict] = None
    ) -> List[Recommendation]:
        """
        Recommandations personnalisées basées sur l'historique

        Args:
            user_history: Liste des product_ids vus/achetés
            top_k: Nombre de recommandations
            filters: Filtres optionnels

        Returns:
            Liste de Recommendation
        """
        if not user_history:
            logger.warning("Empty user history")
            return []

        # Fetch embeddings des produits de l'historique
        embeddings = []

        for product_id in user_history:
            key = f"product:{product_id}"
            product_data = self.redis.json().get(key)

            if product_data and "embedding" in product_data:
                embeddings.append(product_data["embedding"])

        if not embeddings:
            logger.warning("No embeddings found in user history")
            return []

        # Average des embeddings = user profile
        user_embedding = np.mean(embeddings, axis=0).tolist()

        # Search avec exclusion des produits déjà vus
        recommendations = []
        seen = set(user_history)

        # Fetch more to account for exclusions
        candidates = self.search_by_vector(user_embedding, top_k * 3, filters)

        for rec in candidates:
            if rec.product_id not in seen:
                recommendations.append(rec)

                if len(recommendations) >= top_k:
                    break

        return recommendations

    # ========================================================================
    # Hybrid & Re-ranking
    # ========================================================================

    def rerank_with_business_logic(
        self,
        recommendations: List[Recommendation],
        boosts: Optional[Dict[str, float]] = None
    ) -> List[Recommendation]:
        """
        Re-ranker avec règles métier

        Args:
            recommendations: Recommandations initiales
            boosts: Boosts optionnels {"category:electronics": 1.2, "rating:5": 1.1}

        Returns:
            Recommandations re-rankées
        """
        if not boosts:
            return recommendations

        # Apply boosts
        for rec in recommendations:
            boost_factor = 1.0

            # Category boost
            category_boost_key = f"category:{rec.category}"
            if category_boost_key in boosts:
                boost_factor *= boosts[category_boost_key]

            # Apply
            rec.score *= boost_factor

        # Re-sort
        recommendations.sort(key=lambda x: x.score, reverse=True)

        return recommendations

    # ========================================================================
    # Analytics
    # ========================================================================

    def get_index_stats(self) -> Dict:
        """Statistiques de l'index"""
        try:
            info = self.redis.ft(self.index_name).info()

            return {
                "num_docs": info.get("num_docs", 0),
                "num_terms": info.get("num_terms", 0),
                "num_records": info.get("num_records", 0),
                "inverted_sz_mb": info.get("inverted_sz_mb", 0),
                "vector_index_sz_mb": info.get("vector_index_sz_mb", 0)
            }
        except Exception as e:
            logger.error(f"Failed to get index stats: {e}")
            return {}


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    # Initialize
    engine = RecommendationEngine()

    print("\n🎯 Recommendation Engine Demo\n")

    # Créer produits exemples
    products = [
        Product(
            id="prod_1",
            name="Laptop Dell XPS 15",
            category="electronics",
            price=1299.99,
            rating=4.5,
            description="High-performance laptop for professionals"
        ),
        Product(
            id="prod_2",
            name="MacBook Pro 16",
            category="electronics",
            price=2499.99,
            rating=4.8,
            description="Premium laptop with M2 chip"
        ),
        Product(
            id="prod_3",
            name="Wireless Mouse",
            category="electronics",
            price=29.99,
            rating=4.3,
            description="Ergonomic wireless mouse"
        ),
        Product(
            id="prod_4",
            name="Red Elegant Dress",
            category="clothing",
            price=89.99,
            rating=4.6,
            description="Beautiful red evening dress"
        ),
        Product(
            id="prod_5",
            name="Blue Summer Dress",
            category="clothing",
            price=59.99,
            rating=4.4,
            description="Light summer dress perfect for beach"
        ),
    ]

    # Add products
    print("📦 Adding products...")
    engine.add_products_batch(products)
    time.sleep(2)  # Wait for indexing

    # Similar products
    print("\n🔍 Similar Products to 'Laptop Dell XPS 15':")
    similar = engine.similar_products("prod_1", top_k=3)

    for i, rec in enumerate(similar, 1):
        print(f"   {i}. {rec.name} (score: {rec.score:.3f}, ${rec.price})")

    # Semantic search
    print("\n🔎 Semantic Search: 'elegant dress for special occasion'")
    search_results = engine.search_by_text(
        "elegant dress for special occasion",
        top_k=3,
        filters={"category": "clothing"}
    )

    for i, rec in enumerate(search_results, 1):
        print(f"   {i}. {rec.name} (score: {rec.score:.3f})")

    # Personalized recommendations
    print("\n👤 Personalized Recommendations (history: laptop + mouse):")
    personalized = engine.personalized_recommendations(
        user_history=["prod_1", "prod_3"],
        top_k=3
    )

    for i, rec in enumerate(personalized, 1):
        print(f"   {i}. {rec.name} (score: {rec.score:.3f})")

    # Index stats
    print("\n📊 Index Statistics:")
    stats = engine.get_index_stats()
    print(f"   Documents: {stats.get('num_docs', 0)}")
    print(f"   Vector index size: {stats.get('vector_index_sz_mb', 0):.2f} MB")
```

---

## 5. Monitoring et métriques

### 5.1 KPIs critiques

```yaml
# 1. Recommendation quality
click_through_rate:
  baseline: 3.5%
  target: 5.5%

conversion_rate:
  baseline: 1.2%
  target: 2.0%

# 2. Performance
recommendation_latency_ms:
  p50: < 20ms
  p95: < 50ms
  p99: < 100ms

# 3. Vector search accuracy
recall_at_10:
  target: > 95%

# 4. Index size
vector_index_memory_gb:
  10M vectors (384-dim): ~2GB
  10M vectors (768-dim): ~4GB
```

---

## 6. Conclusion

### Points clés

- ✅ **RediSearch HNSW** : Latence < 10ms pour vector search
- ✅ **Hybrid search** : Vector + filters natif
- ✅ **Embeddings 384-dim** : Balance performance/accuracy
- ✅ **Batch processing** : Génération embeddings optimisée
- ✅ **Pre-filtering** : Résultats garantis de matcher contraintes

### Prochaines lectures

- [RediSearch Documentation](https://redis.io/docs/stack/search/)
- [HNSW Algorithm](https://arxiv.org/abs/1603.09320)

---

**📚 Ressources** :
- [Vector Similarity in Redis](https://redis.io/docs/stack/search/reference/vectors/)
- [Sentence Transformers](https://www.sbert.net/)

⏭️ [Cas #8 : IoT et Time-Series avec RedisTimeSeries](/16-etudes-cas-patterns-reels/08-cas-iot-timeseries.md)
