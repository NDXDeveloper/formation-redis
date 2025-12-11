🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.6 - Bonnes pratiques Linux : THP, Swap, Overcommit memory

## Introduction

Redis est extrêmement sensible à la configuration du système d'exploitation Linux. Des paramètres kernel mal configurés peuvent causer :

- 🔴 **Latences imprévisibles** (THP)
- 🔴 **Freeze complet** de Redis (Swap)
- 🔴 **Crashes OOM Killer** (Overcommit)
- 🔴 **Performances dégradées** (Limites système)

> **⚠️ Fait critique :** Un Redis parfaitement configuré peut avoir des performances catastrophiques si le kernel Linux n'est pas optimisé. Cette section est **OBLIGATOIRE** pour tout déploiement en production.

---

## Transparent Huge Pages (THP)

### 1. Comprendre le problème THP

#### Qu'est-ce que THP ?

```
┌─────────────────────────────────────────────────────────────────┐
│                    GESTION MÉMOIRE LINUX                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pages mémoire standard:                                        │
│  ├── Taille: 4 KB                                               │
│  ├── Nombreuses pages → Beaucoup d'entrées TLB                  │
│  └── TLB misses fréquents → Overhead                            │
│                                                                 │
│  Huge Pages (2 MB ou 1 GB):                                     │
│  ├── Moins de pages pour même mémoire                           │
│  ├── Moins d'entrées TLB                                        │
│  ├── Moins de TLB misses                                        │
│  └── Performances améliorées (théoriquement)                    │
│                                                                 │
│  Transparent Huge Pages (THP):                                  │
│  ├── Kernel essaie automatiquement d'utiliser huge pages        │
│  ├── "Transparent" = sans intervention application              │
│  ├── Défragmentation en background (khugepaged)                 │
│  └── PROBLÈME pour Redis!                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Pourquoi THP est catastrophique pour Redis

```
┌─────────────────────────────────────────────────────────────────┐
│              PROBLÈMES THP AVEC REDIS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Copy-on-Write (COW) amplifié:                               │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ Sans THP: Modification 1 clé = Copy 4 KB               │  │
│     │ Avec THP: Modification 1 clé = Copy 2 MB !             │  │
│     │ Impact: 500x plus de mémoire copiée                    │  │
│     └────────────────────────────────────────────────────────┘  │
│                                                                 │
│  2. BGSAVE/BGREWRITEAOF bloquant:                               │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ Fork process → COW sur modifications                   │  │
│     │ THP → Copies massives de 2 MB                          │  │
│     │ Latence: +100ms à +10 secondes!                        │  │
│     └────────────────────────────────────────────────────────┘  │
│                                                                 │
│  3. Défragmentation (khugepaged):                               │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ Process kernel qui compacte les pages                  │  │
│     │ Peut bloquer Redis pendant compaction                  │  │
│     │ Latence imprévisible: spikes aléatoires                │  │
│     └────────────────────────────────────────────────────────┘  │
│                                                                 │
│  4. Mémoire fragmentée:                                         │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ Redis alloue/libère fréquemment                        │  │
│     │ THP essaie de créer huge pages contiguës               │  │
│     │ Résultat: Fragmentation accrue, OOM prématuré          │  │
│     └────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Impact mesurable

```bash
# Test avec THP activé
echo always > /sys/kernel/mm/transparent_hugepage/enabled
redis-benchmark -t set,get -n 1000000 -d 100
# Résultats typiques:
# SET: 85,000 ops/sec
# GET: 90,000 ops/sec
# p99 latency: 1.5ms avec spikes à 100ms+

# Test avec THP désactivé
echo never > /sys/kernel/mm/transparent_hugepage/enabled
redis-benchmark -t set,get -n 1000000 -d 100
# Résultats typiques:
# SET: 95,000 ops/sec (+12%)
# GET: 100,000 ops/sec (+11%)
# p99 latency: 0.8ms, pas de spikes

# Impact latence lors de BGSAVE avec THP:
# Avec THP: +5-10 secondes de latence
# Sans THP: +50-200ms de latence
```

### 2. Vérifier l'état de THP

```bash
#!/bin/bash
# check-thp-status.sh

echo "=== TRANSPARENT HUGE PAGES STATUS ==="

# Vérifier état actuel
echo "1. Current THP status:"
cat /sys/kernel/mm/transparent_hugepage/enabled
# Output: [always] madvise never
# [always] = activé (MAUVAIS)
# [never] = désactivé (BON)

echo ""
echo "2. THP defrag status:"
cat /sys/kernel/mm/transparent_hugepage/defrag
# Devrait aussi être [never]

echo ""
echo "3. khugepaged (THP daemon):"
if pgrep -x "khugepaged" > /dev/null; then
    echo "⚠️  khugepaged is running"
else
    echo "✅ khugepaged is not running"
fi

# Vérifier si Redis a détecté THP
echo ""
echo "4. Redis THP warning:"
if redis-cli INFO server | grep -q "WARNING.*transparent huge pages"; then
    echo "❌ Redis has detected THP is enabled!"
else
    echo "✅ No THP warning from Redis"
fi

# Statistiques THP
echo ""
echo "5. THP statistics:"
grep -r "" /sys/kernel/mm/transparent_hugepage/khugepaged/ 2>/dev/null | grep -v "Binary"
```

### 3. Désactiver THP de manière permanente

#### Méthode 1 : Via /etc/rc.local (simple)

```bash
#!/bin/bash
# disable-thp-rc-local.sh

echo "=== Disabling THP via rc.local ==="

# Créer script rc.local s'il n'existe pas
cat > /etc/rc.local << 'EOF'
#!/bin/bash
# Disable Transparent Huge Pages for Redis

if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi

if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi

exit 0
EOF

# Rendre exécutable
chmod +x /etc/rc.local

# Appliquer immédiatement
/etc/rc.local

# Vérifier
cat /sys/kernel/mm/transparent_hugepage/enabled

echo "✅ THP disabled via rc.local"
```

#### Méthode 2 : Via systemd service (recommandé)

```bash
#!/bin/bash
# disable-thp-systemd.sh

echo "=== Disabling THP via systemd service ==="

# Créer service systemd
cat > /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages (THP) for Redis
Before=redis.service redis-server.service
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
systemctl daemon-reload

# Activer le service (démarre au boot)
systemctl enable disable-thp.service

# Démarrer maintenant
systemctl start disable-thp.service

# Vérifier status
systemctl status disable-thp.service

# Vérifier que THP est bien désactivé
cat /sys/kernel/mm/transparent_hugepage/enabled

echo "✅ THP disabled via systemd"
```

#### Méthode 3 : Via GRUB (kernel boot parameter)

```bash
#!/bin/bash
# disable-thp-grub.sh

echo "=== Disabling THP via GRUB ==="

# Backup GRUB config
cp /etc/default/grub /etc/default/grub.backup

# Ajouter paramètre kernel
if ! grep -q "transparent_hugepage=never" /etc/default/grub; then
    sed -i 's/GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="transparent_hugepage=never /' /etc/default/grub
fi

# Mettre à jour GRUB
if [ -f /boot/grub/grub.cfg ]; then
    # Debian/Ubuntu
    update-grub
elif [ -f /boot/grub2/grub.cfg ]; then
    # RedHat/CentOS
    grub2-mkconfig -o /boot/grub2/grub.cfg
fi

echo "✅ THP will be disabled at next boot"
echo "⚠️  REBOOT REQUIRED for changes to take effect"
echo ""
echo "After reboot, verify with:"
echo "  cat /sys/kernel/mm/transparent_hugepage/enabled"
```

#### Méthode 4 : Configuration Ansible

```yaml
# ansible/disable-thp.yml
---
- name: Disable Transparent Huge Pages for Redis
  hosts: redis_servers
  become: yes
  tasks:
    - name: Create systemd service to disable THP
      copy:
        dest: /etc/systemd/system/disable-thp.service
        content: |
          [Unit]
          Description=Disable Transparent Huge Pages (THP)
          Before=redis.service redis-server.service
          After=sysinit.target local-fs.target

          [Service]
          Type=oneshot
          ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
          ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'
          RemainAfterExit=yes

          [Install]
          WantedBy=multi-user.target
        mode: '0644'

    - name: Enable and start disable-thp service
      systemd:
        name: disable-thp
        enabled: yes
        state: started
        daemon_reload: yes

    - name: Verify THP is disabled
      shell: cat /sys/kernel/mm/transparent_hugepage/enabled
      register: thp_status
      changed_when: false

    - name: Display THP status
      debug:
        msg: "THP Status: {{ thp_status.stdout }}"

    - name: Fail if THP is not disabled
      fail:
        msg: "THP is still enabled!"
      when: "'[never]' not in thp_status.stdout"
```

---

## Swap et Swappiness

### 1. Comprendre le problème du Swap

```
┌─────────────────────────────────────────────────────────────────┐
│                    SWAP ET REDIS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Swap normal (applications):                                    │
│  ├── RAM pleine → Kernel swap pages peu utilisées vers disque   │
│  ├── Libère RAM pour autres applications                        │
│  └── Performance acceptable pour apps normales                  │
│                                                                 │
│  Swap avec Redis (DÉSASTRE):                                    │
│  ├── Redis = in-memory database (TOUT doit être en RAM)         │
│  ├── Si swap → Données Redis sur disque lent                    │
│  ├── Accès disque = 1000x plus lent que RAM                     │
│  ├── Latence: 0.1ms → 10-100ms                                  │
│  ├── Clients timeout                                            │
│  ├── Redis peut freeze complètement                             │
│  └── OOM Killer peut tuer Redis                                 │
│                                                                 │
│  Symptômes de swap:                                             │
│  ├── Latence extrême et imprévisible                            │
│  ├── CPU élevé (page faults)                                    │
│  ├── I/O disque élevé                                           │
│  ├── Redis unresponsive                                         │
│  └── Connexions timeouts en cascade                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Vérifier l'utilisation du Swap

```bash
#!/bin/bash
# check-swap-usage.sh

echo "=== SWAP USAGE CHECK ==="

# 1. État général du swap
echo "1. System swap status:"
free -h

# 2. Utilisation swap
SWAP_TOTAL=$(free -m | grep Swap | awk '{print $2}')
SWAP_USED=$(free -m | grep Swap | awk '{print $3}')

echo ""
echo "2. Swap utilization:"
if [ "$SWAP_TOTAL" -eq 0 ]; then
    echo "✅ Swap is disabled"
elif [ "$SWAP_USED" -eq 0 ]; then
    echo "✅ Swap exists but not used"
else
    echo "⚠️  Swap is being used: ${SWAP_USED}MB / ${SWAP_TOTAL}MB"
fi

# 3. Swappiness
echo ""
echo "3. Swappiness value:"
SWAPPINESS=$(cat /proc/sys/vm/swappiness)
echo "Current swappiness: $SWAPPINESS"

if [ "$SWAPPINESS" -eq 0 ]; then
    echo "✅ Optimal for Redis (0)"
elif [ "$SWAPPINESS" -eq 1 ]; then
    echo "✅ Good for Redis (1)"
elif [ "$SWAPPINESS" -le 10 ]; then
    echo "⚠️  Acceptable (but prefer 0 or 1)"
else
    echo "❌ Too high for Redis! Should be 0 or 1"
fi

# 4. Vérifier si Redis swap
echo ""
echo "4. Redis process swap usage:"
REDIS_PID=$(pgrep -x redis-server)

if [ -n "$REDIS_PID" ]; then
    if [ -f "/proc/$REDIS_PID/status" ]; then
        VMSWAP=$(grep VmSwap /proc/$REDIS_PID/status | awk '{print $2}')
        if [ "$VMSWAP" -eq 0 ]; then
            echo "✅ Redis is not swapping"
        else
            echo "🚨 CRITICAL: Redis is swapping ${VMSWAP}KB!"
            echo "   This will cause severe performance issues"
        fi
    fi
else
    echo "Redis process not found"
fi

# 5. Historique swap (depuis démarrage)
echo ""
echo "5. System swap in/out statistics:"
cat /proc/vmstat | grep -E "pswpin|pswpout"
```

### 3. Configuration Swap optimale

#### Option 1 : Désactiver complètement le Swap (recommandé)

```bash
#!/bin/bash
# disable-swap.sh

echo "=== DISABLING SWAP ==="

# 1. Désactiver swap immédiatement
swapoff -a

# 2. Rendre permanent (supprimer de fstab)
cp /etc/fstab /etc/fstab.backup
sed -i '/swap/d' /etc/fstab

# 3. Vérifier
echo ""
echo "Verification:"
free -h
swapon --show

echo ""
if [ $(swapon --show | wc -l) -eq 0 ]; then
    echo "✅ Swap is completely disabled"
else
    echo "❌ Swap is still active"
fi

echo ""
echo "⚠️  IMPORTANT: Monitor RAM usage closely"
echo "   Ensure Redis has enough RAM to avoid OOM"
```

#### Option 2 : Swappiness = 0 (swap en dernier recours)

```bash
#!/bin/bash
# configure-swappiness.sh

echo "=== CONFIGURING SWAPPINESS ==="

# 1. Définir swappiness à 0 (immédiat)
sysctl vm.swappiness=0

# 2. Rendre permanent
cat >> /etc/sysctl.conf << EOF

# Swappiness for Redis (0 = avoid swap unless absolutely necessary)
vm.swappiness = 0
EOF

# 3. Recharger sysctl
sysctl -p

# 4. Vérifier
echo ""
echo "Current swappiness:"
cat /proc/sys/vm/swappiness

echo ""
echo "✅ Swappiness set to 0"
echo "   Swap will only be used in emergency (OOM situation)"
```

### 4. Monitoring du Swap

```python
#!/usr/bin/env python3
# monitor-swap.py

import psutil
import time
import sys

def check_redis_swap():
    """Vérifie si Redis utilise le swap"""
    for proc in psutil.process_iter(['pid', 'name', 'memory_info']):
        try:
            if 'redis' in proc.info['name'].lower():
                # Obtenir info swap du process
                with open(f"/proc/{proc.info['pid']}/status", 'r') as f:
                    for line in f:
                        if line.startswith('VmSwap:'):
                            swap_kb = int(line.split()[1])
                            if swap_kb > 0:
                                print(f"🚨 ALERT: Redis (PID {proc.info['pid']}) is swapping {swap_kb} KB!")
                                return False
                            else:
                                print(f"✅ Redis (PID {proc.info['pid']}) is not swapping")
                                return True
        except (psutil.NoSuchProcess, psutil.AccessDenied, FileNotFoundError):
            pass

    print("⚠️  Redis process not found")
    return None

def check_system_swap():
    """Vérifie utilisation swap système"""
    swap = psutil.swap_memory()

    print(f"\nSystem Swap Status:")
    print(f"  Total: {swap.total / (1024**3):.2f} GB")
    print(f"  Used: {swap.used / (1024**3):.2f} GB ({swap.percent}%)")
    print(f"  Free: {swap.free / (1024**3):.2f} GB")

    if swap.percent > 10:
        print(f"⚠️  WARNING: System is using {swap.percent}% swap")
        return False
    elif swap.percent > 0:
        print(f"⚠️  Swap is being used: {swap.percent}%")
        return True
    else:
        print(f"✅ No swap in use")
        return True

if __name__ == '__main__':
    print("=== REDIS SWAP MONITORING ===")

    while True:
        redis_ok = check_redis_swap()
        system_ok = check_system_swap()

        if redis_ok == False:
            print("\n🚨 CRITICAL: Redis is swapping - investigate immediately!")
            # Envoyer alerte
            sys.exit(1)

        print(f"\n[{time.strftime('%Y-%m-%d %H:%M:%S')}] Next check in 60s...")
        time.sleep(60)
```

---

## Overcommit Memory

### 1. Comprendre Overcommit

```
┌─────────────────────────────────────────────────────────────────┐
│              MEMORY OVERCOMMIT LINUX                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Modes d'overcommit (vm.overcommit_memory):                     │
│                                                                 │
│  0 = Heuristic (défaut) - MAUVAIS pour Redis                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Kernel estime si mémoire disponible                        │ │
│  │ Peut refuser malloc() même si RAM libre                    │ │
│  │ Redis fork() pour BGSAVE → malloc fail → CRASH             │ │
│  │ Imprévisible et dangereux                                  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  1 = Always overcommit - BON pour Redis                         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Kernel accepte toujours malloc()                           │ │
│  │ Fork() réussit toujours (COW)                              │ │
│  │ BGSAVE/AOF rewrite fonctionne                              │ │
│  │ Recommandé pour Redis                                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  2 = Never overcommit - MAUVAIS pour Redis                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Strictement limité à RAM + swap * ratio                    │ │
│  │ Fork() échoue si pas assez de mémoire virtuelle            │ │
│  │ Incompatible avec Redis                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Pourquoi Redis nécessite overcommit = 1

```
┌─────────────────────────────────────────────────────────────────┐
│          REDIS FORK PROCESS ET COPY-ON-WRITE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. État initial:                                               │
│     ┌──────────────────────────────────────────┐                │
│     │  Redis process: 8 GB de données en RAM   │                │
│     └──────────────────────────────────────────┘                │
│                                                                 │
│  2. BGSAVE appelé → fork():                                     │
│     ┌──────────────────────────────────────────┐                │
│     │  Parent (Redis): 8 GB                    │                │
│     │  Child (BGSAVE): 8 GB (virtuel COW)      │                │
│     │  Total virtual: 16 GB                    │ ◄── MALLOC!    │
│     └──────────────────────────────────────────┘                │
│                                                                 │
│  3. Sans overcommit_memory=1:                                   │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ Kernel refuse malloc() de 16 GB                        │  │
│     │ fork() FAIL → BGSAVE impossible                        │  │
│     │ Redis WARNING ou CRASH                                 │  │
│     └────────────────────────────────────────────────────────┘  │
│                                                                 │
│  4. Avec overcommit_memory=1:                                   │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ Kernel accepte malloc() de 16 GB                       │  │
│     │ fork() SUCCESS                                         │  │
│     │ COW: Seules pages modifiées réellement copiées         │  │
│     │ Utilisation réelle: 8 GB + ~500 MB (writes pendant)    │  │
│     │ BGSAVE fonctionne ✅                                   │  │
│     └────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Vérifier et configurer overcommit

```bash
#!/bin/bash
# configure-overcommit.sh

echo "=== MEMORY OVERCOMMIT CONFIGURATION ==="

# 1. Vérifier valeur actuelle
echo "1. Current overcommit_memory:"
CURRENT=$(cat /proc/sys/vm/overcommit_memory)
echo "Value: $CURRENT"

case $CURRENT in
    0)
        echo "❌ HEURISTIC mode (default) - BAD for Redis"
        ;;
    1)
        echo "✅ ALWAYS mode - GOOD for Redis"
        ;;
    2)
        echo "❌ NEVER mode - BAD for Redis"
        ;;
esac

# 2. Configurer overcommit_memory = 1
echo ""
echo "2. Setting overcommit_memory to 1..."

# Immédiat
sysctl vm.overcommit_memory=1

# Permanent
if ! grep -q "vm.overcommit_memory" /etc/sysctl.conf; then
    cat >> /etc/sysctl.conf << EOF

# Memory overcommit for Redis
# 1 = Always overcommit (required for Redis fork/BGSAVE)
vm.overcommit_memory = 1
EOF
fi

# Recharger
sysctl -p

# 3. Vérifier
echo ""
echo "3. Verification:"
cat /proc/sys/vm/overcommit_memory

# 4. Vérifier warnings Redis
echo ""
echo "4. Checking Redis warnings:"
if redis-cli INFO server | grep -q "WARNING.*overcommit_memory"; then
    echo "❌ Redis still has overcommit warning"
else
    echo "✅ No overcommit warning from Redis"
fi

echo ""
echo "✅ Overcommit memory configured for Redis"
```

---

## Autres paramètres kernel critiques

### 1. TCP et networking

```bash
#!/bin/bash
# configure-tcp-redis.sh

cat >> /etc/sysctl.conf << 'EOF'

# ============================================================================
# TCP AND NETWORKING FOR REDIS
# ============================================================================

# Augmenter backlog de connexions TCP
# Redis peut avoir beaucoup de clients
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TCP keepalive pour détecter connexions mortes
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 30

# Réutilisation rapide des sockets TIME_WAIT
net.ipv4.tcp_tw_reuse = 1

# TCP Fast Open (réduit latence)
net.ipv4.tcp_fastopen = 3

# Buffer sizes TCP (important pour débit)
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# Nombre maximum de file descriptors
fs.file-max = 2097152

EOF

# Appliquer
sysctl -p

echo "✅ TCP parameters configured for Redis"
```

### 2. Limites système (ulimit)

```bash
#!/bin/bash
# configure-limits-redis.sh

cat > /etc/security/limits.d/redis.conf << 'EOF'
# ============================================================================
# SYSTEM LIMITS FOR REDIS USER
# ============================================================================

# Nombre maximum de fichiers ouverts
redis soft nofile 65535
redis hard nofile 65535

# Nombre maximum de processus
redis soft nproc 65535
redis hard nproc 65535

# Taille maximale de la stack
redis soft stack 10240
redis hard stack 10240

# Core dumps (optionnel, pour debugging)
redis soft core unlimited
redis hard core unlimited

# ============================================================================
EOF

echo "✅ System limits configured for Redis user"

# Vérifier après redémarrage Redis
echo ""
echo "After Redis restart, verify with:"
echo "  sudo -u redis bash -c 'ulimit -a'"
```

### 3. Désactiver OOM Killer pour Redis

```bash
#!/bin/bash
# disable-oom-killer-redis.sh

echo "=== DISABLING OOM KILLER FOR REDIS ==="

# Trouver PID Redis
REDIS_PID=$(pgrep -x redis-server)

if [ -z "$REDIS_PID" ]; then
    echo "❌ Redis process not found"
    exit 1
fi

echo "Redis PID: $REDIS_PID"

# Désactiver OOM killer pour ce process
echo -1000 > /proc/$REDIS_PID/oom_score_adj

# Vérifier
SCORE=$(cat /proc/$REDIS_PID/oom_score_adj)
echo "OOM score: $SCORE"

if [ "$SCORE" -eq -1000 ]; then
    echo "✅ Redis protected from OOM killer"
else
    echo "❌ Failed to protect Redis from OOM killer"
fi

# Rendre permanent via systemd
cat > /etc/systemd/system/redis-oom-protect.service << 'EOF'
[Unit]
Description=Protect Redis from OOM Killer
After=redis.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo -1000 > /proc/$(pgrep -x redis-server)/oom_score_adj'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable redis-oom-protect.service

echo ""
echo "✅ OOM protection will persist after Redis restart"
```

---

## Configuration sysctl.conf complète

```bash
# /etc/sysctl.conf - PRODUCTION REDIS CONFIGURATION
# ============================================================================
# Apply with: sysctl -p
# ============================================================================

# ============================================================================
# MEMORY MANAGEMENT
# ============================================================================

# Memory overcommit - CRITIQUE pour Redis
# 1 = Always overcommit (requis pour fork/BGSAVE)
vm.overcommit_memory = 1

# Overcommit ratio (si mode = 2, non utilisé avec mode = 1)
vm.overcommit_ratio = 100

# Swappiness - Minimiser usage du swap
# 0 = Utiliser swap seulement en cas d'urgence
vm.swappiness = 0

# Transparent Huge Pages - Désactivé via systemd service
# (ne peut pas être configuré via sysctl)

# ============================================================================
# TCP AND NETWORKING
# ============================================================================

# TCP backlog - Augmenter pour haute concurrence
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TCP keepalive pour connexions longues
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 30

# Réutilisation rapide des sockets TIME_WAIT
net.ipv4.tcp_tw_reuse = 1

# TCP Fast Open
net.ipv4.tcp_fastopen = 3

# Buffer TCP (important pour débit)
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# Netfilter connection tracking
net.netfilter.nf_conntrack_max = 1048576

# ============================================================================
# FILE SYSTEM
# ============================================================================

# Nombre maximum de file descriptors
fs.file-max = 2097152

# Inotify limits (si beaucoup de files watched)
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192

# ============================================================================
# KERNEL
# ============================================================================

# Kernel panic behavior (optionnel)
# kernel.panic = 10
# kernel.panic_on_oops = 1

# PID max (pour nombreux processus)
kernel.pid_max = 65536

# ============================================================================
```

---

## Script de validation complète

```bash
#!/bin/bash
# validate-linux-config-redis.sh

echo "==========================================="
echo "REDIS LINUX CONFIGURATION VALIDATION"
echo "==========================================="

ERRORS=0
WARNINGS=0

# ============================================================================
# 1. TRANSPARENT HUGE PAGES
# ============================================================================
echo ""
echo "1. Checking Transparent Huge Pages..."
THP_ENABLED=$(cat /sys/kernel/mm/transparent_hugepage/enabled)

if [[ $THP_ENABLED == *"[never]"* ]]; then
    echo "   ✅ THP is disabled"
else
    echo "   ❌ CRITICAL: THP is enabled!"
    echo "      Current: $THP_ENABLED"
    echo "      Action: Disable THP immediately"
    ERRORS=$((ERRORS + 1))
fi

THP_DEFRAG=$(cat /sys/kernel/mm/transparent_hugepage/defrag)
if [[ $THP_DEFRAG == *"[never]"* ]]; then
    echo "   ✅ THP defrag is disabled"
else
    echo "   ⚠️  WARNING: THP defrag not disabled"
    echo "      Current: $THP_DEFRAG"
    WARNINGS=$((WARNINGS + 1))
fi

# ============================================================================
# 2. SWAP
# ============================================================================
echo ""
echo "2. Checking Swap configuration..."

SWAP_TOTAL=$(free -m | grep Swap | awk '{print $2}')
SWAP_USED=$(free -m | grep Swap | awk '{print $3}')

if [ "$SWAP_TOTAL" -eq 0 ]; then
    echo "   ✅ Swap is disabled (optimal)"
else
    echo "   ⚠️  Swap is enabled (${SWAP_TOTAL}MB total)"

    if [ "$SWAP_USED" -eq 0 ]; then
        echo "      ✅ But swap is not being used"
    else
        echo "      ❌ CRITICAL: Swap is being used (${SWAP_USED}MB)"
        ERRORS=$((ERRORS + 1))
    fi
fi

SWAPPINESS=$(cat /proc/sys/vm/swappiness)
echo "   Swappiness: $SWAPPINESS"

if [ "$SWAPPINESS" -le 1 ]; then
    echo "   ✅ Swappiness is optimal ($SWAPPINESS)"
elif [ "$SWAPPINESS" -le 10 ]; then
    echo "   ⚠️  Swappiness is acceptable ($SWAPPINESS) but prefer 0"
    WARNINGS=$((WARNINGS + 1))
else
    echo "   ❌ CRITICAL: Swappiness too high ($SWAPPINESS)"
    echo "      Should be 0 or 1"
    ERRORS=$((ERRORS + 1))
fi

# Vérifier si Redis swap
REDIS_PID=$(pgrep -x redis-server)
if [ -n "$REDIS_PID" ]; then
    if [ -f "/proc/$REDIS_PID/status" ]; then
        REDIS_SWAP=$(grep VmSwap /proc/$REDIS_PID/status | awk '{print $2}')
        if [ "$REDIS_SWAP" -eq 0 ]; then
            echo "   ✅ Redis process is not swapping"
        else
            echo "   🚨 CRITICAL: Redis is swapping ${REDIS_SWAP}KB"
            ERRORS=$((ERRORS + 1))
        fi
    fi
fi

# ============================================================================
# 3. OVERCOMMIT MEMORY
# ============================================================================
echo ""
echo "3. Checking Memory Overcommit..."

OVERCOMMIT=$(cat /proc/sys/vm/overcommit_memory)
echo "   Overcommit mode: $OVERCOMMIT"

case $OVERCOMMIT in
    0)
        echo "   ❌ CRITICAL: Heuristic mode (0)"
        echo "      Must be 1 for Redis"
        ERRORS=$((ERRORS + 1))
        ;;
    1)
        echo "   ✅ Always overcommit (optimal)"
        ;;
    2)
        echo "   ❌ CRITICAL: Never overcommit (2)"
        echo "      Incompatible with Redis"
        ERRORS=$((ERRORS + 1))
        ;;
esac

# ============================================================================
# 4. TCP CONFIGURATION
# ============================================================================
echo ""
echo "4. Checking TCP configuration..."

SOMAXCONN=$(cat /proc/sys/net/core/somaxconn)
echo "   somaxconn: $SOMAXCONN"

if [ "$SOMAXCONN" -ge 1024 ]; then
    echo "   ✅ somaxconn is sufficient"
else
    echo "   ⚠️  WARNING: somaxconn is low ($SOMAXCONN)"
    echo "      Recommend 65535 for Redis"
    WARNINGS=$((WARNINGS + 1))
fi

TCP_BACKLOG=$(cat /proc/sys/net/ipv4/tcp_max_syn_backlog)
if [ "$TCP_BACKLOG" -ge 1024 ]; then
    echo "   ✅ tcp_max_syn_backlog is sufficient ($TCP_BACKLOG)"
else
    echo "   ⚠️  WARNING: tcp_max_syn_backlog is low"
    WARNINGS=$((WARNINGS + 1))
fi

# ============================================================================
# 5. FILE LIMITS
# ============================================================================
echo ""
echo "5. Checking file limits..."

FILE_MAX=$(cat /proc/sys/fs/file-max)
echo "   fs.file-max: $FILE_MAX"

if [ "$FILE_MAX" -ge 65535 ]; then
    echo "   ✅ System file limit is sufficient"
else
    echo "   ⚠️  WARNING: fs.file-max is low"
    WARNINGS=$((WARNINGS + 1))
fi

# Vérifier limites Redis user
if [ -n "$REDIS_PID" ]; then
    NOFILE=$(cat /proc/$REDIS_PID/limits | grep "open files" | awk '{print $4}')
    echo "   Redis nofile limit: $NOFILE"

    if [ "$NOFILE" -ge 10000 ]; then
        echo "   ✅ Redis file limit is sufficient"
    else
        echo "   ⚠️  WARNING: Redis file limit is low"
        WARNINGS=$((WARNINGS + 1))
    fi
fi

# ============================================================================
# 6. REDIS WARNINGS
# ============================================================================
echo ""
echo "6. Checking Redis warnings..."

if command -v redis-cli &> /dev/null; then
    REDIS_INFO=$(redis-cli INFO server 2>/dev/null)

    if echo "$REDIS_INFO" | grep -q "WARNING.*transparent huge pages"; then
        echo "   ❌ Redis detected THP enabled"
        ERRORS=$((ERRORS + 1))
    fi

    if echo "$REDIS_INFO" | grep -q "WARNING.*overcommit_memory"; then
        echo "   ❌ Redis detected wrong overcommit_memory"
        ERRORS=$((ERRORS + 1))
    fi

    if [ $ERRORS -eq 0 ]; then
        echo "   ✅ No critical warnings from Redis"
    fi
fi

# ============================================================================
# SUMMARY
# ============================================================================
echo ""
echo "==========================================="
echo "VALIDATION SUMMARY"
echo "==========================================="
echo "Errors: $ERRORS"
echo "Warnings: $WARNINGS"
echo ""

if [ $ERRORS -eq 0 ] && [ $WARNINGS -eq 0 ]; then
    echo "✅ ALL CHECKS PASSED"
    echo "Linux configuration is optimal for Redis"
    exit 0
elif [ $ERRORS -eq 0 ]; then
    echo "⚠️  PASSED WITH WARNINGS"
    echo "Configuration is acceptable but can be improved"
    exit 0
else
    echo "❌ VALIDATION FAILED"
    echo "Critical issues must be fixed before production"
    exit 1
fi
```

---

## Configuration automatisée complète

```bash
#!/bin/bash
# setup-linux-for-redis.sh
# Configuration automatique complète d'un serveur Linux pour Redis

set -e

echo "==========================================="
echo "LINUX CONFIGURATION FOR REDIS"
echo "==========================================="

# Vérifier root
if [ "$EUID" -ne 0 ]; then
    echo "❌ This script must be run as root"
    exit 1
fi

# ============================================================================
# 1. DISABLE TRANSPARENT HUGE PAGES
# ============================================================================
echo ""
echo "1. Disabling Transparent Huge Pages..."

# Désactiver immédiatement
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Créer service systemd pour désactiver au boot
cat > /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages (THP) for Redis
Before=redis.service redis-server.service
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable disable-thp.service
systemctl start disable-thp.service

echo "   ✅ THP disabled"

# ============================================================================
# 2. CONFIGURE SWAP
# ============================================================================
echo ""
echo "2. Configuring Swap..."

# Option: Désactiver complètement (décommenter si souhaité)
# swapoff -a
# sed -i '/swap/d' /etc/fstab
# echo "   ✅ Swap disabled"

# Ou: Configurer swappiness à 0
sysctl -w vm.swappiness=0
echo "   ✅ Swappiness set to 0"

# ============================================================================
# 3. CONFIGURE SYSCTL
# ============================================================================
echo ""
echo "3. Configuring kernel parameters..."

# Backup existant
cp /etc/sysctl.conf /etc/sysctl.conf.backup

# Ajouter configuration
cat >> /etc/sysctl.conf << 'EOF'

# ============================================================================
# REDIS PRODUCTION CONFIGURATION
# ============================================================================

# Memory overcommit
vm.overcommit_memory = 1
vm.swappiness = 0

# TCP
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fastopen = 3

# Buffers
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# File descriptors
fs.file-max = 2097152

EOF

# Appliquer
sysctl -p

echo "   ✅ Kernel parameters configured"

# ============================================================================
# 4. CONFIGURE LIMITS
# ============================================================================
echo ""
echo "4. Configuring system limits..."

cat > /etc/security/limits.d/redis.conf << 'EOF'
redis soft nofile 65535
redis hard nofile 65535
redis soft nproc 65535
redis hard nproc 65535
redis soft stack 10240
redis hard stack 10240
EOF

echo "   ✅ System limits configured"

# ============================================================================
# SUMMARY
# ============================================================================
echo ""
echo "==========================================="
echo "✅ CONFIGURATION COMPLETE"
echo "==========================================="
echo ""
echo "Changes applied:"
echo "  - THP disabled"
echo "  - Swappiness set to 0"
echo "  - Memory overcommit enabled"
echo "  - TCP parameters optimized"
echo "  - System limits increased"
echo ""
echo "⚠️  IMPORTANT:"
echo "  - Redis service must be restarted to apply limits"
echo "  - Run validation script to verify:"
echo "    ./validate-linux-config-redis.sh"
echo ""
```

---

## Checklist de configuration Linux

### Checklist déploiement

- [ ] **THP désactivé**
  ```bash
  cat /sys/kernel/mm/transparent_hugepage/enabled
  # Doit afficher [never]
  ```

- [ ] **THP persiste au reboot**
  - Service systemd créé et activé
  - Ou paramètre GRUB configuré

- [ ] **Swap désactivé OU swappiness = 0**
  ```bash
  cat /proc/sys/vm/swappiness
  # Doit être 0
  ```

- [ ] **Overcommit memory = 1**
  ```bash
  cat /proc/sys/vm/overcommit_memory
  # Doit être 1
  ```

- [ ] **somaxconn augmenté**
  ```bash
  cat /proc/sys/net/core/somaxconn
  # Doit être ≥ 1024
  ```

- [ ] **File limits augmentés**
  ```bash
  ulimit -n
  # Doit être ≥ 10000
  ```

- [ ] **Aucun warning Redis**
  ```bash
  redis-cli INFO server | grep WARNING
  # Ne doit rien retourner
  ```

### Checklist monitoring

- [ ] **Surveillance swap**
  - Alertes si swap utilisé
  - Monitoring process Redis

- [ ] **Surveillance THP**
  - Vérification quotidienne
  - Alertes si réactivé

- [ ] **Surveillance mémoire**
  - RAM usage < 80%
  - Pas de fragmentation excessive

---

## 📚 Ressources complémentaires

### Documentation

- [Redis Administration](https://redis.io/docs/management/admin/)
- [Linux Kernel Documentation](https://www.kernel.org/doc/)
- [Transparent Huge Pages](https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html)

### Outils

- **sysctl** - Configuration kernel
- **vmstat** - Statistiques mémoire
- **free** - Utilisation mémoire
- **ulimit** - Limites système

---

**Section suivante :** [12.7 - Dimensionnement et planification de capacité](./07-dimensionnement-planification-capacite.md)

⏭️ [Dimensionnement et planification de capacité](/12-redis-production-securite/07-dimensionnement-planification-capacite.md)
