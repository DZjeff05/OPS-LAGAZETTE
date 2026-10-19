# Gazette de presse – architecture de la plateforme

Oct 19, 2026 · @Rayan Chentouf

La gazette repose sur du **clustering d'actualités** (*topic detection*) : on ne range pas les articles dans des catégories fixées à l'avance, ce sont les **sujets du jour qui émergent** des articles eux-mêmes. Chaque matin, la plateforme publie un résumé par sujet, avec une ligne par article, sa source et le lien vers l'article complet.

## Le pipeline (chaque nuit, vers 5h)

Six étapes s'enchaînent chaque nuit, de la collecte des flux à la page publiée à 6h.

### 1. Collecte

- Utiliser les **flux RSS** des 4 médias plutôt que le scraping : plus stable, plus propre juridiquement, et ils fournissent déjà titre, chapô, lien et date.
- Récupérer tout ce qui a été publié depuis la veille, par exemple de 6h à 6h.
- Stocker chaque article avec : source, titre, chapô, URL, date et langue.
- Dédoublonner par URL et par titre quasi identique, car un même article apparaît souvent dans plusieurs flux.

### 2. Représentation des articles

- Calculer un **embedding multilingue** de titre + chapô, par exemple avec `multilingual-e5` ou `paraphrase-multilingual-mpnet`.
- Le multilingue est indispensable avec des médias internationaux : « manifestation du 17 octobre » et « October 17 protest » tombent au même endroit.
- En complément, extraire les **entités nommées** (lieux, personnes, dates) avec spaCy. Deux articles qui partagent « 17 octobre » et « Paris » sont très probablement liés.

### 3. Regroupement (le « chemin inverse »)

- Appliquer un clustering **sans nombre de groupes fixé à l'avance** : HDBSCAN, ou un clustering agglomératif avec un seuil de distance cosinus.
- Les groupes qui couvrent **au moins 2 sources** deviennent les « sujets du jour ».
- Les articles isolés partent dans une rubrique « En bref ».
- Classer les sujets par importance : nombre de sources, puis nombre d'articles.

### 4. Rédaction par un LLM

Pour chaque groupe, le LLM reçoit les titres et chapôs, et produit :

- un **titre d'événement**, par exemple « Manifestation du 17 octobre » ;
- un **résumé de 3–4 phrases** de « ce que dit la presse », en signalant les différences d'angle entre médias ;
- pour chaque article, **une ligne** de résumé, avec le nom de la source et le lien vers l'article complet.

Les grandes rubriques (Politique, International…) peuvent être attribuées **après coup** à chaque sujet : elles découlent des sujets, et non l'inverse.

### 5. Continuité dans le temps

- Comparer chaque sujet du jour aux sujets des jours précédents, via la distance entre centroïdes des groupes.
- Cela permet d'afficher « suite de… » et de construire une page par événement qui s'enrichit au fil des jours.

### 6. Publication

- Générer une page HTML statique, ou passer par un petit CMS, avec un cron à 6h.
- Ajouter éventuellement une newsletter.

## Point juridique

En France, les **droits voisins des éditeurs de presse** (loi de 2019) encadrent la reprise de leurs contenus. L'approche retenue va dans le bon sens : résumés courts rédigés par la plateforme, source citée, lien vers l'article. Précautions supplémentaires :

- ne pas reproduire les chapôs tels quels ;
- ne pas reprendre les images ;
- respecter les conditions d'utilisation des flux RSS.

Si la plateforme est publique ou commerciale, faire valider le cadre par un juriste : certains éditeurs exigent un accord.

## Questions ouvertes pour le prototype

- [ ] **Langage et stack** : Python, naturel pour cette partie ?
- [ ] **Les 4 médias** : lesquels, pour vérifier leurs flux RSS ?
- [ ] **LLM pour les résumés** : API (Claude, OpenAI) ou modèle local ?

Une fois ces points tranchés : prototype complet avec collecte, clustering, résumés et page HTML générée.
