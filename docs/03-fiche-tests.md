# 03 - Fiche tests

## Test automatisé GitHub Actions

- **Workflow concerné** : `01-ci.yml`
- **Lien vers le run réussi** : https://github.com/candidat-12-ec06/projet-cicd/actions (voir le dernier run du workflow "01 - CI - Build et test")
- **Ce qui est testé** :
  - Présence des fichiers obligatoires (`Dockerfile`, `compose.yml`, `site/index.html`, `site/version.json`, `docs/08-compte-rendu-final.md`)
  - Validité de la syntaxe Docker Compose (`docker compose config`)
  - Construction de l'image Docker (`docker build`)
  - Démarrage du conteneur et réponse HTTP sur `/` et `/version.json`
  - Vérification du contenu de la page (grep sur "Projet CICD")
- **Résultat** : tous les steps passent au vert (✅), le conteneur répond sur les deux endpoints et le contenu HTML contient bien "Projet CICD"

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

Résultat observé : la page HTML du site Catal-Log s'affiche correctement avec le titre, le pipeline et les cartes techniques. Le fichier `version.json` retourne les métadonnées du projet (version 1.0.0, auteur 12, contexte EC06). Le service tester valide automatiquement les deux endpoints et affiche "Tous les tests OK" avant de s'arrêter.

## Simulation de scaling

Si l'environnement le permet :

```bash
docker compose up -d --scale web=2
docker compose ps
```

Résultat observé : Le lancement de la deuxième instance a échoué avec l'erreur Bind for 0.0.0.0:8080 failed: port is already allocated.

Analyse : Ce résultat est cohérent avec la configuration actuelle car le port 8080 est mappé en "dur" sur l'hôte. Pour valider le scaling, il faudrait modifier le compose.yml pour supprimer le mappage de port ou utiliser un reverse-proxy. Cela confirme la limite technique identifiée dans la section suivante.

## Tests complémentaires

En complément des tests de base, les vérifications suivantes ont été réalisées :

| Test | Commande / méthode | Résultat attendu |
|------|-------------------|------------------|
| Healthcheck Docker | `docker inspect --format='{{.State.Health.Status}}' <container>` | `healthy` |
| Taille de l'image | `docker images projet-cicd-nginx:local` | 73.7MB (Alpine) |
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
