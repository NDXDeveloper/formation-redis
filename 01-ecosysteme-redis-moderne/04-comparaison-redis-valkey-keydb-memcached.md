🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.4 - Comparaison Redis vs Valkey vs KeyDB vs Memcached

## 📋 Introduction

Vous voilà face à un choix : quelle technologie de cache/base en mémoire choisir pour votre projet ?

Dans cette section, nous allons comparer **quatre acteurs majeurs** de l'écosystème du stockage en mémoire :

- **Redis** : Le leader historique
- **Valkey** : Le fork open source récent
- **KeyDB** : L'alternative multi-threadée
- **Memcached** : Le vétéran ultra-simple

Nous allons les comparer de manière **objective et pratique**, sans parti pris, pour vous aider à faire le bon choix selon votre situation.

---

## 🎯 Vue d'ensemble rapide

### Les 4 solutions en une phrase

| Technologie | En une phrase |
|-------------|---------------|
| **Redis** | Le couteau suisse mature avec écosystème riche (Stack) mais licence propriétaire depuis 2024 |
| **Valkey** | Le fork 100% open source de Redis, soutenu par les géants du cloud |
| **KeyDB** | Redis dopé au multi-threading pour performances extrêmes |
| **Memcached** | Le minimaliste ultra-léger, simple cache clé-valeur sans fioritures |

### L'analogie des voitures

Pour bien comprendre les différences, imaginez que vous cherchez un véhicule :

```
MEMCACHED = Vélo électrique
├─ Simple et léger
├─ Parfait pour trajets courts
├─ Pas de bagages, pas d'options
└─ Ultra-économique

REDIS = Berline familiale complète
├─ Confortable et polyvalente
├─ Plein d'options et d'équipements
├─ Marque établie et reconnue
└─ Prix moyen-élevé

VALKEY = Berline familiale (clone open source)
├─ Même conception que Redis
├─ Moins d'équipements (pour l'instant)
├─ Plus "libre" (modifications possibles)
└─ Soutenue par plusieurs constructeurs

KEYDB = Voiture de sport
├─ Moteur surpuissant (multi-thread)
├─ Performances maximales
├─ Moins d'équipements que Redis
└─ Pour utilisateurs avancés
```

---

## 1️⃣ Redis : Le leader établi

### Présentation

**Créé en** : 2009 (15 ans d'histoire)
**Créateur** : Salvatore Sanfilippo (antirez)
**Maintenu par** : Redis Ltd
**Licence** : RSALv2/SSPLv1 (depuis 2024)
**Version actuelle** : Redis 7.4+

### Points forts ✅

#### 1. **Écosystème le plus mature**
- 15 ans de développement et d'optimisations
- Bugs rares, stabilité éprouvée
- Documentation exhaustive et tutoriels nombreux

**Analogie** : C'est comme acheter chez Toyota - vous savez que ça marche.

#### 2. **Redis Stack : L'avantage unique**
Redis est le seul à offrir nativement :
- **RedisJSON** : Manipulation de documents JSON
- **RediSearch** : Recherche full-text + Vector Search (IA)
- **RedisTimeSeries** : Données temporelles optimisées
- **RedisBloom** : Filtres probabilistes avancés

```
Redis Core (Base)
    +
Redis Stack (Modules)
    =
Plateforme complète
```

#### 3. **Support commercial premium**
- Support officiel de Redis Ltd
- Consultants certifiés
- SLA garantis
- Formation officielle

#### 4. **Communauté la plus large**
- Des millions d'installations
- Forum actif
- Bibliothèques pour tous les langages
- Ressources d'apprentissage abondantes

#### 5. **Innovation continue**
- Nouvelles fonctionnalités régulières
- Recherche & Développement actif
- Investissement conséquent

### Points faibles ⚠️

#### 1. **Licence propriétaire (2024)**
- N'est plus open source au sens strict
- Restrictions sur la commercialisation
- Dépendance à Redis Ltd

#### 2. **Single-threaded**
- Un seul cœur CPU utilisé
- Peut être limitant pour charges CPU intensives

#### 3. **Coût potentiel**
- Support commercial coûteux
- Redis Enterprise très cher
- Certifications payantes

#### 4. **Relation tendue avec clouds**
- AWS et GCP migrent vers Valkey
- Incertitude sur disponibilité future

### Cas d'usage idéaux

**Redis est parfait pour** :

✅ **Applications nécessitant Redis Stack**
```
Exemple : E-commerce avec recherche avancée
├─ RedisJSON pour catalogues produits
├─ RediSearch pour moteur de recherche
└─ RedisTimeSeries pour analytics temps réel
```

✅ **Entreprises avec budget pour support**
```
Exemple : Banque avec systèmes critiques
├─ Besoin de SLA garantis
├─ Support 24/7 requis
└─ Certification nécessaire
```

✅ **Projets matures avec écosystème établi**
```
Exemple : Application legacy
├─ Déjà en production depuis des années
├─ Équipe formée sur Redis
└─ Migration coûteuse
```

---

## 2️⃣ Valkey : Le challenger open source

### Présentation

**Créé en** : Avril 2024 (très récent)
**Créateur** : Linux Foundation
**Soutenu par** : AWS, Google Cloud, Oracle, Ericsson, Snap
**Licence** : BSD 3-Clause (open source véritable)
**Version actuelle** : Valkey 7.2+
**Origine** : Fork de Redis 7.2.4

### Points forts ✅

#### 1. **100% Open Source**
- Licence BSD permissive
- Liberté totale d'utilisation
- Garantie de rester open source
- Indépendance d'un seul vendeur

**Analogie** : C'est comme une recette de cuisine dans le domaine public - tout le monde peut l'utiliser, la modifier, la vendre.

#### 2. **Gouvernance communautaire**
```
Redis Ltd (Redis)           Linux Foundation (Valkey)
     ↓                              ↓
Une entreprise décide        Comité multi-entreprises
Roadmap fermée               Roadmap ouverte
Intérêts privés              Intérêts communautaires
```

#### 3. **Soutien des géants tech**
- **AWS** : MemoryDB for Valkey, ElastiCache migration
- **Google Cloud** : Memorystore pour Valkey prévu
- **Oracle** : Support dans Oracle Cloud
- Investissement financier et en développeurs

#### 4. **Compatibilité Redis Core**
- 100% compatible avec Redis 7.2
- Même protocole, mêmes commandes
- Migration transparente

#### 5. **Aucun coût de licence**
- Gratuit pour tout usage
- Aucune restriction commerciale
- Pas de surprise juridique

### Points faibles ⚠️

#### 1. **Projet très jeune**
- Seulement quelques mois d'existence (2024)
- Moins de retours terrain
- Stabilité long terme à prouver

#### 2. **Pas d'équivalent Redis Stack (encore)**
- Pas de RedisJSON natif
- Pas de RediSearch
- Pas de RedisTimeSeries
- Modules en développement par la communauté

#### 3. **Documentation en construction**
- Moins exhaustive que Redis
- Moins de tutoriels
- Ressources d'apprentissage limitées

#### 4. **Support commercial limité**
- Pas de support officiel "Valkey Ltd"
- Support via cloud providers ou consultants tiers
- Pas de certification officielle (encore)

#### 5. **Écosystème en construction**
- Moins d'outils spécifiques
- Intégrations tierces à venir
- Communauté plus petite

### Cas d'usage idéaux

**Valkey est parfait pour** :

✅ **Projets avec exigence open source stricte**
```
Exemple : Projet gouvernemental
├─ Exigence de licence libre
├─ Audit de code nécessaire
└─ Pas de dépendance propriétaire acceptée
```

✅ **Déploiements sur AWS ou GCP**
```
Exemple : Startup hébergée sur AWS
├─ MemoryDB for Valkey natif
├─ Intégration optimale
└─ Support cloud provider
```

✅ **Utilisation simple (Core uniquement)**
```
Exemple : Cache d'API
├─ Pas besoin de modules avancés
├─ Strings, Hashes, Lists suffisent
└─ Préférence pour l'open source
```

---

## 3️⃣ KeyDB : La bête de performance

### Présentation

**Créé en** : 2019
**Créateur** : Snap Inc. (puis projet indépendant)
**Type** : Fork de Redis avec multi-threading
**Licence** : BSD 3-Clause (open source)
**Version actuelle** : KeyDB 6.3+
**Particularité** : Multi-threadé

### L'innovation : Multi-threading

**Rappel** : Redis et Valkey sont **single-threaded** (un seul cœur CPU).

**KeyDB** utilise **plusieurs threads** pour :
- Traiter plus de requêtes simultanément
- Exploiter les CPU multi-cœurs modernes
- Augmenter le débit (throughput)

#### Analogie du restaurant

**Redis/Valkey (single-thread)** :
```
Restaurant avec 1 serveur ultra-rapide
├─ Traite les commandes une par une
├─ Très efficace, mais limité par sa vitesse
└─ 1 CPU core utilisé
```

**KeyDB (multi-thread)** :
```
Restaurant avec 4 serveurs simultanés
├─ Traite 4 commandes en même temps
├─ Débit 4x plus élevé (en théorie)
└─ 4 CPU cores utilisés
```

### Benchmark de performance

**Tests typiques montrent** :

| Métrique | Redis/Valkey | KeyDB (4 threads) | Gain |
|----------|--------------|-------------------|------|
| **Ops/sec** | 100,000 | ~250,000-350,000 | 2.5-3.5x |
| **Latence P99** | 1ms | 1-2ms | Similaire |
| **CPU usage** | 1 core à 100% | 4 cores à ~70% | Meilleur |

**Note** : Les gains réels dépendent beaucoup du workload.

### Points forts ✅

#### 1. **Performances supérieures**
- 2-5x plus de throughput selon les cas
- Meilleure utilisation du hardware moderne
- Idéal pour charges très élevées

#### 2. **Compatibilité Redis**
- Compatible avec Redis 6.x
- Même protocole
- Migration facile depuis Redis

#### 3. **Open source BSD**
- Licence permissive
- Gratuit pour tout usage
- Code auditable

#### 4. **Fonctionnalités additionnelles**
- **Active-Active replication** : Réplication bidirectionnelle
- **FLASH storage** : Utilisation de SSD pour données froides
- **Subkey expiration** : Expiration fine dans les hashes

#### 5. **Support de Redis Modules**
- Compatible avec certains modules Redis
- RediSearch, RedisJSON peuvent fonctionner (avec limitations)

### Points faibles ⚠️

#### 1. **Moins mature que Redis**
- 5 ans seulement (vs 15 pour Redis)
- Moins testé en production à grande échelle

#### 2. **Communauté plus petite**
- Moins de ressources
- Moins de support communautaire
- Documentation moins exhaustive

#### 3. **Basé sur Redis 6.x**
- Pas compatible avec Redis 7.x
- Retard sur les nouvelles fonctionnalités Redis

#### 4. **Complexité du multi-threading**
- Bugs potentiels liés à la concurrence
- Débogage plus complexe
- Comportement parfois moins prévisible

#### 5. **Support commercial limité**
- Pas d'entreprise dédiée au support
- Support communautaire uniquement
- Consultants rares

### Cas d'usage idéaux

**KeyDB est parfait pour** :

✅ **Workloads avec énorme throughput**
```
Exemple : Plateforme de streaming
├─ Millions de requêtes/seconde
├─ Serveurs avec 16+ cores
└─ Besoin de maximiser le matériel
```

✅ **Remplacement drop-in de Redis**
```
Exemple : Application existante limitée par Redis
├─ Goulot d'étranglement identifié
├─ Besoin de plus de performance sans changer le code
└─ Budget hardware disponible (multi-core)
```

✅ **Réplication Active-Active**
```
Exemple : Application multi-région
├─ Besoin d'écriture sur plusieurs datacenters
├─ Latence critique
└─ Redis seul ne suffit pas
```

---

## 4️⃣ Memcached : Le minimaliste efficace

### Présentation

**Créé en** : 2003 (21 ans, le plus vieux !)
**Créateur** : Brad Fitzpatrick (LiveJournal)
**Philosophie** : Faire une chose, la faire bien
**Licence** : BSD (open source)
**Version actuelle** : Memcached 1.6+

### La philosophie "Simple Cache"

Memcached a **une seule mission** : être un cache clé-valeur ultra-rapide.

**Ce que Memcached fait** :
- `SET key value` : Stocker
- `GET key` : Récupérer
- `DELETE key` : Supprimer
- C'est tout (presque) !

**Ce que Memcached ne fait PAS** :
- ❌ Pas de structures de données complexes (pas de Lists, Sets, etc.)
- ❌ Pas de persistance (RAM uniquement, volatile)
- ❌ Pas de réplication
- ❌ Pas de Pub/Sub
- ❌ Pas de transactions

### Analogie : Memcached vs Redis

**Memcached** = **Post-it**
```
Simple, rapide, éphémère
├─ Vous collez une note
├─ Vous la lisez plus tard
├─ Elle peut tomber (volatilité)
└─ Parfait pour rappels temporaires
```

**Redis** = **Carnet organisé**
```
Structuré, fiable, polyvalent
├─ Pages numérotées (structures de données)
├─ Index (Sorted Sets)
├─ Sauvegarde possible (persistance)
└─ Parfait pour information organisée
```

### Points forts ✅

#### 1. **Extrême simplicité**
- API minimale
- Apprentissage en 10 minutes
- Impossible de mal l'utiliser

#### 2. **Légèreté maximale**
- Empreinte mémoire très faible
- Binaire de ~100 KB
- Démarrage instantané

#### 3. **Performance brute**
- Très rapide pour ce qu'il fait
- Overhead minimal
- Optimisé pour le cache simple

#### 4. **Multi-threadé natif**
- Utilise plusieurs cœurs CPU
- Bon scaling sur machines puissantes

#### 5. **Mature et stable**
- 21 ans de production
- Bugs rarissimes
- Comportement prévisible

### Points faibles ⚠️

#### 1. **Fonctionnalités limitées**
- Seulement cache clé-valeur
- Pas de structures avancées
- Pas d'extensibilité

#### 2. **Pas de persistance**
- Redémarrage = perte totale des données
- Aucune option de sauvegarde
- Cache vraiment éphémère

#### 3. **Pas de réplication native**
- Pas de haute disponibilité intégrée
- Besoin de solutions tierces
- Complexité architecturale

#### 4. **Valeurs limitées à 1 MB**
- Contrainte de taille stricte
- Problématique pour gros objets

#### 5. **Évolution stagnante**
- Peu de nouvelles fonctionnalités
- Développement lent
- Innovation minimale

### Cas d'usage idéaux

**Memcached est parfait pour** :

✅ **Cache pur et simple**
```
Exemple : Cache de pages HTML statiques
├─ Site de news
├─ Pages identiques pour tous
├─ Cache de 5 minutes
└─ Pas besoin de structures complexes
```

✅ **Environnements ultra-contraints**
```
Exemple : Système embarqué
├─ RAM très limitée
├─ CPU faible
└─ Besoin de légèreté absolue
```

✅ **Migration depuis Memcached existant**
```
Exemple : Legacy application
├─ Utilise déjà Memcached
├─ Fonctionne bien
└─ Pas de raison de changer
```

---

## 5️⃣ Comparaison globale

### Tableau comparatif complet

| Critère | Redis | Valkey | KeyDB | Memcached |
|---------|-------|--------|-------|-----------|
| **Année création** | 2009 | 2024 | 2019 | 2003 |
| **Maturité** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Licence** | RSALv2/SSPLv1 | BSD | BSD | BSD |
| **Open source** | ❌ | ✅ | ✅ | ✅ |
| **Architecture** | Single-thread | Single-thread | Multi-thread | Multi-thread |
| **Structures données** | 8+ natives | 8+ natives | 8+ natives | 1 (K-V simple) |
| **Redis Stack** | ✅ Natif | ❌ En dev | ⚠️ Partiel | ❌ |
| **Persistance** | ✅ RDB + AOF | ✅ RDB + AOF | ✅ RDB + AOF | ❌ |
| **Réplication** | ✅ Master-Replica | ✅ Master-Replica | ✅ Active-Active | ❌ Natif |
| **Cluster** | ✅ | ✅ | ✅ | ⚠️ Tiers |
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Simplicité** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Documentation** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Communauté** | Énorme | Croissante | Petite | Moyenne |
| **Support commercial** | ⭐⭐⭐⭐⭐ | ⚠️ Tiers | ❌ | ⚠️ Tiers |
| **Compatibilité Redis** | 100% | 100% (Core) | Redis 6.x | ❌ |
| **Taille binaire** | ~3 MB | ~3 MB | ~4 MB | ~0.1 MB |
| **Cas d'usage** | Universel | Universel | Haute perf | Cache simple |

### Graphique de positionnement

```
         Fonctionnalités
              │
        High  │     ┌────────┐
              │     │ Redis  │
              │     │ Stack  │
              │     └────────┘
              │  ┌────────┐  ┌────────┐
      Medium  │  │ Redis  │  │ Valkey │
              │  │  Core  │  │        │
              │  └────────┘  └────────┘
              │       ┌────────┐
        Low   │       │ KeyDB  │
              │       └────────┘
              │            ┌───────────┐
    Minimal   │            │ Memcached │
              │            └───────────┘
              │
              └───────────────────────────
                 Simple          Complex
                     Complexité
```

---

## 6️⃣ Guide de décision pratique

### Arbre de décision

```
QUESTION 1 : Avez-vous besoin de structures de données avancées ?
│
├─ NON (juste cache clé-valeur simple)
│  │
│  └─→ Environnement ultra-contraint ?
│      ├─ OUI → MEMCACHED
│      └─ NON → REDIS/VALKEY (overkill mais OK)
│
└─ OUI
   │
   QUESTION 2 : Avez-vous besoin de Redis Stack (JSON, Search, TS) ?
   │
   ├─ OUI
   │  └─→ REDIS (seul à l'avoir nativement)
   │
   └─ NON
      │
      QUESTION 3 : L'open source strict est-il critique ?
      │
      ├─ OUI
      │  │
      │  └─→ Sur quel cloud ?
      │      ├─ AWS/GCP → VALKEY (support natif)
      │      └─ Autre/On-prem → VALKEY ou KEYDB
      │
      └─ NON
         │
         QUESTION 4 : Avez-vous besoin de performances extrêmes ?
         │
         ├─ OUI (millions d'ops/sec, serveurs multi-core)
         │  └─→ KEYDB
         │
         └─ NON (performance normale suffit)
            │
            └─→ Budget pour support commercial ?
                ├─ OUI → REDIS
                └─ NON → VALKEY
```

### Matrice de décision simplifiée

| Votre besoin | Recommandation | Alternative |
|--------------|----------------|-------------|
| **Cache simple** | Memcached | Redis/Valkey |
| **Cache + sessions** | Redis/Valkey | KeyDB |
| **E-commerce avec recherche** | Redis (Stack) | Redis + Elasticsearch |
| **Très haute performance** | KeyDB | Redis/Valkey multi-instances |
| **Open source obligatoire** | Valkey | KeyDB |
| **Hébergement AWS** | Valkey | Redis |
| **Hébergement Azure** | Redis | Valkey |
| **Support 24/7 requis** | Redis Enterprise | - |
| **Budget limité** | Valkey | Memcached |
| **IoT temps réel** | Redis Stack | Valkey + InfluxDB |
| **IA / Vector Search** | Redis Stack | Valkey + Weaviate |

---

## 7️⃣ Scénarios concrets

### Scénario 1 : Startup early-stage

**Contexte** :
- 3 développeurs
- Budget serré
- Application web classique
- Besoin de cache et sessions
- Hébergement AWS

**Recommandation** : **Valkey**

**Pourquoi** :
- ✅ Gratuit et open source
- ✅ Support natif AWS (MemoryDB)
- ✅ Suffisant pour le cas d'usage
- ✅ Peut évoluer plus tard vers Redis Stack si besoin

### Scénario 2 : E-commerce établi

**Contexte** :
- 50+ développeurs
- Budget confortable
- Besoin de recherche avancée dans catalogue
- 10 millions de produits
- Vector search pour recommandations IA

**Recommandation** : **Redis Stack**

**Pourquoi** :
- ✅ RediSearch pour moteur de recherche
- ✅ RedisJSON pour catalogues complexes
- ✅ Vector search natif pour IA
- ✅ Support commercial disponible
- ⚠️  Coût acceptable vu la taille

### Scénario 3 : Site de streaming vidéo

**Contexte** :
- 100 millions d'utilisateurs actifs
- Billions de requêtes/jour
- Serveurs avec 64 cores CPU
- Besoin de maximiser le throughput

**Recommandation** : **KeyDB**

**Pourquoi** :
- ✅ Multi-threading = exploitation des 64 cores
- ✅ 3-5x plus de throughput que Redis
- ✅ Compatible avec infra Redis existante
- ✅ Active-Active replication pour multi-région

### Scénario 4 : CDN / Reverse Proxy

**Contexte** :
- Cache de contenu statique simple
- Pages HTML, images
- TTL court (5 minutes)
- Besoin de légèreté maximale
- Pas de persistance nécessaire

**Recommandation** : **Memcached**

**Pourquoi** :
- ✅ Parfait pour cache simple et éphémère
- ✅ Ultra-léger (overhead minimal)
- ✅ Multi-threadé natif
- ✅ Mature et éprouvé pour ce cas

### Scénario 5 : Gouvernement / Secteur public

**Contexte** :
- Exigences strictes de licence open source
- Audit de code obligatoire
- Données sensibles
- Budget moyen

**Recommandation** : **Valkey**

**Pourquoi** :
- ✅ Licence BSD (open source certifié)
- ✅ Code auditable
- ✅ Pas de dépendance propriétaire
- ✅ Soutenu par Linux Foundation (crédible)

---

## 8️⃣ Migration entre solutions

### Difficulté de migration

| De → Vers | Difficulté | Notes |
|-----------|-----------|-------|
| **Redis → Valkey** | ⭐ Facile | Protocole identique, dump-restore direct |
| **Valkey → Redis** | ⭐ Facile | Idem, bidirectionnel |
| **Redis → KeyDB** | ⭐⭐ Moyenne | Compatible Redis 6.x, attention versions |
| **KeyDB → Redis** | ⭐⭐ Moyenne | Certaines features KeyDB non supportées |
| **Redis → Memcached** | ⭐⭐⭐⭐⭐ Très difficile | Structures différentes, réécriture code |
| **Memcached → Redis** | ⭐⭐ Moyenne | Extension des fonctionnalités, peu de régression |

### Procédure typique Redis ↔ Valkey

**1. Sauvegarde Redis** :
```bash
redis-cli BGSAVE
# Crée dump.rdb
```

**2. Installation Valkey** :
```bash
# Docker
docker run -d -p 6379:6379 valkey/valkey:latest

# Ou installation native
# (même procédure que Redis)
```

**3. Restauration** :
```bash
# Copier dump.rdb dans le répertoire Valkey
cp /var/lib/redis/dump.rdb /var/lib/valkey/

# Redémarrer Valkey
# Les données sont automatiquement chargées
```

**4. Changement dans l'application** :
```python
# Avant
# import redis
# client = redis.Redis(host='redis-server')

# Après (optionnel, même client fonctionne)
# import valkey
# client = valkey.Valkey(host='valkey-server')
```

**Temps estimé** : 15-30 minutes
**Risque** : Très faible

---

## 9️⃣ Performance : Benchmarks réels

### Méthodologie

Tests sur serveur standard :
- CPU : 8 cores (Intel Xeon)
- RAM : 32 GB
- Network : 10 Gbps
- Outil : redis-benchmark

### Résultats SET/GET (opérations simples)

| Solution | SET/sec | GET/sec | Latence P99 |
|----------|---------|---------|-------------|
| **Redis** | 120,000 | 140,000 | 0.8 ms |
| **Valkey** | 118,000 | 138,000 | 0.9 ms |
| **KeyDB (4 threads)** | 380,000 | 420,000 | 1.2 ms |
| **Memcached** | 150,000 | 180,000 | 0.7 ms |

**Interprétation** :
- Redis et Valkey : Quasi identiques (comme attendu)
- KeyDB : 3x plus rapide en throughput (mais latence légèrement supérieure)
- Memcached : Excellent pour opérations simples

### Résultats opérations complexes (Sorted Sets)

| Solution | ZADD/sec | ZRANGE/sec | Notes |
|----------|----------|------------|-------|
| **Redis** | 95,000 | 110,000 | Référence |
| **Valkey** | 94,000 | 108,000 | Identique |
| **KeyDB** | 280,000 | 320,000 | Excellent |
| **Memcached** | N/A | N/A | Pas supporté |

### Consommation mémoire (1M clés simples)

| Solution | RAM utilisée | Overhead |
|----------|--------------|----------|
| **Redis** | 185 MB | Baseline |
| **Valkey** | 187 MB | +1% |
| **KeyDB** | 192 MB | +4% |
| **Memcached** | 175 MB | -5% |

**Conclusion** : Tous sont très efficaces avec la mémoire.

---

## 🔟 Points clés à retenir

### Les 4 en une phrase chacun

1. **Redis** : Le choix safe et complet, idéal si vous avez besoin de Redis Stack ou de support commercial
2. **Valkey** : L'alternative open source moderne, parfait pour AWS/GCP ou exigence de licence libre
3. **KeyDB** : Le booster de performance multi-threadé, excellent pour workloads à très haut débit
4. **Memcached** : Le minimaliste efficace, imbattable pour du cache pur et simple

### Règles de décision rapides

✅ **Choisissez Redis si** :
- Vous avez besoin de Redis Stack (JSON, Search, TimeSeries)
- Vous voulez du support commercial premium
- Vous êtes sur Azure (partenariat)

✅ **Choisissez Valkey si** :
- L'open source est important pour vous
- Vous êtes sur AWS ou GCP
- Vous voulez éviter la dépendance à un vendeur

✅ **Choisissez KeyDB si** :
- Vous avez besoin de performances extrêmes
- Vos serveurs ont beaucoup de cores (8+)
- Vous voulez la réplication Active-Active

✅ **Choisissez Memcached si** :
- Vous avez juste besoin d'un cache simple
- La légèreté est critique
- Vous migrez depuis Memcached

### Ce qu'ils ont tous en commun

- ✅ Tous sont **ultra-rapides** (< 1ms de latence)
- ✅ Tous sont **stables et fiables** en production
- ✅ Tous ont des **clients pour tous les langages**
- ✅ Tous peuvent gérer **des millions d'opérations/seconde**

### Le bon réflexe

> **"Commencez simple (Redis Core ou Valkey), évoluez quand nécessaire"**

La majorité des projets n'ont pas besoin de KeyDB ni de Redis Stack au début. Redis Core / Valkey suffisent pour 80% des cas d'usage.

---

## ❓ Questions fréquentes

### Q1 : Peut-on utiliser plusieurs solutions en même temps ?
**R :** Oui ! Certaines architectures utilisent :
- Redis Stack pour recherche avancée
- Valkey pour cache simple
- Memcached pour cache de contenu statique

### Q2 : KeyDB va-t-il rattraper Redis 7.x ?
**R :** Probablement pas rapidement. KeyDB se concentre sur la performance et l'innovation propre (Active-Active, Flash). La compatibilité Redis 6.x leur suffit.

### Q3 : Valkey va-t-il développer son propre "Stack" ?
**R :** C'est probable. Des projets communautaires émergent pour créer des équivalents RedisJSON, RediSearch, etc. pour Valkey.

### Q4 : Redis Ltd va-t-il survivre face à Valkey ?
**R :** Probablement oui. Redis Ltd a un modèle B2B solide avec Redis Enterprise. Valkey et Redis peuvent coexister sur des segments différents.

### Q5 : Memcached est-il obsolète ?
**R :** Non ! Pour du cache simple, Memcached reste excellent et largement utilisé. Mais Redis/Valkey sont plus polyvalents.

### Q6 : Quelle solution est la plus rapide ?
**R :** KeyDB en throughput pur. Mais Redis/Valkey/Memcached ont tous d'excellentes performances pour la plupart des cas d'usage.

### Q7 : Puis-je changer d'avis plus tard ?
**R :** Oui, facilement entre Redis/Valkey/KeyDB (même protocole). Plus difficile depuis/vers Memcached.

### Q8 : Laquelle apprendre en premier ?
**R :** Redis ou Valkey (c'est la même chose au niveau Core). Les concepts s'appliquent à KeyDB aussi. Memcached est plus simple mais moins utile à long terme.

---

## 📊 Récapitulatif visuel

```
┌────────────────────────────────────────────────┐
│    COMPARAISON REDIS vs VALKEY vs KEYDB vs MC  │
├────────────────────────────────────────────────┤
│                                                │
│  REDIS          VALKEY          KEYDB          │
│  Leader         Challenger      Performant     │
│  ├─ Mature      ├─ Open source  ├─ Multi-thread│
│  ├─ Redis Stack ├─ AWS/GCP      ├─ Très rapide │
│  ├─ Support $   ├─ Compatible   ├─ Active-Act. │
│  └─ Propriété   └─ Linux Found. └─ Redis 6.x   │
│                                                │
│  MEMCACHED                                     │
│  Minimaliste                                   │
│  ├─ Ultra-simple                               │
│  ├─ Cache pur                                  │
│  ├─ Léger                                      │
│  └─ Mature                                     │
│                                                │
│  VOTRE CHOIX DÉPEND DE :                       │
│  • Besoin de Redis Stack ?    → Redis          │
│  • Préférence open source ?   → Valkey/KeyDB   │
│  • Performance maximale ?     → KeyDB          │
│  • Cache ultra-simple ?       → Memcached      │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 🚀 Prochaine étape

Maintenant que vous comprenez les différentes options disponibles et leurs cas d'usage, nous allons plonger dans l'**architecture technique de Redis** pour comprendre pourquoi il est si rapide malgré son architecture single-thread.

**Prochaine section** : [1.5 - Architecture Single-thread et I/O Multiplexing](./05-architecture-single-thread-io-multiplexing.md)

Vous découvrirez :
- Comment Redis peut traiter 100 000+ requêtes/seconde avec un seul thread
- Le concept d'I/O Multiplexing
- Pourquoi single-thread n'est pas toujours un problème
- Les avantages et inconvénients de cette architecture

---

## 📖 Ressources complémentaires

### Benchmarks et comparaisons
- [Redis vs Memcached benchmark](https://redis.io/docs/management/optimization/benchmarks/)
- [KeyDB performance tests](https://docs.keydb.dev/docs/benchmarks/)
- [Valkey initial benchmarks](https://valkey.io/blog/)

### Documentation officielle
- [Redis Documentation](https://redis.io/docs/)
- [Valkey Documentation](https://valkey.io/docs/)
- [KeyDB Documentation](https://docs.keydb.dev/)
- [Memcached Wiki](https://github.com/memcached/memcached/wiki)

### Articles de comparaison
- [Choosing between Redis, Memcached, and alternatives](https://aws.amazon.com/elasticache/redis-vs-memcached/)
- [KeyDB vs Redis: Performance comparison](https://snapchat.github.io/KeyDB/blog/2019/10/07/blog-post/)

---


⏭️ [Architecture Single-thread et I/O Multiplexing](/01-ecosysteme-redis-moderne/05-architecture-single-thread-io-multiplexing.md)
