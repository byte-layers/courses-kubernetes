---
title: "Pod recréé ou conteneur redémarré"
description: "Deux pannes, deux réparations : un pod supprimé est remplacé par un autre, un conteneur planté est relancé dans le même pod. Savoir les distinguer dans kubectl get pods, et en réunion."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 15
requires: "0.4.0"
---

« Le pod a été recréé » et « le pod redémarre » se ressemblent, et ne disent pas la même chose. Dans le premier cas, un **nouveau pod** a pris la place de l'ancien ; dans le second, le pod est resté et c'est son **conteneur** qui a été relancé. Vous saurez lire la différence dans `kubectl get pods`, et ce qu'elle change pour l'application.

**Prérequis** : savoir appliquer un Deployment et ouvrir un tunnel avec `kubectl port-forward`, ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment). Cette page se lit après ce cours. Il vous faut un cluster qui répond à `kubectl get nodes`, `curl`, et le port **28611** libre.

# Pod recréé ou conteneur redémarré

## Un pod de démonstration, à part de vote

Les deux pannes se provoquent sur un Deployment fait pour cela, `restart-demo`, à supprimer à la fin : votre Deployment `vote` n'est pas touché. `kubectl create deployment` en crée un sans fichier, à partir d'un nom et d'une image. C'est un raccourci pour un essai jetable : une application qui dure se décrit dans un fichier, transmis avec `kubectl apply`.

```exec
kubectl create deployment restart-demo --image=ghcr.io/bngams/kube-vote:1.0
```

Attendez que son pod soit prêt :

```exec
kubectl rollout status deployment/restart-demo --timeout=120s
```

Regardez ce pod. `-l app=restart-demo` ne garde que les pods qui portent cette étiquette, celle que `kubectl create deployment` a choisie, et `-o wide` ajoute des colonnes, dont l'adresse IP du pod dans le cluster :

```exec
kubectl get pods -l app=restart-demo -o wide
```

Retenez quatre colonnes : `NAME`, `RESTARTS`, `AGE` et `IP`. Les dernières colonnes ne servent pas ici.

```text
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
restart-demo-cb69f5c66-s7xzg   1/1     Running   0          19s   10.244.218.126   layers-k8s   <none>           <none>
```

## Première panne : le pod disparaît

Supprimez le pod, comme le ferait une erreur de manipulation ou la perte du serveur qui le porte. Là aussi, `-l` désigne le pod par son étiquette, ce qui évite de recopier son nom :

```exec
kubectl delete pod -l app=restart-demo
```

Attendez que le Deployment ait retrouvé un pod prêt :

```exec
kubectl rollout status deployment/restart-demo --timeout=120s
```

Regardez de nouveau ces colonnes :

```exec
kubectl get pods -l app=restart-demo -o wide
```

Tout a changé, sauf le compteur : le nom se termine par un autre suffixe, l'adresse IP est différente, et `RESTARTS` vaut toujours 0.

```text
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
restart-demo-cb69f5c66-lc5nf   1/1     Running   0          9s    10.244.218.87    layers-k8s   <none>           <none>
```

Ce n'est pas l'ancien pod qui est revenu. C'est **un autre pod**, fabriqué par le Deployment d'après son modèle, pour retrouver le nombre d'exemplaires souhaité. On dit que le pod a été **recréé**.

## Deuxième panne : le conteneur plante

L'autre panne vient de l'application elle-même : un bug, un manque de mémoire. L'image de `vote` contient une page `/crash` qui arrête le programme du conteneur. Pour l'atteindre, ouvrez un tunnel vers le pod, en arrière-plan, sur le port 28611 :

```exec
kubectl port-forward deployment/restart-demo 28611:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande.** C'est la construction du cours [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), qui l'explique morceau par morceau : `28611:80` relie le port 28611 de votre machine au port 80 du conteneur, `> /dev/null 2>&1` jette ce que la commande afficherait, et `&` la lance en arrière-plan.

Demandez la page `/crash` avec `curl` :

```exec
curl -s http://localhost:28611/crash
```

L'application annonce son propre arrêt :

```text
💥 Le conteneur de restart-demo-cb69f5c66-lc5nf va s'arrêter... Kubernetes va le redémarrer.
```

Laissez passer une dizaine de secondes, puis regardez ces colonnes une dernière fois :

```exec
kubectl get pods -l app=restart-demo -o wide
```

Cette fois, rien n'a changé, sauf le compteur : même nom, même adresse IP, et `RESTARTS` vaut 1. Si `STATUS` affiche encore `Completed`, relancez la commande quelques secondes plus tard.

```text
NAME                           READY   STATUS    RESTARTS      AGE   IP              NODE         NOMINATED NODE   READINESS GATES
restart-demo-cb69f5c66-lc5nf   1/1     Running   1 (8s ago)    45s   10.244.218.87   layers-k8s   <none>           <none>
```

Le pod est resté en place, et Kubernetes a **redémarré le conteneur** à l'intérieur. Le tunnel, branché sur le pod, a tenu lui aussi. Arrêtez-le :

```exec
pkill -f "port-forward deployment/restart-demo 28611"
```

## Ce qui change, ce qui ne change pas

Les deux pannes se comparent ainsi :

| | Pod supprimé | Conteneur planté |
|---|---|---|
| Qui répare ? | le **Deployment**, qui crée un **nouveau pod** | le **kubelet**, l'agent de Kubernetes sur le nœud, qui **redémarre le conteneur** dans le même pod |
| Nom du pod | **change** | **identique** |
| Adresse IP du pod | **change** | **identique** |
| Colonne `RESTARTS` | repart à 0 | augmente de 1 |
| Un tunnel `port-forward` | casse, il faut le relancer | continue de fonctionner |

Pour savoir laquelle des deux a eu lieu, regardez donc deux colonnes de `kubectl get pods` : un `AGE` tout neuf avec `RESTARTS` à 0 désigne un pod recréé ; un `AGE` ancien avec `RESTARTS` qui a augmenté désigne un conteneur redémarré.

## Deux phrases qu'on entend en réunion

Ces deux pannes ont chacune leur phrase, qu'on entend souvent dans une équipe projet.

> 🗣️ « **Le pod a été recréé.** » Un pod a disparu (supprimé, ou perdu avec son serveur) et un autre l'a remplacé. C'est le fonctionnement normal de Kubernetes : un pod n'est pas fait pour durer, et rien ne doit dépendre de son nom ni de son adresse.

> 🗣️ « **Le pod redémarre en boucle.** » Celui qui dit cela parle de la colonne `RESTARTS` qui augmente sans cesse : l'application plante, Kubernetes la relance, elle replante… Le statut affiché devient alors `CrashLoopBackOff`. Kubernetes fait son travail, mais **l'application a un problème** que lui ne peut pas corriger. Un prochain cours vous fait provoquer cette panne, et la diagnostiquer.

## Supprimer le Deployment de démonstration

Il reste à ranger. Supprimer le Deployment supprime aussi son pod :

```exec
kubectl delete deployment restart-demo
```

> 📖 [Le cycle de vie d'un pod, et le redémarrage des conteneurs](https://kubernetes.io/fr/docs/concepts/workloads/pods/pod-lifecycle/) · [Les Deployments](https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/)

## Ce qu'il faut retenir

- Un pod **recréé** est un autre pod : nouveau nom, nouvelle adresse IP, `RESTARTS` à 0. C'est le Deployment qui le crée.
- Un conteneur **redémarré** reste dans le même pod : même nom, même adresse, `RESTARTS` + 1.
- « Redémarre en boucle » parle de `RESTARTS` qui grimpe : le problème est dans l'application, pas dans Kubernetes.

```quiz
prompt: "`kubectl get pods` affiche un pod dont AGE vaut 3 jours et RESTARTS vaut 12. Que s'est-il passé ?"
answers:
  - "Le pod a été supprimé et recréé 12 fois en 3 jours"
  - "Le pod est le même depuis 3 jours, et son conteneur a été redémarré 12 fois"
  - "Le Deployment a été appliqué 12 fois"
correct: 2
explanation: |
  Un pod recréé repart avec un AGE neuf et RESTARTS à 0. Ici le pod a 3 jours : c'est son conteneur qui a planté et que Kubernetes a relancé, 12 fois. L'application a sans doute un problème à chercher.
```

```ack
J'ai compris
```
