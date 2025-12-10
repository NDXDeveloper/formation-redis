🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.1 Pub/Sub classique : Le "Fire and Forget"

## Introduction

Le mécanisme Pub/Sub (Publisher/Subscriber) de Redis est l'un des patterns de messaging les plus simples et rapides disponibles. Basé sur le principe du **"Fire and Forget"** (tirer et oublier), il permet une communication asynchrone ultra-performante au prix d'une absence totale de garanties de livraison.

### Concept fondamental

```
Publisher                    Redis                    Subscribers
    |                          |                          |
    |--- PUBLISH channel msg --|                          |
    |                          |------- message --------> |
    |                          |------- message --------> |
    |                          |------- message --------> |
    |<-- (integer) 3 ----------|                          |
    |   (3 subscribers)        |                          |
```

**Caractéristiques clés** :
- ✅ Latence ultra-faible (< 1ms)
- ✅ Broadcast natif (un message → N subscribers)
- ❌ Aucune persistance
- ❌ Perte de messages si subscriber déconnecté
- ❌ Pas d'historique

---

## Commandes fondamentales

### PUBLISH : Envoyer un message

```bash
# Syntaxe
PUBLISH channel message

# Exemples
PUBLISH notifications:users "New user registered"
# Retourne : (integer) 5  # Nombre de subscribers qui ont reçu

PUBLISH alerts:critical '{"severity":"high","service":"payment"}'
# Retourne : (integer) 0  # Aucun subscriber actif

PUBLISH chat:room:123 "Hello everyone!"
# Retourne : (integer) 12  # 12 users dans le chat
```

**Points importants** :
- Le retour indique le nombre de **subscribers actifs au moment de la publication**
- Si retour = 0, le message est **perdu définitivement**
- Performance : ~100,000 messages/seconde sur matériel standard

### SUBSCRIBE : S'abonner à un channel

```bash
# Syntaxe
SUBSCRIBE channel [channel ...]

# Exemples
SUBSCRIBE notifications:users
# Entre en mode "listening", bloque la connexion

SUBSCRIBE chat:room:1 chat:room:2 chat:room:3
# S'abonner à plusieurs channels simultanément

# Réponse typique
1) "subscribe"
2) "notifications:users"
3) (integer) 1  # Nombre total de subscriptions actives

# Puis, pour chaque message reçu :
1) "message"
2) "notifications:users"
3) "User 123 logged in"
```

**Caractéristiques** :
- La connexion Redis entre en **mode blocking**
- Seules les commandes Pub/Sub sont autorisées
- Impossible d'exécuter d'autres commandes Redis (GET, SET, etc.)
- Nécessite une connexion dédiée par subscriber

### PSUBSCRIBE : Pattern matching

```bash
# Syntaxe
PSUBSCRIBE pattern [pattern ...]

# Exemples
PSUBSCRIBE notifications:*
# Reçoit : notifications:users, notifications:orders, etc.

PSUBSCRIBE events:user:*
# Reçoit : events:user:123, events:user:456, etc.

PSUBSCRIBE logs:*:error
# Reçoit : logs:app:error, logs:db:error, etc.

# Réponse pour chaque message
1) "pmessage"
2) "notifications:*"     # Le pattern qui a matché
3) "notifications:users" # Le channel exact
4) "Message content"     # Le message
```

**Patterns supportés** :
- `*` : Match n'importe quelle séquence de caractères
- `?` : Match exactement un caractère
- `[abc]` : Match un caractère parmi a, b, ou c

### UNSUBSCRIBE : Se désabonner

```bash
# Syntaxe
UNSUBSCRIBE [channel [channel ...]]

# Exemples
UNSUBSCRIBE notifications:users
# Se désabonner d'un channel spécifique

UNSUBSCRIBE
# Se désabonner de TOUS les channels (sans pattern)

PUNSUBSCRIBE notifications:*
# Se désabonner d'un pattern
```

### Commandes d'introspection

```bash
# Lister tous les channels actifs
PUBSUB CHANNELS
# Retourne la liste des channels avec au moins 1 subscriber

# Lister channels avec pattern
PUBSUB CHANNELS notif*
# Retourne : notifications:users, notifications:orders

# Nombre de subscribers par channel
PUBSUB NUMSUB channel1 channel2
# Retourne : 1) "channel1" 2) (integer) 5 3) "channel2" 4) (integer) 3

# Nombre de pattern subscriptions actives
PUBSUB NUMPAT
# Retourne : (integer) 7
```

---

## Implémentation dans différents langages

### Python avec redis-py

```python
import redis
import json
import threading
import time

# ============================================
# PUBLISHER
# ============================================
def publisher_example():
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)

    # Publier un message simple
    num_subscribers = r.publish('notifications', 'Hello World')
    print(f"Message délivré à {num_subscribers} subscriber(s)")

    # Publier des données structurées
    event = {
        'type': 'user_login',
        'user_id': 123,
        'timestamp': time.time()
    }
    r.publish('events:auth', json.dumps(event))

# ============================================
# SUBSCRIBER (version simple)
# ============================================
def subscriber_simple():
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    pubsub = r.pubsub()

    # S'abonner à un channel
    pubsub.subscribe('notifications')

    print("En écoute...")
    for message in pubsub.listen():
        if message['type'] == 'message':
            print(f"Reçu: {message['data']}")

# ============================================
# SUBSCRIBER avec gestion d'erreurs
# ============================================
def subscriber_robust():
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    pubsub = r.pubsub()

    # S'abonner à plusieurs channels
    pubsub.subscribe('notifications', 'alerts', 'events:auth')

    # Pattern subscription
    pubsub.psubscribe('logs:*')

    print("Subscriber démarré...")

    while True:
        try:
            message = pubsub.get_message()

            if message:
                msg_type = message['type']

                if msg_type == 'subscribe':
                    print(f"✓ Abonné à: {message['channel']}")

                elif msg_type == 'psubscribe':
                    print(f"✓ Pattern abonné: {message['pattern']}")

                elif msg_type == 'message':
                    channel = message['channel']
                    data = message['data']
                    print(f"[{channel}] {data}")

                    # Traitement spécifique par channel
                    if channel == 'alerts':
                        handle_alert(data)
                    elif channel == 'events:auth':
                        handle_auth_event(json.loads(data))

                elif msg_type == 'pmessage':
                    pattern = message['pattern']
                    channel = message['channel']
                    data = message['data']
                    print(f"[Pattern:{pattern}] [{channel}] {data}")

            time.sleep(0.01)  # Éviter CPU spin

        except KeyboardInterrupt:
            print("\nArrêt du subscriber...")
            pubsub.unsubscribe()
            pubsub.punsubscribe()
            break

        except Exception as e:
            print(f"Erreur: {e}")
            time.sleep(1)  # Backoff avant retry

def handle_alert(data):
    print(f"🚨 ALERTE: {data}")

def handle_auth_event(event):
    print(f"🔐 Auth: User {event['user_id']} logged in")

# ============================================
# SUBSCRIBER avec threading
# ============================================
class PubSubListener(threading.Thread):
    def __init__(self, channels, patterns=None):
        super().__init__(daemon=True)
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.pubsub = self.redis.pubsub()
        self.channels = channels
        self.patterns = patterns or []

    def run(self):
        self.pubsub.subscribe(*self.channels)
        if self.patterns:
            self.pubsub.psubscribe(*self.patterns)

        print(f"Thread listening on {self.channels} and patterns {self.patterns}")

        for message in self.pubsub.listen():
            if message['type'] in ['message', 'pmessage']:
                self.handle_message(message)

    def handle_message(self, message):
        # Override dans les classes filles
        print(f"Reçu: {message}")

    def stop(self):
        self.pubsub.unsubscribe()
        self.pubsub.punsubscribe()
        self.pubsub.close()

# Utilisation
listener = PubSubListener(['notifications', 'alerts'])
listener.start()

# Faire autre chose pendant que le thread écoute
time.sleep(60)
listener.stop()
```

### Node.js avec ioredis

```javascript
const Redis = require('ioredis');

// ============================================
// PUBLISHER
// ============================================
async function publisherExample() {
    const redis = new Redis();

    // Publier un message
    const numSubscribers = await redis.publish('notifications', 'Hello World');
    console.log(`Délivré à ${numSubscribers} subscriber(s)`);

    // Publier JSON
    const event = {
        type: 'user_login',
        userId: 123,
        timestamp: Date.now()
    };
    await redis.publish('events:auth', JSON.stringify(event));

    await redis.quit();
}

// ============================================
// SUBSCRIBER (version simple)
// ============================================
async function subscriberSimple() {
    const redis = new Redis();

    await redis.subscribe('notifications', (err, count) => {
        if (err) {
            console.error('Erreur subscription:', err);
            return;
        }
        console.log(`Abonné à ${count} channel(s)`);
    });

    redis.on('message', (channel, message) => {
        console.log(`[${channel}] ${message}`);
    });
}

// ============================================
// SUBSCRIBER avec pattern matching
// ============================================
async function subscriberWithPatterns() {
    const redis = new Redis();

    // Subscribe à des channels spécifiques
    await redis.subscribe('notifications', 'alerts');

    // Subscribe avec patterns
    await redis.psubscribe('events:*', 'logs:*:error');

    // Handler pour messages normaux
    redis.on('message', (channel, message) => {
        console.log(`[${channel}] ${message}`);

        // Traitement par channel
        switch(channel) {
            case 'notifications':
                handleNotification(message);
                break;
            case 'alerts':
                handleAlert(message);
                break;
        }
    });

    // Handler pour pattern messages
    redis.on('pmessage', (pattern, channel, message) => {
        console.log(`[Pattern:${pattern}] [${channel}] ${message}`);

        if (pattern === 'events:*') {
            handleEvent(channel, JSON.parse(message));
        }
    });

    // Handler pour subscription confirmations
    redis.on('subscribe', (channel, count) => {
        console.log(`✓ Abonné à ${channel} (total: ${count})`);
    });

    redis.on('psubscribe', (pattern, count) => {
        console.log(`✓ Pattern ${pattern} (total: ${count})`);
    });

    // Handler d'erreurs
    redis.on('error', (err) => {
        console.error('Redis error:', err);
    });
}

function handleNotification(message) {
    console.log('📬 Notification:', message);
}

function handleAlert(message) {
    console.log('🚨 ALERTE:', message);
}

function handleEvent(channel, event) {
    console.log('📊 Event:', event);
}

// ============================================
// SUBSCRIBER avec reconnexion automatique
// ============================================
class RobustSubscriber {
    constructor(channels = [], patterns = []) {
        this.channels = channels;
        this.patterns = patterns;
        this.redis = null;
        this.setupRedis();
    }

    setupRedis() {
        this.redis = new Redis({
            retryStrategy: (times) => {
                const delay = Math.min(times * 50, 2000);
                console.log(`Retry connection in ${delay}ms...`);
                return delay;
            },
            reconnectOnError: (err) => {
                console.log('Reconnect on error:', err.message);
                return true;
            }
        });

        this.redis.on('connect', () => {
            console.log('✓ Connected to Redis');
            this.subscribe();
        });

        this.redis.on('ready', () => {
            console.log('✓ Redis ready');
        });

        this.redis.on('error', (err) => {
            console.error('Redis error:', err.message);
        });

        this.redis.on('reconnecting', () => {
            console.log('⟳ Reconnecting...');
        });

        this.redis.on('message', this.handleMessage.bind(this));
        this.redis.on('pmessage', this.handlePMessage.bind(this));
    }

    async subscribe() {
        if (this.channels.length > 0) {
            await this.redis.subscribe(...this.channels);
            console.log(`Subscribed to: ${this.channels.join(', ')}`);
        }

        if (this.patterns.length > 0) {
            await this.redis.psubscribe(...this.patterns);
            console.log(`Pattern subscribed: ${this.patterns.join(', ')}`);
        }
    }

    handleMessage(channel, message) {
        console.log(`[${channel}] ${message}`);
        // Override dans les classes filles
    }

    handlePMessage(pattern, channel, message) {
        console.log(`[${pattern}] [${channel}] ${message}`);
        // Override dans les classes filles
    }

    async close() {
        if (this.redis) {
            await this.redis.quit();
        }
    }
}

// Utilisation
const subscriber = new RobustSubscriber(
    ['notifications', 'alerts'],
    ['events:*', 'logs:*']
);

// Graceful shutdown
process.on('SIGINT', async () => {
    console.log('\nShutting down...');
    await subscriber.close();
    process.exit(0);
});
```

### Go avec go-redis

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

// ============================================
// PUBLISHER
// ============================================
func publisherExample(ctx context.Context) {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Publier un message simple
    numSubscribers, err := rdb.Publish(ctx, "notifications", "Hello World").Result()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Délivré à %d subscriber(s)\n", numSubscribers)

    // Publier des données structurées
    event := map[string]interface{}{
        "type":      "user_login",
        "user_id":   123,
        "timestamp": time.Now().Unix(),
    }

    eventJSON, _ := json.Marshal(event)
    rdb.Publish(ctx, "events:auth", eventJSON)
}

// ============================================
// SUBSCRIBER (version simple)
// ============================================
func subscriberSimple(ctx context.Context) {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Créer un PubSub
    pubsub := rdb.Subscribe(ctx, "notifications")
    defer pubsub.Close()

    // Attendre confirmation de subscription
    _, err := pubsub.Receive(ctx)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Subscribed to notifications")

    // Recevoir les messages
    ch := pubsub.Channel()

    for msg := range ch {
        fmt.Printf("[%s] %s\n", msg.Channel, msg.Payload)
    }
}

// ============================================
// SUBSCRIBER avec pattern matching
// ============================================
func subscriberWithPatterns(ctx context.Context) {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Subscribe à plusieurs channels et patterns
    pubsub := rdb.Subscribe(ctx, "notifications", "alerts")
    pubsub.PSubscribe(ctx, "events:*", "logs:*:error")
    defer pubsub.Close()

    // Attendre confirmations
    for i := 0; i < 4; i++ {
        _, err := pubsub.Receive(ctx)
        if err != nil {
            log.Fatal(err)
        }
    }
    fmt.Println("All subscriptions confirmed")

    // Recevoir les messages
    ch := pubsub.Channel()

    for msg := range ch {
        fmt.Printf("[%s] %s\n", msg.Channel, msg.Payload)

        // Traitement par channel
        switch msg.Channel {
        case "notifications":
            handleNotification(msg.Payload)
        case "alerts":
            handleAlert(msg.Payload)
        default:
            if msg.Pattern != "" {
                handlePatternMessage(msg.Pattern, msg.Channel, msg.Payload)
            }
        }
    }
}

func handleNotification(payload string) {
    fmt.Println("📬 Notification:", payload)
}

func handleAlert(payload string) {
    fmt.Println("🚨 ALERTE:", payload)
}

func handlePatternMessage(pattern, channel, payload string) {
    fmt.Printf("📊 [Pattern:%s] [%s] %s\n", pattern, channel, payload)
}

// ============================================
// SUBSCRIBER robuste avec reconnexion
// ============================================
type RobustSubscriber struct {
    client   *redis.Client
    channels []string
    patterns []string
    ctx      context.Context
    cancel   context.CancelFunc
}

func NewRobustSubscriber(channels, patterns []string) *RobustSubscriber {
    ctx, cancel := context.WithCancel(context.Background())

    return &RobustSubscriber{
        client: redis.NewClient(&redis.Options{
            Addr:         "localhost:6379",
            MaxRetries:   5,
            DialTimeout:  10 * time.Second,
            ReadTimeout:  30 * time.Second,
            WriteTimeout: 30 * time.Second,
        }),
        channels: channels,
        patterns: patterns,
        ctx:      ctx,
        cancel:   cancel,
    }
}

func (s *RobustSubscriber) Start() {
    go s.listen()
}

func (s *RobustSubscriber) listen() {
    for {
        select {
        case <-s.ctx.Done():
            return
        default:
            s.subscribe()
        }
    }
}

func (s *RobustSubscriber) subscribe() {
    pubsub := s.client.Subscribe(s.ctx, s.channels...)

    if len(s.patterns) > 0 {
        pubsub.PSubscribe(s.ctx, s.patterns...)
    }

    defer pubsub.Close()

    fmt.Println("✓ Subscribed successfully")

    ch := pubsub.Channel()

    for {
        select {
        case <-s.ctx.Done():
            return
        case msg, ok := <-ch:
            if !ok {
                fmt.Println("Channel closed, reconnecting...")
                time.Sleep(1 * time.Second)
                return
            }
            s.handleMessage(msg)
        }
    }
}

func (s *RobustSubscriber) handleMessage(msg *redis.Message) {
    fmt.Printf("[%s] %s\n", msg.Channel, msg.Payload)
}

func (s *RobustSubscriber) Stop() {
    s.cancel()
    s.client.Close()
}

// ============================================
// MAIN
// ============================================
func main() {
    ctx := context.Background()

    // Démarrer subscriber dans une goroutine
    go subscriberWithPatterns(ctx)

    // Attendre un peu
    time.Sleep(1 * time.Second)

    // Publier quelques messages
    publisherExample(ctx)

    // Garder le programme actif
    time.Sleep(10 * time.Second)
}
```

---

## Patterns d'architecture

### Pattern 1 : Invalidation de cache distribuée

```python
import redis
import json
import hashlib

class DistributedCache:
    def __init__(self):
        # Connexion pour cache operations
        self.cache_redis = redis.Redis(host='localhost', port=6379, db=0)

        # Connexion dédiée pour Pub/Sub
        self.pubsub_redis = redis.Redis(host='localhost', port=6379, db=0)
        self.pubsub = self.pubsub_redis.pubsub()

        # Local cache (L1)
        self.local_cache = {}

        # Subscribe aux invalidations
        self.pubsub.subscribe('cache:invalidate')
        self.start_listening()

    def get(self, key):
        # 1. Check local cache (L1)
        if key in self.local_cache:
            print(f"✓ L1 cache hit: {key}")
            return self.local_cache[key]

        # 2. Check Redis cache (L2)
        value = self.cache_redis.get(key)
        if value:
            print(f"✓ L2 cache hit: {key}")
            self.local_cache[key] = value
            return value

        # 3. Cache miss
        print(f"✗ Cache miss: {key}")
        return None

    def set(self, key, value, ttl=300):
        # 1. Set dans Redis
        self.cache_redis.setex(key, ttl, value)

        # 2. Set dans local cache
        self.local_cache[key] = value

        # 3. Invalider sur les autres instances
        self.invalidate_others(key)

    def invalidate_others(self, key):
        """Invalider le cache sur les autres instances"""
        # Générer un token unique pour éviter boucle infinie
        instance_id = id(self)

        message = json.dumps({
            'key': key,
            'instance_id': instance_id
        })

        self.pubsub_redis.publish('cache:invalidate', message)

    def start_listening(self):
        """Écouter les invalidations en background"""
        import threading

        def listen():
            for message in self.pubsub.listen():
                if message['type'] == 'message':
                    self.handle_invalidation(message['data'])

        thread = threading.Thread(target=listen, daemon=True)
        thread.start()

    def handle_invalidation(self, data):
        """Handler pour les messages d'invalidation"""
        msg = json.loads(data)
        key = msg['key']
        sender_id = msg['instance_id']

        # Ignorer nos propres messages
        if sender_id == id(self):
            return

        # Invalider local cache
        if key in self.local_cache:
            del self.local_cache[key]
            print(f"✓ Invalidated local cache: {key}")

# Utilisation
cache1 = DistributedCache()
cache2 = DistributedCache()

# Instance 1 set une valeur
cache1.set('user:123', '{"name":"John","age":30}')

# Instance 2 la récupère (sera en L2)
value = cache2.get('user:123')

# Instance 1 update
cache1.set('user:123', '{"name":"John","age":31}')
# → cache2 reçoit invalidation et supprime de L1
```

### Pattern 2 : Live notifications WebSocket

```javascript
// Server-side (Node.js avec Socket.IO)
const express = require('express');
const http = require('http');
const socketIO = require('socket.io');
const Redis = require('ioredis');

const app = express();
const server = http.createServer(app);
const io = socketIO(server);

// Redis connections
const redisPublisher = new Redis();
const redisSubscriber = new Redis();

// Map userId -> socket connections
const userSockets = new Map();

// Subscribe à tous les channels de notifications
redisSubscriber.psubscribe('notifications:user:*');

redisSubscriber.on('pmessage', (pattern, channel, message) => {
    // Extract user ID from channel
    const userId = channel.split(':')[2];

    // Get socket connections pour cet user
    const sockets = userSockets.get(userId) || [];

    // Envoyer à tous les sockets de l'user
    sockets.forEach(socket => {
        socket.emit('notification', JSON.parse(message));
    });

    console.log(`Sent notification to user ${userId} (${sockets.length} connections)`);
});

// WebSocket connection handler
io.on('connection', (socket) => {
    console.log('Client connected:', socket.id);

    // Authentification (simplifié)
    socket.on('authenticate', (userId) => {
        socket.userId = userId;

        // Ajouter à la map
        if (!userSockets.has(userId)) {
            userSockets.set(userId, []);
        }
        userSockets.get(userId).push(socket);

        console.log(`User ${userId} authenticated (${userSockets.get(userId).length} connections)`);
    });

    socket.on('disconnect', () => {
        if (socket.userId) {
            const sockets = userSockets.get(socket.userId) || [];
            const index = sockets.indexOf(socket);
            if (index > -1) {
                sockets.splice(index, 1);
            }

            // Cleanup si plus de connexions
            if (sockets.length === 0) {
                userSockets.delete(socket.userId);
            }
        }
        console.log('Client disconnected:', socket.id);
    });
});

// API endpoint pour envoyer une notification
app.post('/notify/:userId', express.json(), async (req, res) => {
    const { userId } = req.params;
    const notification = req.body;

    // Publier via Redis Pub/Sub
    await redisPublisher.publish(
        `notifications:user:${userId}`,
        JSON.stringify(notification)
    );

    res.json({ success: true });
});

server.listen(3000, () => {
    console.log('Server listening on port 3000');
});
```

```javascript
// Client-side (Browser)
const socket = io('http://localhost:3000');

socket.on('connect', () => {
    console.log('Connected to server');

    // S'authentifier
    const userId = getCurrentUserId();
    socket.emit('authenticate', userId);
});

socket.on('notification', (notification) => {
    console.log('Received notification:', notification);

    // Afficher notification dans l'UI
    showNotification(notification);
});

function showNotification(notification) {
    const div = document.createElement('div');
    div.className = 'notification';
    div.textContent = notification.message;
    document.body.appendChild(div);

    setTimeout(() => div.remove(), 5000);
}
```

### Pattern 3 : Fan-out pour microservices

```python
import redis
import json
import logging
from typing import Dict, Any

class EventBus:
    """Event bus basé sur Redis Pub/Sub pour architecture microservices"""

    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379)
        self.logger = logging.getLogger(__name__)

    def publish_event(self, event_type: str, payload: Dict[str, Any]):
        """Publier un événement sur le bus"""
        event = {
            'type': event_type,
            'payload': payload,
            'timestamp': time.time(),
            'source': 'service-name'
        }

        channel = f'events:{event_type}'
        message = json.dumps(event)

        num_subscribers = self.redis.publish(channel, message)
        self.logger.info(f"Event {event_type} published to {num_subscribers} services")

        return num_subscribers

# Service 1: Order Service
class OrderService:
    def __init__(self):
        self.event_bus = EventBus()

    def create_order(self, user_id, items):
        # Logique métier...
        order = {
            'order_id': 'ORD-123',
            'user_id': user_id,
            'items': items,
            'total': 99.99
        }

        # Publier événement
        self.event_bus.publish_event('order.created', order)

        return order

# Service 2: Inventory Service (subscriber)
class InventoryService:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379)
        self.pubsub = self.redis.pubsub()
        self.pubsub.subscribe('events:order.created')

    def listen(self):
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                self.handle_order_created(json.loads(message['data']))

    def handle_order_created(self, event):
        order = event['payload']
        print(f"Inventory: Reserving stock for order {order['order_id']}")
        # Logique de réservation...

# Service 3: Notification Service (subscriber)
class NotificationService:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379)
        self.pubsub = self.redis.pubsub()
        self.pubsub.subscribe('events:order.created')

    def listen(self):
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                self.handle_order_created(json.loads(message['data']))

    def handle_order_created(self, event):
        order = event['payload']
        user_id = order['user_id']
        print(f"Notification: Sending confirmation to user {user_id}")
        # Logique d'envoi email/SMS...

# Service 4: Analytics Service (subscriber)
class AnalyticsService:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379)
        self.pubsub = self.redis.pubsub()

        # Subscribe à tous les événements
        self.pubsub.psubscribe('events:*')

    def listen(self):
        for message in self.pubsub.listen():
            if message['type'] == 'pmessage':
                self.track_event(message['channel'], json.loads(message['data']))

    def track_event(self, channel, event):
        event_type = event['type']
        print(f"Analytics: Tracking {event_type}")
        # Logique d'analytics...
```

---

## Limitations et pièges

### 1. Perte de messages garantie

```python
import redis
import time

r = redis.Redis()
pubsub = r.pubsub()

# ❌ PIÈGE : Subscribe APRÈS publication
r.publish('channel', 'Message 1')  # Publié AVANT subscription
time.sleep(0.1)

pubsub.subscribe('channel')  # Subscribe maintenant
# → Message 1 est PERDU définitivement

r.publish('channel', 'Message 2')  # Sera reçu
time.sleep(0.1)

for message in pubsub.listen():
    if message['type'] == 'message':
        print(message['data'])
        # Recevra uniquement "Message 2"
        break
```

**Solution** : Toujours subscribe AVANT que les publishers commencent.

### 2. Connexion bloquée

```python
import redis

r = redis.Redis()

# ❌ PIÈGE : Une connexion en mode subscribe ne peut plus faire autre chose
pubsub = r.pubsub()
pubsub.subscribe('notifications')

# Impossible de faire ça sur la même connexion :
try:
    r.set('key', 'value')  # ❌ ERREUR
except redis.exceptions.ConnectionError as e:
    print("Connexion en mode Pub/Sub uniquement!")

# ✅ SOLUTION : Utiliser des connexions séparées
r_normal = redis.Redis()  # Pour operations normales
r_pubsub = redis.Redis()  # Pour Pub/Sub uniquement
pubsub = r_pubsub.pubsub()
pubsub.subscribe('notifications')

r_normal.set('key', 'value')  # ✓ Fonctionne
```

### 3. Pas de buffer ni queue

```python
import redis
import time

# Scenario : Subscriber lent
r = redis.Redis()
pubsub = r.pubsub()
pubsub.subscribe('fast-channel')

# Publisher envoie 1000 messages très rapidement
for i in range(1000):
    r.publish('fast-channel', f'Message {i}')

time.sleep(1)

# Subscriber traite lentement
count = 0
for message in pubsub.listen():
    if message['type'] == 'message':
        count += 1
        time.sleep(0.1)  # Traitement lent
        if count >= 10:
            break

# Résultat : Seulement ~10 messages reçus
# Les 990 autres sont perdus car aucun buffering
```

**Solution** : Si besoin de buffer, utiliser Redis Streams.

### 4. Absence de persistence

```bash
# Publisher
PUBLISH orders:created '{"order_id":"123","amount":99.99}'
# (integer) 0  # Aucun subscriber!

# Le message est perdu, aucun moyen de le récupérer
# Même si un subscriber se connecte 1 seconde après
```

**Solution** : Pour messages critiques, utiliser Streams ou hybrid pattern.

### 5. Pattern matching performance

```python
import redis
import time

r = redis.Redis()

# ❌ PIÈGE : Pattern subscriptions sont plus coûteux
pubsub = r.pubsub()

# Ceci est rapide
pubsub.subscribe('notifications:user:123')

# Ceci est plus lent (doit matcher chaque publication)
pubsub.psubscribe('notifications:user:*')

# Si 10,000 channels différents reçoivent des messages,
# chaque pattern subscription doit vérifier TOUS les channels
```

**Recommandations** :
- Préférer subscriptions exactes quand possible
- Limiter le nombre de pattern subscriptions
- Être conscient du coût en O(N*M) où N=patterns, M=channels

### 6. Memory leak avec subscriptions non-nettoyées

```python
import redis

# ❌ PIÈGE : Oublier de nettoyer les subscriptions
def broken_subscriber():
    r = redis.Redis()
    pubsub = r.pubsub()
    pubsub.subscribe('channel')

    # ... traiter quelques messages ...

    # Oubli de cleanup !
    return  # La connexion reste ouverte!

# Appelé 1000 fois → 1000 connexions zombies

# ✅ SOLUTION : Toujours cleanup
def good_subscriber():
    r = redis.Redis()
    pubsub = r.pubsub()
    try:
        pubsub.subscribe('channel')
        # ... traiter messages ...
    finally:
        pubsub.unsubscribe()
        pubsub.close()
        r.close()
```

---

## Monitoring et debugging

### Commandes de diagnostic

```bash
# 1. Voir les channels actifs
redis-cli PUBSUB CHANNELS
# Liste tous les channels avec au moins 1 subscriber

# 2. Compter subscribers par channel
redis-cli PUBSUB NUMSUB notifications orders alerts
# Retourne le count pour chaque channel

# 3. Compter pattern subscriptions
redis-cli PUBSUB NUMPAT
# Nombre total de PSUBSCRIBE actifs

# 4. Monitorer en temps réel
redis-cli --csv PSUBSCRIBE '*'
# Voir TOUS les messages publiés (debug uniquement!)
```

### Script de monitoring

```python
import redis
import time
from collections import defaultdict

class PubSubMonitor:
    def __init__(self):
        self.redis = redis.Redis()
        self.stats = defaultdict(lambda: {
            'messages': 0,
            'subscribers': 0,
            'last_seen': None
        })

    def collect_stats(self):
        """Collecter les statistiques Pub/Sub"""
        # Lister tous les channels
        channels = self.redis.pubsub_channels()

        for channel in channels:
            # Compter subscribers
            numsub = self.redis.pubsub_numsub(channel)
            count = numsub[0][1] if numsub else 0

            self.stats[channel]['subscribers'] = count
            self.stats[channel]['last_seen'] = time.time()

        # Nombre de pattern subscriptions
        numpat = self.redis.pubsub_numpat()

        return {
            'channels': len(channels),
            'total_patterns': numpat,
            'channel_stats': dict(self.stats)
        }

    def monitor_continuous(self, interval=5):
        """Monitoring continu"""
        while True:
            stats = self.collect_stats()

            print(f"\n=== Pub/Sub Stats (at {time.strftime('%H:%M:%S')}) ===")
            print(f"Active channels: {stats['channels']}")
            print(f"Pattern subscriptions: {stats['total_patterns']}")

            for channel, data in stats['channel_stats'].items():
                print(f"  {channel}: {data['subscribers']} subscribers")

            time.sleep(interval)

# Utilisation
monitor = PubSubMonitor()
monitor.monitor_continuous()
```

---

## Comparaison Pub/Sub vs autres mécanismes

| Critère | Pub/Sub | Streams | Kafka |
|---------|---------|---------|-------|
| **Setup** | Immédiat | Simple | Complexe |
| **Latence** | < 1ms | 1-5ms | 5-50ms |
| **Throughput** | 100K msg/s | 50-80K msg/s | 1M+ msg/s |
| **Persistance** | ❌ Non | ✅ Oui | ✅ Oui |
| **Ordre garantie** | ✅ Par channel | ✅ Strict | ✅ Par partition |
| **Rejouabilité** | ❌ Non | ✅ Oui | ✅ Oui |
| **Consumer groups** | ❌ Non | ✅ Oui | ✅ Oui |
| **Scalabilité** | Limitée | Moyenne | Excellente |
| **Ops complexity** | Très faible | Faible | Élevée |

---

## Quand utiliser Pub/Sub ?

### ✅ Cas d'usage idéaux

1. **Invalidation de cache distribuée**
   - Latence critique
   - Perte acceptable (cache sera regénéré)

2. **Live dashboard / metrics**
   - Affichage temps réel
   - Données éphémères

3. **Chat / messaging temps réel**
   - WebSocket notifications
   - Présence online

4. **Broadcast de configuration**
   - Reload config sur tous les serveurs
   - Event rare

5. **Debugging / development**
   - Monitoring temporaire
   - Logs de dev

### ❌ À éviter

1. **Commandes business critiques**
   - Paiements, orders, transactions
   - → Utiliser Streams

2. **Audit trail / compliance**
   - Besoin de rejouabilité
   - → Utiliser Streams

3. **Job queues avec retry**
   - Traitement garanti
   - → Utiliser Streams ou Lists

4. **Event sourcing**
   - Historique complet requis
   - → Utiliser Streams

5. **High throughput avec garanties**
   - > 100K msg/s avec persistence
   - → Utiliser Kafka

---

## Checklist de production

Avant de déployer Pub/Sub en production :

- [ ] Comprendre et accepter la perte potentielle de messages
- [ ] Implémenter reconnexion automatique subscribers
- [ ] Utiliser connexions dédiées pour Pub/Sub
- [ ] Monitorer nombre de subscribers actifs
- [ ] Limiter l'usage de pattern subscriptions
- [ ] Documenter les channels et leurs sémantiques
- [ ] Tester le comportement lors de crashes
- [ ] Prévoir alerting sur `PUBSUB NUMSUB` = 0
- [ ] Considérer hybrid pattern (Pub/Sub + Streams) pour messages critiques
- [ ] Benchmarker avec charge réaliste

---

## Conclusion

Redis Pub/Sub est un mécanisme puissant et ultra-performant pour la communication temps réel, mais son modèle "Fire and Forget" impose des contraintes fortes :

**Forces** :
- ✅ Simplicité maximale
- ✅ Latence ultra-faible
- ✅ Broadcast natif
- ✅ Pattern matching flexible

**Faiblesses** :
- ❌ Aucune garantie de livraison
- ❌ Pas de persistance
- ❌ Pas de rejouabilité
- ❌ Scalabilité limitée

**Règle d'or** : Utilisez Pub/Sub uniquement pour des messages dont la perte est acceptable. Pour tout le reste, préférez Redis Streams.

---


⏭️ [Sharded Pub/Sub (Redis 7) : Amélioration du scaling](/08-communication-flux-donnees/02-sharded-pubsub-redis7.md)
