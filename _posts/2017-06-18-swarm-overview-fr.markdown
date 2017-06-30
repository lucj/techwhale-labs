---
layout: post
title:  "Vue d'ensemble"
author: "@lucjuggery"
tags: [FR]
categories: "Swarm"
terms: 3
---


Note: **un problème d'affichage nécessite de redimensionner la fenêtre du navigateur afin que les 3 terminaux utilisés s'affichent correctement.**

## Présentation

Swarm est la solution de clustering de Docker et permet entre autres:
- le management d'un cluster d'hôtes Docker
- l'orchestration de services sur ce cluster

Un cluster Swarm est composé de 2 types de nodes:
- les `managers` dont le rôle est de maintenir l'état du cluster. En interne, une implémentation de l'algorithme de consensus `Raft` est utilisée pour assurer cette fonction.
- les `workers` dont le rôle est d'exécuter des jobs (= lancer des containers)

Note: par défaut, un manager est également un worker.

Le schéma suivant montre la vision haut niveau d'un cluster Swarm.

![Voting App]({{ site.baseurl }}/images/swarm-2.png)

Par défaut, un Swarm est sécurisé:
- encryption des logs nécessaires à la gestion du cluster
- communication TLS entre les différents nodes
- auto-rotation des certificats

![Voting App]({{ site.baseurl }}/images/swarm-1.png)

## Création

Nous allons créer un cluster Swarm composé d'un manager et de 2 workers.

La commande suivante est suffisante pour créer un cluster Swarm. Ce cluster ne comporte qu'un seul node pour le moment.

```.term1
docker swarm init --advertise-addr $(hostname -i)
```

Nous obtenons le résultat suivant:

```
Swarm initialized: current node (preife0qe9vjyum4rv13qm33l) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-0jo31iectxf4uo4airmn1cepphe9mbg4j8j6276as56i6gi82c-8ggnqa3165gb0x8idf8tqs68p \
    10.0.107.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Le daemon Docker de ce terminal est maintenant en Swarm mode, nous pouvons lister les nodes de notre cluster.

```.term1
docker node ls
```

```
ID                            HOSTNAME  STATUS   AVAILABILITY MANAGER STATUS
preife0qe9vjyum4rv13qm33l *   node1     Ready    Active       Leader
```

Nous voyons ici que le node1 fonctionne correctement et qu'il est le `Leader` des nodes managers (un seul manager pour le moment).

Depuis les 2 autres terminaux, nous lançons la commande `docker swarm join...` obtenue lors de l'initialisation du Swarm.

```
docker swarm join \
    --token SWMTKN-1-0jo31iectxf4uo4airmn1cepphe9mbg4j8j6276as56i6gi82c-8ggnqa3165gb0x8idf8tqs68p \
    10.0.107.3:2377
```

Nous obtenons alors la réponse suivante

```
This node joined a swarm as a worker.
```

Depuis le premier terminal, nous pouvons alors lister les nodes présents dans notre cluster.

```.term1
docker node ls
```

Aux IDs prêts, nous obtenons la liste suivante, montrant que le cluster contient maintenant 3 nodes.

```
ID                            HOSTNAME  STATUS   AVAILABILITY  MANAGER STATUS
pgmk0zublslbmh2dqpbq0zs7i     node2     Ready    Active
preife0qe9vjyum4rv13qm33l *   node1     Ready    Active        Leader
u8ngxoxj2tvto18kzemefm9vo     node3     Ready    Active
```

Le node1 est l'unique manager du cluster. node2 et node3 sont des workers.

Note: toutes les commandes du client Docker doivent être envoyées sur un manager, si nous lançons la commande précédente sur node2 ou node3, nous obtenons l'erreur suivante.

```.term2
docker node ls
```

```
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
```

## Inspection d'un node

Comme pour les autres primitives de la plateforme (containers, images, volumes, networks), la commande inspect permet d'obtenir une vue détaillée d'un node.

```.term1
docker node inspect node1
```

```
[                                                                                                        
    {                                                                                                    
        "ID": "preife0qe9vjyum4rv13qm33l",                                                               
        "Version": {                                                                                     
            "Index": 8                                                                                   
        },                                                                                               
        "CreatedAt": "2017-06-21T06:40:21.962915021Z",                                                   
        "UpdatedAt": "2017-06-21T06:40:22.469719295Z", 
        "Spec": {
            "Labels": {},
            "Role": "manager",
            "Availability": "active"
        },
        "Description": {
            "Hostname": "node1",
            "Platform": {
                "Architecture": "x86_64",
                "OS": "linux"
            },
            "Resources": {
                "NanoCPUs": 8000000000,
                "MemoryBytes": 64424685568
            },
            "Engine": {
                "EngineVersion": "17.05.0-ce",
                "Plugins": [                                                                             
                    {                                                                                    
                        "Type": "Network",                                                               
                        "Name": "bridge"                                                                 
                    },                                                                                   
                    {                                                                                    
                        "Type": "Network",                                                               
                        "Name": "host"                                                                   
                    },
                    {
                        "Type": "Network",
                        "Name": "ipvlan"
                    },
                    {
                        "Type": "Network",
                        "Name": "macvlan"
                    },
                    {
                        "Type": "Network",
                        "Name": "null"
                    },
                    {
                        "Type": "Network",
                        "Name": "overlay"
                    },
                    {
                        "Type": "Volume",
                        "Name": "local"
                    }
                ]
            }
        },
        "Status": {
            "State": "ready",
            "Addr": "10.0.107.3"
        },
        "ManagerStatus": {
            "Leader": true,
            "Reachability": "reachable",
            "Addr": "10.0.107.3:2377"
        }
    }
]
```

Cette commande retourne entre autres le status du node, son type (manager vs worker), ...

Nous pouvons utiliser les Go template afin d'obtenir une information précise contenue dans cette structure json. Par exemple, la commande suivante permet de récupérer l'adresse IP du node directement.

```.term1
docker node inspect -f "{{ "{{ .Status.Addr "}}}}" node1
```

## Mise à jour d'un node
 
Une fois les nodes crées, il est possible de changer leur type et de:
- promouvoir un worker en manager
- destituer un manager en worker

### Promotion de node2

```.term1
docker node promote node2
```

```
Node node2 promoted to a manager in the swarm.
```

Les commandes docker peuvent être maintenant lancée depuis node1 ou node2 puisque tous 2 sont des managers.

Depuis le node1:
```.term1
docker node ls
```

Depuis le node2:
```.term2
docker node ls
```

Les 2 commandes donnent le résultat suivant:

```
ID                            HOSTNAME  STATUS   AVAILABILITY   MANAGER STATUS
pgmk0zublslbmh2dqpbq0zs7i     node2     Ready    Active         Reachable
preife0qe9vjyum4rv13qm33l *   node1     Ready    Active         Leader
u8ngxoxj2tvto18kzemefm9vo     node3     Ready    Active
```

Le node2 a l'entrée `Reachable` dans la colonne `MANAGER STATUS`, ce qui signifie qu'il est du type `manager`, mais pas `Leader`.

### Destitution du node2

Depuis node1 ou node2, nous pouvons destituer le node2 et le repasser en worker.

```.term1
docker node demote node2
```

```
Manager node2 demoted in the swarm.
```
    
Si nous listons une nouvelle fois les nodes du cluster, nous pouvons voir que le node2 n'a plus d'entrée dans la colonne `MANAGER STATUS`.

```.term1
docker node ls
```

```
ID                            HOSTNAME  STATUS   AVAILABILITY   MANAGER STATUS
pgmk0zublslbmh2dqpbq0zs7i     node2     Ready    Active         
preife0qe9vjyum4rv13qm33l *   node1     Ready    Active         Leader
u8ngxoxj2tvto18kzemefm9vo     node3     Ready    Active
```

## Service

Dans cette partie, nous allons créer un service simple, le déployer et le scaler sur le cluster Swarm. Nous changerons également la disponibilité d'un node pour voir l'impact sur les tâches en cours.


### Création d'un service

La commande suivante permet de définir un service:
- basé sur l'image `instavote/vote:latest`
- nommé `vote`
- dont le port 80 est publié sur le port 8080 de chaque hôte du cluster
- ayant 6 réplicas

Note: le nombre de réplicas correspond au nombre de tâches qui seront instanciées pour ce service. Chaque tâche lancera un container basée sur l'image définie dans le service.

```.term1
docker service create \
    --name vote \
    --publish 8080:80 \
    --replicas 6 \
    instavote/vote
```

Après quelques secondes, le temps que l'image `instavote/vote:latest` soit téléchargée sur chaque node, les 6 tâches du services sont lancées. La commande suivante liste les tâches du service `vote`. Nous pouvons voir que 2 tâches ont été lancées sur chaque node.

```.term1
docker service ps vote
```

```
ID            NAME    IMAGE                   NODE   DESIRED STATE  CURRENT STATE          ERROR PORTS
xnon20jonsfd  vote.1  instavote/vote:latest   node2  Running        Running 13 minutes ago
rganh2g8y8b7  vote.2  instavote/vote:latest   node3  Running        Running 13 minutes ago             
on8oog1833yq  vote.3  instavote/vote:latest   node1  Running        Running 13 minutes ago             
hmp2wtvojxro  vote.4  instavote/vote:latest   node2  Running        Running 13 minutes ago             
vdizjy291q4t  vote.5  instavote/vote:latest   node3  Running        Running 13 minutes ago             
mjpn0ybsg6pj  vote.6  instavote/vote:latest   node1  Running        Running 13 minutes ago 
```

Le service publie le port 80 sur le port 8080 du swarm via le mecanisme de routing mesh. Cela signifie que le service sera accessible depuis le port 8080 de chaque node du cluster. Nous pouvons vérifier le routing mesh en envoyant une requête sur le port 8080 de node1, node2 ou node3. Quelque soit le node sur lequel la requête est envoyée, le résultat est le même: l'interface permettant la selection entre `cats` et `dogs`.

![Cats vs Dogs]({{ site.baseurl }}/images/s1-vote.png)

Note: seule la partie front-end est disponible sur cet exemple, il n'y a pas le backend permettant de prendre en compte la sélection.

Les requêtes envoyées sur le port 8080 d'un node du Swarm sont traitées selon un algorithme de round-robin entre les différents containers intanciés pour le service. Nous pouvons l'observer en regardant l'ID du container sur l'interface web.

[Envoie d'une requête sur le node1](/){:data-term=".term1"}{:data-port="8080"}

[Envoie d'une requête sur le node2](/){:data-term=".term1"}{:data-port="8080"}

[Envoie d'une requête sur le node3](/){:data-term=".term1"}{:data-port="8080"}

### Ajout du service de visualisation

La commande suivante défini un service qui servira à la visualisation des containers sur le cluster.

```.term1
docker service create \
    --name visualizer \
    --mount type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock \
    --constraint 'node.role == manager' \
    --publish "8000:8080" dockersamples/visualizer:stable
```
 
La définition de ce service contient les informations suivantes:
- le nom du service: visualizer
- bind-mount de la socket du daemon docker (afin de permettre au container du visualizer de dialoguer avec le daemon Docker sur lequel il tourne)
- specification d'une contrainte de déploiement pour que le replica du service tourne sur le node qui a le rôle `manager`
- publication du port 8080 du container sur le port 8000 sur chaque hôte du Swarm

Nous pouvons alors voir la répartition des containers sur les nodes via [l'interface du visualiseur](/){:data-term=".term1"}{:data-port="8000"}

L'interface donne le même résultat que lorsque l'on liste les service en ligne de commande, mais c'est une façon assez ludique de visualiser les containers.

![Visualiseur]({{ site.baseurl }}/images/s1-visualiseur-1.png)

### Passage du node2 en `drain`

Un node est dans l'un des états suivants:

- `active`, il peut recevoir des nouvelles tâches
- `pause`, il ne peut pas recevoir de nouvelles tâches mais les tâches en cours restent inchangées
- `drain`, il ne peut plus recevoir de nouvelles tâches et les tâches en cours sont re-schédulées sur d'autres nodes

La commande suivante change l'`availability` du node2 et le met dans l'état `drain`.

```.term1
docker node update --availability drain node2    
```

Regardons comment la répartition des tâches précédentes a été modifiée.

![Visualiseur]({{ site.baseurl }}/images/s1-visualiseur-2.png)

Nous pouvons voir qu'il n'y a plus de tâche sur le node2. Elles ont été stoppées et reschédulées sur les autres nodes.
 
La commande suivante change l'`availability` du node2 en le repositionnant dans l'état `active`. Les tâches existantes sur node1 et node3 restent inchangées mais de nouvelles tâches peuvent de nouveau être schédulées sur node2.

```.term1
docker node update --availability active node2
```

## Rolling upgrade

Nous allons à présent modifier paramêtres du service vote de façon à définir un rolling upgrade qui mettra les tâches à jour 2 par 2 toutes les 15 secondes. Par défaut, les tâches sont mises à jour l'une après l'autre.

Pour faire cela, nous définissons les paramêtres `--update-parallelism` que nous fixons à 2, ainsi que `--update-delay` que nous fixons à 15s.

```.term1
docker service update \
    --update-parallelism 2 \
    --update-delay 15s \
    vote
```

Une fois ces paramêtres modifiés, nous mettons à jour l'image utilisée pour le service. La nouvelle version utilisée aura le tag `indent` au lieu de `latest`.

```.term1
docker service update --image instavote/vote:indent vote
```

Si nous observons [l'interface du visualiseur](/){:data-term=".term1"}{:data-port="8000"} nous pouvons observer que la mise à jour se fait de façon graduelle.

![Visualiseur]({{ site.baseurl }}/images/s1-visualiseur-3.png)

Après quelques secondes, tous les containers de l'application sont basés sur l'image ayant le tag `intend`.

![Visualiseur]({{ site.baseurl }}/images/s1-visualiseur-4.png)

## Résumé

Nous avons vu dans ce labs de nombreuses commandes pour la gestion du Swarm et la modification du role et de l'availability des nodes qui le composent. Nous avons également crée un service que nous avons mis à jour en définissant des paramêtres de `rolling upgrade`.
