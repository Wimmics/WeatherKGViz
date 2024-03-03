# DemoKGViz-2

## Projet de fusion des projets Angular et VueJS

Ce projet vise à migrer le projet DemoKGViz-1 existant vers le framework VueJS, afin de le fusionner au projet VueJS existant Le but ultime est d'avoir un projet beaucoup plus maintenable qui n'utilise qu'un seul framework.

## La Map

Le focus est la conversion de la map Angular vers VueJS, la map a été maintenant intégrée à l'ancien projet VueJS, Voici les fonctionnalités et infos à savoir :

- La Map affiche la température moyenne de chaque région, on peut sélectionner l'année que l'on souhaite.
- En appuyant sur une région, un popup affichera l'évolution de la température de la région sélectionnée à partir de chaque station dans la région.
- On peut délimiter la date de notre analyse en utilisant le slider pour un visuel plus concis.
- Dans la section station, on peut choisir la mesure à analyser (température, précipitations, humidité, vitesse du vent et direction du vent).
- La légende montre les mesures les plus importantes
- On peut choisir la station à analyser en appuyant dessus.
- On peut supprimer toutes les stations choisies d'un clic.

## Repository files

Le dossier src est organisé comme suit :

- assets: contient la représentation en région de la France
- component : Ensemble des components permettant d'afficher la bar latéralles et les différents graphiques
- config: contient la description de tous les paramètres affichables
- img: les images
- queries: contient l'ensemble des queries SPARQL
- store: continent les différents stores permettant nottament de gérer les appel à la base de données
- utils: fonctions de traitement supplémentaires
- App.vue: fichier paramètre de Vue
- main.js : fichier paramètre de javascript

## Déploiement du site

### Compilation

Tout d'abord, il faut se rendre sur la branche main du projet.

Ensuite, il faut effectuer la commande permettant de compiler le projet ci-dessous

```
npm run build
```

Une fois que la compilation est terminée, on peut voir qu'un nouveau dossier appelé dist est apparu. A l'intérieur se trouve les fichiers notre site Internet.

### Github Pages

#### Mettre le site sur le dépôt

Après avoir compilé le projet, se rendre sur la page du dépôt et sélectionner la branche de déploiement `deploy`

Ensuite, cliquer sur `Add File`, puis `Upload File`.

Pour finir, glisser l'intégralité des fichiers présents dans le dossier `/dist` à l'intérieur et cliquer sur `Commit changes`.

#### Créer la page GitHub

La prochaine étape est de déployer le site en ligne. Pour cela, il faut se rendre sur le dépôt, puis dans les réglages du dépôt.

Ensuite, se rendre dans la catégorie `Pages`.

Puis, cliquer sur `Source` et sélectionner `Deploy from a branch`.

Il faut sélectionner la branche qui contient notre site, qui est la branche `deploy`.

Pour finir, il reste à appuyer sur `Save` pour mettre en ligne le site. On peut voir l'adresse à laquelle le site sera disponible.
