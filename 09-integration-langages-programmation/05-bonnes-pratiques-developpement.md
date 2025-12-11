🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5 Bonnes pratiques de développement

## Introduction

Écrire du code qui **fonctionne** est une chose, écrire du code **maintenable, testable et performant** en est une autre. Cette section présente les bonnes pratiques essentielles pour développer avec Redis de manière professionnelle.

Les principes couverts s'appliquent à tous les langages et garantissent :

- 📦 **Maintenabilité** : Code facile à comprendre et modifier
- 🧪 **Testabilité** : Code facile à tester
- 🚀 **Performance** : Code optimisé
- 🔒 **Sécurité** : Code sécurisé
- 📊 **Observabilité** : Code instrumenté et monitorable

---

## 1. Architecture et séparation des préoccupations

### Principe : Repository Pattern

Isoler la logique Redis dans une couche dédiée (repository/DAO) pour :
- Faciliter les tests (mock)
- Changer d'implémentation facilement
- Centraliser la logique métier

#### Python : Repository Pattern

```python
from abc import ABC, abstractmethod
from typing import Optional, List, Dict
import redis
import json
from datetime import timedelta

class UserRepository(ABC):
    """Interface abstraite pour le repository utilisateur"""

    @abstractmethod
    def get_by_id(self, user_id: int) -> Optional[Dict]:
        pass

    @abstractmethod
    def save(self, user: Dict, ttl: Optional[int] = None) -> bool:
        pass

    @abstractmethod
    def delete(self, user_id: int) -> bool:
        pass

    @abstractmethod
    def get_all(self, user_ids: List[int]) -> Dict[int, Dict]:
        pass

class RedisUserRepository(UserRepository):
    """Implémentation Redis du repository"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.key_prefix = "user"
        self.default_ttl = 3600

    def _make_key(self, user_id: int) -> str:
        """Génère une clé Redis structurée"""
        return f"{self.key_prefix}:{user_id}"

    def get_by_id(self, user_id: int) -> Optional[Dict]:
        """
        Récupère un utilisateur par son ID

        Args:
            user_id: ID de l'utilisateur

        Returns:
            Dict contenant les données utilisateur ou None
        """
        try:
            key = self._make_key(user_id)
            data = self.redis.get(key)

            if data is None:
                return None

            return json.loads(data)

        except redis.RedisError as e:
            # Logger l'erreur
            print(f"Error fetching user {user_id}: {e}")
            raise

    def save(self, user: Dict, ttl: Optional[int] = None) -> bool:
        """
        Sauvegarde un utilisateur

        Args:
            user: Dictionnaire contenant les données utilisateur
            ttl: Time-to-live en secondes (optionnel)

        Returns:
            True si succès, False sinon
        """
        try:
            user_id = user.get('id')
            if not user_id:
                raise ValueError("User must have an 'id' field")

            key = self._make_key(user_id)
            value = json.dumps(user)

            ttl = ttl or self.default_ttl
            self.redis.setex(key, ttl, value)

            return True

        except (redis.RedisError, ValueError) as e:
            print(f"Error saving user: {e}")
            return False

    def delete(self, user_id: int) -> bool:
        """Supprime un utilisateur"""
        try:
            key = self._make_key(user_id)
            result = self.redis.delete(key)
            return result > 0

        except redis.RedisError as e:
            print(f"Error deleting user {user_id}: {e}")
            return False

    def get_all(self, user_ids: List[int]) -> Dict[int, Dict]:
        """
        Récupère plusieurs utilisateurs en une fois (optimisé)

        Args:
            user_ids: Liste d'IDs utilisateur

        Returns:
            Dictionnaire {user_id: user_data}
        """
        if not user_ids:
            return {}

        try:
            # Utiliser pipeline pour réduire les round-trips
            pipe = self.redis.pipeline()
            keys = [self._make_key(uid) for uid in user_ids]

            for key in keys:
                pipe.get(key)

            results = pipe.execute()

            # Construire le dictionnaire de résultats
            users = {}
            for user_id, data in zip(user_ids, results):
                if data:
                    users[user_id] = json.loads(data)

            return users

        except redis.RedisError as e:
            print(f"Error fetching multiple users: {e}")
            raise

# Service layer utilisant le repository
class UserService:
    """Service métier utilisant le repository"""

    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo

    def get_user_profile(self, user_id: int) -> Optional[Dict]:
        """
        Récupère le profil utilisateur avec logique métier
        """
        user = self.user_repo.get_by_id(user_id)

        if not user:
            return None

        # Enrichir avec logique métier
        user['display_name'] = f"{user.get('first_name', '')} {user.get('last_name', '')}".strip()

        return user

    def update_user(self, user_id: int, updates: Dict) -> bool:
        """
        Met à jour un utilisateur avec validation
        """
        # Récupérer l'utilisateur existant
        user = self.user_repo.get_by_id(user_id)

        if not user:
            raise ValueError(f"User {user_id} not found")

        # Appliquer les mises à jour
        user.update(updates)

        # Validation métier
        if 'email' in user and '@' not in user['email']:
            raise ValueError("Invalid email format")

        # Sauvegarder
        return self.user_repo.save(user)

# Utilisation
if __name__ == "__main__":
    # Configuration
    redis_client = redis.Redis(host='localhost', decode_responses=True)

    # Instancier le repository
    user_repo = RedisUserRepository(redis_client)

    # Instancier le service
    user_service = UserService(user_repo)

    # Utiliser le service
    user = {
        'id': 123,
        'first_name': 'Alice',
        'last_name': 'Smith',
        'email': 'alice@example.com'
    }

    user_service.update_user(123, user)
    profile = user_service.get_user_profile(123)

    print(f"Profile: {profile}")
```

#### Node.js : Repository Pattern

```javascript
import Redis from 'ioredis';

/**
 * Interface du repository (TypeScript)
 */
interface IUserRepository {
    getById(userId: number): Promise<User | null>;
    save(user: User, ttl?: number): Promise<boolean>;
    delete(userId: number): Promise<boolean>;
    getAll(userIds: number[]): Promise<Map<number, User>>;
}

/**
 * Modèle User
 */
interface User {
    id: number;
    firstName: string;
    lastName: string;
    email: string;
}

/**
 * Repository Redis pour les utilisateurs
 */
class RedisUserRepository implements IUserRepository {
    private redis: Redis;
    private keyPrefix = 'user';
    private defaultTtl = 3600;

    constructor(redisClient: Redis) {
        this.redis = redisClient;
    }

    /**
     * Génère une clé Redis structurée
     */
    private makeKey(userId: number): string {
        return `${this.keyPrefix}:${userId}`;
    }

    /**
     * Récupère un utilisateur par ID
     */
    async getById(userId: number): Promise<User | null> {
        try {
            const key = this.makeKey(userId);
            const data = await this.redis.get(key);

            if (!data) {
                return null;
            }

            return JSON.parse(data);

        } catch (err) {
            console.error(`Error fetching user ${userId}:`, err);
            throw err;
        }
    }

    /**
     * Sauvegarde un utilisateur
     */
    async save(user: User, ttl?: number): Promise<boolean> {
        try {
            if (!user.id) {
                throw new Error("User must have an 'id' field");
            }

            const key = this.makeKey(user.id);
            const value = JSON.stringify(user);
            const expiration = ttl || this.defaultTtl;

            await this.redis.set(key, value, 'EX', expiration);

            return true;

        } catch (err) {
            console.error('Error saving user:', err);
            return false;
        }
    }

    /**
     * Supprime un utilisateur
     */
    async delete(userId: number): Promise<boolean> {
        try {
            const key = this.makeKey(userId);
            const result = await this.redis.del(key);
            return result > 0;

        } catch (err) {
            console.error(`Error deleting user ${userId}:`, err);
            return false;
        }
    }

    /**
     * Récupère plusieurs utilisateurs (optimisé avec pipeline)
     */
    async getAll(userIds: number[]): Promise<Map<number, User>> {
        if (userIds.length === 0) {
            return new Map();
        }

        try {
            // Pipeline pour optimiser
            const pipeline = this.redis.pipeline();
            const keys = userIds.map(uid => this.makeKey(uid));

            keys.forEach(key => pipeline.get(key));

            const results = await pipeline.exec();

            // Construire la Map de résultats
            const users = new Map<number, User>();

            results?.forEach(([err, data], index) => {
                if (!err && data) {
                    const user = JSON.parse(data as string);
                    users.set(userIds[index], user);
                }
            });

            return users;

        } catch (err) {
            console.error('Error fetching multiple users:', err);
            throw err;
        }
    }
}

/**
 * Service métier
 */
class UserService {
    private userRepo: IUserRepository;

    constructor(userRepository: IUserRepository) {
        this.userRepo = userRepository;
    }

    /**
     * Récupère un profil utilisateur enrichi
     */
    async getUserProfile(userId: number): Promise<User | null> {
        const user = await this.userRepo.getById(userId);

        if (!user) {
            return null;
        }

        // Enrichir avec logique métier
        return {
            ...user,
            displayName: `${user.firstName} ${user.lastName}`.trim(),
        };
    }

    /**
     * Met à jour un utilisateur avec validation
     */
    async updateUser(userId: number, updates: Partial<User>): Promise<boolean> {
        // Récupérer l'utilisateur existant
        const user = await this.userRepo.getById(userId);

        if (!user) {
            throw new Error(`User ${userId} not found`);
        }

        // Appliquer les mises à jour
        const updatedUser = { ...user, ...updates };

        // Validation métier
        if (updatedUser.email && !updatedUser.email.includes('@')) {
            throw new Error('Invalid email format');
        }

        // Sauvegarder
        return await this.userRepo.save(updatedUser);
    }
}

// Utilisation
const redis = new Redis();
const userRepo = new RedisUserRepository(redis);
const userService = new UserService(userRepo);

// Exemple
(async () => {
    const user: User = {
        id: 123,
        firstName: 'Alice',
        lastName: 'Smith',
        email: 'alice@example.com',
    };

    await userService.updateUser(123, user);
    const profile = await userService.getUserProfile(123);

    console.log('Profile:', profile);
})();
```

---

## 2. Configuration et environnement

### Principe : Configuration externalisée

Ne jamais hardcoder les paramètres de connexion. Utiliser des variables d'environnement et fichiers de configuration.

#### Python : Configuration avec pydantic

```python
from pydantic import BaseSettings, Field, validator
from typing import Optional
import redis

class RedisConfig(BaseSettings):
    """Configuration Redis avec validation"""

    host: str = Field(default='localhost', env='REDIS_HOST')
    port: int = Field(default=6379, env='REDIS_PORT')
    password: Optional[str] = Field(default=None, env='REDIS_PASSWORD')
    db: int = Field(default=0, env='REDIS_DB')

    # Pool settings
    max_connections: int = Field(default=50, env='REDIS_MAX_CONNECTIONS')
    socket_timeout: int = Field(default=5, env='REDIS_SOCKET_TIMEOUT')
    socket_connect_timeout: int = Field(default=5, env='REDIS_CONNECT_TIMEOUT')

    # Application settings
    default_ttl: int = Field(default=3600, env='REDIS_DEFAULT_TTL')
    key_prefix: str = Field(default='app', env='REDIS_KEY_PREFIX')

    @validator('port')
    def validate_port(cls, v):
        if not 1 <= v <= 65535:
            raise ValueError('Port must be between 1 and 65535')
        return v

    @validator('max_connections')
    def validate_max_connections(cls, v):
        if v < 1:
            raise ValueError('max_connections must be at least 1')
        return v

    class Config:
        env_file = '.env'
        env_file_encoding = 'utf-8'

    def create_client(self) -> redis.Redis:
        """Factory method pour créer un client Redis"""
        pool = redis.ConnectionPool(
            host=self.host,
            port=self.port,
            password=self.password,
            db=self.db,
            max_connections=self.max_connections,
            socket_timeout=self.socket_timeout,
            socket_connect_timeout=self.socket_connect_timeout,
            decode_responses=True,
        )

        return redis.Redis(connection_pool=pool)

# Utilisation
config = RedisConfig()  # Charge depuis .env automatiquement
redis_client = config.create_client()

print(f"✅ Connected to Redis at {config.host}:{config.port}")
```

#### Fichier .env

```bash
# .env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=mysecretpassword
REDIS_DB=0
REDIS_MAX_CONNECTIONS=100
REDIS_SOCKET_TIMEOUT=5
REDIS_CONNECT_TIMEOUT=10
REDIS_DEFAULT_TTL=3600
REDIS_KEY_PREFIX=myapp
```

#### Node.js : Configuration avec dotenv

```javascript
import dotenv from 'dotenv';
import Redis from 'ioredis';
import { z } from 'zod';

// Charger .env
dotenv.config();

/**
 * Schéma de validation avec Zod
 */
const RedisConfigSchema = z.object({
    host: z.string().default('localhost'),
    port: z.number().min(1).max(65535).default(6379),
    password: z.string().optional(),
    db: z.number().min(0).max(15).default(0),
    maxConnections: z.number().min(1).default(50),
    connectTimeout: z.number().min(1000).default(5000),
    commandTimeout: z.number().min(1000).default(5000),
    defaultTtl: z.number().min(1).default(3600),
    keyPrefix: z.string().default('app'),
});

type RedisConfig = z.infer<typeof RedisConfigSchema>;

/**
 * Charge et valide la configuration
 */
function loadRedisConfig(): RedisConfig {
    const rawConfig = {
        host: process.env.REDIS_HOST,
        port: parseInt(process.env.REDIS_PORT || '6379'),
        password: process.env.REDIS_PASSWORD,
        db: parseInt(process.env.REDIS_DB || '0'),
        maxConnections: parseInt(process.env.REDIS_MAX_CONNECTIONS || '50'),
        connectTimeout: parseInt(process.env.REDIS_CONNECT_TIMEOUT || '5000'),
        commandTimeout: parseInt(process.env.REDIS_COMMAND_TIMEOUT || '5000'),
        defaultTtl: parseInt(process.env.REDIS_DEFAULT_TTL || '3600'),
        keyPrefix: process.env.REDIS_KEY_PREFIX,
    };

    // Valider avec Zod
    const config = RedisConfigSchema.parse(rawConfig);

    return config;
}

/**
 * Factory pour créer un client Redis
 */
function createRedisClient(config: RedisConfig): Redis {
    return new Redis({
        host: config.host,
        port: config.port,
        password: config.password,
        db: config.db,
        maxRetriesPerRequest: 3,
        connectTimeout: config.connectTimeout,
        commandTimeout: config.commandTimeout,
        retryStrategy(times) {
            const delay = Math.min(times * 50, 2000);
            return delay;
        },
    });
}

// Utilisation
try {
    const config = loadRedisConfig();
    const redis = createRedisClient(config);

    console.log(`✅ Connected to Redis at ${config.host}:${config.port}`);

} catch (err) {
    console.error('❌ Configuration error:', err);
    process.exit(1);
}
```

---

## 3. Logging et observabilité

### Principe : Logger avec contexte

Toujours logger avec suffisamment de contexte pour débugger en production.

#### Python : Structured logging

```python
import logging
import json
from datetime import datetime
from typing import Any, Dict
import redis

class StructuredLogger:
    """Logger structuré pour Redis operations"""

    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
        self.logger.setLevel(logging.INFO)

        # Handler pour JSON
        handler = logging.StreamHandler()
        handler.setFormatter(self._json_formatter())
        self.logger.addHandler(handler)

    def _json_formatter(self):
        """Formatter JSON pour logs structurés"""
        class JsonFormatter(logging.Formatter):
            def format(self, record):
                log_data = {
                    'timestamp': datetime.utcnow().isoformat(),
                    'level': record.levelname,
                    'message': record.getMessage(),
                    'logger': record.name,
                }

                # Ajouter extra fields si présents
                if hasattr(record, 'extra'):
                    log_data.update(record.extra)

                return json.dumps(log_data)

        return JsonFormatter()

    def log_redis_operation(
        self,
        operation: str,
        key: str,
        success: bool,
        duration_ms: float,
        error: str = None
    ):
        """Log une opération Redis avec contexte"""
        extra = {
            'operation': operation,
            'key': key,
            'success': success,
            'duration_ms': duration_ms,
        }

        if error:
            extra['error'] = error

        if success:
            self.logger.info(
                f"Redis {operation} succeeded",
                extra={'extra': extra}
            )
        else:
            self.logger.error(
                f"Redis {operation} failed",
                extra={'extra': extra}
            )

class InstrumentedRedisClient:
    """Client Redis avec logging automatique"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.logger = StructuredLogger('redis.client')

    def get(self, key: str) -> Any:
        """GET avec logging"""
        import time

        start = time.time()
        error = None
        success = True
        result = None

        try:
            result = self.redis.get(key)
        except Exception as e:
            success = False
            error = str(e)
            raise
        finally:
            duration_ms = (time.time() - start) * 1000
            self.logger.log_redis_operation(
                'GET',
                key,
                success,
                duration_ms,
                error
            )

        return result

    def set(self, key: str, value: Any, ex: int = None) -> bool:
        """SET avec logging"""
        import time

        start = time.time()
        error = None
        success = True

        try:
            if ex:
                self.redis.setex(key, ex, value)
            else:
                self.redis.set(key, value)
        except Exception as e:
            success = False
            error = str(e)
            raise
        finally:
            duration_ms = (time.time() - start) * 1000
            self.logger.log_redis_operation(
                'SET',
                key,
                success,
                duration_ms,
                error
            )

        return success

# Utilisation
redis_client = redis.Redis(decode_responses=True)
instrumented = InstrumentedRedisClient(redis_client)

instrumented.set('user:123', 'Alice')
value = instrumented.get('user:123')

# Output (JSON structuré):
# {"timestamp": "2024-01-15T10:30:45.123", "level": "INFO",
#  "message": "Redis SET succeeded", "operation": "SET",
#  "key": "user:123", "success": true, "duration_ms": 1.23}
```

---

## 4. Gestion des clés et namespaces

### Principe : Convention de nommage stricte

Utiliser des conventions de nommage cohérentes et documentées.

```python
class KeyBuilder:
    """Builder pour générer des clés Redis structurées"""

    def __init__(self, app_name: str, environment: str = 'production'):
        self.app_name = app_name
        self.environment = environment

    def user_key(self, user_id: int) -> str:
        """user:{app}:{env}:{id}"""
        return f"user:{self.app_name}:{self.environment}:{user_id}"

    def session_key(self, session_id: str) -> str:
        """session:{app}:{env}:{id}"""
        return f"session:{self.app_name}:{self.environment}:{session_id}"

    def cache_key(self, resource: str, resource_id: str) -> str:
        """cache:{app}:{env}:{resource}:{id}"""
        return f"cache:{self.app_name}:{self.environment}:{resource}:{resource_id}"

    def counter_key(self, counter_name: str) -> str:
        """counter:{app}:{env}:{name}"""
        return f"counter:{self.app_name}:{self.environment}:{counter_name}"

    def lock_key(self, resource: str) -> str:
        """lock:{app}:{env}:{resource}"""
        return f"lock:{self.app_name}:{self.environment}:{resource}"

# Utilisation
kb = KeyBuilder('myapp', 'production')

user_key = kb.user_key(123)           # user:myapp:production:123
session_key = kb.session_key('abc')   # session:myapp:production:abc
cache_key = kb.cache_key('post', '5') # cache:myapp:production:post:5

print(f"User key: {user_key}")
```

---

## 5. Gestion des erreurs avec contexte

### Pattern : Custom exceptions

```python
class RedisException(Exception):
    """Exception de base pour Redis"""
    pass

class RedisConnectionException(RedisException):
    """Erreur de connexion Redis"""
    def __init__(self, host: str, port: int, original_error: Exception):
        self.host = host
        self.port = port
        self.original_error = original_error
        super().__init__(
            f"Failed to connect to Redis at {host}:{port}: {original_error}"
        )

class RedisTimeoutException(RedisException):
    """Timeout Redis"""
    def __init__(self, operation: str, key: str, timeout: float):
        self.operation = operation
        self.key = key
        self.timeout = timeout
        super().__init__(
            f"Redis {operation} timeout after {timeout}s for key '{key}'"
        )

class RedisKeyNotFoundException(RedisException):
    """Clé Redis non trouvée"""
    def __init__(self, key: str):
        self.key = key
        super().__init__(f"Redis key not found: '{key}'")

# Utilisation avec gestion d'erreurs claire
def get_user_or_fail(redis_client, user_id: int) -> Dict:
    """Récupère un utilisateur ou lève une exception explicite"""
    key = f"user:{user_id}"

    try:
        data = redis_client.get(key)
    except redis.ConnectionError as e:
        raise RedisConnectionException('localhost', 6379, e)
    except redis.TimeoutError as e:
        raise RedisTimeoutException('GET', key, 5.0)

    if data is None:
        raise RedisKeyNotFoundException(key)

    return json.loads(data)

# Utilisation
try:
    user = get_user_or_fail(redis_client, 123)
except RedisKeyNotFoundException as e:
    print(f"User not found: {e.key}")
except RedisConnectionException as e:
    print(f"Connection failed: {e.host}:{e.port}")
except RedisTimeoutException as e:
    print(f"Timeout: {e.operation} on {e.key}")
```

---

## 6. Performance : Batch operations

### Principe : Minimiser les round-trips

```python
from typing import List, Dict
import redis

class OptimizedRedisOperations:
    """Opérations Redis optimisées"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def get_multiple_optimized(self, keys: List[str]) -> Dict[str, str]:
        """
        ✅ BON : MGET pour récupérer plusieurs clés en 1 RTT
        """
        if not keys:
            return {}

        values = self.redis.mget(keys)

        return {
            key: value
            for key, value in zip(keys, values)
            if value is not None
        }

    def set_multiple_optimized(self, data: Dict[str, str]) -> bool:
        """
        ✅ BON : Pipeline pour SET multiples en 1 RTT
        """
        if not data:
            return True

        pipe = self.redis.pipeline()

        for key, value in data.items():
            pipe.set(key, value)

        results = pipe.execute()

        return all(results)

    def increment_multiple(self, counters: List[str]) -> Dict[str, int]:
        """
        ✅ BON : Pipeline pour INCR multiples
        """
        if not counters:
            return {}

        pipe = self.redis.pipeline()

        for counter in counters:
            pipe.incr(counter)

        results = pipe.execute()

        return {
            counter: value
            for counter, value in zip(counters, results)
        }

# Benchmark
import time

def benchmark_batch_operations():
    """Compare opérations individuelles vs batch"""
    redis_client = redis.Redis(decode_responses=True)
    ops = OptimizedRedisOperations(redis_client)

    keys = [f'key:{i}' for i in range(100)]

    # ❌ LENT : Opérations individuelles
    start = time.time()
    for key in keys:
        redis_client.get(key)
    individual_time = time.time() - start
    print(f"Individual GETs: {individual_time:.3f}s")

    # ✅ RAPIDE : Batch avec MGET
    start = time.time()
    ops.get_multiple_optimized(keys)
    batch_time = time.time() - start
    print(f"Batch MGET: {batch_time:.3f}s")

    speedup = individual_time / batch_time
    print(f"🚀 Speedup: {speedup:.1f}x")

# Output typique:
# Individual GETs: 0.156s
# Batch MGET: 0.003s
# 🚀 Speedup: 52.0x

benchmark_batch_operations()
```

---

## 7. Patterns de cache avancés

### Cache avec invalidation

```python
from typing import Optional, Callable, List
import redis
import hashlib
import json

class SmartCache:
    """Cache intelligent avec invalidation et dépendances"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.default_ttl = 3600

    def cache_with_dependencies(
        self,
        key: str,
        fetch_fn: Callable,
        dependencies: List[str],
        ttl: int = None
    ):
        """
        Cache avec tracking des dépendances

        Si une dépendance change, le cache est invalidé
        """
        ttl = ttl or self.default_ttl

        # Vérifier le cache
        cached = self.redis.get(key)
        if cached:
            # Vérifier que les dépendances n'ont pas changé
            if self._dependencies_valid(key, dependencies):
                return json.loads(cached)

        # Cache miss ou dépendances invalides
        data = fetch_fn()

        # Stocker la donnée
        self.redis.setex(key, ttl, json.dumps(data))

        # Stocker les hash des dépendances
        self._store_dependencies(key, dependencies)

        return data

    def _dependencies_valid(self, key: str, dependencies: List[str]) -> bool:
        """Vérifie si les dépendances sont toujours valides"""
        deps_key = f"{key}:deps"
        stored_deps = self.redis.hgetall(deps_key)

        if not stored_deps:
            return False

        # Vérifier chaque dépendance
        for dep in dependencies:
            dep_value = self.redis.get(dep)
            if not dep_value:
                return False

            # Comparer le hash
            current_hash = hashlib.md5(dep_value.encode()).hexdigest()
            stored_hash = stored_deps.get(dep)

            if current_hash != stored_hash:
                return False

        return True

    def _store_dependencies(self, key: str, dependencies: List[str]):
        """Stocke les hash des dépendances"""
        deps_key = f"{key}:deps"

        deps_hash = {}
        for dep in dependencies:
            dep_value = self.redis.get(dep)
            if dep_value:
                deps_hash[dep] = hashlib.md5(dep_value.encode()).hexdigest()

        if deps_hash:
            self.redis.hset(deps_key, mapping=deps_hash)
            self.redis.expire(deps_key, self.default_ttl)

    def invalidate(self, pattern: str):
        """Invalide tous les caches matchant le pattern"""
        cursor = 0
        deleted = 0

        while True:
            cursor, keys = self.redis.scan(
                cursor,
                match=pattern,
                count=100
            )

            if keys:
                deleted += self.redis.delete(*keys)

            if cursor == 0:
                break

        return deleted

# Utilisation
cache = SmartCache(redis.Redis(decode_responses=True))

def fetch_user_profile(user_id):
    """Fonction coûteuse à cacher"""
    print(f"Fetching user {user_id} from database...")
    return {'id': user_id, 'name': 'Alice', 'score': 100}

# Cache avec dépendances
user_id = 123
profile = cache.cache_with_dependencies(
    key=f'profile:{user_id}',
    fetch_fn=lambda: fetch_user_profile(user_id),
    dependencies=[f'user:{user_id}', 'settings:global'],
    ttl=3600
)

# Si user:123 ou settings:global changent, le cache sera invalidé
```

---

## 8. Testing avec mocks

### Python : Mocking Redis

```python
import unittest
from unittest.mock import Mock, patch, MagicMock
import redis

class TestUserRepository(unittest.TestCase):
    """Tests unitaires du repository avec mock Redis"""

    def setUp(self):
        """Setup exécuté avant chaque test"""
        self.mock_redis = Mock(spec=redis.Redis)
        self.repo = RedisUserRepository(self.mock_redis)

    def test_get_by_id_success(self):
        """Test GET réussi"""
        # Arrange
        user_id = 123
        expected_user = {'id': 123, 'name': 'Alice'}
        self.mock_redis.get.return_value = json.dumps(expected_user)

        # Act
        result = self.repo.get_by_id(user_id)

        # Assert
        self.assertEqual(result, expected_user)
        self.mock_redis.get.assert_called_once_with('user:123')

    def test_get_by_id_not_found(self):
        """Test GET clé inexistante"""
        # Arrange
        self.mock_redis.get.return_value = None

        # Act
        result = self.repo.get_by_id(999)

        # Assert
        self.assertIsNone(result)

    def test_get_by_id_redis_error(self):
        """Test GET avec erreur Redis"""
        # Arrange
        self.mock_redis.get.side_effect = redis.ConnectionError("Connection failed")

        # Act & Assert
        with self.assertRaises(redis.ConnectionError):
            self.repo.get_by_id(123)

    def test_save_success(self):
        """Test SAVE réussi"""
        # Arrange
        user = {'id': 123, 'name': 'Alice'}

        # Act
        result = self.repo.save(user, ttl=3600)

        # Assert
        self.assertTrue(result)
        self.mock_redis.setex.assert_called_once()
        args = self.mock_redis.setex.call_args[0]
        self.assertEqual(args[0], 'user:123')
        self.assertEqual(args[1], 3600)

    def test_get_all_multiple_users(self):
        """Test GET multiple avec pipeline"""
        # Arrange
        user_ids = [1, 2, 3]
        mock_pipeline = MagicMock()
        self.mock_redis.pipeline.return_value = mock_pipeline
        mock_pipeline.execute.return_value = [
            json.dumps({'id': 1, 'name': 'Alice'}),
            json.dumps({'id': 2, 'name': 'Bob'}),
            None  # User 3 non trouvé
        ]

        # Act
        result = self.repo.get_all(user_ids)

        # Assert
        self.assertEqual(len(result), 2)
        self.assertIn(1, result)
        self.assertIn(2, result)
        self.assertNotIn(3, result)

# Exécuter les tests
if __name__ == '__main__':
    unittest.main()
```

### Node.js : Testing avec Jest et ioredis-mock

```javascript
import { jest } from '@jest/globals';
import RedisMock from 'ioredis-mock';

describe('RedisUserRepository', () => {
    let redis;
    let repository;

    beforeEach(() => {
        // Utiliser ioredis-mock au lieu du vrai Redis
        redis = new RedisMock();
        repository = new RedisUserRepository(redis);
    });

    afterEach(async () => {
        await redis.flushall();
        await redis.disconnect();
    });

    describe('getById', () => {
        it('should return user when found', async () => {
            // Arrange
            const user = {
                id: 123,
                firstName: 'Alice',
                lastName: 'Smith',
                email: 'alice@example.com',
            };
            await redis.set('user:123', JSON.stringify(user));

            // Act
            const result = await repository.getById(123);

            // Assert
            expect(result).toEqual(user);
        });

        it('should return null when user not found', async () => {
            // Act
            const result = await repository.getById(999);

            // Assert
            expect(result).toBeNull();
        });

        it('should throw error on Redis failure', async () => {
            // Arrange
            const mockGet = jest.spyOn(redis, 'get')
                .mockRejectedValue(new Error('Redis connection failed'));

            // Act & Assert
            await expect(repository.getById(123))
                .rejects
                .toThrow('Redis connection failed');

            mockGet.mockRestore();
        });
    });

    describe('save', () => {
        it('should save user with TTL', async () => {
            // Arrange
            const user = {
                id: 123,
                firstName: 'Alice',
                lastName: 'Smith',
                email: 'alice@example.com',
            };

            // Act
            const result = await repository.save(user, 3600);

            // Assert
            expect(result).toBe(true);

            const saved = await redis.get('user:123');
            expect(JSON.parse(saved)).toEqual(user);

            const ttl = await redis.ttl('user:123');
            expect(ttl).toBeGreaterThan(0);
        });

        it('should return false on error', async () => {
            // Arrange
            const user = { id: 123, firstName: 'Alice' };
            const mockSet = jest.spyOn(redis, 'set')
                .mockRejectedValue(new Error('Redis error'));

            // Act
            const result = await repository.save(user);

            // Assert
            expect(result).toBe(false);

            mockSet.mockRestore();
        });
    });

    describe('getAll', () => {
        it('should return multiple users with pipeline', async () => {
            // Arrange
            await redis.set('user:1', JSON.stringify({ id: 1, firstName: 'Alice' }));
            await redis.set('user:2', JSON.stringify({ id: 2, firstName: 'Bob' }));

            // Act
            const result = await repository.getAll([1, 2, 3]);

            // Assert
            expect(result.size).toBe(2);
            expect(result.get(1)).toEqual({ id: 1, firstName: 'Alice' });
            expect(result.get(2)).toEqual({ id: 2, firstName: 'Bob' });
            expect(result.has(3)).toBe(false);
        });
    });
});
```

---

## 9. Documentation du code

### Principe : Documenter les décisions et contraintes

```python
from typing import Optional, Dict, List
import redis

class CachedUserService:
    """
    Service de gestion des utilisateurs avec cache Redis.

    Ce service implémente le pattern Cache-Aside avec les garanties suivantes :
    - TTL par défaut de 1h pour éviter les données stales
    - Invalidation explicite lors des modifications
    - Fallback sur la base de données si Redis indisponible

    Architecture:
        Application → CachedUserService → [Redis Cache]
                                       ↓
                                   [Database]

    Exemple:
        >>> service = CachedUserService(redis_client, db_client)
        >>> user = service.get_user(123)  # Cache miss → fetch from DB
        >>> user = service.get_user(123)  # Cache hit → from Redis
        >>> service.update_user(123, {'name': 'Bob'})  # Invalidate cache

    Notes:
        - Ce service n'est PAS thread-safe pour les opérations concurrentes
        - Les données en cache peuvent être légèrement obsolètes (max 1h)
        - Redis indisponible dégrade les performances mais n'empêche pas le service
    """

    def __init__(self, redis_client: redis.Redis, db_client):
        """
        Initialise le service avec cache.

        Args:
            redis_client: Client Redis configuré
            db_client: Client de base de données

        Raises:
            ValueError: Si les clients sont None
        """
        if redis_client is None or db_client is None:
            raise ValueError("Clients cannot be None")

        self.redis = redis_client
        self.db = db_client
        self.cache_ttl = 3600  # 1 heure

    def get_user(self, user_id: int) -> Optional[Dict]:
        """
        Récupère un utilisateur avec cache Redis.

        Le comportement est le suivant:
        1. Tentative de lecture depuis Redis
        2. Si cache miss ou erreur Redis → lecture depuis DB
        3. Si trouvé en DB → mise en cache pour la prochaine fois

        Args:
            user_id: ID unique de l'utilisateur

        Returns:
            Dictionnaire contenant les données utilisateur, ou None si non trouvé

        Raises:
            DatabaseError: Si erreur critique de la base de données

        Exemples:
            >>> user = service.get_user(123)
            >>> print(user['name'])
            'Alice'

        Performance:
            - Cache hit: ~1ms
            - Cache miss: ~10-50ms (dépend de la DB)

        Notes:
            - Les erreurs Redis ne font PAS échouer l'opération
            - La DB est la source de vérité (authoritative source)
        """
        cache_key = f"user:{user_id}"

        # Tentative cache
        try:
            cached = self.redis.get(cache_key)
            if cached:
                return json.loads(cached)
        except redis.RedisError as e:
            # Logger mais continuer sans cache
            logger.warning(f"Redis error (degraded mode): {e}")

        # Cache miss : fetch from DB
        user = self.db.get_user(user_id)

        if user:
            # Mise en cache pour la prochaine fois
            try:
                self.redis.setex(
                    cache_key,
                    self.cache_ttl,
                    json.dumps(user)
                )
            except redis.RedisError:
                pass  # Silent fail sur erreur cache

        return user
```

---

## 10. Checklist des bonnes pratiques

### Avant de déployer en production :

#### Architecture
- ✅ Repository pattern implémenté pour isolation
- ✅ Séparation claire entre logique métier et accès Redis
- ✅ Pas de dépendance directe à Redis dans la logique métier

#### Configuration
- ✅ Configuration externalisée (variables d'environnement)
- ✅ Validation de la configuration au démarrage
- ✅ Pas de credentials hardcodés

#### Logging
- ✅ Logs structurés (JSON) avec contexte complet
- ✅ Logs des erreurs avec stack trace
- ✅ Logs des métriques de performance

#### Gestion des clés
- ✅ Convention de nommage documentée et respectée
- ✅ Préfixes d'environnement (dev/staging/prod)
- ✅ TTL défini pour toutes les clés

#### Gestion des erreurs
- ✅ Exceptions custom pour clarté
- ✅ Retry logic avec exponential backoff
- ✅ Fallback en cas d'indisponibilité Redis

#### Performance
- ✅ Pipeline pour opérations groupées
- ✅ MGET/MSET utilisés quand approprié
- ✅ Pas de KEYS en production (utiliser SCAN)

#### Tests
- ✅ Tests unitaires avec mocks
- ✅ Tests d'intégration avec Redis réel
- ✅ Couverture > 80%

#### Documentation
- ✅ Code documenté (docstrings, JSDoc, etc.)
- ✅ README avec instructions de configuration
- ✅ Décisions architecturales documentées

#### Sécurité
- ✅ Authentification activée
- ✅ TLS/SSL en production
- ✅ Pas de données sensibles en clair

#### Monitoring
- ✅ Métriques exportées (Prometheus/Datadog)
- ✅ Alertes configurées
- ✅ Dashboard de monitoring

---

## Points clés à retenir

🔑 **Repository Pattern** : Isoler Redis dans une couche dédiée

🔑 **Configuration externalisée** : Variables d'environnement avec validation

🔑 **Logging structuré** : JSON avec contexte complet pour débug

🔑 **Convention de nommage** : Clés structurées et préfixées

🔑 **Gestion d'erreurs** : Exceptions custom et fallbacks

🔑 **Batch operations** : Pipeline/MGET pour performance

🔑 **Tests avec mocks** : Isoler Redis dans les tests unitaires

🔑 **Documentation** : Code et décisions bien documentés

🔑 **Monitoring** : Métriques et alertes en production

---

## Ressources complémentaires

- **Clean Code** : Robert C. Martin
- **Repository Pattern** : Martin Fowler
- **12-Factor App** : https://12factor.net/
- **Structured Logging** : https://www.structlog.org/

---

## Prochaine section

➡️ **Section 9.6** : Testing et mocking Redis - Stratégies de tests avancées

**Niveau** : Intermédiaire
**Durée estimée** : 60 minutes
**Prérequis** : Sections 9.1-9.5

⏭️ [Testing et mocking Redis](/09-integration-langages-programmation/06-testing-mocking-redis.md)
