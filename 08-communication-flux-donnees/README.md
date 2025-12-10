🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 8 : Communication et Flux de Données

## Introduction au module

La communication asynchrone et le traitement de flux de données sont devenus des composants essentiels des architectures modernes distribuées. Redis offre plusieurs mécanismes de messaging, chacun avec ses propres caractéristiques, garanties et cas d'usage optimaux.

Ce module explore en profondeur les trois principaux mécanismes de communication dans Redis :
- **Pub/Sub** : Communication en temps réel avec modèle publisher-subscriber
- **Redis Streams** : Flux de données persistants avec consumer groups
- **Lists** : Files d'attente simples pour patterns producteur-consommateur

---

## Vue d'ensemble des mécanismes de messaging

### Les trois paradigmes de Redis

Redis propose trois approches distinctes pour la communication inter-processus et le traitement de flux :

| Mécanisme | Paradigme | Persistance | Complexité | Cas d'usage principal |
|-----------|-----------|-------------|------------|----------------------|
| **Pub/Sub** | Fire-and-forget | ❌ Non | Faible | Notifications temps réel |
| **Streams** | Event sourcing | ✅ Oui | Moyenne | Traitement fiable de flux |
| **Lists** | FIFO/LIFO Queue | ✅ Oui | Faible | Job queues simples |

### Matrice décisionnelle rapide

```
Besoin de garantie de livraison ?
    ├─ NON → Pub/Sub (performance maximale)
    └─ OUI →
        ├─ Traitement parallèle avec consumer groups ?
        │   └─ OUI → Redis Streams
        └─ NON →
            └─ File d'attente simple ?
                └─ OUI → Lists (LPUSH/BRPOP)
```

---

## Pub/Sub vs Streams : Comparaison détaillée

### 1. Garanties de livraison

#### Pub/Sub : "Fire and Forget"

```bash
# Publisher
PUBLISH notifications:users "User 123 logged in"
# Retourne : (integer) 2  # Nombre de subscribers actifs

# Subscriber (doit être connecté AVANT la publication)
SUBSCRIBE notifications:users
# Si déconnecté → message perdu définitivement
```

**Caractéristiques** :
- ❌ Aucune persistance
- ❌ Les messages non livrés sont perdus
- ✅ Latence ultra-faible (microseconds)
- ✅ Idéal pour événements non critiques

#### Streams : Persistance et rejouabilité

```bash
# Producer
XADD events:login * user_id 123 timestamp 1702234567
# Retourne : "1702234567000-0"  # ID du message stocké

# Consumer - peut lire depuis le début
XREAD COUNT 10 STREAMS events:login 0-0
# Messages persistés jusqu'à suppression explicite
```

**Caractéristiques** :
- ✅ Persistance complète (RDB/AOF)
- ✅ Rejouabilité illimitée
- ✅ Lecture depuis n'importe quel point
- ⚠️ Latence légèrement supérieure (millisecondes)

---

### 2. Modèle de consommation

#### Pub/Sub : Broadcast simple

```bash
# Scénario : Notification temps réel à plusieurs services

# Publisher
PUBLISH user:events '{"type":"login","user_id":123}'

# Subscriber 1 (Service Analytics)
SUBSCRIBE user:events
# Reçoit le message

# Subscriber 2 (Service Notifications)
SUBSCRIBE user:events
# Reçoit le MÊME message simultanément

# Subscriber 3 (connecté 1 seconde trop tard)
SUBSCRIBE user:events
# Ne recevra PAS les messages précédents
```

**Cas d'usage Pub/Sub** :
- Dashboard temps réel (WebSockets)
- Invalidation de cache distribuée
- Notifications push
- Live chat/messaging

#### Streams : Consumer Groups avec ACK

```bash
# Création du consumer group
XGROUP CREATE orders:stream payment-processor $ MKSTREAM

# Consumer 1 dans le groupe
XREADGROUP GROUP payment-processor consumer-1 COUNT 1 STREAMS orders:stream >
# Reçoit : 1) ordre ID 1702234567000-0

# Consumer 2 dans le même groupe
XREADGROUP GROUP payment-processor consumer-2 COUNT 1 STREAMS orders:stream >
# Reçoit : 2) ordre ID 1702234568000-0  (différent de consumer-1)

# Acknowledgement après traitement
XACK orders:stream payment-processor 1702234567000-0

# En cas d'échec, message réattribué automatiquement
XPENDING orders:stream payment-processor  # Voir messages non-ACKés
```

**Cas d'usage Streams** :
- Pipeline de traitement de données
- Event sourcing
- Systèmes de facturation/paiement
- Audit trail avec rejouabilité

---

### 3. Scalabilité horizontale

#### Pub/Sub : Scalabilité limitée

```python
# Problème : Tous les subscribers reçoivent TOUS les messages
import redis

# 10 workers qui écoutent le même channel
for i in range(10):
    worker = redis.Redis()
    pubsub = worker.pubsub()
    pubsub.subscribe('tasks')
    # Problème : Les 10 workers recevront chaque message
    # → Travail dupliqué 10 fois !
```

**Solution Pub/Sub pour load balancing** :
```python
# Pattern Sharded Pub/Sub (Redis 7+)
# Distribue automatiquement entre subscribers

# Publisher
redis.spublish('tasks:{shard}', task_data)

# Les subscribers sont automatiquement répartis
pubsub.ssubscribe('tasks:{shard}')
```

#### Streams : Load balancing natif

```python
import redis

# Consumer Group : chaque message traité par UN SEUL consumer
r = redis.Redis()

# Worker 1
while True:
    messages = r.xreadgroup(
        groupname='workers',
        consumername='worker-1',
        streams={'tasks': '>'},
        count=10,
        block=5000
    )
    for message in messages:
        process(message)
        r.xack('tasks', 'workers', message[0])

# Worker 2, 3, 4... traiteront des messages DIFFÉRENTS
# → Distribution automatique de la charge
```

---

### 4. Gestion des erreurs et retry

#### Pub/Sub : Aucune gestion native

```python
# Pub/Sub : Si le traitement échoue, le message est perdu
pubsub = redis.pubsub()
pubsub.subscribe('tasks')

for message in pubsub.listen():
    try:
        process(message['data'])
    except Exception as e:
        # ❌ Impossible de rejouer le message
        # ❌ Doit implémenter sa propre logique de retry
        logger.error(f"Message perdu : {e}")
```

#### Streams : Retry et Dead Letter Queue

```python
import redis
from datetime import datetime, timedelta

r = redis.Redis()

# 1. Traitement avec retry automatique
def process_with_retry(stream_name, group_name, consumer_name):
    while True:
        # Lire nouveaux messages
        messages = r.xreadgroup(group_name, consumer_name, {stream_name: '>'}, count=10)

        for message in messages:
            msg_id, data = message[1][0]
            try:
                process(data)
                r.xack(stream_name, group_name, msg_id)
            except Exception as e:
                # Ne pas ACK → sera retraité
                logger.error(f"Erreur: {e}, message sera retryé")

        # 2. Récupérer les messages en timeout (non-ACKés > 5 min)
        pending = r.xpending_range(
            stream_name,
            group_name,
            '-', '+',
            count=100
        )

        now = datetime.now()
        for entry in pending:
            # entry: [msg_id, consumer, idle_time, delivery_count]
            if entry['time_since_delivered'] > 300000:  # 5 minutes
                # Claim le message pour le retraiter
                r.xclaim(
                    stream_name,
                    group_name,
                    consumer_name,
                    min_idle_time=300000,
                    message_ids=[entry['message_id']]
                )

# 3. Dead Letter Queue après N tentatives
def move_to_dlq(stream_name, group_name, max_retries=3):
    pending = r.xpending_range(stream_name, group_name, '-', '+', count=100)

    for entry in pending:
        if entry['times_delivered'] >= max_retries:
            # Récupérer le message complet
            msg = r.xrange(stream_name, entry['message_id'], entry['message_id'])

            # Copier vers DLQ
            r.xadd(f"{stream_name}:dlq", msg[0][1])

            # ACK l'original pour le retirer de pending
            r.xack(stream_name, group_name, entry['message_id'])
```

---

### 5. Observabilité et monitoring

#### Pub/Sub : Monitoring limité

```bash
# Informations disponibles
PUBSUB CHANNELS pattern*    # Liste des channels actifs
PUBSUB NUMSUB channel       # Nombre de subscribers par channel
PUBSUB NUMPAT              # Nombre de pattern subscriptions

# Exemple
PUBSUB NUMSUB notifications:users
# Retourne : 1) "notifications:users" 2) (integer) 5
```

**Limitations** :
- ❌ Pas d'historique des messages
- ❌ Pas de métriques de latence
- ❌ Impossible de tracer un message spécifique

#### Streams : Observabilité complète

```bash
# 1. Informations sur le stream
XINFO STREAM orders:stream
# Retourne : length, first-entry, last-entry, groups, etc.

# 2. État des consumer groups
XINFO GROUPS orders:stream
# Retourne : name, consumers, pending, last-delivered-id

# 3. Détail par consumer
XINFO CONSUMERS orders:stream payment-processor
# Retourne : name, pending, idle time pour chaque consumer

# 4. Messages en attente (pending)
XPENDING orders:stream payment-processor
# Retourne : count, min-id, max-id, consumers avec leurs counts

# 5. Détail des messages pending
XPENDING orders:stream payment-processor - + 10
# Retourne : [id, consumer, idle_ms, delivery_count] pour chaque message
```

**Exemple de monitoring complet** :

```python
def monitor_stream_health(stream_name, group_name):
    r = redis.Redis()

    # Métriques globales
    info = r.xinfo_stream(stream_name)
    stream_length = info['length']

    # Métriques par consumer group
    groups = r.xinfo_groups(stream_name)
    for group in groups:
        pending_count = group['pending']
        lag = group['lag']  # Messages non-consommés

        # Métriques par consumer
        consumers = r.xinfo_consumers(stream_name, group['name'])
        for consumer in consumers:
            idle_time = consumer['idle']
            pending = consumer['pending']

            # Alerter si consumer inactif
            if idle_time > 60000 and pending > 0:
                alert(f"Consumer {consumer['name']} idle avec {pending} messages")

    # Analyser les messages problématiques
    pending_details = r.xpending_range(stream_name, group_name, '-', '+', 100)
    for msg in pending_details:
        if msg['times_delivered'] > 5:
            alert(f"Message {msg['message_id']} failed {msg['times_delivered']} times")
```

---

## Tableau comparatif global

| Critère | Pub/Sub | Streams | Lists |
|---------|---------|---------|-------|
| **Persistance** | ❌ Aucune | ✅ Complète (RDB/AOF) | ✅ Complète |
| **Garantie de livraison** | ❌ Best effort | ✅ At-least-once | ✅ Exactly-once possible |
| **Order preservation** | ✅ Oui | ✅ Oui (strict) | ✅ Oui (FIFO) |
| **Multi-consumer** | ✅ Broadcast | ✅ Consumer groups | ⚠️ Manuel |
| **Load balancing** | ❌ Non natif | ✅ Automatique | ⚠️ Pattern à implémenter |
| **Rejouabilité** | ❌ Impossible | ✅ Complète | ⚠️ Une seule fois |
| **ACK/NACK** | ❌ Non | ✅ Oui | ⚠️ Implicite (LPOP) |
| **Retry automatique** | ❌ Non | ✅ Oui (pending) | ❌ Non |
| **Monitoring** | ⚠️ Limité | ✅ Complet | ⚠️ Basique |
| **Latence** | ✅ Ultra-faible (μs) | ⚠️ Faible (ms) | ⚠️ Faible (ms) |
| **Complexité** | ✅ Très simple | ⚠️ Moyenne | ✅ Simple |
| **Memory overhead** | ✅ Minimal | ⚠️ Modéré | ✅ Faible |
| **Message history** | ❌ Non | ✅ Illimité | ❌ Non (consommé = supprimé) |
| **Pattern matching** | ✅ Oui (PSUBSCRIBE) | ❌ Non | ❌ Non |
| **Blocking operations** | ✅ Oui | ✅ Oui (XREAD BLOCK) | ✅ Oui (BRPOP) |

---

## Cas d'usage détaillés

### Quand utiliser Pub/Sub ?

✅ **Idéal pour** :
```python
# 1. Invalidation de cache distribuée
redis.publish('cache:invalidate', 'user:123')

# 2. Live notifications WebSocket
redis.publish('notifications:user:456', json.dumps({
    'type': 'new_message',
    'from': 'user:789'
}))

# 3. Broadcast à tous les serveurs d'application
redis.publish('config:reload', 'settings.json')

# 4. Dashboard temps réel (metrics)
redis.publish('metrics:live', json.dumps({
    'cpu': 45.2,
    'memory': 78.5,
    'timestamp': time.time()
}))
```

❌ **À éviter pour** :
- Traitement de commandes/transactions
- Systèmes de paiement
- Logs d'audit
- Tout ce qui nécessite une garantie de livraison

---

### Quand utiliser Streams ?

✅ **Idéal pour** :
```python
# 1. Pipeline de traitement de commandes
redis.xadd('orders:stream', {
    'order_id': '12345',
    'user_id': '789',
    'amount': 99.99,
    'status': 'pending'
})

# 2. Event sourcing
redis.xadd('events:user:123', {
    'event': 'profile_updated',
    'field': 'email',
    'old_value': 'old@example.com',
    'new_value': 'new@example.com',
    'timestamp': time.time()
})

# 3. Logs applicatifs centralisés
redis.xadd('logs:app', {
    'level': 'error',
    'service': 'payment-service',
    'message': 'Payment failed',
    'trace_id': 'abc-123'
})

# 4. Data pipeline avec transformations
# Stage 1: Raw data
redis.xadd('pipeline:raw', {'data': raw_data})
# Stage 2: Processed data
redis.xadd('pipeline:processed', {'data': processed_data})
# Stage 3: Enriched data
redis.xadd('pipeline:enriched', {'data': enriched_data})
```

✅ **Parfait pour** :
- Systèmes nécessitant une garantie de traitement
- Architectures event-driven
- Audit trail et compliance
- Replay de scénarios pour debugging

---

### Quand utiliser Lists ?

✅ **Idéal pour** :
```python
# 1. Job queue simple
redis.lpush('jobs:email', json.dumps({
    'to': 'user@example.com',
    'subject': 'Welcome',
    'template': 'welcome.html'
}))

# Worker
job = redis.brpop('jobs:email', timeout=5)

# 2. Rate limiting avec window
redis.lpush(f'requests:{user_id}', timestamp)
redis.ltrim(f'requests:{user_id}', 0, 99)  # Garder 100 dernières

# 3. Recent items (feed)
redis.lpush('feed:user:123', post_id)
redis.ltrim('feed:user:123', 0, 49)  # Garder 50 posts
recent_posts = redis.lrange('feed:user:123', 0, 9)  # 10 plus récents
```

⚠️ **Limitations** :
- Pas de consumer groups natifs
- Consommation = suppression (pas de rejouabilité)
- Pas d'ACK explicites

---

## Scénarios de migration

### De Pub/Sub vers Streams

**Scénario** : Vous utilisez Pub/Sub mais perdez trop de messages.

```python
# AVANT (Pub/Sub)
import redis

r = redis.Redis()
pubsub = r.pubsub()
pubsub.subscribe('orders')

for message in pubsub.listen():
    try:
        process_order(message['data'])
    except Exception:
        # Message perdu si erreur !
        pass

# APRÈS (Streams)
import redis

r = redis.Redis()

# Setup
r.xgroup_create('orders:stream', 'order-processor', mkstream=True)

# Worker
while True:
    messages = r.xreadgroup(
        groupname='order-processor',
        consumername='worker-1',
        streams={'orders:stream': '>'},
        count=10,
        block=5000
    )

    for stream, entries in messages:
        for msg_id, data in entries:
            try:
                process_order(data)
                r.xack('orders:stream', 'order-processor', msg_id)
            except Exception as e:
                # Message reste en pending, sera retryé
                logger.error(f"Will retry: {e}")
```

**Bénéfices** :
- ✅ Zéro perte de message
- ✅ Retry automatique
- ✅ Monitoring complet
- ✅ Scalabilité horizontale simple

---

### De Lists vers Streams

**Scénario** : Vous avez besoin de consumer groups et de monitoring.

```python
# AVANT (Lists)
while True:
    job = redis.brpop('jobs', timeout=5)
    if job:
        process(job[1])
        # Problème : Si crash pendant process(), job perdu

# APRÈS (Streams)
r.xgroup_create('jobs:stream', 'workers', mkstream=True)

while True:
    messages = r.xreadgroup(
        groupname='workers',
        consumername='worker-1',
        streams={'jobs:stream': '>'},
        count=1,
        block=5000
    )

    if messages:
        msg_id = messages[0][1][0][0]
        data = messages[0][1][0][1]

        try:
            process(data)
            r.xack('jobs:stream', 'workers', msg_id)
        except Exception:
            # Message reste en pending pour retry
            pass
```

---

## Patterns hybrides

### Combinaison Pub/Sub + Streams

**Use case** : Notifications temps réel + audit trail

```python
import redis
import json

r = redis.Redis()

def send_notification(user_id, notification):
    data = json.dumps(notification)

    # 1. Pub/Sub pour livraison immédiate (si online)
    r.publish(f'notifications:user:{user_id}', data)

    # 2. Streams pour persistance (si offline)
    r.xadd(f'notifications:user:{user_id}:history', {
        'type': notification['type'],
        'message': notification['message'],
        'timestamp': notification['timestamp']
    })

    # 3. Trimmer pour éviter croissance infinie
    r.xtrim(f'notifications:user:{user_id}:history', maxlen=1000)

# Consumer online (Pub/Sub)
pubsub = r.pubsub()
pubsub.subscribe('notifications:user:123')

# Consumer offline recovery (Streams)
last_seen = r.get('user:123:last_notification_id') or '0-0'
history = r.xread({'notifications:user:123:history': last_seen}, count=100)
```

---

## Checklist de décision

Utilisez cette checklist pour choisir le bon mécanisme :

### Questions à se poser

1. **Puis-je perdre des messages ?**
   - ✅ Oui → Pub/Sub
   - ❌ Non → Streams ou Lists

2. **Ai-je besoin de rejouer des messages ?**
   - ✅ Oui → Streams uniquement
   - ❌ Non → Pub/Sub ou Lists

3. **Combien de consumers ?**
   - Un seul → Lists suffisent
   - Plusieurs (même message) → Pub/Sub
   - Plusieurs (charge distribuée) → Streams

4. **Latence critique < 1ms ?**
   - ✅ Oui → Pub/Sub
   - ❌ Non → Streams ou Lists acceptable

5. **Besoin d'ACK explicites ?**
   - ✅ Oui → Streams uniquement
   - ❌ Non → Pub/Sub ou Lists

6. **Besoin de monitoring détaillé ?**
   - ✅ Oui → Streams
   - ❌ Non → Pub/Sub ou Lists

7. **Ordre de traitement important ?**
   - ✅ Critique → Streams (garantie stricte)
   - ⚠️ Important → Lists (FIFO simple)
   - ❌ Peu important → Pub/Sub

---

## Performance et overhead

### Benchmarks comparatifs

```bash
# Pub/Sub
redis-benchmark -t publish -c 50 -n 100000
# ~100,000 ops/sec avec latence < 1ms

# Streams (sans consumer groups)
redis-benchmark -c 50 -n 100000 XADD mystream * field value
# ~80,000 ops/sec avec latence ~1-2ms

# Streams (avec consumer groups et ACK)
# ~40,000-60,000 ops/sec avec latence ~2-5ms

# Lists
redis-benchmark -t lpush,rpop -c 50 -n 100000
# ~90,000 ops/sec
```

### Consommation mémoire

```python
import redis
import sys

r = redis.Redis()

# Pub/Sub : Aucune persistance
# Memory overhead : ~0 bytes (juste connexions actives)

# Streams : Par message
message = {'field1': 'value1', 'field2': 'value2', 'field3': 'value3'}
r.xadd('test:stream', message)
# Memory : ~150-200 bytes par message + overhead structure

# Lists : Par élément
r.lpush('test:list', 'value')
# Memory : ~80-100 bytes par élément + overhead structure
```

---

## Conclusion et recommandations

### Règles d'or

1. **Par défaut, commencez simple** :
   - Notifications non-critiques → Pub/Sub
   - Job queues simples → Lists
   - Tout le reste → Streams

2. **Évoluez selon vos besoins** :
   - Pub/Sub perd des messages → Migrez vers Streams
   - Lists deviennent complexes → Migrez vers Streams
   - Latence critique < 1ms → Restez sur Pub/Sub

3. **Ne sur-complexifiez pas** :
   - Pas besoin de Streams si Lists + retry manuel suffisent
   - Pas besoin de Streams si les messages peuvent être perdus

4. **Pensez observabilité dès le départ** :
   - Si debugging important → Streams
   - Si "fire and forget" acceptable → Pub/Sub

### Pour aller plus loin

Les sections suivantes du module détaillent :
- **8.1** : Pub/Sub classique en profondeur
- **8.2** : Sharded Pub/Sub pour le scaling
- **8.3-8.5** : Redis Streams (concepts, consumer groups, patterns avancés)
- **8.6** : Comparaison avec Kafka et autres solutions

---

**Points clés à retenir** :
- 🚀 Pub/Sub = Vitesse maximale, zéro garantie
- 🔒 Streams = Garanties fortes, observabilité complète
- 📦 Lists = Simplicité, cas d'usage limités
- 🎯 Le choix dépend de vos contraintes métier, pas de préférences techniques


⏭️ [Pub/Sub classique : Le "Fire and Forget"](/08-communication-flux-donnees/01-pubsub-classique-fire-and-forget.md)
