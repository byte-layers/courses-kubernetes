---
title: "Passer à l'échelle avec les réplicas"
description: "Cours pratique Kubernetes : faire passer une application de 1 à 3 exemplaires, les perdre tous d'un coup, voir un raccourci effacé par le fichier, puis éteindre l'application sans la supprimer."
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

Jusqu'ici, Kubernetes maintenait **un** exemplaire de `vote` en vie. Vous allez maintenant lui en demander plusieurs, les supprimer tous d'un coup, découvrir pourquoi « le fichier fait foi », puis éteindre l'application sans la supprimer. À la fin, vous savez changer le nombre d'exemplaires d'une application en modifiant son état souhaité, et vous savez pourquoi on modifie le **fichier** plutôt que de taper une commande de raccourci.

**Niveau** : débutant. Vous savez écrire et appliquer le fichier d'un Deployment, et vous connaissez les notions de pod, de Deployment et de réconciliation : c'est le cours [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment). Chaque nouvelle commande est expliquée quand elle arrive.

**Ce qu'il faut** : un cluster qui répond à `kubectl get nodes`, celui du lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install), un éditeur de texte, `curl`, et le port **28620** de votre machine libre.

**Vos fichiers** vivent dans `~/byte-layers/kubernetes/l2`. Rien n'est supprimé à la fin : le Deployment `vote` reste dans le cluster avec **3 réplicas**, et son fichier `vote.deploy.yml` reste dans ce dossier, car les cours suivants s'en servent. Le fichier final se trouve aussi dans [`solution/vote.deploy.yml`](../solution/vote.deploy.yml).

## Comment ça se passe

Deux étapes à suivre dans l'ordre, environ une heure en tout. La première fait passer `vote` de 1 à 3 exemplaires, regarde lequel répond, puis les supprime tous. La seconde essaie le raccourci `kubectl scale`, puis éteint et rallume l'application. Vous modifiez vous-même le fichier là où le texte montre une ligne `# TODO` : un bouton vérifie alors l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.
