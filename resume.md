# Architecture

```mermaid
flowchart LR
    Cluster["Cluster<br/>(Entraînement & Embeddings)"]
    AWS["AWS<br/>(Bridge)"]
    RPI["Raspberry Pi<br/>(Hébergeur & Base de données)"]
    API["Serveur<br/>(API pour Retrieval)"]
    Web["Site Web"]

    Cluster --> AWS
    AWS --> RPI
    RPI --> API
    RPI --> Web
```
##
### <u> Sécurité </u>
Une attention particulière à été prêter à la sécurité de cette architecture en respectant au mieux la règle suivante: <br>
"C'est le système sécurisé qui se connecte au système vulnérable et non l'inverse." <br>
A ces fins, le cluster se connecte via SFTP sur le raspberry pi en utilisant le serveur AWS (ouverture du port 22 avec filtre sur le AWS avec controle des IPs pour ne laisser passer que le cluster, puis tailscale entre AWS et le pi).
Le cluster sait donc se connecter successivement a l'AWS et au raspberry avec des connexions ssh, mais ni AWS ni le raspberry ne sont en mesure de se connecter au cluster.

Lors de cette connexion SFTP, le cluster ne peut lire que dans `input` et déposer ses metrics et poids dans `output`, de ce fait, il ne peut pas acceder au reste des données du raspberry, ce qui rajoute un controle supplémentaire sur les accès.

Un transfert des poids était également necessaire entre le Cluster et le Serveur API. [Inserer ici comment cela à été réaliser].

Enfin pour l'utilisation de l'API [Inserer ici comment cela à été réaliser].

##
# Base de données
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

Base de données sauvegardé tout les samedis à 0h00. <br>
Seuls les 3 dernières sauvegardes sont conservés, ce qui fait un historique de 3 semaines.

##
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

##
### <u>Découpage:</u>

L'API de GitHub plafonne chaque recherche à 1000 résultats. Pour contourner cette limite, l'espace de recherche est découpé en segments (tranche d'étoiles × licence × plage de dates). Lorsqu'un segment dépasse 1000 repos, il est scindé en deux sous-segments plus fins, d'abord par étoiles, puis par dates si les étoiles sont indivisibles, et ainsi de suite jusqu'à passer sous le seuil. Les sous-segments sont placés dans une file d'attente de type FILO : le dernier segment créé est le premier traité, ce qui permet de descendre un segment saturé jusqu'à ses feuilles avant d'en ouvrir un autre. La progression de chaque segment est sauvegardée pour pouvoir reprendre la collecte à tout moment.

##
### Récupération

<u>**Première récupération**</u>

Dans un premier temps, au début du projet afin de se consistuer un DataWarehouse conséquent, nous avons laisser tourner la récuperation pendant 2 semaines, en récuperant des repositories depuis le début de la création de git.

<u>**Récupération lors du projet**</u>

Ensuite, lorsque nous avons collecter une grosse quantités de données (Environs 5 millions de repos), nous modifions le script de récupération, cette fois-ci, il ne récupère que sur les 5 derniers jours de modification (push, création), toujours en ignorant le nombre d'etoiles.

Nous stoppons la récuperation les Week-end, car c'est à ce moment que s'effectue la sauvegarde de la base de données et les entrainements.

# Entrainement
## <u>Données </u>
Sélections des repos ayant plus de X etoiles et X caractères dans ReadMe.<br>
Les données qui (voir quels données sont séléctionner) sont déposer dans un espace prévu à cet effet tout les samedi à 3h00.

Le samedi à 6h00, le cluster se connecte sur le Raspberry PI, dans l'espace SFTP, afin de récuperer les données alors déposés pour faire son entrainement.

## <u>Entrainement</u>
L'entrainement est alors inité tout les samedis à 6h00 via un Cron. Il execute le fichier batch prévu à cet effet afin de pouvoir s'allouer un espace sur la node GPUE-07 avec 1 gpu, 8 cpu et 48G de ram. Le script est prévu de tourner 6h au maximum.<br>
L'entrainement s'effectue alors avec le même jeu de donnée sur tous les modèles de notre pool models. Une fois l'entrainement effecuté, nous calculons les métriques de nos modèles que nous redéposons sur le 

## <u>Jeu de validation</u>


# Models

### <u>intfloat/multilingual-e5-small</u>
- **Paramètres :** 118M (384 dimensions)
- **Langues supportées :** 100 langues (basé sur XLM-RoBERTa)
- **Contexte max :** 512 tokens
- **Point fort :** Extrêmement léger et rapide sur CPU, idéal pour l'indexation de gros volumes à faible coût.

##
### <u>intfloat/multilingual-e5-base</u>
- **Paramètres :** 278M (768 dimensions)
- **Langues supportées :** 100 langues
- **Contexte max :** 512 tokens
- **Point fort :** Le meilleur compromis direct de la famille E5, offrant un gain net de précision sémantique par rapport au modèle *small* tout en restant très véloce.

##
### <u>Snowflake/snowflake-arctic-embed-m-v2.0</u>
- **Paramètres :** 305M au total, dont 113M actifs hors embeddings (768 dimensions)
- **Langues supportées :** 74 langues
- **Contexte max :** 8 192 tokens
- **Point fort :** Excellentes performances sur les benchmarks de recherche (MTEB) sans sacrifier l'anglais, avec support Matryoshka (MRL) permettant de compresser les vecteurs jusqu'à 256 dimensions sans perte notable.

##
### <u>google/embeddinggemma-300m</u> [Actuellement en cours d'étude pour implémentation]
- **Paramètres :** ~300M (768 dimensions)
- **Langues supportées :** Plus de 100 langues
- **Contexte max :** 2 048 tokens
- **Point fort :** Conçu spécifiquement pour l'exécution locale (CPU, mobile, faible RAM) avec support MRL (128 à 768 dimensions) et gestion native de prompts spécialisés par tâche (recherche, code, QA).

##
### <u>nomic-ai/nomic-embed-text-v2-moe</u>
- **Paramètres :** 475M au total, dont 305M actifs à l'inférence (768 dimensions)
- **Langues supportées :** ~100 langues
- **Contexte max :** 512 tokens
- **Point fort :** Architecture *Mixture-of-Experts* (8 experts, routage top-2) offrant la qualité d'un modèle deux fois plus gros pour le coût de calcul d'un modèle moyen.

##
### <u>BAAI/bge-m3</u>
- **Paramètres :** 568M (1 024 dimensions)
- **Langues supportées :** Plus de 100 langues
- **Contexte max :** 8 192 tokens
- **Point fort :** La référence open-source pour le RAG hybride : il génère en une seule passe des vecteurs denses (sémantique), des vecteurs sparses (mots-clés type BM25) et du multi-vecteur (type ColBERT).

##
### <u>Alibaba-NLP/gte-multilingual-base</u>
- **Paramètres :** 305M (768 dimensions)
- **Langues supportées :** Plus de 70 langues
- **Contexte max :** 8 192 tokens
- **Point fort :** Très rapide à l'inférence pour un modèle à contexte long (8k), capable de produire des représentations denses élastiques et sparses pour la recherche sur documents longs.

##
### <u>Qwen/Qwen3-Embedding-0.6B</u>
- **Paramètres :** 600M (jusqu'à 1 024 dimensions)
- **Langues supportées :** Plus de 100 langues (langages naturels et langages de programmation)
- **Contexte max :** 32 768 tokens
- **Point fort :** Fenêtre de contexte massive (32k) et support des instructions personnalisées sur les requêtes (*instruction-aware*), offrant une précision de pointe sur les textes techniques, le code et les documents très longs.

##
### <u>intfloat/multilingual-e5-large-instruct</u>
- **Paramètres :** 560M (1 024 dimensions)
- **Langues supportées :** 100 langues (basé sur XLM-RoBERTa-large)
- **Contexte max :** 512 tokens
- **Point fort :** Version *instruction-tuned* la plus puissante de la famille E5 : le préfixe d'instruction sur la requête permet d'adapter finement la recherche au domaine visé (ici, la documentation de dépôts logiciels) avec une excellente précision sémantique.

##
### <u>ibm-granite/granite-embedding-278m-multilingual</u>
- **Paramètres :** 278M (768 dimensions)
- **Langues supportées :** Plus de 100 langues (dont 12 langues entraînées intensivement)
- **Contexte max :** 512 tokens
- **Point fort :** Modèle entreprise ultra-propre (licence Apache 2.0, sans code distant `trust_remote_code`), très rapide à l'inférence et spécialement optimisé pour le retrieval technique et documentaire.

##
### <u>jinaai/jina-embeddings-v3</u>
- **Paramètres :** 570M (1 024 dimensions)
- **Langues supportées :** 89 langues (plus principaux langages de programmation)
- **Contexte max :** 8 192 tokens
- **Point fort :** Utilise des adaptateurs LoRA spécialisés par tâche (retrieval, clustering, classification) et le support Matryoshka (MRL de 32 à 1 024 dimensions), offrant d'excellentes performances sur le mélange de texte naturel et de code informatique.

##
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


##
### <u>E5 (EmbEddings from bidirEctional Encoder rEpresentations)</u>

Famille de modèles d'embedding de texte développée par Microsoft, basée sur des architectures Transformers (comme BERT) entraînées par apprentissage contrastif.
Transforme les requêtes et les documents en vecteurs denses pour mesurer leur proximité sémantique (via similarité cosinus).

__Caractéristiques :__
- Recherche sémantique
- Nécessite l'utilisation de préfixes spécifiques (`query: ` et `passage: `)
- Plus lourd en calcul et stockage (index vectoriel) que BM25

##
### <u>RRF (Reciprocal Rank Fusion)</u>

Algorithme de fusion de classements permettant de combiner les résultats issus de plusieurs moteurs de recherche différents (ex: BM25 + recherche vectorielle E5).
Attribue un nouveau score à chaque document selon son rang dans chaque liste, via la formule : `1 / (k + rang)` (avec `k` souvent fixé à 60).

__Caractéristiques :__
- Pilier de la recherche hybride moderne
- Indépendant des échelles de scores (additionne des rangs, pas des scores bruts incompatibles)
- Ne nécessite aucun entraînement ni réglage complexe de poids

##

## <u>Metrics</u>

### <u>Recall@10 (Rappel à 10)</u>

Métrique d'évaluation mesurant la capacité d'un système à retrouver l'ensemble des documents pertinents parmi les 10 premiers résultats retournés.
Calcul : `hits / min(10, nb pertinents)`

__Caractéristiques :__
- Mesure l'exhaustivité (ne rate pas d'infos clés)
- Ignore l'ordre de classement au sein du top 10 (le 1er vaut autant que le 10e)
- Essentiel pour evaluer un systeme de Retrival

##
### <u>MRR (Mean Reciprocal Rank)</u>

Métrique d'évaluation qui mesure à quelle vitesse le système affiche le tout premier résultat pertinent.
Calcule la moyenne de l'inverse du rang du premier document pertinent trouvé sur un ensemble de requêtes (ex: 1er = 1, 2e = 0.5, 3e = 0.33...).

__Caractéristiques :__
- Focalisé sur le premier bon résultat uniquement
- Idéal quand il n'y a qu'une seule bonne réponse attendue (ex: question-réponse factuelle)
- Très sensible aux erreurs en haut de liste

##
### <u>nDCG@10 (Normalized Discounted Cumulative Gain à 10)</u>

Métrique d'évaluation de référence qui mesure la qualité globale du classement sur les 10 premiers résultats en tenant compte à la fois du degré de pertinence et de la position.
Applique une pénalité logarithmique (Discount) aux documents pertinents qui apparaissent trop bas dans la liste, puis normalise le score entre 0 et 1 par rapport au classement idéal.

__Caractéristiques :__
- Gère la pertinence graduée (ex: pas pertinent = 0, moyen = 1, très pertinent = 2)
- Récompense fortement les meilleurs résultats placés tout en haut (rang 1, 2, 3)
- Standard industriel pour évaluer les moteurs de recherche et les modèles de Reranking
##
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
- AWS : Il ne sert que de pont de connexion entre le Cluster et le raspberry, car ne souhaitant pas exposé le port 22 de ma box personelle, et le Cluster ayant une plage de diffusion réduite.

Ainsi, les différentes tâches seraient traités par différents systèmes aux capacités différentes.

### <u>Sécurité</u>
Cependant, ce choix d'utiliser plusieurs structures pour effecuter le projet, bien que résolvant le problème initial de puissance, à également apporté un nouveau problème : la sécurité.
Effectivement, qui dit plus de point d'entrée, dit plus de faille possible. Ce qui est un gros problème si l'on sait que le Cluster ici est un cluster de calcul universitaire, et que le serveur tiers m'a été gracieusement mis à disposition pour mon API (Merci Mewen).

Il a donc été nécessaire de mettre en place des mesures de sécurités pour s'assurer que tout se déroule correctement. Ainsi, comme évoquer précédemment au point `Architecture`, il a été decider que le cluster se connecterais de lui même au raspberry pi, et que rien ne pourrait se connecter au Cluster. De ce fait, même si mon raspberry venait à être compromis, le cluster n'en serait pas inquiété.

Toutes les connexions se feraient via des protocoles sécurisés (Tailscale, SFTP, WireGuard), et chaque connexion à mon raspberry pi était alors loggé et fournit dans un rapport matinal sur mon téléphone via `ntfy`, indiquant les clés ssh utilisés pour se connectés, le nombre de connexions effectués et les commandes sudo qui ont été utilisés.<br>
Ce rapport, bien que informatif, me permet alors d'avoir un suivi complet de ce qu'il se passe sur le raspberry.
##
# Points à améliorer:
Voici une liste des points à améliorer que j'ai relevé durant mon projet:
- Manque de redondance des tâches : Si un de système viens à cesser de fonctionner, il n'y a rien pour le fallback.
- Manque de redondance des données : Toutes les données sont présentes uniquement sur le raspberry, si il casse, je perds tout.
- Simplifié l'architecture : Cela se réfère également au premier point, mais bien que cela puisse être interessant d'avoir une architecture complexe pour un projet universitaire, dans un cadre pratique réel, on va préféré la simplicité.

En définitive, les principaux points à améliorer sont inhérent aux choix de conceptions de l'architecture qui ont été fait.
