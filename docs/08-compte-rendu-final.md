# 08 - Compte rendu final

## 1. Synthèse

Ce projet met en place une chaîne CI/CD complète pour l'entreprise fictive Catal-Log. Un site web statique est conteneurisé dans une image Docker Nginx, construite et testée automatiquement à chaque push via GitHub Actions, publiée dans GitHub Container Registry avec des tags traçables, validée en recette simulée puis promue en production simulée sans reconstruction de l'image. L'ensemble repose sur des runners hébergés par GitHub, sans infrastructure externe à administrer.

## 2. Fonctionnement technique

Le chemin complet d'un changement, du commit à la production simulée :

1. **Commit et push** : le développeur modifie le code (site, Dockerfile, docs) et pousse sur la branche `main`.

2. **Build et tests (01-ci.yml)** : le workflow se déclenche automatiquement. Il vérifie la présence des fichiers obligatoires, valide la syntaxe Docker Compose, construit l'image Docker avec le SHA du commit comme tag, démarre un conteneur et exécute des requêtes HTTP pour vérifier que le site répond correctement sur `/` et `/version.json`. Le contenu HTML est également vérifié par un `grep`.

3. **Publication GHCR (02-publish-ghcr.yml)** : sur la branche `main`, le workflow construit l'image et la publie dans GHCR. Deux tags sont appliqués : `sha-23a01d2` pour la traçabilité exacte et `latest` comme tag glissant. Le digest SHA256 est affiché dans le résumé du workflow.

4. **Validation recette (03-promote.yml, job 1)** : le workflow est déclenché manuellement via `workflow_dispatch` avec le tag source en paramètre. L'image est téléchargée depuis GHCR (pas de rebuild), démarrée et testée par des requêtes HTTP dans l'environnement GitHub `recette`.

5. **Promotion production-simulee (03-promote.yml, job 2)** : si la recette est validée, l'image existante est re-taguée `production-simulee` et poussée sur GHCR. Aucun rebuild n'a lieu : c'est le même artefact binaire, identifiable par son digest.

## 3. Conteneurisation (C12)

### Dockerfile

L'image est basée sur `nginx:1.27-alpine`, choisie pour sa légèreté (~40 Mo) et sa surface d'attaque réduite. Le Dockerfile :
- copie le contenu du dossier `site/` dans le répertoire de publication Nginx ;
- applique les permissions adéquates (`chmod 755`) ;
- expose le port 80 ;
- intègre un healthcheck basé sur `wget` pour vérifier que Nginx répond.

```dockerfile
FROM nginx:1.27-alpine
COPY site/ /usr/share/nginx/html/
RUN chmod -R 755 /usr/share/nginx/html
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -q -O - http://127.0.0.1/ >/dev/null || exit 1
```

### Exécution conteneurisée

L'image est exécutable localement :
```bash
docker build -t projet-cicd-nginx:local .
docker run --rm -p 8080:80 projet-cicd-nginx:local
```
Le site est alors accessible sur `http://127.0.0.1:8080/`. La page affiche le pipeline CI/CD complet avec les cartes techniques (conteneurisation, automatisation, sécurité, orchestration) et charge dynamiquement les informations de `version.json`.

### Preuves

- Le workflow `01-ci.yml` prouve le build et le test automatisé à chaque push.
- Le workflow `02-publish-ghcr.yml` prouve la publication dans GHCR avec tag `sha-23a01d2` et `latest`.
- L'image est visible sur https://github.com/nayimjr08/projet-cicd/pkgs/container/projet-cicd

## 4. Orchestration et scaling (C13)

### compose.yml

Le fichier `compose.yml` décrit deux services coordonnés :
- **web** : conteneur Nginx servant le site, avec healthcheck et réseau dédié ;
- **tester** : conteneur léger (`curlimages/curl`) qui attend la bonne santé du service web puis exécute des tests HTTP de validation.

Les deux services communiquent via un réseau Bridge isolé (`cicd_net`).

### Simulation de scaling

```bash
docker compose up -d --scale web=2
docker compose ps
```

Cette commande crée deux instances du service `web`. Les deux conteneurs répondent sur le port 80 en interne, mais sans load balancer, le trafic n'est pas réparti automatiquement.

### Limites

Docker Compose est un outil d'orchestration locale, adapté au développement et à la démonstration. Il ne répond pas aux exigences d'une production réelle :

| Besoin production | Docker Compose | Kubernetes |
|-------------------|---------------|------------|
| Haute disponibilité | ❌ Un seul hôte | ✅ Multi-nœuds, auto-healing |
| Répartition de charge | ❌ Pas de load balancer intégré | ✅ Service + Ingress |
| Rolling update | ❌ Arrêt puis redémarrage | ✅ Déploiement progressif |
| Auto-scaling | ❌ Manuel (`--scale`) | ✅ HPA basé sur les métriques |
| Gestion des secrets | ❌ Variables en clair | ✅ Secrets Kubernetes chiffrés |
| Supervision | ❌ Pas intégrée | ✅ Intégration Prometheus/Grafana |
| Rollback automatisé | ❌ Manuel | ✅ Rollback automatique sur échec de probe |

Docker Compose reste pertinent pour ce projet pédagogique : il permet de comprendre la coordination de conteneurs et de valider le fonctionnement de l'image avant publication.

## 5. Automatisation et sécurité (C14)

### Workflows GitHub Actions

Trois workflows structurent la chaîne :

| Workflow | Déclencheur | Rôle |
|----------|------------|------|
| `01-ci.yml` | Push, PR, manuel | Build, tests automatisés, validation |
| `02-publish-ghcr.yml` | Push sur `main`, manuel | Publication GHCR avec tags et digest |
| `03-promote.yml` | Manuel (`workflow_dispatch`) | Validation recette + promotion sans rebuild |

### GHCR et traçabilité

GitHub Container Registry stocke les images avec leurs tags et digests. Chaque image publiée est liée à un commit précis via le tag `sha-23a01d2`, ce qui permet de retrouver exactement quel code a produit quelle image.

### GITHUB_TOKEN

Le `GITHUB_TOKEN` est un token éphémère, généré automatiquement par GitHub Actions, dont les permissions sont limitées au bloc `permissions` du workflow. Il n'est jamais stocké dans le code.

### Permissions minimales

Chaque workflow déclare uniquement les permissions nécessaires :
- `01-ci.yml` : `contents: read` (lecture seule du code)
- `02-publish-ghcr.yml` : `contents: read` + `packages: write` (publication GHCR)
- `03-promote.yml` : `contents: read` + `packages: write` (promotion GHCR)

### Absence de secrets dans le code

Aucun secret n'est stocké dans le dépôt. L'authentification GHCR repose exclusivement sur le `GITHUB_TOKEN` automatique.

## 6. Production réelle

### Gestion des secrets

En production réelle, les secrets (tokens, clés API, credentials) ne doivent jamais figurer dans le code source. Ils doivent être gérés via :
- **GitHub Secrets** pour les pipelines CI/CD (tokens de registre, clés de déploiement) ;
- **Un coffre de secrets** (HashiCorp Vault, AWS Secrets Manager) pour les applications, avec rotation automatique et audit des accès ;
- **Des variables d'environnement injectées** au runtime, jamais en dur dans les images Docker ou les fichiers de configuration versionnés.

Le `GITHUB_TOKEN` utilisé dans ce projet est un bon exemple de gestion sécurisée : éphémère, scopé et non stocké.

### Rollback

Le rollback s'appuie sur la traçabilité de la chaîne :
1. Identifier la version précédente via son tag (`sha-<commit>`) ou son digest ;
2. Relancer le workflow `03-promote.yml` avec le tag de la version à restaurer ;
3. L'image précédente est re-taguée `production-simulee` et poussée — sans rebuild, donc sans risque de divergence.

Le digest garantit que l'image restaurée est bit-à-bit identique à celle qui était en production précédemment.

### Sauvegarde / restauration

Éléments à sauvegarder :
- **Dépôt Git** : clone miroir vers un emplacement distinct (autre provider, stockage objet) ;
- **Images GHCR** : politique de rétention des tags, export `docker save` pour archivage ;
- **Secrets** : documentés dans un coffre de secrets externe, jamais dans le code ;
- **Configuration des environnements** : règles de protection, reviewers, à documenter pour recréation.

Restauration : restaurer le dépôt depuis le miroir, recréer les secrets et les environnements GitHub, vérifier les workflows avec un push de test, reconstruire l'image depuis le commit tracé si nécessaire.

### Éléments complémentaires

**Contrôle des vulnérabilités** : intégrer un scanner (Trivy, Grype) dans le pipeline CI pour bloquer la publication d'images contenant des CVE critiques. L'image Alpine réduit déjà la surface d'attaque.

**Validation manuelle avant production** : configurer les environnements GitHub avec des reviewers obligatoires et un timer d'attente. Cela empêche toute promotion non approuvée et permet d'annuler un déploiement avant son exécution.

## 7. Preuves

| Preuve | Lien |
|--------|------|
| Dépôt GitHub | https://github.com/nayimjr08/projet-cicd |
| Run CI (01-ci.yml) | https://github.com/nayimjr08/projet-cicd/actions (workflow "01 - CI") |
| Run publication (02-publish-ghcr.yml) | https://github.com/nayimjr08/projet-cicd/actions (workflow "02 - Publication GHCR") |
| Image GHCR (tags + digest) | https://github.com/nayimjr08/projet-cicd/pkgs/container/projet-cicd |
| Run promotion (03-promote.yml) | https://github.com/nayimjr08/projet-cicd/actions (workflow "03 - Promotion") |
| Environnement recette | Visible dans le run du workflow 03 — job `validate-recette` |
| Environnement production-simulee | Visible dans le run du workflow 03 — job `promote-production-simulee` |
| Test local Docker / Docker Compose | Réalisé avec succès — voir `docs/03-fiche-tests.md` |

## 8. Difficultés et apprentissages

- **Comprendre le GITHUB_TOKEN** : le token est généré automatiquement par GitHub Actions, limité aux permissions déclarées dans le workflow, et détruit à la fin du run. Ce mécanisme évite de stocker des secrets dans le code.
- **Différence tag vs digest** : le tag est une étiquette humaine, mutable et pratique. Le digest est un identifiant cryptographique, immuable et fiable. Les deux sont nécessaires : le tag pour nommer, le digest pour garantir.
- **Promotion sans rebuild** : comprendre pourquoi on ne reconstruit pas l'image lors de la promotion a été un point clé. Reconstruire introduirait un risque de divergence (dépendances mises à jour, environnement différent). La promotion par re-tag garantit l'identité binaire.
- **Limites de Docker Compose** : Docker Compose est utile pour coordonner des conteneurs en local, mais n'offre ni haute disponibilité, ni load balancing, ni auto-scaling. En production, Kubernetes ou un orchestrateur similaire prendrait le relais.
- **Permissions minimales** : appliquer le principe du moindre privilège dans les workflows réduit l'impact d'une éventuelle compromission. Chaque workflow ne demande que ce dont il a strictement besoin.
- **Cycle complet CI/CD** : du commit au déploiement, chaque étape (build, test, publication, validation, promotion) ajoute de la confiance dans l'artefact. La traçabilité de bout en bout est assurée par les tags, digests et logs GitHub Actions.
