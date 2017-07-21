---
layout: post
title:  "Raft logs"
author: "@lucjuggery"
tags: [FR]
categories: "Swarm"
terms: 1
---

Dans ce lab nous allons démistifier les logs qui sont utilisés par l'algorithme Raft afin de gérer l'état du cluster.

Ces logs sont crées sur le Leader des managers et répliqués sur chaque manager.

![Swarm architecture]({{ site.baseurl }}/images/swarm-2.png)

## Swarm mode

### Initialisation du Swarm

Lorsque nous sommes sur un node qui n'est pas en mode Swarm, le répertoire `swarm` du dossier d'installation de Docker est vide.

```.term1
ls /graph/swarm
```

Note: sur une installation locale, le dossier d'installation par défaut est `/var/lib/docker`.

La commande suivante nous permet d'initialiser le swarm.

```.term1
docker swarm init --advertise-addr $(hostname -i)
```

```
Swarm initialized: current node (yvx4v7mh3e2bm75texkqj1iyc) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-41koom95vtl46v9hwyucr19i9ry8nksrm543lipiz6nsnvp8e1-c1xaxoffso12idddq
8tst5yta 10.0.23.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

A partir du moment ou le daemon Docker est en mode Swarm (suite à la commande d'initialisation précédente), plusieurs éléments sont présents dans le répertoire `swarm`.

```.term1
find /graph/swarm
```

```
/graph/swarm
/graph/swarm/docker-state.json
/graph/swarm/certificates
/graph/swarm/certificates/swarm-root-ca.crt
/graph/swarm/certificates/swarm-node.key
/graph/swarm/certificates/swarm-node.crt
/graph/swarm/worker
/graph/swarm/worker/tasks.db
/graph/swarm/raft
/graph/swarm/raft/snap-v3-encrypted
/graph/swarm/raft/wal-v3-encrypted
/graph/swarm/raft/wal-v3-encrypted/0000000000000000-0000000000000000.wal
/graph/swarm/raft/wal-v3-encrypted/0.tmp
/graph/swarm/state.json
```

Les logs sont situés dans le répertoire `raft`, les clés d'encryption dans le répertoire `certificates`.

### Création d'un secret

La commande suivante permet de créer un secret nommé `passwd` et contenant une chaine de caractère.

```.term1
echo 'A2e5bc21' | docker secret create passwd -
```

Si nous listons les secrets existants sur notre Swarm, seul le secret précédemment créé apparait, son contenu n'est plus visible.

```.term1
docker secret ls
```

```
ID                          NAME     CREATED        UPDATED
4mbzd3pt9jk9z2lqehm7e77bb   passwd   8 seconds ago  8 seconds ago
```

Nous verrons par la suite que ce secret est en clair dans les logs de Raft.

## A propos des logs de Raft

La version 1.13 de la plateforme Docker a introduit la gestion des secrets dans le contexte d'un Swarm. Ce sont des information sensibles, par exemple des identifiants de connexion à des services tiers.

Les secrets sont stockées en clair dans les logs utilisés par l'implémentation de l'algorithme Raft et c'est notamment pour cette raison que logs sont cryptés, afin d'assurer la confidentialité de ces informations.

Les secrets sont généralement créés par les Ops lors du lancement de l'application puis fournis au service qui en ont besoin. Ils seront alors accessibles, dans les containers du service, depuis un système de fichiers temporaire sous `/run/secrets/NOM_DU_SECRET`.

## Décryptage des logs

Nous allons utiliser ici l'utilitaire `swarm-rafttool`, un binaire qui se trouve dans la librairie `SwarmKit` utilisée par Docker pour la gestion des clusters Swarm.

### Installation de l'environnement Go

Le binaire `swarm-rafttool` étant écrit en `Go`, nous installons le package `go` dans notre instance basée sur `alpine`..

```.term1
apk update && apk add go
export GOPATH=$HOME/go
```

### Installation de swarm-rafttool

La commande suvante install le binaires `swarm-rafttool` (soyez patient, cela peut prendre quelques dizaines de secondes).

```.term1
go get github.com/docker/swarmkit/cmd/swarm-rafttool
```

Nous vérifions l'installation et les options possibles.

```.term1
$GOPATH/bin/swarm-rafttool
```

```
Tool to translate and decrypt the raft logs of a swarm manager

Usage:
 /root/go/bin/swarm-rafttool [command]

Available Commands:
  decrypt Decrypt a swarm manager’s raft logs to an optional directory
  dump-wal Display entries from the Raft log
  dump-snapshot Display entries from the latest Raft snapshot
  dump-object Display an object from the Raft snapshot/WAL

Flags:
  -h, --help help for /root/go/bin/swarm-rafttool
  -d, --state-dir string State directory (default “/var/lib/swarmd”)
  --unlock-key string Unlock key, if raft logs are encrypted

Use "/root/go/bin/swarm-rafttool [command] — help" for more information about a command.
```

Dans la suite, nous utiliserons la command `dump-wal` afin de décrypter et visualiser les entrées du fichier de logs.

### Décryptage

Afin de décrypter le fichier de log, nous créons un script shell qui va tout d'abord copier le fichier puis lancer le binaire `swarm-rafttool` sur cette copie.
L'étape de copie est nécessaire car `swarm-rafttool` ne permet pas de décrypter les logs en cours d'usage.

Avec un petit coup de `vi`, copiez dans un fichier `dump.sh` le contenu suivant:

```
d=$(date "+%Y%m%dT%H%M%S")
SWARM_DIR=/graph/swarm
WORK_DIR=/tmp
DUMP_FILE=$WORK_DIR/dump-$d
STATE_DIR=$WORK_DIR/swarm-$d
cp -r $SWARM_DIR $STATE_DIR
$GOPATH/bin/swarm-rafttool dump-wal --state-dir $STATE_DIR > $DUMP_FILE
echo $DUMP_FILE
```

Nous pouvons alors lancer ce script et observer le contenu des logs.

```.term1
chmod +x ./dump.sh
./dump.sh | xargs cat
```

La sortie est relativement verbeuse, et peux être décomposée en plusieurs `Entry`. Je vous invite à examiner les premières qui sont relative à la mise en place du Swarm..

La dernière `Entry` (ci-dessous) concerne la création du secret `passwd`.

```
Entry Index=12, Term=2, Type=EntryNormal:
id: 102286946670602
action: <
  action: STORE_ACTION_CREATE
  secret: <
    id: "4mbzd3pt9jk9z2lqehm7e77bb"
    meta: <
      version: <
        index: 11
      >
      created_at: <
        seconds: 1499070018
        nanos: 989531240
      >
      updated_at: <
        seconds: 1499070018
        nanos: 989531240
      >
    >
    spec: <
      annotations: <
        name: "passwd"
      >
      data: "A2e5bc21\n"
    >
  >
>
```

Comme nous pouvons le voir, le contenu du secret est en clair dans le log. L'encryption des logs est donc obligatoire pour préserver la sécurité de ces informations sensibles.

## Autolock

Si un manager est compromis, les logs cryptés et les clés d'encryption sont récupérables. Il est alors facile pour un hacker de décrypter les logs et d'avoir ainsi accès aux données sensibles, comme nous venons de le faire. Pour empêcher cela, un Swarm peut être locké. Une clé d'encryption est alors générée et utilisée pour encrypter les clés publique / privée (celles servant à encrypter / décrypter les logs).

Cette nouvelle clé, appelée `Unlock key`, doit être sauvegardée offline et fournie manuellement au daemon Docker après un restart.

Nous visualisons le contenu des clés situées dans le sous répertoire `certificates`.

```.term1
clear
cat /graph/swarm/certificates/swarm-node.crt
cat /graph/swarm/certificates/swarm-node.key
```

La commande suivante met à jour le Swarm et active la fonctionnalité d'`Autolock`.

Note: il est également possible d'activer l'`Autolock` lors de la création du Swarm.

```.term1
docker swarm update --autolock=true
```

```
Swarm updated.
To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-y4plj3mAYXoS4OiHHU9TC23vjKM6dgmcjdFju/2YTX0

Please remember to store this key in a password manager, since without it y
ou
will not be able to restart the manager.
```

Si nous observons une nouvelle fois le contenu des clés, nous pouvons voir qu'elles ont été encryptées.

```.term1
clear
cat /graph/swarm/certificates/swarm-node.crt
cat /graph/swarm/certificates/swarm-node.key
```

### Redémarage du daemon

Reperez le PID du processus `dockerd` et envoyez lui un signal `KILL`.

```
# Liste des processus en cours
ps aux
PID   USER     TIME   COMMAND
    1 root       0:00 /bin/sh -c cat /etc/hosts >/etc/hosts.bak &&     sed 's/^::1.*
    7 root       0:00 dockerd
    8 root       0:00 script -q -c /bin/bash -l /dev/null
   10 root       0:00 sh -c /bin/bash -l
   11 root       0:00 /bin/bash -l
   29 root       0:00 /usr/sbin/sshd -o PermitRootLogin=yes -o PrintMotd=no
   39 root       0:00 docker-containerd -l unix:///var/run/docker/libcontainerd/dock
  190 root       0:00 ps aux

# Stop le processus dockerd
kill 7
```

Redémarrez ensuite le daemon avec la commande suivante.

```
nohup dockerd &
```

Il n'est alors pas possible de lancer de commande sur le swarm tant que le manager n'a pas été unlocké.

```.term1
docker node ls
```
```
Error response from daemon: Swarm is encrypted and needs to be unlocked bef
ore it can be used. Please use "docker swarm unlock" to unlock it.
```

Nous délockons alors le manager en lui précisant le token récupéré lors de l'opétation de lock.

```.term1
docker swarm unlock
```
```
Please enter unlock key:
```

```
docker node ls
ID                            HOSTNAME  STATUS  AVAILABILITY        MANAGER STATUS
x7xrfx5cddxoxhan2ekc695oa *   node1     Ready   Active              Leader
```

## En résumé

Ce labs donne un rapide aperçu des logs générés par l'algorithme de concensus Raft. Nous l'avons illustré en nous basant sur un Swarm simplement composé d'un seul manager. Vous trouverez des détails supplémentaires dans cet [article](https://medium.com/lucjuggery/raft-logs-on-swarm-mode-1351eff1e690) (en anglais).
