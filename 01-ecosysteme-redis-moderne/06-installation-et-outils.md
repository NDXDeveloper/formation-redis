🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.6 - Installation et outils (Docker, binaire, Redis Insight, redis-cli)

## 📋 Introduction

Maintenant que vous comprenez **ce qu'est Redis**, son **écosystème moderne**, et son **architecture**, il est temps de mettre les mains dans le code !

Dans cette section, nous allons :
- ✅ Installer Redis (ou Valkey) sur votre machine
- ✅ Découvrir **redis-cli** (l'interface en ligne de commande)
- ✅ Installer **Redis Insight** (l'interface graphique moderne)
- ✅ Faire vos premières commandes
- ✅ Configurer votre environnement

**Bonne nouvelle** : Redis est extrêmement facile à installer et à démarrer !

---

## 🎯 Quelle version installer ?

### Le choix : Redis ou Valkey ?

Rappelez-vous de la section 1.3 :

| Option | Recommandation |
|--------|----------------|
| **Redis** | Si vous voulez Redis Stack (JSON, Search, etc.) |
| **Valkey** | Si vous préférez l'open source pur ou êtes sur AWS |
| **Les deux** | Vous pouvez installer les deux sur des ports différents ! |

**Pour cette formation**, nous utiliserons principalement **Redis** dans les exemples, mais tout fonctionne identiquement avec **Valkey**.

**Note importante** : Les commandes `redis-cli`, `redis-server`, etc. deviennent `valkey-cli`, `valkey-server` avec Valkey, mais le fonctionnement est identique.

---

## 1️⃣ Méthode 1 : Docker (Recommandée pour débuter)

### Pourquoi Docker ?

**Docker** est la méthode la plus simple et la plus propre pour débuter :

- ✅ **Installation en 2 minutes**
- ✅ **Pas de pollution de votre système**
- ✅ **Facile à supprimer**
- ✅ **Identique sur Windows, Mac, Linux**
- ✅ **Version exacte garantie**

**Analogie** : Docker, c'est comme une machine virtuelle ultra-légère. Redis tourne dans son propre petit univers isolé.

### Prérequis : Installer Docker

Si vous n'avez pas Docker :

**Windows/Mac** : [Téléchargez Docker Desktop](https://www.docker.com/products/docker-desktop/)
**Linux** :
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install docker.io

# CentOS/RHEL
sudo yum install docker
```

**Vérification** :
```bash
docker --version
# Devrait afficher : Docker version 24.x.x
```

### Installation de Redis avec Docker

#### Option A : Redis (officiel)

**Commande simple** :
```bash
docker run -d --name redis-test -p 6379:6379 redis:latest
```

**Décryptage de la commande** :
- `docker run` : Démarre un conteneur
- `-d` : Mode détaché (en arrière-plan)
- `--name redis-test` : Nom du conteneur
- `-p 6379:6379` : Expose le port 6379 (port par défaut de Redis)
- `redis:latest` : Image Docker officielle, dernière version

**Redis Stack (avec modules JSON, Search, etc.)** :
```bash
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest
```
Note : Le port 8001 est pour Redis Insight inclus.

#### Option B : Valkey (open source)

```bash
docker run -d --name valkey-test -p 6379:6379 valkey/valkey:latest
```

### Vérifier que ça fonctionne

```bash
# Voir les conteneurs en cours
docker ps

# Devrait afficher quelque chose comme :
# CONTAINER ID   IMAGE          STATUS         PORTS                    NAMES
# abc123def456   redis:latest   Up 2 minutes   0.0.0.0:6379->6379/tcp   redis-test
```

### Se connecter à Redis

**Méthode 1 : Depuis votre machine** (après avoir installé redis-cli, voir plus bas)
```bash
redis-cli
127.0.0.1:6379> PING
PONG
```

**Méthode 2 : Directement dans le conteneur**
```bash
docker exec -it redis-test redis-cli
127.0.0.1:6379> PING
PONG
```

### Commandes Docker utiles

```bash
# Arrêter Redis
docker stop redis-test

# Redémarrer Redis
docker start redis-test

# Voir les logs
docker logs redis-test

# Supprimer le conteneur (après l'avoir arrêté)
docker rm redis-test

# Entrer dans le conteneur (shell)
docker exec -it redis-test /bin/sh
```

### Docker Compose (pour une installation persistante)

Créez un fichier `docker-compose.yml` :

```yaml
version: '3.8'

services:
  redis:
    image: redis:latest
    container_name: redis-dev
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped

volumes:
  redis-data:
```

**Démarrer** :
```bash
docker-compose up -d
```

**Avantages** :
- Configuration versionnée
- Persistance des données (volume)
- Redémarrage automatique
- Facile à partager

---

## 2️⃣ Méthode 2 : Installation native (Binaire)

### Linux (Ubuntu/Debian)

#### Redis

**Via le gestionnaire de paquets** :
```bash
sudo apt update
sudo apt install redis-server

# Vérifier l'installation
redis-server --version
# Redis server v=7.x.x

# Démarrer Redis
sudo systemctl start redis-server

# Activer au démarrage
sudo systemctl enable redis-server

# Vérifier le statut
sudo systemctl status redis-server
```

**Depuis les sources (dernière version)** :
```bash
# Télécharger
wget https://download.redis.io/redis-stable.tar.gz
tar xzf redis-stable.tar.gz
cd redis-stable

# Compiler
make

# Tester (optionnel mais recommandé)
make test

# Installer
sudo make install

# Démarrer
redis-server
```

#### Valkey

```bash
# Télécharger depuis GitHub
wget https://github.com/valkey-io/valkey/archive/refs/tags/7.2.5.tar.gz
tar xzf 7.2.5.tar.gz
cd valkey-7.2.5

# Compiler et installer (comme Redis)
make
sudo make install

# Démarrer
valkey-server
```

### macOS

#### Via Homebrew (Recommandé)

**Redis** :
```bash
# Installer Homebrew si nécessaire
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Installer Redis
brew install redis

# Démarrer Redis
brew services start redis

# Ou manuellement
redis-server /usr/local/etc/redis.conf
```

**Valkey** :
```bash
# Via Homebrew
brew install valkey

# Démarrer
brew services start valkey
```

### Windows

#### Option 1 : WSL2 (Windows Subsystem for Linux)

**Recommandé pour Windows 10/11** :

```bash
# Dans WSL2 (Ubuntu)
sudo apt update
sudo apt install redis-server
redis-server
```

#### Option 2 : Binaires non officiels

Il existe des builds Windows non officiels, mais **Docker est fortement recommandé** sur Windows.

**Téléchargement** : [Redis for Windows (Microsoft Archive)](https://github.com/microsoftarchive/redis/releases)

⚠️ **Attention** : Ces builds sont anciens et non maintenus.

---

## 3️⃣ redis-cli : L'outil en ligne de commande

### Qu'est-ce que redis-cli ?

**redis-cli** (Redis Command Line Interface) est l'outil principal pour interagir avec Redis.

**Analogie** : C'est comme le terminal MySQL ou psql pour PostgreSQL.

### Démarrer redis-cli

**Si Redis tourne en local** :
```bash
redis-cli
```

**Se connecter à un serveur distant** :
```bash
redis-cli -h hostname -p port -a password
# Exemple :
redis-cli -h redis.example.com -p 6379 -a mySecretPassword
```

**Paramètres utiles** :
```bash
redis-cli [options]

Options :
  -h <hostname>    : Hôte (défaut: 127.0.0.1)
  -p <port>        : Port (défaut: 6379)
  -a <password>    : Mot de passe
  -n <db>          : Numéro de base (0-15)
  --raw            : Affichage brut (sans formatage)
  --csv            : Sortie CSV
  --latency        : Mode latence
  --stat           : Statistiques en temps réel
```

### Premières commandes

Une fois dans redis-cli, vous verrez :
```
127.0.0.1:6379>
```

**Commandes de test** :

```redis
# Test de connexion
127.0.0.1:6379> PING
PONG

# Stocker une valeur
127.0.0.1:6379> SET nom "Alice"
OK

# Récupérer une valeur
127.0.0.1:6379> GET nom
"Alice"

# Incrémenter un compteur
127.0.0.1:6379> INCR compteur
(integer) 1

127.0.0.1:6379> INCR compteur
(integer) 2

# Lister toutes les clés (attention en production !)
127.0.0.1:6379> KEYS *
1) "nom"
2) "compteur"

# Supprimer une clé
127.0.0.1:6379> DEL nom
(integer) 1

# Quitter
127.0.0.1:6379> EXIT
```

### Fonctionnalités avancées de redis-cli

#### Auto-complétion

Tapez une commande puis **TAB** :
```redis
127.0.0.1:6379> GE[TAB]
# Propose : GET, GETBIT, GETEX, GETRANGE, etc.
```

#### Aide intégrée

```redis
127.0.0.1:6379> HELP GET

  GET key
  summary: Get the value of a key
  since: 1.0.0
  group: string

127.0.0.1:6379> HELP @string
# Liste toutes les commandes de type String
```

#### Mode interactif vs mode commande

**Mode interactif** (ce qu'on a vu) :
```bash
redis-cli
127.0.0.1:6379> GET mykey
```

**Mode commande directe** :
```bash
# Une commande et sortie immédiate
redis-cli GET mykey

# Utile dans des scripts
redis-cli SET compteur 10
redis-cli INCR compteur
```

#### Mode pipe (batch de commandes)

```bash
# Créer un fichier commands.txt
echo -e "SET key1 value1\nSET key2 value2\nGET key1" > commands.txt

# Exécuter toutes les commandes
cat commands.txt | redis-cli
```

#### Mode monitoring

**Voir toutes les commandes en temps réel** :
```bash
redis-cli MONITOR
# Affiche toutes les commandes exécutées
# Utile pour le debugging, mais JAMAIS en production (très gourmand)
```

#### Statistiques en temps réel

```bash
redis-cli --stat
# Affiche des stats toutes les secondes
------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections
10         1.00M    1       0       100 (+0)            10
10         1.00M    1       0       110 (+10)           10
```

#### Mode latence

```bash
redis-cli --latency
# Mesure la latence réseau
min: 0, max: 1, avg: 0.23 (100 samples)
```

### Commandes d'information

```redis
# Informations générales
127.0.0.1:6379> INFO
# Affiche des tonnes d'informations

# Section spécifique
127.0.0.1:6379> INFO memory
127.0.0.1:6379> INFO stats
127.0.0.1:6379> INFO replication

# Configuration
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "0"

# Nombre de clés par base
127.0.0.1:6379> DBSIZE
(integer) 2

# Nettoyer tout (ATTENTION : DESTRUCTIF !)
127.0.0.1:6379> FLUSHALL
OK
```

---

## 4️⃣ Redis Insight : L'interface graphique moderne

### Qu'est-ce que Redis Insight ?

**Redis Insight** est l'outil graphique officiel de Redis Ltd, gratuit et très puissant.

**Fonctionnalités** :
- 🔍 **Browser** : Naviguer dans vos données visuellement
- 📊 **Profiler** : Analyser les commandes en temps réel
- 📈 **Monitoring** : Voir les métriques et graphiques
- 🧪 **Workbench** : Exécuter des commandes avec auto-complétion
- 📝 **CLI intégré** : redis-cli dans l'interface

**Analogie** : Si redis-cli est le terminal Linux, Redis Insight est l'équivalent d'une interface type phpMyAdmin pour MySQL.

### Installation

#### Windows / macOS / Linux (Desktop)

**Téléchargement** : [Redis Insight](https://redis.io/insight/)

**Installation** :
- Windows : Exécutez le `.exe`
- macOS : Glissez dans Applications
- Linux : `.AppImage` ou `.deb`

#### Docker

```bash
docker run -d --name redis-insight \
  -p 5540:5540 \
  redis/redisinsight:latest
```

Puis ouvrez : http://localhost:5540

#### Via Redis Stack

Si vous avez installé Redis Stack, Redis Insight est déjà inclus sur le port 8001 :

http://localhost:8001

### Configuration initiale

**1. Lancer Redis Insight**

**2. Ajouter une connexion** :
- Cliquez sur "Add Redis Database"
- **Host** : `localhost` (ou `127.0.0.1`)
- **Port** : `6379`
- **Name** : `Redis Local`
- Cliquez "Add Redis Database"

**3. Se connecter** :
- Double-cliquez sur votre connexion

### Utilisation

#### L'interface Browser

```
┌─────────────────────────────────────────────┐
│  Redis Insight - Browser                    │
├─────────────────────────────────────────────┤
│  🔍 Recherche : [________]                  │
├─────────────────────────────────────────────┤
│  📁 Keys (10)                               │
│    ├─ user:1 (hash)                         │
│    ├─ user:2 (hash)                         │
│    ├─ session:abc123 (string)               │
│    ├─ counter (string)                      │
│    └─ leaderboard (sorted set)              │
├─────────────────────────────────────────────┤
│  Détails de la clé sélectionnée             │
│  user:1 (hash)                              │
│  ┌─────────┬──────────┐                     │
│  │ Field   │ Value    │                     │
│  ├─────────┼──────────┤                     │
│  │ name    │ Alice    │                     │
│  │ email   │ a@m.com  │                     │
│  │ age     │ 30       │                     │
│  └─────────┴──────────┘                     │
└─────────────────────────────────────────────┘
```

**Fonctionnalités** :
- Voir toutes vos clés par type
- Filtrer/rechercher
- Voir/éditer les valeurs
- Supprimer des clés
- Ajouter de nouvelles clés visuellement

#### Le Workbench (CLI amélioré)

Interface pour exécuter des commandes avec :
- ✨ Auto-complétion intelligente
- 📝 Historique des commandes
- 🎨 Coloration syntaxique
- 📊 Résultats formatés joliment

```
Workbench
┌────────────────────────────────────────┐
│ > SET user:3 "Bob"                     │
│ OK                                     │
│                                        │
│ > GET user:3                           │
│ "Bob"                                  │
│                                        │
│ > HSET profile:1 name "Alice" age 30   │
│ (integer) 2                            │
│                                        │
│ > HGETALL profile:1                    │
│ 1) "name"                              │
│ 2) "Alice"                             │
│ 3) "age"                               │
│ 4) "30"                                │
└────────────────────────────────────────┘
```

#### Le Profiler

**Voir en temps réel toutes les commandes** exécutées :

```
Profiler (temps réel)
┌──────────┬─────────┬──────────────────────┐
│ Time     │ Client  │ Command              │
├──────────┼─────────┼──────────────────────┤
│ 12:34:56 │ 192...  │ GET user:1           │
│ 12:34:57 │ 192...  │ SET session:abc "x"  │
│ 12:34:58 │ 127...  │ INCR counter         │
│ 12:34:59 │ 127...  │ ZADD board 100 "a"   │
└──────────┴─────────┴──────────────────────┘
```

**Très utile pour** :
- Débugger
- Voir ce que votre application fait réellement
- Identifier des commandes lentes

#### Les Dashboards

**Visualisation des métriques** :
- Utilisation mémoire
- Nombre de clés
- Hit ratio du cache
- Commandes par seconde
- Clients connectés
- Graphiques en temps réel

---

## 5️⃣ Configuration de base

### Le fichier redis.conf

Redis se configure via un fichier `redis.conf`.

**Localisation typique** :
- Linux : `/etc/redis/redis.conf`
- macOS : `/usr/local/etc/redis.conf`
- Docker : Monter le fichier en volume

**Voir la configuration actuelle** :
```redis
127.0.0.1:6379> CONFIG GET *
# Affiche toute la configuration
```

### Paramètres importants pour débuter

#### 1. Bind (Sécurité)

```conf
# Écouter seulement sur localhost (sécurisé)
bind 127.0.0.1

# Écouter sur toutes les interfaces (DANGER en production)
bind 0.0.0.0
```

#### 2. Port

```conf
# Port par défaut
port 6379

# Changer si vous avez plusieurs instances
port 6380
```

#### 3. Mot de passe

```conf
# Définir un mot de passe (IMPORTANT en production)
requirepass VotreMotDePasseFort123!

# Se connecter ensuite :
# redis-cli -a VotreMotDePasseFort123!
```

#### 4. Persistance

```conf
# Snapshots RDB (sauvegarde sur disque)
save 900 1      # Sauvegarder si 1 clé change en 15 min
save 300 10     # Sauvegarder si 10 clés changent en 5 min
save 60 10000   # Sauvegarder si 10k clés changent en 1 min

# AOF (Append Only File - log de toutes les commandes)
appendonly yes
appendfilename "appendonly.aof"
```

#### 5. Mémoire maximale

```conf
# Limiter la mémoire utilisée (important !)
maxmemory 256mb

# Que faire quand la limite est atteinte ?
maxmemory-policy allkeys-lru  # Supprimer les clés les moins utilisées
```

#### 6. Logs

```conf
# Niveau de log
loglevel notice  # debug, verbose, notice, warning

# Fichier de log
logfile /var/log/redis/redis-server.log
```

### Modifier la configuration à chaud

**Sans redémarrer** :
```redis
# Changer un paramètre
127.0.0.1:6379> CONFIG SET maxmemory 512mb
OK

# Sauvegarder dans redis.conf
127.0.0.1:6379> CONFIG REWRITE
OK
```

### Configuration minimale pour développement

Créez un fichier `redis-dev.conf` :

```conf
# Bind local
bind 127.0.0.1

# Port standard
port 6379

# Pas de mot de passe (dev seulement !)
# requirepass ""

# Persistance activée
save 900 1
save 300 10
save 60 10000
appendonly yes

# Mémoire limitée
maxmemory 256mb
maxmemory-policy allkeys-lru

# Logs
loglevel notice
logfile ""  # stdout

# Nom de la base
dbfilename dump.rdb
dir ./data
```

**Démarrer avec cette config** :
```bash
redis-server redis-dev.conf
```

---

## 6️⃣ Vérifier que tout fonctionne

### Checklist de vérification

#### 1. Redis démarre bien

```bash
# Vérifier le processus
ps aux | grep redis
# ou
docker ps  # si Docker

# Vérifier les logs
tail -f /var/log/redis/redis-server.log
# ou
docker logs redis-test
```

#### 2. Le port est ouvert

```bash
# Linux/Mac
netstat -an | grep 6379
# ou
lsof -i :6379

# Devrait afficher :
# tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN
```

#### 3. Connexion possible

```bash
redis-cli PING
# Devrait répondre : PONG
```

#### 4. Commandes de base fonctionnent

```redis
127.0.0.1:6379> SET test "hello"
OK
127.0.0.1:6379> GET test
"hello"
127.0.0.1:6379> DEL test
(integer) 1
```

#### 5. Informations système

```redis
127.0.0.1:6379> INFO server
# Version de Redis
# OS
# Architecture

127.0.0.1:6379> INFO memory
# Utilisation mémoire
# Pic mémoire
```

### Test de performance rapide

```bash
# Benchmark simple
redis-benchmark -q -n 10000

# Résultat typique :
PING_INLINE: 71428.57 requests per second
PING_MBULK: 74626.87 requests per second
SET: 70921.99 requests per second
GET: 72463.77 requests per second
```

Si vous voyez des valeurs comme ça, tout va bien ! ✅

---

## 7️⃣ Environnements multiples

### Plusieurs instances sur une machine

**Pourquoi ?**
- Tester différentes versions
- Isoler les environnements (dev, test, staging)
- Sharding (utiliser plusieurs cœurs CPU)

**Comment ?**

#### Méthode 1 : Docker (le plus simple)

```bash
# Instance 1 (Redis)
docker run -d --name redis-main -p 6379:6379 redis:latest

# Instance 2 (Redis Stack)
docker run -d --name redis-stack -p 6380:6379 redis/redis-stack:latest

# Instance 3 (Valkey)
docker run -d --name valkey-test -p 6381:6379 valkey/valkey:latest
```

**Se connecter** :
```bash
redis-cli -p 6379  # Redis main
redis-cli -p 6380  # Redis Stack
redis-cli -p 6381  # Valkey
```

#### Méthode 2 : Fichiers de config multiples

```bash
# redis-6379.conf
port 6379
pidfile /var/run/redis_6379.pid
logfile /var/log/redis_6379.log
dbfilename dump_6379.rdb

# redis-6380.conf
port 6380
pidfile /var/run/redis_6380.pid
logfile /var/log/redis_6380.log
dbfilename dump_6380.rdb
```

**Démarrer** :
```bash
redis-server /path/to/redis-6379.conf &
redis-server /path/to/redis-6380.conf &
```

### Docker Compose pour un environnement complet

```yaml
version: '3.8'

services:
  redis-main:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - redis-main-data:/data

  redis-stack:
    image: redis/redis-stack:latest
    ports:
      - "6380:6379"
      - "8001:8001"  # Redis Insight
    volumes:
      - redis-stack-data:/data

  valkey:
    image: valkey/valkey:latest
    ports:
      - "6381:6379"
    volumes:
      - valkey-data:/data

volumes:
  redis-main-data:
  redis-stack-data:
  valkey-data:
```

**Démarrer tout** :
```bash
docker-compose up -d
```

**Résultat** : 3 instances Redis/Valkey + Redis Insight en un seul coup !

---

## 8️⃣ Commandes essentielles pour débuter

### Gestion de Redis

```bash
# Démarrer
redis-server [config-file]

# Arrêter proprement
redis-cli SHUTDOWN

# Redémarrer (si service systemd)
sudo systemctl restart redis-server

# Voir les logs
tail -f /var/log/redis/redis-server.log
```

### Backup rapide

```bash
# Sauvegarder maintenant (snapshot RDB)
redis-cli BGSAVE
# Crée dump.rdb en arrière-plan

# Copier le fichier
cp /var/lib/redis/dump.rdb /backup/redis-backup-$(date +%Y%m%d).rdb
```

### Diagnostic

```bash
# Tester la latence
redis-cli --latency

# Voir les commandes lentes
redis-cli SLOWLOG GET 10

# Informations complètes
redis-cli INFO > redis-info.txt
```

---

## 9️⃣ Premiers pas pratiques

### Exercice mental : Votre première application

Imaginez que vous créez un compteur de vues pour un blog :

**1. Stocker le compteur** :
```redis
SET article:123:vues 0
```

**2. Incrémenter à chaque visite** :
```redis
INCR article:123:vues
# (integer) 1
INCR article:123:vues
# (integer) 2
```

**3. Récupérer le nombre de vues** :
```redis
GET article:123:vues
# "2"
```

**4. Stocker les détails de l'article** :
```redis
HSET article:123 titre "Introduction à Redis" auteur "Alice" date "2024-01-15"
```

**5. Récupérer l'article** :
```redis
HGETALL article:123
# 1) "titre"
# 2) "Introduction à Redis"
# 3) "auteur"
# 4) "Alice"
# 5) "date"
# 6) "2024-01-15"
```

**6. Ajouter à un leaderboard** :
```redis
ZADD articles:populaires 2 "article:123"
```

**Avec seulement 6 commandes, vous avez** :
- ✅ Un compteur de vues
- ✅ Un stockage d'article
- ✅ Un classement des articles populaires

**C'est ça, la puissance de Redis !**

---

## 🔟 Résolution de problèmes courants

### Problème 1 : "Connection refused"

**Symptôme** :
```bash
redis-cli
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

**Solutions** :
```bash
# 1. Redis n'est pas démarré
redis-server
# ou
sudo systemctl start redis-server

# 2. Mauvais port
redis-cli -p 6380  # Si Redis écoute sur un autre port

# 3. Redis bind sur une autre adresse
# Vérifier la config :
grep bind /etc/redis/redis.conf
```

### Problème 2 : "NOAUTH Authentication required"

**Symptôme** :
```redis
127.0.0.1:6379> GET test
(error) NOAUTH Authentication required.
```

**Solution** :
```bash
# Se connecter avec le mot de passe
redis-cli -a VotreMotDePasse

# Ou s'authentifier après connexion
127.0.0.1:6379> AUTH VotreMotDePasse
OK
```

### Problème 3 : Port déjà utilisé

**Symptôme** :
```
Address already in use
```

**Solutions** :
```bash
# 1. Trouver qui utilise le port
lsof -i :6379

# 2. Arrêter l'autre instance
redis-cli SHUTDOWN

# 3. Ou utiliser un autre port
redis-server --port 6380
```

### Problème 4 : Mémoire insuffisante

**Symptôme** :
```
OOM command not allowed when used memory > 'maxmemory'
```

**Solutions** :
```redis
# 1. Augmenter la limite
CONFIG SET maxmemory 512mb

# 2. Ou activer l'éviction
CONFIG SET maxmemory-policy allkeys-lru

# 3. Ou nettoyer
FLUSHDB  # Attention, supprime tout !
```

### Problème 5 : Redis ne démarre pas (Linux)

**Solutions** :
```bash
# 1. Vérifier les logs
tail -100 /var/log/redis/redis-server.log

# 2. Permissions
sudo chown redis:redis /var/lib/redis
sudo chown redis:redis /var/log/redis

# 3. THP (Transparent Huge Pages)
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

---

## 📚 Récapitulatif

### Ce que vous avez maintenant

- ✅ **Redis ou Valkey installé** (via Docker ou natif)
- ✅ **redis-cli** pour la ligne de commande
- ✅ **Redis Insight** pour l'interface graphique
- ✅ **Configuration de base** comprise
- ✅ **Premières commandes** exécutées

### Les outils à votre disposition

| Outil | Usage | Quand l'utiliser |
|-------|-------|------------------|
| **redis-cli** | Ligne de commande | Scripts, administration, debug rapide |
| **Redis Insight** | Interface graphique | Exploration, visualisation, développement |
| **MONITOR** | Suivi en temps réel | Debug, comprendre ce qui se passe |
| **INFO** | Métriques système | Monitoring, diagnostic |
| **redis-benchmark** | Tests de performance | Valider l'installation, benchmarks |

### Commandes essentielles à retenir

```redis
# Test de connexion
PING

# Stocker/récupérer
SET key value
GET key

# Supprimer
DEL key

# Incrémenter
INCR counter

# Informations
INFO
CONFIG GET *
DBSIZE

# Lister les clés (DEV uniquement)
KEYS *
# ou mieux :
SCAN 0

# Aide
HELP command
HELP @category
```

---

## 🚀 Prochaines étapes

**Félicitations !** Vous avez terminé le Module 1 : L'écosystème Redis Moderne.

Vous savez maintenant :
- ✅ Ce qu'est Redis et pourquoi il est rapide
- ✅ La différence entre Redis Core et Redis Stack
- ✅ Le contexte du changement de licence et Valkey
- ✅ Comment Redis se compare aux alternatives
- ✅ Pourquoi l'architecture single-thread fonctionne
- ✅ Comment installer et utiliser les outils

**Module 2 : Structures de données natives**

Dans le prochain module, nous allons :
- Explorer les 8+ structures de données de Redis
- Apprendre les commandes pour chaque type
- Comprendre les cas d'usage de chaque structure
- Maîtriser les opérations atomiques

**Préparez-vous** : Gardez Redis Insight et redis-cli ouverts, nous allons beaucoup pratiquer !

---

## 📖 Ressources complémentaires

### Documentation officielle
- [Redis Installation Guide](https://redis.io/docs/install/)
- [Redis CLI Documentation](https://redis.io/docs/ui/cli/)
- [Redis Insight](https://redis.io/insight/)
- [Valkey Getting Started](https://valkey.io/docs/getting-started/)

### Tutoriels interactifs
- [Try Redis](https://try.redis.io/) - Redis dans le navigateur
- [Redis University](https://university.redis.com/) - Cours gratuits

### Configuration
- [Redis Configuration Guide](https://redis.io/docs/management/config/)
- [Redis Security](https://redis.io/docs/management/security/)

### Docker
- [Redis Docker Hub](https://hub.docker.com/_/redis)
- [Redis Stack Docker](https://redis.io/docs/install/install-stack/docker/)
- [Valkey Docker Hub](https://hub.docker.com/r/valkey/valkey)

---

**Fin du Module 1** 🎉

⏭️ [Structures de données natives (Redis Core)](/02-structures-donnees-natives/README.md)
