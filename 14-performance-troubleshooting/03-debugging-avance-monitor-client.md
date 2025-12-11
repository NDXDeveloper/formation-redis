🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.3 - Debugging avancé : MONITOR, CLIENT LIST, CLIENT KILL

## 🎯 Objectifs de cette section

- Maîtriser les outils de debugging en temps réel de Redis
- Comprendre l'impact et les risques de MONITOR en production
- Analyser et gérer les connexions clients efficacement
- Identifier et résoudre les problèmes de connexions
- Développer des stratégies de debugging sécurisées

---

## ⚠️ Avertissement critique : MONITOR en production

### Le paradoxe de MONITOR

**MONITOR** est l'outil de debugging le plus puissant de Redis, mais aussi **le plus dangereux**.

```
┌─────────────────────────────────────────────────┐
│  MONITOR : TRÈS PUISSANT = TRÈS DANGEREUX       │
├─────────────────────────────────────────────────┤
│                                                 │
│  ✅ Avantages :                                 │
│     • Voir TOUTES les commandes en temps réel   │
│     • Debug instantané                          │
│     • Capture complète du trafic                │
│                                                 │
│  ❌ Dangers :                                   │
│     • Impact MAJEUR sur les performances        │
│     • Peut réduire le throughput de 50%+        │
│     • Augmente la latence                       │
│     • Consomme énormément de bande passante     │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Règle d'or** : MONITOR est un **outil de dernier recours** en production.

### Impact sur les performances

Tests comparatifs :

```
Sans MONITOR :
├─ Throughput : 100,000 ops/sec
├─ Latency P99 : 0.5ms
└─ CPU : 40%

Avec MONITOR actif (1 client) :
├─ Throughput : 50,000 ops/sec  (-50%)
├─ Latency P99 : 2.5ms          (+400%)
└─ CPU : 75%                     (+87.5%)

Avec MONITOR actif (3 clients) :
├─ Throughput : 20,000 ops/sec  (-80%)
├─ Latency P99 : 8ms            (+1500%)
└─ CPU : 95%                     (+137.5%)
```

**Explication** : Chaque commande doit être :
1. Exécutée normalement
2. Formatée pour MONITOR
3. Envoyée à TOUS les clients MONITOR
4. Ce processus est **synchrone** et **bloquant**

---

## 🔍 MONITOR : Le debugger en temps réel

### Qu'est-ce que MONITOR ?

MONITOR transforme un client Redis en **espion passif** qui reçoit toutes les commandes traitées par le serveur.

### Utilisation de base

```bash
# Démarrer le monitoring
redis-cli MONITOR

# Output en temps réel :
1702293847.123456 [0 127.0.0.1:43501] "SET" "user:123" "John"
1702293847.234567 [0 127.0.0.1:43502] "GET" "user:123"
1702293847.345678 [0 192.168.1.10:52301] "INCR" "counter:visits"
1702293847.456789 [0 127.0.0.1:43501] "DEL" "temp:session:abc"
...
```

**Format de sortie** :
```
timestamp [db client_address:port] "command" "arg1" "arg2" ...
```

### Utilisation sécurisée en production

#### Règle #1 : Limiter la durée

```bash
# ✅ Maximum 10 secondes
timeout 10 redis-cli MONITOR > monitor_capture.txt

# ✅ Ou avec limite de lignes
redis-cli MONITOR | head -n 1000 > monitor_capture.txt

# ❌ JAMAIS sans limite
redis-cli MONITOR  # DANGER : peut tourner indéfiniment
```

#### Règle #2 : Filtrer en temps réel

```bash
# ✅ Capturer uniquement les commandes spécifiques
redis-cli MONITOR | grep "KEYS" > keys_usage.txt

# ✅ Filtrer par pattern de clé
redis-cli MONITOR | grep "user:" > user_operations.txt

# ✅ Exclure les commandes de monitoring
redis-cli MONITOR | grep -v "INFO\|PING" > clean_monitor.txt
```

#### Règle #3 : Rediriger vers un fichier

```bash
# ✅ Ne jamais afficher dans le terminal en production
redis-cli MONITOR > /var/log/redis/monitor_$(date +%s).log &
MONITOR_PID=$!

# Attendre 5 secondes
sleep 5

# Tuer le process MONITOR
kill $MONITOR_PID
```

#### Règle #4 : Monitorer l'impact

```bash
#!/bin/bash
# Script de monitoring sécurisé avec vérification d'impact

echo "Starting MONITOR with safety checks..."

# Capturer les métriques avant
BEFORE_OPS=$(redis-cli INFO stats | grep instantaneous_ops_per_sec | cut -d: -f2)
echo "Ops/sec before MONITOR: $BEFORE_OPS"

# Démarrer MONITOR avec timeout
timeout 5 redis-cli MONITOR > monitor_output.txt &
MONITOR_PID=$!

# Attendre 2 secondes pour stabilisation
sleep 2

# Vérifier l'impact
DURING_OPS=$(redis-cli INFO stats | grep instantaneous_ops_per_sec | cut -d: -f2)
echo "Ops/sec during MONITOR: $DURING_OPS"

# Calculer la dégradation
DEGRADATION=$(echo "scale=2; (($BEFORE_OPS - $DURING_OPS) / $BEFORE_OPS) * 100" | bc)

if (( $(echo "$DEGRADATION > 30" | bc -l) )); then
    echo "WARNING: Performance degradation > 30% ($DEGRADATION%). Stopping MONITOR."
    kill $MONITOR_PID
    exit 1
fi

# Continuer le monitoring
wait $MONITOR_PID

# Métriques après
AFTER_OPS=$(redis-cli INFO stats | grep instantaneous_ops_per_sec | cut -d: -f2)
echo "Ops/sec after MONITOR: $AFTER_OPS"

echo "MONITOR completed. Output saved to monitor_output.txt"
```

### Analyse des captures MONITOR

#### Script d'analyse des patterns

```python
#!/usr/bin/env python3
"""
Analyse des captures MONITOR pour identifier les patterns
"""
import re
from collections import Counter, defaultdict
import sys

def parse_monitor_line(line):
    """Parse une ligne MONITOR"""
    # Format : timestamp [db ip:port] "command" "arg1" "arg2"...
    pattern = r'(\d+\.\d+) \[(\d+) ([\d\.:]+)\] (.+)'
    match = re.match(pattern, line)

    if not match:
        return None

    timestamp, db, client, command_str = match.groups()

    # Parser les commandes (entre quotes)
    commands = re.findall(r'"([^"]*)"', command_str)

    if not commands:
        return None

    return {
        'timestamp': float(timestamp),
        'db': int(db),
        'client': client,
        'command': commands[0] if commands else 'UNKNOWN',
        'args': commands[1:] if len(commands) > 1 else []
    }

def analyze_monitor_capture(filename):
    """Analyse complète d'une capture MONITOR"""

    print("=== MONITOR CAPTURE ANALYSIS ===\n")

    commands_counter = Counter()
    clients = defaultdict(list)
    key_patterns = Counter()
    errors = []

    total_lines = 0
    parsed_lines = 0

    with open(filename, 'r') as f:
        for line in f:
            total_lines += 1
            parsed = parse_monitor_line(line.strip())

            if not parsed:
                errors.append(line.strip())
                continue

            parsed_lines += 1

            # Compter les commandes
            commands_counter[parsed['command']] += 1

            # Tracker par client
            clients[parsed['client']].append(parsed['command'])

            # Extraire les patterns de clés
            if parsed['args']:
                key = parsed['args'][0]
                if ':' in key:
                    pattern = key.split(':')[0] + ':*'
                    key_patterns[pattern] += 1

    # Résultats
    print(f"Total lines: {total_lines}")
    print(f"Parsed lines: {parsed_lines}")
    print(f"Errors: {len(errors)}\n")

    print("--- Top 20 Commands ---")
    for cmd, count in commands_counter.most_common(20):
        percentage = (count / parsed_lines) * 100
        print(f"{cmd:20} {count:8} ({percentage:5.2f}%)")

    print("\n--- Top 10 Clients by Activity ---")
    clients_sorted = sorted(clients.items(), key=lambda x: len(x[1]), reverse=True)
    for client, cmds in clients_sorted[:10]:
        print(f"{client:30} {len(cmds):8} commands")

    print("\n--- Top 10 Key Patterns ---")
    for pattern, count in key_patterns.most_common(10):
        percentage = (count / parsed_lines) * 100
        print(f"{pattern:30} {count:8} ({percentage:5.2f}%)")

    # Détection de problèmes
    print("\n--- ISSUES DETECTED ---\n")

    # Commandes dangereuses
    dangerous = ['KEYS', 'FLUSHDB', 'FLUSHALL', 'CONFIG']
    found_dangerous = {cmd: commands_counter[cmd] for cmd in dangerous if commands_counter[cmd] > 0}

    if found_dangerous:
        print("⚠️  DANGEROUS COMMANDS DETECTED:")
        for cmd, count in found_dangerous.items():
            print(f"   {cmd}: {count} occurrences")
        print()

    # Client très actif (> 50% des commandes)
    for client, cmds in clients_sorted[:3]:
        percentage = (len(cmds) / parsed_lines) * 100
        if percentage > 50:
            print(f"⚠️  HIGH ACTIVITY CLIENT: {client}")
            print(f"   Responsible for {percentage:.2f}% of all commands")
            # Top commandes de ce client
            client_cmds = Counter(cmds)
            print("   Top commands:")
            for cmd, count in client_cmds.most_common(5):
                print(f"      {cmd}: {count}")
            print()

    # Commandes O(N)
    on_commands = ['SMEMBERS', 'HGETALL', 'LRANGE', 'ZRANGE', 'SORT']
    found_on = {cmd: commands_counter[cmd] for cmd in on_commands if commands_counter[cmd] > 0}

    if found_on:
        print("⚠️  O(N) COMMANDS DETECTED:")
        for cmd, count in found_on.items():
            print(f"   {cmd}: {count} occurrences")
        print("   Consider using SCAN variants (SSCAN, HSCAN, etc.)")
        print()

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python3 analyze_monitor.py <monitor_capture.txt>")
        sys.exit(1)

    analyze_monitor_capture(sys.argv[1])
```

### Cas d'usage de MONITOR

#### 1. Debug d'une commande spécifique

```bash
# Problème : Une application génère des erreurs mais on ne sait pas quelle commande
redis-cli MONITOR | grep "192.168.1.50" | head -n 100
```

#### 2. Identifier un client qui utilise KEYS *

```bash
# Capturer uniquement les KEYS
timeout 30 redis-cli MONITOR | grep '"KEYS"' > keys_usage.txt

# Analyser
cat keys_usage.txt
# 1702293847.123456 [0 192.168.1.100:43501] "KEYS" "*"
# → Client 192.168.1.100:43501 est le coupable
```

#### 3. Profiler une application spécifique

```bash
# Filtrer par IP de l'application
redis-cli MONITOR | grep "10.0.1.25" | tee app_profiling.txt
```

#### 4. Détecter des patterns d'accès anormaux

```bash
# Capturer pendant 10 secondes
timeout 10 redis-cli MONITOR > sample.txt

# Analyser
python3 analyze_monitor.py sample.txt
```

---

## 👥 CLIENT LIST : Gestion des connexions

### Qu'est-ce que CLIENT LIST ?

**CLIENT LIST** affiche tous les clients actuellement connectés avec leurs caractéristiques.

### Utilisation de base

```bash
redis-cli CLIENT LIST
```

**Sortie exemple** :

```
id=3 addr=127.0.0.1:43501 fd=8 name= age=30 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=26 qbuf-free=32742 obl=0 oll=0 omem=0 events=r cmd=get user=default
id=4 addr=192.168.1.100:52301 fd=9 name=webapp age=120 idle=5 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=set user=default
id=5 addr=10.0.1.50:33201 fd=10 name= age=3600 idle=3600 flags=N db=1 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=ping user=default
```

### Comprendre les champs

| Champ | Description | Valeurs typiques |
|-------|-------------|------------------|
| **id** | ID unique du client | Entier incrémental |
| **addr** | Adresse IP:port | 192.168.1.100:43501 |
| **fd** | File descriptor | Entier |
| **name** | Nom du client (optionnel) | webapp, worker, cache |
| **age** | Durée de connexion (secondes) | Variable |
| **idle** | Temps depuis dernière commande (sec) | 0 = actif, >0 = idle |
| **flags** | Flags du client | N, S, M, P, etc. |
| **db** | Database actuelle | 0-15 |
| **sub** | Nombre de subscriptions | 0+ |
| **psub** | Nombre de pattern subscriptions | 0+ |
| **multi** | État transaction | -1 = aucune, >=0 = en cours |
| **qbuf** | Query buffer size | Bytes |
| **qbuf-free** | Query buffer free space | Bytes |
| **obl** | Output buffer length | Bytes |
| **oll** | Output list length | Éléments |
| **omem** | Output buffer memory | Bytes |
| **events** | File descriptor events | r = readable, w = writable |
| **cmd** | Dernière commande | get, set, etc. |
| **user** | Utilisateur ACL | default, app, admin |

### Interprétation des flags

| Flag | Signification | Action |
|------|---------------|--------|
| **N** | Normal client | Aucune |
| **M** | Master (replication) | Monitoring |
| **S** | Slave (replica) | Monitoring |
| **O** | Client en MONITOR | ⚠️ Impact performance |
| **x** | Multi/exec transaction | Normal |
| **b** | Blocked (BLPOP, etc.) | Normal |
| **i** | Waiting for VM I/O | Rare |
| **d** | Watched keys (WATCH) | Normal |
| **u** | Unblocked | Normal |
| **A** | Close ASAP | Client problématique |
| **c** | Connection to be closed | En cours de fermeture |

### Commandes CLIENT avancées

#### CLIENT LIST avec filtres (Redis 6.2+)

```bash
# Filtrer par type
redis-cli CLIENT LIST TYPE normal
redis-cli CLIENT LIST TYPE master
redis-cli CLIENT LIST TYPE replica
redis-cli CLIENT LIST TYPE pubsub

# Filtrer par ID
redis-cli CLIENT LIST ID 3 5 7
```

#### CLIENT INFO

Informations sur le client actuel :

```bash
redis-cli CLIENT INFO
```

#### CLIENT SETNAME / CLIENT GETNAME

Nommer les connexions pour faciliter le debugging :

```bash
# Côté application (Python exemple)
import redis
r = redis.Redis(host='localhost', port=6379)
r.client_setname('webapp-worker-1')

# Côté admin
redis-cli CLIENT LIST | grep webapp-worker
```

**Best practice** : Toujours nommer vos connexions en production !

```python
# Pattern de nommage recommandé
# {app_name}-{component}-{instance_id}

# Exemples :
"api-gateway-01"
"worker-queue-03"
"cache-warmer-02"
"analytics-processor-01"
```

---

## 🔪 CLIENT KILL : Terminer des connexions

### Quand utiliser CLIENT KILL ?

**Situations légitimes** :
- Client exécutant MONITOR en production (oublié)
- Client avec une commande bloquante infinie
- Connection leak (client zombie)
- Client malveillant ou compromis
- Test de résilience applicative

### Syntaxe

```bash
# Kill par ID (recommandé)
CLIENT KILL ID client-id

# Kill par adresse IP:port
CLIENT KILL ADDR ip:port

# Kill avec filtres (Redis 2.8.12+)
CLIENT KILL TYPE normal|master|slave|pubsub
CLIENT KILL USER username
CLIENT KILL SKIPME yes|no  # Inclure/exclure le client actuel
```

### Exemples d'utilisation

#### 1. Tuer un client MONITOR oublié

```bash
# Identifier le client MONITOR
redis-cli CLIENT LIST | grep "flags=O"
# id=42 addr=192.168.1.50:43501 fd=10 name=debug age=3600 idle=0 flags=O ...

# Tuer ce client
redis-cli CLIENT KILL ID 42
```

#### 2. Tuer tous les clients idle > 1 heure

```bash
#!/bin/bash
# kill-idle-clients.sh

IDLE_THRESHOLD=3600  # 1 heure

redis-cli CLIENT LIST | while read line; do
    # Extraire ID et idle
    id=$(echo "$line" | grep -oP 'id=\K\d+')
    idle=$(echo "$line" | grep -oP 'idle=\K\d+')

    if [ "$idle" -gt "$IDLE_THRESHOLD" ]; then
        echo "Killing idle client ID $id (idle: ${idle}s)"
        redis-cli CLIENT KILL ID $id
    fi
done
```

#### 3. Tuer les clients d'une application spécifique

```bash
# Par nom de client
redis-cli CLIENT LIST | grep "name=old-app" | \
  grep -oP 'id=\K\d+' | \
  xargs -I {} redis-cli CLIENT KILL ID {}

# Par IP
redis-cli CLIENT KILL TYPE normal ADDR 192.168.1.100
```

#### 4. Tuer tous les clients normaux (maintenance)

```bash
# ⚠️ Attention : Va déconnecter toutes les applications !
redis-cli CLIENT KILL TYPE normal SKIPME yes
```

### Impact de CLIENT KILL

**Ce qui se passe** :
1. Le client reçoit une erreur de connexion
2. La connexion est fermée immédiatement
3. Commandes en cours sont annulées
4. Le client doit gérer la reconnexion

**Côté application** :

```python
import redis
from redis.exceptions import ConnectionError
import time

def resilient_redis_operation():
    """Exemple de gestion de reconnexion"""
    max_retries = 3
    retry_delay = 1

    for attempt in range(max_retries):
        try:
            r = redis.Redis(host='localhost', port=6379)
            result = r.get('mykey')
            return result
        except ConnectionError as e:
            if attempt < max_retries - 1:
                print(f"Connection lost, retrying in {retry_delay}s...")
                time.sleep(retry_delay)
                retry_delay *= 2  # Exponential backoff
            else:
                raise
```

---

## 🎭 Scénarios de troubleshooting avancés

### Scénario 1 : Latence élevée soudaine

**Symptômes** :
- Applications signalent des timeouts
- `SLOWLOG` ne montre rien de spécial
- CPU normal

**Investigation** :

```bash
# 1. Vérifier les clients connectés
redis-cli CLIENT LIST > clients_snapshot.txt

# 2. Analyser les clients suspects
cat clients_snapshot.txt | awk '{print $7, $9, $13}' | sort -k2 -n -r | head -20
# Format : idle=X qbuf=Y omem=Z

# 3. Identifier les gros buffers
cat clients_snapshot.txt | awk '{
    match($0, /id=([0-9]+)/, id);
    match($0, /omem=([0-9]+)/, omem);
    if (omem[1] > 1048576) {  # > 1MB
        print "Client ID:", id[1], "Buffer:", omem[1]/1048576, "MB"
    }
}'
```

**Cause potentielle** : Client avec gros output buffer (slow consumer).

**Solution** :

```bash
# Identifier et killer le client
redis-cli CLIENT KILL ID <client-id>

# Configurer des limites
redis-cli CONFIG SET client-output-buffer-limit "normal 0 0 0"
redis-cli CONFIG SET client-output-buffer-limit "pubsub 32mb 8mb 60"
```

### Scénario 2 : Connection leak

**Symptômes** :
- Nombre de clients augmente continuellement
- Applications fonctionnent mais accumulent les connexions
- Risque d'atteindre `maxclients`

**Investigation** :

```bash
#!/bin/bash
# monitor-connections.sh

while true; do
    TIMESTAMP=$(date +%s)
    CLIENTS=$(redis-cli CLIENT LIST | wc -l)

    echo "$TIMESTAMP,$CLIENTS" >> connections_log.csv

    # Grouper par application
    redis-cli CLIENT LIST | \
      grep -oP 'name=\K[^ ]+' | \
      sort | uniq -c | sort -rn > connections_by_app_${TIMESTAMP}.txt

    sleep 60
done
```

**Analyse** :

```python
#!/usr/bin/env python3
import pandas as pd
import matplotlib.pyplot as plt

# Charger les données
df = pd.read_csv('connections_log.csv', names=['timestamp', 'clients'])

# Détecter les leaks (augmentation continue)
df['growth_rate'] = df['clients'].diff()

if df['growth_rate'].mean() > 0.1:  # Augmentation moyenne > 0.1 client/min
    print("⚠️  CONNECTION LEAK DETECTED")
    print(f"Average growth: {df['growth_rate'].mean():.2f} clients/min")
    print(f"Total growth: {df['clients'].iloc[-1] - df['clients'].iloc[0]} clients")

# Visualiser
plt.figure(figsize=(12, 6))
plt.plot(df['timestamp'], df['clients'])
plt.xlabel('Time')
plt.ylabel('Connected Clients')
plt.title('Redis Connections Over Time')
plt.savefig('connections_trend.png')
```

**Solution** :

```bash
# Identifier l'application responsable
cat connections_by_app_* | grep -v "^$" | sort | uniq -c | sort -rn

# Killer les connexions idle de cette app
redis-cli CLIENT LIST | grep "name=leaky-app" | \
  grep -oP 'id=\K\d+' | \
  xargs -I {} redis-cli CLIENT KILL ID {}
```

### Scénario 3 : Client bloqué sur BLPOP

**Symptômes** :
- Client reste connecté mais ne répond plus
- Consomme une connexion inutilement
- `flags=b` dans CLIENT LIST

**Investigation** :

```bash
# Identifier les clients bloqués
redis-cli CLIENT LIST | grep "flags=.*b"

# Exemple de sortie :
# id=10 addr=192.168.1.100:52301 fd=12 name=worker-1 age=7200 idle=7200 flags=b db=0 ...
```

**Causes possibles** :
1. Queue vide, client attend indéfiniment
2. Timeout BLPOP trop long
3. Producer en panne

**Solution** :

```bash
# Option 1 : Vérifier si la queue existe et a des éléments
redis-cli LLEN queue:tasks
# Si 0, le client attend pour rien

# Option 2 : Débloquer en insérant un message
redis-cli LPUSH queue:tasks "dummy"

# Option 3 : Killer le client pour qu'il se reconnecte
redis-cli CLIENT KILL ID 10

# Recommandation long terme : Utiliser un timeout
# Au lieu de : BLPOP queue:tasks 0  (infini)
# Faire : BLPOP queue:tasks 30  (30 secondes)
```

### Scénario 4 : Application utilise KEYS en production

**Symptômes** :
- Pics de latence réguliers
- `SLOWLOG` montre des `KEYS *`
- CPU spikes

**Investigation avec MONITOR** :

```bash
# Capture courte pour identifier le coupable
timeout 10 redis-cli MONITOR | grep '"KEYS"' | tee keys_culprit.txt

# Sortie exemple :
# 1702293847.123456 [0 192.168.1.100:43501] "KEYS" "user:*"
# 1702293847.456789 [0 192.168.1.100:43501] "KEYS" "session:*"
```

**Analyse** :

```bash
# Identifier l'application via CLIENT LIST
redis-cli CLIENT LIST | grep "192.168.1.100:43501"
# id=15 addr=192.168.1.100:43501 fd=18 name=legacy-app age=300 ...
```

**Solution immédiate** :

```bash
# Tuer le client temporairement
redis-cli CLIENT KILL ADDR 192.168.1.100:43501

# Bloquer cette IP temporairement (iptables)
sudo iptables -A INPUT -s 192.168.1.100 -p tcp --dport 6379 -j DROP
```

**Solution long terme** :
- Corriger l'application pour utiliser `SCAN`
- Renommer la commande KEYS dans redis.conf :

```conf
rename-command KEYS ""  # Désactiver complètement
# ou
rename-command KEYS "KEYS_DANGEROUS_DO_NOT_USE"
```

---

## 📊 Monitoring continu des clients

### Script de monitoring automatisé

```python
#!/usr/bin/env python3
"""
Monitoring continu des clients Redis avec alertes
"""
import redis
import time
import json
from datetime import datetime

class RedisClientMonitor:
    def __init__(self, host='localhost', port=6379):
        self.r = redis.Redis(host=host, port=port, decode_responses=True)
        self.alerts = []

    def get_clients_stats(self):
        """Récupère les statistiques des clients"""
        clients = self.r.client_list()

        stats = {
            'total': len(clients),
            'by_type': {},
            'idle_clients': [],
            'high_buffer_clients': [],
            'blocked_clients': [],
            'monitor_clients': []
        }

        for client in clients:
            # Par type
            client_type = 'normal'
            if 'M' in client['flags']:
                client_type = 'master'
            elif 'S' in client['flags']:
                client_type = 'slave'
            elif 'P' in client['flags']:
                client_type = 'pubsub'

            stats['by_type'][client_type] = stats['by_type'].get(client_type, 0) + 1

            # Clients idle > 1 heure
            if client['idle'] > 3600:
                stats['idle_clients'].append({
                    'id': client['id'],
                    'addr': client['addr'],
                    'idle': client['idle'],
                    'name': client.get('name', '')
                })

            # Gros buffers (> 10MB)
            if client.get('omem', 0) > 10 * 1024 * 1024:
                stats['high_buffer_clients'].append({
                    'id': client['id'],
                    'addr': client['addr'],
                    'omem': client['omem'],
                    'name': client.get('name', '')
                })

            # Clients bloqués
            if 'b' in client['flags']:
                stats['blocked_clients'].append({
                    'id': client['id'],
                    'addr': client['addr'],
                    'age': client['age'],
                    'name': client.get('name', '')
                })

            # Clients MONITOR
            if 'O' in client['flags']:
                stats['monitor_clients'].append({
                    'id': client['id'],
                    'addr': client['addr'],
                    'age': client['age']
                })

        return stats

    def check_alerts(self, stats):
        """Vérifie et génère les alertes"""
        alerts = []

        # Alerte : Trop de clients
        max_clients = int(self.r.config_get('maxclients')['maxclients'])
        if stats['total'] > max_clients * 0.9:
            alerts.append({
                'severity': 'warning',
                'message': f"High client count: {stats['total']}/{max_clients} (90%+)"
            })

        # Alerte : Clients MONITOR en production
        if stats['monitor_clients']:
            alerts.append({
                'severity': 'critical',
                'message': f"MONITOR clients active: {len(stats['monitor_clients'])}",
                'details': stats['monitor_clients']
            })

        # Alerte : Gros buffers
        if stats['high_buffer_clients']:
            alerts.append({
                'severity': 'warning',
                'message': f"High buffer clients: {len(stats['high_buffer_clients'])}",
                'details': stats['high_buffer_clients']
            })

        # Alerte : Beaucoup de clients idle
        if len(stats['idle_clients']) > 50:
            alerts.append({
                'severity': 'info',
                'message': f"Many idle clients: {len(stats['idle_clients'])}",
                'suggestion': "Consider implementing connection timeout"
            })

        return alerts

    def run(self, interval=60):
        """Boucle de monitoring"""
        print("Starting Redis Client Monitor...")
        print(f"Monitoring every {interval} seconds")
        print("-" * 60)

        while True:
            try:
                stats = self.get_clients_stats()
                alerts = self.check_alerts(stats)

                # Afficher les statistiques
                timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
                print(f"\n[{timestamp}]")
                print(f"Total clients: {stats['total']}")
                print(f"By type: {stats['by_type']}")
                print(f"Idle clients (>1h): {len(stats['idle_clients'])}")
                print(f"Blocked clients: {len(stats['blocked_clients'])}")
                print(f"High buffer clients: {len(stats['high_buffer_clients'])}")

                # Afficher les alertes
                if alerts:
                    print("\n⚠️  ALERTS:")
                    for alert in alerts:
                        severity = alert['severity'].upper()
                        print(f"   [{severity}] {alert['message']}")
                        if 'details' in alert:
                            for detail in alert['details'][:3]:  # Max 3
                                print(f"      - ID: {detail['id']}, Addr: {detail['addr']}")

                time.sleep(interval)

            except KeyboardInterrupt:
                print("\nMonitoring stopped.")
                break
            except Exception as e:
                print(f"Error: {e}")
                time.sleep(interval)

if __name__ == "__main__":
    monitor = RedisClientMonitor()
    monitor.run(interval=60)
```

### Métriques Prometheus

```python
# prometheus_redis_clients.py
from prometheus_client import start_http_server, Gauge
import redis
import time

# Définir les métriques
redis_clients_total = Gauge('redis_clients_total', 'Total number of connected clients')
redis_clients_idle = Gauge('redis_clients_idle', 'Number of idle clients', ['threshold'])
redis_clients_blocked = Gauge('redis_clients_blocked', 'Number of blocked clients')
redis_clients_monitor = Gauge('redis_clients_monitor', 'Number of MONITOR clients')

def collect_metrics():
    r = redis.Redis(host='localhost', port=6379)

    clients = r.client_list()

    redis_clients_total.set(len(clients))

    idle_1h = sum(1 for c in clients if c['idle'] > 3600)
    idle_10m = sum(1 for c in clients if c['idle'] > 600)

    redis_clients_idle.labels(threshold='1h').set(idle_1h)
    redis_clients_idle.labels(threshold='10m').set(idle_10m)

    blocked = sum(1 for c in clients if 'b' in c['flags'])
    redis_clients_blocked.set(blocked)

    monitor = sum(1 for c in clients if 'O' in c['flags'])
    redis_clients_monitor.set(monitor)

if __name__ == '__main__':
    start_http_server(8000)
    while True:
        collect_metrics()
        time.sleep(15)
```

---

## 🎓 Best Practices

### MONITOR

✅ **DO**
- Utiliser seulement en dernier recours
- Limiter à < 30 secondes
- Rediriger vers un fichier
- Filtrer en temps réel
- Monitorer l'impact
- Analyser offline

❌ **DON'T**
- Ne jamais laisser tourner indéfiniment
- Ne pas utiliser sur des instances à haute charge
- Ne jamais avoir plusieurs clients MONITOR simultanés
- Ne pas afficher dans le terminal (trop lent)

### CLIENT LIST

✅ **DO**
- Monitorer régulièrement
- Nommer tous vos clients
- Tracker les tendances
- Configurer des alertes
- Documenter les patterns normaux

❌ **DON'T**
- Ne pas ignorer les clients idle
- Ne pas laisser des MONITOR actifs
- Ne pas négliger les gros buffers

### CLIENT KILL

✅ **DO**
- Toujours vérifier avant de kill
- Logger les actions
- Comprendre l'impact sur l'application
- Avoir une stratégie de reconnexion côté app
- Utiliser pour les clients clairement problématiques

❌ **DON'T**
- Ne jamais kill sans analyse
- Ne pas killer des replicas en production
- Ne pas utiliser TYPE normal sans SKIPME
- Ne pas oublier de corriger la cause racine

---

## 🔗 Checklist de debugging

### En cas de problème de performance

- [ ] Vérifier `CLIENT LIST` pour clients suspects
- [ ] Identifier les clients avec gros buffers (`omem` > 10MB)
- [ ] Vérifier les clients idle > 1 heure
- [ ] Chercher des flags anormaux (O, A)
- [ ] Si nécessaire : capture MONITOR courte (< 10s)
- [ ] Analyser les patterns de commandes
- [ ] Identifier le client problématique
- [ ] Killer si nécessaire
- [ ] Corriger la cause racine dans l'application

### En cas de connection leak

- [ ] Monitorer le nombre de clients dans le temps
- [ ] Grouper par nom/application
- [ ] Identifier la source de la croissance
- [ ] Vérifier le code de l'application (pool config)
- [ ] Implémenter des timeouts de connexion
- [ ] Killer les connexions idle
- [ ] Mettre en place un monitoring continu

### Maintenance préventive

- [ ] Nommer toutes les connexions applicatives
- [ ] Configurer `timeout` dans redis.conf
- [ ] Limiter `maxclients` de manière appropriée
- [ ] Monitorer les métriques de clients
- [ ] Auditer régulièrement CLIENT LIST
- [ ] Former les équipes dev sur les bonnes pratiques
- [ ] Documenter les patterns normaux

---

## 📚 Ressources complémentaires

### Documentation officielle
- [Redis MONITOR](https://redis.io/commands/monitor/)
- [Redis CLIENT commands](https://redis.io/commands/?group=connection)
- [Client handling](https://redis.io/docs/manual/admin/)

### Outils
- **redis-cli** : Outils natifs MONITOR, CLIENT LIST
- **Redis Insight** : Visualisation des clients connectés
- Scripts de cette section (Python, Bash)

### Patterns de résilience applicative
- Connection pooling
- Automatic reconnection
- Exponential backoff
- Circuit breaker pattern

---

## 🎯 Points clés à retenir

1. **MONITOR = dernier recours** → Impact majeur sur les performances
2. **Toujours limiter MONITOR** → < 30 secondes maximum
3. **CLIENT LIST = premier outil** → Analyser avant MONITOR
4. **Nommer vos clients** → Facilite énormément le debugging
5. **CLIENT KILL avec précaution** → Comprendre l'impact
6. **Monitoring continu** → Détecter les problèmes avant la crise
7. **Résilience applicative** → Gérer les déconnexions gracieusement
8. **Documentation** → Noter les patterns normaux vs anormaux

---

**🚀 Section suivante** : [14.4 - Problèmes de latence : Causes et solutions](./04-problemes-latence-causes-solutions.md)

⏭️ [Problèmes de latence : Causes et solutions](/14-performance-troubleshooting/04-problemes-latence-causes-solutions.md)
