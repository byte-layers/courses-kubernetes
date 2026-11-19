---
title: "La v3 plante : diagnostiquer avec describe et logs"
description: "Déployer une version qui plante au démarrage, reconnaître CrashLoopBackOff, puis trouver la cause avec kubectl describe et kubectl logs"
---

L'équipe livre maintenant la v3, qui ajoute un titre personnalisable. Vous la déployez comme la v2, et cette fois la mise à jour ne se passe pas bien. Vous allez apprendre à lire ce que Kubernetes montre, puis à trouver la cause.

## Déployer la v3

Dans `~/byte-layers/kubernetes/l6/vote.deploy.yml`, passez l'image en version `3.0` : c'est la même ligne qu'à l'étape précédente, et seuls les chiffres de la fin changent. Enregistrez, puis appliquez :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Suivez la mise à jour comme pour la v2, en lui laissant cette fois 45 secondes. Faites une prévision : la commande va-t-elle se terminer par `successfully rolled out` ?

```exec
kubectl rollout status deployment/vote --timeout=45s
```

La mise à jour reste bloquée en chemin, et la commande abandonne au bout du délai. Certaines lignes se répètent ; en voici l'essentiel :

```text
Waiting for deployment "vote" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "vote" rollout to finish: 2 out of 3 new replicas have been updated...
error: timed out waiting for the condition
```

La ligne `2 out of 3` n'apparaît pas à tous les coups : la section suivante dit pourquoi.

## Lire l'état des pods : Error et CrashLoopBackOff

Regardez où en sont les pods de `vote` :

```exec
kubectl get pods -l app=vote
```

Les pods de la v3 sont ceux dont la colonne `RESTARTS` n'est pas à 0, et elle augmente : Kubernetes les relance, et ils replantent. Il y en a deux dans cet exemple ; chez vous, il peut n'y en avoir qu'un, à côté de trois pods de la v2.

```text
NAME                    READY   STATUS             RESTARTS      AGE
vote-566d594846-g7c4x   1/1     Running            0             65s
vote-566d594846-wjcrl   1/1     Running            0             66s
vote-56f645655-4nvvh    0/1     CrashLoopBackOff   2 (27s ago)   45s
vote-56f645655-d9jz2    0/1     Error              2 (26s ago)   27s
```

| Statut | Signification |
|---|---|
| `Error` | le conteneur vient de s'arrêter en erreur |
| `CrashLoopBackOff` | le conteneur plante en boucle. Kubernetes le relance, mais en **attendant de plus en plus longtemps** entre deux essais (10 s, 20 s, 40 s… jusqu'à 5 minutes), pour ne pas épuiser le nœud |

Regardez maintenant le Deployment :

```exec
kubectl get deployments vote
```

Dans cet exemple, la colonne `READY` ne dit plus `3/3` :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   2/3     2            2           71s
```

L'application répond encore, grâce aux pods de la v2 qui restent. Mais ici, il n'en reste que **2 sur 3** : un pod sain a été retiré, alors que ses remplaçants plantent. Ce retrait est une affaire de chance : lors de nos essais, il était arrivé au bout de 45 secondes environ une fois sur deux, et il peut encore arriver plus tard, à chaque redémarrage d'un pod de la v3. Si vous lisez `3/3`, vos trois pods de la v2 sont toujours là, pour l'instant. Ce constat a une explication et un remède, les sondes de santé, qui sont le sujet d'un prochain cours. Ici, on s'occupe de la panne.

```hint
C'est la même ligne `image:` qu'à l'étape précédente : remplacez `2.0` par `3.0`, enregistrez, puis lancez `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml`.
```

```hint
Une fois le fichier appliqué, il faut une vingtaine de secondes pour qu'un pod de la v3 ait planté au moins une fois : `kubectl get pods -l app=vote` montre sa colonne `RESTARTS`.
```

```verify
IMG=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "Le Deployment vote est introuvable dans le namespace courant. Rejouez l'étape « Partir de l'application de vote en version 1.0 »."
  exit 1
fi
case "$IMG" in
  *kube-vote:3.0) : ;;
  *) echo "Le cluster demande l'image $IMG, pas la 3.0. Dans vote.deploy.yml, passez la version à 3.0, enregistrez, puis appliquez le fichier."
     exit 1 ;;
esac
for i in 1 2 3 4 5 6 7 8; do
  R=$(kubectl get pods -l app=vote -o jsonpath='{range .items[*]}{.spec.containers[0].image}{" "}{.status.containerStatuses[0].restartCount}{"\n"}{end}' 2>/dev/null | grep 'kube-vote:3.0 [1-9]')
  [ -n "$R" ] && break
  sleep 5
done
if [ -z "$R" ]; then
  echo "La v3 est demandée, mais aucun de ses pods n'a encore planté. Attendez quelques secondes, regardez kubectl get pods -l app=vote, puis réessayez."
  exit 1
fi
echo "La v3 est déployée, et ses pods plantent en boucle : il reste à trouver pourquoi."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l6/vote.deploy.yml <<'EOF'
# vote : 3 réplicas, réglages lus dans la ConfigMap vote-config
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
          image: ghcr.io/bngams/kube-vote:3.0
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: vote-config
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```
````

## Décrire le pod en échec avec kubectl describe

Face à un pod en échec, on suit toujours la même démarche : **décrire**, puis **lire les logs**. Les deux commandes demandent le nom du pod. À la main, vous le copieriez depuis la liste des pods. Ici, pour que la commande marche chez vous sans rien recopier, le nom est remplacé par une formule entre `$(` et `)`, qui va chercher le pod de `vote` **le plus récent**, donc un pod de la v3 :

```exec
kubectl describe $(kubectl get pods -l app=vote --sort-by=.metadata.creationTimestamp -o name | tail -n 1)
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl describe` | affiche le détail d'un objet, avec ses événements |
> | `$( … )` | exécute d'abord la commande entre parenthèses, et met sa réponse à cet endroit de la ligne |
> | `kubectl get pods -l app=vote` | liste les pods de `vote` |
> | `--sort-by=.metadata.creationTimestamp` | les trie par date de création, le plus récent en dernier |
> | `-o name` | n'affiche que leur nom, sous la forme `pod/vote-…` |
> | `\| tail -n 1` | le `\|` passe cette liste à `tail`, qui n'en garde que la dernière ligne |

Parmi les nombreuses lignes de la réponse, deux passages sont utiles. Le premier est l'état du conteneur, vers le milieu :

```text
    State:          Terminated
      Reason:       Error
      Exit Code:    1
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
    Ready:          False
    Restart Count:  2
```

Selon l'instant, la première ligne dit `Terminated`, comme ici, ou `Waiting` avec la raison `CrashLoopBackOff`, quand Kubernetes attend avant de relancer le conteneur. Le second passage est la liste des événements, tout en bas, dont voici la dernière ligne :

```text
  Warning  BackOff    11s (x3 over 25s)  kubelet            Back-off restarting failed container vote in pod vote-56f645655-d9jz2_vote-app(…)
```

`describe` confirme que **le conteneur s'arrête en erreur** (code de sortie `1`), mais pas pourquoi.

## Trouver la cause dans les logs

La raison, c'est l'application qui la donne, dans ses **logs**, son journal. La même formule désigne le même pod :

```exec
kubectl logs $(kubectl get pods -l app=vote --sort-by=.metadata.creationTimestamp -o name | tail -n 1)
```

La réponse fait une centaine de lignes, et sa **fin** n'est pas très parlante :

```text
    raise HaltServer(reason, self.WORKER_BOOT_ERROR)
gunicorn.errors.HaltServer: <HaltServer 'Worker failed to boot.' 3>
```

La vraie cause se trouve **plus haut**. Remontez dans le terminal, ou ne gardez que les lignes qui contiennent le mot `KeyError` (une erreur fréquente en Python : « clé introuvable »). Comme dans la formule, le `|` passe ce qui précède à la commande qui suit : ici les logs à `grep`, qui n'affiche que les lignes contenant ce mot :

```exec
kubectl logs $(kubectl get pods -l app=vote --sort-by=.metadata.creationTimestamp -o name | tail -n 1) | grep KeyError
```

Le terminal affiche la même ligne, une fois par processus de l'application :

```text
KeyError: 'VOTE_TITLE'
KeyError: 'VOTE_TITLE'
```

Si une de ces commandes ne répond rien, le conteneur vient juste de redémarrer et n'a encore rien écrit : relancez-la quelques secondes plus tard. `kubectl logs` accepte aussi l'option `--previous`, placée juste après `logs`, qui affiche les logs du conteneur **précédent**, celui d'avant le dernier redémarrage.

Voilà le coupable : la v3 attend un réglage `VOTE_TITLE` que personne ne lui a donné. C'est le titre personnalisable annoncé par l'équipe, mais elle a oublié de le dire à l'exploitation.

> 🗣️ **En réunion projet.** Vous savez maintenant ce que signifie « *le pod est en CrashLoopBackOff* » : l'application plante au démarrage, Kubernetes la relance en boucle. Ce n'est **pas** un problème de Kubernetes, mais de l'application ou de sa configuration. Et la première question à poser est : « *qu'est-ce que disent les logs ?* »

> 📖 [Déboguer un pod](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/) · [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/) · [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)

## Questions sur le diagnostic

```quiz
title: "Diagnostiquer un pod qui plante"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Un pod affiche `CrashLoopBackOff`. Qu'est-ce que cela veut dire ?"
    answers:
      - "Le nœud manque de place pour ce pod"
      - "L'image n'a pas pu être téléchargée"
      - "Le conteneur plante en boucle, et Kubernetes le relance en espaçant ses essais"
    correct: 3
    explanation: "L'application démarre, s'arrête en erreur, et Kubernetes la relance en attendant de plus en plus longtemps entre deux essais."
  - prompt: "`kubectl describe` indique `Exit Code: 1` pour le conteneur. Où chercher la raison de l'erreur ?"
    answers:
      - "Dans `kubectl logs`, le journal de l'application"
      - "Dans `kubectl get deployments`"
      - "Dans le fichier `vote.deploy.yml`"
    correct: 1
    explanation: "`describe` dit que le conteneur s'arrête en erreur. Le pourquoi, c'est l'application qui l'écrit dans ses logs."
  - prompt: "`kubectl logs` ne renvoie rien, parce que le conteneur vient de redémarrer. Quelle option affiche les logs du conteneur qui vient de planter ?"
    answers:
      - "`--all`"
      - "`--previous`"
      - "`--watch`"
    correct: 2
    explanation: "`--previous` affiche le journal du conteneur précédent, celui d'avant le dernier redémarrage."
```
