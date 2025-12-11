🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.6 Testing et mocking Redis

## Introduction

Tester du code qui dépend de Redis est **essentiel** mais présente des défis spécifiques :

- 🧪 **Dépendance externe** : Redis est un service tiers
- ⚡ **Performance** : Les tests doivent rester rapides
- 🔄 **État partagé** : Redis peut avoir des effets de bord entre tests
- 🎭 **Scénarios complexes** : Timeouts, pannes, cluster resharding

Cette section couvre les stratégies pour tester efficacement votre code Redis, du mocking simple aux tests d'intégration avancés.

---

## Types de tests

```
┌─────────────────────────────────────────────────────────────┐
│                    Pyramide des tests                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                       ▲                                     │
│                      ╱ ╲                                    │
│                     ╱   ╲  E2E Tests                        │
│                    ╱     ╲  (Redis réel + App complète)     │
│                   ╱───────╲                                 │
│                  ╱         ╲                                │
│                 ╱           ╲ Tests d'intégration           │
│                ╱             ╲ (Redis réel ou Docker)       │
│               ╱───────────────╲                             │
│              ╱                 ╲                            │
│             ╱                   ╲ Tests unitaires           │
│            ╱                     ╲ (Mock Redis)             │
│           ╱───────────────────────╲                         │
│                                                             │
│  Quantité:    Beaucoup → → → Peu                            │
│  Vitesse:     Rapide   → → → Lent                           │
│  Isolation:   Haute    → → → Basse                          │
└─────────────────────────────────────────────────────────────┘
```

### 1. Tests unitaires avec mocks

**Objectif** : Tester la logique métier sans dépendance Redis
- ✅ Très rapides (<1ms par test)
- ✅ Pas de setup infrastructure
- ⚠️ Ne valident pas l'intégration réelle

### 2. Tests d'intégration

**Objectif** : Tester avec un Redis réel (local ou conteneur)
- ✅ Valident le comportement réel
- ✅ Détectent les bugs d'intégration
- ⚠️ Plus lents (~10-100ms par test)
- ⚠️ Nécessitent Redis disponible

### 3. Tests end-to-end (E2E)

**Objectif** : Tester l'application complète
- ✅ Valident le workflow complet
- ⚠️ Très lents (secondes)
- ⚠️ Fragiles

---

## Tests unitaires avec mocks

### Python : unittest.mock

```python
import unittest
from unittest.mock import Mock, MagicMock, patch, call
import redis
import json
from typing import Dict, Optional

# Code à tester (voir section 9.5)
class UserRepository:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.key_prefix = "user"

    def get_by_id(self, user_id: int) -> Optional[Dict]:
        key = f"{self.key_prefix}:{user_id}"
        data = self.redis.get(key)

        if data is None:
            return None

        return json.loads(data)

    def save(self, user: Dict, ttl: int = 3600) -> bool:
        user_id = user.get('id')
        if not user_id:
            raise ValueError("User must have an 'id' field")

        key = f"{self.key_prefix}:{user_id}"
        value = json.dumps(user)

        self.redis.setex(key, ttl, value)
        return True

    def delete(self, user_id: int) -> bool:
        key = f"{self.key_prefix}:{user_id}"
        result = self.redis.delete(key)
        return result > 0

    def get_all(self, user_ids: list) -> Dict[int, Dict]:
        if not user_ids:
            return {}

        pipe = self.redis.pipeline()
        keys = [f"{self.key_prefix}:{uid}" for uid in user_ids]

        for key in keys:
            pipe.get(key)

        results = pipe.execute()

        users = {}
        for user_id, data in zip(user_ids, results):
            if data:
                users[user_id] = json.loads(data)

        return users

# Tests unitaires
class TestUserRepository(unittest.TestCase):
    """Suite de tests unitaires pour UserRepository"""

    def setUp(self):
        """Exécuté avant chaque test"""
        # Créer un mock Redis
        self.mock_redis = Mock(spec=redis.Redis)
        self.repo = UserRepository(self.mock_redis)

    def tearDown(self):
        """Exécuté après chaque test"""
        pass

    # Tests GET
    def test_get_by_id_success(self):
        """Test : Récupération réussie d'un utilisateur"""
        # Arrange
        user_id = 123
        expected_user = {'id': 123, 'name': 'Alice', 'email': 'alice@example.com'}
        self.mock_redis.get.return_value = json.dumps(expected_user)

        # Act
        result = self.repo.get_by_id(user_id)

        # Assert
        self.assertEqual(result, expected_user)
        self.mock_redis.get.assert_called_once_with('user:123')

    def test_get_by_id_not_found(self):
        """Test : Utilisateur non trouvé"""
        # Arrange
        self.mock_redis.get.return_value = None

        # Act
        result = self.repo.get_by_id(999)

        # Assert
        self.assertIsNone(result)
        self.mock_redis.get.assert_called_once_with('user:999')

    def test_get_by_id_redis_error(self):
        """Test : Erreur Redis lors du GET"""
        # Arrange
        self.mock_redis.get.side_effect = redis.ConnectionError("Connection refused")

        # Act & Assert
        with self.assertRaises(redis.ConnectionError):
            self.repo.get_by_id(123)

    def test_get_by_id_invalid_json(self):
        """Test : JSON invalide dans Redis"""
        # Arrange
        self.mock_redis.get.return_value = "invalid json {"

        # Act & Assert
        with self.assertRaises(json.JSONDecodeError):
            self.repo.get_by_id(123)

    # Tests SAVE
    def test_save_success(self):
        """Test : Sauvegarde réussie"""
        # Arrange
        user = {'id': 123, 'name': 'Alice', 'email': 'alice@example.com'}

        # Act
        result = self.repo.save(user, ttl=3600)

        # Assert
        self.assertTrue(result)
        self.mock_redis.setex.assert_called_once()

        # Vérifier les arguments exacts
        args, kwargs = self.mock_redis.setex.call_args
        self.assertEqual(args[0], 'user:123')
        self.assertEqual(args[1], 3600)
        self.assertEqual(json.loads(args[2]), user)

    def test_save_without_id(self):
        """Test : Sauvegarde sans ID (doit échouer)"""
        # Arrange
        user = {'name': 'Alice', 'email': 'alice@example.com'}

        # Act & Assert
        with self.assertRaises(ValueError) as context:
            self.repo.save(user)

        self.assertIn("must have an 'id'", str(context.exception))
        self.mock_redis.setex.assert_not_called()

    def test_save_redis_error(self):
        """Test : Erreur Redis lors du SAVE"""
        # Arrange
        user = {'id': 123, 'name': 'Alice'}
        self.mock_redis.setex.side_effect = redis.TimeoutError("Timeout")

        # Act & Assert
        with self.assertRaises(redis.TimeoutError):
            self.repo.save(user)

    # Tests DELETE
    def test_delete_success(self):
        """Test : Suppression réussie"""
        # Arrange
        self.mock_redis.delete.return_value = 1  # 1 clé supprimée

        # Act
        result = self.repo.delete(123)

        # Assert
        self.assertTrue(result)
        self.mock_redis.delete.assert_called_once_with('user:123')

    def test_delete_not_found(self):
        """Test : Suppression d'un utilisateur inexistant"""
        # Arrange
        self.mock_redis.delete.return_value = 0  # 0 clé supprimée

        # Act
        result = self.repo.delete(999)

        # Assert
        self.assertFalse(result)

    # Tests GET ALL (pipeline)
    def test_get_all_multiple_users(self):
        """Test : Récupération de plusieurs utilisateurs"""
        # Arrange
        user_ids = [1, 2, 3]

        # Mock du pipeline
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
        self.assertEqual(result[1]['name'], 'Alice')
        self.assertEqual(result[2]['name'], 'Bob')

        # Vérifier que get a été appelé 3 fois sur le pipeline
        self.assertEqual(mock_pipeline.get.call_count, 3)
        mock_pipeline.get.assert_has_calls([
            call('user:1'),
            call('user:2'),
            call('user:3')
        ])

    def test_get_all_empty_list(self):
        """Test : get_all avec liste vide"""
        # Act
        result = self.repo.get_all([])

        # Assert
        self.assertEqual(result, {})
        self.mock_redis.pipeline.assert_not_called()

    def test_get_all_pipeline_error(self):
        """Test : Erreur lors de l'exécution du pipeline"""
        # Arrange
        mock_pipeline = MagicMock()
        self.mock_redis.pipeline.return_value = mock_pipeline
        mock_pipeline.execute.side_effect = redis.ConnectionError("Connection lost")

        # Act & Assert
        with self.assertRaises(redis.ConnectionError):
            self.repo.get_all([1, 2, 3])

# Tests paramétrés avec subTest
class TestUserRepositoryParametrized(unittest.TestCase):
    """Tests paramétrés pour différents scénarios"""

    def setUp(self):
        self.mock_redis = Mock(spec=redis.Redis)
        self.repo = UserRepository(self.mock_redis)

    def test_get_by_id_various_ids(self):
        """Test avec différents IDs"""
        test_cases = [
            (1, {'id': 1, 'name': 'User1'}),
            (999, {'id': 999, 'name': 'User999'}),
            (12345, {'id': 12345, 'name': 'User12345'}),
        ]

        for user_id, expected_user in test_cases:
            with self.subTest(user_id=user_id):
                # Arrange
                self.mock_redis.get.return_value = json.dumps(expected_user)

                # Act
                result = self.repo.get_by_id(user_id)

                # Assert
                self.assertEqual(result, expected_user)
                self.mock_redis.get.assert_called_with(f'user:{user_id}')

                # Reset mock pour le prochain test
                self.mock_redis.reset_mock()

# Exécuter les tests
if __name__ == '__main__':
    # Mode verbose
    unittest.main(verbosity=2)
```

### Python : pytest (alternative moderne)

```python
import pytest
from unittest.mock import Mock, MagicMock
import redis
import json

# Fixtures pytest
@pytest.fixture
def mock_redis():
    """Fixture qui fournit un mock Redis"""
    return Mock(spec=redis.Redis)

@pytest.fixture
def user_repo(mock_redis):
    """Fixture qui fournit un repository avec mock Redis"""
    return UserRepository(mock_redis)

@pytest.fixture
def sample_user():
    """Fixture qui fournit un utilisateur de test"""
    return {
        'id': 123,
        'name': 'Alice',
        'email': 'alice@example.com'
    }

# Tests avec pytest
class TestUserRepositoryPytest:
    """Tests avec pytest (plus concis que unittest)"""

    def test_get_by_id_success(self, user_repo, mock_redis, sample_user):
        """Test récupération réussie"""
        # Arrange
        mock_redis.get.return_value = json.dumps(sample_user)

        # Act
        result = user_repo.get_by_id(123)

        # Assert
        assert result == sample_user
        mock_redis.get.assert_called_once_with('user:123')

    def test_get_by_id_not_found(self, user_repo, mock_redis):
        """Test utilisateur non trouvé"""
        mock_redis.get.return_value = None

        result = user_repo.get_by_id(999)

        assert result is None

    def test_get_by_id_redis_error(self, user_repo, mock_redis):
        """Test erreur Redis"""
        mock_redis.get.side_effect = redis.ConnectionError("Connection refused")

        with pytest.raises(redis.ConnectionError):
            user_repo.get_by_id(123)

    def test_save_success(self, user_repo, mock_redis, sample_user):
        """Test sauvegarde réussie"""
        result = user_repo.save(sample_user, ttl=3600)

        assert result is True
        mock_redis.setex.assert_called_once()

    def test_save_without_id(self, user_repo):
        """Test sauvegarde sans ID"""
        user_without_id = {'name': 'Alice'}

        with pytest.raises(ValueError, match="must have an 'id'"):
            user_repo.save(user_without_id)

    # Tests paramétrés avec pytest
    @pytest.mark.parametrize("user_id,expected_key", [
        (1, 'user:1'),
        (999, 'user:999'),
        (12345, 'user:12345'),
    ])
    def test_get_by_id_various_ids(self, user_repo, mock_redis, user_id, expected_key):
        """Test avec différents IDs (paramétré)"""
        mock_redis.get.return_value = json.dumps({'id': user_id})

        user_repo.get_by_id(user_id)

        mock_redis.get.assert_called_with(expected_key)

    # Test avec marqueurs
    @pytest.mark.slow
    def test_get_all_many_users(self, user_repo, mock_redis):
        """Test avec beaucoup d'utilisateurs (marqué comme lent)"""
        user_ids = list(range(1000))

        mock_pipeline = MagicMock()
        mock_redis.pipeline.return_value = mock_pipeline
        mock_pipeline.execute.return_value = [None] * 1000

        result = user_repo.get_all(user_ids)

        assert len(result) == 0
```

---

### Node.js : Jest avec ioredis-mock

```javascript
import { jest } from '@jest/globals';
import RedisMock from 'ioredis-mock';
import { UserRepository } from './UserRepository';

describe('UserRepository', () => {
    let redis;
    let repository;

    // Setup avant chaque test
    beforeEach(() => {
        // Utiliser ioredis-mock au lieu du vrai Redis
        redis = new RedisMock();
        repository = new UserRepository(redis);
    });

    // Cleanup après chaque test
    afterEach(async () => {
        await redis.flushall();
        await redis.disconnect();
    });

    describe('getById', () => {
        it('should return user when found', async () => {
            // Arrange
            const user = {
                id: 123,
                name: 'Alice',
                email: 'alice@example.com',
            };
            await redis.set('user:123', JSON.stringify(user));

            // Act
            const result = await repository.getById(123);

            // Assert
            expect(result).toEqual(user);
        });

        it('should return null when user not found', async () => {
            const result = await repository.getById(999);
            expect(result).toBeNull();
        });

        it('should throw error on Redis failure', async () => {
            // Mock d'erreur
            const mockGet = jest.spyOn(redis, 'get')
                .mockRejectedValue(new Error('Redis connection failed'));

            await expect(repository.getById(123))
                .rejects
                .toThrow('Redis connection failed');

            mockGet.mockRestore();
        });

        it('should throw error on invalid JSON', async () => {
            await redis.set('user:123', 'invalid json {');

            await expect(repository.getById(123))
                .rejects
                .toThrow();
        });
    });

    describe('save', () => {
        it('should save user with TTL', async () => {
            const user = {
                id: 123,
                name: 'Alice',
                email: 'alice@example.com',
            };

            const result = await repository.save(user, 3600);

            expect(result).toBe(true);

            // Vérifier que la clé existe
            const saved = await redis.get('user:123');
            expect(JSON.parse(saved)).toEqual(user);

            // Vérifier le TTL
            const ttl = await redis.ttl('user:123');
            expect(ttl).toBeGreaterThan(0);
            expect(ttl).toBeLessThanOrEqual(3600);
        });

        it('should throw error when user has no id', async () => {
            const user = { name: 'Alice', email: 'alice@example.com' };

            await expect(repository.save(user))
                .rejects
                .toThrow("User must have an 'id'");
        });

        it('should return false on Redis error', async () => {
            const user = { id: 123, name: 'Alice' };
            const mockSet = jest.spyOn(redis, 'set')
                .mockRejectedValue(new Error('Redis error'));

            const result = await repository.save(user);

            expect(result).toBe(false);
            mockSet.mockRestore();
        });
    });

    describe('delete', () => {
        it('should delete existing user', async () => {
            await redis.set('user:123', JSON.stringify({ id: 123 }));

            const result = await repository.delete(123);

            expect(result).toBe(true);

            const exists = await redis.exists('user:123');
            expect(exists).toBe(0);
        });

        it('should return false when user not found', async () => {
            const result = await repository.delete(999);
            expect(result).toBe(false);
        });
    });

    describe('getAll', () => {
        it('should return multiple users', async () => {
            // Arrange
            await redis.set('user:1', JSON.stringify({ id: 1, name: 'Alice' }));
            await redis.set('user:2', JSON.stringify({ id: 2, name: 'Bob' }));
            await redis.set('user:3', JSON.stringify({ id: 3, name: 'Charlie' }));

            // Act
            const result = await repository.getAll([1, 2, 3]);

            // Assert
            expect(result.size).toBe(3);
            expect(result.get(1)).toEqual({ id: 1, name: 'Alice' });
            expect(result.get(2)).toEqual({ id: 2, name: 'Bob' });
            expect(result.get(3)).toEqual({ id: 3, name: 'Charlie' });
        });

        it('should handle missing users', async () => {
            await redis.set('user:1', JSON.stringify({ id: 1, name: 'Alice' }));

            const result = await repository.getAll([1, 2, 3]);

            expect(result.size).toBe(1);
            expect(result.has(1)).toBe(true);
            expect(result.has(2)).toBe(false);
            expect(result.has(3)).toBe(false);
        });

        it('should return empty map for empty input', async () => {
            const result = await repository.getAll([]);
            expect(result.size).toBe(0);
        });
    });

    // Tests avec données paramétrées
    describe.each([
        [1, 'user:1'],
        [999, 'user:999'],
        [12345, 'user:12345'],
    ])('getById with various IDs', (userId, expectedKey) => {
        it(`should use correct key format for user ${userId}`, async () => {
            const user = { id: userId, name: `User${userId}` };
            await redis.set(expectedKey, JSON.stringify(user));

            const result = await repository.getById(userId);

            expect(result).toEqual(user);
        });
    });

    // Tests de snapshot (Jest)
    it('should match user snapshot', async () => {
        const user = {
            id: 123,
            name: 'Alice',
            email: 'alice@example.com',
            created: '2024-01-01',
        };

        await repository.save(user);
        const result = await repository.getById(123);

        expect(result).toMatchSnapshot();
    });
});

// Tests avec mocks purs (sans ioredis-mock)
describe('UserRepository with pure mocks', () => {
    let mockRedis;
    let repository;

    beforeEach(() => {
        mockRedis = {
            get: jest.fn(),
            set: jest.fn(),
            del: jest.fn(),
            pipeline: jest.fn(),
        };
        repository = new UserRepository(mockRedis);
    });

    it('should call Redis get with correct key', async () => {
        mockRedis.get.mockResolvedValue(JSON.stringify({ id: 123 }));

        await repository.getById(123);

        expect(mockRedis.get).toHaveBeenCalledWith('user:123');
        expect(mockRedis.get).toHaveBeenCalledTimes(1);
    });
});
```

---

## Tests d'intégration avec Redis réel

### Python : Tests d'intégration avec pytest

```python
import pytest
import redis
import time
from typing import Generator

# Configuration Redis de test
TEST_REDIS_HOST = 'localhost'
TEST_REDIS_PORT = 6379
TEST_REDIS_DB = 15  # DB séparée pour les tests

@pytest.fixture(scope='function')
def redis_client() -> Generator[redis.Redis, None, None]:
    """
    Fixture qui fournit un vrai client Redis pour les tests

    Scope 'function' = nouvelle instance par test
    """
    client = redis.Redis(
        host=TEST_REDIS_HOST,
        port=TEST_REDIS_PORT,
        db=TEST_REDIS_DB,
        decode_responses=True
    )

    # Setup : vérifier que Redis est disponible
    try:
        client.ping()
    except redis.ConnectionError:
        pytest.skip("Redis not available")

    # Nettoyer la DB de test avant le test
    client.flushdb()

    yield client

    # Teardown : nettoyer après le test
    client.flushdb()
    client.close()

@pytest.fixture(scope='function')
def user_repo_integration(redis_client):
    """Fixture qui fournit un repository avec vrai Redis"""
    return UserRepository(redis_client)

@pytest.mark.integration
class TestUserRepositoryIntegration:
    """Tests d'intégration avec Redis réel"""

    def test_save_and_get_roundtrip(self, user_repo_integration):
        """Test : Sauvegarde puis récupération (roundtrip)"""
        # Arrange
        user = {
            'id': 123,
            'name': 'Alice',
            'email': 'alice@example.com'
        }

        # Act : Sauvegarder
        save_result = user_repo_integration.save(user, ttl=60)

        # Act : Récupérer
        retrieved_user = user_repo_integration.get_by_id(123)

        # Assert
        assert save_result is True
        assert retrieved_user == user

    def test_ttl_expiration(self, user_repo_integration, redis_client):
        """Test : Vérification de l'expiration TTL"""
        # Arrange
        user = {'id': 123, 'name': 'Alice'}

        # Act : Sauvegarder avec TTL court
        user_repo_integration.save(user, ttl=2)

        # Assert : Clé existe
        assert user_repo_integration.get_by_id(123) is not None

        # Wait pour expiration
        time.sleep(3)

        # Assert : Clé a expiré
        assert user_repo_integration.get_by_id(123) is None

    def test_delete_removes_key(self, user_repo_integration, redis_client):
        """Test : La suppression retire bien la clé"""
        # Arrange
        user = {'id': 123, 'name': 'Alice'}
        user_repo_integration.save(user)

        # Act
        delete_result = user_repo_integration.delete(123)

        # Assert
        assert delete_result is True
        assert user_repo_integration.get_by_id(123) is None
        assert redis_client.exists('user:123') == 0

    def test_get_all_with_pipeline(self, user_repo_integration):
        """Test : get_all utilise bien le pipeline"""
        # Arrange
        users = [
            {'id': 1, 'name': 'Alice'},
            {'id': 2, 'name': 'Bob'},
            {'id': 3, 'name': 'Charlie'},
        ]

        for user in users:
            user_repo_integration.save(user)

        # Act
        result = user_repo_integration.get_all([1, 2, 3, 999])

        # Assert
        assert len(result) == 3
        assert result[1]['name'] == 'Alice'
        assert result[2]['name'] == 'Bob'
        assert result[3]['name'] == 'Charlie'
        assert 999 not in result

    def test_concurrent_writes(self, user_repo_integration):
        """Test : Écritures concurrentes"""
        import threading

        def save_user(user_id):
            user = {'id': user_id, 'name': f'User{user_id}'}
            user_repo_integration.save(user)

        # Créer 10 threads qui écrivent en parallèle
        threads = []
        for i in range(10):
            t = threading.Thread(target=save_user, args=(i,))
            threads.append(t)
            t.start()

        # Attendre que tous les threads terminent
        for t in threads:
            t.join()

        # Vérifier que tous les utilisateurs ont été sauvegardés
        result = user_repo_integration.get_all(list(range(10)))
        assert len(result) == 10

    def test_special_characters_in_values(self, user_repo_integration):
        """Test : Caractères spéciaux dans les valeurs"""
        user = {
            'id': 123,
            'name': 'Alice "The Boss" O\'Reilly',
            'bio': 'Loves: emojis 🎉, JSON {}, and \n newlines'
        }

        user_repo_integration.save(user)
        retrieved = user_repo_integration.get_by_id(123)

        assert retrieved == user

    def test_large_value(self, user_repo_integration):
        """Test : Valeur volumineuse"""
        large_bio = 'A' * 10000  # 10KB
        user = {
            'id': 123,
            'name': 'Alice',
            'bio': large_bio
        }

        user_repo_integration.save(user)
        retrieved = user_repo_integration.get_by_id(123)

        assert retrieved['bio'] == large_bio
        assert len(retrieved['bio']) == 10000

# Tests de performance
@pytest.mark.integration
@pytest.mark.slow
class TestUserRepositoryPerformance:
    """Tests de performance avec Redis réel"""

    def test_batch_operations_performance(self, user_repo_integration, benchmark):
        """Benchmark : get_all avec 100 utilisateurs"""
        # Setup
        for i in range(100):
            user = {'id': i, 'name': f'User{i}'}
            user_repo_integration.save(user)

        # Benchmark
        result = benchmark(
            user_repo_integration.get_all,
            list(range(100))
        )

        assert len(result) == 100

    def test_save_throughput(self, user_repo_integration, redis_client):
        """Test : Throughput des écritures"""
        import time

        num_operations = 1000
        user = {'id': 1, 'name': 'Alice'}

        start = time.time()
        for i in range(num_operations):
            user['id'] = i
            user_repo_integration.save(user)
        duration = time.time() - start

        ops_per_sec = num_operations / duration
        print(f"\n💾 Save throughput: {ops_per_sec:.0f} ops/sec")

        # Assert : Au moins 1000 ops/sec sur Redis local
        assert ops_per_sec > 1000
```

### Node.js : Tests d'intégration avec Jest

```javascript
import Redis from 'ioredis';
import { UserRepository } from './UserRepository';

// Configuration Redis de test
const TEST_REDIS_CONFIG = {
    host: 'localhost',
    port: 6379,
    db: 15, // DB séparée pour les tests
};

describe('UserRepository Integration Tests', () => {
    let redis;
    let repository;

    // Setup avant tous les tests
    beforeAll(async () => {
        redis = new Redis(TEST_REDIS_CONFIG);

        // Vérifier que Redis est disponible
        try {
            await redis.ping();
        } catch (err) {
            console.warn('⚠️ Redis not available, skipping integration tests');
            return;
        }
    });

    // Setup avant chaque test
    beforeEach(async () => {
        if (!redis) return;

        // Nettoyer la DB de test
        await redis.flushdb();

        repository = new UserRepository(redis);
    });

    // Cleanup après tous les tests
    afterAll(async () => {
        if (redis) {
            await redis.flushdb();
            await redis.quit();
        }
    });

    describe('save and get roundtrip', () => {
        it('should save and retrieve user', async () => {
            if (!redis) return;

            const user = {
                id: 123,
                firstName: 'Alice',
                lastName: 'Smith',
                email: 'alice@example.com',
            };

            await repository.save(user, 60);
            const retrieved = await repository.getById(123);

            expect(retrieved).toEqual(user);
        });
    });

    describe('TTL expiration', () => {
        it('should expire after TTL', async () => {
            if (!redis) return;

            const user = { id: 123, firstName: 'Alice' };

            await repository.save(user, 1); // 1 seconde

            // Vérifier existence
            let retrieved = await repository.getById(123);
            expect(retrieved).not.toBeNull();

            // Attendre expiration
            await new Promise(resolve => setTimeout(resolve, 1500));

            // Vérifier expiration
            retrieved = await repository.getById(123);
            expect(retrieved).toBeNull();
        }, 3000); // Timeout du test à 3s
    });

    describe('delete', () => {
        it('should remove key from Redis', async () => {
            if (!redis) return;

            const user = { id: 123, firstName: 'Alice' };
            await repository.save(user);

            const deleted = await repository.delete(123);

            expect(deleted).toBe(true);
            expect(await repository.getById(123)).toBeNull();
            expect(await redis.exists('user:123')).toBe(0);
        });
    });

    describe('getAll with pipeline', () => {
        it('should retrieve multiple users efficiently', async () => {
            if (!redis) return;

            const users = [
                { id: 1, firstName: 'Alice' },
                { id: 2, firstName: 'Bob' },
                { id: 3, firstName: 'Charlie' },
            ];

            for (const user of users) {
                await repository.save(user);
            }

            const result = await repository.getAll([1, 2, 3, 999]);

            expect(result.size).toBe(3);
            expect(result.get(1).firstName).toBe('Alice');
            expect(result.get(2).firstName).toBe('Bob');
            expect(result.get(3).firstName).toBe('Charlie');
            expect(result.has(999)).toBe(false);
        });
    });

    describe('performance', () => {
        it('should handle high throughput', async () => {
            if (!redis) return;

            const numOperations = 1000;
            const user = { id: 1, firstName: 'Alice' };

            const start = Date.now();

            for (let i = 0; i < numOperations; i++) {
                user.id = i;
                await repository.save(user);
            }

            const duration = Date.now() - start;
            const opsPerSec = (numOperations / duration) * 1000;

            console.log(`💾 Save throughput: ${opsPerSec.toFixed(0)} ops/sec`);

            // Au moins 500 ops/sec sur Redis local
            expect(opsPerSec).toBeGreaterThan(500);
        });
    });
});
```

---

## Tests avec Docker

### docker-compose.yml pour les tests

```yaml
version: '3.8'

services:
  redis-test:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-test-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  redis-test-data:
```

### Script de test avec Docker

```bash
#!/bin/bash
# test-with-docker.sh

set -e

echo "🐳 Starting Redis container..."
docker-compose up -d redis-test

echo "⏳ Waiting for Redis to be ready..."
timeout 30 bash -c 'until docker-compose exec -T redis-test redis-cli ping; do sleep 1; done'

echo "🧪 Running tests..."
pytest tests/integration/ -v

echo "🧹 Cleaning up..."
docker-compose down -v

echo "✅ Tests completed!"
```

---

## Tests avancés : Scénarios d'erreur

### Python : Simuler des pannes Redis

```python
import pytest
import redis
from unittest.mock import Mock, patch
import time

class TestRedisFailureScenarios:
    """Tests de scénarios de panne Redis"""

    def test_connection_refused(self):
        """Test : Redis indisponible dès le départ"""
        # Client avec mauvais port
        client = redis.Redis(host='localhost', port=9999)
        repo = UserRepository(client)

        with pytest.raises(redis.ConnectionError):
            repo.get_by_id(123)

    def test_connection_lost_during_operation(self):
        """Test : Perte de connexion pendant une opération"""
        mock_redis = Mock(spec=redis.Redis)
        repo = UserRepository(mock_redis)

        # Simuler une perte de connexion
        mock_redis.get.side_effect = redis.ConnectionError("Connection lost")

        with pytest.raises(redis.ConnectionError):
            repo.get_by_id(123)

    def test_timeout_error(self):
        """Test : Timeout sur opération longue"""
        mock_redis = Mock(spec=redis.Redis)
        repo = UserRepository(mock_redis)

        mock_redis.get.side_effect = redis.TimeoutError("Operation timed out")

        with pytest.raises(redis.TimeoutError):
            repo.get_by_id(123)

    def test_recovery_after_error(self):
        """Test : Récupération après une erreur"""
        mock_redis = Mock(spec=redis.Redis)
        repo = UserRepository(mock_redis)

        # Première tentative : erreur
        mock_redis.get.side_effect = [
            redis.ConnectionError("Connection lost"),
            json.dumps({'id': 123, 'name': 'Alice'})
        ]

        # Premier appel échoue
        with pytest.raises(redis.ConnectionError):
            repo.get_by_id(123)

        # Second appel réussit (après reconnexion)
        result = repo.get_by_id(123)
        assert result is not None

    @pytest.mark.integration
    def test_redis_restart_scenario(self, redis_client):
        """Test : Simulation d'un restart Redis"""
        repo = UserRepository(redis_client)

        # Sauvegarder un utilisateur
        user = {'id': 123, 'name': 'Alice'}
        repo.save(user)

        # Simuler un restart (flush)
        redis_client.flushdb()

        # L'utilisateur n'existe plus
        result = repo.get_by_id(123)
        assert result is None
```

---

## Configuration CI/CD

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run unit tests
        run: pytest tests/unit/ -v --cov

      - name: Run integration tests
        run: pytest tests/integration/ -v --cov-append
        env:
          REDIS_HOST: localhost
          REDIS_PORT: 6379

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

## Bonnes pratiques de testing

### ✅ 1. Isoler les tests

```python
# ✅ BON : Chaque test est isolé
def test_save_user(redis_client):
    redis_client.flushdb()  # Nettoyer avant
    repo = UserRepository(redis_client)
    # ... test
    redis_client.flushdb()  # Nettoyer après

# ❌ MAUVAIS : Tests dépendants
def test_save_then_get():
    # Ce test dépend de l'ordre d'exécution
    pass
```

### ✅ 2. Utiliser des fixtures

```python
# ✅ BON : Réutilisation via fixtures
@pytest.fixture
def redis_client():
    client = redis.Redis(db=15)
    client.flushdb()
    yield client
    client.flushdb()

def test_a(redis_client):
    pass

def test_b(redis_client):
    pass
```

### ✅ 3. Tester les cas limites

```python
def test_edge_cases():
    # Empty values
    repo.save({'id': 1, 'name': ''})

    # Special characters
    repo.save({'id': 2, 'name': 'O\'Reilly'})

    # Unicode
    repo.save({'id': 3, 'name': '🎉'})

    # Large values
    repo.save({'id': 4, 'bio': 'A' * 10000})
```

### ✅ 4. Séparer tests unitaires et d'intégration

```bash
tests/
├── unit/               # Mocks, rapides
│   ├── test_repository.py
│   └── test_service.py
└── integration/        # Redis réel, lents
    ├── test_repository_integration.py
    └── test_cache_integration.py
```

### ✅ 5. Utiliser des marqueurs

```python
# Tests lents
@pytest.mark.slow
def test_performance():
    pass

# Tests nécessitant Redis
@pytest.mark.integration
def test_with_redis():
    pass

# Tests instables
@pytest.mark.flaky
def test_timing_sensitive():
    pass

# Exécution sélective
# pytest -m "not slow"          # Exclure tests lents
# pytest -m integration         # Seulement intégration
```

---

## Anti-patterns à éviter

### ❌ 1. Tests qui modifient l'état global

```python
# ❌ MAUVAIS
def test_bad():
    global_redis_client.set('key', 'value')
    # Pollue l'état pour les autres tests
```

### ❌ 2. Tests sans assertions

```python
# ❌ MAUVAIS : Pas d'assertion
def test_bad():
    repo.save(user)
    # Oubli d'assert !

# ✅ BON
def test_good():
    result = repo.save(user)
    assert result is True
```

### ❌ 3. Tests qui dépendent du timing

```python
# ❌ MAUVAIS : Fragile
def test_bad():
    repo.save(user, ttl=1)
    time.sleep(0.9)  # Peut échouer selon le timing
    assert repo.get_by_id(123) is not None

# ✅ BON : Timing explicite
def test_good():
    repo.save(user, ttl=2)
    assert repo.get_by_id(123) is not None
    time.sleep(3)
    assert repo.get_by_id(123) is None
```

### ❌ 4. Mocks trop spécifiques

```python
# ❌ MAUVAIS : Mock trop détaillé
mock_redis.get.return_value = '{"id":123,"name":"Alice","email":"alice@example.com"}'
# Fragile aux changements

# ✅ BON : Mock avec données de test
@pytest.fixture
def sample_user():
    return {'id': 123, 'name': 'Alice'}

def test(mock_redis, sample_user):
    mock_redis.get.return_value = json.dumps(sample_user)
```

---

## Checklist de testing

Avant de considérer votre code bien testé :

### Tests unitaires
- ✅ Tous les cas normaux couverts
- ✅ Tous les cas d'erreur couverts
- ✅ Cas limites testés (empty, null, special chars)
- ✅ Mocks utilisés pour isolation
- ✅ Pas de dépendances Redis réelles
- ✅ Rapides (<1ms par test)

### Tests d'intégration
- ✅ Roundtrip complet testé (write + read)
- ✅ TTL vérifié
- ✅ Opérations batch testées
- ✅ Cleanup après chaque test
- ✅ DB de test séparée (ex: db=15)

### Scénarios d'erreur
- ✅ Connection refused
- ✅ Timeout
- ✅ Données invalides (JSON malformé)
- ✅ Clés manquantes
- ✅ Valeurs null

### CI/CD
- ✅ Tests automatisés dans pipeline
- ✅ Redis disponible via service container
- ✅ Coverage reporté (>80% recommandé)

### Documentation
- ✅ README avec instructions de test
- ✅ Tests documentés (docstrings)
- ✅ Fixtures documentées

---

## Points clés à retenir

🔑 **Pyramide des tests** : Beaucoup de tests unitaires, moins d'intégration

🔑 **Isolation** : Chaque test doit être indépendant

🔑 **Fixtures** : Réutiliser le setup avec fixtures

🔑 **Mocks pour unité** : ioredis-mock, unittest.mock

🔑 **Redis réel pour intégration** : DB séparée (db=15)

🔑 **Docker pour CI** : Service container dans GitHub Actions

🔑 **Marqueurs** : Séparer tests lents/rapides

🔑 **Cleanup** : Toujours nettoyer après les tests

🔑 **Coverage** : Viser >80% de couverture

---

## Ressources complémentaires

- **pytest documentation** : https://docs.pytest.org/
- **Jest documentation** : https://jestjs.io/
- **ioredis-mock** : https://github.com/stipsan/ioredis-mock
- **Testing Best Practices** : https://testingjavascript.com/

---

## Conclusion du module 9

Vous avez maintenant une compréhension complète de l'intégration de Redis avec les langages de programmation :

1. **Clients Redis** (9.1) : Panorama et choix
2. **Pools de connexions** (9.2) : Performance et scalabilité
3. **Gestion d'erreurs** (9.3) : Résilience et retry logic
4. **Async/Await** (9.4) : Programmation asynchrone
5. **Bonnes pratiques** (9.5) : Code maintenable
6. **Testing** (9.6) : Qualité et fiabilité

**Prochains modules** : Architecture haute disponibilité, Cluster, Production

**Niveau** : Intermédiaire
**Durée estimée** : 90 minutes
**Prérequis** : Sections 9.1-9.5

⏭️ [Architecture Haute Disponibilité (DevOps)](/10-architecture-haute-disponibilite/README.md)
