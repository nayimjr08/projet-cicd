# 05 - Preuve recette simulée

## Workflow de validation recette

- **Workflow concerné** : `03-promote.yml` (job `validate-recette`)
- **Environnement GitHub** : `recette`
- **Tag source validé** : À compléter — par exemple `latest` ou `sha-a1b2c3d`
- **Digest observé** : À compléter — visible dans le `$GITHUB_STEP_SUMMARY` du run
- **Lien du run** : À compléter — `https://github.com/nayimjr08/projet-cicd/actions/runs/XXXXXXXXXX`

## Résultat

Le workflow de promotion est déclenché manuellement via `workflow_dispatch`. Le paramètre `source_tag` indique quel tag GHCR doit être validé (par défaut `latest`).

Lors de la phase de recette simulée, le job `validate-recette` exécute les opérations suivantes :
1. **Pull de l'image** depuis GHCR (sans rebuild) : `docker pull ghcr.io/.../projet-cicd:<tag>`
2. **Démarrage du conteneur** en mode détaché sur le port 8080
3. **Tests HTTP** : requêtes `curl` sur `/` et `/version.json` pour vérifier que le site répond correctement
4. **Affichage du digest** dans le résumé du workflow pour traçabilité

Le test HTTP confirme que :
- La page d'accueil est servie correctement (code HTTP 200, contenu HTML valide)
- Le fichier `version.json` est accessible et retourne les informations de version attendues

L'environnement GitHub `recette` permet de simuler un gate de validation : en production réelle, cet environnement pourrait nécessiter une approbation manuelle, des reviewers désignés ou des règles de protection spécifiques.

## Preuve

À compléter — insérer une capture d'écran du run GitHub Actions montrant :
- Le job `validate-recette` au statut ✅
- Le résumé (`GITHUB_STEP_SUMMARY`) affichant l'image source et le digest observé
