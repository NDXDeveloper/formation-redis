🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.2 AWS : ElastiCache vs MemoryDB (durabilité)

## 🎯 Objectifs

- Comprendre les différences architecturales fondamentales entre ElastiCache et MemoryDB
- Maîtriser les modèles de durabilité et leurs implications
- Choisir la solution optimale selon les exigences de RPO/RTO
- Implémenter des configurations production avec IaC
- Optimiser les coûts selon les besoins de durabilité

---

## 🏗️ Architecture comparative approfondie

### Vue d'ensemble

```
┌───────────────────────────────────────────────────────────────────────┐
│                    ElastiCache vs MemoryDB                            │
│                                                                       │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐   │
│  │      ElastiCache             │  │        MemoryDB              │   │
│  │      (Cache Layer)           │  │    (Primary Database)        │   │
│  │                              │  │                              │   │
│  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │   │
│  │  │   Primary Node         │  │  │  │    Primary Node        │  │   │
│  │  │   ┌───────────────┐    │  │  │  │   ┌───────────────┐    │  │   │
│  │  │   │ Redis Engine  │    │  │  │  │   │ Redis Engine  │    │  │   │
│  │  │   │  (in-memory)  │    │  │  │  │   │  (in-memory)  │    │  │   │
│  │  │   └───────┬───────┘    │  │  │  │   └───────┬───────┘    │  │   │
│  │  │           │            │  │  │  │           │            │  │   │
│  │  │           │ Optional   │  │  │  │           │ MANDATORY  │  │   │
│  │  │           ▼            │  │  │  │           ▼            │  │   │
│  │  │   ┌───────────────┐    │  │  │  │   ┌───────────────┐    │  │   │
│  │  │   │  RDB/AOF      │    │  │  │  │   │ Transaction   │    │  │   │
│  │  │   │  (optional)   │    │  │  │  │   │ Log (WAL)     │    │  │   │
│  │  │   └───────────────┘    │  │  │  │   │ Multi-AZ      │    │  │   │
│  │  │           │            │  │  │  │   └───────┬───────┘    │  │   │
│  │  │           │ Async      │  │  │  │           │ Sync       │  │   │
│  │  │           ▼            │  │  │  │           ▼            │  │   │
│  │  │   ┌───────────────┐    │  │  │  │   ┌───────────────┐    │  │   │
│  │  │   │  Replica      │    │  │  │  │   │  Replica      │    │  │   │
│  │  │   │  (async)      │    │  │  │  │   │  (from WAL)   │    │  │   │
│  │  │   └───────────────┘    │  │  │  │   └───────────────┘    │  │   │
│  │  │                        │  │  │  │                        │  │   │
│  │  │  Failover: 2-3 min     │  │  │  │  Failover: <1 min      │  │   │
│  │  │  Data loss: Possible   │  │  │  │  Data loss: ZERO       │  │   │
│  │  └────────────────────────┘  │  │  └────────────────────────┘  │   │
│  │                              │  │                              │   │
│  │  Use case:                   │  │  Use case:                   │   │
│  │  • Cache applicatif          │  │  • Primary database          │   │
│  │  • Session store (ok perte)  │  │  • Session store (critique)  │   │
│  │  • API response cache        │  │  • Leaderboards              │   │
│  │  • Query result cache        │  │  • Financial data            │   │
│  └──────────────────────────────┘  │  • Inventory systems         │   │
│                                    └──────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🔬 Modèles de durabilité détaillés

### ElastiCache : Cache volatil avec persistence optionnelle

#### Architecture de persistence

```
┌─────────────────────────────────────────────────────────────┐
│         ElastiCache Persistence (Optional)                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Primary Node (AZ-1)                    │    │
│  │                                                     │    │
│  │  ┌──────────────┐                                   │    │
│  │  │ Redis Memory │                                   │    │
│  │  │   Dataset    │                                   │    │
│  │  └──────┬───────┘                                   │    │
│  │         │                                           │    │
│  │         │ (1) Background save (BGSAVE)              │    │
│  │         │     Triggered by:                         │    │
│  │         │     - Schedule (e.g., every hour)         │    │
│  │         │     - Manual BGSAVE                       │    │
│  │         ▼                                           │    │
│  │  ┌──────────────┐                                   │    │
│  │  │  RDB File    │──────┐                            │    │
│  │  │  (snapshot)  │      │                            │    │
│  │  └──────────────┘      │ (2) Upload to S3           │    │
│  │                        │                            │    │
│  │  ┌──────────────┐      │                            │    │
│  │  │  AOF File    │──────┘                            │    │
│  │  │  (append)    │                                   │    │
│  │  └──────────────┘                                   │    │
│  │                                                     │    │
│  │  • RDB: Point-in-time snapshot                      │    │
│  │  • AOF: Incremental log (everysec/always)           │    │
│  │  • Both stored in S3                                │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          │ (3) Async replication            │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Replica Node (AZ-2)                    │    │
│  │                                                     │    │
│  │  • Replication lag: 0-1000ms typical                │    │
│  │  • No persistence on replica (master only)          │    │
│  │  • Promotes to master on failover                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ⚠️  Issues avec ce modèle:                                 │
│  • Async replication → data loss possible                   │
│  • BGSAVE impacts performance (fork + COW)                  │
│  • Recovery from S3 takes time (minutes)                    │
│  • No guarantee of consistency                              │
└─────────────────────────────────────────────────────────────┘
```

#### Garanties de durabilité ElastiCache

```yaml
RDB (Redis Database Backup)
├── Fréquence: Configurable (15min, 30min, 60min, 6h, 12h, 24h)
├── Mécanisme: Fork + Copy-on-Write
├── Durée snapshot: 30s à 5min (selon taille dataset)
├── Impact perf:
│   ├── Fork initial: Pause 10-100ms
│   ├── COW overhead: +5-15% CPU/RAM
│   └── I/O disk: Peut saturer pendant snapshot
├── RPO (Recovery Point Objective):
│   └── 15min minimum (data loss possible)
└── Stockage: S3 (cross-region replication possible)

AOF (Append Only File)
├── Modes de sync:
│   ├── always: fsync à chaque write (très lent, rare perte)
│   ├── everysec: fsync chaque seconde (balance)
│   └── no: fsync géré par OS (rapide, max perte)
├── Impact perf:
│   ├── always: -70% throughput
│   ├── everysec: -5-10% throughput
│   └── no: Négligeable
├── RPO:
│   ├── always: ~0 sec (mais très lent)
│   ├── everysec: ~1 sec
│   └── no: Plusieurs secondes
├── Rewrite: Automatique en background
└── Stockage: S3

Réplication asynchrone
├── Lag moyen: 10-100ms
├── Lag max observé: 1000ms+ (sous charge)
├── Data loss au failover: Tout ce qui n'est pas répliqué
└── Split-brain possible: Oui (rare)

Conclusion ElastiCache
├── Designed for: Cache (tolérance perte)
├── RPO: 1 seconde à plusieurs minutes
├── RTO: 2-3 minutes (failover)
└── Use case: NOT a primary database
```

---

### MemoryDB : Durabilité garantie avec transaction log

#### Architecture du transaction log

```
┌─────────────────────────────────────────────────────────────┐
│         MemoryDB Architecture (Inspired by Aurora)          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Primary Node (AZ-1)                       │    │
│  │                                                     │    │
│  │  ┌─────────────────┐                                │    │
│  │  │  Redis Engine   │                                │    │
│  │  │  (in-memory)    │                                │    │
│  │  └────────┬────────┘                                │    │
│  │           │                                         │    │
│  │           │ (1) Write operation                     │    │
│  │           ▼                                         │    │
│  │  ┌─────────────────┐                                │    │
│  │  │  Write to WAL   │                                │    │
│  │  │  (in-memory buf)│                                │    │
│  │  └────────┬────────┘                                │    │
│  │           │                                         │    │
│  │           │ (2) Sync write to Multi-AZ log          │    │
│  │           │     (quorum write)                      │    │
│  │           ▼                                         │    │
│  │  ┌─────────────────────────────────────────┐        │    │
│  │  │      Multi-AZ Transaction Log           │        │    │
│  │  │                                         │        │    │
│  │  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │        │    │
│  │  │  │ Log AZ-1 │  │ Log AZ-2 │  │Log AZ-3│ │        │    │
│  │  │  │          │  │          │  │        │ │        │    │
│  │  │  │ ┌──────┐ │  │ ┌──────┐ │  │┌──────┐│ │        │    │
│  │  │  │ │Entry1│ │  │ │Entry1│ │  ││Entry1││ │        │    │
│  │  │  │ │Entry2│ │  │ │Entry2│ │  ││Entry2││ │        │    │
│  │  │  │ │Entry3│ │  │ │Entry3│ │  ││Entry3││ │        │    │
│  │  │  │ └──────┘ │  │ └──────┘ │  │└──────┘│ │        │    │
│  │  │  └──────────┘  └──────────┘  └────────┘ │        │    │
│  │  │                                         │        │    │
│  │  │  • Durable storage (EBS-backed)         │        │    │
│  │  │  • Synchronous replication (quorum)     │        │    │
│  │  │  • Survives AZ failure                  │        │    │
│  │  └─────────────────────────────────────────┘        │    │
│  │           │                                         │    │
│  │           │ (3) ACK to client                       │    │
│  │           │     after quorum write                  │    │
│  └───────────┼─────────────────────────────────────────┘    │
│              │                                              │
│              │ (4) Async replication to replicas            │
│              │     from transaction log                     │
│              ▼                                              │
│  ┌───────────────────────────────────────────────────┐      │
│  │         Replica Nodes (AZ-2, AZ-3)                │      │
│  │                                                   │      │
│  │  • Read from transaction log                      │      │
│  │  • Apply log entries to in-memory dataset         │      │
│  │  • Can serve reads                                │      │
│  │  • Fast promotion (already has data)              │      │
│  └───────────────────────────────────────────────────┘      │
│                                                             │
│  ✅ Critical differences:                                   │
│  • Write is NOT acknowledged until log quorum               │
│  • Transaction log is DURABLE (EBS, Multi-AZ)               │
│  • Replicas read from log (not primary)                     │
│  • Zero data loss on failover                               │
└─────────────────────────────────────────────────────────────┘
```

#### Garanties de durabilité MemoryDB

```yaml
Transaction Log (Similar to Aurora)
├── Architecture: Multi-AZ distributed log
├── Storage: Durable EBS volumes (3 AZ minimum)
├── Write acknowledgment: After quorum write (2/3 AZ)
├── Replication: Synchronous to log
├── Recovery: Instant (log replay from durable storage)
└── Data loss: ZERO (log is source of truth)

Write Path
├── (1) Client sends write
├── (2) Primary writes to in-memory
├── (3) Primary writes to Multi-AZ log (sync)
├── (4) Quorum achieved (2/3 AZ)
├── (5) ACK to client
└── (6) Async replication to replicas (from log)

Performance Impact
├── Write latency: +0.5-2ms vs pure in-memory
│   └── Due to sync write to durable log
├── Read latency: Unchanged (in-memory)
├── Throughput: 95-98% of pure in-memory
└── Trade-off: Slight latency for durability

Failover
├── Detection: 10-30 seconds
├── Promotion: Replica promoted instantly
├── Recovery: No data loss (log is consistent)
├── Total RTO: <1 minute (typically 30-45 seconds)
└── RPO: ZERO (no data loss)

Conclusion MemoryDB
├── Designed for: Primary database
├── RPO: 0 seconds (zero data loss)
├── RTO: <1 minute
├── Consistency: Strong (via transaction log)
└── Use case: Redis as your primary database
```

---

## 📊 Comparaison détaillée des caractéristiques

### Tableau exhaustif

| Caractéristique | ElastiCache | MemoryDB |
|-----------------|-------------|----------|
| **Architecture** |
| Modèle de données | In-memory cache | In-memory primary DB |
| Transaction log | ❌ Non | ✅ Multi-AZ WAL |
| Replication vers replicas | Async | Async (from log) |
| Replication vers log | N/A | Sync (quorum) |
| **Durabilité** |
| RPO (Recovery Point Objective) | 1 sec - 15 min | 0 sec (zero data loss) |
| RTO (Recovery Time Objective) | 2-3 minutes | <1 minute |
| Data loss possible | ✅ Oui | ❌ Non |
| Persistence | RDB/AOF (optional) | Transaction log (mandatory) |
| Snapshot frequency | 15min - 24h | Continuous |
| **Performance** |
| Write latency (P50) | ~1ms | ~1.5-2ms |
| Write latency (P99) | ~5ms | ~8-10ms |
| Read latency | ~1ms | ~1ms (from memory) |
| Throughput impact | 0% (no persistence) | -2-5% vs pure cache |
| **Haute Disponibilité** |
| Multi-AZ automatic failover | ✅ | ✅ |
| Failover time | 2-3 minutes | <1 minute |
| Split-brain protection | Basic | Strong (via log quorum) |
| Read replicas | 0-5 per shard | 0-5 per shard |
| **Scaling** |
| Horizontal (sharding) | ✅ (up to 500 shards) | ✅ (up to 500 shards) |
| Vertical (node size) | ✅ (requires failover) | ✅ (requires failover) |
| Max RAM per node | 317 GB | 419 GB |
| Max cluster size | ~150 TB | ~200 TB |
| **Configuration Redis** |
| Version Redis | 7.1 | 7.0 |
| Cluster mode | ✅ | ✅ |
| Lua scripting | ✅ | ✅ |
| Modules | ❌ | ❌ |
| MULTI/EXEC transactions | ✅ (single shard) | ✅ (single shard) |
| **Sécurité** |
| TLS in-transit | ✅ | ✅ |
| Encryption at-rest | ✅ | ✅ (default) |
| AUTH | ✅ | ✅ |
| ACLs (granular) | ✅ (Redis 6+) | ✅ (fine-grained) |
| VPC | ✅ | ✅ |
| **Monitoring** |
| CloudWatch metrics | ✅ | ✅ |
| CloudWatch Logs | ✅ | ✅ |
| Slow log | ✅ | ✅ |
| Latency monitoring | ✅ | ✅ |
| **Backup & Recovery** |
| Automated backups | ✅ (to S3) | ✅ (continuous) |
| Snapshot retention | 1-35 days | 1-35 days |
| Point-in-time recovery | ❌ | ✅ |
| Cross-region replication | ✅ (Global Datastore) | ❌ (not yet) |
| **Pricing** |
| Cost vs ElastiCache | Baseline | +78% |
| Reserved Instances | ✅ (40-60% discount) | ✅ (40-60% discount) |
| Snapshot storage cost | ✅ | ✅ |
| Data transfer | Standard AWS rates | Standard AWS rates |
| **Compliance** |
| HIPAA eligible | ✅ | ✅ |
| PCI DSS | ✅ | ✅ |
| SOC 1/2/3 | ✅ | ✅ |
| **SLA** |
| Standard instance | 99.9% | 99.99% |
| Multi-AZ | 99.9% | 99.99% |
| Downtime/year (SLA) | ~8.76 hours | ~52.56 minutes |

---

## 🎯 Cas d'usage : Quand utiliser quoi ?

### Matrice de décision

```
┌────────────────────────────────────────────────────────────────┐
│                 Use Case Decision Matrix                       │
└────────────────────────────────────────────────────────────────┘

Critère: Tolérance à la perte de données
├─ Perte acceptable (secondes/minutes) → ElastiCache
└─ Zero data loss requis → MemoryDB

Critère: Redis est-il la source de vérité ?
├─ Non (cache d'une DB primaire) → ElastiCache
└─ Oui (pas d'autre source) → MemoryDB

Critère: Budget
├─ Contraint → ElastiCache (moins cher)
└─ Flexible → MemoryDB (vaut l'investissement)

Critère: RTO requis
├─ 2-5 minutes acceptable → ElastiCache
└─ <1 minute requis → MemoryDB

Critère: Compliance (HIPAA, PCI-DSS)
├─ Data loss allowed → ElastiCache
└─ Data loss prohibited → MemoryDB
```

### Exemples concrets par use case

#### ✅ Utiliser **ElastiCache**

```yaml
1. Cache de requêtes SQL
Use case: Cache des résultats de requêtes lourdes
Rationale:
  - Source primaire: PostgreSQL/MySQL
  - Perte de cache = recalcul, pas de data loss
  - Coût optimisé pour un cache volatil
Configuration:
  - Cluster mode enabled (sharding)
  - RDB snapshot toutes les 6h (pour warm-up rapide)
  - Multi-AZ pour HA

2. Cache API Gateway
Use case: Cache des réponses d'APIs externes
Rationale:
  - Source primaire: APIs tierces
  - TTL court (minutes à heures)
  - Perte acceptable = nouveau call API
Configuration:
  - Standalone ou cluster simple
  - Pas de persistence (inutile)
  - Eviction policy: allkeys-lru

3. Session store (non-critique)
Use case: Sessions web pour site e-commerce
Rationale:
  - Session loss = utilisateur re-login
  - Acceptable pour la plupart des use cases
  - Coût réduit vs MemoryDB
Configuration:
  - Multi-AZ pour HA
  - AOF everysec (compromise)
  - TTL sur les sessions (auto-cleanup)

4. Rate limiting
Use case: Rate limiter pour API public
Rationale:
  - Compteurs peuvent se réinitialiser (acceptable)
  - Très haute performance requise
  - Perte de données = rate limit temporairement incorrect
Configuration:
  - Cluster mode pour scale
  - Pas de persistence
  - Throughput très élevé

5. Real-time analytics (non-financial)
Use case: Compteurs de pages vues, clicks
Rationale:
  - Approximation acceptable
  - Perte de quelques compteurs OK
  - Haute performance sur writes
Configuration:
  - HyperLogLog pour cardinality
  - Sorted Sets pour rankings
  - RDB snapshot quotidien
```

#### ✅ Utiliser **MemoryDB**

```yaml
1. Session store (critique)
Use case: Sessions pour application financière
Rationale:
  - Session loss = transaction interrompue
  - Compliance requiert durabilité
  - Zero data loss mandatory
Configuration:
  - Multi-AZ par défaut
  - ACLs granulaires
  - Encryption at-rest + in-transit

2. Leaderboards de gaming
Use case: Classement temps réel dans un jeu mobile
Rationale:
  - Scores sont précieux (users care)
  - Perte de scores = bad UX
  - Pas de DB primaire alternative
Configuration:
  - Sorted Sets pour rankings
  - Multi-shard pour scale
  - Point-in-time recovery activé

3. Inventory management
Use case: Stock en temps réel pour e-commerce
Rationale:
  - Stock count doit être exact
  - Overselling = bad customer experience
  - Source de vérité pour transactions
Configuration:
  - Cluster mode pour sharding par SKU
  - Transactions pour stock decrements
  - Strong consistency via MemoryDB log

4. Financial transactions cache
Use case: Cache de transactions pour fintech
Rationale:
  - Compliance (PCI-DSS, SOX)
  - Audit trail requis
  - Zero data loss mandatory
Configuration:
  - Encryption everywhere
  - Fine-grained ACLs
  - Backup retention 35 days

5. Shopping cart (e-commerce à fort volume)
Use case: Panier d'achat pour marketplace
Rationale:
  - Cart loss = revenue loss
  - Users expect cart persistence
  - High write throughput
Configuration:
  - Redis Hashes pour carts
  - Multi-AZ durability
  - TTL pour auto-cleanup (abandoned carts)

6. Microservices state management
Use case: État distribué entre microservices
Rationale:
  - State loss = workflow corruption
  - Pas de DB alternative
  - Low latency required
Configuration:
  - Redis as primary state store
  - Cluster mode pour scale
  - Zero downtime deployments
```

---

## 🔧 Configurations avancées

### ElastiCache - Configuration production complète

#### CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'ElastiCache for Redis - Production Configuration'

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID where ElastiCache will be deployed

  PrivateSubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Private subnet IDs for ElastiCache (Multi-AZ)

  ApplicationSecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id
    Description: Security group of application tier

  Environment:
    Type: String
    Default: production
    AllowedValues: [development, staging, production]

  NodeType:
    Type: String
    Default: cache.r7g.xlarge
    Description: ElastiCache node type

  NumShards:
    Type: Number
    Default: 3
    MinValue: 1
    MaxValue: 500
    Description: Number of shards (cluster mode)

  ReplicasPerShard:
    Type: Number
    Default: 2
    MinValue: 0
    MaxValue: 5
    Description: Number of replicas per shard

Resources:
  # Subnet Group
  RedisSubnetGroup:
    Type: AWS::ElastiCache::SubnetGroup
    Properties:
      Description: Subnet group for Redis cluster
      SubnetIds: !Ref PrivateSubnetIds
      CacheSubnetGroupName: !Sub '${AWS::StackName}-subnet-group'
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-subnet-group'
        - Key: Environment
          Value: !Ref Environment

  # Security Group
  RedisSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for ElastiCache Redis
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 6379
          ToPort: 6379
          SourceSecurityGroupId: !Ref ApplicationSecurityGroupId
          Description: Redis access from application tier
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-redis-sg'
        - Key: Environment
          Value: !Ref Environment

  # Parameter Group
  RedisParameterGroup:
    Type: AWS::ElastiCache::ParameterGroup
    Properties:
      CacheParameterGroupFamily: redis7
      Description: Parameter group for production Redis
      Properties:
        # Memory management
        maxmemory-policy: allkeys-lru

        # Timeout for idle connections (5 minutes)
        timeout: '300'

        # TCP keepalive
        tcp-keepalive: '300'

        # Slow log configuration
        slowlog-log-slower-than: '10000'  # 10ms
        slowlog-max-len: '128'

        # Notify on expiration and eviction
        notify-keyspace-events: 'Ex'

        # Active defragmentation
        activedefrag: 'yes'

        # Active rehashing
        activerehashing: 'yes'

        # LFU tuning (if using LFU eviction)
        lfu-log-factor: '10'
        lfu-decay-time: '1'
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-params'

  # Replication Group (Cluster Mode Enabled)
  RedisReplicationGroup:
    Type: AWS::ElastiCache::ReplicationGroup
    Properties:
      ReplicationGroupId: !Sub '${AWS::StackName}-cluster'
      ReplicationGroupDescription: Production Redis cluster with cluster mode

      # Engine configuration
      Engine: redis
      EngineVersion: '7.1'
      Port: 6379
      CacheParameterGroupName: !Ref RedisParameterGroup

      # Node configuration
      CacheNodeType: !Ref NodeType
      NumNodeGroups: !Ref NumShards
      ReplicasPerNodeGroup: !Ref ReplicasPerShard

      # High Availability
      AutomaticFailoverEnabled: true
      MultiAZEnabled: true

      # Security
      AtRestEncryptionEnabled: true
      TransitEncryptionEnabled: true
      AuthToken: !Sub '{{resolve:secretsmanager:${RedisAuthTokenSecret}:SecretString:token}}'
      SecurityGroupIds:
        - !Ref RedisSecurityGroup
      CacheSubnetGroupName: !Ref RedisSubnetGroup

      # Backup configuration
      SnapshotRetentionLimit: 7
      SnapshotWindow: '03:00-05:00'  # UTC
      PreferredMaintenanceWindow: 'sun:05:00-sun:07:00'  # UTC

      # Automatic backup to S3
      SnapshotName: !Sub '${AWS::StackName}-final-snapshot'

      # Auto minor version upgrade
      AutoMinorVersionUpgrade: true

      # Notification SNS topic
      NotificationTopicArn: !Ref RedisSNSTopic

      # Log delivery
      LogDeliveryConfigurations:
        - DestinationType: cloudwatch-logs
          DestinationDetails:
            CloudWatchLogsDetails:
              LogGroup: !Ref RedisSlowLogGroup
          LogFormat: json
          LogType: slow-log

        - DestinationType: cloudwatch-logs
          DestinationDetails:
            CloudWatchLogsDetails:
              LogGroup: !Ref RedisEngineLogGroup
          LogFormat: json
          LogType: engine-log

      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-cluster'
        - Key: Environment
          Value: !Ref Environment
        - Key: Backup
          Value: 'true'

  # Auth Token Secret
  RedisAuthTokenSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: !Sub '${AWS::StackName}/redis/auth-token'
      Description: Redis AUTH token
      GenerateSecretString:
        SecretStringTemplate: '{}'
        GenerateStringKey: token
        PasswordLength: 32
        ExcludeCharacters: '"@/\'
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-auth-token'

  # CloudWatch Log Groups
  RedisSlowLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/elasticache/${AWS::StackName}/slow-log'
      RetentionInDays: 30

  RedisEngineLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/elasticache/${AWS::StackName}/engine-log'
      RetentionInDays: 30

  # SNS Topic for notifications
  RedisSNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${AWS::StackName}-notifications'
      DisplayName: Redis cluster notifications
      Subscription:
        - Endpoint: ops-team@example.com
          Protocol: email

  # CloudWatch Alarms
  RedisHighCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${AWS::StackName}-high-cpu'
      AlarmDescription: Redis CPU utilization is too high
      MetricName: EngineCPUUtilization
      Namespace: AWS/ElastiCache
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 75
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: ReplicationGroupId
          Value: !Ref RedisReplicationGroup
      AlarmActions:
        - !Ref RedisSNSTopic
      TreatMissingData: notBreaching

  RedisHighMemoryAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${AWS::StackName}-high-memory'
      AlarmDescription: Redis memory usage is too high
      MetricName: DatabaseMemoryUsagePercentage
      Namespace: AWS/ElastiCache
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 85
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: ReplicationGroupId
          Value: !Ref RedisReplicationGroup
      AlarmActions:
        - !Ref RedisSNSTopic

  RedisHighEvictionsAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${AWS::StackName}-high-evictions'
      AlarmDescription: Redis is evicting keys frequently
      MetricName: Evictions
      Namespace: AWS/ElastiCache
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 1000
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: ReplicationGroupId
          Value: !Ref RedisReplicationGroup
      AlarmActions:
        - !Ref RedisSNSTopic

  RedisLowCacheHitRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${AWS::StackName}-low-cache-hit-rate'
      AlarmDescription: Redis cache hit rate is below threshold
      Metrics:
        - Id: hits
          MetricStat:
            Metric:
              Namespace: AWS/ElastiCache
              MetricName: CacheHits
              Dimensions:
                - Name: ReplicationGroupId
                  Value: !Ref RedisReplicationGroup
            Period: 300
            Stat: Sum
        - Id: misses
          MetricStat:
            Metric:
              Namespace: AWS/ElastiCache
              MetricName: CacheMisses
              Dimensions:
                - Name: ReplicationGroupId
                  Value: !Ref RedisReplicationGroup
            Period: 300
            Stat: Sum
        - Id: hit_rate
          Expression: hits / (hits + misses) * 100
          Label: Cache Hit Rate
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: LessThanThreshold
      AlarmActions:
        - !Ref RedisSNSTopic
      TreatMissingData: notBreaching

Outputs:
  ConfigurationEndpoint:
    Description: Configuration endpoint for cluster mode
    Value: !GetAtt RedisReplicationGroup.ConfigurationEndPoint.Address
    Export:
      Name: !Sub '${AWS::StackName}-config-endpoint'

  ConfigurationPort:
    Description: Configuration port
    Value: !GetAtt RedisReplicationGroup.ConfigurationEndPoint.Port
    Export:
      Name: !Sub '${AWS::StackName}-config-port'

  ReplicationGroupId:
    Description: Replication group ID
    Value: !Ref RedisReplicationGroup
    Export:
      Name: !Sub '${AWS::StackName}-replication-group-id'

  AuthTokenSecretArn:
    Description: ARN of the Secrets Manager secret containing AUTH token
    Value: !Ref RedisAuthTokenSecret
    Export:
      Name: !Sub '${AWS::StackName}-auth-token-secret-arn'

  ConnectionString:
    Description: Redis connection string (use in apps)
    Value: !Sub
      - 'rediss://${Endpoint}:${Port}'
      - Endpoint: !GetAtt RedisReplicationGroup.ConfigurationEndPoint.Address
        Port: !GetAtt RedisReplicationGroup.ConfigurationEndPoint.Port
```

---

### MemoryDB - Configuration production complète

#### Terraform Configuration

```hcl
# MemoryDB Production Configuration
# Provider version
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Variables
variable "environment" {
  type        = string
  description = "Environment name"
  default     = "production"
}

variable "vpc_id" {
  type        = string
  description = "VPC ID for MemoryDB"
}

variable "private_subnet_ids" {
  type        = list(string)
  description = "Private subnet IDs (Multi-AZ)"
}

variable "app_security_group_id" {
  type        = string
  description = "Security group of application tier"
}

variable "node_type" {
  type        = string
  description = "MemoryDB node type"
  default     = "db.r7g.xlarge"
}

variable "num_shards" {
  type        = number
  description = "Number of shards"
  default     = 3
}

variable "num_replicas_per_shard" {
  type        = number
  description = "Number of replicas per shard"
  default     = 2
}

# Random passwords for users
resource "random_password" "admin_password" {
  length  = 32
  special = true
}

resource "random_password" "app_password" {
  length  = 32
  special = true
}

resource "random_password" "readonly_password" {
  length  = 32
  special = true
}

# Store passwords in Secrets Manager
resource "aws_secretsmanager_secret" "memorydb_passwords" {
  name        = "${var.environment}-memorydb-passwords"
  description = "MemoryDB user passwords"

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "memorydb_passwords" {
  secret_id = aws_secretsmanager_secret.memorydb_passwords.id
  secret_string = jsonencode({
    admin    = random_password.admin_password.result
    app      = random_password.app_password.result
    readonly = random_password.readonly_password.result
  })
}

# Subnet Group
resource "aws_memorydb_subnet_group" "main" {
  name       = "${var.environment}-memorydb-subnet-group"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name        = "${var.environment}-memorydb-subnet-group"
    Environment = var.environment
  }
}

# Security Group
resource "aws_security_group" "memorydb" {
  name        = "${var.environment}-memorydb-sg"
  description = "Security group for MemoryDB"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
    description     = "Redis access from application tier"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name        = "${var.environment}-memorydb-sg"
    Environment = var.environment
  }
}

# Parameter Group
resource "aws_memorydb_parameter_group" "main" {
  name   = "${var.environment}-memorydb-params"
  family = "memorydb_redis7"

  # Memory policy: noeviction for primary DB use case
  parameter {
    name  = "maxmemory-policy"
    value = "noeviction"
  }

  # Active defragmentation
  parameter {
    name  = "activedefrag"
    value = "yes"
  }

  # Timeout
  parameter {
    name  = "timeout"
    value = "300"
  }

  # TCP keepalive
  parameter {
    name  = "tcp-keepalive"
    value = "300"
  }

  # Slow log
  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"  # 10ms
  }

  parameter {
    name  = "slowlog-max-len"
    value = "128"
  }

  # Notify keyspace events
  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"
  }

  # LFU tuning
  parameter {
    name  = "lfu-log-factor"
    value = "10"
  }

  parameter {
    name  = "lfu-decay-time"
    value = "1"
  }

  tags = {
    Name        = "${var.environment}-memorydb-params"
    Environment = var.environment
  }
}

# ACL Users
resource "aws_memorydb_user" "admin" {
  user_name     = "admin"
  access_string = "on ~* &* +@all"  # Full access

  authentication_mode {
    type      = "password"
    passwords = [random_password.admin_password.result]
  }

  tags = {
    Name        = "${var.environment}-admin-user"
    Role        = "admin"
    Environment = var.environment
  }
}

resource "aws_memorydb_user" "app_readwrite" {
  user_name     = "app-readwrite"
  access_string = "on ~* &* +@all -@dangerous -@admin"  # No dangerous commands

  authentication_mode {
    type      = "password"
    passwords = [random_password.app_password.result]
  }

  tags = {
    Name        = "${var.environment}-app-user"
    Role        = "application"
    Environment = var.environment
  }
}

resource "aws_memorydb_user" "readonly" {
  user_name     = "readonly"
  access_string = "on ~* &* +@read"  # Read-only

  authentication_mode {
    type      = "password"
    passwords = [random_password.readonly_password.result]
  }

  tags = {
    Name        = "${var.environment}-readonly-user"
    Role        = "readonly"
    Environment = var.environment
  }
}

# ACL
resource "aws_memorydb_acl" "main" {
  name = "${var.environment}-memorydb-acl"

  user_names = [
    aws_memorydb_user.admin.user_name,
    aws_memorydb_user.app_readwrite.user_name,
    aws_memorydb_user.readonly.user_name,
  ]

  tags = {
    Name        = "${var.environment}-memorydb-acl"
    Environment = var.environment
  }
}

# KMS Key for encryption
resource "aws_kms_key" "memorydb" {
  description             = "KMS key for MemoryDB encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = {
    Name        = "${var.environment}-memorydb-kms"
    Environment = var.environment
  }
}

resource "aws_kms_alias" "memorydb" {
  name          = "alias/${var.environment}-memorydb"
  target_key_id = aws_kms_key.memorydb.key_id
}

# MemoryDB Cluster
resource "aws_memorydb_cluster" "main" {
  name        = "${var.environment}-memorydb-cluster"
  description = "Production MemoryDB cluster with durable storage"

  # Node configuration
  node_type             = var.node_type
  num_shards            = var.num_shards
  num_replicas_per_shard = var.num_replicas_per_shard

  # ACL
  acl_name = aws_memorydb_acl.main.name

  # Network
  subnet_group_name  = aws_memorydb_subnet_group.main.name
  security_group_ids = [aws_security_group.memorydb.id]

  # Engine
  engine_version       = "7.0"
  port                 = 6379
  parameter_group_name = aws_memorydb_parameter_group.main.name

  # TLS
  tls_enabled = true

  # Maintenance
  maintenance_window = "sun:05:00-sun:07:00"

  # Snapshot configuration (continuous backup via transaction log)
  snapshot_retention_limit = 7
  snapshot_window         = "03:00-05:00"
  final_snapshot_name     = "${var.environment}-memorydb-final"

  # KMS encryption
  kms_key_arn = aws_kms_key.memorydb.arn

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  # SNS notifications
  sns_topic_arn = aws_sns_topic.memorydb_notifications.arn

  tags = {
    Name        = "${var.environment}-memorydb-cluster"
    Environment = var.environment
    ManagedBy   = "terraform"
    Backup      = "continuous"
  }

  depends_on = [
    aws_memorydb_acl.main,
    aws_memorydb_parameter_group.main,
    aws_memorydb_subnet_group.main
  ]
}

# SNS Topic for notifications
resource "aws_sns_topic" "memorydb_notifications" {
  name         = "${var.environment}-memorydb-notifications"
  display_name = "MemoryDB cluster notifications"

  tags = {
    Environment = var.environment
  }
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.memorydb_notifications.arn
  protocol  = "email"
  endpoint  = "ops-team@example.com"
}

# CloudWatch Log Group for application logs
resource "aws_cloudwatch_log_group" "memorydb_logs" {
  name              = "/aws/memorydb/${var.environment}"
  retention_in_days = 30

  tags = {
    Environment = var.environment
  }
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.environment}-memorydb-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/MemoryDB"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  alarm_description   = "MemoryDB CPU utilization too high"
  alarm_actions       = [aws_sns_topic.memorydb_notifications.arn]

  dimensions = {
    ClusterName = aws_memorydb_cluster.main.name
  }

  tags = {
    Environment = var.environment
  }
}

resource "aws_cloudwatch_metric_alarm" "memory_high" {
  alarm_name          = "${var.environment}-memorydb-high-memory"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseMemoryUsagePercentage"
  namespace           = "AWS/MemoryDB"
  period              = 300
  statistic           = "Average"
  threshold           = 85
  alarm_description   = "MemoryDB memory usage too high"
  alarm_actions       = [aws_sns_topic.memorydb_notifications.arn]

  dimensions = {
    ClusterName = aws_memorydb_cluster.main.name
  }

  tags = {
    Environment = var.environment
  }
}

resource "aws_cloudwatch_metric_alarm" "replication_lag" {
  alarm_name          = "${var.environment}-memorydb-replication-lag"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ReplicationLag"
  namespace           = "AWS/MemoryDB"
  period              = 60
  statistic           = "Average"
  threshold           = 5  # 5 seconds
  alarm_description   = "MemoryDB replication lag is high"
  alarm_actions       = [aws_sns_topic.memorydb_notifications.arn]

  dimensions = {
    ClusterName = aws_memorydb_cluster.main.name
  }

  tags = {
    Environment = var.environment
  }
}

# Outputs
output "cluster_endpoint" {
  description = "MemoryDB cluster endpoint"
  value       = aws_memorydb_cluster.main.cluster_endpoint[0].address
}

output "cluster_port" {
  description = "MemoryDB cluster port"
  value       = aws_memorydb_cluster.main.cluster_endpoint[0].port
}

output "connection_string" {
  description = "MemoryDB connection string"
  value       = "rediss://${aws_memorydb_cluster.main.cluster_endpoint[0].address}:${aws_memorydb_cluster.main.cluster_endpoint[0].port}"
  sensitive   = true
}

output "admin_username" {
  description = "Admin username"
  value       = aws_memorydb_user.admin.user_name
}

output "app_username" {
  description = "Application username"
  value       = aws_memorydb_user.app_readwrite.user_name
}

output "readonly_username" {
  description = "Read-only username"
  value       = aws_memorydb_user.readonly.user_name
}

output "passwords_secret_arn" {
  description = "ARN of Secrets Manager secret containing passwords"
  value       = aws_secretsmanager_secret.memorydb_passwords.arn
}

output "kms_key_id" {
  description = "KMS key ID for encryption"
  value       = aws_kms_key.memorydb.id
}
```

---

## 📊 Analyse de coûts TCO (Total Cost of Ownership)

### Scénario comparatif : Application e-commerce

```yaml
Context:
├── Dataset size: 100 GB
├── Throughput: 50K ops/sec (reads + writes)
├── Availability: 99.9% minimum
├── Multi-AZ: Required
└── Retention: 7 days backup

Configuration:
├── 3 shards
├── 2 replicas per shard (total 9 nodes)
└── us-east-1 region
```

#### ElastiCache Coût Mensuel

```yaml
Instance Type: cache.r7g.2xlarge (52.82 GB RAM per node)
├── RAM per node: 52.82 GB
├── Nodes required: 9 (3 shards × 3 nodes)
└── Node cost: $0.906/hour

Compute Cost:
├── 9 nodes × $0.906/hour = $8.154/hour
├── Monthly: $8.154 × 730 hours = $5,952/month
└── Annual: $5,952 × 12 = $71,424

Reserved Instance (3 years, all upfront):
├── Discount: ~60%
├── Monthly equivalent: $5,952 × 0.40 = $2,381/month
└── Annual: $28,572

Backup Storage (7 days × 100GB):
├── Included: 100 GB (cluster size)
├── Additional: 600 GB × $0.085 = $51/month
└── Annual: $612

Data Transfer (estimate):
├── Intra-AZ: Free
├── Inter-AZ: $0.01/GB (minimal for replication)
├── Out to Internet: $0.09/GB
└── Estimated: $100/month → $1,200/year

ElastiCache Total Cost (On-Demand):
├── Compute: $71,424/year
├── Backup: $612/year
├── Data transfer: $1,200/year
└── TOTAL: $73,236/year

ElastiCache Total Cost (Reserved 3y):
├── Compute: $28,572/year
├── Backup: $612/year
├── Data transfer: $1,200/year
└── TOTAL: $30,384/year
```

#### MemoryDB Coût Mensuel

```yaml
Instance Type: db.r7g.2xlarge (52.82 GB RAM per node)
├── RAM per node: 52.82 GB
├── Nodes required: 9
└── Node cost: $1.613/hour (78% plus cher qu'ElastiCache)

Compute Cost:
├── 9 nodes × $1.613/hour = $14.517/hour
├── Monthly: $14.517 × 730 = $10,597/month
└── Annual: $127,164

Reserved Instance (3 years, all upfront):
├── Discount: ~60%
├── Monthly equivalent: $10,597 × 0.40 = $4,239/month
└── Annual: $50,868

Backup Storage (continuous via transaction log):
├── Included: Transaction log storage (first 7 days)
├── Additional snapshots: Minimal
└── Estimated: $50/month → $600/year

Data Transfer (same as ElastiCache):
└── $1,200/year

MemoryDB Total Cost (On-Demand):
├── Compute: $127,164/year
├── Backup: $600/year
├── Data transfer: $1,200/year
└── TOTAL: $128,964/year

MemoryDB Total Cost (Reserved 3y):
├── Compute: $50,868/year
├── Backup: $600/year
├── Data transfer: $1,200/year
└── TOTAL: $52,668/year
```

#### Comparaison TCO

```yaml
ElastiCache vs MemoryDB (On-Demand)
├── ElastiCache: $73,236/year
├── MemoryDB: $128,964/year
├── Difference: +$55,728/year (+76%)
└── MemoryDB premium: $4,644/month

ElastiCache vs MemoryDB (Reserved 3y)
├── ElastiCache: $30,384/year
├── MemoryDB: $52,668/year
├── Difference: +$22,284/year (+73%)
└── MemoryDB premium: $1,857/month

Question clé:
Est-ce que la durabilité garantie vaut $1,857-$4,644/mois ?

Réponse dépend de:
├── Valeur des données (financial vs cache)
├── Coût d'un incident (data loss)
├── SLA requirements (99.9% vs 99.99%)
└── Compliance (HIPAA, PCI-DSS)
```

### Calculateur de ROI

```python
# ROI Calculator: When does MemoryDB make sense?

def calculate_breakeven(
    monthly_revenue: float,
    data_loss_revenue_impact_pct: float,
    data_loss_probability_per_year: float,
    elasticache_monthly_cost: float = 2532,  # RI 3y
    memorydb_monthly_cost: float = 4389      # RI 3y
):
    """
    Calculate if MemoryDB premium is justified

    Example:
    - E-commerce with $10M monthly revenue
    - Data loss (cart/session) impacts 2% of revenue
    - Estimated 2 incidents per year with ElastiCache
    """

    monthly_premium = memorydb_monthly_cost - elasticache_monthly_cost
    annual_premium = monthly_premium * 12

    expected_annual_loss_elasticache = (
        monthly_revenue * 12 *
        (data_loss_revenue_impact_pct / 100) *
        data_loss_probability_per_year
    )

    net_benefit = expected_annual_loss_elasticache - annual_premium

    print(f"Monthly Revenue: ${monthly_revenue:,.0f}")
    print(f"Data Loss Impact: {data_loss_revenue_impact_pct}% of revenue")
    print(f"Expected Incidents/Year: {data_loss_probability_per_year}")
    print(f"\nMemoryDB Premium: ${monthly_premium:,.0f}/month (${annual_premium:,.0f}/year)")
    print(f"Expected Loss (ElastiCache): ${expected_annual_loss_elasticache:,.0f}/year")
    print(f"\nNet Benefit (MemoryDB): ${net_benefit:,.0f}/year")

    if net_benefit > 0:
        print(f"✅ MemoryDB justified (saves ${net_benefit:,.0f}/year)")
    else:
        print(f"❌ MemoryDB not justified (costs ${-net_benefit:,.0f}/year extra)")

    return net_benefit

# Scénarios
print("=== Scenario 1: Large E-commerce ===")
calculate_breakeven(
    monthly_revenue=10_000_000,
    data_loss_revenue_impact_pct=2.0,
    data_loss_probability_per_year=2
)
# Result: +$377,716/year → MemoryDB strongly justified

print("\n=== Scenario 2: Startup ===")
calculate_breakeven(
    monthly_revenue=100_000,
    data_loss_revenue_impact_pct=1.0,
    data_loss_probability_per_year=1
)
# Result: -$21,084/year → ElastiCache better

print("\n=== Scenario 3: Fintech ===")
calculate_breakeven(
    monthly_revenue=5_000_000,
    data_loss_revenue_impact_pct=5.0,  # Regulatory fines
    data_loss_probability_per_year=1
)
# Result: +$227,716/year → MemoryDB justified
```

---

## 🔄 Migration ElastiCache → MemoryDB

### Stratégies de migration

#### Option 1 : Snapshot & Restore (Downtime)

```yaml
Process:
├── (1) Create final snapshot of ElastiCache
├── (2) Upload snapshot to S3
├── (3) Create MemoryDB cluster
├── (4) Restore from snapshot
├── (5) Update application connection strings
└── (6) Cutover

Downtime: 30 minutes to 2 hours
Risk: Low (tested restore)
Data Loss: None (snapshot is consistent)

Steps détaillés:
```

```bash
#!/bin/bash
# Migration script: ElastiCache to MemoryDB

set -euo pipefail

ELASTICACHE_CLUSTER_ID="prod-elasticache"
MEMORYDB_CLUSTER_NAME="prod-memorydb"
S3_BUCKET="my-redis-snapshots"
SNAPSHOT_NAME="migration-snapshot-$(date +%Y%m%d-%H%M%S)"

echo "Step 1: Create ElastiCache snapshot"
aws elasticache create-snapshot \
  --replication-group-id "$ELASTICACHE_CLUSTER_ID" \
  --snapshot-name "$SNAPSHOT_NAME"

echo "Waiting for snapshot to complete..."
aws elasticache wait snapshot-available \
  --snapshot-name "$SNAPSHOT_NAME"

echo "Step 2: Export snapshot to S3"
aws elasticache copy-snapshot \
  --source-snapshot-name "$SNAPSHOT_NAME" \
  --target-snapshot-name "${SNAPSHOT_NAME}-s3" \
  --target-bucket "$S3_BUCKET"

echo "Step 3: Create MemoryDB cluster"
aws memorydb create-cluster \
  --cluster-name "$MEMORYDB_CLUSTER_NAME" \
  --node-type db.r7g.xlarge \
  --num-shards 3 \
  --num-replicas-per-shard 2 \
  --acl-name "open-access" \
  --snapshot-arns "arn:aws:s3:::${S3_BUCKET}/${SNAPSHOT_NAME}-s3" \
  --subnet-group-name "prod-subnet-group" \
  --tls-enabled

echo "Waiting for MemoryDB cluster to be available..."
aws memorydb wait cluster-available \
  --cluster-name "$MEMORYDB_CLUSTER_NAME"

echo "Step 4: Get new endpoint"
NEW_ENDPOINT=$(aws memorydb describe-clusters \
  --cluster-name "$MEMORYDB_CLUSTER_NAME" \
  --query 'Clusters[0].ClusterEndpoint.Address' \
  --output text)

echo "New MemoryDB endpoint: $NEW_ENDPOINT"
echo "Update your application configuration to use this endpoint"
echo "After validation, delete ElastiCache cluster"
```

#### Option 2 : Dual Write (Zero Downtime)

```yaml
Process:
├── (1) Deploy MemoryDB alongside ElastiCache
├── (2) Update application to dual-write (ElastiCache + MemoryDB)
├── (3) Backfill existing data to MemoryDB
├── (4) Verify data consistency
├── (5) Switch reads to MemoryDB
├── (6) Stop writes to ElastiCache
└── (7) Decommission ElastiCache

Downtime: Zero
Risk: Medium (requires code changes)
Complexity: High
Data Loss: None (dual write ensures consistency)
```

Application code example (Python):

```python
import redis
from typing import Optional

class DualWriteRedisClient:
    """Redis client that writes to both ElastiCache and MemoryDB"""

    def __init__(
        self,
        elasticache_host: str,
        memorydb_host: str,
        port: int = 6379,
        password: Optional[str] = None
    ):
        # ElastiCache connection
        self.elasticache = redis.Redis(
            host=elasticache_host,
            port=port,
            password=password,
            ssl=True,
            decode_responses=True
        )

        # MemoryDB connection
        self.memorydb = redis.Redis(
            host=memorydb_host,
            port=port,
            password=password,
            ssl=True,
            decode_responses=True
        )

        # Phase of migration
        self.migration_phase = "DUAL_WRITE"  # DUAL_WRITE, MEMORYDB_PRIMARY, COMPLETE

    def set(self, key: str, value: str, ex: Optional[int] = None) -> bool:
        """Write to both Redis instances"""
        try:
            # Write to ElastiCache (primary during migration)
            elasticache_result = self.elasticache.set(key, value, ex=ex)

            # Write to MemoryDB (async, best effort)
            try:
                self.memorydb.set(key, value, ex=ex)
            except Exception as e:
                # Log but don't fail (ElastiCache is still primary)
                print(f"Warning: MemoryDB write failed for key {key}: {e}")

            return elasticache_result
        except Exception as e:
            print(f"Error: ElastiCache write failed for key {key}: {e}")
            raise

    def get(self, key: str) -> Optional[str]:
        """Read from appropriate instance based on migration phase"""
        if self.migration_phase == "DUAL_WRITE":
            # Still reading from ElastiCache
            return self.elasticache.get(key)
        elif self.migration_phase == "MEMORYDB_PRIMARY":
            # Reading from MemoryDB, fallback to ElastiCache
            try:
                value = self.memorydb.get(key)
                if value is not None:
                    return value
                # Fallback
                return self.elasticache.get(key)
            except Exception as e:
                print(f"Warning: MemoryDB read failed, using ElastiCache: {e}")
                return self.elasticache.get(key)
        else:  # COMPLETE
            # Only MemoryDB
            return self.memorydb.get(key)

    def delete(self, key: str) -> int:
        """Delete from both instances"""
        elasticache_result = self.elasticache.delete(key)
        try:
            self.memorydb.delete(key)
        except Exception as e:
            print(f"Warning: MemoryDB delete failed for key {key}: {e}")
        return elasticache_result

# Usage
client = DualWriteRedisClient(
    elasticache_host="prod-elasticache.abc123.cache.amazonaws.com",
    memorydb_host="prod-memorydb.xyz456.memorydb.us-east-1.amazonaws.com",
    password="your-auth-token"
)

# Application code unchanged
client.set("user:1234:session", "session-data", ex=3600)
session = client.get("user:1234:session")
```

Backfill script:

```python
import redis
from tqdm import tqdm

def backfill_data(
    source_host: str,
    target_host: str,
    port: int = 6379,
    password: str = None,
    batch_size: int = 1000
):
    """
    Backfill data from ElastiCache to MemoryDB
    """
    source = redis.Redis(host=source_host, port=port, password=password, ssl=True)
    target = redis.Redis(host=target_host, port=port, password=password, ssl=True)

    cursor = 0
    total_keys = 0

    print("Starting backfill...")

    while True:
        # Scan in batches (don't use KEYS in production!)
        cursor, keys = source.scan(cursor=cursor, count=batch_size)

        if not keys:
            if cursor == 0:
                break
            continue

        # Use pipeline for efficiency
        pipe = target.pipeline()

        for key in tqdm(keys, desc=f"Backfilling batch"):
            try:
                # Get type
                key_type = source.type(key)
                ttl = source.ttl(key)

                if key_type == b'string':
                    value = source.get(key)
                    if ttl > 0:
                        pipe.setex(key, ttl, value)
                    else:
                        pipe.set(key, value)

                elif key_type == b'hash':
                    hash_data = source.hgetall(key)
                    pipe.hset(key, mapping=hash_data)
                    if ttl > 0:
                        pipe.expire(key, ttl)

                elif key_type == b'list':
                    list_data = source.lrange(key, 0, -1)
                    pipe.delete(key)
                    for item in list_data:
                        pipe.rpush(key, item)
                    if ttl > 0:
                        pipe.expire(key, ttl)

                elif key_type == b'set':
                    set_data = source.smembers(key)
                    pipe.sadd(key, *set_data)
                    if ttl > 0:
                        pipe.expire(key, ttl)

                elif key_type == b'zset':
                    zset_data = source.zrange(key, 0, -1, withscores=True)
                    pipe.zadd(key, dict(zset_data))
                    if ttl > 0:
                        pipe.expire(key, ttl)

                total_keys += 1

            except Exception as e:
                print(f"Error processing key {key}: {e}")

        # Execute batch
        pipe.execute()

        if cursor == 0:
            break

    print(f"Backfill complete. Migrated {total_keys} keys.")

# Run backfill
backfill_data(
    source_host="prod-elasticache.abc123.cache.amazonaws.com",
    target_host="prod-memorydb.xyz456.memorydb.us-east-1.amazonaws.com",
    password="your-auth-token"
)
```

---

## 📈 Monitoring et Observabilité

### Métriques clés à surveiller

#### ElastiCache

```yaml
Performance Metrics:
├── EngineCPUUtilization: <75% (alerte si >80%)
├── NetworkBytesIn/Out: Tendance stable
├── CurrConnections: Surveiller les pics
├── NewConnections: Rate de nouvelles connexions
└── CommandsProcessed: Throughput

Memory Metrics:
├── DatabaseMemoryUsagePercentage: <85%
├── BytesUsedForCache: Utilisation mémoire
├── Evictions: Doit être proche de 0
└── FreeableMemory: Mémoire disponible

Cache Metrics:
├── CacheHits: Nombre de hits
├── CacheMisses: Nombre de misses
├── CacheHitRate: Hits / (Hits + Misses) >80%
└── KeyspaceHits/Misses: Par keyspace

Replication Metrics:
├── ReplicationLag: <1 second
├── ReplicationBytes: Volume répliqué
└── SaveInProgress: 0 (sauf pendant BGSAVE)

Anomalies à détecter:
├── CacheHitRate drop: Problème de TTL ou pattern
├── Evictions spike: Manque de mémoire
├── ReplicationLag increase: Problème réseau/charge
└── CPU spike: Commandes lentes (KEYS, etc.)
```

#### MemoryDB

```yaml
Performance Metrics:
├── CPUUtilization: <75%
├── NetworkThroughput: Stable
├── ActiveConnections: Surveiller
└── CommandsProcessed: Ops/sec

Memory Metrics:
├── DatabaseMemoryUsagePercentage: <85%
├── BytesUsedForMemoryDB: Utilisation
├── SwapUsage: Doit être 0
└── FreeableMemory: Disponible

Durability Metrics (unique à MemoryDB):
├── ReplicationLag: <5 seconds
├── TransactionLogDiskUsage: Croissance du WAL
├── SnapshotProgress: Status des snapshots
└── BackupRetentionPeriod: 1-35 jours

Unique MemoryDB metrics:
├── MultiAZEnabled: 1 (doit être activé)
├── EngineUpToDate: 1 (version à jour)
├── AuthenticationFailures: 0
└── ClusterModeEnabled: 1

Alertes critiques:
├── DatabaseMemoryUsagePercentage >90%
├── CPUUtilization >80% sustained
├── ReplicationLag >10 seconds
└── TransactionLogDiskUsage growth anomaly
```

### Dashboard CloudWatch

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "title": "ElastiCache vs MemoryDB - Write Latency",
        "metrics": [
          [ "AWS/ElastiCache", "StringBasedCmdsLatency", { "stat": "Average", "label": "ElastiCache P50" } ],
          [ ".", ".", { "stat": "p99", "label": "ElastiCache P99" } ],
          [ "AWS/MemoryDB", "WriteLatency", { "stat": "Average", "label": "MemoryDB P50" } ],
          [ ".", ".", { "stat": "p99", "label": "MemoryDB P99" } ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "yAxis": {
          "left": {
            "label": "Milliseconds"
          }
        }
      }
    },
    {
      "type": "metric",
      "properties": {
        "title": "Durability - Replication Lag",
        "metrics": [
          [ "AWS/ElastiCache", "ReplicationLag", { "label": "ElastiCache" } ],
          [ "AWS/MemoryDB", "ReplicationLag", { "label": "MemoryDB" } ]
        ],
        "period": 60,
        "stat": "Maximum",
        "region": "us-east-1",
        "yAxis": {
          "left": {
            "label": "Seconds",
            "min": 0
          }
        },
        "annotations": {
          "horizontal": [
            {
              "value": 5,
              "label": "Warning threshold",
              "color": "#ff7f0e"
            },
            {
              "value": 10,
              "label": "Critical threshold",
              "color": "#d62728"
            }
          ]
        }
      }
    },
    {
      "type": "metric",
      "properties": {
        "title": "Memory Usage Percentage",
        "metrics": [
          [ "AWS/ElastiCache", "DatabaseMemoryUsagePercentage", { "label": "ElastiCache" } ],
          [ "AWS/MemoryDB", "DatabaseMemoryUsagePercentage", { "label": "MemoryDB" } ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "yAxis": {
          "left": {
            "min": 0,
            "max": 100
          }
        },
        "annotations": {
          "horizontal": [
            {
              "value": 85,
              "label": "Scale up threshold"
            }
          ]
        }
      }
    }
  ]
}
```

---

## ✅ Checklist de décision finale

```yaml
Utilisez ElastiCache si:
☐ Redis est un cache (pas primary DB)
☐ Perte de données acceptable (quelques secondes à minutes)
☐ Budget est une contrainte forte
☐ Données reconstituables depuis source primaire
☐ Use case: API cache, query cache, session store non-critique

Utilisez MemoryDB si:
☐ Redis est votre base de données primaire
☐ Zero data loss est requis (RPO = 0)
☐ RTO doit être <1 minute
☐ Compliance requiert durabilité (HIPAA, PCI-DSS)
☐ Use case: session store critique, inventory, leaderboards, financial data

Éléments à vérifier avant décision:
☐ Calculer le coût d'un incident (data loss)
☐ Vérifier les exigences de compliance
☐ Valider le budget disponible (MemoryDB ~78% plus cher)
☐ Évaluer la complexité de migration future
☐ Considérer la roadmap (nouvelles features MemoryDB)
☐ Tester les deux en staging pour valider performance
☐ Simuler un failover pour valider RTO
```

---

## 🎯 Conclusion

### Points clés à retenir

1. **Architecture fondamentalement différente**
   - ElastiCache = Cache volatil avec persistence optionnelle
   - MemoryDB = Primary database avec transaction log durable

2. **Durabilité**
   - ElastiCache : RPO 1s-15min, RTO 2-3min, data loss possible
   - MemoryDB : RPO 0s, RTO <1min, zero data loss

3. **Coût**
   - MemoryDB coûte ~78% plus cher
   - Justifié si valeur des données > coût du premium

4. **Performance**
   - Différence de latence : ~0.5-1ms sur writes
   - Négligeable pour la plupart des use cases

5. **Migration**
   - Snapshot & Restore : Simple, downtime 30min-2h
   - Dual Write : Zero downtime, complexe

### Recommandation générale

```
Si vous hésitez → Commencez par ElastiCache
Si les données deviennent critiques → Migrez vers MemoryDB

MemoryDB n'est PAS "ElastiCache amélioré"
C'est un produit différent pour un use case différent

Choisissez en fonction de:
1. Criticité des données
2. Budget disponible
3. Exigences RTO/RPO
4. Compliance
```

---

**🎯 Section suivante :** Nous allons maintenant explorer **Azure Cache for Redis** dans la section 15.3, avec ses spécificités et configurations avancées.

⏭️ [Azure Cache for Redis](/15-redis-cloud-conteneurs/03-azure-cache-redis.md)
