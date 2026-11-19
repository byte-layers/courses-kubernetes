---
title: "Dire à Kubernetes si l'application va bien : les sondes"
description: "Cours pratique Kubernetes : voir une version qui plante retirer des pods sains, ajouter une sonde de readiness et une sonde de liveness, rejouer la panne sans dégât, puis livrer la version corrigée."
tags:
  - skill
  - kubernetes
  - containers
difficulty: beginner
duration: 60
introTitle: "Présentation"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Une nouvelle version de `vote` plante au démarrage, et pendant qu'elle plante, Kubernetes **retire des pods sains** de l'ancienne version. Vous allez voir pourquoi, puis donner à Kubernetes un moyen de vérifier lui-même la santé de l'application : des **sondes** (*probes*). Vous rejouerez ensuite la même panne, sans dégât cette fois, avant de livrer la version corrigée.

Les fichiers de départ vous sont fournis, et c'est vous qui les faites évoluer : les lignes marquées `# TODO` sont à compléter vous-même, à partir de ce que le texte vient d'expliquer. Le bouton de vérification contrôle l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.

**Niveau** : débutant. Vous savez décrire une application dans un Deployment et l'appliquer avec `kubectl apply`, ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), et vous savez ce que sont des réplicas, vus dans [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling). Le Service `vote` vient de [Donner une adresse stable avec un Service](https://byte-layers.com/fr/skills/k8s-services), l'application de vote de [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app), le namespace `vote-app` de [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas), et la ConfigMap `vote-config` de [Sortir les réglages de l'image avec une ConfigMap](https://byte-layers.com/fr/skills/k8s-configmap).

**Ce qu'il faut** : un cluster `layers-k8s` auquel `kubectl` répond, un éditeur de texte, un navigateur, et le port **28661** de votre machine libre.

**Où vivent les fichiers** : dans `~/byte-layers/kubernetes/l6/`. Rien n'est supprimé à la fin.

## Comment ça se passe

Cinq étapes à suivre dans l'ordre, environ une heure en tout. La première est un support : elle remet l'application de vote en version 2.0, sans sonde, dans le namespace `vote-app`, même si votre cluster est vide. Les quatre suivantes sont le cours : voir la panne retirer un pod sain, ajouter les deux sondes, rejouer la panne avec les sondes, puis corriger la v3 pour de bon.

**Ce qui reste dans le cluster à la fin** : l'application de vote complète dans `vote-app`, avec `vote` en version **3.0** sur 3 réplicas, ses deux sondes sur `/healthz`, et le réglage `VOTE_TITLE` dans la ConfigMap `vote-config`. Vos fichiers `vote.deploy.yml` et `vote.config.yml` disent la même chose que le cluster. Ne supprimez rien si vous continuez le parcours.
