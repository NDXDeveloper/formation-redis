🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.2 Strings : Caching, Compteurs et opérations atomiques

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Pourquoi les Strings sont la structure la plus polyvalente de Redis
- ✅ Comment implémenter un cache efficace
- ✅ Les opérations atomiques sur les compteurs (INCR/DECR)
- ✅ Les opérations bitwise pour optimiser la mémoire
- ✅ Les cas d'usage avancés (rate limiting, sessions, flags)

---

## 📘 Les Strings : Plus que du texte

### Qu'est-ce qu'un String dans Redis ?

Contrairement à ce que le nom suggère, un **String** dans Redis n'est **pas limité au texte**. C'est une **séquence de bytes binaire** qui peut contenir :

- ✅ Du texte (UTF-8, ASCII, etc.)
- ✅ Des nombres (entiers, flottants)
- ✅ Du JSON sérialisé
- ✅ Des données binaires (images, protobuf, etc.)
- ✅ N'importe quoi jusqu'à **512 MB** !

```bash
# Tout ceci est valide
127.0.0.1:6379> SET text "Hello World"
OK

127.0.0.1:6379> SET number "42"
OK

127.0.0.1:6379> SET json '{"name":"Alice","age":30}'
OK

127.0.0.1:6379> SET binary "\x00\x01\x02\xFF"
OK
```

### Pourquoi les Strings sont partout

Les Strings représentent environ **80-90% des clés** dans une application Redis typique car :
- 🚀 **O(1)** pour SET et GET (ultra-rapide)
- 💾 Stockage simple et efficace
- 🔧 Opérations atomiques natives (INCR, APPEND, etc.)
- 🎯 Parfait pour le caching

---

## 🔧 Commandes fondamentales

### SET : Écrire une valeur

```bash
# Syntaxe basique
127.0.0.1:6379> SET mykey "Hello"
OK

# SET avec expiration (EX = secondes, PX = millisecondes)
127.0.0.1:6379> SET session:abc123 "user_data" EX 3600
OK  # Expire dans 1 heure

127.0.0.1:6379> SET token:xyz "jwt_token" PX 30000
OK  # Expire dans 30 secondes

# SET avec condition NX (Not eXists - seulement si la clé n'existe pas)
127.0.0.1:6379> SET lock:resource "locked" NX EX 10
OK  # Crée le lock

127.0.0.1:6379> SET lock:resource "locked" NX EX 10
(nil)  # Échec car le lock existe déjà

# SET avec condition XX (eXists - seulement si la clé existe)
127.0.0.1:6379> SET config:theme "dark"
OK

127.0.0.1:6379> SET config:theme "light" XX
OK  # Succès car la clé existe

127.0.0.1:6379> SET config:new "value" XX
(nil)  # Échec car la clé n'existe pas

# SET avec GET (Redis 6.2+) - Retourne l'ancienne valeur
127.0.0.1:6379> SET counter "10"
OK

127.0.0.1:6379> SET counter "20" GET
"10"  # Retourne l'ancienne valeur

# SET avec EXAT (expire à un timestamp Unix précis)
127.0.0.1:6379> SET reminder "Meeting" EXAT 1735567200
OK  # Expire le 30 décembre 2024 à 10:00 UTC

# SET avec KEEPTTL (Redis 6.0+) - Garde le TTL existant
127.0.0.1:6379> SET mykey "value" EX 100
OK

127.0.0.1:6379> TTL mykey
(integer) 98

127.0.0.1:6379> SET mykey "new_value" KEEPTTL
OK

127.0.0.1:6379> TTL mykey
(integer) 95  # Le TTL est préservé
```

### GET : Lire une valeur

```bash
# Récupérer une valeur
127.0.0.1:6379> GET mykey
"Hello"

# Si la clé n'existe pas
127.0.0.1:6379> GET nonexistent
(nil)

# Si la clé n'est pas un String
127.0.0.1:6379> LPUSH mylist "item"
(integer) 1

127.0.0.1:6379> GET mylist
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

### MSET et MGET : Opérations multiples

```bash
# Définir plusieurs clés en une seule commande (atomique)
127.0.0.1:6379> MSET key1 "value1" key2 "value2" key3 "value3"
OK

# Récupérer plusieurs clés
127.0.0.1:6379> MGET key1 key2 key3 nonexistent
1) "value1"
2) "value2"
3) "value3"
4) (nil)

# MSETNX : SET multiple uniquement si TOUTES les clés n'existent pas
127.0.0.1:6379> MSETNX newkey1 "val1" newkey2 "val2"
(integer) 1  # Succès

127.0.0.1:6379> MSETNX newkey1 "val1" newkey3 "val3"
(integer) 0  # Échec car newkey1 existe, newkey3 n'est PAS créé
```

**Avantage de MSET/MGET** : Réduction du nombre d'aller-retours réseau (Round Trip Time - RTT).

```bash
# ❌ Inefficace : 3 RTT
GET key1
GET key2
GET key3

# ✅ Efficace : 1 RTT
MGET key1 key2 key3
```

### GETSET : Lire et écrire atomiquement (déprécié)

```bash
# Ancienne syntaxe (Redis < 6.2)
127.0.0.1:6379> GETSET mykey "new_value"
"old_value"

# ✅ Nouvelle syntaxe recommandée (Redis 6.2+)
127.0.0.1:6379> SET mykey "new_value" GET
"old_value"
```

### GETDEL et GETEX : Récupérer avec effet de bord

```bash
# GETDEL : Récupérer puis supprimer (Redis 6.2+)
127.0.0.1:6379> SET temp "data"
OK

127.0.0.1:6379> GETDEL temp
"data"

127.0.0.1:6379> GET temp
(nil)  # La clé a été supprimée

# GETEX : Récupérer et modifier le TTL (Redis 6.2+)
127.0.0.1:6379> SET mykey "value"
OK

127.0.0.1:6379> GETEX mykey EX 60
"value"  # Récupère la valeur ET définit un TTL de 60 secondes

127.0.0.1:6379> TTL mykey
(integer) 58
```

---

## 💾 Cas d'usage #1 : Caching simple

### Cache de résultats de requêtes

```bash
# Scénario : Cache d'une requête de profil utilisateur

# 1. Vérifier si le résultat est en cache
127.0.0.1:6379> GET cache:user:profile:123
(nil)  # Cache miss

# 2. Après avoir récupéré depuis la DB, mettre en cache pour 5 minutes
127.0.0.1:6379> SET cache:user:profile:123 '{"id":123,"name":"Alice","email":"alice@example.com"}' EX 300
OK

# 3. Requêtes suivantes = cache hit
127.0.0.1:6379> GET cache:user:profile:123
"{\"id\":123,\"name\":\"Alice\",\"email\":\"alice@example.com\"}"

# 4. Invalider le cache lors d'une modification
127.0.0.1:6379> DEL cache:user:profile:123
(integer) 1
```

**Code application (pseudo-code)** :
```python
def get_user_profile(user_id):
    cache_key = f"cache:user:profile:{user_id}"

    # Essayer le cache
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss : requête DB
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)

    # Mettre en cache pour 5 minutes
    redis.set(cache_key, json.dumps(user), ex=300)

    return user
```

### Cache de réponses API

```bash
# Cache d'une API météo externe
127.0.0.1:6379> SET cache:api:weather:paris '{"temp":15,"condition":"cloudy"}' EX 1800
OK  # Cache pendant 30 minutes

# Cache d'un calcul coûteux
127.0.0.1:6379> SET cache:compute:fibonacci:100 "354224848179261915075" EX 86400
OK  # Cache pendant 24 heures
```

### Stratégie : Cache-Aside (Lazy Loading)

C'est le pattern le plus courant :

```bash
# Pseudo-code du pattern
fonction get_data(key):
    # 1. Chercher dans le cache
    valeur = redis.GET("cache:" + key)

    si valeur existe:
        retourner valeur  # Cache hit

    # 2. Cache miss : aller chercher la source
    valeur = source_de_donnees.fetch(key)

    # 3. Mettre en cache pour la prochaine fois
    redis.SET("cache:" + key, valeur, EX=ttl)

    retourner valeur
```

---

## 🔢 Cas d'usage #2 : Compteurs atomiques

### INCR et DECR : Incrémentation atomique

Les commandes INCR/DECR sont **atomiques**, ce qui signifie qu'elles sont **thread-safe** sans besoin de locks !

```bash
# Incrémenter une valeur (créée à 0 si inexistante)
127.0.0.1:6379> INCR views:article:123
(integer) 1

127.0.0.1:6379> INCR views:article:123
(integer) 2

127.0.0.1:6379> INCR views:article:123
(integer) 3

# Décrémenter
127.0.0.1:6379> DECR stock:product:456
(integer) 99

# Incrémenter par une valeur spécifique
127.0.0.1:6379> INCRBY likes:post:789 5
(integer) 5

127.0.0.1:6379> INCRBY likes:post:789 10
(integer) 15

# Décrémenter par une valeur spécifique
127.0.0.1:6379> DECRBY stock:product:456 3
(integer) 96

# Incrémenter un nombre flottant
127.0.0.1:6379> SET balance:user:123 "100.50"
OK

127.0.0.1:6379> INCRBYFLOAT balance:user:123 25.75
"126.25"

127.0.0.1:6379> INCRBYFLOAT balance:user:123 -10.5
"115.75"  # Décrémentation avec un nombre négatif
```

**Pourquoi c'est important** :

```bash
# ❌ MAUVAIS : Race condition possible
GET counter
# Dans le code : value = parse(result) + 1
SET counter value

# Entre GET et SET, un autre client peut modifier la valeur !
# Résultat : compteur incorrect

# ✅ BON : Opération atomique
INCR counter
# Redis garantit qu'aucune autre opération ne peut s'intercaler
```

### Exemple : Compteur de vues d'article

```bash
# Chaque fois qu'un utilisateur visite l'article
127.0.0.1:6379> INCR stats:article:42:views
(integer) 1

127.0.0.1:6379> INCR stats:article:42:views
(integer) 2

# Récupérer le nombre total de vues
127.0.0.1:6379> GET stats:article:42:views
"2"

# Réinitialiser le compteur
127.0.0.1:6379> SET stats:article:42:views 0
OK
```

### Exemple : Compteur de likes avec contrainte

```bash
# Un utilisateur like un post
127.0.0.1:6379> INCR post:123:likes
(integer) 1

# Vérifier avant de décrémenter (unlike)
127.0.0.1:6379> GET post:123:likes
"1"

# Dans votre code :
# if (likes > 0) { redis.DECR(...) }

127.0.0.1:6379> DECR post:123:likes
(integer) 0

# Redis n'empêche pas les valeurs négatives !
127.0.0.1:6379> DECR post:123:likes
(integer) -1  # ⚠️ Possible si pas de validation

# Solution : Utiliser Lua script pour validation atomique
```

### Exemple : Inventaire de produit

```bash
# Stock initial
127.0.0.1:6379> SET inventory:product:456 "100"
OK

# Vente de 3 unités
127.0.0.1:6379> DECRBY inventory:product:456 3
(integer) 97

# Réapprovisionnement de 50 unités
127.0.0.1:6379> INCRBY inventory:product:456 50
(integer) 147

# Vérification du stock avant vente (dans votre code)
stock = redis.GET("inventory:product:456")
if stock >= quantity_requested:
    redis.DECRBY("inventory:product:456", quantity_requested)
```

---

## 🎯 Cas d'usage #3 : Rate Limiting simple

### Rate Limiting avec INCR et EXPIRE

```bash
# Limiter à 5 requêtes par minute par utilisateur

# Requête 1
127.0.0.1:6379> SET rate:user:123:2024-12-09-14:30 "0" EX 60 NX
OK

127.0.0.1:6379> INCR rate:user:123:2024-12-09-14:30
(integer) 1

# Requête 2
127.0.0.1:6379> INCR rate:user:123:2024-12-09-14:30
(integer) 2

# Requête 3-5
127.0.0.1:6379> INCR rate:user:123:2024-12-09-14:30
(integer) 3

# ... jusqu'à 5

# Requête 6 (refusée dans votre code)
127.0.0.1:6379> GET rate:user:123:2024-12-09-14:30
"5"
# Dans le code : if count >= 5 : return "429 Too Many Requests"

# Après 60 secondes, la clé expire automatiquement
127.0.0.1:6379> GET rate:user:123:2024-12-09-14:30
(nil)  # Clé expirée, limite réinitialisée
```

**Code application** :
```python
def check_rate_limit(user_id, limit=5, window=60):
    now = datetime.now()
    key = f"rate:user:{user_id}:{now.strftime('%Y-%m-%d-%H:%M')}"

    # Essayer de créer la clé avec TTL
    redis.set(key, 0, ex=window, nx=True)

    # Incrémenter
    count = redis.incr(key)

    if count > limit:
        return False, f"Rate limit exceeded. Try again in {redis.ttl(key)} seconds"

    return True, count
```

---

## 🧮 Opérations sur les sous-chaînes

### GETRANGE : Récupérer une partie

```bash
# Créer une chaîne
127.0.0.1:6379> SET message "Hello World"
OK

# Récupérer du caractère 0 à 4 (inclus)
127.0.0.1:6379> GETRANGE message 0 4
"Hello"

# Récupérer du caractère 6 à la fin
127.0.0.1:6379> GETRANGE message 6 -1
"World"

# Indices négatifs : -1 = dernier caractère
127.0.0.1:6379> GETRANGE message -5 -1
"World"
```

### SETRANGE : Modifier une partie

```bash
# Créer une chaîne
127.0.0.1:6379> SET message "Hello World"
OK

# Remplacer à partir de l'index 6
127.0.0.1:6379> SETRANGE message 6 "Redis"
(integer) 11  # Longueur totale de la chaîne

127.0.0.1:6379> GET message
"Hello Redis"

# Si SETRANGE dépasse la longueur, Redis remplit avec des \x00
127.0.0.1:6379> SET short "Hi"
OK

127.0.0.1:6379> SETRANGE short 5 "!"
(integer) 6

127.0.0.1:6379> GET short
"Hi\x00\x00\x00!"  # \x00 = null bytes
```

### APPEND : Ajouter à la fin

```bash
# Créer une chaîne
127.0.0.1:6379> SET log "Event: "
OK

# Ajouter du texte
127.0.0.1:6379> APPEND log "User logged in"
(integer) 21  # Longueur totale

127.0.0.1:6379> GET log
"Event: User logged in"

# Si la clé n'existe pas, APPEND crée la clé
127.0.0.1:6379> APPEND newkey "Hello"
(integer) 5

127.0.0.1:6379> APPEND newkey " World"
(integer) 11

127.0.0.1:6379> GET newkey
"Hello World"
```

**Cas d'usage** : Logs ou buffers incrémentaux.

### STRLEN : Longueur de la chaîne

```bash
127.0.0.1:6379> SET mykey "Hello"
OK

127.0.0.1:6379> STRLEN mykey
(integer) 5

127.0.0.1:6379> STRLEN nonexistent
(integer) 0
```

---

## 🔲 Opérations bitwise

Les Strings peuvent être manipulées **au niveau du bit**, ce qui permet des optimisations mémoire incroyables !

### SETBIT et GETBIT : Manipulation bit par bit

```bash
# Définir le bit à l'index 7 à 1
127.0.0.1:6379> SETBIT mykey 7 1
(integer) 0  # Ancienne valeur du bit

# Récupérer la valeur du bit
127.0.0.1:6379> GETBIT mykey 7
(integer) 1

127.0.0.1:6379> GETBIT mykey 100
(integer) 0  # Les bits non définis sont 0

# Visualiser la valeur
127.0.0.1:6379> GET mykey
"\x01"  # En hexadécimal
```

### BITCOUNT : Compter les bits à 1

```bash
# Créer un bitmap
127.0.0.1:6379> SETBIT users:online 100 1
(integer) 0
127.0.0.1:6379> SETBIT users:online 250 1
(integer) 0
127.0.0.1:6379> SETBIT users:online 500 1
(integer) 0

# Compter combien de bits sont à 1
127.0.0.1:6379> BITCOUNT users:online
(integer) 3  # 3 utilisateurs en ligne

# BITCOUNT sur une plage
127.0.0.1:6379> BITCOUNT users:online 0 100
(integer) 1  # 1 bit dans les 100 premiers bytes
```

### BITOP : Opérations entre bitmaps

```bash
# Créer deux bitmaps
127.0.0.1:6379> SETBIT key1 1 1
127.0.0.1:6379> SETBIT key1 3 1
127.0.0.1:6379> SETBIT key2 1 1
127.0.0.1:6379> SETBIT key2 5 1

# AND : Intersection
127.0.0.1:6379> BITOP AND result key1 key2
(integer) 1

127.0.0.1:6379> GETBIT result 1
(integer) 1  # Présent dans les deux

127.0.0.1:6379> GETBIT result 3
(integer) 0  # Présent seulement dans key1

# OR : Union
127.0.0.1:6379> BITOP OR result key1 key2
(integer) 1

127.0.0.1:6379> BITCOUNT result
(integer) 3  # Bits 1, 3, 5

# XOR : Différence symétrique
127.0.0.1:6379> BITOP XOR result key1 key2
(integer) 1

127.0.0.1:6379> BITCOUNT result
(integer) 2  # Bits 3 et 5 (présents dans un seul)

# NOT : Inversion
127.0.0.1:6379> BITOP NOT result key1
(integer) 1
```

### Cas d'usage : Tracking de présence

```bash
# Suivre les utilisateurs actifs par jour
# Chaque utilisateur a un ID unique (0-999999)

# Utilisateur 123 actif le 9 décembre
127.0.0.1:6379> SETBIT active:2024-12-09 123 1
(integer) 0

# Utilisateur 456 actif le 9 décembre
127.0.0.1:6379> SETBIT active:2024-12-09 456 1
(integer) 0

# Combien d'utilisateurs actifs aujourd'hui ?
127.0.0.1:6379> BITCOUNT active:2024-12-09
(integer) 2

# L'utilisateur 123 était-il actif ?
127.0.0.1:6379> GETBIT active:2024-12-09 123
(integer) 1

# Utilisateurs actifs à la fois le 9 et le 10 décembre (intersection)
127.0.0.1:6379> BITOP AND active:both-days active:2024-12-09 active:2024-12-10
(integer) 125000

127.0.0.1:6379> BITCOUNT active:both-days
(integer) 1  # 1 utilisateur actif les deux jours
```

**Avantage mémoire** :
- 1 million d'utilisateurs = 1 million de bits = **125 KB** seulement !
- Comparez avec un Set : ~90 bytes × 1M = **90 MB**
- **Économie : 720× moins de mémoire**

---

## 📊 Cas d'usage avancés

### 1. Session Store

```bash
# Stocker une session avec expiration automatique
127.0.0.1:6379> SET session:abc123 '{"user_id":42,"role":"admin","cart":[1,2,3]}' EX 3600
OK  # Expire dans 1 heure

# Prolonger la session à chaque requête
127.0.0.1:6379> EXPIRE session:abc123 3600
(integer) 1

# Ou utiliser GETEX pour récupérer et prolonger
127.0.0.1:6379> GETEX session:abc123 EX 3600
"{\"user_id\":42,\"role\":\"admin\",\"cart\":[1,2,3]}"

# Détruire la session (logout)
127.0.0.1:6379> DEL session:abc123
(integer) 1
```

### 2. Distributed Lock simple

```bash
# Acquérir un lock (NX = seulement si n'existe pas)
127.0.0.1:6379> SET lock:resource:payment:123 "worker-1" NX EX 10
OK  # Lock acquis pour 10 secondes

# Autre worker essaie d'acquérir le même lock
127.0.0.1:6379> SET lock:resource:payment:123 "worker-2" NX EX 10
(nil)  # Échec, ressource déjà lockée

# Libérer le lock (attention : vérifier que c'est bien votre lock !)
127.0.0.1:6379> GET lock:resource:payment:123
"worker-1"

# Si c'est bien notre lock, le supprimer
127.0.0.1:6379> DEL lock:resource:payment:123
(integer) 1
```

⚠️ **Note** : Ce pattern simple n'est pas totalement sûr. Pour un locking robuste, utilisez **Redlock** (Module 6.5).

### 3. Feature Flags

```bash
# Activer/désactiver des fonctionnalités
127.0.0.1:6379> SET feature:new-ui "enabled"
OK

127.0.0.1:6379> SET feature:beta-search "disabled"
OK

# Vérifier dans l'application
127.0.0.1:6379> GET feature:new-ui
"enabled"

# Flags avec pourcentage de rollout
127.0.0.1:6379> SET feature:new-checkout:rollout "25"
OK  # 25% des utilisateurs
```

### 4. Configuration distribuée

```bash
# Stocker des configs partagées entre services
127.0.0.1:6379> MSET \
  config:api:max-requests "1000" \
  config:api:timeout "30" \
  config:db:pool-size "10"
OK

# Récupérer toutes les configs d'un coup
127.0.0.1:6379> MGET config:api:max-requests config:api:timeout config:db:pool-size
1) "1000"
2) "30"
3) "10"
```

### 5. Monitoring de santé (Health Check)

```bash
# Service A met à jour son heartbeat toutes les 10 secondes
127.0.0.1:6379> SET health:service-a "OK" EX 30
OK

# Monitoring vérifie la présence
127.0.0.1:6379> GET health:service-a
"OK"  # Service vivant

# Si le service plante, la clé expire après 30 secondes
127.0.0.1:6379> GET health:service-a
(nil)  # Service considéré mort
```

---

## ⚡ Optimisations et bonnes pratiques

### 1. Utilisez MGET/MSET pour les opérations en batch

```bash
# ❌ Inefficace : 3 requêtes réseau
GET user:123:name
GET user:123:email
GET user:123:age

# ✅ Efficace : 1 requête réseau
MGET user:123:name user:123:email user:123:age
```

### 2. Sérialisez intelligemment

```bash
# ❌ JSON pour des données simples
SET user:123:score '{"score":1500}'  # Overhead inutile

# ✅ Valeur directe
SET user:123:score "1500"

# ✅ JSON pour des objets complexes
SET user:123:profile '{"name":"Alice","email":"alice@ex.com","prefs":{"theme":"dark"}}'
```

### 3. Utilisez des TTLs appropriés

```bash
# Cache de données qui changent rarement : TTL long
SET cache:static:homepage "..." EX 86400  # 24 heures

# Cache de données volatiles : TTL court
SET cache:live:stock-price "..." EX 60  # 1 minute

# Sessions utilisateur
SET session:abc "..." EX 3600  # 1 heure
```

### 4. Préfixez par type de donnée (optionnel mais utile)

```bash
# Facilite le debugging et le monitoring
SET cache:user:123 "..."
SET session:abc123 "..."
SET config:api:timeout "..."
SET metric:requests:count "..."
```

### 5. Attention aux valeurs très grandes

```bash
# ❌ Évitez de stocker des images/fichiers énormes
SET image:logo "... 10 MB de données binaires ..."

# ✅ Stockez une référence (URL, path, S3 key)
SET image:logo:url "https://cdn.example.com/logo.png"
```

---

## 🚨 Pièges courants

### 1. Race conditions sans atomicité

```bash
# ❌ Non-atomique : Race condition
GET counter
# (supposons qu'on récupère "10")
# Dans le code : new_value = 10 + 1
SET counter 11
# Si 2 clients font ça en même temps, on perd un incrément !

# ✅ Atomique
INCR counter
```

### 2. Oublier les TTLs sur les caches

```bash
# ❌ Cache qui ne se vide jamais
SET cache:user:123 "..."
# Redis se remplit jusqu'à OOM !

# ✅ Cache avec expiration
SET cache:user:123 "..." EX 300
```

### 3. Utiliser GET pour vérifier l'existence

```bash
# ❌ Inefficace pour juste vérifier
GET mykey
# (ignore la valeur, regarde juste si nil)

# ✅ Utilisez EXISTS
EXISTS mykey
(integer) 1
```

### 4. INCR sur des non-nombres

```bash
127.0.0.1:6379> SET mykey "hello"
OK

127.0.0.1:6379> INCR mykey
(error) ERR value is not an integer or out of range

# Toujours initialiser correctement
127.0.0.1:6379> SET counter "0"
OK
```

### 5. Confondre bytes et caractères

```bash
127.0.0.1:6379> SET emoji "👋"
OK

127.0.0.1:6379> STRLEN emoji
(integer) 4  # Pas 1 ! L'emoji fait 4 bytes en UTF-8

127.0.0.1:6379> GETRANGE emoji 0 1
"\xf0\x9f"  # Partie d'un caractère, illisible
```

---

## 📊 Récapitulatif des commandes

| Commande | Usage | Complexité | Notes |
|----------|-------|------------|-------|
| `SET` | Écrire une valeur | O(1) | Options : EX, PX, NX, XX, KEEPTTL |
| `GET` | Lire une valeur | O(1) | Retourne nil si inexistant |
| `MSET` | Écrire plusieurs valeurs | O(N) | Atomique |
| `MGET` | Lire plusieurs valeurs | O(N) | Réduit les RTT |
| `INCR/DECR` | Incrémenter/Décrémenter | O(1) | Atomique, thread-safe |
| `INCRBY/DECRBY` | Incr/Decr par valeur | O(1) | Atomique |
| `INCRBYFLOAT` | Incr avec décimales | O(1) | Atomique |
| `APPEND` | Ajouter à la fin | O(1) | Amortized |
| `GETRANGE` | Sous-chaîne | O(N) | N = longueur extraite |
| `SETRANGE` | Modifier sous-chaîne | O(1) | Si pas de réallocation |
| `STRLEN` | Longueur | O(1) | En bytes, pas caractères |
| `SETBIT/GETBIT` | Manipulation de bits | O(1) | |
| `BITCOUNT` | Compter bits à 1 | O(N) | N = taille de la chaîne |
| `BITOP` | Opérations entre bitmaps | O(N) | AND, OR, XOR, NOT |
| `GETDEL` | Récupérer et supprimer | O(1) | Redis 6.2+ |
| `GETEX` | Récupérer et modifier TTL | O(1) | Redis 6.2+ |

---

## 🎓 Points clés à retenir

1. ✅ **Les Strings ne sont pas que du texte** : données binaires, JSON, nombres, etc.
2. ✅ **INCR/DECR sont atomiques** : parfaits pour les compteurs sans race conditions
3. ✅ **Utilisez MGET/MSET** pour réduire les aller-retours réseau
4. ✅ **Toujours définir un TTL** pour les caches et sessions
5. ✅ **Les bitmaps** permettent des économies mémoire massives pour les flags
6. ✅ **SET avec NX** = lock simple et rapide
7. ⚠️ **Les Strings peuvent aller jusqu'à 512 MB** mais restez raisonnables

---

## 🚀 Prochaine étape

Maintenant que vous maîtrisez les Strings, passons aux **Lists** pour créer des files d'attente et des timelines !

➡️ **Section suivante** : [2.3 Lists : Files d'attente simples](./03-lists-files-attente.md)

---

**Durée estimée** : 1h30
**Niveau** : Débutant à Intermédiaire
**Prérequis** : Section 2.1 complétée

⏭️ [Lists : Files d'attente simples (LPUSH/RPOP)](/02-structures-donnees-natives/03-lists-files-attente.md)
