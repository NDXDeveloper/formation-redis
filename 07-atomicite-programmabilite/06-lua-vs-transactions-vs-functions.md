🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.6 Quand utiliser Lua vs Transactions vs Functions ?

## Introduction : Naviguer dans les options d'atomicité

Après avoir exploré en profondeur les différentes approches pour garantir l'atomicité dans Redis, une question essentielle se pose : **quelle solution choisir pour mon cas d'usage ?**

Ce guide de décision vous aidera à naviguer entre les cinq approches principales :

1. **Commandes atomiques natives** (INCR, GETSET, etc.)
2. **Transactions MULTI/EXEC**
3. **Optimistic Locking (WATCH)**
4. **Scripts Lua (EVAL/EVALSHA)**
5. **Redis Functions (Redis 7+)**

### Vue d'ensemble rapide

```
Complexité croissante →
Performance décroissante ↓

Commande native ──► MULTI/EXEC ──► WATCH ──► Lua Script ──► Redis Functions
    ⚡ Ultra-rapide     ⚡ Très rapide    ⚡ Rapide     ⚡ Rapide      ⚡ Rapide
    🎯 Cas simples      🎯 Séquences     🎯 CAS       🎯 Logique     🎯 Bibliothèques
    💡 Aucune config    💡 Simple        💡 Retry     💡 Atomique    💡 Persisté
    ❌ Très limité      ❌ Pas de CAS    ❌ Conflits  ❌ Complexe    ❌ Redis 7+
```

## Tableau comparatif complet

| Critère | Native | MULTI/EXEC | WATCH | Lua | Functions (7+) |
|---------|--------|------------|-------|-----|----------------|
| **Atomicité** | ✅ ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ ✅ |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Logique conditionnelle** | ❌ | ❌ | ⚠️ Limitée | ✅ ✅ ✅ | ✅ ✅ ✅ |
| **Latence (RTT)** | 1 RTT | 1 RTT | 2-10 RTT | 1 RTT | 1 RTT |
| **Retry nécessaire** | ❌ | ❌ | ✅ Oui | ❌ | ❌ |
| **Complexité code** | Minimale | Faible | Moyenne | Élevée | Élevée |
| **Persistance** | N/A | N/A | N/A | ⚠️ Cache | ✅ AOF/RDB |
| **Versioning** | N/A | N/A | N/A | Manuel | Intégré |
| **Debugging** | ✅ Simple | ✅ Simple | ⚠️ Moyen | ❌ Difficile | ✅ Meilleur |
| **Cluster support** | ✅ ✅ ✅ | ✅ ✅ | ✅ | ⚠️ Limité | ✅ ✅ |
| **Version Redis** | Toutes | Toutes | Toutes | 2.6+ | 7.0+ |
| **Réplication** | ✅ | ✅ | ✅ | ❌ Scripts | ✅ Automatique |
| **Organisation code** | N/A | Client | Client | Scripts | Bibliothèques |
| **Maintenance** | Aucune | Faible | Moyenne | Moyenne | Faible |
| **Cas d'usage** | 1 opération | Séquences | Read-Check-Write | Logique complexe | Production (7+) |

### Légende

- ✅ ✅ ✅ : Excellent
- ✅ ✅ : Très bon
- ✅ : Bon
- ⚠️ : Acceptable avec limitations
- ❌ : Non supporté ou problématique
- ⭐ : Performance (5 = meilleure)
- RTT : Round Trip Time (aller-retour réseau)

## Arbres de décision

### Arbre de décision principal

```
Avez-vous besoin d'atomicité ?
├─ Non → Utilisez des commandes simples
└─ Oui → Continuer ↓

Combien d'opérations ?
├─ Une seule opération
│   └─ → Commande atomique native (INCR, GETSET, etc.)
│
└─ Plusieurs opérations → Continuer ↓

Avez-vous besoin de logique conditionnelle ?
├─ Non (séquence simple)
│   └─ → MULTI/EXEC
│
└─ Oui → Continuer ↓

La condition dépend-elle de la valeur actuelle ?
├─ Non
│   └─ → Lua Script ou Functions
│
└─ Oui → Continuer ↓

Les conflits sont-ils rares (<10%) ?
├─ Oui
│   └─ → WATCH + retry
│
└─ Non (conflits fréquents)
    └─ → Lua Script ou Functions

Utilisez-vous Redis 7+ ?
├─ Oui
│   └─ → Redis Functions (recommandé)
│
└─ Non
    └─ → Lua Script (EVAL/EVALSHA)
```

### Arbre de décision par version Redis

```
Quelle version de Redis ?
│
├─ Redis < 7.0
│   ├─ Logique simple → MULTI/EXEC
│   ├─ Condition + lecture → WATCH
│   └─ Logique complexe → Lua Script
│
└─ Redis 7.0+
    ├─ Logique simple → MULTI/EXEC
    ├─ Condition + conflits rares → WATCH
    ├─ Scripts ponctuels → Lua Script
    └─ Code production → Redis Functions ✅ (recommandé)
```

### Arbre de décision par performance requise

```
Quelle est votre contrainte de latence ?
│
├─ < 1ms (ultra-critique)
│   ├─ Une opération → Commande native
│   └─ Plusieurs → MULTI/EXEC (si pas de condition)
│       └─ Avec condition → Lua Script (le plus court possible)
│
├─ < 5ms (haute performance)
│   ├─ Sans condition → MULTI/EXEC
│   └─ Avec condition → Lua Script ou Functions
│
└─ < 50ms (normale)
    ├─ Conflits rares → WATCH (acceptable avec retry)
    └─ Conflits fréquents → Lua Script ou Functions
```

## Cas d'usage détaillés

### Cas 1 : Incrément simple

**Besoin** : Incrémenter un compteur

```python
# ✅ MEILLEURE SOLUTION : Commande native
r.incr('page:views')
r.incrby('user:points', 10)

# ❌ OVERKILL : Ne pas utiliser MULTI/EXEC
# pipe = r.pipeline()
# pipe.incr('page:views')
# pipe.execute()

# ❌ OVERKILL : Ne pas utiliser Lua
# script = "return redis.call('INCR', KEYS[1])"
```

**Justification** :
- Une seule opération
- Déjà atomique par nature
- Performance maximale
- Simplicité maximale

---

### Cas 2 : Création d'utilisateur avec plusieurs structures

**Besoin** : Créer un utilisateur avec profil + index + stats

```python
# ✅ MEILLEURE SOLUTION : MULTI/EXEC
pipe = r.pipeline(transaction=True)
pipe.hset('user:1001:profile', mapping={
    'name': 'Alice',
    'email': 'alice@example.com'
})
pipe.sadd('index:emails', 'alice@example.com')
pipe.hincrby('stats:users', 'total', 1)
pipe.zadd('users:created', {1001: time.time()})
pipe.execute()

# ❌ INUTILE : Lua serait overkill ici
# (pas de logique conditionnelle)
```

**Justification** :
- Séquence d'opérations sans condition
- Atomicité garantie
- Simple à comprendre et maintenir
- Performance optimale

---

### Cas 3 : Transfert de points avec vérification

**Besoin** : Transférer des points entre utilisateurs si solde suffisant

#### Option A : WATCH (si conflits rares)

```python
# ✅ BON SI : Conflits < 10%
def transfer_with_watch(from_user, to_user, amount):
    max_retries = 5

    for attempt in range(max_retries):
        try:
            with r.pipeline() as pipe:
                pipe.watch(f'user:{from_user}:points')

                balance = int(pipe.get(f'user:{from_user}:points') or 0)

                if balance < amount:
                    pipe.unwatch()
                    return {'error': 'Insufficient points'}

                pipe.multi()
                pipe.decrby(f'user:{from_user}:points', amount)
                pipe.incrby(f'user:{to_user}:points', amount)
                pipe.execute()

                return {'success': True, 'attempts': attempt + 1}

        except redis.WatchError:
            if attempt < max_retries - 1:
                time.sleep(0.001 * (2 ** attempt))
                continue
            return {'error': 'Max retries exceeded'}
```

**Quand utiliser** :
- ✅ Conflits rares (< 10% des tentatives)
- ✅ Faible charge concurrente
- ✅ Pas de Redis 7 disponible

#### Option B : Lua Script (si conflits fréquents)

```lua
-- ✅ MEILLEUR SI : Conflits > 10%
local from_key = KEYS[1]
local to_key = KEYS[2]
local amount = tonumber(ARGV[1])

local balance = tonumber(redis.call('GET', from_key)) or 0

if balance < amount then
    return {ok = false, error = 'Insufficient points'}
end

redis.call('DECRBY', from_key, amount)
redis.call('INCRBY', to_key, amount)

return {ok = true, new_balance = balance - amount}
```

**Quand utiliser** :
- ✅ Conflits fréquents (> 10%)
- ✅ Haute concurrence
- ✅ Performance critique (1 seul RTT)
- ✅ Redis < 7.0

#### Option C : Redis Functions (Redis 7+)

```lua
-- ✅ MEILLEUR POUR PRODUCTION (Redis 7+)
#!lua name=payments

local function transfer_points(keys, args)
    local from_key = keys[1]
    local to_key = keys[2]
    local amount = tonumber(args[1])

    local balance = tonumber(redis.call('GET', from_key)) or 0

    if balance < amount then
        return {ok = false, error = 'Insufficient points'}
    end

    redis.call('DECRBY', from_key, amount)
    redis.call('INCRBY', to_key, amount)

    return {ok = true, new_balance = balance - amount}
end

redis.register_function('transfer_points', transfer_points)
```

**Quand utiliser** :
- ✅ Redis 7.0+ disponible
- ✅ Code production
- ✅ Besoin de persistance
- ✅ Maintenance long terme

---

### Cas 4 : Rate Limiting

**Besoin** : Limiter les requêtes API (100 req/minute par utilisateur)

#### Solution recommandée : Lua Script ou Functions

```lua
-- ✅ MEILLEURE SOLUTION : Lua (pour la complexité)
#!lua name=ratelimit

local function check_rate_limit(keys, args)
    local key = keys[1]
    local limit = tonumber(args[1])
    local window = tonumber(args[2])
    local now = tonumber(args[3])

    -- Nettoyer les anciennes entrées
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

    -- Compter les requêtes dans la fenêtre
    local count = redis.call('ZCARD', key)

    if count >= limit then
        return {
            allowed = false,
            remaining = 0,
            retry_after = window
        }
    end

    -- Ajouter la requête actuelle
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window)

    return {
        allowed = true,
        remaining = limit - count - 1,
        retry_after = 0
    }
end

redis.register_function('check_rate_limit', check_rate_limit)
```

**Pourquoi pas WATCH ?**
- ❌ Trop de retry sous charge
- ❌ Performance dégradée
- ❌ Logique trop complexe

**Pourquoi pas MULTI/EXEC ?**
- ❌ Impossible (besoin de logique conditionnelle)

---

### Cas 5 : Batch updates de 1000 clés

**Besoin** : Mettre à jour 1000 clés avec des valeurs différentes

```python
# ✅ MEILLEURE SOLUTION : MULTI/EXEC par lots
def batch_update_optimized(updates, batch_size=500):
    """
    updates: dict {'key1': 'value1', 'key2': 'value2', ...}
    """
    keys = list(updates.keys())

    for i in range(0, len(keys), batch_size):
        batch = keys[i:i + batch_size]

        pipe = r.pipeline(transaction=True)
        for key in batch:
            pipe.set(key, updates[key])
        pipe.execute()

        # Petite pause pour ne pas saturer
        if i + batch_size < len(keys):
            time.sleep(0.01)

# ❌ MAUVAIS : Un seul MULTI/EXEC pour tout
# Bloquerait Redis trop longtemps

# ❌ MAUVAIS : Lua script pour tout
# Même problème de blocage
```

**Justification** :
- Batching optimal (500 commandes par transaction)
- Équilibre performance / latence
- Ne bloque pas Redis trop longtemps
- Permet à d'autres clients d'interagir

---

### Cas 6 : Système de reservation de billets

**Besoin** : Réserver N billets si disponibles, avec lock temporaire

```lua
-- ✅ MEILLEURE SOLUTION : Lua/Functions (logique complexe)
#!lua name=ticketing

local function reserve_tickets(keys, args)
    local stock_key = keys[1]
    local reservation_key = keys[2]
    local quantity = tonumber(args[1])
    local user_id = args[2]
    local ttl = tonumber(args[3])

    -- Vérifier le stock
    local stock = tonumber(redis.call('GET', stock_key)) or 0

    if stock < quantity then
        return {
            success = false,
            error = 'insufficient_stock',
            available = stock,
            requested = quantity
        }
    end

    -- Décrémenter le stock
    redis.call('DECRBY', stock_key, quantity)

    -- Créer la réservation temporaire
    local reservation_id = redis.call('INCR', 'reservation:counter')

    redis.call('HSET', reservation_key,
        'id', reservation_id,
        'user_id', user_id,
        'quantity', quantity,
        'status', 'pending',
        'created_at', redis.call('TIME')[1]
    )
    redis.call('EXPIRE', reservation_key, ttl)

    return {
        success = true,
        reservation_id = reservation_id,
        remaining_stock = stock - quantity
    }
end

redis.register_function('reserve_tickets', reserve_tickets)
```

**Pourquoi cette solution ?**
- ✅ Logique complexe (vérification + modifications multiples)
- ✅ Atomicité critique (évite survente)
- ✅ Une seule transaction (performance)
- ✅ Code réutilisable et testable

---

### Cas 7 : Analytics en temps réel (HyperLogLog)

**Besoin** : Compter les visiteurs uniques par page

```python
# ✅ MEILLEURE SOLUTION : Commandes natives
def track_visitor(page_id, user_id):
    # HyperLogLog est déjà atomique
    r.pfadd(f'page:{page_id}:unique_visitors', user_id)
    r.incr(f'page:{page_id}:total_views')

def get_stats(page_id):
    return {
        'unique': r.pfcount(f'page:{page_id}:unique_visitors'),
        'total': int(r.get(f'page:{page_id}:total_views') or 0)
    }

# ❌ INUTILE : MULTI/EXEC ou Lua
# Les commandes sont déjà atomiques individuellement
```

---

### Cas 8 : Workflow avec états multiples

**Besoin** : Gérer un workflow avec transitions d'états validées

```lua
-- ✅ MEILLEURE SOLUTION : Lua/Functions (validation complexe)
#!lua name=workflow

local VALID_TRANSITIONS = {
    draft = {published = true, archived = true},
    published = {archived = true, draft = true},
    archived = {}
}

local function transition_state(keys, args)
    local doc_key = keys[1]
    local new_state = args[1]

    local current = redis.call('HGET', doc_key, 'state')

    if not current then
        return redis.error_reply('Document not found')
    end

    -- Validation de la transition
    if not VALID_TRANSITIONS[current] or
       not VALID_TRANSITIONS[current][new_state] then
        return redis.error_reply(
            string.format('Invalid transition: %s -> %s', current, new_state))
    end

    -- Appliquer la transition
    redis.call('HSET', doc_key,
        'state', new_state,
        'updated_at', redis.call('TIME')[1]
    )

    return {success = true, from = current, to = new_state}
end

redis.register_function('transition_state', transition_state)
```

**Pourquoi pas WATCH ?**
- ❌ Logique de validation trop complexe
- ❌ Devrait être côté client (pas idéal)
- ❌ Risque d'incohérence si code client varie

---

## Matrices de décision par contexte

### Par taille d'équipe

| Contexte | Recommandation | Justification |
|----------|----------------|---------------|
| **Solo developer** | MULTI/EXEC + Lua Scripts | Simple, flexible, pas besoin de sophistication |
| **Petite équipe (2-5)** | MULTI/EXEC + Lua Scripts | Organisation légère, maintenance faible |
| **Équipe moyenne (6-20)** | Functions (si 7+) sinon Lua | Meilleur versioning et organisation |
| **Grande équipe (20+)** | Functions + Bibliothèques | Organisation cruciale, maintenance long terme |

### Par criticité business

| Criticité | Tolérance conflit | Recommandation |
|-----------|-------------------|----------------|
| **Faible** | Élevée | WATCH acceptable |
| **Moyenne** | Moyenne | Lua Scripts |
| **Haute** | Faible | Lua Scripts / Functions |
| **Critique** | Aucune | Functions + Tests exhaustifs |

### Par charge concurrente

| Charge | Conflits | Solution optimale |
|--------|----------|-------------------|
| **Faible** (< 100 req/s) | < 1% | WATCH + retry |
| **Moyenne** (100-1K req/s) | 1-5% | WATCH ou Lua selon complexité |
| **Haute** (1K-10K req/s) | 5-15% | Lua Scripts obligatoire |
| **Très haute** (> 10K req/s) | > 15% | Lua Scripts + optimisation |

### Par complexité de logique

| Complexité | Description | Solution |
|------------|-------------|----------|
| **Triviale** | 1 commande | Commande native |
| **Simple** | Séquence linéaire | MULTI/EXEC |
| **Moyenne** | 1-2 conditions | WATCH ou Lua |
| **Complexe** | Boucles, multiples conditions | Lua Scripts |
| **Très complexe** | State machines, workflows | Functions (bibliothèques) |

## Patterns de migration

### Migration 1 : De commandes individuelles vers MULTI/EXEC

**Avant** :
```python
# ❌ 3 aller-retours réseau, pas atomique
r.set('user:1001:name', 'Alice')
r.sadd('users:active', 1001)
r.hincrby('stats:users', 'total', 1)
```

**Après** :
```python
# ✅ 1 aller-retour, atomique
pipe = r.pipeline(transaction=True)
pipe.set('user:1001:name', 'Alice')
pipe.sadd('users:active', 1001)
pipe.hincrby('stats:users', 'total', 1)
pipe.execute()
```

**Gain** :
- Latence : -66% (3 RTT → 1 RTT)
- Atomicité : ✅

---

### Migration 2 : De WATCH vers Lua (conflits fréquents)

**Avant** (WATCH avec 30% de retry) :
```python
def transfer_with_watch(from_id, to_id, amount):
    for _ in range(10):  # Souvent 3-5 tentatives nécessaires
        try:
            pipe.watch(f'balance:{from_id}')
            balance = int(pipe.get(f'balance:{from_id}'))
            # ... logique + MULTI/EXEC
        except redis.WatchError:
            continue
    # Moyenne : 3 tentatives = 6 RTT
```

**Après** (Lua) :
```lua
-- Toujours 1 tentative = 1 RTT
local balance = tonumber(redis.call('GET', KEYS[1]))
if balance >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    redis.call('INCRBY', KEYS[2], ARGV[1])
    return 1
end
return 0
```

**Gain** :
- Tentatives : 3-5 → 1 (toujours)
- Latence : -83% (6 RTT → 1 RTT en moyenne)
- CPU client : -50% (moins de retry logic)

---

### Migration 3 : De Lua vers Functions (Redis 7+)

**Avant** (Lua Scripts) :
```python
# Gestion manuelle du cache de scripts
increment_script = """
local current = tonumber(redis.call('GET', KEYS[1])) or 0
return redis.call('SET', KEYS[1], current + tonumber(ARGV[1]))
"""

sha = r.script_load(increment_script)

try:
    result = r.evalsha(sha, 1, 'counter', 5)
except redis.exceptions.NoScriptError:
    sha = r.script_load(increment_script)  # Recharger
    result = r.evalsha(sha, 1, 'counter', 5)
```

**Après** (Functions) :
```lua
#!lua name=counters

local function safe_increment(keys, args)
    local current = tonumber(redis.call('GET', keys[1])) or 0
    return redis.call('SET', keys[1], current + tonumber(args[1]))
end

redis.register_function('safe_increment', safe_increment)
```

```python
# Pas de gestion de cache nécessaire
result = r.fcall('safe_increment', 1, 'counter', 5)
```

**Gain** :
- Persistance : ✅ (survit aux redémarrts)
- Code plus propre : ✅
- Pas de gestion NoScriptError : ✅
- Organisation en bibliothèques : ✅

## Anti-patterns à éviter

### Anti-pattern 1 : Utiliser Lua pour une seule commande

```lua
-- ❌ MAUVAIS
local script = "return redis.call('INCR', KEYS[1])"
r.eval(script, 1, 'counter')

-- ✅ BON
r.incr('counter')
```

**Pourquoi c'est mauvais** :
- Overhead inutile (parsing Lua)
- Complexité sans bénéfice
- Moins performant qu'une commande native

---

### Anti-pattern 2 : MULTI/EXEC avec logique conditionnelle côté client

```python
# ❌ MAUVAIS : Race condition
balance = int(r.get('balance'))
if balance >= 100:
    pipe = r.pipeline()
    pipe.decrby('balance', 100)
    pipe.lpush('purchases', 'item')
    pipe.execute()
```

**Problème** : Entre le GET et le DECRBY, un autre client peut modifier le solde.

**Solution** : Utiliser WATCH ou Lua.

---

### Anti-pattern 3 : Transaction géante

```python
# ❌ MAUVAIS : Bloque Redis trop longtemps
pipe = r.pipeline(transaction=True)
for i in range(100000):  # 100k commandes !
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
```

**Problème** :
- Bloque tous les autres clients
- Consommation mémoire élevée
- Risque de timeout

**Solution** : Diviser en lots de 500-1000.

---

### Anti-pattern 4 : Lua script trop long

```lua
-- ❌ MAUVAIS : Script qui tourne 500ms
local cursor = "0"
repeat
    local result = redis.call('SCAN', cursor, 'MATCH', '*', 'COUNT', 1000)
    cursor = result[1]
    -- ... traitement ...
until cursor == "0"  -- Peut prendre 500ms !
```

**Problème** : Bloque Redis pendant toute l'exécution.

**Solution** :
- Limiter les itérations (max 10000)
- Batching côté client
- Utiliser un worker externe

---

### Anti-pattern 5 : Dépendance circulaire avec WATCH

```python
# ❌ MAUVAIS : Peut deadlock
def operation_a():
    pipe.watch('key1', 'key2')
    # ... utilise key1 puis key2 ...

def operation_b():
    pipe.watch('key2', 'key1')  # Ordre inversé !
    # ... utilise key2 puis key1 ...

# Si A et B s'exécutent en parallèle → retry infini possible
```

**Solution** : Toujours surveiller les clés dans le même ordre.

## Recommandations finales par scénario

### Startup / MVP

```
Priorité : Rapidité de développement

1. Commandes natives (80% des cas)
2. MULTI/EXEC (15% des cas)
3. Lua Scripts simples (5% des cas)

❌ Éviter : Sophistication prématurée
✅ Focus : Simplicité et itération rapide
```

### Passage en production

```
Priorité : Fiabilité

1. Audit du code existant
2. Identifier les race conditions
3. Ajouter MULTI/EXEC où nécessaire
4. Lua Scripts pour logique critique
5. Si Redis 7+ : migrer vers Functions

✅ Tests de charge obligatoires
✅ Monitoring des conflits WATCH
```

### Scale-up (haute charge)

```
Priorité : Performance

1. Profiler les hotspots
2. Remplacer WATCH par Lua si conflits > 10%
3. Optimiser les scripts Lua (< 5ms)
4. Batching agressif
5. Considérer le sharding

⚠️ Chaque milliseconde compte
⚠️ Éviter les scripts > 10ms
```

### Entreprise / Long terme

```
Priorité : Maintenabilité

1. Redis 7+ obligatoire
2. Organisation en bibliothèques Functions
3. Versioning strict
4. Tests unitaires exhaustifs
5. Documentation complète
6. Code review systématique

✅ Functions pour tout code production
✅ Lua seulement pour prototypage
```

### Migration legacy

```
Priorité : Sécurité

1. Audit complet du code Redis
2. Identifier les patterns problématiques
3. Migration progressive (feature flags)
4. Tests A/B entre ancien et nouveau
5. Rollback plan

⚠️ Ne jamais tout migrer d'un coup
✅ Approche incrémentale obligatoire
```

## Checklist de décision finale

### Avant de choisir une approche, vérifiez :

**Besoins fonctionnels** :
```
☐ Combien d'opérations Redis ? (1 / 2-5 / 5+)
☐ Y a-t-il de la logique conditionnelle ? (Oui / Non)
☐ La condition dépend-elle de valeurs Redis ? (Oui / Non)
☐ Besoin de boucles ou d'itérations ? (Oui / Non)
```

**Contraintes techniques** :
```
☐ Version de Redis ? (< 7.0 / 7.0+)
☐ Latence acceptable ? (< 1ms / < 5ms / < 50ms)
☐ Charge concurrente estimée ? (faible / moyenne / haute)
☐ Taux de conflits attendu ? (< 5% / 5-15% / > 15%)
```

**Contraintes projet** :
```
☐ Taille de l'équipe ? (solo / petite / grande)
☐ Criticité business ? (faible / moyenne / haute)
☐ Horizon de maintenance ? (< 6 mois / 6-24 mois / > 2 ans)
☐ Budget de test ? (limité / normal / exhaustif)
```

### Puis appliquez cette règle :

```
SI une_seule_operation:
    → Commande native

SINON SI pas_de_condition:
    → MULTI/EXEC

SINON SI condition_simple ET conflits_rares:
    → WATCH + retry

SINON SI redis_7_plus ET production:
    → Redis Functions

SINON:
    → Lua Scripts
```

## Tableau de synthèse final

| Scénario | Redis < 7 | Redis 7+ | Justification |
|----------|-----------|----------|---------------|
| Incrément simple | Native | Native | Déjà atomique |
| Séquence sans condition | MULTI/EXEC | MULTI/EXEC | Simple et rapide |
| Read-Check-Write, conflits rares | WATCH | WATCH | Optimistic locking |
| Read-Check-Write, conflits fréquents | Lua | Functions | Atomicité garantie |
| Logique complexe, prototype | Lua | Lua | Flexibilité |
| Logique complexe, production | Lua | **Functions** | Maintenance |
| Rate limiting | Lua | Functions | Logique + performance |
| Workflow / State machine | Lua | **Functions** | Organisation |
| Batch updates | MULTI/EXEC (lots) | MULTI/EXEC (lots) | Équilibre |
| Analytics temps réel | Native (HLL) | Native (HLL) | Optimisé |

## Conclusion

Le choix entre les différentes approches d'atomicité dans Redis n'est pas binaire. Chaque solution a sa place :

### Résumé des recommandations

1. **Commandes natives** : Toujours en premier choix pour les opérations simples
2. **MULTI/EXEC** : Pour les séquences sans logique conditionnelle
3. **WATCH** : Quand les conflits sont rares (< 10%) et la logique simple
4. **Lua Scripts** : Pour la logique complexe sur Redis < 7
5. **Redis Functions** : **Solution recommandée** pour toute logique complexe sur Redis 7+

### Règle d'or

> **"Utilisez la solution la plus simple qui répond à votre besoin."**

Ne pas chercher la perfection technique au détriment de la simplicité. Un code simple est :
- Plus facile à débugger
- Plus facile à maintenir
- Plus facile à optimiser si nécessaire
- Moins susceptible de contenir des bugs

### Évolution recommandée

```
Phase 1 (MVP):
  Commandes natives + MULTI/EXEC

Phase 2 (Production v1):
  + Lua Scripts pour cas critiques

Phase 3 (Scale):
  Optimisation des scripts
  Monitoring des performances

Phase 4 (Maturité):
  Migration vers Functions (Redis 7+)
  Organisation en bibliothèques
  Tests exhaustifs
```

### Derniers conseils

✅ **À FAIRE** :
- Commencer simple
- Mesurer avant d'optimiser
- Tester sous charge
- Documenter les choix
- Monitorer en production

❌ **À ÉVITER** :
- Sur-ingénierie prématurée
- Scripts Lua trop longs (> 50ms)
- WATCH avec taux de conflits élevé
- Transactions géantes (> 1000 commandes)
- Ignorer les tests de charge

---

**📚 Points clés à retenir** :
- Il n'y a **pas de solution universelle**
- **Commencer simple**, complexifier si nécessaire
- **MULTI/EXEC** pour 80% des cas d'usage
- **Lua/Functions** pour la logique complexe
- **Redis Functions** = futur pour Redis 7+
- **Mesurer et profiler** avant d'optimiser
- **Tester sous charge** avec la vraie concurrence

**🎯 Module suivant** : Module 8 - Communication et flux de données

---

*Ce guide de décision conclut le module sur l'atomicité et la programmabilité. Vous disposez maintenant de tous les outils pour faire des choix éclairés selon votre contexte spécifique.*

⏭️ [Communication et flux de données](/08-communication-flux-donnees/README.md)
