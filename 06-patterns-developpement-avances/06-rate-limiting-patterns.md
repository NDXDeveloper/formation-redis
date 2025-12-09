🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6 Rate Limiting : Fixed Window, Sliding Window, Token Bucket

## Introduction

Le **Rate Limiting** est une technique essentielle pour protéger vos services contre les abus, garantir une qualité de service équitable entre utilisateurs, et respecter les quotas d'APIs externes. Redis, avec ses opérations atomiques et son support natif de TTL, est parfaitement adapté pour implémenter différents algorithmes de rate limiting.

Cette section explore les principaux algorithmes : Fixed Window, Sliding Window (Log et Counter), et Token Bucket, avec leurs avantages, inconvénients et implémentations production-ready.

## Pourquoi le Rate Limiting ?

### Cas d'usage typiques

**1. Protection contre les abus (DDoS, bruteforce)**

```text
Scénario : Attaque bruteforce sur login

Sans rate limiting:
- Attacker tente 10,000 passwords/seconde
- Serveur surchargé
- Service indisponible pour tous ❌

Avec rate limiting (5 tentatives/minute):
- Attacker limité à 5 tentatives
- Serveur protégé
- Service disponible ✅
```

**2. Fairness entre utilisateurs (Quality of Service)**

```text
Scénario : API partagée entre clients

Sans rate limiting:
- Client A fait 10,000 req/s
- Client B ne peut plus accéder
- Client C attend ❌

Avec rate limiting (100 req/s par client):
- Chaque client a son quota
- Ressources partagées équitablement
- Tous les clients satisfaits ✅
```

**3. Respect des quotas APIs externes**

```text
Scénario : Appel d'une API tierce (ex: Twitter API)

Twitter API limit: 300 req/15min

Sans rate limiting interne:
- Application fait 500 requêtes
- Twitter bloque l'application ❌
- Service indisponible 15 minutes

Avec rate limiting (290 req/15min):
- Application respecte la limite
- Pas de blocage
- Service stable ✅
```

**4. Contrôle des coûts**

```text
Scénario : Service cloud avec pricing par requête

Sans rate limiting:
- Bug génère 1M requêtes
- Facture = $10,000 ❌

Avec rate limiting (10K req/hour):
- Maximum 10K requêtes
- Facture contrôlée ✅
```

---

## Pattern 1: Fixed Window Counter

### Principe

Le **Fixed Window** divise le temps en fenêtres fixes (ex: 1 minute) et compte les requêtes dans chaque fenêtre.

```text
Timeline:
─────────────────────────────────────────────────────────────
         Window 1            Window 2            Window 3
    ├──────────────────┤├──────────────────┤├──────────────
    00:00          00:60 01:00          02:00 02:00

Limit: 10 requests per window

Window 1:  [★★★★★★★★]     8 requests  ✅ OK
Window 2:  [★★★★★★★★★★★★] 12 requests ❌ 2 rejected
Window 3:  [★★★]           3 requests  ✅ OK
```

### Avantages et inconvénients

**Avantages :**
- ✅ Très simple à implémenter
- ✅ Très rapide (O(1))
- ✅ Peu de mémoire (une clé par fenêtre)
- ✅ Compatible avec TTL Redis

**Inconvénients :**
- ❌ Burst à la frontière des fenêtres
- ❌ Peut permettre 2× la limite

### Problème du burst

```text
PROBLÈME : Burst à la frontière

Limit: 10 req/minute

Timeline:
─────────────────────────────────────────────────────────────
           Window 1                 Window 2
      ├──────────────────────┤├──────────────────────┤
      00:00            00:59  01:00            01:59

t=00:59  [★★★★★★★★★★] 10 requests (window 1) ✅
         ↓ Window change
t=01:00  [★★★★★★★★★★] 10 requests (window 2) ✅

Result: 20 requests in 1 second! (10 at 00:59 + 10 at 01:00)
Expected: Maximum 10 requests per minute
```

### Implémentation Python

```python
import redis
import time
from datetime import datetime

class FixedWindowRateLimiter:
    """
    Rate limiter avec Fixed Window Counter
    """

    def __init__(self, redis_client, max_requests=100, window_seconds=60):
        """
        Args:
            redis_client: Client Redis
            max_requests: Nombre max de requêtes par fenêtre
            window_seconds: Durée de la fenêtre en secondes
        """
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds

    def allow_request(self, user_id):
        """
        Vérifie si la requête est autorisée

        Returns:
            tuple: (allowed: bool, remaining: int, reset_time: int)
        """
        # Calculer la fenêtre actuelle
        current_window = int(time.time() / self.window_seconds)
        key = f"rate_limit:fixed:{user_id}:{current_window}"

        # Obtenir le compteur actuel
        current_count = self.redis.get(key)
        current_count = int(current_count) if current_count else 0

        # Vérifier la limite
        if current_count >= self.max_requests:
            # Calculer le temps de reset
            reset_time = (current_window + 1) * self.window_seconds
            remaining = 0

            print(f"❌ Rate limit exceeded for {user_id}")
            print(f"   Current: {current_count}/{self.max_requests}")
            print(f"   Reset in: {reset_time - int(time.time())}s")

            return False, remaining, reset_time

        # Incrémenter le compteur
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, self.window_seconds * 2)  # Safety margin
        results = pipe.execute()

        new_count = results[0]
        remaining = self.max_requests - new_count
        reset_time = (current_window + 1) * self.window_seconds

        print(f"✓ Request allowed for {user_id}")
        print(f"   Used: {new_count}/{self.max_requests}")
        print(f"   Remaining: {remaining}")

        return True, remaining, reset_time


# ============================================================
# EXEMPLE D'UTILISATION
# ============================================================

def test_fixed_window():
    """Test du Fixed Window rate limiter"""
    redis_client = redis.Redis(decode_responses=True)

    # Limiter à 5 requêtes par 10 secondes
    limiter = FixedWindowRateLimiter(
        redis_client,
        max_requests=5,
        window_seconds=10
    )

    print("=" * 60)
    print("FIXED WINDOW RATE LIMITER TEST")
    print("=" * 60 + "\n")

    user_id = "user:123"

    # Tester 7 requêtes
    for i in range(7):
        print(f"\nRequest {i + 1}:")
        allowed, remaining, reset_time = limiter.allow_request(user_id)

        if not allowed:
            print(f"Blocked! Wait {reset_time - int(time.time())}s")

        time.sleep(0.5)


# ============================================================
# IMPLÉMENTATION AVEC LUA SCRIPT (Plus performant)
# ============================================================

class OptimizedFixedWindowRateLimiter:
    """
    Version optimisée avec Lua script atomique
    """

    LUA_SCRIPT = """
    local key = KEYS[1]
    local max_requests = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])

    local current = redis.call('GET', key)

    if current and tonumber(current) >= max_requests then
        return {0, tonumber(current), 0}
    end

    local count = redis.call('INCR', key)

    if count == 1 then
        redis.call('EXPIRE', key, window * 2)
    end

    return {1, count, max_requests - count}
    """

    def __init__(self, redis_client, max_requests=100, window_seconds=60):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds

        # Enregistrer le script Lua
        self.script_sha = self.redis.script_load(self.LUA_SCRIPT)

    def allow_request(self, user_id):
        """Vérifie si la requête est autorisée (version atomique)"""
        current_window = int(time.time() / self.window_seconds)
        key = f"rate_limit:fixed:{user_id}:{current_window}"

        # Exécuter le script Lua
        result = self.redis.evalsha(
            self.script_sha,
            1,
            key,
            self.max_requests,
            self.window_seconds
        )

        allowed = bool(result[0])
        current_count = result[1]
        remaining = result[2]

        reset_time = (current_window + 1) * self.window_seconds

        return allowed, remaining, reset_time


# ============================================================
# DECORATOR POUR FLASK/FASTAPI
# ============================================================

from functools import wraps
from flask import request, jsonify

def rate_limit(max_requests=100, window_seconds=60):
    """
    Decorator pour rate limiting sur endpoints Flask
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            redis_client = redis.Redis(decode_responses=True)
            limiter = FixedWindowRateLimiter(
                redis_client,
                max_requests=max_requests,
                window_seconds=window_seconds
            )

            # Identifier l'utilisateur (IP ou user_id)
            user_id = request.remote_addr

            allowed, remaining, reset_time = limiter.allow_request(user_id)

            if not allowed:
                return jsonify({
                    'error': 'Rate limit exceeded',
                    'retry_after': reset_time - int(time.time())
                }), 429

            # Ajouter les headers de rate limiting
            response = func(*args, **kwargs)
            if isinstance(response, tuple):
                response, status_code = response
            else:
                status_code = 200

            headers = {
                'X-RateLimit-Limit': str(max_requests),
                'X-RateLimit-Remaining': str(remaining),
                'X-RateLimit-Reset': str(reset_time)
            }

            return response, status_code, headers

        return wrapper
    return decorator


# Utilisation avec Flask
# @app.route('/api/data')
# @rate_limit(max_requests=100, window_seconds=60)
# def get_data():
#     return {'data': 'some data'}
```

---

## Pattern 2: Sliding Window Log

### Principe

Le **Sliding Window Log** enregistre le timestamp de chaque requête et compte combien de requêtes sont dans la fenêtre glissante actuelle.

```text
Timeline (limit: 10 req/minute):
─────────────────────────────────────────────────────────────

Current time: 12:01:30

Sliding Window = [12:00:30 - 12:01:30] (last 60 seconds)

Request log:
  12:00:15  ← Outside window (removed)
  12:00:45  ★ In window
  12:00:50  ★ In window
  12:01:00  ★ In window
  12:01:10  ★ In window
  12:01:20  ★ In window
  12:01:25  ★ In window
             │
             └─ Count: 6 requests ✅ Allow

New request at 12:01:30:
  Add timestamp to log
  Remove timestamps < 12:00:30
  Count = 7 ✅ Still OK
```

### Avantages et inconvénients

**Avantages :**
- ✅ Précision parfaite (vraie fenêtre glissante)
- ✅ Pas de burst à la frontière
- ✅ Limite strictement respectée

**Inconvénients :**
- ❌ Beaucoup de mémoire (stocke tous les timestamps)
- ❌ Performance O(N) où N = nombre de requêtes
- ❌ Cleanup nécessaire des vieux timestamps

### Implémentation Python

```python
import redis
import time

class SlidingWindowLogRateLimiter:
    """
    Rate limiter avec Sliding Window Log
    Utilise un Sorted Set Redis pour stocker les timestamps
    """

    def __init__(self, redis_client, max_requests=100, window_seconds=60):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds

    def allow_request(self, user_id):
        """
        Vérifie si la requête est autorisée

        Returns:
            tuple: (allowed: bool, remaining: int)
        """
        key = f"rate_limit:sliding_log:{user_id}"
        now = time.time()
        window_start = now - self.window_seconds

        # Utiliser une transaction pour atomicité
        pipe = self.redis.pipeline()

        # 1. Supprimer les entrées expirées (avant la fenêtre)
        pipe.zremrangebyscore(key, 0, window_start)

        # 2. Compter les requêtes dans la fenêtre
        pipe.zcard(key)

        # 3. Ajouter la requête actuelle (on l'ajoute avant de vérifier)
        pipe.zadd(key, {str(now): now})

        # 4. Définir expiration de la clé
        pipe.expire(key, self.window_seconds)

        results = pipe.execute()

        # Le compte est avant l'ajout de la requête actuelle
        current_count = results[1]

        if current_count >= self.max_requests:
            # Supprimer la requête qu'on vient d'ajouter
            self.redis.zrem(key, str(now))

            remaining = 0
            print(f"❌ Rate limit exceeded for {user_id}")
            print(f"   Current: {current_count}/{self.max_requests}")

            return False, remaining

        remaining = self.max_requests - current_count - 1

        print(f"✓ Request allowed for {user_id}")
        print(f"   Used: {current_count + 1}/{self.max_requests}")
        print(f"   Remaining: {remaining}")

        return True, remaining


# ============================================================
# VERSION OPTIMISÉE AVEC LUA
# ============================================================

class OptimizedSlidingWindowLogRateLimiter:
    """
    Version optimisée avec Lua script
    """

    LUA_SCRIPT = """
    local key = KEYS[1]
    local now = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local max_requests = tonumber(ARGV[3])
    local request_id = ARGV[4]

    local window_start = now - window

    -- Supprimer les anciennes entrées
    redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

    -- Compter les requêtes dans la fenêtre
    local count = redis.call('ZCARD', key)

    if count >= max_requests then
        return {0, count, 0}
    end

    -- Ajouter la nouvelle requête
    redis.call('ZADD', key, now, request_id)
    redis.call('EXPIRE', key, window)

    return {1, count + 1, max_requests - count - 1}
    """

    def __init__(self, redis_client, max_requests=100, window_seconds=60):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.script_sha = self.redis.script_load(self.LUA_SCRIPT)

    def allow_request(self, user_id):
        """Version atomique avec Lua"""
        key = f"rate_limit:sliding_log:{user_id}"
        now = time.time()
        request_id = f"{now}:{id(self)}"

        result = self.redis.evalsha(
            self.script_sha,
            1,
            key,
            now,
            self.window_seconds,
            self.max_requests,
            request_id
        )

        allowed = bool(result[0])
        current_count = result[1]
        remaining = result[2]

        return allowed, remaining


# ============================================================
# TEST ET COMPARAISON
# ============================================================

def test_sliding_window_log():
    """Test du Sliding Window Log"""
    redis_client = redis.Redis(decode_responses=True)
    redis_client.flushdb()

    limiter = SlidingWindowLogRateLimiter(
        redis_client,
        max_requests=5,
        window_seconds=10
    )

    print("=" * 60)
    print("SLIDING WINDOW LOG TEST")
    print("=" * 60 + "\n")

    user_id = "user:456"

    # Test avec timestamps précis
    print("Making 5 requests at t=0:")
    for i in range(5):
        allowed, remaining = limiter.allow_request(user_id)
        print(f"  Request {i + 1}: {'✓' if allowed else '✗'} (remaining: {remaining})")

    print("\nRequest at t=0 (should fail):")
    allowed, remaining = limiter.allow_request(user_id)
    print(f"  {'✓' if allowed else '✗'} (remaining: {remaining})")

    print("\nWaiting 6 seconds...")
    time.sleep(6)

    print("\nRequest at t=6 (should succeed - window sliding):")
    allowed, remaining = limiter.allow_request(user_id)
    print(f"  {'✓' if allowed else '✗'} (remaining: {remaining})")
```

---

## Pattern 3: Sliding Window Counter

### Principe

Le **Sliding Window Counter** est un compromis entre Fixed Window et Sliding Log. Il utilise deux compteurs (fenêtre actuelle et précédente) et calcule une approximation de la fenêtre glissante.

```text
Timeline:
─────────────────────────────────────────────────────────────

Window size: 60 seconds
Current time: 12:01:30 (30 seconds into current window)

Previous Window [12:00:00 - 12:01:00]:  count = 8
Current Window  [12:01:00 - 12:02:00]:  count = 3

Calculation:
  30 seconds into current window = 50% progress

  Estimated count = current_count + (previous_count × overlap_percentage)
                  = 3 + (8 × 50%)
                  = 3 + 4
                  = 7 requests

If limit = 10: 7 < 10 ✅ Allow
```

### Formule

```text
estimated_count = current_window_count +
                  (previous_window_count × overlap_ratio)

overlap_ratio = (window_size - elapsed_time_in_current_window) / window_size

Example:
  window_size = 60s
  current_time = 12:01:45 (45s into current window)
  elapsed = 45s

  overlap_ratio = (60 - 45) / 60 = 15 / 60 = 0.25 (25%)

  If previous = 10, current = 2:
  estimated = 2 + (10 × 0.25) = 2 + 2.5 = 4.5 ≈ 5
```

### Avantages et inconvénients

**Avantages :**
- ✅ Bon compromis précision/performance
- ✅ Mémoire O(1) (seulement 2 compteurs)
- ✅ Performance O(1)
- ✅ Meilleur que Fixed Window pour les bursts

**Inconvénients :**
- ❌ Approximation (pas parfaitement précis)
- ❌ Peut permettre légèrement plus que la limite
- ❌ Plus complexe à comprendre

### Implémentation Python

```python
import redis
import time
import math

class SlidingWindowCounterRateLimiter:
    """
    Rate limiter avec Sliding Window Counter
    Utilise deux fenêtres (actuelle et précédente)
    """

    def __init__(self, redis_client, max_requests=100, window_seconds=60):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds

    def allow_request(self, user_id):
        """
        Vérifie si la requête est autorisée

        Returns:
            tuple: (allowed: bool, remaining: int)
        """
        now = time.time()
        current_window = int(now / self.window_seconds)
        previous_window = current_window - 1

        # Clés pour les deux fenêtres
        current_key = f"rate_limit:sliding_counter:{user_id}:{current_window}"
        previous_key = f"rate_limit:sliding_counter:{user_id}:{previous_window}"

        # Obtenir les compteurs
        pipe = self.redis.pipeline()
        pipe.get(current_key)
        pipe.get(previous_key)
        results = pipe.execute()

        current_count = int(results[0]) if results[0] else 0
        previous_count = int(results[1]) if results[1] else 0

        # Calculer le pourcentage d'overlap avec la fenêtre précédente
        elapsed_time_in_current_window = now - (current_window * self.window_seconds)
        overlap_ratio = (self.window_seconds - elapsed_time_in_current_window) / self.window_seconds

        # Estimation du nombre de requêtes dans la fenêtre glissante
        estimated_count = current_count + (previous_count * overlap_ratio)

        if estimated_count >= self.max_requests:
            remaining = 0
            print(f"❌ Rate limit exceeded for {user_id}")
            print(f"   Estimated: {estimated_count:.2f}/{self.max_requests}")
            print(f"   Current window: {current_count}")
            print(f"   Previous window: {previous_count} (×{overlap_ratio:.2%})")

            return False, remaining

        # Incrémenter le compteur de la fenêtre actuelle
        pipe = self.redis.pipeline()
        pipe.incr(current_key)
        pipe.expire(current_key, self.window_seconds * 2)
        pipe.execute()

        remaining = math.floor(self.max_requests - estimated_count - 1)

        print(f"✓ Request allowed for {user_id}")
        print(f"   Estimated: {estimated_count + 1:.2f}/{self.max_requests}")
        print(f"   Remaining: {remaining}")

        return True, remaining


# ============================================================
# VERSION OPTIMISÉE AVEC LUA
# ============================================================

class OptimizedSlidingWindowCounterRateLimiter:
    """
    Version optimisée avec Lua script
    """

    LUA_SCRIPT = """
    local current_key = KEYS[1]
    local previous_key = KEYS[2]
    local now = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local max_requests = tonumber(ARGV[3])
    local current_window_start = tonumber(ARGV[4])

    -- Obtenir les compteurs
    local current_count = tonumber(redis.call('GET', current_key) or 0)
    local previous_count = tonumber(redis.call('GET', previous_key) or 0)

    -- Calculer l'overlap ratio
    local elapsed = now - current_window_start
    local overlap_ratio = (window - elapsed) / window

    -- Estimation
    local estimated_count = current_count + (previous_count * overlap_ratio)

    if estimated_count >= max_requests then
        return {0, math.floor(estimated_count), 0}
    end

    -- Incrémenter
    redis.call('INCR', current_key)
    redis.call('EXPIRE', current_key, window * 2)

    local new_estimated = estimated_count + 1
    local remaining = math.floor(max_requests - new_estimated)

    return {1, math.floor(new_estimated), remaining}
    """

    def __init__(self, redis_client, max_requests=100, window_seconds=60):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.script_sha = self.redis.script_load(self.LUA_SCRIPT)

    def allow_request(self, user_id):
        """Version atomique avec Lua"""
        now = time.time()
        current_window = int(now / self.window_seconds)
        previous_window = current_window - 1
        current_window_start = current_window * self.window_seconds

        current_key = f"rate_limit:sliding_counter:{user_id}:{current_window}"
        previous_key = f"rate_limit:sliding_counter:{user_id}:{previous_window}"

        result = self.redis.evalsha(
            self.script_sha,
            2,
            current_key,
            previous_key,
            now,
            self.window_seconds,
            self.max_requests,
            current_window_start
        )

        allowed = bool(result[0])
        estimated_count = result[1]
        remaining = result[2]

        return allowed, remaining
```

---

## Pattern 4: Token Bucket

### Principe

Le **Token Bucket** est comme un seau avec des jetons. Les jetons sont ajoutés à un taux constant, et chaque requête consomme un jeton. Si le seau est vide, la requête est refusée.

```text
┌─────────────────────────────────────────────────────────────┐
│                    TOKEN BUCKET                             │
└─────────────────────────────────────────────────────────────┘

Bucket capacity: 10 tokens
Refill rate: 1 token/second

State at t=0:  [🪙🪙🪙🪙🪙🪙🪙🪙🪙🪙] 10 tokens

t=0   Request → take 1 token → [🪙🪙🪙🪙🪙🪙🪙🪙🪙] 9 tokens ✅
t=1   Request → take 1 token → [🪙🪙🪙🪙🪙🪙🪙🪙] 8 tokens ✅
      + Refill 1 token       → [🪙🪙🪙🪙🪙🪙🪙🪙🪙] 9 tokens

t=2   Request → take 1 token → [🪙🪙🪙🪙🪙🪙🪙🪙] 8 tokens ✅
      + Refill 1 token       → [🪙🪙🪙🪙🪙🪙🪙🪙🪙] 9 tokens

Burst of 10 requests at t=3:
  Take 9 tokens              → [⚪] 0 tokens
  10th request fails ❌

Wait 5 seconds...
  + Refill 5 tokens          → [🪙🪙🪙🪙🪙] 5 tokens

New request at t=8           → [🪙🪙🪙🪙] 4 tokens ✅
```

### Algorithme

```text
ALGORITHM: Token Bucket

State:
- tokens: current number of tokens in bucket
- last_refill: timestamp of last refill
- capacity: maximum tokens (bucket size)
- refill_rate: tokens added per second

On each request:
1. Calculate elapsed time since last refill
   elapsed = now - last_refill

2. Calculate tokens to add
   tokens_to_add = elapsed × refill_rate

3. Refill bucket (up to capacity)
   tokens = min(tokens + tokens_to_add, capacity)
   last_refill = now

4. Check if enough tokens
   if tokens >= 1:
       tokens = tokens - 1
       return ALLOW
   else:
       return DENY
```

### Avantages et inconvénients

**Avantages :**
- ✅ Permet les bursts contrôlés (jusqu'à capacity)
- ✅ Lissage du trafic
- ✅ Mémoire O(1)
- ✅ Flexible (différents rates pour différentes actions)

**Inconvénients :**
- ❌ Plus complexe à implémenter
- ❌ Nécessite calcul à chaque requête
- ❌ Pas de fenêtre temporelle fixe

### Implémentation Python

```python
import redis
import time
import json

class TokenBucketRateLimiter:
    """
    Rate limiter avec Token Bucket algorithm
    """

    def __init__(self, redis_client, capacity=100, refill_rate=10):
        """
        Args:
            redis_client: Client Redis
            capacity: Taille du bucket (nombre max de tokens)
            refill_rate: Nombre de tokens ajoutés par seconde
        """
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate

    def allow_request(self, user_id, tokens_required=1):
        """
        Vérifie si la requête est autorisée

        Args:
            user_id: ID de l'utilisateur
            tokens_required: Nombre de tokens requis (par défaut 1)

        Returns:
            tuple: (allowed: bool, tokens_available: float)
        """
        key = f"rate_limit:token_bucket:{user_id}"
        now = time.time()

        # Obtenir l'état actuel du bucket
        bucket_data = self.redis.get(key)

        if bucket_data:
            bucket = json.loads(bucket_data)
            tokens = bucket['tokens']
            last_refill = bucket['last_refill']
        else:
            # Premier accès : bucket plein
            tokens = self.capacity
            last_refill = now

        # Calculer les tokens à ajouter depuis le dernier refill
        elapsed = now - last_refill
        tokens_to_add = elapsed * self.refill_rate

        # Refill le bucket (sans dépasser capacity)
        tokens = min(tokens + tokens_to_add, self.capacity)

        # Vérifier si assez de tokens
        if tokens >= tokens_required:
            # Consommer les tokens
            tokens -= tokens_required

            # Sauvegarder le nouvel état
            new_bucket = {
                'tokens': tokens,
                'last_refill': now
            }
            self.redis.setex(
                key,
                int(self.capacity / self.refill_rate) * 2,  # TTL
                json.dumps(new_bucket)
            )

            print(f"✓ Request allowed for {user_id}")
            print(f"   Tokens used: {tokens_required}")
            print(f"   Tokens remaining: {tokens:.2f}/{self.capacity}")

            return True, tokens
        else:
            # Pas assez de tokens
            # On met à jour quand même pour le refill
            new_bucket = {
                'tokens': tokens,
                'last_refill': now
            }
            self.redis.setex(
                key,
                int(self.capacity / self.refill_rate) * 2,
                json.dumps(new_bucket)
            )

            print(f"❌ Rate limit exceeded for {user_id}")
            print(f"   Tokens needed: {tokens_required}")
            print(f"   Tokens available: {tokens:.2f}/{self.capacity}")

            # Calculer le temps d'attente
            tokens_needed = tokens_required - tokens
            wait_time = tokens_needed / self.refill_rate
            print(f"   Wait time: {wait_time:.1f}s")

            return False, tokens


# ============================================================
# VERSION OPTIMISÉE AVEC LUA
# ============================================================

class OptimizedTokenBucketRateLimiter:
    """
    Token Bucket avec Lua script pour atomicité
    """

    LUA_SCRIPT = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])
    local tokens_required = tonumber(ARGV[3])
    local now = tonumber(ARGV[4])
    local ttl = tonumber(ARGV[5])

    -- Obtenir l'état actuel
    local bucket_data = redis.call('GET', key)
    local tokens, last_refill

    if bucket_data then
        local bucket = cjson.decode(bucket_data)
        tokens = tonumber(bucket.tokens)
        last_refill = tonumber(bucket.last_refill)
    else
        tokens = capacity
        last_refill = now
    end

    -- Refill
    local elapsed = now - last_refill
    local tokens_to_add = elapsed * refill_rate
    tokens = math.min(tokens + tokens_to_add, capacity)

    -- Check et consommation
    if tokens >= tokens_required then
        tokens = tokens - tokens_required

        local new_bucket = {
            tokens = tokens,
            last_refill = now
        }

        redis.call('SETEX', key, ttl, cjson.encode(new_bucket))

        return {1, tokens}
    else
        local new_bucket = {
            tokens = tokens,
            last_refill = now
        }

        redis.call('SETEX', key, ttl, cjson.encode(new_bucket))

        return {0, tokens}
    end
    """

    def __init__(self, redis_client, capacity=100, refill_rate=10):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.script_sha = self.redis.script_load(self.LUA_SCRIPT)

    def allow_request(self, user_id, tokens_required=1):
        """Version atomique avec Lua"""
        key = f"rate_limit:token_bucket:{user_id}"
        now = time.time()
        ttl = int(self.capacity / self.refill_rate) * 2

        result = self.redis.evalsha(
            self.script_sha,
            1,
            key,
            self.capacity,
            self.refill_rate,
            tokens_required,
            now,
            ttl
        )

        allowed = bool(result[0])
        tokens_available = result[1]

        return allowed, tokens_available


# ============================================================
# TOKEN BUCKET AVEC COÛTS VARIABLES
# ============================================================

class WeightedTokenBucketRateLimiter:
    """
    Token Bucket avec coûts différents par opération
    """

    def __init__(self, redis_client, capacity=1000, refill_rate=10):
        self.limiter = OptimizedTokenBucketRateLimiter(
            redis_client,
            capacity=capacity,
            refill_rate=refill_rate
        )

        # Coûts par type d'opération
        self.costs = {
            'read': 1,          # Lecture simple
            'write': 5,         # Écriture
            'search': 10,       # Recherche complexe
            'report': 50,       # Génération de rapport
            'export': 100,      # Export massif
        }

    def allow_operation(self, user_id, operation_type):
        """
        Vérifie si l'opération est autorisée

        Args:
            user_id: ID de l'utilisateur
            operation_type: Type d'opération ('read', 'write', etc.)
        """
        cost = self.costs.get(operation_type, 1)

        allowed, tokens = self.limiter.allow_request(user_id, tokens_required=cost)

        if allowed:
            print(f"✓ {operation_type} operation allowed (cost: {cost} tokens)")
        else:
            print(f"❌ {operation_type} operation denied (cost: {cost} tokens)")

        return allowed, tokens


# ============================================================
# TEST
# ============================================================

def test_token_bucket():
    """Test du Token Bucket"""
    redis_client = redis.Redis(decode_responses=True)
    redis_client.flushdb()

    # Bucket avec 10 tokens, refill 2 tokens/seconde
    limiter = TokenBucketRateLimiter(
        redis_client,
        capacity=10,
        refill_rate=2
    )

    print("=" * 60)
    print("TOKEN BUCKET RATE LIMITER TEST")
    print("=" * 60 + "\n")

    user_id = "user:789"

    # Test burst
    print("Testing burst (10 requests immediately):")
    for i in range(12):
        allowed, tokens = limiter.allow_request(user_id)
        print(f"  Request {i + 1}: {'✓' if allowed else '✗'} ({tokens:.2f} tokens left)")

    print("\nWaiting 5 seconds for refill...")
    time.sleep(5)

    print("\nAfter 5 seconds (should have ~10 tokens):")
    allowed, tokens = limiter.allow_request(user_id)
    print(f"  Request: {'✓' if allowed else '✗'} ({tokens:.2f} tokens left)")


def test_weighted_token_bucket():
    """Test avec coûts variables"""
    redis_client = redis.Redis(decode_responses=True)
    redis_client.flushdb()

    limiter = WeightedTokenBucketRateLimiter(
        redis_client,
        capacity=100,
        refill_rate=10
    )

    print("\n" + "=" * 60)
    print("WEIGHTED TOKEN BUCKET TEST")
    print("=" * 60 + "\n")

    user_id = "user:premium"

    # Différentes opérations
    operations = [
        'read', 'read', 'write', 'search', 'read',
        'write', 'report', 'export', 'read'
    ]

    for op in operations:
        allowed, tokens = limiter.allow_operation(user_id, op)
        print(f"  Remaining tokens: {tokens:.2f}/100\n")
        time.sleep(0.5)
```

---

## Implémentation Node.js

### Fixed Window

```javascript
const Redis = require('ioredis');

class FixedWindowRateLimiter {
    constructor(redis, maxRequests = 100, windowSeconds = 60) {
        this.redis = redis;
        this.maxRequests = maxRequests;
        this.windowSeconds = windowSeconds;
    }

    async allowRequest(userId) {
        const currentWindow = Math.floor(Date.now() / 1000 / this.windowSeconds);
        const key = `rate_limit:fixed:${userId}:${currentWindow}`;

        const currentCount = await this.redis.get(key) || 0;

        if (currentCount >= this.maxRequests) {
            const resetTime = (currentWindow + 1) * this.windowSeconds;
            return {
                allowed: false,
                remaining: 0,
                resetTime: resetTime
            };
        }

        const pipeline = this.redis.pipeline();
        pipeline.incr(key);
        pipeline.expire(key, this.windowSeconds * 2);
        const results = await pipeline.exec();

        const newCount = results[0][1];
        const remaining = this.maxRequests - newCount;
        const resetTime = (currentWindow + 1) * this.windowSeconds;

        return {
            allowed: true,
            remaining: remaining,
            resetTime: resetTime,
            current: newCount
        };
    }
}

// Usage
async function testFixedWindow() {
    const redis = new Redis();
    const limiter = new FixedWindowRateLimiter(redis, 5, 10);

    console.log('='.repeat(60));
    console.log('FIXED WINDOW TEST');
    console.log('='.repeat(60) + '\n');

    for (let i = 0; i < 7; i++) {
        const result = await limiter.allowRequest('user:123');
        console.log(`Request ${i + 1}: ${result.allowed ? '✓' : '✗'} (${result.remaining} remaining)`);
        await new Promise(resolve => setTimeout(resolve, 500));
    }

    await redis.quit();
}

testFixedWindow();
```

### Sliding Window Log

```javascript
class SlidingWindowLogRateLimiter {
    constructor(redis, maxRequests = 100, windowSeconds = 60) {
        this.redis = redis;
        this.maxRequests = maxRequests;
        this.windowSeconds = windowSeconds;

        // Lua script for atomic operations
        this.luaScript = `
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window = tonumber(ARGV[2])
            local max_requests = tonumber(ARGV[3])
            local request_id = ARGV[4]

            local window_start = now - window

            redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

            local count = redis.call('ZCARD', key)

            if count >= max_requests then
                return {0, count, 0}
            end

            redis.call('ZADD', key, now, request_id)
            redis.call('EXPIRE', key, window)

            return {1, count + 1, max_requests - count - 1}
        `;
    }

    async allowRequest(userId) {
        const key = `rate_limit:sliding_log:${userId}`;
        const now = Date.now() / 1000;
        const requestId = `${now}:${Math.random()}`;

        const result = await this.redis.eval(
            this.luaScript,
            1,
            key,
            now,
            this.windowSeconds,
            this.maxRequests,
            requestId
        );

        return {
            allowed: Boolean(result[0]),
            current: result[1],
            remaining: result[2]
        };
    }
}
```

### Token Bucket

```javascript
class TokenBucketRateLimiter {
    constructor(redis, capacity = 100, refillRate = 10) {
        this.redis = redis;
        this.capacity = capacity;
        this.refillRate = refillRate;

        this.luaScript = `
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local tokens_required = tonumber(ARGV[3])
            local now = tonumber(ARGV[4])
            local ttl = tonumber(ARGV[5])

            local bucket_data = redis.call('GET', key)
            local tokens, last_refill

            if bucket_data then
                local bucket = cjson.decode(bucket_data)
                tokens = tonumber(bucket.tokens)
                last_refill = tonumber(bucket.last_refill)
            else
                tokens = capacity
                last_refill = now
            end

            local elapsed = now - last_refill
            local tokens_to_add = elapsed * refill_rate
            tokens = math.min(tokens + tokens_to_add, capacity)

            if tokens >= tokens_required then
                tokens = tokens - tokens_required

                local new_bucket = {
                    tokens = tokens,
                    last_refill = now
                }

                redis.call('SETEX', key, ttl, cjson.encode(new_bucket))

                return {1, tokens}
            else
                local new_bucket = {
                    tokens = tokens,
                    last_refill = now
                }

                redis.call('SETEX', key, ttl, cjson.encode(new_bucket))

                return {0, tokens}
            end
        `;
    }

    async allowRequest(userId, tokensRequired = 1) {
        const key = `rate_limit:token_bucket:${userId}`;
        const now = Date.now() / 1000;
        const ttl = Math.floor(this.capacity / this.refillRate) * 2;

        const result = await this.redis.eval(
            this.luaScript,
            1,
            key,
            this.capacity,
            this.refillRate,
            tokensRequired,
            now,
            ttl
        );

        return {
            allowed: Boolean(result[0]),
            tokensAvailable: result[1]
        };
    }
}

// Express middleware
function createRateLimitMiddleware(limiter) {
    return async (req, res, next) => {
        const userId = req.user?.id || req.ip;

        const result = await limiter.allowRequest(userId);

        // Add headers
        res.setHeader('X-RateLimit-Limit', limiter.maxRequests || limiter.capacity);
        res.setHeader('X-RateLimit-Remaining', result.remaining || Math.floor(result.tokensAvailable));

        if (!result.allowed) {
            res.setHeader('Retry-After', 60);
            return res.status(429).json({
                error: 'Too Many Requests',
                message: 'Rate limit exceeded. Please try again later.'
            });
        }

        next();
    };
}

// Usage with Express
// const limiter = new TokenBucketRateLimiter(redis, 100, 10);
// app.use('/api', createRateLimitMiddleware(limiter));
```

---

## Comparaison des algorithmes

### Tableau comparatif

| Algorithme | Mémoire | Performance | Précision | Bursts | Complexité |
|------------|---------|-------------|-----------|--------|------------|
| **Fixed Window** | ⭐⭐⭐⭐⭐ O(1) | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐ Faible | ❌ Permet 2× | ⭐⭐⭐⭐⭐ Simple |
| **Sliding Log** | ⭐⭐ O(N) | ⭐⭐⭐ Moyen | ⭐⭐⭐⭐⭐ Parfait | ✅ Contrôlé | ⭐⭐ Complexe |
| **Sliding Counter** | ⭐⭐⭐⭐ O(1) | ⭐⭐⭐⭐ Bon | ⭐⭐⭐⭐ Bon | ✅ Meilleur | ⭐⭐⭐ Moyen |
| **Token Bucket** | ⭐⭐⭐⭐ O(1) | ⭐⭐⭐⭐ Bon | ⭐⭐⭐⭐ Bon | ✅ Flexible | ⭐⭐⭐ Moyen |

### Visualisation des bursts

```text
Limit: 10 req/min

FIXED WINDOW:
──────────────────────────────────────────────────────────
Window 1     │ Window 2
00:59 [★★★★★★★★★★] 10 req ✅
01:00 [★★★★★★★★★★] 10 req ✅
Result: 20 req in 1 second! ❌

SLIDING WINDOW LOG:
──────────────────────────────────────────────────────────
Any 60-second window: MAX 10 requests
00:59 to 01:59: [★★★★★★★★★★] EXACTLY 10 ✅

SLIDING WINDOW COUNTER:
──────────────────────────────────────────────────────────
Estimation based on 2 windows
Slightly more accurate than Fixed Window
May allow ~11-12 req (approximation) ⚠️

TOKEN BUCKET:
──────────────────────────────────────────────────────────
Burst of 10 initially: [★★★★★★★★★★] ✅
Then 1 token/6s refill
Smooth traffic over time ✅
```

### Quand utiliser chaque algorithme ?

```python
# ✅ Fixed Window
"""
Bon pour:
- APIs publiques simples
- Protection basique
- Haute performance requise
- Mémoire limitée

Cas d'usage:
- Rate limiting général
- Protection anti-spam basique
- Quotas journaliers
"""

# ✅ Sliding Window Log
"""
Bon pour:
- Limites strictes requises
- Pas de tolérance aux bursts
- Précision critique

Cas d'usage:
- APIs de paiement
- Opérations critiques
- Respect strict de quotas externes
"""

# ✅ Sliding Window Counter
"""
Bon pour:
- Compromis performance/précision
- Limites moyennement strictes
- Production standard

Cas d'usage:
- APIs d'entreprise
- Services SaaS
- Rate limiting général (recommandé)
"""

# ✅ Token Bucket
"""
Bon pour:
- Trafic variable
- Bursts contrôlés acceptables
- Coûts variables par opération

Cas d'usage:
- CDN
- File upload/download
- APIs avec opérations différentes
- Gaming (actions différentes)
"""
```

---

## Patterns avancés

### Multi-tier Rate Limiting

```python
class MultiTierRateLimiter:
    """
    Rate limiting à plusieurs niveaux
    """

    def __init__(self, redis_client):
        self.redis = redis_client

        # Différents tiers
        self.tiers = {
            'free': {
                'per_second': 1,
                'per_minute': 10,
                'per_hour': 100,
                'per_day': 1000
            },
            'basic': {
                'per_second': 5,
                'per_minute': 100,
                'per_hour': 1000,
                'per_day': 10000
            },
            'premium': {
                'per_second': 50,
                'per_minute': 1000,
                'per_hour': 10000,
                'per_day': 100000
            }
        }

    def allow_request(self, user_id, tier='free'):
        """
        Vérifie toutes les limites du tier
        """
        limits = self.tiers[tier]

        # Vérifier chaque limite
        checks = [
            ('per_second', FixedWindowRateLimiter(
                self.redis, limits['per_second'], 1)),
            ('per_minute', FixedWindowRateLimiter(
                self.redis, limits['per_minute'], 60)),
            ('per_hour', FixedWindowRateLimiter(
                self.redis, limits['per_hour'], 3600)),
            ('per_day', FixedWindowRateLimiter(
                self.redis, limits['per_day'], 86400)),
        ]

        for name, limiter in checks:
            allowed, remaining, reset_time = limiter.allow_request(user_id)

            if not allowed:
                print(f"❌ {tier.upper()} tier {name} limit exceeded")
                return False, name, remaining, reset_time

        print(f"✓ Request allowed for {tier.upper()} tier user")
        return True, None, None, None
```

### Adaptive Rate Limiting

```python
class AdaptiveRateLimiter:
    """
    Rate limiter adaptatif basé sur la charge système
    """

    def __init__(self, redis_client, base_limit=100):
        self.redis = redis_client
        self.base_limit = base_limit

    def get_current_load(self):
        """
        Obtient la charge système actuelle
        (Simplifié - en prod, utiliser vraies métriques)
        """
        # Simuler : 0.0 (pas de charge) à 1.0 (pleine charge)
        return 0.5  # 50% de charge

    def get_adjusted_limit(self):
        """
        Ajuste la limite en fonction de la charge
        """
        load = self.get_current_load()

        if load < 0.5:
            # Charge faible : autoriser plus
            multiplier = 1.5
        elif load < 0.8:
            # Charge normale
            multiplier = 1.0
        else:
            # Charge élevée : réduire
            multiplier = 0.5

        adjusted_limit = int(self.base_limit * multiplier)

        print(f"System load: {load*100:.0f}%")
        print(f"Adjusted limit: {adjusted_limit} req/min (base: {self.base_limit})")

        return adjusted_limit

    def allow_request(self, user_id):
        """Vérifie avec limite ajustée"""
        current_limit = self.get_adjusted_limit()

        limiter = FixedWindowRateLimiter(
            self.redis,
            max_requests=current_limit,
            window_seconds=60
        )

        return limiter.allow_request(user_id)
```

### Rate Limiting par endpoint

```python
class EndpointRateLimiter:
    """
    Rate limiter différent par endpoint
    """

    def __init__(self, redis_client):
        self.redis = redis_client

        # Configuration par endpoint
        self.limits = {
            '/api/search': {'limit': 10, 'window': 60},    # Expensive
            '/api/read': {'limit': 100, 'window': 60},     # Cheap
            '/api/write': {'limit': 50, 'window': 60},     # Medium
            '/api/export': {'limit': 5, 'window': 3600},   # Very expensive
        }

    def allow_request(self, user_id, endpoint):
        """Vérifie la limite pour l'endpoint spécifique"""
        config = self.limits.get(endpoint, {'limit': 60, 'window': 60})

        limiter = FixedWindowRateLimiter(
            self.redis,
            max_requests=config['limit'],
            window_seconds=config['window']
        )

        key = f"{user_id}:{endpoint}"
        return limiter.allow_request(key)
```

---

## Bonnes pratiques

### Checklist de production

**Configuration :**
- ✅ Choisir l'algorithme adapté au cas d'usage
- ✅ Définir des limites réalistes (tester en production)
- ✅ Différencier les tiers d'utilisateurs (free/premium)
- ✅ Rate limiting par endpoint (coûts différents)

**Headers HTTP :**
- ✅ `X-RateLimit-Limit`: Limite totale
- ✅ `X-RateLimit-Remaining`: Requêtes restantes
- ✅ `X-RateLimit-Reset`: Timestamp de reset
- ✅ `Retry-After`: Temps d'attente en secondes (429)

**Monitoring :**
- ✅ Tracker le nombre de requêtes bloquées
- ✅ Alerter si taux de blocage > 5%
- ✅ Mesurer l'impact sur la latence
- ✅ Logger les abus (potentiels attacks)

**Error Handling :**
- ✅ Retourner 429 (Too Many Requests)
- ✅ Message d'erreur clair
- ✅ Indiquer quand réessayer
- ✅ Ne pas révéler d'infos sensibles

**Testing :**
- ✅ Tester les limites exactes
- ✅ Tester les bursts
- ✅ Tester le comportement à la frontière
- ✅ Load testing avec rate limiter actif

### Anti-patterns à éviter

```python
# ❌ BAD: Limites trop restrictives
limiter = FixedWindowRateLimiter(redis_client, max_requests=1, window_seconds=3600)
# 1 requête par heure = service inutilisable

# ✅ GOOD: Limites raisonnables
limiter = FixedWindowRateLimiter(redis_client, max_requests=100, window_seconds=60)


# ❌ BAD: Pas de différenciation
# Tous les endpoints ont la même limite
# /api/read et /api/export coûtent pareil

# ✅ GOOD: Limites par endpoint
limits = {
    '/api/read': 1000,    # Cheap operation
    '/api/export': 10,    # Expensive operation
}


# ❌ BAD: Bloquer sans explication
if not limiter.allow_request(user_id):
    return 429  # Pas d'info pour l'utilisateur

# ✅ GOOD: Messages clairs
if not limiter.allow_request(user_id):
    return {
        'error': 'Rate limit exceeded',
        'limit': 100,
        'window': '60 seconds',
        'retry_after': 30
    }, 429


# ❌ BAD: Rate limiting uniquement côté client
# Client peut bypass facilement

# ✅ GOOD: Rate limiting côté serveur (obligatoire)
@app.route('/api/data')
@rate_limit(100, 60)
def get_data():
    return data
```

---

## Conclusion

Le **Rate Limiting** est essentiel pour protéger vos services et garantir une qualité de service équitable. Les points clés :

**Choix de l'algorithme :**
- **Fixed Window** : Simple, rapide, mais permet bursts
- **Sliding Window Log** : Précis, mais gourmand en mémoire
- **Sliding Window Counter** : Bon compromis (recommandé)
- **Token Bucket** : Flexible, bon pour trafic variable

**Recommandations :**
1. Commencer avec Fixed Window pour simplicité
2. Passer à Sliding Window Counter si bursts problématiques
3. Utiliser Token Bucket pour coûts variables
4. Sliding Window Log seulement si précision critique

**Best Practices :**
- Toujours implémenter côté serveur
- Utiliser des limites raisonnables
- Différencier par tier et endpoint
- Inclure les headers HTTP appropriés
- Monitoring obligatoire
- Messages d'erreur clairs

Le rate limiting n'est pas juste une protection technique, c'est aussi une question d'UX et de business model !

---


⏭️ [Session Store et gestion d'état](/06-patterns-developpement-avances/07-session-store-gestion-etat.md)
