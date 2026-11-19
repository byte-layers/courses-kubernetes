---
title: "Sortir l'application du namespace default"
description: "Remettre l'application de vote dans default, lister les namespaces du cluster, puis supprimer l'application de default sans toucher aux fichiers"
---

Depuis votre premier Deployment, `kubectl` vous répond souvent `… in default namespace`. Vous allez voir ce qu'est ce namespace, ce que le cluster en contient d'autres, puis en sortir l'application de vote.

## Remettre l'application de vote dans default

Cette section est un **support** : elle ne fait pas partie de ce que le cours enseigne. Elle met le cluster dans l'état dont la suite a besoin, que vous arriviez du cours précédent ou avec un cluster vide. Chaque geste est une commande à lancer, et vous pouvez la jouer sans vous arrêter sur chaque ligne.

La première commande remet `kubectl` sur le namespace `default`. Elle ne change rien si c'est déjà le cas.

```exec
kubectl config set-context --current --namespace=default
```

La deuxième supprime le namespace `vote-app`, que ce cours va créer. Si vous suivez ce cours pour la première fois, il n'existe pas et la commande ne fait rien. Si vous le refaites, elle peut prendre une trentaine de secondes.

```exec
kubectl delete namespace vote-app --ignore-not-found
```

Les fichiers de ce cours vont dans `~/byte-layers/kubernetes/l4/app/`. Créez ce dossier. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l4/app
```

Ce dossier reçoit les sept fichiers de l'application de vote, tels que les cours précédents les ont laissés.

Écrivez d'abord le fichier de `redis`, la file d'attente où `vote` dépose les votes : son Deployment et son Service. La commande crée le fichier `redis.yml` et y recopie tout ce qui suit, jusqu'à la ligne `EOF` :

```exec
cat > ~/byte-layers/kubernetes/l4/app/redis.yml <<'EOF'
# redis : la file d'attente où vote dépose les votes. Joignable dans le cluster sous le nom "redis".
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: ghcr.io/bngams/kube-redis:8-alpine
          ports:
            - containerPort: 6379
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
--- # second objet du fichier : le Service de redis
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
EOF
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'on lui donne en entrée |
> | `> ~/byte-layers/kubernetes/l4/app/redis.yml` | envoie cette copie dans le fichier, au lieu de l'afficher ; le fichier est créé, ou remplacé s'il existe |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles |
> | `EOF` (dernière ligne) | marque la fin de ces lignes ; elle n'est pas écrite dans le fichier |

Écrivez de la même façon le fichier de `db`, la base PostgreSQL qui garde les votes, avec son Service :

```exec
cat > ~/byte-layers/kubernetes/l4/app/db.yml <<'EOF'
# db : la base PostgreSQL qui garde les votes. Joignable dans le cluster sous le nom "db".
# Les identifiants sont écrits en clair : un raccourci assumé, à ne pas reproduire. Leur place est dans un Secret.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: ghcr.io/bngams/kube-postgres:15-alpine
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
          ports:
            - containerPort: 5432
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
--- # second objet du fichier : le Service de db
apiVersion: v1
kind: Service
metadata:
  name: db
  labels:
    app: db
spec:
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
EOF
```

Le fichier suivant est celui de `worker`, qui sort les votes de `redis` et les enregistre dans `db`. Personne ne l'appelle : il n'a pas de Service.

```exec
cat > ~/byte-layers/kubernetes/l4/app/worker.yml <<'EOF'
# worker : sort les votes de redis et les enregistre dans db. Personne ne l'appelle : pas de Service.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  labels:
    app: worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
        - name: worker
          image: ghcr.io/bngams/kube-worker:1.0
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF
```

Vient ensuite `result`, la page des résultats en direct, qui lit `db` :

```exec
cat > ~/byte-layers/kubernetes/l4/app/result.yml <<'EOF'
# result : la page des résultats en direct, qui lit db. Joignable dans le cluster sous le nom "result".
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result
  labels:
    app: result
spec:
  replicas: 1
  selector:
    matchLabels:
      app: result
  template:
    metadata:
      labels:
        app: result
    spec:
      containers:
        - name: result
          image: ghcr.io/bngams/kube-result:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
--- # second objet du fichier : le Service de result
apiVersion: v1
kind: Service
metadata:
  name: result
  labels:
    app: result
spec:
  selector:
    app: result
  ports:
    - port: 80
      targetPort: 80
EOF
```

Il reste les trois fichiers de `vote`, la page où l'on vote. Le premier est son Deployment, à 3 réplicas :

```exec
cat > ~/byte-layers/kubernetes/l4/app/vote.deploy.yml <<'EOF'
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

Le deuxième est son Service, l'adresse stable `vote` :

```exec
cat > ~/byte-layers/kubernetes/l4/app/vote.svc.yml <<'EOF'
# Une adresse stable, "vote", qui répartit les visiteurs entre tous les pods portant l'étiquette app: vote
apiVersion: v1
kind: Service
metadata:
  name: vote
  labels:
    app: vote
spec:
  selector:
    app: vote
  ports:
    - port: 80
      targetPort: 80
EOF
```

Le troisième est son Ingress, la porte d'entrée qui envoie les visites web vers le Service `vote` :

```exec
cat > ~/byte-layers/kubernetes/l4/app/vote.ingress.yml <<'EOF'
# La porte d'entrée du cluster : toutes les requêtes reçues sur "/" sont envoyées au Service vote
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vote
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vote
                port:
                  number: 80
EOF
```

Transmettez ces sept fichiers au cluster. Quand on donne un dossier à `kubectl apply -f`, il applique tous les fichiers qui s'y trouvent :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l4/app/
```

Attendez enfin que les cinq Deployments soient prêts. `kubectl wait` rend la main quand la condition demandée est vraie, ici `Available` (le Deployment a tous ses pods prêts) pour tous les Deployments du namespace (`--all`), et abandonne au bout de 120 secondes :

```exec
kubectl wait --for=condition=Available deployment --all --timeout=120s
```

Vous devez lire cinq lignes qui se terminent par `condition met`, une par Deployment.

```verify
if ! kubectl get namespace default >/dev/null 2>&1; then
  echo "kubectl ne répond pas. Vérifiez que votre cluster est démarré, puis réessayez."
  exit 1
fi
for d in redis db worker result vote; do
  if ! kubectl get deployment "$d" -n default >/dev/null 2>&1; then
    echo "Le Deployment $d n'est pas dans le namespace default. Relancez les commandes de cette section, dans l'ordre."
    exit 1
  fi
done
if ! kubectl wait --for=condition=Available deployment redis db worker result vote -n default --timeout=45s >/dev/null 2>&1; then
  echo "Les cinq Deployments existent, mais tous leurs pods ne sont pas prêts. Regardez kubectl get pods -n default, attendez un peu, puis réessayez."
  exit 1
fi
if ! kubectl get ingress vote -n default >/dev/null 2>&1; then
  echo "L'Ingress vote n'est pas dans default. Relancez kubectl apply -f ~/byte-layers/kubernetes/l4/app/"
  exit 1
fi
echo "L'application de vote tourne dans default : 5 Deployments prêts, et l'Ingress vote."
```

## Lister les namespaces du cluster

Un **namespace** est un espace de rangement dans le cluster, comme un quartier dans une ville. **`default` est le namespace par défaut** : tout ce qu'on crée sans préciser de namespace y atterrit. C'est là que vous travaillez depuis le début. Listez les namespaces de votre cluster :

```exec
kubectl get namespaces
```

Sur votre poste, vous lisez cinq namespaces. Vos âges seront différents.

```text
NAME              STATUS   AGE
default           Active   2d
ingress-nginx     Active   2d
kube-node-lease   Active   2d
kube-public       Active   2d
kube-system       Active   2d
```

`ingress-nginx` est le namespace du contrôleur d'Ingress ; sur un poste hébergé (un VS Code dans le navigateur, fourni par un formateur), il n'apparaît pas, car le contrôleur y est rangé dans `kube-system`. `kube-system` contient les composants de Kubernetes lui-même : n'y touchez jamais. Vous pouvez en revanche y regarder. L'option `-n`, suivie d'un nom, adresse une commande à un autre namespace que le vôtre :

```exec
kubectl get pods -n kube-system
```

Sur votre poste, vous retrouvez les pièces du cluster, qui tournent elles aussi dans des pods. Sur un poste hébergé, la liste est plus courte.

```text
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-59556d9b4c-lvbrt   1/1     Running   0          2d
calico-node-njv64                          1/1     Running   0          2d
coredns-66bc5c9577-g9jjn                   1/1     Running   0          2d
etcd-layers-k8s                            1/1     Running   0          2d
kube-apiserver-layers-k8s                  1/1     Running   0          2d
kube-controller-manager-layers-k8s         1/1     Running   0          2d
kube-proxy-pj58p                           1/1     Running   0          2d
kube-scheduler-layers-k8s                  1/1     Running   0          2d
storage-provisioner                        1/1     Running   0          2d
```

Aucun de ces pods n'apparaît dans votre `kubectl get pods` habituel, et les vôtres n'apparaissent pas ici : chaque namespace a sa propre liste.

En pratique, `default` est un fourre-tout. On range chaque application, chaque équipe ou chaque environnement dans **son propre namespace**, pour séparer les noms, les réglages, les droits et les ressources. C'est ce que vous allez faire pour l'application de vote, en commençant par la sortir de `default`.

## Supprimer l'application de default

`kubectl delete -f` supprime du cluster ce qu'un fichier décrit. Comme `kubectl apply -f`, il accepte un dossier. Avant de lancer la commande, faites votre prévision : vos fichiers seront-ils supprimés eux aussi ?

```exec
kubectl delete -f ~/byte-layers/kubernetes/l4/app/
```

`kubectl` annonce chaque objet supprimé, et le namespace d'où il part :

```text
deployment.apps "db" deleted from default namespace
service "db" deleted from default namespace
deployment.apps "redis" deleted from default namespace
service "redis" deleted from default namespace
deployment.apps "result" deleted from default namespace
service "result" deleted from default namespace
deployment.apps "vote" deleted from default namespace
ingress.networking.k8s.io "vote" deleted from default namespace
service "vote" deleted from default namespace
deployment.apps "worker" deleted from default namespace
```

Regardez maintenant le dossier :

```exec
ls ~/byte-layers/kubernetes/l4/app/
```

Les sept fichiers sont toujours là. `kubectl delete -f` supprime seulement les objets **dans le cluster** ; vos fichiers, eux, restent intacts. Il reste à voir ce que contient encore `default` :

```exec
kubectl get deployments
```

Le namespace ne contient plus aucun Deployment, donc plus aucun pod de l'application :

```text
No resources found in default namespace.
```

```verify
if ! kubectl get namespace default >/dev/null 2>&1; then
  echo "kubectl ne répond pas. Vérifiez que votre cluster est démarré, puis réessayez."
  exit 1
fi
if [ ! -f "$HOME/byte-layers/kubernetes/l4/app/vote.deploy.yml" ]; then
  echo "Le fichier ~/byte-layers/kubernetes/l4/app/vote.deploy.yml n'existe pas. Jouez d'abord la première section de cette étape."
  exit 1
fi
left=$(kubectl get deployments,services,ingress -n default -o name 2>/dev/null | grep -E '/(vote|redis|db|worker|result)$' | tr '\n' ' ')
if [ -n "$left" ]; then
  echo "Il reste des objets de l'application dans default : $left"
  echo "Lancez : kubectl delete -f ~/byte-layers/kubernetes/l4/app/ -n default"
  exit 1
fi
echo "Le namespace default ne contient plus l'application, et vos fichiers sont intacts."
```
