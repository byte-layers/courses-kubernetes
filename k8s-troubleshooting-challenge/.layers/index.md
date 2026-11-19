---
title: "Chasse aux pannes"
description: "Quatre pannes frappent l'application de vote en production, l'une après l'autre : constater le symptôme, enquêter avec méthode, réparer, et remplir la fiche d'incident."
tags:
  - challenge
  - kubernetes
  - containers
difficulty: beginner
duration: 90
introTitle: "Le défi"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

L'application de vote est en production… et les incidents commencent. Quatre pannes ont été préparées. Chacune remplace un de vos objets par une version sabotée, comme le ferait une erreur de manipulation ou une livraison ratée. À vous de constater le symptôme, de mener l'enquête, de réparer, puis de documenter l'incident.

Ce challenge prouve que vous savez diagnostiquer une application sur Kubernetes avec une méthode, et pas au hasard. Hormis la mise en place et le déclenchement de chaque panne, il n'y a **aucune commande à recopier**. Un bouton vérifie que l'application remarche vraiment, quelle que soit la façon dont vous l'avez réparée.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : lire l'état d'un objet avec `kubectl get`, `kubectl describe` et `kubectl logs`, comme dans tous les cours du parcours, et reconnaître les statuts du bit [Lire une panne : CrashLoopBackOff, ImagePullBackOff, OOMKilled](https://byte-layers.com/fr/bits/k8s-failure-catalog). L'application et son cahier des charges sont ceux du lab [Mettre la voting app en production](https://byte-layers.com/fr/labs/k8s-voting-app-production).
- **Ce qu'il faut avoir** : un cluster `layers-k8s` auquel `kubectl` répond, un navigateur, et les ports **28670** et **28671** de votre machine libres.
- **Le terrain de jeu** : le namespace `prod`. La première étape y remet l'application en état de marche, que vous arriviez du lab ou avec un cluster vide. N'oubliez pas `-n prod` dans **toutes** vos commandes : sans lui, `kubectl` regarde dans `default`, qui est vide, et vous répond qu'il n'y a rien.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l7/`, avec les fichiers de l'application et votre fiche d'incident. La dernière étape supprime le namespace `prod` ; le dossier, lui, reste sur votre machine.
- **L'ordre compte** : jouez les pannes une par une, dans l'ordre, et réparez chacune avant de déclencher la suivante.

## Les règles du jeu

1. Chaque panne commence par une commande qui **déclenche la panne**. Son contenu est encodé, et sa réponse part dans un fichier au lieu de s'afficher : la lire trahirait l'objet saboté. Ne décodez pas la commande et n'ouvrez pas les fichiers `panne-N.log` : ce serait tricher 😉.
2. Vous constatez le **symptôme** dans le navigateur, puis vous menez l'enquête avec `kubectl`. Comptez une quinzaine de minutes avant d'ouvrir les indices.
3. Vous **réparez**. Le plus sûr est de réappliquer le bon fichier, choisi dans le tableau ci-dessous. Il est interdit de supprimer le namespace pour tout recommencer !
4. Vous remplissez une ligne de la **fiche d'incident**, puis vous lancez la vérification.

Pour réparer, il faut savoir quel fichier décrit le bon état de chaque objet. Tous sont dans `~/byte-layers/kubernetes/l7/` :

| Objet | Fichier à réappliquer (avec `-n prod`) |
|---|---|
| Ingress `vote` | `vote.ingress.yml` |
| Service et Deployment `vote` | `vote.svc.yml`, `vote.deploy.yml` |
| ConfigMap `vote-config` | `vote.config.yml` |
| Secret `db-credentials` | `db.secret.yml` |
| `redis`, `db`, `result` (Deployment et Service) | `redis.yml`, `db.yml`, `result.yml` |
| `worker` (Deployment) | `worker.yml` |

Après chaque réparation, attendez quelques secondes et revérifiez dans le navigateur que tout remarche. Si les résultats ne bougent toujours pas alors que tout est `Running`, relancez `worker` et `result` avec `kubectl rollout restart deployment worker result -n prod`, puis relancez le tunnel vers la page des résultats, donné dans la mise en place.

## La méthode d'enquête

Une bonne enquête va **du symptôme vers la cause**, en zoomant progressivement :

| Étape | Question | Outils |
|---|---|---|
| 1. Le symptôme | Que voit l'utilisateur ? Quelle page, quelle action ne marche plus ? | le navigateur |
| 2. Vue d'ensemble | Quel objet n'est pas dans son état normal ? | `kubectl get all -n prod`, `kubectl get ingress -n prod` |
| 3. Le détail | Que dit cet objet sur lui-même ? | `kubectl describe <type> <nom> -n prod`. Regardez notamment `Events` (l'historique), `Backends` pour un Ingress (où il envoie les visiteurs), `Selector` et `Endpoints` pour un Service (l'étiquette recherchée, et les pods effectivement servis) |
| 4. L'application | Que dit l'application ? | `kubectl logs <pod> -n prod` (copiez le nom du pod depuis `kubectl get pods -n prod` ; ajoutez `--previous` pour un pod en `CrashLoopBackOff`) |
| 5. La réparation | Quel fichier décrit le bon état ? | `kubectl apply -n prod -f …` |

## La fiche d'incident

Pour chaque panne, vous remplissez une ligne : le symptôme (ce que voit l'utilisateur), la cause (ce que vous avez trouvé, et avec quelle commande), la réparation, et comment l'éviter à l'avenir. C'est exactement ce qu'on attend d'une équipe d'exploitation après un incident (on parle de *post-mortem*), et c'est souvent le chef de projet qui présente ce compte rendu. La mise en place crée la fiche pour vous, dans le fichier `~/byte-layers/kubernetes/l7/incidents.md`. Les vérifications ne la lisent pas : elle est pour vous.

# Mise en place : la production en état de marche

Cette étape est un **support** : avec le déclenchement des pannes, c'est la seule qui donne des commandes. Elle remet l'application de vote en production dans le namespace `prod`, telle que la laisse le lab [Mettre la voting app en production](https://byte-layers.com/fr/labs/k8s-voting-app-production). Si vous en venez, elle ne change rien à ce qui tourne déjà.

## Faire place nette

Si l'application des cours précédents tourne encore dans le namespace `vote-app`, son Ingress occupe la porte d'entrée. Supprimez ce namespace, et tout ce qu'il contient. S'il n'existe plus, la commande répond `NotFound` : c'est sans conséquence.

```exec
kubectl delete namespace vote-app
```

Revenez ensuite au namespace `default`, pour que `kubectl` ne travaille plus dans un namespace supprimé :

```exec
kubectl config set-context --current --namespace=default
```

## Écrire la fiche d'incident et les fichiers de l'application

Créez le dossier de travail. `mkdir -p` ne dit rien s'il existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l7
```

Si vous avez déjà joué ce challenge, le dossier garde les réponses des anciens déclenchements, les fichiers `panne-N.log`, et les vérifications croiraient les pannes déjà déclenchées. Supprimez-les : `find` cherche dans le dossier les fichiers dont le nom correspond à `panne-*.log`, et `-delete` les supprime. La commande ne dit rien, qu'elle en trouve ou non.

```exec
find ~/byte-layers/kubernetes/l7 -name 'panne-*.log' -delete
```

Créez ensuite la fiche d'incident, un tableau vide de quatre lignes. Vous la remplirez en ouvrant ce fichier dans votre éditeur. Si vous rejouez un jour cette étape après avoir commencé la chasse, ne relancez pas cette commande : elle remettrait votre fiche à zéro.

```exec
cat > ~/byte-layers/kubernetes/l7/incidents.md <<'EOF'
# Fiche d'incident

| Panne | Symptôme (ce que voit l'utilisateur) | Cause (ce que j'ai trouvé, et avec quelle commande) | Réparation | Comment l'éviter à l'avenir ? |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
EOF
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'il reçoit en entrée |
> | `> ~/byte-layers/kubernetes/l7/incidents.md` | envoie cette copie dans le fichier, qu'il crée, ou dont il remplace le contenu s'il existe déjà |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles, jusqu'à la ligne `EOF` |
> | les lignes entre les deux | le contenu du fichier |
> | `EOF` | marque la fin du contenu ; ce mot n'est pas écrit dans le fichier |

Les neuf fichiers de l'application s'écrivent de la même façon. Ce sont ceux qui décrivent le **bon état** de chaque objet : vous y reviendrez à chaque réparation. Le premier est la ConfigMap `vote-config`.

```exec
cat > ~/byte-layers/kubernetes/l7/vote.config.yml <<'EOF'
# Réglages de vote : VOTE_TITLE est exigé par la version 3.0
apiVersion: v1
kind: ConfigMap
metadata:
  name: vote-config
data:
  OPTION_A: Montagne
  OPTION_B: Mer
  VOTE_TITLE: "Votre destination de vacances ?"
EOF
```

Le deuxième est le Secret `db-credentials`, les identifiants de la base.

```exec
cat > ~/byte-layers/kubernetes/l7/db.secret.yml <<'EOF'
# Les identifiants de la base, rangés dans un Secret plutôt qu'en clair dans le Deployment
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
EOF
```

`redis.yml` décrit la file d'attente où `vote` dépose les votes, avec son Service.

```exec
cat > ~/byte-layers/kubernetes/l7/redis.yml <<'EOF'
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

`db.yml` décrit la base de données PostgreSQL, avec son Service.

```exec
cat > ~/byte-layers/kubernetes/l7/db.yml <<'EOF'
# db : les identifiants viennent du Secret "db-credentials" (à créer avant).
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

`worker.yml` décrit le programme qui sort les votes de `redis` et les enregistre dans `db`.

```exec
cat > ~/byte-layers/kubernetes/l7/worker.yml <<'EOF'
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

`result.yml` décrit la page des résultats, avec son Service.

```exec
cat > ~/byte-layers/kubernetes/l7/result.yml <<'EOF'
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

`vote.deploy.yml` décrit la page de vote : la version 3.0, en 3 réplicas, avec ses sondes de santé.

```exec
cat > ~/byte-layers/kubernetes/l7/vote.deploy.yml <<'EOF'
# vote : version 3.0, réglages dans vote-config, sondes de santé sur /healthz
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
```

`vote.svc.yml` décrit le Service de `vote`.

```exec
cat > ~/byte-layers/kubernetes/l7/vote.svc.yml <<'EOF'
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

`vote.ingress.yml`, enfin, décrit la porte d'entrée : l'Ingress qui envoie les visites vers le Service `vote`.

```exec
cat > ~/byte-layers/kubernetes/l7/vote.ingress.yml <<'EOF'
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

```ack
J'ai la fiche d'incident et les neuf fichiers dans ~/byte-layers/kubernetes/l7
```

## Déployer l'application dans prod

Sur un poste hébergé, le contrôleur d'Ingress tourne déjà : ne lancez pas les deux premières commandes de cette section. Sur votre poste, assurez-vous que le contrôleur NGINX est actif. S'il l'est déjà, la commande ne change rien :

```exec
minikube -p layers-k8s addons enable ingress
```

Attendez ensuite que le pod du contrôleur soit prêt, pendant 3 minutes au plus :

```exec
kubectl wait -n ingress-nginx --for=condition=Ready pod -l app.kubernetes.io/component=controller --timeout=180s
```

Créez le namespace `prod`. S'il existe déjà, la commande répond `AlreadyExists` : c'est sans conséquence, rien n'est modifié.

```exec
kubectl create namespace prod
```

Envoyez d'abord la ConfigMap et le Secret, que les pods lisent à leur démarrage :

```exec
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/vote.config.yml -f ~/byte-layers/kubernetes/l7/db.secret.yml
```

Envoyez ensuite les quatre morceaux que le visiteur ne voit pas sur la page de vote :

```exec
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/redis.yml -f ~/byte-layers/kubernetes/l7/db.yml -f ~/byte-layers/kubernetes/l7/worker.yml -f ~/byte-layers/kubernetes/l7/result.yml
```

Terminez par `vote`, son Service et son Ingress :

```exec
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/vote.deploy.yml -f ~/byte-layers/kubernetes/l7/vote.svc.yml -f ~/byte-layers/kubernetes/l7/vote.ingress.yml
```

Attendez que les cinq Deployments aient tous leurs pods prêts. `kubectl wait` attend qu'une condition soit remplie, ici `Available`, pour tous les Deployments (`--all`) de `prod`, pendant 2 minutes au plus :

```exec
kubectl wait -n prod --for=condition=Available deployment --all --timeout=120s
```

Le terminal répond une ligne par Deployment :

```text
deployment.apps/db condition met
deployment.apps/redis condition met
deployment.apps/result condition met
deployment.apps/vote condition met
deployment.apps/worker condition met
```

```ack
Les cinq Deployments de prod sont disponibles
```

## Ouvrir la page de vote et la page des résultats

Vous aurez besoin des deux pages pendant toute la chasse, pour observer les symptômes. Sur un poste hébergé, la porte d'entrée est déjà ouverte sur le port 8000 du poste : ne lancez pas la première commande. Sur votre poste, ouvrez un tunnel vers le contrôleur d'Ingress, la porte d'entrée de la page de vote :

```exec
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 28670:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward` | ouvre un tunnel entre votre machine et le cluster |
> | `-n ingress-nginx service/ingress-nginx-controller` | la cible du tunnel : le Service du contrôleur, dans son namespace |
> | `28670:80` | le port 28670 de votre machine mène au port 80 de ce Service |
> | `> /dev/null` | jette ce que la commande affiche, pour ne pas encombrer le terminal |
> | `2>&1` | jette aussi ses messages d'erreur, au même endroit |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main, et le tunnel reste ouvert |

Ouvrez un second tunnel, vers la page des résultats :

```exec
kubectl port-forward -n prod service/result 28671:80 > /dev/null 2>&1 &
```

Ouvrez les deux pages dans votre navigateur, côte à côte si possible. Sur un poste hébergé, passez par l'onglet **PORTS** de VS Code.

| Page | Adresse sur votre poste |
|---|---|
| Vote | `http://localhost:28670` (le port 8000 sur un poste hébergé) |
| Résultats | `http://localhost:28671` |

Votez une fois, pour voir l'application dans son état normal : une coche confirme le vote, et la page des résultats bouge. Laissez les deux tunnels ouverts. Si une page ne répond plus du tout pendant la chasse, y compris après une réparation, son tunnel s'est peut-être arrêté : relancez simplement la même commande.

```ack
Les deux pages s'affichent, et mon vote apparaît dans les résultats
```

# Panne 1

## Déclencher la panne 1

Lancez la commande. Elle décode un fichier saboté et l'envoie dans `prod`, sans rien afficher :

```exec
echo "YXBpVmVyc2lvbjogYXBwcy92MQpraW5kOiBEZXBsb3ltZW50Cm1ldGFkYXRhOgogIG5hbWU6IHJlZGlzCiAgbGFiZWxzOgogICAgYXBwOiByZWRpcwpzcGVjOgogIHJlcGxpY2FzOiAwCiAgc2VsZWN0b3I6CiAgICBtYXRjaExhYmVsczoKICAgICAgYXBwOiByZWRpcwogIHRlbXBsYXRlOgogICAgbWV0YWRhdGE6CiAgICAgIGxhYmVsczoKICAgICAgICBhcHA6IHJlZGlzCiAgICBzcGVjOgogICAgICBjb250YWluZXJzOgogICAgICAgIC0gbmFtZTogcmVkaXMKICAgICAgICAgIGltYWdlOiBnaGNyLmlvL2JuZ2Ftcy9rdWJlLXJlZGlzOjgtYWxwaW5lCiAgICAgICAgICBwb3J0czoKICAgICAgICAgICAgLSBjb250YWluZXJQb3J0OiA2Mzc5CiAgICAgICAgICByZXNvdXJjZXM6CiAgICAgICAgICAgIHJlcXVlc3RzOgogICAgICAgICAgICAgIGNwdTogNTBtCiAgICAgICAgICAgICAgbWVtb3J5OiAzMk1pCg==" | base64 --decode | kubectl apply -n prod -f - > ~/byte-layers/kubernetes/l7/panne-1.log 2>&1
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `echo "…"` | affiche le texte entre guillemets : le fichier saboté, encodé en base64 pour ne pas être lisible d'un coup d'œil |
> | `\|` | passe ce qu'affiche la commande de gauche à la commande de droite, au lieu de l'afficher |
> | `base64 --decode` | décode le texte, et rend le fichier d'origine |
> | `kubectl apply -n prod -f -` | envoie le fichier dans `prod` ; le `-` après `-f` dit de lire le fichier sur l'entrée de la commande, et non sur le disque |
> | `> ~/byte-layers/kubernetes/l7/panne-1.log` | envoie la réponse de `kubectl` dans ce fichier au lieu de l'afficher |
> | `2>&1` | y envoie aussi ses messages d'erreur |

Les trois autres pannes se déclenchent de la même façon.

## Trouver la panne 1 et la réparer

Retournez dans le navigateur. Rechargez la page de vote, votez, regardez la page des résultats : quelque chose ne marche plus. Suivez la méthode d'enquête, réparez, puis remplissez la première ligne de votre fiche d'incident.

**Critères :**

- les 5 morceaux de l'application tournent dans `prod`, chacun avec au moins un pod prêt ;
- la page de vote répond par la porte d'entrée ;
- on peut voter, et les votes sont traités ;
- le namespace `prod` n'a pas été supprimé pour tout recommencer. La vérification ne peut pas le voir, elle vous fait confiance.

```hint
Partez du symptôme : la page de vote s'affiche-t-elle ? Que se passe-t-il quand vous cliquez pour voter ?
```

```hint
Demandez-vous quel morceau `vote` appelle quand on vote, puis passez à la vue d'ensemble. Regardez la colonne `READY` des Deployments : `kubectl get deployments -n prod`. L'un d'eux n'est pas dans son état normal.
```

```hint
Un Deployment peut être « réglé » pour n'avoir aucun pod : c'est son nombre de réplicas. Le fichier qui décrit le bon état de ce morceau est dans votre dossier : réappliquez-le avec `kubectl apply -n prod -f`.
```

```verify
N=1
LOG="$HOME/byte-layers/kubernetes/l7/panne-$N.log"
if ! kubectl get namespace prod >/dev/null 2>&1; then
  echo "ÉCHEC : le namespace prod n'existe pas. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
if ! grep -Eq "configured|created|unchanged" "$LOG" 2>/dev/null; then
  echo "ÉCHEC : la panne $N n'a pas été déclenchée, ou sa commande a échoué. Lancez la commande de la section « Déclencher la panne $N »."
  echo "  Si vous l'avez déjà lancée, et dans ce cas seulement, lisez sa réponse dans $LOG."
  exit 1
fi
END=$((SECONDS + 25))
for d in vote redis db worker result; do
  R=$(kubectl get deployment $d -n prod -o jsonpath='{.spec.replicas}' 2>/dev/null)
  LEFT=$((END - SECONDS)); [ "$LEFT" -lt 2 ] && LEFT=2
  if [ -z "$R" ] || [ "$R" -lt 1 ] || ! kubectl rollout status deployment/$d -n prod --timeout=${LEFT}s >/dev/null 2>&1; then
    echo "ÉCHEC : la panne $N est toujours là. Le premier critère n'est pas rempli : un des cinq morceaux n'a pas de pod prêt."
    echo "  Reprenez la méthode d'enquête, en commençant par la vue d'ensemble."
    echo "  Si vous venez de réparer, laissez aux pods le temps de démarrer et réessayez."
    exit 1
  fi
done
if kubectl get service ingress-nginx-controller -n ingress-nginx >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/ingress-nginx/services/ingress-nginx-controller:80/proxy/"
elif kubectl get service traefik -n kube-system >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/kube-system/services/traefik:80/proxy/"
else
  echo "ÉCHEC : aucun contrôleur d'Ingress n'a été trouvé dans le cluster. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
BACK=$(kubectl get ingress -n prod -o jsonpath='{.items[*].spec.rules[*].http.paths[*].backend.service.name}' 2>/dev/null)
CHAIN=1
[ -z "$BACK" ] && CHAIN=0
for S in $BACK; do
  SEL=$(kubectl get service "$S" -n prod -o go-template='{{range $k, $v := .spec.selector}}{{$k}}={{$v}},{{end}}' 2>/dev/null | sed 's/,$//')
  if [ -z "$SEL" ] || [ -z "$(kubectl get pods -n prod -l "$SEL" --field-selector=status.phase=Running -o name 2>/dev/null)" ]; then
    CHAIN=0
  fi
done
PODS=$(kubectl get pods -n prod -l app=vote -o jsonpath='{.items[*].metadata.name}')
OK=0
for i in 1 2 3 4 5 6; do
  [ "$CHAIN" = 1 ] || break
  HOST=$(kubectl get --raw "$DOOR" 2>/dev/null | sed -n 's/.*Servi par : <strong>\([^<]*\)<.*/\1/p')
  if [ -n "$HOST" ]; then
    case " $PODS " in *" $HOST "*) OK=1; break ;; esac
  fi
  sleep 3
done
if [ "$OK" != 1 ]; then
  echo "ÉCHEC : la panne $N est toujours là. Les cinq morceaux ont un pod prêt, mais la page de vote ne répond pas par la porte d'entrée."
  echo "  Reprenez la méthode d'enquête à partir de ce symptôme, sans sauter d'étape."
  exit 1
fi
if [ "$(kubectl exec -n prod deployment/redis -- redis-cli ping 2>/dev/null)" != "PONG" ]; then
  echo "ÉCHEC : la panne $N est toujours là. La file d'attente redis ne répond pas : un vote ne peut pas être déposé."
  exit 1
fi
for i in 1 2 3 4; do
  Q=$(kubectl exec -n prod deployment/redis -- redis-cli llen votes 2>/dev/null)
  [ "$Q" = "0" ] && break
  sleep 2
done
if [ "$Q" != "0" ]; then
  echo "ÉCHEC : la panne $N est toujours là. $Q vote(s) attendent dans la file redis, et personne ne les traite : les résultats ne bougent pas."
  echo "  Si tous les pods sont Running, relancez worker : kubectl rollout restart deployment worker -n prod"
  exit 1
fi
echo "RÉUSSI : la panne $N est réparée. Les 5 morceaux tournent, la porte d'entrée répond, et les votes sont traités."
```

````solution
Voici ce que votre fiche d'incident pouvait dire de cette panne :

| Symptôme | Cause | Commande qui la révèle | Comment l'éviter |
|---|---|---|---|
| La page de vote s'affiche, mais voter renvoie `Internal Server Error` | `redis` a été mis à **0 réplica** : `vote` ne peut plus déposer les votes | `kubectl get deployments -n prod` affiche `redis 0/0` | Le fichier fait foi : des modifications tracées dans Git, et une relecture avant d'appliquer |

La réparation consiste à réappliquer le fichier qui décrit le bon état de `redis`, puis à attendre son pod :

```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/redis.yml
kubectl rollout status deployment/redis -n prod --timeout=50s
```
````

# Panne 2

## Déclencher la panne 2

L'application remarche : passez à l'incident suivant. La commande a la même forme que la précédente, avec un autre fichier saboté :

```exec
echo "YXBpVmVyc2lvbjogbmV0d29ya2luZy5rOHMuaW8vdjEKa2luZDogSW5ncmVzcwptZXRhZGF0YToKICBuYW1lOiB2b3RlCnNwZWM6CiAgcnVsZXM6CiAgICAtIGh0dHA6CiAgICAgICAgcGF0aHM6CiAgICAgICAgICAtIHBhdGg6IC8KICAgICAgICAgICAgcGF0aFR5cGU6IFByZWZpeAogICAgICAgICAgICBiYWNrZW5kOgogICAgICAgICAgICAgIHNlcnZpY2U6CiAgICAgICAgICAgICAgICBuYW1lOiB2b3RlLWZyb250ZW5kCiAgICAgICAgICAgICAgICBwb3J0OgogICAgICAgICAgICAgICAgICBudW1iZXI6IDgwCg==" | base64 --decode | kubectl apply -n prod -f - > ~/byte-layers/kubernetes/l7/panne-2.log 2>&1
```

## Trouver la panne 2 et la réparer

Retournez dans le navigateur, cherchez ce qui ne marche plus, puis menez l'enquête. Réparez, et remplissez la deuxième ligne de votre fiche d'incident.

**Critères :** les mêmes que pour la panne 1. Les 5 morceaux tournent, la page de vote répond par la porte d'entrée, les votes sont traités, et le namespace `prod` n'a pas été supprimé.

```hint
Partez du symptôme : quelle page ne s'affiche plus, et que répond-elle ? Les pods de `vote` tournent-ils ? Si oui, le problème est sur le **chemin** qui mène jusqu'à eux.
```

```hint
Le chemin d'une visite passe par le navigateur, le contrôleur d'Ingress, l'**Ingress**, le **Service**, puis les pods. Suivez la chaîne depuis le début : `kubectl describe ingress vote -n prod`.
```

```hint
Lisez la ligne `Backends` de la description : vers quel Service l'Ingress envoie-t-il les visiteurs ? Ce Service existe-t-il dans `kubectl get services -n prod` ? Le fichier qui décrit le bon Ingress est dans votre dossier.
```

```verify
N=2
LOG="$HOME/byte-layers/kubernetes/l7/panne-$N.log"
if ! kubectl get namespace prod >/dev/null 2>&1; then
  echo "ÉCHEC : le namespace prod n'existe pas. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
if ! grep -Eq "configured|created|unchanged" "$LOG" 2>/dev/null; then
  echo "ÉCHEC : la panne $N n'a pas été déclenchée, ou sa commande a échoué. Lancez la commande de la section « Déclencher la panne $N »."
  echo "  Si vous l'avez déjà lancée, et dans ce cas seulement, lisez sa réponse dans $LOG."
  exit 1
fi
END=$((SECONDS + 25))
for d in vote redis db worker result; do
  R=$(kubectl get deployment $d -n prod -o jsonpath='{.spec.replicas}' 2>/dev/null)
  LEFT=$((END - SECONDS)); [ "$LEFT" -lt 2 ] && LEFT=2
  if [ -z "$R" ] || [ "$R" -lt 1 ] || ! kubectl rollout status deployment/$d -n prod --timeout=${LEFT}s >/dev/null 2>&1; then
    echo "ÉCHEC : la panne $N est toujours là. Le premier critère n'est pas rempli : un des cinq morceaux n'a pas de pod prêt."
    echo "  Reprenez la méthode d'enquête, en commençant par la vue d'ensemble."
    echo "  Si vous venez de réparer, laissez aux pods le temps de démarrer et réessayez."
    exit 1
  fi
done
if kubectl get service ingress-nginx-controller -n ingress-nginx >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/ingress-nginx/services/ingress-nginx-controller:80/proxy/"
elif kubectl get service traefik -n kube-system >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/kube-system/services/traefik:80/proxy/"
else
  echo "ÉCHEC : aucun contrôleur d'Ingress n'a été trouvé dans le cluster. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
BACK=$(kubectl get ingress -n prod -o jsonpath='{.items[*].spec.rules[*].http.paths[*].backend.service.name}' 2>/dev/null)
CHAIN=1
[ -z "$BACK" ] && CHAIN=0
for S in $BACK; do
  SEL=$(kubectl get service "$S" -n prod -o go-template='{{range $k, $v := .spec.selector}}{{$k}}={{$v}},{{end}}' 2>/dev/null | sed 's/,$//')
  if [ -z "$SEL" ] || [ -z "$(kubectl get pods -n prod -l "$SEL" --field-selector=status.phase=Running -o name 2>/dev/null)" ]; then
    CHAIN=0
  fi
done
PODS=$(kubectl get pods -n prod -l app=vote -o jsonpath='{.items[*].metadata.name}')
OK=0
for i in 1 2 3 4 5 6; do
  [ "$CHAIN" = 1 ] || break
  HOST=$(kubectl get --raw "$DOOR" 2>/dev/null | sed -n 's/.*Servi par : <strong>\([^<]*\)<.*/\1/p')
  if [ -n "$HOST" ]; then
    case " $PODS " in *" $HOST "*) OK=1; break ;; esac
  fi
  sleep 3
done
if [ "$OK" != 1 ]; then
  echo "ÉCHEC : la panne $N est toujours là. Les cinq morceaux ont un pod prêt, mais la page de vote ne répond pas par la porte d'entrée."
  echo "  Reprenez la méthode d'enquête à partir de ce symptôme, sans sauter d'étape."
  exit 1
fi
if [ "$(kubectl exec -n prod deployment/redis -- redis-cli ping 2>/dev/null)" != "PONG" ]; then
  echo "ÉCHEC : la panne $N est toujours là. La file d'attente redis ne répond pas : un vote ne peut pas être déposé."
  exit 1
fi
for i in 1 2 3 4; do
  Q=$(kubectl exec -n prod deployment/redis -- redis-cli llen votes 2>/dev/null)
  [ "$Q" = "0" ] && break
  sleep 2
done
if [ "$Q" != "0" ]; then
  echo "ÉCHEC : la panne $N est toujours là. $Q vote(s) attendent dans la file redis, et personne ne les traite : les résultats ne bougent pas."
  echo "  Si tous les pods sont Running, relancez worker : kubectl rollout restart deployment worker -n prod"
  exit 1
fi
echo "RÉUSSI : la panne $N est réparée. Les 5 morceaux tournent, la porte d'entrée répond, et les votes sont traités."
```

````solution
Voici ce que votre fiche d'incident pouvait dire de cette panne :

| Symptôme | Cause | Commande qui la révèle | Comment l'éviter |
|---|---|---|---|
| La page de vote ne s'affiche plus : erreur `503` avec le contrôleur NGINX, sur votre poste, ou `404` avec Traefik, sur un poste hébergé | L'Ingress envoie les visiteurs vers un Service **qui n'existe pas**, `vote-frontend` | `kubectl describe ingress vote -n prod` affiche `services "vote-frontend" not found` à la ligne `Backends` | Vérifier les noms à la relecture, et tester l'accès après chaque déploiement |

La réparation consiste à réappliquer le fichier qui décrit le bon Ingress :

```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/vote.ingress.yml
```
````

# Panne 3

## Déclencher la panne 3

La porte d'entrée répond de nouveau : déclenchez la troisième panne.

```exec
echo "YXBpVmVyc2lvbjogdjEKa2luZDogU2VydmljZQptZXRhZGF0YToKICBuYW1lOiB2b3RlCiAgbGFiZWxzOgogICAgYXBwOiB2b3RlCnNwZWM6CiAgc2VsZWN0b3I6CiAgICBhcHA6IHZvdGVzCiAgcG9ydHM6CiAgICAtIHBvcnQ6IDgwCiAgICAgIHRhcmdldFBvcnQ6IDgwCg==" | base64 --decode | kubectl apply -n prod -f - > ~/byte-layers/kubernetes/l7/panne-3.log 2>&1
```

## Trouver la panne 3 et la réparer

Retournez dans le navigateur, constatez le symptôme, puis menez l'enquête sans rien supposer : c'est l'enquête, pas l'intuition, qui tranche. Réparez, et remplissez la troisième ligne de votre fiche d'incident.

**Critères :** les mêmes que pour la panne 1. Les 5 morceaux tournent, la page de vote répond par la porte d'entrée, les votes sont traités, et le namespace `prod` n'a pas été supprimé.

```hint
Le symptôme ressemble à celui de la panne précédente. Un même symptôme peut avoir **deux causes différentes** : refaites le chemin d'une visite depuis le début, sans sauter d'étape. L'Ingress envoie-t-il vers un Service qui existe, cette fois ?
```

```hint
Si l'Ingress est correct, passez au maillon suivant : `kubectl describe service vote -n prod`. La ligne `Endpoints` doit lister des adresses de pods.
```

```hint
Comparez le `Selector` du Service, l'étiquette qu'il recherche, avec les étiquettes réelles des pods : `kubectl get pods -n prod --show-labels`. Le fichier qui décrit le bon Service est dans votre dossier.
```

```verify
N=3
LOG="$HOME/byte-layers/kubernetes/l7/panne-$N.log"
if ! kubectl get namespace prod >/dev/null 2>&1; then
  echo "ÉCHEC : le namespace prod n'existe pas. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
if ! grep -Eq "configured|created|unchanged" "$LOG" 2>/dev/null; then
  echo "ÉCHEC : la panne $N n'a pas été déclenchée, ou sa commande a échoué. Lancez la commande de la section « Déclencher la panne $N »."
  echo "  Si vous l'avez déjà lancée, et dans ce cas seulement, lisez sa réponse dans $LOG."
  exit 1
fi
END=$((SECONDS + 25))
for d in vote redis db worker result; do
  R=$(kubectl get deployment $d -n prod -o jsonpath='{.spec.replicas}' 2>/dev/null)
  LEFT=$((END - SECONDS)); [ "$LEFT" -lt 2 ] && LEFT=2
  if [ -z "$R" ] || [ "$R" -lt 1 ] || ! kubectl rollout status deployment/$d -n prod --timeout=${LEFT}s >/dev/null 2>&1; then
    echo "ÉCHEC : la panne $N est toujours là. Le premier critère n'est pas rempli : un des cinq morceaux n'a pas de pod prêt."
    echo "  Reprenez la méthode d'enquête, en commençant par la vue d'ensemble."
    echo "  Si vous venez de réparer, laissez aux pods le temps de démarrer et réessayez."
    exit 1
  fi
done
if kubectl get service ingress-nginx-controller -n ingress-nginx >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/ingress-nginx/services/ingress-nginx-controller:80/proxy/"
elif kubectl get service traefik -n kube-system >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/kube-system/services/traefik:80/proxy/"
else
  echo "ÉCHEC : aucun contrôleur d'Ingress n'a été trouvé dans le cluster. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
BACK=$(kubectl get ingress -n prod -o jsonpath='{.items[*].spec.rules[*].http.paths[*].backend.service.name}' 2>/dev/null)
CHAIN=1
[ -z "$BACK" ] && CHAIN=0
for S in $BACK; do
  SEL=$(kubectl get service "$S" -n prod -o go-template='{{range $k, $v := .spec.selector}}{{$k}}={{$v}},{{end}}' 2>/dev/null | sed 's/,$//')
  if [ -z "$SEL" ] || [ -z "$(kubectl get pods -n prod -l "$SEL" --field-selector=status.phase=Running -o name 2>/dev/null)" ]; then
    CHAIN=0
  fi
done
PODS=$(kubectl get pods -n prod -l app=vote -o jsonpath='{.items[*].metadata.name}')
OK=0
for i in 1 2 3 4 5 6; do
  [ "$CHAIN" = 1 ] || break
  HOST=$(kubectl get --raw "$DOOR" 2>/dev/null | sed -n 's/.*Servi par : <strong>\([^<]*\)<.*/\1/p')
  if [ -n "$HOST" ]; then
    case " $PODS " in *" $HOST "*) OK=1; break ;; esac
  fi
  sleep 3
done
if [ "$OK" != 1 ]; then
  echo "ÉCHEC : la panne $N est toujours là. Les cinq morceaux ont un pod prêt, mais la page de vote ne répond pas par la porte d'entrée."
  echo "  Reprenez la méthode d'enquête à partir de ce symptôme, sans sauter d'étape."
  exit 1
fi
if [ "$(kubectl exec -n prod deployment/redis -- redis-cli ping 2>/dev/null)" != "PONG" ]; then
  echo "ÉCHEC : la panne $N est toujours là. La file d'attente redis ne répond pas : un vote ne peut pas être déposé."
  exit 1
fi
for i in 1 2 3 4; do
  Q=$(kubectl exec -n prod deployment/redis -- redis-cli llen votes 2>/dev/null)
  [ "$Q" = "0" ] && break
  sleep 2
done
if [ "$Q" != "0" ]; then
  echo "ÉCHEC : la panne $N est toujours là. $Q vote(s) attendent dans la file redis, et personne ne les traite : les résultats ne bougent pas."
  echo "  Si tous les pods sont Running, relancez worker : kubectl rollout restart deployment worker -n prod"
  exit 1
fi
echo "RÉUSSI : la panne $N est réparée. Les 5 morceaux tournent, la porte d'entrée répond, et les votes sont traités."
```

````solution
Voici ce que votre fiche d'incident pouvait dire de cette panne :

| Symptôme | Cause | Commande qui la révèle | Comment l'éviter |
|---|---|---|---|
| La page de vote ne s'affiche plus : erreur `503`, alors que l'Ingress semble correct | Le **sélecteur** du Service `vote` cherche l'étiquette `app: votes`, qui ne correspond à aucun pod : ils portent `app: vote` | `kubectl describe service vote -n prod` affiche une ligne `Endpoints` vide | Une étiquette mal recopiée suffit : des outils de validation, et des tests automatiques après chaque déploiement |

Les pannes 2 et 3 ont le même symptôme, mais pas la même cause. Seule l'enquête le long de la chaîne, du contrôleur d'Ingress jusqu'aux pods, permet de trancher.

La réparation consiste à réappliquer le fichier qui décrit le bon Service :

```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/vote.svc.yml
```
````

# Panne 4

## Déclencher la panne 4

Plus qu'une. Déclenchez la dernière panne :

```exec
echo "YXBpVmVyc2lvbjogYXBwcy92MQpraW5kOiBEZXBsb3ltZW50Cm1ldGFkYXRhOgogIG5hbWU6IHdvcmtlcgogIGxhYmVsczoKICAgIGFwcDogd29ya2VyCnNwZWM6CiAgcmVwbGljYXM6IDEKICBzdHJhdGVneToKICAgIHR5cGU6IFJlY3JlYXRlCiAgc2VsZWN0b3I6CiAgICBtYXRjaExhYmVsczoKICAgICAgYXBwOiB3b3JrZXIKICB0ZW1wbGF0ZToKICAgIG1ldGFkYXRhOgogICAgICBsYWJlbHM6CiAgICAgICAgYXBwOiB3b3JrZXIKICAgIHNwZWM6CiAgICAgIGNvbnRhaW5lcnM6CiAgICAgICAgLSBuYW1lOiB3b3JrZXIKICAgICAgICAgIGltYWdlOiBnaGNyLmlvL2JuZ2Ftcy9rdWJlLXdvcmtlcjoxLjEKICAgICAgICAgIHJlc291cmNlczoKICAgICAgICAgICAgcmVxdWVzdHM6CiAgICAgICAgICAgICAgY3B1OiA1MG0KICAgICAgICAgICAgICBtZW1vcnk6IDY0TWkK" | base64 --decode | kubectl apply -n prod -f - > ~/byte-layers/kubernetes/l7/panne-4.log 2>&1
```

## Trouver la panne 4 et la réparer

Retournez dans le navigateur. Cette fois, regardez bien les deux pages, et votez plusieurs fois. Menez l'enquête, réparez, et remplissez la dernière ligne de votre fiche d'incident.

**Critères :** les mêmes que pour la panne 1. Les 5 morceaux tournent, la page de vote répond par la porte d'entrée, les votes sont traités, et le namespace `prod` n'a pas été supprimé.

```hint
Partez du symptôme : comparez les deux pages après plusieurs votes. Laquelle ne réagit plus ?
```

```hint
Demandez-vous qui transporte les votes de `redis` vers `db`. Regardez l'état des pods de ce morceau avec `kubectl get pods -n prod`, puis décrivez celui qui est en échec avec `kubectl describe pod`. Le statut affiché fait partie du catalogue des pannes.
```

```hint
Les `Events`, en bas de la description du pod, disent quelle image Kubernetes n'arrive pas à télécharger. Comparez-la avec celle du fichier qui décrit le bon état de ce morceau, puis réappliquez ce fichier.
```

```verify
N=4
LOG="$HOME/byte-layers/kubernetes/l7/panne-$N.log"
if ! kubectl get namespace prod >/dev/null 2>&1; then
  echo "ÉCHEC : le namespace prod n'existe pas. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
if ! grep -Eq "configured|created|unchanged" "$LOG" 2>/dev/null; then
  echo "ÉCHEC : la panne $N n'a pas été déclenchée, ou sa commande a échoué. Lancez la commande de la section « Déclencher la panne $N »."
  echo "  Si vous l'avez déjà lancée, et dans ce cas seulement, lisez sa réponse dans $LOG."
  exit 1
fi
END=$((SECONDS + 25))
for d in vote redis db worker result; do
  R=$(kubectl get deployment $d -n prod -o jsonpath='{.spec.replicas}' 2>/dev/null)
  LEFT=$((END - SECONDS)); [ "$LEFT" -lt 2 ] && LEFT=2
  if [ -z "$R" ] || [ "$R" -lt 1 ] || ! kubectl rollout status deployment/$d -n prod --timeout=${LEFT}s >/dev/null 2>&1; then
    echo "ÉCHEC : la panne $N est toujours là. Le premier critère n'est pas rempli : un des cinq morceaux n'a pas de pod prêt."
    echo "  Reprenez la méthode d'enquête, en commençant par la vue d'ensemble."
    echo "  Si vous venez de réparer, laissez aux pods le temps de démarrer et réessayez."
    exit 1
  fi
done
if kubectl get service ingress-nginx-controller -n ingress-nginx >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/ingress-nginx/services/ingress-nginx-controller:80/proxy/"
elif kubectl get service traefik -n kube-system >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/kube-system/services/traefik:80/proxy/"
else
  echo "ÉCHEC : aucun contrôleur d'Ingress n'a été trouvé dans le cluster. Rejouez l'étape « Mise en place : la production en état de marche »."
  exit 1
fi
BACK=$(kubectl get ingress -n prod -o jsonpath='{.items[*].spec.rules[*].http.paths[*].backend.service.name}' 2>/dev/null)
CHAIN=1
[ -z "$BACK" ] && CHAIN=0
for S in $BACK; do
  SEL=$(kubectl get service "$S" -n prod -o go-template='{{range $k, $v := .spec.selector}}{{$k}}={{$v}},{{end}}' 2>/dev/null | sed 's/,$//')
  if [ -z "$SEL" ] || [ -z "$(kubectl get pods -n prod -l "$SEL" --field-selector=status.phase=Running -o name 2>/dev/null)" ]; then
    CHAIN=0
  fi
done
PODS=$(kubectl get pods -n prod -l app=vote -o jsonpath='{.items[*].metadata.name}')
OK=0
for i in 1 2 3 4 5 6; do
  [ "$CHAIN" = 1 ] || break
  HOST=$(kubectl get --raw "$DOOR" 2>/dev/null | sed -n 's/.*Servi par : <strong>\([^<]*\)<.*/\1/p')
  if [ -n "$HOST" ]; then
    case " $PODS " in *" $HOST "*) OK=1; break ;; esac
  fi
  sleep 3
done
if [ "$OK" != 1 ]; then
  echo "ÉCHEC : la panne $N est toujours là. Les cinq morceaux ont un pod prêt, mais la page de vote ne répond pas par la porte d'entrée."
  echo "  Reprenez la méthode d'enquête à partir de ce symptôme, sans sauter d'étape."
  exit 1
fi
if [ "$(kubectl exec -n prod deployment/redis -- redis-cli ping 2>/dev/null)" != "PONG" ]; then
  echo "ÉCHEC : la panne $N est toujours là. La file d'attente redis ne répond pas : un vote ne peut pas être déposé."
  exit 1
fi
for i in 1 2 3 4; do
  Q=$(kubectl exec -n prod deployment/redis -- redis-cli llen votes 2>/dev/null)
  [ "$Q" = "0" ] && break
  sleep 2
done
if [ "$Q" != "0" ]; then
  echo "ÉCHEC : la panne $N est toujours là. $Q vote(s) attendent dans la file redis, et personne ne les traite : les résultats ne bougent pas."
  echo "  Si tous les pods sont Running, relancez worker : kubectl rollout restart deployment worker -n prod"
  exit 1
fi
echo "RÉUSSI : la panne $N est réparée. Les 5 morceaux tournent, la porte d'entrée répond, et les votes sont traités."
```

````solution
Voici ce que votre fiche d'incident pouvait dire de cette panne :

| Symptôme | Cause | Commande qui la révèle | Comment l'éviter |
|---|---|---|---|
| On vote, la coche apparaît, mais les résultats ne bougent plus | `worker` a été déployé avec une **image qui n'existe pas**, `kube-worker:1.1`, et la stratégie `Recreate` a supprimé l'ancien pod avant de lancer le nouveau | `kubectl get pods -n prod` affiche `ImagePullBackOff` ou `ErrImagePull`, puis `kubectl describe pod` donne le nom de l'image introuvable | Vérifier que l'image est publiée avant de la déployer, et préférer la mise à jour progressive, qui aurait gardé l'ancien pod |

La réparation consiste à réappliquer le fichier qui décrit le bon état de `worker`, puis à attendre son pod :

```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/worker.yml
kubectl rollout status deployment/worker -n prod --timeout=50s
```
````

# Après la chasse : débriefer, puis tout ranger

## Relire sa fiche d'incident

Les quatre pannes sont réparées. Relisez votre fiche d'incident, `~/byte-layers/kubernetes/l7/incidents.md`, en la comparant à ces trois points, ceux qu'on souligne dans un débriefing :

- **Les pannes 2 et 3** ont le même symptôme et des causes différentes. Seule l'enquête le long de la chaîne, du navigateur au contrôleur, à l'Ingress, au Service puis aux pods, permet de trancher.
- **Pendant la panne 4, les votes n'étaient pas perdus.** Ils attendaient dans la file `redis`, et dès que `worker` est revenu, il les a tous traités : c'est l'intérêt d'une file d'attente entre deux morceaux d'une application.
- **Avec la stratégie par défaut, la mise à jour progressive, la panne 4 aurait été invisible** : l'ancien pod de `worker` aurait continué à tourner pendant que le nouveau échouait. Le fichier saboté imposait la stratégie `Recreate` (« tout arrêter, puis redémarrer »). Elle se justifie parfois, mais elle supprime ce filet de sécurité.

```ack
Ma fiche d'incident a quatre lignes remplies, avec la commande qui a révélé chaque cause
```

## Supprimer la production et fermer les tunnels

La chasse est terminée : rendez le cluster propre. Arrêtez les deux tunnels ouverts à la mise en place, puis supprimez le namespace `prod`, ce qui supprime toute l'application. Votre dossier `~/byte-layers/kubernetes/l7/`, avec vos fichiers et votre fiche d'incident, reste sur votre machine, et le contrôleur d'Ingress reste installé dans le cluster. La chasse terminée, vous pouvez ouvrir les fichiers `panne-N.log` : ils ne trahissent plus rien.

**Critères :**

- plus aucun tunnel `kubectl port-forward` n'occupe les ports 28670 et 28671 ;
- le namespace `prod` n'existe plus, ou il est en cours de suppression.

```hint
`pkill -f` arrête les programmes dont la ligne de commande contient le texte donné, par exemple `pkill -f "port-forward -n prod service/result 28671"`.
```

```hint
Supprimer un namespace supprime tout ce qu'il contient : `kubectl delete namespace`, suivi du nom. La commande peut mettre une trentaine de secondes à rendre la main. Sur un poste hébergé, seul le tunnel du port 28671 est à arrêter.
```

```verify
LEFT=$(pgrep -fl "port-forward.*2867[01]" 2>/dev/null)
if [ -n "$LEFT" ]; then
  echo "ÉCHEC : un tunnel est encore ouvert sur le port 28670 ou 28671 :"
  echo "$LEFT" | sed 's/^/  /'
  echo "  Arrêtez-le avec pkill -f, suivi entre guillemets d'un morceau de sa ligne de commande."
  exit 1
fi
PHASE=$(kubectl get namespace prod -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$PHASE" = "Active" ]; then
  echo "ÉCHEC : le namespace prod existe encore."
  echo "  Supprimez-le avec kubectl delete namespace."
  exit 1
fi
if ! kubectl get namespaces >/dev/null 2>&1; then
  echo "ÉCHEC : le cluster ne répond pas. Vérifiez-le avec kubectl get nodes, puis réessayez."
  exit 1
fi
echo "RÉUSSI : les tunnels sont fermés et la production est supprimée. Vous savez enquêter sur une panne, la réparer et la documenter."
```

````solution
```bash
pkill -f "port-forward -n ingress-nginx service/ingress-nginx-controller 28670"
pkill -f "port-forward -n prod service/result 28671"
kubectl delete namespace prod --timeout=50s
```
````
