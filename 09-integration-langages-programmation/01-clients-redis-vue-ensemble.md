🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1 Clients Redis : Vue d'ensemble

## Introduction

Un **client Redis** est une bibliothèque qui implémente le protocole RESP (Redis Serialization Protocol) et expose une API dans votre langage préféré pour communiquer avec Redis. Choisir le bon client est crucial car il impacte directement les performances, la maintenabilité et la fiabilité de votre application.

Cette section présente les clients Redis les plus populaires et matures pour les langages les plus utilisés en entreprise.

---

## Critères de choix d'un client Redis

Avant de plonger dans les bibliothèques spécifiques, voici les critères à évaluer :

| Critère | Description | Importance |
|---------|-------------|------------|
| **Maturité** | Âge du projet, nombre de contributeurs, fréquence des mises à jour | ⭐⭐⭐⭐⭐ |
| **Performance** | Latence, throughput, gestion du pipelining | ⭐⭐⭐⭐⭐ |
| **Support RESP3** | Compatibilité avec le nouveau protocole Redis 6+ | ⭐⭐⭐⭐ |
| **Fonctionnalités** | Cluster, Sentinel, Streams, Pub/Sub, Client-side caching | ⭐⭐⭐⭐⭐ |
| **Documentation** | Qualité et exhaustivité de la documentation | ⭐⭐⭐⭐⭐ |
| **Support async** | Programmation asynchrone native | ⭐⭐⭐⭐ |
| **Pool de connexions** | Gestion automatique des connexions | ⭐⭐⭐⭐⭐ |
| **Communauté** | Taille, activité, support | ⭐⭐⭐⭐ |

---

## Vue d'ensemble par langage

### 🐍 Python

#### **redis-py** (Recommandé)

**URL** : https://github.com/redis/redis-py
**Installation** : `pip install redis`
**Statut** : Client officiel Redis, le plus utilisé

**Points forts :**
- ✅ Client officiel maintenu par Redis Ltd.
- ✅ API simple et intuitive
- ✅ Support complet : Cluster, Sentinel, Streams, JSON, Search
- ✅ Versions sync et async (avec `asyncio`)
- ✅ Excellent support RESP3
- ✅ Connection pooling automatique

**Points faibles :**
- ⚠️ Performance moyenne comparé aux clients C-based
- ⚠️ API async moins mature que l'API sync

**Exemple de base :**

```python
import redis

# Client synchrone
client = redis.Redis(
    host='localhost',
    port=6379,
    db=0,
    decode_responses=True,  # Retourne des strings au lieu de bytes
    socket_connect_timeout=5,
    socket_timeout=5
)

# Test de connexion
client.ping()  # PONG

# Opérations basiques
client.set('key', 'value')
value = client.get('key')  # 'value'

# Pipeline pour performances
pipe = client.pipeline()
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
pipe.get('key1')
results = pipe.execute()  # [True, True, 'value1']

# Fermeture
client.close()
```

**Exemple asynchrone :**

```python
import asyncio
import redis.asyncio as redis

async def main():
    # Client asynchrone
    client = await redis.Redis(
        host='localhost',
        port=6379,
        decode_responses=True
    )

    # Opérations non-bloquantes
    await client.set('async:key', 'value')
    value = await client.get('async:key')

    # Pipeline asynchrone
    async with client.pipeline() as pipe:
        await pipe.set('key1', 'val1')
        await pipe.set('key2', 'val2')
        results = await pipe.execute()

    await client.close()

asyncio.run(main())
```

#### **Alternative : redis-om-python**

**URL** : https://github.com/redis/redis-om-python
**Usage** : ORM-like pour Redis avec validation Pydantic

```python
from redis_om import HashModel, Field

class User(HashModel):
    name: str = Field(index=True)
    email: str = Field(index=True)
    age: int

# Création
user = User(name="Alice", email="alice@example.com", age=30)
user.save()

# Recherche
users = User.find(User.name == "Alice").all()
```

---

### 🟢 Node.js

#### **ioredis** (Recommandé)

**URL** : https://github.com/redis/ioredis
**Installation** : `npm install ioredis`
**Statut** : Le plus populaire, robuste et feature-complete

**Points forts :**
- ✅ Performance excellente
- ✅ API Promise/async-await native
- ✅ Support complet : Cluster, Sentinel, Streams, Pub/Sub
- ✅ Reconnexion automatique intelligente
- ✅ Pipelining et transactions
- ✅ Lua scripting intégré
- ✅ TypeScript support

**Points faibles :**
- ⚠️ Légèrement plus verbeux que certains clients
- ⚠️ Configuration initiale peut être complexe

**Exemple de base :**

```javascript
import Redis from 'ioredis';

// Configuration complète
const redis = new Redis({
    host: 'localhost',
    port: 6379,
    password: 'yourpassword',
    db: 0,
    retryStrategy(times) {
        const delay = Math.min(times * 50, 2000);
        return delay;
    },
    reconnectOnError(err) {
        const targetError = 'READONLY';
        if (err.message.includes(targetError)) {
            return true; // Reconnecte sur erreur READONLY
        }
        return false;
    },
    maxRetriesPerRequest: 3,
    enableReadyCheck: true,
    lazyConnect: false
});

// Events
redis.on('connect', () => console.log('Connecté à Redis'));
redis.on('error', (err) => console.error('Erreur Redis:', err));

// Opérations basiques
await redis.set('key', 'value');
const value = await redis.get('key'); // 'value'

// Pipeline
const pipeline = redis.pipeline();
pipeline.set('key1', 'value1');
pipeline.set('key2', 'value2');
pipeline.get('key1');
const results = await pipeline.exec();
// [[null, 'OK'], [null, 'OK'], [null, 'value1']]

// Lua script
const result = await redis.eval(
    "return redis.call('set', KEYS[1], ARGV[1])",
    1,
    'mykey',
    'myvalue'
);

// Fermeture
await redis.quit();
```

**Exemple Cluster :**

```javascript
import Redis from 'ioredis';

const cluster = new Redis.Cluster([
    { host: 'localhost', port: 7000 },
    { host: 'localhost', port: 7001 },
    { host: 'localhost', port: 7002 }
], {
    redisOptions: {
        password: 'yourpassword'
    },
    enableReadyCheck: true,
    maxRedirections: 16,
    retryDelayOnFailover: 100,
    retryDelayOnClusterDown: 300
});

// Utilisation identique au client standard
await cluster.set('key', 'value');
const value = await cluster.get('key');
```

#### **Alternative : node-redis**

**URL** : https://github.com/redis/node-redis
**Installation** : `npm install redis`
**Statut** : Client officiel Redis

Moins mature historiquement mais rattrape son retard. Bonne option si vous préférez le client officiel.

```javascript
import { createClient } from 'redis';

const client = createClient({
    url: 'redis://localhost:6379'
});

await client.connect();
await client.set('key', 'value');
const value = await client.get('key');
await client.disconnect();
```

---

### ☕ Java

#### **Jedis** (Simple et synchrone)

**URL** : https://github.com/redis/jedis
**Installation** : Maven/Gradle
**Statut** : Client officiel Redis, le plus ancien

```xml
<!-- Maven -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>5.0.0</version>
</dependency>
```

**Points forts :**
- ✅ API simple et directe
- ✅ Bien documenté
- ✅ Support Cluster et Sentinel
- ✅ Thread-safe avec JedisPool

**Points faibles :**
- ⚠️ Synchrone uniquement (bloquant)
- ⚠️ Performance inférieure à Lettuce

**Exemple :**

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

public class RedisExample {
    public static void main(String[] args) {
        // Configuration du pool
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(50);
        poolConfig.setMaxIdle(10);
        poolConfig.setMinIdle(5);
        poolConfig.setTestOnBorrow(true);

        // Création du pool
        JedisPool pool = new JedisPool(
            poolConfig,
            "localhost",
            6379,
            2000, // timeout
            "password"
        );

        // Utilisation avec try-with-resources
        try (Jedis jedis = pool.getResource()) {
            // Opérations basiques
            jedis.set("key", "value");
            String value = jedis.get("key"); // "value"

            // Pipeline
            var pipeline = jedis.pipelined();
            pipeline.set("key1", "value1");
            pipeline.set("key2", "value2");
            pipeline.get("key1");
            var results = pipeline.syncAndReturnAll();

            // Transaction
            var transaction = jedis.multi();
            transaction.set("key", "newvalue");
            transaction.incr("counter");
            transaction.exec();

        } catch (Exception e) {
            System.err.println("Erreur Redis: " + e.getMessage());
        }

        // Fermeture du pool
        pool.close();
    }
}
```

#### **Lettuce** (Asynchrone et réactif - Recommandé)

**URL** : https://github.com/lettuce-io/lettuce-core
**Installation** : Maven/Gradle
**Statut** : Client avancé, le plus performant

```xml
<!-- Maven -->
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>6.3.0.RELEASE</version>
</dependency>
```

**Points forts :**
- ✅ Support asynchrone complet (CompletableFuture)
- ✅ Programmation réactive (Reactor)
- ✅ Performance excellente
- ✅ Thread-safe par défaut
- ✅ Support RESP3
- ✅ Client-side caching

**Exemple synchrone :**

```java
import io.lettuce.core.*;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.sync.RedisCommands;

public class LettuceExample {
    public static void main(String[] args) {
        // Client avec URI
        RedisClient client = RedisClient.create("redis://password@localhost:6379/0");

        // Connexion
        StatefulRedisConnection<String, String> connection = client.connect();
        RedisCommands<String, String> commands = connection.sync();

        // Opérations
        commands.set("key", "value");
        String value = commands.get("key");

        // Fermeture
        connection.close();
        client.shutdown();
    }
}
```

**Exemple asynchrone :**

```java
import io.lettuce.core.*;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.async.RedisAsyncCommands;
import java.util.concurrent.CompletableFuture;

public class LettuceAsyncExample {
    public static void main(String[] args) throws Exception {
        RedisClient client = RedisClient.create("redis://localhost:6379");
        StatefulRedisConnection<String, String> connection = client.connect();
        RedisAsyncCommands<String, String> async = connection.async();

        // Opérations asynchrones
        CompletableFuture<String> setFuture = async.set("key", "value");
        CompletableFuture<String> getFuture = async.get("key");

        // Composition de futures
        getFuture.thenAccept(value ->
            System.out.println("Value: " + value)
        );

        // Attente (dans un vrai projet, ne pas bloquer)
        getFuture.get();

        connection.close();
        client.shutdown();
    }
}
```

---

### 🐹 Go

#### **go-redis** (Recommandé)

**URL** : https://github.com/redis/go-redis
**Installation** : `go get github.com/redis/go-redis/v9`
**Statut** : Client officiel Redis, le plus populaire

**Points forts :**
- ✅ Client officiel
- ✅ API idiomatique Go
- ✅ Support context.Context natif
- ✅ Excellent support Cluster et Sentinel
- ✅ Pipeline et transactions
- ✅ Performance excellente
- ✅ Thread-safe

**Exemple complet :**

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()

    // Configuration du client
    rdb := redis.NewClient(&redis.Options{
        Addr:         "localhost:6379",
        Password:     "",
        DB:           0,
        PoolSize:     50,
        MinIdleConns: 10,
        MaxRetries:   3,
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
    })
    defer rdb.Close()

    // Test de connexion
    if err := rdb.Ping(ctx).Err(); err != nil {
        panic(fmt.Sprintf("Impossible de se connecter: %v", err))
    }

    // Opérations basiques
    err := rdb.Set(ctx, "key", "value", 0).Err()
    if err != nil {
        panic(err)
    }

    val, err := rdb.Get(ctx, "key").Result()
    if err != nil {
        panic(err)
    }
    fmt.Println("key:", val)

    // Gestion des clés inexistantes
    val2, err := rdb.Get(ctx, "nonexistent").Result()
    if err == redis.Nil {
        fmt.Println("Clé n'existe pas")
    } else if err != nil {
        panic(err)
    } else {
        fmt.Println("nonexistent:", val2)
    }

    // Pipeline
    pipe := rdb.Pipeline()
    incr := pipe.Incr(ctx, "counter")
    pipe.Expire(ctx, "counter", time.Hour)
    _, err = pipe.Exec(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Println("counter:", incr.Val())

    // Transaction avec WATCH (optimistic locking)
    err = rdb.Watch(ctx, func(tx *redis.Tx) error {
        // Get current value
        val, err := tx.Get(ctx, "key").Result()
        if err != nil && err != redis.Nil {
            return err
        }

        // Operation dans une transaction
        _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
            pipe.Set(ctx, "key", val+"_modified", 0)
            return nil
        })
        return err
    }, "key")

    if err != nil {
        panic(err)
    }
}
```

**Exemple Cluster :**

```go
package main

import (
    "context"
    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()

    // Client cluster
    rdb := redis.NewClusterClient(&redis.ClusterOptions{
        Addrs: []string{
            "localhost:7000",
            "localhost:7001",
            "localhost:7002",
        },
        Password:     "",
        MaxRetries:   3,
        PoolSize:     50,
        MinIdleConns: 10,
    })
    defer rdb.Close()

    // Utilisation identique
    rdb.Set(ctx, "key", "value", 0)
    val, _ := rdb.Get(ctx, "key").Result()
    println(val)
}
```

#### **Alternative : redigo**

**URL** : https://github.com/gomodule/redigo
**Statut** : Ancien client populaire, moins actif aujourd'hui

Moins recommandé car go-redis est devenu le standard de facto.

---

### 🐘 PHP

#### **phpredis** (Extension C - Recommandé)

**URL** : https://github.com/phpredis/phpredis
**Installation** : `pecl install redis`
**Statut** : Extension C native, la plus performante

**Points forts :**
- ✅ Performance excellente (écrit en C)
- ✅ API simple
- ✅ Support Cluster et Sentinel
- ✅ Session handler intégré

**Exemple :**

```php
<?php

// Connexion simple
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);
$redis->auth('password');
$redis->select(0);

// Test de connexion
if ($redis->ping()) {
    echo "Connecté!\n";
}

// Opérations basiques
$redis->set('key', 'value');
$value = $redis->get('key');
echo "key: $value\n";

// TTL
$redis->setex('tempkey', 3600, 'value'); // Expire dans 1h

// Pipeline
$redis->multi(Redis::PIPELINE);
$redis->set('key1', 'value1');
$redis->set('key2', 'value2');
$redis->get('key1');
$results = $redis->exec();
print_r($results);

// Transaction
$redis->multi(Redis::MULTI);
$redis->set('key', 'newvalue');
$redis->incr('counter');
$redis->exec();

// Hash
$redis->hSet('user:1', 'name', 'Alice');
$redis->hSet('user:1', 'email', 'alice@example.com');
$user = $redis->hGetAll('user:1');
print_r($user);

// Session handler (dans php.ini ou code)
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://127.0.0.1:6379');

$redis->close();
```

#### **Predis** (Pure PHP)

**URL** : https://github.com/predis/predis
**Installation** : `composer require predis/predis`
**Statut** : Client pur PHP, portable

**Points forts :**
- ✅ Pure PHP (pas d'extension C requise)
- ✅ Portable
- ✅ Bon pour développement

**Points faibles :**
- ⚠️ Performance inférieure à phpredis
- ⚠️ Support Cluster limité

```php
<?php
require 'vendor/autoload.php';

use Predis\Client;

// Configuration
$client = new Client([
    'scheme' => 'tcp',
    'host'   => '127.0.0.1',
    'port'   => 6379,
    'password' => 'password',
]);

// Opérations
$client->set('key', 'value');
$value = $client->get('key');

// Pipeline
$pipe = $client->pipeline();
$pipe->set('key1', 'value1');
$pipe->set('key2', 'value2');
$pipe->get('key1');
$responses = $pipe->execute();
```

---

### 💠 .NET / C#

#### **StackExchange.Redis** (Recommandé)

**URL** : https://github.com/StackExchange/StackExchange.Redis
**Installation** : `dotnet add package StackExchange.Redis`
**Statut** : Le standard de facto pour .NET

**Points forts :**
- ✅ Performance excellente
- ✅ API async/await native
- ✅ Multiplexing intelligent
- ✅ Support Cluster et Sentinel
- ✅ Thread-safe
- ✅ Reconnexion automatique

**Exemple :**

```csharp
using StackExchange.Redis;
using System;
using System.Threading.Tasks;

public class RedisExample
{
    public static async Task Main(string[] args)
    {
        // Configuration
        var configOptions = new ConfigurationOptions
        {
            EndPoints = { "localhost:6379" },
            Password = "yourpassword",
            ConnectTimeout = 5000,
            SyncTimeout = 5000,
            AbortOnConnectFail = false,
            ConnectRetry = 3
        };

        // Connexion (réutiliser cette instance globalement)
        var connection = await ConnectionMultiplexer.ConnectAsync(configOptions);
        var db = connection.GetDatabase();

        // Test de connexion
        var ping = await db.PingAsync();
        Console.WriteLine($"Ping: {ping.TotalMilliseconds}ms");

        // Opérations basiques
        await db.StringSetAsync("key", "value");
        var value = await db.StringGetAsync("key");
        Console.WriteLine($"key: {value}");

        // TTL
        await db.StringSetAsync("tempkey", "value", TimeSpan.FromHours(1));

        // Opérations atomiques
        await db.StringIncrementAsync("counter");
        var counter = await db.StringGetAsync("counter");
        Console.WriteLine($"counter: {counter}");

        // Hash
        await db.HashSetAsync("user:1", new HashEntry[]
        {
            new HashEntry("name", "Alice"),
            new HashEntry("email", "alice@example.com")
        });
        var user = await db.HashGetAllAsync("user:1");

        // Batch (pipeline)
        var batch = db.CreateBatch();
        var task1 = batch.StringSetAsync("key1", "value1");
        var task2 = batch.StringSetAsync("key2", "value2");
        var task3 = batch.StringGetAsync("key1");
        batch.Execute();
        await Task.WhenAll(task1, task2, task3);
        Console.WriteLine($"key1: {task3.Result}");

        // Transaction
        var tran = db.CreateTransaction();
        tran.AddCondition(Condition.KeyNotExists("lock"));
        var t1 = tran.StringSetAsync("key", "newvalue");
        var t2 = tran.StringIncrementAsync("counter");
        bool committed = await tran.ExecuteAsync();
        Console.WriteLine($"Transaction committed: {committed}");

        // Pub/Sub
        var sub = connection.GetSubscriber();
        await sub.SubscribeAsync("mychannel", (channel, message) =>
        {
            Console.WriteLine($"Message reçu: {message}");
        });

        await sub.PublishAsync("mychannel", "Hello Redis!");

        // Fermeture
        await connection.CloseAsync();
    }
}
```

**Particularité importante** : StackExchange.Redis utilise un modèle **multiplexed** :
- Une seule connexion TCP partagée entre tous les threads
- Fire-and-forget par défaut (async)
- Très performant mais comportement différent des autres clients

```csharp
// ConnectionMultiplexer doit être SINGLETON
// ❌ MAUVAIS
public void BadPractice()
{
    using var conn = ConnectionMultiplexer.Connect("localhost");
    var db = conn.GetDatabase();
    db.StringSet("key", "value");
}

// ✅ BON
public class RedisService
{
    private static readonly Lazy<ConnectionMultiplexer> _lazy =
        new Lazy<ConnectionMultiplexer>(() =>
            ConnectionMultiplexer.Connect("localhost")
        );

    public static IDatabase Database => _lazy.Value.GetDatabase();
}
```

---

## Tableau comparatif récapitulatif

| Langage | Client recommandé | Alternative | Performance | Async | Cluster | Maturité |
|---------|-------------------|-------------|-------------|-------|---------|----------|
| **Python** | redis-py | redis-om-python | ⭐⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐⭐ |
| **Node.js** | ioredis | node-redis | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐⭐ |
| **Java** | Lettuce | Jedis | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐⭐ |
| **Go** | go-redis | redigo | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐⭐ |
| **PHP** | phpredis | Predis | ⭐⭐⭐⭐ | ❌ | ✅ | ⭐⭐⭐⭐ |
| **.NET** | StackExchange.Redis | - | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐⭐ |

---

## Fonctionnalités avancées supportées

### Client-side caching (Redis 6+)

**Disponible dans :**
- ✅ redis-py (via RESP3)
- ✅ Lettuce
- ⚠️ ioredis (support expérimental)
- ✅ go-redis
- ⚠️ StackExchange.Redis (support partiel)

**Exemple avec Lettuce :**

```java
import io.lettuce.core.*;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.support.caching.*;

// Configuration du cache client
CacheFrontend<String, String> frontend = ClientSideCaching.enable(
    CacheAccessor.forMap(new ConcurrentHashMap<>()),
    connection,
    TrackingArgs.Builder.enabled()
);

// Les GET seront automatiquement mis en cache côté client
String value = frontend.get("key");
```

### Redis Streams

**Support complet dans :**
- ✅ Tous les clients majeurs

**Exemple avec ioredis :**

```javascript
// Producteur
await redis.xadd(
    'mystream',
    '*',
    'field1', 'value1',
    'field2', 'value2'
);

// Consommateur
const messages = await redis.xread(
    'BLOCK', 5000,
    'STREAMS', 'mystream', '0'
);
```

### Redis JSON

**Nécessite RedisJSON module côté serveur**

**Support dans :**
- ✅ redis-py (`redis.commands.json`)
- ✅ ioredis (via commandes custom)
- ✅ Jedis/Lettuce (via commandes custom)

**Exemple Python :**

```python
import redis
from redis.commands.json.path import Path

client = redis.Redis(decode_responses=True)

# Stockage JSON
user = {
    "name": "Alice",
    "age": 30,
    "tags": ["python", "redis"]
}
client.json().set("user:1", Path.root_path(), user)

# Récupération
retrieved = client.json().get("user:1")

# Modification partielle
client.json().set("user:1", Path(".age"), 31)
```

---

## Bonnes pratiques universelles

### 1. Réutiliser les connexions (Pool ou Singleton)

```python
# ❌ MAUVAIS : Nouvelle connexion par requête
def bad():
    client = redis.Redis()
    value = client.get('key')
    client.close()

# ✅ BON : Pool global
pool = redis.ConnectionPool(host='localhost', port=6379)
client = redis.Redis(connection_pool=pool)
```

### 2. Toujours définir des timeouts

```javascript
// ✅ Avec timeouts
const redis = new Redis({
    connectTimeout: 5000,
    commandTimeout: 3000
});
```

### 3. Gérer les erreurs de connexion

```go
// ✅ Gestion explicite
val, err := rdb.Get(ctx, "key").Result()
if err == redis.Nil {
    // Clé n'existe pas
} else if err != nil {
    // Erreur réseau ou autre
    log.Printf("Erreur Redis: %v", err)
} else {
    // OK
    fmt.Println(val)
}
```

### 4. Utiliser async quand disponible

```csharp
// ✅ Async en .NET
await db.StringSetAsync("key", "value");
var value = await db.StringGetAsync("key");
```

### 5. Préférer les pipelines pour les opérations groupées

```python
# ✅ Pipeline = 1 RTT au lieu de N
pipe = client.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
```

---

## Checklist de sélection

Avant de choisir votre client, vérifiez :

- ✅ Le client est-il **activement maintenu** ? (dernier commit < 3 mois)
- ✅ Supporte-t-il votre **version de Redis** ?
- ✅ A-t-il une **bonne documentation** ?
- ✅ Supporte-t-il les fonctionnalités dont vous avez besoin ?
  - Cluster ?
  - Sentinel ?
  - Streams ?
  - Pub/Sub ?
  - Redis Stack (JSON, Search, TimeSeries) ?
- ✅ Les performances sont-elles adaptées à votre charge ?
- ✅ Le support **async** est-il nécessaire pour votre application ?
- ✅ Y a-t-il une communauté active en cas de problème ?

---

## Ressources et documentation

### Documentation officielle des clients

- **Python (redis-py)** : https://redis-py.readthedocs.io/
- **Node.js (ioredis)** : https://github.com/redis/ioredis#readme
- **Java (Lettuce)** : https://lettuce.io/core/release/reference/
- **Java (Jedis)** : https://github.com/redis/jedis#readme
- **Go (go-redis)** : https://redis.uptrace.dev/
- **PHP (phpredis)** : https://github.com/phpredis/phpredis#readme
- **.NET (StackExchange.Redis)** : https://stackexchange.github.io/StackExchange.Redis/

### Liste complète des clients

Page officielle avec tous les clients certifiés : https://redis.io/docs/clients/

---

## Points clés à retenir

🔑 **Privilégiez les clients officiels ou largement adoptés** pour éviter les bugs et l'abandon de projet

🔑 **Vérifiez le support de vos fonctionnalités critiques** (Cluster, Sentinel, Streams, Redis Stack)

🔑 **Utilisez les versions asynchrones** pour les applications à haute concurrence

🔑 **Configurez toujours des timeouts et retry logic** pour la résilience

🔑 **Réutilisez les connexions** via pools ou singletons selon le langage

🔑 **Lisez la documentation du client** : chaque implémentation a ses spécificités

🔑 **Testez les performances** dans votre contexte avant de déployer en production

---

## Prochaine section

➡️ **Section 9.2** : Connexion et pool de connexions - Configuration optimale et gestion avancée

**Niveau** : Intermédiaire
**Durée estimée** : 45-60 minutes
**Prérequis** : Connaissance de base d'au moins un langage présenté

⏭️ [Connexion et pool de connexions](/09-integration-langages-programmation/02-connexion-pool-connexions.md)
