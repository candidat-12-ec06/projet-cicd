01 - Cadrage du projet
Identité
Nom et prénom : 12

Dépôt GitHub : https://github.com/candidat-12-ec06/projet-cicd

Date de démarrage : 27 avril 2025

Objectif
Mettre en place une chaîne CI/CD permettant de construire, tester, publier et promouvoir une image Docker Nginx contenant un site web statique pour le scénario Catal-Log. L'ensemble de la chaîne repose sur GitHub Actions et GitHub Container Registry (GHCR), complété par des tests de validation sur un environnement local Linux.

Contraintes du projet
Travail individuel.

Aucune infrastructure fournie par le formateur.

Utilisation de GitHub Actions pour l'automatisation.

Validation obligatoire sur un environnement Docker local.

Choix personnels
Dépôt : Le dépôt est public pour faciliter la vérification. Le nommage suit la convention projet-cicd.

Stratégie de tags : Utilisation de tags sha-, latest pour le suivi, et production-simulee pour la promotion manuelle (sans nouveau build).

Environnement local : Docker (v29.4.1) et Docker Compose (v5.1.3) sont utilisés pour valider le fonctionnement avant chaque push. Les tests docker compose up --build ont été réalisés avec succès. Le site est accessible sur http://localhost:8080 et les deux endpoints (/ et /version.json) répondent correctement.

VM personnelle : Pour ce projet, j'ai choisi d'utiliser une machine virtuelle sous Debian 12 (Bookworm). Ce choix permet de s'isoler de la machine hôte et de travailler sur un environnement Linux standardisé, stable et représentatif des serveurs de production réels. L'installation de Docker a été effectuée via les dépôts officiels Docker pour Debian.

Branche de travail : Utilisation de la branche main. Pour un projet d'entreprise, une stratégie de branches (GitFlow) serait privilégiée.
