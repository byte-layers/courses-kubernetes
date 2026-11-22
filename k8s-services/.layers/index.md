---
title: "Donner une adresse stable avec un Service"
description: "Cours pratique Kubernetes : voir les adresses des pods changer, écrire un Service qui leur donne un nom et une adresse fixes, puis l'appeler depuis l'intérieur du cluster grâce au DNS."
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

Vos 3 réplicas de `vote` tournent, mais personne ne sait vraiment comment les joindre : les pods changent d'adresse à chaque remplacement. Vous allez voir ce problème de vos yeux, puis donner à `vote` une **adresse stable** avec un **Service**, et l'appeler par son nom depuis l'intérieur du cluster. À la fin, vous savez écrire un Service, lire à quels pods il transmet les requêtes, et expliquer comment les morceaux d'une application se trouvent sans connaître une seule adresse IP.

**Niveau** : débutant. Vous savez déployer une application avec un Deployment et changer son nombre de réplicas : ce sont les cours [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment) et [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling). Chaque champ d'un Service est expliqué quand il arrive.

**Ce qu'il faut** : un cluster qui répond à `kubectl get nodes`, celui du lab [Démarrer son cluster Kubernetes avec minikube](https://byte-layers.com/fr/labs/k8s-minikube-install), et un éditeur de texte. Aucun port de votre machine n'est utilisé.

**Vos fichiers** vivent dans `~/byte-layers/kubernetes/l3`. Rien n'est supprimé à la fin : le Deployment `vote`, son Service et vos fichiers restent en place, car les cours suivants s'en servent.

## Comment ça se passe

Le cours compte deux étapes, à suivre dans l'ordre, pour environ une heure en tout. La première montre le problème : vous supprimez un pod et vous comparez les adresses. La seconde le règle : vous écrivez le Service en remplaçant des lignes `# TODO`, puis vous l'essayez depuis un pod de test. Un bouton vérifie l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.
