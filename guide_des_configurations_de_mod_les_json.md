# Guide des configurations de modèles (JSON)

Ce document décrit le format des fichiers de configuration `.json` à déposer sur le Raspberry Pi dans `/mnt/pgdata/export/configs/active/` (et à référencer dans `/mnt/pgdata/export/configs/manifest.json`) pour piloter les runs d'inférence et de fine-tuning sur le cluster.

---

## 1. Sans fine-tuning (Inférence standard)

Les fichiers existants (`bge-m3.json`, `e5-base.json`, `gte-multilingual.json`) fonctionnent sans modification : le pipeline charge le modèle pré-entraîné depuis le cache hors-ligne du cluster (`~/raid-etu/models`), calcule les embeddings, sauvegarde un instantané du modèle et lance l'évaluation.

### Exemple (`bge-m3.json`)

```json
{
  "model_name": "BAAI/bge-m3",
  "revision": "5617a9f61b028005a4858fdac845db406aefb181",
  "model_version": "bge-m3-5617a9f6",
  "chunking_version": "c1",
  "embedding_dim": 1024,
  "passage_prefix": "",
  "query_prefix": "",
  "normalize": true,
  "max_tokens": 512,
  "trust_remote_code": false,
  "dtype": "float16",
  "batch_size": 256,
  "default_tag": "bgem3"
}
```

> **Astuce — Réutiliser un modèle déjà sauvegardé sans le réentraîner :**  
> Pour exécuter une inférence simple à partir d'un modèle précédemment fine-tuné et sauvegardé sur le cluster, ajoutez simplement la clé `"base_saved_model"` (visible dans `/mnt/pgdata/export/inbox/saved_models.json` sur le Pi) :
> ```json
> "base_saved_model": "2026-W47/bge-m3-ft-v1"
> ```

---

## 2. Avec fine-tuning (Réentraînement + Inférence + Évaluation)

Pour déclencher une phase d'entraînement avant la génération des embeddings, ajoutez `"mode": "finetune"` ainsi que le bloc `"training"`.

### Exemple (`bge-m3-ft-v1.json`)

```json
{
  "mode": "finetune",
  "base_saved_model": "2026-W47/bge-m3-5617a9f6",
  "model_name": "BAAI/bge-m3",
  "revision": "5617a9f61b028005a4858fdac845db406aefb181",
  "model_version": "bge-m3-ft-v1",
  "chunking_version": "c1",
  "embedding_dim": 1024,
  "passage_prefix": "",
  "query_prefix": "",
  "normalize": true,
  "max_tokens": 512,
  "trust_remote_code": false,
  "dtype": "float16",
  "batch_size": 256,
  "default_tag": "bgem3ft1",
  "training": {
    "dataset_file": "train_pairs.jsonl",
    "epochs": 2,
    "train_batch_size": 16,
    "learning_rate": 2e-5,
    "warmup_ratio": 0.1
  }
}
```

---

## 3. Référence des paramètres

### Paramètres généraux (obligatoires)

| Clé | Type | Description |
| :--- | :--- | :--- |
| `model_name` | `string` | Identifiant HuggingFace du modèle (ex: `"BAAI/bge-m3"`, `"intfloat/multilingual-e5-base"`). |
| `revision` | `string` | Hash exact du commit HuggingFace présent dans le cache local du cluster. |
| `model_version` | `string` | Nom unique identifiant cette version du modèle (utilisé pour l'indexation et le dossier de sauvegarde). |
| `chunking_version` | `string` | Version de la stratégie de découpage (ex: `"c1"`). |
| `embedding_dim` | `int` | Dimension des vecteurs de sortie (ex: `384`, `768`, `1024`). |
| `passage_prefix` | `string` | Préfixe ajouté devant chaque chunk/passage (`"passage: "` pour E5, `""` pour BGE-M3 et GTE). |
| `query_prefix` | `string` | Préfixe ajouté devant chaque requête (`"query: "` pour E5, `""` pour BGE-M3 et GTE). |
| `normalize` | `bool` | Normalisation L2 des vecteurs (`true` recommandé). |
| `max_tokens` | `int` | Longueur maximale de séquence en tokens (`512`). |
| `trust_remote_code` | `bool` | Autoriser l'exécution de code personnalisé du dépôt modèle (`true` requis pour GTE). |
| `dtype` | `string` | Précision d'inférence (`"float16"` ou `"float32"`). |
| `batch_size` | `int` | Taille des lots GPU lors de l'encodage (ex: `256`). |
| `default_tag` | `string` | Suffixe ajouté au `run_id` (**12 caractères alphanumériques maximum**, sans tirets ni underscores). |

### Paramètres spécifiques au fine-tuning

| Clé | Type | Description |
| :--- | :--- | :--- |
| `mode` | `string` | Mettre `"finetune"` pour activer l'entraînement avant l'encodage. Omit ou autre valeur = inférence seule. |
| `base_saved_model` | `string` *(opt.)* | Chemin relatif dans `~/raid-etu/models/save/` (ex: `"2026-W47/bge-m3-5617a9f6"`) pour repartir d'une sauvegarde locale. Si absent ou `null`, repart du modèle brut (`model_name` / `revision`). |
| `training.dataset_file` | `string` | Nom du fichier `.jsonl` présent dans `/mnt/pgdata/export/data/` sur le Pi. |
| `training.epochs` | `int` | Nombre de passes complètes sur le jeu d'entraînement (défaut : `2`). |
| `training.train_batch_size` | `int` | Taille de batch par GPU pendant l'entraînement (défaut : `16`). |
| `training.learning_rate` | `float` | Taux d'apprentissage (défaut : `2e-5`). |
| `training.warmup_ratio` | `float` | Proportion des pas dédiée au warmup (défaut : `0.1`). |

---

## 4. Règles importantes et bonnes pratiques

1. **Identifiants distincts pour un modèle fine-tuné :**  
   Modifiez toujours `model_version` et `default_tag` (max 12 caractères alphanumériques, ex: `bgem3ft1`) par rapport au modèle de base. Cela évite d'écraser le modèle original dans `~/raid-etu/models/save/` et garantit un fichier de métriques distinct (`metrics_<run_id>-bgem3ft1.json`).
2. **Format du jeu d'entraînement (`dataset_file`) :**  
   Le fichier déposé dans `/mnt/pgdata/export/data/` doit être au format JSON Lines (`.jsonl`), avec une paire par ligne :
   ```json
   {"anchor": "requête ou description recherchée", "positive": "extrait de README ou dépôt pertinent"}
   ```
   Les préfixes `query_prefix` et `passage_prefix` de la configuration sont automatiquement appliqués respectivement aux champs `anchor` et `positive` lors du chargement.
3. **Traçabilité automatique :**  
   À chaque sauvegarde, le fichier exact de configuration utilisé est enregistré dans `run_model_config.json` à l'intérieur du dossier du modèle sur le cluster, ainsi que dans `saved_models.json` et `metrics_<run_id>.json` dans `/mnt/pgdata/export/inbox/` sur le Raspberry Pi.