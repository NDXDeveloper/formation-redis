🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.5 Sets : Unicité et opérations ensemblistes

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Comment les Sets garantissent l'unicité automatique
- ✅ Les opérations ensemblistes (union, intersection, différence)
- ✅ L'échantillonnage aléatoire avec SPOP et SRANDMEMBER
- ✅ Les cas d'usage réels (tags, followers, permissions)
- ✅ La différence entre Sets et Sorted Sets

---

## 📘 Les Sets : Collections uniques non ordonnées

### Qu'est-ce qu'un Set dans Redis ?

Un **Set** est une **collection non ordonnée d'éléments uniques**. Pensez à un ensemble mathématique : chaque élément n'apparaît qu'une seule fois, et il n'y a pas d'ordre spécifique.

```bash
# Visualisation d'un Set
tags:article:42 → { "redis", "nosql", "database", "cache" }
                    ↑ Pas d'ordre, pas de doublons
```

**Caractéristiques** :
- ✅ **Unicité garantie** : impossible d'avoir des doublons
- ✅ **Non ordonné** : pas de notion de premier/dernier
- ✅ Ajout/suppression/vérification en **O(1)**
- ✅ Maximum théorique : **2³² - 1 membres** (~4 milliards)
- ✅ Les membres sont des **strings**

### Pourquoi utiliser des Sets ?

Les Sets sont **idéaux** pour :
- 🏷️ **Tags et catégories** : articles, produits, médias
- 👥 **Relations sociales** : followers, following, amis
- 🔐 **Permissions et rôles** : qui a accès à quoi
- ✅ **Déduplication** : tracking de vues uniques
- 🎲 **Échantillonnage aléatoire** : tirer au sort, rotation

---

## 🔧 Opérations de base

### SADD : Ajouter des membres

```bash
# Ajouter un membre
127.0.0.1:6379> SADD tags:article:1 "redis"
(integer) 1  # 1 membre ajouté

# Ajouter plusieurs membres à la fois
127.0.0.1:6379> SADD tags:article:1 "nosql" "database" "cache"
(integer) 3

# Essayer d'ajouter un doublon
127.0.0.1:6379> SADD tags:article:1 "redis"
(integer) 0  # 0 = déjà présent, non ajouté

# Vérifier le contenu
127.0.0.1:6379> SMEMBERS tags:article:1
1) "cache"
2) "database"
3) "nosql"
4) "redis"
# ⚠️ Ordre non garanti ! Peut varier d'un appel à l'autre
```

**Important** : L'ordre d'affichage n'est **pas garanti**. Ne comptez jamais sur un ordre spécifique avec les Sets.

### SISMEMBER : Vérifier l'appartenance

```bash
# Vérifier si un membre existe
127.0.0.1:6379> SISMEMBER tags:article:1 "redis"
(integer) 1  # 1 = présent

127.0.0.1:6379> SISMEMBER tags:article:1 "python"
(integer) 0  # 0 = absent

# Très rapide : O(1) !
```

### SMISMEMBER : Vérifier plusieurs membres (Redis 6.2+)

```bash
# Vérifier plusieurs membres en une seule commande
127.0.0.1:6379> SMISMEMBER tags:article:1 "redis" "python" "nosql" "java"
1) (integer) 1  # "redis" présent
2) (integer) 0  # "python" absent
3) (integer) 1  # "nosql" présent
4) (integer) 0  # "java" absent
```

**Avantage** : Réduit les RTT (1 requête au lieu de N).

### SCARD : Compter les membres

```bash
# Nombre de membres dans le Set
127.0.0.1:6379> SCARD tags:article:1
(integer) 4

# Très rapide : O(1)
```

### SMEMBERS : Récupérer tous les membres

```bash
# Obtenir tous les membres
127.0.0.1:6379> SMEMBERS tags:article:1
1) "cache"
2) "database"
3) "nosql"
4) "redis"
```

⚠️ **Attention** : SMEMBERS est **O(N)**. Sur un Set avec 100 000 membres, cela peut être lent. Utilisez SSCAN pour de gros Sets.

### SREM : Supprimer des membres

```bash
# Supprimer un membre
127.0.0.1:6379> SREM tags:article:1 "cache"
(integer) 1  # 1 membre supprimé

# Supprimer plusieurs membres
127.0.0.1:6379> SREM tags:article:1 "nosql" "database"
(integer) 2

# Vérifier
127.0.0.1:6379> SMEMBERS tags:article:1
1) "redis"

# Supprimer un membre inexistant
127.0.0.1:6379> SREM tags:article:1 "python"
(integer) 0  # Pas supprimé car absent
```

---

## 🎲 Échantillonnage aléatoire

### SRANDMEMBER : Récupérer un membre aléatoire

```bash
# Créer un Set
127.0.0.1:6379> SADD colors "red" "green" "blue" "yellow" "purple"
(integer) 5

# Récupérer UN membre aléatoire (sans le retirer)
127.0.0.1:6379> SRANDMEMBER colors
"blue"

127.0.0.1:6379> SRANDMEMBER colors
"red"  # Différent à chaque fois

# Récupérer N membres aléatoires uniques
127.0.0.1:6379> SRANDMEMBER colors 3
1) "yellow"
2) "green"
3) "purple"

# Avec COUNT négatif : peut retourner des doublons
127.0.0.1:6379> SRANDMEMBER colors -7
1) "blue"
2) "red"
3) "blue"   # Doublon possible
4) "green"
5) "yellow"
6) "red"    # Doublon possible
7) "purple"
```

**Cas d'usage** : Sélection aléatoire, recommendations, A/B testing.

### SPOP : Retirer et récupérer aléatoirement

```bash
# Créer un Set
127.0.0.1:6379> SADD lottery "ticket1" "ticket2" "ticket3" "ticket4" "ticket5"
(integer) 5

# Tirer un ticket aléatoire ET le retirer
127.0.0.1:6379> SPOP lottery
"ticket3"

127.0.0.1:6379> SCARD lottery
(integer) 4  # Il reste 4 tickets

# Tirer plusieurs tickets
127.0.0.1:6379> SPOP lottery 2
1) "ticket1"
2) "ticket5"

127.0.0.1:6379> SMEMBERS lottery
1) "ticket2"
2) "ticket4"
```

**Cas d'usage** : Loteries, tirage au sort, distribution de tâches unique.

---

## 🔗 Opérations ensemblistes

### SUNION : Union (tous les membres de plusieurs Sets)

```bash
# Créer plusieurs Sets
127.0.0.1:6379> SADD set1 "a" "b" "c"
(integer) 3

127.0.0.1:6379> SADD set2 "c" "d" "e"
(integer) 3

127.0.0.1:6379> SADD set3 "e" "f" "g"
(integer) 3

# Union : tous les éléments uniques
127.0.0.1:6379> SUNION set1 set2 set3
1) "a"
2) "b"
3) "c"
4) "d"
5) "e"
6) "f"
7) "g"
# 7 éléments uniques (duplicata "c" et "e" fusionnés)
```

**Schéma** :
```
set1: {a, b, c}
set2: {c, d, e}
set3: {e, f, g}
         ↓
SUNION: {a, b, c, d, e, f, g}
```

### SINTER : Intersection (membres communs)

```bash
# Intersection : membres présents dans TOUS les Sets
127.0.0.1:6379> SINTER set1 set2
1) "c"  # Seul "c" est dans set1 ET set2

# Intersection de 3 Sets
127.0.0.1:6379> SINTER set1 set2 set3
(empty array)  # Aucun membre commun aux 3

# Autre exemple
127.0.0.1:6379> SADD languages:alice "python" "javascript" "go"
(integer) 3

127.0.0.1:6379> SADD languages:bob "python" "java" "go"
(integer) 3

127.0.0.1:6379> SINTER languages:alice languages:bob
1) "python"
2) "go"
# Langages que Alice ET Bob connaissent
```

**Schéma** :
```
set1: {a, b, c}
set2: {c, d, e}
         ↓
SINTER: {c}  (intersection)
```

### SDIFF : Différence (membres de A mais pas de B)

```bash
# Différence : éléments dans set1 mais PAS dans set2
127.0.0.1:6379> SDIFF set1 set2
1) "a"
2) "b"
# "c" n'apparaît pas car il est aussi dans set2

# Ordre important !
127.0.0.1:6379> SDIFF set2 set1
1) "d"
2) "e"
# Éléments dans set2 mais PAS dans set1

# Différence avec plusieurs Sets
127.0.0.1:6379> SDIFF set1 set2 set3
1) "a"
2) "b"
# Éléments dans set1 mais ni dans set2 ni dans set3
```

**Schéma** :
```
set1: {a, b, c}
set2: {c, d, e}
         ↓
SDIFF set1 set2: {a, b}
SDIFF set2 set1: {d, e}
```

---

## 💾 Stocker les résultats : SUNIONSTORE, SINTERSTORE, SDIFFSTORE

Au lieu de simplement retourner les résultats, vous pouvez les **stocker** dans une nouvelle clé :

```bash
# Créer des Sets de tags
127.0.0.1:6379> SADD articles:redis "article1" "article2" "article3"
(integer) 3

127.0.0.1:6379> SADD articles:nosql "article2" "article3" "article4"
(integer) 3

# Stocker l'union
127.0.0.1:6379> SUNIONSTORE articles:redis-or-nosql articles:redis articles:nosql
(integer) 4  # 4 articles au total

127.0.0.1:6379> SMEMBERS articles:redis-or-nosql
1) "article1"
2) "article2"
3) "article3"
4) "article4"

# Stocker l'intersection
127.0.0.1:6379> SINTERSTORE articles:redis-and-nosql articles:redis articles:nosql
(integer) 2

127.0.0.1:6379> SMEMBERS articles:redis-and-nosql
1) "article2"
2) "article3"

# Stocker la différence
127.0.0.1:6379> SDIFFSTORE articles:only-redis articles:redis articles:nosql
(integer) 1

127.0.0.1:6379> SMEMBERS articles:only-redis
1) "article1"
```

**Cas d'usage** :
- Pré-calculer des résultats pour des requêtes fréquentes
- Créer des Sets temporaires avec TTL

```bash
# Calculer et mettre en cache pendant 1 heure
SINTERSTORE cache:common-interests user:123:interests user:456:interests
EXPIRE cache:common-interests 3600
```

---

## 📊 Cas d'usage #1 : Système de tags

```bash
# Tagger des articles
127.0.0.1:6379> SADD article:1:tags "redis" "nosql" "database"
(integer) 3

127.0.0.1:6379> SADD article:2:tags "redis" "cache" "performance"
(integer) 3

127.0.0.1:6379> SADD article:3:tags "nosql" "mongodb" "database"
(integer) 3

# Vérifier si un article a un tag spécifique
127.0.0.1:6379> SISMEMBER article:1:tags "redis"
(integer) 1  # Oui

# Compter les tags d'un article
127.0.0.1:6379> SCARD article:1:tags
(integer) 3

# Ajouter un nouveau tag
127.0.0.1:6379> SADD article:1:tags "tutorial"
(integer) 1

# Retirer un tag
127.0.0.1:6379> SREM article:2:tags "performance"
(integer) 1

# Index inversé : articles par tag
127.0.0.1:6379> SADD tag:redis:articles "1" "2"
(integer) 2

127.0.0.1:6379> SADD tag:nosql:articles "1" "3"
(integer) 2

# Trouver tous les articles avec le tag "redis"
127.0.0.1:6379> SMEMBERS tag:redis:articles
1) "1"
2) "2"

# Trouver les articles avec "redis" ET "nosql" (intersection)
127.0.0.1:6379> SINTER tag:redis:articles tag:nosql:articles
1) "1"  # Seul l'article 1 a les deux tags

# Trouver les articles avec "redis" OU "nosql" (union)
127.0.0.1:6379> SUNION tag:redis:articles tag:nosql:articles
1) "1"
2) "2"
3) "3"
```

---

## 👥 Cas d'usage #2 : Réseau social (followers)

```bash
# Alice suit Bob et Charlie
127.0.0.1:6379> SADD user:alice:following "bob" "charlie"
(integer) 2

# Bob suit Alice et Dave
127.0.0.1:6379> SADD user:bob:following "alice" "dave"
(integer) 2

# Charlie suit Alice
127.0.0.1:6379> SADD user:charlie:following "alice"
(integer) 1

# Followers (inverse)
127.0.0.1:6379> SADD user:alice:followers "bob" "charlie"
(integer) 2

127.0.0.1:6379> SADD user:bob:followers "alice"
(integer) 1

# Combien de personnes Alice suit-elle ?
127.0.0.1:6379> SCARD user:alice:following
(integer) 2

# Combien de followers a Alice ?
127.0.0.1:6379> SCARD user:alice:followers
(integer) 2

# Alice suit-elle Bob ?
127.0.0.1:6379> SISMEMBER user:alice:following "bob"
(integer) 1  # Oui

# Alice suit-elle Dave ?
127.0.0.1:6379> SISMEMBER user:alice:following "dave"
(integer) 0  # Non

# Amis communs entre Alice et Bob (intersection)
127.0.0.1:6379> SINTER user:alice:following user:bob:following
1) "alice"  # Hmm, pas idéal (Bob suit Alice)

# Meilleur : qui Alice ET Bob suivent tous les deux ?
127.0.0.1:6379> SADD user:alice:following "eve"
(integer) 1
127.0.0.1:6379> SADD user:bob:following "eve"
(integer) 1

127.0.0.1:6379> SINTER user:alice:following user:bob:following
1) "alice"
2) "eve"

# Personnes qu'Alice suit mais pas Bob (différence)
127.0.0.1:6379> SDIFF user:alice:following user:bob:following
1) "bob"
2) "charlie"

# Suggestions : amis d'amis (followers de followers)
# Qui suit Charlie ? (amis potentiels pour Alice)
127.0.0.1:6379> SMEMBERS user:charlie:following
1) "alice"
```

---

## 🔐 Cas d'usage #3 : Système de permissions

```bash
# Définir les permissions d'un utilisateur
127.0.0.1:6379> SADD user:alice:permissions "read" "write" "delete"
(integer) 3

127.0.0.1:6379> SADD user:bob:permissions "read" "write"
(integer) 2

# Vérifier une permission
127.0.0.1:6379> SISMEMBER user:alice:permissions "delete"
(integer) 1  # Alice peut supprimer

127.0.0.1:6379> SISMEMBER user:bob:permissions "delete"
(integer) 0  # Bob ne peut pas supprimer

# Révoquer une permission
127.0.0.1:6379> SREM user:alice:permissions "delete"
(integer) 1

# Accorder une permission
127.0.0.1:6379> SADD user:bob:permissions "admin"
(integer) 1

# Permissions par ressource
127.0.0.1:6379> SADD resource:document:123:viewers "alice" "bob" "charlie"
(integer) 3

127.0.0.1:6379> SADD resource:document:123:editors "alice" "bob"
(integer) 2

# Alice peut-elle voir le document ?
127.0.0.1:6379> SISMEMBER resource:document:123:viewers "alice"
(integer) 1  # Oui

# Dave peut-il éditer ?
127.0.0.1:6379> SISMEMBER resource:document:123:editors "dave"
(integer) 0  # Non

# Qui peut à la fois voir ET éditer ?
127.0.0.1:6379> SINTER resource:document:123:viewers resource:document:123:editors
1) "alice"
2) "bob"
```

---

## 🎯 Cas d'usage #4 : Déduplication de vues

```bash
# Tracker les utilisateurs ayant vu un article
127.0.0.1:6379> SADD article:42:viewers "user:123"
(integer) 1

127.0.0.1:6379> SADD article:42:viewers "user:456"
(integer) 1

# Utilisateur 123 revisite l'article
127.0.0.1:6379> SADD article:42:viewers "user:123"
(integer) 0  # Déjà compté, pas de doublon

# Nombre de vues uniques
127.0.0.1:6379> SCARD article:42:viewers
(integer) 2

# L'utilisateur 789 a-t-il vu l'article ?
127.0.0.1:6379> SISMEMBER article:42:viewers "user:789"
(integer) 0  # Non

# Tous les utilisateurs ayant vu l'article
127.0.0.1:6379> SMEMBERS article:42:viewers
1) "user:123"
2) "user:456"
```

**Avec TTL pour nettoyage automatique** :
```bash
# Tracking des vues uniques du jour
127.0.0.1:6379> SADD views:2024-12-09:article:42 "user:123"
(integer) 1

127.0.0.1:6379> EXPIRE views:2024-12-09:article:42 86400
(integer) 1  # Expire dans 24 heures

# Vues uniques de la semaine : union des 7 derniers jours
127.0.0.1:6379> SUNION views:2024-12-03:article:42 views:2024-12-04:article:42 views:2024-12-05:article:42 views:2024-12-06:article:42 views:2024-12-07:article:42 views:2024-12-08:article:42 views:2024-12-09:article:42
# (retourne tous les viewers uniques de la semaine)
```

---

## 🔄 SMOVE : Déplacer un membre entre Sets

```bash
# Créer deux Sets
127.0.0.1:6379> SADD todo "task1" "task2" "task3"
(integer) 3

127.0.0.1:6379> SADD done "task0"
(integer) 1

# Déplacer task2 de "todo" vers "done"
127.0.0.1:6379> SMOVE todo done "task2"
(integer) 1  # Succès

127.0.0.1:6379> SMEMBERS todo
1) "task1"
2) "task3"

127.0.0.1:6379> SMEMBERS done
1) "task0"
2) "task2"

# Si le membre n'existe pas dans la source
127.0.0.1:6379> SMOVE todo done "task99"
(integer) 0  # Échec
```

**Cas d'usage** : Workflows, statuts de tâches (todo → in-progress → done).

---

## 🔍 SSCAN : Scanner de gros Sets

Comme SCAN et HSCAN, SSCAN permet d'itérer sur un Set sans bloquer Redis.

```bash
# Créer un gros Set (imaginez 100 000 membres)
127.0.0.1:6379> SADD huge:set "member1" "member2" "member3"
# ... (beaucoup de membres)

# Scanner par batches
127.0.0.1:6379> SSCAN huge:set 0 COUNT 10
1) "17"  # Curseur suivant
2) 1) "member1"
   2) "member2"
   3) "member3"
   # ... jusqu'à 10 membres

# Continuer avec le curseur
127.0.0.1:6379> SSCAN huge:set 17 COUNT 10
# ...

# Scanner avec pattern matching
127.0.0.1:6379> SSCAN huge:set 0 MATCH user:* COUNT 100
```

**Quand utiliser SSCAN** :
- ✅ Set avec > 10 000 membres
- ✅ En production, éviter de bloquer Redis
- ❌ Petits Sets (< 1000 membres) → SMEMBERS suffit

---

## ⚡ Complexité et performance

| Commande | Complexité | Notes |
|----------|------------|-------|
| `SADD` | O(1) | Par membre ajouté |
| `SREM` | O(N) | N = membres à supprimer |
| `SISMEMBER` | O(1) | Très rapide |
| `SMISMEMBER` | O(N) | N = membres à vérifier |
| `SCARD` | O(1) | |
| `SMEMBERS` | O(N) | N = taille du Set, attention ! |
| `SRANDMEMBER` | O(1) | Sans COUNT |
| `SRANDMEMBER` | O(N) | Avec COUNT |
| `SPOP` | O(1) | Sans COUNT |
| `SPOP` | O(N) | Avec COUNT |
| `SUNION` | O(N) | N = taille de tous les Sets |
| `SINTER` | O(N*M) | N = plus petit Set, M = nombre de Sets |
| `SDIFF` | O(N) | N = taille de tous les Sets |
| `SMOVE` | O(1) | |
| `SSCAN` | O(1) | Par appel (itération) |

**Optimisation** : SINTER est plus rapide si vous mettez le **plus petit Set en premier**.

```bash
# ✅ Optimal
SINTER small:set huge:set  # Redis commence par small:set

# ❌ Moins optimal
SINTER huge:set small:set
```

---

## 🚨 Pièges courants à éviter

### 1. SMEMBERS sur de gros Sets

```bash
# ❌ DANGEREUX : Set avec 1 million de membres
SMEMBERS huge:users  # Peut bloquer Redis !

# ✅ Utilisez SSCAN
SSCAN huge:users 0 COUNT 1000
```

### 2. Oublier que les Sets ne sont PAS ordonnés

```bash
127.0.0.1:6379> SADD numbers "1" "2" "3" "4" "5"
(integer) 5

127.0.0.1:6379> SMEMBERS numbers
1) "3"
2) "1"
3) "5"
4) "2"
5) "4"
# ⚠️ Ordre aléatoire ! Ne pas compter dessus

# ✅ Si vous avez besoin d'ordre, utilisez un Sorted Set
ZADD sorted:numbers 1 "1" 2 "2" 3 "3" 4 "4" 5 "5"
ZRANGE sorted:numbers 0 -1
```

### 3. Confondre SDIFF et SINTER

```bash
# SDIFF : éléments dans A mais PAS dans B
SDIFF setA setB  # setA - setB

# SINTER : éléments dans A ET B
SINTER setA setB  # setA ∩ setB

# Bien comprendre la différence !
```

### 4. Ne pas profiter de SMISMEMBER (Redis 6.2+)

```bash
# ❌ Multiple RTT
exists1 = redis.sismember("myset", "a")
exists2 = redis.sismember("myset", "b")
exists3 = redis.sismember("myset", "c")

# ✅ Un seul RTT (Redis 6.2+)
results = redis.smismember("myset", "a", "b", "c")
```

### 5. Utiliser SPOP alors que vous voulez SRANDMEMBER

```bash
# SPOP RETIRE le membre
127.0.0.1:6379> SPOP myset
"value"  # Le membre est SUPPRIMÉ du Set

# SRANDMEMBER ne retire PAS
127.0.0.1:6379> SRANDMEMBER myset
"value"  # Le membre reste dans le Set

# Choisissez selon votre besoin !
```

---

## 🆚 Sets vs Sorted Sets : Quand utiliser quoi ?

### Utilisez un **Set** si :

```bash
# ✅ Pas besoin d'ordre
SADD tags "redis" "nosql" "cache"

# ✅ Vérification d'appartenance simple
SISMEMBER users:online "alice"

# ✅ Opérations ensemblistes (union, intersection)
SINTER interests:alice interests:bob

# ✅ Échantillonnage aléatoire
SRANDMEMBER winners 3

# ✅ Unicité sans ordre ni score
SADD unique:visitors "user:123"
```

### Utilisez un **Sorted Set** si :

```bash
# ✅ Besoin d'ordre par score
ZADD leaderboard 1500 "player1"

# ✅ Classement, ranking
ZREVRANGE leaderboard 0 9  # Top 10

# ✅ Range queries par score
ZRANGEBYSCORE prices 10 50  # Produits entre 10€ et 50€

# ✅ Données temporelles ordonnées
ZADD events 1733748000 "event1"  # timestamp comme score
```

**Tableau de comparaison** :

| Critère | Set | Sorted Set |
|---------|-----|------------|
| Ordre | ❌ Non ordonné | ✅ Ordonné par score |
| Unicité | ✅ Oui | ✅ Oui |
| Accès par score | ❌ Non | ✅ Oui (ZRANGEBYSCORE) |
| Opérations ensemblistes | ✅ SUNION, SINTER, SDIFF | ⚠️ ZUNION, ZINTER (plus complexe) |
| Échantillonnage aléatoire | ✅ SRANDMEMBER, SPOP | ❌ Non natif |
| Complexité ajout | O(1) | O(log N) |
| Cas d'usage | Tags, followers, permissions | Leaderboards, priorités, timelines |

---

## 📋 Checklist : Quand utiliser un Set

### ✅ Utilisez un Set pour :
- Collections **uniques** sans ordre spécifique
- **Déduplication** automatique
- Vérification rapide d'**appartenance** (O(1))
- **Opérations ensemblistes** (union, intersection, différence)
- **Tags, catégories, labels**
- Relations **many-to-many** (followers, permissions)
- **Échantillonnage aléatoire** (loterie, rotation)

### ❌ N'utilisez PAS un Set pour :
- Données **ordonnées** → Sorted Set ou List
- Besoins de **classement/ranking** → Sorted Set
- Stocker des **valeurs avec métadonnées** → Hash
- **Compteurs** → Sorted Set (score = compteur) ou String (INCR)
- **Doublons autorisés** → List

---

## 🎓 Points clés à retenir

1. ✅ **Set = collection unique non ordonnée** : pas de doublons, pas d'ordre garanti
2. ✅ **SISMEMBER en O(1)** : vérification d'appartenance ultra-rapide
3. ✅ **Opérations ensemblistes** : SUNION, SINTER, SDIFF pour combiner des Sets
4. ✅ **SRANDMEMBER** : échantillonnage sans retirer, SPOP : avec retrait
5. ✅ **SMEMBERS = O(N)** : attention aux gros Sets, utilisez SSCAN
6. ⚠️ **Pas d'ordre** : si vous avez besoin d'ordre, utilisez Sorted Set ou List
7. ⚠️ **Membres = strings** : pas de scores, pas de métadonnées
8. 🎯 Parfait pour : tags, followers, permissions, déduplication

---

## 🚀 Prochaine étape

Maintenant que vous maîtrisez les Sets pour les collections uniques, découvrons les **Sorted Sets** pour ajouter la notion de score et d'ordre !

➡️ **Section suivante** : [2.6 Sorted Sets : Leaderboards, Géospatial et indexation](./06-sorted-sets-leaderboards-geospatial.md)

---

**Durée estimée** : 1h30
**Niveau** : Débutant à Intermédiaire
**Prérequis** : Sections 2.1 à 2.4 complétées

⏭️ [Sorted Sets : Leaderboards, Géospatial et indexation](/02-structures-donnees-natives/06-sorted-sets-leaderboards-geospatial.md)
