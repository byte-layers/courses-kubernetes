---
title: "De la VM à Kubernetes : ce qui change vraiment"
description: "La même application de vote, exploitée de six façons : une machine virtuelle, puis deux, des conteneurs, et enfin un cluster. Ce que chaque marche règle, ce qu'elle laisse à faire, et le piège du cluster d'une seule machine."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 15
requires: "0.5.0"
---

Quand on change la façon d'héberger une application, l'application reste la même : seule l'infrastructure change. Cette page suit une même application, l'application de vote, exploitée de six façons, de la plus simple à la plus outillée. Chaque marche règle un problème de la précédente, et en apporte un nouveau. Vous saurez ce que les conteneurs changent par rapport à une machine virtuelle, ce que Kubernetes ajoute aux conteneurs, et pourquoi l'installer sur une seule machine ne donne pas ce qu'on en attend.

**Pour qui ?** Pour tout le monde, et en particulier pour les profils non techniques. Il n'y a rien à installer ni à lancer : ni cluster, ni terminal, et aucun fichier n'est créé sur votre machine.

**Ce qui est supposé** : vous connaissez l'application de vote, que vous avez lancée en conteneurs dans [Échauffement : la voting app en conteneurs](https://byte-layers.com/fr/labs/voting-app-containers-warmup), puis sur un cluster dans [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app). Vous avez vu un cluster réparer une application et la mettre à jour, ce qu'enseignent [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment) et [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update).

# De la VM à Kubernetes : ce qui change vraiment

L'application de vote a cinq pièces : `vote`, la page où l'on vote ; `redis`, qui garde les votes en attente ; le `worker`, qui les enregistre ; `db`, la base de données ; et `result`, la page des résultats. Les six sections qui suivent ont chacune leur scène : le bouton **Jouer** l'ouvre en grand, et les pastilles roses donnent le détail de chaque pièce.

## Une seule machine virtuelle qui fait tout

Une machine virtuelle, ou VM, est un ordinateur simulé par un logiciel : on la loue chez un hébergeur, ou on la crée sur un serveur de l'entreprise. Ici, tout est installé à la main sur la même VM : les cinq pièces de l'application, et tout ce qu'elles demandent, Python, .NET, Node.js, PostgreSQL et Redis. Un manuel de déploiement d'une quarantaine de pages dit dans quel ordre taper les quelque 150 commandes. Ces chiffres sont inventés, mais l'ordre de grandeur est réaliste.

```scene
lesson: assets/vm-to-kubernetes.scene.json
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
lesson: assets/vm-to-kubernetes.scene.json
step: two-vms
```

Les ennuis de la première marche sont toujours là, en double :

- les adresses IP sont écrites en dur d'une VM à l'autre : si une machine change d'adresse, il faut corriger la configuration à la main ;
- il y a deux systèmes à tenir à jour, et deux expertises à avoir ;
- il y a deux manuels de déploiement, et deux sauvegardes à faire ;
- multiplier les machines se fait VM par VM, ce qui est encore plus difficile ;
- la haute disponibilité et le plan de reprise sont à concevoir deux fois.

## Les conteneurs : un seul outil à installer

Chaque pièce devient une image, rangée dans un registre : Docker Hub, `ghcr.io`, ou un registre interne à l'entreprise. Sur la VM, on n'installe plus qu'une chose, le moteur de conteneurs (le *runtime*) : ici Docker, avec Compose. Le manuel de quarante pages devient un fichier, `compose.yml`, et une commande, `docker compose up`.

```scene
lesson: assets/vm-to-kubernetes.scene.json
step: containers
```

Le déploiement est standardisé, et il devient reproductible : le même paquet sert partout, sur un nouveau système, sur une VM de plus ou pour un environnement de démonstration. Il suffit d'y installer le moteur. Grossir la machine reste facile. Multiplier les copies devient possible, comme les trois copies de `vote` de la scène, à condition de gérer soi-même le routage avec un reverse proxy, un serveur placé devant les copies, qui reçoit les requêtes et les leur répartit. Mais il n'y a toujours qu'une machine : si elle tombe, tout tombe.

Multiplier les copies fait aussi apparaître de nouvelles questions. Qui distribue les requêtes ? Comment une pièce trouve-t-elle les autres ? Que se passe-t-il si la machine tombe ? Comment faire la même chose sur plusieurs machines ? Ce sont les questions auxquelles répond un orchestrateur, comme le raconte [Pourquoi un orchestrateur](https://byte-layers.com/fr/bits/orchestrator-why).

## Les conteneurs, quand il reste deux systèmes

Reprenons l'application à deux systèmes. Une fois en conteneurs, les pièces Linux se regroupent : la base de données et la page de résultats n'ont plus besoin de Windows.

```scene
lesson: assets/vm-to-kubernetes.scene.json
step: containers-two-hosts
```

> ⚠️ Une image Linux ne tourne que sur un hôte Linux, une image Windows que sur un hôte Windows. La VM Windows reste donc nécessaire tant que le worker n'est pas porté en .NET moderne, la version actuelle de .NET, qui tourne aussi sous Linux.

Il reste deux machines, deux fichiers Compose, et le routage entre elles se règle toujours à la main.

## Le piège : Kubernetes sur une seule machine

La tentation est d'installer Kubernetes sur l'ancienne VM, « comme un Docker de plus ». On obtient un cluster d'un seul nœud, c'est-à-dire Compose avec d'autres commandes.

```scene
lesson: assets/vm-to-kubernetes.scene.json
step: single-node-trap
```

Le tableau fait le compte de ce que ce cluster d'un nœud apporte.

| Gagné | Pas gagné |
|---|---|
| la réparation automatique des pods | la haute disponibilité : il y a toujours **une** machine |
| les mises à jour progressives et le retour arrière | une mise à l'échelle qui dépasse la taille de la VM |
| les Services | |

Un cluster d'un seul nœud est utile pour apprendre, et c'est ce que vous avez utilisé dans ce parcours. Il ne convient pas à la production.

## Un vrai cluster : six décisions à prendre

Passer à un vrai cluster, c'est répartir l'application sur plusieurs machines, et donc décider. La scène montre un exemple installé sur ses propres machines, ce qu'on appelle « sur site » (*on-premise*), avec Rancher, un outil open source édité par SUSE qui crée et gère des clusters, et RKE2, sa distribution de Kubernetes.

```scene
lesson: assets/vm-to-kubernetes.scene.json
step: real-cluster
```

On y trouve trois nœuds pour le plan de contrôle (le *control plane*, la partie du cluster qui décide), pour que le cluster survive à la perte de l'un d'eux. Viennent ensuite un groupe de trois nœuds Linux, sur lesquels l'ordonnanceur répartit les pods, et un nœud Windows pour le worker. Le plan de contrôle est toujours sous Linux : les nœuds Windows ne font que du « travail ». Autour viennent une porte d'entrée, du stockage, des sauvegardes, un registre interne, et encore la supervision, la sécurité et les droits d'accès.

Chaque pièce de ce schéma est une décision :

1. **Comment créer le cluster ?** Un cloud managé, où le fournisseur exploite le plan de contrôle à votre place, Rancher et RKE2, OpenShift, la plateforme de Red Hat, Talos, un système conçu pour Kubernetes…
2. **Quelle topologie ?** La haute disponibilité, le nombre de nœuds, et les zones, des centres de données distincts entre lesquels on répartit les nœuds.
3. **Quel dimensionnement ?** La taille et le nombre des machines.
4. **Linux et Windows ?** Des groupes de nœuds (*node pools*) et des règles de placement, les affinités : dans la scène, un `nodeSelector` envoie le worker sur le nœud Windows.
5. **Les données ?** Dans le cluster, ou dans une base de données managée, en dehors.
6. **Les anciennes VM ?** Supprimées, ou recyclées en nœuds.

## Ce que chaque marche change

Le tableau suivant résume l'échelle que vous venez de parcourir. Lisez la dernière ligne avec autant d'attention que les autres.

| | VM | Conteneurs (Docker, Compose) | Orchestrateur (Kubernetes) |
|---|---|---|---|
| Déployer | un manuel de 40 pages | `docker compose up` | `kubectl apply`, automatisable à partir de Git |
| Nouvel environnement (démonstration, recette…) | tout réinstaller | installer le moteur | un namespace ou un cluster de plus |
| Grossir la machine | facile | facile | des réservations, la taille des nœuds |
| Multiplier les copies | difficile : cloner, router | sur une machine, avec un routage à gérer | natif : des réplicas sur plusieurs nœuds |
| Panne | une intervention humaine | un redémarrage local | une réparation automatique |
| Haute disponibilité | à concevoir | à concevoir | native, s'il y a plusieurs nœuds |
| Coût d'entrée | faible | faible | élevé : compétences, infrastructure, outillage |

Ce coût d'entrée se décide projet par projet : [Kubernetes dans votre projet : quand, et à quel prix](https://byte-layers.com/fr/bits/k8s-when-to-use) en donne la grille.

> 📖 [Docker Compose](https://docs.docker.com/compose/) · [Un cluster pour la production](https://kubernetes.io/docs/setup/production-environment/) · [Les conteneurs Windows dans Kubernetes](https://kubernetes.io/docs/concepts/windows/intro/) · [RKE2](https://docs.rke2.io/)

## Récap

- L'application ne change pas d'une marche à l'autre : c'est la façon de l'exploiter qui change.
- Les conteneurs standardisent le déploiement : une image par pièce, un moteur à installer, un fichier et une commande. Ils ne donnent pas la haute disponibilité.
- Kubernetes installé sur une seule machine répare et met à jour, mais il n'apporte pas la haute disponibilité : c'est le piège.
- Un vrai cluster a plusieurs nœuds, et chaque pièce autour de lui est une décision : la création, la topologie, le dimensionnement, les systèmes, les données, les anciennes VM.
- Le coût d'entrée d'un orchestrateur est élevé : des compétences, une infrastructure, un outillage.

```quiz
prompt: "Une équipe remplace Compose par Kubernetes, sur son unique VM. Qu'a-t-elle gagné ?"
answers:
  - "La haute disponibilité : l'application survit à la panne de la machine"
  - "Rien : un cluster d'un seul nœud ne peut pas faire tourner d'application"
  - "La réparation automatique des pods, les mises à jour progressives et le retour arrière, mais pas la haute disponibilité"
correct: 3
explanation: |
  Un cluster d'un seul nœud fonctionne, et il apporte la réparation automatique, les mises à jour progressives, le retour arrière et les Services. Mais il n'y a toujours qu'une machine : si elle tombe, tout tombe.
```

```ack
J'ai compris
```
