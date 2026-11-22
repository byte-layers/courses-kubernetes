---
title: "Pourquoi un orchestrateur : de la machine au cluster"
description: "Déployer l'application de vote sur une machine virtuelle, puis deux, puis en conteneurs : ce que chaque marche règle et ce qu'elle laisse à faire. Puis le rôle d'un orchestrateur, et les deux suites possibles : Docker Swarm ou Kubernetes."
tags:
  - bit
  - containers
  - docker
  - kubernetes
difficulty: beginner
duration: 15
requires: "0.8.0"
---

Un orchestrateur de conteneurs est un logiciel qui fait tourner des conteneurs sur un groupe de machines, le **cluster**, et qui les maintient dans l'état qu'on lui a décrit. Cette page répond à une question simple : **pourquoi s'en servir ?** Pour y répondre, nous élaborons le déploiement d'une application, l'application de vote, sur des marches successives, de la plus simple à la plus outillée : une machine, deux machines, des conteneurs, puis un orchestrateur. Chaque marche règle un problème de la précédente, et en apporte un nouveau.

**Pour qui ?** Pour tout le monde, et en particulier pour les profils non techniques. Il n'y a rien à installer ni à lancer : ni cluster, ni terminal, et aucun fichier n'est créé sur votre machine.

**Ce qui est supposé** : rien de particulier. L'application de vote est présentée dans la première section ; pour la voir tourner en conteneurs, [Échauffement : la voting app en conteneurs](https://byte-layers.com/fr/labs/voting-app-containers-warmup) la lance pas à pas.

# Pourquoi un orchestrateur : de la machine au cluster

## L'application à déployer

Voici l'application dont nous élaborons le déploiement. Le bouton **Jouer** d'une scène l'ouvre en grand, et les pastilles roses donnent le détail de chaque pièce.

```scene
lesson: assets/vm-to-orchestrator-plan.scene.json
step: voting-app
```

L'application de vote a **cinq pièces** : `vote`, la page où l'on vote ; `redis`, qui garde les votes en attente ; le `worker`, qui les enregistre ; `db`, la base de données ; et `result`, la page des résultats. Elles sont écrites dans **trois langages** : Python, .NET et Node.js. Toute la question est de savoir où et comment les faire tourner.

## Une seule machine virtuelle qui fait tout

Une machine virtuelle, ou VM, est un ordinateur simulé par un logiciel : on la loue chez un hébergeur, ou on la crée sur un serveur de l'entreprise. Ici, tout est installé à la main sur la même VM : les cinq pièces de l'application, et tout ce qu'elles demandent, Python, .NET, Node.js, PostgreSQL et Redis. Un manuel de déploiement d'une quarantaine de pages dit dans quel ordre taper les quelque 150 commandes. Ces chiffres sont inventés, mais l'ordre de grandeur est réaliste.

```scene
lesson: assets/vm-to-orchestrator.scene.json
step: single-vm
```

Ce que cette marche permet, et ce qu'elle ne permet pas, tient en trois lignes.

| Besoin | Sur une seule VM |
|---|---|
| Grossir la machine, ce qu'on appelle la mise à l'échelle verticale (*vertical scaling*) : plus de processeur, plus de mémoire | facile, mais il faut redémarrer, et il y a une limite |
| Multiplier les machines, la mise à l'échelle horizontale (*horizontal scaling*) | difficile : il faut cloner la VM et ajouter un répartiteur de charge (*load balancer*), la pièce qui distribue les requêtes entre les copies… et que devient la base de données ? |
| Sauvegarder | une copie de la VM entière, un *snapshot*, et un seul point de défaillance : si la machine tombe, tout tombe |

Reste la haute disponibilité, c'est-à-dire une application qui continue de répondre quand une machine tombe. Il faudrait une seconde VM, un répartiteur de charge, une bascule de l'une vers l'autre, des sauvegardes, un plan de reprise : rien de cela n'existe encore, tout reste à concevoir et à documenter.

## Plusieurs machines, plusieurs systèmes

Imaginez maintenant qu'une pièce impose son système : un ancien worker écrit en .NET Framework, l'ancienne version de .NET, qui ne tourne que sur Windows Server. Il faut une seconde VM, et l'application se répartit sur deux machines. Ce scénario est inventé, mais il est très courant.

```scene
lesson: assets/vm-to-orchestrator.scene.json
step: two-vms
```

Les ennuis de la première marche sont toujours là, **en double**. Le tableau en fait le compte.

| Besoin | Sur deux VM |
|---|---|
| Relier les pièces | des adresses IP écrites en dur d'une VM à l'autre : si une machine change d'adresse, on corrige la configuration à la main |
| Tenir les systèmes à jour | deux systèmes, deux expertises |
| Déployer, sauvegarder | deux manuels de déploiement, deux sauvegardes |
| Multiplier les machines | VM par VM, encore plus difficile |
| Haute disponibilité, plan de reprise | à concevoir deux fois |

## Les conteneurs : un seul outil à installer

Chaque pièce devient une image, rangée dans un registre : Docker Hub, `ghcr.io`, ou un registre interne à l'entreprise. Sur la VM, on n'installe plus qu'une chose, le moteur de conteneurs (le *runtime*) : ici Docker, avec Compose. Le manuel de quarante pages devient un fichier, `compose.yml`, et une commande, `docker compose up`.

```scene
lesson: assets/vm-to-orchestrator.scene.json
step: containers
```

Le déploiement est **standardisé**, et il devient **reproductible** : le même paquet sert partout, sur un nouveau système, sur une VM de plus ou pour un environnement de démonstration. Il suffit d'y installer le moteur. Grossir la machine reste facile. Multiplier les copies devient possible, comme les trois copies de `vote` de la scène, à condition de gérer soi-même le routage avec un reverse proxy, un serveur placé devant les copies, qui reçoit les requêtes et les leur répartit. Mais il n'y a toujours qu'une machine : si elle tombe, tout tombe.

Multiplier les copies fait aussi apparaître de nouvelles questions. Qui distribue les requêtes ? Comment une pièce trouve-t-elle les autres ? Que se passe-t-il si la machine tombe ? Comment faire la même chose sur plusieurs machines ? Ce sont les questions auxquelles répond un orchestrateur.

## Les conteneurs, quand il reste deux systèmes

Reprenons les deux machines de la marche précédente, avec la même répartition, et passons-les en conteneurs : sur chacune, un moteur de conteneurs, un fichier Compose et un reverse proxy.

```scene
lesson: assets/vm-to-orchestrator.scene.json
step: containers-two-hosts
```

La VM Linux porte `vote`, en trois copies derrière son proxy, et `redis`. La VM Windows porte le `worker`, `db` et `result`, avec **son propre proxy** devant `result`. Garder deux systèmes a un coût de plus : sur Docker Hub, les images officielles de PostgreSQL et de Node.js sont prévues pour Linux, et sur un Windows Server il faut **construire soi-même des images Windows** pour `db` et `result`. Le reste se fait à la main.

| Ce qui reste à faire à la main | Pourquoi |
|---|---|
| Tenir **deux configurations** | deux fichiers Compose et deux `.env` ; le mot de passe de la base ne sert que sur la VM Windows (`db`, `worker`, `result`), mais l'adresse de `redis`, le réseau commun et l'accès au registre d'images sont à tenir à jour sur les deux machines |
| Configurer **deux reverse proxies** | un par machine : l'un devant les copies de `vote`, l'autre devant `result` |
| Brancher **le réseau** entre les machines | Compose ne relie que les conteneurs d'une même machine : pour que le `worker` lise `redis` sur l'autre VM, il faut un réseau commun, par exemple un réseau *overlay* de Docker, ou des ports ouverts |
| Garder **la VM Windows** | une image Linux ne tourne que sur un hôte Linux, une image Windows que sur un hôte Windows : la VM Windows reste nécessaire tant que le worker n'est pas porté en .NET moderne, la version actuelle de .NET, qui tourne aussi sous Linux |

## Vers un orchestrateur

Plusieurs machines, des copies à répartir, un réseau à brancher entre elles : c'est le travail d'un **orchestrateur**.

```scene
lesson: assets/vm-to-orchestrator-plan.scene.json
step: orchestrator
```

On décrit l'application **une seule fois**. L'orchestrateur pose les copies sur les machines du cluster, les relie par un réseau commun et **relance ailleurs** ce qui tombe. Plusieurs outils tiennent ce rôle : **Docker Swarm**, intégré à Docker ; **Kubernetes**, et ses distributions (k3s, RKE2, OpenShift, ou les offres des fournisseurs de cloud) ; **Nomad**, de HashiCorp.

> 📖 [Docker Compose](https://docs.docker.com/compose/) · [Docker Swarm](https://docs.docker.com/engine/swarm/) · [Kubernetes : vue d'ensemble](https://kubernetes.io/fr/docs/concepts/overview/)

## Trois questions pour vérifier ce que vous avez retenu

```quiz
title: "Pourquoi un orchestrateur"
tracksProgress: false
showNotice: false
place: stage
questions:
  - prompt: "L'application passe d'une VM installée à la main à des conteneurs lancés par Compose, sur la même VM. Qu'a-t-elle gagné ?"
    answers:
      - "La haute disponibilité : si la VM tombe, l'application continue de répondre"
      - "Un déploiement standardisé et reproductible : un fichier, une commande, et le moteur de conteneurs comme seul outil à installer"
      - "Le réseau entre plusieurs machines, branché tout seul"
    correct: 2
    explanation: "Les conteneurs standardisent le déploiement : une image par pièce, un fichier Compose, une commande. Mais il n'y a toujours qu'une machine : si elle tombe, tout tombe."
  - prompt: "Le worker, écrit en .NET Framework, ne tourne que sous Windows. L'équipe passe l'application en conteneurs. Que devient la VM Windows ?"
    answers:
      - "Elle disparaît : le moteur de conteneurs de la VM Linux lance aussi les images Windows"
      - "Elle reste, pour l'image Windows du worker, tant qu'il n'est pas porté en .NET moderne"
      - "Elle reste, et toutes les pièces doivent y être déplacées"
    correct: 2
    explanation: "Une image Linux ne tourne que sur un hôte Linux, une image Windows que sur un hôte Windows. Tant que le worker n'est pas porté en .NET moderne, il garde sa VM Windows."
  - prompt: "Une machine du cluster tombe. Que fait l'orchestrateur ?"
    answers:
      - "Il relance sur les machines qui restent les copies qui tournaient dessus, d'après la description de l'application"
      - "Il attend qu'un administrateur redémarre la machine"
      - "Il arrête toute l'application, pour éviter des données incohérentes"
    correct: 1
    explanation: "L'orchestrateur compare ce qui tourne à la description de l'application. Quand une machine tombe, il repose ses copies ailleurs, à condition qu'il reste des machines pour les accueillir."
```

## Récap : vers Docker Swarm ou Kubernetes

- L'application ne change pas d'une marche à l'autre : c'est la façon de la déployer et de l'exploiter qui change.
- Les conteneurs **standardisent le déploiement** : une image par pièce, un moteur à installer, un fichier et une commande. Ils ne donnent pas la haute disponibilité, et sur deux machines, la configuration, le réseau et le système de chaque hôte restent à gérer à la main.
- Un **orchestrateur** règle ce reste : l'application est décrite une fois, il pose les copies sur plusieurs machines, les relie par un réseau commun et relance ailleurs ce qui tombe.
- Deux suites sont courantes, et le choix dépend de l'équipe et de l'application. **Docker Swarm** est intégré à Docker et reprend le format de Compose : il est **simple** à prendre en main, voir [Déployer sur un cluster Swarm](https://byte-layers.com/fr/skills/docker-swarm). **Kubernetes** est **plus riche**, et **plus exigeant** en compétences et en infrastructure, voir [Ce que Kubernetes apporte](https://byte-layers.com/fr/bits/k8s-benefits).
- Il en existe d'autres, comme **Nomad**, de HashiCorp : l'outil change, le rôle reste le même. [Pourquoi Kubernetes : face à Swarm, Mesos et Nomad](https://byte-layers.com/fr/bits/why-kubernetes) les compare.

```ack
J'ai compris
```
