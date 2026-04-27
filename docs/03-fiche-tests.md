# 03 - Fiche tests

## Test automatisé GitHub Actions

- **Workflow concerné** : `01-ci.yml`
- **Lien vers le run réussi** : À compléter — `https://github.com/nayimjr08/projet-cicd/actions/runs/XXXXXXXXXX`
- **Ce qui est testé** :
  - Présence des fichiers obligatoires (`Dockerfile`, `compose.yml`, `site/index.html`, `site/version.json`, `docs/08-compte-rendu-final.md`)
  - Validité de la syntaxe Docker Compose (`docker compose config`)
  - Construction de l'image Docker (`docker build`)
  - Démarrage du conteneur et réponse HTTP sur `/` et `/version.json`
  - Vérification du contenu de la page (grep sur "Projet CICD")
- **Résultat** : À compléter — tous les steps doivent être au vert (✅)

## Test local Docker ou Docker Compose

### Situation A — Test réalisé

Commandes utilisées :

```bash
# Construction et test avec Docker seul
docker build -t projet-cicd-nginx:local .
docker run --rm -d --name test-local -p 8080:80 projet-cicd-nginx:local
curl -fsS http://127.0.0.1:8080/
curl -fsS http://127.0.0.1:8080/version.json
docker rm -f test-local
```

Ou avec Docker Compose :

```bash
docker compose up --build
# Dans un second terminal :
curl -fsS http://127.0.0.1:8080/
curl -fsS http://127.0.0.1:8080/version.json
docker compose down
```

Résultat observé : À compléter — décrire ce qui s'affiche (page HTML du site Catal-Log, contenu JSON de version.json, le service tester qui valide les deux endpoints).

### Situation B — Test local impossible

_(Supprimer cette section si le test local a été réalisé, ou supprimer la section A si le test local est impossible.)_

Justification : À compléter — par exemple : Docker non disponible sur la machine personnelle, restrictions réseau, machine trop limitée en ressources, etc.

## Simulation de scaling

Si l'environnement le permet :

```bash
docker compose up -d --scale web=2
docker compose ps
```

Résultat observé : À compléter — par exemple : "La commande crée deux instances du service `web`. La commande `docker compose ps` montre bien deux conteneurs `web` en état `running`. Les deux répondent sur le port 80 en interne du réseau Docker, mais sans load balancer configuré, les requêtes ne sont pas automatiquement réparties."

> **Note** : lors du scaling, il faut retirer le mapping de port `ports: "8080:80"` du compose.yml car deux conteneurs ne peuvent pas mapper le même port hôte. On conserve uniquement `expose: "80"` pour la communication interne.

## Tests complémentaires

En complément des tests de base, les vérifications suivantes ont été réalisées :

| Test | Commande / méthode | Résultat attendu |
|------|-------------------|------------------|
| Healthcheck Docker | `docker inspect --format='{{.State.Health.Status}}' <container>` | `healthy` |
| Taille de l'image | `docker images projet-cicd-nginx:local` | ~40-50 Mo (Alpine) |
| Syntaxe Compose | `docker compose config` | Pas d'erreur |
| Contenu HTML | `curl -s http://127.0.0.1:8080/ \| grep "Catal-Log"` | Chaîne trouvée |

## Limites de la simulation

La simulation locale avec Docker Compose présente plusieurs limites par rapport à un environnement de production :

- **Absence de load balancer** : lors du scaling, les requêtes ne sont pas réparties entre les instances. Il faudrait un reverse proxy (Nginx, Traefik, HAProxy) devant les conteneurs pour distribuer le trafic.
- **Pas de haute disponibilité** : tous les conteneurs tournent sur une seule machine. Si l'hôte tombe, tout le service est indisponible.
- **Pas de supervision** : aucun outil de monitoring (Prometheus, Grafana) ne surveille l'état des conteneurs ni ne génère d'alertes.
- **Dépendance à l'environnement local** : les tests dépendent de la machine du développeur (version Docker, ressources CPU/RAM, réseau). Les résultats peuvent varier d'un poste à l'autre.
- **Pas de persistance de données** : pour un site statique ce n'est pas un problème, mais en production avec des bases de données, il faudrait gérer les volumes et les sauvegardes.
- **Pas de gestion de secrets** : les variables d'environnement sont en clair dans le compose.yml local, alors qu'en production elles devraient être injectées depuis un coffre de secrets.
