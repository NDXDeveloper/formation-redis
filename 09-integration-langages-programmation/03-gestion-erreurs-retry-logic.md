🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.3 Gestion des erreurs et retry logic

## Introduction

Dans un environnement distribué, les erreurs sont **inévitables**. Redis peut être temporairement indisponible, le réseau peut être congestionné, ou une commande peut échouer pour diverses raisons. Une gestion d'erreurs robuste est cruciale pour :

- 🛡️ **Résilience** : Survivre aux défaillances temporaires
- 📊 **Fiabilité** : Garantir la cohérence des données
- 👁️ **Observabilité** : Détecter et diagnostiquer les problèmes
- ⚡ **Performance** : Éviter les cascades de défaillances

Cette section couvre les stratégies de gestion d'erreurs et de retry pour construire des applications Redis robustes.

---

## Taxonomie des erreurs Redis

### Types d'erreurs

| Type | Cause | Récupérable | Stratégie |
|------|-------|-------------|-----------|
| **Connection Error** | Redis indisponible, réseau down | ✅ | Retry avec backoff |
| **Timeout Error** | Opération trop lente | ✅ | Retry avec timeout augmenté |
| **Response Error** | Commande invalide, mauvais type | ❌ | Corriger le code |
| **READONLY Error** | Écriture sur replica | ✅ | Rediriger vers master |
| **MOVED/ASK** | Cluster resharding | ✅ | Suivre redirection |
| **OOM Error** | Mémoire pleine | ⚠️ | Éviction ou scale up |
| **NOAUTH** | Authentification manquante | ❌ | Configurer credentials |
| **BUSY** | Script Lua en cours | ✅ | Retry après délai |
| **LOADING** | Redis en cours de démarrage | ✅ | Retry avec backoff |

### Classification pour la gestion

```
┌─────────────────────────────────────────────────┐
│              Erreurs Redis                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌───────────────────┐     ┌─────────────────┐  │
│  │ Erreurs réseau    │     │ Erreurs Redis   │  │
│  ├───────────────────┤     ├─────────────────┤  │
│  │ • Connection      │     │ • WRONGTYPE     │  │
│  │ • Timeout         │     │ • NOAUTH        │  │
│  │ • DNS failure     │     │ • MOVED/ASK     │  │
│  │                   │     │ • READONLY      │  │
│  │ → RETRY           │     │ • OOM           │  │
│  └───────────────────┘     │ → LOGIC         │  │
│                            └─────────────────┘  │
│                                                 │
│  ┌───────────────────┐    ┌─────────────────┐   │
│  │ Erreurs           │    │ Erreurs         │   │
│  │ applicatives      │    │ programmation   │   │
│  ├───────────────────┤    ├─────────────────┤   │
│  │ • Circuit open    │    │ • Syntaxe       │   │
│  │ • Rate limit      │    │ • Null pointer  │   │
│  │ • Degraded mode   │    │ • Type error    │   │
│  │                   │    │                 │   │
│  │ → FALLBACK        │    │ → FIX CODE      │   │
│  └───────────────────┘    └─────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

## Stratégies de retry

### 1. Retry simple (linéaire)

**Principe** : Réessayer après un délai fixe

```python
# ❌ PROBLÈME : Peut surcharger Redis en cas de défaillance
import time

def simple_retry(func, max_attempts=3, delay=1):
    for attempt in range(max_attempts):
        try:
            return func()
        except redis.ConnectionError:
            if attempt < max_attempts - 1:
                time.sleep(delay)  # Toujours 1 seconde
            else:
                raise
```

**Inconvénients :**
- Peut créer un "thundering herd" (ruée simultanée)
- Délai fixe inadapté si Redis est surchargé
- Tous les clients retentent en même temps

### 2. Exponential Backoff (recommandé)

**Principe** : Doubler le délai à chaque tentative

```
Tentative 1 : Délai = 100ms
Tentative 2 : Délai = 200ms
Tentative 3 : Délai = 400ms
Tentative 4 : Délai = 800ms
Tentative 5 : Délai = 1600ms (plafonné à max)
```

**Avantages :**
- Réduit la charge sur Redis
- Laisse le temps à Redis de récupérer
- Évite les cascades de défaillances

### 3. Exponential Backoff avec Jitter (optimal)

**Principe** : Ajouter un facteur aléatoire pour distribuer les retries

```python
import random
import time

def calculate_backoff(attempt, base_delay=0.1, max_delay=30):
    """
    Calcule le délai avec exponential backoff + jitter

    Formule : min(max_delay, base_delay * 2^attempt) * (0.5 + random(0, 0.5))
    """
    exponential_delay = base_delay * (2 ** attempt)
    delay = min(exponential_delay, max_delay)
    # Jitter : +/- 50% aléatoire
    jitter = delay * (0.5 + random.random() * 0.5)
    return jitter

# Exemple de délais générés
for i in range(6):
    delay = calculate_backoff(i)
    print(f"Tentative {i+1}: {delay:.2f}s")

# Output exemple :
# Tentative 1: 0.07s
# Tentative 2: 0.15s
# Tentative 3: 0.31s
# Tentative 4: 0.68s
# Tentative 5: 1.24s
# Tentative 6: 2.71s
```

---

## Implémentations par langage

### Python (redis-py)

#### Décorateur de retry avec exponential backoff

```python
import redis
import time
import random
import logging
from functools import wraps
from typing import Callable, Any, Tuple, Type

logger = logging.getLogger(__name__)

class RetryConfig:
    """Configuration des tentatives de retry"""
    def __init__(
        self,
        max_attempts: int = 3,
        base_delay: float = 0.1,
        max_delay: float = 30.0,
        exponential_base: float = 2.0,
        jitter: bool = True,
        retriable_exceptions: Tuple[Type[Exception], ...] = None
    ):
        self.max_attempts = max_attempts
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter
        self.retriable_exceptions = retriable_exceptions or (
            redis.ConnectionError,
            redis.TimeoutError,
            redis.BusyLoadingError,
        )

def retry_with_backoff(config: RetryConfig = None):
    """
    Décorateur pour retry avec exponential backoff et jitter

    Usage:
        @retry_with_backoff(RetryConfig(max_attempts=5))
        def my_redis_operation():
            return redis_client.get('key')
    """
    if config is None:
        config = RetryConfig()

    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            last_exception = None

            for attempt in range(config.max_attempts):
                try:
                    result = func(*args, **kwargs)

                    # Log si retry réussi après échec
                    if attempt > 0:
                        logger.info(
                            f"✅ {func.__name__} succeeded after {attempt + 1} attempts"
                        )

                    return result

                except config.retriable_exceptions as e:
                    last_exception = e

                    if attempt < config.max_attempts - 1:
                        # Calcul du délai avec exponential backoff
                        delay = config.base_delay * (config.exponential_base ** attempt)
                        delay = min(delay, config.max_delay)

                        # Ajout du jitter
                        if config.jitter:
                            jitter_factor = 0.5 + random.random() * 0.5
                            delay *= jitter_factor

                        logger.warning(
                            f"⚠️ {func.__name__} failed (attempt {attempt + 1}/{config.max_attempts}): "
                            f"{type(e).__name__}: {e}. Retrying in {delay:.2f}s"
                        )

                        time.sleep(delay)
                    else:
                        logger.error(
                            f"❌ {func.__name__} failed after {config.max_attempts} attempts: "
                            f"{type(e).__name__}: {e}"
                        )

            # Toutes les tentatives ont échoué
            raise last_exception

        return wrapper
    return decorator

# Utilisation
redis_client = redis.Redis(host='localhost', port=6379, socket_timeout=5)

@retry_with_backoff(RetryConfig(max_attempts=5, base_delay=0.1, max_delay=10))
def get_user(user_id: int) -> dict:
    """Récupère un utilisateur avec retry automatique"""
    data = redis_client.get(f'user:{user_id}')
    if data:
        import json
        return json.loads(data)
    return None

@retry_with_backoff(RetryConfig(max_attempts=3))
def increment_counter(key: str) -> int:
    """Incrémente un compteur avec retry"""
    return redis_client.incr(key)

# Test
try:
    user = get_user(123)
    print(f"User: {user}")

    count = increment_counter('page:views')
    print(f"Views: {count}")
except redis.RedisError as e:
    print(f"Operation failed: {e}")
```

#### Gestionnaire d'erreurs avec fallback

```python
import redis
from typing import Optional, Any, Callable
import logging

logger = logging.getLogger(__name__)

class ResilientRedisClient:
    """Client Redis avec gestion d'erreurs avancée et fallback"""

    def __init__(self, redis_client: redis.Redis):
        self.client = redis_client
        self.error_count = 0
        self.last_error = None

    def get_with_fallback(
        self,
        key: str,
        fallback_fn: Optional[Callable[[], Any]] = None,
        default: Any = None
    ) -> Any:
        """
        GET avec fallback en cas d'erreur

        Args:
            key: Clé Redis
            fallback_fn: Fonction à appeler en cas d'erreur
            default: Valeur par défaut si fallback_fn est None
        """
        try:
            value = self.client.get(key)
            self.error_count = 0  # Reset sur succès
            return value

        except redis.ConnectionError as e:
            self.error_count += 1
            self.last_error = e
            logger.error(f"Redis connection error for key '{key}': {e}")

            # Utiliser le fallback
            if fallback_fn:
                logger.info(f"Using fallback for key '{key}'")
                return fallback_fn()
            return default

        except redis.TimeoutError as e:
            self.error_count += 1
            self.last_error = e
            logger.error(f"Redis timeout for key '{key}': {e}")

            if fallback_fn:
                return fallback_fn()
            return default

        except redis.ResponseError as e:
            # Erreur de commande (non récupérable par retry)
            logger.error(f"Redis response error for key '{key}': {e}")
            raise

    def set_safe(
        self,
        key: str,
        value: Any,
        ex: Optional[int] = None,
        suppress_errors: bool = False
    ) -> bool:
        """
        SET avec gestion d'erreurs sûre

        Args:
            key: Clé Redis
            value: Valeur à stocker
            ex: Expiration en secondes
            suppress_errors: Si True, retourne False au lieu de lever l'exception

        Returns:
            True si succès, False si échec (seulement si suppress_errors=True)
        """
        try:
            if ex:
                self.client.setex(key, ex, value)
            else:
                self.client.set(key, value)

            self.error_count = 0
            return True

        except redis.RedisError as e:
            self.error_count += 1
            self.last_error = e
            logger.error(f"Failed to set key '{key}': {type(e).__name__}: {e}")

            if suppress_errors:
                return False
            raise

    def execute_with_circuit_breaker(
        self,
        operation: Callable,
        threshold: int = 5
    ) -> Optional[Any]:
        """
        Exécute une opération avec circuit breaker simple

        Args:
            operation: Fonction à exécuter
            threshold: Nombre d'erreurs avant d'ouvrir le circuit
        """
        if self.error_count >= threshold:
            logger.warning(
                f"Circuit breaker OPEN: {self.error_count} consecutive errors. "
                f"Last error: {self.last_error}"
            )
            return None

        try:
            result = operation()
            self.error_count = 0  # Reset sur succès
            return result
        except redis.RedisError as e:
            self.error_count += 1
            self.last_error = e
            logger.error(f"Operation failed: {e}")

            if self.error_count >= threshold:
                logger.error(f"Circuit breaker opened after {threshold} errors")

            raise

    def get_health_status(self) -> dict:
        """Retourne le statut de santé du client"""
        return {
            'healthy': self.error_count < 5,
            'error_count': self.error_count,
            'last_error': str(self.last_error) if self.last_error else None,
        }

# Utilisation
redis_client = redis.Redis(host='localhost', decode_responses=True)
resilient_client = ResilientRedisClient(redis_client)

# GET avec fallback vers base de données
def fetch_from_database(user_id):
    # Simuler récupération DB
    return {'id': user_id, 'name': 'Fallback User'}

user_id = 123
user = resilient_client.get_with_fallback(
    f'user:{user_id}',
    fallback_fn=lambda: fetch_from_database(user_id)
)

# SET avec suppression d'erreurs
success = resilient_client.set_safe('session:abc', 'data', ex=3600, suppress_errors=True)
if not success:
    logger.warning("Failed to save session, continuing without cache")

# Vérifier l'état de santé
health = resilient_client.get_health_status()
print(f"Redis health: {health}")
```

---

### Node.js (ioredis)

#### Retry avec exponential backoff

```javascript
import Redis from 'ioredis';

/**
 * Configuration du retry avec exponential backoff et jitter
 */
const redis = new Redis({
    host: 'localhost',
    port: 6379,

    // Stratégie de retry personnalisée
    retryStrategy(times) {
        // times : nombre de tentatives
        const maxRetries = 10;
        const baseDelay = 100; // ms
        const maxDelay = 30000; // 30 secondes

        if (times > maxRetries) {
            // Abandon après maxRetries tentatives
            console.error(`❌ Max retries (${maxRetries}) atteint, abandon`);
            return null; // null = arrêt des tentatives
        }

        // Exponential backoff : 100ms, 200ms, 400ms, 800ms...
        let delay = Math.min(baseDelay * Math.pow(2, times - 1), maxDelay);

        // Jitter : ±25% aléatoire
        const jitter = delay * 0.25 * (Math.random() - 0.5);
        delay = Math.floor(delay + jitter);

        console.log(`⚠️ Tentative ${times}, retry dans ${delay}ms`);
        return delay;
    },

    // Gestion des erreurs spécifiques
    reconnectOnError(err) {
        const targetErrors = ['READONLY', 'ECONNRESET'];

        if (targetErrors.some(e => err.message.includes(e))) {
            console.log(`🔄 Reconnexion suite à: ${err.message}`);
            return true; // Reconnecte
        }

        return false; // Ne reconnecte pas
    },

    maxRetriesPerRequest: 3, // Max retry par commande
    enableOfflineQueue: false, // Échoue rapidement si déconnecté
});

// Events de monitoring
redis.on('error', (err) => {
    console.error('❌ Redis error:', err.message);
});

redis.on('reconnecting', (delay) => {
    console.log(`🔄 Reconnexion dans ${delay}ms...`);
});

redis.on('connect', () => {
    console.log('✅ Redis connecté');
});

export default redis;
```

#### Wrapper avec gestion d'erreurs avancée

```javascript
import Redis from 'ioredis';
import { EventEmitter } from 'events';

class ResilientRedisClient extends EventEmitter {
    constructor(config) {
        super();

        this.redis = new Redis({
            ...config,
            retryStrategy: this.retryStrategy.bind(this),
            reconnectOnError: this.reconnectOnError.bind(this),
        });

        this.stats = {
            successCount: 0,
            errorCount: 0,
            consecutiveErrors: 0,
            lastError: null,
            circuitBreakerOpen: false,
        };

        this.circuitBreakerThreshold = config.circuitBreakerThreshold || 5;
        this.setupEventHandlers();
    }

    setupEventHandlers() {
        this.redis.on('error', (err) => {
            this.stats.errorCount++;
            this.stats.consecutiveErrors++;
            this.stats.lastError = err;
            this.emit('error', err);

            // Ouvrir le circuit breaker si trop d'erreurs
            if (this.stats.consecutiveErrors >= this.circuitBreakerThreshold) {
                this.stats.circuitBreakerOpen = true;
                this.emit('circuit-breaker-open', this.stats);
                console.error(
                    `🔥 Circuit breaker OPEN après ${this.stats.consecutiveErrors} erreurs`
                );
            }
        });

        this.redis.on('ready', () => {
            // Reset sur reconnexion réussie
            if (this.stats.circuitBreakerOpen) {
                console.log('✅ Circuit breaker CLOSED');
            }
            this.stats.consecutiveErrors = 0;
            this.stats.circuitBreakerOpen = false;
            this.emit('circuit-breaker-closed');
        });
    }

    retryStrategy(times) {
        const maxRetries = 10;
        const baseDelay = 100;
        const maxDelay = 30000;

        if (times > maxRetries) {
            return null;
        }

        let delay = Math.min(baseDelay * Math.pow(2, times - 1), maxDelay);
        const jitter = delay * 0.25 * (Math.random() - 0.5);
        return Math.floor(delay + jitter);
    }

    reconnectOnError(err) {
        const retriableErrors = ['READONLY', 'ECONNRESET', 'ETIMEDOUT'];
        return retriableErrors.some(e => err.message.includes(e));
    }

    /**
     * Exécute une commande avec gestion d'erreurs
     */
    async executeCommand(command, ...args) {
        // Vérifier le circuit breaker
        if (this.stats.circuitBreakerOpen) {
            throw new Error('Circuit breaker is open, request rejected');
        }

        try {
            const result = await this.redis[command](...args);

            // Reset sur succès
            this.stats.successCount++;
            this.stats.consecutiveErrors = 0;

            return result;
        } catch (err) {
            this.stats.errorCount++;
            this.stats.consecutiveErrors++;

            console.error(
                `❌ Command ${command} failed: ${err.message} ` +
                `(consecutive errors: ${this.stats.consecutiveErrors})`
            );

            throw err;
        }
    }

    /**
     * GET avec fallback
     */
    async getWithFallback(key, fallbackFn) {
        try {
            const value = await this.executeCommand('get', key);
            return value;
        } catch (err) {
            console.warn(`⚠️ GET failed for key '${key}', using fallback`);

            if (fallbackFn) {
                return await fallbackFn();
            }

            return null;
        }
    }

    /**
     * SET sans exception
     */
    async setSafe(key, value, ex = null) {
        try {
            if (ex) {
                await this.executeCommand('set', key, value, 'EX', ex);
            } else {
                await this.executeCommand('set', key, value);
            }
            return true;
        } catch (err) {
            console.error(`❌ Failed to set key '${key}': ${err.message}`);
            return false;
        }
    }

    /**
     * Statistiques
     */
    getStats() {
        return {
            ...this.stats,
            successRate: this.stats.successCount /
                        (this.stats.successCount + this.stats.errorCount) * 100,
        };
    }

    /**
     * Health check
     */
    async healthCheck() {
        try {
            const start = Date.now();
            await this.redis.ping();
            const latency = Date.now() - start;

            return {
                healthy: !this.stats.circuitBreakerOpen,
                latency: `${latency}ms`,
                stats: this.getStats(),
            };
        } catch (err) {
            return {
                healthy: false,
                error: err.message,
                stats: this.getStats(),
            };
        }
    }

    async close() {
        await this.redis.quit();
    }
}

// Utilisation
const client = new ResilientRedisClient({
    host: 'localhost',
    port: 6379,
    circuitBreakerThreshold: 5,
});

// Event listeners
client.on('circuit-breaker-open', (stats) => {
    console.error('🚨 ALERTE : Circuit breaker ouvert !', stats);
    // Envoyer alerte à monitoring system
});

client.on('circuit-breaker-closed', () => {
    console.log('✅ Circuit breaker fermé, service restauré');
});

// Utilisation avec fallback
const userId = 123;
const user = await client.getWithFallback(
    `user:${userId}`,
    async () => {
        // Fallback : récupérer depuis DB
        console.log('📦 Fetching from database...');
        return await db.users.findOne({ id: userId });
    }
);

// SET sécurisé
const saved = await client.setSafe('session:abc', 'data', 3600);
if (!saved) {
    console.warn('Session not saved to cache, continuing...');
}

// Health check
const health = await client.healthCheck();
console.log('Health:', health);

export default client;
```

---

### Go (go-redis)

#### Configuration avec retry et backoff

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "math"
    "math/rand"
    "time"

    "github.com/redis/go-redis/v9"
)

// RetryConfig configuration des tentatives
type RetryConfig struct {
    MaxAttempts    int
    BaseDelay      time.Duration
    MaxDelay       time.Duration
    ExponentialBase float64
    Jitter         bool
}

// DefaultRetryConfig retourne une config par défaut
func DefaultRetryConfig() RetryConfig {
    return RetryConfig{
        MaxAttempts:     5,
        BaseDelay:       100 * time.Millisecond,
        MaxDelay:        30 * time.Second,
        ExponentialBase: 2.0,
        Jitter:          true,
    }
}

// CalculateBackoff calcule le délai avec exponential backoff et jitter
func (rc RetryConfig) CalculateBackoff(attempt int) time.Duration {
    // Exponential : baseDelay * (exponentialBase ^ attempt)
    expDelay := float64(rc.BaseDelay) * math.Pow(rc.ExponentialBase, float64(attempt))

    // Plafonner au max
    delay := time.Duration(math.Min(expDelay, float64(rc.MaxDelay)))

    // Ajouter jitter si activé
    if rc.Jitter {
        jitterFactor := 0.5 + rand.Float64()*0.5 // 0.5 à 1.0
        delay = time.Duration(float64(delay) * jitterFactor)
    }

    return delay
}

// IsRetriableError détermine si une erreur justifie un retry
func IsRetriableError(err error) bool {
    if err == nil {
        return false
    }

    // Erreurs réseau : retry
    if errors.Is(err, context.DeadlineExceeded) {
        return true
    }

    // Redis specific errors
    redisErr, ok := err.(redis.Error)
    if !ok {
        return false
    }

    retriableMessages := []string{
        "LOADING",      // Redis en cours de démarrage
        "READONLY",     // Écriture sur replica
        "BUSY",         // Script Lua en cours
        "TRYAGAIN",     // Cluster resharding
        "CLUSTERDOWN",  // Cluster temporairement down
    }

    for _, msg := range retriableMessages {
        if redisErr.Error() == msg ||
           len(redisErr.Error()) > len(msg) && redisErr.Error()[:len(msg)] == msg {
            return true
        }
    }

    return false
}

// RetryWithBackoff exécute une fonction avec retry
func RetryWithBackoff(
    ctx context.Context,
    config RetryConfig,
    operation func() error,
) error {
    var lastErr error

    for attempt := 0; attempt < config.MaxAttempts; attempt++ {
        // Exécuter l'opération
        err := operation()

        if err == nil {
            // Succès
            if attempt > 0 {
                log.Printf("✅ Operation succeeded after %d attempts", attempt+1)
            }
            return nil
        }

        lastErr = err

        // Vérifier si l'erreur justifie un retry
        if !IsRetriableError(err) {
            log.Printf("❌ Non-retriable error: %v", err)
            return err
        }

        // Dernière tentative ?
        if attempt == config.MaxAttempts-1 {
            log.Printf("❌ All %d attempts failed: %v", config.MaxAttempts, err)
            break
        }

        // Calculer le délai
        backoff := config.CalculateBackoff(attempt)

        log.Printf(
            "⚠️ Attempt %d/%d failed: %v. Retrying in %v",
            attempt+1, config.MaxAttempts, err, backoff,
        )

        // Attendre avant le prochain retry
        select {
        case <-time.After(backoff):
            // Continue
        case <-ctx.Done():
            return ctx.Err()
        }
    }

    return fmt.Errorf("operation failed after %d attempts: %w",
                      config.MaxAttempts, lastErr)
}

// ResilientClient client Redis avec retry
type ResilientClient struct {
    client      *redis.Client
    retryConfig RetryConfig
    stats       *ClientStats
}

// ClientStats statistiques du client
type ClientStats struct {
    TotalCommands      int64
    SuccessCount       int64
    ErrorCount         int64
    ConsecutiveErrors  int
    CircuitBreakerOpen bool
    LastError          error
}

// NewResilientClient crée un client résilient
func NewResilientClient(rdb *redis.Client, config RetryConfig) *ResilientClient {
    return &ResilientClient{
        client:      rdb,
        retryConfig: config,
        stats:       &ClientStats{},
    }
}

// Get récupère une valeur avec retry
func (rc *ResilientClient) Get(ctx context.Context, key string) (string, error) {
    rc.stats.TotalCommands++

    var result string
    err := RetryWithBackoff(ctx, rc.retryConfig, func() error {
        var err error
        result, err = rc.client.Get(ctx, key).Result()
        return err
    })

    rc.updateStats(err)
    return result, err
}

// Set stocke une valeur avec retry
func (rc *ResilientClient) Set(
    ctx context.Context,
    key string,
    value interface{},
    expiration time.Duration,
) error {
    rc.stats.TotalCommands++

    err := RetryWithBackoff(ctx, rc.retryConfig, func() error {
        return rc.client.Set(ctx, key, value, expiration).Err()
    })

    rc.updateStats(err)
    return err
}

// GetWithFallback GET avec fonction de fallback
func (rc *ResilientClient) GetWithFallback(
    ctx context.Context,
    key string,
    fallback func() (string, error),
) (string, error) {
    value, err := rc.Get(ctx, key)

    if err == redis.Nil {
        // Clé n'existe pas (pas une erreur)
        return "", nil
    }

    if err != nil {
        log.Printf("⚠️ Redis GET failed for key '%s', using fallback", key)
        if fallback != nil {
            return fallback()
        }
        return "", err
    }

    return value, nil
}

// SetSafe SET sans lever d'exception
func (rc *ResilientClient) SetSafe(
    ctx context.Context,
    key string,
    value interface{},
    expiration time.Duration,
) bool {
    err := rc.Set(ctx, key, value, expiration)
    if err != nil {
        log.Printf("❌ Failed to set key '%s': %v", key, err)
        return false
    }
    return true
}

// updateStats met à jour les statistiques
func (rc *ResilientClient) updateStats(err error) {
    if err == nil {
        rc.stats.SuccessCount++
        rc.stats.ConsecutiveErrors = 0
    } else {
        rc.stats.ErrorCount++
        rc.stats.ConsecutiveErrors++
        rc.stats.LastError = err

        // Circuit breaker simple
        if rc.stats.ConsecutiveErrors >= 5 {
            if !rc.stats.CircuitBreakerOpen {
                log.Printf("🔥 Circuit breaker OPEN after %d errors",
                          rc.stats.ConsecutiveErrors)
                rc.stats.CircuitBreakerOpen = true
            }
        }
    }
}

// GetStats retourne les statistiques
func (rc *ResilientClient) GetStats() ClientStats {
    return *rc.stats
}

// HealthCheck vérifie la santé
func (rc *ResilientClient) HealthCheck(ctx context.Context) (bool, error) {
    start := time.Now()
    err := rc.client.Ping(ctx).Err()
    latency := time.Since(start)

    if err != nil {
        return false, fmt.Errorf("health check failed (%v): %w", latency, err)
    }

    log.Printf("✅ Health check OK (latency: %v)", latency)
    return true, nil
}

func main() {
    ctx := context.Background()

    // Client Redis standard
    rdb := redis.NewClient(&redis.Options{
        Addr:         "localhost:6379",
        MaxRetries:   3,
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
    })
    defer rdb.Close()

    // Créer le client résilient
    config := DefaultRetryConfig()
    config.MaxAttempts = 5

    resilient := NewResilientClient(rdb, config)

    // Utilisation avec retry automatique
    err := resilient.Set(ctx, "user:123", "Alice", time.Hour)
    if err != nil {
        log.Printf("Failed to set: %v", err)
    }

    value, err := resilient.Get(ctx, "user:123")
    if err != nil {
        log.Printf("Failed to get: %v", err)
    } else {
        fmt.Printf("Value: %s\n", value)
    }

    // GET avec fallback
    user, err := resilient.GetWithFallback(ctx, "user:999", func() (string, error) {
        log.Println("📦 Fetching from database...")
        return "Default User", nil
    })
    fmt.Printf("User: %s\n", user)

    // SET sans exception
    saved := resilient.SetSafe(ctx, "session:abc", "data", 30*time.Minute)
    if !saved {
        log.Println("⚠️ Session not cached, continuing...")
    }

    // Stats
    stats := resilient.GetStats()
    fmt.Printf("\n=== Stats ===\n")
    fmt.Printf("Total commands: %d\n", stats.TotalCommands)
    fmt.Printf("Success: %d\n", stats.SuccessCount)
    fmt.Printf("Errors: %d\n", stats.ErrorCount)
    fmt.Printf("Circuit breaker: %v\n", stats.CircuitBreakerOpen)

    // Health check
    healthy, err := resilient.HealthCheck(ctx)
    if !healthy {
        log.Printf("❌ Unhealthy: %v", err)
    }
}
```

---

## Circuit Breaker Pattern

Le **Circuit Breaker** protège votre application contre les cascades de défaillances en "coupant le circuit" après un certain nombre d'erreurs.

### États du Circuit Breaker

```
┌─────────────────────────────────────────────────────┐
│                 Circuit Breaker                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│    ┌──────────┐                                     │
│    │  CLOSED  │  ◄─ Fonctionnement normal           │
│    └────┬─────┘                                     │
│         │ Erreurs >= threshold                      │
│         ▼                                           │
│    ┌──────────┐                                     │
│    │   OPEN   │  ◄─ Toutes requêtes rejetées        │
│    └────┬─────┘                                     │
│         │ Après timeout                             │
│         ▼                                           │
│    ┌──────────┐                                     │
│    │ HALF-OPEN│  ◄─ Test de récupération            │
│    └────┬─────┘                                     │
│         │ Succès → CLOSED                           │
│         │ Échec → OPEN                              │
│         └──────────────────────────┐                │
│                                    │                │
└────────────────────────────────────┼────────────────┘
                                     │
                              (retour à CLOSED)
```

### Implémentation simple (Node.js)

```javascript
class CircuitBreaker {
    constructor(threshold = 5, timeout = 60000) {
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.threshold = threshold;
        this.timeout = timeout;
        this.nextAttempt = Date.now();
    }

    async execute(operation) {
        if (this.state === 'OPEN') {
            if (Date.now() < this.nextAttempt) {
                throw new Error('Circuit breaker is OPEN');
            }
            // Passer en HALF-OPEN pour tester
            this.state = 'HALF-OPEN';
        }

        try {
            const result = await operation();
            this.onSuccess();
            return result;
        } catch (err) {
            this.onFailure();
            throw err;
        }
    }

    onSuccess() {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }

    onFailure() {
        this.failureCount++;

        if (this.failureCount >= this.threshold) {
            this.state = 'OPEN';
            this.nextAttempt = Date.now() + this.timeout;
            console.error(
                `🔥 Circuit breaker OPEN for ${this.timeout}ms`
            );
        }
    }

    getState() {
        return {
            state: this.state,
            failureCount: this.failureCount,
            threshold: this.threshold,
        };
    }
}

// Utilisation
const breaker = new CircuitBreaker(5, 30000); // 5 erreurs, 30s timeout

async function getUser(userId) {
    return breaker.execute(async () => {
        return await redis.get(`user:${userId}`);
    });
}

try {
    const user = await getUser(123);
} catch (err) {
    if (err.message === 'Circuit breaker is OPEN') {
        console.log('Service dégradé, utiliser fallback');
    }
}
```

---

## Logging et monitoring des erreurs

### Structure de log recommandée

```python
import logging
import json
from datetime import datetime

class RedisErrorLogger:
    """Logger structuré pour erreurs Redis"""

    def __init__(self):
        self.logger = logging.getLogger('redis.errors')

    def log_error(self, error_type, operation, key=None, details=None):
        """Log une erreur avec contexte complet"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'error_type': error_type,
            'operation': operation,
            'key': key,
            'details': details or {},
        }

        self.logger.error(json.dumps(log_entry))

    def log_retry(self, attempt, max_attempts, delay, error):
        """Log une tentative de retry"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'event': 'retry',
            'attempt': attempt,
            'max_attempts': max_attempts,
            'delay_seconds': delay,
            'error': str(error),
        }

        self.logger.warning(json.dumps(log_entry))

# Utilisation
error_logger = RedisErrorLogger()

try:
    redis_client.get('user:123')
except redis.TimeoutError as e:
    error_logger.log_error(
        'TimeoutError',
        'GET',
        key='user:123',
        details={'timeout': 5, 'error': str(e)}
    )
```

### Métriques à monitorer

```javascript
// Métriques Prometheus-style
const metrics = {
    redis_operations_total: 0,
    redis_errors_total: 0,
    redis_retries_total: 0,
    redis_circuit_breaker_open: 0,
    redis_latency_ms: [],
};

function recordOperation(success, latency, retries = 0) {
    metrics.redis_operations_total++;

    if (!success) {
        metrics.redis_errors_total++;
    }

    if (retries > 0) {
        metrics.redis_retries_total += retries;
    }

    metrics.redis_latency_ms.push(latency);
}

// Exporter pour Prometheus/Datadog/etc.
function exportMetrics() {
    const avgLatency = metrics.redis_latency_ms.reduce((a, b) => a + b, 0) /
                      metrics.redis_latency_ms.length;

    return {
        ...metrics,
        redis_error_rate: metrics.redis_errors_total / metrics.redis_operations_total,
        redis_avg_latency_ms: avgLatency,
    };
}
```

---

## Bonnes pratiques

### ✅ 1. Distinguer erreurs temporaires vs permanentes

```python
# ✅ Retry sur erreurs temporaires uniquement
RETRIABLE_ERRORS = (
    redis.ConnectionError,
    redis.TimeoutError,
    redis.BusyLoadingError,
)

NON_RETRIABLE_ERRORS = (
    redis.ResponseError,  # Commande invalide
    redis.DataError,      # Mauvais type de données
)

try:
    result = redis_client.get('key')
except RETRIABLE_ERRORS as e:
    # Retry avec backoff
    pass
except NON_RETRIABLE_ERRORS as e:
    # Logger et corriger le code
    raise
```

### ✅ 2. Limiter le nombre de retries

```javascript
// ✅ Maximum raisonnable de retries
const MAX_RETRIES = 5; // Pas 100 !

// Temps total maximum
const totalTime = Array.from({length: MAX_RETRIES}, (_, i) =>
    100 * Math.pow(2, i)
).reduce((a, b) => a + b);
// ~3.1 secondes pour 5 retries
```

### ✅ 3. Utiliser des timeouts contextuels

```go
// ✅ Timeout adapté au contexte
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

result, err := resilientClient.Get(ctx, "key")
```

### ✅ 4. Logger avec niveau approprié

```python
# ✅ Niveaux de log appropriés
logger.debug("Tentative de connexion Redis")
logger.info("Retry réussi après 3 tentatives")
logger.warning("Timeout Redis, retry dans 2s")
logger.error("Échec après tous les retries")
logger.critical("Circuit breaker ouvert, service dégradé")
```

### ✅ 5. Implémenter des fallbacks

```javascript
// ✅ Toujours avoir un plan B
async function getUser(userId) {
    try {
        return await redis.get(`user:${userId}`);
    } catch (err) {
        logger.warn('Redis failed, falling back to database');
        return await database.users.findOne({ id: userId });
    }
}
```

### ✅ 6. Monitorer et alerter

```python
# ✅ Alertes sur seuils critiques
if error_rate > 0.05:  # > 5% d'erreurs
    send_alert('Redis error rate elevated', severity='warning')

if circuit_breaker_open:
    send_alert('Redis circuit breaker open', severity='critical')
```

---

## Anti-patterns à éviter

### ❌ 1. Retry infini

```python
# ❌ DANGEREUX : Peut bloquer indéfiniment
while True:
    try:
        return redis_client.get('key')
    except:
        time.sleep(1)
```

### ❌ 2. Retry sans backoff

```javascript
// ❌ Surcharge Redis en cas de problème
for (let i = 0; i < 10; i++) {
    try {
        return await redis.get('key');
    } catch (err) {
        // Retry immédiatement !
    }
}
```

### ❌ 3. Ignorer les erreurs silencieusement

```go
// ❌ Erreur ignorée sans log
result, _ := rdb.Get(ctx, "key").Result()
// Si erreur, result est vide et on ne sait pas pourquoi !
```

### ❌ 4. Retry sur toutes les erreurs

```python
# ❌ Retry même sur erreurs non récupérables
try:
    redis_client.get('key')
except Exception:  # Trop large !
    retry()
```

---

## Checklist de gestion d'erreurs

Avant de déployer en production :

- ✅ **Retry configuré** avec exponential backoff et jitter
- ✅ **Limite de retries** raisonnable (3-5 max)
- ✅ **Timeouts définis** pour chaque opération
- ✅ **Erreurs classifiées** (retriable vs non-retriable)
- ✅ **Logging structuré** avec contexte complet
- ✅ **Circuit breaker** implémenté si critique
- ✅ **Fallback strategy** définie
- ✅ **Métriques exportées** (error rate, retry count)
- ✅ **Alertes configurées** sur seuils critiques
- ✅ **Tests de résilience** effectués (chaos engineering)

---

## Points clés à retenir

🔑 **Toujours utiliser exponential backoff** avec jitter pour éviter thundering herd

🔑 **Distinguer erreurs temporaires** (retry) vs permanentes (fix code)

🔑 **Limiter les retries** : 3-5 tentatives maximum

🔑 **Implémenter circuit breaker** pour protéger contre cascades de défaillances

🔑 **Logger avec contexte** : operation, key, error type, attempt number

🔑 **Monitorer les métriques** : error rate, retry rate, circuit breaker state

🔑 **Avoir un fallback** : database, cache local, valeur par défaut

🔑 **Tester la résilience** : simuler pannes réseau, timeouts, Redis down

---

## Ressources complémentaires

- **AWS retry guidelines** : https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
- **Circuit breaker pattern** : https://martinfowler.com/bliki/CircuitBreaker.html
- **Redis error handling** : https://redis.io/docs/manual/patterns/error-handling/

---

## Prochaine section

➡️ **Section 9.4** : Async/Await et programmation réactive - Maximiser les performances

**Niveau** : Intermédiaire
**Durée estimée** : 60 minutes
**Prérequis** : Sections 9.1-9.3

⏭️ [Async/Await et programmation réactive](/09-integration-langages-programmation/04-async-await-programmation-reactive.md)
