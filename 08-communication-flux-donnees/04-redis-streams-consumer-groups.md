🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.4 Redis Streams : Consumer Groups et Traitement Parallèle

## Introduction

Les **Consumer Groups** sont la fonctionnalité killer de Redis Streams, permettant un traitement parallèle, fiable et scalable des messages. Ils offrent des garanties similaires à Kafka avec une complexité opérationnelle bien moindre.

### Le problème sans Consumer Groups

```python
# ❌ PROBLÈME : Plusieurs consumers avec XREAD simple
# Consumer 1
messages = redis.xread({'orders': last_id})
# Process...

# Consumer 2 (même code)
messages = redis.xread({'orders': last_id})
# → Reçoit LES MÊMES messages que Consumer 1!

# Résultat : Travail dupliqué, pas de distribution de charge
```

### La solution : Consumer Groups

```python
# ✅ SOLUTION : Consumer Groups
# Consumer 1
messages = redis.xreadgroup('workers', 'consumer-1', {'orders': '>'})
# Reçoit : messages A, B, C

# Consumer 2
messages = redis.xreadgroup('workers', 'consumer-2', {'orders': '>'})
# Reçoit : messages D, E, F (DIFFÉRENTS de consumer-1)

# → Distribution automatique, pas de duplication
```

---

## Architecture des Consumer Groups

### Concepts fondamentaux

```
Stream: orders:stream
┌─────────────────────────────────────────────────────────┐
│ Message 1   Message 2   Message 3   Message 4   ...     │
└─────────────────────────────────────────────────────────┘
            │                   │
            ▼                   ▼
    ┌─────────────┐     ┌─────────────┐
    │ Consumer    │     │ Consumer    │
    │ Group:      │     │ Group:      │
    │ "workers"   │     │ "analytics" │
    └─────────────┘     └─────────────┘
         │                     │
    ┌────┴────┐            ┌───┴───┐
    ▼         ▼            ▼       ▼
Consumer-1 Consumer-2   Consumer-3 Consumer-4
(Msg 1,3) (Msg 2,4)    (Msg 1,2) (Msg 3,4)

Caractéristiques :
✅ Plusieurs groups sur même stream
✅ Chaque group voit TOUS les messages
✅ Dans un group, chaque message va à UN SEUL consumer
✅ Load balancing automatique
```

### États des messages

```
Message Lifecycle dans un Consumer Group:

1. PENDING
   ├─ Message livré à un consumer
   ├─ Pas encore ACKé
   └─ Peut être réclamé si timeout

2. ACKNOWLEDGED
   ├─ Consumer a ACKé le message
   ├─ Traitement confirmé
   └─ Retiré de PENDING

3. DEAD (optionnel)
   ├─ Trop de tentatives
   ├─ Déplacé vers DLQ
   └─ Intervention manuelle requise
```

### Comparaison avec autres systèmes

| Aspect | XREAD simple | **Consumer Groups** | Kafka Consumer Groups |
|--------|--------------|---------------------|----------------------|
| **Distribution** | ❌ Manuel | ✅ Automatique | ✅ Automatique |
| **Load balancing** | ❌ Non | ✅ Oui | ✅ Oui |
| **ACK explicites** | ❌ Non | ✅ Oui | ✅ Oui |
| **Retry automatique** | ❌ Non | ✅ Oui (XPENDING) | ✅ Oui |
| **Message tracking** | ❌ Non | ✅ Par consumer | ✅ Par partition |
| **Parallel processing** | ⚠️ Manuel | ✅ Natif | ✅ Natif |
| **Rejouabilité** | ✅ Oui | ✅ Oui | ✅ Oui |
| **Scalabilité** | Limitée | Moyenne | Excellente |
| **Complexité setup** | Très simple | Simple | Moyenne |

---

## Commandes Consumer Groups

### XGROUP CREATE : Créer un group

```bash
# Syntaxe
XGROUP CREATE stream groupname id|$ [MKSTREAM] [ENTRIESREAD entries-read]

# 1. Créer group depuis le début du stream
XGROUP CREATE orders:stream workers 0
# Lira tous les messages depuis le début

# 2. Créer group depuis la fin (messages futurs uniquement)
XGROUP CREATE orders:stream workers $
# $ = position courante, lira seulement nouveaux messages

# 3. Créer group avec création automatique du stream
XGROUP CREATE orders:stream workers $ MKSTREAM
# Si stream n'existe pas, le crée automatiquement

# 4. Créer group depuis un ID spécifique
XGROUP CREATE orders:stream workers 1702301234567-0
# Lira messages APRÈS cet ID

# Erreur si group existe déjà
# (error) BUSYGROUP Consumer Group name already exists
```

### XGROUP SETID : Changer position du group

```bash
# Syntaxe
XGROUP SETID stream groupname id|$

# 1. Revenir au début
XGROUP SETID orders:stream workers 0
# Group relira tous les messages

# 2. Sauter à la fin
XGROUP SETID orders:stream workers $
# Ignorer tous les messages actuels

# 3. Sauter à un ID spécifique
XGROUP SETID orders:stream workers 1702301234567-0
# Utile pour recovery après crash
```

### XGROUP DESTROY : Supprimer un group

```bash
# Syntaxe
XGROUP DESTROY stream groupname

# Exemple
XGROUP DESTROY orders:stream workers
# Retourne : (integer) 1

# Attention : Supprime aussi tous les PEL (Pending Entry Lists)
```

### XGROUP DELCONSUMER : Supprimer un consumer

```bash
# Syntaxe
XGROUP DELCONSUMER stream groupname consumername

# Exemple
XGROUP DELCONSUMER orders:stream workers consumer-1
# Retourne : (integer) 3  # Nombre de pending messages

# Messages pending du consumer sont réattribués
```

### XREADGROUP : Lire avec un consumer group

```bash
# Syntaxe
XREADGROUP GROUP groupname consumername [COUNT count] [BLOCK milliseconds]
          [NOACK] STREAMS key [key ...] id [id ...]

# 1. Lire nouveaux messages
XREADGROUP GROUP workers consumer-1 STREAMS orders:stream >
# > = nouveaux messages non-livrés

# 2. Avec limite
XREADGROUP GROUP workers consumer-1 COUNT 10 STREAMS orders:stream >
# Max 10 messages

# 3. Mode blocking
XREADGROUP GROUP workers consumer-1 BLOCK 5000 STREAMS orders:stream >
# Attendre max 5 secondes

# 4. NOACK (pas d'ajout à PEL, pour messages non-critiques)
XREADGROUP GROUP workers consumer-1 NOACK STREAMS orders:stream >
# Messages non trackés, pas de retry possible

# 5. Lire messages pending de CE consumer
XREADGROUP GROUP workers consumer-1 STREAMS orders:stream 0
# 0 = messages pending de ce consumer uniquement

# 6. Lire depuis plusieurs streams
XREADGROUP GROUP workers consumer-1 STREAMS orders:stream events:stream > >
# Multi-stream processing

# Exemple de retour
1) 1) "orders:stream"
   2) 1) 1) "1702301234567-0"
         2) 1) "order_id"
            2) "123"
            3) "status"
            4) "pending"
```

### XACK : Acquitter des messages

```bash
# Syntaxe
XACK stream groupname id [id ...]

# 1. ACK un message
XACK orders:stream workers 1702301234567-0
# Retourne : (integer) 1

# 2. ACK plusieurs messages
XACK orders:stream workers 1702301234567-0 1702301234568-0 1702301234569-0
# Retourne : (integer) 3

# 3. ACK messages qui n'existent plus
XACK orders:stream workers 9999999999999-0
# Retourne : (integer) 0  # Message pas dans PEL

# Note : ACK retire le message de la PEL (Pending Entry List)
```

### XPENDING : Inspecter messages pending

```bash
# Syntaxe 1 : Vue d'ensemble
XPENDING stream groupname

# Exemple
XPENDING orders:stream workers
# Retourne :
# 1) (integer) 5                    # Nombre total pending
# 2) "1702301234567-0"              # Plus petit ID pending
# 3) "1702301234571-0"              # Plus grand ID pending
# 4) 1) 1) "consumer-1"             # Consumer avec pending
#       2) "3"                       # Nombre de pending
#    2) 1) "consumer-2"
#       2) "2"

# Syntaxe 2 : Détails des pending
XPENDING stream groupname [[IDLE min-idle-time] start end count [consumer]]

# 1. Les 10 premiers pending
XPENDING orders:stream workers - + 10

# 2. Pending d'un consumer spécifique
XPENDING orders:stream workers - + 10 consumer-1

# 3. Pending avec idle time > 5 minutes
XPENDING orders:stream workers IDLE 300000 - + 10
# 300000 ms = 5 minutes

# Exemple de retour détaillé
1) 1) "1702301234567-0"              # Message ID
   2) "consumer-1"                    # Consumer name
   3) (integer) 300456                # Idle time (ms)
   4) (integer) 3                     # Delivery count (tentatives)
```

### XCLAIM : Récupérer messages d'un autre consumer

```bash
# Syntaxe
XCLAIM stream groupname consumername min-idle-time id [id ...]
       [IDLE ms] [TIME unix-time-milliseconds] [RETRYCOUNT count] [FORCE] [JUSTID]

# 1. Claim messages avec idle > 5 min
XCLAIM orders:stream workers consumer-2 300000 1702301234567-0

# 2. Claim plusieurs messages
XCLAIM orders:stream workers consumer-2 300000 1702301234567-0 1702301234568-0

# 3. Avec JUSTID (retourner seulement IDs, plus performant)
XCLAIM orders:stream workers consumer-2 300000 1702301234567-0 JUSTID

# 4. Force claim (même si idle < min-idle-time)
XCLAIM orders:stream workers consumer-2 0 1702301234567-0 FORCE

# 5. Réinitialiser delivery count
XCLAIM orders:stream workers consumer-2 300000 1702301234567-0 RETRYCOUNT 0

# Exemple de retour
1) 1) "1702301234567-0"
   2) 1) "order_id"
      2) "123"
      3) "status"
      4) "pending"
```

### XAUTOCLAIM : Auto-claim (Redis 6.2+)

```bash
# Syntaxe
XAUTOCLAIM stream groupname consumername min-idle-time start [COUNT count] [JUSTID]

# 1. Auto-claim messages avec idle > 5 min
XAUTOCLAIM orders:stream workers consumer-2 300000 0-0

# 2. Avec limite
XAUTOCLAIM orders:stream workers consumer-2 300000 0-0 COUNT 10

# 3. Avec JUSTID (plus performant)
XAUTOCLAIM orders:stream workers consumer-2 300000 0-0 COUNT 10 JUSTID

# Exemple de retour
1) "0-0"                    # Next cursor (pour pagination)
2) 1) 1) "1702301234567-0" # Messages claimed
      2) [field value ...]
```

---

## Implémentation par langage

### Python avec redis-py

```python
import redis
import time
import json
import logging
from typing import Dict, List, Optional, Callable
from dataclasses import dataclass

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ============================================
# Consumer Group Manager
# ============================================
class ConsumerGroupManager:
    """Gestion des consumer groups"""

    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name

    def create_group(self, group_name: str, start_id: str = '0', mkstream: bool = True):
        """Créer un consumer group"""
        try:
            self.redis.xgroup_create(
                self.stream_name,
                group_name,
                id=start_id,
                mkstream=mkstream
            )
            logger.info(f"✓ Created group '{group_name}' on stream '{self.stream_name}'")
        except redis.exceptions.ResponseError as e:
            if 'BUSYGROUP' in str(e):
                logger.warning(f"Group '{group_name}' already exists")
            else:
                raise

    def destroy_group(self, group_name: str):
        """Supprimer un consumer group"""
        deleted = self.redis.xgroup_destroy(self.stream_name, group_name)
        if deleted:
            logger.info(f"✓ Destroyed group '{group_name}'")
        return deleted

    def set_id(self, group_name: str, message_id: str):
        """Changer la position du group"""
        self.redis.xgroup_setid(self.stream_name, group_name, id=message_id)
        logger.info(f"✓ Set group '{group_name}' ID to {message_id}")

    def delete_consumer(self, group_name: str, consumer_name: str):
        """Supprimer un consumer"""
        pending = self.redis.xgroup_delconsumer(
            self.stream_name,
            group_name,
            consumer_name
        )
        logger.info(f"✓ Deleted consumer '{consumer_name}' ({pending} pending messages)")
        return pending

    def get_groups_info(self):
        """Obtenir infos sur tous les groups"""
        try:
            return self.redis.xinfo_groups(self.stream_name)
        except redis.exceptions.ResponseError:
            return []

    def get_consumers_info(self, group_name: str):
        """Obtenir infos sur les consumers d'un group"""
        return self.redis.xinfo_consumers(self.stream_name, group_name)

# ============================================
# Consumer Worker
# ============================================
@dataclass
class Message:
    """Représentation d'un message"""
    id: str
    data: Dict
    stream: str
    group: str
    consumer: str

class StreamConsumer:
    """Consumer avec retry automatique et DLQ"""

    def __init__(
        self,
        stream_name: str,
        group_name: str,
        consumer_name: str,
        max_retries: int = 3,
        claim_min_idle: int = 300000  # 5 minutes
    ):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name
        self.consumer_name = consumer_name
        self.max_retries = max_retries
        self.claim_min_idle = claim_min_idle
        self.running = False

    def start(self, handler: Callable[[Message], bool]):
        """Démarrer le consumer"""
        self.running = True

        logger.info(f"Consumer '{self.consumer_name}' started")

        while self.running:
            try:
                # 1. Lire nouveaux messages
                self._process_new_messages(handler)

                # 2. Récupérer messages pending de ce consumer
                self._process_pending_messages(handler)

                # 3. Claim messages en timeout d'autres consumers
                self._claim_stuck_messages(handler)

            except KeyboardInterrupt:
                logger.info("Shutting down...")
                self.running = False
                break
            except Exception as e:
                logger.error(f"Error in consumer loop: {e}")
                time.sleep(1)

    def _process_new_messages(self, handler: Callable[[Message], bool]):
        """Traiter nouveaux messages"""
        try:
            # XREADGROUP avec > pour nouveaux messages
            response = self.redis.xreadgroup(
                self.group_name,
                self.consumer_name,
                {self.stream_name: '>'},
                count=10,
                block=1000  # Block 1 seconde
            )

            if response:
                for stream, messages in response:
                    for msg_id, data in messages:
                        message = Message(
                            id=msg_id,
                            data=data,
                            stream=stream,
                            group=self.group_name,
                            consumer=self.consumer_name
                        )

                        success = self._handle_message(message, handler)

                        if success:
                            self._ack_message(msg_id)
                        # Si échec, reste en pending pour retry

        except redis.exceptions.ResponseError as e:
            logger.error(f"Error reading new messages: {e}")

    def _process_pending_messages(self, handler: Callable[[Message], bool]):
        """Traiter messages pending de ce consumer"""
        try:
            # XREADGROUP avec 0 pour messages pending
            response = self.redis.xreadgroup(
                self.group_name,
                self.consumer_name,
                {self.stream_name: '0'},
                count=10
            )

            if response:
                for stream, messages in response:
                    for msg_id, data in messages:
                        # Vérifier nombre de tentatives
                        if self._should_retry(msg_id):
                            message = Message(
                                id=msg_id,
                                data=data,
                                stream=stream,
                                group=self.group_name,
                                consumer=self.consumer_name
                            )

                            success = self._handle_message(message, handler)

                            if success:
                                self._ack_message(msg_id)
                        else:
                            # Trop de tentatives → DLQ
                            self._move_to_dlq(msg_id, data)
                            self._ack_message(msg_id)

        except redis.exceptions.ResponseError as e:
            logger.error(f"Error processing pending: {e}")

    def _claim_stuck_messages(self, handler: Callable[[Message], bool]):
        """Récupérer messages coincés chez d'autres consumers"""
        try:
            # Utiliser XAUTOCLAIM si disponible (Redis 6.2+)
            try:
                result = self.redis.execute_command(
                    'XAUTOCLAIM',
                    self.stream_name,
                    self.group_name,
                    self.consumer_name,
                    str(self.claim_min_idle),
                    '0-0',
                    'COUNT', '10'
                )

                if result and len(result) > 1:
                    messages = result[1]
                    for msg in messages:
                        msg_id = msg[0]
                        data = dict(zip(msg[1][::2], msg[1][1::2]))

                        if self._should_retry(msg_id):
                            message = Message(
                                id=msg_id,
                                data=data,
                                stream=self.stream_name,
                                group=self.group_name,
                                consumer=self.consumer_name
                            )

                            success = self._handle_message(message, handler)

                            if success:
                                self._ack_message(msg_id)
                        else:
                            self._move_to_dlq(msg_id, data)
                            self._ack_message(msg_id)

            except redis.exceptions.ResponseError:
                # Fallback vers XPENDING + XCLAIM
                self._claim_with_xpending(handler)

        except Exception as e:
            logger.error(f"Error claiming stuck messages: {e}")

    def _claim_with_xpending(self, handler: Callable[[Message], bool]):
        """Claim avec XPENDING (fallback)"""
        pending = self.redis.xpending_range(
            self.stream_name,
            self.group_name,
            '-',
            '+',
            count=10,
            idle=self.claim_min_idle
        )

        for entry in pending:
            msg_id = entry['message_id']

            # Claim le message
            claimed = self.redis.xclaim(
                self.stream_name,
                self.group_name,
                self.consumer_name,
                self.claim_min_idle,
                [msg_id]
            )

            if claimed:
                msg_id, data = claimed[0]

                if self._should_retry(msg_id):
                    message = Message(
                        id=msg_id,
                        data=data,
                        stream=self.stream_name,
                        group=self.group_name,
                        consumer=self.consumer_name
                    )

                    success = self._handle_message(message, handler)

                    if success:
                        self._ack_message(msg_id)
                else:
                    self._move_to_dlq(msg_id, data)
                    self._ack_message(msg_id)

    def _handle_message(self, message: Message, handler: Callable[[Message], bool]) -> bool:
        """Traiter un message avec le handler"""
        try:
            logger.info(f"Processing message {message.id}")
            success = handler(message)

            if success:
                logger.info(f"✓ Successfully processed {message.id}")
            else:
                logger.warning(f"✗ Failed to process {message.id}")

            return success

        except Exception as e:
            logger.error(f"✗ Error processing {message.id}: {e}")
            return False

    def _ack_message(self, msg_id: str):
        """Acquitter un message"""
        self.redis.xack(self.stream_name, self.group_name, msg_id)
        logger.debug(f"ACKed message {msg_id}")

    def _should_retry(self, msg_id: str) -> bool:
        """Vérifier si on doit retry"""
        pending = self.redis.xpending_range(
            self.stream_name,
            self.group_name,
            msg_id,
            msg_id,
            count=1
        )

        if pending:
            delivery_count = pending[0]['times_delivered']
            return delivery_count < self.max_retries

        return True

    def _move_to_dlq(self, msg_id: str, data: Dict):
        """Déplacer vers Dead Letter Queue"""
        dlq_stream = f"{self.stream_name}:dlq"

        dlq_data = {
            'original_id': msg_id,
            'original_data': json.dumps(data),
            'failed_at': str(time.time()),
            'consumer': self.consumer_name
        }

        self.redis.xadd(dlq_stream, dlq_data)
        logger.warning(f"Moved message {msg_id} to DLQ")

    def stop(self):
        """Arrêter le consumer"""
        self.running = False

# ============================================
# Monitoring
# ============================================
class ConsumerGroupMonitor:
    """Monitoring des consumer groups"""

    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name

    def get_group_stats(self, group_name: str) -> Dict:
        """Statistiques d'un group"""
        # Infos générales du group
        groups = self.redis.xinfo_groups(self.stream_name)
        group_info = next((g for g in groups if g['name'] == group_name), None)

        if not group_info:
            return {}

        # Infos consumers
        consumers = self.redis.xinfo_consumers(self.stream_name, group_name)

        # Messages pending
        pending_summary = self.redis.xpending(self.stream_name, group_name)

        return {
            'group_name': group_name,
            'pending_messages': group_info['pending'],
            'last_delivered_id': group_info['last-delivered-id'],
            'consumers': len(consumers),
            'consumer_details': [
                {
                    'name': c['name'],
                    'pending': c['pending'],
                    'idle_ms': c['idle']
                }
                for c in consumers
            ],
            'pending_summary': {
                'total': pending_summary[0] if pending_summary else 0,
                'min_id': pending_summary[1] if pending_summary else None,
                'max_id': pending_summary[2] if pending_summary else None
            }
        }

    def get_stuck_messages(self, group_name: str, min_idle_ms: int = 300000) -> List:
        """Trouver messages coincés"""
        pending = self.redis.xpending_range(
            self.stream_name,
            group_name,
            '-',
            '+',
            count=100
        )

        stuck = []
        for entry in pending:
            if entry['time_since_delivered'] >= min_idle_ms:
                stuck.append({
                    'message_id': entry['message_id'],
                    'consumer': entry['consumer'],
                    'idle_ms': entry['time_since_delivered'],
                    'delivery_count': entry['times_delivered']
                })

        return stuck

    def print_stats(self, group_name: str):
        """Afficher statistiques"""
        stats = self.get_group_stats(group_name)

        if not stats:
            print(f"Group '{group_name}' not found")
            return

        print(f"\n{'=' * 60}")
        print(f"Consumer Group: {stats['group_name']}")
        print(f"{'=' * 60}")
        print(f"Pending messages: {stats['pending_messages']}")
        print(f"Active consumers: {stats['consumers']}")
        print(f"Last delivered ID: {stats['last_delivered_id']}")

        print(f"\nConsumers:")
        for consumer in stats['consumer_details']:
            idle_sec = consumer['idle_ms'] / 1000
            print(f"  {consumer['name']}:")
            print(f"    Pending: {consumer['pending']}")
            print(f"    Idle: {idle_sec:.1f}s")

        stuck = self.get_stuck_messages(group_name)
        if stuck:
            print(f"\n⚠️  Stuck messages: {len(stuck)}")
            for msg in stuck[:5]:  # Top 5
                print(f"  {msg['message_id']}:")
                print(f"    Consumer: {msg['consumer']}")
                print(f"    Idle: {msg['idle_ms']/1000:.1f}s")
                print(f"    Attempts: {msg['delivery_count']}")

# ============================================
# Exemple d'utilisation
# ============================================
if __name__ == "__main__":
    stream_name = 'orders:stream'
    group_name = 'order-processors'

    # Setup
    manager = ConsumerGroupManager(stream_name)
    manager.create_group(group_name, start_id='$', mkstream=True)

    # Handler pour traiter les messages
    def process_order(message: Message) -> bool:
        """Traiter une commande"""
        try:
            order_id = message.data.get('order_id')
            amount = message.data.get('amount')

            print(f"Processing order {order_id} for ${amount}")

            # Simuler traitement
            time.sleep(0.5)

            # Simuler échec aléatoire (20% de chance)
            import random
            if random.random() < 0.2:
                raise Exception("Random processing error")

            return True

        except Exception as e:
            logger.error(f"Error: {e}")
            return False

    # Démarrer consumer
    consumer = StreamConsumer(
        stream_name,
        group_name,
        'consumer-1',
        max_retries=3,
        claim_min_idle=10000  # 10 secondes pour demo
    )

    # Démarrer dans un thread
    import threading

    thread = threading.Thread(target=consumer.start, args=(process_order,))
    thread.daemon = True
    thread.start()

    # Produire quelques messages
    time.sleep(1)

    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    for i in range(10):
        redis_client.xadd(stream_name, {
            'order_id': str(1000 + i),
            'amount': str(99.99 * (i + 1)),
            'status': 'pending'
        })
        time.sleep(0.2)

    # Monitoring
    time.sleep(5)

    monitor = ConsumerGroupMonitor(stream_name)
    monitor.print_stats(group_name)

    # Attendre
    try:
        thread.join(timeout=60)
    except KeyboardInterrupt:
        consumer.stop()
```

### Node.js avec ioredis

```javascript
const Redis = require('ioredis');
const { EventEmitter } = require('events');

// ============================================
// Consumer Group Manager
// ============================================
class ConsumerGroupManager {
    constructor(streamName) {
        this.redis = new Redis();
        this.streamName = streamName;
    }

    async createGroup(groupName, startId = '0', mkstream = true) {
        try {
            const args = [this.streamName, groupName, startId];
            if (mkstream) {
                args.push('MKSTREAM');
            }

            await this.redis.xgroup('CREATE', ...args);
            console.log(`✓ Created group '${groupName}' on stream '${this.streamName}'`);
        } catch (err) {
            if (err.message.includes('BUSYGROUP')) {
                console.log(`Group '${groupName}' already exists`);
            } else {
                throw err;
            }
        }
    }

    async destroyGroup(groupName) {
        const deleted = await this.redis.xgroup('DESTROY', this.streamName, groupName);
        if (deleted) {
            console.log(`✓ Destroyed group '${groupName}'`);
        }
        return deleted;
    }

    async setId(groupName, messageId) {
        await this.redis.xgroup('SETID', this.streamName, groupName, messageId);
        console.log(`✓ Set group '${groupName}' ID to ${messageId}`);
    }

    async deleteConsumer(groupName, consumerName) {
        const pending = await this.redis.xgroup(
            'DELCONSUMER',
            this.streamName,
            groupName,
            consumerName
        );
        console.log(`✓ Deleted consumer '${consumerName}' (${pending} pending messages)`);
        return pending;
    }

    async getGroupsInfo() {
        try {
            return await this.redis.xinfo('GROUPS', this.streamName);
        } catch (err) {
            return [];
        }
    }

    async getConsumersInfo(groupName) {
        return await this.redis.xinfo('CONSUMERS', this.streamName, groupName);
    }

    async close() {
        await this.redis.quit();
    }
}

// ============================================
// Stream Consumer
// ============================================
class StreamConsumer extends EventEmitter {
    constructor(streamName, groupName, consumerName, options = {}) {
        super();

        this.redis = new Redis();
        this.streamName = streamName;
        this.groupName = groupName;
        this.consumerName = consumerName;

        this.maxRetries = options.maxRetries || 3;
        this.claimMinIdle = options.claimMinIdle || 300000; // 5 minutes
        this.blockTime = options.blockTime || 1000;
        this.running = false;
    }

    async start(handler) {
        this.running = true;
        this.handler = handler;

        console.log(`Consumer '${this.consumerName}' started`);

        while (this.running) {
            try {
                // 1. Process new messages
                await this.processNewMessages();

                // 2. Process pending messages
                await this.processPendingMessages();

                // 3. Claim stuck messages
                await this.claimStuckMessages();

            } catch (err) {
                console.error('Error in consumer loop:', err);
                await this.sleep(1000);
            }
        }

        console.log('Consumer stopped');
    }

    async processNewMessages() {
        try {
            const response = await this.redis.xreadgroup(
                'GROUP',
                this.groupName,
                this.consumerName,
                'COUNT',
                10,
                'BLOCK',
                this.blockTime,
                'STREAMS',
                this.streamName,
                '>'
            );

            if (response && response.length > 0) {
                const [streamName, messages] = response[0];

                for (const [messageId, fields] of messages) {
                    const data = this.parseFields(fields);

                    const message = {
                        id: messageId,
                        data: data,
                        stream: streamName,
                        group: this.groupName,
                        consumer: this.consumerName
                    };

                    const success = await this.handleMessage(message);

                    if (success) {
                        await this.ackMessage(messageId);
                    }
                    // Si échec, reste en pending
                }
            }
        } catch (err) {
            console.error('Error reading new messages:', err);
        }
    }

    async processPendingMessages() {
        try {
            const response = await this.redis.xreadgroup(
                'GROUP',
                this.groupName,
                this.consumerName,
                'COUNT',
                10,
                'STREAMS',
                this.streamName,
                '0'
            );

            if (response && response.length > 0) {
                const [streamName, messages] = response[0];

                for (const [messageId, fields] of messages) {
                    const shouldRetry = await this.shouldRetry(messageId);

                    if (shouldRetry) {
                        const data = this.parseFields(fields);

                        const message = {
                            id: messageId,
                            data: data,
                            stream: streamName,
                            group: this.groupName,
                            consumer: this.consumerName
                        };

                        const success = await this.handleMessage(message);

                        if (success) {
                            await this.ackMessage(messageId);
                        }
                    } else {
                        // Trop de tentatives → DLQ
                        await this.moveToDLQ(messageId, this.parseFields(fields));
                        await this.ackMessage(messageId);
                    }
                }
            }
        } catch (err) {
            console.error('Error processing pending:', err);
        }
    }

    async claimStuckMessages() {
        try {
            // Utiliser XAUTOCLAIM si disponible
            const result = await this.redis.call(
                'XAUTOCLAIM',
                this.streamName,
                this.groupName,
                this.consumerName,
                this.claimMinIdle.toString(),
                '0-0',
                'COUNT',
                '10'
            );

            if (result && result.length > 1) {
                const messages = result[1];

                for (const msg of messages) {
                    const messageId = msg[0];
                    const fields = msg[1];
                    const data = this.parseFields(fields);

                    const shouldRetry = await this.shouldRetry(messageId);

                    if (shouldRetry) {
                        const message = {
                            id: messageId,
                            data: data,
                            stream: this.streamName,
                            group: this.groupName,
                            consumer: this.consumerName
                        };

                        const success = await this.handleMessage(message);

                        if (success) {
                            await this.ackMessage(messageId);
                        }
                    } else {
                        await this.moveToDLQ(messageId, data);
                        await this.ackMessage(messageId);
                    }
                }
            }
        } catch (err) {
            // Fallback vers XPENDING + XCLAIM
            await this.claimWithXPending();
        }
    }

    async claimWithXPending() {
        const pending = await this.redis.xpending(
            this.streamName,
            this.groupName,
            '-',
            '+',
            10
        );

        for (const entry of pending) {
            const [messageId, consumer, idleTime, deliveryCount] = entry;

            if (idleTime >= this.claimMinIdle) {
                const claimed = await this.redis.xclaim(
                    this.streamName,
                    this.groupName,
                    this.consumerName,
                    this.claimMinIdle,
                    messageId
                );

                if (claimed && claimed.length > 0) {
                    const [msgId, fields] = claimed[0];
                    const data = this.parseFields(fields);

                    if (deliveryCount < this.maxRetries) {
                        const message = {
                            id: msgId,
                            data: data,
                            stream: this.streamName,
                            group: this.groupName,
                            consumer: this.consumerName
                        };

                        const success = await this.handleMessage(message);

                        if (success) {
                            await this.ackMessage(msgId);
                        }
                    } else {
                        await this.moveToDLQ(msgId, data);
                        await this.ackMessage(msgId);
                    }
                }
            }
        }
    }

    async handleMessage(message) {
        try {
            console.log(`Processing message ${message.id}`);
            const success = await this.handler(message);

            if (success) {
                console.log(`✓ Successfully processed ${message.id}`);
            } else {
                console.log(`✗ Failed to process ${message.id}`);
            }

            return success;
        } catch (err) {
            console.error(`✗ Error processing ${message.id}:`, err);
            return false;
        }
    }

    async ackMessage(messageId) {
        await this.redis.xack(this.streamName, this.groupName, messageId);
        console.log(`ACKed message ${messageId}`);
    }

    async shouldRetry(messageId) {
        const pending = await this.redis.xpending(
            this.streamName,
            this.groupName,
            messageId,
            messageId,
            1
        );

        if (pending && pending.length > 0) {
            const [, , , deliveryCount] = pending[0];
            return deliveryCount < this.maxRetries;
        }

        return true;
    }

    async moveToDLQ(messageId, data) {
        const dlqStream = `${this.streamName}:dlq`;

        const dlqData = {
            original_id: messageId,
            original_data: JSON.stringify(data),
            failed_at: Date.now().toString(),
            consumer: this.consumerName
        };

        await this.redis.xadd(dlqStream, '*', ...this.flattenObject(dlqData));
        console.log(`⚠️  Moved message ${messageId} to DLQ`);
    }

    parseFields(fields) {
        const data = {};
        for (let i = 0; i < fields.length; i += 2) {
            data[fields[i]] = fields[i + 1];
        }
        return data;
    }

    flattenObject(obj) {
        const result = [];
        for (const [key, value] of Object.entries(obj)) {
            result.push(key, String(value));
        }
        return result;
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    stop() {
        this.running = false;
    }

    async close() {
        this.stop();
        await this.redis.quit();
    }
}

// ============================================
// Consumer Group Monitor
// ============================================
class ConsumerGroupMonitor {
    constructor(streamName) {
        this.redis = new Redis();
        this.streamName = streamName;
    }

    async getGroupStats(groupName) {
        // Infos générales
        const groups = await this.redis.xinfo('GROUPS', this.streamName);
        const groupInfo = groups.find(g => g[1] === groupName);

        if (!groupInfo) {
            return null;
        }

        // Infos consumers
        const consumers = await this.redis.xinfo('CONSUMERS', this.streamName, groupName);

        // Pending
        const pendingSummary = await this.redis.xpending(this.streamName, groupName);

        return {
            groupName: groupName,
            pendingMessages: groupInfo[3],
            lastDeliveredId: groupInfo[5],
            consumers: consumers.length,
            consumerDetails: consumers.map(c => ({
                name: c[1],
                pending: c[3],
                idleMs: c[5]
            })),
            pendingSummary: {
                total: pendingSummary[0] || 0,
                minId: pendingSummary[1] || null,
                maxId: pendingSummary[2] || null
            }
        };
    }

    async getStuckMessages(groupName, minIdleMs = 300000) {
        const pending = await this.redis.xpending(
            this.streamName,
            groupName,
            '-',
            '+',
            100
        );

        const stuck = [];
        for (const entry of pending) {
            const [messageId, consumer, idleTime, deliveryCount] = entry;

            if (idleTime >= minIdleMs) {
                stuck.push({
                    messageId,
                    consumer,
                    idleMs: idleTime,
                    deliveryCount
                });
            }
        }

        return stuck;
    }

    async printStats(groupName) {
        const stats = await this.getGroupStats(groupName);

        if (!stats) {
            console.log(`Group '${groupName}' not found`);
            return;
        }

        console.log('\n' + '='.repeat(60));
        console.log(`Consumer Group: ${stats.groupName}`);
        console.log('='.repeat(60));
        console.log(`Pending messages: ${stats.pendingMessages}`);
        console.log(`Active consumers: ${stats.consumers}`);
        console.log(`Last delivered ID: ${stats.lastDeliveredId}`);

        console.log('\nConsumers:');
        for (const consumer of stats.consumerDetails) {
            const idleSec = consumer.idleMs / 1000;
            console.log(`  ${consumer.name}:`);
            console.log(`    Pending: ${consumer.pending}`);
            console.log(`    Idle: ${idleSec.toFixed(1)}s`);
        }

        const stuck = await this.getStuckMessages(groupName);
        if (stuck.length > 0) {
            console.log(`\n⚠️  Stuck messages: ${stuck.length}`);
            for (const msg of stuck.slice(0, 5)) {
                console.log(`  ${msg.messageId}:`);
                console.log(`    Consumer: ${msg.consumer}`);
                console.log(`    Idle: ${(msg.idleMs / 1000).toFixed(1)}s`);
                console.log(`    Attempts: ${msg.deliveryCount}`);
            }
        }
    }

    async close() {
        await this.redis.quit();
    }
}

// ============================================
// Exemple d'utilisation
// ============================================
async function main() {
    const streamName = 'orders:stream';
    const groupName = 'order-processors';

    // Setup
    const manager = new ConsumerGroupManager(streamName);
    await manager.createGroup(groupName, '$', true);

    // Handler
    async function processOrder(message) {
        try {
            const orderId = message.data.order_id;
            const amount = message.data.amount;

            console.log(`Processing order ${orderId} for $${amount}`);

            // Simuler traitement
            await new Promise(resolve => setTimeout(resolve, 500));

            // Simuler échec aléatoire (20%)
            if (Math.random() < 0.2) {
                throw new Error('Random processing error');
            }

            return true;
        } catch (err) {
            console.error('Error:', err.message);
            return false;
        }
    }

    // Consumer
    const consumer = new StreamConsumer(
        streamName,
        groupName,
        'consumer-1',
        {
            maxRetries: 3,
            claimMinIdle: 10000, // 10 secondes pour demo
            blockTime: 1000
        }
    );

    // Démarrer consumer
    consumer.start(processOrder);

    // Produire messages
    await new Promise(resolve => setTimeout(resolve, 1000));

    const redis = new Redis();
    for (let i = 0; i < 10; i++) {
        await redis.xadd(
            streamName,
            '*',
            'order_id', (1000 + i).toString(),
            'amount', (99.99 * (i + 1)).toFixed(2),
            'status', 'pending'
        );
        await new Promise(resolve => setTimeout(resolve, 200));
    }

    // Monitoring
    await new Promise(resolve => setTimeout(resolve, 5000));

    const monitor = new ConsumerGroupMonitor(streamName);
    await monitor.printStats(groupName);

    // Cleanup
    await new Promise(resolve => setTimeout(resolve, 5000));
    consumer.stop();
    await consumer.close();
    await manager.close();
    await monitor.close();
    await redis.quit();
}

main().catch(console.error);
```

---

## Patterns avancés

### Pattern 1 : Multiple consumers pour scalabilité

```python
import redis
import multiprocessing
import time

def worker_process(worker_id: int, stream_name: str, group_name: str):
    """Process worker"""
    consumer = StreamConsumer(
        stream_name,
        group_name,
        f'consumer-{worker_id}',
        max_retries=3
    )

    def handler(message: Message) -> bool:
        print(f"[Worker {worker_id}] Processing {message.id}")
        time.sleep(0.5)
        return True

    consumer.start(handler)

# Démarrer 5 workers
if __name__ == '__main__':
    stream_name = 'orders:stream'
    group_name = 'workers'
    num_workers = 5

    # Setup group
    manager = ConsumerGroupManager(stream_name)
    manager.create_group(group_name, '$', mkstream=True)

    # Démarrer workers
    processes = []
    for i in range(num_workers):
        p = multiprocessing.Process(
            target=worker_process,
            args=(i, stream_name, group_name)
        )
        p.start()
        processes.append(p)

    # Attendre
    for p in processes:
        p.join()
```

### Pattern 2 : Priority queues

```python
class PriorityStreamProcessor:
    """Traiter messages par priorité"""

    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.streams = {
            'high': 'tasks:high',
            'medium': 'tasks:medium',
            'low': 'tasks:low'
        }
        self.group_name = 'task-processors'

    def setup(self):
        """Setup consumer groups"""
        for stream in self.streams.values():
            try:
                self.redis.xgroup_create(stream, self.group_name, '$', mkstream=True)
            except redis.exceptions.ResponseError:
                pass

    def consume(self, consumer_name: str):
        """Consumer avec priorités"""
        while True:
            # 1. Essayer haute priorité
            messages = self._read_from_stream('high', consumer_name, count=10)
            if messages:
                self._process_messages(messages, consumer_name)
                continue

            # 2. Essayer moyenne priorité
            messages = self._read_from_stream('medium', consumer_name, count=10)
            if messages:
                self._process_messages(messages, consumer_name)
                continue

            # 3. Essayer basse priorité
            messages = self._read_from_stream('low', consumer_name, count=5)
            if messages:
                self._process_messages(messages, consumer_name)
                continue

            # 4. Rien à faire, attendre
            time.sleep(1)

    def _read_from_stream(self, priority: str, consumer_name: str, count: int):
        """Lire d'un stream de priorité"""
        stream = self.streams[priority]

        response = self.redis.xreadgroup(
            self.group_name,
            consumer_name,
            {stream: '>'},
            count=count,
            block=100
        )

        if response:
            return response[0][1]

        return None

    def _process_messages(self, messages, consumer_name: str):
        """Traiter messages"""
        for msg_id, data in messages:
            print(f"[{consumer_name}] Processing {msg_id}: {data}")
            time.sleep(0.1)

            # ACK
            for stream in self.streams.values():
                try:
                    self.redis.xack(stream, self.group_name, msg_id)
                except:
                    pass
```

### Pattern 3 : Fan-out avec multiple groups

```python
class FanOutProcessor:
    """Un stream, plusieurs groups pour traitement différent"""

    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.groups = {
            'email-sender': self.send_email,
            'sms-sender': self.send_sms,
            'analytics': self.track_analytics,
            'audit': self.log_audit
        }

    def setup(self):
        """Setup tous les groups"""
        for group_name in self.groups.keys():
            try:
                self.redis.xgroup_create(
                    self.stream_name,
                    group_name,
                    '0',  # Traiter depuis le début
                    mkstream=True
                )
            except redis.exceptions.ResponseError:
                pass

    def start_consumer(self, group_name: str, consumer_name: str):
        """Démarrer un consumer pour un group"""
        handler = self.groups[group_name]

        while True:
            response = self.redis.xreadgroup(
                group_name,
                consumer_name,
                {self.stream_name: '>'},
                count=10,
                block=1000
            )

            if response:
                for stream, messages in response:
                    for msg_id, data in messages:
                        try:
                            handler(data)
                            self.redis.xack(self.stream_name, group_name, msg_id)
                        except Exception as e:
                            print(f"Error in {group_name}: {e}")

    def send_email(self, data):
        print(f"[EMAIL] Sending to {data.get('email')}")

    def send_sms(self, data):
        print(f"[SMS] Sending to {data.get('phone')}")

    def track_analytics(self, data):
        print(f"[ANALYTICS] Event: {data.get('event_type')}")

    def log_audit(self, data):
        print(f"[AUDIT] Action: {data.get('action')}")

# Utilisation
processor = FanOutProcessor('user:events')
processor.setup()

# Démarrer un consumer par group
import threading

for group_name in processor.groups.keys():
    thread = threading.Thread(
        target=processor.start_consumer,
        args=(group_name, f'{group_name}-consumer-1')
    )
    thread.daemon = True
    thread.start()
```

---

## Conclusion

Les **Consumer Groups** transforment Redis Streams en un système de messaging robuste et scalable :

**Avantages** :
- ✅ Distribution automatique de charge
- ✅ Traitement parallèle natif
- ✅ Retry automatique avec XPENDING
- ✅ ACK explicites pour fiabilité
- ✅ Monitoring complet

**Cas d'usage idéaux** :
- Job queues avec workers multiples
- Event processing pipelines
- Order processing systems
- Data ingestion avec ETL

**Règle d'or** : Si vous avez besoin de > 1 consumer avec garanties de livraison, utilisez Consumer Groups.

---


⏭️ [Redis Streams : XREAD vs XREADGROUP, gestion des ACK](/08-communication-flux-donnees/05-redis-streams-xread-xreadgroup-ack.md)
