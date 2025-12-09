🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.6 Sorted Sets : Leaderboards, Géospatial et indexation

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Comment les Sorted Sets combinent unicité et ordre
- ✅ Les opérations de range (ZRANGE, ZRANGEBYSCORE, ZRANGEBYLEX)
- ✅ L'implémentation de leaderboards temps réel
- ✅ Les commandes géospatiales (GEOADD, GEORADIUS)
- ✅ Les cas d'usage avancés (priorités, timestamps, indexation)

---

## 📘 Les Sorted Sets : Collections ordonnées par score

### Qu'est-ce qu'un Sorted Set ?

Un **Sorted Set** (ZSet) est une collection où chaque membre est associé à un **score** (nombre flottant). Les membres sont **automatiquement ordonnés** par leur score, du plus petit au plus grand.

```bash
# Visualisation d'un Sorted Set
leaderboard → {
    "alice":  1500,  ← membre: score
    "bob":    2300,
    "charlie": 1800
}

# Ordre automatique par score :
1. "alice"   (1500)
2. "charlie" (1800)
3. "bob"     (2300)
```

**Caractéristiques** :
- ✅ **Membres uniques** : comme un Set
- ✅ **Ordonnés par score** : tri automatique
- ✅ **Score = nombre flottant** (double precision)
- ✅ Ajout/mise à jour en **O(log N)**
- ✅ Accès par rang ou par score
- ✅ Maximum théorique : **2³² - 1 membres** (~4 milliards)

### Pourquoi utiliser des Sorted Sets ?

Les Sorted Sets sont **idéaux** pour :
- 🏆 **Leaderboards** : classements de jeux, rankings
- ⏰ **Files de priorité** : tâches avec priorité/urgence
- 📅 **Timelines** : événements ordonnés par timestamp
- 📍 **Géolocalisation** : proximité géographique
- 📊 **Top N** : produits les plus vendus, articles populaires
- 🔢 **Indexation secondaire** : requêtes par plage de valeurs

---

## 🔧 Commandes de base

### ZADD : Ajouter des membres avec leur score

```bash
# Ajouter un membre avec son score
127.0.0.1:6379> ZADD leaderboard 1500 "alice"
(integer) 1  # 1 membre ajouté

# Ajouter plusieurs membres
127.0.0.1:6379> ZADD leaderboard 2300 "bob" 1800 "charlie"
(integer) 2

# Syntaxe : ZADD key score member [score member ...]

# Modifier le score d'un membre existant
127.0.0.1:6379> ZADD leaderboard 1600 "alice"
(integer) 0  # 0 = membre existant mis à jour

# Options de ZADD (Redis 3.0.2+)
127.0.0.1:6379> ZADD leaderboard NX 2000 "dave"
# NX = ajouter seulement si n'existe pas
(integer) 1

127.0.0.1:6379> ZADD leaderboard NX 2500 "dave"
# Échec car dave existe déjà
(integer) 0

127.0.0.1:6379> ZADD leaderboard XX 2500 "dave"
# XX = mettre à jour seulement si existe
(integer) 0  # 0 membres ajoutés (mais score mis à jour)

# ZADD avec GT (Greater Than) : met à jour seulement si nouveau score > ancien
127.0.0.1:6379> ZADD leaderboard GT 2400 "dave"
(integer) 0  # Pas ajouté, mais pas mis à jour car 2400 < 2500

127.0.0.1:6379> ZADD leaderboard GT 2600 "dave"
(integer) 0  # Score mis à jour car 2600 > 2500

# ZADD avec LT (Less Than) : met à jour seulement si nouveau score < ancien
127.0.0.1:6379> ZADD leaderboard LT 2400 "dave"
(integer) 0  # Score mis à jour car 2400 < 2600

# ZADD avec CH (CHanged) : retourne le nombre d'éléments modifiés
127.0.0.1:6379> ZADD leaderboard CH 2500 "dave" 1900 "eve"
(integer) 2  # dave mis à jour + eve ajouté

# ZADD avec INCR : incrémenter le score (équivalent à ZINCRBY)
127.0.0.1:6379> ZADD leaderboard INCR 100 "alice"
"1700"  # Nouveau score d'alice
```

### ZRANGE : Récupérer par rang (index)

```bash
# Créer un Sorted Set
127.0.0.1:6379> ZADD scores 100 "alice" 200 "bob" 150 "charlie" 180 "dave"
(integer) 4

# Récupérer du rang 0 à 2 (3 premiers, ordre croissant)
127.0.0.1:6379> ZRANGE scores 0 2
1) "alice"    # score 100
2) "charlie"  # score 150
3) "dave"     # score 180

# Récupérer TOUS les membres
127.0.0.1:6379> ZRANGE scores 0 -1
1) "alice"
2) "charlie"
3) "dave"
4) "bob"

# Avec les scores (WITHSCORES)
127.0.0.1:6379> ZRANGE scores 0 -1 WITHSCORES
1) "alice"
2) "100"
3) "charlie"
4) "150"
5) "dave"
6) "180"
7) "bob"
8) "200"

# Indices négatifs : -1 = dernier, -2 = avant-dernier
127.0.0.1:6379> ZRANGE scores -2 -1 WITHSCORES
1) "dave"
2) "180"
3) "bob"
4) "200"
```

### ZREVRANGE : Récupérer en ordre inverse (décroissant)

```bash
# Top 3 des meilleurs scores (ordre décroissant)
127.0.0.1:6379> ZREVRANGE scores 0 2 WITHSCORES
1) "bob"      # score 200 (le plus haut)
2) "dave"     # score 180
3) "charlie"  # score 150

# Tous les membres en ordre décroissant
127.0.0.1:6379> ZREVRANGE scores 0 -1
1) "bob"
2) "dave"
3) "charlie"
4) "alice"
```

**Astuce** : Utilisez ZREVRANGE pour les leaderboards (du meilleur au pire).

### ZRANK et ZREVRANK : Obtenir le rang d'un membre

```bash
# Rang en ordre croissant (0-based)
127.0.0.1:6379> ZRANK scores "alice"
(integer) 0  # Premier (score le plus bas)

127.0.0.1:6379> ZRANK scores "bob"
(integer) 3  # Quatrième (score le plus haut)

# Rang en ordre décroissant
127.0.0.1:6379> ZREVRANK scores "bob"
(integer) 0  # Premier (meilleur score)

127.0.0.1:6379> ZREVRANK scores "alice"
(integer) 3  # Quatrième (moins bon score)

# Si le membre n'existe pas
127.0.0.1:6379> ZRANK scores "zoe"
(nil)
```

### ZSCORE : Obtenir le score d'un membre

```bash
# Récupérer le score d'un membre
127.0.0.1:6379> ZSCORE scores "alice"
"100"

127.0.0.1:6379> ZSCORE scores "bob"
"200"

# Si le membre n'existe pas
127.0.0.1:6379> ZSCORE scores "unknown"
(nil)
```

### ZCARD : Nombre de membres

```bash
# Compter les membres
127.0.0.1:6379> ZCARD scores
(integer) 4

# Très rapide : O(1)
```

### ZREM : Supprimer des membres

```bash
# Supprimer un membre
127.0.0.1:6379> ZREM scores "charlie"
(integer) 1  # 1 membre supprimé

# Supprimer plusieurs membres
127.0.0.1:6379> ZREM scores "dave" "eve"
(integer) 1  # dave supprimé (eve n'existait pas)

# Vérifier
127.0.0.1:6379> ZRANGE scores 0 -1
1) "alice"
2) "bob"
```

---

## 🔢 Opérations sur les scores

### ZINCRBY : Incrémenter le score

```bash
# Créer un leaderboard
127.0.0.1:6379> ZADD game:points 0 "alice" 0 "bob" 0 "charlie"
(integer) 3

# Alice gagne 10 points
127.0.0.1:6379> ZINCRBY game:points 10 "alice"
"10"

# Bob gagne 15 points
127.0.0.1:6379> ZINCRBY game:points 15 "bob"
"15"

# Alice gagne encore 5 points
127.0.0.1:6379> ZINCRBY game:points 5 "alice"
"15"  # Total : 15

# Vérifier le classement
127.0.0.1:6379> ZREVRANGE game:points 0 -1 WITHSCORES
1) "alice"
2) "15"
3) "bob"
4) "15"
5) "charlie"
6) "0"

# Décrémenter (score négatif)
127.0.0.1:6379> ZINCRBY game:points -5 "alice"
"10"
```

**Cas d'usage** : Points de jeu, votes, popularité.

### ZMSCORE : Obtenir les scores de plusieurs membres (Redis 6.2+)

```bash
# Récupérer plusieurs scores en une seule commande
127.0.0.1:6379> ZMSCORE game:points "alice" "bob" "charlie" "dave"
1) "10"
2) "15"
3) "0"
4) (nil)  # dave n'existe pas
```

---

## 📊 Range queries : Requêtes par plage

### ZRANGEBYSCORE : Récupérer par plage de scores

```bash
# Créer un Sorted Set de prix
127.0.0.1:6379> ZADD products 10 "item1" 25 "item2" 50 "item3" 75 "item4" 100 "item5"
(integer) 5

# Produits entre 20€ et 60€
127.0.0.1:6379> ZRANGEBYSCORE products 20 60
1) "item2"  # 25
2) "item3"  # 50

# Avec les scores
127.0.0.1:6379> ZRANGEBYSCORE products 20 60 WITHSCORES
1) "item2"
2) "25"
3) "item3"
4) "50"

# Intervalles ouverts : ( = exclu, [ = inclu (par défaut)
127.0.0.1:6379> ZRANGEBYSCORE products (20 60
1) "item3"  # Exclut 20, donc pas item2 (25)
# Attend, erreur dans mon exemple. Corrigeons :

127.0.0.1:6379> ZRANGEBYSCORE products (25 60
1) "item3"  # Exclut 25, donc pas item2

127.0.0.1:6379> ZRANGEBYSCORE products 20 (50
1) "item2"  # Exclut 50, donc pas item3

# -inf et +inf : moins l'infini et plus l'infini
127.0.0.1:6379> ZRANGEBYSCORE products -inf 30
1) "item1"
2) "item2"

127.0.0.1:6379> ZRANGEBYSCORE products 70 +inf
1) "item4"
2) "item5"

# LIMIT : pagination (offset, count)
127.0.0.1:6379> ZRANGEBYSCORE products -inf +inf LIMIT 0 2
1) "item1"
2) "item2"

127.0.0.1:6379> ZRANGEBYSCORE products -inf +inf LIMIT 2 2
1) "item3"
2) "item4"
```

### ZREVRANGEBYSCORE : Par plage de scores en ordre inverse

```bash
# Même chose mais en ordre décroissant
127.0.0.1:6379> ZREVRANGEBYSCORE products 60 20 WITHSCORES
1) "item3"
2) "50"
3) "item2"
4) "25"

# ⚠️ Attention : les bornes sont inversées (max min, pas min max)
127.0.0.1:6379> ZREVRANGEBYSCORE products 100 50
1) "item5"
2) "item4"
3) "item3"
```

### ZRANGEBYLEX : Récupérer par ordre lexicographique

Quand tous les membres ont le **même score**, vous pouvez faire des requêtes lexicographiques (alphabétiques).

```bash
# Créer un Sorted Set avec score identique (0)
127.0.0.1:6379> ZADD words 0 "apple" 0 "banana" 0 "cherry" 0 "date" 0 "elderberry"
(integer) 5

# Tous les mots entre "b" et "d" (lexicographique)
127.0.0.1:6379> ZRANGEBYLEX words [b [d
1) "banana"
2) "cherry"
3) "date"

# [ = inclusif, ( = exclusif
127.0.0.1:6379> ZRANGEBYLEX words (b (d
1) "cherry"  # Exclut "banana" et "date"

# Tous les mots commençant par "a" à "c"
127.0.0.1:6379> ZRANGEBYLEX words [a [c
1) "apple"
2) "banana"
3) "cherry"

# - et + : début et fin
127.0.0.1:6379> ZRANGEBYLEX words - [c
1) "apple"
2) "banana"
3) "cherry"

127.0.0.1:6379> ZRANGEBYLEX words [d +
1) "date"
2) "elderberry"

# LIMIT pour pagination
127.0.0.1:6379> ZRANGEBYLEX words - + LIMIT 0 3
1) "apple"
2) "banana"
3) "cherry"
```

**Cas d'usage** : Auto-complétion, suggestions, recherche préfixe.

---

## 🏆 Cas d'usage #1 : Leaderboard de jeu

### Leaderboard basique

```bash
# Initialiser les scores
127.0.0.1:6379> ZADD game:leaderboard 0 "player1" 0 "player2" 0 "player3"
(integer) 3

# Joueur 1 marque 100 points
127.0.0.1:6379> ZINCRBY game:leaderboard 100 "player1"
"100"

# Joueur 2 marque 150 points
127.0.0.1:6379> ZINCRBY game:leaderboard 150 "player2"
"150"

# Joueur 3 marque 120 points
127.0.0.1:6379> ZINCRBY game:leaderboard 120 "player3"
"120"

# Joueur 1 marque encore 50 points
127.0.0.1:6379> ZINCRBY game:leaderboard 50 "player1"
"150"

# Top 10 des meilleurs joueurs
127.0.0.1:6379> ZREVRANGE game:leaderboard 0 9 WITHSCORES
1) "player1"
2) "150"
3) "player2"
4) "150"
5) "player3"
6) "120"

# Rang d'un joueur spécifique
127.0.0.1:6379> ZREVRANK game:leaderboard "player3"
(integer) 2  # Troisième place

# Score d'un joueur
127.0.0.1:6379> ZSCORE game:leaderboard "player1"
"150"

# Nombre total de joueurs
127.0.0.1:6379> ZCARD game:leaderboard
(integer) 3

# Joueurs avec plus de 130 points
127.0.0.1:6379> ZRANGEBYSCORE game:leaderboard 130 +inf WITHSCORES
1) "player1"
2) "150"
3) "player2"
4) "150"
```

### Leaderboard avec contexte (nom + score + rang)

```python
# Pseudo-code Python
def get_leaderboard_top(n=10):
    """Top N joueurs"""
    return redis.zrevrange("game:leaderboard", 0, n-1, withscores=True)

def get_player_rank(player_id):
    """Rang et score d'un joueur"""
    rank = redis.zrevrank("game:leaderboard", player_id)
    score = redis.zscore("game:leaderboard", player_id)
    total = redis.zcard("game:leaderboard")

    return {
        "rank": rank + 1,  # +1 car 0-based
        "score": score,
        "total_players": total
    }

def get_nearby_players(player_id, context=2):
    """Joueurs autour d'un joueur (contexte)"""
    rank = redis.zrevrank("game:leaderboard", player_id)

    start = max(0, rank - context)
    end = rank + context

    return redis.zrevrange("game:leaderboard", start, end, withscores=True)

# Exemple d'affichage
# 1. Alice - 2500 pts
# 2. Bob - 2300 pts
# 3. Charlie - 2100 pts  ← Joueur actuel
# 4. Dave - 2000 pts
# 5. Eve - 1900 pts
```

---

## ⏰ Cas d'usage #2 : File de priorité

```bash
# Ajouter des tâches avec priorité (plus petit = plus urgent)
127.0.0.1:6379> ZADD tasks 1 "critical-bug-fix"
(integer) 1

127.0.0.1:6379> ZADD tasks 5 "feature-request"
(integer) 1

127.0.0.1:6379> ZADD tasks 3 "refactoring"
(integer) 1

127.0.0.1:6379> ZADD tasks 2 "security-patch"
(integer) 1

# Récupérer la tâche la plus urgente
127.0.0.1:6379> ZRANGE tasks 0 0
1) "critical-bug-fix"  # Priorité 1

# Récupérer et supprimer la tâche la plus urgente
127.0.0.1:6379> ZPOPMIN tasks
1) "critical-bug-fix"
2) "1"

# Prochaine tâche
127.0.0.1:6379> ZPOPMIN tasks
1) "security-patch"
2) "2"

# ZPOPMAX : récupérer la moins urgente
127.0.0.1:6379> ZPOPMAX tasks
1) "feature-request"
2) "5"

# Récupérer les 3 tâches les plus urgentes sans les supprimer
127.0.0.1:6379> ZADD tasks 1 "bug1" 2 "bug2" 3 "bug3" 4 "bug4"
(integer) 4

127.0.0.1:6379> ZRANGE tasks 0 2 WITHSCORES
1) "bug1"
2) "1"
3) "bug2"
4) "2"
5) "bug3"
6) "3"
```

**ZPOPMIN et ZPOPMAX** (Redis 5.0+) :
```bash
# ZPOPMIN : retire et retourne le membre avec le score le plus bas
127.0.0.1:6379> ZPOPMIN tasks 2
1) "bug1"
2) "1"
3) "bug2"
4) "2"

# ZPOPMAX : retire et retourne le membre avec le score le plus haut
127.0.0.1:6379> ZPOPMAX tasks 1
1) "bug4"
2) "4"
```

---

## 📅 Cas d'usage #3 : Timeline avec timestamps

```bash
# Stocker des événements avec leur timestamp comme score
127.0.0.1:6379> ZADD timeline 1733745600 "event1:User logged in"
(integer) 1

127.0.0.1:6379> ZADD timeline 1733749200 "event2:API call"
(integer) 1

127.0.0.1:6379> ZADD timeline 1733752800 "event3:User logged out"
(integer) 1

# Événements entre deux timestamps
127.0.0.1:6379> ZRANGEBYSCORE timeline 1733745600 1733750000
1) "event1:User logged in"
2) "event2:API call"

# Derniers 10 événements
127.0.0.1:6379> ZREVRANGE timeline 0 9
1) "event3:User logged out"
2) "event2:API call"
3) "event1:User logged in"

# Événements des dernières 24 heures
# timestamp_now = 1733760000
# timestamp_24h_ago = timestamp_now - 86400
127.0.0.1:6379> ZRANGEBYSCORE timeline 1733673600 1733760000
# (retourne les événements de la journée)

# Supprimer les événements de plus de 7 jours
127.0.0.1:6379> ZREMRANGEBYSCORE timeline -inf 1733155200
(integer) 0  # Nombre d'événements supprimés
```

**Astuce** : Utilisez des timestamps Unix comme scores pour des requêtes temporelles.

---

## 📍 Cas d'usage #4 : Géolocalisation

Redis utilise les Sorted Sets pour stocker des coordonnées géographiques !

### GEOADD : Ajouter des positions

```bash
# Ajouter des positions (longitude, latitude, nom)
127.0.0.1:6379> GEOADD cities 2.3522 48.8566 "Paris"
(integer) 1

127.0.0.1:6379> GEOADD cities -0.1276 51.5074 "London" 13.4050 52.5200 "Berlin"
(integer) 2

# En interne, Redis stocke ça dans un Sorted Set
127.0.0.1:6379> TYPE cities
zset

# Vous pouvez voir les scores (encodage geohash)
127.0.0.1:6379> ZRANGE cities 0 -1 WITHSCORES
1) "London"
2) "3663832405125283"
3) "Paris"
4) "3663850803137628"
5) "Berlin"
6) "3677832748890298"
```

### GEODIST : Distance entre deux points

```bash
# Distance entre Paris et London (par défaut en mètres)
127.0.0.1:6379> GEODIST cities "Paris" "London"
"343575.8671"  # ~344 km

# Avec unité spécifique
127.0.0.1:6379> GEODIST cities "Paris" "London" km
"343.5759"

127.0.0.1:6379> GEODIST cities "Paris" "Berlin" km
"877.4559"

# Unités disponibles : m, km, mi (miles), ft (feet)
127.0.0.1:6379> GEODIST cities "Paris" "London" mi
"213.5163"  # miles
```

### GEORADIUS : Trouver des points dans un rayon

```bash
# Villes dans un rayon de 500 km autour de Paris
127.0.0.1:6379> GEORADIUS cities 2.3522 48.8566 500 km
1) "Paris"
2) "London"

# Avec distances
127.0.0.1:6379> GEORADIUS cities 2.3522 48.8566 500 km WITHDIST
1) 1) "Paris"
   2) "0.0000"
2) 1) "London"
   2) "343.5759"

# Avec coordonnées
127.0.0.1:6379> GEORADIUS cities 2.3522 48.8566 500 km WITHCOORD
1) 1) "Paris"
   2) 1) "2.35219955444335938"
      2) "48.85661220395509474"
2) 1) "London"
   2) 1) "-0.12759864330291748"
      2) "51.50739773636909416"

# Avec distance + coordonnées + ordre par distance
127.0.0.1:6379> GEORADIUS cities 2.3522 48.8566 1000 km WITHDIST WITHCOORD ASC
1) 1) "Paris"
   2) "0.0000"
   3) 1) "2.35219955444335938"
      2) "48.85661220395509474"
2) 1) "London"
   2) "343.5759"
   3) 1) "-0.12759864330291748"
      2) "51.50739773636909416"
3) 1) "Berlin"
   2) "877.4559"
   3) 1) "13.40500175952911377"
      2) "52.52000108120943819"

# Limiter le nombre de résultats
127.0.0.1:6379> GEORADIUS cities 2.3522 48.8566 1000 km COUNT 2
1) "Paris"
2) "London"
```

### GEORADIUSBYMEMBER : Rayon autour d'un membre existant

```bash
# Villes dans un rayon de 600 km autour de Paris
127.0.0.1:6379> GEORADIUSBYMEMBER cities "Paris" 600 km WITHDIST
1) 1) "Paris"
   2) "0.0000"
2) 1) "London"
   2) "343.5759"

# Plus simple que de redemander les coordonnées de Paris !
```

### GEOPOS : Obtenir les coordonnées

```bash
# Récupérer les coordonnées d'une ou plusieurs villes
127.0.0.1:6379> GEOPOS cities "Paris" "London"
1) 1) "2.35219955444335938"
   2) "48.85661220395509474"
2) 1) "-0.12759864330291748"
   2) "51.50739773636909416"
```

### GEOSEARCH : Recherche géospatiale moderne (Redis 6.2+)

```bash
# Remplace GEORADIUS avec une syntaxe plus claire
127.0.0.1:6379> GEOSEARCH cities FROMMEMBER "Paris" BYRADIUS 500 km
1) "Paris"
2) "London"

# Recherche par boîte (rectangle)
127.0.0.1:6379> GEOSEARCH cities FROMLONLAT 2.3522 48.8566 BYBOX 1000 1000 km
1) "Paris"
2) "London"
3) "Berlin"
```

**Cas d'usage géospatial** :
- 🏪 Trouver les magasins les plus proches
- 🚕 Matching chauffeur-passager (Uber-like)
- 🍕 Livraison de nourriture (restaurants à proximité)
- 📱 Applications de rencontre (utilisateurs proches)

---

## 🔢 Opérations de comptage

### ZCOUNT : Compter dans une plage de scores

```bash
# Créer un Sorted Set
127.0.0.1:6379> ZADD ages 25 "alice" 30 "bob" 22 "charlie" 35 "dave" 28 "eve"
(integer) 5

# Combien ont entre 25 et 30 ans inclus ?
127.0.0.1:6379> ZCOUNT ages 25 30
(integer) 3  # alice, bob, eve

# Combien ont moins de 30 ans ?
127.0.0.1:6379> ZCOUNT ages -inf 30
(integer) 4

# Combien ont plus de 30 ans ?
127.0.0.1:6379> ZCOUNT ages (30 +inf
(integer) 1  # dave (35)
```

### ZLEXCOUNT : Compter dans une plage lexicographique

```bash
# Avec notre Set de mots (tous score = 0)
127.0.0.1:6379> ZLEXCOUNT words [a [c
(integer) 3  # apple, banana, cherry

127.0.0.1:6379> ZLEXCOUNT words [b +
(integer) 4  # banana, cherry, date, elderberry
```

---

## 🔄 Opérations ensemblistes sur Sorted Sets

### ZUNION : Union de Sorted Sets (Redis 6.2+)

```bash
# Créer deux Sorted Sets
127.0.0.1:6379> ZADD votes:2024-01 10 "alice" 20 "bob" 15 "charlie"
(integer) 3

127.0.0.1:6379> ZADD votes:2024-02 12 "alice" 18 "bob" 25 "dave"
(integer) 3

# Union : additionner les scores
127.0.0.1:6379> ZUNION 2 votes:2024-01 votes:2024-02 WITHSCORES
1) "charlie"
2) "15"
3) "alice"
4) "22"   # 10 + 12
5) "dave"
6) "25"
7) "bob"
8) "38"   # 20 + 18

# Par défaut : SUM (additionner)
# Options : MIN (prendre le min), MAX (prendre le max)
127.0.0.1:6379> ZUNION 2 votes:2024-01 votes:2024-02 AGGREGATE MIN WITHSCORES
1) "alice"
2) "10"   # min(10, 12)
3) "charlie"
4) "15"
5) "bob"
6) "18"   # min(20, 18)
7) "dave"
8) "25"

127.0.0.1:6379> ZUNION 2 votes:2024-01 votes:2024-02 AGGREGATE MAX WITHSCORES
1) "charlie"
2) "15"
3) "alice"
4) "12"   # max(10, 12)
5) "bob"
6) "20"   # max(20, 18)
7) "dave"
8) "25"

# Avec poids (multiplier les scores)
127.0.0.1:6379> ZUNION 2 votes:2024-01 votes:2024-02 WEIGHTS 1 2 WITHSCORES
1) "charlie"
2) "15"      # 15 * 1
3) "alice"
4) "34"      # (10 * 1) + (12 * 2)
5) "bob"
6) "56"      # (20 * 1) + (18 * 2)
7) "dave"
8) "50"      # 25 * 2
```

### ZINTER : Intersection de Sorted Sets (Redis 6.2+)

```bash
# Membres présents dans les DEUX Sets
127.0.0.1:6379> ZINTER 2 votes:2024-01 votes:2024-02 WITHSCORES
1) "alice"
2) "22"   # 10 + 12
3) "bob"
4) "38"   # 20 + 18

# Seuls alice et bob sont dans les deux Sets
```

### ZUNIONSTORE et ZINTERSTORE : Stocker les résultats

```bash
# Stocker l'union dans une nouvelle clé
127.0.0.1:6379> ZUNIONSTORE votes:total 2 votes:2024-01 votes:2024-02
(integer) 4  # 4 membres au total

127.0.0.1:6379> ZRANGE votes:total 0 -1 WITHSCORES
1) "charlie"
2) "15"
3) "alice"
4) "22"
5) "dave"
6) "25"
7) "bob"
8) "38"

# Stocker l'intersection
127.0.0.1:6379> ZINTERSTORE votes:common 2 votes:2024-01 votes:2024-02
(integer) 2

127.0.0.1:6379> ZRANGE votes:common 0 -1 WITHSCORES
1) "alice"
2) "22"
3) "bob"
4) "38"
```

---

## 🗑️ Suppression par rang ou par score

### ZREMRANGEBYRANK : Supprimer par rang

```bash
# Créer un Sorted Set
127.0.0.1:6379> ZADD numbers 1 "one" 2 "two" 3 "three" 4 "four" 5 "five"
(integer) 5

# Supprimer les 2 premiers (rangs 0 et 1)
127.0.0.1:6379> ZREMRANGEBYRANK numbers 0 1
(integer) 2  # 2 membres supprimés

127.0.0.1:6379> ZRANGE numbers 0 -1
1) "three"
2) "four"
3) "five"

# Supprimer les N derniers
127.0.0.1:6379> ZREMRANGEBYRANK numbers -1 -1
(integer) 1  # "five" supprimé
```

### ZREMRANGEBYSCORE : Supprimer par plage de scores

```bash
# Créer un Sorted Set
127.0.0.1:6379> ZADD scores 10 "a" 20 "b" 30 "c" 40 "d" 50 "e"
(integer) 5

# Supprimer les scores entre 20 et 40
127.0.0.1:6379> ZREMRANGEBYSCORE scores 20 40
(integer) 3  # b, c, d supprimés

127.0.0.1:6379> ZRANGE scores 0 -1 WITHSCORES
1) "a"
2) "10"
3) "e"
4) "50"
```

### ZREMRANGEBYLEX : Supprimer par plage lexicographique

```bash
# Avec notre Set de mots (score = 0)
127.0.0.1:6379> ZREMRANGEBYLEX words [b [d
(integer) 3  # banana, cherry, date supprimés

127.0.0.1:6379> ZRANGE words 0 -1
1) "apple"
2) "elderberry"
```

---

## 🔍 ZSCAN : Scanner de gros Sorted Sets

```bash
# Scanner un Sorted Set par batches
127.0.0.1:6379> ZSCAN leaderboard 0 COUNT 10
1) "17"  # Curseur suivant
2) 1) "player1"
   2) "150"
   3) "player2"
   4) "120"
   # ... jusqu'à 10 membres

# Continuer avec le curseur
127.0.0.1:6379> ZSCAN leaderboard 17 COUNT 10
# ...

# Scanner avec pattern matching
127.0.0.1:6379> ZSCAN leaderboard 0 MATCH player:* COUNT 100
```

---

## ⚡ Complexité et performance

| Commande | Complexité | Notes |
|----------|------------|-------|
| `ZADD` | O(log N) | Par membre ajouté |
| `ZREM` | O(M log N) | M = membres à supprimer |
| `ZRANGE/ZREVRANGE` | O(log N + M) | M = éléments retournés |
| `ZRANK/ZREVRANK` | O(log N) | |
| `ZSCORE` | O(1) | |
| `ZCARD` | O(1) | |
| `ZINCRBY` | O(log N) | |
| `ZRANGEBYSCORE` | O(log N + M) | M = éléments retournés |
| `ZCOUNT` | O(log N) | |
| `ZUNION/ZINTER` | O(N*K)+O(M*log M) | N = plus grand Set, K = nombre de Sets |
| `ZPOPMIN/ZPOPMAX` | O(log N * M) | M = nombre d'éléments pop |
| `GEOADD` | O(log N) | Par élément |
| `GEORADIUS` | O(N+log M) | N = éléments dans rayon, M = total |

**Note importante** : Les Sorted Sets utilisent une **skip list** + **hash table** en interne, d'où le O(log N) pour la plupart des opérations.

---

## 🚨 Pièges courants à éviter

### 1. Confondre ZRANGE et ZRANGEBYSCORE

```bash
# ZRANGE : par RANG (index)
ZRANGE myset 0 10  # Les 11 premiers membres (rangs 0 à 10)

# ZRANGEBYSCORE : par SCORE
ZRANGEBYSCORE myset 0 10  # Membres avec scores entre 0 et 10
```

### 2. Oublier WITHSCORES

```bash
# ❌ Difficile de comprendre sans les scores
ZREVRANGE leaderboard 0 9
1) "alice"
2) "bob"
3) "charlie"

# ✅ Avec les scores, c'est clair
ZREVRANGE leaderboard 0 9 WITHSCORES
1) "alice"
2) "2500"
3) "bob"
4) "2300"
5) "charlie"
6) "2100"
```

### 3. ZREVRANGEBYSCORE avec bornes dans le mauvais ordre

```bash
# ❌ ERREUR : bornes inversées
ZREVRANGEBYSCORE products 20 60  # Retourne vide !

# ✅ CORRECT : max avant min
ZREVRANGEBYSCORE products 60 20
```

### 4. Utiliser des floats pour des identifiants

```bash
# ❌ Les scores sont des floats → perte de précision
ZADD users 123456789012345678 "user_id"
ZSCORE users "user_id"
"123456789012345680"  # Arrondi !

# ✅ Utilisez des chaînes pour les IDs
HSET user:123456789012345678 name "Alice"
```

### 5. Ne pas penser aux cas d'égalité de scores

```bash
# Si deux membres ont le même score, l'ordre est lexicographique
127.0.0.1:6379> ZADD myset 10 "zebra" 10 "apple" 10 "mango"
(integer) 3

127.0.0.1:6379> ZRANGE myset 0 -1
1) "apple"   # Ordre alphabétique car même score
2) "mango"
3) "zebra"
```

---

## 📋 Checklist : Quand utiliser un Sorted Set

### ✅ Utilisez un Sorted Set pour :
- **Leaderboards** et classements
- **Files de priorité** (tâches, jobs)
- **Timelines** ordonnées par timestamp
- **Top N** (produits populaires, articles tendances)
- **Range queries** par score ou temps
- **Géolocalisation** (proximité)
- **Auto-complétion** (avec scores lexicographiques)
- Données avec notion de **rang** ou **ordre**

### ❌ N'utilisez PAS un Sorted Set pour :
- Collections **sans ordre** → Set
- Pas besoin de **scores** → Set ou List
- **Structures imbriquées** → Hash ou JSON
- **Doublons autorisés** → List
- Accès **seulement par clé**, sans range queries → Hash

---

## 🎓 Points clés à retenir

1. ✅ **Sorted Set = Set + scores** : unicité + ordre automatique
2. ✅ **O(log N)** pour ajout/suppression : très efficace même avec des millions de membres
3. ✅ **ZRANGE vs ZRANGEBYSCORE** : par rang vs par score
4. ✅ **ZINCRBY** : parfait pour les compteurs ordonnés (leaderboards)
5. ✅ **GEORADIUS** : géolocalisation native avec Sorted Sets
6. ✅ **ZPOPMIN/ZPOPMAX** : file de priorité atomique
7. ⚠️ **Scores = floats** : attention à la précision
8. ⚠️ **Égalité de scores** : ordre lexicographique
9. 🎯 Idéal pour : leaderboards, priorités, timelines, proximité

---

## 🚀 Prochaine étape

Vous maîtrisez maintenant les structures de données principales de Redis ! Explorons les structures plus spécialisées comme **HyperLogLog** pour le comptage unique probabiliste.

➡️ **Section suivante** : [2.7 HyperLogLog : Comptage unique probabiliste](./07-hyperloglog-comptage-unique.md)

---

**Durée estimée** : 2h
**Niveau** : Intermédiaire
**Prérequis** : Sections 2.1 à 2.5 complétées

⏭️ [Structures probabilistes : HyperLogLog](/02-structures-donnees-natives/07-hyperloglog-comptage-unique.md)
