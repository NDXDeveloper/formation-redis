🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 7 : Atomicité et programmabilité

## Introduction : Le défi de la concurrence dans Redis

Redis est conçu pour être **ultra-rapide** et peut traiter des dizaines de milliers de commandes par seconde. Cette performance exceptionnelle s'accompagne d'un défi majeur : **comment garantir la cohérence des données lorsque plusieurs clients effectuent des opérations simultanées ?**

### Le problème fondamental : Race Conditions

Considérons un scénario classique d'e-commerce où nous gérons un stock de produits :

```
Situation initiale : stock:product:123 = 5

Client A lit : stock = 5
Client B lit : stock = 5
Client A achète 3 unités → stock = 2
Client B achète 3 unités → stock = 2

Résultat attendu : 5 - 3 - 3 = -1 (rupture de stock)
Résultat obtenu : 2 (données corrompues !)
```

Ce type de problème, appelé **race condition**, peut avoir des conséquences graves :
- Survente de produits en rupture de stock
- Corruption de compteurs (likes, vues, statistiques)
- Incohérence des transactions bancaires
- Duplication de traitements coûteux

### L'architecture single-thread de Redis : Une force... et une contrainte

Redis s'exécute dans un **thread unique** pour traiter les commandes. Cela signifie que :

✅ **Avantages** :
- Chaque commande individuelle est **atomique** par nature
- Pas de locks complexes ni de deadlocks
- Performance prévisible et déterministe

⚠️ **Limitations** :
- Une séquence de plusieurs commandes n'est **pas atomique**
- Entre deux commandes, d'autres clients peuvent intervenir
- Les opérations Read-Modify-Write sont particulièrement vulnérables

## Les solutions d'atomicité dans Redis

Redis propose plusieurs mécanismes pour résoudre ces problèmes de concurrence :

### 1. **Commandes atomiques natives**

Certaines commandes Redis sont conçues pour être atomiques et éviter les race conditions :

```redis
# ❌ MAUVAIS : Non atomique (lecture + écriture séparées)
GET counter
# ... calcul dans le client ...
SET counter 42

# ✅ BON : Atomique
INCR counter              # Incrément atomique
INCRBY counter 10         # Incrément de 10, atomique
GETSET key new_value      # Récupère l'ancienne valeur et set la nouvelle, atomique
```

**Limitation** : Ces commandes couvrent uniquement des cas d'usage simples et prédéfinis.

### 2. **Transactions avec MULTI/EXEC**

Les transactions permettent de grouper plusieurs commandes pour qu'elles soient exécutées de manière isolée :

```redis
MULTI
DECRBY stock:product:123 3
LPUSH orders:pending {order_data}
EXEC
```

**Caractéristiques** :
- Isolation garantie pendant l'exécution
- Pas de rollback automatique en cas d'erreur
- Performance optimale pour des séquences simples
- Limitations sur les opérations conditionnelles

### 3. **Optimistic Locking avec WATCH**

WATCH permet d'implémenter un mécanisme de verrouillage optimiste :

```redis
WATCH stock:product:123
stock = GET stock:product:123
if stock >= 3:
    MULTI
    DECRBY stock:product:123 3
    EXEC  # Échoue si stock a changé depuis WATCH
```

**Cas d'usage** : Idéal pour les opérations conditionnelles où les conflits sont rares.

### 4. **Scripting Lua : Atomicité personnalisée**

Lua permet d'écrire des **scripts personnalisés exécutés atomiquement** sur le serveur Redis :

```lua
-- Script Lua exécuté atomiquement
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1
else
    return 0
end
```

**Avantages** :
- Logique métier complexe exécutée atomiquement
- Réduction du nombre d'aller-retours réseau
- Pas de race conditions possibles

### 5. **Redis Functions (Redis 7+) : L'évolution du scripting**

Redis Functions est la nouvelle génération de programmabilité, introduite dans Redis 7 :

```lua
#!lua name=mylib

local function check_and_decrement(keys, args)
    local stock = redis.call('GET', keys[1])
    if tonumber(stock) >= tonumber(args[1]) then
        redis.call('DECRBY', keys[1], args[1])
        return {ok = true, remaining = stock - args[1]}
    else
        return {ok = false, error = 'Insufficient stock'}
    end
end

redis.register_function('check_and_decrement', check_and_decrement)
```

**Nouveautés** :
- Bibliothèques de fonctions persistantes
- Meilleure organisation du code
- Gestion d'erreur améliorée
- Compatibilité avec les clusters

## Exemple détaillé : Système de réservation atomique

Imaginons un système de réservation de billets pour un événement. Nous devons gérer :
- Un nombre limité de places disponibles
- Des réservations simultanées
- Une fenêtre de temps limitée pour confirmer

### Approche 1 : Script Lua classique

```lua
-- reserve_ticket.lua
-- KEYS[1] = event:stock
-- KEYS[2] = event:reservations
-- ARGV[1] = user_id
-- ARGV[2] = quantity
-- ARGV[3] = reservation_ttl (en secondes)

local stock = tonumber(redis.call('GET', KEYS[1]))
local quantity = tonumber(ARGV[2])

-- Vérification du stock disponible
if not stock or stock < quantity then
    return {err = 'Insufficient tickets', available = stock or 0}
end

-- Génération d'un ID de réservation unique
local reservation_id = redis.call('INCR', 'reservation:counter')
local reservation_key = 'reservation:' .. reservation_id

-- Décrémentation du stock
redis.call('DECRBY', KEYS[1], quantity)

-- Création de la réservation temporaire avec expiration
redis.call('HSET', reservation_key,
    'user_id', ARGV[1],
    'event', KEYS[1],
    'quantity', quantity,
    'status', 'pending',
    'created_at', redis.call('TIME')[1]
)
redis.call('EXPIRE', reservation_key, ARGV[3])

-- Ajout à l'index des réservations de l'événement
redis.call('SADD', KEYS[2], reservation_id)

return {
    ok = true,
    reservation_id = reservation_id,
    remaining_stock = stock - quantity
}
```

**Exécution depuis un client** :

```bash
redis-cli --eval reserve_ticket.lua \
    event:123:stock event:123:reservations , \
    user:456 2 300
```

**Analyse du script** :
1. **Atomicité complète** : Toute la logique s'exécute sans interruption
2. **Vérifications conditionnelles** : Le stock est vérifié avant modification
3. **Opérations multiples** : Stock, réservation, index, tout est cohérent
4. **Gestion des expirations** : TTL automatique pour les réservations non confirmées
5. **Pas de race condition** : Impossible que deux clients réservent les mêmes places

### Approche 2 : Redis Function (Redis 7+)

```lua
#!lua name=ticketing

-- Fonction pour réserver des billets
local function reserve_tickets(keys, args)
    local event_stock_key = keys[1]
    local event_reservations_key = keys[2]
    local user_id = args[1]
    local quantity = tonumber(args[2])
    local ttl = tonumber(args[3])

    -- Validation des paramètres
    if not quantity or quantity <= 0 then
        return redis.error_reply('Invalid quantity')
    end

    -- Vérification du stock avec gestion d'erreur
    local stock = redis.call('GET', event_stock_key)
    stock = stock and tonumber(stock) or 0

    if stock < quantity then
        return {
            status = 'error',
            code = 'INSUFFICIENT_STOCK',
            available = stock,
            requested = quantity
        }
    end

    -- Transaction atomique
    local reservation_id = redis.call('INCR', 'global:reservation_counter')
    local reservation_key = 'reservation:' .. reservation_id

    -- Mise à jour du stock
    local new_stock = redis.call('DECRBY', event_stock_key, quantity)

    -- Création de la réservation
    redis.call('HSET', reservation_key,
        'user_id', user_id,
        'event', event_stock_key,
        'quantity', quantity,
        'status', 'pending',
        'timestamp', redis.call('TIME')[1]
    )

    -- Expiration automatique
    redis.call('EXPIRE', reservation_key, ttl)

    -- Indexation
    redis.call('ZADD', event_reservations_key, redis.call('TIME')[1], reservation_id)

    return {
        status = 'success',
        reservation_id = tostring(reservation_id),
        remaining = new_stock,
        expires_in = ttl
    }
end

-- Fonction pour confirmer une réservation
local function confirm_reservation(keys, args)
    local reservation_id = args[1]
    local reservation_key = 'reservation:' .. reservation_id

    -- Vérification de l'existence
    local exists = redis.call('EXISTS', reservation_key)
    if exists == 0 then
        return redis.error_reply('Reservation not found or expired')
    end

    -- Vérification du statut
    local status = redis.call('HGET', reservation_key, 'status')
    if status ~= 'pending' then
        return redis.error_reply('Reservation already ' .. status)
    end

    -- Confirmation atomique
    redis.call('HSET', reservation_key,
        'status', 'confirmed',
        'confirmed_at', redis.call('TIME')[1]
    )

    -- Suppression de l'expiration (réservation permanente)
    redis.call('PERSIST', reservation_key)

    return {status = 'confirmed', reservation_id = reservation_id}
end

-- Fonction pour annuler une réservation
local function cancel_reservation(keys, args)
    local reservation_id = args[1]
    local reservation_key = 'reservation:' .. reservation_id

    -- Récupération des détails avant suppression
    local details = redis.call('HGETALL', reservation_key)
    if #details == 0 then
        return redis.error_reply('Reservation not found')
    end

    -- Conversion du tableau HGETALL en table
    local reservation = {}
    for i = 1, #details, 2 do
        reservation[details[i]] = details[i + 1]
    end

    -- Restitution du stock seulement si pending
    if reservation.status == 'pending' then
        local event_key = reservation.event
        local quantity = tonumber(reservation.quantity)
        redis.call('INCRBY', event_key, quantity)
    end

    -- Suppression de la réservation
    redis.call('DEL', reservation_key)

    -- Nettoyage de l'index
    local event_reservations = reservation.event .. ':reservations'
    redis.call('ZREM', event_reservations, reservation_id)

    return {
        status = 'cancelled',
        stock_restored = reservation.status == 'pending',
        quantity = reservation.quantity
    }
end

-- Enregistrement des fonctions
redis.register_function{
    function_name = 'reserve_tickets',
    callback = reserve_tickets,
    flags = { 'no-writes' }  -- Ajusté car on écrit
}

redis.register_function{
    function_name = 'confirm_reservation',
    callback = confirm_reservation
}

redis.register_function{
    function_name = 'cancel_reservation',
    callback = cancel_reservation
}
```

**Chargement de la bibliothèque** :

```bash
redis-cli FUNCTION LOAD "#!lua name=ticketing
... [code complet] ...
"
```

**Utilisation** :

```bash
# Réservation
redis-cli FCALL reserve_tickets 2 event:123:stock event:123:reservations user:456 2 300

# Confirmation
redis-cli FCALL confirm_reservation 0 12345

# Annulation
redis-cli FCALL cancel_reservation 0 12345
```

**Avantages de Redis Functions vs Lua Script** :

| Aspect | Lua Script (EVAL) | Redis Functions (FCALL) |
|--------|------------------|------------------------|
| **Persistance** | ❌ Script doit être rechargé | ✅ Chargé une fois, persiste |
| **Organisation** | ❌ Un script = un fichier | ✅ Bibliothèques de fonctions |
| **Naming** | ❌ Hash SHA1 peu lisible | ✅ Noms explicites |
| **Versioning** | ❌ Difficile | ✅ Gestion de versions intégrée |
| **Debugging** | ⚠️ Limité | ✅ Meilleur support |
| **Performance** | ✅ Très rapide | ✅ Très rapide |
| **Cluster** | ⚠️ Limitations | ✅ Meilleur support |

## Patterns avancés : Gestion d'erreurs et robustesse

### Pattern 1 : Gestion des erreurs Lua avec pcall

```lua
#!lua name=robust_operations

local function safe_increment_with_limit(keys, args)
    local key = keys[1]
    local increment = tonumber(args[1])
    local max_value = tonumber(args[2])

    -- Protection contre les erreurs avec pcall
    local success, current_value = pcall(function()
        local val = redis.call('GET', key)
        return val and tonumber(val) or 0
    end)

    if not success then
        return redis.error_reply('Failed to read current value')
    end

    -- Vérification de la limite
    if current_value + increment > max_value then
        return {
            status = 'error',
            code = 'LIMIT_EXCEEDED',
            current = current_value,
            requested = increment,
            limit = max_value
        }
    end

    -- Incrémentation sécurisée
    local new_value = redis.call('INCRBY', key, increment)

    return {
        status = 'success',
        value = new_value,
        remaining = max_value - new_value
    }
end

redis.register_function('safe_increment_with_limit', safe_increment_with_limit)
```

### Pattern 2 : Opération atomique avec rollback manuel

```lua
#!lua name=transaction_patterns

local function transfer_with_rollback(keys, args)
    local from_account = keys[1]
    local to_account = keys[2]
    local amount = tonumber(args[1])

    -- État initial pour rollback potentiel
    local from_balance = tonumber(redis.call('GET', from_account)) or 0

    -- Vérification de fonds suffisants
    if from_balance < amount then
        return redis.error_reply('Insufficient funds')
    end

    -- Début de la transaction
    redis.call('DECRBY', from_account, amount)

    -- Simulation d'une vérification (ex: compte destinataire existe)
    local to_exists = redis.call('EXISTS', to_account)

    if to_exists == 0 then
        -- ROLLBACK manuel : restauration
        redis.call('INCRBY', from_account, amount)
        return redis.error_reply('Destination account does not exist')
    end

    -- Transaction complète
    redis.call('INCRBY', to_account, amount)

    -- Log de la transaction
    local tx_id = redis.call('INCR', 'transaction:counter')
    redis.call('HSET', 'transaction:' .. tx_id,
        'from', from_account,
        'to', to_account,
        'amount', amount,
        'timestamp', redis.call('TIME')[1]
    )

    return {
        status = 'success',
        transaction_id = tx_id,
        from_balance = from_balance - amount,
        to_balance = redis.call('GET', to_account)
    }
end

redis.register_function('transfer_with_rollback', transfer_with_rollback)
```

### Pattern 3 : Opération idempotente

```lua
#!lua name=idempotent_operations

local function process_payment_idempotent(keys, args)
    local payment_id = args[1]
    local amount = tonumber(args[2])
    local account = keys[1]

    local payment_key = 'payment:' .. payment_id

    -- Vérification d'idempotence
    local already_processed = redis.call('EXISTS', payment_key)

    if already_processed == 1 then
        -- Retourner le résultat existant sans retraiter
        local status = redis.call('HGET', payment_key, 'status')
        return {
            status = 'already_processed',
            payment_id = payment_id,
            message = 'Payment already processed',
            original_status = status
        }
    end

    -- Traitement du paiement
    redis.call('INCRBY', account, amount)

    -- Enregistrement pour idempotence
    redis.call('HSET', payment_key,
        'status', 'completed',
        'amount', amount,
        'account', account,
        'timestamp', redis.call('TIME')[1]
    )

    -- Expiration après 24h (configurable)
    redis.call('EXPIRE', payment_key, 86400)

    return {
        status = 'success',
        payment_id = payment_id,
        new_balance = redis.call('GET', account)
    }
end

redis.register_function('process_payment_idempotent', process_payment_idempotent)
```

## Performance et bonnes pratiques

### 1. **Minimiser la complexité algorithmique**

```lua
-- ❌ MAUVAIS : O(n) pour chaque recherche
local function find_user_bad(keys, args)
    local all_users = redis.call('KEYS', 'user:*')  -- ⚠️ KEYS = O(n)
    for _, user_key in ipairs(all_users) do
        local name = redis.call('HGET', user_key, 'name')
        if name == args[1] then
            return user_key
        end
    end
    return nil
end

-- ✅ BON : O(1) avec indexation appropriée
local function find_user_good(keys, args)
    local name = args[1]
    local user_id = redis.call('HGET', 'index:users:by_name', name)
    if user_id then
        return 'user:' .. user_id
    end
    return nil
end
```

### 2. **Limiter la durée d'exécution**

```lua
local function process_batch_with_limit(keys, args)
    local pattern = keys[1]
    local max_iterations = 1000  -- Limite de sécurité
    local processed = 0

    local cursor = "0"
    repeat
        local result = redis.call('SCAN', cursor, 'MATCH', pattern, 'COUNT', 100)
        cursor = result[1]
        local found_keys = result[2]

        for _, key in ipairs(found_keys) do
            redis.call('EXPIRE', key, 3600)
            processed = processed + 1

            -- Protection contre boucle infinie
            if processed >= max_iterations then
                return {
                    status = 'partial',
                    processed = processed,
                    warning = 'Max iterations reached'
                }
            end
        end
    until cursor == "0"

    return {status = 'complete', processed = processed}
end
```

### 3. **Utiliser les commandes optimisées**

```lua
-- ❌ MAUVAIS : Multiple commandes individuelles
local function increment_multiple_bad(keys, args)
    for i, key in ipairs(keys) do
        redis.call('INCR', key)  -- N appels réseau
    end
end

-- ✅ BON : Utilisation de commandes groupées
local function increment_multiple_good(keys, args)
    -- Pipelining implicite dans le script
    for i, key in ipairs(keys) do
        redis.call('INCR', key)
    end
    -- Ou encore mieux si applicable :
    -- redis.call('MSET', ...) pour les SET multiples
end
```

## Debugging et monitoring

### Activer les logs de scripts Lua

```bash
# Voir les scripts lents
redis-cli SLOWLOG GET 10

# Monitorer l'exécution en temps réel
redis-cli MONITOR

# Obtenir des statistiques sur les fonctions
redis-cli FUNCTION STATS
```

### Instrumenter les fonctions

```lua
#!lua name=monitored_operations

local function instrumented_operation(keys, args)
    local start_time = redis.call('TIME')

    -- Opération principale
    local result = redis.call('GET', keys[1])

    local end_time = redis.call('TIME')
    local duration = (end_time[1] - start_time[1]) * 1000000 + (end_time[2] - start_time[2])

    -- Logging des métriques
    redis.call('HINCRBY', 'metrics:function_calls', 'instrumented_operation', 1)
    redis.call('HINCRBY', 'metrics:function_duration', 'instrumented_operation', duration)

    return result
end

redis.register_function('instrumented_operation', instrumented_operation)
```

## Quand utiliser chaque approche ?

### Commandes atomiques natives
- ✅ Incréments simples, compteurs
- ✅ Opérations sur une seule clé
- ✅ Performance maximale requise

### MULTI/EXEC (Transactions)
- ✅ Séquence d'opérations sans logique conditionnelle
- ✅ Besoin d'isolation temporaire
- ✅ Pas de calculs complexes entre les commandes

### WATCH + MULTI/EXEC
- ✅ Opérations conditionnelles basées sur l'état actuel
- ✅ Race conditions peu fréquentes
- ✅ Retry acceptable en cas de conflit

### Scripts Lua (EVAL/EVALSHA)
- ✅ Logique métier complexe
- ✅ Calculs conditionnels multiples
- ✅ Scripts ponctuels ou prototypes

### Redis Functions
- ✅ **Recommandé pour tout nouveau développement (Redis 7+)**
- ✅ Logique métier réutilisable et persistante
- ✅ Organisation en bibliothèques
- ✅ Environnements de production avec Redis 7+
- ✅ Clusters Redis

## Conclusion

L'atomicité dans Redis est un sujet fondamental qui impacte directement la **fiabilité** et la **cohérence** de vos applications. Les mécanismes que nous venons d'explorer vous donnent les outils nécessaires pour :

1. **Éliminer les race conditions** dans vos opérations critiques
2. **Implémenter une logique métier complexe** de manière atomique
3. **Optimiser les performances** en réduisant les aller-retours réseau
4. **Garantir la cohérence des données** même sous forte concurrence

**Règle d'or** : Privilégiez toujours la solution la plus simple qui répond à votre besoin. Une commande atomique native sera toujours plus performante qu'un script Lua, mais un script Lua bien écrit sera toujours préférable à une logique applicative fragile.

Dans les sections suivantes, nous explorerons en détail chacune de ces approches avec des cas d'usage concrets et des patterns de production éprouvés.

---

**📚 Points clés à retenir** :
- Redis est single-thread : chaque commande est atomique, mais pas les séquences
- Les scripts Lua/Functions s'exécutent atomiquement et bloquent les autres opérations
- Redis Functions (v7+) est désormais la méthode recommandée pour la programmabilité
- L'atomicité a un coût : les scripts trop longs peuvent bloquer Redis
- Toujours valider et tester vos scripts avant production

**🔜 Prochaine section** : [7.1 Transactions : MULTI/EXEC et pipeline transactionnel](./01-transactions-multi-exec.md)

⏭️ [Transactions : MULTI/EXEC et pipeline transactionnel](/07-atomicite-programmabilite/01-transactions-multi-exec.md)
