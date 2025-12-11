🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #3 : Leaderboard de jeu vidéo temps réel

## Vue d'ensemble

**Niveau** : ⭐⭐ Intermédiaire
**Complexité technique** : Moyenne
**Impact production** : Critique (engagement utilisateur)
**Technologies** : Redis Core (Sorted Sets)

---

## 1. Contexte et problématique

### Scénario business

Un jeu vidéo compétitif multi-joueurs (type Fortnite, Apex Legends, ou League of Legends) avec les caractéristiques suivantes :

**Chiffres clés** :
- 50 millions de joueurs enregistrés
- 2 millions de joueurs actifs simultanément (peak)
- 100 000 parties terminées par minute
- 500 000 mises à jour de score par minute
- SLA : Latence < 10ms (p99) pour lecture de classement
- Disponibilité : 99.99% (critical pour engagement)

**Besoins métier** :

1. **Leaderboards multiples et hiérarchiques**
   - Global (tous les joueurs)
   - Par région (NA, EU, ASIA, etc.)
   - Par saison (saison actuelle, historiques)
   - Par mode de jeu (Solo, Duo, Squad)
   - Par niveau (Bronze, Silver, Gold, etc.)
   - Personnalisés (amis, guilde)

2. **Opérations temps réel**
   - Mise à jour du score instantanée (fin de partie)
   - Affichage du classement global en < 10ms
   - Classement du joueur ("Vous êtes 47,823ème")
   - Top N joueurs (Top 10, Top 100, Top 1000)
   - Joueurs autour d'un joueur spécifique (±10 positions)

3. **Requêtes complexes**
   - Rang d'un joueur dans plusieurs leaderboards simultanément
   - Évolution du rang sur 24h/7j
   - Comparaison avec amis
   - Détection de cheaters (progression anormale)

4. **Contraintes techniques**
   - Atomicité : Pas de race conditions sur les scores
   - Cohérence : Tous les joueurs voient le même classement
   - Performance : Sub-10ms même avec 50M joueurs
   - Coût : Infrastructure maîtrisée

### Problèmes à résoudre

#### 1. **Classement à grande échelle**

```
❌ Problème : Comment classer 50M joueurs en temps réel ?
- Tri naïf : O(N log N) = 50M × log(50M) ≈ 1.3 milliards d'opérations
- Base SQL : INDEX + ORDER BY = 5-30 secondes
- Impossible en temps réel
```

#### 2. **Mises à jour fréquentes**

```
Scénario : Joueur termine une partie
1. Score +150 points
2. Mise à jour du classement global
3. Mise à jour du classement régional
4. Mise à jour du classement de la saison
5. Mise à jour du classement par mode
6. Notification si Top 1000
7. Update stats pour amis

= 7+ opérations par fin de partie
× 100k parties/min = 700k updates/min
```

#### 3. **Tie-breaking (égalité de scores)**

```
Problème : 2 joueurs avec 1000 points
- Qui est classé devant ?
- Critères : Premier arrivé ? Timestamp ? Victoires ?

Solution classique : Score composite
score = points * 10^10 + timestamp_inversé
```

#### 4. **Hot keys (concentration du trafic)**

```
Problème : Leaderboard global = 1 seule clé Redis
- 500k writes/min sur la même clé
- Risque de saturation d'un shard
- Latence augmente avec contention
```

---

## 2. Analyse des alternatives

### Option 1 : Base de données relationnelle (PostgreSQL)

```sql
CREATE TABLE player_scores (
    player_id UUID PRIMARY KEY,
    username VARCHAR(50),
    score BIGINT NOT NULL,
    region VARCHAR(10),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_score ON player_scores(score DESC);
CREATE INDEX idx_region_score ON player_scores(region, score DESC);

-- Récupérer le classement d'un joueur
SELECT COUNT(*) + 1 AS rank
FROM player_scores
WHERE score > (SELECT score FROM player_scores WHERE player_id = 'player123');

-- Récupérer le Top 10
SELECT player_id, username, score,
       ROW_NUMBER() OVER (ORDER BY score DESC) AS rank
FROM player_scores
LIMIT 10;
```

**Avantages** :
- ✅ ACID complet
- ✅ Queries SQL puissantes
- ✅ Données persistantes

**Inconvénients** :
- ❌ Latence : 50-500ms (même avec index)
- ❌ `COUNT(*)` coûteux pour déterminer le rang
- ❌ `ROW_NUMBER()` window function = full table scan
- ❌ Write amplification : chaque update = reindex
- ❌ Scaling difficile (sharding complexe)

**Benchmark réel** (PostgreSQL 14, 50M lignes) :
```
Requête                          Latence
--------------------------------------
Top 10                           120ms
Rang d'un joueur                 350ms
Joueurs 100-110 (pagination)     200ms
Update score + reindex           50ms
```

**Verdict** : ❌ **Inadapté** pour leaderboards temps réel à grande échelle.

---

### Option 2 : MongoDB avec Aggregation

```javascript
// Collection players
db.players.createIndex({ score: -1 })

// Récupérer le rang d'un joueur
db.players.aggregate([
  { $match: { score: { $gt: playerScore } } },
  { $count: "rank" }
])

// Top 10
db.players.find().sort({ score: -1 }).limit(10)
```

**Avantages** :
- ✅ Meilleur que SQL pour agrégations
- ✅ Scaling horizontal natif (sharding)
- ✅ Schéma flexible

**Inconvénients** :
- ❌ Latence : 20-100ms
- ❌ Aggregation pipeline coûteuse
- ❌ Index B-tree pas optimal pour ranking
- ❌ Consommation mémoire élevée

**Verdict** : ⚠️ **Acceptable mais pas optimal** pour rankings compétitifs.

---

### Option 3 : Redis Sorted Sets ✅

```bash
# Ajouter/Update score (atomique, O(log N))
ZADD leaderboard:global 1250 "player123"

# Récupérer le rang (O(log N))
ZREVRANK leaderboard:global "player123"
# → 47822 (rang 0-indexed)

# Top 10 (O(log N + M))
ZREVRANGE leaderboard:global 0 9 WITHSCORES
# → [("player456", 9850), ("player789", 9720), ...]

# Joueurs autour de player123 (O(log N))
ZREVRANK leaderboard:global "player123"  # → 47822
ZREVRANGE leaderboard:global 47817 47827 WITHSCORES
# → 5 joueurs avant, player123, 5 joueurs après
```

**Avantages** :
- ✅ Latence : **< 1ms** pour toutes les opérations
- ✅ Complexité O(log N) garantie (Skip List)
- ✅ Opérations atomiques (pas de race conditions)
- ✅ Memory efficient (Skip List = 2× overhead vs Hash)
- ✅ Score updates en O(log N) même avec 50M éléments
- ✅ Pagination triviale avec ZRANGE

**Inconvénients** :
- ⚠️ In-memory uniquement (mais acceptable)
- ⚠️ Hot key potentiel (mais solutions existent)
- ⚠️ Pas de querying complexe (mais pas nécessaire)

**Benchmark réel** (Redis 7, Sorted Set 50M membres) :
```
Opération                        Latence
--------------------------------------
ZADD (update score)              0.2ms
ZREVRANK (get rank)              0.3ms
ZREVRANGE Top 10                 0.5ms
ZREVRANGE autour joueur          0.4ms
ZINCRBY (increment atomic)       0.2ms
```

**Trade-off assumé** :
- ➕ Performance × 1000 vs PostgreSQL
- ➕ Simplicité maximale
- ➕ Atomicité native
- ➖ RAM required (mais 50M × 100 bytes = 5GB, acceptable)

**Verdict** : ✅ **Solution optimale** pour leaderboards temps réel.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────┐
│                    Game Servers (Backend)                   │
│  ┌────────────┐   ┌─────────────┐  ┌────────────┐           │
│  │  Match 1   │   │  Match 2    │  │  Match N   │           │
│  │ (60 players│   │ (100 players│  │ (...)      │           │
│  └─────┬──────┘   └─────┬───────┘  └─────┬──────┘           │
└────────┼────────────────┼────────────────┼──────────────────┘
         │ Match End      │                │
         │ Events         │                │
         ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│              Score Processing Service                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Validation (anti-cheat)                          │    │
│  │  - Score calculation (kills, placement, time)       │    │
│  │  - Multi-leaderboard routing                        │    │
│  │  - Batch updates (pipeline)                         │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│           Redis Leaderboard Cluster                         │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Sorted Sets (Skip List data structure)            │     │
│  │                                                    │     │
│  │  leaderboard:global              (50M members)     │     │
│  │  leaderboard:season:2024-q4      (50M members)     │     │
│  │  leaderboard:region:NA           (15M members)     │     │
│  │  leaderboard:region:EU           (20M members)     │     │
│  │  leaderboard:mode:solo           (30M members)     │     │
│  │  leaderboard:mode:squad          (25M members)     │     │
│  │  leaderboard:daily:2024-12-11    (2M members)      │     │
│  │  ...                                               │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  Master-Replica Setup (High Availability)                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ Master   │──│ Replica1 │  │ Replica2 │                   │
│  │ (Writes) │  │ (Reads)  │  │ (Reads)  │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              Leaderboard API Service                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  REST API Endpoints:                                │    │
│  │  - GET /leaderboard/global/top?limit=100            │    │
│  │  - GET /leaderboard/player/{id}/rank                │    │
│  │  - GET /leaderboard/player/{id}/neighbors           │    │
│  │  - GET /leaderboard/friends/{id}                    │    │
│  │  - POST /leaderboard/update (internal only)         │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Game Clients                              │
│  - Real-time leaderboard display                             │
│  - Player rank notifications                                 │
│  - Friend comparisons                                        │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Flux de données

#### **Flux de mise à jour de score (Write Path)**

```
1. Match terminé (Game Server)
   ├─ Player A: 15 kills, 2nd place → 250 points
   ├─ Player B: 8 kills, 5th place → 180 points
   └─ Player C: 3 kills, eliminated → 50 points

2. Score Processing Service
   ├─ Validation anti-cheat
   ├─ Score calculation
   │  └─ score = (placement_points + kill_points + bonus)
   └─ Determine leaderboards à update:
      ├─ Global
      ├─ Region (player's region)
      ├─ Season (current season)
      ├─ Mode (solo/duo/squad)
      └─ Daily (today's date)

3. Batch update via Pipeline (atomique)
   ├─ ZADD leaderboard:global 250 "playerA"
   ├─ ZADD leaderboard:season:2024-q4 250 "playerA"
   ├─ ZADD leaderboard:region:NA 250 "playerA"
   ├─ ZADD leaderboard:mode:solo 250 "playerA"
   ├─ ZADD leaderboard:daily:2024-12-11 250 "playerA"
   └─ ... (repeat for players B, C)

4. Execute pipeline (1 RTT)
   └─ Latency: ~5ms for 10 updates

5. Check for achievements/notifications
   ├─ If rank improved → notify player
   ├─ If entered Top 1000 → special notification
   └─ If broke personal record → achievement

Total write latency: 10-15ms
Throughput: 100k updates/minute
```

#### **Flux de lecture de leaderboard (Read Path)**

```
1. Player opens leaderboard screen
   ├─ Request: GET /leaderboard/global/top?limit=100
   └─ Request: GET /leaderboard/player/123/rank

2. Leaderboard API Service
   ├─ Cache check (optional, TTL 10s)
   │  └─ If hit → return (latency ~1ms)
   └─ If miss → query Redis

3. Redis queries (parallel)
   ├─ ZREVRANGE leaderboard:global 0 99 WITHSCORES
   │  └─ Returns: Top 100 players (latency ~0.5ms)
   └─ ZREVRANK leaderboard:global "player123"
      └─ Returns: Rank 47823 (latency ~0.3ms)

4. Enrich data (optional)
   ├─ Fetch player metadata (username, avatar)
   │  └─ Hash lookup: HMGET player:123 username avatar
   └─ Calculate percentile: rank / total_members

5. Return JSON to client
   └─ Total latency: 3-8ms (p95)

Read throughput: 50k requests/sec per replica
```

### 3.3 Décisions architecturales clés

#### **Choix 1 : Sorted Sets vs Alternative structures**

**Pourquoi pas Hash + manual sorting ?**

```python
# ❌ Hash (pas de ordering natif)
redis.hset("scores", "player123", 1250)
# Pour obtenir le classement :
all_scores = redis.hgetall("scores")  # O(N) !
sorted_scores = sorted(all_scores.items(), key=lambda x: x[1], reverse=True)  # O(N log N) !
rank = next(i for i, (p, s) in enumerate(sorted_scores) if p == "player123")

# ✅ Sorted Set (ordering natif)
redis.zadd("leaderboard", {{"player123": 1250}})
rank = redis.zrevrank("leaderboard", "player123")  # O(log N) !
```

**Trade-off assumé** :
- ➕ O(log N) garanti pour toutes opérations
- ➕ Pas de computation côté application
- ➖ Consommation mémoire +50% vs Hash (Skip List overhead)

---

#### **Choix 2 : Score composite pour tie-breaking**

**Problème** : 2 joueurs avec même score → ordre arbitraire

```python
# ❌ Approche naïve
redis.zadd("leaderboard", {"player123": 1000, "player456": 1000})
# Ordre undefined si scores égaux

# ✅ Score composite : score × 10^10 + timestamp_inversé
import time

def create_composite_score(points: int, timestamp: float = None) -> float:
    """
    Créer un score composite pour tie-breaking

    Format: PPPPPPPPPPTTTTTTTTTT (20 digits)
    - 10 digits pour points (max 9,999,999,999)
    - 10 digits pour timestamp inversé (pour ordre chronologique)
    """
    if timestamp is None:
        timestamp = time.time()

    # Inverser timestamp pour ordre "premier arrivé = premier servi"
    # Max timestamp ~2^32 = 4,294,967,295 (year 2106)
    inverted_timestamp = 9999999999 - int(timestamp % 10000000000)

    composite = points * 10_000_000_000 + inverted_timestamp
    return float(composite)

# Exemple
score1 = create_composite_score(1000, timestamp=1702300000.0)
# → 10000000000001297700000 (1000 points, timestamp X)

score2 = create_composite_score(1000, timestamp=1702300001.0)
# → 10000000000001297699999 (1000 points, timestamp X+1)

# score1 > score2 → player avec timestamp plus ancien classé devant
```

**Trade-off assumé** :
- ➕ Ordre déterministe et juste
- ➕ Pas de requête supplémentaire pour tie-break
- ➖ Complexité légère de calcul (acceptable)

---

#### **Choix 3 : Multiple Leaderboards vs Single avec tags**

**Alternative envisagée** : 1 seul leaderboard avec metadata

```python
# ❌ Single leaderboard avec filtering
redis.zadd("leaderboard:all", {
    "player123:NA:solo": 1000,
    "player456:EU:squad": 950
})
# Problème : impossible de filter sur "NA" efficacement sans scan

# ✅ Multiple leaderboards spécialisés
redis.zadd("leaderboard:global", {"player123": 1000})
redis.zadd("leaderboard:region:NA", {"player123": 1000})
redis.zadd("leaderboard:mode:solo", {"player123": 1000})
# Requêtes ciblées et rapides
```

**Trade-off assumé** :
- ➕ Queries ultra-rapides (pas de filtering)
- ➕ Isolation (maintenance indépendante)
- ➖ Write amplification (5-10 leaderboards par update)
- ➖ Mémoire ×5-10 (mais acceptable)

---

## 4. Modélisation des données

### 4.1 Structure des Sorted Sets

**Clé** : `leaderboard:{scope}:{identifier}`
**Type** : Sorted Set (ZSET)
**Score** : Float (composite score)
**Member** : String (player_id)

```
Exemples de clés:

# Leaderboards globaux
leaderboard:global                    → Tous les joueurs, toutes saisons
leaderboard:season:2024-q4            → Saison en cours
leaderboard:season:2024-q3            → Saison précédente (archive)

# Leaderboards régionaux
leaderboard:region:NA                 → Amérique du Nord
leaderboard:region:EU                 → Europe
leaderboard:region:ASIA               → Asie
leaderboard:region:SA                 → Amérique du Sud
leaderboard:region:OCE                → Océanie

# Leaderboards par mode
leaderboard:mode:solo                 → Mode solo
leaderboard:mode:duo                  → Mode duo
leaderboard:mode:squad                → Mode squad (4 joueurs)

# Leaderboards temporels
leaderboard:daily:2024-12-11          → Journalier
leaderboard:weekly:2024-W50           → Hebdomadaire
leaderboard:monthly:2024-12           → Mensuel

# Leaderboards par rang
leaderboard:tier:bronze               → Joueurs Bronze
leaderboard:tier:silver               → Joueurs Silver
leaderboard:tier:gold                 → Joueurs Gold
leaderboard:tier:platinum             → Joueurs Platinum
leaderboard:tier:diamond              → Joueurs Diamond

# Leaderboards personnalisés
leaderboard:guild:guild_abc123        → Guilde spécifique
leaderboard:friends:player123         → Amis d'un joueur
```

### 4.2 Métadonnées de joueurs

**Clé** : `player:{player_id}`
**Type** : Hash

```
player:player123
├─ username: "ProGamer123"
├─ avatar_url: "https://cdn.example.com/avatars/123.jpg"
├─ level: "50"
├─ region: "NA"
├─ tier: "gold"
├─ total_matches: "1247"
├─ total_wins: "213"
├─ total_kills: "5892"
├─ created_at: "2024-01-15T10:00:00Z"
└─ last_match_at: "2024-12-11T14:30:00Z"
```

### 4.3 Index secondaires (pour requêtes rapides)

```
# Set des joueurs par région (pour bulk operations)
players:region:NA → Set { player123, player456, ... }

# Set des joueurs online
players:online → Set { player789, player321, ... }

# Set des amis d'un joueur
friends:player123 → Set { player456, player789, ... }
```

---

## 5. Implémentation technique

### 5.1 Code Python (Production-Ready)

```python
"""
Leaderboard Service avec Redis Sorted Sets
Implémentation production-ready avec composite scores, multi-leaderboards, pagination
"""

import time
import math
from typing import List, Dict, Any, Optional, Tuple
from dataclasses import dataclass
from enum import Enum
import logging

import redis
from redis.exceptions import RedisError

# Configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'db': 0,
    'decode_responses': True,
    'socket_timeout': 3,
    'retry_on_timeout': True,
    'max_connections': 100
}

# Constantes
SCORE_MULTIPLIER = 10_000_000_000  # 10^10 pour composite score
MAX_TIMESTAMP = 9_999_999_999


# ============================================================================
# Data Classes
# ============================================================================

class LeaderboardType(Enum):
    """Types de leaderboards"""
    GLOBAL = "global"
    SEASON = "season"
    REGION = "region"
    MODE = "mode"
    DAILY = "daily"
    WEEKLY = "weekly"
    TIER = "tier"
    GUILD = "guild"


@dataclass
class PlayerScore:
    """Score d'un joueur dans un leaderboard"""
    player_id: str
    username: str
    score: int
    rank: int
    percentile: float
    composite_score: float

    def to_dict(self) -> Dict:
        return {
            'player_id': self.player_id,
            'username': self.username,
            'score': self.score,
            'rank': self.rank,
            'percentile': round(self.percentile, 2)
        }


@dataclass
class LeaderboardPage:
    """Page de résultats d'un leaderboard"""
    players: List[PlayerScore]
    total_players: int
    page: int
    page_size: int
    total_pages: int


# ============================================================================
# Leaderboard Service
# ============================================================================

class LeaderboardService:
    """
    Service de gestion des leaderboards

    Features:
    - Composite scores pour tie-breaking
    - Multiple leaderboards (global, region, mode, etc.)
    - Pagination efficace
    - Rank lookup en O(log N)
    - Batch updates atomiques
    - Player neighbors (autour d'un joueur)
    """

    def __init__(self, redis_config: Dict = None):
        config = redis_config or REDIS_CONFIG
        self.redis = redis.Redis(**config)

        # Test connexion
        try:
            self.redis.ping()
            logger.info("LeaderboardService initialized")
        except RedisError as e:
            logger.error(f"Redis connection failed: {e}")
            raise

    # ========================================================================
    # Score Management
    # ========================================================================

    def create_composite_score(
        self,
        points: int,
        timestamp: float = None,
        tie_breaker: Optional[int] = None
    ) -> float:
        """
        Créer un score composite pour ordering et tie-breaking

        Format: PPPPPPPPPPTTTTTTTTTT
        - 10 digits: points (max 9,999,999,999)
        - 10 digits: timestamp inversé ou tie-breaker

        Args:
            points: Score de base (0 à 9,999,999,999)
            timestamp: Unix timestamp (auto si None)
            tie_breaker: Valeur custom pour tie-breaking (override timestamp)

        Returns:
            Composite score as float
        """
        if points < 0 or points >= SCORE_MULTIPLIER:
            raise ValueError(f"Points must be in range [0, {SCORE_MULTIPLIER})")

        if tie_breaker is not None:
            # Custom tie-breaker
            if tie_breaker < 0 or tie_breaker >= SCORE_MULTIPLIER:
                raise ValueError(f"Tie breaker must be in range [0, {SCORE_MULTIPLIER})")
            inverted_tie = MAX_TIMESTAMP - tie_breaker
        else:
            # Timestamp-based tie-breaking
            if timestamp is None:
                timestamp = time.time()

            # Inverser pour "premier arrivé = premier servi"
            inverted_timestamp = MAX_TIMESTAMP - int(timestamp % SCORE_MULTIPLIER)
            inverted_tie = inverted_timestamp

        composite = points * SCORE_MULTIPLIER + inverted_tie
        return float(composite)

    def parse_composite_score(self, composite_score: float) -> Tuple[int, int]:
        """
        Parser un composite score

        Returns:
            (points, tie_breaker_value)
        """
        composite_int = int(composite_score)
        points = composite_int // SCORE_MULTIPLIER
        tie_value = composite_int % SCORE_MULTIPLIER

        return points, tie_value

    # ========================================================================
    # Core Operations
    # ========================================================================

    def update_score(
        self,
        player_id: str,
        points: int,
        leaderboards: List[str],
        timestamp: float = None
    ) -> Dict[str, int]:
        """
        Mettre à jour le score d'un joueur dans plusieurs leaderboards

        Args:
            player_id: ID du joueur
            points: Nouveaux points
            leaderboards: Liste des leaderboards à update
            timestamp: Timestamp pour tie-breaking

        Returns:
            Dict {leaderboard: nouveau_rang}
        """
        composite_score = self.create_composite_score(points, timestamp)

        # Pipeline pour atomicité
        pipe = self.redis.pipeline()

        for lb in leaderboards:
            key = self._get_leaderboard_key(lb)
            pipe.zadd(key, {player_id: composite_score})

        try:
            pipe.execute()
            logger.info(
                f"Score updated: player={player_id}, points={points}, "
                f"leaderboards={len(leaderboards)}"
            )

            # Récupérer les rangs (optionnel, pas dans pipeline pour perf)
            ranks = {}
            for lb in leaderboards:
                rank = self.get_player_rank(player_id, lb)
                if rank is not None:
                    ranks[lb] = rank

            return ranks

        except RedisError as e:
            logger.error(f"Failed to update score: {e}")
            raise

    def increment_score(
        self,
        player_id: str,
        points_delta: int,
        leaderboard: str
    ) -> float:
        """
        Incrémenter le score d'un joueur (atomique)

        Note: ZINCRBY ne supporte que l'incrémentation simple.
        Pour conserver le tie-breaking, utiliser update_score() à la place.

        Args:
            player_id: ID du joueur
            points_delta: Points à ajouter (peut être négatif)
            leaderboard: Leaderboard concerné

        Returns:
            Nouveau score composite
        """
        key = self._get_leaderboard_key(leaderboard)

        try:
            # ZINCRBY : O(log N)
            new_score = self.redis.zincrby(key, points_delta, player_id)

            logger.info(
                f"Score incremented: player={player_id}, delta={points_delta}, "
                f"leaderboard={leaderboard}"
            )

            return float(new_score)

        except RedisError as e:
            logger.error(f"Failed to increment score: {e}")
            raise

    def get_player_rank(self, player_id: str, leaderboard: str) -> Optional[int]:
        """
        Obtenir le rang d'un joueur (1-indexed)

        Returns:
            Rang (1 = meilleur) ou None si joueur absent
        """
        key = self._get_leaderboard_key(leaderboard)

        try:
            # ZREVRANK : O(log N), retourne rank 0-indexed
            rank = self.redis.zrevrank(key, player_id)

            if rank is None:
                return None

            # Convertir en 1-indexed
            return rank + 1

        except RedisError as e:
            logger.error(f"Failed to get player rank: {e}")
            raise

    def get_player_score(self, player_id: str, leaderboard: str) -> Optional[int]:
        """
        Obtenir le score d'un joueur

        Returns:
            Score (points seulement) ou None si absent
        """
        key = self._get_leaderboard_key(leaderboard)

        try:
            composite_score = self.redis.zscore(key, player_id)

            if composite_score is None:
                return None

            points, _ = self.parse_composite_score(composite_score)
            return points

        except RedisError as e:
            logger.error(f"Failed to get player score: {e}")
            raise

    def get_total_players(self, leaderboard: str) -> int:
        """Obtenir le nombre total de joueurs dans un leaderboard"""
        key = self._get_leaderboard_key(leaderboard)

        try:
            return self.redis.zcard(key)
        except RedisError as e:
            logger.error(f"Failed to get total players: {e}")
            raise

    # ========================================================================
    # Leaderboard Queries
    # ========================================================================

    def get_top_players(
        self,
        leaderboard: str,
        limit: int = 100,
        offset: int = 0,
        with_metadata: bool = True
    ) -> List[PlayerScore]:
        """
        Récupérer les meilleurs joueurs (Top N)

        Args:
            leaderboard: Nom du leaderboard
            limit: Nombre de joueurs à retourner
            offset: Offset pour pagination
            with_metadata: Si True, fetch username et autres metadata

        Returns:
            Liste de PlayerScore
        """
        key = self._get_leaderboard_key(leaderboard)

        try:
            # ZREVRANGE avec WITHSCORES : O(log N + M)
            results = self.redis.zrevrange(
                key,
                offset,
                offset + limit - 1,
                withscores=True
            )

            total_players = self.redis.zcard(key)

            players = []
            for i, (player_id, composite_score) in enumerate(results):
                rank = offset + i + 1
                points, _ = self.parse_composite_score(composite_score)
                percentile = (rank / total_players) * 100 if total_players > 0 else 0

                # Fetch metadata si demandé
                username = player_id
                if with_metadata:
                    username = self._get_player_username(player_id)

                players.append(PlayerScore(
                    player_id=player_id,
                    username=username,
                    score=points,
                    rank=rank,
                    percentile=percentile,
                    composite_score=composite_score
                ))

            return players

        except RedisError as e:
            logger.error(f"Failed to get top players: {e}")
            raise

    def get_players_around(
        self,
        player_id: str,
        leaderboard: str,
        range_size: int = 5,
        with_metadata: bool = True
    ) -> List[PlayerScore]:
        """
        Récupérer les joueurs autour d'un joueur spécifique

        Args:
            player_id: Joueur central
            leaderboard: Nom du leaderboard
            range_size: Nombre de joueurs avant et après (total = 2×range_size + 1)
            with_metadata: Fetch metadata

        Returns:
            Liste de PlayerScore (range_size avant, player, range_size après)
        """
        key = self._get_leaderboard_key(leaderboard)

        try:
            # Récupérer le rang du joueur
            rank = self.redis.zrevrank(key, player_id)

            if rank is None:
                logger.warning(f"Player {player_id} not found in {leaderboard}")
                return []

            # Calculer range
            start = max(0, rank - range_size)
            end = rank + range_size

            # ZREVRANGE : O(log N + M)
            results = self.redis.zrevrange(key, start, end, withscores=True)

            total_players = self.redis.zcard(key)

            players = []
            for i, (pid, composite_score) in enumerate(results):
                current_rank = start + i + 1
                points, _ = self.parse_composite_score(composite_score)
                percentile = (current_rank / total_players) * 100

                username = pid
                if with_metadata:
                    username = self._get_player_username(pid)

                players.append(PlayerScore(
                    player_id=pid,
                    username=username,
                    score=points,
                    rank=current_rank,
                    percentile=percentile,
                    composite_score=composite_score
                ))

            return players

        except RedisError as e:
            logger.error(f"Failed to get players around: {e}")
            raise

    def get_leaderboard_page(
        self,
        leaderboard: str,
        page: int = 1,
        page_size: int = 100,
        with_metadata: bool = True
    ) -> LeaderboardPage:
        """
        Récupérer une page de leaderboard (pagination)

        Args:
            leaderboard: Nom du leaderboard
            page: Numéro de page (1-indexed)
            page_size: Taille de la page
            with_metadata: Fetch metadata

        Returns:
            LeaderboardPage
        """
        offset = (page - 1) * page_size
        players = self.get_top_players(leaderboard, page_size, offset, with_metadata)

        total_players = self.get_total_players(leaderboard)
        total_pages = math.ceil(total_players / page_size) if page_size > 0 else 0

        return LeaderboardPage(
            players=players,
            total_players=total_players,
            page=page,
            page_size=page_size,
            total_pages=total_pages
        )

    def get_player_full_stats(
        self,
        player_id: str,
        leaderboards: List[str]
    ) -> Dict[str, Dict]:
        """
        Récupérer les stats d'un joueur dans plusieurs leaderboards

        Returns:
            Dict {leaderboard: {rank, score, percentile}}
        """
        stats = {}

        for lb in leaderboards:
            rank = self.get_player_rank(player_id, lb)
            score = self.get_player_score(player_id, lb)

            if rank is not None and score is not None:
                total = self.get_total_players(lb)
                percentile = (rank / total) * 100 if total > 0 else 0

                stats[lb] = {
                    'rank': rank,
                    'score': score,
                    'percentile': round(percentile, 2),
                    'total_players': total
                }

        return stats

    # ========================================================================
    # Batch Operations
    # ========================================================================

    def batch_update_scores(
        self,
        updates: List[Dict[str, Any]],
        leaderboard: str
    ) -> int:
        """
        Batch update de scores (optimisé)

        Args:
            updates: Liste de {player_id, points, timestamp}
            leaderboard: Leaderboard à update

        Returns:
            Nombre de joueurs mis à jour
        """
        key = self._get_leaderboard_key(leaderboard)

        # Préparer les données
        mapping = {}
        for update in updates:
            player_id = update['player_id']
            points = update['points']
            timestamp = update.get('timestamp', time.time())

            composite_score = self.create_composite_score(points, timestamp)
            mapping[player_id] = composite_score

        try:
            # ZADD avec mapping : O(M × log N)
            count = self.redis.zadd(key, mapping)

            logger.info(
                f"Batch update: {len(updates)} players in {leaderboard}, "
                f"{count} updated"
            )

            return count

        except RedisError as e:
            logger.error(f"Batch update failed: {e}")
            raise

    def remove_player(self, player_id: str, leaderboards: List[str]) -> int:
        """
        Retirer un joueur de plusieurs leaderboards

        Returns:
            Nombre de leaderboards modifiés
        """
        pipe = self.redis.pipeline()

        for lb in leaderboards:
            key = self._get_leaderboard_key(lb)
            pipe.zrem(key, player_id)

        try:
            results = pipe.execute()
            removed = sum(results)

            logger.info(f"Player {player_id} removed from {removed} leaderboards")
            return removed

        except RedisError as e:
            logger.error(f"Failed to remove player: {e}")
            raise

    # ========================================================================
    # Leaderboard Management
    # ========================================================================

    def reset_leaderboard(self, leaderboard: str) -> bool:
        """Reset (supprimer) un leaderboard"""
        key = self._get_leaderboard_key(leaderboard)

        try:
            deleted = self.redis.delete(key)
            logger.info(f"Leaderboard {leaderboard} reset")
            return deleted > 0

        except RedisError as e:
            logger.error(f"Failed to reset leaderboard: {e}")
            raise

    def copy_leaderboard(self, source: str, destination: str) -> int:
        """
        Copier un leaderboard (utile pour archivage de saison)

        Returns:
            Nombre de joueurs copiés
        """
        src_key = self._get_leaderboard_key(source)
        dst_key = self._get_leaderboard_key(destination)

        try:
            # Récupérer tous les membres avec scores
            all_members = self.redis.zrange(src_key, 0, -1, withscores=True)

            if not all_members:
                logger.warning(f"Source leaderboard {source} is empty")
                return 0

            # Copier via ZADD
            mapping = {member: score for member, score in all_members}
            count = self.redis.zadd(dst_key, mapping)

            logger.info(f"Copied {count} players from {source} to {destination}")
            return count

        except RedisError as e:
            logger.error(f"Failed to copy leaderboard: {e}")
            raise

    # ========================================================================
    # Helper Methods
    # ========================================================================

    def _get_leaderboard_key(self, leaderboard: str) -> str:
        """Construire la clé Redis du leaderboard"""
        if leaderboard.startswith("leaderboard:"):
            return leaderboard
        return f"leaderboard:{leaderboard}"

    def _get_player_username(self, player_id: str) -> str:
        """Récupérer le username d'un joueur depuis metadata"""
        try:
            username = self.redis.hget(f"player:{player_id}", "username")
            return username or player_id
        except RedisError:
            return player_id

    # ========================================================================
    # Analytics
    # ========================================================================

    def get_leaderboard_stats(self, leaderboard: str) -> Dict:
        """Statistiques d'un leaderboard"""
        key = self._get_leaderboard_key(leaderboard)

        try:
            total = self.redis.zcard(key)

            if total == 0:
                return {
                    'total_players': 0,
                    'top_score': 0,
                    'median_score': 0,
                    'bottom_score': 0
                }

            # Top score
            top_entry = self.redis.zrevrange(key, 0, 0, withscores=True)
            top_score, _ = self.parse_composite_score(top_entry[0][1]) if top_entry else (0, 0)

            # Median score
            median_idx = total // 2
            median_entry = self.redis.zrevrange(key, median_idx, median_idx, withscores=True)
            median_score, _ = self.parse_composite_score(median_entry[0][1]) if median_entry else (0, 0)

            # Bottom score
            bottom_entry = self.redis.zrange(key, 0, 0, withscores=True)
            bottom_score, _ = self.parse_composite_score(bottom_entry[0][1]) if bottom_entry else (0, 0)

            return {
                'total_players': total,
                'top_score': top_score,
                'median_score': median_score,
                'bottom_score': bottom_score
            }

        except RedisError as e:
            logger.error(f"Failed to get leaderboard stats: {e}")
            return {}


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    # Initialize service
    service = LeaderboardService()

    # Simuler une fin de partie avec 10 joueurs
    print("\n🎮 Simulating match end with 10 players...")

    players = [
        ("player001", 250, "ProGamer"),
        ("player002", 230, "EliteSniper"),
        ("player003", 210, "Ninja123"),
        ("player004", 200, "SpeedRunner"),
        ("player005", 180, "TacticalPro"),
        ("player006", 150, "CasualPlayer"),
        ("player007", 120, "Noob42"),
        ("player008", 100, "LuckyShot"),
        ("player009", 80, "FirstTimer"),
        ("player010", 50, "JustStarted"),
    ]

    # Batch update scores
    updates = [
        {'player_id': pid, 'points': score}
        for pid, score, _ in players
    ]

    service.batch_update_scores(updates, "global")
    service.batch_update_scores(updates, "season:2024-q4")
    service.batch_update_scores(updates, "region:NA")

    print("✅ Scores updated in 3 leaderboards")

    # Récupérer Top 10
    print("\n🏆 Top 10 Global Leaderboard:")
    top_players = service.get_top_players("global", limit=10)

    for player in top_players:
        print(
            f"   #{player.rank}: {player.username} - "
            f"{player.score} pts (Top {player.percentile:.1f}%)"
        )

    # Récupérer stats d'un joueur
    print("\n📊 Player stats (player005):")
    stats = service.get_player_full_stats(
        "player005",
        ["global", "season:2024-q4", "region:NA"]
    )

    for lb, data in stats.items():
        print(
            f"   {lb}: Rank #{data['rank']} / {data['total_players']} "
            f"({data['score']} pts, Top {data['percentile']}%)"
        )

    # Joueurs autour de player005
    print("\n👥 Players around player005:")
    neighbors = service.get_players_around("player005", "global", range_size=2)

    for player in neighbors:
        marker = "👉" if player.player_id == "player005" else "  "
        print(
            f"   {marker} #{player.rank}: {player.player_id} - {player.score} pts"
        )

    # Stats du leaderboard
    print("\n📈 Leaderboard statistics:")
    stats = service.get_leaderboard_stats("global")
    print(f"   Total players: {stats['total_players']}")
    print(f"   Top score: {stats['top_score']}")
    print(f"   Median score: {stats['median_score']}")
    print(f"   Bottom score: {stats['bottom_score']}")
```

### 5.2 Code Node.js (API REST)

```javascript
/**
 * Leaderboard API avec Express.js et ioredis
 * Production-ready avec rate limiting et caching
 */

const express = require('express');
const Redis = require('ioredis');
const app = express();

// Configuration
const REDIS_CONFIG = {
  host: 'localhost',
  port: 6379,
  retryStrategy: (times) => Math.min(times * 50, 2000),
};

const redis = new Redis(REDIS_CONFIG);
const SCORE_MULTIPLIER = 10_000_000_000;
const MAX_TIMESTAMP = 9_999_999_999;

// Middleware
app.use(express.json());

// ============================================================================
// Helper Functions
// ============================================================================

function createCompositeScore(points, timestamp = null) {
  if (points < 0 || points >= SCORE_MULTIPLIER) {
    throw new Error(`Points out of range: ${points}`);
  }

  if (timestamp === null) {
    timestamp = Date.now() / 1000;
  }

  const invertedTimestamp = MAX_TIMESTAMP - Math.floor(timestamp % SCORE_MULTIPLIER);
  const composite = points * SCORE_MULTIPLIER + invertedTimestamp;

  return composite;
}

function parseCompositeScore(compositeScore) {
  const compositeInt = Math.floor(compositeScore);
  const points = Math.floor(compositeInt / SCORE_MULTIPLIER);
  const tieValue = compositeInt % SCORE_MULTIPLIER;

  return { points, tieValue };
}

function getLeaderboardKey(leaderboard) {
  if (leaderboard.startsWith('leaderboard:')) {
    return leaderboard;
  }
  return `leaderboard:${leaderboard}`;
}

// ============================================================================
// API Endpoints
// ============================================================================

/**
 * GET /leaderboard/:name/top
 * Récupérer les meilleurs joueurs
 *
 * Query params:
 * - limit: nombre de joueurs (default: 100)
 * - offset: offset pour pagination (default: 0)
 */
app.get('/leaderboard/:name/top', async (req, res) => {
  try {
    const { name } = req.params;
    const limit = parseInt(req.query.limit) || 100;
    const offset = parseInt(req.query.offset) || 0;

    if (limit > 1000) {
      return res.status(400).json({ error: 'Limit too high (max: 1000)' });
    }

    const key = getLeaderboardKey(name);

    // ZREVRANGE avec WITHSCORES
    const results = await redis.zrevrange(
      key,
      offset,
      offset + limit - 1,
      'WITHSCORES'
    );

    // Parse results
    const players = [];
    for (let i = 0; i < results.length; i += 2) {
      const playerId = results[i];
      const compositeScore = parseFloat(results[i + 1]);
      const { points } = parseCompositeScore(compositeScore);
      const rank = offset + (i / 2) + 1;

      players.push({
        player_id: playerId,
        score: points,
        rank,
      });
    }

    // Total count
    const totalPlayers = await redis.zcard(key);

    res.json({
      players,
      total_players: totalPlayers,
      limit,
      offset,
    });

  } catch (err) {
    console.error('Error fetching top players:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

/**
 * GET /leaderboard/:name/player/:playerId
 * Récupérer le rang et score d'un joueur
 */
app.get('/leaderboard/:name/player/:playerId', async (req, res) => {
  try {
    const { name, playerId } = req.params;
    const key = getLeaderboardKey(name);

    // Parallel queries
    const [rank, compositeScore, totalPlayers] = await Promise.all([
      redis.zrevrank(key, playerId),
      redis.zscore(key, playerId),
      redis.zcard(key),
    ]);

    if (rank === null || compositeScore === null) {
      return res.status(404).json({ error: 'Player not found in leaderboard' });
    }

    const { points } = parseCompositeScore(parseFloat(compositeScore));
    const rankOneIndexed = rank + 1;
    const percentile = ((rankOneIndexed / totalPlayers) * 100).toFixed(2);

    res.json({
      player_id: playerId,
      rank: rankOneIndexed,
      score: points,
      percentile: parseFloat(percentile),
      total_players: totalPlayers,
    });

  } catch (err) {
    console.error('Error fetching player rank:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

/**
 * GET /leaderboard/:name/player/:playerId/neighbors
 * Récupérer les joueurs autour d'un joueur
 *
 * Query params:
 * - range: nombre de joueurs avant/après (default: 5)
 */
app.get('/leaderboard/:name/player/:playerId/neighbors', async (req, res) => {
  try {
    const { name, playerId } = req.params;
    const range = parseInt(req.query.range) || 5;
    const key = getLeaderboardKey(name);

    // Get player rank
    const rank = await redis.zrevrank(key, playerId);

    if (rank === null) {
      return res.status(404).json({ error: 'Player not found' });
    }

    // Calculate range
    const start = Math.max(0, rank - range);
    const end = rank + range;

    // ZREVRANGE
    const results = await redis.zrevrange(key, start, end, 'WITHSCORES');

    // Parse results
    const players = [];
    for (let i = 0; i < results.length; i += 2) {
      const pid = results[i];
      const compositeScore = parseFloat(results[i + 1]);
      const { points } = parseCompositeScore(compositeScore);
      const currentRank = start + (i / 2) + 1;

      players.push({
        player_id: pid,
        score: points,
        rank: currentRank,
        is_target: pid === playerId,
      });
    }

    res.json({ players });

  } catch (err) {
    console.error('Error fetching neighbors:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

/**
 * POST /leaderboard/:name/update
 * Mettre à jour le score d'un joueur (internal API)
 *
 * Body:
 * {
 *   "player_id": "player123",
 *   "points": 1500
 * }
 */
app.post('/leaderboard/:name/update', async (req, res) => {
  try {
    const { name } = req.params;
    const { player_id, points } = req.body;

    if (!player_id || points === undefined) {
      return res.status(400).json({ error: 'Missing player_id or points' });
    }

    const key = getLeaderboardKey(name);
    const compositeScore = createCompositeScore(points);

    // ZADD
    await redis.zadd(key, compositeScore, player_id);

    // Get new rank
    const rank = await redis.zrevrank(key, player_id);

    res.json({
      success: true,
      player_id,
      score: points,
      rank: rank + 1,
    });

  } catch (err) {
    console.error('Error updating score:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

/**
 * POST /leaderboard/:name/batch-update
 * Batch update de scores (internal API)
 *
 * Body:
 * {
 *   "updates": [
 *     { "player_id": "player1", "points": 100 },
 *     { "player_id": "player2", "points": 200 }
 *   ]
 * }
 */
app.post('/leaderboard/:name/batch-update', async (req, res) => {
  try {
    const { name } = req.params;
    const { updates } = req.body;

    if (!Array.isArray(updates) || updates.length === 0) {
      return res.status(400).json({ error: 'Invalid updates array' });
    }

    const key = getLeaderboardKey(name);

    // Prepare ZADD arguments
    const zaddArgs = [];
    for (const update of updates) {
      const { player_id, points } = update;
      if (player_id && points !== undefined) {
        const compositeScore = createCompositeScore(points);
        zaddArgs.push(compositeScore, player_id);
      }
    }

    if (zaddArgs.length === 0) {
      return res.status(400).json({ error: 'No valid updates' });
    }

    // ZADD with multiple members
    const count = await redis.zadd(key, ...zaddArgs);

    res.json({
      success: true,
      updated: count,
      total_submitted: updates.length,
    });

  } catch (err) {
    console.error('Error batch updating:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

/**
 * GET /leaderboard/:name/stats
 * Statistiques d'un leaderboard
 */
app.get('/leaderboard/:name/stats', async (req, res) => {
  try {
    const { name } = req.params;
    const key = getLeaderboardKey(name);

    const total = await redis.zcard(key);

    if (total === 0) {
      return res.json({
        total_players: 0,
        top_score: 0,
        median_score: 0,
        bottom_score: 0,
      });
    }

    // Parallel queries
    const [topEntry, medianEntry, bottomEntry] = await Promise.all([
      redis.zrevrange(key, 0, 0, 'WITHSCORES'),
      redis.zrevrange(key, Math.floor(total / 2), Math.floor(total / 2), 'WITHSCORES'),
      redis.zrange(key, 0, 0, 'WITHSCORES'),
    ]);

    const { points: topScore } = parseCompositeScore(parseFloat(topEntry[1] || 0));
    const { points: medianScore } = parseCompositeScore(parseFloat(medianEntry[1] || 0));
    const { points: bottomScore } = parseCompositeScore(parseFloat(bottomEntry[1] || 0));

    res.json({
      total_players: total,
      top_score: topScore,
      median_score: medianScore,
      bottom_score: bottomScore,
    });

  } catch (err) {
    console.error('Error fetching stats:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// ============================================================================
// Server Start
// ============================================================================

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`🚀 Leaderboard API listening on port ${PORT}`);
  console.log(`   GET  /leaderboard/:name/top`);
  console.log(`   GET  /leaderboard/:name/player/:playerId`);
  console.log(`   GET  /leaderboard/:name/player/:playerId/neighbors`);
  console.log(`   POST /leaderboard/:name/update`);
  console.log(`   POST /leaderboard/:name/batch-update`);
  console.log(`   GET  /leaderboard/:name/stats`);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, closing connections...');
  await redis.quit();
  process.exit(0);
});

module.exports = app;
```

---

## 6. Cas avancés et optimisations

### 6.1 Sharding des leaderboards (Hot Key mitigation)

**Problème** : Leaderboard global = 1 seule clé → hot key

**Solution** : Sharding horizontal avec merge côté application

```python
class ShardedLeaderboard:
    """Leaderboard shardé pour distribuer la charge"""

    def __init__(self, num_shards: int = 10):
        self.num_shards = num_shards
        self.redis = redis.Redis(**REDIS_CONFIG)

    def _get_shard(self, player_id: str) -> int:
        """Déterminer le shard via hash consistant"""
        hash_value = int(hashlib.md5(player_id.encode()).hexdigest(), 16)
        return hash_value % self.num_shards

    def update_score(self, player_id: str, points: int):
        """Update score dans le shard approprié"""
        shard = self._get_shard(player_id)
        key = f"leaderboard:global:shard:{shard}"

        composite_score = create_composite_score(points)
        self.redis.zadd(key, {player_id: composite_score})

    def get_global_rank(self, player_id: str) -> int:
        """Calculer le rang global (merge de tous les shards)"""
        player_shard = self._get_shard(player_id)
        player_key = f"leaderboard:global:shard:{player_shard}"

        # Score du joueur
        player_score = self.redis.zscore(player_key, player_id)
        if player_score is None:
            return None

        # Compter joueurs avec score supérieur dans TOUS les shards
        total_better = 0

        for shard in range(self.num_shards):
            key = f"leaderboard:global:shard:{shard}"
            # ZCOUNT : nombre de joueurs avec score > player_score
            count = self.redis.zcount(key, player_score + 0.01, '+inf')
            total_better += count

        return total_better + 1

    def get_top_players_merged(self, limit: int = 100) -> List:
        """Merger les top players de tous les shards"""
        all_players = []

        # Fetch top N×2 de chaque shard (pour être sûr d'avoir assez)
        for shard in range(self.num_shards):
            key = f"leaderboard:global:shard:{shard}"
            players = self.redis.zrevrange(key, 0, limit * 2 - 1, withscores=True)
            all_players.extend(players)

        # Tri global et limitation
        all_players.sort(key=lambda x: x[1], reverse=True)
        return all_players[:limit]

# Trade-offs:
# ➕ Distribue la charge sur N shards (×N throughput)
# ➕ Pas de hot key
# ➖ get_global_rank() plus lent (N requêtes)
# ➖ Pagination globale complexe
```

### 6.2 Leaderboards temporels avec expiration automatique

```python
def create_daily_leaderboard():
    """Créer un leaderboard journalier avec TTL automatique"""
    from datetime import datetime, timedelta

    today = datetime.now().strftime("%Y-%m-%d")
    key = f"leaderboard:daily:{today}"

    # TTL = 7 jours (garder historique)
    ttl_seconds = 7 * 24 * 3600

    # Update scores...
    redis.zadd(key, {"player123": 1000})

    # Set TTL
    redis.expire(key, ttl_seconds)

    logger.info(f"Daily leaderboard {key} created with TTL {ttl_seconds}s")

# Avantages:
# ✅ Nettoyage automatique (pas de cron job)
# ✅ Mémoire libérée automatiquement
```

### 6.3 Composite scores avancés (multi-critères)

```python
def create_advanced_composite_score(
    wins: int,
    kills: int,
    survival_time: int
) -> float:
    """
    Score composite avec plusieurs critères

    Format: WWWWKKKKTTTT (12 digits)
    - 4 digits: wins (max 9999)
    - 4 digits: kills (max 9999)
    - 4 digits: survival_time en minutes (max 9999)
    """
    if any(x < 0 or x > 9999 for x in [wins, kills, survival_time]):
        raise ValueError("Values must be in range [0, 9999]")

    composite = wins * 100_000_000 + kills * 10_000 + survival_time
    return float(composite)

# Exemple
score_player1 = create_advanced_composite_score(wins=50, kills=1200, survival_time=1500)
score_player2 = create_advanced_composite_score(wins=50, kills=1200, survival_time=1400)

# player1 > player2 car survival_time supérieur (tie-break sur 3e critère)
```

---

## 7. Monitoring et métriques

### 7.1 KPIs critiques

```yaml
# Dashboard Grafana

# 1. Latence des opérations
zadd_latency_ms:
  target: < 1ms
  p99: < 2ms

zrevrank_latency_ms:
  target: < 1ms
  p99: < 3ms

# 2. Throughput
score_updates_per_second:
  normal: 1000/s
  peak: 10000/s

leaderboard_reads_per_second:
  normal: 5000/s
  peak: 50000/s

# 3. Taille des leaderboards
leaderboard_size_members:
  global: 50M
  seasonal: 50M
  daily: 2M

# 4. Memory usage
leaderboard_memory_gb:
  per_1M_members: ~100MB
  total: ~50GB

# 5. Hot key detection
hot_key_ops_per_second:
  warning: > 10000/s
  critical: > 50000/s
```

### 7.2 Alertes

```yaml
# Prometheus Alerts

groups:
  - name: leaderboard_alerts
    rules:

      # Latence excessive
      - alert: LeaderboardHighLatency
        expr: histogram_quantile(0.99, redis_command_duration_seconds{cmd="zadd"}) > 0.005
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "ZADD p99 latency > 5ms"

      # Hot key détecté
      - alert: LeaderboardHotKey
        expr: rate(redis_commands_total{key=~"leaderboard:global.*"}[1m]) > 50000
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Hot key detected on global leaderboard (>50k ops/s)"

      # Leaderboard size anormal
      - alert: LeaderboardSizeExplosion
        expr: redis_sorted_set_size{key="leaderboard:global"} > 100000000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Global leaderboard > 100M members (memory concern)"
```

---

## 8. Conclusion

### Points clés à retenir

- ✅ **Sorted Sets = Structure native optimale** pour leaderboards (O(log N))
- ✅ **Latence sub-milliseconde** garantie même avec 50M joueurs
- ✅ **Composite scores** pour tie-breaking déterministe
- ✅ **Multiple leaderboards** sans duplication de logique
- ✅ **Atomicité native** : pas de race conditions
- ✅ **Simplicité d'implémentation** vs alternatives SQL/NoSQL
- ✅ **Performance ×1000** vs PostgreSQL pour même use case

### Quand NE PAS utiliser Sorted Sets

- ❌ **Leaderboards avec filtres complexes** : RediSearch plus approprié
- ❌ **Historique complet des scores** : Base relationnelle nécessaire
- ❌ **Leaderboards > 500M joueurs** : Considérer sharding ou alternative
- ❌ **Requêtes analytiques avancées** : Warehouse (BigQuery, Snowflake)

### Comparaison finale

| Critère | PostgreSQL | MongoDB | Redis Sorted Set |
|---------|------------|---------|------------------|
| Latence (p99) | 100-500ms | 20-100ms | **< 2ms** |
| Throughput writes | 1k/s | 10k/s | **100k/s** |
| Complexité O() | O(N log N) | O(log N) | **O(log N)** |
| Memory usage | Disk | Disk+RAM | **RAM only** |
| Scaling | Difficile | Bon | **Excellent** |

### Prochaines lectures

- [Cas #4 : Analytics temps réel](./04-cas-analytics-temps-reel.md) → HyperLogLog + TimeSeries
- [Sorted Sets avancés](../02-structures-donnees-natives/06-sorted-sets-leaderboards-geospatial.md) → ZUNIONSTORE, ZINTERSTORE
- [Redis Cluster](../11-architecture-distribuee-scaling/03-distribution-donnees-hash-slots.md) → Scaling horizontal

---

**📚 Ressources complémentaires** :
- [Redis Sorted Sets Documentation](https://redis.io/docs/data-types/sorted-sets/)
- [Leaderboard Patterns (Redis Labs)](https://redis.com/solutions/use-cases/leaderboards/)
- [Skip List Algorithm Explained](https://en.wikipedia.org/wiki/Skip_list)

⏭️ [Cas #4 : Analytics temps réel (HyperLogLog + TimeSeries)](/16-etudes-cas-patterns-reels/04-cas-analytics-temps-reel.md)
