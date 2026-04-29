# 06 - Preuve promotion production-simulee

## Promotion

- **Workflow concerné** : `03-promote.yml` (job `promote-production-simulee`)
- **Environnement GitHub** : `production-simulee`
- **Tag source** : `latest` (ou `sha-23a01d2`)
- **Tag cible** : `production-simulee`
- **Lien du run** : https://github.com/candidat-12-ec06/projet-cicd/actions (voir le run du workflow "03 - Promotion recette vers production-simulee")

## Point essentiel

La promotion réutilise une image existante. Elle ne reconstruit pas l'image. C'est un principe fondamental d'une chaîne CI/CD fiable : l'artefact qui arrive en production est exactement celui qui a été testé, octet pour octet.

Le job `promote-production-simulee` exécute ces trois commandes :
```bash
docker pull ghcr.io/candidat-12-ec06/projet-cicd:latest              # Télécharge l'image existante
docker tag  ghcr.io/candidat-12-ec06/projet-cicd:latest \
            ghcr.io/candidat-12-ec06/projet-cicd:production-simulee   # Ajoute un nouveau tag
docker push ghcr.io/candidat-12-ec06/projet-cicd:production-simulee   # Pousse le tag, pas une nouvelle image
```

Il n'y a aucun `docker build` dans ce job. L'image n'est pas reconstruite.

## Preuve que la promotion s'est faite sans rebuild

Les éléments suivants prouvent l'absence de rebuild :

1. **Le code du workflow** : le job `promote-production-simulee` ne contient aucune instruction `docker build` ni aucun `checkout` du code source. Il travaille uniquement avec des images déjà publiées sur GHCR.

2. **Le digest identique** : le digest de l'image `production-simulee` est le même que celui de l'image source. On peut le vérifier avec :
   ```bash
   docker inspect ghcr.io/candidat-12-ec06/projet-cicd:latest --format='{{index .RepoDigests 0}}'
   docker inspect ghcr.io/candidat-12-ec06/projet-cicd:production-simulee --format='{{index .RepoDigests 0}}'
   ```
   Les deux commandes retournent le même digest SHA256.

3. **Le résumé du workflow** (`$GITHUB_STEP_SUMMARY`) affiche explicitement « Mode : promotion sans rebuild ».

4. **Les logs GitHub Actions** : le step « Promouvoir le même artefact » montre uniquement des opérations `pull`, `tag` et `push`, sans aucune phase de build.
