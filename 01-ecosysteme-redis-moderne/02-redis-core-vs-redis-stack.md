🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.2 - Le changement de paradigme : Redis Core vs Redis Stack

## 📋 Introduction

Si vous avez exploré la documentation Redis ou cherché des tutoriels, vous avez probablement rencontré deux termes qui peuvent prêter à confusion : **Redis Core** et **Redis Stack**. S'agit-il de deux produits différents ? Lequel devez-vous apprendre ? Lequel installer ?

Dans cette section, nous allons clarifier cette distinction fondamentale qui représente l'une des évolutions les plus importantes de l'écosystème Redis ces dernières années.

## 🎯 En résumé

> **Redis Core** est la version "classique" et minimaliste de Redis, tandis que **Redis Stack** est une version étendue qui ajoute des capacités avancées via des modules supplémentaires.

Pensez à cela comme la différence entre :
- Un **téléphone basique** (Redis Core) qui fait appels et SMS
- Un **smartphone moderne** (Redis Stack) avec appareil photo, GPS, applications diverses

---

## 1️⃣ Redis Core : Les fondations solides

### Qu'est-ce que Redis Core ?

**Redis Core** est la version originale et minimaliste de Redis, celle qui existe depuis 2009. C'est le "Redis pur" avec uniquement les fonctionnalités fondamentales.

### Les capacités de Redis Core

Redis Core offre un ensemble de **structures de données natives** extrêmement bien optimisées :

| Structure | Description | Cas d'usage typique |
|-----------|-------------|---------------------|
| **Strings** | Chaînes de caractères ou binaires | Cache, compteurs |
| **Lists** | Listes ordonnées | Files d'attente, flux |
| **Sets** | Ensembles non ordonnés | Tags, relations |
| **Sorted Sets** | Ensembles triés par score | Leaderboards, priorités |
| **Hashes** | Objets clé-valeur imbriqués | Profils utilisateur |
| **Bitmaps** | Manipulation de bits | États binaires |
| **HyperLogLog** | Comptage probabiliste | Compteurs uniques |
| **Streams** | Flux de messages | Logs, événements |

### L'analogie de la boîte à outils de base

Imaginez que Redis Core est comme **la boîte à outils d'un artisan** :

```
┌─────────────────────────────────────┐
│     BOÎTE À OUTILS REDIS CORE       │
├─────────────────────────────────────┤
│  🔨 Marteau (Strings)               │
│  🪛 Tournevis (Hashes)              │
│  📏 Mètre (Sorted Sets)             │
│  ✂️  Ciseaux (Lists)                │
│  🔧 Clé anglaise (Sets)             │
└─────────────────────────────────────┘
```

Avec ces outils de base, vous pouvez accomplir **80% des tâches courantes** :
- Créer un cache performant
- Gérer des sessions utilisateur
- Implémenter des compteurs
- Créer des files d'attente
- Faire des leaderboards

### Avantages de Redis Core

#### ✅ Simplicité
- Moins de fonctionnalités = moins de complexité
- Documentation claire et mature
- Courbe d'apprentissage douce

#### ✅ Légèreté
- Empreinte mémoire minimale
- Démarrage rapide
- Parfait pour les environnements contraints

#### ✅ Stabilité
- Code éprouvé depuis 15 ans
- Bugs rarissimes
- Performances prévisibles

#### ✅ Compatibilité universelle
- Supporté par absolument tous les clients Redis
- Aucune dépendance externe
- Fonctionne partout (cloud, on-premise, embedded)

### Limitations de Redis Core

Cependant, pour certains cas d'usage modernes, Redis Core montre ses limites :

#### ❌ Pas de gestion native de JSON
```redis
# Stocker un objet JSON dans Redis Core
SET user:1 '{"name":"Alice","age":30,"city":"Paris"}'

# Problème : Pour modifier juste l'âge, vous devez :
# 1. Récupérer tout le JSON
# 2. Le parser
# 3. Le modifier
# 4. Le sauvegarder entièrement

# C'est inefficace et non-atomique !
```

#### ❌ Pas de recherche full-text
```redis
# Avec Redis Core, impossible de faire :
# "Trouve tous les utilisateurs dont le nom contient 'ali'"

# Vous devez :
# 1. Charger tous les utilisateurs
# 2. Les filtrer dans votre application
# 3. C'est lent et inefficace !
```

#### ❌ Pas de séries temporelles optimisées
Pour stocker des métriques IoT ou du monitoring, Redis Core n'a pas de structure dédiée. Vous devez "bricoler" avec des Sorted Sets, ce qui est loin d'être optimal.

---

## 2️⃣ Redis Stack : L'évolution moderne

### La naissance de Redis Stack

En 2022, Redis Labs (maintenant Redis Inc.) a lancé **Redis Stack**, qui combine :
- **Redis Core** (la base)
- **+ Des modules supplémentaires** (extensions puissantes)

### L'analogie du smartphone

Si Redis Core est un téléphone basique, Redis Stack est un smartphone complet :

```
┌──────────────────────────────────────────┐
│          REDIS STACK                     │
├──────────────────────────────────────────┤
│  📱 Redis Core (le système de base)      │
├──────────────────────────────────────────┤
│  📦 MODULES ADDITIONNELS :               │
│                                          │
│  📄 RedisJSON    → Documents JSON        │
│  🔍 RediSearch   → Recherche full-text   │
│  📊 RedisTimeSeries → Séries temporelles │
│  🎲 RedisBloom   → Filtres probabilistes │
│  📈 RedisGraph   → Base de graphes       │
└──────────────────────────────────────────┘
```

### Les modules de Redis Stack

Détaillons les principaux modules :

#### 1. **RedisJSON** 📄

**Problème résolu** : Manipuler des documents JSON de manière native et efficace.

**Sans RedisJSON (Redis Core)** :
```redis
SET user:1 '{"name":"Alice","age":30,"email":"alice@mail.com"}'
# Pour changer l'âge :
# 1. GET user:1
# 2. Parse JSON dans l'application
# 3. Modifie age
# 4. SET user:1 avec le nouveau JSON complet
```

**Avec RedisJSON** :
```redis
JSON.SET user:1 $ '{"name":"Alice","age":30,"email":"alice@mail.com"}'
# Pour changer juste l'âge :
JSON.SET user:1 $.age 31
# C'est tout ! Atomique et efficace !
```

**Analogie** : C'est comme avoir un éditeur de texte qui vous permet de modifier un seul mot dans un document, au lieu de devoir retaper tout le document.

**Cas d'usage** :
- Sessions utilisateur complexes
- Configuration d'applications
- Catalogues produits e-commerce
- Données de profil utilisateur

#### 2. **RediSearch** 🔍

**Problème résolu** : Faire des recherches full-text, des filtres et des agrégations complexes.

**Sans RediSearch** :
```
Pour trouver "tous les produits contenant 'phone' et coûtant moins de 500€" :
→ Impossible directement dans Redis Core
→ Il faut charger tous les produits dans l'application et filtrer
→ Lent et inefficace
```

**Avec RediSearch** :
```redis
# Créer un index
FT.CREATE products ON JSON
  PREFIX 1 product:
  SCHEMA
    $.name TEXT
    $.price NUMERIC

# Rechercher
FT.SEARCH products "@name:phone @price:[0 500]"
# Résultat instantané !
```

**Analogie** : C'est comme avoir Google intégré dans Redis. Au lieu de lire tous les livres d'une bibliothèque, vous avez un index qui vous mène directement au bon livre.

**Cas d'usage** :
- Moteurs de recherche e-commerce
- Recherche dans des catalogues
- Auto-complétion
- Recherche géospatiale
- **Vector Search pour l'IA** (embeddings, RAG)

#### 3. **RedisTimeSeries** 📊

**Problème résolu** : Stocker et analyser des données temporelles (métriques, capteurs IoT).

**Sans RedisTimeSeries** :
```redis
# Stocker des températures avec Sorted Sets
ZADD sensor:temp 1701234567 "23.5"
ZADD sensor:temp 1701234627 "23.7"
# Calculer une moyenne sur une période = compliqué
```

**Avec RedisTimeSeries** :
```redis
# Créer une série
TS.CREATE sensor:temp

# Ajouter des points
TS.ADD sensor:temp * 23.5
TS.ADD sensor:temp * 23.7

# Récupérer la moyenne des 5 dernières minutes
TS.RANGE sensor:temp - + AGGREGATION avg 300000
```

**Analogie** : C'est comme avoir Excel avec ses graphiques intégrés dans Redis, au lieu de devoir exporter les données pour les analyser.

**Cas d'usage** :
- Monitoring (CPU, mémoire, réseau)
- Capteurs IoT (température, humidité)
- Métriques d'application (latence, throughput)
- Données financières (prix d'actions)

#### 4. **RedisBloom** 🎲

**Problème résolu** : Tester efficacement l'appartenance à un ensemble avec une empreinte mémoire minimale.

**Concept** : Les filtres de Bloom répondent à la question "Cet élément existe-t-il ?" avec :
- ✅ Garantie : Si la réponse est NON, c'est certain
- ⚠️ Probabilité : Si la réponse est OUI, c'est probable (avec un taux d'erreur configurable)

**Cas d'usage** :
```redis
# Vérifier si un email existe déjà (pour éviter les doublons)
BF.ADD emails alice@mail.com
BF.EXISTS emails bob@mail.com  # → 0 (n'existe pas)
BF.EXISTS emails alice@mail.com  # → 1 (existe)
```

**Avantage** : Un filtre de Bloom peut contenir **des millions d'éléments** en utilisant seulement quelques mégaoctets de RAM !

**Analogie** : C'est comme avoir un videur de boîte de nuit qui se souvient de tous les visages qu'il a vus, mais qui prend très peu de place dans sa tête.

**Cas d'usage** :
- Détecter les emails en double
- Cache de requêtes négatives
- Détection de spam
- Déduplication de données

#### 5. **RedisGraph** 📈 (Note : Deprecated depuis 2024)

RedisGraph permettait de créer des bases de données de graphes, mais il a été déprécié en 2024. Nous ne le couvrirons pas dans cette formation.

---

## 3️⃣ Comparaison détaillée : Core vs Stack

### Tableau comparatif complet

| Critère | Redis Core | Redis Stack |
|---------|------------|-------------|
| **Taille binaire** | ~3 MB | ~30 MB |
| **Mémoire de base** | ~1 MB | ~10 MB |
| **Temps de démarrage** | < 1 seconde | ~2-3 secondes |
| **Courbe d'apprentissage** | Simple | Moyenne |
| **Structures de données** | 8 natives | 8 + modules |
| **Recherche full-text** | ❌ | ✅ (RediSearch) |
| **JSON natif** | ❌ | ✅ (RedisJSON) |
| **Séries temporelles** | Limité | ✅ (RedisTimeSeries) |
| **Filtres probabilistes** | HyperLogLog seulement | ✅ (RedisBloom complet) |
| **Vector Search (IA)** | ❌ | ✅ (RediSearch) |
| **Compatibilité** | Universelle | Bonne (clients récents) |
| **Cas d'usage** | Cache, sessions, queues | Tous + recherche, IA, IoT |

### Visualisation de l'écosystème

```
┌────────────────────────────────────────────────┐
│                                                │
│  REDIS CORE (Fondation)                        │
│  ├─ Strings                                    │
│  ├─ Lists                                      │
│  ├─ Sets                                       │
│  ├─ Sorted Sets                                │
│  ├─ Hashes                                     │
│  └─ Streams                                    │
│                                                │
└──────────────┬─────────────────────────────────┘
               │
               ↓
┌────────────────────────────────────────────────┐
│                                                │
│  REDIS STACK (Extension)                       │
│  ├─ Tout Redis Core                            │
│  ├─ + RedisJSON (documents)                    │
│  ├─ + RediSearch (recherche + vecteurs)        │
│  ├─ + RedisTimeSeries (métriques)              │
│  └─ + RedisBloom (filtres)                     │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 4️⃣ Quand choisir Redis Core ?

### ✅ Redis Core est idéal si :

#### 1. **Votre cas d'usage est simple**
- Cache de pages web
- Sessions utilisateur basiques
- Compteurs simples
- Files d'attente FIFO

**Exemple** : Un blog WordPress qui cache ses pages HTML.

#### 2. **Vous avez des contraintes de ressources**
- Environnement avec peu de RAM
- Conteneurs Docker minimaux
- Devices embarqués (IoT edge)

**Exemple** : Un Raspberry Pi qui gère un cache local.

#### 3. **Vous recherchez la compatibilité maximale**
- Support garanti par tous les clients
- Fonctionne sur tous les clouds
- Pas de dépendances

**Exemple** : Une application qui doit tourner chez n'importe quel client.

#### 4. **Vous débutez avec Redis**
- Moins de concepts à apprendre
- Documentation plus simple
- Communauté plus large

**Exemple** : Vous apprenez Redis et voulez maîtriser les bases d'abord.

### 📊 Scénarios typiques pour Redis Core

```
┌─────────────────────────────────────────┐
│  Cas d'usage Redis Core                 │
├─────────────────────────────────────────┤
│  • Site web avec cache de pages         │
│  • API avec rate limiting simple        │
│  • Sessions utilisateur e-commerce      │
│  • Leaderboard de jeu vidéo             │
│  • File d'attente d'emails              │
│  • Compteur de vues temps réel          │
└─────────────────────────────────────────┘
```

---

## 5️⃣ Quand choisir Redis Stack ?

### ✅ Redis Stack est idéal si :

#### 1. **Vous manipulez des documents JSON**
```javascript
// Votre application travaille avec des objets complexes
const user = {
  name: "Alice",
  preferences: {
    theme: "dark",
    language: "fr",
    notifications: true
  },
  tags: ["premium", "verified"]
};
// → RedisJSON est fait pour ça !
```

#### 2. **Vous avez besoin de recherche avancée**
- Recherche full-text dans un catalogue
- Filtres multiples combinés
- Auto-complétion
- Recherche géospatiale

**Exemple** : Un site e-commerce avec "Cherche des smartphones sous 500€ à Paris".

#### 3. **Vous gérez des séries temporelles**
- Monitoring d'infrastructure
- Capteurs IoT
- Métriques applicatives
- Données financières

**Exemple** : Dashboard temps réel affichant la charge CPU de 100 serveurs.

#### 4. **Vous faites de l'IA moderne**
- Vector Search pour RAG (Retrieval-Augmented Generation)
- Recherche sémantique
- Recommandations par similarité

**Exemple** : Chatbot qui recherche dans une base de connaissances avec des embeddings.

### 📊 Scénarios typiques pour Redis Stack

```
┌──────────────────────────────────────────┐
│  Cas d'usage Redis Stack                 │
├──────────────────────────────────────────┤
│  • E-commerce avec recherche complexe    │
│  • Application avec documents JSON       │
│  • Dashboard de monitoring temps réel    │
│  • Chatbot IA avec RAG                   │
│  • Plateforme IoT avec millions capteurs │
│  • Système de recommandations            │
└──────────────────────────────────────────┘
```

---

## 6️⃣ Migration : De Core à Stack

### Est-ce que Redis Stack remplace Redis Core ?

**Non !** Redis Stack **inclut** Redis Core. C'est une extension, pas un remplacement.

```
Redis Core ⊂ Redis Stack

Tout ce qui fonctionne avec Redis Core fonctionne avec Redis Stack
```

### Peut-on migrer facilement ?

**Oui !** La migration est transparente :

1. **Compatibilité totale** : Vos commandes Redis Core continuent de fonctionner
2. **Pas de réécriture** : Votre code existant fonctionne tel quel
3. **Adoption progressive** : Vous pouvez utiliser les nouveaux modules quand vous en avez besoin

**Exemple de migration progressive** :

```
Étape 1 : Vous utilisez Redis Core pour le cache
         ↓
Étape 2 : Vous installez Redis Stack (Redis Core continue de fonctionner)
         ↓
Étape 3 : Vous commencez à utiliser RedisJSON pour les nouvelles features
         ↓
Étape 4 : Vous ajoutez RediSearch pour le moteur de recherche
         ↓
Résultat : Redis Core + nouveaux modules cohabitent parfaitement
```

### Coût de la migration

| Aspect | Effort requis |
|--------|---------------|
| **Installation** | Minimal (changer l'image Docker) |
| **Code existant** | Aucun changement |
| **Apprentissage** | Moyen (nouveaux modules) |
| **Ressources** | +10-20 MB RAM, +30 MB disque |
| **Performance** | Pas d'impact sur Core |

---

## 7️⃣ L'analogie finale : La voiture

Pour bien comprendre la relation Core/Stack, voici une analogie automobile :

### Redis Core = Voiture de base

```
┌─────────────────────────────────┐
│    🚗 VOITURE DE BASE           │
├─────────────────────────────────┤
│  • Moteur                       │
│  • 4 roues                      │
│  • Volant                       │
│  • Freins                       │
│  • Sièges                       │
└─────────────────────────────────┘

✅ Fiable, éprouvée
✅ Consomme peu
✅ Facile à conduire
✅ Répare partout
```

### Redis Stack = Voiture équipée

```
┌─────────────────────────────────┐
│    🚗 VOITURE ÉQUIPÉE           │
├─────────────────────────────────┤
│  Tout de la voiture de base +   │
│                                 │
│  📡 GPS (RediSearch)            │
│  📸 Caméra de recul (RedisJSON) │
│  📊 Ordinateur de bord (TimeSeries)
│  🔊 Système audio (RedisBloom)  │
└─────────────────────────────────┘

✅ Plus de fonctionnalités
✅ Meilleur confort
✅ Cas d'usage avancés
⚠️  Un peu plus complexe
⚠️  Légèrement plus lourd
```

**Point crucial** : La voiture équipée roule exactement comme la voiture de base. Les équipements supplémentaires sont là quand vous en avez besoin, mais n'empêchent pas la conduite normale.

---

## 8️⃣ Recommandations pratiques

### Pour les débutants

**Commencez par Redis Core**, puis évoluez vers Stack quand vous en aurez besoin.

**Parcours d'apprentissage recommandé** :

```
1. Semaine 1-2 : Redis Core
   └─ Maîtriser les structures de base

2. Semaine 3-4 : Patterns Redis Core
   └─ Cache, sessions, queues

3. Semaine 5+ : Redis Stack (si besoin)
   └─ JSON, Search, TimeSeries selon cas d'usage
```

### Pour les projets en production

**Posez-vous ces questions** :

| Question | Core ✅ | Stack ✅ |
|----------|---------|----------|
| Cache simple ? | ✅ | ✅ |
| Sessions utilisateur basiques ? | ✅ | ✅ |
| Documents JSON complexes ? | ❌ | ✅ |
| Recherche full-text ? | ❌ | ✅ |
| Métriques temps réel ? | Limité | ✅ |
| IA / Vector Search ? | ❌ | ✅ |
| Environnement contraint ? | ✅ | ❌ |
| Besoin de compatibilité max ? | ✅ | Bonne |

### Matrice de décision

```
        Complexité du cas d'usage
             │
        High │     ┌───────────────┐
             │     │  Redis Stack  │
             │     │   recommandé  │
             │     └───────────────┘
             │
             │  ┌──────────┐  ┌──────────┐
      Medium │  │  Redis   │  │  Redis   │
             │  │   Core   │  │  Stack   │
             │  │   OK     │  │  mieux   │
             │  └──────────┘  └──────────┘
             │
        Low  │  ┌──────────────────────┐
             │  │     Redis Core       │
             │  │     parfait          │
             │  └──────────────────────┘
             │
             └─────────────────────────────
                Low      Medium      High
                    Ressources disponibles
```

---

## 9️⃣ Points clés à retenir

### ✅ L'essentiel

1. **Redis Stack inclut Redis Core**
   - Stack = Core + modules supplémentaires
   - Pas de remplacement, mais une extension

2. **Redis Core reste pertinent**
   - Parfait pour 80% des cas d'usage
   - Plus léger, plus simple
   - Universellement compatible

3. **Redis Stack ouvre de nouveaux horizons**
   - JSON natif avec RedisJSON
   - Recherche puissante avec RediSearch
   - IoT et monitoring avec RedisTimeSeries
   - IA moderne avec Vector Search

4. **La migration est sans douleur**
   - Compatibilité ascendante totale
   - Adoption progressive possible
   - Coexistence parfaite Core/Stack

5. **Le choix dépend de votre contexte**
   - Simple et léger → Core
   - Avancé et riche → Stack
   - Débutant → Commencer par Core

### 🎯 Règle d'or

> Commencez avec Redis Core pour apprendre les fondamentaux, puis adoptez Redis Stack quand vos besoins le justifient. Vous ne regretterez jamais d'avoir appris Core en premier.

---

## 🔟 Questions fréquentes

### Q1 : Redis Stack est-il gratuit ?
**R :** Oui ! Redis Stack est open source (jusqu'au changement de licence 2024, voir section suivante). Il existe aussi une version cloud payante avec support.

### Q2 : Redis Stack est-il plus lent que Redis Core ?
**R :** Non. Les modules n'impactent pas les performances de Core. Si vous n'utilisez que des commandes Core, vous avez les mêmes performances.

### Q3 : Puis-je utiliser seulement certains modules de Stack ?
**R :** Oui, techniquement possible mais plus complexe. L'approche recommandée est d'installer Redis Stack complet (il reste léger).

### Q4 : Mes clients Redis existants supportent-ils Stack ?
**R :** Les clients récents oui. Pour RedisJSON et RediSearch, vérifiez que votre client supporte les commandes personnalisées. La plupart des clients populaires ont un support dédié.

### Q5 : Y a-t-il une différence de consommation mémoire ?
**R :** Au démarrage, Stack utilise ~10-20 MB de plus. En fonctionnement, ça dépend de ce que vous stockez. Un document JSON peut être plus compact qu'un Hash mal structuré.

### Q6 : Redis Stack remplace-t-il Elasticsearch ?
**R :** Pour certains cas, oui ! RediSearch peut remplacer Elasticsearch pour :
- Recherche en temps réel (< 10ms)
- Petits à moyens volumes (< 100 millions de documents)
- Vector Search pour IA

Pour le Big Data et analyses massives, Elasticsearch reste préférable.

### Q7 : Dois-je réapprendre Redis avec Stack ?
**R :** Non ! Tout ce que vous savez sur Redis Core reste valide. Stack ajoute simplement de nouvelles commandes pour de nouveaux cas d'usage.

### Q8 : Quelle version installer pour apprendre ?
**R :** **Redis Stack**. Même si vous débutez, autant avoir accès à tout. Vous utiliserez Core au début et découvrirez les modules plus tard.

---

## 📚 Récapitulatif visuel

```
┌───────────────────────────────────────────────────┐
│          REDIS CORE vs REDIS STACK                │
├───────────────────────────────────────────────────┤
│                                                   │
│  REDIS CORE                  REDIS STACK          │
│  ├─ Simple                   ├─ Complet           │
│  ├─ Léger (3MB)              ├─ Riche (30MB)      │
│  ├─ 8 structures             ├─ 8 + modules       │
│  ├─ Cache, sessions          ├─ Tout + JSON, IA   │
│  └─ Compatible 100%          └─ Compatible moderne│
│                                                   │
│  QUAND L'UTILISER ?                               │
│                                                   │
│  Core                        Stack                │
│  • Débutant                  • JSON documents     │
│  • Cache simple              • Recherche avancée  │
│  • RAM limitée               • Séries temporelles │
│  • Max compatibilité         • IA / Vector Search │
│                                                   │
│  MIGRATION : Core → Stack = Transparent ! ✅      │
│                                                   │
└───────────────────────────────────────────────────┘
```

---

## 🚀 Prochaine étape

Maintenant que vous comprenez la différence entre Redis Core et Redis Stack, nous allons aborder un sujet crucial qui a secoué l'écosystème en 2024 : **le changement de licence de Redis et la naissance de Valkey**.

C'est un tournant historique qui a des implications importantes pour votre choix de technologie.

**Prochaine section** : [1.3 - Le séisme de 2024 : Changement de licence et le fork Valkey](./03-changement-licence-et-fork-valkey.md)

---

## 📖 Ressources complémentaires

- [Documentation Redis Stack](https://redis.io/docs/stack/)
- [RedisJSON Commands](https://redis.io/commands/?group=json)
- [RediSearch Documentation](https://redis.io/docs/stack/search/)
- [RedisTimeSeries Guide](https://redis.io/docs/stack/timeseries/)
- [RedisBloom Overview](https://redis.io/docs/stack/bloom/)

---


⏭️ [Le séisme de 2024 : Changement de licence et le fork Valkey](/01-ecosysteme-redis-moderne/03-changement-licence-et-fork-valkey.md)
