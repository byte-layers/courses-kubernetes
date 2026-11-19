---
title: "Ranger un mot de passe dans un Secret"
description: "Cours pratique Kubernetes : écrire un Secret pour les identifiants de la base de données, puis le brancher sur son Deployment à la place des valeurs écrites en clair."
tags:
  - skill
  - kubernetes
  - containers
difficulty: beginner
duration: 35
introTitle: "Présentation"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Le fichier de la base de données de l'application de vote contient son mot de passe **en clair**, une mauvaise pratique assumée jusqu'ici. Vous allez sortir ce mot de passe du Deployment pour le ranger dans un objet dédié, le **Secret**, puis dire à la base de le lire là.

Ce cours ne vous donne pas les fichiers complets. Vous les **construisez** en remplaçant des lignes `# TODO`, à partir de ce que le texte vient d'expliquer. Le bouton de vérification contrôle l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.

**Niveau** : débutant. Vous savez décrire une application dans un Deployment et l'appliquer avec `kubectl apply`, ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), et vous avez déjà rencontré l'application de vote et ses cinq morceaux dans [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app). Le namespace `vote-app` vient de [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas).

**Ce qu'il faut** : un cluster `layers-k8s` auquel `kubectl` répond, un éditeur de texte, `curl` ou un navigateur pour voter une fois, et le port **28651** de votre machine libre.

**Où vivent les fichiers** : dans `~/byte-layers/kubernetes/l5/`. Rien n'est supprimé à la fin : la suite du parcours s'en sert.

## Comment ça se passe

Trois étapes à suivre dans l'ordre, environ 35 minutes en tout. La première est un support : elle remet l'application de vote dans son état de départ, dans le namespace `vote-app`, même si votre cluster est vide. Les deux suivantes sont le cours : écrire le Secret, puis le brancher sur la base.

**Ce qui reste dans le cluster à la fin** : l'application de vote complète dans `vote-app`, dont la base lit ses identifiants dans le Secret `db-credentials`. Si vous arrivez de [Fermer le réseau avec des NetworkPolicies](https://byte-layers.com/fr/skills/k8s-network-policies), le quota `pods-max`, les quatre règles réseau et l'Ingress de `vote-app` sont toujours là : ce cours n'y touche pas. La suite du parcours part de cet état : ne supprimez rien.
