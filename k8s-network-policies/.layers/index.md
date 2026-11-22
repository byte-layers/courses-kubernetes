---
title: "Fermer le réseau avec des NetworkPolicies"
description: "Cours pratique Kubernetes, avec des blancs à compléter : constater que le réseau est ouvert, écrire les règles réseau de l'application de vote, puis vérifier ce qui est fermé et ce qui reste ouvert."
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

Votre application de vote est rangée dans son namespace, mais n'importe quel pod du cluster peut encore joindre sa base de données. Vous allez **fermer le réseau** : tout interdire par défaut, puis n'ouvrir que les flux dont l'application a besoin, avec des **NetworkPolicies**. À la fin, un pod étranger ne joint plus ni `db` ni `redis`, et le vote fonctionne toujours.

Ce cours ne vous donne pas le fichier complet. Deux règles sont fournies, vous **complétez** la troisième et vous **écrivez** la quatrième, à partir de ce que le texte vient d'expliquer. Le bouton de vérification contrôle l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.

**Niveau** : débutant. Vous connaissez l'application de vote et ses cinq morceaux, ce que montre le lab [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app), vous savez qu'un Service retrouve ses pods par leurs étiquettes, ce qu'enseigne [Donner une adresse stable avec un Service](https://byte-layers.com/fr/skills/k8s-services), et vous savez écrire un fichier YAML et l'appliquer, ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment). Chaque champ d'une NetworkPolicy est expliqué quand il arrive.

**Ce qu'il faut** : un cluster `layers-k8s` auquel `kubectl` répond, démarré comme dans le lab [Démarrer son cluster Kubernetes avec minikube](https://byte-layers.com/fr/labs/k8s-minikube-install), un éditeur de texte, un navigateur, `curl`, et le port **28641** de votre machine libre.

**Où vivent les fichiers** : dans `~/byte-layers/kubernetes/l4/`. Les fichiers de l'application sont dans le sous-dossier `app/`, et vous écrivez les règles dans `vote-app.netpol.yml`. Rien n'est supprimé à la fin.

## Comment ça se passe

Trois étapes à suivre dans l'ordre, environ une heure en tout. La première commence par un support, qui remet l'application de vote dans le namespace `vote-app` même si votre cluster est vide, puis vous fait constater que le réseau est ouvert. La deuxième est le cœur du cours : vous décrivez une règle dans un éditeur visuel, puis vous complétez et écrivez celles de l'application. La troisième vérifie ce qui est fermé, ce qui reste ouvert, et situe les NetworkPolicies parmi les autres règles qu'on pose sur un namespace.

**Ce qui reste dans le cluster à la fin** : l'application de vote complète dans le namespace `vote-app`, avec son quota `pods-max` (12 pods) et ses quatre règles réseau. `vote-app` est le namespace dans lequel votre `kubectl` travaille. Les cours suivants, sur la configuration puis sur les mises à jour, partent de cet état : ne supprimez rien.
