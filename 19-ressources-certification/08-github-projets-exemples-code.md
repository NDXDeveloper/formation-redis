🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.8 GitHub : Projets et exemples de code

## Introduction

GitHub est une mine d'or pour apprendre Redis à travers des exemples de code réels, des projets open source et des implémentations pratiques. Cette section recense les repositories essentiels, des projets d'apprentissage aux applications de production, pour tous les niveaux.

## 🌟 Pourquoi explorer GitHub pour Redis ?

### Avantages de l'apprentissage par le code

- ✅ **Exemples réels** : Code utilisé en production
- ✅ **Best practices** : Patterns éprouvés et documentés
- ✅ **Inspiration** : Découvrir de nouvelles approches
- ✅ **Contribution** : Apprendre en contribuant
- ✅ **Veille technique** : Suivre les évolutions
- ✅ **Portfolio** : Forker et adapter pour vos projets

## 🔴 Repositories officiels

### Redis (Core)

**URL** : https://github.com/redis/redis

**Statistiques** :
- ⭐ 66,000+ stars
- 🍴 25,000+ forks
- 👥 1,000+ contributors

**Structure du repo** :
```
redis/
├── src/           # Code source C
├── tests/         # Suite de tests
├── deps/          # Dépendances
├── utils/         # Utilitaires (redis-cli, etc.)
└── redis.conf     # Configuration par défaut
```

**Points d'intérêt** :
- `/src/` : Code source Redis (C)
- `/tests/` : Tests unitaires et fonctionnels (Tcl)
- `/deps/` : Dépendances externes (jemalloc, hiredis)
- `CONTRIBUTING.md` : Guide de contribution
- `00-RELEASENOTES` : Changelog détaillé

**Pour qui** :
- Contributeurs potentiels
- Comprendre les internals
- Apprendre le C et architecture systèmes

---

### Valkey

**URL** : https://github.com/valkey-io/valkey

**Statistiques** :
- ⭐ 16,000+ stars (croissance rapide)
- 🍴 500+ forks
- 👥 Fork de Redis maintenu par Linux Foundation

**Différences notables** :
- Gouvernance open source transparente
- Process de contribution communautaire
- Focus sur la compatibilité Redis

**Pour qui** :
- Alternative open source pure
- Contribution à un projet Linux Foundation
- Comprendre le fork post-licence change

---

## 📚 Clients officiels

### node-redis (Node.js)

**URL** : https://github.com/redis/node-redis

**Statistiques** : ⭐ 16,000+ stars

**Exemples inclus** :
```javascript
// examples/
├── basic.js              # Connexion basique
├── pub-sub.js           # Pub/Sub
├── pipelining.js        # Pipeline
├── transactions.js      # MULTI/EXEC
├── lua-scripts.js       # Scripting Lua
├── streams.js           # Redis Streams
└── json.js              # RedisJSON
```

**Points forts** :
- ✅ Exemples TypeScript modernes
- ✅ Async/await patterns
- ✅ Tous les cas d'usage couverts

**Lien direct** : https://github.com/redis/node-redis/tree/master/examples

---

### redis-py (Python)

**URL** : https://github.com/redis/redis-py

**Statistiques** : ⭐ 12,000+ stars

**Documentation et exemples** :
- `/docs/examples/` : Exemples complets
- Support asyncio
- Type hints (Python 3.8+)

**Cas d'usage couverts** :
- Connection pooling
- Retry logic
- Cluster support
- Sentinel support
- Streams et Consumer Groups

---

### Jedis (Java)

**URL** : https://github.com/redis/jedis

**Statistiques** : ⭐ 11,000+ stars

**Exemples** :
```java
// examples/
├── BasicExample.java
├── PoolExample.java
├── ClusterExample.java
├── SentinelExample.java
└── PipelineExample.java
```

---

### Lettuce (Java - Réactif)

**URL** : https://github.com/lettuce-io/lettuce-core

**Statistiques** : ⭐ 5,000+ stars

**Points forts** :
- ✅ Client asynchrone et réactif
- ✅ Support Reactive Streams
- ✅ Thread-safe
- ✅ Exemples avec Spring Boot

---

### go-redis (Go)

**URL** : https://github.com/redis/go-redis

**Statistiques** : ⭐ 19,000+ stars

**Exemples** :
```go
// example/
├── basic/
├── cluster/
├── sentinel/
├── pipeline/
└── lua/
```

**Points forts** :
- ✅ API idiomatique Go
- ✅ Context support
- ✅ Excellent documentation

---

### StackExchange.Redis (.NET)

**URL** : https://github.com/StackExchange/StackExchange.Redis

**Statistiques** : ⭐ 5,000+ stars

**Par** : Stack Overflow (utilisé en production)

**Points forts** :
- ✅ High performance
- ✅ Async/await support
- ✅ Utilisé à grande échelle par SO

---

## 🎯 Awesome Lists

### awesome-redis

**URL** : https://github.com/JamzyWang/awesome-redis

**Contenu** :
- 📚 Ressources d'apprentissage
- 🛠️ Outils et utilities
- 📦 Clients dans tous les langages
- 🔌 Modules et extensions
- 📖 Articles et tutoriels
- 🎥 Vidéos et talks

**Sections notables** :
- Redis modules (RediSearch, RedisJSON, etc.)
- GUI clients (Redis Insight, RedisDesktopManager)
- Monitoring tools
- Deployment tools

---

### awesome-scalability

**URL** : https://github.com/binhnguyennus/awesome-scalability

**Section Redis** :
- Architecture patterns avec Redis
- Cas d'usage grande échelle
- Benchmarks et performance

---

## 🚀 Projets d'exemple officiels

### redis-developer (Redis Labs)

**URL** : https://github.com/redis-developer

**Organisation complète avec** :
- Code samples dans tous les langages
- Projets démo complets
- Workshops et tutoriels
- Integrations tierces

**Repositories notables** :
```
redis-developer/
├── basic-redis-examples/
├── redis-microservices-demo/
├── redis-streams-demo/
├── redis-search-demo/
└── redis-vector-search-demo/
```

---

### redis-examples

**URL** : https://github.com/redis/redis-examples

**Exemples officiels** :
```
redis-examples/
├── caching/
│   ├── cache-aside/
│   ├── write-through/
│   └── write-behind/
├── session-store/
├── rate-limiting/
├── leaderboard/
└── pubsub/
```

---

## 💡 Projets par cas d'usage

### Caching

#### django-redis

**URL** : https://github.com/jazzband/django-redis

**Description** : Backend Redis pour Django cache framework

**Statistiques** : ⭐ 2,800+ stars

**Cas d'usage** :
- Cache de requêtes SQL
- Cache de templates
- Session storage
- Cache distribué

---

#### flask-caching

**URL** : https://github.com/pallets-eco/flask-caching

**Description** : Extension caching pour Flask avec support Redis

**Statistiques** : ⭐ 800+ stars

---

### Job Queues

#### Bull (Node.js)

**URL** : https://github.com/OptimalBits/bull

**Description** : Queue de jobs premium basée sur Redis

**Statistiques** : ⭐ 15,000+ stars

**Features** :
- ✅ Job scheduling
- ✅ Job retry avec backoff
- ✅ Priority queues
- ✅ Job events et progress
- ✅ Dashboard web (Bull Board)

**Exemple** :
```javascript
const Queue = require('bull');
const videoQueue = new Queue('video transcoding', 'redis://127.0.0.1:6379');

videoQueue.process(async (job) => {
  // Traitement du job
  return transcodeVideo(job.data);
});
```

---

#### BullMQ (Bull v4)

**URL** : https://github.com/taskforcesh/bullmq

**Description** : Nouvelle version de Bull avec TypeScript

**Statistiques** : ⭐ 5,000+ stars

**Améliorations** :
- ✅ TypeScript natif
- ✅ Meilleure performance
- ✅ Support Streams
- ✅ Workers distribués

---

#### Celery (Python)

**URL** : https://github.com/celery/celery

**Description** : Distributed task queue (support Redis)

**Statistiques** : ⭐ 24,000+ stars

**Utilisation avec Redis** :
```python
# Configuration
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
```

---

#### Sidekiq (Ruby)

**URL** : https://github.com/sidekiq/sidekiq

**Description** : Background jobs pour Ruby

**Statistiques** : ⭐ 13,000+ stars

**Points forts** :
- ✅ Très performant
- ✅ Dashboard web
- ✅ Utilisé par GitHub, Shopify

---

### Session Management

#### express-session with connect-redis

**URL** : https://github.com/tj/connect-redis

**Description** : Redis session store pour Express.js

**Statistiques** : ⭐ 4,000+ stars

**Exemple** :
```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'secret',
  resave: false,
  saveUninitialized: false
}));
```

---

### Rate Limiting

#### redis-rate-limiter

**URL** : https://github.com/TabDigital/redis-rate-limiter

**Description** : Rate limiting flexible avec Redis

**Implémentations** :
- Fixed window
- Sliding window
- Token bucket

---

#### node-rate-limiter-flexible

**URL** : https://github.com/animir/node-rate-limiter-flexible

**Description** : Rate limiter avancé avec Redis

**Statistiques** : ⭐ 2,000+ stars

**Features** :
- ✅ Multiple algorithms
- ✅ Cluster support
- ✅ Protection DDoS
- ✅ Points & duration system

---

### Real-time Applications

#### Socket.io Redis Adapter

**URL** : https://github.com/socketio/socket.io-redis-adapter

**Description** : Adapter Redis pour Socket.io

**Use case** : Scaling WebSocket servers

**Exemple** :
```javascript
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');

const io = new Server();
io.adapter(createAdapter(pubClient, subClient));
```

---

### Pub/Sub et Messaging

#### redis-streams-consumer

**URL** : https://github.com/redis/redis-streams-consumer

**Description** : Consumer Groups simplifié pour Node.js

---

#### ioredis-mock

**URL** : https://github.com/stipsan/ioredis-mock

**Description** : Mock ioredis pour tests

**Statistiques** : ⭐ 300+ stars

**Utilité** :
- ✅ Tests unitaires sans Redis
- ✅ CI/CD simplifié
- ✅ Compatible ioredis API

---

## 🏗️ Architecture et Microservices

### microservices-demo (Weaveworks)

**URL** : https://github.com/microservices-demo/microservices-demo

**Description** : Démo microservices avec Redis

**Tech stack** :
- Docker/Kubernetes
- Redis pour cache et sessions
- Monitoring complet

---

### eShopOnContainers (Microsoft)

**URL** : https://github.com/dotnet-architecture/eShopOnContainers

**Description** : Architecture microservices .NET

**Utilisation Redis** :
- Distributed cache
- Event bus (Pub/Sub)
- Basket service

---

## 🎓 Projets d'apprentissage

### redis-labs (Tutoriels)

**URL** : https://github.com/redis-university

**Repositories** :
- RU101: Introduction samples
- RU102: Language-specific examples
- RU301: Scaling examples
- Labs et workshops

---

### build-your-own-redis

**URL** : https://github.com/codecrafters-io/build-your-own-x

**Description** : Construire un mini-Redis de zéro

**Langages** :
- Python, Go, Rust, JavaScript, etc.

**Apprentissage** :
- Protocol RESP
- Commandes de base
- Persistence basique
- Networking

---

### redis-clone

**URL** : Plusieurs implémentations communautaires

**Exemples** :
- **mini-redis** (Rust) : https://github.com/tokio-rs/mini-redis
- **Simple Redis** (Python) : https://github.com/darioarias/simple-redis
- **Go Redis Server** : https://github.com/tidwall/redcon

**Pourquoi** :
- Comprendre les internals
- Apprendre les protocoles réseau
- Practice systems programming

---

## 🔧 Outils et Utilities

### Redis GUI Clients

#### RedisInsight

**URL** : https://github.com/RedisInsight/RedisInsight

**Description** : GUI officiel Redis

---

#### Another Redis Desktop Manager

**URL** : https://github.com/qishibo/AnotherRedisDesktopManager

**Statistiques** : ⭐ 30,000+ stars

**Points forts** :
- ✅ Open source gratuit
- ✅ Multi-plateforme
- ✅ Support Cluster et Sentinel

---

#### Redis Commander

**URL** : https://github.com/joeferner/redis-commander

**Statistiques** : ⭐ 3,000+ stars

**Description** : Web-based Redis manager

---

### Monitoring et Analytics

#### redis-exporter (Prometheus)

**URL** : https://github.com/oliver006/redis_exporter

**Statistiques** : ⭐ 3,000+ stars

**Usage** :
```bash
docker run -d \
  -p 9121:9121 \
  oliver006/redis_exporter \
  --redis.addr=redis://redis:6379
```

---

#### redis-stat

**URL** : https://github.com/junegunn/redis-stat

**Description** : Monitoring temps réel dans le terminal

---

### Migration et Backup

#### redis-dump

**URL** : https://github.com/delano/redis-dump

**Description** : Dump/restore Redis data en JSON

---

#### riot (Redis Input/Output Tools)

**URL** : https://github.com/redis-developer/riot

**Description** : Import/export massif de données

**Features** :
- ✅ Import CSV, JSON, XML
- ✅ Export vers fichiers
- ✅ Réplication entre instances
- ✅ Migration vers Redis Stack

---

## 🐳 Docker et Kubernetes

### docker-redis

**URL** : https://github.com/docker-library/redis

**Description** : Images Docker officielles Redis

**Tags populaires** :
- `redis:7-alpine` : Version légère
- `redis:7` : Version stable
- `redis:latest` : Dernière version

---

### redis-operator (Kubernetes)

**URL** : https://github.com/spotahome/redis-operator

**Statistiques** : ⭐ 3,500+ stars

**Description** : Opérateur K8s pour Redis HA

**Features** :
- ✅ Sentinel automatique
- ✅ Failover géré
- ✅ Scaling automatique

---

#### redis-cluster-operator

**URL** : https://github.com/ucloud/redis-cluster-operator

**Description** : Opérateur pour Redis Cluster

---

## 🎯 Templates et Starters

### Node.js + Redis Starter

**URL** : https://github.com/hagopj13/node-express-boilerplate

**Stack** :
- Express.js
- Redis (caching)
- JWT auth
- Docker

---

### Django + Redis Template

**URL** : https://github.com/cookiecutter/cookiecutter-django

**Includes** :
- Django
- Redis (cache + Celery)
- Docker Compose
- Production-ready

---

### Spring Boot + Redis

**URL** : https://github.com/spring-projects/spring-data-examples

**Path** : `/redis/`

**Exemples** :
- Spring Data Redis
- Cache abstraction
- Repositories
- Reactive Redis

---

## 📊 Benchmarking et Testing

### redis-benchmark alternatives

#### memtier_benchmark

**URL** : https://github.com/RedisLabs/memtier_benchmark

**Description** : Benchmarking avancé

**Features** :
- ✅ Multiple protocoles
- ✅ Statistiques détaillées
- ✅ Graphs de latence

---

#### redis-rb-cluster

**URL** : https://github.com/redis/redis-rb-cluster

**Description** : Tests de cluster Redis

---

## 🌍 Projets internationaux remarquables

### Système de messagerie temps réel (Chine)

**Exemples de grandes plateformes** :
- Weibo (Twitter chinois)
- Douyin (TikTok)

**Open source inspiré** :
- https://github.com/Terry-Mao/goim (Go IM system)

---

### E-commerce à grande échelle

#### Saleor (Headless e-commerce)

**URL** : https://github.com/saleor/saleor

**Utilisation Redis** :
- Cache produits
- Sessions
- Celery tasks

---

## 💻 Contribution Guidelines

### Comment contribuer

**Repositories accueillants** :
1. **Documentation** : https://github.com/redis/redis-doc
2. **Clients** : node-redis, redis-py, etc.
3. **Modules** : RediSearch, RedisJSON
4. **Tools** : Redis Insight, exporters

**Process typique** :
```bash
# 1. Fork le repository
# 2. Clone votre fork
git clone https://github.com/YOUR_USERNAME/redis.git

# 3. Créer une branche
git checkout -b feature/my-contribution

# 4. Faire vos modifications
# 5. Tests
make test

# 6. Commit et push
git commit -m "Add feature X"
git push origin feature/my-contribution

# 7. Créer une Pull Request
```

**Lire** : `CONTRIBUTING.md` de chaque projet

---

## 🔍 Recherche efficace sur GitHub

### Syntaxe de recherche

**Par langage** :
```
redis language:python
redis language:javascript
```

**Par étoiles** :
```
redis stars:>1000
```

**Par date** :
```
redis pushed:>2024-01-01
```

**Combinaisons** :
```
redis caching language:go stars:>100
```

---

### GitHub Topics

**URL** : https://github.com/topics/redis

**Topics pertinents** :
- `#redis`
- `#redis-client`
- `#redis-cluster`
- `#redis-cache`
- `#redis-streams`
- `#redis-sentinel`

---

## 📚 Collections et Listes

### Explore Collections

**URL** : https://github.com/collections

**Collections incluant Redis** :
- Database tools
- DevOps tools
- Microservices
- Real-time apps

---

## 🎯 Projets par niveau

### Niveau Débutant

**À explorer** :
1. **redis-examples** : Exemples officiels de base
2. **node-redis/examples** : Patterns Node.js
3. **redis-py/examples** : Patterns Python
4. **Build your own Redis** : Comprendre les bases

**Temps** : 1-2 semaines

---

### Niveau Intermédiaire

**À explorer** :
1. **Bull/BullMQ** : Job queues production
2. **Socket.io-redis** : Real-time scaling
3. **connect-redis** : Session management
4. **redis-developer** : Projets complets

**Temps** : 1-2 mois

---

### Niveau Avancé

**À explorer** :
1. **Redis source code** : Comprendre les internals
2. **microservices-demo** : Architecture distribuée
3. **redis-operator** : Kubernetes operators
4. **memtier_benchmark** : Performance testing

**Temps** : 3-6 mois

---

## 🔗 Liens essentiels - Récapitulatif

### Repositories officiels

| Projet | URL | Stars |
|--------|-----|-------|
| Redis Core | https://github.com/redis/redis | 66K+ |
| Valkey | https://github.com/valkey-io/valkey | 16K+ |
| node-redis | https://github.com/redis/node-redis | 16K+ |
| redis-py | https://github.com/redis/redis-py | 12K+ |
| go-redis | https://github.com/redis/go-redis | 19K+ |

### Awesome Lists

| Liste | URL |
|-------|-----|
| awesome-redis | https://github.com/JamzyWang/awesome-redis |
| awesome-scalability | https://github.com/binhnguyennus/awesome-scalability |

### Projets populaires

| Projet | URL | Use case |
|--------|-----|----------|
| Bull | https://github.com/OptimalBits/bull | Job queues |
| Celery | https://github.com/celery/celery | Distributed tasks |
| Another Redis DM | https://github.com/qishibo/AnotherRedisDesktopManager | GUI client |
| redis-exporter | https://github.com/oliver006/redis_exporter | Monitoring |

### Learning

| Ressource | URL |
|-----------|-----|
| redis-developer | https://github.com/redis-developer |
| mini-redis | https://github.com/tokio-rs/mini-redis |
| redis-examples | https://github.com/redis/redis-examples |

---

## ❓ FAQ

**Q : Par où commencer sur GitHub ?**
R : Commencez par redis-examples puis explorez les clients dans votre langage.

**Q : Comment trouver des projets pour contribuer ?**
R : Cherchez le label "good first issue" sur les repos Redis.

**Q : Les exemples sont-ils à jour ?**
R : Vérifiez la date du dernier commit. Privilégiez les repos actifs.

**Q : Puis-je utiliser le code en production ?**
R : Vérifiez la licence. La plupart sont MIT/Apache permissives.

**Q : Comment rester notifié des nouveautés ?**
R : "Watch" les repos importants (Releases only pour limiter le bruit).

**Q : Faut-il étudier le code source Redis ?**
R : Pas nécessaire pour utiliser Redis, mais très formateur pour comprendre les internals.

**Q : Les projets en langages autres que C sont-ils utiles ?**
R : Oui ! Les clients et tools sont essentiels pour l'usage quotidien.

---

## 📝 Checklist exploration GitHub

### Étape 1 : Setup (1h)

- [ ] Créer compte GitHub (si pas déjà fait)
- [ ] Star redis/redis et valkey/valkey
- [ ] Watch releases de Redis et votre client
- [ ] Explorer awesome-redis
- [ ] Bookmark redis-developer

### Étape 2 : Exploration (1 semaine)

- [ ] Cloner redis-examples
- [ ] Tester 5 exemples dans votre langage
- [ ] Lire le code de Bull ou Celery
- [ ] Explorer un GUI client (Another Redis DM)
- [ ] Forker un projet pour expérimentation

### Étape 3 : Pratique (1 mois)

- [ ] Implémenter un pattern (cache, rate limit)
- [ ] Contribuer à la documentation
- [ ] Créer un petit projet Redis
- [ ] Partager sur GitHub
- [ ] Demander review communauté

### Étape 4 : Contribution (ongoing)

- [ ] Identifier un projet pour contribuer
- [ ] Résoudre un "good first issue"
- [ ] Améliorer la documentation
- [ ] Partager vos projets
- [ ] Aider d'autres développeurs

---

## 🚀 Conclusion

GitHub pour Redis c'est :
- ✅ **66K+ repos** liés à Redis
- ✅ **Code production-ready** dans tous les langages
- ✅ **Apprentissage pratique** par l'exemple
- ✅ **Communauté active** pour contribution
- ✅ **Veille technique** avec Watch/Star

**Premier pas** : Explorez redis-developer et clonez redis-examples !

**Liens directs** :
- https://github.com/redis-developer
- https://github.com/redis/redis-examples

---


⏭️ Retour au [Sommaire](/SOMMAIRE.md)
