🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.5 Triggers and Functions (Redis Stack)

## Introduction : Les Triggers dans Redis Stack

**Redis Stack** introduit une fonctionnalité puissante qui étend Redis Functions : les **Triggers**. Un trigger est une fonction qui s'exécute **automatiquement** en réponse à des événements dans Redis, comme la modification d'une clé.

### Le paradigme événementiel dans Redis

```
Sans Triggers (Approche Pull):
┌─────────┐         ┌─────────┐
│  App 1  │────────►│         │
└─────────┘  SET    │  Redis  │
                    │         │
┌─────────┐         │         │
│  App 2  │────────►│         │
└─────────┘  GET    └─────────┘
                         ▲
                         │
                    Polling requis
                    pour détecter
                    les changements

Avec Triggers (Approche Push):
┌─────────┐         ┌─────────┐          ┌─────────────┐
│  App 1  │────────►│         │─trigger─►│   Function  │
└─────────┘  SET    │  Redis  │ auto     │  (callback) │
                    │ Stack   │          └─────────────┘
                    │         │               │
                    │         │          Réaction
                    └─────────┘          automatique
```

### Différence entre Redis Functions et Triggers

| Aspect | Redis Functions | Triggers (Redis Stack) |
|--------|-----------------|------------------------|
| **Exécution** | Manuelle via FCALL | Automatique (événement) |
| **Invocation** | Application cliente | Redis lui-même |
| **Cas d'usage** | Logique métier atomique | Réactions événementielles |
| **Latence** | Synchrone (bloquant) | Asynchrone (non-bloquant) |
| **Disponibilité** | Redis 7+ | Redis Stack uniquement |
| **Performance** | Haute (atomique) | Moyenne (overhead événement) |

### Cas d'usage des Triggers

✅ **Utilisations idéales** :
- Audit automatique des modifications
- Synchronisation inter-services
- Notifications en temps réel
- Mise à jour de caches dérivés
- Calculs de métriques agrégées
- Validation de données
- Workflow automatisés

❌ **À éviter** :
- Opérations critiques nécessitant confirmation
- Logique métier complexe bloquante
- Modifications circulaires (trigger → modification → trigger...)

## Installation et configuration

### Prérequis

```bash
# Vérifier que Redis Stack est installé
redis-cli MODULE LIST

# Doit inclure:
# 1) "name": "redisgears_2"  # Module pour Triggers and Functions
```

### Installation de Redis Stack

```bash
# Docker (recommandé pour développement)
docker run -d -p 6379:6379 redis/redis-stack-server:latest

# Ou via package manager (Ubuntu/Debian)
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get install redis-stack-server
```

## Anatomie d'un Trigger

### Structure de base

```lua
#!lua name=mylib

-- Fonction trigger (sera appelée automatiquement)
local function on_key_change(client, data)
    --[[
    Paramètres reçus automatiquement:

    client: Informations sur le client qui a fait la modification
        - client.id: ID du client
        - client.addr: Adresse IP
        - client.cmd: Commande exécutée (SET, DEL, etc.)

    data: Données de l'événement
        - data.key: Clé modifiée
        - data.event: Type d'événement ('set', 'del', 'expired', etc.)
    ]]

    redis.log(redis.LOG_NOTICE,
        string.format('Key changed: %s by command: %s',
            data.key, client.cmd))

    -- Logique de réaction
    redis.call('INCR', 'stats:modifications')
end

-- Enregistrer la fonction
redis.register_function('on_key_change', on_key_change)

-- Enregistrer le trigger sur un pattern de clés
redis.register_keyspace_trigger(
    'my_trigger',           -- Nom du trigger
    'user:*',               -- Pattern de clés
    'on_key_change',        -- Fonction callback
    {
        'prefix'            -- Mode: 'prefix' ou 'regex'
    }
)
```

### Types d'événements

```lua
-- Événements disponibles pour les triggers

local function universal_trigger(client, data)
    local event = data.event

    if event == 'set' then
        -- Clé créée ou modifiée via SET
        redis.log(redis.LOG_NOTICE, 'SET: ' .. data.key)

    elseif event == 'del' then
        -- Clé supprimée
        redis.log(redis.LOG_NOTICE, 'DEL: ' .. data.key)

    elseif event == 'expired' then
        -- Clé expirée (TTL atteint)
        redis.log(redis.LOG_NOTICE, 'EXPIRED: ' .. data.key)

    elseif event == 'evicted' then
        -- Clé évictée (politique d'éviction)
        redis.log(redis.LOG_NOTICE, 'EVICTED: ' .. data.key)

    elseif event == 'hset' then
        -- Champ de hash modifié
        redis.log(redis.LOG_NOTICE, 'HSET: ' .. data.key)

    elseif event == 'lpush' or event == 'rpush' then
        -- Élément ajouté à une liste
        redis.log(redis.LOG_NOTICE, 'LIST PUSH: ' .. data.key)

    elseif event == 'zadd' then
        -- Élément ajouté à un sorted set
        redis.log(redis.LOG_NOTICE, 'ZADD: ' .. data.key)
    end

    return 'OK'
end
```

## Exemples progressifs

### Exemple 1 : Audit Trail automatique

```lua
#!lua name=audit

-- === Configuration ===
local AUDIT_LOG_KEY = 'audit:log'
local MAX_AUDIT_ENTRIES = 10000

-- === Helpers ===
local function get_timestamp()
    local time = redis.call('TIME')
    return tonumber(time[1]) + (tonumber(time[2]) / 1000000)
end

local function serialize_audit_entry(client, data)
    return string.format(
        '{"timestamp":%.6f,"key":"%s","event":"%s","cmd":"%s","client":"%s"}',
        get_timestamp(),
        data.key,
        data.event,
        client.cmd or 'unknown',
        client.addr or 'unknown'
    )
end

-- === Fonction Trigger ===
local function audit_trail(client, data)
    --[[
    Enregistre automatiquement toutes les modifications
    dans un log d'audit
    ]]

    -- Créer l'entrée d'audit
    local entry = serialize_audit_entry(client, data)

    -- Ajouter au log (liste triée par timestamp)
    local timestamp = get_timestamp()
    redis.call('ZADD', AUDIT_LOG_KEY, timestamp, entry)

    -- Limiter la taille du log
    local count = redis.call('ZCARD', AUDIT_LOG_KEY)
    if count > MAX_AUDIT_ENTRIES then
        local to_remove = count - MAX_AUDIT_ENTRIES
        redis.call('ZREMRANGEBYRANK', AUDIT_LOG_KEY, 0, to_remove - 1)
    end

    -- Stats
    redis.call('HINCRBY', 'audit:stats', 'total_events', 1)
    redis.call('HINCRBY', 'audit:stats', 'event:' .. data.event, 1)

    redis.log(redis.LOG_NOTICE,
        string.format('[AUDIT] %s on %s', data.event, data.key))
end

-- === Fonctions utilitaires ===
local function get_audit_log(keys, args)
    --[[
    Récupère les entrées d'audit
    ARGS[1]: start_time (timestamp, optionnel)
    ARGS[2]: end_time (timestamp, optionnel)
    ARGS[3]: limit (nombre d'entrées, défaut 100)
    ]]

    local start_time = tonumber(args[1]) or '-inf'
    local end_time = tonumber(args[2]) or '+inf'
    local limit = tonumber(args[3]) or 100

    local entries = redis.call(
        'ZREVRANGEBYSCORE',
        AUDIT_LOG_KEY,
        end_time,
        start_time,
        'LIMIT', 0, limit
    )

    return {
        count = #entries,
        entries = entries
    }
end

local function get_audit_stats(keys, args)
    return redis.call('HGETALL', 'audit:stats')
end

local function clear_audit_log(keys, args)
    redis.call('DEL', AUDIT_LOG_KEY)
    redis.call('DEL', 'audit:stats')
    return 'OK'
end

-- === Enregistrement ===
redis.register_function('audit_trail', audit_trail)
redis.register_function('get_audit_log', get_audit_log)
redis.register_function('get_audit_stats', get_audit_stats)
redis.register_function('clear_audit_log', clear_audit_log)

-- Enregistrer les triggers sur différents patterns
redis.register_keyspace_trigger(
    'audit_users',
    'user:*',
    'audit_trail',
    {'prefix'}
)

redis.register_keyspace_trigger(
    'audit_orders',
    'order:*',
    'audit_trail',
    {'prefix'}
)

redis.register_keyspace_trigger(
    'audit_products',
    'product:*',
    'audit_trail',
    {'prefix'}
)
```

**Utilisation en Python** :

```python
import redis
import json
import time
from datetime import datetime, timedelta

class AuditSystem:
    """
    Système d'audit automatique avec Redis Stack Triggers
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self._ensure_loaded()

    def _ensure_loaded(self):
        """Charge la bibliothèque si nécessaire"""
        library_code = """
        #!lua name=audit
        -- [Code complet ci-dessus]
        """

        try:
            self.redis.function_load(library_code, replace=True)
        except Exception as e:
            print(f"Warning: {e}")

    def get_recent_events(self, limit=100):
        """Récupère les événements récents"""
        result = self.redis.fcall_ro(
            'get_audit_log',
            0,
            0,  # start_time (all)
            0,  # end_time (all)
            limit
        )

        if isinstance(result, list) and len(result) >= 2:
            entries = result[1] if isinstance(result[1], list) else []
            return [json.loads(entry) for entry in entries]

        return []

    def get_events_in_range(self, start_time, end_time, limit=100):
        """Récupère les événements dans une plage de temps"""
        result = self.redis.fcall_ro(
            'get_audit_log',
            0,
            int(start_time.timestamp()),
            int(end_time.timestamp()),
            limit
        )

        entries = result[1] if isinstance(result, list) and len(result) >= 2 else []
        return [json.loads(entry) for entry in entries]

    def get_statistics(self):
        """Récupère les statistiques d'audit"""
        stats_raw = self.redis.fcall_ro('get_audit_stats', 0)

        if isinstance(stats_raw, list):
            stats = {}
            for i in range(0, len(stats_raw), 2):
                stats[stats_raw[i]] = int(stats_raw[i + 1])
            return stats

        return {}

    def clear(self):
        """Efface le log d'audit"""
        return self.redis.fcall('clear_audit_log', 0)

    def print_recent_events(self, limit=10):
        """Affiche les événements récents"""
        events = self.get_recent_events(limit)

        print(f"\n{'='*80}")
        print(f"Recent Audit Events (last {len(events)})")
        print(f"{'='*80}\n")

        for event in events:
            timestamp = datetime.fromtimestamp(event['timestamp'])
            print(f"[{timestamp.strftime('%Y-%m-%d %H:%M:%S')}] "
                  f"{event['event'].upper():10s} | "
                  f"{event['key']:30s} | "
                  f"{event['cmd']:10s} | "
                  f"{event['client']}")

        print(f"\n{'='*80}\n")

# Test du système d'audit
r = redis.Redis(decode_responses=True)
audit = AuditSystem(r)

# Effacer l'ancien log
audit.clear()

# Effectuer des opérations (les triggers se déclenchent automatiquement)
r.set('user:1001', 'John Doe')
r.hset('user:1001:profile', 'email', 'john@example.com')
r.set('order:5001', 'pending')
r.set('product:laptop', 'MacBook Pro')
r.delete('user:1001')

# Attendre un peu pour que les triggers se déclenchent
time.sleep(0.5)

# Afficher les événements
audit.print_recent_events()

# Statistiques
stats = audit.get_statistics()
print("Statistics:")
for key, value in stats.items():
    print(f"  {key}: {value}")

# Output exemple:
# ================================================================================
# Recent Audit Events (last 5)
# ================================================================================
#
# [2024-12-10 15:30:45] DEL        | user:1001                      | DEL        | 127.0.0.1:52134
# [2024-12-10 15:30:45] SET        | product:laptop                 | SET        | 127.0.0.1:52134
# [2024-12-10 15:30:45] SET        | order:5001                     | SET        | 127.0.0.1:52134
# [2024-12-10 15:30:45] HSET       | user:1001:profile              | HSET       | 127.0.0.1:52134
# [2024-12-10 15:30:45] SET        | user:1001                      | SET        | 127.0.0.1:52134
#
# ================================================================================
#
# Statistics:
#   total_events: 5
#   event:set: 3
#   event:hset: 1
#   event:del: 1
```

### Exemple 2 : Synchronisation de cache automatique

```lua
#!lua name=cache_sync

--[[
    Synchronise automatiquement un cache dérivé
    Quand une donnée source change, met à jour le cache
]]

-- === Configuration ===
local CACHE_PREFIX = 'cache:'
local SOURCE_PREFIX = 'source:'

-- === Helpers ===
local function derive_cache_key(source_key)
    -- Transformer 'source:user:123' en 'cache:user:123'
    return CACHE_PREFIX .. string.sub(source_key, #SOURCE_PREFIX + 1)
end

local function compute_derived_value(source_key)
    -- Lire la valeur source
    local source_value = redis.call('GET', source_key)

    if not source_value then
        return nil
    end

    -- Transformation (exemple : uppercase)
    local derived = string.upper(source_value)

    -- Enrichissement avec métadonnées
    local metadata = string.format(
        '{"value":"%s","computed_at":%d}',
        derived,
        redis.call('TIME')[1]
    )

    return metadata
end

-- === Fonction Trigger ===
local function sync_cache_on_change(client, data)
    --[[
    Synchronise le cache quand la source change
    ]]

    local source_key = data.key
    local event = data.event

    -- Ne traiter que les clés source
    if not string.find(source_key, '^' .. SOURCE_PREFIX) then
        return
    end

    local cache_key = derive_cache_key(source_key)

    if event == 'set' then
        -- Mettre à jour le cache
        local derived = compute_derived_value(source_key)
        if derived then
            redis.call('SET', cache_key, derived)
            redis.call('EXPIRE', cache_key, 3600)  -- 1h TTL

            redis.log(redis.LOG_NOTICE,
                string.format('[CACHE_SYNC] Updated: %s', cache_key))
        end

    elseif event == 'del' or event == 'expired' then
        -- Supprimer du cache
        redis.call('DEL', cache_key)

        redis.log(redis.LOG_NOTICE,
            string.format('[CACHE_SYNC] Deleted: %s', cache_key))
    end

    -- Stats
    redis.call('HINCRBY', 'cache_sync:stats', 'syncs', 1)
end

local function force_cache_rebuild(keys, args)
    --[[
    Reconstruit tout le cache depuis les sources
    ]]

    local rebuilt = 0
    local cursor = "0"

    repeat
        local result = redis.call('SCAN', cursor, 'MATCH', SOURCE_PREFIX .. '*', 'COUNT', 100)
        cursor = result[1]
        local source_keys = result[2]

        for _, source_key in ipairs(source_keys) do
            local cache_key = derive_cache_key(source_key)
            local derived = compute_derived_value(source_key)

            if derived then
                redis.call('SET', cache_key, derived)
                redis.call('EXPIRE', cache_key, 3600)
                rebuilt = rebuilt + 1
            end
        end

    until cursor == "0"

    return {rebuilt = rebuilt}
end

local function get_cache_stats(keys, args)
    local stats = redis.call('HGETALL', 'cache_sync:stats')

    -- Compter les entrées de cache
    local cache_count = 0
    local cursor = "0"
    repeat
        local result = redis.call('SCAN', cursor, 'MATCH', CACHE_PREFIX .. '*', 'COUNT', 100)
        cursor = result[1]
        cache_count = cache_count + #result[2]
    until cursor == "0"

    return {
        stats = stats,
        cache_entries = cache_count
    }
end

-- === Enregistrement ===
redis.register_function('sync_cache_on_change', sync_cache_on_change)
redis.register_function('force_cache_rebuild', force_cache_rebuild)
redis.register_function('get_cache_stats', get_cache_stats)

-- Trigger sur les clés source
redis.register_keyspace_trigger(
    'cache_sync_trigger',
    SOURCE_PREFIX .. '*',
    'sync_cache_on_change',
    {'prefix'}
)
```

**Utilisation** :

```python
class CacheSyncSystem:
    """
    Système de synchronisation de cache automatique
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self._ensure_loaded()

    def _ensure_loaded(self):
        # [Code de chargement similaire]
        pass

    def set_source(self, key, value):
        """Définit une valeur source (trigger auto)"""
        full_key = f'source:{key}'
        self.redis.set(full_key, value)
        return full_key

    def get_cached(self, key):
        """Récupère la valeur cachée dérivée"""
        cache_key = f'cache:{key}'
        cached = self.redis.get(cache_key)

        if cached:
            return json.loads(cached)
        return None

    def rebuild_all(self):
        """Force la reconstruction du cache"""
        result = self.redis.fcall('force_cache_rebuild', 0)
        return result

    def get_stats(self):
        """Statistiques du cache"""
        result = self.redis.fcall_ro('get_cache_stats', 0)
        return result

# Test
cache_sync = CacheSyncSystem(r)

# Définir des sources (triggers se déclenchent automatiquement)
cache_sync.set_source('user:1001', 'alice')
cache_sync.set_source('user:1002', 'bob')
cache_sync.set_source('user:1003', 'charlie')

# Attendre la synchronisation
time.sleep(0.3)

# Vérifier le cache
for user_id in ['user:1001', 'user:1002', 'user:1003']:
    cached = cache_sync.get_cached(user_id)
    print(f"{user_id}: {cached}")

# Output:
# user:1001: {'value': 'ALICE', 'computed_at': 1702226445}
# user:1002: {'value': 'BOB', 'computed_at': 1702226445}
# user:1003: {'value': 'CHARLIE', 'computed_at': 1702226445}
```

### Exemple 3 : Notifications en temps réel

```lua
#!lua name=notifications

--[[
    Système de notifications push en temps réel
    Les triggers publient automatiquement sur des channels Pub/Sub
]]

-- === Configuration ===
local NOTIFICATION_CHANNEL_PREFIX = 'notifications:'

-- === Helpers ===
local function build_notification(client, data, extra_info)
    local timestamp = redis.call('TIME')[1]

    return string.format(
        '{"timestamp":%s,"event":"%s","key":"%s","cmd":"%s","client":"%s","extra":%s}',
        timestamp,
        data.event,
        data.key,
        client.cmd or 'unknown',
        client.addr or 'unknown',
        extra_info or '{}'
    )
end

local function extract_entity_id(key, prefix)
    -- Extrait l'ID depuis 'prefix:entity:123' -> '123'
    local pattern = '^' .. prefix .. ':([^:]+):([^:]+)'
    local entity_type, entity_id = string.match(key, pattern)
    return entity_type, entity_id
end

-- === Fonction Trigger ===
local function notify_on_change(client, data)
    --[[
    Publie une notification quand une clé change
    ]]

    local key = data.key
    local event = data.event

    -- Déterminer le channel de notification
    local entity_type, entity_id = extract_entity_id(key, 'data')

    if not entity_type or not entity_id then
        return  -- Ignorer les clés non conformes
    end

    -- Enrichir avec des données additionnelles si SET
    local extra_info = '{}'
    if event == 'set' then
        local value = redis.call('GET', key)
        if value and #value < 200 then  -- Limiter la taille
            extra_info = string.format('{"value":"%s"}', value)
        end
    end

    -- Construire la notification
    local notification = build_notification(client, data, extra_info)

    -- Publier sur le channel spécifique à l'entité
    local channel = NOTIFICATION_CHANNEL_PREFIX .. entity_type .. ':' .. entity_id
    redis.call('PUBLISH', channel, notification)

    -- Publier aussi sur le channel global du type
    local global_channel = NOTIFICATION_CHANNEL_PREFIX .. entity_type .. ':*'
    redis.call('PUBLISH', global_channel, notification)

    -- Stats
    redis.call('HINCRBY', 'notifications:stats', 'total', 1)
    redis.call('HINCRBY', 'notifications:stats', entity_type, 1)

    redis.log(redis.LOG_NOTICE,
        string.format('[NOTIFY] Published to %s', channel))
end

local function get_notification_stats(keys, args)
    return redis.call('HGETALL', 'notifications:stats')
end

-- === Enregistrement ===
redis.register_function('notify_on_change', notify_on_change)
redis.register_function('get_notification_stats', get_notification_stats)

-- Triggers sur différents patterns
redis.register_keyspace_trigger(
    'notify_data',
    'data:*',
    'notify_on_change',
    {'prefix'}
)
```

**Client Python avec Pub/Sub** :

```python
import threading
import json

class NotificationSystem:
    """
    Système de notifications en temps réel avec Redis Stack
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.pubsub = self.redis.pubsub()
        self.listeners = {}
        self._ensure_loaded()

    def _ensure_loaded(self):
        # [Code de chargement]
        pass

    def subscribe_to_entity(self, entity_type, entity_id, callback):
        """
        S'abonne aux notifications d'une entité spécifique
        """
        channel = f'notifications:{entity_type}:{entity_id}'

        if channel not in self.listeners:
            self.listeners[channel] = []
            self.pubsub.subscribe(channel)

        self.listeners[channel].append(callback)

        # Démarrer le thread d'écoute si pas encore fait
        if not hasattr(self, '_listener_thread'):
            self._start_listener()

    def subscribe_to_entity_type(self, entity_type, callback):
        """
        S'abonne à toutes les notifications d'un type d'entité
        """
        channel = f'notifications:{entity_type}:*'

        if channel not in self.listeners:
            self.listeners[channel] = []
            self.pubsub.psubscribe(channel)  # Pattern subscribe

        self.listeners[channel].append(callback)

        if not hasattr(self, '_listener_thread'):
            self._start_listener()

    def _start_listener(self):
        """Démarre le thread d'écoute des notifications"""
        def listen():
            for message in self.pubsub.listen():
                if message['type'] in ['message', 'pmessage']:
                    channel = message['channel']
                    if isinstance(channel, bytes):
                        channel = channel.decode('utf-8')

                    data = message['data']
                    if isinstance(data, bytes):
                        data = data.decode('utf-8')

                    try:
                        notification = json.loads(data)

                        # Appeler les callbacks
                        for listener_channel, callbacks in self.listeners.items():
                            if self._match_channel(channel, listener_channel):
                                for callback in callbacks:
                                    try:
                                        callback(notification)
                                    except Exception as e:
                                        print(f"Callback error: {e}")
                    except json.JSONDecodeError:
                        pass

        self._listener_thread = threading.Thread(target=listen, daemon=True)
        self._listener_thread.start()

    def _match_channel(self, actual, pattern):
        """Vérifie si un channel correspond au pattern"""
        if pattern == actual:
            return True
        if pattern.endswith(':*'):
            prefix = pattern[:-2]
            return actual.startswith(prefix)
        return False

    def get_stats(self):
        """Récupère les statistiques des notifications"""
        stats_raw = self.redis.fcall_ro('get_notification_stats', 0)

        if isinstance(stats_raw, list):
            stats = {}
            for i in range(0, len(stats_raw), 2):
                stats[stats_raw[i]] = int(stats_raw[i + 1])
            return stats

        return {}

# Utilisation
notif_system = NotificationSystem(r)

# Callbacks
def on_user_change(notification):
    print(f"[USER CHANGED] {notification['key']} - Event: {notification['event']}")

def on_order_change(notification):
    print(f"[ORDER CHANGED] {notification['key']} - Event: {notification['event']}")

def on_any_user_change(notification):
    print(f"[ANY USER] {notification['key']}")

# S'abonner
notif_system.subscribe_to_entity('user', '1001', on_user_change)
notif_system.subscribe_to_entity('order', '5001', on_order_change)
notif_system.subscribe_to_entity_type('user', on_any_user_change)

# Attendre que le listener démarre
time.sleep(0.5)

# Effectuer des modifications (triggers auto)
r.set('data:user:1001', 'Alice Updated')
r.set('data:user:1002', 'Bob Created')
r.set('data:order:5001', 'Order Confirmed')
r.delete('data:user:1001')

# Attendre les notifications
time.sleep(1)

# Statistiques
stats = notif_system.get_stats()
print(f"\nNotification stats: {stats}")

# Output:
# [ANY USER] data:user:1001
# [USER CHANGED] data:user:1001 - Event: set
# [ANY USER] data:user:1002
# [ORDER CHANGED] data:order:5001 - Event: set
# [ANY USER] data:user:1001
# [USER CHANGED] data:user:1001 - Event: del
#
# Notification stats: {'total': 4, 'user': 3, 'order': 1}
```

## Patterns avancés

### Pattern 1 : Validation de données avec rollback

```lua
#!lua name=validation

--[[
    Valide les données avant acceptation
    Annule (DEL) si invalide
]]

local function validate_email(email)
    return string.match(email, '^[%w.%%+-]+@[%w.-]+%.%w+$') ~= nil
end

local function validate_age(age)
    local num = tonumber(age)
    return num and num >= 18 and num <= 120
end

local function validate_user_data(client, data)
    --[[
    Valide automatiquement les données utilisateur
    Supprime la clé si invalide
    ]]

    if data.event ~= 'set' and data.event ~= 'hset' then
        return
    end

    local key = data.key

    -- Valider seulement les clés user
    if not string.find(key, '^user:%d+:profile$') then
        return
    end

    -- Récupérer les données
    local profile = redis.call('HGETALL', key)
    local profile_data = {}
    for i = 1, #profile, 2 do
        profile_data[profile[i]] = profile[i + 1]
    end

    -- Validations
    local errors = {}

    if profile_data.email and not validate_email(profile_data.email) then
        table.insert(errors, 'Invalid email format')
    end

    if profile_data.age and not validate_age(profile_data.age) then
        table.insert(errors, 'Age must be between 18 and 120')
    end

    -- Si invalide, supprimer et logger
    if #errors > 0 then
        redis.call('DEL', key)

        local error_msg = table.concat(errors, ', ')
        redis.log(redis.LOG_WARNING,
            string.format('[VALIDATION] Rejected %s: %s', key, error_msg))

        -- Enregistrer dans une liste d'erreurs
        redis.call('LPUSH', 'validation:errors',
            string.format('%s|%s|%s',
                redis.call('TIME')[1], key, error_msg))
        redis.call('LTRIM', 'validation:errors', 0, 999)

        -- Stats
        redis.call('HINCRBY', 'validation:stats', 'rejected', 1)
    else
        redis.call('HINCRBY', 'validation:stats', 'accepted', 1)
    end
end

redis.register_function('validate_user_data', validate_user_data)

redis.register_keyspace_trigger(
    'validation_trigger',
    'user:*:profile',
    'validate_user_data',
    {'prefix'}
)
```

### Pattern 2 : Agrégation automatique de métriques

```lua
#!lua name=metrics

--[[
    Agrège automatiquement des métriques
    depuis les événements individuels
]]

local function aggregate_metrics(client, data)
    --[[
    Agrège les métriques en temps réel
    ]]

    local key = data.key
    local event = data.event

    -- Pattern: metric:service:endpoint:timestamp
    local service, endpoint, timestamp = string.match(
        key,
        '^metric:([^:]+):([^:]+):(%d+)$'
    )

    if not service or not endpoint then
        return
    end

    local value = tonumber(redis.call('GET', key)) or 0

    -- Agrégations par minute
    local minute = math.floor(tonumber(timestamp) / 60) * 60
    local minute_key = string.format('agg:minute:%s:%s:%d', service, endpoint, minute)
    redis.call('INCRBY', minute_key, value)
    redis.call('EXPIRE', minute_key, 3600)  -- 1h

    -- Agrégations par heure
    local hour = math.floor(tonumber(timestamp) / 3600) * 3600
    local hour_key = string.format('agg:hour:%s:%s:%d', service, endpoint, hour)
    redis.call('INCRBY', hour_key, value)
    redis.call('EXPIRE', hour_key, 86400)  -- 24h

    -- Agrégations par jour
    local day = math.floor(tonumber(timestamp) / 86400) * 86400
    local day_key = string.format('agg:day:%s:%s:%d', service, endpoint, day)
    redis.call('INCRBY', day_key, value)
    redis.call('EXPIRE', day_key, 2592000)  -- 30 jours

    -- Stats globales
    redis.call('HINCRBY', string.format('stats:%s:%s', service, endpoint), 'total', value)
    redis.call('HINCRBY', string.format('stats:%s:%s', service, endpoint), 'count', 1)

    redis.log(redis.LOG_DEBUG,
        string.format('[METRICS] Aggregated %s: %d', key, value))
end

redis.register_function('aggregate_metrics', aggregate_metrics)

redis.register_keyspace_trigger(
    'metrics_aggregation',
    'metric:*',
    'aggregate_metrics',
    {'prefix'}
)
```

## Gestion et monitoring des Triggers

### Lister les triggers actifs

```python
def list_triggers(redis_client):
    """
    Liste tous les triggers enregistrés
    """
    # Les triggers sont enregistrés dans les fonctions
    libs = redis_client.function_list()

    triggers = []
    for lib in libs:
        lib_name = lib['library_name']

        # Chercher les enregistrements de triggers
        # (Ceci est une approximation, l'API exacte peut varier)
        for func in lib['functions']:
            if 'trigger' in func.get('description', '').lower():
                triggers.append({
                    'library': lib_name,
                    'function': func['name'],
                    'description': func.get('description', '')
                })

    return triggers

# Utilisation
triggers = list_triggers(r)
for trigger in triggers:
    print(f"Library: {trigger['library']}")
    print(f"  Function: {trigger['function']}")
    print(f"  Description: {trigger['description']}\n")
```

### Désactiver temporairement un trigger

```lua
-- Pattern pour désactiver/activer des triggers

local TRIGGER_ENABLED = {}

local function is_trigger_enabled(trigger_name)
    if TRIGGER_ENABLED[trigger_name] == nil then
        -- Vérifier dans Redis
        local enabled = redis.call('GET', 'trigger:enabled:' .. trigger_name)
        TRIGGER_ENABLED[trigger_name] = (enabled ~= '0')
    end
    return TRIGGER_ENABLED[trigger_name]
end

local function conditional_trigger(client, data)
    if not is_trigger_enabled('my_trigger') then
        return  -- Trigger désactivé
    end

    -- Logique normale du trigger
    -- ...
end

local function enable_trigger(keys, args)
    local trigger_name = args[1]
    redis.call('SET', 'trigger:enabled:' .. trigger_name, '1')
    TRIGGER_ENABLED[trigger_name] = true
    return 'Enabled'
end

local function disable_trigger(keys, args)
    local trigger_name = args[1]
    redis.call('SET', 'trigger:enabled:' .. trigger_name, '0')
    TRIGGER_ENABLED[trigger_name] = false
    return 'Disabled'
end
```

### Monitoring des performances

```python
class TriggerMonitor:
    """
    Monitoring des performances des triggers
    """

    def __init__(self, redis_client):
        self.redis = redis_client

    def get_trigger_metrics(self):
        """
        Récupère les métriques de tous les triggers
        """
        metrics = {}

        # Chercher les stats de chaque trigger
        cursor = 0
        while True:
            cursor, keys = self.redis.scan(
                cursor,
                match='*:stats',
                count=100
            )

            for key in keys:
                stats = self.redis.hgetall(key)
                if stats:
                    metrics[key] = {
                        k.decode() if isinstance(k, bytes) else k:
                        int(v.decode() if isinstance(v, bytes) else v)
                        for k, v in stats.items()
                    }

            if cursor == 0:
                break

        return metrics

    def analyze_trigger_performance(self):
        """
        Analyse la performance des triggers
        """
        metrics = self.get_trigger_metrics()

        analysis = {
            'total_triggers': len(metrics),
            'total_events': sum(m.get('total', 0) for m in metrics.values()),
            'by_trigger': {}
        }

        for trigger_key, stats in metrics.items():
            trigger_name = trigger_key.replace(':stats', '')
            total = stats.get('total', 0)

            analysis['by_trigger'][trigger_name] = {
                'total_executions': total,
                'percentage': (total / analysis['total_events'] * 100)
                              if analysis['total_events'] > 0 else 0,
                'stats': stats
            }

        return analysis

# Utilisation
monitor = TriggerMonitor(r)
analysis = monitor.analyze_trigger_performance()

print(f"Total triggers: {analysis['total_triggers']}")
print(f"Total events: {analysis['total_events']}\n")

for trigger_name, data in analysis['by_trigger'].items():
    print(f"{trigger_name}:")
    print(f"  Executions: {data['total_executions']}")
    print(f"  Percentage: {data['percentage']:.2f}%")
    print(f"  Details: {data['stats']}\n")
```

## Limitations et considérations

### 1. Performance et overhead

```
Sans trigger:
SET key value ────► [Redis] ────► OK
                     ~0.1ms

Avec trigger:
SET key value ────► [Redis] ────► [Trigger] ────► [Fonction Lua] ────► OK
                     ~0.1ms        ~0.05ms         ~0.5-5ms

Overhead total: ~0.5-5ms par opération
```

**Impact** :
- Latence additionnelle de 0.5-5ms par événement
- Consommation CPU accrue
- Peut saturer sous très forte charge (>100k ops/s)

**Mitigation** :
- Trigger seulement ce qui est nécessaire
- Logique trigger la plus légère possible
- Batching si possible
- Monitoring actif

### 2. Risque de boucles infinies

```lua
-- ❌ DANGER : Boucle infinie
local function dangerous_trigger(client, data)
    -- Ce trigger modifie une clé...
    redis.call('SET', 'counter', '1')
    -- ...qui déclenche le même trigger à nouveau !
    -- → BOUCLE INFINIE
end
```

**Protection** :

```lua
-- ✅ Protection contre les boucles
local RECURSION_LIMIT = 3
local recursion_depth = 0

local function safe_trigger(client, data)
    recursion_depth = recursion_depth + 1

    if recursion_depth > RECURSION_LIMIT then
        redis.log(redis.LOG_WARNING, 'Recursion limit exceeded')
        recursion_depth = 0
        return
    end

    -- Logique du trigger
    redis.call('SET', 'other_key', 'value')

    recursion_depth = recursion_depth - 1
end
```

### 3. Ordre d'exécution non garanti

```lua
-- Si deux triggers s'appliquent à la même clé,
-- l'ordre d'exécution n'est PAS garanti

-- Trigger A
redis.register_keyspace_trigger('trigger_a', 'user:*', 'callback_a', {'prefix'})

-- Trigger B
redis.register_keyspace_trigger('trigger_b', 'user:*', 'callback_b', {'prefix'})

-- Lors d'un SET user:123, A et B sont appelés,
-- mais l'ordre peut varier : A→B ou B→A
```

**Solution** : Un seul trigger qui orchestre les actions.

### 4. Pas de rollback automatique

```lua
-- Si un trigger échoue, les modifications précédentes
-- ne sont PAS annulées automatiquement

local function risky_trigger(client, data)
    redis.call('INCR', 'counter1')  -- ✅ Réussi
    redis.call('INCR', 'string_key')  -- ❌ Erreur (pas un nombre)
    redis.call('INCR', 'counter2')  -- ❌ Pas exécuté
    -- counter1 reste incrémenté !
end
```

## Bonnes pratiques

### Checklist de développement

```
☐ Les triggers sont idempotents (réexécution = même résultat)
☐ La logique est minimale (< 5ms d'exécution)
☐ Protection contre les boucles infinies
☐ Gestion d'erreur robuste (pcall)
☐ Logging approprié pour le debugging
☐ Tests sous charge
☐ Monitoring des performances
☐ Documentation des side-effects
☐ Plan de désactivation d'urgence
☐ Éviter les opérations bloquantes (SCAN avec limite)
```

### Template de trigger production

```lua
#!lua name=production_trigger_template

-- === Configuration ===
local VERSION = "1.0.0"
local TRIGGER_NAME = "my_trigger"
local MAX_EXECUTION_TIME_MS = 5

-- === Protection ===
local execution_count = 0
local MAX_RECURSION = 3

local function is_enabled()
    local enabled = redis.call('GET', 'trigger:enabled:' .. TRIGGER_NAME)
    return enabled ~= '0'
end

-- === Fonction Trigger ===
local function my_trigger_function(client, data)
    -- 0. Vérifier si activé
    if not is_enabled() then
        return
    end

    -- 1. Protection récursion
    execution_count = execution_count + 1
    if execution_count > MAX_RECURSION then
        redis.log(redis.LOG_WARNING,
            string.format('[%s] Recursion limit exceeded', TRIGGER_NAME))
        execution_count = 0
        return
    end

    -- 2. Timer de début
    local start_time = redis.call('TIME')

    -- 3. Logique métier avec gestion d'erreur
    local success, result = pcall(function()
        -- VOTRE LOGIQUE ICI
        redis.log(redis.LOG_NOTICE,
            string.format('[%s] Processing %s', TRIGGER_NAME, data.key))

        -- ...

        return true
    end)

    -- 4. Timer de fin
    local end_time = redis.call('TIME')
    local elapsed_ms = (end_time[1] - start_time[1]) * 1000 +
                       (end_time[2] - start_time[2]) / 1000

    -- 5. Logging si trop lent
    if elapsed_ms > MAX_EXECUTION_TIME_MS then
        redis.log(redis.LOG_WARNING,
            string.format('[%s] Slow execution: %.2fms', TRIGGER_NAME, elapsed_ms))
    end

    -- 6. Stats
    redis.call('HINCRBY', TRIGGER_NAME .. ':stats', 'total', 1)
    if success then
        redis.call('HINCRBY', TRIGGER_NAME .. ':stats', 'success', 1)
    else
        redis.call('HINCRBY', TRIGGER_NAME .. ':stats', 'errors', 1)
        redis.log(redis.LOG_ERROR,
            string.format('[%s] Error: %s', TRIGGER_NAME, tostring(result)))
    end

    -- 7. Nettoyage
    execution_count = execution_count - 1
end

-- === Utilitaires ===
local function enable_trigger(keys, args)
    redis.call('SET', 'trigger:enabled:' .. TRIGGER_NAME, '1')
    return 'Enabled'
end

local function disable_trigger(keys, args)
    redis.call('SET', 'trigger:enabled:' .. TRIGGER_NAME, '0')
    return 'Disabled'
end

local function get_trigger_stats(keys, args)
    return redis.call('HGETALL', TRIGGER_NAME .. ':stats')
end

-- === Enregistrement ===
redis.register_function(TRIGGER_NAME, my_trigger_function)
redis.register_function(TRIGGER_NAME .. '_enable', enable_trigger)
redis.register_function(TRIGGER_NAME .. '_disable', disable_trigger)
redis.register_function(TRIGGER_NAME .. '_stats', get_trigger_stats)

redis.register_keyspace_trigger(
    TRIGGER_NAME .. '_keyspace',
    'monitored:*',
    TRIGGER_NAME,
    {'prefix'}
)
```

## Conclusion

Les **Triggers and Functions** de Redis Stack apportent une dimension événementielle à Redis, permettant de créer des architectures réactives et automatisées. Les cas d'usage sont nombreux :

✅ **Avantages** :
- Automatisation complète des réactions
- Réduction du code client
- Cohérence centralisée de la logique
- Architecture événementielle native

⚠️ **Points d'attention** :
- Overhead de performance (0.5-5ms par événement)
- Complexité du debugging
- Risques de boucles infinies
- Nécessite Redis Stack (pas disponible en Redis OSS)

**Recommandation** : Utilisez les triggers pour des cas d'usage **non-critiques** où l'automatisation apporte une vraie valeur (audit, synchronisation, notifications). Pour la logique métier critique, préférez les Redis Functions appelées explicitement.

---

**📚 Points clés à retenir** :
- Triggers = **exécution automatique** en réponse à des événements
- Disponible uniquement dans **Redis Stack**
- Parfait pour : audit, sync, notifications, métriques
- Overhead : **~0.5-5ms** par événement
- Toujours protéger contre les **boucles infinies**
- Monitoring et **désactivation d'urgence** obligatoires en production

**🔜 Prochaine section** : [7.6 Quand utiliser Lua vs Transactions vs Functions ?](./06-lua-vs-transactions-vs-functions.md)

⏭️ [Quand utiliser Lua vs Transactions vs Functions ?](/07-atomicite-programmabilite/06-lua-vs-transactions-vs-functions.md)
