# CLAUDE.md

## Objectif d'apprentissage (à respecter en priorité)

L'utilisateur est analytics engineer et utilise ce projet pour **monter en compétence sur Databricks**.

- **Expliquer chaque changement avant de le faire** : quoi, pourquoi, quel concept Databricks/Spark est en jeu, et les alternatives écartées. Attendre son accord avant de modifier un fichier.
- Privilégier des changements petits et ciblés, un concept à la fois.
- Relier les explications aux bonnes pratiques Databricks (Unity Catalog, Delta Lake, Auto Loader, Jobs, etc.) et signaler les écarts plutôt que les corriger en silence.
- Répondre en français.

## Le projet

**Swiss Hydro & Power Analytics** : lakehouse medallion sur la production d'électricité, les prix et l'hydrologie en Suisse (PySpark · Databricks · Microsoft Fabric).

Le code est constitué de notebooks Databricks (`.ipynb`) dans `notebooks/`, exécutés dans un workspace Databricks (pas en local : ils utilisent `spark`, `dbutils`, `display`).

### Architecture medallion (Unity Catalog)

Catalogue `workspace` (paramétrable), un schéma par couche :

| Schéma | Contenu |
|---|---|
| `energy_raw` | Volume `landing` : fichiers bruts des sources, jamais modifiés |
| `energy_bronze` | Tables Delta brutes + métadonnées d'ingestion |
| `energy_silver` | Données nettoyées, typées, horodatées en UTC |
| `energy_gold` | KPI métier pour Power BI et l'analyse |

Landing : `/Volumes/<catalog>/energy_raw/landing/<source>/<dataset>/`, avec les dossiers `entsoe/generation`, `entsoe/prices`, `ofev/discharge`, `meteoswiss/weather`, `ofen/reservoirs`.

### Notebooks

| Notebook | Rôle |
|---|---|
| `00_setup` | Crée les schémas, le Volume `landing` et les dossiers par source (idempotent) |
| `01_bronze_prices` | Extrait les prix day-ahead (API ENTSO-E, `A44`, zone `10YCH-SWISSGRIDZ`) mois par mois et dépose le XML brut dans le landing |
| `02_bronze_entsoe_prices` | Charge les XML du landing dans `energy_bronze.entsoe_prices_raw` (un fichier = une ligne) |

État actuel : seule la chaîne des prix ENTSO-E existe, jusqu'à Bronze. Silver, Gold et les autres sources restent à faire.

## Conventions

- **Nommage des notebooks** : `NN_<couche>_<source>_<dataset>`, numérotés dans l'ordre d'exécution.
- **Notebooks documentés en français** : cellule markdown d'en-tête (projet, source, destination, tableau des étapes), puis une section markdown avant chaque bloc de code, et des commentaires de code en français.
- **Paramètres via widgets** (`dbutils.widgets`) plutôt que des valeurs en dur (catalogue, projet, dates, zone).
- **Secrets** : jamais de token dans le code ; lecture via `dbutils.secrets.get(scope="entsoe", key="api-token")`.
- **Idempotence** : tout notebook doit pouvoir être relancé sans erreur ni doublon (`IF NOT EXISTS`, `overwrite`, ou `MERGE`).
- **Landing immuable** : les fichiers bruts ne sont jamais modifiés ni écrasés. Nom : `<dataset>_<zone>_<début>_<fin>_<horodatage ingestion UTC>.xml`.
- **Bronze = données telles que reçues** : pas de parsing, colonnes techniques préfixées par `_` (`_source_file`, `_source_path`, `_file_modified_at`, `_ingested_at`).
- **Temps** : tout est stocké en UTC ; les dates saisies par l'utilisateur sont en heure suisse (`Europe/Zurich`) et converties.
- **Noms de tables complets** : `catalogue.schéma.table`.
