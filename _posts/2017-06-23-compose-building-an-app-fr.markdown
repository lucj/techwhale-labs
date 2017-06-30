---
layout: post
title:  "Construction d'une application Compose"
author: "@lucjuggery"
tags: [FR]
categories: "Docker-Compose"
---

Dans ce lab, nous allons créer une application simple composée de plusieurs "micro-services".
Dans un premier temps, nous lancerons les éléments de cette application manuellement dans des containers mais sans utiliser Docker Compose.
Dans un second temps nous définirons l'application avec Docker Compose.

## L'application

L'application a pour but de simuler l'envoi de données par une device IoT ainsi que la réception / sauvegarde / visualisation de ces données.

Elle est composée des éléments suivants:
- un simulateur (qui simule une device IoT) qui génère des données aléatoires et les envoie via http
- un collecteur en charge de la réception et du stockage des données
- une base de données utilisée pour la persistance des données
- un dashboard de visualisation

### Le simulateur

Le code de ce composant est disponible sur le repository GitHub [lucj/iot-simulateur](https://github.com/lucj/iot-simulator.git)

C'est un simple script shell qui:
- récupère la date courante
- génère une valeur aléatoire qui simule une température
- envoie un payload au serveur dont l'URL et le port sont passés en paramêtres

```
#!/bin/sh

# Usage
function usage {
    echo "Usage: simulator.sh -h HOST -p PORT"
    exit 1
}

# Parse arguments
while getopts h:p: FLAG; do
  case $FLAG in
    h)
      HOST=$OPTARG
      ;;
    p)
      PORT=$OPTARG
      ;;
    \?)
      usage
      ;;
  esac
done

# Make sure HOST and PORT are specified
if [ -z "$HOST" -o -z "$PORT" ]; then
   usage
fi

# Generate and send random data
while(true); do
    # Current date
    d=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

    # Random temperature between 20 and 34°C
    temp=$(( ( RANDOM % 15 )  + 20 ))

    # Send data to specified HOST:PORT
    data='{"ts":"'$d'", "type": "temp", "value": '$temp', "sensor_id": 123 }'

    curl -XPOST -H "Content-Type: application/json" -d "$data" http://$HOST:$PORT/data

    sleep 1
done
```

Le payload envoyé a le format suivant

```
{
  "ts":"2017-06-25T09:53:16Z",
  "type": "temp",
  "value": 34,
  "sensor_id": 123
}
```

Il représente le format de données que pourrait envoyer une device IoT, identifiée par son id (`sensor_id`).

### Le collecteur de données

Ce composant est développé avec la plateforme `Node.js`. Le code est disponible sur le repository GitHub [lucj/iot-collector](https://github.com/lucj/iot-collector.git)

Son rôle principal est d'exposer un endpoint en HTTP POST sur `/data` sur lequel les données de température, envoyées par le simulateur, sont reçues. Le collecteur est également en charge de sauvegarder les données dans une base `InfluxDB`. Le morceau de code ci-dessous illustre cette fonctionnalité.

```
...
app.post('/data',
         function(req, res, next){
             influx.writePoints([
                 {
                  measurement: 'data',
                  tags: { type: req.body.type },
                  fields: { sensor_id: req.body.sensor_id, value: req.body.value },
                  timestamp: new Date(req.body.ts).getTime() * 1000000
                 }
             ]).then(() => {
               winston.info(req.body);
               return res.sendStatus(201);
             })
             .catch( err => {
               winston.error(err.message);
               return res.sendStatus(500);
             });
         });
...
```

### La base de données

[InfluxDB](https://www.influxdata.com/time-series-platform/influxdb/) est la base de données utilisée par le collecteur pour sauvegarder les données reçues. C'est une base très performante pour le sauvegarde de données temporelles.

### Le dashboard

Afin de visualiser les données, nous utilisons [Grafana](https://grafana.com/) un outils qui s'intègre très facilement sur différentes sources de données (Prometheus, InfluxDB, ...) et qui permet de créer des dashboards de très bonne qualité.

## Lancement de l'application

Dans un premier temps, nous allons lancer les différents services de l'application manuellement dans des containers. Afin que les services puissent communiquer entre eux, nous allons créer un réseau bridge (`user-defined bridge`) et attacher chacun des containers à ce réseau.

### Création du network

La commande suivante crée un réseau de type bridge que nous appelons `mynet`.

```.term1
docker network create mynet
```

Note: nous verrons les networks en details dans un labs dédié. Pour le moment nous pouvons considérer un réseau comme un moyen d'isoler un groupe de containers leur permettant de communiquer entre eux.

### Lancement des services

Nous allons maintenant lancer un à un les composants de notre application dans des containers.

Chacun des containers:
- sera lancé en background
- sera attaché au network `mynet` crée précédemment
- aura un nom dédié

#### La base de données

Nous utilisons l'image officielle de `InfluxDB` disponible dans le [Docker Hub](https://hub.docker.com/_/influxdb/). La commande suivante permet de lancer un container basé sur `InfluxDB`. Nous le nommons `db`.

```.term1
docker container run -p 8086:8086 --net mynet --name db -d influxdb:1.3
```

Une fois le container lancé, nous créons la base de données `iot` dont nous aurons besoin plus tard.

```.term1
curl -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE iot"
```

Nous pouvons vérifier que la base est bien créée avec la commande suivante.

```.term1
curl -XPOST http://localhost:8086/query --data-urlencode "q=SHOW DATABASES" | jq
```

```
{
  "results": [
    {
      "statement_id": 0,
      "series": [
        {
          "name": "databases",
          "columns": [
            "name"
          ],
          "values": [
            [
              "iot"
            ],
            [
              "_internal"
            ]
          ]
        }
      ]
    }
  ]
}
```

#### Le collecteur

La commande suivante lance le collecteur dans un container basé sur l'image `lucj/iot-collector:1.0`. Nous appelons ce container `collector`.

```.term1
docker container run --net mynet --name collector -d lucj/iot-collector:1.0
```

Nous pouvons voir les logs du collector avec la commande suivante.

```.term1
docker container logs collector
```

A ce stade, nous pouvons seulement voir que le serveur web est lancé. Il est en attente de requêtes HTTP.

```
> iot@1.0.0 start /app
> node index.js

info: server listening on port 1337
```

#### Le simulateur 

La commande suivante lance le simulateur dans un container basé sur l'image `lucj/iot-simulator:1.0`. Nous précisons les options -h et -p avec l'URL et le port du collector afin que le simulateur puisse envoyer les données sur le container que nous avons lancé précédemment. Nous nommons ce container `simulator`.

```.term1
docker container run --net mynet --name simulator -d lucj/iot-simulator -h collector -p 1337
```

#### Le dashboard

La commande suivante lance Grafana, qui nous permettra de visualiser les données reçues. Nous publions en même temps le port 3000 exposé par le container sur le port 3000 de la machine hôte afin de manipuler l'interface depuis l'extérieur. Nous appelons ce container `dashboard`.

```.term1
docker container run -p 3000:3000 --net mynet --name dashboard -d grafana/grafana:4.3.2
```

Nous pouvons alors acceder à [l'interface de Grafana](/){:data-term=".term1"}{:data-port="3000"}

Le login et le password par défaut est `admin`.

![Grafana login]({{ site.baseurl }}/images/grafana-login.png)

Nous commençons par ajouter une data source. Nous sélectionnons le backend `InfluxDB` et nous utilisons directement utiliser l'url `http://db:8086` car un container attaché au réseau mynet peut communiquer avec les autres containers du réseau en utilisant leur nom. La résolution du nom `db` est effectué par le daemon Docker.

![Grafana data source]({{ site.baseurl }}/images/grafana-data-source.png)

Nous pouvons ensuite créer un dashboard (très simple) pour la visualisation des données. Dans cet exemple nous avons en plus spécifié un rafraichissement automatique toutes les 5 secondes afin de voir l'évolution de la courbe en quasi temps réel car le simulateur envoie des données en continu.

![Grafana dashboard]({{ site.baseurl }}/images/grafana-dashboard.png)

## Définition de l'application dans le format Compose

L'approche que nous avons utilisée ci-dessus peut devenir relativement compliquée et sujette à des erreurs dans le cas d'une application composée d'un grand nombre de services. Nous allons voir ici comment définir l'application en utilisant le format Docker Compose.

Nous créons un fichier nommé docker-compose.yml contenant nos services.

```.term1
cat<<EOF>docker-compose.yml
version: "3"
services:
  db:
    image: influxdb:1.3
    container_name: influx
    restart: on-failure
  dashboard:
    image: grafana/grafana:4.3.2
    ports:
      - 3000:3000
    restart: on-failure
  simulator:
    image: lucj/iot-simulator:1.0
    command: -h collector -p 1337
    restart: on-failure
  collector:
    image: lucj/iot-collector:1.0
    restart: on-failure
EOF
```
  
Pour chaque service, nous avons ajouté l'option `restart: on-failure` de façon à ce qu'un container soit relancé en cas de défaillance.

La commande suivante lance l'application en background.

```.term1
docker-compose up -d
```

Nous pouvons alors lister les containers de l'application: il y a bien un container par service.

```.term1
docker-compose ps
```

```
      Name                    Command               State            Ports
-----------------------------------------------------------------------------------
root_collector_1   npm start                        Up       1337/tcp
root_dashboard_1   /run.sh                          Up       0.0.0.0:3000->3000/tcp
root_db_1          /entrypoint.sh influxd           Up       8086/tcp
root_simulator_1   /app/simulator.sh -h colle ...   Up
```

Un fois que les services de notre application sont lancée, il faut créer la base de données afin que le collector puisse persister les données et que le dashboard puisse les lire. Pour cela nous lancer un processus dans le containe, que nous avons nommé `influx`, du service `db`.

```.term1
docker exec influx curl -XPOST http://db:8086/query --data-urlencode "q=CREATE DATABASE iot"
```

Note: nous avons spécifié l'option `containe_name` pour l'unique container du service db. Spécifier un nom de container pour un service ne permettra pas de scaler ce service (créer plusieurs containers) par la suite.

```
{"results":[{"statement_id":0}]}
```

Nous pouvons alors acceder à [l'interface de Grafana](/){:data-term=".term1"}{:data-port="3000"} et effectuer la même configuration que précédemment.

Note: lorque nous avons lancé les containers manuellement, nous avons crée un réseau et attaché chacun des containers à celui-ci. Lorque l'on utilise Docker Compose, un réseau de type `user-defined bridge` est crée automatiquement et l'ensemble des containers y sont attachés ce qui permet la communication entre les containers via le nom du service.

La commande suivante permet d'arrêter l'application et de supprimer les différents composants.

```.term1
docker-compose down
```

### Utilisation d'un volume

Afin de persister les donneées à l'extérieur du container `db`, nous définissons le volume nommé `data` dans le fichier docker-compose.yml. Nous faisons un mount de ce volume sur le réperoire `/var/lib/influxdb` du container car c'est dans ce répertoire que Influx conserve les données.

Nous modifions alors le docker-compose.yml afin qu'il contienne la structure suivante.

```
version: "3"
services:
  db:
    image: influxdb
    container_name: influx
    volumes:
      - data:/var/lib/influxdb
    restart: on-failure
  dashboard:
    image: grafana/grafana
    ports:
      - 3000:3000
  simulator:
    image: lucj/iot-simulator
    command: -h collector -p 1337
  collector:
    image: lucj/iot-collector
volumes:
  data:
```

Nous lançons l'application avec la commande `up`.

```.term1
docker-compose up -d
```

La permière que nous lançons l'application, nous aurons besoin de créer la base `iot` comme nous l'avons fait précédemment.

```.term1
docker exec influx curl -XPOST http://db:8086/query --data-urlencode "q=CREATE DATABASE iot"
```

Si nous listons les volumes, nous pouvons voir que le volume data a été crée.

```.term1
docker volume ls | grep data
```

```
DRIVER              VOLUME NAME
...
local               root_data
```

Note: par défaut, le nom de chaque composants d'une application Compose (networks, volumes, containers) est préfixé par le nom du répertoire courant.

Nous pouvons alors inspecter le volume et récupérer son path sur la machine hôte.

```.term1
docker volume inspect root_data
```

```
[
    {
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "root",
            "com.docker.compose.volume": "data"
        },
        "Mountpoint": "/graph/volumes/root_data/_data",
        "Name": "root_data",
        "Options": {},
        "Scope": "local"
    }
]
```

Les données générées lors du lancement du service `db` sont stockées dans ce volume.

```.term1
find /graph/volumes/root_data/_data
```

```
/graph/volumes/root_data/_data
/graph/volumes/root_data/_data/meta
/graph/volumes/root_data/_data/meta/meta.db
/graph/volumes/root_data/_data/data
/graph/volumes/root_data/_data/data/_internal
/graph/volumes/root_data/_data/data/_internal/monitor
/graph/volumes/root_data/_data/data/_internal/monitor/1
/graph/volumes/root_data/_data/data/iot
/graph/volumes/root_data/_data/data/iot/autogen
/graph/volumes/root_data/_data/data/iot/autogen/2
/graph/volumes/root_data/_data/wal
/graph/volumes/root_data/_data/wal/_internal
/graph/volumes/root_data/_data/wal/_internal/monitor
/graph/volumes/root_data/_data/wal/_internal/monitor/1
/graph/volumes/root_data/_data/wal/_internal/monitor/1/_00001.wal
/graph/volumes/root_data/_data/wal/iot
/graph/volumes/root_data/_data/wal/iot/autogen
/graph/volumes/root_data/_data/wal/iot/autogen/2
/graph/volumes/root_data/_data/wal/iot/autogen/2/_00001.wal
```

Nous retrouvons des références à la base de données `iot` que nous avons créée.
