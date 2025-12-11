🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.7 Client-side routing vs Proxy-based routing

## Introduction

Le routing des requêtes dans Redis Cluster constitue l'un des défis architecturaux majeurs d'un système distribué : comment acheminer efficacement une requête vers le nœud responsable des données concernées ? Contrairement aux bases de données centralisées où toutes les requêtes arrivent sur le même serveur, Redis Cluster nécessite un mécanisme de routage intelligent pour diriger chaque opération vers le nœud qui possède les hash slots correspondants.

Deux approches fondamentalement différentes existent pour résoudre ce problème : le **client-side routing** (routing intelligent côté client) et le **proxy-based routing** (routing via un proxy intermédiaire). Chaque approche présente des avantages et des compromis distincts en termes de latence, complexité, maintenabilité et tolérance aux pannes.

Cette section explore en détail ces deux architectures, leurs implémentations, leurs mécanismes internes, et fournit les critères pour choisir l'approche la plus adaptée à chaque contexte.

## Architecture client-side routing

### Principe fondamental

```
┌─────────────────────────────────────────────────────────────┐
│            Client-Side Routing (Smart Client)               │
└─────────────────────────────────────────────────────────────┘

Le client connaît la topologie complète du cluster et route
directement vers le bon nœud.


    Application
        │
        │ Client Redis Intelligent
        │ ├─ Maintient table de routing (slots → nœuds)
        │ ├─ Calcule le slot de chaque clé
        │ └─ Connecte directement au bon nœud
        │
        ┌─────────┼─────────┐
        │         │         │
        ▼         ▼         ▼
    ┌─────┐   ┌─────┐   ┌─────┐
    │Node │   │Node │   │Node │
    │  A  │   │  B  │   │  C  │
    └─────┘   └─────┘   └─────┘
    Slots     Slots     Slots
    0-5460    5461-     10923-
              10922     16383


Avantages :
═══════════
✓ Latence minimale : 1 hop direct
✓ Pas de SPOF : Pas de proxy centralisé
✓ Scalabilité : Chaque client route indépendamment
✓ Performance maximale


Inconvénients :
═══════════════
✗ Complexité client : Bibliothèque cluster-aware requise
✗ Chaque client maintient état du cluster
✗ Gestion des redirections nécessaire
✗ Overhead mémoire par client
```

### Mécanisme de découverte de topologie

```
┌─────────────────────────────────────────────────────────────┐
│        Découverte et synchronisation de la topologie        │
└─────────────────────────────────────────────────────────────┘

Étape 1 : Initialisation du client
═══════════════════════════════════

Client démarre avec "seed nodes" (liste initiale de nœuds)
    │
    │ CLUSTER SLOTS
    ▼
┌───────────────────────────────────────────────────────────┐
│ Retourne la table complète de mapping slots → nœuds       │
├───────────────────────────────────────────────────────────┤
│ Slot 0-5460      → Node A (192.168.1.10:6379)             │
│ Slot 5461-10922  → Node B (192.168.1.11:6379)             │
│ Slot 10923-16383 → Node C (192.168.1.12:6379)             │
└───────────────────────────────────────────────────────────┘
    │
    ▼
Client construit sa table de routing locale


Étape 2 : Requête normale
══════════════════════════

Application : GET user:1000
    │
    ├─ Client calcule : CRC16("user:1000") & 16383 = 5798
    ├─ Lookup table : Slot 5798 → Node A
    └─ Connexion directe : 192.168.1.10:6379
    │
    ▼
GET user:1000 ─────────────────────> Node A
                                      └─> Retourne valeur


Étape 3 : Gestion des redirections -MOVED
══════════════════════════════════════════

Application : GET user:5000
    │
    ├─ Client calcule : Slot 8754
    ├─ Table locale (obsolète) : Slot 8754 → Node A
    └─> Envoi vers Node A (MAUVAIS nœud)
    │
    ▼
GET user:5000 ─────────────> Node A
                              │
                              └─> -MOVED 8754 192.168.1.11:6379
    │
    ├─ Client met à jour sa table locale
    ├─ Slot 8754 → Node B (mise à jour)
    └─> Retry automatique vers Node B
    │
    ▼
GET user:5000 ─────────────────────> Node B
                                      └─> Retourne valeur ✓


Étape 4 : Gestion des redirections -ASK (resharding)
═════════════════════════════════════════════════════

Slot en cours de migration de Node A vers Node B

Application : GET migrating:key
    │
    └─> Node A (ancien propriétaire)
        │
        ├─ Clé déjà migrée ?
        │  └─> OUI : -ASK 8000 192.168.1.11:6379
        │
        ▼
Client reçoit -ASK
    │
    ├─ ASKING (commande spéciale)
    └─> Node B
        │
        ▼
    ASKING
    GET migrating:key ─────────> Node B
                                  └─> Retourne valeur ✓

Note : -ASK est temporaire, table locale NON mise à jour
```

### Implémentation de la table de routing

```python
# ═══════════════════════════════════════════════════════════
# EXEMPLE D'IMPLÉMENTATION : TABLE DE ROUTING CLIENT-SIDE
# ═══════════════════════════════════════════════════════════

import redis
from rediscluster import RedisCluster
import crc16  # Pour calcul de hash slot

class RedisClusterClient:
    """
    Client Redis Cluster avec routing intelligent
    """

    def __init__(self, startup_nodes):
        """
        startup_nodes : Liste de {'host': ..., 'port': ...}
        """
        self.startup_nodes = startup_nodes
        self.slot_cache = {}  # slot → node mapping
        self.connections = {}  # node → connection pool
        self._refresh_table()

    def _refresh_table(self):
        """
        Récupérer la topologie du cluster via CLUSTER SLOTS
        """
        # Essayer chaque seed node jusqu'à succès
        for node in self.startup_nodes:
            try:
                conn = redis.Redis(
                    host=node['host'],
                    port=node['port'],
                    decode_responses=True
                )

                # CLUSTER SLOTS retourne la topologie complète
                slots_info = conn.execute_command('CLUSTER', 'SLOTS')

                # Parser la réponse
                for slot_range in slots_info:
                    start_slot = slot_range[0]
                    end_slot = slot_range[1]
                    master_info = slot_range[2]  # [host, port, node_id]

                    master_node = {
                        'host': master_info[0],
                        'port': master_info[1],
                        'node_id': master_info[2]
                    }

                    # Remplir le cache pour chaque slot
                    for slot in range(start_slot, end_slot + 1):
                        self.slot_cache[slot] = master_node

                print(f"✓ Topologie chargée : {len(self.slot_cache)} slots")
                return

            except Exception as e:
                print(f"✗ Erreur connexion à {node}: {e}")
                continue

        raise Exception("Impossible de se connecter au cluster")

    def _get_slot(self, key):
        """
        Calculer le hash slot d'une clé
        """
        # Gérer les hash tags {xxx}
        if '{' in key and '}' in key:
            start = key.index('{')
            end = key.index('}')
            if end > start + 1:
                key = key[start + 1:end]

        # CRC16 modulo 16384
        return crc16.crc16xmodem(key.encode()) & 16383

    def _get_connection(self, node):
        """
        Obtenir ou créer une connexion vers un nœud
        """
        node_key = f"{node['host']}:{node['port']}"

        if node_key not in self.connections:
            self.connections[node_key] = redis.Redis(
                host=node['host'],
                port=node['port'],
                decode_responses=True
            )

        return self.connections[node_key]

    def get(self, key):
        """
        GET avec routing automatique
        """
        slot = self._get_slot(key)
        max_redirects = 5  # Protection contre boucles infinies

        for attempt in range(max_redirects):
            # Trouver le nœud responsable
            target_node = self.slot_cache.get(slot)

            if not target_node:
                print("Slot non trouvé, rafraîchissement de la table...")
                self._refresh_table()
                target_node = self.slot_cache[slot]

            try:
                # Exécuter la commande
                conn = self._get_connection(target_node)
                return conn.get(key)

            except redis.ResponseError as e:
                error_msg = str(e)

                # Gérer -MOVED
                if error_msg.startswith('MOVED'):
                    # Format : MOVED 8754 192.168.1.11:6379
                    parts = error_msg.split()
                    new_slot = int(parts[1])
                    new_host, new_port = parts[2].split(':')

                    # Mettre à jour le cache
                    self.slot_cache[new_slot] = {
                        'host': new_host,
                        'port': int(new_port)
                    }

                    print(f"↻ MOVED: Slot {new_slot} → {new_host}:{new_port}")
                    continue  # Retry

                # Gérer -ASK
                elif error_msg.startswith('ASK'):
                    # Format : ASK 8754 192.168.1.11:6379
                    parts = error_msg.split()
                    ask_host, ask_port = parts[2].split(':')

                    # Envoyer ASKING puis retry
                    ask_conn = redis.Redis(
                        host=ask_host,
                        port=int(ask_port),
                        decode_responses=True
                    )
                    ask_conn.execute_command('ASKING')

                    print(f"↻ ASK: Temporairement vers {ask_host}:{ask_port}")
                    return ask_conn.get(key)

                else:
                    raise

        raise Exception(f"Trop de redirections pour clé {key}")

    def set(self, key, value):
        """
        SET avec routing automatique (similaire à GET)
        """
        slot = self._get_slot(key)
        target_node = self.slot_cache.get(slot)

        if not target_node:
            self._refresh_table()
            target_node = self.slot_cache[slot]

        conn = self._get_connection(target_node)
        return conn.set(key, value)


# ═══════════════════════════════════════════════════════════
# UTILISATION
# ═══════════════════════════════════════════════════════════

if __name__ == "__main__":
    # Initialiser le client avec seed nodes
    client = RedisClusterClient(startup_nodes=[
        {'host': '192.168.1.10', 'port': 6379},
        {'host': '192.168.1.11', 'port': 6379},
        {'host': '192.168.1.12', 'port': 6379},
    ])

    # Utilisation transparente
    client.set("user:1000", "Alice")
    value = client.get("user:1000")
    print(f"Value: {value}")  # Alice

    # Le client gère automatiquement :
    # - Calcul du slot
    # - Routing vers le bon nœud
    # - Redirections -MOVED/-ASK
    # - Mise à jour de la table de routing
```

### Bibliothèques client-side populaires

```
┌─────────────────────────────────────────────────────────────┐
│       Bibliothèques Redis Cluster par langage               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PYTHON                                                      │
│ ══════                                                      │
│ • redis-py-cluster                                          │
│   └─> from rediscluster import RedisCluster                 │
│   └─> Routing automatique, pool de connexions               │
│   └─> Gestion complète -MOVED/-ASK                          │
│                                                             │
│ • redis-py (>= 4.2.0)                                       │
│   └─> Support natif du mode cluster                         │
│   └─> from redis.cluster import RedisCluster                │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ NODE.JS                                                     │
│ ═══════                                                     │
│ • ioredis                                                   │
│   └─> const Redis = require('ioredis');                     │
│   └─> new Redis.Cluster([...nodes])                         │
│   └─> Excellente gestion du cluster                         │
│   └─> Reconnexion automatique                               │
│                                                             │
│ • node-redis (>= 4.0)                                       │
│   └─> Support cluster natif                                 │
│   └─> createCluster({ rootNodes: [...] })                   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ JAVA                                                        │
│ ════                                                        │
│ • Jedis                                                     │
│   └─> JedisCluster cluster = new JedisCluster(...)          │
│   └─> Gestion slots, pool de connexions                     │
│   └─> Thread-safe                                           │
│                                                             │
│ • Lettuce                                                   │
│   └─> RedisClusterClient                                    │
│   └─> Reactive et asynchrone                                │
│   └─> Topologie auto-refresh                                │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ GO                                                          │
│ ══                                                          │
│ • go-redis                                                  │
│   └─> redis.NewClusterClient(&redis.ClusterOptions{...})    │
│   └─> Routing automatique                                   │
│   └─> Pool de connexions par nœud                           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PHP                                                         │
│ ═══                                                         │
│ • phpredis                                                  │
│   └─> RedisCluster()                                        │
│   └─> Extension C native                                    │
│                                                             │
│ • Predis                                                    │
│   └─> Pure PHP                                              │
│   └─> Support cluster via option 'cluster'                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Architecture proxy-based routing

### Principe fondamental

```
┌─────────────────────────────────────────────────────────────┐
│              Proxy-Based Routing (Dumb Client)              │
└─────────────────────────────────────────────────────────────┘

Le client se connecte à un proxy qui route vers le bon nœud.


    Application
        │
        │ Client Redis Standard (pas cluster-aware)
        │ └─> Connexion simple comme Redis standalone
        │
        ▼
    ┌────────┐
    │ PROXY  │  ← Point d'entrée unique
    │        │  ├─ Maintient table de routing
    │        │  ├─ Calcule les slots
    │        │  └─> Route vers nœuds backend
    └───┬────┘
        │
        ┌─────────┼─────────┐
        │         │         │
        ▼         ▼         ▼
    ┌─────┐   ┌─────┐   ┌─────┐
    │Node │   │Node │   │Node │
    │  A  │   │  B  │   │  C  │
    └─────┘   └─────┘   └─────┘


Avantages :
═══════════
✓ Simplicité client : Client standard suffit
✓ Centralisation : Configuration en un seul endroit
✓ Compatibilité : Fonctionne avec tous les clients Redis
✓ Abstraction : Changements cluster transparents


Inconvénients :
═══════════════
✗ Latence : +1 hop (client → proxy → nœud)
✗ SPOF : Proxy = point de défaillance unique
✗ Bottleneck : Proxy limite le throughput
✗ Overhead : Ressources supplémentaires nécessaires
```

### Solutions proxy populaires

```
┌─────────────────────────────────────────────────────────────┐
│                Proxies Redis Cluster                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ TWEMPROXY (Nutcracker)                                      │
│ ══════════════════════                                      │
│ Créé par : Twitter                                          │
│ Type : Proxy léger en C                                     │
│ Performance : Haute (10k-100k req/sec par proxy)            │
│                                                             │
│ Caractéristiques :                                          │
│ ├─ Sharding avec consistent hashing                         │
│ ├─ Pipelining et pooling de connexions                      │
│ ├─ Auto-ejection des nœuds défaillants                      │
│ └─ Support multi-clusters                                   │
│                                                             │
│ Limitations :                                               │
│ ├─ Pas de support natif Redis Cluster (seulement sharding)  │
│ ├─ Configuration statique (redémarrage requis)              │
│ └─ Pas de hot-reload de configuration                       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ENVOY PROXY                                                 │
│ ═══════════                                                 │
│ Créé par : Lyft                                             │
│ Type : Proxy L7 moderne                                     │
│ Performance : Haute                                         │
│                                                             │
│ Caractéristiques :                                          │
│ ├─ Support Redis Cluster natif                              │
│ ├─ Configuration dynamique (xDS API)                        │
│ ├─ Health checks avancés                                    │
│ ├─ Observabilité (métriques, tracing)                       │
│ ├─ Circuit breaking                                         │
│ └─ Retry policies                                           │
│                                                             │
│ Avantages :                                                 │
│ ├─ Écosystème service mesh (Istio)                          │
│ ├─ Configuration moderne (YAML)                             │
│ └─ Très extensible                                          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PREDIXY                                                     │
│ ═══════                                                     │
│ Type : Proxy dédié Redis                                    │
│ Performance : Très haute                                    │
│                                                             │
│ Caractéristiques :                                          │
│ ├─ Support Redis Cluster complet                            │
│ ├─ Support Redis Sentinel                                   │
│ ├─ Multi-threading                                          │
│ ├─ Hot-reload de configuration                              │
│ └─ Monitoring intégré                                       │
│                                                             │
│ Avantages :                                                 │
│ ├─ Spécialisé pour Redis                                    │
│ ├─ Performance excellente                                   │
│ └─ Configuration flexible                                   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ HAPROXY                                                     │
│ ═══════                                                     │
│ Type : Load balancer générique                              │
│ Performance : Haute                                         │
│                                                             │
│ Caractéristiques :                                          │
│ ├─ Load balancing avancé                                    │
│ ├─ Health checks                                            │
│ ├─ SSL termination                                          │
│ └─ Très mature et stable                                    │
│                                                             │
│ Limitations pour Redis Cluster :                            │
│ ├─ Pas de routing par hash slot natif                       │
│ ├─ Simple load balancing round-robin                        │
│ └─> Nécessite configuration manuelle des backends           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Configuration Twemproxy

```yaml
# ═══════════════════════════════════════════════════════════
# TWEMPROXY (Nutcracker) - Configuration
# ═══════════════════════════════════════════════════════════

# /etc/nutcracker/nutcracker.yml

# Cluster Redis
redis_cluster:
  # Écoute sur ce port
  listen: 0.0.0.0:6379

  # Hash function pour distribuer les clés
  hash: fnv1a_64

  # Distribution des clés (modulo, ketama=consistent hashing)
  distribution: ketama

  # Timeout en millisecondes
  timeout: 400

  # Nombre de tentatives en cas d'échec
  server_retry_timeout: 2000
  server_failure_limit: 3

  # Auto-eject des serveurs en échec
  auto_eject_hosts: true

  # Pool de connexions
  preconnect: true

  # Mode Redis (vs Memcached)
  redis: true

  # Serveurs backend
  servers:
    - 192.168.1.10:6379:1  # weight=1
    - 192.168.1.11:6379:1
    - 192.168.1.12:6379:1
```

```bash
# Démarrage de Twemproxy
# ──────────────────────

# Installation
sudo apt-get install nutcracker

# Ou depuis les sources
git clone https://github.com/twitter/twemproxy.git
cd twemproxy
autoreconf -fvi
./configure --enable-debug=log
make
sudo make install

# Lancer
nutcracker -c /etc/nutcracker/nutcracker.yml -v 11 -o /var/log/nutcracker.log

# Options :
# -c : Fichier de configuration
# -v : Niveau de verbosité (1-11)
# -o : Fichier de log
# -d : Daemon mode


# Test de connexion
# ─────────────────

# Le client se connecte au proxy comme à un Redis standalone
redis-cli -h localhost -p 6379 SET user:1000 "Alice"
# Le proxy route automatiquement vers le bon backend


# Monitoring
# ──────────

# Stats du proxy
echo "stats" | nc localhost 22222 | python -m json.tool

# Métriques importantes :
# - client_connections : Connexions actives
# - server_ejections : Nœuds éjectés
# - forward_error : Erreurs de forwarding
# - fragments : Requêtes fragmentées (multi-key)
```

### Configuration Envoy pour Redis Cluster

```yaml
# ═══════════════════════════════════════════════════════════
# ENVOY PROXY - Configuration Redis Cluster
# ═══════════════════════════════════════════════════════════

# envoy-redis-cluster.yaml

static_resources:
  listeners:
    - name: redis_listener
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 6379

      filter_chains:
        - filters:
            - name: envoy.filters.network.redis_proxy
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.redis_proxy.v3.RedisProxy

                # Préfixe pour les stats
                stat_prefix: redis_stats

                # Configuration du cluster
                settings:
                  # Mode cluster
                  op_timeout: 5s
                  enable_redirection: true  # Gérer -MOVED/-ASK
                  enable_command_stats: true

                # Pool de connexions
                prefix_routes:
                  catch_all_route:
                    cluster: redis_cluster

                # Pipelining
                downstream_auth_password:
                  inline_string: "password"  # Si authentification requise

  clusters:
    - name: redis_cluster

      # Type de découverte : cluster Redis
      cluster_type:
        name: envoy.clusters.redis
        typed_config:
          "@type": type.googleapis.com/google.protobuf.Empty

      # Découverte des nœuds
      load_assignment:
        cluster_name: redis_cluster
        endpoints:
          - lb_endpoints:
              # Seed nodes
              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.1.10
                      port_value: 6379
              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.1.11
                      port_value: 6379
              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.1.12
                      port_value: 6379

      # Health checks
      health_checks:
        - timeout: 1s
          interval: 5s
          unhealthy_threshold: 3
          healthy_threshold: 2
          custom_health_check:
            name: envoy.health_checkers.redis
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.health_checkers.redis.v3.Redis
              key: health_check_key

      # Timeout
      connect_timeout: 1s

      # DNS
      dns_lookup_family: V4_ONLY

# Configuration admin
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901
```

```bash
# Lancement d'Envoy
# ─────────────────

# Installation
# Voir : https://www.envoyproxy.io/docs/envoy/latest/start/install

# Lancer avec la configuration
envoy -c envoy-redis-cluster.yaml

# Le proxy écoute sur :6379
# Admin interface sur :9901


# Monitoring via admin interface
# ───────────────────────────────

# Stats
curl http://localhost:9901/stats

# Clusters status
curl http://localhost:9901/clusters

# Config dump
curl http://localhost:9901/config_dump


# Test de connexion
# ─────────────────

redis-cli -h localhost -p 6379
127.0.0.1:6379> SET test "value"
OK
# Envoy route automatiquement vers le bon nœud du cluster
```

### Configuration Predixy

```conf
# ═══════════════════════════════════════════════════════════
# PREDIXY - Configuration Redis Cluster
# ═══════════════════════════════════════════════════════════

# /etc/predixy/predixy.conf

# Général
Name PredixyRedisCluster
Bind 0.0.0.0:6379
WorkerThreads 4

# Logging
Log /var/log/predixy/predixy.log
LogRotate 1d
LogLevel Notice

# Authority (optionnel, pour auth)
# Authority {
#     Auth "redis_password" {
#         Mode read
#     }
# }

# Cluster Redis
ClusterServerPool {
    # Mot de passe (si requis)
    # Password redis_password

    # Seed nodes
    MasterReadPriority 60
    StaticSlaveReadPriority 50
    DynamicSlaveReadPriority 50
    RefreshInterval 1
    ServerTimeout 1
    ServerFailureLimit 10
    ServerRetryTimeout 1
    KeepAlive 120

    # Nœuds du cluster
    Servers {
        + 192.168.1.10:6379
        + 192.168.1.11:6379
        + 192.168.1.12:6379
    }
}

# Latency monitoring
LatencyMonitor all {
    # Commandes à monitorer
    Commands {
        + all
    }

    # Seuils en microsecondes
    TimeSpan {
        + 1000   # 1ms
        + 5000   # 5ms
        + 10000  # 10ms
    }
}
```

```bash
# Installation et lancement de Predixy
# ─────────────────────────────────────

# Compilation depuis sources
git clone https://github.com/joyieldInc/predixy.git
cd predixy
make

# Lancer
./src/predixy /etc/predixy/predixy.conf

# Ou en daemon
./src/predixy /etc/predixy/predixy.conf --Daemonize yes


# Monitoring
# ──────────

# Stats via INFO
redis-cli -h localhost -p 6379 INFO

# Métriques Predixy
redis-cli -h localhost -p 6379 INFO SERVER
redis-cli -h localhost -p 6379 INFO STATS
redis-cli -h localhost -p 6379 INFO LATENCY


# Latency histogram
redis-cli -h localhost -p 6379 INFO LATENCY
# Retourne distribution des latences par commande
```

## Comparaison détaillée

### Latence

```
┌─────────────────────────────────────────────────────────────┐
│                 Comparaison de latence                      │
└─────────────────────────────────────────────────────────────┘

CLIENT-SIDE ROUTING
═══════════════════

Client → Node (direct)
│
└─> Latence = Network RTT + Redis Processing
    ≈ 0.3-0.5 ms (même datacenter)


PROXY-BASED ROUTING
═══════════════════

Client → Proxy → Node
│        │
│        └─> +1 hop supplémentaire
│
└─> Latence = 2 × Network RTT + Proxy Processing + Redis Processing
    ≈ 0.8-1.5 ms (même datacenter)


Overhead du proxy :
───────────────────
• Parsing de la commande : ~0.05-0.1 ms
• Calcul du slot : ~0.01 ms
• Forwarding : ~0.1-0.2 ms
• Buffering : ~0.05 ms

Total overhead : ~0.2-0.4 ms


Impact en percentile :
──────────────────────

                  p50      p95      p99
Client-side     0.4 ms   0.8 ms   1.2 ms
Proxy-based     0.9 ms   1.8 ms   3.5 ms

Différence     +125%    +125%    +191%
```

### Throughput et scalabilité

```
┌─────────────────────────────────────────────────────────────┐
│              Throughput : Client vs Proxy                   │
└─────────────────────────────────────────────────────────────┘

CLIENT-SIDE ROUTING
═══════════════════

Chaque client route indépendamment
└─> Throughput total = N_clients × Throughput_par_client

Exemple : 100 clients à 10k req/sec chacun
        = 1,000,000 req/sec total

Scalabilité : LINÉAIRE ✓


PROXY-BASED ROUTING
═══════════════════

Tous les clients passent par le proxy
└─> Throughput total = Capacité_du_proxy

Capacité typique d'un proxy :
├─ Twemproxy : 50k-100k req/sec
├─ Envoy : 100k-200k req/sec
├─ Predixy : 200k-300k req/sec

Scalabilité : LIMITÉE par le proxy ✗

Solution : Déployer plusieurs proxies
          └─> Load balancer devant les proxies
              └─> Complexité accrue


Schéma avec multiple proxies :
══════════════════════════════

                Load Balancer (L4)
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
      Proxy 1      Proxy 2      Proxy 3
         │            │            │
         └────────────┼────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
      Node A       Node B       Node C

Throughput = N_proxies × Capacity_per_proxy
Mais complexité opérationnelle ++
```

### Haute disponibilité

```
┌─────────────────────────────────────────────────────────────┐
│              Points de défaillance (SPOF)                   │
└─────────────────────────────────────────────────────────────┘

CLIENT-SIDE ROUTING
═══════════════════

Pas de SPOF ✓

    Client 1 ─────┐
    Client 2 ─────┼──> Node A, B, C
    Client 3 ─────┘

• Si un client crash → Autres clients non affectés
• Si un nœud Redis crash → Failover automatique
• Chaque client est autonome


PROXY-BASED ROUTING
═══════════════════

Proxy = SPOF ✗

    Client 1 ─────┐
    Client 2 ─────┼──> PROXY ──> Node A, B, C
    Client 3 ─────┘       ↑
                         SPOF

• Si proxy crash → Tous les clients affectés
• Nécessite haute disponibilité du proxy


Solution HA pour proxy :
════════════════════════

Option 1 : Keepalived + VIP
────────────────────────────

    Clients ──> VIP (192.168.1.100)
                 │
                 ├─> Proxy 1 (MASTER) ✓
                 └─> Proxy 2 (BACKUP) ⟳

Keepalived bascule la VIP si Proxy 1 tombe


Option 2 : Load Balancer (HAProxy/Nginx)
─────────────────────────────────────────

             HAProxy (actif-actif)
                 │
         ┌───────┼───────┐
         ▼       ▼       ▼
      Proxy 1  Proxy 2  Proxy 3

Health checks détectent les proxies défaillants


Option 3 : Service Mesh (Kubernetes)
─────────────────────────────────────

Clients ──> Kubernetes Service
               │
               ├─> Proxy Pod 1
               ├─> Proxy Pod 2
               └─> Proxy Pod 3

K8s gère automatiquement la HA
```

### Complexité opérationnelle

```
┌─────────────────────────────────────────────────────────────┐
│            Complexité de gestion et maintenance             │
└─────────────────────────────────────────────────────────────┘

CLIENT-SIDE ROUTING
═══════════════════

Développement :
├─ Bibliothèque cluster-aware requise
├─ Gestion des redirections dans le code
├─ Gestion des retry et timeouts
└─> Complexité : MOYENNE-ÉLEVÉE pour les développeurs

Opérations :
├─ Chaque client maintient sa propre vue du cluster
├─ Pas d'infrastructure supplémentaire
├─ Déploiement application = mise à jour automatique
└─> Complexité : FAIBLE pour les ops


PROXY-BASED ROUTING
═══════════════════

Développement :
├─ Client Redis standard suffit
├─ Pas de logique cluster dans l'application
├─ Code plus simple et portable
└─> Complexité : FAIBLE pour les développeurs

Opérations :
├─ Infrastructure proxy à déployer et maintenir
├─ Monitoring du proxy nécessaire
├─ Configuration proxy à gérer
├─ Mise à jour proxy = opération planifiée
├─ HA du proxy à configurer
└─> Complexité : MOYENNE-ÉLEVÉE pour les ops


Changements de topologie cluster :
═══════════════════════════════════

CLIENT-SIDE :
└─> Clients détectent et s'adaptent automatiquement
    via redirections -MOVED

PROXY-BASED :
└─> Proxy doit être reconfiguré
    ├─ Twemproxy : Redémarrage requis
    ├─ Envoy : Hot reload possible
    └─ Predixy : Découverte automatique ✓
```

### Observabilité et debugging

```
┌─────────────────────────────────────────────────────────────┐
│              Monitoring et troubleshooting                  │
└─────────────────────────────────────────────────────────────┘

CLIENT-SIDE ROUTING
═══════════════════

Métriques distribuées :
├─ Chaque client maintient ses propres métriques
├─ Aggregation nécessaire pour vue globale
└─> Plus difficile à monitorer

Debugging :
├─ Logs dispersés (un par client)
├─ Corrélation complexe
└─> Tracing distribué recommandé (OpenTelemetry)

Avantages :
├─ Pas de point central de collecte
└─ Chaque application peut logger différemment


PROXY-BASED ROUTING
═══════════════════

Métriques centralisées :
├─ Le proxy collecte toutes les métriques
├─ Vue globale immédiate
├─ Histogrammes de latence
├─ Compteurs par commande
└─> Monitoring simplifié ✓

Debugging :
├─ Logs centralisés au niveau du proxy
├─ Traçabilité complète des requêtes
├─ Inspection du trafic possible
└─> Troubleshooting facilité ✓

Exemple métriques Envoy :
─────────────────────────
redis.redis_stats.downstream_cx_active
redis.redis_stats.downstream_rq_total
redis.redis_stats.command.get.latency
redis.redis_stats.command.set.latency
```

## Architectures hybrides

### Proxy pour certains clients, direct pour d'autres

```
┌─────────────────────────────────────────────────────────────┐
│                  Architecture Hybride                       │
└─────────────────────────────────────────────────────────────┘

Cas d'usage : Mix de clients modernes et legacy


    Applications modernes          Applications legacy
    (cluster-aware)                (non cluster-aware)
         │                                │
         │                                ▼
         │                           ┌────────┐
         │                           │ PROXY  │
         │                           └────┬───┘
         │                                │
         └────────────┬───────────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
      Node A       Node B       Node C


Avantages :
═══════════
✓ Migration progressive vers client-side
✓ Support des clients non migrables
✓ Optimisation pour applications critiques
✓ Compatibilité backwards


Configuration :
═══════════════

# Applications modernes se connectent directement
APP_REDIS_NODES=192.168.1.10:6379,192.168.1.11:6379,192.168.1.12:6379

# Applications legacy via proxy
LEGACY_REDIS_HOST=proxy.example.com:6379
```

### Proxy pour isolation et multi-tenancy

```
Cas d'usage : Isolation de tenants avec quotas et ACLs


         Tenant A           Tenant B           Tenant C
            │                  │                  │
            ▼                  ▼                  ▼
        ┌───────┐          ┌───────┐          ┌───────┐
        │Proxy A│          │Proxy B│          │Proxy C│
        │       │          │       │          │       │
        │Rate   │          │Rate   │          │Rate   │
        │Limit  │          │Limit  │          │Limit  │
        └───┬───┘          └───┬───┘          └───┬───┘
            │                  │                  │
            └──────────────────┼──────────────────┘
                               │
                  ┌────────────┼────────────┐
                  ▼            ▼            ▼
               Node A       Node B       Node C
           {tenant:A}:* {tenant:B}:* {tenant:C}:*


Avantages :
═══════════
✓ Rate limiting par tenant
✓ Isolation des connexions
✓ Métriques séparées par tenant
✓ Quotas et throttling
✓ ACLs au niveau proxy


Configuration Envoy pour multi-tenancy :
═════════════════════════════════════════

# Utiliser des filtres Envoy pour rate limiting
rate_limits:
  - stage: 0
    actions:
      - request_headers:
          header_name: "x-tenant-id"
          descriptor_key: "tenant_id"

# Chaque tenant a ses propres limites
```

## Guide de décision

### Arbre de décision

```
Dois-je utiliser Client-Side ou Proxy-Based Routing ?
══════════════════════════════════════════════════════

┌─ Latence ultra-faible critique (< 1ms) ?
│  └─> OUI : CLIENT-SIDE ✓
│  └─> NON : ↓
│
├─ Nombre de clients très élevé (> 1000) ?
│  └─> OUI : CLIENT-SIDE ✓ (éviter bottleneck proxy)
│  └─> NON : ↓
│
├─ Clients legacy non migrables ?
│  └─> OUI : PROXY-BASED ✓
│  └─> NON : ↓
│
├─ Besoin monitoring centralisé fort ?
│  └─> OUI : PROXY-BASED ✓
│  └─> NON : ↓
│
├─ Équipe dev familière avec cluster Redis ?
│  └─> OUI : CLIENT-SIDE ✓
│  └─> NON : PROXY-BASED ✓
│
├─ Infrastructure ops mature (K8s, service mesh) ?
│  └─> OUI : PROXY-BASED ✓ (intégration facile)
│  └─> NON : CLIENT-SIDE ✓
│
├─ Besoin rate limiting / ACL complexes ?
│  └─> OUI : PROXY-BASED ✓
│  └─> NON : CLIENT-SIDE ✓
│
└─ Par défaut : CLIENT-SIDE ✓ (performance maximale)
```

### Matrice de décision

```
┌─────────────────────────────────────────────────────────────┐
│              Matrice de décision détaillée                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Critère              Client-Side    Proxy-Based             │
│ ══════════════════   ════════════   ═══════════             │
│                                                             │
│ Performance          ★★★★★         ★★★☆☆
│ Latence              < 1ms          1-2ms                   │
│ Throughput           Illimité       Limité (proxy)          │
│                                                             │
│ Scalabilité          ★★★★★         ★★★☆☆
│ Horizontal scaling   Excellent      Bon (+ LB)              │
│                                                             │
│ Simplicité Dev       ★★★☆☆         ★★★★★
│ Courbe d'apprentis.  Moyenne        Faible                  │
│ Code application     + Complexe     Simple                  │
│                                                             │
│ Simplicité Ops       ★★★★☆         ★★★☆☆
│ Infrastructure       Minimale       + Proxy layer           │
│ Maintenance          Faible         Moyenne                 │
│                                                             │
│ Observabilité        ★★★☆☆         ★★★★★
│ Monitoring           Distribué      Centralisé              │
│ Debugging            Complexe       Simplifié               │
│                                                             │
│ Haute Disponibilité  ★★★★★         ★★★☆☆
│ SPOF                 Aucun          Proxy (si non HA)       │
│                                                             │
│ Coût                 ★★★★★         ★★★☆☆
│ Infrastructure       Aucun          + Proxies               │
│ Opérationnel         Faible         Moyen                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Recommandations par cas d'usage

```
┌─────────────────────────────────────────────────────────────┐
│           Recommandations par type d'application            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ APIs HAUTE PERFORMANCE (Trading, Gaming, Real-time)         │
│ ═══════════════════════════════════════════════             │
│ Recommandation : CLIENT-SIDE ★★★★★
│ Raison : Latence critique, throughput élevé                 │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ APPLICATIONS WEB STANDARDS                                  │
│ ══════════════════════════                                  │
│ Recommandation : CLIENT-SIDE ★★★★☆
│ Raison : Performance + Simplicité ops                       │
│ Alternative : Proxy si équipe ops préfère                   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ MICROSERVICES (Service Mesh - Istio/Linkerd)                │
│ ═════════════════════════════════════════════               │
│ Recommandation : PROXY-BASED (Envoy) ★★★★★
│ Raison : Intégration native service mesh                    │
│         Observabilité centralisée                           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ APPLICATIONS LEGACY (Non migrables)                         │
│ ═══════════════════════════                                 │
│ Recommandation : PROXY-BASED ★★★★★
│ Raison : Seule option viable                                │
│         Client standard suffit                              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ MULTI-TENANCY STRICT                                        │
│ ════════════════════                                        │
│ Recommandation : PROXY-BASED ★★★★☆
│ Raison : Rate limiting et isolation par tenant              │
│         ACLs granulaires                                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ENVIRONNEMENT KUBERNETES                                    │
│ ════════════════════════                                    │
│ Recommandation : PROXY-BASED ★★★★☆
│ Raison : Pods éphémères, service discovery natif            │
│ Alternative : Client-side si performance critique           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ STARTUP / POC / MVP                                         │
│ ═══════════════════                                         │
│ Recommandation : CLIENT-SIDE ★★★★☆
│ Raison : Simplicité infrastructure                          │
│         Moins de composants à gérer                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Procédures de maintenance

### Migration de proxy vers client-side

```bash
# ═══════════════════════════════════════════════════════════
# PROCÉDURE : MIGRATION PROXY → CLIENT-SIDE
# ═══════════════════════════════════════════════════════════

# Contexte : Application utilise actuellement un proxy Twemproxy
#            Objectif : Migrer vers client-side routing pour performance


# PHASE 1 : Préparation (Semaine -2)
# ───────────────────────────────────

# 1. Auditer les applications
#    - Lister toutes les apps connectées au proxy
#    - Identifier les dépendances
#    - Estimer l'effort de migration

# 2. Choisir la bibliothèque client
#    Python : redis-py-cluster
#    Node.js : ioredis
#    Java : Jedis ou Lettuce

# 3. Environnement de staging
#    - Déployer cluster Redis identique en staging
#    - Tester la bibliothèque client


# PHASE 2 : Implémentation (Semaine -1)
# ──────────────────────────────────────

# 1. Refactoring du code
# Avant (via proxy) :
import redis
client = redis.Redis(host='proxy.example.com', port=6379)

# Après (client-side) :
from rediscluster import RedisCluster
startup_nodes = [
    {"host": "192.168.1.10", "port": 6379},
    {"host": "192.168.1.11", "port": 6379},
    {"host": "192.168.1.12", "port": 6379}
]
client = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)

# 2. Gestion des erreurs
try:
    value = client.get("key")
except redis.exceptions.ClusterError as e:
    logger.error(f"Cluster error: {e}")
    # Fallback ou retry

# 3. Tests intensifs en staging
#    - Tests fonctionnels
#    - Tests de charge
#    - Tests de failover


# PHASE 3 : Déploiement progressif (Semaine 0)
# ─────────────────────────────────────────────

# Stratégie : Blue-Green ou Canary

# Jour 1 : Déployer 5% du trafic
APP_REDIS_MODE=cluster  # Feature flag
APP_REDIS_CLUSTER_NODES=192.168.1.10:6379,192.168.1.11:6379

# Jour 2-3 : Monitoring intensif
#    - Latence p50, p95, p99
#    - Taux d'erreur
#    - Redirections -MOVED

# Jour 4 : Augmenter à 25%
# Jour 7 : Augmenter à 50%
# Jour 10 : Augmenter à 100%


# PHASE 4 : Décommissionnement du proxy
# ──────────────────────────────────────

# Une fois 100% migré et stable (2-4 semaines)

# 1. Vérifier qu'aucun client ne se connecte au proxy
netstat -an | grep :6379 | grep ESTABLISHED | wc -l

# 2. Arrêter le proxy
systemctl stop nutcracker

# 3. Monitoring post-migration (1 semaine)
#    - Vérifier stabilité
#    - Comparer métriques avant/après

# 4. Désinstaller le proxy
apt-get remove nutcracker
```

### Maintenance d'un proxy en production

```bash
# ═══════════════════════════════════════════════════════════
# MAINTENANCE D'UN PROXY REDIS EN PRODUCTION
# ═══════════════════════════════════════════════════════════


# Scenario 1 : Mise à jour du proxy sans downtime
# ────────────────────────────────────────────────

# Prérequis : Avoir au moins 2 proxies derrière un load balancer

# Étape 1 : Retirer Proxy 1 du load balancer
curl -X POST http://lb.example.com/api/backend/disable?server=proxy1

# Étape 2 : Attendre drain des connexions existantes (30-60s)
watch -n 5 'netstat -an | grep :6379 | grep proxy1 | grep ESTABLISHED | wc -l'

# Étape 3 : Arrêter Proxy 1
systemctl stop nutcracker

# Étape 4 : Mise à jour
apt-get update && apt-get install nutcracker

# Étape 5 : Redémarrer
systemctl start nutcracker

# Étape 6 : Tests
redis-cli -h proxy1 PING

# Étape 7 : Réintégrer dans le load balancer
curl -X POST http://lb.example.com/api/backend/enable?server=proxy1

# Étape 8 : Répéter pour Proxy 2, 3, etc.


# Scenario 2 : Ajout de nœuds Redis au cluster (via proxy)
# ──────────────────────────────────────────────────────────

# Le proxy doit être reconfiguré pour connaître les nouveaux nœuds

# Twemproxy : Redémarrage requis
# ─────────────────────────────

# 1. Éditer la configuration
sudo vim /etc/nutcracker/nutcracker.yml

servers:
  - 192.168.1.10:6379:1
  - 192.168.1.11:6379:1
  - 192.168.1.12:6379:1
  - 192.168.1.13:6379:1  # Nouveau nœud

# 2. Valider la configuration
nutcracker -t -c /etc/nutcracker/nutcracker.yml

# 3. Redémarrage avec procédure rolling (voir Scenario 1)


# Predixy : Hot reload
# ────────────────────

# 1. Éditer la configuration
sudo vim /etc/predixy/predixy.conf

Servers {
    + 192.168.1.10:6379
    + 192.168.1.11:6379
    + 192.168.1.12:6379
    + 192.168.1.13:6379  # Nouveau
}

# 2. Reload sans interruption
kill -USR1 $(cat /var/run/predixy.pid)

# Predixy recharge la config sans couper les connexions ✓


# Envoy : Configuration dynamique (xDS)
# ─────────────────────────────────────

# Envoy supporte configuration dynamique via Control Plane
# Pas de redémarrage nécessaire


# Scenario 3 : Monitoring continu du proxy
# ─────────────────────────────────────────

# Script de monitoring
#!/bin/bash

PROXY_HOST="proxy.example.com"
PROXY_STATS_PORT=22222

# Métriques à surveiller
echo "=== Proxy Health Check ==="

# Connexions actives
active_conn=$(echo "stats" | nc $PROXY_HOST $PROXY_STATS_PORT | jq '.client_connections')
echo "Active connections: $active_conn"

# Erreurs de forwarding
forward_errors=$(echo "stats" | nc $PROXY_HOST $PROXY_STATS_PORT | jq '.forward_error')
echo "Forward errors: $forward_errors"

# Backend ejections
ejections=$(echo "stats" | nc $PROXY_HOST $PROXY_STATS_PORT | jq '.server_ejections')
echo "Backend ejections: $ejections"

# Alerter si anomalie
if [ $forward_errors -gt 100 ]; then
    echo "⚠️  High forward errors!"
    # Envoyer alerte
fi

if [ $ejections -gt 0 ]; then
    echo "⚠️  Some backends are ejected!"
    # Envoyer alerte
fi
```

## Conclusion

Le choix entre client-side et proxy-based routing est une décision architecturale majeure qui impacte performance, complexité et maintenabilité du système. Il n'existe pas de solution universellement supérieure : chaque approche présente des avantages et des compromis adaptés à des contextes spécifiques.

**Client-side routing** privilégie :
- Performance maximale (latence sub-milliseconde)
- Scalabilité illimitée
- Absence de SPOF
- Autonomie des clients

**Proxy-based routing** privilégie :
- Simplicité pour les développeurs
- Monitoring centralisé
- Compatibilité avec clients legacy
- Contrôle et gouvernance (rate limiting, ACLs)

La tendance actuelle favorise le **client-side routing** pour les nouvelles applications, grâce aux bibliothèques matures disponibles et aux exigences croissantes de performance. Le **proxy-based routing** reste pertinent dans les contextes legacy, multi-tenancy strict, ou lorsque l'infrastructure existante (service mesh) s'y prête naturellement.

Une approche **hybride** peut également être envisagée, combinant les avantages des deux architectures selon les besoins spécifiques de chaque composant du système.

---

**Points clés à retenir :**

- **Client-side** : Latence minimale, scalabilité maximale, pas de SPOF
- **Proxy-based** : Simplicité dev, monitoring centralisé, compatibilité legacy
- **Redirections** : -MOVED (permanent) vs -ASK (temporaire pendant resharding)
- **Découverte** : CLUSTER SLOTS pour obtenir la topologie
- **HA Proxy** : Keepalived, Load Balancer, ou Kubernetes Service
- **Performance** : Client-side ~2x plus rapide que proxy-based
- **Migration** : Privilégier migration progressive (canary, blue-green)
- **Décision** : Évaluer latence, throughput, ops complexity, observabilité

La prochaine section (11.8) explorera la gestion des Hot Keys et Big Keys dans un cluster distribué.

⏭️ [Gestion des Hot Keys et Big Keys dans un cluster](/11-architecture-distribuee-scaling/08-gestion-hot-keys-big-keys.md)
