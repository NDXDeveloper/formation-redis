🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.9 - Benchmarking avec redis-benchmark

## 🎯 Objectifs de cette section

- Maîtriser l'outil redis-benchmark
- Comprendre les métriques de performance
- Mettre en place des benchmarks reproductibles
- Interpréter correctement les résultats
- Comparer différentes configurations
- Identifier les goulots d'étranglement
- Établir des baselines de performance

---

## 📚 Introduction : Le benchmarking Redis

### Pourquoi benchmarker ?

Le **benchmarking** permet de :

```
1. ÉTABLIR UNE BASELINE
   └─ Connaître les performances normales de l'instance

2. DÉTECTER LES RÉGRESSIONS
   └─ Comparer avant/après un changement

3. DIMENSIONNER L'INFRASTRUCTURE
   └─ Prévoir les besoins en ressources

4. OPTIMISER LA CONFIGURATION
   └─ Identifier les meilleurs paramètres

5. VALIDER LES ATTENTES
   └─ Vérifier que Redis peut supporter la charge
```

### Métriques clés

```
┌─────────────────────────────────────────────────┐
│  MÉTRIQUES DE PERFORMANCE REDIS                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  Throughput (débit)                             │
│  └─ Requests/sec (ops/sec)                      │
│     Mesure : Combien de commandes par seconde   │
│     Objectif : Maximiser                        │
│                                                 │
│  Latency (latence)                              │
│  └─ Millisecondes (ms)                          │
│     Mesure : Temps de réponse                   │
│     Objectif : Minimiser                        │
│     Important : P50, P95, P99                   │
│                                                 │
│  CPU utilization                                │
│  └─ Pourcentage (%)                             │
│     Mesure : Charge CPU de Redis                │
│     Objectif : < 80% pour headroom              │
│                                                 │
│  Memory usage                                   │
│  └─ Bytes                                       │
│     Mesure : Consommation mémoire               │
│     Objectif : < maxmemory                      │
│                                                 │
│  Network bandwidth                              │
│  └─ MB/s                                        │
│     Mesure : Bande passante utilisée            │
│     Objectif : < capacité réseau                │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Performances théoriques

**Ordres de grandeur attendus** :

```
Single-threaded Redis (1 core) :
├─ GET/SET simple : 80,000 - 120,000 ops/sec
├─ Pipelined : 500,000 - 1,000,000 ops/sec
└─ Latence P99 : < 1ms (réseau local)

Multi-threaded I/O Redis 6+ :
├─ GET/SET simple : 150,000 - 200,000 ops/sec
└─ Pipelined : 1,000,000 - 2,000,000 ops/sec

Facteurs limitants :
├─ CPU (single-thread performance)
├─ Réseau (bandwidth, latency)
├─ Mémoire (si swap)
└─ Disque (si persistence active)
```

---

## 🔧 redis-benchmark : L'outil natif

### Installation et disponibilité

```bash
# redis-benchmark est installé avec Redis
which redis-benchmark
# /usr/bin/redis-benchmark

# Version
redis-benchmark --version
# redis-benchmark 7.0.0
```

### Usage de base

```bash
# Benchmark simple (défaut)
redis-benchmark

# Output :
# ====== PING_INLINE ======
#   100000 requests completed in 0.91 seconds
#   50 parallel clients
#   3 bytes payload
#   keep alive: 1
#
# 99.99% <= 1 milliseconds
# 110132.16 requests per second
```

### Options principales

```bash
# Options de base
redis-benchmark -h localhost        # Host
redis-benchmark -p 6379            # Port
redis-benchmark -a password        # Authentification
redis-benchmark -c 50              # Nombre de clients parallèles
redis-benchmark -n 100000          # Nombre total de requêtes
redis-benchmark -d 256             # Taille des données (bytes)

# Exemples
redis-benchmark -c 100 -n 1000000  # 100 clients, 1M requêtes
redis-benchmark -d 1024            # Valeurs de 1KB
```

### Tests sélectifs

```bash
# Tester seulement certaines commandes
redis-benchmark -t get,set
redis-benchmark -t lpush,lpop
redis-benchmark -t zadd,zrem

# Liste des tests disponibles :
# - ping_inline
# - ping_mbulk
# - set
# - get
# - incr
# - lpush
# - rpush
# - lpop
# - rpop
# - sadd
# - hset
# - spop
# - zadd
# - zpopmin
# - lrange_100 (premier 100 éléments)
# - lrange_300 (premier 300 éléments)
# - lrange_500 (premier 500 éléments)
# - lrange_600 (premier 600 éléments)
# - mset
```

### Options avancées

```bash
# Pipelining (batch de commandes)
redis-benchmark -P 16              # 16 commandes par pipeline

# Quiet mode (affiche seulement les résultats)
redis-benchmark -q

# Mode CSV (pour exports)
redis-benchmark --csv

# Durée au lieu de nombre de requêtes
redis-benchmark -t get -c 50 --rps 10000  # 10K req/sec pendant test

# Clés aléatoires (au lieu de séquentielles)
redis-benchmark -r 1000000         # Pool de 1M clés aléatoires

# Idle connections
redis-benchmark --idle-timeout 60  # Timeout des connexions idle
```

---

## 📊 Méthodologie de benchmarking

### Framework BENCHMARK

```
B - Baseline : Établir une référence
E - Environment : Environnement stable
N - Numbers : Nombre de tests suffisant
C - Conditions : Conditions réalistes
H - History : Historique des résultats
M - Metrics : Métriques pertinentes
A - Analyze : Analyse approfondie
R - Reproduce : Reproductibilité
K - Knowledge : Documentation
```

### Étape 1 : Préparation de l'environnement

```bash
#!/bin/bash
# prepare-benchmark.sh

echo "=== BENCHMARK PREPARATION ==="
echo ""

# 1. Vérifier que Redis tourne
if ! systemctl is-active --quiet redis; then
    echo "❌ Redis is not running"
    exit 1
fi
echo "✅ Redis is running"

# 2. Vérifier la config Redis
echo ""
echo "Redis Configuration:"
redis-cli INFO server | grep redis_version
redis-cli CONFIG GET maxmemory
redis-cli CONFIG GET appendonly

# 3. Vérifier les ressources système
echo ""
echo "System Resources:"
echo "CPU cores: $(nproc)"
echo "Memory:"
free -h | grep Mem
echo "Disk I/O:"
iostat -x 1 2 | tail -n +4

# 4. Nettoyer l'instance (pour benchmarks propres)
echo ""
read -p "Flush all data for clean benchmark? [y/N]: " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    redis-cli FLUSHALL
    echo "✅ Data flushed"
fi

# 5. Désactiver temporairement la persistence (pour benchmark pur)
echo ""
read -p "Disable persistence for benchmark? [y/N]: " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    redis-cli CONFIG SET save ""
    redis-cli CONFIG SET appendonly no
    echo "✅ Persistence disabled"
    echo "⚠️  Remember to re-enable after benchmark!"
fi

# 6. Informations réseau
echo ""
echo "Network Info:"
IFACE=$(ip route | grep default | awk '{print $5}')
echo "Interface: $IFACE"
ethtool $IFACE 2>/dev/null | grep Speed || echo "Speed: Unknown"

echo ""
echo "=== READY FOR BENCHMARK ==="
```

### Étape 2 : Benchmark de base (baseline)

```bash
#!/bin/bash
# baseline-benchmark.sh

OUTPUT_DIR="benchmark_results"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
mkdir -p "$OUTPUT_DIR"

echo "=== BASELINE BENCHMARK - $TIMESTAMP ==="
echo ""

# Fonction pour logger et exécuter
run_benchmark() {
    local name=$1
    local cmd=$2
    local output="${OUTPUT_DIR}/${name}_${TIMESTAMP}.txt"

    echo "Running: $name"
    echo "Command: $cmd"
    echo ""

    echo "=== $name ===" > "$output"
    echo "Timestamp: $TIMESTAMP" >> "$output"
    echo "Command: $cmd" >> "$output"
    echo "" >> "$output"

    eval "$cmd" >> "$output" 2>&1

    echo "✅ Saved to: $output"
    echo ""
}

# 1. Benchmark simple (baseline)
run_benchmark "baseline_simple" \
    "redis-benchmark -q -c 50 -n 100000"

# 2. Benchmark avec différentes tailles de données
run_benchmark "baseline_100b" \
    "redis-benchmark -q -c 50 -n 100000 -d 100"

run_benchmark "baseline_1kb" \
    "redis-benchmark -q -c 50 -n 100000 -d 1024"

run_benchmark "baseline_10kb" \
    "redis-benchmark -q -c 50 -n 100000 -d 10240"

# 3. Benchmark avec pipelining
run_benchmark "baseline_pipeline_p10" \
    "redis-benchmark -q -c 50 -n 100000 -P 10"

run_benchmark "baseline_pipeline_p50" \
    "redis-benchmark -q -c 50 -n 100000 -P 50"

# 4. Benchmark par type de commande
for cmd in set get incr lpush lpop sadd spop zadd hset; do
    run_benchmark "baseline_${cmd}" \
        "redis-benchmark -q -t ${cmd} -c 50 -n 100000"
done

# 5. Benchmark avec clés aléatoires
run_benchmark "baseline_random_keys" \
    "redis-benchmark -q -c 50 -n 100000 -r 100000"

# 6. Générer un résumé
SUMMARY="${OUTPUT_DIR}/summary_${TIMESTAMP}.txt"
echo "=== BENCHMARK SUMMARY ===" > "$SUMMARY"
echo "Date: $(date)" >> "$SUMMARY"
echo "" >> "$SUMMARY"

echo "Redis Info:" >> "$SUMMARY"
redis-cli INFO server | grep -E "redis_version|os|arch_bits" >> "$SUMMARY"
echo "" >> "$SUMMARY"

echo "System Info:" >> "$SUMMARY"
echo "CPU: $(nproc) cores" >> "$SUMMARY"
echo "Memory: $(free -h | grep Mem | awk '{print $2}')" >> "$SUMMARY"
echo "" >> "$SUMMARY"

echo "Results:" >> "$SUMMARY"
grep "requests per second" "${OUTPUT_DIR}"/*_${TIMESTAMP}.txt >> "$SUMMARY"

echo ""
echo "=== BENCHMARK COMPLETE ==="
echo "Summary: $SUMMARY"
```

### Étape 3 : Benchmark détaillé avec latences

```bash
#!/bin/bash
# latency-benchmark.sh

echo "=== LATENCY BENCHMARK ==="
echo ""

# Benchmark avec distribution de latence
redis-benchmark -t get,set -c 50 -n 1000000 --csv > latency_results.csv

# Extraire les percentiles
echo "Latency Distribution:"
echo ""

# Parser les résultats
awk -F',' 'NR>1 {
    test=$1
    rps=$2

    # Approximation des percentiles depuis le throughput
    # (simplifié, redis-benchmark ne donne pas les percentiles directement)

    printf "%-15s %10s req/sec\n", test, rps
}' latency_results.csv

# Test de latence intrinsèque
echo ""
echo "Intrinsic Latency Test (30s):"
redis-cli --intrinsic-latency 30

# Latency monitoring
echo ""
echo "Latency History (60s):"
timeout 60 redis-cli --latency-history
```

### Étape 4 : Benchmark de stress

```bash
#!/bin/bash
# stress-benchmark.sh

echo "=== STRESS BENCHMARK ==="
echo ""

# Fonction pour monitorer les ressources pendant le benchmark
monitor_resources() {
    local duration=$1
    local output=$2

    {
        echo "timestamp,cpu_user,cpu_system,mem_used_mb,net_rx_mb,net_tx_mb"

        end_time=$((POSIX_TIME + duration))
        while [ $(date +%s) -lt $end_time ]; do
            timestamp=$(date +%s)

            # CPU
            cpu_stats=$(top -bn1 | grep "Cpu(s)" | awk '{print $2,$4}')
            cpu_user=$(echo $cpu_stats | cut -d' ' -f1)
            cpu_sys=$(echo $cpu_stats | cut -d' ' -f2)

            # Memory
            mem_used=$(free -m | awk 'NR==2{print $3}')

            # Network (approximatif)
            net_stats=$(cat /proc/net/dev | grep eth0 | awk '{print $2,$10}')
            net_rx=$(echo $net_stats | cut -d' ' -f1)
            net_tx=$(echo $net_stats | cut -d' ' -f2)
            net_rx_mb=$((net_rx / 1024 / 1024))
            net_tx_mb=$((net_tx / 1024 / 1024))

            echo "$timestamp,$cpu_user,$cpu_sys,$mem_used,$net_rx_mb,$net_tx_mb"

            sleep 1
        done
    } > "$output"
}

# Test 1 : Charge croissante
echo "Test 1: Increasing Load"
for clients in 10 50 100 200 500; do
    echo "  Testing with $clients clients..."
    monitor_resources 30 "stress_c${clients}_resources.csv" &
    MONITOR_PID=$!

    redis-benchmark -c $clients -n 100000 -q -t set,get \
        > "stress_c${clients}_results.txt"

    kill $MONITOR_PID 2>/dev/null
    wait $MONITOR_PID 2>/dev/null

    sleep 5  # Cooldown
done

# Test 2 : Stress avec pipelining
echo ""
echo "Test 2: Pipeline Stress"
for pipeline in 1 10 50 100 500; do
    echo "  Testing with pipeline=$pipeline..."
    redis-benchmark -c 50 -n 100000 -P $pipeline -q -t set,get \
        > "stress_p${pipeline}_results.txt"

    sleep 5
done

# Test 3 : Gros payloads
echo ""
echo "Test 3: Large Payloads"
for size in 1024 10240 102400 1048576; do
    echo "  Testing with ${size} bytes payload..."
    redis-benchmark -c 50 -n 10000 -d $size -q -t set,get \
        > "stress_size${size}_results.txt"

    sleep 5
done

echo ""
echo "=== STRESS BENCHMARK COMPLETE ==="
```

---

## 📈 Interprétation des résultats

### Lire les outputs de redis-benchmark

```
====== SET ======
  100000 requests completed in 0.91 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1
  host configuration "save":
  host configuration "appendonly": no

0.00% <= 0.1 milliseconds
18.97% <= 0.2 milliseconds
87.24% <= 0.3 milliseconds
98.54% <= 0.4 milliseconds
99.83% <= 0.5 milliseconds
99.95% <= 0.6 milliseconds
100.00% <= 0.6 milliseconds
110132.16 requests per second

Interprétation :
├─ Throughput : 110,132 ops/sec (bon)
├─ P50 : < 0.3ms (87% < 0.3ms)
├─ P99 : < 0.5ms (99.83% < 0.5ms)
└─ P100 (max) : 0.6ms (excellent)
```

### Calculer les métriques dérivées

```python
#!/usr/bin/env python3
"""
Analyse des résultats de benchmark
"""
import re
import statistics
from pathlib import Path

class BenchmarkAnalyzer:
    def __init__(self, results_dir='benchmark_results'):
        self.results_dir = Path(results_dir)
        self.results = {}

    def parse_result_file(self, filepath):
        """Parse un fichier de résultat redis-benchmark"""
        with open(filepath, 'r') as f:
            content = f.read()

        # Extraire les métriques
        metrics = {}

        # Requests per second
        rps_match = re.search(r'([\d.]+) requests per second', content)
        if rps_match:
            metrics['rps'] = float(rps_match.group(1))

        # Latences (percentiles)
        percentiles = {}
        for match in re.finditer(r'([\d.]+)% <= ([\d.]+) milliseconds', content):
            percentile = float(match.group(1))
            latency = float(match.group(2))
            percentiles[percentile] = latency

        if percentiles:
            metrics['percentiles'] = percentiles

            # Calculer P50, P95, P99 si disponibles
            if 50.0 in percentiles:
                metrics['p50'] = percentiles[50.0]
            else:
                # Approximation
                sorted_p = sorted(percentiles.items())
                for p, lat in sorted_p:
                    if p >= 50:
                        metrics['p50'] = lat
                        break

            metrics['p95'] = percentiles.get(95.0) or \
                            max(lat for p, lat in percentiles.items() if p <= 95)
            metrics['p99'] = percentiles.get(99.0) or \
                            max(lat for p, lat in percentiles.items() if p <= 99)
            metrics['max'] = max(percentiles.values())

        return metrics

    def load_all_results(self):
        """Charge tous les résultats"""
        for filepath in self.results_dir.glob('*.txt'):
            if filepath.name.startswith('summary'):
                continue

            test_name = filepath.stem
            self.results[test_name] = self.parse_result_file(filepath)

    def compare_results(self, test1, test2):
        """Compare deux tests"""
        if test1 not in self.results or test2 not in self.results:
            return None

        r1 = self.results[test1]
        r2 = self.results[test2]

        comparison = {}

        # Throughput
        if 'rps' in r1 and 'rps' in r2:
            rps_diff = r2['rps'] - r1['rps']
            rps_pct = (rps_diff / r1['rps']) * 100
            comparison['throughput'] = {
                'test1': r1['rps'],
                'test2': r2['rps'],
                'diff': rps_diff,
                'pct_change': rps_pct
            }

        # Latences
        for metric in ['p50', 'p95', 'p99', 'max']:
            if metric in r1 and metric in r2:
                diff = r2[metric] - r1[metric]
                pct = (diff / r1[metric]) * 100
                comparison[metric] = {
                    'test1': r1[metric],
                    'test2': r2[metric],
                    'diff': diff,
                    'pct_change': pct
                }

        return comparison

    def generate_report(self):
        """Génère un rapport complet"""
        print("=" * 80)
        print("BENCHMARK ANALYSIS REPORT")
        print("=" * 80)
        print()

        # Résumé par test
        print("Test Results Summary:")
        print()
        print(f"{'Test Name':<40} {'RPS':<15} {'P50':<10} {'P99':<10} {'Max':<10}")
        print("-" * 80)

        sorted_tests = sorted(
            self.results.items(),
            key=lambda x: x[1].get('rps', 0),
            reverse=True
        )

        for test_name, metrics in sorted_tests:
            rps = metrics.get('rps', 0)
            p50 = metrics.get('p50', 0)
            p99 = metrics.get('p99', 0)
            max_lat = metrics.get('max', 0)

            print(f"{test_name:<40} {rps:<15,.0f} {p50:<10.2f} {p99:<10.2f} {max_lat:<10.2f}")

        print()
        print("-" * 80)

        # Top performers
        print()
        print("Top 5 by Throughput:")
        for i, (test_name, metrics) in enumerate(sorted_tests[:5], 1):
            rps = metrics.get('rps', 0)
            print(f"  {i}. {test_name}: {rps:,.0f} ops/sec")

        # Analyse statistique
        if len(self.results) > 1:
            print()
            print("Statistical Analysis:")

            all_rps = [m.get('rps', 0) for m in self.results.values() if 'rps' in m]
            if all_rps:
                print(f"  Average RPS: {statistics.mean(all_rps):,.0f}")
                print(f"  Median RPS:  {statistics.median(all_rps):,.0f}")
                print(f"  Std Dev:     {statistics.stdev(all_rps):,.0f}")
                print(f"  Min RPS:     {min(all_rps):,.0f}")
                print(f"  Max RPS:     {max(all_rps):,.0f}")

        print()
        print("=" * 80)

    def export_csv(self, output_file='benchmark_summary.csv'):
        """Exporte les résultats en CSV"""
        with open(output_file, 'w') as f:
            # Header
            f.write("test_name,rps,p50,p95,p99,max_latency\n")

            # Data
            for test_name, metrics in self.results.items():
                rps = metrics.get('rps', '')
                p50 = metrics.get('p50', '')
                p95 = metrics.get('p95', '')
                p99 = metrics.get('p99', '')
                max_lat = metrics.get('max', '')

                f.write(f"{test_name},{rps},{p50},{p95},{p99},{max_lat}\n")

        print(f"Exported to: {output_file}")

if __name__ == "__main__":
    analyzer = BenchmarkAnalyzer()
    analyzer.load_all_results()
    analyzer.generate_report()
    analyzer.export_csv()
```

---

## 🔬 Benchmarks personnalisés

### Test custom avec commandes spécifiques

```bash
# Fichier de commandes personnalisées
cat > custom_commands.txt << 'EOF'
SET key1 "value1"
GET key1
INCR counter
SADD myset "member1"
ZADD myzset 1 "item1"
HSET myhash field1 "value1"
EOF

# Exécuter le benchmark custom
redis-benchmark -c 50 -n 100000 -r 10000 \
    $(cat custom_commands.txt | sed 's/^/-e "/; s/$/"/' | tr '\n' ' ')
```

### Benchmark de scénarios réalistes

```python
#!/usr/bin/env python3
"""
Benchmark de scénarios applicatifs réalistes
"""
import redis
import time
import random
import string
from concurrent.futures import ThreadPoolExecutor
from collections import defaultdict

class ScenarioBenchmark:
    def __init__(self, host='localhost', port=6379):
        self.host = host
        self.port = port
        self.results = defaultdict(list)

    def _get_redis_connection(self):
        """Créer une connexion Redis"""
        return redis.Redis(host=self.host, port=self.port, decode_responses=True)

    def _random_string(self, length=10):
        """Générer une chaîne aléatoire"""
        return ''.join(random.choices(string.ascii_letters, k=length))

    def scenario_user_session(self, user_id):
        """
        Scénario : Session utilisateur
        - Créer session
        - Lire données utilisateur
        - Mettre à jour compteurs
        - Expiration
        """
        r = self._get_redis_connection()

        start = time.perf_counter()

        # Créer session
        session_id = f"session:{user_id}:{random.randint(1000, 9999)}"
        r.setex(session_id, 3600, self._random_string(100))

        # Lire données utilisateur
        user_data = r.hgetall(f"user:{user_id}")

        # Incrémenter compteurs
        pipe = r.pipeline()
        pipe.incr(f"user:{user_id}:page_views")
        pipe.incr(f"stats:total_page_views")
        pipe.execute()

        # Ajouter à un leaderboard
        r.zincrby("leaderboard:active_users", 1, f"user:{user_id}")

        end = time.perf_counter()

        return (end - start) * 1000  # ms

    def scenario_cache_read_write(self, key_id):
        """
        Scénario : Cache avec read/write
        - Tentative de lecture
        - Si miss, écriture
        - Si hit, retour
        """
        r = self._get_redis_connection()

        start = time.perf_counter()

        key = f"cache:item:{key_id}"

        # Tentative de lecture
        data = r.get(key)

        if not data:
            # Cache miss : générer et stocker
            data = self._random_string(1000)
            r.setex(key, 300, data)  # TTL 5 minutes

        end = time.perf_counter()

        return (end - start) * 1000  # ms

    def scenario_leaderboard_update(self, user_id):
        """
        Scénario : Mise à jour leaderboard
        - Calculer score
        - Mettre à jour sorted set
        - Récupérer rang
        """
        r = self._get_redis_connection()

        start = time.perf_counter()

        # Calculer un score (simulé)
        score = random.randint(1000, 100000)

        # Mettre à jour le leaderboard
        r.zadd("leaderboard:global", {f"user:{user_id}": score})

        # Récupérer le rang de l'utilisateur
        rank = r.zrevrank("leaderboard:global", f"user:{user_id}")

        # Récupérer le top 10
        top10 = r.zrevrange("leaderboard:global", 0, 9, withscores=True)

        end = time.perf_counter()

        return (end - start) * 1000  # ms

    def scenario_queue_processing(self, task_id):
        """
        Scénario : Queue de tâches
        - Push task
        - Pop task
        - Marquer comme traitée
        """
        r = self._get_redis_connection()

        start = time.perf_counter()

        # Push task
        task_data = self._random_string(500)
        r.lpush("queue:tasks", task_data)

        # Pop task
        task = r.rpop("queue:tasks")

        if task:
            # Marquer comme traitée
            r.sadd("queue:processed", task)

        end = time.perf_counter()

        return (end - start) * 1000  # ms

    def run_scenario(self, scenario_func, name, iterations=10000, workers=10):
        """
        Exécute un scénario avec plusieurs workers
        """
        print(f"Running scenario: {name}")
        print(f"  Iterations: {iterations}")
        print(f"  Workers: {workers}")

        latencies = []

        start_time = time.time()

        with ThreadPoolExecutor(max_workers=workers) as executor:
            futures = []

            for i in range(iterations):
                future = executor.submit(scenario_func, i)
                futures.append(future)

            for future in futures:
                try:
                    latency = future.result()
                    latencies.append(latency)
                except Exception as e:
                    print(f"  Error: {e}")

        end_time = time.time()

        # Calculer les métriques
        duration = end_time - start_time
        throughput = iterations / duration

        latencies.sort()
        p50 = latencies[int(len(latencies) * 0.50)]
        p95 = latencies[int(len(latencies) * 0.95)]
        p99 = latencies[int(len(latencies) * 0.99)]
        avg = sum(latencies) / len(latencies)

        results = {
            'name': name,
            'iterations': iterations,
            'workers': workers,
            'duration': duration,
            'throughput': throughput,
            'latency_avg': avg,
            'latency_p50': p50,
            'latency_p95': p95,
            'latency_p99': p99,
            'latency_max': max(latencies)
        }

        self.results[name] = results

        print(f"  ✅ Complete in {duration:.2f}s")
        print(f"  Throughput: {throughput:.2f} ops/sec")
        print(f"  Latency P50: {p50:.2f}ms")
        print(f"  Latency P99: {p99:.2f}ms")
        print()

        return results

    def run_all_scenarios(self):
        """Exécute tous les scénarios"""
        print("=" * 60)
        print("SCENARIO BENCHMARKS")
        print("=" * 60)
        print()

        # Setup : données initiales
        r = self._get_redis_connection()

        # Créer quelques utilisateurs
        for i in range(1000):
            r.hset(f"user:{i}", mapping={
                'name': f'User{i}',
                'email': f'user{i}@example.com',
                'score': random.randint(0, 1000)
            })

        # Exécuter les scénarios
        self.run_scenario(self.scenario_user_session, "User Session", 10000, 20)
        self.run_scenario(self.scenario_cache_read_write, "Cache R/W", 10000, 20)
        self.run_scenario(self.scenario_leaderboard_update, "Leaderboard", 10000, 20)
        self.run_scenario(self.scenario_queue_processing, "Queue Processing", 10000, 20)

        # Rapport final
        self.generate_report()

    def generate_report(self):
        """Génère un rapport des scénarios"""
        print()
        print("=" * 60)
        print("SCENARIO BENCHMARK REPORT")
        print("=" * 60)
        print()

        print(f"{'Scenario':<25} {'Throughput':<15} {'P50 (ms)':<12} {'P99 (ms)':<12}")
        print("-" * 60)

        for name, results in self.results.items():
            print(f"{name:<25} {results['throughput']:<15.2f} {results['latency_p50']:<12.2f} {results['latency_p99']:<12.2f}")

if __name__ == "__main__":
    benchmark = ScenarioBenchmark()
    benchmark.run_all_scenarios()
```

---

## 🔀 Comparaison de configurations

### Benchmark A/B Testing

```bash
#!/bin/bash
# ab-test-benchmark.sh

echo "=== A/B CONFIGURATION BENCHMARK ==="
echo ""

# Configuration A : Défaut
echo "Configuration A: Default"
redis-cli CONFIG SET save "900 1 300 10 60 10000"
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec

sleep 2

redis-benchmark -q -c 50 -n 100000 -t set,get \
    > config_a_results.txt

echo "  Config A results saved"

# Configuration B : Performance (pas de persistence)
echo ""
echo "Configuration B: No Persistence"
redis-cli CONFIG SET save ""
redis-cli CONFIG SET appendonly no

sleep 2

redis-benchmark -q -c 50 -n 100000 -t set,get \
    > config_b_results.txt

echo "  Config B results saved"

# Comparer
echo ""
echo "=== COMPARISON ==="

echo "Config A (with persistence):"
grep "requests per second" config_a_results.txt

echo ""
echo "Config B (no persistence):"
grep "requests per second" config_b_results.txt

# Calculer le gain
SET_A=$(grep "SET:" config_a_results.txt | grep -oP '\d+\.\d+' | head -1)
SET_B=$(grep "SET:" config_b_results.txt | grep -oP '\d+\.\d+' | head -1)

if [ -n "$SET_A" ] && [ -n "$SET_B" ]; then
    GAIN=$(echo "scale=2; ($SET_B - $SET_A) / $SET_A * 100" | bc)
    echo ""
    echo "Performance gain (SET): ${GAIN}%"
fi

# Restaurer config A
redis-cli CONFIG SET save "900 1 300 10 60 10000"
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec
```

---

## 📊 Visualisation des résultats

### Générer des graphiques

```python
#!/usr/bin/env python3
"""
Génération de graphiques de benchmark
"""
import matplotlib.pyplot as plt
import pandas as pd
from pathlib import Path

def plot_throughput_comparison(csv_file='benchmark_summary.csv'):
    """Graphique de comparaison de throughput"""
    df = pd.read_csv(csv_file)

    # Trier par RPS
    df = df.sort_values('rps', ascending=True)

    # Créer le graphique
    fig, ax = plt.subplots(figsize=(12, 8))

    bars = ax.barh(df['test_name'], df['rps'])

    # Colorer selon la performance
    colors = ['red' if x < 50000 else 'yellow' if x < 100000 else 'green'
              for x in df['rps']]
    for bar, color in zip(bars, colors):
        bar.set_color(color)

    ax.set_xlabel('Requests per Second')
    ax.set_title('Redis Benchmark - Throughput Comparison')
    ax.grid(axis='x', alpha=0.3)

    # Ajouter les valeurs
    for i, (idx, row) in enumerate(df.iterrows()):
        ax.text(row['rps'], i, f"  {row['rps']:,.0f}",
                va='center', fontsize=8)

    plt.tight_layout()
    plt.savefig('benchmark_throughput.png', dpi=300)
    print("Saved: benchmark_throughput.png")

def plot_latency_percentiles(csv_file='benchmark_summary.csv'):
    """Graphique des percentiles de latence"""
    df = pd.read_csv(csv_file)

    fig, ax = plt.subplots(figsize=(12, 8))

    x = range(len(df))
    width = 0.25

    ax.bar([i - width for i in x], df['p50'], width, label='P50', alpha=0.8)
    ax.bar(x, df['p95'], width, label='P95', alpha=0.8)
    ax.bar([i + width for i in x], df['p99'], width, label='P99', alpha=0.8)

    ax.set_xlabel('Test')
    ax.set_ylabel('Latency (ms)')
    ax.set_title('Redis Benchmark - Latency Percentiles')
    ax.set_xticks(x)
    ax.set_xticklabels(df['test_name'], rotation=45, ha='right')
    ax.legend()
    ax.grid(axis='y', alpha=0.3)

    plt.tight_layout()
    plt.savefig('benchmark_latency.png', dpi=300)
    print("Saved: benchmark_latency.png")

def plot_throughput_vs_latency(csv_file='benchmark_summary.csv'):
    """Scatter plot throughput vs latency"""
    df = pd.read_csv(csv_file)

    fig, ax = plt.subplots(figsize=(12, 8))

    scatter = ax.scatter(df['rps'], df['p99'],
                        s=100, alpha=0.6, c=df['p99'],
                        cmap='RdYlGn_r')

    # Annoter les points
    for idx, row in df.iterrows():
        ax.annotate(row['test_name'],
                   (row['rps'], row['p99']),
                   fontsize=8, alpha=0.7)

    ax.set_xlabel('Throughput (req/sec)')
    ax.set_ylabel('P99 Latency (ms)')
    ax.set_title('Redis Benchmark - Throughput vs Latency')
    ax.grid(alpha=0.3)

    plt.colorbar(scatter, label='P99 Latency (ms)')
    plt.tight_layout()
    plt.savefig('benchmark_scatter.png', dpi=300)
    print("Saved: benchmark_scatter.png")

if __name__ == "__main__":
    plot_throughput_comparison()
    plot_latency_percentiles()
    plot_throughput_vs_latency()
```

---

## 🎓 Best Practices

### Checklist de benchmarking

**Avant le benchmark** :
- [ ] Environnement stable (pas d'autres processus)
- [ ] Redis config connue et documentée
- [ ] Persistence désactivée (ou notée)
- [ ] Données initiales contrôlées
- [ ] Réseau stable
- [ ] Ressources système suffisantes

**Pendant le benchmark** :
- [ ] Tests répétés (minimum 3 fois)
- [ ] Warmup avant mesures
- [ ] Monitoring des ressources actif
- [ ] Logs sauvegardés
- [ ] Conditions documentées

**Après le benchmark** :
- [ ] Résultats archivés avec timestamp
- [ ] Analyse statistique effectuée
- [ ] Comparaison avec baseline
- [ ] Anomalies investiguées
- [ ] Configuration restaurée

### Règles d'or

```
1. BASELINE FIRST
   └─ Toujours établir une référence avant optimisation

2. STABLE ENVIRONMENT
   └─ Même hardware, même config, même données

3. MULTIPLE RUNS
   └─ Minimum 3 runs, utiliser la médiane

4. REALISTIC WORKLOAD
   └─ Tester des scénarios d'usage réel

5. DOCUMENT EVERYTHING
   └─ Config, conditions, résultats

6. MONITOR RESOURCES
   └─ CPU, RAM, Network, Disk

7. AVOID SYNTHETIC ONLY
   └─ redis-benchmark + tests applicatifs

8. PERCENTILES MATTER
   └─ P99 > Average (outliers importants)
```

### Pièges à éviter

```
❌ Benchmarker avec persistence active
   → Fausse les résultats de throughput

❌ Ne pas laisser de warmup
   → Premiers résultats faussés

❌ Ignorer les percentiles hauts
   → P99 peut révéler des problèmes

❌ Un seul run
   → Variabilité non capturée

❌ Environnement instable
   → Résultats non reproductibles

❌ Comparer avec config différentes
   → Comparaison invalide

❌ Ne regarder que le throughput
   → Latence peut être catastrophique

❌ Benchmarks synthétiques uniquement
   → Ne reflète pas l'usage réel
```

---

## 🎯 Points clés à retenir

1. **redis-benchmark** → Outil standard pour benchmarks
2. **Baseline obligatoire** → Référence avant optimisation
3. **Percentiles P95/P99** → Plus importants que moyenne
4. **Environnement stable** → Reproductibilité essentielle
5. **Multiple runs** → Minimum 3, utiliser médiane
6. **Scénarios réalistes** → Tester l'usage applicatif
7. **Monitor resources** → CPU, RAM, Network, Disk
8. **Documenter tout** → Config, conditions, résultats

---

**🚀 Section suivante** : [14.10 - Cas pratiques de troubleshooting](./10-cas-pratiques-troubleshooting.md)

⏭️ [Redis dans le Cloud et Conteneurs](/15-redis-cloud-conteneurs/README.md)
