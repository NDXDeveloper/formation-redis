🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.1 Transactions : MULTI/EXEC et pipeline transactionnel

## Introduction : Qu'est-ce qu'une transaction dans Redis ?

Les transactions Redis diffèrent fondamentalement des transactions ACID traditionnelles des bases de données relationnelles. Comprendre ces différences est crucial pour éviter les erreurs conceptuelles qui peuvent mener à des bugs de production subtils.

### Le modèle transactionnel de Redis

```redis
MULTI               # Début de la transaction
SET user:100:name "Alice"
INCR user:100:visits
LPUSH user:100:actions "login"
EXEC                # Exécution atomique
```

**Ce que Redis garantit** :
- ✅ **Isolation** : Les commandes sont mises en file d'attente et exécutées séquentiellement
- ✅ **Atomicité d'exécution** : Toutes les commandes s'exécutent d'un seul bloc
- ✅ **Ordre d'exécution** : Les commandes sont exécutées dans l'ordre exact

**Ce que Redis ne garantit PAS** :
- ❌ **Rollback automatique** : Si une commande échoue, les précédentes ne sont pas annulées
- ❌ **Isolation pendant la mise en file** : Entre MULTI et EXEC, d'autres clients peuvent modifier les données
- ❌ **Vérifications de contraintes** : Pas de validation de cohérence avant exécution

### Comparaison avec les transactions SQL

| Aspect | Redis (MULTI/EXEC) | SQL (BEGIN/COMMIT) |
|--------|-------------------|-------------------|
| **Isolation** | ⚠️ Partielle (uniquement pendant EXEC) | ✅ Complète (configurable) |
| **Atomicité** | ✅ Exécution groupée | ✅ Tout ou rien |
| **Rollback** | ❌ Non supporté | ✅ ROLLBACK complet |
| **Durabilité** | ⚠️ Dépend de la config de persistance | ✅ Garantie après COMMIT |
| **Cohérence** | ❌ Pas de contraintes | ✅ Contraintes d'intégrité |
| **Performance** | ✅ Ultra-rapide | ⚠️ Overhead significatif |

## Fonctionnement interne de MULTI/EXEC

### Phase 1 : Mise en file d'attente (MULTI → EXEC)

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET key1 "value1"
QUEUED
127.0.0.1:6379> INCR counter
QUEUED
127.0.0.1:6379> GET key1
QUEUED
```

**Que se passe-t-il en interne** :

1. **MULTI** : Redis crée une file d'attente temporaire pour cette connexion
2. **Commandes suivantes** : Chaque commande est validée syntaxiquement puis ajoutée à la file
3. **Réponse immédiate** : Le client reçoit "QUEUED" sans exécution réelle
4. **État de la connexion** : Marquée comme "en transaction"

```
État mémoire pendant MULTI:
┌─────────────────────────────────┐
│  Client Connection Context      │
├─────────────────────────────────┤
│  • in_transaction: true         │
│  • command_queue: [             │
│      SET key1 "value1",         │
│      INCR counter,              │
│      GET key1                   │
│    ]                            │
└─────────────────────────────────┘
```

### Phase 2 : Exécution atomique (EXEC)

```redis
127.0.0.1:6379> EXEC
1) OK
2) (integer) 1
3) "value1"
```

**Processus d'exécution** :

1. **Verrouillage** : Redis empêche toute autre commande de s'intercaler
2. **Exécution séquentielle** : Chaque commande de la file est exécutée dans l'ordre
3. **Collecte des résultats** : Les résultats sont agrégés dans un tableau
4. **Réponse groupée** : Un seul aller-retour réseau pour tous les résultats
5. **Nettoyage** : La file est vidée, la connexion repasse en mode normal

```
Timeline d'exécution:
Time ──────────────────────────────────────►
     │
     │ MULTI
     │   ↓ (queue commands)
     │ SET, INCR, GET
     │   ↓
     │ EXEC ─────┐
     │           │ [Atomic Block - No Interruption]
     │           │ • Execute SET
     │           │ • Execute INCR
     │           │ • Execute GET
     │           └──► Return results array
     │
```

## Exemple détaillé : Système de transfert de points

Imaginons un système de gamification où les utilisateurs peuvent transférer des points entre eux.

### Version sans transaction (VULNÉRABLE)

```python
import redis

r = redis.Redis(decode_responses=True)

def transfer_points_unsafe(from_user, to_user, amount):
    """
    ❌ DANGEREUX : Race condition possible
    """
    # 1. Vérifier le solde (lecture)
    from_balance = int(r.get(f"user:{from_user}:points") or 0)

    if from_balance < amount:
        return {"error": "Insufficient points"}

    # ⚠️ DANGER : Entre cette vérification et les opérations suivantes,
    # un autre client peut modifier les soldes !

    # 2. Déduire les points (écriture)
    r.decrby(f"user:{from_user}:points", amount)

    # 3. Ajouter les points (écriture)
    r.incrby(f"user:{to_user}:points", amount)

    return {"success": True}

# Scénario problématique :
# Thread A: transfer_points_unsafe("alice", "bob", 100)
#   → lit balance=100, OK
# Thread B: transfer_points_unsafe("alice", "charlie", 100)
#   → lit balance=100, OK (avant que A ne déduise)
# Thread A: déduit 100 → balance=0
# Thread B: déduit 100 → balance=-100 ❌ CORRUPTION !
```

### Version avec MULTI/EXEC (PARTIELLE)

```python
def transfer_points_with_transaction(from_user, to_user, amount):
    """
    ⚠️ INCOMPLET : La transaction ne résout pas tout
    """
    # Vérification HORS transaction
    from_balance = int(r.get(f"user:{from_user}:points") or 0)

    if from_balance < amount:
        return {"error": "Insufficient points"}

    # ⚠️ PROBLÈME : La vérification et la transaction sont séparées
    # La balance peut changer entre les deux !

    # Transaction pour les écritures
    pipe = r.pipeline(transaction=True)
    pipe.decrby(f"user:{from_user}:points", amount)
    pipe.incrby(f"user:{to_user}:points", amount)

    # Enregistrement de la transaction
    tx_id = r.incr("transaction:counter")
    pipe.hset(f"transaction:{tx_id}", mapping={
        "from": from_user,
        "to": to_user,
        "amount": amount,
        "timestamp": int(time.time())
    })

    results = pipe.execute()
    return {"success": True, "transaction_id": tx_id}
```

**Problème identifié** : La vérification du solde est effectuée AVANT MULTI, créant une fenêtre de vulnérabilité. La solution complète nécessite WATCH (voir section 7.2).

### Transaction correcte avec logique simple

```python
def simple_transfer_with_transaction(from_user, to_user, amount):
    """
    ✅ CORRECT pour des transferts sans vérification préalable
    ou avec validation côté application
    """
    pipe = r.pipeline(transaction=True)

    # Toutes les commandes sont atomiques
    pipe.decrby(f"user:{from_user}:points", amount)
    pipe.incrby(f"user:{to_user}:points", amount)

    # Log de l'opération
    tx_id = r.incr("transaction:counter")
    pipe.hset(f"transaction:{tx_id}", mapping={
        "from": from_user,
        "to": to_user,
        "amount": amount,
        "timestamp": int(time.time()),
        "status": "completed"
    })
    pipe.expire(f"transaction:{tx_id}", 86400)  # Expire après 24h

    try:
        results = pipe.execute()
        return {
            "success": True,
            "transaction_id": tx_id,
            "from_new_balance": results[0],
            "to_new_balance": results[1]
        }
    except redis.RedisError as e:
        return {"error": str(e)}
```

## Anatomie d'une transaction Redis en Node.js

```javascript
const Redis = require('ioredis');
const redis = new Redis();

/**
 * Enregistrement d'une commande e-commerce
 * Démontre l'utilisation de MULTI/EXEC avec gestion d'erreur
 */
async function createOrder(userId, productId, quantity, price) {
    const orderId = await redis.incr('order:counter');
    const timestamp = Date.now();

    // Création du pipeline transactionnel
    const pipeline = redis.multi();

    // 1. Créer l'objet commande
    pipeline.hset(`order:${orderId}`, {
        user_id: userId,
        product_id: productId,
        quantity: quantity,
        price: price,
        total: price * quantity,
        status: 'pending',
        created_at: timestamp
    });

    // 2. Ajouter à l'index des commandes de l'utilisateur
    pipeline.zadd(`user:${userId}:orders`, timestamp, orderId);

    // 3. Ajouter à l'index global des commandes en attente
    pipeline.zadd('orders:pending', timestamp, orderId);

    // 4. Mettre à jour les statistiques
    pipeline.hincrby('stats:daily', 'total_orders', 1);
    pipeline.hincrbyfloat('stats:daily', 'total_revenue', price * quantity);

    // 5. Décrémenter le stock (sans vérification dans cette version simple)
    pipeline.decrby(`product:${productId}:stock`, quantity);

    try {
        // Exécution atomique de toutes les commandes
        const results = await pipeline.exec();

        // Vérification des résultats (ioredis retourne [error, result] pour chaque commande)
        const errors = results.filter(([err]) => err !== null);
        if (errors.length > 0) {
            console.error('Transaction had errors:', errors);
            return { success: false, errors };
        }

        return {
            success: true,
            orderId,
            message: 'Order created successfully'
        };

    } catch (error) {
        console.error('Transaction failed:', error);
        return { success: false, error: error.message };
    }
}

// Utilisation
createOrder(1001, 5042, 2, 29.99)
    .then(result => console.log(result));
```

### Analyse détaillée du code

**Points clés** :

1. **Pipeline vs Transaction** :
```javascript
// Pipeline simple (pas de transaction)
const pipeline = redis.pipeline();

// Pipeline transactionnel (avec MULTI/EXEC)
const pipeline = redis.multi();
```

2. **Gestion des résultats** :
```javascript
// ioredis retourne un tableau de [error, result]
const results = await pipeline.exec();
// results = [
//   [null, 'OK'],              // HSET réussi
//   [null, 1],                 // ZADD réussi
//   [null, 1],                 // ZADD réussi
//   [null, 1],                 // HINCRBY réussi
//   [null, '59.98'],           // HINCRBYFLOAT réussi
//   [Error('...'), undefined]  // DECRBY échoué (si erreur)
// ]
```

3. **Pas de rollback automatique** :
Si `DECRBY` échoue (par exemple, stock devient négatif), les commandes précédentes restent appliquées. C'est une limitation fondamentale de Redis.

## Pipeline vs Transaction : Clarification

### Pipeline simple (sans MULTI/EXEC)

```python
# Pipeline NON transactionnel
pipe = r.pipeline(transaction=False)
pipe.get("key1")
pipe.set("key2", "value2")
pipe.incr("counter")
results = pipe.execute()

# ⚠️ D'autres clients peuvent s'intercaler entre les commandes !
```

**Caractéristiques** :
- Optimisation réseau : Toutes les commandes envoyées en un seul bloc
- Réponses groupées : Un seul aller-retour pour tous les résultats
- **Pas d'atomicité** : D'autres commandes peuvent s'exécuter entre les vôtres
- Usage : Quand vous voulez juste optimiser le réseau, pas garantir l'atomicité

### Pipeline transactionnel (avec MULTI/EXEC)

```python
# Pipeline transactionnel
pipe = r.pipeline(transaction=True)  # ou juste r.pipeline() (défaut)
pipe.get("key1")
pipe.set("key2", "value2")
pipe.incr("counter")
results = pipe.execute()

# ✅ Exécution atomique garantie
```

**Caractéristiques** :
- Tout ce qu'offre le pipeline simple
- **Plus** : Atomicité d'exécution (MULTI/EXEC automatique)
- Usage : Quand vous avez besoin d'atomicité ET d'optimisation réseau

### Comparaison visuelle

```
Pipeline simple (transaction=False):
Client → Redis: GET key1
                ↓ Execute immediately
Client ← Redis: "value1"
                [Other clients can execute commands here]
Client → Redis: SET key2 "value2"
                ↓ Execute immediately
Client ← Redis: OK
                [Other clients can execute commands here]
Client → Redis: INCR counter
                ↓ Execute immediately
Client ← Redis: (integer) 5

═══════════════════════════════════════════════════════

Pipeline transactionnel (transaction=True):
Client → Redis: MULTI
Client → Redis: GET key1
Client → Redis: SET key2 "value2"
Client → Redis: INCR counter
Client → Redis: EXEC
                ↓ Execute ALL atomically
                [No other commands can interrupt]
Client ← Redis: [
                  "value1",
                  OK,
                  (integer) 5
                ]
```

## Gestion des erreurs dans les transactions

### Types d'erreurs

Redis distingue deux types d'erreurs dans les transactions :

#### 1. Erreurs de syntaxe (avant EXEC)

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET key value
QUEUED
127.0.0.1:6379> INVALID_COMMAND
(error) ERR unknown command 'INVALID_COMMAND'
127.0.0.1:6379> INCR counter
QUEUED
127.0.0.1:6379> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
```

**Comportement** :
- La transaction est complètement abandonnée
- Aucune commande n'est exécutée
- Redis détecte l'erreur immédiatement

#### 2. Erreurs d'exécution (pendant EXEC)

```redis
127.0.0.1:6379> SET mykey "string_value"
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET key1 "value1"
QUEUED
127.0.0.1:6379> INCR mykey
QUEUED  # Syntaxe valide, mais mykey n'est pas un nombre
127.0.0.1:6379> SET key2 "value2"
QUEUED
127.0.0.1:6379> EXEC
1) OK
2) (error) ERR value is not an integer or out of range
3) OK
```

**Comportement CRITIQUE** :
- ⚠️ Les commandes avant l'erreur **SONT APPLIQUÉES**
- ⚠️ Les commandes après l'erreur **SONT AUSSI APPLIQUÉES**
- ⚠️ **Pas de rollback automatique**

### Pattern de gestion des erreurs en Python

```python
def transaction_with_error_handling(r: redis.Redis):
    """
    Démontre la gestion robuste des erreurs dans une transaction
    """
    pipe = r.pipeline(transaction=True)

    try:
        # Phase 1 : Mise en file d'attente
        pipe.set("user:1000:name", "Alice")
        pipe.incrby("user:1000:score", 10)
        pipe.lpush("user:1000:actions", "achievement_unlocked")

        # Phase 2 : Exécution
        results = pipe.execute()

        # Phase 3 : Validation des résultats
        if not all(results):
            # Certaines commandes ont échoué, mais déjà appliquées !
            log_transaction_warning({
                'message': 'Partial transaction success',
                'results': results
            })
            # Compensation manuelle si nécessaire
            compensate_partial_transaction()

        return {"success": True, "results": results}

    except redis.exceptions.ResponseError as e:
        # Erreur de syntaxe détectée avant EXEC
        log_error(f"Transaction aborted: {e}")
        return {"success": False, "error": "syntax_error"}

    except redis.exceptions.ConnectionError as e:
        # Problème réseau
        log_error(f"Connection error: {e}")
        return {"success": False, "error": "connection_error"}

    except Exception as e:
        # Autres erreurs
        log_error(f"Unexpected error: {e}")
        return {"success": False, "error": "unknown_error"}

def compensate_partial_transaction():
    """
    Logique de compensation manuelle en cas d'échec partiel
    """
    # Exemple : Enregistrer dans une queue pour traitement ultérieur
    r.lpush("failed_transactions", json.dumps({
        "timestamp": time.time(),
        "operation": "user_achievement",
        "status": "requires_manual_review"
    }))
```

## Cas d'usage optimaux pour MULTI/EXEC

### ✅ Cas 1 : Mise à jour cohérente de plusieurs structures

```python
def update_user_profile(user_id: str, profile_data: dict):
    """
    Mise à jour atomique d'un profil utilisateur réparti sur plusieurs structures
    """
    pipe = r.pipeline(transaction=True)

    # 1. Hash principal du profil
    pipe.hset(f"user:{user_id}:profile", mapping={
        "name": profile_data["name"],
        "email": profile_data["email"],
        "updated_at": int(time.time())
    })

    # 2. Index de recherche par email
    old_email = r.hget(f"user:{user_id}:profile", "email")
    if old_email and old_email != profile_data["email"]:
        pipe.srem("index:emails", old_email)
    pipe.sadd("index:emails", profile_data["email"])

    # 3. Statistiques de mise à jour
    pipe.hincrby("stats:profile_updates", "total", 1)
    pipe.hincrby("stats:profile_updates", f"user:{user_id}", 1)

    # 4. Log d'audit
    pipe.zadd("audit:profile_updates", {
        f"{user_id}:{int(time.time())}": int(time.time())
    })

    pipe.execute()
```

### ✅ Cas 2 : Incréments multiples atomiques

```python
def record_page_view(page_id: str, user_id: str, session_id: str):
    """
    Enregistrement atomique d'une vue de page dans plusieurs compteurs
    """
    timestamp = int(time.time())

    pipe = r.pipeline(transaction=True)

    # Compteurs globaux
    pipe.incr(f"page:{page_id}:views:total")
    pipe.incr(f"page:{page_id}:views:today:{time.strftime('%Y-%m-%d')}")
    pipe.incr(f"user:{user_id}:views:total")

    # HyperLogLog pour visiteurs uniques
    pipe.pfadd(f"page:{page_id}:unique_visitors", user_id)
    pipe.pfadd(f"page:{page_id}:unique_visitors:today", user_id)

    # Timeline de vues
    pipe.zadd(f"page:{page_id}:views:timeline", {session_id: timestamp})

    # Top pages de l'utilisateur
    pipe.zincrby(f"user:{user_id}:top_pages", 1, page_id)

    results = pipe.execute()

    return {
        "total_views": results[0],
        "views_today": results[1],
        "unique_visitors": results[3]
    }
```

### ✅ Cas 3 : Opérations de nettoyage groupées

```python
def cleanup_expired_sessions(max_age_seconds: int = 3600):
    """
    Nettoyage atomique des sessions expirées
    """
    current_time = int(time.time())
    cutoff_time = current_time - max_age_seconds

    # Trouver les sessions expirées
    expired = r.zrangebyscore("sessions:active", 0, cutoff_time)

    if not expired:
        return {"cleaned": 0}

    pipe = r.pipeline(transaction=True)

    for session_id in expired:
        # Supprimer la session
        pipe.delete(f"session:{session_id}")

        # Retirer de l'index actif
        pipe.zrem("sessions:active", session_id)

        # Ajouter à l'index des sessions expirées (pour analyse)
        pipe.zadd("sessions:expired", {session_id: current_time})

    # Statistiques
    pipe.hincrby("stats:cleanup", "sessions_cleaned", len(expired))
    pipe.hset("stats:cleanup", "last_cleanup", current_time)

    pipe.execute()

    return {"cleaned": len(expired), "timestamp": current_time}
```

## ❌ Anti-patterns : Quand NE PAS utiliser MULTI/EXEC

### Anti-pattern 1 : Logique conditionnelle dans la transaction

```python
# ❌ NE FONCTIONNE PAS : Les conditions ne peuvent pas utiliser les résultats
def bad_conditional_transaction(user_id: str):
    pipe = r.pipeline(transaction=True)

    pipe.get(f"user:{user_id}:balance")
    # ⚠️ PROBLÈME : On ne peut pas lire le résultat avant EXEC !

    # Cette condition ne fonctionnera PAS comme attendu
    # if balance > 100:  # On n'a pas encore le résultat !
    #     pipe.decrby(f"user:{user_id}:balance", 50)

    pipe.execute()
    # Les résultats ne sont disponibles qu'ici

# ✅ SOLUTION : Utiliser WATCH (voir section 7.2) ou Lua (section 7.3)
```

### Anti-pattern 2 : Transactions trop longues

```python
# ❌ MAUVAIS : Trop de commandes dans une transaction
def bad_bulk_operation():
    pipe = r.pipeline(transaction=True)

    # Danger : 10,000 commandes dans une seule transaction
    for i in range(10000):
        pipe.set(f"key:{i}", f"value:{i}")

    # ⚠️ Bloque Redis pendant toute l'exécution
    # ⚠️ Consomme beaucoup de mémoire pour la file d'attente
    # ⚠️ Augmente la latence pour tous les autres clients
    pipe.execute()

# ✅ MEILLEUR : Diviser en petits lots
def good_bulk_operation(batch_size: int = 1000):
    keys_values = [(f"key:{i}", f"value:{i}") for i in range(10000)]

    for i in range(0, len(keys_values), batch_size):
        batch = keys_values[i:i + batch_size]
        pipe = r.pipeline(transaction=True)

        for key, value in batch:
            pipe.set(key, value)

        pipe.execute()
        time.sleep(0.01)  # Pause entre les lots pour ne pas monopoliser Redis
```

### Anti-pattern 3 : Utiliser MULTI/EXEC pour une seule commande

```python
# ❌ INUTILE : Overhead sans bénéfice
def bad_single_command():
    pipe = r.pipeline(transaction=True)
    pipe.incr("counter")
    result = pipe.execute()
    return result[0]

# ✅ BON : Commande directe (déjà atomique)
def good_single_command():
    return r.incr("counter")
```

## Performance et optimisations

### Benchmark : Impact du pipelining

```python
import time
import redis

r = redis.Redis()

# Test 1 : Commandes individuelles
start = time.time()
for i in range(1000):
    r.set(f"key:{i}", f"value:{i}")
time_individual = time.time() - start

# Test 2 : Pipeline simple
start = time.time()
pipe = r.pipeline(transaction=False)
for i in range(1000):
    pipe.set(f"key:{i}", f"value:{i}")
pipe.execute()
time_pipeline = time.time() - start

# Test 3 : Transaction (MULTI/EXEC)
start = time.time()
pipe = r.pipeline(transaction=True)
for i in range(1000):
    pipe.set(f"key:{i}", f"value:{i}")
pipe.execute()
time_transaction = time.time() - start

print(f"Individuel    : {time_individual:.3f}s")
print(f"Pipeline      : {time_pipeline:.3f}s ({time_individual/time_pipeline:.1f}x plus rapide)")
print(f"Transaction   : {time_transaction:.3f}s ({time_individual/time_transaction:.1f}x plus rapide)")

# Résultats typiques (localhost) :
# Individuel    : 0.523s
# Pipeline      : 0.018s (29.1x plus rapide)
# Transaction   : 0.019s (27.5x plus rapide)
```

**Analyse** :
- Le pipeline élimine la latence réseau (RTT) entre les commandes
- La différence entre pipeline et transaction est minime (overhead de MULTI/EXEC négligeable)
- Sur un réseau distant (RTT élevé), le gain est encore plus spectaculaire

### Optimisation : Taille des lots (batch size)

```python
def optimal_batch_size_finder(total_operations: int):
    """
    Trouve la taille de lot optimale pour votre environnement
    """
    batch_sizes = [100, 500, 1000, 2000, 5000]
    results = {}

    for batch_size in batch_sizes:
        start = time.time()

        for i in range(0, total_operations, batch_size):
            pipe = r.pipeline(transaction=True)

            for j in range(min(batch_size, total_operations - i)):
                pipe.set(f"test:key:{i+j}", f"value:{i+j}")

            pipe.execute()

        duration = time.time() - start
        results[batch_size] = duration
        print(f"Batch size {batch_size:5d}: {duration:.3f}s")

    # Nettoyage
    r.flushdb()

    return results

# Utilisation
optimal_batch_size_finder(10000)

# Résultats typiques :
# Batch size   100: 0.892s
# Batch size   500: 0.234s  ← Bon compromis
# Batch size  1000: 0.189s  ← Meilleure performance
# Batch size  2000: 0.198s
# Batch size  5000: 0.312s  (trop grand, overhead mémoire)
```

**Recommandation** :
- Pour la plupart des cas : **500-1000 commandes** par transaction
- Opérations complexes : **100-500**
- Opérations simples : **1000-2000**

## Transactions imbriquées et comportement spécial

### Transactions ne peuvent PAS être imbriquées

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET key1 "value1"
QUEUED
127.0.0.1:6379> MULTI
(error) ERR MULTI calls can not be nested
```

**Explication** : Redis ne supporte pas les transactions imbriquées. Si vous avez besoin de logique complexe, utilisez Lua ou Redis Functions.

### Annulation d'une transaction avec DISCARD

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET key1 "value1"
QUEUED
127.0.0.1:6379> INCR counter
QUEUED
127.0.0.1:6379> DISCARD
OK
127.0.0.1:6379> GET key1
(nil)  # Rien n'a été exécuté
```

**Usage en Python** :

```python
def transaction_with_validation(user_id: str, amount: int):
    """
    Transaction avec validation et possibilité d'annulation
    """
    pipe = r.pipeline(transaction=True)

    try:
        # Validation métier
        current_balance = int(r.get(f"user:{user_id}:balance") or 0)
        if current_balance < amount:
            # Annuler la transaction (si elle était démarrée)
            pipe.reset()  # Équivalent de DISCARD
            return {"error": "Insufficient funds"}

        # Transaction valide
        pipe.decrby(f"user:{user_id}:balance", amount)
        pipe.lpush(f"user:{user_id}:transactions", json.dumps({
            "amount": -amount,
            "timestamp": time.time()
        }))

        results = pipe.execute()
        return {"success": True, "new_balance": results[0]}

    except Exception as e:
        pipe.reset()  # Annulation en cas d'erreur
        raise
```

## Comparaison avec les alternatives

### MULTI/EXEC vs WATCH (Optimistic Locking)

```python
# MULTI/EXEC : Pas de vérification d'état
def multi_exec_approach(user_id: str, amount: int):
    pipe = r.pipeline(transaction=True)
    pipe.decrby(f"user:{user_id}:balance", amount)
    pipe.execute()
    # ⚠️ Pas de vérification si le solde est suffisant

# WATCH : Vérification d'état avec retry
def watch_approach(user_id: str, amount: int):
    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(f"user:{user_id}:balance")
                balance = int(pipe.get(f"user:{user_id}:balance") or 0)

                if balance < amount:
                    pipe.unwatch()
                    return {"error": "Insufficient funds"}

                pipe.multi()
                pipe.decrby(f"user:{user_id}:balance", amount)
                pipe.execute()
                return {"success": True}

            except redis.WatchError:
                # Retry si la valeur a changé
                continue
```

**Quand utiliser quoi** :
- **MULTI/EXEC** : Opérations simples sans conditions
- **WATCH** : Opérations conditionnelles, conflicts rares
- **Lua/Functions** : Logique complexe, conflicts fréquents

### MULTI/EXEC vs Lua Script

```python
# MULTI/EXEC : Limité à des séquences linéaires
def multi_exec_limited():
    pipe = r.pipeline(transaction=True)
    pipe.get("key1")
    # ❌ On ne peut pas utiliser le résultat dans la transaction
    pipe.set("key2", "fixed_value")
    pipe.execute()

# Lua : Logique complète sur le serveur
lua_script = """
local value = redis.call('GET', KEYS[1])
if tonumber(value) > 10 then
    redis.call('SET', KEYS[2], 'high')
else
    redis.call('SET', KEYS[2], 'low')
end
return value
"""

result = r.eval(lua_script, 2, "key1", "key2")
```

## Debugging et monitoring des transactions

### Suivre les transactions en temps réel

```bash
# Monitorer toutes les commandes
redis-cli MONITOR

# Filtrer uniquement les transactions
redis-cli MONITOR | grep -E "MULTI|EXEC|DISCARD"
```

### Analyser les transactions lentes

```bash
# Voir les commandes lentes (inclut les transactions)
redis-cli SLOWLOG GET 10

# Configuration du seuil de slowlog (microsecondes)
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms
```

### Métriques importantes

```python
def get_transaction_metrics(r: redis.Redis):
    """
    Collecte des métriques sur les transactions
    """
    info = r.info('stats')

    return {
        "total_commands": info['total_commands_processed'],
        "instantaneous_ops": info['instantaneous_ops_per_sec'],
        "rejected_connections": info['rejected_connections'],
        "sync_full": info.get('sync_full', 0),
        "sync_partial_ok": info.get('sync_partial_ok', 0)
    }
```

## Résumé et bonnes pratiques

### ✅ À FAIRE

1. **Utiliser les transactions pour garantir l'atomicité** de séquences d'opérations liées
2. **Grouper les opérations** pour réduire la latence réseau
3. **Limiter la taille** des transactions (500-1000 commandes max)
4. **Gérer les erreurs** explicitement et prévoir la compensation
5. **Tester les cas limites** : erreurs, timeouts, failures partielles
6. **Monitorer les performances** avec SLOWLOG et métriques

### ❌ À ÉVITER

1. **Ne pas utiliser pour la logique conditionnelle** (préférer WATCH ou Lua)
2. **Ne pas créer de transactions géantes** (milliers de commandes)
3. **Ne pas compter sur le rollback automatique** (n'existe pas)
4. **Ne pas imbriquer les MULTI** (non supporté)
5. **Ne pas utiliser pour une seule commande** (overhead inutile)

### Checklist de décision

```
Ai-je besoin d'atomicité ?
├─ Non → Commandes individuelles ou pipeline simple
└─ Oui → Continuer ↓

Ai-je besoin de logique conditionnelle ?
├─ Oui → Utiliser WATCH ou Lua/Functions
└─ Non → Continuer ↓

Les opérations sont-elles nombreuses (>1000) ?
├─ Oui → Diviser en lots + considérer Lua
└─ Non → Continuer ↓

                → Utiliser MULTI/EXEC ✅
```

## Conclusion

Les transactions MULTI/EXEC sont un outil puissant mais spécialisé dans l'arsenal Redis. Elles excellent dans des scénarios spécifiques :
- Mise à jour cohérente de structures multiples
- Optimisation réseau via pipelining
- Opérations groupées sans logique conditionnelle

Cependant, elles ont des limitations importantes :
- Pas de rollback automatique
- Pas de logique conditionnelle pendant la transaction
- Ne conviennent pas aux opérations très longues

Pour des besoins plus complexes, les sections suivantes exploreront WATCH pour l'optimistic locking et Lua/Functions pour la logique métier avancée.

---

**📚 Points clés à retenir** :
- MULTI/EXEC garantit l'**atomicité d'exécution** mais **pas le rollback**
- Les erreurs d'exécution n'annulent **pas** les commandes précédentes
- Pipeline transactionnel = optimisation réseau + atomicité
- Limiter les transactions à 500-1000 commandes pour des performances optimales
- Pour la logique conditionnelle, utiliser WATCH (section 7.2) ou Lua (section 7.3)

**🔜 Prochaine section** : [7.2 Optimistic Locking avec WATCH](./02-optimistic-locking-watch.md)

⏭️ [Optimistic Locking avec WATCH](/07-atomicite-programmabilite/02-optimistic-locking-watch.md)
