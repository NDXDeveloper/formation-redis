🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.1 Le dilemme : Vitesse vs Durabilité

## Introduction

Redis incarne parfaitement le **compromis fondamental** de toute base de données : il est impossible d'avoir simultanément des performances maximales ET une durabilité absolue des données. Ce dilemme est au cœur de toute décision d'architecture impliquant Redis.

Le principe est simple : **chaque garantie de durabilité a un coût en performance**, et inversement, chaque optimisation de performance réduit les garanties de durabilité.

### Le théorème fondamental

```
Performance ⬆️  =  Durabilité ⬇️
Performance ⬇️  =  Durabilité ⬆️
```

**Pourquoi ce dilemme existe-t-il ?**

1. **Redis = In-Memory Database** : Les données vivent en RAM (ultra-rapide)
2. **La durabilité = Écriture disque** : Nécessite des I/O physiques (lentes)
3. **Loi physique immuable** : RAM >> SSD >> HDD en termes de vitesse

## Comprendre les ordres de grandeur

### Latences typiques des opérations

| Opération | Latence typique | Débit | Facteur de vitesse |
|-----------|-----------------|-------|-------------------|
| **Lecture RAM** | 100 ns | - | Référence (1x) |
| **Lecture SSD** | 100 μs | 500 MB/s | 1000x plus lent |
| **Lecture HDD** | 10 ms | 150 MB/s | 100 000x plus lent |
| **Réseau (1Gbps)** | 0.5-1 ms | 125 MB/s | 5 000-10 000x plus lent |
| **Redis GET** | 0.1-1 ms (avec réseau) | 100K+ ops/s | - |
| **Redis SET** | 0.1-1 ms (avec réseau) | 100K+ ops/s | - |
| **Redis SET + fsync disque** | 5-20 ms | 50-200 ops/s | **50-200x plus lent** |

### Impact visuel des écritures disque

```
Sans persistance (RAM uniquement) :
[GET][GET][GET][GET][GET][GET][GET][GET][GET][GET]  ← 100K ops/seconde
Temps: 10μs par opération

Avec persistance (fsync each write) :
[SET + fsync........][SET + fsync........][SET + ...]  ← 100-200 ops/seconde
Temps: 5-10ms par opération
```

**Conclusion** : Les écritures disque sont **50 à 1000 fois plus lentes** que les opérations en RAM pure.

## Le spectre Vitesse-Durabilité

Redis offre un **continuum de configurations** permettant de choisir son point d'équilibre :

```
Vitesse maximale                                    Durabilité maximale
│                                                                      │
│    Cache pur     │  RDB espacé  │   RDB fréquent + AOF   │  AOF always │
│    (no persist)  │  (5-15 min)  │      (everysec)        │  (chaque op) │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
   0% durabilité                                           100% durabilité
   100K+ ops/s                                             50-200 ops/s
```

### Configurations détaillées du spectre

| Configuration | Perte max | Débit écriture | Latence P99 | Cas d'usage |
|---------------|-----------|----------------|-------------|-------------|
| **Aucune persistance** | Toutes données | 100K+ ops/s | <1ms | Cache régénérable |
| **RDB seul (15min)** | 15 minutes | 90K+ ops/s | <2ms | Cache critique |
| **RDB (5min) + AOF no** | 5 minutes | 80K+ ops/s | <3ms | Non recommandé |
| **RDB + AOF everysec** | ~1 seconde | 60-80K ops/s | <5ms | **Production standard** |
| **AOF always** | Aucune* | 50-500 ops/s | 10-50ms | Données critiques |

*Sauf crash kernel ou panne matérielle

## Analyse approfondie des compromis

### 1. Mode "Sans persistance" (Cache pur)

**Configuration :**
```conf
# redis.conf
save ""          # Désactiver tous les snapshots RDB
appendonly no    # Désactiver AOF
```

**Caractéristiques :**
- ✅ **Performance maximale absolue** : 100K+ ops/s
- ✅ **Latence minimale** : <1ms P99
- ✅ **Aucune utilisation disque**
- ✅ **Aucun overhead CPU/IO**
- ❌ **Perte totale des données** au redémarrage
- ❌ **Reconstruction nécessaire** depuis source

**Quand l'utiliser :**
- Cache de requêtes SQL recalculables
- Cache de résultats d'API externes
- Sessions avec authentification stateless
- Données temporaires non critiques

**Métriques de production :**
```
Ops/sec: 100,000+
Latence P50: 0.2ms
Latence P99: 0.8ms
CPU usage: 15-25%
I/O disque: 0%
```

### 2. Mode "RDB périodique seul"

**Configuration :**
```conf
# redis.conf
save 900 1       # Snapshot si 1+ modifications en 15min
save 300 10      # Snapshot si 10+ modifications en 5min
save 60 10000    # Snapshot si 10K+ modifications en 1min
appendonly no
```

**Caractéristiques :**
- ✅ **Performances excellentes** : 80-90K ops/s
- ✅ **Fichiers compacts** : Format binaire optimisé
- ✅ **Backups simples** : Un seul fichier .rdb
- ✅ **Restauration rapide** : Chargement binaire direct
- ⚠️ **Perte de données possible** : Entre deux snapshots (5-15min)
- ❌ **Fork coûteux en mémoire** : Copy-on-write
- ❌ **Pics de CPU/IO** : Lors des snapshots

**Quand l'utiliser :**
- Cache avec warm-up acceptable
- Session store (perte de quelques minutes acceptable)
- Leaderboards, statistiques non critiques

**Métriques de production :**
```
Ops/sec: 80,000-90,000
Latence P50: 0.3ms
Latence P99: 1.5ms
CPU usage: 20-30% (pics à 50% lors du save)
I/O disque: Pics périodiques
Mémoire: 2x dataset lors du fork
```

**Impact du fork (copy-on-write) :**
```
Dataset: 10 GB en RAM
Fork pour RDB: Réservation de 10 GB supplémentaires (20 GB total)
Durée du snapshot: 2-5 secondes (SSD), 10-30 secondes (HDD)
Pendant le snapshot: Légère augmentation latence (10-20%)
```

### 3. Mode "Hybride RDB + AOF everysec" (RECOMMANDÉ)

**Configuration :**
```conf
# redis.conf - Configuration production recommandée
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec              # ← Compromis optimal
aof-use-rdb-preamble yes          # Redis 7+ : format hybride

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Caractéristiques :**
- ✅ **Bon équilibre durabilité/performance** : 60-80K ops/s
- ✅ **Perte minimale** : Maximum 1 seconde de données
- ✅ **Récupération robuste** : AOF + dernier snapshot RDB
- ✅ **Production-ready** : Testé et éprouvé
- ⚠️ **Impact modéré sur performance** : -20 à -30% vs no-persist
- ⚠️ **Gestion AOF nécessaire** : Réécritures périodiques
- ⚠️ **Utilisation disque** : Fichiers AOF + RDB

**Quand l'utiliser :**
- **90% des cas de production**
- Session store critique
- Job queues
- Compteurs métiers
- Toute donnée non recalculable

**Métriques de production :**
```
Ops/sec: 60,000-80,000
Latence P50: 0.5ms
Latence P99: 3-5ms
CPU usage: 25-35%
I/O disque: Modéré (fsync chaque seconde)
Fiabilité: Perte max 1 seconde
```

**Évolution de la latence :**
```
Sans AOF:     [0.2ms] [0.2ms] [0.2ms] [0.2ms] [0.2ms]
Avec AOF:     [0.3ms] [0.4ms] [0.3ms] [2ms*]  [0.3ms]
              ↑                        ↑
              Normal                   fsync toutes les 1s
```

### 4. Mode "AOF always" (Durabilité maximale)

**Configuration :**
```conf
# redis.conf - Durabilité maximale
save ""                    # Optionnel: désactiver RDB
appendonly yes
appendfilename "appendonly.aof"
appendfsync always         # ← fsync après CHAQUE commande

aof-use-rdb-preamble yes
```

**Caractéristiques :**
- ✅ **Durabilité maximale** : Aucune perte (sauf crash hardware)
- ✅ **Garanties ACID** : Chaque écriture confirmée sur disque
- ✅ **Audit trail complet** : Toutes les opérations journalisées
- ❌ **Performance dégradée** : 50-500 ops/s (100-1000x plus lent)
- ❌ **Latence élevée** : 10-50ms P99
- ❌ **Forte utilisation disque** : I/O constants

**Quand l'utiliser :**
- Données financières (transactions, paiements)
- Systèmes de vote ou enchères
- Audit logs critiques
- Compliance stricte (RGPD, SOX, PCI-DSS)

**Métriques de production :**
```
Ops/sec: 50-500 (dépend fortement du disque)
Latence P50: 5-10ms
Latence P99: 20-50ms
CPU usage: 15-25% (bloqué par I/O)
I/O disque: Très élevé (100% du temps)
Fiabilité: Maximale
```

**Impact dramatique sur le débit :**
```
Test benchmark:
- Sans persistance: 98,000 SET/s
- AOF everysec:     67,000 SET/s  (-31%)
- AOF always:          380 SET/s  (-99.6%)
```

## Tableau décisionnel : Choisir sa configuration

### Matrice RPO vs Performance

| RPO acceptable | Perte données max | Configuration | Performance | Complexité |
|----------------|-------------------|---------------|-------------|------------|
| **Infini** | Toutes | Aucune persistence | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **15 minutes** | 15 minutes | RDB espacé (15min) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **5 minutes** | 5 minutes | RDB fréquent (5min) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **1 seconde** | ~1 seconde | **RDB + AOF everysec** | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **0 seconde** | Aucune* | AOF always | ⭐⭐ | ⭐⭐⭐ |

*Sauf défaillance matérielle grave

### Impact sur les ressources systèmes

| Configuration | CPU | RAM | I/O Disque | Espace disque | Réseau |
|---------------|-----|-----|------------|---------------|--------|
| **No persist** | Faible | 1x dataset | Aucun | Aucun | Normal |
| **RDB seul** | Pics | 2x dataset (fork) | Pics | 1x dataset | Normal |
| **AOF everysec** | Moyen | 1x dataset | Continu modéré | 2-5x dataset | Normal |
| **AOF always** | Moyen | 1x dataset | Continu élevé | 2-5x dataset | Normal |
| **Hybride** | Moyen | 2x dataset (fork) | Continu modéré | 3-6x dataset | Normal |

### Guide de décision par secteur d'activité

| Secteur | Configuration recommandée | RPO | Justification |
|---------|---------------------------|-----|---------------|
| **E-commerce (panier)** | RDB + AOF everysec | 1s | Perte acceptable, frustration limitée |
| **E-commerce (inventaire)** | AOF always | 0s | Impact financier direct |
| **Finance (trading)** | AOF always + Réplication | 0s | Conformité réglementaire |
| **Finance (cache cours)** | RDB 5min ou no persist | 5min | Recalculable depuis source |
| **Gaming (leaderboard)** | RDB + AOF everysec | 1s | Équité et engagement joueurs |
| **Gaming (cache assets)** | RDB 15min | 15min | Reconstructible |
| **SaaS (sessions)** | RDB + AOF everysec | 1s | Expérience utilisateur |
| **SaaS (cache API)** | RDB 15min | 15min | Régénérable |
| **IoT (telemetry)** | RDB + AOF everysec | 1s | Analytics et monitoring |
| **Médias sociaux (feed)** | RDB 5-15min | 5-15min | Régénérable, pas critique |
| **Médias sociaux (messages)** | AOF everysec | 1s | Engagement et rétention |

## Les pièges courants à éviter

### ❌ Piège #1 : "Désactiver AOF pour la performance"

**Erreur fréquente :**
```conf
# ❌ MAUVAIS pour des données non-cache
appendonly no
save 900 1
```

**Risque :** Perte de 15 minutes de données lors d'un crash.

**Scénario réel :**
- E-commerce : 15 minutes de commandes perdues
- Gaming : 15 minutes de progression perdues
- Session store : Déconnexion massive d'utilisateurs

**Solution :**
```conf
# ✅ BON : Équilibre optimal
appendonly yes
appendfsync everysec
save 900 1
aof-use-rdb-preamble yes
```

### ❌ Piège #2 : "AOF always pour tout"

**Erreur fréquente :**
```conf
# ❌ OVERKILL pour la plupart des cas
appendfsync always
```

**Impact :**
- Performance divisée par 100-200
- Coût infrastructure multiplié par 5-10
- Expérience utilisateur dégradée (latence)

**Quand c'est justifié :**
- Transactions financières uniquement
- Données réglementées
- Systèmes d'enchères en temps réel

**Sinon, préférer :**
```conf
# ✅ BON : Suffisant pour 95% des cas
appendfsync everysec
```

### ❌ Piège #3 : "Oublier les snapshots avec AOF"

**Erreur fréquente :**
```conf
# ❌ INCOMPLET
appendonly yes
appendfsync everysec
save ""  # ← Erreur : désactivation des snapshots
```

**Conséquences :**
- Fichier AOF qui grandit indéfiniment
- Temps de démarrage très long
- Pas de backup simple (AOF = format log)

**Configuration optimale :**
```conf
# ✅ COMPLET : Hybride
appendonly yes
appendfsync everysec
save 900 1         # ← Garder les snapshots !
save 300 10
save 60 10000
aof-use-rdb-preamble yes  # Format hybride
```

## Calcul du coût réel de la durabilité

### Exemple concret : Application e-commerce

**Contexte :**
- 10,000 requêtes/seconde
- 50% lectures, 50% écritures
- Dataset : 20 GB

#### Scénario A : No persistence (cache pur)

```
Serveur requis : 1x instance (4 CPU, 32 GB RAM)
Coût mensuel : 200€
Débit : 10,000 ops/s
Latence P99 : 0.8ms

Risque : Perte totale lors crash (warm-up 5-10 minutes)
```

#### Scénario B : RDB + AOF everysec (production standard)

```
Serveur requis : 1x instance (8 CPU, 64 GB RAM, SSD)
Coût mensuel : 450€
Débit : 7,000 ops/s  (-30%)
Latence P99 : 3ms

Protection : Perte max 1 seconde
```

#### Scénario C : AOF always (durabilité maximale)

```
Serveurs requis : 3x instances + Load balancer
Coût mensuel : 1,800€
Débit : 500 ops/s par instance (paralleliser)
Latence P99 : 25ms

Protection : Aucune perte
```

**Rapport coût/bénéfice :**

| Configuration | Coût | Durabilité | Ratio coût/durabilité |
|---------------|------|------------|-----------------------|
| No persist | 200€ | 0% | - |
| RDB + AOF everysec | 450€ | 99.99% | **Optimal** |
| AOF always | 1,800€ | 99.9999% | Coût x4, gain marginal |

**Conclusion :** Pour la majorité des cas, `RDB + AOF everysec` offre le meilleur rapport coût/bénéfice.

## Recommandations de production par contexte

### Contexte 1 : Startup / MVP

**Contraintes :**
- Budget limité
- Traffic faible (<1000 ops/s)
- Tolérance aux bugs

**Configuration recommandée :**
```conf
# Équilibre coût/fiabilité
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
```

**Justification :** Protection raisonnable sans sur-engineering.

### Contexte 2 : Scale-up / Croissance

**Contraintes :**
- Traffic en augmentation (10K-100K ops/s)
- SLA à respecter (99.9%)
- Budget modéré

**Configuration recommandée :**
```conf
# Production standard + monitoring
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# Optimisations
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**+ Architecture :**
- 1 Master + 2 Replicas
- Redis Sentinel pour failover
- Monitoring (Prometheus + Grafana)

### Contexte 3 : Enterprise / High-availability

**Contraintes :**
- Traffic intense (>100K ops/s)
- SLA strict (99.99% ou plus)
- Conformité réglementaire

**Configuration recommandée :**
```conf
# Durabilité maximale raisonnable
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec  # Pas "always" sauf exigence légale
aof-use-rdb-preamble yes

# Optimisations avancées
hz 100                        # Fréquence serveur interne
tcp-backlog 511
maxmemory-policy allkeys-lru
```

**+ Architecture :**
- Redis Cluster (3 masters + 3 replicas minimum)
- Cross-datacenter replication
- Backups automatiques multi-région
- Monitoring avancé + alerting 24/7

## Optimiser le compromis : Techniques avancées

### 1. Utiliser le format AOF hybride (Redis 7+)

```conf
# ✅ MEILLEUR DES DEUX MONDES
aof-use-rdb-preamble yes
```

**Avantages :**
- Fichier AOF plus compact (préambule RDB)
- Démarrage plus rapide
- Réécritures AOF moins fréquentes

**Gain typique :**
- Taille fichier : -60% vs AOF pur
- Temps de démarrage : -70%

### 2. Ajuster la fréquence de fsync

```conf
# Pour workload avec bursts d'écriture
appendfsync everysec

# Tuning avancé (nécessite Redis recompilé ou module)
# hz 100  # Augmenter pour fsync plus précis
```

### 3. Optimiser les snapshots RDB

```conf
# Snapshots adaptatifs selon le traffic
save 900 1        # Baseline : peu de modifications
save 300 100      # Traffic modéré
save 60 10000     # Traffic intense

# Limiter l'impact du fork
rdb-save-incremental-fsync yes  # Écriture progressive
```

### 4. Utiliser la réplication pour "décharger" la persistance

**Architecture recommandée :**
```
Master (écritures)
  ├─ appendonly yes
  ├─ appendfsync no     ← Pas de fsync sur master (optionnel)
  ├─ save ""            ← Pas de RDB sur master (optionnel)

Replica 1 (lecture + persistance)
  ├─ appendonly yes
  ├─ appendfsync everysec  ← Durabilité sur replica
  ├─ save 900 1
```

**Avantages :**
- Master optimisé pour performance pure
- Replica garantit la durabilité
- Isolation des I/O disque

**Risques :**
- Dépendance à la réplication
- Latence de réplication (ms)
- Complexité opérationnelle

## Métriques à monitorer

### Indicateurs de performance

```bash
# Commande Redis INFO
redis-cli INFO stats | grep ops
instantaneous_ops_per_sec:67823

# Latence
redis-cli --latency
min: 0, max: 12, avg: 0.34 (1524 samples)

# Latence percentiles
redis-cli --latency-history
```

**Seuils d'alerte recommandés :**

| Métrique | Normal | Warning | Critical |
|----------|--------|---------|----------|
| ops/sec | >50K | <50K | <10K |
| Latence P99 | <5ms | 5-20ms | >20ms |
| Latence P999 | <20ms | 20-50ms | >50ms |

### Indicateurs de durabilité

```bash
# Dernier snapshot réussi
redis-cli INFO persistence | grep rdb_last_save_time

# État AOF
redis-cli INFO persistence | grep aof_enabled
redis-cli INFO persistence | grep aof_last_rewrite_time_sec

# Échecs de persistance
redis-cli INFO persistence | grep rdb_last_bgsave_status
redis-cli INFO persistence | grep aof_last_bgrewrite_status
```

**Alertes critiques :**
- `rdb_last_bgsave_status:err` → Snapshot échoué
- `aof_last_bgrewrite_status:err` → Réécriture AOF échouée
- Dernier snapshot > 2x l'intervalle configuré

## Checklist de décision finale

### Questions à se poser

1. **Quelle perte de données est acceptable ?**
   - Aucune → AOF always (+ coût x5-10)
   - 1 seconde → RDB + AOF everysec ✅ (recommandé)
   - 5-15 minutes → RDB seul
   - Toutes → No persistence

2. **Les données sont-elles recalculables ?**
   - Oui (cache) → Persistance légère acceptable
   - Non (source de vérité) → Durabilité forte requise

3. **Quel est le budget infrastructure ?**
   - Limité → Optimiser pour performance
   - Standard → Configuration hybride
   - Élevé → Durabilité maximale + HA

4. **Quelles sont les exigences légales ?**
   - RGPD, SOX, PCI-DSS → AOF + encryption + audit
   - Aucune → Flexibilité

5. **Quel est le SLA visé ?**
   - 99.9% → RDB + AOF everysec + Replica
   - 99.99% → Cluster + Cross-DC replication
   - 99.999% → Active-Active + Durabilité maximale

### Décision rapide par type d'application

```
Cache API/DB pure          → No persistence ou RDB espacé
Session store              → RDB + AOF everysec
Job queues                 → RDB + AOF everysec + Réplication
Compteurs/Analytics        → RDB + AOF everysec
Transactions financières   → AOF always + Réplication + Backups
Leaderboards/Gaming        → RDB + AOF everysec
IoT Time-Series            → RDB + AOF everysec (ou TimeSeries dédié)
```

## Conclusion

Le dilemme vitesse vs durabilité n'a pas de solution universelle. La clé est de :

1. **Comprendre vos contraintes réelles** (RPO, RTO, budget)
2. **Choisir consciemment** votre point d'équilibre
3. **Monitorer activement** les métriques de performance et durabilité
4. **Tester régulièrement** vos procédures de recovery
5. **Réévaluer périodiquement** selon l'évolution de votre application

**Règle d'or** : Pour 90% des applications en production, la configuration `RDB + AOF everysec` offre le meilleur compromis entre performance, durabilité et simplicité opérationnelle.

---

**Points clés à retenir :**
- Les écritures disque sont 50-1000x plus lentes que la RAM
- AOF `everysec` offre le meilleur rapport durabilité/performance
- AOF `always` divise le débit par 100-200
- La configuration hybride RDB + AOF est recommandée pour la production
- Toujours monitorer et tester vos stratégies de persistance

---


⏭️ [RDB (Redis Database) : Snapshots et fonctionnement](/05-persistance-fiabilite/02-rdb-snapshots-fonctionnement.md)
