🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.2 Redis University et parcours d'apprentissage

## Introduction

**Redis University** est la plateforme officielle de formation gratuite proposée par Redis Ltd. Elle offre des cours structurés, des labs pratiques et des certifications pour tous les niveaux, du débutant à l'expert.

**URL principale** : https://university.redis.com/

## 🎓 Vue d'ensemble de Redis University

### Caractéristiques principales

| Aspect | Détails |
|--------|---------|
| **Coût** | Gratuit (cours) / Payant (certifications) |
| **Langues** | Anglais principalement |
| **Format** | Vidéos + Labs interactifs + Quiz |
| **Durée** | 2h à 40h selon les cours |
| **Certificat** | Badge numérique à la fin de chaque cours |
| **Plateforme** | LMS en ligne accessible 24/7 |
| **Prérequis** | Compte gratuit requis |

### Avantages clés

- ✅ **Totalement gratuit** pour les cours (seules les certifications sont payantes)
- ✅ **Labs interactifs** avec environnements Redis préconfigurés
- ✅ **Progression à votre rythme** - cours disponibles en permanence
- ✅ **Badges vérifiables** à afficher sur LinkedIn
- ✅ **Contenu officiel** maintenu par l'équipe Redis
- ✅ **Exemples de code** dans plusieurs langages

## 📚 Catalogue des cours

### 🟢 Niveau Débutant

#### RU101: Introduction to Redis Data Structures
**URL** : https://university.redis.com/courses/ru101/

**Durée** : ~2 heures
**Contenu** :
- Qu'est-ce que Redis ?
- Installation et premiers pas
- Strings, Lists, Sets, Hashes
- Sorted Sets et cas d'usage
- Commandes de base

**Pour qui** : Débutants complets, développeurs découvrant Redis

**Ce que vous apprendrez** :
- Comprendre le modèle clé-valeur
- Manipuler les structures de données natives
- Choisir la bonne structure selon le cas d'usage
- Utiliser redis-cli et Redis Insight

---

#### RU102JS: Redis for JavaScript Developers
**URL** : https://university.redis.com/courses/ru102js/

**Durée** : ~4 heures
**Contenu** :
- Client Node.js (node-redis)
- Patterns de caching
- Session management
- Rate limiting
- Leaderboards

**Pour qui** : Développeurs JavaScript/Node.js

**Ce que vous apprendrez** :
- Intégrer Redis dans une application Node.js
- Implémenter des patterns courants
- Gérer les connexions et le pool
- Optimiser les performances

---

#### RU102PY: Redis for Python Developers
**URL** : https://university.redis.com/courses/ru102py/

**Durée** : ~4 heures
**Contenu** :
- Client Python (redis-py)
- Patterns d'application
- Gestion de cache
- Queues et pub/sub

**Pour qui** : Développeurs Python

---

#### RU102J: Redis for Java Developers
**URL** : https://university.redis.com/courses/ru102j/

**Durée** : ~4 heures
**Contenu** :
- Client Java (Jedis, Lettuce)
- Spring Data Redis
- Patterns d'intégration
- Best practices Java

**Pour qui** : Développeurs Java/Spring

---

### 🟡 Niveau Intermédiaire

#### RU201: RediSearch
**URL** : https://university.redis.com/courses/ru201/

**Durée** : ~2 heures
**Contenu** :
- Indexation full-text
- Requêtes complexes
- Agrégations
- Vector similarity search

**Pour qui** : Développeurs voulant implémenter un moteur de recherche

**Ce que vous apprendrez** :
- Créer et gérer des index
- Effectuer des recherches full-text
- Utiliser les agrégations
- Optimiser les performances de recherche

---

#### RU202: Redis Streams
**URL** : https://university.redis.com/courses/ru202/

**Durée** : ~2 heures
**Contenu** :
- Concepts des Streams
- Producer/Consumer patterns
- Consumer Groups
- Traitement parallèle

**Pour qui** : Développeurs travaillant avec des flux de données

**Ce que vous apprendrez** :
- Architecture event-driven avec Redis
- Implémenter des pipelines de traitement
- Gérer le back-pressure
- Comparer Streams vs Pub/Sub

---

#### RU203: Querying, Indexing, and Full-Text Search
**URL** : https://university.redis.com/courses/ru203/

**Durée** : ~3 heures
**Contenu** :
- RedisJSON et indexation
- Recherche sur documents JSON
- Requêtes avancées
- Optimisation des index

**Pour qui** : Développeurs voulant stocker et requêter du JSON

---

### 🔴 Niveau Avancé

#### RU301: Running Redis at Scale
**URL** : https://university.redis.com/courses/ru301/

**Durée** : ~6 heures
**Contenu** :
- Replication et Sentinel
- Redis Cluster
- Sharding et partitioning
- Monitoring et observabilité
- Performance tuning

**Pour qui** : DevOps, SRE, architectes

**Ce que vous apprendrez** :
- Déployer Redis en haute disponibilité
- Configurer et gérer un cluster
- Monitorer et optimiser les performances
- Gérer les opérations en production

---

#### RU302: Redis Security
**URL** : https://university.redis.com/courses/ru302/

**Durée** : ~2 heures
**Contenu** :
- ACLs (Access Control Lists)
- TLS/SSL encryption
- Network security
- Best practices de sécurité

**Pour qui** : DevOps, administrateurs, security engineers

**Ce que vous apprendrez** :
- Sécuriser Redis en production
- Configurer les ACLs granulaires
- Implémenter le chiffrement
- Auditer et monitorer les accès

---

#### RU330: Redis as a Vector Database
**URL** : https://university.redis.com/courses/ru330/

**Durée** : ~2-3 heures
**Contenu** :
- Vector embeddings
- Similarity search
- RAG (Retrieval-Augmented Generation)
- Intégration avec LLMs

**Pour qui** : Développeurs IA/ML, data scientists

**Ce que vous apprendrez** :
- Stocker et rechercher des vecteurs
- Implémenter des systèmes RAG
- Optimiser les recherches de similarité
- Intégrer avec OpenAI, LangChain, etc.

---

### 🎯 Cours spécialisés

#### RU204: Redis Probabilistic Data Structures
**URL** : https://university.redis.com/courses/ru204/

**Contenu** :
- HyperLogLog
- Bloom Filters
- Cuckoo Filters
- Top-K, Count-Min Sketch

---

## 🗺️ Parcours d'apprentissage recommandés

### Parcours Développeur

```
1. RU101: Introduction to Redis Data Structures (2h)
   └─> Comprendre les bases

2. RU102[JS/PY/J]: Redis for [Your Language] (4h)
   └─> Intégration dans votre stack

3. RU202: Redis Streams (2h)
   └─> Gestion des flux de données

4. RU201: RediSearch (2h)
   └─> Indexation et recherche

Total: ~10-12 heures
```

**Certification suggérée** : Redis Certified Developer

---

### Parcours DevOps/SRE

```
1. RU101: Introduction to Redis Data Structures (2h)
   └─> Comprendre les fondamentaux

2. RU301: Running Redis at Scale (6h)
   └─> Architecture distribuée

3. RU302: Redis Security (2h)
   └─> Sécurisation en production

Total: ~10 heures
```

**Certification suggérée** : Redis Certified Administrator

---

### Parcours Data Science / IA

```
1. RU101: Introduction to Redis Data Structures (2h)
   └─> Bases de Redis

2. RU330: Redis as a Vector Database (3h)
   └─> Vector search et RAG

3. RU201: RediSearch (2h)
   └─> Indexation avancée

4. RU204: Probabilistic Data Structures (2h)
   └─> Structures probabilistes

Total: ~9 heures
```

---

### Parcours Architecte

```
1. RU101: Introduction to Redis Data Structures (2h)
2. RU301: Running Redis at Scale (6h)
3. RU201: RediSearch (2h)
4. RU202: Redis Streams (2h)
5. RU302: Redis Security (2h)

Total: ~14 heures
```

**Certifications suggérées** : Developer + Administrator

---

## 🎮 Format des cours et labs

### Structure typique d'un cours

1. **Vidéos** (10-15 min chacune)
   - Explications théoriques
   - Démonstrations pratiques
   - Slides téléchargeables

2. **Labs interactifs**
   - Environnement Redis préconfiguré
   - Exercices guidés pas à pas
   - Validation automatique
   - Code d'exemple fourni

3. **Quiz**
   - Questions à choix multiples
   - Validation des connaissances
   - Retours immédiats

4. **Projet final**
   - Mise en pratique complète
   - Scénario réel
   - Code à compléter

### Exemple de lab interactif

Les labs se font directement dans le navigateur avec :
- Terminal redis-cli intégré
- Éditeur de code en ligne
- Environnement Redis préconfigré
- Feedback en temps réel

**Pas d'installation locale requise** !

## 📜 Badges et certificats

### Badges de cours

À la fin de chaque cours, vous recevez :
- **Badge numérique vérifiable**
- **Certificat de complétion** (PDF)
- **Partage automatique** sur LinkedIn possible

### Progression tracking

- Visualisation de votre progression
- Historique des cours complétés
- Badges collectés
- Temps total d'apprentissage

**URL de votre profil** : `https://university.redis.com/certificates/[votre-id]`

## 🚀 Comment commencer

### Étape 1 : Créer un compte

1. Allez sur https://university.redis.com/
2. Cliquez sur "Sign Up" (gratuit)
3. Créez votre compte avec email
4. Confirmez votre email

### Étape 2 : Choisir votre parcours

1. Consultez le catalogue de cours
2. Sélectionnez selon votre profil
3. Commencez par RU101 si débutant

### Étape 3 : Suivre le cours

1. Regardez les vidéos (sous-titres disponibles)
2. Complétez les labs interactifs
3. Passez les quiz
4. Terminez le projet final

### Étape 4 : Obtenir votre badge

1. Complétez 100% du cours
2. Réussissez les quiz (80% minimum)
3. Badge automatiquement délivré
4. Ajoutez-le à votre profil LinkedIn

## 💡 Conseils d'apprentissage

### Pour maximiser votre apprentissage

- ✅ **Prenez des notes** pendant les vidéos
- ✅ **Pratiquez en parallèle** sur votre machine locale
- ✅ **Complétez les labs** sans sauter d'étapes
- ✅ **Relisez la documentation** pour approfondir
- ✅ **Rejoignez les discussions** dans les forums du cours
- ✅ **Appliquez immédiatement** dans vos projets

### Ordre recommandé

1. **Commencez simple** : RU101 pour tous
2. **Spécialisez-vous** selon votre langage (RU102x)
3. **Approfondissez** avec les cours avancés (RU2xx)
4. **Maîtrisez l'ops** avec RU301 si pertinent

### Gestion du temps

- **2h/semaine** : 1 cours/mois
- **5h/semaine** : 1 cours/semaine
- **10h/semaine** : Parcours complet en 1 mois

**Conseil** : Mieux vaut 30 min régulières que 3h sporadiques !

## 🔄 Mises à jour des cours

### Fréquence de mise à jour

- Cours mis à jour **2-4 fois/an**
- Ajout de nouveaux cours **trimestriellement**
- Contenu aligné sur les **dernières versions** de Redis

### Nouveautés récentes (2024-2025)

- **RU330** : Redis as a Vector Database (nouveau)
- **Mise à jour RU301** : Contenu Redis 7.x
- **Labs améliorés** : Interface plus moderne
- **Support mobile** : Cours accessibles sur tablette

## 📱 Accès et compatibilité

### Plateformes supportées

- ✅ Desktop (Chrome, Firefox, Safari, Edge)
- ✅ Tablette (iPad, Android)
- ⚠️ Mobile (lecture vidéo OK, labs limités)

### Prérequis techniques

- Navigateur moderne
- Connexion Internet stable
- Pas d'installation logicielle requise

### Accessibilité

- Sous-titres anglais sur toutes les vidéos
- Transcriptions téléchargeables
- Interface adaptable

## 🎯 Après Redis University

### Prochaines étapes

1. **Pratiquez** sur des projets réels
2. **Passez une certification** (voir sections 19.3 et 19.4)
3. **Rejoignez la communauté** (voir section 19.6)
4. **Lisez des livres** spécialisés (voir section 19.5)

### Ressources complémentaires

- **Redis documentation** : Approfondissement technique
- **Redis blogs** : Cas d'usage et best practices
- **GitHub examples** : Code d'exemple réel
- **Stack Overflow** : Support communautaire

## 🆚 Redis University vs Autres ressources

| Critère | Redis University | Udemy/Coursera | YouTube |
|---------|-----------------|----------------|---------|
| **Coût** | Gratuit | Payant (~20-100€) | Gratuit |
| **Officiel** | ✅ Oui | ❌ Non | ❌ Non |
| **Labs** | ✅ Intégrés | ⚠️ Variables | ❌ Non |
| **Certification** | ✅ Officielle | ⚠️ Plateforme | ❌ Non |
| **Mise à jour** | ✅ Régulière | ⚠️ Variable | ⚠️ Variable |
| **Structuré** | ✅ Parcours | ✅ Cours | ❌ Dispersé |

**Recommandation** : Commencez par Redis University (officiel et gratuit)

## 📊 Statistiques Redis University

- **100,000+** étudiants inscrits
- **10+** cours disponibles
- **50+** heures de contenu vidéo
- **100+** labs interactifs
- **20+** langues de sous-titres (sélection)
- **95%** taux de satisfaction

## 🔗 Liens directs - Récapitulatif

### Accès rapide aux cours

| Cours | URL |
|-------|-----|
| RU101 - Introduction | https://university.redis.com/courses/ru101/ |
| RU102JS - JavaScript | https://university.redis.com/courses/ru102js/ |
| RU102PY - Python | https://university.redis.com/courses/ru102py/ |
| RU102J - Java | https://university.redis.com/courses/ru102j/ |
| RU201 - RediSearch | https://university.redis.com/courses/ru201/ |
| RU202 - Streams | https://university.redis.com/courses/ru202/ |
| RU301 - Running at Scale | https://university.redis.com/courses/ru301/ |
| RU302 - Security | https://university.redis.com/courses/ru302/ |
| RU330 - Vector Database | https://university.redis.com/courses/ru330/ |

### Autres liens utiles

| Ressource | URL |
|-----------|-----|
| Page principale | https://university.redis.com/ |
| Catalogue complet | https://university.redis.com/courses/ |
| FAQ | https://university.redis.com/faq/ |
| Support | support@redis.com |

## ❓ FAQ - Redis University

**Q : Les cours sont-ils vraiment gratuits ?**
R : Oui, tous les cours sont 100% gratuits. Seules les certifications officielles sont payantes (~100-200€).

**Q : Combien de temps les badges sont-ils valides ?**
R : Les badges n'expirent jamais. Votre progression est conservée indéfiniment.

**Q : Puis-je télécharger les vidéos ?**
R : Non, les vidéos sont en streaming uniquement. Mais vous pouvez télécharger les slides et le code.

**Q : Y a-t-il un support si je suis bloqué ?**
R : Oui, chaque cours a un forum de discussion. Vous pouvez aussi contacter le support.

**Q : Les cours couvrent-ils Valkey ?**
R : Non, les cours sont spécifiques à Redis. Pour Valkey, référez-vous à la documentation officielle.

**Q : Faut-il suivre les cours dans l'ordre ?**
R : RU101 est recommandé en premier. Ensuite, l'ordre dépend de vos besoins.

**Q : Les labs fonctionnent-ils sur tous les navigateurs ?**
R : Oui, sur tous les navigateurs modernes. Chrome et Firefox sont recommandés.

## 🎓 Conclusion

Redis University est **la ressource d'apprentissage officielle** incontournable pour :
- ✅ Apprendre Redis gratuitement
- ✅ Pratiquer avec des labs interactifs
- ✅ Obtenir des badges vérifiables
- ✅ Se préparer aux certifications

**Action recommandée** : Créez votre compte dès maintenant et commencez par RU101 !

---


⏭️ [Certification Redis Certified Developer](/19-ressources-certification/03-certification-redis-developer.md)
