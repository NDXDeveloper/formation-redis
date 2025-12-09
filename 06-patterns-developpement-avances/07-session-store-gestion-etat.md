🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.7 Session Store et gestion d'état

## Introduction

La **gestion des sessions** est fondamentale dans toute application web moderne. Redis est devenu le standard de facto pour stocker les sessions grâce à sa rapidité, son support natif de TTL, et sa capacité à gérer des millions de sessions actives simultanément.

Cette section explore les différents patterns de session management, de la simple session store aux architectures distribuées complexes, avec des implémentations production-ready.

## Pourquoi Redis pour les sessions ?

### Comparaison des approches

**Client-Side (Cookies/LocalStorage)**

```text
Avantages:
✅ Pas de stockage serveur
✅ Scalabilité infinie
✅ Pas de synchronisation

Inconvénients:
❌ Taille limitée (4KB cookies)
❌ Sécurité (données exposées)
❌ Pas de contrôle serveur
❌ Bandwidth (envoyé à chaque requête)
```

**Server-Side (Memory)**

```text
Avantages:
✅ Sécurisé (données côté serveur)
✅ Taille illimitée
✅ Contrôle total

Inconvénients:
❌ Perdu au redémarrage
❌ Pas de partage entre instances
❌ Consomme RAM serveur
❌ Pas scalable horizontalement
```

**Server-Side (Redis)**

```text
Avantages:
✅ Rapide (in-memory)
✅ Persistant (optionnel)
✅ Partagé entre instances
✅ TTL automatique
✅ Scalable horizontalement
✅ Clustering disponible

Inconvénients:
⚠️ Infrastructure additionnelle
⚠️ Latence réseau minimale
```

### Architecture typique

```text
┌─────────────────────────────────────────────────────────────┐
│            REDIS SESSION STORE ARCHITECTURE                  │
└─────────────────────────────────────────────────────────────┘

User Browser                                          Redis
     │                                                  │
     │  1. HTTP Request                                 │
     │     + Cookie: session_id=abc123                  │
     ├────────────────────────────>                     │
     │                             │                    │
     │                     App Server 1                 │
     │                             │                    │
     │                             ├─ GET session:abc123
     │                             │───────────────────>│
     │                             │                    │
     │                             │<─ User data ───────┤
     │                             │                    │
     │  2. HTTP Response           │                    │
     │<─────────────────────────────                    │
     │                                                  │
     │  3. HTTP Request                                 │
     │     + Cookie: session_id=abc123                  │
     ├────────────────────────────────────────────>     │
     │                                     │            │
     │                             App Server 2         │
     │                                     │            │
     │                                     ├─ GET session:abc123
     │                                     │───────────>│
     │                                     │            │
     │                                     │<─ Same data┤
     │                                     │            │
     │  4. HTTP Response                   │            │
     │<─────────────────────────────────────            │

✅ Same session across multiple servers
✅ No sticky sessions needed
✅ Instant session access
```

---

## Pattern 1: Basic Session Store

### Structure de session

```text
Session Key: session:abc123def456...

Session Data (Hash):
┌─────────────────────────────────────────┐
│ user_id          : 12345                │
│ username         : alice                │
│ email            : alice@example.com    │
│ role             : admin                │
│ login_time       : 1670000000           │
│ last_activity    : 1670003600           │
│ ip_address       : 192.168.1.1          │
│ user_agent       : Mozilla/5.0...       │
│ csrf_token       : xyz789...            │
│ shopping_cart    : {"items":[...]}      │
└─────────────────────────────────────────┘

TTL: 1800 seconds (30 minutes)
```

### Implémentation Python (Flask)

```python
import redis
import uuid
import json
import time
from datetime import datetime, timedelta
from functools import wraps
from flask import Flask, request, make_response, jsonify

class RedisSessionStore:
    """
    Session store Redis pour Flask
    """

    def __init__(self, redis_client, prefix='session:', ttl=1800):
        """
        Args:
            redis_client: Client Redis
            prefix: Préfixe pour les clés de session
            ttl: Durée de vie en secondes (30 min par défaut)
        """
        self.redis = redis_client
        self.prefix = prefix
        self.ttl = ttl

    def create_session(self, user_data):
        """
        Crée une nouvelle session

        Args:
            user_data: Dictionnaire avec les données utilisateur

        Returns:
            str: ID de session
        """
        session_id = str(uuid.uuid4())
        session_key = f"{self.prefix}{session_id}"

        # Enrichir avec métadonnées
        session_data = {
            **user_data,
            'created_at': time.time(),
            'last_activity': time.time(),
            'ip_address': request.remote_addr if request else 'unknown',
            'user_agent': request.headers.get('User-Agent', 'unknown') if request else 'unknown'
        }

        # Stocker dans Redis (Hash)
        self.redis.hset(session_key, mapping=session_data)
        self.redis.expire(session_key, self.ttl)

        print(f"✓ Session created: {session_id}")
        print(f"  User: {user_data.get('username', 'unknown')}")
        print(f"  TTL: {self.ttl}s")

        return session_id

    def get_session(self, session_id):
        """
        Récupère une session

        Returns:
            dict or None: Données de session
        """
        if not session_id:
            return None

        session_key = f"{self.prefix}{session_id}"

        # Récupérer toutes les données de la session
        session_data = self.redis.hgetall(session_key)

        if not session_data:
            print(f"✗ Session not found: {session_id}")
            return None

        # Mettre à jour last_activity et prolonger TTL
        self.redis.hset(session_key, 'last_activity', time.time())
        self.redis.expire(session_key, self.ttl)

        return session_data

    def update_session(self, session_id, data):
        """
        Met à jour des données de session
        """
        if not session_id:
            return False

        session_key = f"{self.prefix}{session_id}"

        if not self.redis.exists(session_key):
            return False

        # Mettre à jour les données
        self.redis.hset(session_key, mapping=data)
        self.redis.hset(session_key, 'last_activity', time.time())
        self.redis.expire(session_key, self.ttl)

        print(f"✓ Session updated: {session_id}")
        return True

    def delete_session(self, session_id):
        """
        Supprime une session (logout)
        """
        if not session_id:
            return False

        session_key = f"{self.prefix}{session_id}"
        deleted = self.redis.delete(session_key)

        if deleted:
            print(f"✓ Session deleted: {session_id}")

        return bool(deleted)

    def get_all_user_sessions(self, user_id):
        """
        Récupère toutes les sessions d'un utilisateur
        (utile pour déconnexion de tous les appareils)
        """
        pattern = f"{self.prefix}*"
        sessions = []

        for key in self.redis.scan_iter(match=pattern):
            session_data = self.redis.hgetall(key)
            if session_data.get('user_id') == str(user_id):
                session_id = key.decode('utf-8').replace(self.prefix, '')
                sessions.append({
                    'session_id': session_id,
                    'data': session_data
                })

        return sessions

    def delete_all_user_sessions(self, user_id):
        """
        Supprime toutes les sessions d'un utilisateur
        """
        sessions = self.get_all_user_sessions(user_id)
        count = 0

        for session in sessions:
            if self.delete_session(session['session_id']):
                count += 1

        print(f"✓ Deleted {count} sessions for user {user_id}")
        return count


# ============================================================
# FLASK INTEGRATION
# ============================================================

app = Flask(__name__)
redis_client = redis.Redis(decode_responses=True)
session_store = RedisSessionStore(redis_client, ttl=1800)

def get_session_id_from_cookie():
    """Récupère le session_id du cookie"""
    return request.cookies.get('session_id')

def set_session_cookie(response, session_id):
    """Définit le cookie de session"""
    response.set_cookie(
        'session_id',
        session_id,
        max_age=1800,  # 30 minutes
        httponly=True,  # Pas accessible en JavaScript
        secure=True,    # HTTPS uniquement (en production)
        samesite='Lax'  # Protection CSRF
    )
    return response

def require_session(func):
    """Decorator pour protéger les routes nécessitant une session"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        session_id = get_session_id_from_cookie()
        session_data = session_store.get_session(session_id)

        if not session_data:
            return jsonify({'error': 'Unauthorized', 'message': 'Please login'}), 401

        # Ajouter les données de session à la requête
        request.session = session_data

        return func(*args, **kwargs)

    return wrapper


# ============================================================
# ROUTES
# ============================================================

@app.route('/login', methods=['POST'])
def login():
    """Endpoint de login"""
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    # Validation (simplifié)
    if not username or not password:
        return jsonify({'error': 'Missing credentials'}), 400

    # Vérifier les credentials (simplifié)
    # En production : vérifier dans la DB avec hash bcrypt
    if password == 'demo123':
        # Créer la session
        user_data = {
            'user_id': '12345',
            'username': username,
            'email': f'{username}@example.com',
            'role': 'user'
        }

        session_id = session_store.create_session(user_data)

        # Retourner la réponse avec cookie
        response = make_response(jsonify({
            'success': True,
            'message': 'Login successful',
            'user': user_data
        }))

        return set_session_cookie(response, session_id)
    else:
        return jsonify({'error': 'Invalid credentials'}), 401


@app.route('/profile')
@require_session
def profile():
    """Endpoint protégé nécessitant une session"""
    return jsonify({
        'user': {
            'id': request.session.get('user_id'),
            'username': request.session.get('username'),
            'email': request.session.get('email'),
            'role': request.session.get('role')
        },
        'session_info': {
            'created_at': request.session.get('created_at'),
            'last_activity': request.session.get('last_activity')
        }
    })


@app.route('/update-profile', methods=['POST'])
@require_session
def update_profile():
    """Mise à jour du profil"""
    data = request.get_json()
    session_id = get_session_id_from_cookie()

    # Mettre à jour les données de session
    update_data = {
        'email': data.get('email', request.session.get('email'))
    }

    session_store.update_session(session_id, update_data)

    return jsonify({'success': True, 'message': 'Profile updated'})


@app.route('/logout', methods=['POST'])
@require_session
def logout():
    """Endpoint de logout"""
    session_id = get_session_id_from_cookie()
    session_store.delete_session(session_id)

    response = make_response(jsonify({
        'success': True,
        'message': 'Logout successful'
    }))

    # Supprimer le cookie
    response.set_cookie('session_id', '', expires=0)

    return response


@app.route('/logout-all', methods=['POST'])
@require_session
def logout_all():
    """Déconnexion de tous les appareils"""
    user_id = request.session.get('user_id')
    count = session_store.delete_all_user_sessions(user_id)

    response = make_response(jsonify({
        'success': True,
        'message': f'Logged out from {count} devices'
    }))

    response.set_cookie('session_id', '', expires=0)

    return response


# ============================================================
# SESSION MONITORING
# ============================================================

@app.route('/admin/sessions')
def admin_sessions():
    """Liste toutes les sessions actives (admin)"""
    pattern = f"{session_store.prefix}*"
    sessions = []

    for key in redis_client.scan_iter(match=pattern):
        session_data = redis_client.hgetall(key)
        ttl = redis_client.ttl(key)

        sessions.append({
            'session_id': key.decode('utf-8').replace(session_store.prefix, ''),
            'username': session_data.get('username'),
            'last_activity': session_data.get('last_activity'),
            'ttl': ttl
        })

    return jsonify({
        'total': len(sessions),
        'sessions': sessions
    })
```

---

## Pattern 2: Session avec données complexes

### JSON Encoding pour données complexes

```python
import json

class EnhancedRedisSessionStore(RedisSessionStore):
    """
    Session store avec support de types complexes
    """

    def create_session(self, user_data):
        """Crée une session avec sérialisation JSON"""
        session_id = str(uuid.uuid4())
        session_key = f"{self.prefix}{session_id}"

        session_data = {
            **user_data,
            'created_at': time.time(),
            'last_activity': time.time(),
            'ip_address': request.remote_addr if request else 'unknown',
            'user_agent': request.headers.get('User-Agent', '') if request else ''
        }

        # Sérialiser les données complexes en JSON
        serialized_data = {}
        for key, value in session_data.items():
            if isinstance(value, (dict, list)):
                serialized_data[key] = json.dumps(value)
            else:
                serialized_data[key] = str(value)

        self.redis.hset(session_key, mapping=serialized_data)
        self.redis.expire(session_key, self.ttl)

        return session_id

    def get_session(self, session_id):
        """Récupère et désérialise les données"""
        if not session_id:
            return None

        session_key = f"{self.prefix}{session_id}"
        session_data = self.redis.hgetall(session_key)

        if not session_data:
            return None

        # Désérialiser les valeurs JSON
        deserialized_data = {}
        for key, value in session_data.items():
            try:
                # Tenter de parser en JSON
                deserialized_data[key] = json.loads(value)
            except (json.JSONDecodeError, TypeError):
                # Garder comme string si pas JSON
                deserialized_data[key] = value

        # Mettre à jour last_activity
        self.redis.hset(session_key, 'last_activity', time.time())
        self.redis.expire(session_key, self.ttl)

        return deserialized_data


# ============================================================
# EXEMPLE AVEC SHOPPING CART
# ============================================================

class ShoppingCartSession:
    """
    Session avec panier d'achat
    """

    def __init__(self, session_store):
        self.session_store = session_store

    def add_to_cart(self, session_id, item):
        """Ajoute un produit au panier"""
        session = self.session_store.get_session(session_id)

        if not session:
            return False

        # Récupérer le panier actuel
        cart = session.get('cart', [])
        if isinstance(cart, str):
            cart = json.loads(cart)

        # Vérifier si l'item existe déjà
        existing_item = next(
            (x for x in cart if x['product_id'] == item['product_id']),
            None
        )

        if existing_item:
            # Augmenter la quantité
            existing_item['quantity'] += item.get('quantity', 1)
        else:
            # Ajouter le nouvel item
            cart.append({
                'product_id': item['product_id'],
                'name': item['name'],
                'price': item['price'],
                'quantity': item.get('quantity', 1)
            })

        # Mettre à jour la session
        self.session_store.update_session(session_id, {'cart': cart})

        print(f"✓ Item added to cart: {item['name']}")
        return True

    def get_cart(self, session_id):
        """Récupère le panier"""
        session = self.session_store.get_session(session_id)

        if not session:
            return []

        cart = session.get('cart', [])
        if isinstance(cart, str):
            cart = json.loads(cart)

        return cart

    def clear_cart(self, session_id):
        """Vide le panier"""
        self.session_store.update_session(session_id, {'cart': []})
        print(f"✓ Cart cleared for session {session_id}")
        return True

    def calculate_total(self, session_id):
        """Calcule le total du panier"""
        cart = self.get_cart(session_id)
        total = sum(item['price'] * item['quantity'] for item in cart)
        return total


# Routes pour le panier
@app.route('/cart', methods=['GET'])
@require_session
def get_cart():
    """Récupère le panier"""
    session_id = get_session_id_from_cookie()
    cart_session = ShoppingCartSession(session_store)

    cart = cart_session.get_cart(session_id)
    total = cart_session.calculate_total(session_id)

    return jsonify({
        'cart': cart,
        'total': total,
        'item_count': sum(item['quantity'] for item in cart)
    })


@app.route('/cart/add', methods=['POST'])
@require_session
def add_to_cart():
    """Ajoute un produit au panier"""
    session_id = get_session_id_from_cookie()
    cart_session = ShoppingCartSession(session_store)

    item = request.get_json()
    cart_session.add_to_cart(session_id, item)

    return jsonify({'success': True, 'message': 'Item added to cart'})
```

---

## Pattern 3: Session avec TTL sliding

### Auto-prolongation du TTL

```python
class SlidingSessionStore(RedisSessionStore):
    """
    Session store avec TTL sliding (auto-prolongation)
    """

    def __init__(self, redis_client, prefix='session:',
                 ttl=1800, sliding_window=300):
        """
        Args:
            ttl: Durée totale de la session
            sliding_window: Fenêtre de prolongation (5 min)
        """
        super().__init__(redis_client, prefix, ttl)
        self.sliding_window = sliding_window

    def get_session(self, session_id):
        """
        Récupère la session et prolonge le TTL si nécessaire
        """
        if not session_id:
            return None

        session_key = f"{self.prefix}{session_id}"
        session_data = self.redis.hgetall(session_key)

        if not session_data:
            return None

        # Vérifier le TTL restant
        ttl_remaining = self.redis.ttl(session_key)

        # Si moins de 5 minutes restantes, prolonger
        if ttl_remaining < self.sliding_window:
            self.redis.expire(session_key, self.ttl)
            print(f"🔄 Session TTL extended: {session_id} (+{self.ttl}s)")

        # Mettre à jour last_activity
        self.redis.hset(session_key, 'last_activity', time.time())

        return session_data
```

### Visualisation du TTL sliding

```text
Timeline avec Sliding TTL:
─────────────────────────────────────────────────────────────

Scenario: TTL = 30 min, Sliding window = 5 min

t=0     Login
        ├─ Session created
        └─ TTL = 30 min (expire at t=30)

t=10    Activity
        ├─ TTL remaining = 20 min
        └─ No extension (> 5 min remaining)

t=26    Activity
        ├─ TTL remaining = 4 min (< 5 min)
        └─ Extend TTL to 30 min (expire at t=56) ✅

t=50    Activity
        ├─ TTL remaining = 6 min
        └─ No extension

t=52    Activity
        ├─ TTL remaining = 4 min (< 5 min)
        └─ Extend TTL to 30 min (expire at t=82) ✅

Result: Session stays alive as long as user is active!
```

---

## Pattern 4: Distributed Sessions

### Session Replication

```python
class ReplicatedSessionStore:
    """
    Session store avec réplication sur plusieurs Redis
    """

    def __init__(self, redis_instances, prefix='session:', ttl=1800):
        """
        Args:
            redis_instances: Liste de clients Redis
        """
        self.instances = redis_instances
        self.prefix = prefix
        self.ttl = ttl

    def create_session(self, user_data):
        """Crée une session sur toutes les instances"""
        session_id = str(uuid.uuid4())
        session_key = f"{self.prefix}{session_id}"

        session_data = {
            **user_data,
            'created_at': time.time(),
            'last_activity': time.time()
        }

        # Écrire sur toutes les instances
        success_count = 0
        for instance in self.instances:
            try:
                instance.hset(session_key, mapping=session_data)
                instance.expire(session_key, self.ttl)
                success_count += 1
            except Exception as e:
                print(f"⚠️  Failed to replicate session: {e}")

        print(f"✓ Session replicated to {success_count}/{len(self.instances)} instances")

        return session_id

    def get_session(self, session_id):
        """Récupère depuis la première instance disponible"""
        if not session_id:
            return None

        session_key = f"{self.prefix}{session_id}"

        # Essayer chaque instance
        for i, instance in enumerate(self.instances):
            try:
                session_data = instance.hgetall(session_key)
                if session_data:
                    print(f"✓ Session found on instance {i}")

                    # Mettre à jour last_activity sur toutes les instances
                    for inst in self.instances:
                        try:
                            inst.hset(session_key, 'last_activity', time.time())
                            inst.expire(session_key, self.ttl)
                        except:
                            pass

                    return session_data
            except Exception as e:
                print(f"⚠️  Instance {i} failed: {e}")
                continue

        print(f"✗ Session not found on any instance: {session_id}")
        return None

    def delete_session(self, session_id):
        """Supprime de toutes les instances"""
        if not session_id:
            return False

        session_key = f"{self.prefix}{session_id}"
        deleted_count = 0

        for instance in self.instances:
            try:
                if instance.delete(session_key):
                    deleted_count += 1
            except:
                pass

        print(f"✓ Session deleted from {deleted_count}/{len(self.instances)} instances")
        return deleted_count > 0
```

---

## Implémentation Node.js

### Express Session avec Redis

```javascript
const express = require('express');
const Redis = require('ioredis');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { v4: uuidv4 } = require('uuid');

const app = express();
const redis = new Redis();

// ============================================================
// CONFIGURATION EXPRESS-SESSION
// ============================================================

app.use(session({
    store: new RedisStore({ client: redis }),
    secret: 'your-secret-key-change-in-production',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: false,      // true en production avec HTTPS
        httpOnly: true,     // Protection XSS
        maxAge: 1800000,    // 30 minutes
        sameSite: 'lax'     // Protection CSRF
    },
    name: 'sessionId'       // Nom du cookie
}));

app.use(express.json());

// ============================================================
// CUSTOM SESSION STORE
// ============================================================

class CustomRedisSessionStore {
    constructor(redis, options = {}) {
        this.redis = redis;
        this.prefix = options.prefix || 'session:';
        this.ttl = options.ttl || 1800; // 30 minutes
    }

    async createSession(userData) {
        const sessionId = uuidv4();
        const sessionKey = `${this.prefix}${sessionId}`;

        const sessionData = {
            ...userData,
            createdAt: Date.now(),
            lastActivity: Date.now()
        };

        await this.redis.hmset(sessionKey, sessionData);
        await this.redis.expire(sessionKey, this.ttl);

        console.log(`✓ Session created: ${sessionId}`);
        return sessionId;
    }

    async getSession(sessionId) {
        if (!sessionId) return null;

        const sessionKey = `${this.prefix}${sessionId}`;
        const sessionData = await this.redis.hgetall(sessionKey);

        if (!sessionData || Object.keys(sessionData).length === 0) {
            console.log(`✗ Session not found: ${sessionId}`);
            return null;
        }

        // Update last activity and extend TTL
        await this.redis.hset(sessionKey, 'lastActivity', Date.now());
        await this.redis.expire(sessionKey, this.ttl);

        return sessionData;
    }

    async updateSession(sessionId, data) {
        if (!sessionId) return false;

        const sessionKey = `${this.prefix}${sessionId}`;

        if (!(await this.redis.exists(sessionKey))) {
            return false;
        }

        await this.redis.hmset(sessionKey, data);
        await this.redis.hset(sessionKey, 'lastActivity', Date.now());
        await this.redis.expire(sessionKey, this.ttl);

        console.log(`✓ Session updated: ${sessionId}`);
        return true;
    }

    async deleteSession(sessionId) {
        if (!sessionId) return false;

        const sessionKey = `${this.prefix}${sessionId}`;
        const deleted = await this.redis.del(sessionKey);

        if (deleted) {
            console.log(`✓ Session deleted: ${sessionId}`);
        }

        return Boolean(deleted);
    }

    async getAllUserSessions(userId) {
        const pattern = `${this.prefix}*`;
        const sessions = [];

        const keys = await this.redis.keys(pattern);

        for (const key of keys) {
            const sessionData = await this.redis.hgetall(key);
            if (sessionData.userId === userId.toString()) {
                const sessionId = key.replace(this.prefix, '');
                sessions.push({
                    sessionId,
                    data: sessionData
                });
            }
        }

        return sessions;
    }

    async deleteAllUserSessions(userId) {
        const sessions = await this.getAllUserSessions(userId);
        let count = 0;

        for (const session of sessions) {
            if (await this.deleteSession(session.sessionId)) {
                count++;
            }
        }

        console.log(`✓ Deleted ${count} sessions for user ${userId}`);
        return count;
    }
}

const sessionStore = new CustomRedisSessionStore(redis);

// ============================================================
// MIDDLEWARE
// ============================================================

function requireSession(req, res, next) {
    if (!req.session || !req.session.userId) {
        return res.status(401).json({
            error: 'Unauthorized',
            message: 'Please login'
        });
    }
    next();
}

// ============================================================
// ROUTES
// ============================================================

app.post('/login', async (req, res) => {
    const { username, password } = req.body;

    if (!username || !password) {
        return res.status(400).json({ error: 'Missing credentials' });
    }

    // Validate credentials (simplified)
    if (password === 'demo123') {
        // Store in session
        req.session.userId = '12345';
        req.session.username = username;
        req.session.email = `${username}@example.com`;
        req.session.role = 'user';

        await new Promise((resolve, reject) => {
            req.session.save((err) => {
                if (err) reject(err);
                else resolve();
            });
        });

        res.json({
            success: true,
            message: 'Login successful',
            user: {
                userId: req.session.userId,
                username: req.session.username,
                email: req.session.email
            }
        });
    } else {
        res.status(401).json({ error: 'Invalid credentials' });
    }
});

app.get('/profile', requireSession, (req, res) => {
    res.json({
        user: {
            userId: req.session.userId,
            username: req.session.username,
            email: req.session.email,
            role: req.session.role
        },
        session: {
            id: req.sessionID,
            cookie: req.session.cookie
        }
    });
});

app.post('/logout', requireSession, (req, res) => {
    req.session.destroy((err) => {
        if (err) {
            return res.status(500).json({ error: 'Logout failed' });
        }

        res.clearCookie('sessionId');
        res.json({
            success: true,
            message: 'Logout successful'
        });
    });
});

// ============================================================
// SHOPPING CART
// ============================================================

app.get('/cart', requireSession, (req, res) => {
    const cart = req.session.cart || [];
    const total = cart.reduce((sum, item) => sum + (item.price * item.quantity), 0);

    res.json({
        cart,
        total,
        itemCount: cart.reduce((sum, item) => sum + item.quantity, 0)
    });
});

app.post('/cart/add', requireSession, (req, res) => {
    const item = req.body;

    if (!req.session.cart) {
        req.session.cart = [];
    }

    // Check if item already in cart
    const existingItem = req.session.cart.find(
        x => x.productId === item.productId
    );

    if (existingItem) {
        existingItem.quantity += item.quantity || 1;
    } else {
        req.session.cart.push({
            productId: item.productId,
            name: item.name,
            price: item.price,
            quantity: item.quantity || 1
        });
    }

    req.session.save((err) => {
        if (err) {
            return res.status(500).json({ error: 'Failed to update cart' });
        }

        res.json({
            success: true,
            message: 'Item added to cart',
            cart: req.session.cart
        });
    });
});

app.delete('/cart/clear', requireSession, (req, res) => {
    req.session.cart = [];

    req.session.save((err) => {
        if (err) {
            return res.status(500).json({ error: 'Failed to clear cart' });
        }

        res.json({
            success: true,
            message: 'Cart cleared'
        });
    });
});

// ============================================================
// SESSION MONITORING
// ============================================================

app.get('/admin/sessions', async (req, res) => {
    const pattern = 'sess:*';  // express-session prefix
    const keys = await redis.keys(pattern);
    const sessions = [];

    for (const key of keys) {
        const sessionData = await redis.get(key);
        const ttl = await redis.ttl(key);

        try {
            const parsed = JSON.parse(sessionData);
            sessions.push({
                sessionId: key.replace('sess:', ''),
                username: parsed.username,
                userId: parsed.userId,
                ttl: ttl
            });
        } catch (e) {
            // Skip invalid sessions
        }
    }

    res.json({
        total: sessions.length,
        sessions
    });
});

// Start server
const PORT = 3000;
app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
```

---

## Sécurité des sessions

### Checklist de sécurité

**Configuration des cookies :**

```python
# ✅ GOOD: Cookie sécurisé
response.set_cookie(
    'session_id',
    session_id,
    max_age=1800,
    httponly=True,    # ✅ Pas accessible en JavaScript (XSS protection)
    secure=True,      # ✅ HTTPS uniquement
    samesite='Lax'    # ✅ CSRF protection
)

# ❌ BAD: Cookie non sécurisé
response.set_cookie(
    'session_id',
    session_id
)
# Vulnérable à XSS, CSRF, man-in-the-middle
```

### Protection CSRF

```python
import hmac
import hashlib

def generate_csrf_token(session_id, secret_key):
    """Génère un token CSRF lié à la session"""
    return hmac.new(
        secret_key.encode(),
        session_id.encode(),
        hashlib.sha256
    ).hexdigest()

def validate_csrf_token(session_id, token, secret_key):
    """Valide le token CSRF"""
    expected_token = generate_csrf_token(session_id, secret_key)
    return hmac.compare_digest(expected_token, token)

# Middleware Flask
def csrf_protect(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        if request.method in ['POST', 'PUT', 'DELETE']:
            session_id = get_session_id_from_cookie()
            csrf_token = request.headers.get('X-CSRF-Token')

            if not csrf_token or not validate_csrf_token(
                session_id, csrf_token, app.config['SECRET_KEY']
            ):
                return jsonify({'error': 'Invalid CSRF token'}), 403

        return func(*args, **kwargs)
    return wrapper
```

### Session Hijacking Protection

```python
class SecureSessionStore(RedisSessionStore):
    """
    Session store avec protection contre le hijacking
    """

    def create_session(self, user_data):
        """Crée une session avec fingerprint"""
        session_id = super().create_session(user_data)
        session_key = f"{self.prefix}{session_id}"

        # Stocker le fingerprint du client
        fingerprint = self._generate_fingerprint()
        self.redis.hset(session_key, 'fingerprint', fingerprint)

        return session_id

    def get_session(self, session_id):
        """Vérifie le fingerprint à chaque requête"""
        session_data = super().get_session(session_id)

        if not session_data:
            return None

        # Vérifier le fingerprint
        stored_fingerprint = session_data.get('fingerprint')
        current_fingerprint = self._generate_fingerprint()

        if stored_fingerprint != current_fingerprint:
            print(f"⚠️  Session hijacking detected: {session_id}")
            self.delete_session(session_id)
            return None

        return session_data

    def _generate_fingerprint(self):
        """Génère un fingerprint du client"""
        if not request:
            return 'unknown'

        # Combiner plusieurs attributs
        components = [
            request.remote_addr,
            request.headers.get('User-Agent', ''),
            request.headers.get('Accept-Language', '')
        ]

        fingerprint_string = '|'.join(components)

        return hashlib.sha256(
            fingerprint_string.encode()
        ).hexdigest()
```

---

## Session Cleanup et Monitoring

### Automatic Cleanup

```python
import threading
import time

class SessionCleanupWorker:
    """
    Worker pour nettoyer les sessions expirées
    """

    def __init__(self, redis_client, prefix='session:', interval=300):
        """
        Args:
            interval: Intervalle de nettoyage (secondes)
        """
        self.redis = redis_client
        self.prefix = prefix
        self.interval = interval
        self.running = False

    def start(self):
        """Démarre le worker de nettoyage"""
        self.running = True
        thread = threading.Thread(target=self._cleanup_loop, daemon=True)
        thread.start()
        print(f"✓ Session cleanup worker started (interval: {self.interval}s)")

    def stop(self):
        """Arrête le worker"""
        self.running = False
        print("✓ Session cleanup worker stopped")

    def _cleanup_loop(self):
        """Boucle de nettoyage"""
        while self.running:
            try:
                self._cleanup_expired_sessions()
            except Exception as e:
                print(f"⚠️  Cleanup error: {e}")

            time.sleep(self.interval)

    def _cleanup_expired_sessions(self):
        """Nettoie les sessions expirées"""
        pattern = f"{self.prefix}*"
        cleaned = 0

        for key in self.redis.scan_iter(match=pattern):
            # Vérifier si la clé existe encore (peut avoir expiré)
            if not self.redis.exists(key):
                cleaned += 1

        if cleaned > 0:
            print(f"🧹 Cleaned {cleaned} expired sessions")


# Démarrer le worker
cleanup_worker = SessionCleanupWorker(redis_client, interval=300)
cleanup_worker.start()
```

### Session Analytics

```python
class SessionAnalytics:
    """
    Analytics et monitoring des sessions
    """

    def __init__(self, redis_client, prefix='session:'):
        self.redis = redis_client
        self.prefix = prefix

    def get_stats(self):
        """Récupère les statistiques de sessions"""
        pattern = f"{self.prefix}*"
        keys = list(self.redis.scan_iter(match=pattern))

        total_sessions = len(keys)
        active_users = set()
        sessions_by_role = {}
        oldest_session = None
        newest_session = None

        for key in keys:
            session_data = self.redis.hgetall(key)

            # User tracking
            user_id = session_data.get('user_id')
            if user_id:
                active_users.add(user_id)

            # Role distribution
            role = session_data.get('role', 'unknown')
            sessions_by_role[role] = sessions_by_role.get(role, 0) + 1

            # Session age
            created_at = float(session_data.get('created_at', 0))
            if not oldest_session or created_at < oldest_session:
                oldest_session = created_at
            if not newest_session or created_at > newest_session:
                newest_session = created_at

        return {
            'total_sessions': total_sessions,
            'unique_users': len(active_users),
            'sessions_by_role': sessions_by_role,
            'oldest_session_age': time.time() - oldest_session if oldest_session else 0,
            'newest_session_age': time.time() - newest_session if newest_session else 0
        }

    def get_user_activity(self, user_id):
        """Récupère l'activité d'un utilisateur"""
        sessions = []
        pattern = f"{self.prefix}*"

        for key in self.redis.scan_iter(match=pattern):
            session_data = self.redis.hgetall(key)

            if session_data.get('user_id') == str(user_id):
                ttl = self.redis.ttl(key)

                sessions.append({
                    'session_id': key.decode('utf-8').replace(self.prefix, ''),
                    'created_at': session_data.get('created_at'),
                    'last_activity': session_data.get('last_activity'),
                    'ip_address': session_data.get('ip_address'),
                    'user_agent': session_data.get('user_agent'),
                    'ttl_remaining': ttl
                })

        return sessions


# Route pour les analytics
@app.route('/admin/analytics')
def session_analytics():
    """Endpoint d'analytics"""
    analytics = SessionAnalytics(redis_client)
    stats = analytics.get_stats()

    return jsonify(stats)
```

---

## Bonnes pratiques

### Checklist de production

**Sécurité :**
- ✅ Cookies avec `httponly`, `secure`, `samesite`
- ✅ Session ID aléatoire et imprévisible (UUID v4)
- ✅ Validation du fingerprint (IP + User-Agent)
- ✅ Protection CSRF
- ✅ TTL approprié (30 min standard)
- ✅ Rotation de session après login

**Performance :**
- ✅ Utiliser HASH pour les sessions (plus rapide que String)
- ✅ TTL sliding pour les sessions actives
- ✅ Cleanup automatique des sessions expirées
- ✅ Monitoring du nombre de sessions actives
- ✅ Limitation du nombre de sessions par utilisateur

**Scalabilité :**
- ✅ Redis avec persistence (RDB + AOF)
- ✅ Redis Cluster pour haute disponibilité
- ✅ Session replication optionnelle
- ✅ Load balancer sans sticky sessions

**Monitoring :**
- ✅ Nombre de sessions actives
- ✅ Taux de création/suppression
- ✅ Durée moyenne des sessions
- ✅ Alertes si pic anormal

### Anti-patterns à éviter

```python
# ❌ BAD: Stocker des données sensibles en clair
session_data = {
    'password': 'user_password_123',  # NEVER!
    'credit_card': '1234-5678-9012'   # NEVER!
}

# ✅ GOOD: Stocker seulement les IDs et métadonnées
session_data = {
    'user_id': '12345',
    'role': 'user'
}


# ❌ BAD: Session ID prévisible
session_id = f"user_{user_id}_{timestamp}"

# ✅ GOOD: UUID aléatoire
session_id = str(uuid.uuid4())


# ❌ BAD: TTL trop long
ttl = 86400 * 7  # 7 jours

# ✅ GOOD: TTL raisonnable avec sliding
ttl = 1800  # 30 minutes avec auto-prolongation


# ❌ BAD: Pas de nettoyage
# Les sessions expirent mais les clés restent

# ✅ GOOD: Cleanup automatique
cleanup_worker = SessionCleanupWorker(redis_client)
cleanup_worker.start()
```

---

## Conclusion

La gestion des sessions avec Redis offre le meilleur compromis entre performance, scalabilité et simplicité. Les points clés :

**Avantages Redis pour les sessions :**
- ⚡ Performance excellente (in-memory)
- 🔄 TTL automatique (pas de cleanup manuel)
- 📊 Scalabilité horizontale
- 🌐 Partage entre instances
- 💾 Persistence optionnelle

**Patterns recommandés :**
1. **Basic Session Store** : Simple et efficace (80% des cas)
2. **Sliding TTL** : Auto-prolongation pour sessions actives
3. **Secure Sessions** : Fingerprinting + CSRF protection
4. **Shopping Cart** : Données complexes avec JSON

**Best Practices :**
- Toujours sécuriser les cookies (httponly, secure, samesite)
- Utiliser UUID v4 pour session ID
- TTL de 30 minutes avec sliding
- Monitoring obligatoire
- Cleanup automatique
- Protection contre le hijacking

Redis est devenu le standard pour la gestion des sessions, et pour de bonnes raisons : c'est rapide, fiable, et facile à utiliser !

---


⏭️ [Files d'attente et Job Queues](/06-patterns-developpement-avances/08-files-attente-job-queues.md)
