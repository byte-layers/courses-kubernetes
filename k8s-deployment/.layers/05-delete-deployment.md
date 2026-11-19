---
title: "Supprimer le Deployment, puis le remettre"
description: "Voir la chaîne Deployment, ReplicaSet, pod, supprimer l'état souhaité, puis le rétablir depuis le fichier"
---

Dernière expérience : que se passe-t-il si l'on supprime le **Deployment** lui-même ? Vous regardez d'abord ce qu'il possède, vous le supprimez, puis vous remettez l'application en place depuis votre fichier.

## Deployment, ReplicaSet, pod : qui possède qui

`kubectl get` accepte plusieurs types d'objets à la fois, séparés par des virgules. Demandez les Deployments, les ReplicaSets et les pods :

```exec
kubectl get deployments,replicasets,pods
```

Entre votre Deployment et son pod, il y a un objet que vous n'avez pas écrit :

```text
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vote   1/1     1            1           6m

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/vote-644d47956b   1         1         1       6m

NAME                        READY   STATUS    RESTARTS      AGE
pod/vote-644d47956b-qm48q   1/1     Running   1 (2m ago)    4m
```

> 🧠 **Qui possède qui ?** Un Deployment ne gère pas ses pods directement : il passe par un intermédiaire, le **ReplicaSet**, dont le seul travail est de maintenir le bon nombre de pods. C'est lui qui a recréé votre pod quand vous l'avez supprimé. C'est aussi de lui que vient le premier suffixe du nom des pods (`644d47956b`).

La chaîne se lit de haut en bas, chaque objet possédant le suivant :

```text
Deployment vote                     <- l'objectif, que vous écrivez
└── ReplicaSet vote-644d47956b      <- le "gardien du nombre", créé automatiquement
    └── Pod vote-644d47956b-qm48q   <- l'exemplaire qui tourne
```

## Supprimer le Deployment

Faites une prévision : si vous supprimez le Deployment, son pod sera-t-il recréé, comme quand vous aviez supprimé le pod seul ?

À vous de supprimer le Deployment : la commande suit le modèle de `kubectl delete pod`, avec le type `deployment` et le nom `vote`. Kubernetes répond :

```text
deployment.apps "vote" deleted from default namespace
```

Regardez tout de suite ce que deviennent les trois objets :

```exec
kubectl get deployments,replicasets,pods
```

Le Deployment et son ReplicaSet ont disparu, et le pod est en train de s'arrêter. Si vous lisez déjà `No resources found`, il avait fini de s'arrêter.

```text
NAME                        READY   STATUS        RESTARTS      AGE
pod/vote-644d47956b-qm48q   1/1     Terminating   1 (3m ago)    5m
```

Relancez la même commande quelques secondes plus tard :

```exec
kubectl get deployments,replicasets,pods
```

Il ne reste rien :

```text
No resources found in default namespace.
```

Cette fois, le pod a disparu **et ne revient pas**. En supprimant le Deployment, vous avez supprimé **l'état souhaité** : Kubernetes n'a plus aucun objectif pour `vote`, il fait donc le ménage, et toute la chaîne part avec lui.

```hint
`kubectl delete` prend un type puis un nom. Pour le pod, c'était `kubectl delete pod` suivi de son nom ; ici, le type est `deployment`.
```

```verify
FILE="$HOME/byte-layers/kubernetes/l1/vote.deploy.yml"
if ! kubectl version --request-timeout=10s >/dev/null 2>&1; then
  echo "kubectl ne joint pas le cluster. Vérifiez qu'il est démarré avec kubectl get nodes, puis réessayez."
  exit 1
fi
if [ ! -f "$FILE" ] || grep -q '# TODO' "$FILE"; then
  echo "Le fichier ~/byte-layers/kubernetes/l1/vote.deploy.yml est absent, ou contient encore des # TODO. Reprenez l'étape « Décrire l'état souhaité : le Deployment » avant celle-ci."
  exit 1
fi
if kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote existe encore. Supprimez-le : kubectl delete, puis le type deployment, puis son nom."
  exit 1
fi
LEFT=$(kubectl get pods -l app=vote -o name)
for i in 1 2 3 4 5 6 7 8 9 10; do
  [ -z "$LEFT" ] && break
  sleep 3
  LEFT=$(kubectl get pods -l app=vote -o name)
done
if [ -n "$LEFT" ]; then
  echo "Le Deployment est supprimé, mais il reste un pod de vote : $LEFT. S'il est en Terminating, attendez quelques secondes et réessayez."
  exit 1
fi
echo "Plus de Deployment, plus de ReplicaSet, plus de pod : il n'y a plus d'état souhaité pour vote."
```

````solution
```bash
kubectl delete deployment vote
```
````

## Remettre l'application en place depuis le fichier

Lancez la vérification de la section précédente avant de continuer : celle-ci recrée le Deployment.

Le cluster n'a plus rien, mais vous n'avez rien perdu : l'état souhaité est toujours écrit dans `~/byte-layers/kubernetes/l1/vote.deploy.yml`. C'est tout l'intérêt de le décrire dans un fichier.

À vous de remettre l'application en place : transmettez de nouveau ce fichier au cluster, avec la commande de l'étape « Décrire l'état souhaité : le Deployment ». Attendez ensuite que le pod soit prêt :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Regardez la chaîne une dernière fois :

```exec
kubectl get deployments,replicasets,pods
```

Les trois objets sont revenus, et le pod porte un nouveau nom, avec un compteur `RESTARTS` reparti de zéro :

```text
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vote   1/1     1            1           9s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/vote-644d47956b   1         1         1       9s

NAME                        READY   STATUS    RESTARTS   AGE
pod/vote-644d47956b-x7k2p   1/1     Running   0          9s
```

**Laissez ce Deployment en place** : le prochain cours du parcours s'en sert.

```hint
La commande qui transmet un fichier YAML au cluster est `kubectl apply`, avec `-f` suivi du chemin du fichier.
```

```verify
if ! kubectl version --request-timeout=10s >/dev/null 2>&1; then
  echo "kubectl ne joint pas le cluster. Vérifiez qu'il est démarré avec kubectl get nodes, puis réessayez."
  exit 1
fi
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote n'est pas revenu. Transmettez le fichier : kubectl apply -f ~/byte-layers/kubernetes/l1/vote.deploy.yml"
  exit 1
fi
SPEC=$(kubectl get deployment vote -o jsonpath='{.spec.replicas} {.spec.template.spec.containers[0].image}')
if [ "$SPEC" != "1 ghcr.io/bngams/kube-vote:1.0" ]; then
  echo "Le Deployment vote doit demander 1 exemplaire de ghcr.io/bngams/kube-vote:1.0 (trouvé : $SPEC). Corrigez le fichier, puis relancez kubectl apply."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "Le Deployment vote est revenu, mais son pod n'est pas encore prêt. Regardez kubectl get pods, puis réessayez."
  exit 1
fi
echo "L'application est de nouveau en place : 1 pod prêt, en version 1.0, pour le prochain cours."
```

````solution
```bash
kubectl apply -f ~/byte-layers/kubernetes/l1/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

## Récap

- **Un ordre** (`kubectl run`) est exécuté une fois, puis oublié. **Un état souhaité** (un Deployment) est surveillé en permanence.
- Kubernetes compare sans cesse **l'état souhaité** et **la réalité**, et corrige l'écart : c'est la **réconciliation**.
- On décrit l'état souhaité dans un fichier **YAML**, qu'on transmet avec `kubectl apply -f`.
- Quand un **pod est supprimé**, le Deployment, par son ReplicaSet, en crée un nouveau, sous un nouveau nom. Quand un **conteneur plante**, Kubernetes le redémarre dans le même pod, et `RESTARTS` augmente de 1.
- Supprimer le Deployment, c'est supprimer l'objectif : les pods disparaissent pour de bon.

Le fichier final se trouve dans [`solution/vote.deploy.yml`](../solution/vote.deploy.yml).

> 📖 [Les Deployments](https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/) · [Les ReplicaSets](https://kubernetes.io/fr/docs/concepts/workloads/controllers/replicaset/) · [kubectl, aide-mémoire officiel](https://kubernetes.io/fr/docs/reference/kubectl/cheatsheet/)

Pour vous tester sans guide : le challenge [Un Deployment qui se répare, sans guide](https://byte-layers.com/fr/challenges/k8s-deployment-challenge).

## Questions sur le Deployment

```quiz
title: "Déployer avec un Deployment"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Vous supprimez un pod géré par un Deployment. Que se passe-t-il ?"
    answers:
      - "Rien : le pod a disparu, il faut le relancer à la main"
      - "Un nouveau pod est créé, sous un nouveau nom"
      - "Le même pod redémarre, et RESTARTS augmente de 1"
    correct: 2
    explanation: "L'état souhaité dit 1 pod, la réalité en montre 0 : le Deployment, par son ReplicaSet, en crée un nouveau."
  - prompt: "Vous supprimez le Deployment lui-même. Et maintenant ?"
    answers:
      - "Ses pods continuent de tourner, sans surveillance"
      - "Kubernetes recrée le Deployment"
      - "Le ReplicaSet et les pods disparaissent avec lui, et rien ne revient"
    correct: 3
    explanation: "Supprimer le Deployment, c'est supprimer l'état souhaité : il n'y a plus d'objectif à tenir."
  - prompt: "Le conteneur d'un pod plante. Le nom du pod change-t-il ?"
    answers:
      - "Non : Kubernetes redémarre le conteneur dans le même pod, et RESTARTS augmente de 1"
      - "Oui : un nouveau pod remplace toujours un pod en panne"
    correct: 1
    explanation: "Le pod est resté en place : seul son conteneur a été relancé. Le nom ne change que quand le pod lui-même est remplacé."
  - prompt: "Dans un Deployment, pourquoi l'étiquette du `template` doit-elle être la même que celle du `selector` ?"
    answers:
      - "Parce que YAML interdit deux valeurs différentes pour une même clé"
      - "Parce que le selector dit au Deployment quels pods sont les siens : ceux qui portent cette étiquette"
      - "Parce que l'étiquette donne son nom au pod"
    correct: 2
    explanation: "Le template colle l'étiquette sur chaque pod fabriqué, et le selector s'en sert pour les reconnaître. Si elles diffèrent, Kubernetes refuse le fichier."
```
