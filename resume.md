
<div style="text-align: center;">

  # <u>BigData - SmallBrain </u>
  
</div>

# Projet
### <u>Groupe</u>
- Aymeric MABIRE
---
### <u>Objectif</u>
L'objectif de concevoir un outils de Retrival basé sur les ReadMe de GitHub. Son interface sera exposé sur un Site Web hebergé, et il sera possible de pouvoir choisir parmis les 3 meilleurs modèles de l'entrainement.

---
### <u>Contraintes</u>
- Le scrapping doit être effecuté de manière régulière pour conserver la base de donnée à jour.
- Les entrainements doivent être effectué de manières automatiques.
- Plusieurs modèles doivent être mis en place (MLOps).

### <u>Git</u>
- https://github.com/BigDataSmallBrain
---
# Architecture

```mermaid
flowchart LR
    Cluster["Cluster<br/>(Entraînement & Embeddings)"]
    AWS["AWS<br/>(Bridge)"]
    RPI["Raspberry Pi<br/>(Hébergeur & Base de données)"]
    API["Serveur<br/>(API pour Retrieval)"]
    Web["Site Web"]

    Cluster -->|X| AWS
    AWS -->|X| RPI
    RPI -->|Fast-API| API
    RPI -->|Tailscale Funnel| Web
    AWS -->|WireGuard| API
```
---

### <u>Architecture API:</u> [En attente du serveur]

**Système bi-conteneur:**
- API (`api_retrieval`)
- - Port public 35000
- - 8 coeurs, 32Go RAM, GPU (Intel)
- - Accès en lecture seule au dossier des modèles, et fonctionnement entièrement hors ligne.
- - Ne tourne pas en root : elle accède au GPU via le groupe du serveur (`RENDER_GID`).

- SSH (`ssh_models_receiver`)
- - Conteneur fait par nos soins (Alpine, openssh, rsync) : l'image `dockette/ssh` prévue au départ n'était qu'un client SSH.
- - Connexion uniquement par clé, sans mot de passe ni root.
- - Aucun tunnel ni terminal : il ne sert qu'au dépôt des modèles via `rsync`.
- - Publié uniquement sur l'adresse WireGuard du serveur, jamais sur Internet.

Dossier en commun entre API et SSH pour le dépôt des modèles.<br>
Les deux images sont construites par GitHub Actions et publiées sur le GitHub Container Registry. Le serveur les récupère via Portainer.

Il ne reste plus que la validation de Mewen pour le déploiement, ainsi que la route de recherche (`/search`) de l'API.
---
### <u> Sécurité </u>
Une attention particulière à été prêter à la sécurité de cette architecture en respectant au mieux la règle suivante: <br>
"C'est le système sécurisé qui se connecte au système vulnérable et non l'inverse." <br>
A ces fins, le cluster se connecte via SFTP sur le raspberry pi en utilisant le serveur AWS (ouverture du port 22 avec filtre sur le AWS avec controle des IPs pour ne laisser passer que le cluster, puis tailscale entre AWS et le pi).
Le cluster sait donc se connecter successivement a l'AWS et au raspberry avec des connexions ssh, mais ni AWS ni le raspberry ne sont en mesure de se connecter au cluster.

Lors de cette connexion SFTP, le cluster ne peut lire que dans `input` et déposer ses metrics et poids dans `output`, de ce fait, il ne peut pas acceder au reste des données du raspberry, ce qui rajoute un controle supplémentaire sur les accès.

Un transfert des poids était également necessaire entre le Cluster et le Serveur API. Il respecte la même règle, et passe lui aussi par le serveur AWS:
- AWS se connecte au serveur via **WireGuard**. Le tunnel ne laisse passer que l'adresse du serveur, et le serveur ne peut pas ouvrir de connexion vers AWS.
- Le cluster ne passe pas directement par WireGuard : il utilise le même utilisateur d'AWS que pour le raspberry (`jump`), dont la clé n'autorise que des redirections vers le raspberry et vers le conteneur SSH du serveur.

Ainsi : Cluster > SSH > AWS > WireGuard > Conteneur SSH > Dossier des modèles.

Enfin pour l'utilisation de l'API [Inserer ici comment cela à été réaliser].

Le site Web, hébergé sur le raspberry pi, est quant à lui exposé sur Internet via **Tailscale Funnel**. Ainsi, aucun port n'est ouvert sur ma box personnelle et mon adresse IP n'est pas exposée, Tailscale se chargeant également du certificat HTTPS.<br>
Le site n'écoute que localement (`127.0.0.1:8000`) et tourne en tant que service systemd durci (`smallbrain.service`) : il ne peut écrire que dans le dossier des demandes (`data/requests`), et redémarre automatiquement en cas d'erreur ou de redémarrage du raspberry.<br>
Il limite chaque visiteur à 120 requêtes par minute (30 pour la Recherche et les README), et aucun script n'est écrit dans ses pages : la Content-Security-Policy interdit tout JavaScript intégré. La connexion à l'Admin bloque une adresse après 5 échecs, pour une durée qui double à chaque récidive (jusqu'à 24 h). Les mots de passe sont hachés avec PBKDF2 (600 000 itérations), et une session déconnectée est révoquée. La page de requêtes SQL est limitée à 10 secondes par requête, 10 requêtes par minute et par visiteur, 2 requêtes en même temps et 256 Mo de fichiers temporaires.

Le déploiement continu du raspberry tourne en root : pour qu'un service compromis ne puisse pas s'en servir pour devenir root, il ne lance qu'une copie du script appartenant à root, et n'installe que des fichiers lus dans les objets git du commit dont GitHub confirme qu'il est la tête de `main` avec une CI verte.

Les README affichés sur le site proviennent de repositories quelconques : ils sont donc nettoyés avant affichage, et tout ce qui pourrait exécuter du code est retiré (scripts, liens `javascript:`...).

Les demandes de réentraînement déposées depuis le site respectent la même règle : c'est le cluster qui vient les récupérer, et il les considère comme non fiables. Il ne garde que ses propres valeurs pour le modèle (`model_name`, `revision`, `embedding_dim`, `trust_remote_code`), n'accepte que certains paramètres avec des bornes, et vérifie chaque chemin de sauvegarde. Ainsi, même si le site venait à être compromis, il ne pourrait pas faire exécuter n'importe quoi au cluster. De même, le manifest de l'export est vérifié avant que ses noms de fichiers servent dans un chemin ou une commande SFTP, et aucune commande SFTP ne peut contenir de saut de ligne : même un raspberry compromis ne pourrait pas lancer de commande sur le cluster.

Pour vérifier l'état de la CI, le raspberry utilise un jeton GitHub en lecture seule, limité à la consultation des GitHub Actions. Il est stocké dans un fichier uniquement lisible par root (`/etc/smallbrain-deploy.env`).<br>
Seuls les commits de `main` dont la CI est validée sont déployés.

---
# Base de données
Nous avons une base de données PSQL `repos` sur le raspberry pour la conservation de nos données, dont les principales tables sont:
- Repos : Contient les repos alimenter par notre scrapping
- model_evaluations : Contient les métriques hébdomadaire par runs et par models.
- retrain_requests : Contient l'historique des demandes de réentrainement effectuées depuis le site.

Ces bases de données sont alors sauvegardé tout les samedis à 0h00 (`backup.timer`). <br>
Seuls les 3 dernières sauvegardes sont conservés, ce qui fait un historique de 3 semaines.

## Repos
| Column | Type | Nullable | Default | Storage |
| :--- | :--- | :--- | :--- | :--- |
| id | bigint | not null | `nextval('repos_id_seq'::regclass)` | plain |
| name | text | not null | | extended |
| url | text | | | extended |
| description | text | | | extended |
| content_language | text | | | extended |
| readme_compressed | bytea | | | extended |
| readme_length | integer | | | plain |
| stars | integer | | | plain |
| forks | integer | | | plain |
| primary_language | text | | | extended |
| license | text | | | extended |
| topics | text | | | extended |
| created_at | text | | | extended |
| pushed_at | text | | | extended |
| archived | integer | | | plain |
| last_fetched_at | text | | | extended |

## Model Evaluations
Conserve l'historique hebdomadaire des évaluations de recherche par modèle et par run.<br>
- Mode Append-Only : Il est uniquement possible de rajouter des données, impossible de les modifier ou les ecrasers.
- Unicité : Les contraintes d'unicités empêche l'insertion des doublons dans la base.
- Entrée des donneés : L'ingestion est déclenchée dès que le cluster dépose un fichier dans `inbox`, via `importer.path` exécutant le service `importer.service` pour alimenter la base de données. Les fichiers sont alors traités puis déplacés dans le dossier `processed`.<br>
Un passage de secours est également effectué tous les jours à 12h00 via le timer `importer.timer`, afin de retraiter les fichiers qui n'auraient pas pu l'être (base indisponible par exemple).<br>
<br>

| Colonne | Type PostgreSQL | Contraintes | Description |
| :--- | :--- | :--- | :--- |
| `id` | `BIGSERIAL` | `PRIMARY KEY` | Identifiant des metriques |
| `run_id` | `TEXT` | `NOT NULL UNIQUE` | Identifiant unique du run (ex. `2026-W46-gemma`) |
| `source_file` | `TEXT` | `NOT NULL UNIQUE` | Nom du fichier JSON de metriques traités |
| `run_date` | `TIMESTAMPTZ` | `NOT NULL` | Horodatage de fin d'exécution (`finished_at`, sinon `started_at`) |
| `embedding_dim` | `INTEGER` | | Dimension des vecteurs générés |
| `validation_set_version` | `TEXT` | | Version du jeu de validation utilisé |
| `corpus` | `TEXT` | | Périmètre du corpus évalué |
| `n_queries_evaluated` | `INTEGER` | | Nombre de requêtes évaluées sur le jeu de test |
| `duration_seconds` | `DOUBLE PRECISION` | | Durée totale d'exécution/entraînement sur le cluster (en secondes) |
| `bm25_recall_at_10` | `DOUBLE PRECISION` | | Score Recall@10 - recherche lexicale BM25 |
| `bm25_mrr` | `DOUBLE PRECISION` | | Score MRR - recherche lexicale BM25 |
| `bm25_ndcg_at_10` | `DOUBLE PRECISION` | | Score nDCG@10 - recherche lexicale BM25 |
| `e5_recall_at_10` | `DOUBLE PRECISION` | | Score Recall@10 - recherche dense (embeddings) |
| `e5_mrr` | `DOUBLE PRECISION` | | Score MRR - recherche dense (embeddings) |
| `e5_ndcg_at_10` | `DOUBLE PRECISION` | | Score nDCG@10 - recherche dense (embeddings) |
| `rrf_recall_at_10` | `DOUBLE PRECISION` | | Score Recall@10 - fusion hybride (*Reciprocal Rank Fusion*) |
| `rrf_mrr` | `DOUBLE PRECISION` | | Score MRR - fusion hybride (*Reciprocal Rank Fusion*) |
| `rrf_ndcg_at_10` | `DOUBLE PRECISION` | | Score nDCG@10 - fusion hybride (*Reciprocal Rank Fusion*) |
| `details` | `JSONB` | `NOT NULL DEFAULT '{}'` | Métriques détaillés : scores par tranche (`by_size`), intervalles de confiance (`ci95`) et tests appariés (`paired_diff`) |
| `model_name` | `TEXT` | `NOT NULL` | Nom HuggingFace du modèle évalué |
| `config` | `JSONB` | | Configuration exacte utilisée pour le run (préfixes, révision, bloc `training`, point de départ...) |

<br>

## Retrain Requests
Conserve l'historique des demandes de réentraînement déposées depuis le site et prises en charge par le cluster.<br>
- Mode Append-Only : Comme pour les autres tables, l'importeur ne peut qu'ajouter des lignes.
- Entrée des données : Lorsque le cluster prend en charge une demande, il dépose un accusé `request_<id>.json` dans `inbox`, qui est directement enregistré par l'importeur. Le site archive alors la demande et affiche l'état de chaque run.

| Colonne | Type PostgreSQL | Contraintes | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | `PRIMARY KEY` | Identifiant de la demande (ex. `20261121T143000Z-retrain`) |
| `status` | `TEXT` | `NOT NULL` | `submitted` si au moins un run a été lancé, sinon `rejected` |
| `processed_at` | `TIMESTAMPTZ` | `NOT NULL` | Date de la prise en charge par le cluster |
| `imported_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT now()` | Date de l'import sur le raspberry |
| `raw` | `JSONB` | `NOT NULL` | Accusé complet : runs lancés (`run_id`, job Slurm), erreurs et avertissements |

<br>

## Repo Chunks
Contient les chunks des README et leurs embeddings, utilisés pour l'inférence.<br>
Chaque run hebdomadaire dépose un index complet et neuf : la version des chunks contient la semaine de l'export (ex. `e5-small-614241f6-c1-2026-W47`). Un README modifié est donc pris en compte la semaine suivante, sans que l'importeur n'ait à modifier quoi que ce soit (il reste en Append-Only).

Les anciens index sont supprimés chaque samedi à 5h00 (`purge_chunks.py`) : seules les 3 dernières semaines importées avec succès sont conservées pour chaque modèle, comme les sauvegardes des modèles. Tant qu'aucune semaine n'a été importée, rien n'est supprimé.

<br>

La table `pipeline_runs` reçoit également une colonne `config` (`JSONB`), afin de pouvoir consulter depuis le site la configuration utilisée par chaque run, réussi ou non.


---
# Scrapping
Nous utilisons l'API de GitHub afin de pouvoir récuperer des repositories correspondants à nos critères:
- README d'au moins 100 caractères
- Licence : mit, apache-2.0, bsd-3-clause, gpl-3.0, unlicense
- Langues : en ou fr
- Aucune limite d'étoiles
- 7 tranches de départ : >50 000, 10 000–50 000, 5 000–9 999, 1 000–4 999, 500–999, 100–499 et 0–99

Et lorsque ces critères sont remplis, nous collectons alors:
- Nom (clé unique)
- URL
- Description
- Langue détectée
- README (compressé en gzip)
- Longueur du README
- Etoiles
- Forks
- Langage principal
- Licence
- Topics (10 maximum)
- Date de création 
- Date de dernier commit
- Statut archivé
- Date de collecte

Les repos déjà présents sont mis à jour lorsqu'ils sont récuperer à nouveau durant le scrapping, jamais dupliqués.

L'objectif lors de cette récupération, est de collecté un maximum de données, faisant alors abstraction du nombres d'étoiles, car cela pourrait avoir son utilité à l'avenir.

Egalement, l'API de GitHub impose un nombre limité de requête par heure, et si ce nombre est dépasser, ou que trop de requête sont envoyés en un laps de temps très court, des avertissements commencent à être émits, pouvant amener à une suspension du compte.<br>
Pour eviter ce problème, il a été mis en place:
- **Un limiteur de débit à fenêtre glissante**, partagé entre tous les threads, qui plafonne le script à 1500 requêtes par minute, soit sous les 2000 points par minute autorisés par GitHub.
- **Une marge de sécurité sur le quota horaire** : à chaque réponse, le script lit le quota restant. Lorsqu'il passe sous un seuil (1600 points), la collecte est suspendue, puis reprend automatiquement après la réinitialisation du quota (`resetAt`) communiquée par GitHub.
- **Une gestion des erreurs avec temporisation** : si GitHub signale tout de même une limite secondaire, le script attend le délai demandé par l'API (en-tête `Retry-After`, sinon heure de reset, sinon pause croissante de 60 s, 120 s, etc.) avant de réessayer. Les erreurs réseau et serveur (502, 503, 504, timeout) sont retentées jusqu'à 5 fois avec une attente doublée à chaque tentative.

Ces différentes mesures lors du scrapping permettent ainsi une collecte efficace, tout en respectant les conditions d'utilisation de GitHub.

---
### <u>Découpage:</u>

L'API de GitHub plafonne chaque recherche à 1000 résultats. Pour contourner cette limite, l'espace de recherche est découpé en segments (tranche d'étoiles × licence × plage de dates). Lorsqu'un segment dépasse 1000 repos, il est scindé en deux sous-segments plus fins, d'abord par étoiles, puis par dates si les étoiles sont indivisibles, et ainsi de suite jusqu'à passer sous le seuil. Les sous-segments sont placés dans une file d'attente de type FILO : le dernier segment créé est le premier traité, ce qui permet de descendre un segment saturé jusqu'à ses feuilles avant d'en ouvrir un autre. La progression de chaque segment est sauvegardée pour pouvoir reprendre la collecte à tout moment.

Lorsqu'un segment est interrompu (quota épuisé, début du week-end, arrêt du service), il est remis au sommet de la pile avec sa position, et c'est donc lui qui est repris en premier.

---
### <u>Récupération:</u>

<u>**Première récupération**</u>

Dans un premier temps, au début du projet afin de se consistuer un DataWarehouse conséquent, nous avons laisser tourner la récuperation pendant 2 semaines, en récuperant des repositories depuis le début de la création de git.

<u>**Récupération lors du projet**</u>

Ensuite, lorsque nous avons collecter une grosse quantités de données (Environs 5 millions de repos), nous modifions le script de récupération, cette fois-ci, il ne récupère que les repos modifiés (push, création) il y a 5 jours ou moins, toujours en ignorant le nombre d'etoiles.

Le collecteur tourne alors sur le raspberry pi en tant que service (`collector.service`), et effectue une passe complète de ces 5 derniers jours toutes les 24h. Les repos encore actifs sont donc revus plusieurs fois, ce qui permet de mettre à jour leurs étoiles et leur README.

La récupération est mise en pause automatiquement du samedi 00h00 au lundi 00h00, car c'est à ce moment que s'effectue la sauvegarde de la base de données et les entrainements.

---
### <u>Détection de la langue:</u>

La détection de la langue faisait parfois échouer la collecte : la librairie utilisée (`langdetect`) pouvait renvoyer une erreur sur certains textes, et faisait alors échouer tout le segment en cours. Elle n'est également pas prévue pour être utilisée par plusieurs threads en même temps.

La détection a donc été reprise sur le modèle du script `tag_language`, déjà utilisé pour re-taguer la base:
- La langue est détectée sur le début du README, nettoyé de ses badges, images, liens et balises HTML (la description n'est utilisée qu'en secours).
- La détection est déterministe, et protégée pour l'utilisation par plusieurs threads.
- En cas d'erreur, la langue est considérée comme inconnue : le repo est simplement ignoré, sans faire échouer le reste du segment.

# Entrainement
### <u>Données </u>
Sélections des repos ayant au moins 1000 étoiles, dont le README est en anglais ou en français (et d'au moins 100 caractères, filtre déjà appliqué par le collecteur).<br>
Ces données sont déposées dans un espace prévu à cet effet tout les samedi à 3h00 (`export.timer`). L'export contient également la distribution exacte de ce corpus, affichée sur la page Distribution du site.

Le dimanche à 0h00, le cluster se connecte sur le Raspberry PI, dans l'espace SFTP, afin de récuperer les données alors déposés pour faire son entrainement.

### <u>Entrainement</u>
L'entrainement est alors inité tout les dimanches à 0h00 via un Cron. Il execute le fichier batch prévu à cet effet afin de pouvoir s'allouer un espace sur la node GPUE-06 avec 1 gpu, 16 cpu et 48G de ram. Le script est prévu de tourner 12h au maximum. L'entrainement est prévu de prendre environ 8h, afin de ne pas monopoliser le cluster, il a été decider de le faire tourner entre 0h00 et 12h.<br>
L'entrainement s'effectue alors avec le même jeu de donnée sur tous les modèles de notre pool models. Une fois l'entrainement effecuté, nous calculons les métriques de nos modèles que nous redéposons sur le raspberry pi.

La base de données n'acceptant que des embeddings de 384 dimensions, seuls les chunks des modèles de cette dimension sont déposés. Les autres modèles passent automatiquement en run d'analyse : seules leurs métriques sont déposées.

### <u>Jeu de validation</u>
Le jeu de validation se compose donc de pair requête-repos avec une information sur la pertinence :
- Pertinent : Le repos corresponds parfaitement à la requête.
- Non Pertinent : Le repos ne corresponds pas à la requête (simple evocation du sujet).
- Doute : Le sujet corresponds partiellement à la requête .

Ces niveaux de pertinences permettent donc de pouvoir determiner quels repos conseillé et estimer les performances de nos modèles.

Les requêtes sont prelevés de requête type qui pourrait être utiliser par un utilisateur.

### <u>Evaluation</u>
Les modèles sont donc evaluer à partir des métriques présenter ci-dessous. Les 3 modèles les plus performants parmi les runs hebdomadaires *(ceux qui nous fournissent les résultats les plus pertinents sur les 10 premiers retours)*, sont donc transferer sur le Serveur API afin de pouvoir les mettrent à disposition des utilisateurs.<br>
Les runs taggés (tests de configuration) ne font pas partie de ce classement : ils sont comparés à part au run hebdomadaire du même modèle.
Les modèles, à l'image de la base de données, seront donc sauvegarder pendant 3 semaines, ce qui permet d'avoir un suivis de nos performances, mais également de pouvoir recharger un ancien modèle au besoin.<br>
Tout les modèles sont conserver, ainsi, il sera parfaitement possible de mettre à disposition plus de modèles sans avoir à les entrainer de nouveaux.<br>
Les metriques alors obtenues pour chaques modeles sont déposer sur le raspberry pi via SFTP, puis sont alimentés dans la base de données metrics pour avoir une historisation claire et propre de celles ci.

### <u>Réentrainement à la demande</u>
En plus du run hebdomadaire, il est possible de demander le réentrainement d'un ou plusieurs modèles depuis la zone Admin du site.

Le déroulement est le suivant:
1. La demande est déposée depuis le site, elle est alors enregistrée sur le raspberry pi.
2. Chaque nuit à 00h00, le cluster récupère la demande et lance un entrainement pour chacun des modèles demandés.
3. Une fois l'entrainement terminé, le cluster dépose les métriques sur le raspberry pi, qui sont directement importées dans la base de données.
4. Les résultats sont alors visibles depuis la page Monitoring du site.

Pour chaque modèle, il est possible de choisir:
- Le mode : fine-tuning, ou simple lancement avec de nouveaux hyperparamètres.
- Le point de départ : le modèle de base, ou une sauvegarde des 3 dernières semaines.
- Les hyperparamètres : epochs, batch, learning rate, nombre de tokens maximum...
- Le tag du run : facultatif, il permet de retrouver plus facilement le run dans le Monitoring. Les runs taggés sont comparés au run hebdomadaire du même modèle, et peuvent être testés depuis la page Recherche.
- L'action : Test ou Load.

### <u>Gestion des modèles</u>
La zone Admin permet aussi de gérer les modèles du cluster. Ces demandes sont appliquées par le cluster au même passage de 00h00 :
- **Importer** un modèle depuis Hugging Face (dépôt, nom court, tag, précision et préfixes). Un job Slurm sur la node GPU le télécharge, vérifie ses vecteurs hors ligne, puis l'ajoute au run hebdomadaire. Pour la sécurité, un modèle qui exige `trust_remote_code` est refusé, et seuls les poids au format `safetensors` sont téléchargés (jamais de pickle, qui exécute du code au chargement).
- **Désactiver** un modèle : il n'est plus lancé par le run hebdomadaire ni proposé au réentraînement, mais sa configuration et ses poids sont conservés. Il peut être réactivé.
- **Supprimer** un modèle : sa configuration est archivée sur le cluster et retirée du run hebdomadaire.

Pour le fine-tuning, le jeu de données est directement déposé depuis le site (fichier JSONL).

### <u>Test et Load</u>
- **Test :** Permet d'essayer une configuration sans modifier celle du cluster.
- **Load :** Si l'entrainement réussit, la configuration remplace celle du cluster et sera utilisée lors des prochains runs hebdomadaires. L'ancienne configuration est conservée.

Lorsqu'un modèle fine-tuné est chargé, ses poids sont conservés tant que la configuration les utilise. Une fois remplacés par un nouveau Load, ils sont encore gardés 3 semaines, puis supprimés automatiquement.<br>
Le tag choisi ne s'applique qu'au run demandé : les runs hebdomadaires gardent le format `AAAA-WW-{model}`.

---
# Interface Utilisateur [En cours]
### <u>Site Web</u>
Le site Web sert donc d'interface utilisateur de notre outils. Il offrira plusieurs possibilités:
- Consulter le Monitoring des Modeles pour voir nos performances
- Avoir un Dashboard de notre systeme
- Pouvoir envoyer une requete au modèle choisis et recevoir les 10 meilleurs repos concernant les topics
- Consulter la base de données via requête SQL et consulter ainsi les ReadMe, affichés comme sur GitHub (mise en forme, images, liens).
- Avoir une visualisation de la distribution de nos données : celle de toute la base, et celle exacte du corpus d'inférence.
- Pouvoir effectuer un réentrainement de nos modèles (Admin).
- Ajouter des modèles pour les inférences (Admin).

---
### <u>Model Switch</u>
Il sera donc possible de pouvoir switch le modèle utiliser parmis les 3 meilleurs modèles de l'entrainement actuelle. Ces modèles seront déjà présent sur le Serveur API et seront mis à jour ou remplacer chaque semaine à la suite de l'entrainement, tout en s'assurant de la continuité de fonctionnement du service.<br>
Les runs taggés de la semaine sont également proposés, afin de pouvoir tester une configuration avant de l'adopter (Load).

---
### <u>Monitoring Models</u>
Le but du monitoring des modèles est de pouvoir consulter rapidement les performances de nos modèles selon nos métriques.<br>
Le classement ne prend en compte que les runs hebdomadaires (un par modèle). Les runs taggés sont affichés à part, avec leur écart par rapport au run hebdomadaire du même modèle et les paramètres modifiés, afin de déterminer si leur configuration est pertinente.<br>
Il est également possible de consulter la configuration exacte utilisée par chaque run, ainsi que le résultat d'un Load (configuration adoptée ou non). Les résultats détaillés et l'historique des runs sont affichés par pages de 10 runs.<br>
Le réentraînement des modèles se fait depuis la zone Admin.

---
### <u>Dashboard système</u>
Le Dashboard système permet de voir l'état actuel de notre système:
- **PostgreSQL :** état de la base, temps de réponse, taille et nombre de repos.
- **API d'inférence :** état et temps de réponse.
- **Cluster :** dernier run, et activité du cluster (dernier signal reçu, jobs en file).
- **Collecteur :** état (en collecte, quota épuisé, pause du week-end...), date de reprise, nombre de tokens GitHub utilisés et restants, nombre de workers actifs et nombre total de repos collectés.
- **Builds :** état du dernier build (CI) sur `main` de chaque repository. Le détail des commits n'est visible que par les administrateurs, les repositories étant privés.

---
### <u>Tester le site en local</u>
Le site peut être testé sur mon ordinateur avant d'être poussé sur GitHub (`site_local/local.sh run`).<br>
Il utilise une copie réduite de la base du raspberry (1000 repos, ainsi que les runs, évaluations et accusés), avec les mêmes tables, rôles et droits, une API d'inférence simulée et un état fictif du collecteur.<br>
Il lance le site présent dans le dossier `site`, sur la branche actuellement extraite : il suffit donc de changer de branche pour tester une autre version.

---
# CI/CD
### <u>Intégration continue (CI)</u>
Chaque push ou Pull Request sur les branches `main` et `dev` des repositories site, Cluster, Raspberry et Collector déclenche une vérification automatique via GitHub Actions:
- Recherche des erreurs dans le code Python (`ruff`) et dans les scripts shell (`shellcheck`).
- Vérification des fichiers de services systemd, et des configurations des modèles pour le Cluster.
- Tests automatiques (`pytest`), un peu plus de 200 au total : formulaire Admin, rendu des README, validation des demandes par le cluster, Test et Load, métriques d'évaluation, importeur, collecteur, déploiement...
- Vérification des valeurs partagées entre les repositories (bornes des hyperparamètres, dimension importable, format des tags) : un même fichier `contract.json` est présent dans site, Cluster et Raspberry, et la CI échoue si l'un d'eux change seul.

Les tests du Cluster n'ont pas besoin de GPU : les librairies de calcul sont simulées, seule la logique est vérifiée.

---
### <u>Déploiement continu (CD)</u>
Afin de respecter la règle de sécurité du projet, ce n'est pas GitHub qui se connecte au raspberry pi pour déployer : c'est le raspberry qui vient vérifier toutes les 5 minutes s'il y a une nouvelle version (`deploy.timer`).

Pour chaque repository (site, Raspberry, Collector), le déploiement ne se fait que si:
- Un nouveau commit est présent sur `main`.
- La CI de ce commit est validée.
- Aucune modification n'a été faite directement sur le raspberry (une modification déjà identique à la nouvelle version, comme un fichier supprimé à la main que le commit supprime aussi, ne bloque pas).

Il applique ensuite ce qui a changé (dépendances, services, importeur...) et redémarre les services concernés, uniquement si des fichiers qu'ils utilisent ont changé (une modification de la documentation ou des tests ne redémarre rien). Pour le site, si celui-ci ne répond plus après la mise à jour, le raspberry revient automatiquement à la version précédente.<br>
Les migrations de la base de données ne sont jamais lancées automatiquement, et une nouvelle tâche planifiée n'est jamais activée automatiquement : elles sont seulement signalées.<br>
Une notification est envoyée sur mon téléphone via `ntfy` à chaque déploiement ou échec.

Le Cluster n'a quant à lui pas de déploiement automatique, puisque rien ne doit se connecter à lui : il est mis à jour manuellement depuis le cluster (`git pull`).

Toutes les modifications se font donc sur la branche `dev`, puis passent sur `main` via une Pull Request, et ne sont jamais faites directement sur le raspberry.

---
### <u>Tâches planifiées</u>
Toutes les tâches planifiées du raspberry sont des timers systemd versionnés dans le repository Raspberry : sauvegarde, export, purge des chunks, import des résultats et du signal de vie du cluster, déploiement et rapport des connexions.<br>
Le raspberry peut ainsi être reconstruit entièrement à partir des repositories.

---
# Models
### <u>intfloat/multilingual-e5-small</u>
- **Paramètres :** 118M (384 dimensions)
- **Langues supportées :** 100 langues (basé sur XLM-RoBERTa)
- **Contexte max :** 512 tokens
- **Point fort :** Extrêmement léger et rapide sur CPU, idéal pour l'indexation de gros volumes à faible coût.

---
### <u>intfloat/multilingual-e5-base</u>
- **Paramètres :** 278M (768 dimensions)
- **Langues supportées :** 100 langues
- **Contexte max :** 512 tokens
- **Point fort :** Le meilleur compromis direct de la famille E5, offrant un gain net de précision sémantique par rapport au modèle *small* tout en restant très véloce.

---
### <u>Snowflake/snowflake-arctic-embed-m-v2.0</u>
- **Paramètres :** 305M au total, dont 113M actifs hors embeddings (768 dimensions)
- **Langues supportées :** 74 langues
- **Contexte max :** 8 192 tokens
- **Point fort :** Excellentes performances sur les benchmarks de recherche (MTEB) sans sacrifier l'anglais, avec support Matryoshka (MRL) permettant de compresser les vecteurs jusqu'à 256 dimensions sans perte notable.

---
### <u>google/embeddinggemma-300m</u>
- **Paramètres :** ~300M (768 dimensions)
- **Langues supportées :** Plus de 100 langues
- **Contexte max :** 2 048 tokens
- **Point fort :** Conçu spécifiquement pour l'exécution locale (CPU, mobile, faible RAM) avec support MRL (128 à 768 dimensions) et gestion native de prompts spécialisés par tâche (recherche, code, QA).

---
### <u>nomic-ai/nomic-embed-text-v2-moe</u>
- **Paramètres :** 475M au total, dont 305M actifs à l'inférence (768 dimensions)
- **Langues supportées :** ~100 langues
- **Contexte max :** 512 tokens
- **Point fort :** Architecture *Mixture-of-Experts* (8 experts, routage top-2) offrant la qualité d'un modèle deux fois plus gros pour le coût de calcul d'un modèle moyen.

---
### <u>BAAI/bge-m3</u>
- **Paramètres :** 568M (1 024 dimensions)
- **Langues supportées :** Plus de 100 langues
- **Contexte max :** 8 192 tokens
- **Point fort :** La référence open-source pour le RAG hybride : il génère en une seule passe des vecteurs denses (sémantique), des vecteurs sparses (mots-clés type BM25) et du multi-vecteur (type ColBERT).

---
### <u>Alibaba-NLP/gte-multilingual-base</u>
- **Paramètres :** 305M (768 dimensions)
- **Langues supportées :** Plus de 70 langues
- **Contexte max :** 8 192 tokens
- **Point fort :** Très rapide à l'inférence pour un modèle à contexte long (8k), capable de produire des représentations denses élastiques et sparses pour la recherche sur documents longs.

---
### <u>Qwen/Qwen3-Embedding-0.6B</u>
- **Paramètres :** 600M (jusqu'à 1 024 dimensions)
- **Langues supportées :** Plus de 100 langues (langages naturels et langages de programmation)
- **Contexte max :** 32 768 tokens
- **Point fort :** Fenêtre de contexte massive (32k) et support des instructions personnalisées sur les requêtes (*instruction-aware*), offrant une précision de pointe sur les textes techniques, le code et les documents très longs.

---
### <u>intfloat/multilingual-e5-large-instruct</u>
- **Paramètres :** 560M (1 024 dimensions)
- **Langues supportées :** 100 langues (basé sur XLM-RoBERTa-large)
- **Contexte max :** 512 tokens
- **Point fort :** Version *instruction-tuned* la plus puissante de la famille E5 : le préfixe d'instruction sur la requête permet d'adapter finement la recherche au domaine visé (ici, la documentation de dépôts logiciels) avec une excellente précision sémantique.

---
### <u>ibm-granite/granite-embedding-278m-multilingual</u>
- **Paramètres :** 278M (768 dimensions)
- **Langues supportées :** Plus de 100 langues (dont 12 langues entraînées intensivement)
- **Contexte max :** 512 tokens
- **Point fort :** Modèle entreprise ultra-propre (licence Apache 2.0, sans code distant `trust_remote_code`), très rapide à l'inférence et spécialement optimisé pour le retrieval technique et documentaire.

---
### <u>jinaai/jina-embeddings-v3</u>
- **Paramètres :** 570M (1 024 dimensions)
- **Langues supportées :** 89 langues (plus principaux langages de programmation)
- **Contexte max :** 8 192 tokens
- **Point fort :** Utilise des adaptateurs LoRA spécialisés par tâche (retrieval, clustering, classification) et le support Matryoshka (MRL de 32 à 1 024 dimensions), offrant d'excellentes performances sur le mélange de texte naturel et de code informatique.

---
# Metrics & Méthodes :

## <u>Méthodes:</u>

### <u>BM25 :</u>

Algorithme de recherche d'information et de classement textuel basé sur le modèle probabiliste de pertinence.
Calcule un score de pertinence pour chaque document par rapport aux termes d'une requête en s'appuyant sur trois facteurs principaux.
- Fréquence des termes (Term Frequency - TF) avec saturation (k₁)
- Fréquence inverse de document (Inverse Document Frequency - IDF)
- Normalisation de la longueur du document (b) 

__Caractéristiques :__
- Recherche Lexicale (recherche de termes exacts)
- Léger et rapide
- Recherche hybride


---
### <u>E5 (EmbEddings from bidirEctional Encoder rEpresentations)</u>

Famille de modèles d'embedding de texte développée par Microsoft, basée sur des architectures Transformers (comme BERT) entraînées par apprentissage contrastif.
Transforme les requêtes et les documents en vecteurs denses pour mesurer leur proximité sémantique (via similarité cosinus).

__Caractéristiques :__
- Recherche sémantique
- Nécessite l'utilisation de préfixes spécifiques (`query: ` et `passage: `)
- Plus lourd en calcul et stockage (index vectoriel) que BM25

---
### <u>RRF (Reciprocal Rank Fusion)</u>

Algorithme de fusion de classements permettant de combiner les résultats issus de plusieurs moteurs de recherche différents (ex: BM25 + recherche vectorielle E5).
Attribue un nouveau score à chaque document selon son rang dans chaque liste, via la formule : `1 / (k + rang)` (avec `k` souvent fixé à 60).

__Caractéristiques :__
- Pilier de la recherche hybride moderne
- Indépendant des échelles de scores (additionne des rangs, pas des scores bruts incompatibles)
- Ne nécessite aucun entraînement ni réglage complexe de poids

---

## <u>Metrics</u>
### <u>Recall@10 (Rappel à 10)</u>

Métrique d'évaluation mesurant la capacité d'un système à retrouver l'ensemble des documents pertinents parmi les 10 premiers résultats retournés.
Calcul : `hits / min(10, nb pertinents)`

__Caractéristiques :__
- Mesure l'exhaustivité (ne rate pas d'infos clés)
- Ignore l'ordre de classement au sein du top 10 (le 1er vaut autant que le 10e)
- Essentiel pour evaluer un systeme de Retrival

---
### <u>MRR (Mean Reciprocal Rank)</u>

Métrique d'évaluation qui mesure à quelle vitesse le système affiche le tout premier résultat pertinent.
Calcule la moyenne de l'inverse du rang du premier document pertinent trouvé sur un ensemble de requêtes (ex: 1er = 1, 2e = 0.5, 3e = 0.33...).

__Caractéristiques :__
- Focalisé sur le premier bon résultat uniquement
- Idéal quand il n'y a qu'une seule bonne réponse attendue (ex: question-réponse factuelle)
- Très sensible aux erreurs en haut de liste

---
### <u>nDCG@10 (Normalized Discounted Cumulative Gain à 10)</u>

Métrique d'évaluation de référence qui mesure la qualité globale du classement sur les 10 premiers résultats en tenant compte à la fois du degré de pertinence et de la position.
Applique une pénalité logarithmique (Discount) aux documents pertinents qui apparaissent trop bas dans la liste, puis normalise le score entre 0 et 1 par rapport au classement idéal.

__Caractéristiques :__
- Gère la pertinence graduée (ex: pas pertinent = 0, moyen = 1, très pertinent = 2)
- Récompense fortement les meilleurs résultats placés tout en haut (rang 1, 2, 3)
- Standard industriel pour évaluer les moteurs de recherche et les modèles de Reranking
---
# Difficultés:
### <u>Matériel </u>
L'une des principales difficulté qui à été identifié concerne le materiel. En effet, dans un premier temps j'ai commencer par utiliser un serveur AWS classique (version d'essais avec X credits). Cependant, je me suis vite rendu compte que je commencais à subir un problème d'espace de stockage (30 Go) et de puissance de calcul (2 Go de Ram et 0 GPU).<br>
Or, si je souhaitais augmenter la puissance et l'espace que j'avais alors à disposition, il me fallait augmenter la consommation de credits (car plus gros forfait), mais après un rapide calcul en cout de credit par jour, il est apparu que je n'aurais pas assez de crédits pour tenir sur l'intégralité du temps de projet.

C'est pourquoi, je me suis tourné vers une solution local. Ayant alors un raspberry pi avec 4 Go de RAM inutilisé ainsi qu'un SSD de 2 To, j'ai remplacé le serveur AWS qui me limitait par une solution alternative, car le projet necessitait de tourner de manière continue, ce qui n'aurait pas été possible sur le cluster universitaire ni sur mon ordinateur fixe personnel.

Cependant, malgré le fait que le problème de stockage ai été résolu, il demeurait encore le problème de la puissance, car le raspberry pi ne pouvait simultanément s'occuper de l'entrainement (CPU/GPU + RAM), de l'inférence (GPU/CPU + RAM), de l'hebergement du site (CPU + RAM) et du scrapping (CPU + RAM).

Il a donc été necessaire de répartir ces différentes tâches dans différentes structures étant capable de les encaisser:
- Raspberry : Site + Scrapping (peu couteux en RAM).
- Cluster : Entrainement (Présence de puissant GPU et d'une forte capacité de calcul)
- Serveur Tiers : Inférence lors du Retrival (Puissance de calcul moyenne)
- AWS : Il ne sert que de pont de connexion entre le Cluster et le raspberry, ainsi qu'entre le Cluster et le serveur d'API, car ne souhaitant pas exposé le port 22 de ma box personelle, et le Cluster ayant une plage de diffusion réduite.

Ainsi, les différentes tâches seraient traités par différents systèmes aux capacités différentes.

### <u>Sécurité</u>
Cependant, ce choix d'utiliser plusieurs structures pour effecuter le projet, bien que résolvant le problème initial de puissance, à également apporté un nouveau problème : la sécurité.
Effectivement, qui dit plus de point d'entrée, dit plus de faille possible. Ce qui est un gros problème si l'on sait que le Cluster ici est un cluster de calcul universitaire, et que le serveur tiers m'a été gracieusement mis à disposition pour mon API (Merci Mewen).

Il a donc été nécessaire de mettre en place des mesures de sécurités pour s'assurer que tout se déroule correctement. Ainsi, comme évoquer précédemment au point `Architecture`, il a été decider que le cluster se connecterais de lui même au raspberry pi, et que rien ne pourrait se connecter au Cluster. De ce fait, même si mon raspberry venait à être compromis, le cluster n'en serait pas inquiété.

Toutes les connexions se feraient via des protocoles sécurisés (Tailscale, SFTP, WireGuard), et chaque connexion à mon raspberry pi était alors loggé et fournit dans un rapport matinal sur mon téléphone via `ntfy`, indiquant les clés ssh utilisés pour se connectés, le nombre de connexions effectués et les commandes sudo qui ont été utilisés.<br>
Ce rapport, bien que informatif, me permet alors d'avoir un suivi complet de ce qu'il se passe sur le raspberry.

---
# Points à améliorer:
Voici une liste des points à améliorer que j'ai relevé durant mon projet:
- Manque de redondance des tâches : Si un de système viens à cesser de fonctionner, il n'y a rien pour le fallback.
- Manque de redondance des données : Toutes les données sont présentes uniquement sur le raspberry, si il casse, je perds tout.
- Simplifié l'architecture : Cela se réfère également au premier point, mais bien que cela puisse être interessant d'avoir une architecture complexe pour un projet universitaire, dans un cadre pratique réel, on va préféré la simplicité.
- Le jeu de donnée de validation a été réaliser par moi même, il pourrait être pertinent de recevoir l'avis de plusieurs annotateurs pour eviter les biais.
- L'import des chunks en base n'est possible que pour les modèles de 384 dimensions (`repo_chunks` en `halfvec(384)`) : les autres modèles ne sont évalués qu'en run d'analyse.
- La page de requêtes SQL du site est publique : même limitées (lecture seule, 10 secondes, 10 par minute et par visiteur), elles restent une porte ouverte sur la base.
- Le Cluster n'a pas de déploiement automatique : sa mise à jour reste manuelle.
- Le script réellement utilisé pour le rapport des connexions (`/usr/local/bin/auth_report.sh`) n'est pas encore versionné.
- Les modifications des fichiers de configuration `.env` (non versionnés car ils contiennent des secrets) nécessitent un redémarrage manuel du service concerné.

En définitive, les principaux points à améliorer sont inhérent aux choix de conceptions de l'architecture qui ont été fait.


---
<u>**Avertissement (Disclaimer)**<br></u>
Des outils d'intelligence artificielle générative (Claude et Gemini) ont été utilisés comme assistance au développement pour certaines parties du code de ce projet.<br>
L'ensemble du code généré a été systématiquement relu, vérifié et testé manuellement.<br>
Les choix techniques, l'architecture globale ainsi que la rédaction des rapports et des livrables ont été intégralement réalisés par l'étudiant.


