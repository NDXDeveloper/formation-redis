🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2 RedisJSON : Stocker et manipuler des documents JSON natifs

## Introduction

Avec Redis Core, stocker des objets JSON nécessite de les **sérialiser en String**, ce qui entraîne plusieurs problèmes : perte de structure, impossibilité de modifier partiellement les données, et logique applicative complexe pour maintenir la cohérence.

**RedisJSON** résout ces problèmes en permettant de stocker et manipuler des **documents JSON nativement**, avec support complet de JSONPath pour accéder et modifier des sous-parties du document de manière atomique.

---

## Pourquoi RedisJSON ?

### Le problème avec Redis Core

```bash
# Stocker un profil utilisateur en String
SET user:1001 '{"id":1001,"name":"Alice","email":"alice@example.com","preferences":{"theme":"dark","notifications":true},"tags":["premium","verified"]}'

# 🔴 Pour modifier uniquement le thème :
# 1. GET user:1001
# 2. Désérialiser JSON (côté application)
# 3. Modifier preferences.theme
# 4. Resérialiser JSON
# 5. SET user:1001 <nouveau_json>

# Problèmes :
# - Race condition (2 clients modifient simultanément)
# - Transfert réseau du JSON complet (aller-retour)
# - Logic applicative complexe
# - Pas atomique sans WATCH/MULTI/EXEC
```

### La solution RedisJSON

```bash
# Stocker le document JSON nativement
JSON.SET user:1001 $ '{"id":1001,"name":"Alice","email":"alice@example.com","preferences":{"theme":"dark","notifications":true},"tags":["premium","verified"]}'

# ✅ Modifier uniquement le thème (opération atomique)
JSON.SET user:1001 $.preferences.theme '"light"'

# Résultat : 1 commande, atomique, pas de transfert du JSON complet
```

---

## Installation et vérification

### Vérifier que RedisJSON est disponible

```bash
# Vérifier les modules chargés
redis-cli MODULE LIST

# Devrait contenir :
# 1) 1) "name"
#    2) "ReJSON"
#    3) "ver"
#    4) 20400  # Version 2.4.0
```

### Avec Docker (Redis Stack)

```bash
# Démarrer Redis Stack avec RedisJSON
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest

# Tester RedisJSON
redis-cli JSON.SET test $ '{"hello":"world"}'
redis-cli JSON.GET test
# Résultat: {"hello":"world"}
```

---

## Syntaxe JSONPath

RedisJSON utilise **JSONPath** pour naviguer dans les documents JSON.

### Syntaxe de base

| Syntaxe | Description | Exemple |
|---------|-------------|---------|
| `$` | Racine du document | `$` |
| `.` | Accès à une propriété | `$.name` |
| `[]` | Accès par index (tableau) | `$.tags[0]` |
| `[*]` | Tous les éléments d'un tableau | `$.tags[*]` |
| `..` | Recherche récursive | `$..email` |
| `[start:end]` | Slice de tableau | `$.items[0:3]` |

### Exemples de JSONPath

```json
{
  "user": {
    "id": 1001,
    "name": "Alice",
    "contacts": [
      {"type": "email", "value": "alice@example.com"},
      {"type": "phone", "value": "+33612345678"}
    ]
  }
}
```

```bash
# Récupérer le nom
$.user.name  # "Alice"

# Récupérer tous les contacts
$.user.contacts  # [{"type":"email",...}, {"type":"phone",...}]

# Récupérer le premier contact
$.user.contacts[0]  # {"type":"email","value":"alice@example.com"}

# Récupérer toutes les valeurs de contacts
$.user.contacts[*].value  # ["alice@example.com", "+33612345678"]

# Recherche récursive de tous les "type"
$..type  # ["email", "phone"]
```

---

## Commandes fondamentales

### JSON.SET : Créer ou remplacer un document

```bash
# Créer un document à la racine
JSON.SET product:101 $ '{"name":"Laptop","price":1299.99,"stock":45}'

# Remplacer un champ spécifique
JSON.SET product:101 $.price 1199.99

# Créer un nouveau champ
JSON.SET product:101 $.discount 0.15

# Remplacer un sous-document
JSON.SET product:101 $.specs '{"ram":"16GB","cpu":"Intel i7"}'
```

**Options** :

```bash
# NX : Créer seulement si la clé n'existe pas
JSON.SET product:102 $ '{"name":"Mouse"}' NX
# OK si product:102 n'existait pas, (nil) sinon

# XX : Modifier seulement si la clé existe
JSON.SET product:102 $.price 29.99 XX
# OK si product:102 existait, (nil) sinon
```

---

### JSON.GET : Récupérer un document

```bash
# Récupérer le document complet
JSON.GET product:101

# Résultat :
# {"name":"Laptop","price":1199.99,"stock":45,"discount":0.15,"specs":{"ram":"16GB","cpu":"Intel i7"}}

# Récupérer un champ spécifique
JSON.GET product:101 $.name
# ["Laptop"]  # Retourne toujours un tableau

# Récupérer plusieurs chemins
JSON.GET product:101 $.name $.price $.stock
# {"$.name":["Laptop"],"$.price":[1199.99],"$.stock":[45]}

# Formater avec indentation
JSON.GET product:101 $ INDENT "\t" NEWLINE "\n" SPACE " "
```

**Options de formatage** :

```bash
# INDENT, NEWLINE, SPACE pour un JSON lisible
JSON.GET user:1001 $ INDENT "  " NEWLINE "\n"

# Résultat formaté :
# {
#   "id": 1001,
#   "name": "Alice",
#   ...
# }
```

---

### JSON.DEL : Supprimer un document ou un champ

```bash
# Supprimer un champ
JSON.DEL product:101 $.discount
# Retourne 1 (nombre d'éléments supprimés)

# Supprimer un élément de tableau
JSON.DEL product:101 $.tags[0]

# Supprimer le document complet
JSON.DEL product:101
# Équivalent à DEL product:101
```

---

### JSON.TYPE : Obtenir le type d'une valeur

```bash
JSON.SET demo $ '{"string":"hello","number":42,"bool":true,"null":null,"array":[1,2,3],"object":{"key":"value"}}'

JSON.TYPE demo $.string  # ["string"]
JSON.TYPE demo $.number  # ["integer"]
JSON.TYPE demo $.bool    # ["boolean"]
JSON.TYPE demo $.null    # ["null"]
JSON.TYPE demo $.array   # ["array"]
JSON.TYPE demo $.object  # ["object"]
```

---

## Manipulation de chaînes

### JSON.STRLEN : Longueur d'une chaîne

```bash
JSON.SET user:1001 $ '{"name":"Alice","bio":"Software engineer"}'

JSON.STRLEN user:1001 $.name
# [5]  # "Alice" a 5 caractères

JSON.STRLEN user:1001 $.bio
# [17]
```

### JSON.STRAPPEND : Ajouter à une chaîne

```bash
JSON.STRAPPEND user:1001 $.bio '" and Redis enthusiast"'

JSON.GET user:1001 $.bio
# ["Software engineer and Redis enthusiast"]
```

---

## Manipulation de nombres

### JSON.NUMINCRBY : Incrémenter un nombre

```bash
JSON.SET product:101 $ '{"name":"Laptop","price":1299.99,"stock":45,"views":0}'

# Incrémenter les vues
JSON.NUMINCRBY product:101 $.views 1
# [1]

JSON.NUMINCRBY product:101 $.views 10
# [11]

# Décrémenter le stock
JSON.NUMINCRBY product:101 $.stock -1
# [44]
```

### JSON.NUMMULTBY : Multiplier un nombre

```bash
# Appliquer une remise de 10% (multiplier par 0.9)
JSON.NUMMULTBY product:101 $.price 0.9
# [1169.991]  # 1299.99 * 0.9
```

**Cas d'usage** : Promotions, ajustements de prix

```bash
# Black Friday : -20% sur tous les produits
# (À faire pour chaque produit)
JSON.NUMMULTBY product:101 $.price 0.8
JSON.NUMMULTBY product:102 $.price 0.8
```

---

## Manipulation de tableaux

### JSON.ARRAPPEND : Ajouter des éléments

```bash
JSON.SET user:1001 $ '{"name":"Alice","tags":["premium"]}'

# Ajouter un tag
JSON.ARRAPPEND user:1001 $.tags '"verified"'
# [2]  # Nouvelle taille du tableau

# Ajouter plusieurs tags
JSON.ARRAPPEND user:1001 $.tags '"active"' '"vip"'
# [4]

JSON.GET user:1001 $.tags
# [["premium","verified","active","vip"]]
```

### JSON.ARRINSERT : Insérer à une position

```bash
# Insérer "early-adopter" en position 1
JSON.ARRINSERT user:1001 $.tags 1 '"early-adopter"'
# [5]

JSON.GET user:1001 $.tags
# [["premium","early-adopter","verified","active","vip"]]
```

### JSON.ARRPOP : Retirer le dernier élément

```bash
# Retirer le dernier élément
JSON.ARRPOP user:1001 $.tags
# ["vip"]

# Retirer à l'index 0 (premier élément)
JSON.ARRPOP user:1001 $.tags 0
# ["premium"]

JSON.GET user:1001 $.tags
# [["early-adopter","verified","active"]]
```

### JSON.ARRTRIM : Conserver une portion du tableau

```bash
JSON.SET user:1001 $.history '["login","purchase","logout","login","purchase","logout","login"]'

# Conserver les 3 derniers éléments
JSON.ARRLEN user:1001 $.history  # [7]
JSON.ARRTRIM user:1001 $.history 4 6  # Index 4 à 6 (3 derniers)
# [3]

JSON.GET user:1001 $.history
# [["purchase","logout","login"]]
```

**Cas d'usage** : Limiter l'historique à N dernières actions

### JSON.ARRINDEX : Chercher un élément

```bash
JSON.SET user:1001 $.tags '["premium","verified","active"]'

# Trouver l'index de "verified"
JSON.ARRINDEX user:1001 $.tags '"verified"'
# [1]  # Trouvé à l'index 1

# Chercher un élément absent
JSON.ARRINDEX user:1001 $.tags '"vip"'
# [-1]  # Non trouvé
```

### JSON.ARRLEN : Taille du tableau

```bash
JSON.ARRLEN user:1001 $.tags
# [3]
```

---

## Manipulation d'objets

### JSON.OBJKEYS : Lister les clés d'un objet

```bash
JSON.SET product:101 $ '{"name":"Laptop","price":1299.99,"specs":{"ram":"16GB","cpu":"Intel i7"}}'

# Clés de la racine
JSON.OBJKEYS product:101 $
# [["name","price","specs"]]

# Clés du sous-objet specs
JSON.OBJKEYS product:101 $.specs
# [["ram","cpu"]]
```

### JSON.OBJLEN : Nombre de clés d'un objet

```bash
JSON.OBJLEN product:101 $
# [3]  # name, price, specs

JSON.OBJLEN product:101 $.specs
# [2]  # ram, cpu
```

---

## Cas d'usage modernes

### 1️⃣ Panier e-commerce avec mise à jour atomique

**Contexte** : Un utilisateur ajoute/modifie/supprime des articles dans son panier

```bash
# Créer un panier
JSON.SET cart:user:1001 $ '{
  "user_id": 1001,
  "created_at": "2024-12-09T10:00:00Z",
  "items": [],
  "total": 0,
  "currency": "EUR"
}'

# Ajouter un article
JSON.ARRAPPEND cart:user:1001 $.items '{
  "product_id": 101,
  "name": "Laptop Dell XPS",
  "quantity": 1,
  "unit_price": 1299.99,
  "subtotal": 1299.99
}'

# Mettre à jour le total
JSON.NUMINCRBY cart:user:1001 $.total 1299.99

# Ajouter un second article
JSON.ARRAPPEND cart:user:1001 $.items '{
  "product_id": 202,
  "name": "Mouse Logitech",
  "quantity": 2,
  "unit_price": 29.99,
  "subtotal": 59.98
}'

JSON.NUMINCRBY cart:user:1001 $.total 59.98

# Récupérer le panier complet
JSON.GET cart:user:1001 $

# Résultat :
# [{
#   "user_id": 1001,
#   "created_at": "2024-12-09T10:00:00Z",
#   "items": [
#     {"product_id": 101, "name": "Laptop Dell XPS", "quantity": 1, "unit_price": 1299.99, "subtotal": 1299.99},
#     {"product_id": 202, "name": "Mouse Logitech", "quantity": 2, "unit_price": 29.99, "subtotal": 59.98}
#   ],
#   "total": 1359.97,
#   "currency": "EUR"
# }]

# Modifier la quantité du premier article (index 0)
JSON.SET cart:user:1001 $.items[0].quantity 2
JSON.NUMMULTBY cart:user:1001 $.items[0].subtotal 2
# Recalculer le total (nécessite logique applicative ou Lua)

# Supprimer un article (index 1)
JSON.DEL cart:user:1001 $.items[1]
JSON.NUMINCRBY cart:user:1001 $.total -59.98
```

**Avantages** :
- ✅ Opérations atomiques (pas de race condition)
- ✅ Pas de transfert du JSON complet
- ✅ TTL pour expiration automatique (panier abandonné)

```bash
# Expirer le panier après 24h d'inactivité
EXPIRE cart:user:1001 86400
```

---

### 2️⃣ Configuration d'application dynamique

**Contexte** : Configuration modifiable à chaud sans redémarrage

```bash
# Configuration initiale
JSON.SET config:app $ '{
  "database": {
    "host": "db.example.com",
    "port": 5432,
    "pool_size": 20,
    "timeout_ms": 5000
  },
  "cache": {
    "enabled": true,
    "ttl_seconds": 3600
  },
  "features": {
    "new_ui": false,
    "beta_api": true
  },
  "limits": {
    "max_requests_per_minute": 1000,
    "max_upload_size_mb": 100
  }
}'

# Activer une feature flag
JSON.SET config:app $.features.new_ui 'true'

# Modifier une limite
JSON.SET config:app $.limits.max_requests_per_minute 1500

# Ajouter une nouvelle feature
JSON.SET config:app $.features.dark_mode 'true'

# L'application lit la config périodiquement
JSON.GET config:app $
```

**Pattern** : Les applications peuvent utiliser **Client-Side Caching** pour être notifiées automatiquement des changements.

---

### 3️⃣ Profil utilisateur enrichi

**Contexte** : Profil avec préférences, historique, badges

```bash
# Créer un profil complet
JSON.SET user:1001 $ '{
  "id": 1001,
  "username": "alice_dev",
  "email": "alice@example.com",
  "created_at": "2024-01-15T10:00:00Z",
  "profile": {
    "first_name": "Alice",
    "last_name": "Dubois",
    "avatar_url": "https://cdn.example.com/avatars/alice.jpg",
    "bio": "Full-stack developer | Redis enthusiast"
  },
  "preferences": {
    "theme": "dark",
    "language": "fr",
    "notifications": {
      "email": true,
      "push": false,
      "sms": false
    }
  },
  "stats": {
    "posts_count": 42,
    "followers_count": 256,
    "following_count": 89
  },
  "badges": ["early-adopter", "contributor"],
  "last_login": "2024-12-09T08:30:00Z"
}'

# L'utilisateur se connecte → Incrémenter le compteur de connexions
JSON.SET user:1001 $.stats.login_count 1  # Initialiser si n'existe pas
JSON.NUMINCRBY user:1001 $.stats.login_count 1

# Mise à jour de la dernière connexion
JSON.SET user:1001 $.last_login '"2024-12-09T09:15:00Z"'

# L'utilisateur poste du contenu
JSON.NUMINCRBY user:1001 $.stats.posts_count 1

# Ajouter un badge
JSON.ARRAPPEND user:1001 $.badges '"verified"'

# Changer le thème
JSON.SET user:1001 $.preferences.theme '"light"'

# Désactiver les notifications email
JSON.SET user:1001 $.preferences.notifications.email 'false'

# Récupérer uniquement les préférences
JSON.GET user:1001 $.preferences
# [{"theme":"light","language":"fr","notifications":{"email":false,"push":false,"sms":false}}]

# Récupérer uniquement le nom complet
JSON.GET user:1001 $.profile.first_name $.profile.last_name
# {"$.profile.first_name":["Alice"],"$.profile.last_name":["Dubois"]}
```

**Optimisation** : Utiliser un TTL pour les sessions

```bash
# Session active : renouveler le TTL à chaque action
EXPIRE user:1001:session 1800  # 30 minutes
```

---

### 4️⃣ Cache de résultats API complexes

**Contexte** : Cacher des réponses API avec données imbriquées

```bash
# Réponse API /api/orders/12345
JSON.SET cache:api:orders:12345 $ '{
  "order_id": 12345,
  "status": "shipped",
  "customer": {
    "id": 1001,
    "name": "Alice Dubois",
    "email": "alice@example.com"
  },
  "items": [
    {
      "product_id": 101,
      "name": "Laptop",
      "quantity": 1,
      "price": 1299.99
    },
    {
      "product_id": 202,
      "name": "Mouse",
      "quantity": 2,
      "price": 29.99
    }
  ],
  "shipping": {
    "address": "123 Rue de la Paix, Paris",
    "carrier": "Chronopost",
    "tracking": "CP12345678"
  },
  "total": 1359.97,
  "cached_at": "2024-12-09T10:00:00Z"
}'

# Définir un TTL (expiration après 5 minutes)
EXPIRE cache:api:orders:12345 300

# Si le statut change → Mettre à jour partiellement
JSON.SET cache:api:orders:12345 $.status '"delivered"'
JSON.SET cache:api:orders:12345 $.shipping.delivered_at '"2024-12-09T15:30:00Z"'

# Invalider le cache si nécessaire
DEL cache:api:orders:12345
```

**Pattern** : Cache-Aside avec invalidation sélective

```javascript
// Pseudo-code application
async function getOrder(orderId) {
  const cacheKey = `cache:api:orders:${orderId}`;

  // Tenter de récupérer du cache
  let order = await redis.json.get(cacheKey, '$');

  if (!order) {
    // Cache miss → Récupérer de la DB
    order = await database.getOrder(orderId);

    // Mettre en cache
    await redis.json.set(cacheKey, '$', order);
    await redis.expire(cacheKey, 300);  // 5 minutes
  }

  return order;
}
```

---

### 5️⃣ Événements et audit trail

**Contexte** : Stocker des événements avec contexte riche

```bash
# Événement : Connexion utilisateur
JSON.SET event:20241209_100530_login_1001 $ '{
  "event_type": "user_login",
  "timestamp": "2024-12-09T10:05:30Z",
  "user_id": 1001,
  "username": "alice_dev",
  "context": {
    "ip": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "device": "desktop",
    "location": {
      "country": "FR",
      "city": "Paris"
    }
  },
  "metadata": {
    "session_id": "sess_abc123",
    "referrer": "https://example.com/login"
  }
}'

# Événement : Achat
JSON.SET event:20241209_103000_purchase_12345 $ '{
  "event_type": "purchase",
  "timestamp": "2024-12-09T10:30:00Z",
  "order_id": 12345,
  "user_id": 1001,
  "amount": 1359.97,
  "currency": "EUR",
  "items": [
    {"product_id": 101, "quantity": 1},
    {"product_id": 202, "quantity": 2}
  ],
  "payment": {
    "method": "credit_card",
    "last4": "1234",
    "success": true
  }
}'

# Rechercher tous les événements d'un utilisateur (nécessite RediSearch)
# Voir section 3.3 pour l'indexation
```

**Avec RediSearch** (prévisualisation) :

```bash
# Créer un index sur les événements
FT.CREATE idx:events
  ON JSON
  PREFIX 1 event:
  SCHEMA
    $.user_id AS user_id NUMERIC
    $.event_type AS event_type TAG
    $.timestamp AS timestamp NUMERIC SORTABLE

# Rechercher tous les achats de l'utilisateur 1001
FT.SEARCH idx:events "@user_id:[1001 1001] @event_type:{purchase}"
```

---

## Performance : RedisJSON vs String sérialisé

### Benchmark : GET/SET complet

```bash
# Test : Document JSON de 2KB, 100K opérations

# Redis Core (String)
SET user:1001 '{"id":1001,...}'  # 2KB
# Throughput : ~80,000 ops/sec

# RedisJSON
JSON.SET user:1001 $ '{"id":1001,...}'
# Throughput : ~75,000 ops/sec (-6%)
```

**Conclusion** : Performance comparable pour GET/SET complet.

---

### Benchmark : Modification partielle

```bash
# Test : Modifier un champ dans un document de 2KB

# Redis Core (String) : GET + Modify + SET
# Temps : ~0.8ms (3 RTT réseau)
# Bande passante : 4KB transférés (2KB × 2)

# RedisJSON : JSON.SET partiel
# Temps : ~0.3ms (1 RTT réseau)
# Bande passante : ~50 bytes transférés

# Gain : 2.6x plus rapide, 98% de bande passante économisée
```

---

### Benchmark : Opérations sur tableaux

```bash
# Test : Ajouter un élément à un tableau de 100 éléments

# Redis Core :
# 1. GET (2KB)
# 2. Désérialiser
# 3. Array.push()
# 4. Sérialiser
# 5. SET (2.05KB)
# Temps : ~1.2ms

# RedisJSON :
# JSON.ARRAPPEND user:1001 $.items '{"id":101}'
# Temps : ~0.2ms

# Gain : 6x plus rapide
```

---

## Considérations de mémoire

### Overhead de RedisJSON

```bash
# Test : 1 million de documents (2KB chacun)

# Redis Core (String sérialisé) :
# 1M × 2KB = 2GB
# + Overhead Redis (pointeurs, metadata) = ~2.3GB

# RedisJSON :
# 1M × 2KB = 2GB
# + Overhead RedisJSON (structure interne) = ~2.6GB

# Différence : +13% de mémoire
```

**Conclusion** : RedisJSON consomme **10-15% de mémoire supplémentaire**, mais avec des gains de performance significatifs pour les modifications partielles.

---

## Compression et optimisation

### Compression automatique

RedisJSON utilise **msgpack** en interne pour compresser les données.

```bash
# Document non compressé (String) : 5000 bytes
SET doc:1 '{"field1":"value1","field2":"value2",...}'

# Même document en RedisJSON : ~3500 bytes (compression msgpack)
JSON.SET doc:1 $ '{"field1":"value1","field2":"value2",...}'

# Gain : ~30% de compression automatique
```

---

## Limitations et bonnes pratiques

### ⚠️ Limitations

1. **Taille maximale d'un document** : 512MB (mais recommandé < 1MB)
2. **Profondeur maximale** : 128 niveaux d'imbrication
3. **JSONPath** : Pas de support complet (expressions complexes limitées)
4. **Pas de joins** : Un document = une clé

### ✅ Bonnes pratiques

#### 1. Nommer les clés de manière cohérente

```bash
# ✅ Bon : Préfixe clair + ID
JSON.SET user:1001 $ '{...}'
JSON.SET order:12345 $ '{...}'
JSON.SET product:101 $ '{...}'

# ❌ Mauvais : Pas de convention
JSON.SET alice_profile $ '{...}'
JSON.SET order_data_12345 $ '{...}'
```

#### 2. Utiliser des TTL pour les données temporaires

```bash
# Session utilisateur : expire après 30 minutes
JSON.SET session:abc123 $ '{"user_id":1001,...}'
EXPIRE session:abc123 1800

# Cache API : expire après 5 minutes
JSON.SET cache:api:users:1001 $ '{...}'
EXPIRE cache:api:users:1001 300
```

#### 3. Limiter la taille des documents

```bash
# ✅ Bon : Document < 100KB
JSON.SET user:1001 $ '{"id":1001,"name":"Alice",...}'  # 5KB

# ⚠️ Attention : Document > 1MB
# → Envisager de découper en plusieurs clés
# user:1001:profile, user:1001:preferences, user:1001:history
```

#### 4. Utiliser les opérations atomiques

```bash
# ✅ Bon : Incrémentation atomique
JSON.NUMINCRBY user:1001 $.stats.views 1

# ❌ Mauvais : GET + Modify + SET (race condition)
views = JSON.GET user:1001 $.stats.views
views += 1
JSON.SET user:1001 $.stats.views views
```

#### 5. Préférer JSONPath précis

```bash
# ✅ Bon : Chemin précis (rapide)
JSON.GET user:1001 $.profile.name

# ⚠️ Attention : Recherche récursive (plus lent)
JSON.GET user:1001 $..name
```

---

## Intégration avec RediSearch

RedisJSON devient **encore plus puissant** combiné avec RediSearch.

```bash
# Créer un index sur des documents JSON
FT.CREATE idx:users
  ON JSON
  PREFIX 1 user:
  SCHEMA
    $.username AS username TEXT SORTABLE
    $.email AS email TAG
    $.stats.posts_count AS posts_count NUMERIC SORTABLE
    $.created_at AS created_at NUMERIC SORTABLE

# Maintenant, vous pouvez :
# - Rechercher par texte
# - Filtrer par champs
# - Trier
# - Agréger

# Exemple : Trouver les 10 utilisateurs les plus actifs
FT.SEARCH idx:users "*" SORTBY posts_count DESC LIMIT 0 10
```

**Voir Section 3.3** pour plus de détails sur RediSearch.

---

## Migration depuis Redis Core

### Stratégie de migration progressive

```javascript
// Fonction utilitaire pour migrer progressivement
async function migrateToJSON(key) {
  // 1. Récupérer la String existante
  const jsonString = await redis.get(key);

  if (!jsonString) return;

  // 2. Valider que c'est du JSON
  let data;
  try {
    data = JSON.parse(jsonString);
  } catch (e) {
    console.error(`Key ${key} is not valid JSON`);
    return;
  }

  // 3. Migrer vers RedisJSON
  await redis.json.set(key, '$', data);

  console.log(`Migrated ${key} to RedisJSON`);
}

// Migration par batch
const cursor = 0;
do {
  const [newCursor, keys] = await redis.scan(cursor, 'MATCH', 'user:*', 'COUNT', 100);

  for (const key of keys) {
    await migrateToJSON(key);
  }

  cursor = newCursor;
} while (cursor !== '0');
```

---

## Exemples d'intégration avec différents langages

### Python (redis-py)

```python
import redis
from redis.commands.json.path import Path

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Créer un document
r.json().set('user:1001', Path.root_path(), {
    'id': 1001,
    'name': 'Alice',
    'tags': ['premium']
})

# Récupérer
user = r.json().get('user:1001')
print(user)  # {'id': 1001, 'name': 'Alice', 'tags': ['premium']}

# Modifier partiellement
r.json().set('user:1001', '$.name', 'Alice Dubois')

# Ajouter à un tableau
r.json().arrappend('user:1001', '$.tags', 'verified')

# Incrémenter
r.json().set('user:1001', '$.login_count', 0)
r.json().numincrby('user:1001', '$.login_count', 1)
```

### Node.js (node-redis)

```javascript
import { createClient } from 'redis';

const client = await createClient()
  .on('error', err => console.log('Redis Client Error', err))
  .connect();

// Créer un document
await client.json.set('user:1001', '$', {
  id: 1001,
  name: 'Alice',
  tags: ['premium']
});

// Récupérer
const user = await client.json.get('user:1001');
console.log(user);

// Modifier partiellement
await client.json.set('user:1001', '$.name', '"Alice Dubois"');

// Ajouter à un tableau
await client.json.arrAppend('user:1001', '$.tags', '"verified"');

// Incrémenter
await client.json.numIncrBy('user:1001', '$.login_count', 1);
```

### Java (Jedis)

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.json.Path2;

Jedis jedis = new Jedis("localhost", 6379);

// Créer un document
Map<String, Object> user = new HashMap<>();
user.put("id", 1001);
user.put("name", "Alice");
user.put("tags", Arrays.asList("premium"));

jedis.jsonSet("user:1001", Path2.ROOT_PATH, user);

// Récupérer
Object result = jedis.jsonGet("user:1001");
System.out.println(result);

// Modifier partiellement
jedis.jsonSet("user:1001", new Path2("$.name"), "Alice Dubois");

// Incrémenter
jedis.jsonNumIncrBy("user:1001", new Path2("$.login_count"), 1);
```

---

## Comparaison : RedisJSON vs MongoDB

| Fonctionnalité | RedisJSON | MongoDB |
|----------------|-----------|---------|
| **Latence** | < 1ms (mémoire) | 5-50ms (disque + cache) |
| **Throughput** | 100K+ ops/sec | 10-50K ops/sec |
| **Requêtes complexes** | RediSearch requis | Natif (agrégations, joins) |
| **Persistance** | Optionnelle (RDB/AOF) | Par défaut (durable) |
| **Scaling** | Horizontal (Cluster) | Horizontal (Sharding) |
| **Atomicité** | Opérations simples | Transactions ACID |
| **Cas d'usage** | Cache, sessions, temps réel | Base principale, analytics |

**Conclusion** : RedisJSON pour **haute performance**, MongoDB pour **requêtes complexes**.

---

## Troubleshooting

### Erreur : "ReJSON module not loaded"

```bash
# Vérifier les modules chargés
MODULE LIST

# Si ReJSON absent, utiliser Redis Stack ou charger le module
redis-server --loadmodule /path/to/rejson.so
```

### Erreur : "Path does not exist"

```bash
# ❌ Tentative de modification d'un chemin inexistant
JSON.SET user:1001 $.preferences.theme '"dark"'
# (error) ERR new objects must be created at the root

# ✅ Créer l'objet parent d'abord
JSON.SET user:1001 $.preferences '{}'
JSON.SET user:1001 $.preferences.theme '"dark"'
```

### Erreur : "JSON string not ended properly"

```bash
# ❌ Guillemets non échappés
JSON.SET user:1001 $ '{"name":"Alice"}'  # Erreur si shell interprète mal

# ✅ Utiliser des guillemets simples ou échapper
JSON.SET user:1001 $ '{"name":"Alice"}'
# Ou
JSON.SET user:1001 $ "{\"name\":\"Alice\"}"
```

---

## Ressources

### Documentation officielle
- [RedisJSON Documentation](https://redis.io/docs/stack/json/)
- [RedisJSON Commands](https://redis.io/commands/?group=json)
- [JSONPath Syntax](https://redis.io/docs/stack/json/path/)

### Outils
- [Redis Insight](https://redis.io/docs/stack/insight/) - GUI avec support RedisJSON
- [RedisJSON Python Client](https://github.com/redis/redis-py)
- [RedisJSON Node.js Client](https://github.com/redis/node-redis)

---

## Résumé

**RedisJSON permet de** :
- ✅ Stocker des documents JSON nativement (pas de sérialisation)
- ✅ Modifier partiellement les données (opérations atomiques)
- ✅ Utiliser JSONPath pour naviguer dans les documents
- ✅ Économiser de la bande passante réseau (70-90%)
- ✅ Améliorer les performances (2-6x plus rapide pour modifications partielles)

**Cas d'usage idéaux** :
- 🛒 Paniers e-commerce
- 👤 Profils utilisateurs
- ⚙️ Configurations d'application
- 📊 Cache de réponses API
- 📝 Événements et audit trail

**Limitations** :
- +10-15% de mémoire vs String
- Documents < 1MB recommandé
- Pas de requêtes complexes sans RediSearch

---

**Prêt pour la recherche avancée ?** Passons à la section suivante : [3.3 RediSearch - Indexation et Full-text Search](./03-redisearch-indexation-fulltext.md)

⏭️ [RediSearch : Indexation et Full-text search](/03-structures-donnees-etendues/03-redisearch-indexation-fulltext.md)
