🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.2 Encryption at Rest et in Transit

## Introduction

Le chiffrement est une exigence fondamentale de la sécurité des données personnelles et sensibles. Pour Redis, qui stocke des données en mémoire et sur disque, deux types de chiffrement sont essentiels :

- **Encryption in Transit (TLS/SSL)** : Protection des données pendant leur transmission sur le réseau
- **Encryption at Rest** : Protection des données stockées sur disque (RDB, AOF, backups)

Cette section détaille les exigences réglementaires, les implémentations techniques et les procédures de conformité pour le chiffrement Redis.

---

## Cadre réglementaire du chiffrement

### RGPD - Article 32 : Sécurité du traitement

**Obligation légale :**
> "Le responsable du traitement met en œuvre les mesures techniques et organisationnelles appropriées afin de garantir un niveau de sécurité adapté au risque, incluant si nécessaire le **chiffrement des données à caractère personnel**."

**Interprétation pour Redis :**
- Le chiffrement n'est pas toujours obligatoire mais **fortement recommandé**
- Nécessaire si les données présentent un risque pour les personnes (PII, données sensibles)
- L'absence de chiffrement doit être justifiée dans l'analyse de risques

**Critères d'évaluation du besoin de chiffrement :**
```
┌──────────────────────────────────────────────────────────────┐
│ Données stockées dans Redis        │ Chiffrement requis ?    │
├────────────────────────────────────┼─────────────────────────┤
│ Données publiques non sensibles    │ Non obligatoire         │
│ Identifiants techniques (user_id)  │ Recommandé (TLS mini)   │
│ Données personnelles (nom, email)  │ OUI (TLS + at-rest)     │
│ Données sensibles Art.9 (santé)    │ OUI + chiffrement app   │
│ Données financières (cartes)       │ OUI + HSM recommandé    │
│ Credentials (passwords, tokens)    │ OUI obligatoire         │
└────────────────────────────────────┴─────────────────────────┘
```

### PCI DSS - Requirements 3 et 4

**Requirement 3 : Protect Stored Cardholder Data**
- 3.4 : Chiffrement obligatoire pour les données de carte (PAN) stockées
- Algorithmes acceptés : AES-256, RSA 2048+, ECC

**Requirement 4 : Encrypt Transmission of Cardholder Data**
- 4.1 : TLS 1.2+ obligatoire pour transmission sur réseaux publics
- 4.2 : Ne jamais utiliser de protocoles non chiffrés (telnet, FTP, HTTP)

**Application à Redis :**
```
PCI DSS Scope pour Redis :
✅ TLS 1.2+ OBLIGATOIRE si Redis stocke/transmet des données de carte
✅ Chiffrement at-rest OBLIGATOIRE pour RDB/AOF contenant PAN
✅ Clés de chiffrement stockées dans HSM ou Key Management System
❌ INTERDIT de stocker CVV/CVC même chiffré
❌ INTERDIT de stocker code PIN même chiffré
```

### HIPAA - Security Rule §164.312

**§164.312(a)(2)(iv) : Encryption and Decryption (Addressable)**

**Statut "Addressable" ≠ Optionnel :**
- Si vous n'implémentez pas le chiffrement, vous DEVEZ documenter pourquoi et quelle mesure alternative équivalente est en place
- Dans la pratique, le chiffrement est quasi-obligatoire pour les PHI (Protected Health Information)

**Exigences pour Redis :**
```
HIPAA Encryption Requirements :
□ Chiffrement en transit (TLS 1.2+) pour toutes les communications
□ Chiffrement at-rest pour les données PHI
□ Gestion sécurisée des clés (pas en clair dans config)
□ Logs des accès aux clés de chiffrement
□ Rotation régulière des clés (au minimum annuelle)
□ Procédure de revocation des clés compromises
```

### SOC 2 - Common Criteria CC6.6

**Trust Services Criteria - Confidentiality :**
- Les données confidentielles doivent être chiffrées en transit et at-rest
- Les clés de chiffrement doivent être protégées et rotées
- L'accès aux clés doit être tracé et audité

### ISO 27001 - Annexe A.10

**A.10.1.1 : Policy on the use of cryptographic controls**
- Politique documentée d'utilisation du chiffrement
- Identification des algorithmes et protocoles approuvés
- Procédures de gestion du cycle de vie des clés

**A.10.1.2 : Key management**
- Génération sécurisée des clés
- Distribution contrôlée
- Stockage sécurisé (HSM, KMS)
- Rotation et révocation
- Destruction sécurisée en fin de vie

---

## Encryption in Transit (TLS/SSL)

### Vue d'ensemble

**Objectif :** Protéger les données Redis pendant leur transmission sur le réseau.

**Portée du chiffrement :**
```
┌────────────────────────────────────────────────────────────┐
│ Connexions à chiffrer avec TLS :                           │
│                                                            │
│ 1. Client → Redis Server (port 6379 → 6380)                │
│ 2. Redis Replica ← Master (replication)                    │
│ 3. Redis Sentinel ↔ Redis instances                        │
│ 4. Redis Cluster : Nœud ↔ Nœud (gossip protocol)           │
│ 5. Application ↔ Redis (via client library)                │
└────────────────────────────────────────────────────────────┘
```

### Protocoles et versions

#### Versions TLS acceptables

```
┌───────────────────────────────────────────────────────────────┐
│ Version TLS    │ Statut PCI DSS │ Statut HIPAA  │ Recommandé  │
├────────────────┼────────────────┼───────────────┼─────────────┤
│ SSL 2.0        │ ❌ INTERDIT    │ ❌ INTERDIT   │ NON         │
│ SSL 3.0        │ ❌ INTERDIT    │ ❌ INTERDIT   │ NON         │
│ TLS 1.0        │ ❌ INTERDIT    │ ⚠️ Déprécié   │ NON         │
│ TLS 1.1        │ ❌ INTERDIT    │ ⚠️ Déprécié   │ NON         │
│ TLS 1.2        │ ✅ Accepté     │ ✅ Minimum    │ Acceptable  │
│ TLS 1.3        │ ✅ Recommandé  │ ✅ Recommandé │ OUI         │
└───────────────────────────────────────────────────────────────┘

Note : PCI DSS 4.0 (mars 2024) interdit TLS 1.0/1.1 depuis juin 2024
```

**Configuration Redis pour TLS 1.3 uniquement (recommandé) :**
```conf
# redis.conf
tls-port 6380
port 0  # Désactiver le port non-TLS

# Forcer TLS 1.3
tls-protocols "TLSv1.3"

# Alternative : Accepter TLS 1.2 et 1.3
# tls-protocols "TLSv1.2 TLSv1.3"
```

#### Cipher suites approuvées

**Principes de sélection :**
- Forward Secrecy (PFS) : Utiliser ECDHE ou DHE
- Authentification forte : Préférer ECDSA ou RSA ≥2048 bits
- Chiffrement symétrique fort : AES-256-GCM ou ChaCha20-Poly1305
- Pas de chiffrements RC4, DES, 3DES, MD5, SHA1

**Cipher suites TLS 1.3 (recommandées) :**
```
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
```

**Cipher suites TLS 1.2 (si nécessaire pour compatibilité) :**
```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```

**Configuration Redis :**
```conf
# Forcer les cipher suites sécurisées
tls-ciphers "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256"

# Pour TLS 1.2, spécifier explicitement
# tls-ciphersuites "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256"
```

**Validation des cipher suites :**
```bash
# Tester les cipher suites acceptées
nmap --script ssl-enum-ciphers -p 6380 <redis-host>

# Avec openssl
openssl s_client -connect <redis-host>:6380 -tls1_3 -cipher 'TLS_AES_256_GCM_SHA384'
```

### Gestion des certificats

#### Types de certificats

**1. Certificats auto-signés**
```
❌ Utilisation : Développement/Test UNIQUEMENT
⚠️ Risques :
  - Pas de validation de l'identité du serveur
  - Vulnérable aux attaques Man-in-the-Middle (MITM)
  - Non conforme pour PCI DSS, HIPAA en production
```

**2. Certificats signés par CA interne**
```
✅ Utilisation : Environnements privés (intranet, VPC)
✅ Avantages :
  - Contrôle total sur le cycle de vie
  - Pas de coût récurrent
⚠️ Exigences :
  - CA racine doit être protégée (HSM recommandé)
  - Distribution sécurisée du certificat CA aux clients
  - Processus de révocation fonctionnel (CRL, OCSP)
```

**3. Certificats signés par CA publique**
```
✅ Utilisation : Exposition externe, conformité stricte
✅ Avantages :
  - Confiance native dans les navigateurs/OS
  - Révocation reconnue universellement
⚠️ Contraintes :
  - Coût (sauf Let's Encrypt gratuit)
  - Validation du domaine requise
  - Renouvellement tous les 90 jours (Let's Encrypt)
```

#### Configuration Redis TLS complète

**Structure des fichiers :**
```
/etc/redis/tls/
├── ca-cert.pem           # Certificat de l'autorité de certification
├── redis-cert.pem        # Certificat du serveur Redis
├── redis-key.pem         # Clé privée du serveur (600 permissions)
├── redis-dhparam.pem     # Paramètres Diffie-Hellman (optionnel)
└── client-cert.pem       # Certificat client (si mutual TLS)
```

**Configuration redis.conf :**
```conf
################################## TLS/SSL #####################################

# Port TLS (standard : 6380)
tls-port 6380

# Désactiver le port non-TLS (sécurité renforcée)
port 0

# Certificat du serveur Redis
tls-cert-file /etc/redis/tls/redis-cert.pem

# Clé privée du serveur (PROTÉGER avec 600)
tls-key-file /etc/redis/tls/redis-key.pem

# Certificat(s) de l'autorité de certification
tls-ca-cert-file /etc/redis/tls/ca-cert.pem

# Paramètres Diffie-Hellman (Forward Secrecy)
tls-dh-params-file /etc/redis/tls/redis-dhparam.pem

# Versions TLS autorisées (forcer TLS 1.3 recommandé)
tls-protocols "TLSv1.3"

# Cipher suites (TLS 1.3)
tls-ciphers "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256"

# Préférence des cipher suites (serveur impose l'ordre)
tls-prefer-server-ciphers yes

# Session caching (améliore les performances)
tls-session-caching yes
tls-session-cache-size 20480      # 20MB
tls-session-cache-timeout 300     # 5 minutes

########################## MUTUAL TLS (mTLS) ##################################

# Authentification mutuelle (client doit présenter un certificat valide)
tls-auth-clients yes

########################## RÉPLICATION TLS ####################################

# Activer TLS pour la réplication Master → Replica
tls-replication yes

########################## CLUSTER TLS ########################################

# Activer TLS pour le bus cluster (gossip protocol)
tls-cluster yes
```

---

## Encryption at Rest

### Vue d'ensemble

**Objectif :** Protéger les données Redis stockées sur disque.

**Données concernées :**
```
1. RDB snapshots (dump.rdb)
2. AOF files (appendonly.aof)
3. Backups (copies de RDB/AOF)
4. Swap (si activé, ⚠️ déconseillé)
5. Core dumps (en cas de crash)
6. Logs (si contiennent des données sensibles)
```

**Approches disponibles :**
```
┌───────────────────────────────────────────────────────────────────┐
│ Approche               │ Niveau      │ Performance  │ Complexité  │
├────────────────────────┼─────────────┼──────────────┼─────────────┤
│ Chiffrement filesystem │ OS          │ Excellent    │ Faible      │
│ Chiffrement disque     │ Hardware    │ Excellent    │ Faible      │
│ Redis Enterprise       │ Application │ Bon          │ Moyenne     │
│ Chiffrement applicatif │ Application │ Moyen        │ Élevée      │
└───────────────────────────────────────────────────────────────────┘
```

### Option 1 : Chiffrement du filesystem (recommandé)

#### LUKS (Linux Unified Key Setup)

**Avantages :**
- Transparent pour Redis
- Performance native (accélération matérielle)
- Mature et éprouvé

**Mise en œuvre :**

```bash
# 1. Créer la partition chiffrée
cryptsetup luksFormat /dev/sdb

# 2. Ouvrir la partition chiffrée
cryptsetup luksOpen /dev/sdb redis_encrypted

# 3. Créer le filesystem
mkfs.ext4 /dev/mapper/redis_encrypted

# 4. Monter le filesystem
mkdir -p /var/lib/redis-encrypted
mount /dev/mapper/redis_encrypted /var/lib/redis-encrypted

# 5. Configurer Redis
# redis.conf
dir /var/lib/redis-encrypted
```

---

## Checklist de conformité

### Encryption in Transit

```
Configuration TLS :
□ TLS 1.2 ou 1.3 activé (TLS 1.3 recommandé)
□ Port non-TLS désactivé (port 0)
□ Certificats valides (CA publique ou interne)
□ Cipher suites sécurisées uniquement
□ Forward Secrecy activé (ECDHE)
□ TLS pour réplication master-replica
□ TLS pour cluster bus (si cluster)
□ mTLS configuré si requis par la conformité

Gestion des certificats :
□ Certificats signés par CA de confiance
□ Clés privées protégées (600 permissions)
□ SAN (Subject Alternative Names) configurés
□ Rotation planifiée (30-60 jours avant expiration)
□ Monitoring de l'expiration (alertes)
□ Procédure de révocation documentée
□ Backup des certificats sécurisé

Validation :
□ Tests de connexion TLS réussis
□ Scan de vulnérabilités TLS (nmap, testssl.sh)
□ Version TLS validée (openssl s_client)
□ Cipher suites validées
□ Performance TLS mesurée (<20% overhead)
```

### Encryption at Rest

```
Configuration :
□ Filesystem chiffré (LUKS) OU
□ Disque auto-chiffrant (SED) OU
□ Redis Enterprise avec chiffrement natif
□ Chiffrement applicatif si données très sensibles
□ Backups chiffrés (GPG, S3 SSE-KMS)
□ Logs anonymisés ou chiffrés

Gestion des clés :
□ Clés stockées dans KMS/HSM (pas en clair)
□ Rotation annuelle minimum (trimestrielle recommandée)
□ Accès aux clés tracé et audité
□ Procédure de révocation documentée
□ Backup des clés sécurisé (offline, multiple locations)
□ Destruction sécurisée en fin de vie (shred, degauss)

Validation :
□ Tests de restauration depuis backup chiffré
□ Vérification de l'intégrité des données chiffrées
□ Performance at-rest mesurée (<10% overhead)
□ Audit des accès aux clés
```

### Documentation et procédures

```
□ Politique de chiffrement documentée et approuvée
□ Classification des données (quel niveau de chiffrement)
□ Procédures de génération des certificats
□ Procédures de rotation (TLS et at-rest)
□ Runbook de gestion des incidents (compromission)
□ Registre des clés (qui a accès, quand, pourquoi)
□ Formation des équipes sur la gestion des clés
□ Tests réguliers des procédures (quarterly)
```

### Conformité réglementaire

```
PCI DSS :
□ TLS 1.2+ pour toutes les transmissions
□ Chiffrement AES-256 pour données at-rest
□ Clés dans HSM ou équivalent
□ Rotation annuelle des clés
□ Logs des accès aux clés (10.3)
□ Tests de pénétration annuels (11.3)

HIPAA :
□ Chiffrement addressable documenté
□ TLS 1.2+ pour PHI en transit
□ Chiffrement at-rest pour PHI
□ Gestion des clés conforme
□ Logs d'audit des accès (6 ans)

RGPD :
□ Chiffrement adapté au risque (Article 32)
□ Documentation des mesures techniques
□ Tests réguliers (Article 32.1.d)
□ Notification en cas de violation (Article 33)
```

---

## Conclusion

Le chiffrement est une pierre angulaire de la conformité Redis. Cette section a couvert :

- ✅ **Exigences réglementaires** détaillées (RGPD, PCI DSS, HIPAA, SOC 2, ISO 27001)
- ✅ **Encryption in Transit** : Configuration TLS complète avec certificats, mutual TLS, rotation
- ✅ **Encryption at Rest** : LUKS, SED, Redis Enterprise, chiffrement applicatif
- ✅ **Gestion des clés** : KMS, HSM, rotation, révocation
- ✅ **Procédures opérationnelles** : Génération, déploiement, monitoring, incident response
- ✅ **Checklists** de conformité exhaustives

**Points critiques à retenir :**
1. TLS 1.3 est le standard recommandé (TLS 1.2 minimum)
2. Le chiffrement at-rest est obligatoire pour les données sensibles
3. La gestion des clés est aussi importante que le chiffrement lui-même
4. La rotation régulière est essentielle (certificats et clés)
5. La documentation et les tests sont requis pour la conformité

**Prochaines étapes :**
- Implémenter TLS selon la checklist
- Configurer le chiffrement at-rest approprié
- Établir un processus de gestion des clés
- Documenter toutes les procédures
- Former les équipes
- Planifier les audits réguliers

⏭️ [Audit logging et traçabilité](/17-gouvernance-conformite/03-audit-logging-tracabilite.md)
