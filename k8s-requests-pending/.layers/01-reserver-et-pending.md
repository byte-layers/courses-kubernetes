---
title: "Une réservation trop grande : le pod reste Pending"
description: "Remettre vote en place, voir sur quel nœud tournent les pods, réserver 64 processeurs et lire pourquoi le nouveau pod ne démarre pas"
---

Avant de réserver quoi que ce soit, vous regardez où tournent vos pods. Vous ajoutez ensuite une réservation à `vote`, volontairement démesurée, pour voir ce que fait Kubernetes quand il n'y a plus de place.

## Préparer le dossier et remettre vote en place

Cette section est un **support** : elle met le cluster dans l'état de départ, et vous pouvez la jouer sans vous y arrêter. Créez d'abord le dossier de travail. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l2
```

Écrivez ensuite le fichier du Deployment `vote` : 3 exemplaires de l'application `vote`, en version 1.0, sans aucune réservation pour l'instant.

```exec
cat > ~/byte-layers/kubernetes/l2/vote.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: vote
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
    spec:
      containers:
        - name: vote
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
EOF
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'il reçoit en entrée |
> | `> ~/byte-layers/kubernetes/l2/vote.deploy.yml` | envoie cette copie dans le fichier, qu'il crée, ou dont il remplace le contenu s'il existe déjà |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles, jusqu'à la ligne `EOF` |
> | les lignes entre les deux | le contenu du fichier |
> | `EOF` | marque la fin du contenu ; ce mot n'est pas écrit dans le fichier |

Transmettez ce fichier au cluster. Si `vote` tourne déjà avec ce réglage, `kubectl apply` répond `unchanged` et ne touche à rien :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
```

Attendez que les 3 pods soient prêts. `kubectl rollout status` suit le déploiement et rend la main quand il est terminé, ou au bout de 120 secondes :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

La dernière ligne doit être celle-ci. Si la commande échoue avec `connection refused`, votre cluster est arrêté : relancez-le comme dans le lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install).

```text
deployment "vote" successfully rolled out
```

```ack
vote tourne avec ses 3 réplicas
```

## Voir sur quel nœud tourne chaque pod

Un cluster Kubernetes est fait de plusieurs serveurs, les **nœuds**. Demandez à voir sur quel nœud tourne chaque pod, avec l'option `-o wide` (*output wide*, un affichage plus large) :

```exec
kubectl get pods -o wide
```

Quatre colonnes sont apparues à droite du tableau habituel. Lisez les deux premières, `IP` et `NODE`, et ignorez les deux dernières, `NOMINATED NODE` et `READINESS GATES`, qui servent à des cas avancés.

```text
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
vote-644d47956b-bkzd7   1/1     Running   0          3s    10.244.218.121   layers-k8s   <none>           <none>
vote-644d47956b-gd62n   1/1     Running   0          3s    10.244.218.120   layers-k8s   <none>           <none>
vote-644d47956b-r6m47   1/1     Running   0          3s    10.244.218.122   layers-k8s   <none>           <none>
```

| Colonne | Ce qu'elle dit |
|---|---|
| `IP` | l'adresse du pod dans le réseau interne du cluster. Chaque pod a la sienne |
| `NODE` | le nœud sur lequel il tourne : `layers-k8s` sur votre poste, `k3d-layers-k8s-server-0` sur un poste hébergé, où les adresses commencent par `10.42.` |

Vos trois pods sont sur **le même nœud**, puisque votre cluster d'apprentissage n'en a qu'un.

> 🧠 **Sur un vrai cluster**, il y a plusieurs nœuds, souvent répartis dans plusieurs salles ou zones. Un composant de Kubernetes, l'**ordonnanceur** (*scheduler*), choisit pour chaque nouveau pod le nœud le plus adapté, et essaie par défaut de répartir les réplicas plutôt que de tout mettre au même endroit. Si un nœud tombe en panne, ses pods sont recréés sur les nœuds restants au bout de quelques minutes : c'est la même réconciliation que pour un pod supprimé, à l'échelle d'un serveur entier. C'est pour cela qu'on demande **au moins 2 ou 3 réplicas** pour une application importante : un seul réplica, c'est un seul point de panne.

```ack
Mes trois pods sont sur le même nœud
```

## Réserver du processeur et de la mémoire : les requests

Un nœud n'a pas des ressources infinies. Pour que l'ordonnanceur puisse bien placer les pods, chaque conteneur peut **réserver** une part de processeur (CPU) et de mémoire : ce sont les **requests**. Un pod n'est placé sur un nœud que si la somme des réservations déjà faites sur ce nœud lui laisse assez de place, **même si le nœud est en réalité peu utilisé** : l'ordonnanceur compte les réservations, pas la consommation réelle.

Les deux ressources se mesurent ainsi :

| Ressource | Unité | Exemples |
|---|---|---|
| `cpu` | en processeurs. `1` = un processeur entier, `100m` = un dixième de processeur (*m* pour millième) | `2`, `500m`, `100m` |
| `memory` | en octets, avec un suffixe. `Mi` = mébioctet, environ un million d'octets | `64Mi`, `512Mi`, `1Gi` |

Faites une expérience : demandez des pods **démesurés**, qui réservent chacun **64 processeurs**. Aucun nœud d'un cluster d'apprentissage n'est aussi gros.

Ouvrez `~/byte-layers/kubernetes/l2/vote.deploy.yml` dans votre éditeur. Vous allez ajouter **4 lignes** à la toute fin du fichier, après la ligne `- containerPort: 80`. Voici où elles se placent (les lignes existantes sont reprises pour vous repérer) :

```yaml
          ports:
            - containerPort: 80
          resources:              # <- nouvelle ligne
            requests:             # <- nouvelle ligne
              cpu: # TODO : 64 processeurs, écrit simplement 64
              memory: 64Mi        # <- nouvelle ligne
```

Pour éviter les pièges d'indentation : placez le curseur à la fin de la ligne `- containerPort: 80`, appuyez sur Entrée, puis **effacez tous les espaces** que l'éditeur ajoute automatiquement, pour être en tout début de ligne. Tapez alors les 4 lignes avec leurs espaces : `resources:` commence **exactement sous le `p` de `ports:`** (10 espaces), `requests:` a 2 espaces de plus, `cpu` et `memory` encore 2 de plus. Enregistrez, puis appliquez :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
```

Si `kubectl apply` répond par une erreur au lieu de `configured`, ce tableau donne la cause la plus fréquente :

| Message d'erreur au moment du `apply` | Cause probable |
|---|---|
| un message qui commence par `error converting YAML to JSON`, par exemple `yaml: line 24: did not find expected '-' indicator` | un décalage d'indentation dans les nouvelles lignes, à la ligne indiquée ou juste avant |
| un long message qui se termine par `unknown field "spec.template.spec.containers[0].ports[0].resources"` | `resources:` est trop décalé vers la droite, sous `containerPort` au lieu de `ports` |

Avant de lister les pods, faites une prévision : que deviennent les trois pods qui tournaient, et combien de nouveaux pods voyez-vous ?

```exec
kubectl get pods
```

Si vous ne voyez encore que trois pods, relancez la commande. Vous devez en lire quatre :

```text
NAME                    READY   STATUS    RESTARTS   AGE
vote-584fbb796-6bvk2    0/1     Pending   0          6s
vote-644d47956b-bkzd7   1/1     Running   0          21s
vote-644d47956b-gd62n   1/1     Running   0          21s
vote-644d47956b-r6m47   1/1     Running   0          21s
```

Deux choses se passent à la fois :

- **Un nouveau pod reste bloqué en `Pending`** : Kubernetes ne trouve aucun nœud assez grand pour lui. Le pod ne disparaît pas : il attend qu'une place se libère, ou qu'on ajoute un nœud.
- **Les trois anciens pods tournent toujours.** Modifier le modèle de pod revient à demander une **nouvelle version** de l'application. Kubernetes la déploie progressivement : avec ses réglages par défaut et 3 réplicas, il ne retire un ancien pod qu'une fois un nouveau prêt. Puisque le nouveau ne démarre pas, il garde les anciens : **l'application reste disponible**. Ce mécanisme, la *mise à jour progressive*, est l'objet d'un prochain cours.

```hint
Les 4 lignes vont tout à la fin du fichier. `resources:` s'aligne sur `ports:`, à 10 espaces du bord, jamais avec une tabulation.
```

```hint
La ligne à compléter devient `              cpu: 64`, à 14 espaces du bord, au même niveau que `memory: 64Mi`. Enregistrez, puis relancez `kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml`.
```

```verify
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote n'existe pas. Rejouez la section « Préparer le dossier et remettre vote en place »."
  exit 1
fi
CPU=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')
case "$CPU" in
  64|64000m) : ;;
  "") echo "Le Deployment vote ne réserve pas de processeur. Ajoutez les 4 lignes à la fin du fichier, enregistrez, puis relancez kubectl apply."
      exit 1 ;;
  *)  echo "Le Deployment vote réserve $CPU de processeur, et non 64. Écrivez cpu: 64, enregistrez, puis relancez kubectl apply."
      exit 1 ;;
esac
found=0
for i in 1 2 3 4 5 6 7 8 9 10; do
  REASON=$(kubectl get pods -l app=vote --field-selector=status.phase=Pending -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="PodScheduled")].reason}{" "}{end}')
  case "$REASON" in *Unschedulable*) found=1; break ;; esac
  sleep 2
done
if [ "$found" != 1 ]; then
  echo "Aucun pod de vote n'est bloqué en Pending faute de place. Regardez kubectl get pods, puis réessayez."
  exit 1
fi
AVAIL=$(kubectl get deployment vote -o jsonpath='{.status.availableReplicas}')
if [ "$AVAIL" != "3" ]; then
  echo "Un pod est bien Pending, mais seulement ${AVAIL:-0} ancien(s) pod(s) sur 3 répondent. Rejouez la section « Préparer le dossier et remettre vote en place », puis refaites le changement."
  exit 1
fi
echo "Un pod à 64 processeurs attend en Pending, et les 3 anciens pods servent toujours."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué. La dernière ligne laisse à l'ordonnanceur le temps de constater qu'il n'a pas de place :

```bash
cat > ~/byte-layers/kubernetes/l2/vote.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: vote
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
    spec:
      containers:
        - name: vote
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 64
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
sleep 5
```
````

## Lire pourquoi le pod est bloqué

Le Deployment résume la situation en une ligne :

```exec
kubectl get deployments
```

`READY 3/3` : trois pods servent toujours les visiteurs. `UP-TO-DATE 1` : un seul pod correspond à la nouvelle version demandée, et c'est celui qui attend. Au bout de 10 minutes, Kubernetes finira par signaler que la mise à jour n'avance pas, mais sans toucher aux pods qui fonctionnent.

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   3/3     1            3           2m21s
```

Demandez maintenant à Kubernetes pourquoi le pod est bloqué. `kubectl describe` donne le détail d'un objet, avec ses événements ; il lui faut le nom du pod `Pending`, que la commande entre parenthèses va chercher pour vous :

```exec
kubectl describe $(kubectl get pods --field-selector=status.phase=Pending -o name)
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl get pods` | liste les pods |
> | `--field-selector=status.phase=Pending` | … en ne gardant que ceux dont le statut est `Pending` |
> | `-o name` | … et en n'affichant que leur nom, sous la forme `pod/vote-584fbb796-6bvk2` |
> | `$( … )` | exécute d'abord la commande entre parenthèses, et met sa réponse à cet endroit de la ligne |
> | `kubectl describe` | donne le détail de l'objet ainsi nommé : la ligne revient à taper `kubectl describe pod/vote-584fbb796-6bvk2` |

Tout en bas, la section `Events` donne la raison :

```text
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  44s (x3 over 60s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

« *0 nœud disponible sur 1 : processeur insuffisant* ». Le message vient du `default-scheduler`, l'ordonnanceur qui choisit le nœud de chaque pod. La fin de la ligne dit qu'il n'a trouvé aucun pod moins important à déloger pour faire de la place.

```ack
J'ai lu « Insufficient cpu » dans les événements du pod
```

> 📖 [Gestion des ressources des conteneurs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [L'ordonnanceur de Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) · [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)

## Questions sur les réservations

```quiz
title: "Une réservation trop grande"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Un nœud a 4 processeurs, presque inoccupés en réalité. Ses pods en réservent déjà 3,5. Un nouveau pod en réserve 1. Est-il placé sur ce nœud ?"
    answers:
      - "Oui : le nœud est presque inoccupé"
      - "Non : l'ordonnanceur compte les réservations, pas la consommation réelle"
      - "Oui, mais il tournera au ralenti"
    correct: 2
    explanation: "3,5 + 1 dépasse 4. Le pod reste `Pending` tant qu'aucun nœud n'a assez de place réservable."
  - prompt: "Que veut dire `cpu: 100m` ?"
    answers:
      - "100 processeurs"
      - "100 mégaoctets de mémoire"
      - "Un dixième de processeur"
    correct: 3
    explanation: "*m* veut dire millième : `100m` = 100 millièmes de processeur. La mémoire, elle, s'écrit avec un suffixe comme `Mi`."
  - prompt: "Après le changement à 64 processeurs, pourquoi l'application répond-elle encore ?"
    answers:
      - "Parce que le pod `Pending` répond quand même aux visiteurs"
      - "Parce que Kubernetes ne retire un ancien pod qu'une fois un nouveau prêt, et que le nouveau ne démarre pas"
      - "Parce que Kubernetes a refusé le fichier"
    correct: 2
    explanation: "Le fichier a été accepté, et un pod de la nouvelle version a été créé. Comme il ne démarre pas, les trois anciens restent en place."
  - prompt: "Où lire la raison pour laquelle un pod reste `Pending` ?"
    answers:
      - "Dans la section `Events`, à la fin de `kubectl describe` sur ce pod"
      - "Dans la colonne `READY` de `kubectl get pods`"
      - "Dans la page de l'application"
    correct: 1
    explanation: "L'ordonnanceur y écrit un événement `FailedScheduling`, avec la ressource qui manque."
```
