---
title: "Ouvrir une porte d'entrée avec un Ingress"
description: "Cours pratique Kubernetes : comprendre le couple Ingress et contrôleur, écrire la règle qui envoie les visites vers un Service, entrer par cette porte et situer les façons d'exposer une application."
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

Votre application tourne dans le cluster, mais rien n'y entre depuis l'extérieur : un tunnel `kubectl port-forward` ne sert qu'à vous, et ne mène qu'à un seul pod. Vous allez ouvrir une **porte d'entrée** unique, un **Ingress**, qui reçoit les visites web et les répartit entre les réplicas de `vote`. À la fin, vous savez dire ce que fait un contrôleur d'Ingress, écrire une règle d'aiguillage, la voir fonctionner, et situer l'Ingress parmi les façons d'exposer une application.

**Niveau** : débutant. Vous savez déployer une application avec un Deployment et régler son nombre de réplicas : ce sont les cours [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment) et [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling). Le Service, sur lequel l'Ingress s'appuie, est rappelé en quelques lignes là où il sert.

**Ce qu'il faut** : un cluster qui répond à `kubectl get nodes`, celui du lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install), un éditeur de texte, un navigateur, `curl`, et le port **28632** de votre machine libre.

**Vos fichiers** vivent dans `~/byte-layers/kubernetes/l3`. À la fin, vous retirez l'Ingress : le Deployment `vote`, son Service, le contrôleur d'Ingress et vos fichiers restent en place.

## Comment ça se passe

Le cours compte deux étapes, à suivre dans l'ordre, pour environ 45 minutes en tout. La première installe le contrôleur si votre cluster n'en a pas, puis vous fait écrire la règle d'aiguillage en remplaçant des lignes `# TODO`. La seconde vous fait entrer par la porte, depuis le terminal puis depuis le navigateur, et la referme. Quelques gestes dépendent de l'endroit où tourne votre cluster, sur votre poste ou sur un poste hébergé (un VS Code dans le navigateur, fourni par un formateur) : le texte le dit à chaque fois.
