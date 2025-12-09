🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.8 Files d'attente et Job Queues

## Introduction

Les **files d'attente** (queues) sont essentielles pour découpler les composants d'une application, gérer des tâches asynchrones, et construire des systèmes distribués robustes. Redis, avec ses structures de données natives et ses opérations atomiques, est parfaitement adapté pour implémenter des systèmes de queues performants.

Cette section explore différents patterns de queues : simple queue, reliable queue, priority queue, delayed jobs, avec des implémentations production-ready.

## Pourquoi utiliser des Job Queues ?

### Cas d'usage typiques

**1. Traitement asynchrone**

```text
Scénario : Upload et traitement d'image

Sans queue (synchrone):
User uploads image (5MB)
    ↓
Server processes:
  - Resize image (2s)
  - Generate thumbnails (3s)
  - Apply filters (2s)
  - Upload to CDN (3s)
    ↓
Response after 10 seconds ❌
Poor user experience

Avec queue (asynchrone):
User uploads image
    ↓
Add job to queue (50ms)
    ↓
Response immediately ✅
Good user experience
    ↓
Workers process in background
```

**2. Peak Shaving (Lissage des pics)**

```text
Scénario : Black Friday - 10,000 orders/minute

Sans queue:
Orders → Direct processing
         ↓
         Database overload ❌
         System crash

Avec queue:
Orders → Queue (buffer)
         ↓
         Workers process at sustainable rate
         (100 orders/minute per worker)
         ↓
         System stable ✅
```

**3. Task Distribution**

```text
Scénario : Video encoding avec 10 workers

                  Queue
                    │
        ┌───────────┼───────────┐
        │           │           │
    Worker 1    Worker 2    Worker 3
    (Idle)      (Busy)      (Idle)
        │                       │
        └───── Get next job ────┘
               from queue
```

**4. Retry Logic**

```text
Scénario : Email sending avec failures

Direct send → SMTP timeout → Lost ❌

Queue send → Failed → Retry (3 attempts) → Success ✅
```

---

## Pattern 1: Simple Queue (FIFO)

### Principe avec LIST

Redis LIST est parfait pour implémenter une queue FIFO (First In, First Out).

```text
┌─────────────────────────────────────────────────────────────┐
│                    SIMPLE QUEUE (LIST)                      │
└─────────────────────────────────────────────────────────────┘

Producer                  Redis LIST                  Consumer
   │                    queue:jobs                        │
   │                                                      │
   │  LPUSH job1                                          │
   ├──────────────────> [job1]                            │
   │                                                      │
   │  LPUSH job2                                          │
   ├──────────────────> [job2, job1]                      │
   │                                                      │
   │  LPUSH job3                                          │
   ├──────────────────> [job3, job2, job1]                │
   │                                                      │
   │                    [job3, job2, job1]                │
   │                                       BRPOP ─────────┤
   │                    [job3, job2] <────────────────────┤
   │                                     (got job1)       │
   │                                                      │
   │                    [job3, job2]                      │
   │                                       BRPOP ─────────┤
   │                    [job3] <──────────────────────────┤
   │                                     (got job2)       │

Operations:
- LPUSH queue:jobs job_data  → Add job (Producer)
- BRPOP queue:jobs timeout   → Get job (Consumer, blocking)
- LLEN queue:jobs            → Queue size
```

### Implémentation Python

```python
import redis
import json
import uuid
import time
from datetime import datetime
from typing import Dict, Any, Optional

class SimpleQueue:
    """
    Simple FIFO queue avec Redis LIST
    """

    def __init__(self, redis_client, queue_name='jobs'):
        self.redis = redis_client
        self.queue_name = f"queue:{queue_name}"

    def enqueue(self, job_data: Dict[str, Any]) -> str:
        """
        Ajoute un job dans la queue

        Args:
            job_data: Données du job

        Returns:
            str: Job ID
        """
        job_id = str(uuid.uuid4())

        job = {
            'id': job_id,
            'data': job_data,
            'enqueued_at': time.time(),
            'status': 'pending'
        }

        # Ajouter à la queue (LPUSH = add to left/head)
        self.redis.lpush(self.queue_name, json.dumps(job))

        print(f"✓ Job enqueued: {job_id}")
        print(f"  Queue size: {self.size()}")

        return job_id

    def dequeue(self, timeout=0) -> Optional[Dict[str, Any]]:
        """
        Récupère un job de la queue

        Args:
            timeout: Temps d'attente en secondes (0 = non-bloquant)

        Returns:
            dict or None: Job data
        """
        if timeout > 0:
            # BRPOP = Blocking Right POP (from right/tail)
            result = self.redis.brpop(self.queue_name, timeout=timeout)
        else:
            # RPOP = Non-blocking
            result = self.redis.rpop(self.queue_name)
            if result:
                result = (self.queue_name, result)
            else:
                return None

        if not result:
            return None

        _, job_json = result
        job = json.loads(job_json)

        print(f"✓ Job dequeued: {job['id']}")

        return job

    def size(self) -> int:
        """Retourne la taille de la queue"""
        return self.redis.llen(self.queue_name)

    def clear(self):
        """Vide la queue"""
        deleted = self.redis.delete(self.queue_name)
        print(f"✓ Queue cleared ({deleted} keys deleted)")


# ============================================================
# PRODUCER (Producteur de jobs)
# ============================================================

class JobProducer:
    """Producteur de jobs"""

    def __init__(self, queue):
        self.queue = queue

    def submit_email_job(self, to, subject, body):
        """Soumettre un job d'envoi d'email"""
        job_data = {
            'type': 'send_email',
            'to': to,
            'subject': subject,
            'body': body
        }

        return self.queue.enqueue(job_data)

    def submit_image_processing_job(self, image_url, operations):
        """Soumettre un job de traitement d'image"""
        job_data = {
            'type': 'process_image',
            'image_url': image_url,
            'operations': operations
        }

        return self.queue.enqueue(job_data)


# ============================================================
# CONSUMER / WORKER
# ============================================================

class JobWorker:
    """Worker qui traite les jobs"""

    def __init__(self, queue, worker_id='worker-1'):
        self.queue = queue
        self.worker_id = worker_id
        self.running = False

    def start(self):
        """Démarre le worker"""
        self.running = True
        print(f"🚀 Worker {self.worker_id} started")

        while self.running:
            # Attendre un job (bloquant, timeout 5s)
            job = self.queue.dequeue(timeout=5)

            if job:
                self.process_job(job)
            else:
                print(f"  {self.worker_id}: No jobs, waiting...")

    def stop(self):
        """Arrête le worker"""
        self.running = False
        print(f"🛑 Worker {self.worker_id} stopped")

    def process_job(self, job):
        """Traite un job"""
        job_type = job['data'].get('type')

        print(f"\n{self.worker_id} processing job {job['id']} (type: {job_type})")

        try:
            if job_type == 'send_email':
                self._send_email(job['data'])
            elif job_type == 'process_image':
                self._process_image(job['data'])
            else:
                print(f"  Unknown job type: {job_type}")

            print(f"  ✓ Job {job['id']} completed")

        except Exception as e:
            print(f"  ✗ Job {job['id']} failed: {e}")

    def _send_email(self, data):
        """Simuler l'envoi d'un email"""
        print(f"  📧 Sending email to {data['to']}")
        time.sleep(1)  # Simuler le traitement
        print(f"  ✓ Email sent")

    def _process_image(self, data):
        """Simuler le traitement d'image"""
        print(f"  🖼️  Processing image: {data['image_url']}")
        time.sleep(2)  # Simuler le traitement
        print(f"  ✓ Image processed")


# ============================================================
# EXEMPLE D'UTILISATION
# ============================================================

def example_simple_queue():
    """Exemple complet avec producer et consumer"""
    redis_client = redis.Redis(decode_responses=True)

    # Créer la queue
    queue = SimpleQueue(redis_client, queue_name='email_jobs')

    # Producer : Ajouter des jobs
    producer = JobProducer(queue)

    print("=" * 60)
    print("ADDING JOBS TO QUEUE")
    print("=" * 60 + "\n")

    producer.submit_email_job(
        to='user@example.com',
        subject='Welcome!',
        body='Thanks for signing up'
    )

    producer.submit_email_job(
        to='admin@example.com',
        subject='New User',
        body='A new user signed up'
    )

    producer.submit_image_processing_job(
        image_url='https://example.com/image.jpg',
        operations=['resize', 'thumbnail']
    )

    print(f"\nQueue size: {queue.size()}")

    # Worker : Traiter les jobs
    print("\n" + "=" * 60)
    print("WORKER PROCESSING JOBS")
    print("=" * 60 + "\n")

    worker = JobWorker(queue)

    # Traiter quelques jobs manuellement
    for _ in range(3):
        job = queue.dequeue()
        if job:
            worker.process_job(job)
        else:
            print("No more jobs")
            break


# ============================================================
# MULTI-WORKER AVEC THREADING
# ============================================================

import threading

def run_worker(queue, worker_id, duration=10):
    """Exécute un worker pour une durée donnée"""
    worker = JobWorker(queue, worker_id=worker_id)

    def worker_thread():
        start_time = time.time()
        worker.running = True

        while worker.running and (time.time() - start_time) < duration:
            job = queue.dequeue(timeout=1)
            if job:
                worker.process_job(job)

    thread = threading.Thread(target=worker_thread)
    thread.start()
    return thread


def example_multi_worker():
    """Exemple avec plusieurs workers concurrents"""
    redis_client = redis.Redis(decode_responses=True)
    queue = SimpleQueue(redis_client, queue_name='tasks')
    producer = JobProducer(queue)

    print("=" * 60)
    print("MULTI-WORKER EXAMPLE")
    print("=" * 60 + "\n")

    # Ajouter 10 jobs
    print("Adding 10 jobs...")
    for i in range(10):
        producer.submit_email_job(
            to=f'user{i}@example.com',
            subject=f'Email {i}',
            body=f'Body {i}'
        )

    print(f"\nQueue size: {queue.size()}")

    # Démarrer 3 workers
    print("\nStarting 3 workers...\n")
    threads = []
    for i in range(3):
        thread = run_worker(queue, f'worker-{i}', duration=15)
        threads.append(thread)

    # Attendre les workers
    for thread in threads:
        thread.join()

    print(f"\nFinal queue size: {queue.size()}")
```

---

## Pattern 2: Reliable Queue (BRPOPLPUSH)

### Problème de la Simple Queue

```text
PROBLÈME : Job perdu si le worker crash

Worker                    Queue                   Processing
  │                    [job3, job2, job1]              │
  │                                                    │
  │  BRPOP                                             │
  ├──────────────────> [job3, job2]                    │
  │                                                    │
  │  Got job1                                          │
  │                                                    │
  │  Start processing ───────────────────────────────> │
  │                                                    │
  💥 CRASH!                                            │
  │                                                    │
  ❌ job1 is LOST                                      │
  (removed from queue but not processed)

Solution: BRPOPLPUSH (atomic move)
```

### BRPOPLPUSH : Atomic Move

```text
┌─────────────────────────────────────────────────────────────┐
│              RELIABLE QUEUE (BRPOPLPUSH)                    │
└─────────────────────────────────────────────────────────────┘

Operation: BRPOPLPUSH source destination timeout

Worker              Queue                Processing Queue
  │              queue:jobs              queue:processing
  │           [job3, job2, job1]               []
  │                                             │
  │  BRPOPLPUSH queue:jobs queue:processing     │
  ├─────────────────────────────────────────────────────>
  │                                             │
  │           [job3, job2]                 [job1]
  │                                             │
  │  Processing job1...                         │
  │                                             │
  │  ✓ Success                                  │
  │                                             │
  │  LREM queue:processing job1 ────────────────┤
  │                                        []   │

If worker crashes:
  💥 CRASH during processing

  Recovery worker checks queue:processing
  ├─ Finds unfinished job1
  └─ Re-enqueue to queue:jobs ✅
```

### Implémentation Python

```python
class ReliableQueue:
    """
    Reliable queue avec BRPOPLPUSH
    """

    def __init__(self, redis_client, queue_name='jobs'):
        self.redis = redis_client
        self.queue_name = f"queue:{queue_name}"
        self.processing_name = f"queue:{queue_name}:processing"
        self.worker_id = str(uuid.uuid4())[:8]

    def enqueue(self, job_data: Dict[str, Any]) -> str:
        """Ajoute un job"""
        job_id = str(uuid.uuid4())

        job = {
            'id': job_id,
            'data': job_data,
            'enqueued_at': time.time(),
            'attempts': 0,
            'max_attempts': 3
        }

        self.redis.lpush(self.queue_name, json.dumps(job))

        print(f"✓ Job enqueued: {job_id}")
        return job_id

    def dequeue(self, timeout=0) -> Optional[Dict[str, Any]]:
        """
        Récupère un job de manière fiable
        Utilise BRPOPLPUSH pour déplacer atomiquement
        """
        if timeout > 0:
            # BRPOPLPUSH = atomic move from queue to processing
            result = self.redis.brpoplpush(
                self.queue_name,
                self.processing_name,
                timeout=timeout
            )
        else:
            result = self.redis.rpoplpush(
                self.queue_name,
                self.processing_name
            )

        if not result:
            return None

        job = json.loads(result)

        # Ajouter metadata du worker
        job['worker_id'] = self.worker_id
        job['started_at'] = time.time()

        print(f"✓ Job dequeued: {job['id']} by {self.worker_id}")

        return job

    def ack(self, job: Dict[str, Any]):
        """
        Marque le job comme complété (acknowledge)
        Supprime de la processing queue
        """
        job_json = json.dumps(job)

        # Supprimer de la processing queue
        removed = self.redis.lrem(self.processing_name, 1, job_json)

        if removed:
            print(f"✓ Job acknowledged: {job['id']}")
        else:
            print(f"⚠️  Job not found in processing queue: {job['id']}")

        return removed > 0

    def nack(self, job: Dict[str, Any], requeue=True):
        """
        Marque le job comme échoué (negative acknowledge)

        Args:
            requeue: Si True, remettre dans la queue principale
        """
        job_json_old = json.dumps(job)

        # Supprimer de processing
        self.redis.lrem(self.processing_name, 1, job_json_old)

        if requeue and job.get('attempts', 0) < job.get('max_attempts', 3):
            # Incrémenter attempts
            job['attempts'] = job.get('attempts', 0) + 1
            job['last_error_at'] = time.time()

            # Remettre dans la queue
            self.redis.lpush(self.queue_name, json.dumps(job))

            print(f"⚠️  Job requeued: {job['id']} (attempt {job['attempts']})")
        else:
            print(f"✗ Job failed permanently: {job['id']}")
            # Optionnel : ajouter à une dead letter queue
            self._move_to_dlq(job)

    def _move_to_dlq(self, job):
        """Déplace vers Dead Letter Queue"""
        dlq_name = f"{self.queue_name}:dlq"
        self.redis.lpush(dlq_name, json.dumps(job))
        print(f"  ➜ Moved to DLQ")

    def recover_stale_jobs(self, timeout_seconds=300):
        """
        Récupère les jobs bloqués dans processing
        (worker crashed avant de les terminer)
        """
        print("\n🔍 Checking for stale jobs...")

        # Récupérer tous les jobs en processing
        processing_jobs = self.redis.lrange(self.processing_name, 0, -1)

        recovered = 0
        now = time.time()

        for job_json in processing_jobs:
            job = json.loads(job_json)
            started_at = job.get('started_at', 0)

            # Si le job tourne depuis trop longtemps
            if (now - started_at) > timeout_seconds:
                print(f"  Found stale job: {job['id']}")

                # Supprimer de processing
                self.redis.lrem(self.processing_name, 1, job_json)

                # Remettre dans la queue
                job['attempts'] = job.get('attempts', 0) + 1
                if job['attempts'] < job.get('max_attempts', 3):
                    self.redis.lpush(self.queue_name, json.dumps(job))
                    recovered += 1
                    print(f"    ↻ Recovered and requeued")
                else:
                    self._move_to_dlq(job)
                    print(f"    ✗ Max attempts reached, moved to DLQ")

        print(f"✓ Recovered {recovered} stale jobs\n")
        return recovered


# ============================================================
# RELIABLE WORKER
# ============================================================

class ReliableWorker:
    """Worker avec gestion d'erreurs et retry"""

    def __init__(self, queue, worker_id='worker-1'):
        self.queue = queue
        self.worker_id = worker_id
        self.running = False

    def start(self):
        """Démarre le worker"""
        self.running = True
        print(f"🚀 Reliable Worker {self.worker_id} started")

        # Récupérer les jobs bloqués au démarrage
        self.queue.recover_stale_jobs()

        while self.running:
            job = self.queue.dequeue(timeout=5)

            if job:
                success = self.process_job(job)

                if success:
                    # Acknowledger le job
                    self.queue.ack(job)
                else:
                    # Negative acknowledge (requeue si attempts < max)
                    self.queue.nack(job, requeue=True)

    def process_job(self, job) -> bool:
        """
        Traite un job

        Returns:
            bool: True si succès, False si échec
        """
        job_id = job['id']
        job_type = job['data'].get('type')

        print(f"\n{self.worker_id} processing {job_id} (attempt {job.get('attempts', 0) + 1})")

        try:
            # Simuler traitement
            if job_type == 'send_email':
                self._send_email(job['data'])
            elif job_type == 'fail_test':
                # Simuler un échec
                raise Exception("Simulated failure")
            else:
                print(f"  Processing {job_type}...")
                time.sleep(1)

            print(f"  ✓ Job {job_id} completed")
            return True

        except Exception as e:
            print(f"  ✗ Job {job_id} failed: {e}")
            return False

    def _send_email(self, data):
        """Simuler l'envoi d'email"""
        print(f"  📧 Sending email to {data['to']}")
        time.sleep(1)


# ============================================================
# EXEMPLE AVEC RETRY
# ============================================================

def example_reliable_queue():
    """Exemple avec reliable queue et retry"""
    redis_client = redis.Redis(decode_responses=True)
    queue = ReliableQueue(redis_client, queue_name='reliable_jobs')

    print("=" * 60)
    print("RELIABLE QUEUE WITH RETRY")
    print("=" * 60 + "\n")

    # Ajouter des jobs (dont un qui va fail)
    queue.enqueue({'type': 'send_email', 'to': 'user@example.com'})
    queue.enqueue({'type': 'fail_test', 'reason': 'Test retry'})
    queue.enqueue({'type': 'send_email', 'to': 'admin@example.com'})

    print(f"\nQueue size: {queue.size()}")

    # Worker traite les jobs
    worker = ReliableWorker(queue)

    # Traiter manuellement pour voir les retries
    for i in range(6):  # Le job failed sera retry 3 fois
        job = queue.dequeue(timeout=1)
        if job:
            success = worker.process_job(job)
            if success:
                queue.ack(job)
            else:
                queue.nack(job, requeue=True)
        else:
            print("No more jobs")
            break
        time.sleep(0.5)

    # Vérifier la DLQ
    dlq_size = redis_client.llen(f"{queue.queue_name}:dlq")
    print(f"\nDead Letter Queue size: {dlq_size}")
```

---

## Pattern 3: Priority Queue

### Principe avec Sorted Set

```text
┌─────────────────────────────────────────────────────────────┐
│                    PRIORITY QUEUE (ZSET)                    │
└─────────────────────────────────────────────────────────────┘

Redis ZSET (Sorted Set):
- Members: job_id
- Score: priority (lower = higher priority)

Example:
ZADD queue:priority 1 job_urgent     (priority 1 = highest)
ZADD queue:priority 5 job_normal
ZADD queue:priority 10 job_low

queue:priority (sorted by score):
┌─────────────────────────────────┐
│ Score  │  Member                │
├─────────────────────────────────┤
│   1    │  job_urgent     ← Pop  │
│   5    │  job_normal            │
│  10    │  job_low               │
└─────────────────────────────────┘

Operations:
ZADD queue:priority priority job_id  → Add job
ZPOPMIN queue:priority              → Get highest priority
ZRANGE queue:priority 0 -1          → List all jobs
```

### Implémentation Python

```python
class PriorityQueue:
    """
    Priority queue avec Redis Sorted Set
    """

    PRIORITY_HIGH = 1
    PRIORITY_NORMAL = 5
    PRIORITY_LOW = 10

    def __init__(self, redis_client, queue_name='priority_jobs'):
        self.redis = redis_client
        self.queue_name = f"queue:{queue_name}"
        self.data_prefix = f"{self.queue_name}:data:"

    def enqueue(self, job_data: Dict[str, Any], priority: int = PRIORITY_NORMAL) -> str:
        """
        Ajoute un job avec priorité

        Args:
            job_data: Données du job
            priority: Priorité (1 = haute, 10 = basse)
        """
        job_id = str(uuid.uuid4())

        job = {
            'id': job_id,
            'data': job_data,
            'priority': priority,
            'enqueued_at': time.time()
        }

        # Stocker les données du job
        job_key = f"{self.data_prefix}{job_id}"
        self.redis.set(job_key, json.dumps(job))
        self.redis.expire(job_key, 3600)  # 1 hour TTL

        # Ajouter à la sorted set avec priorité comme score
        self.redis.zadd(self.queue_name, {job_id: priority})

        priority_name = {
            self.PRIORITY_HIGH: 'HIGH',
            self.PRIORITY_NORMAL: 'NORMAL',
            self.PRIORITY_LOW: 'LOW'
        }.get(priority, 'CUSTOM')

        print(f"✓ Job enqueued: {job_id} (priority: {priority_name})")
        print(f"  Queue size: {self.size()}")

        return job_id

    def dequeue(self, timeout=0) -> Optional[Dict[str, Any]]:
        """
        Récupère le job avec la plus haute priorité
        """
        if timeout > 0:
            # BZPOPMIN = Blocking ZPOPMIN
            result = self.redis.bzpopmin(self.queue_name, timeout=timeout)
        else:
            # ZPOPMIN = Pop minimum score (highest priority)
            result = self.redis.zpopmin(self.queue_name, count=1)
            if result:
                result = (self.queue_name, result[0][0], result[0][1])
            else:
                return None

        if not result:
            return None

        _, job_id, priority = result

        # Récupérer les données du job
        job_key = f"{self.data_prefix}{job_id}"
        job_json = self.redis.get(job_key)

        if not job_json:
            print(f"⚠️  Job data not found: {job_id}")
            return None

        job = json.loads(job_json)

        # Supprimer les données (job pris en charge)
        self.redis.delete(job_key)

        print(f"✓ Job dequeued: {job_id} (priority: {priority})")

        return job

    def size(self) -> int:
        """Retourne la taille de la queue"""
        return self.redis.zcard(self.queue_name)

    def peek(self, count=10):
        """
        Affiche les N prochains jobs sans les retirer
        """
        jobs = self.redis.zrange(
            self.queue_name,
            0,
            count - 1,
            withscores=True
        )

        print(f"\nNext {len(jobs)} jobs:")
        for job_id, priority in jobs:
            print(f"  - {job_id} (priority: {priority})")


# ============================================================
# EXEMPLE D'UTILISATION
# ============================================================

def example_priority_queue():
    """Exemple avec priority queue"""
    redis_client = redis.Redis(decode_responses=True)
    queue = PriorityQueue(redis_client)

    print("=" * 60)
    print("PRIORITY QUEUE EXAMPLE")
    print("=" * 60 + "\n")

    # Ajouter des jobs avec différentes priorités
    queue.enqueue(
        {'type': 'send_email', 'to': 'user1@example.com'},
        priority=PriorityQueue.PRIORITY_LOW
    )

    queue.enqueue(
        {'type': 'send_email', 'to': 'urgent@example.com'},
        priority=PriorityQueue.PRIORITY_HIGH
    )

    queue.enqueue(
        {'type': 'send_email', 'to': 'user2@example.com'},
        priority=PriorityQueue.PRIORITY_NORMAL
    )

    queue.enqueue(
        {'type': 'send_email', 'to': 'critical@example.com'},
        priority=PriorityQueue.PRIORITY_HIGH
    )

    # Voir l'ordre
    queue.peek(count=10)

    # Traiter dans l'ordre de priorité
    print("\nProcessing jobs in priority order:")
    while queue.size() > 0:
        job = queue.dequeue()
        if job:
            print(f"  Processing: {job['data']['to']}")
            time.sleep(0.5)
```

---

## Pattern 4: Delayed/Scheduled Jobs

### Principe avec Sorted Set + Timestamp

```text
┌─────────────────────────────────────────────────────────────┐
│               DELAYED JOBS (ZSET + Timestamp)                │
└─────────────────────────────────────────────────────────────┘

Concept: Score = Unix timestamp when job should execute

Example:
Current time: 1670000000

ZADD queue:delayed 1670000060 job1  (execute in 60s)
ZADD queue:delayed 1670000120 job2  (execute in 120s)
ZADD queue:delayed 1670000030 job3  (execute in 30s)

queue:delayed (sorted by timestamp):
┌─────────────────────────────────────┐
│ Timestamp  │  Job                   │
├─────────────────────────────────────┤
│ 1670000030 │  job3 ← Ready now!     │
│ 1670000060 │  job1                  │
│ 1670000120 │  job2                  │
└─────────────────────────────────────┘

Scheduler checks periodically:
ZRANGEBYSCORE queue:delayed -inf now
  → Get jobs ready to execute
ZREM queue:delayed job3
  → Remove from delayed queue
LPUSH queue:ready job3
  → Add to ready queue
```

### Implémentation Python

```python
class DelayedQueue:
    """
    Queue avec support de delayed/scheduled jobs
    """

    def __init__(self, redis_client, queue_name='delayed_jobs'):
        self.redis = redis_client
        self.delayed_queue = f"queue:{queue_name}:delayed"
        self.ready_queue = f"queue:{queue_name}:ready"
        self.data_prefix = f"queue:{queue_name}:data:"

    def enqueue(self, job_data: Dict[str, Any], delay_seconds: int = 0) -> str:
        """
        Ajoute un job avec délai

        Args:
            job_data: Données du job
            delay_seconds: Délai avant exécution (0 = immédiat)
        """
        job_id = str(uuid.uuid4())
        execute_at = time.time() + delay_seconds

        job = {
            'id': job_id,
            'data': job_data,
            'enqueued_at': time.time(),
            'execute_at': execute_at,
            'delay': delay_seconds
        }

        # Stocker les données
        job_key = f"{self.data_prefix}{job_id}"
        self.redis.set(job_key, json.dumps(job))
        self.redis.expire(job_key, delay_seconds + 3600)

        if delay_seconds > 0:
            # Ajouter à la delayed queue
            self.redis.zadd(self.delayed_queue, {job_id: execute_at})
            print(f"✓ Job scheduled: {job_id} (in {delay_seconds}s)")
        else:
            # Ajouter directement à la ready queue
            self.redis.lpush(self.ready_queue, job_id)
            print(f"✓ Job enqueued: {job_id} (immediate)")

        return job_id

    def dequeue(self, timeout=0) -> Optional[Dict[str, Any]]:
        """Récupère un job prêt"""
        if timeout > 0:
            result = self.redis.brpop(self.ready_queue, timeout=timeout)
        else:
            result = self.redis.rpop(self.ready_queue)
            if result:
                result = (self.ready_queue, result)

        if not result:
            return None

        _, job_id = result

        # Récupérer les données
        job_key = f"{self.data_prefix}{job_id}"
        job_json = self.redis.get(job_key)

        if not job_json:
            return None

        job = json.loads(job_json)
        self.redis.delete(job_key)

        return job

    def process_delayed_jobs(self) -> int:
        """
        Déplace les jobs prêts de delayed → ready
        À appeler périodiquement par un scheduler

        Returns:
            int: Nombre de jobs déplacés
        """
        now = time.time()

        # Récupérer tous les jobs prêts (score <= now)
        ready_job_ids = self.redis.zrangebyscore(
            self.delayed_queue,
            '-inf',
            now
        )

        if not ready_job_ids:
            return 0

        moved = 0
        for job_id in ready_job_ids:
            # Supprimer de delayed queue
            removed = self.redis.zrem(self.delayed_queue, job_id)

            if removed:
                # Ajouter à ready queue
                self.redis.lpush(self.ready_queue, job_id)
                moved += 1
                print(f"  ⏰ Job ready: {job_id}")

        if moved > 0:
            print(f"✓ Moved {moved} delayed jobs to ready queue")

        return moved

    def start_scheduler(self, interval=1):
        """
        Démarre le scheduler qui déplace les jobs delayed → ready

        Args:
            interval: Intervalle de vérification (secondes)
        """
        import threading

        def scheduler_loop():
            print(f"🕐 Scheduler started (interval: {interval}s)")
            while True:
                try:
                    self.process_delayed_jobs()
                except Exception as e:
                    print(f"⚠️  Scheduler error: {e}")
                time.sleep(interval)

        thread = threading.Thread(target=scheduler_loop, daemon=True)
        thread.start()
        return thread


# ============================================================
# EXEMPLE D'UTILISATION
# ============================================================

def example_delayed_queue():
    """Exemple avec delayed jobs"""
    redis_client = redis.Redis(decode_responses=True)
    queue = DelayedQueue(redis_client, queue_name='scheduled')

    print("=" * 60)
    print("DELAYED QUEUE EXAMPLE")
    print("=" * 60 + "\n")

    # Ajouter des jobs avec différents délais
    queue.enqueue(
        {'type': 'send_reminder', 'to': 'user@example.com'},
        delay_seconds=5
    )

    queue.enqueue(
        {'type': 'send_email', 'to': 'immediate@example.com'},
        delay_seconds=0  # Immédiat
    )

    queue.enqueue(
        {'type': 'cleanup_temp_files'},
        delay_seconds=10
    )

    # Démarrer le scheduler
    scheduler_thread = queue.start_scheduler(interval=1)

    # Worker traite les jobs prêts
    print("\nWorker waiting for jobs...\n")

    for i in range(15):
        job = queue.dequeue(timeout=1)
        if job:
            print(f"✓ Processing: {job['data']['type']}")
        else:
            print("  No jobs ready, waiting...")
        time.sleep(1)


# ============================================================
# CRON-LIKE SCHEDULING
# ============================================================

class CronQueue(DelayedQueue):
    """
    Queue avec support de scheduling type cron
    """

    def schedule_daily(self, job_data: Dict[str, Any], hour: int, minute: int = 0) -> str:
        """
        Schedule un job quotidien à une heure donnée

        Args:
            hour: Heure (0-23)
            minute: Minute (0-59)
        """
        from datetime import datetime, timedelta

        now = datetime.now()
        scheduled_time = now.replace(hour=hour, minute=minute, second=0, microsecond=0)

        # Si l'heure est déjà passée aujourd'hui, programmer pour demain
        if scheduled_time <= now:
            scheduled_time += timedelta(days=1)

        delay_seconds = int((scheduled_time - now).total_seconds())

        print(f"Scheduling daily job for {scheduled_time.strftime('%H:%M')}")
        print(f"  First execution in {delay_seconds}s")

        return self.enqueue(job_data, delay_seconds=delay_seconds)

    def schedule_weekly(self, job_data: Dict[str, Any], day_of_week: int,
                       hour: int = 9, minute: int = 0) -> str:
        """
        Schedule un job hebdomadaire

        Args:
            day_of_week: Jour (0=Lundi, 6=Dimanche)
            hour: Heure
            minute: Minute
        """
        from datetime import datetime, timedelta

        now = datetime.now()
        days_ahead = day_of_week - now.weekday()

        if days_ahead <= 0:
            days_ahead += 7

        scheduled_time = now + timedelta(days=days_ahead)
        scheduled_time = scheduled_time.replace(
            hour=hour,
            minute=minute,
            second=0,
            microsecond=0
        )

        delay_seconds = int((scheduled_time - now).total_seconds())

        days = ['Monday', 'Tuesday', 'Wednesday', 'Thursday',
                'Friday', 'Saturday', 'Sunday']

        print(f"Scheduling weekly job for {days[day_of_week]} at {hour:02d}:{minute:02d}")
        print(f"  First execution in {delay_seconds}s ({delay_seconds/3600:.1f} hours)")

        return self.enqueue(job_data, delay_seconds=delay_seconds)
```

---

## Implémentation Node.js avec Bull/BullMQ

### Bull Queue (Production-Ready)

```javascript
const Queue = require('bull');
const Redis = require('ioredis');

// ============================================================
// BULL QUEUE SETUP
// ============================================================

// Créer une connexion Redis
const redis = new Redis({
    host: 'localhost',
    port: 6379,
    maxRetriesPerRequest: null,
    enableReadyCheck: false
});

// Créer une queue
const emailQueue = new Queue('email-jobs', {
    redis: {
        host: 'localhost',
        port: 6379
    },
    defaultJobOptions: {
        attempts: 3,              // Retry 3 fois
        backoff: {
            type: 'exponential',  // Backoff exponentiel
            delay: 2000           // Départ à 2s
        },
        removeOnComplete: true,   // Nettoyer les jobs complétés
        removeOnFail: false       // Garder les jobs échoués
    }
});

// ============================================================
// PRODUCER
// ============================================================

async function addJobs() {
    console.log('='.repeat(60));
    console.log('ADDING JOBS TO BULL QUEUE');
    console.log('='.repeat(60) + '\n');

    // Job immédiat
    await emailQueue.add('send-email', {
        to: 'user@example.com',
        subject: 'Welcome!',
        body: 'Thanks for signing up'
    });

    // Job avec priorité
    await emailQueue.add('send-email', {
        to: 'urgent@example.com',
        subject: 'Urgent!',
        body: 'Important message'
    }, {
        priority: 1  // Plus haute priorité
    });

    // Job delayed (5 secondes)
    await emailQueue.add('send-email', {
        to: 'delayed@example.com',
        subject: 'Delayed',
        body: 'This arrives in 5 seconds'
    }, {
        delay: 5000  // 5 secondes
    });

    // Job répété (cron-like)
    await emailQueue.add('daily-report', {
        type: 'report',
        recipients: ['admin@example.com']
    }, {
        repeat: {
            cron: '0 9 * * *',  // Tous les jours à 9h
            tz: 'Europe/Paris'
        }
    });

    console.log('✓ Jobs added\n');
}

// ============================================================
// CONSUMER / WORKER
// ============================================================

// Process email jobs
emailQueue.process('send-email', 5, async (job) => {
    // 5 = concurrency (5 jobs en parallèle)

    console.log(`\nProcessing job ${job.id}`);
    console.log(`  Type: ${job.name}`);
    console.log(`  Data:`, job.data);
    console.log(`  Attempts: ${job.attemptsMade + 1}/${job.opts.attempts}`);

    // Simuler l'envoi d'email
    await sendEmail(job.data);

    // Mettre à jour le progress
    await job.progress(50);

    // Simuler plus de traitement
    await new Promise(resolve => setTimeout(resolve, 1000));

    await job.progress(100);

    console.log(`  ✓ Job ${job.id} completed`);

    // Retourner un résultat
    return { sent: true, timestamp: Date.now() };
});

// Process daily reports
emailQueue.process('daily-report', async (job) => {
    console.log(`\n📊 Generating daily report...`);
    console.log(`  Recipients:`, job.data.recipients);

    // Simuler génération de rapport
    await new Promise(resolve => setTimeout(resolve, 2000));

    console.log(`  ✓ Report sent`);

    return { reportGenerated: true };
});

async function sendEmail(data) {
    console.log(`  📧 Sending email to ${data.to}`);
    await new Promise(resolve => setTimeout(resolve, 500));

    // Simuler un échec aléatoire (10% de chance)
    if (Math.random() < 0.1) {
        throw new Error('SMTP timeout');
    }

    console.log(`  ✓ Email sent`);
}

// ============================================================
// EVENT LISTENERS
// ============================================================

emailQueue.on('completed', (job, result) => {
    console.log(`\n✅ Job ${job.id} completed`);
    console.log(`   Result:`, result);
});

emailQueue.on('failed', (job, err) => {
    console.log(`\n❌ Job ${job.id} failed`);
    console.log(`   Error: ${err.message}`);
    console.log(`   Attempts: ${job.attemptsMade}/${job.opts.attempts}`);

    if (job.attemptsMade >= job.opts.attempts) {
        console.log(`   ⚠️  Max attempts reached, moving to failed jobs`);
    }
});

emailQueue.on('progress', (job, progress) => {
    console.log(`📊 Job ${job.id} progress: ${progress}%`);
});

emailQueue.on('stalled', (job) => {
    console.log(`⚠️  Job ${job.id} stalled (worker crashed?)`);
});

// ============================================================
// QUEUE MONITORING
// ============================================================

async function getQueueStats() {
    const counts = await emailQueue.getJobCounts();

    console.log('\n' + '='.repeat(60));
    console.log('QUEUE STATISTICS');
    console.log('='.repeat(60));
    console.log(`Waiting:    ${counts.waiting}`);
    console.log(`Active:     ${counts.active}`);
    console.log(`Completed:  ${counts.completed}`);
    console.log(`Failed:     ${counts.failed}`);
    console.log(`Delayed:    ${counts.delayed}`);
    console.log('='.repeat(60) + '\n');
}

// ============================================================
// ADMIN OPERATIONS
// ============================================================

async function retryFailedJobs() {
    console.log('🔄 Retrying all failed jobs...');

    const failedJobs = await emailQueue.getFailed();
    console.log(`Found ${failedJobs.length} failed jobs`);

    for (const job of failedJobs) {
        await job.retry();
        console.log(`  ↻ Retrying job ${job.id}`);
    }
}

async function cleanOldJobs() {
    console.log('🧹 Cleaning old jobs...');

    // Nettoyer les jobs complétés de plus de 24h
    await emailQueue.clean(24 * 3600 * 1000, 'completed');

    // Nettoyer les jobs échoués de plus de 7 jours
    await emailQueue.clean(7 * 24 * 3600 * 1000, 'failed');

    console.log('✓ Cleanup completed');
}

async function pauseQueue() {
    await emailQueue.pause();
    console.log('⏸️  Queue paused');
}

async function resumeQueue() {
    await emailQueue.resume();
    console.log('▶️  Queue resumed');
}

// ============================================================
// WEB DASHBOARD (Express + Bull Board)
// ============================================================

const express = require('express');
const { createBullBoard } = require('@bull-board/api');
const { BullAdapter } = require('@bull-board/api/bullAdapter');
const { ExpressAdapter } = require('@bull-board/express');

const app = express();

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

createBullBoard({
    queues: [new BullAdapter(emailQueue)],
    serverAdapter: serverAdapter,
});

app.use('/admin/queues', serverAdapter.getRouter());

// API endpoints
app.get('/api/queue/stats', async (req, res) => {
    const counts = await emailQueue.getJobCounts();
    res.json(counts);
});

app.post('/api/queue/pause', async (req, res) => {
    await emailQueue.pause();
    res.json({ success: true, message: 'Queue paused' });
});

app.post('/api/queue/resume', async (req, res) => {
    await emailQueue.resume();
    res.json({ success: true, message: 'Queue resumed' });
});

app.post('/api/queue/retry-failed', async (req, res) => {
    await retryFailedJobs();
    res.json({ success: true, message: 'Failed jobs retried' });
});

const PORT = 3000;
app.listen(PORT, () => {
    console.log(`\n🌐 Dashboard: http://localhost:${PORT}/admin/queues`);
    console.log(`📊 API: http://localhost:${PORT}/api/queue/stats\n`);
});

// ============================================================
// MAIN
// ============================================================

async function main() {
    try {
        // Ajouter des jobs
        await addJobs();

        // Afficher les stats toutes les 5 secondes
        setInterval(async () => {
            await getQueueStats();
        }, 5000);

    } catch (error) {
        console.error('Error:', error);
    }
}

main();
```

---

## Monitoring et Observability

### Métriques importantes

```python
class QueueMetrics:
    """
    Collecte de métriques pour monitoring
    """

    def __init__(self, redis_client, queue_name='jobs'):
        self.redis = redis_client
        self.queue_name = f"queue:{queue_name}"
        self.metrics_key = f"{self.queue_name}:metrics"

    def record_enqueue(self):
        """Enregistre un job ajouté"""
        pipe = self.redis.pipeline()
        pipe.hincrby(self.metrics_key, 'total_enqueued', 1)
        pipe.hincrby(self.metrics_key, 'current_size', 1)
        pipe.execute()

    def record_dequeue(self):
        """Enregistre un job retiré"""
        pipe = self.redis.pipeline()
        pipe.hincrby(self.metrics_key, 'total_dequeued', 1)
        pipe.hincrby(self.metrics_key, 'current_size', -1)
        pipe.execute()

    def record_success(self, duration_ms):
        """Enregistre un succès"""
        pipe = self.redis.pipeline()
        pipe.hincrby(self.metrics_key, 'total_succeeded', 1)
        pipe.hincrbyfloat(self.metrics_key, 'total_duration_ms', duration_ms)
        pipe.execute()

    def record_failure(self):
        """Enregistre un échec"""
        self.redis.hincrby(self.metrics_key, 'total_failed', 1)

    def get_metrics(self) -> Dict[str, Any]:
        """Récupère toutes les métriques"""
        metrics = self.redis.hgetall(self.metrics_key)

        # Convertir en nombres
        for key in metrics:
            try:
                if '.' in metrics[key]:
                    metrics[key] = float(metrics[key])
                else:
                    metrics[key] = int(metrics[key])
            except:
                pass

        # Calculer des métriques dérivées
        if metrics.get('total_dequeued', 0) > 0:
            metrics['success_rate'] = (
                metrics.get('total_succeeded', 0) /
                metrics['total_dequeued'] * 100
            )

            metrics['avg_duration_ms'] = (
                metrics.get('total_duration_ms', 0) /
                metrics['total_succeeded']
            ) if metrics.get('total_succeeded', 0) > 0 else 0

        return metrics

    def display_metrics(self):
        """Affiche les métriques"""
        metrics = self.get_metrics()

        print("\n" + "=" * 60)
        print("QUEUE METRICS")
        print("=" * 60)
        print(f"Current size:     {metrics.get('current_size', 0)}")
        print(f"Total enqueued:   {metrics.get('total_enqueued', 0)}")
        print(f"Total dequeued:   {metrics.get('total_dequeued', 0)}")
        print(f"Total succeeded:  {metrics.get('total_succeeded', 0)}")
        print(f"Total failed:     {metrics.get('total_failed', 0)}")
        print(f"Success rate:     {metrics.get('success_rate', 0):.2f}%")
        print(f"Avg duration:     {metrics.get('avg_duration_ms', 0):.2f}ms")
        print("=" * 60 + "\n")


# ============================================================
# PROMETHEUS METRICS
# ============================================================

from prometheus_client import Counter, Histogram, Gauge

# Définir les métriques
queue_enqueued_total = Counter(
    'queue_enqueued_total',
    'Total jobs enqueued',
    ['queue_name']
)

queue_dequeued_total = Counter(
    'queue_dequeued_total',
    'Total jobs dequeued',
    ['queue_name']
)

queue_succeeded_total = Counter(
    'queue_succeeded_total',
    'Total jobs succeeded',
    ['queue_name']
)

queue_failed_total = Counter(
    'queue_failed_total',
    'Total jobs failed',
    ['queue_name']
)

queue_size = Gauge(
    'queue_size',
    'Current queue size',
    ['queue_name']
)

queue_processing_duration = Histogram(
    'queue_processing_duration_seconds',
    'Job processing duration',
    ['queue_name', 'job_type']
)


class MonitoredQueue(SimpleQueue):
    """Queue avec monitoring Prometheus"""

    def enqueue(self, job_data):
        job_id = super().enqueue(job_data)

        queue_enqueued_total.labels(queue_name=self.queue_name).inc()
        queue_size.labels(queue_name=self.queue_name).set(self.size())

        return job_id

    def dequeue(self, timeout=0):
        job = super().dequeue(timeout)

        if job:
            queue_dequeued_total.labels(queue_name=self.queue_name).inc()
            queue_size.labels(queue_name=self.queue_name).set(self.size())

        return job
```

---

## Bonnes pratiques

### Checklist de production

**Architecture :**
- ✅ Utiliser Reliable Queue (BRPOPLPUSH) en production
- ✅ Implémenter Dead Letter Queue pour jobs échoués
- ✅ Retry logic avec exponential backoff
- ✅ Priority queues pour jobs critiques
- ✅ Delayed jobs pour scheduling

**Performance :**
- ✅ Multiple workers pour scalabilité
- ✅ Batch processing quand possible
- ✅ Monitoring de la taille de la queue
- ✅ Auto-scaling des workers selon la charge
- ✅ Rate limiting sur les workers

**Fiabilité :**
- ✅ Job timeout pour éviter les workers bloqués
- ✅ Recovery des stale jobs au démarrage
- ✅ Idempotence des jobs (safe de retry)
- ✅ Logs détaillés pour debugging
- ✅ Alertes si queue trop longue

**Monitoring :**
- ✅ Métriques : taille, throughput, latence, errors
- ✅ Dashboard temps réel (Bull Board, Grafana)
- ✅ Alerting sur anomalies
- ✅ Distributed tracing pour jobs complexes

### Anti-patterns à éviter

```python
# ❌ BAD: Jobs non-idempotents
def process_payment(amount):
    charge_credit_card(amount)  # Appelé 2x si retry!

# ✅ GOOD: Jobs idempotents
def process_payment(payment_id):
    if not already_processed(payment_id):
        charge_credit_card(payment_id)
        mark_as_processed(payment_id)


# ❌ BAD: Pas de timeout
while True:
    job = queue.dequeue(timeout=None)  # Bloque indéfiniment
    process(job)

# ✅ GOOD: Timeout raisonnable
while True:
    job = queue.dequeue(timeout=5)
    if job:
        process(job)


# ❌ BAD: Pas de limite de retry
job['attempts'] = float('inf')  # Retry forever

# ✅ GOOD: Limite de retry
job['max_attempts'] = 3
if job['attempts'] >= job['max_attempts']:
    move_to_dlq(job)


# ❌ BAD: Jobs trop gros
queue.enqueue({
    'image_data': base64_encoded_10mb_image  # 10MB dans Redis!
})

# ✅ GOOD: Référence aux données
queue.enqueue({
    'image_url': 's3://bucket/image.jpg'  # Juste l'URL
})
```

---

## Conclusion

Les **Job Queues** sont essentielles pour construire des applications scalables et résilientes. Les points clés :

**Patterns implémentés :**
1. **Simple Queue (LIST)** : Rapide mais non-fiable
2. **Reliable Queue (BRPOPLPUSH)** : Production-ready avec retry
3. **Priority Queue (ZSET)** : Jobs par ordre d'importance
4. **Delayed Queue (ZSET + timestamp)** : Scheduling et cron

**Bibliothèques recommandées :**
- **Python** : RQ (simple), Celery (complet), custom avec Redis
- **Node.js** : Bull/BullMQ (excellent, production-ready)

**Best Practices :**
- Toujours utiliser Reliable Queue en production
- Implémenter retry logic avec exponential backoff
- Dead Letter Queue pour jobs définitivement échoués
- Monitoring obligatoire (métriques + alertes)
- Jobs idempotents (safe de retry)
- Multiple workers pour scalabilité

**Use Cases :**
- Background processing (emails, images, videos)
- Peak shaving (absorber les pics de charge)
- Scheduled tasks (cron-like)
- Microservices communication
- Long-running tasks

Redis est parfait pour les job queues : rapide, fiable, et simple à utiliser !

---


⏭️ [Atomicité et programmabilité](/07-atomicite-programmabilite/README.md)
