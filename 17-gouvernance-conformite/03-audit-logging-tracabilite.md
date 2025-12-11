🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.3 Audit Logging et Traçabilité

## Introduction

L'audit logging est une exigence fondamentale de la conformité réglementaire. Il permet de répondre aux questions essentielles du "Qui, Quoi, Quand, Où, Pourquoi" pour chaque action effectuée sur les données sensibles. Pour Redis, qui ne dispose pas nativement d'un système d'audit robuste, cette exigence représente un défi technique nécessitant des solutions tierces et des procédures rigoureuses.

**Définition de l'audit logging :**
> L'enregistrement systématique et horodaté de tous les événements de sécurité et accès aux données permettant la reconstruction complète d'une séquence d'activités et l'identification des acteurs impliqués.

---

## Cadre réglementaire de l'audit logging

### RGPD - Article 32 : Sécurité du traitement

**Article 32.1.d :**
> "Une procédure visant à tester, à analyser et à évaluer régulièrement l'efficacité des mesures techniques et organisationnelles pour assurer la sécurité du traitement."

**Implications pour Redis :**
- Traçabilité des accès aux données personnelles
- Logs permettant de détecter les violations de données (Article 33)
- Conservation des logs pour démontrer la conformité (accountability)
- Capacité à identifier les accès non autorisés

**Durée de conservation :**
- Aucune durée minimale spécifiée dans le RGPD
- Recommandation CNIL : **12 mois minimum** pour les logs de sécurité
- Possibilité de conservation plus longue si justifiée (intérêt légitime)

### PCI DSS - Requirement 10 : Track and Monitor Access

**10.1 : Implement audit trails**
> Établir un processus pour relier tous les accès aux composants système à chaque utilisateur individuel.

**Événements obligatoires à logger (10.2) :**
```
10.2.1 : Tous les accès individuels aux données de titulaire de carte (PAN)
10.2.2 : Toutes les actions effectuées par un utilisateur avec privilèges
10.2.3 : Tous les accès à l'audit trail
10.2.4 : Tentatives d'accès logiques invalides
10.2.5 : Utilisation et modifications des mécanismes d'identification/authentification
10.2.6 : Initialisation, arrêt, pause de l'audit trail
10.2.7 : Création et suppression d'objets au niveau système
```

**Contenu minimum de chaque log (10.3) :**
```
10.3.1 : Identification utilisateur
10.3.2 : Type d'événement
10.3.3 : Date et heure
10.3.4 : Succès ou échec
10.3.5 : Origine de l'événement
10.3.6 : Identité ou nom de la donnée, composant système ou ressource affectée
```

**Revue des logs (10.6) :**
```
10.6.1 : Revue quotidienne des logs (au minimum)
10.6.2 : Revue périodique des logs de tous les composants système
10.6.3 : Suivi des exceptions et anomalies
```

**Rétention PCI DSS (10.7) :**
```
10.7.2 : Conserver l'historique d'audit au moins 12 mois
10.7.3 : Au moins 3 mois doivent être immédiatement disponibles pour analyse
```

### HIPAA - Security Rule §164.312

**§164.312(b) : Audit controls (Required)**
> "Implémenter des mécanismes matériels, logiciels et/ou procéduraux qui enregistrent et examinent l'activité dans les systèmes d'information contenant des PHI."

**Exigences HIPAA pour Redis :**
```
□ Enregistrement de tous les accès aux PHI (Protected Health Information)
□ Identification de l'utilisateur effectuant l'accès
□ Type d'accès (lecture, modification, suppression)
□ Date et heure de l'accès
□ Échecs d'authentification
□ Conservation minimum 6 ans (§164.316(b)(2))
```

**§164.308(a)(1)(ii)(D) : Information system activity review (Required)**
> "Examiner régulièrement les enregistrements d'activité du système d'information, tels que les logs d'audit."

### SOC 2 - Common Criteria CC7.2

**Détection et analyse des incidents :**
```
CC7.2 : Le système détecte les incidents de sécurité, les rapporte et analyse
        leur impact potentiel.
```

**Exigences pour les logs :**
- Centralisation dans un SIEM
- Corrélation des événements
- Alerting en temps réel
- Analyse forensique en cas d'incident
- Revue régulière (au moins mensuelle)

### ISO 27001 - Annexe A.12.4

**A.12.4.1 : Event logging (Required)**
```
Objectif : Enregistrer les événements et générer des preuves

Mesures :
- Logs utilisateurs, exceptions, fautes
- Identité utilisateur, date/heure, événement, origine, succès/échec
- Protection des logs contre altération
- Synchronisation temporelle (NTP)
```

**A.12.4.2 : Protection of log information (Required)**
```
- Logs protégés contre modifications non autorisées
- Accès restreint aux logs (read-only pour la plupart)
- Backup régulier des logs
```

**A.12.4.3 : Administrator and operator logs (Required)**
```
- Activités des administrateurs et opérateurs enregistrées
- Revue régulière des logs privilégiés
```

**A.12.4.4 : Clock synchronization (Required)**
```
- Synchronisation NTP obligatoire
- Précision temporelle pour corrélation
```

---

## Limitations natives de Redis en matière d'audit

### Ce que Redis fournit (natif)

**1. Slowlog**
```bash
# Log des commandes lentes
CONFIG SET slowlog-log-slower-than 10000  # 10ms
CONFIG SET slowlog-max-len 128

SLOWLOG GET 10
```

**Limitations :**
- ❌ Pas d'identification de l'utilisateur (avant Redis 6 ACL)
- ❌ Pas d'IP source
- ❌ Seulement les commandes lentes (pas toutes)
- ❌ Taille limitée (in-memory, pas persistant)
- ❌ Pas d'horodatage précis (timestamp Unix)

**2. Commande MONITOR**
```bash
# Streaming de toutes les commandes en temps réel
redis-cli MONITOR
```

**Limitations :**
- ❌ Impact performance majeur (production inacceptable)
- ❌ Pas d'identification utilisateur
- ❌ Pas de persistance (output console uniquement)
- ❌ Pas de filtrage
- ❌ TRÈS DANGEREUX : peut exposer des données sensibles

**3. Logs système (redis-server.log)**
```bash
# /var/log/redis/redis-server.log
loglevel notice  # debug, verbose, notice, warning
```

**Contenu :**
- Démarrage/arrêt du serveur
- Erreurs de connexion
- Changements de configuration
- Réplication/cluster events
- Warnings et erreurs

**Limitations :**
- ❌ Pas de log des commandes exécutées
- ❌ Pas d'audit trail des accès aux données
- ❌ Logs génériques (opérationnels, pas sécurité)

**4. ACL LOG (Redis 6+)**
```bash
# Log des échecs ACL
ACL LOG 10

# Exemple de sortie
1) "count"
2) (integer) 1
3) "reason"
4) "command"
5) "context"
6) "toplevel"
7) "object"
8) "GET"
9) "username"
10) "alice"
11) "age-seconds"
12) "5"
13) "client-info"
14) "id=7 addr=192.168.1.10:52144 ..."
```

**Avantages :**
- ✅ Identification utilisateur (ACL)
- ✅ IP source
- ✅ Commande refusée
- ✅ Raison du refus

**Limitations :**
- ❌ Seulement les échecs ACL (pas les succès)
- ❌ In-memory (taille limitée)
- ❌ Pas de persistance automatique
- ❌ Pas de centralisation

### Conclusion : Nécessité de solutions tierces

**Pour une conformité PCI DSS, HIPAA, SOC 2, Redis natif est INSUFFISANT.**

Il faut implémenter :
1. **Proxy avec audit logging** (Envoy, HAProxy, twemproxy modifié)
2. **Instrumentation applicative** (logs côté application)
3. **Monitoring externe** (Packet capture, eBPF)
4. **Module Redis custom** (extension C)
5. **Redis Enterprise** (solution commerciale avec audit)

---

## Solutions d'audit logging pour Redis

### Solution 1 : Proxy avec audit logging (recommandé)

**Architecture :**
```
[Application] → [Proxy avec audit] → [Redis]
                      ↓
                  [SIEM/Log Storage]
```

#### Option 1A : Envoy Proxy

**Avantages :**
- Proxy moderne, performant, cloud-native
- Support TLS natif
- Extensible (filtres personnalisés)
- Metrics et tracing intégrés

**Configuration Envoy pour Redis :**

```yaml
# envoy.yaml
static_resources:
  listeners:
  - name: redis_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 6380

    filter_chains:
    - filters:
      # 1. Filtre Redis proxy
      - name: envoy.filters.network.redis_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.redis_proxy.v3.RedisProxy
          stat_prefix: redis_stats

          # Connexion au Redis backend
          settings:
            op_timeout: 5s
            enable_redirection: true
            enable_command_stats: true

          # Prefix routing (si cluster)
          prefix_routes:
            routes:
            - prefix: "/"
              cluster: redis_cluster

      # 2. Filtre d'audit (custom Lua filter)
      - name: envoy.filters.network.lua
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.lua.v3.Lua
          inline_code: |
            function envoy_on_request(request_handle)
              local metadata = request_handle:streamInfo():dynamicMetadata()
              local client_ip = request_handle:connection():remoteAddress()
              local timestamp = os.time()

              -- Log l'accès (envoyer vers syslog ou fichier)
              log_audit({
                timestamp = timestamp,
                client_ip = client_ip,
                user = metadata:get("user") or "unknown",
                command = metadata:get("command") or "unknown",
                success = true
              })
            end

      # 3. TLS (optionnel)
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
            - certificate_chain:
                filename: "/etc/envoy/certs/redis-cert.pem"
              private_key:
                filename: "/etc/envoy/certs/redis-key.pem"

  clusters:
  - name: redis_cluster
    connect_timeout: 1s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: MAGLEV

    # Health checking
    health_checks:
    - timeout: 1s
      interval: 5s
      unhealthy_threshold: 3
      healthy_threshold: 2
      custom_health_check:
        name: envoy.health_checkers.redis
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.health_checkers.redis.v3.Redis
          key: health_check

    # Backend Redis
    load_assignment:
      cluster_name: redis_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: redis-master.internal
                port_value: 6379

# Admin interface pour metrics
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901
```

**Audit logging avec Envoy + syslog :**

```yaml
# Ajouter un tap filter pour audit complet
static_resources:
  listeners:
  - name: redis_listener
    # ... (config précédente)

    filter_chains:
    - filters:
      - name: envoy.filters.network.tap
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.tap.v3.Tap
          common_config:
            static_config:
              match_config:
                any_match: true
              output_config:
                sinks:
                - format: JSON_BODY_AS_STRING
                  file_per_tap:
                    path_prefix: /var/log/envoy/redis_audit_
```

**Parser les logs Envoy :**

```python
# Script Python pour extraire les commandes Redis des logs Envoy
import json
import re
from datetime import datetime

def parse_envoy_redis_audit(log_line):
    """
    Parser une ligne de log Envoy et extraire les détails d'audit
    """
    try:
        log = json.loads(log_line)

        # Extraire les informations pertinentes
        audit_event = {
            'timestamp': datetime.fromtimestamp(log['timestamp']).isoformat(),
            'client_ip': log.get('downstream_remote_address', 'unknown'),
            'command': extract_redis_command(log.get('request', '')),
            'response_code': log.get('response_code', 'unknown'),
            'duration_ms': log.get('duration', 0),
            'bytes_sent': log.get('bytes_sent', 0),
            'bytes_received': log.get('bytes_received', 0),
        }

        # Envoyer vers SIEM
        send_to_siem(audit_event)

        return audit_event

    except Exception as e:
        print(f"Error parsing log: {e}")
        return None

def extract_redis_command(request_data):
    """Extraire la commande Redis du payload"""
    # Redis protocol : *<argc>\r\n$<len>\r\n<data>\r\n...
    match = re.search(r'\$\d+\r\n(\w+)\r\n', request_data)
    if match:
        return match.group(1).upper()
    return "UNKNOWN"

# Traitement continu des logs
with open('/var/log/envoy/redis_audit.log', 'r') as f:
    for line in f:
        parse_envoy_redis_audit(line)
```

#### Option 1B : Redis Audit Proxy (custom)

**Solution sur-mesure avec logging complet :**

```python
#!/usr/bin/env python3
"""
Redis Audit Proxy
Proxy transparent avec audit logging complet pour conformité PCI DSS/HIPAA

Architecture :
[Client] → [Audit Proxy :6380] → [Redis :6379]
                ↓
          [Audit Logs → SIEM]
"""

import asyncio
import logging
import json
import time
from datetime import datetime
import aioredis
import hashlib

# Configuration
PROXY_HOST = '0.0.0.0'
PROXY_PORT = 6380
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
AUDIT_LOG_FILE = '/var/log/redis/audit.log'
SIEM_ENDPOINT = 'https://siem.example.com/api/events'

# Configuration du logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger('redis-audit-proxy')

# Audit logger séparé
audit_logger = logging.getLogger('redis-audit')
audit_handler = logging.FileHandler(AUDIT_LOG_FILE)
audit_handler.setFormatter(logging.Formatter('%(message)s'))
audit_logger.addHandler(audit_handler)
audit_logger.setLevel(logging.INFO)

# Commandes sensibles nécessitant un audit détaillé
SENSITIVE_COMMANDS = {
    'GET', 'SET', 'DEL', 'HGET', 'HSET', 'HDEL',
    'MGET', 'MSET', 'KEYS', 'SCAN', 'FLUSHDB', 'FLUSHALL',
    'CONFIG', 'SCRIPT', 'EVAL', 'AUTH', 'ACL'
}

# Commandes à bloquer complètement (configurable)
BLOCKED_COMMANDS = {
    'FLUSHALL',  # Trop dangereux en production
    'FLUSHDB',
    'DEBUG',
    'SHUTDOWN'
}

class RedisAuditProxy:
    """
    Proxy Redis avec audit logging complet
    """

    def __init__(self):
        self.redis_pool = None
        self.stats = {
            'total_commands': 0,
            'blocked_commands': 0,
            'audit_events': 0,
            'errors': 0
        }

    async def init_redis_pool(self):
        """Initialiser le pool de connexions Redis backend"""
        self.redis_pool = await aioredis.create_redis_pool(
            f'redis://{REDIS_HOST}:{REDIS_PORT}',
            minsize=10,
            maxsize=100,
            timeout=5
        )
        logger.info(f"Connected to Redis backend at {REDIS_HOST}:{REDIS_PORT}")

    async def handle_client(self, reader, writer):
        """
        Gérer une connexion client
        """
        client_address = writer.get_extra_info('peername')
        session_id = hashlib.sha256(
            f"{client_address}{time.time()}".encode()
        ).hexdigest()[:16]

        logger.info(f"New connection from {client_address} (session: {session_id})")

        try:
            while True:
                # Lire la commande Redis (RESP protocol)
                command_data = await self.read_redis_command(reader)

                if not command_data:
                    break  # Connexion fermée

                # Parser la commande
                command = self.parse_redis_command(command_data)

                # Audit logging
                audit_event = await self.create_audit_event(
                    session_id=session_id,
                    client_address=client_address,
                    command=command,
                    raw_data=command_data
                )

                # Vérifier si la commande est bloquée
                if self.is_command_blocked(command):
                    response = self.create_error_response(
                        "ERR command not allowed by proxy policy"
                    )
                    audit_event['blocked'] = True
                    audit_event['block_reason'] = 'policy_violation'
                    self.stats['blocked_commands'] += 1
                else:
                    # Transférer au Redis backend
                    start_time = time.time()
                    response = await self.forward_to_redis(command_data)
                    duration_ms = (time.time() - start_time) * 1000

                    audit_event['duration_ms'] = round(duration_ms, 2)
                    audit_event['success'] = not response.startswith(b'-ERR')
                    audit_event['blocked'] = False

                # Écrire l'audit log
                self.write_audit_log(audit_event)

                # Envoyer la réponse au client
                writer.write(response)
                await writer.drain()

                self.stats['total_commands'] += 1

        except Exception as e:
            logger.error(f"Error handling client {client_address}: {e}")
            self.stats['errors'] += 1

        finally:
            writer.close()
            await writer.wait_closed()
            logger.info(f"Connection closed: {client_address} (session: {session_id})")

    async def read_redis_command(self, reader):
        """
        Lire une commande Redis au format RESP
        """
        try:
            # RESP protocol : *<argc>\r\n$<len>\r\n<arg>\r\n...
            line = await reader.readline()

            if not line:
                return None

            if not line.startswith(b'*'):
                return line  # Commande inline (telnet-style)

            # Parser le nombre d'arguments
            argc = int(line[1:-2])

            command_parts = []
            for _ in range(argc):
                # Lire la longueur de l'argument
                length_line = await reader.readline()
                arg_length = int(length_line[1:-2])

                # Lire l'argument
                arg = await reader.readexactly(arg_length)
                await reader.readexactly(2)  # \r\n

                command_parts.append(arg)

            return b'*' + str(argc).encode() + b'\r\n' + b''.join(
                b'$' + str(len(part)).encode() + b'\r\n' + part + b'\r\n'
                for part in command_parts
            )

        except asyncio.IncompleteReadError:
            return None
        except Exception as e:
            logger.error(f"Error reading Redis command: {e}")
            return None

    def parse_redis_command(self, command_data):
        """
        Parser une commande Redis RESP pour extraction
        """
        try:
            if command_data.startswith(b'*'):
                # RESP array format
                parts = []
                lines = command_data.split(b'\r\n')
                i = 1  # Skip first line (*N)
                while i < len(lines):
                    if lines[i].startswith(b'$'):
                        parts.append(lines[i+1].decode('utf-8', errors='replace'))
                        i += 2
                    else:
                        i += 1

                return {
                    'command': parts[0].upper() if parts else 'UNKNOWN',
                    'args': parts[1:] if len(parts) > 1 else [],
                    'argc': len(parts)
                }
            else:
                # Inline format
                parts = command_data.decode('utf-8', errors='replace').strip().split()
                return {
                    'command': parts[0].upper() if parts else 'UNKNOWN',
                    'args': parts[1:] if len(parts) > 1 else [],
                    'argc': len(parts)
                }
        except Exception as e:
            logger.error(f"Error parsing command: {e}")
            return {'command': 'PARSE_ERROR', 'args': [], 'argc': 0}

    async def create_audit_event(self, session_id, client_address, command, raw_data):
        """
        Créer un événement d'audit structuré
        """
        event = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'event_type': 'redis_command',
            'session_id': session_id,
            'client_ip': client_address[0],
            'client_port': client_address[1],
            'command': command['command'],
            'argc': command['argc'],
            'sensitive': command['command'] in SENSITIVE_COMMANDS,
            'proxy_version': '1.0.0',
        }

        # Ajouter les arguments (avec redaction si sensible)
        if command['command'] in SENSITIVE_COMMANDS:
            event['args'] = self.redact_sensitive_args(command)
        else:
            event['args'] = command['args'][:10]  # Limiter à 10 args

        # Taille de la commande (pour détecter les big keys)
        event['command_size_bytes'] = len(raw_data)

        return event

    def redact_sensitive_args(self, command):
        """
        Redacter les valeurs sensibles dans les arguments
        (PCI DSS : ne jamais logger les PANs, CVV, etc.)
        """
        cmd = command['command']
        args = command['args']

        # Pour SET, HSET : Ne pas logger la valeur
        if cmd in ('SET', 'SETEX', 'SETNX', 'HSET', 'HMSET'):
            if len(args) >= 2:
                return [args[0], '<REDACTED>'] + args[2:]

        # Pour GET, DEL : Logger seulement les clés
        if cmd in ('GET', 'DEL', 'HGET', 'HDEL'):
            return args  # Les clés sont OK (identifiants, pas valeurs)

        # Par défaut : Redacter tous les args
        return ['<REDACTED>' for _ in args]

    def is_command_blocked(self, command):
        """Vérifier si une commande est bloquée par policy"""
        return command['command'] in BLOCKED_COMMANDS

    async def forward_to_redis(self, command_data):
        """
        Transférer la commande au Redis backend
        """
        try:
            # Utiliser une connexion du pool
            async with self.redis_pool.get() as conn:
                # Envoyer la commande raw
                conn.writer.write(command_data)
                await conn.writer.drain()

                # Lire la réponse
                response = await self.read_redis_response(conn.reader)
                return response

        except Exception as e:
            logger.error(f"Error forwarding to Redis: {e}")
            return self.create_error_response(f"ERR proxy error: {str(e)}")

    async def read_redis_response(self, reader):
        """Lire une réponse Redis RESP"""
        try:
            first_byte = await reader.readexactly(1)

            if first_byte == b'+':  # Simple string
                line = await reader.readline()
                return first_byte + line

            elif first_byte == b'-':  # Error
                line = await reader.readline()
                return first_byte + line

            elif first_byte == b':':  # Integer
                line = await reader.readline()
                return first_byte + line

            elif first_byte == b'$':  # Bulk string
                length_line = await reader.readline()
                length = int(length_line[:-2])

                if length == -1:  # Null
                    return first_byte + length_line

                data = await reader.readexactly(length + 2)  # +2 for \r\n
                return first_byte + length_line + data

            elif first_byte == b'*':  # Array
                count_line = await reader.readline()
                count = int(count_line[:-2])

                if count == -1:  # Null array
                    return first_byte + count_line

                result = first_byte + count_line
                for _ in range(count):
                    element = await self.read_redis_response(reader)
                    result += element

                return result

            else:
                logger.error(f"Unknown RESP type: {first_byte}")
                return b'-ERR unknown response type\r\n'

        except Exception as e:
            logger.error(f"Error reading Redis response: {e}")
            return b'-ERR error reading response\r\n'

    def create_error_response(self, message):
        """Créer une réponse d'erreur RESP"""
        return f"-{message}\r\n".encode()

    def write_audit_log(self, audit_event):
        """
        Écrire un événement d'audit
        Format : JSON (un événement par ligne)
        """
        try:
            # Log vers fichier (JSON Lines format)
            audit_logger.info(json.dumps(audit_event))

            # Optionnel : Envoyer vers SIEM en temps réel
            # asyncio.create_task(self.send_to_siem(audit_event))

            self.stats['audit_events'] += 1

        except Exception as e:
            logger.error(f"Error writing audit log: {e}")

    async def send_to_siem(self, audit_event):
        """Envoyer un événement vers le SIEM (optionnel)"""
        try:
            # Exemple avec HTTP POST vers SIEM
            import aiohttp
            async with aiohttp.ClientSession() as session:
                async with session.post(
                    SIEM_ENDPOINT,
                    json=audit_event,
                    headers={'Content-Type': 'application/json'},
                    timeout=aiohttp.ClientTimeout(total=5)
                ) as response:
                    if response.status != 200:
                        logger.warning(f"SIEM returned {response.status}")
        except Exception as e:
            logger.error(f"Error sending to SIEM: {e}")

    async def start(self):
        """Démarrer le proxy"""
        await self.init_redis_pool()

        server = await asyncio.start_server(
            self.handle_client,
            PROXY_HOST,
            PROXY_PORT
        )

        addr = server.sockets[0].getsockname()
        logger.info(f"Redis Audit Proxy listening on {addr}")

        async with server:
            await server.serve_forever()

# Point d'entrée
if __name__ == '__main__':
    proxy = RedisAuditProxy()
    try:
        asyncio.run(proxy.start())
    except KeyboardInterrupt:
        logger.info("Proxy stopped by user")
        logger.info(f"Stats: {proxy.stats}")
```

**Déploiement du proxy :**

```bash
# 1. Installer les dépendances
pip3 install aioredis aiohttp

# 2. Créer un service systemd
cat > /etc/systemd/system/redis-audit-proxy.service <<EOF
[Unit]
Description=Redis Audit Proxy
After=network.target redis.service

[Service]
Type=simple
User=redis-proxy
Group=redis-proxy
ExecStart=/usr/local/bin/redis-audit-proxy.py
Restart=always
RestartSec=5

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/redis

[Install]
WantedBy=multi-user.target
EOF

# 3. Créer l'utilisateur
useradd -r -s /bin/false redis-proxy

# 4. Permissions
chown redis-proxy:redis-proxy /var/log/redis/audit.log
chmod 640 /var/log/redis/audit.log

# 5. Démarrer
systemctl enable redis-audit-proxy
systemctl start redis-audit-proxy

# 6. Vérifier
systemctl status redis-audit-proxy
tail -f /var/log/redis/audit.log
```

**Exemple de log d'audit généré :**
```json
{
  "timestamp": "2024-12-11T14:32:05.123456Z",
  "event_type": "redis_command",
  "session_id": "a7f3c9d1e5b2f4a8",
  "client_ip": "192.168.1.10",
  "client_port": 52341,
  "command": "GET",
  "argc": 2,
  "args": ["user:12345:email"],
  "sensitive": true,
  "command_size_bytes": 45,
  "duration_ms": 1.23,
  "success": true,
  "blocked": false,
  "proxy_version": "1.0.0"
}
```

### Solution 2 : Instrumentation applicative

**Principe :** Logger les accès Redis dans l'application elle-même.

**Avantages :**
- ✅ Contexte applicatif riche (user_id, transaction_id, etc.)
- ✅ Pas d'infrastructure supplémentaire
- ✅ Filtrage intelligent possible

**Inconvénients :**
- ❌ Nécessite modification de chaque application
- ❌ Peut être contourné (si accès direct à Redis)
- ❌ Overhead de développement

**Implémentation Python (wrapper Redis) :**

```python
import redis
import logging
import json
import time
from datetime import datetime
from functools import wraps

# Configuration
audit_logger = logging.getLogger('redis.audit')
audit_handler = logging.FileHandler('/var/log/app/redis_audit.log')
audit_handler.setFormatter(logging.Formatter('%(message)s'))
audit_logger.addHandler(audit_handler)
audit_logger.setLevel(logging.INFO)

def audit_redis_command(func):
    """
    Décorateur pour auditer les commandes Redis
    """
    @wraps(func)
    def wrapper(self, *args, **kwargs):
        # Début chrono
        start_time = time.time()

        # Contexte
        command = func.__name__.upper()

        # Exécuter la commande
        try:
            result = func(self, *args, **kwargs)
            success = True
            error = None
        except Exception as e:
            result = None
            success = False
            error = str(e)
            raise
        finally:
            # Durée
            duration_ms = (time.time() - start_time) * 1000

            # Créer l'événement d'audit
            audit_event = {
                'timestamp': datetime.utcnow().isoformat() + 'Z',
                'command': command,
                'args': self._redact_args(command, args),
                'success': success,
                'duration_ms': round(duration_ms, 2),
                'error': error,
                # Contexte applicatif
                'user_id': getattr(self, 'current_user_id', None),
                'session_id': getattr(self, 'current_session_id', None),
                'request_id': getattr(self, 'current_request_id', None),
                'ip_address': getattr(self, 'current_ip_address', None),
            }

            # Logger
            audit_logger.info(json.dumps(audit_event))

        return result

    return wrapper

class AuditedRedisClient:
    """
    Client Redis avec audit logging automatique
    """

    def __init__(self, *args, **kwargs):
        self.redis = redis.Redis(*args, **kwargs)

        # Contexte applicatif (à définir avant chaque opération)
        self.current_user_id = None
        self.current_session_id = None
        self.current_request_id = None
        self.current_ip_address = None

    def set_context(self, user_id=None, session_id=None, request_id=None, ip_address=None):
        """Définir le contexte applicatif pour l'audit"""
        self.current_user_id = user_id
        self.current_session_id = session_id
        self.current_request_id = request_id
        self.current_ip_address = ip_address

    def _redact_args(self, command, args):
        """Redacter les arguments sensibles"""
        if command in ('SET', 'SETEX', 'HSET'):
            # Ne logger que la clé, pas la valeur
            return [str(args[0]), '<REDACTED>'] + [str(a) for a in args[2:]]
        else:
            return [str(a) for a in args[:5]]  # Limiter à 5 args

    @audit_redis_command
    def get(self, key):
        return self.redis.get(key)

    @audit_redis_command
    def set(self, key, value, *args, **kwargs):
        return self.redis.set(key, value, *args, **kwargs)

    @audit_redis_command
    def delete(self, *keys):
        return self.redis.delete(*keys)

    @audit_redis_command
    def hget(self, name, key):
        return self.redis.hget(name, key)

    @audit_redis_command
    def hset(self, name, key, value):
        return self.redis.hset(name, key, value)

    # ... Ajouter toutes les méthodes Redis à auditer

# Utilisation dans l'application
redis_client = AuditedRedisClient(host='localhost', port=6379)

# Dans le handler de requête HTTP
def handle_user_request(request):
    # Définir le contexte
    redis_client.set_context(
        user_id=request.user.id,
        session_id=request.session.session_key,
        request_id=request.request_id,
        ip_address=request.META['REMOTE_ADDR']
    )

    # Les opérations Redis sont maintenant auditées avec contexte
    user_data = redis_client.get(f'user:{request.user.id}:profile')

    # ...
```

### Solution 3 : Redis Enterprise (commercial)

**Redis Enterprise** propose un audit logging natif.

**Fonctionnalités :**
- ✅ Audit logging complet (qui, quoi, quand)
- ✅ Filtrage par commande, utilisateur, database
- ✅ Export vers syslog, SIEM
- ✅ Performance minimale (<5% overhead)
- ✅ Conformité PCI DSS, HIPAA certifiée

**Configuration (Redis Enterprise) :**

```bash
# Via rladmin CLI
rladmin cluster config audit_protocol enable
rladmin cluster config audit_address syslog://siem.example.com:514
rladmin cluster config audit_protocol_filter "SET,GET,DEL,HSET,HGET,HDEL"

# Via API REST
curl -k -u "admin@example.com:password" \
  -X PUT \
  -H "Content-Type: application/json" \
  -d '{
    "audit_protocol": {
      "enabled": true,
      "protocol": "syslog",
      "address": "siem.example.com:514",
      "filter": "SET,GET,DEL,HSET,HGET,HDEL",
      "format": "json"
    }
  }' \
  https://localhost:9443/v1/cluster/audit
```

**Format des logs Redis Enterprise :**
```json
{
  "timestamp": "2024-12-11T14:32:05.123Z",
  "cluster_id": "cluster-1",
  "database_id": "db-1",
  "user": "alice",
  "client_ip": "192.168.1.10",
  "command": "GET",
  "key": "user:12345:email",
  "success": true,
  "latency_ms": 0.95
}
```

---

## Contenu des logs d'audit

### Événements obligatoires à logger

#### 1. Accès aux données (PCI DSS 10.2.1)

```json
{
  "event_category": "data_access",
  "timestamp": "2024-12-11T14:32:05.123Z",
  "user": "app_user_alice",
  "user_id": "12345",
  "client_ip": "192.168.1.10",
  "command": "GET",
  "key_pattern": "user:*:email",
  "key_accessed": "user:12345:email",
  "data_category": "PII",
  "success": true,
  "duration_ms": 1.23,
  "session_id": "sess_abc123",
  "request_id": "req_xyz789"
}
```

**Champs requis :**
- `timestamp` : ISO 8601 avec timezone (UTC recommandé)
- `user` : Identifiant unique de l'utilisateur/application
- `client_ip` : Adresse IP source
- `command` : Commande Redis exécutée
- `key_accessed` : Clé(s) accédée(s)
- `success` : Booléen (succès ou échec)

**Champs recommandés :**
- `data_category` : Classification de la donnée (PII, PHI, PCI, Public)
- `session_id` : Identifiant de session pour corrélation
- `request_id` : Identifiant de requête end-to-end
- `duration_ms` : Durée d'exécution (détection anomalies)

#### 2. Actions privilégiées (PCI DSS 10.2.2)

```json
{
  "event_category": "privileged_action",
  "timestamp": "2024-12-11T14:35:10.456Z",
  "user": "admin_john",
  "role": "redis_admin",
  "client_ip": "10.0.1.5",
  "command": "CONFIG",
  "subcommand": "SET",
  "parameter": "maxmemory",
  "old_value": "4gb",
  "new_value": "8gb",
  "success": true,
  "authorized_by": "change_request_CR-2024-1234"
}
```

**Commandes privilégiées à logger :**
- `CONFIG SET/GET/REWRITE`
- `ACL SETUSER/DELUSER/SAVE`
- `SHUTDOWN`
- `DEBUG`
- `SCRIPT FLUSH/LOAD`
- `BGREWRITEAOF`
- `BGSAVE`
- `REPLICAOF`
- `CLUSTER *`

#### 3. Tentatives d'accès échouées (PCI DSS 10.2.4)

```json
{
  "event_category": "failed_access",
  "timestamp": "2024-12-11T14:40:15.789Z",
  "user": "unknown_user",
  "client_ip": "203.0.113.45",
  "command": "AUTH",
  "failure_reason": "invalid_password",
  "attempt_count": 3,
  "lockout_triggered": false,
  "source_country": "CN",
  "threat_score": 75
}
```

**Types d'échecs à logger :**
- Authentification échouée (AUTH, ACL)
- Permission refusée (ACL)
- Commande inexistante
- Syntaxe invalide
- Timeout
- Connexion refusée (maxclients)

#### 4. Modifications d'authentification (PCI DSS 10.2.5)

```json
{
  "event_category": "auth_modification",
  "timestamp": "2024-12-11T15:00:00.123Z",
  "admin_user": "admin_sarah",
  "action": "acl_user_created",
  "target_user": "app_service_xyz",
  "permissions": ["+@read", "+@write", "~keys:*"],
  "password_changed": true,
  "mfa_enabled": false
}
```

**Événements d'authentification :**
- Création d'utilisateur ACL
- Modification de permissions ACL
- Suppression d'utilisateur
- Changement de mot de passe (`CONFIG SET requirepass`)
- Rotation de credentials

#### 5. Démarrage/Arrêt du système (PCI DSS 10.2.6)

```json
{
  "event_category": "system_lifecycle",
  "timestamp": "2024-12-11T16:00:00.000Z",
  "event": "redis_restart",
  "initiated_by": "systemd",
  "reason": "configuration_change",
  "previous_uptime_seconds": 86400,
  "config_changes": ["maxmemory", "tls-port"],
  "data_persisted": true,
  "rdb_last_save": "2024-12-11T15:59:55.000Z"
}
```

#### 6. Opérations sur les objets système (PCI DSS 10.2.7)

```json
{
  "event_category": "system_object",
  "timestamp": "2024-12-11T17:00:00.000Z",
  "user": "admin_mike",
  "action": "database_flush",
  "command": "FLUSHDB",
  "database_id": 0,
  "keys_deleted": 15234,
  "memory_freed_mb": 512,
  "confirmed": true,
  "change_request": "CR-2024-5678"
}
```

**Opérations critiques :**
- `FLUSHDB` / `FLUSHALL`
- Création/suppression de database
- Modification de la réplication (`REPLICAOF`)
- Modifications du cluster (`CLUSTER ADDSLOTS`, `CLUSTER FAILOVER`)

---

## Rétention et archivage des logs

### Politiques de rétention réglementaires

```
┌──────────────────────────────────────────────────────────────────┐
│ Réglementation │ Durée minimale  │ Disponibilité immédiate       │
├────────────────┼─────────────────┼───────────────────────────────┤
│ PCI DSS        │ 12 mois         │ 3 mois                        │
│ HIPAA          │ 6 ans           │ N/A (à définir)               │
│ SOC 2          │ 12 mois (rec.)  │ 3 mois (recommandé)           │
│ RGPD           │ Aucune (rec.)   │ 12 mois (CNIL)                │
│ ISO 27001      │ Selon politique │ Selon besoin                  │
└──────────────────────────────────────────────────────────────────┘
```

### Stratégie de rétention en 3 tiers

**Tier 1 : Hot Storage (0-90 jours)**
```
- Stockage : Disque SSD local ou distribué
- Format : JSON Lines
- Indexation : Elasticsearch, Splunk
- Accès : Temps réel, recherche full-text
- Performance : < 1 seconde pour requêtes
- Coût : Élevé (SSD, infrastructure SIEM)
```

**Tier 2 : Warm Storage (90 jours - 12 mois)**
```
- Stockage : Object storage (S3, Azure Blob)
- Format : Compressed JSON (gzip, lz4)
- Indexation : Partielle (métadonnées uniquement)
- Accès : Minutes à heures
- Performance : Recherche par date/utilisateur uniquement
- Coût : Moyen (S3 Standard ou Infrequent Access)
```

**Tier 3 : Cold Storage (12 mois - 6 ans)**
```
- Stockage : Archivage (S3 Glacier, Azure Archive)
- Format : Tar.gz par mois
- Indexation : Aucune (restauration complète requise)
- Accès : Heures à jours
- Performance : Export complet puis recherche locale
- Coût : Très faible (quelques $ par TB/mois)
```

### Implémentation de la rotation

**Script de rotation et archivage :**

```bash
#!/bin/bash
#
# Redis Audit Log Rotation and Archival
# Implémente la stratégie 3-tier : Hot → Warm → Cold
#

set -e

# Configuration
AUDIT_LOG="/var/log/redis/audit.log"
HOT_DIR="/var/log/redis/audit/hot"
WARM_DIR="/mnt/nfs/redis-audit/warm"
S3_BUCKET="s3://company-audit-logs/redis"
S3_GLACIER_BUCKET="s3://company-audit-archive/redis"

# Durées de rétention (jours)
HOT_RETENTION=90
WARM_RETENTION=365
COLD_RETENTION=2190  # 6 ans

# Rotation quotidienne (appelé par cron à 00:05)
rotate_daily() {
    local date=$(date +%Y-%m-%d)
    local archive_file="${HOT_DIR}/audit-${date}.log.gz"

    echo "[$(date)] Starting daily rotation"

    # 1. Compresser le log actuel
    if [ -f "$AUDIT_LOG" ]; then
        gzip -c "$AUDIT_LOG" > "$archive_file"
        echo "[$(date)] Compressed to $archive_file ($(du -h $archive_file | cut -f1))"

        # 2. Truncate le log actuel
        : > "$AUDIT_LOG"

        # 3. Vérifier l'intégrité
        if gzip -t "$archive_file"; then
            echo "[$(date)] Integrity check passed"
        else
            echo "[$(date)] ERROR: Archive corrupted!" >&2
            exit 1
        fi

        # 4. Calculer le checksum
        sha256sum "$archive_file" > "${archive_file}.sha256"
    fi
}

# Tier 1 → Tier 2 (Hot → Warm)
move_to_warm() {
    echo "[$(date)] Moving old hot logs to warm storage"

    find "$HOT_DIR" -name "audit-*.log.gz" -mtime +$HOT_RETENTION | while read -r file; do
        local basename=$(basename "$file")
        local warm_file="${WARM_DIR}/${basename}"

        # Copier vers warm storage
        cp "$file" "$warm_file"
        cp "${file}.sha256" "${warm_file}.sha256"

        # Vérifier la copie
        if sha256sum -c "${warm_file}.sha256"; then
            echo "[$(date)] Moved to warm: $basename"
            rm -f "$file" "${file}.sha256"
        else
            echo "[$(date)] ERROR: Warm copy verification failed for $basename" >&2
        fi
    done
}

# Tier 2 → Tier 3 (Warm → Cold/Glacier)
move_to_cold() {
    echo "[$(date)] Archiving warm logs to Glacier"

    # Grouper par mois
    for year_month in $(find "$WARM_DIR" -name "audit-*.log.gz" -mtime +$WARM_RETENTION | \
                        sed 's/.*audit-\([0-9]\{4\}-[0-9]\{2\}\)-.*/\1/' | sort -u); do

        local archive_name="audit-${year_month}.tar.gz"
        local temp_archive="/tmp/${archive_name}"

        echo "[$(date)] Creating monthly archive: $archive_name"

        # Créer l'archive mensuelle
        tar -czf "$temp_archive" -C "$WARM_DIR" $(find "$WARM_DIR" -name "audit-${year_month}-*.log.gz" -printf "%f\n")

        # Upload vers S3 Glacier
        aws s3 cp "$temp_archive" "${S3_GLACIER_BUCKET}/${archive_name}" \
            --storage-class GLACIER \
            --metadata retention-until="$(date -d "+$COLD_RETENTION days" +%Y-%m-%d)"

        if [ $? -eq 0 ]; then
            echo "[$(date)] Archived to Glacier: $archive_name"

            # Supprimer les fichiers sources (après vérification)
            find "$WARM_DIR" -name "audit-${year_month}-*.log.gz" -delete
            find "$WARM_DIR" -name "audit-${year_month}-*.sha256" -delete

            rm -f "$temp_archive"
        else
            echo "[$(date)] ERROR: Glacier upload failed for $archive_name" >&2
        fi
    done
}

# Purge des archives expirées (Tier 3)
purge_expired() {
    echo "[$(date)] Purging expired cold archives"

    # Lister les archives et vérifier la date de rétention
    aws s3api list-objects-v2 --bucket "${S3_GLACIER_BUCKET#s3://}" | \
    jq -r '.Contents[] | select(.StorageClass == "GLACIER") | .Key' | while read -r key; do

        # Récupérer les métadonnées
        retention_until=$(aws s3api head-object \
            --bucket "${S3_GLACIER_BUCKET#s3://}" \
            --key "$key" \
            --query 'Metadata."retention-until"' \
            --output text)

        # Vérifier si expiré
        if [ "$(date +%s)" -gt "$(date -d "$retention_until" +%s)" ]; then
            echo "[$(date)] Deleting expired archive: $key"
            aws s3 rm "${S3_GLACIER_BUCKET}/${key}"
        fi
    done
}

# Statistiques
show_stats() {
    echo ""
    echo "=== Redis Audit Log Statistics ==="
    echo "Hot storage (0-90d):   $(du -sh $HOT_DIR | cut -f1)"
    echo "Warm storage (90-365d): $(du -sh $WARM_DIR | cut -f1)"
    echo "Cold storage (365d+):  $(aws s3 ls ${S3_GLACIER_BUCKET}/ --recursive --summarize | grep 'Total Size' | awk '{print $3}')"
    echo ""
    echo "Total events today:    $(zcat ${HOT_DIR}/audit-$(date +%Y-%m-%d).log.gz 2>/dev/null | wc -l)"
    echo "Total hot events:      $(zcat ${HOT_DIR}/audit-*.log.gz 2>/dev/null | wc -l)"
    echo "=================================="
}

# Point d'entrée
case "${1:-rotate}" in
    rotate)
        rotate_daily
        ;;
    move-warm)
        move_to_warm
        ;;
    move-cold)
        move_to_cold
        ;;
    purge)
        purge_expired
        ;;
    stats)
        show_stats
        ;;
    all)
        rotate_daily
        move_to_warm
        move_to_cold
        show_stats
        ;;
    *)
        echo "Usage: $0 {rotate|move-warm|move-cold|purge|stats|all}"
        exit 1
        ;;
esac

echo "[$(date)] Done"
```

**Configuration cron :**
```bash
# /etc/cron.d/redis-audit-rotation

# Rotation quotidienne à 00:05
5 0 * * * redis /usr/local/bin/redis-audit-rotate.sh rotate

# Migration Hot → Warm hebdomadaire (dimanche 01:00)
0 1 * * 0 redis /usr/local/bin/redis-audit-rotate.sh move-warm

# Migration Warm → Cold mensuelle (1er du mois 02:00)
0 2 1 * * redis /usr/local/bin/redis-audit-rotate.sh move-cold

# Purge des archives expirées (trimestrielle)
0 3 1 1,4,7,10 * redis /usr/local/bin/redis-audit-rotate.sh purge

# Statistiques hebdomadaires (lundi 08:00)
0 8 * * 1 redis /usr/local/bin/redis-audit-rotate.sh stats | mail -s "Redis Audit Stats" ops@example.com
```

---

## Intégration SIEM

### Centralisation des logs

**Flux de données :**
```
[Redis Audit Proxy] → [Syslog/Filebeat] → [Logstash] → [Elasticsearch] → [Kibana]
                          ↓
                    [Splunk/Datadog/Sentinel]
```

### Configuration Filebeat (Elastic Stack)

```yaml
# /etc/filebeat/filebeat.yml

filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/redis/audit.log

  # Parser JSON
  json.keys_under_root: true
  json.add_error_key: true

  # Champs additionnels
  fields:
    log_type: redis_audit
    environment: production
    datacenter: eu-west-1
    compliance: pci-dss

  # Multiline (si nécessaire)
  multiline.type: pattern
  multiline.pattern: '^\{'
  multiline.negate: true
  multiline.match: after

  # Enrichissement
  processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~

  # Filtrage (exclure les commandes non-sensibles si volume élevé)
  #exclude_lines: ['^.*"command":"PING".*$']

# Output vers Logstash
output.logstash:
  hosts: ["logstash-1.internal:5044", "logstash-2.internal:5044"]
  loadbalance: true
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
  ssl.certificate: "/etc/filebeat/client.crt"
  ssl.key: "/etc/filebeat/client.key"

# Monitoring
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["https://elasticsearch.internal:9200"]
  username: "filebeat_monitor"
  password: "${FILEBEAT_MONITOR_PASSWORD}"

# Logging
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

### Pipeline Logstash

```ruby
# /etc/logstash/conf.d/redis-audit.conf

input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/server.crt"
    ssl_key => "/etc/logstash/certs/server.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
  }
}

filter {
  # Filter sur les logs Redis audit uniquement
  if [fields][log_type] == "redis_audit" {

    # Parser JSON (si pas déjà fait par Filebeat)
    if [message] =~ /^\{/ {
      json {
        source => "message"
        target => "audit"
      }
    }

    # Enrichissement IP geolocation
    if [audit][client_ip] {
      geoip {
        source => "[audit][client_ip]"
        target => "geoip"
      }
    }

    # Classification de la sensibilité
    if [audit][command] in ["GET", "SET", "HGET", "HSET", "DEL", "HDEL"] {
      mutate {
        add_field => { "sensitivity" => "high" }
      }
    } else {
      mutate {
        add_field => { "sensitivity" => "low" }
      }
    }

    # Détection d'anomalies (durée)
    if [audit][duration_ms] and [audit][duration_ms] > 100 {
      mutate {
        add_tag => [ "slow_query" ]
      }
    }

    # Détection d'échecs
    if [audit][success] == false or [audit][blocked] == true {
      mutate {
        add_tag => [ "failure" ]
      }
    }

    # Détection de patterns suspects
    if [audit][command] == "KEYS" and [audit][args][0] == "*" {
      mutate {
        add_tag => [ "dangerous_pattern" ]
      }
    }

    # Timestamp parsing
    date {
      match => [ "[audit][timestamp]", "ISO8601" ]
      target => "@timestamp"
    }

    # Supprimer le message original (déjà parsé)
    mutate {
      remove_field => [ "message" ]
    }
  }
}

output {
  # Output vers Elasticsearch
  if [fields][log_type] == "redis_audit" {
    elasticsearch {
      hosts => ["https://elasticsearch-1.internal:9200", "https://elasticsearch-2.internal:9200"]
      index => "redis-audit-%{+YYYY.MM.dd}"
      user => "logstash_writer"
      password => "${LOGSTASH_ES_PASSWORD}"
      ssl => true
      cacert => "/etc/logstash/certs/ca.crt"

      # ILM policy pour retention automatique
      ilm_enabled => true
      ilm_rollover_alias => "redis-audit"
      ilm_pattern => "{now/d}-000001"
      ilm_policy => "redis-audit-policy"
    }

    # Output vers Splunk (optionnel, si double SIEM)
    #http {
    #  url => "https://splunk.example.com:8088/services/collector/event"
    #  headers => {
    #    "Authorization" => "Splunk ${SPLUNK_HEC_TOKEN}"
    #  }
    #  format => "json"
    #  ssl_certificate_validation => true
    #}
  }

  # Alerting temps réel (failures critiques)
  if "failure" in [tags] and [audit][command] in ["AUTH", "ACL"] {
    email {
      to => "security@example.com"
      from => "redis-audit@example.com"
      subject => "ALERT: Redis authentication failure"
      body => "Failure detected: %{[audit]}"
      address => "smtp.example.com"
      port => 587
    }
  }
}
```

### Index Lifecycle Management (ILM)

```json
{
  "policy": "redis-audit-policy",
  "phases": {
    "hot": {
      "min_age": "0ms",
      "actions": {
        "rollover": {
          "max_size": "50GB",
          "max_age": "1d"
        },
        "set_priority": {
          "priority": 100
        }
      }
    },
    "warm": {
      "min_age": "90d",
      "actions": {
        "shrink": {
          "number_of_shards": 1
        },
        "forcemerge": {
          "max_num_segments": 1
        },
        "set_priority": {
          "priority": 50
        }
      }
    },
    "cold": {
      "min_age": "365d",
      "actions": {
        "freeze": {},
        "set_priority": {
          "priority": 0
        }
      }
    },
    "delete": {
      "min_age": "2190d",
      "actions": {
        "delete": {}
      }
    }
  }
}
```

---

## Checklist de conformité

### Configuration et déploiement

```
Audit Logging Infrastructure :
□ Solution d'audit implémentée (proxy, instrumentation, Redis Enterprise)
□ Logs centralisés dans un SIEM
□ Tous les Redis instances loggés (prod, staging si données sensibles)
□ TLS activé entre composants (Filebeat, Logstash, Elasticsearch)
□ Authentification configurée (SIEM, visualisations)

Contenu des logs :
□ Timestamp UTC avec précision (ms)
□ Identification utilisateur (username ou app_id)
□ IP source
□ Commande exécutée
□ Arguments (avec redaction si sensible)
□ Succès/Échec
□ Durée d'exécution
□ Session ID et Request ID (corrélation)

Événements loggés :
□ Tous les accès aux données sensibles (PII, PHI, PCI)
□ Toutes les actions privilégiées (CONFIG, ACL, CLUSTER)
□ Tentatives d'authentification (succès et échecs)
□ Modifications de configuration
□ Démarrage/Arrêt du serveur
□ Opérations destructrices (FLUSHDB, DEL)
□ Échecs ACL
```

### Rétention et archivage

```
Stratégie de rétention :
□ Politique documentée et approuvée
□ Durées conformes (PCI: 12 mois, HIPAA: 6 ans, etc.)
□ Tier 1 (Hot) : 0-90 jours, accès immédiat
□ Tier 2 (Warm) : 90-365 jours, accès rapide
□ Tier 3 (Cold) : 365+ jours, archivage Glacier/Tape
□ Rotation automatisée (cron, scripts testés)
□ Vérification d'intégrité (checksums)
□ Chiffrement des archives (GPG, S3 SSE)

Backup et résilience :
□ Logs sauvegardés dans 2+ locations géographiques
□ Tests de restauration mensuels
□ RTO/RPO documentés
□ Plan de reprise d'activité (DRP)
```

### Sécurité des logs

```
Protection :
□ Logs read-only pour non-admins
□ Accès aux logs tracé (audit de l'audit)
□ Intégrité protégée (write-once storage ou signature)
□ Logs non modifiables (append-only)
□ Ségrégation des accès (principe du moindre privilège)
□ Chiffrement at-rest (filesystem ou storage)
□ Chiffrement in-transit (TLS pour Filebeat, Logstash)

Compliance :
□ Synchronisation NTP (<1 seconde de précision)
□ Timezone UTC pour tous les systèmes
□ Pas de données sensibles en clair (PANs redactés)
□ Conformité RGPD (logs = données personnelles si user_id)
```

### Revue et analyse

```
Processus de revue :
□ Revue quotidienne des logs de sécurité (PCI DSS 10.6.1)
□ Dashboards configurés (échecs, anomalies, tendances)
□ Alertes temps réel (échecs auth, commandes dangereuses)
□ Revue hebdomadaire des actions privilégiées
□ Revue mensuelle des accès (qui accède à quoi)
□ Rapport trimestriel pour le management
□ Audit annuel par tiers externe

Détection d'anomalies :
□ Baseline de comportement établi
□ Alertes sur déviations (volume, durée, patterns)
□ Machine learning (si SIEM avancé)
□ Corrélation multi-sources (logs app + Redis + système)

Forensique :
□ Capacité de reconstruction d'événements
□ Requêtes forensiques documentées (playbooks)
□ Conservation des preuves (chain of custody)
□ Exports forensiques sécurisés
```

### Documentation et formation

```
□ Politique d'audit logging documentée et approuvée
□ Procédures opérationnelles (SOPs) pour :
  □ Rotation des logs
  □ Accès aux logs (qui, quand, comment)
  □ Réponse aux incidents (analyse logs)
  □ Restauration depuis archive
□ Runbooks pour scénarios courants
□ Formation annuelle des équipes
□ Tests réguliers (quarterly)
□ Registre des accès aux logs (audit de l'audit)
```

---

## Conclusion

L'audit logging est une exigence non-négociable pour la conformité Redis. Cette section a couvert :

- ✅ **Cadre réglementaire** complet (RGPD, PCI DSS, HIPAA, SOC 2, ISO 27001)
- ✅ **Solutions d'audit** : Proxy custom (Python complet), Envoy, Redis Enterprise
- ✅ **Contenu des logs** : 6 catégories d'événements obligatoires
- ✅ **Rétention** : Stratégie 3-tier avec scripts de rotation
- ✅ **Intégration SIEM** : Filebeat, Logstash, Elasticsearch (Elastic Stack)
- ✅ **Checklists** exhaustives (60+ points de contrôle)

**Points critiques à retenir :**
1. Redis natif est INSUFFISANT pour l'audit (nécessite solutions tierces)
2. Tous les accès aux données sensibles DOIVENT être loggés
3. Rétention minimale : 12 mois (PCI DSS), 6 ans (HIPAA)
4. Centralisation SIEM obligatoire (corrélation, alerting)
5. Revue quotidienne des logs requise (PCI DSS)
6. Les logs d'audit sont eux-mêmes des données sensibles (protection stricte)

**Prochaines étapes :**
- Implémenter une solution d'audit (proxy recommandé)
- Configurer la centralisation SIEM
- Établir la politique de rétention
- Créer les dashboards et alertes
- Former les équipes aux procédures
- Planifier les revues régulières

⏭️ [Gestion des accès et des permissions (RBAC)](/17-gouvernance-conformite/04-gestion-acces-permissions-rbac.md)
