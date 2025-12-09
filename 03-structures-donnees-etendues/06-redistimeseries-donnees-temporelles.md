🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.6 RedisTimeSeries : Gestion de données temporelles (IoT, monitoring)

## Introduction

Les **séries temporelles** (time-series) sont des données indexées chronologiquement : métriques système, capteurs IoT, prix financiers, logs applicatifs, etc. Ces données ont des caractéristiques spécifiques :

- 📈 **Insertions séquentielles** : Données toujours ajoutées dans l'ordre chronologique
- 🔍 **Requêtes par plage** : "Température entre 10h et 12h"
- 📊 **Agrégations** : Moyennes, min/max, sommes sur des intervalles
- 🗜️ **Compaction** : Réduire la granularité des données anciennes
- 📉 **Rétention** : Supprimer automatiquement les données trop anciennes

**RedisTimeSeries** est un module Redis optimisé pour gérer ces données avec :
- ✅ **Ingestion ultra-rapide** : 100K+ writes/sec par série
- ✅ **Agrégations automatiques** : Downsampling en temps réel
- ✅ **Requêtes efficaces** : Latence < 1ms pour les requêtes
- ✅ **Compression** : Réduction de 90% de la mémoire vs stockage naïf
- ✅ **Labels** : Métadonnées pour filtrer et grouper les séries

---

## Pourquoi RedisTimeSeries ?

### Le problème avec Redis Core

**Avec Sorted Sets** (approche classique) :

```bash
# Stocker des mesures de température avec Sorted Sets
ZADD temp:sensor1 1702123200 "22.5"
ZADD temp:sensor1 1702123260 "23.1"
ZADD temp:sensor1 1702123320 "22.8"

# Problèmes :
# ❌ Pas d'agrégations natives (moyenne, min, max)
# ❌ Pas de compaction automatique
# ❌ Pas de gestion de rétention
# ❌ Stockage inefficace (pas de compression)
# ❌ Requêtes complexes nécessitent du code applicatif
```

**Avec RedisTimeSeries** :

```bash
# Créer une série temporelle
TS.CREATE temp:sensor1 RETENTION 86400000 LABELS sensor_id 1 type temperature

# Ajouter des mesures
TS.ADD temp:sensor1 * 22.5
TS.ADD temp:sensor1 * 23.1
TS.ADD temp:sensor1 * 22.8

# Requête avec agrégation automatique
TS.RANGE temp:sensor1 1702123200000 1702209600000 AGGREGATION avg 60000
# Moyenne par minute sur 24h

# Avantages :
# ✅ Agrégations natives
# ✅ Compaction automatique
# ✅ Rétention automatique
# ✅ Compression optimisée (~90% de réduction)
# ✅ API simple et intuitive
```

---

## Installation et vérification

### Vérifier que RedisTimeSeries est disponible

```bash
# Vérifier les modules chargés
redis-cli MODULE LIST

# Devrait contenir :
# 1) 1) "name"
#    2) "timeseries"
#    3) "ver"
#    4) 10812  # Version 1.8.12
```

### Avec Docker (Redis Stack)

```bash
# Démarrer Redis Stack avec RedisTimeSeries
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest

# Tester RedisTimeSeries
redis-cli TS.CREATE test RETENTION 60000
redis-cli TS.ADD test * 42
redis-cli TS.RANGE test - +
# OK : RedisTimeSeries fonctionne
```

---

## Concepts fondamentaux

### Timestamps

Les timestamps sont en **millisecondes** depuis l'epoch Unix.

```bash
# Timestamp explicite (millisecondes)
TS.ADD sensor:1 1702123200000 25.5

# Timestamp automatique (timestamp actuel)
TS.ADD sensor:1 * 25.5
```

**Conversion** :

```python
import time

# Obtenir le timestamp actuel en millisecondes
timestamp_ms = int(time.time() * 1000)

# Convertir une date en timestamp
from datetime import datetime
dt = datetime(2024, 12, 9, 10, 0, 0)
timestamp_ms = int(dt.timestamp() * 1000)
```

---

### Labels (métadonnées)

Les labels permettent de filtrer et grouper les séries.

```bash
# Créer des séries avec labels
TS.CREATE temp:sensor1 LABELS sensor_id 1 location datacenter-a type temperature
TS.CREATE temp:sensor2 LABELS sensor_id 2 location datacenter-a type temperature
TS.CREATE temp:sensor3 LABELS sensor_id 3 location datacenter-b type temperature

# Requête multi-séries par labels
TS.MRANGE - + FILTER location=datacenter-a
# Retourne temp:sensor1 et temp:sensor2

TS.MRANGE - + FILTER type=temperature location=datacenter-b
# Retourne temp:sensor3
```

---

### Rétention (RETENTION)

Définit combien de temps conserver les données (en millisecondes).

```bash
# Rétention de 24 heures
TS.CREATE sensor:1 RETENTION 86400000

# Rétention de 7 jours
TS.CREATE sensor:2 RETENTION 604800000

# Rétention infinie (0 ou omis)
TS.CREATE sensor:3
```

**Suppression automatique** : Les données plus anciennes que la rétention sont supprimées automatiquement.

---

### Politique de duplication (DUPLICATE_POLICY)

Que faire si on insère deux fois la même valeur au même timestamp ?

```bash
# BLOCK : Erreur (défaut)
TS.CREATE sensor:1 DUPLICATE_POLICY BLOCK

# LAST : Garder la dernière valeur
TS.CREATE sensor:2 DUPLICATE_POLICY LAST

# FIRST : Garder la première valeur
TS.CREATE sensor:3 DUPLICATE_POLICY FIRST

# MIN : Garder la valeur minimale
TS.CREATE sensor:4 DUPLICATE_POLICY MIN

# MAX : Garder la valeur maximale
TS.CREATE sensor:5 DUPLICATE_POLICY MAX

# SUM : Additionner les valeurs
TS.CREATE sensor:6 DUPLICATE_POLICY SUM
```

**Cas d'usage** :
- `LAST` : Capteurs (garder la mesure la plus récente)
- `SUM` : Compteurs (additionner les événements)
- `MAX` : Pics de trafic (conserver le maximum)

---

## Commandes principales

### TS.CREATE : Créer une série temporelle

```bash
# Syntaxe complète
TS.CREATE key
  [RETENTION retention_ms]
  [ENCODING {COMPRESSED|UNCOMPRESSED}]
  [CHUNK_SIZE size]
  [DUPLICATE_POLICY {BLOCK|FIRST|LAST|MIN|MAX|SUM}]
  [LABELS label value...]
```

**Exemples** :

```bash
# Série simple
TS.CREATE temperature

# Avec rétention et labels
TS.CREATE temp:cpu1
  RETENTION 3600000
  DUPLICATE_POLICY LAST
  LABELS server web-01 metric cpu_usage

# Avec compression et chunk size
TS.CREATE metrics:requests
  RETENTION 604800000
  ENCODING COMPRESSED
  CHUNK_SIZE 4096
  LABELS service api endpoint /users
```

---

### TS.ADD : Ajouter une mesure

```bash
# Avec timestamp automatique
TS.ADD temp:sensor1 * 25.5
# Retourne le timestamp utilisé : (integer) 1702123456789

# Avec timestamp explicite
TS.ADD temp:sensor1 1702123200000 24.3

# Créer la série à la volée (ON_DUPLICATE)
TS.ADD temp:sensor2 * 26.1 RETENTION 86400000 LABELS type temperature

# Avec politique de duplication
TS.ADD counter:requests * 1 ON_DUPLICATE SUM
```

---

### TS.MADD : Ajouter plusieurs mesures

```bash
# Ajouter à plusieurs séries en une seule commande
TS.MADD
  temp:sensor1 * 25.5
  temp:sensor2 * 26.1
  temp:sensor3 * 24.8

# Retourne les timestamps pour chaque série
# 1) (integer) 1702123456789
# 2) (integer) 1702123456790
# 3) (integer) 1702123456791
```

**Performance** : Plus efficace que plusieurs `TS.ADD` séparés.

---

### TS.INCRBY / TS.DECRBY : Incrémenter/Décrémenter

```bash
# Créer un compteur
TS.CREATE counter:requests DUPLICATE_POLICY SUM

# Incrémenter
TS.INCRBY counter:requests 1
TS.INCRBY counter:requests 5
TS.INCRBY counter:requests 3

# Décrémenter
TS.DECRBY counter:requests 2

# Avec timestamp explicite
TS.INCRBY counter:requests 1 TIMESTAMP 1702123200000
```

---

### TS.RANGE : Requête sur une série

```bash
# Syntaxe
TS.RANGE key from_timestamp to_timestamp
  [LATEST]
  [FILTER_BY_TS ts...]
  [FILTER_BY_VALUE min max]
  [COUNT count]
  [AGGREGATION aggregator bucket_duration]
  [ALIGN align]
  [BUCKETTIMESTAMP bt]
```

**Exemples** :

```bash
# Toutes les données
TS.RANGE temp:sensor1 - +

# Plage spécifique (timestamps en millisecondes)
TS.RANGE temp:sensor1 1702123200000 1702209600000

# Avec agrégation (moyenne par heure)
TS.RANGE temp:sensor1 1702123200000 1702209600000
  AGGREGATION avg 3600000

# Limiter le nombre de résultats
TS.RANGE temp:sensor1 - + COUNT 100

# Filtrer par valeur (températures entre 20 et 30)
TS.RANGE temp:sensor1 - + FILTER_BY_VALUE 20 30
```

**Résultat** :

```
1) 1) (integer) 1702123200000
   2) "25.5"
2) 1) (integer) 1702126800000
   2) "26.1"
3) 1) (integer) 1702130400000
   2) "24.8"
```

---

### TS.REVRANGE : Requête en ordre inversé

```bash
# Les 10 dernières mesures
TS.REVRANGE temp:sensor1 - + COUNT 10
```

---

### TS.MRANGE : Requête multi-séries

```bash
# Toutes les séries avec un label spécifique
TS.MRANGE - + FILTER type=temperature

# Avec agrégation
TS.MRANGE 1702123200000 1702209600000
  AGGREGATION avg 3600000
  FILTER location=datacenter-a

# Grouper par label
TS.MRANGE - +
  FILTER type=temperature
  GROUPBY location
  REDUCE avg
```

**Résultat** :

```
1) 1) "temp:sensor1"
   2) 1) 1) "sensor_id"
         2) "1"
      2) 1) "location"
         2) "datacenter-a"
   3) 1) 1) (integer) 1702123200000
         2) "25.5"
      2) 1) (integer) 1702126800000
         2) "26.1"
2) 1) "temp:sensor2"
   2) 1) 1) "sensor_id"
         2) "2"
      2) 1) "location"
         2) "datacenter-a"
   3) 1) 1) (integer) 1702123200000
         2) "24.8"
```

---

### TS.GET : Dernière valeur

```bash
# Obtenir la dernière mesure
TS.GET temp:sensor1

# Résultat :
# 1) (integer) 1702123456789  # Timestamp
# 2) "25.5"                    # Valeur
```

---

### TS.MGET : Dernières valeurs de plusieurs séries

```bash
# Dernière valeur de toutes les séries de type temperature
TS.MGET LATEST FILTER type=temperature

# Résultat :
# 1) 1) "temp:sensor1"
#    2) 1) 1) "sensor_id"
#          2) "1"
#    3) 1) (integer) 1702123456789
#       2) "25.5"
# 2) 1) "temp:sensor2"
#    2) 1) 1) "sensor_id"
#          2) "2"
#    3) 1) (integer) 1702123456790
#       2) "26.1"
```

---

### TS.INFO : Informations sur une série

```bash
TS.INFO temp:sensor1

# Résultat :
# 1) "totalSamples"
# 2) (integer) 1440  # Nombre de samples
# 3) "memoryUsage"
# 4) (integer) 4184  # Mémoire utilisée (bytes)
# 5) "firstTimestamp"
# 6) (integer) 1702123200000
# 7) "lastTimestamp"
# 8) (integer) 1702209600000
# 9) "retentionTime"
# 10) (integer) 86400000
# 11) "chunkCount"
# 12) (integer) 1
# 13) "labels"
# 14) 1) 1) "sensor_id"
#        2) "1"
#     2) 1) "type"
#        2) "temperature"
```

---

## Agrégations automatiques : TS.CREATERULE

### Concept de downsampling

**Problème** : Conserver des données haute résolution (1 seconde) sur 1 an = énorme volume.

**Solution** : Créer des agrégations automatiques :
- Résolution 1s → Conserver 24h
- Résolution 1min (moyenne sur 1 min) → Conserver 7 jours
- Résolution 1h (moyenne sur 1h) → Conserver 1 an

---

### Créer une règle d'agrégation

```bash
# Série source (haute résolution)
TS.CREATE temp:sensor1:raw
  RETENTION 86400000
  LABELS sensor_id 1 resolution raw

# Série agrégée (moyenne sur 1 minute)
TS.CREATE temp:sensor1:1min
  RETENTION 604800000
  LABELS sensor_id 1 resolution 1min

# Créer la règle d'agrégation
TS.CREATERULE temp:sensor1:raw temp:sensor1:1min
  AGGREGATION avg 60000

# Série agrégée (moyenne sur 1 heure)
TS.CREATE temp:sensor1:1hour
  RETENTION 31536000000
  LABELS sensor_id 1 resolution 1hour

TS.CREATERULE temp:sensor1:raw temp:sensor1:1hour
  AGGREGATION avg 3600000
```

**Fonctionnement** :
- Chaque fois qu'une valeur est ajoutée à `temp:sensor1:raw`, RedisTimeSeries calcule automatiquement les agrégations
- Les séries agrégées sont mises à jour en temps réel
- Pas besoin de code applicatif pour gérer le downsampling

---

### Fonctions d'agrégation disponibles

```bash
# Moyenne
TS.CREATERULE source dest AGGREGATION avg 60000

# Somme
TS.CREATERULE source dest AGGREGATION sum 60000

# Min / Max
TS.CREATERULE source dest AGGREGATION min 60000
TS.CREATERULE source dest AGGREGATION max 60000

# Premier / Dernier
TS.CREATERULE source dest AGGREGATION first 60000
TS.CREATERULE source dest AGGREGATION last 60000

# Range (max - min)
TS.CREATERULE source dest AGGREGATION range 60000

# Comptage
TS.CREATERULE source dest AGGREGATION count 60000

# Écart-type
TS.CREATERULE source dest AGGREGATION std.p 60000
TS.CREATERULE source dest AGGREGATION std.s 60000

# Variance
TS.CREATERULE source dest AGGREGATION var.p 60000
TS.CREATERULE source dest AGGREGATION var.s 60000
```

---

### Lister les règles

```bash
# Lister les règles d'agrégation d'une série
TS.INFO temp:sensor1:raw

# Résultat inclut :
# "rules"
# 1) 1) "temp:sensor1:1min"
#    2) (integer) 60000
#    3) "avg"
# 2) 1) "temp:sensor1:1hour"
#    2) (integer) 3600000
#    3) "avg"
```

---

### Supprimer une règle

```bash
TS.DELETERULE temp:sensor1:raw temp:sensor1:1min
```

---

## Cas d'usage modernes

### 1️⃣ Monitoring système (CPU, RAM, disque)

**Contexte** : Surveiller les métriques système de plusieurs serveurs

```bash
# Créer les séries pour chaque serveur
TS.CREATE metrics:cpu:server01
  RETENTION 86400000
  DUPLICATE_POLICY LAST
  LABELS server server01 metric cpu_usage datacenter paris

TS.CREATE metrics:cpu:server02
  RETENTION 86400000
  DUPLICATE_POLICY LAST
  LABELS server server02 metric cpu_usage datacenter paris

TS.CREATE metrics:memory:server01
  RETENTION 86400000
  DUPLICATE_POLICY LAST
  LABELS server server01 metric memory_usage datacenter paris

# Créer des agrégations (moyenne sur 5 minutes)
TS.CREATE metrics:cpu:server01:5min RETENTION 604800000
TS.CREATERULE metrics:cpu:server01 metrics:cpu:server01:5min AGGREGATION avg 300000

# Envoyer des métriques (toutes les 10 secondes)
TS.ADD metrics:cpu:server01 * 23.5
TS.ADD metrics:memory:server01 * 45.2

# Requête : CPU moyen sur la dernière heure
TS.RANGE metrics:cpu:server01 1702120000000 1702123600000 AGGREGATION avg 60000

# Requête multi-serveurs : CPU de tous les serveurs à Paris
TS.MRANGE - + FILTER metric=cpu_usage datacenter=paris

# Dashboard : Dernière valeur de toutes les métriques
TS.MGET LATEST FILTER datacenter=paris
```

---

### 2️⃣ IoT : Capteurs de température et humidité

**Contexte** : Réseau de capteurs dans plusieurs bâtiments

```bash
# Créer les séries pour chaque capteur
TS.CREATE sensor:temp:building_a:room_101
  RETENTION 2592000000  # 30 jours
  LABELS building building_a room room_101 type temperature

TS.CREATE sensor:humidity:building_a:room_101
  RETENTION 2592000000
  LABELS building building_a room room_101 type humidity

TS.CREATE sensor:temp:building_b:room_201
  RETENTION 2592000000
  LABELS building building_b room room_201 type temperature

# Agrégations : Moyenne par heure (conservation 1 an)
TS.CREATE sensor:temp:building_a:room_101:hourly RETENTION 31536000000
TS.CREATERULE sensor:temp:building_a:room_101 sensor:temp:building_a:room_101:hourly
  AGGREGATION avg 3600000

# Ingestion des mesures (toutes les minutes)
TS.ADD sensor:temp:building_a:room_101 * 22.5
TS.ADD sensor:humidity:building_a:room_101 * 65.2

# Analyse : Température moyenne par bâtiment
TS.MRANGE 1702123200000 1702209600000
  FILTER type=temperature
  GROUPBY building
  REDUCE avg

# Alerte : Températures anormales (> 30°C)
TS.MRANGE - +
  FILTER type=temperature
  FILTER_BY_VALUE 30 +inf
```

---

### 3️⃣ Application web : Métriques de trafic

**Contexte** : Tracker les requêtes HTTP, latences, erreurs

```bash
# Séries pour les compteurs de requêtes
TS.CREATE metrics:requests:total
  RETENTION 604800000
  DUPLICATE_POLICY SUM
  LABELS service api endpoint all

TS.CREATE metrics:requests:endpoint_users
  RETENTION 604800000
  DUPLICATE_POLICY SUM
  LABELS service api endpoint /users

TS.CREATE metrics:requests:endpoint_orders
  RETENTION 604800000
  DUPLICATE_POLICY SUM
  LABELS service api endpoint /orders

# Séries pour les latences
TS.CREATE metrics:latency:p50
  RETENTION 604800000
  LABELS service api metric latency_p50

TS.CREATE metrics:latency:p99
  RETENTION 604800000
  LABELS service api metric latency_p99

# Agrégations : Requêtes par minute (conservation 30 jours)
TS.CREATE metrics:requests:total:1min RETENTION 2592000000
TS.CREATERULE metrics:requests:total metrics:requests:total:1min
  AGGREGATION sum 60000

# Ingestion (chaque requête)
TS.INCRBY metrics:requests:total 1
TS.INCRBY metrics:requests:endpoint_users 1
TS.ADD metrics:latency:p50 * 45  # 45ms

# Dashboard : Requêtes par seconde (RPS)
TS.RANGE metrics:requests:total:1min - + AGGREGATION sum 1000

# Analyse : Quel endpoint a le plus de trafic ?
TS.MGET LATEST FILTER service=api
```

---

### 4️⃣ Finance : Prix des actifs

**Contexte** : Stocker les prix de cryptomonnaies / actions

```bash
# Créer les séries pour chaque actif
TS.CREATE price:btc:usd
  RETENTION 31536000000  # 1 an
  DUPLICATE_POLICY LAST
  LABELS asset BTC pair BTC/USD

TS.CREATE price:eth:usd
  RETENTION 31536000000
  DUPLICATE_POLICY LAST
  LABELS asset ETH pair ETH/USD

# Agrégations : OHLC (Open, High, Low, Close)
TS.CREATE price:btc:usd:1min:open RETENTION 2592000000
TS.CREATERULE price:btc:usd price:btc:usd:1min:open AGGREGATION first 60000

TS.CREATE price:btc:usd:1min:high RETENTION 2592000000
TS.CREATERULE price:btc:usd price:btc:usd:1min:high AGGREGATION max 60000

TS.CREATE price:btc:usd:1min:low RETENTION 2592000000
TS.CREATERULE price:btc:usd price:btc:usd:1min:low AGGREGATION min 60000

TS.CREATE price:btc:usd:1min:close RETENTION 2592000000
TS.CREATERULE price:btc:usd price:btc:usd:1min:close AGGREGATION last 60000

# Ingestion (tick par tick)
TS.ADD price:btc:usd * 42350.50
TS.ADD price:btc:usd * 42360.20
TS.ADD price:btc:usd * 42345.80

# Requête : Prix BTC sur la dernière heure avec candlesticks (1 min)
# Récupérer OHLC
TS.MRANGE 1702120000000 1702123600000
  FILTER asset=BTC pair=BTC/USD

# Analyse : Volatilité (range = high - low)
TS.RANGE price:btc:usd 1702123200000 1702209600000 AGGREGATION range 60000
```

---

### 5️⃣ Gaming : Analytics de joueurs en temps réel

**Contexte** : Tracker les connexions, sessions, achats

```bash
# Séries pour les métriques de jeu
TS.CREATE game:players:online
  RETENTION 604800000
  DUPLICATE_POLICY LAST
  LABELS game mmorpg metric players_online

TS.CREATE game:revenue:daily
  RETENTION 31536000000
  DUPLICATE_POLICY SUM
  LABELS game mmorpg metric revenue

TS.CREATE game:sessions:started
  RETENTION 604800000
  DUPLICATE_POLICY SUM
  LABELS game mmorpg metric sessions_started

# Agrégations : Joueurs moyens par heure
TS.CREATE game:players:online:hourly RETENTION 2592000000
TS.CREATERULE game:players:online game:players:online:hourly AGGREGATION avg 3600000

# Ingestion
TS.ADD game:players:online * 15423  # 15,423 joueurs en ligne
TS.INCRBY game:revenue:daily 9.99   # Achat in-game
TS.INCRBY game:sessions:started 1   # Nouvelle session

# Dashboard : Joueurs en ligne (temps réel)
TS.GET game:players:online

# Analytics : Revenu total du mois
TS.RANGE game:revenue:daily 1704067200000 1706745600000 AGGREGATION sum 86400000

# Analyse : Peak d'utilisateurs (max par jour)
TS.RANGE game:players:online 1702123200000 1702209600000 AGGREGATION max 86400000
```

---

## Compression et optimisation

### Compression automatique

RedisTimeSeries utilise **Gorilla compression** (algorithme Facebook) optimisé pour les time-series.

```bash
# Exemple de compression

# Sans compression (stockage naïf) :
# 1440 samples (1 par minute sur 24h)
# Taille : 1440 × 16 bytes = 23 KB

# Avec compression RedisTimeSeries :
# Taille : ~2-3 KB
# Ratio de compression : 90%
```

**Activation** :

```bash
# Compression activée par défaut
TS.CREATE sensor:1 ENCODING COMPRESSED

# Désactiver la compression (plus rapide, mais plus de mémoire)
TS.CREATE sensor:2 ENCODING UNCOMPRESSED
```

**Conseil** : Garder la compression activée sauf si vous avez des contraintes de latence extrêmes (< 100µs).

---

### Chunk Size

RedisTimeSeries stocke les données par chunks (blocs).

```bash
# Chunk size par défaut : 4096 bytes
TS.CREATE sensor:1

# Chunk size personnalisé
TS.CREATE sensor:2 CHUNK_SIZE 8192

# Plus petit chunk : Moins de mémoire, plus de chunks
TS.CREATE sensor:3 CHUNK_SIZE 2048

# Plus grand chunk : Plus de mémoire, moins de chunks
TS.CREATE sensor:4 CHUNK_SIZE 16384
```

**Trade-off** :
- Petit chunk (2048) : Utilise moins de mémoire, mais plus d'overhead
- Grand chunk (8192+) : Utilise plus de mémoire, mais moins d'overhead

**Recommandation** : Garder le défaut (4096) sauf cas spécifiques.

---

## Performance et benchmarks

### Ingestion

```bash
# Benchmark : Ingestion continue

# 1 série, 1 mesure/seconde
# Throughput : ~100K writes/sec
# Latence moyenne : < 0.1ms

# 1000 séries, 1 mesure/seconde chacune
# Throughput total : ~100K writes/sec (limité par single-thread Redis)
# Latence moyenne : < 0.5ms
```

**Optimisation** : Utiliser `TS.MADD` pour insérer dans plusieurs séries en une commande.

```bash
# ✅ Bon : TS.MADD (1 round-trip)
TS.MADD sensor:1 * 25.5 sensor:2 * 26.1 sensor:3 * 24.8

# ❌ Moins efficace : 3 TS.ADD (3 round-trips)
TS.ADD sensor:1 * 25.5
TS.ADD sensor:2 * 26.1
TS.ADD sensor:3 * 24.8
```

---

### Requêtes

```bash
# Benchmark : Requêtes

# TS.RANGE sur 1 série (1h de données, 1 sample/sec)
# Latence : < 1ms

# TS.RANGE avec agrégation (1 jour de données, avg par heure)
# Latence : 2-5ms

# TS.MRANGE sur 100 séries (1h de données)
# Latence : 10-20ms

# TS.MGET sur 1000 séries
# Latence : 5-10ms
```

---

### Impact mémoire

```bash
# Exemple : 1 série, 86400 samples (1 par seconde sur 24h)

# Sans compression :
# 86400 × 16 bytes = 1.38 MB

# Avec compression RedisTimeSeries :
# ~150-200 KB (compression ~90%)

# Avec agrégations (1 raw + 2 agrégées) :
# Raw (24h, 1s) : 150 KB
# 1min (7 jours) : 168 KB
# 1h (1 an) : 140 KB
# Total : ~460 KB pour 1 an de données
```

---

## Comparaison : RedisTimeSeries vs alternatives

| Critère | RedisTimeSeries | InfluxDB | TimescaleDB | Prometheus |
|---------|-----------------|----------|-------------|------------|
| **Latence écriture** | < 0.1ms | 1-5ms | 5-20ms | 1-10ms |
| **Latence lecture** | < 1ms | 5-50ms | 10-100ms | 5-50ms |
| **Throughput** | 100K+ writes/sec | 50-100K/sec | 10-50K/sec | 50K/sec |
| **Compression** | 90% (Gorilla) | 80% | 70% | 85% |
| **Agrégations temps réel** | ✅ Natives | ✅ Continues | ⚠️ Manuelles | ✅ Recording rules |
| **Stockage** | Mémoire (+ RDB/AOF) | Disque | Disque (PostgreSQL) | Disque |
| **Requêtes complexes** | Limitées | ✅ InfluxQL | ✅ SQL complet | ✅ PromQL |
| **Cas d'usage** | Métriques temps réel | TSDB généraliste | Analytics complexes | Monitoring infra |

**Avantages RedisTimeSeries** :
- ✅ Latence ultra-faible (< 1ms)
- ✅ Throughput exceptionnel (100K+/sec)
- ✅ Agrégations automatiques
- ✅ Intégration Redis native (même infrastructure)

**Quand choisir une alternative** :
- **InfluxDB** : Si vous avez besoin d'un TSDB dédié avec InfluxQL
- **TimescaleDB** : Si vous avez besoin de requêtes SQL complexes
- **Prometheus** : Si vous faites du monitoring d'infrastructure avec scraping

---

## Bonnes pratiques

### ✅ 1. Utiliser des labels pour filtrer et grouper

```bash
# ✅ Bon : Labels riches
TS.CREATE metrics:cpu
  LABELS server web-01 datacenter paris rack 42 environment prod

# Requête facile
TS.MRANGE - + FILTER datacenter=paris environment=prod

# ❌ Mauvais : Pas de labels
TS.CREATE metrics:cpu:web01:paris:rack42:prod
# Requête difficile (KEYS + filtrage applicatif)
```

---

### ✅ 2. Créer des agrégations pour les données historiques

```bash
# ✅ Bon : Stratégie de rétention multi-niveaux
# Raw : 1s → 24h
TS.CREATE sensor:1:raw RETENTION 86400000

# 1 min : avg → 7 jours
TS.CREATE sensor:1:1min RETENTION 604800000
TS.CREATERULE sensor:1:raw sensor:1:1min AGGREGATION avg 60000

# 1h : avg → 1 an
TS.CREATE sensor:1:1hour RETENTION 31536000000
TS.CREATERULE sensor:1:raw sensor:1:1hour AGGREGATION avg 3600000

# ❌ Mauvais : Tout conserver en haute résolution
TS.CREATE sensor:1 RETENTION 31536000000  # 1 an à 1s = énorme
```

---

### ✅ 3. Utiliser TS.MADD pour l'ingestion batch

```bash
# ✅ Bon : 1 commande pour plusieurs séries
TS.MADD
  sensor:1 * 25.5
  sensor:2 * 26.1
  sensor:3 * 24.8

# ❌ Moins efficace : Plusieurs commandes
TS.ADD sensor:1 * 25.5
TS.ADD sensor:2 * 26.1
TS.ADD sensor:3 * 24.8
```

---

### ✅ 4. Choisir la bonne politique de duplication

```bash
# ✅ Bon : LAST pour les capteurs (valeur la plus récente)
TS.CREATE temp:sensor1 DUPLICATE_POLICY LAST

# ✅ Bon : SUM pour les compteurs (additionner)
TS.CREATE counter:requests DUPLICATE_POLICY SUM

# ✅ Bon : MAX pour les pics (conserver le maximum)
TS.CREATE metrics:peak_traffic DUPLICATE_POLICY MAX

# ❌ Mauvais : BLOCK (erreur si duplicate)
# Utiliser seulement si vous êtes sûr qu'il n'y aura pas de duplicatas
```

---

### ✅ 5. Monitorer l'usage mémoire

```bash
# Vérifier l'usage mémoire d'une série
TS.INFO sensor:1

# Résultat :
# "memoryUsage"
# (integer) 4184  # bytes

# Calculer l'usage total (script Lua ou application)
```

---

### ✅ 6. Utiliser des conventions de nommage

```bash
# ✅ Bon : Convention hiérarchique
metrics:cpu:server01
metrics:memory:server01
metrics:disk:server01
sensor:temp:building_a:room_101
game:players:online

# Facilite les requêtes par pattern
TS.MRANGE - + FILTER server=server01  # Toutes les métriques de server01

# ❌ Mauvais : Pas de convention
cpu_server01
mem_srv1
disk_usage_server_01
```

---

## Intégration avec les langages

### Python (redis-py)

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Créer une série
r.ts().create('sensor:1', retention_msecs=86400000, labels={'type': 'temperature'})

# Ajouter des mesures
timestamp_ms = int(time.time() * 1000)
r.ts().add('sensor:1', timestamp_ms, 25.5)

# Avec timestamp automatique
r.ts().add('sensor:1', '*', 26.1)

# Requête
result = r.ts().range('sensor:1', '-', '+')
for timestamp, value in result:
    print(f"{timestamp}: {value}")

# Avec agrégation
result = r.ts().range('sensor:1', '-', '+', aggregation_type='avg', bucket_size_msec=60000)
```

---

### Node.js (node-redis)

```javascript
import { createClient } from 'redis';

const client = await createClient().connect();

// Créer une série
await client.ts.create('sensor:1', {
  RETENTION: 86400000,
  LABELS: { type: 'temperature' }
});

// Ajouter des mesures
await client.ts.add('sensor:1', '*', 25.5);

// Requête
const result = await client.ts.range('sensor:1', '-', '+');
console.log(result);

// Avec agrégation
const aggregated = await client.ts.range('sensor:1', '-', '+', {
  AGGREGATION: {
    type: 'AVG',
    timeBucket: 60000  // 1 minute
  }
});
```

---

### Go (go-redis)

```go
package main

import (
    "context"
    "fmt"
    "time"
    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })

    // Créer une série
    rdb.Do(ctx, "TS.CREATE", "sensor:1", "RETENTION", 86400000, "LABELS", "type", "temperature")

    // Ajouter une mesure
    timestamp := time.Now().UnixMilli()
    rdb.Do(ctx, "TS.ADD", "sensor:1", timestamp, 25.5)

    // Requête
    result, _ := rdb.Do(ctx, "TS.RANGE", "sensor:1", "-", "+").Result()
    fmt.Println(result)
}
```

---

## Troubleshooting

### Erreur : "ERR TSDB: the key does not exist"

```bash
# ❌ Erreur
TS.ADD sensor:1 * 25.5
# (error) ERR TSDB: the key does not exist

# ✅ Solution : Créer la série d'abord
TS.CREATE sensor:1
TS.ADD sensor:1 * 25.5

# Ou créer à la volée
TS.ADD sensor:1 * 25.5 RETENTION 86400000 LABELS type temperature
```

---

### Erreur : "ERR TSDB: timestamp must be higher than the last timestamp"

```bash
# ❌ Erreur : Timestamp dans le passé
TS.ADD sensor:1 1702123200000 25.5
TS.ADD sensor:1 1702123100000 24.3  # Plus ancien
# (error) ERR TSDB: timestamp must be higher than the last timestamp

# ✅ Solution : Assurer que les timestamps sont croissants
# Ou utiliser DUPLICATE_POLICY pour gérer les doublons
TS.CREATE sensor:1 DUPLICATE_POLICY LAST
```

---

### Performance dégradée

```bash
# ✅ Diagnostics :

# 1. Vérifier le nombre de samples
TS.INFO sensor:1
# "totalSamples" trop élevé ? → Réduire la RETENTION

# 2. Vérifier la mémoire utilisée
TS.INFO sensor:1
# "memoryUsage" élevé ? → Activer la compression

# 3. Vérifier le nombre de séries
DBSIZE
# Trop de séries ? → Consolider ou utiliser Redis Cluster

# 4. Vérifier les agrégations
TS.INFO sensor:1:raw
# Trop de règles d'agrégation ? → Simplifier
```

---

## Résumé

**RedisTimeSeries permet de** :
- ✅ Ingestion ultra-rapide (100K+ writes/sec)
- ✅ Requêtes low-latency (< 1ms)
- ✅ Agrégations automatiques (downsampling)
- ✅ Compression efficace (90% de réduction)
- ✅ Rétention automatique des données
- ✅ Filtrage et groupement par labels

**Commandes principales** :
- `TS.CREATE` : Créer une série
- `TS.ADD` / `TS.MADD` : Insérer des mesures
- `TS.RANGE` / `TS.MRANGE` : Requêtes
- `TS.CREATERULE` : Agrégations automatiques
- `TS.GET` / `TS.MGET` : Dernières valeurs

**Cas d'usage idéaux** :
- 📊 Monitoring système (CPU, RAM, disque)
- 🌡️ IoT (capteurs, télémétrie)
- 📈 Métriques applicatives (requêtes, latences)
- 💹 Finance (prix, trading)
- 🎮 Gaming analytics (joueurs, sessions)

**Performance** :
- Latence écriture : < 0.1ms
- Latence lecture : < 1ms
- Compression : 90%
- Throughput : 100K+ writes/sec

---

**Prêt pour les filtres probabilistes ?** Passons à la section suivante : [3.7 RedisBloom - Filtres de Bloom et Cuckoo](./07-redisbloom-filtres-bloom-cuckoo.md)

⏭️ [RedisBloom : Filtres de Bloom et Cuckoo](/03-structures-donnees-etendues/07-redisbloom-filtres-bloom-cuckoo.md)
