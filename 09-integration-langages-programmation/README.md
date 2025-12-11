🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 9 : Intégration avec les langages de programmation

## Vue d'ensemble du module

L'intégration de Redis dans vos applications est l'étape qui transforme la théorie en pratique. Ce module vous guidera à travers les meilleures pratiques pour interagir avec Redis depuis vos applications, quel que soit le langage que vous utilisez.

## Objectifs d'apprentissage

À la fin de ce module, vous serez capable de :

- ✅ Choisir et utiliser le client Redis adapté à votre langage
- ✅ Gérer efficacement les connexions avec les pools
- ✅ Implémenter une gestion d'erreurs robuste avec retry logic
- ✅ Utiliser les patterns asynchrones pour maximiser les performances
- ✅ Appliquer les bonnes pratiques de développement avec Redis
- ✅ Tester et mocker Redis dans vos tests unitaires

## Prérequis

- Connaissance intermédiaire d'au moins un langage de programmation
- Compréhension des structures de données Redis (Modules 1-2)
- Notions de base sur les architectures client-serveur
- Familiarité avec les concepts asynchrones (async/await, promises, callbacks)

---

## Introduction : Redis côté application

### Le principe fondamental

Redis est une base de données **réseau** : votre application communique avec le serveur Redis via le protocole TCP. Cette communication se fait à travers un **client Redis**, une bibliothèque qui encapsule le protocole RESP (REdis Serialization Protocol) et expose des méthodes simples à utiliser.

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   Application   │ ◄─────► │  Client Redis    │ ◄─────► │  Serveur Redis  │
│   (votre code)  │   API   │  (bibliothèque)  │  RESP   │   (redis-server)│
└─────────────────┘         └──────────────────┘         └─────────────────┘
```

### Les défis de l'intégration

Intégrer Redis dans une application production nécessite de gérer plusieurs aspects critiques :

1. **Performance** : Minimiser la latence réseau
2. **Fiabilité** : Gérer les connexions perdues et les timeouts
3. **Scalabilité** : Utiliser les pools de connexions
4. **Maintenabilité** : Écrire du code propre et testable
5. **Sécurité** : Gérer les credentials et les erreurs

---

## Concepts clés transversaux

### 1. Le protocole RESP (REdis Serialization Protocol)

Redis utilise un protocole texte simple et lisible. Bien que vous n'ayez pas à l'implémenter vous-même, comprendre ses bases aide à diagnostiquer les problèmes.

**Exemple de communication RESP :**

```
Client → Serveur : *3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
Serveur → Client : +OK\r\n
```

Les clients Redis abstraient ce protocole :

```python
# Python (redis-py)
redis.set("key", "value")  # → Génère la commande RESP
```

```javascript
// Node.js (ioredis)
await redis.set("key", "value");  // → Génère la commande RESP
```

### 2. Connexions synchrones vs asynchrones

**Synchrone (bloquant) :**
```python
# Python - synchrone
import redis
client = redis.Redis(host='localhost', port=6379)
value = client.get('user:123')  # Bloque jusqu'à la réponse
print(value)
```

**Asynchrone (non-bloquant) :**
```javascript
// Node.js - asynchrone
import Redis from 'ioredis';
const client = new Redis();
const value = await client.get('user:123');  // Non-bloquant
console.log(value);
```

**Pourquoi l'asynchrone ?**
- Permet de traiter d'autres requêtes pendant l'attente réseau
- Crucial pour les serveurs web à haute concurrence
- Réduit l'utilisation des threads/processus

### 3. Le pool de connexions

Une connexion TCP a un coût de création (handshake, authentification). Le pool réutilise les connexions :

```
┌───────────────────────────────────────┐
│         Pool de connexions            │
│                                       │
│  ┌────────┐  ┌────────┐  ┌────────┐   │
│  │ Conn 1 │  │ Conn 2 │  │ Conn 3 │   │
│  └────────┘  └────────┘  └────────┘   │
│     ▲            ▲            ▲       │
└─────┼────────────┼────────────┼───────┘
      │            │            │
   Thread 1    Thread 2     Thread 3
```

**Configuration type d'un pool :**

```python
# Python
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,      # Maximum de connexions
    socket_connect_timeout=2,
    socket_timeout=2,
    retry_on_timeout=True
)
client = redis.Redis(connection_pool=pool)
```

```javascript
// Node.js (ioredis)
const redis = new Redis({
    host: 'localhost',
    port: 6379,
    maxRetriesPerRequest: 3,
    enableReadyCheck: true,
    lazyConnect: false
});
```

```go
// Go (go-redis)
client := redis.NewClient(&redis.Options{
    Addr:         "localhost:6379",
    PoolSize:     50,
    MinIdleConns: 10,
    MaxRetries:   3,
})
```

### 4. Gestion des erreurs courantes

Tous les clients doivent gérer ces scénarios :

| Erreur | Cause | Solution |
|--------|-------|----------|
| `Connection refused` | Redis n'est pas démarré | Vérifier le service Redis |
| `Connection timeout` | Réseau lent/congestion | Augmenter le timeout |
| `NOAUTH` | Authentification manquante | Configurer le password |
| `READONLY` | Écriture sur replica | Rediriger vers le master |
| `OOM` | Mémoire pleine | Vérifier maxmemory et évictions |

---

## Exemples pratiques multi-langages

### Exemple 1 : Connexion basique et opérations CRUD

#### Python (redis-py)

```python
import redis
from redis.exceptions import RedisError, ConnectionError, TimeoutError

class RedisClient:
    def __init__(self, host='localhost', port=6379, password=None):
        """Initialise un client Redis avec bonnes pratiques"""
        try:
            self.client = redis.Redis(
                host=host,
                port=port,
                password=password,
                decode_responses=True,  # Retourne des strings, pas des bytes
                socket_connect_timeout=5,
                socket_timeout=5,
                retry_on_timeout=True,
                health_check_interval=30  # Vérifie la connexion tous les 30s
            )
            # Test de connexion
            self.client.ping()
            print("✅ Connexion Redis établie")
        except ConnectionError as e:
            print(f"❌ Impossible de se connecter à Redis: {e}")
            raise

    def set_user(self, user_id: int, user_data: dict, ttl: int = 3600):
        """Stocke un utilisateur avec TTL"""
        try:
            key = f"user:{user_id}"
            # Sérialisation JSON (alternative à hash)
            import json
            return self.client.setex(
                key,
                ttl,
                json.dumps(user_data)
            )
        except RedisError as e:
            print(f"Erreur lors du SET: {e}")
            return False

    def get_user(self, user_id: int):
        """Récupère un utilisateur"""
        try:
            key = f"user:{user_id}"
            data = self.client.get(key)
            if data:
                import json
                return json.loads(data)
            return None
        except RedisError as e:
            print(f"Erreur lors du GET: {e}")
            return None

    def close(self):
        """Ferme proprement la connexion"""
        self.client.close()

# Utilisation
if __name__ == "__main__":
    rc = RedisClient()

    # Création
    rc.set_user(123, {"name": "Alice", "email": "alice@example.com"})

    # Lecture
    user = rc.get_user(123)
    print(f"Utilisateur: {user}")

    # Nettoyage
    rc.close()
```

#### Node.js (ioredis)

```javascript
import Redis from 'ioredis';

class RedisClient {
    constructor(options = {}) {
        const defaultOptions = {
            host: options.host || 'localhost',
            port: options.port || 6379,
            password: options.password || null,
            retryStrategy: (times) => {
                // Reconnexion exponentielle avec maximum
                const delay = Math.min(times * 50, 2000);
                return delay;
            },
            maxRetriesPerRequest: 3,
            enableReadyCheck: true,
            lazyConnect: false,
            showFriendlyErrorStack: process.env.NODE_ENV === 'development'
        };

        this.client = new Redis(defaultOptions);

        // Gestion des événements
        this.client.on('connect', () => {
            console.log('✅ Connexion Redis établie');
        });

        this.client.on('error', (err) => {
            console.error('❌ Erreur Redis:', err);
        });

        this.client.on('close', () => {
            console.log('🔌 Connexion Redis fermée');
        });
    }

    /**
     * Stocke un utilisateur avec TTL
     * @param {number} userId - ID de l'utilisateur
     * @param {Object} userData - Données utilisateur
     * @param {number} ttl - TTL en secondes
     */
    async setUser(userId, userData, ttl = 3600) {
        try {
            const key = `user:${userId}`;
            const value = JSON.stringify(userData);

            // EX = expiration en secondes
            await this.client.set(key, value, 'EX', ttl);
            return true;
        } catch (err) {
            console.error('Erreur lors du SET:', err);
            return false;
        }
    }

    /**
     * Récupère un utilisateur
     * @param {number} userId - ID de l'utilisateur
     * @returns {Object|null} Données utilisateur ou null
     */
    async getUser(userId) {
        try {
            const key = `user:${userId}`;
            const data = await this.client.get(key);

            if (data) {
                return JSON.parse(data);
            }
            return null;
        } catch (err) {
            console.error('Erreur lors du GET:', err);
            return null;
        }
    }

    /**
     * Ferme proprement la connexion
     */
    async close() {
        await this.client.quit();
    }
}

// Utilisation avec async/await
(async () => {
    const rc = new RedisClient();

    // Création
    await rc.setUser(123, { name: 'Alice', email: 'alice@example.com' });

    // Lecture
    const user = await rc.getUser(123);
    console.log('Utilisateur:', user);

    // Nettoyage
    await rc.close();
})();
```

#### Go (go-redis)

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

type RedisClient struct {
    client *redis.Client
    ctx    context.Context
}

// NewRedisClient initialise un client Redis avec bonnes pratiques
func NewRedisClient(addr string, password string) (*RedisClient, error) {
    client := redis.NewClient(&redis.Options{
        Addr:         addr,
        Password:     password,
        DB:           0,
        PoolSize:     50,              // Pool de connexions
        MinIdleConns: 10,              // Connexions idle minimum
        MaxRetries:   3,               // Tentatives de retry
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
    })

    ctx := context.Background()

    // Test de connexion
    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("impossible de se connecter à Redis: %w", err)
    }

    fmt.Println("✅ Connexion Redis établie")
    return &RedisClient{client: client, ctx: ctx}, nil
}

// SetUser stocke un utilisateur avec TTL
func (rc *RedisClient) SetUser(userID int, userData User, ttl time.Duration) error {
    key := fmt.Sprintf("user:%d", userID)

    // Sérialisation JSON
    jsonData, err := json.Marshal(userData)
    if err != nil {
        return fmt.Errorf("erreur de sérialisation: %w", err)
    }

    // SET avec expiration
    err = rc.client.Set(rc.ctx, key, jsonData, ttl).Err()
    if err != nil {
        return fmt.Errorf("erreur lors du SET: %w", err)
    }

    return nil
}

// GetUser récupère un utilisateur
func (rc *RedisClient) GetUser(userID int) (*User, error) {
    key := fmt.Sprintf("user:%d", userID)

    // GET
    data, err := rc.client.Get(rc.ctx, key).Result()
    if err == redis.Nil {
        return nil, nil // Clé n'existe pas
    } else if err != nil {
        return nil, fmt.Errorf("erreur lors du GET: %w", err)
    }

    // Désérialisation JSON
    var user User
    if err := json.Unmarshal([]byte(data), &user); err != nil {
        return nil, fmt.Errorf("erreur de désérialisation: %w", err)
    }

    return &user, nil
}

// Close ferme proprement la connexion
func (rc *RedisClient) Close() error {
    return rc.client.Close()
}

func main() {
    // Initialisation
    rc, err := NewRedisClient("localhost:6379", "")
    if err != nil {
        panic(err)
    }
    defer rc.Close()

    // Création
    user := User{Name: "Alice", Email: "alice@example.com"}
    if err := rc.SetUser(123, user, time.Hour); err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    // Lecture
    retrievedUser, err := rc.GetUser(123)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    if retrievedUser != nil {
        fmt.Printf("Utilisateur: %+v\n", retrievedUser)
    } else {
        fmt.Println("Utilisateur non trouvé")
    }
}
```

---

### Exemple 2 : Pattern Cache-Aside avec gestion d'erreurs

#### Python

```python
import redis
import time
from functools import wraps
from typing import Optional, Callable, Any

def cache_aside(ttl: int = 3600, key_prefix: str = "cache"):
    """
    Décorateur implémentant le pattern Cache-Aside

    Usage:
        @cache_aside(ttl=600, key_prefix="product")
        def get_product(product_id: int):
            # Logique coûteuse (DB, API, etc.)
            return fetch_from_database(product_id)
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Construction de la clé de cache
            cache_key = f"{key_prefix}:{func.__name__}:{args}:{kwargs}"

            try:
                # 1. Tentative de lecture dans le cache
                cached_value = redis_client.get(cache_key)
                if cached_value:
                    print(f"🎯 Cache HIT: {cache_key}")
                    import json
                    return json.loads(cached_value)

                print(f"❌ Cache MISS: {cache_key}")

            except redis.RedisError as e:
                print(f"⚠️ Erreur cache (lecture): {e}")
                # Continue sans cache en cas d'erreur

            # 2. Exécution de la fonction originale
            result = func(*args, **kwargs)

            # 3. Stockage dans le cache
            try:
                import json
                redis_client.setex(
                    cache_key,
                    ttl,
                    json.dumps(result)
                )
                print(f"💾 Valeur mise en cache: {cache_key}")
            except redis.RedisError as e:
                print(f"⚠️ Erreur cache (écriture): {e}")
                # Continue sans mettre en cache

            return result

        return wrapper
    return decorator

# Exemple d'utilisation
redis_client = redis.Redis(decode_responses=True)

@cache_aside(ttl=600, key_prefix="product")
def get_product_from_db(product_id: int) -> dict:
    """Simule une requête base de données coûteuse"""
    print(f"🔍 Fetching product {product_id} from database...")
    time.sleep(0.5)  # Simule latence DB
    return {
        "id": product_id,
        "name": f"Product {product_id}",
        "price": 99.99
    }

# Premier appel : Cache MISS → Requête DB
product = get_product_from_db(123)
print(product)

# Second appel : Cache HIT → Pas de requête DB
product = get_product_from_db(123)
print(product)
```

#### Node.js

```javascript
import Redis from 'ioredis';

const redis = new Redis();

/**
 * Implémente le pattern Cache-Aside
 * @param {Function} fetchFunction - Fonction pour récupérer la donnée
 * @param {string} cacheKey - Clé Redis
 * @param {number} ttl - TTL en secondes
 */
async function cacheAside(fetchFunction, cacheKey, ttl = 3600) {
    try {
        // 1. Tentative de lecture dans le cache
        const cachedValue = await redis.get(cacheKey);

        if (cachedValue) {
            console.log(`🎯 Cache HIT: ${cacheKey}`);
            return JSON.parse(cachedValue);
        }

        console.log(`❌ Cache MISS: ${cacheKey}`);

    } catch (err) {
        console.warn(`⚠️ Erreur cache (lecture): ${err.message}`);
        // Continue sans cache en cas d'erreur
    }

    // 2. Exécution de la fonction de fetch
    const result = await fetchFunction();

    // 3. Stockage dans le cache
    try {
        await redis.set(cacheKey, JSON.stringify(result), 'EX', ttl);
        console.log(`💾 Valeur mise en cache: ${cacheKey}`);
    } catch (err) {
        console.warn(`⚠️ Erreur cache (écriture): ${err.message}`);
        // Continue sans mettre en cache
    }

    return result;
}

// Fonction simulant une requête DB coûteuse
async function getProductFromDB(productId) {
    console.log(`🔍 Fetching product ${productId} from database...`);
    await new Promise(resolve => setTimeout(resolve, 500)); // Simule latence
    return {
        id: productId,
        name: `Product ${productId}`,
        price: 99.99
    };
}

// Utilisation
(async () => {
    const productId = 123;
    const cacheKey = `product:${productId}`;

    // Premier appel : Cache MISS
    let product = await cacheAside(
        () => getProductFromDB(productId),
        cacheKey,
        600
    );
    console.log('Product:', product);

    // Second appel : Cache HIT
    product = await cacheAside(
        () => getProductFromDB(productId),
        cacheKey,
        600
    );
    console.log('Product:', product);

    await redis.quit();
})();
```

---

## Bonnes pratiques universelles

### 1. ✅ Toujours utiliser un pool de connexions

```python
# ❌ MAUVAIS : Une connexion par requête
def bad_practice():
    client = redis.Redis()
    value = client.get('key')
    client.close()

# ✅ BON : Réutilisation via pool
pool = redis.ConnectionPool()
client = redis.Redis(connection_pool=pool)
```

### 2. ✅ Gérer les timeouts et retries

```javascript
// ✅ Configuration avec timeout et retry
const redis = new Redis({
    retryStrategy: (times) => Math.min(times * 50, 2000),
    maxRetriesPerRequest: 3,
    enableOfflineQueue: false  // Évite accumulation de commandes
});
```

### 3. ✅ Utiliser des clés structurées

```python
# ❌ MAUVAIS
redis.set("123", data)
redis.set("user", data)

# ✅ BON : Namespace clair
redis.set("user:123:profile", data)
redis.set("user:123:preferences", data)
redis.set("session:abc123", data)
```

### 4. ✅ Sérialiser les données complexes

```javascript
// ✅ JSON pour les objets
const user = { name: 'Alice', age: 30 };
await redis.set('user:123', JSON.stringify(user));

// ✅ Hash pour les structures plates
await redis.hset('user:123', 'name', 'Alice', 'age', '30');
```

### 5. ✅ Définir des TTL par défaut

```python
# ✅ Toujours avec TTL pour éviter les fuites mémoire
redis.setex('cache:data', 3600, value)  # 1 heure

# ❌ Dangereux : pas de TTL
redis.set('cache:data', value)
```

### 6. ✅ Fermer proprement les connexions

```go
// ✅ Utiliser defer pour garantir la fermeture
client := redis.NewClient(options)
defer client.Close()
```

---

## Anti-patterns à éviter

### ❌ 1. Oublier de gérer les erreurs réseau

```python
# ❌ DANGEREUX
def bad():
    return redis.get('key')  # Crash si Redis indisponible

# ✅ CORRECT
def good():
    try:
        return redis.get('key')
    except redis.ConnectionError:
        # Fallback ou log
        return None
```

### ❌ 2. Bloquer avec des commandes lentes

```javascript
// ❌ BLOQUE REDIS
await redis.keys('user:*');  // O(N) - scanne TOUTES les clés

// ✅ UTILISE SCAN
let cursor = '0';
do {
    const [newCursor, keys] = await redis.scan(cursor, 'MATCH', 'user:*', 'COUNT', 100);
    cursor = newCursor;
    // Traite keys...
} while (cursor !== '0');
```

### ❌ 3. Stocker des données trop volumineuses

```python
# ❌ Valeur > 1MB
redis.set('huge:data', big_json)  # Ralentit Redis

# ✅ Compresser ou paginer
import zlib
compressed = zlib.compress(big_json.encode())
redis.set('huge:data', compressed)
```

---

## Architecture de référence

```
┌──────────────────────────────────────────────────────────────┐
│                      Application Layer                       │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐            │
│  │  Service A │   │  Service B │   │  Service C │            │
│  └──────┬─────┘   └──────┬─────┘   └──────┬─────┘            │
│         │                │                │                  │
└─────────┼────────────────┼────────────────┼──────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                    Redis Client Libraries                   │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│  │  redis-py  │   │  ioredis   │   │  go-redis  │           │
│  │ (Python)   │   │ (Node.js)  │   │   (Go)     │           │
│  └──────┬─────┘   └──────┬─────┘   └──────┬─────┘           │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          │                                  │
│                  Connection Pool                            │
│         ┌────────┬───────┼───────┬────────┐                 │
└─────────┼────────┼───────┼───────┼────────┼─────────────────┘
          │        │       │       │        │
          ▼        ▼       ▼       ▼        ▼
     ┌────────────────────────────────────────┐
     │         Redis Server / Cluster         │
     │  ┌──────┐  ┌───────┐  ┌───────┐        │
     │  │Master│  │Replica│  │Replica│        │
     │  └──────┘  └───────┘  └───────┘        │
     └────────────────────────────────────────┘
```

---

## Structure du module

Ce module est organisé en 6 sections progressives :

1. **Clients Redis** : Panorama des bibliothèques disponibles
2. **Connexion et Pools** : Gestion efficace des connexions
3. **Gestion d'erreurs** : Retry logic et résilience
4. **Async/Await** : Programmation asynchrone et réactive
5. **Bonnes pratiques** : Patterns de développement
6. **Testing** : Tests unitaires et mocking

---

## Ressources complémentaires

### Bibliothèques officielles recommandées

- **Python** : `redis-py` - https://github.com/redis/redis-py
- **Node.js** : `ioredis` - https://github.com/redis/ioredis
- **Java** : `Jedis` ou `Lettuce` - https://github.com/redis/jedis
- **Go** : `go-redis` - https://github.com/redis/go-redis
- **PHP** : `phpredis` - https://github.com/phpredis/phpredis
- **.NET** : `StackExchange.Redis` - https://github.com/StackExchange/StackExchange.Redis

### Documentation

- Redis Clients : https://redis.io/docs/clients/
- RESP Protocol : https://redis.io/docs/reference/protocol-spec/
- Best Practices : https://redis.io/docs/manual/patterns/

---

## Points clés à retenir

🔑 **Utilisez toujours un pool de connexions** pour éviter le coût de création répété

🔑 **Gérez les erreurs réseau** avec des timeouts et retry logic appropriés

🔑 **Préférez l'asynchrone** pour les applications à haute concurrence

🔑 **Structurez vos clés** avec des namespaces clairs (user:123:profile)

🔑 **Définissez des TTL** pour éviter les fuites mémoire

🔑 **Testez avec des mocks** pour isoler votre code de Redis

🔑 **Évitez KEYS en production** : utilisez SCAN

🔑 **Sérialisez intelligemment** : JSON pour flexibilité, Hash pour performance

---

## Prochaines étapes

Maintenant que vous comprenez les principes généraux, explorez les sections suivantes pour approfondir chaque aspect :

- ➡️ **Section 9.1** : Clients Redis - Vue d'ensemble détaillée des bibliothèques
- ➡️ **Section 9.2** : Connexion et pool de connexions
- ➡️ **Section 9.3** : Gestion des erreurs et retry logic
- ➡️ **Section 9.4** : Async/Await et programmation réactive
- ➡️ **Section 9.5** : Bonnes pratiques de développement
- ➡️ **Section 9.6** : Testing et mocking Redis

---

**Niveau** : Intermédiaire
**Durée estimée** : 4-6 heures
**Prérequis** : Modules 1-2, connaissance d'un langage de programmation

⏭️ [Clients Redis : Vue d'ensemble (Python, Node.js, Java, Go, PHP, .NET)](/09-integration-langages-programmation/01-clients-redis-vue-ensemble.md)
