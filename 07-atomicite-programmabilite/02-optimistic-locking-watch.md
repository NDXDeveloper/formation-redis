🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2 Optimistic Locking avec WATCH

## Introduction : Le problème du Read-Modify-Write

Dans la section précédente, nous avons vu que MULTI/EXEC permet de grouper des commandes de manière atomique, mais avec une limitation majeure : **impossible d'effectuer une logique conditionnelle basée sur les valeurs actuelles**.

### Le scénario problématique

Imaginez un système de réservation d'hôtel :

```python
# ❌ PROBLÈME : Race condition inévitable
def book_room_unsafe(room_id: str, user_id: str):
    # 1. Lire le statut de la chambre
    status = r.get(f"room:{room_id}:status")

    # 2. Vérifier la disponibilité
    if status != "available":
        return {"error": "Room not available"}

    # ⚠️ DANGER : Entre cette vérification et la réservation,
    # un autre client peut réserver la même chambre !

    # 3. Effectuer la réservation
    r.set(f"room:{room_id}:status", "booked")
    r.set(f"room:{room_id}:guest", user_id)

    return {"success": True}
```

**Timeline du problème** :

```
Time    Client A                    Client B
──────────────────────────────────────────────────────
t0      GET room:101:status
        → "available"
t1                                  GET room:101:status
                                    → "available"
t2      if status == "available":
        ✓ condition vraie
t3                                  if status == "available":
                                    ✓ condition vraie
t4      SET room:101:status "booked"
        SET room:101:guest "alice"
t5                                  SET room:101:status "booked"
                                    SET room:101:guest "bob"

RÉSULTAT : Double réservation ! Alice est écrasée par Bob.
```

### La solution : Optimistic Locking avec WATCH

**WATCH** implémente un mécanisme de **verrouillage optimiste** (optimistic locking) :

1. **"Surveiller"** une ou plusieurs clés
2. **Lire** leurs valeurs et effectuer la logique métier
3. **Tenter** une transaction
4. Si les clés surveillées ont changé → **La transaction échoue automatiquement**
5. **Réessayer** si nécessaire

Cette approche est dite "optimiste" car elle suppose que les **conflits sont rares** et qu'il est plus efficace de détecter et réessayer plutôt que de verrouiller préventivement.

## Fonctionnement de WATCH

### Cycle de vie complet

```redis
127.0.0.1:6379> WATCH mykey
OK

127.0.0.1:6379> val = GET mykey
"10"

# ... logique métier (calcul, vérifications) ...

127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET mykey 20
QUEUED
127.0.0.1:6379> INCR counter
QUEUED
127.0.0.1:6379> EXEC
1) OK
2) (integer) 5

# ✅ Succès : mykey n'a pas changé depuis WATCH
```

### Que se passe-t-il en cas de conflit ?

```redis
# Client A
127.0.0.1:6379> WATCH mykey
OK
127.0.0.1:6379> GET mykey
"10"

# Pendant ce temps, Client B modifie mykey
# Client B: SET mykey 15

# Client A continue
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET mykey 20
QUEUED
127.0.0.1:6379> EXEC
(nil)  # ❌ Transaction annulée !

# La transaction retourne nil au lieu d'un tableau de résultats
```

### Mécanisme interne : CAS (Compare-And-Set)

WATCH implémente un mécanisme **CAS (Compare-And-Set)** :

```
État au moment du WATCH:
┌─────────────────────────────┐
│ mykey → version: 42         │
└─────────────────────────────┘

Client surveille:
┌─────────────────────────────┐
│ watched_keys = {            │
│   "mykey": version 42       │
│ }                           │
└─────────────────────────────┘

Avant EXEC, Redis vérifie:
Si version_actuelle(mykey) == 42:
    → Exécuter la transaction
Sinon:
    → Annuler (retourner nil)
```

## Implémentation complète avec retry logic

### Pattern de base en Python

```python
import redis
import time
from typing import Optional, Dict, Any

def watch_with_retry(
    r: redis.Redis,
    watch_key: str,
    operation_func,
    max_retries: int = 10,
    retry_delay: float = 0.001
) -> Optional[Dict[str, Any]]:
    """
    Pattern générique pour WATCH avec retry automatique

    Args:
        r: Instance Redis
        watch_key: Clé à surveiller
        operation_func: Fonction à exécuter (reçoit la valeur actuelle)
        max_retries: Nombre maximum de tentatives
        retry_delay: Délai entre les tentatives (secondes)

    Returns:
        Résultat de l'opération ou None si échec après max_retries
    """
    for attempt in range(max_retries):
        try:
            with r.pipeline() as pipe:
                # 1. Surveiller la clé
                pipe.watch(watch_key)

                # 2. Lire la valeur actuelle
                current_value = pipe.get(watch_key)

                # 3. Exécuter la logique métier
                result = operation_func(current_value)

                # Si la logique métier refuse l'opération
                if result.get("abort"):
                    pipe.unwatch()  # Arrêter la surveillance
                    return result

                # 4. Démarrer la transaction
                pipe.multi()

                # 5. Ajouter les commandes de modification
                for cmd in result.get("commands", []):
                    getattr(pipe, cmd["method"])(*cmd["args"])

                # 6. Exécuter
                pipe.execute()

                # ✅ Succès
                return {"success": True, "attempts": attempt + 1}

        except redis.WatchError:
            # ❌ Conflit détecté, réessayer
            if attempt < max_retries - 1:
                time.sleep(retry_delay * (2 ** attempt))  # Backoff exponentiel
                continue
            else:
                return {
                    "success": False,
                    "error": "Max retries exceeded",
                    "attempts": max_retries
                }

    return {"success": False, "error": "Unexpected exit"}
```

### Exemple 1 : Réservation de chambre d'hôtel (sécurisée)

```python
def book_room_safe(r: redis.Redis, room_id: str, user_id: str) -> Dict[str, Any]:
    """
    Réservation sécurisée avec WATCH
    Évite les double-réservations même sous forte concurrence
    """
    room_key = f"room:{room_id}:status"
    guest_key = f"room:{room_id}:guest"

    def operation(current_status):
        # Décoder la valeur (bytes → string)
        status = current_status.decode('utf-8') if current_status else None

        # Vérification métier
        if status != "available":
            return {
                "abort": True,
                "error": f"Room not available (status: {status})"
            }

        # Préparer les commandes à exécuter
        return {
            "abort": False,
            "commands": [
                {"method": "set", "args": (room_key, "booked")},
                {"method": "set", "args": (guest_key, user_id)},
                {"method": "lpush", "args": (
                    f"bookings:log",
                    f"{room_id}:{user_id}:{int(time.time())}"
                )}
            ]
        }

    result = watch_with_retry(r, room_key, operation, max_retries=5)

    if result.get("success"):
        return {
            "success": True,
            "room_id": room_id,
            "user_id": user_id,
            "attempts": result.get("attempts", 1)
        }
    else:
        return result

# Test sous concurrence
import threading

def concurrent_booking_test():
    """
    Test avec 10 clients tentant de réserver simultanément
    """
    r = redis.Redis(decode_responses=False)
    r.set("room:101:status", "available")

    results = []

    def try_book(user_id):
        result = book_room_safe(r, "101", f"user_{user_id}")
        results.append(result)

    # Lancer 10 threads simultanés
    threads = []
    for i in range(10):
        t = threading.Thread(target=try_book, args=(i,))
        threads.append(t)
        t.start()

    # Attendre la fin
    for t in threads:
        t.join()

    # Analyser les résultats
    successes = [r for r in results if r.get("success")]
    failures = [r for r in results if not r.get("success")]

    print(f"Succès: {len(successes)} (attendu: 1)")
    print(f"Échecs: {len(failures)} (attendu: 9)")
    print(f"Gagnant: {successes[0]['user_id'] if successes else 'None'}")

    # Vérification finale
    final_guest = r.get("room:101:guest")
    print(f"Guest final: {final_guest}")

    return len(successes) == 1  # ✅ Test réussi si exactement 1 succès

# concurrent_booking_test()
```

### Exemple 2 : Décrémentation avec limite (e-commerce)

```python
def purchase_with_stock_check(
    r: redis.Redis,
    product_id: str,
    quantity: int,
    user_id: str
) -> Dict[str, Any]:
    """
    Achat avec vérification atomique du stock
    Évite les surventes
    """
    stock_key = f"product:{product_id}:stock"

    max_retries = 3
    for attempt in range(max_retries):
        try:
            with r.pipeline() as pipe:
                # Surveiller le stock
                pipe.watch(stock_key)

                # Lire le stock actuel
                current_stock = pipe.get(stock_key)
                stock = int(current_stock) if current_stock else 0

                # Vérification métier : stock suffisant ?
                if stock < quantity:
                    pipe.unwatch()
                    return {
                        "success": False,
                        "error": "Insufficient stock",
                        "available": stock,
                        "requested": quantity
                    }

                # Commencer la transaction
                pipe.multi()

                # Décrémenter le stock
                pipe.decrby(stock_key, quantity)

                # Créer la commande
                order_id = int(time.time() * 1000)  # Simple ID basé sur timestamp
                pipe.hset(f"order:{order_id}", mapping={
                    "product_id": product_id,
                    "user_id": user_id,
                    "quantity": quantity,
                    "status": "pending",
                    "created_at": int(time.time())
                })

                # Ajouter à l'index des commandes de l'utilisateur
                pipe.zadd(f"user:{user_id}:orders", {order_id: int(time.time())})

                # Statistiques
                pipe.hincrby("stats:products", f"{product_id}:sold", quantity)

                # Exécuter
                results = pipe.execute()

                return {
                    "success": True,
                    "order_id": order_id,
                    "remaining_stock": results[0],  # Résultat du DECRBY
                    "attempts": attempt + 1
                }

        except redis.WatchError:
            # Conflit : un autre client a modifié le stock
            if attempt < max_retries - 1:
                time.sleep(0.001 * (2 ** attempt))  # Backoff exponentiel
                continue
            else:
                return {
                    "success": False,
                    "error": "Transaction failed after retries",
                    "attempts": max_retries
                }

    return {"success": False, "error": "Unexpected error"}

# Test de concurrence
def stress_test_purchases():
    """
    100 clients tentent d'acheter un produit avec stock limité
    """
    r = redis.Redis(decode_responses=True)
    product_id = "laptop_pro_15"
    initial_stock = 10

    # Initialiser le stock
    r.set(f"product:{product_id}:stock", initial_stock)

    results = []
    lock = threading.Lock()

    def attempt_purchase(user_id):
        result = purchase_with_stock_check(r, product_id, 1, f"user_{user_id}")
        with lock:
            results.append(result)

    # 100 clients, 10 produits disponibles
    threads = []
    for i in range(100):
        t = threading.Thread(target=attempt_purchase, args=(i,))
        threads.append(t)
        t.start()

    for t in threads:
        t.join()

    # Analyse
    successes = [r for r in results if r.get("success")]
    failures = [r for r in results if not r.get("success")]

    print(f"\n=== Résultats du stress test ===")
    print(f"Achats réussis: {len(successes)}/{initial_stock}")
    print(f"Achats refusés: {len(failures)}")

    final_stock = int(r.get(f"product:{product_id}:stock"))
    print(f"Stock final: {final_stock} (attendu: 0)")

    # Distribution des tentatives
    attempts_distribution = {}
    for result in successes:
        attempts = result.get("attempts", 1)
        attempts_distribution[attempts] = attempts_distribution.get(attempts, 0) + 1

    print(f"\nDistribution des tentatives pour les succès:")
    for attempts, count in sorted(attempts_distribution.items()):
        print(f"  {attempts} tentative(s): {count} transactions")

    return len(successes) == initial_stock and final_stock == 0

# stress_test_purchases()
```

## Surveillance de plusieurs clés

WATCH peut surveiller plusieurs clés simultanément :

```python
def transfer_between_accounts(
    r: redis.Redis,
    from_account: str,
    to_account: str,
    amount: float
) -> Dict[str, Any]:
    """
    Transfert d'argent entre deux comptes avec vérification atomique
    Surveille DEUX clés : from_account ET to_account
    """
    from_key = f"account:{from_account}:balance"
    to_key = f"account:{to_account}:balance"

    max_retries = 5
    for attempt in range(max_retries):
        try:
            with r.pipeline() as pipe:
                # Surveiller les DEUX comptes
                pipe.watch(from_key, to_key)

                # Lire les soldes actuels
                from_balance = float(pipe.get(from_key) or 0)
                to_balance = float(pipe.get(to_key) or 0)

                # Vérifications métier
                if from_balance < amount:
                    pipe.unwatch()
                    return {
                        "success": False,
                        "error": "Insufficient funds",
                        "balance": from_balance
                    }

                # Vérifier que le compte destinataire existe
                if not pipe.exists(to_key):
                    pipe.unwatch()
                    return {
                        "success": False,
                        "error": "Destination account does not exist"
                    }

                # Transaction
                pipe.multi()

                # Débit
                new_from_balance = from_balance - amount
                pipe.set(from_key, new_from_balance)

                # Crédit
                new_to_balance = to_balance + amount
                pipe.set(to_key, new_to_balance)

                # Log de la transaction
                tx_id = f"tx:{int(time.time()*1000)}"
                pipe.hset(tx_id, mapping={
                    "from": from_account,
                    "to": to_account,
                    "amount": amount,
                    "timestamp": int(time.time()),
                    "status": "completed"
                })
                pipe.expire(tx_id, 86400)  # Expire après 24h

                # Exécuter
                results = pipe.execute()

                return {
                    "success": True,
                    "from_balance": new_from_balance,
                    "to_balance": new_to_balance,
                    "transaction_id": tx_id,
                    "attempts": attempt + 1
                }

        except redis.WatchError:
            if attempt < max_retries - 1:
                time.sleep(0.002 * (2 ** attempt))
                continue
            else:
                return {
                    "success": False,
                    "error": "Max retries exceeded",
                    "attempts": max_retries
                }

    return {"success": False, "error": "Unexpected error"}

# Test : Transferts concurrents depuis le même compte
def concurrent_transfers_test():
    r = redis.Redis(decode_responses=True)

    # Initialisation
    r.set("account:alice:balance", 1000)
    r.set("account:bob:balance", 0)
    r.set("account:charlie:balance", 0)

    results = []
    lock = threading.Lock()

    def transfer(to_account, amount):
        result = transfer_between_accounts(r, "alice", to_account, amount)
        with lock:
            results.append(result)

    # Alice tente 10 transferts simultanés
    threads = []
    for i in range(5):
        t1 = threading.Thread(target=transfer, args=("bob", 100))
        t2 = threading.Thread(target=transfer, args=("charlie", 100))
        threads.extend([t1, t2])
        t1.start()
        t2.start()

    for t in threads:
        t.join()

    # Vérification finale
    alice_final = float(r.get("account:alice:balance"))
    bob_final = float(r.get("account:bob:balance"))
    charlie_final = float(r.get("account:charlie:balance"))
    total = alice_final + bob_final + charlie_final

    successes = len([r for r in results if r.get("success")])

    print(f"\n=== Test de transferts concurrents ===")
    print(f"Alice: {alice_final}€")
    print(f"Bob: {bob_final}€")
    print(f"Charlie: {charlie_final}€")
    print(f"Total: {total}€ (doit être 1000€)")
    print(f"Transferts réussis: {successes}")

    return total == 1000  # ✅ Conservation de la masse monétaire

# concurrent_transfers_test()
```

## Pattern avancé : Retry avec backoff exponentiel

```python
import time
import random
from typing import Callable, Any, Optional

class OptimisticLockManager:
    """
    Gestionnaire réutilisable pour les opérations avec WATCH
    Implémente un backoff exponentiel avec jitter
    """

    def __init__(
        self,
        redis_client: redis.Redis,
        max_retries: int = 10,
        base_delay: float = 0.001,
        max_delay: float = 1.0,
        jitter: bool = True
    ):
        self.redis = redis_client
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.jitter = jitter

    def execute(
        self,
        watch_keys: list,
        read_func: Callable,
        validate_func: Callable,
        write_func: Callable
    ) -> Dict[str, Any]:
        """
        Exécute une opération optimistic locking avec retry intelligent

        Args:
            watch_keys: Clés à surveiller
            read_func: Fonction pour lire les données (reçoit pipeline)
            validate_func: Fonction de validation (reçoit les données lues)
            write_func: Fonction d'écriture (reçoit pipeline et données)

        Returns:
            Résultat de l'opération
        """
        stats = {
            "attempts": 0,
            "watch_errors": 0,
            "validation_errors": 0,
            "total_wait_time": 0
        }

        for attempt in range(self.max_retries):
            stats["attempts"] += 1

            try:
                with self.redis.pipeline() as pipe:
                    # 1. Surveiller les clés
                    if isinstance(watch_keys, list):
                        pipe.watch(*watch_keys)
                    else:
                        pipe.watch(watch_keys)

                    # 2. Lire les données
                    data = read_func(pipe)

                    # 3. Valider
                    validation_result = validate_func(data)
                    if not validation_result.get("valid"):
                        pipe.unwatch()
                        stats["validation_errors"] += 1
                        return {
                            "success": False,
                            "error": validation_result.get("error"),
                            "stats": stats
                        }

                    # 4. Commencer la transaction
                    pipe.multi()

                    # 5. Écrire
                    write_result = write_func(pipe, data, validation_result)

                    # 6. Exécuter
                    results = pipe.execute()

                    return {
                        "success": True,
                        "results": results,
                        "write_result": write_result,
                        "stats": stats
                    }

            except redis.WatchError:
                stats["watch_errors"] += 1

                if attempt < self.max_retries - 1:
                    # Calculer le délai avec backoff exponentiel
                    delay = min(
                        self.base_delay * (2 ** attempt),
                        self.max_delay
                    )

                    # Ajouter du jitter pour éviter la synchronisation
                    if self.jitter:
                        delay = delay * (0.5 + random.random())

                    stats["total_wait_time"] += delay
                    time.sleep(delay)
                else:
                    return {
                        "success": False,
                        "error": "Max retries exceeded",
                        "stats": stats
                    }

        return {"success": False, "error": "Unexpected exit", "stats": stats}

# Exemple d'utilisation
def increment_with_limit_using_manager(product_id: str, increment: int, max_value: int):
    """
    Incrémenter un compteur avec une limite maximale
    """
    manager = OptimisticLockManager(
        redis.Redis(decode_responses=True),
        max_retries=5,
        base_delay=0.001,
        jitter=True
    )

    key = f"product:{product_id}:views"

    def read_data(pipe):
        value = pipe.get(key)
        return {"current_value": int(value) if value else 0}

    def validate(data):
        current = data["current_value"]
        if current + increment > max_value:
            return {
                "valid": False,
                "error": f"Limit exceeded: {current} + {increment} > {max_value}"
            }
        return {"valid": True}

    def write(pipe, data, validation):
        new_value = data["current_value"] + increment
        pipe.set(key, new_value)
        return {"new_value": new_value}

    result = manager.execute(key, read_data, validate, write)

    if result.get("success"):
        print(f"✅ Succès après {result['stats']['attempts']} tentative(s)")
        print(f"   Nouvelle valeur: {result['write_result']['new_value']}")
        if result['stats']['watch_errors'] > 0:
            print(f"   Conflits détectés: {result['stats']['watch_errors']}")
            print(f"   Temps d'attente total: {result['stats']['total_wait_time']:.3f}s")
    else:
        print(f"❌ Échec: {result.get('error')}")
        print(f"   Tentatives: {result['stats']['attempts']}")

    return result
```

## Gestion des deadlocks et timeout

### Problème : WATCH sans EXEC

```python
# ❌ DANGER : WATCH sans EXEC/UNWATCH
def dangerous_watch_pattern():
    pipe = r.pipeline()
    pipe.watch("key1")

    value = pipe.get("key1")

    # Oubli d'appeler EXEC ou UNWATCH
    # → La clé reste surveillée jusqu'à la fermeture de la connexion !
    # → Peut bloquer d'autres opérations sur cette connexion

    return value  # ⚠️ WATCH jamais terminé !

# ✅ CORRECT : Toujours nettoyer avec UNWATCH
def safe_watch_pattern():
    pipe = r.pipeline()
    try:
        pipe.watch("key1")
        value = pipe.get("key1")

        # Logique métier...

        pipe.unwatch()  # ✅ Nettoyage explicite
        return value
    except Exception as e:
        pipe.unwatch()  # ✅ Nettoyage même en cas d'erreur
        raise
```

### Pattern avec timeout

```python
import signal
from contextlib import contextmanager

class WatchTimeout(Exception):
    pass

@contextmanager
def watch_with_timeout(pipe: redis.client.Pipeline, keys, timeout_seconds: float = 5.0):
    """
    Context manager pour WATCH avec timeout automatique
    """
    def timeout_handler(signum, frame):
        raise WatchTimeout("WATCH operation timed out")

    # Configurer le timeout (Linux/Unix uniquement)
    old_handler = signal.signal(signal.SIGALRM, timeout_handler)
    signal.setitimer(signal.ITIMER_REAL, timeout_seconds)

    try:
        if isinstance(keys, list):
            pipe.watch(*keys)
        else:
            pipe.watch(keys)

        yield pipe

    except WatchTimeout:
        pipe.unwatch()
        raise
    finally:
        # Restaurer le signal handler
        signal.setitimer(signal.ITIMER_REAL, 0)
        signal.signal(signal.SIGALRM, old_handler)

# Utilisation
def operation_with_timeout(key: str, timeout: float = 2.0):
    pipe = r.pipeline()

    try:
        with watch_with_timeout(pipe, key, timeout):
            value = pipe.get(key)

            # Logique métier (potentiellement longue)
            time.sleep(1.5)  # Simulation

            pipe.multi()
            pipe.set(key, "new_value")
            pipe.execute()

            return {"success": True}

    except WatchTimeout:
        return {"success": False, "error": "Operation timed out"}
```

## Comparaison des stratégies de concurrence

### WATCH vs Lua Script vs Distributed Lock

```python
# Stratégie 1 : WATCH (Optimistic Locking)
def strategy_watch(r: redis.Redis, key: str, amount: int):
    """
    ✅ Avantages:
    - Pas de verrouillage préventif
    - Excellentes performances si peu de conflits
    - Retry automatique simple

    ❌ Inconvénients:
    - Peut nécessiter plusieurs tentatives
    - Performance dégradée sous forte contention
    - Complexité du code (retry logic)
    """
    max_retries = 5
    for _ in range(max_retries):
        try:
            with r.pipeline() as pipe:
                pipe.watch(key)
                current = int(pipe.get(key) or 0)

                if current >= amount:
                    pipe.multi()
                    pipe.decrby(key, amount)
                    pipe.execute()
                    return {"success": True, "method": "watch"}
                else:
                    pipe.unwatch()
                    return {"success": False, "error": "Insufficient"}
        except redis.WatchError:
            continue

    return {"success": False, "error": "Max retries"}

# Stratégie 2 : Lua Script (Atomicité garantie)
def strategy_lua(r: redis.Redis, key: str, amount: int):
    """
    ✅ Avantages:
    - Une seule tentative nécessaire
    - Atomicité garantie
    - Performance constante
    - Logique complexe possible

    ❌ Inconvénients:
    - Bloque Redis pendant l'exécution
    - Moins flexible (modifier = recharger le script)
    """
    script = """
    local current = tonumber(redis.call('GET', KEYS[1])) or 0
    local amount = tonumber(ARGV[1])

    if current >= amount then
        redis.call('DECRBY', KEYS[1], amount)
        return {1, current - amount}
    else
        return {0, current}
    end
    """

    result = r.eval(script, 1, key, amount)
    return {
        "success": result[0] == 1,
        "method": "lua",
        "remaining": result[1]
    }

# Stratégie 3 : Distributed Lock (Pessimistic Locking)
def strategy_lock(r: redis.Redis, key: str, amount: int):
    """
    ✅ Avantages:
    - Garantie de séquentialité
    - Pas de retry nécessaire
    - Convient aux opérations très longues

    ❌ Inconvénients:
    - Overhead du lock
    - Risque de deadlock si mal géré
    - Performance réduite (sérialisation forcée)
    """
    lock_key = f"{key}:lock"
    lock_acquired = r.set(lock_key, "1", nx=True, ex=5)

    if not lock_acquired:
        return {"success": False, "error": "Lock not acquired", "method": "lock"}

    try:
        current = int(r.get(key) or 0)

        if current >= amount:
            r.decrby(key, amount)
            return {"success": True, "method": "lock"}
        else:
            return {"success": False, "error": "Insufficient", "method": "lock"}
    finally:
        r.delete(lock_key)

# Benchmark comparatif
def benchmark_concurrency_strategies():
    """
    Compare les 3 stratégies sous charge
    """
    r = redis.Redis(decode_responses=True)

    strategies = {
        "watch": strategy_watch,
        "lua": strategy_lua,
        "lock": strategy_lock
    }

    for name, strategy_func in strategies.items():
        # Initialisation
        r.set("test_balance", 1000)

        results = []
        start = time.time()

        # 100 threads tentant 10 décrements chacun
        def worker():
            for _ in range(10):
                result = strategy_func(r, "test_balance", 1)
                results.append(result)

        threads = [threading.Thread(target=worker) for _ in range(100)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()

        duration = time.time() - start
        final_balance = int(r.get("test_balance"))
        successes = len([r for r in results if r.get("success")])

        print(f"\n=== Stratégie: {name.upper()} ===")
        print(f"Durée: {duration:.3f}s")
        print(f"Succès: {successes}/1000")
        print(f"Balance finale: {final_balance} (attendu: 0)")
        print(f"Cohérence: {'✅' if final_balance == 0 else '❌'}")

# benchmark_concurrency_strategies()
```

### Matrice de décision

| Critère | WATCH | Lua Script | Distributed Lock |
|---------|-------|------------|------------------|
| **Performance (faible contention)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Performance (forte contention)** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Simplicité du code** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Logique complexe** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Garantie d'atomicité** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Risque de contention** | Moyen | Faible | Élevé |
| **Overhead réseau** | Moyen | Faible | Élevé |

## Debugging et monitoring

### Logger les conflits WATCH

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class WatchConflictTracker:
    """
    Tracker pour analyser les patterns de conflits
    """
    def __init__(self):
        self.total_attempts = 0
        self.total_conflicts = 0
        self.conflicts_by_key = {}
        self.max_retries_seen = 0

    def record_attempt(self, key: str, attempt: int, success: bool):
        self.total_attempts += 1

        if not success:
            self.total_conflicts += 1
            self.conflicts_by_key[key] = self.conflicts_by_key.get(key, 0) + 1

        self.max_retries_seen = max(self.max_retries_seen, attempt)

    def get_stats(self):
        conflict_rate = (self.total_conflicts / self.total_attempts * 100) if self.total_attempts > 0 else 0

        return {
            "total_attempts": self.total_attempts,
            "total_conflicts": self.total_conflicts,
            "conflict_rate": f"{conflict_rate:.2f}%",
            "max_retries_seen": self.max_retries_seen,
            "hottest_keys": sorted(
                self.conflicts_by_key.items(),
                key=lambda x: x[1],
                reverse=True
            )[:5]
        }

# Usage global
tracker = WatchConflictTracker()

def monitored_watch_operation(r: redis.Redis, key: str, operation_func):
    """
    Wrapper qui log automatiquement les conflits
    """
    max_retries = 10

    for attempt in range(max_retries):
        try:
            with r.pipeline() as pipe:
                pipe.watch(key)

                result = operation_func(pipe)

                pipe.multi()
                # ... commandes ...
                pipe.execute()

                # Enregistrer le succès
                tracker.record_attempt(key, attempt + 1, True)
                logger.info(f"✅ {key} succeeded on attempt {attempt + 1}")

                return {"success": True, "attempts": attempt + 1}

        except redis.WatchError:
            # Enregistrer le conflit
            tracker.record_attempt(key, attempt + 1, False)
            logger.warning(f"⚠️ {key} conflict on attempt {attempt + 1}")

            if attempt < max_retries - 1:
                time.sleep(0.001 * (2 ** attempt))
                continue
            else:
                logger.error(f"❌ {key} failed after {max_retries} attempts")
                return {"success": False, "attempts": max_retries}

    return {"success": False}

# Afficher les statistiques périodiquement
def print_conflict_stats():
    stats = tracker.get_stats()
    print("\n=== Statistiques de conflits WATCH ===")
    print(f"Tentatives totales: {stats['total_attempts']}")
    print(f"Conflits: {stats['total_conflicts']}")
    print(f"Taux de conflit: {stats['conflict_rate']}")
    print(f"Max retries observés: {stats['max_retries_seen']}")
    print("\nClés les plus conflictuelles:")
    for key, conflicts in stats['hottest_keys']:
        print(f"  {key}: {conflicts} conflits")
```

## Cas d'usage réels et patterns

### Pattern 1 : Rate Limiting avec WATCH

```python
def rate_limit_with_watch(
    r: redis.Redis,
    user_id: str,
    limit: int,
    window_seconds: int
) -> Dict[str, Any]:
    """
    Rate limiting avec fenêtre glissante utilisant WATCH
    Permet de limiter les requêtes par utilisateur
    """
    key = f"ratelimit:{user_id}"
    now = time.time()

    max_retries = 3
    for attempt in range(max_retries):
        try:
            with r.pipeline() as pipe:
                pipe.watch(key)

                # Nettoyer les anciennes entrées
                cutoff = now - window_seconds
                pipe.zremrangebyscore(key, 0, cutoff)

                # Compter les requêtes dans la fenêtre
                count = pipe.zcount(key, cutoff, now)

                if count >= limit:
                    pipe.unwatch()
                    return {
                        "allowed": False,
                        "limit": limit,
                        "remaining": 0,
                        "reset_at": cutoff + window_seconds
                    }

                # Ajouter la requête actuelle
                pipe.multi()
                pipe.zadd(key, {str(now): now})
                pipe.expire(key, window_seconds)
                pipe.execute()

                return {
                    "allowed": True,
                    "limit": limit,
                    "remaining": limit - count - 1,
                    "reset_at": now + window_seconds
                }

        except redis.WatchError:
            if attempt < max_retries - 1:
                continue
            else:
                return {"allowed": False, "error": "Too many conflicts"}

    return {"allowed": False, "error": "Unexpected error"}
```

### Pattern 2 : Leaderboard avec gestion de ties

```python
def update_leaderboard_with_watch(
    r: redis.Redis,
    player_id: str,
    new_score: int
) -> Dict[str, Any]:
    """
    Mise à jour d'un leaderboard avec gestion intelligente des ex-aequo
    """
    leaderboard_key = "leaderboard:global"
    player_score_key = f"player:{player_id}:score"

    max_retries = 5
    for attempt in range(max_retries):
        try:
            with r.pipeline() as pipe:
                # Surveiller le score du joueur ET le leaderboard
                pipe.watch(player_score_key, leaderboard_key)

                # Lire le score actuel
                old_score = pipe.get(player_score_key)
                old_score = int(old_score) if old_score else 0

                # Ne mettre à jour que si le nouveau score est meilleur
                if new_score <= old_score:
                    pipe.unwatch()
                    return {
                        "updated": False,
                        "reason": "Score not improved",
                        "old_score": old_score,
                        "new_score": new_score
                    }

                # Obtenir le rang actuel
                old_rank = pipe.zrevrank(leaderboard_key, player_id)

                # Transaction
                pipe.multi()

                # Mettre à jour le score
                pipe.set(player_score_key, new_score)

                # Mettre à jour le leaderboard
                # Score composite : (score * 1000000) + timestamp pour gérer les ties
                composite_score = new_score * 1000000 + int(time.time())
                pipe.zadd(leaderboard_key, {player_id: composite_score})

                # Limiter le leaderboard aux top 100
                pipe.zremrangebyrank(leaderboard_key, 0, -101)

                results = pipe.execute()

                # Calculer le nouveau rang
                new_rank = r.zrevrank(leaderboard_key, player_id)

                return {
                    "updated": True,
                    "old_score": old_score,
                    "new_score": new_score,
                    "old_rank": old_rank,
                    "new_rank": new_rank,
                    "rank_change": (old_rank - new_rank) if old_rank is not None else None,
                    "attempts": attempt + 1
                }

        except redis.WatchError:
            if attempt < max_retries - 1:
                time.sleep(0.001 * (2 ** attempt))
                continue
            else:
                return {
                    "updated": False,
                    "error": "Max retries exceeded",
                    "attempts": max_retries
                }

    return {"updated": False, "error": "Unexpected error"}
```

## Limitations et pièges à éviter

### ❌ Piège 1 : WATCH sur une clé inexistante

```python
# Comportement surprenant
r.delete("nonexistent")

pipe = r.pipeline()
pipe.watch("nonexistent")  # ✅ Fonctionne même si la clé n'existe pas

# Pendant ce temps, un autre client crée la clé
# r.set("nonexistent", "value")

pipe.multi()
pipe.set("another_key", "value")
result = pipe.execute()  # ❌ Retourne nil (transaction annulée)

# WATCH détecte la création de la clé comme une modification !
```

### ❌ Piège 2 : WATCH avec pipeline non-transactionnel

```python
# ❌ MAUVAIS : WATCH sans transaction
pipe = r.pipeline(transaction=False)
pipe.watch("key1")
pipe.get("key1")  # Cette commande s'exécute immédiatement !
# WATCH est déjà désactivé à ce stade

# ✅ BON : WATCH nécessite un pipeline transactionnel
pipe = r.pipeline(transaction=True)  # ou juste pipeline()
pipe.watch("key1")
# Commandes en file d'attente jusqu'à EXEC
```

### ❌ Piège 3 : Oublier UNWATCH après échec de validation

```python
# ❌ MAUVAIS : WATCH reste actif
def bad_pattern(r: redis.Redis):
    pipe = r.pipeline()
    pipe.watch("key1")

    value = pipe.get("key1")

    if not_valid(value):
        return {"error": "Invalid"}  # ⚠️ WATCH toujours actif !

    pipe.multi()
    # ...
    pipe.execute()

# ✅ BON : Toujours UNWATCH en cas d'abandon
def good_pattern(r: redis.Redis):
    pipe = r.pipeline()
    try:
        pipe.watch("key1")

        value = pipe.get("key1")

        if not_valid(value):
            pipe.unwatch()  # ✅ Nettoyage explicite
            return {"error": "Invalid"}

        pipe.multi()
        # ...
        pipe.execute()
    except Exception:
        pipe.unwatch()  # ✅ Nettoyage en cas d'erreur
        raise
```

## Résumé et guide de décision

### Quand utiliser WATCH ?

✅ **Utilisez WATCH quand** :
- Vous avez besoin de Read-Modify-Write atomique
- Les conflits sont **rares** (< 10% de tentatives)
- La logique de validation est **simple**
- Vous pouvez accepter des retries
- Vous voulez éviter les scripts Lua

❌ **N'utilisez PAS WATCH quand** :
- Les conflits sont **fréquents** (> 25% de tentatives)
- La logique est **complexe** (préférer Lua)
- Vous ne pouvez pas tolérer les retries
- Les opérations sont très longues

### Checklist d'implémentation

```
☐ Implémenter une retry logic avec backoff exponentiel
☐ Limiter le nombre maximum de tentatives (3-10)
☐ Toujours appeler UNWATCH en cas d'abandon
☐ Logger les conflits pour détecter les hot keys
☐ Tester sous charge pour valider la stratégie
☐ Surveiller le taux de conflits en production
☐ Avoir un plan B si les conflits explosent (Lua script)
```

### Comparaison finale

| Approche | Complexité | Performance | Atomicité | Retry | Cas d'usage |
|----------|-----------|-------------|-----------|-------|-------------|
| **WATCH** | Moyenne | Bonne si peu de conflits | ✅ | Nécessaire | Read-Modify-Write avec validation |
| **MULTI/EXEC** | Faible | Excellente | ✅ | Non | Séquences simples sans validation |
| **Lua Script** | Élevée | Excellente | ✅ | Non | Logique complexe, forte contention |
| **Functions** | Élevée | Excellente | ✅ | Non | Logique réutilisable (Redis 7+) |

## Conclusion

WATCH est un outil puissant pour implémenter de l'optimistic locking dans Redis. Il brille dans les scénarios où :
- Les conflits sont rares
- La validation métier est nécessaire avant modification
- Vous voulez éviter la complexité des scripts Lua

Cependant, il nécessite une implémentation soigneuse :
- Retry logic robuste avec backoff
- Nettoyage systématique avec UNWATCH
- Monitoring des conflits pour détecter les problèmes

Pour des cas d'usage plus complexes ou avec forte contention, considérez les scripts Lua (section 7.3) ou Redis Functions (section 7.4) qui offrent une atomicité garantie sans retry.

---

**📚 Points clés à retenir** :
- WATCH implémente un **optimistic locking** (CAS)
- La transaction échoue (retourne `nil`) si les clés surveillées changent
- Toujours implémenter une **retry logic** avec backoff exponentiel
- Appeler **UNWATCH** en cas d'abandon pour libérer les ressources
- Privilégier Lua/Functions si les **conflits dépassent 25%** des tentatives

**🔜 Prochaine section** : [7.3 Scripting Lua : Créer ses propres commandes atomiques](./03-scripting-lua-commandes-atomiques.md)

⏭️ [Scripting Lua : Créer ses propres commandes atomiques](/07-atomicite-programmabilite/03-scripting-lua-commandes-atomiques.md)
