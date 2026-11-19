---
title: "Mettre à jour et réparer, sans guide"
description: "Livrer une nouvelle version sans coupure, trouver pourquoi la suivante plante, revenir en arrière, poser des sondes et livrer la version corrigée, à partir d'un cahier des charges."
tags:
  - challenge
  - kubernetes
  - containers
difficulty: beginner
duration: 45
introTitle: "Le défi"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Ce challenge prouve que vous savez mettre à jour une application sur Kubernetes, et la réparer quand la mise à jour se passe mal. Hormis la mise en place du point de départ, il n'y a **aucune commande à recopier** : chaque tâche donne un objectif et des critères, puis un bouton vérifie l'état réel du cluster. Toute suite de commandes qui produit le bon résultat est acceptée.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : ce qu'enseignent [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update) et [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes), et ranger un réglage dans une ConfigMap, vu dans [Sortir les réglages de l'image avec une ConfigMap](https://byte-layers.com/fr/skills/k8s-configmap).
- **Ce qu'il faut avoir** : un cluster `layers-k8s` auquel `kubectl` répond, et un éditeur de texte. Le challenge se joue dans le namespace courant de `kubectl` : `vote-app` si vous suivez le parcours.
- **Une application à vous** : le challenge ne touche pas à l'application de vote. Il déploie sa propre application, `l6-ballot`, à partir de la même image que `vote`. Elle n'a besoin de rien d'autre pour démarrer.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l6/challenge/`. La dernière tâche le supprime, avec tout ce que le challenge a créé dans le cluster.
- **Les noms comptent** : les vérifications cherchent les noms exacts, tous préfixés par `l6-`. Rien d'autre n'est inspecté ni supprimé.
- **Le fichier fait foi** : chaque changement du Deployment passe par le fichier `ballot.deploy.yml`, puis par `kubectl apply`. Les vérifications regardent le cluster, et certaines regardent aussi votre fichier et le dernier fichier appliqué.
- **L'ordre compte** : chaque tâche part de l'état laissé par la précédente.

# Tâche 1 : livrer la v2

## Mettre en place le point de départ

Cette section est la seule à donner des commandes : elle installe l'application sur laquelle porte le challenge. Si vous rejouez ce challenge, supprimez d'abord ce qu'un passage précédent a pu laisser : le Deployment `l6-ballot` et la ConfigMap `l6-ballot-config`. `--ignore-not-found` évite une erreur quand il n'y a rien à supprimer, et la commande ne répond alors rien :

```exec
kubectl delete deployment/l6-ballot configmap/l6-ballot-config --ignore-not-found
```

Créez ensuite le dossier de travail :

```exec
mkdir -p ~/byte-layers/kubernetes/l6/challenge
```

Écrivez ensuite le fichier de départ, `ballot.deploy.yml` : un Deployment `l6-ballot` de 2 réplicas, en version `1.0`. La commande crée le fichier et y recopie tout ce qui suit, jusqu'à la ligne `EOF` :

```exec
cat > ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l6-ballot
  labels:
    app: l6-ballot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: l6-ballot
  template:
    metadata:
      labels:
        app: l6-ballot
    spec:
      containers:
        - name: ballot
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'on lui donne en entrée |
> | `> ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml` | envoie cette copie dans le fichier, au lieu de l'afficher ; le fichier est créé, ou remplacé s'il existe |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles |
> | `EOF` (dernière ligne) | marque la fin de ces lignes ; elle n'est pas écrite dans le fichier |

Appliquez ce fichier :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml
```

Attendez que les 2 pods soient prêts :

```exec
kubectl rollout status deployment/l6-ballot --timeout=120s
```

Quand la commande répond `successfully rolled out`, le point de départ est en place.

## Passer l6-ballot en version 2.0

L'équipe de développement livre la version `2.0` de l'image. Déployez-la, sans interrompre l'application.

**Critères :**

- le Deployment `l6-ballot` demande l'image `ghcr.io/bngams/kube-vote:2.0` ;
- il demande toujours 2 réplicas, et la mise à jour est terminée : ses 2 pods sont disponibles, en `2.0` ;
- c'est votre fichier `ballot.deploy.yml` qui dit `2.0`, et c'est lui qui a été appliqué.

```hint
Une nouvelle version, c'est une nouvelle image dans le modèle de pod. Une seule ligne du fichier change.
```

```hint
Après avoir enregistré le fichier, appliquez-le, puis suivez la mise à jour avec `kubectl rollout status deployment/l6-ballot`.
```

```verify
D=l6-ballot
F="$HOME/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml"
IMG=$(kubectl get deployment "$D" -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "ÉCHEC : aucun Deployment ne s'appelle $D dans le namespace courant."
  echo "  Lancez les commandes de la section « Mettre en place le point de départ »."
  exit 1
fi
case "$IMG" in
  *kube-vote:2.0) : ;;
  *) echo "ÉCHEC : $D demande l'image $IMG, pas ghcr.io/bngams/kube-vote:2.0."
     exit 1 ;;
esac
LAST=$(kubectl get deployment "$D" -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' 2>/dev/null)
if ! grep -Eq '^[[:space:]]*image:[[:space:]]*"?ghcr\.io/bngams/kube-vote:2\.0' "$F" 2>/dev/null; then
  echo "ÉCHEC : le cluster est en 2.0, mais votre fichier ballot.deploy.yml ne dit pas 2.0."
  echo "  Le fichier fait foi : changez-y l'image, puis appliquez-le."
  exit 1
fi
case "$LAST" in
  *kube-vote:2.0*) : ;;
  *) echo "ÉCHEC : le dernier fichier appliqué ne demandait pas la 2.0."
     echo "  Appliquez ballot.deploy.yml avec kubectl apply -f."
     exit 1 ;;
esac
N=$(kubectl get deployment "$D" -o jsonpath='{.spec.replicas}' 2>/dev/null)
if [ "$N" != "2" ]; then
  echo "ÉCHEC : $D demande $N réplica(s), alors que le cahier des charges en veut 2."
  echo "  Remettez replicas: 2 dans ballot.deploy.yml, puis appliquez le fichier."
  exit 1
fi
if ! kubectl rollout status deployment/"$D" --timeout=45s >/dev/null 2>&1; then
  echo "ÉCHEC : la mise à jour vers la 2.0 n'est pas terminée."
  echo "  Regardez les pods : kubectl get pods -l app=$D"
  exit 1
fi
echo "RÉUSSI : $D tourne en 2.0, et votre fichier dit la même chose."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l6-ballot
  labels:
    app: l6-ballot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: l6-ballot
  template:
    metadata:
      labels:
        app: l6-ballot
    spec:
      containers:
        - name: ballot
          image: ghcr.io/bngams/kube-vote:2.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml
kubectl rollout status deployment/l6-ballot --timeout=50s
```
````

# Tâche 2 : trouver pourquoi la v3 plante

## Déployer la 3.0 et noter la cause de la panne

L'équipe livre maintenant la version `3.0`. Déployez-la : elle plante au démarrage. Trouvez pourquoi, comme le ferait une équipe d'exploitation, et gardez une trace de la cause dans le fichier `~/byte-layers/kubernetes/l6/challenge/cause.txt`, en y envoyant la ligne utile avec `>`, ou en l'y collant avec votre éditeur.

**Critères :**

- la version `3.0` de `l6-ballot` a été déployée ;
- le fichier `~/byte-layers/kubernetes/l6/challenge/cause.txt` existe ;
- il contient le nom du réglage que l'application réclame et ne trouve pas, en majuscules, tel qu'il apparaît dans ses logs.

```hint
Face à un pod en échec, la démarche est toujours la même : décrire le pod, puis lire ses logs. La raison, c'est l'application qui la donne.
```

```hint
Les logs font une centaine de lignes, et la fin n'est pas la plus parlante. L'erreur utile est une erreur Python : cherchez le mot `Error` plus haut. Si les logs sont vides, l'option `--previous` affiche ceux du conteneur qui vient de planter.
```

```hint
`kubectl logs <nom-du-pod> | grep KeyError` ne garde que la ligne utile. Envoyez-la dans le fichier avec `>`.
```

```verify
D=l6-ballot
REPORT="$HOME/byte-layers/kubernetes/l6/challenge/cause.txt"
if ! kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment $D n'existe pas. Faites d'abord la tâche 1."
  exit 1
fi
if ! kubectl get replicasets -l app="$D" -o jsonpath='{range .items[*]}{.spec.template.spec.containers[0].image}{"\n"}{end}' 2>/dev/null | grep -q 'kube-vote:3.0$'; then
  echo "ÉCHEC : la version 3.0 de $D n'a jamais été déployée."
  echo "  Passez l'image en 3.0 dans ballot.deploy.yml, puis appliquez le fichier."
  exit 1
fi
if [ ! -f "$REPORT" ]; then
  echo "ÉCHEC : le fichier $REPORT n'existe pas."
  echo "  Notez-y la cause de la panne, lue dans les logs d'un pod de la v3."
  exit 1
fi
if ! grep -q 'VOTE_TITLE' "$REPORT"; then
  echo "ÉCHEC : $REPORT ne contient pas le nom de ce que l'application réclame."
  echo "  Ce nom est écrit en majuscules dans ses logs, sur une ligne qui contient le mot Error."
  exit 1
fi
echo "RÉUSSI : la cause est trouvée. La v3 réclame un réglage VOTE_TITLE que personne ne lui a donné."
```

````solution
Le fichier passe en `3.0`, puis une boucle attend que le pod le plus récent ait écrit son erreur dans ses logs, et envoie la ligne utile dans `cause.txt` :

```bash
cat > ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l6-ballot
  labels:
    app: l6-ballot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: l6-ballot
  template:
    metadata:
      labels:
        app: l6-ballot
    spec:
      containers:
        - name: ballot
          image: ghcr.io/bngams/kube-vote:3.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml
for i in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15; do
  POD=$(kubectl get pods -l app=l6-ballot --sort-by=.metadata.creationTimestamp -o name | tail -n 1)
  { kubectl logs "$POD" 2>/dev/null; kubectl logs --previous "$POD" 2>/dev/null; } | grep KeyError > ~/byte-layers/kubernetes/l6/challenge/cause.txt && break
  sleep 3
done
cat ~/byte-layers/kubernetes/l6/challenge/cause.txt
```
````

# Tâche 3 : rétablir le service

## Revenir à la 2.0 et remettre le fichier en cohérence

La priorité est de rétablir le service, avant de corriger quoi que ce soit. Ramenez `l6-ballot` à la version précédente, puis faites en sorte que votre fichier dise la même chose que le cluster.

**Critères :**

- le Deployment `l6-ballot` demande de nouveau l'image `ghcr.io/bngams/kube-vote:2.0` ;
- ses 2 pods sont disponibles ;
- votre fichier `ballot.deploy.yml` dit `2.0` lui aussi, et c'est lui qui a été appliqué en dernier.

```hint
Kubernetes garde l'historique des versions d'un Deployment. Une sous-commande de `kubectl rollout` revient à la précédente : une seule fois.
```

```hint
Revenir en arrière ne touche pas à votre fichier : il demande toujours la `3.0`. Corrigez-le, puis appliquez-le.
```

```verify
D=l6-ballot
F="$HOME/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml"
IMG=$(kubectl get deployment "$D" -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "ÉCHEC : le Deployment $D n'existe pas. Faites d'abord la tâche 1."
  exit 1
fi
if ! kubectl get replicasets -l app="$D" -o jsonpath='{range .items[*]}{.spec.template.spec.containers[0].image}{"\n"}{end}' 2>/dev/null | grep -q 'kube-vote:3.0$'; then
  echo "ÉCHEC : la version 3.0 n'a jamais été déployée. Faites d'abord la tâche 2."
  exit 1
fi
case "$IMG" in
  *kube-vote:2.0) : ;;
  *) echo "ÉCHEC : $D demande l'image $IMG, pas la 2.0."
     echo "  Revenez à la version précédente, une seule fois. kubectl get deployments $D -o wide montre l'image demandée."
     exit 1 ;;
esac
if ! grep -Eq '^[[:space:]]*image:[[:space:]]*"?ghcr\.io/bngams/kube-vote:2\.0' "$F" 2>/dev/null; then
  echo "ÉCHEC : le cluster est revenu en 2.0, mais votre fichier ballot.deploy.yml demande encore une autre version."
  echo "  Au prochain kubectl apply, la v3 cassée reviendrait."
  exit 1
fi
LAST=$(kubectl get deployment "$D" -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' 2>/dev/null)
case "$LAST" in
  *kube-vote:2.0*) : ;;
  *) echo "ÉCHEC : votre fichier dit 2.0, mais le dernier fichier appliqué demandait une autre version."
     echo "  Appliquez ballot.deploy.yml."
     exit 1 ;;
esac
N=$(kubectl get deployment "$D" -o jsonpath='{.spec.replicas}' 2>/dev/null)
if [ "$N" != "2" ]; then
  echo "ÉCHEC : $D demande $N réplica(s), alors que le cahier des charges en veut 2."
  echo "  Remettez replicas: 2 dans ballot.deploy.yml, puis appliquez le fichier."
  exit 1
fi
if ! kubectl rollout status deployment/"$D" --timeout=45s >/dev/null 2>&1; then
  echo "ÉCHEC : le retour à la 2.0 n'est pas terminé."
  echo "  Regardez les pods : kubectl get pods -l app=$D"
  exit 1
fi
echo "RÉUSSI : $D est revenu en 2.0, et votre fichier dit la même chose."
```

````solution
```bash
kubectl rollout undo deployment l6-ballot
cat > ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l6-ballot
  labels:
    app: l6-ballot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: l6-ballot
  template:
    metadata:
      labels:
        app: l6-ballot
    spec:
      containers:
        - name: ballot
          image: ghcr.io/bngams/kube-vote:2.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml
kubectl rollout status deployment/l6-ballot --timeout=50s
```
````

# Tâche 4 : protéger les mises à jour

## Ajouter une sonde de readiness et une sonde de liveness

Pendant que la v3 plantait, Kubernetes a pu retirer un pod sain. Faites en sorte que cela ne se reproduise pas : donnez à Kubernetes le moyen de vérifier lui-même que l'application répond. Elle a une page prévue pour cela, `/healthz`, sur le port `80`.

**Critères :**

- le conteneur de `l6-ballot` a une sonde de **readiness** et une sonde de **liveness** ;
- chacune fait une requête HTTP sur `/healthz`, port `80` ;
- `l6-ballot` est toujours en `2.0`, et ses 2 pods sont disponibles avec leurs sondes.

```hint
Les sondes s'écrivent dans le conteneur, au même niveau que `image` et `resources` : un bloc `readinessProbe` et un bloc `livenessProbe`.
```

```hint
Chaque bloc contient un `httpGet` avec un `path` et un `port`, et peut préciser `periodSeconds`, le temps entre deux questions.
```

```verify
D=l6-ballot
if ! kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment $D n'existe pas. Faites d'abord la tâche 1."
  exit 1
fi
for P in readinessProbe livenessProbe; do
  PATHV=$(kubectl get deployment "$D" -o jsonpath="{.spec.template.spec.containers[0].$P.httpGet.path}" 2>/dev/null)
  PORTV=$(kubectl get deployment "$D" -o jsonpath="{.spec.template.spec.containers[0].$P.httpGet.port}" 2>/dev/null)
  if [ -z "$PORTV" ]; then
    echo "ÉCHEC : le conteneur de $D n'a pas de $P qui fait une requête HTTP (httpGet)."
    exit 1
  fi
  if [ "$PATHV" != "/healthz" ] || [ "$PORTV" != "80" ]; then
    echo "ÉCHEC : la $P interroge « ${PATHV:-/} » sur le port $PORTV, pas /healthz sur le port 80."
    exit 1
  fi
done
IMG=$(kubectl get deployment "$D" -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
case "$IMG" in
  *kube-vote:2.0) : ;;
  *) echo "ÉCHEC : les sondes sont là, mais $D demande l'image $IMG : cette tâche se fait en 2.0."
     exit 1 ;;
esac
N=$(kubectl get deployment "$D" -o jsonpath='{.spec.replicas}' 2>/dev/null)
if [ "$N" != "2" ]; then
  echo "ÉCHEC : $D demande $N réplica(s), alors que le cahier des charges en veut 2."
  echo "  Remettez replicas: 2 dans ballot.deploy.yml, puis appliquez le fichier."
  exit 1
fi
if ! kubectl rollout status deployment/"$D" --timeout=45s >/dev/null 2>&1; then
  echo "ÉCHEC : les sondes sont écrites, mais les pods ne sont pas tous prêts."
  echo "  Regardez-les : kubectl get pods -l app=$D, puis kubectl describe pod sur l'un d'eux."
  exit 1
fi
echo "RÉUSSI : les deux sondes interrogent /healthz, et les 2 pods de $D y répondent."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l6-ballot
  labels:
    app: l6-ballot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: l6-ballot
  template:
    metadata:
      labels:
        app: l6-ballot
    spec:
      containers:
        - name: ballot
          image: ghcr.io/bngams/kube-vote:2.0
          ports:
            - containerPort: 80
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
              cpu: 50m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml
kubectl rollout status deployment/l6-ballot --timeout=50s
```
````

# Tâche 5 : livrer la v3 pour de bon

## Donner son réglage à la v3, puis la déployer

Vous savez ce que la v3 réclame. Donnez-le-lui dans une ConfigMap, puis livrez la `3.0`, avec ses sondes.

**Critères :**

- une ConfigMap s'appelle exactement `l6-ballot-config`, et contient le réglage que la v3 réclame, avec le titre de votre choix ;
- le conteneur de `l6-ballot` lit ses réglages dans cette ConfigMap ;
- le Deployment `l6-ballot` demande l'image `ghcr.io/bngams/kube-vote:3.0`, et garde ses deux sondes ;
- la mise à jour est terminée : ses 2 pods sont disponibles, en `3.0`.

```hint
Le nom du réglage est celui que vous avez noté dans `cause.txt`. Une ConfigMap se décrit dans un fichier, avec ses réglages sous `data:`.
```

```hint
Dans le conteneur, un bloc `envFrom` avec un `configMapRef` donne à l'application tous les réglages d'une ConfigMap.
```

```hint
L'ordre compte : appliquez la ConfigMap avant le Deployment. Les pods lisent leurs réglages à leur démarrage.
```

```verify
D=l6-ballot
CM=l6-ballot-config
if ! kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment $D n'existe pas. Faites d'abord la tâche 1."
  exit 1
fi
T=$(kubectl get configmap "$CM" -o jsonpath='{.data.VOTE_TITLE}' 2>/dev/null)
if [ -z "$T" ]; then
  echo "ÉCHEC : aucune ConfigMap $CM ne contient le réglage que la v3 réclame."
  echo "  Créez-la dans le namespace courant, avec ce réglage sous data:."
  exit 1
fi
FROM=$(kubectl get deployment "$D" -o jsonpath='{.spec.template.spec.containers[0].envFrom[*].configMapRef.name} {.spec.template.spec.containers[0].env[*].valueFrom.configMapKeyRef.name}' 2>/dev/null)
case " $FROM " in
  *" $CM "*) : ;;
  *) echo "ÉCHEC : le conteneur de $D ne lit pas ses réglages dans la ConfigMap $CM."
     exit 1 ;;
esac
IMG=$(kubectl get deployment "$D" -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
case "$IMG" in
  *kube-vote:3.0) : ;;
  *) echo "ÉCHEC : $D demande l'image $IMG, pas la 3.0."
     exit 1 ;;
esac
for P in readinessProbe livenessProbe; do
  if [ "$(kubectl get deployment "$D" -o jsonpath="{.spec.template.spec.containers[0].$P.httpGet.path}" 2>/dev/null)" != "/healthz" ]; then
    echo "ÉCHEC : $D a perdu sa $P sur /healthz. La v3 se livre avec ses sondes."
    exit 1
  fi
done
N=$(kubectl get deployment "$D" -o jsonpath='{.spec.replicas}' 2>/dev/null)
if [ "$N" != "2" ]; then
  echo "ÉCHEC : $D demande $N réplica(s), alors que le cahier des charges en veut 2."
  echo "  Remettez replicas: 2 dans ballot.deploy.yml, puis appliquez le fichier."
  exit 1
fi
if ! kubectl rollout status deployment/"$D" --timeout=45s >/dev/null 2>&1; then
  echo "ÉCHEC : la 3.0 est demandée, mais ses pods ne sont pas tous prêts."
  echo "  Lisez leurs logs. Un pod qui plantait avant l'ajout du réglage le lira à son prochain redémarrage."
  exit 1
fi
echo "RÉUSSI : $D tourne en 3.0 sur ses 2 pods, avec le titre « $T » lu dans $CM."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l6/challenge/ballot.config.yml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: l6-ballot-config
data:
  VOTE_TITLE: "Votre destination de vacances ?"
EOF
cat > ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l6-ballot
  labels:
    app: l6-ballot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: l6-ballot
  template:
    metadata:
      labels:
        app: l6-ballot
    spec:
      containers:
        - name: ballot
          image: ghcr.io/bngams/kube-vote:3.0
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: l6-ballot-config
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
              cpu: 50m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/challenge/ballot.config.yml
kubectl apply -f ~/byte-layers/kubernetes/l6/challenge/ballot.deploy.yml
kubectl rollout status deployment/l6-ballot --timeout=50s
```
````

# Tâche 6 : ne rien laisser derrière soi

## Supprimer le Deployment, la ConfigMap et le dossier

Supprimez tout ce que ce challenge a créé : le Deployment `l6-ballot`, la ConfigMap `l6-ballot-config`, et le dossier `~/byte-layers/kubernetes/l6/challenge`.

**Critères :** plus de Deployment `l6-ballot` ni de pods à lui, plus de ConfigMap `l6-ballot-config`, plus de dossier. L'application de vote et vos autres fichiers ne sont pas concernés.

```hint
`kubectl delete` supprime un objet par son type et son nom, ou tout ce qu'un fichier décrit avec `-f`. Supprimer un Deployment supprime ses pods.
```

```hint
Pour un dossier et tout ce qu'il contient : `rm -r`, en vérifiant bien le chemin avant de valider.
```

```verify
D=l6-ballot
if ! kubectl get deployments -o name >/dev/null 2>&1; then
  echo "ÉCHEC : kubectl ne joint pas le cluster : impossible de vérifier le nettoyage."
  echo "  Vérifiez que votre cluster est démarré, puis réessayez."
  exit 1
fi
if kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment $D existe encore."
  exit 1
fi
for i in 1 2 3 4 5 6 7 8; do
  LEFT=$(kubectl get pods -l app="$D" -o name 2>/dev/null)
  [ -z "$LEFT" ] && break
  sleep 5
done
if [ -n "$LEFT" ]; then
  echo "ÉCHEC : il reste des pods de $D : $(echo "$LEFT" | tr '\n' ' ')"
  echo "  S'ils sont en train de s'arrêter, attendez quelques secondes et réessayez."
  exit 1
fi
if kubectl get configmap l6-ballot-config >/dev/null 2>&1; then
  echo "ÉCHEC : la ConfigMap l6-ballot-config existe encore."
  exit 1
fi
if [ -e "$HOME/byte-layers/kubernetes/l6/challenge" ]; then
  echo "ÉCHEC : le dossier ~/byte-layers/kubernetes/l6/challenge existe encore."
  exit 1
fi
echo "RÉUSSI : tout est propre. Vous savez mettre à jour une application, et la réparer."
```

````solution
```bash
kubectl delete deployment l6-ballot
kubectl delete configmap l6-ballot-config
rm -r ~/byte-layers/kubernetes/l6/challenge
```
````
