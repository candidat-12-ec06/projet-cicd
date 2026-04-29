# 08 - Compte rendu final

## 1. Synthèse

Ce projet met en place une chaîne CI/CD complète pour l'entreprise fictive Catal-Log. Un site web statique est conteneurisé dans une image Docker Nginx, construite et testée automatiquement à chaque push via GitHub Actions, publiée dans GitHub Container Registry avec des tags traçables, validée en recette simulée puis promue en production simulée sans reconstruction de l'image. L'ensemble repose sur des runners hébergés par GitHub, complété par une validation locale sur une machine virtuelle Linux (Debian 12).

## 2. Fonctionnement technique

Le chemin complet d'un changement, du commit à la production simulée :

1. **Commit et push** : le développeur modifie le code (site, Dockerfile, docs) et pousse sur la branche `main`.
2. **Build et tests (01-ci.yml)** : le workflow se déclenche automatiquement. Il vérifie la présence des fichiers obligatoires, valide la syntaxe Docker Compose, construit l'image Docker avec le SHA du commit comme tag, démarre un conteneur et exécute des requêtes HTTP pour vérifier que le site répond correctement sur `/` et `/version.json`. Le contenu HTML est également vérifié par un `grep`.
3. **Publication GHCR (02-publish-ghcr.yml)** : sur la branche `main`, le workflow construit l'image et la publie dans GHCR. Deux tags sont appliqués : `sha-23a01d2` pour la traçabilité exacte et `latest` comme tag glissant. Le digest SHA256 est affiché dans le résumé du workflow.
4. **Validation recette (03-promote.yml, job 1)** : le workflow est déclenché manuellement via `workflow_dispatch` avec le tag source en paramètre. L'image est téléchargée depuis GHCR (pas de rebuild), démarrée et testée par des requêtes HTTP dans l'environnement GitHub `recette`.
5. **Promotion production-simulee (03-promote.yml, job 2)** : si la recette est validée, l'image existante est re-taguée `production-simulee` et poussée sur GHCR. Aucun rebuild n'a lieu : c'est le même artefact binaire, identifiable par son digest.

## 3. Conteneurisation (C12)

### Dockerfile

L'image est basée sur `nginx:1.27-alpine`, choisie pour sa légèreté (~43 Mo) et sa surface d'attaque réduite. Le Dockerfile :
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
  CMD wget -q -O - [http://127.0.0.1/](http://127.0.0.1/) >/dev/null || exit 1
Exécution conteneurisée L'image a été construite et testée avec succès localement sur une machine virtuelle sous Debian 12 (Bookworm) avec Docker v29.4.1 :Bashdocker build -t projet-cicd-nginx:local .
docker run -d --rm -p 8080:80 projet-cicd-nginx:local
Le site est alors accessible sur http://127.0.0.1:8080/. La page affiche le pipeline CI/CD complet avec les cartes techniques (conteneurisation, automatisation, sécurité, orchestration) et charge dynamiquement les informations de version.json.4. Orchestration et scaling (C13)compose.ymlLe fichier compose.yml décrit deux services coordonnés :web : conteneur Nginx servant le site, avec healthcheck et réseau dédié ;tester : conteneur léger (curlimages/curl) qui attend la bonne santé du service web puis exécute des tests HTTP de validation.Les deux services communiquent via un réseau Bridge isolé (cicd_net).Simulation de scaling (Test réel)Afin d'éprouver l'architecture, un test de mise à l'échelle a été réalisé sur la VM Debian :Bashdocker compose up -d --scale web=2
Résultat observé : Le lancement de la deuxième instance a échoué avec l'erreur suivante :Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint projet-cicd-web-2 (...): Bind for 0.0.0.0:8080 failed: port is already allocatedAnalyse : Ce comportement est normal et démontre parfaitement une limite de l'architecture locale. Le port 8080 de la machine hôte étant explicitement mappé en "dur" (8080:80) pour la première instance, il n'est plus disponible pour la seconde. Pour qu'un tel scaling fonctionne, il faudrait retirer le mappage de port fixe ou placer un Load Balancer (répartiteur de charge) devant les conteneurs.LimitesDocker Compose est un outil d'orchestration locale, adapté au développement. Il ne répond pas aux exigences d'une production réelle :Besoin productionDocker ComposeKubernetesHaute disponibilité❌ Un seul hôte✅ Multi-nœuds, auto-healingRépartition de charge❌ Pas de load balancer intégré✅ Service + IngressRolling update❌ Arrêt puis redémarrage✅ Déploiement progressifAuto-scaling❌ Manuel (--scale)✅ HPA basé sur les métriquesGestion des secrets❌ Variables en clair✅ Secrets Kubernetes chiffrés5. Automatisation et sécurité (C14)Workflows GitHub ActionsWorkflowDéclencheurRôle01-ci.ymlPush, PR, manuelBuild, tests automatisés, validation02-publish-ghcr.ymlPush sur main, manuelPublication GHCR avec tags et digest03-promote.ymlManuel (workflow_dispatch)Validation recette + promotion sans rebuildGITHUB_TOKEN et PermissionsLe GITHUB_TOKEN est un token éphémère. Chaque workflow déclare uniquement les permissions nécessaires (principe de moindre privilège) :01-ci.yml : contents: read (lecture seule du code)02-publish-ghcr.yml et 03-promote.yml : contents: read + packages: write (publication/promotion GHCR)Aucun secret n'est stocké dans le dépôt.6. Production réelleGestion des secretsEn production réelle, les secrets (tokens, clés API) ne doivent jamais figurer dans le code source. Ils doivent être gérés via des GitHub Secrets pour les pipelines, ou un coffre de secrets (ex: HashiCorp Vault) avec des variables d'environnement injectées au runtime.RollbackLe rollback s'appuie sur la traçabilité de la chaîne :Identifier la version précédente via son tag (sha-<commit>) ou son digest.Relancer le workflow 03-promote.yml avec le tag de la version à restaurer.L'image précédente est re-taguée production-simulee et poussée — sans rebuild, le digest garantit une copie bit-à-bit parfaite.7. ## 7. Preuve
| Preuve | Lien |
|--------|------|
| Dépôt GitHub | https://github.com/candidat-12-ec06/projet-cicd |
| Run CI | https://github.com/candidat-12-ec06/projet-cicd/actions/runs/xxxxxxx |
| Run Publication | https://github.com/candidat-12-ec06/projet-cicd/actions/runs/xxxxxxx |
| Image GHCR | https://github.com/candidat-12-ec06/projet-cicd/pkgs/container/projet-cicd |
| Run Promotion | https://github.com/candidat-12-ec06/projet-cicd/actions/runs/xxxxxxx |
| Tests Locaux | Réalisés et validés sur VM Debian 12 (voir docs/03-fiche-tests.md) |.8. Difficultés et apprentissagesConfiguration de l'environnement Linux : L'installation de Docker sur Debian 12 a nécessité la configuration manuelle des dépôts officiels APT et le nettoyage des anciens paquets, ce qui m'a permis de mieux comprendre la gestion des sources sous Linux.Conflits réseau Docker (Scaling) : Le test de mise à l'échelle m'a confronté concrètement à l'erreur port is already allocated. Cela illustre parfaitement pourquoi l'orchestration avancée (Kubernetes) ou les répartiteurs de charge sont indispensables en production pour gérer dynamiquement l'exposition des ports.Promotion sans rebuild : Comprendre pourquoi on ne reconstruit pas l'image lors de la promotion a été un point clé. Reconstruire introduirait un risque de divergence. La promotion par re-tag garantit l'identité binaire de l'artefact testé.Différence tag vs digest : Le tag est une étiquette humaine (mutable), le digest est un identifiant cryptographique (immuable). Les deux sont complémentaires dans une chaîne CI/CD robuste.
