🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.4 Hashes : Représentation d'objets et optimisation mémoire

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Comment les Hashes représentent des objets avec plusieurs champs
- ✅ L'avantage mémoire des Hashes vs Strings JSON
- ✅ Les commandes essentielles pour manipuler les Hashes
- ✅ L'optimisation automatique (ziplist vs hashtable)
- ✅ Les cas d'usage réels (profils, produits, configurations)

---

## 📘 Les Hashes : Des dictionnaires de champs

### Qu'est-ce qu'un Hash dans Redis ?

Un **Hash** est une structure qui associe des **champs (fields)** à des **valeurs**, comme un **dictionnaire** ou un **objet** dans la plupart des langages de programmation.

```bash
# Visualisation d'un Hash
user:123 → {
    "name": "Alice",
    "email": "alice@example.com",
    "age": "30",
    "city": "Paris"
}
```

**Caractéristiques** :
- ✅ Chaque Hash peut contenir jusqu'à **2³² - 1 champs** (~4 milliards)
- ✅ Les champs sont des **strings**
- ✅ Les valeurs sont des **strings** (mais peuvent être des nombres)
- ✅ Accès à un champ en **O(1)**
- ✅ Parfait pour représenter des **objets** avec plusieurs attributs

### Pourquoi utiliser des Hashes ?

**Avantage #1 : Sémantique claire**
```bash
# ❌ Avec des Strings : Plusieurs clés pour un objet
SET user:123:name "Alice"
SET user:123:email "alice@example.com"
SET user:123:age "30"

# ✅ Avec un Hash : Un seul objet structuré
HSET user:123 name "Alice" email "alice@example.com" age "30"
```

**Avantage #2 : Économie de mémoire** (nous y reviendrons)

**Avantage #3 : Opérations atomiques sur des champs**
```bash
# Incrémenter un compteur dans un objet
HINCRBY user:123 login_count 1
```

---

## 🔧 Commandes de base

### HSET : Définir un ou plusieurs champs

```bash
# Créer un Hash avec un champ
127.0.0.1:6379> HSET user:123 name "Alice"
(integer) 1  # Nombre de champs ajoutés

# Ajouter d'autres champs
127.0.0.1:6379> HSET user:123 email "alice@example.com"
(integer) 1

127.0.0.1:6379> HSET user:123 age "30"
(integer) 1

# HSET multiple (Redis 4.0+) : définir plusieurs champs à la fois
127.0.0.1:6379> HSET user:456 name "Bob" email "bob@example.com" age "25" city "London"
(integer) 4

# Modifier un champ existant
127.0.0.1:6379> HSET user:123 age "31"
(integer) 0  # 0 = champ existant mis à jour (pas ajouté)

# Vérifier
127.0.0.1:6379> HGET user:123 age
"31"
```

**Note** : `HMSET` (Hash Multi SET) était utilisé avant Redis 4.0, mais est maintenant **déprécié**. Utilisez `HSET` à la place.

### HSETNX : SET seulement si le champ n'existe pas

```bash
# Définir uniquement si le champ n'existe pas
127.0.0.1:6379> HSETNX user:123 country "France"
(integer) 1  # Succès, champ créé

127.0.0.1:6379> HSETNX user:123 country "Germany"
(integer) 0  # Échec, champ existe déjà

127.0.0.1:6379> HGET user:123 country
"France"  # Valeur inchangée
```

### HGET : Récupérer un champ

```bash
# Récupérer une valeur
127.0.0.1:6379> HGET user:123 name
"Alice"

127.0.0.1:6379> HGET user:123 email
"alice@example.com"

# Si le champ n'existe pas
127.0.0.1:6379> HGET user:123 phone
(nil)

# Si la clé n'existe pas
127.0.0.1:6379> HGET user:999 name
(nil)
```

### HMGET : Récupérer plusieurs champs

```bash
# Récupérer plusieurs champs en une seule commande
127.0.0.1:6379> HMGET user:123 name email age
1) "Alice"
2) "alice@example.com"
3) "31"

# Les champs inexistants retournent (nil)
127.0.0.1:6379> HMGET user:123 name phone city
1) "Alice"
2) (nil)
3) (nil)
```

**Avantage** : Réduit les aller-retours réseau (un seul RTT au lieu de N).

### HGETALL : Récupérer tous les champs et valeurs

```bash
# Récupérer tout le Hash
127.0.0.1:6379> HGETALL user:123
1) "name"
2) "Alice"
3) "email"
4) "alice@example.com"
5) "age"
6) "31"
7) "country"
8) "France"

# Format : champ1, valeur1, champ2, valeur2, ...
```

⚠️ **Attention** : HGETALL est **O(N)** où N = nombre de champs. Sur un Hash avec 10 000 champs, cela peut être lent. Utilisez HSCAN pour de gros Hashes.

### HKEYS : Récupérer tous les champs

```bash
# Obtenir seulement les noms de champs
127.0.0.1:6379> HKEYS user:123
1) "name"
2) "email"
3) "age"
4) "country"
```

### HVALS : Récupérer toutes les valeurs

```bash
# Obtenir seulement les valeurs (sans les champs)
127.0.0.1:6379> HVALS user:123
1) "Alice"
2) "alice@example.com"
3) "31"
4) "France"
```

### HLEN : Nombre de champs

```bash
# Compter le nombre de champs
127.0.0.1:6379> HLEN user:123
(integer) 4

# Très rapide : O(1)
```

### HEXISTS : Vérifier l'existence d'un champ

```bash
# Vérifier si un champ existe
127.0.0.1:6379> HEXISTS user:123 email
(integer) 1  # 1 = existe

127.0.0.1:6379> HEXISTS user:123 phone
(integer) 0  # 0 = n'existe pas
```

### HDEL : Supprimer des champs

```bash
# Supprimer un champ
127.0.0.1:6379> HDEL user:123 country
(integer) 1  # Nombre de champs supprimés

# Supprimer plusieurs champs
127.0.0.1:6379> HDEL user:123 age email
(integer) 2

# Vérifier
127.0.0.1:6379> HGETALL user:123
1) "name"
2) "Alice"

# Supprimer un champ inexistant
127.0.0.1:6379> HDEL user:123 phone
(integer) 0  # Pas supprimé car inexistant
```

---

## 🔢 Opérations numériques

### HINCRBY : Incrémenter un champ entier

```bash
# Créer un compteur dans un Hash
127.0.0.1:6379> HSET stats:user:123 page_views "0"
(integer) 1

# Incrémenter
127.0.0.1:6379> HINCRBY stats:user:123 page_views 1
(integer) 1

127.0.0.1:6379> HINCRBY stats:user:123 page_views 5
(integer) 6

# Si le champ n'existe pas, il est créé à 0 puis incrémenté
127.0.0.1:6379> HINCRBY stats:user:123 clicks 10
(integer) 10

# Décrémenter avec un nombre négatif
127.0.0.1:6379> HINCRBY stats:user:123 clicks -3
(integer) 7
```

### HINCRBYFLOAT : Incrémenter un champ flottant

```bash
# Créer un champ avec décimales
127.0.0.1:6379> HSET account:123 balance "100.50"
(integer) 1

# Incrémenter avec décimales
127.0.0.1:6379> HINCRBYFLOAT account:123 balance 25.75
"126.25"

# Décrémenter
127.0.0.1:6379> HINCRBYFLOAT account:123 balance -10.5
"115.75"

# Gestion de la précision
127.0.0.1:6379> HINCRBYFLOAT account:123 balance 0.1
"115.85000000000001"  # Attention aux flottants !
```

⚠️ **Attention** : Comme en programmation, les flottants ont des imprécisions. Pour les montants financiers, multipliez par 100 et utilisez des entiers (centimes).

---

## 📊 Cas d'usage #1 : Profil utilisateur

### Stocker un profil complet

```bash
# Créer un profil utilisateur
127.0.0.1:6379> HSET user:alice \
  id "123" \
  username "alice" \
  email "alice@example.com" \
  first_name "Alice" \
  last_name "Dupont" \
  age "30" \
  city "Paris" \
  country "France" \
  created_at "2024-01-15" \
  status "active"
(integer) 10

# Récupérer tout le profil
127.0.0.1:6379> HGETALL user:alice
 1) "id"
 2) "123"
 3) "username"
 4) "alice"
 5) "email"
 6) "alice@example.com"
 7) "first_name"
 8) "Alice"
 9) "last_name"
10) "Dupont"
11) "age"
12) "30"
13) "city"
14) "Paris"
15) "country"
16) "France"
17) "created_at"
18) "2024-01-15"
19) "status"
20) "active"

# Récupérer seulement certains champs
127.0.0.1:6379> HMGET user:alice username email city
1) "alice"
2) "alice@example.com"
3) "Paris"

# Mettre à jour un champ
127.0.0.1:6379> HSET user:alice city "Lyon"
(integer) 0

# Incrémenter l'âge lors d'un anniversaire
127.0.0.1:6379> HINCRBY user:alice age 1
(integer) 31
```

**Code application** :
```python
def get_user_profile(username):
    key = f"user:{username}"
    profile = redis.hgetall(key)
    return {k.decode(): v.decode() for k, v in profile.items()}

def update_user_email(username, new_email):
    key = f"user:{username}"
    redis.hset(key, "email", new_email)
```

---

## 📦 Cas d'usage #2 : Produit e-commerce

```bash
# Créer une fiche produit
127.0.0.1:6379> HSET product:123 \
  name "MacBook Pro 14\"" \
  sku "MBPRO14-2024" \
  price "1999.99" \
  stock "15" \
  category "laptops" \
  brand "Apple" \
  weight_kg "1.6" \
  rating "4.8" \
  review_count "0"
(integer) 9

# Récupérer les infos essentielles
127.0.0.1:6379> HMGET product:123 name price stock
1) "MacBook Pro 14\""
2) "1999.99"
3) "15"

# Vente : décrémenter le stock
127.0.0.1:6379> HINCRBY product:123 stock -1
(integer) 14

# Ajout d'un avis : incrémenter le compteur
127.0.0.1:6379> HINCRBY product:123 review_count 1
(integer) 1

# Mise à jour du prix (promotion)
127.0.0.1:6379> HSET product:123 price "1799.99"
(integer) 0

# Vérifier si le produit a une description
127.0.0.1:6379> HEXISTS product:123 description
(integer) 0

# Ajouter la description
127.0.0.1:6379> HSET product:123 description "Puce M3 Pro, 18 Go RAM, SSD 512 Go"
(integer) 1
```

---

## ⚙️ Cas d'usage #3 : Configuration d'application

```bash
# Configuration globale de l'application
127.0.0.1:6379> HSET config:app \
  max_upload_size_mb "10" \
  session_timeout_sec "3600" \
  api_rate_limit "100" \
  maintenance_mode "false" \
  debug_enabled "false" \
  cache_ttl_sec "300"
(integer) 6

# Récupérer une config spécifique
127.0.0.1:6379> HGET config:app session_timeout_sec
"3600"

# Activer le mode maintenance
127.0.0.1:6379> HSET config:app maintenance_mode "true"
(integer) 0

# Récupérer toute la config
127.0.0.1:6379> HGETALL config:app
 1) "max_upload_size_mb"
 2) "10"
 3) "session_timeout_sec"
 4) "3600"
 5) "api_rate_limit"
 6) "100"
 7) "maintenance_mode"
 8) "true"
 9) "debug_enabled"
10) "false"
11) "cache_ttl_sec"
12) "300"
```

**Avantage** : Modification en temps réel sans redémarrage de l'application !

---

## 📈 Cas d'usage #4 : Compteurs et statistiques

```bash
# Statistiques d'un article de blog
127.0.0.1:6379> HSET stats:article:42 \
  views "0" \
  likes "0" \
  comments "0" \
  shares "0"
(integer) 4

# Utilisateur visite l'article
127.0.0.1:6379> HINCRBY stats:article:42 views 1
(integer) 1

# Utilisateur like l'article
127.0.0.1:6379> HINCRBY stats:article:42 likes 1
(integer) 1

# Utilisateur commente
127.0.0.1:6379> HINCRBY stats:article:42 comments 1
(integer) 1

# Récupérer toutes les stats
127.0.0.1:6379> HGETALL stats:article:42
1) "views"
2) "1"
3) "likes"
4) "1"
5) "comments"
6) "1"
7) "shares"
8) "0"

# Statistiques quotidiennes d'un utilisateur
127.0.0.1:6379> HSET daily:stats:user:123:2024-12-09 \
  login_count "1" \
  api_calls "0" \
  errors "0"
(integer) 3

127.0.0.1:6379> HINCRBY daily:stats:user:123:2024-12-09 api_calls 1
(integer) 1

# Ajouter un TTL pour nettoyer les vieilles stats
127.0.0.1:6379> EXPIRE daily:stats:user:123:2024-12-09 2592000
(integer) 1  # Expire dans 30 jours
```

---

## 💾 Optimisation mémoire : Le secret des Hashes

### Hash encoding : ziplist vs hashtable

Redis utilise deux encodages internes pour les Hashes selon leur taille :

**1. Ziplist (compact)** - Utilisé quand :
- Nombre de champs ≤ 512 (configurable : `hash-max-ziplist-entries`)
- Taille de chaque valeur ≤ 64 bytes (configurable : `hash-max-ziplist-value`)

**2. Hashtable (standard)** - Utilisé quand les limites ziplist sont dépassées

```bash
# Vérifier l'encodage interne
127.0.0.1:6379> HSET small:hash field1 "value1" field2 "value2"
(integer) 2

127.0.0.1:6379> OBJECT ENCODING small:hash
"ziplist"  # Encodage compact

# Créer un Hash avec beaucoup de champs
127.0.0.1:6379> HSET large:hash field1 "value1"
(integer) 1

# Ajouter 600 champs (dépasse la limite de 512)
# ... (imaginez 600 HSET)

127.0.0.1:6379> OBJECT ENCODING large:hash
"hashtable"  # Encodage standard
```

### Économie de mémoire : Démonstration

**Scénario** : Stocker 1 million de profils utilisateurs avec 5 champs chacun.

**Option 1 : Strings séparées**
```bash
# 5 millions de clés !
SET user:1:name "Alice"
SET user:1:email "alice@example.com"
SET user:1:age "30"
SET user:1:city "Paris"
SET user:1:status "active"
# × 1 million d'utilisateurs
```

**Overhead** : Environ **90 bytes par clé** (pointeur, métadonnées Redis)
- Total overhead : 5 000 000 × 90 = **450 MB** juste pour les métadonnées !

**Option 2 : String JSON**
```bash
SET user:1 '{"name":"Alice","email":"alice@example.com","age":"30","city":"Paris","status":"active"}'
# × 1 million d'utilisateurs
```

**Overhead** : 90 bytes × 1 000 000 = **90 MB** pour les métadonnées
- Mais : Pas d'accès granulaire (doit parser tout le JSON à chaque fois)

**Option 3 : Hashes (optimal)**
```bash
HSET user:1 name "Alice" email "alice@example.com" age "30" city "Paris" status "active"
# × 1 million d'utilisateurs
```

**Overhead** : 90 bytes × 1 000 000 = **90 MB** pour les métadonnées
- **Bonus** : Accès direct à chaque champ sans parsing
- **Bonus** : Opérations atomiques (HINCRBY, etc.)
- **Bonus** : Ziplist pour économiser encore plus si < 512 champs

**Tableau comparatif** :

| Approche | Clés Redis | Overhead métadonnées | Accès granulaire | Opérations atomiques |
|----------|------------|---------------------|------------------|---------------------|
| Strings séparées | 5 000 000 | ~450 MB | ✅ Oui | ❌ Non |
| String JSON | 1 000 000 | ~90 MB | ❌ Non (parsing) | ❌ Non |
| **Hashes** | **1 000 000** | **~90 MB** | **✅ Oui** | **✅ Oui** |

**Conclusion** : Les Hashes sont le meilleur compromis !

---

## 🔍 HSCAN : Scanner de gros Hashes

Comme SCAN pour les clés, HSCAN permet d'itérer sur un Hash sans bloquer Redis.

```bash
# Créer un Hash avec beaucoup de champs
127.0.0.1:6379> HSET products:catalog \
  item1 "Product 1" \
  item2 "Product 2" \
  item3 "Product 3"
# ... (imaginez 10 000 champs)

# Scanner le Hash par batches
127.0.0.1:6379> HSCAN products:catalog 0 COUNT 10
1) "17"  # Curseur suivant
2) 1) "item1"
   2) "Product 1"
   3) "item2"
   4) "Product 2"
   5) "item3"
   6) "Product 3"
   # ... jusqu'à 10 champs

# Continuer avec le curseur
127.0.0.1:6379> HSCAN products:catalog 17 COUNT 10
# ...

# Scanner avec un pattern
127.0.0.1:6379> HSCAN products:catalog 0 MATCH item* COUNT 100
```

**Quand utiliser HSCAN** :
- ✅ Hash avec > 1000 champs
- ✅ Besoin d'itérer sans bloquer Redis
- ✅ En production avec de gros volumes

**Ne PAS utiliser** :
- ❌ Petits Hashes (< 100 champs) → HGETALL est plus simple

---

## 🆚 Hash vs String JSON : Quand utiliser quoi ?

### Utilisez un Hash si :

```bash
# ✅ Accès fréquent à des champs individuels
HGET user:123 email
HSET user:123 city "Lyon"

# ✅ Opérations atomiques sur des compteurs
HINCRBY stats:article:42 views 1

# ✅ Modification partielle fréquente
HSET product:123 stock "14"

# ✅ Beaucoup d'attributs (> 5 champs)
HSET user:123 name "..." email "..." age "..." ... (15 champs)
```

### Utilisez un String JSON si :

```bash
# ✅ Objet complexe avec structures imbriquées
SET order:123 '{"items":[{"id":1,"qty":2},{"id":2,"qty":1}],"total":99.99}'

# ✅ Objet récupéré/modifié en entier (jamais partiellement)
GET cache:api:result
SET cache:api:result '{"status":"ok","data":[...]}'

# ✅ Compatibilité avec d'autres systèmes (export JSON)

# ✅ Peu de champs (2-3 max)
SET session:abc '{"user_id":123,"expires":1234567890}'
```

### Comparaison pratique

**Scénario** : Profil utilisateur avec 10 attributs

```bash
# Hash : Accès granulaire
127.0.0.1:6379> HGET user:123 email  # O(1), très rapide
"alice@example.com"

127.0.0.1:6379> HSET user:123 city "Berlin"  # O(1), atomique
(integer) 0

# String JSON : Doit récupérer et parser TOUT l'objet
127.0.0.1:6379> GET user:123
'{"name":"Alice",...,"city":"Paris"}'  # Récupérer tout
# Dans le code : parser JSON, modifier, sérialiser, réécrire
SET user:123 '{"name":"Alice",...,"city":"Berlin"}'  # Réécrire tout
```

**Résultat** :
- Hash : 2 commandes Redis (1 read, 1 write)
- JSON : 2 commandes Redis + parsing/sérialisation JSON côté client

---

## 🎭 Cas d'usage avancé : Sessions avec TTL

```bash
# Créer une session avec Hash
127.0.0.1:6379> HSET session:abc123 \
  user_id "42" \
  username "alice" \
  role "admin" \
  login_time "1733748000" \
  last_activity "1733748000"
(integer) 5

# Définir un TTL sur toute la session
127.0.0.1:6379> EXPIRE session:abc123 3600
(integer) 1  # Expire dans 1 heure

# À chaque requête : prolonger la session
127.0.0.1:6379> EXPIRE session:abc123 3600
(integer) 1

# Mettre à jour la dernière activité
127.0.0.1:6379> HSET session:abc123 last_activity "1733751600"
(integer) 0

# Récupérer la session
127.0.0.1:6379> HGETALL session:abc123
 1) "user_id"
 2) "42"
 3) "username"
 4) "alice"
 5) "role"
 6) "admin"
 7) "login_time"
 8) "1733748000"
 9) "last_activity"
10) "1733751600"

# Vérifier le TTL restant
127.0.0.1:6379> TTL session:abc123
(integer) 3456  # Secondes restantes

# Logout : supprimer la session
127.0.0.1:6379> DEL session:abc123
(integer) 1
```

---

## 🛠️ HSTRLEN et HRANDFIELD : Commandes utiles

### HSTRLEN : Longueur de la valeur d'un champ (Redis 3.2+)

```bash
127.0.0.1:6379> HSET user:123 bio "Redis enthusiast and developer"
(integer) 1

127.0.0.1:6379> HSTRLEN user:123 bio
(integer) 31  # Nombre de bytes

127.0.0.1:6379> HSTRLEN user:123 nonexistent
(integer) 0
```

### HRANDFIELD : Récupérer un/des champ(s) aléatoire(s) (Redis 6.2+)

```bash
# Créer un Hash avec plusieurs champs
127.0.0.1:6379> HSET colors red "#FF0000" green "#00FF00" blue "#0000FF" yellow "#FFFF00"
(integer) 4

# Récupérer un champ aléatoire (nom seulement)
127.0.0.1:6379> HRANDFIELD colors
"green"

# Récupérer 2 champs aléatoires
127.0.0.1:6379> HRANDFIELD colors 2
1) "blue"
2) "red"

# Récupérer avec les valeurs (WITHVALUES)
127.0.0.1:6379> HRANDFIELD colors 2 WITHVALUES
1) "yellow"
2) "#FFFF00"
3) "green"
4) "#00FF00"

# Négatif : Peut retourner des doublons
127.0.0.1:6379> HRANDFIELD colors -5
1) "red"
2) "red"
3) "blue"
4) "green"
5) "yellow"
```

**Cas d'usage** : Sélection aléatoire (quiz, recommendations, A/B testing).

---

## ⚡ Complexité et performance

| Commande | Complexité | Notes |
|----------|------------|-------|
| `HSET` | O(1) | Par champ |
| `HGET` | O(1) | |
| `HMSET` | O(N) | N = nombre de champs (déprécié, utilisez HSET) |
| `HMGET` | O(N) | N = nombre de champs demandés |
| `HGETALL` | O(N) | N = nombre de champs total, peut être lent |
| `HDEL` | O(N) | N = nombre de champs à supprimer |
| `HEXISTS` | O(1) | |
| `HLEN` | O(1) | |
| `HKEYS/HVALS` | O(N) | N = nombre de champs |
| `HINCRBY` | O(1) | Atomique |
| `HSCAN` | O(1) | Par appel (itération) |

---

## 🚨 Pièges courants à éviter

### 1. HGETALL sur de gros Hashes

```bash
# ❌ DANGEREUX : Hash avec 50 000 champs
HGETALL huge:catalog  # Peut bloquer Redis pendant des millisecondes

# ✅ Utilisez HSCAN pour itérer
HSCAN huge:catalog 0 COUNT 100
```

### 2. Stocker des structures imbriquées dans un Hash

```bash
# ❌ Les valeurs de Hash sont des strings, pas des objets
HSET order:123 items '[{"id":1,"qty":2}]'
# Vous devrez parser côté client, perd l'avantage du Hash

# ✅ Si vous avez des structures imbriquées, utilisez JSON
SET order:123 '{"items":[{"id":1,"qty":2}],"total":99.99}'

# ✅ Ou utilisez RedisJSON (Redis Stack)
JSON.SET order:123 $ '{"items":[{"id":1,"qty":2}],"total":99.99}'
```

### 3. Oublier que les valeurs sont des strings

```bash
127.0.0.1:6379> HSET data count "10"
(integer) 1

127.0.0.1:6379> HGET data count
"10"  # C'est une string "10", pas un nombre 10

# Dans votre code Python :
# count = redis.hget("data", "count")  # Type: bytes
# count = int(count)  # Conversion nécessaire
```

### 4. HINCRBY sur un champ non-numérique

```bash
127.0.0.1:6379> HSET user:123 name "Alice"
(integer) 1

127.0.0.1:6379> HINCRBY user:123 name 1
(error) ERR hash value is not an integer

# Assurez-vous que le champ contient un nombre
127.0.0.1:6379> HSET user:123 age "30"
(integer) 1

127.0.0.1:6379> HINCRBY user:123 age 1
(integer) 31  # OK
```

### 5. Utiliser des Hashes pour des clés-valeurs simples

```bash
# ❌ Overhead inutile pour une seule valeur
HSET cache:key value "data"
# Puis HGET cache:key value

# ✅ Utilisez un String simple
SET cache:key "data"
GET cache:key
```

**Règle** : Utilisez un Hash si vous avez **au moins 3-5 champs**. Pour moins, les Strings suffisent.

---

## 📋 Checklist : Quand utiliser un Hash

### ✅ Utilisez un Hash pour :
- Représenter un **objet** avec plusieurs attributs (utilisateur, produit, etc.)
- Accès **granulaire** fréquent à des champs spécifiques
- **Compteurs multiples** dans un même objet (stats, métriques)
- **Économie de mémoire** vs plusieurs clés String
- **Configuration** avec plusieurs paramètres

### ❌ N'utilisez PAS un Hash pour :
- Structures **imbriquées** complexes → String JSON ou RedisJSON
- Accès **toujours en bloc** (tout ou rien) → String JSON
- Valeurs **uniques** (1-2 champs max) → String
- Besoin de **recherche** ou **index** → RediSearch
- Données **relationnelles** → Base de données SQL

---

## 📊 Comparaison : Hash vs autres structures

| Besoin | Structure recommandée |
|--------|----------------------|
| Objet avec 10 attributs | **Hash** |
| Objet avec structures imbriquées | String JSON ou RedisJSON |
| Compteur unique | String avec INCR |
| Compteurs multiples dans un objet | **Hash** avec HINCRBY |
| Liste ordonnée | List |
| Collection unique | Set |
| Classement par score | Sorted Set |
| Cache simple (clé → valeur) | String |

---

## 🎓 Points clés à retenir

1. ✅ **Hash = dictionnaire** de champs → valeurs (comme un objet)
2. ✅ **Économie de mémoire** vs Strings séparées (1 clé au lieu de N)
3. ✅ **Accès O(1)** à chaque champ individuellement
4. ✅ **HINCRBY** pour des compteurs atomiques dans un objet
5. ✅ **Ziplist** automatique pour les petits Hashes (< 512 champs)
6. ✅ **HSCAN** pour itérer sur de gros Hashes sans bloquer
7. ⚠️ **HGETALL est O(N)** : attention aux gros Hashes
8. ⚠️ **Valeurs = strings** : conversion nécessaire côté client
9. 🎯 Parfait pour : profils utilisateurs, produits, configurations

---

## 🚀 Prochaine étape

Maintenant que vous maîtrisez les Hashes pour représenter des objets, découvrons les **Sets** pour gérer des collections uniques et des opérations ensemblistes !

➡️ **Section suivante** : [2.5 Sets : Unicité et opérations ensemblistes](./05-sets-unicite-operations.md)

---

**Durée estimée** : 1h30
**Niveau** : Débutant à Intermédiaire
**Prérequis** : Sections 2.1, 2.2 et 2.3 complétées

⏭️ [Sets : Unicité et opérations ensemblistes](/02-structures-donnees-natives/05-sets-unicite-operations.md)
