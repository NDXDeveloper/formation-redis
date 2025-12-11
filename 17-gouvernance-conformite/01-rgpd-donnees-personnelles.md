🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.1 RGPD et données personnelles dans Redis

## Introduction

Le Règlement Général sur la Protection des Données (RGPD) est le cadre légal le plus exigeant au monde en matière de protection des données personnelles. Son application à Redis, base de données in-memory souvent utilisée pour le caching et les données temporaires, soulève des défis spécifiques que nous allons analyser en profondeur.

### Pourquoi le RGPD s'applique à Redis

**Idées reçues dangereuses :**
- ❌ "Redis est juste un cache temporaire, le RGPD ne s'applique pas"
- ❌ "Les données ont un TTL court, pas besoin de conformité"
- ❌ "C'est de la donnée technique, pas des données personnelles"

**Réalité juridique :**
- ✅ Toute donnée personnelle, même temporaire, est concernée
- ✅ Le RGPD s'applique dès la collecte, indépendamment de la durée de conservation
- ✅ Les identifiants techniques (session_id, user_id, IP) sont des données personnelles

### Périmètre d'application

**Le RGPD s'applique si :**
1. Votre organisation est établie dans l'UE (peu importe où sont les clients)
2. Vous traitez des données de résidents UE (peu importe où est votre organisation)
3. Vous proposez des biens/services à des personnes dans l'UE
4. Vous surveillez le comportement de personnes dans l'UE

**Redis est concerné si :**
- Stockage de sessions utilisateurs (user_id, IP, historique navigation)
- Cache de profils utilisateurs (nom, email, préférences)
- Queues contenant des données personnelles
- Analytics temps réel (tracking comportemental)
- Leaderboards avec pseudonymes
- Rate limiting basé sur IP ou user_id
- Toute donnée permettant d'identifier directement ou indirectement une personne

---

## Article 4 : Définitions clés

### 1. Donnée personnelle (Article 4.1)

> "Toute information se rapportant à une personne physique identifiée ou identifiable"

**Dans Redis, exemples de données personnelles :**

#### Identifiants directs
```
user:12345:profile → {"name": "Marie Dupont", "email": "marie@example.com"}
session:abc123 → {"user_id": 12345, "ip": "192.168.1.1"}
```

#### Identifiants indirects (permettant l'identification par recoupement)
```
device:uuid-4567 → {"fingerprint": "...", "visits": [...]}
ip:192.168.1.1:visits → ["page1", "page2", "page3"]
```

#### Données pseudonymisées (toujours des données personnelles !)
```
hashed_user:7a4b8c9d → {"preferences": {...}}
# Si vous avez la table de correspondance, c'est une donnée personnelle
```

**Données non personnelles :**
```
stats:global:pageviews → 150000  # Agrégat anonyme
product:456:stock → 42           # Pas de lien avec une personne
```

### 2. Traitement (Article 4.2)

> "Toute opération effectuée sur des données personnelles"

**En Redis, TOUT est traitement :**
- `SET user:123 {...}` → Collecte/Enregistrement
- `GET user:123` → Consultation
- `HSET user:123 email "new@email.com"` → Modification
- `DEL user:123` → Effacement
- `BGSAVE` → Structuration/Conservation
- Réplication vers replica → Communication/Transmission
- `EXPIRE user:123 3600` → Limitation de la conservation

### 3. Responsable du traitement vs Sous-traitant (Articles 4.7 et 4.8)

**Distinction critique pour la responsabilité juridique :**

#### Vous êtes RESPONSABLE si :
- Vous décidez des finalités (pourquoi stocker dans Redis ?)
- Vous décidez des moyens (quelles données, combien de temps ?)
- **Responsabilité principale** en cas de violation

#### Vous êtes SOUS-TRAITANT si :
- Vous traitez pour le compte d'un autre (ex: agence, hébergeur)
- Vous suivez les instructions du responsable
- **Responsabilité solidaire** en cas de violation

**Implications pour Redis :**
```
┌─────────────────────────────────────────────────────────────┐
│ Entreprise X (Responsable de traitement)                    │
│ • Décide de stocker les sessions utilisateurs dans Redis    │
│ • Définit la durée de rétention (TTL)                       │
│ • Détermine les mesures de sécurité requises                │
│                                                             │
│   ↓ Contrat de sous-traitance (Article 28)                  │
│                                                             │
│ Cloud Provider Y (Sous-traitant)                            │
│ • Héberge Redis selon les instructions de X                 │
│ • Applique les mesures de sécurité demandées                │
│ • Ne peut pas utiliser les données pour ses propres fins    │
└─────────────────────────────────────────────────────────────┘
```

**Contrat de sous-traitance obligatoire (voir Article 28) :**
Si vous utilisez AWS ElastiCache, Azure Cache, ou tout hébergeur, un contrat formel doit exister.

---

## Article 5 : Principes fondamentaux

### Principe 1 : Licéité, loyauté, transparence (Article 5.1.a)

**Obligation :**
Le traitement doit reposer sur une base légale (Article 6) et être transparent pour les personnes.

**Application à Redis :**

#### Licéité : Identifier la base légale
```yaml
# Exemple : Session store
Traitement: Stockage session utilisateur dans Redis
Base légale: Exécution d'un contrat (Article 6.1.b)
  → L'utilisateur a créé un compte, le cache de session est nécessaire au service

# Exemple : Analytics comportementales
Traitement: Tracking des pages visitées
Base légale: Consentement (Article 6.1.a) OU Intérêt légitime (Article 6.1.f)
  → Si consentement: banner cookie obligatoire
  → Si intérêt légitime: test de proportionnalité requis
```

#### Transparence : Information des personnes
**Obligation d'information (Articles 13-14) :**
Votre politique de confidentialité doit mentionner :
```
"Nous utilisons Redis pour stocker temporairement vos données de session
afin d'améliorer les performances de notre service. Ces données sont
conservées 24 heures puis automatiquement supprimées."
```

**Checklist transparence :**
```
□ Finalité du stockage Redis documentée
□ Durée de conservation communiquée (TTL)
□ Droits des personnes expliqués (accès, effacement, etc.)
□ Destinataires des données précisés (ex: équipe support)
□ Transferts hors UE mentionnés si applicable
□ Existence de décisions automatisées indiquée (Article 22)
```

### Principe 2 : Limitation des finalités (Article 5.1.b)

**Obligation :**
Les données doivent être collectées pour des finalités déterminées, explicites et légitimes.

**Interdiction de réutilisation incompatible :**

❌ **Anti-pattern :**
```python
# Données collectées pour "authentification"
session_data = redis.get(f"session:{session_id}")

# ❌ VIOLATION : Réutilisées pour "marketing" sans nouvelle base légale
send_marketing_email(session_data['email'])
```

✅ **Pattern conforme :**
```python
# Finalité 1 : Authentification (consentement via CGU)
redis.setex(f"session:{sid}", 3600, user_data)

# Finalité 2 : Marketing (consentement explicite séparé requis)
if user.marketing_consent:
    redis.sadd("marketing:subscribers", user.email)
```

**Documentation des finalités :**
```
# Registre des traitements Redis
session:* → Finalité: Authentification utilisateur
cart:* → Finalité: Panier d'achat temporaire
analytics:* → Finalité: Statistiques d'usage (consentement requis)
rate_limit:* → Finalité: Protection contre abus (intérêt légitime)
```

### Principe 3 : Minimisation des données (Article 5.1.c)

**Obligation :**
Collecter uniquement les données adéquates, pertinentes et limitées au nécessaire.

**Application Redis :**

❌ **Stockage excessif :**
```python
# ❌ Stocker tout le profil pour juste afficher le nom
redis.hset(f"user:{user_id}:cache", mapping={
    "name": "Marie",
    "email": "marie@example.com",
    "phone": "+33612345678",
    "address": "123 rue...",
    "ssn": "1234567890123",  # ❌ Numéro sécu en cache ?!
    "credit_card": "1234-5678-9012-3456",  # ❌ JAMAIS !
})
```

✅ **Minimisation :**
```python
# ✅ Stocker uniquement ce qui est nécessaire pour l'affichage
redis.hset(f"user:{user_id}:display", mapping={
    "name": "Marie",
    "avatar_url": "https://..."
})

# Données sensibles uniquement en DB principale avec chiffrement
```

**Checklist minimisation :**
```
Pour chaque clé Redis, se demander :
□ Cette donnée est-elle strictement nécessaire à la finalité ?
□ Ne puis-je pas la recalculer/récupérer à la demande ?
□ La durée de conservation est-elle le strict minimum ?
□ Puis-je anonymiser ou pseudonymiser cette donnée ?
```

### Principe 4 : Exactitude (Article 5.1.d)

**Obligation :**
Les données doivent être exactes et mises à jour si nécessaire.

**Problématique Redis :**
Le cache peut devenir obsolète (stale cache).

**Solutions conformes :**

#### 1. Invalidation proactive
```python
def update_user_email(user_id, new_email):
    # 1. Mise à jour DB principale
    db.users.update(user_id, email=new_email)

    # 2. ✅ Invalidation du cache immédiate
    redis.delete(f"user:{user_id}:profile")

    # OU mise à jour directe du cache
    redis.hset(f"user:{user_id}:profile", "email", new_email)
```

#### 2. TTL court pour données changeantes
```python
# Données peu volatiles : TTL long
redis.setex("product:123:name", 86400, "Product Name")

# Données volatiles : TTL court
redis.setex(f"user:{user_id}:online_status", 60, "online")
```

#### 3. Pattern "Cache Aside" avec versioning
```python
def get_user_profile(user_id):
    # Vérifier le numéro de version
    cache_key = f"user:{user_id}:v{get_user_version(user_id)}"
    cached = redis.get(cache_key)

    if cached:
        return cached

    # Si changement de version, l'ancienne clé est obsolète
    profile = db.get_user(user_id)
    redis.setex(cache_key, 3600, profile)
    return profile
```

### Principe 5 : Limitation de la conservation (Article 5.1.e)

**Obligation :**
Conserver les données uniquement le temps nécessaire aux finalités.

**Redis et TTL : Un atout pour la conformité**

✅ **Bonne pratique : TTL systématique**
```python
# ❌ Sans TTL : Conservation indéfinie
redis.set("session:abc", session_data)

# ✅ Avec TTL : Conservation limitée
redis.setex("session:abc", 3600, session_data)  # 1 heure

# ✅ Pattern : TTL par type de donnée
TTL_CONFIG = {
    "session": 3600,        # 1 heure
    "cart": 86400 * 7,      # 7 jours
    "verification_code": 300,  # 5 minutes
    "rate_limit": 60,       # 1 minute
}

redis.setex(f"{data_type}:{key}", TTL_CONFIG[data_type], value)
```

**Politique de rétention documentée :**
```
┌─────────────────────────────────────────────────────────────┐
│ Politique de rétention Redis - Exemple                      │
├─────────────────────────────────────────────────────────────┤
│ Type de donnée    │ Durée Redis │ Durée Backup │ Base légale│
├───────────────────┼─────────────┼──────────────┼────────────┤
│ Session auth      │ 24h (TTL)   │ 7j           │ Contrat    │
│ Panier d'achat    │ 30j (TTL)   │ 60j          │ Contrat    │
│ Token 2FA         │ 5min (TTL)  │ 24h          │ Sécurité   │
│ Analytics user    │ 90j (TTL)   │ 13 mois      │ Consentement
│ Rate limit IP     │ 1h (TTL)    │ 30j          │ Int. légit.│
└─────────────────────────────────────────────────────────────┘
```

**Attention aux backups RDB/AOF :**
```bash
# Le TTL est préservé dans RDB
# À la restauration, les clés expireront normalement
redis-cli --rdb dump.rdb | grep -A5 "session:abc"
# → Vérifier que le TTL est présent

# ⚠️ Politique de rétention des backups doit être cohérente
# Si TTL Redis = 24h, ne pas conserver les backups 1 an !
```

### Principe 6 : Intégrité et confidentialité (Article 5.1.f)

**Obligation :**
Sécurité appropriée, y compris protection contre le traitement non autorisé.

**Mesures de sécurité Redis (détaillées plus loin) :**

#### Confidentialité
```
□ TLS/SSL activé (chiffrement en transit)
□ ACL configurées (contrôle d'accès granulaire)
□ Firewall restrictif (pas d'exposition Internet)
□ Chiffrement at-rest (filesystem ou Redis Enterprise)
□ Authentification forte (pas de mot de passe par défaut)
```

#### Intégrité
```
□ ACL en lecture seule pour utilisateurs non admin
□ Commandes dangereuses désactivées (FLUSHDB, CONFIG, SCRIPT)
□ Audit logs activés (traçabilité des modifications)
□ Checksums RDB/AOF
□ Réplication pour redondance
```

#### Disponibilité
```
□ Haute disponibilité (Sentinel ou Cluster)
□ Backups réguliers et testés
□ Monitoring et alerting
□ Plan de reprise d'activité
```

### Principe 7 : Accountability (Article 5.2)

**Obligation :**
Le responsable du traitement doit être en mesure de démontrer la conformité.

**Documentation requise :**

#### 1. Registre des traitements (Article 30)
```markdown
## Traitement : Cache utilisateur Redis

**Responsable :** Société XYZ SAS
**DPO :** dpo@xyz.com
**Date création :** 2024-01-15
**Dernière révision :** 2024-12-01

**Finalités :**
- Amélioration des performances (temps de réponse < 100ms)
- Réduction de la charge DB principale

**Base légale :** Article 6.1.b (Exécution du contrat)

**Catégories de données :**
- Identifiants (user_id, session_id)
- Données de profil (nom, avatar)
- Préférences utilisateur (langue, thème)

**Catégories de personnes :**
- Utilisateurs authentifiés du service web

**Destinataires :**
- Application web (backend API)
- Équipe DevOps (support niveau 3 uniquement)

**Transferts hors UE :** Non

**Durées de conservation :**
- En cache Redis : 24 heures (TTL automatique)
- Backups RDB : 30 jours
- Logs d'audit : 12 mois

**Mesures de sécurité techniques :**
- Chiffrement TLS 1.3 (clients et inter-nœuds)
- ACL Redis avec comptes nominatifs
- Firewall : Accès limité aux serveurs applicatifs (10.0.1.0/24)
- Backups chiffrés (AES-256) dans S3 avec versioning
- Monitoring 24/7 avec alerting PagerDuty
- Réplication Master-Replica (2 replicas)

**Mesures de sécurité organisationnelles :**
- Accès admin via bastion host + MFA
- Revue trimestrielle des accès
- Formation annuelle RGPD des équipes
- Procédure d'effacement documentée

**DPIA effectuée :** Non (pas de traitement à grande échelle de données sensibles)
**Risques identifiés :** Accès non autorisé (atténué par ACL+TLS), Perte de données (atténué par backups)
```

#### 2. Documentation technique
```bash
# Architecture diagram (à jour)
# Configuration files (versionned in Git)
# Runbooks (procédures opérationnelles)
# Logs de changements (audit trail)
```

#### 3. Preuves de conformité
```
□ Résultats des audits de sécurité
□ Certificats de formation des équipes
□ Tests de restauration de backups
□ Rapports de tests de pénétration
□ Logs d'exécution des procédures d'effacement
```

---

## Article 6 : Licéité du traitement

### Les 6 bases légales

Tout traitement dans Redis DOIT reposer sur une des 6 bases légales :

#### 1. Consentement (Article 6.1.a)

**Exigences :**
- Libre, spécifique, éclairé, univoque
- Possibilité de retirer aussi facilement que de donner
- Consentement distinct pour chaque finalité

**Cas d'usage Redis :**
```python
# Analytics utilisateur, tracking comportemental
if user.consent_analytics:
    redis.lpush(f"user:{user_id}:page_views", {
        "page": request.path,
        "timestamp": time.time()
    })
    redis.ltrim(f"user:{user_id}:page_views", 0, 99)  # Garder 100 dernières
    redis.expire(f"user:{user_id}:page_views", 86400 * 90)  # 90 jours
```

**Retrait du consentement :**
```python
def withdraw_analytics_consent(user_id):
    # 1. Marquer le retrait en DB
    db.users.update(user_id, consent_analytics=False)

    # 2. ✅ Effacement immédiat des données Redis
    redis.delete(f"user:{user_id}:page_views")
    redis.delete(f"user:{user_id}:behavior_score")

    # 3. Planifier la purge des backups
    schedule_backup_purge(user_id, "analytics")

    # 4. Logger l'opération (audit)
    audit_log.info(f"Consent withdrawn and data deleted for user {user_id}")
```

#### 2. Exécution d'un contrat (Article 6.1.b)

**La base légale la plus courante pour Redis**

**Cas d'usage :**
```python
# Session store : Nécessaire pour que l'utilisateur utilise le service
redis.setex(f"session:{session_id}", 3600, {
    "user_id": user_id,
    "authenticated": True,
    "csrf_token": "..."
})

# Panier d'achat : Nécessaire pour finaliser la commande
redis.hset(f"cart:{user_id}", mapping={
    "item:123": 2,
    "item:456": 1
})
redis.expire(f"cart:{user_id}", 86400 * 30)  # 30 jours
```

**Test de nécessité :**
> "Le service peut-il fonctionner sans cette donnée dans Redis ?"
> Si NON → Base légale "contrat" valide
> Si OUI → Trouver une autre base légale (ex: consentement, intérêt légitime)

#### 3. Obligation légale (Article 6.1.c)

**Rarement applicable pour Redis**

**Exemple :**
```python
# Conservation des logs d'accès pour des raisons légales (ex: cybersécurité)
# Directive NIS, LCEN en France
redis.lpush("security:access_logs", {
    "user_id": user_id,
    "ip": request.ip,
    "action": "login_success",
    "timestamp": time.time()
})
redis.expire("security:access_logs", 86400 * 365)  # 1 an (obligation légale)
```

#### 4. Sauvegarde des intérêts vitaux (Article 6.1.d)

**Très rarement applicable (urgences médicales, etc.)**

#### 5. Mission d'intérêt public (Article 6.1.e)

**Applicable pour les organismes publics**

#### 6. Intérêt légitime (Article 6.1.f)

**Base légale flexible mais nécessitant un test de proportionnalité**

**Test en 3 étapes :**
1. **But légitime :** Améliorer la sécurité, prévenir la fraude, optimiser le service
2. **Nécessité :** Le traitement est-il nécessaire pour atteindre ce but ?
3. **Équilibre :** Les intérêts de l'organisation ne priment-ils pas indûment sur les droits des personnes ?

**Cas d'usage Redis :**
```python
# Rate limiting anti-brute force (intérêt légitime : sécurité)
def check_rate_limit(ip_address):
    key = f"rate_limit:{ip_address}:login_attempts"
    attempts = redis.incr(key)

    if attempts == 1:
        redis.expire(key, 300)  # 5 minutes

    if attempts > 5:
        # Bloquer temporairement
        redis.setex(f"blocked:{ip_address}", 900, "1")  # 15 minutes
        return False

    return True

# Documentation de l'intérêt légitime requis :
# "Nous utilisons le rate limiting pour protéger nos utilisateurs contre
# les attaques par force brute. Cette mesure est proportionnée car elle
# stocke uniquement l'IP (anonymisée après 5 min) et ne bloque que temporairement."
```

---

## Articles 12-14 : Information des personnes

### Obligation de transparence

**Ce qui doit être communiqué concernant Redis :**

```markdown
### Politique de confidentialité - Section Cache et Performance

**Utilisation de technologies de cache**
Nous utilisons Redis, une base de données en mémoire, pour améliorer
les performances de notre service et vous offrir une expérience utilisateur
optimale.

**Données stockées temporairement :**
- Identifiant de session (conservé 24 heures)
- Panier d'achat (conservé 30 jours)
- Préférences d'affichage (conservées 90 jours)

**Finalités :**
- Maintenir votre session connectée
- Accélérer le chargement des pages (temps de réponse < 100ms)
- Mémoriser votre panier entre les visites

**Durées de conservation :**
Les données en cache sont automatiquement supprimées après les délais
indiqués ci-dessus. Les copies de sauvegarde sont conservées 30 jours
pour des raisons de sécurité et de reprise d'activité.

**Vos droits :**
Vous pouvez à tout moment demander l'accès, la rectification ou
l'effacement de vos données personnelles, y compris celles en cache,
en nous contactant à privacy@example.com.

**Localisation :**
Les serveurs Redis sont situés en Union Européenne (France - région eu-west-3).

**Sécurité :**
Toutes les communications avec Redis sont chiffrées (TLS 1.3) et
l'accès est strictement contrôlé.
```

**Checklist information :**
```
□ Mention de l'utilisation de Redis (ou "cache") dans la politique
□ Finalités explicites et compréhensibles
□ Durées de conservation (TTL) indiquées
□ Base légale identifiée
□ Droits des personnes rappelés
□ Coordonnées du DPO/contact confidentialité
□ Localisation géographique des données
□ Mesures de sécurité résumées
□ Politique accessible (pas enfouie dans des CGU illisibles)
```

---

## Articles 15-22 : Droits des personnes

### 1. Droit d'accès (Article 15)

**Obligation :**
Fournir une copie des données personnelles traitées + informations sur le traitement.

**Procédure d'accès Redis :**

```python
def export_user_data_from_redis(user_id):
    """
    Exporter toutes les données personnelles d'un utilisateur stockées dans Redis
    Délai de réponse : 1 mois (Article 12.3)
    """

    export = {
        "user_id": user_id,
        "export_date": datetime.now().isoformat(),
        "data_categories": {}
    }

    # 1. Session data
    session_keys = redis.keys(f"session:*")
    for key in session_keys:
        session_data = redis.get(key)
        if session_data and session_data.get("user_id") == user_id:
            export["data_categories"]["session"] = {
                "session_id": key.split(":")[-1],
                "created_at": session_data.get("created_at"),
                "last_activity": session_data.get("last_activity"),
                "ip_address": session_data.get("ip")  # Si stockée
            }

    # 2. Shopping cart
    cart_data = redis.hgetall(f"cart:{user_id}")
    if cart_data:
        export["data_categories"]["shopping_cart"] = {
            "items": cart_data,
            "ttl": redis.ttl(f"cart:{user_id}")
        }

    # 3. Analytics / Page views (si consentement)
    page_views = redis.lrange(f"user:{user_id}:page_views", 0, -1)
    if page_views:
        export["data_categories"]["analytics"] = {
            "page_views": page_views,
            "consent_date": get_consent_date(user_id)
        }

    # 4. Rate limiting data (IP-based)
    # ⚠️ Attention : L'IP est une donnée personnelle
    user_ips = get_user_ips_from_logs(user_id)  # Depuis logs applicatifs
    rate_limit_data = {}
    for ip in user_ips:
        if redis.exists(f"rate_limit:{ip}"):
            rate_limit_data[ip] = redis.get(f"rate_limit:{ip}")

    if rate_limit_data:
        export["data_categories"]["rate_limiting"] = rate_limit_data

    # 5. Métadonnées sur la conservation
    export["retention_info"] = {
        "session": "24 hours (automatic TTL)",
        "cart": "30 days (automatic TTL)",
        "analytics": "90 days (automatic TTL)",
        "backups": "30 days in encrypted S3",
        "audit_logs": "12 months"
    }

    # 6. Informations sur le traitement
    export["processing_info"] = {
        "controller": "Company XYZ",
        "dpo_contact": "dpo@xyz.com",
        "purposes": [
            "Session management",
            "Shopping cart persistence",
            "Performance optimization"
        ],
        "legal_basis": "Contract execution (Art. 6.1.b)",
        "recipients": ["Web application", "DevOps team (L3 support only)"],
        "transfers": "None (all data within EU)",
        "security_measures": [
            "TLS 1.3 encryption",
            "ACL with nominal accounts",
            "Firewall restrictions",
            "Regular backups",
            "24/7 monitoring"
        ]
    }

    return export

# Exemple d'utilisation
user_export = export_user_data_from_redis(12345)

# Format JSON pour l'utilisateur
with open(f"user_data_export_{user_id}.json", "w") as f:
    json.dump(user_export, f, indent=2, ensure_ascii=False)

# Chiffrer avant envoi (RGPD recommandé)
encrypt_file(f"user_data_export_{user_id}.json")
```

**Format de fourniture :**
```
✅ Format structuré (JSON, CSV)
✅ Lisible par l'utilisateur (pas de dumps binaires)
✅ Complet (toutes les catégories de données)
✅ Accompagné d'explications (pas juste des données brutes)
❌ Ne pas envoyer par email non chiffré si données sensibles
```

### 2. Droit de rectification (Article 16)

**Obligation :**
Corriger les données inexactes sans délai.

**Procédure de rectification Redis :**

```python
def rectify_user_data(user_id, field, new_value):
    """
    Rectifier une donnée utilisateur dans Redis et propager la modification
    """

    # 1. Validation de la demande
    if not validate_rectification_request(user_id, field):
        raise ValueError("Invalid rectification request")

    # 2. Mise à jour DB principale (source de vérité)
    db.users.update(user_id, {field: new_value})

    # 3. ✅ Invalidation du cache Redis
    # Option A : Suppression (le cache se refera à la demande)
    redis.delete(f"user:{user_id}:profile")

    # Option B : Mise à jour directe du cache
    redis.hset(f"user:{user_id}:profile", field, new_value)

    # 4. Si données répliquées ailleurs
    redis.delete(f"user:{user_id}:display_name")  # Cache dérivé
    redis.hdel(f"leaderboard:names", user_id)  # Si dans un leaderboard

    # 5. Logging pour audit
    audit_log.info(f"Data rectified for user {user_id}: {field} updated", extra={
        "user_id": user_id,
        "field": field,
        "timestamp": time.time(),
        "requester": get_current_user()
    })

    # 6. Notification (si requis)
    notify_user(user_id, "Your data has been updated successfully")

    return {"status": "success", "field": field, "updated_at": datetime.now()}
```

**Propagation de la rectification :**
```
1. Base de données principale (PostgreSQL, MongoDB...)
2. Cache Redis (invalidation ou mise à jour)
3. Réplicas Redis (propagation automatique)
4. Backups futurs (contiendront la donnée rectifiée)
5. ⚠️ Backups existants : difficile à rectifier → Justifier la conservation

Documentation :
"Les backups existants conservent les données anciennes pour des raisons
techniques (impossibilité de modification rétroactive). Cependant, en cas
de restauration, les données rectifiées seront prioritaires (écrasement)."
```

### 3. Droit à l'effacement / "Droit à l'oubli" (Article 17)

**L'article le plus critique pour Redis**

#### Cas d'application obligatoire

L'effacement DOIT être effectué si :
1. Les données ne sont plus nécessaires aux finalités (TTL expiré naturellement)
2. La personne retire son consentement (et pas d'autre base légale)
3. La personne s'oppose au traitement (Article 21) et pas de motif légitime impérieux
4. Traitement illicite (ex: absence de base légale)
5. Obligation légale d'effacement
6. Données collectées pour services à un enfant (< 16 ans en France)

#### Exceptions (refus légitime d'effacement)

```python
# Vérification des exceptions avant effacement
def can_delete_user_data(user_id):
    exceptions = []

    # Exception a) Liberté d'expression (rare pour Redis)

    # Exception b) Obligation légale
    if legal_retention_required(user_id):
        exceptions.append("Legal retention obligation (Art. 17.3.b)")
        # Ex: Logs de connexion pour cybersécurité (LCEN)

    # Exception c) Motif d'intérêt public (santé, archives)

    # Exception d) Constatation, exercice, défense de droits en justice
    if has_pending_litigation(user_id):
        exceptions.append("Pending legal proceedings (Art. 17.3.e)")

    # Exception e) Archivage, recherche scientifique, statistiques
    if data_used_in_research(user_id):
        exceptions.append("Scientific research (Art. 17.3.d)")
        # Note : Doit être anonymisé si possible

    return len(exceptions) == 0, exceptions
```

#### Procédure d'effacement complète

**Cette procédure doit être documentée et testée régulièrement**

```python
class RGPDErasureService:
    """
    Service centralisé pour l'effacement RGPD
    Audit trail complet, idempotent, réversible (pendant 30 jours)
    """

    def erase_user_data(self, user_id, reason, requester):
        """
        Effacement complet des données personnelles d'un utilisateur

        Args:
            user_id: Identifiant utilisateur
            reason: Motif de l'effacement ("user_request", "account_deletion", etc.)
            requester: Qui demande l'effacement (user, admin, automated_process)

        Returns:
            ErasureReport avec détails de l'opération
        """

        # 0. Vérification des droits et exceptions
        can_delete, exceptions = can_delete_user_data(user_id)
        if not can_delete:
            return ErasureReport(
                user_id=user_id,
                status="refused",
                reason=f"Cannot delete due to: {', '.join(exceptions)}",
                timestamp=datetime.now()
            )

        # 1. Créer un rapport d'effacement
        report = ErasureReport(user_id=user_id, reason=reason, requester=requester)

        # 2. ✅ Phase 1 : Effacement Redis (données chaudes)
        redis_keys_deleted = self._erase_from_redis(user_id, report)

        # 3. ✅ Phase 2 : Marquage en base de données principale
        db_result = self._mark_user_as_deleted(user_id, report)

        # 4. ✅ Phase 3 : Gestion des backups
        backup_result = self._schedule_backup_purge(user_id, report)

        # 5. ✅ Phase 4 : Logs d'audit (conservation pour preuve de conformité)
        audit_result = self._create_audit_trail(user_id, report)

        # 6. ✅ Phase 5 : Notification des sous-traitants (si applicable)
        if has_subprocessors():
            self._notify_subprocessors_for_erasure(user_id)

        # 7. Finaliser le rapport
        report.finalize(success=True)

        # 8. Notification à la personne concernée
        send_confirmation_email(user_id, report)

        return report

    def _erase_from_redis(self, user_id, report):
        """Effacement de toutes les données Redis de l'utilisateur"""

        keys_deleted = []

        # Pattern 1 : Clés avec user_id dans le nom
        patterns = [
            f"user:{user_id}:*",
            f"session:*",  # Nécessite scan et vérification du contenu
            f"cart:{user_id}",
            f"preferences:{user_id}",
            f"analytics:user:{user_id}:*",
        ]

        for pattern in patterns:
            if "*" in pattern and "user_id" not in pattern:
                # Scan requis pour vérifier le contenu
                keys_to_check = redis.scan_iter(match=pattern, count=100)
                for key in keys_to_check:
                    data = redis.get(key)
                    if data and self._contains_user_data(data, user_id):
                        redis.delete(key)
                        keys_deleted.append(key)
            else:
                # Suppression directe
                keys = redis.keys(pattern) if "*" in pattern else [pattern]
                for key in keys:
                    if redis.delete(key):
                        keys_deleted.append(key)

        # Pattern 2 : Données dans des structures partagées
        # Ex: Leaderboard contenant le nom de l'utilisateur
        redis.zrem("leaderboard:global", f"user:{user_id}")
        redis.hdel("user_names", user_id)

        # Pattern 3 : Données associées à l'IP (rate limiting)
        # ⚠️ Attention : Ne supprimer que si l'IP est liée uniquement à cet utilisateur
        user_ips = self._get_user_ips(user_id)
        for ip in user_ips:
            # Vérifier que l'IP n'est pas partagée (NAT, WiFi public)
            if not self._ip_used_by_other_users(ip):
                redis.delete(f"rate_limit:{ip}")

        # Logging pour le rapport
        report.add_step("redis_erasure", {
            "keys_deleted": len(keys_deleted),
            "keys_list": keys_deleted[:50],  # Limiter pour ne pas alourdir
            "timestamp": datetime.now()
        })

        return keys_deleted

    def _mark_user_as_deleted(self, user_id, report):
        """
        Marquer l'utilisateur comme supprimé en DB
        Conserver user_id pour éviter réinscription avec même ID (fraude)
        """
        db.users.update(user_id, {
            "deleted_at": datetime.now(),
            "deletion_reason": report.reason,

            # Anonymisation des champs (plutôt que suppression)
            "email": f"deleted_{user_id}@anonymized.local",
            "name": "Deleted User",
            "phone": None,
            "address": None,

            # Conservation minimale pour intégrité référentielle
            # (si user_id utilisé comme FK ailleurs)
            "user_id": user_id,  # Conserver l'ID

            # Flag pour empêcher réutilisation
            "status": "deleted_gdpr"
        })

        report.add_step("database_anonymization", {
            "user_id": user_id,
            "anonymized": True,
            "timestamp": datetime.now()
        })

    def _schedule_backup_purge(self, user_id, report):
        """
        Gérer l'effacement dans les backups Redis
        """

        # Option 1 : Attendre l'expiration naturelle des backups (30 jours)
        # + Marquer pour non-restauration
        backup_policy.mark_user_for_exclusion(user_id)

        # Option 2 : Purge active des backups (plus complexe)
        # Nécessite de recharger chaque backup, supprimer les clés, re-sauvegarder
        # À éviter sauf exigence réglementaire stricte

        report.add_step("backup_policy", {
            "action": "marked_for_exclusion",
            "retention_until": datetime.now() + timedelta(days=30),
            "note": "Backups will naturally expire in 30 days"
        })

    def _create_audit_trail(self, user_id, report):
        """
        Créer une trace d'audit de l'effacement
        Conservation : 3-5 ans (preuve de conformité RGPD)
        """

        audit_entry = {
            "event": "rgpd_erasure",
            "user_id": user_id,  # ✅ Conserver l'ID pour l'audit (exception Art. 17.3.e)
            "reason": report.reason,
            "requester": report.requester,
            "timestamp": report.timestamp,
            "steps": report.steps,
            "success": report.success,

            # Données supprimées (résumé, pas les données elles-mêmes)
            "data_categories_deleted": [
                "session_data",
                "shopping_cart",
                "analytics",
                "preferences"
            ],

            # Conservation pour preuve
            "retention_until": datetime.now() + timedelta(days=1825)  # 5 ans
        }

        # Stockage sécurisé de l'audit (DB séparée, write-only, chiffrée)
        audit_db.insert(audit_entry)

        # ⚠️ Ne PAS stocker l'audit dans Redis (serait supprimé aussi)

        report.add_step("audit_trail_created", {
            "audit_id": audit_entry["id"],
            "retention_period": "5 years"
        })

    def _notify_subprocessors_for_erasure(self, user_id):
        """
        Notifier les sous-traitants de la demande d'effacement
        (Obligation Article 28.3.e)
        """

        # Ex: Service d'email, analytics tiers, CDN, etc.
        subprocessors = [
            {"name": "EmailService", "api": "/api/delete_user"},
            {"name": "AnalyticsPlatform", "api": "/api/gdpr/erase"},
        ]

        for subprocessor in subprocessors:
            try:
                response = requests.post(
                    subprocessor["api"],
                    json={"user_id": user_id},
                    headers={"Authorization": f"Bearer {SUBPROCESSOR_API_KEY}"}
                )

                if response.status_code == 200:
                    audit_log.info(f"Erasure notified to {subprocessor['name']}")
                else:
                    audit_log.error(f"Failed to notify {subprocessor['name']}: {response.text}")

            except Exception as e:
                audit_log.error(f"Error notifying subprocessor: {e}")

# Exemple d'utilisation
erasure_service = RGPDErasureService()

# Demande utilisateur
report = erasure_service.erase_user_data(
    user_id=12345,
    reason="user_request_right_to_erasure",
    requester="user_self_service"
)

# Génération du rapport pour l'utilisateur
print(report.to_json())
```

**Délais de traitement :**
- Délai de réponse : **1 mois** (Article 12.3)
- Prorogation possible : +2 mois si complexité (avec justification)
- Effacement effectif : **Sans délai indu** (immédiatement dès validation)

**Confirmation à la personne :**
```
Objet : Confirmation de suppression de vos données personnelles

Madame, Monsieur,

Nous accusons réception de votre demande de suppression de vos données
personnelles conformément à l'article 17 du RGPD.

Nous vous confirmons que vos données ont été supprimées de nos systèmes :

✅ Données en cache (Redis) : Supprimées le 11/12/2024 à 14:32 UTC
✅ Base de données principale : Anonymisées le 11/12/2024 à 14:32 UTC
✅ Sauvegardes : Marquées pour exclusion (expiration naturelle dans 30 jours)
✅ Logs d'audit : Anonymisés après 12 mois

Pour des raisons de conformité légale, nous conservons une trace
d'audit de cette suppression pendant 5 ans (article 17.3.e du RGPD).

Si vous avez des questions, contactez notre DPO : dpo@example.com

Cordialement,
L'équipe Confidentialité
```

### 4. Droit à la limitation du traitement (Article 18)

**Obligation :**
"Geler" les données (ne plus les traiter) dans certains cas :
- Contestation de l'exactitude (pendant vérification)
- Traitement illicite mais personne ne veut l'effacement
- Données plus nécessaires mais personne en a besoin pour droits en justice
- Opposition au traitement (pendant vérification)

**Implémentation Redis :**

```python
def limit_user_data_processing(user_id, reason):
    """
    Limiter le traitement des données (RGPD Article 18)
    = Marquage des données, pas d'utilisation active
    """

    # 1. Marquer en DB
    db.users.update(user_id, {
        "processing_limited": True,
        "limitation_reason": reason,
        "limitation_date": datetime.now()
    })

    # 2. ✅ Action Redis : Déplacer vers un namespace "frozen"
    # Pattern : Renommer les clés pour éviter utilisation accidentelle

    user_keys = redis.keys(f"user:{user_id}:*")
    for key in user_keys:
        # Renommer : user:123:profile → limited:user:123:profile
        new_key = f"limited:{key}"
        redis.rename(key, new_key)

        # Supprimer le TTL (conservation tant que limitation active)
        redis.persist(new_key)

    # 3. Interdire l'accès applicatif
    # L'application doit vérifier le flag processing_limited avant tout GET

    # 4. Logging
    audit_log.info(f"Processing limited for user {user_id}: {reason}")

def check_if_processing_limited(user_id):
    """Vérifier avant toute utilisation de données"""
    user = db.users.get(user_id)
    if user.processing_limited:
        raise ProcessingLimitedException(
            f"User {user_id} data processing is limited (GDPR Art. 18)"
        )

# Lever la limitation
def unlimit_user_data_processing(user_id):
    # 1. Retirer le marquage DB
    db.users.update(user_id, {"processing_limited": False})

    # 2. Restaurer les clés Redis
    limited_keys = redis.keys(f"limited:user:{user_id}:*")
    for key in limited_keys:
        original_key = key.replace("limited:", "")
        redis.rename(key, original_key)

        # Restaurer le TTL approprié
        key_type = original_key.split(":")[-1]
        if key_type in TTL_CONFIG:
            redis.expire(original_key, TTL_CONFIG[key_type])
```

### 5. Droit à la portabilité (Article 20)

**Obligation :**
Fournir les données dans un format structuré, couramment utilisé, lisible par machine.

**Implémentation :**

```python
def export_user_data_portable(user_id):
    """
    Export des données au format portable (RGPD Article 20)
    Uniquement les données fournies par l'utilisateur (pas les données dérivées)
    """

    portable_data = {
        "format": "JSON",
        "version": "1.0",
        "exported_at": datetime.now().isoformat(),
        "user_id": user_id,
        "data": {}
    }

    # ✅ Données fournies par l'utilisateur
    user_profile = redis.hgetall(f"user:{user_id}:profile")
    portable_data["data"]["profile"] = {
        "name": user_profile.get("name"),
        "email": user_profile.get("email"),
        "preferences": json.loads(user_profile.get("preferences", "{}"))
    }

    # ✅ Panier d'achat (créé par l'utilisateur)
    cart = redis.hgetall(f"cart:{user_id}")
    portable_data["data"]["shopping_cart"] = cart

    # ❌ Exclure les données dérivées/calculées
    # - Scores de comportement
    # - Recommendations (générées par algorithme)
    # - Logs d'accès (données observées, pas fournies)

    return portable_data

# Transmission directe à un autre responsable si demandé
def transmit_data_to_service(user_id, target_service_api):
    """
    Transmission directe à un autre service (si techniquement possible)
    """
    portable_data = export_user_data_portable(user_id)

    response = requests.post(
        target_service_api,
        json=portable_data,
        headers={"Content-Type": "application/json"}
    )

    if response.status_code == 200:
        audit_log.info(f"Data transmitted for user {user_id} to {target_service_api}")
    else:
        raise TransmissionException(f"Failed to transmit: {response.text}")
```

### 6. Droit d'opposition (Article 21)

**Traitement basé sur intérêt légitime (Art. 6.1.f) ou intérêt public (Art. 6.1.e) :**
→ La personne peut s'opposer (l'organisation doit cesser sauf motif impérieux)

**Marketing direct :**
→ Droit absolu d'opposition (pas de motif impérieux possible)

**Implémentation :**

```python
def handle_user_opposition(user_id, scope):
    """
    Gestion du droit d'opposition (RGPD Article 21)

    scope : "analytics", "marketing", "profiling", etc.
    """

    if scope == "marketing":
        # Droit absolu : Pas de vérification de motif impérieux
        redis.srem("marketing:subscribers", user_id)
        redis.delete(f"marketing:user:{user_id}:*")
        db.users.update(user_id, {"marketing_consent": False})

        audit_log.info(f"User {user_id} opposed to marketing")

    elif scope == "analytics":
        # Vérifier si motif impérieux existe
        # (ex: analytics nécessaires pour facturation, sécurité)
        if not has_legitimate_grounds_for_analytics(user_id):
            redis.delete(f"analytics:user:{user_id}:*")
            redis.srem("analytics:tracked_users", user_id)
            db.users.update(user_id, {"analytics_consent": False})

            audit_log.info(f"User {user_id} opposed to analytics")
        else:
            # Refus motivé avec explication
            return {
                "status": "refused",
                "reason": "Legitimate grounds exist (fraud detection, billing)"
            }

    elif scope == "profiling":
        # Décisions automatisées (Article 22)
        redis.delete(f"profile:user:{user_id}:behavior_score")
        redis.srem("profiling:active_users", user_id)
        db.users.update(user_id, {"profiling_enabled": False})

        audit_log.info(f"User {user_id} opposed to profiling")

    return {"status": "success", "scope": scope}
```

---

## Article 25 : Protection des données dès la conception (Privacy by Design)

### Principes pour Redis

#### 1. Minimisation par défaut

```python
# ❌ Anti-pattern : Tout stocker "au cas où"
redis.hset(f"user:{user_id}:cache", mapping=full_user_object)

# ✅ Pattern : Stocker uniquement le nécessaire
redis.hset(f"user:{user_id}:display", mapping={
    "name": user.name,
    "avatar_url": user.avatar_url
})
```

#### 2. Pseudonymisation

```python
# ✅ Utiliser des identifiants opaques plutôt que des PII
session_id = generate_random_token(32)  # UUID ou token aléatoire
redis.setex(f"session:{session_id}", 3600, {
    "user_id": user.id,  # Référence interne
    # Pas de nom, email, etc. dans la session
})

# ✅ Pseudonymisation des clés sensibles
from hashlib import sha256
def pseudonymize_email(email):
    return sha256(email.encode() + SALT).hexdigest()

email_hash = pseudonymize_email("user@example.com")
redis.setex(f"email_verification:{email_hash}", 300, verification_code)
```

#### 3. Chiffrement applicatif (pour données très sensibles)

```python
from cryptography.fernet import Fernet

class EncryptedRedisCache:
    def __init__(self, redis_client, encryption_key):
        self.redis = redis_client
        self.cipher = Fernet(encryption_key)

    def set(self, key, value, ttl=None):
        # Chiffrer avant stockage
        encrypted = self.cipher.encrypt(json.dumps(value).encode())

        if ttl:
            self.redis.setex(key, ttl, encrypted)
        else:
            self.redis.set(key, encrypted)

    def get(self, key):
        encrypted = self.redis.get(key)
        if not encrypted:
            return None

        # Déchiffrer après récupération
        decrypted = self.cipher.decrypt(encrypted)
        return json.loads(decrypted)

# Utilisation
secure_cache = EncryptedRedisCache(redis, ENCRYPTION_KEY)
secure_cache.set(f"sensitive:{user_id}", {"ssn": "123-45-6789"}, ttl=300)
```

**Note :** Le chiffrement applicatif protège même si Redis est compromis, mais impacte les performances.

#### 4. TTL par défaut

```python
# ✅ Politique : Jamais de clé sans TTL (sauf exception justifiée)
DEFAULT_TTL = 86400  # 24 heures

def safe_redis_set(key, value, ttl=None):
    """Wrapper qui force un TTL par défaut"""
    if ttl is None:
        ttl = DEFAULT_TTL
        logger.warning(f"No TTL specified for {key}, using default {DEFAULT_TTL}s")

    redis.setex(key, ttl, value)
```

#### 5. Séparation des environnements

```
Dev/Staging : Données anonymisées/synthétiques uniquement
Production : Données réelles + sécurité renforcée

❌ Jamais de copie de production vers dev sans anonymisation
```

---

## Article 28 : Sous-traitant

### Contrat de sous-traitance obligatoire

Si vous utilisez un hébergeur cloud pour Redis (AWS, Azure, GCP, Redis Cloud, etc.), vous DEVEZ avoir un contrat de sous-traitance (DPA - Data Processing Agreement).

#### Clauses obligatoires (Article 28.3)

```markdown
## Data Processing Agreement - Redis Hosting

**Entre :**
- Société XYZ (Responsable de traitement)
- CloudProvider Inc. (Sous-traitant)

**Objet :** Hébergement de bases de données Redis contenant des données personnelles

**Article 1 : Objet et durée du traitement**
Le Sous-traitant s'engage à traiter les données personnelles pour le seul
compte du Responsable, conformément aux instructions documentées ci-après.

**Article 2 : Nature et finalités du traitement**
- Nature : Hébergement et maintenance de serveurs Redis
- Finalités : Cache de performance, stockage de sessions utilisateurs
- Catégories de données : Identifiants utilisateurs, préférences, sessions
- Catégories de personnes : Utilisateurs du service web

**Article 3 : Instructions du Responsable**
Le Sous-traitant s'engage à :
a) Ne traiter les données que sur instruction documentée du Responsable
b) Ne pas utiliser les données pour ses propres finalités
c) Garantir que les personnes autorisées sont soumises à la confidentialité

**Article 4 : Mesures de sécurité (Article 32 RGPD)**
Le Sous-traitant met en œuvre :
☑ Chiffrement TLS 1.3
☑ Authentification multi-facteurs pour les accès admin
☑ Segmentation réseau (VPC privé)
☑ Backups chiffrés quotidiens
☑ Monitoring 24/7 avec alerting
☑ Tests de pénétration annuels
☑ Certification SOC 2 Type II / ISO 27001

**Article 5 : Sous-traitance ultérieure**
Le Sous-traitant ne peut recourir à un autre sous-traitant qu'avec
l'autorisation écrite préalable du Responsable.

Liste des sous-traitants autorisés :
- AWS (infrastructure sous-jacente)
- CloudFlare (CDN et protection DDoS)

Le Responsable sera informé de tout changement (ajout/remplacement)
avec un délai de 30 jours pour s'opposer.

**Article 6 : Assistance aux droits des personnes**
Le Sous-traitant assiste le Responsable dans la mesure du possible
pour répondre aux demandes d'exercice des droits :
- Droit d'accès (Article 15)
- Droit de rectification (Article 16)
- Droit à l'effacement (Article 17)
- Etc.

Délai d'assistance : 5 jours ouvrés

**Article 7 : Notification des violations**
Le Sous-traitant notifie le Responsable de toute violation de données
dans les 24 heures suivant la découverte.

La notification comprend :
- Nature de la violation
- Données concernées
- Mesures de mitigation prises
- Point de contact

**Article 8 : Audits et inspections**
Le Responsable (ou auditeur mandaté) peut :
- Demander la preuve de conformité (certifications SOC 2, rapports d'audit)
- Effectuer des audits sur site (avec préavis de 30 jours)
- Accéder aux logs et configurations Redis (via interface sécurisée)

Fréquence : 1 audit complet par an minimum

**Article 9 : Fin du contrat**
À la fin du contrat, le Sous-traitant s'engage à :
a) Supprimer toutes les données personnelles OU
b) Restituer les données au Responsable (export sécurisé)
c) Fournir une attestation de suppression

Délai : 30 jours après fin du contrat

**Article 10 : Localisation des données**
Les données sont hébergées exclusivement en Union Européenne :
- Région primaire : eu-west-3 (Paris, France)
- Région backup : eu-west-1 (Irlande)

Tout transfert hors UE nécessite un mécanisme de transfert approprié
(clauses contractuelles types, décision d'adéquation).

**Article 11 : Responsabilité**
Le Sous-traitant est pleinement responsable envers le Responsable
des dommages causés par un traitement non conforme au RGPD.
```

### Transferts hors UE (Chapitre V RGPD)

Si vos données Redis quittent l'UE (ex: réplication cross-region vers USA), vous DEVEZ avoir un mécanisme de transfert légal :

#### Options post-Schrems II :

1. **Décision d'adéquation** (Article 45)
   - Liste des pays : Royaume-Uni, Suisse, Israël, Japon, etc.
   - Transfert libre comme au sein de l'UE

2. **Clauses Contractuelles Types (CCT)** (Article 46)
   - Modèle fourni par la Commission Européenne
   - + Transfer Impact Assessment (TIA) obligatoire
   - Évaluer les lois locales (ex: CLOUD Act USA, lois surveillance Chine)

3. **Binding Corporate Rules (BCR)** (Article 47)
   - Pour les groupes multinationaux

4. **Dérogations** (Article 49)
   - Consentement explicite de la personne
   - Nécessité contractuelle
   - Intérêt public important
   → Usage exceptionnel uniquement

**Exemple de TIA pour Redis :**
```
Transfer Impact Assessment - Redis Replication USA

1. Pays destinataire : États-Unis
2. Base légale du transfert : Clauses Contractuelles Types (Module 2)
3. Législation locale :
   - CLOUD Act : Permet accès US aux données sur demande DOJ
   - FISA 702 : Surveillance sans mandat pour non-US persons

4. Évaluation du risque :
   - Probabilité d'accès : Faible (données non stratégiques)
   - Impact : Moyen (données personnelles utilisateurs EU)

5. Garanties supplémentaires :
   ☑ Chiffrement end-to-end (TLS 1.3)
   ☑ Chiffrement at-rest (AES-256)
   ☑ Minimisation des données transférées (uniquement sessions)
   ☑ Pseudonymisation (user_id opaques, pas de PII directes)
   ☑ Clause contractuelle : Notification en cas de demande gouvernementale
   ☑ Recours juridique prévu (contester les demandes injustifiées)

6. Conclusion :
   Le transfert peut être effectué avec les garanties additionnelles ci-dessus.
   Revue annuelle obligatoire de la TIA.
```

---

## Article 32 : Sécurité du traitement

### Mesures techniques et organisationnelles

**État de l'art applicable à Redis :**

#### Mesures techniques minimales

```yaml
Network Security:
  ☑ Bind sur interfaces privées uniquement (pas 0.0.0.0)
  ☑ Firewall avec allow-list stricte (IP sources autorisées)
  ☑ Segmentation réseau (VPC / VLAN dédiés)
  ☑ Pas d'accès direct depuis Internet
  ☑ VPN ou bastion host pour accès admin

Authentication & Authorization:
  ☑ ACL Redis 6+ (comptes nominatifs)
  ☑ Mots de passe forts (>16 caractères, complexité)
  ☑ MFA pour accès administratif
  ☑ Rotation trimestrielle des credentials
  ☑ Pas de compte sans mot de passe (protected-mode yes)

Encryption:
  ☑ TLS 1.3 pour toutes les connexions (clients + inter-nœuds)
  ☑ Certificats valides (pas auto-signés en production)
  ☑ Chiffrement at-rest (filesystem chiffré ou Redis Enterprise)
  ☑ Backups chiffrés (AES-256)

Data Protection:
  ☑ TTL systématiques (limitation conservation)
  ☑ Politique d'éviction configurée (maxmemory-policy)
  ☑ Backups réguliers (RPO < 24h)
  ☑ Tests de restauration mensuels
  ☑ Réplication pour haute disponibilité

Monitoring & Logging:
  ☑ Centralisation des logs (SIEM)
  ☑ Audit logs (commandes critiques tracées)
  ☑ Alerting sur événements suspects
  ☑ Détection d'anomalies (ML-based si possible)

Hardening:
  ☑ Commandes dangereuses désactivées (rename-command)
  ☑ Version Redis à jour (patches de sécurité < 30j)
  ☑ OS durci (CIS benchmark)
  ☑ Principe du moindre privilège (ACL granulaires)
```

#### Mesures organisationnelles

```
Politique de sécurité:
  ☑ Politique Redis documentée et approuvée
  ☑ Revue annuelle de la politique
  ☑ Classification des données (voir section précédente)

Gestion des accès:
  ☑ Procédure de provisioning/deprovisioning
  ☑ Revue semestrielle des droits d'accès
  ☑ Ségrégation des duties (dev vs prod)

Formation:
  ☑ Formation annuelle RGPD pour tous
  ☑ Formation spécialisée Redis pour DevOps
  ☑ Tests de phishing réguliers

Incident Response:
  ☑ Plan de réponse aux incidents documenté
  ☑ Équipe dédiée (CSIRT)
  ☑ Exercices de simulation (war games)
  ☑ Procédure de notification 72h (RGPD)

Change Management:
  ☑ Processus formel de validation des changements
  ☑ Tests en non-prod obligatoires
  ☑ Rollback plan systématique

Audit & Compliance:
  ☑ Audits internes trimestriels
  ☑ Pentest externe annuel
  ☑ Certifications (ISO 27001, SOC 2)
```

### Tests de sécurité réguliers

**Programme de tests minimum :**

```python
# Checklist tests de sécurité Redis

# 1. Tests d'authentification
□ Tentative de connexion sans mot de passe (doit échouer)
□ Tentative avec credentials invalides (doit échouer + logger)
□ Brute force protection (rate limiting fonctionne ?)

# 2. Tests d'autorisation
□ Utilisateur read-only peut-il écrire ? (doit échouer)
□ Utilisateur non-admin peut-il CONFIG SET ? (doit échouer)
□ ACL par pattern de clés fonctionne ?

# 3. Tests réseau
□ Tentative de connexion depuis IP non autorisée (doit échouer)
□ Redis répond-il depuis Internet ? (NON)
□ TLS obligatoire ? (connexion sans TLS doit échouer)

# 4. Tests de persistance
□ Backup automatique fonctionne ?
□ Restauration d'un backup réussit ?
□ Backup est-il chiffré ?

# 5. Tests de résilience
□ Failover automatique (Sentinel) fonctionne ?
□ Données préservées après redémarrage ?
□ Réplication synchronisée ?

# 6. Tests de monitoring
□ Alerte déclenchée si authentification échouée x fois ?
□ Alerte déclenchée si utilisation mémoire > seuil ?
□ Logs correctement centralisés dans SIEM ?
```

---

## Article 33 : Notification de violation de données

### Obligation de notification (72 heures)

**Déclencheurs d'une violation :**
- Accès non autorisé à Redis (intrusion)
- Perte de données (corruption, suppression malveillante)
- Modification non autorisée (intégrité compromise)
- Fuite de backup non chiffré
- Réplication vers une destination non autorisée

### Procédure de notification

```python
class DataBreachNotificationService:
    """
    Gestion des notifications de violation RGPD (Article 33)
    """

    NOTIFICATION_DEADLINE_HOURS = 72

    def detect_breach(self, incident_type, details):
        """
        Détection d'une violation
        """

        breach = DataBreach(
            id=generate_breach_id(),
            incident_type=incident_type,
            detected_at=datetime.now(),
            details=details,
            status="detected"
        )

        # 1. Évaluation de la gravité
        severity = self._assess_severity(breach)
        breach.severity = severity

        # 2. Si grave : Notification immédiate RSSI + DPO
        if severity in ["high", "critical"]:
            self._alert_security_team(breach)

        # 3. Containment (isoler l'incident)
        self._contain_breach(breach)

        # 4. Investigation
        investigation_report = self._investigate(breach)
        breach.investigation = investigation_report

        # 5. Décision de notification
        requires_notification = self._requires_cnil_notification(breach)

        if requires_notification:
            self._notify_supervisory_authority(breach)

            # 6. Notification personnes concernées si risque élevé
            if self._high_risk_to_individuals(breach):
                self._notify_affected_individuals(breach)

        return breach

    def _assess_severity(self, breach):
        """
        Évaluation de la gravité selon critères RGPD
        """

        score = 0

        # 1. Type de données compromises
        if breach.involves_special_categories():  # Art. 9 (santé, etc.)
            score += 4
        elif breach.involves_financial_data():
            score += 3
        elif breach.involves_credentials():
            score += 3
        elif breach.involves_identifiers_only():
            score += 1

        # 2. Volume de personnes affectées
        affected_count = breach.count_affected_individuals()
        if affected_count > 100000:
            score += 3
        elif affected_count > 10000:
            score += 2
        elif affected_count > 1000:
            score += 1

        # 3. Nature de la violation
        if breach.type == "unauthorized_access":
            score += 3
        elif breach.type == "data_loss":
            score += 2
        elif breach.type == "data_modification":
            score += 2

        # 4. Mesures de protection existantes
        if not breach.data_was_encrypted():
            score += 2
        if not breach.data_was_pseudonymized():
            score += 1

        # Classification
        if score >= 8:
            return "critical"
        elif score >= 5:
            return "high"
        elif score >= 3:
            return "medium"
        else:
            return "low"

    def _requires_cnil_notification(self, breach):
        """
        Déterminer si notification CNIL obligatoire (72h)
        """

        # Notification obligatoire sauf si :
        # "Il est improbable que la violation engendre un risque
        #  pour les droits et libertés des personnes" (Art. 33.1)

        # Facteurs de risque :
        risk_factors = []

        if not breach.data_was_encrypted():
            risk_factors.append("Données non chiffrées")

        if breach.involves_special_categories():
            risk_factors.append("Données sensibles (Art. 9)")

        if breach.count_affected_individuals() > 100:
            risk_factors.append("Nombre élevé de personnes")

        if breach.type == "unauthorized_access":
            risk_factors.append("Accès non autorisé (risque d'usurpation)")

        # Si au moins un facteur : Notification obligatoire
        return len(risk_factors) > 0

    def _high_risk_to_individuals(self, breach):
        """
        Risque élevé = Notification aux personnes obligatoire (Art. 34)
        """

        # Ex: Vol de credentials, données santé non chiffrées, etc.

        high_risk_conditions = [
            breach.severity == "critical",
            breach.involves_special_categories() and not breach.data_was_encrypted(),
            breach.type == "unauthorized_access" and breach.involves_credentials(),
        ]

        return any(high_risk_conditions)

    def _notify_supervisory_authority(self, breach):
        """
        Notification à l'autorité de contrôle (CNIL en France)
        Délai : 72 heures (Article 33)
        """

        # Vérifier le délai
        hours_since_detection = (datetime.now() - breach.detected_at).total_seconds() / 3600

        if hours_since_detection > self.NOTIFICATION_DEADLINE_HOURS:
            logger.critical(f"⚠️ 72h deadline exceeded for breach {breach.id}")
            # Justification du retard obligatoire

        # Contenu de la notification (Article 33.3)
        notification = {
            # a) Nature de la violation
            "nature": breach.incident_type,
            "description": breach.details,

            # b) Contact DPO
            "dpo_contact": {
                "name": "Jane Doe",
                "email": "dpo@example.com",
                "phone": "+33123456789"
            },

            # c) Conséquences probables
            "consequences": breach.assess_consequences(),

            # d) Mesures prises ou envisagées
            "remediation": {
                "containment": "Redis instance isolated from network",
                "investigation": "Forensic analysis ongoing",
                "prevention": "ACL hardening, TLS enforcement",
                "notification_individuals": "Planned if high risk confirmed"
            },

            # Informations complémentaires
            "affected_individuals_count": breach.count_affected_individuals(),
            "data_categories": breach.list_data_categories(),
            "detected_at": breach.detected_at.isoformat(),
            "reported_at": datetime.now().isoformat(),
        }

        # Envoi via formulaire CNIL ou email sécurisé
        cnil_api.submit_breach_notification(notification)

        # Logging
        audit_log.info(f"Breach {breach.id} notified to supervisory authority", extra=notification)

    def _notify_affected_individuals(self, breach):
        """
        Notification aux personnes concernées (Article 34)
        Uniquement si risque élevé
        """

        affected_users = breach.get_affected_user_ids()

        for user_id in affected_users:
            user = db.users.get(user_id)

            email_content = f"""
Objet : Information importante concernant vos données personnelles

Madame, Monsieur,

Nous vous informons qu'un incident de sécurité a affecté nos systèmes
et pourrait concerner vos données personnelles.

**Nature de l'incident :**
{breach.get_user_friendly_description()}

**Données potentiellement concernées :**
{breach.list_affected_data_categories_for_user(user_id)}

**Conséquences possibles :**
{breach.explain_consequences_for_user()}

**Mesures que vous pouvez prendre :**
- Changez votre mot de passe immédiatement
- Activez l'authentification à deux facteurs
- Surveillez vos comptes pour détecter toute activité suspecte

**Mesures que nous avons prises :**
- Isolation immédiate des systèmes compromis
- Renforcement de la sécurité (TLS, ACL, monitoring)
- Investigation forensique en cours
- Notification à la CNIL conformément au RGPD

**Vos droits :**
Conformément au RGPD, vous pouvez exercer vos droits (accès, effacement, etc.)
en nous contactant à privacy@example.com.

Pour toute question, contactez notre DPO : dpo@example.com

Nous nous excusons sincèrement pour cet incident et prenons toutes
les mesures nécessaires pour éviter qu'il ne se reproduise.

Cordialement,
[Signature]
            """

            send_email(user.email, "⚠️ Information importante - Sécurité de vos données", email_content)

            # Notification in-app également
            create_notification(user_id, "security_breach", email_content)

# Exemple de détection automatique
def monitor_redis_for_breaches():
    """
    Monitoring continu pour détecter les violations
    """

    # 1. Authentifications échouées massives
    failed_auths = redis.get("security:failed_auths:count")
    if int(failed_auths) > 100:
        breach_service.detect_breach("potential_brute_force", {
            "failed_attempts": failed_auths,
            "timeframe": "last_hour"
        })

    # 2. Commandes suspectes (via logs)
    suspicious_commands = ["CONFIG", "FLUSHALL", "SCRIPT FLUSH"]
    for cmd in suspicious_commands:
        if check_logs_for_unauthorized_command(cmd):
            breach_service.detect_breach("unauthorized_command_execution", {
                "command": cmd,
                "detected_in": "redis.log"
            })

    # 3. Connexions depuis IP non autorisées
    # (nécessite parsing des logs ou proxy avec audit)
```

---

## Article 35 : Analyse d'impact (DPIA)

### Quand effectuer une DPIA pour Redis ?

**DPIA obligatoire si (Article 35.3) :**
1. Évaluation systématique et approfondie basée sur traitement automatisé (profilage)
2. Traitement à grande échelle de données sensibles (Art. 9) ou pénales
3. Surveillance systématique à grande échelle d'une zone accessible au public

**Exemples Redis nécessitant DPIA :**
- Tracking comportemental de millions d'utilisateurs pour scoring crédit
- Stockage de données de santé (diagnostics, traitements)
- Surveillance géolocalisée en temps réel
- Profilage pour décisions automatisées (refus crédit, tarification)

**Pas de DPIA requise (généralement) :**
- Cache de sessions utilisateurs (service classique)
- Panier d'achat temporaire
- Rate limiting anti-abus
- Leaderboard de jeu vidéo (pseudonymes)

### Template de DPIA pour Redis

```markdown
# Analyse d'Impact relative à la Protection des Données (DPIA)
# Traitement : [NOM DU TRAITEMENT REDIS]

**Date :** 11/12/2024
**Version :** 1.0
**Responsable :** [Nom du responsable de traitement]
**DPO :** [Contact DPO]

---

## 1. Description du traitement

### 1.1 Contexte et finalités
**Finalités :**
- [Ex: Personnalisation de l'expérience utilisateur basée sur historique]
- [Ex: Recommandations produits en temps réel]

**Base légale :**
- Article 6.1.f (Intérêt légitime) : Améliorer l'expérience utilisateur

### 1.2 Données traitées
| Catégorie | Exemples | Sensibilité |
|-----------|----------|-------------|
| Identifiants | user_id, session_id | Moyenne |
| Comportement | Pages visitées, clics, durées | Moyenne |
| Préférences | Langue, catégories favorites | Faible |

**Volume :** ~5 millions d'utilisateurs

### 1.3 Architecture technique
```
[Application Web] → TLS 1.3 → [Redis Cluster 6 nœuds]
                                  ↓
                          [Backups S3 chiffrés]
```

**Localisation :** Union Européenne (France)
**Durée de conservation :** 90 jours (TTL automatique)

---

## 2. Nécessité et proportionnalité

### 2.1 Le traitement est-il nécessaire ?
☑ Oui
Justification : Sans historique comportemental, impossible de personnaliser
l'expérience (ex: recommandations pertinentes, interface adaptée).

### 2.2 Le traitement est-il proportionné ?
☑ Oui
Mesures de proportionnalité :
- Données minimales (uniquement IDs pages, pas contenu)
- TTL court (90 jours, pas conservation indéfinie)
- Pseudonymisation (user_id opaque, pas nom/email dans Redis)
- Droit d'opposition facile (toggle dans paramètres)

### 2.3 Alternatives envisagées
| Alternative | Raison du rejet |
|-------------|----------------|
| Pas de personnalisation | Expérience utilisateur dégradée (compétitivité) |
| Stockage en DB principale | Performances inacceptables (latence > 500ms) |
| Anonymisation complète | Impossible de personnaliser (perte de lien user) |

---

## 3. Risques pour les personnes

### 3.1 Identification des risques

| Risque | Probabilité | Impact | Niveau |
|--------|-------------|--------|--------|
| **R1 : Accès non autorisé** | Faible | Élevé | Modéré |
| Description : Attaquant accède à Redis et exfiltre les historiques |
| Conséquences : Exposition préférences utilisateurs, possible profilage malveillant |

| **R2 : Profilage abusif** | Moyenne | Moyen | Modéré |
| Description : Utilisation des données pour discrimination (tarification, publicité) |
| Conséquences : Atteinte à l'équité, manipulation comportementale |

| **R3 : Perte de contrôle** | Faible | Moyen | Faible |
| Description : Utilisateur ignore l'utilisation de ses données comportementales |
| Conséquences : Manque de transparence, confiance érodée |

| **R4 : Réidentification** | Faible | Élevé | Modéré |
| Description : Recoupement avec d'autres sources pour réidentifier utilisateurs |
| Conséquences : Pseudonymisation contournée, vie privée compromise |

---

## 4. Mesures de protection

### 4.1 Mesures techniques

| Mesure | Risque atténué | Statut |
|--------|----------------|--------|
| **TLS 1.3 + Certificats valides** | R1 (Accès non autorisé) | ✅ Implémenté |
| **ACL Redis granulaires** | R1 | ✅ Implémenté |
| **Firewall strict (allow-list)** | R1 | ✅ Implémenté |
| **Chiffrement at-rest (S3)** | R1 | ✅ Implémenté |
| **Pseudonymisation (user_id opaque)** | R4 (Réidentification) | ✅ Implémenté |
| **TTL automatique (90j)** | R2, R4 | ✅ Implémenté |
| **Minimisation données** | R2, R4 | ✅ Implémenté |
| **Monitoring + Alerting** | R1 | ✅ Implémenté |

### 4.2 Mesures organisationnelles

| Mesure | Risque atténué | Statut |
|--------|----------------|--------|
| **Politique de confidentialité claire** | R3 (Perte de contrôle) | ✅ Implémenté |
| **Consentement ou droit d'opposition** | R2, R3 | ✅ Implémenté |
| **Formation équipes RGPD** | Tous | ✅ Annuel |
| **Audit annuel** | Tous | ✅ Planifié |
| **Tests de pénétration** | R1 | ✅ Annuel |
| **Procédure incident < 72h** | R1 | ✅ Documenté |

---

## 5. Consultation des parties prenantes

### 5.1 DPO consulté
☑ Oui
**Avis :** Conforme sous réserve de mise en œuvre intégrale des mesures ci-dessus.
**Date :** 01/12/2024

### 5.2 Représentants des personnes concernées
☐ Non applicable (pas de représentants)
☑ Tests utilisateurs effectués
**Feedback :** 82% apprécient la personnalisation, 15% désactivent (droit respecté)

---

## 6. Validation et décision

### 6.1 Risque résiduel acceptable ?
☑ Oui

Après mise en œuvre des mesures :
- R1 : Faible (multiples couches de sécurité)
- R2 : Faible (usage limité aux recommendations, pas décisions automatisées critiques)
- R3 : Très faible (transparence + droit d'opposition)
- R4 : Faible (pseudonymisation + minimisation)

### 6.2 Décision
☑ Traitement autorisé avec les mesures définies
☐ Traitement autorisé sous conditions
☐ Traitement refusé

**Validé par :** [CISO]
**Date :** 11/12/2024

### 6.3 Revue
**Fréquence de revue :** Annuelle OU en cas de changement majeur
**Prochaine revue :** 11/12/2025

---

## 7. Documentation et archivage

Cette DPIA est conservée conformément à l'Article 5.2 (accountability)
et disponible sur demande de l'autorité de contrôle.

**Référence :** DPIA-REDIS-2024-001
**Stockage :** Système de gestion documentaire sécurisé
```

---

## Checklist de conformité RGPD finale

### Phase préparatoire

```
□ Inventaire de tous les traitements Redis effectué
□ Bases légales identifiées pour chaque traitement
□ Classification des données (public, confidentiel, sensible)
□ DPIA effectuée si nécessaire
□ Registre des traitements (Article 30) à jour
□ DPO désigné (si obligatoire) ou contact confidentialité défini
```

### Transparence et droits

```
□ Politique de confidentialité mentionne Redis
□ Finalités, durées, destinataires documentés
□ Procédures d'exercice des droits implémentées :
  □ Droit d'accès (export Redis)
  □ Droit de rectification (invalidation cache)
  □ Droit à l'effacement (suppression complète)
  □ Droit à la limitation (freezing des données)
  □ Droit à la portabilité (export JSON)
  □ Droit d'opposition (opt-out analytics)
□ Délais de réponse respectés (1 mois)
□ Formulaires de demande accessibles
```

### Sécurité technique

```
□ TLS 1.3 activé (clients et inter-nœuds)
□ ACL configurées (comptes nominatifs, moindre privilège)
□ Authentification forte (pas de passwordless)
□ Firewall restrictif (allow-list uniquement)
□ Commandes dangereuses désactivées
□ Chiffrement backups (AES-256)
□ TTL systématiques (limitation conservation)
□ Monitoring et alerting configurés
□ Backups testés mensuellement
```

### Sécurité organisationnelle

```
□ Politique de sécurité Redis approuvée
□ Formation RGPD annuelle des équipes
□ Procédures documentées (provisioning, effacement, etc.)
□ Gestion des changements formalisée
□ Revue des accès semestrielle
□ Tests de sécurité réguliers (pentests annuels)
□ Plan de réponse aux incidents < 72h
□ Contrat de sous-traitance (Article 28) si cloud
```

### Conformité continue

```
□ Audits internes trimestriels
□ Revue annuelle de la DPIA (si applicable)
□ Veille réglementaire (changements RGPD)
□ Logs d'audit conservés (12 mois minimum)
□ Documentation à jour
□ Registre des violations tenu
□ Certification (ISO 27001, SOC 2) si applicable
```

---

## Ressources et documentation

### Textes officiels

- **RGPD complet** : https://eur-lex.europa.eu/eli/reg/2016/679/oj
- **Lignes directrices CEPD** : https://edpb.europa.eu/our-work-tools/general-guidance_fr
- **Guidelines CNIL** : https://www.cnil.fr/fr/reglement-europeen-protection-donnees

### Outils pratiques

- **Générateur DPIA CNIL** : https://www.cnil.fr/fr/outil-pia-telechargez-et-installez-le-logiciel-de-la-cnil
- **Modèles de clauses contractuelles** : https://ec.europa.eu/info/law/law-topic/data-protection/international-dimension-data-protection/standard-contractual-clauses-scc_fr
- **Registre des traitements (template)** : https://www.cnil.fr/fr/RGDP-le-registre-des-activites-de-traitement

### Formation

- **MOOC CNIL** : https://atelier-rgpd.cnil.fr/
- **Certification DPO** : https://www.cnil.fr/fr/devenir-delegue-la-protection-des-donnees

---

**Cette section établit les fondations de la conformité RGPD pour Redis. Les compliance officers et architectes disposent maintenant d'un cadre complet pour auditer et sécuriser leurs instances Redis conformément à la réglementation européenne.**

**Points clés à retenir :**
1. Redis est pleinement soumis au RGPD (pas d'exception pour les caches)
2. Chaque traitement nécessite une base légale (Article 6)
3. Les droits des personnes doivent être implémentés (Articles 15-22)
4. La sécurité est obligatoire (Article 32) : TLS + ACL + TTL
5. Les violations doivent être notifiées sous 72h (Article 33)
6. La conformité est continue, pas ponctuelle (audits réguliers)

⏭️ [Encryption at rest et in transit](/17-gouvernance-conformite/02-encryption-at-rest-in-transit.md)
