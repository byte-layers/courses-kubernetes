---
title: "Vers la prod : dimensionner les nœuds d'un cluster Kubernetes, la place réservable et la marge"
description: "Combien de pods tiennent sur un nœud Kubernetes : la part que prennent le système, le kubelet et les agents, l'ordonnanceur qui compte les réservations et non la consommation, et la place à garder pour survivre à la perte d'un nœud. Peu de gros nœuds ou beaucoup de petits."
tags:
  - bit
  - kubernetes
  - installation
difficulty: beginner
duration: 10
requires: "0.6.0"
---

Vous avez tout pratiqué sur un cluster d'un seul nœud, minikube, sans compter la place. En production, il faut décider combien de machines acheter ou louer, et de quelle taille. Un cluster se dimensionne nœud par nœud : combien de mémoire et de processeur sur chaque machine, et combien de machines. La fiche d'une machine donne sa capacité ; vos pods n'en verront qu'une partie. Et c'est la **réservation** de chaque pod, pas ce qu'il consomme, qui décide s'il trouve sa place.

**Pour qui ?** Pour qui prépare un cluster ou choisit la taille de ses nœuds. Rien à installer, rien à lancer.

**Ce qui est supposé** : savoir ce que sont un pod et un nœud, et avoir vu [la topologie d'un cluster](https://byte-layers.com/fr/bits/k8s-cluster-topology). Réserver des ressources se pratique dans [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending).

# Vers la prod : dimensionner les nœuds d'un cluster

## Ce qu'un nœud offre vraiment aux pods

Un nœud de 16 Go. Combien de pods de 2 Go peut-il porter ?

```scene
lesson: assets/k8s-cluster-sizing.scene.json
step: allocatable
```

Pas huit. Le système, le runtime et le **kubelet** ont leur réserve, et les agents des **DaemonSets** (journaux, supervision, réseau) tournent sur chaque nœud. Ce qui reste s'appelle l'**allouable** (*allocatable*) : ici 12 Go, six pods. Le septième attend, à l'état `Pending`. Les deux chiffres se lisent dans `kubectl describe node`, aux lignes `Capacity` et `Allocatable`.

> 💡 **Remarque.** Un service géré règle la réserve du système pour vous, et elle pèse plus sur un petit nœud : de l'ordre de 2 à 3 Go sur 16 Go chez GKE.

## Réserver juste

Les six pods réservent 2 Go chacun. Mais en consomment-ils autant ?

```scene
lesson: assets/k8s-cluster-sizing.scene.json
step: requests
```

L'ordonnanceur place un pod d'après sa **réservation** (`requests`), jamais d'après ce qu'il consomme. Un nœud peut donc refuser un pod alors qu'il est à moitié vide. Réserver ce que le pod utilise vraiment, plus une marge, fait tenir plus de pods sur les mêmes machines. Pour le savoir, on mesure : `kubectl top pod`, ou la supervision du cluster.

## Garder la place d'un nœud

Un nœud finira par tomber. Ses pods devront trouver de la place ailleurs.

```scene
lesson: assets/k8s-cluster-sizing.scene.json
step: node-loss
```

Avec trois nœuds, chacun garde **un tiers** de sa place libre : c'est le prix pour survivre à la perte de l'un d'eux. Sans cette marge, les pods du nœud perdu restent en `Pending`. Plus les nœuds sont nombreux, plus la marge par nœud est petite : un sixième avec six nœuds.

## Peu de gros nœuds, ou beaucoup de petits ?

La même capacité totale se découpe de plusieurs façons.

| | Peu de gros nœuds | Beaucoup de petits nœuds |
|---|---|---|
| **Perdre un nœud** | une grosse part des pods à replacer | une petite part |
| **Marge à garder** | grande (un tiers avec trois nœuds) | petite (un sixième avec six) |
| **Réserve du système et agents** | pèse peu sur chaque nœud | se répète sur chaque nœud |
| **Gros pods** (une base, un traitement) | ils tiennent | ils risquent de ne tenir nulle part |
| **Machines à gérer** | peu | beaucoup |

Un nœud porte aussi un nombre de pods limité : Kubernetes en recommande **110 au plus**, et chaque fournisseur fixe sa propre limite. On part en général de trois nœuds au moins, assez grands pour le plus gros pod, puis on ajoute des nœuds plutôt que de les grossir.

> 📖 [Réserver des ressources aux démons du système](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/) · [Requests et limits des conteneurs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [Les limites d'un grand cluster](https://kubernetes.io/docs/setup/best-practices/cluster-large/) · [La taille des nœuds chez GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes)

## Ce qu'il faut retenir

- La capacité d'un nœud n'est pas pour vos pods : le système, le kubelet et les agents en prennent une part. Le reste est l'allouable.
- L'ordonnanceur place les pods d'après leurs réservations, pas leur consommation : réserver juste fait tenir plus de pods.
- Gardez assez de place libre pour replacer les pods d'un nœud perdu : un tiers par nœud avec trois nœuds.
- Beaucoup de petits nœuds réduisent la marge, peu de gros nœuds réduisent la réserve répétée : assez grands pour le plus gros pod, assez nombreux pour la panne.

```quiz
prompt: "Un nœud de 16 Go n'accepte plus de pods, alors que `kubectl top node` indique 6 Go utilisés. Que se passe-t-il ?"
answers:
  - "Les pods déjà placés réservent toute la place allouable, même s'ils n'en consomment qu'une partie"
  - "Le nœud est en panne et refuse tous les pods"
  - "Kubernetes garde toujours la moitié d'un nœud libre"
correct: 1
explanation: |
  L'ordonnanceur additionne les réservations (`requests`) des pods déjà placés et les compare à l'allouable du nœud. Ce que les pods consomment vraiment n'entre pas dans le calcul : des réservations trop larges laissent un nœud à moitié vide et des pods en `Pending`.
```

```quiz
prompt: "Un cluster a trois nœuds remplis chacun à 90 % de leur place réservable. Que se passe-t-il si l'un d'eux tombe ?"
answers:
  - "Ses pods sont recréés sur les deux autres sans problème"
  - "Une partie de ses pods ne trouve pas de place et reste en Pending"
  - "Les deux autres nœuds s'agrandissent automatiquement"
correct: 2
explanation: |
  Il reste 10 % de place sur chacun des deux autres nœuds, soit de quoi accueillir à peu près un cinquième des pods du nœud perdu. Les autres restent en `Pending` tant qu'on n'ajoute pas de nœud. Avec trois nœuds, il faut garder environ un tiers de place libre sur chacun.
```

```ack
J'ai compris
```
