---
title: "Partir de l'application de vote en version 1.0"
description: "Étape support : se placer dans le namespace vote-app et y remettre l'application de vote dans son état de départ, avec vote en version 1.0"
---

Cette étape est un **support** : elle ne fait pas partie de ce que le cours enseigne. Elle met le cluster dans l'état dont la suite a besoin, que vous arriviez du cours précédent ou avec un cluster vide. Chaque geste est une commande à lancer, et vous pouvez la jouer sans vous arrêter sur chaque ligne : tout ce qu'elle utilise vient des cours précédents.

## Se placer dans le namespace vote-app

L'application de vote vit dans son propre namespace, `vote-app`. Créez-le :

```exec
kubectl create namespace vote-app
```

Si vous avez suivi [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas), ce namespace existe déjà, et la commande répond `AlreadyExists` : c'est sans conséquence, rien n'est modifié.

Demandez ensuite à `kubectl` d'utiliser ce namespace par défaut, pour ne pas avoir à écrire `-n vote-app` à chaque commande :

```exec
kubectl config set-context --current --namespace=vote-app
```

La réponse confirme le réglage :

```text
Context "layers-k8s" modified.
```

## Écrire les fichiers de départ

Les fichiers de ce cours vont dans `~/byte-layers/kubernetes/l6/`. Créez ce dossier. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l6
```

Écrivez d'abord le fichier que vous modifierez pendant tout le cours : `vote.deploy.yml`, le Deployment de `vote`, la page où l'on vote. Il demande 3 réplicas de l'image en version `1.0`, qui lisent leurs réglages dans la ConfigMap `vote-config`. La commande crée le fichier et y recopie tout ce qui suit, jusqu'à la ligne `EOF` :

```exec
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
          image: ghcr.io/bngams/kube-vote:1.0    # TODO : seule la version, après les deux-points, change
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
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'on lui donne en entrée |
> | `> ~/byte-layers/kubernetes/l6/vote.deploy.yml` | envoie cette copie dans le fichier, au lieu de l'afficher ; le fichier est créé, ou remplacé s'il existe |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles |
> | `EOF` (dernière ligne) | marque la fin de ces lignes ; elle n'est pas écrite dans le fichier |

Écrivez de la même façon `vote.config.yml`, la ConfigMap qui range les deux options du vote à part de l'image :

```exec
cat > ~/byte-layers/kubernetes/l6/vote.config.yml <<'EOF'
# Les réglages de vote, rangés à part : on les change sans toucher à l'image ni au Deployment
apiVersion: v1
kind: ConfigMap
metadata:
  name: vote-config
data:
  OPTION_A: Montagne
  OPTION_B: Mer
EOF
```

Le dernier fichier, `base.yml`, regroupe tout le reste de l'application, que ce cours ne modifie pas. Ce tableau dit ce qu'il contient :

| Objet | Rôle |
|---|---|
| Service `vote` | l'adresse stable `vote`, qui répartit les visiteurs entre les pods portant l'étiquette `app: vote` |
| Deployment et Service `redis` | la file d'attente où `vote` dépose les votes |
| Secret `db-credentials` | les identifiants de la base |
| Deployment et Service `db` | la base PostgreSQL qui garde les votes, et lit ses identifiants dans le Secret |
| Deployment `worker` | sort les votes de `redis` et les enregistre dans `db` |
| Deployment et Service `result` | la page des résultats en direct |

Écrivez-le de la même façon que les deux autres :

```exec
cat > ~/byte-layers/kubernetes/l6/base.yml <<'EOF'
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
--- # redis : la file d'attente où vote dépose les votes. Joignable dans le cluster sous le nom "redis".
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
--- # le Service qui lui donne une adresse stable
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
--- # Les identifiants de la base, rangés dans un Secret plutôt qu'en clair dans le Deployment
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
--- # db : la base PostgreSQL qui garde les votes. Ses identifiants viennent du Secret "db-credentials".
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
          envFrom:
            - secretRef:
                name: db-credentials
          ports:
            - containerPort: 5432
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
--- # le Service qui lui donne une adresse stable
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
--- # worker : sort les votes de redis et les enregistre dans db. Personne ne l'appelle : pas de Service.
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
--- # result : la page des résultats en direct, qui lit db. Joignable dans le cluster sous le nom "result".
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
--- # le Service qui lui donne une adresse stable
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

## Appliquer les fichiers et attendre que vote soit prêt

Appliquez les trois fichiers d'un coup : `-f` désigne ici le dossier, et `kubectl` applique tous les fichiers qu'il contient. Si l'application tourne déjà dans cet état, `kubectl` répond `unchanged` pour chaque objet ; sinon il les crée, ou remet `vote` en version 1.0. Si vous aviez choisi d'autres options de vote dans un cours précédent, elles redeviennent `Montagne` et `Mer`, et `kubectl` répond `configured` pour la ConfigMap.

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/
```

Attendez que les 3 pods de `vote` soient prêts. `kubectl rollout status` suit le Deployment et rend la main quand il est à jour, ou échoue au bout de 2 minutes. Au premier lancement, les images se téléchargent : la commande peut prendre une minute.

```exec
kubectl rollout status deployment/vote --timeout=120s
```

La dernière ligne de la réponse dit que tout est en place :

```text
deployment "vote" successfully rolled out
```

Regardez enfin le Deployment :

```exec
kubectl get deployments vote
```

Vous devez lire `3/3` dans la colonne `READY` :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   3/3     3            3           1s
```

```ack
vote tourne en version 1.0, avec 3 réplicas, dans vote-app
```
