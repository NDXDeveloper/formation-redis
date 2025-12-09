🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 5 : Persistance et fiabilité des données

## Introduction

Redis est avant tout une **base de données en mémoire** (in-memory database), ce qui lui confère des performances exceptionnelles avec des temps de réponse de l'ordre de la microseconde. Cependant, cette architecture pose une question fondamentale : **que se passe-t-il en cas de redémarrage du serveur ou de panne ?**

Par défaut, sans mécanisme de persistance, toutes les données stockées dans Redis seraient perdues lors d'un arrêt du processus. C'est pourquoi Redis propose plusieurs mécanismes de **persistance sur disque** permettant de garantir la durabilité des données tout en préservant les performances.

## Pourquoi la persistance est-elle critique ?

### Cas d'usage nécessitant la persistance

| Cas d'usage | Niveau de criticité | Perte de données acceptable |
|-------------|---------------------|----------------------------|
| **Cache pur** | Faible | Oui - Les données peuvent être reconstruites |
| **Session store** | Moyenne | Limitée - Déconnexion des utilisateurs |
| **Job queues** | Élevée | Non - Perte de tâches en cours |
| **Compteurs métiers** | Très élevée | Non - Impact financier potentiel |
| **Base de données primaire** | Critique | Aucune - Conformité et intégrité |

### Impact sur l'architecture

La stratégie de persistance choisie influence directement :

- **La fiabilité** : Garanties en cas de panne (RPO - Recovery Point Objective)
- **Les performances** : Impact des écritures sur disque
- **La capacité de récupération** : Temps de redémarrage (RTO - Recovery Time Objective)
- **L'utilisation des ressources** : CPU, I/O disque, espace de stockage

## Vue d'ensemble des mécanismes de persistance

Redis propose **trois approches principales** pour la persistance des données :

### 1. RDB (Redis Database Backup)

**Principe** : Création périodique de snapshots complets de l'état de la base de données.

**Caractéristiques** :
- Format binaire compact
- Snapshots à intervalles définis
- Opération non-bloquante (fork du processus)
- Fichier unique `.rdb`

**Avantages** :
- ✅ Très compact et rapide à charger
- ✅ Performance d'écriture minimalement impactée
- ✅ Idéal pour les backups
- ✅ Parfait pour la réplication

**Inconvénients** :
- ❌ Perte de données possible entre deux snapshots
- ❌ Fork coûteux en mémoire (copy-on-write)
- ❌ RPO potentiellement élevé (minutes)

### 2. AOF (Append Only File)

**Principe** : Journalisation de toutes les commandes d'écriture dans un fichier log.

**Caractéristiques** :
- Format texte lisible (commandes Redis)
- Enregistrement continu ou périodique (fsync)
- Fichier `.aof` qui grandit continuellement
- Réécriture périodique (compaction)

**Avantages** :
- ✅ Durabilité maximale (jusqu'à 1 seconde de perte)
- ✅ RPO très faible
- ✅ Format auditable et réparable
- ✅ Append-only plus robuste face à la corruption

**Inconvénients** :
- ❌ Fichiers plus volumineux que RDB
- ❌ Potentiellement plus lent au redémarrage
- ❌ Impact sur les performances d'écriture
- ❌ Nécessite des réécritures périodiques

### 3. Hybride RDB + AOF

**Principe** : Combinaison des deux approches pour bénéficier de leurs avantages respectifs.

**Caractéristiques** :
- RDB pour les snapshots périodiques
- AOF pour les modifications récentes
- Stratégie recommandée en production

## Comparaison détaillée des stratégies

### Tableau comparatif : RDB vs AOF vs Hybride

| Critère | RDB seul | AOF seul | Hybride (RDB + AOF) |
|---------|----------|----------|---------------------|
| **Durabilité** | ⭐⭐ Moyenne | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐⭐ Excellente |
| **Perte de données max** | Minutes | 1 seconde | 1 seconde |
| **Performances écriture** | ⭐⭐⭐⭐⭐ Excellentes | ⭐⭐⭐ Bonnes | ⭐⭐⭐⭐ Très bonnes |
| **Temps de récupération** | ⭐⭐⭐⭐⭐ Très rapide | ⭐⭐⭐ Moyen | ⭐⭐⭐⭐ Rapide |
| **Taille fichier** | ⭐⭐⭐⭐⭐ Compact | ⭐⭐ Volumineux | ⭐⭐⭐ Moyen |
| **Utilisation CPU** | ⭐⭐⭐⭐ Faible | ⭐⭐⭐ Moyenne | ⭐⭐⭐ Moyenne |
| **Utilisation I/O** | ⭐⭐⭐⭐ Faible | ⭐⭐ Élevée | ⭐⭐⭐ Moyenne |
| **Complexité gestion** | ⭐⭐⭐⭐⭐ Simple | ⭐⭐⭐⭐ Simple | ⭐⭐⭐ Moyenne |
| **Idéal pour backups** | ⭐⭐⭐⭐⭐ Oui | ⭐⭐ Non | ⭐⭐⭐⭐ Oui |

### Tableau comparatif : Configurations fsync AOF

| Mode fsync | Durabilité | Performance | Perte de données max | Usage recommandé |
|------------|------------|-------------|----------------------|------------------|
| **no** | ⭐ | ⭐⭐⭐⭐⭐ | Jusqu'à 30 secondes | Jamais en production |
| **everysec** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ~1 seconde | **Recommandé par défaut** |
| **always** | ⭐⭐⭐⭐⭐ | ⭐⭐ | Aucune* | Données critiques uniquement |

*Sauf crash du système d'exploitation ou panne matérielle

## Recommandations de production

### Matrice de décision selon le cas d'usage

| Type d'application | Stratégie recommandée | Configuration |
|--------------------|-----------------------|---------------|
| **Cache volatile** | RDB uniquement | Snapshots espacés (30min-1h) |
| **Cache avec warm-up critique** | RDB + AOF | RDB 15min, AOF everysec |
| **Session store** | AOF everysec | + Réplication Master-Replica |
| **Message queues / Jobs** | AOF everysec | + RDB backups réguliers |
| **Base de données primaire** | **Hybride (RDB + AOF)** | RDB + AOF everysec + Réplication |
| **Données financières** | AOF always | + Réplication synchrone + Backups |

### Configuration de production recommandée (baseline)

```conf
# Stratégie hybride recommandée
save 900 1        # Snapshot après 900s si au moins 1 modification
save 300 10       # Snapshot après 300s si au moins 10 modifications
save 60 10000     # Snapshot après 60s si au moins 10000 modifications

appendonly yes                    # Activer AOF
appendfilename "appendonly.aof"
appendfsync everysec             # Compromis durabilité/performance optimal

# AOF Rewrite (compaction automatique)
auto-aof-rewrite-percentage 100  # Réécrire quand taille double
auto-aof-rewrite-min-size 64mb   # Taille minimale avant réécriture

# Format AOF (Redis 7+)
aof-use-rdb-preamble yes        # Préambule RDB + delta AOF (hybride)

# Répertoire de travail
dir /var/lib/redis
```

### Checklist de production

#### ✅ Configuration de base
- [ ] Activer RDB avec snapshots réguliers
- [ ] Activer AOF avec fsync everysec
- [ ] Configurer `aof-use-rdb-preamble yes` (Redis 7+)
- [ ] Définir un répertoire de travail avec permissions appropriées

#### ✅ Haute disponibilité
- [ ] Configurer au minimum 1 replica (idéalement 2+)
- [ ] Activer la réplication asynchrone
- [ ] Mettre en place Redis Sentinel pour le failover automatique
- [ ] Tester régulièrement les scénarios de basculement

#### ✅ Backups et disaster recovery
- [ ] Automatiser les backups des fichiers RDB
- [ ] Stocker les backups hors du serveur Redis (S3, NAS, etc.)
- [ ] Définir une politique de rétention (ex: 7 jours + 4 semaines + 6 mois)
- [ ] Tester régulièrement la restauration depuis backup
- [ ] Documenter la procédure de recovery (RTO/RPO)

#### ✅ Monitoring
- [ ] Surveiller la taille des fichiers AOF/RDB
- [ ] Alerter sur les échecs de snapshot
- [ ] Monitorer les métriques `rdb_last_save_time` et `aof_last_rewrite_time`
- [ ] Suivre les performances I/O disque

#### ✅ Optimisation système
- [ ] Désactiver Transparent Huge Pages (THP)
- [ ] Configurer `vm.overcommit_memory = 1`
- [ ] Assurer un espace disque suffisant (2-3x la RAM)
- [ ] Utiliser des disques SSD pour AOF si possible

## Compromis fondamentaux à comprendre

### Le triangle impossible : CAP appliqué à Redis

```
        Durabilité
            ▲
           ╱ ╲
          ╱   ╲
         ╱     ╲
        ╱       ╲
       ╱         ╲
Performance ◄───► Simplicité
```

**Vous devez choisir vos priorités :**

1. **Performance maximale** → RDB espacé, pas d'AOF
   - Perte de données acceptable
   - Cas d'usage : cache pur

2. **Durabilité maximale** → AOF always + Réplication
   - Impact sur les performances
   - Cas d'usage : données financières

3. **Équilibre optimal** → Hybride RDB + AOF everysec
   - Compromis raisonnable
   - **Cas d'usage : 90% des applications en production**

### Facteurs de décision clés

| Question | Impact sur la décision |
|----------|------------------------|
| Quelle perte de données est acceptable ? | Définit le mode fsync AOF |
| Quel est le temps de récupération tolérable ? | Influence RDB vs AOF |
| Quel est le volume de modifications ? | Impacte la fréquence de réécriture AOF |
| Quelles sont les contraintes de coût ? | Détermine l'infrastructure nécessaire |
| Existe-t-il une source de vérité externe ? | Permet cache sans persistance forte |

## Scénarios de failure et stratégies de mitigation

### Types de pannes et impact

| Type de panne | RDB seul | AOF everysec | Hybride | Mitigation |
|---------------|----------|--------------|---------|------------|
| **Arrêt propre** | ✅ Pas de perte | ✅ Pas de perte | ✅ Pas de perte | - |
| **Crash Redis** | ❌ Dernières minutes | ⚠️ ~1 seconde | ⚠️ ~1 seconde | Réplication |
| **Crash OS** | ❌ Dernières minutes | ⚠️ Données en buffer | ⚠️ Données en buffer | AOF always |
| **Panne disque** | ❌ Toutes données | ❌ Toutes données | ❌ Toutes données | Réplication + Backups |
| **Corruption fichier** | ⚠️ Snapshot invalide | ⚠️ Réparable | ⚠️ Réparable | Outils redis-check-* |

### Plan de reprise d'activité (DRP)

**Objectifs à définir :**

- **RPO (Recovery Point Objective)** : Quelle perte de données maximum ?
  - Cache : 1 heure acceptable
  - Session store : 1 minute acceptable
  - Transactionnel : 0 seconde requis

- **RTO (Recovery Time Objective)** : Combien de temps pour restaurer ?
  - Petit dataset (<10GB) : quelques secondes
  - Dataset moyen (10-100GB) : quelques minutes
  - Gros dataset (>100GB) : peut prendre 10-30 minutes

**Stratégie de reprise :**

1. **Tier 1 - Haute disponibilité** : Réplication + Sentinel (basculement automatique en <1min)
2. **Tier 2 - Disaster Recovery** : Backups dans datacenter secondaire (récupération en <1h)
3. **Tier 3 - Catastrophe** : Backups géographiquement distribués (récupération en <24h)

## Évolution des mécanismes de persistance

### Redis 7+ : Améliorations significatives

**Format AOF hybride** :
- Préambule RDB compact
- Suivi des modifications en AOF
- Meilleur des deux mondes
- Activation : `aof-use-rdb-preamble yes`

**AOF Multi-Part (Redis 7.0+)** :
- Fichiers AOF segmentés (base + incremental)
- Réécriture non-bloquante améliorée
- Meilleure gestion des gros datasets

**Amélioration des performances** :
- Optimisation du fork() pour RDB
- Réduction de la latence lors des fsync
- Meilleure gestion mémoire lors des snapshots

## Considérations pour le cloud et les conteneurs

### Environnements cloud

**Limitations à connaître :**
- Les disques cloud (EBS, Azure Disks) ont des IOPS limitées
- Impact sur AOF avec fort taux d'écriture
- Coût du stockage pour les gros volumes

**Bonnes pratiques cloud :**
- Utiliser des volumes SSD optimisés IOPS (io1/io2 sur AWS)
- Activer les backups automatiques du provider
- Considérer les solutions managées (ElastiCache, MemoryDB)
- Snapshot réguliers vers S3/Blob Storage

### Conteneurs et orchestration

**Kubernetes et Docker :**
- **TOUJOURS** utiliser des volumes persistants (PersistentVolumes)
- Éviter les volumes éphémères pour les données persistées
- Configurer des PVC avec StorageClass appropriées
- Tester la reprise après destruction de pod

**Exemple problématique (à éviter) :**
```yaml
# ❌ MAUVAIS : Données éphémères
spec:
  containers:
  - name: redis
    image: redis:7
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}  # Perdu à chaque restart !
```

**Configuration recommandée :**
```yaml
# ✅ BON : Persistance garantie
spec:
  containers:
  - name: redis
    image: redis:7
    volumeMounts:
    - name: redis-data
      mountPath: /data
  volumes:
  - name: redis-data
    persistentVolumeClaim:
      claimName: redis-pvc
```

## Points clés à retenir

### Règles d'or de la persistance Redis

1. **Ne jamais désactiver la persistance en production** (sauf cache pur acceptant la perte totale)
2. **Privilégier l'approche hybride RDB + AOF** pour 90% des cas d'usage
3. **Toujours tester vos procédures de restauration** avant d'en avoir besoin
4. **La réplication ne remplace pas les backups** (corruption répliquée, suppression accidentelle)
5. **Monitorer activement** les métriques de persistance

### Métriques critiques à surveiller

| Métrique | Commande | Seuil d'alerte | Action |
|----------|----------|----------------|--------|
| Dernier snapshot réussi | `INFO persistence` | > 2x intervalle configuré | Vérifier logs, espace disque |
| Taille fichier AOF | `INFO persistence` | > 10x taille RDB | Forcer réécriture AOF |
| Temps réécriture AOF | `INFO stats` | > 5 minutes | Optimiser config ou hardware |
| Échecs de save | `INFO stats` | > 0 | Investiguer immédiatement |
| Fragmentation mémoire | `INFO memory` | > 1.5 | Considérer restart |

## Structure du module

Ce module est organisé en sections progressives pour approfondir chaque aspect de la persistance :

1. **Le dilemme : Vitesse vs Durabilité** - Comprendre les compromis fondamentaux
2. **RDB (Redis Database)** - Maîtriser les snapshots et leur fonctionnement interne
3. **AOF (Append Only File)** - Comprendre la journalisation et la sécurité maximale
4. **Comparaison RDB vs AOF** - Analyse détaillée des avantages et inconvénients
5. **Stratégies hybrides** - Configurations optimales pour la production
6. **Backup et restauration** - Bonnes pratiques et procédures de recovery

Chaque section approfondit un aspect spécifique avec des exemples concrets, des configurations recommandées et des cas d'usage réels.

---

## Ressources complémentaires

### Documentation officielle
- [Redis Persistence - Official Documentation](https://redis.io/docs/management/persistence/)
- [Redis Configuration - persistence section](https://redis.io/docs/management/config/)

### Outils utiles
- `redis-check-rdb` : Vérifier l'intégrité des fichiers RDB
- `redis-check-aof` : Vérifier et réparer les fichiers AOF
- `redis-cli --rdb` : Générer un snapshot à la demande

### Commandes de diagnostic
```bash
# Forcer un snapshot immédiat
redis-cli BGSAVE

# Forcer une réécriture AOF
redis-cli BGREWRITEAOF

# Vérifier l'état de la persistance
redis-cli INFO persistence

# Dernière sauvegarde réussie (timestamp Unix)
redis-cli LASTSAVE
```

---


⏭️ [Le dilemme : Vitesse vs Durabilité](/05-persistance-fiabilite/01-dilemme-vitesse-vs-durabilite.md)
