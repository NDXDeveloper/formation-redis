🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 2 : Structures de données natives (Redis Core)

## 🎯 Objectifs du module

Ce module constitue le **cœur de Redis** et la base de tout ce que vous construirez par la suite. Vous allez découvrir les 8 structures de données natives qui font la puissance et la polyvalence de Redis, bien au-delà d'un simple cache clé-valeur.

À la fin de ce module, vous serez capable de :
- ✅ Choisir la structure de données optimale pour chaque cas d'usage
- ✅ Manipuler efficacement les commandes de chaque type
- ✅ Comprendre les implications en termes de performance (complexité O)
- ✅ Éviter les pièges courants et les anti-patterns

---

## 🧠 Pourquoi Redis n'est pas "juste un cache"

Beaucoup de développeurs découvrent Redis comme un **cache** rapide pour soulager leur base de données. C'est vrai, mais c'est réducteur ! Redis est une **structure de données serveur** (data structures server), ce qui signifie que vous pouvez stocker et manipuler des types de données complexes directement dans Redis, sans avoir à tout sérialiser/désérialiser.

### Exemple concret : Compteur de likes

**Approche naïve (cache simple)** :
```bash
# ❌ Inefficace : récupérer, modifier, réécrire
GET article:123:likes      # "42"
# Dans votre code : parse, +1, serialize
SET article:123:likes 43
```

**Approche Redis (opération atomique)** :
```bash
# ✅ Efficace : opération atomique serveur-side
INCR article:123:likes     # Retourne 43 directement
# Pas de race condition, pas de parsing !
```

Cette différence peut sembler minime, mais elle devient **critique** sous charge avec des milliers d'utilisateurs simultanés.

---

## 📚 Vue d'ensemble des structures natives

Redis Core propose **8 types de données** principaux, chacun optimisé pour des cas d'usage spécifiques :

| Structure | Icône | Cas d'usage principal | Complexité typique |
|-----------|-------|----------------------|-------------------|
| **Strings** | 🔤 | Cache, compteurs, flags | O(1) |
| **Lists** | 📝 | Files d'attente, logs, timeline | O(1) aux extrémités |
| **Hashes** | 🗂️ | Objets, profils utilisateurs | O(1) par champ |
| **Sets** | 🎲 | Tags, followers, déduplication | O(1) add/remove |
| **Sorted Sets** | 🏆 | Leaderboards, priorités, scores | O(log N) |
| **HyperLogLog** | 📊 | Comptage unique approché | O(1) constant |
| **Bitmaps** | 🔲 | Analytics, flags binaires | O(1) |
| **Streams** | 🌊 | Event sourcing, logs, IoT | O(1) append |

> **Note** : Les Streams sont techniquement dans Redis Core depuis Redis 5.0, bien qu'ils soient souvent considérés comme une fonctionnalité avancée.

---

## 🎓 Comment aborder ce module

### 1️⃣ Commencez par les Strings (section 2.2)
Même si cela semble basique, les Strings sont **omniprésents** et comprennent des subtilités importantes (atomicité, opérations bitwise, etc.).

### 2️⃣ Progressez selon vos besoins
Vous n'avez **pas besoin** de maîtriser toutes les structures immédiatement. Voici nos recommandations :

**Parcours minimum** (développeur web) :
- Strings (cache, sessions)
- Hashes (objets utilisateurs)
- Sets (tags, permissions)
- Sorted Sets (leaderboards, tris)

**Parcours avancé** (architecture système) :
- Ajoutez Lists (queues)
- HyperLogLog (analytics)
- Bitmaps (optimisation mémoire)

### 3️⃣ Testez chaque exemple
Tous les exemples de ce module sont **exécutables directement** dans `redis-cli`. Nous vous encourageons à :
```bash
# Lancer Redis localement
docker run -d -p 6379:6379 redis:latest

# Se connecter
redis-cli

# Tester les exemples
127.0.0.1:6379> SET hello "world"
OK
127.0.0.1:6379> GET hello
"world"
```

---

## 🔍 Aperçu rapide : Quand utiliser quelle structure ?

### Scénario 1 : "Je veux stocker une session utilisateur"
```bash
# ✅ Solution : Hash (un objet avec plusieurs champs)
HSET session:abc123 user_id 42 username "alice" last_seen 1638360000
EXPIRE session:abc123 3600  # Expire dans 1 heure
```

### Scénario 2 : "Je veux un système de suivi de visiteurs uniques"
```bash
# ✅ Solution : HyperLogLog (comptage approximatif en O(1))
PFADD visitors:2024-12-09 user123
PFADD visitors:2024-12-09 user456
PFCOUNT visitors:2024-12-09  # Retourne ~2 (approximation)
```

### Scénario 3 : "Je veux un leaderboard de jeu en temps réel"
```bash
# ✅ Solution : Sorted Set (tri par score)
ZADD leaderboard 1500 "player1"
ZADD leaderboard 2300 "player2"
ZADD leaderboard 1800 "player3"

# Top 10 des meilleurs scores
ZREVRANGE leaderboard 0 9 WITHSCORES
```

### Scénario 4 : "Je veux une file d'attente de jobs"
```bash
# ✅ Solution : List (FIFO avec LPUSH/RPOP)
LPUSH jobs:queue "process-payment-123"
LPUSH jobs:queue "send-email-456"

# Worker consomme
RPOP jobs:queue  # Retourne "process-payment-123"
```

### Scénario 5 : "Je veux suivre quels utilisateurs ont aimé un post"
```bash
# ✅ Solution : Set (unicité automatique)
SADD post:789:likes user123
SADD post:789:likes user456
SADD post:789:likes user123  # Ignoré (déjà présent)

SCARD post:789:likes  # Retourne 2
```

---

## 🚨 Avertissement sur la complexité

Chaque commande Redis a une **complexité algorithmique** (notée en Big O). Cette notion est **cruciale** en production :

```bash
# ❌ DANGER : O(N) - Peut bloquer Redis si N est grand
KEYS user:*           # Scanne TOUTES les clés !
SMEMBERS huge_set     # Retourne TOUS les membres !

# ✅ SAFE : O(1) ou O(log N)
SCAN 0 MATCH user:*   # Itération par batch
SISMEMBER huge_set member123  # Vérification en O(1)
```

**Règle d'or** : En production, évitez les commandes O(N) sur de grandes collections. Nous détaillerons cela dans la section 2.9.

---

## 🗺️ Plan du module

### Section 2.1 : Le modèle clé-valeur et navigation
Introduction au système de clés de Redis, conventions de nommage et outils de navigation (redis-cli, Redis Insight).

### Section 2.2 : Strings - Caching, Compteurs et opérations atomiques
La structure la plus simple mais la plus utilisée. Cache, compteurs, flags, et opérations bitwise.

### Section 2.3 : Lists - Files d'attente simples
Listes chaînées pour créer des queues FIFO/LIFO, timelines et logs.

### Section 2.4 : Hashes - Représentation d'objets et optimisation mémoire
Stocker des objets avec plusieurs champs, optimisation mémoire pour petites hashes.

### Section 2.5 : Sets - Unicité et opérations ensemblistes
Collections non ordonnées, déduplication, intersections, unions et différences.

### Section 2.6 : Sorted Sets - Leaderboards, Géospatial et indexation
Collections ordonnées par score, le couteau suisse de Redis pour le ranking.

### Section 2.7 : HyperLogLog - Comptage unique probabiliste
Compter des éléments uniques avec une mémoire constante (12 KB max !).

### Section 2.8 : Bitmaps - Gestion efficace d'états binaires
Manipulation bit à bit pour analytics, présence, et optimisation mémoire extrême.

### Section 2.9 : Complexité algorithmique (Big O) des commandes
Comprendre la performance de chaque commande pour éviter les blocages en production.

---

## 💡 Conseils avant de commencer

### 1. Nommage des clés
Adoptez une convention dès le début :
```bash
# ✅ Bon : namespace:type:id:field
user:profile:123:email
product:inventory:456:stock

# ❌ Mauvais : pas de structure
u123email
prod_456_inv
```

### 2. Pensez "opérations atomiques"
Redis est single-threaded, ce qui garantit l'atomicité des commandes individuelles :
```bash
# ✅ Cette séquence est atomique
INCR views:article:123
SADD viewed:user:42 article:123
```

### 3. Utilisez EXPIRE dès que possible
Évitez l'accumulation de données inutiles :
```bash
SET cache:query:abc "result"
EXPIRE cache:query:abc 300  # 5 minutes

# Ou en une seule commande (Redis 6.2+)
SET cache:query:abc "result" EX 300
```

### 4. Redis-cli est votre ami
Avant de coder, testez dans redis-cli :
```bash
127.0.0.1:6379> HELP @string    # Aide sur les commandes String
127.0.0.1:6379> HELP ZADD       # Aide spécifique sur ZADD
127.0.0.1:6379> INFO memory     # Stats mémoire
```

---

## 📊 Comparaison avec d'autres bases de données

Si vous venez d'un monde **SQL/NoSQL**, voici quelques correspondances :

| Concept SQL | Concept MongoDB | Structure Redis équivalente |
|-------------|-----------------|----------------------------|
| Table avec colonnes | Collection avec documents | Hash par ligne |
| Index unique | Index unique | Set des valeurs |
| ORDER BY score | .sort() | Sorted Set |
| COUNT DISTINCT | $group + $sum | HyperLogLog |
| Colonne booléenne | Boolean field | Bitmap |

**Exemple** : Stocker des utilisateurs

**SQL** :
```sql
CREATE TABLE users (id INT, name VARCHAR, email VARCHAR);
INSERT INTO users VALUES (1, 'Alice', 'alice@example.com');
```

**Redis** :
```bash
HSET user:1 name "Alice" email "alice@example.com"
```

---

## 🎯 Objectif final

À la fin de ce module, vous devriez être capable de regarder un problème et penser immédiatement :

> "Ah, c'est un cas pour un **Sorted Set** avec un score basé sur le timestamp !"

ou

> "Ici, un **HyperLogLog** serait parfait pour compter les visiteurs uniques sans exploser la mémoire."

C'est cette **intuition** que nous allons développer ensemble à travers les 9 sections de ce module.

---

## 📖 Lectures préalables recommandées

Avant de plonger dans les structures, assurez-vous d'avoir compris :
- ✅ Le modèle in-memory de Redis (Module 1)
- ✅ L'architecture single-threaded (Module 1.5)
- ✅ Les bases de redis-cli (Module 1.6)

Si ces concepts ne sont pas clairs, nous vous recommandons de revoir le Module 1 avant de continuer.

---

## 🚀 Prêt à commencer ?

Les structures de données Redis sont comme des **outils dans une boîte à outils** : chacun a sa spécialité. Apprenons maintenant à les maîtriser !

➡️ **Commencez par la section 2.1** : [Le modèle clé-valeur et navigation](./01-modele-cle-valeur-et-navigation.md)

---

**Durée estimée du module** : 6-8 heures
**Niveau** : Débutant à Intermédiaire
**Prérequis** : Module 1 complété

⏭️ [Le modèle clé-valeur et navigation (CLI & GUI)](/02-structures-donnees-natives/01-modele-cle-valeur-et-navigation.md)
