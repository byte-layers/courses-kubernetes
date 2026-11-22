---
title: "Ce que Kubernetes apporte : un standard, la haute disponibilité, la mise à l'échelle"
description: "Ce que Kubernetes apporte, scène par scène sur l'application de vote : le standard OCI et la portabilité, l'état voulu dans Git et la démarche SRE, la haute disponibilité quand un nœud tombe (et le piège d'un cluster d'une seule machine), la mise à l'échelle sur plusieurs nœuds, la CNCF et le cloud native."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 12
requires: "0.8.0"
---

Kubernetes est un orchestrateur de conteneurs : il fait tourner une application sur un groupe de machines, le **cluster**, et la maintient dans l'état qu'on lui a décrit. Cette page montre **ce qu'il apporte**, en cinq idées, chacune avec sa scène : un standard, l'état voulu dans Git, une machine qui tombe, la charge qui monte, et tout un écosystème. L'exemple est l'application de vote : `vote`, la page où l'on vote ; `redis`, qui garde les votes en attente ; le `worker`, qui les enregistre ; `db`, la base de données ; et `result`, la page des résultats.

**Pour qui ?** Pour tout le monde, y compris les profils non techniques. Rien à installer, rien à lancer.

**Ce qui est supposé** : savoir ce qu'est un conteneur, et pourquoi on confie des conteneurs à un orchestrateur, ce que raconte [Pourquoi un orchestrateur](https://byte-layers.com/fr/bits/vm-to-orchestrator).

# Ce que Kubernetes apporte

## Un standard : les mêmes images, partout

Une application en conteneurs, c'est une image par pièce et une description de l'ensemble. Où peut-elle tourner ?

```scene
lesson: assets/k8s-benefits.scene.json
step: standard
```

Les images suivent le **standard OCI** (*Open Container Initiative*), lancé en 2015 sous l'égide de la Linux Foundation : il fixe le format des images, leur exécution et leur distribution par un registre. La description est écrite pour Kubernetes. Le même paquet tourne sur vos machines, chez un fournisseur ou sur votre poste. Seuls quelques réglages changent, comme le stockage ou la porte d'entrée : c'est la **portabilité**.

## L'état voulu dans Git : une démarche SRE

Avec Kubernetes, on ne donne pas les ordres un par un : on décrit ce qu'on veut obtenir.

```scene
lesson: assets/k8s-benefits.scene.json
step: desired-state
```

L'**état voulu**, combien de copies et quelle version, est écrit dans des fichiers rangés dans Git. Une commande, `kubectl apply`, ou un outil comme Argo CD les applique au cluster. Kubernetes compare cet état à ce qui tourne et corrige l'écart : c'est la **réconciliation**. Chaque changement laisse une trace, chaque pièce s'observe. C'est la démarche **SRE** (*Site Reliability Engineering*, l'ingénierie de la fiabilité) : une exploitation décrite, automatisée et observée.

## Une machine tombe : la haute disponibilité

Un cluster, ce sont plusieurs machines, les **nœuds**. Que se passe-t-il quand l'une s'arrête ?

```scene
lesson: assets/k8s-benefits.scene.json
step: node-down
```

Les copies de `vote` des autres nœuds **continuent de répondre**, et Kubernetes relance ailleurs ce que portait le nœud perdu. C'est la **haute disponibilité**, à une condition : plusieurs copies, sur plusieurs nœuds. Par défaut, il attend **environ cinq minutes** avant de relancer, au cas où le nœud reviendrait.

> 💡 **Remarque.** Une base de données ne repart avec ses données que si elles vivent sur un stockage que les autres nœuds peuvent joindre.

## Le piège : Kubernetes sur une seule machine

Ces gains supposent plusieurs nœuds. La tentation est d'installer Kubernetes sur une seule machine, « comme un Docker de plus ».

```scene
lesson: assets/k8s-benefits.scene.json
step: single-node
```

Sur un seul nœud, Kubernetes **relance** une copie qui s'arrête, remplace les copies une à une lors d'une mise à jour et donne une adresse stable à chaque pièce. Mais si la machine tombe, **rien ne repart** : il n'y a pas d'autre nœud. Un cluster d'un nœud, comme minikube sur votre poste, sert à **apprendre** et à tester, pas à la production.

## La charge monte : la mise à l'échelle sur plusieurs nœuds

Le soir du vote, les requêtes affluent. Une machine plus grosse finit toujours par atteindre sa limite.

```scene
lesson: assets/k8s-benefits.scene.json
step: scale
```

On demande plus de copies, une ligne dans le fichier, et Kubernetes les **répartit** sur les nœuds. Un réglage peut aussi le faire selon la charge, le *HorizontalPodAutoscaler*. Quand les nœuds sont pleins, on en ajoute ; chez un fournisseur, un outil peut les ajouter tout seul. La limite n'est plus la taille d'une machine, mais celle du **cluster**.

## Tout un écosystème : la CNCF et le cloud native

Kubernetes n'est pas seul : il vit au milieu d'outils qui suivent les mêmes règles.

```scene
lesson: assets/k8s-benefits.scene.json
step: ecosystem
```

Créé chez Google et ouvert en 2014, Kubernetes a été confié à la **CNCF** (*Cloud Native Computing Foundation*), une fondation créée en 2015 sous la Linux Foundation. Elle porte aussi containerd, Prometheus, Helm et plus d'une centaine d'autres projets, pensés pour des applications faites de pièces en conteneurs : c'est le **cloud native**. Proposé en service géré par les grands clouds, Kubernetes est **quasiment** le standard de fait, sans être le seul orchestrateur.

## Les cinq gains, et leurs conditions

Le tableau reprend les cinq idées, avec ce que chacune demande.

| Gain | Ce qui le permet | À condition |
|---|---|---|
| **Portabilité** | des images OCI, une description Kubernetes | d'adapter quelques réglages à chaque endroit |
| **Démarche SRE** | l'état voulu dans Git, la réconciliation, la supervision | de toujours passer par les fichiers |
| **Haute disponibilité** | la relance sur les autres nœuds | d'avoir plusieurs nœuds et plusieurs copies |
| **Mise à l'échelle** | des copies réparties, des nœuds ajoutés | d'avoir de la place dans le cluster, ou d'en ajouter |
| **Écosystème** | la CNCF et ses projets | de choisir parmi beaucoup d'outils |

> 📖 [Kubernetes : vue d'ensemble](https://kubernetes.io/fr/docs/concepts/overview/) · [La gestion déclarative des objets](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/) · [L'autoscaling horizontal](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) · [L'Open Container Initiative](https://opencontainers.org/) · [La CNCF](https://www.cncf.io/)

## Trois questions pour vérifier ce que vous avez retenu

```quiz
title: "Ce que Kubernetes apporte"
tracksProgress: false
showNotice: false
place: stage
questions:
  - prompt: "Une équipe a construit ses images avec Docker. Elle veut les faire tourner sur un cluster Kubernetes chez un fournisseur. Que doit-elle faire de ses images ?"
    answers:
      - "Rien de particulier : elles suivent le standard OCI, que les moteurs des nœuds savent lancer"
      - "Les reconstruire avec un outil propre à Kubernetes"
      - "Les convertir au format du fournisseur"
    correct: 1
    explanation: "Le standard OCI fixe le format des images, leur exécution et leur distribution. Une image construite avec Docker est lancée par containerd ou CRI-O sur un nœud Kubernetes, chez vous comme chez un fournisseur."
  - prompt: "Un cluster de trois nœuds fait tourner trois copies de vote, une par nœud. Un nœud tombe. Que se passe-t-il ?"
    answers:
      - "L'application s'arrête jusqu'à ce qu'on répare le nœud"
      - "Les deux autres copies répondent, et Kubernetes relance la troisième sur un nœud qui reste"
      - "Kubernetes répare la machine en panne"
    correct: 2
    explanation: "Les copies des autres nœuds continuent de servir les votes. Après un délai, environ cinq minutes par défaut, Kubernetes relance ailleurs ce que portait le nœud perdu. La machine, elle, reste à réparer."
  - prompt: "Une équipe installe Kubernetes sur son unique serveur pour avoir la haute disponibilité. L'a-t-elle obtenue ?"
    answers:
      - "Oui : Kubernetes relance tout ce qui s'arrête"
      - "Oui, à condition de lancer plusieurs copies de chaque pièce"
      - "Non : si le serveur tombe, il n'y a aucun autre nœud où relancer les copies"
    correct: 3
    explanation: "Sur un seul nœud, Kubernetes relance une copie qui s'arrête, mais pas une machine qui tombe. Plusieurs copies sur le même serveur tombent avec lui. Un cluster d'un nœud sert à apprendre et à tester."
```

## Récap

- Les images suivent le standard **OCI** et la description est celle de Kubernetes : la même application tourne sur vos machines, chez un fournisseur ou sur votre poste, à quelques réglages près.
- L'**état voulu** est décrit dans des fichiers rangés dans Git ; Kubernetes compare et corrige l'écart. Une exploitation décrite, automatisée et observée : la démarche **SRE**.
- Quand un nœud tombe, les autres copies répondent et Kubernetes relance ailleurs ce qu'il portait : la **haute disponibilité**, s'il y a plusieurs nœuds.
- Sur un seul nœud, rien ne repart quand la machine tombe : utile pour apprendre, pas pour la production.
- Quand la charge monte, les copies se répartissent sur plusieurs nœuds, et le cluster peut grandir.
- Porté par la **CNCF**, entouré d'un écosystème **cloud native**, Kubernetes est quasiment le standard de fait.
- Ces gains ont un prix : [Ce que Kubernetes demande : attention au choix](https://byte-layers.com/fr/bits/k8s-when-to-use) dit lequel, et quand Kubernetes n'est pas le bon choix.

```ack
J'ai compris
```
