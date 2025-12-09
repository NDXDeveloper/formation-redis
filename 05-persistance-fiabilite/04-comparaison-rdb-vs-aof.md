🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 Comparaison RDB vs AOF : Avantages et inconvénients

## Introduction

Le choix entre RDB et AOF (ou leur combinaison) est l'une des décisions architecturales les plus importantes lors du déploiement de Redis en production. Chaque mécanisme représente une **philosophie différente** de la persistance :

- **RDB** : Snapshots périodiques, privilégie la performance et la simplicité
- **AOF** : Journal continu, privilégie la durabilité et l'intégrité

Cette section propose une analyse comparative complète pour vous aider à faire le bon choix selon votre contexte.

## Vue d'ensemble comparative

### Tableau comparatif global

| Critère | RDB | AOF | Gagnant |
|---------|-----|-----|---------|
| **Durabilité** | ⭐⭐ | ⭐⭐⭐⭐⭐ | AOF |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | RDB |
| **Taille fichier** | ⭐⭐⭐⭐⭐ | ⭐⭐ | RDB |
| **Temps de récupération** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | RDB |
| **Simplicité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | RDB |
| **Audit trail** | ⭐ | ⭐⭐⭐⭐⭐ | AOF |
| **Réparabilité** | ⭐ | ⭐⭐⭐⭐⭐ | AOF |
| **Consommation mémoire** | ⭐⭐ (fork) | ⭐⭐⭐⭐ | AOF |
| **Overhead CPU** | ⭐⭐⭐⭐ | ⭐⭐⭐ | RDB |
| **Overhead I/O** | ⭐⭐⭐⭐ | ⭐⭐ | RDB |

### Les deux philosophies

```
RDB : "Prenons des photos périodiques"
┌──────────────────────────────────────┐
│ État T0 → [Photo] → ... (15 min) →   │
│ État T1 → [Photo] → ... (15 min) →   │
│ État T2 → [Photo]                    │
└──────────────────────────────────────┘
Perte possible : Dernières 15 minutes

AOF : "Enregistrons tout en continu"
┌──────────────────────────────────────┐
│ CMD1 → CMD2 → CMD3 → CMD4 → CMD5 →   │
│ [log] [log] [log] [log] [log] ...    │
└──────────────────────────────────────┘
Perte possible : Dernière seconde
```

## Comparaison détaillée par critère

### 1. Durabilité et perte de données

#### Tableau de perte de données maximale

| Configuration | Perte maximale | Probabilité perte | Acceptabilité |
|---------------|----------------|-------------------|---------------|
| **RDB seul (15 min)** | 15 minutes | Moyenne | Cache important |
| **RDB seul (5 min)** | 5 minutes | Moyenne | Cache critique |
| **RDB seul (1 min)** | 1 minute | Moyenne | Semi-critique |
| **AOF no** | 30 secondes | Faible | ❌ Non recommandé |
| **AOF everysec** | ~1 seconde | Très faible | ✅ Production standard |
| **AOF always** | Aucune* | Très très faible | Finance/Compliance |
| **RDB + AOF everysec** | ~1 seconde | Très faible | ✅ **Recommandé** |

*Sauf crash matériel ou kernel

#### Scénarios de perte de données

**Scénario 1 : Crash Redis (kill -9)**
```
RDB seul (save 900 1) :
- Dernier snapshot : Il y a 10 minutes
- Perte : 10 minutes de données
- Impact : Moyen à élevé

AOF everysec :
- Dernier fsync : Il y a 0.3 secondes
- Perte : ~0.3 secondes de données
- Impact : Très faible

RDB + AOF everysec :
- Redis se recharge depuis AOF (plus récent)
- Perte : ~0.3 secondes
- Impact : Très faible
```

**Scénario 2 : Crash système complet**
```
RDB seul :
- Perte : Temps depuis dernier snapshot (5-15 min)

AOF everysec :
- Perte : Jusqu'à 1 seconde (buffer OS)
- Note : Le fsync est dans le buffer OS

AOF always :
- Perte : Aucune (ou dernière commande en cours)
```

**Scénario 3 : Corruption du disque**
```
RDB seul :
- Si fichier RDB corrompu → Tout perdu
- Récupération : Depuis backup uniquement

AOF :
- Corruption partielle → Réparable avec redis-check-aof
- Corruption totale → Depuis backup
- Avantage : Réparation possible
```

### 2. Performance et latence

#### Tableau d'impact sur les performances

| Métrique | Sans persist | RDB seul | AOF everysec | AOF always |
|----------|--------------|----------|--------------|------------|
| **Throughput (SET/s)** | 100,000 | 95,000 | 70,000 | 450 |
| **Latence P50** | 0.2ms | 0.2ms | 0.3ms | 8ms |
| **Latence P99** | 0.5ms | 1.5ms | 4ms | 35ms |
| **Latence P999** | 0.8ms | 3ms | 15ms | 80ms |
| **Jitter** | Très faible | Moyen (pics) | Moyen | Élevé |
| **CPU usage (idle)** | 5% | 6% | 10% | 15% |
| **CPU usage (load)** | 25% | 35% (pics) | 35% | 30% |

#### Distribution de la latence

```
RDB seul :
[0.2][0.2][0.2][0.2][15ms*][0.2][0.2]...
                      ↑
              Pic lors du fork (rare)

AOF everysec :
[0.3][0.3][3ms*][0.3][0.3][12ms*][0.3]...
            ↑                ↑
        fsync 1s        fsync lent (disque)

AOF always :
[8][10][7][9][35*][8][9][12]...
               ↑
          Disque lent ponctuel
```

#### Impact sur le throughput en production

**Benchmark réaliste (50% GET, 50% SET) :**

| Configuration | Ops/sec | % vs baseline | Latence médiane |
|---------------|---------|---------------|-----------------|
| No persistence | 85,000 | 100% | 0.2ms |
| RDB (save 900 1) | 82,000 | 96% | 0.2ms |
| RDB (save 60 100) | 75,000 | 88% | 0.3ms |
| AOF no | 78,000 | 92% | 0.2ms |
| AOF everysec | 58,000 | 68% | 0.4ms |
| AOF always | 380 | 0.4% | 10ms |
| RDB + AOF everysec | 55,000 | 65% | 0.4ms |

**Conclusion** : AOF always divise le throughput par **200**.

### 3. Consommation des ressources

#### Tableau de consommation RAM

| Configuration | RAM de base | Pic mémoire (fork) | Total requis |
|---------------|-------------|-------------------|--------------|
| **No persistence** | 10 GB | - | 10 GB |
| **RDB seul** | 10 GB | +5-10 GB (COW) | 15-20 GB |
| **AOF seul** | 10 GB | +0.5 GB (buffers) | 10.5 GB |
| **RDB + AOF** | 10 GB | +5-10 GB (fork) | 15-20 GB |

**Explication** : RDB nécessite jusqu'à 2x la RAM lors du fork (Copy-on-Write).

#### Tableau de consommation disque

| Mécanisme | Taille typique (10 GB RAM) | Croissance | Espace recommandé |
|-----------|---------------------------|------------|-------------------|
| **RDB seul** | 3 GB | Stable | 10 GB (3x) |
| **AOF pur** | 15-20 GB (sans rewrite) | Continue | 50 GB (5x) |
| **AOF hybride** | 4-5 GB | Modérée | 20 GB (2x) |
| **RDB + AOF** | 7-8 GB (total) | Modérée | 30 GB (3x) |

#### Tableau d'utilisation I/O

| Configuration | Pattern I/O | Fréquence | Impact SSD | Impact HDD |
|---------------|-------------|-----------|------------|------------|
| **RDB seul** | Burst (écriture snapshot) | Périodique (5-15 min) | Faible | Moyen |
| **AOF everysec** | Continu (fsync 1/s) | Constant | Moyen | Élevé |
| **AOF always** | Continu (fsync chaque op) | Maximum | Très élevé | Critique |
| **RDB + AOF** | Burst + Continu | Mixte | Moyen | Élevé |

### 4. Temps de récupération (RTO)

#### Tableau de temps de chargement

| Dataset | RDB | AOF pur | AOF hybride | Impact métier |
|---------|-----|---------|-------------|---------------|
| **1 GB** | 5s | 30s | 8s | Négligeable |
| **10 GB** | 30s | 5 min | 45s | Acceptable |
| **50 GB** | 3 min | 25 min | 4 min | Impact moyen |
| **100 GB** | 6 min | 50 min | 8 min | Impact élevé |
| **500 GB** | 30 min | 4h+ | 40 min | Critique |

**Facteurs d'influence :**
- Type de disque (SSD >> HDD)
- Type de données (entiers < strings < blobs)
- Compression activée ou non
- Charge CPU pendant le chargement

#### Comparaison en situation critique

**Scénario : Datacenter principal down, bascule vers DR**

```
RDB seul :
1. Copier RDB depuis backup (5 min)
2. Charger RDB (2 min pour 20 GB)
Total: 7 minutes
Perte de données : Derniers 15 minutes

AOF pur :
1. Copier AOF depuis backup (15 min, fichier plus gros)
2. Charger AOF (10 min pour 20 GB AOF)
Total: 25 minutes
Perte de données : Dernière seconde

AOF hybride :
1. Copier AOF depuis backup (7 min)
2. Charger AOF hybride (3 min)
Total: 10 minutes
Perte de données : Dernière seconde

Verdict : AOF hybride = meilleur compromis RTO/RPO
```

### 5. Complexité opérationnelle

#### Tableau de complexité de gestion

| Aspect | RDB | AOF | Hybride |
|--------|-----|-----|---------|
| **Configuration initiale** | ⭐⭐⭐⭐⭐ Simple | ⭐⭐⭐⭐ Simple | ⭐⭐⭐ Moyenne |
| **Monitoring** | ⭐⭐⭐⭐⭐ Simple | ⭐⭐⭐ Moyen | ⭐⭐⭐ Moyen |
| **Backup** | ⭐⭐⭐⭐⭐ Un fichier | ⭐⭐⭐ Fichier(s) + taille | ⭐⭐⭐⭐ Acceptable |
| **Restauration** | ⭐⭐⭐⭐⭐ Très simple | ⭐⭐⭐⭐ Simple | ⭐⭐⭐⭐ Simple |
| **Troubleshooting** | ⭐⭐⭐ Moyen | ⭐⭐⭐⭐ Bon (logs) | ⭐⭐⭐ Moyen |
| **Gestion de la taille** | ⭐⭐⭐⭐⭐ Automatique | ⭐⭐ Réécritures requises | ⭐⭐⭐ Réécritures auto |
| **Debugging** | ⭐⭐ Binaire | ⭐⭐⭐⭐⭐ Lisible | ⭐⭐⭐ Mixte |

#### Maintenance requise

**RDB seul :**
```bash
# Maintenance minimale
- Vérifier succès des snapshots : quotidien
- Backups : automatisé
- Espace disque : stable
```

**AOF :**
```bash
# Maintenance régulière
- Vérifier réécritures AOF : quotidien
- Surveiller taille AOF : continu
- Forcer réécriture si nécessaire : hebdomadaire
- Vérifier fsync delays : continu
- Nettoyer anciens fichiers AOF : mensuel
```

**Hybride :**
```bash
# Maintenance modérée
- Surveiller RDB + AOF : quotidien
- Vérifier cohérence : hebdomadaire
- Backups des deux : automatisé
- Espace disque : surveiller
```

### 6. Cas d'usage et recommandations

#### Matrice de décision par RPO

| RPO acceptable | Configuration recommandée | Justification |
|----------------|--------------------------|---------------|
| **1 heure+** | RDB seul (save 3600 1) | Performance max, simplicité |
| **15-30 minutes** | RDB seul (save 900 1) | Bon compromis cache |
| **5 minutes** | RDB fréquent (save 300 10) | Cache critique |
| **1 minute** | RDB très fréquent + surveillance | Limite avant AOF |
| **1 seconde** | **AOF everysec + RDB** | ✅ Production standard |
| **0 seconde** | AOF always + Réplication | Finance, compliance |

#### Matrice de décision par type d'application

| Type d'application | RDB seul | AOF seul | RDB + AOF | Justification |
|-------------------|----------|----------|-----------|---------------|
| **Cache API/DB** | ✅ Recommandé | ❌ Overkill | ❌ Overkill | Données recalculables |
| **Cache avec warm-up long** | ✅ Acceptable | ⚠️ Possible | ⚠️ Possible | Équilibre perf/durabilité |
| **Session store** | ❌ Perte sessions | ✅ Acceptable | ✅ **Recommandé** | Expérience utilisateur |
| **Job queues** | ❌ Perte jobs | ✅ Minimum | ✅ **Recommandé** | Intégrité des tâches |
| **Compteurs métiers** | ❌ Perte données | ✅ Minimum | ✅ **Recommandé** | Exactitude requise |
| **Leaderboards** | ⚠️ Acceptable | ✅ Bon | ✅ **Recommandé** | Équité joueurs |
| **Données financières** | ❌ Inacceptable | ✅ AOF always | ✅ AOF always + RDB | Compliance |
| **Logs/Analytics** | ⚠️ Acceptable | ✅ Bon | ✅ Recommandé | Dépend criticité |

#### Matrice de décision par contraintes

| Contrainte principale | Configuration | Compromis |
|----------------------|---------------|-----------|
| **Budget limité** | RDB seul | Performance, perte données |
| **Performance critique** | RDB seul ou AOF no | Durabilité |
| **Espace disque limité** | RDB seul | Durabilité |
| **RAM limitée** | AOF seul | Pas de fork |
| **Durabilité critique** | AOF always | Performance |
| **Compliance stricte** | AOF always + Réplication | Performance, coût |
| **Équilibre optimal** | **RDB + AOF everysec** | Aucun (meilleur compromis) |

### 7. Cas d'échec et résilience

#### Tableau de résilience aux pannes

| Type de panne | RDB | AOF everysec | AOF always | Hybride |
|---------------|-----|--------------|------------|---------|
| **Crash Redis propre** | ✅ Pas de perte | ✅ Pas de perte | ✅ Pas de perte | ✅ Pas de perte |
| **Kill -9 (brutal)** | ❌ Perte 5-15min | ⚠️ Perte ~1s | ✅ Pas de perte | ⚠️ Perte ~1s |
| **Crash OS** | ❌ Perte 5-15min | ⚠️ Perte buffer OS | ⚠️ Perte possible | ⚠️ Perte ~1s |
| **Panne disque** | ❌ Tout perdu | ❌ Tout perdu | ❌ Tout perdu | ❌ Tout perdu |
| **Corruption fichier** | ❌ Irréparable | ✅ Réparable | ✅ Réparable | ⚠️ Partiellement |
| **Disque plein** | ⚠️ Snapshot échoue | ❌ Bloque écritures | ❌ Bloque écritures | ❌ Bloque |
| **OOM (Out of Memory)** | ❌ Fork impossible | ✅ Continue | ✅ Continue | ⚠️ Fork impossible |

#### Stratégies de mitigation

**Pour RDB :**
```
Problème : Perte de données importante
Solutions :
- Snapshots plus fréquents (save 60 100)
- + Réplication (1 master + 2 replicas)
- + Backups réguliers
- Accepter le compromis pour les caches
```

**Pour AOF :**
```
Problème : Performance réduite
Solutions :
- Utiliser SSD
- Format hybride (Redis 7+)
- Réplication pour distribuer la charge
- Scaling horizontal si nécessaire
```

**Pour Hybride :**
```
Avantage : Cumule les bénéfices
- Durabilité d'AOF
- Performance proche de RDB
- Récupération rapide
- Réparation possible
```

## Comparaison par scénario réel

### Scénario 1 : E-commerce - Cache de produits

**Contexte :**
- 50,000 produits en cache
- 10,000 requêtes/seconde
- Warm-up depuis DB : 5 minutes
- Perte de cache = Charge DB × 10

**Analyse :**

| Critère | RDB | AOF | Hybride |
|---------|-----|-----|---------|
| Perte acceptable ? | ✅ Oui (5 min) | ✅ Oui | ✅ Oui |
| Performance ? | ✅ Optimale | ⚠️ Acceptable | ⚠️ Acceptable |
| Coût ? | ✅ Faible | ⚠️ Moyen | ⚠️ Moyen |

**Recommandation :**
```conf
# RDB seul suffit (cache)
save 900 1
save 300 10
appendonly no
```

**Justification :** Données recalculables, performance critique.

### Scénario 2 : SaaS - Session store

**Contexte :**
- 100,000 sessions actives
- Durée session : 1-4 heures
- Perte = Déconnexion utilisateurs
- Impact business : Élevé

**Analyse :**

| Critère | RDB | AOF | Hybride |
|---------|-----|-----|---------|
| Perte acceptable ? | ❌ Non (15 min) | ✅ Oui (1s) | ✅ Oui (1s) |
| Performance ? | ✅ Optimale | ✅ Bonne | ✅ Bonne |
| Expérience utilisateur ? | ❌ Mauvaise | ✅ Bonne | ✅ Bonne |

**Recommandation :**
```conf
# Hybride (durabilité + performance)
save 900 1
save 300 10
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
```

**Justification :** Équilibre entre durabilité et performance.

### Scénario 3 : Gaming - Leaderboard temps réel

**Contexte :**
- Classement de 10 millions de joueurs
- 50,000 mises à jour/seconde
- Perte = Injustice perçue
- Réputation critique

**Analyse :**

| Critère | RDB | AOF | Hybride |
|---------|-----|-----|---------|
| Équité ? | ❌ Perte 15 min | ✅ Perte 1s | ✅ Perte 1s |
| Performance ? | ✅ Excellente | ⚠️ Acceptable | ⚠️ Acceptable |
| Intégrité ? | ❌ Compromise | ✅ Maintenue | ✅ Maintenue |

**Recommandation :**
```conf
# Hybride + Réplication
# Master
save 900 1
save 300 10
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# + 2 Replicas pour HA
```

**Justification :** Intégrité critique + haute disponibilité.

### Scénario 4 : Finance - Transactions

**Contexte :**
- Transactions bancaires
- Aucune perte tolérée
- Compliance PCI-DSS
- Audit trail obligatoire

**Analyse :**

| Critère | RDB | AOF | Hybride |
|---------|-----|-----|---------|
| Conformité ? | ❌ Non | ✅ Oui (always) | ✅ Oui (always) |
| Audit ? | ❌ Non | ✅ Complet | ✅ Complet |
| Durabilité ? | ❌ Insuffisante | ✅ Maximale | ✅ Maximale |
| Performance ? | ✅ Bonne | ❌ Faible | ❌ Faible |

**Recommandation :**
```conf
# AOF always + Réplication synchrone
appendonly yes
appendfsync always
aof-use-rdb-preamble yes
save 3600 1  # RDB pour backup simple

# Architecture :
# - 3 masters (load balancing)
# - 2 replicas par master
# - Réplication synchrone
# - Backups multi-région
```

**Justification :** Durabilité absolue requise, coût justifié.

### Scénario 5 : IoT - Time-series data

**Contexte :**
- 100,000 capteurs
- 1 million de points/minute
- Analyse historique
- Perte partielle acceptable

**Analyse :**

| Critère | RDB | AOF | Hybride |
|---------|-----|-----|---------|
| Volume ? | ⚠️ Snapshots fréquents | ⚠️ AOF très gros | ✅ Géré |
| Performance ? | ✅ Bonne | ⚠️ Acceptable | ✅ Bonne |
| Perte acceptable ? | ✅ Oui (5 min) | ✅ Oui | ✅ Oui |

**Recommandation :**
```conf
# Hybride avec réécritures agressives
save 300 10
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# Réécritures fréquentes (volume élevé)
auto-aof-rewrite-percentage 50
auto-aof-rewrite-min-size 32mb

# Alternative : RedisTimeSeries (module dédié)
```

**Justification :** Volume élevé nécessite gestion active de l'AOF.

## Recommandations de production

### Configuration par niveau de criticité

#### Niveau 1 : Non-critique (Cache pur)

**Configuration :**
```conf
# RDB espacé
save 3600 1
save 900 10
appendonly no

# Optionnel : désactiver complètement
save ""
appendonly no
```

**Caractéristiques :**
- Performance maximale
- Perte acceptable
- Simplicité opérationnelle

**Exemples :** Cache API externe, cache de contenu statique

#### Niveau 2 : Critique (Standard)

**Configuration :**
```conf
# Hybride (RECOMMANDÉ)
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Caractéristiques :**
- Équilibre optimal
- Perte max 1 seconde
- Production-ready

**Exemples :** Session store, job queues, leaderboards

#### Niveau 3 : Très critique (Finance, Santé)

**Configuration :**
```conf
# AOF always + RDB + Réplication
save 3600 1  # RDB pour backups

appendonly yes
appendfsync always
aof-use-rdb-preamble yes

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
no-appendfsync-on-rewrite no

# + Réplication
replicaof <master-ip> <master-port>  # Sur replicas
```

**Caractéristiques :**
- Durabilité maximale
- Performance sacrifiée
- Compliance

**Exemples :** Transactions financières, dossiers médicaux

### Checklist de décision

#### Questions à se poser

1. **Quelle perte de données est acceptable ?**
   - Aucune → AOF always
   - 1 seconde → AOF everysec
   - 5-15 minutes → RDB seul
   - Toutes → No persistence

2. **Les données sont-elles recalculables ?**
   - Oui → RDB seul acceptable
   - Non → AOF obligatoire

3. **Quel est le volume d'écritures ?**
   - Faible (<1000/s) → Toutes options
   - Moyen (1K-10K/s) → RDB ou hybride
   - Élevé (>10K/s) → RDB ou AOF everysec
   - Très élevé (>100K/s) → RDB seul ou sharding

4. **Quelle est la contrainte de performance ?**
   - Critique → RDB seul
   - Importante → Hybride
   - Normale → Hybride
   - Secondaire → AOF always acceptable

5. **Y a-t-il des exigences de compliance ?**
   - Oui (audit) → AOF obligatoire
   - Non → Libre choix

6. **Quel est le budget infrastructure ?**
   - Limité → RDB seul
   - Moyen → Hybride
   - Élevé → AOF always + Réplication

### Arbre de décision

```
Données recalculables ?
├─ OUI → Cache
│   ├─ Warm-up court (<1min) → No persistence
│   └─ Warm-up long (>1min) → RDB seul
│
└─ NON → Données importantes
    │
    ├─ Perte acceptable > 5 min ?
    │   └─ OUI → RDB seul (save 300 10)
    │
    └─ NON → Durabilité requise
        │
        ├─ Perte acceptable ~1 sec ?
        │   └─ OUI → RDB + AOF everysec ✅ (90% des cas)
        │
        └─ NON → Aucune perte tolérée
            └─ → AOF always + Réplication
```

## Coût total de possession (TCO)

### Comparaison des coûts

**Hypothèse :** Dataset 20 GB, 10,000 ops/s

| Configuration | Serveur | Disque | Backup | Monitoring | Total/mois |
|---------------|---------|--------|--------|------------|------------|
| **RDB seul** | 1x (32GB RAM) | 100 GB | 50 GB S3 | Standard | 250€ |
| **AOF everysec** | 1x (32GB RAM) | 150 GB SSD | 100 GB S3 | Standard | 320€ |
| **Hybride** | 1x (32GB RAM) | 150 GB SSD | 80 GB S3 | Standard | 300€ |
| **AOF always** | 3x (32GB RAM) | 150 GB SSD | 100 GB S3 | Avancé | 950€ |

**Analyse :**
- RDB : Coût minimal, perte de données
- Hybride : +20% coût, durabilité excellente
- AOF always : +280% coût, durabilité maximale

### Retour sur investissement (ROI)

**Calcul du coût d'une perte de données :**

| Type d'incident | RDB seul | Hybride | Économie |
|-----------------|----------|---------|----------|
| Perte 15 min sessions (e-commerce) | 50K€ | 0€ | 50K€ |
| Perte 10 min jobs | 20K€ | 0€ | 20K€ |
| Perte compliance (finance) | 500K€+ | 0€ | 500K€+ |

**Conclusion :** Le surcoût du mode hybride (20%) est négligeable face au coût d'un incident.

## Évolution et migrations

### Passer de RDB à AOF

```bash
# 1. Activer AOF sans arrêt
redis-cli CONFIG SET appendonly yes

# 2. Attendre la première réécriture AOF
# (Redis va créer l'AOF depuis l'état actuel)

# 3. Vérifier
redis-cli INFO persistence | grep aof_enabled
aof_enabled:1

# 4. Rendre permanent dans redis.conf
echo "appendonly yes" >> /etc/redis/redis.conf
```

**Downtime :** Aucun

### Passer de AOF à RDB

```bash
# 1. Désactiver AOF
redis-cli CONFIG SET appendonly no

# 2. Forcer un snapshot
redis-cli BGSAVE

# 3. Vérifier
redis-cli LASTSAVE

# 4. Supprimer les fichiers AOF (optionnel)
rm /var/lib/redis/appendonly.aof*

# 5. Rendre permanent
sed -i 's/appendonly yes/appendonly no/' /etc/redis/redis.conf
```

**Downtime :** Aucun

### Migrer vers le format hybride (Redis 7+)

```bash
# 1. Mettre à jour Redis vers 7.0+

# 2. Activer le format hybride
redis-cli CONFIG SET aof-use-rdb-preamble yes

# 3. Forcer une réécriture pour convertir
redis-cli BGREWRITEAOF

# 4. Vérifier la nouvelle structure
ls -lh /var/lib/redis/appendonlydir/
# appendonly.aof.1.base.rdb
# appendonly.aof.1.incr.aof
# appendonly.aof.manifest

# 5. Rendre permanent
echo "aof-use-rdb-preamble yes" >> /etc/redis/redis.conf
```

**Gain :** Taille fichier -60%, temps de chargement -70%

## Mythes et idées reçues

### Mythe 1 : "AOF est toujours plus lent que RDB"

**Réalité :**
- AOF everysec : -20 à -30% performance (acceptable)
- AOF always : -99% performance (vrai)
- Impact dépend du disque (SSD vs HDD)

### Mythe 2 : "RDB seul suffit toujours pour un cache"

**Réalité :**
- Cache simple : Oui
- Cache avec warm-up long (>5 min) : AOF aide
- Cache critique (business impact) : Hybride recommandé

### Mythe 3 : "On ne peut pas réparer un fichier RDB corrompu"

**Réalité :**
- RDB : Effectivement non réparable
- AOF : Réparable avec redis-check-aof
- Solution : Toujours avoir des backups multiples

### Mythe 4 : "Le format hybride est moins fiable"

**Réalité :**
- Format hybride (Redis 7+) cumule les avantages
- Plus compact que AOF pur
- Plus rapide à charger
- Aussi durable

### Mythe 5 : "Il faut choisir entre RDB et AOF"

**Réalité :**
- Les deux peuvent (et doivent) coexister
- Configuration hybride = best practice
- Seul cas RDB ou AOF seul : contraintes spécifiques

## Résumé exécutif

### Recommandation universelle

**Pour 90% des cas de production :**
```conf
# Configuration hybride (RDB + AOF everysec)
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Pourquoi ?**
- ✅ Durabilité excellente (perte max 1s)
- ✅ Performance acceptable (-20 à -30%)
- ✅ Récupération rapide (format hybride)
- ✅ Backups simples (RDB)
- ✅ Audit possible (AOF)
- ✅ Production-ready

### Cas particuliers

| Situation | Configuration |
|-----------|---------------|
| Cache pur | RDB seul |
| Finance/Santé | AOF always + Réplication |
| Write-heavy (>50K/s) | Cluster + RDB + AOF |
| Budget très limité | RDB seul + Backups fréquents |
| Compliance audit | AOF always |

### Points clés à retenir

1. **Il n'y a pas de solution universelle** : Le choix dépend de vos contraintes
2. **La configuration hybride est un excellent compromis** pour la majorité des cas
3. **RDB privilégie la performance**, AOF privilégie la durabilité
4. **Le format hybride (Redis 7+)** combine les avantages des deux
5. **Toujours tester** vos procédures de restauration
6. **Les backups ne remplacent pas la réplication** (et inversement)
7. **Monitorer activement** les métriques de persistance

---


⏭️ [Stratégies hybrides pour la production](/05-persistance-fiabilite/05-strategies-hybrides-production.md)
