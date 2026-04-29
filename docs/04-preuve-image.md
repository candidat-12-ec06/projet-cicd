# 04 - Preuve image GHCR

## Image publiée

- **Nom de l'image** : `ghcr.io/candidat-12-ec06/projet-cicd`
- **Tag principal** : `sha-23a01d2` et `latest`
- **Digest** : visible dans le résumé (GITHUB_STEP_SUMMARY) du workflow `02-publish-ghcr.yml` — format `sha256:...`
- **Lien GHCR** : https://github.com/candidat-12-ec06/projet-cicd/pkgs/container/projet-cicd

## Workflow de publication

L'image est publiée automatiquement par le workflow `02-publish-ghcr.yml` à chaque push sur la branche `main`. Le workflow :
1. Se connecte à GHCR via le `GITHUB_TOKEN` ;
2. Génère les métadonnées (tags et labels OCI) avec `docker/metadata-action` ;
3. Construit et pousse l'image avec `docker/build-push-action` ;
4. Affiche un résumé dans le `$GITHUB_STEP_SUMMARY` avec les tags et le digest.

## Explication : tag et digest

Le **tag** est une étiquette lisible associée à une image (par exemple `latest`, `sha-23a01d2`, `production-simulee`). Il est mutable : on peut re-taguer une image ou déplacer un tag vers une autre version. Le tag `latest` par exemple est mis à jour à chaque publication.

Le **digest** est un hash SHA256 calculé sur le contenu exact de l'image. Il est immuable : si un seul octet change, le digest change. C'est l'identifiant unique et fiable de l'artefact.

Ces deux mécanismes sont complémentaires pour la traçabilité et le rollback :
- Le **tag** permet de nommer et retrouver facilement une version (on promeut `sha-23a01d2` en production) ;
- Le **digest** garantit que l'artefact promu est exactement celui qui a été testé, sans altération possible ;
- En cas de problème, on peut revenir à une version précédente en re-promouvant une image identifiée par son digest, avec la certitude que c'est le même binaire.
