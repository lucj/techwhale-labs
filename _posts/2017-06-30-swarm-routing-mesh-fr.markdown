---
layout: post
title:  "Routing Mesh"
author: "@lucjuggery"
tags: [FR]
categories: "Swarm"
terms: 2
---

Dans ce lab nous allons illustrer le mécanisme de Routing Mesh utilisé dans un Swarm.

## Présentation

Le mode Swarm de Docker facilite la publication des ports d'un service afin de les rendre accessible depuis l'extérieur du Swarm. Tous les nodes du Swarm participent à un réseau `ingress` effectuant le routage en entrée. Celui-ci permet à chaque node du Swarm d'accepter des connexions sur les ports publiés d'un service, même si aucune tâche de ce service ne tourne sur le node en question. Le routing mesh route alors les requêtes vers un node sur lequel tourne un container du service.

Le schéma suivant illustre ce mécanisme.

![Routing Mesh]({{ site.baseurl }}/images/ingress-routing-mesh.png)

## Création d'un swarm

Pour montrer ce mécanisme, nous allons créer un Swarm avec 2 nodes.

La commande suivante initialise le Swarm.

```.term1
docker swarm init --advertise-addr $(hostname -i)
```

```
Swarm initialized: current node (i73fq7u9r0yxuwfchpsraofb6) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5fa2jlkmcqg6ceb8nl8h566fwiegi2bvyhlh6bzdg3sva0kmsg-a88bsyrz7b5twrdlt
q4ttfrfm 10.0.64.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Il faut alors copier-coller la commande de join, retournée par la commande précédent, dans le second terminal. Cela permettra d'ajouter le second node en tant que worker à notre Swarm.

Note: lorsque vous faites le copier-coller, faites attention à bien le faire en plusieurs fois (une fois par ligne).

```
$ docker swarm join --token SWMTKN-1-5fa2jlkmcqg6ceb8nl8h566fwiegi2bvyhlh6bzdg3sva0kmsg-a88bsyrz7b5twrdltq4ttfrfm 10.0.64.3:2377
This node joined a swarm as a worker.
```

Le Swarm est composé de 2 nodes.

```.term1
docker node ls
```

```
ID                            HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
1dnejmbax1nxoil5hskh8k9y9     node2     Ready   Active
i73fq7u9r0yxuwfchpsraofb6 *   node1     Ready   Active        Leader
```

## Lancement d'un service

Nous lançons à présent un service avec les caractéristiques suivantes:
- basé sur l'image `instavote/vote`
- nommé `vote`
- mode répliqué avec un seul réplica (configuration par défaut)
- publication du port 80 sur le port 8080 sur le Swarm

```.term1
docker service create --name vote --publish 8080:80 instavote/vote
```

Après quelques secondes, le temps que l'image soit téléchargée depuis le Docker Hub, nous pouvons voir que le service est disponible.

```.term1
docker service ls
```

```
ID             NAME  MODE        REPLICAS  IMAGE                  PORTS
yqj26nfcr57q   vote  replicated  1/1       instavote/vote:latest  *:8080->80/tcp
```

La commande suivante nous permet de voir que l'unique tâche du service `vote` (un seul réplica ayant été défini) a été lancée sur le node1.

```.term1
docker service ps vote
```

```
ID            NAME   IMAGE                  NODE  DESIRED STATE  CURRENT STATE          ERROR PORTS
4hkchcicdlzv  vote.1 instavote/vote:latest  node1 Running        Running 2 minutes ago
```

## Routing Mesh

Le Routing Mesh permet au service d'être accessible depuis le port 8080 de chaque node du Swarm.

[Accès au service depuis le node1](/){:data-term=".term1"}{:data-port="8080"}

[Accès au service depuis le node2](/){:data-term=".term2"}{:data-port="8080"}

Dans les cas nous accedons à l'interface de vote qui permet de choisir entre `cats` et `dogs`. 

![Vote]({{ site.baseurl }}/images/swarm-routing-mesh-vote.png)

## Quelques explications

Le routing mesh repose sur `iptables` (règles de firewall) et `ipvs` (load balancer au niveau 4).

### Rêgles de routage sur la machine hôte

Lors de la création du service, des règles de routage, relatives au port 8080, ont étés définies dans `iptables` sur chaque node.

Depuis le node1, nous examinons la table `nat`.

```.term1
iptables -nvL -t nat
```

Le traffic qui arrive sur le port `8080` du node1 est redirigé sur l'IP `172.19.0.2`, comme le montre la chaine de `DOCKER-INGRESS`.

```
...
Chain DOCKER-INGRESS (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.19.0.2:8080
   34  2116 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
...
```

### Namespace réseau

Depuis node1, nous pouvons lister les namespaces réseau existant.

```.term1
cd /var/run/docker/netns && ls
```

Nous obtenons un résultat similaire à celui-ci

```
1-jk0jdmskj8  50d8c6c2e4c2  ingress_sbox
```

Dans cet exemple, le namespace `1-jk0jdmskj8` correspond au network `ingress` créé sur le Swarm. L'identifiant de ce réseau peut-être obtenu en listant les réseaux existants.

```.term1
docker network ls
```

```
NETWORK ID          NAME                DRIVER              SCOPE
42bcaf663e70        bridge              bridge              local
49dbd666d00a        docker_gwbridge     bridge              local
541cddf6642f        host                host                local
jk0jdmskj80c        ingress             overlay             swarm
d60dedda7ccc        none                null                local
```

En utilisant `nsenter`, nous lançons un shell dans le namespace `ingress_sbox`.

```.term1
nsenter --net=ingress_sbox sh
```

Nous pouvons voir que l'une des interface réseau présente dans ce namespace à l'IP référencée dans la table `nat` de `iptables` (172.19.0.2 dans cet exemple).

```.term1
ip a
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:0a:ff:00:02 brd ff:ff:ff:ff:ff:ff
    inet 10.255.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
8: eth1@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
```

Toujours depuis le shell en exécution dans le namespace `ingress_sbox`, nous regardons la table `mangle` de `iptables`.

```.term1
iptables -nvL -t mangle
```

```
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 MARK set 0x100

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  *      *       0.0.0.0/0            10.255.0.4           MARK set 0x100

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

Les packets arrivant sur le port `8080` sont estampilés avec la valeur `0x100` (256 en décimal) dans la chaine PREROUTING et sont envoyés sur l'IP `10.255.0.4` dans le chaine OUTPUT. Cette adresse IP correspond à l'adresse IP virtuelle (VIP) du service `vote` comme le montre la commande d'inspection suivante:

```.term1
docker service inspect -f "{{ "{{ .Endpoint.VirtualIPs " }}}}" vote
```

```
[{cyxk0zhe2qcy7eh9sfgjsb3od 10.255.0.4/16}]
```

### Load balancer L4

IPVS est une technologie présente dans le kernel Linux depuis plusieurs années et permet de faire du load-balancing au niveau 4.

Nous commançons par installer le package `ipvsadm` qui n'est pas présent par défaut dans l'image `alpine`.

```.term1
apk update && apk add ipvsadm
```

Nous listons ensuite les tables `ipvs` avec la commande suivante.

```.term1
ipvsadm -ln
```

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  256 rr
  -> 10.255.0.5:0                 Masq    1      0          0
```

Nous pouvons voir que les paquets marqués avec `256` (0x100) sont formardé sur une IP particulière. Il s'agit de l'adresse IP du seul container du service. Nous pouvons le vérifier en listant les tâches du service puis en inspectant le container depuis l'hôte sur lequel il tourne.

Nous n'avons qu'un seul container pour le service `vote`. La commande suivante met le service à jour en spécifiant 3 réplicas.

```.term1
docker service update --replicas 3 vote
```

3 tâches sont alors créées sur le cluster, et pour chacune d'entre elles un container sera lancé.

Si nous listons une nouvelle fois les tables de `ipvs`, nous pouvons voir que le traffic sera load-balancé entre les 3 containers.

```.term1
ipvsadm -ln
```

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  256 rr
  -> 10.255.0.5:0                 Masq    1      0          0
  -> 10.255.0.6:0                 Masq    1      0          0
  -> 10.255.0.7:0                 Masq    1      0          0
```

## Résumé

Ce lab est une overview des technologies impliquées dans le routing mesh, notamment `iptabes` et `ipvs`. L'avantage de Swarm est notamment de cacher une grande partie de cette complexité à l'utilisateur.

Le port publié par un service étant disponible sur chaque node, on mettra souvent un load-balancer en front-end du Swarm de façon à repartir la charge.

