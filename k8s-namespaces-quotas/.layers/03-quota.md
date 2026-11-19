---
title: "Plafonner un namespace avec un quota"
description: "Créer un quota de 12 pods, demander plus que le plafond, lire le refus de Kubernetes, puis revenir à l'état du fichier"
---

Rien n'empêche aujourd'hui une équipe de lancer des centaines de pods dans son namespace, au détriment des autres. Un **quota** (*ResourceQuota*) fixe un plafond pour tout un namespace. Vous allez en poser un sur `vote-app`, puis le dépasser volontairement pour voir ce que fait Kubernetes.

Si vous reprenez ce cours après avoir redémarré votre cluster, lancez d'abord `kubectl get pods` : la réponse doit montrer les pods de l'application. Si elle parle de `default`, relancez `kubectl config set-context --current --namespace=vote-app`.

## Créer un quota de 12 pods

Créez un quota nommé `pods-max`, qui limite `vote-app` à **12 pods**. `--hard` donne le plafond, sous la forme `ressource=valeur`. Comme pour le namespace, la commande crée l'objet sans fichier : c'est un raccourci, et dans un vrai projet un quota s'écrit dans un fichier, comme le reste.

```exec
kubectl create quota pods-max --hard=pods=12
```

Un quota tient ses comptes. Demandez-lui où il en est :

```exec
kubectl describe quota pods-max
```

Il annonce son plafond (`Hard`) et ce qui est déjà consommé (`Used`) : les 7 pods de l'application.

```text
Name:       pods-max
Namespace:  vote-app
Resource    Used  Hard
--------    ----  ----
pods        7     12
```

```verify
hard=$(kubectl get resourcequota -n vote-app -o jsonpath='{range .items[*]}{.spec.hard.pods}{"\n"}{end}' 2>/dev/null)
if [ -z "$hard" ]; then
  echo "Aucun quota sur les pods dans le namespace vote-app."
  echo "Lancez : kubectl create quota pods-max --hard=pods=12 -n vote-app"
  exit 1
fi
if ! echo "$hard" | grep -qx 12; then
  echo "Un quota existe dans vote-app, mais son plafond de pods n'est pas 12 (trouvé : $(echo $hard))."
  echo "Retrouvez son nom avec kubectl get quota -n vote-app, supprimez-le avec kubectl delete quota suivi de ce nom, puis recréez-le avec --hard=pods=12."
  exit 1
fi
echo "Le namespace vote-app est plafonné à 12 pods."
```

## Demander plus de pods que le quota n'en permet

Votre application compte 7 pods, dont 3 pour `vote`. Vous allez demander 10 exemplaires de `vote`, ce qui porterait le total à 14. Avant de lancer la commande, faites votre prévision : combien d'exemplaires de `vote` Kubernetes va-t-il créer ?

```exec
kubectl scale deployment vote --replicas=10
```

Laissez passer quelques secondes, puis regardez le Deployment :

```exec
kubectl get deployment vote
```

Le Deployment en veut 10, mais il n'en a que 8 :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   8/10    8            8           15s
```

Huit exemplaires seulement ont été créés (`READY` peut rester un peu plus bas quelques secondes, le temps qu'ils démarrent). Les 8 `vote` et les 4 autres pods font 12, le plafond. Le quota le confirme :

```exec
kubectl describe quota pods-max
```

Le plafond est atteint :

```text
Name:       pods-max
Namespace:  vote-app
Resource    Used  Hard
--------    ----  ----
pods        12    12
```

Kubernetes ne se contente pas de refuser : il explique son refus dans les **événements** du namespace. `--field-selector reason=FailedCreate` ne garde que les créations refusées :

```exec
kubectl get events --field-selector reason=FailedCreate
```

Le Deployment a essayé plusieurs fois, d'où plusieurs lignes. Chacune est longue, et se lit jusqu'au bout :

```text
LAST SEEN   TYPE      REASON         OBJECT                     MESSAGE
8s          Warning   FailedCreate   replicaset/vote-fc57495c   Error creating: pods "vote-fc57495c-k45rt" is forbidden: exceeded quota: pods-max, requested: pods=1, used: pods=12, limited: pods=12
```

Tout y est : le quota qui a refusé (`pods-max`), ce qui était demandé (1 pod de plus), ce qui est consommé (12) et le plafond (12). Le Deployment, lui, n'abandonne pas : il continue de vouloir 10 exemplaires, et les créera dès que le quota le permettra.

## Revenir à l'état du fichier

Le nombre 10 n'a jamais été écrit dans votre fichier : `kubectl scale` a modifié le cluster directement. Remettez l'application dans son état normal en réappliquant le fichier, qui dit 3 réplicas. C'est l'idée de [Le fichier fait foi](https://byte-layers.com/fr/bits/file-is-the-source).

```exec
kubectl apply -f ~/byte-layers/kubernetes/l4/app/vote.deploy.yml
```

Attendez que le Deployment ait retrouvé ses 3 exemplaires. `kubectl rollout status` suit un Deployment jusqu'à ce qu'il soit dans l'état demandé :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Interrogez le quota une dernière fois :

```exec
kubectl describe quota pods-max
```

Le compte revient à 7 pods sur 12. Juste après la commande précédente, il affiche encore 12 : les pods en trop mettent une trentaine de secondes à s'arrêter, et ils comptent jusqu'au bout. Relancez la commande jusqu'à lire 7.

```text
Name:       pods-max
Namespace:  vote-app
Resource    Used  Hard
--------    ----  ----
pods        7     12
```

Le quota reste en place : `vote-app` garde son plafond pour la suite.

```verify
want=$(kubectl get deployment vote -n vote-app -o jsonpath='{.spec.replicas}' 2>/dev/null)
if [ -z "$want" ]; then
  echo "Le Deployment vote n'est pas dans vote-app. Reprenez l'étape précédente."
  exit 1
fi
if [ "$want" != "3" ]; then
  echo "Le Deployment vote demande $want exemplaires, pas 3."
  echo "Réappliquez le fichier : kubectl apply -f ~/byte-layers/kubernetes/l4/app/vote.deploy.yml"
  exit 1
fi
if ! kubectl rollout status deployment/vote -n vote-app --timeout=45s >/dev/null 2>&1; then
  echo "Le Deployment vote demande bien 3 exemplaires, mais ils ne sont pas tous prêts. Attendez un peu, puis réessayez."
  exit 1
fi
if ! kubectl get resourcequota -n vote-app -o jsonpath='{range .items[*]}{.spec.hard.pods}{"\n"}{end}' 2>/dev/null | grep -qx 12; then
  echo "Le quota de 12 pods n'est plus dans vote-app. Recréez-le : kubectl create quota pods-max --hard=pods=12 -n vote-app"
  exit 1
fi
echo "vote est revenu à 3 exemplaires, et le quota de 12 pods est toujours en place."
```

## Les autres plafonds : ResourceQuota et LimitRange

Un quota peut plafonner bien d'autres choses que le nombre de pods, et il a un cousin qui regarde chaque pod un par un :

| Règle | Ce qu'elle encadre | Exemple |
|---|---|---|
| **ResourceQuota** | le **total** d'un namespace | 12 pods, 4 processeurs réservés, 8 Gi de mémoire, 2 volumes… |
| **LimitRange** (plage de limites) | chaque pod **individuellement** | réservation par défaut si le fichier n'en donne pas, maximum par conteneur |

> 🗣️ **En réunion projet.** Les quotas servent à **partager un cluster** entre équipes sans qu'une seule ne monopolise tout, et à **répartir les coûts** : la somme des quotas dit à quoi chaque équipe a droit.

> 📖 [Les quotas de ressources](https://kubernetes.io/docs/concepts/policy/resource-quotas/) · [Les LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) · [Les namespaces](https://kubernetes.io/fr/docs/concepts/overview/working-with-objects/namespaces/)

## Récap

- Un **namespace** range une application, une équipe ou un environnement. On évite de tout mettre dans `default`.
- `kubectl create namespace` le crée, `-n` adresse une commande à un autre namespace, `-A` les montre tous, et `kubectl config set-context --current --namespace=…` change celui dans lequel `kubectl` travaille.
- Les mêmes fichiers se déploient dans n'importe quel namespace : ils n'en citent aucun.
- Un **quota** plafonne les ressources d'un namespace. Au-delà, Kubernetes refuse de créer, et le dit dans les événements : `exceeded quota`.
- Un quota sert à partager un cluster et à répartir les coûts.

## Questions sur les namespaces et les quotas

```quiz
title: "Namespaces et quotas"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Vous lancez `kubectl delete -f ~/byte-layers/kubernetes/l4/app/`. Que deviennent les fichiers du dossier ?"
    answers:
      - "Ils sont supprimés avec les objets"
      - "Ils restent intacts : seuls les objets du cluster sont supprimés"
      - "Ils sont vidés, mais gardés"
    correct: 2
    explanation: "`kubectl delete -f` lit les fichiers pour savoir quoi supprimer dans le cluster. Il ne les modifie pas."
  - prompt: "Après `kubectl config set-context --current --namespace=vote-app`, que fait `kubectl get pods` ?"
    answers:
      - "Il liste les pods de tous les namespaces"
      - "Il liste les pods de `default`, comme avant"
      - "Il liste les pods de `vote-app`"
    correct: 3
    explanation: "Le réglage change le namespace que `kubectl` utilise quand vous n'en précisez pas. `-n` vise un autre namespace, `-A` les vise tous."
  - prompt: "Un namespace est plafonné à 12 pods et en contient 7. Vous demandez 10 exemplaires d'un Deployment qui en avait 3. Que se passe-t-il ?"
    answers:
      - "La commande est refusée, et le Deployment reste à 3"
      - "Le Deployment passe à 8 exemplaires, et les 2 derniers sont refusés par le quota"
      - "Les 10 exemplaires sont créés, et 2 autres pods du namespace sont arrêtés"
    correct: 2
    explanation: "Le quota refuse chaque pod qui ferait dépasser le plafond, un par un. L'événement `exceeded quota` l'explique, et le Deployment continue de vouloir 10 exemplaires."
```
