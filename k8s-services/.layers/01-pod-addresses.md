---
title: "Des adresses de pods qui changent"
description: "Remettre vote en place, lire l'adresse IP de chaque pod, en supprimer un et comparer"
---

Avant de créer quoi que ce soit, vous regardez le problème que le Service vient régler. Vous partez de l'application `vote` avec ses 3 réplicas, vous notez l'adresse de chaque pod, puis vous en supprimez un.

## Préparer le dossier et remettre vote en place

Cette section est un **support** : elle met le cluster dans l'état de départ, et vous pouvez la jouer sans vous y arrêter. Créez d'abord le dossier de travail. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l3
```

Écrivez ensuite le fichier du Deployment `vote`, tel qu'il est à la fin du cours [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending) : 3 réplicas, chacun réservant un dixième de processeur et 64 Mi de mémoire.

```exec
cat > ~/byte-layers/kubernetes/l3/vote.deploy.yml <<'EOF'
# vote : 3 réplicas, chacun réservant un dixième de processeur et 64 Mi de mémoire
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
              cpu: 100m
              memory: 64Mi
EOF
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'il reçoit en entrée |
> | `> ~/byte-layers/kubernetes/l3/vote.deploy.yml` | envoie cette copie dans le fichier, qu'il crée, ou dont il remplace le contenu s'il existe déjà |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles, jusqu'à la ligne `EOF` |
> | les lignes entre les deux | le contenu du fichier |
> | `EOF` | marque la fin du contenu ; ce mot n'est pas écrit dans le fichier |

Transmettez ce fichier au cluster. Si `vote` tourne déjà avec ce réglage, `kubectl apply` répond `unchanged` et ne touche à rien :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l3/vote.deploy.yml
```

Attendez que les 3 pods soient prêts. `kubectl rollout status` suit le déploiement et rend la main quand il est terminé, ou au bout de 120 secondes :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

La dernière ligne doit être celle-ci :

```text
deployment "vote" successfully rolled out
```

```ack
vote tourne avec ses 3 réplicas
```

## Lire l'adresse IP de chaque pod

Chaque pod reçoit une **adresse IP** dans le réseau interne du cluster, comme chaque ordinateur sur un réseau. Affichez-les avec l'option `-o wide`, qui ajoute des colonnes à la liste :

```exec
kubectl get pods -o wide
```

Le terminal affiche, parmi d'autres colonnes, l'adresse de chaque pod dans la colonne `IP`. Vos noms et vos adresses seront différents :

```text
NAME                  READY   STATUS    RESTARTS   AGE   IP              NODE
vote-fc57495c-5m552   1/1     Running   0          2m    10.244.218.91   layers-k8s
vote-fc57495c-pg7gv   1/1     Running   0          2m    10.244.218.90   layers-k8s
vote-fc57495c-r6x2b   1/1     Running   0          2m    10.244.218.89   layers-k8s
```

Sur un poste hébergé (un VS Code dans le navigateur, fourni par un formateur), le nœud porte un nom qui commence par `k3d-` et les adresses commencent par `10.42.`.

Gardez cette liste dans un fichier, pour la comparer à ce que vous obtiendrez après la suppression. `-l app=vote` ne retient que les pods qui portent l'étiquette `app: vote`, et `>` envoie la réponse dans le fichier au lieu de l'afficher :

```exec
kubectl get pods -l app=vote -o wide > ~/byte-layers/kubernetes/l3/pods-before.txt
```

`cat` affiche ce fichier, pour vérifier qu'il contient bien vos 3 pods :

```exec
cat ~/byte-layers/kubernetes/l3/pods-before.txt
```

```ack
J'ai les 3 adresses dans pods-before.txt
```

## Supprimer un pod et comparer les adresses

Vous savez que le Deployment remplace un pod supprimé. Faites une prévision avant de continuer : le pod remplaçant aura-t-il la même adresse que celui qu'il remplace ?

Supprimez le premier pod de **votre** liste. Cette commande n'a pas de bouton, car le nom est différent chez vous : tapez-la dans votre terminal, en remplaçant le nom ci-dessous par celui affiché chez vous (seule la fin change).

```bash
kubectl delete pod vote-fc57495c-5m552
```

Regardez de nouveau les adresses :

```exec
kubectl get pods -o wide
```

Deux pods n'ont pas bougé, et un nouveau est arrivé, reconnaissable à son âge. Si sa colonne `IP` affiche encore `<none>`, relancez la commande quelques secondes plus tard.

```text
NAME                  READY   STATUS    RESTARTS   AGE   IP              NODE
vote-fc57495c-hjvvd   1/1     Running   0          4s    10.244.218.93   layers-k8s
vote-fc57495c-pg7gv   1/1     Running   0          2m    10.244.218.90   layers-k8s
vote-fc57495c-r6x2b   1/1     Running   0          2m    10.244.218.89   layers-k8s
```

Le pod remplaçant a une **nouvelle adresse** (`10.244.218.93` ici). Comparez avec le fichier `pods-before.txt` : l'adresse du pod supprimé n'existe plus. Un autre morceau de l'application qui aurait noté cette adresse pour joindre `vote` serait maintenant perdu. Et même avec une adresse fixe, lequel des 3 pods faudrait-il appeler ?

> 🧠 **Les pods sont éphémères.** Ils naissent, meurent et sont remplacés en permanence : c'est le fonctionnement normal de Kubernetes. On ne s'adresse donc **jamais** directement à un pod. Il faut un intermédiaire stable, qui sait à tout moment quels pods sont vivants.

```hint
`kubectl get pods` donne le nom complet de chaque pod. La suppression s'écrit `kubectl delete pod` suivi d'un de ces noms.
```

```verify
F="$HOME/byte-layers/kubernetes/l3/pods-before.txt"
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote n'existe pas. Reprenez la section « Préparer le dossier et remettre vote en place »."
  exit 1
fi
if ! grep -q '^vote-' "$F" 2>/dev/null; then
  echo "Le fichier ~/byte-layers/kubernetes/l3/pods-before.txt ne contient pas vos pods. Relancez la commande qui l'écrit, dans la section précédente."
  exit 1
fi
NOW=$(kubectl get pods -l app=vote -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')
gone=0
for p in $(grep '^vote-' "$F" | cut -d' ' -f1); do
  echo "$NOW" | grep -qx "$p" || gone=$((gone + 1))
done
if [ "$gone" -lt 1 ]; then
  echo "Les pods notés dans pods-before.txt existent tous encore : supprimez-en un avec kubectl delete pod suivi de son nom."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "vote n'a pas encore retrouvé ses 3 pods prêts. Regardez kubectl get pods, puis réessayez."
  exit 1
fi
echo "Un pod a été remplacé, et son remplaçant a une nouvelle adresse."
```

````solution
Cette commande supprime le premier pod de `vote` sans que vous ayez à recopier son nom : `kubectl get pods -o name` donne les noms, `head -1` garde le premier, et `$(…)` le place dans la commande.

```bash
kubectl delete $(kubectl get pods -l app=vote -o name | head -1)
kubectl rollout status deployment/vote --timeout=60s
```
````

> 📖 [Les pods, documentation officielle](https://kubernetes.io/fr/docs/concepts/workloads/pods/)
