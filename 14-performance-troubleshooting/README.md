🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 14 : Performance et Troubleshooting Redis

## 🎯 Objectifs du module

Ce module avancé vous permettra de :
- Diagnostiquer et résoudre les problèmes de performance Redis en production
- Maîtriser les outils de profiling et d'analyse mémoire
- Identifier et corriger les goulots d'étranglement
- Mettre en place une méthodologie systématique de troubleshooting
- Optimiser les performances et la stabilité de vos instances Redis

---

## 📊 Vue d'ensemble : Les défis de performance Redis

### Symptômes courants en production

Redis est réputé pour ses performances exceptionnelles, mais plusieurs facteurs peuvent dégrader ces performances :

| Symptôme | Impact | Causes probables |
|----------|--------|------------------|
| **Latence élevée** | Timeouts clients, dégradation UX | Commandes lentes, réseau, CPU |
| **Consommation mémoire excessive** | OOM, évictions fréquentes | Big keys, memory leaks, fragmentation |
| **Évictions fréquentes** | Cache miss, perte de données | Mémoire insuffisante, mauvaise politique |
| **Réplication en retard** | Données désynchronisées | Charge élevée, réseau lent |
| **Connexions saturées** | Refus de connexion | Pool mal configuré, leaks |
| **CPU 100%** | Ralentissement global | Commandes O(N), trafic élevé |

### Principe de base : Redis est single-threaded

**Implication critique** : Une seule commande lente peut bloquer toutes les autres requêtes.

```
┌─────────────────────────────────────┐
│     Client 1: GET key (0.1ms)       │ ✅ Rapide
├─────────────────────────────────────┤
│     Client 2: KEYS * (5000ms)       │ ⚠️ BLOQUE TOUT
├─────────────────────────────────────┤
│     Client 3: GET key (ATTEND...)   │ ❌ Timeout
├─────────────────────────────────────┤
│     Client 4: SET key (ATTEND...)   │ ❌ Timeout
└─────────────────────────────────────┘
```

**Règle d'or** : Dans Redis, une commande lente affecte tous les clients.

---

## 🔍 Méthodologie systématique de troubleshooting

### 1. Approche descendante (Top-Down)

Suivez cette méthodologie en 7 étapes pour tout problème de performance :

```
┌──────────────────────────────────────────────────┐
│  ÉTAPE 1 : COLLECTE DES SYMPTÔMES                │
│  - Quand le problème apparaît-il ?               │
│  - Quelle est la fréquence ?                     │
│  - Y a-t-il des patterns temporels ?             │
└──────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────┐
│  ÉTAPE 2 : VÉRIFICATION DE L'ÉTAT GLOBAL         │
│  - redis-cli INFO (métriques clés)               │
│  - État de la mémoire, CPU, réseau               │
│  - Logs système et Redis                         │
└──────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────┐
│  ÉTAPE 3 : ANALYSE DE LA LATENCE                 │
│  - LATENCY DOCTOR                                │
│  - SLOWLOG                                       │
│  - Analyse réseau                                │
└──────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────┐
│  ÉTAPE 4 : ANALYSE MÉMOIRE                       │
│  - Utilisation totale                            │
│  - Fragmentation                                 │
│  - Big keys / Hot keys                           │
└──────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────┐
│  ÉTAPE 5 : ANALYSE DES COMMANDES                 │
│  - Types de commandes exécutées                  │
│  - Patterns d'accès                              │
│  - Charges anormales                             │
└──────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────┐
│  ÉTAPE 6 : DIAGNOSTIC APPROFONDI                 │
│  - Tests de charge contrôlés                     │
│  - Reproduction du problème                      │
│  - Isolation de la cause                         │
└──────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────┐
│  ÉTAPE 7 : CORRECTION ET VALIDATION              │
│  - Application de la solution                    │
│  - Tests de validation                           │
│  - Monitoring post-correction                    │
└──────────────────────────────────────────────────┘
```

### 2. Quick Check : Les 5 commandes essentielles

Avant toute investigation approfondie, exécutez ces commandes :

```bash
# 1. État global de l'instance
redis-cli INFO stats

# 2. Utilisation mémoire et fragmentation
redis-cli INFO memory

# 3. Commandes lentes récentes
redis-cli SLOWLOG GET 10

# 4. État de la latence
redis-cli --latency-history

# 5. Clients connectés et état
redis-cli CLIENT LIST
```

**Temps d'exécution** : < 30 secondes
**Objectif** : Obtenir une vision 360° rapide de l'état de Redis

---

## 🛠️ Arsenal d'outils de troubleshooting

### Outils natifs Redis

| Commande/Outil | Objectif | Cas d'usage | Impact |
|----------------|----------|-------------|---------|
| **INFO** | Métriques globales | Monitoring continu | ✅ Aucun |
| **SLOWLOG** | Commandes lentes | Profiling | ✅ Minimal |
| **LATENCY** | Analyse de latence | Diagnostics latence | ✅ Minimal |
| **MEMORY DOCTOR** | Analyse mémoire | Problèmes mémoire | ✅ Minimal |
| **--bigkeys** | Détection big keys | Scan périodique | ⚠️ Moyen |
| **--memkeys** | Analyse mémoire par clé | Deep dive mémoire | ⚠️ Moyen |
| **MONITOR** | Capture en temps réel | Debug urgent | ❌ Élevé |
| **CLIENT LIST** | État des connexions | Problèmes connexions | ✅ Aucun |
| **SCRIPT DEBUG** | Debug Lua | Scripts complexes | ⚠️ Dev only |

### Outils externes recommandés

```
┌─────────────────────────────────────────────────┐
│  MONITORING & OBSERVABILITÉ                     │
├─────────────────────────────────────────────────┤
│  • Redis Exporter + Prometheus + Grafana        │
│  • Redis Insight (GUI officiel)                 │
│  • New Relic / Datadog / AppDynamics            │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  BENCHMARKING & LOAD TESTING                    │
├─────────────────────────────────────────────────┤
│  • redis-benchmark (natif)                      │
│  • memtier_benchmark (avancé)                   │
│  • Apache JMeter                                │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  ANALYSE RÉSEAU & SYSTÈME                       │
├─────────────────────────────────────────────────┤
│  • tcpdump / wireshark                          │
│  • iftop / nethogs                              │
│  • strace / perf                                │
└─────────────────────────────────────────────────┘
```

---

## 🎭 Les 7 problèmes classiques et leur signature

### 1. **Commandes O(N) bloquantes**

**Signature** :
- Pics de latence soudains
- CPU à 100% pendant les pics
- SLOWLOG montre des commandes KEYS, SMEMBERS, HGETALL

**Commandes RED FLAG** :
```
KEYS *              ❌ Scan complet du keyspace
SMEMBERS big_set    ❌ Récupération d'un gros set
HGETALL big_hash    ❌ Récupération d'un gros hash
SORT large_list     ❌ Tri en mémoire
SUNION big_sets     ❌ Union de gros sets
```

### 2. **Memory Fragmentation**

**Signature** :
- `used_memory_rss` >> `used_memory`
- Ratio de fragmentation > 1.5
- Mémoire système élevée sans données proportionnelles

**Calcul de fragmentation** :
```
fragmentation_ratio = used_memory_rss / used_memory

Valeurs :
< 1.0   : Swapping (DANGER)
1.0-1.5 : Normal
> 1.5   : Fragmentation excessive
> 2.0   : Problème critique
```

### 3. **Big Keys**

**Signature** :
- Latence élevée sur certaines opérations
- Mémoire haute mais peu de clés
- Commandes lentes sur des clés spécifiques

**Seuils problématiques** :
```
String  : > 10 MB
List    : > 10,000 éléments
Set     : > 10,000 membres
Hash    : > 10,000 champs
Sorted  : > 10,000 membres
```

### 4. **Hot Keys**

**Signature** :
- Latence élevée même sur des GET simples
- CPU haute
- Concentration du trafic sur quelques clés

**Pattern typique** :
```
95% du trafic sur 5% des clés
→ Goulot d'étranglement single-thread
→ Impossible à scale horizontalement
```

### 5. **Connection Pool saturation**

**Signature** :
- Erreurs "connection refused"
- Timeouts côté application
- `CLIENT LIST` montre beaucoup de connexions idle

**Causes courantes** :
- Pool trop petit
- Connection leaks (non-fermeture)
- Timeout trop court
- Pic de trafic soudain

### 6. **Évictions excessives**

**Signature** :
- `evicted_keys` augmente rapidement
- Cache hit rate en baisse
- `used_memory` proche de `maxmemory`

**Cascade d'effets** :
```
Évictions ↑ → Cache miss ↑ → Charge DB ↑ → Latence ↑
```

### 7. **Réplication Lag**

**Signature** :
- `master_repl_offset` >> `slave_repl_offset`
- Données différentes master/replica
- Buffer de réplication plein

**Causes** :
- Bande passante insuffisante
- Charge élevée sur le master
- Réplication bloquée par des commandes lentes

---

## 📈 Méthodologie d'analyse de performance

### Framework METRICSS

Un acronyme pour guider votre analyse :

```
M - Memory (Mémoire)
    ├─ Utilisation totale
    ├─ Fragmentation
    └─ Big keys

E - Evictions (Évictions)
    ├─ Taux d'éviction
    ├─ Politique utilisée
    └─ Hit/Miss ratio

T - Throughput (Débit)
    ├─ Ops/sec
    ├─ Network I/O
    └─ Commands/sec par type

R - Replication (Réplication)
    ├─ Lag master/replica
    ├─ Buffer de réplication
    └─ État de sync

I - I/O Operations
    ├─ Persistance (RDB/AOF)
    ├─ Disk I/O
    └─ Fsync latency

C - Connections (Connexions)
    ├─ Nombre de clients
    ├─ Rejected connections
    └─ Client buffer

S - Slowlog & CPU
    ├─ Commandes lentes
    ├─ CPU utilization
    └─ Latency spikes

S - System Resources
    ├─ RAM disponible
    ├─ Swap
    └─ Network bandwidth
```

### Processus d'investigation type

#### Phase 1 : Triage (5 minutes)

```bash
# Quick health check
redis-cli INFO | grep -E "used_memory_human|evicted_keys|connected_clients|instantaneous_ops_per_sec|mem_fragmentation_ratio"

# Check des commandes lentes
redis-cli SLOWLOG GET 5

# État de latence
redis-cli --latency
```

**Décision** : Problème mineur ✅ → Monitoring | Problème majeur ❌ → Phase 2

#### Phase 2 : Diagnostic (15 minutes)

```bash
# Analyse mémoire détaillée
redis-cli INFO memory > memory_snapshot.txt
redis-cli MEMORY DOCTOR

# Identification des big keys
redis-cli --bigkeys

# Analyse des clients
redis-cli CLIENT LIST | grep -v "idle=0"

# Capture de patterns (avec précaution)
redis-cli MONITOR | head -n 1000
```

#### Phase 3 : Root Cause Analysis (30+ minutes)

- Corrélation avec les métriques système (CPU, RAM, I/O)
- Analyse des logs applicatifs
- Reproduction en environnement de test
- Tests de charge contrôlés

#### Phase 4 : Résolution et validation

- Application du correctif
- Tests A/B si possible
- Monitoring intensif post-correctif
- Documentation de l'incident

---

## 🚨 Scénarios d'urgence : Quick Fixes

### Urgence 1 : Redis à 100% CPU

**Action immédiate** :
```bash
# 1. Identifier la commande qui bloque
redis-cli SLOWLOG GET 1

# 2. Tuer le client problématique si nécessaire
redis-cli CLIENT LIST | grep "cmd=KEYS"
redis-cli CLIENT KILL ID <client-id>

# 3. Vérifier qu'il n'y a pas de MONITOR actif
redis-cli CLIENT LIST | grep "flag=M"
```

### Urgence 2 : Mémoire saturée (OOM imminant)

**Action immédiate** :
```bash
# 1. Vérifier l'état actuel
redis-cli INFO memory | grep -E "used_memory|maxmemory|evicted_keys"

# 2. Forcer des évictions si nécessaire
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 3. Identifier et supprimer les big keys temporaires
redis-cli --bigkeys
# Puis supprimer manuellement les clés temporaires volumineuses

# 4. Si critique : augmenter temporairement maxmemory
redis-cli CONFIG SET maxmemory 8gb
```

### Urgence 3 : Latence soudaine et inexpliquée

**Action immédiate** :
```bash
# 1. Vérifier si c'est un problème système
redis-cli --latency-history

# 2. Vérifier si c'est lié à la persistance
redis-cli INFO persistence | grep -E "rdb_bgsave_in_progress|aof_rewrite_in_progress"

# 3. Désactiver temporairement la persistance si nécessaire
redis-cli CONFIG SET save ""
redis-cli CONFIG SET appendonly no

# 4. Vérifier le réseau
ping -c 10 <redis-host>
```

---

## 📊 Tableaux de référence : Seuils et alertes

### Seuils de métriques critiques

| Métrique | Bon | Attention | Critique | Action |
|----------|-----|-----------|----------|---------|
| **Latency P99** | < 1ms | 1-5ms | > 5ms | Investigation |
| **Memory Usage** | < 75% | 75-90% | > 90% | Scale up |
| **Fragmentation** | 1.0-1.5 | 1.5-2.0 | > 2.0 | Restart |
| **Hit Ratio** | > 90% | 70-90% | < 70% | Optimisation |
| **Evictions/sec** | 0 | < 10 | > 100 | Augmenter RAM |
| **Connected Clients** | < 70% max | 70-90% | > 90% | Augmenter limite |
| **CPU Usage** | < 60% | 60-80% | > 80% | Optimisation |
| **Replication Lag** | < 1s | 1-10s | > 10s | Investigation |

### Commandes : Complexité et risques

| Commande | Complexité | Risque | Alternative |
|----------|------------|--------|-------------|
| GET | O(1) | ✅ Aucun | - |
| SET | O(1) | ✅ Aucun | - |
| KEYS * | O(N) | ❌ Bloquant | SCAN |
| SMEMBERS | O(N) | ⚠️ Si gros set | SSCAN |
| HGETALL | O(N) | ⚠️ Si gros hash | HSCAN |
| LRANGE 0 -1 | O(N) | ⚠️ Si grosse list | Pagination |
| SORT | O(N log N) | ❌ Bloquant | Sorted Set |
| SUNION | O(N) | ⚠️ Si gros sets | SSCAN multiple |
| ZUNIONSTORE | O(N*M) | ⚠️ Si nombreux sets | Optimiser |

---

## 🎓 Best Practices de troubleshooting

### DO ✅

1. **Toujours monitorer en continu** (Prometheus + Grafana)
2. **Conserver un historique de SLOWLOG**
3. **Documenter chaque incident** (post-mortem)
4. **Tester les correctifs en dev/staging d'abord**
5. **Avoir des runbooks pour les scénarios courants**
6. **Utiliser redis-cli --latency-history régulièrement**
7. **Faire des backups avant toute intervention majeure**
8. **Analyser les patterns d'accès régulièrement**

### DON'T ❌

1. **NE JAMAIS utiliser MONITOR en production** (sauf urgence absolue < 1 min)
2. **NE JAMAIS utiliser KEYS * sur une instance de production**
3. **NE PAS redémarrer Redis sans comprendre la cause**
4. **NE PAS modifier la configuration sans backup**
5. **NE PAS ignorer les warnings de MEMORY DOCTOR**
6. **NE PAS augmenter maxmemory sans analyser la cause**
7. **NE PAS désactiver la persistance sans raison valable**
8. **NE PAS utiliser FLUSHALL/FLUSHDB sans triple confirmation**

### Checklist pré-intervention

Avant toute modification majeure en production :

```
☐ Backup de la configuration actuelle
☐ Snapshot des données (RDB) si applicable
☐ Fenêtre de maintenance planifiée
☐ Plan de rollback préparé
☐ Équipe en alerte/disponible
☐ Monitoring intensif activé
☐ Communication aux utilisateurs si nécessaire
☐ Documentation de la procédure
```

---

## 🔗 Structure du module

Ce module est organisé en 9 sections progressives :

1. **[Profiling : Slowlog et analyse des commandes lentes](01-profiling-slowlog-commandes-lentes.md)**
   - Configuration et utilisation du SLOWLOG
   - Analyse des patterns de commandes lentes
   - Identification des goulots d'étranglement

2. **[Memory Analysis : --bigkeys, --memkeys et memory doctor](02-memory-analysis-bigkeys-memkeys.md)**
   - Détection des big keys
   - Analyse de la distribution mémoire
   - Utilisation de MEMORY DOCTOR

3. **[Debugging avancé : MONITOR, CLIENT LIST, CLIENT KILL](03-debugging-avance-monitor-client.md)**
   - Capture du trafic en temps réel
   - Gestion des connexions clients
   - Debugging de problèmes complexes

4. **[Problèmes de latence : Causes et solutions](04-problemes-latence-causes-solutions.md)**
   - Identification des sources de latence
   - LATENCY DOCTOR et LATENCY MONITOR
   - Solutions par type de latence

5. **[Out of Memory (OOM) : Diagnostic et résolution](05-out-of-memory-diagnostic-resolution.md)**
   - Prévention et détection précoce
   - Stratégies de récupération
   - Optimisation de l'utilisation mémoire

6. **[Corruption de données et recovery](06-corruption-donnees-recovery.md)**
   - Détection de corruption
   - Procédures de récupération
   - Prévention

7. **[Fragmentation mémoire : Detection et défragmentation](07-fragmentation-memoire-defragmentation.md)**
   - Métriques de fragmentation
   - Active defragmentation
   - Quand redémarrer vs défragmenter

8. **[Tuning et optimisation des commandes](08-tuning-optimisation-commandes.md)**
   - Optimisation des patterns d'accès
   - Refactoring de commandes coûteuses
   - Utilisation efficace des structures de données

9. **[Benchmarking avec redis-benchmark](09-benchmarking-redis-benchmark.md)**
   - Utilisation de redis-benchmark
   - Interprétation des résultats
   - Tests de charge réalistes

---

## 📚 Ressources complémentaires

### Documentation officielle
- [Redis Latency Troubleshooting](https://redis.io/docs/management/optimization/latency/)
- [Redis Memory Optimization](https://redis.io/docs/management/optimization/memory-optimization/)
- [Redis Administration](https://redis.io/docs/management/admin/)

### Outils recommandés
- **Redis Insight** : GUI officiel pour monitoring et debugging
- **redis-cli** : Outil en ligne de commande indispensable
- **Prometheus + Grafana** : Stack de monitoring moderne
- **redis-benchmark** : Outil de benchmarking natif

### Communauté
- Redis Slack / Discord
- Stack Overflow (tag: redis)
- Redis GitHub Issues

---

## 🎯 Prérequis pour ce module

Avant d'aborder ce module avancé, vous devriez maîtriser :

- ✅ Les structures de données Redis (Module 2)
- ✅ Le cycle de vie de la donnée (Module 4)
- ✅ La persistance (Module 5)
- ✅ Le monitoring de base (Module 13)
- ✅ Les commandes Redis INFO, CONFIG

**Niveau** : Avancé (DevOps / SRE / Architecte)

---

## 📝 Ce que vous saurez faire après ce module

À l'issue de ce module, vous serez capable de :

- ✅ Diagnostiquer 90% des problèmes de performance Redis
- ✅ Utiliser tous les outils natifs de troubleshooting
- ✅ Mettre en place une méthodologie systématique
- ✅ Optimiser des instances Redis en production
- ✅ Prévenir les problèmes avant qu'ils ne surviennent
- ✅ Documenter et résoudre les incidents complexes
- ✅ Former vos équipes aux bonnes pratiques

---

**⚠️ Note importante** : Le troubleshooting de production nécessite prudence et méthodologie. Testez toujours vos interventions en environnement de développement/staging avant de les appliquer en production. En cas de doute, préférez l'escalade vers des experts plutôt qu'une intervention hasardeuse.

**🚀 Prochain chapitre** : [14.1 - Profiling : Slowlog et analyse des commandes lentes](./01-profiling-slowlog-commandes-lentes.md)

⏭️ [Profiling : Slowlog et analyse des commandes lentes](/14-performance-troubleshooting/01-profiling-slowlog-commandes-lentes.md)
