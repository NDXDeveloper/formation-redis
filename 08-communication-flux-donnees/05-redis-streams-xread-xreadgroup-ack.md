🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.5 Redis Streams : XREAD vs XREADGROUP, Gestion des ACK

## Introduction

Redis Streams offre deux modèles de consommation fondamentalement différents : **XREAD** pour la lecture simple et **XREADGROUP** pour la lecture coordonnée avec consumer groups. Le choix entre les deux détermine les garanties de livraison, la gestion des échecs, et la complexité de l'architecture.

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────┐
│                      XREAD                              │
│  ┌─────────────────────────────────────────────┐        │
│  │ • Lecture directe du stream                 │        │
│  │ • Pas de tracking                           │        │
│  │ • Pas d'ACK                                 │        │
│  │ • Multiple consumers = duplication          │        │
│  │ • Simple et performant                      │        │
│  └─────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   XREADGROUP                            │
│  ┌─────────────────────────────────────────────┐        │
│  │ • Lecture via consumer group                │        │
│  │ • Tracking automatique (PEL)                │        │
│  │ • ACK obligatoires                          │        │
│  │ • Multiple consumers = distribution         │        │
│  │ • Fiable avec retry                         │        │
│  └─────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

---

## XREAD : Lecture Simple

### Caractéristiques

```python
# XREAD : Lecture directe sans tracking
messages = redis.xread({'stream': last_id}, count=10)

# Caractéristiques :
✅ Simplicité maximale
✅ Performance optimale
✅ Stateless (pas de state côté Redis)
✅ Rejouabilité totale

❌ Pas de distribution automatique
❌ Pas de tracking des messages traités
❌ Pas de retry automatique
❌ Consumer doit gérer son state
```

### Cas d'usage XREAD

```python
import redis
import time
from typing import Optional

class SimpleStreamReader:
    """
    Lecteur simple avec XREAD
    Idéal pour : logs, monitoring, debugging
    """

    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.last_id = '0-0'  # Géré localement

    def read_from_beginning(self):
        """Lire tout le stream depuis le début"""
        self.last_id = '0-0'
        return self.read_batch()

    def read_new_only(self):
        """Lire seulement les nouveaux messages"""
        self.last_id = '$'  # Position actuelle
        return self.read_batch()

    def read_batch(self, count: int = 100) -> list:
        """Lire un batch de messages"""
        messages = self.redis.xread(
            {self.stream_name: self.last_id},
            count=count
        )

        if messages:
            stream, entries = messages[0]

            # Mettre à jour last_id pour prochain read
            if entries:
                self.last_id = entries[-1][0]

            return entries

        return []

    def tail_forever(self, callback):
        """Tail -f style : suivre le stream en continu"""
        print(f"Tailing {self.stream_name}...")

        while True:
            messages = self.redis.xread(
                {self.stream_name: self.last_id},
                count=10,
                block=1000  # Block 1 seconde
            )

            if messages:
                stream, entries = messages[0]

                for msg_id, data in entries:
                    callback(msg_id, data)
                    self.last_id = msg_id

# Exemple d'utilisation
reader = SimpleStreamReader('logs:app')

# Tail style
reader.read_new_only()
reader.tail_forever(lambda msg_id, data: print(f"{msg_id}: {data}"))
```

### Pattern : Single Consumer avec Checkpointing

```python
class CheckpointedReader:
    """
    XREAD avec checkpointing manuel pour fiabilité
    """

    def __init__(self, stream_name: str, consumer_id: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.consumer_id = consumer_id
        self.checkpoint_key = f"checkpoint:{stream_name}:{consumer_id}"

    def load_checkpoint(self) -> str:
        """Charger dernière position sauvegardée"""
        checkpoint = self.redis.get(self.checkpoint_key)
        return checkpoint if checkpoint else '0-0'

    def save_checkpoint(self, message_id: str):
        """Sauvegarder position"""
        self.redis.set(self.checkpoint_key, message_id)

    def process_stream(self, handler):
        """Traiter stream avec checkpointing"""
        last_id = self.load_checkpoint()
        print(f"Starting from checkpoint: {last_id}")

        while True:
            messages = self.redis.xread(
                {self.stream_name: last_id},
                count=10,
                block=5000
            )

            if messages:
                stream, entries = messages[0]

                for msg_id, data in entries:
                    try:
                        # Traiter message
                        handler(msg_id, data)

                        # Sauvegarder checkpoint (commit)
                        self.save_checkpoint(msg_id)
                        last_id = msg_id

                    except Exception as e:
                        print(f"Error processing {msg_id}: {e}")
                        # Ne pas sauvegarder checkpoint = retry au restart
                        return

# Utilisation
reader = CheckpointedReader('orders:stream', 'processor-1')

def process_order(msg_id, data):
    print(f"Processing order: {data}")
    # Si exception → pas de checkpoint → rejouera au restart
    time.sleep(0.1)

reader.process_stream(process_order)
```

### Avantages et Inconvénients XREAD

| Avantages | Inconvénients |
|-----------|---------------|
| ✅ Simplicité maximale | ❌ Pas de coordination entre consumers |
| ✅ Performance optimale | ❌ Duplication si multiple readers |
| ✅ Stateless côté Redis | ❌ State management manuel |
| ✅ Rejouabilité facile | ❌ Pas de retry automatique |
| ✅ Pas de cleanup nécessaire | ❌ Pas de load balancing |
| ✅ Debugging facile | ❌ Pas de dead letter queue |

---

## XREADGROUP : Lecture Coordonnée

### Caractéristiques

```python
# XREADGROUP : Lecture avec consumer group
messages = redis.xreadgroup(
    'mygroup',
    'consumer-1',
    {'stream': '>'},
    count=10
)

# Caractéristiques :
✅ Distribution automatique
✅ Tracking via PEL (Pending Entry List)
✅ ACK obligatoires
✅ Retry automatique
✅ Load balancing natif
✅ Dead letter queue possible

❌ Plus complexe
❌ State côté Redis
❌ Nécessite cleanup (XACK)
⚠️ Performance légèrement inférieure
```

### Cas d'usage XREADGROUP

```python
class ConsumerGroupReader:
    """
    Lecteur avec consumer group
    Idéal pour : job processing, order processing, events
    """

    def __init__(
        self,
        stream_name: str,
        group_name: str,
        consumer_name: str
    ):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name
        self.consumer_name = consumer_name

    def read_new_messages(self, count: int = 10):
        """Lire nouveaux messages (non encore livrés au group)"""
        messages = self.redis.xreadgroup(
            self.group_name,
            self.consumer_name,
            {self.stream_name: '>'},  # > = nouveaux messages
            count=count,
            block=1000
        )

        if messages:
            return messages[0][1]  # Liste de (msg_id, data)

        return []

    def read_pending_messages(self, count: int = 10):
        """Lire messages pending de CE consumer"""
        messages = self.redis.xreadgroup(
            self.group_name,
            self.consumer_name,
            {self.stream_name: '0'},  # 0 = pending
            count=count
        )

        if messages:
            return messages[0][1]

        return []

    def acknowledge(self, message_id: str):
        """Acquitter un message traité"""
        return self.redis.xack(
            self.stream_name,
            self.group_name,
            message_id
        )

    def process_with_ack(self, handler):
        """Traiter avec ACK automatique"""
        while True:
            # 1. Lire nouveaux messages
            messages = self.read_new_messages(count=10)

            for msg_id, data in messages:
                try:
                    # Traiter
                    handler(msg_id, data)

                    # ACK seulement si succès
                    self.acknowledge(msg_id)
                    print(f"✓ ACKed {msg_id}")

                except Exception as e:
                    print(f"✗ Error processing {msg_id}: {e}")
                    # Pas d'ACK → reste en pending

# Utilisation
reader = ConsumerGroupReader('orders:stream', 'processors', 'worker-1')

def process_order(msg_id, data):
    print(f"Processing: {data}")
    # Si exception → pas d'ACK → retry automatique

reader.process_with_ack(process_order)
```

### Avantages et Inconvénients XREADGROUP

| Avantages | Inconvénients |
|-----------|---------------|
| ✅ Distribution automatique | ❌ Plus complexe à setup |
| ✅ Load balancing natif | ❌ State côté Redis (PEL) |
| ✅ Retry automatique | ❌ Nécessite cleanup (XACK) |
| ✅ Tracking des échecs | ⚠️ Performance légèrement moindre |
| ✅ Dead letter queue | ❌ Debugging plus complexe |
| ✅ Scalable horizontalement | ❌ Coût mémoire du PEL |

---

## Gestion des ACK : Patterns et Garanties

### Les 3 Niveaux de Garantie

```
┌─────────────────────────────────────────────────────────┐
│  1. AT-MOST-ONCE (≤ 1 delivery)                         │
│     • XREAD sans checkpoint                             │
│     • XREADGROUP avec NOACK                             │
│     • Perte possible, pas de duplication                │
│     • Performance maximale                              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  2. AT-LEAST-ONCE (≥ 1 delivery)                        │
│     • XREADGROUP avec ACK après traitement              │
│     • Pas de perte, duplication possible                │
│     • Standard pour la plupart des cas                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  3. EXACTLY-ONCE (= 1 delivery)                         │
│     • XREADGROUP + Idempotency key                      │
│     • Ni perte ni duplication                           │
│     • Complexité maximale                               │
└─────────────────────────────────────────────────────────┘
```

### Pattern 1 : At-Most-Once (Fire and Forget)

```python
class AtMostOnceProcessor:
    """
    Traitement at-most-once avec NOACK
    Message peut être perdu mais jamais dupliqué
    """

    def __init__(self, stream_name: str, group_name: str, consumer_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name
        self.consumer_name = consumer_name

    def process_fire_and_forget(self, handler):
        """Traiter sans ACK (NOACK)"""
        while True:
            # NOACK = pas d'ajout à PEL
            messages = self.redis.execute_command(
                'XREADGROUP',
                'GROUP', self.group_name, self.consumer_name,
                'COUNT', '10',
                'BLOCK', '1000',
                'NOACK',  # ← Pas de tracking!
                'STREAMS', self.stream_name, '>'
            )

            if messages and messages[0]:
                stream_data = messages[0][1]

                for msg in stream_data:
                    msg_id = msg[0]
                    fields = msg[1]
                    data = dict(zip(fields[::2], fields[1::2]))

                    try:
                        handler(msg_id, data)
                    except Exception as e:
                        # Message perdu si erreur!
                        print(f"✗ Lost message {msg_id}: {e}")

# Utilisation
processor = AtMostOnceProcessor('metrics:stream', 'analytics', 'worker-1')

def process_metric(msg_id, data):
    print(f"Metric: {data}")
    # Si crash ici → message perdu

processor.process_fire_and_forget(process_metric)
```

**Quand utiliser** :
- Metrics non-critiques
- Logs
- Best-effort notifications
- Performance critique > fiabilité

### Pattern 2 : At-Least-Once (Standard)

```python
class AtLeastOnceProcessor:
    """
    Traitement at-least-once
    Garantie de traitement, duplication possible
    """

    def __init__(self, stream_name: str, group_name: str, consumer_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name
        self.consumer_name = consumer_name

    def process_at_least_once(self, handler):
        """
        Pattern standard :
        1. Lire message (ajouté à PEL)
        2. Traiter
        3. ACK seulement si succès
        """
        while True:
            messages = self.redis.xreadgroup(
                self.group_name,
                self.consumer_name,
                {self.stream_name: '>'},
                count=10,
                block=1000
            )

            if messages:
                for stream, entries in messages:
                    for msg_id, data in entries:
                        try:
                            # Traiter
                            handler(msg_id, data)

                            # ACK APRÈS traitement
                            self.redis.xack(
                                self.stream_name,
                                self.group_name,
                                msg_id
                            )
                            print(f"✓ Processed & ACKed {msg_id}")

                        except Exception as e:
                            print(f"✗ Error {msg_id}: {e}")
                            # Pas d'ACK → reste en PEL → retry

# Utilisation
processor = AtLeastOnceProcessor('orders:stream', 'processors', 'worker-1')

def process_order(msg_id, data):
    print(f"Processing order: {data}")

    # Traitement NON-idempotent
    # Peut être appelé plusieurs fois pour même message!

    # Si crash AVANT ACK → message retraité
    # Si crash APRÈS ACK → message définitivement traité

processor.process_at_least_once(process_order)
```

**Quand utiliser** :
- Cas d'usage standard (80% des cas)
- Order processing
- Payment processing (avec idempotency)
- Email sending (avec deduplication)

### Pattern 3 : Exactly-Once Semantic

```python
class ExactlyOnceProcessor:
    """
    Traitement exactly-once avec idempotency
    Ni perte ni duplication
    """

    def __init__(
        self,
        stream_name: str,
        group_name: str,
        consumer_name: str
    ):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name
        self.consumer_name = consumer_name

    def process_exactly_once(self, handler):
        """
        Pattern exactly-once :
        1. Lire message
        2. Vérifier idempotency key
        3. Traiter si pas déjà fait
        4. Sauvegarder idempotency key + ACK atomiquement
        """
        while True:
            messages = self.redis.xreadgroup(
                self.group_name,
                self.consumer_name,
                {self.stream_name: '>'},
                count=10,
                block=1000
            )

            if messages:
                for stream, entries in messages:
                    for msg_id, data in entries:
                        idempotency_key = f"processed:{self.stream_name}:{msg_id}"

                        try:
                            # Vérifier si déjà traité
                            if self.redis.exists(idempotency_key):
                                print(f"⊗ Already processed {msg_id}, skipping")
                                self.redis.xack(
                                    self.stream_name,
                                    self.group_name,
                                    msg_id
                                )
                                continue

                            # Traiter
                            result = handler(msg_id, data)

                            # Transaction atomique : save result + ACK
                            pipe = self.redis.pipeline()

                            # Marquer comme traité (TTL pour cleanup)
                            pipe.setex(
                                idempotency_key,
                                86400,  # 24h
                                '1'
                            )

                            # ACK
                            pipe.xack(
                                self.stream_name,
                                self.group_name,
                                msg_id
                            )

                            pipe.execute()

                            print(f"✓ Exactly-once processed {msg_id}")

                        except Exception as e:
                            print(f"✗ Error {msg_id}: {e}")
                            # Pas d'idempotency key ni ACK → retry

# Utilisation
processor = ExactlyOnceProcessor('payments:stream', 'processors', 'worker-1')

def process_payment(msg_id, data):
    print(f"Processing payment: {data}")

    # Traitement peut être non-idempotent
    # Grâce à idempotency key, exécuté une seule fois

    amount = float(data['amount'])
    # Débiter compte, etc.

    return {'status': 'success', 'amount': amount}

processor.process_exactly_once(process_payment)
```

**Quand utiliser** :
- Payments / financial transactions
- Inventory management
- Account balance updates
- Tout système où duplication = problème métier

### Pattern 4 : ACK optimiste (avant traitement)

```python
class OptimisticAckProcessor:
    """
    ACK optimiste : ACK AVANT traitement
    ⚠️ Dangereux : perte possible si crash
    """

    def __init__(self, stream_name: str, group_name: str, consumer_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name
        self.consumer_name = consumer_name

    def process_optimistic(self, handler):
        """ACK immédiatement, traiter après"""
        while True:
            messages = self.redis.xreadgroup(
                self.group_name,
                self.consumer_name,
                {self.stream_name: '>'},
                count=10,
                block=1000
            )

            if messages:
                for stream, entries in messages:
                    for msg_id, data in entries:
                        # ACK IMMÉDIATEMENT
                        self.redis.xack(
                            self.stream_name,
                            self.group_name,
                            msg_id
                        )

                        try:
                            # Traiter APRÈS ACK
                            handler(msg_id, data)
                            print(f"✓ Processed {msg_id}")

                        except Exception as e:
                            # ⚠️ Message déjà ACKé = PERDU!
                            print(f"✗ LOST message {msg_id}: {e}")

                            # Fallback : écrire vers DLQ
                            self.save_to_dlq(msg_id, data, str(e))

    def save_to_dlq(self, msg_id, data, error):
        """Sauvegarder en DLQ pour investigation manuelle"""
        self.redis.xadd(
            f"{self.stream_name}:dlq",
            {
                'original_id': msg_id,
                'data': json.dumps(data),
                'error': error,
                'timestamp': str(time.time())
            }
        )

# ⚠️ UTILISER SEULEMENT SI :
# - Performance critique
# - Perte acceptable
# - Avec fallback DLQ
```

**Quand utiliser** :
- Performance > fiabilité
- Messages non-critiques
- Avec DLQ comme safety net
- Jamais pour financial/critical data

---

## Comparaison Détaillée

### Tableau de Décision Complet

| Critère | XREAD | XREADGROUP |
|---------|-------|------------|
| **Setup** | Aucun | Créer group |
| **State management** | Manuel (app) | Automatique (Redis) |
| **Multiple consumers** | Duplication | Distribution |
| **Load balancing** | ❌ Manuel | ✅ Automatique |
| **ACK requis** | ❌ Non | ✅ Oui |
| **Retry automatique** | ❌ Non | ✅ Via PEL |
| **Dead messages** | ❌ Non | ✅ XPENDING |
| **Rejouabilité** | ✅ Totale | ✅ Partielle |
| **Performance** | ✅✅ Optimale | ✅ Excellente |
| **Overhead mémoire** | Minimal | PEL storage |
| **Complexité** | Très simple | Moyenne |
| **Debugging** | Facile | Plus complexe |
| **Guarantees** | At-most-once | At-least-once |
| **Idempotence** | Pas requis | Recommandé |

### Matrice de Choix

```python
def choose_consumption_model(requirements):
    """
    Aide à choisir entre XREAD et XREADGROUP
    """

    # XREADGROUP obligatoire si :
    if (requirements['multiple_consumers'] and
        requirements['load_balancing']):
        return "XREADGROUP"

    if requirements['guaranteed_processing']:
        return "XREADGROUP"

    if requirements['retry_on_failure']:
        return "XREADGROUP"

    # XREAD suffit si :
    if (requirements['single_consumer'] and
        requirements['simple_processing']):
        return "XREAD"

    if requirements['read_only'] or requirements['debugging']:
        return "XREAD"

    if requirements['tail_logs']:
        return "XREAD"

    # Par défaut : XREADGROUP pour robustesse
    return "XREADGROUP"

# Exemples
print(choose_consumption_model({
    'multiple_consumers': True,
    'load_balancing': True,
    'guaranteed_processing': True,
    'retry_on_failure': True
}))
# → "XREADGROUP"

print(choose_consumption_model({
    'single_consumer': True,
    'simple_processing': True,
    'read_only': True,
    'debugging': False
}))
# → "XREAD"
```

---

## Patterns de Migration

### De XREAD vers XREADGROUP

```python
class MigrationHelper:
    """
    Helper pour migrer de XREAD vers XREADGROUP
    """

    def __init__(self, stream_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name

    def find_last_processed_id(self, checkpoint_key: str) -> str:
        """Trouver dernier message traité (depuis checkpoint XREAD)"""
        last_id = self.redis.get(checkpoint_key)
        return last_id if last_id else '0-0'

    def create_group_from_checkpoint(
        self,
        group_name: str,
        checkpoint_key: str
    ):
        """Créer group en continuant depuis checkpoint"""
        last_processed = self.find_last_processed_id(checkpoint_key)

        try:
            self.redis.xgroup_create(
                self.stream_name,
                group_name,
                last_processed,  # Partir d'où XREAD s'était arrêté
                mkstream=True
            )
            print(f"✓ Created group from checkpoint {last_processed}")
        except redis.exceptions.ResponseError as e:
            if 'BUSYGROUP' in str(e):
                print(f"Group already exists")
            else:
                raise

    def dual_mode_processor(
        self,
        group_name: str,
        consumer_name: str,
        checkpoint_key: str,
        use_group: bool
    ):
        """
        Processor qui supporte les deux modes
        Permet migration progressive
        """
        if use_group:
            # Mode XREADGROUP
            messages = self.redis.xreadgroup(
                group_name,
                consumer_name,
                {self.stream_name: '>'},
                count=10,
                block=1000
            )

            if messages:
                for stream, entries in messages:
                    for msg_id, data in entries:
                        # Traiter
                        self.process_message(msg_id, data)

                        # ACK
                        self.redis.xack(self.stream_name, group_name, msg_id)

        else:
            # Mode XREAD
            last_id = self.find_last_processed_id(checkpoint_key)

            messages = self.redis.xread(
                {self.stream_name: last_id},
                count=10,
                block=1000
            )

            if messages:
                stream, entries = messages[0]

                for msg_id, data in entries:
                    # Traiter
                    self.process_message(msg_id, data)

                    # Checkpoint
                    self.redis.set(checkpoint_key, msg_id)

    def process_message(self, msg_id, data):
        """Logique de traitement (identique dans les 2 modes)"""
        print(f"Processing {msg_id}: {data}")

# Plan de migration
helper = MigrationHelper('orders:stream')

# Étape 1 : Mode XREAD actuel
# helper.dual_mode_processor('group', 'consumer-1', 'checkpoint:old', use_group=False)

# Étape 2 : Créer group depuis checkpoint
helper.create_group_from_checkpoint('processors', 'checkpoint:old')

# Étape 3 : Basculer vers XREADGROUP
# helper.dual_mode_processor('processors', 'consumer-1', 'checkpoint:old', use_group=True)
```

---

## Best Practices

### 1. Quand utiliser XREAD

```python
# ✅ BON : Lecture pour debugging
def tail_stream_for_debug(stream_name):
    reader = SimpleStreamReader(stream_name)
    reader.read_new_only()
    reader.tail_forever(lambda id, data: print(f"{id}: {data}"))

# ✅ BON : Single consumer, traitement simple
def process_logs(stream_name):
    reader = SimpleStreamReader(stream_name)
    while True:
        messages = reader.read_batch(count=100)
        for msg_id, data in messages:
            save_to_elasticsearch(data)

# ✅ BON : Fan-out avec plusieurs XREAD indépendants
def multiple_independent_readers(stream_name):
    # Reader 1 : Analytics
    analytics = SimpleStreamReader(stream_name)

    # Reader 2 : Archiving
    archiver = SimpleStreamReader(stream_name)

    # Les deux lisent TOUT le stream indépendamment
    # Pas de coordination nécessaire
```

### 2. Quand utiliser XREADGROUP

```python
# ✅ BON : Multiple workers, load balancing
def scalable_job_processing(stream_name, group_name):
    # Worker 1
    worker1 = ConsumerGroupReader(stream_name, group_name, 'worker-1')

    # Worker 2
    worker2 = ConsumerGroupReader(stream_name, group_name, 'worker-2')

    # Messages distribués automatiquement

# ✅ BON : Guaranteed processing
def critical_order_processing(stream_name, group_name):
    processor = AtLeastOnceProcessor(stream_name, group_name, 'processor-1')
    processor.process_at_least_once(handle_order)

# ✅ BON : Retry automatique
def resilient_processing(stream_name, group_name):
    consumer = StreamConsumer(
        stream_name,
        group_name,
        'consumer-1',
        max_retries=3
    )
    consumer.start(handler)
```

### 3. Gestion des ACK

```python
class ACKBestPractices:
    """Best practices pour ACK"""

    def __init__(self, stream_name, group_name, consumer_name):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name
        self.consumer_name = consumer_name

    def pattern_safe_ack(self, handler):
        """
        Pattern safe : ACK après traitement complet
        """
        messages = self.redis.xreadgroup(
            self.group_name,
            self.consumer_name,
            {self.stream_name: '>'},
            count=10
        )

        if messages:
            for stream, entries in messages:
                for msg_id, data in entries:
                    try:
                        # 1. Traiter complètement
                        handler(msg_id, data)

                        # 2. Persister résultat si nécessaire
                        # save_result(msg_id, result)

                        # 3. ACK seulement si tout OK
                        self.redis.xack(
                            self.stream_name,
                            self.group_name,
                            msg_id
                        )

                    except Exception as e:
                        # Pas d'ACK = retry automatique
                        logger.error(f"Error {msg_id}: {e}")

    def pattern_batch_ack(self, handler):
        """
        Pattern batch : ACK plusieurs messages d'un coup
        """
        messages = self.redis.xreadgroup(
            self.group_name,
            self.consumer_name,
            {self.stream_name: '>'},
            count=100  # Batch plus grand
        )

        if messages:
            stream, entries = messages[0]

            processed_ids = []

            for msg_id, data in entries:
                try:
                    handler(msg_id, data)
                    processed_ids.append(msg_id)

                except Exception as e:
                    logger.error(f"Error {msg_id}: {e}")

            # ACK batch
            if processed_ids:
                self.redis.xack(
                    self.stream_name,
                    self.group_name,
                    *processed_ids
                )
                logger.info(f"Batch ACKed {len(processed_ids)} messages")

    def pattern_deferred_ack(self, handler):
        """
        Pattern deferred : ACK après async operation
        """
        import asyncio

        messages = self.redis.xreadgroup(
            self.group_name,
            self.consumer_name,
            {self.stream_name: '>'},
            count=10
        )

        if messages:
            tasks = []

            for stream, entries in messages:
                for msg_id, data in entries:
                    # Lancer async
                    task = self._process_async(msg_id, data, handler)
                    tasks.append(task)

            # Attendre toutes les tasks
            results = asyncio.run(asyncio.gather(*tasks))

            # ACK celles qui ont réussi
            for msg_id, success in results:
                if success:
                    self.redis.xack(
                        self.stream_name,
                        self.group_name,
                        msg_id
                    )

    async def _process_async(self, msg_id, data, handler):
        """Process async avec ACK différé"""
        try:
            await handler(msg_id, data)
            return (msg_id, True)
        except Exception as e:
            logger.error(f"Error {msg_id}: {e}")
            return (msg_id, False)
```

### 4. Monitoring des ACK

```python
class ACKMonitor:
    """Monitoring de la santé des ACK"""

    def __init__(self, stream_name: str, group_name: str):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.stream_name = stream_name
        self.group_name = group_name

    def get_pending_stats(self) -> dict:
        """Statistiques des messages pending"""
        pending = self.redis.xpending(self.stream_name, self.group_name)

        if not pending or pending[0] == 0:
            return {'total_pending': 0}

        return {
            'total_pending': pending[0],
            'min_id': pending[1],
            'max_id': pending[2],
            'consumers': pending[3]
        }

    def get_unacked_messages(self, min_idle_ms: int = 60000) -> list:
        """Trouver messages non-ACKés depuis longtemps"""
        pending_details = self.redis.xpending_range(
            self.stream_name,
            self.group_name,
            '-',
            '+',
            count=100
        )

        unacked = []
        for entry in pending_details:
            if entry['time_since_delivered'] >= min_idle_ms:
                unacked.append({
                    'message_id': entry['message_id'],
                    'consumer': entry['consumer'],
                    'idle_ms': entry['time_since_delivered'],
                    'delivery_count': entry['times_delivered']
                })

        return unacked

    def alert_if_stuck(self, threshold: int = 100):
        """Alerter si trop de messages stuck"""
        stats = self.get_pending_stats()

        if stats['total_pending'] > threshold:
            print(f"⚠️  ALERT: {stats['total_pending']} pending messages")

            unacked = self.get_unacked_messages()
            print(f"   {len(unacked)} stuck messages (idle > 60s)")

            return True

        return False

    def print_health(self):
        """Afficher santé globale"""
        stats = self.get_pending_stats()

        print(f"\n{'=' * 60}")
        print(f"ACK Health: {self.stream_name} / {self.group_name}")
        print(f"{'=' * 60}")
        print(f"Total pending: {stats.get('total_pending', 0)}")

        if stats.get('total_pending', 0) > 0:
            unacked = self.get_unacked_messages()
            print(f"Stuck messages (>60s): {len(unacked)}")

            if unacked:
                print(f"\nTop 5 stuck messages:")
                for msg in unacked[:5]:
                    print(f"  {msg['message_id']}:")
                    print(f"    Consumer: {msg['consumer']}")
                    print(f"    Idle: {msg['idle_ms']/1000:.1f}s")
                    print(f"    Attempts: {msg['delivery_count']}")

# Utilisation
monitor = ACKMonitor('orders:stream', 'processors')
monitor.print_health()

# Alerting périodique
import schedule
schedule.every(1).minutes.do(monitor.alert_if_stuck)
```

---

## Cas d'Usage et Recommandations

### Matrice de Décision Finale

| Scenario | Recommandation | Garantie | Pourquoi |
|----------|---------------|----------|----------|
| **Logs aggregation** | XREAD | At-most-once | Simple, perte acceptable |
| **Metrics collection** | XREAD ou XREADGROUP+NOACK | At-most-once | Performance > fiabilité |
| **Job processing** | XREADGROUP | At-least-once | Distribution + retry |
| **Order processing** | XREADGROUP + idempotency | Exactly-once | Critical data |
| **Email sending** | XREADGROUP + dedup | At-least-once | Retry OK si dedupliqué |
| **Payment processing** | XREADGROUP + idempotency | Exactly-once | Financial critical |
| **Analytics pipeline** | Multiple XREAD | N/A | Fan-out indépendant |
| **Debugging** | XREAD | N/A | Read-only, pas de side effects |

### Checklist de Décision

```python
def decision_checklist():
    """
    Checklist pour choisir pattern de consommation
    """

    questions = {
        'multiple_consumers': "Plusieurs consumers en parallèle ?",
        'load_balancing': "Besoin de load balancing automatique ?",
        'guaranteed_delivery': "Garantie de traitement requise ?",
        'retry_needed': "Retry automatique nécessaire ?",
        'idempotent': "Traitement idempotent ?",
        'critical_data': "Données critiques (finance, etc.) ?",
        'performance_critical': "Performance ultra-critique ?",
        'simple_use_case': "Cas d'usage simple (logs, debug) ?"
    }

    # Score
    use_xreadgroup = 0
    use_xread = 0

    # Répondre aux questions
    for key, question in questions.items():
        # answer = input(f"{question} (y/n): ").lower() == 'y'
        answer = True  # Exemple

        if key in ['multiple_consumers', 'load_balancing',
                   'guaranteed_delivery', 'retry_needed']:
            if answer:
                use_xreadgroup += 1

        if key in ['simple_use_case', 'performance_critical']:
            if answer:
                use_xread += 1

    # Décision
    if use_xreadgroup > use_xread:
        print("\n✓ Recommandation : XREADGROUP")

        # Niveau de garantie
        if questions['critical_data']:
            print("  → Avec exactly-once (idempotency)")
        else:
            print("  → Avec at-least-once (standard)")
    else:
        print("\n✓ Recommandation : XREAD")
        print("  → Avec checkpointing si fiabilité importante")
```

---

## Conclusion

### Règles d'Or

1. **Par défaut → XREADGROUP**
   - Plus robuste, scalable, maintenance facile
   - Overhead acceptable pour la plupart des cas

2. **XREAD pour cas spécifiques**
   - Debugging, monitoring, logs
   - Single consumer avec traitement simple
   - Performance absolument critique

3. **Garanties de livraison**
   - At-most-once → XREAD ou NOACK
   - At-least-once → XREADGROUP standard
   - Exactly-once → XREADGROUP + idempotency

4. **ACK Strategy**
   - ACK après traitement complet (safe)
   - Batch ACK si performance critique
   - Never ACK avant traitement (sauf cas très spécifiques)

5. **Monitoring**
   - Toujours monitorer PEL size
   - Alerter sur messages stuck
   - Cleanup régulier des dead consumers

### Points Clés

- 🎯 **XREAD** = Simplicité, performance, stateless
- 🎯 **XREADGROUP** = Distribution, fiabilité, retry
- 🎯 **ACK** = Garanties de traitement
- 🎯 **Idempotency** = Exactly-once semantic
- 🎯 **PEL** = État des messages en attente

**En pratique** : 80% des cas utilisent XREADGROUP avec at-least-once, 15% utilisent XREAD, 5% nécessitent exactly-once.

---


⏭️ [Choix architectural : Pub/Sub vs Streams vs Lists vs Kafka](/08-communication-flux-donnees/06-choix-architectural-pubsub-streams-kafka.md)
