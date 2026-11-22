---
title: "Lancer un pod à la main"
description: "kubectl run lance un pod, kubectl get pods le montre, kubectl delete pod le supprime"
---

Vous commencez par la façon la plus directe de lancer l'application : demander à Kubernetes de démarrer **un pod**. Vous le supprimez ensuite, pour voir ce que Kubernetes en fait.

## Préparer le dossier et vérifier le cluster

Cette section prépare votre poste : c'est un support, que vous pouvez jouer sans vous y attarder. Vérifiez d'abord que votre cluster répond. `kubectl` est la « télécommande » de Kubernetes, l'outil en ligne de commande qui lui parle, et `get nodes` liste les **nœuds**, les machines du cluster :

```exec
kubectl get nodes
```

Le nœud doit être `Ready` :

```text
NAME         STATUS   ROLES           AGE   VERSION
layers-k8s   Ready    control-plane   12m   v1.34.0
```

Si la commande échoue avec `connection refused`, le cluster est arrêté : le lab [Démarrer son cluster Kubernetes avec minikube](https://byte-layers.com/fr/labs/k8s-minikube-install) dit comment le relancer.

Créez ensuite le dossier de ce cours. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l1
```

Si vous avez déjà suivi ce cours, il en reste peut-être des objets dans le cluster. Cette commande supprime le Deployment `vote` et le pod `vote-solo`, écrits sous la forme `type/nom`, pour repartir de zéro. `--ignore-not-found` lui dit de ne pas se plaindre quand il n'y a rien à supprimer : la première fois, elle n'affiche donc rien.

```exec
kubectl delete deployment/vote pod/vote-solo --ignore-not-found
```

## Lancer un pod avec kubectl run

> 🧠 **Un pod, c'est quoi ?** C'est la plus petite unité que Kubernetes sait faire tourner : une « enveloppe » autour d'un (ou parfois plusieurs) conteneur(s). Pour l'instant, retenez simplement **1 pod = 1 exemplaire de l'application qui tourne**.

Lancez un pod à partir de l'image de `vote`. Le tableau sous la commande en détaille les trois morceaux.

```exec
kubectl run vote-solo --image=ghcr.io/bngams/kube-vote:1.0
```

| Élément | Rôle |
|---|---|
| `kubectl run` | « lance un pod » |
| `vote-solo` | le nom qu'on donne au pod |
| `--image=ghcr.io/bngams/kube-vote:1.0` | l'image à utiliser : `ghcr.io` est l'adresse du registre, `bngams` le compte qui la publie, `kube-vote` le nom de l'image, `1.0` sa version |

Kubernetes répond `pod/vote-solo created`, puis télécharge l'image et démarre le conteneur. Plutôt que de guetter, demandez à `kubectl wait` d'attendre que le pod soit prêt. `--for=condition=Ready` désigne ce qu'on attend : Kubernetes tient pour chaque pod une liste de **conditions**, et `Ready` devient vraie quand le pod peut recevoir des visiteurs. `--timeout=120s` abandonne au bout de deux minutes :

```exec
kubectl wait --for=condition=Ready pod/vote-solo --timeout=120s
```

Quand la commande répond `pod/vote-solo condition met`, regardez où en est le pod :

```exec
kubectl get pods
```

Le terminal affiche la liste des pods. `1/1` veut dire qu'un conteneur est prêt sur un attendu, et `Running` qu'il tourne :

```text
NAME        READY   STATUS    RESTARTS   AGE
vote-solo   1/1     Running   0          8s
```

## Supprimer le pod lancé à la main

Vous allez maintenant **supprimer ce pod**, comme le ferait une panne ou une erreur de manipulation. Avant de le faire, faites une prévision : une fois le pod supprimé, Kubernetes va-t-il en relancer un ?

À vous de taper la commande dans le terminal : elle s'écrit `kubectl delete pod`, suivie du nom du pod. La suppression peut prendre quelques secondes, le temps que l'application s'arrête proprement. Kubernetes répond ensuite :

```text
pod "vote-solo" deleted from default namespace
```

Listez de nouveau les pods :

```exec
kubectl get pods
```

Il n'y a plus rien :

```text
No resources found in default namespace.
```

> 🗂️ **« default namespace » ?** Un *namespace* est un espace de rangement à l'intérieur du cluster. Pour l'instant, tout ce que vous créez va dans celui qui s'appelle `default`. En entreprise, chaque équipe ou projet a généralement le sien : un prochain cours y est consacré.

```hint
Le pod s'appelle `vote-solo` : c'est le nom que vous lui avez donné avec `kubectl run`.
```

```verify
if ! kubectl version --request-timeout=10s >/dev/null 2>&1; then
  echo "kubectl ne joint pas le cluster. Vérifiez qu'il est démarré avec kubectl get nodes, puis réessayez."
  exit 1
fi
if kubectl get pod vote-solo >/dev/null 2>&1; then
  echo "Le pod vote-solo existe encore. Supprimez-le : kubectl delete pod, suivi de son nom."
  exit 1
fi
if ! kubectl get events --field-selector involvedObject.name=vote-solo -o jsonpath='{range .items[*]}{.reason}{"\n"}{end}' 2>/dev/null | grep -q Killing; then
  echo "Le cluster n'a aucune trace d'un pod vote-solo lancé puis supprimé depuis une heure. Lancez-le avec la commande kubectl run de la section précédente, attendez qu'il soit prêt, puis supprimez-le."
  exit 1
fi
echo "Le pod vote-solo a été lancé puis supprimé, et personne ne l'a relancé."
```

````solution
```bash
kubectl delete pod vote-solo
```
````

> 🧠 **Ce qui vient de se passer.** Le pod a disparu et **personne ne l'a relancé**. Vous avez donné un **ordre** à Kubernetes (« lance ce pod »), il l'a exécuté, et son travail s'est arrêté là. Pour qu'il surveille l'application et la relance en cas de problème, il faut lui donner un **objectif** plutôt qu'un ordre. C'est le rôle du **Deployment**, que vous écrivez à l'étape suivante.

> 📖 [`kubectl run`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/) · [`kubectl delete`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/) · [Les pods](https://kubernetes.io/fr/docs/concepts/workloads/pods/pod/)
