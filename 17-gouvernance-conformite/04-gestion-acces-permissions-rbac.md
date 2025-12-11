🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.4 Gestion des Accès et des Permissions (RBAC)

## Introduction

La gestion des accès et des permissions est un pilier fondamental de la sécurité et de la conformité. Le principe du **moindre privilège** (Principle of Least Privilege - PoLP) exige que chaque utilisateur, application ou service n'ait accès qu'aux ressources strictement nécessaires à sa fonction. Pour Redis, l'introduction des **ACL (Access Control Lists)** dans la version 6.0 a révolutionné la gestion des permissions, permettant enfin un contrôle granulaire conforme aux exigences réglementaires.

**Définition RBAC (Role-Based Access Control) :**
> Modèle de contrôle d'accès où les permissions sont attribuées à des rôles plutôt qu'à des utilisateurs individuels. Les utilisateurs se voient ensuite attribuer des rôles selon leur fonction.

---

## Cadre réglementaire du contrôle d'accès

### RGPD - Article 32 : Sécurité du traitement

**Article 32.1 :**
> "Le responsable du traitement met en œuvre les mesures techniques et organisationnelles appropriées [...] notamment :
> b) des moyens permettant de garantir la confidentialité, l'intégrité, la disponibilité et la résilience constantes des systèmes et des services de traitement"

**Implications pour Redis :**
- Authentification obligatoire (pas d'accès anonyme)
- Contrôle d'accès granulaire selon le besoin
- Séparation des privilèges (admins vs users)
- Traçabilité des accès (logs d'audit)

**Article 5.1.f : Intégrité et confidentialité**
> "Traitées de façon à garantir une sécurité appropriée des données à caractère personnel, y compris la protection contre le traitement non autorisé ou illicite"

**Traduction technique :**
```
❌ Avant Redis 6 : Un seul mot de passe pour tous (requirepass)
   → Impossible de distinguer les utilisateurs
   → Tout le monde a tous les privilèges
   → Non-conforme RGPD

✅ Redis 6+ ACL : Comptes nominatifs avec permissions granulaires
   → Identification de chaque acteur
   → Principe du moindre privilège
   → Audit trail complet
   → Conforme RGPD
```

### PCI DSS - Requirements 7 et 8

**Requirement 7 : Restrict access to cardholder data**

**7.1 : Limit access to system components and cardholder data**
```
7.1.1 : Définir les besoins d'accès pour chaque rôle
7.1.2 : Restreindre l'accès aux utilisateurs privilégiés
7.1.3 : Attribuer l'accès basé sur la classification de données
7.1.4 : Exiger une autorisation documentée pour les accès
```

**7.2 : Establish access control systems**
```
7.2.1 : Couvrir tous les composants système
7.2.2 : Attribuer des privilèges selon le principe du moindre privilège
7.2.3 : Par défaut : "deny all" (refus par défaut)
7.2.4 : Authentification multi-facteurs pour accès admin
```

**7.3 : Document and review roles and responsibilities**
```
7.3.1 : Documenter tous les rôles et privilèges
7.3.2 : Documenter les approbations pour les accès privilégiés
7.3.3 : Revue au moins annuelle des rôles
```

**Requirement 8 : Identify and authenticate access**

**8.1 : Define and implement policies and procedures**
```
8.1.1 : Attribuer un identifiant unique à chaque utilisateur
8.1.2 : Interdire les comptes partagés/génériques
```

**8.2 : Strong authentication and password management**
```
8.2.1 : Authentification forte (MFA pour admins)
8.2.2 : Mots de passe complexes (longueur, complexité)
8.2.3 : Exiger un changement de mot de passe lors du premier login
8.2.4 : Changer les mots de passe tous les 90 jours minimum
8.2.5 : Ne pas réutiliser les 4 derniers mots de passe
8.2.6 : Définir les mots de passe/passphrases pour une première utilisation
```

**Application à Redis :**
```
□ Compte unique par utilisateur/application (ACL USER)
□ Pas de compte "default" en production
□ Mots de passe forts (>16 caractères)
□ Rotation trimestrielle des credentials
□ ACL basées sur le principe du moindre privilège
□ Documentation de tous les rôles (matrice RACI)
□ Revue annuelle des accès
□ MFA pour accès administratif (via bastion host)
```

### HIPAA - Security Rule §164.308

**§164.308(a)(3) : Workforce Security (Required)**

**(i) Authorization and/or supervision (Addressable)**
> "Implémenter des procédures pour l'autorisation et/ou la supervision de membres du personnel qui travaillent avec des PHI."

**(ii) Workforce clearance procedure (Addressable)**
> "Implémenter des procédures pour déterminer qu'un membre du personnel a l'autorisation appropriée pour accéder aux PHI."

**(iii) Termination procedures (Addressable)**
> "Implémenter des procédures pour terminer l'accès aux PHI lorsque l'emploi d'un membre du personnel se termine."

**§164.308(a)(4) : Information Access Management (Required)**

**(i) Isolating healthcare clearinghouse functions (Required si applicable)**
> "Si une entité est un healthcare clearinghouse, isoler les fonctions du clearinghouse."

**(ii) Access authorization (Addressable)**
> "Implémenter des politiques et procédures pour accorder l'accès aux PHI, par exemple, via des listes d'accès ou contrôles similaires."

**(iii) Access establishment and modification (Addressable)**
> "Implémenter des politiques et procédures qui établissent, documentent, révisent et modifient les droits d'accès d'un utilisateur aux PHI."

**§164.312(a)(1) : Access Control (Required)**
> "Implémenter des mesures techniques qui permettent uniquement aux personnes ou logiciels autorisés d'accéder aux PHI."

**Sous-spécifications (toutes Addressable mais quasi-obligatoires) :**
```
(i)   Unique user identification : Identifiant unique par utilisateur
(ii)  Emergency access procedure : Procédure d'accès d'urgence (break-glass)
(iii) Automatic logoff : Déconnexion automatique après inactivité
(iv)  Encryption and decryption : Chiffrement des PHI
```

**Application à Redis :**
```
□ Chaque utilisateur/app a un compte ACL unique
□ Permissions documentées et approuvées (workflow formel)
□ Procédure de provisioning (création de compte)
□ Procédure de deprovisioning (révocation à la fin de contrat)
□ Revue trimestrielle des accès (qui peut accéder à quoi)
□ Compte d'urgence break-glass (procédure documentée)
□ Timeout de session (via application ou proxy)
□ Chiffrement TLS obligatoire
```

### SOC 2 - Common Criteria

**CC6.1 : Logical and Physical Access Controls**
> "L'entité implémente des contrôles d'accès logiques et physiques pour restreindre l'accès non autorisé."

**CC6.2 : Prior to Issuing System Credentials**
> "Avant d'émettre des credentials système et d'accorder l'accès, l'entité enregistre et approuve les autorisations d'accès."

**CC6.3 : Revokes Access**
> "L'entité révoque l'accès au système lorsqu'il n'est plus approprié."

**Contrôles attendus pour Redis :**
```
1. Processus formel d'approbation pour nouveaux accès
   □ Demande écrite (ticket, formulaire)
   □ Justification métier
   □ Approbation du manager
   □ Validation par le responsable sécurité

2. Attribution selon le principe du moindre privilège
   □ Évaluation des besoins réels
   □ Attribution du rôle minimal suffisant
   □ Documentation de l'attribution

3. Révision périodique (au moins annuelle)
   □ Liste de tous les accès actuels
   □ Validation avec les managers
   □ Révocation des accès obsolètes

4. Révocation lors de changement de rôle ou départ
   □ Processus automatisé (intégré RH si possible)
   □ Délai maximum : 24h après notification
   □ Vérification post-révocation
```

### ISO 27001 - Annexe A.9

**A.9.1 : Business requirements for access control**

**A.9.1.1 : Access control policy (Required)**
> "Une politique de contrôle d'accès devrait être établie, documentée et revue selon les besoins métier et de sécurité."

**A.9.1.2 : Access to networks and network services (Required)**
> "Les utilisateurs ne devraient avoir accès qu'au réseau et aux services réseaux pour lesquels ils ont été spécifiquement autorisés."

**A.9.2 : User access management**

**A.9.2.1 : User registration and de-registration (Required)**
> "Un processus formel d'enregistrement et de désenregistrement des utilisateurs devrait être implémenté."

**A.9.2.2 : User access provisioning (Required)**
> "Un processus formel de provisioning des accès utilisateurs devrait être implémenté."

**A.9.2.3 : Management of privileged access rights (Required)**
> "L'allocation et l'utilisation des droits d'accès privilégiés devraient être restreintes et contrôlées."

**A.9.2.4 : Management of secret authentication information (Required)**
> "L'allocation d'informations d'authentification secrètes devrait être contrôlée via un processus de gestion formel."

**A.9.2.5 : Review of user access rights (Required)**
> "Les propriétaires des actifs devraient revoir les droits d'accès des utilisateurs à intervalles réguliers."

**A.9.2.6 : Removal or adjustment of access rights (Required)**
> "Les droits d'accès devraient être retirés ou ajustés lors de changement, départ ou fin de contrat."

**A.9.4 : System and application access control**

**A.9.4.1 : Information access restriction (Required)**
> "L'accès à l'information et aux fonctions des systèmes applicatifs devrait être restreint conformément à la politique de contrôle d'accès."

**A.9.4.2 : Secure log-on procedures (Required)**
> "Lorsque requis par la politique de contrôle d'accès, l'accès aux systèmes devrait être contrôlé par une procédure de connexion sécurisée."

**A.9.4.3 : Password management system (Required)**
> "Les systèmes de gestion de mots de passe devraient être interactifs et assurer des mots de passe de qualité."

---

## Redis ACL (Access Control Lists)

### Évolution historique

**Redis < 6.0 (Legacy) :**
```conf
# redis.conf
requirepass MySecretPassword123!

# Limitation :
# - Un seul mot de passe pour TOUTE l'instance
# - Pas d'identification utilisateur
# - Pas de granularité (tout ou rien)
# - Non-conforme pour PCI DSS, HIPAA, SOC 2
```

**Redis 6.0+ (ACL) :**
```bash
# Comptes nominatifs avec permissions granulaires
ACL SETUSER alice on >StrongPass123! ~keys:* +@read +@write
ACL SETUSER bob on >AnotherPass456! ~cache:* +@read
ACL SETUSER admin on >AdminPass789! ~* +@all

# Avantages :
# ✅ Identification unique de chaque utilisateur
# ✅ Principe du moindre privilège
# ✅ Audit trail (qui a fait quoi)
# ✅ Conforme aux exigences réglementaires
```

### Architecture ACL

**Composants d'une règle ACL :**
```
ACL SETUSER <username> <flags> <passwords> <keys> <commands>

Où :
- username : Identifiant unique (alphanumérique + underscore)
- flags : État du compte (on/off, nopass, etc.)
- passwords : Mots de passe (hash SHA-256)
- keys : Patterns de clés accessibles (~pattern)
- commands : Commandes autorisées (+cmd ou catégories +@cat)
```

**Syntaxe des permissions :**
```
FLAGS (état du compte) :
  on       : Compte activé
  off      : Compte désactivé
  nopass   : Pas de mot de passe (⚠️ dangereux, dev uniquement)
  resetpass: Supprimer tous les mots de passe

PASSWORDS :
  >password  : Ajouter un mot de passe (clair, sera hashé)
  #<hash>    : Ajouter un mot de passe déjà hashé (SHA-256)
  <password  : Supprimer un mot de passe spécifique
  nopass     : Autoriser connexion sans mot de passe
  resetpass  : Supprimer tous les mots de passe

KEY PATTERNS (clés accessibles) :
  ~*           : Toutes les clés
  ~keys:*      : Clés commençant par "keys:"
  ~user:123:*  : Clés d'un utilisateur spécifique
  allkeys      : Alias pour ~*
  resetkeys    : Retirer tous les patterns

COMMANDS (commandes autorisées) :
  +@all       : Toutes les commandes
  +@read      : Commandes lecture seule
  +@write     : Commandes écriture
  +@admin     : Commandes admin
  +@dangerous : Commandes dangereuses (FLUSHDB, etc.)
  +GET        : Autoriser une commande spécifique
  -DEL        : Interdire une commande spécifique
  allcommands : Alias pour +@all
  nocommands  : Interdire toutes les commandes
```

### Catégories de commandes

**Catégories prédéfinies :**
```bash
# Lister toutes les catégories
ACL CAT

# Catégories principales :
@read       : Lecture (GET, HGET, LRANGE, SMEMBERS, etc.)
@write      : Écriture (SET, HSET, LPUSH, SADD, etc.)
@admin      : Administration (CONFIG, ACL, SHUTDOWN, etc.)
@dangerous  : Dangereuses (FLUSHDB, FLUSHALL, KEYS *, etc.)
@keyspace   : Gestion clés (DEL, EXISTS, EXPIRE, etc.)
@string     : Commandes String
@list       : Commandes List
@set        : Commandes Set
@sortedset  : Commandes Sorted Set
@hash       : Commandes Hash
@pubsub     : Pub/Sub
@stream     : Streams
@scripting  : Lua scripting
@geo        : Géospatial
@hyperloglog: HyperLogLog
@bitmap     : Bitmaps
@transaction: Transactions (MULTI, EXEC)
@connection : Connexion (AUTH, PING, QUIT)
@slow       : Commandes lentes potentielles

# Lister les commandes d'une catégorie
ACL CAT read
# 1) "get"
# 2) "strlen"
# 3) "hget"
# ...
```

### Exemples de configuration ACL

#### Rôle 1 : Application en lecture seule

```bash
# Application qui lit uniquement des données de cache
ACL SETUSER app_reader \
  on \
  >ReaderPass2024! \
  ~cache:* \
  +@read \
  -@write \
  -@admin \
  -@dangerous

# Permissions :
# ✅ Lire les clés cache:*
# ✅ GET, HGET, LRANGE, SMEMBERS, etc.
# ❌ Aucune écriture
# ❌ Aucune commande admin
```

**Validation :**
```bash
# Connexion avec le compte
redis-cli -u redis://app_reader:ReaderPass2024!@localhost:6379

# ✅ Autorisé
GET cache:product:123

# ❌ Interdit (clé hors du pattern)
GET user:456:email
# (error) NOPERM this user has no permissions to access one of the keys used as arguments

# ❌ Interdit (commande write)
SET cache:test "value"
# (error) NOPERM this user has no permissions to run the 'set' command
```

#### Rôle 2 : Application avec écriture limitée

```bash
# Application qui peut lire et écrire dans son namespace
ACL SETUSER app_service_xyz \
  on \
  >ServicePass2024! \
  ~service:xyz:* \
  +@read \
  +@write \
  +@keyspace \
  -@admin \
  -@dangerous \
  -FLUSHDB \
  -FLUSHALL \
  -KEYS

# Permissions :
# ✅ Lire et écrire dans service:xyz:*
# ✅ GET, SET, HSET, DEL, EXPIRE
# ❌ Pas d'admin
# ❌ Pas de FLUSHDB/FLUSHALL
# ❌ Pas de KEYS (danger performance)
```

#### Rôle 3 : Administrateur limité (DevOps)

```bash
# Admin qui peut tout faire sauf supprimer les données
ACL SETUSER devops_admin \
  on \
  >AdminPass2024! \
  ~* \
  +@all \
  -@dangerous \
  -FLUSHDB \
  -FLUSHALL \
  -SHUTDOWN

# Permissions :
# ✅ Toutes les clés
# ✅ CONFIG, ACL, INFO, MONITOR
# ✅ BGSAVE, REWRITE
# ❌ FLUSHDB/FLUSHALL (sécurité)
# ❌ SHUTDOWN (haute disponibilité)
```

#### Rôle 4 : Super-admin (break-glass uniquement)

```bash
# Compte d'urgence avec tous les privilèges
# ⚠️ À utiliser uniquement en cas d'urgence critique
ACL SETUSER superadmin \
  on \
  >SuperSecretPass2024!ChangeMeNow \
  ~* \
  +@all

# Permissions :
# ✅ Accès complet sans restriction
# ⚠️ Utilisation doit être loggée et justifiée
# ⚠️ Audit obligatoire après usage
```

#### Rôle 5 : Application analytics (lecture + agrégations)

```bash
# Application d'analytics qui lit et fait des agrégations
ACL SETUSER app_analytics \
  on \
  >AnalyticsPass2024! \
  ~analytics:* ~stats:* \
  +@read \
  +@sortedset \
  +@hyperloglog \
  +@stream \
  -@write \
  -@admin \
  -@dangerous

# Permissions :
# ✅ Lire analytics:* et stats:*
# ✅ ZRANGE, ZREVRANGE (sorted sets pour leaderboards)
# ✅ PFCOUNT, PFADD (HyperLogLog pour comptage unique)
# ✅ XREAD, XREADGROUP (streams pour events)
# ❌ Pas d'écriture directe (sauf PFADD/XADD si nécessaire)
```

#### Rôle 6 : Service de cache partagé

```bash
# Service qui peut uniquement gérer son cache
ACL SETUSER cache_service \
  on \
  >CachePass2024! \
  ~cache:shared:* \
  +@read \
  +SET +SETEX +DEL +EXPIRE \
  -@admin \
  -@dangerous

# Permissions :
# ✅ GET/SET/DEL dans cache:shared:*
# ✅ Gestion TTL (SETEX, EXPIRE)
# ❌ Pas de commandes complexes
# ❌ Pas d'admin
```

### Configuration ACL via fichier

**Fichier users.acl :**
```acl
# /etc/redis/users.acl
# Format : un utilisateur par ligne

# Désactiver l'utilisateur default (sécurité)
user default off -@all

# Applications
user app_reader on #<sha256-hash> ~cache:* +@read -@write -@admin -@dangerous
user app_service_xyz on #<sha256-hash> ~service:xyz:* +@read +@write +@keyspace -@admin -@dangerous
user app_analytics on #<sha256-hash> ~analytics:* ~stats:* +@read +@sortedset +@hyperloglog -@write

# Admins
user devops_admin on #<sha256-hash> ~* +@all -@dangerous -FLUSHDB -FLUSHALL -SHUTDOWN
user superadmin on #<sha256-hash> ~* +@all

# Monitoring (lecture seule de métriques)
user prometheus_exporter on #<sha256-hash> nokeys +INFO +CLIENT +PING +SLOWLOG +LATENCY
```

**Configuration redis.conf :**
```conf
# Charger les ACL depuis un fichier
aclfile /etc/redis/users.acl

# Activer l'ACL log (échecs de permissions)
acllog-max-len 128
```

**Générer les hashes de mots de passe :**
```bash
# Option 1 : Utiliser Redis CLI
redis-cli ACL GENPASS 32
# Génère un mot de passe aléatoire de 32 caractères

# Option 2 : Hasher un mot de passe existant
echo -n "MyPassword123!" | sha256sum | awk '{print $1}'
# Résultat : <hash-sha256>

# Option 3 : Avec OpenSSL
echo -n "MyPassword123!" | openssl dgst -sha256 | awk '{print $2}'
```

**Recharger les ACL sans redémarrage :**
```bash
# Recharger depuis le fichier
ACL LOAD

# Sauvegarder les ACL actuelles dans le fichier
ACL SAVE
```

### Gestion des mots de passe

**Politique de mots de passe forts (PCI DSS 8.2.3) :**
```
Exigences :
- Longueur minimum : 12 caractères (16+ recommandé)
- Complexité : Majuscules, minuscules, chiffres, symboles
- Pas de mots du dictionnaire
- Pas de patterns évidents (azerty, 123456, etc.)
- Pas de réutilisation des 4 derniers mots de passe
- Changement tous les 90 jours (PCI DSS 8.2.4)
```

**Génération de mots de passe sécurisés :**
```bash
#!/bin/bash
# Script de génération de mots de passe conformes

generate_redis_password() {
    # Génère un mot de passe de 24 caractères aléatoires
    # Inclut : a-z, A-Z, 0-9, symboles sûrs

    local length=${1:-24}

    # Utiliser /dev/urandom (cryptographiquement sécurisé)
    tr -dc 'A-Za-z0-9!@#$%^&*()_+-=' < /dev/urandom | head -c $length
    echo
}

# Générer un mot de passe
password=$(generate_redis_password 24)
echo "Generated password: $password"

# Hasher pour ACL
hash=$(echo -n "$password" | sha256sum | awk '{print $1}')
echo "SHA-256 hash: $hash"

# Commande ACL (à exécuter dans Redis)
echo "ACL SETUSER myuser on #$hash ~* +@all"
```

**Rotation des mots de passe (procédure) :**

```bash
#!/bin/bash
# Script de rotation de mot de passe Redis ACL

set -e

USERNAME="$1"
NEW_PASSWORD="$2"

if [ -z "$USERNAME" ] || [ -z "$NEW_PASSWORD" ]; then
    echo "Usage: $0 <username> <new_password>"
    exit 1
fi

# Hash du nouveau mot de passe
NEW_HASH=$(echo -n "$NEW_PASSWORD" | sha256sum | awk '{print $1}')

echo "Rotating password for user: $USERNAME"

# Étape 1 : Ajouter le nouveau mot de passe (l'ancien reste valide)
redis-cli ACL SETUSER "$USERNAME" "#$NEW_HASH"
echo "[1/4] New password added (old password still valid)"

# Étape 2 : Attendre que toutes les applications migrent (période de grâce)
echo "[2/4] Waiting for applications to migrate (60 seconds grace period)"
echo "      Update your application configs with the new password now!"
sleep 60

# Étape 3 : Vérifier qu'il n'y a plus de connexions avec l'ancien password
ACTIVE_CONNECTIONS=$(redis-cli CLIENT LIST | grep "user=$USERNAME" | wc -l)
echo "[3/4] Active connections for $USERNAME: $ACTIVE_CONNECTIONS"

if [ "$ACTIVE_CONNECTIONS" -gt 0 ]; then
    echo "⚠️  Warning: $ACTIVE_CONNECTIONS connections still active"
    echo "    Waiting 30 more seconds..."
    sleep 30
fi

# Étape 4 : Supprimer l'ancien mot de passe (resetpass puis ajouter le nouveau)
redis-cli ACL SETUSER "$USERNAME" resetpass "#$NEW_HASH"
echo "[4/4] Old password revoked, new password is now the only valid one"

# Sauvegarder les ACL
redis-cli ACL SAVE
echo "✅ Password rotation completed and saved"

# Audit log
echo "[$(date)] Password rotated for user: $USERNAME" >> /var/log/redis/password_rotation.log
```

**Procédure de rotation coordonnée :**
```
1. Générer un nouveau mot de passe fort
2. Ajouter le nouveau password (l'ancien reste valide)
3. Déployer le nouveau password dans toutes les applications (rolling update)
4. Période de grâce : 24-48h pour migration complète
5. Vérifier qu'aucune connexion n'utilise l'ancien password
6. Révoquer l'ancien password
7. Logger l'opération (audit trail)
8. Notifier les équipes
```

---

## Principe du moindre privilège (PoLP)

### Méthodologie d'application

**Étape 1 : Identification des besoins**
```
Questions à poser pour chaque utilisateur/application :
1. Quelle est la fonction métier ?
2. Quelles données Redis doit-il accéder ?
3. Quelles opérations doit-il effectuer (lecture, écriture, admin) ?
4. Y a-t-il des commandes spécifiques nécessaires ?
5. Y a-t-il des commandes à interdire explicitement ?

Exemple :
- Fonction : Service de cache produit
- Données : Clés cache:product:*
- Opérations : Lecture (GET) et écriture (SET, SETEX, DEL)
- Commandes : GET, MGET, SET, SETEX, DEL, EXPIRE
- Interdictions : KEYS, FLUSHDB, CONFIG, toutes commandes admin
```

**Étape 2 : Mapping vers permissions ACL**
```bash
# Besoin identifié → Traduction ACL

Besoin : "Lire les produits en cache"
ACL : ~cache:product:* +GET +MGET

Besoin : "Écrire dans le cache avec TTL"
ACL : ~cache:product:* +SET +SETEX +EXPIRE

Besoin : "Supprimer des entrées de cache"
ACL : ~cache:product:* +DEL

Résultat final :
ACL SETUSER cache_product on >pass \
  ~cache:product:* \
  +GET +MGET +SET +SETEX +DEL +EXPIRE \
  -@admin -@dangerous
```

**Étape 3 : Validation et tests**
```bash
# Tester avec le compte créé
redis-cli -u redis://cache_product:pass@localhost:6379

# Tests positifs (devrait fonctionner)
SET cache:product:123 "data"       # ✅
GET cache:product:123              # ✅
SETEX cache:product:456 3600 "x"   # ✅
DEL cache:product:789              # ✅

# Tests négatifs (devrait échouer)
SET user:999:name "John"           # ❌ Clé hors namespace
FLUSHDB                            # ❌ Commande dangereuse
CONFIG GET *                       # ❌ Commande admin
KEYS cache:product:*               # ❌ Commande bloquée (performance)
```

**Étape 4 : Documentation**
```markdown
## Compte : cache_product

**Fonction :** Service de cache des informations produit
**Propriétaire :** Équipe E-commerce
**Approbation :** Manager E-commerce + RSSI
**Date création :** 2024-12-11
**Dernière revue :** 2024-12-11

**Permissions :**
- Namespace : cache:product:*
- Lecture : GET, MGET
- Écriture : SET, SETEX, DEL
- Gestion TTL : EXPIRE
- Interdictions : Admin, Dangerous, KEYS

**Justification :**
Service nécessite un cache rapide pour les données produit.
Accès limité au strict nécessaire (principe du moindre privilège).

**Applications utilisant ce compte :**
- product-service-api (v2.3.1)
- product-recommendation-engine (v1.8.0)

**Revue prévue :** Trimestrielle (prochaine : 2025-03-11)
```

### Matrice RACI des permissions

**Exemple de matrice pour Redis :**

| Rôle / Permissions | Lecture | Écriture | Config | ACL | Flush | Cluster | Justification |
|--------------------|---------|----------|--------|-----|-------|---------|---------------|
| **app_reader** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | Lecture seule cache |
| **app_writer** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | Cache read/write |
| **app_analytics** | ✅ | 📊 | ❌ | ❌ | ❌ | ❌ | Analytics + agrégations |
| **devops_operator** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | Ops quotidiennes |
| **devops_admin** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | Admin infrastructure |
| **security_auditor** | 📊 | ❌ | 📊 | 📊 | ❌ | ❌ | Audit et conformité |
| **superadmin** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Break-glass uniquement |

**Légende :**
- ✅ : Accès complet
- ❌ : Accès interdit
- 📊 : Accès lecture seule (métriques, logs)

### Ségrégation des duties (SoD)

**Principe :**
Séparer les responsabilités pour éviter qu'une seule personne puisse commettre une fraude ou erreur critique sans détection.

**Application à Redis :**
```
Séparation 1 : Développement vs Production
- Développeurs : Accès lecture seule à prod (pour debug)
- DevOps : Accès admin prod, pas de modif code

Séparation 2 : Création vs Approbation
- Opérateur : Peut créer un compte ACL (draft)
- Admin : Doit approuver avant activation

Séparation 3 : Opérations vs Audit
- DevOps : Gère l'infrastructure Redis
- Security : Audit des accès, indépendant de DevOps

Séparation 4 : Données vs Infrastructure
- DBA : Gère les données (backup, restore)
- SysAdmin : Gère l'infrastructure (serveurs, réseau)
```

**Implémentation technique :**
```bash
# Environnement DEV
ACL SETUSER dev_alice on >pass ~* +@all
# Développeurs ont accès complet en dev

# Environnement PROD
ACL SETUSER dev_alice on >pass nokeys +INFO +CLIENT +SLOWLOG +LATENCY
# Développeurs n'ont que lecture métriques en prod

# DevOps PROD
ACL SETUSER devops_bob on >pass ~* +@all -@dangerous -FLUSHDB -FLUSHALL

# Security Auditor
ACL SETUSER security_carol on >pass nokeys +INFO +CLIENT +ACL +CONFIG
# Peut auditer mais pas modifier
```

---

## Gestion du cycle de vie des comptes

### Processus de provisioning

**Workflow formel (conformité SOC 2, ISO 27001) :**

```
┌─────────────────────────────────────────────────────────────────┐
│ Étape 1 : Demande                                               │
├─────────────────────────────────────────────────────────────────┤
│ - Demandeur : Manager ou Tech Lead                              │
│ - Formulaire : Justification métier + besoins détaillés         │
│ - Ticket : Système de ticketing (Jira, ServiceNow)              │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 2 : Évaluation et approbation                             │
├─────────────────────────────────────────────────────────────────┤
│ - Évaluateur : Architecte sécurité ou DBA Redis                 │
│ - Vérifications :                                               │
│   □ Justification valide                                        │
│   □ Principe du moindre privilège respecté                      │
│   □ Pas de compte existant réutilisable                         │
│   □ Classification des données accessibles                      │
│ - Approbateur : RSSI ou responsable sécurité                    │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 3 : Création du compte                                    │
├─────────────────────────────────────────────────────────────────┤
│ - Exécutant : DevOps ou DBA                                     │
│ - Actions :                                                     │
│   1. Générer mot de passe fort (24+ caractères)                 │
│   2. Créer le compte ACL avec permissions minimales             │
│   3. Documenter dans le registre des accès                      │
│   4. Tester les permissions                                     │
│   5. Activer le compte                                          │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 4 : Distribution sécurisée                                │
├─────────────────────────────────────────────────────────────────┤
│ - Credentials transmis via canal sécurisé :                     │
│   • Vault (HashiCorp Vault, AWS Secrets Manager)                │
│   • PGP/GPG chiffré                                             │
│   • Pas par email/Slack non chiffré !                           │
│ - Confirmation de réception                                     │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 5 : Premier accès et validation                           │
├─────────────────────────────────────────────────────────────────┤
│ - Utilisateur/App se connecte avec credentials temporaires      │
│ - Changement de mot de passe obligatoire (si humain)            │
│ - Test des permissions                                          │
│ - Signature de l'acceptable use policy                          │
└─────────────────────────────────────────────────────────────────┘
```

**Script de provisioning automatisé :**

```bash
#!/bin/bash
# Redis ACL User Provisioning Script
# Conforme SOC 2 / ISO 27001

set -e

# Configuration
REDIS_HOST="${REDIS_HOST:-localhost}"
REDIS_PORT="${REDIS_PORT:-6379}"
REDIS_ADMIN_USER="${REDIS_ADMIN_USER:-admin}"
REDIS_ADMIN_PASS="${REDIS_ADMIN_PASS}"
VAULT_ADDR="${VAULT_ADDR:-https://vault.internal:8200}"
AUDIT_LOG="/var/log/redis/user_provisioning.log"

# Fonction de logging
log_audit() {
    echo "[$(date -u +"%Y-%m-%dT%H:%M:%SZ")] $1" | tee -a "$AUDIT_LOG"
}

# Fonction de génération de mot de passe
generate_password() {
    # Générer un mot de passe de 24 caractères
    tr -dc 'A-Za-z0-9!@#$%^&*()_+-=' < /dev/urandom | head -c 24
}

# Fonction principale de provisioning
provision_user() {
    local username="$1"
    local role="$2"
    local justification="$3"
    local approver="$4"
    local ticket_id="$5"

    log_audit "START: Provisioning user $username (role: $role)"
    log_audit "Ticket: $ticket_id | Approver: $approver"
    log_audit "Justification: $justification"

    # Vérifier si l'utilisateur existe déjà
    if redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       ACL GETUSER "$username" &>/dev/null; then
        echo "ERROR: User $username already exists"
        log_audit "ERROR: User $username already exists - ABORTED"
        exit 1
    fi

    # Générer le mot de passe
    local password=$(generate_password)
    local password_hash=$(echo -n "$password" | sha256sum | awk '{print $1}')

    log_audit "Password generated (hash: ${password_hash:0:16}...)"

    # Déterminer les permissions selon le rôle
    local acl_rules=""
    case "$role" in
        "app_reader")
            acl_rules="~cache:* +@read -@write -@admin -@dangerous"
            ;;
        "app_writer")
            acl_rules="~app:* +@read +@write +@keyspace -@admin -@dangerous -KEYS -FLUSHDB"
            ;;
        "devops_operator")
            acl_rules="~* +@all -@dangerous -FLUSHDB -FLUSHALL -SHUTDOWN"
            ;;
        "devops_admin")
            acl_rules="~* +@all -@dangerous"
            ;;
        *)
            echo "ERROR: Unknown role $role"
            log_audit "ERROR: Unknown role $role - ABORTED"
            exit 1
            ;;
    esac

    log_audit "ACL rules: $acl_rules"

    # Créer l'utilisateur
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       ACL SETUSER "$username" on "#$password_hash" $acl_rules

    if [ $? -eq 0 ]; then
        log_audit "User $username created successfully"
    else
        log_audit "ERROR: Failed to create user $username"
        exit 1
    fi

    # Sauvegarder les ACL
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       ACL SAVE

    log_audit "ACL configuration saved to disk"

    # Stocker le mot de passe dans Vault
    vault kv put secret/redis/users/"$username" \
        password="$password" \
        created_at="$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
        created_by="$(whoami)" \
        role="$role" \
        ticket_id="$ticket_id" \
        approver="$approver"

    log_audit "Password stored in Vault at secret/redis/users/$username"

    # Mettre à jour le registre des accès
    cat >> /var/log/redis/access_registry.csv <<EOF
$username,$role,"$justification",$approver,$ticket_id,$(date -u +"%Y-%m-%dT%H:%M:%SZ"),active
EOF

    log_audit "Access registry updated"

    # Notification (email, Slack, etc.)
    send_notification "$approver" "Redis user $username has been provisioned (ticket: $ticket_id)"

    log_audit "END: User $username provisioned successfully"

    echo "✅ User $username created successfully"
    echo "   Password stored in Vault: secret/redis/users/$username"
    echo "   Retrieve with: vault kv get secret/redis/users/$username"
}

# Fonction de notification (exemple avec email)
send_notification() {
    local recipient="$1"
    local message="$2"

    echo "$message" | mail -s "Redis User Provisioning" "$recipient@example.com"
}

# Point d'entrée
if [ $# -lt 5 ]; then
    echo "Usage: $0 <username> <role> <justification> <approver> <ticket_id>"
    echo ""
    echo "Roles:"
    echo "  - app_reader      : Read-only access to cache"
    echo "  - app_writer      : Read/write access to app namespace"
    echo "  - devops_operator : Operational access (no dangerous commands)"
    echo "  - devops_admin    : Administrative access"
    exit 1
fi

provision_user "$1" "$2" "$3" "$4" "$5"
```

**Utilisation :**
```bash
# Provisionner un nouvel utilisateur
sudo ./provision_redis_user.sh \
    "app_service_checkout" \
    "app_writer" \
    "E-commerce checkout service requires Redis for session management" \
    "alice.security@example.com" \
    "JIRA-12345"

# Le script :
# 1. Vérifie que l'utilisateur n'existe pas
# 2. Génère un mot de passe fort
# 3. Crée le compte ACL avec les bonnes permissions
# 4. Sauvegarde dans Vault
# 5. Log dans l'audit trail
# 6. Notifie l'approbateur
```

### Processus de deprovisioning

**Déclencheurs de révocation :**
```
1. Fin de contrat / Départ d'un employé
2. Changement de rôle (ne nécessite plus l'accès)
3. Fin de projet temporaire
4. Compromission suspectée du compte
5. Inactivité prolongée (>90 jours selon politique)
6. Non-conformité lors de l'audit des accès
```

**Workflow de révocation :**
```
┌─────────────────────────────────────────────────────────────────┐
│ Étape 1 : Détection / Notification                              │
├─────────────────────────────────────────────────────────────────┤
│ - Déclencheur : RH (départ), Manager (changement), Audit        │
│ - Ticket : Création automatique ou manuelle                     │
│ - SLA : 24h maximum après notification                          │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 2 : Évaluation de l'impact                                │
├─────────────────────────────────────────────────────────────────┤
│ - Vérifier les services dépendants                              │
│ - Planifier la révocation (immédiate ou programmée)             │
│ - Identifier les credentials à changer (si compromission)       │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 3 : Désactivation                                         │
├─────────────────────────────────────────────────────────────────┤
│ 1. Désactiver le compte (ACL SETUSER username off)              │
│ 2. Fermer les connexions actives (CLIENT KILL)                  │
│ 3. Vérifier qu'aucune connexion résiduelle                      │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 4 : Archivage et suppression                              │
├─────────────────────────────────────────────────────────────────┤
│ - Archiver la configuration ACL (backup)                        │
│ - Supprimer le compte après 30 jours (période de grâce)         │
│ - Supprimer de Vault                                            │
│ - Mettre à jour le registre des accès                           │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Étape 5 : Audit post-révocation                                 │
├─────────────────────────────────────────────────────────────────┤
│ - Revue des logs d'accès du compte révoqué                      │
│ - Détection d'activités suspectes                               │
│ - Rapport de révocation                                         │
└─────────────────────────────────────────────────────────────────┘
```

**Script de deprovisioning :**

```bash
#!/bin/bash
# Redis ACL User Deprovisioning Script

set -e

REDIS_HOST="${REDIS_HOST:-localhost}"
REDIS_PORT="${REDIS_PORT:-6379}"
REDIS_ADMIN_USER="${REDIS_ADMIN_USER:-admin}"
REDIS_ADMIN_PASS="${REDIS_ADMIN_PASS}"
AUDIT_LOG="/var/log/redis/user_deprovisioning.log"

log_audit() {
    echo "[$(date -u +"%Y-%m-%dT%H:%M:%SZ")] $1" | tee -a "$AUDIT_LOG"
}

deprovision_user() {
    local username="$1"
    local reason="$2"
    local ticket_id="$3"

    log_audit "START: Deprovisioning user $username"
    log_audit "Reason: $reason | Ticket: $ticket_id"

    # Vérifier que l'utilisateur existe
    if ! redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       ACL GETUSER "$username" &>/dev/null; then
        echo "ERROR: User $username does not exist"
        log_audit "ERROR: User $username not found - ABORTED"
        exit 1
    fi

    # Archiver la configuration actuelle
    local config_backup="/var/backups/redis/acl_${username}_$(date +%Y%m%d_%H%M%S).txt"
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       ACL GETUSER "$username" > "$config_backup"

    log_audit "Configuration backed up to $config_backup"

    # Lister les connexions actives de cet utilisateur
    local active_connections=$(redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       CLIENT LIST | grep "user=$username" | wc -l)

    log_audit "Active connections for $username: $active_connections"

    if [ "$active_connections" -gt 0 ]; then
        echo "⚠️  Warning: $active_connections active connections will be killed"

        # Killer toutes les connexions de cet utilisateur
        redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
           -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
           CLIENT LIST | grep "user=$username" | awk '{print $2}' | cut -d= -f2 | while read id; do
            redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
               -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
               CLIENT KILL ID "$id"
        done

        log_audit "All connections for $username killed"
    fi

    # Désactiver le compte (ne pas supprimer immédiatement)
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       ACL SETUSER "$username" off

    log_audit "User $username disabled"

    # Sauvegarder
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
       -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
       ACL SAVE

    # Planifier la suppression définitive (30 jours)
    echo "redis-cli ACL DELUSER $username && redis-cli ACL SAVE" | at now + 30 days
    log_audit "Scheduled deletion in 30 days"

    # Mettre à jour le registre des accès
    sed -i "s/,$username,.*,active$/,$username,.*,revoked - $reason - $(date -u +"%Y-%m-%dT%H:%M:%SZ")/" \
        /var/log/redis/access_registry.csv

    log_audit "Access registry updated"

    # Archiver le mot de passe de Vault (ne pas supprimer immédiatement)
    vault kv metadata put secret/redis/users/"$username" \
        revoked=true \
        revoked_at="$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
        revoked_reason="$reason" \
        revoked_ticket="$ticket_id"

    log_audit "Vault entry marked as revoked"

    # Générer un rapport d'activité du compte révoqué
    generate_activity_report "$username"

    log_audit "END: User $username deprovisioned"

    echo "✅ User $username deprovisioned successfully"
    echo "   - Account disabled"
    echo "   - Connections killed: $active_connections"
    echo "   - Configuration backed up: $config_backup"
    echo "   - Permanent deletion scheduled in 30 days"
}

generate_activity_report() {
    local username="$1"
    local report_file="/var/log/redis/deprovision_reports/${username}_$(date +%Y%m%d_%H%M%S).txt"

    mkdir -p /var/log/redis/deprovision_reports

    cat > "$report_file" <<EOF
Redis User Deprovisioning Report
================================
User: $username
Date: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
Initiated by: $(whoami)

Last 30 days activity:
$(grep "$username" /var/log/redis/audit.log | tail -100)

Configuration at time of deprovisioning:
$(cat /var/backups/redis/acl_${username}_*.txt | tail -1)

EOF

    log_audit "Activity report generated: $report_file"
}

# Point d'entrée
if [ $# -lt 3 ]; then
    echo "Usage: $0 <username> <reason> <ticket_id>"
    exit 1
fi

deprovision_user "$1" "$2" "$3"
```

---

## Revue périodique des accès

### Fréquence de revue (conformité)

```
┌───────────────────────────────────────────────────────────────┐
│ Type de revue        │ Fréquence minimale │ Réglementation   │
├──────────────────────┼────────────────────┼──────────────────┤
│ Comptes privilégiés  │ Trimestrielle      │ PCI DSS 7.3.3    │
│ Tous les comptes     │ Annuelle           │ SOC 2, ISO 27001 │
│ Comptes inactifs     │ Mensuelle          │ Bonne pratique   │
│ Comptes d'urgence    │ Après chaque usage │ HIPAA            │
└───────────────────────────────────────────────────────────────┘
```

### Procédure de revue trimestrielle

**Checklist de revue :**
```
□ Générer la liste complète des comptes ACL
□ Identifier le propriétaire/responsable de chaque compte
□ Vérifier la date de dernière utilisation
□ Valider que les permissions sont toujours appropriées
□ Vérifier l'alignement avec les rôles actuels des personnes
□ Identifier les comptes inactifs (>90 jours sans usage)
□ Identifier les comptes orphelins (propriétaire parti)
□ Vérifier la rotation des mots de passe (<90 jours)
□ Documenter les anomalies détectées
□ Planifier les actions correctives
□ Obtenir les signatures d'approbation
□ Archiver le rapport de revue
```

**Script de génération de rapport :**

```bash
#!/bin/bash
# Redis Access Review Report Generator

REDIS_HOST="localhost"
REDIS_PORT="6379"
REDIS_ADMIN_USER="admin"
REDIS_ADMIN_PASS="$REDIS_ADMIN_PASS"
REPORT_FILE="redis_access_review_$(date +%Y%m%d).html"

# Récupérer la liste des utilisateurs
users=$(redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
    -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
    ACL LIST | awk '{print $2}')

# Générer le rapport HTML
cat > "$REPORT_FILE" <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Redis Access Review Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { color: #333; }
        table { border-collapse: collapse; width: 100%; margin-top: 20px; }
        th { background-color: #4CAF50; color: white; padding: 12px; text-align: left; }
        td { border: 1px solid #ddd; padding: 8px; }
        tr:nth-child(even) { background-color: #f2f2f2; }
        .active { color: green; font-weight: bold; }
        .inactive { color: orange; font-weight: bold; }
        .disabled { color: red; font-weight: bold; }
    </style>
</head>
<body>
    <h1>Redis Access Review Report</h1>
    <p><strong>Date:</strong> $(date +"%Y-%m-%d %H:%M:%S")</p>
    <p><strong>Redis Instance:</strong> $REDIS_HOST:$REDIS_PORT</p>
    <p><strong>Total Users:</strong> $(echo "$users" | wc -w)</p>

    <h2>User Details</h2>
    <table>
        <tr>
            <th>Username</th>
            <th>Status</th>
            <th>Permissions</th>
            <th>Key Pattern</th>
            <th>Last Activity</th>
            <th>Action Required</th>
        </tr>
EOF

for user in $users; do
    # Récupérer les détails du compte
    user_info=$(redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" \
        -u "redis://$REDIS_ADMIN_USER:$REDIS_ADMIN_PASS@$REDIS_HOST:$REDIS_PORT" \
        ACL GETUSER "$user")

    # Parser le statut
    if echo "$user_info" | grep -q "flags.*on"; then
        status="<span class='active'>ACTIVE</span>"
    else
        status="<span class='disabled'>DISABLED</span>"
    fi

    # Extraire les permissions
    permissions=$(echo "$user_info" | grep "commands" | cut -d: -f2 | head -c 50)
    key_pattern=$(echo "$user_info" | grep "keys" | cut -d: -f2 | head -c 30)

    # Chercher la dernière activité dans les logs d'audit
    last_activity=$(grep "$user" /var/log/redis/audit.log | tail -1 | awk '{print $1}' || echo "N/A")

    # Déterminer l'action requise
    if [ "$last_activity" == "N/A" ]; then
        action="<span class='inactive'>REVIEW: No activity logged</span>"
    else
        # Calculer les jours depuis la dernière activité
        last_date=$(date -d "$last_activity" +%s 2>/dev/null || echo "0")
        now=$(date +%s)
        days_inactive=$(( (now - last_date) / 86400 ))

        if [ "$days_inactive" -gt 90 ]; then
            action="<span class='inactive'>REVIEW: Inactive $days_inactive days</span>"
        else
            action="OK"
        fi
    fi

    # Ajouter la ligne au rapport
    cat >> "$REPORT_FILE" <<EOF
        <tr>
            <td>$user</td>
            <td>$status</td>
            <td>$permissions...</td>
            <td>$key_pattern...</td>
            <td>$last_activity</td>
            <td>$action</td>
        </tr>
EOF
done

# Fermer le HTML
cat >> "$REPORT_FILE" <<'EOF'
    </table>

    <h2>Recommendations</h2>
    <ul>
        <li>Review and disable accounts inactive for >90 days</li>
        <li>Verify that permissions follow principle of least privilege</li>
        <li>Rotate passwords for all active accounts (every 90 days)</li>
        <li>Document justification for any privileged access</li>
    </ul>

    <h2>Approval</h2>
    <table>
        <tr>
            <td><strong>Reviewed by:</strong></td>
            <td>_______________________</td>
            <td><strong>Date:</strong></td>
            <td>___________</td>
        </tr>
        <tr>
            <td><strong>Approved by (CISO):</strong></td>
            <td>_______________________</td>
            <td><strong>Date:</strong></td>
            <td>___________</td>
        </tr>
    </table>
</body>
</html>
EOF

echo "✅ Access review report generated: $REPORT_FILE"
echo "   Open with: xdg-open $REPORT_FILE"

# Optionnel : Envoyer par email
# mail -s "Redis Access Review Report" -a "$REPORT_FILE" security@example.com < /dev/null
```

---

## Checklist de conformité RBAC

### Configuration initiale

```
Désactivation du compte default :
□ Compte "default" désactivé en production
□ requirepass supprimé (remplacé par ACL)
□ Pas d'accès sans authentification

Comptes administratifs :
□ Au moins 2 comptes admin (redondance)
□ Compte break-glass documenté
□ MFA activé pour accès admin (via bastion)
□ Mots de passe >16 caractères

Documentation :
□ Politique de contrôle d'accès rédigée
□ Matrice RACI des rôles et permissions
□ Procédures de provisioning documentées
□ Procédures de deprovisioning documentées
□ Workflow d'approbation formalisé
```

### Principe du moindre privilège

```
□ Aucun compte avec +@all sauf superadmin break-glass
□ Permissions limitées au strict nécessaire
□ Namespaces clé utilisés (~pattern)
□ Commandes dangereuses explicitement bloquées
□ Catégories @dangerous interdites par défaut
□ KEYS * interdit (performance + sécurité)
□ FLUSHDB/FLUSHALL interdits pour non-admin
```

### Gestion des mots de passe

```
□ Longueur minimum 12 caractères (16+ recommandé)
□ Complexité : maj, min, chiffres, symboles
□ Pas de mots du dictionnaire
□ Rotation tous les 90 jours (PCI DSS)
□ Pas de réutilisation (4 derniers)
□ Stockage sécurisé (Vault, Secrets Manager)
□ Transmission sécurisée (chiffré, pas email)
□ Hash SHA-256 dans users.acl
```

### Ségrégation et revue

```
Ségrégation des duties :
□ Dev vs Prod séparés
□ Création vs Approbation séparés
□ Ops vs Audit séparés

Revue périodique :
□ Revue trimestrielle des comptes privilégiés
□ Revue annuelle de tous les comptes
□ Identification des inactifs (>90j)
□ Vérification alignement rôle/personne
□ Rapport signé par CISO
```

### Provisioning/Deprovisioning

```
Provisioning :
□ Formulaire de demande standardisé
□ Justification métier obligatoire
□ Approbation formelle (manager + sécurité)
□ Délai de création < 24h
□ Distribution sécurisée des credentials
□ Premier accès validé et logué

Deprovisioning :
□ Révocation < 24h après notification
□ Connexions actives fermées
□ Archivage de la configuration
□ Audit post-révocation
□ Suppression définitive après 30j
```

### Audit et conformité

```
□ Tous les accès loggés (voir section 17.3)
□ Échecs d'authentification loggés
□ Modifications ACL loggées
□ Logs conservés 12 mois (PCI DSS)
□ Revue quotidienne des logs sécurité
□ Alertes sur tentatives suspectes
□ Rapport mensuel des accès
□ Tests annuels de pénétration
```

---

## Conclusion

La gestion des accès et des permissions est un pilier critique de la conformité Redis. Cette section a couvert :

- ✅ **Cadre réglementaire** exhaustif (RGPD, PCI DSS, HIPAA, SOC 2, ISO 27001)
- ✅ **Système ACL Redis 6+** : Syntaxe complète, catégories, exemples
- ✅ **Principe du moindre privilège** : Méthodologie d'application
- ✅ **Rôles prédéfinis** : 6 rôles types documentés
- ✅ **Cycle de vie complet** : Provisioning et deprovisioning automatisés
- ✅ **Scripts opérationnels** : Bash complets et prêts pour production
- ✅ **Revue périodique** : Procédures et générateur de rapport HTML
- ✅ **Checklists de conformité** : 80+ points de contrôle

**Points critiques à retenir :**
1. Redis < 6.0 est NON-CONFORME (un seul password)
2. Redis 6+ ACL est OBLIGATOIRE pour la conformité
3. Chaque utilisateur/application DOIT avoir un compte unique
4. Le compte "default" DOIT être désactivé en production
5. Principe du moindre privilège est NON-NÉGOCIABLE
6. Revue périodique OBLIGATOIRE (trimestrielle pour privilégiés)
7. Provisioning/Deprovisioning doivent être formels et documentés
8. MFA requis pour accès administratif (PCI DSS, HIPAA)

**Prochaines étapes :**
- Migrer vers Redis 6+ si version antérieure
- Implémenter les ACL selon les exemples fournis
- Désactiver le compte "default"
- Déployer les scripts de provisioning/deprovisioning
- Établir le calendrier de revue périodique
- Former les équipes aux procédures
- Planifier l'audit annuel

⏭️ [Politique de rétention des données](/17-gouvernance-conformite/05-politique-retention-donnees.md)
