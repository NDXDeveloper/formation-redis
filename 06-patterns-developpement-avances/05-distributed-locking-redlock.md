🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5 Distributed Locking : Le pattern Redlock

## Introduction

Dans un système distribué, coordonner l'accès exclusif à une ressource partagée est un problème fondamental. Le **Distributed Locking** permet de garantir qu'une seule instance d'une application peut exécuter une section critique à un moment donné, même quand plusieurs instances s'exécutent en parallèle.

Redis propose une solution élégante avec le pattern **Redlock**, mais son utilisation correcte nécessite une compréhension profonde de ses garanties et limitations.

## Le problème des verrous distribués

### Scénario sans verrou

```text
PROBLÈME : Traitement concurrent sans coordination

Instance 1                    Redis                    Instance 2
    │                           │                          │
    ├─ Check job:123 status ───>│                          │
    │<─ "pending" ──────────────┤                          │
    │                           │                          │
    │                           │<─── Check job:123 ───────┤
    │                           ├──> "pending" ───────────>│
    │                           │                          │
    ├─ Process job:123 ────────>│                          │
    │                           │                          │
    │                           │<─── Process job:123 ─────┤
    │                           │                          │
    ❌ Job traité 2 fois!       │                          │
    ❌ Résultats incohérents    │                          │
    ❌ Waste de ressources      │                          │
```

### Cas d'usage typiques

**1. Job processing (traitement de tâches)**
```text
Scénario : 10 workers lisent depuis une queue

Sans lock:
- Worker 1 lit job:123
- Worker 2 lit job:123 (en même temps)
- Les deux traitent le job
- Double processing! ❌

Avec lock:
- Worker 1 acquiert lock:job:123
- Worker 2 tente d'acquérir lock:job:123 → FAIL
- Worker 2 passe au job suivant
- Worker 1 traite job:123
- Single processing ✅
```

**2. Rate limiting distribué**
```text
Scénario : Limiter les appels API à 100/minute par user

Sans lock:
- Instance 1 : check count = 99, increment → 100 ✅
- Instance 2 : check count = 99, increment → 100 ✅ (race!)
- Résultat : 101 calls (limite dépassée) ❌

Avec lock:
- Instance 1 : acquire lock → check count = 99 → increment → release
- Instance 2 : attend le lock → check count = 100 → refuse
- Résultat : 100 calls exactement ✅
```

**3. Resource allocation (slots, tickets, inventory)**
```text
Scénario : E-commerce - Vente flash avec 1 produit restant

Sans lock:
- User A : check stock = 1 → achète → stock = 0
- User B : check stock = 1 → achète → stock = -1 (overselling!) ❌

Avec lock:
- User A : acquire lock → check stock = 1 → achète → release
- User B : attend lock → check stock = 0 → refuse
- No overselling ✅
```

---

## Simple Redis Lock (Approche basique)

### Principe

Le lock simple utilise une commande atomique Redis : `SET key value NX EX seconds`

```text
SET lock:resource "owner-id" NX EX 30

NX : Only set if Not eXists (atomique)
EX : EXpire after N seconds (auto-release si crash)

Returns:
- OK si lock acquis
- nil si lock déjà pris
```

### Implémentation Python basique

```python
import redis
import uuid
import time

class SimpleRedisLock:
    """
    Verrou Redis simple avec SET NX EX
    """

    def __init__(self, redis_client, lock_name, ttl=30):
        self.redis = redis_client
        self.lock_name = f"lock:{lock_name}"
        self.ttl = ttl
        self.lock_id = str(uuid.uuid4())  # ID unique pour ce lock

    def acquire(self, blocking=True, timeout=None):
        """
        Acquiert le verrou

        Args:
            blocking: Si True, attend jusqu'à obtenir le lock
            timeout: Temps max d'attente (secondes)

        Returns:
            bool: True si lock acquis, False sinon
        """
        start_time = time.time()

        while True:
            # Tentative d'acquisition atomique
            acquired = self.redis.set(
                self.lock_name,
                self.lock_id,
                nx=True,  # Only if not exists
                ex=self.ttl  # Expire after TTL seconds
            )

            if acquired:
                print(f"✓ Lock '{self.lock_name}' acquired by {self.lock_id}")
                return True

            # Lock déjà pris
            if not blocking:
                print(f"✗ Lock '{self.lock_name}' already held")
                return False

            # Vérifier timeout
            if timeout and (time.time() - start_time) >= timeout:
                print(f"✗ Lock '{self.lock_name}' timeout after {timeout}s")
                return False

            # Attendre un peu avant de réessayer
            time.sleep(0.1)

    def release(self):
        """
        Libère le verrou (seulement si on le possède)
        """
        # Script Lua pour vérifier que c'est bien notre lock avant de le supprimer
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """

        result = self.redis.eval(lua_script, 1, self.lock_name, self.lock_id)

        if result:
            print(f"✓ Lock '{self.lock_name}' released by {self.lock_id}")
            return True
        else:
            print(f"⚠️  Lock '{self.lock_name}' not owned by {self.lock_id}")
            return False

    def __enter__(self):
        """Support du context manager"""
        self.acquire()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Libération automatique du lock"""
        self.release()


# ============================================================
# EXEMPLE D'UTILISATION
# ============================================================

def process_job_with_lock(job_id):
    """
    Traite un job avec protection par lock
    """
    redis_client = redis.Redis(decode_responses=True)

    # Utiliser le context manager
    with SimpleRedisLock(redis_client, f"job:{job_id}", ttl=30):
        print(f"Processing job {job_id}...")

        # Section critique protégée
        time.sleep(2)  # Simuler un traitement

        print(f"Job {job_id} completed")

    # Lock automatiquement libéré


# ============================================================
# TEST CONCURRENT
# ============================================================

import threading

def worker_thread(worker_id, job_id):
    """Thread worker simulant un traitement concurrent"""
    print(f"Worker {worker_id} starting...")

    redis_client = redis.Redis(decode_responses=True)
    lock = SimpleRedisLock(redis_client, f"job:{job_id}", ttl=10)

    # Tentative non-bloquante
    if lock.acquire(blocking=False):
        try:
            print(f"  Worker {worker_id} is processing job {job_id}")
            time.sleep(3)
            print(f"  Worker {worker_id} finished job {job_id}")
        finally:
            lock.release()
    else:
        print(f"  Worker {worker_id} could not acquire lock (job being processed)")


def test_concurrent_access():
    """Test avec plusieurs workers concurrents"""
    print("=" * 60)
    print("TEST: Concurrent Access with Simple Lock")
    print("=" * 60 + "\n")

    job_id = 123
    threads = []

    # Lancer 5 workers concurrents
    for i in range(5):
        t = threading.Thread(target=worker_thread, args=(i, job_id))
        threads.append(t)
        t.start()

    # Attendre tous les threads
    for t in threads:
        t.join()

    print("\n✓ Test completed: Only one worker processed the job")
```

### Visualisation du flux

```text
┌─────────────────────────────────────────────────────────────┐
│               SIMPLE REDIS LOCK FLOW                        │
└─────────────────────────────────────────────────────────────┘

Worker 1                      Redis                      Worker 2
   │                            │                            │
   │                            │                            │
   ├─ SET lock:job NX EX 30 ───>│                            │
   │                            │                            │
   │<─────── OK ────────────────┤  Lock acquired ✅          │
   │                            │                            │
   │                            │<─── SET lock:job NX EX ────┤
   │                            │                            │
   │                            ├────────── nil ────────────>│
   │                            │   Lock already held ❌     │
   │                            │                            │
   │  [Processing job...]       │                            │
   │                            │                            │
   │  DEL lock:job (Lua script) │                            │
   ├───────────────────────────>│                            │
   │                            │                            │
   │<─────── 1 ─────────────────┤  Lock released ✅          │
   │                            │                            │
   │                            │<─── SET lock:job NX EX ────┤
   │                            │                            │
   │                            ├────────── OK ─────────────>│
   │                            │   Lock acquired ✅         │
```

---

## Problèmes du Simple Lock

### 1. Single Point of Failure

```text
PROBLÈME : Si le master Redis crash, les locks sont perdus

┌─────────────────────────────────────────────────────────────┐
│                    REDIS MASTER CRASH                       │
└─────────────────────────────────────────────────────────────┘

Time    Worker 1         Master Redis         Replica         Worker 2
────────────────────────────────────────────────────────────────────
t0      Acquire lock ──> [lock:job = id1]       │              │
t1      Processing...    [lock:job = id1]       │              │
t2      Processing...    💥 CRASH!              │              │
t3      Processing...         X            Promoted to         │
                                            Master             │
t4      Processing...                      [empty!]            │
                                         (no lock data)        │
t5      Processing...                      [empty]  <─── Acquire lock
                                                         (succeeds!)
t6      ❌ Both workers processing the same job!

Résultat : Lock perdu → Double processing
```

### 2. Clock Drift (Dérive d'horloge)

```text
PROBLÈME : TTL expire trop tôt à cause de dérive d'horloge

Worker                        Redis (clock +5s ahead)
  │                                 │
  ├─ SET lock:job NX EX 10 ────────>│  t=0 (Redis time)
  │                                 │  Lock expires at t=10
  │                                 │
  │  [Processing... long task]      │
  │  [Takes 8 seconds]              │  t=5 (real time)
  │                                 │  But Redis thinks t=10
  │                                 │  Lock expires! ❌
  │                                 │
  │  Still processing...            │  Lock gone!
  │                                 │
  │                                 │<─── SET lock:job (Worker 2)
  │                                 │     Acquires lock ✅
  │                                 │
  ❌ Both workers now have "the lock"
```

### 3. Network Partition

```text
PROBLÈME : Worker isolé par partition réseau

Worker A            Network           Redis            Worker B
   │                   │                 │                 │
   │──────────────────>│────────────────>│                 │
   │  Acquire lock     │                 │  Lock acquired  │
   │<──────────────────┤<────────────────┤                 │
   │                   │                 │                 │
   │  Processing...    │                 │                 │
   │                   │                 │                 │
   │                   ✂️ PARTITION      │                 │
   │  X X X X X X X X  │                 │                 │
   │  Can't reach      │                 │                 │
   │  Redis to renew   │                 │  TTL expires    │
   │                   │                 │  (no renewal)   │
   │                   │                 │                 │
   │  Still thinks it  │                 │<────────────────┤
   │  has the lock!    │                 │  Worker B       │
   │                   │                 │  acquires lock  │
   │                   │                 │                 │
   ❌ Both workers think they have the lock
```

---

## Le pattern Redlock

### Principe

**Redlock** résout les problèmes du simple lock en utilisant **plusieurs instances Redis indépendantes** (pas de réplication) et en exigeant un **quorum** pour considérer le lock acquis.

### Architecture Redlock

```text
┌─────────────────────────────────────────────────────────────┐
│              REDLOCK ARCHITECTURE (N=5)                     │
└─────────────────────────────────────────────────────────────┘

                           Worker
                             │
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
         ┌────────┐     ┌────────┐     ┌────────┐
         │Redis 1 │     │Redis 2 │     │Redis 3 │
         │Master  │     │Master  │     │Master  │
         └────────┘     └────────┘     └────────┘
              ▲              ▲
              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                        ┌────────┐     ┌────────┐
                        │Redis 4 │     │Redis 5 │
                        │Master  │     │Master  │
                        └────────┘     └────────┘

Quorum = N/2 + 1 = 3/5

Lock acquired if:
✅ At least 3/5 instances respond OK
✅ Total time < Lock TTL
✅ Valid time remaining > 0
```

### Algorithme Redlock

```text
ALGORITHM: Acquire Redlock
────────────────────────────────────────────────────────────

1. Get current timestamp (milliseconds)
   start_time = now()

2. Try to acquire lock on ALL N instances sequentially:
   FOR each Redis instance:
       SET lock:resource "unique-id" NX PX ttl
       (with small timeout, e.g., 5-50ms)

3. Calculate elapsed time:
   elapsed = now() - start_time

4. Lock is acquired if and only if:
   - Got OK from at least N/2 + 1 instances (quorum)
   - elapsed < lock_ttl (we're not too slow)
   - validity_time = lock_ttl - elapsed > 0

5. If lock acquired:
   - Use resource with remaining validity_time
   - When done, release lock on ALL instances

6. If lock NOT acquired:
   - Release lock on ALL instances (cleanup)
   - Retry after random delay

ALGORITHM: Release Redlock
────────────────────────────────────────────────────────────

1. FOR each Redis instance:
       Execute Lua script:
       if redis.call("get",KEYS[1]) == ARGV[1] then
           return redis.call("del",KEYS[1])
       else
           return 0
       end

2. Don't check results (best effort release)
```

---

## Implémentation Python complète

### Redlock Implementation

```python
import redis
import uuid
import time
import random

class Redlock:
    """
    Implémentation du pattern Redlock pour locks distribués
    """

    # Script Lua pour release atomique
    UNLOCK_SCRIPT = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """

    def __init__(self, redis_instances, lock_name, ttl=10000, retry_count=3, retry_delay=200):
        """
        Args:
            redis_instances: Liste de connexions Redis
            lock_name: Nom du lock
            ttl: Time-to-live en millisecondes
            retry_count: Nombre de tentatives
            retry_delay: Délai entre tentatives (ms)
        """
        self.instances = redis_instances
        self.lock_name = f"redlock:{lock_name}"
        self.ttl = ttl
        self.retry_count = retry_count
        self.retry_delay = retry_delay
        self.lock_id = str(uuid.uuid4())
        self.quorum = len(redis_instances) // 2 + 1
        self.validity_time = 0

    def acquire(self):
        """
        Acquiert le lock Redlock

        Returns:
            bool: True si lock acquis avec quorum
        """
        for attempt in range(self.retry_count):
            if attempt > 0:
                # Random delay avant retry (évite thundering herd)
                delay = random.uniform(0, self.retry_delay) / 1000.0
                time.sleep(delay)

            # Tentative d'acquisition
            if self._try_acquire():
                return True

        # Échec après tous les retries
        return False

    def _try_acquire(self):
        """
        Tentative unique d'acquisition du lock
        """
        start_time = self._current_time_ms()
        acquired_count = 0

        # Tenter d'acquérir sur toutes les instances
        for instance in self.instances:
            if self._acquire_instance(instance):
                acquired_count += 1

        # Calculer le temps écoulé
        elapsed = self._current_time_ms() - start_time

        # Calculer le temps de validité restant
        self.validity_time = self.ttl - elapsed - self._drift()

        # Vérifier le quorum et la validité
        if acquired_count >= self.quorum and self.validity_time > 0:
            print(f"✓ Redlock acquired: {acquired_count}/{len(self.instances)} instances")
            print(f"  Lock ID: {self.lock_id}")
            print(f"  Validity: {self.validity_time}ms")
            return True
        else:
            # Échec : nettoyer les locks partiels
            print(f"✗ Redlock failed: {acquired_count}/{len(self.instances)} instances")
            print(f"  Quorum needed: {self.quorum}")
            self._release_all()
            return False

    def _acquire_instance(self, instance):
        """
        Tente d'acquérir le lock sur une instance Redis
        """
        try:
            # SET avec NX (not exists) et PX (expire milliseconds)
            result = instance.set(
                self.lock_name,
                self.lock_id,
                nx=True,
                px=self.ttl
            )
            return result is not None
        except Exception as e:
            print(f"⚠️  Failed to acquire on instance: {e}")
            return False

    def release(self):
        """
        Libère le lock sur toutes les instances
        """
        released_count = 0

        for instance in self.instances:
            try:
                result = instance.eval(
                    self.UNLOCK_SCRIPT,
                    1,
                    self.lock_name,
                    self.lock_id
                )
                if result:
                    released_count += 1
            except Exception as e:
                print(f"⚠️  Failed to release on instance: {e}")

        print(f"✓ Redlock released on {released_count}/{len(self.instances)} instances")

    def _release_all(self):
        """
        Libère le lock sur toutes les instances (best effort)
        """
        for instance in self.instances:
            try:
                instance.eval(self.UNLOCK_SCRIPT, 1, self.lock_name, self.lock_id)
            except:
                pass  # Ignore errors during cleanup

    def _current_time_ms(self):
        """Timestamp actuel en millisecondes"""
        return int(time.time() * 1000)

    def _drift(self):
        """
        Clock drift adjustment (recommandation Redlock)
        Typiquement 1% du TTL
        """
        return int(self.ttl * 0.01)

    def __enter__(self):
        """Support du context manager"""
        if not self.acquire():
            raise Exception("Failed to acquire Redlock")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Libération automatique"""
        self.release()


# ============================================================
# CONFIGURATION DES INSTANCES REDIS
# ============================================================

def setup_redis_instances():
    """
    Configure 5 instances Redis indépendantes
    """
    # En production : 5 serveurs physiques différents
    # Pour le test : 5 instances sur ports différents

    instances = [
        redis.Redis(host='localhost', port=6379, decode_responses=True),
        redis.Redis(host='localhost', port=6380, decode_responses=True),
        redis.Redis(host='localhost', port=6381, decode_responses=True),
        redis.Redis(host='localhost', port=6382, decode_responses=True),
        redis.Redis(host='localhost', port=6383, decode_responses=True),
    ]

    return instances


# ============================================================
# EXEMPLE D'UTILISATION
# ============================================================

def process_critical_section():
    """
    Exécute une section critique protégée par Redlock
    """
    instances = setup_redis_instances()

    lock = Redlock(
        instances,
        lock_name="critical:resource",
        ttl=10000,  # 10 secondes
        retry_count=3,
        retry_delay=200
    )

    if lock.acquire():
        try:
            print("\n🔒 Lock acquired - Processing critical section...")

            # Section critique
            time.sleep(2)
            print("   Processing...")
            time.sleep(2)
            print("   Almost done...")

            print("✓ Critical section completed\n")

        finally:
            lock.release()
    else:
        print("❌ Could not acquire lock")


# ============================================================
# TEST AVEC CONTEXT MANAGER
# ============================================================

def process_with_context_manager():
    """
    Utilisation simplifiée avec context manager
    """
    instances = setup_redis_instances()

    try:
        with Redlock(instances, "critical:job", ttl=5000):
            print("🔒 Processing with Redlock protection...")
            time.sleep(1)
            print("✓ Done")
    except Exception as e:
        print(f"❌ Failed to acquire lock: {e}")


# ============================================================
# TEST CONCURRENT
# ============================================================

import threading

def worker_with_redlock(worker_id):
    """Worker utilisant Redlock"""
    print(f"Worker {worker_id} starting...")

    instances = setup_redis_instances()
    lock = Redlock(instances, "shared:resource", ttl=5000)

    if lock.acquire():
        try:
            print(f"  ✓ Worker {worker_id} acquired lock")
            time.sleep(2)
            print(f"  ✓ Worker {worker_id} finished work")
        finally:
            lock.release()
    else:
        print(f"  ✗ Worker {worker_id} could not acquire lock")


def test_concurrent_redlock():
    """Test avec plusieurs workers concurrents"""
    print("=" * 60)
    print("TEST: Concurrent Access with Redlock")
    print("=" * 60 + "\n")

    threads = []

    # Lancer 3 workers concurrents
    for i in range(3):
        t = threading.Thread(target=worker_with_redlock, args=(i,))
        threads.append(t)
        t.start()
        time.sleep(0.1)  # Petit délai entre démarrages

    for t in threads:
        t.join()

    print("\n✓ Test completed")


# ============================================================
# BENCHMARK : Simple Lock vs Redlock
# ============================================================

def benchmark_lock_acquisition():
    """
    Compare les performances des deux approches
    """
    print("=" * 60)
    print("BENCHMARK: Lock Acquisition Performance")
    print("=" * 60 + "\n")

    # Simple lock
    redis_client = redis.Redis(decode_responses=True)
    simple_lock = SimpleRedisLock(redis_client, "benchmark", ttl=10)

    start = time.time()
    simple_lock.acquire()
    simple_elapsed = (time.time() - start) * 1000
    simple_lock.release()

    print(f"Simple Lock: {simple_elapsed:.2f}ms")

    # Redlock
    instances = setup_redis_instances()
    redlock = Redlock(instances, "benchmark", ttl=10000)

    start = time.time()
    redlock.acquire()
    redlock_elapsed = (time.time() - start) * 1000
    redlock.release()

    print(f"Redlock:     {redlock_elapsed:.2f}ms")
    print(f"\nOverhead: {redlock_elapsed - simple_elapsed:.2f}ms ({redlock_elapsed / simple_elapsed:.1f}x slower)")


# ============================================================
# MAIN
# ============================================================

if __name__ == '__main__':
    # Test basique
    print("=" * 60)
    print("REDLOCK BASIC TEST")
    print("=" * 60 + "\n")
    process_critical_section()

    # Test concurrent
    test_concurrent_redlock()

    # Benchmark
    benchmark_lock_acquisition()
```

---

## Implémentation Node.js

### Redlock avec node-redlock

```javascript
const Redis = require('ioredis');
const Redlock = require('redlock');

// ============================================================
// CONFIGURATION DES INSTANCES
// ============================================================

function setupRedisInstances() {
    // Créer 5 instances Redis indépendantes
    return [
        new Redis({ host: 'localhost', port: 6379 }),
        new Redis({ host: 'localhost', port: 6380 }),
        new Redis({ host: 'localhost', port: 6381 }),
        new Redis({ host: 'localhost', port: 6382 }),
        new Redis({ host: 'localhost', port: 6383 }),
    ];
}

// ============================================================
// REDLOCK CLIENT
// ============================================================

function createRedlockClient() {
    const clients = setupRedisInstances();

    const redlock = new Redlock(clients, {
        // Retry automatique
        retryCount: 3,
        retryDelay: 200,  // milliseconds
        retryJitter: 50,  // random jitter

        // Drift factor (clock drift adjustment)
        driftFactor: 0.01,  // 1%

        // Timeouts
        automaticExtensionThreshold: 500,  // extend if < 500ms remaining
    });

    // Event listeners
    redlock.on('clientError', (err) => {
        console.error('Redlock client error:', err);
    });

    return { redlock, clients };
}

// ============================================================
// EXEMPLE BASIQUE
// ============================================================

async function basicExample() {
    console.log('='.repeat(60));
    console.log('REDLOCK BASIC EXAMPLE');
    console.log('='.repeat(60) + '\n');

    const { redlock, clients } = createRedlockClient();

    try {
        // Acquérir le lock
        const lock = await redlock.acquire(
            ['redlock:critical:resource'],
            5000  // TTL: 5 secondes
        );

        console.log('✓ Lock acquired');
        console.log(`  Resource: ${lock.resources}`);
        console.log(`  Value: ${lock.value}`);
        console.log(`  Expiration: ${lock.expiration}`);

        // Section critique
        console.log('\n🔒 Processing critical section...');
        await new Promise(resolve => setTimeout(resolve, 2000));
        console.log('✓ Processing completed\n');

        // Libérer le lock
        await lock.release();
        console.log('✓ Lock released');

    } catch (error) {
        console.error('❌ Error:', error.message);
    } finally {
        // Fermer les connexions
        await Promise.all(clients.map(client => client.quit()));
    }
}

// ============================================================
// USING BLOCK (Pattern recommandé)
// ============================================================

async function usingBlockExample() {
    console.log('\n' + '='.repeat(60));
    console.log('REDLOCK USING BLOCK PATTERN');
    console.log('='.repeat(60) + '\n');

    const { redlock, clients } = createRedlockClient();

    try {
        // Using block : acquire + execute + release automatique
        await redlock.using(
            ['redlock:job:123'],
            5000,  // TTL
            async (signal) => {
                // Section critique
                console.log('🔒 Lock acquired, processing job...');

                // Vérifier si le lock est toujours valide
                if (signal.aborted) {
                    throw new Error('Lock expired during processing');
                }

                await new Promise(resolve => setTimeout(resolve, 1000));
                console.log('✓ Job completed');
            }
        );

        console.log('✓ Lock automatically released');

    } catch (error) {
        console.error('❌ Error:', error.message);
    } finally {
        await Promise.all(clients.map(client => client.quit()));
    }
}

// ============================================================
// LOCK EXTENSION (Prolonger le TTL)
// ============================================================

async function lockExtensionExample() {
    console.log('\n' + '='.repeat(60));
    console.log('LOCK EXTENSION EXAMPLE');
    console.log('='.repeat(60) + '\n');

    const { redlock, clients } = createRedlockClient();

    try {
        // Acquérir avec TTL court
        let lock = await redlock.acquire(['redlock:long:task'], 3000);
        console.log('✓ Lock acquired (3s TTL)');

        // Traitement long
        for (let i = 0; i < 5; i++) {
            console.log(`  Processing step ${i + 1}/5...`);
            await new Promise(resolve => setTimeout(resolve, 1000));

            // Prolonger le lock si nécessaire
            if (i < 4) {
                lock = await lock.extend(3000);
                console.log('  ↻ Lock extended (+3s)');
            }
        }

        console.log('✓ Long task completed');
        await lock.release();
        console.log('✓ Lock released');

    } catch (error) {
        console.error('❌ Error:', error.message);
    } finally {
        await Promise.all(clients.map(client => client.quit()));
    }
}

// ============================================================
// MULTI-RESOURCE LOCK
// ============================================================

async function multiResourceExample() {
    console.log('\n' + '='.repeat(60));
    console.log('MULTI-RESOURCE LOCK EXAMPLE');
    console.log('='.repeat(60) + '\n');

    const { redlock, clients } = createRedlockClient();

    try {
        // Acquérir plusieurs ressources atomiquement
        const lock = await redlock.acquire(
            [
                'redlock:user:123',
                'redlock:account:456',
                'redlock:transaction:789'
            ],
            5000
        );

        console.log('✓ Multi-resource lock acquired');
        console.log(`  Resources: ${lock.resources.length}`);

        // Opération nécessitant toutes les ressources
        console.log('\n🔒 Processing multi-resource transaction...');
        await new Promise(resolve => setTimeout(resolve, 1000));
        console.log('✓ Transaction completed\n');

        await lock.release();
        console.log('✓ All resources released');

    } catch (error) {
        console.error('❌ Error:', error.message);
    } finally {
        await Promise.all(clients.map(client => client.quit()));
    }
}

// ============================================================
// CONCURRENT TEST
// ============================================================

async function concurrentTest() {
    console.log('\n' + '='.repeat(60));
    console.log('CONCURRENT ACCESS TEST');
    console.log('='.repeat(60) + '\n');

    const { redlock, clients } = createRedlockClient();

    async function worker(id) {
        try {
            console.log(`Worker ${id} attempting to acquire lock...`);

            const lock = await redlock.acquire(
                ['redlock:shared:resource'],
                3000
            );

            console.log(`  ✓ Worker ${id} acquired lock`);
            await new Promise(resolve => setTimeout(resolve, 1000));
            console.log(`  ✓ Worker ${id} completed work`);

            await lock.release();

        } catch (error) {
            console.log(`  ✗ Worker ${id} failed: ${error.message}`);
        }
    }

    // Lancer 5 workers concurrents
    await Promise.all([
        worker(1),
        worker(2),
        worker(3),
        worker(4),
        worker(5)
    ]);

    console.log('\n✓ Concurrent test completed');

    await Promise.all(clients.map(client => client.quit()));
}

// ============================================================
// ERROR HANDLING
// ============================================================

async function errorHandlingExample() {
    console.log('\n' + '='.repeat(60));
    console.log('ERROR HANDLING EXAMPLE');
    console.log('='.repeat(60) + '\n');

    const { redlock, clients } = createRedlockClient();

    try {
        await redlock.using(
            ['redlock:critical:resource'],
            5000,
            async (signal) => {
                console.log('🔒 Processing...');

                // Simuler une erreur
                await new Promise(resolve => setTimeout(resolve, 500));
                throw new Error('Processing failed!');
            }
        );
    } catch (error) {
        console.error('❌ Error caught:', error.message);
        console.log('✓ Lock was automatically released despite error');
    } finally {
        await Promise.all(clients.map(client => client.quit()));
    }
}

// ============================================================
// PATTERN : JOB QUEUE WITH REDLOCK
// ============================================================

class JobQueue {
    constructor(redlock) {
        this.redlock = redlock;
    }

    async processJob(jobId) {
        console.log(`\nProcessing job ${jobId}...`);

        try {
            await this.redlock.using(
                [`redlock:job:${jobId}`],
                30000,  // 30s TTL
                async (signal) => {
                    console.log(`  ✓ Lock acquired for job ${jobId}`);

                    // Simulate job processing
                    await this.simulateJobProcessing(jobId, signal);

                    console.log(`  ✓ Job ${jobId} completed`);
                }
            );
        } catch (error) {
            if (error.message.includes('lock already held')) {
                console.log(`  ⚠️  Job ${jobId} is being processed by another worker`);
            } else {
                console.error(`  ❌ Job ${jobId} failed:`, error.message);
            }
        }
    }

    async simulateJobProcessing(jobId, signal) {
        // Check signal periodically
        for (let i = 0; i < 5; i++) {
            if (signal.aborted) {
                throw new Error('Job processing aborted (lock expired)');
            }

            console.log(`    Step ${i + 1}/5...`);
            await new Promise(resolve => setTimeout(resolve, 500));
        }
    }
}

async function jobQueueExample() {
    console.log('\n' + '='.repeat(60));
    console.log('JOB QUEUE WITH REDLOCK');
    console.log('='.repeat(60));

    const { redlock, clients } = createRedlockClient();
    const queue = new JobQueue(redlock);

    // Process multiple jobs concurrently
    await Promise.all([
        queue.processJob(1),
        queue.processJob(2),
        queue.processJob(1),  // Duplicate - should fail
        queue.processJob(3),
    ]);

    console.log('\n✓ Job queue test completed');

    await Promise.all(clients.map(client => client.quit()));
}

// ============================================================
// MAIN
// ============================================================

async function main() {
    try {
        await basicExample();
        await usingBlockExample();
        await lockExtensionExample();
        await multiResourceExample();
        await concurrentTest();
        await errorHandlingExample();
        await jobQueueExample();

    } catch (error) {
        console.error('Fatal error:', error);
    }
}

// Run
main();
```

---

## Cas d'usage réels

### 1. Scheduled Jobs (Cron distribué)

```python
import schedule
import time

class DistributedScheduler:
    """
    Scheduler distribué avec Redlock pour éviter les doublons
    """

    def __init__(self, redis_instances):
        self.redis_instances = redis_instances

    def run_once(self, job_name, func, *args, **kwargs):
        """
        Exécute un job une seule fois, même avec plusieurs instances
        """
        lock = Redlock(
            self.redis_instances,
            f"cron:{job_name}",
            ttl=60000,  # 1 minute max
            retry_count=1  # Pas de retry pour les crons
        )

        if lock.acquire():
            try:
                print(f"🕐 Executing scheduled job: {job_name}")
                result = func(*args, **kwargs)
                print(f"✓ Job {job_name} completed")
                return result
            finally:
                lock.release()
        else:
            print(f"⏭️  Job {job_name} already running on another instance")
            return None


# Utilisation
def send_daily_report():
    print("Generating daily report...")
    time.sleep(2)
    print("Report sent!")

def backup_database():
    print("Backing up database...")
    time.sleep(3)
    print("Backup completed!")

# Setup
instances = setup_redis_instances()
scheduler = DistributedScheduler(instances)

# Schedule jobs (lancé sur toutes les instances, mais exécuté une seule fois)
schedule.every().day.at("09:00").do(
    scheduler.run_once,
    "daily_report",
    send_daily_report
)

schedule.every().day.at("02:00").do(
    scheduler.run_once,
    "db_backup",
    backup_database
)

# Run scheduler
while True:
    schedule.run_pending()
    time.sleep(60)
```

### 2. Rate Limiting distribué

```python
class DistributedRateLimiter:
    """
    Rate limiter distribué avec Redlock
    """

    def __init__(self, redis_instances):
        self.redis_instances = redis_instances

    def allow_request(self, user_id, max_requests=100, window=60):
        """
        Vérifie si la requête est autorisée

        Args:
            user_id: ID de l'utilisateur
            max_requests: Nombre max de requêtes
            window: Fenêtre en secondes

        Returns:
            bool: True si autorisé
        """
        lock_key = f"ratelimit:{user_id}"
        counter_key = f"counter:{user_id}"

        lock = Redlock(
            self.redis_instances,
            lock_key,
            ttl=1000,  # 1 seconde
            retry_count=3
        )

        if lock.acquire():
            try:
                # Utiliser la première instance pour le compteur
                redis_client = self.redis_instances[0]

                # Obtenir le compteur actuel
                count = redis_client.get(counter_key)
                count = int(count) if count else 0

                if count >= max_requests:
                    print(f"❌ Rate limit exceeded for user {user_id}")
                    return False

                # Incrémenter
                pipe = redis_client.pipeline()
                pipe.incr(counter_key)
                pipe.expire(counter_key, window)
                pipe.execute()

                print(f"✓ Request allowed for user {user_id} ({count + 1}/{max_requests})")
                return True

            finally:
                lock.release()
        else:
            # Si on ne peut pas acquérir le lock, on refuse par sécurité
            print(f"⚠️  Could not acquire lock for user {user_id}, request denied")
            return False


# Utilisation
limiter = DistributedRateLimiter(setup_redis_instances())

# Simuler des requêtes
for i in range(105):
    allowed = limiter.allow_request("user:123", max_requests=100, window=60)
    if not allowed:
        print(f"Request {i + 1} blocked")
```

### 3. Inventory Management (E-commerce)

```python
class DistributedInventory:
    """
    Gestion d'inventaire distribué sans overselling
    """

    def __init__(self, redis_instances):
        self.redis_instances = redis_instances

    def purchase_item(self, product_id, quantity=1):
        """
        Achète un produit avec garantie anti-overselling
        """
        lock = Redlock(
            self.redis_instances,
            f"inventory:{product_id}",
            ttl=5000,
            retry_count=5,
            retry_delay=100
        )

        if lock.acquire():
            try:
                # Utiliser la première instance pour l'inventaire
                redis_client = self.redis_instances[0]
                stock_key = f"stock:{product_id}"

                # Vérifier le stock
                current_stock = redis_client.get(stock_key)
                current_stock = int(current_stock) if current_stock else 0

                if current_stock < quantity:
                    print(f"❌ Insufficient stock for product {product_id}")
                    print(f"   Available: {current_stock}, Requested: {quantity}")
                    return False

                # Décrémenter le stock
                new_stock = redis_client.decrby(stock_key, quantity)

                print(f"✓ Purchase successful!")
                print(f"   Product: {product_id}")
                print(f"   Quantity: {quantity}")
                print(f"   Remaining stock: {new_stock}")

                return True

            finally:
                lock.release()
        else:
            print(f"❌ Could not acquire lock for product {product_id}")
            return False


# Simulation vente flash avec 10 workers concurrents
import threading

def simulate_purchase(inventory, product_id, worker_id):
    print(f"Worker {worker_id} attempting purchase...")
    success = inventory.purchase_item(product_id, quantity=1)
    if success:
        print(f"  Worker {worker_id} purchased successfully")
    else:
        print(f"  Worker {worker_id} purchase failed")

# Setup
instances = setup_redis_instances()
inventory = DistributedInventory(instances)

# Initialiser le stock
instances[0].set("stock:product:123", 5)  # Seulement 5 items

# Lancer 10 workers qui tentent d'acheter
threads = []
for i in range(10):
    t = threading.Thread(
        target=simulate_purchase,
        args=(inventory, "product:123", i)
    )
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# Vérifier le stock final
final_stock = instances[0].get("stock:product:123")
print(f"\n✓ Final stock: {final_stock}")
print(f"  Expected: 0 (5 purchases from 10 attempts)")
```

---

## Limitations et alternatives

### Limitations de Redlock

```text
┌─────────────────────────────────────────────────────────────┐
│                    REDLOCK LIMITATIONS                      │
└─────────────────────────────────────────────────────────────┘

1. ❌ Pas de garantie absolue en cas de :
   - Pause GC (garbage collection) longue
   - Clock drift important entre instances
   - Network partition prolongée

2. ⚠️  Complexité opérationnelle :
   - Besoin de N instances Redis (N=5 recommandé)
   - Monitoring de toutes les instances
   - Coordination réseau

3. 🐌 Performance :
   - Latence = N × RTT (séquentiel)
   - Overhead par rapport à simple lock

4. 💰 Coût :
   - N instances Redis à maintenir
   - Plus de ressources nécessaires
```

### Tableau comparatif

| Aspect | Simple Lock | Redlock | ZooKeeper | etcd |
|--------|-------------|---------|-----------|------|
| **Setup** | ⭐⭐⭐⭐⭐ Simple | ⭐⭐⭐ Medium | ⭐⭐ Complex | ⭐⭐ Complex |
| **Performance** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐ Good | ⭐⭐⭐ Good | ⭐⭐⭐ Good |
| **Garanties** | ⭐⭐ Weak | ⭐⭐⭐ Medium | ⭐⭐⭐⭐⭐ Strong | ⭐⭐⭐⭐⭐ Strong |
| **Disponibilité** | ⭐⭐ Low | ⭐⭐⭐⭐ High | ⭐⭐⭐⭐⭐ Very High | ⭐⭐⭐⭐⭐ Very High |
| **Use case** | Cache, jobs | Critical jobs | Coordination | Coordination |

### Quand utiliser quoi ?

```python
# ✅ Simple Redis Lock
def use_simple_lock():
    """
    Bon pour :
    - Cache invalidation
    - Job processing (non-critique)
    - Rate limiting (soft)
    - Une seule instance Redis suffit
    - Performance critique
    """
    lock = SimpleRedisLock(redis_client, "job:123")
    with lock:
        process_job()

# ✅ Redlock
def use_redlock():
    """
    Bon pour :
    - Opérations importantes mais pas critiques
    - E-commerce (inventory)
    - Scheduled jobs (cron)
    - Balance entre performance et garanties
    - Tolérance aux erreurs acceptable
    """
    lock = Redlock(redis_instances, "inventory:123")
    with lock:
        update_inventory()

# ✅ ZooKeeper/etcd
def use_zookeeper():
    """
    Bon pour :
    - Transactions financières
    - Leader election
    - Configuration distribuée
    - Garanties fortes requises
    - Coordination de services
    - Tolérance zéro aux erreurs
    """
    # Utiliser ZooKeeper ou etcd pour strong consistency
    pass
```

---

## Bonnes pratiques

### Checklist de production

**Configuration Redlock :**
- ✅ Utiliser N=5 instances minimum (quorum = 3)
- ✅ Instances sur machines/datacenters différents
- ✅ Pas de réplication master-slave (indépendantes)
- ✅ Monitoring de toutes les instances

**TTL et Timing :**
- ✅ TTL > temps de traitement attendu × 2
- ✅ Clock drift factor configuré (1%)
- ✅ Retry avec exponential backoff
- ✅ Random jitter sur les retries

**Error Handling :**
- ✅ Toujours libérer le lock (finally/cleanup)
- ✅ Gérer les cas où le lock expire pendant le traitement
- ✅ Logging complet des acquisitions/releases
- ✅ Alertes si taux d'échec > seuil

**Testing :**
- ✅ Tester avec failures d'instances
- ✅ Tester avec clock drift simulé
- ✅ Tester avec network partitions
- ✅ Tester les cas de contention (concurrent access)

### Anti-patterns à éviter

```python
# ❌ BAD: TTL trop court
lock = Redlock(instances, "job", ttl=1000)  # 1 seconde
with lock:
    time.sleep(5)  # Processing prend 5 secondes
    # Lock expire avant la fin!

# ✅ GOOD: TTL approprié
lock = Redlock(instances, "job", ttl=10000)  # 10 secondes
with lock:
    time.sleep(5)  # OK, marge de sécurité


# ❌ BAD: Pas de cleanup en cas d'erreur
lock = Redlock(instances, "resource", ttl=5000)
lock.acquire()
process()  # Si erreur ici, lock jamais libéré!
lock.release()

# ✅ GOOD: Toujours cleanup
lock = Redlock(instances, "resource", ttl=5000)
try:
    lock.acquire()
    process()
finally:
    lock.release()


# ❌ BAD: Lock sur clé trop large
lock = Redlock(instances, "users", ttl=5000)  # Tous les users!
with lock:
    update_user(user_id)  # Bloque TOUS les users

# ✅ GOOD: Lock granulaire
lock = Redlock(instances, f"user:{user_id}", ttl=5000)
with lock:
    update_user(user_id)  # Bloque seulement ce user
```

---

## Monitoring et métriques

### Métriques importantes

```python
from prometheus_client import Counter, Histogram, Gauge

# Métriques Redlock
redlock_acquisitions = Counter(
    'redlock_acquisitions_total',
    'Total lock acquisitions',
    ['status']  # 'success', 'failure'
)

redlock_duration = Histogram(
    'redlock_acquisition_duration_seconds',
    'Time to acquire lock'
)

redlock_validity = Histogram(
    'redlock_validity_time_seconds',
    'Remaining validity time after acquisition'
)

redlock_releases = Counter(
    'redlock_releases_total',
    'Total lock releases'
)

redlock_active_locks = Gauge(
    'redlock_active_locks',
    'Number of currently held locks'
)


class MonitoredRedlock(Redlock):
    """
    Redlock avec monitoring Prometheus
    """

    def acquire(self):
        start = time.time()

        success = super().acquire()

        duration = time.time() - start
        redlock_duration.observe(duration)

        if success:
            redlock_acquisitions.labels(status='success').inc()
            redlock_validity.observe(self.validity_time / 1000.0)
            redlock_active_locks.inc()
        else:
            redlock_acquisitions.labels(status='failure').inc()

        return success

    def release(self):
        super().release()
        redlock_releases.inc()
        redlock_active_locks.dec()
```

---

## Conclusion

Le **Distributed Locking** est essentiel pour coordonner des applications distribuées. Les points clés :

**Simple Redis Lock :**
- ✅ Facile à implémenter
- ✅ Excellent pour cas non-critiques
- ❌ Pas de garanties fortes
- ❌ Single point of failure

**Redlock :**
- ✅ Meilleures garanties que simple lock
- ✅ Haute disponibilité (quorum)
- ✅ Bon compromis performance/fiabilité
- ❌ Configuration plus complexe
- ❌ Pas de garanties absolues

**ZooKeeper/etcd :**
- ✅ Garanties fortes (consensus)
- ✅ Coordination avancée
- ❌ Plus complexe et plus lent
- ❌ Overhead important

**Recommandations :**
1. Commencer avec Simple Lock si possible
2. Passer à Redlock pour plus de fiabilité
3. Utiliser ZooKeeper/etcd si garanties critiques
4. Toujours tester avec failures
5. Monitoring obligatoire en production

Le choix dépend de votre cas d'usage : privilégier la simplicité tant que les garanties ne sont pas critiques, puis monter en complexité si nécessaire.

---


⏭️ [Rate Limiting : Fixed Window, Sliding Window, Token Bucket](/06-patterns-developpement-avances/06-rate-limiting-patterns.md)
