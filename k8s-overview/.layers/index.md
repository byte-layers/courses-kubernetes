---
title: "Kubernetes en un coup d'œil : le cluster, puis ses pièces"
description: "Ce qu'on voit d'abord, un control plane qui décide et des nœuds qui exécutent, puis ce qu'il y a dedans : l'API server, etcd, le scheduler, le kubelet, et les trois interfaces CRI, CNI et CSI."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 12
requires: "0.5.0"
---

Kubernetes a la réputation d'être compliqué, et son vocabulaire y est pour beaucoup : control plane, kubelet, etcd, CRI, CNI… Cette page pose ces mots un à un sur un même schéma, en partant de ce qu'on voit de loin pour finir par ce qu'il y a sous le capot. Vous saurez dire de quoi un cluster est fait, qui décide et qui exécute, et à quoi servent les trois sigles CRI, CNI et CSI.

**Pour qui ?** Pour tout le monde, profils non techniques compris. Il n'y a rien à installer ni à lancer : ni cluster, ni terminal, et aucun fichier n'est créé sur votre machine.

**Ce qui est supposé** : vous savez ce qu'est un conteneur, ce que rappelle [Échauffement : la voting app en conteneurs](https://byte-layers.com/fr/labs/voting-app-containers-warmup), et pourquoi on confie des conteneurs à un orchestrateur, ce que raconte [Pourquoi un orchestrateur](https://byte-layers.com/fr/bits/orchestrator-why).

# Kubernetes en un coup d'œil

Les six sections qui suivent ont chacune leur scène : le bouton **Jouer** l'ouvre en grand, et la pastille rose d'une pièce ouvre son détail. Les trois premières et la dernière montrent ce qu'on voit d'abord. La quatrième suit une demande de bout en bout, et la cinquième ouvre le capot.

## Un cluster : un cerveau, et des machines qui travaillent

Un **cluster** est un groupe de machines pilotées comme une seule. Il a deux parties. Le **control plane** en est le cerveau : il reçoit les demandes, s'en souvient et décide. Les **nœuds** (*nodes*) sont les serveurs qui font tourner les applications, dans des **pods**. Un pod enveloppe un conteneur, parfois plusieurs : c'est la plus petite unité lancée par Kubernetes.

```scene
lesson: assets/k8s-overview.scene.json
step: cluster
```

Vous ne vous connectez à aucune de ces machines. Avec `kubectl`, l'outil en ligne de commande de Kubernetes, vous dites au control plane ce que vous voulez : « trois copies de cette application ». C'est lui qui choisit les nœuds, et vous n'avez pas à savoir lesquels.

On appelle souvent les nœuds qui portent les applications des *workers*, pour les distinguer des machines qui portent le control plane.

## Dans le control plane : quatre pièces, une seule porte

Le cerveau n'est pas d'un seul bloc. Il est fait de quatre pièces, qui ont chacune un seul rôle.

```scene
lesson: assets/k8s-overview.scene.json
step: control-plane
```

| Pièce | Rôle |
|---|---|
| **API server** | la porte unique du cluster : il reçoit toutes les demandes, les vôtres comme celles des autres pièces |
| **etcd** | la mémoire : une base de données où est noté tout ce qu'on a demandé et tout ce qui existe |
| **Scheduler** (l'ordonnanceur) | il choisit sur quel nœud placer chaque nouveau pod |
| **Contrôleurs** (*controller manager*) | ils comparent en permanence ce qui est demandé à ce qui existe, et corrigent l'écart |

Le scheduler et les contrôleurs ne se parlent pas entre eux, et ne touchent pas à etcd : tous passent par l'API server, comme vous. Lui seul lit et écrit dans la mémoire du cluster.

## Dans un nœud : un agent, un moteur, des pods

Chaque nœud porte les mêmes pièces, quel que soit le nombre de pods qu'il fait tourner.

```scene
lesson: assets/k8s-overview.scene.json
step: node
```

| Pièce | Rôle |
|---|---|
| **kubelet** | l'agent du cluster sur la machine : il apprend de l'API server quels pods lui sont confiés, et veille à ce qu'ils tournent |
| **Runtime** | le programme qui crée réellement les conteneurs, containerd le plus souvent. Le kubelet lui délègue ce travail |
| **kube-proxy** | il tient à jour les règles réseau du nœud, pour qu'une requête trouve le bon pod |

Retenez la répartition : **Kubernetes décide quoi lancer et où, le runtime sait comment le lancer.**

## Le trajet d'une demande, de kubectl au conteneur

Vous demandez un pod. Dans la scène, le bouton **pas à pas** fait avancer la demande un temps à la fois.

```scene
lesson: assets/k8s-overview.scene.json
step: request
```

1. `kubectl` envoie votre demande à l'**API server**.
2. L'API server la note dans **etcd** : un pod est demandé, il n'existe pas encore.
3. Le **scheduler** voit ce pod sans nœud, et lui en choisit un.
4. Le **kubelet** de ce nœud apprend qu'un pod lui est confié, et demande au **runtime** de télécharger l'image et de lancer le conteneur.
5. Le kubelet rapporte l'état du pod à l'API server : `Running`.

Personne n'a donné d'ordre à personne. Chaque pièce regarde, par l'API server, ce qui est demandé, et fait sa part pour que la réalité y ressemble.

## Sous le capot : trois prises standard, CRI, CNI, CSI

Kubernetes ne lance pas lui-même les conteneurs, ne câble pas lui-même le réseau, et ne fabrique pas de disques. Il délègue ces trois travaux à des pièces que l'on choisit, et qui se branchent sur trois interfaces standard, comme des appareils sur des prises.

```scene
lesson: assets/k8s-overview.scene.json
step: interfaces
```

| Sigle | Nom complet | Ce qui s'y branche | Exemples |
|---|---|---|---|
| **CRI** | *Container Runtime Interface* | le runtime, qui lance les conteneurs | containerd, CRI-O |
| **CNI** | *Container Network Interface* | le plugin réseau, qui donne une adresse à chaque pod et relie les pods entre eux | Calico, Cilium, Flannel |
| **CSI** | *Container Storage Interface* | le pilote de stockage, qui fournit des disques aux pods | les disques d'un cloud, Longhorn, Ceph |

C'est ce qui explique deux choses que vous entendrez souvent. « Kubernetes » ne désigne pas un produit unique : chaque distribution, et chaque offre de cloud, assemble Kubernetes avec ses propres pièces. Et pourtant, d'un cluster à l'autre, les commandes et les fichiers restent les mêmes : ce que vous apprenez sur un cluster d'apprentissage vaut sur un cluster de production.

> 💡 Dans le parcours, vous croiserez deux de ces pièces sans avoir à les installer : le runtime, que `kubectl get nodes -o wide` nomme dans sa dernière colonne, et le plugin réseau Calico, qui rendra possibles les règles réseau du layer « Isoler ».

## Deux entrées : la vôtre, et celle des utilisateurs

Le cluster a deux portes, qui ne servent pas aux mêmes personnes.

```scene
lesson: assets/k8s-overview.scene.json
step: two-doors
```

Par l'**API server**, vous dites ce qui doit tourner. Par les **nœuds**, les utilisateurs atteignent les applications : leurs requêtes ne passent jamais par le control plane.

La conséquence est rassurante : si le control plane s'arrête, plus rien ne se décide ni ne se répare, mais les pods déjà lancés continuent de répondre. C'est aussi pour cela que, dans un Kubernetes managé, celui que louent les fournisseurs de cloud (EKS chez AWS, GKE chez Google, AKS chez Azure), le fournisseur peut gérer le control plane à votre place : vous ne vous occupez que des nœuds et de vos applications.

> 📖 [Les composants de Kubernetes](https://kubernetes.io/docs/concepts/overview/components/) · [L'architecture d'un cluster](https://kubernetes.io/docs/concepts/architecture/) · [L'interface avec le runtime (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/) · [Les plugins réseau (CNI)](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) · [Les volumes et la CSI](https://kubernetes.io/docs/concepts/storage/volumes/#csi)

## Ce qu'il faut retenir

- Un cluster a deux parties : le control plane décide, les nœuds exécutent, dans des pods.
- Tout passe par l'API server, `kubectl` compris. Lui seul lit et écrit dans etcd.
- Sur chaque nœud, le kubelet veille sur les pods et délègue leur lancement au runtime.
- CRI, CNI et CSI sont trois prises standard : le runtime, le réseau et le stockage s'y branchent, et se choisissent.
- Vous entrez par l'API server, les utilisateurs par les nœuds.

La suite du parcours vous fait démarrer un cluster, puis retrouver ces pièces une par une dans le vôtre, avec [Ce qu'il y a dans un cluster Kubernetes](https://byte-layers.com/fr/bits/k8s-cluster-anatomy).

```quiz
prompt: "Une équipe remplace Calico par Cilium dans son cluster. Quelle interface rend ce changement possible sans toucher à Kubernetes lui-même ?"
answers:
  - "CRI, l'interface du runtime de conteneurs"
  - "CNI, l'interface du réseau"
  - "CSI, l'interface du stockage"
correct: 2
explanation: |
  Calico et Cilium sont deux plugins réseau : ils se branchent sur la CNI. La CRI concerne le runtime (containerd, CRI-O), et la CSI le stockage.
```

```quiz
prompt: "Le control plane d'un cluster tombe en panne pendant dix minutes. Que se passe-t-il pour les utilisateurs de l'application ?"
answers:
  - "L'application s'arrête : sans control plane, les pods s'éteignent"
  - "Les pods déjà lancés continuent de répondre, mais rien ne peut être déployé ni réparé"
  - "Rien ne change, et les pods en panne sont toujours remplacés"
correct: 2
explanation: |
  Les requêtes des utilisateurs arrivent sur les nœuds, pas sur le control plane : les pods en place continuent de répondre. En revanche, plus personne ne décide : un pod qui tombe pendant la panne n'est pas remplacé, et `kubectl` ne répond plus.
```

```ack
J'ai compris
```
