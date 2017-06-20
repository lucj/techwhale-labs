---
layout: post
title:  "Les namespaces"
date:   2017-06-18
author: "@lucjuggery"
tags: [FR]
categories: Containers
---

L'utilisation des namespaces permet de limiter la vision du système qu'à un processus
Ce lab montre des exemples d'utilisation de la commande `unshare` pour exécuter des processus dans des nouveaux namespaces.

## Illustration du namespace Network

La commande suivante liste les interfaces réseau locales

```.term1
ip a
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 02:42:e2:4c:fe:c3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
3479: eth0@if3480: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:0a:00:1b:03 brd ff:ff:ff:ff:ff:ff
    inet 10.0.27.3/24 scope global eth0
       valid_lft forever preferred_lft forever
3481: eth1@if3482: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:12:00:1d brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.29/16 scope global eth1
       valid_lft forever preferred_lft forever
```

Nous utilisons l'option `-n` de unshare pour lancer un process `sh` dans un nouveau namespace network.

```.term1
unshare -n sh
```

Seul l'interface réseau local est disponible dans ce namespace.

```.term1
ip a
```

```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

La commande suivante permet de sortir du namespace.

```.term1
exit
```

## Illustration du PID namespace

Nous utilisons l'option `-p` pour lancer un processus dans un nouveau namespace PID et vérifier les processus visible

```.term1
unshare -p ps aux
```

```
PID   USER     TIME   COMMAND
    1 root       0:00 /bin/sh -c cat /etc/hosts >/etc/hos
    9 root       0:00 dockerd
   10 root       0:00 script -q -c /bin/bash -l /dev/null
   12 root       0:00 sh -c /bin/bash -l
   13 root       0:00 /bin/bash -l
   36 root       0:00 docker-containerd -l unix:///var/ru
  145 root       0:00 ps aux
```

Les processus du système sont visibles car ils ont lus depuis /proc de la machine hôte

La commande suivante permet de sortir du namespace.

```.term1
exit
```

## Illustration du namespace IPC (Inter-process communication)

Nous créons une file de message avec la commande suivante

```.term1
ipcmk -Q
```

La commande suivante liste les files de messages existantes.

```.term1
ipcs -q
```

```
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x69c43364 0          root       644        0            0
```

Nous utilisons l'option `-i` de `unshare` pour lancer un processus `sh` dans un nouveau namespace IPC

```.term1
unshare --ipc sh
```

Nous vérifions la liste des files de messages dans le nouveau namespace.

```.term1
ipcs -q
```

```
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

Cette liste est vide car nous sommes dans un namespaces IPC différents de celui du process initial.

La commande suivante permet de sortir du namespace.

```.term1
exit
```
