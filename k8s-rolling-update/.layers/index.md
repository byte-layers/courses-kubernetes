---
title: "Mettre à jour sans coupure et revenir en arrière"
description: "Cours pratique Kubernetes : déployer une nouvelle version sans interruption, lire l'historique, diagnostiquer une version qui plante avec describe et logs, puis revenir en arrière avec kubectl rollout undo."
tags:
  - skill
  - kubernetes
  - containers
difficulty: beginner
duration: 75
introTitle: "Présentation"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

L'équipe de développement livre deux nouvelles versions de `vote` : une **v2** avec un nouveau design, puis une **v3**… qui exige un réglage que personne n'a pensé à livrer. Vous allez déployer la v2 **sans coupure**, voir la v3 planter, **diagnostiquer** la panne comme le ferait une équipe d'exploitation, puis **revenir en arrière** en une commande et remettre votre fichier en cohérence avec le cluster.

Le fichier de départ vous est fourni, et c'est vous qui le faites évoluer : la ligne marquée `# TODO` est à modifier vous-même, à partir de ce que le texte vient d'expliquer. Le bouton de vérification contrôle l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.

**Niveau** : débutant. Vous savez décrire une application dans un Deployment et l'appliquer avec `kubectl apply`, ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), et vous savez ce que sont des réplicas, vus dans [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling). Le Service `vote` et le pod de test jetable viennent de [Donner une adresse stable avec un Service](https://byte-layers.com/fr/skills/k8s-services), l'application de vote et ses cinq morceaux de [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app), le namespace `vote-app` de [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas), et la ConfigMap `vote-config` de [Sortir les réglages de l'image avec une ConfigMap](https://byte-layers.com/fr/skills/k8s-configmap).

**Ce qu'il faut** : un cluster `layers-k8s` auquel `kubectl` répond, un éditeur de texte, un navigateur, et le port **28660** de votre machine libre.

**Où vivent les fichiers** : dans `~/byte-layers/kubernetes/l6/`. Rien n'est supprimé à la fin : le cours suivant s'en sert.

## Comment ça se passe

Quatre étapes à suivre dans l'ordre, environ 1 h 15 en tout. La première est un support : elle remet l'application de vote en version 1.0 dans le namespace `vote-app`, même si votre cluster est vide. Les trois suivantes sont le cours : déployer la v2 sans coupure et lire l'historique, voir la v3 planter et trouver pourquoi, puis revenir en arrière.

**Ce qui reste dans le cluster à la fin** : l'application de vote complète dans `vote-app`, avec `vote` en version **2.0** sur 3 réplicas, et la v3 cassée gardée dans l'historique du Deployment. Votre fichier `vote.deploy.yml` dit lui aussi `2.0`. Les contenus suivants en ont besoin : ne supprimez rien.
