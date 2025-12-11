🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2 Connexion et pool de connexions

## Introduction

La gestion des connexions Redis est un aspect critique qui impacte directement les **performances**, la **fiabilité** et la **scalabilité** de votre application. Une mauvaise gestion peut entraîner :

- 🐌 Latences élevées (création répétée de connexions)
- 💥 Épuisement des ressources (file descriptors, mémoire)
- 🔌 Connexions perdues non détectées
- 📉 Dégradation progressive des performances

Cette section couvre en profondeur la gestion des connexions et l'utilisation optimale des pools.

---

## Comprendre le coût d'une connexion

### Anatomie d'une connexion Redis

Établir une connexion TCP avec Redis implique plusieurs étapes coûteuses :

```
┌──────────────────────────────────────────────────────────────┐
│                    Établissement connexion                   │
├──────────────────────────────────────────────────────────────┤
│  1. TCP Handshake (SYN, SYN-ACK, ACK)          ~1-3ms        │
│  2. Authentification (AUTH command)             ~1ms         │
│  3. Sélection base de données (SELECT)          ~0.5ms       │
│  4. Initialisation client (info, config)        ~1ms         │
├──────────────────────────────────────────────────────────────┤
│  TOTAL par connexion                            ~3-6ms       │
└──────────────────────────────────────────────────────────────┘
```

**Impact sur une application :**

```python
# ❌ ANTI-PATTERN : Nouvelle connexion par requête
def get_user(user_id):
    client = redis.Redis()  # 5ms de setup
    user = client.get(f'user:{user_id}')  # 1ms
    client.close()  # 1ms
    return user
# Total : ~7ms dont 6ms de overhead (85% de perte !)

# ✅ BONNE PRATIQUE : Réutilisation via pool
pool = redis.ConnectionPool()
client = redis.Redis(connection_pool=pool)

def get_user(user_id):
    return client.get(f'user:{user_id}')  # ~1ms
# Total : ~1ms (7x plus rapide)
```

**Métrique importante :** Sur 10 000 requêtes/sec, l'anti-pattern crée **10 000 connexions/sec** vs **réutilisation de 50 connexions** avec un pool.

---

## Fonctionnement d'un pool de connexions

### Principe de base

Un pool maintient un ensemble de connexions **réutilisables** :

```
Application Threads/Workers
    │      │      │      │
    │      │      │      │
    ▼      ▼      ▼      ▼
┌─────────────────────────────┐
│    Connection Pool          │
│  ┌─────┐ ┌─────┐ ┌─────┐    │
│  │Conn1│ │Conn2│ │Conn3│    │ ◄─── Connexions actives
│  └─────┘ └─────┘ └─────┘    │
│  ┌─────┐ ┌─────┐            │
│  │Conn4│ │Conn5│ [idle]     │ ◄─── Connexions idle
│  └─────┘ └─────┘            │
└─────────────────────────────┘
         │
         ▼
    Redis Server
```

### Cycle de vie d'une connexion

1. **Acquisition** : Thread demande une connexion
2. **Utilisation** : Exécution des commandes Redis
3. **Libération** : Retour au pool (pas de fermeture)
4. **Éviction** : Fermeture si idle trop longtemps ou max atteint

---

## Configuration optimale des pools

### Paramètres clés

| Paramètre | Description | Valeur typique | Impact |
|-----------|-------------|----------------|--------|
| **max_connections** | Nombre maximum de connexions | 50-200 | Limite l'épuisement ressources |
| **min_idle_connections** | Connexions idle minimum | 10-20 | Réduit latence des pics |
| **connection_timeout** | Timeout établissement | 5-10s | Évite blocage infini |
| **socket_timeout** | Timeout lecture/écriture | 3-5s | Détecte connexions mortes |
| **retry_on_timeout** | Retry automatique | true | Améliore résilience |
| **health_check_interval** | Fréquence validation | 30s | Détecte connexions stales |

### Formules de dimensionnement

**Taille pool = (Threads * Connection utilization) + Buffer**

```
Exemples :
- API REST (20 threads, 10% utilization) : 20 * 0.1 + 10 = ~12 connexions
- Worker async (100 coroutines, 50% utilization) : 100 * 0.5 + 20 = ~70 connexions
- Batch processor (1 thread, 100% utilization) : 1 * 1 + 2 = ~3 connexions
```

---

## Implémentations par langage

### Python (redis-py)

#### Configuration de base

```python
import redis
import time
from redis.connection import ConnectionPool

# Configuration production-ready
pool_config = {
    'host': 'localhost',
    'port': 6379,
    'password': None,
    'db': 0,
    'decode_responses': True,

    # Pool settings
    'max_connections': 50,
    'socket_connect_timeout': 5,
    'socket_timeout': 5,
    'socket_keepalive': True,
    'socket_keepalive_options': {
        1: 1,   # TCP_KEEPIDLE
        2: 1,   # TCP_KEEPINTVL
        3: 3    # TCP_KEEPCNT
    },
    'retry_on_timeout': True,
    'health_check_interval': 30,
}

# Création du pool (à faire au démarrage de l'app)
pool = ConnectionPool(**pool_config)

# Client réutilisable
redis_client = redis.Redis(connection_pool=pool)

# Test
try:
    redis_client.ping()
    print("✅ Connexion établie")
except redis.ConnectionError as e:
    print(f"❌ Erreur connexion: {e}")
```

#### Gestion avancée du pool

```python
import redis
from contextlib import contextmanager
import logging

logger = logging.getLogger(__name__)

class RedisConnectionManager:
    """Gestionnaire de pool Redis avec monitoring"""

    def __init__(self, **config):
        self.pool = ConnectionPool(**config)
        self.client = redis.Redis(connection_pool=self.pool)
        logger.info(f"Pool Redis créé (max: {config.get('max_connections', 50)})")

    @contextmanager
    def get_connection(self):
        """Context manager pour acquisition sûre de connexion"""
        conn = None
        try:
            # La connexion est acquise du pool
            conn = self.pool.get_connection('_')
            yield conn
        except redis.ConnectionError as e:
            logger.error(f"Erreur acquisition connexion: {e}")
            raise
        finally:
            # Toujours libérer la connexion
            if conn:
                self.pool.release(conn)

    def get_pool_stats(self):
        """Statistiques du pool"""
        return {
            'created_connections': self.pool._created_connections,
            'available_connections': len(self.pool._available_connections),
            'in_use_connections': len(self.pool._in_use_connections),
        }

    def health_check(self):
        """Vérifie la santé des connexions"""
        try:
            self.client.ping()
            return True
        except Exception as e:
            logger.error(f"Health check failed: {e}")
            return False

    def close(self):
        """Ferme proprement le pool"""
        self.pool.disconnect()
        logger.info("Pool Redis fermé")

# Utilisation
redis_manager = RedisConnectionManager(
    host='localhost',
    port=6379,
    max_connections=50,
    socket_timeout=5
)

# Opération normale
try:
    redis_manager.client.set('key', 'value')
    value = redis_manager.client.get('key')
except redis.RedisError as e:
    logger.error(f"Redis error: {e}")

# Monitoring
stats = redis_manager.get_pool_stats()
print(f"Pool stats: {stats}")

# Fermeture propre (au shutdown de l'app)
redis_manager.close()
```

#### Pattern pour applications web (Flask/FastAPI)

```python
from flask import Flask, g
import redis

app = Flask(__name__)

# Pool global (créé une seule fois)
redis_pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=100,
    decode_responses=True
)

def get_redis():
    """Obtient un client Redis pour la requête courante"""
    if 'redis' not in g:
        g.redis = redis.Redis(connection_pool=redis_pool)
    return g.redis

@app.teardown_appcontext
def teardown_redis(exception):
    """Pas besoin de fermer : le pool gère tout"""
    redis_client = g.pop('redis', None)
    # Rien à faire : la connexion retourne au pool automatiquement

@app.route('/user/<int:user_id>')
def get_user(user_id):
    r = get_redis()
    user = r.get(f'user:{user_id}')
    return {'user': user}

if __name__ == '__main__':
    app.run()
```

---

### Node.js (ioredis)

#### Configuration de base

```javascript
import Redis from 'ioredis';

// Configuration production-ready
const redisConfig = {
    host: 'localhost',
    port: 6379,
    password: null,
    db: 0,

    // Reconnection strategy
    retryStrategy(times) {
        const delay = Math.min(times * 50, 2000);
        return delay;
    },

    // Connection pool settings
    maxRetriesPerRequest: 3,
    enableReadyCheck: true,
    enableOfflineQueue: false, // Échoue rapidement si Redis est down

    // Timeouts
    connectTimeout: 10000,
    commandTimeout: 5000,

    // Keep-alive
    keepAlive: 30000,

    // Events
    lazyConnect: false, // Connecte immédiatement
    showFriendlyErrorStack: process.env.NODE_ENV !== 'production',
};

// Client global (singleton pattern)
const redis = new Redis(redisConfig);

// Event handlers
redis.on('connect', () => {
    console.log('✅ Redis connecté');
});

redis.on('ready', () => {
    console.log('✅ Redis prêt');
});

redis.on('error', (err) => {
    console.error('❌ Redis error:', err);
});

redis.on('close', () => {
    console.log('🔌 Redis connexion fermée');
});

redis.on('reconnecting', (delay) => {
    console.log(`🔄 Reconnexion dans ${delay}ms`);
});

redis.on('end', () => {
    console.log('⛔ Redis connexion terminée définitivement');
});

export default redis;
```

#### Classe wrapper avec monitoring

```javascript
import Redis from 'ioredis';
import EventEmitter from 'events';

class RedisConnectionManager extends EventEmitter {
    constructor(config) {
        super();
        this.config = config;
        this.client = null;
        this.stats = {
            totalCommands: 0,
            errors: 0,
            reconnections: 0,
            lastError: null,
        };
        this.healthCheckInterval = null;
    }

    async connect() {
        this.client = new Redis({
            ...this.config,
            retryStrategy: (times) => {
                this.stats.reconnections++;
                const delay = Math.min(times * 50, 2000);
                this.emit('reconnecting', { times, delay });
                return delay;
            },
        });

        // Event listeners
        this.client.on('connect', () => {
            this.emit('connected');
            console.log('✅ Redis connecté');
        });

        this.client.on('error', (err) => {
            this.stats.errors++;
            this.stats.lastError = err;
            this.emit('error', err);
            console.error('❌ Redis error:', err.message);
        });

        // Health check périodique
        this.startHealthCheck();

        // Test de connexion initiale
        try {
            await this.client.ping();
            console.log('✅ Redis ready');
            return this.client;
        } catch (err) {
            console.error('❌ Connexion initiale échouée:', err);
            throw err;
        }
    }

    startHealthCheck(interval = 30000) {
        this.healthCheckInterval = setInterval(async () => {
            try {
                await this.client.ping();
                this.emit('healthy');
            } catch (err) {
                this.emit('unhealthy', err);
                console.error('❌ Health check failed:', err);
            }
        }, interval);
    }

    stopHealthCheck() {
        if (this.healthCheckInterval) {
            clearInterval(this.healthCheckInterval);
        }
    }

    async execute(command, ...args) {
        try {
            this.stats.totalCommands++;
            return await this.client[command](...args);
        } catch (err) {
            this.stats.errors++;
            this.stats.lastError = err;
            throw err;
        }
    }

    getStats() {
        return {
            ...this.stats,
            connectionStatus: this.client.status,
            uptime: process.uptime(),
        };
    }

    async close() {
        this.stopHealthCheck();
        if (this.client) {
            await this.client.quit();
            console.log('🔌 Redis connexion fermée proprement');
        }
    }
}

// Utilisation
const manager = new RedisConnectionManager({
    host: 'localhost',
    port: 6379,
    maxRetriesPerRequest: 3,
});

// Event handlers personnalisés
manager.on('connected', () => {
    console.log('Manager: Connexion établie');
});

manager.on('error', (err) => {
    console.error('Manager: Erreur détectée:', err.message);
});

manager.on('unhealthy', (err) => {
    console.error('Manager: Health check failed, alerting monitoring system');
    // Envoyer alerte à Datadog/CloudWatch/etc.
});

// Connexion
await manager.connect();

// Utilisation
await manager.execute('set', 'key', 'value');
const value = await manager.execute('get', 'key');

// Stats
console.log('Stats:', manager.getStats());

// Fermeture propre (lors du shutdown)
process.on('SIGTERM', async () => {
    await manager.close();
    process.exit(0);
});

export default manager;
```

#### Pattern pour Express.js

```javascript
import express from 'express';
import Redis from 'ioredis';

const app = express();

// Pool Redis global (singleton)
const redis = new Redis({
    host: 'localhost',
    port: 6379,
    maxRetriesPerRequest: 3,
    enableOfflineQueue: false,
});

// Middleware pour attacher Redis à req
app.use((req, res, next) => {
    req.redis = redis;
    next();
});

// Middleware de health check
app.get('/health', async (req, res) => {
    try {
        await redis.ping();
        res.json({ status: 'healthy', redis: 'connected' });
    } catch (err) {
        res.status(503).json({ status: 'unhealthy', redis: 'disconnected' });
    }
});

// Route exemple
app.get('/user/:id', async (req, res) => {
    try {
        const user = await req.redis.get(`user:${req.params.id}`);
        if (!user) {
            return res.status(404).json({ error: 'User not found' });
        }
        res.json({ user: JSON.parse(user) });
    } catch (err) {
        res.status(500).json({ error: 'Redis error' });
    }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
    console.log('SIGTERM received, closing Redis connection...');
    await redis.quit();
    process.exit(0);
});

app.listen(3000, () => {
    console.log('Server running on port 3000');
});
```

---

### Go (go-redis)

#### Configuration de base

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

var (
    ctx = context.Background()
    rdb *redis.Client
)

// Configuration production-ready
func initRedis() *redis.Client {
    client := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,

        // Pool settings
        PoolSize:     50,              // Nombre max de connexions
        MinIdleConns: 10,              // Connexions idle minimum
        MaxIdleConns: 20,              // Connexions idle maximum

        // Timeouts
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
        PoolTimeout:  4 * time.Second, // Timeout acquisition connexion

        // Retry
        MaxRetries:      3,
        MinRetryBackoff: 8 * time.Millisecond,
        MaxRetryBackoff: 512 * time.Millisecond,

        // Health check
        ConnMaxIdleTime: 5 * time.Minute,
        ConnMaxLifetime: 10 * time.Minute,
    })

    // Test de connexion
    if err := client.Ping(ctx).Err(); err != nil {
        panic(fmt.Sprintf("Impossible de se connecter à Redis: %v", err))
    }

    fmt.Println("✅ Redis connecté")
    return client
}

func main() {
    rdb = initRedis()
    defer rdb.Close()

    // Utilisation
    if err := rdb.Set(ctx, "key", "value", 0).Err(); err != nil {
        panic(err)
    }

    val, err := rdb.Get(ctx, "key").Result()
    if err != nil {
        panic(err)
    }
    fmt.Println("key:", val)
}
```

#### Package avec monitoring et statistiques

```go
package redismanager

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
)

type Manager struct {
    client *redis.Client
    stats  *Stats
    mu     sync.RWMutex
}

type Stats struct {
    TotalCommands   int64
    Errors          int64
    Hits            int64
    Misses          int64
    LastError       error
    LastHealthCheck time.Time
}

type Config struct {
    Addr         string
    Password     string
    DB           int
    PoolSize     int
    MinIdleConns int
}

// NewManager crée un nouveau gestionnaire Redis
func NewManager(cfg Config) (*Manager, error) {
    client := redis.NewClient(&redis.Options{
        Addr:         cfg.Addr,
        Password:     cfg.Password,
        DB:           cfg.DB,
        PoolSize:     cfg.PoolSize,
        MinIdleConns: cfg.MinIdleConns,
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
    })

    // Test connexion
    ctx := context.Background()
    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("connexion failed: %w", err)
    }

    manager := &Manager{
        client: client,
        stats:  &Stats{},
    }

    // Démarrer health check périodique
    go manager.startHealthCheck(30 * time.Second)

    log.Println("✅ Redis Manager initialized")
    return manager, nil
}

// Get récupère une valeur avec statistiques
func (m *Manager) Get(ctx context.Context, key string) (string, error) {
    m.mu.Lock()
    m.stats.TotalCommands++
    m.mu.Unlock()

    val, err := m.client.Get(ctx, key).Result()

    m.mu.Lock()
    defer m.mu.Unlock()

    if err == redis.Nil {
        m.stats.Misses++
        return "", nil // Clé n'existe pas
    } else if err != nil {
        m.stats.Errors++
        m.stats.LastError = err
        return "", err
    }

    m.stats.Hits++
    return val, nil
}

// Set stocke une valeur
func (m *Manager) Set(ctx context.Context, key string, value interface{}, expiration time.Duration) error {
    m.mu.Lock()
    m.stats.TotalCommands++
    m.mu.Unlock()

    err := m.client.Set(ctx, key, value, expiration).Err()

    if err != nil {
        m.mu.Lock()
        m.stats.Errors++
        m.stats.LastError = err
        m.mu.Unlock()
    }

    return err
}

// GetPoolStats retourne les statistiques du pool
func (m *Manager) GetPoolStats() *redis.PoolStats {
    return m.client.PoolStats()
}

// GetStats retourne les statistiques du manager
func (m *Manager) GetStats() Stats {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return *m.stats
}

// startHealthCheck vérifie périodiquement la santé
func (m *Manager) startHealthCheck(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for range ticker.C {
        ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
        err := m.client.Ping(ctx).Err()
        cancel()

        m.mu.Lock()
        m.stats.LastHealthCheck = time.Now()
        if err != nil {
            m.stats.Errors++
            m.stats.LastError = err
            log.Printf("❌ Health check failed: %v", err)
        }
        m.mu.Unlock()
    }
}

// Close ferme proprement les connexions
func (m *Manager) Close() error {
    log.Println("Closing Redis Manager...")
    return m.client.Close()
}

// PrintStats affiche les statistiques
func (m *Manager) PrintStats() {
    stats := m.GetStats()
    poolStats := m.GetPoolStats()

    fmt.Println("=== Redis Manager Stats ===")
    fmt.Printf("Total Commands: %d\n", stats.TotalCommands)
    fmt.Printf("Errors: %d\n", stats.Errors)
    fmt.Printf("Cache Hits: %d\n", stats.Hits)
    fmt.Printf("Cache Misses: %d\n", stats.Misses)
    if stats.Hits+stats.Misses > 0 {
        hitRatio := float64(stats.Hits) / float64(stats.Hits+stats.Misses) * 100
        fmt.Printf("Hit Ratio: %.2f%%\n", hitRatio)
    }
    fmt.Printf("Last Health Check: %v\n", stats.LastHealthCheck)

    fmt.Println("\n=== Pool Stats ===")
    fmt.Printf("Total Connections: %d\n", poolStats.TotalConns)
    fmt.Printf("Idle Connections: %d\n", poolStats.IdleConns)
    fmt.Printf("Stale Connections: %d\n", poolStats.StaleConns)
}
```

#### Utilisation du manager

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "yourapp/redismanager"
)

func main() {
    // Configuration
    cfg := redismanager.Config{
        Addr:         "localhost:6379",
        Password:     "",
        DB:           0,
        PoolSize:     50,
        MinIdleConns: 10,
    }

    // Initialisation
    manager, err := redismanager.NewManager(cfg)
    if err != nil {
        log.Fatalf("Failed to initialize Redis: %v", err)
    }
    defer manager.Close()

    ctx := context.Background()

    // Utilisation normale
    if err := manager.Set(ctx, "user:123", "Alice", time.Hour); err != nil {
        log.Printf("Error setting key: %v", err)
    }

    value, err := manager.Get(ctx, "user:123")
    if err != nil {
        log.Printf("Error getting key: %v", err)
    } else {
        log.Printf("user:123 = %s", value)
    }

    // Simulation de charge
    for i := 0; i < 1000; i++ {
        key := fmt.Sprintf("key:%d", i)
        manager.Set(ctx, key, fmt.Sprintf("value:%d", i), 5*time.Minute)
        manager.Get(ctx, key)
    }

    // Affichage des stats
    manager.PrintStats()

    // Graceful shutdown
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    <-sigChan

    log.Println("Shutting down...")
    manager.PrintStats()
}
```

---

## Monitoring du pool

### Métriques clés à surveiller

| Métrique | Description | Seuil d'alerte |
|----------|-------------|----------------|
| **Active connections** | Connexions en cours d'utilisation | > 80% du max |
| **Idle connections** | Connexions disponibles | < 20% du min |
| **Wait time** | Temps d'attente pour acquérir | > 100ms |
| **Connection errors** | Échecs de connexion | > 1% |
| **Timeouts** | Timeouts d'opération | > 0.5% |

### Exemple de monitoring avec Python

```python
import redis
import time
from prometheus_client import Gauge, Counter

# Métriques Prometheus
pool_active = Gauge('redis_pool_active_connections', 'Active connections')
pool_idle = Gauge('redis_pool_idle_connections', 'Idle connections')
pool_created = Counter('redis_pool_created_connections', 'Total connections created')
pool_errors = Counter('redis_pool_errors', 'Connection errors')

class MonitoredRedisPool:
    def __init__(self, **config):
        self.pool = redis.ConnectionPool(**config)
        self.client = redis.Redis(connection_pool=self.pool)

    def get_metrics(self):
        """Collecte et exporte les métriques"""
        stats = self.client.connection_pool

        active = len(stats._in_use_connections)
        idle = len(stats._available_connections)

        pool_active.set(active)
        pool_idle.set(idle)
        pool_created.inc(stats._created_connections)

        return {
            'active': active,
            'idle': idle,
            'created': stats._created_connections,
        }

    def monitor(self, interval=10):
        """Monitore en continu le pool"""
        while True:
            metrics = self.get_metrics()
            print(f"Pool metrics: {metrics}")
            time.sleep(interval)

# Utilisation
pool = MonitoredRedisPool(
    host='localhost',
    max_connections=50,
    socket_timeout=5
)

# Démarrer monitoring dans un thread séparé
import threading
monitor_thread = threading.Thread(target=pool.monitor, daemon=True)
monitor_thread.start()
```

---

## Problèmes courants et solutions

### 1. Pool exhaustion (épuisement du pool)

**Symptômes :**
```
redis.exceptions.ConnectionError: Error while reading from socket: (104, 'Connection reset by peer')
TimeoutError: Timeout waiting for connection from pool
```

**Causes :**
- Pool trop petit pour la charge
- Connexions non libérées (leaks)
- Opérations bloquantes

**Solutions :**

```python
# ✅ Solution 1 : Augmenter la taille du pool
pool = redis.ConnectionPool(max_connections=200)  # Au lieu de 50

# ✅ Solution 2 : Identifier les fuites avec monitoring
def find_leaks():
    stats = pool._in_use_connections
    if len(stats) > 40:  # 80% du pool
        print(f"⚠️ Possible leak: {len(stats)} connexions en use")
        # Logger les tracebacks pour debug

# ✅ Solution 3 : Utiliser context managers
from contextlib import closing
with closing(pool.get_connection('_')) as conn:
    # Utilisation
    pass
# Connexion automatiquement libérée
```

### 2. Connexions stale (obsolètes)

**Symptômes :**
```
redis.exceptions.ConnectionError: Error while reading from socket
```

**Causes :**
- Connexions idle trop longtemps
- Firewall/Load balancer coupe les connexions
- Redis restart

**Solutions :**

```javascript
// ✅ Activer health checks
const redis = new Redis({
    host: 'localhost',
    port: 6379,
    // Health check avant chaque commande
    enableReadyCheck: true,
    // Ping périodique des connexions idle
    keepAlive: 30000, // 30 secondes
});
```

```go
// ✅ Configurer max lifetime
client := redis.NewClient(&redis.Options{
    ConnMaxIdleTime: 5 * time.Minute,  // Ferme si idle > 5min
    ConnMaxLifetime: 30 * time.Minute, // Renouvelle après 30min
})
```

### 3. Thundering herd (ruée simultanée)

**Symptômes :**
- Pics soudains de connexions
- Pool exhaustion temporaire

**Cause :**
- Expiration simultanée de clés cachées
- Redémarrage d'application

**Solution :**

```python
import random

# ✅ Jitter sur les TTL
base_ttl = 3600
jitter = random.randint(-300, 300)  # ±5 minutes
redis.setex('key', base_ttl + jitter, value)

# ✅ Warm-up progressif au démarrage
def warmup_pool(pool, target_size=20):
    """Pré-crée des connexions au démarrage"""
    for _ in range(target_size):
        conn = pool.get_connection('_')
        pool.release(conn)
```

---

## Bonnes pratiques de production

### ✅ 1. Singleton pattern pour le pool

```javascript
// ❌ MAUVAIS : Nouveau pool par module
export function getRedis() {
    return new Redis(); // Crée un nouveau pool !
}

// ✅ BON : Pool global singleton
const redisClient = new Redis();
export default redisClient;
```

### ✅ 2. Graceful shutdown

```python
# ✅ Fermeture propre au shutdown
import atexit
import signal

def cleanup():
    print("Closing Redis pool...")
    pool.disconnect()

atexit.register(cleanup)
signal.signal(signal.SIGTERM, lambda s, f: cleanup())
```

### ✅ 3. Timeouts adaptés au use case

```go
// ✅ Timeouts courts pour API temps réel
apiClient := redis.NewClient(&redis.Options{
    ReadTimeout:  500 * time.Millisecond,
    WriteTimeout: 500 * time.Millisecond,
})

// ✅ Timeouts longs pour batch processing
batchClient := redis.NewClient(&redis.Options{
    ReadTimeout:  30 * time.Second,
    WriteTimeout: 30 * time.Second,
})
```

### ✅ 4. Séparation des pools par usage

```python
# ✅ Pool séparé pour cache vs sessions vs queues
cache_pool = redis.ConnectionPool(max_connections=100, db=0)
session_pool = redis.ConnectionPool(max_connections=50, db=1)
queue_pool = redis.ConnectionPool(max_connections=20, db=2)

cache_client = redis.Redis(connection_pool=cache_pool)
session_client = redis.Redis(connection_pool=session_pool)
queue_client = redis.Redis(connection_pool=queue_pool)
```

### ✅ 5. Health checks réguliers

```javascript
// ✅ Health check endpoint pour k8s/monitoring
app.get('/health/redis', async (req, res) => {
    try {
        const start = Date.now();
        await redis.ping();
        const latency = Date.now() - start;

        res.json({
            status: 'healthy',
            latency: `${latency}ms`,
            connections: redis.status,
        });
    } catch (err) {
        res.status(503).json({
            status: 'unhealthy',
            error: err.message,
        });
    }
});
```

---

## Checklist de configuration

Avant de déployer en production, vérifiez :

- ✅ **Pool dimensionné** selon la charge attendue
- ✅ **Timeouts configurés** (connect, read, write)
- ✅ **Retry logic** activée avec backoff exponentiel
- ✅ **Health checks** périodiques (30-60s)
- ✅ **Monitoring** des métriques du pool
- ✅ **Graceful shutdown** implémenté
- ✅ **Logs** des erreurs de connexion
- ✅ **Alertes** sur pool exhaustion (>80% utilisation)
- ✅ **Keep-alive** activé pour connexions longue durée
- ✅ **Max lifetime** configuré pour renouvellement

---

## Points clés à retenir

🔑 **Toujours utiliser un pool** : Ne jamais créer de connexions ad-hoc

🔑 **Dimensionner correctement** : Formule = (Threads × Utilization) + Buffer

🔑 **Configurer les timeouts** : Éviter les blocages infinis

🔑 **Monitorer le pool** : Active connections, idle connections, wait time

🔑 **Gérer les erreurs** : Retry logic avec backoff exponentiel

🔑 **Graceful shutdown** : Toujours fermer proprement le pool

🔑 **Health checks** : Valider périodiquement les connexions

🔑 **Keep-alive** : Éviter les connexions stale

---

## Ressources complémentaires

- **redis-py connection pooling** : https://redis-py.readthedocs.io/en/stable/connections.html
- **ioredis connection** : https://github.com/redis/ioredis#connect-to-redis
- **go-redis pooling** : https://redis.uptrace.dev/guide/go-redis.html#connecting-to-redis-server

---

## Prochaine section

➡️ **Section 9.3** : Gestion des erreurs et retry logic - Construire une application résiliente

**Niveau** : Intermédiaire
**Durée estimée** : 60 minutes
**Prérequis** : Section 9.1 (Clients Redis)

⏭️ [Gestion des erreurs et retry logic](/09-integration-langages-programmation/03-gestion-erreurs-retry-logic.md)
