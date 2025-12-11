🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.7 - Dimensionnement et planification de capacité

## Introduction

Le dimensionnement correct de Redis est crucial pour :

- ⚡ **Performance optimale** - Éviter throttling CPU/RAM/I/O
- 💰 **Coûts maîtrisés** - Ne pas sur-dimensionner inutilement
- 🛡️ **Stabilité** - Éviter OOM, évictions non désirées
- 📈 **Scalabilité** - Planifier la croissance future

> **⚠️ Erreurs courantes :** Sous-dimensionner la RAM (évictions massives), sous-estimer l'overhead réseau, ne pas prévoir la croissance.

---

## Composants à dimensionner

```
┌─────────────────────────────────────────────────────────────────┐
│              DIMENSIONS CRITIQUES REDIS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. MÉMOIRE (RAM) - CRITIQUE                                    │
│     ├── Données utilisateur                                     │
│     ├── Overhead Redis (pointers, structures)                   │
│     ├── Fragmentation mémoire                                   │
│     ├── Réplication (COW lors BGSAVE)                           │
│     └── Marge de croissance                                     │
│                                                                 │
│  2. CPU                                                         │
│     ├── Single-threaded (core unique pour commandes)            │
│     ├── I/O threads (Redis 6+)                                  │
│     ├── BGSAVE/AOF rewrite                                      │
│     └── TLS overhead                                            │
│                                                                 │
│  3. RÉSEAU                                                      │
│     ├── Bande passante                                          │
│     ├── Packets per second (PPS)                                │
│     ├── Latency réseau                                          │
│     └── Connexions concurrentes                                 │
│                                                                 │
│  4. DISQUE (I/O)                                                │
│     ├── Persistance RDB/AOF                                     │
│     ├── IOPS (read/write)                                       │
│     ├── Espace disque (backups)                                 │
│     └── Latency disque                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Dimensionnement Mémoire (RAM)

### 1. Formule de calcul mémoire

```
RAM_TOTALE_NÉCESSAIRE =
    DONNÉES_BRUTES
    × (1 + OVERHEAD_REDIS)
    × (1 + FRAGMENTATION)
    × (1 + COW_BGSAVE)
    + MARGE_CROISSANCE
    + OS_OVERHEAD

Où:
├── DONNÉES_BRUTES : Taille des données stockées
├── OVERHEAD_REDIS : 20-50% (structures internes)
├── FRAGMENTATION : 10-50% (dépend du pattern d'usage)
├── COW_BGSAVE : 10-100% (si persistance active)
├── MARGE_CROISSANCE : 30-50% (anticiper croissance)
└── OS_OVERHEAD : 500MB-1GB (système d'exploitation)
```

### 2. Calcul overhead Redis par structure

```
┌─────────────────────────────────────────────────────────────────┐
│           OVERHEAD MÉMOIRE PAR TYPE DE DONNÉE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  String (SET):                                                  │
│  ├── Clé: ~96 bytes + longueur clé                              │
│  ├── Valeur: ~56 bytes + longueur valeur                        │
│  └── Total overhead: ~152 bytes                                 │
│                                                                 │
│  Hash (HSET):                                                   │
│  ├── Hash object: ~96 bytes                                     │
│  ├── Par field: ~64 bytes + longueur field + valeur             │
│  ├── Encoding: listpack si < 512 fields (économique)            │
│  └── Overhead: 25-40% selon taille                              │
│                                                                 │
│  List:                                                          │
│  ├── List object: ~96 bytes                                     │
│  ├── Encoding: listpack ou linkedlist                           │
│  ├── Overhead listpack: ~5-10%                                  │
│  └── Overhead linkedlist: ~40-60%                               │
│                                                                 │
│  Set:                                                           │
│  ├── Set object: ~96 bytes                                      │
│  ├── Intset (si petits integers): ~10%                          │
│  └── Hash table: ~50-70%                                        │
│                                                                 │
│  Sorted Set:                                                    │
│  ├── Zset object: ~96 bytes                                     │
│  ├── Encoding: listpack ou skiplist+dict                        │
│  ├── Overhead listpack: ~15-20%                                 │
│  └── Overhead skiplist: ~60-100%                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Exemples de calcul mémoire

#### Exemple 1 : Session Store

```
Scénario:
├── 10 millions de sessions actives
├── Taille moyenne session: 2 KB (JSON)
├── Structure: Hash (HSET)
├── TTL: 24 heures
└── Persistance: AOF everysec

Calcul:
1. Données brutes
   = 10,000,000 sessions × 2 KB
   = 20 GB

2. Overhead Redis (30% pour Hash)
   = 20 GB × 0.30
   = 6 GB

3. Fragmentation (20% typique)
   = (20 + 6) GB × 0.20
   = 5.2 GB

4. COW BGSAVE (50% du dataset)
   = 26 GB × 0.50
   = 13 GB

5. Total avant marge
   = 20 + 6 + 5.2 + 13
   = 44.2 GB

6. Marge de croissance (30%)
   = 44.2 × 0.30
   = 13.3 GB

7. OS overhead
   = 1 GB

RAM TOTALE NÉCESSAIRE = 44.2 + 13.3 + 1 = 58.5 GB
Recommandation: Instance 64 GB RAM

Configuration maxmemory:
├── RAM disponible: 64 GB
├── OS overhead: -1 GB
├── Marge sécurité: -5 GB
└── maxmemory: 58 GB (90% RAM dispo)
```

#### Exemple 2 : Cache applicatif

```
Scénario:
├── Cache de 5 millions de clés
├── Taille moyenne: 500 bytes (HTML fragments)
├── Structure: String (SET)
├── TTL: 1 heure (rotation rapide)
├── Persistance: Aucune
└── Politique éviction: allkeys-lru

Calcul:
1. Données brutes
   = 5,000,000 × 500 bytes
   = 2.5 GB

2. Overhead Redis (50% pour String)
   = 2.5 × 0.50
   = 1.25 GB

3. Fragmentation (25%)
   = 3.75 × 0.25
   = 0.94 GB

4. COW BGSAVE (pas de persistance)
   = 0 GB

5. Total avant marge
   = 2.5 + 1.25 + 0.94
   = 4.69 GB

6. Marge croissance (30%)
   = 4.69 × 0.30
   = 1.4 GB

7. OS overhead
   = 0.5 GB

RAM TOTALE = 4.69 + 1.4 + 0.5 = 6.6 GB
Recommandation: Instance 8 GB RAM

Configuration maxmemory: 7 GB
```

#### Exemple 3 : Analytics avec HyperLogLog

```
Scénario:
├── 1 million de compteurs HyperLogLog
├── Chaque HLL: ~12 KB
├── Comptage utilisateurs uniques par jour/page
├── Persistance: RDB quotidien
└── Rétention: 365 jours

Calcul:
1. Données brutes
   = 1,000,000 × 12 KB
   = 12 GB

2. Overhead Redis (20% pour HLL)
   = 12 × 0.20
   = 2.4 GB

3. Fragmentation (15% - HLL stable)
   = 14.4 × 0.15
   = 2.2 GB

4. COW BGSAVE (30% du dataset)
   = 14.4 × 0.30
   = 4.3 GB

5. Total avant marge
   = 12 + 2.4 + 2.2 + 4.3
   = 20.9 GB

6. Marge croissance (30%)
   = 20.9 × 0.30
   = 6.3 GB

7. OS overhead
   = 1 GB

RAM TOTALE = 20.9 + 6.3 + 1 = 28.2 GB
Recommandation: Instance 32 GB RAM

Configuration maxmemory: 28 GB
```

### 4. Script de calcul mémoire

```python
#!/usr/bin/env python3
# redis-memory-calculator.py

class RedisMemoryCalculator:
    """Calculateur de dimensionnement mémoire Redis"""

    # Overhead par type de structure (pourcentage)
    OVERHEAD = {
        'string': 0.50,  # 50%
        'hash': 0.30,    # 30%
        'list': 0.35,    # 35%
        'set': 0.60,     # 60%
        'zset': 0.80,    # 80%
        'hyperloglog': 0.20  # 20%
    }

    def __init__(self, num_keys, avg_size_bytes, structure_type='string'):
        self.num_keys = num_keys
        self.avg_size_bytes = avg_size_bytes
        self.structure_type = structure_type
        self.os_overhead_gb = 1.0

    def calculate_raw_data(self):
        """Calcule taille données brutes"""
        return (self.num_keys * self.avg_size_bytes) / (1024**3)  # GB

    def calculate_redis_overhead(self, raw_data_gb):
        """Calcule overhead Redis"""
        overhead_pct = self.OVERHEAD.get(self.structure_type, 0.40)
        return raw_data_gb * overhead_pct

    def calculate_fragmentation(self, current_gb, frag_ratio=0.20):
        """Calcule fragmentation mémoire"""
        return current_gb * frag_ratio

    def calculate_cow_overhead(self, current_gb, persistence=True, cow_ratio=0.50):
        """Calcule overhead Copy-on-Write"""
        if not persistence:
            return 0
        return current_gb * cow_ratio

    def calculate_growth_margin(self, current_gb, growth_pct=0.30):
        """Calcule marge de croissance"""
        return current_gb * growth_pct

    def calculate_total(self, persistence=True, fragmentation=0.20,
                       cow_ratio=0.50, growth=0.30):
        """Calcule RAM totale nécessaire"""

        # Étapes de calcul
        raw_data = self.calculate_raw_data()
        redis_overhead = self.calculate_redis_overhead(raw_data)
        subtotal = raw_data + redis_overhead

        fragmentation_mem = self.calculate_fragmentation(subtotal, fragmentation)
        subtotal += fragmentation_mem

        cow_overhead = self.calculate_cow_overhead(subtotal, persistence, cow_ratio)
        subtotal += cow_overhead

        growth_margin = self.calculate_growth_margin(subtotal, growth)

        total = subtotal + growth_margin + self.os_overhead_gb

        # Résultats détaillés
        results = {
            'raw_data_gb': round(raw_data, 2),
            'redis_overhead_gb': round(redis_overhead, 2),
            'fragmentation_gb': round(fragmentation_mem, 2),
            'cow_overhead_gb': round(cow_overhead, 2),
            'growth_margin_gb': round(growth_margin, 2),
            'os_overhead_gb': self.os_overhead_gb,
            'total_ram_gb': round(total, 2),
            'recommended_instance_gb': self._round_to_instance_size(total),
            'maxmemory_gb': round(total - self.os_overhead_gb - 1, 2)  # -1GB safety
        }

        return results

    def _round_to_instance_size(self, ram_gb):
        """Arrondit à une taille d'instance standard"""
        sizes = [2, 4, 8, 16, 32, 64, 128, 256, 384, 512]
        for size in sizes:
            if ram_gb <= size:
                return size
        return 512

    def print_report(self, results):
        """Affiche rapport détaillé"""
        print("=" * 60)
        print("REDIS MEMORY SIZING REPORT")
        print("=" * 60)
        print(f"\nConfiguration:")
        print(f"  Number of keys: {self.num_keys:,}")
        print(f"  Average size: {self.avg_size_bytes:,} bytes")
        print(f"  Structure type: {self.structure_type}")

        print(f"\nMemory Breakdown:")
        print(f"  Raw data:           {results['raw_data_gb']:>8.2f} GB")
        print(f"  Redis overhead:     {results['redis_overhead_gb']:>8.2f} GB")
        print(f"  Fragmentation:      {results['fragmentation_gb']:>8.2f} GB")
        print(f"  COW overhead:       {results['cow_overhead_gb']:>8.2f} GB")
        print(f"  Growth margin:      {results['growth_margin_gb']:>8.2f} GB")
        print(f"  OS overhead:        {results['os_overhead_gb']:>8.2f} GB")
        print(f"  " + "-" * 40)
        print(f"  TOTAL RAM NEEDED:   {results['total_ram_gb']:>8.2f} GB")

        print(f"\nRecommendations:")
        print(f"  Instance size:      {results['recommended_instance_gb']:>8} GB")
        print(f"  maxmemory setting:  {results['maxmemory_gb']:>8.2f} GB")
        print("=" * 60)


# Exemples d'utilisation
if __name__ == '__main__':
    print("\n### EXAMPLE 1: Session Store ###")
    calc1 = RedisMemoryCalculator(
        num_keys=10_000_000,
        avg_size_bytes=2048,
        structure_type='hash'
    )
    results1 = calc1.calculate_total(persistence=True, fragmentation=0.20, cow_ratio=0.50)
    calc1.print_report(results1)

    print("\n\n### EXAMPLE 2: Cache Layer ###")
    calc2 = RedisMemoryCalculator(
        num_keys=5_000_000,
        avg_size_bytes=500,
        structure_type='string'
    )
    results2 = calc2.calculate_total(persistence=False, fragmentation=0.25)
    calc2.print_report(results2)

    print("\n\n### EXAMPLE 3: Analytics (HyperLogLog) ###")
    calc3 = RedisMemoryCalculator(
        num_keys=1_000_000,
        avg_size_bytes=12288,
        structure_type='hyperloglog'
    )
    results3 = calc3.calculate_total(persistence=True, fragmentation=0.15, cow_ratio=0.30)
    calc3.print_report(results3)
```

---

## Dimensionnement CPU

### 1. Considérations CPU

```
┌─────────────────────────────────────────────────────────────────┐
│                    CPU POUR REDIS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis est SINGLE-THREADED pour les commandes                   │
│  ├── 1 core principal = traitement des commandes                │
│  ├── Performance limitée par single-core speed                  │
│  └── Préférer haute fréquence (3+ GHz) vs multi-core            │
│                                                                 │
│  Threads additionnels:                                          │
│  ├── I/O threads (Redis 6+): 2-4 threads recommandés            │
│  ├── BGSAVE fork: 1 thread                                      │
│  ├── AOF rewrite: 1 thread                                      │
│  └── Défragmentation active: 1 thread                           │
│                                                                 │
│  Overhead TLS:                                                  │
│  ├── Encryption/Decryption CPU intensive                        │
│  ├── +20-30% CPU avec TLS                                       │
│  └── I/O threads aident à distribuer la charge                  │
│                                                                 │
│  Recommandations:                                               │
│  ├── Minimum: 2 vCPU (1 pour Redis, 1 pour OS/background)       │
│  ├── Standard: 4 vCPU (confortable)                             │
│  ├── Haute charge: 8 vCPU (avec I/O threads)                    │
│  └── TLS activé: +25% vCPU                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Calcul charge CPU

```python
#!/usr/bin/env python3
# redis-cpu-calculator.py

def calculate_cpu_requirements(ops_per_sec, avg_command_time_us, tls_enabled=False, io_threads=4):
    """
    Calcule les besoins CPU pour Redis

    Args:
        ops_per_sec: Opérations par seconde
        avg_command_time_us: Temps moyen commande (microsecondes)
        tls_enabled: TLS activé (overhead 25%)
        io_threads: Nombre de I/O threads
    """

    # CPU utilisé par le main thread
    main_thread_usage = (ops_per_sec * avg_command_time_us) / 1_000_000  # Conversion en secondes
    main_thread_pct = (main_thread_usage / 1.0) * 100  # % d'utilisation d'1 core

    # Overhead TLS
    if tls_enabled:
        tls_overhead_pct = main_thread_pct * 0.25
    else:
        tls_overhead_pct = 0

    # I/O threads (distribués sur plusieurs cores)
    io_thread_usage_pct = main_thread_pct * 0.3 if io_threads > 1 else 0  # ~30% du main thread

    # Background tasks
    background_pct = 10  # BGSAVE, AOF, etc.

    # Total
    total_cpu_pct = main_thread_pct + tls_overhead_pct + io_thread_usage_pct + background_pct

    # Nombre de vCPU recommandés
    vcpu_needed = max(2, int((total_cpu_pct / 80) + 0.5))  # 80% utilisation max

    return {
        'ops_per_sec': ops_per_sec,
        'main_thread_pct': round(main_thread_pct, 1),
        'tls_overhead_pct': round(tls_overhead_pct, 1),
        'io_threads_pct': round(io_thread_usage_pct, 1),
        'background_pct': background_pct,
        'total_cpu_pct': round(total_cpu_pct, 1),
        'vcpu_recommended': vcpu_needed
    }

# Exemples
if __name__ == '__main__':
    print("=== CPU SIZING EXAMPLES ===\n")

    # Exemple 1: Cache à faible charge
    print("1. Low traffic cache:")
    result1 = calculate_cpu_requirements(
        ops_per_sec=10_000,
        avg_command_time_us=50,
        tls_enabled=False
    )
    print(f"   Ops/sec: {result1['ops_per_sec']:,}")
    print(f"   Main thread: {result1['main_thread_pct']}%")
    print(f"   Total CPU: {result1['total_cpu_pct']}%")
    print(f"   Recommended: {result1['vcpu_recommended']} vCPU\n")

    # Exemple 2: Session store haute charge
    print("2. High traffic session store:")
    result2 = calculate_cpu_requirements(
        ops_per_sec=50_000,
        avg_command_time_us=100,
        tls_enabled=True,
        io_threads=4
    )
    print(f"   Ops/sec: {result2['ops_per_sec']:,}")
    print(f"   Main thread: {result2['main_thread_pct']}%")
    print(f"   TLS overhead: {result2['tls_overhead_pct']}%")
    print(f"   Total CPU: {result2['total_cpu_pct']}%")
    print(f"   Recommended: {result2['vcpu_recommended']} vCPU\n")

    # Exemple 3: Très haute charge
    print("3. Very high traffic:")
    result3 = calculate_cpu_requirements(
        ops_per_sec=100_000,
        avg_command_time_us=80,
        tls_enabled=True,
        io_threads=4
    )
    print(f"   Ops/sec: {result3['ops_per_sec']:,}")
    print(f"   Main thread: {result3['main_thread_pct']}%")
    print(f"   TLS overhead: {result3['tls_overhead_pct']}%")
    print(f"   Total CPU: {result3['total_cpu_pct']}%")
    print(f"   Recommended: {result3['vcpu_recommended']} vCPU")
```

---

## Dimensionnement Réseau

### 1. Calcul bande passante

```python
#!/usr/bin/env python3
# redis-network-calculator.py

def calculate_network_requirements(ops_per_sec, avg_value_size_bytes,
                                  read_write_ratio=0.5):
    """
    Calcule les besoins réseau pour Redis

    Args:
        ops_per_sec: Opérations par seconde
        avg_value_size_bytes: Taille moyenne valeur (bytes)
        read_write_ratio: Ratio lecture/écriture (0.5 = 50% read)
    """

    # Overhead protocole Redis (~100 bytes par commande)
    protocol_overhead = 100

    # Taille moyenne par opération
    avg_op_size = avg_value_size_bytes + protocol_overhead

    # Bande passante nécessaire
    bytes_per_sec = ops_per_sec * avg_op_size

    # En Mbps
    mbits_per_sec = (bytes_per_sec * 8) / (1024 * 1024)

    # En Gbps
    gbits_per_sec = mbits_per_sec / 1024

    # Pics (3x la moyenne typiquement)
    peak_mbits_per_sec = mbits_per_sec * 3
    peak_gbits_per_sec = gbits_per_sec * 3

    # Recommandations
    if peak_gbits_per_sec < 1:
        recommended_link = "1 Gbps"
    elif peak_gbits_per_sec < 10:
        recommended_link = "10 Gbps"
    else:
        recommended_link = "25+ Gbps"

    return {
        'ops_per_sec': ops_per_sec,
        'avg_value_size_kb': round(avg_value_size_bytes / 1024, 2),
        'bandwidth_avg_mbps': round(mbits_per_sec, 2),
        'bandwidth_avg_gbps': round(gbits_per_sec, 3),
        'bandwidth_peak_mbps': round(peak_mbits_per_sec, 2),
        'bandwidth_peak_gbps': round(peak_gbits_per_sec, 3),
        'recommended_link': recommended_link
    }

# Exemples
if __name__ == '__main__':
    print("=== NETWORK SIZING EXAMPLES ===\n")

    # Exemple 1: Cache petites valeurs
    print("1. Cache with small values:")
    result1 = calculate_network_requirements(
        ops_per_sec=50_000,
        avg_value_size_bytes=500
    )
    for key, value in result1.items():
        print(f"   {key}: {value}")

    print("\n2. Session store medium values:")
    result2 = calculate_network_requirements(
        ops_per_sec=30_000,
        avg_value_size_bytes=2048
    )
    for key, value in result2.items():
        print(f"   {key}: {value}")

    print("\n3. High throughput large values:")
    result3 = calculate_network_requirements(
        ops_per_sec=100_000,
        avg_value_size_bytes=5120
    )
    for key, value in result3.items():
        print(f"   {key}: {value}")
```

---

## Dimensionnement Disque

### 1. Calcul espace disque

```
┌─────────────────────────────────────────────────────────────────┐
│                    ESPACE DISQUE REQUIS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RDB (Snapshot):                                                │
│  ├── Taille: ~Dataset size (compressé)                          │
│  ├── Fréquence: Selon config (ex: quotidien)                    │
│  └── Espace: Dataset × Rétention jours × 1.1                    │
│                                                                 │
│  AOF (Append Only File):                                        │
│  ├── Taille: 1.5-3x dataset size (non compressé)                │
│  ├── Rewrite: Ramène à ~dataset size                            │
│  └── Espace: Dataset × 3 (pendant rewrite)                      │
│                                                                 │
│  Logs:                                                          │
│  ├── Taille: Variable (100MB-1GB/jour)                          │
│  ├── Rétention: 7-30 jours                                      │
│  └── Espace: ~10-30 GB                                          │
│                                                                 │
│  Formule totale:                                                │
│  DISQUE = (RDB × Rétention) + (AOF × 3) + Logs + Marge          │
│         = Dataset × (Rétention + 3) + 30GB + 20% marge          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Calcul IOPS

```python
#!/usr/bin/env python3
# redis-disk-calculator.py

def calculate_disk_requirements(dataset_size_gb, rdb_enabled=True, aof_enabled=True,
                               rdb_retention_days=7, log_retention_days=30):
    """
    Calcule les besoins disque pour Redis

    Args:
        dataset_size_gb: Taille du dataset (GB)
        rdb_enabled: RDB activé
        aof_enabled: AOF activé
        rdb_retention_days: Rétention RDB (jours)
        log_retention_days: Rétention logs (jours)
    """

    # RDB backups
    rdb_space_gb = 0
    if rdb_enabled:
        rdb_space_gb = dataset_size_gb * rdb_retention_days * 1.1  # +10% overhead

    # AOF
    aof_space_gb = 0
    if aof_enabled:
        aof_space_gb = dataset_size_gb * 3  # Pendant rewrite

    # Logs
    log_space_gb = (log_retention_days / 30) * 10  # ~10GB par mois

    # Marge
    subtotal = rdb_space_gb + aof_space_gb + log_space_gb
    margin_gb = subtotal * 0.20  # 20% marge

    total_disk_gb = subtotal + margin_gb

    # IOPS requirements
    # RDB write: burst pendant BGSAVE
    # AOF write: continu selon ops/sec
    rdb_iops = 0
    if rdb_enabled:
        # Burst IOPS pendant snapshot (ex: 10GB en 60s = 170 MB/s = ~1300 IOPS @128KB)
        rdb_iops = int((dataset_size_gb * 1024 / 60) / 0.128)  # Simplified

    aof_iops = 0
    if aof_enabled:
        # AOF everysec: ~100-500 IOPS continus
        aof_iops = 500

    recommended_iops = max(rdb_iops, aof_iops) + 500  # +500 pour logs/OS

    # Type de disque recommandé
    if recommended_iops < 3000:
        disk_type = "SSD SATA"
    elif recommended_iops < 10000:
        disk_type = "SSD NVMe"
    else:
        disk_type = "SSD NVMe High Performance"

    return {
        'dataset_size_gb': dataset_size_gb,
        'rdb_space_gb': round(rdb_space_gb, 2),
        'aof_space_gb': round(aof_space_gb, 2),
        'log_space_gb': round(log_space_gb, 2),
        'margin_gb': round(margin_gb, 2),
        'total_disk_gb': round(total_disk_gb, 2),
        'recommended_iops': recommended_iops,
        'disk_type': disk_type
    }

# Exemples
if __name__ == '__main__':
    print("=== DISK SIZING EXAMPLES ===\n")

    # Exemple 1: Cache sans persistance
    print("1. Cache (no persistence):")
    result1 = calculate_disk_requirements(
        dataset_size_gb=20,
        rdb_enabled=False,
        aof_enabled=False
    )
    for key, value in result1.items():
        print(f"   {key}: {value}")

    print("\n2. Session store (RDB only):")
    result2 = calculate_disk_requirements(
        dataset_size_gb=50,
        rdb_enabled=True,
        aof_enabled=False,
        rdb_retention_days=7
    )
    for key, value in result2.items():
        print(f"   {key}: {value}")

    print("\n3. Database (RDB + AOF):")
    result3 = calculate_disk_requirements(
        dataset_size_gb=100,
        rdb_enabled=True,
        aof_enabled=True,
        rdb_retention_days=14
    )
    for key, value in result3.items():
        print(f"   {key}: {value}")
```

---

## Planification de capacité et croissance

### 1. Modèle de croissance

```python
#!/usr/bin/env python3
# redis-capacity-planning.py

import math

def forecast_capacity(current_dataset_gb, current_keys,
                     growth_rate_monthly=0.10, months_ahead=12):
    """
    Prévision de capacité Redis

    Args:
        current_dataset_gb: Taille actuelle dataset (GB)
        current_keys: Nombre actuel de clés
        growth_rate_monthly: Taux de croissance mensuel (0.10 = 10%)
        months_ahead: Nombre de mois à prévoir
    """

    forecast = []

    for month in range(1, months_ahead + 1):
        # Croissance composée
        projected_keys = current_keys * math.pow(1 + growth_rate_monthly, month)
        projected_dataset_gb = current_dataset_gb * math.pow(1 + growth_rate_monthly, month)

        # Calcul RAM nécessaire avec overhead
        ram_needed_gb = projected_dataset_gb * 2.5  # Factor 2.5 (overhead + fragmentation + COW)

        # Instance size recommandée
        instance_sizes = [8, 16, 32, 64, 128, 256, 384, 512]
        recommended_instance = next((size for size in instance_sizes if size >= ram_needed_gb), 512)

        forecast.append({
            'month': month,
            'keys': int(projected_keys),
            'dataset_gb': round(projected_dataset_gb, 2),
            'ram_needed_gb': round(ram_needed_gb, 2),
            'recommended_instance_gb': recommended_instance,
            'utilization_pct': round((ram_needed_gb / recommended_instance) * 100, 1)
        })

    return forecast

def print_forecast(forecast):
    """Affiche prévisions"""
    print("=" * 90)
    print("REDIS CAPACITY FORECAST")
    print("=" * 90)
    print(f"{'Month':<8} {'Keys':>15} {'Dataset GB':>12} {'RAM Needed':>12} {'Instance':>10} {'Usage %':>8}")
    print("-" * 90)

    for entry in forecast:
        print(f"{entry['month']:<8} {entry['keys']:>15,} {entry['dataset_gb']:>12.2f} "
              f"{entry['ram_needed_gb']:>12.2f} {entry['recommended_instance_gb']:>10} "
              f"{entry['utilization_pct']:>8.1f}%")

    # Alertes
    print("\n" + "=" * 90)
    print("ALERTS & RECOMMENDATIONS:")
    print("=" * 90)

    for entry in forecast:
        if entry['utilization_pct'] > 80:
            print(f"⚠️  Month {entry['month']}: Utilization {entry['utilization_pct']}% - "
                  f"UPGRADE SOON to {entry['recommended_instance_gb']*2}GB instance")
        elif entry['utilization_pct'] > 90:
            print(f"🚨 Month {entry['month']}: Utilization {entry['utilization_pct']}% - "
                  f"CRITICAL: UPGRADE IMMEDIATELY")

# Exemple
if __name__ == '__main__':
    # Scénario: Application en croissance
    forecast = forecast_capacity(
        current_dataset_gb=20,  # 20GB actuellement
        current_keys=5_000_000,  # 5M clés
        growth_rate_monthly=0.15,  # 15% croissance par mois
        months_ahead=12
    )

    print_forecast(forecast)
```

### 2. Stratégies de scaling

```
┌─────────────────────────────────────────────────────────────────┐
│                  STRATÉGIES DE SCALING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VERTICAL SCALING (Scale Up):                                   │
│  ├── Augmenter RAM/CPU de l'instance                            │
│  ├── Avantages:                                                 │
│  │   • Simple (pas de sharding)                                 │
│  │   • Multi-key operations possibles                           │
│  │   • Transactions fonctionnent                                │
│  ├── Inconvénients:                                             │
│  │   • Limite physique (512GB-1TB max)                          │
│  │   • Downtime pendant resize                                  │
│  │   • Single point of failure                                  │
│  └── Quand: Dataset < 200GB, pas de contrainte HA extrême       │
│                                                                 │
│  HORIZONTAL SCALING (Scale Out):                                │
│  ├── Redis Cluster (sharding)                                   │
│  ├── Avantages:                                                 │
│  │   • Scalabilité illimitée                                    │
│  │   • Haute disponibilité native                               │
│  │   • Pas de limite de taille                                  │
│  ├── Inconvénients:                                             │
│  │   • Multi-key operations limitées                            │
│  │   • Complexité opérationnelle                                │
│  │   • Migration difficile                                      │
│  └── Quand: Dataset > 200GB, haute disponibilité critique       │
│                                                                 │
│  READ SCALING:                                                  │
│  ├── Ajouter replicas (read-only)                               │
│  ├── Avantages:                                                 │
│  │   • Scale reads facilement                                   │
│  │   • Haute disponibilité                                      │
│  │   • Simple à mettre en place                                 │
│  ├── Inconvénients:                                             │
│  │   • Writes pas scalés                                        │
│  │   • Replication lag possible                                 │
│  │   • Coût RAM (chaque replica = RAM complète)                 │
│  └── Quand: Read-heavy workload (90%+ reads)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Matrices de dimensionnement

### 1. Tailles d'instances par use case

```
┌─────────────────────────────────────────────────────────────────┐
│          REDIS INSTANCE SIZING MATRIX                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Use Case               Keys      Dataset   RAM     CPU  Disk   │
│  ────────────────────────────────────────────────────────────   │
│  Dev/Test              100K       100MB     2GB     2    20GB   │
│  Small Cache           1M         2GB       8GB     2    50GB   │
│  Medium Cache          5M         10GB      32GB    4    100GB  │
│  Large Cache           20M        40GB      128GB   8    200GB  │
│  Session Store         10M        20GB      64GB    4    150GB  │
│  Job Queue             5M         10GB      32GB    4    200GB  │
│  Analytics (HLL)       1M         12GB      32GB    2    100GB  │
│  Primary Database      50M        100GB     256GB   8    1TB    │
│  Large Primary DB      200M       400GB     512GB   16   2TB    │
│  ────────────────────────────────────────────────────────────   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Sizing par Cloud Provider

#### AWS (ElastiCache/MemoryDB)

```
Instance Type        Memory   vCPU   Network   Price/h (approx)
──────────────────────────────────────────────────────────────
cache.t3.micro       0.5 GB   2      Low       $0.017
cache.t3.small       1.4 GB   2      Low       $0.034
cache.t3.medium      3.1 GB   2      Low       $0.068
cache.m6g.large      6.4 GB   2      Up to 10G $0.161
cache.m6g.xlarge     12.9 GB  4      Up to 10G $0.322
cache.m6g.2xlarge    25.6 GB  8      Up to 10G $0.644
cache.r6g.large      13.1 GB  2      Up to 10G $0.201
cache.r6g.xlarge     26.3 GB  4      Up to 10G $0.403
cache.r6g.2xlarge    52.5 GB  8      Up to 12G $0.806
cache.r6g.4xlarge    105 GB   16     Up to 12G $1.612
cache.r6g.8xlarge    209 GB   32     25 Gbps   $3.223
cache.r6g.12xlarge   314 GB   48     38 Gbps   $4.835
cache.r6g.16xlarge   419 GB   64     50 Gbps   $6.446
```

#### Azure (Azure Cache for Redis)

```
Tier            Memory   vCPU   Network        Price/h (approx)
─────────────────────────────────────────────────────────────
Basic C0        250 MB   Shared Low           $0.020
Basic C1        1 GB     1      Low            $0.044
Standard C2     2.5 GB   2      Moderate       $0.121
Standard C3     6 GB     4      Moderate       $0.242
Premium P1      6 GB     2      Up to 10G      $0.371
Premium P2      13 GB    4      Up to 10G      $0.742
Premium P3      26 GB    4      Up to 10G      $1.483
Premium P4      53 GB    8      Up to 10G      $2.967
Premium P5      120 GB   20     Up to 10G      $6.708
```

---

## Checklist de dimensionnement

### Checklist Planification

- [ ] **Estimation du dataset**
  - Nombre de clés projeté
  - Taille moyenne par clé
  - Type de structures (String, Hash, etc.)

- [ ] **Calcul overhead**
  - Redis overhead (20-50%)
  - Fragmentation (10-30%)
  - COW pour persistance (0-100%)

- [ ] **Marge de croissance**
  - Croissance mensuelle estimée
  - Horizon de planification (6-12 mois)
  - Budget de scaling

- [ ] **Persistance**
  - RDB: fréquence et rétention
  - AOF: mode (everysec/always)
  - Espace disque nécessaire

- [ ] **Performance**
  - Ops/sec attendues
  - Latence p99 cible
  - Pattern de charge (pics)

- [ ] **Haute disponibilité**
  - Réplication (nombre de replicas)
  - Sentinel ou Cluster
  - Multi-AZ/région

### Checklist Post-déploiement

- [ ] **Monitoring configuré**
  - Utilisation mémoire
  - Évictions
  - Hit rate
  - Latency

- [ ] **Alertes définies**
  - RAM > 80%
  - Évictions > seuil
  - Latency dégradée

- [ ] **Tests de charge**
  - Benchmarks effectués
  - Performances validées

- [ ] **Plan de scaling**
  - Triggers d'upgrade définis
  - Procédure de resize documentée

---

## 📚 Ressources complémentaires

### Outils

- **redis-benchmark** - Benchmarking
- **redis-rdb-tools** - Analyse RDB
- **Cloud calculators** - AWS/Azure/GCP pricing
- **redis-memory-analyzer** - Analyse mémoire

### Documentation

- [Redis Memory Optimization](https://redis.io/docs/management/optimization/memory-optimization/)
- [Redis Benchmarks](https://redis.io/docs/management/optimization/benchmarks/)
- [AWS ElastiCache Sizing](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/CacheNodes.SelectSize.html)

---

**Section suivante :** [12.8 - Mises à jour sans downtime (Rolling upgrades)](./08-mises-a-jour-rolling-upgrades.md)

⏭️ [Mises à jour sans downtime (Rolling upgrades)](/12-redis-production-securite/08-mises-a-jour-rolling-upgrades.md)
