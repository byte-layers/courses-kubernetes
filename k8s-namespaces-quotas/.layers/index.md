---
title: "Ranger et encadrer avec un namespace et un quota"
description: "Cours pratique Kubernetes : sortir l'application de vote du namespace default, la ranger dans son propre namespace, puis plafonner ce namespace avec un quota et voir Kubernetes refuser des pods."
tags:
  - skill
  - kubernetes
  - containers
difficulty: beginner
duration: 45
introTitle: "Présentation"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Votre application de vote tourne, et ses cinq morceaux se trouvent grâce aux Services. Mais tout vit dans `default`, le namespace fourre-tout du cluster, et rien ne limite ce qu'une équipe y consomme. Vous allez **ranger** l'application dans son propre namespace, `vote-app`, puis **plafonner** ce namespace avec un quota, jusqu'à voir Kubernetes refuser des pods.

**Niveau** : débutant. Vous avez déjà déployé l'application de vote et ses cinq morceaux, ce que fait le lab [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app), vous lui avez ouvert une porte d'entrée avec [Ouvrir une porte d'entrée avec un Ingress](https://byte-layers.com/fr/skills/k8s-ingress), et vous savez changer le nombre de réplicas, ce qu'enseigne [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling). Les namespaces et les quotas sont expliqués ici, à partir de zéro.

**Ce qu'il faut** : un cluster `layers-k8s` auquel `kubectl` répond, celui du lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install), un navigateur, et le port **28640** de votre machine libre.

**Où vivent les fichiers** : dans `~/byte-layers/kubernetes/l4/app/`. Rien n'est supprimé à la fin : le cours suivant s'en sert.

## Comment ça se passe

Trois étapes à suivre dans l'ordre, environ 45 minutes en tout : sortir l'application de `default`, la ranger dans `vote-app`, puis poser un quota. La première commence par un support, qui remet l'application de vote dans `default` même si votre cluster est vide. Chaque étape fait taper les commandes, demande une prévision avant les plus parlantes, puis vérifie l'état réel du cluster.

**Ce qui reste dans le cluster à la fin** : l'application de vote complète (7 pods) dans le namespace `vote-app`, plafonné à 12 pods par le quota `pods-max`, et `vote-app` est devenu le namespace dans lequel votre `kubectl` travaille. Le namespace `default` ne contient plus l'application. Les contenus suivants partent de cet état : ne supprimez rien.
