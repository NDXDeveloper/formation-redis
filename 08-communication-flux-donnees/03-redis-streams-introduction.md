🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.3 Redis Streams : Introduction et Concepts Fondamentaux

## Introduction

Redis Streams, introduit dans Redis 5.0 (2018), est une structure de données qui combine les avantages des logs immuables (comme Kafka) avec la simplicité et la performance de Redis. C'est la réponse de Redis au besoin de messaging fiable et persistant avec garanties de livraison.

### Pourquoi Redis Streams existe ?

```
Avant Redis Streams :
├─ Pub/Sub : Ultra-rapide mais aucune persistance
├─ Lists : Persistance mais pas de consumer groups
└─ Besoin : Kafka-like features avec simplicité Redis

Après Redis Streams :
✅ Persistance complète (RDB/AOF)
✅ Consumer groups natifs
✅ ACK/NACK explicites
✅ Rejouabilité illimitée
✅ Performance proche du Pub/Sub
```

### Le problème résolu

```python
# ❌ AVANT : Pub/Sub + List pour fiabilité
# Complex, fragile, dupplication de données

# Publisher
redis.publish('orders', order_data)  # Temps réel
redis.lpush('orders:backup', order_data)  # Backup

# Subscriber
pubsub.subscribe('orders')
# Si crash → perte de message
# Doit aussi lire de 'orders:backup'

# ✅ APRÈS : Redis Streams unifié
redis.xadd('orders', order_data)
# → Persisté automatiquement
# → Rejouable à volonté
# → Consumer groups natifs
# → ACK explicites
```

---

## Architecture et concepts fondamentaux

### Le modèle : Append-only log

```
Stream : orders:stream

┌────────────────────────────────────────────────────────────┐
│ 1234567890000-0   1234567890001-0   1234567890002-0        │
│ order_id: 123     order_id: 124     order_id: 125          │
│ amount: 99.99     amount: 149.99    amount: 299.99         │
│ status: pending   status: pending   status: pending        │
│                                                            │
│ ◄───────────────── Append only ────────────────────►       │
│ Oldest                                          Newest     │
└────────────────────────────────────────────────────────────┘

Caractéristiques :
✅ Append-only (immutable)
✅ Ordre strict garanti
✅ IDs uniques auto-générés
✅ Accès random par ID
✅ Range queries efficaces
```

### Anatomie d'une entrée (Entry)

```bash
# Structure d'une entrée
Entry ID: 1234567890123-0
    │         │        │
    │         │        └─ Sequence number (in millisecond)
    │         └────────── Timestamp (milliseconds)
    └──────────────────── Auto-generated unique ID

# Exemple complet
1702301234567-0   # Entry ID
    ├─ field1: value1
    ├─ field2: value2
    └─ field3: value3
```

**Propriétés des IDs** :
- Format : `<millisecondsTime>-<sequenceNumber>`
- Générés automatiquement par Redis
- Strictement monotones (croissants)
- Garantissent l'ordre chronologique
- Peuvent être spécifiés manuellement (rare)

### Comparaison avec autres structures

| Aspect | Pub/Sub | Lists | **Streams** | Kafka |
|--------|---------|-------|------------|-------|
| **Persistance** | ❌ Non | ✅ Oui | ✅ Oui | ✅ Oui |
| **Ordre** | ✅ Oui | ✅ FIFO | ✅ Strict | ✅ Par partition |
| **IDs uniques** | ❌ Non | ❌ Non | ✅ Auto | ✅ Offset |
| **Consumer groups** | ❌ Non | ❌ Non | ✅ Natif | ✅ Natif |
| **Rejouabilité** | ❌ Non | ❌ Non | ✅ Illimitée | ✅ Illimitée |
| **ACK/NACK** | ❌ Non | ⚠️ Implicite | ✅ Explicite | ✅ Explicite |
| **Range queries** | ❌ Non | ⚠️ Limité | ✅ Efficace | ❌ Non |
| **Latence** | < 1ms | ~1ms | ~1-2ms | ~5-50ms |
| **Scalabilité** | Limitée | Limitée | Moyenne | Excellente |
| **Complexité ops** | Très faible | Très faible | Faible | Élevée |

---

## Commandes fondamentales

### XADD : Ajouter une entrée

```bash
# Syntaxe de base
XADD stream_name ID field1 value1 [field2 value2 ...]

# Exemples
# 1. ID auto-généré (recommandé : *)
XADD orders:stream * order_id 123 amount 99.99 status pending
# Retourne : "1702301234567-0"

# 2. Avec timestamp spécifique
XADD events:stream 1702301234567-0 event login user_id 456
# Retourne : "1702301234567-0"

# 3. Avec MAXLEN (limiter taille)
XADD logs:stream MAXLEN 1000 * level error message "Connection failed"
# Garde seulement 1000 entrées les plus récentes

# 4. Avec MAXLEN approximatif (plus performant)
XADD metrics:stream MAXLEN ~ 10000 * cpu 45.2 memory 78.5
# ~ = approximatif, permet batching des trimming

# 5. MINID pour trimmer par temps
XADD events:stream MINID 1702301234567-0 * event logout user_id 789
# Supprime toutes les entrées < MINID

# 6. NOMKSTREAM : ne pas créer stream si inexistant
XADD mystream NOMKSTREAM * field value
# Retourne nil si stream n'existe pas
```

**Options importantes** :
- `*` : ID auto-généré (recommandé)
- `MAXLEN` : Limiter le nombre d'entrées
- `MAXLEN ~` : Approximatif, plus performant
- `MINID` : Trimmer par ID minimum
- `NOMKSTREAM` : Ne pas créer automatiquement

### XREAD : Lire des entrées

```bash
# Syntaxe
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]

# 1. Lire toutes les entrées depuis le début
XREAD STREAMS orders:stream 0-0
# Retourne toutes les entrées

# 2. Lire avec limite
XREAD COUNT 10 STREAMS orders:stream 0-0
# Retourne max 10 entrées

# 3. Lire depuis un ID spécifique
XREAD STREAMS orders:stream 1702301234567-0
# Retourne entrées APRÈS cet ID

# 4. Lire les nouvelles entrées uniquement
XREAD STREAMS orders:stream $
# $ = ID du dernier message actuel

# 5. Mode blocking (attendre nouveaux messages)
XREAD BLOCK 5000 STREAMS orders:stream $
# Bloque max 5 secondes en attendant

# 6. Blocking infini
XREAD BLOCK 0 STREAMS orders:stream $
# Bloque indéfiniment jusqu'à nouveau message

# 7. Lire depuis plusieurs streams
XREAD STREAMS orders:stream events:stream 0-0 0-0
# Retourne :
# 1) 1) "orders:stream"
#    2) [[entries]]
# 2) 1) "events:stream"
#    2) [[entries]]

# Exemple de retour
1) 1) "orders:stream"
   2) 1) 1) "1702301234567-0"
         2) 1) "order_id"
            2) "123"
            3) "amount"
            4) "99.99"
      2) 1) "1702301234568-0"
         2) 1) "order_id"
            2) "124"
```

### XRANGE : Range queries

```bash
# Syntaxe
XRANGE key start end [COUNT count]

# 1. Toutes les entrées
XRANGE orders:stream - +
# - = min ID, + = max ID

# 2. Range spécifique
XRANGE orders:stream 1702301234000-0 1702301235000-0
# Entrées entre ces deux IDs

# 3. Avec limite
XRANGE orders:stream - + COUNT 100
# Max 100 entrées

# 4. Depuis timestamp spécifique
XRANGE orders:stream 1702301234000 +
# Depuis ce timestamp

# 5. Incomplete IDs (Redis complète automatiquement)
XRANGE orders:stream 1702301234 1702301235
# Équivalent à 1702301234000-0 et 1702301235999-9999

# Exemple pratique : dernières 24h
# timestamp = now - 86400000 (24h en ms)
XRANGE events:stream ($(date +%s000) - 86400000) +
```

### XREVRANGE : Reverse range

```bash
# Comme XRANGE mais ordre inversé (newest first)
XREVRANGE orders:stream + -
# Retourne entrées de la plus récente à la plus ancienne

# Les 10 dernières entrées
XREVRANGE orders:stream + - COUNT 10

# Utile pour "recent items"
XREVRANGE logs:stream + - COUNT 100
```

### XLEN : Longueur du stream

```bash
# Nombre d'entrées dans le stream
XLEN orders:stream
# Retourne : (integer) 1523

# O(1) operation (très rapide)
```

### XDEL : Supprimer des entrées

```bash
# Syntaxe
XDEL key ID [ID ...]

# Supprimer une entrée
XDEL orders:stream 1702301234567-0
# Retourne : (integer) 1

# Supprimer plusieurs entrées
XDEL orders:stream 1702301234567-0 1702301234568-0 1702301234569-0
# Retourne : (integer) 3

# Note : Crée des "gaps" dans le stream
# Les IDs supprimés ne sont pas réutilisés
```

### XTRIM : Trimmer le stream

```bash
# Syntaxe
XTRIM key MAXLEN|MINID [=|~] threshold [LIMIT count]

# 1. Garder seulement N dernières entrées
XTRIM orders:stream MAXLEN 1000
# Garde les 1000 plus récentes

# 2. Approximatif (plus performant)
XTRIM orders:stream MAXLEN ~ 1000
# Garde ~1000 (peut être légèrement plus)

# 3. Trimmer par ID minimum
XTRIM orders:stream MINID 1702301234567-0
# Supprime toutes les entrées < cet ID

# 4. Avec LIMIT (éviter blocking long)
XTRIM orders:stream MAXLEN ~ 1000 LIMIT 100
# Supprime max 100 entrées par opération
```

### XINFO : Introspection

```bash
# 1. Informations générales sur le stream
XINFO STREAM orders:stream
# Retourne : length, first-entry, last-entry, groups, etc.

# 2. Informations sur consumer groups
XINFO GROUPS orders:stream
# Liste des groups sur ce stream

# 3. Informations sur consumers d'un group
XINFO CONSUMERS orders:stream mygroup
# Liste des consumers actifs

# 4. Aide sur commandes XINFO
XINFO HELP
```

---

## Implémentation par langage

### Python avec redis-py

```python
import redis
import time
import json
from typing import Dict, List, Optional

# ============================================
# Producer (XADD)
# ============================================
class StreamProducer:
    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name

    def add_message(self, data: Dict, max_len: Optional[int] = None) -> str:
        """Ajouter un message au stream"""
        # XADD avec ID auto-généré (*)
        if max_len:
            message_id = self.redis.xadd(
                self.stream_name,
                data,
                maxlen=max_len,
                approximate=True  # ~ pour performance
            )
        else:
            message_id = self.redis.xadd(self.stream_name, data)

        print(f"✓ Added message {message_id}")
        return message_id

    def add_order(self, order_id: int, amount: float, status: str) -> str:
        """Ajouter une commande"""
        data = {
            'order_id': str(order_id),
            'amount': str(amount),
            'status': status,
            'timestamp': str(time.time())
        }
        return self.add_message(data)

    def add_event(self, event_type: str, user_id: int, details: Dict) -> str:
        """Ajouter un événement"""
        data = {
            'event_type': event_type,
            'user_id': str(user_id),
            'details': json.dumps(details),
            'timestamp': str(time.time())
        }
        return self.add_message(data)

# ============================================
# Consumer simple (XREAD)
# ============================================
class SimpleStreamConsumer:
    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.last_id = '0-0'  # Commencer au début

    def read_new_messages(self, count: int = 10, block: Optional[int] = None):
        """Lire nouveaux messages depuis last_id"""
        if block is not None:
            # Mode blocking
            messages = self.redis.xread(
                {self.stream_name: self.last_id},
                count=count,
                block=block
            )
        else:
            # Mode non-blocking
            messages = self.redis.xread(
                {self.stream_name: self.last_id},
                count=count
            )

        if messages:
            stream_messages = messages[0][1]  # Extract messages

            for msg_id, data in stream_messages:
                self.process_message(msg_id, data)
                self.last_id = msg_id  # Update last processed

        return len(messages[0][1]) if messages else 0

    def process_message(self, msg_id: str, data: Dict):
        """Traiter un message"""
        print(f"[{msg_id}] Processing: {data}")

    def listen_forever(self):
        """Écouter en continu (blocking mode)"""
        print(f"Listening to {self.stream_name}...")

        while True:
            try:
                # Block 5 secondes en attendant
                self.read_new_messages(count=10, block=5000)
            except KeyboardInterrupt:
                print("\nStopping consumer...")
                break
            except Exception as e:
                print(f"Error: {e}")
                time.sleep(1)

# ============================================
# Range queries
# ============================================
class StreamReader:
    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name

    def read_all(self, count: Optional[int] = None) -> List:
        """Lire toutes les entrées"""
        if count:
            messages = self.redis.xrange(self.stream_name, '-', '+', count=count)
        else:
            messages = self.redis.xrange(self.stream_name, '-', '+')

        return messages

    def read_range(self, start_id: str, end_id: str, count: Optional[int] = None) -> List:
        """Lire range spécifique"""
        if count:
            return self.redis.xrange(self.stream_name, start_id, end_id, count=count)
        return self.redis.xrange(self.stream_name, start_id, end_id)

    def read_since(self, since_id: str, count: Optional[int] = None) -> List:
        """Lire depuis un ID"""
        return self.read_range(since_id, '+', count)

    def read_last_n(self, n: int) -> List:
        """Lire les N derniers messages"""
        return self.redis.xrevrange(self.stream_name, '+', '-', count=n)

    def read_last_24h(self) -> List:
        """Lire dernières 24h"""
        # Calculer timestamp il y a 24h
        timestamp_24h_ago = int((time.time() - 86400) * 1000)
        start_id = f"{timestamp_24h_ago}-0"

        return self.redis.xrange(self.stream_name, start_id, '+')

    def get_stream_info(self) -> Dict:
        """Obtenir infos sur le stream"""
        length = self.redis.xlen(self.stream_name)

        # Premier et dernier message
        first = self.redis.xrange(self.stream_name, '-', '+', count=1)
        last = self.redis.xrevrange(self.stream_name, '+', '-', count=1)

        return {
            'length': length,
            'first_id': first[0][0] if first else None,
            'last_id': last[0][0] if last else None,
            'first_data': first[0][1] if first else None,
            'last_data': last[0][1] if last else None
        }

# ============================================
# Stream management
# ============================================
class StreamManager:
    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name

    def trim_by_length(self, max_len: int, approximate: bool = True):
        """Trimmer par longueur"""
        if approximate:
            deleted = self.redis.xtrim(self.stream_name, maxlen=max_len, approximate=True)
        else:
            deleted = self.redis.xtrim(self.stream_name, maxlen=max_len)

        print(f"Trimmed {deleted} messages")
        return deleted

    def trim_by_id(self, min_id: str):
        """Trimmer par ID minimum"""
        deleted = self.redis.xtrim(self.stream_name, minid=min_id)
        print(f"Trimmed {deleted} messages before {min_id}")
        return deleted

    def trim_old_messages(self, hours: int):
        """Supprimer messages plus vieux que N heures"""
        timestamp = int((time.time() - hours * 3600) * 1000)
        min_id = f"{timestamp}-0"

        return self.trim_by_id(min_id)

    def delete_message(self, message_id: str):
        """Supprimer un message spécifique"""
        deleted = self.redis.xdel(self.stream_name, message_id)
        return deleted > 0

    def get_length(self) -> int:
        """Obtenir longueur du stream"""
        return self.redis.xlen(self.stream_name)

# ============================================
# Exemple d'utilisation
# ============================================
if __name__ == "__main__":
    # Producer
    producer = StreamProducer('orders:stream')

    # Ajouter quelques commandes
    for i in range(5):
        producer.add_order(
            order_id=1000 + i,
            amount=99.99 * (i + 1),
            status='pending'
        )
        time.sleep(0.1)

    # Reader
    reader = StreamReader('orders:stream')

    # Lire toutes les entrées
    print("\n=== All messages ===")
    all_messages = reader.read_all()
    for msg_id, data in all_messages:
        print(f"{msg_id}: {data}")

    # Infos sur le stream
    print("\n=== Stream info ===")
    info = reader.get_stream_info()
    print(f"Length: {info['length']}")
    print(f"First: {info['first_id']}")
    print(f"Last: {info['last_id']}")

    # Consumer en mode blocking
    print("\n=== Starting consumer ===")
    consumer = SimpleStreamConsumer('orders:stream')
    consumer.last_id = info['last_id']  # Commencer après le dernier

    # Écouter pendant 10 secondes
    import threading

    def consume():
        consumer.read_new_messages(count=10, block=10000)

    thread = threading.Thread(target=consume)
    thread.start()

    # Publier pendant que consumer écoute
    time.sleep(1)
    producer.add_order(1005, 599.99, 'pending')

    thread.join()
```

### Node.js avec ioredis

```javascript
const Redis = require('ioredis');

// ============================================
// Producer (XADD)
// ============================================
class StreamProducer {
    constructor(streamName) {
        this.redis = new Redis();
        this.streamName = streamName;
    }

    async addMessage(data, maxLen = null) {
        let args = [this.streamName];

        if (maxLen) {
            args.push('MAXLEN', '~', maxLen);
        }

        args.push('*');  // Auto-generated ID

        // Ajouter field-value pairs
        for (const [key, value] of Object.entries(data)) {
            args.push(key, String(value));
        }

        const messageId = await this.redis.xadd(...args);
        console.log(`✓ Added message ${messageId}`);

        return messageId;
    }

    async addOrder(orderId, amount, status) {
        const data = {
            order_id: orderId,
            amount: amount,
            status: status,
            timestamp: Date.now()
        };

        return await this.addMessage(data);
    }

    async addEvent(eventType, userId, details) {
        const data = {
            event_type: eventType,
            user_id: userId,
            details: JSON.stringify(details),
            timestamp: Date.now()
        };

        return await this.addMessage(data);
    }

    async close() {
        await this.redis.quit();
    }
}

// ============================================
// Consumer simple (XREAD)
// ============================================
class SimpleStreamConsumer {
    constructor(streamName) {
        this.redis = new Redis();
        this.streamName = streamName;
        this.lastId = '0-0';  // Start from beginning
    }

    async readNewMessages(count = 10, block = null) {
        const args = {
            count: count,
            streams: [this.streamName],
            key: [this.lastId]
        };

        if (block !== null) {
            args.block = block;
        }

        const results = await this.redis.xread(
            'COUNT', count,
            ...(block !== null ? ['BLOCK', block] : []),
            'STREAMS', this.streamName, this.lastId
        );

        if (results && results.length > 0) {
            const [streamName, messages] = results[0];

            for (const [messageId, fields] of messages) {
                // Convert fields array to object
                const data = {};
                for (let i = 0; i < fields.length; i += 2) {
                    data[fields[i]] = fields[i + 1];
                }

                await this.processMessage(messageId, data);
                this.lastId = messageId;
            }

            return messages.length;
        }

        return 0;
    }

    async processMessage(messageId, data) {
        console.log(`[${messageId}] Processing:`, data);
    }

    async listenForever() {
        console.log(`Listening to ${this.streamName}...`);

        while (true) {
            try {
                await this.readNewMessages(10, 5000);  // Block 5 seconds
            } catch (err) {
                console.error('Error:', err);
                await new Promise(resolve => setTimeout(resolve, 1000));
            }
        }
    }

    async close() {
        await this.redis.quit();
    }
}

// ============================================
// Range queries
// ============================================
class StreamReader {
    constructor(streamName) {
        this.redis = new Redis();
        this.streamName = streamName;
    }

    async readAll(count = null) {
        const args = [this.streamName, '-', '+'];
        if (count) {
            args.push('COUNT', count);
        }

        return await this.redis.xrange(...args);
    }

    async readRange(startId, endId, count = null) {
        const args = [this.streamName, startId, endId];
        if (count) {
            args.push('COUNT', count);
        }

        return await this.redis.xrange(...args);
    }

    async readSince(sinceId, count = null) {
        return await this.readRange(sinceId, '+', count);
    }

    async readLastN(n) {
        return await this.redis.xrevrange(this.streamName, '+', '-', 'COUNT', n);
    }

    async readLast24h() {
        const timestamp24hAgo = Date.now() - 86400000;
        const startId = `${timestamp24hAgo}-0`;

        return await this.redis.xrange(this.streamName, startId, '+');
    }

    async getStreamInfo() {
        const length = await this.redis.xlen(this.streamName);

        const first = await this.redis.xrange(this.streamName, '-', '+', 'COUNT', 1);
        const last = await this.redis.xrevrange(this.streamName, '+', '-', 'COUNT', 1);

        return {
            length: length,
            firstId: first[0] ? first[0][0] : null,
            lastId: last[0] ? last[0][0] : null,
            firstData: first[0] ? first[0][1] : null,
            lastData: last[0] ? last[0][1] : null
        };
    }

    async close() {
        await this.redis.quit();
    }
}

// ============================================
// Stream management
// ============================================
class StreamManager {
    constructor(streamName) {
        this.redis = new Redis();
        this.streamName = streamName;
    }

    async trimByLength(maxLen, approximate = true) {
        const args = [this.streamName, 'MAXLEN'];

        if (approximate) {
            args.push('~');
        }

        args.push(maxLen);

        const deleted = await this.redis.xtrim(...args);
        console.log(`Trimmed ${deleted} messages`);

        return deleted;
    }

    async trimById(minId) {
        const deleted = await this.redis.xtrim(this.streamName, 'MINID', minId);
        console.log(`Trimmed ${deleted} messages before ${minId}`);

        return deleted;
    }

    async trimOldMessages(hours) {
        const timestamp = Date.now() - (hours * 3600000);
        const minId = `${timestamp}-0`;

        return await this.trimById(minId);
    }

    async deleteMessage(messageId) {
        const deleted = await this.redis.xdel(this.streamName, messageId);
        return deleted > 0;
    }

    async getLength() {
        return await this.redis.xlen(this.streamName);
    }

    async close() {
        await this.redis.quit();
    }
}

// ============================================
// Exemple d'utilisation
// ============================================
async function main() {
    // Producer
    const producer = new StreamProducer('orders:stream');

    // Ajouter quelques commandes
    console.log('=== Adding orders ===');
    for (let i = 0; i < 5; i++) {
        await producer.addOrder(1000 + i, 99.99 * (i + 1), 'pending');
        await new Promise(resolve => setTimeout(resolve, 100));
    }

    // Reader
    const reader = new StreamReader('orders:stream');

    // Lire toutes les entrées
    console.log('\n=== All messages ===');
    const allMessages = await reader.readAll();
    for (const [messageId, fields] of allMessages) {
        console.log(`${messageId}:`, fields);
    }

    // Infos sur le stream
    console.log('\n=== Stream info ===');
    const info = await reader.getStreamInfo();
    console.log(`Length: ${info.length}`);
    console.log(`First: ${info.firstId}`);
    console.log(`Last: ${info.lastId}`);

    // Consumer
    console.log('\n=== Starting consumer ===');
    const consumer = new SimpleStreamConsumer('orders:stream');
    consumer.lastId = info.lastId;  // Start after last

    // Listen for 5 seconds
    setTimeout(async () => {
        await producer.addOrder(1005, 599.99, 'pending');
    }, 1000);

    await consumer.readNewMessages(10, 5000);

    // Cleanup
    await producer.close();
    await reader.close();
    await consumer.close();
}

main().catch(console.error);
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
// Producer (XADD)
// ============================================
type StreamProducer struct {
    client     *redis.Client
    streamName string
}

func NewStreamProducer(streamName string) *StreamProducer {
    return &StreamProducer{
        client: redis.NewClient(&redis.Options{
            Addr: "localhost:6379",
        }),
        streamName: streamName,
    }
}

func (p *StreamProducer) AddMessage(ctx context.Context, data map[string]interface{}, maxLen int64) (string, error) {
    args := &redis.XAddArgs{
        Stream: p.streamName,
        Values: data,
    }

    if maxLen > 0 {
        args.MaxLen = maxLen
        args.Approx = true
    }

    messageID, err := p.client.XAdd(ctx, args).Result()
    if err != nil {
        return "", err
    }

    fmt.Printf("✓ Added message %s\n", messageID)
    return messageID, nil
}

func (p *StreamProducer) AddOrder(ctx context.Context, orderID int, amount float64, status string) (string, error) {
    data := map[string]interface{}{
        "order_id":  orderID,
        "amount":    amount,
        "status":    status,
        "timestamp": time.Now().Unix(),
    }

    return p.AddMessage(ctx, data, 0)
}

func (p *StreamProducer) AddEvent(ctx context.Context, eventType string, userID int, details map[string]interface{}) (string, error) {
    detailsJSON, _ := json.Marshal(details)

    data := map[string]interface{}{
        "event_type": eventType,
        "user_id":    userID,
        "details":    string(detailsJSON),
        "timestamp":  time.Now().Unix(),
    }

    return p.AddMessage(ctx, data, 0)
}

func (p *StreamProducer) Close() error {
    return p.client.Close()
}

// ============================================
// Consumer simple (XREAD)
// ============================================
type SimpleStreamConsumer struct {
    client     *redis.Client
    streamName string
    lastID     string
}

func NewSimpleStreamConsumer(streamName string) *SimpleStreamConsumer {
    return &SimpleStreamConsumer{
        client: redis.NewClient(&redis.Options{
            Addr: "localhost:6379",
        }),
        streamName: streamName,
        lastID:     "0-0",
    }
}

func (c *SimpleStreamConsumer) ReadNewMessages(ctx context.Context, count int64, block time.Duration) (int, error) {
    args := &redis.XReadArgs{
        Streams: []string{c.streamName, c.lastID},
        Count:   count,
        Block:   block,
    }

    results, err := c.client.XRead(ctx, args).Result()
    if err != nil {
        if err == redis.Nil {
            return 0, nil  // No messages
        }
        return 0, err
    }

    if len(results) > 0 {
        messages := results[0].Messages

        for _, msg := range messages {
            c.ProcessMessage(msg.ID, msg.Values)
            c.lastID = msg.ID
        }

        return len(messages), nil
    }

    return 0, nil
}

func (c *SimpleStreamConsumer) ProcessMessage(messageID string, data map[string]interface{}) {
    fmt.Printf("[%s] Processing: %v\n", messageID, data)
}

func (c *SimpleStreamConsumer) ListenForever(ctx context.Context) {
    fmt.Printf("Listening to %s...\n", c.streamName)

    for {
        select {
        case <-ctx.Done():
            return
        default:
            _, err := c.ReadNewMessages(ctx, 10, 5*time.Second)
            if err != nil {
                log.Printf("Error: %v", err)
                time.Sleep(1 * time.Second)
            }
        }
    }
}

func (c *SimpleStreamConsumer) Close() error {
    return c.client.Close()
}

// ============================================
// Range queries
// ============================================
type StreamReader struct {
    client     *redis.Client
    streamName string
}

func NewStreamReader(streamName string) *StreamReader {
    return &StreamReader{
        client: redis.NewClient(&redis.Options{
            Addr: "localhost:6379",
        }),
        streamName: streamName,
    }
}

func (r *StreamReader) ReadAll(ctx context.Context, count int64) ([]redis.XMessage, error) {
    args := &redis.XRangeArgs{
        Stream: r.streamName,
        Start:  "-",
        Stop:   "+",
    }

    if count > 0 {
        args.Count = count
    }

    return r.client.XRange(ctx, args.Stream, args.Start, args.Stop).Result()
}

func (r *StreamReader) ReadRange(ctx context.Context, startID, endID string, count int64) ([]redis.XMessage, error) {
    return r.client.XRange(ctx, r.streamName, startID, endID).Result()
}

func (r *StreamReader) ReadLastN(ctx context.Context, n int64) ([]redis.XMessage, error) {
    return r.client.XRevRange(ctx, r.streamName, "+", "-").Result()
}

func (r *StreamReader) ReadLast24h(ctx context.Context) ([]redis.XMessage, error) {
    timestamp24hAgo := time.Now().Add(-24 * time.Hour).UnixMilli()
    startID := fmt.Sprintf("%d-0", timestamp24hAgo)

    return r.client.XRange(ctx, r.streamName, startID, "+").Result()
}

func (r *StreamReader) GetStreamInfo(ctx context.Context) (map[string]interface{}, error) {
    length, err := r.client.XLen(ctx, r.streamName).Result()
    if err != nil {
        return nil, err
    }

    first, _ := r.client.XRange(ctx, r.streamName, "-", "+").Result()
    last, _ := r.client.XRevRange(ctx, r.streamName, "+", "-").Result()

    info := map[string]interface{}{
        "length": length,
    }

    if len(first) > 0 {
        info["first_id"] = first[0].ID
        info["first_data"] = first[0].Values
    }

    if len(last) > 0 {
        info["last_id"] = last[0].ID
        info["last_data"] = last[0].Values
    }

    return info, nil
}

func (r *StreamReader) Close() error {
    return r.client.Close()
}

// ============================================
// Stream management
// ============================================
type StreamManager struct {
    client     *redis.Client
    streamName string
}

func NewStreamManager(streamName string) *StreamManager {
    return &StreamManager{
        client: redis.NewClient(&redis.Options{
            Addr: "localhost:6379",
        }),
        streamName: streamName,
    }
}

func (m *StreamManager) TrimByLength(ctx context.Context, maxLen int64, approximate bool) (int64, error) {
    args := &redis.XTrimArgs{
        MaxLen: maxLen,
        Approx: approximate,
    }

    deleted, err := m.client.XTrimMaxLen(ctx, m.streamName, maxLen).Result()
    if err != nil {
        return 0, err
    }

    fmt.Printf("Trimmed %d messages\n", deleted)
    return deleted, nil
}

func (m *StreamManager) TrimByID(ctx context.Context, minID string) (int64, error) {
    deleted, err := m.client.XTrimMinID(ctx, m.streamName, minID).Result()
    if err != nil {
        return 0, err
    }

    fmt.Printf("Trimmed %d messages before %s\n", deleted, minID)
    return deleted, nil
}

func (m *StreamManager) TrimOldMessages(ctx context.Context, hours int) (int64, error) {
    timestamp := time.Now().Add(-time.Duration(hours) * time.Hour).UnixMilli()
    minID := fmt.Sprintf("%d-0", timestamp)

    return m.TrimByID(ctx, minID)
}

func (m *StreamManager) DeleteMessage(ctx context.Context, messageID string) (bool, error) {
    deleted, err := m.client.XDel(ctx, m.streamName, messageID).Result()
    return deleted > 0, err
}

func (m *StreamManager) GetLength(ctx context.Context) (int64, error) {
    return m.client.XLen(ctx, m.streamName).Result()
}

func (m *StreamManager) Close() error {
    return m.client.Close()
}

// ============================================
// Main
// ============================================
func main() {
    ctx := context.Background()

    // Producer
    producer := NewStreamProducer("orders:stream")
    defer producer.Close()

    // Add some orders
    fmt.Println("=== Adding orders ===")
    for i := 0; i < 5; i++ {
        producer.AddOrder(ctx, 1000+i, 99.99*float64(i+1), "pending")
        time.Sleep(100 * time.Millisecond)
    }

    // Reader
    reader := NewStreamReader("orders:stream")
    defer reader.Close()

    // Read all
    fmt.Println("\n=== All messages ===")
    allMessages, _ := reader.ReadAll(ctx, 0)
    for _, msg := range allMessages {
        fmt.Printf("%s: %v\n", msg.ID, msg.Values)
    }

    // Stream info
    fmt.Println("\n=== Stream info ===")
    info, _ := reader.GetStreamInfo(ctx)
    fmt.Printf("Length: %v\n", info["length"])
    fmt.Printf("First: %v\n", info["first_id"])
    fmt.Printf("Last: %v\n", info["last_id"])

    // Consumer
    fmt.Println("\n=== Starting consumer ===")
    consumer := NewSimpleStreamConsumer("orders:stream")
    defer consumer.Close()

    consumer.lastID = info["last_id"].(string)

    // Listen for 5 seconds
    go func() {
        time.Sleep(1 * time.Second)
        producer.AddOrder(ctx, 1005, 599.99, "pending")
    }()

    consumer.ReadNewMessages(ctx, 10, 5*time.Second)
}
```

---

## Patterns de base

### Pattern 1 : Event Sourcing

```python
class EventStore:
    """Store d'événements avec Redis Streams"""

    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)

    def append_event(self, aggregate_id: str, event_type: str, data: Dict) -> str:
        """Ajouter un événement à l'event store"""
        stream_name = f"events:{aggregate_id}"

        event = {
            'event_type': event_type,
            'data': json.dumps(data),
            'timestamp': str(time.time())
        }

        message_id = self.redis.xadd(stream_name, event)

        return message_id

    def get_events(self, aggregate_id: str) -> List:
        """Récupérer tous les événements d'un aggregate"""
        stream_name = f"events:{aggregate_id}"

        events = self.redis.xrange(stream_name, '-', '+')

        result = []
        for msg_id, data in events:
            result.append({
                'id': msg_id,
                'event_type': data['event_type'],
                'data': json.loads(data['data']),
                'timestamp': float(data['timestamp'])
            })

        return result

    def replay_events(self, aggregate_id: str, until_id: Optional[str] = None):
        """Rejouer événements jusqu'à un ID spécifique"""
        stream_name = f"events:{aggregate_id}"

        end_id = until_id or '+'
        events = self.redis.xrange(stream_name, '-', end_id)

        return events

# Utilisation
store = EventStore()

# Créer une série d'événements pour un user
store.append_event('user:123', 'user.created', {'name': 'John', 'email': 'john@example.com'})
store.append_event('user:123', 'email.changed', {'old': 'john@example.com', 'new': 'john.doe@example.com'})
store.append_event('user:123', 'profile.updated', {'field': 'phone', 'value': '+1234567890'})

# Récupérer historique complet
events = store.get_events('user:123')
for event in events:
    print(f"{event['id']}: {event['event_type']} - {event['data']}")
```

### Pattern 2 : Time-series data

```python
class TimeSeriesStream:
    """Série temporelle avec Redis Streams"""

    def __init__(self, metric_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = f"metrics:{metric_name}"

    def add_datapoint(self, value: float, tags: Dict = None) -> str:
        """Ajouter un point de données"""
        data = {
            'value': str(value),
            'timestamp': str(time.time())
        }

        if tags:
            data['tags'] = json.dumps(tags)

        # Auto-trim pour garder seulement 24h de données
        message_id = self.redis.xadd(
            self.stream_name,
            data,
            maxlen=100000,  # ~1 point/seconde pendant 24h
            approximate=True
        )

        return message_id

    def get_range(self, start_time: float, end_time: float) -> List:
        """Récupérer points dans une fenêtre de temps"""
        start_id = f"{int(start_time * 1000)}-0"
        end_id = f"{int(end_time * 1000)}-9999"

        messages = self.redis.xrange(self.stream_name, start_id, end_id)

        result = []
        for msg_id, data in messages:
            result.append({
                'timestamp': float(data['timestamp']),
                'value': float(data['value']),
                'tags': json.loads(data.get('tags', '{}'))
            })

        return result

    def get_latest(self, n: int = 10) -> List:
        """Récupérer les N derniers points"""
        messages = self.redis.xrevrange(self.stream_name, '+', '-', count=n)

        result = []
        for msg_id, data in messages:
            result.append({
                'id': msg_id,
                'timestamp': float(data['timestamp']),
                'value': float(data['value'])
            })

        return result

# Utilisation
cpu_metrics = TimeSeriesStream('cpu_usage')

# Ajouter des métriques
for i in range(60):
    cpu_metrics.add_datapoint(
        value=45.0 + (i % 10),
        tags={'server': 'prod-1', 'region': 'us-east'}
    )
    time.sleep(0.1)

# Récupérer dernières 10 minutes
start = time.time() - 600
latest = cpu_metrics.get_range(start, time.time())
print(f"Got {len(latest)} datapoints from last 10 minutes")
```

### Pattern 3 : Audit Log

```python
class AuditLog:
    """Audit log immuable avec Redis Streams"""

    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = 'audit:log'

    def log_action(self, user_id: str, action: str, resource: str, details: Dict = None) -> str:
        """Logger une action utilisateur"""
        entry = {
            'user_id': user_id,
            'action': action,
            'resource': resource,
            'timestamp': str(time.time()),
            'ip_address': details.get('ip') if details else 'unknown'
        }

        if details:
            entry['details'] = json.dumps(details)

        message_id = self.redis.xadd(self.stream_name, entry)

        return message_id

    def get_user_activity(self, user_id: str, limit: int = 100) -> List:
        """Récupérer activité d'un user"""
        # Lire tout et filtrer (inefficace pour gros volumes)
        # Pour production, utiliser un index secondaire ou RediSearch

        all_logs = self.redis.xrevrange(self.stream_name, '+', '-', count=10000)

        user_logs = []
        for msg_id, data in all_logs:
            if data.get('user_id') == user_id:
                user_logs.append({
                    'id': msg_id,
                    'action': data['action'],
                    'resource': data['resource'],
                    'timestamp': float(data['timestamp']),
                    'details': json.loads(data.get('details', '{}'))
                })

                if len(user_logs) >= limit:
                    break

        return user_logs

    def get_audit_trail(self, resource: str, start_time: float = None) -> List:
        """Audit trail pour une ressource"""
        start_id = f"{int(start_time * 1000)}-0" if start_time else '-'

        all_logs = self.redis.xrange(self.stream_name, start_id, '+')

        resource_logs = []
        for msg_id, data in all_logs:
            if data.get('resource') == resource:
                resource_logs.append({
                    'id': msg_id,
                    'user_id': data['user_id'],
                    'action': data['action'],
                    'timestamp': float(data['timestamp'])
                })

        return resource_logs

# Utilisation
audit = AuditLog()

# Logger des actions
audit.log_action('user:123', 'read', 'document:456', {'ip': '192.168.1.1'})
audit.log_action('user:123', 'update', 'document:456', {'ip': '192.168.1.1', 'changes': ['title']})
audit.log_action('user:789', 'delete', 'document:456', {'ip': '10.0.0.5'})

# Audit trail
trail = audit.get_audit_trail('document:456')
for entry in trail:
    print(f"{entry['timestamp']}: {entry['user_id']} {entry['action']}")
```

---

## Quand utiliser Redis Streams ?

### ✅ Cas d'usage idéaux

1. **Event sourcing et CQRS**
   - Historique complet requis
   - Rejouabilité des événements

2. **Message queues avec garanties**
   - Traitement fiable
   - Consumer groups
   - Retry automatique

3. **Time-series légers**
   - IoT sensors
   - Application metrics
   - Pas besoin de RedisTimeSeries

4. **Audit logs**
   - Immuabilité
   - Compliance requirements

5. **Data pipelines**
   - ETL workflows
   - Multi-stage processing

### ❌ Quand ne PAS utiliser

1. **Ultra-faible latence requise (< 1ms)**
   - → Pub/Sub mieux adapté

2. **Pas besoin de persistance**
   - → Pub/Sub plus simple

3. **Très haut débit (> 1M msg/s)**
   - → Kafka plus adapté

4. **Queries complexes**
   - → Base de données traditionnelle

---

## Tableau de décision rapide

| Besoin | Solution recommandée |
|--------|---------------------|
| Notifications temps réel, perte acceptable | **Pub/Sub** |
| Messages critiques, pas de perte | **Streams** |
| Job queue simple, un consumer | **Lists + BRPOP** |
| Job queue, plusieurs workers | **Streams + Consumer Groups** |
| Event sourcing | **Streams** |
| Audit trail | **Streams** |
| Chat/messaging temps réel | **Pub/Sub** |
| Time-series (< 1M points) | **Streams** |
| Time-series (> 1M points) | **RedisTimeSeries** |
| Très haut débit (> 100K/s) | **Kafka** |

---

## Conclusion

Redis Streams représente un sweet spot entre la simplicité de Redis et les fonctionnalités avancées de systèmes comme Kafka :

**Forces** :
- ✅ Persistance complète
- ✅ Rejouabilité illimitée
- ✅ Consumer groups natifs (section suivante)
- ✅ Performance excellente
- ✅ Simple à opérer

**Limitations** :
- ⚠️ Scalabilité limitée vs Kafka
- ⚠️ Pas de partitionnement natif
- ⚠️ Range queries uniquement par ID

**Règle d'or** : Streams est idéal pour 80% des cas d'usage qui nécessitent fiabilité + performance, sans la complexité de Kafka.

---


⏭️ [Redis Streams : Consumer Groups et traitement parallèle](/08-communication-flux-donnees/04-redis-streams-consumer-groups.md)
