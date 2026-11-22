---
title: "Choisir une distribution Kubernetes"
description: "Quel Kubernetes faire tourner, et qui s'en occupe : la distribution officielle, installée par vous ou clé en main chez un fournisseur de cloud (KaaS), ou une distribution spécifique, outillée pour l'entreprise (OpenShift, Rancher, Talos) ou légère (minikube, kind, k3s). Ce que chaque choix vous laisse à faire."
tags:
  - bit
  - kubernetes
  - installation
difficulty: beginner
duration: 8
requires: "0.8.0"
---

Kubernetes fait tourner vos conteneurs sur un groupe de machines, le **cluster** : un **control plane** qui décide, des **nœuds** qui exécutent. Avant d'en démarrer un, il faut choisir **quel Kubernetes** faire tourner et **qui s'en occupe**. Les réponses se rangent en deux familles : la distribution officielle, publiée par le projet Kubernetes, et les distributions spécifiques, qui la reprennent avec leurs propres choix.

**Pour qui ?** Pour qui n'a encore jamais choisi ni installé de cluster. Rien à installer, rien à lancer.

**Ce qui est supposé** : savoir ce que sont le control plane et les nœuds. [Kubernetes en un coup d'œil](https://byte-layers.com/fr/bits/k8s-overview) les présente.

# Choisir une distribution Kubernetes

## Deux familles : la distribution officielle et les distributions spécifiques

Tout part du même projet open source. Qui en tire quoi ?

```scene
lesson: assets/k8s-distribution-choice.scene.json
step: map
```

Le projet Kubernetes publie chaque version : c'est la **distribution officielle**. Vous l'installez sur vos machines, ou un fournisseur de cloud la fait tourner pour vous. Des éditeurs en tirent des **distributions spécifiques** : le même Kubernetes, complété d'outils pour l'entreprise ou allégé jusqu'à tenir sur un poste. Quatre cases, donc, et une question pour chacune : **qui s'occupe de quoi ?** Les sections suivantes les reprennent une à une, sur la même carte.

## La distribution officielle, installée par vous

Vous avez des machines, dans votre salle ou louées chez un hébergeur. Que reste-t-il à faire ?

```scene
lesson: assets/k8s-distribution-choice.scene.json
step: self
```

**Tout.** `kubeadm`, l'outil du projet, monte le cluster sur des machines qui existent déjà ; `kubespray` fait le même travail avec Ansible ; `kops` crée en plus les machines dans le cloud. Ensuite, sauvegarder etcd, **mettre à jour** chaque pièce et remplacer une machine restent votre travail. En échange, rien ne vous lie à un fournisseur : c'est le choix du **contrôle complet**.

## La distribution officielle, clé en main chez un fournisseur

Les grands clouds la proposent en service géré, ou **KaaS** (*Kubernetes as a Service*) : EKS chez AWS, GKE chez Google, AKS chez Microsoft, Scaleway Kapsule ou OVHcloud Managed Kubernetes en Europe.

```scene
lesson: assets/k8s-distribution-choice.scene.json
step: managed
```

Le **control plane** passe au fournisseur : ses machines, etcd et ses sauvegardes. Vos **nœuds**, le plus souvent, restent à vous : vous choisissez leur nombre et leur taille, le fournisseur crée les machines. C'est la voie la plus courte vers un cluster qui sert, au prix d'une **dépendance** au fournisseur et de sa facture.

> ⚠️ **Si vous pensez ne plus avoir de mises à jour à faire** : chaque version de Kubernetes a une durée de support, chez le fournisseur aussi. Monter de version reste à planifier, nœuds compris.

## Les distributions outillées pour l'entreprise

Une entreprise veut souvent plus qu'un cluster nu : une console, un registre d'images, quelqu'un à appeler.

```scene
lesson: assets/k8s-distribution-choice.scene.json
step: platform
```

**OpenShift**, de Red Hat, et sa version communautaire **OKD** livrent Kubernetes avec des **outils autour**, testés ensemble ; OpenShift, payant, y ajoute le support de Red Hat. D'autres visent l'entreprise autrement : **Rancher**, de SUSE, pilote plusieurs clusters depuis une console ; **Talos Linux** remplace le système des machines par un système fait pour Kubernetes, sans SSH. Vos fichiers et `kubectl` restent ceux de Kubernetes.

## Les distributions légères : un cluster sur votre poste

Pour découvrir Kubernetes, inutile de louer quatre machines : on **émule** un cluster sur la sienne.

```scene
lesson: assets/k8s-distribution-choice.scene.json
step: light
```

`minikube` démarre un cluster d'un seul nœud par défaut, dans un conteneur Docker ou une machine virtuelle ; `kind` et `k3d` font tourner leurs nœuds dans des conteneurs. **Ils ne visent pas la production** : une machine, et sa panne arrête tout. D'autres distributions légères, comme **k3s** ou **MicroK8s**, servent aussi en production, sur de petites machines.

## Les quatre choix côte à côte

Le tableau reprend les quatre cases de la carte, avec qui s'occupe du control plane et des nœuds.

| Choix | Exemples | Le control plane | Les nœuds | Pour |
|---|---|---|---|---|
| Officielle, installée par vous | kubeadm, kubespray, kops | à vous | à vous | garder la main, sur vos machines |
| Officielle, clé en main (KaaS) | EKS, GKE, AKS, Kapsule | au fournisseur | à vous, le plus souvent | aller vite, dans le cloud |
| Spécifique, outillée | OpenShift, OKD, Rancher, Talos | à vous, ou au fournisseur selon l'offre | à vous, le plus souvent | une plateforme d'entreprise, du support |
| Spécifique, légère | minikube, kind, k3d, k3s, MicroK8s | à vous, souvent sur le même nœud que vos pods | une machine, ou quelques petites | apprendre, tester, petites machines |

Le nombre de distributions est grand : la [liste des distributions certifiées par la CNCF](https://www.cncf.io/training/certification/software-conformance/) en donne une idée. Une distribution certifiée accepte les mêmes fichiers et les mêmes commandes `kubectl`.

> 📖 [Les outils d'installation officiels](https://kubernetes.io/docs/setup/production-environment/tools/) · [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) · [kops](https://kops.sigs.k8s.io/) · [Kubespray](https://kubespray.io/) · [minikube](https://minikube.sigs.k8s.io/docs/) · [kind](https://kind.sigs.k8s.io/) · [k3s](https://docs.k3s.io/) · [OKD](https://okd.io/) · [Talos Linux](https://www.talos.dev/)

## Trois questions pour vérifier ce que vous avez retenu

```quiz
title: "Choisir une distribution Kubernetes"
tracksProgress: false
showNotice: false
place: stage
questions:
  - prompt: "Une équipe passe d'un cluster installé avec kubeadm à un service géré. Quelle tâche disparaît de sa liste ?"
    answers:
      - "Sauvegarder etcd"
      - "Choisir la taille des nœuds"
      - "Planifier les montées de version de Kubernetes"
    correct: 1
    explanation: "Avec un service géré, etcd fait partie du control plane, qui revient au fournisseur : ses sauvegardes aussi. La taille des nœuds reste un choix de l'équipe, et les montées de version restent à planifier, car chaque version a une durée de support."
  - prompt: "Une entreprise veut Kubernetes avec une console web, un registre d'images et un support à appeler. Quelle famille regarder ?"
    answers:
      - "Une distribution légère, comme minikube"
      - "Une distribution outillée pour l'entreprise, comme OpenShift"
      - "La distribution officielle installée avec kubeadm, qui fournit ces outils"
    correct: 2
    explanation: "OpenShift livre Kubernetes avec des outils testés ensemble, dont une console et un registre, et Red Hat vend le support. kubeadm installe un cluster nu : les outils restent à choisir et à installer. minikube sert à apprendre sur un poste."
  - prompt: "Vous voulez découvrir Kubernetes sur votre ordinateur, sans machine à louer. Que choisir ?"
    answers:
      - "minikube, qui émule un cluster d'un nœud sur votre poste"
      - "kops, qui crée les machines chez un fournisseur de cloud"
      - "Un service géré comme GKE"
    correct: 1
    explanation: "minikube démarre un cluster d'un seul nœud, dans un conteneur Docker ou une machine virtuelle, sur votre poste. kops et un service géré montent de vrais clusters sur des machines louées, faits pour durer, et facturés."
```

## Récap

- Deux familles : la **distribution officielle** de Kubernetes, et des **distributions spécifiques** qui la reprennent avec leurs choix.
- La distribution officielle s'installe par vous (kubeadm, kubespray, kops : tout reste à vous) ou **clé en main** chez un fournisseur, qui prend le control plane.
- Les distributions outillées (OpenShift, OKD, Rancher, Talos) ajoutent des outils, une gestion ou un système pensés pour l'entreprise.
- Les distributions légères font tenir un cluster sur un poste ou une petite machine : minikube, kind et k3d pour apprendre, k3s et MicroK8s aussi en production.
- Quel que soit le choix, une distribution certifiée accepte les mêmes fichiers et les mêmes commandes `kubectl`.
- À vous de jouer : [Démarrer son cluster Kubernetes avec minikube](https://byte-layers.com/fr/labs/k8s-minikube-install), le lab qui lance un cluster sur votre poste.

```ack
J'ai compris
```
