
# 🎯 Formation Redis

![License](https://img.shields.io/badge/License-CC%20BY%204.0-blue.svg)
![Redis Version](https://img.shields.io/badge/Redis-7.x-red.svg)
![Modules](https://img.shields.io/badge/Modules-19%2F19-brightgreen.svg)
![Language](https://img.shields.io/badge/Langue-Français-blue.svg)
![Level](https://img.shields.io/badge/Niveau-Débutant%20à%20Avancé-orange.svg)

**Une formation moderne pour maîtriser Redis, du cache simple aux architectures distribuées.**

<div align="center">
  <img src="https://redis.io/wp-content/uploads/2024/04/Logotype.svg?auto=webp&quality=85,75&width=120" alt="Redis Logo" width="200"/>
</div>

---

## 📖 Navigation

- [À propos](#-%C3%A0-propos)
- [Contenu](#-contenu-de-la-formation)
- [Pour qui ?](#-pour-qui-)
- [Démarrage rapide](#-d%C3%A9marrage-rapide)
- [Structure](#-structure-du-projet)
- [Parcours suggéré](#-parcours-sugg%C3%A9r%C3%A9s)
- [Licence](#-licence)
- [Contact](#%E2%80%8D-%C3%A0-propos-de-lauteur)

---

## 📋 À propos

Cette formation est née du constat qu'il manquait une ressource francophone complète et à jour sur Redis, couvrant à la fois Redis Core et Redis Stack, tout en intégrant les évolutions récentes de l'écosystème (changement de licence 2024, fork Valkey, Vector Search pour l'IA...).

**✨ Ce que vous trouverez ici :**
- 📚 **19 modules progressifs** structurés pour un apprentissage fluide
- 🎯 **Redis Core + Redis Stack** (JSON, Search, TimeSeries, Bloom)
- 🏗️ **Architecture complète** : Sentinel, Cluster, Cloud, Kubernetes
- 🔒 **Production-ready** : sécurité, monitoring, troubleshooting
- 💡 **8 études de cas réels** (e-commerce, IoT, IA/RAG, gaming...)
- 🇫🇷 **100% en français** et gratuit (CC BY 4.0)
- 🆕 **À jour 2025** : Redis 7.x, Valkey, Vector Search, CRDT

**Durée estimée :** 25-35 heures • **Format :** Théorique sans exercices (code réutilisable inclus)

> **Note :** Cette formation n'inclut pas d'exercices pratiques traditionnels, mais propose du code complet et des études de cas détaillées pour une application immédiate.

---

## 📚 Contenu de la formation

### 🌟 Fondamentaux (Modules 1-5)
1. **L'écosystème Redis Moderne** - Redis vs Valkey, Core vs Stack, architecture single-thread
2. **Structures de données natives** - Strings, Lists, Sets, Sorted Sets, HyperLogLog, Bitmaps
3. **Structures étendues (Redis Stack)** - JSON, Search, TimeSeries, Bloom, Vector Search
4. **Cycle de vie des données** - TTL, éviction, LRU/LFU, SCAN, key patterns
5. **Persistance et fiabilité** - RDB, AOF, stratégies hybrides, backup

### 💻 Développement (Modules 6-9)
6. **Patterns de développement** - Caching, Redlock, Rate Limiting, Cache Avalanche
7. **Atomicité et programmabilité** - Transactions, Lua, Redis Functions
8. **Communication et flux** - Pub/Sub, Streams, Consumer Groups
9. **Intégration langages** - Python, Node.js, Go, Java, tests

### 🚀 DevOps & Production (Modules 10-15)
10. **Haute Disponibilité** - Réplication Master-Replica, Sentinel, failover
11. **Architecture Distribuée** - Redis Cluster, sharding, hash slots, cross-DC
12. **Production et Sécurité** - ACLs, TLS, tuning Linux, rolling upgrades
13. **Monitoring** - Prometheus, Grafana, métriques clés, alerting
14. **Performance et Troubleshooting** - Slowlog, memory analysis, OOM, latence
15. **Cloud et Conteneurs** - AWS/Azure/GCP, Kubernetes, Docker, coûts

### 🎓 Expertise (Modules 16-19)
16. **Études de cas réels** - 8 cas (e-commerce, IoT, IA/RAG, gaming...)
17. **Gouvernance et Conformité** - RGPD, encryption, audit, certifications
18. **Évolutions et Futur** - Redis 7.x, Valkey, IA, roadmap
19. **Ressources et Certification** - Documentation, Redis University, communautés

> 📄 **Table des matières détaillée** disponible dans [SOMMAIRE.md](/SOMMAIRE.md)

---

## 👥 Pour qui ?

### 🌱 Développeurs débutants
Vous découvrez Redis ? Commencez par les modules 1-5 pour maîtriser les fondamentaux.

### 🌿 Développeurs intermédiaires
Vous utilisez Redis en cache ? Explorez les modules 6-9 pour les patterns avancés et Redis Stack.

### 🌳 DevOps & Architectes
Vous déployez en production ? Plongez dans les modules 10-15 pour la HA, le cluster et la sécurité.

### 🎯 Tech Leads & Décideurs
Modules 1, 3, 16-18 pour comprendre l'écosystème et les cas d'usage stratégiques.

---

## 🚀 Démarrage rapide

### Prérequis

```bash
# Vérifier Docker (recommandé pour débuter)
docker --version

# OU installer Redis Stack localement
# https://redis.io/docs/install/install-stack/
```

### Installation Redis Stack (Docker)

```bash
# Lancer Redis Stack (inclut RedisInsight GUI)
docker run -d \
  --name redis-stack \
  -p 6379:6379 \
  -p 8001:8001 \
  redis/redis-stack:latest

# Tester la connexion
docker exec -it redis-stack redis-cli ping
# Résultat attendu : PONG
```

### Accéder à RedisInsight (GUI)

Ouvrez votre navigateur : **http://localhost:8001**

### Cloner cette formation

```bash
git clone https://github.com/NDXDeveloper/formation-redis-complete.git
cd formation-redis-complete
```

---

## 📁 Structure du projet

```
formation-redis-complete/
├── README.md                    # Ce fichier
├── SOMMAIRE.md                  # Table des matières détaillée
├── LICENSE                      # CC BY 4.0
│
├── modules/
│   ├── module-01-ecosysteme/
│   │   ├── 1.1-qu-est-ce-que-redis.md
│   │   ├── 1.2-core-vs-stack.md
│   │   ├── ...
│   │   └── 1.6-installation.md
│   │
│   ├── module-02-structures-natives/
│   ├── module-03-redis-stack/
│   ├── ...
│   └── module-19-ressources/
│
└── assets/
    ├── schemas/                 # Diagrammes et schémas
    └── code-examples/           # Exemples de code réutilisables
```

---

## 🎯 Parcours suggérés

### 📘 Parcours Développeur (15-20h)

| Étape | Modules | Focus | Durée |
|-------|---------|-------|-------|
| 1️⃣ | 1-2 | Découverte et structures natives | 3-4h |
| 2️⃣ | 3-4 | Redis Stack et gestion données | 3-4h |
| 3️⃣ | 6-7 | Patterns et programmation | 4-5h |
| 4️⃣ | 9, 16 | Intégration et cas d'usage | 3-4h |

### 🛠️ Parcours DevOps (20-25h)

| Étape | Modules | Focus | Durée |
|-------|---------|-------|-------|
| 1️⃣ | 1, 4-5 | Bases et persistance | 3-4h |
| 2️⃣ | 10-11 | HA et clustering | 5-6h |
| 3️⃣ | 12-13 | Sécurité et monitoring | 4-5h |
| 4️⃣ | 14-15 | Performance et cloud | 5-6h |
| 5️⃣ | 17 | Gouvernance | 2-3h |

### 🚀 Parcours Complet (30-35h)

Suivez les 19 modules dans l'ordre pour une maîtrise complète.

---

## 💡 Comment utiliser cette formation

### Mode apprentissage linéaire
👉 Suivez l'ordre des modules, 1 section à la fois (30-45 min/section)

### Mode référence
👉 Utilisez le [SOMMAIRE.md](/SOMMAIRE.md) pour accéder directement à un sujet précis

### Mode projet
👉 Allez au [Module 16](/16-etudes-cas-patterns-reels/README.md) et implémentez un cas d'usage réel

### Mode préparation certification
👉 Consultez le [Module 19](/19-ressources-certification/README.md) pour Redis University

**💡 Conseil :** Lancez Redis Stack en Docker et testez chaque commande pendant votre lecture !

---

## ❓ Questions fréquentes

**Q : Pourquoi pas d'exercices pratiques ?**
R : Le choix a été fait de fournir du contenu théorique dense avec du code réutilisable et des études de cas complètes, permettant à chacun d'adapter à son contexte.

**Q : Redis ou Valkey ?**
R : Les deux sont couverts dès le Module 1. La formation s'applique aux deux (protocole compatible).

**Q : Quelle version de Redis ?**
R : Formation basée sur Redis 7.x et Redis Stack récent (2024-2025).

**Q : Combien de temps pour finir ?**
R : Comptez 25-35h sur 4-8 semaines (1-2h par jour, 3-5 jours/semaine).

**Q : Puis-je utiliser pour enseigner ?**
R : Oui ! Licence CC BY 4.0 (voir section Licence pour l'attribution).

**Q : C'est à jour avec le changement de licence Redis 2024 ?**
R : Oui, le Module 1 couvre ce sujet en détail ainsi que le fork Valkey.

---

## 📝 Licence

Cette formation est publiée sous licence **Creative Commons Attribution 4.0 International (CC BY 4.0)**.

✅ **Vous êtes libre de :**
- Partager : copier, distribuer et communiquer le matériel
- Adapter : remixer, transformer et créer à partir du matériel
- Usage commercial autorisé

**📋 À condition de :**
- Attribution : Créditer l'œuvre, fournir un lien vers la licence et indiquer si des modifications ont été apportées

**Attribution suggérée :**
```
Formation Redis par Nicolas DEOUX
https://github.com/NDXDeveloper/formation-redis-complete
Licence CC BY 4.0
```

Voir le fichier [LICENSE](/LICENSE) pour le texte complet.

---

## 👨‍💻 À propos de l'auteur

**Nicolas DEOUX**
Développeur passionné par les technologies de bases de données et l'architecture distribuée.

Cette formation est le fruit d'une veille technologique continue et d'une volonté de partager les connaissances avec la communauté francophone.

**📬 Contact :**
- 📧 Email : [NDXDev@gmail.com](mailto:NDXDev@gmail.com)
- 💼 LinkedIn : [nicolas-deoux](https://www.linkedin.com/in/nicolas-deoux-ab295980/)
- 🐙 GitHub : [@NDXDeveloper](https://github.com/NDXDeveloper)

---

## 🙏 Remerciements

Un grand merci à :
- La communauté Redis et les mainteneurs de Redis Stack
- La Linux Foundation et le projet Valkey
- Les contributeurs de la documentation Redis
- Tous les développeurs qui partagent leurs connaissances

**Ressources qui ont inspiré cette formation :**
[Redis Documentation](https://redis.io/docs/) • [Redis University](https://university.redis.com/) • [Valkey](https://valkey.io/) • [Antirez Blog](http://antirez.com/)

---

## 🌟 Soutenir le projet

Si cette formation vous a été utile :
- ⭐ Donnez une étoile au repo
- 🔗 Partagez avec votre équipe ou sur les réseaux
- 💬 Partagez votre retour d'expérience

Chaque geste compte et encourage la création de contenu éducatif francophone ! 🙌

---

<div align="center">

**🎯 Bon apprentissage avec Redis ! 🚀**

[![Stars](https://img.shields.io/github/stars/NDXDeveloper/formation-redis-complete?style=social)](https://github.com/NDXDeveloper/formation-redis-complete)
[![Follow](https://img.shields.io/github/followers/NDXDeveloper?style=social)](https://github.com/NDXDeveloper)

**[⬆ Retour en haut](#-formation-redis)**

*Dernière mise à jour : Décembre 2025*
*Version : 1.0.0*

</div>
