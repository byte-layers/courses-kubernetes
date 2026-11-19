---
title: "Lire une panne : CrashLoopBackOff, ImagePullBackOff, OOMKilled"
description: "Les statuts d'un pod en panne, ce que chacun veut dire, et où chercher la cause : deux pannes à provoquer en une minute, puis le catalogue à garder sous la main."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 15
requires: "0.4.0"
---

Un pod qui ne tourne pas affiche un statut : `CrashLoopBackOff`, `ImagePullBackOff`, `Pending`… Chacun de ces mots désigne une sorte de panne précise, et dit où chercher sa cause. Vous allez provoquer deux pannes très courantes, lire ce que Kubernetes en montre, puis repartir avec un catalogue des statuts.

**Prérequis** : avoir déjà diagnostiqué un pod qui plante avec `kubectl describe` et `kubectl logs`, ce qu'enseigne [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update). La dernière ligne du catalogue parle des sondes, qu'enseigne [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes). Cette page se lit après ces deux cours. Il vous faut un cluster `layers-k8s` auquel `kubectl` répond. Les deux Deployments créés ici portent le préfixe `l6-`, vivent dans le namespace courant de `kubectl`, et sont supprimés à la fin : il ne reste rien dans le cluster, et aucun fichier n'est créé.

# Lire une panne : CrashLoopBackOff, ImagePullBackOff, OOMKilled

Face à un pod en échec, la démarche ne change pas : la colonne `STATUS` de `kubectl get pods` dit **quelle sorte** de panne c'est, puis `kubectl describe pod` et `kubectl logs` en donnent la **cause**. Ce qui change d'une panne à l'autre, c'est l'endroit où la cause est écrite.

## Une image introuvable : ErrImagePull, puis ImagePullBackOff

Première panne : demander une image qui n'existe pas. `kubectl create deployment` crée un Deployment d'un seul pod sans écrire de fichier, ce qui convient à un essai jetable. L'image `kube-vote` existe en version 1.0, 2.0 et 3.0, mais pas en 4.0 :

```exec
kubectl create deployment l6-typo --image=ghcr.io/bngams/kube-vote:4.0
```

Laissez à ce Deployment 20 secondes pour démarrer. Faites une prévision : que va répondre la commande ?

```exec
kubectl rollout status deployment/l6-typo --timeout=20s
```

Le pod n'arrive jamais à l'état prêt, et la commande abandonne au bout du délai :

```text
Waiting for deployment "l6-typo" rollout to finish: 0 of 1 updated replicas are available...
error: timed out waiting for the condition
```

Regardez son pod. `kubectl create deployment` lui a donné l'étiquette `app=l6-typo`, qui sert ici à le désigner :

```exec
kubectl get pods -l app=l6-typo
```

Le statut passe d'abord par `ErrImagePull`, puis `ImagePullBackOff` : Kubernetes réessaie de télécharger l'image, en espaçant ses tentatives. La colonne `RESTARTS` reste à 0, puisque le conteneur n'a jamais démarré.

```text
NAME                       READY   STATUS             RESTARTS   AGE
l6-typo-6567b5f6d9-txhxr   0/1     ImagePullBackOff   0          20s
```

Ici, il n'y a pas de logs à lire : aucune application n'a tourné. La cause est dans les événements du pod, tout en bas de sa description :

```exec
kubectl describe pod -l app=l6-typo
```

Parmi les dernières lignes, celle-ci donne la raison :

```text
  Warning  Failed     5s (x2 over 18s)  kubelet            Failed to pull image "ghcr.io/bngams/kube-vote:4.0": Error response from daemon: manifest unknown
```

`manifest unknown` : le registre ne connaît pas cette version de l'image. C'est une faute de frappe dans la version, ou une image pas encore publiée. Le même statut apparaît quand le registre demande des droits d'accès que le cluster n'a pas.

## Trop peu de mémoire : OOMKilled

Seconde panne : une application qui dépasse la mémoire qu'on lui accorde. Créez un second Deployment, avec une image qui fonctionne. `--replicas=0` le crée sans aucun pod pour l'instant, le temps de le régler :

```exec
kubectl create deployment l6-oom --image=ghcr.io/bngams/kube-vote:2.0 --replicas=0
```

Vous connaissez les réservations (`requests`), la part de mémoire réservée pour un conteneur, vues dans [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending). Une **limite** (`limits`) est son pendant : un plafond que le conteneur n'a pas le droit de dépasser. `kubectl set resources` en pose une sur le Deployment, sans passer par un fichier. Donnez-lui 20 Mi de mémoire, bien trop peu pour cette application :

```exec
kubectl set resources deployment l6-oom --limits=memory=20Mi
```

Demandez maintenant un pod, avec `kubectl scale`, le raccourci vu dans [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling) :

```exec
kubectl scale deployment l6-oom --replicas=1
```

Regardez ce pod tout de suite :

```exec
kubectl get pods -l app=l6-oom
```

Pendant une seconde ou deux, tout a l'air d'aller : le conteneur démarre, et le pod peut même apparaître `Running`. C'est en chargeant l'application qu'il dépasse son plafond. Attendez que Kubernetes ait noté ce premier arrêt. `kubectl wait` rend la main dès que le champ indiqué par `--for` a la valeur attendue, ici la raison du dernier arrêt du conteneur, ou échoue au bout de 50 secondes :

```exec
kubectl wait pod -l app=l6-oom --for=jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'=OOMKilled --timeout=50s
```

La commande répond `condition met`. Regardez de nouveau le pod :

```exec
kubectl get pods -l app=l6-oom
```

Selon l'instant, la colonne `STATUS` affiche `OOMKilled`, `CrashLoopBackOff` ou brièvement `Running`, et la colonne `RESTARTS` augmente :

```text
NAME                     READY   STATUS      RESTARTS     AGE
l6-oom-bd8965dd5-j6th6   0/1     OOMKilled   1 (2s ago)   3s
```

`CrashLoopBackOff` dit seulement que le conteneur s'arrête en boucle. Pour savoir pourquoi, demandez la description du pod :

```exec
kubectl describe pod -l app=l6-oom
```

Cette fois, la cause est dans l'état du conteneur, vers le milieu de la réponse :

```text
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
    Ready:          False
    Restart Count:  1
    Limits:
      memory:  20Mi
```

*OOM* signifie *Out Of Memory* : l'application a dépassé son plafond de mémoire, et le système l'a arrêtée net, sans lui laisser le temps d'écrire quoi que ce soit dans ses logs. Le code de sortie `137` est celui d'un programme arrêté de force. Il faut alors relever la limite, ou chercher pourquoi l'application consomme autant.

## Supprimer les deux Deployments

Les deux essais ont servi. Supprimez-les, ce qui supprime aussi leurs pods :

```exec
kubectl delete deployment l6-typo l6-oom
```

## Le catalogue des pannes

Voici les statuts qu'on rencontre le plus souvent, avec ce que chacun veut dire et l'endroit où chercher la cause. Les deux pannes que vous venez de provoquer s'y trouvent, à côté de celles des cours précédents.

| Statut | Ce qui se passe | Où chercher |
|---|---|---|
| `Pending` | le pod attend une place sur un nœud | `kubectl describe pod` (événements) |
| `ContainerCreating` | l'image se télécharge, le conteneur démarre | patienter |
| `CreateContainerConfigError` | une ConfigMap ou un Secret attendu est introuvable | `kubectl describe pod` (événements) |
| `ErrImagePull` / `ImagePullBackOff` | l'image ne peut pas être téléchargée (nom, version, droits d'accès) | `kubectl describe pod` (événements) |
| `CrashLoopBackOff` | le conteneur s'arrête en boucle, le plus souvent parce que l'application plante au démarrage | `kubectl logs` (et `--previous`), puis `Last State` dans `kubectl describe pod` |
| `OOMKilled` | l'application a dépassé sa limite de mémoire (souvent visible dans `Last State` plutôt que dans `STATUS`) | `kubectl describe pod` (`Last State`) |
| `Running` mais `0/1` | l'application tourne, mais sa sonde de *readiness* échoue | `kubectl describe pod` (événements `Unhealthy`) |
| `Terminating` | le pod est en train de s'arrêter | patienter |

Le statut `Pending` est expliqué dans [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending), `CrashLoopBackOff` dans [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update), et les sondes dans [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes).

> 🗣️ **En réunion projet.** Aucun de ces statuts ne veut dire « Kubernetes est en panne ». Chacun décrit un problème de l'application, de son image ou de sa configuration, que Kubernetes signale. La première question à poser reste la même : « *que disent la description du pod et ses logs ?* »

> 📖 [Déboguer un pod](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/) · [Le cycle de vie d'un pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) · [Les images et leur téléchargement](https://kubernetes.io/docs/concepts/containers/images/) · [Réserver et limiter la mémoire](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)

## Récap

- La colonne `STATUS` dit quelle sorte de panne c'est ; `describe` et `logs` disent pourquoi.
- `ImagePullBackOff` : l'image ne se télécharge pas. La cause est dans les événements du pod.
- `CrashLoopBackOff` : l'application plante en boucle. La cause est dans ses logs.
- `OOMKilled` : l'application a dépassé sa limite de mémoire. La cause est dans `Last State`, pas dans les logs.

```quiz
title: "Lire une panne"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Un pod affiche `ImagePullBackOff` et `RESTARTS` à 0. Où chercher la cause ?"
    answers:
      - "Dans `kubectl logs` : l'application y écrit son erreur"
      - "Dans les événements de `kubectl describe pod` : aucun conteneur n'a démarré, il n'y a donc pas de logs"
      - "Dans `kubectl rollout history`"
    correct: 2
    explanation: "L'image n'a pas pu être téléchargée : l'application n'a jamais tourné. Les événements du pod donnent la raison du refus."
  - prompt: "`kubectl describe pod` affiche `Last State: Terminated`, `Reason: OOMKilled`. Que s'est-il passé ?"
    answers:
      - "Le nœud n'avait plus de place pour le pod"
      - "L'application a dépassé sa limite de mémoire, et le système l'a arrêtée"
      - "La sonde de liveness a échoué"
    correct: 2
    explanation: "OOM veut dire Out Of Memory. Il faut relever la limite, ou chercher pourquoi l'application consomme autant."
  - prompt: "Un pod est `Running`, mais sa colonne `READY` indique `0/1`. Que faut-il comprendre ?"
    answers:
      - "L'application tourne, mais sa sonde de readiness échoue : le pod ne reçoit pas de visiteurs"
      - "Le pod attend une place sur un nœud"
      - "L'image est en cours de téléchargement"
    correct: 1
    explanation: "`Running` ne suffit pas : c'est la colonne `READY` qui dit si le pod est compté comme prêt."
```

```ack
J'ai compris
```
