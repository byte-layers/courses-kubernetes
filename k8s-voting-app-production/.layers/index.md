---
title: "Mettre la voting app en production"
description: "Lab de synthèse Kubernetes : déployer l'application de vote complète dans un namespace prod à partir d'un cahier des charges, dans un ordre réfléchi, puis faire sa recette exigence par exigence."
tags:
  - lab
  - kubernetes
  - containers
difficulty: beginner
duration: 60
introTitle: "Présentation"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Aux cours précédents, vous avez été guidé pas à pas pour acquérir les bases. Place à la pratique : vous allez **mettre en production** l'application de vote complète dans un nouveau namespace, `prod`, à partir d'un **cahier des charges**, puis vérifier vous-même qu'elle le respecte. Cette vérification s'appelle la **recette**.

Il y a moins de commandes toutes faites ici. Chaque exigence donne un objectif et des critères, et c'est vous qui écrivez les commandes, en vous appuyant sur vos fichiers. Un bouton vérifie l'état réel du cluster, les indices se dévoilent un par un, et la solution reste en dernier recours.

**Niveau** : débutant, en fin de parcours. Vous avez déjà déployé l'application de vote et ses cinq morceaux, dans [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app), et vous lui avez ouvert une porte d'entrée, dans [Ouvrir une porte d'entrée avec un Ingress](https://byte-layers.com/fr/skills/k8s-ingress). Vous savez créer un namespace et lui adresser une commande avec `-n`, ce qu'enseigne [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas). La ConfigMap, le Secret et les sondes de santé viennent de [Sortir les réglages de l'image avec une ConfigMap](https://byte-layers.com/fr/skills/k8s-configmap), de [Ranger un mot de passe dans un Secret](https://byte-layers.com/fr/skills/k8s-secret) et de [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes).

**Ce qu'il faut** : un cluster `layers-k8s` auquel `kubectl` répond, celui du lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install), un navigateur, `curl`, et les ports **28670** et **28671** de votre machine libres.

**Où vivent les fichiers** : dans `~/byte-layers/kubernetes/l7/`. Rien n'est supprimé de ce dossier à la fin.

## Comment ça se passe

Le lab compte quatre étapes, à suivre dans l'ordre, pour environ une heure. Les deux premières sont des **supports** : elles suppriment le namespace `vote-app` des cours précédents, puis rassemblent dans un seul dossier les neuf fichiers de l'application, que vous arriviez du cours précédent ou avec un cluster vide. La troisième est la mission : après la lecture du cahier des charges, cinq sections, chacune avec ses critères et sa vérification. La quatrième est la recette : vous passez le cahier des charges en revue, jusqu'à voir un vote arriver sur la page des résultats.

Quelques gestes dépendent de l'endroit où tourne votre cluster, sur votre poste ou sur un poste hébergé (un VS Code dans le navigateur, fourni par un formateur) : le texte le dit à chaque fois.

**Ce qui reste dans le cluster à la fin** : l'application de vote complète dans le namespace `prod`, avec son Ingress. Le namespace `vote-app` n'existe plus, et `default` est redevenu le namespace dans lequel votre `kubectl` travaille. Le challenge [Chasse aux pannes](https://byte-layers.com/fr/challenges/k8s-troubleshooting-challenge) part de cet état : ne supprimez rien si vous l'enchaînez.

# Faire place nette

Cette étape est un **support** : elle prépare le cluster, et chaque geste est une commande à lancer.

## Supprimer le namespace vote-app

Votre application tourne dans le namespace `vote-app`. La production aura son propre namespace, `prod` : commencez par supprimer `vote-app`, pour repartir d'une situation claire et éviter que deux Ingress se disputent la porte d'entrée. Supprimer un namespace supprime **tout ce qu'il contient** : Deployments, Services, ConfigMap, Secret, quota, règles réseau. Vos fichiers, eux, restent intacts sur votre machine.

```exec
kubectl delete namespace vote-app
```

La commande peut mettre une trentaine de secondes à rendre la main, le temps que tout soit supprimé :

```text
namespace "vote-app" deleted
```

Si vous arrivez avec un cluster vide, elle répond `Error from server (NotFound): namespaces "vote-app" not found` : c'est sans conséquence, il n'y avait rien à supprimer.

## Revenir au namespace default

Depuis le cours sur les namespaces, votre `kubectl` travaille dans `vote-app`, qui n'existe plus. Revenez au namespace d'origine, `default`. Pour la production, vous désignerez `prod` explicitement, avec `-n prod`, dans chaque commande : sur un environnement de production, mieux vaut écrire où l'on agit que le supposer.

```exec
kubectl config set-context --current --namespace=default
```

La réponse confirme le réglage. Sur un poste hébergé, le nom entre guillemets est `k3d-layers-k8s` :

```text
Context "layers-k8s" modified.
```

Vérifiez le résultat des deux commandes en demandant la liste des namespaces du cluster :

```exec
kubectl get namespaces
```

`vote-app` a disparu de la liste. Sur votre poste, elle ressemble à ceci :

```text
NAME              STATUS   AGE
default           Active   54m
ingress-nginx     Active   53m
kube-node-lease   Active   54m
kube-public       Active   54m
kube-system       Active   54m
```

```ack
vote-app n'est plus dans la liste, et kubectl travaille dans default
```

# Rassembler les fichiers et vérifier le contrôleur d'Ingress

Cette étape est aussi un **support**. Au fil du parcours, vous avez écrit ou reçu neuf fichiers, rangés dans plusieurs dossiers. Elle les réécrit tous, dans leur dernière version, dans un seul dossier : vous pouvez la jouer sans vous arrêter sur chaque ligne, tout vient des cours précédents.

## Créer le dossier de la mise en production

Créez le dossier de travail. `mkdir -p` crée aussi les dossiers parents qui manquent, et ne dit rien si le dossier existe déjà :

```exec
mkdir -p ~/byte-layers/kubernetes/l7
```

## Les deux fichiers de réglages : la ConfigMap et le Secret

Écrivez d'abord `vote.config.yml`, la ConfigMap `vote-config` : les deux options du vote, et le titre `VOTE_TITLE` que la version 3.0 de `vote` exige.

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

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'il reçoit en entrée |
> | `> ~/byte-layers/kubernetes/l7/vote.config.yml` | envoie cette copie dans le fichier, qu'il crée, ou dont il remplace le contenu s'il existe déjà |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles, jusqu'à la ligne `EOF` |
> | les lignes entre les deux | le contenu du fichier |
> | `EOF` | marque la fin du contenu ; ce mot n'est pas écrit dans le fichier |

Les neuf fichiers s'écrivent de la même façon. Écrivez ensuite `db.secret.yml`, le Secret `db-credentials`, qui porte l'utilisateur et le mot de passe de la base.

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

## Les trois fichiers de vote

`vote.deploy.yml` est le Deployment de `vote`, la page où l'on vote, dans son dernier état : la version 3.0, en 3 réplicas, avec ses réglages lus dans `vote-config`, ses deux sondes de santé et ses réservations de ressources.

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

`vote.svc.yml` est son Service, l'adresse stable `vote` à l'intérieur du cluster.

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

`vote.ingress.yml` est la règle d'aiguillage de la porte d'entrée : tout ce qui arrive sur `/` part vers le Service `vote`.

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

## Les quatre fichiers des autres morceaux

Les quatre autres morceaux tiennent chacun dans un fichier, qui décrit un Deployment et, quand un autre morceau l'appelle, son Service. `redis.yml` décrit la file d'attente où `vote` dépose les votes.

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

`db.yml` décrit la base de données PostgreSQL, dans la version qui lit ses identifiants dans le Secret `db-credentials`.

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

`result.yml` décrit la page des résultats, qui lit `db`.

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

Listez le dossier pour vérifier que rien ne manque :

```exec
ls ~/byte-layers/kubernetes/l7
```

Vous devez y trouver au moins ces neuf fichiers, présentés ici comme sur macOS :

```text
db.secret.yml		redis.yml		vote.config.yml		vote.ingress.yml	worker.yml
db.yml			result.yml		vote.deploy.yml		vote.svc.yml
```

```ack
J'ai les neuf fichiers dans ~/byte-layers/kubernetes/l7
```

## Vérifier que le cluster a un contrôleur d'Ingress

Le cahier des charges demande une porte d'entrée, et un Ingress ne fait rien sans son **contrôleur**, le programme qui reçoit les visites. Sur un poste hébergé, le contrôleur Traefik tourne déjà : ne lancez pas les deux commandes de cette section.

Sur votre poste, vous avez activé le contrôleur NGINX dans le cours sur l'Ingress. Cette commande s'en assure : s'il est déjà actif, elle ne change rien, sinon elle l'installe, en une minute environ. `-p layers-k8s` désigne votre cluster par le nom de son profil.

```exec
minikube -p layers-k8s addons enable ingress
```

Attendez ensuite que le contrôleur soit prêt. `kubectl wait` attend qu'une condition soit remplie, ici « le pod du contrôleur est prêt », pendant 3 minutes au plus :

```exec
kubectl wait -n ingress-nginx --for=condition=Ready pod -l app.kubernetes.io/component=controller --timeout=180s
```

Le terminal répond quand le pod est prêt :

```text
pod/ingress-nginx-controller-9cc49f96f-rpt2t condition met
```

```ack
Mon cluster a un contrôleur d'Ingress prêt
```

# La mission : mettre en production

Vous avez les fichiers, et un cluster sans application. Il reste à mettre le tout en production, en respectant ce qu'on vous demande.

## Lire le cahier des charges et réfléchir à l'ordre

Voici le **cahier des charges** de la mise en production, tel qu'un chef de projet pourrait le rédiger :

| # | Exigence |
|---|---|
| 1 | Toute l'application vit dans un namespace nommé `prod` |
| 2 | Les 5 morceaux tournent : `vote`, `redis`, `worker`, `db`, `result` |
| 3 | `vote` tourne en **version 3.0**, en **3 réplicas**, avec ses **sondes de santé** et ses **réservations** de ressources |
| 4 | Les deux options et le titre du vote viennent de la ConfigMap `vote-config` |
| 5 | Les identifiants de la base viennent du Secret `db-credentials` |
| 6 | La page de vote est accessible par la **porte d'entrée**, l'Ingress |
| 7 | Un vote apparaît sur la page des **résultats** |

Tout ce qu'il faut existe déjà : les neuf fichiers de `~/byte-layers/kubernetes/l7/`. Aucun n'est à modifier. Pour envoyer un fichier dans un namespace, vous connaissez l'option : `-n prod`, à placer avant ou après `-f`, peu importe.

Avant de vous lancer, réfléchissez à l'**ordre** : quels objets les autres attendent-ils ?

```quiz
prompt: "Vous envoyez d'abord `vote.deploy.yml` dans `prod`, avant tout le reste. Dans quel état trouvez-vous les pods de `vote` ?"
answers:
  - "`Running` : un pod n'a besoin que de son image pour démarrer"
  - "`CreateContainerConfigError` : ils attendent la ConfigMap `vote-config`, qui n'existe pas encore dans `prod`"
  - "`Pending` : il n'y a pas encore de Service pour eux"
correct: 2
explanation: |
  Les pods de `vote` lisent la ConfigMap, ceux de `db` lisent le Secret. S'ils démarrent avant, ils restent en `CreateContainerConfigError` tant que ces objets n'existent pas. On crée donc d'abord le namespace, puis la ConfigMap et le Secret, et seulement ensuite les Deployments.
```

Les cinq sections qui suivent prennent les objets dans cet ordre : le namespace, les réglages, les quatre morceaux que le visiteur ne voit pas sur la page de vote, puis `vote`, puis la porte d'entrée.

> 🧠 **L'ordre est-il vraiment obligatoire ?** Pas tout à fait : grâce à la réconciliation, un pod qui démarre trop tôt finit par démarrer correctement une fois ce qu'il attend créé. Mais un bon ordre évite des erreurs transitoires et rend un déploiement **prévisible**. En entreprise, des outils comme **Helm** ou **Kustomize** regroupent tous ces fichiers en un seul paquet, installé d'une seule commande.

## Créer le namespace prod

L'exigence 1 demande que toute l'application vive dans un namespace nommé `prod`. Créez-le.

**Critère :** un namespace s'appelle exactement `prod`.

```hint
La commande est la même que pour `vote-app`, dans le cours sur les namespaces : `kubectl create namespace`, suivie du nom.
```

```verify
PHASE=$(kubectl get namespace prod -o jsonpath='{.status.phase}' 2>/dev/null)
if [ -z "$PHASE" ]; then
  echo "ÉCHEC : aucun namespace ne s'appelle prod."
  echo "  Créez-le, puis relisez la liste : kubectl get namespaces"
  exit 1
fi
if [ "$PHASE" != "Active" ]; then
  echo "ÉCHEC : le namespace prod existe, mais il est dans l'état « $PHASE » : il est en cours de suppression."
  echo "  Attendez qu'il ait disparu de kubectl get namespaces, puis recréez-le."
  exit 1
fi
echo "RÉUSSI : le namespace prod existe (exigence 1)."
```

````solution
```bash
kubectl create namespace prod
```
````

## Envoyer la ConfigMap et le Secret

Les exigences 4 et 5 demandent que les réglages de `vote` viennent de la ConfigMap `vote-config`, et les identifiants de la base du Secret `db-credentials`. Envoyer ces objets ne remplit pas encore ces exigences, mais les prépare : les pods les liront à leur démarrage. Envoyez ces deux objets dans `prod` avant tout Deployment.

**Critères :**

- la ConfigMap `vote-config` existe dans `prod`, avec ses 3 réglages : `OPTION_A`, `OPTION_B` et `VOTE_TITLE` ;
- le Secret `db-credentials` existe dans `prod`, avec ses 2 valeurs : `POSTGRES_USER` et `POSTGRES_PASSWORD`.

```hint
Chaque commande ressemble à `kubectl apply -n prod -f <fichier>`. Sans `-n prod`, l'objet part dans `default`.
```

```hint
Vous pouvez donner plusieurs fichiers d'un coup, en répétant `-f` devant chaque chemin : `kubectl apply -n prod -f <premier fichier> -f <second fichier>`. Les deux fichiers sont `db.secret.yml` et `vote.config.yml`, dans `~/byte-layers/kubernetes/l7/`.
```

```verify
if ! kubectl get namespace prod >/dev/null 2>&1; then
  echo "ÉCHEC : le namespace prod n'existe pas. Reprenez la section « Créer le namespace prod »."
  exit 1
fi
if ! kubectl get configmap vote-config -n prod >/dev/null 2>&1; then
  echo "ÉCHEC : il n'y a pas de ConfigMap vote-config dans prod."
  if kubectl get configmap vote-config -n default >/dev/null 2>&1; then
    echo "  Il y en a une dans default : la commande a été lancée sans -n prod. Supprimez-la avec kubectl delete -f suivi du même fichier."
  fi
  echo "  Envoyez le fichier vote.config.yml dans le namespace prod."
  exit 1
fi
for k in OPTION_A OPTION_B VOTE_TITLE; do
  V=$(kubectl get configmap vote-config -n prod -o jsonpath="{.data.$k}")
  if [ -z "$V" ]; then
    echo "ÉCHEC : la ConfigMap vote-config de prod n'a pas de réglage $k."
    echo "  Renvoyez le fichier vote.config.yml, sans le modifier."
    exit 1
  fi
done
if ! kubectl get secret db-credentials -n prod >/dev/null 2>&1; then
  echo "ÉCHEC : la ConfigMap est là, mais il n'y a pas de Secret db-credentials dans prod."
  echo "  Envoyez le fichier db.secret.yml dans le namespace prod."
  exit 1
fi
for k in POSTGRES_USER POSTGRES_PASSWORD; do
  V=$(kubectl get secret db-credentials -n prod -o jsonpath="{.data.$k}")
  if [ -z "$V" ]; then
    echo "ÉCHEC : le Secret db-credentials de prod n'a pas de valeur $k."
    echo "  Renvoyez le fichier db.secret.yml, sans le modifier."
    exit 1
  fi
done
echo "RÉUSSI : vote-config (3 réglages) et db-credentials (2 valeurs) sont dans prod, prêts pour les exigences 4 et 5."
```

````solution
```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/db.secret.yml -f ~/byte-layers/kubernetes/l7/vote.config.yml
```
````

## Déployer redis, db, worker et result

L'exigence 2 demande que les 5 morceaux tournent. Commencez par les quatre que le visiteur ne voit pas directement sur la page de vote : `redis`, `db`, `worker` et `result`. Quand vos commandes sont passées, suivez le démarrage avec `kubectl get pods -n prod`. `worker` et `result` ont besoin de la base : s'ils démarrent avant elle, ils peuvent redémarrer une fois, le temps de la trouver.

**Critères :**

- les Deployments `redis`, `db`, `worker` et `result` existent dans `prod`, et chacun a son pod prêt ;
- les Services `redis`, `db` et `result` existent dans `prod` ;
- `db` lit ses identifiants dans le Secret `db-credentials` (exigence 5).

```hint
Chaque morceau a son fichier, qui contient le Deployment et, sauf pour `worker`, le Service : `redis.yml`, `db.yml`, `worker.yml`, `result.yml`. Quatre fichiers, donc quatre `-f`, ou quatre commandes.
```

```hint
`kubectl rollout status deployment/db -n prod` suit le démarrage d'un morceau et rend la main quand son pod est prêt.
```

```verify
if ! kubectl get namespace prod >/dev/null 2>&1; then
  echo "ÉCHEC : le namespace prod n'existe pas. Reprenez la section « Créer le namespace prod »."
  exit 1
fi
for d in redis db worker result; do
  if ! kubectl get deployment $d -n prod >/dev/null 2>&1; then
    echo "ÉCHEC : il n'y a pas de Deployment $d dans prod."
    if kubectl get deployment $d -n default >/dev/null 2>&1; then
      echo "  Il y en a un dans default : la commande a été lancée sans -n prod. Supprimez-le avec kubectl delete -f suivi du même fichier."
    fi
    echo "  Envoyez le fichier $d.yml dans le namespace prod."
    exit 1
  fi
done
for s in redis db result; do
  if ! kubectl get service $s -n prod >/dev/null 2>&1; then
    echo "ÉCHEC : il n'y a pas de Service $s dans prod. Renvoyez $s.yml, sans le modifier : il décrit aussi le Service."
    exit 1
  fi
done
case "$(kubectl get deployment db -n prod -o jsonpath='{.spec.template.spec.containers[*].envFrom}{.spec.template.spec.containers[*].env}')" in
  *db-credentials*) : ;;
  *) echo "ÉCHEC : le Deployment db de prod ne lit pas le Secret db-credentials."
     echo "  Envoyez le fichier db.yml de ~/byte-layers/kubernetes/l7/, sans le modifier."
     exit 1 ;;
esac
END=$((SECONDS + 45))
for d in redis db worker result; do
  LEFT=$((END - SECONDS)); [ "$LEFT" -lt 2 ] && LEFT=2
  if ! kubectl rollout status deployment/$d -n prod --timeout=${LEFT}s >/dev/null 2>&1; then
    echo "ÉCHEC : le pod de $d n'est pas prêt."
    echo "  Regardez son état : kubectl get pods -n prod"
    echo "  S'il démarre encore, réessayez dans quelques secondes. Si le pod de db est en CreateContainerConfigError, il attend le Secret : reprenez la section précédente."
    exit 1
  fi
done
echo "RÉUSSI : redis, db, worker et result tournent dans prod, et db lit le Secret."
```

````solution
```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/redis.yml -f ~/byte-layers/kubernetes/l7/db.yml
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/worker.yml -f ~/byte-layers/kubernetes/l7/result.yml
```
````

## Déployer vote en version 3.0, avec son Service

Il manque le cinquième morceau, celui de l'exigence 3 : `vote`, dans son dernier état. Déployez-le dans `prod`, avec son Service.

**Critères :**

- le Deployment `vote` de `prod` demande l'image `ghcr.io/bngams/kube-vote:3.0` et **3 réplicas**, tous prêts ;
- ses pods ont une sonde `readinessProbe` et une sonde `livenessProbe`, et réservent du processeur et de la mémoire ;
- ils lisent leurs réglages dans la ConfigMap `vote-config` (exigence 4) ;
- le Service `vote` de `prod` mène à ces pods.

```hint
Deux fichiers cette fois : `vote.deploy.yml` et `vote.svc.yml`. Ils disent déjà tout ce que demande l'exigence 3 : vous n'avez qu'à les envoyer au bon endroit.
```

```hint
Si les pods restent en `CreateContainerConfigError`, la ConfigMap `vote-config` manque dans `prod` : `kubectl get configmaps -n prod` le montre.
```

```verify
D=$(kubectl get deployment vote -n prod -o json 2>/dev/null)
if [ -z "$D" ]; then
  echo "ÉCHEC : il n'y a pas de Deployment vote dans prod."
  if kubectl get deployment vote -n default >/dev/null 2>&1; then
    echo "  Il y en a un dans default : la commande a été lancée sans -n prod. Supprimez-le avec kubectl delete -f suivi du même fichier."
  fi
  echo "  Envoyez le fichier vote.deploy.yml dans le namespace prod."
  exit 1
fi
IMG=$(kubectl get deployment vote -n prod -o jsonpath='{.spec.template.spec.containers[0].image}')
case "$IMG" in
  *kube-vote:3.0) : ;;
  *) echo "ÉCHEC : vote demande l'image $IMG, pas ghcr.io/bngams/kube-vote:3.0 (exigence 3)."
     echo "  Envoyez le fichier vote.deploy.yml de ~/byte-layers/kubernetes/l7/, sans le modifier."
     exit 1 ;;
esac
R=$(kubectl get deployment vote -n prod -o jsonpath='{.spec.replicas}')
if [ "$R" != "3" ]; then
  echo "ÉCHEC : vote demande $R réplica(s), le cahier des charges en veut 3."
  exit 1
fi
for p in readinessProbe livenessProbe; do
  if [ -z "$(kubectl get deployment vote -n prod -o jsonpath="{.spec.template.spec.containers[0].$p}")" ]; then
    echo "ÉCHEC : le Deployment vote de prod n'a pas de sonde $p (exigence 3)."
    exit 1
  fi
done
for r in cpu memory; do
  if [ -z "$(kubectl get deployment vote -n prod -o jsonpath="{.spec.template.spec.containers[0].resources.requests.$r}")" ]; then
    echo "ÉCHEC : le Deployment vote de prod ne réserve pas de ressource « $r » (exigence 3)."
    exit 1
  fi
done
case "$(kubectl get deployment vote -n prod -o jsonpath='{.spec.template.spec.containers[0].envFrom}{.spec.template.spec.containers[0].env}')" in
  *vote-config*) : ;;
  *) echo "ÉCHEC : le Deployment vote de prod ne lit pas la ConfigMap vote-config (exigence 4)."
     exit 1 ;;
esac
if ! kubectl rollout status deployment/vote -n prod --timeout=45s >/dev/null 2>&1; then
  echo "ÉCHEC : les 3 pods de vote ne sont pas tous prêts."
  echo "  Regardez leur état : kubectl get pods -n prod"
  echo "  S'ils démarrent encore, réessayez dans quelques secondes. S'ils sont en CreateContainerConfigError, la ConfigMap manque dans prod."
  exit 1
fi
if ! kubectl get service vote -n prod >/dev/null 2>&1; then
  echo "ÉCHEC : vote tourne, mais il n'y a pas de Service vote dans prod."
  echo "  Envoyez le fichier vote.svc.yml dans le namespace prod."
  exit 1
fi
OUT=""
for i in 1 2 3 4 5; do
  OUT=$(kubectl get --raw "/api/v1/namespaces/prod/services/vote:80/proxy/healthz" 2>/dev/null)
  case "$OUT" in *'"version":"3.0"'*) break ;; esac
  sleep 2
done
case "$OUT" in
  *'"version":"3.0"'*) : ;;
  *) echo "ÉCHEC : le Service vote de prod ne mène pas à un pod de vote en version 3.0."
     echo "  Comparez son sélecteur et les étiquettes des pods : kubectl describe service vote -n prod"
     exit 1 ;;
esac
echo "RÉUSSI : vote tourne en 3.0 sur 3 réplicas, avec ses sondes, ses réservations, ses réglages lus dans vote-config et son Service (exigences 3 et 4)."
```

````solution
```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/vote.deploy.yml -f ~/byte-layers/kubernetes/l7/vote.svc.yml
```
````

## Ouvrir la porte d'entrée

L'exigence 6 demande que la page de vote soit accessible par la porte d'entrée. Il reste donc un dernier objet à envoyer dans `prod` : l'Ingress.

**Critères :**

- un Ingress existe dans `prod`, et sa règle envoie les visites vers le Service `vote`, sur le port 80 ;
- une visite reçue par le contrôleur d'Ingress arrive à un pod de `vote` du namespace `prod`.

```hint
Le fichier s'appelle `vote.ingress.yml`, et il s'envoie comme les autres.
```

```hint
Si `kubectl apply` répond `failed calling webhook`, le contrôleur d'Ingress n'est pas prêt : reprenez la section « Vérifier que le cluster a un contrôleur d'Ingress ». S'il répond que le chemin `/` est déjà défini dans un autre Ingress, c'est qu'il en reste un ailleurs : `kubectl get ingress -A` le montre.
```

```verify
if ! kubectl get ingress vote -n prod >/dev/null 2>&1; then
  echo "ÉCHEC : il n'y a pas d'Ingress vote dans prod."
  echo "  Envoyez le fichier vote.ingress.yml dans le namespace prod, et lisez la réponse de la commande."
  exit 1
fi
SVC=$(kubectl get ingress vote -n prod -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}')
PORT=$(kubectl get ingress vote -n prod -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.port.number}')
if [ "$SVC" != "vote" ] || [ "$PORT" != "80" ]; then
  echo "ÉCHEC : la règle envoie les visites vers le Service « ${SVC:-vide} », port ${PORT:-vide} : il faut le Service vote, port 80."
  echo "  Envoyez le fichier vote.ingress.yml de ~/byte-layers/kubernetes/l7/, sans le modifier."
  exit 1
fi
if kubectl get service ingress-nginx-controller -n ingress-nginx >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/ingress-nginx/services/ingress-nginx-controller:80/proxy/"
elif kubectl get service traefik -n kube-system >/dev/null 2>&1; then
  DOOR="/api/v1/namespaces/kube-system/services/traefik:80/proxy/"
else
  echo "ÉCHEC : aucun contrôleur d'Ingress n'a été trouvé dans le cluster."
  echo "  Sur votre poste, reprenez la section « Vérifier que le cluster a un contrôleur d'Ingress »."
  exit 1
fi
PODS=$(kubectl get pods -n prod -l app=vote -o jsonpath='{.items[*].metadata.name}')
OK=0
for i in 1 2 3 4 5 6 7 8 9 10; do
  HOST=$(kubectl get --raw "$DOOR" 2>/dev/null | sed -n 's/.*Servi par : <strong>\([^<]*\)<.*/\1/p')
  if [ -n "$HOST" ]; then
    case " $PODS " in *" $HOST "*) OK=1; break ;; esac
  fi
  sleep 3
done
if [ "$OK" != 1 ]; then
  echo "ÉCHEC : l'Ingress vote existe, mais une visite envoyée au contrôleur n'arrive pas à un pod de vote du namespace prod."
  if [ -n "$HOST" ]; then
    echo "  C'est le pod « $HOST » qui a répondu : un autre Ingress capte les visites. Cherchez-le avec kubectl get ingress -A."
  else
    echo "  Vérifiez que vote tourne (kubectl get pods -n prod) et que son Service existe (kubectl get services -n prod)."
  fi
  exit 1
fi
echo "RÉUSSI : une visite reçue par le contrôleur d'Ingress arrive au pod $HOST, dans prod (exigence 6)."
```

````solution
```bash
kubectl apply -n prod -f ~/byte-layers/kubernetes/l7/vote.ingress.yml
```
````

# La recette : vérifier soi-même

Un déploiement n'est terminé que lorsqu'on a **vérifié** qu'il fonctionne. Les boutons de vérification vous ont accompagné jusqu'ici ; en production, c'est vous qui passez le cahier des charges en revue, exigence par exigence.

## Passer les objets en revue

Les exigences 1 à 5 portent sur des objets. Une seule commande en montre l'essentiel : à la place d'un type d'objet, `kubectl get` accepte le mot `all`, qui demande d'un coup les pods, les Services, les Deployments et les ReplicaSets d'un namespace. Avant de la lancer, faites une prévision : combien de pods, de Services, de Deployments attendez-vous ?

```exec
kubectl get all -n prod
```

Vous devez voir 7 pods `Running` (3 `vote`, 1 de chacun des autres), 4 Services (`db`, `redis`, `result`, `vote`), 5 Deployments tous complets (`1/1`, ou `3/3` pour `vote`) et leurs 5 ReplicaSets. Chez vous, les noms des pods, les adresses et les âges diffèrent :

```text
NAME                          READY   STATUS    RESTARTS   AGE
pod/db-78fb95db9-nsdtx        1/1     Running   0          6s
pod/redis-5b95d4d69f-bxccc    1/1     Running   0          6s
pod/result-656b779d76-4nzwr   1/1     Running   0          6s
pod/vote-68596c5d8c-8jkcd     1/1     Running   0          6s
pod/vote-68596c5d8c-btmnw     1/1     Running   0          6s
pod/vote-68596c5d8c-lxrhg     1/1     Running   0          6s
pod/worker-7b7bc56f88-dq4dh   1/1     Running   0          6s

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/db       ClusterIP   10.104.95.219   <none>        5432/TCP   6s
service/redis    ClusterIP   10.96.205.229   <none>        6379/TCP   6s
service/result   ClusterIP   10.97.237.244   <none>        80/TCP     6s
service/vote     ClusterIP   10.107.47.56    <none>        80/TCP     6s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/db       1/1     1            1           6s
deployment.apps/redis    1/1     1            1           6s
deployment.apps/result   1/1     1            1           6s
deployment.apps/vote     3/3     3            3           6s
deployment.apps/worker   1/1     1            1           6s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/db-78fb95db9        1         1         1       6s
replicaset.apps/redis-5b95d4d69f    1         1         1       6s
replicaset.apps/result-656b779d76   1         1         1       6s
replicaset.apps/vote-68596c5d8c     3         3         3       6s
replicaset.apps/worker-7b7bc56f88   1         1         1       6s
```

La ConfigMap, le Secret et l'Ingress n'apparaissent pas dans `get all`. Demandez-les à part, en donnant les trois types séparés par des virgules :

```exec
kubectl get configmaps,secrets,ingress -n prod
```

La ConfigMap `vote-config` doit compter 3 réglages dans la colonne `DATA` (les deux options et `VOTE_TITLE`), et le Secret `db-credentials` 2. Ignorez `kube-root-ca.crt`, créée automatiquement dans chaque namespace. La colonne `ADDRESS` de l'Ingress peut rester vide quelques instants, et sur un poste hébergé la colonne `CLASS` affiche `traefik`.

```text
NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      6s
configmap/vote-config        3      6s

NAME                    TYPE     DATA   AGE
secret/db-credentials   Opaque   2      6s

NAME                             CLASS   HOSTS   ADDRESS        PORTS   AGE
ingress.networking.k8s.io/vote   nginx   *       192.168.49.2   80      6s
```

Ces listes disent que les objets existent, pas ce qu'ils contiennent. Pour l'exigence 3, demandez à `vote` de se décrire :

```exec
kubectl describe deployment vote -n prod
```

Retrouvez dans la réponse les lignes `Replicas` (3 demandés, 3 disponibles), `Image` (la version `3.0`), `Requests` (les réservations), `Liveness` et `Readiness` (les deux sondes), et `Environment Variables from`, qui cite `vote-config` : c'est l'exigence 4. La même commande sur `db` cite `db-credentials`, pour l'exigence 5.

```ack
J'ai retrouvé les 7 pods, les 4 Services, les 5 Deployments, la ConfigMap, le Secret, l'Ingress, et le détail de vote
```

## Entrer par la porte d'entrée

L'exigence 6 se vérifie comme un visiteur : en entrant par la porte. Sur un poste hébergé, elle est déjà ouverte sur le port 8000 du poste : ne lancez pas la commande du tunnel, et écrivez `8000` à la place de `28670` dans la suite.

Sur votre poste, ouvrez un tunnel vers le **contrôleur** d'Ingress, comme dans le cours sur l'Ingress : le port 28670 de votre machine mène au port 80 du Service `ingress-nginx-controller`.

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

Le terminal affiche un numéro de tâche, du type `[1] 12345`, et vous rend la main. Demandez la page de vote par cette porte, depuis le terminal, en n'en gardant que la ligne qui annonce la version. Si rien ne s'affiche, le tunnel n'a pas fini de s'ouvrir : relancez `curl` une seconde plus tard. Sur un poste hébergé, tapez la commande à la main, avec `8000` à la place de `28670`.

```exec
curl -s http://localhost:28670/ | grep version
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `curl -s http://localhost:28670/` | télécharge la page de vote ; `-s` retire l'affichage de la progression |
> | `\|` | passe la page à la commande de droite, au lieu de l'afficher |
> | `grep version` | ne garde que les lignes qui contiennent le mot `version` |

Un pod de `vote` a répondu, et la page annonce sa version :

```text
    <div id="version">v3.0 <span>nouveau design</span></div>
```

Ouvrez ensuite `http://localhost:28670` dans votre navigateur, ou passez par l'onglet **PORTS** de VS Code sur un poste hébergé. Le badge doit indiquer **v3.0**, avec le titre « Votre destination de vacances ? » en haut de la page, et les deux options de la ConfigMap, « Montagne » et « Mer ».

```ack
La page de vote s'affiche par la porte d'entrée, en v3.0, avec son titre
```

## Voter et voir le résultat bouger

Il reste l'exigence 7, celle qui prouve que les cinq morceaux coopèrent : un vote doit apparaître sur la page des résultats. Ouvrez un second tunnel, vers le Service `result`. Attention, il est maintenant dans `prod` :

```exec
kubectl port-forward -n prod service/result 28671:80 > /dev/null 2>&1 &
```

Ouvrez `http://localhost:28671` dans votre navigateur, à côté de la page de vote, ou passez par l'onglet **PORTS** de VS Code sur un poste hébergé. Puis **votez** : le résultat doit bouger.

> ⚠️ **La page des résultats ne répond pas, ou ne bouge pas ?** Relancez la commande du tunnel sans sa fin, donc `kubectl port-forward -n prod service/result 28671:80` : elle affiche alors son message d'erreur, et `Ctrl+C` l'arrête. Une fois la cause corrigée, relancez la commande complète.
>
> - **`services "result" not found`** : vous avez sans doute oublié `-n prod`. Sans lui, `kubectl` cherche dans `default`.
> - **`error: timed out waiting for the condition`** : le Service `result` ne trouve aucun pod à servir. Vérifiez que `result` tourne bien dans `prod`, avec `kubectl get pods -n prod`.
> - **Les résultats ne bougent pas alors que tout est `Running`** : `worker` ou `result` ont peut-être démarré avant la base. Relancez-les avec `kubectl rollout restart deployment worker result -n prod`, puis relancez le tunnel vers `result`.

**Critère :** au moins un vote est enregistré dans la base de données de `prod`.

Le vote est aussi dans la base. Comptez les votes par option avec `psql`, l'outil en ligne de commande de PostgreSQL, lancé dans le pod de `db` par `kubectl exec`, comme dans le lab [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app) :

```exec
kubectl exec -n prod deployment/db -- psql -U postgres -c "select vote, count(*) from votes group by vote"
```

L'option « Montagne » est notée `a` dans la base, et « Mer » `b` :

```text
 vote | count 
------+-------
 a    |     1
(1 row)
```

```hint
Le vote se fait dans le navigateur, sur la page `http://localhost:28670` (le port 8000 sur un poste hébergé). Si elle ne s'affiche pas, relancez la commande du tunnel de la section « Entrer par la porte d'entrée ».
```

```verify
for d in vote redis db worker result; do
  if ! kubectl get deployment $d -n prod >/dev/null 2>&1; then
    echo "ÉCHEC : il n'y a pas de Deployment $d dans prod : reprenez l'étape « La mission : mettre en production »."
    exit 1
  fi
done
N=0
for i in 1 2 3 4 5; do
  N=$(kubectl exec -n prod deployment/db -- psql -U postgres -tAc "select count(*) from votes" 2>/dev/null)
  case "$N" in ''|0) sleep 2 ;; *) break ;; esac
done
case "$N" in
  ''|0) echo "ÉCHEC : aucun vote n'est encore arrivé dans la base de prod."
        echo "  Votez sur la page de vote, http://localhost:28670 sur votre poste, puis réessayez."
        echo "  Si le vote ne part pas, lisez : kubectl logs -n prod deployment/worker"
        exit 1 ;;
esac
echo "RÉUSSI : $N vote(s) dans la base de prod. Votre clic a traversé vote, redis, worker et db. La page des résultats, c'est vous qui l'avez vue bouger (exigence 7)."
```

````solution
Le vote se fait d'un clic dans le navigateur. Cette commande fait la même chose sans navigateur : elle envoie à la page de vote, par la porte d'entrée, le formulaire que le bouton « Montagne » enverrait, puis elle laisse 3 secondes à `worker` pour porter le vote dans la base. Sur un poste hébergé, remplacez `28670` par `8000`.

```bash
curl -s -o /dev/null -d "vote=a" http://localhost:28670/
sleep 3
```
````

## Fermer les deux tunnels

Les 7 exigences sont remplies : votre application est en production. Les deux tunnels tournent encore en arrière-plan. `pkill -f` arrête les programmes dont la ligne de commande contient le texte donné. Arrêtez d'abord celui de la porte d'entrée, si vous l'avez ouvert (sur un poste hébergé, il n'existe pas) :

```exec
pkill -f "port-forward -n ingress-nginx service/ingress-nginx-controller 28670"
```

Arrêtez ensuite celui de la page des résultats :

```exec
pkill -f "port-forward -n prod service/result 28671"
```

Les deux pages ne répondent plus, et c'est normal. L'application, elle, continue de tourner dans `prod` : laissez-la en place si vous enchaînez avec la [Chasse aux pannes](https://byte-layers.com/fr/challenges/k8s-troubleshooting-challenge). Sinon, `kubectl delete namespace prod` supprime toute l'application en une commande.

## Récap

- Un déploiement complet, c'est un **ensemble de fichiers** appliqués dans un ordre réfléchi, dans un namespace dédié.
- Ce que les pods lisent à leur démarrage, la ConfigMap et le Secret, se crée avant eux.
- `-n prod` dit explicitement où l'on agit : sans lui, la commande part dans le namespace courant.
- Un déploiement n'est fini qu'une fois **vérifié** contre le cahier des charges : les objets, la porte d'entrée, puis un vrai vote qui traverse toute l'application.

Les neuf fichiers se trouvent dans le dossier `solution/` de ce lab : [`vote.config.yml`](../solution/vote.config.yml), [`db.secret.yml`](../solution/db.secret.yml), [`vote.deploy.yml`](../solution/vote.deploy.yml), [`vote.svc.yml`](../solution/vote.svc.yml), [`vote.ingress.yml`](../solution/vote.ingress.yml), [`redis.yml`](../solution/redis.yml), [`db.yml`](../solution/db.yml), [`worker.yml`](../solution/worker.yml) et [`result.yml`](../solution/result.yml).

> 📖 [Les namespaces, documentation officielle](https://kubernetes.io/fr/docs/concepts/overview/working-with-objects/namespaces/) · [Gérer des objets avec des fichiers et `kubectl apply`](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/) (en anglais)

## Pour aller plus loin

- **Tout en un paquet.** [Helm](https://helm.sh/fr/docs/) et [Kustomize](https://kustomize.io/) (ce second site est en anglais) regroupent les fichiers d'une application et leurs variantes par environnement : c'est ce qu'une équipe utilise quand les neuf fichiers deviennent cinquante.
- **Fermer le réseau de la production.** Les règles réseau écrites dans [Fermer le réseau avec des NetworkPolicies](https://byte-layers.com/fr/skills/k8s-network-policies) s'appliquent à un namespace : en production, on les enverrait aussi dans `prod`.
