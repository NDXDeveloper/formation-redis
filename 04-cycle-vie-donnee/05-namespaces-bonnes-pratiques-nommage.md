🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5 Namespaces et bonnes pratiques de nommage (Key patterns)

## Introduction

Dans Redis, **tout est une clé**. Contrairement aux bases de données relationnelles avec leurs schémas, tables et colonnes, Redis est un simple espace de clés plat (flat keyspace). Cette simplicité est à la fois sa force et son défi : sans discipline de nommage, un keyspace Redis peut rapidement devenir un chaos ingérable.

Les namespaces et patterns de nommage ne sont pas une fonctionnalité native de Redis, mais une **convention d'organisation** que vous imposez. Bien implémentés, ils permettent :

- 🎯 **Organisation logique** : Retrouver facilement les données
- 🔍 **Recherche efficace** : Filtrer avec SCAN/KEYS
- 🚀 **Scalabilité** : Partitionnement et sharding prévisibles
- 🔒 **Sécurité** : ACLs granulaires par namespace
- 📊 **Monitoring** : Métriques par type de données
- 🧹 **Maintenance** : Nettoyage et TTL par catégorie

## Le keyspace Redis : Architecture plate

### Structure fondamentale

```
Redis Database (DB 0):
┌────────────────────────────────────┐
│ Flat Keyspace                      │
│                                    │
│ "user:1:name"                      │
│ "user:1:email"                     │
│ "user:2:name"                      │
│ "session:abc123"                   │
│ "cache:api:weather:paris"          │
│ "queue:emails"                     │
│ "counter:views:article:42"         │
│ "lock:resource:db"                 │
│ ...                                │
│                                    │
│ PAS de hiérarchie réelle           │
│ PAS de dossiers                    │
│ PAS de tables                      │
└────────────────────────────────────┘
```

### Implication de la structure plate

```c
// Internalement, Redis stocke TOUT dans un dictionnaire
typedef struct redisDb {
    dict *dict;           // Hash table: key → value
    dict *expires;        // Hash table: key → expiration
    // ...
} redisDb;

// Une clé est juste une string, RIEN de plus
// "user:1:name" n'a PAS de relation sémantique avec "user:2:name"
// C'est juste deux strings différentes
```

**Conséquence critique** : Toute organisation logique doit être **imposée par convention** dans le nom des clés.

## Séparateurs : Le choix fondamental

### Les trois séparateurs courants

#### 1. Deux-points `:` (RECOMMANDÉ)

```bash
user:123:profile
cache:api:weather:paris
session:abc123:data
counter:views:article:42
```

**Avantages** :
- ✅ Standard de facto dans la communauté Redis
- ✅ Lisible et concis
- ✅ Pas d'échappement nécessaire
- ✅ Compatible URL-encoding
- ✅ Fonctionne avec SCAN patterns

**Documentation officielle** : Redis utilise `:` dans tous ses exemples.

#### 2. Slash `/` (style REST)

```bash
user/123/profile
cache/api/weather/paris
session/abc123/data
```

**Avantages** :
- ✅ Familier aux développeurs web
- ✅ Style hiérarchique visuel

**Inconvénients** :
- ⚠️ Moins standard dans Redis
- ⚠️ Peut nécessiter échappement dans certains contextes

#### 3. Pipe `|` (rare)

```bash
user|123|profile
cache|api|weather|paris
```

**Inconvénients** :
- ❌ Non standard
- ❌ Moins lisible
- ❌ Peut causer confusion avec opérateurs Unix

### Comparaison technique

```bash
# SCAN avec pattern
SCAN 0 MATCH user:*     # ✅ Standard
SCAN 0 MATCH user/*     # ⚠️ Fonctionne mais moins idiomatique
SCAN 0 MATCH user|*     # ⚠️ Fonctionne mais déroutant

# Tri lexicographique
user:1:name
user:2:name
user:10:name   # Ordre: 1, 10, 2 (tri string !)

# Longueur
"user:123:profile"  → 17 bytes
"user/123/profile"  → 17 bytes
"user|123|profile"  → 17 bytes
# Identique, choix purement conventionnel
```

**Recommandation** : **Toujours utiliser `:`** sauf raison spécifique.

### Mélange de séparateurs (ANTI-PATTERN)

```bash
# ❌ MAUVAIS : Incohérent
user:123/profile
cache|api:weather/paris
session:abc123|data

# ✅ BON : Cohérent
user:123:profile
cache:api:weather:paris
session:abc123:data
```

## Patterns de nommage standard

### Pattern général recommandé

```
<type>:<entity>:<id>:<attribute>
  │      │       │      │
  │      │       │      └─ Attribut spécifique (optionnel)
  │      │       └─ Identifiant unique
  │      └─ Type d'entité
  └─ Catégorie de données
```

**Exemples** :

```bash
# Utilisateur
user:123:profile          # Type: user, ID: 123, Attr: profile
user:123:settings
user:123:preferences

# Session
session:abc123:data       # Type: session, ID: abc123
session:abc123:cart

# Cache
cache:api:weather:paris   # Type: cache, Source: api, Query: weather:paris
cache:db:user:123
cache:query:a1b2c3

# Compteur
counter:views:article:42  # Type: counter, Metric: views, Target: article:42
counter:likes:post:789

# File d'attente
queue:emails              # Type: queue, Purpose: emails
queue:jobs:high
queue:notifications

# Lock
lock:resource:database    # Type: lock, Resource: database
lock:file:upload:123
```

### Pattern avec namespace d'application

Pour applications multi-tenants ou microservices :

```bash
<app>:<type>:<entity>:<id>:<attribute>
  │     │      │       │      │
  │     │      │       │      └─ Attribut
  │     │      │       └─ ID
  │     │      └─ Entité
  │     └─ Type
  └─ Application/Tenant

# Exemples
myapp:user:123:profile
myapp:session:abc:data
myapp:cache:api:result

# Multi-tenant
tenant:acme:user:123:profile
tenant:acme:cache:data
tenant:xyz:user:456:profile
```

### Pattern hiérarchique profond

```bash
# E-commerce
product:category:electronics:subcategory:phones:brand:apple:model:iphone15

# Trop profond ! Difficile à maintenir
# Préférer:
product:electronics:phones:apple:iphone15

# Ou séparer les dimensions:
product:iphone15:category   → "electronics:phones"
product:iphone15:brand      → "apple"
product:iphone15:price      → "999.99"
```

**Règle** : Maximum 4-5 niveaux de profondeur.

## Conventions par type de données

### Strings (valeurs simples)

```bash
# Profil utilisateur (JSON sérialisé)
user:123:profile          → '{"name":"Alice","age":30}'

# Configuration
config:app:max_connections → "100"
config:app:timeout         → "30"

# Feature flags
feature:new_ui:enabled     → "true"
feature:beta:users         → "user:1,user:2,user:3"
```

### Hashes (objets structurés)

```bash
# Utilisateur comme hash
user:123                   # Hash
  ├─ name: "Alice"
  ├─ email: "alice@example.com"
  ├─ age: "30"
  └─ city: "Paris"

# HGETALL user:123
# Vs
# user:123:name, user:123:email (strings séparées)

# Avantage: Structure atomique, HINCRBY, etc.
```

**Pattern recommandé pour hashes** :

```bash
<type>:<id>              # Hash contient tous les attributs
user:123
product:456
order:789
```

### Lists (files d'attente, historiques)

```bash
# Files d'attente
queue:emails              # LPUSH/RPOP
queue:jobs:high
queue:jobs:low
queue:notifications

# Historique
history:user:123:logins   # LPUSH timestamp
history:user:123:purchases
log:errors:app            # LPUSH log entries
```

### Sets (collections uniques)

```bash
# Tags
tags:article:42           # SADD tag1 tag2 tag3
tags:user:123:interests

# Membres
members:group:admins      # SADD user:1 user:2
members:room:lobby

# Indices inversés
index:tag:redis           # SADD article:1 article:5 article:9
index:author:alice        # SADD post:1 post:3
```

### Sorted Sets (classements, indices)

```bash
# Leaderboards
leaderboard:game:global   # ZADD score member
leaderboard:game:2024

# Index temporel
index:posts:by_date       # ZADD timestamp post_id
index:users:by_created

# Priorités
tasks:by_priority         # ZADD priority task_id
```

### Streams (événements)

```bash
# Événements
stream:events:user:123    # XADD
stream:logs:application
stream:orders:new

# Pattern général
stream:<domain>:<type>
```

## Cas d'usage détaillés

### Cas 1 : Session Store

**Besoins** :
- Sessions utilisateur HTTP
- Données associées (panier, préférences)
- TTL automatique

**Nommage** :

```bash
# Option 1 : String avec JSON
session:<session_id>:data
  → '{"user_id":123,"cart":[1,2,3],"created_at":"..."}'

# Option 2 : Hash (RECOMMANDÉ)
session:<session_id>
  ├─ user_id: "123"
  ├─ cart: "1,2,3"
  ├─ created_at: "2024-12-09T15:30:00Z"
  └─ last_activity: "2024-12-09T16:00:00Z"

# Données additionnelles
session:<session_id>:cart       # List pour panier détaillé
session:<session_id>:preferences # Hash pour préférences
```

**Implémentation** :

```python
# Création session
session_id = generate_session_id()
redis.hset(f"session:{session_id}", mapping={
    "user_id": user_id,
    "created_at": datetime.now().isoformat()
})
redis.expire(f"session:{session_id}", 1800)  # 30 min

# Panier dans une liste séparée
redis.lpush(f"session:{session_id}:cart", product_id)
redis.expire(f"session:{session_id}:cart", 1800)

# Lecture
session_data = redis.hgetall(f"session:{session_id}")
cart_items = redis.lrange(f"session:{session_id}:cart", 0, -1)
```

### Cas 2 : Cache multi-niveaux

**Besoins** :
- Cache de différentes sources (DB, API, calculs)
- TTL variables
- Invalidation sélective

**Nommage** :

```bash
# Par source
cache:db:<table>:<id>
cache:api:<service>:<endpoint>:<params_hash>
cache:computed:<function>:<args_hash>

# Exemples concrets
cache:db:users:123                    # User depuis DB
cache:api:weather:current:paris       # API météo
cache:api:github:user:octocat         # API GitHub
cache:computed:fibonacci:50           # Calcul coûteux

# Avec versioning
cache:v2:db:users:123                 # Version du cache
cache:v2:api:weather:current:paris
```

**Implémentation** :

```python
def cache_key(source, *parts):
    """Génère clé de cache cohérente"""
    return f"cache:{source}:" + ":".join(str(p) for p in parts)

# Utilisation
def get_user_from_db(user_id):
    key = cache_key("db", "users", user_id)

    # Try cache
    cached = redis.get(key)
    if cached:
        return json.loads(cached)

    # Cache miss
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(key, 3600, json.dumps(user))
    return user

def get_weather(city):
    key = cache_key("api", "weather", "current", city.lower())

    cached = redis.get(key)
    if cached:
        return json.loads(cached)

    weather = api.fetch_weather(city)
    redis.setex(key, 1800, json.dumps(weather))
    return weather
```

**Invalidation** :

```python
# Invalider tous les caches DB d'un type
keys = redis.keys("cache:db:users:*")  # ⚠️ Utiliser SCAN en prod
for key in keys:
    redis.delete(key)

# Invalider par pattern (avec SCAN)
cursor = 0
while True:
    cursor, keys = redis.scan(cursor, match="cache:api:weather:*", count=100)
    if keys:
        redis.delete(*keys)
    if cursor == 0:
        break
```

### Cas 3 : Rate Limiting

**Besoins** :
- Limitation par IP, user, API key
- Fenêtres temporelles
- Plusieurs limites (par seconde, minute, heure)

**Nommage** :

```bash
# Pattern général
rate:<scope>:<identifier>:<window>

# Par IP
rate:ip:192.168.1.100:second    # Limite par seconde
rate:ip:192.168.1.100:minute    # Limite par minute
rate:ip:192.168.1.100:hour      # Limite par heure

# Par user
rate:user:123:api:calls:minute
rate:user:123:api:calls:hour

# Par API key
rate:apikey:abc123:requests:second

# Avec timestamp (sliding window)
rate:ip:192.168.1.100:1733847600  # Timestamp epoch
```

**Implémentation Fixed Window** :

```python
def rate_limit_fixed(redis, identifier, limit=100, window=60):
    """Rate limiting avec fenêtre fixe"""
    current_window = int(time.time() // window)
    key = f"rate:ip:{identifier}:{current_window}"

    count = redis.incr(key)

    if count == 1:
        redis.expire(key, window * 2)  # Safety margin

    return count <= limit

# Utilisation
if not rate_limit_fixed(redis, client_ip, limit=100, window=60):
    raise RateLimitExceeded("Too many requests")
```

**Implémentation Sliding Window** :

```python
def rate_limit_sliding(redis, identifier, limit=100, window=60):
    """Rate limiting avec fenêtre glissante"""
    now = time.time()
    key = f"rate:sliding:{identifier}"

    # Supprime événements expirés
    redis.zremrangebyscore(key, 0, now - window)

    # Compte événements dans la fenêtre
    count = redis.zcard(key)

    if count < limit:
        # Ajoute nouvel événement
        redis.zadd(key, {f"{now}:{random.random()}": now})
        redis.expire(key, window)
        return True

    return False
```

### Cas 4 : Distributed Locking

**Besoins** :
- Locks sur ressources
- TTL automatique (éviter deadlocks)
- Identification du propriétaire

**Nommage** :

```bash
# Pattern
lock:<resource_type>:<resource_id>

# Exemples
lock:user:123                    # Lock sur user
lock:file:upload:abc             # Lock sur upload
lock:resource:database:write     # Lock écriture DB
lock:job:processing:order:456    # Lock traitement job

# Valeur : Identifier unique du lock owner
# SET lock:user:123 "server1:thread42:uuid123" NX EX 10
```

**Implémentation** :

```python
import uuid

def acquire_lock(redis, resource, ttl=10):
    """Acquiert un lock distribué"""
    identifier = f"{socket.gethostname()}:{threading.get_ident()}:{uuid.uuid4()}"
    key = f"lock:{resource}"

    if redis.set(key, identifier, nx=True, ex=ttl):
        return identifier
    return None

def release_lock(redis, resource, identifier):
    """Libère un lock de manière atomique"""
    key = f"lock:{resource}"

    # Script Lua pour atomicité
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """

    return redis.eval(script, 1, key, identifier)

# Utilisation
lock_id = acquire_lock(redis, "user:123:profile")
if lock_id:
    try:
        # Opération critique
        update_user_profile(123)
    finally:
        release_lock(redis, "user:123:profile", lock_id)
else:
    raise ResourceLocked("Profile is being updated")
```

### Cas 5 : Compteurs et analytics

**Besoins** :
- Métriques diverses (vues, likes, etc.)
- Agrégation temporelle
- Incréments atomiques

**Nommage** :

```bash
# Compteurs simples
counter:views:article:42
counter:likes:post:123
counter:downloads:file:abc

# Avec dimension temporelle
counter:views:article:42:2024:12:09      # Par jour
counter:views:article:42:2024:12         # Par mois
counter:views:article:42:2024            # Par année

# Avec granularité heure
counter:views:article:42:2024:12:09:15   # 15h

# Pattern général
counter:<metric>:<entity>:<id>[:<timestamp>]
```

**Implémentation** :

```python
def increment_counter(redis, metric, entity, entity_id, granularity='day'):
    """Incrémente compteur avec granularité temporelle"""
    now = datetime.now()

    if granularity == 'day':
        timestamp = now.strftime('%Y:%m:%d')
    elif granularity == 'hour':
        timestamp = now.strftime('%Y:%m:%d:%H')
    elif granularity == 'minute':
        timestamp = now.strftime('%Y:%m:%d:%H:%M')
    else:
        timestamp = ''

    key = f"counter:{metric}:{entity}:{entity_id}"
    if timestamp:
        key += f":{timestamp}"

    count = redis.incr(key)

    # TTL automatique selon granularité
    if granularity == 'day':
        redis.expire(key, 86400 * 90)  # 90 jours
    elif granularity == 'hour':
        redis.expire(key, 3600 * 24 * 7)  # 7 jours

    return count

# Utilisation
views_today = increment_counter(redis, "views", "article", 42, "day")
views_this_hour = increment_counter(redis, "views", "article", 42, "hour")

# Lecture agrégée
def get_views_range(redis, article_id, start_date, end_date):
    """Récupère vues sur une période"""
    total = 0
    current = start_date

    while current <= end_date:
        key = f"counter:views:article:{article_id}:{current.strftime('%Y:%m:%d')}"
        count = redis.get(key)
        if count:
            total += int(count)
        current += timedelta(days=1)

    return total
```

### Cas 6 : Tags et recherche

**Besoins** :
- Tags sur entités
- Recherche par tags
- Indices inversés

**Nommage** :

```bash
# Tags directs (Set)
tags:article:42              # SADD redis nosql database
tags:user:123:interests      # SADD python redis docker

# Index inversé (Set)
index:tag:redis              # SADD article:42 article:58 article:91
index:tag:python             # SADD article:42 article:101

# Pattern général
tags:<entity>:<id>           # Tags de l'entité
index:tag:<tag>              # Entités avec ce tag
```

**Implémentation** :

```python
def add_tags(redis, entity_type, entity_id, *tags):
    """Ajoute tags et met à jour indices"""
    entity_key = f"tags:{entity_type}:{entity_id}"
    entity_ref = f"{entity_type}:{entity_id}"

    for tag in tags:
        # Tag direct
        redis.sadd(entity_key, tag)

        # Index inversé
        index_key = f"index:tag:{tag.lower()}"
        redis.sadd(index_key, entity_ref)

def remove_tags(redis, entity_type, entity_id, *tags):
    """Supprime tags et met à jour indices"""
    entity_key = f"tags:{entity_type}:{entity_id}"
    entity_ref = f"{entity_type}:{entity_id}"

    for tag in tags:
        redis.srem(entity_key, tag)
        index_key = f"index:tag:{tag.lower()}"
        redis.srem(index_key, entity_ref)

def find_by_tags(redis, entity_type, *tags, operator='AND'):
    """Trouve entités par tags"""
    if operator == 'AND':
        # Intersection
        keys = [f"index:tag:{tag.lower()}" for tag in tags]
        return redis.sinter(*keys)
    else:  # OR
        # Union
        keys = [f"index:tag:{tag.lower()}" for tag in tags]
        return redis.sunion(*keys)

# Utilisation
add_tags(redis, "article", 42, "redis", "nosql", "cache")
add_tags(redis, "article", 43, "redis", "python")

# Recherche
redis_articles = find_by_tags(redis, "article", "redis")
# {'article:42', 'article:43'}

redis_and_python = find_by_tags(redis, "article", "redis", "python", operator='AND')
# {'article:43'}
```

## Versioning et évolution de schéma

### Besoin de versioning

```bash
# V1 : Structure initiale
user:123:profile → '{"name":"Alice","email":"alice@example.com"}'

# V2 : Ajout de champs
user:123:profile → '{"name":"Alice","email":"alice@example.com","phone":"+33..."}'

# Problème : Comment distinguer V1 de V2 ?
```

### Stratégies de versioning

#### 1. Version dans la clé (RECOMMANDÉ)

```bash
# Pattern
<type>:v<version>:<entity>:<id>

# Exemples
user:v1:123:profile
user:v2:123:profile
cache:v3:api:weather:paris
```

**Avantages** :
- ✅ Cohabitation de versions
- ✅ Migration progressive
- ✅ Rollback facile

**Migration** :

```python
def migrate_user_v1_to_v2(redis, user_id):
    """Migre user de V1 à V2"""
    old_key = f"user:v1:{user_id}:profile"
    new_key = f"user:v2:{user_id}:profile"

    # Lire V1
    old_data = redis.get(old_key)
    if not old_data:
        return False

    data = json.loads(old_data)

    # Transformer
    data['phone'] = None  # Nouveau champ
    data['version'] = 2

    # Écrire V2
    redis.set(new_key, json.dumps(data))

    # Optionnel : Supprimer V1
    # redis.delete(old_key)

    return True

# Lecture avec fallback
def get_user_profile(redis, user_id):
    """Lit profil avec fallback V2 → V1"""
    # Essaye V2
    v2_key = f"user:v2:{user_id}:profile"
    data = redis.get(v2_key)
    if data:
        return json.loads(data)

    # Fallback V1
    v1_key = f"user:v1:{user_id}:profile"
    data = redis.get(v1_key)
    if data:
        # Migration lazy
        migrate_user_v1_to_v2(redis, user_id)
        return json.loads(data)

    return None
```

#### 2. Version dans la valeur

```bash
# Clé sans version
user:123:profile

# Valeur avec version
{
    "_version": 2,
    "name": "Alice",
    "email": "alice@example.com",
    "phone": "+33..."
}
```

**Avantages** :
- ✅ Clés stables
- ✅ Backward compatibility possible

**Inconvénients** :
- ❌ Nécessite désérialisation pour lire version
- ❌ Migration sur place uniquement

#### 3. Metadata séparée

```bash
# Données
user:123:profile → {...}

# Metadata
meta:user:123:profile → '{"version":2,"schema":"profile_v2"}'
```

**Avantages** :
- ✅ Séparation concerns
- ✅ Metadata interrogeable sans lire data

**Inconvénients** :
- ❌ Deux requêtes
- ❌ Risque de désynchronisation

## Patterns avancés

### Composition de clés dynamique

```python
class KeyBuilder:
    """Builder pour construction cohérente de clés"""

    def __init__(self, separator=':'):
        self.parts = []
        self.separator = separator

    def add(self, *parts):
        """Ajoute parties"""
        self.parts.extend(str(p) for p in parts)
        return self

    def build(self):
        """Construit clé finale"""
        return self.separator.join(self.parts)

    @classmethod
    def user(cls, user_id, *extra):
        """Clé user"""
        return cls().add('user', user_id, *extra).build()

    @classmethod
    def cache(cls, source, *path):
        """Clé cache"""
        return cls().add('cache', source, *path).build()

    @classmethod
    def counter(cls, metric, entity, entity_id, timestamp=None):
        """Clé counter"""
        builder = cls().add('counter', metric, entity, entity_id)
        if timestamp:
            builder.add(timestamp)
        return builder.build()

# Utilisation
user_key = KeyBuilder.user(123, 'profile')
# "user:123:profile"

cache_key = KeyBuilder.cache('api', 'weather', 'paris')
# "cache:api:weather:paris"

counter_key = KeyBuilder.counter('views', 'article', 42, '2024:12:09')
# "counter:views:article:42:2024:12:09"
```

### Namespaces par environnement

```bash
# Pattern avec environnement
<env>:<type>:<entity>:<id>

# Développement
dev:user:123:profile
dev:cache:api:weather

# Staging
staging:user:123:profile
staging:cache:api:weather

# Production
prod:user:123:profile
prod:cache:api:weather

# Ou via databases séparées (DB 0, 1, 2)
```

**Configuration** :

```python
import os

class RedisClient:
    def __init__(self):
        self.env = os.getenv('ENVIRONMENT', 'dev')
        self.redis = redis.Redis()

    def make_key(self, *parts):
        """Ajoute prefix environnement"""
        return f"{self.env}:" + ":".join(str(p) for p in parts)

    def get(self, *parts):
        key = self.make_key(*parts)
        return self.redis.get(key)

    def set(self, *parts, value, **kwargs):
        key = self.make_key(*parts)
        return self.redis.set(key, value, **kwargs)

# Utilisation
client = RedisClient()
client.set('user', 123, 'profile', value='{"name":"Alice"}')
# Stocke: "dev:user:123:profile" ou "prod:user:123:profile"
```

### Multi-tenancy

```bash
# Pattern
<tenant_id>:<type>:<entity>:<id>

# Tenant ACME
acme:user:123:profile
acme:cache:data
acme:counter:views:article:42

# Tenant XYZ
xyz:user:456:profile
xyz:cache:data
xyz:counter:views:article:42
```

**Isolation complète** :

```python
class TenantRedisClient:
    def __init__(self, tenant_id):
        self.tenant_id = tenant_id
        self.redis = redis.Redis()

    def make_key(self, *parts):
        return f"{self.tenant_id}:" + ":".join(str(p) for p in parts)

    def get(self, *parts):
        return self.redis.get(self.make_key(*parts))

    def set(self, *parts, value, **kwargs):
        return self.redis.set(self.make_key(*parts), value, **kwargs)

    def scan_tenant(self, pattern='*', count=100):
        """Scan uniquement les clés du tenant"""
        full_pattern = f"{self.tenant_id}:{pattern}"
        cursor = 0
        while True:
            cursor, keys = self.redis.scan(cursor, match=full_pattern, count=count)
            yield from keys
            if cursor == 0:
                break

# Utilisation
acme_client = TenantRedisClient('acme')
xyz_client = TenantRedisClient('xyz')

acme_client.set('user', 123, 'name', value='Alice')
xyz_client.set('user', 123, 'name', value='Bob')

# Isolation totale
print(acme_client.get('user', 123, 'name'))  # "Alice"
print(xyz_client.get('user', 123, 'name'))   # "Bob"
```

## Recherche et filtrage

### SCAN avec patterns

```bash
# Scan par namespace
SCAN 0 MATCH user:* COUNT 100
SCAN 0 MATCH cache:api:* COUNT 100
SCAN 0 MATCH session:* COUNT 100

# Wildcards
SCAN 0 MATCH user:*:profile COUNT 100       # Tous les profils
SCAN 0 MATCH counter:views:article:* COUNT 100  # Tous compteurs vues articles

# Un seul wildcard par niveau
SCAN 0 MATCH user:*:*                       # user:123:profile, user:123:settings, etc.
```

**Implémentation sûre** :

```python
def scan_keys(redis, pattern, count=100):
    """Scan avec pattern, retourne generator"""
    cursor = 0
    while True:
        cursor, keys = redis.scan(cursor, match=pattern, count=count)
        yield from keys
        if cursor == 0:
            break

# Utilisation
for key in scan_keys(redis, "user:*:profile"):
    profile = redis.get(key)
    process(profile)

# Collecte en liste (attention mémoire !)
user_profiles = list(scan_keys(redis, "user:*:profile"))
```

### KEYS : À éviter en production

```bash
# ❌ BLOQUE Redis en production
KEYS user:*

# ✅ Utiliser SCAN à la place
SCAN 0 MATCH user:* COUNT 100
```

**Pourquoi KEYS est dangereux** :

```
KEYS avec 1 million de clés:
├─ Bloque Redis pendant ~1 seconde
├─ Aucune requête servie pendant ce temps
└─ Peut causer timeout des clients

SCAN avec 1 million de clés:
├─ Traite par batch de 100 (configurable)
├─ Chaque itération < 1ms
└─ Requêtes clients servies entre les batchs
```

### Suppression par pattern

```python
def delete_by_pattern(redis, pattern, batch_size=100):
    """Supprime clés par pattern (safe avec SCAN)"""
    cursor = 0
    total_deleted = 0

    while True:
        cursor, keys = redis.scan(cursor, match=pattern, count=batch_size)

        if keys:
            # Supprime batch
            redis.delete(*keys)
            total_deleted += len(keys)

        if cursor == 0:
            break

    return total_deleted

# Utilisation
deleted = delete_by_pattern(redis, "cache:api:*")
print(f"Deleted {deleted} cache entries")
```

## Performance et optimisations

### Longueur des clés

```bash
# Impact mémoire
# Overhead par clé : ~100 bytes (RedisObject + dict entry + key)

# Clé courte
u:123:p             → 7 bytes clé + 100 bytes overhead = 107 bytes

# Clé descriptive
user:123:profile    → 17 bytes clé + 100 bytes overhead = 117 bytes

# Différence : 10 bytes par clé
# Sur 10M clés : 100 MB de différence

# Recommandation : Préférer la lisibilité
# 100 MB sur 10M clés est négligeable vs maintenabilité
```

**Équilibre lisibilité/performance** :

```bash
# ❌ Trop court (illisible)
u:123:p
c:a:w:p

# ✅ Bon équilibre
user:123:profile
cache:api:weather:paris

# ⚠️ Trop long (rare mais possible)
application:production:cache:api:external:weather:current:paris:france:europe
```

### Hash slots et clustering

Pour Redis Cluster, les clés sont distribuées selon **hash slots** :

```bash
# Hash slot calculé sur la clé entière
user:123:profile        → Slot 15495
user:123:settings       → Slot 10234  # Slots différents !

# Problème : Multi-key ops impossibles
MGET user:123:profile user:123:settings
# (error) CROSSSLOT Keys in request don't hash to the same slot
```

**Solution : Hash tags** :

```bash
# Utiliser {} pour forcer même slot
user:{123}:profile      → Slot basé sur "123"
user:{123}:settings     → Slot basé sur "123"  # Même slot !

# Multi-key operations OK
MGET user:{123}:profile user:{123}:settings
# ✅ Fonctionne dans un cluster
```

**Pattern recommandé pour cluster** :

```bash
# Grouper par entité avec hash tag
user:{user_id}:profile
user:{user_id}:settings
user:{user_id}:preferences

# Opérations multi-clés possibles
MGET user:{123}:profile user:{123}:settings
DEL user:{123}:profile user:{123}:settings user:{123}:preferences
```

### Préfixes et ACLs

Redis ACLs permettent des règles par pattern :

```bash
# redis.conf ou CONFIG SET

# User "cache-reader" : Lecture seule sur cache:*
ACL SETUSER cache-reader on >password ~cache:* +get +mget

# User "session-manager" : Full sur session:*
ACL SETUSER session-manager on >password ~session:* +@all

# User "admin" : Full sur tout
ACL SETUSER admin on >password ~* +@all
```

**Stratégie de nommage pour ACLs** :

```bash
# Organiser par privilège
cache:*           → Lecture par tous
session:*         → Gestion par session-service
admin:*           → Admin uniquement
metrics:*         → Lecture par monitoring

# Exemple règles
ACL SETUSER app on >pass ~cache:* ~session:* +get +set +del
ACL SETUSER monitoring on >pass ~metrics:* +get +scan
ACL SETUSER admin on >pass ~* +@all
```

## Anti-patterns à éviter

### 1. Clés trop génériques

```bash
# ❌ MAUVAIS : Collisions garanties
data
temp
cache
config

# ✅ BON : Spécifique
user:123:data
temp:upload:abc123
cache:api:result:xyz
config:app:database
```

### 2. Séparateurs incohérents

```bash
# ❌ MAUVAIS : Mélange séparateurs
user:123/profile
cache|api:result
session_abc123

# ✅ BON : Cohérent
user:123:profile
cache:api:result
session:abc123
```

### 3. ID non uniques

```bash
# ❌ MAUVAIS : ID peut collisionner
user:profile        # Quel user ?
article:data        # Quel article ?

# ✅ BON : ID unique obligatoire
user:123:profile
article:42:data
```

### 4. Espaces dans les clés

```bash
# ❌ MAUVAIS : Espaces = problèmes
"user 123 profile"
"cache api weather"

# ✅ BON : Pas d'espaces
user:123:profile
cache:api:weather
```

### 5. Caractères spéciaux

```bash
# ⚠️ À éviter
user:123:@email
cache:api:result?query=1
session:abc#123

# ✅ Préférer
user:123:email
cache:api:result:query_1
session:abc_123
```

### 6. Clés dynamiques non maîtrisées

```python
# ❌ MAUVAIS : User input direct
search_term = request.get('q')
key = f"cache:search:{search_term}"
# Risque : Injection, clés infinies si pas de limite

# ✅ BON : Validation et hashing
search_term = request.get('q')
if len(search_term) > 100:
    raise ValueError("Search term too long")

# Hash pour clés longues/complexes
search_hash = hashlib.md5(search_term.encode()).hexdigest()
key = f"cache:search:{search_hash}"
```

### 7. Pas de TTL sur données temporaires

```bash
# ❌ MAUVAIS : Fuite mémoire
SET session:abc123 "{data}"
SET cache:temp:xyz "{data}"

# ✅ BON : TTL systématique
SETEX session:abc123 1800 "{data}"
SETEX cache:temp:xyz 300 "{data}"
```

### 8. Hiérarchie trop profonde

```bash
# ❌ MAUVAIS : 10 niveaux
app:prod:service:api:cache:db:table:users:country:france:city:paris:user:123

# ✅ BON : Maximum 5 niveaux
app:cache:users:france:paris:123
```

## Documentation et gouvernance

### Registry de namespaces

Maintenir un document centralé :

```yaml
# namespaces.yml
namespaces:
  user:
    description: "Données utilisateurs"
    pattern: "user:<id>:<attribute>"
    examples:
      - "user:123:profile"
      - "user:123:settings"
    ttl: "permanent"
    owner: "user-service"

  session:
    description: "Sessions HTTP"
    pattern: "session:<session_id>[:<attribute>]"
    examples:
      - "session:abc123"
      - "session:abc123:cart"
    ttl: "1800 seconds"
    owner: "auth-service"

  cache:
    description: "Cache multi-sources"
    pattern: "cache:<source>:<path>"
    examples:
      - "cache:db:users:123"
      - "cache:api:weather:paris"
    ttl: "variable (300-3600s)"
    owner: "all services"
```

### Validation programmatique

```python
import re

class KeyValidator:
    """Valide les clés selon conventions"""

    PATTERNS = {
        'user': r'^user:\d+:[a-z_]+$',
        'session': r'^session:[a-f0-9]+(?::[a-z_]+)?$',
        'cache': r'^cache:[a-z]+:.+$',
        'counter': r'^counter:[a-z_]+:[a-z]+:\d+(?::\d{4}:\d{2}:\d{2})?$',
    }

    @classmethod
    def validate(cls, key):
        """Valide format de clé"""
        namespace = key.split(':')[0]

        if namespace not in cls.PATTERNS:
            raise ValueError(f"Unknown namespace: {namespace}")

        pattern = cls.PATTERNS[namespace]
        if not re.match(pattern, key):
            raise ValueError(f"Invalid key format: {key}")

        return True

    @classmethod
    def suggest_fix(cls, key):
        """Suggère correction"""
        # Implémentation de suggestions...
        pass

# Utilisation
try:
    KeyValidator.validate("user:123:profile")  # ✅
    KeyValidator.validate("user:abc:profile")  # ❌ ValueError
except ValueError as e:
    print(f"Invalid key: {e}")
```

### Code review checklist

```markdown
## Redis Key Naming Review

Avant de merger, vérifier :

☐ Séparateur cohérent (`:` partout)
☐ Namespace clair et documenté
☐ ID unique présent
☐ Maximum 5 niveaux de profondeur
☐ Pas d'espaces ni caractères spéciaux
☐ TTL défini si données temporaires
☐ Pattern compatible SCAN
☐ Compatible Redis Cluster (hash tags si nécessaire)
☐ Validation des inputs utilisateur
☐ Documentation à jour
```

## Résumé : Best practices

### Checklist complète

```
✅ Toujours utiliser `:` comme séparateur
✅ Pattern : <type>:<entity>:<id>:<attribute>
✅ ID unique obligatoire
✅ Maximum 4-5 niveaux de profondeur
✅ Namespace par type de données
✅ TTL sur données temporaires
✅ Validation des inputs
✅ Hash tags pour clustering ({id})
✅ SCAN au lieu de KEYS
✅ Documentation centralisée
✅ Versioning pour évolution schéma
✅ Builder/helper pour cohérence
✅ ACLs par namespace
✅ Monitoring par pattern
```

### Pattern de référence

```bash
# Standard recommandé
<namespace>:<entity>:<id>[:<attribute>][:<timestamp>]

# Exemples
user:123:profile
session:abc123:data
cache:api:weather:paris
counter:views:article:42:2024:12:09
lock:resource:database
queue:emails
stream:events:orders

# Avec versioning
user:v2:123:profile
cache:v3:api:result

# Multi-tenant
tenant:acme:user:123:profile

# Cluster-ready
user:{123}:profile
user:{123}:settings
```

## Conclusion

Le nommage des clés Redis n'est pas qu'une question de convention esthétique. C'est une **architecture de données** qui impacte :

- **Maintenabilité** : Retrouver et comprendre les données
- **Performance** : Recherche, clustering, ACLs
- **Scalabilité** : Partitionnement prévisible
- **Fiabilité** : Éviter collisions et bugs
- **Sécurité** : ACLs granulaires

Investir du temps dans la définition de conventions solides dès le début d'un projet Redis évite des refactoring coûteux plus tard. Les patterns présentés ici sont éprouvés en production et constituent une base solide pour vos projets.

La section suivante abordera SCAN vs KEYS et comment ne jamais bloquer la production lors de l'exploration du keyspace.

⏭️ [SCAN vs KEYS : Ne jamais bloquer la production](/04-cycle-vie-donnee/06-scan-vs-keys-production.md)
