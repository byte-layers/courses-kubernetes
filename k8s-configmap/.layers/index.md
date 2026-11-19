---
title: "Sortir les réglages de l'image avec une ConfigMap"
description: "Cours pratique Kubernetes : écrire une ConfigMap, la brancher sur un Deployment, puis changer un réglage et redémarrer les pods pour qu'ils le lisent."
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

Votre application de vote tourne, mais ses réglages sont gravés dans l'image : la question posée est toujours « Chats ou Chiens ? ». Vous allez changer cette question **sans reconstruire l'image**, en rangeant les réglages dans une **ConfigMap**, puis vous verrez pourquoi un réglage modifié n'est lu qu'après un redémarrage des pods.

Ce cours ne vous donne pas les fichiers complets. Vous les **construisez** en remplaçant des lignes `# TODO`, à partir de ce que le texte vient d'expliquer. Le bouton de vérification contrôle l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.

**Niveau** : débutant. Vous savez décrire une application dans un Deployment et l'appliquer avec `kubectl apply`, ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), et vous avez déjà rencontré l'application de vote et ses cinq morceaux dans [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app). Le namespace `vote-app` vient de [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas).

**Ce qu'il faut** : un cluster `layers-k8s` auquel `kubectl` répond, un éditeur de texte, un navigateur, et le port **28650** de votre machine libre.

**Où vivent les fichiers** : dans `~/byte-layers/kubernetes/l5/`. Rien n'est supprimé à la fin : le cours suivant s'en sert. La vérification y garde aussi un fichier caché, `.vote-options`, qui lui sert à savoir si vous avez changé les options.

## Comment ça se passe

Quatre étapes à suivre dans l'ordre, environ 45 minutes en tout. La première est un support : elle remet l'application de vote dans son état de départ, dans le namespace `vote-app`, même si votre cluster est vide. Les trois suivantes sont le cours : écrire la ConfigMap, la brancher sur `vote`, puis changer un réglage et redémarrer.

**Ce qui reste dans le cluster à la fin** : l'application de vote complète dans `vote-app`, avec la ConfigMap `vote-config` que les trois pods de `vote` lisent. Si vous arrivez de [Fermer le réseau avec des NetworkPolicies](https://byte-layers.com/fr/skills/k8s-network-policies), le quota `pods-max`, les quatre règles réseau et l'Ingress de `vote-app` sont toujours là : ce cours n'y touche pas. La suite du parcours part de cet état : ne supprimez rien.
