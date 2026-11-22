---
title: "Partir d'un réseau ouvert"
description: "Étape de départ : remettre l'application de vote dans vote-app, vérifier que le cluster sait appliquer des règles réseau, puis constater qu'un pod étranger joint la base de données"
---

Avant de fermer quoi que ce soit, il faut voir ce qui est ouvert. Cette étape remet l'application de vote en place, puis vous fait tester le réseau depuis un pod qui n'a rien à voir avec elle.

## Remettre l'application de vote dans vote-app

Cette section est un **support** : elle ne fait pas partie de ce que le cours enseigne. Elle met le cluster dans l'état dont la suite a besoin, que vous arriviez du cours précédent ou avec un cluster vide. Chaque geste est une commande à lancer, et vous pouvez la jouer sans vous arrêter sur chaque ligne : tout ce qu'elle utilise vient de [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas).

L'application de vote vit dans son propre namespace, `vote-app`. Créez-le :

```exec
kubectl create namespace vote-app
```

Si ce namespace existe déjà, la commande répond `AlreadyExists` : c'est sans conséquence, rien n'est modifié. Demandez ensuite à `kubectl` de travailler dans ce namespace, pour ne pas avoir à écrire `-n vote-app` à chaque commande :

```exec
kubectl config set-context --current --namespace=vote-app
```

Les fichiers de l'application vont dans `~/byte-layers/kubernetes/l4/app/`. Créez ce dossier. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l4/app
```

Ce dossier reçoit les sept fichiers de l'application de vote. Si vous les avez déjà, ces commandes les réécrivent à l'identique.

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

Posez le quota du namespace, 12 pods au plus. S'il existe déjà, la commande répond `AlreadyExists`, sans conséquence :

```exec
kubectl create quota pods-max --hard=pods=12
```

Supprimez les règles réseau du namespace, au cas où vous referiez ce cours : il doit partir d'un réseau sans aucune règle. S'il n'y en a pas, la commande répond `No resources found` :

```exec
kubectl delete networkpolicy --all
```

Attendez enfin que les cinq Deployments soient prêts. `kubectl wait` rend la main quand tous les Deployments du namespace (`--all`) ont leurs pods prêts (`Available`), et abandonne au bout de 120 secondes :

```exec
kubectl wait --for=condition=Available deployment --all --timeout=120s
```

Vous devez lire cinq lignes qui se terminent par `condition met`, une par Deployment.

## Vérifier que le cluster sait appliquer des règles réseau

Une règle réseau n'est qu'un objet rangé dans le cluster. Pour qu'elle ait un effet, il faut que le réseau du cluster sache l'appliquer : c'est le rôle du composant réseau, le **CNI** (*Container Network Interface*). Sur votre poste, c'est **Calico**, que minikube n'installe que si on le lui demande (l'option `--cni=calico` du démarrage). Vérifiez qu'il est là. `-l` ne garde que les pods qui portent l'étiquette donnée :

```exec
kubectl get pods -n kube-system -l k8s-app=calico-node
```

Sur votre poste, vous devez lire un pod `calico-node-…` à l'état `Running` :

```text
NAME                READY   STATUS    RESTARTS   AGE
calico-node-njv64   1/1     Running   0          2d
```

Le tableau dit quoi faire selon ce que vous lisez :

| Ce que vous lisez | Ce que cela veut dire |
|---|---|
| un pod `calico-node-…` `Running` | votre cluster est prêt |
| `No resources found`, sur votre poste | le cluster a été démarré sans Calico : les règles seraient acceptées, mais **sans aucun effet, et sans message d'erreur**. Recréez le cluster comme dans le lab [Démarrer son cluster Kubernetes avec minikube](https://byte-layers.com/fr/labs/k8s-minikube-install), puis reprenez cette étape |
| `No resources found`, sur un poste hébergé (un VS Code dans le navigateur, fourni par un formateur) | c'est normal : le cluster y applique les règles réseau avec son propre composant, sans Calico |

## Tester le réseau depuis un pod étranger à l'application

Les Services rendent les morceaux de l'application joignables… par **tout le monde**. Lancez un pod de test, qui n'a rien à voir avec l'application, et demandez-lui s'il peut ouvrir une connexion vers la base de données et vers `redis`. Avant de valider, faites votre prévision : que va-t-il répondre pour chacun des deux ?

```exec
kubectl run test --rm -i --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- sh -c "nc -z -w 3 db 5432 && echo db: ouvert || echo db: bloqué; nc -z -w 3 redis 6379 && echo redis: ouvert || echo redis: bloqué"
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl run test` | lance un pod nommé `test`, dans le namespace courant |
> | `--rm -i --restart=Never` | supprime le pod quand il a fini (`--rm`), affiche ce qu'il écrit (`-i`), et ne le relance pas (`--restart=Never`) |
> | `--image=…/kube-busybox:1.37` | une petite image qui contient des outils réseau |
> | `-- sh -c "…"` | la commande que le pod exécute : tout ce qui est entre guillemets |
> | `nc -z -w 3 db 5432` | `nc` (*netcat*) teste seulement si une connexion vers le Service `db`, port 5432, est possible (`-z`), sans rien envoyer, et abandonne au bout de 3 secondes (`-w 3`) |
> | `&& echo db: ouvert` | si la connexion réussit, affiche `db: ouvert` |
> | `\|\| echo db: bloqué` | sinon, affiche `db: bloqué` |
> | `;` | sépare le test de `db` de celui de `redis`, écrit de la même façon |

Le terminal affiche la réponse du pod. Avant la réponse, `kubectl` affiche parfois deux lignes en anglais (`All commands and output from this session…`, puis `If you don't see a command prompt, try pressing enter.`) : elles sont sans importance.

```text
db: ouvert
redis: ouvert
pod "test" deleted from vote-app namespace
```

Si `db` apparaît `bloqué`, la base termine son démarrage : attendez 30 secondes et relancez.

> ⚠️ **Pourquoi c'est un problème ?** Ce pod de test n'a rien à voir avec l'application, et il joint pourtant la base de données. Si un jour un pod est compromis (une faille dans une application, une image piégée), il pourra atteindre **tout** le reste. Par défaut, le réseau de Kubernetes est entièrement ouvert.

```ack
J'ai lu « db: ouvert » et « redis: ouvert »
```
