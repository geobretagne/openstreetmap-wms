Serveur WMS OpenStreetMap
basé sur :
* imposm3 https://github.com/omniscale/imposm3 by omniscale - Apache Licence 2.0
* basemaps https://github.com/mapserver/basemaps provides mapfiles suitables for imposm3 databases
* mapproxy https://mapproxy.org/ "caches, accelerates and transforms data from existing map services"



Installer les dépendances.
osm2pgsql ne sert à rien mais il installe l'environnement postgis.

```
aptitude install git cgi-mapserver nginx mapserver-bin fcgiwrap osm2pgsql
```

Récupérer le dépôt dans un volume performant (SSD) et gros (>100 Go)

```
git clone https://github.com/patin/couffin/osm2wms.git
```

Créer un environnement pour ce déploiement. Ici, pour un environnement de test :

```
# first copy the env template
cp env_template env_test
vi env_test
[change things]
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

Le script suivant installe et construit basemaps et mapproxy.
Voir dans etc/nginx comment servir les mapfiles.

```
./osm_install ./env_test
```

Le script suivant récupère les pbf répertoriés dans etc/pbf.list.
Attention le script télécharge bêtement tous les pbf à chaque lancement, sans reprise.
Cette tâche pourra être lancée périodiquement pour renouveller les données.

```
./osm_fetch ./env_test
```

Le script suivant agrège les pbf et les charge dans postgis.
Sur un serveur OVH T32 SSD, 4h20 pour intégrer France, Outre-Mer et pays limitrophes.
En interactif il est fortement recommandé de lancer ceci dans un screen.
Cette tâche pourra être lancée périodiquement pour renouveller les données.

```
./osm_write ./env_test
```

Le script suivant lance un démon mapproxy en mode développement.
Se référer à la doc gunicorn/mapproxy pour passer en production.
https://mapproxy.org/docs/nightly/deployment.html

```
./osm_serve ./env_test
```


