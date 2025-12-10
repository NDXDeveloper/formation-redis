🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.2 Sharded Pub/Sub (Redis 7) : Amélioration du Scaling

## Introduction

Redis 7.0 (mars 2022) a introduit **Sharded Pub/Sub**, une évolution majeure du mécanisme Pub/Sub classique qui résout ses limitations en termes de scalabilité dans les architectures distribuées, notamment avec Redis Cluster.

### Le problème du Pub/Sub classique

```
┌────────────────────────────────────────────────────┐
│           Redis Cluster (3 masters)                │
│                                                    │
│  Node 1         Node 2         Node 3              │
│    │              │              │                 │
│    │              │              │                 │
│    └──────────────┴──────────────┘                 │
│            Broadcast global                        │
│    Chaque PUBLISH propage à TOUS les nœuds         │
└────────────────────────────────────────────────────┘

❌ Problème : O(N) messages internes par publication
❌ Limite : ~10-20K channels avant dégradation
❌ Impact : Bandwidth gaspillé entre nœuds
```

### La solution : Sharded Pub/Sub

```
┌────────────────────────────────────────────────────┐
│           Redis Cluster (3 masters)                │
│                                                    │
│  Node 1         Node 2         Node 3              │
│  Shard 1       Shard 2        Shard 3              │
│  Hash: 0-5460  5461-10922    10923-16383           │
│    │              │              │                 │
│    │              │              │                 │
│  SPUBLISH      SPUBLISH       SPUBLISH             │
│  channel:A     channel:B      channel:C            │
│  (local)       (local)        (local)              │
└────────────────────────────────────────────────────┘

✅ Solution : O(1) - Pas de propagation globale
✅ Scaling : Linéaire avec nombre de nœuds
✅ Performance : ~10x meilleur en cluster
```

---

## Comparaison Pub/Sub classique vs Sharded

### Architecture

| Aspect | Pub/Sub classique | Sharded Pub/Sub |
|--------|------------------|-----------------|
| **Propagation** | Broadcast global | Local au shard |
| **Routing** | Tous les nœuds | Hash slot du channel |
| **Messages internes** | O(N) par publish | O(1) |
| **Subscribers** | Sur tous les nœuds | Sur le shard du channel |
| **Compatibilité** | Redis standalone + Cluster | Redis 7+ uniquement |
| **Pattern matching** | ✅ PSUBSCRIBE | ❌ Non supporté |
| **Overhead cluster** | Élevé (broadcast) | Minimal (local) |

### Performance en Redis Cluster

```bash
# Benchmark : 100K publications avec 3 nœuds cluster

# Pub/Sub classique
redis-benchmark -t publish -c 50 -n 100000
# Résultat : ~30,000 ops/sec
# Network bandwidth : ~500 MB/s entre nœuds
# CPU usage : Élevé sur tous les nœuds

# Sharded Pub/Sub
redis-benchmark -t spublish -c 50 -n 100000
# Résultat : ~95,000 ops/sec
# Network bandwidth : ~50 MB/s entre nœuds
# CPU usage : Distribué, chaque nœud ne traite que son shard
```

**Amélioration** :
- 🚀 Throughput : **+300%**
- 📉 Network overhead : **-90%**
- 💻 CPU efficiency : **+250%**

---

## Commandes Sharded Pub/Sub

### SPUBLISH : Publication shardée

```bash
# Syntaxe
SPUBLISH shardchannel message

# Exemples
SPUBLISH notifications:user:123 "New message"
# Retourne : (integer) 2  # Subscribers sur CE shard uniquement

SPUBLISH events:order:456 '{"status":"completed"}'
# Retourne : (integer) 0  # Aucun subscriber sur ce shard

SPUBLISH metrics:server:prod "cpu:45.2"
# Le message reste LOCAL au shard qui possède "metrics:server:prod"
```

**Différences avec PUBLISH** :
- Le channel est hashé comme une clé Redis normale
- Seuls les subscribers sur le même shard reçoivent le message
- Pas de propagation aux autres nœuds du cluster

### SSUBSCRIBE : Souscription shardée

```bash
# Syntaxe
SSUBSCRIBE shardchannel [shardchannel ...]

# Exemples
SSUBSCRIBE notifications:user:123
# S'abonne sur le shard qui possède ce channel

SSUBSCRIBE events:order:*
# ❌ ERREUR : Pattern matching non supporté!

SSUBSCRIBE notifications:user:123 notifications:user:456
# S'abonne à plusieurs channels
# Peut nécessiter connexions à plusieurs shards

# Réponse
1) "ssubscribe"
2) "notifications:user:123"
3) (integer) 1

# Messages reçus
1) "smessage"
2) "notifications:user:123"
3) "Message content"
```

### SUNSUBSCRIBE : Désinscription

```bash
# Syntaxe
SUNSUBSCRIBE [shardchannel [shardchannel ...]]

# Exemples
SUNSUBSCRIBE notifications:user:123
# Se désabonne d'un channel spécifique

SUNSUBSCRIBE
# Se désabonne de TOUS les sharded channels
```

### Commandes d'introspection

```bash
# Channels shardés actifs
PUBSUB SHARDCHANNELS
# Liste les sharded channels avec au moins 1 subscriber

# Avec pattern (pas de subscription, juste listing)
PUBSUB SHARDCHANNELS notif*

# Nombre de subscribers par sharded channel
PUBSUB SHARDNUMSUB channel1 channel2
# Retourne le count LOCAL au shard (pas global!)
```

---

## Implémentation par langage

### Python avec redis-py

```python
import redis
import time
import json

# redis-py 4.3.0+ requis pour Sharded Pub/Sub

# ============================================
# PUBLISHER avec Sharded Pub/Sub
# ============================================
def sharded_publisher():
    # Connexion à Redis 7+
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)

    # Vérifier version Redis
    info = r.info('server')
    if int(info['redis_version'].split('.')[0]) < 7:
        raise Exception("Redis 7+ requis pour Sharded Pub/Sub")

    # Publier sur des channels shardés
    channels = [
        'notifications:user:123',
        'notifications:user:456',
        'events:order:789'
    ]

    for i in range(10):
        for channel in channels:
            message = f"Message {i} to {channel}"

            # SPUBLISH au lieu de PUBLISH
            num_subs = r.execute_command('SPUBLISH', channel, message)
            print(f"[{channel}] Delivered to {num_subs} subscriber(s)")

        time.sleep(1)

# ============================================
# SUBSCRIBER avec Sharded Pub/Sub
# ============================================
def sharded_subscriber():
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    pubsub = r.pubsub()

    # SSUBSCRIBE aux channels shardés
    channels = ['notifications:user:123', 'events:order:789']

    # redis-py ne supporte pas encore nativement SSUBSCRIBE
    # Utilisation via execute_command
    for channel in channels:
        pubsub.execute_command('SSUBSCRIBE', channel)

    print(f"Subscribed to sharded channels: {channels}")

    # Écouter les messages
    for message in pubsub.listen():
        print(f"Raw message: {message}")

        # Format pour sharded pub/sub
        if message['type'] == 'smessage':
            channel = message['channel']
            data = message['data']
            print(f"[SHARDED] [{channel}] {data}")

# ============================================
# Comparaison Pub/Sub vs Sharded Pub/Sub
# ============================================
class MessagingClient:
    def __init__(self, use_sharded=False):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.use_sharded = use_sharded
        self.pubsub = self.redis.pubsub()

    def subscribe(self, *channels):
        if self.use_sharded:
            # Sharded Pub/Sub
            for channel in channels:
                self.pubsub.execute_command('SSUBSCRIBE', channel)
            print(f"✓ Sharded subscribe: {channels}")
        else:
            # Pub/Sub classique
            self.pubsub.subscribe(*channels)
            print(f"✓ Classic subscribe: {channels}")

    def publish(self, channel, message):
        if self.use_sharded:
            # SPUBLISH
            count = self.redis.execute_command('SPUBLISH', channel, message)
        else:
            # PUBLISH
            count = self.redis.publish(channel, message)

        return count

    def listen(self):
        message_type = 'smessage' if self.use_sharded else 'message'

        for msg in self.pubsub.listen():
            if msg['type'] == message_type:
                yield msg['channel'], msg['data']

# ============================================
# Exemple avec Redis Cluster
# ============================================
from redis.cluster import RedisCluster

def sharded_pubsub_cluster():
    # Connexion à un cluster Redis 7+
    cluster = RedisCluster(
        host='localhost',
        port=7000,
        decode_responses=True
    )

    # Les channels sont automatiquement hashés vers les bons shards
    channels = {
        'notifications:user:1': None,  # Shard A
        'notifications:user:2': None,  # Shard B
        'notifications:user:3': None,  # Shard C
    }

    # Vérifier quel shard possède chaque channel
    for channel in channels.keys():
        # Calculer le hash slot
        slot = cluster.keyslot(channel)
        node = cluster.nodes_manager.slots[slot][0]
        channels[channel] = f"{node['host']}:{node['port']}"
        print(f"Channel '{channel}' -> Slot {slot} -> Node {channels[channel]}")

    # Publier avec SPUBLISH
    for channel in channels.keys():
        count = cluster.execute_command('SPUBLISH', channel, f"Message for {channel}")
        print(f"Published to {channel}: {count} subscriber(s)")

# ============================================
# Pattern : Sharding par user ID
# ============================================
class UserNotificationService:
    """Service de notifications shardé par user"""

    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)

    def send_notification(self, user_id, notification):
        """Envoyer notification à un user spécifique"""
        channel = f"notifications:user:{user_id}"

        payload = json.dumps({
            'type': notification['type'],
            'message': notification['message'],
            'timestamp': time.time()
        })

        # SPUBLISH garantit que seul le shard du user reçoit
        count = self.redis.execute_command('SPUBLISH', channel, payload)

        return count > 0  # True si user a un subscriber actif

    def subscribe_user(self, user_id, callback):
        """Subscriber pour un user spécifique"""
        channel = f"notifications:user:{user_id}"

        pubsub = self.redis.pubsub()
        pubsub.execute_command('SSUBSCRIBE', channel)

        print(f"Listening for notifications: user {user_id}")

        for message in pubsub.listen():
            if message['type'] == 'smessage':
                notification = json.loads(message['data'])
                callback(notification)

# Utilisation
service = UserNotificationService()

# Publisher
service.send_notification(123, {
    'type': 'message',
    'message': 'You have a new message'
})

# Subscriber
def handle_notification(notif):
    print(f"📬 {notif['type']}: {notif['message']}")

service.subscribe_user(123, handle_notification)
```

### Node.js avec ioredis

```javascript
const Redis = require('ioredis');

// ioredis 5.0+ requis pour Sharded Pub/Sub

// ============================================
// PUBLISHER avec Sharded Pub/Sub
// ============================================
async function shardedPublisher() {
    const redis = new Redis();

    // Vérifier version Redis
    const info = await redis.info('server');
    const version = parseInt(info.match(/redis_version:(\d+)/)[1]);

    if (version < 7) {
        throw new Error('Redis 7+ required for Sharded Pub/Sub');
    }

    const channels = [
        'notifications:user:123',
        'notifications:user:456',
        'events:order:789'
    ];

    for (let i = 0; i < 10; i++) {
        for (const channel of channels) {
            const message = `Message ${i} to ${channel}`;

            // SPUBLISH au lieu de publish
            const count = await redis.spublish(channel, message);
            console.log(`[${channel}] Delivered to ${count} subscriber(s)`);
        }

        await new Promise(resolve => setTimeout(resolve, 1000));
    }

    await redis.quit();
}

// ============================================
// SUBSCRIBER avec Sharded Pub/Sub
// ============================================
async function shardedSubscriber() {
    const redis = new Redis();

    const channels = ['notifications:user:123', 'events:order:789'];

    // SSUBSCRIBE aux channels shardés
    await redis.ssubscribe(...channels);
    console.log(`Subscribed to sharded channels: ${channels.join(', ')}`);

    // Handler pour messages shardés
    redis.on('smessage', (channel, message) => {
        console.log(`[SHARDED] [${channel}] ${message}`);
    });

    // Handler pour confirmation subscription
    redis.on('ssubscribe', (channel, count) => {
        console.log(`✓ Subscribed to ${channel} (total: ${count})`);
    });

    // Handler d'erreurs
    redis.on('error', (err) => {
        console.error('Redis error:', err);
    });
}

// ============================================
// Classe wrapper unifiée
// ============================================
class MessagingClient {
    constructor(useSharded = false) {
        this.redis = new Redis();
        this.useSharded = useSharded;
    }

    async subscribe(...channels) {
        if (this.useSharded) {
            await this.redis.ssubscribe(...channels);
            console.log(`✓ Sharded subscribe: ${channels.join(', ')}`);
        } else {
            await this.redis.subscribe(...channels);
            console.log(`✓ Classic subscribe: ${channels.join(', ')}`);
        }
    }

    async publish(channel, message) {
        let count;

        if (this.useSharded) {
            count = await this.redis.spublish(channel, message);
        } else {
            count = await this.redis.publish(channel, message);
        }

        return count;
    }

    onMessage(callback) {
        const eventType = this.useSharded ? 'smessage' : 'message';

        this.redis.on(eventType, (channel, message) => {
            callback(channel, message);
        });
    }

    async close() {
        await this.redis.quit();
    }
}

// ============================================
// Exemple avec Redis Cluster
// ============================================
const { Cluster } = require('ioredis');

async function shardedPubsubCluster() {
    const cluster = new Cluster([
        { host: 'localhost', port: 7000 },
        { host: 'localhost', port: 7001 },
        { host: 'localhost', port: 7002 }
    ]);

    const channels = [
        'notifications:user:1',
        'notifications:user:2',
        'notifications:user:3'
    ];

    // Vérifier quel nœud possède chaque channel
    for (const channel of channels) {
        // ioredis calcule automatiquement le slot
        const slot = calculateSlot(channel);
        const node = cluster.slots[slot];
        console.log(`Channel '${channel}' -> Slot ${slot} -> Node ${node[0]}:${node[1]}`);
    }

    // Publier avec SPUBLISH
    for (const channel of channels) {
        const count = await cluster.spublish(channel, `Message for ${channel}`);
        console.log(`Published to ${channel}: ${count} subscriber(s)`);
    }

    await cluster.quit();
}

function calculateSlot(key) {
    // Extraction de la hash tag si présente
    const match = key.match(/\{([^}]+)\}/);
    const hashKey = match ? match[1] : key;

    // Calcul CRC16 (simplifié pour l'exemple)
    return crc16(hashKey) % 16384;
}

// ============================================
// Pattern : Service de notifications shardé
// ============================================
class UserNotificationService {
    constructor() {
        this.redis = new Redis();
    }

    async sendNotification(userId, notification) {
        const channel = `notifications:user:${userId}`;

        const payload = JSON.stringify({
            type: notification.type,
            message: notification.message,
            timestamp: Date.now()
        });

        // SPUBLISH garantit que seul le shard du user reçoit
        const count = await this.redis.spublish(channel, payload);

        return count > 0; // True si user a un subscriber actif
    }

    async subscribeUser(userId, callback) {
        const channel = `notifications:user:${userId}`;

        await this.redis.ssubscribe(channel);
        console.log(`Listening for notifications: user ${userId}`);

        this.redis.on('smessage', (ch, message) => {
            if (ch === channel) {
                const notification = JSON.parse(message);
                callback(notification);
            }
        });
    }

    async close() {
        await this.redis.quit();
    }
}

// Utilisation
const service = new UserNotificationService();

// Publisher
await service.sendNotification(123, {
    type: 'message',
    message: 'You have a new message'
});

// Subscriber
await service.subscribeUser(123, (notification) => {
    console.log(`📬 ${notification.type}: ${notification.message}`);
});

// ============================================
// Migration helper : Pub/Sub -> Sharded
// ============================================
class MigrationHelper {
    constructor() {
        this.classicRedis = new Redis();
        this.shardedRedis = new Redis();
    }

    async dualPublish(channel, message) {
        // Publier sur les deux systèmes pendant la migration
        const [classicCount, shardedCount] = await Promise.all([
            this.classicRedis.publish(channel, message),
            this.shardedRedis.spublish(channel, message)
        ]);

        console.log(`Published: classic=${classicCount}, sharded=${shardedCount}`);

        return {
            classic: classicCount,
            sharded: shardedCount
        };
    }

    async dualSubscribe(channel, callback) {
        // S'abonner aux deux pendant la migration
        await this.classicRedis.subscribe(channel);
        await this.shardedRedis.ssubscribe(channel);

        this.classicRedis.on('message', (ch, msg) => {
            if (ch === channel) callback('classic', ch, msg);
        });

        this.shardedRedis.on('smessage', (ch, msg) => {
            if (ch === channel) callback('sharded', ch, msg);
        });
    }

    async close() {
        await Promise.all([
            this.classicRedis.quit(),
            this.shardedRedis.quit()
        ]);
    }
}
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
// PUBLISHER avec Sharded Pub/Sub
// ============================================
func shardedPublisher(ctx context.Context) {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Vérifier version Redis
    info, err := rdb.Info(ctx, "server").Result()
    if err != nil {
        log.Fatal(err)
    }
    // Parse version (simplifié)

    channels := []string{
        "notifications:user:123",
        "notifications:user:456",
        "events:order:789",
    }

    for i := 0; i < 10; i++ {
        for _, channel := range channels {
            message := fmt.Sprintf("Message %d to %s", i, channel)

            // SPUBLISH
            count, err := rdb.Do(ctx, "SPUBLISH", channel, message).Int64()
            if err != nil {
                log.Printf("Error: %v", err)
                continue
            }

            fmt.Printf("[%s] Delivered to %d subscriber(s)\n", channel, count)
        }

        time.Sleep(1 * time.Second)
    }
}

// ============================================
// SUBSCRIBER avec Sharded Pub/Sub
// ============================================
func shardedSubscriber(ctx context.Context) {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    defer rdb.Close()

    // Créer un PubSub shardé
    pubsub := rdb.SSubscribe(ctx, "notifications:user:123", "events:order:789")
    defer pubsub.Close()

    fmt.Println("Subscribed to sharded channels")

    // Recevoir messages
    ch := pubsub.Channel()

    for msg := range ch {
        fmt.Printf("[SHARDED] [%s] %s\n", msg.Channel, msg.Payload)
    }
}

// ============================================
// Wrapper unifié
// ============================================
type MessagingClient struct {
    client     *redis.Client
    useSharded bool
}

func NewMessagingClient(useSharded bool) *MessagingClient {
    return &MessagingClient{
        client: redis.NewClient(&redis.Options{
            Addr: "localhost:6379",
        }),
        useSharded: useSharded,
    }
}

func (m *MessagingClient) Subscribe(ctx context.Context, channels ...string) *redis.PubSub {
    if m.useSharded {
        fmt.Printf("✓ Sharded subscribe: %v\n", channels)
        return m.client.SSubscribe(ctx, channels...)
    } else {
        fmt.Printf("✓ Classic subscribe: %v\n", channels)
        return m.client.Subscribe(ctx, channels...)
    }
}

func (m *MessagingClient) Publish(ctx context.Context, channel, message string) (int64, error) {
    if m.useSharded {
        return m.client.Do(ctx, "SPUBLISH", channel, message).Int64()
    } else {
        return m.client.Publish(ctx, channel, message).Result()
    }
}

func (m *MessagingClient) Close() error {
    return m.client.Close()
}

// ============================================
// Exemple avec Redis Cluster
// ============================================
func shardedPubsubCluster(ctx context.Context) {
    cluster := redis.NewClusterClient(&redis.ClusterOptions{
        Addrs: []string{
            "localhost:7000",
            "localhost:7001",
            "localhost:7002",
        },
    })
    defer cluster.Close()

    channels := []string{
        "notifications:user:1",
        "notifications:user:2",
        "notifications:user:3",
    }

    // Publier avec SPUBLISH
    for _, channel := range channels {
        count, err := cluster.Do(ctx, "SPUBLISH", channel, fmt.Sprintf("Message for %s", channel)).Int64()
        if err != nil {
            log.Printf("Error: %v", err)
            continue
        }
        fmt.Printf("Published to %s: %d subscriber(s)\n", channel, count)
    }
}

// ============================================
// Service de notifications shardé
// ============================================
type Notification struct {
    Type      string  `json:"type"`
    Message   string  `json:"message"`
    Timestamp int64   `json:"timestamp"`
}

type UserNotificationService struct {
    client *redis.Client
}

func NewUserNotificationService() *UserNotificationService {
    return &UserNotificationService{
        client: redis.NewClient(&redis.Options{
            Addr: "localhost:6379",
        }),
    }
}

func (s *UserNotificationService) SendNotification(ctx context.Context, userID int, notif Notification) (bool, error) {
    channel := fmt.Sprintf("notifications:user:%d", userID)

    notif.Timestamp = time.Now().Unix()
    payload, err := json.Marshal(notif)
    if err != nil {
        return false, err
    }

    // SPUBLISH garantit que seul le shard du user reçoit
    count, err := s.client.Do(ctx, "SPUBLISH", channel, payload).Int64()
    if err != nil {
        return false, err
    }

    return count > 0, nil
}

func (s *UserNotificationService) SubscribeUser(ctx context.Context, userID int, callback func(Notification)) {
    channel := fmt.Sprintf("notifications:user:%d", userID)

    pubsub := s.client.SSubscribe(ctx, channel)
    defer pubsub.Close()

    fmt.Printf("Listening for notifications: user %d\n", userID)

    ch := pubsub.Channel()
    for msg := range ch {
        var notif Notification
        if err := json.Unmarshal([]byte(msg.Payload), &notif); err != nil {
            log.Printf("Error unmarshaling: %v", err)
            continue
        }

        callback(notif)
    }
}

func (s *UserNotificationService) Close() error {
    return s.client.Close()
}

// ============================================
// MAIN
// ============================================
func main() {
    ctx := context.Background()

    // Exemple service notifications
    service := NewUserNotificationService()
    defer service.Close()

    // Subscriber en goroutine
    go service.SubscribeUser(ctx, 123, func(notif Notification) {
        fmt.Printf("📬 %s: %s\n", notif.Type, notif.Message)
    })

    // Attendre que subscriber soit prêt
    time.Sleep(1 * time.Second)

    // Publisher
    _, err := service.SendNotification(ctx, 123, Notification{
        Type:    "message",
        Message: "You have a new message",
    })
    if err != nil {
        log.Fatal(err)
    }

    // Garder le programme actif
    time.Sleep(5 * time.Second)
}
```

---

## Patterns et cas d'usage

### Pattern 1 : Sharding par tenant/customer

```python
import redis
import json

class MultiTenantMessaging:
    """
    Système de messaging multi-tenant avec sharding automatique
    Chaque tenant a ses propres channels shardés
    """

    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)

    def publish_to_tenant(self, tenant_id, event_type, data):
        """Publier un événement pour un tenant spécifique"""
        channel = f"tenant:{tenant_id}:events:{event_type}"

        message = json.dumps({
            'tenant_id': tenant_id,
            'event_type': event_type,
            'data': data,
            'timestamp': time.time()
        })

        # SPUBLISH garantit isolation par tenant
        count = self.redis.execute_command('SPUBLISH', channel, message)

        return count

    def subscribe_tenant(self, tenant_id, event_types=None):
        """Subscriber pour un tenant spécifique"""
        if event_types is None:
            event_types = ['*']

        pubsub = self.redis.pubsub()
        channels = [f"tenant:{tenant_id}:events:{et}" for et in event_types]

        for channel in channels:
            pubsub.execute_command('SSUBSCRIBE', channel)

        print(f"Subscribed to tenant {tenant_id} events: {event_types}")

        for message in pubsub.listen():
            if message['type'] == 'smessage':
                event = json.loads(message['data'])
                self.handle_tenant_event(event)

    def handle_tenant_event(self, event):
        """Handler pour événements tenant"""
        print(f"[Tenant {event['tenant_id']}] {event['event_type']}: {event['data']}")

# Utilisation
messaging = MultiTenantMessaging()

# Publisher
messaging.publish_to_tenant('acme-corp', 'user.login', {'user_id': 123})
messaging.publish_to_tenant('globex', 'user.login', {'user_id': 456})

# Subscriber (chaque tenant isolé)
# Tenant A ne recevra QUE ses événements
messaging.subscribe_tenant('acme-corp', ['user.login', 'user.logout'])
```

**Avantages du sharding** :
- ✅ Isolation complète entre tenants
- ✅ Scalabilité linéaire (1 tenant = 1 shard potentiel)
- ✅ Pas de "noisy neighbor" problem
- ✅ Performance prévisible par tenant

### Pattern 2 : Hot channel distribution

```python
class LoadBalancedChannels:
    """
    Distribuer la charge des hot channels en les shardant
    """

    def __init__(self, num_shards=10):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.num_shards = num_shards

    def get_shard_channel(self, base_channel, shard_id):
        """Générer un channel shardé"""
        return f"{base_channel}:shard:{shard_id}"

    def publish_distributed(self, base_channel, message):
        """Publier sur un shard aléatoire pour distribuer la charge"""
        import random

        shard_id = random.randint(0, self.num_shards - 1)
        channel = self.get_shard_channel(base_channel, shard_id)

        count = self.redis.execute_command('SPUBLISH', channel, message)

        return shard_id, count

    def subscribe_all_shards(self, base_channel):
        """S'abonner à tous les shards d'un channel"""
        pubsub = self.redis.pubsub()

        for shard_id in range(self.num_shards):
            channel = self.get_shard_channel(base_channel, shard_id)
            pubsub.execute_command('SSUBSCRIBE', channel)

        print(f"Subscribed to {self.num_shards} shards of {base_channel}")

        for message in pubsub.listen():
            if message['type'] == 'smessage':
                # Extraire shard_id du channel
                channel = message['channel']
                shard_id = int(channel.split(':shard:')[1])

                print(f"[Shard {shard_id}] {message['data']}")

# Utilisation pour high-traffic channels
load_balancer = LoadBalancedChannels(num_shards=10)

# Publishers distribuent automatiquement
for i in range(100):
    shard, count = load_balancer.publish_distributed('hot-channel', f'Message {i}')
    print(f"Published to shard {shard}")

# Subscriber consomme tous les shards
load_balancer.subscribe_all_shards('hot-channel')
```

### Pattern 3 : Geographic sharding

```python
class GeoShardedMessaging:
    """
    Sharding géographique pour latence minimale
    """

    REGIONS = {
        'us-east': 0,
        'us-west': 1,
        'eu-west': 2,
        'ap-south': 3
    }

    def __init__(self, region):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.region = region

    def publish_regional(self, event_type, data):
        """Publier un événement régional"""
        channel = f"region:{self.region}:events:{event_type}"

        message = json.dumps({
            'region': self.region,
            'event_type': event_type,
            'data': data,
            'timestamp': time.time()
        })

        # SPUBLISH reste local au datacenter/shard
        count = self.redis.execute_command('SPUBLISH', channel, message)

        return count

    def publish_global(self, event_type, data):
        """Publier à toutes les régions (fan-out manuel)"""
        results = {}

        for region in self.REGIONS.keys():
            channel = f"region:{region}:events:{event_type}"
            message = json.dumps({
                'region': 'global',
                'event_type': event_type,
                'data': data,
                'timestamp': time.time()
            })

            count = self.redis.execute_command('SPUBLISH', channel, message)
            results[region] = count

        return results

    def subscribe_regional(self):
        """Subscriber pour événements régionaux uniquement"""
        pubsub = self.redis.pubsub()

        # S'abonner uniquement aux événements de notre région
        channel = f"region:{self.region}:events:*"

        # Note : Pattern matching non supporté en sharded!
        # Il faut subscribe explicitement à chaque event_type
        event_types = ['user.action', 'order.created', 'payment.completed']

        for event_type in event_types:
            full_channel = f"region:{self.region}:events:{event_type}"
            pubsub.execute_command('SSUBSCRIBE', full_channel)

        print(f"Subscribed to regional events in {self.region}")

        for message in pubsub.listen():
            if message['type'] == 'smessage':
                event = json.loads(message['data'])
                print(f"[{self.region}] {event['event_type']}: {event['data']}")

# Utilisation
# Service US-East
us_service = GeoShardedMessaging('us-east')
us_service.publish_regional('user.action', {'action': 'login', 'user_id': 123})

# Service EU-West
eu_service = GeoShardedMessaging('eu-west')
eu_service.publish_regional('user.action', {'action': 'login', 'user_id': 456})

# Chaque région ne reçoit QUE ses événements locaux
```

---

## Migration de Pub/Sub vers Sharded Pub/Sub

### Stratégie de migration progressive

```python
class PubSubMigration:
    """
    Helper pour migration progressive de Pub/Sub vers Sharded Pub/Sub
    """

    def __init__(self, migration_percentage=0):
        """
        migration_percentage: 0-100
        0 = 100% classic Pub/Sub
        100 = 100% Sharded Pub/Sub
        """
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.migration_percentage = migration_percentage

    def publish(self, channel, message):
        """Publier avec routing intelligent"""
        import random

        # Décider quel système utiliser
        use_sharded = random.randint(0, 100) < self.migration_percentage

        if use_sharded:
            # Nouveau système sharded
            count = self.redis.execute_command('SPUBLISH', channel, message)
            system = 'sharded'
        else:
            # Ancien système classic
            count = self.redis.publish(channel, message)
            system = 'classic'

        # Logging pour monitoring
        self.log_publish(channel, system, count)

        return count, system

    def dual_publish(self, channel, message):
        """Publier sur les DEUX systèmes pendant la transition"""
        classic_count = self.redis.publish(channel, message)
        sharded_count = self.redis.execute_command('SPUBLISH', channel, message)

        # Alerter si discordance
        if classic_count != sharded_count:
            self.alert_mismatch(channel, classic_count, sharded_count)

        return {
            'classic': classic_count,
            'sharded': sharded_count
        }

    def log_publish(self, channel, system, count):
        """Logger les publications pour metrics"""
        # Envoyer à système de monitoring
        print(f"[{system.upper()}] Published to {channel}: {count} subscribers")

    def alert_mismatch(self, channel, classic_count, sharded_count):
        """Alerter si différence entre systèmes"""
        diff = abs(classic_count - sharded_count)
        if diff > 0:
            print(f"⚠️  MISMATCH on {channel}: classic={classic_count}, sharded={sharded_count}")

# Phase 1 : Dual publish (0% migration)
migration = PubSubMigration(migration_percentage=0)
migration.dual_publish('events:test', 'Test message')

# Phase 2 : Ramper le traffic (25% migration)
migration = PubSubMigration(migration_percentage=25)
for i in range(100):
    migration.publish('events:test', f'Message {i}')

# Phase 3 : Majorité sharded (75% migration)
migration = PubSubMigration(migration_percentage=75)

# Phase 4 : Complet sharded (100% migration)
migration = PubSubMigration(migration_percentage=100)
```

### Checklist de migration

```python
class MigrationChecklist:
    """
    Vérifications avant migration
    """

    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)

    def check_redis_version(self):
        """Vérifier version Redis >= 7.0"""
        info = self.redis.info('server')
        version = info['redis_version']
        major = int(version.split('.')[0])

        status = major >= 7
        print(f"{'✅' if status else '❌'} Redis version: {version} (>= 7.0 required)")

        return status

    def check_client_library_support(self):
        """Vérifier support Sharded Pub/Sub dans client"""
        try:
            # Tester SPUBLISH
            self.redis.execute_command('SPUBLISH', 'test', 'test')
            print("✅ Client library supports Sharded Pub/Sub")
            return True
        except redis.exceptions.ResponseError as e:
            if 'unknown command' in str(e).lower():
                print("❌ Client library does not support Sharded Pub/Sub")
                return False
            return True

    def check_pattern_subscriptions(self):
        """Identifier pattern subscriptions (non supportées)"""
        # Obtenir liste des patterns actifs
        numpat = self.redis.pubsub_numpat()

        if numpat > 0:
            print(f"⚠️  Warning: {numpat} pattern subscriptions actives")
            print("   Pattern matching NOT supported in Sharded Pub/Sub")
            print("   Action requise : Convertir en subscriptions explicites")
            return False
        else:
            print("✅ No pattern subscriptions")
            return True

    def analyze_channels(self):
        """Analyser les channels existants"""
        channels = self.redis.pubsub_channels()

        print(f"\n📊 Analyse de {len(channels)} channels:")

        # Obtenir subscriber counts
        if channels:
            counts = self.redis.pubsub_numsub(*channels)

            for i in range(0, len(counts), 2):
                channel = counts[i]
                sub_count = counts[i + 1]
                print(f"  {channel}: {sub_count} subscribers")

        return channels

    def estimate_shard_distribution(self, channels):
        """Estimer distribution dans cluster shardé"""
        if not channels:
            return

        # Simuler hash slot distribution
        slot_distribution = {}

        for channel in channels:
            slot = self.calculate_hash_slot(channel)
            shard = slot // (16384 // 3)  # 3 shards example

            if shard not in slot_distribution:
                slot_distribution[shard] = []

            slot_distribution[shard].append(channel)

        print(f"\n📍 Distribution simulée (3 shards):")
        for shard, chans in slot_distribution.items():
            print(f"  Shard {shard}: {len(chans)} channels")

    def calculate_hash_slot(self, key):
        """Calculer hash slot Redis"""
        import binascii

        # Extraire hash tag si présente
        start = key.find('{')
        if start != -1:
            end = key.find('}', start)
            if end != -1 and end != start + 1:
                key = key[start + 1:end]

        # CRC16
        crc = binascii.crc_hqx(key.encode('utf-8'), 0)
        return crc % 16384

    def run_full_check(self):
        """Exécuter tous les checks"""
        print("=" * 60)
        print("MIGRATION CHECKLIST: Pub/Sub → Sharded Pub/Sub")
        print("=" * 60)

        checks = [
            self.check_redis_version(),
            self.check_client_library_support(),
            self.check_pattern_subscriptions()
        ]

        channels = self.analyze_channels()
        self.estimate_shard_distribution(channels)

        print("\n" + "=" * 60)
        if all(checks):
            print("✅ READY FOR MIGRATION")
        else:
            print("❌ ACTION REQUIRED BEFORE MIGRATION")
        print("=" * 60)

        return all(checks)

# Utilisation
checklist = MigrationChecklist()
ready = checklist.run_full_check()
```

---

## Limitations et considérations

### 1. Pas de pattern matching

```bash
# ❌ ERREUR avec Sharded Pub/Sub
SSUBSCRIBE notifications:*
# Error: Pattern matching not supported with sharded channels

# ✅ SOLUTION : Subscribe explicitement
SSUBSCRIBE notifications:user:123
SSUBSCRIBE notifications:user:456
SSUBSCRIBE notifications:order:789
```

**Workaround** :
```python
def subscribe_multiple_users(user_ids):
    """Subscribe à plusieurs users explicitement"""
    pubsub = redis.pubsub()

    channels = [f"notifications:user:{uid}" for uid in user_ids]

    for channel in channels:
        pubsub.execute_command('SSUBSCRIBE', channel)

    return pubsub
```

### 2. Incompatible avec Redis Standalone

```python
# Sharded Pub/Sub optimisé pour Redis Cluster
# Fonctionne en standalone mais sans avantage

import redis

# Standalone : Aucun avantage de Sharded Pub/Sub
r = redis.Redis(host='localhost', port=6379)
r.execute_command('SPUBLISH', 'channel', 'message')
# Fonctionne, mais équivalent à PUBLISH

# Cluster : Avantages complets
from redis.cluster import RedisCluster
cluster = RedisCluster(host='localhost', port=7000)
cluster.execute_command('SPUBLISH', 'channel', 'message')
# Bénéficie du sharding automatique
```

### 3. Migration nécessite Redis 7+

```bash
# Avant migration, vérifier version sur TOUS les nœuds
redis-cli INFO server | grep redis_version

# Si version < 7.0, upgrade requis
# Attention : Upgrade cluster Redis nécessite downtime ou rolling upgrade
```

### 4. Subscriber doit connaître les channels exacts

```python
# Problème : Découverte dynamique de channels difficile
# Classic Pub/Sub : PUBSUB CHANNELS pattern*
# Sharded : Pas de pattern matching

# Solution : Maintenir registry des channels
class ChannelRegistry:
    def __init__(self):
        self.redis = redis.Redis()

    def register_channel(self, channel):
        """Enregistrer un channel shardé"""
        self.redis.sadd('sharded:channels', channel)

    def list_channels(self, prefix=None):
        """Lister channels enregistrés"""
        channels = self.redis.smembers('sharded:channels')

        if prefix:
            channels = [c for c in channels if c.startswith(prefix)]

        return channels

    def subscribe_all(self, prefix=None):
        """Subscribe à tous les channels enregistrés"""
        channels = self.list_channels(prefix)

        pubsub = self.redis.pubsub()
        for channel in channels:
            pubsub.execute_command('SSUBSCRIBE', channel)

        return pubsub
```

---

## Benchmarks détaillés

### Test de scalabilité

```python
import redis
import time
import threading
from statistics import mean, median

def benchmark_pubsub(use_sharded=False, num_channels=100, num_messages=1000):
    """
    Benchmark Classic vs Sharded Pub/Sub
    """
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)

    # Préparer channels
    channels = [f"bench:channel:{i}" for i in range(num_channels)]

    # Subscriber threads
    received = []
    lock = threading.Lock()

    def subscriber_thread(channels_subset):
        pubsub = r.pubsub()

        for channel in channels_subset:
            if use_sharded:
                pubsub.execute_command('SSUBSCRIBE', channel)
            else:
                pubsub.subscribe(channel)

        start = time.time()
        count = 0

        for message in pubsub.listen():
            if message['type'] in ['message', 'smessage']:
                count += 1
                if count >= num_messages:
                    break

        duration = time.time() - start

        with lock:
            received.append({
                'count': count,
                'duration': duration,
                'rate': count / duration
            })

    # Démarrer subscribers (1 par 10 channels)
    threads = []
    channels_per_thread = 10

    for i in range(0, num_channels, channels_per_thread):
        subset = channels[i:i + channels_per_thread]
        t = threading.Thread(target=subscriber_thread, args=(subset,))
        t.start()
        threads.append(t)

    # Attendre que subscribers soient prêts
    time.sleep(2)

    # Publisher
    start = time.time()

    for i in range(num_messages):
        channel = channels[i % num_channels]
        message = f"Message {i}"

        if use_sharded:
            r.execute_command('SPUBLISH', channel, message)
        else:
            r.publish(channel, message)

    publish_duration = time.time() - start
    publish_rate = num_messages / publish_duration

    # Attendre fin subscribers
    for t in threads:
        t.join(timeout=30)

    # Résultats
    system = "Sharded" if use_sharded else "Classic"

    print(f"\n{'=' * 60}")
    print(f"{system} Pub/Sub Benchmark")
    print(f"{'=' * 60}")
    print(f"Channels: {num_channels}")
    print(f"Messages: {num_messages}")
    print(f"Subscribers: {len(threads)}")
    print(f"\nPublisher:")
    print(f"  Duration: {publish_duration:.2f}s")
    print(f"  Rate: {publish_rate:.0f} msg/s")
    print(f"\nSubscribers:")

    if received:
        rates = [r['rate'] for r in received]
        print(f"  Avg rate: {mean(rates):.0f} msg/s")
        print(f"  Median rate: {median(rates):.0f} msg/s")
        print(f"  Min rate: {min(rates):.0f} msg/s")
        print(f"  Max rate: {max(rates):.0f} msg/s")

    return {
        'system': system,
        'publish_rate': publish_rate,
        'subscriber_rates': [r['rate'] for r in received]
    }

# Exécuter benchmarks
classic_results = benchmark_pubsub(use_sharded=False, num_channels=100, num_messages=10000)
time.sleep(5)

sharded_results = benchmark_pubsub(use_sharded=True, num_channels=100, num_messages=10000)

# Comparaison
print(f"\n{'=' * 60}")
print("COMPARISON")
print(f"{'=' * 60}")
improvement = (sharded_results['publish_rate'] / classic_results['publish_rate'] - 1) * 100
print(f"Throughput improvement: {improvement:+.1f}%")
```

---

## Quand utiliser Sharded Pub/Sub ?

### ✅ Cas d'usage idéaux

1. **Redis Cluster avec nombreux channels**
   - > 1000 channels actifs simultanément
   - Distribution géographique

2. **Multi-tenancy avec isolation**
   - Chaque tenant = channels dédiés
   - Performance prévisible par tenant

3. **High throughput avec scaling horizontal**
   - > 50K messages/sec
   - Besoin de scaler linéairement

4. **Microservices distribués**
   - Chaque service = subset de channels
   - Pas de broadcast global nécessaire

### ❌ Quand rester sur Classic Pub/Sub

1. **Redis Standalone**
   - Aucun avantage de Sharded Pub/Sub
   - Classic Pub/Sub plus simple

2. **Pattern matching requis**
   - PSUBSCRIBE essentiel
   - Sharded ne supporte pas patterns

3. **Peu de channels (< 100)**
   - Overhead de Sharded non justifié
   - Classic Pub/Sub suffisant

4. **Besoin de broadcast global**
   - Tous les subscribers doivent recevoir
   - Classic Pub/Sub plus adapté

---

## Tableau de décision final

| Critère | Classic Pub/Sub | Sharded Pub/Sub |
|---------|----------------|-----------------|
| **Architecture** | Standalone ou Cluster | **Cluster** uniquement optimal |
| **Nombre de channels** | < 1000 | **> 1000** |
| **Pattern matching** | **✅ Oui** | ❌ Non |
| **Throughput** | ~30K msg/s (cluster) | **~100K msg/s** (cluster) |
| **Network overhead** | Élevé (broadcast) | **Minimal** (local) |
| **Latence** | < 1ms | < 1ms |
| **Scalabilité** | Limitée | **Linéaire** |
| **Broadcast global** | **✅ Natif** | ⚠️ Manuel |
| **Version Redis** | Toutes | **7.0+** |
| **Complexité** | Simple | Moyenne |
| **Migration** | N/A | **Effort requis** |

---

## Checklist de production

Avant de déployer Sharded Pub/Sub :

- [ ] Redis 7.0+ sur TOUS les nœuds
- [ ] Redis Cluster configuré et fonctionnel
- [ ] Client libraries supportant Sharded Pub/Sub
- [ ] Aucune dépendance sur pattern matching (PSUBSCRIBE)
- [ ] Channels explicitement listés (pas de découverte dynamique)
- [ ] Plan de migration testé (dual publish si nécessaire)
- [ ] Monitoring adapté (PUBSUB SHARDCHANNELS, SHARDNUMSUB)
- [ ] Tests de charge validés
- [ ] Rollback plan préparé
- [ ] Documentation mise à jour

---

## Conclusion

**Sharded Pub/Sub** est une évolution majeure qui résout les limitations de scalabilité du Pub/Sub classique dans Redis Cluster, mais introduit ses propres contraintes.

### Quand migrer ?

```
Nb channels actifs > 1000 ET (Redis Cluster OU besoin de scaler)
    └─> Sharded Pub/Sub

Pattern matching requis OU Redis Standalone
    └─> Classic Pub/Sub

Garanties de livraison requises
    └─> Redis Streams
```

### Règle d'or

- **< 1000 channels** → Classic Pub/Sub suffit
- **> 1000 channels + Cluster** → Migrer vers Sharded Pub/Sub
- **Besoin de garanties** → Utiliser Streams, pas Pub/Sub

**Points clés** :
- 🚀 Performance x3 en cluster
- 📉 Network overhead -90%
- ❌ Pas de pattern matching
- ✅ Scalabilité linéaire
- 🎯 Optimisé pour multi-tenancy et microservices

---


⏭️ [Redis Streams : Introduction et concepts fondamentaux](/08-communication-flux-donnees/03-redis-streams-introduction.md)
