🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.3 - Le séisme de 2024 : Changement de licence et le fork Valkey

## 📋 Introduction

En mars 2024, l'écosystème Redis a connu un bouleversement majeur qui a fait la une des actualités tech. Redis Ltd (anciennement Redis Labs) a annoncé un changement de licence qui a transformé Redis d'un projet **entièrement open source** en un produit sous **licence propriétaire**.

Cette décision a déclenché une réaction en chaîne : la **Linux Foundation** a créé **Valkey**, un fork open source de Redis, soutenu par des géants comme AWS, Google Cloud et Oracle.

Si vous débutez avec Redis, vous vous demandez probablement : "Qu'est-ce que cela signifie pour moi ?" Cette section va tout clarifier.

---

## 🎯 En résumé rapide

> **Mars 2024** : Redis change de licence et n'est plus open source
> **→ Valkey** est créé comme alternative 100% open source
> **→ Les deux sont compatibles** et évoluent en parallèle
> **→ Vous pouvez choisir** selon vos besoins et valeurs

---

## 1️⃣ Comprendre les licences logicielles

Avant de plonger dans l'histoire, comprenons ce qu'est une licence logicielle.

### Qu'est-ce qu'une licence logicielle ?

Une **licence** est un contrat juridique qui définit :
- Ce que vous **pouvez faire** avec le logiciel
- Ce que vous **ne pouvez pas faire**
- Vos **obligations** si vous l'utilisez

**Analogie du terrain** :

Imaginez que le logiciel est un terrain :

| Type de licence | Analogie | Exemple |
|-----------------|----------|---------|
| **Open Source** | Parc public : tout le monde peut entrer, jouer, organiser des événements, même commerciaux | Linux, PostgreSQL |
| **Propriétaire** | Terrain privé : vous devez payer pour entrer et respecter les règles du propriétaire | Windows, Oracle DB |
| **Source Available** | Parc semi-privé : vous pouvez entrer gratuitement, mais pas organiser d'événements commerciaux | Redis (2024+) |

### Les licences open source

**Caractéristiques de l'open source véritable** :

- ✅ **Utilisation libre** : Pour n'importe quel usage
- ✅ **Code source accessible** : Vous pouvez lire le code
- ✅ **Modification libre** : Vous pouvez le changer
- ✅ **Distribution libre** : Vous pouvez le partager
- ✅ **Usage commercial** : Vous pouvez en faire un business

**Les licences open source classiques** :

- **MIT** : Ultra-permissive, faites ce que vous voulez
- **Apache 2.0** : Permissive avec protection des brevets
- **BSD** : Simple et permissive (ancienne licence de Redis)
- **GPL** : Copyleft, les modifications doivent rester open source

### Redis avant 2024 : Licence BSD

Jusqu'en mars 2024, Redis utilisait la **licence BSD 3-Clause**, l'une des plus permissives :

```
En gros, la licence BSD disait :
"Prenez Redis, faites-en ce que vous voulez, même un produit commercial,
tant que vous gardez notre copyright dans le code."
```

**Conséquences pratiques** :
- Amazon pouvait créer ElastiCache (Redis managé)
- Google pouvait créer Memorystore
- Toute entreprise pouvait vendre du Redis sans reverser d'argent à Redis Ltd

---

## 2️⃣ L'histoire de Redis : De l'open source au propriétaire

### 2009-2023 : L'ère open source (15 ans)

**2009** : Salvatore Sanfilippo (antirez) crée Redis
- Licence BSD
- Projet communautaire
- Adopté massivement

**2011** : Création de Redis Labs
- Offre du support commercial
- Développe Redis Enterprise (version étendue payante)
- Redis Core reste BSD

**2015-2020** : Explosion de l'adoption
- Redis devient #1 des bases NoSQL
- Utilisé par des millions d'applications
- Intégré dans tous les clouds

**Problème qui émerge** : Les cloud providers (AWS, Google, Azure) proposent Redis comme service managé et génèrent des centaines de millions de dollars **sans contribuer financièrement** au projet.

### Le modèle économique sous tension

**Analogie du restaurant** :

Imaginez que vous créez une recette de pizza incroyable et la partagez librement :

```
Vous (Redis Labs) :
└─ Créez la recette (Redis)
└─ La partagez gratuitement (open source BSD)
└─ Espérez vendre du consulting et du support

Les géants (AWS, Google, Azure) :
└─ Prennent votre recette
└─ Ouvrent des pizzerias géantes
└─ Font des millions sans vous reverser un centime
└─ Les clients préfèrent leurs pizzerias (pratique, intégrée)

Résultat :
└─ Vous avez du mal à monétiser votre innovation
└─ Les géants profitent de votre travail
```

C'est exactement ce qui s'est passé avec Redis.

### Mars 2024 : Le changement historique

**20 mars 2024**, Redis Ltd annonce :

> À partir de Redis 7.4, nous passons à une **double licence propriétaire** :
> - **RSALv2** (Redis Source Available License)
> - **SSPLv1** (Server Side Public License)

**Traduction** : Redis n'est plus open source selon la définition officielle.

### Que signifient ces nouvelles licences ?

#### RSALv2 et SSPLv1 en termes simples

**Ce que vous pouvez toujours faire** :
- ✅ Utiliser Redis dans vos applications
- ✅ L'utiliser en production
- ✅ Le modifier pour vos besoins internes
- ✅ L'utiliser gratuitement

**Ce que vous ne pouvez plus faire** :
- ❌ Vendre Redis comme un service managé (type AWS ElastiCache)
- ❌ Créer un produit concurrent à Redis Enterprise
- ❌ L'inclure dans une distribution commerciale sans autorisation

**Analogie de la recette** :

```
Nouvelle licence =
"Vous pouvez utiliser ma recette de pizza gratuitement pour :
 ✅ Votre restaurant
 ✅ Vos amis
 ✅ Des événements privés

Mais vous ne pouvez PAS :
 ❌ Ouvrir une pizzeria qui ne vend que mes pizzas
 ❌ Vendre ma recette à d'autres
 ❌ Créer une chaîne de restaurants basée sur ma recette"
```

### Pourquoi ce changement ?

**Position de Redis Ltd** :

> "Les cloud providers génèrent des centaines de millions de dollars avec Redis sans contribuer au développement. Ce n'est pas durable. Nous devons protéger notre capacité à innover."

**Arguments principaux** :
1. AWS, Google, Azure gagnent des fortunes avec Redis
2. Redis Ltd peine à monétiser face à ces géants
3. Besoin de financer le développement continu
4. Protéger l'innovation (Redis Stack, nouveaux modules)

**Chiffres clés** :
- ElastiCache (AWS) : estimé à 1 milliard $/an de revenus
- Redis Ltd : ~100-150 millions $/an
- **Ratio 10:1** en faveur d'AWS

### Réactions de la communauté

La réaction a été **immédiate et divisée** :

#### 😡 Camp "Trahison"
- "Redis abandonne ses racines open source"
- "15 ans de contributions communautaires trahies"
- "On ne peut plus faire confiance"

#### 🤔 Camp "Je comprends"
- "C'est légitime face aux pratiques des clouds"
- "Il faut bien financer le développement"
- "Les utilisateurs individuels ne sont pas impactés"

#### 😐 Camp "Pragmatique"
- "C'est l'évolution naturelle des projets open source"
- "MongoDB, Elasticsearch ont fait pareil"
- "L'important est la continuité technique"

---

## 3️⃣ Valkey : La réponse de la communauté

### La naissance de Valkey

**2 jours après l'annonce de Redis Ltd**, la **Linux Foundation** annonce :

> Nous forkon Redis 7.2.4 (dernière version BSD) pour créer **Valkey**, un projet 100% open source.

### Qu'est-ce qu'un fork ?

**Analogie de la route** :

```
       Redis (jusqu'en 2024)
              │
              │ BSD 3-Clause
              │
      2009 ───┴──── 2024
              │
              │ Mars 2024 : Bifurcation
              │
         ┌────┴────┐
         │         │
    Redis Ltd   Valkey
         │         │
     Propriétaire  Open Source
     RSALv2/SSPLv1  BSD 3-Clause
         │         │
         ↓         ↓
    Redis 7.4+   Valkey 7.2+
```

**Fork = Copie légale** d'un projet qui évolue ensuite indépendamment.

### Qui soutient Valkey ?

**Sponsors fondateurs** :
- 🔵 **AWS** (Amazon)
- 🔴 **Google Cloud**
- 🟠 **Oracle**
- 🟣 **Ericsson**
- 🟢 **Snap Inc.**

**Linux Foundation** : Organisation à but non lucratif qui héberge :
- Linux (le système d'exploitation)
- Kubernetes
- Node.js
- Et maintenant Valkey

**Pourquoi ces entreprises soutiennent Valkey ?**

1. **Protéger leurs investissements** : ElastiCache, Memorystore sont basés sur Redis
2. **Éviter la dépendance** à Redis Ltd
3. **Garantir l'open source** pour leurs clients
4. **Influence sur la roadmap** via la gouvernance ouverte

### Caractéristiques de Valkey

| Aspect | Détail |
|--------|--------|
| **Licence** | BSD 3-Clause (open source véritable) |
| **Compatibilité** | 100% compatible avec Redis 7.2 |
| **Gouvernance** | Communautaire (Linux Foundation) |
| **Développement** | Contributions de multiples entreprises |
| **Objectif** | Rester open source à jamais |
| **Nom** | "Valk" = Valkyrie (mythologie nordique) |

### Valkey vs Redis : La divergence

**Au lancement (avril 2024)** :
- Valkey = Redis 7.2.4 exact (même code)
- 100% compatible

**Évolution (2024-2025)** :
- Les deux projets ajoutent des fonctionnalités indépendamment
- Compatibilité maintenue mais pas garantie à l'infini
- Valkey suit sa propre roadmap

```
Compatibilité dans le temps :

2024 ════════════════ 100% compatible
2025 ══════════════── ~99% compatible
2026 ════════════════ ~95% compatible (?)
2027+ ═══════════════ Divergence croissante (?)
```

---

## 4️⃣ Implications pratiques pour vous

### Si vous êtes développeur

#### Nouveau projet en 2024-2025 ?

**Deux options valides** :

**Option 1 : Choisir Redis**
```
✅ Accès à Redis Stack (modules avancés)
✅ Documentation officielle complète
✅ Support commercial disponible
✅ Écosystème mature
⚠️  Licence propriétaire
⚠️  Dépendance à Redis Ltd
```

**Option 2 : Choisir Valkey**
```
✅ 100% open source
✅ Soutenu par des géants tech
✅ Indépendance garantie
✅ Gouvernance communautaire
⚠️  Plus récent (moins de retours terrain)
⚠️  Pas d'équivalent Redis Stack (encore)
```

**Critères de décision** :

| Critère | Redis | Valkey |
|---------|-------|--------|
| Besoin de Redis Stack | ✅ Oui | ❌ Non (pas encore) |
| Priorité open source | ❌ Non | ✅ Oui |
| Support commercial souhaité | ✅ Oui | ⚠️  Tiers uniquement |
| Projet critique | ✅ Mature | ⚠️  Nouveau |
| Cloud provider (AWS/GCP) | ⚠️  Attention | ✅ Supporté |

#### Projet existant avec Redis ?

**Pas de panique !**

- ✅ Votre Redis actuel continue de fonctionner
- ✅ Pas d'obligation de migrer
- ✅ Vous pouvez rester en Redis 7.2 pendant des années
- ✅ Migration vers Valkey possible si besoin

**Scénarios possibles** :

```
Scénario 1 : Rester sur Redis
└─ Si satisfait de Redis Ltd
└─ Si utilise Redis Stack
└─ Si pas de problème avec la licence

Scénario 2 : Migrer vers Valkey
└─ Si préférence open source forte
└─ Si hébergé sur AWS/GCP (transition naturelle)
└─ Si veut éviter la dépendance propriétaire

Scénario 3 : Attendre et voir
└─ Observer l'évolution des deux projets
└─ Décider en 2025-2026
└─ Pas d'urgence
```

### Si vous êtes en entreprise

#### Considérations légales

**Questions à poser à votre service juridique** :

1. Notre usage actuel est-il compatible avec la nouvelle licence Redis ?
2. Prévoyons-nous de vendre un service basé sur Redis ?
3. Avons-nous besoin de garanties open source pour nos clients ?
4. Quel est notre niveau de dépendance à Redis ?

**Dans 99% des cas** : Vous n'êtes **pas impacté** par le changement de licence car vous utilisez juste Redis comme brique technique interne.

#### Stratégies d'entreprise

**Stratégie 1 : Redis Enterprise**
- Souscrivez au support Redis Ltd
- Accès à toutes les fonctionnalités
- Support garanti
- Coût : 💰💰💰

**Stratégie 2 : Valkey open source**
- Déployez Valkey
- Utilisez les services managés (AWS MemoryDB pour Valkey)
- Support communautaire ou tiers
- Coût : 💰 (infrastructure seulement)

**Stratégie 3 : Hybride**
- Redis pour le développement et tests
- Valkey pour la production
- Transition progressive

---

## 5️⃣ L'écosystème en 2024-2025

### La fracture de l'écosystème

```
                 AVANT 2024
        ┌────────────────────────┐
        │    Écosystème Redis    │
        │      (Unifié)          │
        └────────────────────────┘
                     │
                     ↓
                 APRÈS 2024
        ┌────────────┬───────────┐
        │            │           │
    ┌───▼────┐   ┌───▼────┐  ┌───▼─────┐
    │ Redis  │   │ Valkey │  │ KeyDB   │
    │  Ltd   │   │ (LF)   │  │ & Autre │
    └────────┘   └────────┘  └─────────┘
```

### Support des cloud providers

#### AWS
- **ElastiCache** : Passe à Valkey progressivement
- **MemoryDB** : Annonce support Valkey
- Position : **Pro-Valkey**

#### Google Cloud
- **Memorystore** : Évalue Valkey
- Position : **Neutre, mais penche Valkey**

#### Azure
- **Azure Cache for Redis** : Reste sur Redis Ltd (partenariat)
- Position : **Pro-Redis Ltd**

### Bibliothèques et clients

**Bonne nouvelle** : Les clients Redis (Python, Node.js, Java, etc.) fonctionnent avec **les deux** !

```python
# Même code pour Redis et Valkey
import redis  # ou 'valkey' dans le futur

client = redis.Redis(host='localhost', port=6379)
client.set('key', 'value')
print(client.get('key'))
```

**Compatibilité** : Tant que les commandes sont identiques, pas de changement de code.

---

## 6️⃣ Comparaison Redis vs Valkey (fin 2024)

### Tableau comparatif

| Critère | Redis 7.4+ | Valkey 7.2+ |
|---------|-----------|-------------|
| **Licence** | RSALv2/SSPLv1 | BSD 3-Clause |
| **Open Source** | ❌ Source Available | ✅ Vrai open source |
| **Gouvernance** | Redis Ltd | Linux Foundation |
| **Redis Stack** | ✅ Inclus | ❌ Pas encore |
| **Maturité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ (nouveau) |
| **Support commercial** | ✅ Redis Ltd | ⚠️  Tiers |
| **Cloud AWS** | ⚠️  Limité | ✅ Natif |
| **Cloud GCP** | ⚠️  Limité | ✅ Prévu |
| **Cloud Azure** | ✅ Partenaire | ⚠️  À venir |
| **Communauté** | Établie | Croissante |
| **Innovation** | Redis Ltd | Multi-entreprises |
| **Compatibilité Redis Core** | ✅ 100% | ✅ 100% |

### Forces et faiblesses

#### Redis (Redis Ltd)

**Forces** ✅
- Écosystème mature et complet
- Redis Stack (JSON, Search, TimeSeries)
- Documentation exhaustive
- Support commercial premium
- Innovation continue (15 ans d'historique)

**Faiblesses** ⚠️
- Licence propriétaire (limite certains usages)
- Dépendance à une seule entreprise
- Relation tendue avec cloud providers
- Coût potentiel du support

#### Valkey (Linux Foundation)

**Forces** ✅
- 100% open source (BSD)
- Gouvernance communautaire
- Soutenu par des géants (AWS, Google, Oracle)
- Indépendance garantie
- Gratuité assurée

**Faiblesses** ⚠️
- Jeune projet (quelques mois seulement)
- Pas d'équivalent Redis Stack (en développement)
- Documentation en construction
- Moins de retours terrain
- Incertitude sur l'évolution long terme

---

## 7️⃣ L'avenir : Scénarios possibles

### Scénario 1 : Coexistence durable (le plus probable)

```
Redis Ltd                          Valkey (Linux Foundation)
    │                                     │
    │ Se concentre sur l'entreprise       │ Se concentre sur l'open source
    │ Redis Stack avancé                  │ Core solide et performant
    │ Support premium                     │ Communauté active
    │                                     │
    └──────────── Deux écosystèmes ───────┘
                  complémentaires
```

**Résultat** :
- Redis pour les entreprises avec besoins avancés
- Valkey pour l'open source et les clouds
- Compatibilité maintenue au niveau Core

### Scénario 2 : Domination de Valkey

```
    Redis Ltd
        │
        │ Perd des parts de marché
        │ Face à Valkey + Cloud providers
        ↓
    Niche enterprise

    Valkey
        │
        │ Devient le standard de facto
        │ Développe son propre "Stack"
        ↓
    Nouveau leader
```

**Probabilité** : Moyenne (dépend de l'exécution de Valkey)

### Scénario 3 : Réconciliation

```
    Redis Ltd
        │
        │ Revient à l'open source
        │ (pression communautaire/économique)
        ↓
    Fusion ou convergence avec Valkey
```

**Probabilité** : Faible (Redis Ltd semble déterminé)

---

## 8️⃣ Recommandations pratiques

### Pour les débutants qui apprennent

**Conseil** : Apprenez **Redis Core d'abord**, la base est identique dans Redis et Valkey.

```
Votre parcours d'apprentissage :
1. Maîtrisez Redis Core (Strings, Lists, Hashes, etc.)
   └─ Ces concepts sont universels

2. Comprenez les patterns (Cache, Sessions, Queues)
   └─ Indépendants de la licence

3. Plus tard, choisissez votre camp :
   ├─ Redis Stack pour fonctionnalités avancées
   └─ Valkey pour open source pur
```

**Peu importe votre choix final**, les fondamentaux restent les mêmes.

### Pour les projets nouveaux

**Matrice de décision** :

```
Vous avez besoin de :                  → Choisissez
├─ Redis Stack (JSON, Search, TS)     → Redis
├─ Open source strict                  → Valkey
├─ Support commercial premium          → Redis
├─ Déploiement sur AWS                 → Valkey (natif)
├─ Déploiement sur Azure               → Redis (partenaire)
└─ Simple cache, Core uniquement       → Les deux ! (au choix)
```

### Pour les migrations

**Redis → Valkey est simple** (même protocole) :

```bash
# 1. Sauvegarde Redis
redis-cli --rdb dump.rdb

# 2. Installation Valkey
# (même procédure que Redis)

# 3. Restauration
valkey-server --dir . --dbfilename dump.rdb

# 4. Changement de host dans l'application
# redis://localhost:6379 → valkey://localhost:6379
```

**Valkey → Redis** : Identique dans l'autre sens !

---

## 9️⃣ Points clés à retenir

### ✅ L'essentiel en 10 points

1. **Mars 2024** : Redis change de licence (BSD → RSALv2/SSPLv1)
2. **Conséquence** : Redis n'est plus open source au sens strict
3. **Réaction** : Création de Valkey (fork open source BSD)
4. **Valkey** : Soutenu par AWS, Google, Oracle, Linux Foundation
5. **Compatibilité** : Les deux sont compatibles au niveau Core
6. **Pour vous** : Dans 99% des cas, pas d'impact immédiat
7. **Choix** : Redis (avancé + support) vs Valkey (open source)
8. **Migration** : Possible dans les deux sens (protocole identique)
9. **Apprendre** : Les fondamentaux Redis/Valkey sont identiques
10. **Futur** : Les deux projets coexisteront probablement

### 🎯 Votre décision dépend de

| Priorité | Redis | Valkey |
|----------|-------|--------|
| Fonctionnalités avancées | ⭐⭐⭐ | ⭐ |
| Open source pur | ⭐ | ⭐⭐⭐ |
| Maturité | ⭐⭐⭐ | ⭐⭐ |
| Indépendance | ⭐ | ⭐⭐⭐ |
| Support commercial | ⭐⭐⭐ | ⭐ |
| Cloud AWS/GCP | ⭐ | ⭐⭐⭐ |

---

## 🔟 Questions fréquentes

### Q1 : Dois-je changer immédiatement ?
**R :** Non ! Redis 7.2 restera supporté pendant des années. Pas d'urgence.

### Q2 : Mon code va-t-il casser ?
**R :** Non. Les commandes Redis Core sont identiques dans Valkey.

### Q3 : Puis-je utiliser Redis gratuitement ?
**R :** Oui ! La licence interdit seulement de revendre Redis comme service. L'utiliser dans votre appli est gratuit.

### Q4 : Valkey est-il vraiment compatible ?
**R :** Oui, à 100% au niveau Core. C'est un fork direct de Redis 7.2.4.

### Q5 : Redis Stack existe-t-il pour Valkey ?
**R :** Pas encore officiellement (fin 2024). Des alternatives sont en développement.

### Q6 : Qui va gagner entre Redis et Valkey ?
**R :** Probablement les deux ! Coexistence sur différents segments (entreprise vs open source).

### Q7 : Faut-il apprendre Redis ET Valkey ?
**R :** Non, apprenez Redis Core. C'est 95% identique. Les spécificités viendront après.

### Q8 : AWS va-t-il abandonner Redis ?
**R :** ElastiCache migre progressivement vers Valkey, mais supporte encore Redis.

### Q9 : C'est légal de forker Redis comme ça ?
**R :** Oui ! La licence BSD de Redis 7.2 autorisait explicitement le fork.

### Q10 : Redis Ltd va-t-il survivre ?
**R :** Probablement. Ils ont des clients entreprises payants et Redis Stack.

---

## 📚 Chronologie récapitulative

```
2009 ──────── Création de Redis (Salvatore Sanfilippo)
               Licence BSD

2011 ──────── Création de Redis Labs

2015-2023 ─── Explosion de Redis
               Adoption massive
               Cloud providers font des millions

Mars 2024 ─── Redis Ltd change la licence
               → RSALv2 et SSPLv1
               → Fin de l'open source véritable

Avril 2024 ─── Linux Foundation crée Valkey
               → Fork de Redis 7.2.4 BSD
               → Soutenu par AWS, Google, Oracle

2024-2025 ─── Période de transition
               → Les deux projets évoluent en parallèle
               → Écosystème se divise progressivement

2026+ ───────  Futur incertain
               → Coexistence ou domination ?
               → À suivre...
```

---

## 🚀 Prochaine étape

Maintenant que vous comprenez le contexte historique et politique de l'écosystème Redis, nous allons faire une **comparaison technique objective** entre toutes les alternatives disponibles : Redis, Valkey, KeyDB et Memcached.

Cette comparaison vous aidera à choisir la meilleure option pour vos projets spécifiques.

**Prochaine section** : [1.4 - Comparaison Redis vs Valkey vs KeyDB vs Memcached](./04-comparaison-redis-valkey-keydb-memcached.md)

---

## 📖 Ressources complémentaires

### Articles et annonces officielles
- [Redis License Change Announcement (Mars 2024)](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)
- [Valkey Announcement - Linux Foundation](https://www.linuxfoundation.org/press/linux-foundation-launches-open-source-valkey-community)
- [AWS Blog - Why we support Valkey](https://aws.amazon.com/blogs/opensource/valkey-new-open-source-alternative/)

### Discussions communautaires
- Hacker News : Discussions enflammées mais instructives
- Reddit r/redis : Avis de la communauté
- GitHub Issues : Débats techniques

### Comparaisons techniques
- [Valkey vs Redis benchmarks](https://valkey.io/benchmarks/)
- [Cloud provider positions](https://techcrunch.com/2024/03/20/redis-license-change/)

---


⏭️ [Comparaison Redis vs Valkey vs KeyDB vs Memcached](/01-ecosysteme-redis-moderne/04-comparaison-redis-valkey-keydb-memcached.md)
