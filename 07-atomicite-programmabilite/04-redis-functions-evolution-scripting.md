🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4 Redis Functions (Redis 7+) : L'évolution du scripting

## Introduction : Pourquoi Redis Functions ?

Redis 7.0 (mars 2022) a introduit **Redis Functions**, une évolution majeure du scripting Lua qui adresse les limitations des scripts EVAL/EVALSHA. Cette nouvelle approche transforme Redis en une plateforme encore plus robuste pour la logique métier serveur-side.

### Le problème avec les scripts Lua traditionnels

```python
# Approche EVAL/EVALSHA classique - Problèmes multiples

# 1. Scripts non persistés
script = """
    local value = redis.call('GET', KEYS[1])
    return value
"""
# ⚠️ Le script doit être rechargé après chaque redémarrage de Redis

# 2. Identification par SHA1
sha = r.script_load(script)
# SHA: "6b1bf486c81ceb7edf3c093f4c48582e38c0e791"
# ⚠️ Difficile à mémoriser, à débugger, peu lisible

# 3. Gestion manuelle du cache
try:
    result = r.evalsha(sha, 1, "mykey")
except redis.exceptions.NoScriptError:
    # Script expiré du cache, recharger
    sha = r.script_load(script)
    result = r.evalsha(sha, 1, "mykey")

# 4. Organisation difficile
# ⚠️ Un fichier par script, pas de notion de bibliothèque
# ⚠️ Pas de versioning intégré
```

### La solution : Redis Functions

```lua
#!lua name=mylib

-- 1. Bibliothèque nommée et persistée
-- Survit aux redémarrages de Redis

-- 2. Fonctions nommées avec métadonnées
redis.register_function{
    function_name = 'get_value',
    callback = function(keys, args)
        return redis.call('GET', keys[1])
    end,
    flags = { 'no-writes' }  -- Métadonnées
}

-- 3. Organisation en bibliothèque
redis.register_function{
    function_name = 'set_value',
    callback = function(keys, args)
        redis.call('SET', keys[1], args[1])
        return 'OK'
    end
}
```

**Appel depuis Python** :
```python
# Pas de gestion de SHA1, pas de cache à gérer
result = r.fcall('get_value', 1, 'mykey')
# ✅ Simple, lisible, persisté
```

### Avantages de Redis Functions

| Aspect | Lua Scripts (EVAL) | Redis Functions |
|--------|-------------------|-----------------|
| **Persistance** | ❌ Non (cache volatile) | ✅ Survit aux redémarrages |
| **Identification** | SHA1 illisible | Noms explicites |
| **Organisation** | Scripts isolés | Bibliothèques |
| **Versioning** | Manuel | Intégré |
| **Métadonnées** | Aucune | Flags, descriptions |
| **Debugging** | Difficile | Facilité (noms, logs) |
| **Cluster** | Limité | Support amélioré |
| **AOF/RDB** | Non sauvegardé | Sauvegardé |
| **Réplication** | Non répliqué | Répliqué |

## Anatomie d'une Redis Function

### Structure de base

```lua
#!lua name=library_name

-- Déclaration de la bibliothèque
-- name: nom unique de la bibliothèque

-- Fonctions locales (helpers, non exposées)
local function helper_function(value)
    return value * 2
end

-- Fonction publique enregistrée
local function my_function(keys, args)
    local key = keys[1]
    local value = args[1]

    -- Utiliser le helper
    local result = helper_function(tonumber(value))

    redis.call('SET', key, result)
    return result
end

-- Enregistrement de la fonction
redis.register_function{
    function_name = 'my_function',
    callback = my_function,
    flags = { 'no-cluster' },  -- Optionnel
    description = 'Doubles a value and stores it'  -- Optionnel
}

-- On peut enregistrer plusieurs fonctions dans la même bibliothèque
redis.register_function('another_function', function(keys, args)
    return 'hello'
end)
```

### Chargement d'une bibliothèque

```python
import redis

r = redis.Redis(decode_responses=True)

# Code de la bibliothèque
library_code = """
#!lua name=mylib

local function double_value(keys, args)
    local value = tonumber(args[1])
    redis.call('SET', keys[1], value * 2)
    return value * 2
end

redis.register_function('double_value', double_value)
"""

# Chargement de la bibliothèque
r.function_load(library_code)

# Appel de la fonction
result = r.fcall('double_value', 1, 'mykey', 5)
print(result)  # 10
```

### Gestion des bibliothèques

```python
# Lister toutes les bibliothèques
libraries = r.function_list()
for lib in libraries:
    print(f"Library: {lib['library_name']}")
    for func in lib['functions']:
        print(f"  - {func['name']}: {func.get('description', 'No description')}")

# Obtenir le code d'une bibliothèque
code = r.function_dump()  # Sérialisation binaire
# Utile pour backup/restore

# Supprimer une bibliothèque
r.function_delete('mylib')

# Recharger (flush + load)
r.function_flush()  # Supprime toutes les bibliothèques
r.function_load(library_code)

# Restaurer depuis un dump
r.function_restore(code)
```

## Syntaxe avancée et patterns

### Pattern 1 : Bibliothèque complète avec helpers

```lua
#!lua name=ecommerce

-- === Helpers (non exposés) ===

local function validate_positive(value)
    if not value or tonumber(value) <= 0 then
        return false, 'Value must be positive'
    end
    return true, tonumber(value)
end

local function get_timestamp()
    local time = redis.call('TIME')
    return tonumber(time[1])
end

local function log_operation(operation, details)
    redis.log(redis.LOG_NOTICE,
        string.format('[ECOMMERCE] %s: %s', operation, details))
end

-- === Fonctions publiques ===

local function purchase_product(keys, args)
    --[[
    Effectue un achat avec validation de stock

    KEYS[1]: product:ID:stock
    KEYS[2]: product:ID:sold
    KEYS[3]: user:ID:orders
    ARGS[1]: user_id
    ARGS[2]: quantity
    ARGS[3]: price
    ]]

    local stock_key = keys[1]
    local sold_key = keys[2]
    local orders_key = keys[3]

    local user_id = args[1]
    local quantity = args[2]
    local price = args[3]

    -- Validation
    local valid, qty = validate_positive(quantity)
    if not valid then
        return redis.error_reply(qty)
    end

    -- Vérifier le stock
    local stock = tonumber(redis.call('GET', stock_key)) or 0

    if stock < qty then
        log_operation('PURCHASE_FAILED',
            string.format('Insufficient stock: %d < %d', stock, qty))
        return {
            success = false,
            error = 'insufficient_stock',
            available = stock,
            requested = qty
        }
    end

    -- Transaction
    local new_stock = redis.call('DECRBY', stock_key, qty)
    redis.call('INCRBY', sold_key, qty)

    -- Créer la commande
    local order_id = redis.call('INCR', 'global:order_counter')
    local timestamp = get_timestamp()

    redis.call('HSET', string.format('order:%d', order_id),
        'user_id', user_id,
        'quantity', qty,
        'price', price,
        'total', qty * tonumber(price),
        'status', 'completed',
        'timestamp', timestamp
    )

    -- Ajouter à l'historique de l'utilisateur
    redis.call('ZADD', orders_key, timestamp, order_id)

    log_operation('PURCHASE_SUCCESS',
        string.format('User %s bought %d items', user_id, qty))

    return {
        success = true,
        order_id = order_id,
        remaining_stock = new_stock,
        total_price = qty * tonumber(price)
    }
end

local function cancel_order(keys, args)
    --[[
    Annule une commande et restitue le stock

    KEYS[1]: order:ID
    KEYS[2]: product:ID:stock
    KEYS[3]: product:ID:sold
    ARGS[1]: order_id
    ]]

    local order_key = keys[1]
    local stock_key = keys[2]
    local sold_key = keys[3]
    local order_id = args[1]

    -- Vérifier que la commande existe
    local exists = redis.call('EXISTS', order_key)
    if exists == 0 then
        return redis.error_reply('Order not found')
    end

    -- Récupérer les détails
    local order = redis.call('HGETALL', order_key)
    local order_data = {}
    for i = 1, #order, 2 do
        order_data[order[i]] = order[i + 1]
    end

    -- Vérifier le statut
    if order_data.status ~= 'completed' then
        return redis.error_reply('Order already cancelled or processed')
    end

    local quantity = tonumber(order_data.quantity)

    -- Restituer le stock
    redis.call('INCRBY', stock_key, quantity)
    redis.call('DECRBY', sold_key, quantity)

    -- Mettre à jour le statut
    redis.call('HSET', order_key,
        'status', 'cancelled',
        'cancelled_at', get_timestamp()
    )

    log_operation('ORDER_CANCELLED',
        string.format('Order %s cancelled, stock restored: %d', order_id, quantity))

    return {
        success = true,
        order_id = order_id,
        stock_restored = quantity
    }
end

local function get_product_stats(keys, args)
    --[[
    Récupère les statistiques d'un produit

    KEYS[1]: product:ID:stock
    KEYS[2]: product:ID:sold
    ]]

    local stock_key = keys[1]
    local sold_key = keys[2]

    local stock = tonumber(redis.call('GET', stock_key)) or 0
    local sold = tonumber(redis.call('GET', sold_key)) or 0

    return {
        stock = stock,
        sold = sold,
        total = stock + sold,
        availability_rate = (stock + sold > 0) and (stock / (stock + sold) * 100) or 0
    }
end

-- === Enregistrement des fonctions ===

redis.register_function{
    function_name = 'purchase_product',
    callback = purchase_product,
    flags = {},
    description = 'Purchase a product with stock validation'
}

redis.register_function{
    function_name = 'cancel_order',
    callback = cancel_order,
    flags = {},
    description = 'Cancel an order and restore stock'
}

redis.register_function{
    function_name = 'get_product_stats',
    callback = get_product_stats,
    flags = { 'no-writes' },
    description = 'Get product statistics'
}
```

**Utilisation en Python** :

```python
class EcommerceLibrary:
    """
    Wrapper Python pour la bibliothèque ecommerce Redis Functions
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self._ensure_library_loaded()

    def _ensure_library_loaded(self):
        """Charge la bibliothèque si elle n'existe pas"""
        try:
            libs = self.redis.function_list(library='ecommerce')
            if not libs:
                self._load_library()
        except:
            self._load_library()

    def _load_library(self):
        """Charge le code de la bibliothèque"""
        library_code = """
        #!lua name=ecommerce
        -- [Code complet ci-dessus]
        """
        try:
            self.redis.function_load(library_code, replace=True)
        except redis.exceptions.ResponseError as e:
            if 'Library already exists' in str(e):
                # Utiliser REPLACE pour mettre à jour
                self.redis.function_load(library_code, replace=True)
            else:
                raise

    def purchase_product(self, product_id, user_id, quantity, price):
        """
        Achète un produit
        """
        result = self.redis.fcall(
            'purchase_product',
            3,
            f'product:{product_id}:stock',
            f'product:{product_id}:sold',
            f'user:{user_id}:orders',
            user_id,
            quantity,
            price
        )
        return self._parse_result(result)

    def cancel_order(self, order_id, product_id):
        """
        Annule une commande
        """
        result = self.redis.fcall(
            'cancel_order',
            3,
            f'order:{order_id}',
            f'product:{product_id}:stock',
            f'product:{product_id}:sold',
            order_id
        )
        return self._parse_result(result)

    def get_product_stats(self, product_id):
        """
        Récupère les statistiques d'un produit
        """
        result = self.redis.fcall_ro(  # Read-only
            'get_product_stats',
            2,
            f'product:{product_id}:stock',
            f'product:{product_id}:sold'
        )
        return self._parse_result(result)

    def _parse_result(self, result):
        """Parse le résultat Redis en dict Python"""
        if isinstance(result, list) and len(result) > 0:
            # Convertir en dict si c'est un tableau de paires clé-valeur
            if len(result) % 2 == 0:
                return {result[i]: result[i+1] for i in range(0, len(result), 2)}
        return result

# Test
r = redis.Redis(decode_responses=True)
ecommerce = EcommerceLibrary(r)

# Initialiser des données de test
r.set('product:laptop:stock', 10)
r.set('product:laptop:sold', 0)

# Achat
result = ecommerce.purchase_product('laptop', 'user:123', 2, 999.99)
print("Purchase:", result)
# {'success': True, 'order_id': 1, 'remaining_stock': 8, 'total_price': 1999.98}

# Stats
stats = ecommerce.get_product_stats('laptop')
print("Stats:", stats)
# {'stock': 8, 'sold': 2, 'total': 10, 'availability_rate': 80.0}

# Annulation
cancel = ecommerce.cancel_order(1, 'laptop')
print("Cancel:", cancel)
# {'success': True, 'order_id': 1, 'stock_restored': 2}
```

## Flags et métadonnées

### Types de flags disponibles

```lua
redis.register_function{
    function_name = 'my_function',
    callback = my_callback,
    flags = {
        'no-writes',      -- Fonction en lecture seule
        'allow-oom',      -- Autorisée même si Redis est en OOM
        'allow-stale',    -- Autorisée sur replica en mode stale
        'no-cluster',     -- Ne peut pas tourner sur cluster
        'allow-cross-slot-keys'  -- Autorise les clés sur différents slots (cluster)
    }
}
```

### Détail des flags

#### 1. no-writes (Read-Only)

```lua
#!lua name=analytics

-- Fonction d'analyse en lecture seule
redis.register_function{
    function_name = 'get_user_analytics',
    callback = function(keys, args)
        local user_key = keys[1]

        -- Seulement des lectures
        local profile = redis.call('HGETALL', user_key)
        local orders_count = redis.call('ZCARD', user_key .. ':orders')
        local last_login = redis.call('GET', user_key .. ':last_login')

        return {
            profile = profile,
            orders_count = orders_count,
            last_login = last_login
        }
    end,
    flags = { 'no-writes' }  -- Pas d'écriture possible
}
```

**Avantages** :
- Peut utiliser `FCALL_RO` (optimisé pour lecture)
- Peut s'exécuter sur des replicas
- Indication claire d'intention

```python
# Appel read-only (peut être routé vers un replica)
result = r.fcall_ro('get_user_analytics', 1, 'user:123')
```

#### 2. allow-oom (Out of Memory)

```lua
#!lua name=emergency

-- Fonction qui doit fonctionner même en OOM
redis.register_function{
    function_name = 'emergency_cleanup',
    callback = function(keys, args)
        local pattern = args[1]
        local deleted = 0

        local cursor = "0"
        repeat
            local result = redis.call('SCAN', cursor, 'MATCH', pattern)
            cursor = result[1]
            local keys_found = result[2]

            for _, key in ipairs(keys_found) do
                redis.call('DEL', key)
                deleted = deleted + 1
            end
        until cursor == "0"

        return deleted
    end,
    flags = { 'allow-oom' }  -- Fonctionne même si Redis est plein
}
```

#### 3. no-cluster

```lua
#!lua name=global_operations

-- Fonction qui ne peut pas tourner sur un cluster
-- (car elle accède à plusieurs slots)
redis.register_function{
    function_name = 'global_stats',
    callback = function(keys, args)
        -- Cette fonction nécessite un accès global
        -- incompatible avec le sharding du cluster
        local total_keys = redis.call('DBSIZE')
        local info = redis.call('INFO', 'memory')

        return {
            total_keys = total_keys,
            memory_info = info
        }
    end,
    flags = { 'no-cluster' }  -- Échoue sur Redis Cluster
}
```

#### 4. allow-cross-slot-keys (Cluster)

```lua
#!lua name=cluster_safe

-- Fonction qui peut accéder à des clés sur différents slots
redis.register_function{
    function_name = 'multi_slot_operation',
    callback = function(keys, args)
        -- Peut accéder à keys[1] et keys[2] même s'ils sont sur des slots différents
        local value1 = redis.call('GET', keys[1])
        local value2 = redis.call('GET', keys[2])

        return {value1, value2}
    end,
    flags = { 'allow-cross-slot-keys' }
}
```

## Versioning et mises à jour

### Pattern de versioning

```lua
#!lua name=mylib_v2

-- Version 2.0 de la bibliothèque
-- Inclut des améliorations et nouvelles fonctionnalités

local VERSION = "2.0.0"

local function get_version(keys, args)
    return VERSION
end

local function my_function_v2(keys, args)
    -- Nouvelle implémentation avec fonctionnalités supplémentaires
    local key = keys[1]
    local value = args[1]

    -- Nouvelle logique...
    redis.call('SET', key, value)

    -- Logger la version utilisée
    redis.log(redis.LOG_NOTICE, 'Using mylib version: ' .. VERSION)

    return {version = VERSION, result = 'OK'}
end

redis.register_function('get_version', get_version)
redis.register_function('my_function', my_function_v2)
```

### Stratégie de mise à jour (Blue-Green)

```python
class FunctionDeployer:
    """
    Déploiement sécurisé de nouvelles versions de fonctions
    """

    def __init__(self, redis_client):
        self.redis = redis_client

    def deploy_new_version(self, library_name, new_code, test_func=None):
        """
        Déploie une nouvelle version avec rollback possible

        Args:
            library_name: Nom de la bibliothèque
            new_code: Code de la nouvelle version
            test_func: Fonction de test optionnelle
        """
        # 1. Backup de la version actuelle
        print("📦 Backup de la version actuelle...")
        try:
            current_dump = self.redis.function_dump()
            current_libs = self.redis.function_list(library=library_name)
        except:
            current_dump = None
            current_libs = []

        try:
            # 2. Charger la nouvelle version
            print("🚀 Chargement de la nouvelle version...")
            self.redis.function_load(new_code, replace=True)

            # 3. Tests de validation
            if test_func:
                print("🧪 Exécution des tests...")
                if not test_func(self.redis):
                    raise Exception("Tests échoués")

            print("✅ Déploiement réussi!")
            return {'success': True, 'version': 'deployed'}

        except Exception as e:
            print(f"❌ Erreur détectée: {e}")

            # 4. Rollback si erreur
            if current_dump:
                print("🔄 Rollback vers la version précédente...")
                self.redis.function_flush()
                self.redis.function_restore(current_dump)
                print("↩️  Rollback effectué")

            return {'success': False, 'error': str(e), 'rolled_back': True}

    def get_library_info(self, library_name):
        """Obtient les infos sur une bibliothèque"""
        libs = self.redis.function_list(library=library_name)
        if libs:
            lib = libs[0]
            return {
                'name': lib['library_name'],
                'functions': [f['name'] for f in lib['functions']],
                'function_count': len(lib['functions'])
            }
        return None

# Utilisation
deployer = FunctionDeployer(r)

# Nouvelle version du code
new_code = """
#!lua name=mylib_v2
-- [code de la nouvelle version]
"""

# Fonction de test
def test_new_version(redis_client):
    try:
        # Tester la nouvelle fonction
        result = redis_client.fcall('my_function', 1, 'test_key', 'test_value')
        return result is not None
    except Exception as e:
        print(f"Test failed: {e}")
        return False

# Déploiement avec tests
result = deployer.deploy_new_version('mylib_v2', new_code, test_new_version)
print(result)
```

## Comparaison approfondie : EVAL vs Functions

### Exemple identique avec les deux approches

```lua
-- === Approche EVAL (Scripts Lua classiques) ===

-- Script doit être envoyé ou chargé avant utilisation
local increment_script = [[
    local key = KEYS[1]
    local amount = tonumber(ARGV[1])
    local max = tonumber(ARGV[2])

    local current = tonumber(redis.call('GET', key)) or 0

    if current + amount > max then
        return redis.error_reply('Would exceed maximum')
    end

    return redis.call('INCRBY', key, amount)
]]
```

```python
# Utilisation EVAL
sha = r.script_load(increment_script)

try:
    result = r.evalsha(sha, 1, 'counter', 5, 100)
except redis.exceptions.NoScriptError:
    # Script pas dans le cache, recharger
    sha = r.script_load(increment_script)
    result = r.evalsha(sha, 1, 'counter', 5, 100)
```

```lua
-- === Approche Redis Functions ===

#!lua name=counters

local function increment_with_max(keys, args)
    local key = keys[1]
    local amount = tonumber(args[1])
    local max = tonumber(args[2])

    local current = tonumber(redis.call('GET', key)) or 0

    if current + amount > max then
        return redis.error_reply('Would exceed maximum')
    end

    return redis.call('INCRBY', key, amount)
end

redis.register_function{
    function_name = 'increment_with_max',
    callback = increment_with_max,
    description = 'Increment with maximum value check'
}
```

```python
# Utilisation Functions
# Pas de gestion de cache nécessaire
result = r.fcall('increment_with_max', 1, 'counter', 5, 100)
# ✅ Simple, pas de NoScriptError possible
```

### Tableau comparatif détaillé

| Critère | EVAL/EVALSHA | Redis Functions |
|---------|--------------|-----------------|
| **Persistance** | Cache volatile | Persisté (AOF/RDB) |
| **Après redémarrage** | ❌ Rechargement nécessaire | ✅ Automatiquement disponible |
| **Identification** | SHA1 (ex: 6b1bf486c8...) | Nom lisible (ex: increment_with_max) |
| **Organisation** | Scripts individuels | Bibliothèques logiques |
| **Versioning** | Manuel (nommage de variables) | Intégré (replace, libraries) |
| **Réplication** | ❌ Non répliqué | ✅ Répliqué automatiquement |
| **Cluster** | Limité | Support amélioré |
| **Métadonnées** | Aucune | Flags, descriptions |
| **Debugging** | SHA1 dans les logs | Nom de fonction dans les logs |
| **Hot reload** | Possible via SCRIPT LOAD | Possible via REPLACE |
| **Backup** | ❌ Scripts externes | ✅ Inclus dans FUNCTION DUMP |
| **Performance** | Légèrement plus rapide | Comparable |
| **Complexité** | Gestion manuelle du cache | Automatique |
| **Cas d'usage** | Scripts ponctuels/prototypes | Production, long terme |

## Cas d'usage avancés

### 1. Système de workflow avec états

```lua
#!lua name=workflow

-- Système de workflow avec transitions d'états validées

local VALID_TRANSITIONS = {
    pending = { approved = true, rejected = true },
    approved = { processing = true, cancelled = true },
    processing = { completed = true, failed = true },
    rejected = { pending = true },
    completed = {},
    failed = { pending = true },
    cancelled = {}
}

local function is_valid_transition(from_state, to_state)
    if not VALID_TRANSITIONS[from_state] then
        return false, 'Invalid source state'
    end

    if not VALID_TRANSITIONS[from_state][to_state] then
        return false, string.format('Invalid transition: %s -> %s', from_state, to_state)
    end

    return true
end

local function transition_workflow(keys, args)
    --[[
    Transition d'état avec validation

    KEYS[1]: workflow:ID
    KEYS[2]: workflow:ID:history
    ARGS[1]: new_state
    ARGS[2]: user_id
    ARGS[3]: comment
    ]]

    local workflow_key = keys[1]
    local history_key = keys[2]
    local new_state = args[1]
    local user_id = args[2]
    local comment = args[3] or ''

    -- Récupérer l'état actuel
    local current_state = redis.call('HGET', workflow_key, 'state')

    if not current_state then
        return redis.error_reply('Workflow not found')
    end

    -- Valider la transition
    local valid, error_msg = is_valid_transition(current_state, new_state)
    if not valid then
        return redis.error_reply(error_msg)
    end

    -- Effectuer la transition
    local timestamp = redis.call('TIME')[1]

    redis.call('HSET', workflow_key,
        'state', new_state,
        'updated_by', user_id,
        'updated_at', timestamp
    )

    -- Enregistrer dans l'historique
    local history_entry = string.format('%s|%s|%s|%s|%s',
        timestamp, current_state, new_state, user_id, comment)

    redis.call('LPUSH', history_key, history_entry)
    redis.call('LTRIM', history_key, 0, 99)  -- Garder les 100 dernières entrées

    redis.log(redis.LOG_NOTICE,
        string.format('Workflow transition: %s -> %s by %s',
            current_state, new_state, user_id))

    return {
        success = true,
        previous_state = current_state,
        new_state = new_state,
        timestamp = timestamp
    }
end

local function get_workflow_state(keys, args)
    local workflow_key = keys[1]

    local workflow = redis.call('HGETALL', workflow_key)
    local workflow_data = {}
    for i = 1, #workflow, 2 do
        workflow_data[workflow[i]] = workflow[i + 1]
    end

    return workflow_data
end

local function get_workflow_history(keys, args)
    local history_key = keys[1]
    local limit = tonumber(args[1]) or 10

    return redis.call('LRANGE', history_key, 0, limit - 1)
end

redis.register_function{
    function_name = 'transition_workflow',
    callback = transition_workflow,
    description = 'Transition workflow state with validation'
}

redis.register_function{
    function_name = 'get_workflow_state',
    callback = get_workflow_state,
    flags = { 'no-writes' },
    description = 'Get current workflow state'
}

redis.register_function{
    function_name = 'get_workflow_history',
    callback = get_workflow_history,
    flags = { 'no-writes' },
    description = 'Get workflow transition history'
}
```

**Classe Python** :

```python
class WorkflowManager:
    """
    Gestion de workflows avec Redis Functions
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self._ensure_loaded()

    def _ensure_loaded(self):
        """Charge la bibliothèque workflow si nécessaire"""
        # [Code de chargement similaire aux exemples précédents]
        pass

    def create_workflow(self, workflow_id, initial_data):
        """Crée un nouveau workflow"""
        workflow_key = f'workflow:{workflow_id}'

        self.redis.hset(workflow_key, mapping={
            'id': workflow_id,
            'state': 'pending',
            'created_at': int(time.time()),
            **initial_data
        })

        return {'workflow_id': workflow_id, 'state': 'pending'}

    def transition(self, workflow_id, new_state, user_id, comment=''):
        """Change l'état du workflow"""
        try:
            result = self.redis.fcall(
                'transition_workflow',
                2,
                f'workflow:{workflow_id}',
                f'workflow:{workflow_id}:history',
                new_state,
                user_id,
                comment
            )
            return self._parse_result(result)
        except redis.exceptions.ResponseError as e:
            return {'success': False, 'error': str(e)}

    def get_state(self, workflow_id):
        """Récupère l'état actuel"""
        result = self.redis.fcall_ro(
            'get_workflow_state',
            1,
            f'workflow:{workflow_id}'
        )
        return self._parse_result(result)

    def get_history(self, workflow_id, limit=10):
        """Récupère l'historique des transitions"""
        history = self.redis.fcall_ro(
            'get_workflow_history',
            1,
            f'workflow:{workflow_id}:history',
            limit
        )

        # Parser l'historique
        parsed = []
        for entry in history:
            parts = entry.split('|')
            parsed.append({
                'timestamp': int(parts[0]),
                'from_state': parts[1],
                'to_state': parts[2],
                'user_id': parts[3],
                'comment': parts[4] if len(parts) > 4 else ''
            })

        return parsed

    def _parse_result(self, result):
        if isinstance(result, list) and len(result) % 2 == 0:
            return {result[i]: result[i+1] for i in range(0, len(result), 2)}
        return result

# Utilisation
wf = WorkflowManager(r)

# Créer un workflow
wf.create_workflow('order-123', {
    'customer': 'john@example.com',
    'amount': 99.99
})

# Transitions
wf.transition('order-123', 'approved', 'admin-1', 'Validated by admin')
wf.transition('order-123', 'processing', 'system', 'Auto-started')
wf.transition('order-123', 'completed', 'system', 'Shipped')

# Vérifier l'état
state = wf.get_state('order-123')
print("Current state:", state['state'])

# Historique
history = wf.get_history('order-123')
for entry in history:
    print(f"{entry['timestamp']}: {entry['from_state']} -> {entry['to_state']} by {entry['user_id']}")
```

### 2. Cache avec invalidation intelligente

```lua
#!lua name=smart_cache

-- Cache avec invalidation basée sur les dépendances

local function set_with_dependencies(keys, args)
    --[[
    Stocke une valeur avec ses dépendances

    KEYS[1]: cache:key
    ARGV[1]: value
    ARGV[2..n]: dependency keys (JSON array string)
    ]]

    local cache_key = keys[1]
    local value = args[1]
    local dependencies_json = args[2]

    -- Parser les dépendances (format simple: "dep1,dep2,dep3")
    local dependencies = {}
    if dependencies_json and dependencies_json ~= '' then
        for dep in string.gmatch(dependencies_json, '([^,]+)') do
            table.insert(dependencies, dep)
        end
    end

    -- Stocker la valeur
    redis.call('SET', cache_key, value)
    redis.call('EXPIRE', cache_key, 3600)  -- TTL par défaut 1h

    -- Stocker les dépendances
    if #dependencies > 0 then
        local deps_key = cache_key .. ':deps'
        redis.call('DEL', deps_key)
        for _, dep in ipairs(dependencies) do
            redis.call('SADD', deps_key, dep)
        end
        redis.call('EXPIRE', deps_key, 3600)

        -- Index inverse : quelles clés dépendent de cette dépendance
        for _, dep in ipairs(dependencies) do
            local reverse_key = 'cache:dep_index:' .. dep
            redis.call('SADD', reverse_key, cache_key)
            redis.call('EXPIRE', reverse_key, 3600)
        end
    end

    return {
        success = true,
        key = cache_key,
        dependencies_count = #dependencies
    }
end

local function invalidate_by_dependency(keys, args)
    --[[
    Invalide toutes les entrées de cache dépendant d'une clé

    KEYS[1]: dependency key
    ]]

    local dependency = keys[1]
    local reverse_key = 'cache:dep_index:' .. dependency

    -- Trouver toutes les clés de cache qui dépendent de cette dependency
    local dependent_keys = redis.call('SMEMBERS', reverse_key)

    if #dependent_keys == 0 then
        return {invalidated = 0}
    end

    -- Invalider chaque clé de cache
    local invalidated = 0
    for _, cache_key in ipairs(dependent_keys) do
        local existed = redis.call('DEL', cache_key)
        if existed == 1 then
            invalidated = invalidated + 1
        end

        -- Supprimer les métadonnées de dépendances
        redis.call('DEL', cache_key .. ':deps')
    end

    -- Nettoyer l'index inverse
    redis.call('DEL', reverse_key)

    redis.log(redis.LOG_NOTICE,
        string.format('Cache invalidation: %d keys for dependency %s',
            invalidated, dependency))

    return {
        invalidated = invalidated,
        dependency = dependency
    }
end

local function get_cached(keys, args)
    --[[
    Récupère une valeur du cache avec validation des dépendances

    KEYS[1]: cache key
    ]]

    local cache_key = keys[1]

    -- Vérifier si la clé existe
    local value = redis.call('GET', cache_key)

    if not value then
        return {hit = false, value = nil}
    end

    -- Vérifier les dépendances
    local deps_key = cache_key .. ':deps'
    local dependencies = redis.call('SMEMBERS', deps_key)

    -- Si des dépendances ont été supprimées, invalider le cache
    for _, dep in ipairs(dependencies) do
        if redis.call('EXISTS', dep) == 0 then
            redis.call('DEL', cache_key, deps_key)
            return {
                hit = false,
                value = nil,
                reason = 'dependency_invalidated',
                invalidated_by = dep
            }
        end
    end

    return {
        hit = true,
        value = value,
        dependencies_count = #dependencies
    }
end

redis.register_function{
    function_name = 'cache_set_with_deps',
    callback = set_with_dependencies,
    description = 'Set cache value with dependencies tracking'
}

redis.register_function{
    function_name = 'cache_invalidate_by_dep',
    callback = invalidate_by_dependency,
    description = 'Invalidate all cache entries depending on a key'
}

redis.register_function{
    function_name = 'cache_get',
    callback = get_cached,
    flags = { 'no-writes' },
    description = 'Get cached value with dependency validation'
}
```

## Migration depuis EVAL vers Functions

### Stratégie de migration

```python
class ScriptMigration:
    """
    Helper pour migrer des scripts EVAL vers Functions
    """

    @staticmethod
    def convert_eval_to_function(script_name, lua_code):
        """
        Convertit un script EVAL en Function

        Args:
            script_name: Nom de la fonction
            lua_code: Code Lua du script EVAL
        """
        # Ajouter le header de bibliothèque
        function_code = f"#!lua name=migrated_{script_name}\n\n"

        # Wrapper la logique dans une fonction
        function_code += f"local function {script_name}(keys, args)\n"

        # Remplacer KEYS et ARGV par keys et args
        converted = lua_code.replace('KEYS[', 'keys[')
        converted = converted.replace('ARGV[', 'args[')

        function_code += "    " + converted.replace('\n', '\n    ')
        function_code += "\nend\n\n"

        # Enregistrer la fonction
        function_code += f"redis.register_function('{script_name}', {script_name})\n"

        return function_code

    @staticmethod
    def parallel_execution(redis_client, function_name, use_function=True):
        """
        Permet d'exécuter en parallèle EVAL et Function pour comparaison
        """
        def decorator(func):
            def wrapper(*args, **kwargs):
                if use_function:
                    # Utiliser Function
                    return redis_client.fcall(function_name, *args, **kwargs)
                else:
                    # Utiliser EVAL (legacy)
                    return func(*args, **kwargs)
            return wrapper
        return decorator

# Exemple de migration
old_script = """
local value = redis.call('GET', KEYS[1])
if tonumber(value) > 10 then
    redis.call('INCR', KEYS[1])
end
return value
"""

# Conversion automatique
migration = ScriptMigration()
new_function = migration.convert_eval_to_function('check_and_increment', old_script)

print(new_function)
# Output:
# #!lua name=migrated_check_and_increment
#
# local function check_and_increment(keys, args)
#     local value = redis.call('GET', keys[1])
#     if tonumber(value) > 10 then
#         redis.call('INCR', keys[1])
#     end
#     return value
# end
#
# redis.register_function('check_and_increment', check_and_increment)
```

### Checklist de migration

```
☐ Identifier tous les scripts EVAL/EVALSHA en production
☐ Grouper les scripts par domaine fonctionnel (futures bibliothèques)
☐ Convertir chaque script en fonction
☐ Ajouter des métadonnées (flags, descriptions)
☐ Tests unitaires complets
☐ Déploiement en parallèle (A/B testing)
☐ Monitoring des deux versions
☐ Basculement progressif (feature flag)
☐ Suppression de l'ancien code EVAL
☐ Documentation mise à jour
```

## Bonnes pratiques de production

### 1. Organisation du code

```
project/
├── redis_functions/
│   ├── libraries/
│   │   ├── auth.lua           # Bibliothèque authentification
│   │   ├── ecommerce.lua      # Bibliothèque e-commerce
│   │   ├── analytics.lua      # Bibliothèque analytics
│   │   └── cache.lua          # Bibliothèque cache
│   ├── tests/
│   │   ├── test_auth.py
│   │   ├── test_ecommerce.py
│   │   └── test_analytics.py
│   ├── deploy.py              # Script de déploiement
│   └── loader.py              # Chargement automatique
```

### 2. Template de bibliothèque production

```lua
#!lua name=mylib
--[[
    Library: mylib
    Version: 1.0.0
    Description: [Description de la bibliothèque]
    Author: [Nom]
    Date: [Date]

    Functions:
        - function_name_1: [description]
        - function_name_2: [description]
]]

-- === Configuration ===
local VERSION = "1.0.0"
local MAX_ITERATIONS = 10000
local DEFAULT_TTL = 3600

-- === Helpers privés ===

local function log(level, message)
    redis.log(level, string.format('[MYLIB v%s] %s', VERSION, message))
end

local function validate_required(value, name)
    if not value or value == '' then
        return redis.error_reply(string.format('ERR %s is required', name))
    end
    return true
end

local function get_timestamp()
    local time = redis.call('TIME')
    return tonumber(time[1])
end

-- === Fonctions publiques ===

local function my_function(keys, args)
    --[[
    Description détaillée de la fonction

    Parameters:
        KEYS[1]: [description]
        ARGS[1]: [description]

    Returns:
        [description du retour]

    Errors:
        ERR_CODE_1: [description]

    Example:
        FCALL my_function 1 mykey arg1
    ]]

    -- Validation
    local valid = validate_required(args[1], 'arg1')
    if type(valid) == 'table' and valid.err then
        return valid
    end

    -- Logique métier
    log(redis.LOG_NOTICE, 'Executing my_function')

    -- ...

    return {success = true, version = VERSION}
end

-- === Enregistrement ===

redis.register_function{
    function_name = 'my_function',
    callback = my_function,
    flags = {},
    description = 'Description courte de la fonction'
}

-- Fonction de version (utile pour debugging)
redis.register_function{
    function_name = 'get_version',
    callback = function(keys, args)
        return VERSION
    end,
    flags = { 'no-writes' },
    description = 'Get library version'
}
```

### 3. Tests automatisés

```python
import unittest
import redis

class TestMyLibFunctions(unittest.TestCase):
    """
    Tests pour la bibliothèque mylib
    """

    @classmethod
    def setUpClass(cls):
        """Charge la bibliothèque une fois pour tous les tests"""
        cls.redis = redis.Redis(decode_responses=True)

        with open('libraries/mylib.lua', 'r') as f:
            library_code = f.read()

        cls.redis.function_load(library_code, replace=True)

    def setUp(self):
        """Nettoie avant chaque test"""
        self.redis.flushdb()

    def test_my_function_success(self):
        """Test cas nominal"""
        result = self.redis.fcall('my_function', 1, 'test_key', 'test_value')
        self.assertIn('success', result)
        self.assertTrue(result['success'])

    def test_my_function_validation(self):
        """Test validation des paramètres"""
        with self.assertRaises(redis.exceptions.ResponseError):
            self.redis.fcall('my_function', 1, 'test_key', '')

    def test_my_function_concurrent(self):
        """Test exécution concurrente"""
        import threading
        results = []

        def call_function():
            result = self.redis.fcall('my_function', 1, 'test_key', 'value')
            results.append(result)

        threads = [threading.Thread(target=call_function) for _ in range(10)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()

        self.assertEqual(len(results), 10)
        self.assertTrue(all(r.get('success') for r in results))

    def test_version(self):
        """Test fonction de version"""
        version = self.redis.fcall_ro('get_version', 0)
        self.assertIsNotNone(version)
        self.assertRegex(version, r'^\d+\.\d+\.\d+$')

if __name__ == '__main__':
    unittest.main()
```

### 4. Monitoring en production

```python
class FunctionMonitor:
    """
    Monitoring des Redis Functions en production
    """

    def __init__(self, redis_client):
        self.redis = redis_client

    def get_function_stats(self):
        """Récupère les statistiques des fonctions"""
        stats = {}

        libraries = self.redis.function_list()

        for lib in libraries:
            lib_name = lib['library_name']
            stats[lib_name] = {
                'function_count': len(lib['functions']),
                'functions': {}
            }

            for func in lib['functions']:
                func_name = func['name']
                stats[lib_name]['functions'][func_name] = {
                    'name': func_name,
                    'description': func.get('description', 'N/A'),
                    'flags': func.get('flags', [])
                }

        return stats

    def health_check(self):
        """Vérifie la santé des fonctions"""
        try:
            libs = self.redis.function_list()
            return {
                'status': 'healthy',
                'library_count': len(libs),
                'total_functions': sum(len(lib['functions']) for lib in libs)
            }
        except Exception as e:
            return {
                'status': 'unhealthy',
                'error': str(e)
            }

    def export_for_backup(self, output_file):
        """Exporte toutes les fonctions pour backup"""
        dump = self.redis.function_dump()

        with open(output_file, 'wb') as f:
            f.write(dump)

        return {'status': 'exported', 'file': output_file}

    def restore_from_backup(self, input_file):
        """Restaure depuis un backup"""
        with open(input_file, 'rb') as f:
            dump = f.read()

        self.redis.function_restore(dump, policy='FLUSH')

        return {'status': 'restored', 'file': input_file}

# Utilisation
monitor = FunctionMonitor(r)

# Stats
stats = monitor.get_function_stats()
print(f"Total libraries: {len(stats)}")

# Health check
health = monitor.health_check()
print(f"Status: {health['status']}")

# Backup
monitor.export_for_backup('/backup/redis_functions.dump')
```

## Conclusion

Redis Functions représente une **évolution majeure** du scripting dans Redis. Les avantages sont significatifs :

✅ **Avantages principaux** :
- Persistance automatique (AOF/RDB)
- Organisation en bibliothèques logiques
- Noms lisibles et explicites
- Métadonnées riches (flags, descriptions)
- Versioning intégré
- Meilleur support du clustering
- Réplication automatique

⚠️ **Points d'attention** :
- Nécessite Redis 7.0+
- Migration depuis EVAL requiert du travail
- Légèrement plus de mémoire (métadonnées)

**Recommandation** : Pour tout nouveau projet avec Redis 7+, utilisez **Redis Functions** plutôt que EVAL. Pour les projets existants, planifiez une migration progressive.

---

**📚 Points clés à retenir** :
- Redis Functions est l'**approche recommandée** pour Redis 7+
- Les fonctions sont **persistées** et **répliquées** automatiquement
- Organisation en **bibliothèques** avec fonctions nommées
- **Flags** pour indiquer les capacités (no-writes, allow-oom, etc.)
- **Versioning** et déploiement facilités
- Migration depuis EVAL nécessite conversion mais apporte de gros bénéfices

**🔜 Prochaine section** : [7.5 Triggers and Functions (Redis Stack)](./05-triggers-and-functions-redis-stack.md)

⏭️ [Triggers and Functions (Redis Stack)](/07-atomicite-programmabilite/05-triggers-and-functions-redis-stack.md)
