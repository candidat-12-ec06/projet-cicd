# 07 - Sécurité minimale

## Permissions GitHub Actions

Chaque workflow déclare uniquement les permissions dont il a besoin, conformément au principe du moindre privilège :

| Workflow | Permissions | Justification |
|----------|------------|---------------|
| `01-ci.yml` | `contents: read` | Le workflow a seulement besoin de lire le code source pour construire et tester. Il ne publie rien. |
| `02-publish-ghcr.yml` | `contents: read`, `packages: write` | Lecture du code source pour le build, et écriture dans GHCR pour publier l'image Docker. |
| `03-promote.yml` | `contents: read`, `packages: write` | Lecture pour accéder au dépôt, écriture dans GHCR pour re-taguer et pousser l'image promue. |

Aucun workflow ne demande `contents: write`, `actions: write` ou d'autres permissions élargies. Cela limite l'impact en cas de compromission d'un workflow : un attaquant ne pourrait pas modifier le code du dépôt ni altérer les paramètres du projet.

## Gestion des secrets

### Pourquoi aucun secret ne doit être stocké dans le code

Stocker un secret (mot de passe, token, clé API) dans le code source est une faille critique :
- Le secret est visible dans l'historique Git, même après suppression du fichier ;
- Toute personne ayant accès au dépôt (collaborateurs, forks sur un dépôt public) peut le lire ;
- Des bots scannent en permanence les dépôts publics à la recherche de secrets exposés ;
- Un secret compromis peut permettre un accès non autorisé à des services ou des données.

### Usage du GITHUB_TOKEN

Le `GITHUB_TOKEN` est un token éphémère généré automatiquement par GitHub Actions pour chaque exécution de workflow. Ses caractéristiques :
- Il est créé au début du run et révoqué à la fin ;
- Ses permissions sont limitées à celles déclarées dans le bloc `permissions` du workflow ;
- Il n'est jamais stocké dans le code ni dans les secrets du dépôt ;
- Il permet de s'authentifier auprès de GHCR pour publier les images Docker.

Dans ce projet, le `GITHUB_TOKEN` est utilisé dans les workflows `02-publish-ghcr.yml` et `03-promote.yml` pour se connecter à GHCR via `docker/login-action`.

### Éléments à placer dans GitHub Secrets en production réelle

En production, les éléments suivants devraient être gérés comme des secrets :
- **Tokens d'accès à des registres externes** (Docker Hub, AWS ECR, Azure ACR) si le GHCR ne suffit pas ;
- **Clés SSH ou tokens de déploiement** pour pousser sur des serveurs distants ;
- **Clés API de services tiers** (monitoring, notification Slack, scan de vulnérabilités) ;
- **Credentials de base de données** si des migrations sont exécutées dans le pipeline.

Pour un projet d'envergure, un coffre de secrets (HashiCorp Vault, AWS Secrets Manager) remplacerait GitHub Secrets, avec rotation automatique des credentials et audit des accès.

## Rollback

Le rollback consiste à revenir à une version précédente en cas de problème après une promotion.

**Méthode par tag** : si la version `sha-a1b2c3d` est en production et qu'un problème est détecté, on peut promouvoir la version précédente (`sha-9f8e7d6`) en relançant le workflow `03-promote.yml` avec le tag source correspondant. L'ancienne image est re-taguée `production-simulee` et poussée sur GHCR.

**Méthode par digest** : le digest SHA256 identifie de manière immuable chaque image. Même si un tag a été déplacé, le digest permet de retrouver et re-déployer exactement l'artefact souhaité :
```bash
docker pull ghcr.io/<owner>/projet-cicd@sha256:<digest_précédent>
docker tag ghcr.io/<owner>/projet-cicd@sha256:<digest_précédent> ghcr.io/<owner>/projet-cicd:production-simulee
docker push ghcr.io/<owner>/projet-cicd:production-simulee
```

**Avantage de la promotion sans rebuild** : puisque la promotion ne reconstruit jamais l'image, le rollback est instantané et fiable. On ne dépend pas de la disponibilité des dépendances ni de la reproductibilité du build au moment du rollback.

## Sauvegarde / restauration

### Éléments à sauvegarder

| Élément | Méthode de sauvegarde | Fréquence |
|---------|----------------------|-----------|
| **Dépôt GitHub** (code, workflows, docs) | Clone miroir (`git clone --mirror`) vers un emplacement distinct (autre provider Git, stockage objet) | À chaque release ou quotidiennement |
| **Images GHCR** | Politique de rétention des tags (ne pas supprimer les images taguées avec un SHA de commit) ; export `docker save` pour archivage long terme | À chaque publication |
| **Documentation et preuves** | Incluses dans le dépôt Git, donc sauvegardées avec le clone miroir | Avec le dépôt |
| **Configuration des workflows** | Versionnée dans `.github/workflows/`, sauvegardée avec le dépôt | Avec le dépôt |
| **Secrets GitHub** | Documentés dans un coffre de secrets externe (Vault, gestionnaire de mots de passe), jamais dans le code | En continu |
| **Configuration des environnements** | Règles de protection, reviewers : à documenter pour pouvoir les recréer | À chaque modification |

### Restauration

En cas de perte du dépôt :
1. Restaurer le dépôt depuis le clone miroir ;
2. Recréer les secrets dans GitHub Secrets ou le coffre de secrets ;
3. Recréer les environnements GitHub (`recette`, `production-simulee`) avec leurs règles de protection ;
4. Vérifier que les workflows s'exécutent correctement avec un push de test.

En cas de perte d'une image :
1. Reconstruire l'image depuis le commit correspondant (le SHA est tracé dans les tags) ;
2. Republier manuellement sur GHCR ;
3. Re-promouvoir si nécessaire.

## Deux éléments complémentaires

### 1. Contrôle des vulnérabilités

En production réelle, chaque image publiée devrait être scannée automatiquement pour détecter des vulnérabilités connues (CVE). Des outils comme **Trivy**, **Grype** ou **Snyk** peuvent être intégrés directement dans le pipeline CI :

```yaml
- name: Scanner l'image avec Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: 1  # Bloque le pipeline si une CVE critique est trouvée
```

Ce scan bloquerait la publication d'une image contenant des dépendances vulnérables. L'image Alpine utilisée dans ce projet réduit déjà la surface d'attaque par rapport à une image Debian ou Ubuntu.

### 2. Validation manuelle avant production

Les environnements GitHub (`recette`, `production-simulee`) peuvent être configurés avec des **règles de protection** :
- **Reviewers requis** : un ou plusieurs membres de l'équipe doivent approuver le déploiement avant qu'il ne s'exécute ;
- **Timer d'attente** : un délai configurable entre l'approbation et l'exécution, permettant d'annuler en cas d'erreur ;
- **Restriction de branches** : seules les images provenant de certaines branches peuvent être promues.

Dans ce projet pédagogique, les environnements sont créés dans GitHub mais sans reviewers obligatoires. En production réelle, cette validation manuelle serait un verrou de sécurité essentiel pour éviter qu'un artefact non validé n'atteigne la production.
