---
layout: post
title:  "Les cgroups"
date:   2017-06-17
author: "@lucjuggery"
tags: [FR]
categories: Containers
---

Ce labs montre l'utilisation des cgroups pour limiter la mémoire allouée à un processus.

## Liste des cgroups présents

La commande suivante liste les cgroups que nous pouvons utiliser afin d'imposer des contraintes d'utilisation de resources à un processus.

```.term1
ls /sys/fs/cgroup
```

```
blkio             devices           net_cls,net_prio
cpu               freezer           net_prio
cpu,cpuacct       hugetlb           perf_event
cpuacct           memory            pids
cpuset            net_cls           systemd
```

Dans l'exemple suivante nous allons nous focaliser sur le cgroups `memory` afin d'imposer une limite maximale de RAM.

## Création d’un cgroups mémoire

```.term1
mkdir /sys/fs/cgroup/memory/my_group
```

`/sys/fs/cgroups/memory` est un speudo filesystem. La création du répertoire `my_group` créé automatiquement les sous répertoires suivante

```.term1
ls /sys/fs/cgroup/memory/my_group
```

```
cgroup.clone_children
cgroup.event_control
cgroup.procs
memory.failcnt
memory.force_empty
memory.kmem.failcnt
memory.kmem.limit_in_bytes
memory.kmem.max_usage_in_bytes
memory.kmem.slabinfo
memory.kmem.tcp.failcnt
memory.kmem.tcp.limit_in_bytes
memory.kmem.tcp.max_usage_in_bytes
memory.kmem.tcp.usage_in_bytes
memory.kmem.usage_in_bytes
memory.limit_in_bytes
memory.max_usage_in_bytes
memory.move_charge_at_immigrate
memory.numa_stat
memory.oom_control
memory.pressure_level
memory.soft_limit_in_bytes
memory.stat
memory.swappiness
memory.usage_in_bytes
memory.use_hierarchy
notify_on_release
tasks
```

## Limite d'utilisation de la RAM

Afin d'imposer une limite d'utilisation de la RAM, nous settons le champ `memory.limit_in_bytes` à 20M

```.term1
echo -n 20M > /sys/fs/cgroup/memory/my_group/memory.limit_in_bytes
```

Vérifions que la limite a bien été settée:
```.term1
cat /sys/fs/cgroup/memory/my_group/memory.limit_in_bytes
```
```
20971520
```

## Création d’un processus de test

Le processus suivant va consommer de plus en plus de RAM avec le temps

```.term1
A="A"; while true; do A="$A$A"; sleep 3; done &
```

```
[1] 151
```

## Ajout du processus dans my_group

Nous ajoutons le PID du processus précédent (151 dans cet exemple) dans le group `my_group` afin que ce dernier soit contraint par la limite max de RAM.
Note: comme nous avons lancé le processus en background, nous récupérons son PID avec la variable `$!`.

```.term1
echo -n $! > /sys/fs/cgroup/memory/my_group/tasks
```

## Suivi de l’utilisation de la RAM

La commande suivante permet de suivre l'évolution de l'utilisation de la RAM.

```.term1
watch tail /sys/fs/cgroup/memory/my_group/memory.usage_in_bytes
```

Au bout de quelques secondes, le niveau d'utilisation de la RAM n'augmente plus, le processus étant contraint par la limite imposée dans le cgroups `my_group`.

```
Every 2s: tail /sys/fs/cgroup/memory/my_group/memory.usage_in_bytes 2017-06-18 05:32:25

20971520
```

Note: il est nécessaire de faire un `CTRL-C` dans le terminal afin de sortir du processus `watch` lancé ci-dessus.
