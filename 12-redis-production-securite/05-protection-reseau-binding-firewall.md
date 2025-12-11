🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.5 - Protection réseau : Binding, Firewall, VPC

## Introduction

La sécurité réseau est la **première barrière de défense** pour protéger Redis. Sans protection réseau adéquate, même avec authentification et TLS, Redis reste vulnérable aux attaques. Cette section couvre la défense en profondeur au niveau réseau.

> **⚠️ Fait alarmant :** Des milliers d'instances Redis sont exposées publiquement sur Internet sans protection. Les scans automatisés trouvent et compromettent ces instances en quelques minutes.

### Principe de défense en profondeur

```
┌─────────────────────────────────────────────────────────────────┐
│                    COUCHE 1 : INTERNET                          │
│  Exposition publique = DANGER                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │ BLOCK
┌────────────────────────────▼────────────────────────────────────┐
│              COUCHE 2 : FIREWALL PÉRIMÈTRE                      │
│  • Cloud Security Groups (AWS, Azure, GCP)                      │
│  • Firewall hardware                                            │
│  • WAF (Web Application Firewall)                               │
└────────────────────────────┬────────────────────────────────────┘
                             │ WHITELIST
┌────────────────────────────▼────────────────────────────────────┐
│                COUCHE 3 : VPC / VLAN                            │
│  • Isolation réseau privé                                       │
│  • Segmentation par subnet                                      │
│  • Pas de route vers Internet                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │ ROUTE
┌────────────────────────────▼────────────────────────────────────┐
│           COUCHE 4 : FIREWALL LOCAL (iptables)                  │
│  • Rules au niveau OS                                           │
│  • Whitelist IP sources                                         │
│  • Drop tout le reste                                           │
└────────────────────────────┬────────────────────────────────────┘
                             │ ALLOW
┌────────────────────────────▼────────────────────────────────────┐
│            COUCHE 5 : BIND REDIS (redis.conf)                   │
│  • Écoute sur IP privée uniquement                              │
│  • JAMAIS 0.0.0.0 en production                                 │
│  • JAMAIS interface publique                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │ LISTEN
┌────────────────────────────▼────────────────────────────────────┐
│          COUCHE 6 : PROTECTED MODE (redis.conf)                 │
│  • Protection si pas d'authentification                         │
│  • Refuse connexions non-localhost si pas de password           │
└────────────────────────────┬────────────────────────────────────┘
                             │ CHECK
┌────────────────────────────▼────────────────────────────────────┐
│         COUCHE 7 : AUTHENTIFICATION (ACLs)                      │
│  • Validation credentials                                       │
│  • Permissions granulaires                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Configuration Bind (redis.conf)

### 1. Comprendre la directive bind

```conf
# ============================================================================
# DIRECTIVE BIND - CRITIQUE POUR LA SÉCURITÉ
# ============================================================================

# bind définit sur quelles interfaces réseau Redis écoute
# IMPORTANT: Ce n'est PAS un mécanisme de sécurité en soi,
#            c'est une première couche de défense

# ❌ DANGEREUX - Écoute sur TOUTES les interfaces
bind 0.0.0.0
# Conséquence: Redis accessible depuis Internet si pas de firewall!

# ❌ DANGEREUX - Écoute sur toutes interfaces IPv4 et IPv6
bind 0.0.0.0 ::
# Même problème, pire avec IPv6

# ✅ SÉCURISÉ - Localhost uniquement (dev/test)
bind 127.0.0.1
# Redis accessible uniquement depuis la machine locale

# ✅ SÉCURISÉ - IP privée spécifique (production)
bind 10.0.1.50
# Redis accessible uniquement via cette IP privée

# ✅ SÉCURISÉ - Plusieurs interfaces privées
bind 127.0.0.1 10.0.1.50
# Redis accessible en local ET via IP privée

# ✅ SÉCURISÉ - IPv4 et IPv6 privées
bind 127.0.0.1 ::1 10.0.1.50 fd00::1
# Support IPv4 et IPv6, uniquement IPs privées
```

### 2. Configurations bind par environnement

```conf
# ============================================================================
# DÉVELOPPEMENT LOCAL
# ============================================================================
# Environnement: Poste développeur
# Sécurité: Faible (localhost uniquement)

bind 127.0.0.1
port 6379
protected-mode yes

# ============================================================================
```

```conf
# ============================================================================
# STAGING / PREPROD
# ============================================================================
# Environnement: Serveur dans VPC privé
# Sécurité: Moyenne (IP privée + firewall)

# Bind sur IP privée du subnet
bind 10.0.1.50

# Port standard ou custom
port 6379

# Protected mode (backup si bind mal configuré)
protected-mode yes

# Authentification OBLIGATOIRE
aclfile /etc/redis/users.acl

# TLS recommandé
tls-port 6380
port 0  # Désactiver port non-TLS

# ============================================================================
```

```conf
# ============================================================================
# PRODUCTION
# ============================================================================
# Environnement: Production dans VPC hautement sécurisé
# Sécurité: Maximale (IP privée + firewall + TLS + ACLs)

# Bind UNIQUEMENT sur IP privée interne
bind 10.0.1.50

# TLS OBLIGATOIRE
tls-port 6380
port 0  # Désactiver complètement port non-chiffré

# Protected mode (toujours activé)
protected-mode yes

# Authentification avec ACLs
aclfile /etc/redis/users.acl

# Timeout connexions inactives
timeout 300

# TCP keepalive
tcp-keepalive 300

# ============================================================================
```

### 3. Vérification de la configuration bind

```bash
#!/bin/bash
# check-redis-bind.sh

echo "=== REDIS BIND CONFIGURATION CHECK ==="

# 1. Vérifier bind dans redis.conf
echo "1. Configuration bind dans redis.conf:"
grep "^bind " /etc/redis/redis.conf

# 2. Vérifier interfaces sur lesquelles Redis écoute
echo ""
echo "2. Interfaces actives (netstat):"
netstat -tlnp | grep redis-server

# Ou avec ss (moderne)
echo ""
echo "3. Interfaces actives (ss):"
ss -tlnp | grep redis-server

# 4. Vérifier via Redis directement
echo ""
echo "4. Configuration bind active:"
redis-cli CONFIG GET bind

# 5. Test de connexion depuis différentes IPs
echo ""
echo "5. Test connectivité:"

# Test localhost
if redis-cli -h 127.0.0.1 PING > /dev/null 2>&1; then
    echo "✅ Localhost (127.0.0.1): ACCESSIBLE"
else
    echo "❌ Localhost (127.0.0.1): INACCESSIBLE"
fi

# Test IP privée
PRIVATE_IP=$(hostname -I | awk '{print $1}')
if redis-cli -h $PRIVATE_IP PING > /dev/null 2>&1; then
    echo "✅ IP privée ($PRIVATE_IP): ACCESSIBLE"
else
    echo "❌ IP privée ($PRIVATE_IP): INACCESSIBLE"
fi

# Test IP publique (doit échouer!)
PUBLIC_IP=$(curl -s ifconfig.me)
if redis-cli -h $PUBLIC_IP PING > /dev/null 2>&1; then
    echo "🚨 ALERTE SÉCURITÉ: IP publique ($PUBLIC_IP): ACCESSIBLE"
    echo "    Redis est exposé sur Internet!"
else
    echo "✅ IP publique ($PUBLIC_IP): INACCESSIBLE (bon)"
fi

# 6. Shodan check (optionnel)
echo ""
echo "6. Vérifier exposition sur Shodan:"
echo "   https://www.shodan.io/host/$PUBLIC_IP"
echo "   (Redis ne doit PAS apparaître!)"
```

---

## Firewall Linux (iptables / firewalld)

### 1. iptables - Configuration basique

```bash
#!/bin/bash
# setup-iptables-redis.sh

echo "=== Configuration iptables pour Redis ==="

# Politique par défaut: DROP tout
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Autoriser loopback (localhost)
iptables -A INPUT -i lo -j ACCEPT

# Autoriser connexions établies et reliées
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Autoriser SSH (pour administration)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# REDIS: Autoriser UNIQUEMENT depuis application servers
# Remplacer par vos IPs
APP_SERVER_1="10.0.1.10"
APP_SERVER_2="10.0.1.11"
APP_SERVER_3="10.0.1.12"

iptables -A INPUT -p tcp --dport 6379 -s $APP_SERVER_1 -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -s $APP_SERVER_2 -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -s $APP_SERVER_3 -j ACCEPT

# REDIS TLS: Si port TLS séparé
iptables -A INPUT -p tcp --dport 6380 -s $APP_SERVER_1 -j ACCEPT
iptables -A INPUT -p tcp --dport 6380 -s $APP_SERVER_2 -j ACCEPT
iptables -A INPUT -p tcp --dport 6380 -s $APP_SERVER_3 -j ACCEPT

# Bloquer explicitement tout le reste vers Redis
iptables -A INPUT -p tcp --dport 6379 -j DROP
iptables -A INPUT -p tcp --dport 6380 -j DROP

# Log des tentatives bloquées (optionnel)
iptables -A INPUT -p tcp --dport 6379 -j LOG --log-prefix "REDIS_BLOCK: "
iptables -A INPUT -p tcp --dport 6380 -j LOG --log-prefix "REDIS_TLS_BLOCK: "

# Sauvegarder les règles
iptables-save > /etc/iptables/rules.v4

echo "✅ iptables configuré"
echo "Vérifier avec: iptables -L -n -v"
```

### 2. iptables - Configuration avancée avec whitelist subnet

```bash
#!/bin/bash
# setup-iptables-redis-advanced.sh

echo "=== Configuration iptables avancée pour Redis ==="

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# Politique par défaut
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback
iptables -A INPUT -i lo -j ACCEPT

# Connexions établies
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH avec rate limiting (anti brute-force)
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# REDIS: Autoriser subnet entier (application layer)
APP_SUBNET="10.0.1.0/24"

# Redis standard (6379)
iptables -A INPUT -p tcp --dport 6379 -s $APP_SUBNET -m conntrack --ctstate NEW -m recent --set --name redis
iptables -A INPUT -p tcp --dport 6379 -s $APP_SUBNET -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 100 -j DROP
iptables -A INPUT -p tcp --dport 6379 -s $APP_SUBNET -j ACCEPT

# Redis TLS (6380)
iptables -A INPUT -p tcp --dport 6380 -s $APP_SUBNET -m conntrack --ctstate NEW -m recent --set --name redis_tls
iptables -A INPUT -p tcp --dport 6380 -s $APP_SUBNET -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 100 -j DROP
iptables -A INPUT -p tcp --dport 6380 -s $APP_SUBNET -j ACCEPT

# Sentinel (26379) - si utilisé
iptables -A INPUT -p tcp --dport 26379 -s $APP_SUBNET -j ACCEPT

# Cluster bus (16379) - si cluster Redis
iptables -A INPUT -p tcp --dport 16379 -s $APP_SUBNET -j ACCEPT

# Drop tout le reste vers Redis avec log
iptables -A INPUT -p tcp --dport 6379 -j LOG --log-prefix "REDIS_UNAUTHORIZED: " --log-level 4
iptables -A INPUT -p tcp --dport 6379 -j DROP
iptables -A INPUT -p tcp --dport 6380 -j LOG --log-prefix "REDIS_TLS_UNAUTHORIZED: " --log-level 4
iptables -A INPUT -p tcp --dport 6380 -j DROP

# Protection DDoS basique
iptables -A INPUT -p tcp --syn -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# ICMP (ping) avec rate limiting
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Sauvegarder
iptables-save > /etc/iptables/rules.v4

echo "✅ iptables avancé configuré"
echo ""
echo "Règles actives:"
iptables -L -n -v --line-numbers
```

### 3. firewalld - Configuration RedHat/CentOS

```bash
#!/bin/bash
# setup-firewalld-redis.sh

echo "=== Configuration firewalld pour Redis ==="

# Activer firewalld
systemctl enable firewalld
systemctl start firewalld

# Créer une zone dédiée pour Redis
firewall-cmd --permanent --new-zone=redis

# Définir les sources autorisées (application servers)
firewall-cmd --permanent --zone=redis --add-source=10.0.1.10/32
firewall-cmd --permanent --zone=redis --add-source=10.0.1.11/32
firewall-cmd --permanent --zone=redis --add-source=10.0.1.12/32

# Ou un subnet entier
# firewall-cmd --permanent --zone=redis --add-source=10.0.1.0/24

# Autoriser ports Redis dans la zone
firewall-cmd --permanent --zone=redis --add-port=6379/tcp
firewall-cmd --permanent --zone=redis --add-port=6380/tcp  # TLS

# Sentinel (si utilisé)
# firewall-cmd --permanent --zone=redis --add-port=26379/tcp

# Cluster (si utilisé)
# firewall-cmd --permanent --zone=redis --add-port=16379/tcp

# Zone public: bloquer Redis explicitement
firewall-cmd --permanent --zone=public --remove-port=6379/tcp
firewall-cmd --permanent --zone=public --remove-port=6380/tcp

# Recharger
firewall-cmd --reload

# Vérifier
echo ""
echo "Configuration active:"
firewall-cmd --zone=redis --list-all
firewall-cmd --zone=public --list-all

echo ""
echo "✅ firewalld configuré"
```

### 4. nftables - Configuration moderne

```bash
#!/bin/bash
# setup-nftables-redis.sh

cat > /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f

# Flush existing rules
flush ruleset

# Define variables
define APP_SUBNET = 10.0.1.0/24
define REDIS_PORT = 6379
define REDIS_TLS_PORT = 6380

table inet filter {
    # Rate limiting for Redis
    set redis_ratelimit {
        type ipv4_addr
        size 65536
        flags dynamic,timeout
        timeout 60s
    }

    chain input {
        type filter hook input priority 0; policy drop;

        # Allow loopback
        iif lo accept

        # Allow established/related
        ct state established,related accept

        # Allow SSH
        tcp dport 22 ct state new accept

        # Redis: Allow only from application subnet
        ip saddr $APP_SUBNET tcp dport $REDIS_PORT ct state new \
            limit rate over 100/minute burst 200 packets \
            add @redis_ratelimit { ip saddr } drop

        ip saddr $APP_SUBNET tcp dport $REDIS_PORT accept

        # Redis TLS
        ip saddr $APP_SUBNET tcp dport $REDIS_TLS_PORT accept

        # Log unauthorized Redis attempts
        tcp dport { $REDIS_PORT, $REDIS_TLS_PORT } \
            log prefix "REDIS_BLOCK: " level warn drop

        # ICMP
        ip protocol icmp icmp type echo-request limit rate 1/second accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
EOF

# Activer nftables
systemctl enable nftables
systemctl restart nftables

echo "✅ nftables configuré"
echo ""
echo "Vérifier avec: nft list ruleset"
```

---

## Cloud Security Groups

### 1. AWS Security Groups

```bash
#!/bin/bash
# aws-security-group-redis.sh

# Configuration AWS CLI requise: aws configure

VPC_ID="vpc-0123456789abcdef0"
REDIS_SG_NAME="redis-production-sg"
REDIS_SG_DESC="Security Group for Redis Production"
APP_SG_ID="sg-app123456789abcdef"  # Security Group des app servers

echo "=== Création AWS Security Group pour Redis ==="

# 1. Créer Security Group
SG_ID=$(aws ec2 create-security-group \
    --group-name "$REDIS_SG_NAME" \
    --description "$REDIS_SG_DESC" \
    --vpc-id "$VPC_ID" \
    --output text \
    --query 'GroupId')

echo "Security Group créé: $SG_ID"

# 2. Ajouter tags
aws ec2 create-tags \
    --resources "$SG_ID" \
    --tags Key=Name,Value="Redis Production" \
           Key=Environment,Value=production \
           Key=ManagedBy,Value=terraform

# 3. Règles INGRESS (entrantes)

# Redis port 6379 depuis app servers uniquement
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --ip-permissions \
        IpProtocol=tcp,FromPort=6379,ToPort=6379,UserIdGroupPairs="[{GroupId=$APP_SG_ID,Description='App servers to Redis'}]"

# Redis TLS port 6380
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --ip-permissions \
        IpProtocol=tcp,FromPort=6380,ToPort=6380,UserIdGroupPairs="[{GroupId=$APP_SG_ID,Description='App servers to Redis TLS'}]"

# Sentinel (si utilisé)
# aws ec2 authorize-security-group-ingress \
#     --group-id "$SG_ID" \
#     --ip-permissions \
#         IpProtocol=tcp,FromPort=26379,ToPort=26379,UserIdGroupPairs="[{GroupId=$APP_SG_ID}]"

# Cluster bus (si cluster Redis)
# aws ec2 authorize-security-group-ingress \
#     --group-id "$SG_ID" \
#     --ip-permissions \
#         IpProtocol=tcp,FromPort=16379,ToPort=16379,UserIdGroupPairs="[{GroupId=$SG_ID}]"

# SSH pour administration (depuis bastion uniquement)
BASTION_SG_ID="sg-bastion123456789"
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --ip-permissions \
        IpProtocol=tcp,FromPort=22,ToPort=22,UserIdGroupPairs="[{GroupId=$BASTION_SG_ID,Description='SSH from bastion'}]"

# 4. Règles EGRESS (sortantes)
# Par défaut AWS autorise tout en sortie, restreindre si nécessaire

echo "✅ Security Group configuré: $SG_ID"
echo ""
echo "Vérifier dans AWS Console ou:"
echo "aws ec2 describe-security-groups --group-ids $SG_ID"
```

#### Terraform pour AWS Security Group

```hcl
# terraform/redis-security-group.tf

resource "aws_security_group" "redis" {
  name        = "redis-production-sg"
  description = "Security Group for Redis Production"
  vpc_id      = var.vpc_id

  # Redis standard port depuis app servers
  ingress {
    description     = "Redis from app servers"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app_servers.id]
  }

  # Redis TLS port
  ingress {
    description     = "Redis TLS from app servers"
    from_port       = 6380
    to_port         = 6380
    protocol        = "tcp"
    security_groups = [aws_security_group.app_servers.id]
  }

  # Sentinel (optionnel)
  ingress {
    description     = "Redis Sentinel"
    from_port       = 26379
    to_port         = 26379
    protocol        = "tcp"
    security_groups = [aws_security_group.app_servers.id]
  }

  # Cluster bus - entre instances Redis uniquement
  ingress {
    description = "Redis Cluster bus"
    from_port   = 16379
    to_port     = 16379
    protocol    = "tcp"
    self        = true  # Depuis ce security group lui-même
  }

  # SSH depuis bastion
  ingress {
    description     = "SSH from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  # Egress: Autoriser tout (ajuster selon besoins)
  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "Redis Production"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Output l'ID pour référence
output "redis_security_group_id" {
  value       = aws_security_group.redis.id
  description = "Security Group ID for Redis"
}
```

### 2. Azure Network Security Group (NSG)

```bash
#!/bin/bash
# azure-nsg-redis.sh

# Configuration Azure CLI requise: az login

RESOURCE_GROUP="redis-production-rg"
NSG_NAME="redis-production-nsg"
LOCATION="westeurope"

echo "=== Création Azure NSG pour Redis ==="

# 1. Créer NSG
az network nsg create \
    --resource-group "$RESOURCE_GROUP" \
    --name "$NSG_NAME" \
    --location "$LOCATION"

# 2. Règles de sécurité

# Redis port 6379 depuis app subnet
az network nsg rule create \
    --resource-group "$RESOURCE_GROUP" \
    --nsg-name "$NSG_NAME" \
    --name "Allow-Redis-From-App" \
    --priority 100 \
    --source-address-prefixes "10.0.1.0/24" \
    --destination-address-prefixes "*" \
    --destination-port-ranges 6379 \
    --protocol Tcp \
    --access Allow \
    --direction Inbound \
    --description "Allow Redis from application subnet"

# Redis TLS port 6380
az network nsg rule create \
    --resource-group "$RESOURCE_GROUP" \
    --nsg-name "$NSG_NAME" \
    --name "Allow-Redis-TLS-From-App" \
    --priority 110 \
    --source-address-prefixes "10.0.1.0/24" \
    --destination-address-prefixes "*" \
    --destination-port-ranges 6380 \
    --protocol Tcp \
    --access Allow \
    --direction Inbound

# SSH depuis bastion
az network nsg rule create \
    --resource-group "$RESOURCE_GROUP" \
    --nsg-name "$NSG_NAME" \
    --name "Allow-SSH-From-Bastion" \
    --priority 200 \
    --source-address-prefixes "10.0.0.10/32" \
    --destination-address-prefixes "*" \
    --destination-port-ranges 22 \
    --protocol Tcp \
    --access Allow \
    --direction Inbound

# Bloquer tout le reste (implicite, mais explicite c'est mieux)
az network nsg rule create \
    --resource-group "$RESOURCE_GROUP" \
    --nsg-name "$NSG_NAME" \
    --name "Deny-All-Inbound" \
    --priority 4096 \
    --source-address-prefixes "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges "*" \
    --protocol "*" \
    --access Deny \
    --direction Inbound

echo "✅ Azure NSG configuré"
echo ""
echo "Vérifier:"
echo "az network nsg show --resource-group $RESOURCE_GROUP --name $NSG_NAME"
```

### 3. GCP Firewall Rules

```bash
#!/bin/bash
# gcp-firewall-redis.sh

# Configuration gcloud requise: gcloud init

PROJECT_ID="my-project-id"
NETWORK="redis-vpc"

echo "=== Création GCP Firewall Rules pour Redis ==="

# 1. Redis port 6379 depuis app instances (par tag)
gcloud compute firewall-rules create redis-allow-from-app \
    --project="$PROJECT_ID" \
    --network="$NETWORK" \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:6379 \
    --source-tags=app-server \
    --target-tags=redis-server \
    --description="Allow Redis from app servers"

# 2. Redis TLS port 6380
gcloud compute firewall-rules create redis-tls-allow-from-app \
    --project="$PROJECT_ID" \
    --network="$NETWORK" \
    --direction=INGRESS \
    --priority=1010 \
    --action=ALLOW \
    --rules=tcp:6380 \
    --source-tags=app-server \
    --target-tags=redis-server

# 3. Sentinel (optionnel)
gcloud compute firewall-rules create redis-sentinel-allow \
    --project="$PROJECT_ID" \
    --network="$NETWORK" \
    --direction=INGRESS \
    --priority=1020 \
    --action=ALLOW \
    --rules=tcp:26379 \
    --source-tags=app-server,redis-server \
    --target-tags=redis-server

# 4. Cluster bus (entre redis instances)
gcloud compute firewall-rules create redis-cluster-bus \
    --project="$PROJECT_ID" \
    --network="$NETWORK" \
    --direction=INGRESS \
    --priority=1030 \
    --action=ALLOW \
    --rules=tcp:16379 \
    --source-tags=redis-server \
    --target-tags=redis-server

# 5. SSH depuis bastion
gcloud compute firewall-rules create redis-ssh-from-bastion \
    --project="$PROJECT_ID" \
    --network="$NETWORK" \
    --direction=INGRESS \
    --priority=2000 \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-tags=bastion \
    --target-tags=redis-server

# 6. Bloquer tout le reste vers Redis (optionnel, par défaut deny)
gcloud compute firewall-rules create redis-deny-all \
    --project="$PROJECT_ID" \
    --network="$NETWORK" \
    --direction=INGRESS \
    --priority=65534 \
    --action=DENY \
    --rules=all \
    --destination-ranges=0.0.0.0/0 \
    --target-tags=redis-server

echo "✅ GCP Firewall Rules configurés"
echo ""
echo "Vérifier:"
echo "gcloud compute firewall-rules list --filter='name~redis'"
```

---

## VPC et isolation réseau

### 1. Architecture VPC recommandée

```
┌────────────────────────────────────────────────────────────────┐
│                         VPC: 10.0.0.0/16                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  PUBLIC SUBNET: 10.0.0.0/24                              │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐          │  │
│  │  │  Bastion   │  │    NAT     │  │   ALB      │          │  │
│  │  │   Host     │  │  Gateway   │  │            │          │  │
│  │  └────────────┘  └────────────┘  └────────────┘          │  │
│  │  Route: 0.0.0.0/0 → Internet Gateway                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓ SSH only                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  PRIVATE SUBNET APP: 10.0.1.0/24                         │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐          │  │
│  │  │    App     │  │    App     │  │    App     │          │  │
│  │  │  Server 1  │  │  Server 2  │  │  Server 3  │          │  │
│  │  └────────────┘  └────────────┘  └────────────┘          │  │
│  │  Route: 0.0.0.0/0 → NAT Gateway                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓ Port 6379/6380 only             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  PRIVATE SUBNET DATA: 10.0.2.0/24 (ISOLATED)             │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐          │  │
│  │  │   Redis    │  │   Redis    │  │   Redis    │          │  │
│  │  │  Master    │  │  Replica 1 │  │  Replica 2 │          │  │
│  │  └────────────┘  └────────────┘  └────────────┘          │  │
│  │  Route: AUCUNE route vers Internet (NO NAT)              │  │
│  │  Communication: Uniquement vers subnet APP               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘

Règles de sécurité:
├── Bastion → App Subnet: SSH (22)
├── App Subnet → Data Subnet: Redis (6379, 6380)
├── Data Subnet → Data Subnet: Réplication (6379, 26379, 16379)
└── Internet → Data Subnet: BLOCKED (pas de route)
```

### 2. Terraform VPC pour Redis (AWS)

```hcl
# terraform/vpc-redis.tf

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "redis-production-vpc"
  }
}

# Internet Gateway (pour subnet public)
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "redis-production-igw"
  }
}

# ============================================================================
# PUBLIC SUBNET (Bastion, NAT Gateway)
# ============================================================================

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.0.0/24"
  availability_zone       = "eu-west-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

# Route table pour subnet public
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-route-table"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# NAT Gateway (pour que app servers puissent sortir)
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = {
    Name = "nat-gateway-eip"
  }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public.id

  tags = {
    Name = "redis-nat-gateway"
  }
}

# ============================================================================
# PRIVATE SUBNET APP (Application Servers)
# ============================================================================

resource "aws_subnet" "app" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "eu-west-1a"

  tags = {
    Name = "app-subnet"
  }
}

# Route table pour subnet app (via NAT)
resource "aws_route_table" "app" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = {
    Name = "app-route-table"
  }
}

resource "aws_route_table_association" "app" {
  subnet_id      = aws_subnet.app.id
  route_table_id = aws_route_table.app.id
}

# ============================================================================
# PRIVATE SUBNET DATA (Redis - ISOLATED)
# ============================================================================

resource "aws_subnet" "data" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "eu-west-1a"

  tags = {
    Name = "data-subnet-redis"
  }
}

# Route table pour subnet data (PAS de route Internet!)
resource "aws_route_table" "data" {
  vpc_id = aws_vpc.main.id

  # Aucune route vers Internet
  # Communication uniquement au sein du VPC

  tags = {
    Name = "data-route-table-isolated"
  }
}

resource "aws_route_table_association" "data" {
  subnet_id      = aws_subnet.data.id
  route_table_id = aws_route_table.data.id
}

# ============================================================================
# OUTPUTS
# ============================================================================

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}

output "app_subnet_id" {
  value = aws_subnet.app.id
}

output "data_subnet_id" {
  value = aws_subnet.data.id
}
```

---

## Protected Mode

### Fonctionnement de protected-mode

```conf
# ============================================================================
# PROTECTED MODE - Sécurité de secours
# ============================================================================

# Protected mode est une sécurité par défaut de Redis
# Activé par défaut depuis Redis 3.2

protected-mode yes

# Comportement:
# - Si bind = 0.0.0.0 ou interface publique
# - ET pas d'authentification (requirepass ou ACLs)
# - ALORS refuse connexions non-localhost

# Scénarios:

# Scénario 1: bind 0.0.0.0 + protected-mode yes + pas de password
# → Connexions refusées sauf depuis 127.0.0.1

# Scénario 2: bind 0.0.0.0 + protected-mode yes + requirepass défini
# → Connexions acceptées (mais nécessitent auth)

# Scénario 3: bind IP_PRIVEE + protected-mode yes
# → Connexions acceptées depuis cette IP (mais auth toujours recommandée)

# IMPORTANT: protected-mode n'est PAS une vraie sécurité
# C'est une protection contre les mauvaises configurations
# NE PAS compter uniquement dessus!

# En production:
# - bind sur IP privée
# - protected-mode yes (toujours)
# - Authentification obligatoire (ACLs)
# - Firewall configuré
```

---

## Monitoring et détection d'intrusion

### 1. Monitoring des connexions

```bash
#!/bin/bash
# monitor-redis-connections.sh

echo "=== REDIS CONNECTION MONITORING ==="

# 1. Connexions actives
echo "1. Active connections:"
redis-cli CLIENT LIST | wc -l
echo "connections active"

# 2. Connexions par IP source
echo ""
echo "2. Connections by source IP:"
redis-cli CLIENT LIST | awk '{for(i=1;i<=NF;i++){if($i~/addr=/){print $i}}}' | \
    cut -d= -f2 | cut -d: -f1 | sort | uniq -c | sort -rn

# 3. Connexions suspectes (IPs non autorisées)
echo ""
echo "3. Checking for unauthorized IPs:"
ALLOWED_IPS=("10.0.1.10" "10.0.1.11" "10.0.1.12" "127.0.0.1")

redis-cli CLIENT LIST | grep -oP 'addr=\K[^:]+' | sort -u | while read ip; do
    if [[ ! " ${ALLOWED_IPS[@]} " =~ " ${ip} " ]]; then
        echo "⚠️  Unauthorized IP connected: $ip"
        # Optionnel: Bloquer automatiquement
        # redis-cli CLIENT KILL ADDR $ip:*
    fi
done

# 4. Tentatives de connexion dans logs système
echo ""
echo "4. Recent connection attempts (from system logs):"
grep "redis-server" /var/log/syslog | grep -i "accept\|connection" | tail -10

# 5. Tentatives bloquées par iptables
echo ""
echo "5. Blocked attempts (iptables logs):"
grep "REDIS_BLOCK" /var/log/syslog | tail -10
```

### 2. Détection de scans de ports

```bash
#!/bin/bash
# detect-port-scans.sh

echo "=== PORT SCAN DETECTION ==="

# Analyser logs iptables pour détecter patterns de scan
echo "Analyzing firewall logs for scan patterns..."

# SYN scans
echo ""
echo "1. Potential SYN scans (multiple attempts from same IP):"
grep "REDIS_BLOCK" /var/log/syslog | \
    grep "SYN" | \
    awk '{print $10}' | cut -d= -f2 | \
    sort | uniq -c | sort -rn | head -10

# Scans sur multiple ports
echo ""
echo "2. IPs scanning multiple ports:"
grep "DPT=" /var/log/syslog | \
    awk '{print $10, $12}' | \
    cut -d= -f2,4 | \
    sort | uniq | \
    awk '{ip=$1; port=$2; count[ip]++; ports[ip]=ports[ip]" "port}
         END {for(i in count) if(count[i]>5) print i": "count[i]" ports -"ports[i]}'

# Alerter si détection
SCAN_COUNT=$(grep "REDIS_BLOCK" /var/log/syslog | wc -l)
if [ $SCAN_COUNT -gt 100 ]; then
    echo ""
    echo "🚨 ALERT: $SCAN_COUNT blocked attempts detected!"
    echo "Consider blocking IPs at network level"
fi
```

### 3. fail2ban pour Redis

```bash
#!/bin/bash
# setup-fail2ban-redis.sh

# Installer fail2ban
apt-get install -y fail2ban

# Créer filtre pour Redis
cat > /etc/fail2ban/filter.d/redis.conf << 'EOF'
[Definition]
failregex = .*REDIS_BLOCK.*SRC=<HOST>
            .*WRONGPASS.*from <HOST>
            .*Invalid password.*<HOST>

ignoreregex =
EOF

# Créer jail pour Redis
cat > /etc/fail2ban/jail.d/redis.conf << 'EOF'
[redis]
enabled = true
port = 6379,6380
filter = redis
logpath = /var/log/syslog
          /var/log/redis/redis.log
maxretry = 5
findtime = 300
bantime = 3600
action = iptables-multiport[name=Redis, port="6379,6380", protocol=tcp]

[redis-aggressive]
enabled = true
port = 6379,6380
filter = redis
logpath = /var/log/syslog
maxretry = 3
findtime = 60
bantime = 86400
action = iptables-multiport[name=Redis, port="6379,6380", protocol=tcp]
EOF

# Redémarrer fail2ban
systemctl restart fail2ban

echo "✅ fail2ban configuré pour Redis"
echo ""
echo "Vérifier:"
echo "fail2ban-client status redis"
```

---

## Checklist de sécurité réseau

### Checklist Configuration

- [ ] **Bind configuré sur IP privée uniquement**
  ```bash
  grep "^bind " /etc/redis/redis.conf
  # Doit montrer IP privée, PAS 0.0.0.0
  ```

- [ ] **Protected mode activé**
  ```bash
  redis-cli CONFIG GET protected-mode
  # Doit retourner "yes"
  ```

- [ ] **Port non-TLS désactivé (si TLS utilisé)**
  ```bash
  grep "^port " /etc/redis/redis.conf
  # Doit être "port 0" si TLS activé
  ```

- [ ] **Firewall local configuré**
  ```bash
  iptables -L -n | grep 6379
  # Doit montrer règles restrictives
  ```

- [ ] **Cloud Security Groups configurés**
  - Whitelist uniquement app servers
  - Deny all autres sources

- [ ] **VPC isolation configurée**
  - Redis dans subnet privé
  - Pas de route Internet directe
  - NAT Gateway si besoin de sortir

- [ ] **Ports non-standards considérés**
  - Changer 6379 vers port custom peut réduire scans automatiques

- [ ] **Monitoring connexions actif**
  - Alertes sur IPs non autorisées
  - Détection de scans de ports

### Checklist Tests

- [ ] **Test connexion depuis app server**
  ```bash
  redis-cli -h <redis-ip> -p 6379 PING
  # Doit fonctionner
  ```

- [ ] **Test connexion depuis Internet (doit échouer)**
  ```bash
  redis-cli -h <public-ip> -p 6379 PING
  # Doit timeout ou refuser
  ```

- [ ] **Scan de ports externe**
  ```bash
  nmap <public-ip>
  # Redis NE DOIT PAS apparaître
  ```

- [ ] **Vérifier exposition Shodan**
  - https://www.shodan.io/search?query=redis
  - Votre IP ne doit pas apparaître

- [ ] **Test fail2ban**
  - Générer échecs auth
  - Vérifier ban automatique

### Checklist Monitoring

- [ ] **Alertes connexions suspectes**
  - IP non autorisée
  - Tentatives multiples

- [ ] **Logs centralisés**
  - iptables logs → SIEM
  - Redis logs → SIEM

- [ ] **Dashboards réseau**
  - Connexions actives
  - Tentatives bloquées
  - Bande passante

### Checklist Incident

- [ ] **Procédure blocage IP**
  ```bash
  # Bloquer IP immédiatement
  iptables -I INPUT 1 -s <malicious-ip> -j DROP

  # Ou dans cloud
  aws ec2 revoke-security-group-ingress ...
  ```

- [ ] **Analyse post-incident**
  - Logs firewall
  - Logs Redis
  - Logs applicatifs

- [ ] **Mise à jour règles**
  - Renforcer si nécessaire
  - Documenter incident

---

## Scripts d'automatisation

### Script de validation complète

```bash
#!/bin/bash
# validate-redis-network-security.sh

echo "========================================="
echo "REDIS NETWORK SECURITY VALIDATION"
echo "========================================="

ERRORS=0

# 1. Vérifier bind
echo ""
echo "1. Checking bind configuration..."
BIND_CONFIG=$(grep "^bind " /etc/redis/redis.conf | awk '{print $2}')

if [[ $BIND_CONFIG == "0.0.0.0" ]]; then
    echo "❌ CRITICAL: Redis bound to 0.0.0.0 (all interfaces)"
    ERRORS=$((ERRORS + 1))
elif [[ $BIND_CONFIG == "127.0.0.1" ]]; then
    echo "✅ OK: Redis bound to localhost (dev/test only)"
elif [[ $BIND_CONFIG =~ ^10\.|^172\.(1[6-9]|2[0-9]|3[0-1])\.|^192\.168\. ]]; then
    echo "✅ OK: Redis bound to private IP: $BIND_CONFIG"
else
    echo "⚠️  WARNING: Redis bound to: $BIND_CONFIG (verify this is correct)"
fi

# 2. Vérifier protected mode
echo ""
echo "2. Checking protected mode..."
PROTECTED=$(redis-cli CONFIG GET protected-mode | tail -1)

if [[ $PROTECTED == "yes" ]]; then
    echo "✅ OK: Protected mode enabled"
else
    echo "❌ CRITICAL: Protected mode disabled"
    ERRORS=$((ERRORS + 1))
fi

# 3. Vérifier écoute réseau
echo ""
echo "3. Checking network listeners..."
LISTENERS=$(ss -tlnp | grep redis-server)
echo "$LISTENERS"

if echo "$LISTENERS" | grep -q "0.0.0.0:6379"; then
    echo "❌ CRITICAL: Redis listening on 0.0.0.0"
    ERRORS=$((ERRORS + 1))
fi

# 4. Vérifier firewall
echo ""
echo "4. Checking firewall..."
if command -v iptables &> /dev/null; then
    REDIS_RULES=$(iptables -L -n | grep -c "6379\|6380")
    if [ $REDIS_RULES -gt 0 ]; then
        echo "✅ OK: Firewall rules found for Redis"
        iptables -L -n | grep "6379\|6380"
    else
        echo "⚠️  WARNING: No firewall rules found for Redis"
    fi
fi

# 5. Test connexion publique
echo ""
echo "5. Testing public exposure..."
PUBLIC_IP=$(curl -s ifconfig.me)
echo "Public IP: $PUBLIC_IP"

if timeout 3 bash -c "echo > /dev/tcp/$PUBLIC_IP/6379" 2>/dev/null; then
    echo "🚨 CRITICAL SECURITY ISSUE: Redis accessible from public IP!"
    ERRORS=$((ERRORS + 1))
else
    echo "✅ OK: Redis not accessible from public IP"
fi

# 6. Vérifier authentification
echo ""
echo "6. Checking authentication..."
if redis-cli PING 2>/dev/null | grep -q "PONG"; then
    echo "⚠️  WARNING: Redis accessible without authentication"
else
    echo "✅ OK: Redis requires authentication"
fi

# Résultat final
echo ""
echo "========================================="
if [ $ERRORS -eq 0 ]; then
    echo "✅ VALIDATION PASSED"
    echo "Network security looks good!"
    exit 0
else
    echo "❌ VALIDATION FAILED"
    echo "$ERRORS critical issue(s) found"
    exit 1
fi
```

---

## 📚 Ressources complémentaires

### Documentation

- [Redis Security](https://redis.io/docs/management/security/)
- [Redis Network Security Best Practices](https://redis.io/docs/management/security/#network-security)
- [iptables Documentation](https://netfilter.org/documentation/)

### Outils

- **nmap** - Port scanning
- **tcpdump** - Network analysis
- **Wireshark** - Packet inspection
- **fail2ban** - Intrusion prevention
- **Shodan** - Internet device search

### Standards

- **CIS Benchmarks** - Redis Security Configuration
- **NIST SP 800-123** - Guide to Network Security
- **PCI DSS** - Network segmentation requirements

---

**Section suivante :** [12.6 - Bonnes pratiques Linux : THP, Swap, Overcommit memory](./06-bonnes-pratiques-linux.md)

⏭️ [Bonnes pratiques Linux : THP, Swap, Overcommit memory](/12-redis-production-securite/06-bonnes-pratiques-linux.md)
