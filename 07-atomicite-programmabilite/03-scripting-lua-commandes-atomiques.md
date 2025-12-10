🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3 Scripting Lua : Créer ses propres commandes atomiques

## Introduction : La puissance de Lua dans Redis

Redis intègre un interpréteur **Lua 5.1** qui permet d'exécuter des scripts personnalisés **directement sur le serveur**. Cette capacité transforme Redis d'un simple data store en une plateforme programmable capable d'exécuter de la logique métier complexe de manière **atomique**.

### Pourquoi Lua dans Redis ?

**Le problème fondamental** :
```python
# ❌ Logique métier côté client : 3 aller-retours réseau
balance = redis.get("account:balance")        # RTT 1
if int(balance) >= 100:                       # Logique locale
    redis.decrby("account:balance", 100)      # RTT 2
    redis.lpush("transactions", "purchase")   # RTT 3
    # ⚠️ Race condition entre GET et DECRBY
```

**La solution Lua** :
```lua
-- ✅ Logique métier côté serveur : 1 seul aller-retour
local balance = tonumber(redis.call('GET', KEYS[1]))
if balance >= 100 then
    redis.call('DECRBY', KEYS[1], 100)
    redis.call('LPUSH', KEYS[2], 'purchase')
    return {1, balance - 100}
else
    return {0, balance}
end
-- ✅ Atomicité garantie
-- ✅ Un seul RTT
-- ✅ Pas de race condition
```

### Avantages du scripting Lua

| Avantage | Description | Impact |
|----------|-------------|--------|
| **Atomicité** | Le script entier s'exécute sans interruption | Élimine les race conditions |
| **Performance** | Réduction drastique des RTT | 10-100x plus rapide pour les opérations complexes |
| **Logique complexe** | Conditions, boucles, calculs | Impossible avec MULTI/EXEC |
| **Simplicité** | Code côté serveur | Moins de bugs côté client |
| **Consistance** | Même logique pour tous les clients | Pas de divergence d'implémentation |

### Limitations importantes

⚠️ **Points critiques à comprendre** :
- Les scripts Lua **bloquent** Redis pendant leur exécution
- Pas de randomness (pas de `math.random()`)
- Pas d'accès réseau ou fichiers
- Pas de gestion du temps (horloge non fiable)
- Script trop long = blocage des autres clients

## Anatomie d'un script Lua Redis

### Structure de base

```lua
-- Commentaire en Lua (double tiret)

-- KEYS[1], KEYS[2], ... → Clés Redis passées en paramètre
-- ARGV[1], ARGV[2], ... → Arguments passés en paramètre

-- Lecture
local value = redis.call('GET', KEYS[1])

-- Écriture
redis.call('SET', KEYS[1], ARGV[1])

-- Retour de valeur
return value
```

### Exécution avec EVAL et EVALSHA

```python
import redis

r = redis.Redis(decode_responses=True)

# Méthode 1 : EVAL (envoie le script à chaque fois)
script = """
local value = redis.call('GET', KEYS[1])
return value
"""

result = r.eval(script, 1, "mykey")  # 1 = nombre de clés

# Méthode 2 : EVALSHA (utilise le hash SHA1 du script)
script_sha = r.script_load(script)  # Charge le script une fois
result = r.evalsha(script_sha, 1, "mykey")  # Réutilise via SHA1

# Méthode 3 : Helper Python (recommandé)
from redis.commands.core import Script

increment_script = Script(r, """
    local current = tonumber(redis.call('GET', KEYS[1])) or 0
    local new_value = current + tonumber(ARGV[1])
    redis.call('SET', KEYS[1], new_value)
    return new_value
""")

result = increment_script(keys=["counter"], args=[5])
```

### Paramètres : KEYS vs ARGV

```lua
-- KEYS[] : Clés Redis (pour le clustering et l'analyse)
-- ARGV[] : Arguments arbitraires

-- ✅ BON : Distinguer clés et valeurs
local stock = redis.call('GET', KEYS[1])  -- Clé
local quantity = tonumber(ARGV[1])        -- Valeur

-- ❌ MAUVAIS : Tout dans ARGV
-- local stock = redis.call('GET', ARGV[1])  -- Problématique pour le clustering
```

**Pourquoi cette distinction ?**
- Redis Cluster utilise `KEYS[]` pour déterminer le shard
- Les outils d'analyse peuvent scanner les clés utilisées
- Bonne pratique pour la maintenabilité

## API Lua Redis : redis.call() vs redis.pcall()

### redis.call() : Propagation des erreurs

```lua
-- redis.call() propage les erreurs et stoppe le script
local value = redis.call('GET', KEYS[1])

-- Si la commande échoue, le script s'arrête immédiatement
local result = redis.call('INCR', 'mystring')  -- Erreur si mystring n'est pas un nombre
-- Le code suivant ne sera JAMAIS exécuté en cas d'erreur

return value
```

### redis.pcall() : Capture des erreurs

```lua
-- redis.pcall() capture les erreurs et les retourne
local value = redis.pcall('GET', KEYS[1])

-- Si la commande échoue, on reçoit une table d'erreur
local result = redis.pcall('INCR', 'mystring')

-- On peut vérifier le type de retour
if type(result) == 'table' and result.err then
    -- Gérer l'erreur
    return {error = result.err}
end

return result
```

### Exemple comparatif complet

```lua
-- Script avec gestion d'erreur robuste
local function safe_increment(key, amount)
    -- Tentative d'incrémentation
    local result = redis.pcall('INCRBY', key, amount)

    -- Vérification d'erreur
    if type(result) == 'table' and result.err then
        -- Si erreur (ex: la clé contient une string)
        -- Initialiser à 0 et réessayer
        redis.call('SET', key, 0)
        result = redis.call('INCRBY', key, amount)
    end

    return result
end

return safe_increment(KEYS[1], tonumber(ARGV[1]))
```

## Types de données Lua et conversions

### Conversions Redis ↔ Lua

```lua
-- Redis → Lua
local str = redis.call('GET', 'mykey')          -- string | nil
local num = redis.call('INCR', 'counter')       -- number
local list = redis.call('LRANGE', 'list', 0, -1) -- table (array)
local hash = redis.call('HGETALL', 'hash')      -- table (flat array)

-- Lua → Redis
return "string"           -- Redis: bulk string
return 123                -- Redis: integer
return {1, 2, 3}          -- Redis: array
return redis.error_reply("Custom error")  -- Redis: error
return redis.status_reply("OK")           -- Redis: status
```

### Tableau de conversion

| Type Redis | Type Lua | Exemple |
|------------|----------|---------|
| Integer | `number` | `42` |
| Bulk string | `string` | `"hello"` |
| Array | `table` (indexed) | `{1, 2, 3}` |
| Null | `false` | `false` |
| Status reply | `table` {ok="..."} | `{ok="OK"}` |
| Error reply | `table` {err="..."} | `{err="ERR ..."}` |

### Pièges de conversion : HGETALL

```lua
-- ⚠️ HGETALL retourne un tableau plat, pas une table associative
local hash = redis.call('HGETALL', KEYS[1])
-- hash = {"field1", "value1", "field2", "value2"}

-- ❌ Accès direct ne fonctionne PAS
-- local value = hash["field1"]  -- nil !

-- ✅ Conversion en table associative
local function hgetall_to_table(flat_array)
    local result = {}
    for i = 1, #flat_array, 2 do
        result[flat_array[i]] = flat_array[i + 1]
    end
    return result
end

local hash_table = hgetall_to_table(hash)
local value = hash_table["field1"]  -- ✅ Fonctionne
```

## Exemples progressifs : De simple à complexe

### Exemple 1 : Incrément conditionnel

```lua
-- increment_with_max.lua
-- KEYS[1] : clé du compteur
-- ARGV[1] : incrément
-- ARGV[2] : valeur maximale

local current = tonumber(redis.call('GET', KEYS[1])) or 0
local increment = tonumber(ARGV[1])
local max_value = tonumber(ARGV[2])

-- Vérification de la limite
if current + increment > max_value then
    return redis.error_reply('ERR would exceed maximum value')
end

-- Incrémenter
local new_value = redis.call('INCRBY', KEYS[1], increment)
return new_value
```

**Utilisation en Python** :

```python
script = r.register_script("""
local current = tonumber(redis.call('GET', KEYS[1])) or 0
local increment = tonumber(ARGV[1])
local max_value = tonumber(ARGV[2])

if current + increment > max_value then
    return redis.error_reply('ERR would exceed maximum value')
end

local new_value = redis.call('INCRBY', KEYS[1], increment)
return new_value
""")

try:
    result = script(keys=['views:video:123'], args=[1, 1000000])
    print(f"New value: {result}")
except redis.ResponseError as e:
    print(f"Error: {e}")
```

### Exemple 2 : Achat avec vérification de stock atomique

```lua
-- purchase_product.lua
-- KEYS[1] : product:ID:stock
-- KEYS[2] : product:ID:sold
-- KEYS[3] : user:ID:purchases
-- ARGV[1] : user_id
-- ARGV[2] : quantity
-- ARGV[3] : timestamp

local stock_key = KEYS[1]
local sold_key = KEYS[2]
local purchases_key = KEYS[3]

local user_id = ARGV[1]
local quantity = tonumber(ARGV[2])
local timestamp = tonumber(ARGV[3])

-- Validation des paramètres
if not quantity or quantity <= 0 then
    return redis.error_reply('ERR invalid quantity')
end

-- Vérifier le stock disponible
local stock = tonumber(redis.call('GET', stock_key)) or 0

if stock < quantity then
    return {
        ok = false,
        error = 'insufficient_stock',
        available = stock,
        requested = quantity
    }
end

-- Décrémenter le stock
local new_stock = redis.call('DECRBY', stock_key, quantity)

-- Incrémenter les ventes
redis.call('INCRBY', sold_key, quantity)

-- Enregistrer l'achat pour l'utilisateur
local purchase_data = string.format('%s:%d:%d', user_id, quantity, timestamp)
redis.call('LPUSH', purchases_key, purchase_data)

-- Limiter l'historique à 1000 achats
redis.call('LTRIM', purchases_key, 0, 999)

-- Retourner le succès avec détails
return {
    ok = true,
    remaining_stock = new_stock,
    quantity_purchased = quantity,
    timestamp = timestamp
}
```

**Utilisation** :

```python
import time

purchase_script = r.register_script("""
-- [script complet ci-dessus]
""")

def purchase_product(product_id, user_id, quantity):
    """
    Achat atomique avec vérification de stock
    """
    result = purchase_script(
        keys=[
            f'product:{product_id}:stock',
            f'product:{product_id}:sold',
            f'user:{user_id}:purchases'
        ],
        args=[user_id, quantity, int(time.time())]
    )

    if isinstance(result, list) and len(result) > 0:
        # Conversion du résultat Lua en dict Python
        return {
            'ok': result[0] == 1 if isinstance(result[0], int) else result[0],
            'remaining_stock': result[1] if len(result) > 1 else None,
            'error': result[2] if len(result) > 2 and result[0] == 0 else None
        }

    return result

# Test
r.set('product:laptop:stock', 10)
r.set('product:laptop:sold', 0)

result = purchase_product('laptop', 'user:1001', 2)
print(result)
# {'ok': True, 'remaining_stock': 8, 'error': None}

# Tentative d'achat au-delà du stock
result = purchase_product('laptop', 'user:1002', 20)
print(result)
# {'ok': False, 'remaining_stock': None, 'error': 'insufficient_stock'}
```

### Exemple 3 : Rate Limiting avec Token Bucket

```lua
-- rate_limit_token_bucket.lua
-- Implémente l'algorithme Token Bucket pour le rate limiting
-- KEYS[1] : clé du bucket (ex: "ratelimit:api:user:123")
-- ARGV[1] : capacity (nombre max de tokens)
-- ARGV[2] : refill_rate (tokens par seconde)
-- ARGV[3] : requested (nombre de tokens demandés)
-- ARGV[4] : current_time (timestamp en secondes)

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

-- Récupérer l'état actuel du bucket
local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1])
local last_refill = tonumber(bucket[2])

-- Initialiser si le bucket n'existe pas
if not tokens then
    tokens = capacity
    last_refill = now
end

-- Calculer les tokens à ajouter depuis le dernier refill
local time_passed = now - last_refill
local tokens_to_add = time_passed * refill_rate

-- Mettre à jour les tokens (avec limite = capacity)
tokens = math.min(capacity, tokens + tokens_to_add)

-- Vérifier si on peut satisfaire la requête
local allowed = false
if tokens >= requested then
    tokens = tokens - requested
    allowed = true
end

-- Sauvegarder le nouvel état
redis.call('HSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) * 2)

-- Retourner le résultat
return {
    allowed = allowed,
    tokens_remaining = math.floor(tokens),
    retry_after = allowed and 0 or math.ceil((requested - tokens) / refill_rate)
}
```

**Classe Python pour gérer le rate limiting** :

```python
class TokenBucketRateLimiter:
    """
    Rate limiter basé sur Token Bucket avec script Lua
    """

    def __init__(self, redis_client, capacity=100, refill_rate=10):
        """
        Args:
            capacity: Nombre maximum de tokens
            refill_rate: Tokens ajoutés par seconde
        """
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate

        self.script = redis_client.register_script("""
            -- [script Lua complet ci-dessus]
        """)

    def allow(self, key, requested=1):
        """
        Vérifie si la requête est autorisée

        Args:
            key: Identifiant unique (ex: user_id, ip_address)
            requested: Nombre de tokens demandés

        Returns:
            dict avec allowed, tokens_remaining, retry_after
        """
        bucket_key = f"ratelimit:{key}"
        now = time.time()

        result = self.script(
            keys=[bucket_key],
            args=[self.capacity, self.refill_rate, requested, now]
        )

        return {
            'allowed': bool(result[0]),
            'tokens_remaining': result[1],
            'retry_after': result[2]
        }

    def get_status(self, key):
        """
        Obtient le statut actuel du bucket sans consommer de tokens
        """
        bucket_key = f"ratelimit:{key}"
        bucket = self.redis.hmget(bucket_key, 'tokens', 'last_refill')

        if bucket[0] is None:
            return {
                'tokens': self.capacity,
                'capacity': self.capacity,
                'refill_rate': self.refill_rate
            }

        now = time.time()
        tokens = float(bucket[0])
        last_refill = float(bucket[1])

        # Calculer les tokens actuels
        time_passed = now - last_refill
        tokens_to_add = time_passed * self.refill_rate
        current_tokens = min(self.capacity, tokens + tokens_to_add)

        return {
            'tokens': int(current_tokens),
            'capacity': self.capacity,
            'refill_rate': self.refill_rate
        }

# Utilisation
limiter = TokenBucketRateLimiter(
    r,
    capacity=100,      # 100 requêtes max
    refill_rate=10     # 10 requêtes/seconde
)

# Simuler des requêtes API
user_id = "user:1001"

for i in range(110):
    result = limiter.allow(user_id, requested=1)

    if result['allowed']:
        print(f"Request {i+1}: ✅ Allowed ({result['tokens_remaining']} tokens left)")
    else:
        print(f"Request {i+1}: ❌ Rate limited (retry after {result['retry_after']}s)")
        break

# Vérifier le statut
status = limiter.get_status(user_id)
print(f"\nStatus: {status['tokens']}/{status['capacity']} tokens")
```

### Exemple 4 : Distributed Lock avec auto-release

```lua
-- distributed_lock_acquire.lua
-- KEYS[1] : clé du lock
-- ARGV[1] : lock_id unique (pour empêcher le unlock par d'autres)
-- ARGV[2] : ttl en secondes

local lock_key = KEYS[1]
local lock_id = ARGV[1]
local ttl = tonumber(ARGV[2])

-- Tenter d'acquérir le lock (SET NX)
local acquired = redis.call('SET', lock_key, lock_id, 'NX', 'EX', ttl)

if acquired then
    return {success = true, lock_id = lock_id}
else
    -- Retourner l'ID du détenteur actuel
    local current_holder = redis.call('GET', lock_key)
    return {success = false, held_by = current_holder}
end
```

```lua
-- distributed_lock_release.lua
-- KEYS[1] : clé du lock
-- ARGV[1] : lock_id (doit correspondre pour release)

local lock_key = KEYS[1]
local lock_id = ARGV[1]

-- Vérifier que le lock appartient bien à celui qui veut le release
local current_id = redis.call('GET', lock_key)

if current_id == lock_id then
    redis.call('DEL', lock_key)
    return {success = true, released = true}
else
    return {success = false, error = 'lock_not_owned'}
end
```

**Classe Python pour le distributed lock** :

```python
import uuid
import contextlib

class RedisLock:
    """
    Distributed lock implémenté avec Lua pour garantir l'atomicité
    """

    def __init__(self, redis_client, lock_name, ttl=10):
        self.redis = redis_client
        self.lock_name = f"lock:{lock_name}"
        self.ttl = ttl
        self.lock_id = str(uuid.uuid4())

        self.acquire_script = redis_client.register_script("""
            local lock_key = KEYS[1]
            local lock_id = ARGV[1]
            local ttl = tonumber(ARGV[2])

            local acquired = redis.call('SET', lock_key, lock_id, 'NX', 'EX', ttl)

            if acquired then
                return 1
            else
                return 0
            end
        """)

        self.release_script = redis_client.register_script("""
            local lock_key = KEYS[1]
            local lock_id = ARGV[1]

            local current_id = redis.call('GET', lock_key)

            if current_id == lock_id then
                redis.call('DEL', lock_key)
                return 1
            else
                return 0
            end
        """)

    def acquire(self, timeout=None):
        """
        Acquiert le lock avec retry optionnel
        """
        start_time = time.time()

        while True:
            acquired = self.acquire_script(
                keys=[self.lock_name],
                args=[self.lock_id, self.ttl]
            )

            if acquired:
                return True

            if timeout is not None and (time.time() - start_time) >= timeout:
                return False

            time.sleep(0.01)  # Attendre avant de réessayer

    def release(self):
        """
        Libère le lock
        """
        released = self.release_script(
            keys=[self.lock_name],
            args=[self.lock_id]
        )
        return bool(released)

    @contextlib.contextmanager
    def context(self, timeout=None):
        """
        Context manager pour utilisation avec 'with'
        """
        acquired = self.acquire(timeout=timeout)
        if not acquired:
            raise TimeoutError(f"Could not acquire lock: {self.lock_name}")

        try:
            yield
        finally:
            self.release()

# Utilisation
def critical_section():
    """
    Fonction critique qui ne doit s'exécuter qu'une fois à la fois
    """
    lock = RedisLock(r, "my_critical_section", ttl=5)

    with lock.context(timeout=10):
        print("Début de la section critique")
        # Opérations critiques...
        time.sleep(2)
        print("Fin de la section critique")

# Test avec threads concurrents
import threading

def worker(worker_id):
    try:
        critical_section()
        print(f"Worker {worker_id}: ✅ Completed")
    except TimeoutError:
        print(f"Worker {worker_id}: ❌ Timeout")

threads = [threading.Thread(target=worker, args=(i,)) for i in range(5)]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

## Patterns avancés

### Pattern 1 : Boucles et itérations

```lua
-- batch_expire.lua
-- Définir une expiration pour plusieurs clés avec un pattern
-- KEYS[1] : pattern (ex: "session:*")
-- ARGV[1] : ttl en secondes

local pattern = KEYS[1]
local ttl = tonumber(ARGV[1])
local cursor = "0"
local count = 0

repeat
    -- SCAN avec le pattern
    local result = redis.call('SCAN', cursor, 'MATCH', pattern, 'COUNT', 100)
    cursor = result[1]
    local keys = result[2]

    -- Définir l'expiration pour chaque clé trouvée
    for i, key in ipairs(keys) do
        redis.call('EXPIRE', key, ttl)
        count = count + 1
    end

until cursor == "0"

return count
```

**⚠️ Attention** : Les boucles dans Lua peuvent bloquer Redis longtemps. Utilisez avec précaution et limitez les itérations.

### Pattern 2 : Fonctions réutilisables

```lua
-- multi_operations.lua
-- Démontre l'utilisation de fonctions locales

-- Fonction helper pour parser une valeur JSON simple
local function parse_json_simple(json_str)
    -- Implémentation simplifiée pour l'exemple
    -- En production, utilisez cjson si disponible
    return json_str
end

-- Fonction pour valider un email (simplifiée)
local function is_valid_email(email)
    return string.match(email, "@") ~= nil
end

-- Fonction pour créer un utilisateur
local function create_user(user_id, email, name)
    if not is_valid_email(email) then
        return redis.error_reply('ERR invalid email format')
    end

    local user_key = 'user:' .. user_id

    -- Vérifier si l'utilisateur existe déjà
    if redis.call('EXISTS', user_key) == 1 then
        return redis.error_reply('ERR user already exists')
    end

    -- Créer l'utilisateur
    redis.call('HSET', user_key,
        'email', email,
        'name', name,
        'created_at', redis.call('TIME')[1]
    )

    -- Ajouter à l'index d'emails
    redis.call('HSET', 'index:emails', email, user_id)

    return {success = true, user_id = user_id}
end

-- Point d'entrée du script
local user_id = ARGV[1]
local email = ARGV[2]
local name = ARGV[3]

return create_user(user_id, email, name)
```

### Pattern 3 : Gestion d'erreur avec pcall

```lua
-- safe_batch_operations.lua
-- Exécute plusieurs opérations avec gestion d'erreur individuelle

local results = {}
local errors = {}

-- Opération 1 : Peut échouer
local success, result = pcall(function()
    return redis.call('INCR', KEYS[1])
end)

if success then
    results[1] = result
else
    errors[1] = {operation = 'INCR', error = tostring(result)}
end

-- Opération 2 : Peut échouer
local success, result = pcall(function()
    return redis.call('LPUSH', KEYS[2], ARGV[1])
end)

if success then
    results[2] = result
else
    errors[2] = {operation = 'LPUSH', error = tostring(result)}
end

-- Opération 3 : Peut échouer
local success, result = pcall(function()
    return redis.call('ZADD', KEYS[3], ARGV[2], ARGV[3])
end)

if success then
    results[3] = result
else
    errors[3] = {operation = 'ZADD', error = tostring(result)}
end

-- Retourner les résultats et les erreurs
return {
    results = results,
    errors = errors,
    success_count = #results,
    error_count = #errors
}
```

### Pattern 4 : Transactions compensatoires (Saga pattern)

```lua
-- saga_transfer.lua
-- Transfert avec compensation automatique en cas d'erreur
-- KEYS[1] : from_account
-- KEYS[2] : to_account
-- KEYS[3] : transaction_log
-- ARGV[1] : amount
-- ARGV[2] : transaction_id

local from_account = KEYS[1]
local to_account = KEYS[2]
local tx_log = KEYS[3]
local amount = tonumber(ARGV[1])
local tx_id = ARGV[2]

-- État de compensation
local compensate = false
local compensation_steps = {}

-- Step 1 : Vérifier le solde
local from_balance = tonumber(redis.call('GET', from_account)) or 0
if from_balance < amount then
    return redis.error_reply('ERR insufficient funds')
end

-- Step 2 : Débiter le compte source
local success, result = pcall(function()
    return redis.call('DECRBY', from_account, amount)
end)

if not success then
    return redis.error_reply('ERR failed to debit source account')
end

table.insert(compensation_steps, {account = from_account, amount = amount, operation = 'credit'})

-- Step 3 : Créditer le compte destination
success, result = pcall(function()
    return redis.call('INCRBY', to_account, amount)
end)

if not success then
    -- COMPENSATION : Rembourser le compte source
    compensate = true
    redis.call('INCRBY', from_account, amount)

    redis.call('HSET', tx_log, tx_id,
        string.format('FAILED:compensated:%s', tostring(result)))

    return redis.error_reply('ERR failed to credit destination, transaction compensated')
end

-- Step 4 : Logger la transaction
redis.call('HSET', tx_log, tx_id,
    string.format('SUCCESS:from=%s:to=%s:amount=%d', from_account, to_account, amount))

return {
    success = true,
    transaction_id = tx_id,
    compensated = compensate
}
```

## Performance et optimisations

### Éviter les commandes O(n) dangereuses

```lua
-- ❌ MAUVAIS : KEYS bloque Redis
local all_keys = redis.call('KEYS', '*')  -- O(n) sur toute la DB !
for i, key in ipairs(all_keys) do
    redis.call('EXPIRE', key, 3600)
end

-- ✅ BON : SCAN itère sans bloquer
local cursor = "0"
local count = 0

repeat
    local result = redis.call('SCAN', cursor, 'MATCH', 'user:*', 'COUNT', 100)
    cursor = result[1]
    local keys = result[2]

    for i, key in ipairs(keys) do
        redis.call('EXPIRE', key, 3600)
        count = count + 1

        -- Limite de sécurité pour éviter les scripts trop longs
        if count >= 10000 then
            return {partial = true, processed = count}
        end
    end
until cursor == "0"

return {complete = true, processed = count}
```

### Limiter la durée d'exécution

```lua
-- Script avec limite de temps auto-imposée
local start_time = redis.call('TIME')
local start_seconds = tonumber(start_time[1])
local start_micros = tonumber(start_time[2])

local processed = 0
local max_duration_ms = 50  -- Maximum 50ms

while true do
    -- Faire le travail
    -- ... opérations ...
    processed = processed + 1

    -- Vérifier le temps écoulé tous les 100 items
    if processed % 100 == 0 then
        local current_time = redis.call('TIME')
        local current_seconds = tonumber(current_time[1])
        local current_micros = tonumber(current_time[2])

        local elapsed_ms = (current_seconds - start_seconds) * 1000 +
                          (current_micros - start_micros) / 1000

        if elapsed_ms > max_duration_ms then
            return {
                partial = true,
                processed = processed,
                reason = 'time_limit_exceeded'
            }
        end
    end

    -- Condition de sortie normale
    if processed >= 1000 then
        break
    end
end

return {complete = true, processed = processed}
```

### Optimiser les conversions de types

```lua
-- ❌ LENT : Conversions répétées
for i = 1, 1000 do
    local value = redis.call('GET', 'key:' .. tostring(i))
    local num = tonumber(value)  -- Conversion à chaque itération
    -- ...
end

-- ✅ RAPIDE : Minimiser les conversions
local keys = {}
for i = 1, 1000 do
    keys[i] = 'key:' .. i
end

local values = redis.call('MGET', unpack(keys))
for i, value in ipairs(values) do
    local num = tonumber(value)
    -- ...
end
```

## Debugging de scripts Lua

### Utiliser redis.log() pour le debugging

```lua
-- Logging dans les logs Redis
redis.log(redis.LOG_DEBUG, 'Debug message')
redis.log(redis.LOG_VERBOSE, 'Verbose message')
redis.log(redis.LOG_NOTICE, 'Notice message')
redis.log(redis.LOG_WARNING, 'Warning message')

-- Exemple pratique
local value = redis.call('GET', KEYS[1])
redis.log(redis.LOG_NOTICE, 'Got value: ' .. tostring(value))

if not value then
    redis.log(redis.LOG_WARNING, 'Key does not exist: ' .. KEYS[1])
    return nil
end

return value
```

### Script de test avec assertions

```lua
-- test_script.lua
-- Script avec validations intégrées pour le debugging

local function assert(condition, message)
    if not condition then
        redis.log(redis.LOG_WARNING, 'ASSERTION FAILED: ' .. message)
        return redis.error_reply('ASSERTION FAILED: ' .. message)
    end
end

-- Validation des paramètres
assert(#KEYS >= 1, 'At least 1 key required')
assert(#ARGV >= 1, 'At least 1 argument required')

local key = KEYS[1]
local value = ARGV[1]

-- Validation du type
local key_type = redis.call('TYPE', key)['ok']
assert(key_type == 'string' or key_type == 'none',
       'Key must be string type, got: ' .. key_type)

-- Logging du state
redis.log(redis.LOG_NOTICE, string.format(
    'Setting %s = %s', key, value))

redis.call('SET', key, value)

return 'OK'
```

### Tester avec redis-cli

```bash
# Charger et tester un script
redis-cli SCRIPT LOAD "$(cat my_script.lua)"
# SHA: 5e7d...

# Exécuter avec EVALSHA
redis-cli EVALSHA 5e7d... 1 mykey arg1 arg2

# Voir les scripts chargés
redis-cli SCRIPT EXISTS 5e7d...

# Vider le cache de scripts
redis-cli SCRIPT FLUSH

# Killer un script qui tourne trop longtemps
redis-cli SCRIPT KILL
```

### Framework de test en Python

```python
import unittest

class LuaScriptTest(unittest.TestCase):
    """
    Tests unitaires pour scripts Lua
    """

    def setUp(self):
        self.redis = redis.Redis(decode_responses=True)
        self.redis.flushdb()

        # Charger le script
        self.script = self.redis.register_script("""
            local key = KEYS[1]
            local amount = tonumber(ARGV[1])

            local current = tonumber(redis.call('GET', key)) or 0
            local new_value = current + amount

            redis.call('SET', key, new_value)
            return new_value
        """)

    def test_increment_from_zero(self):
        """Test incrément depuis 0"""
        result = self.script(keys=['counter'], args=[5])
        self.assertEqual(result, 5)

    def test_increment_from_existing(self):
        """Test incrément depuis une valeur existante"""
        self.redis.set('counter', 10)
        result = self.script(keys=['counter'], args=[5])
        self.assertEqual(result, 15)

    def test_concurrent_increments(self):
        """Test d'incréments concurrents"""
        import threading

        def increment():
            for _ in range(100):
                self.script(keys=['counter'], args=[1])

        threads = [threading.Thread(target=increment) for _ in range(10)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()

        final = int(self.redis.get('counter'))
        self.assertEqual(final, 1000)  # 10 threads * 100 increments

    def tearDown(self):
        self.redis.flushdb()

if __name__ == '__main__':
    unittest.main()
```

## Limitations et contraintes de Lua dans Redis

### 1. Pas de randomness

```lua
-- ❌ NE FONCTIONNE PAS
local random = math.random()  -- Erreur !

-- ✅ SOLUTION : Utiliser un seed externe
local seed = tonumber(ARGV[1])
math.randomseed(seed)
local random = math.random()
```

### 2. Pas de gestion du temps fiable

```lua
-- ⚠️ ATTENTION : TIME peut ne pas être fiable pour les durées
local start = redis.call('TIME')
-- ... opérations ...
local finish = redis.call('TIME')

-- Le temps peut "reculer" si l'horloge système est ajustée
```

### 3. Pas d'accès réseau ou système de fichiers

```lua
-- ❌ IMPOSSIBLE
local socket = require('socket')  -- Erreur !
local file = io.open('file.txt')  -- Erreur !
```

### 4. Mémoire limitée pour le script

```lua
-- ⚠️ Les scripts très gros peuvent échouer
-- Limite : typiquement quelques MB

-- ✅ SOLUTION : Diviser en plusieurs scripts
-- ou utiliser Redis Functions (v7+)
```

### 5. Script bloque Redis

```lua
-- ❌ DANGEREUX : Boucle infinie
while true do
    -- Bloque Redis indéfiniment !
end

-- ✅ TOUJOURS avoir une condition de sortie
local iterations = 0
local max_iterations = 10000

while iterations < max_iterations do
    -- ... travail ...
    iterations = iterations + 1
end
```

## Cas d'usage avancés : Scripts production-ready

### 1. Système de queue avec priorités

```lua
-- priority_queue_push.lua
-- KEYS[1] : queue name
-- ARGV[1] : priority (0-9, 0 = highest)
-- ARGV[2] : item data
-- ARGV[3] : timestamp

local queue_name = KEYS[1]
local priority = tonumber(ARGV[1])
local item_data = ARGV[2]
local timestamp = tonumber(ARGV[3])

-- Validation
if priority < 0 or priority > 9 then
    return redis.error_reply('ERR priority must be between 0 and 9')
end

-- Score composite : (priority * 1000000000) + timestamp
-- Permet de trier par priorité d'abord, puis par timestamp
local score = (priority * 1000000000) + timestamp

-- Générer un ID unique
local item_id = redis.call('INCR', queue_name .. ':id_counter')

-- Stocker l'item
local item_key = queue_name .. ':items:' .. item_id
redis.call('SET', item_key, item_data)
redis.call('EXPIRE', item_key, 86400)  -- 24h TTL

-- Ajouter à la queue
redis.call('ZADD', queue_name .. ':queue', score, item_id)

-- Statistiques
redis.call('HINCRBY', queue_name .. ':stats', 'total_pushed', 1)
redis.call('HINCRBY', queue_name .. ':stats', 'priority:' .. priority, 1)

return {
    item_id = item_id,
    score = score,
    position = redis.call('ZRANK', queue_name .. ':queue', item_id)
}
```

```lua
-- priority_queue_pop.lua
-- KEYS[1] : queue name
-- ARGV[1] : max items to pop

local queue_name = KEYS[1]
local max_items = tonumber(ARGV[1]) or 1

-- Obtenir les items avec le score le plus faible (haute priorité)
local items = redis.call('ZRANGE', queue_name .. ':queue', 0, max_items - 1, 'WITHSCORES')

if #items == 0 then
    return nil  -- Queue vide
end

local results = {}

-- Traiter chaque item
for i = 1, #items, 2 do
    local item_id = items[i]
    local score = items[i + 1]

    -- Récupérer les données
    local item_key = queue_name .. ':items:' .. item_id
    local item_data = redis.call('GET', item_key)

    if item_data then
        table.insert(results, {
            id = item_id,
            data = item_data,
            score = score
        })

        -- Supprimer de la queue et les données
        redis.call('ZREM', queue_name .. ':queue', item_id)
        redis.call('DEL', item_key)

        -- Stats
        redis.call('HINCRBY', queue_name .. ':stats', 'total_popped', 1)
    end
end

return results
```

### 2. Distributed Counter avec sharding

```lua
-- distributed_counter_incr.lua
-- Implémente un compteur distribué sur plusieurs shards
-- Pour haute performance sous forte contention
-- KEYS[1] : counter name
-- ARGV[1] : increment value
-- ARGV[2] : number of shards

local counter_name = KEYS[1]
local increment = tonumber(ARGV[1])
local num_shards = tonumber(ARGV[2]) or 10

-- Choisir un shard aléatoirement (basé sur le temps)
local time = redis.call('TIME')
local shard = (time[1] + time[2]) % num_shards

-- Incrémenter le shard
local shard_key = counter_name .. ':shard:' .. shard
local new_value = redis.call('INCRBY', shard_key, increment)

return {
    shard = shard,
    shard_value = new_value
}
```

```lua
-- distributed_counter_get.lua
-- Lit la valeur totale du compteur distribué
-- KEYS[1] : counter name
-- ARGV[1] : number of shards

local counter_name = KEYS[1]
local num_shards = tonumber(ARGV[1]) or 10

local total = 0

-- Sommer tous les shards
for i = 0, num_shards - 1 do
    local shard_key = counter_name .. ':shard:' .. i
    local value = tonumber(redis.call('GET', shard_key)) or 0
    total = total + value
end

return total
```

## Comparaison Lua vs Alternatives

### Tableau récapitulatif

| Critère | Lua Script | MULTI/EXEC | WATCH | Redis Functions (7+) |
|---------|-----------|------------|-------|---------------------|
| **Atomicité** | ✅ Garantie | ✅ Garantie | ✅ Avec retry | ✅ Garantie |
| **Logique complexe** | ✅ Illimitée | ❌ Limitée | ⚠️ Difficile | ✅ Illimitée |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Retry nécessaire** | ❌ Non | ❌ Non | ✅ Oui | ❌ Non |
| **Persistance** | ⚠️ Cache SHA | ❌ Non | ❌ Non | ✅ Persisté |
| **Debugging** | ⚠️ Limité | ✅ Simple | ⚠️ Complexe | ✅ Meilleur |
| **Cluster support** | ⚠️ Limité | ✅ Bon | ⚠️ Limité | ✅ Excellent |
| **Maintenance** | ⚠️ Scripts séparés | ✅ Code client | ⚠️ Code client | ✅ Bibliothèques |

### Quand utiliser Lua ?

✅ **Utilisez Lua quand** :
- Vous avez besoin de logique conditionnelle complexe
- Les opérations doivent être 100% atomiques
- Vous voulez réduire drastiquement les RTT
- Redis < 7.0 (Functions pas disponible)
- Performance critique

❌ **N'utilisez PAS Lua quand** :
- L'opération est très simple (une seule commande suffit)
- Le script serait très long (> 50ms d'exécution)
- Vous avez besoin de randomness ou accès réseau
- Redis 7+ disponible (préférer Functions)

## Bonnes pratiques de production

### Checklist

```
☐ Le script s'exécute en < 50ms dans le pire cas
☐ Pas de commandes O(n) sur de grandes collections
☐ Limite d'itérations imposée dans les boucles
☐ Gestion d'erreur avec pcall pour les opérations risquées
☐ Logging avec redis.log() pour le debugging
☐ Tests unitaires complets
☐ Tests de charge/concurrence
☐ Documentation des KEYS et ARGV
☐ Validation des paramètres en début de script
☐ Utilisation de EVALSHA pour éviter d'envoyer le script à chaque fois
```

### Template de script production

```lua
-- script_template.lua
-- Description : [Description du script]
-- Author : [Nom]
-- Date : [Date]
--
-- Parameters:
--   KEYS[1] : [description]
--   ARGV[1] : [description]
--
-- Returns:
--   [description du retour]
--
-- Errors:
--   ERR_CODE_1 : [description]
--   ERR_CODE_2 : [description]

-- === Configuration ===
local MAX_ITERATIONS = 10000
local TIMEOUT_MS = 50

-- === Helper Functions ===
local function validate_params()
    if #KEYS < 1 then
        return redis.error_reply('ERR at least 1 key required')
    end
    if #ARGV < 1 then
        return redis.error_reply('ERR at least 1 argument required')
    end
    return true
end

local function log_debug(message)
    redis.log(redis.LOG_NOTICE, '[SCRIPT] ' .. message)
end

-- === Main Logic ===
local function main()
    -- Validation
    local valid = validate_params()
    if type(valid) == 'table' and valid.err then
        return valid
    end

    -- Extraction des paramètres
    local key = KEYS[1]
    local value = ARGV[1]

    log_debug('Processing key: ' .. key)

    -- Logique métier
    -- ...

    return {success = true}
end

-- === Entry Point ===
return main()
```

## Conclusion

Les scripts Lua transforment Redis en une plateforme programmable puissante, capable d'exécuter de la logique métier complexe de manière **atomique et performante**. Les avantages sont considérables :
- Élimination des race conditions
- Réduction drastique de la latence réseau
- Centralisation de la logique métier

Cependant, ils nécessitent une utilisation disciplinée :
- Limiter la durée d'exécution
- Éviter les commandes bloquantes O(n)
- Tester exhaustivement avant production

Pour Redis 7+, considérez **Redis Functions** (section 7.4) qui offre une meilleure organisation, persistance et support des clusters.

---

**📚 Points clés à retenir** :
- Les scripts Lua s'exécutent **atomiquement** et **bloquent** Redis
- Utiliser `redis.call()` pour propager les erreurs, `redis.pcall()` pour les capturer
- Distinguer **KEYS[]** (clés Redis) de **ARGV[]** (arguments)
- Limiter la durée à **< 50ms** pour ne pas impacter les autres clients
- Utiliser **EVALSHA** pour éviter d'envoyer le script à chaque exécution
- Toujours avoir une **condition de sortie** dans les boucles

**🔜 Prochaine section** : [7.4 Redis Functions : L'évolution du scripting](./04-redis-functions-evolution-scripting.md)

⏭️ [Redis Functions (Redis 7+) : L'évolution du scripting](/07-atomicite-programmabilite/04-redis-functions-evolution-scripting.md)
