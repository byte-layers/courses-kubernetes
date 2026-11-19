---
title: "De 1 à 3 exemplaires"
description: "Remettre vote en place, demander 3 réplicas dans le fichier, voir quel pod répond, puis supprimer tous les pods d'un coup"
---

Pour avoir plusieurs exemplaires de `vote`, vous n'allez lancer aucun pod vous-même : vous allez **changer l'état souhaité**, et laisser Kubernetes calculer ce qu'il faut faire. Vous regardez ensuite lequel des exemplaires répond, puis ce qui se passe quand ils disparaissent tous en même temps.

## Préparer le dossier et remettre vote en place

Cette section est un **support** : elle met le cluster dans l'état de départ, et vous pouvez la jouer sans vous y arrêter. Créez d'abord le dossier de travail. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l2
```

Écrivez ensuite le fichier du Deployment `vote`, tel qu'il est à la fin du cours [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment) : un seul exemplaire de l'application `vote`, en version 1.0.

```exec
cat > ~/byte-layers/kubernetes/l2/vote.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: vote
spec:
  replicas: 1
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

Attendez que le pod soit prêt. `kubectl rollout status` suit le déploiement et rend la main quand il est terminé, ou au bout de 120 secondes :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Vérifiez enfin que le Deployment est là :

```exec
kubectl get deployments
```

Le terminal doit afficher `vote` avec `READY 1/1`. Si la commande échoue avec `connection refused`, votre cluster est arrêté : relancez-le comme dans le lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install).

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   1/1     1            1           10s
```

```ack
vote affiche 1/1
```

## Demander 3 exemplaires dans le fichier

Pour avoir 3 exemplaires de `vote`, il suffit de **changer l'état souhaité** : c'est le champ `replicas` du fichier.

Ouvrez `~/byte-layers/kubernetes/l2/vote.deploy.yml` dans votre éditeur. Repérez la ligne `replicas: 1`, vers le haut du fichier, sous le premier `spec:`. Remplacez **seulement le chiffre** pour demander 3 exemplaires, puis enregistrez. Voici la ligne à compléter :

```yaml
spec:
  replicas: # TODO : 3 exemplaires
```

Transmettez le nouvel état souhaité au cluster :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
```

Cette fois, le terminal ne dit plus `created`, mais `configured` : le Deployment existait déjà, il a été **mis à jour**.

```text
deployment.apps/vote configured
```

Attendez que les nouveaux pods soient prêts, avec la même commande d'attente :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Avant de lister les pods, faites une prévision : le premier pod, celui qui tournait déjà, a-t-il été remplacé ?

```exec
kubectl get pods
```

Deux nouveaux pods sont apparus, reconnaissables à leur âge. Le premier pod, lui, n'a pas bougé.

```text
NAME                    READY   STATUS    RESTARTS   AGE
vote-644d47956b-b7v6j   1/1     Running   0          21s
vote-644d47956b-kl4tf   1/1     Running   0          2s
vote-644d47956b-t8k5r   1/1     Running   0          2s
```

Le Deployment, lui, résume la situation en une ligne :

```exec
kubectl get deployments
```

`READY 3/3` se lit : 3 pods prêts sur 3 demandés.

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   3/3     3            3           21s
```

> 🧠 **Ce qui vient de se passer.** C'est la **même boucle de réconciliation** que pour un seul pod. Kubernetes a comparé l'état souhaité (3) avec la réalité (1), constaté qu'il en manquait 2, et les a créés. Vous n'avez pas dit « crée 2 pods », vous avez dit « je veux 3 pods » : c'est à lui de calculer ce qu'il faut faire.
>
> Les trois pods ont le même début de nom, `vote-644d47956b-` : ils sont fabriqués par **le même ReplicaSet**, à partir du même modèle. Ce sont des **réplicas**, des copies identiques et interchangeables.

```hint
Une seule ligne change dans le fichier : celle qui commence par `replicas:`, sous le premier `spec:`. Le reste du fichier ne bouge pas.
```

```hint
La ligne devient `  replicas: 3`, avec ses deux espaces au début. Enregistrez, puis relancez `kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml`.
```

```verify
F="$HOME/byte-layers/kubernetes/l2/vote.deploy.yml"
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote n'existe pas. Rejouez la section « Préparer le dossier et remettre vote en place »."
  exit 1
fi
IN_FILE=$(sed -E 's/#.*//' "$F" 2>/dev/null | grep -E '^[[:space:]]*replicas:' | head -1 | tr -cd '0-9')
if [ "$IN_FILE" != "3" ]; then
  echo "Le fichier vote.deploy.yml ne demande pas 3 réplicas (trouvé : ${IN_FILE:-rien}). Écrivez replicas: 3, enregistrez, puis relancez kubectl apply."
  exit 1
fi
WANT=$(kubectl get deployment vote -o jsonpath='{.spec.replicas}')
if [ "$WANT" != "3" ]; then
  echo "Le fichier demande 3 réplicas, mais le cluster en demande encore $WANT : il reste à lancer kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml"
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "Les 3 pods ne sont pas encore prêts. Regardez leur état avec kubectl get pods, puis réessayez."
  exit 1
fi
READY=$(kubectl get deployment vote -o jsonpath='{.status.readyReplicas}')
if [ "$READY" != "3" ]; then
  echo "Seulement ${READY:-0} pod(s) prêt(s) sur 3. Regardez leur état avec kubectl get pods, puis réessayez."
  exit 1
fi
echo "vote tourne en 3 exemplaires, et le fichier dit la même chose."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué :

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
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

## Qui répond aux visiteurs : un seul pod, toujours le même

Trois exemplaires tournent. Mais lequel répond quand on ouvre l'application ? Chaque pod de `vote` écrit son propre nom dans la page, sur une ligne « Servi par » : c'est ce qui va vous permettre de le savoir.

Ouvrez d'abord un tunnel entre le port 28620 de votre machine et l'application, avec `kubectl port-forward`. Cette commande occupe normalement le terminal tant que le tunnel est ouvert : la fin de la ligne la fait tourner en arrière-plan, pour qu'elle vous rende la main.

```exec
kubectl port-forward deployment/vote 28620:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward` | ouvre un tunnel temporaire entre votre machine et le cluster |
> | `deployment/vote` | la cible du tunnel : l'application du Deployment `vote` |
> | `28620:80` | le port 28620 de votre machine mène au port 80 du pod |
> | `> /dev/null` | jette ce que la commande affiche, pour ne pas encombrer le terminal |
> | `2>&1` | jette aussi ses messages d'erreur, au même endroit |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main tout de suite |

Demandez maintenant la page trois fois de suite. Avant de lancer, faites une prévision : avec trois pods, le nom affiché va-t-il changer d'une fois à l'autre ?

```exec
curl -s http://localhost:28620 | grep "Servi par"
curl -s http://localhost:28620 | grep "Servi par"
curl -s http://localhost:28620 | grep "Servi par"
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `curl -s http://localhost:28620` | demande la page de l'application ; `-s` n'affiche que la page, sans barre de progression |
> | `\|` | passe cette page à la commande qui suit, au lieu de l'afficher |
> | `grep "Servi par"` | ne garde que les lignes qui contiennent « Servi par » |

Si `curl` ne répond rien, le tunnel n'a pas fini de s'ouvrir : attendez une seconde et relancez. Vous lisez trois fois la même ligne, avec le nom d'un de vos pods :

```text
          Servi par : <strong>vote-644d47956b-b7v6j</strong>
          Servi par : <strong>vote-644d47956b-b7v6j</strong>
          Servi par : <strong>vote-644d47956b-b7v6j</strong>
```

C'est **toujours le même pod** qui répond. `kubectl port-forward` choisit un pod au démarrage et s'y branche : les deux autres tournent, mais ne reçoivent aucun visiteur. Pour voir la page elle-même, ouvrez `http://localhost:28620` dans un navigateur et rechargez-la plusieurs fois ; sur un poste hébergé, passez par l'onglet **PORTS** de VS Code.

> 💡 **Il manque un répartiteur.** Pour que les visiteurs soient répartis entre les réplicas, il faut placer devant eux un **Service** : une adresse unique qui distribue les requêtes à tous les pods portant l'étiquette `app: vote`. C'est l'objet d'un prochain cours, où vous verrez le nom du pod changer d'une requête à l'autre.

Le tunnel a fini son travail. Arrêtez-le : `pkill -f` arrête le programme dont la ligne de commande contient le texte entre guillemets.

```exec
pkill -f "port-forward deployment/vote 28620"
```

```ack
C'est toujours le même pod qui a répondu
```

## Panne générale : supprimer tous les pods d'un coup

Vous savez qu'un pod supprimé est recréé. Que se passe-t-il si **tous** les pods disparaissent en même temps, comme lors d'une grosse panne ? Plutôt que de copier trois noms, vous allez désigner les pods par leur **étiquette**. Avant de lancer, faites une prévision : combien de pods retrouverez-vous, et porteront-ils les mêmes noms ?

```exec
kubectl delete pods -l app=vote
```

| Élément | Rôle |
|---|---|
| `delete pods` | supprimer des pods |
| `-l app=vote` | … tous ceux qui portent l'étiquette (*label*) `app: vote`, quel que soit leur nom. C'est le même post-it que celui utilisé par le Deployment pour reconnaître ses pods |

La commande liste les pods supprimés, puis rend la main au bout de quelques secondes, le temps que les applications s'arrêtent proprement :

```text
pod "vote-644d47956b-b7v6j" deleted from default namespace
pod "vote-644d47956b-kl4tf" deleted from default namespace
pod "vote-644d47956b-t8k5r" deleted from default namespace
```

Trois nouveaux pods sont apparus pendant que les anciens s'arrêtaient. Attendez qu'ils soient prêts :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Regardez ensuite ce qui tourne, maintenant que le Deployment a fini son travail :

```exec
kubectl get pods
```

Trois pods, trois nouveaux noms : l'état souhaité est de nouveau respecté.

```text
NAME                    READY   STATUS    RESTARTS   AGE
vote-644d47956b-cxgdh   1/1     Running   0          2s
vote-644d47956b-fgr8m   1/1     Running   0          2s
vote-644d47956b-wb8nx   1/1     Running   0          2s
```

```ack
Trois nouveaux pods ont remplacé les anciens
```

> 📖 [Les Deployments](https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/) · [Les labels et les sélecteurs](https://kubernetes.io/fr/docs/concepts/overview/working-with-objects/labels/) · [`kubectl port-forward`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)

## Questions sur les réplicas

```quiz
title: "De 1 à 3 exemplaires"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Le fichier demandait 1 réplica, vous écrivez 3 et vous appliquez. Que fait Kubernetes ?"
    answers:
      - "Il supprime le pod existant et en crée 3 nouveaux"
      - "Il compare 3 à ce qui tourne, constate qu'il manque 2 pods, et les crée"
      - "Il attend que vous lanciez vous-même les 2 pods manquants"
    correct: 2
    explanation: "C'est la boucle de réconciliation : vous donnez l'état souhaité, Kubernetes calcule l'écart et le corrige. Le pod qui tournait déjà n'est pas touché."
  - prompt: "Trois réplicas de `vote` tournent, et vous ouvrez l'application par `kubectl port-forward deployment/vote`. Qui répond ?"
    answers:
      - "Les trois pods, à tour de rôle"
      - "Un seul pod, choisi au démarrage du tunnel"
      - "Le Deployment lui-même"
    correct: 2
    explanation: "`kubectl port-forward` se branche sur un pod. Pour répartir les visiteurs entre les réplicas, il faut un Service."
  - prompt: "Que désigne `-l app=vote` dans `kubectl delete pods -l app=vote` ?"
    answers:
      - "Le pod qui s'appelle `app=vote`"
      - "Tous les pods qui portent l'étiquette `app: vote`, quel que soit leur nom"
      - "Le Deployment `vote` et tout ce qu'il contient"
    correct: 2
    explanation: "`-l` sélectionne par étiquette (*label*). Le Deployment, lui, n'est pas supprimé : c'est pour cela que les pods reviennent."
```
