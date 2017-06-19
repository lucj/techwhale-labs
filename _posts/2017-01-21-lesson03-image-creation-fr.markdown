---
layout: post
title:  "Création d'images"
date:   2017-01-21
author: "@lucjuggery"
img: "docker-image.png"
tags: [FR]
categories: Images
---

Dans ce lab, nous allons voir comment créer une image à partir d'un container. Même si ce n'est pas l'approche recommandée, il est intéressant de la voir au moins une fois. Ensuite, nous créerons une image avec un Dockerfile et nous étudierons comment l'image est construite pour avoir une meilleure compréhension des briques sous-jacentes.

## Création d'une image à partir d'un container

Nous commençons par lancer un shell intéractive dans un container basé sur l'image `ubuntu`. Nous lui donnons le nom `fig`.

```.term1
docker container run -ti --name fig ubuntu bash
```

Comme nous l'avons fait dans le lab précédent, nous installons le package `figlet` dans ce container.

```.term1
apt-get update -y
apt-get install figlet
```

Une fois le package installé, nous sortons du container.

```.term1
exit
```

Nous créons une image à partir du container `fig` lancé précédemment et nous l'appelons `myfiglet`.

```.term1
docker container commit fig myfiglet
```

Nous pouvons voir que l'image que nous venons de créer est lister avec la commande suivante

```.term1
docker image ls
```

Nous pouvons maintenant lancer un container à partir de l'image `myfiglet` et lui spécifier la commande à lancer.

```.term1
docker container run myfiglet figlet hello
```

Comme le package `figlet` est présent dans l'image, la commande retourne le résultat suivant.

```
 _          _ _
| |__   ___| | | ___
| '_ \ / _ \ | |/ _ \
| | | |  __/ | | (_) |
|_| |_|\___|_|_|\___/

```

Cet exemple montre que l'on peut créer un container y ajouter manuellement des packages et ensuite le commiter pour créer une nouvelle image. Nous pouvons ensuite utiliser cette image pour instancier de nouveaux containers qui contiendront les packages installés. Cette approche n'est pas recommandée et elle n'est pas très portable.

Dans la suite de ce lab, nous allons voir la méthode la plus utilisée pour créer une image. Celle-ci utilise un Dockerfile, fichier texte qui contient toutes les instructions pour construire une image.

## Création d'une image à partir d'un Dockerfile

Pour illustrer ceci, nous allons utiliser une application ultra simple construite avec Node.js, qui ne fait que écrire un message sur la sortie standard en indiquant le nom du host sur lequel elle est lancée (le nom du container).

Créez un fichier `index.js` contenant le code suivant.

```
var os = require("os");
var hostname = os.hostname();
console.log("hello from " + hostname);
```

Nous allons Dockerizer cette application en commençant par écrire un Dockerfile. Nous utilisons `alpine` comme image de base, ajoutons le runtime `Node.js` et copions ensuite le code source.
Nous spécifions également la commande à lancer dans le container.

Créez un fichier Dockerfile contenant les instructions suivantes.

```
FROM alpine
RUN apk update && apk add nodejs
COPY . /app
WORKDIR /app
CMD ["node","index.js"]
```

Nous construisons notre première image à partir de ce Dockerfile et l'appelons `hello:1.0`

```.term1
docker image build -t hello:1.0 .
```

Nous pouvons ensuite lancer un container à partir de l'image que nous venons de créer.

```.term1
docker container run hello:1.0
```

et obtenir un résultat similaire à celui ci-dessous (avec un ID différent)

```
hello from 92d79b6de29f
```


Il y toujours plusieurs façons d'écrire un Dockerfile, nous pouvons partir d'une distribution Linux et intaller un runtime (c'est ce que nous avons fait ci-dessus) ou alors utiliser une image de base qui contient déjà le runtime.

Pour illustrer cela, nous créons un nouveau Dockerfile dans le lequel nous spécifions `mhart/alpine-node:6.9.4` comme image de base. Ce n'est pas une image officielle mais elle est cependant très connue et utilisée.

Créez un fichier Dockerfile-v2 contenant les instructions suivantes:

```
FROM mhart/alpine-node:6.9.4
COPY . /app
WORKDIR /app
CMD ["node","index.js"]
```

Il n'y a pas de grosses différences avec la première version du Dockerfile, on utilise simplement ici une image de base qui contient déjà le runtime Node.js.
Dans cet exemple, ce n'est pas compliqué d'installé Node.js mais pour des environnements plus compliqué il est souvent intéressant d'utiliser une image de base contenant déjà ce que l'on veut. 

Nous créons alors une nouvelle image basée sur Dockerfile-v2.

```.term1
docker image build -f Dockerfile-v2 -t hello:2.0 .
```

Note: comme le fichier utilisé pour construire l'image s'appelle Dockerfile-v2, et non Dockerfile, nous devons le spécifier avec l'option -f.

Nous pouvons maintenant lancer un container à partir de cette image et obtenir le même résultat que précédemment (avec un autre ID).

```.term1
docker container run hello:2.0
```
