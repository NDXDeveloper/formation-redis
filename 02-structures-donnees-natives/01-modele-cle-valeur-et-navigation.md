🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1 Le modèle clé-valeur et navigation (CLI & GUI)

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Le fonctionnement du modèle clé-valeur de Redis
- ✅ Comment nommer vos clés efficacement
- ✅ Les commandes essentielles de navigation
- ✅ Comment utiliser redis-cli et Redis Insight
- ✅ Les bonnes pratiques de structuration des clés

---

## 📘 Le modèle clé-valeur : Les fondamentaux

### Qu'est-ce qu'une clé dans Redis ?

Dans Redis, **tout repose sur des clés**. Une clé est simplement une **chaîne de caractères** qui identifie de manière unique une valeur stockée en mémoire.

```bash
# Anatomie d'une paire clé-valeur
127.0.0.1:6379> SET mykey "Hello Redis"
OK
# ↑ clé    ↑ valeur
```

**Caractéristiques importantes** :
- ✅ Les clés sont **sensibles à la casse** : `User` ≠ `user`
- ✅ Taille maximum : **512 MB** (mais utilisez des clés courtes !)
- ✅ Peut contenir n'importe quel caractère, y compris espaces et symboles
- ✅ Redis stocke environ **90 bytes d'overhead** par clé

### Les clés sont "plates" (flat namespace)

Contrairement aux bases de données SQL, Redis n'a **pas de tables, pas de schémas, pas de hiérarchies natives**. Tout est dans un seul espace de noms global.

```bash
# ❌ Redis n'a PAS de structure comme ça :
Database
  └── Table: users
      └── Row: id=123

# ✅ Redis a plutôt ça :
clé: "user:123:name" → valeur: "Alice"
clé: "user:123:email" → valeur: "alice@example.com"
clé: "user:456:name" → valeur: "Bob"
```

---

## 🏗️ Conventions de nommage : La clé d'un projet propre

### La convention `:` (deux-points)

La communauté Redis utilise massivement le **deux-points** comme séparateur hiérarchique :

```bash
# Pattern recommandé : object:type:id:field
user:profile:123:name
user:profile:123:email
user:settings:123:theme

product:inventory:456:stock
product:inventory:456:price

cache:api:weather:paris
cache:api:weather:london
```

**Pourquoi c'est important ?**
- 🔍 Facilite les recherches avec pattern matching
- 📊 Améliore la lisibilité dans Redis Insight
- 🛠️ Permet des opérations groupées avec SCAN

### Exemples de conventions par cas d'usage

#### 1. Application e-commerce
```bash
# Utilisateurs
user:123:profile              # Hash : données du profil
user:123:cart                 # Hash : panier en cours
user:123:orders               # List : historique des commandes
user:123:wishlist             # Set : articles favoris

# Produits
product:456:details           # Hash : infos produit
product:456:inventory         # String : stock disponible
product:456:reviews           # Sorted Set : avis clients
product:category:electronics  # Set : IDs de produits dans la catégorie
```

#### 2. Application de chat
```bash
# Messages
chat:room:general:messages    # List : messages du salon
chat:room:general:members     # Set : utilisateurs présents
chat:user:123:unread          # String : compteur de non-lus
chat:dm:123:456               # List : conversation privée user 123 ↔ 456
```

#### 3. Système de cache
```bash
# Cache avec namespace
cache:db:query:users:list     # String : résultat de la requête
cache:api:github:user:octocat # String : réponse API
cache:computed:stats:daily    # Hash : statistiques calculées

# Session utilisateur
session:abc123                # Hash : données de session
```

### Bonnes pratiques de nommage

✅ **À FAIRE** :
```bash
# Utilisez des noms descriptifs
user:profile:123:email

# Incluez le type d'objet
post:article:456:title

# Séparez avec des deux-points
analytics:visits:2024-12-09

# Utilisez le même ordre partout
resource:action:id
```

❌ **À ÉVITER** :
```bash
# Trop court, pas de contexte
u123e

# Mélange de séparateurs
user_profile-123.email

# Trop long (> 100 caractères)
application:production:region:eu-west:service:api:endpoint:users:cache:query:list:active:sorted

# Pas de structure
JohnDoeEmailAddress
```

---

## 🔧 Navigation avec redis-cli

### Démarrage et connexion

```bash
# Lancer redis-cli (connexion locale par défaut)
$ redis-cli
127.0.0.1:6379>

# Connexion à un serveur distant
$ redis-cli -h redis.example.com -p 6379 -a "password"

# Connexion avec authentification (Redis 6+)
$ redis-cli -h localhost -p 6379 --user admin --pass securepassword

# Sélectionner une base de données (0-15 par défaut)
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]>  # Notez le [1]
```

### Commandes de navigation essentielles

#### 1. SET et GET : Les commandes de base

```bash
# Créer une clé simple
127.0.0.1:6379> SET greeting "Hello World"
OK

# Récupérer la valeur
127.0.0.1:6379> GET greeting
"Hello World"

# SET avec expiration (EX = secondes, PX = millisecondes)
127.0.0.1:6379> SET temp:token "abc123" EX 300
OK  # Expire dans 5 minutes

# SET uniquement si la clé n'existe pas (NX = Not eXists)
127.0.0.1:6379> SET config:theme "dark" NX
OK
127.0.0.1:6379> SET config:theme "light" NX
(nil)  # Échec car la clé existe déjà

# SET uniquement si la clé existe (XX = eXists)
127.0.0.1:6379> SET config:theme "light" XX
OK  # Succès car la clé existe
```

#### 2. EXISTS : Vérifier l'existence d'une clé

```bash
# Vérifier une seule clé
127.0.0.1:6379> EXISTS greeting
(integer) 1  # 1 = existe, 0 = n'existe pas

# Vérifier plusieurs clés
127.0.0.1:6379> EXISTS greeting config:theme nonexistent
(integer) 2  # 2 clés existent sur les 3
```

#### 3. DEL : Supprimer des clés

```bash
# Supprimer une clé
127.0.0.1:6379> DEL greeting
(integer) 1  # Nombre de clés supprimées

# Supprimer plusieurs clés
127.0.0.1:6379> DEL key1 key2 key3
(integer) 3

# DEL est bloquant ! Utilisez UNLINK pour les grosses clés
127.0.0.1:6379> UNLINK huge:list
(integer) 1  # Suppression asynchrone (non-bloquante)
```

#### 4. TYPE : Identifier le type d'une clé

```bash
127.0.0.1:6379> SET mystring "value"
OK
127.0.0.1:6379> LPUSH mylist "item"
(integer) 1
127.0.0.1:6379> SADD myset "member"
(integer) 1

# Vérifier les types
127.0.0.1:6379> TYPE mystring
string
127.0.0.1:6379> TYPE mylist
list
127.0.0.1:6379> TYPE myset
set
127.0.0.1:6379> TYPE nonexistent
none
```

#### 5. KEYS : Lister les clés (⚠️ DANGER en production !)

```bash
# Lister TOUTES les clés
127.0.0.1:6379> KEYS *
1) "user:123:name"
2) "user:123:email"
3) "user:456:name"

# Rechercher avec un pattern
127.0.0.1:6379> KEYS user:*
1) "user:123:name"
2) "user:123:email"
3) "user:456:name"

# Pattern avec ?
127.0.0.1:6379> KEYS user:???:name
1) "user:123:name"
2) "user:456:name"

# Pattern avec []
127.0.0.1:6379> KEYS user:[12]*
1) "user:123:name"
2) "user:200:name"
```

⚠️ **ATTENTION** : `KEYS` est une commande **O(N)** qui bloque Redis pendant son exécution. **Ne l'utilisez JAMAIS en production !** Utilisez `SCAN` à la place (voir ci-dessous).

#### 6. SCAN : L'alternative safe à KEYS

```bash
# Scanner toutes les clés (par batch)
127.0.0.1:6379> SCAN 0
1) "17"  # Curseur suivant
2) 1) "user:123:name"
   2) "user:123:email"
   3) "product:456:stock"

# Continuer avec le curseur suivant
127.0.0.1:6379> SCAN 17
1) "0"   # 0 = fin du scan
2) 1) "user:456:name"
   2) "cache:api:result"

# SCAN avec pattern matching
127.0.0.1:6379> SCAN 0 MATCH user:* COUNT 100
1) "0"
2) 1) "user:123:name"
   2) "user:123:email"
   3) "user:456:name"

# Explication des paramètres :
# - Curseur initial : 0
# - MATCH : pattern de recherche
# - COUNT : hint (Redis peut retourner plus ou moins)
```

**Pourquoi SCAN est meilleur que KEYS** :
- ✅ Ne bloque pas le serveur (itération incrémentale)
- ✅ Utilisable en production
- ✅ Peut être interrompu et repris
- ⚠️ Peut retourner des duplicatas (mais c'est rare)
- ⚠️ Ne garantit pas un nombre exact de résultats par appel

#### 7. DBSIZE : Compter les clés

```bash
# Nombre total de clés dans la base actuelle
127.0.0.1:6379> DBSIZE
(integer) 42

# Très rapide : O(1) car Redis maintient un compteur
```

#### 8. FLUSHDB et FLUSHALL : Tout supprimer

```bash
# Supprimer toutes les clés de la base actuelle
127.0.0.1:6379> FLUSHDB
OK

# Supprimer toutes les clés de TOUTES les bases (0-15)
127.0.0.1:6379> FLUSHALL
OK

# Version asynchrone (recommandée pour les gros volumes)
127.0.0.1:6379> FLUSHDB ASYNC
OK
```

⚠️ **DANGER** : Ces commandes sont **destructives et irréversibles** !

---

## 🖥️ Redis Insight : L'interface graphique moderne

### Qu'est-ce que Redis Insight ?

**Redis Insight** est l'outil GUI officiel de Redis pour :
- 🔍 Naviguer visuellement dans les données
- 📊 Analyser les performances (Profiler, Slowlog)
- 🛠️ Exécuter des commandes avec auto-complétion
- 📈 Visualiser les statistiques mémoire

### Installation

```bash
# Via Docker (recommandé)
docker run -d --name redisinsight \
  -p 5540:5540 \
  redis/redisinsight:latest

# Accès : http://localhost:5540
```

Ou téléchargez depuis : https://redis.io/insight/

### Fonctionnalités principales

#### 1. Browser : Navigation visuelle

L'onglet **Browser** vous permet de :
- 📂 Voir les clés organisées par namespace (grâce aux `:`)
- 🔍 Filtrer avec des patterns
- ✏️ Éditer les valeurs directement
- 📊 Voir le type de chaque clé

**Exemple d'arborescence** :
```
📁 user:
  📁 123:
    📄 name (string) → "Alice"
    📄 email (string) → "alice@example.com"
    📊 cart (hash) → 3 fields
  📁 456:
    📄 name (string) → "Bob"
```

#### 2. Workbench : Console avancée

Comme redis-cli mais avec :
- ✨ Auto-complétion intelligente
- 📜 Historique des commandes
- 🎨 Coloration syntaxique
- 📋 Résultats formatés (JSON, etc.)

```bash
# Exemple dans Workbench
> HGETALL user:123:profile
1) "name"
2) "Alice"
3) "age"
4) "30"
5) "city"
6) "Paris"

# Redis Insight formate automatiquement en tableau
```

#### 3. Profiler : Monitoring en temps réel

Capture **toutes les commandes** exécutées sur Redis :
- ⏱️ Timestamp de chaque commande
- 🔢 Durée d'exécution
- 📝 Commande complète avec arguments

**Cas d'usage** : Identifier les requêtes lentes ou inefficaces.

---

## 📚 Cas d'usage : Navigation dans un projet réel

Imaginons une application de blog avec des utilisateurs, des articles et des commentaires.

### 1. Créer la structure de données

```bash
# Utilisateur 123
127.0.0.1:6379> HSET user:123:profile name "Alice" email "alice@blog.com" bio "Rédactrice tech"
(integer) 3

# Article 1
127.0.0.1:6379> HSET post:1:meta title "Introduction à Redis" author_id "123" views "0"
(integer) 3
127.0.0.1:6379> SET post:1:content "Redis est une base de données in-memory..."
OK

# Commentaires sur l'article 1
127.0.0.1:6379> LPUSH post:1:comments "Super article !"
(integer) 1
127.0.0.1:6379> LPUSH post:1:comments "Très clair, merci"
(integer) 2

# Tags de l'article
127.0.0.1:6379> SADD post:1:tags "redis" "database" "nosql"
(integer) 3

# Liste des articles de l'utilisateur 123
127.0.0.1:6379> SADD user:123:posts "1" "2" "3"
(integer) 3
```

### 2. Explorer les données

```bash
# Lister tous les posts
127.0.0.1:6379> SCAN 0 MATCH post:*
1) "0"
2) 1) "post:1:meta"
   2) "post:1:content"
   3) "post:1:comments"
   4) "post:1:tags"

# Compter les clés par namespace
127.0.0.1:6379> SCAN 0 MATCH user:* COUNT 1000
# (retourne toutes les clés user:*)

# Vérifier le type de chaque structure
127.0.0.1:6379> TYPE post:1:meta
hash
127.0.0.1:6379> TYPE post:1:comments
list
127.0.0.1:6379> TYPE post:1:tags
set
```

### 3. Récupérer des informations

```bash
# Profil complet d'un utilisateur
127.0.0.1:6379> HGETALL user:123:profile
1) "name"
2) "Alice"
3) "email"
4) "alice@blog.com"
5) "bio"
6) "Rédactrice tech"

# Métadonnées d'un article
127.0.0.1:6379> HGETALL post:1:meta
1) "title"
2) "Introduction à Redis"
3) "author_id"
4) "123"
5) "views"
6) "0"

# Tous les commentaires (ordre LIFO)
127.0.0.1:6379> LRANGE post:1:comments 0 -1
1) "Très clair, merci"
2) "Super article !"

# Tous les tags
127.0.0.1:6379> SMEMBERS post:1:tags
1) "nosql"
2) "database"
3) "redis"
```

### 4. Mettre à jour les données

```bash
# Incrémenter les vues
127.0.0.1:6379> HINCRBY post:1:meta views 1
(integer) 1

# Ajouter un nouveau commentaire
127.0.0.1:6379> LPUSH post:1:comments "J'ai appris plein de choses"
(integer) 3

# Ajouter un tag
127.0.0.1:6379> SADD post:1:tags "tutorial"
(integer) 1

# Vérifier
127.0.0.1:6379> HGET post:1:meta views
"1"
127.0.0.1:6379> LLEN post:1:comments
(integer) 3
127.0.0.1:6379> SCARD post:1:tags
(integer) 4
```

---

## 🎨 Visualisation dans Redis Insight

Avec la structure ci-dessus, Redis Insight afficherait :

```
📁 user:
  📁 123:
    📊 profile (hash)    → 3 fields
    📦 posts (set)       → 3 members

📁 post:
  📁 1:
    📊 meta (hash)       → 3 fields
    📄 content (string)  → "Redis est une..."
    📝 comments (list)   → 3 items
    📦 tags (set)        → 4 members
```

**Avantages** :
- Vision claire de la hiérarchie (grâce aux `:`)
- Navigation intuitive par dossiers
- Modification en direct des valeurs
- Aperçu rapide du nombre d'éléments

---

## 🔍 Commandes avancées de navigation

### RENAME : Renommer une clé

```bash
127.0.0.1:6379> SET oldkey "value"
OK
127.0.0.1:6379> RENAME oldkey newkey
OK
127.0.0.1:6379> GET newkey
"value"
127.0.0.1:6379> GET oldkey
(nil)

# RENAMENX : Renommer uniquement si la destination n'existe pas
127.0.0.1:6379> SET key1 "value1"
OK
127.0.0.1:6379> SET key2 "value2"
OK
127.0.0.1:6379> RENAMENX key1 key2
(integer) 0  # Échec car key2 existe
```

### MOVE : Déplacer une clé vers une autre base

```bash
# Déplacer de la base 0 à la base 1
127.0.0.1:6379> SET mykey "value"
OK
127.0.0.1:6379> MOVE mykey 1
(integer) 1

127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> GET mykey
"value"
```

### COPY : Copier une clé (Redis 6.2+)

```bash
# Copier vers une nouvelle clé
127.0.0.1:6379> SET source "original"
OK
127.0.0.1:6379> COPY source destination
(integer) 1
127.0.0.1:6379> GET destination
"original"

# COPY avec REPLACE pour écraser si existe
127.0.0.1:6379> COPY source destination REPLACE
(integer) 1
```

### RANDOMKEY : Obtenir une clé aléatoire

```bash
127.0.0.1:6379> RANDOMKEY
"user:123:name"

127.0.0.1:6379> RANDOMKEY
"post:1:content"

# Utile pour les tests ou échantillonnage
```

---

## 🛠️ Tips et astuces redis-cli

### 1. Mode interactif amélioré

```bash
# Afficher les commandes au fur et à mesure
$ redis-cli --verbose

# Format de sortie en CSV
$ redis-cli --csv LRANGE mylist 0 -1

# Format de sortie en JSON (Redis 7+)
$ redis-cli --json GET mykey
```

### 2. Exécution en one-liner

```bash
# Exécuter une commande sans entrer en mode interactif
$ redis-cli SET hello "world"
OK

$ redis-cli GET hello
"world"

# Pipe et redirection
$ echo "PING" | redis-cli
PONG

$ redis-cli --scan --pattern "user:*" > users_keys.txt
```

### 3. Mode REPL avec historique

```bash
# redis-cli garde l'historique dans ~/.rediscli_history
127.0.0.1:6379> # Utilisez ↑ et ↓ pour naviguer

# Recherche dans l'historique avec Ctrl+R
(reverse-i-search)`set': SET mykey "value"
```

### 4. Aide intégrée

```bash
# Aide générale
127.0.0.1:6379> HELP

# Aide par catégorie
127.0.0.1:6379> HELP @string
127.0.0.1:6379> HELP @list
127.0.0.1:6379> HELP @set

# Aide spécifique à une commande
127.0.0.1:6379> HELP ZADD
127.0.0.1:6379> HELP SCAN
```

### 5. Auto-complétion

Dans redis-cli, appuyez sur **Tab** pour auto-compléter :
```bash
127.0.0.1:6379> HG<Tab>
HGET      HGETALL   HINCRBY   HINCRBYFLOAT
```

---

## 🚨 Pièges courants à éviter

### 1. Utiliser KEYS en production

```bash
# ❌ INTERDIT en production (bloque Redis)
KEYS user:*

# ✅ Utilisez SCAN à la place
SCAN 0 MATCH user:* COUNT 100
```

### 2. Oublier les namespaces

```bash
# ❌ Difficile à gérer
SET 123name "Alice"
SET 123email "alice@example.com"

# ✅ Structure claire
SET user:123:name "Alice"
SET user:123:email "alice@example.com"
```

### 3. Créer des clés trop longues

```bash
# ❌ Gaspillage de mémoire (90 bytes d'overhead par clé !)
SET application:production:region:eu-west:microservice:api:endpoint:users:cache:query:result:list:active "data"

# ✅ Plus compact
SET app:prod:eu:api:users:cache "data"
```

### 4. Ne pas typer les clés dans le nom

```bash
# ❌ Impossible de savoir le type sans TYPE
SET data:123 "value"
LPUSH data:456 "item"

# ✅ Incluez un indice du type
SET cache:string:123 "value"
LPUSH queue:list:456 "item"
```

### 5. Utiliser DEL sur de grosses structures

```bash
# ❌ Bloque Redis pendant la suppression
DEL huge:list:with:millions:of:items

# ✅ Utilisez UNLINK (async)
UNLINK huge:list:with:millions:of:items
```

---

## 📊 Récapitulatif des commandes de navigation

| Commande | Usage | Complexité | Production |
|----------|-------|------------|-----------|
| `SET/GET` | Lire/écrire une clé | O(1) | ✅ Safe |
| `EXISTS` | Vérifier l'existence | O(1) | ✅ Safe |
| `DEL` | Supprimer (synchrone) | O(N) | ⚠️ Attention |
| `UNLINK` | Supprimer (async) | O(1) | ✅ Safe |
| `TYPE` | Identifier le type | O(1) | ✅ Safe |
| `KEYS` | Lister avec pattern | O(N) | ❌ Jamais |
| `SCAN` | Itération incrémentale | O(1) par appel | ✅ Safe |
| `DBSIZE` | Compter les clés | O(1) | ✅ Safe |
| `RENAME` | Renommer une clé | O(1) | ✅ Safe |
| `COPY` | Copier une clé | O(N) | ⚠️ Attention |
| `FLUSHDB` | Tout supprimer | O(N) | ❌ Danger |

---

## 🎓 Bonnes pratiques : Checklist

Avant de continuer vers les structures de données spécifiques, assurez-vous de :

- ✅ Avoir choisi une convention de nommage cohérente
- ✅ Utiliser SCAN au lieu de KEYS
- ✅ Connaître la différence entre DEL et UNLINK
- ✅ Avoir installé Redis Insight pour la visualisation
- ✅ Comprendre que Redis est un espace de noms "flat"
- ✅ Savoir utiliser les commandes de base (SET, GET, EXISTS, TYPE)

---

## 🚀 Prochaine étape

Maintenant que vous maîtrisez la navigation et le modèle clé-valeur, vous êtes prêt à plonger dans les **structures de données** proprement dites !

➡️ **Section suivante** : [2.2 Strings : Caching, Compteurs et opérations atomiques](./02-strings-caching-compteurs.md)

---

**Durée estimée** : 1 heure
**Niveau** : Débutant
**Prérequis** : Module 1.6 (Installation et outils)

⏭️ [Strings : Caching, Compteurs et opérations atomiques](/02-structures-donnees-natives/02-strings-caching-compteurs.md)
