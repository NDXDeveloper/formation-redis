🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.3 - Authentification et gestion des utilisateurs

## Introduction

L'authentification et la gestion des utilisateurs constituent la **première ligne de défense** de votre infrastructure Redis. Une stratégie d'authentification robuste combine :

- 🔐 **Authentification forte** - Mots de passe complexes, multi-facteurs si possible
- 👥 **Gestion du cycle de vie** - Provisioning, rotation, révocation
- 📊 **Audit et traçabilité** - Qui a fait quoi et quand
- 🔄 **Automatisation** - Réduire l'erreur humaine
- 🛡️ **Défense en profondeur** - Plusieurs couches de sécurité

> **⚠️ Redis par défaut n'a PAS d'authentification activée !**
> C'est votre responsabilité de configurer l'authentification avant la mise en production.

---

## Architecture d'authentification multi-niveaux

```
┌─────────────────────────────────────────────────────────────────┐
│                     NIVEAU 1 : RÉSEAU                           │
│  • Firewall / Security Groups                                   │
│  • VPC / VLAN isolation                                         │
│  • IP Whitelisting                                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     NIVEAU 2 : TLS/SSL                          │
│  • Chiffrement des communications                               │
│  • Validation des certificats                                   │
│  • Mutual TLS (optionnel)                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  NIVEAU 3 : AUTHENTIFICATION                    │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  Option A: requirepass (legacy)                      │       │
│  │  • Un seul mot de passe global                       │       │
│  │  • Tous les utilisateurs = même identité             │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  Option B: ACLs (Redis 6.0+) ✅ RECOMMANDÉ           │       │
│  │  • Multi-utilisateurs                                │       │
│  │  • Passwords individuels                             │       │
│  │  • Traçabilité par utilisateur                       │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  Option C: ACLs + Vault (enterprise) 🏆              │       │
│  │  • Gestion centralisée des secrets                   │       │
│  │  • Rotation automatique                              │       │
│  │  • Audit trail complet                               │       │
│  └──────────────────────────────────────────────────────┘       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                   NIVEAU 4 : AUTORISATION                       │
│  • Permissions granulaires (ACLs)                               │
│  • Restriction par commande                                     │
│  • Restriction par pattern de clés                              │
│  • Restriction Pub/Sub                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Stratégies d'authentification

### 1. requirepass - Legacy (avant Redis 6.0)

#### Configuration

```conf
# redis.conf - Authentication legacy

# Mot de passe global
requirepass "VotreMotDePasseTresComplexe2024!@#$"

# Protection mode (doit rester activé)
protected-mode yes

# Bind sur interface privée
bind 10.0.1.50
```

#### Limitations critiques

```
❌ Un seul mot de passe pour tous les utilisateurs
❌ Pas de traçabilité individuelle
❌ Impossible de révoquer l'accès d'un seul utilisateur
❌ Rotation = impact sur tous les clients
❌ Pas de permissions granulaires
```

#### Quand l'utiliser ?

```
✅ Redis < 6.0 (pas d'autre choix)
✅ Dev/staging simple
✅ Migration progressive vers ACLs
❌ JAMAIS en production moderne (Redis 6.0+)
```

### 2. ACLs natives - Moderne (Redis 6.0+)

#### Configuration de base

```conf
# redis.conf - ACLs natives

# Charger ACLs depuis fichier
aclfile /etc/redis/users.acl

# Désactiver requirepass (incompatible avec ACLs multi-users)
# requirepass ""

# Protected mode
protected-mode yes
bind 10.0.1.50
```

#### Avantages

```
✅ Multi-utilisateurs avec identités séparées
✅ Traçabilité par utilisateur (ACL LOG)
✅ Permissions granulaires
✅ Rotation sans impact global
✅ Révocation immédiate
✅ Principe du moindre privilège
```

### 3. ACLs + HashiCorp Vault - Enterprise

#### Architecture

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ Application  │         │   Vault      │         │    Redis     │
│              │         │  (Secrets)   │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ 1. Request credentials │                        │
       ├───────────────────────>│                        │
       │                        │                        │
       │ 2. Generate dynamic    │                        │
       │    user + password     │                        │
       │<───────────────────────┤                        │
       │                        │                        │
       │ 3. Create user in Redis│                        │
       │                        ├───────────────────────>│
       │                        │   ACL SETUSER          │
       │                        │                        │
       │ 4. Connect with        │                        │
       │    credentials         │                        │
       ├────────────────────────────────────────────────>│
       │                        │                        │
       │ 5. After TTL, revoke   │                        │
       │                        ├───────────────────────>│
       │                        │   ACL DELUSER          │
```

#### Avantages Vault

```
✅ Credentials dynamiques (générés à la demande)
✅ TTL automatique (expiration)
✅ Révocation automatique
✅ Audit trail complet
✅ Rotation automatique
✅ Pas de secrets en clair dans configs
✅ Gestion centralisée multi-environnements
```

---

## Politiques de mots de passe

### Standards de complexité

#### Politique minimale (NIST)

```
Exigences minimales:
├── Longueur : Minimum 12 caractères (16+ recommandé)
├── Complexité :
│   ├── Au moins 1 majuscule (A-Z)
│   ├── Au moins 1 minuscule (a-z)
│   ├── Au moins 1 chiffre (0-9)
│   └── Au moins 1 caractère spécial (!@#$%^&*())
├── Interdictions :
│   ├── Pas de mots du dictionnaire
│   ├── Pas de patterns évidents (123456, qwerty)
│   ├── Pas de répétitions (aaaa, 1111)
│   └── Pas d'informations personnelles
└── Historique :
    └── Ne pas réutiliser les 5 derniers mots de passe
```

#### Politique renforcée (production critique)

```
Exigences renforcées:
├── Longueur : Minimum 20 caractères
├── Complexité : Tous les types de caractères
├── Entropie : Minimum 80 bits
├── Génération : Générateur cryptographique (pas de mots)
├── Stockage : Hash SHA256 uniquement
└── Rotation : Tous les 90 jours maximum
```

### Génération de mots de passe sécurisés

```bash
#!/bin/bash
# generate-secure-password.sh

# Méthode 1: OpenSSL (cryptographiquement sûr)
openssl rand -base64 32

# Méthode 2: urandom
tr -dc 'A-Za-z0-9!@#$%^&*()_+=' < /dev/urandom | head -c 32

# Méthode 3: pwgen (si installé)
pwgen -s -y -n 32 1

# Méthode 4: Python
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# Méthode 5: Redis ACL GENPASS (Redis 6.2+)
redis-cli ACL GENPASS 32

# Exemple de sortie:
# Q8n$Kp2@mL9#Xv4&Rt7!Wz3^Jh6*Fy1
```

### Validation de la force d'un mot de passe

```python
#!/usr/bin/env python3
# validate_password_strength.py

import re
import math

def calculate_entropy(password):
    """Calcule l'entropie du mot de passe"""
    charset_size = 0

    if re.search(r'[a-z]', password):
        charset_size += 26  # Lowercase
    if re.search(r'[A-Z]', password):
        charset_size += 26  # Uppercase
    if re.search(r'[0-9]', password):
        charset_size += 10  # Digits
    if re.search(r'[^a-zA-Z0-9]', password):
        charset_size += 32  # Special chars (estimation)

    if charset_size == 0:
        return 0

    entropy = len(password) * math.log2(charset_size)
    return entropy

def validate_password(password):
    """Valide un mot de passe selon politique"""
    errors = []

    # Longueur
    if len(password) < 16:
        errors.append(f"❌ Trop court: {len(password)} chars (minimum 16)")

    # Majuscules
    if not re.search(r'[A-Z]', password):
        errors.append("❌ Manque majuscules")

    # Minuscules
    if not re.search(r'[a-z]', password):
        errors.append("❌ Manque minuscules")

    # Chiffres
    if not re.search(r'[0-9]', password):
        errors.append("❌ Manque chiffres")

    # Caractères spéciaux
    if not re.search(r'[^a-zA-Z0-9]', password):
        errors.append("❌ Manque caractères spéciaux")

    # Patterns évidents
    if re.search(r'(.)\1{3,}', password):
        errors.append("❌ Répétitions détectées (aaaa, 1111)")

    if re.search(r'(012|123|234|345|456|567|678|789|abc|bcd)', password.lower()):
        errors.append("❌ Séquence évidente détectée")

    # Entropie
    entropy = calculate_entropy(password)
    if entropy < 60:
        errors.append(f"❌ Entropie trop faible: {entropy:.1f} bits (minimum 60)")

    # Résultats
    if errors:
        print("🔴 Mot de passe FAIBLE:")
        for error in errors:
            print(f"  {error}")
        print(f"  Entropie: {entropy:.1f} bits")
        return False
    else:
        print("✅ Mot de passe FORT:")
        print(f"  Longueur: {len(password)} caractères")
        print(f"  Entropie: {entropy:.1f} bits")
        return True

if __name__ == '__main__':
    import sys

    if len(sys.argv) != 2:
        print("Usage: python3 validate_password_strength.py 'YourPassword'")
        sys.exit(1)

    password = sys.argv[1]
    validate_password(password)
```

### Stockage sécurisé des mots de passe

```bash
# ============================================================================
# BONNES PRATIQUES DE STOCKAGE
# ============================================================================

# ❌ JAMAIS EN CLAIR
user alice on >PlainTextPassword123!  # DANGEREUX

# ✅ TOUJOURS HASHER
# 1. Générer hash SHA256
echo -n "PlainTextPassword123!" | sha256sum
# Output: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92

# 2. Utiliser dans ACL
user alice on #8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92 ~* +@all

# ============================================================================
# STOCKAGE FICHIERS
# ============================================================================

# users.acl - Permissions strictes
chmod 600 /etc/redis/users.acl
chown redis:redis /etc/redis/users.acl

# Backup chiffré
gpg --symmetric --cipher-algo AES256 users.acl
# Produit: users.acl.gpg

# Décryptage
gpg --decrypt users.acl.gpg > users.acl
```

---

## Gestion du cycle de vie des utilisateurs

### 1. Provisioning (Création)

#### Processus manuel

```bash
#!/bin/bash
# provision-user.sh

USERNAME=$1
ROLE=$2

if [ -z "$USERNAME" ] || [ -z "$ROLE" ]; then
    echo "Usage: $0 <username> <role>"
    exit 1
fi

# Générer mot de passe fort
PASSWORD=$(redis-cli ACL GENPASS 32)

# Créer utilisateur selon rôle
case $ROLE in
    "app")
        redis-cli ACL SETUSER "$USERNAME" on ">$PASSWORD" ~app:* +@read +@write -@dangerous
        ;;
    "cache")
        redis-cli ACL SETUSER "$USERNAME" on ">$PASSWORD" ~cache:* +@read +@write +@string -@dangerous
        ;;
    "monitoring")
        redis-cli ACL SETUSER "$USERNAME" on ">$PASSWORD" ~* +@read +info +ping -@write
        ;;
    "admin")
        redis-cli ACL SETUSER "$USERNAME" on ">$PASSWORD" ~* +@all
        ;;
    *)
        echo "❌ Rôle inconnu: $ROLE"
        exit 1
        ;;
esac

# Sauvegarder
redis-cli ACL SAVE

# Afficher credentials (UNE SEULE FOIS)
echo "✅ Utilisateur créé:"
echo "Username: $USERNAME"
echo "Password: $PASSWORD"
echo "Role: $ROLE"
echo ""
echo "⚠️  Sauvegarder ces credentials de manière sécurisée!"
echo "⚠️  Ce mot de passe ne sera plus affiché."

# Optionnel: Envoyer vers Vault ou secret manager
# vault kv put secret/redis/$USERNAME password=$PASSWORD role=$ROLE
```

#### Provisioning automatisé avec Terraform

```hcl
# terraform/redis-users.tf

# Provider Redis (utilise redis-cli sous le capot)
terraform {
  required_providers {
    redis = {
      source  = "terraform-providers/redis"
      version = "~> 1.0"
    }
  }
}

provider "redis" {
  host     = "redis.example.com"
  port     = 6379
  username = "admin"
  password = var.redis_admin_password
}

# Définir utilisateurs
resource "redis_acl" "app_backend" {
  username = "app_backend"
  enabled  = true

  passwords = [
    sha256(random_password.app_backend.result)
  ]

  key_patterns = ["app:*", "cache:*"]

  commands = [
    "+@read",
    "+@write",
    "+@hash",
    "+@string",
    "-@dangerous",
    "-@admin"
  ]
}

resource "random_password" "app_backend" {
  length  = 32
  special = true
}

# Stocker password dans Vault
resource "vault_generic_secret" "app_backend" {
  path = "secret/redis/app_backend"

  data_json = jsonencode({
    username = "app_backend"
    password = random_password.app_backend.result
  })
}

# Output (temporaire, pour premier déploiement)
output "app_backend_password" {
  value     = random_password.app_backend.result
  sensitive = true
}
```

### 2. Modification (Update)

```bash
#!/bin/bash
# update-user-permissions.sh

USERNAME=$1
ACTION=$2  # add ou remove
PERMISSION=$3

if [ -z "$USERNAME" ] || [ -z "$ACTION" ] || [ -z "$PERMISSION" ]; then
    echo "Usage: $0 <username> <add|remove> <permission>"
    exit 1
fi

# Backup avant modification
redis-cli ACL SAVE
cp /etc/redis/users.acl /backup/users.acl.$(date +%Y%m%d_%H%M%S)

# Modifier permissions
case $ACTION in
    "add")
        redis-cli ACL SETUSER "$USERNAME" "+$PERMISSION"
        ;;
    "remove")
        redis-cli ACL SETUSER "$USERNAME" "-$PERMISSION"
        ;;
    *)
        echo "❌ Action invalide: $ACTION"
        exit 1
        ;;
esac

# Sauvegarder
redis-cli ACL SAVE

# Vérifier
echo "✅ Permissions mises à jour:"
redis-cli ACL GETUSER "$USERNAME"

# Audit log
echo "$(date) - User $USERNAME - $ACTION $PERMISSION - by $(whoami)" >> /var/log/redis/acl-changes.log
```

### 3. Désactivation temporaire

```bash
#!/bin/bash
# disable-user.sh

USERNAME=$1
REASON=$2

if [ -z "$USERNAME" ]; then
    echo "Usage: $0 <username> [reason]"
    exit 1
fi

# Vérifier si utilisateur existe
if ! redis-cli ACL GETUSER "$USERNAME" > /dev/null 2>&1; then
    echo "❌ Utilisateur $USERNAME n'existe pas"
    exit 1
fi

# Désactiver
redis-cli ACL SETUSER "$USERNAME" off

# Killer connexions actives
redis-cli CLIENT KILL USER "$USERNAME"

# Sauvegarder
redis-cli ACL SAVE

echo "✅ Utilisateur $USERNAME désactivé"
echo "Reason: ${REASON:-Not specified}"

# Audit
echo "$(date) - User $USERNAME DISABLED - Reason: ${REASON:-Not specified} - by $(whoami)" >> /var/log/redis/acl-changes.log

# Alerter équipe
# send-alert "Redis user $USERNAME disabled: ${REASON:-Not specified}"
```

### 4. Révocation (Suppression)

```bash
#!/bin/bash
# revoke-user.sh

USERNAME=$1
CONFIRM=$2

if [ -z "$USERNAME" ]; then
    echo "Usage: $0 <username> CONFIRM"
    exit 1
fi

if [ "$CONFIRM" != "CONFIRM" ]; then
    echo "⚠️  Cette action est IRRÉVERSIBLE!"
    echo "Pour confirmer, exécutez:"
    echo "  $0 $USERNAME CONFIRM"
    exit 1
fi

# Backup avant suppression
redis-cli ACL SAVE
cp /etc/redis/users.acl /backup/users.acl.before-delete-$USERNAME.$(date +%Y%m%d_%H%M%S)

# Vérifier utilisation
ACTIVE_CONNECTIONS=$(redis-cli CLIENT LIST | grep "user=$USERNAME" | wc -l)
if [ $ACTIVE_CONNECTIONS -gt 0 ]; then
    echo "⚠️  WARNING: $ACTIVE_CONNECTIONS connexion(s) active(s)"
    echo "Continuer quand même? (yes/no)"
    read -r response
    if [ "$response" != "yes" ]; then
        echo "Annulé"
        exit 0
    fi
fi

# Killer connexions
redis-cli CLIENT KILL USER "$USERNAME"

# Supprimer utilisateur
redis-cli ACL DELUSER "$USERNAME"

# Sauvegarder
redis-cli ACL SAVE

echo "✅ Utilisateur $USERNAME supprimé"

# Audit
echo "$(date) - User $USERNAME DELETED - by $(whoami)" >> /var/log/redis/acl-changes.log

# Notification
# send-alert "Redis user $USERNAME has been deleted"
```

---

## Rotation des credentials

### Stratégie de rotation

```
┌─────────────────────────────────────────────────────────────┐
│              ROTATION DES MOTS DE PASSE                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Fréquence recommandée:                                     │
│  ├── Production critique : 60 jours                         │
│  ├── Production standard : 90 jours                         │
│  └── Staging/Dev : 180 jours                                │
│                                                             │
│  Déclencheurs de rotation immédiate:                        │
│  ├── Départ d'un employé                                    │
│  ├── Suspicion de compromission                             │
│  ├── Après incident de sécurité                             │
│  └── Logs montrant accès non autorisé                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Rotation manuelle sans downtime

```bash
#!/bin/bash
# rotate-password-zero-downtime.sh

USERNAME=$1

if [ -z "$USERNAME" ]; then
    echo "Usage: $0 <username>"
    exit 1
fi

echo "=== ROTATION PASSWORD: $USERNAME ==="

# 1. Générer nouveau mot de passe
NEW_PASSWORD=$(redis-cli ACL GENPASS 32)
echo "1. Nouveau mot de passe généré: $NEW_PASSWORD"

# 2. Ajouter nouveau password (SANS supprimer l'ancien)
# Redis permet plusieurs passwords actifs simultanément
redis-cli ACL SETUSER "$USERNAME" ">$NEW_PASSWORD"
redis-cli ACL SAVE

echo "2. ✅ Nouveau password ajouté (ancien toujours valide)"

# 3. Afficher les deux passwords (pour mise à jour apps)
echo ""
echo "3. Configuration actuelle:"
redis-cli ACL GETUSER "$USERNAME" | grep "passwords"

echo ""
echo "4. ⚠️  ÉTAPES SUIVANTES:"
echo "   a) Mettre à jour les applications avec le NOUVEAU password"
echo "   b) Attendre 24-48h (fenêtre de migration)"
echo "   c) Vérifier qu'aucune connexion n'utilise l'ancien password"
echo "   d) Exécuter: $0 $USERNAME cleanup"
echo ""
echo "5. Nouveau password: $NEW_PASSWORD"
echo "   (Sauvegarder de manière sécurisée)"

# Si argument "cleanup" fourni
if [ "$2" == "cleanup" ]; then
    echo ""
    echo "=== CLEANUP: Suppression ancien password ==="

    # Obtenir hash de l'ancien password (premier dans la liste)
    # Note: Cette partie nécessite parsing du output ACL GETUSER
    # En production, utiliser script plus robuste ou Vault

    echo "⚠️  Cette opération va supprimer l'ANCIEN password"
    echo "Confirmer que toutes les apps utilisent le nouveau? (yes/no)"
    read -r confirm

    if [ "$confirm" == "yes" ]; then
        # Reset passwords puis ajouter seulement le nouveau
        redis-cli ACL SETUSER "$USERNAME" resetpass ">$NEW_PASSWORD"
        redis-cli ACL SAVE
        echo "✅ Ancien password supprimé"
    else
        echo "Annulé"
    fi
fi
```

### Rotation automatisée avec Vault

```python
#!/usr/bin/env python3
# vault-redis-rotation.py

import hvac
import redis
import hashlib
import secrets
import schedule
import time
from datetime import datetime

# Configuration
VAULT_ADDR = "https://vault.example.com"
VAULT_TOKEN = "s.xxxxxxxxxxxxx"
REDIS_HOST = "redis.example.com"
REDIS_PORT = 6379
REDIS_ADMIN_USER = "admin"
REDIS_ADMIN_PASS = "AdminPassword"

# Clients
vault_client = hvac.Client(url=VAULT_ADDR, token=VAULT_TOKEN)
redis_client = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    username=REDIS_ADMIN_USER,
    password=REDIS_ADMIN_PASS,
    decode_responses=True
)

def generate_secure_password(length=32):
    """Génère un mot de passe cryptographiquement sûr"""
    return secrets.token_urlsafe(length)

def hash_password(password):
    """Hash password en SHA256 pour Redis"""
    return hashlib.sha256(password.encode()).hexdigest()

def rotate_user_password(username):
    """Rotate password pour un utilisateur"""

    print(f"[{datetime.now()}] Rotating password for: {username}")

    try:
        # 1. Générer nouveau password
        new_password = generate_secure_password()

        # 2. Mettre à jour dans Redis (ajoute, ne supprime pas l'ancien)
        redis_client.execute_command(
            'ACL', 'SETUSER', username, f'>{new_password}'
        )
        redis_client.execute_command('ACL', 'SAVE')

        # 3. Stocker dans Vault avec metadata
        vault_client.secrets.kv.v2.create_or_update_secret(
            path=f'redis/{username}',
            secret={
                'password': new_password,
                'rotated_at': datetime.now().isoformat(),
                'rotated_by': 'automated-rotation',
                'version': vault_client.secrets.kv.v2.read_secret_version(
                    path=f'redis/{username}'
                ).get('data', {}).get('metadata', {}).get('version', 0) + 1
            }
        )

        print(f"✅ Password rotated for {username}")
        print(f"   New password stored in Vault: redis/{username}")

        # 4. Notification (optionnel)
        # send_notification(f"Password rotated for Redis user: {username}")

        return True

    except Exception as e:
        print(f"❌ Error rotating password for {username}: {e}")
        return False

def cleanup_old_passwords(username, grace_period_hours=48):
    """Nettoie les anciens passwords après période de grâce"""

    print(f"[{datetime.now()}] Cleanup old passwords for: {username}")

    try:
        # Récupérer nouveau password depuis Vault
        secret = vault_client.secrets.kv.v2.read_secret_version(
            path=f'redis/{username}'
        )
        new_password = secret['data']['data']['password']

        # Reset puis ajouter seulement le nouveau
        redis_client.execute_command(
            'ACL', 'SETUSER', username, 'resetpass', f'>{new_password}'
        )
        redis_client.execute_command('ACL', 'SAVE')

        print(f"✅ Old passwords cleaned up for {username}")

    except Exception as e:
        print(f"❌ Error cleaning up passwords for {username}: {e}")

def rotate_all_users():
    """Rotate tous les utilisateurs applicatifs"""

    # Liste des utilisateurs à rotate (exclure admin)
    users_to_rotate = [
        'app_backend',
        'cache_manager',
        'session_store',
        'queue_worker',
        'monitoring'
    ]

    for username in users_to_rotate:
        rotate_user_password(username)
        time.sleep(2)  # Éviter la surcharge

# Scheduler
schedule.every(90).days.do(rotate_all_users)  # Rotation tous les 90 jours
schedule.every().monday.at("02:00").do(rotate_all_users)  # Backup: chaque lundi 2h

if __name__ == '__main__':
    print("=== Redis Password Rotation Service ===")
    print("Rotation frequency: Every 90 days or Monday 02:00")

    # Optionnel: Rotation immédiate au démarrage
    # rotate_all_users()

    # Loop
    while True:
        schedule.run_pending()
        time.sleep(3600)  # Check every hour
```

---

## Audit et traçabilité

### 1. Audit des authentifications

```bash
#!/bin/bash
# audit-redis-auth.sh

echo "=== REDIS AUTHENTICATION AUDIT ==="
echo "Date: $(date)"
echo ""

# 1. Historique des échecs d'authentification
echo "1. Failed authentication attempts (last 100):"
redis-cli ACL LOG 100 | grep -A 5 "reason: auth" | head -50

# 2. Statistiques par utilisateur
echo ""
echo "2. Authentication attempts by user:"
redis-cli ACL LOG 100 | grep "username:" | sort | uniq -c | sort -rn

# 3. Patterns d'attaque
echo ""
echo "3. Suspicious patterns:"
redis-cli ACL LOG 100 | grep -E "(reason: auth|reason: command)" | \
    awk '{print $4}' | sort | uniq -c | sort -rn | head -10

# 4. Connexions actives par utilisateur
echo ""
echo "4. Active connections by user:"
redis-cli CLIENT LIST | grep -oP 'user=\K[^ ]+' | sort | uniq -c | sort -rn

# 5. Utilisateurs jamais utilisés
echo ""
echo "5. Unused users (potential cleanup candidates):"
DEFINED_USERS=$(redis-cli ACL LIST | awk '{print $2}')
ACTIVE_USERS=$(redis-cli CLIENT LIST | grep -oP 'user=\K[^ ]+' | sort -u)

for user in $DEFINED_USERS; do
    if ! echo "$ACTIVE_USERS" | grep -q "^$user$"; then
        echo "  - $user (no active connections)"
    fi
done

# 6. Utilisateurs avec permissions élevées
echo ""
echo "6. Users with elevated permissions:"
redis-cli ACL LIST | grep -E "(\+@all|allcommands)"

# 7. Génération rapport
REPORT_FILE="/var/log/redis/auth-audit-$(date +%Y%m%d).log"
echo ""
echo "Full report saved to: $REPORT_FILE"
{
    echo "=== REDIS AUTH AUDIT ==="
    echo "Date: $(date)"
    echo ""
    redis-cli ACL LOG 1000
} > "$REPORT_FILE"
```

### 2. Monitoring en temps réel

```python
#!/usr/bin/env python3
# realtime-auth-monitor.py

import redis
import time
from datetime import datetime
from collections import defaultdict

# Configuration
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_USER = 'monitoring'
REDIS_PASS = 'MonitoringP@ss2024!'

# Seuils d'alerte
FAILED_AUTH_THRESHOLD = 5  # 5 échecs en 1 minute
UNUSUAL_USER_THRESHOLD = 3  # 3 users différents depuis même IP

# Connexion
r = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    username=REDIS_USER,
    password=REDIS_PASS,
    decode_responses=True
)

# État
failed_auths = defaultdict(int)
user_ips = defaultdict(set)

def check_failed_auths():
    """Surveille les échecs d'authentification"""

    acl_log = r.execute_command('ACL', 'LOG', '100')

    recent_failures = []
    for entry in acl_log:
        if entry['reason'] == 'auth':
            age = float(entry['age-seconds'])
            if age < 60:  # Dernière minute
                username = entry.get('username', 'unknown')
                client_info = entry.get('client-info', '')
                recent_failures.append((username, client_info))

    # Compter par utilisateur
    user_failures = defaultdict(int)
    for username, _ in recent_failures:
        user_failures[username] += 1

    # Alertes
    for username, count in user_failures.items():
        if count >= FAILED_AUTH_THRESHOLD:
            alert(
                f"🚨 BRUTE FORCE DETECTED: {count} failed auth attempts "
                f"for user '{username}' in last minute"
            )

            # Action: Bloquer temporairement?
            # r.execute_command('ACL', 'SETUSER', username, 'off')

def check_unusual_patterns():
    """Détecte patterns inhabituels"""

    # Obtenir connexions actives
    clients = r.execute_command('CLIENT', 'LIST')

    # Parser
    current_connections = []
    for line in clients.split('\n'):
        if not line:
            continue

        fields = {}
        for field in line.split():
            if '=' in field:
                key, value = field.split('=', 1)
                fields[key] = value

        if 'user' in fields and 'addr' in fields:
            user = fields['user']
            ip = fields['addr'].split(':')[0]
            current_connections.append((user, ip))

    # Analyser patterns
    ip_users = defaultdict(set)
    for user, ip in current_connections:
        ip_users[ip].add(user)

    # Alertes: Plusieurs users depuis même IP
    for ip, users in ip_users.items():
        if len(users) >= UNUSUAL_USER_THRESHOLD:
            alert(
                f"⚠️  UNUSUAL PATTERN: {len(users)} different users "
                f"from same IP: {ip} - Users: {', '.join(users)}"
            )

def check_new_users():
    """Détecte nouveaux utilisateurs créés"""

    global known_users

    current_users = set(r.execute_command('ACL', 'LIST'))

    if 'known_users' in globals():
        new_users = current_users - known_users
        if new_users:
            alert(f"ℹ️  NEW USER(S) CREATED: {', '.join(new_users)}")

    known_users = current_users

def alert(message):
    """Envoie une alerte"""
    timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    print(f"[{timestamp}] {message}")

    # Écrire dans log
    with open('/var/log/redis/security-alerts.log', 'a') as f:
        f.write(f"[{timestamp}] {message}\n")

    # Envoyer notification (Slack, PagerDuty, etc.)
    # send_to_slack(message)
    # send_to_pagerduty(message)

def main():
    print("=== Redis Authentication Monitor ===")
    print("Monitoring for security events...")
    print("")

    while True:
        try:
            check_failed_auths()
            check_unusual_patterns()
            check_new_users()

            time.sleep(30)  # Check every 30 seconds

        except KeyboardInterrupt:
            print("\n\nMonitoring stopped.")
            break
        except Exception as e:
            print(f"Error: {e}")
            time.sleep(60)

if __name__ == '__main__':
    main()
```

### 3. Logs structurés pour SIEM

```python
#!/usr/bin/env python3
# redis-auth-to-siem.py

import redis
import json
import time
from datetime import datetime
import socket

# Configuration
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_USER = 'monitoring'
REDIS_PASS = 'MonitoringP@ss2024!'
SIEM_HOST = 'siem.example.com'
SIEM_PORT = 514  # Syslog

# Connexion Redis
r = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    username=REDIS_USER,
    password=REDIS_PASS,
    decode_responses=True
)

# Socket Syslog
syslog = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

def send_to_siem(event):
    """Envoie événement au SIEM en format JSON"""

    # Format Syslog avec JSON payload
    message = json.dumps(event, default=str)
    syslog_message = f"<134>{message}"  # Priority 134 = local0.info

    try:
        syslog.sendto(syslog_message.encode(), (SIEM_HOST, SIEM_PORT))
    except Exception as e:
        print(f"Error sending to SIEM: {e}")

def collect_auth_events():
    """Collecte événements d'authentification"""

    acl_log = r.execute_command('ACL', 'LOG', '50')

    for entry in acl_log:
        event = {
            'timestamp': datetime.now().isoformat(),
            'source': 'redis',
            'host': REDIS_HOST,
            'event_type': 'authentication',
            'reason': entry['reason'],
            'username': entry.get('username', 'unknown'),
            'client_info': entry.get('client-info', ''),
            'object': entry.get('object', ''),
            'age_seconds': entry['age-seconds']
        }

        # Classifier selon gravité
        if entry['reason'] == 'auth':
            event['severity'] = 'high'
            event['description'] = f"Failed authentication for user {event['username']}"
        elif entry['reason'] == 'command':
            event['severity'] = 'medium'
            event['description'] = f"Unauthorized command attempt by {event['username']}"
        else:
            event['severity'] = 'low'

        send_to_siem(event)

def collect_connection_events():
    """Collecte événements de connexion"""

    clients = r.execute_command('CLIENT', 'LIST')

    for line in clients.split('\n'):
        if not line:
            continue

        fields = {}
        for field in line.split():
            if '=' in field:
                key, value = field.split('=', 1)
                fields[key] = value

        event = {
            'timestamp': datetime.now().isoformat(),
            'source': 'redis',
            'host': REDIS_HOST,
            'event_type': 'connection',
            'severity': 'info',
            'username': fields.get('user', 'unknown'),
            'client_ip': fields.get('addr', '').split(':')[0],
            'client_port': fields.get('addr', '').split(':')[1] if ':' in fields.get('addr', '') else '',
            'connection_age': fields.get('age', ''),
            'idle_time': fields.get('idle', ''),
            'database': fields.get('db', ''),
        }

        send_to_siem(event)

def main():
    print("=== Redis to SIEM Forwarder ===")
    print(f"Forwarding to: {SIEM_HOST}:{SIEM_PORT}")
    print("")

    while True:
        try:
            collect_auth_events()
            collect_connection_events()

            time.sleep(60)  # Collect every minute

        except KeyboardInterrupt:
            print("\n\nForwarding stopped.")
            break
        except Exception as e:
            print(f"Error: {e}")
            time.sleep(60)

if __name__ == '__main__':
    main()
```

---

## Intégration avec systèmes d'identité

### 1. LDAP/Active Directory (via proxy)

Redis ne supporte pas nativement LDAP, mais vous pouvez utiliser un proxy d'authentification.

#### Architecture

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ Application  │         │  Auth Proxy  │         │    Redis     │
│              │         │   (Custom)   │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ 1. Connect with        │                        │
       │    LDAP credentials    │                        │
       ├───────────────────────>│                        │
       │                        │                        │
       │                        │ 2. Validate LDAP       │
       │                        ├───────────────────────>│
       │                        │     (LDAP Server)      │
       │                        │                        │
       │                        │ 3. Get Redis creds     │
       │                        │    from cache/Vault    │
       │                        │                        │
       │                        │ 4. Connect to Redis    │
       │                        ├───────────────────────>│
       │                        │    with Redis ACL      │
       │                        │                        │
       │ 5. Return connection   │                        │
       │<───────────────────────┤                        │
```

#### Exemple de proxy Python

```python
#!/usr/bin/env python3
# redis-ldap-auth-proxy.py

import ldap
import redis
import socket
import threading
from redis.connection import Connection

# Configuration
LDAP_SERVER = "ldap://ldap.example.com"
LDAP_BASE_DN = "dc=example,dc=com"
REDIS_HOST = "localhost"
REDIS_PORT = 6379
PROXY_PORT = 6380

class LDAPAuthProxy:
    def __init__(self):
        self.redis_pool = {}

    def authenticate_ldap(self, username, password):
        """Authentifie contre LDAP"""
        try:
            conn = ldap.initialize(LDAP_SERVER)
            conn.simple_bind_s(
                f"uid={username},{LDAP_BASE_DN}",
                password
            )
            return True
        except ldap.INVALID_CREDENTIALS:
            return False
        except Exception as e:
            print(f"LDAP error: {e}")
            return False

    def get_redis_credentials(self, username):
        """Récupère credentials Redis pour user LDAP"""
        # Mapper LDAP user → Redis user
        # En prod: Vault ou DB mapping
        redis_users = {
            'john.doe': ('app_backend', 'AppPassword123!'),
            'jane.smith': ('app_readonly', 'ReadPassword456!'),
        }
        return redis_users.get(username, (None, None))

    def handle_client(self, client_socket):
        """Handle connexion client"""
        try:
            # Recevoir credentials
            data = client_socket.recv(1024).decode()
            username, password = data.split(':', 1)

            # Authentifier LDAP
            if not self.authenticate_ldap(username, password):
                client_socket.send(b"-ERR Invalid credentials\r\n")
                return

            # Obtenir credentials Redis
            redis_user, redis_pass = self.get_redis_credentials(username)
            if not redis_user:
                client_socket.send(b"-ERR No Redis mapping\r\n")
                return

            # Créer connexion Redis
            r = redis.Redis(
                host=REDIS_HOST,
                port=REDIS_PORT,
                username=redis_user,
                password=redis_pass
            )

            # Proxy requests
            client_socket.send(b"+OK Authenticated\r\n")
            # ... Proxy logic here ...

        except Exception as e:
            print(f"Error: {e}")
        finally:
            client_socket.close()

    def start(self):
        """Démarre le proxy"""
        server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server.bind(('0.0.0.0', PROXY_PORT))
        server.listen(5)

        print(f"LDAP Auth Proxy listening on port {PROXY_PORT}")

        while True:
            client, addr = server.accept()
            thread = threading.Thread(
                target=self.handle_client,
                args=(client,)
            )
            thread.start()

if __name__ == '__main__':
    proxy = LDAPAuthProxy()
    proxy.start()
```

### 2. OAuth 2.0 / OpenID Connect (via API Gateway)

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  Frontend    │         │ API Gateway  │         │    Redis     │
│              │         │  (Kong/Tyk)  │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ 1. Request with        │                        │
       │    OAuth token         │                        │
       ├───────────────────────>│                        │
       │                        │                        │
       │                        │ 2. Validate token      │
       │                        │    with IdP            │
       │                        │                        │
       │                        │ 3. Get user claims     │
       │                        │    (roles, groups)     │
       │                        │                        │
       │                        │ 4. Map to Redis user   │
       │                        │                        │
       │                        │ 5. Connect Redis       │
       │                        ├───────────────────────>│
       │                        │                        │
```

---

## Checklist de sécurité - Authentification

### Checklist déploiement

- [ ] **Authentification activée**
  - ACLs configurées (Redis 6.0+)
  - requirepass désactivé ou synchronisé avec default user

- [ ] **Utilisateur default désactivé**
  ```bash
  redis-cli ACL GETUSER default | grep "off"
  ```

- [ ] **Mots de passe forts**
  - Minimum 16 caractères
  - Complexité respectée
  - Générés cryptographiquement
  - Hashés en SHA256 dans users.acl

- [ ] **Permissions minimales**
  - Chaque user = rôle spécifique
  - Pas de +@all sauf admins
  - Patterns de clés restrictifs

- [ ] **Rotation planifiée**
  - Fréquence définie (90 jours)
  - Process automatisé ou documentation
  - Calendrier de rotation

- [ ] **Audit activé**
  - ACL LOG surveillé
  - Logs centralisés (SIEM)
  - Alertes configurées

- [ ] **Backup sécurisé**
  - users.acl backupé quotidiennement
  - Backups chiffrés
  - Procédure de restauration testée

- [ ] **Documentation**
  - Liste des utilisateurs et rôles
  - Procédures de provisioning
  - Runbook rotation des passwords
  - Contact emergency

### Checklist audit mensuel

- [ ] **Revoir utilisateurs actifs**
  - Supprimer utilisateurs inutilisés
  - Vérifier permissions toujours nécessaires

- [ ] **Analyser ACL LOG**
  - Échecs d'authentification
  - Tentatives d'accès non autorisées
  - Patterns suspects

- [ ] **Vérifier force des passwords**
  - Re-hasher si nécessaire
  - Rotation selon calendrier

- [ ] **Test de restauration**
  - Backup users.acl restauré en staging
  - Validation fonctionnelle

- [ ] **Mise à jour documentation**
  - Nouveaux utilisateurs documentés
  - Changements de rôles

### Checklist incident

- [ ] **Isoler utilisateur compromis**
  ```bash
  redis-cli ACL SETUSER <user> off
  redis-cli CLIENT KILL USER <user>
  ```

- [ ] **Analyser activité**
  - SLOWLOG pour commandes exécutées
  - ACL LOG pour tentatives
  - CLIENT LIST pour connexions

- [ ] **Rotation immédiate**
  - Tous les passwords potentiellement compromis
  - Notification aux équipes

- [ ] **Post-mortem**
  - Documenter incident
  - Améliorer procédures
  - Renforcer ACLs si nécessaire

---

## 📚 Ressources complémentaires

### Documentation officielle

- [Redis ACL Guide](https://redis.io/docs/management/security/acl/)
- [Redis Security](https://redis.io/docs/management/security/)
- [Authentication Best Practices](https://redis.io/docs/management/security/#authentication)

### Outils recommandés

- **HashiCorp Vault** - Gestion centralisée des secrets
- **CyberArk** - Enterprise secrets management
- **1Password / LastPass** - Stockage sécurisé pour équipes
- **Terraform** - Infrastructure as Code
- **Ansible** - Automation

### Standards de sécurité

- **NIST SP 800-63B** - Digital Identity Guidelines
- **OWASP** - Password Storage Cheat Sheet
- **CIS Benchmarks** - Redis Security Configuration

---

**Section suivante :** [12.4 - Chiffrement : TLS/SSL et impact sur la performance](./04-chiffrement-tls-ssl-performance.md)

⏭️ [Chiffrement : TLS/SSL et impact sur la performance](/12-redis-production-securite/04-chiffrement-tls-ssl-performance.md)
