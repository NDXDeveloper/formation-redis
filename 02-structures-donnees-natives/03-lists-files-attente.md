🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.3 Lists : Files d'attente simples (LPUSH/RPOP)

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Comment fonctionnent les Lists dans Redis (listes chaînées)
- ✅ La différence entre LPUSH/RPUSH et LPOP/RPOP
- ✅ Comment implémenter des files FIFO et LIFO
- ✅ Les opérations bloquantes pour les workers
- ✅ Les cas d'usage réels (queues, timelines, logs)

---

## 📘 Les Lists : Des listes doublement chaînées

### Qu'est-ce qu'une List dans Redis ?

Une **List** dans Redis est une **liste doublement chaînée** (doubly linked list) qui permet d'ajouter et retirer des éléments **aux deux extrémités** en **O(1)**.

```
┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
│  A  │◄──►│  B  │◄──►│  C  │◄──►│  D  │
└─────┘    └─────┘    └─────┘    └─────┘
  ▲                                  ▲
  │                                  │
LEFT                              RIGHT
(tête)                           (queue)
```

**Caractéristiques** :
- ✅ Insertion/suppression en **O(1)** aux extrémités
- ✅ Accès par index en **O(N)**
- ✅ Peut contenir des **doublons**
- ✅ **Ordre préservé** (insertion order)
- ✅ Maximum théorique : **2³² - 1 éléments** (~4 milliards)

### Pourquoi utiliser des Lists ?

Les Lists sont **idéales** pour :
- 🚀 **Files d'attente** (queues) : Jobs à traiter, messages, tâches
- 📜 **Timelines** : Feed d'activités, historique
- 📋 **Logs** : Stocker les N derniers événements
- 🔄 **Stacks** : Opérations LIFO (Last In, First Out)

---

## 🔧 Opérations de base : Push et Pop

### Terminologie : LEFT et RIGHT

```bash
# LEFT = début/tête de la liste (index 0)
# RIGHT = fin/queue de la liste (dernier élément)

        LEFT ◄───── Liste ─────► RIGHT
          0     1     2     3      4
        ┌───┬───┬───┬───┬───┐
        │ A │ B │ C │ D │ E │
        └───┴───┴───┴───┴───┘
```

### LPUSH : Ajouter à gauche (début)

```bash
# Créer une liste en ajoutant à gauche
127.0.0.1:6379> LPUSH mylist "C"
(integer) 1  # Nombre d'éléments dans la liste

127.0.0.1:6379> LPUSH mylist "B"
(integer) 2

127.0.0.1:6379> LPUSH mylist "A"
(integer) 3

# État de la liste : ["A", "B", "C"]
#                      ▲ tête (LEFT)

# LPUSH peut ajouter plusieurs éléments à la fois
127.0.0.1:6379> LPUSH numbers 3 2 1
(integer) 3
# Ordre d'insertion : 1 puis 2 puis 3
# État : ["1", "2", "3"]
```

### RPUSH : Ajouter à droite (fin)

```bash
# Ajouter à la fin de la liste
127.0.0.1:6379> RPUSH mylist "D"
(integer) 4  # ["A", "B", "C", "D"]

127.0.0.1:6379> RPUSH mylist "E"
(integer) 5  # ["A", "B", "C", "D", "E"]

# RPUSH multiple
127.0.0.1:6379> RPUSH queue "task1" "task2" "task3"
(integer) 3
# État : ["task1", "task2", "task3"]
```

### LPOP : Retirer à gauche (début)

```bash
# Créer une liste
127.0.0.1:6379> RPUSH fruits "apple" "banana" "cherry"
(integer) 3

# Retirer le premier élément
127.0.0.1:6379> LPOP fruits
"apple"

# État : ["banana", "cherry"]

127.0.0.1:6379> LPOP fruits
"banana"

# État : ["cherry"]

# Si la liste est vide
127.0.0.1:6379> LPOP fruits
"cherry"

127.0.0.1:6379> LPOP fruits
(nil)  # Plus rien à retirer

# LPOP avec COUNT (Redis 6.2+)
127.0.0.1:6379> RPUSH numbers 1 2 3 4 5
(integer) 5

127.0.0.1:6379> LPOP numbers 3
1) "1"
2) "2"
3) "3"
# État : ["4", "5"]
```

### RPOP : Retirer à droite (fin)

```bash
# Créer une liste
127.0.0.1:6379> LPUSH stack "first" "second" "third"
(integer) 3
# État : ["third", "second", "first"]

# Retirer le dernier élément
127.0.0.1:6379> RPOP stack
"first"

# État : ["third", "second"]

# RPOP avec COUNT
127.0.0.1:6379> RPOP stack 2
1) "second"
2) "third"
# Liste maintenant vide
```

---

## 🎨 Pattern #1 : Queue FIFO (First In, First Out)

### Implémentation d'une file d'attente

```bash
# Producer : Ajoute des jobs à la fin
127.0.0.1:6379> RPUSH jobs:queue "process-payment-1"
(integer) 1

127.0.0.1:6379> RPUSH jobs:queue "process-payment-2"
(integer) 2

127.0.0.1:6379> RPUSH jobs:queue "send-email-3"
(integer) 3

# État : ["process-payment-1", "process-payment-2", "send-email-3"]
#         ▲ prochain à traiter

# Consumer : Retire les jobs du début
127.0.0.1:6379> LPOP jobs:queue
"process-payment-1"  # Premier arrivé, premier traité

127.0.0.1:6379> LPOP jobs:queue
"process-payment-2"

# État : ["send-email-3"]
```

**Visualisation FIFO** :
```
RPUSH →  [A] [B] [C] [D]  ← LPOP
         ▲              ▲
     Nouveau         Premier
      job          à traiter
```

**Code application (pseudo-code)** :
```python
# Producer
def enqueue_job(job_data):
    redis.rpush("jobs:queue", json.dumps(job_data))

# Consumer
def process_jobs():
    while True:
        job = redis.lpop("jobs:queue")
        if job:
            process(json.loads(job))
        else:
            time.sleep(1)  # Attendre
```

---

## 📚 Pattern #2 : Stack LIFO (Last In, First Out)

### Implémentation d'une pile

```bash
# Ajouter à gauche (empiler)
127.0.0.1:6379> LPUSH history "/home"
(integer) 1

127.0.0.1:6379> LPUSH history "/home/user"
(integer) 2

127.0.0.1:6379> LPUSH history "/home/user/documents"
(integer) 3

# État : ["/home/user/documents", "/home/user", "/home"]
#         ▲ dernier ajouté

# Retirer à gauche (dépiler)
127.0.0.1:6379> LPOP history
"/home/user/documents"  # Dernier arrivé, premier traité

127.0.0.1:6379> LPOP history
"/home/user"

# État : ["/home"]
```

**Visualisation LIFO** :
```
LPUSH →  [D] [C] [B] [A]
LPOP  ←  ▲
     Dernier ajouté
   = Premier retiré
```

**Cas d'usage** : Historique de navigation, undo/redo, stack de logs.

---

## 🔍 Opérations de lecture

### LRANGE : Lire une plage d'éléments

```bash
# Créer une liste
127.0.0.1:6379> RPUSH colors "red" "green" "blue" "yellow" "purple"
(integer) 5

# Lire les 3 premiers éléments
127.0.0.1:6379> LRANGE colors 0 2
1) "red"
2) "green"
3) "blue"

# Lire TOUS les éléments
127.0.0.1:6379> LRANGE colors 0 -1
1) "red"
2) "green"
3) "blue"
4) "yellow"
5) "purple"

# Indices négatifs : -1 = dernier, -2 = avant-dernier
127.0.0.1:6379> LRANGE colors -2 -1
1) "yellow"
2) "purple"

# Lire du 2ème à l'avant-dernier
127.0.0.1:6379> LRANGE colors 1 -2
1) "green"
2) "blue"
3) "yellow"
```

⚠️ **Attention** : LRANGE est **O(N)** où N est la taille de la plage. N'utilisez pas `LRANGE mylist 0 -1` sur de très grandes listes en production !

### LLEN : Longueur de la liste

```bash
127.0.0.1:6379> RPUSH mylist "a" "b" "c"
(integer) 3

127.0.0.1:6379> LLEN mylist
(integer) 3

# Très rapide : O(1)
127.0.0.1:6379> LLEN huge:list
(integer) 1000000
```

### LINDEX : Accéder par index

```bash
127.0.0.1:6379> RPUSH items "first" "second" "third"
(integer) 3

# Accès par index (0-based)
127.0.0.1:6379> LINDEX items 0
"first"

127.0.0.1:6379> LINDEX items 1
"second"

# Index négatif
127.0.0.1:6379> LINDEX items -1
"third"  # Dernier élément

127.0.0.1:6379> LINDEX items -2
"second"  # Avant-dernier

# Index hors limite
127.0.0.1:6379> LINDEX items 100
(nil)
```

⚠️ **Complexité** : LINDEX est **O(N)** ! Pour accéder au milieu d'une grande liste, Redis doit parcourir tous les éléments.

---

## ✏️ Opérations de modification

### LSET : Modifier un élément par index

```bash
# Créer une liste
127.0.0.1:6379> RPUSH tasks "TODO: task1" "TODO: task2" "TODO: task3"
(integer) 3

# Modifier le 2ème élément
127.0.0.1:6379> LSET tasks 1 "DONE: task2"
OK

127.0.0.1:6379> LRANGE tasks 0 -1
1) "TODO: task1"
2) "DONE: task2"
3) "TODO: task3"

# Erreur si index invalide
127.0.0.1:6379> LSET tasks 100 "value"
(error) ERR index out of range
```

### LINSERT : Insérer avant ou après

```bash
# Créer une liste
127.0.0.1:6379> RPUSH fruits "apple" "cherry"
(integer) 2

# Insérer AVANT "cherry"
127.0.0.1:6379> LINSERT fruits BEFORE "cherry" "banana"
(integer) 3  # Nouvelle longueur

127.0.0.1:6379> LRANGE fruits 0 -1
1) "apple"
2) "banana"
3) "cherry"

# Insérer APRÈS "apple"
127.0.0.1:6379> LINSERT fruits AFTER "apple" "apricot"
(integer) 4

127.0.0.1:6379> LRANGE fruits 0 -1
1) "apple"
2) "apricot"
3) "banana"
4) "cherry"

# Si la valeur pivot n'existe pas
127.0.0.1:6379> LINSERT fruits BEFORE "orange" "mango"
(integer) -1  # Échec
```

### LREM : Supprimer des éléments

```bash
# Créer une liste avec doublons
127.0.0.1:6379> RPUSH numbers 1 2 3 2 4 2 5
(integer) 7

# Supprimer les 2 premiers "2" (count > 0 = depuis le début)
127.0.0.1:6379> LREM numbers 2 "2"
(integer) 2  # Nombre d'éléments supprimés

127.0.0.1:6379> LRANGE numbers 0 -1
1) "1"
2) "3"
3) "4"
4) "2"
5) "5"

# Supprimer le dernier "2" (count < 0 = depuis la fin)
127.0.0.1:6379> LREM numbers -1 "2"
(integer) 1

127.0.0.1:6379> LRANGE numbers 0 -1
1) "1"
2) "3"
3) "4"
4) "5"

# Supprimer TOUS les "3" (count = 0)
127.0.0.1:6379> RPUSH test "a" "b" "a" "c" "a"
(integer) 5

127.0.0.1:6379> LREM test 0 "a"
(integer) 3  # 3 éléments supprimés

127.0.0.1:6379> LRANGE test 0 -1
1) "b"
2) "c"
```

**Syntaxe LREM** :
- `LREM key count value`
- count > 0 : Supprime les `count` premiers de gauche à droite
- count < 0 : Supprime les `count` premiers de droite à gauche
- count = 0 : Supprime TOUTES les occurrences

---

## ✂️ LTRIM : Garder seulement une plage

```bash
# Créer une liste
127.0.0.1:6379> RPUSH logs "log1" "log2" "log3" "log4" "log5" "log6"
(integer) 6

# Garder seulement les éléments de l'index 1 à 4
127.0.0.1:6379> LTRIM logs 1 4
OK

127.0.0.1:6379> LRANGE logs 0 -1
1) "log2"
2) "log3"
3) "log4"
4) "log5"

# Garder seulement les 3 derniers éléments
127.0.0.1:6379> RPUSH logs "log6" "log7" "log8"
(integer) 7

127.0.0.1:6379> LTRIM logs -3 -1
OK

127.0.0.1:6379> LRANGE logs 0 -1
1) "log6"
2) "log7"
3) "log8"
```

**Cas d'usage** : Limiter la taille d'une liste (logs circulaires).

```bash
# Pattern : Garder seulement les 100 derniers logs
RPUSH logs:recent "new log entry"
LTRIM logs:recent -100 -1
```

---

## ⏳ Opérations bloquantes : BLPOP et BRPOP

### Le problème du polling actif

```bash
# ❌ MAUVAIS : Polling actif (gaspille CPU)
while True:
    job = redis.lpop("jobs:queue")
    if job:
        process(job)
    else:
        time.sleep(0.1)  # Attendre et réessayer
```

### La solution : Opérations bloquantes

```bash
# BLPOP : LPOP bloquant
127.0.0.1:6379> BLPOP jobs:queue 10
# Bloque pendant 10 secondes max en attendant un élément
# Si un élément arrive, retourne immédiatement

# Dans un autre terminal, ajouter un job
127.0.0.1:6379> RPUSH jobs:queue "new-job"
(integer) 1

# Premier terminal reçoit immédiatement :
1) "jobs:queue"  # Nom de la clé
2) "new-job"     # Valeur

# Si timeout (10s) expiré sans élément
127.0.0.1:6379> BLPOP jobs:queue 10
(nil)
(10.02s)  # Temps écoulé
```

### BRPOP : RPOP bloquant

```bash
# Même principe mais retire à droite
127.0.0.1:6379> BRPOP jobs:queue 5
# Attend 5 secondes max
```

### BLPOP/BRPOP sur plusieurs clés

```bash
# Attendre sur plusieurs queues (priorité dans l'ordre)
127.0.0.1:6379> BLPOP high:priority normal:priority low:priority 30
# Vérifie d'abord high:priority, puis normal:priority, puis low:priority
# Dès qu'un élément est disponible, le retourne

# Exemple d'utilisation
127.0.0.1:6379> RPUSH high:priority "urgent-task"
(integer) 1

# BLPOP retourne immédiatement
1) "high:priority"  # De quelle queue provient l'élément
2) "urgent-task"
```

**Avantages** :
- ✅ Pas de CPU gaspillé en polling
- ✅ Réactivité immédiate
- ✅ Gestion de priorités native
- ✅ Timeout configurable

---

## 🔄 RPOPLPUSH et BRPOPLPUSH : Pattern atomique

### RPOPLPUSH : Déplacer atomiquement

```bash
# Créer deux listes
127.0.0.1:6379> RPUSH source "item1" "item2" "item3"
(integer) 3

# Déplacer le dernier de "source" au début de "destination"
127.0.0.1:6379> RPOPLPUSH source destination
"item3"

127.0.0.1:6379> LRANGE source 0 -1
1) "item1"
2) "item2"

127.0.0.1:6379> LRANGE destination 0 -1
1) "item3"

# Opération atomique : pas de race condition possible
```

### Cas d'usage : Pattern "Reliable Queue"

```bash
# Queue principale
127.0.0.1:6379> RPUSH jobs:pending "job1" "job2" "job3"
(integer) 3

# Worker prend un job de "pending" et le met dans "processing"
127.0.0.1:6379> RPOPLPUSH jobs:pending jobs:processing
"job3"

# Si le worker traite correctement, retirer de "processing"
127.0.0.1:6379> LREM jobs:processing 1 "job3"
(integer) 1

# Si le worker crash, le job reste dans "processing"
# Un autre process peut récupérer les jobs abandonnés
127.0.0.1:6379> LRANGE jobs:processing 0 -1
1) "job-abandoned"  # Job non traité suite à un crash
```

**Schéma du pattern** :
```
    jobs:pending                jobs:processing
  ┌──────────────┐           ┌──────────────┐
  │ job1         │           │              │
  │ job2         │ ──────►   │ job3 (actif) │
  │ job3         │ RPOPLPUSH │              │
  └──────────────┘           └──────────────┘
                                    │
                                    │ Traité avec succès
                                    ▼
                                  LREM
```

### BRPOPLPUSH : Version bloquante

```bash
# Attend qu'un job arrive dans "pending" et le déplace vers "processing"
127.0.0.1:6379> BRPOPLPUSH jobs:pending jobs:processing 30
# Bloque jusqu'à 30 secondes
```

**Code worker complet** :
```python
def worker():
    while True:
        # Prendre un job (bloque jusqu'à ce qu'un job arrive)
        job = redis.brpoplpush("jobs:pending", "jobs:processing", 30)

        if job:
            try:
                # Traiter le job
                process(job)

                # Job réussi : retirer de processing
                redis.lrem("jobs:processing", 1, job)
            except Exception as e:
                # Job échoué : remettre dans pending
                redis.rpush("jobs:failed", job)
                redis.lrem("jobs:processing", 1, job)
```

---

## 📊 Cas d'usage réels

### 1. Timeline d'activités (Twitter-like)

```bash
# Ajouter une nouvelle activité à la timeline d'un utilisateur
127.0.0.1:6379> LPUSH timeline:user:123 "Alice liked your post"
(integer) 1

127.0.0.1:6379> LPUSH timeline:user:123 "Bob followed you"
(integer) 2

127.0.0.1:6379> LPUSH timeline:user:123 "New comment on your photo"
(integer) 3

# Récupérer les 10 dernières activités
127.0.0.1:6379> LRANGE timeline:user:123 0 9
1) "New comment on your photo"  # Plus récent
2) "Bob followed you"
3) "Alice liked your post"

# Garder seulement les 1000 dernières activités
127.0.0.1:6379> LTRIM timeline:user:123 0 999
OK
```

### 2. Logs circulaires (derniers N événements)

```bash
# Logger un événement
127.0.0.1:6379> LPUSH logs:app "2024-12-09 14:30:00 - User logged in"
(integer) 1

127.0.0.1:6379> LPUSH logs:app "2024-12-09 14:31:15 - API call to /users"
(integer) 2

# Garder seulement les 100 derniers logs
127.0.0.1:6379> LTRIM logs:app 0 99
OK

# Récupérer les 20 derniers logs
127.0.0.1:6379> LRANGE logs:app 0 19
1) "2024-12-09 14:31:15 - API call to /users"
2) "2024-12-09 14:30:00 - User logged in"
```

### 3. File d'attente de jobs avec priorités

```bash
# Trois niveaux de priorité
127.0.0.1:6379> RPUSH queue:critical "urgent-bug-fix"
(integer) 1

127.0.0.1:6379> RPUSH queue:normal "feature-request"
(integer) 1

127.0.0.1:6379> RPUSH queue:low "refactoring"
(integer) 1

# Worker traite les queues par ordre de priorité
127.0.0.1:6379> BLPOP queue:critical queue:normal queue:low 10
1) "queue:critical"
2) "urgent-bug-fix"  # Priorité la plus haute

# Prochain appel
127.0.0.1:6379> BLPOP queue:critical queue:normal queue:low 10
1) "queue:normal"
2) "feature-request"
```

### 4. Chat : Messages récents

```bash
# Ajouter un message au salon
127.0.0.1:6379> RPUSH chat:room:general "Alice: Hello!"
(integer) 1

127.0.0.1:6379> RPUSH chat:room:general "Bob: Hi Alice"
(integer) 2

127.0.0.1:6379> RPUSH chat:room:general "Charlie: Good morning"
(integer) 3

# Récupérer les 50 derniers messages
127.0.0.1:6379> LRANGE chat:room:general -50 -1
1) "Alice: Hello!"
2) "Bob: Hi Alice"
3) "Charlie: Good morning"

# Limiter à 1000 messages max
127.0.0.1:6379> LTRIM chat:room:general -1000 -1
OK
```

### 5. Undo/Redo stack

```bash
# Stack d'actions utilisateur
127.0.0.1:6379> LPUSH undo:user:123 '{"action":"delete","item":"file.txt"}'
(integer) 1

127.0.0.1:6379> LPUSH undo:user:123 '{"action":"create","item":"folder"}'
(integer) 2

# Undo : Retirer la dernière action
127.0.0.1:6379> LPOP undo:user:123
"{\"action\":\"create\",\"item\":\"folder\"}"

# Mettre dans le stack redo
127.0.0.1:6379> LPUSH redo:user:123 '{"action":"create","item":"folder"}'
(integer) 1

# Redo : Rejouer l'action
127.0.0.1:6379> LPOP redo:user:123
"{\"action\":\"create\",\"item\":\"folder\"}"
```

---

## 🎭 Pattern avancé : Liste circulaire

```bash
# Créer une playlist
127.0.0.1:6379> RPUSH playlist "song1" "song2" "song3"
(integer) 3

# Jouer la chanson suivante ET la remettre à la fin
127.0.0.1:6379> RPOPLPUSH playlist playlist
"song3"  # On joue song3

# État : ["song3", "song1", "song2"]

# Continuer
127.0.0.1:6379> RPOPLPUSH playlist playlist
"song2"  # État : ["song2", "song3", "song1"]

127.0.0.1:6379> RPOPLPUSH playlist playlist
"song1"  # État : ["song1", "song2", "song3"]

# La liste revient à son état initial après 3 opérations (rotation)
```

---

## ⚡ Complexité et Performance

| Commande | Complexité | Notes |
|----------|------------|-------|
| `LPUSH/RPUSH` | O(1) | Pour chaque élément |
| `LPOP/RPOP` | O(1) | |
| `LLEN` | O(1) | Redis maintient un compteur |
| `LINDEX` | O(N) | N = index, éviter sur grandes listes |
| `LRANGE` | O(S+N) | S = start offset, N = éléments retournés |
| `LSET` | O(N) | N = index |
| `LINSERT` | O(N) | Doit trouver l'élément pivot |
| `LREM` | O(N+M) | N = longueur, M = éléments supprimés |
| `LTRIM` | O(N) | N = éléments supprimés |
| `RPOPLPUSH` | O(1) | |
| `BLPOP/BRPOP` | O(N) | N = nombre de clés |

---

## 🚨 Pièges courants à éviter

### 1. Utiliser LINDEX sur de grandes listes

```bash
# ❌ Inefficace : accès au milieu d'une grande liste
LINDEX huge:list 500000  # O(N) !

# ✅ Les Lists sont optimales aux extrémités
LPOP mylist  # O(1)
RPOP mylist  # O(1)
```

**Conseil** : Si vous avez besoin d'accès aléatoire rapide, utilisez un **Sorted Set** ou un **Hash**.

### 2. LRANGE sur toute une grande liste

```bash
# ❌ Dangereux en production
LRANGE huge:list 0 -1  # Peut retourner des millions d'éléments !

# ✅ Paginer les résultats
LRANGE mylist 0 99     # Page 1 (éléments 0-99)
LRANGE mylist 100 199  # Page 2 (éléments 100-199)
```

### 3. Oublier LTRIM sur des listes à croissance infinie

```bash
# ❌ MAUVAIS : Liste qui grandit indéfiniment
LPUSH logs:unlimited "new log"
# Finira par consommer toute la mémoire !

# ✅ BON : Limiter la taille
LPUSH logs:recent "new log"
LTRIM logs:recent 0 999  # Garder max 1000 logs
```

### 4. Polling actif au lieu de BLPOP

```bash
# ❌ Gaspille du CPU
while True:
    job = redis.lpop("queue")
    if not job:
        time.sleep(0.1)

# ✅ Utiliser BLPOP
while True:
    result = redis.blpop("queue", timeout=10)
    if result:
        _, job = result
        process(job)
```

### 5. Ne pas gérer les échecs de jobs

```bash
# ❌ Job perdu si le worker crash
job = redis.lpop("queue")
process(job)  # Si crash ici, job perdu !

# ✅ Pattern Reliable Queue avec RPOPLPUSH
job = redis.rpoplpush("pending", "processing")
try:
    process(job)
    redis.lrem("processing", 1, job)
except:
    redis.rpush("failed", job)
    redis.lrem("processing", 1, job)
```

---

## 🎯 Quand NE PAS utiliser les Lists

### ❌ Pour de l'accès aléatoire fréquent

```bash
# Si vous avez besoin de ça souvent :
LINDEX mylist 42
LINDEX mylist 1337
LINDEX mylist 9999

# → Utilisez plutôt un Hash avec des clés numériques
HSET mydata 42 "value42"
HSET mydata 1337 "value1337"
HGET mydata 42  # O(1) !
```

### ❌ Pour la recherche d'éléments

```bash
# ❌ Les Lists n'ont pas de commande "CONTAINS"
# Il faut faire LRANGE et chercher côté client → O(N)

# → Utilisez plutôt un Set
SADD myset "value"
SISMEMBER myset "value"  # O(1)
```

### ❌ Pour des données ordonnées par score

```bash
# ❌ Les Lists gardent l'ordre d'insertion, pas un ordre par score

# → Utilisez un Sorted Set
ZADD leaderboard 1500 "player1"
ZADD leaderboard 2300 "player2"
ZRANGE leaderboard 0 9  # Top 10 par score
```

---

## 📋 Checklist : Choix de structure

Utilisez une **List** si :
- ✅ Vous ajoutez/retirez principalement aux extrémités
- ✅ Vous avez besoin d'une queue FIFO ou stack LIFO
- ✅ L'ordre d'insertion doit être préservé
- ✅ Vous stockez une timeline ou des logs

N'utilisez **PAS** une List si :
- ❌ Vous avez besoin d'accès aléatoire fréquent (milieu de liste)
- ❌ Vous devez chercher si un élément existe
- ❌ Vous avez besoin de trier par score ou valeur
- ❌ Vous voulez éviter les doublons

---

## 📊 Récapitulatif des commandes

### Commandes de base
```bash
LPUSH key value [value ...]   # Ajouter à gauche
RPUSH key value [value ...]   # Ajouter à droite
LPOP key [count]              # Retirer à gauche
RPOP key [count]              # Retirer à droite
LLEN key                      # Longueur
```

### Lecture
```bash
LRANGE key start stop         # Lire une plage
LINDEX key index              # Accès par index
```

### Modification
```bash
LSET key index value          # Modifier par index
LINSERT key BEFORE|AFTER pivot value  # Insérer
LREM key count value          # Supprimer
LTRIM key start stop          # Garder seulement une plage
```

### Opérations atomiques
```bash
RPOPLPUSH source destination  # Déplacer atomiquement
BRPOPLPUSH source dest timeout  # Version bloquante
```

### Opérations bloquantes
```bash
BLPOP key [key ...] timeout   # LPOP bloquant
BRPOP key [key ...] timeout   # RPOP bloquant
```

---

## 🎓 Points clés à retenir

1. ✅ **Lists = listes doublement chaînées** : O(1) aux extrémités
2. ✅ **LPUSH + RPOP = Queue FIFO** : Premier arrivé, premier servi
3. ✅ **LPUSH + LPOP = Stack LIFO** : Dernier arrivé, premier servi
4. ✅ **BLPOP/BRPOP** : Évitez le polling, bloquez en attendant
5. ✅ **RPOPLPUSH** : Pattern "Reliable Queue" pour ne pas perdre de jobs
6. ✅ **LTRIM** : Indispensable pour limiter la croissance
7. ⚠️ **LINDEX et LRANGE sont O(N)** : Attention aux grandes listes
8. ⚠️ **Lists ≠ accès aléatoire** : Utilisez Hash ou Sorted Set

---

## 🚀 Prochaine étape

Maintenant que vous maîtrisez les Lists pour les queues et timelines, découvrons les **Hashes** pour représenter des objets structurés !

➡️ **Section suivante** : [2.4 Hashes : Représentation d'objets et optimisation mémoire](./04-hashes-objets-optimisation.md)

---

**Durée estimée** : 1h30
**Niveau** : Débutant à Intermédiaire
**Prérequis** : Sections 2.1 et 2.2 complétées

⏭️ [Hashes : Représentation d'objets et optimisation mémoire](/02-structures-donnees-natives/04-hashes-objets-optimisation.md)
