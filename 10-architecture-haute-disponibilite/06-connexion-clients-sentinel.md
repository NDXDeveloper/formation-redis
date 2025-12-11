🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.6 Connexion des clients via Sentinel (Service Discovery)

## Introduction

La connexion des applications clientes à Redis via Sentinel est fondamentalement différente d'une connexion directe. Les clients doivent interroger les Sentinels pour découvrir dynamiquement l'adresse du master actuel, et réagir automatiquement aux failovers. Cette section couvre tous les aspects de l'intégration client-Sentinel en production.

**Principe clé** : Les clients ne se connectent **jamais directement** à une instance Redis. Ils passent toujours par les Sentinels pour le Service Discovery.

---

## 🔍 Concept de Service Discovery

### Architecture client-Sentinel

```
┌─────────────────────────────────────────────────────────────────┐
│  FLUX DE CONNEXION CLIENT → SENTINEL → REDIS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Étape 1: Client démarre                                        │
│  ┌──────────────┐                                               │
│  │ Application  │                                               │
│  │   Client     │                                               │
│  └──────┬───────┘                                               │
│         │                                                       │
│         │ Configuration:                                        │
│         │ sentinels = [                                         │
│         │   "10.0.1.20:26379",                                  │
│         │   "10.0.1.21:26379",                                  │
│         │   "10.0.1.22:26379"                                   │
│         │ ]                                                     │
│         │ master_name = "mymaster"                              │
│         │                                                       │
│  Étape 2: Interroger Sentinels (round-robin ou random)          │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │  Sentinel 1  │ :26379                                        │
│  └──────┬───────┘                                               │
│         │                                                       │
│         │ Query: SENTINEL get-master-addr-by-name mymaster      │
│         │                                                       │
│         ▼                                                       │
│  Response: ["10.0.1.10", "6379"]                                │
│         │                                                       │
│  Étape 3: Connecter au master découvert                         │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │    Master    │ 10.0.1.10:6379                                │
│  │   (Redis)    │                                               │
│  └──────────────┘                                               │
│         │                                                       │
│         │ WRITE/READ operations                                 │
│         ▼                                                       │
│    Application logic                                            │
│                                                                 │
│  Étape 4: Surveillance continue (optionnel)                     │
│  ┌──────────────┐                                               │
│  │   Client     │                                               │
│  └──────┬───────┘                                               │
│         │                                                       │
│         │ Subscribe to: +switch-master                          │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │  Sentinel    │ Pub/Sub notification                          │
│  └──────────────┘                                               │
│         │                                                       │
│         │ Event: Master changed!                                │
│         │ Old: 10.0.1.10:6379                                   │
│         │ New: 10.0.1.11:6379                                   │
│         ▼                                                       │
│  Client reconnects to new master                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pourquoi ne pas se connecter directement ?

```
┌─────────────────────────────────────────────────────────────────┐
│  CONNEXION DIRECTE vs SERVICE DISCOVERY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ ANTI-PATTERN: Connexion directe (hardcodé)                  │
│  ══════════════════════════════════════════════════             │
│                                                                 │
│  # Bad configuration                                            │
│  redis_host = "10.0.1.10"  # ← IP hardcodée                     │
│  redis_port = 6379                                              │
│                                                                 │
│  client = Redis(host=redis_host, port=redis_port)               │
│                                                                 │
│  Problèmes:                                                     │
│  1. Si 10.0.1.10 fail → Client ne sait pas où aller             │
│  2. Après failover (nouveau master 10.0.1.11):                  │
│     • Client continue à essayer 10.0.1.10                       │
│     • Application en erreur jusqu'à redémarrage manuel          │
│  3. Nécessite intervention humaine                              │
│  4. Downtime prolongé                                           │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  ✅ BONNE PRATIQUE: Service Discovery via Sentinel              │
│  ══════════════════════════════════════════════════             │
│                                                                 │
│  # Good configuration                                           │
│  sentinels = [                                                  │
│      ("10.0.1.20", 26379),                                      │
│      ("10.0.1.21", 26379),                                      │
│      ("10.0.1.22", 26379)                                       │
│  ]                                                              │
│  master_name = "mymaster"                                       │
│                                                                 │
│  client = RedisSentinel(                                        │
│      sentinels=sentinels,                                       │
│      master_name=master_name                                    │
│  )                                                              │
│                                                                 │
│  Avantages:                                                     │
│  1. Client interroge Sentinels → Découvre master actuel         │
│  2. Si failover → Sentinels notifient client                    │
│  3. Client reconnecte automatiquement au nouveau master         │
│  4. Transparent pour l'application                              │
│  5. Pas d'intervention humaine                                  │
│  6. Downtime minimal (secondes)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💻 Implémentations par langage

### Python (redis-py)

**Installation** :

```bash
pip install redis
```

**Configuration de base** :

```python
#!/usr/bin/env python3
# redis_sentinel_client.py

from redis.sentinel import Sentinel
import logging

# Configuration logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Configuration Sentinel
SENTINEL_HOSTS = [
    ('10.0.1.20', 26379),
    ('10.0.1.21', 26379),
    ('10.0.1.22', 26379)
]
MASTER_NAME = 'mymaster'
PASSWORD = 'MasterPassword123!'

# ════════════════════════════════════════════════════════════════
# CONFIGURATION BASIQUE
# ════════════════════════════════════════════════════════════════

def create_sentinel_client():
    """Créer client Sentinel avec configuration basique"""

    sentinel = Sentinel(
        SENTINEL_HOSTS,
        socket_timeout=5.0,           # Timeout connexion Sentinel
        socket_connect_timeout=5.0,   # Timeout établissement connexion
        password=PASSWORD              # Password Redis (si configuré)
    )

    # Obtenir connexion au master
    master = sentinel.master_for(
        MASTER_NAME,
        socket_timeout=5.0,
        password=PASSWORD,
        decode_responses=True          # Décoder bytes → str
    )

    return master

# Utilisation
if __name__ == "__main__":
    try:
        client = create_sentinel_client()

        # Test write
        client.set('test:key', 'Hello from Python!')
        logger.info("Write successful")

        # Test read
        value = client.get('test:key')
        logger.info(f"Read value: {value}")

    except Exception as e:
        logger.error(f"Error: {e}")
```

**Configuration avancée avec retry et failover** :

```python
#!/usr/bin/env python3
# redis_sentinel_advanced.py

from redis.sentinel import Sentinel
from redis.exceptions import ConnectionError, TimeoutError, RedisError
import time
import logging
from typing import Optional, Any

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class RedisSentinelClient:
    """Client Redis avec Sentinel - Configuration Production"""

    def __init__(
        self,
        sentinels: list,
        master_name: str,
        password: Optional[str] = None,
        db: int = 0,
        max_retries: int = 3,
        retry_delay: float = 1.0
    ):
        self.sentinels = sentinels
        self.master_name = master_name
        self.password = password
        self.db = db
        self.max_retries = max_retries
        self.retry_delay = retry_delay

        # Créer Sentinel
        self.sentinel = Sentinel(
            self.sentinels,
            socket_timeout=5.0,
            socket_connect_timeout=5.0,
            password=self.password,
            # Sentinel ACL user (si configuré)
            # sentinel_kwargs={'username': 'sentinel_user'}
        )

        # Obtenir master
        self.master = self._get_master()

        # Optionnel: Obtenir replicas pour load balancing reads
        self.slave = self._get_slave()

        logger.info(f"Connected to master via Sentinel")

    def _get_master(self):
        """Obtenir connexion au master"""
        return self.sentinel.master_for(
            self.master_name,
            socket_timeout=5.0,
            socket_connect_timeout=5.0,
            password=self.password,
            db=self.db,
            decode_responses=True,
            # Pool de connexions
            max_connections=50,
            # Health check
            health_check_interval=30
        )

    def _get_slave(self):
        """Obtenir connexion à un replica (pour reads)"""
        return self.sentinel.slave_for(
            self.master_name,
            socket_timeout=5.0,
            socket_connect_timeout=5.0,
            password=self.password,
            db=self.db,
            decode_responses=True
        )

    def _execute_with_retry(self, operation, *args, **kwargs) -> Any:
        """Exécuter opération avec retry sur erreur connexion"""

        last_error = None

        for attempt in range(self.max_retries):
            try:
                return operation(*args, **kwargs)

            except (ConnectionError, TimeoutError) as e:
                last_error = e
                logger.warning(
                    f"Connection error (attempt {attempt + 1}/{self.max_retries}): {e}"
                )

                if attempt < self.max_retries - 1:
                    # Attendre avant retry
                    time.sleep(self.retry_delay * (attempt + 1))

                    # Reconnexion au master (peut avoir changé)
                    try:
                        self.master = self._get_master()
                        logger.info("Reconnected to master")
                    except Exception as reconnect_error:
                        logger.error(f"Reconnection failed: {reconnect_error}")

            except RedisError as e:
                # Erreur Redis (pas connexion) → pas de retry
                logger.error(f"Redis error: {e}")
                raise

        # Tous les retries échoués
        logger.error(f"All {self.max_retries} retries failed")
        raise last_error

    def get(self, key: str, from_replica: bool = False) -> Optional[str]:
        """
        Get value

        Args:
            key: Redis key
            from_replica: Si True, lire depuis replica (load balancing)
        """
        client = self.slave if from_replica and self.slave else self.master
        return self._execute_with_retry(client.get, key)

    def set(self, key: str, value: str, ex: Optional[int] = None) -> bool:
        """
        Set value (toujours sur master)

        Args:
            key: Redis key
            value: Value
            ex: Expiration (secondes)
        """
        if ex:
            return self._execute_with_retry(self.master.setex, key, ex, value)
        else:
            return self._execute_with_retry(self.master.set, key, value)

    def delete(self, key: str) -> int:
        """Delete key (sur master)"""
        return self._execute_with_retry(self.master.delete, key)

    def get_master_address(self) -> tuple:
        """Obtenir adresse actuelle du master"""
        sentinel_info = self.sentinel.discover_master(self.master_name)
        return sentinel_info

    def get_slaves_addresses(self) -> list:
        """Obtenir adresses des replicas"""
        return self.sentinel.discover_slaves(self.master_name)

    def close(self):
        """Fermer connexions"""
        if self.master:
            self.master.close()
        if self.slave:
            self.slave.close()


# ════════════════════════════════════════════════════════════════
# UTILISATION
# ════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    # Configuration
    SENTINELS = [
        ('10.0.1.20', 26379),
        ('10.0.1.21', 26379),
        ('10.0.1.22', 26379)
    ]

    # Créer client
    client = RedisSentinelClient(
        sentinels=SENTINELS,
        master_name='mymaster',
        password='MasterPassword123!',
        max_retries=3
    )

    try:
        # Write (sur master)
        client.set('user:1000', 'Alice', ex=3600)
        logger.info("Write successful")

        # Read (depuis master)
        value = client.get('user:1000')
        logger.info(f"Read from master: {value}")

        # Read (depuis replica - load balancing)
        value = client.get('user:1000', from_replica=True)
        logger.info(f"Read from replica: {value}")

        # Afficher topologie
        master_addr = client.get_master_address()
        logger.info(f"Current master: {master_addr}")

        slaves_addr = client.get_slaves_addresses()
        logger.info(f"Current replicas: {slaves_addr}")

    except Exception as e:
        logger.error(f"Error: {e}")

    finally:
        client.close()
```

---

### Node.js (ioredis)

**Installation** :

```bash
npm install ioredis
```

**Configuration** :

```javascript
// redis-sentinel-client.js

const Redis = require('ioredis');

// ════════════════════════════════════════════════════════════════
// CONFIGURATION SENTINEL
// ════════════════════════════════════════════════════════════════

const sentinelConfig = {
  sentinels: [
    { host: '10.0.1.20', port: 26379 },
    { host: '10.0.1.21', port: 26379 },
    { host: '10.0.1.22', port: 26379 }
  ],
  name: 'mymaster',  // Master name

  // Authentification
  password: 'MasterPassword123!',

  // Sentinel password (si différent)
  // sentinelPassword: 'SentinelPassword123!',

  // Options de connexion
  db: 0,
  connectTimeout: 10000,      // 10s
  commandTimeout: 5000,        // 5s

  // Retry strategy
  retryStrategy(times) {
    const delay = Math.min(times * 50, 2000);
    console.log(`Retry attempt ${times}, delay: ${delay}ms`);
    return delay;
  },

  // Reconnect on error
  reconnectOnError(err) {
    const targetError = 'READONLY';
    if (err.message.includes(targetError)) {
      // Reconnect si on essaie d'écrire sur replica
      console.log('Reconnecting due to READONLY error');
      return true;
    }
    return false;
  },

  // Sentinel options
  sentinelRetryStrategy(times) {
    const delay = Math.min(times * 100, 3000);
    return delay;
  },

  // Préférer IPv4
  family: 4,

  // Enable offline queue
  enableOfflineQueue: true,

  // Enable ready check
  enableReadyCheck: true,

  // Lazy connect (ne pas connecter immédiatement)
  lazyConnect: false
};

// ════════════════════════════════════════════════════════════════
// CRÉER CLIENT
// ════════════════════════════════════════════════════════════════

const redis = new Redis(sentinelConfig);

// ════════════════════════════════════════════════════════════════
// EVENT HANDLERS
// ════════════════════════════════════════════════════════════════

redis.on('connect', () => {
  console.log('Connected to Redis');
});

redis.on('ready', () => {
  console.log('Redis client ready');
});

redis.on('error', (err) => {
  console.error('Redis error:', err);
});

redis.on('close', () => {
  console.log('Redis connection closed');
});

redis.on('reconnecting', () => {
  console.log('Redis reconnecting...');
});

redis.on('end', () => {
  console.log('Redis connection ended');
});

// Événement spécifique Sentinel
redis.on('+switch-master', (info) => {
  console.log('Master switched!', info);
  // info = { name: 'mymaster', oldHost: '10.0.1.10', oldPort: 6379, newHost: '10.0.1.11', newPort: 6379 }
});

// ════════════════════════════════════════════════════════════════
// HELPER CLASS AVEC RETRY LOGIC
// ════════════════════════════════════════════════════════════════

class RedisSentinelClient {
  constructor(config) {
    this.redis = new Redis(config);
    this.maxRetries = 3;
  }

  async executeWithRetry(operation, maxRetries = this.maxRetries) {
    let lastError;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        console.warn(`Attempt ${attempt}/${maxRetries} failed:`, error.message);

        if (attempt < maxRetries) {
          const delay = Math.min(attempt * 100, 1000);
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }

    throw lastError;
  }

  async get(key, options = {}) {
    return this.executeWithRetry(() => this.redis.get(key), options.maxRetries);
  }

  async set(key, value, ex = null) {
    return this.executeWithRetry(async () => {
      if (ex) {
        return this.redis.setex(key, ex, value);
      } else {
        return this.redis.set(key, value);
      }
    });
  }

  async del(key) {
    return this.executeWithRetry(() => this.redis.del(key));
  }

  async getMasterAddress() {
    // Utiliser Sentinel pour obtenir master actuel
    const sentinel = new Redis({
      sentinels: sentinelConfig.sentinels,
      name: sentinelConfig.name,
      sentinelRetryStrategy: sentinelConfig.sentinelRetryStrategy
    });

    try {
      const master = await sentinel.sentinel('get-master-addr-by-name', sentinelConfig.name);
      return { host: master[0], port: parseInt(master[1]) };
    } finally {
      sentinel.disconnect();
    }
  }

  disconnect() {
    this.redis.disconnect();
  }
}

// ════════════════════════════════════════════════════════════════
// UTILISATION
// ════════════════════════════════════════════════════════════════

(async () => {
  const client = new RedisSentinelClient(sentinelConfig);

  try {
    // Write
    await client.set('user:1000', JSON.stringify({ name: 'Alice', age: 30 }), 3600);
    console.log('Write successful');

    // Read
    const userData = await client.get('user:1000');
    console.log('User data:', JSON.parse(userData));

    // Get current master
    const masterAddr = await client.getMasterAddress();
    console.log('Current master:', masterAddr);

  } catch (error) {
    console.error('Error:', error);
  } finally {
    client.disconnect();
  }
})();

// ════════════════════════════════════════════════════════════════
// EXPORT
// ════════════════════════════════════════════════════════════════

module.exports = { RedisSentinelClient, sentinelConfig };
```

---

### Java (Jedis)

**Maven dependency** :

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>5.1.0</version>
</dependency>
```

**Configuration** :

```java
// RedisSentinelClient.java

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.JedisSentinelPool;
import redis.clients.jedis.exceptions.JedisException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.HashSet;
import java.util.Set;

public class RedisSentinelClient {

    private static final Logger logger = LoggerFactory.getLogger(RedisSentinelClient.class);

    private JedisSentinelPool sentinelPool;
    private final String masterName;
    private final int maxRetries;

    // ════════════════════════════════════════════════════════════
    // CONSTRUCTOR
    // ════════════════════════════════════════════════════════════

    public RedisSentinelClient(
        Set<String> sentinels,
        String masterName,
        String password,
        int database,
        int maxRetries
    ) {
        this.masterName = masterName;
        this.maxRetries = maxRetries;

        // Configuration du pool de connexions
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(50);              // Max connexions
        poolConfig.setMaxIdle(10);               // Max idle
        poolConfig.setMinIdle(5);                // Min idle
        poolConfig.setTestOnBorrow(true);        // Test avant utilisation
        poolConfig.setTestOnReturn(true);        // Test après utilisation
        poolConfig.setTestWhileIdle(true);       // Test pendant idle
        poolConfig.setMinEvictableIdleTimeMillis(60000);  // 60s
        poolConfig.setTimeBetweenEvictionRunsMillis(30000); // 30s
        poolConfig.setNumTestsPerEvictionRun(3);
        poolConfig.setBlockWhenExhausted(true);  // Bloquer si pool plein

        // Créer Sentinel pool
        this.sentinelPool = new JedisSentinelPool(
            masterName,
            sentinels,
            poolConfig,
            5000,           // Connection timeout (ms)
            5000,           // Socket timeout (ms)
            password,       // Password
            database        // Database number
        );

        logger.info("Redis Sentinel client initialized for master: {}", masterName);
    }

    // ════════════════════════════════════════════════════════════
    // EXECUTE WITH RETRY
    // ════════════════════════════════════════════════════════════

    private <T> T executeWithRetry(RedisOperation<T> operation) throws JedisException {
        JedisException lastException = null;

        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try (Jedis jedis = sentinelPool.getResource()) {
                return operation.execute(jedis);

            } catch (JedisException e) {
                lastException = e;
                logger.warn("Attempt {}/{} failed: {}", attempt, maxRetries, e.getMessage());

                if (attempt < maxRetries) {
                    try {
                        Thread.sleep(Math.min(attempt * 100, 1000));
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new JedisException("Interrupted during retry", ie);
                    }
                }
            }
        }

        logger.error("All {} retries failed", maxRetries);
        throw lastException;
    }

    @FunctionalInterface
    private interface RedisOperation<T> {
        T execute(Jedis jedis) throws JedisException;
    }

    // ════════════════════════════════════════════════════════════
    // PUBLIC API
    // ════════════════════════════════════════════════════════════

    public String get(String key) {
        return executeWithRetry(jedis -> jedis.get(key));
    }

    public String set(String key, String value) {
        return executeWithRetry(jedis -> jedis.set(key, value));
    }

    public String setex(String key, int seconds, String value) {
        return executeWithRetry(jedis -> jedis.setex(key, seconds, value));
    }

    public Long del(String key) {
        return executeWithRetry(jedis -> jedis.del(key));
    }

    public String getCurrentMaster() {
        try (Jedis jedis = sentinelPool.getResource()) {
            return jedis.getClient().getHost() + ":" + jedis.getClient().getPort();
        }
    }

    public void close() {
        if (sentinelPool != null && !sentinelPool.isClosed()) {
            sentinelPool.close();
            logger.info("Sentinel pool closed");
        }
    }

    // ════════════════════════════════════════════════════════════
    // MAIN (EXEMPLE D'UTILISATION)
    // ════════════════════════════════════════════════════════════

    public static void main(String[] args) {
        // Configuration Sentinels
        Set<String> sentinels = new HashSet<>();
        sentinels.add("10.0.1.20:26379");
        sentinels.add("10.0.1.21:26379");
        sentinels.add("10.0.1.22:26379");

        // Créer client
        RedisSentinelClient client = new RedisSentinelClient(
            sentinels,
            "mymaster",
            "MasterPassword123!",
            0,
            3
        );

        try {
            // Write
            client.setex("user:1000", 3600, "{\"name\":\"Alice\",\"age\":30}");
            logger.info("Write successful");

            // Read
            String userData = client.get("user:1000");
            logger.info("User data: {}", userData);

            // Get current master
            String masterAddr = client.getCurrentMaster();
            logger.info("Current master: {}", masterAddr);

        } catch (Exception e) {
            logger.error("Error", e);
        } finally {
            client.close();
        }
    }
}
```

---

### Go (go-redis)

**Installation** :

```bash
go get github.com/redis/go-redis/v9
```

**Configuration** :

```go
// redis_sentinel_client.go

package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

// ════════════════════════════════════════════════════════════════
// REDIS SENTINEL CLIENT
// ════════════════════════════════════════════════════════════════

type RedisSentinelClient struct {
    client     *redis.Client
    maxRetries int
}

func NewRedisSentinelClient(
    sentinels []string,
    masterName string,
    password string,
    db int,
    maxRetries int,
) *RedisSentinelClient {

    // Configuration Sentinel
    client := redis.NewFailoverClient(&redis.FailoverOptions{
        MasterName:    masterName,
        SentinelAddrs: sentinels,

        // Authentification
        Password: password,
        DB:       db,

        // Timeouts
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,

        // Pool de connexions
        PoolSize:     50,
        MinIdleConns: 10,
        MaxIdleConns: 20,

        // Retry strategy
        MaxRetries:      maxRetries,
        MinRetryBackoff: 100 * time.Millisecond,
        MaxRetryBackoff: 2 * time.Second,

        // Health check
        PoolTimeout: 4 * time.Second,
    })

    // Test connexion
    ctx := context.Background()
    if err := client.Ping(ctx).Err(); err != nil {
        log.Printf("Warning: Initial connection failed: %v", err)
    } else {
        log.Println("Connected to Redis via Sentinel")
    }

    return &RedisSentinelClient{
        client:     client,
        maxRetries: maxRetries,
    }
}

// ════════════════════════════════════════════════════════════════
// OPERATIONS WITH RETRY
// ════════════════════════════════════════════════════════════════

func (r *RedisSentinelClient) executeWithRetry(
    ctx context.Context,
    operation func() error,
) error {
    var lastErr error

    for attempt := 1; attempt <= r.maxRetries; attempt++ {
        err := operation()
        if err == nil {
            return nil
        }

        lastErr = err
        log.Printf("Attempt %d/%d failed: %v", attempt, r.maxRetries, err)

        if attempt < r.maxRetries {
            backoff := time.Duration(attempt*100) * time.Millisecond
            if backoff > 2*time.Second {
                backoff = 2 * time.Second
            }
            time.Sleep(backoff)
        }
    }

    return fmt.Errorf("all %d retries failed: %w", r.maxRetries, lastErr)
}

// ════════════════════════════════════════════════════════════════
// PUBLIC API
// ════════════════════════════════════════════════════════════════

func (r *RedisSentinelClient) Get(ctx context.Context, key string) (string, error) {
    var result string
    var err error

    operation := func() error {
        result, err = r.client.Get(ctx, key).Result()
        return err
    }

    if err := r.executeWithRetry(ctx, operation); err != nil {
        return "", err
    }

    return result, nil
}

func (r *RedisSentinelClient) Set(
    ctx context.Context,
    key string,
    value interface{},
    expiration time.Duration,
) error {
    operation := func() error {
        return r.client.Set(ctx, key, value, expiration).Err()
    }

    return r.executeWithRetry(ctx, operation)
}

func (r *RedisSentinelClient) Del(ctx context.Context, keys ...string) error {
    operation := func() error {
        return r.client.Del(ctx, keys...).Err()
    }

    return r.executeWithRetry(ctx, operation)
}

func (r *RedisSentinelClient) GetMasterAddress(ctx context.Context) (string, error) {
    // Utiliser INFO pour obtenir adresse master
    info, err := r.client.Info(ctx, "replication").Result()
    if err != nil {
        return "", err
    }

    // Parser info pour extraire role et master_host
    // (Simplifié pour l'exemple)
    return fmt.Sprintf("%s:%d", r.client.Options().Addr, 6379), nil
}

func (r *RedisSentinelClient) Close() error {
    return r.client.Close()
}

// ════════════════════════════════════════════════════════════════
// MAIN (EXEMPLE)
// ════════════════════════════════════════════════════════════════

func main() {
    // Configuration Sentinels
    sentinels := []string{
        "10.0.1.20:26379",
        "10.0.1.21:26379",
        "10.0.1.22:26379",
    }

    // Créer client
    client := NewRedisSentinelClient(
        sentinels,
        "mymaster",
        "MasterPassword123!",
        0,
        3,
    )
    defer client.Close()

    ctx := context.Background()

    // Write
    err := client.Set(ctx, "user:1000", `{"name":"Alice","age":30}`, time.Hour)
    if err != nil {
        log.Fatalf("Set failed: %v", err)
    }
    log.Println("Write successful")

    // Read
    userData, err := client.Get(ctx, "user:1000")
    if err != nil {
        log.Fatalf("Get failed: %v", err)
    }
    log.Printf("User data: %s", userData)

    // Get master address
    masterAddr, err := client.GetMasterAddress(ctx)
    if err != nil {
        log.Printf("Failed to get master address: %v", err)
    } else {
        log.Printf("Current master: %s", masterAddr)
    }
}
```

---

## 🔄 Patterns de reconnexion

### Pattern 1 : Reconnexion automatique avec exponential backoff

```python
#!/usr/bin/env python3
# exponential_backoff.py

import time
import random
from redis.sentinel import Sentinel
from redis.exceptions import ConnectionError, TimeoutError
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ExponentialBackoffClient:
    """Client avec exponential backoff"""

    def __init__(self, sentinels, master_name, password):
        self.sentinels = sentinels
        self.master_name = master_name
        self.password = password
        self.max_retries = 5
        self.base_delay = 0.1  # 100ms
        self.max_delay = 30.0  # 30 secondes

        self.sentinel = Sentinel(self.sentinels, socket_timeout=5.0)
        self.master = None
        self._connect()

    def _connect(self):
        """Établir connexion au master"""
        try:
            self.master = self.sentinel.master_for(
                self.master_name,
                socket_timeout=5.0,
                password=self.password,
                decode_responses=True
            )
            logger.info("Connected to master")
        except Exception as e:
            logger.error(f"Connection failed: {e}")
            raise

    def _exponential_backoff(self, attempt):
        """
        Calculer délai avec exponential backoff + jitter

        Formule: delay = min(base * 2^attempt + random_jitter, max_delay)
        """
        delay = min(
            self.base_delay * (2 ** attempt),
            self.max_delay
        )

        # Ajouter jitter (±20%)
        jitter = delay * 0.2 * (2 * random.random() - 1)
        delay += jitter

        return max(0, delay)  # Pas de délai négatif

    def execute_with_backoff(self, operation, *args, **kwargs):
        """Exécuter opération avec exponential backoff"""

        for attempt in range(self.max_retries):
            try:
                return operation(*args, **kwargs)

            except (ConnectionError, TimeoutError) as e:
                if attempt == self.max_retries - 1:
                    logger.error(f"All retries exhausted")
                    raise

                delay = self._exponential_backoff(attempt)
                logger.warning(
                    f"Attempt {attempt + 1} failed, "
                    f"retrying in {delay:.2f}s: {e}"
                )
                time.sleep(delay)

                # Tenter reconnexion
                try:
                    self._connect()
                except Exception as conn_err:
                    logger.error(f"Reconnection failed: {conn_err}")

    def get(self, key):
        return self.execute_with_backoff(self.master.get, key)

    def set(self, key, value):
        return self.execute_with_backoff(self.master.set, key, value)


# Exemple d'utilisation
if __name__ == "__main__":
    SENTINELS = [
        ('10.0.1.20', 26379),
        ('10.0.1.21', 26379),
        ('10.0.1.22', 26379)
    ]

    client = ExponentialBackoffClient(
        sentinels=SENTINELS,
        master_name='mymaster',
        password='MasterPassword123!'
    )

    # Simulation d'utilisation continue
    for i in range(100):
        try:
            client.set(f'test:{i}', f'value-{i}')
            logger.info(f"Iteration {i} successful")
            time.sleep(1)
        except Exception as e:
            logger.error(f"Iteration {i} failed: {e}")
            break
```

### Pattern 2 : Circuit Breaker

```python
#!/usr/bin/env python3
# circuit_breaker.py

import time
from enum import Enum
from redis.sentinel import Sentinel
from redis.exceptions import RedisError
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class CircuitState(Enum):
    CLOSED = "closed"      # Normal, requêtes passent
    OPEN = "open"          # Circuit ouvert, rejeter requêtes
    HALF_OPEN = "half_open"  # Test si service revenu

class CircuitBreakerClient:
    """Client avec Circuit Breaker pattern"""

    def __init__(
        self,
        sentinels,
        master_name,
        password,
        failure_threshold=5,      # Nombre d'échecs avant ouverture
        timeout=60,               # Durée circuit ouvert (secondes)
        success_threshold=2       # Succès requis pour fermer circuit
    ):
        self.sentinels = sentinels
        self.master_name = master_name
        self.password = password

        # Circuit breaker state
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout
        self.last_failure_time = None

        # Redis connection
        self.sentinel = Sentinel(self.sentinels, socket_timeout=5.0)
        self.master = self._get_master()

    def _get_master(self):
        return self.sentinel.master_for(
            self.master_name,
            socket_timeout=5.0,
            password=self.password,
            decode_responses=True
        )

    def _record_success(self):
        """Enregistrer succès"""
        self.failure_count = 0

        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            logger.info(
                f"Success in HALF_OPEN: "
                f"{self.success_count}/{self.success_threshold}"
            )

            if self.success_count >= self.success_threshold:
                logger.info("Circuit CLOSED (recovered)")
                self.state = CircuitState.CLOSED
                self.success_count = 0

    def _record_failure(self, error):
        """Enregistrer échec"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        logger.warning(
            f"Failure recorded: {self.failure_count}/{self.failure_threshold} - {error}"
        )

        if self.failure_count >= self.failure_threshold:
            logger.error("Circuit OPENED (too many failures)")
            self.state = CircuitState.OPEN

    def _should_attempt_request(self):
        """Vérifier si on doit tenter la requête"""

        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            # Vérifier si timeout expiré
            if time.time() - self.last_failure_time >= self.timeout:
                logger.info("Circuit HALF_OPEN (testing recovery)")
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
                return True
            else:
                return False

        if self.state == CircuitState.HALF_OPEN:
            return True

        return False

    def execute(self, operation, *args, **kwargs):
        """Exécuter opération avec circuit breaker"""

        if not self._should_attempt_request():
            raise Exception(
                f"Circuit breaker is OPEN "
                f"(will retry in {self.timeout - (time.time() - self.last_failure_time):.1f}s)"
            )

        try:
            result = operation(*args, **kwargs)
            self._record_success()
            return result

        except RedisError as e:
            self._record_failure(e)
            raise

    def get(self, key):
        return self.execute(self.master.get, key)

    def set(self, key, value):
        return self.execute(self.master.set, key, value)


# Exemple d'utilisation
if __name__ == "__main__":
    SENTINELS = [
        ('10.0.1.20', 26379),
        ('10.0.1.21', 26379),
        ('10.0.1.22', 26379)
    ]

    client = CircuitBreakerClient(
        sentinels=SENTINELS,
        master_name='mymaster',
        password='MasterPassword123!',
        failure_threshold=3,
        timeout=10,
        success_threshold=2
    )

    # Simulation
    for i in range(20):
        try:
            client.set(f'test:{i}', f'value-{i}')
            logger.info(f"✅ Request {i} successful")
        except Exception as e:
            logger.error(f"❌ Request {i} failed: {e}")

        time.sleep(1)
```

---

## 📊 Load Balancing des lectures

### Stratégie : Lire depuis replicas

```python
#!/usr/bin/env python3
# read_load_balancing.py

from redis.sentinel import Sentinel
import random
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LoadBalancedRedisClient:
    """Client avec load balancing des lectures sur replicas"""

    def __init__(self, sentinels, master_name, password):
        self.sentinel = Sentinel(
            sentinels,
            socket_timeout=5.0,
            password=password
        )
        self.master_name = master_name
        self.password = password

        # Connexion master (writes)
        self.master = self.sentinel.master_for(
            master_name,
            socket_timeout=5.0,
            password=password,
            decode_responses=True
        )

        # Pool de connexions replicas (reads)
        self.replica_pool = []
        self._refresh_replica_pool()

    def _refresh_replica_pool(self):
        """Rafraîchir liste des replicas disponibles"""
        try:
            replicas_info = self.sentinel.discover_slaves(self.master_name)
            self.replica_pool = []

            for replica_info in replicas_info:
                host, port = replica_info
                try:
                    replica_client = self.sentinel.slave_for(
                        self.master_name,
                        socket_timeout=5.0,
                        password=self.password,
                        decode_responses=True
                    )
                    self.replica_pool.append(replica_client)
                    logger.info(f"Added replica to pool: {host}:{port}")
                except Exception as e:
                    logger.warning(f"Failed to connect to replica {host}:{port}: {e}")

            if not self.replica_pool:
                logger.warning("No replicas available, will read from master")

        except Exception as e:
            logger.error(f"Failed to refresh replica pool: {e}")

    def _get_replica(self, strategy='random'):
        """
        Obtenir un replica selon la stratégie

        Strategies:
        - 'random': Sélection aléatoire
        - 'round_robin': Tour par tour (TODO)
        - 'least_loaded': Moins chargé (TODO)
        """
        if not self.replica_pool:
            self._refresh_replica_pool()

        if not self.replica_pool:
            logger.warning("No replicas available, using master")
            return self.master

        if strategy == 'random':
            return random.choice(self.replica_pool)

        # Fallback
        return self.replica_pool[0]

    def write(self, key, value, ex=None):
        """Écriture (toujours sur master)"""
        if ex:
            return self.master.setex(key, ex, value)
        else:
            return self.master.set(key, value)

    def read(self, key, prefer_replica=True):
        """
        Lecture

        Args:
            key: Redis key
            prefer_replica: Si True, lire depuis replica (load balancing)
                          Si False, lire depuis master (cohérence)
        """
        if prefer_replica:
            replica = self._get_replica()
            try:
                return replica.get(key)
            except Exception as e:
                logger.warning(f"Failed to read from replica, trying master: {e}")
                return self.master.get(key)
        else:
            return self.master.get(key)

    def delete(self, key):
        """Suppression (sur master)"""
        return self.master.delete(key)


# Utilisation
if __name__ == "__main__":
    SENTINELS = [
        ('10.0.1.20', 26379),
        ('10.0.1.21', 26379),
        ('10.0.1.22', 26379)
    ]

    client = LoadBalancedRedisClient(
        sentinels=SENTINELS,
        master_name='mymaster',
        password='MasterPassword123!'
    )

    # Write (sur master)
    client.write('user:1000', 'Alice', ex=3600)
    logger.info("Write successful")

    # Read (depuis replica - load balanced)
    for i in range(10):
        value = client.read('user:1000', prefer_replica=True)
        logger.info(f"Read {i}: {value}")

    # Read (depuis master - cohérence garantie)
    value = client.read('user:1000', prefer_replica=False)
    logger.info(f"Read from master: {value}")
```

---

## 🎓 Points clés à retenir

1. **Ne JAMAIS hardcoder IPs Redis** : Toujours utiliser Sentinel Service Discovery
2. **Configuration Sentinels multiples** : Minimum 3, nombre impair
3. **Retry logic obligatoire** : Gérer failovers transparents
4. **Exponential backoff** : Éviter surcharge lors de pannes
5. **Circuit breaker** : Protéger application des cascades de pannes
6. **Load balancing reads** : Utiliser replicas pour scalabilité
7. **Event handlers** : Écouter +switch-master pour réactivité
8. **Pool de connexions** : Optimiser performance et ressources
9. **Timeouts appropriés** : Balance entre latence et résilience
10. **Monitoring connexions** : Alerter sur échecs répétés

---

## 🔗 Références

- [Redis Sentinel Clients](https://redis.io/docs/management/sentinel/#sentinel-clients)
- [redis-py Documentation](https://redis-py.readthedocs.io/)
- [ioredis Documentation](https://github.com/redis/ioredis)
- [Jedis Documentation](https://github.com/redis/jedis)
- [go-redis Documentation](https://redis.uptrace.dev/)

---

**Section suivante** : [10.7 Tests de basculement et scénarios de failure](./07-tests-basculement-scenarios-failure.md)

⏭️ [Tests de basculement et scénarios de failure](/10-architecture-haute-disponibilite/07-tests-basculement-scenarios-failure.md)
