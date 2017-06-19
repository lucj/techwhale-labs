---
layout: post
title:  "Vue détaillée"
author: "@lucjuggery"
tags: [FR]
categories: Volumes
---

Dans ce lab, nous allons illustrer la notion de volume. Nous verrons notamment comment définir un volume:
- dans un Dockerfile
- au lancement d'un container en utilisant l'option `-v`
- en utilisant la ligne de commande

Nous verrons également comment monter un répertoire de l'hôte dans un container en utilisant un `bind-mount`.

## Persistance des données dans un container

Nous allons illustrer pourquoi, par défaut, un container ne doit pas être utilisé pour persister des données.

Nous lançons un shell intéractif dans un container basé sur l'image `alpine`, nous l'appelerons `c1`.

```.term1
docker container run --name c1 -ti alpine sh
```

Dans ce container, nous créons le répertoire `/data` et le fichier `hello.txt` dans celui-ci.

```.term1
mkdir /data && cd /data && touch hello.txt
```

Nous pouvons alors sortir du container `c1`.

```.term1
exit
```

Lors de la création du container, une layer read-write est ajoutée au dessus des layers read-only de l'image sous jacente. C'est dasn cette layer que les changements que nous avons apportés dans le container ont étés persistés (création du répertoire `/data` et du fichier `hello.txt`). Nous allons voir comment cette layer est accessible depuis la machine hôte (celle sur laquelle tourne le daemon Docker) et vérifier que nos modifications sont bien présentes.

Nous utilisons la commande `inspect` afin d'obtenir le path, sur l'hôte, de la layer du container `c1`. La clé qui nous intéresse est `GraphDriver`. 

```.term1
docker container inspect c1
```
Nous pouvons scroller dans l'output de la commande suivante jusqu'à la clé `GraphDriver` ou bien nous pouvons utiliser le format Go templates et obtenir directement le contenu de la clé.

```.term1
docker container inspect -f "{{ "{{ json .GraphDriver "}}}}" c1 | python -m json.tool
```

```
{
    "Data": {
        "LowerDir": "/graph/overlay2/55922a6b646ba6681c5eca253a19e90270e3872329a239a82877b2f8c505c9a2-init/diff:/graph/overlay2/30474f5fc34277d1d9e5ed5b48e2fb979eee9805a61a0b2c4bf33b766ba65a16/diff",
        "MergedDir": "/graph/overlay2/55922a6b646ba6681c5eca253a19e90270e3872329a239a82877b2f8c505c9a2/merged",
        "UpperDir": "/graph/overlay2/55922a6b646ba6681c5eca253a19e90270e3872329a239a82877b2f8c505c9a2/diff",
        "WorkDir": "/graph/overlay2/55922a6b646ba6681c5eca253a19e90270e3872329a239a82877b2f8c505c9a2/work"
    },
    "Name": "overlay2"
}
```

Depuis la machine hôte, si nous allons dans le répertoire spécifié dans `UpperDir`, nous pouvons voir que le répertoire `/data`  et le fichier `hello.txt` sont présents.

```.term1
CONTAINER_LAYER_PATH=$(docker container inspect -f "{{ "{{ json .GraphDriver.Data.UpperDir "}}}}" c1 | tr -d '"') 
find $CONTAINER_LAYER_PATH -name hello.txt
```

Si nous supprimons le container `c1`, le répertoire spécifié par `UpperDir` n'existe plus.

```.term1
docker container rm c1
```

```
find $CONTAINER_LAYER_PATH -type f
find: /graph/overlay2/ee4d...8b01/diff: No such file or directory
```

Cela montre que les données créées dans le container ne sont pas persistées et sont supprimées avec le container.

## Définition d'un volume dans un Dockerfile

Nous allons maintenant voir comment les volumes sont utilisés pour permetttre de remédier à ce problème et permettre de persister des données en dehors d'un container.

Nous allons commencer par créer un Dockerfile basé sur l'image `alpine` et définir `/data` en tant que volume. Tous les élements créés dans `/data` seront persistés en dehors de l'union filesystem comme nous allons le voir.

Le Dockerfile contient les 2 instructions suivantes:

```
FROM alpine
VOLUME ["/data"]
```

```.term1
cat<<EOF>Dockerfile
FROM alpine
VOLUME ["/data"]
EOF
```

Nous construisons une image, que nous nommons `imgvol`, en utilisant ce fichier.

```.term1
docker image build -t imgvol .
```

La commande suivante permet de lancer un shell intéractif dans un container, nommé `c2`, basé sur l'image `imgvol`.

```.term1
docker container run --name c2 -ti imgvol
```

Depuis le container, nous créons le fichier `hello.txt` dans le répertoire `/data`.

```.term1
cd /data
touch hello.txt
ls
```

Nous sortons alors du container avec la commande `CTRL-P / CTRL-Q` afin de s'assurer que le container est toujours en exécution. Pour en être sur, `c2` doit être listé dans la commande suivante.

```.term1
docker container ls
```

Avec la commande `inspect`, nous récuperons la clé `Mounts` afin d'avoir le chemin d'accès du volume sur la machine hôte.

```.term1
docker container inspect -f "{{ "{{ json .Mounts "}}}}"  c2 | python -m json.tool
```

Nous obtenons un résultat similaire à celui ci-dessous (aux ID prêts).

```
[
    {
        "Destination": "/data",
        "Driver": "local",
        "Mode": "",
        "Name": "2f5b7c6b77494934293fc7a09198dd3c20406f05272121728632a4aab545401c",
        "Propagation": "",
        "RW": true,
        "Source": "/graph/volumes/2f5b7c6b77494934293fc7a09198dd3c20406f05272121728632a4aab545401c/_data",
        "Type": "volume"
    }
]
```

Le volume `/data` est accessible, sur la machine hôte, dans le path spécifié par la clé `Source`.

La commande suivante permet de vérifier que le fichier `hello.txt` est bien présent.

```.term1
VOLUME_PATH=$(docker container inspect -f "{{ "{{ (index .Mounts 0).Source "}}}}" c2)
find $VOLUME_PATH -name hello.txt
```

```
/graph/volumes/cb5...f49/_data/hello.txt
```

Nous supprimons maintenant le container `c2`.

```.term1
docker container stop c2 && docker container rm c2
```

La commande suivante permet de vérifier que le fichier `hello.txt` existe tojours sur le filesystem de l'hôte.

```.term1
find $VOLUME_PATH -name hello.txt
```

```
/graph/volumes/cb5...f49/_data/hello.txt
```

Cet exemple nous montre qu'un volume permet de persister les données en dehors de l'union filesystem et ceci indépendemment du cycle de vie d'un container.

## Définition d'un volume au lancement d'un container

Précédemment nous avons défini un volume dans le Dockerfile, nous allons maintenant voir comment définir des volumes à l'aide de l'option `-v` au lancement d'un container.

Nous lançons un container avec les caractéristiques suivantes:
- basé sur l'image `alpine`
- nommé `c3`
- exécution en background (option `-d`)
- définition de `/data` en tant que volume (option `-v`)
- spécification d'une commande qui écrit dans le volume ci-dessus 

```.term1
docker container run --name c3 -d -v /data alpine sh -c 'ping 8.8.8.8 > /data/ping.txt'
```

Nous inspectons le container et repérons notamment le chemin d'accès du volume sur la machine hôte.

```.term1
docker inspect -f "{{ "{{ json .Mounts "}}}}" c3 | python -m json.tool
```

```
[
  {
    "Type": "volume",
    "Name": "af621cde2717307e5bf91be850c5a00474d58b8cdc8d6e37f2e373631c2f1331",
    "Source": "/graph/volumes/af621cde2717307e5bf91be850c5a00474d58b8cdc8d6e37f2e373631c2f1331/_data",
    "Destination": "/data",
    "Driver": "local",
    "Mode": "",
    "RW": true,
    "Propagation": ""
  }
]
```

Le volume est accessible via le filesystem de la machine hôte dans le path spécifié par la clé `Source`.

```.term1
VOLUME_PATH=$(docker container inspect -f "{{ "{{ (index .Mounts 0).Source "}}}}" c3)
tail -f $VOLUME_PATH/ping.txt
```

```
64 bytes from 8.8.8.8: seq=34 ttl=37 time=0.462 ms
64 bytes from 8.8.8.8: seq=35 ttl=37 time=0.436 ms
64 bytes from 8.8.8.8: seq=36 ttl=37 time=0.512 ms
64 bytes from 8.8.8.8: seq=37 ttl=37 time=0.487 ms
64 bytes from 8.8.8.8: seq=38 ttl=37 time=0.409 ms
64 bytes from 8.8.8.8: seq=39 ttl=37 time=0.438 ms
64 bytes from 8.8.8.8: seq=40 ttl=37 time=0.477 ms
...
```

Le fichier `ping.txt` est mis à jour régulièrement par la commande `ping` qui tourne dans le container.

Si nous stoppons et supprimons le container, le fichier `ping.txt` sera toujours disponible via le volume, cependant il ne sera plus mis à jour.

## Utilisation des volumes via la CLI

Les commandes relatives aux volumes ont été introduites dans Docker 1.9. Elles permettent de manager le cycle de vie des volumes de manière très simple.

La commande suivante liste l'ensemble des commandes disponibles.

```.term1
docker volume --help
```

La commande `create` permet de créer un nouveau volume. Nous créons ci-dessous un volume nommé `html`.

```.term1
docker volume create --name html
```

La commande `ls` permet de lister les volumes existants, `html` est donc présent dans cette liste.

```.term1
docker volume ls
```

```
DRIVER              VOLUME NAME
local               html
```

Comme pour les containers et les images (et d'autres primitives Docker), ... la commande `inspect` permet d'avoir la vue détaillée d'un volume.

```.term1
docker volume inspect html
```

```
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/graph/volumes/html/_data",
        "Name": "html",
        "Options": {},
        "Scope": "local"
    }
]
```

La clé `Mountpoint` définie ici correspond au chemin d'accès de ce volume sur la machine hôte. Lorsque l'on crée un volume via la CLI, le path contient le nom du volume et non pas un identifiant comme nous l'avons vu plus haut.

Dans l'exemple qui suit, nous allons monter le volume `html` sur le point de montage `/usr/share/nginx/html` d'un container basé sur l'image `nginx`.

Note: /usr/share/nginx/html est le répertoire servi par défaut par nginx, il contient notamment le fichier `index.html`.

```.term1
docker run --name www -d -p 8080:80 -v html:/usr/share/nginx/html nginx
```

Note: nous utilisons l'option `-p` pour mapper le port 80 du container nginx sur le port 8080 de l'hôte.

Depuis l'hôte, regardons le contenu du volume `html`.

```.term1
ls -al /graph/volumes/html/_data
```

```
total 8
drwxr-xr-x    2 root     root            40 Jun 19 11:14 .
drwxr-xr-x    3 root     root            19 Jun 19 11:14 ..
-rw-r--r--    1 root     root           537 May 30 13:03 50x.html
-rw-r--r--    1 root     root           612 May 30 13:03 index.html
```

Le contenu du répertoire `/usr/share/nginx/html` du container a été copié dans le répertoire `/graph/volumes/html/_data` de l'hôte.

Nous pouvons alors accéder à la [page d'accueil](#){:data-term=".term1"}{:data-port="8080"}

Depuis l'hôte, nous pouvons modifier le fichier `index.html` et vérifier que les changements sont pris en compte dans le container.

```.term1
cat<<END >/graph/volumes/html/_data/index.html
SOMEONE HERE ?
END
```

Si nous retournons sur la [page d'accueil](#){:data-term=".term1"}{:data-port="8080"}, nous pouvons voir les modifications effectuées.

## Bind-mount

L'opération de `bind-mount` consiste à monter un répertoire de la machine hôte dans un répertoire du container. Nous utilisons pour cela l'option `-v` lors du lancement d'un container mais nous spécifions 2 chemins d'accès séparés par ':':
- le premier indique un répertoire de l'hôte
- le second indique un répertoire du container

```
docker container run -v HOST_PATH:CONTAINER_PATH [OPTIONS] IMAGE [CMD]
```

Note: HOST_PATH et CONTAINER_PATH peuvent représenter un répertoire ou bien un fichier.

Il y a plusieurs cas à considérer:
- CONTAINER_PATH existe
- CONTAINER_PATH n'existe pas

### Premier cas

Nous lançons un shell intéractif dans un container basé sur l'image `alpine` et nous spécifions un bind-mount entre le répertoire `/bin` de la machine hôte et le répertoire `/hostbin` du container.

```.term1
docker container run -ti -v /bin:/hostbin alpine sh
```

Par défaut, il n'y a pas de répertoire `/hostbin` dans un container `alpine`. Quel est l'effet du `bind-mount` dans ce cas ?

```.term1
ls /hostbin
```
```
ash            fsync          ping
base64         getopt         ping6
bash           grep           pipe_progress
bashbug        gunzip         printenv
bbconfig       gzip           ps
busybox        hostname       pwd
cat            ionice         reformime
catv           iostat         rev
chgrp          ipcalc         rm
chmod          kbd_mode       rmdir
chown          kill           run-parts
conspy         ln             sed
cp             login          setserial
cpio           ls             sh
date           lsblk          sleep
dd             lzop           stat
df             makemime       stty
dmesg          mkdir          su
dnsdomainname  mknod          sync
dumpkmap       mktemp         tar
echo           more           touch
ed             mount          true
egrep          mountpoint     umount
false          mpstat         uname
fatattr        mv             usleep
fdflush        netstat        watch
fgrep          nice           wdctl
findmnt        pidof          zcat
```


Nous pouvons voir que le répertoire /hostbin est accessible dans le container, et qu'il contient l'ensemble des données du réperoitre `/bin` de la machine hôte. Si nous modifions un fichier dans ce répertoire depuis l'hôte, les modifications seront visibles dans le container et vice-et-versa.

Note: pour empêcher qu'un container compromis ne puisse modifier des fichiers de l'hôte, nous pouvons monter un répertoire en `read-only`. 

### Second cas

Nous lançons un shell intéractif dans un container basé sur l'image `nginx`et nous spécifions un bind-mount entre le répertoire `/tmp` de la machine hôte et le répertoire `/usr/share/nginx/html` du container.

```.term1
docker container run -ti -v /tmp:/usr/share/nginx/html nginx bash
```

Nous pouvons observer que le répertoire `/usr/share/nginx/html` du container est masqué par le contenu du répertoire `/tmp` de la machine hôte.

```.term1
ls /usr/share/nginx/html
```

Le bind-mount est très utile notamment dans un contexte de développement car cela permet d'utiliser le code source disponible sur la machine local depuis un container.

Note: depuis la version 17.05, la syntax du bind-mount a évoluée de façon à se rapprocher de la syntaxe utilisée lors de la création de services. L'option `-v HOST_PATH:CONTAINER_PATH` est toujours disponible et ne sera pas dépréciée avant un certain temps. La nouvelle syntax utilise l'option `--mount type=bind,src=HOST_PATH,dst=CONTAINER_PATH`.

Nous pouvons réécrire la commande précédente par:

```.term1
docker container run -ti --mount type=bind,src=/tmp,dst=/usr/share/nginx/html nginx bash
```

De la même façon, le contenu du répertoire `/usr/share/nginx/html` du container est masqué par le contenu de `/tmp`.

```.term1
ls /usr/share/nginx/html
```
