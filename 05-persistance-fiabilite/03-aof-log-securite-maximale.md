🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3 AOF (Append Only File) : Log et sécurité maximale

## Introduction

AOF (Append Only File) est le mécanisme de persistance de Redis qui privilégie la **durabilité maximale** des données. Contrairement à RDB qui crée des snapshots périodiques, AOF fonctionne comme un **journal de transactions** : chaque commande d'écriture est enregistrée dans un fichier log en temps réel.

### Principes fondamentaux

- **Format** : Fichier texte (protocole Redis RESP)
- **Contenu** : Séquence chronologique de toutes les commandes d'écriture
- **Déclenchement** : Continu (chaque opération d'écriture)
- **Mécanisme** : Append (ajout à la fin du fichier)
- **Durabilité** : Configurable (fsync no/everysec/always)

### Le concept du journal de transactions

```
État initial : Base vide
┌─────────────────────────────────────┐
│ appendonly.aof (vide)               │
└─────────────────────────────────────┘

Après quelques commandes :
┌─────────────────────────────────────┐
│ *3                                  │  ← SET key1 "value1"
│ $3                                  │
│ SET                                 │
│ $4                                  │
│ key1                                │
│ $6                                  │
│ value1                              │
│ *3                                  │  ← SET key2 "value2"
│ $3                                  │
│ SET                                 │
│ ...                                 │
└─────────────────────────────────────┘

Au redémarrage : Redis rejoue toutes les commandes
```

**Avantage principal** : Perte de données minimale (configurable de 0 à 1 seconde).

## Architecture et fonctionnement interne

### Le pipeline d'écriture AOF

```
Client envoie SET key value
         ↓
┌─────────────────────────────────────┐
│ 1. Redis exécute la commande        │
│    (mise à jour en mémoire)         │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 2. Conversion en protocole RESP     │
│    (format AOF)                     │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 3. Ajout au buffer AOF              │
│    (en mémoire)                     │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 4. write() vers le fichier AOF      │
│    (buffer OS)                      │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 5. fsync() selon configuration      │
│    (flush vers disque physique)     │
└─────────────────────────────────────┘
         ↓
    Données durables
```

### Les 3 modes de synchronisation (fsync)

Le paramètre `appendfsync` contrôle **quand** les données sont effectivement écrites sur le disque physique :

| Mode | Fréquence fsync | Durabilité | Performance | Perte max |
|------|----------------|------------|-------------|-----------|
| **no** | Jamais (OS décide) | ⭐ | ⭐⭐⭐⭐⭐ | 30 secondes |
| **everysec** | Toutes les secondes | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ~1 seconde |
| **always** | Après chaque commande | ⭐⭐⭐⭐⭐ | ⭐⭐ | Aucune* |

*Sauf crash kernel ou panne matérielle

## Mode 1 : appendfsync no (Dangereux)

### Configuration

```conf
appendonly yes
appendfsync no
```

### Comportement détaillé

**Processus :**
1. Redis écrit les commandes dans le buffer AOF (mémoire)
2. `write()` système transfert vers le buffer OS
3. L'OS décide **quand** faire le fsync (typiquement toutes les 30 secondes)
4. Aucune garantie de durabilité

```
Commande → Buffer Redis → Buffer OS → ... → Disque (30s+ plus tard)
                                      ↑
                              OS contrôle le timing
```

### Caractéristiques

**Avantages :**
- ✅ Performance maximale (proche de "no persistence")
- ✅ Aucun fsync bloquant
- ✅ Latence minimale

**Inconvénients :**
- ❌ **Perte de jusqu'à 30 secondes** de données
- ❌ Aucune garantie de durabilité
- ❌ Dépend du comportement de l'OS

### Métriques typiques

```
Ops/sec: 90,000-100,000
Latence P50: 0.2ms
Latence P99: 0.8ms
Durabilité: Très faible
```

### Quand l'utiliser

❌ **JAMAIS en production** sauf cas très spécifiques :
- Cache avec AOF activé uniquement pour éviter le warm-up long
- Environnement de développement
- Données totalement non-critiques

**Recommandation** : Préférer RDB seul si la durabilité n'est pas importante.

## Mode 2 : appendfsync everysec (RECOMMANDÉ)

### Configuration

```conf
appendonly yes
appendfsync everysec  # ← Compromis optimal
```

### Comportement détaillé

**Processus :**
1. Redis écrit les commandes dans le buffer AOF (immédiat)
2. `write()` système transfert vers le buffer OS (immédiat)
3. Un thread en arrière-plan fait `fsync()` **toutes les secondes**
4. Impact minimal sur le thread principal

```
Thread principal (non-bloqué) :
Commande → Buffer Redis → Buffer OS → Réponse client (0.3ms)

Thread background (1x/seconde) :
Buffer OS → fsync() → Disque (ne bloque pas le principal)
```

### Chronologie du fsync

```
Seconde 0.0: fsync() effectué
  ├─ 0.1s: SET key1 val1  ← buffer OS
  ├─ 0.3s: SET key2 val2  ← buffer OS
  ├─ 0.5s: SET key3 val3  ← buffer OS
  ├─ 0.8s: SET key4 val4  ← buffer OS
Seconde 1.0: fsync() effectué  ← Toutes les données persistées
  ├─ 1.2s: SET key5 val5
  ...
```

### Caractéristiques

**Avantages :**
- ✅ **Excellent compromis durabilité/performance**
- ✅ Perte maximale de ~1 seconde de données
- ✅ Impact minimal sur la latence (fsync en background)
- ✅ Production-ready pour 90% des cas

**Inconvénients :**
- ⚠️ Légère réduction de performance (-10 à -20% vs no persist)
- ⚠️ Possible perte d'1 seconde en cas de crash
- ⚠️ Utilisation I/O disque continue

### Métriques typiques

```
Ops/sec: 60,000-80,000 (-20 à -30% vs no persist)
Latence P50: 0.3ms
Latence P99: 2-5ms
Latence max: Pics à 10-20ms lors du fsync
Durabilité: Excellente (perte max 1s)
```

### Impact du fsync sur la latence

```
Distribution de latence :

Sans AOF :
[0.2ms] [0.2ms] [0.2ms] [0.2ms] [0.2ms] ...
    ↓       ↓       ↓       ↓       ↓
 P50=0.2  P99=0.5  P999=0.8

Avec AOF everysec :
[0.3ms] [0.3ms] [2ms*] [0.3ms] [0.4ms] [15ms*] [0.3ms] ...
                  ↑                      ↑
            fsync en cours        fsync bloque temporairement
    ↓       ↓       ↓       ↓       ↓
 P50=0.3  P99=3    P999=15
```

*Les pics de latence surviennent lorsque le fsync prend plus de temps que prévu (disque saturé, fragmentation).

### Quand l'utiliser

✅ **Cas d'usage recommandés :**
- Session stores
- Job queues
- Compteurs métiers
- Leaderboards
- **Tout cas de production standard**

**Configuration recommandée :**
```conf
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes  # Redis 7+ (format hybride)

# Réécriture automatique
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

## Mode 3 : appendfsync always (Durabilité maximale)

### Configuration

```conf
appendonly yes
appendfsync always  # ← Chaque commande fsync
```

### Comportement détaillé

**Processus :**
1. Redis écrit la commande dans le buffer AOF
2. `write()` système transfert vers le buffer OS
3. `fsync()` **immédiat** pour forcer l'écriture disque
4. Redis **attend** la confirmation du fsync
5. Seulement après, la réponse est envoyée au client

```
Client envoie SET key value
         ↓
Redis exécute (mémoire)
         ↓
Append au buffer AOF
         ↓
write() vers OS buffer
         ↓
fsync() IMMÉDIAT  ← BLOQUANT (5-20ms)
         ↓
Confirmation disque
         ↓
Réponse au client (TOTAL: 5-50ms)
```

### Caractéristiques

**Avantages :**
- ✅ **Durabilité maximale** : Aucune perte de données*
- ✅ Garanties ACID par commande
- ✅ Audit trail complet et fiable
- ✅ Conformité réglementaire (finance, santé)

**Inconvénients :**
- ❌ **Performance catastrophique** : 100-500 ops/s (vs 100K sans persist)
- ❌ Latence très élevée (10-50ms par opération)
- ❌ Utilisation I/O disque maximale (100%)
- ❌ Coût infrastructure x5-10

*Sauf crash kernel, panne matérielle, ou problème physique du disque

### Métriques typiques

```
Ops/sec: 100-500 (dépend fortement du disque)
Latence P50: 8-15ms
Latence P99: 30-80ms
Latence max: 100-200ms
Durabilité: Maximale
Débit écriture: Réduit de 99%+
```

### Benchmark comparatif

| Configuration | SET/sec | Ratio vs baseline |
|---------------|---------|-------------------|
| No persistence | 98,000 | 100% (baseline) |
| AOF no | 92,000 | 94% |
| AOF everysec | 67,000 | 68% |
| AOF always (HDD) | 180 | 0.18% |
| AOF always (SSD) | 450 | 0.46% |
| AOF always (NVMe) | 1,200 | 1.2% |

**Conclusion** : Même avec du matériel haut de gamme (NVMe), AOF always réduit le débit de **98%**.

### Quand l'utiliser

✅ **Uniquement pour :**
- Transactions financières (paiements, trading)
- Systèmes de vote critiques
- Enchères en temps réel
- Compliance stricte (SOX, PCI-DSS, HIPAA)
- Données médicales ou légales

❌ **Ne PAS utiliser pour :**
- Caches (overkill total)
- Session stores (everysec suffit)
- Leaderboards (everysec suffit)
- Analytics temps réel (everysec suffit)

**Architecture recommandée si utilisé :**
```
Load Balancer
      ├─ Redis 1 (AOF always)  ← 500 ops/s
      ├─ Redis 2 (AOF always)  ← 500 ops/s
      ├─ Redis 3 (AOF always)  ← 500 ops/s
      └─ Redis N ...

Débit total: 500 × N ops/s
Coût: x5-10 vs configuration standard
```

## Format du fichier AOF

### Structure du fichier

Le fichier AOF utilise le **protocole RESP** (REdis Serialization Protocol), un format texte lisible :

```
*3              ← Nombre d'arguments (3)
$3              ← Longueur du 1er argument (3 bytes)
SET             ← 1er argument (commande)
$4              ← Longueur du 2e argument (4 bytes)
key1            ← 2e argument (clé)
$6              ← Longueur du 3e argument (6 bytes)
value1          ← 3e argument (valeur)
```

### Exemple complet de fichier AOF

```
*2
$6
SELECT
$1
0
*3
$3
SET
$5
user1
$12
{"id":"123"}
*3
$3
SET
$5
user2
$12
{"id":"456"}
*5
$4
ZADD
$11
leaderboard
$1
100
$5
user1
*3
$4
INCR
$7
counter
```

### Format lisible vs efficace

**Avantages du format texte :**
- ✅ **Lisible humainement** (debugging)
- ✅ **Éditable** (réparation manuelle possible)
- ✅ **Portable** entre versions Redis
- ✅ **Auditable** (voir toutes les opérations)

**Inconvénients :**
- ❌ **Taille importante** (3-5x plus gros que RDB)
- ❌ **Redondant** (même clé répétée plusieurs fois)
- ❌ **Croissance continue** sans réécriture

### Taille du fichier AOF

**Estimation :**
```
Taille AOF (sans réécriture) ≈ Nombre de commandes × 50-200 bytes
```

**Exemple :**
- 1 million de SET : ~100 MB AOF
- 10 millions de SET : ~1 GB AOF
- 100 millions de SET : ~10 GB AOF

**Impact des types de données :**

| Type de données | Taille moyenne par commande |
|-----------------|---------------------------|
| SET (petites strings) | 50-80 bytes |
| SET (grandes strings) | 200-2000 bytes |
| HSET (hash) | 70-100 bytes |
| LPUSH (list) | 60-90 bytes |
| ZADD (sorted set) | 80-120 bytes |
| INCR | 40-60 bytes |

## Réécriture AOF (BGREWRITEAOF)

### Pourquoi réécrire l'AOF ?

Le fichier AOF grandit indéfiniment car il contient **toutes les commandes**, y compris celles qui sont devenues obsolètes :

```
Avant réécriture (100 commandes) :
SET key1 "a"       ← Obsolète
SET key1 "b"       ← Obsolète
SET key1 "c"       ← Obsolète
...
SET key1 "z"       ← Valeur finale
DEL key2           ← Ces commandes
SET key2 "value"   ← S'annulent
INCR counter       ← 100 fois
INCR counter       ← pour atteindre
...                ← le même résultat

Taille: 10 MB

Après réécriture (état final) :
SET key1 "z"       ← Seule la valeur finale
SET key2 "value"
SET counter 100    ← Résultat direct

Taille: 200 KB (réduction de 98%)
```

### Mécanisme de la réécriture

```
1. Déclenchement (automatique ou BGREWRITEAOF)
         ↓
2. Fork du processus (comme RDB)
         ↓
3. Processus enfant génère le nouvel AOF
   (état actuel de la base en commandes)
         ↓
4. Pendant ce temps :
   - Parent continue à servir les requêtes
   - Parent continue à écrire ancien AOF
   - Parent accumule les nouvelles commandes dans un buffer
         ↓
5. Enfant termine le nouvel AOF
         ↓
6. Parent append le buffer de différences
         ↓
7. Rename atomique : new-aof → appendonly.aof
         ↓
8. Ancien AOF supprimé
```

### Configuration de la réécriture automatique

```conf
# Déclencher réécriture si taille AOF a doublé
auto-aof-rewrite-percentage 100

# Taille minimale avant première réécriture
auto-aof-rewrite-min-size 64mb
```

**Logique de déclenchement :**
```
Conditions :
1. Taille actuelle AOF >= auto-aof-rewrite-min-size
2. Taille actuelle >= (Taille après dernière réécriture × (1 + percentage/100))

Exemple :
- Dernière réécriture : 100 MB
- auto-aof-rewrite-percentage: 100
- Déclenchement si : taille actuelle >= 100 × (1 + 100/100) = 200 MB
```

### Tableaux des stratégies de réécriture

| Stratégie | Percentage | Min size | Fréquence | Overhead I/O | Taille AOF |
|-----------|-----------|----------|-----------|--------------|------------|
| **Conservatrice** | 200 | 128mb | Rare | Faible | Grande |
| **Standard** | 100 | 64mb | Modérée | Moyen | Moyenne |
| **Agressive** | 50 | 32mb | Fréquente | Élevé | Petite |

**Impact de la stratégie :**

| Stratégie | Exemple taille | Réécritures/jour | Bon pour |
|-----------|----------------|------------------|----------|
| Conservatrice | 1 GB → 200 MB → 600 MB → ... | 2-3 | Workload stable |
| Standard | 100 MB → 64 MB → 128 MB → ... | 5-10 | **Production standard** |
| Agressive | 50 MB → 32 MB → 48 MB → ... | 20-50 | Write-heavy, espace limité |

### Commande manuelle de réécriture

```bash
# Déclencher une réécriture manuelle
redis-cli BGREWRITEAOF

# Vérifier le statut
redis-cli INFO persistence | grep aof_rewrite_in_progress
aof_rewrite_in_progress:0

# Statistiques de la dernière réécriture
redis-cli INFO persistence | grep aof_last_rewrite
aof_last_rewrite_time_sec:5
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
```

### Optimisation de la réécriture

```conf
# Désactiver fsync pendant la réécriture (plus rapide)
no-appendfsync-on-rewrite no  # Valeur par défaut

# Alternative : activer pour plus de performance (risque !)
no-appendfsync-on-rewrite yes  # Pas de fsync pendant rewrite
```

**Impact de `no-appendfsync-on-rewrite` :**

| Valeur | Performance rewrite | Durabilité pendant rewrite | Recommandation |
|--------|--------------------|-----------------------------|----------------|
| **no** (défaut) | Normal | ✅ Maintenue | Production standard |
| **yes** | 30-50% plus rapide | ⚠️ Réduite (buffer OS) | Acceptable si réplication |

## Format hybride AOF (Redis 7+)

### Le problème résolu

**Avant Redis 7 :**
```
AOF classique :
- Taille importante (format texte)
- Temps de chargement long
- Réécritures fréquentes nécessaires
```

**Avec Redis 7+ :**
```
AOF hybride :
- Préambule RDB (compact, binaire)
- Suivi des commandes récentes en AOF
- Meilleur des deux mondes
```

### Structure du fichier AOF hybride

```
┌──────────────────────────────────────┐
│ Préambule RDB (snapshot complet)     │  ← Compact, rapide à charger
│ REDIS0011...                         │
│ [Données en format RDB]              │
│ ...                                  │
├──────────────────────────────────────┤
│ Séparateur                           │
├──────────────────────────────────────┤
│ Delta AOF (commandes récentes)       │  ← Commandes depuis le snapshot
│ *3                                   │
│ $3                                   │
│ SET                                  │
│ ...                                  │
└──────────────────────────────────────┘
```

### Configuration

```conf
# Activer le format hybride (Redis 7+)
aof-use-rdb-preamble yes  # ← Recommandé !
```

### Avantages du format hybride

**Comparaison de taille :**

| Configuration | Taille fichier | Temps chargement |
|---------------|----------------|------------------|
| AOF pur | 1000 MB | 60 secondes |
| RDB seul | 300 MB | 10 secondes |
| **AOF hybride** | **350 MB** | **12 secondes** |

**Bénéfices :**
- ✅ Taille réduite de 60-70% vs AOF pur
- ✅ Chargement 4-5x plus rapide
- ✅ Réécritures moins fréquentes
- ✅ Compatibilité avec versions anciennes

### Évolution lors des réécritures

```
État initial (après rewrite) :
┌────────────────────┐
│ RDB Preamble       │  100 MB
│ AOF Delta          │  5 MB
└────────────────────┘
Total: 105 MB

Après 1 heure d'activité :
┌────────────────────┐
│ RDB Preamble       │  100 MB (inchangé)
│ AOF Delta          │  50 MB (nouvelles commandes)
└────────────────────┘
Total: 150 MB

Après rewrite :
┌────────────────────┐
│ RDB Preamble       │  102 MB (nouvel état)
│ AOF Delta          │  2 MB (dernières commandes)
└────────────────────┘
Total: 104 MB
```

## AOF Multi-Part (Redis 7.0+)

### Évolution de l'architecture

**Redis 6 et avant : Fichier unique**
```
appendonly.aof  (tout dans un fichier)
```

**Redis 7+ : Multi-part AOF**
```
appendonlydir/
  ├── appendonly.aof.1.base.rdb       ← Base (snapshot RDB)
  ├── appendonly.aof.1.incr.aof       ← Incrémental 1
  ├── appendonly.aof.2.incr.aof       ← Incrémental 2
  └── appendonly.aof.manifest         ← Manifeste (index)
```

### Avantages de l'architecture multi-part

| Aspect | AOF unique | AOF multi-part |
|--------|------------|----------------|
| **Réécriture** | Bloque l'ancien fichier | Non-bloquant |
| **Atomicité** | Rename fragile | Manifeste atomique |
| **Corruption** | Tout le fichier perdu | Seul le segment affecté |
| **Performance** | Réécritures coûteuses | Réécritures optimisées |

### Configuration

```conf
# Redis 7+ utilise automatiquement multi-part
appendonly yes
appenddirname "appendonlydir"  # Répertoire pour les fichiers AOF
```

**Structure des fichiers :**

| Fichier | Type | Description |
|---------|------|-------------|
| `*.base.rdb` | RDB | Snapshot de base (préambule) |
| `*.incr.aof` | AOF | Commandes incrémentales |
| `*.manifest` | JSON | Fichier d'index (quels fichiers utiliser) |

### Exemple de manifeste

```json
{
  "version": 1,
  "sequence": 2,
  "files": [
    {
      "type": "base",
      "file": "appendonly.aof.1.base.rdb",
      "size": 104857600
    },
    {
      "type": "incr",
      "file": "appendonly.aof.1.incr.aof",
      "size": 5242880
    },
    {
      "type": "incr",
      "file": "appendonly.aof.2.incr.aof",
      "size": 2097152
    }
  ]
}
```

## Avantages et inconvénients d'AOF

### Tableau comparatif complet

| Critère | AOF | Évaluation |
|---------|-----|------------|
| **Durabilité** | Excellente (0-1s perte) | ⭐⭐⭐⭐⭐ |
| **Performance** | Bonne (everysec) | ⭐⭐⭐⭐ |
| **Taille fichier** | Importante (3-5x RDB) | ⭐⭐ |
| **Temps chargement** | Moyen (1-5 min) | ⭐⭐⭐ |
| **Lisibilité** | Format texte lisible | ⭐⭐⭐⭐⭐ |
| **Réparabilité** | Possible avec redis-check-aof | ⭐⭐⭐⭐ |
| **Complexité** | Moyenne (réécritures) | ⭐⭐⭐ |
| **Audit trail** | Complet | ⭐⭐⭐⭐⭐ |
| **Overhead CPU** | Moyen | ⭐⭐⭐ |
| **Overhead I/O** | Élevé | ⭐⭐ |

### Avantages détaillés

#### ✅ 1. Durabilité maximale

```
RDB seul : Perte de 5-15 minutes
AOF everysec : Perte de ~1 seconde
AOF always : Aucune perte*

* Sauf crash matériel
```

#### ✅ 2. Journal auditable

```bash
# Voir les dernières opérations
tail -f appendonly.aof

# Rechercher une opération spécifique
grep "user:12345" appendonly.aof

# Replay jusqu'à un point dans le temps
head -n 10000 appendonly.aof > partial.aof
redis-cli --pipe < partial.aof
```

#### ✅ 3. Réparation possible

```bash
# Vérifier et réparer un AOF corrompu
redis-check-aof --fix appendonly.aof

# Sortie :
AOF analyzed: size=102400, ok_up_to=100000, diff=2400
This will shrink the AOF from 102400 bytes, with 2400 bytes to trash.
Continue? [y/N]: y
Successfully truncated AOF
```

**Contrairement à RDB** : Si RDB est corrompu, il est perdu. AOF peut être réparé.

#### ✅ 4. Format lisible

```bash
# Lire un fichier AOF
cat appendonly.aof | grep SET | head -5

# Convertir en commandes Redis
cat appendonly.aof | redis-cli --pipe
```

### Inconvénients détaillés

#### ❌ 1. Taille importante

**Comparaison typique :**
```
Dataset en RAM : 10 GB
RDB : 3 GB (30%)
AOF pur : 15-20 GB (150-200%)
AOF hybride : 4-5 GB (40-50%)
```

**Impact :**
- Espace disque important
- Backups plus coûteux
- Transferts réseau plus longs

#### ❌ 2. Performance réduite

**Impact de fsync everysec :**
```
Sans AOF : 100,000 ops/s
Avec AOF : 70,000 ops/s (-30%)

Pics de latence lors des fsync :
P50 : 0.3ms → 0.4ms (+33%)
P99 : 0.8ms → 5ms (+525%)
P999 : 2ms → 15ms (+650%)
```

#### ❌ 3. Chargement plus lent

**Temps de démarrage :**

| Dataset | RDB | AOF pur | AOF hybride |
|---------|-----|---------|-------------|
| 1 GB | 5s | 30s | 8s |
| 10 GB | 30s | 5 min | 45s |
| 50 GB | 3 min | 25 min | 4 min |
| 100 GB | 6 min | 50 min | 8 min |

**Problème** : Redis est indisponible pendant le chargement.

#### ❌ 4. Overhead I/O continu

```
RDB : Pics d'I/O périodiques
AOF : I/O continu

Impact sur le disque :
- Usure accélérée (SSD)
- Saturation possible (HDD)
- Compétition avec autres processus
```

## Configuration de production optimale

### Configuration complète recommandée

```conf
# === ACTIVATION AOF ===
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec              # Compromis optimal

# === FORMAT HYBRIDE (Redis 7+) ===
aof-use-rdb-preamble yes         # Préambule RDB + delta AOF
appenddirname "appendonlydir"    # Multi-part AOF

# === RÉÉCRITURE AUTOMATIQUE ===
auto-aof-rewrite-percentage 100  # Réécrire si taille doublée
auto-aof-rewrite-min-size 64mb   # Taille min avant réécriture

# === OPTIMISATIONS ===
no-appendfsync-on-rewrite no     # Maintenir durabilité pendant rewrite
aof-load-truncated yes           # Tolérer AOF tronqué au chargement
aof-timestamp-enabled yes        # Redis 7.0+: timestamp dans AOF

# === RÉPERTOIRE ===
dir /var/lib/redis               # Répertoire de travail (SSD recommandé)
```

### Configuration par cas d'usage

#### Cas 1 : Session store (standard)

```conf
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Justification :**
- Perte de 1s acceptable pour sessions
- Performance équilibrée
- Configuration standard

#### Cas 2 : Job queue critique

```conf
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
auto-aof-rewrite-percentage 50    # Réécriture plus fréquente
auto-aof-rewrite-min-size 32mb
```

**Justification :**
- Jobs critiques nécessitent durabilité forte
- Réécritures fréquentes pour limiter taille AOF
- everysec reste suffisant (vs always)

#### Cas 3 : Données financières

```conf
appendonly yes
appendfsync always                # Durabilité maximale
aof-use-rdb-preamble yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
no-appendfsync-on-rewrite no      # Maintenir durabilité TOUJOURS
```

**Justification :**
- Aucune perte de données tolérée
- Performance secondaire
- Conformité réglementaire

**+ Architecture recommandée :**
- Plusieurs instances Redis (scaling horizontal)
- Réplication synchrone
- Backups fréquents

## Monitoring et métriques AOF

### Commandes de diagnostic

```bash
# État général AOF
redis-cli INFO persistence

# Métriques clés
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:5
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:104857600        # Taille actuelle (bytes)
aof_base_size:52428800            # Taille après dernier rewrite
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0               # Nombre de fsync retardés
```

### Métriques critiques à surveiller

| Métrique | Source | Alerte si | Priorité |
|----------|--------|-----------|----------|
| `aof_last_write_status` | INFO persistence | `!= ok` | 🔴 Critique |
| `aof_last_bgrewrite_status` | INFO persistence | `!= ok` | 🔴 Critique |
| `aof_current_size` | INFO persistence | Croissance >50%/jour | 🟡 Warning |
| `aof_delayed_fsync` | INFO persistence | `> 0` (et croissant) | 🟡 Warning |
| `aof_rewrite_in_progress` | INFO persistence | Bloqué depuis >10min | 🟡 Warning |
| Ratio `aof_current_size` / `aof_base_size` | Calculé | `> 3` | 🟢 Info |

### Dashboard Prometheus/Grafana

**Requêtes PromQL utiles :**

```promql
# Taille AOF
redis_aof_current_size_bytes

# Ratio de croissance AOF
redis_aof_current_size_bytes / redis_aof_base_size_bytes

# Nombre de réécritures échouées
increase(redis_aof_last_bgrewrite_status_total{status="err"}[5m])

# Fsync retardés (problème I/O)
redis_aof_delayed_fsync

# Durée des réécritures
redis_aof_last_rewrite_time_seconds
```

### Alertes recommandées

```yaml
# Alert Manager (Prometheus)
groups:
  - name: redis_aof_alerts
    rules:
      - alert: RedisAOFWriteFailure
        expr: redis_aof_last_write_status != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis AOF write failure"

      - alert: RedisAOFGrowthAbnormal
        expr: redis_aof_current_size_bytes / redis_aof_base_size_bytes > 5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis AOF growing abnormally"

      - alert: RedisAOFDelayedFsync
        expr: increase(redis_aof_delayed_fsync[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis experiencing delayed fsyncs (I/O issue)"
```

## Troubleshooting AOF

### Problème 1 : AOF qui grandit trop vite

**Symptôme :**
```bash
redis-cli INFO persistence | grep aof_current_size
aof_current_size:10737418240  # 10 GB !
# Ratio > 10x la base
```

**Causes possibles :**
- Réécritures automatiques désactivées
- Seuil de réécriture trop élevé
- Write-heavy workload
- Réécritures qui échouent

**Solutions :**

1. **Forcer une réécriture manuelle :**
```bash
redis-cli BGREWRITEAOF
```

2. **Ajuster la configuration :**
```conf
# Réécritures plus agressives
auto-aof-rewrite-percentage 50
auto-aof-rewrite-min-size 32mb
```

3. **Vérifier les échecs de réécriture :**
```bash
redis-cli INFO persistence | grep aof_last_bgrewrite_status
# Si "err", investiguer les logs
tail -f /var/log/redis/redis-server.log
```

### Problème 2 : "Delayed fsync"

**Symptôme :**
```bash
redis-cli INFO persistence | grep aof_delayed_fsync
aof_delayed_fsync:47  # Nombre croissant !
```

**Signification :** Le fsync prend plus d'1 seconde (disque saturé ou lent).

**Impact :**
- Augmentation de la latence
- Risque de perte de données >1 seconde
- Potentiel blocage du thread principal

**Diagnostic :**
```bash
# Vérifier les I/O disque
iostat -x 1 10

Device  r/s   w/s  rMB/s  wMB/s  %util
sda     10    250   0.5    125    98%   ← Disque saturé !
```

**Solutions :**

1. **Migration vers SSD :**
```bash
# Impact : 5-10x plus rapide
```

2. **Réduire la charge d'écriture :**
```conf
# Moins de fsync (mais moins de durabilité)
appendfsync no  # NON recommandé
# OU
no-appendfsync-on-rewrite yes  # Acceptable avec réplication
```

3. **Scaling horizontal :**
```
1 Redis → 3 Redis (sharding)
Charge d'écriture divisée par 3
```

### Problème 3 : AOF corrompu au démarrage

**Symptôme :**
```bash
redis-server
# Bad file format reading the append only file: make a backup of your AOF file
# then use ./redis-check-aof --fix <filename>
```

**Solution :**

1. **Backup du fichier :**
```bash
cp appendonly.aof appendonly.aof.broken
```

2. **Réparer avec redis-check-aof :**
```bash
redis-check-aof --fix appendonly.aof

# Sortie :
AOF analyzed: size=1048576, ok_up_to=1000000, diff=48576
This will shrink the AOF from 1048576 bytes, with 48576 bytes to trash.
Continue? [y/N]: y
Successfully truncated AOF
```

3. **Redémarrer Redis :**
```bash
systemctl start redis
```

**Prévention :**
```conf
# Tolérer AOF tronqué (charge malgré corruption en fin de fichier)
aof-load-truncated yes
```

### Problème 4 : Démarrage très lent

**Symptôme :**
```bash
systemctl start redis
# Redis prend 10+ minutes à démarrer
```

**Cause :** Fichier AOF très gros (>10 GB) ou format AOF pur (non-hybride).

**Solutions :**

1. **Activer le format hybride :**
```conf
aof-use-rdb-preamble yes

# Forcer une réécriture pour convertir
redis-cli BGREWRITEAOF
```

**Impact :**
```
AOF pur 10 GB : 10 minutes de chargement
AOF hybride 3 GB : 45 secondes de chargement
```

2. **Réduire la taille AOF :**
```bash
# Réécritures plus fréquentes
auto-aof-rewrite-percentage 50
```

3. **Temporaire : charger depuis RDB :**
```bash
# Désactiver AOF temporairement pour démarrer vite
redis-server --appendonly no

# Une fois démarré, réactiver
redis-cli CONFIG SET appendonly yes
redis-cli BGREWRITEAOF
```

### Problème 5 : Écriture AOF échouée (disque plein)

**Symptôme :**
```bash
redis-cli SET key value
(error) MISCONF Redis is configured to save AOF, but it is currently not able
to persist on disk.
```

**Cause :** Disque plein.

**Diagnostic :**
```bash
df -h /var/lib/redis
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       50G   50G     0 100% /var    ← Disque plein !
```

**Solutions immédiates :**

1. **Libérer de l'espace :**
```bash
# Supprimer anciens logs
rm -f /var/log/redis/*.log.1*

# Supprimer anciens backups
rm -f /backup/redis/dump-*.rdb.gz
```

2. **Désactiver temporairement AOF :**
```bash
# DANGER : perte de durabilité
redis-cli CONFIG SET appendonly no

# Redis repasse en écriture
redis-cli SET key value
OK
```

3. **Solution permanente :**
```bash
# Augmenter le disque OU
# Déplacer Redis sur un volume plus grand
rsync -av /var/lib/redis/ /mnt/bigdisk/redis/
```

## Stratégies de backup AOF

### Backup du fichier AOF

```bash
#!/bin/bash
# Backup AOF (simple)

REDIS_DIR="/var/lib/redis"
BACKUP_DIR="/backup/redis"
DATE=$(date +%Y%m%d_%H%M%S)

# AOF unique (Redis 6)
if [ -f "$REDIS_DIR/appendonly.aof" ]; then
    cp $REDIS_DIR/appendonly.aof $BACKUP_DIR/aof-$DATE.aof
    gzip $BACKUP_DIR/aof-$DATE.aof
fi

# AOF multi-part (Redis 7+)
if [ -d "$REDIS_DIR/appendonlydir" ]; then
    tar czf $BACKUP_DIR/aof-$DATE.tar.gz -C $REDIS_DIR appendonlydir/
fi

# Nettoyage (garder 7 jours)
find $BACKUP_DIR -name "aof-*.gz" -mtime +7 -delete
```

### Combinaison RDB + AOF pour backups

**Stratégie recommandée :**
```bash
#!/bin/bash
# Backup combiné (OPTIMAL)

# 1. Déclencher un snapshot RDB (compact)
redis-cli BGSAVE
while [ $(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2) -eq 1 ]; do
    sleep 1
done

# 2. Copier le RDB (pour backup rapide)
cp /var/lib/redis/dump.rdb /backup/dump-$DATE.rdb

# 3. Copier l'AOF (pour durabilité maximale)
cp /var/lib/redis/appendonly.aof /backup/aof-$DATE.aof

# Restauration : Utiliser RDB (rapide) ou AOF (plus récent)
```

**Avantage :** Double sécurité (RDB rapide + AOF complet).

## Checklist de production AOF

### Configuration optimale

```conf
# === AOF ACTIVÉ ===
appendonly yes
appendfsync everysec              # 90% des cas
aof-use-rdb-preamble yes         # Redis 7+ obligatoire
appenddirname "appendonlydir"

# === RÉÉCRITURE ===
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
no-appendfsync-on-rewrite no

# === ROBUSTESSE ===
aof-load-truncated yes           # Tolérer corruption légère
aof-timestamp-enabled yes        # Redis 7.0+

# === RÉPERTOIRE ===
dir /var/lib/redis               # SSD fortement recommandé
```

### Checklist de déploiement

#### Configuration
- [ ] `appendonly yes` activé
- [ ] `appendfsync everysec` (ou `always` si critique)
- [ ] `aof-use-rdb-preamble yes` (Redis 7+)
- [ ] Réécritures automatiques configurées
- [ ] Répertoire de travail sur SSD

#### Infrastructure
- [ ] Espace disque >= 5x dataset
- [ ] SSD (ou NVMe pour haute performance)
- [ ] I/O monitoring activé (iostat, Prometheus)
- [ ] Alertes sur fsync retardés

#### Monitoring
- [ ] Métriques AOF dans Prometheus/Grafana
- [ ] Alertes sur `aof_last_write_status`
- [ ] Alertes sur croissance anormale AOF
- [ ] Alertes sur `aof_delayed_fsync`

#### Backups
- [ ] Script de backup automatisé (AOF + RDB)
- [ ] Backups stockés hors-site
- [ ] Rétention définie (7j/4w/6m)
- [ ] Procédure de restauration testée

#### Tests
- [ ] Test de redémarrage après crash simulé
- [ ] Test de réparation AOF corrompu
- [ ] Test de restauration depuis backup
- [ ] Mesure du temps de chargement AOF

## Conclusion

AOF est le mécanisme de persistance privilégiant la **durabilité maximale** dans Redis. Ses caractéristiques principales :

### Points forts
- ✅ **Durabilité excellente** : Perte de 0 à 1 seconde maximum
- ✅ **Audit trail complet** : Toutes les opérations journalisées
- ✅ **Réparabilité** : Corruption récupérable avec redis-check-aof
- ✅ **Format lisible** : Debugging et analyse facilités

### Points faibles
- ❌ **Taille importante** : 3-5x plus gros que RDB (avant réécriture)
- ❌ **Performance impactée** : -20 à -30% avec everysec
- ❌ **Chargement plus lent** : Peut prendre plusieurs minutes
- ❌ **Overhead I/O** : Utilisation disque continue

### Recommandations finales

| Cas d'usage | Configuration AOF |
|-------------|-------------------|
| **Cache pur** | ❌ Désactivé (ou RDB seul) |
| **Session store** | ✅ everysec + format hybride |
| **Job queues** | ✅ everysec + format hybride |
| **Données critiques** | ✅ everysec + format hybride + Réplication |
| **Données financières** | ✅ always + Réplication synchrone |

**Configuration production standard (90% des cas) :**
```conf
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Règle d'or** : AOF everysec + format hybride + RDB (combinaison gagnante) offre le meilleur équilibre durabilité/performance/simplicité pour la majorité des applications en production.

---

**Points clés à retenir :**
- `appendfsync everysec` est le compromis optimal (99% des cas)
- Format hybride (Redis 7+) réduit drastiquement taille et temps de chargement
- Réécritures automatiques essentielles pour contrôler la taille AOF
- Toujours tester les procédures de restauration
- SSD fortement recommandé pour workload write-heavy

---


⏭️ [Comparaison RDB vs AOF : Avantages et inconvénients](/05-persistance-fiabilite/04-comparaison-rdb-vs-aof.md)
