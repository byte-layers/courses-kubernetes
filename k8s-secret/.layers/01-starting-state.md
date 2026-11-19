---
title: "Partir de l'application de vote"
description: "Étape support : se placer dans le namespace vote-app et y remettre l'application de vote dans son état de départ"
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

La réponse confirme que le réglage est enregistré :

```text
Context "layers-k8s" modified.
```

## Écrire les fichiers de départ

Les fichiers de ce cours vont dans `~/byte-layers/kubernetes/l5/`. Créez ce dossier, avec un sous-dossier `base` pour les fichiers de départ, que vous ne modifierez pas. `mkdir -p` crée aussi les dossiers parents qui manquent :

```exec
mkdir -p ~/byte-layers/kubernetes/l5/base
```

Écrivez d'abord le fichier de `vote`, la page où l'on vote : son Deployment à 3 réplicas et son Service. La commande crée le fichier `vote.yml` et y recopie tout ce qui suit, jusqu'à la ligne `EOF`. Dans ce fichier, la ligne qui commence par `---` sépare deux objets :

```exec
cat > ~/byte-layers/kubernetes/l5/base/vote.yml <<'EOF'
# vote : la page où l'on vote. 3 réplicas, chacun réservant un dixième de processeur et 64 Mi de mémoire
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
--- # objet suivant
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

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'on lui donne en entrée |
> | `> ~/byte-layers/kubernetes/l5/base/vote.yml` | envoie cette copie dans le fichier, au lieu de l'afficher ; le fichier est créé, ou remplacé s'il existe |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles |
> | `EOF` (dernière ligne) | marque la fin de ces lignes ; elle n'est pas écrite dans le fichier |

Écrivez de la même façon le fichier de `db`, la base PostgreSQL qui garde les votes. Ses identifiants y sont écrits en clair : c'est son état de départ.

```exec
cat > ~/byte-layers/kubernetes/l5/base/db.yml <<'EOF'
# db : la base PostgreSQL qui garde les votes. Joignable dans le cluster sous le nom "db".
# Les identifiants sont écrits en clair : c'est volontaire pour l'instant, leur place est dans un Secret.
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
--- # objet suivant
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

Le dernier fichier regroupe les trois autres morceaux : `redis`, la file d'attente où `vote` dépose les votes, `worker`, qui les enregistre dans la base, et `result`, la page des résultats.

```exec
cat > ~/byte-layers/kubernetes/l5/base/redis-worker-result.yml <<'EOF'
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
--- # objet suivant
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
--- # objet suivant
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
--- # objet suivant
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
--- # objet suivant
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

## Appliquer les fichiers et attendre que tout tourne

Appliquez `db`, puis les trois autres morceaux. Si l'application tourne déjà dans cet état, `kubectl` répond `unchanged` pour chaque objet ; sinon il les crée, ou remet `db` dans son état de départ. Le quota, les règles réseau et l'Ingress que vous avez peut-être posés dans `vote-app` ne sont pas concernés : ces fichiers ne décrivent que des Deployments et des Services.

```exec
kubectl apply -f ~/byte-layers/kubernetes/l5/base/db.yml -f ~/byte-layers/kubernetes/l5/base/redis-worker-result.yml
```

La page de vote, elle, ne doit pas être remise à zéro si elle tourne déjà : ce cours ne la modifie pas. La commande suivante ne l'applique donc que si elle manque. `||` veut dire « sinon » : `kubectl` cherche le Deployment `vote`, et c'est seulement si cette recherche échoue que la seconde commande s'exécute.

```exec
kubectl get deployment vote || kubectl apply -f ~/byte-layers/kubernetes/l5/base/vote.yml
```

Si `vote` existe, la commande affiche sa ligne et s'arrête là. Sinon, elle affiche une erreur `NotFound`, puis `deployment.apps/vote created` et `service/vote created`.

Si vous refaites ce cours, il reste le Secret `db-credentials` d'un passage précédent. Supprimez-le pour repartir de l'état de départ. `--ignore-not-found` évite une erreur quand il n'existe pas, et la commande ne répond alors rien :

```exec
kubectl delete secret db-credentials --ignore-not-found
```

Attendez que les cinq Deployments soient disponibles. `kubectl wait` rend la main dès que la condition `Available` est vraie pour chacun, ou échoue au bout de 3 minutes. Au premier lancement, les images se téléchargent : la commande peut prendre une ou deux minutes.

```exec
kubectl wait --for=condition=Available deployment vote redis db worker result --timeout=180s
```

Chaque Deployment répond `condition met`. Regardez enfin l'ensemble :

```exec
kubectl get deployments
```

Vous devez retrouver les cinq morceaux, tous complets dans la colonne `READY` :

```text
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
db       1/1     1            1           2m
redis    1/1     1            1           2m
result   1/1     1            1           2m
vote     3/3     3            3           2m
worker   1/1     1            1           2m
```

```ack
Les cinq morceaux de l'application tournent dans vote-app
```
