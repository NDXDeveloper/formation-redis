🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.5 RediSearch : Vector Search et cas d'usage IA/RAG

## Introduction

L'essor des modèles de langage (GPT-4, Claude, LLaMA) et de l'IA générative a créé un besoin crucial : **rechercher des informations par similarité sémantique** plutôt que par correspondance exacte de mots-clés.

**Vector Search** (recherche vectorielle) permet de :
- 🔍 **Recherche sémantique** : "Trouver des documents similaires en signification"
- 🤖 **RAG (Retrieval Augmented Generation)** : Fournir du contexte aux LLMs
- 🎯 **Recommendation** : "Produits similaires à celui-ci"
- 🖼️ **Recherche d'images** : Trouver des images visuellement similaires
- 🔬 **Détection de similarité** : Détecter du contenu dupliqué ou similaire

RediSearch intègre nativement le **Vector Search** avec des performances exceptionnelles : **< 5ms de latence** pour des recherches sur des millions de vecteurs.

---

## Concepts fondamentaux

### Qu'est-ce qu'un embedding ?

Un **embedding** (ou vecteur d'embedding) est une représentation numérique d'un texte, d'une image ou de toute donnée, sous forme de tableau de nombres (floats).

**Exemple** :

```python
# Texte original
text = "Redis is an in-memory database"

# Embedding (simplifié, vraie dimension : 768, 1536, etc.)
embedding = [0.023, -0.145, 0.892, 0.234, -0.567, ...]  # 384 dimensions
```

**Propriété clé** : Les textes similaires en signification ont des embeddings proches dans l'espace vectoriel.

```python
# Embeddings générés par un modèle (ex: OpenAI text-embedding-3-small)
embedding_1 = generate_embedding("Redis is fast")
# [0.02, -0.14, 0.89, ...]

embedding_2 = generate_embedding("Redis has great performance")
# [0.03, -0.13, 0.87, ...]  # Proche de embedding_1

embedding_3 = generate_embedding("I love pizza")
# [-0.78, 0.45, -0.23, ...]  # Très différent de embedding_1
```

---

### Métriques de distance

Pour comparer deux vecteurs, on utilise des **métriques de distance** :

#### 1. Cosine Similarity (Similarité cosinus)

Mesure l'angle entre deux vecteurs (indépendant de la magnitude).

```
cosine_similarity = (A · B) / (||A|| × ||B||)
Résultat : [-1, 1] où 1 = identique, -1 = opposé
```

**Usage** : Texte, recommandations (le plus courant)

---

#### 2. Euclidean Distance (L2)

Distance euclidienne classique.

```
L2 = sqrt(Σ(A[i] - B[i])²)
Résultat : [0, +∞] où 0 = identique
```

**Usage** : Images, données normalisées

---

#### 3. Inner Product (IP)

Produit scalaire des vecteurs.

```
IP = Σ(A[i] × B[i])
Résultat : [-∞, +∞] où plus élevé = plus similaire
```

**Usage** : Vecteurs déjà normalisés

---

### Algorithmes d'indexation

#### FLAT (Brute Force)

- **Principe** : Compare la query avec tous les vecteurs
- **Précision** : 100% (exact)
- **Vitesse** : O(N × D) où N = nombre de vecteurs, D = dimensions
- **Usage** : Petits datasets (< 10K vecteurs)

#### HNSW (Hierarchical Navigable Small World)

- **Principe** : Graphe hiérarchique pour recherche approximative
- **Précision** : ~95-99% (configurable)
- **Vitesse** : O(log N)
- **Usage** : Gros datasets (> 10K vecteurs)

---

## Créer un index vectoriel

### Syntaxe de base

```bash
FT.CREATE {index_name}
  ON {HASH | JSON}
  PREFIX {count} {prefix}
  SCHEMA
    {field} VECTOR {algorithm} {parameters}
    [autres champs...]
```

### Exemple avec FLAT (exact search)

```bash
# Index vectoriel simple (FLAT)
FT.CREATE idx:documents
  ON HASH
  PREFIX 1 doc:
  SCHEMA
    content TEXT
    embedding VECTOR FLAT 6
      TYPE FLOAT32
      DIM 384
      DISTANCE_METRIC COSINE
```

**Paramètres FLAT** :
- `TYPE` : Type de données (FLOAT32 ou FLOAT64)
- `DIM` : Nombre de dimensions (384, 768, 1536, etc.)
- `DISTANCE_METRIC` : COSINE, L2, ou IP

---

### Exemple avec HNSW (approximate search)

```bash
# Index vectoriel HNSW (rapide, approximatif)
FT.CREATE idx:documents
  ON HASH
  PREFIX 1 doc:
  SCHEMA
    content TEXT
    embedding VECTOR HNSW 6
      TYPE FLOAT32
      DIM 384
      DISTANCE_METRIC COSINE
```

**Paramètres HNSW** (identiques à FLAT pour la base)

---

### Exemple avec JSON et HNSW

```bash
# Index vectoriel sur JSON (recommandé)
FT.CREATE idx:documents
  ON JSON
  PREFIX 1 doc:
  SCHEMA
    $.title AS title TEXT
    $.content AS content TEXT
    $.category AS category TAG
    $.embedding AS embedding VECTOR HNSW 14
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
      INITIAL_CAP 10000
      M 16
      EF_CONSTRUCTION 200
      EF_RUNTIME 10
```

**Paramètres HNSW avancés** :
- `INITIAL_CAP` : Capacité initiale (optimisation mémoire)
- `M` : Nombre de connexions par nœud (défaut: 16)
  - Plus élevé = meilleure précision, plus de mémoire
  - Recommandé : 16-64
- `EF_CONSTRUCTION` : Taille de la liste lors de l'indexation (défaut: 200)
  - Plus élevé = meilleure précision, indexation plus lente
  - Recommandé : 100-400
- `EF_RUNTIME` : Taille de la liste lors de la recherche (défaut: 10)
  - Plus élevé = meilleure précision, recherche plus lente
  - Recommandé : 10-100

---

## Insérer des documents avec embeddings

### Format des embeddings

Les embeddings doivent être stockés en **binaire** (blob de bytes).

#### Python : Convertir un vecteur en binaire

```python
import numpy as np
import struct

# Vecteur d'embedding (liste de floats)
embedding = [0.023, -0.145, 0.892, 0.234, -0.567, ...]  # 384 dimensions

# Convertir en binaire (FLOAT32)
embedding_bytes = np.array(embedding, dtype=np.float32).tobytes()

# Stocker dans Redis
redis.hset('doc:1', mapping={
    'content': 'Redis is an in-memory database',
    'embedding': embedding_bytes
})
```

#### Node.js : Convertir un vecteur en binaire

```javascript
// Vecteur d'embedding
const embedding = [0.023, -0.145, 0.892, 0.234, -0.567, ...];  // 384 dimensions

// Convertir en Buffer (FLOAT32)
const buffer = Buffer.alloc(embedding.length * 4);  // 4 bytes par float32
embedding.forEach((value, i) => {
  buffer.writeFloatLE(value, i * 4);
});

// Stocker dans Redis
await redis.hSet('doc:1', {
  content: 'Redis is an in-memory database',
  embedding: buffer
});
```

---

### Exemple complet : Insérer des documents

```bash
# Créer l'index
FT.CREATE idx:docs
  ON HASH
  PREFIX 1 doc:
  SCHEMA
    content TEXT
    category TAG
    embedding VECTOR HNSW 6
      TYPE FLOAT32
      DIM 384
      DISTANCE_METRIC COSINE
```

```python
import redis
import numpy as np
from sentence_transformers import SentenceTransformer

# Connexion Redis
r = redis.Redis(host='localhost', port=6379)

# Modèle d'embedding (384 dimensions)
model = SentenceTransformer('all-MiniLM-L6-v2')

# Documents à indexer
documents = [
    {"id": 1, "content": "Redis is an in-memory database", "category": "database"},
    {"id": 2, "content": "Redis supports various data structures", "category": "database"},
    {"id": 3, "content": "Python is a programming language", "category": "programming"},
    {"id": 4, "content": "Redis has excellent performance", "category": "database"},
]

# Insérer les documents avec embeddings
for doc in documents:
    # Générer l'embedding
    embedding = model.encode(doc['content'])
    embedding_bytes = np.array(embedding, dtype=np.float32).tobytes()

    # Stocker dans Redis
    r.hset(f"doc:{doc['id']}", mapping={
        'content': doc['content'],
        'category': doc['category'],
        'embedding': embedding_bytes
    })

print("Documents indexés avec succès!")
```

---

## Recherche vectorielle : KNN

### Syntaxe de base

```bash
FT.SEARCH {index_name}
  "*=>[KNN {k} @{vector_field} $query_vector AS {score_field}]"
  PARAMS 2 query_vector {blob}
  RETURN {count} {fields}...
  SORTBY {score_field} [ASC|DESC]
  DIALECT 2
```

**Paramètres** :
- `KNN {k}` : Nombre de voisins les plus proches à retourner
- `$query_vector` : Paramètre contenant l'embedding de la requête
- `DIALECT 2` : Requis pour la syntaxe KNN

---

### Exemple de recherche

```python
# Requête utilisateur
query = "Tell me about fast databases"

# Générer l'embedding de la requête
query_embedding = model.encode(query)
query_bytes = np.array(query_embedding, dtype=np.float32).tobytes()

# Recherche des 5 documents les plus similaires
result = r.ft('idx:docs').search(
    Query("*=>[KNN 5 @embedding $query_vector AS score]")
    .return_fields('content', 'category', 'score')
    .sort_by('score')
    .dialect(2),
    query_params={'query_vector': query_bytes}
)

# Afficher les résultats
for doc in result.docs:
    print(f"Score: {doc.score}")
    print(f"Content: {doc.content}")
    print(f"Category: {doc.category}")
    print("---")
```

**Résultat** :

```
Score: 0.856
Content: Redis has excellent performance
Category: database
---
Score: 0.823
Content: Redis is an in-memory database
Category: database
---
Score: 0.701
Content: Redis supports various data structures
Category: database
---
```

---

### Recherche hybride : Vector + Filtres

**Puissance de RediSearch** : Combiner recherche vectorielle et filtres métadonnées.

```python
# Recherche : Documents similaires à la query + catégorie = "database"
result = r.ft('idx:docs').search(
    Query("@category:{database} => [KNN 3 @embedding $query_vector AS score]")
    .return_fields('content', 'score')
    .sort_by('score')
    .dialect(2),
    query_params={'query_vector': query_bytes}
)
```

**Syntaxe bash** :

```bash
FT.SEARCH idx:docs
  "(@category:{database})=>[KNN 3 @embedding $query_vector AS score]"
  PARAMS 2 query_vector "<binary_blob>"
  RETURN 2 content score
  SORTBY score ASC
  DIALECT 2
```

---

## Cas d'usage #1 : RAG (Retrieval Augmented Generation)

### Contexte

**Problème** : Les LLMs (GPT-4, Claude) ont une connaissance limitée :
- Coupure de formation (knowledge cutoff)
- Pas de connaissance de vos données privées
- Hallucinations sur des faits précis

**Solution RAG** :
1. Stocker votre documentation en embeddings dans Redis
2. Pour chaque question utilisateur :
   - Trouver les 5 documents les plus pertinents (Vector Search)
   - Concaténer ces documents comme contexte
   - Envoyer au LLM : contexte + question
3. Le LLM répond en se basant sur le contexte fourni

---

### Architecture RAG avec Redis

```
┌─────────────┐
│   User      │
│  Question   │
└──────┬──────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Application                        │
│  1. Generate query embedding         │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Redis (Vector Search)              │
│  2. Find top K similar documents     │
│     FT.SEARCH ... KNN 5 ...          │
└──────┬───────────────────────────────┘
       │
       ▼ (Relevant documents)
┌──────────────────────────────────────┐
│   Application                        │
│  3. Build context from documents     │
│  4. Send to LLM (GPT-4/Claude)       │
│     Prompt: Context + Question       │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   LLM Response                       │
│   (Based on provided context)        │
└──────────────────────────────────────┘
```

---

### Implémentation complète

#### Étape 1 : Indexer la documentation

```python
import redis
import numpy as np
from openai import OpenAI
from redis.commands.search.field import TextField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

# Connexion
r = redis.Redis(host='localhost', port=6379, decode_responses=True)
client = OpenAI()

# Créer l'index vectoriel
try:
    r.ft('idx:docs').dropindex()
except:
    pass

schema = (
    TextField('title'),
    TextField('content'),
    TextField('url'),
    VectorField('embedding',
        'HNSW', {
            'TYPE': 'FLOAT32',
            'DIM': 1536,  # OpenAI text-embedding-3-small
            'DISTANCE_METRIC': 'COSINE'
        }
    )
)

r.ft('idx:docs').create_index(
    schema,
    definition=IndexDefinition(prefix=['doc:'], index_type=IndexType.HASH)
)

# Documentation à indexer
docs = [
    {
        "title": "Redis Cluster",
        "content": "Redis Cluster provides automatic sharding across multiple Redis nodes...",
        "url": "https://redis.io/topics/cluster-tutorial"
    },
    {
        "title": "Redis Persistence",
        "content": "Redis provides different persistence options: RDB and AOF...",
        "url": "https://redis.io/topics/persistence"
    },
    {
        "title": "Redis Pub/Sub",
        "content": "Redis Pub/Sub implements the messaging pattern...",
        "url": "https://redis.io/topics/pubsub"
    },
    # ... plus de documents
]

# Générer embeddings et indexer
for i, doc in enumerate(docs, 1):
    # Générer embedding avec OpenAI
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=doc['content']
    )
    embedding = response.data[0].embedding
    embedding_bytes = np.array(embedding, dtype=np.float32).tobytes()

    # Stocker dans Redis
    r.hset(f'doc:{i}', mapping={
        'title': doc['title'],
        'content': doc['content'],
        'url': doc['url'],
        'embedding': embedding_bytes
    })

print(f"✅ {len(docs)} documents indexés")
```

---

#### Étape 2 : Fonction de recherche RAG

```python
def rag_search(question: str, top_k: int = 5):
    """
    Recherche les documents les plus pertinents pour une question.
    """
    # Générer l'embedding de la question
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=question
    )
    query_embedding = response.data[0].embedding
    query_bytes = np.array(query_embedding, dtype=np.float32).tobytes()

    # Recherche vectorielle dans Redis
    result = r.ft('idx:docs').search(
        Query(f"*=>[KNN {top_k} @embedding $query_vector AS score]")
        .return_fields('title', 'content', 'url', 'score')
        .sort_by('score')
        .dialect(2),
        query_params={'query_vector': query_bytes}
    )

    # Formatter les résultats
    docs = []
    for doc in result.docs:
        docs.append({
            'title': doc.title,
            'content': doc.content,
            'url': doc.url,
            'score': float(doc.score)
        })

    return docs
```

---

#### Étape 3 : Interroger le LLM avec contexte

```python
def ask_with_rag(question: str):
    """
    Question-answering avec RAG.
    """
    # 1. Trouver les documents pertinents
    relevant_docs = rag_search(question, top_k=5)

    # 2. Construire le contexte
    context = "\n\n".join([
        f"Document {i+1}: {doc['title']}\n{doc['content']}"
        for i, doc in enumerate(relevant_docs)
    ])

    # 3. Construire le prompt
    prompt = f"""Answer the question based on the context below. If you cannot answer based on the context, say "I don't have enough information to answer this question."

Context:
{context}

Question: {question}

Answer:"""

    # 4. Interroger le LLM
    response = client.chat.completions.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "You are a helpful assistant that answers questions based on provided context."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.3
    )

    answer = response.choices[0].message.content

    return {
        'answer': answer,
        'sources': [doc['url'] for doc in relevant_docs]
    }

# Utilisation
result = ask_with_rag("How does Redis Cluster work?")
print("Answer:", result['answer'])
print("\nSources:")
for url in result['sources']:
    print(f"  - {url}")
```

**Résultat** :

```
Answer: Redis Cluster provides automatic sharding across multiple Redis nodes.
It distributes data automatically across multiple nodes, allowing horizontal scaling.
Each node in the cluster is responsible for a subset of the hash slots...

Sources:
  - https://redis.io/topics/cluster-tutorial
  - https://redis.io/topics/cluster-spec
```

---

### Optimisations RAG

#### Re-ranking (améliorer la pertinence)

```python
def rag_search_with_reranking(question: str, top_k: int = 10, final_k: int = 5):
    """
    1. Vector Search : Récupérer top_k documents (10)
    2. Re-ranking : Utiliser un modèle cross-encoder pour classer
    3. Retourner final_k documents (5)
    """
    from sentence_transformers import CrossEncoder

    # 1. Vector search (large recall)
    docs = rag_search(question, top_k=top_k)

    # 2. Re-ranking avec cross-encoder
    cross_encoder = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-12-v2')
    pairs = [[question, doc['content']] for doc in docs]
    scores = cross_encoder.predict(pairs)

    # 3. Trier par score du cross-encoder
    for doc, score in zip(docs, scores):
        doc['rerank_score'] = float(score)

    docs_sorted = sorted(docs, key=lambda x: x['rerank_score'], reverse=True)

    return docs_sorted[:final_k]
```

---

#### Chunking (découper les longs documents)

```python
def chunk_document(text: str, chunk_size: int = 500, overlap: int = 50):
    """
    Découpe un long document en chunks avec overlap.
    """
    words = text.split()
    chunks = []

    for i in range(0, len(words), chunk_size - overlap):
        chunk = ' '.join(words[i:i + chunk_size])
        chunks.append(chunk)

    return chunks

# Indexer avec chunks
doc_text = "Very long document..." * 100
chunks = chunk_document(doc_text)

for i, chunk in enumerate(chunks):
    embedding = generate_embedding(chunk)
    r.hset(f'chunk:{doc_id}:{i}', mapping={
        'doc_id': doc_id,
        'chunk_index': i,
        'content': chunk,
        'embedding': embedding
    })
```

---

## Cas d'usage #2 : Recherche sémantique de produits

### Contexte

Permettre aux utilisateurs de rechercher des produits par description naturelle plutôt que par mots-clés exacts.

**Exemple** :
- Recherche traditionnelle : "laptop 15 pouces"
- Recherche sémantique : "ordinateur portable léger pour développeur qui voyage souvent"

---

### Implémentation

```python
# Créer l'index produits avec embeddings
FT.CREATE idx:products
  ON JSON
  PREFIX 1 product:
  SCHEMA
    $.name AS name TEXT WEIGHT 5.0
    $.description AS description TEXT WEIGHT 2.0
    $.category AS category TAG
    $.price AS price NUMERIC SORTABLE
    $.stock AS stock NUMERIC
    $.description_embedding AS embedding VECTOR HNSW 14
      TYPE FLOAT32
      DIM 384
      DISTANCE_METRIC COSINE
      M 16
      EF_CONSTRUCTION 200

# Indexer les produits
products = [
    {
        "name": "MacBook Pro 16",
        "description": "Powerful laptop for developers with M3 Pro chip, 16GB RAM, lightweight design perfect for travel",
        "category": "laptop",
        "price": 2499.99,
        "stock": 20
    },
    {
        "name": "Dell XPS 13",
        "description": "Ultra-portable laptop with excellent battery life, ideal for mobile professionals",
        "category": "laptop",
        "price": 1299.99,
        "stock": 45
    },
    # ... plus de produits
]

for i, product in enumerate(products, 1):
    # Générer embedding de la description
    embedding = model.encode(product['description'])

    # Stocker avec RedisJSON
    redis.json().set(f'product:{i}', '$', {
        **product,
        'description_embedding': embedding.tolist()
    })

# Recherche sémantique
query = "lightweight computer for traveling developer"
query_embedding = model.encode(query)
query_bytes = np.array(query_embedding, dtype=np.float32).tobytes()

result = r.ft('idx:products').search(
    Query("@stock:[1 +inf] => [KNN 5 @embedding $query_vector AS score]")
    .return_fields('name', 'price', 'score')
    .sort_by('score')
    .dialect(2),
    query_params={'query_vector': query_bytes}
)

# Résultats pertinents même sans mots-clés exacts :
# 1. MacBook Pro 16 (score: 0.82)
# 2. Dell XPS 13 (score: 0.79)
```

---

## Cas d'usage #3 : Recommendation Engine

### Contexte

Trouver des produits/contenus similaires à ce que l'utilisateur consulte actuellement.

---

### Implémentation

```python
def find_similar_products(product_id: int, top_k: int = 5):
    """
    Trouve des produits similaires basés sur les embeddings.
    """
    # Récupérer l'embedding du produit actuel
    product = r.json().get(f'product:{product_id}')
    product_embedding_bytes = np.array(
        product['description_embedding'],
        dtype=np.float32
    ).tobytes()

    # Recherche vectorielle (exclure le produit actuel)
    result = r.ft('idx:products').search(
        Query(f"*=>[KNN {top_k + 1} @embedding $query_vector AS score]")
        .return_fields('name', 'price', 'score')
        .sort_by('score')
        .dialect(2),
        query_params={'query_vector': product_embedding_bytes}
    )

    # Filtrer le produit actuel
    similar_products = [
        doc for doc in result.docs
        if doc.id != f'product:{product_id}'
    ][:top_k]

    return similar_products

# Usage dans une page produit
# "Vous aimerez aussi : ..."
similar = find_similar_products(product_id=101, top_k=4)
```

---

## Cas d'usage #4 : Détection de contenu dupliqué

### Contexte

Détecter les documents/produits dupliqués ou très similaires.

---

### Implémentation

```python
def detect_duplicates(threshold: float = 0.95):
    """
    Détecte les documents avec similarité > threshold.
    """
    # Récupérer tous les documents
    all_docs = r.ft('idx:docs').search(Query('*').paging(0, 10000))

    duplicates = []

    for i, doc1 in enumerate(all_docs.docs):
        # Recherche des documents similaires
        embedding_bytes = r.hget(doc1.id, 'embedding')

        result = r.ft('idx:docs').search(
            Query("*=>[KNN 10 @embedding $query_vector AS score]")
            .return_fields('title', 'score')
            .sort_by('score')
            .dialect(2),
            query_params={'query_vector': embedding_bytes}
        )

        # Filtrer par threshold
        for doc2 in result.docs:
            if doc2.id != doc1.id and float(doc2.score) > threshold:
                duplicates.append({
                    'doc1': doc1.id,
                    'doc2': doc2.id,
                    'similarity': float(doc2.score)
                })

    return duplicates

# Détecter les doublons
duplicates = detect_duplicates(threshold=0.95)
print(f"Found {len(duplicates)} potential duplicates")
```

---

## Cas d'usage #5 : Recherche multimodale (texte + images)

### Contexte

Modèles comme CLIP génèrent des embeddings communs pour texte et images, permettant de rechercher des images par description textuelle.

---

### Implémentation

```python
from transformers import CLIPProcessor, CLIPModel
from PIL import Image

# Charger le modèle CLIP
model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

# Créer l'index
FT.CREATE idx:images
  ON HASH
  PREFIX 1 img:
  SCHEMA
    url TEXT
    caption TEXT
    embedding VECTOR HNSW 6
      TYPE FLOAT32
      DIM 512  # CLIP dimensions
      DISTANCE_METRIC COSINE

# Indexer les images
images = [
    {"url": "https://example.com/cat.jpg", "caption": "A cat sitting on a windowsill"},
    {"url": "https://example.com/dog.jpg", "caption": "A dog playing in the park"},
    # ...
]

for i, img_data in enumerate(images, 1):
    # Générer embedding de l'image
    image = Image.open(img_data['url'])
    inputs = processor(images=image, return_tensors="pt")
    embedding = model.get_image_features(**inputs).detach().numpy()[0]

    r.hset(f'img:{i}', mapping={
        'url': img_data['url'],
        'caption': img_data['caption'],
        'embedding': embedding.tobytes()
    })

# Recherche d'images par texte
def search_images_by_text(text_query: str, top_k: int = 5):
    # Générer embedding du texte
    inputs = processor(text=[text_query], return_tensors="pt")
    text_embedding = model.get_text_features(**inputs).detach().numpy()[0]
    text_embedding_bytes = text_embedding.tobytes()

    # Recherche vectorielle
    result = r.ft('idx:images').search(
        Query(f"*=>[KNN {top_k} @embedding $query_vector AS score]")
        .return_fields('url', 'caption', 'score')
        .sort_by('score')
        .dialect(2),
        query_params={'query_vector': text_embedding_bytes}
    )

    return result.docs

# Recherche : "animal playing outside"
results = search_images_by_text("animal playing outside")
# Retourne l'image du chien dans le parc
```

---

## Performance et optimisation

### Benchmark : Latence de recherche

```bash
# Test : 1 million de vecteurs (384 dimensions)

# FLAT (exact search)
# Latence moyenne : 150-300ms
# Précision : 100%

# HNSW (M=16, EF_RUNTIME=10)
# Latence moyenne : 3-8ms
# Précision : 95-98%
# Gain : 40-80x plus rapide

# HNSW (M=32, EF_RUNTIME=50)
# Latence moyenne : 15-25ms
# Précision : 98-99.5%
# Gain : 10-20x plus rapide
```

---

### Tuning HNSW pour optimiser

#### Trade-off : Vitesse vs Précision

```bash
# Configuration RAPIDE (moins précis)
M=16
EF_CONSTRUCTION=100
EF_RUNTIME=10
# Latence : ~3ms, Précision : ~95%

# Configuration ÉQUILIBRÉE
M=16
EF_CONSTRUCTION=200
EF_RUNTIME=20
# Latence : ~5ms, Précision : ~97%

# Configuration PRÉCISE (plus lent)
M=32
EF_CONSTRUCTION=400
EF_RUNTIME=50
# Latence : ~15ms, Précision : ~99%
```

**Recommandation** : Commencer avec M=16, EF_CONSTRUCTION=200, EF_RUNTIME=10, puis ajuster selon vos besoins.

---

### Impact mémoire

```bash
# Exemple : 1 million de vecteurs (1536 dimensions)

# Vecteurs seuls :
# 1M × 1536 × 4 bytes (FLOAT32) = 6.14 GB

# Avec index HNSW (M=16) :
# Vecteurs : 6.14 GB
# Index : ~2-3 GB (graphe HNSW)
# Total : ~8-9 GB

# Overhead : +30-45%
```

---

### Choix de la dimensionnalité

| Modèle d'embedding | Dimensions | Usage |
|--------------------|------------|-------|
| **all-MiniLM-L6-v2** | 384 | General purpose, rapide |
| **all-mpnet-base-v2** | 768 | Meilleure qualité |
| **OpenAI text-embedding-3-small** | 1536 | Production, haute qualité |
| **OpenAI text-embedding-3-large** | 3072 | Qualité maximale |
| **CLIP ViT-B/32** | 512 | Multimodal (texte + images) |

**Trade-off** :
- Plus de dimensions = meilleure qualité, mais plus de mémoire et latence
- Recommandation : 384-1536 dimensions pour la plupart des cas

---

## Comparaison : Redis vs autres Vector Databases

| Critère | Redis Vector Search | Pinecone | Weaviate | Milvus | Qdrant |
|---------|---------------------|----------|----------|--------|--------|
| **Latence** | < 5ms | 20-50ms | 10-30ms | 10-40ms | 5-15ms |
| **Throughput** | 50K+ queries/sec | 5-10K/sec | 10-20K/sec | 10-30K/sec | 20-40K/sec |
| **Scalabilité** | Horizontal (Cluster) | Managed | Horizontal | Horizontal | Horizontal |
| **Filtres hybrides** | ✅ Excellent | ✅ Bon | ✅ Excellent | ✅ Bon | ✅ Excellent |
| **Prix** | Faible (self-hosted) | Élevé (SaaS) | Moyen | Faible | Moyen |
| **Intégration** | Redis natif | API dédiée | API dédiée | API dédiée | API dédiée |
| **Cas d'usage** | RAG, recommandations | ML production | Semantic search | ML research | ML production |

**Avantages Redis** :
- ✅ Latence exceptionnelle (< 5ms)
- ✅ Intégration native avec votre infrastructure Redis existante
- ✅ Filtres hybrides puissants (vector + métadonnées)
- ✅ Coût faible (self-hosted)
- ✅ Pas de service externe à gérer

**Quand choisir une alternative** :
- Weaviate/Pinecone : Si vous voulez une solution managée dédiée
- Milvus : Si vous faites de la recherche ML avancée
- Qdrant : Si vous voulez une alternative self-hosted spécialisée

---

## Bonnes pratiques

### ✅ 1. Normaliser les embeddings

```python
import numpy as np

def normalize_embedding(embedding):
    """
    Normalise un vecteur (utile pour IP distance).
    """
    norm = np.linalg.norm(embedding)
    return embedding / norm if norm > 0 else embedding

# Utilisation
embedding = model.encode(text)
embedding_normalized = normalize_embedding(embedding)
```

---

### ✅ 2. Utiliser le bon modèle d'embedding

```python
# ✅ Bon : Modèle spécialisé pour votre langue
model = SentenceTransformer('sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2')  # Multi-langue

# ❌ Mauvais : Modèle anglais pour du français
model = SentenceTransformer('all-MiniLM-L6-v2')  # Anglais seulement
```

---

### ✅ 3. Chunking pour les longs documents

```python
# ✅ Bon : Découper les longs documents
max_tokens = 512  # Limite du modèle
chunks = chunk_document(long_text, max_tokens)

for i, chunk in enumerate(chunks):
    embedding = model.encode(chunk)
    r.hset(f'doc:{doc_id}:chunk:{i}', ...)

# ❌ Mauvais : Tronquer arbitrairement
text_truncated = long_text[:512]  # Perte d'information
```

---

### ✅ 4. Cacher les embeddings

```python
# ✅ Bon : Générer une seule fois et stocker
embedding = model.encode(text)
r.hset('doc:1', 'embedding', embedding.tobytes())

# ❌ Mauvais : Régénérer à chaque recherche
# (Coûteux : 50-200ms par embedding)
```

---

### ✅ 5. Monitoring de la précision

```python
def evaluate_search_quality(test_queries, expected_results):
    """
    Évalue la qualité de la recherche vectorielle.
    """
    hits = 0

    for query, expected in zip(test_queries, expected_results):
        results = vector_search(query, top_k=5)
        result_ids = [doc.id for doc in results]

        if expected in result_ids:
            hits += 1

    accuracy = hits / len(test_queries)
    print(f"Search accuracy: {accuracy:.2%}")

    return accuracy

# Tester régulièrement
test_queries = ["How does Redis Cluster work?", ...]
expected = ["doc:1", ...]
evaluate_search_quality(test_queries, expected)
```

---

## Intégration avec les LLMs populaires

### OpenAI

```python
from openai import OpenAI

client = OpenAI()

# Générer embedding
response = client.embeddings.create(
    model="text-embedding-3-small",  # 1536 dimensions
    input="Your text here"
)
embedding = response.data[0].embedding

# Interroger GPT-4 avec RAG
context = get_relevant_docs(question)
response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "Answer based on context."},
        {"role": "user", "content": f"Context: {context}\n\nQuestion: {question}"}
    ]
)
```

---

### Anthropic Claude

```python
import anthropic

client = anthropic.Anthropic()

# Claude n'a pas d'API embeddings native
# → Utiliser un modèle externe (Cohere, Voyage AI, etc.)

# Interroger Claude avec RAG
context = get_relevant_docs(question)
message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": f"Context:\n{context}\n\nQuestion: {question}\n\nAnswer based on the context above."
    }]
)
```

---

### Cohere

```python
import cohere

co = cohere.Client('your-api-key')

# Générer embedding
response = co.embed(
    texts=["Your text here"],
    model="embed-english-v3.0"  # ou embed-multilingual-v3.0
)
embedding = response.embeddings[0]

# Interroger avec RAG
context = get_relevant_docs(question)
response = co.chat(
    message=question,
    documents=[{"text": doc} for doc in context],
    model="command-r-plus"
)
```

---

## Troubleshooting

### Erreur : "Wrong vector dimension"

```bash
# ❌ Erreur
(error) Vector dimension mismatch: expected 384, got 768

# Solution : Vérifier la dimension du modèle
model = SentenceTransformer('all-MiniLM-L6-v2')  # 384 dimensions
# vs
model = SentenceTransformer('all-mpnet-base-v2')  # 768 dimensions

# L'index doit correspondre
FT.CREATE ... DIM 384  # Doit matcher le modèle
```

---

### Résultats de mauvaise qualité

```python
# ✅ Solutions possibles :

# 1. Utiliser un meilleur modèle d'embedding
model = SentenceTransformer('all-mpnet-base-v2')  # 768 dim, meilleure qualité

# 2. Augmenter EF_RUNTIME pour plus de précision
# DIM 384 DISTANCE_METRIC COSINE M 16 EF_CONSTRUCTION 200 EF_RUNTIME 50

# 3. Passer de HNSW à FLAT pour 100% de précision
# (Si dataset < 10K vecteurs)

# 4. Ajouter du re-ranking
similar = vector_search(query, top_k=20)
reranked = rerank_with_cross_encoder(query, similar, final_k=5)
```

---

### Performance dégradée

```python
# ✅ Optimisations :

# 1. Réduire M et EF_RUNTIME
# M=16, EF_RUNTIME=10

# 2. Réduire la dimensionnalité du modèle
model = SentenceTransformer('all-MiniLM-L6-v2')  # 384 dim au lieu de 768

# 3. Utiliser Redis Cluster pour distribuer la charge

# 4. Limiter top_k
result = vector_search(query, top_k=5)  # Au lieu de 50
```

---

## Résumé

**Vector Search avec RediSearch permet de** :
- ✅ Recherche sémantique (par signification, pas mots-clés)
- ✅ RAG pour LLMs (contexte pour GPT-4, Claude)
- ✅ Recommendation engines (produits/contenus similaires)
- ✅ Détection de similarité/duplicatas
- ✅ Recherche multimodale (texte + images)

**Algorithmes** :
- `FLAT` : Exact search, petit dataset (< 10K)
- `HNSW` : Approximate search, gros dataset (> 10K)

**Métriques de distance** :
- `COSINE` : Texte, recommandations (le plus courant)
- `L2` : Images, données normalisées
- `IP` : Vecteurs normalisés

**Performance** :
- Latence : < 5ms pour millions de vecteurs
- Throughput : 50K+ requêtes/sec
- 40-80x plus rapide que recherche exacte

**Cas d'usage production** :
- 🤖 Chatbots avec RAG
- 🔍 Moteurs de recherche sémantique
- 🎯 Systèmes de recommandation
- 🖼️ Recherche d'images par description
- 📚 Bases de connaissances intelligentes

---

**Prêt pour les séries temporelles ?** Passons à la section suivante : [3.6 RedisTimeSeries - Gestion de données temporelles](./06-redistimeseries-donnees-temporelles.md)

⏭️ [RedisTimeSeries : Gestion de données temporelles (IoT, monitoring)](/03-structures-donnees-etendues/06-redistimeseries-donnees-temporelles.md)
