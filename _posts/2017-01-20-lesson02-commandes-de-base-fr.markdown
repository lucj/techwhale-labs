---
layout: post
title:  "Les commandes de base"
date:   2017-01-20 12:00:00 +0100
author: "@lucjuggery"
tags: [FR]
categories: Containers
---
## Au programme

Dans ce premier lab, vous allez manipuler les commandes de bases utilisées pour la gestion des containers.

Note: dans chaque lab, lorsque vous verez une commande dans un rectangle avec un fond gris, vous aurez juste besoin de clicker dessus afin qu'elle soit exécutée. Vous pouvez bien sur l'écrire vous même dans le terminal si vous le souhaitez.

La première commande retourne la version du client et celle du daemon

```.term1
docker version
```

Vous pouvez voir ci-dessous le résultat de cette commande lorsqu'elle est exécutée sur masOS. Le client est exécuté directement sur la machine locale, le serveur (ou Docker daemon) tourne dans une machine virtuelle Linux dans xhyve. C'est la raison pour laquelle OS/Arch est différent pour le client et le serveur. Nous pourrions aussi voir cette différence pour une utilisation sous Windows. Exécutée sur Linux, les client et daemon auront le même OS.

```
Client:
 Version:      17.06.0-ce-rc2
 API version:  1.30
 Go version:   go1.8.1
 Git commit:   402dd4a
 Built:        Wed Jun  7 10:02:52 2017
 OS/Arch:      darwin/amd64

Server:
 Version:      17.06.0-ce-rc2
 API version:  1.30 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   402dd4a
 Built:        Wed Jun  7 10:02:04 2017
 OS/Arch:      linux/amd64
 Experimental: true
```

## Lancement de containers

### Serveur web

Lançons un container basé sur l'image `nginx`.

```.term1
docker container run nginx
```

Vous devriez obtenir le résultat suivant:

```
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
ff3d52d8f55f: Pull complete
226f4ec56ba3: Pull complete
53d7dd52b97d: Pull complete
Digest: sha256:41ad9967ea448d7c2b203c699b429abe1ed5af331cd92533900c6d7749
0e0268
Status: Downloaded newer image for nginx:latest
...
```

L'image n'est pas trouvée en locale et est donc récupérée (téléchargée) depuis le Docker Hub (registry par défaut).
Le serveur web tourne en foreground (le terminal ne nous rend pas la main) mais il n'est pas possible de lui envoyer des requêtes car nous n'avons pas publié les ports sur la machine hôte.


Arrêtez le container en faisant un `CTRL-C` dans le terminal.

Nous allons utiliser des options supplémentaires afin:
- de lancer le serveur web en tâche de fond (option -d)
- de publier, sur le port 8080 de l'hôte, le port 80 exposé dans l'image `nginx`  (option -p 8080:80)

```.term1
docker container run -d -p 8080:80 nginx
```

L'identifiant du container est retourné. Le serveur web est disponible sur le port `8080` de la machine hôte. Un `curl` sur `http://localhost:8080` permet d'obtenir la page `index.html` servie par défaut par nginx.

```.term1
curl localhost:8080
```

Nous obtenons alors la réponse suivante.

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>

<p>For online documentation and support please refer to <a href="http://nginx.org/">nginx.org</a>.<br/> Commercial support is available at <a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Shell intéractif

Lançons un shell interactif dans un container basé sur l'image `alpine`, pour cela nous utilisons les options suivantes:
- l'option -t crée un pseudo terminal dans notre container
- l'option -i permet de laisser l'entrée standard ouvert

```.term1
docker container run -ti alpine
```

Une fois l'image `alpine` récupérée, un shell est lancé dans le container, comme le montre le display suivant.

```
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
2aecc7e1714b: Pull complete
Digest: sha256:0b94d1d1b5eb130dd0253374552445b39470653fb1a1ec2d8149094887
6e462c
Status: Downloaded newer image for alpine:latest
/ #
```

Note: nous n'avons pas spécifié de commande à la suite du nom de l'image (`alpine`) car par défaut la commande `sh` est lancée.

Lançons une commande dans notre container afin de connaitre la version de la distribution `alpine` que nous utilisons.

```.term1
cat /etc/issue
```

Le résultat obtenu est le suivant.

```
Welcome to Alpine Linux 3.6
```

Nous pouvons maintenant sortir du container.

```.term1
exit
```

### Lancement d'un process dans un container existant

Généralement un seul processus tourne dans un container, son PID est 1. Il est parfois utile, par exemple pour des fins de debugging, de lancer un autre processus dans un container toujours en execution, c'est ce que nous allons illustrer dans cette partie.

Nous commençons par lancer, en background,  un container basé sur l'image `mongo:3.4` (MongoDB est l'une des bases NoSQL les plus connues / utilisées). Nous utilisons également l'option `--name` afin de spécifier un nom pour notre container.

```.term1
docker container run -d --name mongo mongo:3.4
```

L'image `mongo:3.4` est téléchargée depuis le Docker Hub, le container est créé et son identifiant est renvoyé.

```
Unable to find image 'mongo:3.4' locally
3.4: Pulling from library/mongo
56c7afbcb0f1: Pull complete
ac4863389b54: Pull complete
fccbd1684456: Pull complete
5565b5f177e7: Pull complete
b00971241c47: Pull complete
c0300dc07374: Pull complete
b93b8ba2b93b: Pull complete
661362338965: Pull complete
fa170e22de2e: Pull complete
86992ab45331: Pull complete
e233325a655e: Pull complete
Digest: sha256:c4bc4644b967a4b58022a79cf5c9afcd25ed08180c958a74df57b7753c
fc8649
Status: Downloaded newer image for mongo:3.4
6c15ee6473ab1f756f99623de5c5965c8d7fc0fa319e4e3d0134e9e312264a40
```

La commande `docker container exec` permet de lancer un processus dans un container existant. Nous allons lancer un shell intéractif dans le container `mongo` que nous venons de créer.
Pour cela, nous pouvons fournir l'ID ou bien le nom du container à la commande `exec`.

```.term1
docker container exec -ti mongo bash
```

Nous avons ainsi access à un shell `bash` à l'intérieur du container `mongo`. Regardons quels sont les processus en cours dans ce container.

```.term1
ps ax
```

Le résultat de cette commande est semblabe à celui ci-dessous. Le process de PID 1 est `mongod`, le daemon mongo. C'est la commande qui est lancée par défaut lorsque l'on créé un container à partir d'une image `mongo`.

```
   PID TTY      STAT   TIME COMMAND
     1 ?        Ssl    0:17 mongod
    35 ?        Ss     0:00 bash
    42 ?        R+     0:00 ps ax
```

Nous pouvons également voir que le process `bash`, dans lequel nous sommes, est également listé.

```.term1
exit
```

## Inspection d'un container

La commande `inspect` permet d'obtenir une vue détaillé d'un container (état, paramètres de configuration, ...).
Nous commençons par lancer un container basé sur nginx.

```.term1
docker container run --name www -d nginx
```

Nous utilisons la command `inspect` pour obtenir les détails de ce container.

```.term1
docker container inspect www
```

Cette commande génère beaucoup d'information. Souvent, nous n'aurons besoin que d'un sous-ensemble de ces informations, parfois même d'un seul champ. Pour cela nous pouvons utiliser les Go templates afin de récupérer seulement l'information dont nous avons besoin.

Les 2 commandes suivantes utilisent le format Go template pour récupérer le `hostname` et l'adresse `IP` du container.

La clé `Hostname` est disponible dans `Config`.

```.term1
docker container inspect --format "{{ "{{ .Config.Hostname " }}}}" www
```

La clé `IPAdress` est disponible dans `NetworkSettings`.

```.term1
docker container inspect --format "{{ "{{ .NetworkSettings.IPAddress " }}}}" www
```

**Exercice**: sélectionnez d'autres éléments de la structure json retournée par la commande `inspect` et essayez de les récupérer en utilisant les Go templates.

## Autres commandes pour la gestion des containers

La liste des commandes disponibles pour la gestion des containers est obtenue avec

```.term1
docker container --help
```

Nous avons déjà vu certaines d'entre elles, n'hésitez pas à en expérimenter d'autres.

## Comprendre la layer associée au container 

La layer d'un container, est la layer read-write créé lorsqu'un container est lancé. C'est la layer dans laquelle tous les changements effectués dans le container sont sauvegardés.
Cette layer est supprimée avec le container et ne doit donc pas être utilisée comme un stockage persistant.

Nous allons illustrer cela en commençant par lancer un shell intéractif dans un container basé sur l'image `ubuntu`.

```.term1
docker container run -ti ubuntu
```

`figlet` est un package qui prend un texte en entrée et le formatte de façon marrante. Par défaut ce package n'est pas disponible dans l'image `ubuntu`.

```.term1
figlet
```

La commande devrait donner le résultat suivant.

```
bash: figlet: command not found
```

Nous installons `figlet` dans le container.

```.term1
apt-get update -y
apt-get install figlet
```

Vérifions que le binaire fonctionne

```.term1
figlet Holla
```

Ce qui devrait donner le résultat suivant

```
 _           _ _
| |__   ___ | | | __ _
| '_ \ / _ \| | |/ _` |
| | | | (_) | | | (_| |
|_| |_|\___/|_|_|\__,_|
```

Nous pouvons maintenant sortir du container
```.term1
exit
```

Nous lançons un nouveau container basé sur `ubuntu`.

```.term1
docker container run -ti ubuntu
```

Est ce que le mackage `figlet` est présent ?

```.term1
figlet
```

Nous obtenons alors l'erreur suivante

```
bash: figlet: command not found
```

Comment expliquez-vous ce résultat ?

Chaque container lancé à partir de l'image `ubuntu` est différent des autres. Le second container est différent de celui dans lequel `figlet` a été installé. Chacun correspond à une instance de l'image `ubuntu` et a sa propre layer, ajoutée au dessus des layers de l'image, et dans laquelle tous les changements effectué dans le container sont sauvés.

Sortons du container

```.term1
exit
```

Nous pouvons lister les containers en exécution

```.term1
docker container ls
```

ainsi que l'ensemble des containers (en exécution ou non) sur la machine hôte

```.term1
docker container ls -a
```

Depuis cette liste, récuperez l'ID du container dans lequel le package `figlet` a été installé (ce devrait être le second container de la liste) et redémarrez le avec la commande `start`.

```
docker container start CONTAINER_ID
```

Lancé un shell intéractif dans ce container en utilisant la commande `exec`.

```
docker container exec -ti CONTAINER_ID bash
```

Vérifez que `figlet` est présent dans ce container.

```.term1
figlet still there !
```

Nous pouvons maintenant sortir de ce container une nouvelle fois.

```.term1
exit
```

## Nettoyage

Nous listons l'ensembles des containers créés sur la machine.

```.term1
docker container ls -a
```

Si nous ajoutons l'option `-q`, nous obtenons seulement les IDs des containers.

```.term1
docker container ls -aq
```

Pour supprimer tous les containers, nous pouvons utiliser les commandes `rm` et `ls -aq` conjointement. Nous ajoutons l'option `-f` afin de forcer la suppression des containers encore en exécution. Il faudrait sinon arrêter les containers et les supprimer.

```.term1
docker container rm -f $(docker container ls -aq)
```

Tous les containers ont été supprimés.

```.term1
docker container ls -a
```

## En résumé

Nous avons commencé à jouer avec les containers et à comprendre la layer read-write qui est créée et associée à chaque container. Nous avons également vu les commandes les plus utilisées pour la gestion du cycle de vie des containers (run, exec, ls, rm, inspect).
