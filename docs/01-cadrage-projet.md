# 01 - Cadrage du projet

## Identité

- **Nom et prénom** : 12
- **Dépôt GitHub** : `https://github.com/nayimjr08/projet-cicd`
- **Date de démarrage** : 27 avril 2025

## Objectif

Mettre en place une chaîne CI/CD permettant de construire, tester, publier et promouvoir une image Docker Nginx contenant un site web statique pour le scénario Catal-Log. L'ensemble de la chaîne repose sur GitHub Actions et GitHub Container Registry (GHCR), sans serveur distant à administrer.

## Contraintes du projet

- Travail individuel.
- Aucune infrastructure fournie, préparée, administrée ou maintenue par le formateur.
- Pas de serveur distant, pas de SSH, pas de cloud provider imposé.
- Les traitements principaux sont exécutés dans GitHub Actions (runners hébergés par GitHub).
- Docker local ou Docker Compose sont utilisés si l'environnement personnel le permet ; sinon la limitation doit être justifiée.
- Une VM personnelle peut être utilisée si disponible ; sinon la non-utilisation doit être justifiée.

## Choix personnels

**Dépôt** : le dépôt est public afin de faciliter la vérification des preuves par le formateur. Le nommage suit la convention `projet-cicd` pour rester lisible et cohérent avec le contexte EC06.

**Stratégie de tags** : trois types de tags sont utilisés sur GHCR :
- `sha-<7 caractères>` : tag automatique lié au commit, pour la traçabilité exacte ;
- `latest` : tag glissant mis à jour à chaque publication depuis la branche `main` ;
- `production-simulee` : tag appliqué manuellement lors de la promotion, sans rebuild.

**Environnement local** : À compléter — indiquer si Docker Desktop / Docker Engine est installé localement, si les tests `docker compose up --build` ont pu être réalisés, ou justifier l'impossibilité.

**VM personnelle** : À compléter — indiquer si une VM a été utilisée pour les tests locaux, ou justifier la non-utilisation (par exemple : Docker disponible directement sur la machine hôte, donc pas de nécessité de VM intermédiaire).

**Branche de travail** : le développement se fait directement sur `main` pour ce projet pédagogique. En production réelle, une stratégie de branching (GitFlow, trunk-based) serait mise en place.
