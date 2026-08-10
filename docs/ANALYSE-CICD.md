# BobApp — Mise en place de la CI/CD : documentation et analyse

Ce document accompagne la mise en place du pipeline CI/CD sur GitHub Actions. Il explique les étapes du workflow, propose des seuils de qualité (KPI) à valider avec Bob, et dresse un premier état des lieux des métriques du projet.

## 1. Étapes du workflow CI/CD

Le workflow est défini dans [`.github/workflows/ci.yml`](../.github/workflows/ci.yml) et se déclenche à chaque `push` ou `pull request` sur la branche `main`. Il est composé de 4 jobs, chacun conditionné à la réussite du précédent.

| # | Job | Objectif | Se déclenche si |
|---|-----|----------|------------------|
| 1 | **`backend-tests`** | Compile le back-end (Java 11 / Spring Boot), exécute les tests unitaires avec Maven, et génère le rapport de couverture Jacoco. | À chaque push/PR |
| 2 | **`frontend-tests`** | Installe les dépendances Angular, exécute les tests unitaires (Jasmine/Karma) en mode headless, et génère le rapport de couverture (HTML + lcov). | À chaque push/PR |
| 3 | **`sonarcloud`** | Relance les tests des deux stacks pour disposer des rapports de couverture, puis lance l'analyse de qualité de code via SonarCloud (bugs, vulnérabilités, code smells, duplication, couverture). | Si `backend-tests` **et** `frontend-tests` ont réussi |
| 4 | **`docker-deploy`** | Build les images Docker du back-end et du front-end, et les publie sur Docker Hub (tags `latest` et `<sha-du-commit>`). | Si `backend-tests`, `frontend-tests` **et** `sonarcloud` ont réussi, et uniquement sur un push vers `main` (pas sur les PR) |

Cette structure séquentielle garantit qu'**aucune image n'est déployée si les tests échouent ou si la qualité de code ne passe pas le Quality Gate**, conformément à la demande initiale.

### Outils utilisés
- **Jacoco** : couverture de code back-end (Java), rapport HTML + XML
- **Karma / Istanbul** : couverture de code front-end (Angular), rapport HTML + lcov
- **SonarCloud** : analyse statique de la qualité de code (bugs, vulnérabilités, code smells, duplication, couverture agrégée back + front)
- **Docker Hub** : registre des images `bobapp-back` et `bobapp-front`

## 2. KPIs proposés

Pour objectiver la qualité du projet dans le temps, voici une proposition de seuils à valider avec Bob. Ces seuils sont pilotés via le **Quality Gate SonarCloud**, qui peut bloquer une pull request si l'un d'eux n'est pas respecté.

| KPI | Seuil proposé | Justification |
|-----|----------------|----------------|
| **Couverture de code (New Code)** *(obligatoire)* | **≥ 80 %** sur le code nouvellement ajouté/modifié | C'est le seuil par défaut du profil "Sonar Way" de SonarCloud. Cibler le *New Code* plutôt que l'ensemble du projet permet d'exiger une bonne couverture sur tout ce qui est écrit désormais, sans bloquer le projet à cause du code existant (couverture globale actuelle : 35,2 %, cf. section 3). |
| **New Blocker / Critical issues** | **0** | Repris de la suggestion de Bob. Aucune nouvelle anomalie bloquante ou critique (bug ou vulnérabilité) ne doit être introduite dans une pull request. |
| **Maintainability Rating** | **A** | Limite l'accumulation de dette technique (complexité inutile, duplication de logique) au fil des évolutions. |

**Recommandation** : commencer avec ces seuils sur le *New Code* uniquement (ce que fait déjà le profil par défaut), puis relever progressivement l'exigence de couverture globale du projet à mesure que les développeurs ajoutent des tests sur l'existant.

## 3. Métriques actuelles (après premier passage du pipeline)

Relevé sur le dashboard SonarCloud du projet, après exécution complète du pipeline :

| Métrique | Valeur | Statut |
|----------|--------|--------|
| Quality Gate | **Passed** | ✅ Toutes les conditions du profil "Sonar Way" sont respectées |
| Couverture de code (Overall Code) | **35,2 %** | ⚠️ En dessous du seuil cible de 80 % — attendu, car ce chiffre porte sur l'ensemble du code existant, peu testé avant cette mission |
| Lignes de code analysées | 225 | — |
| Issues ouvertes | 7 | À trier par sévérité (voir section 5) |
| Duplication de code | 0,0 % | ✅ |
| Security Rating | B | 1 issue de sécurité ouverte à examiner |

*Ces chiffres évoluent à chaque nouvelle analyse ; ils reflètent l'état du projet au moment de la rédaction de ce document. Le dashboard SonarCloud fait foi pour le suivi en continu.*

## 4. Retours utilisateurs (Notes et avis)

Note moyenne actuelle : **2,0 / 5 ⭐**, sur la base des avis suivants relevés en ligne :

| Note | Avis |
|------|------|
| ★☆☆☆☆ | *"Je mets une étoile car je ne peux pas en mettre zéro ! Impossible de poster une suggestion de blague, le bouton tourne et fait planter mon navigateur !"* |
| ★★☆☆☆ | *"#BobApp j'ai remonté un bug sur le post de vidéo il y a deux semaines et il est encore présent ! Les devs vous faites quoi ????"* |
| ★☆☆☆☆ | *"Ça fait une semaine que je ne reçois plus rien, j'ai envoyé un email il y a 5 jours mais toujours pas de nouvelles..."* |
| ★★★☆☆ | *"J'ai supprimé ce site de mes favoris ce matin, dommage, vraiment dommage."* |

Ces retours mettent en évidence, au-delà des bugs eux-mêmes, un **délai de correction et de réponse au support jugé trop long** par les utilisateurs — un point de process à adresser en parallèle des corrections techniques.

## 5. Problèmes à résoudre en priorité

Détail des issues relevées sur le dashboard SonarCloud (onglet *Issues*) :

| Fichier | Problème | Type | Sévérité |
|---------|----------|------|----------|
| `back/.../service/JokeService.java` (L22) | *"Save and re-use this Random"* — une nouvelle instance de `Random` est créée à chaque appel au lieu d'être réutilisée | Bug — Reliability | **Critical** |
| `back/.../data/JsonReader.java` (L24) | Fonctionnalité de debug non désactivée en production | Vulnerability — Security | Minor |
| `back/.../controller/JokeController.java` (L21) | Usage d'un type générique wildcard à corriger | Code Smell — Maintainability | Critical |
| `back/.../model/Joke.java` (L4) | Champ `joke` à renommer (nommage peu explicite) | Code Smell — Maintainability | Major |
| `back/.../data/JsonReader.java` (L29, L42) | Ordre des modifiers non conforme / déclaration d'exception inutile | Code Smell — Maintainability | Minor |
| `front/src/app/app.component.ts` (L11), `jokes.service.ts` (L9) | Membres à marquer `readonly` | Code Smell — Maintainability | Major |

### Analyse croisée avec les retours utilisateurs

1. **Priorité n°1 — Bug critique sur `JokeService.java`** : la non-réutilisation de l'objet `Random` est **la seule anomalie de type Bug (vs. simple Code Smell) détectée**, en sévérité Critical. Elle est directement plausible comme cause technique de l'avis *"Ça fait une semaine que je ne reçois plus rien"* : un `Random` mal utilisé peut produire des tirages peu variés ou un comportement de sélection de blague dégradé. **À corriger en priorité et à tester spécifiquement avant la prochaine mise en production.**
2. **Fonctionnalités manquantes dans le périmètre analysé** : les avis mentionnant un plantage lors de l'*"envoi d'une suggestion de blague"* et un bug sur le *"post de vidéo"* ne trouvent **aucune correspondance dans le code actuel** (`JokeController`, `JokeService`, `Joke`, `JsonReader`, `app.component.ts`, `jokes.service.ts` ne couvrent que la lecture/l'affichage d'une blague). **Point à clarifier avec Bob** : ces fonctionnalités existent-elles ailleurs (autre repo, autre branche), ont-elles été retirées, ou sont-elles simplement pas encore développées dans le MVP actuel ?
3. **Vulnérabilité de sécurité (mineure)** sur `JsonReader.java` : une fonctionnalité de debug resterait active en production. À corriger avant toute mise en production réelle, même si la sévérité est faible.
4. **Le reste (5 issues)** sont des Code Smells de maintenabilité (nommage, lisibilité, conventions Java) : aucun impact utilisateur direct identifié, à traiter dans un second temps pour limiter la dette technique.
5. **Réactivité du support** *(hors périmètre technique)* : plusieurs avis mentionnent une absence de réponse après plusieurs jours ; à remonter à Bob côté organisation/support, en parallèle des corrections de code.

---
*Document rédigé dans le cadre de la mise en place de la CI/CD pour BobApp — Projet 10 OpenClassrooms.*