---
title: "Ajouter une sonde de readiness et une sonde de liveness"
description: "Écrire les deux sondes sur /healthz dans le Deployment, appliquer, et les retrouver dans la description du Deployment"
---

Pour que Kubernetes ne croie plus un pod prêt sur sa seule bonne mine, on lui donne un moyen de vérifier lui-même la santé de l'application : des **sondes** (*probes*). L'application de vote a justement une page prévue pour cela, `/healthz`, qui répond `ok` quand tout va bien.

## Readiness et liveness : ce que chaque sonde demande

Une sonde est une question que Kubernetes pose régulièrement à chaque conteneur. Les deux plus courantes ne posent pas la même, et n'ont pas la même conséquence :

| Sonde | Question posée régulièrement | Si la réponse est non |
|---|---|---|
| `readinessProbe` (**prêt ?**) | « peux-tu recevoir des visiteurs ? » | le pod est **retiré du Service** (il ne reçoit plus de visiteurs) et, pendant une mise à jour, il n'est **pas compté comme prêt** : la mise à jour attend |
| `livenessProbe` (**vivant ?**) | « es-tu encore en vie ? » | Kubernetes **redémarre** le conteneur (utile si l'application est bloquée sans avoir planté) |

Ici, les deux sondes posent leur question de la même façon : une requête HTTP sur une page de l'application. Une réponse normale veut dire oui ; une erreur, ou pas de réponse, veut dire non.

## Écrire les deux sondes dans le fichier

Les sondes s'écrivent dans le conteneur, au même niveau que `image` et `resources`. La commande qui suit réécrit `vote.deploy.yml` avec les deux blocs en place, et deux blancs `# TODO` à compléter. Elle remet aussi l'image en `2.0`, la version qui tourne réellement depuis le retour en arrière : le fichier et le cluster sont de nouveau d'accord.

```exec
cat > ~/byte-layers/kubernetes/l6/vote.deploy.yml <<'EOF'
# vote : 3 réplicas, réglages lus dans la ConfigMap vote-config, sondes de santé
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
          image: ghcr.io/bngams/kube-vote:2.0
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: vote-config
          readinessProbe:          # "es-tu prêt ?"
            httpGet:
              path: # TODO : la page de santé de l'application
              port: 80
            periodSeconds: 5       # toutes les 5 secondes
          livenessProbe:           # "es-tu vivant ?"
            httpGet:
              path: # TODO : la même page
              port: 80
            periodSeconds: 10      # toutes les 10 secondes
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
```

Ce tableau dit ce que veut dire chaque ligne d'une sonde :

| Élément | Rôle |
|---|---|
| `httpGet` | la sonde est une requête HTTP, comme celle d'un navigateur |
| `path` | la page demandée |
| `port` | le port du conteneur sur lequel l'application écoute |
| `periodSeconds` | le temps entre deux questions, en secondes |

Ouvrez `~/byte-layers/kubernetes/l6/vote.deploy.yml` dans votre éditeur. Sur chacune des deux lignes `path:`, remplacez le commentaire `# TODO …` par la page de santé de l'application, puis enregistrez.

## Appliquer et retrouver les sondes dans le cluster

Appliquez le fichier. Le modèle de pod a changé, puisqu'il a maintenant des sondes : Kubernetes remplace donc les trois pods, un par un.

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Attendez la fin de cette mise à jour :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Cette fois, Kubernetes n'a compté chaque nouveau pod comme prêt qu'après une bonne réponse de sa sonde de readiness. Demandez la description du Deployment :

```exec
kubectl describe deployment vote
```

Dans la réponse, sous `Pod Template`, repérez ces deux lignes :

```text
    Liveness:   http-get http://:80/healthz delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:  http-get http://:80/healthz delay=0s timeout=1s period=5s #success=1 #failure=3
```

`#failure=3` : il faut **3 échecs de suite** pour que Kubernetes réagisse, ce qui évite de redémarrer un conteneur pour un simple ralentissement passager. C'est une valeur par défaut, que vous n'avez pas eu à écrire, comme `delay`, `timeout` et `#success`.

```hint
La page de santé de l'application est celle que cite le début de cette étape. Un chemin commence toujours par `/`.
```

```hint
Les deux lignes attendues sont identiques : `path: /healthz`. Enregistrez, puis lancez `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml`.
```

```verify
D=$(kubectl get deployment vote -o name 2>/dev/null)
if [ -z "$D" ]; then
  echo "Le Deployment vote est introuvable dans le namespace courant. Rejouez l'étape « Partir de l'application de vote en version 2.0 »."
  exit 1
fi
for P in readinessProbe livenessProbe; do
  PATHV=$(kubectl get deployment vote -o jsonpath="{.spec.template.spec.containers[0].$P.httpGet.path}" 2>/dev/null)
  PORTV=$(kubectl get deployment vote -o jsonpath="{.spec.template.spec.containers[0].$P.httpGet.port}" 2>/dev/null)
  if [ -z "$PORTV" ]; then
    echo "Le Deployment vote n'a pas de $P avec un bloc httpGet. Écrivez le fichier avec la commande de cette étape, complétez-le, puis appliquez-le."
    exit 1
  fi
  if [ "$PATHV" != "/healthz" ]; then
    echo "La $P interroge la page « ${PATHV:-/} », pas /healthz. Complétez la ligne path: de ce bloc, enregistrez, puis appliquez le fichier."
    exit 1
  fi
  if [ "$PORTV" != "80" ]; then
    echo "La $P interroge le port $PORTV, alors que l'application écoute sur le port 80."
    exit 1
  fi
done
if ! kubectl rollout status deployment/vote --timeout=45s >/dev/null 2>&1; then
  echo "Les sondes sont écrites, mais la mise à jour n'est pas terminée. Regardez kubectl get pods -l app=vote, puis réessayez."
  exit 1
fi
echo "Les deux sondes interrogent /healthz sur le port 80, et les 3 pods de vote y répondent."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l6/vote.deploy.yml <<'EOF'
# vote : 3 réplicas, réglages lus dans la ConfigMap vote-config, sondes de santé
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
          image: ghcr.io/bngams/kube-vote:2.0
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: vote-config
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

> 📖 [Les sondes de liveness, readiness et startup](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) · [Configurer les sondes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
