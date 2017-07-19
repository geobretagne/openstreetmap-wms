# Serveur WMS OpenStreetMap

Bbasé sur :

* [imposm3](https://github.com/omniscale/imposm3) by omniscale - Apache Licence 2.0
* [basemaps](https://github.com/mapserver/basemaps) provides mapfiles suitables for imposm3 databases
* [mapproxy](https://mapproxy.org/) "caches, accelerates and transforms data from existing map services"

et s'appuyant sur MapServer, Nginx, Postgresql, PostGIS.

Ce dépôt permet le chargement des données OpenStreetMap dans une base de données PostGIS au moyen d'imposm3; leur transformation en services WMS par MapServer; et leur mise en cache par mapproxy.

En enregistrant plusieurs fois les données avec une simplification de leur géométrie, imposm3 permet d'assurer des performances constantes à des résolutions différentes.

Mapserver+Basemaps construit à la demande des vues cartographiques à partir de la base de données. Ces vues sont proposées en différents styles ("google", "bing"...) qui eux aussi s'adaptent à la résolution demandée.

Enfin, mapproxy "proxifie" les vues proposées par mapproxy, fournit des points d'entrée aux standards WMS, WMTS, TMS, et peut mettre en cache les requêtes de façon à accélérer l'affichage.

## Installation

Installer les dépendances.

osm2pgsql ne sert à rien mais il installe l'environnement postgis.

```
aptitude install git cgi-mapserver nginx mapserver-bin fcgiwrap osm2pgsql
```

Récupérer le dépôt dans un volume performant (SSD) et gros (>100 Go)

```
git clone https://github.com/patin/couffin/osm2wms.git
```

Préparer la base de données

```
sudo su postgres
source env_test
createuser --no-superuser --no-createrole --createdb $DBUSER
createdb -E UTF8 -O "$DBUSER" "$DBNAME"
psql -d $DBNAME -c "CREATE EXTENSION postgis;"
psql -d $DBNAME -c "CREATE EXTENSION hstore;"
echo "ALTER USER $DBUSER WITH PASSWORD '$DBPASS';" |psql -d $DBNAME
```

## Configuration et build

Le déploiement est spécifié par des variables d'environnement.
Un fichier `env_*` est sourcé au démarrage de chaque script.
Pour créer un environnement, copiez l'exemple `env.sample`
et éditez la copie.

Par exemple

```
cp env.sample env_test
vi env_test
``` 

Le script `̀ `osm_install``` installe et construit basemaps et mapproxy.

```
./osm_install ./env_test
```

## Chargement

Le script ```osm_fetch``` récupère les pbf dont les adresses sont recensées
dans ```etc/pbf.list```.
Attention, le script télécharge bêtement tous les pbf à chaque lancement, sans reprise.
Cette tâche pourra être lancée périodiquement pour renouveller les données.

```
./osm_fetch ./env_test
```

Le script ```osm_write``` agrège les pbf et les charge dans postgis.
Sur un serveur OVH T32 SSD il prend 4h20 pour intégrer France, Outre-Mer et pays limitrophes. L'usage de ``̀`screen``` est recommandé.
Cette tâche pourra être lancée périodiquement pour renouveller les données après import des pbf.

Le chargement est effectué dans des tables temporaires qui ne seront
mises en production qu'à la fin. Il n'occasionne donc pas d'interruption
mais peut affecter les performances. En particulier, la phase d'optimisatoin est très consommatrices de ressources.

```
./osm_write ./env_test
```

## Backend WMS

Les mapfiles doivent être reliés à MapServer. On utilise pour cela Nginx et les exemples de configuration figurant dans ̀```etc/nginx``` :

* copier ```etc/nginx/``` dans `̀``/etc/nginx/includes``̀̀ 
* copier ```etc/nginx/mapserver``` dans ```/etc/nginx/sites-available```
* activer la configuration ```mapserver``` dans nginx et relancer celui-ci.

Pour vérifier que MapServer est correctement servi par Nginx :

``̀
curl "http://127.0.0.1/osm/default?SERVICE=WMS&REQUEST=GetCapabilities"
```

Pour vérifier que MapServer peut lire la base de données, on forge un appel ``̀ GetMap``` et on regarde le résultat sur la console :

`̀``
curl "http://127.0.0.1/osm/default?SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&LAYERS=default&SRS=EPSG:3857&BBOX=-5,45,5,55&FORMAT=image/png&WIDTH=400&HEIGHT=400" -o sample.png"
cacaview sample.png
``̀ 


Le script ```osm_serve``` lance un démon mapproxy en mode développement.
Se référer à la doc gunicorn/mapproxy pour passer en production.
https://mapproxy.org/docs/nightly/deployment.html

```
./osm_serve ./env_test
```


