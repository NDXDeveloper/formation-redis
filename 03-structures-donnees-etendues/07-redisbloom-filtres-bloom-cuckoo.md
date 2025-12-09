🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.7 RedisBloom : Filtres de Bloom et Cuckoo

## Introduction

Face à des volumes massifs de données, vérifier l'existence d'un élément peut devenir coûteux. Par exemple :
- "Cet email est-il dans la blacklist de 10 millions d'adresses ?"
- "Ce mot de passe a-t-il fuité dans les 5 milliards connus ?"
- "Cet utilisateur a-t-il déjà vu cet article parmi 100 millions ?"

Stocker tous ces éléments en mémoire (Redis Set) ou en base de données nécessite énormément de ressources.

**RedisBloom** propose des **structures de données probabilistes** qui résolvent ces problèmes avec :
- ✅ **Empreinte mémoire minuscule** : 100x à 1000x moins qu'un Set
- ✅ **Vérifications ultra-rapides** : O(1) constant
- ⚠️ **Trade-off** : Probabilité de faux positifs contrôlée

RedisBloom inclut quatre structures principales :
1. **Bloom Filter** : Vérifier l'appartenance (avec faux positifs)
2. **Cuckoo Filter** : Bloom Filter avec possibilité de suppression
3. **Count-Min Sketch** : Compter la fréquence d'événements
4. **Top-K** : Identifier les K éléments les plus fréquents

---

## Filtres de Bloom (Bloom Filter)

### Concept

Un **filtre de Bloom** est une structure de données probabiliste qui répond à la question : "X est-il dans l'ensemble ?"

**Réponses possibles** :
- ✅ "Non, X n'est PAS dans l'ensemble" (100% sûr)
- ⚠️ "Oui, X est PROBABLEMENT dans l'ensemble" (peut être un faux positif)

**Caractéristiques** :
- Pas de faux négatifs (si le filtre dit "non", c'est garanti)
- Faux positifs possibles (si le filtre dit "oui", c'est probable mais pas certain)
- Probabilité de faux positif configurable (ex: 1%, 0.1%, 0.01%)
- Impossible de supprimer un élément (limitation du Bloom classique)

---

### Fonctionnement interne

```
┌─────────────────────────────────────────────────────┐
│              Filtre de Bloom                        │
│  Tableau de bits : [0 1 0 0 1 0 1 0 0 1 0 ...]      │
│                                                     │
│  Ajout de "alice@example.com" :                     │
│  1. hash1(alice@example.com) = 3 → bit[3] = 1       │
│  2. hash2(alice@example.com) = 7 → bit[7] = 1       │
│  3. hash3(alice@example.com) = 15 → bit[15] = 1     │
│                                                     │
│  Vérification de "alice@example.com" :              │
│  1. bit[3] == 1 ? ✓                                 │
│  2. bit[7] == 1 ? ✓                                 │
│  3. bit[15] == 1 ? ✓                                │
│  → Probablement présent                             │
│                                                     │
│  Vérification de "bob@example.com" :                │
│  1. bit[5] == 1 ? ✗                                 │
│  → Définitivement absent (faux négatif impossible)  │
└─────────────────────────────────────────────────────┘
```

---

### Installation et vérification

```bash
# Vérifier les modules chargés
redis-cli MODULE LIST

# Devrait contenir :
# 1) 1) "name"
#    2) "bf"
#    3) "ver"
#    4) 20800  # Version 2.8.0
```

---

### Commandes Bloom Filter

#### BF.RESERVE : Créer un filtre avec paramètres

```bash
# Syntaxe
BF.RESERVE {key} {error_rate} {capacity}
  [EXPANSION expansion]
  [NONSCALING]

# Exemples

# Filtre avec 1% de faux positifs, capacité 10 000 éléments
BF.RESERVE emails:blacklist 0.01 10000

# Filtre avec 0.1% de faux positifs, capacité 1 million
BF.RESERVE leaked_passwords 0.001 1000000

# Filtre non-extensible (erreur si capacité dépassée)
BF.RESERVE temp_filter 0.01 5000 NONSCALING
```

**Paramètres** :
- `error_rate` : Probabilité de faux positif (0.01 = 1%)
- `capacity` : Nombre d'éléments prévus
- `EXPANSION` : Facteur d'expansion (défaut: 2) si capacité dépassée
- `NONSCALING` : Ne pas s'étendre automatiquement

---

#### BF.ADD : Ajouter un élément

```bash
# Ajouter un email à la blacklist
BF.ADD emails:blacklist "spam@evil.com"
# Retourne : (integer) 1 (ajouté)

# Ajouter un élément déjà présent
BF.ADD emails:blacklist "spam@evil.com"
# Retourne : (integer) 0 (déjà présent)
```

---

#### BF.MADD : Ajouter plusieurs éléments

```bash
# Ajouter plusieurs emails en une commande
BF.MADD emails:blacklist
  "spam1@evil.com"
  "spam2@evil.com"
  "spam3@evil.com"

# Retourne :
# 1) (integer) 1  # Ajouté
# 2) (integer) 1  # Ajouté
# 3) (integer) 0  # Déjà présent
```

---

#### BF.EXISTS : Vérifier l'existence

```bash
# Vérifier si un email est dans la blacklist
BF.EXISTS emails:blacklist "spam@evil.com"
# Retourne : (integer) 1 (probablement présent)

BF.EXISTS emails:blacklist "legitimate@example.com"
# Retourne : (integer) 0 (définitivement absent)
```

---

#### BF.MEXISTS : Vérifier plusieurs éléments

```bash
# Vérifier plusieurs emails
BF.MEXISTS emails:blacklist
  "spam@evil.com"
  "legitimate@example.com"
  "unknown@test.com"

# Retourne :
# 1) (integer) 1  # Probablement présent
# 2) (integer) 0  # Absent
# 3) (integer) 0  # Absent
```

---

#### BF.INFO : Informations sur le filtre

```bash
BF.INFO emails:blacklist

# Résultat :
# 1) "Capacity"
# 2) (integer) 10000
# 3) "Size"
# 4) (integer) 3  # Nombre d'éléments ajoutés
# 5) "Number of filters"
# 6) (integer) 1
# 7) "Number of items inserted"
# 8) (integer) 3
# 9) "Expansion rate"
# 10) (integer) 2
```

---

### Taille mémoire Bloom Filter

**Calcul approximatif** :

```
Bits nécessaires = -(n × ln(p)) / (ln(2)²)

Où :
- n = nombre d'éléments (capacity)
- p = probabilité de faux positif (error_rate)
```

**Exemples** :

```bash
# 10 000 éléments, 1% faux positifs
# Taille : ~12 KB

# 1 million d'éléments, 1% faux positifs
# Taille : ~1.2 MB

# 1 million d'éléments, 0.1% faux positifs
# Taille : ~1.8 MB

# Comparaison avec Redis Set :
# 1 million d'emails (moyenne 25 chars)
# Set : ~25 MB
# Bloom (0.1%) : ~1.8 MB
# Réduction : ~93%
```

---

## Filtres Cuckoo (Cuckoo Filter)

### Concept

Un **filtre Cuckoo** est similaire à un Bloom Filter, mais avec des avantages :
- ✅ **Possibilité de suppression** (contrairement au Bloom)
- ✅ **Meilleure utilisation de l'espace** pour de faibles taux d'erreur
- ⚠️ **Trade-off** : Peut échouer l'insertion si le filtre est saturé

**Usage** : Quand vous avez besoin de supprimer des éléments.

---

### Commandes Cuckoo Filter

#### CF.RESERVE : Créer un filtre Cuckoo

```bash
# Syntaxe
CF.RESERVE {key} {capacity}
  [BUCKETSIZE bucketsize]
  [MAXITERATIONS maxiterations]
  [EXPANSION expansion]

# Exemples

# Filtre Cuckoo pour 100 000 éléments
CF.RESERVE usernames:registered 100000

# Avec paramètres personnalisés
CF.RESERVE usernames:registered 100000
  BUCKETSIZE 4
  MAXITERATIONS 20
  EXPANSION 1
```

**Paramètres** :
- `capacity` : Nombre d'éléments prévus
- `BUCKETSIZE` : Taille des buckets (défaut: 2)
- `MAXITERATIONS` : Nombre de tentatives de réinsertion (défaut: 20)
- `EXPANSION` : Facteur d'expansion (défaut: 1)

---

#### CF.ADD : Ajouter un élément

```bash
# Ajouter un username
CF.ADD usernames:registered "alice123"
# Retourne : (integer) 1 (ajouté)

CF.ADD usernames:registered "alice123"
# Retourne : (integer) 0 (déjà présent)
```

---

#### CF.ADDNX : Ajouter seulement si absent

```bash
# Ajouter seulement si pas déjà présent
CF.ADDNX usernames:registered "bob456"
# Retourne : (integer) 1 (ajouté)

CF.ADDNX usernames:registered "bob456"
# Retourne : (integer) 0 (déjà présent, pas ajouté)
```

---

#### CF.DEL : Supprimer un élément

```bash
# Supprimer un username
CF.DEL usernames:registered "alice123"
# Retourne : (integer) 1 (supprimé)

CF.DEL usernames:registered "alice123"
# Retourne : (integer) 0 (n'existe pas)
```

⚠️ **Attention** : Supprimer un élément qui n'a jamais été ajouté peut causer des faux négatifs !

---

#### CF.EXISTS : Vérifier l'existence

```bash
CF.EXISTS usernames:registered "alice123"
# Retourne : (integer) 1 (probablement présent)

CF.EXISTS usernames:registered "charlie789"
# Retourne : (integer) 0 (probablement absent)
```

---

#### CF.COUNT : Compter les occurrences

```bash
# Nombre de fois qu'un élément a été ajouté (approximatif)
CF.COUNT usernames:registered "alice123"
# Retourne : (integer) 1
```

---

#### CF.INFO : Informations sur le filtre

```bash
CF.INFO usernames:registered

# Résultat :
# 1) "Size"
# 2) (integer) 5
# 3) "Number of buckets"
# 4) (integer) 50000
# 5) "Number of filters"
# 6) (integer) 1
# 7) "Number of items inserted"
# 8) (integer) 5
# 9) "Number of items deleted"
# 10) (integer) 1
# 11) "Bucket size"
# 12) (integer) 2
# 13) "Expansion rate"
# 14) (integer) 1
# 15) "Max iterations"
# 16) (integer) 20
```

---

## Count-Min Sketch (CMS)

### Concept

Un **Count-Min Sketch** est une structure probabiliste pour **compter la fréquence** d'événements.

**Usage** : "Combien de fois l'événement X s'est-il produit ?"

**Caractéristiques** :
- ✅ **Mémoire constante** (indépendante du nombre d'éléments uniques)
- ✅ **Incrémentation ultra-rapide** : O(1)
- ⚠️ **Surestimation possible** : Le compteur peut être légèrement supérieur à la réalité
- ✅ **Jamais de sous-estimation** : Le compteur est toujours ≥ réalité

---

### Commandes Count-Min Sketch

#### CMS.INITBYDIM : Créer un sketch par dimensions

```bash
# Syntaxe
CMS.INITBYDIM {key} {width} {depth}

# Créer un sketch de 2000 colonnes × 7 lignes
CMS.INITBYDIM page_views 2000 7

# Plus large : Plus précis mais plus de mémoire
CMS.INITBYDIM clicks 10000 10
```

**Paramètres** :
- `width` : Nombre de colonnes (plus large = plus précis)
- `depth` : Nombre de lignes (fonctions de hash)

---

#### CMS.INITBYPROB : Créer un sketch par probabilité

```bash
# Syntaxe
CMS.INITBYPROB {key} {error} {probability}

# 1% d'erreur avec 99% de confiance
CMS.INITBYPROB page_views 0.01 0.99

# 0.1% d'erreur avec 99.9% de confiance
CMS.INITBYPROB clicks 0.001 0.999
```

**Paramètres** :
- `error` : Erreur acceptable (ex: 0.01 = 1%)
- `probability` : Confiance (ex: 0.99 = 99%)

---

#### CMS.INCRBY : Incrémenter un compteur

```bash
# Incrémenter le compteur pour "index.html"
CMS.INCRBY page_views "index.html" 1

# Incrémenter de 5
CMS.INCRBY page_views "about.html" 5

# Incrémenter plusieurs éléments
CMS.INCRBY page_views
  "index.html" 1
  "about.html" 1
  "contact.html" 1
```

---

#### CMS.QUERY : Obtenir le compteur

```bash
# Obtenir le nombre de vues pour "index.html"
CMS.QUERY page_views "index.html"
# Retourne : (integer) 142

# Obtenir plusieurs compteurs
CMS.QUERY page_views "index.html" "about.html" "contact.html"
# Retourne :
# 1) (integer) 142
# 2) (integer) 87
# 3) (integer) 53
```

---

#### CMS.MERGE : Fusionner plusieurs sketches

```bash
# Créer des sketches par jour
CMS.INITBYPROB views:2024-12-08 0.01 0.99
CMS.INITBYPROB views:2024-12-09 0.01 0.99

# Incrémenter
CMS.INCRBY views:2024-12-08 "index.html" 100
CMS.INCRBY views:2024-12-09 "index.html" 150

# Fusionner en un sketch hebdomadaire
CMS.MERGE views:week_total 2 views:2024-12-08 views:2024-12-09

# Résultat :
CMS.QUERY views:week_total "index.html"
# Retourne : (integer) 250
```

---

#### CMS.INFO : Informations sur le sketch

```bash
CMS.INFO page_views

# Résultat :
# 1) "width"
# 2) (integer) 2000
# 3) "depth"
# 4) (integer) 7
# 5) "count"
# 6) (integer) 1523  # Nombre total d'incrémentations
```

---

## Top-K (Top-K)

### Concept

Une structure **Top-K** identifie les **K éléments les plus fréquents** dans un flux de données.

**Usage** : "Quels sont les 10 produits les plus vendus ?" "Quels sont les 20 mots-clés les plus recherchés ?"

**Caractéristiques** :
- ✅ **Mémoire constante** : Stocke uniquement les K éléments
- ✅ **Mise à jour en temps réel**
- ✅ **Pas besoin de compter tous les éléments**

---

### Commandes Top-K

#### TOPK.RESERVE : Créer une structure Top-K

```bash
# Syntaxe
TOPK.RESERVE {key} {k} [width] [depth] [decay]

# Garder les 10 éléments les plus fréquents
TOPK.RESERVE trending_products 10

# Avec paramètres personnalisés
TOPK.RESERVE trending_keywords 20 2000 7 0.9
```

**Paramètres** :
- `k` : Nombre d'éléments à garder
- `width` : Largeur du sketch interne (optionnel)
- `depth` : Profondeur du sketch interne (optionnel)
- `decay` : Facteur de décroissance pour les anciens éléments (optionnel)

---

#### TOPK.ADD : Ajouter des éléments

```bash
# Ajouter des produits vendus
TOPK.ADD trending_products "laptop" "mouse" "keyboard" "laptop" "monitor"

# Retourne les éléments éjectés du Top-K (si capacité dépassée)
# 1) (nil)
# 2) (nil)
# 3) (nil)
# 4) (nil)
# 5) (nil)
```

---

#### TOPK.INCRBY : Incrémenter la fréquence

```bash
# Incrémenter la fréquence de "laptop" de 5
TOPK.INCRBY trending_products "laptop" 5

# Incrémenter plusieurs éléments
TOPK.INCRBY trending_products
  "laptop" 2
  "mouse" 3
  "keyboard" 1
```

---

#### TOPK.QUERY : Vérifier si un élément est dans le Top-K

```bash
# Vérifier si "laptop" est dans le Top-10
TOPK.QUERY trending_products "laptop"
# Retourne : (integer) 1 (oui, dans le Top-K)

TOPK.QUERY trending_products "laptop" "mouse" "webcam"
# Retourne :
# 1) (integer) 1  # laptop : dans le Top-K
# 2) (integer) 1  # mouse : dans le Top-K
# 3) (integer) 0  # webcam : pas dans le Top-K
```

---

#### TOPK.COUNT : Obtenir la fréquence estimée

```bash
# Obtenir le compteur de "laptop"
TOPK.COUNT trending_products "laptop"
# Retourne : (integer) 147

# Obtenir plusieurs compteurs
TOPK.COUNT trending_products "laptop" "mouse" "keyboard"
# Retourne :
# 1) (integer) 147
# 2) (integer) 89
# 3) (integer) 62
```

---

#### TOPK.LIST : Lister les éléments du Top-K

```bash
# Obtenir la liste complète du Top-K
TOPK.LIST trending_products

# Résultat :
# 1) "laptop"
# 2) "mouse"
# 3) "keyboard"
# 4) "monitor"
# 5) "headset"
# 6) "webcam"
# 7) "tablet"
# 8) "speaker"
# 9) "charger"
# 10) "cable"
```

---

#### TOPK.INFO : Informations sur la structure

```bash
TOPK.INFO trending_products

# Résultat :
# 1) "k"
# 2) (integer) 10
# 3) "width"
# 4) (integer) 8
# 5) "depth"
# 6) (integer) 7
# 7) "decay"
# 8) "0.9"
```

---

## Cas d'usage modernes

### 1️⃣ Vérification de mots de passe fuités (Bloom Filter)

**Contexte** : Vérifier si un mot de passe est dans la base "Have I Been Pwned" (5+ milliards de mots de passe)

```bash
# Créer le filtre (0.01% faux positifs)
BF.RESERVE leaked_passwords 0.0001 5000000000

# Charger les mots de passe fuités (batch)
# (En pratique : script pour charger depuis un dump)
BF.MADD leaked_passwords
  "password123"
  "123456"
  "qwerty"
  # ... 5 milliards de mots de passe
```

**Vérification lors de l'inscription** :

```python
import redis
import hashlib

r = redis.Redis(decode_responses=True)

def is_password_leaked(password):
    # Hasher le mot de passe (SHA-256)
    password_hash = hashlib.sha256(password.encode()).hexdigest()

    # Vérifier dans le filtre
    result = r.bf().exists('leaked_passwords', password_hash)

    if result == 1:
        return True  # Probablement fuité (faux positif possible)
    else:
        return False  # Définitivement pas fuité

# Usage
if is_password_leaked("password123"):
    print("❌ Ce mot de passe a fuité ! Choisissez-en un autre.")
else:
    print("✅ Mot de passe OK")
```

**Avantages** :
- Bloom Filter (0.0001%, 5B passwords) : ~7.2 GB
- Redis Set (5B passwords × 64 bytes) : ~320 GB
- **Réduction : 98% de mémoire économisée**

---

### 2️⃣ Déduplication d'URLs crawlées (Bloom Filter)

**Contexte** : Web crawler évitant de crawler deux fois la même URL

```bash
# Créer le filtre
BF.RESERVE crawled_urls 0.001 10000000  # 10M URLs, 0.1% faux positifs

# Lors du crawl
def should_crawl_url(url):
    # Vérifier si déjà crawlée
    if r.bf().exists('crawled_urls', url):
        return False  # Probablement déjà crawlée
    else:
        # Marquer comme crawlée
        r.bf().add('crawled_urls', url)
        return True  # Nouvelle URL

# Usage
urls_to_check = [
    "https://example.com/page1",
    "https://example.com/page2",
    "https://example.com/page1",  # Duplicate
]

for url in urls_to_check:
    if should_crawl_url(url):
        print(f"Crawling: {url}")
        # crawl(url)
    else:
        print(f"Skipping (already crawled): {url}")
```

**Résultat** :
```
Crawling: https://example.com/page1
Crawling: https://example.com/page2
Skipping (already crawled): https://example.com/page1
```

---

### 3️⃣ Anti-spam : Blacklist d'emails (Cuckoo Filter)

**Contexte** : Blacklist d'emails avec possibilité de déblocage

```bash
# Créer le filtre Cuckoo
CF.RESERVE email_blacklist 1000000

# Ajouter des emails spammeurs
CF.MADD email_blacklist
  "spam1@evil.com"
  "spam2@evil.com"
  "phishing@scam.com"

# Vérification lors de l'envoi d'email
def can_send_email(email):
    if r.cf().exists('email_blacklist', email):
        return False  # Probablement blacklisté
    else:
        return True  # OK

# Débloquer un email (faux positif)
def unblock_email(email):
    r.cf().delete('email_blacklist', email)
    print(f"✅ {email} removed from blacklist")

# Usage
if can_send_email("spam1@evil.com"):
    print("Sending email...")
else:
    print("❌ Email blocked (blacklisted)")

# Débloquer après vérification
unblock_email("legitimate@example.com")
```

**Pourquoi Cuckoo et pas Bloom ?**
- Cuckoo permet la **suppression** (débloquer un email)
- Bloom ne permet pas la suppression

---

### 4️⃣ Analytics : Pages les plus visitées (Count-Min Sketch)

**Contexte** : Compter les vues de pages sans stocker tous les IDs

```bash
# Créer le sketch
CMS.INITBYPROB page_views 0.001 0.999  # 0.1% erreur, 99.9% confiance

# Incrémenter à chaque visite
def track_page_view(page_url):
    r.cms().incrby('page_views', page_url, 1)

# Tracking
track_page_view("/index.html")
track_page_view("/about.html")
track_page_view("/index.html")
track_page_view("/contact.html")
track_page_view("/index.html")

# Obtenir les statistiques
pages = ["/index.html", "/about.html", "/contact.html", "/404.html"]
counts = r.cms().query('page_views', *pages)

for page, count in zip(pages, counts):
    print(f"{page}: {count} views")
```

**Résultat** :
```
/index.html: 3 views
/about.html: 1 views
/contact.html: 1 views
/404.html: 0 views
```

**Avantages** :
- Count-Min Sketch : ~100 KB (fixe, indépendant du nombre de pages)
- Redis Hash (pour stocker tous les compteurs) : ~10 MB+ (croît avec les pages)

---

### 5️⃣ E-commerce : Produits tendance (Top-K)

**Contexte** : Identifier les 20 produits les plus vendus en temps réel

```bash
# Créer la structure Top-20
TOPK.RESERVE trending_products 20 2000 7 0.9

# Incrémenter à chaque vente
def record_sale(product_id):
    r.topk().add('trending_products', product_id)

# Simulation de ventes
sales = ["laptop_123", "mouse_456", "keyboard_789", "laptop_123",
         "monitor_101", "laptop_123", "mouse_456", "headset_202"]

for product_id in sales:
    record_sale(product_id)

# Obtenir le Top-20
trending = r.topk().list('trending_products')
print("Top 20 produits :")
for i, product in enumerate(trending, 1):
    count = r.topk().count('trending_products', product)[0]
    print(f"{i}. {product}: ~{count} ventes")
```

**Résultat** :
```
Top 20 produits :
1. laptop_123: ~3 ventes
2. mouse_456: ~2 ventes
3. keyboard_789: ~1 vente
4. monitor_101: ~1 vente
5. headset_202: ~1 vente
```

---

### 6️⃣ Détection de fraude : IP suspectes (Bloom + Count-Min)

**Contexte** : Détecter les IPs qui font trop de requêtes

```bash
# Bloom Filter : IPs déjà signalées
BF.RESERVE suspicious_ips 0.01 100000

# Count-Min Sketch : Compteur de requêtes par IP
CMS.INITBYPROB ip_requests 0.001 0.999

# Tracking des requêtes
def track_request(ip_address):
    # Incrémenter le compteur
    r.cms().incrby('ip_requests', ip_address, 1)

    # Vérifier le nombre de requêtes
    count = r.cms().query('ip_requests', ip_address)[0]

    # Si > 1000 requêtes/minute → Suspecte
    if count > 1000:
        # Ajouter à la blacklist
        r.bf().add('suspicious_ips', ip_address)
        return "BLOCKED"

    # Vérifier si déjà blacklistée
    if r.bf().exists('suspicious_ips', ip_address):
        return "BLOCKED"

    return "ALLOWED"

# Simulation
print(track_request("192.168.1.100"))  # ALLOWED
# ... 1001 requêtes plus tard
print(track_request("192.168.1.100"))  # BLOCKED
```

---

### 7️⃣ Cache de résultats de recherche (Bloom Filter)

**Contexte** : Éviter de chercher des résultats inexistants

```bash
# Créer le filtre des requêtes en cache
BF.RESERVE cached_queries 0.01 1000000

# Lors d'une recherche
def search(query):
    # Vérifier si en cache
    if r.bf().exists('cached_queries', query):
        # Récupérer du cache
        cached_result = r.get(f'search_cache:{query}')
        if cached_result:
            print(f"✅ Cache HIT: {query}")
            return cached_result

    # Cache MISS → Rechercher dans la DB
    print(f"❌ Cache MISS: {query}")
    result = perform_expensive_search(query)

    # Mettre en cache
    if result:
        r.set(f'search_cache:{query}', result, ex=3600)
        r.bf().add('cached_queries', query)

    return result

def perform_expensive_search(query):
    # Simulation d'une recherche coûteuse
    return f"Results for '{query}'"

# Usage
search("redis tutorial")  # Cache MISS
search("redis tutorial")  # Cache HIT
```

**Avantage** : Évite de chercher dans Redis pour des requêtes jamais cachées.

---

## Performance et comparaisons

### Bloom Filter : Mémoire vs Précision

```bash
# Test : 1 million d'éléments

# Configuration 1 : 1% faux positifs
BF.RESERVE test1 0.01 1000000
# Mémoire : ~1.2 MB
# Précision : 99% (1% faux positifs)

# Configuration 2 : 0.1% faux positifs
BF.RESERVE test2 0.001 1000000
# Mémoire : ~1.8 MB (+50%)
# Précision : 99.9% (0.1% faux positifs)

# Configuration 3 : 0.01% faux positifs
BF.RESERVE test3 0.0001 1000000
# Mémoire : ~2.4 MB (+100%)
# Précision : 99.99% (0.01% faux positifs)
```

**Trade-off** :
- Plus de précision → Plus de mémoire
- Pour la plupart des cas : 0.1% - 1% est suffisant

---

### Cuckoo Filter vs Bloom Filter

```bash
# Test : 1 million d'éléments

# Bloom Filter (1% faux positifs)
BF.RESERVE bloom_test 0.01 1000000
# Mémoire : ~1.2 MB
# Suppression : ❌ Non supportée
# Faux positifs : 1%

# Cuckoo Filter
CF.RESERVE cuckoo_test 1000000
# Mémoire : ~2.5 MB (+108%)
# Suppression : ✅ Supportée
# Faux positifs : ~0.3% (meilleur que Bloom pour petits taux)

# Latence
# BF.EXISTS : ~0.05ms
# CF.EXISTS : ~0.06ms (légèrement plus lent)
```

**Conclusion** :
- **Bloom** : Plus compact, mais pas de suppression
- **Cuckoo** : Suppression possible, mais ~2x plus de mémoire

---

### Count-Min Sketch : Précision

```bash
# Test : 1 million d'événements, 10 000 éléments uniques

# Configuration 1 : 1% erreur
CMS.INITBYPROB cms1 0.01 0.99
# Mémoire : ~100 KB
# Erreur observée : ~1-2% surestimation

# Configuration 2 : 0.1% erreur
CMS.INITBYPROB cms2 0.001 0.999
# Mémoire : ~1 MB (+900%)
# Erreur observée : ~0.1-0.2% surestimation

# Comparaison avec Redis Hash
# Redis Hash (10K clés × 8 bytes) : ~80 KB
# Mais CMS peut compter des millions d'éléments uniques avec la même mémoire !
```

---

### Top-K : Performance

```bash
# Benchmark : Top-20 sur 1 million d'éléments

# TOPK.RESERVE trending 20
# Mémoire : ~50 KB (fixe, indépendant du volume)

# Latence :
# TOPK.ADD : ~0.05ms
# TOPK.LIST : ~0.02ms
# TOPK.COUNT : ~0.03ms

# Comparaison avec Sorted Set
# ZADD + ZREVRANGE pour Top-20 :
# - Mémoire : ~10 MB+ (stocke tous les éléments)
# - ZADD : ~0.1ms
# - ZREVRANGE : ~0.5ms

# Top-K est 200x plus compact !
```

---

## Bonnes pratiques

### ✅ 1. Choisir le bon taux d'erreur

```bash
# ✅ Bon : Taux d'erreur adapté au cas d'usage

# Anti-spam : 1% acceptable
BF.RESERVE email_blacklist 0.01 1000000

# Sécurité (mots de passe fuités) : 0.01% requis
BF.RESERVE leaked_passwords 0.0001 5000000000

# Cache (peu critique) : 5% acceptable
BF.RESERVE cache_bloom 0.05 100000

# ❌ Mauvais : Taux trop faible pour un cas non-critique
BF.RESERVE temp_filter 0.00001 1000  # Gaspillage de mémoire
```

---

### ✅ 2. Dimensionner correctement la capacité

```bash
# ✅ Bon : Estimer la capacité réelle

# Blacklist d'emails : ~100K adresses prévues
BF.RESERVE email_blacklist 0.01 100000

# URLs crawlées : ~50 millions d'URLs
BF.RESERVE crawled_urls 0.001 50000000

# ❌ Mauvais : Sous-dimensionner
BF.RESERVE email_blacklist 0.01 1000  # Trop petit !
# Résultat : Filtre s'étend automatiquement → Dégradation du taux d'erreur
```

---

### ✅ 3. Utiliser Cuckoo seulement si suppression nécessaire

```bash
# ✅ Bon : Bloom pour des éléments permanents
BF.RESERVE leaked_passwords 0.0001 5000000000

# ✅ Bon : Cuckoo pour des éléments temporaires
CF.RESERVE session_tokens 1000000  # Sessions qui expirent

# ❌ Mauvais : Cuckoo sans besoin de suppression
CF.RESERVE permanent_data 1000000  # Gaspillage de mémoire
```

---

### ✅ 4. Combiner avec TTL pour les données temporaires

```bash
# ✅ Bon : Bloom Filter avec TTL pour un cache temporaire

# Créer le filtre
BF.RESERVE daily_visitors:2024-12-09 0.01 100000

# Définir un TTL de 24h
EXPIRE daily_visitors:2024-12-09 86400

# Le lendemain, le filtre est automatiquement supprimé
```

---

### ✅ 5. Vérifier avec EXISTS avant d'ajouter (optimisation)

```bash
# ✅ Bon : Éviter les ajouts inutiles

# Vérifier avant d'ajouter
exists = r.bf().exists('email_blacklist', 'spam@evil.com')
if not exists:
    r.bf().add('email_blacklist', 'spam@evil.com')

# ❌ Moins efficace : Ajouter systématiquement
r.bf().add('email_blacklist', 'spam@evil.com')  # Retourne 0 si déjà présent
```

---

### ✅ 6. Utiliser Count-Min Sketch pour de gros volumes

```bash
# ✅ Bon : CMS pour compter des millions d'événements

# Count-Min Sketch (mémoire fixe)
CMS.INITBYPROB event_counter 0.001 0.999

# ❌ Mauvais : Redis Hash pour des millions de clés
# HINCRBY events "event_id_123456789" 1  # Mémoire croît indéfiniment
```

---

### ✅ 7. Monitorer les faux positifs

```python
# ✅ Bon : Tracker les faux positifs en production

def check_with_validation(item):
    # Vérification rapide avec Bloom
    if r.bf().exists('cache_bloom', item):
        # Vérifier réellement (cache Redis)
        actual = r.get(f'cache:{item}')

        if actual is None:
            # Faux positif détecté !
            log_false_positive()

        return actual
    else:
        return None

# Analyser les logs pour ajuster le taux d'erreur
```

---

## Intégration avec les langages

### Python (redis-py)

```python
import redis

r = redis.Redis(decode_responses=True)

# Bloom Filter
r.bf().reserve('emails', 0.01, 1000)
r.bf().add('emails', 'alice@example.com')
exists = r.bf().exists('emails', 'alice@example.com')
print(f"Exists: {exists}")  # True

# Cuckoo Filter
r.cf().reserve('usernames', 1000)
r.cf().add('usernames', 'alice123')
r.cf().delete('usernames', 'alice123')

# Count-Min Sketch
r.cms().initbyprob('page_views', 0.001, 0.999)
r.cms().incrby('page_views', 'index.html', 1)
count = r.cms().query('page_views', 'index.html')
print(f"Views: {count[0]}")

# Top-K
r.topk().reserve('trending', 10)
r.topk().add('trending', 'item1', 'item2', 'item1')
top_items = r.topk().list('trending')
print(f"Top items: {top_items}")
```

---

### Node.js (node-redis)

```javascript
import { createClient } from 'redis';

const client = await createClient().connect();

// Bloom Filter
await client.bf.reserve('emails', 0.01, 1000);
await client.bf.add('emails', 'alice@example.com');
const exists = await client.bf.exists('emails', 'alice@example.com');
console.log(`Exists: ${exists}`);  // true

// Cuckoo Filter
await client.cf.reserve('usernames', 1000);
await client.cf.add('usernames', 'alice123');
await client.cf.del('usernames', 'alice123');

// Count-Min Sketch
await client.cms.initByProb('page_views', 0.001, 0.999);
await client.cms.incrBy('page_views', { 'index.html': 1 });
const count = await client.cms.query('page_views', 'index.html');
console.log(`Views: ${count[0]}`);

// Top-K
await client.topK.reserve('trending', 10);
await client.topK.add('trending', 'item1', 'item2', 'item1');
const topItems = await client.topK.list('trending');
console.log(`Top items:`, topItems);
```

---

## Comparaison : RedisBloom vs alternatives

| Structure | RedisBloom | Alternative classique | Ratio mémoire |
|-----------|------------|-----------------------|---------------|
| **Bloom Filter** (1M éléments, 1%) | 1.2 MB | Redis Set : 25 MB | 95% économisé |
| **Cuckoo Filter** (1M éléments) | 2.5 MB | Redis Set : 25 MB | 90% économisé |
| **Count-Min Sketch** (1M événements, 10K uniques) | 100 KB | Redis Hash : 10 MB | 99% économisé |
| **Top-K** (Top-20) | 50 KB | Sorted Set : 10 MB+ | 99.5% économisé |

---

## Troubleshooting

### Erreur : "ERR module not loaded"

```bash
# ❌ Erreur
BF.RESERVE test 0.01 1000
# (error) ERR unknown command 'BF.RESERVE'

# ✅ Solution : Vérifier que RedisBloom est chargé
MODULE LIST

# Si absent, utiliser Redis Stack
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest
```

---

### Faux positifs trop élevés

```bash
# ❌ Problème : Trop de faux positifs observés

# Diagnostic
BF.INFO emails:blacklist
# "Expansion rate" élevé ? → Capacité dépassée

# ✅ Solution 1 : Recréer avec plus de capacité
BF.RESERVE emails:blacklist 0.01 10000000  # Au lieu de 1000000

# ✅ Solution 2 : Réduire le taux d'erreur
BF.RESERVE emails:blacklist 0.001 1000000  # 0.1% au lieu de 1%
```

---

### Cuckoo Filter saturé

```bash
# ❌ Erreur : "item exists"
CF.ADD usernames "alice123"
# (error) ERR item exists

# Diagnostic
CF.INFO usernames
# "Number of items inserted" proche de la capacité ?

# ✅ Solution : Augmenter la capacité
CF.RESERVE usernames 10000000  # Au lieu de 1000000
```

---

## Résumé

**RedisBloom offre 4 structures probabilistes** :

1. **Bloom Filter (BF.*)** :
   - ✅ Vérifier l'appartenance (faux positifs possibles)
   - ✅ Ultra-compact (95-99% mémoire économisée)
   - ❌ Pas de suppression
   - **Cas d'usage** : Blacklists, déduplication, mots de passe fuités

2. **Cuckoo Filter (CF.*)** :
   - ✅ Vérifier l'appartenance + suppression
   - ✅ Compact (90% mémoire économisée)
   - ⚠️ Peut saturer
   - **Cas d'usage** : Sessions, tokens temporaires, anti-spam

3. **Count-Min Sketch (CMS.*)** :
   - ✅ Compter la fréquence d'événements
   - ✅ Mémoire fixe (indépendante du nombre d'éléments)
   - ⚠️ Surestimation possible
   - **Cas d'usage** : Analytics, compteurs massivement distribués

4. **Top-K (TOPK.*)** :
   - ✅ Identifier les K éléments les plus fréquents
   - ✅ Mémoire constante
   - ✅ Temps réel
   - **Cas d'usage** : Trending topics, produits populaires, hot keys

**Quand utiliser RedisBloom ?** :
- Volume massif (millions/milliards d'éléments)
- Mémoire limitée
- Faux positifs acceptables
- Besoin de vitesse (O(1) constant)

**Quand NE PAS utiliser** :
- Besoin de précision à 100%
- Volume faible (< 10K éléments) → Redis Set suffit
- Besoin de requêtes complexes

---

**Félicitations !** Vous avez terminé le module 3 sur les structures de données étendues Redis Stack. Vous maîtrisez maintenant :
- ✅ RedisJSON pour les documents JSON natifs
- ✅ RediSearch pour la recherche full-text et vectorielle
- ✅ RedisTimeSeries pour les données temporelles
- ✅ RedisBloom pour les structures probabilistes

**Prochaine étape ?** Passons au module 4 : Le cycle de vie de la donnée

⏭️ [Le cycle de vie de la donnée](/04-cycle-vie-donnee/README.md)
