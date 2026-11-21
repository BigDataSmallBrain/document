<div style="text-align: center;">

  # <u>SmallBrain - Documents</u>

</div>

Repository des documents du projet SmallBrain : un outil de Retrieval basé sur les README de GitHub, avec une pipeline MLOps automatisée (collecte, entrainement, évaluation, mise à disposition via un site Web).

---
# Documents
- <u>**resume.md :**</u> Le document principal du projet : objectif, architecture, sécurité, base de données, scrapping, entrainement, interface utilisateur, CI/CD, modèles, métriques, difficultés et points à améliorer.
- <u>**Architecture.md :**</u> Organisation des repositories, branches, et arborescence des fichiers de chaque système (raspberry, cluster, serveur d'API, AWS, ordinateur de développement).
- <u>**Database.txt :**</u> Description détaillée des tables PostgreSQL, générée sur le raspberry avec `inspect_db.sh`.
- <u>**guide_des_configurations_de_mod_les_json.md :**</u> Format des fichiers de configuration des modèles (inférence, fine-tuning, Test et Load).

---
# Repositories du projet
| Repository | Système | Rôle |
| :--- | :--- | :--- |
| `site` | Raspberry Pi | Site Web : recherche, base de données, monitoring, système, administration. |
| `Collector` | Raspberry Pi | Collecte des repos GitHub. |
| `Raspberry` | Raspberry Pi | Configuration du raspberry : export, import, sauvegarde, déploiement. |
| `Cluster` | Cluster universitaire | Embeddings, entrainements et évaluation des modèles. |
| `Server` | Serveur tiers | API de Retrieval (Docker). |
| `document` | — | Documentation du projet. |

Lien de l'organisation : https://github.com/BigDataSmallBrain
