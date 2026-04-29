# 05 - Preuve recette simulée

## Workflow de validation recette

- **Workflow concerné** : `03-promote.yml` (job `validate-recette`)
- **Environnement GitHub** : `recette`
- **Tag source validé** : `latest` (ou `sha-23a01d2`)
- **Digest observé** : identique à celui de la publication — visible dans le résumé du run
- **Lien du run** : https://github.com/candidat-12-ec06/projet-cicd/actions (voir le run du workflow "03 - Promotion recette vers production-simulee")

## Résultat

Le workflow de promotion est déclenché manuellement via `workflow_dispatch`. Le paramètre `source_tag` indique quel tag GHCR doit être validé (par défaut `latest`).

Lors de la phase de recette simulée, le job `validate-recette` exécute les opérations suivantes :
1. **Pull de l'image** depuis GHCR (sans rebuild) : `docker pull ghcr.io/candidat-12-ec06/projet-cicd:latest`
2. **Démarrage du conteneur** en mode détaché sur le port 8080
3. **Tests HTTP** : requêtes `curl` sur `/` et `/version.json` pour vérifier que le site répond correctement
4. **Affichage du digest** dans le résumé du workflow pour traçabilité

Le test HTTP confirme que :
- La page d'accueil est servie correctement (code HTTP 200, contenu HTML valide)
- Le fichier `version.json` est accessible et retourne les informations de version attendues

L'environnement GitHub `recette` permet de simuler un gate de validation : en production réelle, cet environnement pourrait nécessiter une approbation manuelle, des reviewers désignés ou des règles de protection spécifiques.

## Preuve

Le run GitHub Actions montre :
- Le job `validate-recette` au statut ✅
- Le résumé (`GITHUB_STEP_SUMMARY`) affichant l'image source et le digest observé
- Les logs du step "Tester l'image en recette simulée" montrent les réponses HTTP 200 sur les deux endpoints
