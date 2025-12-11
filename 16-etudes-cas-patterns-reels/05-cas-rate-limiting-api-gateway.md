🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #5 : Rate Limiting pour API Gateway

## Vue d'ensemble

**Niveau** : ⭐⭐ Intermédiaire
**Complexité technique** : Moyenne
**Impact production** : Critique (protection infrastructure)
**Technologies** : Redis Core + Lua Scripting

---

## 1. Contexte et problématique

### Scénario business

Une API Gateway qui gère le trafic vers des microservices backend (type Kong, Nginx, ou AWS API Gateway) :

**Chiffres clés** :
- 10 000 clients (applications)
- 1 million de requêtes par seconde (peak)
- 50 endpoints différents
- SLA : 99.99% disponibilité
- Latence rate limiting : < 1ms (p99)
- Tiers gratuit : 1000 req/heure
- Tiers payant : 10 000 - 100 000 req/heure

**Besoins métier critiques** :

1. **Protection contre les abus**
   - Prévenir les attaques DDoS
   - Limiter les clients malveillants
   - Protéger les backends de surcharge
   - Fair usage entre clients

2. **Monétisation par quotas**
   - Plans tarifaires basés sur volume
   - Limite par client/API key
   - Limite par endpoint
   - Limite par IP (fallback)

3. **Flexibilité des stratégies**
   - Fixed Window (simple)
   - Sliding Window (précis)
   - Token Bucket (burst allowed)
   - Leaky Bucket (traffic smoothing)
   - Limites combinées (client + IP + endpoint)

4. **Distribution et cohérence**
   - Multiple instances d'API Gateway
   - Compteurs partagés entre instances
   - Pas de race conditions
   - Décisions en < 1ms

### Problèmes à résoudre

#### 1. **Race conditions sur compteurs**

```
Problème : 2 requêtes simultanées d'un client

Instance A                    Instance B
├─ GET rate_limit:user123    ├─ GET rate_limit:user123
│  → count = 999              │  → count = 999
├─ count + 1 = 1000           ├─ count + 1 = 1000
├─ SET rate_limit:user123     ├─ SET rate_limit:user123
│  → 1000                     │  → 1000
└─ Allow (sous limite)        └─ Allow (sous limite)

Résultat : 2 requêtes passent alors que limite = 1000
→ Off-by-one errors accumulés = dépassement significatif
```

#### 2. **Précision des fenêtres temporelles**

```
❌ Fixed Window (problème du burst)
Limite : 1000 req/heure

Window 1 (13:00-14:00) : 1000 requêtes à 13:59
Window 2 (14:00-15:00) : 1000 requêtes à 14:01

Résultat : 2000 requêtes en 2 minutes !
→ Burst autorisé = 2× la limite
```

#### 3. **Performance à grande échelle**

```
Contraintes :
- 1M req/sec à gérer
- Latence < 1ms par décision
- Pas de blocking (async)
- Minimal RTT vers Redis

Problème naïf (3 commandes) :
GET → INCR → EXPIRE = 3 RTT
→ 3ms latency (inacceptable)

Solution : 1 RTT avec Lua
→ < 1ms latency
```

#### 4. **Nettoyage des compteurs expirés**

```
Problème : Millions de clés créées par jour
- 1M clients × 24 heures = 24M keys/jour
- Sans cleanup → Memory leak
- Avec TTL → Automatic cleanup (Redis native)
```

---

## 2. Analyse des alternatives

### Option 1 : In-Memory (Application-Level)

```python
# Rate limiter in-memory (par instance)
rate_limits = {}  # {user_id: {"count": X, "reset_at": timestamp}}

def check_rate_limit(user_id):
    now = time.time()

    if user_id not in rate_limits:
        rate_limits[user_id] = {"count": 1, "reset_at": now + 3600}
        return True

    limit_data = rate_limits[user_id]

    if now > limit_data["reset_at"]:
        # Reset window
        rate_limits[user_id] = {"count": 1, "reset_at": now + 3600}
        return True

    if limit_data["count"] >= 1000:
        return False  # Rate limited

    limit_data["count"] += 1
    return True
```

**Avantages** :
- ✅ Latence ultra-faible (< 0.1ms)
- ✅ Pas de dépendance externe
- ✅ Simple à implémenter

**Inconvénients** :
- ❌ **Pas de synchronisation entre instances** (critical flaw)
- ❌ Limite = 1000 × N instances (contournement facile)
- ❌ Memory leak sans cleanup
- ❌ Perdu au restart

**Verdict** : ❌ **Inadapté** pour architecture distribuée (99% des cas).

---

### Option 2 : Base de données relationnelle (PostgreSQL)

```sql
CREATE TABLE rate_limits (
    user_id VARCHAR(255),
    window_start TIMESTAMP,
    request_count INTEGER,
    PRIMARY KEY (user_id, window_start)
);

CREATE INDEX idx_window ON rate_limits(window_start);

-- Check & increment
BEGIN;
SELECT request_count FROM rate_limits
WHERE user_id = 'user123' AND window_start = date_trunc('hour', NOW())
FOR UPDATE;

-- If count < limit:
UPDATE rate_limits SET request_count = request_count + 1
WHERE user_id = 'user123' AND window_start = date_trunc('hour', NOW());
COMMIT;
```

**Avantages** :
- ✅ Consistance ACID
- ✅ Données persistantes
- ✅ Synchronisation entre instances

**Inconvénients** :
- ❌ Latence : 5-50ms (locks, disk I/O)
- ❌ Contention sur hot rows (same user_id + window)
- ❌ Scaling difficile (millions de writes/sec)
- ❌ Cleanup manuel (cronjob)

**Benchmark** (PostgreSQL 14, 1000 TPS) :
```
Opération                    Latence
--------------------------------------
Check + increment (locked)   15-50ms
Cleanup old windows          5-30s
```

**Verdict** : ❌ **Trop lent** pour rate limiting haute fréquence.

---

### Option 3 : Redis avec commandes séparées (non-atomique)

```python
def check_rate_limit(user_id, limit=1000, window=3600):
    key = f"rate_limit:{user_id}"

    # Non-atomic (race condition possible)
    count = redis.get(key)

    if count is None:
        redis.setex(key, window, 1)
        return True

    if int(count) >= limit:
        return False

    redis.incr(key)
    return True
```

**Avantages** :
- ✅ Latence faible : 1-3ms (3 RTT)
- ✅ Synchronisation entre instances
- ✅ TTL automatique (cleanup)

**Inconvénients** :
- ❌ **Race conditions** (3 commandes séparées)
- ❌ Off-by-one errors
- ❌ 3 RTT au lieu de 1

**Verdict** : ⚠️ **Acceptable mais non-optimal** (race conditions).

---

### Option 4 : Redis avec Lua Scripting ✅

```lua
-- Lua script (atomique, 1 RTT)
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('GET', key)

if current == false then
    redis.call('SETEX', key, window, 1)
    return {1, limit - 1, window}
end

current = tonumber(current)

if current >= limit then
    local ttl = redis.call('TTL', key)
    return {0, 0, ttl}  -- Rate limited
end

redis.call('INCR', key)
return {1, limit - current - 1, redis.call('TTL', key)}
```

**Avantages** :
- ✅ Atomicité complète (Lua = single-threaded)
- ✅ Latence : **< 1ms** (1 RTT)
- ✅ Pas de race conditions
- ✅ TTL automatique
- ✅ Synchronisation native entre instances

**Inconvénients** :
- ⚠️ Complexité du scripting Lua (mais gérable)
- ⚠️ Debugging plus difficile

**Trade-off assumé** :
- ➕ Performance optimale
- ➕ Atomicité garantie
- ➕ Simplicité opérationnelle
- ➖ Courbe d'apprentissage Lua (mais scripts simples)

**Verdict** : ✅ **Solution optimale** pour rate limiting distribué.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Applications                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  Mobile  │  │   Web    │  │  Server  │                   │
│  │   App    │  │   App    │  │   API    │                   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                   │
└───────┼─────────────┼─────────────┼─────────────────────────┘
        │ API Key     │             │
        ▼             ▼             ▼
┌──────────────────────────────────────────────────────────────┐
│              Load Balancer (Layer 7)                         │
│  - TLS termination                                           │
│  - Request routing                                           │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│         API Gateway Instances (Stateless)                    │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐            │
│  │ Gateway 1  │   │ Gateway 2  │   │ Gateway N  │            │
│  │            │   │            │   │            │            │
│  │ Rate Limit │   │ Rate Limit │   │ Rate Limit │            │
│  │ Middleware │   │ Middleware │   │ Middleware │            │
│  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘            │
└────────┼────────────────┼────────────────┼───────────────────┘
         │                │                │
         └────────────────┼────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Redis Rate Limit Store                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Keys Structure:                                    │    │
│  │                                                     │    │
│  │  rate_limit:user:{user_id}:{window}                 │    │
│  │  rate_limit:ip:{ip_address}:{window}                │    │
│  │  rate_limit:endpoint:{endpoint}:{window}            │    │
│  │                                                     │    │
│  │  Example:                                           │    │
│  │  rate_limit:user:usr_123:1702300800  → count: 547   │    │
│  │  TTL: 3600 seconds                                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Master-Replica Setup                                       │
│  ┌──────────┐  ┌──────────┐                                 │
│  │ Master   │──│ Replica  │                                 │
│  │ (Write)  │  │ (Read)   │                                 │
│  └──────────┘  └──────────┘                                 │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Backend Microservices                          │
│  - Protected from overload                                  │
│  - Receive only rate-limited traffic                        │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Flux de requête

#### **Requête normale (sous limite)**

```
1. Client sends request
   GET /api/users/123
   Headers:
     X-API-Key: key_abc123
     X-Client-IP: 203.0.113.42

2. API Gateway receives request
   ├─ Extract API Key → user_id
   ├─ Extract Client IP
   └─ Determine rate limit tier (1000 req/hour)

3. Rate Limit Check (Lua script, atomic)
   key = "rate_limit:user:usr_abc123:1702300800"

   EVALSHA <script_sha> 1 {key} 1000 3600

   Redis returns: [1, 453, 2847]
   ├─ allowed: 1 (true)
   ├─ remaining: 453 requests
   └─ retry_after: 2847 seconds

4. Response headers added
   X-RateLimit-Limit: 1000
   X-RateLimit-Remaining: 453
   X-RateLimit-Reset: 1702304400

5. Forward to backend
   → Process request normally

Total latency: ~0.5ms (rate limit check)
```

#### **Requête bloquée (limite atteinte)**

```
1. Client sends request (1001st in window)

2. API Gateway checks rate limit

   EVALSHA <script_sha> 1 {key} 1000 3600

   Redis returns: [0, 0, 2847]
   ├─ allowed: 0 (false)
   ├─ remaining: 0
   └─ retry_after: 2847 seconds

3. Return 429 Too Many Requests
   HTTP/1.1 429 Too Many Requests
   X-RateLimit-Limit: 1000
   X-RateLimit-Remaining: 0
   X-RateLimit-Reset: 1702304400
   Retry-After: 2847

   Body:
   {
     "error": "rate_limit_exceeded",
     "message": "Rate limit of 1000 requests per hour exceeded",
     "retry_after": 2847
   }

4. Log rate limit hit
   └─ Metrics: rate_limit_exceeded{user=usr_abc123}

Total latency: ~0.5ms (no backend call)
```

### 3.3 Décisions architecturales clés

#### **Choix 1 : Algorithme Fixed Window vs Sliding Window**

**Fixed Window (simple, choisi pour MVP)** :

```
Timeline:
13:00:00 ─────────────── 13:59:59 │ 14:00:00 ─────────────── 14:59:59
   Window 1 (1000 req max)        │    Window 2 (1000 req max)
                                  Reset

Avantages:
✅ Simple à implémenter (1 compteur)
✅ Memory efficient
✅ TTL natif Redis

Inconvénients:
⚠️ Burst possible (2000 req en 2 min)
```

**Sliding Window Log (précis, mais coûteux)** :

```
Stockage: Sorted Set avec timestamp
ZADD rate_limit:user123 {timestamp} {request_id}
ZREMRANGEBYSCORE rate_limit:user123 0 (now-3600)
ZCARD rate_limit:user123  # Count requests in window

Avantages:
✅ Précision absolue
✅ Pas de burst

Inconvénients:
❌ Memory: O(N requests)
❌ Latency: O(log N) per request
```

**Sliding Window Counter (compromis, choisi pour production)** :

```lua
-- Approximation avec 2 windows
local current_window = KEYS[1]
local previous_window = KEYS[2]
local current_count = redis.call('GET', current_window) or 0
local previous_count = redis.call('GET', previous_window) or 0

-- Pondération selon position dans fenêtre
local elapsed = current_timestamp % window_size
local weight = (window_size - elapsed) / window_size

local estimated_count = previous_count * weight + current_count

if estimated_count >= limit then
    return 0  -- Rate limited
end

redis.call('INCR', current_window)
```

**Trade-off assumé** :
- ➕ Précision acceptable (erreur < 1%)
- ➕ Memory divisée par N (vs Sliding Log)
- ➖ Approximation (mais burst limité)

---

#### **Choix 2 : Lua Scripting vs Transactions (MULTI/EXEC)**

**Alternative : Transactions Redis** :

```python
# ❌ Transactions (problème: pas de logique conditionnelle)
pipe = redis.pipeline()
pipe.get(key)
pipe.incr(key)
pipe.expire(key, window)
results = pipe.execute()

# Problème: GET puis INCR = non-atomique dans la logique
# WATCH ne fonctionne pas bien sous haute concurrence
```

**Solution : Lua Script** :

```lua
-- ✅ Lua (atomique et conditionnel)
if current >= limit then
    return 0  -- Décision logique atomique
end
redis.call('INCR', key)
```

**Trade-off assumé** :
- ➕ Atomicité avec logique conditionnelle
- ➕ 1 RTT au lieu de 3
- ➖ Courbe d'apprentissage Lua

---

## 4. Implémentation technique

### 4.1 Code Python (Production-Ready)

```python
"""
Rate Limiter avec Redis et Lua
Implémentation production-ready avec multiple algorithms
"""

import time
import hashlib
from typing import Optional, Tuple, Dict, Any
from dataclasses import dataclass
from enum import Enum
import logging

import redis
from redis.exceptions import RedisError

# Configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'db': 0,
    'decode_responses': True,
    'socket_timeout': 2,
    'max_connections': 100
}


# ============================================================================
# Lua Scripts
# ============================================================================

# Fixed Window Counter (simple)
FIXED_WINDOW_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current_time = tonumber(ARGV[3])

local current = redis.call('GET', key)

if current == false then
    redis.call('SETEX', key, window, 1)
    return {1, limit - 1, window}
end

current = tonumber(current)

if current >= limit then
    local ttl = redis.call('TTL', key)
    return {0, 0, ttl}
end

redis.call('INCR', key)
local ttl = redis.call('TTL', key)
return {1, limit - current - 1, ttl}
"""

# Sliding Window Counter (accurate)
SLIDING_WINDOW_SCRIPT = """
local current_key = KEYS[1]
local previous_key = KEYS[2]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current_time = tonumber(ARGV[3])

-- Get counts
local current_count = tonumber(redis.call('GET', current_key) or 0)
local previous_count = tonumber(redis.call('GET', previous_key) or 0)

-- Calculate position in current window
local elapsed = current_time % window
local weight = (window - elapsed) / window

-- Estimated count with sliding window
local estimated_count = math.floor(previous_count * weight + current_count)

if estimated_count >= limit then
    local ttl = redis.call('TTL', current_key)
    return {0, 0, ttl > 0 and ttl or window}
end

-- Increment current window
redis.call('INCR', current_key)
redis.call('EXPIRE', current_key, window * 2)  -- Keep 2 windows

local remaining = limit - estimated_count - 1
local ttl = redis.call('TTL', current_key)

return {1, remaining, ttl}
"""

# Token Bucket
TOKEN_BUCKET_SCRIPT = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])  -- tokens per second
local current_time = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1])
local last_refill = tonumber(bucket[2])

if tokens == nil then
    tokens = capacity
    last_refill = current_time
end

-- Refill tokens based on time elapsed
local elapsed = current_time - last_refill
local refill_amount = math.floor(elapsed * refill_rate)

if refill_amount > 0 then
    tokens = math.min(capacity, tokens + refill_amount)
    last_refill = current_time
end

-- Check if enough tokens
if tokens < requested then
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', last_refill)
    redis.call('EXPIRE', key, 3600)
    return {0, tokens, math.ceil((requested - tokens) / refill_rate)}
end

-- Consume tokens
tokens = tokens - requested

redis.call('HMSET', key, 'tokens', tokens, 'last_refill', last_refill)
redis.call('EXPIRE', key, 3600)

return {1, tokens, 0}
"""


# ============================================================================
# Data Classes
# ============================================================================

class Algorithm(Enum):
    FIXED_WINDOW = "fixed_window"
    SLIDING_WINDOW = "sliding_window"
    TOKEN_BUCKET = "token_bucket"


@dataclass
class RateLimitResult:
    """Résultat d'une vérification de rate limit"""
    allowed: bool
    remaining: int
    retry_after: int  # seconds until reset

    def to_headers(self, limit: int, reset_timestamp: int) -> Dict[str, str]:
        """Convertir en headers HTTP"""
        return {
            'X-RateLimit-Limit': str(limit),
            'X-RateLimit-Remaining': str(self.remaining),
            'X-RateLimit-Reset': str(reset_timestamp),
            'Retry-After': str(self.retry_after) if not self.allowed else '0'
        }


# ============================================================================
# Rate Limiter
# ============================================================================

class RateLimiter:
    """
    Rate Limiter distribué avec Redis

    Features:
    - Multiple algorithms (Fixed Window, Sliding Window, Token Bucket)
    - Lua scripting pour atomicité
    - Support multi-tier (different limits per client)
    - Headers HTTP standards
    """

    def __init__(self, redis_config: Dict = None):
        config = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**config)

        # Load Lua scripts
        self._load_scripts()

        # Test connexion
        try:
            self.redis.ping()
            logger.info("RateLimiter initialized")
        except RedisError as e:
            logger.error(f"Redis connection failed: {e}")
            raise

    def _load_scripts(self):
        """Charger les scripts Lua dans Redis"""
        try:
            self.fixed_window_sha = self.redis.script_load(FIXED_WINDOW_SCRIPT)
            self.sliding_window_sha = self.redis.script_load(SLIDING_WINDOW_SCRIPT)
            self.token_bucket_sha = self.redis.script_load(TOKEN_BUCKET_SCRIPT)

            logger.info("Lua scripts loaded successfully")
        except RedisError as e:
            logger.error(f"Failed to load Lua scripts: {e}")
            raise

    # ========================================================================
    # Fixed Window
    # ========================================================================

    def check_fixed_window(
        self,
        identifier: str,
        limit: int,
        window_seconds: int = 3600
    ) -> RateLimitResult:
        """
        Rate limit avec Fixed Window

        Args:
            identifier: ID unique (user_id, API key, IP, etc.)
            limit: Nombre de requêtes autorisées
            window_seconds: Taille de la fenêtre en secondes

        Returns:
            RateLimitResult
        """
        current_time = int(time.time())
        window_start = current_time - (current_time % window_seconds)

        key = f"rate_limit:fixed:{identifier}:{window_start}"

        try:
            result = self.redis.evalsha(
                self.fixed_window_sha,
                1,
                key,
                limit,
                window_seconds,
                current_time
            )

            allowed = bool(result[0])
            remaining = int(result[1])
            retry_after = int(result[2])

            if not allowed:
                logger.warning(
                    f"Rate limit exceeded: {identifier} "
                    f"(limit={limit}, window={window_seconds}s)"
                )

            return RateLimitResult(
                allowed=allowed,
                remaining=remaining,
                retry_after=retry_after
            )

        except RedisError as e:
            logger.error(f"Rate limit check failed: {e}")
            # Fail open (allow request on Redis error)
            return RateLimitResult(allowed=True, remaining=limit, retry_after=0)

    # ========================================================================
    # Sliding Window
    # ========================================================================

    def check_sliding_window(
        self,
        identifier: str,
        limit: int,
        window_seconds: int = 3600
    ) -> RateLimitResult:
        """
        Rate limit avec Sliding Window (plus précis)

        Utilise 2 windows pour approximation précise
        """
        current_time = int(time.time())
        current_window = current_time - (current_time % window_seconds)
        previous_window = current_window - window_seconds

        current_key = f"rate_limit:sliding:{identifier}:{current_window}"
        previous_key = f"rate_limit:sliding:{identifier}:{previous_window}"

        try:
            result = self.redis.evalsha(
                self.sliding_window_sha,
                2,
                current_key,
                previous_key,
                limit,
                window_seconds,
                current_time
            )

            allowed = bool(result[0])
            remaining = int(result[1])
            retry_after = int(result[2])

            return RateLimitResult(
                allowed=allowed,
                remaining=remaining,
                retry_after=retry_after
            )

        except RedisError as e:
            logger.error(f"Sliding window check failed: {e}")
            return RateLimitResult(allowed=True, remaining=limit, retry_after=0)

    # ========================================================================
    # Token Bucket
    # ========================================================================

    def check_token_bucket(
        self,
        identifier: str,
        capacity: int,
        refill_rate: float,
        tokens_requested: int = 1
    ) -> RateLimitResult:
        """
        Rate limit avec Token Bucket (allow burst)

        Args:
            identifier: ID unique
            capacity: Capacité maximale du bucket
            refill_rate: Tokens ajoutés par seconde
            tokens_requested: Nombre de tokens à consommer

        Returns:
            RateLimitResult
        """
        key = f"rate_limit:token:{identifier}"
        current_time = int(time.time())

        try:
            result = self.redis.evalsha(
                self.token_bucket_sha,
                1,
                key,
                capacity,
                refill_rate,
                current_time,
                tokens_requested
            )

            allowed = bool(result[0])
            remaining = int(result[1])
            retry_after = int(result[2])

            return RateLimitResult(
                allowed=allowed,
                remaining=remaining,
                retry_after=retry_after
            )

        except RedisError as e:
            logger.error(f"Token bucket check failed: {e}")
            return RateLimitResult(allowed=True, remaining=capacity, retry_after=0)

    # ========================================================================
    # Multi-Level Rate Limiting
    # ========================================================================

    def check_multi_level(
        self,
        user_id: str,
        ip_address: str,
        endpoint: str,
        limits: Dict[str, Tuple[int, int]]
    ) -> RateLimitResult:
        """
        Rate limit multi-niveaux (user + IP + endpoint)

        Args:
            user_id: ID utilisateur
            ip_address: IP address
            endpoint: Endpoint appelé
            limits: Dict {"user": (1000, 3600), "ip": (10000, 3600), ...}

        Returns:
            RateLimitResult (du premier niveau qui bloque)
        """
        checks = [
            ("user", user_id, limits.get("user", (1000, 3600))),
            ("ip", ip_address, limits.get("ip", (10000, 3600))),
            ("endpoint", f"{user_id}:{endpoint}", limits.get("endpoint", (100, 60)))
        ]

        for level, identifier, (limit, window) in checks:
            result = self.check_fixed_window(
                f"{level}:{identifier}",
                limit,
                window
            )

            if not result.allowed:
                logger.warning(f"Rate limit hit at level: {level}")
                return result

        # All checks passed
        return RateLimitResult(allowed=True, remaining=0, retry_after=0)

    # ========================================================================
    # Utilities
    # ========================================================================

    def reset_limit(self, identifier: str, algorithm: Algorithm = Algorithm.FIXED_WINDOW):
        """Reset un rate limit (admin/testing)"""
        pattern = f"rate_limit:{algorithm.value}:{identifier}:*"

        cursor = 0
        deleted = 0

        while True:
            cursor, keys = self.redis.scan(cursor, match=pattern, count=100)
            if keys:
                deleted += self.redis.delete(*keys)

            if cursor == 0:
                break

        logger.info(f"Reset rate limit: {identifier} ({deleted} keys deleted)")
        return deleted

    def get_current_usage(self, identifier: str, window_seconds: int = 3600) -> Dict:
        """Obtenir l'usage actuel d'un identifier"""
        current_time = int(time.time())
        window_start = current_time - (current_time % window_seconds)

        key = f"rate_limit:fixed:{identifier}:{window_start}"

        try:
            count = self.redis.get(key)
            ttl = self.redis.ttl(key)

            return {
                "identifier": identifier,
                "current_count": int(count) if count else 0,
                "ttl": ttl if ttl > 0 else 0,
                "window_start": window_start
            }

        except RedisError as e:
            logger.error(f"Failed to get usage: {e}")
            return {}

    def get_stats(self) -> Dict:
        """Statistiques globales du rate limiter"""
        try:
            # Compter les keys actives
            cursor = 0
            active_limits = 0

            while True:
                cursor, keys = self.redis.scan(
                    cursor,
                    match="rate_limit:*",
                    count=1000
                )
                active_limits += len(keys)

                if cursor == 0:
                    break

            # Métriques Redis
            info = self.redis.info("memory")

            return {
                "active_rate_limits": active_limits,
                "memory_used_mb": info['used_memory'] / (1024 * 1024),
                "redis_version": self.redis.info("server")["redis_version"]
            }

        except RedisError as e:
            logger.error(f"Failed to get stats: {e}")
            return {}


# ============================================================================
# Decorator pour Flask/FastAPI
# ============================================================================

def rate_limit(limit: int = 1000, window: int = 3600, algorithm: Algorithm = Algorithm.FIXED_WINDOW):
    """
    Decorator pour rate limiting automatique

    Usage:
        @app.route('/api/users')
        @rate_limit(limit=100, window=60)
        def get_users():
            return {"users": [...]}
    """
    def decorator(func):
        def wrapper(*args, **kwargs):
            # Extract identifier (API key, user_id, etc.)
            # This is framework-specific
            identifier = kwargs.get('user_id', 'anonymous')

            limiter = RateLimiter()

            if algorithm == Algorithm.FIXED_WINDOW:
                result = limiter.check_fixed_window(identifier, limit, window)
            elif algorithm == Algorithm.SLIDING_WINDOW:
                result = limiter.check_sliding_window(identifier, limit, window)
            else:
                result = limiter.check_token_bucket(identifier, limit, limit/window)

            if not result.allowed:
                # Return 429 Too Many Requests
                return {
                    "error": "rate_limit_exceeded",
                    "retry_after": result.retry_after
                }, 429

            # Add headers to response
            response = func(*args, **kwargs)
            # Add headers here (framework-specific)

            return response

        return wrapper
    return decorator


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    # Initialize rate limiter
    limiter = RateLimiter()

    print("\n🚦 Rate Limiter Demo\n")

    user_id = "user_abc123"

    # Test Fixed Window
    print("📊 Fixed Window Algorithm (10 req/min):")
    for i in range(12):
        result = limiter.check_fixed_window(user_id, limit=10, window_seconds=60)

        status = "✅ ALLOWED" if result.allowed else "❌ BLOCKED"
        print(f"   Request #{i+1}: {status} (remaining: {result.remaining})")

        if not result.allowed:
            print(f"   → Retry after {result.retry_after}s")
            break

        time.sleep(0.1)

    # Reset
    limiter.reset_limit(user_id)

    # Test Sliding Window
    print("\n📊 Sliding Window Algorithm (10 req/min):")
    for i in range(12):
        result = limiter.check_sliding_window(user_id, limit=10, window_seconds=60)

        status = "✅ ALLOWED" if result.allowed else "❌ BLOCKED"
        print(f"   Request #{i+1}: {status} (remaining: {result.remaining})")

        if not result.allowed:
            break

        time.sleep(0.1)

    # Reset
    limiter.reset_limit(user_id, Algorithm.SLIDING_WINDOW)

    # Test Token Bucket
    print("\n📊 Token Bucket Algorithm (capacity: 10, refill: 1/sec):")

    # Burst test
    print("   Burst of 10 requests:")
    for i in range(10):
        result = limiter.check_token_bucket(
            user_id,
            capacity=10,
            refill_rate=1.0,
            tokens_requested=1
        )
        print(f"   Request #{i+1}: {'✅' if result.allowed else '❌'} (tokens: {result.remaining})")

    # 11th request (should fail)
    result = limiter.check_token_bucket(user_id, capacity=10, refill_rate=1.0)
    print(f"   Request #11: {'✅' if result.allowed else '❌'} (tokens: {result.remaining})")
    print(f"   → Wait {result.retry_after}s for refill")

    # Wait and retry
    time.sleep(2)
    result = limiter.check_token_bucket(user_id, capacity=10, refill_rate=1.0)
    print(f"   Request #12 (after 2s): {'✅' if result.allowed else '❌'} (tokens: {result.remaining})")

    # Usage stats
    print("\n📈 Current Usage:")
    usage = limiter.get_current_usage(user_id)
    print(f"   Count: {usage.get('current_count', 0)}")
    print(f"   TTL: {usage.get('ttl', 0)}s")

    # Global stats
    print("\n💾 Global Stats:")
    stats = limiter.get_stats()
    print(f"   Active limits: {stats.get('active_rate_limits', 0)}")
    print(f"   Memory: {stats.get('memory_used_mb', 0):.2f} MB")
```

### 4.2 Code Node.js (Express Middleware)

```javascript
/**
 * Rate Limiter Middleware pour Express
 * Production-ready avec multiple algorithms
 */

const Redis = require('ioredis');

// Configuration
const REDIS_CONFIG = {
  host: 'localhost',
  port: 6379,
  retryStrategy: (times) => Math.min(times * 50, 2000),
};

// Lua Scripts
const FIXED_WINDOW_SCRIPT = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('GET', key)

if current == false then
    redis.call('SETEX', key, window, 1)
    return {1, limit - 1, window}
end

current = tonumber(current)

if current >= limit then
    local ttl = redis.call('TTL', key)
    return {0, 0, ttl}
end

redis.call('INCR', key)
local ttl = redis.call('TTL', key)
return {1, limit - current - 1, ttl}
`;

// ============================================================================
// Rate Limiter Class
// ============================================================================

class RateLimiter {
  constructor(config = REDIS_CONFIG) {
    this.redis = new Redis(config);
    this._loadScripts();
  }

  async _loadScripts() {
    try {
      this.fixedWindowSha = await this.redis.script('LOAD', FIXED_WINDOW_SCRIPT);
      console.log('[RateLimiter] Lua scripts loaded');
    } catch (err) {
      console.error('[RateLimiter] Failed to load scripts:', err);
      throw err;
    }
  }

  /**
   * Check fixed window rate limit
   */
  async checkFixedWindow(identifier, limit, windowSeconds = 3600) {
    const currentTime = Math.floor(Date.now() / 1000);
    const windowStart = currentTime - (currentTime % windowSeconds);
    const key = `rate_limit:fixed:${identifier}:${windowStart}`;

    try {
      const result = await this.redis.evalsha(
        this.fixedWindowSha,
        1,
        key,
        limit,
        windowSeconds
      );

      return {
        allowed: result[0] === 1,
        remaining: result[1],
        retryAfter: result[2],
        resetAt: windowStart + windowSeconds,
      };
    } catch (err) {
      console.error('[RateLimiter] Check failed:', err);
      // Fail open
      return { allowed: true, remaining: limit, retryAfter: 0, resetAt: 0 };
    }
  }

  /**
   * Reset rate limit (testing/admin)
   */
  async reset(identifier) {
    const pattern = `rate_limit:*:${identifier}:*`;
    let cursor = '0';
    let deleted = 0;

    do {
      const [newCursor, keys] = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = newCursor;

      if (keys.length > 0) {
        deleted += await this.redis.del(...keys);
      }
    } while (cursor !== '0');

    return deleted;
  }

  async close() {
    await this.redis.quit();
  }
}

// ============================================================================
// Express Middleware
// ============================================================================

/**
 * Create rate limit middleware
 *
 * @param {Object} options
 * @param {number} options.limit - Max requests
 * @param {number} options.window - Window in seconds
 * @param {function} options.keyGenerator - Function to extract identifier
 * @param {function} options.skip - Function to skip rate limiting
 * @param {function} options.handler - Custom handler for rate limit exceeded
 */
function rateLimitMiddleware(options = {}) {
  const {
    limit = 1000,
    window = 3600,
    keyGenerator = (req) => req.ip || 'anonymous',
    skip = () => false,
    handler = null,
  } = options;

  const limiter = new RateLimiter();

  return async (req, res, next) => {
    try {
      // Skip if condition met
      if (skip(req)) {
        return next();
      }

      // Get identifier
      const identifier = keyGenerator(req);

      // Check rate limit
      const result = await limiter.checkFixedWindow(identifier, limit, window);

      // Add headers
      res.set({
        'X-RateLimit-Limit': limit.toString(),
        'X-RateLimit-Remaining': result.remaining.toString(),
        'X-RateLimit-Reset': result.resetAt.toString(),
      });

      if (!result.allowed) {
        res.set('Retry-After', result.retryAfter.toString());

        // Custom handler or default
        if (handler) {
          return handler(req, res, next);
        }

        return res.status(429).json({
          error: 'rate_limit_exceeded',
          message: `Rate limit of ${limit} requests per ${window} seconds exceeded`,
          retry_after: result.retryAfter,
        });
      }

      next();
    } catch (err) {
      console.error('[RateLimiter] Middleware error:', err);
      // Fail open
      next();
    }
  };
}

// ============================================================================
// Express App Example
// ============================================================================

const express = require('express');
const app = express();

// Global rate limit: 1000 req/hour per IP
app.use(rateLimitMiddleware({
  limit: 1000,
  window: 3600,
  keyGenerator: (req) => req.ip,
}));

// Specific endpoint with stricter limit
app.get('/api/expensive',
  rateLimitMiddleware({
    limit: 10,
    window: 60,
    keyGenerator: (req) => req.headers['x-api-key'] || req.ip,
  }),
  (req, res) => {
    res.json({ message: 'Expensive operation completed' });
  }
);

// Multiple tiers based on API key
app.get('/api/users',
  async (req, res, next) => {
    const apiKey = req.headers['x-api-key'];

    // Determine tier
    let limit, window;
    if (apiKey && apiKey.startsWith('premium_')) {
      limit = 10000;
      window = 3600;
    } else if (apiKey) {
      limit = 1000;
      window = 3600;
    } else {
      limit = 100;
      window = 3600;
    }

    // Apply rate limit
    const limiter = new RateLimiter();
    const result = await limiter.checkFixedWindow(apiKey || req.ip, limit, window);

    res.set({
      'X-RateLimit-Limit': limit.toString(),
      'X-RateLimit-Remaining': result.remaining.toString(),
      'X-RateLimit-Reset': result.resetAt.toString(),
    });

    if (!result.allowed) {
      return res.status(429).json({
        error: 'rate_limit_exceeded',
        retry_after: result.retryAfter,
      });
    }

    next();
  },
  (req, res) => {
    res.json({ users: [] });
  }
);

// Admin endpoint to reset rate limit
app.post('/admin/rate-limit/reset', async (req, res) => {
  const { identifier } = req.body;

  if (!identifier) {
    return res.status(400).json({ error: 'identifier required' });
  }

  const limiter = new RateLimiter();
  const deleted = await limiter.reset(identifier);

  res.json({ deleted, identifier });
});

// const PORT = process.env.PORT || 3000;
// app.listen(PORT, () => {
//   console.log(`🚀 Server listening on port ${PORT}`);
// });

module.exports = { RateLimiter, rateLimitMiddleware };
```

---

## 5. Cas avancés et optimisations

### 5.1 Rate Limiting distribué multi-régions

```python
class MultiRegionRateLimiter:
    """Rate limiter avec synchronisation multi-régions"""

    def __init__(self, redis_clusters: Dict[str, Redis]):
        """
        Args:
            redis_clusters: {"us-east": redis_client1, "eu-west": redis_client2, ...}
        """
        self.clusters = redis_clusters
        self.local_region = "us-east"  # Region of current instance

    def check_global_limit(self, identifier: str, limit: int, window: int):
        """
        Vérifier limite globale across regions

        Strategy: Local check + async sync
        """
        # 1. Check local (fast)
        local_redis = self.clusters[self.local_region]
        local_result = self._check_local(local_redis, identifier, limit, window)

        if not local_result.allowed:
            return local_result

        # 2. Async: Sync count to other regions (fire and forget)
        self._sync_to_regions(identifier, window)

        return local_result

    def _sync_to_regions(self, identifier: str, window: int):
        """Synchroniser le count vers autres régions (async)"""
        import threading

        def sync_worker():
            for region, redis_client in self.clusters.items():
                if region != self.local_region:
                    try:
                        key = f"rate_limit:sync:{identifier}:{window}"
                        redis_client.incr(key)
                        redis_client.expire(key, window)
                    except Exception as e:
                        logger.error(f"Sync to {region} failed: {e}")

        thread = threading.Thread(target=sync_worker, daemon=True)
        thread.start()
```

**Trade-off** :
- ➕ Latence locale (pas de cross-region RTT)
- ➖ Éventuelle incohérence temporaire (acceptable)

### 5.2 Rate Limiting avec Redis Cluster (sharding)

```python
def get_rate_limit_key_with_hash_tag(identifier: str, window_start: int) -> str:
    """
    Générer clé avec hash tag pour garantir même shard

    Important: Dans Redis Cluster, seule la partie entre {} est hashée
    """
    # Toutes les keys d'un user sur le même shard
    return f"rate_limit:{{user:{identifier}}}:{window_start}"

# Avantage: MULTI/EXEC fonctionne (même shard)
# Inconvénient: Hot user = hot shard (mais rare)
```

### 5.3 Dynamic Rate Limiting (ajustement automatique)

```python
class AdaptiveRateLimiter:
    """Rate limiter qui s'adapte à la charge backend"""

    def get_dynamic_limit(self, identifier: str, base_limit: int) -> int:
        """
        Ajuster limite selon charge backend

        Ex: Si backend CPU > 80%, réduire limits de 50%
        """
        backend_load = self.get_backend_load()  # From monitoring

        if backend_load > 0.8:
            return int(base_limit * 0.5)  # Reduce to 50%
        elif backend_load > 0.6:
            return int(base_limit * 0.75)  # Reduce to 75%
        else:
            return base_limit  # Full limit
```

---

## 6. Monitoring et métriques

### 6.1 KPIs critiques

```yaml
# Dashboard Grafana

# 1. Rate limit decisions latency
rate_limit_check_latency_ms:
  target: < 1ms (p95)
  critical: > 5ms

# 2. Rate limit hit rate
rate_limit_hit_rate:
  normal: 1-5%
  suspicious: > 20% (possible attack)

# 3. False positives (legitimate blocked)
false_positive_rate:
  target: < 0.1%

# 4. Redis operations
redis_rate_limit_ops_per_sec:
  current: 50k/sec
  max: 100k/sec

# 5. Memory usage
rate_limit_keys_count:
  estimated: 1M keys
  memory_per_key: ~100 bytes
  total: ~100MB
```

### 6.2 Alertes

```yaml
# Prometheus Alerts

groups:
  - name: rate_limiter_alerts
    rules:

      # Latence excessive
      - alert: RateLimiterHighLatency
        expr: histogram_quantile(0.95, rate_limit_check_duration_seconds) > 0.005
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Rate limiter p95 latency > 5ms"

      # Taux de blocage anormalement élevé
      - alert: RateLimiterHighBlockRate
        expr: rate(rate_limit_blocked_total[5m]) / rate(rate_limit_checks_total[5m]) > 0.20
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Rate limit block rate > 20% (possible attack)"

      # Redis erreurs
      - alert: RateLimiterRedisErrors
        expr: rate(rate_limit_redis_errors_total[5m]) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Rate limiter Redis errors > 10/min"
```

---

## 7. Conclusion

### Points clés à retenir

- ✅ **Lua scripting = Atomicité garantie** (pas de race conditions)
- ✅ **Latence < 1ms** pour décisions de rate limiting
- ✅ **Multiple algorithms** : Fixed Window, Sliding Window, Token Bucket
- ✅ **Distribution native** : Redis synchronise entre instances
- ✅ **TTL automatique** : Pas de memory leak
- ✅ **Fail-open safe** : Continue sur erreur Redis (degraded mode)
- ✅ **Headers HTTP standards** : X-RateLimit-* pour clients

### Quand NE PAS utiliser Redis Rate Limiting

- ❌ **Contraintes de latence ultra-strictes** (< 100µs) : In-memory local
- ❌ **Pas d'infrastructure Redis** : Solutions SaaS (Cloudflare, AWS WAF)
- ❌ **Rate limiting très complexe** : ML-based adaptive (nécessite plus)
- ❌ **Audit trail requis** : Logs centralisés nécessaires (Kafka, Elasticsearch)

### Comparaison finale

| Critère | In-Memory | PostgreSQL | Redis (No Lua) | Redis + Lua |
|---------|-----------|------------|----------------|-------------|
| Latence | < 0.1ms | 10-50ms | 1-3ms | **< 1ms** |
| Distribution | ❌ Non | ✅ Oui | ✅ Oui | ✅ Oui |
| Atomicité | ❌ Non | ✅ Oui | ⚠️ Partielle | ✅ Complète |
| Memory leak | ⚠️ Risque | ✅ Non | ✅ Non (TTL) | ✅ Non (TTL) |
| Ops complexity | Faible | Moyenne | Faible | **Faible** |

### Prochaines lectures

- [Cas #6 : Cache de requêtes SQL](./06-cas-cache-requetes-sql.md) → Cache-aside pattern
- [Scripting Lua](../07-atomicite-programmabilite/03-scripting-lua-commandes-atomiques.md) → Lua avancé
- [Distributed Locking](../06-patterns-developpement-avances/05-distributed-locking-redlock.md) → Redlock pattern

---

**📚 Ressources complémentaires** :
- [Rate Limiting Algorithms Explained](https://konghq.com/blog/how-to-design-a-scalable-rate-limiting-algorithm)
- [Redis Lua Scripting](https://redis.io/docs/manual/programmability/eval-intro/)
- [RFC 6585 - HTTP 429 Status Code](https://tools.ietf.org/html/rfc6585)

⏭️ [Cas #6 : Cache de résultats de requêtes SQL complexes](/16-etudes-cas-patterns-reels/06-cas-cache-requetes-sql.md)
