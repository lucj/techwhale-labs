---
layout: post
title:  "Vue d'ensemble des networks"
author: "@lucjuggery"
tags: [FR]
categories: Network
terms: 2
---

Dans ce lab, nous allons étudier la mise en réseau des containers Docker. Nous verrons notamment les networks qui sont crées par défaut lors de l'installation de la plateforme et passerons en revue différents drivers (bridge, overlay, macvlan), qui ont chacun leur particularité et domaine d'application.

## Les networks créés par défaut

Lors de l'installation de la plateforme Docker, plusieurs networks sont créés. Nous pouvons les lister avec la commande suivante.

```.term1
docker network ls
```
```
NETWORK ID          NAME                DRIVER              SCOPE
78600647cf6b        bridge              bridge              local
f68b62cc1152        host                host                local
5a4a4afa414b        none                null                local
```

Lorsqu'un container est créé, nous pourrons spécifier le network auquel il sera attaché:

- un container attaché au network **none** n'aura pas de connectivité externe. Cela peut-être utile par exemple pour un container servant pour du debug
- un container attaché au network **host** bénéficiera de la stack network de la machine hôte
- un container attaché au network **bridge** pourra communiquer avec les autres containers attaché à ce network. Il faut cependant noter que ce type de network permet seulement une connectivité entre containers tournant sur la même machine

La commande suivante nous permet de lister les interfaces réseau de la machine hôte:

```.term1
ip a show
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:c8:84:82:10 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c8ff:fe84:8210/64 scope link
       valid_lft forever preferred_lft forever
...
```

La chose importante à noter ici est l'existance d'une interface nommée **docker0**. Celle-ci a été créée lors de l'installation de la plateforme. C'est une interface de type **bridge** sur laquelle repose le driver nommé **bridge** que nous avons vu précédemment.

### Lancement d'un container en utilisant le driver **bridge**

Par défaut, si l'option **--network** n'est pas spécifiée au lancement d'un container, celui-ci est attaché au network nommé **bridge**.

La commande suivante lance un container nommé **c1** basé sur l'image alpine.

```.term1
docker container run -d --name c1 alpine sleep 10000
```

Note: nous spécifions *sleep 10000* dans la commande de façon à ce que le processus de PID 1 tourne pendant quelques temps.

Nous utilisons alors la commande **inspect** afin d'obtenir la configuration réseau de ce container.

Note: nous utilisons la notation Go template pour récupérer directement le champ qui nous intéresse (.NetworkdSetings.Networks).

```.term1
docker container inspect -f "{{ "{{ json .NetworkSettings.Networks "}}}}" c1 | python -m json.tool
```

La sortie de cette commande nous donne l'ID du network auquel ce container est attaché, via la clé **NetworkID**. Nous pouvons alors voir que cette valeur correspond à l'ID du network nommé **bridge** que nous a retourné la commande **docker network ls** lancée précédemment. Nous obtenons également d'autres information comme l'IP du container et celle de la passerelle.

```
{
    "bridge": {
        "Aliases": null,
        "DriverOpts": null,
        "EndpointID": "1968a1143dba15cad49dc84b06839b83e42d249a5ca6a83c06092840ad205364",
        "Gateway": "172.17.0.1",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "IPAMConfig": null,
        "IPAddress": "172.17.0.2",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "Links": null,
        "MacAddress": "02:42:ac:11:00:02",
        "NetworkID": "78600647cf6b67dbe6fcc0dcc9b06a59a0b5c36033fa088c030490959901ee16"
    }
}
```

Nous pouvons également inspecter le network **bridge** et extraire la clé **Containers** afin d'obtenir les containers attachés à ce network.

```.term1
docker network inspect -f "{{ "{{ json .Containers "}}}}" bridge | python -m json.tool
```
```
{
    "3f41f1295700be13435e82df1e98f1575f4380cfdcb8b315e4b275485e4c2470": {
        "EndpointID": "1968a1143dba15cad49dc84b06839b83e42d249a5ca6a83c06092840ad205364",
        "IPv4Address": "172.17.0.2/16",
        "IPv6Address": "",
        "MacAddress": "02:42:ac:11:00:02",
        "Name": "c1"
    }
}
```

Nous pouvons alors supprimer le container **c1**.

```.term1
docker rm -f c1
```

### Lancement d'un container en utilisant le driver **host**

Dans un premier temps nous listons les interfaces réseau existantes sur la machine hôte.

```.term1
ip link show
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 02:42:4a:a3:69:00 brd ff:ff:ff:ff:ff:ff
50947: eth0@if50948: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:0a:00:a7:03 brd ff:ff:ff:ff:ff:ff
50949: eth1@if50950: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:12:00:9d brd ff:ff:ff:ff:ff:ff
```

Nous lançonc alors un shell interactif dans un container basé sur l'image alpine. Nous spécifions l'option **--network=host** de façon à ce que le container utilise la stack network de la machine hôte.

```.term1
docker container run -ti --network=host alpine sh
```

Une fois dans le container, si nous listons les interfaces réseau nous obtenons la même liste que précédement.

```.term1
ip link show
```

Nous pouvons alors sortir du container.

```.term1
exit
```

### Lancement d'un container en utilisant le driver **none**

Nous lançons à présent un shell interactif dans un container basé sur l'image alpine mais cette fois nous utilisons l'option **--network=none** de façon à ne pas donner au container une connectivité vers l'extérieur.

```.term1
docker container run -ti --network none alpine sh
```

Si nous listons les interfaces réseau du container, nous voyons que seule **lo** (l'interface locale) est disponible.

```.term1
ip a show
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

Comme l'illustre le message d'erreur obtenu par la commande suivante, il n'est pas possible d'établir de connection avec l'extérieur.

```.term1
ping 8.8.8.8
```
```
PING 8.8.8.8 (8.8.8.8): 56 data bytes
ping: sendto: Network unreachable
```

Nous pouvons alors sortir du container.

```.term1
exit
```

## **bridge** network

Le réseau **bridge** créé par défaut permet à des containers tournant sur la même machine de communiquer entre eux comme nous allons le voir maintenant.

![Bridge0]({{ site.baseurl }}/images/network/docker0.png)

Dans cette partie, nous allons lancer 2 containers, chacun étant attaché au network nommé **bridge**. Il n'est donc pas nécessaire de spécifier l'option **--network** car un container est attaché à ce network par défaut.

Lancement du premier container, nommé **c1**.

```.term1
docker container run -d --name c1 alpine sleep 10000
```

Nous utilons la commande **inspect** afin de retrouver l'address IP de ce container.

```.term1
docker container inspect -f "{{ "{{ .NetworkSettings.IPAddress "}}}}" c1
```

Nous obtenons l'IP suivante:

```
172.17.0.2
```

Note: l'adresse IP obtenue sur votre environnement pourra être différente.

Nous lançons un shell interactif dans un second container basé sur l'image alpine. Nous appelerons ce container **c2**.

```.term1
docker container run -ti --name c2 alpine sh
```

Depuis ce shell, nous essayons de *pinguer* le container **c1** en utilisant l'adresse IP de ce dernier.

```.term1
ping -c 3 172.17.0.2
```
```
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.135 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.098 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.110 ms

--- 172.17.0.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.098/0.114/0.135 ms
```


Cette commande fonctionne correctement, le container **c2** est capable de pinguer le container **c1** en utilisant l'adresse IP de ce dernier.

Nous lançons la même commande que précédemment mais cette fois en utilisant le nom **c1** au lieu de l'adresse IP du container.

```.term1
ping -c 3 c1
```

Nous obtenons alors le message d'erreur suivant.

```
ping: bad address 'c1'
```

Il n'est pas possible, pour des containers attachés au network **bridge** de communiquer via leur nom. Nous verrons que cela est par contre possible si nous définissons notre propre network de type bridge.

Nous pouvons alors sortir du container c2 et supprimer c1 et c2.

```.term1
exit
```
```.term1
docker rm -f c1 c2
```

## "User defined" bridge network

Nous avons vu que, par défaut, 3 networks sont définis (ceux-ci sont nommés **bridge**, **host** et **none**). Il est également possible d'en créer d'autres, c'est ce que nous allons voir dans les sections suivantes.

![User defined bridge]({{ site.baseurl }}/images/network/user-defined-bridge.png)

Nous allons commencer par créer un nouveau network de type bridge que nous appelerons **bnet**. Nous utilisons pour cela la commande **docker network create**.

```.term1
docker network create --driver bridge bnet
```

Note: nous spécifions ici l'option **--driver bridge** pour l'illustration même si ce n'est pas nécessaire car c'est le driver utilisé par défaut.

Nous lançons un container que nous attachons au network **bnet**, nous utilisons pour cela l'option **--network=bnet**. Nous appelerons ce container **c1**. 

```.term1
docker container run -d --name c1 --network bnet alpine sleep 10000
```

Comme précédemment, nous recuperons l'adresse IP de ce container. Cette fois, nous utilisons une clédifférente afin de récupérer l'IP alouée sur le réseau **bnet**.

```.term1
docker container inspect -f "{{ "{{ json .NetworkSettings.Network.bnet.IPAddress "}}}}" c1
```
```
172.19.0.2
```

Note: l'adresse IP obtenue sur votre environnement peux être différente.

Nous lançons alors un shell interactif dans un second container, nommé **c2**, que nous attachons également au network **bnet**.

```.term1
docker container run -ti --name c2 --network bnet alpine sh
```
Depuis **c2** nous pouvons constater que, comme c'est le cas pour le network **bridge** par défaut, les containers attachés au network **bnet** peuvent communiquer via leur adresse IP.

```
ping -c 3 172.19.0.2
```
```
PING 172.19.0.2 (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=1.557 ms
64 bytes from 172.19.0.2: seq=1 ttl=64 time=0.093 ms
64 bytes from 172.19.0.2: seq=2 ttl=64 time=0.088 ms

--- 172.19.0.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.088/0.579/1.557 ms
```

Cependant, dans le cas d'un network de type "user defined", il est également possible de communiquer en utilisant le nom du container.

```.term1
ping -c 3 c1
```
```
PING c1 (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.096 ms
64 bytes from 172.19.0.2: seq=1 ttl=64 time=0.095 ms
64 bytes from 172.19.0.2: seq=2 ttl=64 time=0.089 ms

--- c1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.089/0.093/0.096 ms
```

Nous pouvons alors sortir du container, et supprimer **c1** et **c2**.

```.term1
exit
```

```.term1
docker rm -f c1 c2
```

## Network de type **overlay**

Alors qu'un network de type **bridge** permet une connectivité entre les containers qui tournent sur la mxeême machine hôte, un network de type **overlay** permet d'étendre la connectivité entre des containers qui tournent sur des machines différentes.

![Overlay]({{ site.baseurl }}/images/network/overlay.png)

Un network de type **overlay** ne peut être créé que dans un contexte de cluster d'hôtes Docker. Il y a plusieurs façons de mettre en place un cluster:
- mettre plusieurs hôtes en réseau et les faire communiquer via un key value store (comme Consul, Etcd or Zookeeper). Cela nécessite de fournir au daemon Docker des options supplémentaires lors du lancement
- utiliser un Swarm. C'est l'option recommandée et certainement la plus facile à mettre en place.

Nous allons illustrer les networks de type **overlay** à partir d'un Swarm.

### Création d'un cluster Swarm

Nous commençons par mettre en place un Swarm composé de 2 nodes:

- initialisation du Swarm 

Nous utilisons pour cela la commande suivante

```.term1
docker swarm init --advertise-addr $(hostname -i)
```

Cette commande nous renvoie un résultat similaire à celui-ci (au token prêt)

```
Swarm initialized: current node (yb45ycf1dte88kaug3a93ki2b) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5auo2f16fg1l14mbeltsq3gdm9eyw30mxvkc9xxt0u27pxcn8p-98a9odjnhzfd5v1dvqoltmmoy 10.0.3.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

- ajout d'un second node

Depuis le second terminal, il suffit de rentrer la commande retournée lors de l'initialisation du Swarm. Cette commande à le format suivant (il faudra cependant remplacer le token de cet exemple par celui obtenu lors de l'initialisation).

```
docker swarm join --token SWMTKN-1-6ah9cfii61wwq93arej4smc770nduqaxql46ij0hdfbvi1vg7h-a6lhb0vfthaa8ylsfg957qe61 10.0.167.3:2377
```

Lors de la création du Swarm, plusieurs networks additionnels ont été créés comme le montre la commande suivante.

```.term1
docker network ls
```
```
NETWORK ID          NAME                DRIVER              SCOPE
88a54e9b62bf        bridge              bridge              local
db786f56c493        docker_gwbridge     bridge              local
e258298952db        host                host                local
b7377ba63cic        ingress             overlay             swarm
0b8118bb6d50        none                null                local
```

Le network **ingress** assure la connectivité depuis l'exterieur et la fonction de **routing mesh**.
Le network **docker_gwbridge** assure la connectivité vers l'extérieur.

Afin de permettre la connectivité entre des containers tournant sur les différents hôtes du cluster, nous allons créer un network de type **overlay**. Nous appelerons ce network **ovnet**.

```.term1
docker network create --driver overlay ovnet
```

Nous pouvons alors créer 2 services, en les attachant au network **ovnet**.

```.term1
docker service create --name s1 --network ovnet alpine sleep 100000
docker service create --name s2 --network ovnet alpine sleep 100000
```

Nous vérifions tout dabord que les containers de ces services (chaque service ayant un réplica) tourne chacun sur une machine différente.

```.term1
docker service ps s1
```
```
ID             NAME     IMAGE          NODE    DESIRED STATE   CURRENT STATE           ERROR     PORTS
6flev48l03un   s1.1     alpine:latest  node1   Running         Running 11 seconds ago
```

```.term1
docker service ps s2
```
```
ID             NAME      IMAGE          NODE   DESIRED STATE   CURRENT STATE           ERROR     PORTS
dwd5epc77snp   s2.1      alpine:latest  node2  Running         Running 6 seconds ago
```

Le container du service **s1** tourne sur le **node1** alors que celui du service **s2** tourne sur **node2**.

Nous allons ensuite vérifier que les 2 containers peuvent communiquer entre eux bien qu'ils ne soient pas sur la même machine.

Depuis le container relatif au service **s1**, nous lançons la commande **ping** vers le container du service **s2** en précisant seulement le nom du service.

Note: dans cette commande, nous utilisons un template Go pour récupérer l'ID du container du service **s1** (unique container tournant sur la machine).

```.term1
docker exec -ti $(docker ps --format "{{ "{{ .ID  "}}}}") ping -c 3 s2
```
```
PING s2 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: seq=0 ttl=64 time=0.057 ms
64 bytes from 10.0.0.4: seq=1 ttl=64 time=0.056 ms
64 bytes from 10.0.0.4: seq=2 ttl=64 time=0.063 ms

--- s2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.056/0.058/0.063 ms
```

Cet exemple illustre la communication, via un network de type **overlay**, entre des containers tournant sur des machines différentes.
Comme c'était également le cas dans le cadre d'un network de type "user defined" bridge, la résolution de nom est effectuée par le serveur DNS interne.

### Pour aller plus loin

#### Network namespaces

L'initialisation du Swarm, la création du réseau **overlay** et des containers (via les services **s1** et **s2**) a créé différents namespace network sur les 2 nodes du cluster.

La commande suivante liste ces namespaces sur le node1

```.term1
ls /var/run/docker/netns
```
```
1-b7377ba63c  1-ykmtsf86r3  f07f768f34da  ingress_sbox
```

Les namespaces préfixés par **1-** correspondent aux networks de type **overlay**. Nous pouvons d'ailleurs le voir via la correspondance de ces noms de namespaces avec les IDs des networks.

```.term1
docker network ls --filter driver=overlay
```
```
NETWORK ID          NAME                DRIVER              SCOPE
b7377ba63cic        ingress             overlay             swarm
ykmtsf86r3b9        ovnet               overlay             swarm
```

Le namespace **ingress_sbox** permet la connectivité depuis l'extérieur et assure les fonctions de routing mesh.

Le dernier namespace correspond à celui du container du service **s1** qui tourne sur **node1**. Nous pouvons d'ailleurs le confirmer par la correspondance des résultats des 2 commandes suivantes qui listent les interfaces de ce namespace:

- Liste des interfaces depuis le namespace

```
$ nsenter --net=/var/run/docker/netns/f07f768f34da ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:0a:00:00:03 brd ff:ff:ff:ff:ff:ff
13: eth1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:13:00:03 brd ff:ff:ff:ff:ff:ff
```

- Liste des interfaces depuis le container

```
$docker exec -ti $(docker ps -aq) ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:0a:00:00:03 brd ff:ff:ff:ff:ff:ff
13: eth1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:13:00:03 brd ff:ff:ff:ff:ff:ff
```

#### Les interfaces du container

Les interfaces **eth0** et **eth1** font chacune partie d'une paire d'interfaces virtuelles.

L'interface **eth0** correspond à l'extrémité d'une paire d'interfaces virtuelles dont l'autre extrémité (dont l'identifiant est 12) est liée au namespace network du réseau **ovnet**. Nous pouvons le confirmer en listant les interfaces existantes dans ce namespace avec la commande **nsenter**.

```.term1
NAMESPACE_OVNET="1-$(docker network ls -f name=ovnet -q | cut -c 1-10)"
nsenter --net=/var/run/docker/netns/$NAMESPACE_OVNET ip a
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
    link/ether 32:b8:99:48:24:9d brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 scope global br0
       valid_lft forever preferred_lft forever
10: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN
    link/ether 76:fa:0b:d0:f5:4e brd ff:ff:ff:ff:ff:ff
12: veth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue master br0 state UP
    link/ether 32:b8:99:48:24:9d brd ff:ff:ff:ff:ff:ff
```

De la même façon, nous pouvons montrer que l'interface **eth1** du container correspond à une paire d'interfaces virtuelles dont l'autre extrémité (dont l'identifiant est 14) se trouve attachée au bridge **docker_gwbridge**. Ce bridge étant dans le namespace network de la machine hôte, nous pouvons voir l'interface dans la liste des interfaces de l'hôte.

```.term1
ip a
```
```
...
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 02:42:cd:10:a4:b6 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
7: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:a7:c2:1e:bd brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 scope global docker_gwbridge
       valid_lft forever preferred_lft forever
14: vetha98df89@if13: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker_gwbridge state UP
    link/ether 82:ec:c7:b7:ca:a7 brd ff:ff:ff:ff:ff:ff
```

#### Le namespace ingress_sbox

De la même façon, en inspectant les namespaces, nous pourrions montrer que le namespace **ingres_sbox** a deux interfaces, chacune faisant partie d'une paire d'interfaces virtuelles.

```.term1
nsenter --net=/var/run/docker/netns/ingress_sbox ip a
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen1
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

La paire comportant **eth1** a une extrémité attachée au bridge **docker_gwbridge**
La paire comportant **eth0** a une extrémité attachée au network **ingress**

#### En résumé

La connection des différents namespaces que nous avons vu ci-dessus assure la connectivité des containers avec l'extérieur et également entre eux. Nous ne rentrerons pas ici dans le détail des tables de routage permettant le cheminement des paquets IP.

## Network de type **macvlan**

Un network de type **macvlan** donne à chaque container un accès direct à une interface de la machine hôte, lui permettant ainsi d'être attaché à l'infrastructure réseau existante. Les containers sont adressés par une adresse IP routable sur un sous réseau du réseau externe.

![macvlan]({{ site.baseurl }}/images/network/macvlan.png)

Un network de type **macvlan** peut être créé avec la commande **docker network create** en spécifiant l'option **--network macvlan**.

La commande suivante créé un network de type **macvlan**, nommé **mvnet**. Nous spécifions **eth0** comme étant l'interface parente et définissons un sous réseau.

```.term1
docker network create -d macvlan --subnet 192.168.0.0/24 --gateway 192.168.0.1 -o parent=eth0 mvnet
```

Nous créons ensuite 2 containers attachés à ce network, nous spécifions les IP sur le sous réseau utilisé.

```.term1
docker container run -d --name c1 --network mvnet --ip 192.168.0.3 alpine sleep 10000
docker container run -it --name c2 --network mvnet --ip 192.168.0.4 alpine sh
```

Depuis le container **c2** nous pouvons communiquer avec le container **c1** via son adresse IP ou son nom grace à au serveur DNS interne au daemon Docker.

```.term1
ping -c 3 192.168.0.3
```
```
PING 192.168.0.3 (192.168.0.3): 56 data bytes
64 bytes from 192.168.0.3: seq=0 ttl=64 time=0.104 ms
64 bytes from 192.168.0.3: seq=1 ttl=64 time=0.077 ms
64 bytes from 192.168.0.3: seq=2 ttl=64 time=0.073 ms

--- 192.168.0.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.073/0.084/0.104 ms
```

```.term1
ping -c 3 c1
```
```
PING c1 (192.168.0.3): 56 data bytes
64 bytes from 192.168.0.3: seq=0 ttl=64 time=0.086 ms
64 bytes from 192.168.0.3: seq=1 ttl=64 time=0.080 ms
64 bytes from 192.168.0.3: seq=2 ttl=64 time=0.098 ms

--- c1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.080/0.088/0.098 ms
```

Un network de type **macvlan** a un scope **local**, il est souvent utilisé pour des besoins de performances car il n'y a pas de bridge intermédiaire. Il est également utilisé pour faciliter la migration d'applications vers Docker en l'attachant à une infrastructure sous jacente existante.

Nous pouvons alors sortir du container, et supprimer **c1** et **c2**.

```.term1
exit
```

```.term1
docker rm -f c1 c2
```

