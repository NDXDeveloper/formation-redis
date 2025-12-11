🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cas #1 : Session Store avec RedisJSON et TTL

## Vue d'ensemble

**Niveau** : ⭐⭐ Intermédiaire
**Complexité technique** : Moyenne
**Impact production** : Critique (disponibilité utilisateur)
**Technologies** : Redis Core + RedisJSON

---

## 1. Contexte et problématique

### Scénario business

Une plateforme SaaS multi-tenant (type Notion, Figma, ou Slack) avec les caractéristiques suivantes :

**Chiffres clés** :
- 5 millions d'utilisateurs actifs mensuels
- 500 000 utilisateurs connectés simultanément en peak
- 50 data centers répartis sur 5 continents
- SLA : 99.95% de disponibilité
- Latence cible : < 5ms pour lecture de session (p99)

**Besoins métier** :
- Authentification multi-facteurs (MFA)
- Personnalisation avancée (thèmes, préférences UI, raccourcis clavier)
- Permissions granulaires par workspace/projet
- Tracking de l'activité utilisateur (dernière action, device info)
- Support multi-device (desktop, mobile, tablette)

### Problèmes à résoudre

#### 1. **Scalabilité horizontale**

```
❌ Problème : Sessions en mémoire serveur (sticky sessions)
- Complexité du load balancing
- Perte de session si serveur crashe
- Impossible de scaler dynamiquement

✅ Solution : Session store externalisé
- Load balancing round-robin simple
- Haute disponibilité native
- Auto-scaling des serveurs applicatifs
```

#### 2. **Données de session complexes**

```json
{
  "user_id": "usr_7k3m9p2x",
  "email": "alice@company.com",
  "roles": ["admin", "editor"],
  "workspaces": [
    {
      "id": "ws_abc123",
      "name": "Acme Corp",
      "permissions": ["read", "write", "delete"],
      "last_accessed": "2024-12-11T10:30:00Z"
    }
  ],
  "preferences": {
    "theme": "dark",
    "language": "fr",
    "timezone": "Europe/Paris",
    "keyboard_shortcuts": {
      "save": "Ctrl+S",
      "search": "Ctrl+K"
    }
  },
  "mfa": {
    "enabled": true,
    "verified_at": "2024-12-11T09:15:00Z",
    "method": "totp"
  },
  "devices": [
    {
      "id": "dev_xyz789",
      "type": "desktop",
      "os": "macOS 14.1",
      "browser": "Chrome 120",
      "ip": "192.168.1.10",
      "last_seen": "2024-12-11T10:45:00Z"
    }
  ],
  "metadata": {
    "created_at": "2024-12-11T09:00:00Z",
    "last_activity": "2024-12-11T10:45:00Z",
    "activity_count": 47
  }
}
```

**Contrainte** : Structure imbriquée et évolutive (ajout de champs sans migration).

#### 3. **Expiration intelligente (Sliding Sessions)**

```
Comportement attendu :
- Session inactive > 30 min → expiration automatique
- Chaque activité → refresh du TTL (sliding)
- Option "Remember me" → TTL 30 jours
- Logout explicite → suppression immédiate
```

#### 4. **Géo-réplication et latence**

```
Contrainte multi-région :
- Utilisateur à Tokyo : read depuis datacenter Tokyo
- Utilisateur voyage Paris → Tokyo : session disponible immédiatement
- Cohérence éventuelle acceptable (< 100ms)
```

---

## 2. Analyse des alternatives

### Option 1 : Base de données relationnelle (PostgreSQL)

```sql
CREATE TABLE sessions (
    session_id VARCHAR(255) PRIMARY KEY,
    user_id UUID NOT NULL,
    data JSONB NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_sessions_expires ON sessions(expires_at);
CREATE INDEX idx_sessions_user ON sessions(user_id);
```

**Avantages** :
- ✅ Persistance durable
- ✅ ACID complet
- ✅ Requêtes SQL puissantes

**Inconvénients** :
- ❌ Latence : 5-20ms minimum (même avec index)
- ❌ Scaling complexe (sharding, replication lag)
- ❌ Coût des écritures : chaque activité = UPDATE
- ❌ Cleanup manuel des sessions expirées (background job)

**Verdict** : ❌ **Non adapté** pour lecture ultra-fréquente (chaque requête HTTP).

---

### Option 2 : Cache applicatif (Memcached)

```python
memcached.set(session_id, session_data, ttl=1800)
session = memcached.get(session_id)
```

**Avantages** :
- ✅ Latence sub-milliseconde
- ✅ Simple à déployer
- ✅ TTL natif

**Inconvénients** :
- ❌ Valeurs en bytes uniquement (pas de manipulation JSON native)
- ❌ Pas de commandes atomiques avancées
- ❌ Pas de persistance (perte données si crash)
- ❌ Pas de réplication multi-datacenter native

**Verdict** : ⚠️ **Possible mais limité** (pas de RedisJSON, pas de Streams pour audit).

---

### Option 3 : Redis avec Hashes (sans RedisJSON)

```python
redis.hset(f"session:{session_id}", mapping={
    "user_id": "usr_123",
    "email": "alice@company.com",
    "roles": json.dumps(["admin"]),  # ❌ Sérialisation manuelle
    "workspaces": json.dumps([...])  # ❌ Pas de requêtes imbriquées
})
```

**Avantages** :
- ✅ Latence sub-milliseconde
- ✅ TTL natif avec `EXPIRE`
- ✅ Réplication et Cluster

**Inconvénients** :
- ❌ Structures imbriquées = JSON stringifié (pas de manipulation atomique)
- ❌ Impossible de modifier `preferences.theme` sans tout re-écrire
- ❌ Pas de requêtes JSON (ex: trouver toutes sessions avec `mfa.enabled=true`)

**Verdict** : ⚠️ **Fonctionnel mais sous-optimal** (sérialisation manuelle coûteuse).

---

### Option 4 : Redis avec RedisJSON ✅

```python
redis.json().set(f"session:{session_id}", "$", session_data)
redis.expire(f"session:{session_id}", 1800)

# Modification atomique d'un sous-champ
redis.json().set(f"session:{session_id}", "$.preferences.theme", "dark")

# Requête JSON (avec RediSearch si besoin)
redis.ft().search("@mfa_enabled:true")
```

**Avantages** :
- ✅ Manipulation JSON native et atomique
- ✅ Mise à jour de sous-champs sans sérialiser tout l'objet
- ✅ Latence < 1ms (in-memory)
- ✅ TTL natif avec sliding window facile
- ✅ Réplication multi-région (Sentinel, Cluster, Active-Active)
- ✅ Persistance optionnelle (RDB/AOF)
- ✅ Audit trail possible (Redis Streams)

**Inconvénients** :
- ⚠️ Dépendance à Redis Stack (RedisJSON module)
- ⚠️ Consommation mémoire plus élevée que Hashes
- ⚠️ Coût licensing si Redis Enterprise

**Verdict** : ✅ **Solution optimale** pour notre use case.

---

## 3. Architecture proposée

### 3.1 Vue d'ensemble

```
┌──────────────┐
│   Client     │
│  (Browser)   │
└──────┬───────┘
       │ HTTPS + Cookie (session_id)
       ▼
┌──────────────────────────────────────────┐
│         Load Balancer (Nginx)            │
│  - TLS termination                       │
│  - Round-robin (no sticky sessions)      │
└──────┬───────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│      Application Servers (Stateless)        │
│  - Node.js / Python / Go                    │
│  - Décodage JWT + lookup session            │
│  - Business logic                           │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│         Redis Session Store Layer            │
│  ┌──────────────┐  ┌──────────────┐          │
│  │ Redis Master │──│ Redis Replica│          │
│  │  (Primary)   │  │  (Read-only) │          │
│  └──────────────┘  └──────────────┘          │
│          │                                   │
│  ┌───────▼────────┐                          │
│  │ Redis Sentinel │ (Failover automatique)   │
│  └────────────────┘                          │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│      Redis Streams (Audit Log)               │
│  - Session créée                             │
│  - Session modifiée                          │
│  - Session expirée / logout                  │
└──────────────────────────────────────────────┘
```

### 3.2 Flux de données

#### **Connexion utilisateur (Login)**

```
1. POST /api/auth/login
   ├─ Validation credentials (DB)
   ├─ Génération session_id (UUID v4)
   ├─ Création session Redis:
   │  └─ JSON.SET session:{session_id} $ {data}
   │  └─ EXPIRE session:{session_id} 1800
   ├─ Génération JWT signé (contient session_id)
   └─ Set-Cookie: session_token=JWT; HttpOnly; Secure

Latence : ~10ms (DB) + ~1ms (Redis) = ~11ms
```

#### **Requête authentifiée (Request)**

```
1. GET /api/dashboard
   ├─ Extract JWT from Cookie
   ├─ Verify JWT signature
   ├─ Extract session_id from JWT
   ├─ JSON.GET session:{session_id}
   │  └─ Si non trouvé → 401 Unauthorized
   ├─ Refresh TTL (sliding):
   │  └─ EXPIRE session:{session_id} 1800
   ├─ Mise à jour last_activity:
   │  └─ JSON.SET session:{session_id} $.metadata.last_activity NOW()
   └─ Traitement business logic

Latence : ~0.5ms (Redis GET) + ~0.2ms (EXPIRE) = ~0.7ms
```

#### **Déconnexion (Logout)**

```
1. POST /api/auth/logout
   ├─ Extract session_id from JWT
   ├─ DEL session:{session_id}
   ├─ XADD sessions:audit * session_id {id} event logout
   └─ Clear-Cookie: session_token

Latence : ~0.5ms (Redis DEL)
```

### 3.3 Décisions architecturales clés

#### **Choix 1 : JWT + Redis (Hybrid Token)**

**Alternative envisagée** : JWT self-contained (toutes les données dans le token).

```javascript
// ❌ JWT self-contained
{
  "user_id": "usr_123",
  "roles": ["admin"],
  "workspaces": [{...}],  // Données volumineuses
  "exp": 1702300800
}
// Problème : Token volumineux (> 4KB), révocation impossible
```

```javascript
// ✅ JWT minimal + Redis lookup
{
  "session_id": "sess_abc123",
  "user_id": "usr_123",
  "exp": 1702300800  // Courte durée (5 min)
}
// Avantages : Token léger, révocation instantanée, sliding window facile
```

**Trade-off assumé** :
- ➕ Révocation immédiate (logout, security breach)
- ➕ Données session mutables sans re-générer JWT
- ➖ 1 appel Redis par requête (acceptable < 1ms)

---

#### **Choix 2 : RedisJSON vs Hash**

**Exemple de modification partielle** :

```python
# ❌ Avec Hash : Re-sérialisation complète
session = json.loads(redis.hget(f"session:{sid}", "data"))
session["preferences"]["theme"] = "dark"
redis.hset(f"session:{sid}", "data", json.dumps(session))
# Coût : 2 RTT + sérialisation CPU-intensive

# ✅ Avec RedisJSON : Modification atomique
redis.json().set(f"session:{sid}", "$.preferences.theme", "dark")
# Coût : 1 RTT, pas de sérialisation côté client
```

**Trade-off assumé** :
- ➕ Performance (1 RTT au lieu de 2)
- ➕ Atomicité native pour sous-champs
- ➖ Consommation mémoire +15-20% vs Hash

---

#### **Choix 3 : Sliding Window avec EXPIRE**

**Alternative envisagée** : Recalculer TTL à chaque requête.

```python
# ❌ Approche naïve
current_time = time.time()
session = redis.json().get(f"session:{sid}")
idle_time = current_time - session["metadata"]["last_activity"]
if idle_time > 1800:
    redis.delete(f"session:{sid}")
# Problème : 2 commandes + logique applicative

# ✅ Approche avec EXPIRE
redis.expire(f"session:{sid}", 1800)
# Redis gère automatiquement l'expiration
```

**Trade-off assumé** :
- ➕ Simplicité (1 commande)
- ➕ Performance (pas de calcul côté client)
- ➕ Fiabilité (expiration garantie même si app crash)

---

## 4. Modélisation des données

### 4.1 Schéma RedisJSON

**Clé** : `session:{session_id}`
**Type** : RedisJSON
**TTL** : 1800 secondes (30 minutes) ou 2592000 (30 jours si "Remember me")

```json
{
  "session_id": "sess_9k2m7x4p_20241211_103045",
  "user_id": "usr_7k3m9p2x",
  "email": "alice@company.com",
  "email_verified": true,

  "authentication": {
    "method": "password",
    "mfa_enabled": true,
    "mfa_verified_at": "2024-12-11T09:15:00Z",
    "mfa_method": "totp",
    "remember_me": false
  },

  "roles": ["workspace_admin", "editor"],

  "workspaces": [
    {
      "workspace_id": "ws_abc123",
      "workspace_name": "Acme Corp",
      "permissions": ["read", "write", "delete", "admin"],
      "last_accessed_at": "2024-12-11T10:30:00Z"
    },
    {
      "workspace_id": "ws_def456",
      "workspace_name": "Personal",
      "permissions": ["read", "write"],
      "last_accessed_at": "2024-12-10T15:20:00Z"
    }
  ],

  "preferences": {
    "ui": {
      "theme": "dark",
      "language": "fr",
      "timezone": "Europe/Paris",
      "date_format": "DD/MM/YYYY",
      "time_format": "24h"
    },
    "notifications": {
      "email": true,
      "push": true,
      "desktop": false
    },
    "keyboard_shortcuts": {
      "save": "Ctrl+S",
      "search": "Ctrl+K",
      "command_palette": "Ctrl+P"
    }
  },

  "devices": [
    {
      "device_id": "dev_xyz789",
      "device_type": "desktop",
      "os": "macOS 14.1",
      "browser": "Chrome 120.0.6099.129",
      "ip_address": "203.0.113.42",
      "user_agent": "Mozilla/5.0...",
      "first_seen_at": "2024-12-11T09:00:00Z",
      "last_seen_at": "2024-12-11T10:45:00Z"
    }
  ],

  "security": {
    "ip_whitelist": ["203.0.113.0/24"],
    "suspicious_activity_count": 0,
    "last_password_change": "2024-11-01T10:00:00Z",
    "failed_login_attempts": 0
  },

  "metadata": {
    "created_at": "2024-12-11T09:00:00Z",
    "last_activity_at": "2024-12-11T10:45:23Z",
    "activity_count": 47,
    "version": 1
  }
}
```

### 4.2 Patterns de nommage

```
session:{session_id}                    # Session principale
session:user:{user_id}:active           # Set des sessions actives par user
session:device:{device_id}:current      # Session actuelle par device
session:audit                           # Redis Stream pour audit log
```

**Rationale** :
- `session:{session_id}` : Lookup direct O(1)
- `session:user:{user_id}:active` : Permet de lister toutes les sessions d'un user (pour logout all devices)
- `session:audit` : Compliance et debugging

### 4.3 Index secondaires (optionnel avec RediSearch)

Si besoin de requêtes complexes (ex: trouver toutes les sessions MFA activé) :

```
FT.CREATE idx:sessions ON JSON PREFIX 1 session: SCHEMA
  $.user_id AS user_id TAG
  $.authentication.mfa_enabled AS mfa_enabled TAG
  $.metadata.last_activity_at AS last_activity NUMERIC SORTABLE
  $.devices[0].ip_address AS ip_address TEXT
```

**Usage** :
```python
# Trouver sessions MFA non vérifiées depuis > 1h
redis.ft("idx:sessions").search(
    "@mfa_enabled:{true} @last_activity:[0 (1702294800]"
)
```

---

## 5. Implémentation technique

### 5.1 Code Python (Production-Ready)

```python
"""
Session Store avec RedisJSON et TTL
Implémentation production-ready avec retry logic, circuit breaker, et monitoring
"""

import json
import time
import uuid
from datetime import datetime, timedelta
from typing import Optional, Dict, Any, List
from functools import wraps
import logging

import redis
from redis.commands.json.path import Path
from redis.exceptions import (
    ConnectionError,
    TimeoutError,
    RedisError,
    ResponseError
)

# Configuration logging structuré
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# ============================================================================
# Configuration
# ============================================================================

REDIS_CONFIG = {
    'host': 'localhost',
    'port': 6379,
    'db': 0,
    'password': None,
    'decode_responses': False,  # RedisJSON gère lui-même l'encodage
    'socket_connect_timeout': 2,
    'socket_timeout': 2,
    'socket_keepalive': True,
    'retry_on_timeout': True,
    'health_check_interval': 30,
    'max_connections': 50
}

# TTL par défaut : 30 minutes
DEFAULT_SESSION_TTL = 1800

# TTL "Remember me" : 30 jours
REMEMBER_ME_TTL = 2592000

# Configuration circuit breaker
CIRCUIT_BREAKER_THRESHOLD = 5  # Erreurs consécutives avant ouverture
CIRCUIT_BREAKER_TIMEOUT = 60   # Secondes avant retry


# ============================================================================
# Circuit Breaker (éviter cascading failures)
# ============================================================================

class CircuitBreaker:
    """Pattern Circuit Breaker pour protéger Redis"""

    def __init__(self, threshold: int = 5, timeout: int = 60):
        self.threshold = threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
                logger.info("Circuit breaker entering HALF_OPEN state")
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
                logger.info("Circuit breaker reset to CLOSED")
            return result

        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()

            if self.failure_count >= self.threshold:
                self.state = "OPEN"
                logger.error(f"Circuit breaker OPEN after {self.failure_count} failures")

            raise e


# ============================================================================
# Session Store Class
# ============================================================================

class SessionStore:
    """
    Session Store utilisant RedisJSON avec TTL automatique

    Features:
    - Sliding window sessions
    - Atomic updates de sous-champs JSON
    - Retry logic avec exponential backoff
    - Circuit breaker pour éviter cascading failures
    - Audit trail via Redis Streams
    - Multi-device support
    """

    def __init__(self, redis_config: Dict[str, Any] = None):
        """Initialisation du SessionStore"""
        config = redis_config or REDIS_CONFIG

        # Connexion pool Redis
        self.pool = redis.ConnectionPool(**config)
        self.redis = redis.Redis(connection_pool=self.pool)

        # Circuit breaker
        self.circuit_breaker = CircuitBreaker(
            threshold=CIRCUIT_BREAKER_THRESHOLD,
            timeout=CIRCUIT_BREAKER_TIMEOUT
        )

        # Vérifier que RedisJSON est disponible
        self._check_redisjson_available()

        logger.info("SessionStore initialized successfully")

    def _check_redisjson_available(self):
        """Vérifier que le module RedisJSON est chargé"""
        try:
            modules = self.redis.module_list()
            json_module = any(m[b'name'] == b'ReJSON' for m in modules)
            if not json_module:
                raise Exception("RedisJSON module not loaded")
        except Exception as e:
            logger.error(f"RedisJSON check failed: {e}")
            raise

    def _generate_session_id(self) -> str:
        """Générer un session_id unique et sécurisé"""
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        random_part = uuid.uuid4().hex[:12]
        return f"sess_{random_part}_{timestamp}"

    def _retry_with_backoff(self, func, *args, max_retries=3, **kwargs):
        """Retry avec exponential backoff"""
        for attempt in range(max_retries):
            try:
                return self.circuit_breaker.call(func, *args, **kwargs)
            except (ConnectionError, TimeoutError) as e:
                if attempt == max_retries - 1:
                    logger.error(f"Max retries reached: {e}")
                    raise

                wait_time = (2 ** attempt) + (time.time() % 1)
                logger.warning(f"Retry {attempt + 1}/{max_retries} after {wait_time:.2f}s")
                time.sleep(wait_time)

    # ========================================================================
    # Core Operations
    # ========================================================================

    def create_session(
        self,
        user_id: str,
        email: str,
        roles: List[str],
        workspaces: List[Dict],
        device_info: Dict,
        remember_me: bool = False,
        mfa_verified: bool = False
    ) -> str:
        """
        Créer une nouvelle session utilisateur

        Args:
            user_id: Identifiant unique utilisateur
            email: Email utilisateur
            roles: Liste des rôles (ex: ["admin", "editor"])
            workspaces: Liste des workspaces accessibles
            device_info: Informations sur le device (os, browser, ip)
            remember_me: Si True, TTL = 30 jours au lieu de 30 min
            mfa_verified: Si True, MFA a été vérifié

        Returns:
            session_id généré
        """
        session_id = self._generate_session_id()
        session_key = f"session:{session_id}"

        now = datetime.utcnow().isoformat() + "Z"

        session_data = {
            "session_id": session_id,
            "user_id": user_id,
            "email": email,
            "email_verified": True,

            "authentication": {
                "method": "password",
                "mfa_enabled": mfa_verified,
                "mfa_verified_at": now if mfa_verified else None,
                "mfa_method": "totp" if mfa_verified else None,
                "remember_me": remember_me
            },

            "roles": roles,
            "workspaces": workspaces,

            "preferences": {
                "ui": {
                    "theme": "light",
                    "language": "en",
                    "timezone": "UTC"
                },
                "notifications": {
                    "email": True,
                    "push": True
                }
            },

            "devices": [
                {
                    "device_id": device_info.get("device_id", str(uuid.uuid4())),
                    "device_type": device_info.get("type", "unknown"),
                    "os": device_info.get("os", "unknown"),
                    "browser": device_info.get("browser", "unknown"),
                    "ip_address": device_info.get("ip", "0.0.0.0"),
                    "user_agent": device_info.get("user_agent", ""),
                    "first_seen_at": now,
                    "last_seen_at": now
                }
            ],

            "security": {
                "ip_whitelist": [],
                "suspicious_activity_count": 0,
                "failed_login_attempts": 0
            },

            "metadata": {
                "created_at": now,
                "last_activity_at": now,
                "activity_count": 1,
                "version": 1
            }
        }

        ttl = REMEMBER_ME_TTL if remember_me else DEFAULT_SESSION_TTL

        try:
            # Créer la session avec JSON.SET
            self._retry_with_backoff(
                self.redis.json().set,
                session_key,
                Path.root_path(),
                session_data
            )

            # Définir le TTL
            self._retry_with_backoff(
                self.redis.expire,
                session_key,
                ttl
            )

            # Ajouter à la liste des sessions actives de l'utilisateur
            user_sessions_key = f"session:user:{user_id}:active"
            self._retry_with_backoff(
                self.redis.sadd,
                user_sessions_key,
                session_id
            )
            self._retry_with_backoff(
                self.redis.expire,
                user_sessions_key,
                ttl
            )

            # Audit log via Redis Streams
            self._audit_log("session_created", {
                "session_id": session_id,
                "user_id": user_id,
                "remember_me": remember_me
            })

            logger.info(f"Session created: {session_id} for user {user_id}")
            return session_id

        except RedisError as e:
            logger.error(f"Failed to create session: {e}")
            raise

    def get_session(self, session_id: str) -> Optional[Dict]:
        """
        Récupérer une session par son ID

        Returns:
            Session data (dict) ou None si non trouvée
        """
        session_key = f"session:{session_id}"

        try:
            session_data = self._retry_with_backoff(
                self.redis.json().get,
                session_key
            )

            if session_data is None:
                logger.warning(f"Session not found: {session_id}")
                return None

            logger.debug(f"Session retrieved: {session_id}")
            return session_data

        except RedisError as e:
            logger.error(f"Failed to get session {session_id}: {e}")
            raise

    def refresh_session(self, session_id: str) -> bool:
        """
        Refresh le TTL de la session (sliding window)
        et met à jour last_activity_at

        Returns:
            True si succès, False si session non trouvée
        """
        session_key = f"session:{session_id}"

        try:
            # Vérifier que la session existe
            exists = self._retry_with_backoff(
                self.redis.exists,
                session_key
            )

            if not exists:
                logger.warning(f"Cannot refresh non-existent session: {session_id}")
                return False

            # Récupérer remember_me pour déterminer le TTL
            remember_me = self._retry_with_backoff(
                self.redis.json().get,
                session_key,
                Path("$.authentication.remember_me")
            )

            ttl = REMEMBER_ME_TTL if remember_me else DEFAULT_SESSION_TTL

            # Pipeline pour atomicité
            pipe = self.redis.pipeline()

            # Refresh TTL
            pipe.expire(session_key, ttl)

            # Update last_activity_at
            now = datetime.utcnow().isoformat() + "Z"
            pipe.json().set(
                session_key,
                Path("$.metadata.last_activity_at"),
                now
            )

            # Incrémenter activity_count
            pipe.json().numincrby(
                session_key,
                Path("$.metadata.activity_count"),
                1
            )

            self._retry_with_backoff(pipe.execute)

            logger.debug(f"Session refreshed: {session_id}")
            return True

        except RedisError as e:
            logger.error(f"Failed to refresh session {session_id}: {e}")
            raise

    def update_session_field(
        self,
        session_id: str,
        json_path: str,
        value: Any
    ) -> bool:
        """
        Mettre à jour un champ spécifique de la session (atomic)

        Args:
            session_id: ID de la session
            json_path: Chemin JSONPath (ex: "$.preferences.ui.theme")
            value: Nouvelle valeur

        Returns:
            True si succès, False si session non trouvée
        """
        session_key = f"session:{session_id}"

        try:
            result = self._retry_with_backoff(
                self.redis.json().set,
                session_key,
                Path(json_path),
                value
            )

            if result is None:
                logger.warning(f"Session not found for update: {session_id}")
                return False

            # Refresh la session après modification
            self.refresh_session(session_id)

            logger.info(f"Session updated: {session_id} at {json_path}")
            return True

        except ResponseError as e:
            logger.error(f"Invalid JSON path {json_path}: {e}")
            return False
        except RedisError as e:
            logger.error(f"Failed to update session {session_id}: {e}")
            raise

    def delete_session(self, session_id: str) -> bool:
        """
        Supprimer une session (logout)

        Returns:
            True si supprimée, False si non trouvée
        """
        session_key = f"session:{session_id}"

        try:
            # Récupérer user_id avant suppression (pour nettoyage)
            user_id = self._retry_with_backoff(
                self.redis.json().get,
                session_key,
                Path("$.user_id")
            )

            # Supprimer la session
            deleted = self._retry_with_backoff(
                self.redis.delete,
                session_key
            )

            if deleted == 0:
                logger.warning(f"Session not found for deletion: {session_id}")
                return False

            # Retirer de la liste des sessions actives
            if user_id:
                user_sessions_key = f"session:user:{user_id}:active"
                self._retry_with_backoff(
                    self.redis.srem,
                    user_sessions_key,
                    session_id
                )

            # Audit log
            self._audit_log("session_deleted", {
                "session_id": session_id,
                "user_id": user_id
            })

            logger.info(f"Session deleted: {session_id}")
            return True

        except RedisError as e:
            logger.error(f"Failed to delete session {session_id}: {e}")
            raise

    def get_user_sessions(self, user_id: str) -> List[str]:
        """
        Récupérer toutes les sessions actives d'un utilisateur

        Returns:
            Liste des session_ids
        """
        user_sessions_key = f"session:user:{user_id}:active"

        try:
            sessions = self._retry_with_backoff(
                self.redis.smembers,
                user_sessions_key
            )

            # Convertir bytes en str si nécessaire
            session_ids = [s.decode() if isinstance(s, bytes) else s for s in sessions]

            logger.debug(f"Found {len(session_ids)} active sessions for user {user_id}")
            return session_ids

        except RedisError as e:
            logger.error(f"Failed to get user sessions for {user_id}: {e}")
            raise

    def logout_all_devices(self, user_id: str) -> int:
        """
        Déconnecter tous les devices d'un utilisateur

        Returns:
            Nombre de sessions supprimées
        """
        try:
            session_ids = self.get_user_sessions(user_id)

            if not session_ids:
                logger.info(f"No active sessions for user {user_id}")
                return 0

            # Pipeline pour suppression batch
            pipe = self.redis.pipeline()
            for session_id in session_ids:
                pipe.delete(f"session:{session_id}")

            results = self._retry_with_backoff(pipe.execute)
            deleted_count = sum(results)

            # Nettoyer la liste des sessions actives
            user_sessions_key = f"session:user:{user_id}:active"
            self._retry_with_backoff(
                self.redis.delete,
                user_sessions_key
            )

            # Audit log
            self._audit_log("logout_all_devices", {
                "user_id": user_id,
                "sessions_deleted": deleted_count
            })

            logger.info(f"Logged out {deleted_count} devices for user {user_id}")
            return deleted_count

        except RedisError as e:
            logger.error(f"Failed to logout all devices for {user_id}: {e}")
            raise

    # ========================================================================
    # Audit Trail
    # ========================================================================

    def _audit_log(self, event: str, data: Dict):
        """Enregistrer un événement dans Redis Streams pour audit"""
        try:
            audit_stream = "session:audit"

            # Ajouter timestamp
            data["timestamp"] = datetime.utcnow().isoformat() + "Z"
            data["event"] = event

            # Convertir dict en format Redis Streams (flat dict)
            stream_data = {k: json.dumps(v) if isinstance(v, (dict, list)) else str(v)
                          for k, v in data.items()}

            self.redis.xadd(audit_stream, stream_data, maxlen=100000)

        except RedisError as e:
            # Ne pas bloquer l'opération principale si audit fail
            logger.warning(f"Audit log failed: {e}")

    # ========================================================================
    # Monitoring & Diagnostics
    # ========================================================================

    def get_stats(self) -> Dict:
        """Récupérer des statistiques sur les sessions"""
        try:
            # Scan pour compter les sessions (éviter KEYS *)
            session_count = 0
            cursor = 0

            while True:
                cursor, keys = self.redis.scan(
                    cursor=cursor,
                    match="session:sess_*",
                    count=1000
                )
                session_count += len(keys)

                if cursor == 0:
                    break

            # Métriques Redis
            info = self.redis.info("memory")

            stats = {
                "total_sessions": session_count,
                "memory_used_mb": info['used_memory'] / (1024 * 1024),
                "memory_peak_mb": info['used_memory_peak'] / (1024 * 1024),
                "fragmentation_ratio": info.get('mem_fragmentation_ratio', 0),
                "circuit_breaker_state": self.circuit_breaker.state,
                "circuit_breaker_failures": self.circuit_breaker.failure_count
            }

            return stats

        except RedisError as e:
            logger.error(f"Failed to get stats: {e}")
            return {}

    def close(self):
        """Fermer proprement les connexions"""
        try:
            self.pool.disconnect()
            logger.info("SessionStore closed")
        except Exception as e:
            logger.error(f"Error closing SessionStore: {e}")


# ============================================================================
# Exemple d'utilisation
# ============================================================================

if __name__ == "__main__":
    # Initialiser le store
    store = SessionStore()

    # Créer une session
    session_id = store.create_session(
        user_id="usr_123456",
        email="alice@example.com",
        roles=["admin", "editor"],
        workspaces=[
            {
                "workspace_id": "ws_abc",
                "workspace_name": "Acme Corp",
                "permissions": ["read", "write", "admin"]
            }
        ],
        device_info={
            "type": "desktop",
            "os": "macOS 14.1",
            "browser": "Chrome 120",
            "ip": "203.0.113.42",
            "user_agent": "Mozilla/5.0..."
        },
        remember_me=False,
        mfa_verified=True
    )

    print(f"✅ Session created: {session_id}")

    # Récupérer la session
    session = store.get_session(session_id)
    print(f"✅ Session data: {json.dumps(session, indent=2)}")

    # Rafraîchir la session (sliding window)
    store.refresh_session(session_id)
    print(f"✅ Session refreshed")

    # Modifier un champ (atomic update)
    store.update_session_field(
        session_id,
        "$.preferences.ui.theme",
        "dark"
    )
    print(f"✅ Theme updated to dark")

    # Statistiques
    stats = store.get_stats()
    print(f"✅ Stats: {json.dumps(stats, indent=2)}")

    # Supprimer la session
    store.delete_session(session_id)
    print(f"✅ Session deleted")

    # Fermer le store
    store.close()
```

### 5.2 Code Node.js (Production-Ready)

```javascript
/**
 * Session Store avec RedisJSON et TTL
 * Implémentation Node.js avec ioredis et retry logic
 */

const Redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');

// ============================================================================
// Configuration
// ============================================================================

const REDIS_CONFIG = {
  host: 'localhost',
  port: 6379,
  db: 0,
  password: null,
  retryStrategy: (times) => {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  enableOfflineQueue: false,
  lazyConnect: false,
  connectionName: 'session-store',
};

const DEFAULT_SESSION_TTL = 1800; // 30 minutes
const REMEMBER_ME_TTL = 2592000;  // 30 jours

// ============================================================================
// Session Store Class
// ============================================================================

class SessionStore {
  constructor(config = REDIS_CONFIG) {
    this.redis = new Redis(config);

    // Circuit breaker state
    this.circuitBreaker = {
      state: 'CLOSED',
      failureCount: 0,
      lastFailureTime: null,
      threshold: 5,
      timeout: 60000, // 60 secondes
    };

    // Event handlers
    this.redis.on('error', (err) => {
      console.error('[Redis] Connection error:', err);
      this._handleCircuitBreaker();
    });

    this.redis.on('connect', () => {
      console.log('[Redis] Connected successfully');
      this._resetCircuitBreaker();
    });

    // Vérifier RedisJSON
    this._checkRedisJSON();
  }

  async _checkRedisJSON() {
    try {
      const modules = await this.redis.call('MODULE', 'LIST');
      const hasRedisJSON = modules.some(m => m[1] === 'ReJSON');

      if (!hasRedisJSON) {
        throw new Error('RedisJSON module not loaded');
      }

      console.log('[Redis] RedisJSON module detected');
    } catch (err) {
      console.error('[Redis] RedisJSON check failed:', err);
      throw err;
    }
  }

  _generateSessionId() {
    const timestamp = new Date().toISOString().replace(/[-:T.]/g, '').slice(0, 14);
    const random = uuidv4().replace(/-/g, '').slice(0, 12);
    return `sess_${random}_${timestamp}`;
  }

  _handleCircuitBreaker() {
    this.circuitBreaker.failureCount++;
    this.circuitBreaker.lastFailureTime = Date.now();

    if (this.circuitBreaker.failureCount >= this.circuitBreaker.threshold) {
      this.circuitBreaker.state = 'OPEN';
      console.error('[Circuit Breaker] State changed to OPEN');
    }
  }

  _resetCircuitBreaker() {
    this.circuitBreaker.state = 'CLOSED';
    this.circuitBreaker.failureCount = 0;
    this.circuitBreaker.lastFailureTime = null;
  }

  _checkCircuitBreaker() {
    if (this.circuitBreaker.state === 'OPEN') {
      const timeSinceFailure = Date.now() - this.circuitBreaker.lastFailureTime;

      if (timeSinceFailure > this.circuitBreaker.timeout) {
        this.circuitBreaker.state = 'HALF_OPEN';
        console.log('[Circuit Breaker] State changed to HALF_OPEN');
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }
  }

  // ==========================================================================
  // Core Operations
  // ==========================================================================

  async createSession({
    userId,
    email,
    roles = [],
    workspaces = [],
    deviceInfo = {},
    rememberMe = false,
    mfaVerified = false,
  }) {
    this._checkCircuitBreaker();

    const sessionId = this._generateSessionId();
    const sessionKey = `session:${sessionId}`;
    const now = new Date().toISOString();

    const sessionData = {
      session_id: sessionId,
      user_id: userId,
      email,
      email_verified: true,

      authentication: {
        method: 'password',
        mfa_enabled: mfaVerified,
        mfa_verified_at: mfaVerified ? now : null,
        mfa_method: mfaVerified ? 'totp' : null,
        remember_me: rememberMe,
      },

      roles,
      workspaces,

      preferences: {
        ui: {
          theme: 'light',
          language: 'en',
          timezone: 'UTC',
        },
        notifications: {
          email: true,
          push: true,
        },
      },

      devices: [
        {
          device_id: deviceInfo.deviceId || uuidv4(),
          device_type: deviceInfo.type || 'unknown',
          os: deviceInfo.os || 'unknown',
          browser: deviceInfo.browser || 'unknown',
          ip_address: deviceInfo.ip || '0.0.0.0',
          user_agent: deviceInfo.userAgent || '',
          first_seen_at: now,
          last_seen_at: now,
        },
      ],

      security: {
        ip_whitelist: [],
        suspicious_activity_count: 0,
        failed_login_attempts: 0,
      },

      metadata: {
        created_at: now,
        last_activity_at: now,
        activity_count: 1,
        version: 1,
      },
    };

    const ttl = rememberMe ? REMEMBER_ME_TTL : DEFAULT_SESSION_TTL;

    try {
      // Pipeline pour atomicité
      const pipeline = this.redis.pipeline();

      // Créer la session
      pipeline.call('JSON.SET', sessionKey, '$', JSON.stringify(sessionData));

      // Définir TTL
      pipeline.expire(sessionKey, ttl);

      // Ajouter aux sessions actives de l'utilisateur
      const userSessionsKey = `session:user:${userId}:active`;
      pipeline.sadd(userSessionsKey, sessionId);
      pipeline.expire(userSessionsKey, ttl);

      await pipeline.exec();

      // Audit log
      await this._auditLog('session_created', {
        session_id: sessionId,
        user_id: userId,
        remember_me: rememberMe,
      });

      console.log(`[Session] Created: ${sessionId} for user ${userId}`);
      return sessionId;

    } catch (err) {
      console.error(`[Session] Failed to create session:`, err);
      throw err;
    }
  }

  async getSession(sessionId) {
    this._checkCircuitBreaker();

    const sessionKey = `session:${sessionId}`;

    try {
      const result = await this.redis.call('JSON.GET', sessionKey);

      if (!result) {
        console.warn(`[Session] Not found: ${sessionId}`);
        return null;
      }

      return JSON.parse(result);

    } catch (err) {
      console.error(`[Session] Failed to get session ${sessionId}:`, err);
      throw err;
    }
  }

  async refreshSession(sessionId) {
    this._checkCircuitBreaker();

    const sessionKey = `session:${sessionId}`;

    try {
      // Vérifier existence
      const exists = await this.redis.exists(sessionKey);
      if (!exists) {
        console.warn(`[Session] Cannot refresh non-existent: ${sessionId}`);
        return false;
      }

      // Récupérer remember_me
      const rememberMeResult = await this.redis.call(
        'JSON.GET',
        sessionKey,
        '$.authentication.remember_me'
      );
      const rememberMe = JSON.parse(rememberMeResult)[0];

      const ttl = rememberMe ? REMEMBER_ME_TTL : DEFAULT_SESSION_TTL;
      const now = new Date().toISOString();

      // Pipeline pour atomicité
      const pipeline = this.redis.pipeline();

      // Refresh TTL
      pipeline.expire(sessionKey, ttl);

      // Update last_activity_at
      pipeline.call(
        'JSON.SET',
        sessionKey,
        '$.metadata.last_activity_at',
        JSON.stringify(now)
      );

      // Incrémenter activity_count
      pipeline.call(
        'JSON.NUMINCRBY',
        sessionKey,
        '$.metadata.activity_count',
        1
      );

      await pipeline.exec();

      console.log(`[Session] Refreshed: ${sessionId}`);
      return true;

    } catch (err) {
      console.error(`[Session] Failed to refresh ${sessionId}:`, err);
      throw err;
    }
  }

  async updateSessionField(sessionId, jsonPath, value) {
    this._checkCircuitBreaker();

    const sessionKey = `session:${sessionId}`;

    try {
      const result = await this.redis.call(
        'JSON.SET',
        sessionKey,
        jsonPath,
        JSON.stringify(value)
      );

      if (!result) {
        console.warn(`[Session] Not found for update: ${sessionId}`);
        return false;
      }

      // Refresh après modification
      await this.refreshSession(sessionId);

      console.log(`[Session] Updated: ${sessionId} at ${jsonPath}`);
      return true;

    } catch (err) {
      console.error(`[Session] Failed to update ${sessionId}:`, err);
      throw err;
    }
  }

  async deleteSession(sessionId) {
    this._checkCircuitBreaker();

    const sessionKey = `session:${sessionId}`;

    try {
      // Récupérer user_id avant suppression
      const userIdResult = await this.redis.call(
        'JSON.GET',
        sessionKey,
        '$.user_id'
      );

      const userId = userIdResult ? JSON.parse(userIdResult)[0] : null;

      // Supprimer la session
      const deleted = await this.redis.del(sessionKey);

      if (deleted === 0) {
        console.warn(`[Session] Not found for deletion: ${sessionId}`);
        return false;
      }

      // Retirer de la liste des sessions actives
      if (userId) {
        const userSessionsKey = `session:user:${userId}:active`;
        await this.redis.srem(userSessionsKey, sessionId);
      }

      // Audit log
      await this._auditLog('session_deleted', {
        session_id: sessionId,
        user_id: userId,
      });

      console.log(`[Session] Deleted: ${sessionId}`);
      return true;

    } catch (err) {
      console.error(`[Session] Failed to delete ${sessionId}:`, err);
      throw err;
    }
  }

  async getUserSessions(userId) {
    this._checkCircuitBreaker();

    const userSessionsKey = `session:user:${userId}:active`;

    try {
      const sessions = await this.redis.smembers(userSessionsKey);
      console.log(`[Session] Found ${sessions.length} active for user ${userId}`);
      return sessions;

    } catch (err) {
      console.error(`[Session] Failed to get user sessions for ${userId}:`, err);
      throw err;
    }
  }

  async logoutAllDevices(userId) {
    this._checkCircuitBreaker();

    try {
      const sessionIds = await this.getUserSessions(userId);

      if (sessionIds.length === 0) {
        console.log(`[Session] No active sessions for user ${userId}`);
        return 0;
      }

      // Pipeline pour suppression batch
      const pipeline = this.redis.pipeline();

      sessionIds.forEach((sessionId) => {
        pipeline.del(`session:${sessionId}`);
      });

      const results = await pipeline.exec();
      const deletedCount = results.filter(([err, result]) => !err && result === 1).length;

      // Nettoyer la liste
      const userSessionsKey = `session:user:${userId}:active`;
      await this.redis.del(userSessionsKey);

      // Audit log
      await this._auditLog('logout_all_devices', {
        user_id: userId,
        sessions_deleted: deletedCount,
      });

      console.log(`[Session] Logged out ${deletedCount} devices for user ${userId}`);
      return deletedCount;

    } catch (err) {
      console.error(`[Session] Failed to logout all devices for ${userId}:`, err);
      throw err;
    }
  }

  // ==========================================================================
  // Audit Trail
  // ==========================================================================

  async _auditLog(event, data) {
    try {
      const auditStream = 'session:audit';
      const timestamp = new Date().toISOString();

      const streamData = {
        event,
        timestamp,
        ...Object.entries(data).reduce((acc, [key, value]) => {
          acc[key] = typeof value === 'object' ? JSON.stringify(value) : String(value);
          return acc;
        }, {}),
      };

      await this.redis.xadd(auditStream, 'MAXLEN', '~', '100000', '*', ...Object.entries(streamData).flat());

    } catch (err) {
      // Ne pas bloquer l'opération principale
      console.warn(`[Audit] Log failed:`, err);
    }
  }

  // ==========================================================================
  // Monitoring
  // ==========================================================================

  async getStats() {
    try {
      let sessionCount = 0;
      let cursor = '0';

      // Scan pour compter les sessions
      do {
        const [newCursor, keys] = await this.redis.scan(
          cursor,
          'MATCH', 'session:sess_*',
          'COUNT', 1000
        );
        cursor = newCursor;
        sessionCount += keys.length;
      } while (cursor !== '0');

      // Métriques Redis
      const info = await this.redis.info('memory');
      const memoryLines = info.split('\r\n');

      const getMemoryValue = (key) => {
        const line = memoryLines.find(l => l.startsWith(key));
        return line ? parseFloat(line.split(':')[1]) : 0;
      };

      return {
        total_sessions: sessionCount,
        memory_used_mb: getMemoryValue('used_memory') / (1024 * 1024),
        memory_peak_mb: getMemoryValue('used_memory_peak') / (1024 * 1024),
        fragmentation_ratio: getMemoryValue('mem_fragmentation_ratio'),
        circuit_breaker_state: this.circuitBreaker.state,
        circuit_breaker_failures: this.circuitBreaker.failureCount,
      };

    } catch (err) {
      console.error('[Stats] Failed to get stats:', err);
      return {};
    }
  }

  async close() {
    try {
      await this.redis.quit();
      console.log('[Redis] Connection closed');
    } catch (err) {
      console.error('[Redis] Error closing connection:', err);
    }
  }
}

// ============================================================================
// Exemple d'utilisation
// ============================================================================

(async () => {
  const store = new SessionStore();

  try {
    // Créer une session
    const sessionId = await store.createSession({
      userId: 'usr_123456',
      email: 'alice@example.com',
      roles: ['admin', 'editor'],
      workspaces: [
        {
          workspace_id: 'ws_abc',
          workspace_name: 'Acme Corp',
          permissions: ['read', 'write', 'admin'],
        },
      ],
      deviceInfo: {
        type: 'desktop',
        os: 'macOS 14.1',
        browser: 'Chrome 120',
        ip: '203.0.113.42',
        userAgent: 'Mozilla/5.0...',
      },
      rememberMe: false,
      mfaVerified: true,
    });

    console.log(`✅ Session created: ${sessionId}`);

    // Récupérer la session
    const session = await store.getSession(sessionId);
    console.log(`✅ Session data:`, JSON.stringify(session, null, 2));

    // Rafraîchir la session
    await store.refreshSession(sessionId);
    console.log(`✅ Session refreshed`);

    // Modifier un champ
    await store.updateSessionField(sessionId, '$.preferences.ui.theme', 'dark');
    console.log(`✅ Theme updated to dark`);

    // Statistiques
    const stats = await store.getStats();
    console.log(`✅ Stats:`, JSON.stringify(stats, null, 2));

    // Supprimer la session
    await store.deleteSession(sessionId);
    console.log(`✅ Session deleted`);

  } catch (err) {
    console.error('Error:', err);
  } finally {
    await store.close();
  }
})();

module.exports = SessionStore;
```

---

## 6. Monitoring et métriques

### 6.1 KPIs à surveiller

```python
# Dashboard Grafana - Métriques clés

# 1. Latence des opérations
redis_command_duration_seconds{command="JSON.GET"} # p50, p95, p99

# 2. Throughput
rate(redis_commands_processed_total{command="JSON.GET"}[1m])

# 3. Sessions actives
count(redis_key_size{key=~"session:sess_.*"})

# 4. Hit ratio (proxy = succès get_session)
rate(session_get_success_total[1m]) / rate(session_get_total[1m])

# 5. TTL moyen des sessions
avg(redis_key_ttl{key=~"session:sess_.*"})

# 6. Memory usage
redis_memory_used_bytes / redis_memory_max_bytes

# 7. Circuit breaker state
circuit_breaker_state{component="session_store"}
```

### 6.2 Alertes critiques

```yaml
# Prometheus Alerting Rules

groups:
  - name: session_store_alerts
    interval: 30s
    rules:

      # Latence excessive
      - alert: SessionStorHighLatency
        expr: histogram_quantile(0.99, redis_command_duration_seconds{command="JSON.GET"}) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Session store latency p99 > 10ms"
          description: "{{ $value }}s latency detected"

      # Taux d'erreur élevé
      - alert: SessionStoreHighErrorRate
        expr: rate(session_errors_total[5m]) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Session store error rate > 1%"

      # Circuit breaker ouvert
      - alert: SessionStoreCircuitBreakerOpen
        expr: circuit_breaker_state{component="session_store"} == 2  # OPEN
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Session store circuit breaker OPEN"

      # Memory usage critique
      - alert: SessionStoreMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Session store memory > 90%"

      # Sessions expirées anormalement
      - alert: SessionStoreHighEvictionRate
        expr: rate(redis_evicted_keys_total[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High session eviction rate detected"
```

---

## 7. Scalabilité et évolutions

### 7.1 Phase 1 : Single Instance (MVP)

```
Redis Standalone
├─ 8GB RAM
├─ RDB persistence (save 900 1)
└─ Capacité : ~1M sessions actives

Coût : ~$50/mois (AWS ElastiCache t3.medium)
Disponibilité : 99.5%
```

### 7.2 Phase 2 : Master-Replica (HA Basique)

```
Redis Master-Replica
├─ Master : 16GB RAM (writes)
├─ Replica : 16GB RAM (reads)
├─ Sentinel : 3 instances (failover automatique)
└─ Capacité : ~2M sessions actives

Coût : ~$150/mois
Disponibilité : 99.9%
Failover time : ~30 secondes
```

### 7.3 Phase 3 : Redis Cluster (Scaling Horizontal)

```
Redis Cluster (6 nodes)
├─ 3 Masters : 16GB RAM each (sharding)
├─ 3 Replicas : 16GB RAM each (HA)
├─ Hash slots : 16384 total
└─ Capacité : ~6M sessions actives

Coût : ~$500/mois
Disponibilité : 99.95%
Throughput : 500k ops/sec
```

**Attention** : Multi-key operations limitées (ex: `getUserSessions` nécessite client-side aggregation).

### 7.4 Phase 4 : Multi-Région (Active-Active)

```
Global Distribution (Redis Enterprise Active-Active)
├─ Région US-East : 3-node cluster
├─ Région EU-West : 3-node cluster
├─ Région AP-Southeast : 3-node cluster
├─ CRDT conflict resolution
└─ Capacité : ~20M sessions actives

Coût : ~$2000/mois
Disponibilité : 99.99%
Latence cross-region : < 100ms
```

**Trade-off** : Cohérence éventuelle (acceptable pour sessions).

### 7.5 Optimisations avancées

#### **Compression des sessions (RedisJSON + ZSTD)**

```python
import zstandard as zstd

# Compresser avant stockage (si session > 10KB)
def compress_session(session_data):
    json_str = json.dumps(session_data)
    if len(json_str) > 10240:  # 10KB threshold
        compressor = zstd.ZstdCompressor(level=3)
        compressed = compressor.compress(json_str.encode())
        return {
            "compressed": True,
            "data": compressed.hex()
        }
    return {
        "compressed": False,
        "data": session_data
    }

# Économie mémoire : ~40-60% pour sessions riches
```

#### **Tiering de mémoire (Redis Enterprise)**

```
Hot tier (RAM) : Sessions actives < 5 min
├─ 90% des accès
└─ Latence < 1ms

Warm tier (Flash/SSD) : Sessions inactives 5-30 min
├─ 10% des accès
└─ Latence < 5ms

Économie : ~70% du coût mémoire
```

---

## 8. Cas limites et gestion d'erreurs

### 8.1 Session expirée pendant traitement

```python
# ❌ Race condition
def process_request(session_id):
    session = store.get_session(session_id)  # Session valide
    # ... traitement long (5 sec) ...
    store.refresh_session(session_id)  # ⚠️ Peut avoir expiré entre-temps
    return response

# ✅ Vérification + retry
def process_request_safe(session_id):
    session = store.get_session(session_id)
    if not session:
        raise Unauthorized("Session expired")

    # Refresh immédiatement
    refreshed = store.refresh_session(session_id)
    if not refreshed:
        raise Unauthorized("Session expired during processing")

    # ... traitement ...

    # Re-vérifier avant commit
    if store.get_session(session_id):
        return response
    else:
        raise Unauthorized("Session expired")
```

### 8.2 Corruption de données JSON

```python
# Validation avec JSON Schema
from jsonschema import validate, ValidationError

SESSION_SCHEMA = {
    "type": "object",
    "required": ["session_id", "user_id", "email"],
    "properties": {
        "session_id": {"type": "string"},
        "user_id": {"type": "string"},
        "email": {"type": "string", "format": "email"},
        # ...
    }
}

def get_session_safe(session_id):
    try:
        session = store.get_session(session_id)
        if session:
            validate(instance=session, schema=SESSION_SCHEMA)
        return session
    except ValidationError as e:
        logger.error(f"Session data corrupted: {session_id} - {e}")
        # Supprimer session corrompue
        store.delete_session(session_id)
        return None
```

### 8.3 Dépassement de quota mémoire

```python
# Monitoring proactif avec éviction préventive
def check_memory_pressure():
    stats = store.get_stats()
    memory_ratio = stats["memory_used_mb"] / MAX_MEMORY_MB

    if memory_ratio > 0.85:  # Seuil 85%
        logger.warning("Memory pressure detected, evicting idle sessions")
        evict_idle_sessions(idle_threshold=600)  # > 10 min inactives

    if memory_ratio > 0.95:  # Seuil critique 95%
        logger.critical("Critical memory pressure, emergency eviction")
        evict_idle_sessions(idle_threshold=300)  # > 5 min inactives
```

---

## 9. Sécurité

### 9.1 Authentification et ACLs

```bash
# Redis ACL pour session store (Redis 6+)
ACL SETUSER session_writer on >strong_password ~session:* +json.set +expire +sadd +xadd
ACL SETUSER session_reader on >strong_password ~session:* +json.get +exists +ttl +smembers

# Application utilise session_writer pour create/update
# Load balancer utilise session_reader pour get_session
```

### 9.2 Chiffrement TLS

```python
REDIS_CONFIG_TLS = {
    'host': 'redis.example.com',
    'port': 6380,
    'ssl': True,
    'ssl_cert_reqs': 'required',
    'ssl_ca_certs': '/path/to/ca-cert.pem',
    'ssl_certfile': '/path/to/client-cert.pem',
    'ssl_keyfile': '/path/to/client-key.pem',
}

# Impact latence : +0.5-1ms (acceptable)
```

### 9.3 Données sensibles (PII)

```python
# ❌ Stocker données sensibles en clair
session_data = {
    "email": "alice@example.com",
    "ssn": "123-45-6789",  # ⚠️ Données sensibles
    "credit_card": "4111-1111-1111-1111"
}

# ✅ Chiffrement ou référence
from cryptography.fernet import Fernet

cipher = Fernet(ENCRYPTION_KEY)

session_data = {
    "email": "alice@example.com",
    "sensitive_data_ref": "vault://secrets/user_123",  # Référence vault externe
    "pii_encrypted": cipher.encrypt(b"sensitive").decode()
}
```

---

## 10. Conclusion

### Points clés à retenir

- ✅ **RedisJSON + TTL = Solution optimale** pour sessions complexes et évolutives
- ✅ **Sliding window** avec `EXPIRE` pour inactivité automatique
- ✅ **Atomic updates** de sous-champs JSON évitent sérialisation complète
- ✅ **Circuit breaker + retry logic** pour résilience
- ✅ **Audit trail** via Redis Streams pour compliance
- ✅ **Scalabilité progressive** : Standalone → Replica → Cluster → Multi-région

### Quand NE PAS utiliser cette solution

- ❌ **Transactions multi-sessions** : Utiliser base relationnelle (ACID)
- ❌ **Données critiques non-réversibles** : Persistance durable requise (PostgreSQL)
- ❌ **Budget limité** : Redis Stack peut être coûteux, envisager Redis Core + Hashes
- ❌ **Forte cohérence requise** : Redis = cohérence éventuelle en multi-région

### Prochaines lectures

- [Cas #2 : Moteur de recherche e-commerce](./02-cas-moteur-recherche-ecommerce.md) → RediSearch et indexation
- [Patterns de développement : Client-Side Caching](../06-patterns-developpement-avances/04-client-side-caching.md) → Optimiser davantage
- [Redis Sentinel](../10-architecture-haute-disponibilite/03-redis-sentinel-monitoring-failover.md) → Haute disponibilité

---

**📚 Ressources complémentaires** :
- [RedisJSON Documentation](https://redis.io/docs/stack/json/)
- [Session Management Best Practices (OWASP)](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [Redis Streams for Audit Logs](https://redis.io/docs/data-types/streams/)

⏭️ [Cas #2 : Moteur de recherche e-commerce avec RediSearch](/16-etudes-cas-patterns-reels/02-cas-moteur-recherche-ecommerce.md)
