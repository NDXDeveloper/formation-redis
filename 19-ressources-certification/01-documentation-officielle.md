🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.1 Documentation officielle Redis et Valkey

## Introduction

La documentation officielle est votre source de vérité pour tout ce qui concerne Redis et Valkey. Depuis le changement de licence de Redis en 2024 et la création du fork Valkey, deux écosystèmes de documentation coexistent, chacun avec ses spécificités.

## 🔴 Documentation Redis

### Site principal
**URL** : https://redis.io/

Le site officiel de Redis Ltd. propose une documentation complète et régulièrement mise à jour.

### Sections clés

#### 1. Redis Documentation
**URL** : https://redis.io/docs/

La documentation principale couvre :
- **Getting Started** : Installation et premiers pas
- **Data Types** : Guide complet des structures de données
- **Commands** : Référence exhaustive de toutes les commandes
- **Clients** : Bibliothèques pour tous les langages
- **Management** : Administration et configuration

#### 2. Redis Stack Documentation
**URL** : https://redis.io/docs/stack/

Documentation des modules étendus :
- **RedisJSON** : Manipulation de documents JSON
- **RediSearch** : Moteur de recherche et indexation
- **RedisGraph** : Base de données graphe
- **RedisTimeSeries** : Données temporelles
- **RedisBloom** : Structures probabilistes

#### 3. Redis Commands Reference
**URL** : https://redis.io/commands/

Référence interactive des commandes avec :
- Description détaillée de chaque commande
- Syntaxe et paramètres
- Exemples d'utilisation
- Complexité algorithmique (Big O)
- Version d'introduction
- Groupe de commandes (String, Hash, List, etc.)

**Exemple de navigation** :
```
https://redis.io/commands/set/     → Commande SET
https://redis.io/commands/hgetall/ → Commande HGETALL
https://redis.io/commands/zadd/    → Commande ZADD
```

#### 4. Redis Architecture
**URL** : https://redis.io/docs/management/

Documentation DevOps/SRE :
- **Replication** : Configuration Master-Replica
- **Sentinel** : Haute disponibilité
- **Cluster** : Scaling horizontal
- **Persistence** : RDB et AOF
- **Security** : ACLs, TLS, authentification

#### 5. Redis Insight
**URL** : https://redis.io/insight/

Interface graphique officielle pour :
- Visualiser vos données
- Exécuter des commandes
- Analyser les performances
- Débugger les problèmes

**Téléchargement** : https://redis.io/insight/#insight-form

### Guides par cas d'usage

| Cas d'usage | URL |
|-------------|-----|
| Caching | https://redis.io/docs/manual/patterns/caching/ |
| Session Store | https://redis.io/docs/manual/patterns/session-store/ |
| Pub/Sub | https://redis.io/docs/manual/pubsub/ |
| Streams | https://redis.io/docs/data-types/streams/ |
| Geospatial | https://redis.io/docs/data-types/geospatial/ |
| Rate Limiting | https://redis.io/glossary/rate-limiting/ |

## 🟢 Documentation Valkey

### Site principal
**URL** : https://valkey.io/

Valkey est le fork open source de Redis maintenu par la Linux Foundation, créé suite au changement de licence de Redis en 2024.

### Sections clés

#### 1. Valkey Documentation
**URL** : https://valkey.io/docs/

Documentation complète du fork incluant :
- **Introduction** : Différences avec Redis
- **Topics** : Guides thématiques
- **Commands** : Référence des commandes (compatible Redis)
- **Clients** : Bibliothèques supportées

#### 2. Valkey Commands Reference
**URL** : https://valkey.io/commands/

Référence des commandes identique à Redis avec :
- 100% de compatibilité avec les commandes Redis Core
- Documentation des nouvelles fonctionnalités spécifiques à Valkey
- Exemples et cas d'usage

#### 3. GitHub Repository
**URL** : https://github.com/valkey-io/valkey

Code source et documentation technique :
- Code source complet
- Issues et discussions
- Contributeurs et gouvernance
- Releases et changelog

### Différences clés Redis vs Valkey

| Aspect | Redis | Valkey |
|--------|-------|--------|
| **Licence** | SSPL/RSALv2 (propriétaire) | BSD 3-Clause (open source) |
| **Gouvernance** | Redis Ltd. | Linux Foundation |
| **Redis Stack** | Modules propriétaires inclus | Non inclus (Core uniquement) |
| **Documentation** | Plus étendue (Stack) | Focalisée sur le Core |
| **Compatibilité** | - | 100% compatible Redis Core |
| **Évolution** | Contrôlée par Redis Ltd. | Communautaire |

## 📚 Autres ressources officielles

### Redis GitHub
**URL** : https://github.com/redis/redis

- Code source de Redis (avant fork)
- Issues historiques
- Pull requests et contributions
- Documentation technique dans `/docs`

### Redis Labs (Redis Ltd.) Blog
**URL** : https://redis.io/blog/

Annonces officielles :
- Nouvelles versions
- Nouvelles fonctionnalités
- Changements de licence
- Études de cas clients

### Redis Weekly Newsletter
**URL** : https://redis.com/redis-weekly/

Newsletter hebdomadaire officielle avec :
- Actualités de l'écosystème
- Tutoriels et articles
- Annonces de releases
- Événements communautaires

## 🔍 Comment naviguer efficacement

### 1. Recherche de commandes

**Méthode 1 : Par catégorie**
```
redis.io/commands/ → Filtrer par groupe (String, Hash, etc.)
```

**Méthode 2 : Recherche directe**
```
redis.io/commands/[nom-commande]/
Exemple : redis.io/commands/set/
```

**Méthode 3 : Depuis redis-cli**
```bash
127.0.0.1:6379> HELP SET
127.0.0.1:6379> HELP @string    # Toutes les commandes String
127.0.0.1:6379> HELP @hash      # Toutes les commandes Hash
```

### 2. Recherche par cas d'usage

1. Identifiez votre besoin (cache, queue, leaderboard, etc.)
2. Consultez : https://redis.io/docs/manual/patterns/
3. Choisissez le pattern correspondant
4. Suivez le guide d'implémentation

### 3. Résolution de problèmes

**Ordre recommandé** :
1. **Commandes** : Vérifiez la syntaxe exacte
2. **Topics** : Consultez les guides thématiques
3. **GitHub Issues** : Cherchez si le problème est connu
4. **Stack Overflow** : Tag `redis` pour questions communautaires

## 📖 Documentation par profil

### Pour les développeurs

**Ressources prioritaires** :
- ✅ Commands Reference : https://redis.io/commands/
- ✅ Data Types Guide : https://redis.io/docs/data-types/
- ✅ Clients Libraries : https://redis.io/docs/clients/
- ✅ Design Patterns : https://redis.io/docs/manual/patterns/

**Focus** : Comprendre les structures de données et les commandes

### Pour les DevOps/SRE

**Ressources prioritaires** :
- ✅ Replication : https://redis.io/docs/management/replication/
- ✅ Sentinel : https://redis.io/docs/management/sentinel/
- ✅ Cluster : https://redis.io/topics/cluster-tutorial/
- ✅ Security : https://redis.io/docs/management/security/
- ✅ Persistence : https://redis.io/docs/management/persistence/

**Focus** : Architecture, haute disponibilité, sécurité

### Pour les architectes

**Ressources prioritaires** :
- ✅ Architecture Overview : https://redis.io/docs/about/
- ✅ Scaling : https://redis.io/docs/management/scaling/
- ✅ Best Practices : https://redis.io/docs/manual/patterns/
- ✅ Performance : https://redis.io/docs/management/optimization/

**Focus** : Design patterns, scalabilité, performance

## 🌐 Documentation multilingue

### Redis
- **Anglais** (officiel) : https://redis.io/docs/
- **Chinois** : https://redis.io/docs/latest/?locale=zh_cn
- **Japonais** : https://redis.io/docs/latest/?locale=ja_jp

### Valkey
- **Anglais** uniquement actuellement : https://valkey.io/docs/

**Note** : La documentation anglaise est toujours la plus à jour et complète.

## 📱 Documentation hors ligne

### Redis Insight
Inclut une documentation intégrée accessible sans connexion Internet.

### Cloner la documentation
```bash
# Redis
git clone https://github.com/redis/redis-doc.git

# Valkey
git clone https://github.com/valkey-io/valkey-doc.git
```

### Formats alternatifs
- **Man pages** : Disponibles dans l'installation Redis
- **PDF** : Certaines sections exportables
- **Markdown** : Source sur GitHub

## 🔄 Rester à jour

### Changelog et Release Notes

**Redis** :
- https://github.com/redis/redis/releases
- https://raw.githubusercontent.com/redis/redis/unstable/00-RELEASENOTES

**Valkey** :
- https://github.com/valkey-io/valkey/releases
- Annonces sur le blog : https://valkey.io/blog/

### S'abonner aux mises à jour

1. **Watch** le repository GitHub
2. **Subscribe** à Redis Weekly
3. **Follow** @Redis et @ValkeyIO sur Twitter/X
4. **Join** les Discord/Slack officiels

## ⚠️ Points d'attention

### Versions de documentation

- Toujours vérifier la **version** de Redis/Valkey concernée
- Les commandes et fonctionnalités varient selon les versions
- Utiliser le sélecteur de version sur redis.io

### Redis Stack vs Redis Core

- **Redis Stack** : Modules étendus (JSON, Search, etc.)
- **Redis Core** : Fonctionnalités de base uniquement
- Valkey = équivalent de Redis Core uniquement

### Documentation obsolète

Attention aux sources non officielles :
- ❌ Blogs personnels non maintenus
- ❌ Tutoriels datés (pre-2020)
- ❌ Documentation de forks non maintenus
- ✅ Privilégiez toujours redis.io et valkey.io

## 🔗 Liens essentiels - Récapitulatif

### Redis

| Ressource | URL |
|-----------|-----|
| Site principal | https://redis.io/ |
| Documentation | https://redis.io/docs/ |
| Commandes | https://redis.io/commands/ |
| Redis Stack | https://redis.io/docs/stack/ |
| Redis Insight | https://redis.io/insight/ |
| GitHub | https://github.com/redis/redis |
| Blog | https://redis.io/blog/ |

### Valkey

| Ressource | URL |
|-----------|-----|
| Site principal | https://valkey.io/ |
| Documentation | https://valkey.io/docs/ |
| Commandes | https://valkey.io/commands/ |
| GitHub | https://github.com/valkey-io/valkey |
| Blog | https://valkey.io/blog/ |

## 💡 Astuces de recherche

### Dans la documentation Redis
```
site:redis.io [votre recherche]
Exemple : site:redis.io sentinel configuration
```

### Dans GitHub
```
repo:redis/redis [votre recherche]
Exemple : repo:redis/redis memory optimization
```

### Stack Overflow
```
[redis] [votre question]
Exemple : [redis] cluster resharding
```

## 📊 Qualité de la documentation

### Redis
- ✅ Très complète et détaillée
- ✅ Exemples de code nombreux
- ✅ Régulièrement mise à jour
- ✅ Interface moderne et ergonomique
- ⚠️ Focus commercial sur Redis Stack

### Valkey
- ✅ Documentation technique solide
- ✅ 100% compatible Redis Core
- ✅ Approche open source transparente
- ⚠️ Moins d'exemples et guides
- ⚠️ Communauté encore en croissance

## 🎯 Prochaines étapes

Après avoir exploré la documentation officielle :
1. **Pratiquez** avec Redis Insight pour visualiser les concepts
2. **Consultez** Redis University pour des cours structurés (section 19.2)
3. **Rejoignez** les communautés pour échanger (section 19.6)
4. **Lisez** les blogs techniques pour approfondir (section 19.7)

---


⏭️ [Redis University et parcours d'apprentissage](/19-ressources-certification/02-redis-university-parcours.md)
