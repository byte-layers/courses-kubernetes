---
title: "Réserver des ressources et comprendre un pod Pending"
description: "Cours pratique Kubernetes : voir sur quel nœud tournent les pods, réserver du processeur et de la mémoire avec les requests, provoquer un pod Pending et lire sa raison, puis corriger la réservation."
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

Un cluster n'a pas des ressources infinies. Vous allez voir où Kubernetes place les pods de `vote`, leur faire **réserver** du processeur et de la mémoire, puis demander volontairement trop : un pod reste alors bloqué en `Pending`, et vous lisez pourquoi. À la fin, vous savez écrire une réservation (*requests*), reconnaître un pod qui n'a pas trouvé de place, et expliquer pourquoi l'application est restée disponible pendant ce temps.

**Niveau** : débutant. Vous savez écrire et appliquer le fichier d'un Deployment, et vous connaissez les notions de pod, de ReplicaSet et de réconciliation : c'est le cours [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment). Le fichier de départ demande 3 exemplaires de l'application, ses **réplicas**, par sa ligne `replicas: 3` ; vous n'avez rien d'autre à en savoir ici.

**Ce qu'il faut** : un cluster qui répond à `kubectl get nodes`, celui du lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install), et un éditeur de texte. Aucun port de votre machine n'est utilisé.

**Vos fichiers** vivent dans `~/byte-layers/kubernetes/l2`. Rien n'est supprimé à la fin : le Deployment `vote` reste dans le cluster avec **3 réplicas, chacun réservant un dixième de processeur (`100m`) et 64 Mi de mémoire**, et son fichier `vote.deploy.yml` reste dans ce dossier. C'est de cet état que partent les cours suivants ; le fichier final se trouve aussi dans [`solution/vote.deploy.yml`](../solution/vote.deploy.yml).

## Comment ça se passe

Deux étapes à suivre dans l'ordre, environ 45 minutes en tout. La première ajoute une réservation démesurée et regarde le pod qui ne trouve pas de place. La seconde corrige la réservation, puis montre ce que chaque changement a laissé derrière lui. Vous modifiez vous-même le fichier là où le texte montre une ligne `# TODO` : un bouton vérifie alors l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.
