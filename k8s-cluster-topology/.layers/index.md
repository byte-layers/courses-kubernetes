---
title: "La topologie d'un cluster Kubernetes : un ou trois control planes"
description: "Combien de control planes pour un cluster Kubernetes, et ce qui se passe quand l'un tombe : un seul, puis trois derrière un répartiteur, et la majorité dont etcd a besoin. Pourquoi trois et pas deux, etcd sur les control planes ou à part, et ce que change un service géré."
tags:
  - bit
  - kubernetes
  - installation
difficulty: beginner
duration: 10
requires: "0.6.0"
---

Un cluster Kubernetes, ce sont des machines. Les **nœuds de travail** font tourner les applications, dans des pods. Le **control plane** les commande : il reçoit vos commandes, range l'état du cluster dans sa base, **etcd**, place les pods et recrée ceux qui disparaissent. Avant d'installer un cluster, une question se pose : combien de control planes, et que se passe-t-il quand l'un tombe ? C'est la **topologie** du cluster.

**Pour qui ?** Pour tout le monde, que vous ayez déjà installé un cluster ou non. Rien à installer, rien à lancer.

**Ce qui est supposé** : savoir ce que sont un pod et un nœud. [Ce qu'il y a dans un cluster Kubernetes](https://byte-layers.com/fr/bits/k8s-cluster-anatomy) présente toutes les pièces.

# La topologie d'un cluster Kubernetes

## Un seul control plane

C'est le cluster le plus simple : une machine pour le control plane, quelques-unes pour les applications. Que se passe-t-il si elle s'arrête ?

```scene
lesson: assets/k8s-cluster-topology.scene.json
step: single
```

Les applications **continuent de répondre** : chaque nœud garde ses pods et sait les relancer si un conteneur plante. Mais plus rien ne change. `kubectl` ne répond plus, on ne peut ni déployer ni corriger, et **personne ne remplace** les pods d'un nœud perdu. Pour un cluster d'apprentissage, de test, ou que l'on sait reconstruire vite, c'est suffisant. Pour la production, c'est un point unique de panne.

## Trois control planes, un répartiteur devant

Pour que le cluster survive à la perte d'un control plane, on en met trois, et un **répartiteur** devant leur API.

```scene
lesson: assets/k8s-cluster-topology.scene.json
step: ha
```

`kubectl` et les nœuds ne connaissent qu'**une adresse**, celle du répartiteur. Il envoie chaque demande à un control plane qui répond. Quand l'un tombe, les deux autres prennent tout, et les pods d'un nœud perdu sont recréés ailleurs. La scène va vite : par défaut, Kubernetes attend **environ cinq minutes** avant de les recréer, au cas où le nœud reviendrait.

> ⚠️ **Si le répartiteur est une seule machine**, la panne unique a seulement changé de place. On prend celui du cloud, ou deux machines qui se partagent une adresse (keepalived, kube-vip).

Avec `kubeadm`, l'outil d'installation officiel, cette adresse se donne **dès le premier control plane**, par `controlPlaneEndpoint`. L'ajouter après coup est bien plus délicat.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controlPlaneEndpoint: "api.mon-cluster.example:6443"
```

## Pourquoi trois, et pas deux ?

Tout repose sur etcd. Il n'enregistre une écriture que si **la majorité** de ses membres l'accepte.

```scene
lesson: assets/k8s-cluster-topology.scene.json
step: quorum
```

Sans majorité, etcd refuse d'écrire, et l'API ne peut plus rien enregistrer : les applications tournent, mais le cluster est **figé**. D'où les nombres impairs. Avec deux membres, la majorité est de deux : en perdre un suffit à tout figer. Deux control planes ne tolèrent donc **aucune panne**, comme un seul.

| Membres etcd | Majorité | Pannes tolérées |
|---|---|---|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |

Trois suffisent à la plupart des clusters. Cinq tolèrent deux pannes, au prix d'écritures un peu plus lentes : chaque écriture attend plus de réponses.

## etcd sur les control planes, ou à part ?

Jusqu'ici, chaque machine du control plane portait son etcd. C'est la topologie **empilée** (*stacked*), celle que `kubeadm` installe par défaut. On peut aussi sortir etcd sur ses propres machines : la topologie **etcd externe**.

```scene
lesson: assets/k8s-cluster-topology.scene.json
step: external
```

Les deux pannes sont maintenant **séparées** : un control plane perdu n'emporte aucun membre etcd, un nœud etcd perdu n'emporte aucun control plane. Le prix : **deux fois plus de machines** à installer, à mettre à jour et à surveiller.

| | etcd empilé | etcd externe |
|---|---|---|
| **Machines** | 3 | 6 : trois control planes, trois etcd |
| **Perdre une machine** | on perd un control plane et un membre etcd | on perd l'un ou l'autre |
| **Pour qui** | la plupart des clusters | les grands clusters, ou une équipe qui gère déjà etcd |

## Et avec un service géré ?

Chez un fournisseur de cloud (EKS, GKE, AKS…), le **control plane est à lui** : vous ne voyez ni ses machines ni etcd, seulement une adresse d'API. Selon l'offre, il est réparti sur plusieurs zones ou non : c'est écrit dans l'engagement de service du fournisseur, à lire avant de choisir. La question des nœuds de travail, elle, reste la vôtre : assez de nœuds pour que la perte d'un seul ne fasse pas tomber les applications.

> 📖 [Les deux topologies avec kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) · [Créer un cluster à plusieurs control planes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) · [La taille d'un cluster etcd](https://etcd.io/docs/v3.6/faq/#why-an-odd-number-of-cluster-members) · [Ce qui se passe quand un nœud tombe](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions)

## Ce qu'il faut retenir

- Le control plane commande, les nœuds exécutent : sans control plane, les applications tournent, mais rien ne se répare ni ne se déploie.
- Trois control planes derrière un répartiteur survivent à la perte de l'un d'eux.
- Le répartiteur ne doit pas être, à son tour, une machine unique.
- etcd n'écrit qu'avec une majorité : trois membres tolèrent une panne, deux n'en tolèrent aucune.
- etcd empilé sur les control planes par défaut, externe pour les grands clusters.
- Avec un service géré, le control plane est au fournisseur ; les nœuds restent à vous.

```quiz
prompt: "Un cluster a un seul control plane. Il tombe en panne pendant la nuit. Que constatent les utilisateurs de l'application le matin ?"
answers:
  - "Rien : l'application répond toujours, tant qu'aucun nœud ne tombe"
  - "Une coupure : sans control plane, les pods s'arrêtent"
  - "Une lenteur : les nœuds attendent le control plane à chaque requête"
correct: 1
explanation: |
  Les nœuds gardent leurs pods et le trafic leur arrive toujours : le control plane n'est pas sur le chemin des requêtes. Ce qui manque, c'est la réparation : si un nœud tombe, ses pods ne sont recréés nulle part, et `kubectl` ne répond plus.
```

```quiz
prompt: "Une équipe passe de un à deux control planes, chacun avec son etcd, pour mieux résister aux pannes. Qu'a-t-elle gagné ?"
answers:
  - "Elle tolère maintenant la perte d'un control plane"
  - "Rien : avec deux membres, etcd a besoin des deux pour écrire"
  - "Le double de capacité pour les applications"
correct: 2
explanation: |
  La majorité de deux membres, c'est deux. Si l'un tombe, etcd n'a plus de majorité et refuse d'écrire : le cluster est figé, comme avec un seul control plane. Il en faut trois pour tolérer une panne.
```

```ack
J'ai compris
```
