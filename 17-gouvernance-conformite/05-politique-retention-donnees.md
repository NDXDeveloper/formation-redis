🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.5 Politique de Rétention des Données

## Introduction

La rétention des données est un équilibre délicat entre les obligations légales de conservation et les impératifs de minimisation des données. Une politique de rétention bien conçue doit répondre à trois questions fondamentales : **combien de temps conserver**, **pourquoi conserver**, et **quand supprimer**. Pour Redis, base de données in-memory souvent utilisée pour des données temporaires, cette politique prend une dimension particulière avec les mécanismes de TTL (Time To Live) et d'éviction automatique.

**Définition de la politique de rétention :**
> Ensemble de règles documentées qui définissent les durées de conservation des différentes catégories de données, les motifs de conservation, les responsabilités, et les procédures de purge, conformément aux exigences légales et aux besoins métier.

---

## Cadre réglementaire de la rétention des données

### RGPD - Article 5 : Principes relatifs au traitement

**Article 5.1.e : Limitation de la conservation**
> "Les données à caractère personnel doivent être conservées sous une forme permettant l'identification des personnes concernées pendant une durée n'excédant pas celle nécessaire au regard des finalités pour lesquelles elles sont traitées."

**Implications pour Redis :**
```
✅ Obligation de définir des durées de conservation maximales
✅ Justification métier et légale de chaque durée
✅ Suppression automatique dès que la finalité est atteinte
✅ Documentation de la politique de rétention
❌ Conservation indéfinie interdite (sauf exceptions légales)
```

**Exceptions à la limitation (Article 5.1.e) :**
```
Conservation plus longue autorisée si :
a) Archivage dans l'intérêt public
b) Recherche scientifique ou historique
c) Fins statistiques (avec garanties appropriées)
```

**Article 17.1.a : Droit à l'effacement**
> "La personne concernée a le droit d'obtenir du responsable du traitement l'effacement [...] lorsque les données à caractère personnel ne sont plus nécessaires au regard des finalités pour lesquelles elles ont été collectées."

**Traduction technique pour Redis :**
- TTL automatique aligné sur la durée nécessaire
- Procédure de purge proactive
- Pas de conservation "au cas où"

**Considérations pratiques CNIL :**
```
La CNIL recommande :
- Durées de rétention définies par type de donnée
- Distinction entre "base active" et "archivage"
- Anonymisation plutôt que suppression si besoin statistique
- Documentation dans le registre des traitements (Article 30)
```

### PCI DSS - Requirement 3 : Protect Stored Cardholder Data

**3.1 : Keep cardholder data storage to a minimum**
> "Limiter le stockage des données de carte au minimum nécessaire pour les besoins métier et légaux."

**3.1.1 : Data retention and disposal policies**
```
Exigences :
□ Politique documentée de rétention
□ Durées de conservation définies et justifiées
□ Revue trimestrielle des données stockées
□ Purge automatique des données expirées
□ Destruction sécurisée (pas de récupération possible)
```

**Durées maximales PCI DSS :**
```
┌──────────────────────────────────────────────────────────────┐
│ Type de donnée           │ Durée max PCI DSS                 │
├──────────────────────────┼───────────────────────────────────┤
│ PAN (Primary Account     │ Minimum requis (ex: 90 jours pour │
│ Number)                  │ chargebacks, sauf obligation      │
│                          │ légale supérieure)                │
│ CVV/CVC                  │ ❌ JAMAIS (interdiction absolue)  │
│ PIN/PIN block            │ ❌ JAMAIS (interdiction absolue)  │
│ Track data (magnetic)    │ ❌ JAMAIS après autorisation      │
│ Logs d'audit             │ 12 mois minimum                   │
└──────────────────────────────────────────────────────────────┘

⚠️ Redis ne devrait JAMAIS stocker CVV/PIN même temporairement
```

**3.1.2 : Limit data retention time**
> "Conserver les données de carte uniquement le temps nécessaire aux besoins métier ou légaux."

### HIPAA - Records Retention

**§164.316(b)(2)(i) : Retention period**
> "Conserver la documentation requise pendant 6 ans à partir de la date de sa création ou de la date à laquelle elle était en vigueur pour la dernière fois, selon la date la plus tardive."

**Application à Redis :**
```
Données PHI dans Redis :
- Conservation active : Durée minimale nécessaire (ex: session 24h)
- Backups : 6 ans minimum
- Logs d'audit : 6 ans minimum
- Documentation politique : 6 ans après dernière modification

Note : Les états peuvent avoir des exigences supérieures
(ex: Californie impose parfois 7 ans)
```

**Particularité HIPAA :**
```
La loi impose la durée de conservation de la DOCUMENTATION,
pas nécessairement des données PHI elles-mêmes.

Cependant, pour des raisons médicales et légales, les PHI sont
souvent conservées beaucoup plus longtemps (10-30 ans selon contexte).

Redis : Utilisé pour données temporaires (sessions, cache)
         Durée courte justifiée si pas de finalité long terme
```

### SOC 2 - Data Lifecycle Management

**CC6.5 : Logical and Physical Access Controls**
> "L'entité retire l'accès aux données lorsqu'il n'est plus nécessaire."

**Implication :**
- Suppression automatique des données obsolètes
- Pas d'accumulation de données "zombie"
- Politique de purge documentée et testée

**A1.2 : Data Retention and Disposal (si applicable)**
```
Pour organisations traitant des données clients :
□ Politique de rétention communiquée aux clients
□ Respect des engagements contractuels
□ Destruction sécurisée certifiée
□ Logs de destruction conservés
```

### ISO 27001 - Annexe A.8.3

**A.8.3.2 : Disposal of media (Required)**
> "Les supports contenant des informations devraient être éliminés de manière sécurisée lorsqu'ils ne sont plus nécessaires."

**A.8.3.3 : Physical media transfer (Required)**
> "Les supports contenant des informations devraient être protégés contre accès non autorisé, mauvais usage ou corruption durant le transport."

**Application Redis :**
```
□ Suppression sécurisée des RDB/AOF expirés (shred, degauss)
□ Backups chiffrés avec destruction sécurisée en fin de vie
□ Pas de transfert de dumps non chiffrés
□ Procédure de destruction documentée
```

### Lois sectorielles et territoriales

**France - LCEN (Loi pour la Confiance dans l'Économie Numérique)**
```
Article 6.II : Conservation des logs de connexion
Durée : 12 mois minimum (hébergeurs, fournisseurs d'accès)
Objectif : Identification en cas d'infraction
```

**États-Unis - State Privacy Laws**
```
CCPA/CPRA (Californie) :
- Droit à la suppression des données personnelles
- Délai de réponse : 45 jours (prorogeable 45j)
- Exceptions : Obligations légales, sécurité, transactions

VCDPA (Virginie), CPA (Colorado), etc. : Exigences similaires
```

**Secteur bancaire (ex: France)**
```
Code monétaire et financier :
- Pièces comptables : 10 ans
- Contrats de crédit : 5 ans après fin du contrat
- Relevés bancaires : 5 ans

Impact Redis : Si cache de transactions, durée limitée
                Documentation de la non-rétention long terme
```

---

## Principes de rétention des données

### Classification des données et durées associées

**Matrice de rétention par catégorie :**

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Catégorie        │ Exemple Redis         │ Durée    │ Base légale          │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Session          │ session:*             │ 24h      │ Contrat (nécessaire) │
│                  │ user_session:{id}     │          │                      │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Cache temporaire │ cache:product:*       │ 1-24h    │ Intérêt légitime     │
│                  │ cache:api_response:*  │          │ (performance)        │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Panier d'achat   │ cart:{user_id}        │ 30j      │ Contrat (commande    │
│                  │                       │          │ en cours)            │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Token 2FA        │ 2fa:token:{user}      │ 5min     │ Sécurité (éphémère)  │
│                  │ verification_code:*   │          │                      │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Rate limiting    │ rate_limit:{ip}       │ 1h       │ Intérêt légitime     │
│                  │ throttle:{user_id}    │          │ (protection abus)    │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Analytics        │ analytics:views:*     │ 90j      │ Consentement         │
│ comportementales │ user_behavior:{id}    │          │                      │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Leaderboard      │ leaderboard:monthly   │ 12 mois  │ Contrat (service de  │
│                  │                       │          │ jeu)                 │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Job queue        │ queue:jobs:pending    │ 7j       │ Contrat (traitement  │
│                  │ queue:jobs:processed  │          │ asynchrone)          │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Lock distribué   │ lock:{resource}       │ 30s-5min │ Technique (non PII)  │
│                  │                       │          │                      │
├──────────────────┼───────────────────────┼──────────┼──────────────────────┤
│ Feature flags    │ feature:{flag}:{user} │ Illimité │ Technique (config)   │
│                  │                       │ ou 90j   │                      │
└────────────────────────────────────────────────────────────────────────────┘
```

**Règle d'or :**
> La durée de rétention dans Redis doit être la **plus courte possible** pour atteindre la finalité. Si conservation long terme nécessaire, déplacer vers une base de données permanente.

### Distinction entre conservation active et archivage

**Trois états du cycle de vie des données :**

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. ACTIVE (Redis in-memory)                                          │
├──────────────────────────────────────────────────────────────────────┤
│ - Données accédées fréquemment                                       │
│ - Performance critique (latence < 1ms)                               │
│ - Durée : Strictement limitée à la finalité active                   │
│ - Exemple : Session utilisateur (24h)                                │
└──────────────────────────────────────────────────────────────────────┘
                            ↓ Fin de l'usage actif
┌──────────────────────────────────────────────────────────────────────┐
│ 2. ARCHIVAGE (DB permanente ou cold storage)                         │
├──────────────────────────────────────────────────────────────────────┤
│ - Données rarement accédées                                          │
│ - Obligation légale ou contractuelle de conservation                 │
│ - Performance non critique                                           │
│ - Durée : Selon obligation (ex: 5 ans, 10 ans)                       │
│ - Exemple : Historique transactions (archivage comptable)            │
└──────────────────────────────────────────────────────────────────────┘
                            ↓ Fin de l'obligation légale
┌──────────────────────────────────────────────────────────────────────┐
│ 3. SUPPRESSION DÉFINITIVE                                            │
├──────────────────────────────────────────────────────────────────────┤
│ - Données supprimées de manière irréversible                         │
│ - Pas de récupération possible                                       │
│ - Méthode : Purge, shred, degauss (selon support)                    │
└──────────────────────────────────────────────────────────────────────┘
```

**Erreur courante à éviter :**
```
❌ Garder les données dans Redis "au cas où"
   → Violation du principe de minimisation
   → Risque de violation de données augmenté
   → Non-conformité RGPD

✅ Transférer vers archivage approprié si besoin
   → Redis = données actives uniquement
   → Archivage = DB permanente avec accès restreint
   → Suppression = fin de vie réglementaire
```

---

## Mécanismes de rétention dans Redis

### TTL (Time To Live)

**Principe :**
Redis permet de définir une durée de vie automatique sur chaque clé. À expiration, la clé est automatiquement supprimée.

**Commandes TTL :**
```bash
# Définir un TTL lors de l'écriture
SET key value EX 3600          # Expire dans 3600 secondes (1 heure)
SETEX key 3600 value           # Identique
SET key value PX 3600000       # Expire dans 3600000 millisecondes
PSETEX key 3600000 value       # Identique

# Ajouter un TTL sur une clé existante
EXPIRE key 3600                # Définir TTL en secondes
PEXPIRE key 3600000            # Définir TTL en millisecondes
EXPIREAT key 1702300800        # Expirer à un timestamp Unix spécifique
PEXPIREAT key 1702300800000    # Identique en millisecondes

# Consulter le TTL
TTL key                        # Retourne le TTL en secondes (-1 = pas de TTL, -2 = clé n'existe pas)
PTTL key                       # Retourne le TTL en millisecondes

# Supprimer le TTL (rendre la clé persistante)
PERSIST key                    # ⚠️ Utiliser avec prudence (conformité)
```

**Bonnes pratiques TTL :**

```python
# ✅ Toujours définir un TTL (sauf exceptions documentées)
redis.setex('session:user123', 3600, session_data)

# ✅ TTL cohérent avec la finalité
redis.setex('2fa:code:user123', 300, code)  # 5 minutes pour code 2FA

# ✅ TTL en fonction du type de donnée
TTL_CONFIG = {
    'session': 86400,           # 24 heures
    'cache': 3600,              # 1 heure
    'cart': 86400 * 30,         # 30 jours
    '2fa': 300,                 # 5 minutes
    'rate_limit': 3600,         # 1 heure
    'analytics': 86400 * 90,    # 90 jours
}

data_type = 'session'
redis.setex(f'{data_type}:key', TTL_CONFIG[data_type], value)

# ❌ Éviter les clés sans TTL
redis.set('user:profile', data)  # ⚠️ Pas de TTL = conservation indéfinie

# ❌ Éviter PERSIST sauf justification documentée
redis.persist('important_key')  # ⚠️ Doit être exception rare et loggée
```

**Politique de TTL documentée :**

```yaml
# /etc/redis/ttl-policy.yml
# Politique de TTL par type de donnée

policies:
  session:
    description: "Session utilisateur authentifié"
    ttl_seconds: 86400  # 24h
    justification: "Durée nécessaire pour éviter reconnexions fréquentes (UX)"
    gdpr_basis: "Article 6.1.b - Exécution du contrat"

  cache_api:
    description: "Cache de réponses API externes"
    ttl_seconds: 3600  # 1h
    justification: "Données volatiles, péremption rapide acceptable"
    gdpr_basis: "Article 6.1.f - Intérêt légitime (performance)"

  cart:
    description: "Panier d'achat utilisateur"
    ttl_seconds: 2592000  # 30 jours
    justification: "Permettre reprise commande ultérieure (usage e-commerce standard)"
    gdpr_basis: "Article 6.1.b - Exécution du contrat"

  token_2fa:
    description: "Code de vérification 2FA"
    ttl_seconds: 300  # 5 minutes
    justification: "Durée minimale pour saisie utilisateur, sécurité renforcée"
    gdpr_basis: "Article 6.1.b - Exécution du contrat (sécurité compte)"

  rate_limit:
    description: "Compteur de rate limiting par IP"
    ttl_seconds: 3600  # 1 heure
    justification: "Fenêtre glissante pour protection anti-abus"
    gdpr_basis: "Article 6.1.f - Intérêt légitime (sécurité service)"
    pii_data: false  # IP peut être PII selon CNIL

  analytics:
    description: "Données comportementales utilisateur"
    ttl_seconds: 7776000  # 90 jours
    justification: "Analyse tendances, recommandations personnalisées"
    gdpr_basis: "Article 6.1.a - Consentement (opt-in obligatoire)"
    consent_required: true

# Règles par défaut
defaults:
  default_ttl: 86400  # 24h par défaut si non spécifié
  no_ttl_allowed: false  # Interdit les clés sans TTL
  max_ttl: 31536000  # 1 an maximum (sauf exceptions documentées)

# Exceptions (clés pouvant ne pas avoir de TTL)
exceptions:
  - pattern: "config:*"
    reason: "Configuration système non-PII"
  - pattern: "feature_flag:*"
    reason: "Feature flags techniques non-PII"
```

**Implémentation automatique du TTL :**

```python
# Wrapper Redis avec TTL obligatoire

import redis
import yaml
from typing import Optional

class ComplianceRedisClient:
    """
    Client Redis avec application automatique de la politique de TTL
    """

    def __init__(self, redis_client, policy_file='/etc/redis/ttl-policy.yml'):
        self.redis = redis_client

        # Charger la politique de TTL
        with open(policy_file, 'r') as f:
            self.policy = yaml.safe_load(f)

        self.default_ttl = self.policy['defaults']['default_ttl']
        self.no_ttl_allowed = self.policy['defaults']['no_ttl_allowed']
        self.max_ttl = self.policy['defaults']['max_ttl']
        self.exceptions = self.policy['exceptions']

    def _get_ttl_for_key(self, key: str) -> int:
        """
        Déterminer le TTL approprié pour une clé selon la politique
        """
        # Extraire le type de donnée depuis la clé (convention: type:...)
        key_type = key.split(':', 1)[0] if ':' in key else None

        # Chercher dans les politiques
        if key_type and key_type in self.policy['policies']:
            return self.policy['policies'][key_type]['ttl_seconds']

        # Vérifier les exceptions (clés pouvant ne pas avoir de TTL)
        for exception in self.exceptions:
            if self._matches_pattern(key, exception['pattern']):
                return None  # Pas de TTL (exception)

        # Par défaut
        return self.default_ttl

    def _matches_pattern(self, key: str, pattern: str) -> bool:
        """Vérifier si une clé match un pattern (wildcard support)"""
        import fnmatch
        return fnmatch.fnmatch(key, pattern)

    def _validate_ttl(self, ttl: Optional[int]) -> None:
        """Valider qu'un TTL respecte la politique"""
        if ttl is None and not self.no_ttl_allowed:
            raise ValueError("TTL is required by compliance policy")

        if ttl and ttl > self.max_ttl:
            raise ValueError(f"TTL {ttl}s exceeds maximum allowed {self.max_ttl}s")

    def set(self, key: str, value, ex: Optional[int] = None) -> bool:
        """
        SET avec application automatique de la politique de TTL
        """
        # Déterminer le TTL
        if ex is None:
            ex = self._get_ttl_for_key(key)

        # Valider
        self._validate_ttl(ex)

        # Exécuter
        if ex:
            return self.redis.setex(key, ex, value)
        else:
            # Exception : pas de TTL autorisé pour cette clé
            return self.redis.set(key, value)

    def setex(self, key: str, time: int, value) -> bool:
        """SETEX avec validation du TTL"""
        self._validate_ttl(time)
        return self.redis.setex(key, time, value)

    def get(self, key: str):
        """GET standard"""
        return self.redis.get(key)

    def delete(self, *keys):
        """DEL standard"""
        return self.redis.delete(*keys)

    def audit_keys_without_ttl(self) -> list:
        """
        Auditer les clés sans TTL (non-conformité potentielle)
        """
        keys_without_ttl = []

        # Scanner toutes les clés (à faire par batch en production)
        for key in self.redis.scan_iter(count=1000):
            key_str = key.decode('utf-8')
            ttl = self.redis.ttl(key)

            # TTL = -1 signifie pas de TTL
            if ttl == -1:
                # Vérifier si c'est une exception autorisée
                is_exception = any(
                    self._matches_pattern(key_str, exc['pattern'])
                    for exc in self.exceptions
                )

                if not is_exception:
                    keys_without_ttl.append({
                        'key': key_str,
                        'size': self.redis.memory_usage(key),
                        'type': self.redis.type(key).decode('utf-8')
                    })

        return keys_without_ttl

# Utilisation
redis_client = redis.Redis(host='localhost', port=6379)
compliant_redis = ComplianceRedisClient(redis_client)

# ✅ TTL automatique selon politique
compliant_redis.set('session:user123', session_data)
# → TTL de 86400s appliqué automatiquement

compliant_redis.set('2fa:code:user456', code)
# → TTL de 300s appliqué automatiquement

# ✅ Exception autorisée (config technique)
compliant_redis.set('config:app_version', '2.1.0')
# → Pas de TTL (exception documentée)

# ❌ TTL trop long (violation politique)
try:
    compliant_redis.setex('test:key', 365*86400, 'value')  # 1 an
except ValueError as e:
    print(f"Error: {e}")  # TTL exceeds maximum

# Audit de conformité
non_compliant_keys = compliant_redis.audit_keys_without_ttl()
if non_compliant_keys:
    print(f"⚠️ Found {len(non_compliant_keys)} keys without TTL")
    for key_info in non_compliant_keys:
        print(f"  - {key_info['key']} ({key_info['type']}, {key_info['size']} bytes)")
```

### Politiques d'éviction (maxmemory-policy)

**Principe :**
Lorsque Redis atteint sa limite mémoire (`maxmemory`), il doit décider quelles clés supprimer pour libérer de l'espace. La politique d'éviction détermine le comportement.

**Politiques disponibles :**
```
┌───────────────────────────────────────────────────────────────────────┐
│ Politique            │ Comportement                                   │
├──────────────────────┼────────────────────────────────────────────────┤
│ noeviction           │ Retourne erreur, refuse nouvelles écritures    │
│ (défaut)             │ ⚠️ Non recommandé pour prod                    │
├──────────────────────┼────────────────────────────────────────────────┤
│ allkeys-lru          │ Éviction LRU parmi TOUTES les clés             │
│ (Recommandé cache)   │ ✅ Bon pour cache général                      │
├──────────────────────┼────────────────────────────────────────────────┤
│ allkeys-lfu          │ Éviction LFU parmi TOUTES les clés             │
│ (Redis 4+)           │ ✅ Bon si patterns d'accès distincts           │
├──────────────────────┼────────────────────────────────────────────────┤
│ volatile-lru         │ Éviction LRU parmi clés AVEC TTL               │
│ (Compromis)          │ ✅ Bon si mix cache + données persistantes     │
├──────────────────────┼────────────────────────────────────────────────┤
│ volatile-lfu         │ Éviction LFU parmi clés AVEC TTL               │
│ (Redis 4+)           │ ✅ Bon pour patterns d'accès variés            │
├──────────────────────┼────────────────────────────────────────────────┤
│ volatile-ttl         │ Éviction des clés avec TTL le plus court       │
│                      │ ✅ Bon si TTL = priorité d'éviction            │
├──────────────────────┼────────────────────────────────────────────────┤
│ allkeys-random       │ Éviction aléatoire parmi TOUTES les clés       │
│                      │ ⚠️ Rarement optimal                            │
├──────────────────────┼────────────────────────────────────────────────┤
│ volatile-random      │ Éviction aléatoire parmi clés AVEC TTL         │
│                      │ ⚠️ Rarement optimal                            │
└───────────────────────────────────────────────────────────────────────┘

LRU = Least Recently Used (le moins récemment utilisé)
LFU = Least Frequently Used (le moins fréquemment utilisé)
```

**Configuration redis.conf :**
```conf
# Limite mémoire (obligatoire en production)
maxmemory 4gb

# Politique d'éviction (choisir selon usage)
maxmemory-policy allkeys-lru

# Précision de l'algorithme LRU/LFU (échantillonnage)
maxmemory-samples 5  # Par défaut, augmenter à 10 pour plus de précision

# Configuration LFU (si maxmemory-policy = *-lfu)
lfu-log-factor 10      # Facteur logarithmique de décroissance
lfu-decay-time 1       # Temps de décroissance en minutes
```

**Choix de la politique selon le contexte :**

```
Cas 1 : Redis utilisé uniquement comme CACHE
→ maxmemory-policy allkeys-lru (ou allkeys-lfu)
Justification : Toutes les clés sont évictables, garder les plus utilisées

Cas 2 : Redis mixte (cache + données importantes avec TTL)
→ maxmemory-policy volatile-lru (ou volatile-ttl)
Justification : Évicter uniquement les données temporaires (TTL)

Cas 3 : Redis avec données critiques sans TTL
→ ❌ Problème de conformité !
Solution : Ajouter des TTL OU migrer vers DB permanente

Cas 4 : Redis pour sessions utilisateurs
→ maxmemory-policy volatile-lru
Justification : Sessions ont TTL, évicter les moins utilisées
```

**Exemple de configuration conforme :**

```conf
# redis.conf - Configuration conforme RGPD

# Limite mémoire stricte (éviter OOM)
maxmemory 8gb

# Éviction LRU des clés avec TTL (sessions, cache)
maxmemory-policy volatile-lru

# Précision accrue pour respecter le principe de minimisation
maxmemory-samples 10

# Logging des évictions (audit)
logfile /var/log/redis/redis-server.log
loglevel notice  # Loggera les évictions

# Persistance pour audit (optionnel)
save 900 1       # Snapshot si 1+ changement en 15min
save 300 10      # Snapshot si 10+ changements en 5min
save 60 10000    # Snapshot si 10000+ changements en 1min
```

**Monitoring des évictions :**
```bash
# Vérifier le nombre d'évictions
redis-cli INFO stats | grep evicted_keys
# evicted_keys:12345

# Taux d'éviction élevé = problème potentiel
# → Augmenter maxmemory OU réduire les TTL OU revoir l'architecture
```

---

## Procédures de purge automatisée

### Purge basée sur TTL (native Redis)

**Mécanisme natif :**
Redis supprime automatiquement les clés expirées via deux mécanismes :

1. **Lazy expiration** : Clé supprimée quand accédée après expiration
2. **Active expiration** : Scan périodique (10x/seconde par défaut) d'un échantillon de clés

**Configuration :**
```conf
# Fréquence du scan actif d'expiration (Hz)
hz 10  # Défaut: 10 scans/seconde

# Effort CPU maximal pour l'expiration active (1-10)
# Plus élevé = expiration plus rapide mais plus de CPU
activedefrag yes
active-defrag-cycle-min 1
active-defrag-cycle-max 25
```

**Avantages :**
- ✅ Automatique, pas de code supplémentaire
- ✅ Distribué (chaque instance gère ses expirations)
- ✅ Conforme si TTL correctement définis

**Limitations :**
- ⚠️ Expiration peut être retardée (échantillonnage)
- ⚠️ Clés rarement accédées peuvent persister après TTL
- ⚠️ Pas de contrôle fin sur l'ordre de purge

### Purge programmée (scheduled cleanup)

**Cas d'usage :**
Nettoyage périodique de patterns spécifiques, purge batch, conformité stricte.

**Script de purge automatisée :**

```python
#!/usr/bin/env python3
"""
Redis Data Retention Purge Script
Purge automatique selon politique de rétention documentée
"""

import redis
import yaml
import logging
import time
from datetime import datetime, timedelta
from typing import List, Dict, Optional

# Configuration du logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/redis/purge.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger('redis-purge')

class RedisDataRetentionPurge:
    """
    Service de purge automatique basé sur politique de rétention
    """

    def __init__(self, redis_client, policy_file='/etc/redis/retention-policy.yml'):
        self.redis = redis_client

        # Charger la politique
        with open(policy_file, 'r') as f:
            self.policy = yaml.safe_load(f)

        self.stats = {
            'keys_scanned': 0,
            'keys_purged': 0,
            'bytes_freed': 0,
            'errors': 0
        }

    def purge_expired_patterns(self) -> Dict:
        """
        Purger les clés selon patterns définis dans la politique
        """
        logger.info("Starting scheduled purge")

        for pattern_config in self.policy.get('purge_patterns', []):
            pattern = pattern_config['pattern']
            max_age_days = pattern_config.get('max_age_days')
            reason = pattern_config.get('reason', 'policy_retention')

            logger.info(f"Purging pattern: {pattern} (max_age: {max_age_days} days)")

            try:
                purged = self._purge_pattern(pattern, max_age_days, reason)
                logger.info(f"Purged {purged} keys for pattern {pattern}")
            except Exception as e:
                logger.error(f"Error purging pattern {pattern}: {e}")
                self.stats['errors'] += 1

        logger.info(f"Purge completed. Stats: {self.stats}")
        return self.stats

    def _purge_pattern(self, pattern: str, max_age_days: Optional[int], reason: str) -> int:
        """
        Purger un pattern spécifique
        """
        purged_count = 0
        cursor = 0

        # Calculer le timestamp limite si max_age défini
        cutoff_timestamp = None
        if max_age_days:
            cutoff_date = datetime.now() - timedelta(days=max_age_days)
            cutoff_timestamp = cutoff_date.timestamp()

        # Scanner les clés par batch
        while True:
            cursor, keys = self.redis.scan(cursor, match=pattern, count=1000)

            for key in keys:
                self.stats['keys_scanned'] += 1

                try:
                    key_str = key.decode('utf-8')

                    # Vérifier l'âge de la clé si critère défini
                    if cutoff_timestamp:
                        # Tentative de récupérer le timestamp de création
                        # (nécessite que l'app stocke cette info)
                        created_at = self._get_key_created_at(key_str)

                        if created_at and created_at < cutoff_timestamp:
                            should_purge = True
                        else:
                            should_purge = False
                    else:
                        # Pas de critère d'âge, purger selon TTL ou pattern seul
                        ttl = self.redis.ttl(key)
                        should_purge = (ttl == -1)  # Pas de TTL = à purger

                    if should_purge:
                        # Enregistrer la taille avant suppression
                        size = self.redis.memory_usage(key)
                        if size:
                            self.stats['bytes_freed'] += size

                        # Supprimer
                        self.redis.delete(key)
                        purged_count += 1
                        self.stats['keys_purged'] += 1

                        # Log d'audit
                        logger.debug(f"Purged key: {key_str} (reason: {reason})")

                except Exception as e:
                    logger.error(f"Error processing key {key}: {e}")
                    self.stats['errors'] += 1

            if cursor == 0:
                break  # Fin du scan

        return purged_count

    def _get_key_created_at(self, key: str) -> Optional[float]:
        """
        Récupérer le timestamp de création d'une clé
        Nécessite que l'application stocke cette métadonnée
        """
        # Exemple : Si la clé contient un hash avec 'created_at'
        try:
            if self.redis.type(key) == b'hash':
                created_at = self.redis.hget(key, 'created_at')
                if created_at:
                    return float(created_at)
        except:
            pass

        return None

    def purge_keys_without_ttl(self, dry_run=False) -> int:
        """
        Purger toutes les clés sans TTL (non-conformité)

        Args:
            dry_run: Si True, ne fait que lister sans supprimer
        """
        logger.info(f"Scanning keys without TTL (dry_run={dry_run})")

        count = 0
        cursor = 0

        # Exceptions (clés autorisées sans TTL)
        exceptions = [exc['pattern'] for exc in self.policy.get('exceptions', [])]

        while True:
            cursor, keys = self.redis.scan(cursor, count=1000)

            for key in keys:
                key_str = key.decode('utf-8')
                ttl = self.redis.ttl(key)

                # TTL = -1 signifie pas de TTL
                if ttl == -1:
                    # Vérifier si exception
                    is_exception = any(
                        self._matches_pattern(key_str, pattern)
                        for pattern in exceptions
                    )

                    if not is_exception:
                        if dry_run:
                            logger.warning(f"Would purge: {key_str} (no TTL)")
                        else:
                            self.redis.delete(key)
                            logger.info(f"Purged: {key_str} (no TTL - compliance)")

                        count += 1

            if cursor == 0:
                break

        logger.info(f"Keys without TTL: {count} (purged={not dry_run})")
        return count

    def _matches_pattern(self, key: str, pattern: str) -> bool:
        """Vérifier si une clé match un pattern"""
        import fnmatch
        return fnmatch.fnmatch(key, pattern)

    def generate_retention_report(self) -> str:
        """
        Générer un rapport de rétention des données
        """
        report_lines = [
            "=" * 80,
            "Redis Data Retention Report",
            f"Generated: {datetime.now().isoformat()}",
            "=" * 80,
            ""
        ]

        # Statistiques globales
        info = self.redis.info()
        report_lines.extend([
            "Global Statistics:",
            f"  Total keys: {info.get('db0', {}).get('keys', 0):,}",
            f"  Memory used: {info.get('used_memory_human')}",
            f"  Evicted keys: {info.get('evicted_keys', 0):,}",
            f"  Expired keys: {info.get('expired_keys', 0):,}",
            ""
        ])

        # Analyse par pattern
        report_lines.append("Retention by Pattern:")
        for pattern_config in self.policy.get('purge_patterns', []):
            pattern = pattern_config['pattern']
            max_age = pattern_config.get('max_age_days', 'N/A')

            # Compter les clés de ce pattern
            count = 0
            total_size = 0
            cursor = 0

            while True:
                cursor, keys = self.redis.scan(cursor, match=pattern, count=100)
                count += len(keys)

                for key in keys:
                    size = self.redis.memory_usage(key)
                    if size:
                        total_size += size

                if cursor == 0:
                    break

            report_lines.append(f"  {pattern}:")
            report_lines.append(f"    Keys: {count:,}")
            report_lines.append(f"    Size: {total_size / 1024 / 1024:.2f} MB")
            report_lines.append(f"    Max age: {max_age} days")
            report_lines.append("")

        # Clés sans TTL (non-conformité potentielle)
        no_ttl_count = 0
        cursor = 0
        while True:
            cursor, keys = self.redis.scan(cursor, count=1000)
            for key in keys:
                if self.redis.ttl(key) == -1:
                    no_ttl_count += 1
            if cursor == 0:
                break

        report_lines.extend([
            "Compliance Status:",
            f"  Keys without TTL: {no_ttl_count:,}",
            f"  {'⚠️ NON-COMPLIANT' if no_ttl_count > 0 else '✅ COMPLIANT'}",
            ""
        ])

        return "\n".join(report_lines)

# Configuration
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
POLICY_FILE = '/etc/redis/retention-policy.yml'

# Point d'entrée
if __name__ == '__main__':
    import sys

    # Connexion Redis
    redis_client = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=False)

    # Service de purge
    purge_service = RedisDataRetentionPurge(redis_client, POLICY_FILE)

    # Commande
    command = sys.argv[1] if len(sys.argv) > 1 else 'purge'

    if command == 'purge':
        # Purge selon politique
        stats = purge_service.purge_expired_patterns()
        print(f"✅ Purge completed: {stats['keys_purged']} keys deleted")

    elif command == 'audit':
        # Audit des clés sans TTL
        count = purge_service.purge_keys_without_ttl(dry_run=True)
        print(f"⚠️ Found {count} keys without TTL (non-compliant)")

    elif command == 'fix':
        # Purger les clés sans TTL
        count = purge_service.purge_keys_without_ttl(dry_run=False)
        print(f"✅ Purged {count} non-compliant keys")

    elif command == 'report':
        # Générer un rapport
        report = purge_service.generate_retention_report()
        print(report)

        # Sauvegarder
        with open(f'/var/log/redis/retention_report_{datetime.now().strftime("%Y%m%d")}.txt', 'w') as f:
            f.write(report)

    else:
        print("Usage: redis-purge.py {purge|audit|fix|report}")
        sys.exit(1)
```

**Fichier de politique (/etc/redis/retention-policy.yml) :**
```yaml
# Politique de rétention et purge Redis

# Patterns à purger automatiquement
purge_patterns:
  - pattern: "temp:*"
    max_age_days: 1
    reason: "Données temporaires de calcul"

  - pattern: "cache:old:*"
    max_age_days: 7
    reason: "Ancien namespace de cache obsolète"

  - pattern: "session:expired:*"
    max_age_days: null  # Purger indépendamment de l'âge si pas de TTL
    reason: "Sessions marquées comme expirées"

  - pattern: "analytics:*"
    max_age_days: 90
    reason: "Données analytics au-delà de la période de rétention RGPD"

# Exceptions (clés autorisées sans TTL)
exceptions:
  - pattern: "config:*"
    reason: "Configuration système non-PII"
  - pattern: "feature_flag:*"
    reason: "Feature flags techniques"
  - pattern: "global:counter:*"
    reason: "Compteurs globaux permanents"

# Fréquence d'exécution recommandée
schedule:
  purge: "daily"  # Purge quotidienne à 02:00
  audit: "weekly"  # Audit hebdomadaire
  report: "monthly"  # Rapport mensuel
```

**Automatisation avec cron :**
```bash
# /etc/cron.d/redis-purge

# Purge quotidienne à 02:00
0 2 * * * redis /usr/local/bin/redis-purge.py purge >> /var/log/redis/purge-cron.log 2>&1

# Audit hebdomadaire (dimanche 03:00)
0 3 * * 0 redis /usr/local/bin/redis-purge.py audit | mail -s "Redis Compliance Audit" compliance@example.com

# Rapport mensuel (1er du mois 04:00)
0 4 1 * * redis /usr/local/bin/redis-purge.py report | mail -s "Redis Retention Report" management@example.com
```

---

## Documentation et traçabilité

### Registre des durées de rétention

**Template de documentation (conformité ISO 27001, RGPD) :**

```markdown
# Registre des Durées de Rétention - Redis

**Organisme :** Société XYZ SAS
**DPO :** dpo@xyz.com
**Date de dernière mise à jour :** 2024-12-11
**Version :** 2.1
**Approbation :** CISO (signature requise)

---

## 1. Session Utilisateur

**Pattern Redis :** `session:*`, `user_session:{user_id}`

**Finalité :**
Maintenir l'état de connexion de l'utilisateur pendant sa navigation sur le site web.

**Base légale RGPD :**
Article 6.1.b - Exécution du contrat (fourniture du service)

**Durée de conservation :**
- **Active (Redis) :** 24 heures (86400 secondes)
- **Archive :** N/A (pas d'archivage)
- **Logs :** 12 mois (logs d'audit des connexions)

**Justification de la durée :**
24 heures est un équilibre entre :
- Expérience utilisateur (éviter reconnexions fréquentes)
- Sécurité (limiter la fenêtre d'exposition en cas de vol de session)
- Minimisation des données (RGPD Article 5.1.c)

**Mécanisme de suppression :**
- TTL automatique Redis (SETEX 86400)
- Éviction volatile-lru si mémoire pleine
- Purge manuelle possible via API (déconnexion utilisateur)

**Catégories de données :**
- user_id (identifiant)
- session_token
- IP address (considérée PII selon CNIL)
- timestamp dernière activité

**Revue :** Annuelle (prochaine : 2025-12-11)

---

## 2. Panier d'Achat

**Pattern Redis :** `cart:{user_id}`, `cart:anonymous:{session_id}`

**Finalité :**
Permettre à l'utilisateur de reprendre son panier lors de visites ultérieures (amélioration expérience client).

**Base légale RGPD :**
Article 6.1.b - Exécution du contrat (processus de commande en cours)

**Durée de conservation :**
- **Active (Redis) :** 30 jours (2592000 secondes)
- **Archive :** N/A
- **Après validation commande :** Transfert vers DB transactionnelle (10 ans - obligation comptable)

**Justification de la durée :**
30 jours est conforme aux pratiques e-commerce standards et permet :
- Rappel par email marketing (si consentement séparé)
- Analyse du taux d'abandon de panier
- Temps suffisant pour retour utilisateur

**Mécanisme de suppression :**
- TTL automatique Redis (EXPIRE 2592000)
- Suppression immédiate après validation commande
- Purge manuelle sur demande (droit à l'effacement)

**Catégories de données :**
- product_id (non-PII)
- quantités (non-PII)
- user_id (identifiant)
- timestamp création

**Revue :** Annuelle

---

## 3. Token 2FA / Vérification Email

**Pattern Redis :** `2fa:token:{user_id}`, `email_verification:{token}`

**Finalité :**
Vérification de l'identité de l'utilisateur (authentification forte, activation compte).

**Base légale RGPD :**
Article 6.1.b - Exécution du contrat (sécurisation du compte utilisateur)

**Durée de conservation :**
- **Active (Redis) :** 5 minutes (300 secondes) pour 2FA, 24h pour email
- **Archive :** N/A
- **Logs :** 12 mois (tentatives authentification)

**Justification de la durée :**
- 5 minutes : Durée minimale pour saisie code (UX) tout en limitant fenêtre d'attaque
- 24 heures email : Standard industrie pour activation compte

**Mécanisme de suppression :**
- TTL automatique Redis
- Suppression immédiate après utilisation (validation réussie)

**Catégories de données :**
- Token aléatoire (non-PII en soi)
- user_id associé (PII)
- timestamp génération

**Particularité sécurité :**
Données sensibles (facteur d'authentification) - chiffrement applicatif recommandé

**Revue :** Annuelle

---

## 4. Rate Limiting / Anti-Abus

**Pattern Redis :** `rate_limit:{ip}`, `throttle:{user_id}`, `failed_login:{ip}`

**Finalité :**
Protection du service contre abus (DDoS, brute force, scraping).

**Base légale RGPD :**
Article 6.1.f - Intérêt légitime (sécurité du système)

**Durée de conservation :**
- **Active (Redis) :** 1 heure (3600 secondes) standard, jusqu'à 24h pour blocage temporaire
- **Archive :** N/A
- **Logs :** 30 jours (analyse patterns d'attaque)

**Justification de la durée :**
1 heure est suffisant pour :
- Fenêtre glissante de rate limiting
- Détection d'anomalies
- Réinitialisation automatique après période suspecte

**Mécanisme de suppression :**
- TTL automatique Redis
- Purge manuelle possible (déblocage administratif)

**Catégories de données :**
- IP address (considérée PII indirect selon CNIL)
- Compteur de requêtes
- timestamp

**Test de proportionnalité (intérêt légitime) :**
- ✅ But légitime : Sécurité du service
- ✅ Nécessité : Mesure technique indispensable
- ✅ Équilibre : Impact minimal sur utilisateurs légitimes (blocage temporaire court)

**Revue :** Annuelle

---

## 5. Analytics Comportementales

**Pattern Redis :** `analytics:user:{user_id}:pageviews`, `behavior:{user_id}`

**Finalité :**
Analyse du comportement utilisateur pour amélioration du service et recommandations personnalisées.

**Base légale RGPD :**
Article 6.1.a - **Consentement explicite** (opt-in obligatoire)

**Durée de conservation :**
- **Active (Redis) :** 90 jours (7776000 secondes)
- **Archive :** 13 mois (réglementation cookies CNIL)
- **Après anonymisation :** Illimitée (statistiques agrégées anonymes)

**Justification de la durée :**
90 jours correspond à :
- Durée recommandée CNIL pour données comportementales
- Fenêtre suffisante pour analyse de tendances
- Cycle de vie produit (recommandations basées sur historique récent)

**Mécanisme de suppression :**
- TTL automatique Redis (90 jours)
- Suppression immédiate sur retrait du consentement
- Anonymisation après 90 jours si conservation statistique nécessaire

**Catégories de données :**
- Pages visitées
- Timestamps
- user_id
- Produits vus/cliqués

**Consentement :**
- ⚠️ Opt-in explicite requis (banner cookie conforme ePrivacy)
- ⚠️ Retrait du consentement = suppression immédiate

**Revue :** Trimestrielle (données sensibles)

---

## Tableau récapitulatif

| Type données | Pattern Redis | Durée Redis | Base légale | Revue |
|--------------|---------------|-------------|-------------|-------|
| Session | session:* | 24h | Contrat | Annuelle |
| Panier | cart:* | 30j | Contrat | Annuelle |
| 2FA | 2fa:* | 5min | Contrat | Annuelle |
| Rate limit | rate_limit:* | 1h | Int. légit. | Annuelle |
| Analytics | analytics:* | 90j | Consentement | Trimestrielle |

---

## Validation et approbation

**Revue effectuée par :**
- DPO : ☐ Validé
- RSSI : ☐ Validé
- Architecte Données : ☐ Validé
- Compliance Officer : ☐ Validé

**Approbation finale :**
- CISO : ______________________  Date : ___________

**Prochaine revue prévue :** 2025-12-11
```

### Logs de purge (audit trail)

**Importance :**
Tracer toutes les opérations de purge pour démontrer la conformité (accountability RGPD).

**Format de log standardisé :**
```json
{
  "timestamp": "2024-12-11T02:00:15.123Z",
  "event_type": "data_purge",
  "purge_type": "automated_ttl_expiration",
  "pattern": "analytics:user:*",
  "keys_purged": 1523,
  "bytes_freed": 15234560,
  "reason": "GDPR Article 5.1.e - Data retention policy",
  "retention_period_days": 90,
  "initiated_by": "system_cron",
  "policy_version": "2.1",
  "compliance_framework": ["GDPR", "CNIL"],
  "dry_run": false,
  "success": true,
  "errors": 0
}
```

**Conservation des logs de purge :**
```
Durée : 3-5 ans (preuve de conformité)
Objectif : Démontrer que les suppressions ont été effectuées
Storage : Logs d'audit sécurisés (write-only, chiffrés)
```

---

## Checklist de conformité

### Politique de rétention

```
Documentation :
□ Politique de rétention rédigée et approuvée (CISO, DPO)
□ Durées définies pour chaque type de donnée
□ Justification métier et légale de chaque durée
□ Base légale RGPD identifiée pour chaque traitement
□ Distinction Active / Archive / Suppression claire
□ Registre des durées de rétention à jour (Article 30 RGPD)
□ Version et date de dernière révision documentées

Revue périodique :
□ Revue annuelle de la politique (minimum)
□ Revue trimestrielle pour données sensibles
□ Validation par les parties prenantes (DPO, RSSI, Métier)
□ Mise à jour après changements réglementaires
□ Signature formelle de l'approbation
```

### Implémentation technique

```
Configuration Redis :
□ maxmemory défini (pas de croissance illimitée)
□ maxmemory-policy configurée (allkeys-lru ou volatile-lru)
□ hz configuré (fréquence expiration active)
□ TTL définis sur TOUTES les clés (sauf exceptions documentées)
□ TTL cohérents avec la politique de rétention
□ Pas de clés sans TTL en production (compliance)

Automatisation :
□ Wrapper applicatif force les TTL (pas de SET sans SETEX)
□ Script de purge automatisée déployé
□ Cron configuré pour exécution régulière
□ Monitoring des évictions (alertes si taux élevé)
□ Audit périodique des clés sans TTL
□ Procédure de purge manuelle documentée
```

### Suppression et archivage

```
Mécanismes de suppression :
□ TTL automatique Redis (mécanisme principal)
□ Éviction mémoire configurée (backup automatique)
□ Purge programmée (script cron)
□ Purge sur demande (droit à l'effacement RGPD)
□ Suppression sécurisée des backups (shred, encryption)

Archivage (si applicable) :
□ Distinction claire Redis (actif) vs DB permanente (archive)
□ Transfert automatique après fin période active
□ Archivage chiffré et accès restreint
□ Durée d'archivage documentée et conforme
□ Procédure de restauration testée
```

### Traçabilité et audit

```
Logs de conformité :
□ Toutes les purges loggées (automated + manual)
□ Format structuré (JSON) pour analyse
□ Informations complètes (quoi, quand, pourquoi, combien)
□ Conservation logs purge : 3-5 ans
□ Logs sécurisés (write-only, chiffrés)
□ Revue périodique des logs (détection anomalies)

Rapports :
□ Rapport mensuel des purges
□ Rapport trimestriel de conformité rétention
□ Audit annuel par tiers externe (recommandé)
□ Documentation des exceptions (clés sans TTL justifiées)
□ Tableau de bord (dashboard) des métriques rétention
```

### Tests et validation

```
□ Tests unitaires (TTL appliqués correctement)
□ Tests d'intégration (purge end-to-end)
□ Tests de restauration depuis archive
□ Simulation de droit à l'effacement (RGPD Article 17)
□ Validation de la destruction sécurisée
□ Tests de charge (éviction sous pression mémoire)
□ Revue du code (code review des wrappers TTL)
```

### Formation et sensibilisation

```
□ Équipe dev formée aux politiques de rétention
□ Documentation technique accessible
□ Runbooks pour opérations courantes
□ Formation RGPD annuelle (incluant rétention)
□ Procédures d'escalation documentées
□ Contact DPO communiqué à toutes les équipes
```

---

## Conclusion

La politique de rétention des données est un pilier de la conformité Redis. Cette section a couvert :

- ✅ **Cadre réglementaire** exhaustif (RGPD, PCI DSS, HIPAA, SOC 2, ISO 27001)
- ✅ **Classification des données** avec durées de rétention par catégorie
- ✅ **Mécanismes Redis** : TTL, politiques d'éviction, configuration
- ✅ **Wrapper Python** pour application automatique de la politique
- ✅ **Script de purge** automatisée complet (300+ lignes)
- ✅ **Documentation** : Template de registre conforme RGPD
- ✅ **Traçabilité** : Format de logs d'audit
- ✅ **Checklists** de conformité (60+ points)

**Points critiques à retenir :**
1. **RGPD Article 5.1.e** : Limitation de la conservation est obligatoire
2. **TTL systématique** : Toutes les clés DOIVENT avoir un TTL (sauf exceptions documentées)
3. **Documentation obligatoire** : Registre des durées de rétention requis (Article 30)
4. **Justification** : Chaque durée doit avoir une justification métier et légale
5. **Distinction Active/Archive** : Redis = actif, DB permanente = archive
6. **Pas de conservation "au cas où"** : Violation du principe de minimisation
7. **Traçabilité** : Toutes les purges doivent être loggées
8. **Revue périodique** : Annuelle minimum, trimestrielle pour données sensibles

**Prochaines étapes :**
- Rédiger la politique de rétention formelle
- Définir les durées par type de donnée
- Implémenter les wrappers avec TTL forcé
- Déployer le script de purge automatisée
- Configurer maxmemory et éviction
- Auditer les clés existantes sans TTL
- Former les équipes
- Planifier la revue annuelle

⏭️ [Compliance et certifications](/17-gouvernance-conformite/06-compliance-certifications.md)
