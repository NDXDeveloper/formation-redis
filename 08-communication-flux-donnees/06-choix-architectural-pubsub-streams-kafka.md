🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.6 Choix Architectural : Pub/Sub vs Streams vs Lists vs Kafka

## Introduction

Choisir le bon mécanisme de messaging est une décision architecturale critique qui impacte la performance, la fiabilité, la scalabilité et la complexité opérationnelle de votre système. Cette section vous guide dans ce choix en comparant les solutions Redis (Pub/Sub, Streams, Lists) et les systèmes externes (Kafka, RabbitMQ, NATS).

### Le paysage du messaging

```
┌───────────────────────────────────────────────────────────┐
│                    REDIS (In-Memory)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ Pub/Sub  │  │ Streams  │  │  Lists   │                 │
│  │ < 1ms    │  │ 1-5ms    │  │ 1-3ms    │                 │
│  │ No Pers. │  │ Persist  │  │ Persist  │                 │
│  └──────────┘  └──────────┘  └──────────┘                 │
└───────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────┐
│              EXTERNAL SYSTEMS (Disk-based)                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │  Kafka   │  │ RabbitMQ │  │   NATS   │                 │
│  │ 5-50ms   │  │ 10-100ms │  │ 1-10ms   │                 │
│  │ Persist  │  │ Persist  │  │ Optional │                 │
│  └──────────┘  └──────────┘  └──────────┘                 │
└───────────────────────────────────────────────────────────┘
```

---

## Comparaison Redis : Pub/Sub vs Streams vs Lists

### Tableau Synthétique

| Critère | **Pub/Sub** | **Streams** | **Lists** |
|---------|-------------|-------------|-----------|
| **Latence** | ✅✅ < 1ms | ✅ 1-5ms | ✅ 1-3ms |
| **Throughput** | ✅✅ 100K/s | ✅ 50-80K/s | ✅ 80K/s |
| **Persistance** | ❌ Non | ✅ Oui (RDB/AOF) | ✅ Oui |
| **Garantie livraison** | ❌ At-most-once | ✅ At-least-once | ⚠️ Exactly-once* |
| **Rejouabilité** | ❌ Non | ✅ Illimitée | ❌ Non |
| **Consumer groups** | ❌ Non | ✅ Natif | ❌ Non |
| **Load balancing** | ❌ Manuel | ✅ Automatique | ❌ Manuel |
| **Retry automatique** | ❌ Non | ✅ Oui (PEL) | ❌ Non |
| **Pattern matching** | ✅ PSUBSCRIBE | ❌ Non | ❌ Non |
| **Ordre strict** | ✅ Oui | ✅ Oui | ✅ FIFO |
| **Range queries** | ❌ Non | ✅ Oui | ⚠️ Limité |
| **ACK explicites** | ❌ Non | ✅ Oui | ⚠️ Implicite |
| **Memory overhead** | ✅✅ Minimal | ⚠️ Modéré (PEL) | ✅ Faible |
| **Complexité** | ✅✅ Très simple | ⚠️ Moyenne | ✅ Simple |
| **Scalabilité** | ⚠️ Limitée | ⚠️ Moyenne | ⚠️ Limitée |

*Lists : Exactly-once possible avec BRPOPLPUSH + transactions

### Matrice de Décision Redis

```python
def choose_redis_mechanism(requirements: dict) -> str:
    """
    Aide au choix entre Pub/Sub, Streams et Lists
    """

    # Élimination rapide
    if not requirements.get('persistence_required', False):
        if requirements.get('ultra_low_latency', False):
            return "Pub/Sub"

    # Streams obligatoire si
    if requirements.get('consumer_groups', False):
        return "Streams"

    if requirements.get('rejouabilite', False):
        return "Streams"

    if requirements.get('retry_automatique', False):
        return "Streams"

    if requirements.get('range_queries', False):
        return "Streams"

    # Lists suffisent si
    if (requirements.get('simple_queue', False) and
        requirements.get('single_consumer', False)):
        return "Lists"

    # Pub/Sub si
    if (requirements.get('broadcast', False) and
        not requirements.get('persistence_required', False)):
        return "Pub/Sub"

    if requirements.get('pattern_matching', False):
        return "Pub/Sub"

    # Par défaut : Streams (meilleur compromis)
    return "Streams"

# Tests
scenarios = {
    "Real-time notifications": {
        'persistence_required': False,
        'ultra_low_latency': True,
        'broadcast': True
    },
    "Job queue": {
        'persistence_required': True,
        'consumer_groups': True,
        'retry_automatique': True
    },
    "Simple task queue": {
        'persistence_required': True,
        'simple_queue': True,
        'single_consumer': True
    },
    "Event sourcing": {
        'persistence_required': True,
        'rejouabilite': True,
        'range_queries': True
    }
}

for scenario, reqs in scenarios.items():
    choice = choose_redis_mechanism(reqs)
    print(f"{scenario}: {choice}")

# Output:
# Real-time notifications: Pub/Sub
# Job queue: Streams
# Simple task queue: Lists
# Event sourcing: Streams
```

### Exemples de Code Comparatifs

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# ============================================
# 1. PUB/SUB : Notification temps réel
# ============================================
def pubsub_example():
    """
    Use case : Live dashboard updates
    Caractéristiques : < 1ms, pas de persistance
    """

    # Publisher
    def publish_metric(metric_name: str, value: float):
        r.publish(f'metrics:{metric_name}', str(value))

    # Subscriber
    pubsub = r.pubsub()
    pubsub.subscribe('metrics:cpu', 'metrics:memory')

    for message in pubsub.listen():
        if message['type'] == 'message':
            print(f"Live update: {message['channel']} = {message['data']}")

# ============================================
# 2. STREAMS : Job processing
# ============================================
def streams_example():
    """
    Use case : Order processing avec workers
    Caractéristiques : Persisté, retry, consumer groups
    """

    # Producer
    def submit_order(order_data: dict):
        return r.xadd('orders', order_data)

    # Consumer avec groupe
    r.xgroup_create('orders', 'processors', '$', mkstream=True)

    def process_orders():
        while True:
            messages = r.xreadgroup(
                'processors',
                'worker-1',
                {'orders': '>'},
                count=10,
                block=1000
            )

            if messages:
                for stream, entries in messages:
                    for msg_id, data in entries:
                        try:
                            # Process
                            process_order(data)

                            # ACK
                            r.xack('orders', 'processors', msg_id)
                        except Exception as e:
                            # Pas d'ACK → retry automatique
                            print(f"Error: {e}")

# ============================================
# 3. LISTS : Simple task queue
# ============================================
def lists_example():
    """
    Use case : Background email sending
    Caractéristiques : Simple, FIFO, single consumer
    """

    # Producer
    def queue_email(email_data: dict):
        import json
        r.lpush('emails:queue', json.dumps(email_data))

    # Consumer
    def process_emails():
        while True:
            # BRPOP : blocking pop
            result = r.brpop('emails:queue', timeout=5)

            if result:
                queue_name, email_json = result
                email_data = json.loads(email_json)

                send_email(email_data)

def process_order(data):
    print(f"Processing order: {data}")

def send_email(data):
    print(f"Sending email: {data}")
```

---

## Redis vs Systèmes Externes

### Quand Sortir de Redis ?

```
Redis est excellent MAIS limité par :
┌───────────────────────────────────────────────┐
│ 1. Mémoire                                    │
│    • Tout en RAM                              │
│    • Coût élevé pour gros volumes             │
│    • Limite : ~100GB typiquement              │
├───────────────────────────────────────────────┤
│ 2. Scalabilité                                │
│    • Vertical scaling principalement          │
│    • Partitioning manuel complexe             │
│    • Limite : ~100K msg/s par instance        │
├───────────────────────────────────────────────┤
│ 3. Durabilité                                 │
│    • RDB/AOF = compromis                      │
│    • Pas de réplication synchrone             │
│    • Window de perte possible                 │
└───────────────────────────────────────────────┘

→ Si besoin > ces limites → Kafka, RabbitMQ, etc.
```

### Comparaison Complète

| Critère | Redis Streams | **Kafka** | **RabbitMQ** | **NATS** |
|---------|---------------|-----------|--------------|----------|
| **Latence P99** | 1-5ms | 5-50ms | 10-100ms | 1-10ms |
| **Throughput** | 50K msg/s | **1M+ msg/s** | 10-50K msg/s | 100K+ msg/s |
| **Persistance** | RAM+Disk | **Disk only** | Disk | Optional |
| **Rétention** | Illimitée* | **Jours/Semaines** | Jusqu'à ACK | Aucune† |
| **Rejouabilité** | ✅ Oui | ✅ Oui | ⚠️ Limitée | ⚠️ JetStream |
| **Scalabilité** | Vertical | **Horizontal** | Vertical | Horizontal |
| **Partitioning** | Manuel | **Automatique** | Queues | Subjects |
| **Consumer groups** | ✅ Oui | ✅ Oui | ✅ Oui | ✅ Oui |
| **Order garantie** | ✅ Global | ⚠️ Par partition | ⚠️ Par queue | ❌ Non |
| **Complexité ops** | ✅ Simple | ❌ Élevée | ⚠️ Moyenne | ✅ Simple |
| **Coût RAM** | Élevé | **Faible** | Moyen | Faible |
| **Coût disk** | Faible | **Moyen** | Moyen | Faible |
| **Maturity** | Moyenne | **Très élevée** | Très élevée | Élevée |
| **Ecosystem** | Redis | **Très riche** | Riche | Croissant |

*Redis : Limité par RAM disponible
†NATS : Sauf avec JetStream (persistance optionnelle)

### Matrice de Seuils

```python
def choose_messaging_system(metrics: dict) -> str:
    """
    Choisir le système selon les métriques
    """

    throughput = metrics.get('msg_per_second', 0)
    data_size_gb = metrics.get('data_size_gb', 0)
    retention_days = metrics.get('retention_days', 7)
    latency_required_ms = metrics.get('latency_ms', 100)

    # Kafka si
    if throughput > 100000:
        return "Kafka (high throughput)"

    if data_size_gb > 100:
        return "Kafka (large dataset)"

    if retention_days > 30:
        return "Kafka (long retention)"

    # RabbitMQ si
    if metrics.get('complex_routing', False):
        return "RabbitMQ (complex routing)"

    if metrics.get('priority_queues', False):
        return "RabbitMQ (priority support)"

    # NATS si
    if (latency_required_ms < 5 and
        not metrics.get('persistence_critical', False)):
        return "NATS (ultra-low latency)"

    # Redis Streams si
    if (throughput < 50000 and
        data_size_gb < 50 and
        latency_required_ms < 10):
        return "Redis Streams"

    # Redis Pub/Sub si
    if (not metrics.get('persistence_required', False) and
        latency_required_ms < 2):
        return "Redis Pub/Sub"

    # Par défaut
    return "Kafka (safe default for scale)"

# Tests
scenarios = {
    "IoT data ingestion": {
        'msg_per_second': 500000,
        'data_size_gb': 500,
        'retention_days': 90,
        'latency_ms': 50
    },
    "Microservices events": {
        'msg_per_second': 10000,
        'data_size_gb': 10,
        'retention_days': 7,
        'latency_ms': 5
    },
    "Real-time gaming": {
        'msg_per_second': 50000,
        'data_size_gb': 5,
        'retention_days': 1,
        'latency_ms': 1,
        'persistence_required': False
    }
}

for scenario, metrics in scenarios.items():
    choice = choose_messaging_system(metrics)
    print(f"{scenario}: {choice}")

# Output:
# IoT data ingestion: Kafka (high throughput)
# Microservices events: Redis Streams
# Real-time gaming: Redis Pub/Sub
```

---

## Cas d'Usage Détaillés

### 1. E-Commerce Order Processing

```python
"""
Système de traitement de commandes e-commerce

Requirements:
- 1000-5000 orders/minute (peak)
- Persistence requise
- Retry automatique
- Multiple processing stages
- Audit trail complet
"""

# ✅ SOLUTION: Redis Streams
# Pourquoi :
# - Throughput suffisant (< 100/sec)
# - Persistence + retry natif
# - Consumer groups pour stages
# - Rejouabilité pour audit

import redis

class OrderProcessingPipeline:
    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)

        # Streams pour chaque stage
        self.streams = {
            'orders:received': 'payment-processors',
            'orders:paid': 'inventory-processors',
            'orders:shipped': 'notification-processors'
        }

        # Setup consumer groups
        for stream, group in self.streams.items():
            try:
                self.redis.xgroup_create(stream, group, '$', mkstream=True)
            except:
                pass

    def submit_order(self, order_data):
        """Nouvelle commande"""
        return self.redis.xadd('orders:received', order_data)

    def process_stage(self, stage: str, handler):
        """Traiter un stage du pipeline"""
        stream = f'orders:{stage}'
        group = self.streams.get(stream)

        messages = self.redis.xreadgroup(
            group,
            f'worker-{stage}',
            {stream: '>'},
            count=10,
            block=1000
        )

        if messages:
            for _, entries in messages:
                for msg_id, data in entries:
                    try:
                        # Process
                        result = handler(data)

                        # Move to next stage
                        if result.get('next_stream'):
                            self.redis.xadd(result['next_stream'], data)

                        # ACK
                        self.redis.xack(stream, group, msg_id)

                    except Exception as e:
                        print(f"Error: {e}")
                        # Retry automatique

# ❌ PAS Kafka : Overkill pour ce volume
# ❌ PAS Pub/Sub : Pas de persistence/retry
# ❌ PAS Lists : Pas de consumer groups
```

### 2. High-Volume IoT Data

```python
"""
Système IoT : 100,000 sensors × 1 msg/sec

Requirements:
- 100K messages/sec
- Rétention 90 jours
- Analytics sur historique
- Coût optimisé
"""

# ✅ SOLUTION: Kafka
# Pourquoi :
# - Throughput élevé (100K+/sec)
# - Long retention (90j)
# - Partitioning automatique
# - Écosystème analytics (Spark, Flink)

from kafka import KafkaProducer, KafkaConsumer
import json

class IoTDataPipeline:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

    def ingest_sensor_data(self, sensor_id: str, data: dict):
        """Ingérer données sensor"""
        # Partition par sensor_id pour ordre garanti
        self.producer.send(
            'iot-data',
            value=data,
            key=sensor_id.encode('utf-8')
        )

    def process_stream(self):
        """Consumer pour analytics temps réel"""
        consumer = KafkaConsumer(
            'iot-data',
            bootstrap_servers=['localhost:9092'],
            group_id='analytics-processors',
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )

        for message in consumer:
            self.analyze_data(message.value)

# ❌ PAS Redis Streams : Trop de RAM pour 90j
# ⚠️ Alternative : Redis Streams + S3 archiving
```

### 3. Microservices Event Bus

```python
"""
Event bus pour 50 microservices

Requirements:
- 10K events/sec
- Persistence 7 jours
- Retry automatique
- Low latency (< 10ms)
- Simple ops
"""

# ✅ SOLUTION: Redis Streams
# Pourquoi :
# - Throughput suffisant
# - Latence excellente
# - Ops simples
# - Déjà Redis dans stack

class EventBus:
    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)
        self.events = {
            'user.created': 'user-subscribers',
            'order.placed': 'order-subscribers',
            'payment.completed': 'payment-subscribers'
        }

        # Setup groups
        for event, group in self.events.items():
            try:
                self.redis.xgroup_create(
                    f'events:{event}',
                    group,
                    '$',
                    mkstream=True
                )
            except:
                pass

    def publish(self, event_type: str, payload: dict):
        """Publier événement"""
        return self.redis.xadd(
            f'events:{event_type}',
            payload,
            maxlen=100000  # Limiter à 100K (≈7 jours)
        )

    def subscribe(self, event_type: str, service_name: str, handler):
        """S'abonner à un type d'événement"""
        stream = f'events:{event_type}'
        group = self.events.get(event_type)

        while True:
            messages = self.redis.xreadgroup(
                group,
                service_name,
                {stream: '>'},
                count=10,
                block=1000
            )

            if messages:
                for _, entries in messages:
                    for msg_id, data in entries:
                        try:
                            handler(data)
                            self.redis.xack(stream, group, msg_id)
                        except Exception as e:
                            print(f"Error: {e}")

# ⚠️ Alternative : NATS (si ultra-low latency critique)
# ❌ PAS Kafka : Overkill pour ce volume
```

### 4. Real-Time Chat Application

```python
"""
Application chat temps réel

Requirements:
- < 100ms latency
- 50K concurrent users
- 1M messages/day
- Pas de persistence longue durée
"""

# ✅ SOLUTION: Redis Pub/Sub + Streams hybrid
# Pourquoi :
# - Pub/Sub pour delivery temps réel
# - Streams pour historique récent (24h)

class ChatSystem:
    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)

    def send_message(self, room_id: str, user_id: str, message: str):
        """Envoyer message"""
        data = {
            'user_id': user_id,
            'message': message,
            'timestamp': str(time.time())
        }

        # 1. Pub/Sub pour delivery immédiate
        self.redis.publish(f'chat:room:{room_id}', json.dumps(data))

        # 2. Streams pour historique (24h)
        self.redis.xadd(
            f'chat:room:{room_id}:history',
            data,
            maxlen=1000  # ~1000 derniers messages
        )

    def get_history(self, room_id: str, count: int = 50):
        """Récupérer historique"""
        return self.redis.xrevrange(
            f'chat:room:{room_id}:history',
            '+',
            '-',
            count=count
        )

    def listen_room(self, room_id: str):
        """Écouter messages temps réel"""
        pubsub = self.redis.pubsub()
        pubsub.subscribe(f'chat:room:{room_id}')

        for message in pubsub.listen():
            if message['type'] == 'message':
                yield json.loads(message['data'])

# ⚠️ Alternative : WebSocket + NATS
# ❌ PAS Streams seul : Latency trop élevée
# ❌ PAS Kafka : Overkill total
```

### 5. Financial Transaction Processing

```python
"""
Traitement transactions financières

Requirements:
- Exactly-once delivery
- Audit trail complet
- Rejouabilité illimitée
- Compliance (SOX, PCI-DSS)
"""

# ✅ SOLUTION: Kafka
# Pourquoi :
# - Exactly-once semantic natif
# - Retention infinie
# - Audit trail robuste
# - Compliance-ready

from kafka import KafkaProducer, KafkaConsumer

class TransactionProcessor:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            acks='all',  # Toutes les réplicas
            enable_idempotence=True,  # Exactly-once
            transactional_id='transaction-processor'
        )

        self.producer.init_transactions()

    def process_transaction(self, transaction: dict):
        """
        Traiter transaction avec exactly-once garantie
        """
        try:
            self.producer.begin_transaction()

            # 1. Publier transaction
            self.producer.send('transactions', value=transaction)

            # 2. Publier audit log
            self.producer.send('audit-log', value={
                'transaction_id': transaction['id'],
                'timestamp': time.time(),
                'status': 'processed'
            })

            # Commit transaction atomique
            self.producer.commit_transaction()

        except Exception as e:
            self.producer.abort_transaction()
            raise

# ⚠️ Alternative : Redis Streams + idempotency keys
# ❌ PAS Pub/Sub : Pas de persistence
# ❌ PAS Lists : Pas de rejouabilité
```

---

## Patterns Hybrides

### Pattern 1 : Redis + Kafka (Lambda Architecture)

```python
"""
Architecture Lambda : Speed layer (Redis) + Batch layer (Kafka)
"""

class HybridPipeline:
    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)
        self.kafka = KafkaProducer(bootstrap_servers=['localhost:9092'])

    def ingest_event(self, event_data: dict):
        """
        Dual write : Redis (temps réel) + Kafka (batch/analytics)
        """

        # 1. Redis Streams : Processing temps réel (hot path)
        self.redis.xadd('events:realtime', event_data)

        # 2. Kafka : Long-term storage + analytics (cold path)
        self.kafka.send('events-archive', value=json.dumps(event_data))

    def process_realtime(self):
        """Consumer temps réel (Redis)"""
        messages = self.redis.xreadgroup(
            'realtime-processors',
            'worker-1',
            {'events:realtime': '>'},
            count=100,
            block=100
        )

        # Process avec latence < 10ms
        # TTL court (1h) puis purge

    def process_batch(self):
        """Consumer batch (Kafka)"""
        consumer = KafkaConsumer(
            'events-archive',
            group_id='batch-analytics'
        )

        # Batch analytics, ML, reporting
        # Retention longue (90 jours+)

# Use case : E-commerce analytics
# - Redis : Dashboard temps réel, alerting
# - Kafka : Data warehouse, ML training
```

### Pattern 2 : Redis Pub/Sub + Streams (Hybrid Delivery)

```python
"""
Pattern : Pub/Sub (hot) + Streams (fallback)
"""

class HybridDelivery:
    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)

    def send_notification(self, user_id: str, notification: dict):
        """
        Dual delivery : Pub/Sub (si online) + Streams (backup)
        """
        channel = f'notifications:user:{user_id}'

        # 1. Pub/Sub pour delivery immédiate
        subscribers = self.redis.publish(channel, json.dumps(notification))

        # 2. Si aucun subscriber (user offline) → Streams
        if subscribers == 0:
            self.redis.xadd(
                f'notifications:user:{user_id}:inbox',
                notification,
                maxlen=100  # 100 dernières notifications
            )

    def consume_online(self, user_id: str):
        """Consumer online (Pub/Sub)"""
        pubsub = self.redis.pubsub()
        pubsub.subscribe(f'notifications:user:{user_id}')

        for message in pubsub.listen():
            if message['type'] == 'message':
                self.display_notification(message['data'])

    def consume_offline(self, user_id: str):
        """Consumer offline (Streams)"""
        # Récupérer notifications manquées
        inbox = f'notifications:user:{user_id}:inbox'

        messages = self.redis.xread({inbox: '0-0'}, count=100)

        if messages:
            for stream, entries in messages:
                for msg_id, data in entries:
                    self.display_notification(data)

                    # Cleanup après lecture
                    self.redis.xdel(inbox, msg_id)

# Use case : Mobile app notifications
# - Pub/Sub : User online, delivery immédiate
# - Streams : User offline, queue persistée
```

### Pattern 3 : Redis + RabbitMQ (Routing Complex)

```python
"""
Pattern : Redis (simple) + RabbitMQ (routing complexe)
"""

class HybridRouting:
    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)
        # self.rabbitmq = pika.BlockingConnection(...)

    def route_message(self, message: dict):
        """
        Router intelligent : Redis ou RabbitMQ selon complexité
        """

        # Simple routing → Redis Streams
        if message.get('routing') == 'simple':
            self.redis.xadd(f"queue:{message['queue']}", message)

        # Complex routing → RabbitMQ
        elif message.get('routing') == 'complex':
            # Topic exchange, priority, TTL, dead-letter
            # self.rabbitmq.basic_publish(...)
            pass

# Use case : Système avec besoins mixtes
# - 95% messages simples → Redis (performance)
# - 5% messages complexes → RabbitMQ (features)
```

---

## Coûts et Trade-offs

### Analyse des Coûts

```python
class CostAnalysis:
    """
    Analyse coûts pour 100M messages/jour
    """

    COSTS = {
        'redis': {
            'setup': 'Simple',
            'ram_gb_needed': 100,  # 1KB/msg × 100M
            'ram_cost_month': 500,  # $5/GB/month
            'disk_gb_needed': 10,
            'disk_cost_month': 5,
            'ops_complexity': 'Low',
            'ops_hours_month': 20,
            'total_month': 505
        },
        'kafka': {
            'setup': 'Complex',
            'ram_gb_needed': 32,
            'ram_cost_month': 160,
            'disk_gb_needed': 500,  # Retention + replication
            'disk_cost_month': 50,
            'ops_complexity': 'High',
            'ops_hours_month': 80,
            'total_month': 210 + (80 * 100)  # + ops cost
        },
        'rabbitmq': {
            'setup': 'Medium',
            'ram_gb_needed': 64,
            'ram_cost_month': 320,
            'disk_gb_needed': 200,
            'disk_cost_month': 20,
            'ops_complexity': 'Medium',
            'ops_hours_month': 40,
            'total_month': 340 + (40 * 100)
        }
    }

    @staticmethod
    def compare(msg_per_day: int, retention_days: int):
        """Comparer coûts"""
        print(f"\n{'=' * 60}")
        print(f"Cost Analysis : {msg_per_day:,} msg/day, {retention_days}d retention")
        print(f"{'=' * 60}")

        for system, costs in CostAnalysis.COSTS.items():
            print(f"\n{system.upper()}:")
            print(f"  Setup: {costs['setup']}")
            print(f"  RAM: {costs['ram_gb_needed']}GB @ ${costs['ram_cost_month']}/month")
            print(f"  Disk: {costs['disk_gb_needed']}GB @ ${costs['disk_cost_month']}/month")
            print(f"  Ops: {costs['ops_complexity']} ({costs['ops_hours_month']}h/month)")
            print(f"  TOTAL: ${costs['total_month']}/month")

# Run analysis
CostAnalysis.compare(100_000_000, 7)
```

### Trade-offs Matrix

| Aspect | Redis | Kafka | RabbitMQ |
|--------|-------|-------|----------|
| **Initial Setup** | ✅ 1 hour | ❌ 2-3 days | ⚠️ 1 day |
| **Learning Curve** | ✅ Gentle | ❌ Steep | ⚠️ Moderate |
| **Ops Complexity** | ✅ Low | ❌ High | ⚠️ Medium |
| **Cost (small)** | ✅✅ Low | ⚠️ Medium | ⚠️ Medium |
| **Cost (large)** | ❌ Very High | ✅ Optimized | ⚠️ High |
| **Debugging** | ✅ Easy | ❌ Hard | ⚠️ Medium |
| **Monitoring** | ✅ Simple | ❌ Complex | ⚠️ Medium |
| **Scaling** | ⚠️ Vertical | ✅✅ Horizontal | ⚠️ Vertical |

---

## Guide de Migration

### Redis Streams → Kafka

```python
"""
Migration progressive Redis Streams → Kafka
"""

class MigrationHelper:
    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)
        self.kafka = KafkaProducer(bootstrap_servers=['localhost:9092'])
        self.migration_percentage = 0  # 0-100

    def set_migration_percentage(self, percentage: int):
        """Contrôler % de traffic vers Kafka"""
        self.migration_percentage = percentage

    def dual_write(self, topic: str, data: dict):
        """
        Phase 1 : Dual write (Redis + Kafka)
        """
        # Toujours écrire dans Redis
        redis_id = self.redis.xadd(f'stream:{topic}', data)

        # Écrire dans Kafka selon %
        import random
        if random.randint(0, 100) < self.migration_percentage:
            self.kafka.send(topic, value=json.dumps(data))

            # Logger pour validation
            print(f"Dual write: Redis={redis_id}, Kafka=sent")

    def dual_read(self, topic: str, consumer_name: str):
        """
        Phase 2 : Dual read (Redis ou Kafka selon config)
        """
        import random

        if random.randint(0, 100) < self.migration_percentage:
            # Lire depuis Kafka
            return self._read_kafka(topic, consumer_name)
        else:
            # Lire depuis Redis
            return self._read_redis(topic, consumer_name)

    def _read_redis(self, topic, consumer):
        """Read from Redis"""
        messages = self.redis.xreadgroup(
            f'group-{topic}',
            consumer,
            {f'stream:{topic}': '>'},
            count=10
        )
        return messages

    def _read_kafka(self, topic, consumer):
        """Read from Kafka"""
        # kafka_consumer = KafkaConsumer(topic, group_id=f'group-{topic}')
        pass

# Plan de migration
# Semaine 1 : migration_percentage = 0   (Redis 100%)
# Semaine 2 : migration_percentage = 10  (Kafka 10%)
# Semaine 3 : migration_percentage = 25  (Kafka 25%)
# Semaine 4 : migration_percentage = 50  (Kafka 50%)
# Semaine 5 : migration_percentage = 100 (Kafka 100%)
# Semaine 6 : Désactiver Redis
```

---

## Checklist de Décision Finale

```python
def comprehensive_decision_tree():
    """
    Arbre de décision complet
    """

    print("""
    ÉTAPE 1 : VOLUME
    ────────────────
    • < 10K msg/sec       → Redis viable
    • 10K-100K msg/sec    → Redis ou Kafka
    • > 100K msg/sec      → Kafka obligatoire

    ÉTAPE 2 : RÉTENTION
    ────────────────────
    • < 24h               → Redis Pub/Sub ou Streams
    • 24h - 7j            → Redis Streams
    • 7j - 30j            → Redis Streams ou Kafka
    • > 30j               → Kafka obligatoire

    ÉTAPE 3 : GARANTIES
    ──────────────────
    • At-most-once        → Redis Pub/Sub
    • At-least-once       → Redis Streams ou Kafka
    • Exactly-once        → Redis Streams* ou Kafka
                           *avec idempotency keys

    ÉTAPE 4 : COMPLEXITÉ
    ───────────────────
    • Simple queue        → Redis Lists
    • Consumer groups     → Redis Streams
    • Complex routing     → RabbitMQ
    • Partitioning        → Kafka

    ÉTAPE 5 : LATENCE
    ────────────────
    • < 1ms               → Redis Pub/Sub
    • < 10ms              → Redis Streams ou NATS
    • < 100ms             → Tout système viable
    • > 100ms             → Kafka, batch processing

    ÉTAPE 6 : COÛT
    ─────────────
    • Budget limité       → Redis (si volume OK)
    • Volume élevé        → Kafka (coût optimisé)
    • Enterprise support  → Managed services

    DÉCISION FINALE :
    ────────────────
    Redis Pub/Sub    : Notifications, live updates
    Redis Streams    : Job queues, order processing
    Redis Lists      : Simple background tasks
    Kafka            : IoT, analytics, event sourcing
    RabbitMQ         : Complex routing, priority
    NATS             : Ultra-low latency, simple pub/sub
    """)

comprehensive_decision_tree()
```

### Tableau Récapitulatif Final

| Use Case | Volume | Latence | Rétention | **Recommandation** |
|----------|--------|---------|-----------|-------------------|
| Live dashboard | 10K/s | < 1ms | 0 | **Redis Pub/Sub** |
| Chat messages | 50K/s | < 10ms | 24h | **Redis Pub/Sub + Streams** |
| Job queue | 5K/s | < 50ms | 7j | **Redis Streams** |
| Order processing | 1K/s | < 100ms | 30j | **Redis Streams** |
| Email queue | 100/s | < 1s | 7j | **Redis Lists** |
| IoT ingestion | 100K/s | < 100ms | 90j | **Kafka** |
| Event sourcing | Variable | Variable | Infini | **Kafka** |
| Microservices events | 10K/s | < 10ms | 7j | **Redis Streams** |
| Financial transactions | 1K/s | < 100ms | Infini | **Kafka** |
| Log aggregation | 50K/s | < 100ms | 30j | **Kafka** |

---

## Conclusion

### Règles d'Or

1. **Commencer Simple**
   - Start avec Redis (Streams ou Pub/Sub)
   - Migrer vers Kafka seulement si nécessaire
   - Ne pas sur-engineer

2. **Redis est excellent pour**
   - Volumes < 50K msg/s
   - Latence < 10ms critique
   - Rétention < 30 jours
   - Stack déjà Redis
   - Ops simples requises

3. **Kafka est nécessaire pour**
   - Volumes > 100K msg/s
   - Rétention > 30 jours
   - Partitioning requis
   - Écosystème analytics
   - Multi-datacenter

4. **Patterns Hybrides**
   - Redis (hot path) + Kafka (cold path)
   - Pub/Sub (online) + Streams (offline)
   - Combiner forces de chaque système

5. **Considérations Finales**
   - Coût total (infra + ops)
   - Expertise équipe
   - Contraintes existantes
   - Plan de croissance

### Points Clés

- 🎯 **80% des cas** : Redis Streams suffit
- 🎯 **15% des cas** : Kafka nécessaire
- 🎯 **5% des cas** : Solutions spécialisées (RabbitMQ, NATS)
- 🎯 **Pattern hybride** : Souvent la meilleure solution
- 🎯 **Migration progressive** : Toujours valider avant full switch

**Conseil Final** : Choisissez le système le plus simple qui répond à vos besoins. Vous pourrez toujours migrer plus tard si nécessaire.

---


⏭️ [Intégration avec les langages de programmation](/09-integration-langages-programmation/README.md)
