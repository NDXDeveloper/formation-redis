🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.4 - Chiffrement : TLS/SSL et impact sur la performance

## Introduction

Le chiffrement TLS (Transport Layer Security) pour Redis est devenu **indispensable** en production moderne pour protéger les données en transit. Sans TLS, toutes les communications Redis sont en clair sur le réseau, exposant :

- 🔓 **Données sensibles** (sessions, tokens, informations utilisateurs)
- 🔓 **Mots de passe** d'authentification
- 🔓 **Commandes** exécutées
- 🔓 **Résultats** de requêtes

> **⚠️ Attention :** TLS introduit un overhead de performance (latence +10-30%, débit -15-40%). Ce document vous guide pour activer TLS tout en minimisant l'impact.

---

## TLS dans Redis : Support et versions

### Historique du support TLS

```
Redis 6.0 (Mai 2020)
├── Introduction du support TLS natif
├── TLS pour connexions client-serveur
├── TLS pour réplication master-replica
└── TLS pour cluster

Redis 6.2 (Février 2021)
├── Amélioration performance TLS
├── TLS session caching
└── Mutual TLS (mTLS) amélioré

Redis 7.0 (Avril 2022)
├── Optimisations TLS supplémentaires
├── Support TLS 1.3
└── Meilleure gestion certificats

Redis 7.2 (2023)
├── Performance TLS optimale
└── TLS par défaut dans Redis Stack
```

### Prérequis

```bash
# Vérifier que Redis a été compilé avec support TLS
redis-server --version
# Output doit inclure: "with OpenSSL"

# Exemple:
# Redis server v=7.2.0 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=a1b2c3d4e5f6g7h8 with OpenSSL

# Si pas de support TLS, recompiler Redis avec:
make BUILD_TLS=yes
```

---

## Architecture TLS pour Redis

### 1. TLS Simple (Client → Redis)

```
┌──────────────┐                    ┌──────────────┐
│   Client     │                    │    Redis     │
│              │                    │   Server     │
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       │ 1. ClientHello (TLS handshake)    │
       ├──────────────────────────────────>│
       │                                   │
       │ 2. ServerHello + Certificate      │
       │<──────────────────────────────────┤
       │                                   │
       │ 3. Validate certificate           │
       │    (check CA, hostname, expiry)   │
       │                                   │
       │ 4. Encrypted session established  │
       ├──────────────────────────────────>│
       │                                   │
       │ 5. All commands encrypted         │
       │    (AES-256-GCM, ChaCha20, etc.)  │
       │<─────────────────────────────────>│
```

### 2. Mutual TLS (mTLS) - Client ↔ Redis

```
┌──────────────┐                    ┌──────────────┐
│   Client     │                    │    Redis     │
│              │                    │   Server     │
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       │ 1. ClientHello                    │
       ├──────────────────────────────────>│
       │                                   │
       │ 2. ServerHello + Server Cert      │
       │<──────────────────────────────────┤
       │                                   │
       │ 3. Client Certificate Request     │
       │<──────────────────────────────────┤
       │                                   │
       │ 4. Client sends its certificate   │
       ├──────────────────────────────────>│
       │                                   │
       │ 5. Server validates client cert   │
       │    (check CA, CN, expiry)         │
       │                                   │
       │ 6. Encrypted + Authenticated      │
       │<─────────────────────────────────>│
```

### 3. TLS pour réplication

```
┌──────────────┐  TLS  ┌──────────────┐  TLS  ┌──────────────┐
│    Master    │◄─────►│  Replica 1   │◄─────►│  Replica 2   │
│              │       │              │       │  (chained)   │
└──────────────┘       └──────────────┘       └──────────────┘
       │                                              │
       │  TLS                                    TLS  │
       │                                              │
       ▼                                              ▼
┌──────────────┐                            ┌──────────────┐
│   Sentinel   │                            │   Sentinel   │
└──────────────┘                            └──────────────┘
```

---

## Génération des certificats

### 1. Certificat auto-signé (dev/staging)

```bash
#!/bin/bash
# generate-redis-certs-self-signed.sh

# Répertoire pour certificats
CERT_DIR="/etc/redis/certs"
mkdir -p $CERT_DIR
cd $CERT_DIR

# 1. Créer Certificate Authority (CA) privée
echo "=== Génération CA ==="
openssl genrsa -out ca-key.pem 4096

openssl req -new -x509 -days 3650 -key ca-key.pem -out ca-cert.pem \
    -subj "/C=FR/ST=IDF/L=Paris/O=MyCompany/OU=IT/CN=Redis-CA"

# 2. Créer clé privée du serveur Redis
echo ""
echo "=== Génération clé serveur ==="
openssl genrsa -out redis-key.pem 4096

# 3. Créer Certificate Signing Request (CSR)
echo ""
echo "=== Génération CSR ==="
openssl req -new -key redis-key.pem -out redis.csr \
    -subj "/C=FR/ST=IDF/L=Paris/O=MyCompany/OU=IT/CN=redis.example.com"

# 4. Créer fichier de configuration SAN (Subject Alternative Names)
cat > redis-san.cnf << EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = FR
ST = IDF
L = Paris
O = MyCompany
OU = IT
CN = redis.example.com

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = redis.example.com
DNS.2 = redis-master.example.com
DNS.3 = redis-replica1.example.com
DNS.4 = redis-replica2.example.com
DNS.5 = localhost
IP.1 = 10.0.1.50
IP.2 = 10.0.1.51
IP.3 = 10.0.1.52
IP.4 = 127.0.0.1
EOF

# 5. Signer le certificat avec notre CA
echo ""
echo "=== Signature certificat serveur ==="
openssl x509 -req -days 365 -in redis.csr \
    -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
    -out redis-cert.pem \
    -extensions v3_req -extfile redis-san.cnf

# 6. Créer certificat client (pour mutual TLS)
echo ""
echo "=== Génération certificat client ==="
openssl genrsa -out redis-client-key.pem 4096

openssl req -new -key redis-client-key.pem -out redis-client.csr \
    -subj "/C=FR/ST=IDF/L=Paris/O=MyCompany/OU=IT/CN=redis-client"

openssl x509 -req -days 365 -in redis-client.csr \
    -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
    -out redis-client-cert.pem

# 7. Vérifier les certificats
echo ""
echo "=== Vérification ==="
openssl x509 -in redis-cert.pem -text -noout | grep -A 1 "Subject:"
openssl x509 -in redis-cert.pem -text -noout | grep -A 3 "Subject Alternative Name"
openssl verify -CAfile ca-cert.pem redis-cert.pem
openssl verify -CAfile ca-cert.pem redis-client-cert.pem

# 8. Permissions
echo ""
echo "=== Configuration permissions ==="
chmod 400 redis-key.pem redis-client-key.pem ca-key.pem
chmod 444 redis-cert.pem redis-client-cert.pem ca-cert.pem
chown redis:redis redis-key.pem redis-cert.pem ca-cert.pem

echo ""
echo "✅ Certificats générés dans: $CERT_DIR"
echo ""
echo "Fichiers créés:"
ls -lh $CERT_DIR

echo ""
echo "Configuration redis.conf:"
echo "  tls-cert-file $CERT_DIR/redis-cert.pem"
echo "  tls-key-file $CERT_DIR/redis-key.pem"
echo "  tls-ca-cert-file $CERT_DIR/ca-cert.pem"
```

### 2. Certificat Let's Encrypt (production avec domaine public)

```bash
#!/bin/bash
# setup-letsencrypt-redis.sh

# Prérequis: certbot installé
# apt install certbot

DOMAIN="redis.example.com"
EMAIL="admin@example.com"
CERT_DIR="/etc/redis/certs"

# 1. Obtenir certificat Let's Encrypt
certbot certonly --standalone \
    -d $DOMAIN \
    -m $EMAIL \
    --agree-tos \
    --non-interactive

# 2. Copier certificats pour Redis
mkdir -p $CERT_DIR

cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem $CERT_DIR/redis-cert.pem
cp /etc/letsencrypt/live/$DOMAIN/privkey.pem $CERT_DIR/redis-key.pem
cp /etc/letsencrypt/live/$DOMAIN/chain.pem $CERT_DIR/ca-cert.pem

# 3. Permissions
chown redis:redis $CERT_DIR/*
chmod 400 $CERT_DIR/redis-key.pem
chmod 444 $CERT_DIR/redis-cert.pem $CERT_DIR/ca-cert.pem

# 4. Hook de renouvellement automatique
cat > /etc/letsencrypt/renewal-hooks/deploy/redis-reload.sh << 'EOF'
#!/bin/bash
# Hook exécuté après renouvellement Let's Encrypt

DOMAIN="redis.example.com"
CERT_DIR="/etc/redis/certs"

# Copier nouveaux certificats
cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem $CERT_DIR/redis-cert.pem
cp /etc/letsencrypt/live/$DOMAIN/privkey.pem $CERT_DIR/redis-key.pem
cp /etc/letsencrypt/live/$DOMAIN/chain.pem $CERT_DIR/ca-cert.pem

# Permissions
chown redis:redis $CERT_DIR/*
chmod 400 $CERT_DIR/redis-key.pem
chmod 444 $CERT_DIR/redis-cert.pem $CERT_DIR/ca-cert.pem

# Reload Redis (pas de downtime avec TLS)
redis-cli CONFIG SET tls-cert-file $CERT_DIR/redis-cert.pem
redis-cli CONFIG SET tls-key-file $CERT_DIR/redis-key.pem

echo "$(date): Redis TLS certificates renewed" >> /var/log/redis/cert-renewal.log
EOF

chmod +x /etc/letsencrypt/renewal-hooks/deploy/redis-reload.sh

echo "✅ Let's Encrypt configuré"
echo "Renouvellement automatique: certbot renew (cron)"
```

### 3. Certificat d'entreprise (PKI interne)

```bash
# Utiliser votre PKI d'entreprise
# Exemple avec HashiCorp Vault

# 1. Générer certificat via Vault
vault write pki/issue/redis-role \
    common_name="redis.example.com" \
    alt_names="redis-master.example.com,redis-replica1.example.com" \
    ip_sans="10.0.1.50,10.0.1.51" \
    ttl="8760h"

# 2. Récupérer et sauvegarder
vault read -field=certificate pki/issue/redis-role > /etc/redis/certs/redis-cert.pem
vault read -field=private_key pki/issue/redis-role > /etc/redis/certs/redis-key.pem
vault read -field=ca_chain pki/issue/redis-role > /etc/redis/certs/ca-cert.pem
```

---

## Configuration Redis avec TLS

### 1. Configuration TLS simple (Client → Redis)

```conf
# ============================================================================
# REDIS TLS CONFIGURATION - SIMPLE (Server-side TLS)
# ============================================================================
# Use case: Chiffrement des communications client-serveur
# Auth: Certificat serveur uniquement (clients n'ont pas besoin de certificat)
# ============================================================================

# ----------------------------------------------------------------------------
# RÉSEAU - TLS
# ----------------------------------------------------------------------------

# Activer TLS sur port 6380 (convention, ou utiliser 6379)
tls-port 6380

# Désactiver port non-TLS (SÉCURITÉ CRITIQUE)
port 0

# Bind sur interface
bind 10.0.1.50

# Protected mode
protected-mode yes

# ----------------------------------------------------------------------------
# CERTIFICATS
# ----------------------------------------------------------------------------

# Certificat du serveur Redis
tls-cert-file /etc/redis/certs/redis-cert.pem

# Clé privée du serveur Redis
tls-key-file /etc/redis/certs/redis-key.pem

# Certificat de la CA (pour valider clients si mTLS)
tls-ca-cert-file /etc/redis/certs/ca-cert.pem

# Chemin vers répertoire de CAs supplémentaires (optionnel)
# tls-ca-cert-dir /etc/redis/certs/ca/

# ----------------------------------------------------------------------------
# VALIDATION CLIENT
# ----------------------------------------------------------------------------

# Mode d'authentification TLS des clients
# - "no": Pas de validation certificat client (TLS simple)
# - "optional": Certificat client optionnel
# - "yes": Certificat client OBLIGATOIRE (mutual TLS)
tls-auth-clients no

# ----------------------------------------------------------------------------
# PROTOCOLES ET CIPHERS
# ----------------------------------------------------------------------------

# Protocoles TLS autorisés (TLS 1.2 minimum recommandé)
tls-protocols "TLSv1.2 TLSv1.3"

# Ciphers autorisés (strong ciphers uniquement)
tls-ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256

# Préférer ciphers du serveur
tls-prefer-server-ciphers yes

# Courbes elliptiques (pour ECDHE)
tls-ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256

# ----------------------------------------------------------------------------
# OPTIMISATIONS TLS
# ----------------------------------------------------------------------------

# Session caching (réutilisation des sessions TLS)
# Réduit overhead du handshake TLS
tls-session-caching yes

# Taille du cache de sessions TLS
tls-session-cache-size 20480

# Timeout du cache de sessions (secondes)
tls-session-cache-timeout 300

# ----------------------------------------------------------------------------
# DH PARAMS (pour DHE ciphers, optionnel)
# ----------------------------------------------------------------------------

# Paramètres Diffie-Hellman pour forward secrecy
# Générer avec: openssl dhparam -out /etc/redis/certs/dh2048.pem 2048
# tls-dh-params-file /etc/redis/certs/dh2048.pem

# ============================================================================
```

### 2. Configuration TLS + Mutual TLS (mTLS)

```conf
# ============================================================================
# REDIS TLS CONFIGURATION - MUTUAL TLS (mTLS)
# ============================================================================
# Use case: Chiffrement + Authentification forte des clients
# Auth: Serveur ET clients doivent avoir des certificats
# Sécurité maximale
# ============================================================================

# ----------------------------------------------------------------------------
# RÉSEAU - TLS
# ----------------------------------------------------------------------------

tls-port 6380
port 0
bind 10.0.1.50
protected-mode yes

# ----------------------------------------------------------------------------
# CERTIFICATS
# ----------------------------------------------------------------------------

tls-cert-file /etc/redis/certs/redis-cert.pem
tls-key-file /etc/redis/certs/redis-key.pem
tls-ca-cert-file /etc/redis/certs/ca-cert.pem

# ----------------------------------------------------------------------------
# VALIDATION CLIENT - MUTUAL TLS
# ----------------------------------------------------------------------------

# OBLIGATOIRE: Les clients doivent présenter un certificat valide
tls-auth-clients yes

# Replication: Certificat pour réplication (replica vers master)
tls-replication yes

# Cluster: TLS pour gossip protocol
tls-cluster yes

# ----------------------------------------------------------------------------
# PROTOCOLES ET CIPHERS
# ----------------------------------------------------------------------------

tls-protocols "TLSv1.2 TLSv1.3"

# Ciphers avec forward secrecy (ECDHE, DHE)
tls-ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384

tls-prefer-server-ciphers yes

# ----------------------------------------------------------------------------
# OPTIMISATIONS
# ----------------------------------------------------------------------------

tls-session-caching yes
tls-session-cache-size 20480
tls-session-cache-timeout 300

# ============================================================================
# CONFIGURATION ACL (combiné avec mTLS)
# ============================================================================

# ACLs + mTLS = Défense en profondeur
aclfile /etc/redis/users.acl

# ============================================================================
```

### 3. Configuration réplication avec TLS

```conf
# ============================================================================
# REDIS MASTER - TLS + REPLICATION
# ============================================================================

# TLS configuration
tls-port 6380
port 0
tls-cert-file /etc/redis/certs/master-cert.pem
tls-key-file /etc/redis/certs/master-key.pem
tls-ca-cert-file /etc/redis/certs/ca-cert.pem
tls-auth-clients yes
tls-replication yes

# Authentification
aclfile /etc/redis/users.acl

# Réplication
min-replicas-to-write 1
min-replicas-max-lag 10

# ============================================================================
```

```conf
# ============================================================================
# REDIS REPLICA - TLS + REPLICATION
# ============================================================================

# TLS configuration
tls-port 6380
port 0
tls-cert-file /etc/redis/certs/replica1-cert.pem
tls-key-file /etc/redis/certs/replica1-key.pem
tls-ca-cert-file /etc/redis/certs/ca-cert.pem
tls-auth-clients yes
tls-replication yes

# Connexion au master via TLS
replicaof master.example.com 6380

# Authentification master
masteruser replicator
masterauth ReplicatorP@ss2024!

# Certificat client pour connexion master
tls-client-cert-file /etc/redis/certs/replica1-cert.pem
tls-client-key-file /etc/redis/certs/replica1-key.pem

# ============================================================================
```

---

## Configuration clients avec TLS

### 1. redis-cli avec TLS

```bash
# ============================================================================
# REDIS-CLI avec TLS
# ============================================================================

# TLS simple (sans certificat client)
redis-cli \
    --tls \
    --cert /etc/redis/certs/redis-cert.pem \
    --key /etc/redis/certs/redis-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    -h redis.example.com \
    -p 6380 \
    -a YourPassword \
    PING

# Mutual TLS (avec certificat client)
redis-cli \
    --tls \
    --cert /etc/redis/certs/redis-client-cert.pem \
    --key /etc/redis/certs/redis-client-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    -h redis.example.com \
    -p 6380 \
    -a YourPassword \
    PING

# Créer alias pour faciliter usage
alias redis-cli-tls='redis-cli --tls --cert /etc/redis/certs/redis-client-cert.pem --key /etc/redis/certs/redis-client-key.pem --cacert /etc/redis/certs/ca-cert.pem'

# Utilisation
redis-cli-tls -h redis.example.com -p 6380 -a YourPassword INFO
```

### 2. Python (redis-py)

```python
#!/usr/bin/env python3
# redis-tls-client.py

import redis
import ssl

# TLS simple
r = redis.Redis(
    host='redis.example.com',
    port=6380,
    password='YourPassword',
    ssl=True,
    ssl_cert_reqs=ssl.CERT_REQUIRED,
    ssl_ca_certs='/etc/redis/certs/ca-cert.pem'
)

# Test
r.ping()
print("✅ Connexion TLS réussie")

# Mutual TLS (avec certificat client)
r_mtls = redis.Redis(
    host='redis.example.com',
    port=6380,
    password='YourPassword',
    ssl=True,
    ssl_cert_reqs=ssl.CERT_REQUIRED,
    ssl_ca_certs='/etc/redis/certs/ca-cert.pem',
    ssl_certfile='/etc/redis/certs/redis-client-cert.pem',
    ssl_keyfile='/etc/redis/certs/redis-client-key.pem'
)

r_mtls.ping()
print("✅ Connexion mTLS réussie")

# Configuration avancée
ssl_context = ssl.create_default_context(cafile='/etc/redis/certs/ca-cert.pem')
ssl_context.check_hostname = True
ssl_context.verify_mode = ssl.CERT_REQUIRED

# TLS 1.3 uniquement
ssl_context.minimum_version = ssl.TLSVersion.TLSv1_3

# Charger certificat client
ssl_context.load_cert_chain(
    certfile='/etc/redis/certs/redis-client-cert.pem',
    keyfile='/etc/redis/certs/redis-client-key.pem'
)

r_custom = redis.Redis(
    host='redis.example.com',
    port=6380,
    password='YourPassword',
    ssl=True,
    ssl_context=ssl_context
)
```

### 3. Node.js (ioredis)

```javascript
// redis-tls-client.js

const Redis = require('ioredis');
const fs = require('fs');

// TLS simple
const redisSimple = new Redis({
  host: 'redis.example.com',
  port: 6380,
  password: 'YourPassword',
  tls: {
    ca: fs.readFileSync('/etc/redis/certs/ca-cert.pem'),
    rejectUnauthorized: true,
    checkServerIdentity: (hostname, cert) => {
      // Validation custom si nécessaire
      return undefined; // undefined = OK
    }
  }
});

// Mutual TLS
const redisMTLS = new Redis({
  host: 'redis.example.com',
  port: 6380,
  password: 'YourPassword',
  tls: {
    ca: fs.readFileSync('/etc/redis/certs/ca-cert.pem'),
    cert: fs.readFileSync('/etc/redis/certs/redis-client-cert.pem'),
    key: fs.readFileSync('/etc/redis/certs/redis-client-key.pem'),
    rejectUnauthorized: true
  }
});

// Test
redisSimple.ping((err, result) => {
  if (err) {
    console.error('❌ Erreur:', err);
  } else {
    console.log('✅ TLS connexion réussie:', result);
  }
});

redisMTLS.ping((err, result) => {
  if (err) {
    console.error('❌ Erreur:', err);
  } else {
    console.log('✅ mTLS connexion réussie:', result);
  }
});
```

### 4. Java (Jedis)

```java
// RedisTLSClient.java

import redis.clients.jedis.Jedis;
import redis.clients.jedis.DefaultJedisClientConfig;
import redis.clients.jedis.HostAndPort;

import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManagerFactory;
import javax.net.ssl.KeyManagerFactory;
import java.io.FileInputStream;
import java.security.KeyStore;

public class RedisTLSClient {

    public static void main(String[] args) throws Exception {
        // Configuration TLS
        SSLContext sslContext = createSSLContext();

        // Configuration Jedis avec TLS
        DefaultJedisClientConfig config = DefaultJedisClientConfig.builder()
            .ssl(true)
            .sslContext(sslContext)
            .password("YourPassword")
            .build();

        HostAndPort hostAndPort = new HostAndPort("redis.example.com", 6380);

        try (Jedis jedis = new Jedis(hostAndPort, config)) {
            String result = jedis.ping();
            System.out.println("✅ TLS connexion réussie: " + result);
        }
    }

    private static SSLContext createSSLContext() throws Exception {
        // Charger CA certificate
        KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
        trustStore.load(null, null);

        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm()
        );
        tmf.init(trustStore);

        // Pour mutual TLS, charger aussi certificat client
        // KeyStore keyStore = KeyStore.getInstance("PKCS12");
        // keyStore.load(new FileInputStream("client.p12"), "password".toCharArray());

        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, tmf.getTrustManagers(), null);

        return sslContext;
    }
}
```

---

## Impact sur les performances

### 1. Overhead TLS : Mesures réelles

```bash
#!/bin/bash
# benchmark-tls-overhead.sh

echo "=== REDIS TLS PERFORMANCE BENCHMARK ==="
echo ""

# 1. Baseline - Sans TLS
echo "1. Benchmark SANS TLS (baseline):"
redis-benchmark -h localhost -p 6379 -t set,get -n 1000000 -q -c 50 -d 100

echo ""
echo "2. Benchmark AVEC TLS:"
redis-benchmark -h localhost -p 6380 --tls \
    --cert /etc/redis/certs/redis-client-cert.pem \
    --key /etc/redis/certs/redis-client-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    -t set,get -n 1000000 -q -c 50 -d 100

echo ""
echo "3. Benchmark TLS avec pipeline (optimisation):"
redis-benchmark -h localhost -p 6380 --tls \
    --cert /etc/redis/certs/redis-client-cert.pem \
    --key /etc/redis/certs/redis-client-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    -t set,get -n 1000000 -q -c 50 -d 100 -P 16

echo ""
echo "4. Latency comparison:"
echo "Sans TLS:"
redis-cli -h localhost -p 6379 --latency -i 1 | head -5

echo ""
echo "Avec TLS:"
redis-cli -h localhost -p 6380 --tls \
    --cert /etc/redis/certs/redis-client-cert.pem \
    --key /etc/redis/certs/redis-client-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    --latency -i 1 | head -5
```

### 2. Résultats typiques

```
┌─────────────────────────────────────────────────────────────────┐
│              IMPACT TLS SUR PERFORMANCES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Métrique             │  Sans TLS  │  TLS 1.2   │  TLS 1.3      │
│  ────────────────────────────────────────────────────────────   │
│  Latency (p50)        │  0.3ms     │  0.4ms     │  0.35ms       │
│  Latency (p99)        │  0.8ms     │  1.2ms     │  1.0ms        │
│  Throughput SET       │  85K ops/s │  60K ops/s │  70K ops/s    │
│  Throughput GET       │  95K ops/s │  65K ops/s │  75K ops/s    │
│  CPU usage            │  15%       │  30%       │  25%          │
│  ────────────────────────────────────────────────────────────   │
│  Impact global        │  Baseline  │  -25-30%   │  -15-20%      │
│                                                                 │
│  Avec pipeline (P=16):                                          │
│  Throughput SET       │  180K ops/s│  150K ops/s│  165K ops/s   │
│  Throughput GET       │  200K ops/s│  170K ops/s│  185K ops/s   │
│  Impact avec pipeline │  Baseline  │  -15-20%   │  -10-15%      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Analyse détaillée de l'overhead

```python
#!/usr/bin/env python3
# analyze-tls-overhead.py

import redis
import ssl
import time
import statistics

# Configuration
HOST = 'redis.example.com'
PORT_NOTLS = 6379
PORT_TLS = 6380
ITERATIONS = 10000

def benchmark_redis(use_tls=False):
    """Benchmark Redis avec ou sans TLS"""

    if use_tls:
        ssl_context = ssl.create_default_context(
            cafile='/etc/redis/certs/ca-cert.pem'
        )
        ssl_context.load_cert_chain(
            certfile='/etc/redis/certs/redis-client-cert.pem',
            keyfile='/etc/redis/certs/redis-client-key.pem'
        )

        r = redis.Redis(
            host=HOST,
            port=PORT_TLS,
            ssl=True,
            ssl_context=ssl_context
        )
    else:
        r = redis.Redis(host=HOST, port=PORT_NOTLS)

    # Warmup
    for _ in range(100):
        r.set('warmup', 'value')
        r.get('warmup')

    # Benchmark SET
    set_times = []
    for i in range(ITERATIONS):
        start = time.perf_counter()
        r.set(f'key:{i}', f'value:{i}')
        elapsed = (time.perf_counter() - start) * 1000  # ms
        set_times.append(elapsed)

    # Benchmark GET
    get_times = []
    for i in range(ITERATIONS):
        start = time.perf_counter()
        r.get(f'key:{i}')
        elapsed = (time.perf_counter() - start) * 1000  # ms
        get_times.append(elapsed)

    return {
        'set': {
            'p50': statistics.median(set_times),
            'p95': statistics.quantiles(set_times, n=20)[18],
            'p99': statistics.quantiles(set_times, n=100)[98],
            'avg': statistics.mean(set_times),
            'min': min(set_times),
            'max': max(set_times)
        },
        'get': {
            'p50': statistics.median(get_times),
            'p95': statistics.quantiles(get_times, n=20)[18],
            'p99': statistics.quantiles(get_times, n=100)[98],
            'avg': statistics.mean(get_times),
            'min': min(get_times),
            'max': max(get_times)
        }
    }

if __name__ == '__main__':
    print("=== REDIS TLS OVERHEAD ANALYSIS ===")
    print(f"Iterations: {ITERATIONS}")
    print("")

    print("1. Benchmark WITHOUT TLS...")
    results_notls = benchmark_redis(use_tls=False)

    print("2. Benchmark WITH TLS...")
    results_tls = benchmark_redis(use_tls=True)

    # Afficher résultats
    print("\n=== RESULTS ===\n")

    print("SET Command:")
    print(f"  Without TLS - p50: {results_notls['set']['p50']:.3f}ms, "
          f"p99: {results_notls['set']['p99']:.3f}ms")
    print(f"  With TLS    - p50: {results_tls['set']['p50']:.3f}ms, "
          f"p99: {results_tls['set']['p99']:.3f}ms")

    overhead_set = ((results_tls['set']['p50'] - results_notls['set']['p50'])
                    / results_notls['set']['p50'] * 100)
    print(f"  Overhead: +{overhead_set:.1f}%")

    print("\nGET Command:")
    print(f"  Without TLS - p50: {results_notls['get']['p50']:.3f}ms, "
          f"p99: {results_notls['get']['p99']:.3f}ms")
    print(f"  With TLS    - p50: {results_tls['get']['p50']:.3f}ms, "
          f"p99: {results_tls['get']['p99']:.3f}ms")

    overhead_get = ((results_tls['get']['p50'] - results_notls['get']['p50'])
                    / results_notls['get']['p50'] * 100)
    print(f"  Overhead: +{overhead_get:.1f}%")
```

---

## Optimisations TLS

### 1. TLS Session Resumption

```conf
# redis.conf - Optimisations TLS

# Activer session caching (CRITIQUE pour performance)
tls-session-caching yes

# Taille du cache (augmenter si beaucoup de clients)
tls-session-cache-size 20480  # Default: 20480

# Timeout des sessions en cache (secondes)
tls-session-cache-timeout 300  # 5 minutes

# Cette optimisation évite handshake TLS complet à chaque reconnexion
# Gain: -70% latence sur reconnexions
```

### 2. TLS 1.3 vs TLS 1.2

```
┌─────────────────────────────────────────────────────────────┐
│           TLS 1.2 vs TLS 1.3 COMPARISON                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Aspect              │  TLS 1.2     │  TLS 1.3              │
│  ────────────────────────────────────────────────────────── │
│  Handshake RTT       │  2 RTT       │  1 RTT (-50%)         │
│  0-RTT resumption    │  No          │  Yes (optionnel)      │
│  Cipher suites       │  37          │  5 (forte seulement)  │
│  Forward secrecy     │  Optionnel   │  Obligatoire          │
│  CPU overhead        │  +30%        │  +20%                 │
│  ────────────────────────────────────────────────────────── │
│  Recommendation      │  Minimum     │  Préféré              │
│                                                             │
└─────────────────────────────────────────────────────────────┘

# Configuration pour TLS 1.3 uniquement
tls-protocols "TLSv1.3"

# Ciphers TLS 1.3 (tous sont forts)
tls-ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
```

### 3. Connection pooling côté client

```python
#!/usr/bin/env python3
# redis-tls-pool-optimized.py

import redis
import ssl

# MAUVAIS: Nouvelle connexion à chaque requête
def bad_example():
    r = redis.Redis(
        host='redis.example.com',
        port=6380,
        ssl=True,
        ssl_ca_certs='/etc/redis/certs/ca-cert.pem'
    )
    r.set('key', 'value')  # Full TLS handshake à chaque fois!
    # r est détruit après chaque appel

# BON: Pool de connexions réutilisées
def good_example():
    # Créer pool une fois (au démarrage app)
    pool = redis.ConnectionPool(
        host='redis.example.com',
        port=6380,
        max_connections=50,
        ssl=True,
        ssl_cert_reqs=ssl.CERT_REQUIRED,
        ssl_ca_certs='/etc/redis/certs/ca-cert.pem',
        ssl_certfile='/etc/redis/certs/redis-client-cert.pem',
        ssl_keyfile='/etc/redis/certs/redis-client-key.pem',
        # Optimisations
        socket_keepalive=True,
        socket_keepalive_options={
            socket.TCP_KEEPIDLE: 60,
            socket.TCP_KEEPINTVL: 10,
            socket.TCP_KEEPCNT: 3
        },
        retry_on_timeout=True,
        health_check_interval=30
    )

    # Réutiliser connexions du pool
    r = redis.Redis(connection_pool=pool)

    # Multiples requêtes réutilisent connexions
    r.set('key1', 'value1')  # Peut réutiliser connexion existante
    r.set('key2', 'value2')  # Session TLS déjà établie!

    return pool

# MEILLEUR: Singleton global
_redis_pool = None

def get_redis():
    """Get Redis connection from global pool"""
    global _redis_pool

    if _redis_pool is None:
        ssl_context = ssl.create_default_context(
            cafile='/etc/redis/certs/ca-cert.pem'
        )
        ssl_context.minimum_version = ssl.TLSVersion.TLSv1_3
        ssl_context.load_cert_chain(
            certfile='/etc/redis/certs/redis-client-cert.pem',
            keyfile='/etc/redis/certs/redis-client-key.pem'
        )

        _redis_pool = redis.ConnectionPool(
            host='redis.example.com',
            port=6380,
            max_connections=100,
            ssl=True,
            ssl_context=ssl_context,
            socket_keepalive=True,
            health_check_interval=30
        )

    return redis.Redis(connection_pool=_redis_pool)

# Usage dans votre application
r = get_redis()
r.set('key', 'value')  # Connexions réutilisées = Excellent perf!
```

### 4. Pipelining avec TLS

```python
#!/usr/bin/env python3
# redis-tls-pipeline.py

import redis
import ssl
import time

ssl_context = ssl.create_default_context(cafile='/etc/redis/certs/ca-cert.pem')

r = redis.Redis(
    host='redis.example.com',
    port=6380,
    ssl=True,
    ssl_context=ssl_context
)

# LENT: Requêtes individuelles
start = time.time()
for i in range(1000):
    r.set(f'key:{i}', f'value:{i}')
slow_time = time.time() - start
print(f"Sans pipeline: {slow_time:.2f}s")

# RAPIDE: Pipeline (batch)
start = time.time()
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
fast_time = time.time() - start
print(f"Avec pipeline: {fast_time:.2f}s")

print(f"Speedup: {slow_time / fast_time:.1f}x")

# Résultats typiques:
# Sans pipeline: 2.50s  (1000 handshakes TLS overhead)
# Avec pipeline: 0.15s  (1 seul exchange)
# Speedup: 16.7x
```

---

## Monitoring TLS

### 1. Métriques à surveiller

```bash
#!/bin/bash
# monitor-redis-tls.sh

echo "=== REDIS TLS MONITORING ==="

# 1. Vérifier que TLS est actif
echo "1. TLS Status:"
redis-cli --tls \
    --cert /etc/redis/certs/redis-client-cert.pem \
    --key /etc/redis/certs/redis-client-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    -h redis.example.com -p 6380 \
    CONFIG GET tls-port

# 2. Connexions TLS actives
echo ""
echo "2. Active TLS connections:"
redis-cli --tls \
    --cert /etc/redis/certs/redis-client-cert.pem \
    --key /etc/redis/certs/redis-client-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    -h redis.example.com -p 6380 \
    INFO clients | grep connected_clients

# 3. Latence avec TLS
echo ""
echo "3. Latency with TLS:"
redis-cli --tls \
    --cert /etc/redis/certs/redis-client-cert.pem \
    --key /etc/redis/certs/redis-client-key.pem \
    --cacert /etc/redis/certs/ca-cert.pem \
    -h redis.example.com -p 6380 \
    --latency-history | head -10

# 4. Expiration certificat
echo ""
echo "4. Certificate expiration:"
openssl x509 -in /etc/redis/certs/redis-cert.pem -noout -enddate

# 5. TLS errors dans logs
echo ""
echo "5. Recent TLS errors:"
tail -100 /var/log/redis/redis.log | grep -i "tls\|ssl" | tail -10
```

### 2. Alertes Prometheus

```yaml
# prometheus-alerts-redis-tls.yml

groups:
  - name: redis_tls
    interval: 30s
    rules:
      # Alerte: Certificat expire bientôt
      - alert: RedisCertificateExpiringSoon
        expr: |
          (redis_cert_expiry_timestamp - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Redis TLS certificate expiring soon"
          description: "Redis certificate expires in {{ $value }} days"

      # Alerte: Certificat expiré
      - alert: RedisCertificateExpired
        expr: |
          redis_cert_expiry_timestamp < time()
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Redis TLS certificate expired"
          description: "Redis certificate has expired!"

      # Alerte: Latence élevée (potentiellement TLS overhead)
      - alert: RedisHighLatencyTLS
        expr: |
          redis_commands_duration_seconds_total{quantile="0.99"} > 0.010
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis high latency detected"
          description: "P99 latency is {{ $value }}s (may indicate TLS issues)"

      # Alerte: TLS handshake errors
      - alert: RedisTLSHandshakeErrors
        expr: |
          rate(redis_tls_handshake_errors_total[5m]) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis TLS handshake errors"
          description: "{{ $value }} TLS handshake errors/sec"
```

### 3. Script de vérification certificats

```bash
#!/bin/bash
# check-redis-certs.sh

echo "=== REDIS CERTIFICATE HEALTH CHECK ==="

CERT_DIR="/etc/redis/certs"
CERTS=(
    "redis-cert.pem"
    "redis-client-cert.pem"
    "ca-cert.pem"
)

for cert in "${CERTS[@]}"; do
    CERT_PATH="$CERT_DIR/$cert"

    echo ""
    echo "Checking: $cert"
    echo "─────────────────────────────────"

    if [ ! -f "$CERT_PATH" ]; then
        echo "❌ MISSING: File not found"
        continue
    fi

    # Vérifier validité
    if ! openssl x509 -in "$CERT_PATH" -noout -checkend 0 >/dev/null 2>&1; then
        echo "❌ EXPIRED: Certificate has expired"
        continue
    fi

    # Expiration dans 30 jours?
    if ! openssl x509 -in "$CERT_PATH" -noout -checkend 2592000 >/dev/null 2>&1; then
        echo "⚠️  WARNING: Expires in less than 30 days"
    fi

    # Date expiration
    EXPIRY=$(openssl x509 -in "$CERT_PATH" -noout -enddate | cut -d= -f2)
    echo "Expiry date: $EXPIRY"

    # Jours restants
    EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
    NOW_EPOCH=$(date +%s)
    DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))
    echo "Days remaining: $DAYS_LEFT"

    # Subject
    SUBJECT=$(openssl x509 -in "$CERT_PATH" -noout -subject | cut -d= -f2-)
    echo "Subject: $SUBJECT"

    # Issuer
    ISSUER=$(openssl x509 -in "$CERT_PATH" -noout -issuer | cut -d= -f2-)
    echo "Issuer: $ISSUER"

    # SANs
    SANS=$(openssl x509 -in "$CERT_PATH" -noout -text | grep -A1 "Subject Alternative Name" | tail -1 | sed 's/^[[:space:]]*//')
    if [ -n "$SANS" ]; then
        echo "SANs: $SANS"
    fi

    echo "✅ VALID"
done

echo ""
echo "=== END CHECK ==="
```

---

## Troubleshooting TLS

### Problèmes courants

#### 1. "SSL handshake failed"

```bash
# Diagnostic
openssl s_client -connect redis.example.com:6380 \
    -cert /etc/redis/certs/redis-client-cert.pem \
    -key /etc/redis/certs/redis-client-key.pem \
    -CAfile /etc/redis/certs/ca-cert.pem

# Causes possibles:
# - Certificat expiré
# - CA non reconnue
# - Hostname mismatch
# - Protocols/ciphers incompatibles

# Solutions:
# 1. Vérifier expiration
openssl x509 -in /etc/redis/certs/redis-cert.pem -noout -dates

# 2. Vérifier chaîne de certification
openssl verify -CAfile /etc/redis/certs/ca-cert.pem /etc/redis/certs/redis-cert.pem

# 3. Vérifier SANs
openssl x509 -in /etc/redis/certs/redis-cert.pem -noout -text | grep -A 3 "Subject Alternative Name"
```

#### 2. "Certificate verification failed"

```bash
# Problème: Hostname ne matche pas le certificat

# Solution 1: Ajouter hostname dans SANs
# Regénérer certificat avec bon hostname

# Solution 2: Désactiver validation hostname (DEV UNIQUEMENT!)
# Python:
ssl_context.check_hostname = False

# Node.js:
tls: { rejectUnauthorized: false }  # DANGEREUX!
```

#### 3. "Protocol version mismatch"

```bash
# Redis configuré pour TLS 1.3, client supporte seulement TLS 1.2

# Solution: Élargir protocoles acceptés
# redis.conf:
tls-protocols "TLSv1.2 TLSv1.3"
```

#### 4. Performance dégradée

```bash
# Causes:
# - Pas de session resumption
# - Connexions non poolées
# - Pas de pipelining
# - Ciphers lents

# Diagnostic:
redis-cli --tls --latency-history
redis-cli INFO stats | grep instantaneous_ops_per_sec

# Solutions:
# 1. Activer session caching
tls-session-caching yes

# 2. Utiliser TLS 1.3
tls-protocols "TLSv1.3"

# 3. Ciphers hardware-accelerated
tls-ciphers TLS_AES_128_GCM_SHA256  # AES-NI sur CPU moderne
```

---

## Checklist TLS pour production

### Checklist déploiement

- [ ] **Certificats valides générés**
  - Certificat serveur avec SANs corrects
  - CA certificate présente
  - Certificats clients (si mTLS)

- [ ] **Certificats non expirés**
  ```bash
  openssl x509 -in cert.pem -noout -checkend 0
  ```

- [ ] **Permissions fichiers correctes**
  ```bash
  chmod 400 *.key.pem  # Clés privées
  chmod 444 *.cert.pem # Certificats
  chown redis:redis /etc/redis/certs/*
  ```

- [ ] **TLS activé dans redis.conf**
  - `tls-port 6380`
  - `port 0` (désactiver non-TLS)
  - Certificats configurés

- [ ] **Protocoles sécurisés uniquement**
  - `tls-protocols "TLSv1.2 TLSv1.3"`
  - Pas de SSLv3, TLS 1.0, TLS 1.1

- [ ] **Ciphers forts uniquement**
  - Forward secrecy (ECDHE, DHE)
  - AES-GCM ou ChaCha20-Poly1305

- [ ] **Session caching activé**
  - `tls-session-caching yes`

- [ ] **Réplication TLS configurée**
  - `tls-replication yes` (si réplication)

- [ ] **Clients mis à jour**
  - Support TLS activé
  - Certificats déployés
  - Connection pooling configuré

- [ ] **Tests de connexion**
  ```bash
  redis-cli --tls --cacert ca-cert.pem -h redis.example.com -p 6380 PING
  ```

- [ ] **Benchmarks de performance**
  - Latence p99 acceptable
  - Throughput suffisant

### Checklist monitoring

- [ ] **Expiration certificats surveillée**
  - Alertes à 60, 30, 15, 7 jours

- [ ] **Renouvellement automatisé**
  - Cron ou service pour renouvellement
  - Hooks de reload Redis

- [ ] **TLS errors monitorés**
  - Logs Redis
  - Métriques Prometheus

- [ ] **Performance monitorée**
  - Latence avec/sans TLS
  - CPU overhead

### Checklist incident

- [ ] **Certificat expiré**
  - Renouveler immédiatement
  - Reload Redis sans downtime

- [ ] **TLS handshake errors**
  - Vérifier logs
  - Vérifier chaîne certification
  - Vérifier compatibilité protocols/ciphers

- [ ] **Performance dégradée**
  - Vérifier session caching
  - Vérifier connection pooling
  - Benchmarker TLS overhead

---

## 📚 Ressources complémentaires

### Documentation

- [Redis TLS Documentation](https://redis.io/docs/management/security/encryption/)
- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)

### Outils

- **OpenSSL** - Génération et gestion certificats
- **Let's Encrypt** - Certificats gratuits automatisés
- **certbot** - Client Let's Encrypt
- **HashiCorp Vault** - PKI d'entreprise

### Standards

- **RFC 8446** - TLS 1.3
- **RFC 5246** - TLS 1.2
- **NIST SP 800-52** - Guidelines for TLS

---

**Section suivante :** [12.5 - Protection réseau : Binding, Firewall, VPC](./05-protection-reseau-binding-firewall.md)

⏭️ [Protection réseau : Binding, Firewall, VPC](/12-redis-production-securite/05-protection-reseau-binding-firewall.md)
