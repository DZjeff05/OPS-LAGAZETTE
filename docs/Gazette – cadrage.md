# Gazette – Plan de cadrage

Oct 19, 2026 · @Rayan Chentouf

Plateforme de revue de presse automatisée : collecte chaque matin les articles de 4 médias, les regroupe par événement et publie une édition structurée (titre, résumé, sources, liens).

**Périmètre v1**

- 4 sources : 2 nationales/internationales (flux RSS), 2 régionales (RSS ou sitemap)
- Pipeline quotidien orchestré par Airflow
- Stockage intermédiaire dans un datalake fichiers plats, partitionné par date
- Regroupement des articles par embeddings
- Base de données (SQL ou NoSQL, à trancher) alimentant un front

## Pipeline

### Étape 1 — Collecte

- Récupération des articles via flux RSS ou sitemap selon la source
- Un connecteur par source, interface commune : titre, chapô, URL, date, langue, source
- Dédoublonnage par URL
- Écriture en fichiers plats dans le datalake : `raw/{YYYY-MM-DD}/{source}.json`

### Étape 2 — Traitement (zone staging)

- Lecture des fichiers bruts du jour
- Nettoyage et normalisation : suppression des balises HTML, uniformisation des dates, langue détectée
- Sélection des attributs retenus pour le clustering (à préciser : titre seul, titre + chapô, entités, date)
- Calcul des embeddings multilingues sur les attributs retenus
- Écriture dans le datalake : `processed/{YYYY-MM-DD}/{source}.json`

### Étape 3 — Regroupement

- Chargement des articles traités du jour
- Clustering par similarité cosinus sur les embeddings (HDBSCAN ou agglomératif, seuil à calibrer)
- Affinage par entités nommées (lieux, dates, personnes) pour éviter de fusionner deux événements proches
- Groupes couvrant au moins 2 sources → sujets principaux ; articles isolés → « En bref »

### Étape 4 — Rédaction

- Pour chaque groupe : envoi des titres + chapôs au LLM
- Le LLM produit : titre de l'événement, résumé de 3–4 phrases, une ligne par article avec source
- Optionnel : attribution d'une rubrique (Politique, International, etc.) après coup

### Étape 5 — Continuité

- Comparaison des groupes du jour avec les événements des jours précédents (distance entre centroïdes)
- Si similarité > seuil : rattachement à l'événement existant (affichage « suite de… »)
- Sinon : création d'un nouvel événement

### Étape 6 — Publication

- Écriture en base (articles, sujets, événements)
- Génération de l'édition du jour côté front
- Notification Airflow en cas d'échec sur n'importe quelle étape

## Schéma d'architecture

&#91;embedded content: pipeline · 5 étapes + Airflow\]

Airflow orchestre l'ensemble : chaque étape est une tâche, les dépendances sont explicites, et toute erreur déclenche une alerte.

## Architecture technique

```
[Sources]
  RSS / Sitemap
       │
       ▼
[Airflow DAG quotidien]
       │
       ├─ 1. Collect     → Datalake raw/{date}/{source}.json
       ├─ 2. Process     → Datalake processed/{date}/{source}.json
       │                   (nettoyage + embeddings)
       ├─ 3. Cluster     → groupes d'articles (en mémoire)
       ├─ 4. Summarize   → LLM API (titres + résumés)
       ├─ 5. Link        → rattachement aux événements existants
       └─ 6. Store       → Base de données
                               │
                               ▼
                           [Front]
                       édition du jour
```

Airflow gère les dépendances entre étapes, les relances en cas d'échec partiel et les alertes.

## Choix techniques

**Orchestration : Airflow** Un DAG par pipeline quotidien. Chaque étape est une tâche Python. Airflow gère les dépendances, les relances, les alertes email/Slack sur échec et l'historique des exécutions.

**Datalake : fichiers plats** Stockage local ou objet (S3, MinIO). Partition par date : `raw/{YYYY-MM-DD}/` et `processed/{YYYY-MM-DD}/`. Format JSON ou Parquet. Permet de rejouer n'importe quelle étape sans re-collecter.

**Base de données : à trancher**

| Critère                                          | SQL (PostgreSQL)       | NoSQL (MongoDB)            |
| ------------------------------------------------ | ---------------------- | -------------------------- |
| Relations articles / sujets / événements         | Naturel                | Possible mais moins direct |
| Schéma flexible (attributs variables par source) | Migrations nécessaires | Plus souple                |
| Requêtes côté front (filtres, tri, pagination)   | Mature                 | Correct                    |
| Expérience équipe                                | —                      | —                          |

Recommandation : PostgreSQL si le schéma est stable dès le départ ; MongoDB si les sources ont des structures très hétérogènes.

**Embeddings** Modèle : `multilingual-e5-base` ou `paraphrase-multilingual-mpnet-base-v2` (multilingue, léger, tourne en local). Attributs à vectoriser : à trancher entre titre seul, titre + chapô, ou titre + entités. Le titre seul est plus rapide et souvent suffisant ; le chapô améliore la précision sur des sujets proches. Clustering : HDBSCAN (pas de nombre de groupes à fixer, gère les articles isolés).

**LLM pour les résumés** API externe (Claude, OpenAI) ou modèle local selon contrainte de coût et de confidentialité.

**Front** Application légère lisant la base. Peut être un simple site statique généré à chaque édition, ou une API + interface web selon le besoin.

## Schéma de données

**Datalake**

```
data/
├── raw/
│   └── 2026-10-19/
│       ├── lemonde.json
│       ├── ouest-france.json
│       └── ...
└── processed/
    └── 2026-10-19/
        ├── lemonde.json       # attributs normalisés + embedding
        └── ...
```

**Base de données**

```sql
sources       (id, nom, type, url_flux, langue)

articles      (id, source_id, url UNIQUE, titre, chapo,
               publie_le, langue, collecte_le)

evenements    (id, titre, premiere_apparition, derniere_apparition)

sujets        (id, edition_date, evenement_id, titre, resume,
               rubrique, rang)

sujet_articles(sujet_id, article_id, resume_ligne)

pipeline_runs (id, demarre_le, termine_le, statut,
               nb_articles, erreur)
```

Les embeddings ne sont pas stockés en base : ils sont calculés dans le datalake (zone `processed`) et utilisés en mémoire pendant le clustering, puis abandonnés.

## Périmètre

**Dans le scope**

- Collecte des flux RSS et sitemaps de 4 à 8 sources (PQR + PQN)
- Stockage brut en datalake, partitionné par date
- Nettoyage, normalisation, calcul d'embeddings
- Regroupement automatique des articles par événement (clustering)
- Publication d'une édition quotidienne à 7h
- Supervision du pipeline (alertes, historique des runs)

**Hors scope**

- Résumé des articles par LLM (fonctionnalité future)
- Scraping du texte intégral des articles
- Front-end élaboré — interface minimale suffisante
- Données personnelles (commentaires, profils lecteurs)
- Langues autres que le français et l'anglais

## Contraintes

- **Heure de publication** : édition disponible à 7h chaque matin
- **Fraîcheur** : seuls les articles publiés dans les 24h précédentes sont traités
- **Droits** : on ne conserve que titre, extrait, lien, date — jamais l'article entier ; respect des CGU de chaque source
- **Budget** : zéro — machines locales pour l'instant, infrastructure fournie au TD 4 ou 6
- **Données personnelles** : aucune collecte (pas de commentaires, pas de profils)

## Alternatives écartées

| Sujet                  | Option écartée          | Raison                                                                               | Option retenue          |
| ---------------------- | ----------------------- | ------------------------------------------------------------------------------------ | ----------------------- |
| Orchestration          | GitHub Actions          | Adapté à un seul script, pas à un pipeline multi-étapes avec dépendances et relances | Airflow                 |
| Collecte               | Scraping HTML           | Fragile, risque juridique, charge sur les serveurs                                   | Flux RSS / sitemap      |
| Regroupement           | TF-IDF                  | Ne gère pas le multilingue, ne comprend pas les reformulations                       | Embeddings + HDBSCAN    |
| Regroupement           | LDA                     | Produit des thèmes généraux, pas des événements précis ; mauvais sur textes courts   | Embeddings + HDBSCAN    |
| Stockage intermédiaire | Base de données directe | Impossible de rejouer une étape sans re-collecter                                    | Datalake fichiers plats |

## Risques

| Risque                                     | Impact                               | Mitigation                                                          |
| ------------------------------------------ | ------------------------------------ | ------------------------------------------------------------------- |
| Source sans flux RSS ni sitemap            | Collecte impossible sur cette source | Identifier en amont ; prévoir connecteur scraping léger en fallback |
| Dérive de format RSS (balises changeantes) | Champs manquants, pipeline en erreur | Validation du schéma à l'ingestion, alerte Airflow                  |
| Clustering trop agressif                   | Deux événements distincts fusionnés  | Affinage par entités nommées + seuil à calibrer sur données réelles |
| Latence LLM (si résumé activé plus tard)   | Edition publiée en retard            | Hors scope v1 ; anticiper en lançant le pipeline à 5h               |
| Droits voisins                             | Plateforme bloquée par un éditeur    | Ne conserver que le minimum légal dès le départ                     |

## Rôles dans l'équipe

| Rôle | Responsabilité                     |
| ---- | ---------------------------------- |
| ...  | Collecte et connecteurs sources    |
| ...  | Pipeline Airflow et orchestration  |
| ...  | Traitement, embeddings, clustering |
| ...  | Base de données et schéma          |
| ...  | Front et publication               |
