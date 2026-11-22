---
title: "Vers la prod : installer Kubernetes sur vos machines ou chez un fournisseur"
description: "Un cluster Kubernetes de production, installé sur vos propres machines ou loué chez un fournisseur (EKS, GKE, AKS, OVHcloud, Scaleway…) : qui s'occupe de chaque brique, ce que chaque choix vous laisse à faire, ce qu'il coûte, et pour qui. Avec un exemple sur site, Rancher et RKE2."
tags:
  - bit
  - kubernetes
  - installation
difficulty: beginner
duration: 10
requires: "0.8.0"
---

Vous avez tout pratiqué sur un cluster d'un seul nœud, minikube, sur votre poste. En production, le cluster a plusieurs **control planes** (la partie qui commande le cluster et range son état dans etcd) et plusieurs **nœuds** (les machines qui font tourner vos pods). Reste une question : où tourne-t-il ? Sur **vos propres machines**, ce qu'on appelle « sur site » (*on-premise*), ou **chez un fournisseur** qui vous loue un cluster prêt à l'emploi ? Le choix décide de qui s'occupe de chaque brique, et donc du travail qui vous reste.

**Pour qui ?** Pour qui prépare un cluster de production, ou doit en discuter. Rien à installer, rien à lancer.

**Ce qui est supposé** : savoir combien de control planes et de nœuds il faut, ce que présentent [La topologie d'un cluster Kubernetes](https://byte-layers.com/fr/bits/k8s-cluster-topology) et [Dimensionner les nœuds d'un cluster Kubernetes](https://byte-layers.com/fr/bits/k8s-cluster-sizing). Le choix d'une distribution est dans [Choisir une distribution Kubernetes](https://byte-layers.com/fr/bits/k8s-distribution-choice).

# Vers la prod : installer Kubernetes sur vos machines ou chez un fournisseur

## Installé soi-même : toutes les briques sont à vous

Prenons un cluster de production complet, monté sur vos machines. Qui s'en occupe ?

```scene
lesson: assets/k8s-managed-cluster.scene.json
step: self
```

**Vous**, pour tout ce qui est vert : les machines, les trois control planes et leur etcd, le répartiteur (*load balancer*) devant l'API, celui du trafic entrant, le stockage des volumes. La charge monte ? Il faut une machine de plus, l'installer, la faire rejoindre le cluster. Et chaque **montée de version** est à vous : Kubernetes sort environ trois versions par an, et chacune ne reçoit des correctifs que pendant à peu près un an.

## Chez un fournisseur : il gère le control plane

Le même cluster, loué chez un fournisseur : EKS chez AWS, GKE chez Google, AKS chez Microsoft, ou en Europe chez OVHcloud ou Scaleway.

```scene
lesson: assets/k8s-managed-cluster.scene.json
step: managed
```

Le **control plane** passe au fournisseur : il ne vous en reste qu'une adresse d'API. Il vous aide souvent pour les montées de version et répare les nœuds en panne, selon l'offre et vos réglages. Les nœuds deviennent des **machines louées**, dont vous choisissez la taille et le nombre. Vos applications, vos images et vos données restent à vous : les fournisseurs appellent ce partage le **modèle de responsabilité partagée**.

## Un exemple sur site : Rancher et RKE2

À quoi ressemble un cluster sur site, en vrai ? Voici celui de l'application de vote, sur sept machines.

```scene
lesson: assets/k8s-on-prem-cluster.scene.json
step: on-prem
```

**RKE2** installe Kubernetes sur les machines, **Rancher** (open source, édité par SUSE) le gère depuis une console. Trois control planes, trois nœuds Linux, et un **nœud Windows** pour le worker écrit en .NET Framework : le control plane, lui, est toujours sous Linux. Autour, chaque pièce est à trouver et à installer vous-même : la porte d'entrée, le stockage, les sauvegardes avec Velero, le registre Harbor.

> ⚠️ **Si vous déclarez un Service de type `LoadBalancer` sur site**, il reste sans adresse : c'est le cloud qui fournit habituellement ce répartiteur. Sur vos machines, il faut un outil qui en tienne lieu, comme MetalLB ou kube-vip, ou le répartiteur matériel de votre réseau.

## Ce que chaque choix vous laisse à faire

Le tableau reprend chaque brique d'un cluster de production, et dit qui s'en occupe.

| Brique | Installé soi-même | Chez un fournisseur |
|---|---|---|
| **Machines** | à acheter ou à louer, à installer, à remplacer | louées : vous choisissez leur taille et leur nombre |
| **Control planes et etcd** | à installer, sauvegarder, réparer | gérés, une adresse d'API ; répartis sur plusieurs zones ou non, selon l'offre |
| **Répartiteurs** (API, trafic entrant) | à installer et à doubler | un service loué |
| **Stockage des volumes** | à choisir et à exploiter (une baie, Ceph, Longhorn…) | un service loué, branché au cluster |
| **Montées de version** | toutes à vous | le fournisseur aide ; vous planifiez, nœuds compris |
| **Applications, images, données** | à vous | à vous |

Dans les deux cas, vos fichiers et vos commandes `kubectl` restent les mêmes : un Deployment écrit pour minikube tourne sur l'un comme sur l'autre.

## Ce que chaque choix coûte

Sur site, on paie les **machines** une fois, puis la salle, l'électricité et le réseau. Le coût qui pèse le plus est souvent ailleurs : une **équipe** capable d'installer, de mettre à jour et de réparer le cluster, astreintes comprises.

Chez un fournisseur, on paie **à l'usage**. Le control plane est gratuit dans certaines offres, ou facturé à l'heure : 0,10 $ de l'heure par cluster chez EKS et GKE, comme au niveau Standard d'AKS, dont le niveau gratuit n'a pas d'engagement de disponibilité. S'y ajoutent les machines louées, les volumes, les répartiteurs et le trafic qui sort du cluster. La facture grandit avec le cluster, et changer de fournisseur demande du travail : c'est la **dépendance** au fournisseur.

## Pour qui ?

Le fournisseur convient à une **équipe réduite**, qui veut ses applications en production sans tenir un control plane : c'est le chemin le plus court. Sur site convient à qui a déjà un **datacenter** et l'équipe qui l'exploite, ou de fortes contraintes sur ses données.

Ces contraintes ont un nom : la **souveraineté**. Une administration, un hôpital ou une banque peuvent devoir garder leurs données dans un pays, chez un hébergeur qu'une loi étrangère ne peut pas obliger à les livrer. En France, la qualification **SecNumCloud** de l'ANSSI atteste qu'une offre cloud répond à ces exigences ; seules certaines offres l'ont, à vérifier une à une.

> 💡 **Remarque.** Le choix n'est pas toujours tranché. Des fournisseurs gèrent aussi des clusters installés dans votre datacenter, et Rancher sait gérer, depuis une même console, des clusters sur site et des clusters loués.

> 📖 [Un cluster pour la production](https://kubernetes.io/docs/setup/production-environment/) · [La durée de support des versions](https://kubernetes.io/releases/) · [La responsabilité partagée chez GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/shared-responsibility) · [La sécurité d'EKS, partagée avec AWS](https://docs.aws.amazon.com/eks/latest/userguide/security.html) · [Les niveaux d'AKS](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers) · [RKE2](https://docs.rke2.io/) · [Rancher](https://ranchermanager.docs.rancher.com/) · [MetalLB](https://metallb.io/) · [Les nœuds Windows](https://kubernetes.io/docs/concepts/windows/intro/) · [SecNumCloud](https://cyber.gouv.fr/secnumcloud-pour-les-fournisseurs-de-services-cloud)

## Ce qu'il faut retenir

- Installé soi-même, toutes les briques sont à vous : machines, control planes, répartiteurs, stockage, montées de version.
- Chez un fournisseur, il gère le control plane, et souvent aide aux montées de version et à la réparation des nœuds ; vous louez les machines et choisissez leur taille et leur nombre.
- Dans les deux cas, vos applications, vos images et vos données restent à vous.
- Sur site, on paie des machines et surtout une équipe ; chez un fournisseur, on paie à l'usage, avec une dépendance à lui.

| | Sur vos machines | Chez un fournisseur |
|---|---|---|
| **Vous gérez** | tout | les nœuds (taille, nombre), vos applications et vos données |
| **Vous payez** | les machines, la salle, une équipe | à l'usage : control plane, machines, volumes, trafic |
| **Pour qui** | un datacenter existant, une équipe qui l'exploite, des données à garder chez soi | une équipe réduite, une mise en production rapide |
| **Ce qui peut bloquer** | les compétences, les astreintes | la facture qui grandit, la dépendance, la souveraineté des données |

```quiz
prompt: "Une équipe passe d'un cluster installé sur ses machines à un service géré. Qu'est-ce qui reste à sa charge ?"
answers:
  - "Rien : le fournisseur gère tout, applications comprises"
  - "Le choix des nœuds, ses applications et ses données"
  - "Le control plane et etcd, comme avant"
correct: 2
explanation: |
  Le fournisseur prend le control plane, etcd et souvent la réparation des nœuds. L'équipe choisit encore la taille et le nombre des machines louées, et ses applications, ses images et ses données restent à elle.
```

```quiz
prompt: "Sur un cluster installé sur ses machines, un Service de type LoadBalancer reste sans adresse. Pourquoi ?"
answers:
  - "Kubernetes ne sait pas créer ce répartiteur lui-même : c'est d'habitude le cloud qui le fournit"
  - "Les Services de type LoadBalancer sont interdits sur site"
  - "Il faut un control plane géré par un fournisseur"
correct: 1
explanation: |
  Kubernetes demande un répartiteur, mais ne le crée pas. Dans le cloud, le fournisseur en loue un ; sur site, un outil comme MetalLB ou kube-vip, ou un répartiteur matériel, doit en tenir lieu.
```

```ack
J'ai compris
```
