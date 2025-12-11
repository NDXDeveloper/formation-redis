🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 17 : Gouvernance et Conformité Redis

## Vue d'ensemble du module

La mise en production de Redis dans des environnements réglementés nécessite une approche rigoureuse en matière de gouvernance et de conformité. Ce module s'adresse aux **compliance officers**, **architectes de sécurité**, **DPO** (Data Protection Officers) et **responsables de la gouvernance des données** qui doivent garantir que l'utilisation de Redis respecte les exigences réglementaires applicables.

### Contexte réglementaire

Redis, en tant que base de données in-memory stockant potentiellement des données sensibles, est soumis aux mêmes exigences de conformité que n'importe quel système de traitement de données :

- **RGPD** (Règlement Général sur la Protection des Données) pour l'UE
- **CCPA/CPRA** (California Consumer Privacy Act) pour la Californie
- **HIPAA** (Health Insurance Portability and Accountability Act) pour la santé aux États-Unis
- **PCI DSS** (Payment Card Industry Data Security Standard) pour les données de paiement
- **SOC 2** (Service Organization Control) pour les fournisseurs de services
- **ISO 27001** pour la gestion de la sécurité de l'information
- **Lois sectorielles** spécifiques (bancaire, défense, etc.)

---

## 📋 Enjeux de conformité spécifiques à Redis

### 1. Nature éphémère vs permanence des données

**Problématique :**
Redis est souvent utilisé comme cache temporaire, ce qui peut créer une **illusion de non-persistance** des données. Cependant :

- Les snapshots RDB persistent sur disque
- Les fichiers AOF contiennent l'historique complet
- Les réplicas conservent des copies des données
- Les backups peuvent contenir des données sensibles

**Implication réglementaire :**
Les données en cache sont soumises aux mêmes obligations que les données permanentes (droit à l'oubli, chiffrement, traçabilité).

### 2. Absence de contrôle d'accès natif robuste (versions < 6)

**Problématique historique :**
- Redis < 6.0 : Un seul mot de passe pour toute l'instance
- Pas de granularité par utilisateur, clé ou commande
- Difficile d'appliquer le principe du moindre privilège

**Évolution Redis 6+ :**
- ACL (Access Control Lists) granulaires
- Gestion multi-utilisateurs
- Permissions par commande et par clé
- Logs d'audit des accès

### 3. Chiffrement des données

**Deux niveaux critiques :**

#### A. Chiffrement en transit (Data in Transit)
- **Obligation :** TLS/SSL pour tous les échanges client-serveur et inter-nœuds
- **Standard :** TLS 1.2 minimum (TLS 1.3 recommandé)
- **Certificats :** Gestion du cycle de vie, rotation, révocation

#### B. Chiffrement au repos (Data at Rest)
- **RDB/AOF :** Chiffrement du filesystem ou chiffrement au niveau applicatif
- **Backups :** Stockage chiffré obligatoire
- **Réplicas :** Même niveau de protection que le master

### 4. Traçabilité et auditabilité

**Exigences réglementaires :**
- **Qui** a accédé à quelle donnée
- **Quand** l'accès a eu lieu
- **Quelle opération** a été effectuée
- **Conservation** des logs d'audit (durée réglementaire variable)

**Limitations Redis natives :**
- Pas de log d'audit intégré par défaut
- Nécessité d'implémenter des solutions tierces

### 5. Localisation géographique des données

**Souveraineté des données :**
- RGPD : Restrictions sur les transferts hors UE
- Lois de résidence des données (Russie, Chine, etc.)
- Cloud Act américain : Accès potentiel aux données

**Implications pour Redis :**
- Choix de la région de déploiement
- Réplication cross-datacenter contrôlée
- Documentation de la topologie géographique

---

## 🎯 Objectifs de conformité

### Objectifs primaires

1. **Garantir la confidentialité** des données sensibles stockées dans Redis
2. **Assurer l'intégrité** des données et prévenir les modifications non autorisées
3. **Maintenir la disponibilité** selon les SLA définis
4. **Tracer tous les accès** aux données pour l'audit et la forensique
5. **Respecter les droits** des personnes concernées (RGPD : accès, rectification, effacement)
6. **Documenter** tous les processus et procédures
7. **Former** les équipes aux bonnes pratiques de conformité

### Objectifs secondaires

8. **Minimiser la surface d'attaque** (hardening)
9. **Automatiser** la conformité via Infrastructure as Code
10. **Tester régulièrement** les contrôles de sécurité
11. **Maintenir une veille** réglementaire et technologique

---

## 📊 Matrice de responsabilité (RACI)

Pour une gouvernance efficace, il est essentiel de définir clairement les responsabilités :

| Activité | DPO | RSSI | Architecte | DevOps | Développeur | Compliance Officer |
|----------|-----|------|------------|--------|-------------|-------------------|
| Définition politique de conformité | C | C | I | I | I | **R** |
| Classification des données | **R** | A | C | I | C | C |
| Choix architecture Redis | I | C | **R** | A | C | I |
| Implémentation ACL | I | C | A | **R** | C | I |
| Configuration TLS/SSL | I | C | A | **R** | I | I |
| Audit logs configuration | C | **R** | C | A | I | A |
| Tests de pénétration | I | **R** | C | A | I | C |
| Gestion des incidents | A | **R** | C | A | I | C |
| Revue de conformité annuelle | A | C | I | I | I | **R** |
| Formation équipes | C | C | I | A | **R** | A |

**Légende :** R = Responsible (Réalise), A = Accountable (Approuve), C = Consulted (Consulté), I = Informed (Informé)

---

## ⚖️ Cadre réglementaire applicable

### 1. RGPD (Règlement Général sur la Protection des Données)

**Champ d'application :**
- Toute organisation traitant des données personnelles de résidents UE
- Sanctions jusqu'à 4% du CA mondial ou 20M€

**Articles clés pour Redis :**

#### Article 5 : Principes relatifs au traitement
- **Licéité, loyauté, transparence** : Documenter l'usage de Redis
- **Limitation des finalités** : Ne cacher que ce qui est nécessaire
- **Minimisation des données** : Éviter de stocker des PII si non nécessaire
- **Exactitude** : Mécanismes de correction
- **Limitation de la conservation** : TTL appropriés
- **Intégrité et confidentialité** : Chiffrement, ACL, audit

#### Article 17 : Droit à l'effacement ("droit à l'oubli")
**Implication Redis :**
```
Procédure d'effacement complète :
1. Suppression de la clé principale (DEL)
2. Vérification des réplicas (propagation asynchrone)
3. Purge des backups RDB/AOF selon politique de rétention
4. Suppression des logs d'audit après période réglementaire
5. Documentation de l'effacement (preuve de conformité)
```

#### Article 25 : Protection des données dès la conception
- **Privacy by Design** : Sécurité intégrée dès l'architecture
- **Privacy by Default** : Paramètres les plus protecteurs par défaut

#### Article 32 : Sécurité du traitement
- État de l'art en matière de sécurité
- Chiffrement approprié
- Tests réguliers
- Capacité à restaurer (backups)

#### Article 33 : Notification de violation (72h)
**Checklist incident Redis :**
```
□ Détection de la violation (monitoring, alertes)
□ Évaluation de la gravité (données exposées ?)
□ Containment (isolation de l'instance compromise)
□ Notification DPO/RSSI (immédiat)
□ Notification autorité de contrôle (CNIL) si < 72h
□ Notification personnes concernées si risque élevé
□ Documentation complète de l'incident
□ Analyse post-mortem et plan de remédiation
```

#### Article 35 : Analyse d'impact (DPIA)
**Quand effectuer une DPIA pour Redis ?**
- Traitement à grande échelle de données sensibles
- Surveillance systématique (ex: tracking utilisateur)
- Données de santé, biométrie, données pénales
- Scoring/profilage automatisé

---

### 2. PCI DSS (Payment Card Industry)

**Applicable si Redis stocke des données de carte bancaire**

#### Exigences critiques :

**Requirement 1 & 2 : Firewall et configurations sécurisées**
- Pas d'accès direct depuis Internet
- Changement des mots de passe par défaut
- Désactivation des services non nécessaires

**Requirement 3 : Protection des données du titulaire**
```
❌ INTERDIT de stocker en clair :
- Numéro de carte complet (PAN)
- Code CVV/CVC
- Code PIN

✅ AUTORISÉ (avec chiffrement fort) :
- PAN tronqué (6 premiers + 4 derniers chiffres)
- Token de paiement
```

**Requirement 4 : Chiffrement des transmissions**
- TLS 1.2+ obligatoire
- Certificats valides et à jour

**Requirement 8 : Identification et authentification**
- ACL Redis avec comptes nominatifs
- Pas de compte partagé
- MFA pour accès administratif

**Requirement 10 : Logs et monitoring**
```
Logs obligatoires :
- Tous les accès aux données de carte
- Actions d'utilisateurs privilégiés
- Tentatives d'accès échouées
- Modifications de configuration
- Arrêt/démarrage du service
Conservation : 3 mois en ligne, 12 mois total
```

**Requirement 11 : Tests de sécurité réguliers**
- Scans de vulnérabilités trimestriels
- Pentests annuels
- Tests après changements significatifs

---

### 3. HIPAA (Health Insurance Portability and Accountability Act)

**Applicable aux données de santé aux États-Unis**

#### Security Rule - Safeguards administratifs

**§164.308(a)(1) : Évaluation des risques**
```
Checklist évaluation Redis :
□ Identification des PHI (Protected Health Information) stockées
□ Analyse des menaces (accès non autorisé, perte, altération)
□ Évaluation de la probabilité et de l'impact
□ Documentation des mesures de sécurité
□ Plan de gestion des risques
```

**§164.308(a)(3) : Gestion des accès**
- Autorisations formelles (qui peut accéder à quoi)
- Revue périodique des droits d'accès
- Procédures de terminaison (départ d'un employé)

**§164.308(a)(5) : Audit et logs**
- Conservation 6 ans minimum
- Revue régulière des logs d'accès

#### Security Rule - Safeguards physiques

**§164.310(d) : Contrôles des accès aux postes de travail**
- Workstations autorisés pour accéder à Redis
- Journalisation des accès

#### Security Rule - Safeguards techniques

**§164.312(a)(1) : Contrôle d'accès unique**
- Comptes utilisateurs uniques (ACL)
- Mécanisme d'urgence (accès break-glass documenté)

**§164.312(c)(1) : Intégrité**
- Checksums RDB/AOF
- Protection contre altération malveillante

**§164.312(d) : Authentification des personnes**
- Authentification forte
- Pas de credentials partagés

**§164.312(e)(1) : Chiffrement des transmissions**
- TLS obligatoire
- VPN pour accès distants

#### Breach Notification Rule

**Délai : 60 jours** pour notifier les personnes affectées en cas de violation

---

### 4. SOC 2 (Service Organization Control)

**Applicable aux fournisseurs de services cloud/SaaS**

#### Trust Services Criteria

**CC6.1 : Contrôles d'accès logiques**
```
Contrôles Redis SOC 2 :
□ ACL configurées selon le principe du moindre privilège
□ Authentification forte (pas de passwordless)
□ Comptes de service distincts des comptes utilisateurs
□ Rotation régulière des credentials
□ MFA pour les accès privilégiés
□ Timeout des sessions inactives
```

**CC6.6 : Chiffrement**
- TLS pour toutes les connexions
- Chiffrement at-rest via filesystem chiffré
- Gestion sécurisée des clés de chiffrement (HSM, KMS)

**CC6.7 : Protection contre les menaces**
- Firewall configuré (allow-list uniquement)
- IDS/IPS en place
- Scan de vulnérabilités réguliers
- Patching dans les SLA définis

**CC7.2 : Monitoring et détection d'incidents**
```
Métriques SOC 2 pour Redis :
- Tentatives d'authentification échouées
- Commandes interdites exécutées
- Pics d'utilisation anormaux
- Accès depuis IP non autorisées
- Modifications de configuration non planifiées
```

**CC8.1 : Gestion des changements**
- Change management pour toute modification Redis
- Tests avant déploiement en production
- Rollback plan documenté

---

### 5. ISO 27001

**Norme internationale de gestion de la sécurité**

#### Annexe A - Contrôles applicables à Redis

**A.9 : Contrôle d'accès**
- A.9.1.1 : Politique de contrôle d'accès documentée
- A.9.2.1 : Enregistrement et désenregistrement des utilisateurs (procédure ACL)
- A.9.4.1 : Restriction de l'accès à l'information (ACL par clé/pattern)

**A.10 : Cryptographie**
- A.10.1.1 : Politique d'utilisation des contrôles cryptographiques
- A.10.1.2 : Gestion des clés (rotation, stockage sécurisé)

**A.12 : Sécurité des opérations**
- A.12.1.2 : Gestion des changements (processus formel pour Redis)
- A.12.4.1 : Journalisation des événements (audit logs Redis)
- A.12.4.3 : Logs administrateurs (traçabilité des actions privileged)

**A.17 : Continuité d'activité**
- A.17.1.2 : Plan de continuité incluant Redis (RPO/RTO)
- A.17.2.1 : Disponibilité des services (HA, clustering, backups)

---

## 🛡️ Posture de sécurité globale

### Défense en profondeur (Defense in Depth)

L'approche de conformité Redis doit s'inscrire dans une stratégie multi-couches :

```
┌─────────────────────────────────────────────────────────┐
│ Niveau 7 : Gouvernance et Politiques                    │
│ • Politiques de sécurité documentées                    │
│ • Formation et sensibilisation                          │
│ • Audits réguliers                                      │
├─────────────────────────────────────────────────────────┤
│ Niveau 6 : Surveillance et Détection                    │
│ • SIEM centralisé                                       │
│ • Alertes temps réel                                    │
│ • Threat intelligence                                   │
├─────────────────────────────────────────────────────────┤
│ Niveau 5 : Applicatif                                   │
│ • Validation des entrées                                │
│ • Gestion sécurisée des sessions                        │
│ • Rate limiting                                         │
├─────────────────────────────────────────────────────────┤
│ Niveau 4 : Redis - Contrôles applicatifs                │
│ • ACL granulaires                                       │
│ • Command renaming/disabling                            │
│ • Lua scripts validés                                   │
├─────────────────────────────────────────────────────────┤
│ Niveau 3 : Redis - Sécurité des données                 │
│ • TLS/SSL                                               │
│ • Chiffrement at-rest                                   │
│ • Backups chiffrés                                      │
├─────────────────────────────────────────────────────────┤
│ Niveau 2 : Réseau et Système                            │
│ • Firewall (iptables, Security Groups)                  │
│ • Segmentation réseau (VPC, VLAN)                       │
│ • Bastion hosts / Jump servers                          │
│ • Hardening OS (SELinux, AppArmor)                      │
├─────────────────────────────────────────────────────────┤
│ Niveau 1 : Physique et Infrastructure                   │
│ • Contrôle d'accès datacenter                           │
│ • Redondance hardware                                   │
│ • Protection électrique                                 │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Documentation requise pour la conformité

### 1. Inventaire des traitements

**Registre des activités de traitement (Article 30 RGPD)**

```markdown
## Traitement Redis - [NOM DU SYSTÈME]

**Responsable du traitement :** [Société]
**Finalité :** [Ex: Cache session utilisateur]
**Base légale :** [Consentement / Contrat / Intérêt légitime]

**Catégories de données personnelles :**
- Identifiants (user_id, session_id)
- Données de connexion (IP, timestamp)
- [Autres données...]

**Catégories de personnes concernées :**
- Utilisateurs authentifiés du service

**Destinataires des données :**
- Application web (backend)
- Équipe DevOps (support niveau 3)

**Transferts hors UE :** Non / Oui [Préciser pays et garanties]

**Durée de conservation :**
- En cache : 24 heures (TTL automatique)
- Backups : 30 jours
- Logs d'audit : 12 mois

**Mesures de sécurité :**
- Chiffrement TLS 1.3
- ACL limitant l'accès aux seules applications autorisées
- Monitoring et alerting
- Backups chiffrés (AES-256)
- Accès restreint par VPC

**DPIA effectuée :** Oui / Non [Si oui, référence]
**Date de dernière revue :** [Date]
```

### 2. Cartographie des flux de données

**Diagramme data flow obligatoire**

```
[Client Browser]
    ↓ HTTPS
[Load Balancer]
    ↓ HTTPS
[Application Server] ← Auth
    ↓ TLS 1.3
[Redis Cluster]
    ├─ Master (Write)
    ├─ Replica 1 (Read)
    └─ Replica 2 (Read)
    ↓ Encrypted Backup
[S3 Backup Bucket - Chiffré]
    ↓ Retention Policy
[Cold Archive]
```

Documenter pour chaque flux :
- Protocole et port
- Chiffrement utilisé
- Authentification requise
- Type de données transitant
- Localisation géographique source/destination

### 3. Matrice des risques

| Risque | Probabilité | Impact | Niveau | Mesures d'atténuation | Statut |
|--------|-------------|--------|--------|----------------------|--------|
| Accès non autorisé | Moyenne | Élevé | **Majeur** | ACL + TLS + Firewall | ✅ Traité |
| Perte de données (crash) | Faible | Élevé | Modéré | RDB + AOF + Réplication | ✅ Traité |
| Violation de données | Faible | Critique | **Majeur** | Chiffrement + Monitoring | ✅ Traité |
| Fuite via backup non sécurisé | Moyenne | Critique | **Majeur** | Chiffrement S3 + IAM | ✅ Traité |
| Déni de service (DoS) | Moyenne | Moyen | Modéré | Rate limiting + maxclients | ⚠️ Partiel |
| Exfiltration par insider | Faible | Élevé | Modéré | RBAC + Audit logs + DLP | ✅ Traité |

### 4. Politique de classification des données

**Définir ce qui peut être stocké dans Redis selon le niveau de sensibilité**

| Niveau | Type de données | Autorisé dans Redis ? | Conditions |
|--------|----------------|----------------------|------------|
| **Public** | Données publiques | ✅ Oui | Aucune restriction |
| **Interne** | Données métier non sensibles | ✅ Oui | ACL de base |
| **Confidentiel** | PII, données business sensibles | ⚠️ Avec restrictions | TLS + ACL strictes + TTL court + Backups chiffrés |
| **Secret** | Mots de passe, tokens, cartes bancaires | ❌ Non recommandé | Si absolument nécessaire : Chiffrement applicatif + tout le reste |
| **Réglementé** | Santé, données pénales | ❌ Non sauf conformité prouvée | DPIA + toutes mesures + audit externe |

---

## ✅ Checklist de conformité Redis

### Phase 1 : Conception et architecture

```
□ Classification des données à stocker effectuée
□ DPIA réalisée si nécessaire (données sensibles, grande échelle)
□ Architecture de sécurité validée par le RSSI
□ Documentation d'architecture complète
□ Choix du mode de déploiement (on-prem, cloud managé)
□ Localisation géographique conforme aux exigences
□ Plan de réplication et backup défini
□ RPO/RTO documentés et validés
□ Budget sécurité alloué (TLS, HSM, outils audit, etc.)
```

### Phase 2 : Déploiement sécurisé

```
Système :
□ OS durci (CIS benchmark appliqué)
□ Swap désactivé (ou encrypted swap)
□ THP (Transparent Huge Pages) désactivé
□ Firewall configuré (allow-list stricte)
□ SELinux/AppArmor activé

Redis :
□ Version à jour avec patches de sécurité
□ Configuration bind sur interfaces privées uniquement
□ protected-mode activé
□ Mot de passe fort configuré (requirepass/ACL)
□ ACL définies selon principe du moindre privilège
□ Commandes dangereuses désactivées (FLUSHDB, FLUSHALL, CONFIG, etc.)
□ TLS activé (clients et inter-nœuds)
□ Certificats valides (pas auto-signés en prod)
□ Persistance configurée (RDB/AOF selon besoins)
□ Filesystem backups chiffré
□ maxmemory et politique d'éviction configurés
□ Logging activé avec niveau approprié
□ Rotation des logs configurée
```

### Phase 3 : Contrôles d'accès

```
□ Comptes nominatifs créés (pas de compte partagé)
□ Comptes de service distincts par application
□ Permissions granulaires par utilisateur (ACL)
□ MFA activée pour accès administratif
□ Bastion host / Jump server pour accès production
□ Clés SSH avec passphrase (pas de password auth)
□ Rotation des credentials planifiée (ex: trimestre)
□ Procédure de révocation en cas de départ
□ Liste des administrateurs à jour
```

### Phase 4 : Monitoring et audit

```
□ Centralisation des logs vers SIEM
□ Audit logs activés (via proxy ou module externe)
□ Alertes configurées :
  □ Authentifications échouées répétées
  □ Commandes interdites exécutées
  □ Changements de configuration
  □ Utilisation mémoire > seuil
  □ Latence > seuil
  □ Connexions depuis IP non autorisées
□ Dashboard de conformité créé
□ Rétention des logs conforme (ex: 12 mois HIPAA)
□ Tests réguliers des alertes
□ Procédure d'escalade définie
```

### Phase 5 : Continuité et résilience

```
□ Haute disponibilité configurée (Sentinel ou Cluster)
□ Backups automatisés (RDB/AOF)
□ Backups stockés dans région différente (geo-redundancy)
□ Backups chiffrés
□ Tests de restauration mensuels
□ Documentation de la procédure de restauration
□ Plan de reprise d'activité (DRP) incluant Redis
□ Runbook pour les scénarios d'incident
□ Tests de basculement (failover drills) trimestriels
```

### Phase 6 : Conformité continue

```
□ Revue annuelle de la politique de sécurité Redis
□ Audit de sécurité par tiers externe (pentesting)
□ Scan de vulnérabilités trimestriel
□ Revue des accès semestrielle
□ Formation annuelle des équipes
□ Veille sur les CVE Redis et patches appliqués < 30j
□ Documentation à jour (architecture, procédures)
□ Registre des traitements mis à jour
□ Tests d'intrusion annuels
□ Certification SOC 2 / ISO 27001 (si applicable)
```

---

## 🚨 Gestion des incidents de sécurité

### Procédure de réponse aux incidents (IRP - Incident Response Plan)

#### Phase 1 : Détection et identification

**Indicateurs de compromission (IOC) :**
- Authentifications échouées massives
- Exécution de commandes inhabituelles (KEYS *, CONFIG, SCRIPT FLUSH)
- Pics de trafic réseau inexpliqués
- Connexions depuis IP non autorisées
- Modifications non planifiées de la configuration
- Disparition ou altération de clés

**Actions immédiates :**
1. **Alerter** : RSSI, DPO, responsable infrastructure (< 15 min)
2. **Documenter** : Screenshot, logs, timestamp exact
3. **Ne pas éteindre** le système (préservation des preuves)

#### Phase 2 : Confinement

**Confinement à court terme (immediate containment) :**
```bash
# Isoler l'instance compromise du réseau
# Via firewall : bloquer tout sauf équipe sécurité
iptables -A INPUT -s <IP_TRUSTED> -j ACCEPT
iptables -A INPUT -j DROP

# Révoquer les accès suspectés
ACL DELUSER <username>

# Activer le mode lecture seule si possible
CONFIG SET replica-read-only yes
```

**Confinement à long terme :**
- Basculer le trafic vers un replica sain
- Mise en quarantaine de l'instance compromise
- Préparation d'une nouvelle instance durcie

#### Phase 3 : Éradication

```bash
# Analyse forensique avant toute action destructive
# Capturer l'état mémoire (si compétences disponibles)

# Identifier la source de la compromission
# - Credential compromise ? → Rotation immédiate de tous les credentials
# - Vulnérabilité exploitée ? → Patch d'urgence
# - Malware ? → Scan antivirus, analyse du système

# Reconstruction from secure backup
# Vérification de l'intégrité des backups (hash, signature)
```

#### Phase 4 : Récupération

```bash
# Déployer une nouvelle instance Redis
# - Version patchée
# - Configuration durcie (checklist complète)
# - Nouveaux credentials

# Restaurer depuis backup vérifié
# Valider l'intégrité des données

# Réactiver le trafic progressivement (canary deployment)
# Monitoring renforcé pendant 72h
```

#### Phase 5 : Leçons apprises (Post-Incident Review)

**Rapport d'incident obligatoire incluant :**
- Chronologie détaillée
- Cause racine (Root Cause Analysis)
- Données compromises (scope de la violation)
- Actions correctives prises
- Recommandations pour prévenir la récurrence
- Notification aux autorités si RGPD/HIPAA applicable

**Délai de notification :**
- RGPD : 72h à l'autorité de contrôle
- HIPAA : 60 jours aux personnes affectées
- PCI DSS : Immédiat aux acquéreurs (Visa, Mastercard)

---

## 📐 Modèle de gouvernance Redis

### 1. Comité de gouvernance

**Composition recommandée :**
- Chief Information Security Officer (CISO)
- Data Protection Officer (DPO)
- Architecte sécurité
- Lead DevOps
- Compliance Officer
- Représentant métier (si données critiques)

**Réunions :**
- Trimestrielles minimum
- Extraordinaire en cas d'incident majeur

**Ordre du jour type :**
1. Revue des incidents de sécurité
2. État de la conformité (tableaux de bord)
3. Audits récents et plan de remédiation
4. Changements réglementaires
5. Mises à jour technologiques Redis
6. Budget et ressources
7. Formation et sensibilisation

### 2. Politiques et procédures obligatoires

#### Politique générale de sécurité Redis
```
Objectif : Définir les règles d'usage de Redis
Périmètre : Tous les environnements (dev, staging, prod)
Propriétaire : RSSI
Validation : CISO
Révision : Annuelle
```

**Contenu minimal :**
- Principes généraux (confidentialité, intégrité, disponibilité)
- Classification des données autorisées
- Exigences d'authentification et autorisation
- Chiffrement obligatoire
- Logging et audit
- Gestion des changements
- Sauvegarde et restauration
- Gestion des incidents

#### Procédures opérationnelles standard (SOP)

**SOP-01 : Provisioning d'une nouvelle instance Redis**
```
1. Demande formelle via ticket (justification métier)
2. Validation par architecte et RSSI
3. Classification des données à stocker
4. Choix de la configuration selon classification
5. Déploiement via IaC (Terraform validé)
6. Checklist de sécurité post-déploiement
7. Tests de connectivité et performance
8. Handover à l'équipe opérationnelle
9. Documentation dans CMDB
```

**SOP-02 : Gestion des comptes utilisateurs**
```
Création :
1. Demande via formulaire standardisé
2. Validation par manager et propriétaire des données
3. Création du compte avec principe du moindre privilège
4. Notification à l'utilisateur (canal sécurisé)
5. Formation si premier accès
6. Enregistrement dans le registre des accès

Révocation :
1. Départ notifié par RH
2. Désactivation immédiate du compte ACL
3. Revue des accès récents (audit)
4. Documentation de la révocation
5. Mise à jour du registre des accès
```

**SOP-03 : Gestion des changements Redis**
```
1. Demande de changement (RFC) documentée
2. Analyse d'impact (disponibilité, sécurité, conformité)
3. Validation par Change Advisory Board
4. Tests en environnement non-prod
5. Fenêtre de maintenance planifiée
6. Backup pré-changement
7. Exécution du changement
8. Vérification post-changement (smoke tests)
9. Documentation du changement effectif
10. Communication aux parties prenantes
```

**SOP-04 : Réponse aux demandes RGPD (DSR - Data Subject Requests)**
```
Droit d'accès (Article 15) :
1. Réception demande via canal officiel
2. Vérification identité du demandeur
3. Recherche dans Redis via clés indexées (ex: user:<id>)
4. Export des données au format lisible
5. Remise au demandeur (< 1 mois)

Droit à l'effacement (Article 17) :
1. Réception demande + vérification identité
2. Validation des conditions d'effacement (pas d'obligation légale de conservation)
3. Suppression de toutes les clés liées (DEL multi-keys)
4. Vérification propagation sur réplicas
5. Marquage des backups pour purge future
6. Confirmation écrite au demandeur
7. Logging de l'opération (audit trail)
```

---

## 🔍 Audit et contrôle

### 1. Programme d'audit interne

**Fréquence :**
- Audit sécurité : Trimestriel
- Audit de conformité : Semestriel
- Audit forensique : En cas d'incident

**Périmètre de l'audit :**

#### Configuration système
```bash
# Checklist automatisable
□ OS version et patches à jour
□ Services non nécessaires désactivés
□ Firewall actif et correctement configuré
□ SELinux/AppArmor enabled
□ NTP configuré (horodatage fiable pour logs)
□ Logging système activé
□ Rotation des logs configurée
```

#### Configuration Redis
```bash
# Extraire la config complète
redis-cli CONFIG GET '*' > config-audit-$(date +%Y%m%d).txt

# Points de contrôle critiques
□ bind correctement configuré (pas 0.0.0.0 en prod)
□ protected-mode yes
□ requirepass/ACL configuré
□ TLS activé (tls-port et certificats valides)
□ Commandes dangereuses renommées ou désactivées
□ maxmemory et politique d'éviction définies
□ Persistance activée (save ou appendonly)
□ Réplication configurée si HA requise
□ slowlog parameters configurés
```

#### Revue des accès
```bash
# Lister tous les utilisateurs ACL
ACL LIST

# Pour chaque utilisateur :
□ Le compte est-il toujours nécessaire ?
□ Les permissions sont-elles appropriées ?
□ Dernière utilisation < 90 jours ?
□ Rotation du password effectuée ?

# Revue des connexions actives
CLIENT LIST

# Identifier les clients suspects (IP, commandes)
```

#### Analyse des logs
```bash
# Patterns suspects à rechercher
grep "failed.*auth" redis.log | wc -l  # Échecs d'authentification
grep "CONFIG SET" redis.log              # Modifications config
grep "FLUSHDB\|FLUSHALL" redis.log       # Suppressions massives
grep "SCRIPT" redis.log                  # Exécutions de scripts Lua

# Connexions depuis IP inconnues
# (nécessite parsing des logs ou proxy avec audit)
```

### 2. Tests de pénétration

**Tests externes (Black Box) :**
- Scan de ports et services exposés
- Tentatives de brute force sur l'authentification
- Exploitation de vulnérabilités connues (CVE)
- Injection de commandes via l'application

**Tests internes (Gray Box) :**
- Élévation de privilèges (ACL bypass)
- Exfiltration de données
- Déni de service
- Persistence (backdoors)

**Livrables attendus :**
- Rapport technique détaillé
- Classement des vulnérabilités (CVSS score)
- Preuves de concept (PoC)
- Plan de remédiation priorisé
- Retest après correction

---

## 📊 KPIs et métriques de conformité

### Métriques de sécurité

| KPI | Cible | Mesure | Fréquence |
|-----|-------|--------|-----------|
| Pourcentage d'instances Redis avec TLS | 100% | Config audit | Mensuel |
| Pourcentage d'instances avec ACL configurées | 100% | Config audit | Mensuel |
| Délai moyen de patching CVE critiques | < 7 jours | Ticketing | Continu |
| Nombre de tentatives d'authentification échouées | Baseline | Logs | Quotidien |
| Couverture des backups | 100% | Monitoring | Quotidien |
| Succès des tests de restauration | 100% | Tests mensuels | Mensuel |

### Métriques de conformité

| KPI | Cible | Mesure | Fréquence |
|-----|-------|--------|-----------|
| Taux de complétion des checklists de sécurité | 100% | Audit | Trimestriel |
| Nombre de non-conformités identifiées | Décroissant | Audit | Trimestriel |
| Délai de correction des non-conformités | < 30 jours | Suivi d'audit | Continu |
| Taux de participation aux formations sécurité | > 95% | RH/LMS | Annuel |
| Nombre d'incidents de sécurité | 0 | SIEM | Continu |
| Respect des SLA de notification (RGPD 72h) | 100% | Post-incident review | Post-incident |

### Métriques opérationnelles

| KPI | Cible | Mesure | Fréquence |
|-----|-------|--------|-----------|
| Disponibilité du service Redis | > 99.9% | Monitoring | Continu |
| Durée moyenne de recovery (MTTR) | < 15 min | Incident tracking | Post-incident |
| Taux de succès des failovers automatiques | > 99% | Tests | Mensuel |

---

## 🎓 Formation et sensibilisation

### Programme de formation

#### Pour les développeurs
**Module 1 : Fondamentaux de sécurité Redis (2h)**
- Pourquoi sécuriser Redis ?
- Risques et menaces courantes
- Bonnes pratiques de développement
- Gestion des credentials (ne jamais hardcoder)
- Validation des entrées
- Utilisation sécurisée des ACL

**Module 2 : Conformité et protection des données (1h)**
- RGPD : principes et obligations
- Classification des données
- Durées de rétention (TTL)
- Droit à l'effacement
- Exercice pratique : implémenter un effacement RGPD-compliant

#### Pour les DevOps/Admins
**Module 1 : Déploiement sécurisé (3h)**
- Configuration hardening
- TLS setup et troubleshooting
- ACL avancées
- Monitoring et alerting
- Backup et restauration sécurisés

**Module 2 : Gestion des incidents (2h)**
- Scénarios d'incident courants
- Procédure de réponse
- Forensique de base
- War game simulation

#### Pour les compliance officers
**Module 1 : Audit Redis (2h)**
- Checklist de conformité
- Outils d'audit
- Lecture des rapports techniques
- Validation des contrôles

---

## 📚 Ressources et références

### Standards et frameworks
- **NIST Cybersecurity Framework** : https://www.nist.gov/cyberframework
- **CIS Redis Benchmark** : https://www.cisecurity.org/
- **OWASP Top 10** : https://owasp.org/www-project-top-ten/
- **ISO 27001:2022** : https://www.iso.org/standard/27001

### Réglementations
- **RGPD (texte intégral)** : https://eur-lex.europa.eu/eli/reg/2016/679/oj
- **CNIL guides pratiques** : https://www.cnil.fr/fr/guide-de-la-securite-des-donnees-personnelles
- **PCI DSS v4.0** : https://www.pcisecuritystandards.org/
- **HIPAA** : https://www.hhs.gov/hipaa/

### Documentation Redis
- **Redis Security** : https://redis.io/docs/management/security/
- **Redis ACL** : https://redis.io/docs/management/security/acl/
- **Redis TLS** : https://redis.io/docs/management/security/encryption/

### Outils de conformité
- **OpenVAS** (scan de vulnérabilités) : https://www.openvas.org/
- **Lynis** (audit système) : https://cisofy.com/lynis/
- **OWASP ZAP** (pentesting) : https://www.zaproxy.org/

---

## ✅ Synthèse : Les 10 commandements de la conformité Redis

1. **Tu chiffreras** toutes les connexions (TLS) et les données au repos
2. **Tu authentifieras** avec ACL granulaires (jamais de passwordless)
3. **Tu auditeras** tous les accès aux données sensibles
4. **Tu sauvegarderas** chiffré et tu testeras la restauration
5. **Tu classifieras** les données avant de les stocker dans Redis
6. **Tu minimiseras** la surface d'attaque (firewall, commandes désactivées)
7. **Tu monitoreras** en continu et alerteras intelligemment
8. **Tu documenteras** tout (architecture, procédures, incidents)
9. **Tu formeras** régulièrement les équipes
10. **Tu auditeras** périodiquement ta posture de sécurité

---

**Cette introduction au module de gouvernance et conformité pose les fondations nécessaires. Les sections suivantes détailleront l'implémentation pratique de ces principes pour chaque aspect réglementaire.**

⏭️ [RGPD et données personnelles dans Redis](/17-gouvernance-conformite/01-rgpd-donnees-personnelles.md)
