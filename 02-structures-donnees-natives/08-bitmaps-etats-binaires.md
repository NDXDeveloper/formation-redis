🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.8 Bitmaps : Gestion efficace d'états binaires

## 🎯 Objectifs de cette section

À la fin de cette section, vous comprendrez :
- ✅ Comment les Bitmaps manipulent les données au niveau du bit
- ✅ L'économie de mémoire massive (720× moins qu'un Set)
- ✅ Les commandes SETBIT, GETBIT, BITCOUNT, BITPOS, BITOP
- ✅ Les opérations bitwise (AND, OR, XOR, NOT)
- ✅ Les cas d'usage réels (analytics, flags binaires, permissions)

---

## 📘 Les Bitmaps : Manipulation bit par bit

### Qu'est-ce qu'un Bitmap dans Redis ?

Un **Bitmap** n'est pas une structure de données distincte dans Redis. C'est simplement un **String** manipulé au **niveau du bit**. Chaque bit peut être 0 ou 1, permettant de stocker des **états binaires** de manière ultra-compacte.

```bash
# Visualisation d'un Bitmap
# Chaque position = 1 bit (0 ou 1)

Position: 0  1  2  3  4  5  6  7  8  9  10
Valeur:   1  0  1  1  0  0  1  0  1  0  1
          ▲                    ▲
     user:0 connecté     user:7 déconnecté
```

**Caractéristiques** :
- ✅ **1 bit par valeur** : le plus compact possible
- ✅ Opérations bitwise natives (AND, OR, XOR, NOT)
- ✅ Maximum théorique : **2³² bits** (~512 MB, 4 milliards de positions)
- ✅ Complexité **O(1)** pour SETBIT et GETBIT
- ✅ Parfait pour des **flags binaires** (vrai/faux, présent/absent)

### Pourquoi utiliser des Bitmaps ?

**Le problème classique** :
```bash
# Tracker 1 million d'utilisateurs actifs avec un Set
SADD active:users "user:0"
SADD active:users "user:1"
SADD active:users "user:2"
# ... 1 million d'utilisateurs

SCARD active:users
(integer) 1000000

# Mémoire utilisée : ~90 MB 😱
# (90 bytes × 1M utilisateurs)
```

**La solution Bitmap** :
```bash
# Tracker avec des bits (1 = actif, 0 = inactif)
SETBIT active:users 0 1
SETBIT active:users 1 1
SETBIT active:users 2 1
# ... 1 million d'utilisateurs

BITCOUNT active:users
(integer) 1000000

# Mémoire utilisée : 125 KB 🚀
# (1M bits ÷ 8 = 125 000 bytes)
```

**Économie de mémoire : 720× moins !**

---

## 🔧 Commandes de base

### SETBIT : Définir un bit

```bash
# Définir le bit à la position 7 à 1
127.0.0.1:6379> SETBIT mybitmap 7 1
(integer) 0  # Retourne l'ancienne valeur du bit (0 par défaut)

# Définir le bit 100 à 1
127.0.0.1:6379> SETBIT mybitmap 100 1
(integer) 0

# Définir le bit 7 à 0
127.0.0.1:6379> SETBIT mybitmap 7 0
(integer) 1  # Ancienne valeur était 1

# Vérifier le type
127.0.0.1:6379> TYPE mybitmap
string  # C'est un String !

# Voir la représentation
127.0.0.1:6379> GET mybitmap
"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x10"
# Données binaires (difficilement lisibles)
```

**Points importants** :
- Les bits sont initialisés à **0** par défaut
- SETBIT retourne l'**ancienne valeur** du bit
- Si vous définissez le bit 1000000, Redis alloue automatiquement l'espace nécessaire

### GETBIT : Lire un bit

```bash
# Lire le bit à la position 100
127.0.0.1:6379> GETBIT mybitmap 100
(integer) 1

# Lire un bit jamais défini (défaut = 0)
127.0.0.1:6379> GETBIT mybitmap 500
(integer) 0

# Lire le bit 7
127.0.0.1:6379> GETBIT mybitmap 7
(integer) 0  # On l'avait mis à 0
```

### BITCOUNT : Compter les bits à 1

```bash
# Créer un bitmap
127.0.0.1:6379> SETBIT attendance 0 1
(integer) 0
127.0.0.1:6379> SETBIT attendance 5 1
(integer) 0
127.0.0.1:6379> SETBIT attendance 10 1
(integer) 0
127.0.0.1:6379> SETBIT attendance 15 1
(integer) 0

# Compter combien de bits sont à 1
127.0.0.1:6379> BITCOUNT attendance
(integer) 4

# BITCOUNT sur une plage d'octets (bytes)
127.0.0.1:6379> BITCOUNT attendance 0 0
(integer) 1  # Bits à 1 dans le premier octet

127.0.0.1:6379> BITCOUNT attendance 0 1
(integer) 2  # Bits à 1 dans les deux premiers octets

# ⚠️ Attention : BITCOUNT travaille en OCTETS, pas en BITS
# byte 0 = bits 0-7, byte 1 = bits 8-15, etc.
```

### BITPOS : Trouver la première position

```bash
# Créer un bitmap
127.0.0.1:6379> SETBIT flags 10 1
(integer) 0
127.0.0.1:6379> SETBIT flags 20 1
(integer) 0

# Trouver le premier bit à 1
127.0.0.1:6379> BITPOS flags 1
(integer) 10  # Position du premier bit à 1

# Trouver le premier bit à 0
127.0.0.1:6379> BITPOS flags 0
(integer) 0  # Le premier bit (position 0) est à 0

# BITPOS avec plage (start, end en octets)
127.0.0.1:6379> BITPOS flags 1 2 3
# Cherche le premier bit à 1 entre les octets 2 et 3
```

**Cas d'usage** : Trouver le premier slot disponible, la première tâche non traitée, etc.

---

## 🔄 Opérations bitwise : BITOP

BITOP permet d'effectuer des opérations binaires entre plusieurs bitmaps.

### AND : Intersection

```bash
# Créer deux bitmaps (utilisateurs actifs lundi et mardi)
127.0.0.1:6379> SETBIT active:monday 0 1
127.0.0.1:6379> SETBIT active:monday 1 1
127.0.0.1:6379> SETBIT active:monday 2 1

127.0.0.1:6379> SETBIT active:tuesday 1 1
127.0.0.1:6379> SETBIT active:tuesday 2 1
127.0.0.1:6379> SETBIT active:tuesday 3 1

# AND : utilisateurs actifs lundi ET mardi
127.0.0.1:6379> BITOP AND active:both active:monday active:tuesday
(integer) 1  # Taille du bitmap résultat en octets

127.0.0.1:6379> BITCOUNT active:both
(integer) 2  # Utilisateurs 1 et 2 (actifs les deux jours)

# Vérifier
127.0.0.1:6379> GETBIT active:both 0
(integer) 0  # user:0 : actif lundi mais pas mardi

127.0.0.1:6379> GETBIT active:both 1
(integer) 1  # user:1 : actif les deux jours

127.0.0.1:6379> GETBIT active:both 2
(integer) 1  # user:2 : actif les deux jours

127.0.0.1:6379> GETBIT active:both 3
(integer) 0  # user:3 : actif mardi mais pas lundi
```

**Visualisation AND** :
```
Monday:   1 1 1 0
Tuesday:  0 1 1 1
         ─────────
AND:      0 1 1 0  ← Seulement où les deux sont à 1
```

### OR : Union

```bash
# OR : utilisateurs actifs lundi OU mardi (ou les deux)
127.0.0.1:6379> BITOP OR active:either active:monday active:tuesday
(integer) 1

127.0.0.1:6379> BITCOUNT active:either
(integer) 4  # Utilisateurs 0, 1, 2, 3

# Vérifier
127.0.0.1:6379> GETBIT active:either 0
(integer) 1  # user:0 : actif au moins un jour

127.0.0.1:6379> GETBIT active:either 1
(integer) 1

127.0.0.1:6379> GETBIT active:either 2
(integer) 1

127.0.0.1:6379> GETBIT active:either 3
(integer) 1
```

**Visualisation OR** :
```
Monday:   1 1 1 0
Tuesday:  0 1 1 1
         ─────────
OR:       1 1 1 1  ← À 1 si au moins l'un est à 1
```

### XOR : Différence symétrique

```bash
# XOR : utilisateurs actifs un seul des deux jours (exclusif)
127.0.0.1:6379> BITOP XOR active:only_one active:monday active:tuesday
(integer) 1

127.0.0.1:6379> BITCOUNT active:only_one
(integer) 2  # Utilisateurs 0 et 3

# Vérifier
127.0.0.1:6379> GETBIT active:only_one 0
(integer) 1  # user:0 : lundi seulement

127.0.0.1:6379> GETBIT active:only_one 1
(integer) 0  # user:1 : les deux jours (exclu)

127.0.0.1:6379> GETBIT active:only_one 2
(integer) 0  # user:2 : les deux jours (exclu)

127.0.0.1:6379> GETBIT active:only_one 3
(integer) 1  # user:3 : mardi seulement
```

**Visualisation XOR** :
```
Monday:   1 1 1 0
Tuesday:  0 1 1 1
         ─────────
XOR:      1 0 0 1  ← À 1 si différents, 0 si identiques
```

### NOT : Inversion

```bash
# NOT : inverser tous les bits
127.0.0.1:6379> BITOP NOT active:inverted active:monday
(integer) 1

# Bits qui étaient à 0 sont maintenant à 1, et vice-versa
127.0.0.1:6379> GETBIT active:monday 0
(integer) 1

127.0.0.1:6379> GETBIT active:inverted 0
(integer) 0  # Inversé

127.0.0.1:6379> GETBIT active:monday 4
(integer) 0

127.0.0.1:6379> GETBIT active:inverted 4
(integer) 1  # Inversé
```

**Visualisation NOT** :
```
Monday:   1 1 1 0 0 0
         ─────────────
NOT:      0 0 0 1 1 1  ← Tous inversés
```

---

## 📊 Cas d'usage #1 : Utilisateurs actifs quotidiens

### Tracking de présence

```bash
# Utilisateurs actifs le 9 décembre 2024
# user_id 123 se connecte
127.0.0.1:6379> SETBIT active:2024-12-09 123 1
(integer) 0

# user_id 456 se connecte
127.0.0.1:6379> SETBIT active:2024-12-09 456 1
(integer) 0

# user_id 789 se connecte
127.0.0.1:6379> SETBIT active:2024-12-09 789 1
(integer) 0

# user_id 123 se reconnecte (doublon)
127.0.0.1:6379> SETBIT active:2024-12-09 123 1
(integer) 1  # Déjà à 1, pas de changement dans le comptage

# Nombre d'utilisateurs actifs aujourd'hui
127.0.0.1:6379> BITCOUNT active:2024-12-09
(integer) 3

# Vérifier si user:456 était actif
127.0.0.1:6379> GETBIT active:2024-12-09 456
(integer) 1  # Oui

# Vérifier si user:999 était actif
127.0.0.1:6379> GETBIT active:2024-12-09 999
(integer) 0  # Non
```

### Analytics sur 7 jours

```bash
# Simulons une semaine d'activité
127.0.0.1:6379> SETBIT active:2024-12-09 100 1
127.0.0.1:6379> SETBIT active:2024-12-09 101 1
127.0.0.1:6379> SETBIT active:2024-12-09 102 1

127.0.0.1:6379> SETBIT active:2024-12-10 101 1
127.0.0.1:6379> SETBIT active:2024-12-10 102 1
127.0.0.1:6379> SETBIT active:2024-12-10 103 1

127.0.0.1:6379> SETBIT active:2024-12-11 102 1
127.0.0.1:6379> SETBIT active:2024-12-11 103 1
127.0.0.1:6379> SETBIT active:2024-12-11 104 1

# Utilisateurs actifs au moins une fois cette semaine (OR)
127.0.0.1:6379> BITOP OR active:week-49 active:2024-12-09 active:2024-12-10 active:2024-12-11
(integer) 14

127.0.0.1:6379> BITCOUNT active:week-49
(integer) 5  # users 100, 101, 102, 103, 104

# Utilisateurs actifs TOUS les jours (AND)
127.0.0.1:6379> BITOP AND active:daily-users active:2024-12-09 active:2024-12-10 active:2024-12-11
(integer) 14

127.0.0.1:6379> BITCOUNT active:daily-users
(integer) 1  # Seulement user:102

# Utilisateurs actifs exactement 2 jours sur 3
# (Complexe, nécessite plusieurs opérations BITOP)
```

**Code Python pour analytics** :
```python
from datetime import datetime, timedelta

def track_user_activity(user_id, date=None):
    """Marquer un utilisateur comme actif"""
    if date is None:
        date = datetime.now().strftime("%Y-%m-%d")

    key = f"active:{date}"
    redis.setbit(key, user_id, 1)

    # TTL de 90 jours
    redis.expire(key, 7776000)

def get_daily_active_users(date):
    """Compter les utilisateurs actifs du jour"""
    key = f"active:{date}"
    return redis.bitcount(key)

def get_weekly_active_users(start_date):
    """Utilisateurs actifs au moins une fois dans la semaine"""
    keys = [f"active:{start_date + timedelta(days=i)}"
            for i in range(7)]

    # Union (OR)
    redis.bitop("OR", "active:week:temp", *keys)
    count = redis.bitcount("active:week:temp")
    redis.delete("active:week:temp")

    return count

def get_retention_day_0_to_day_7(cohort_date):
    """Rétention : users actifs jour 0 ET jour 7"""
    day_0 = f"active:{cohort_date}"
    day_7 = f"active:{cohort_date + timedelta(days=7)}"

    redis.bitop("AND", "retention:temp", day_0, day_7)
    retained = redis.bitcount("retention:temp")
    total = redis.bitcount(day_0)
    redis.delete("retention:temp")

    return (retained / total * 100) if total > 0 else 0
```

---

## 🎮 Cas d'usage #2 : Feature flags par utilisateur

```bash
# Définir des feature flags pour chaque utilisateur
# Bit 0 : beta_features
# Bit 1 : dark_mode
# Bit 2 : notifications
# Bit 3 : premium

# User 123 : activer beta_features (bit 0)
127.0.0.1:6379> SETBIT user:123:flags 0 1
(integer) 0

# User 123 : activer dark_mode (bit 1)
127.0.0.1:6379> SETBIT user:123:flags 1 1
(integer) 0

# User 123 : activer premium (bit 3)
127.0.0.1:6379> SETBIT user:123:flags 3 1
(integer) 0

# Vérifier si user:123 a beta_features
127.0.0.1:6379> GETBIT user:123:flags 0
(integer) 1  # Oui

# Vérifier si user:123 a notifications
127.0.0.1:6379> GETBIT user:123:flags 2
(integer) 0  # Non

# Désactiver dark_mode
127.0.0.1:6379> SETBIT user:123:flags 1 0
(integer) 1

# Compter combien de flags sont activés
127.0.0.1:6379> BITCOUNT user:123:flags
(integer) 2  # beta_features et premium

# Récupérer toutes les flags d'un utilisateur
127.0.0.1:6379> GET user:123:flags
"\t"  # Représentation binaire (00001001 en binaire = 9 en décimal)
```

**Avantage** : Stockage ultra-compact (32 flags = 4 bytes au lieu de plusieurs clés).

---

## 🔐 Cas d'usage #3 : Permissions granulaires

```bash
# Système de permissions avec 64 permissions possibles
# Bit 0 : read_users
# Bit 1 : write_users
# Bit 2 : delete_users
# Bit 3 : read_posts
# Bit 4 : write_posts
# ...

# Définir les permissions d'un rôle "admin"
127.0.0.1:6379> SETBIT role:admin:permissions 0 1  # read_users
127.0.0.1:6379> SETBIT role:admin:permissions 1 1  # write_users
127.0.0.1:6379> SETBIT role:admin:permissions 2 1  # delete_users
127.0.0.1:6379> SETBIT role:admin:permissions 3 1  # read_posts
127.0.0.1:6379> SETBIT role:admin:permissions 4 1  # write_posts

# Définir les permissions d'un rôle "editor"
127.0.0.1:6379> SETBIT role:editor:permissions 0 1  # read_users
127.0.0.1:6379> SETBIT role:editor:permissions 3 1  # read_posts
127.0.0.1:6379> SETBIT role:editor:permissions 4 1  # write_posts

# Vérifier une permission spécifique
127.0.0.1:6379> GETBIT role:editor:permissions 2
(integer) 0  # Editor ne peut pas delete_users

# Fusionner les permissions de plusieurs rôles (utilisateur multi-rôles)
127.0.0.1:6379> BITOP OR user:123:effective_perms role:admin:permissions role:editor:permissions
(integer) 1

# Vérifier les permissions effectives
127.0.0.1:6379> GETBIT user:123:effective_perms 2
(integer) 1  # Peut delete_users (via role admin)

# Compter le nombre total de permissions
127.0.0.1:6379> BITCOUNT user:123:effective_perms
(integer) 5
```

---

## 📅 Cas d'usage #4 : Calendrier de présence

```bash
# Suivre la présence d'un employé sur 365 jours
# Bit 0 = 1er janvier, Bit 364 = 31 décembre

# Employé 123 présent le 1er janvier (jour 0)
127.0.0.1:6379> SETBIT employee:123:attendance:2024 0 1
(integer) 0

# Présent le 5 janvier (jour 4)
127.0.0.1:6379> SETBIT employee:123:attendance:2024 4 1
(integer) 0

# Présent les jours 8, 9, 10, 11, 12 (semaine complète)
127.0.0.1:6379> SETBIT employee:123:attendance:2024 8 1
127.0.0.1:6379> SETBIT employee:123:attendance:2024 9 1
127.0.0.1:6379> SETBIT employee:123:attendance:2024 10 1
127.0.0.1:6379> SETBIT employee:123:attendance:2024 11 1
127.0.0.1:6379> SETBIT employee:123:attendance:2024 12 1

# Nombre total de jours travaillés dans l'année
127.0.0.1:6379> BITCOUNT employee:123:attendance:2024
(integer) 7

# Était-il présent le 10 janvier (jour 9) ?
127.0.0.1:6379> GETBIT employee:123:attendance:2024 9
(integer) 1  # Oui

# Était-il présent le 15 janvier (jour 14) ?
127.0.0.1:6379> GETBIT employee:123:attendance:2024 14
(integer) 0  # Non

# Trouver le premier jour travaillé
127.0.0.1:6379> BITPOS employee:123:attendance:2024 1
(integer) 0  # 1er janvier

# Taux de présence sur les 30 premiers jours
127.0.0.1:6379> BITCOUNT employee:123:attendance:2024 0 3
(integer) 7  # 7 jours sur les 4 premières semaines (32 bits)
```

**Mémoire utilisée** :
- 365 bits = 46 bytes par employé
- 1000 employés = 46 KB
- Comparez avec 1000 × 365 = 365 000 entrées dans une DB !

---

## 🎯 Cas d'usage #5 : Bloom filter simple

```bash
# Simuler un bloom filter basique avec Bitmap
# (Version simplifiée, pas cryptographiquement sûr)

# Ajouter "alice" au filter
# Hash simple : sum(ord(c)) mod 1000
# hash("alice") = (97+108+105+99+101) % 1000 = 510
127.0.0.1:6379> SETBIT bloom:users 510 1
(integer) 0

# Ajouter "bob"
# hash("bob") = (98+111+98) % 1000 = 307
127.0.0.1:6379> SETBIT bloom:users 307 1
(integer) 0

# Vérifier si "alice" est probablement présent
127.0.0.1:6379> GETBIT bloom:users 510
(integer) 1  # Probablement oui (peut être faux positif)

# Vérifier si "charlie" est probablement présent
# hash("charlie") = 728
127.0.0.1:6379> GETBIT bloom:users 728
(integer) 0  # Définitivement non

# Note : Pour un vrai bloom filter, utilisez RedisBloom (Redis Stack)
```

---

## 🔢 BITFIELD : Opérations sur plusieurs bits

BITFIELD permet de lire/écrire/incrémenter des **groupes de bits** (entiers signés ou non).

```bash
# Stocker plusieurs compteurs dans un même bitmap
# Positions 0-7 : compteur A
# Positions 8-15 : compteur B
# Positions 16-23 : compteur C

# Définir compteur A à 10 (8 bits, unsigned)
127.0.0.1:6379> BITFIELD counters SET u8 0 10
1) (integer) 0  # Ancienne valeur

# Définir compteur B à 20
127.0.0.1:6379> BITFIELD counters SET u8 8 20
1) (integer) 0

# Définir compteur C à 30
127.0.0.1:6379> BITFIELD counters SET u8 16 30
1) (integer) 0

# Lire compteur A
127.0.0.1:6379> BITFIELD counters GET u8 0
1) (integer) 10

# Incrémenter compteur A de 5
127.0.0.1:6379> BITFIELD counters INCRBY u8 0 5
1) (integer) 15

# Lire tous les compteurs
127.0.0.1:6379> BITFIELD counters GET u8 0 GET u8 8 GET u8 16
1) (integer) 15
2) (integer) 20
3) (integer) 30

# Opérations multiples en une seule commande (atomique)
127.0.0.1:6379> BITFIELD counters INCRBY u8 0 1 INCRBY u8 8 2 INCRBY u8 16 3
1) (integer) 16
2) (integer) 22
3) (integer) 33
```

**Types disponibles** :
- `u8` : unsigned 8 bits (0-255)
- `i8` : signed 8 bits (-128 à 127)
- `u16`, `i16`, `u32`, `i32`, `u64`, `i64`

**Cas d'usage** : Compteurs multiples, données structurées compactes.

---

## 💾 Économie de mémoire : Démonstration

### Comparaison : 1 million d'utilisateurs

| Structure | Mémoire | Opérations | Récupération |
|-----------|---------|------------|--------------|
| **Set** | ~90 MB | SADD, SISMEMBER | ✅ SMEMBERS |
| **HyperLogLog** | 12 KB | PFADD, PFCOUNT | ❌ Comptage seul |
| **Bitmap** | 125 KB | SETBIT, GETBIT | ❌ Vérification individuelle |

**Bitmap détails** :
- 1 million de bits = 1 000 000 ÷ 8 = 125 000 bytes = 125 KB
- Économie vs Set : 90 MB ÷ 125 KB = **720× moins**
- Économie vs HyperLogLog : Non applicable (usages différents)

### Quand choisir Bitmap vs Set

```bash
# ✅ Utilisez Bitmap si :
# - Identifiants séquentiels (user_id 0, 1, 2, ...)
# - États binaires (présent/absent, actif/inactif)
# - Beaucoup d'IDs (> 100 000)
# - Opérations bitwise (AND, OR, XOR)
# - Mémoire limitée

# Exemple : 10 millions d'utilisateurs avec IDs 0-9999999
SETBIT users:active 5000000 1  # User ID 5 000 000
# Mémoire : 10M bits = 1.25 MB

# ✅ Utilisez Set si :
# - Identifiants non séquentiels ("user:abc", "user:xyz")
# - Besoin de lister les membres
# - Petits volumes (< 10 000)
# - Opérations ensemblistes complexes

# Exemple : IDs aléatoires
SADD users:active "user:8a7b3c"
SADD users:active "user:f2e9d1"
# Peut récupérer : SMEMBERS users:active
```

---

## ⚡ Complexité et performance

| Commande | Complexité | Notes |
|----------|------------|-------|
| `SETBIT` | O(1) | |
| `GETBIT` | O(1) | |
| `BITCOUNT` | O(N) | N = taille du bitmap en octets |
| `BITPOS` | O(N) | N = taille du bitmap |
| `BITOP` | O(N) | N = taille du plus grand bitmap |
| `BITFIELD` | O(1) | Par opération |

**Notes importantes** :
- SETBIT sur une position éloignée (ex: bit 100 000 000) peut être lent la première fois car Redis doit allouer l'espace
- BITCOUNT est optimisé avec des algorithmes de comptage rapide (population count)
- Les opérations sur de gros bitmaps (> 10 MB) peuvent bloquer Redis

---

## 🚨 Limitations et pièges

### 1. IDs non séquentiels = gaspillage

```bash
# ❌ MAUVAIS : IDs éparses
SETBIT users "user:abc123" 1  # Erreur : offset doit être un entier !

# Même avec des IDs numériques éparses :
SETBIT users 1000000 1  # OK
SETBIT users 9999999 1  # OK
# Redis alloue 1.25 MB pour 2 utilisateurs ! 💸

# ✅ BON : IDs séquentiels ou mapper vers 0, 1, 2, ...
SETBIT users 0 1
SETBIT users 1 1
SETBIT users 2 1
# Seulement quelques bytes
```

**Solution** : Si vos IDs ne sont pas séquentiels, créez un mapping :
```python
# Mapping ID → position
id_to_pos = {}
next_pos = 0

def get_position(user_id):
    if user_id not in id_to_pos:
        id_to_pos[user_id] = next_pos
        next_pos += 1
    return id_to_pos[user_id]

# Utiliser
pos = get_position("user:abc123")
redis.setbit("users:active", pos, 1)
```

### 2. BITCOUNT et BITPOS en octets

```bash
# ⚠️ BITCOUNT utilise des OCTETS (bytes), pas des bits !
127.0.0.1:6379> SETBIT test 0 1
127.0.0.1:6379> SETBIT test 7 1
127.0.0.1:6379> SETBIT test 8 1

# Compter bits dans le premier octet (bits 0-7)
127.0.0.1:6379> BITCOUNT test 0 0
(integer) 2  # bits 0 et 7

# Compter bits dans les deux premiers octets (bits 0-15)
127.0.0.1:6379> BITCOUNT test 0 1
(integer) 3  # bits 0, 7 et 8

# Conversion : octet N = bits 8*N à 8*N+7
```

### 3. Mémoire pré-allouée

```bash
# Définir un bit très loin alloue TOUTE la mémoire jusqu'à ce bit
127.0.0.1:6379> SETBIT sparse 100000000 1
OK

127.0.0.1:6379> MEMORY USAGE sparse
(integer) 12500016  # ~12.5 MB alloués ! 💸

# Même si un seul bit est utilisé
127.0.0.1:6379> BITCOUNT sparse
(integer) 1
```

**Conseil** : Gardez vos bitmaps denses (IDs contigus).

### 4. Pas de suppression de bits

```bash
# On peut mettre à 0, mais ça ne libère pas la mémoire
127.0.0.1:6379> SETBIT mybitmap 1000000 1
OK

127.0.0.1:6379> MEMORY USAGE mybitmap
(integer) 125016

127.0.0.1:6379> SETBIT mybitmap 1000000 0
(integer) 1  # Mis à 0

127.0.0.1:6379> MEMORY USAGE mybitmap
(integer) 125016  # Mémoire inchangée !

# Pour libérer, il faut supprimer la clé entière
127.0.0.1:6379> DEL mybitmap
(integer) 1
```

### 5. Confusion bit vs byte

```bash
# Position en BITS, plages en OCTETS

# Positions de bits (SETBIT, GETBIT)
SETBIT key 15 1    # Bit 15

# Plages d'octets (BITCOUNT)
BITCOUNT key 0 1   # Octets 0-1 (bits 0-15)
```

---

## 📋 Checklist : Quand utiliser un Bitmap

### ✅ Utilisez un Bitmap pour :
- États **binaires** (oui/non, actif/inactif, présent/absent)
- IDs **séquentiels** et **denses** (0, 1, 2, 3, ...)
- **Analytics** : utilisateurs actifs, engagement
- **Permissions** et feature flags
- **Calendriers** de présence, disponibilité
- Volumes **> 100 000** éléments
- Mémoire **très limitée**
- Opérations **bitwise** (AND, OR, XOR)

### ❌ N'utilisez PAS un Bitmap pour :
- IDs **non séquentiels** ou **éparses** → Set ou Hash
- Besoin de **lister** les membres → Set
- Données **non binaires** (scores, compteurs > 1) → Sorted Set, Hash
- **Petits volumes** (< 1000) → Set est plus simple
- Besoin de **supprimer** des éléments individuels → Set

---

## 🎓 Points clés à retenir

1. ✅ **Bitmap = String manipulé bit par bit** : 1 bit par valeur
2. ✅ **Économie de mémoire massive** : 720× moins qu'un Set
3. ✅ **O(1) pour SETBIT/GETBIT** : ultra-rapide
4. ✅ **Opérations bitwise** : AND, OR, XOR, NOT pour combiner des bitmaps
5. ✅ **BITFIELD** : manipuler des groupes de bits (compteurs compacts)
6. ⚠️ **IDs séquentiels requis** : sinon gaspillage de mémoire
7. ⚠️ **BITCOUNT travaille en octets** : attention aux plages
8. ⚠️ **Pré-allocation** : définir bit 1M alloue 125 KB
9. 🎯 Idéal pour : analytics, flags, permissions, présence

---

## 🚀 Prochaine étape

Vous maîtrisez maintenant toutes les structures de données natives de Redis ! Découvrons la dernière section sur la **complexité algorithmique** pour optimiser vos choix.

➡️ **Section suivante** : [2.9 Complexité algorithmique (Big O) des commandes](./09-complexite-algorithmique-big-o.md)

---

**Durée estimée** : 1h30
**Niveau** : Intermédiaire
**Prérequis** : Sections 2.1 à 2.7 complétées

⏭️ [Complexité algorithmique (Big O) des commandes](/02-structures-donnees-natives/09-complexite-algorithmique-big-o.md)
