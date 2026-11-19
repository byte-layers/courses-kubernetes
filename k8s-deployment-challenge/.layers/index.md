---
title: "Un Deployment qui se répare, sans guide"
description: "Écrire et appliquer un Deployment, provoquer deux pannes, retrouver son ReplicaSet et tout supprimer, à partir d'un cahier des charges, sans guide."
tags:
  - challenge
  - kubernetes
  - containers
difficulty: beginner
duration: 30
introTitle: "Le défi"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Ce challenge prouve que vous savez décrire une application dans un Deployment, et dire ce que Kubernetes fait quand elle tombe en panne. Il n'y a **aucune commande à recopier** : chaque tâche donne un objectif et des critères, puis un bouton vérifie l'état réel du cluster. Toute suite de commandes qui produit le bon résultat est acceptée.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment).
- **Ce qu'il faut avoir** : un cluster qui répond à `kubectl get nodes`, un éditeur de texte, `curl` ou un navigateur, et le port **28612** libre.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l1/challenge` est créé pour vous, et la dernière tâche le supprime. La vérification y garde un fichier caché, `.first-pod`, qui lui sert à reconnaître votre premier pod.
- **Les noms comptent** : les vérifications cherchent le Deployment `challenge-web` et l'étiquette `app: challenge-web`, dans le namespace courant. Rien d'autre n'est inspecté ni supprimé : le Deployment `vote` du cours reste comme il est.
- **L'ordre compte** : chaque tâche reprend le résultat de la précédente. Faites-les dans l'ordre, et lancez la vérification d'une tâche avant de commencer la suivante.

# Tâche 1 : décrire et déployer

## Écrire un Deployment dans un fichier et l'appliquer

```setup
mkdir -p ~/byte-layers/kubernetes/l1/challenge
```

Une équipe vous demande de faire tourner un exemplaire de sa page web, à partir d'une description écrite qu'elle pourra relire. Décrivez l'état souhaité dans un fichier, puis transmettez-le au cluster.

**Critères :**

- le fichier `~/byte-layers/kubernetes/l1/challenge/web.deploy.yml` décrit un Deployment ;
- ce Deployment s'appelle exactement `challenge-web`, et il a été transmis au cluster **à partir du fichier** ;
- il demande **1** réplica ;
- ses pods portent l'étiquette `app: challenge-web` ;
- leur conteneur vient de l'image `ghcr.io/bngams/kube-vote:1.0` ;
- le pod est **prêt**.

```hint
La forme du fichier est celle du Deployment `vote` : `apiVersion`, `kind`, `metadata`, puis `spec` avec `replicas`, `selector` et `template`. Seuls le nom et les étiquettes changent.
```

```hint
L'étiquette `app: challenge-web` s'écrit deux fois, à l'identique : sous `selector.matchLabels` et sous `template.metadata.labels`. Si elles diffèrent, Kubernetes refuse le fichier.
```

```hint
La commande qui transmet un fichier YAML est `kubectl apply`, avec `-f` et le chemin du fichier. `kubectl get deployments` doit ensuite afficher `challenge-web` avec `READY 1/1`.
```

```verify
D=challenge-web
DIR="$HOME/byte-layers/kubernetes/l1/challenge"
FILE="$DIR/web.deploy.yml"
if ! kubectl version --request-timeout=10s >/dev/null 2>&1; then
  echo "ÉCHEC : kubectl ne joint pas le cluster. Vérifiez qu'il est démarré avec kubectl get nodes."
  exit 1
fi
if [ ! -f "$FILE" ]; then
  echo "ÉCHEC : le fichier ~/byte-layers/kubernetes/l1/challenge/web.deploy.yml n'existe pas."
  echo "  Écrivez-y la description du Deployment, puis transmettez-la au cluster."
  exit 1
fi
if ! grep -Eq '^kind:[[:space:]]*"?Deployment"?[[:space:]]*(#.*)?$' "$FILE"; then
  echo "ÉCHEC : le fichier web.deploy.yml ne décrit pas un Deployment (ligne kind)."
  exit 1
fi
if ! kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : il n'y a pas de Deployment $D dans le cluster."
  FOUND=$(kubectl get deployments -o name 2>/dev/null | tr '\n' ' ')
  [ -n "$FOUND" ] && echo "  Trouvé à la place : $FOUND"
  echo "  Transmettez votre fichier, et lisez le message de la commande s'il y en a un."
  exit 1
fi
if [ -z "$(kubectl get deployment "$D" -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}')" ]; then
  echo "ÉCHEC : le Deployment $D existe, mais il n'a pas été transmis à partir d'un fichier."
  echo "  Supprimez-le, puis créez-le avec la commande qui applique un fichier YAML."
  exit 1
fi
REPLICAS=$(kubectl get deployment "$D" -o jsonpath='{.spec.replicas}')
if [ "$REPLICAS" != "1" ]; then
  echo "ÉCHEC : $D demande $REPLICAS réplicas, et non 1."
  exit 1
fi
LABEL=$(kubectl get deployment "$D" -o jsonpath='{.spec.template.metadata.labels.app}')
if [ "$LABEL" != "$D" ]; then
  echo "ÉCHEC : les pods de $D ne portent pas l'étiquette app: $D (trouvé : app: ${LABEL:-rien})."
  exit 1
fi
IMAGE=$(kubectl get deployment "$D" -o jsonpath='{.spec.template.spec.containers[0].image}')
if [ "$IMAGE" != "ghcr.io/bngams/kube-vote:1.0" ]; then
  echo "ÉCHEC : le conteneur de $D utilise l'image « $IMAGE », et non ghcr.io/bngams/kube-vote:1.0."
  exit 1
fi
if ! kubectl rollout status deployment/"$D" --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment $D existe, mais son pod n'est pas prêt."
  echo "  Lisez la colonne STATUS de kubectl get pods."
  exit 1
fi
UID_NOW=$(kubectl get deployment "$D" -o jsonpath='{.metadata.uid}')
if [ "$(cut -d' ' -f1 "$DIR/.first-pod" 2>/dev/null)" != "$UID_NOW" ]; then
  POD=$(kubectl get pods -l app="$D" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1:].metadata.name}')
  echo "$UID_NOW $POD" > "$DIR/.first-pod"
fi
echo "RÉUSSI : $D maintient 1 pod prêt, décrit dans web.deploy.yml."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l1/challenge/web.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: challenge-web
  labels:
    app: challenge-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: challenge-web
  template:
    metadata:
      labels:
        app: challenge-web
    spec:
      containers:
        - name: web
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
EOF
kubectl apply -f ~/byte-layers/kubernetes/l1/challenge/web.deploy.yml
kubectl rollout status deployment/challenge-web --timeout=50s
```
````

# Tâche 2 : la panne du pod

## Faire disparaître le pod et laisser Kubernetes le remplacer

Simulez la perte du pod de `challenge-web`, sans toucher au Deployment, et constatez que l'application revient seule.

**Critères :**

- le pod que la tâche 1 a vu tourner **n'existe plus** ;
- un **autre** pod de `challenge-web` est prêt ;
- le Deployment `challenge-web` est toujours celui de la tâche 1 : il n'a été ni supprimé ni recréé.

```hint
C'est la panne que vous avez provoquée sur le pod de `vote` : une seule commande, qui vise le pod et non le Deployment.
```

```hint
`kubectl get pods` donne le nom exact du pod, avec ses deux suffixes. Après la suppression, la même commande doit montrer un pod sous un autre nom.
```

```verify
D=challenge-web
REF="$HOME/byte-layers/kubernetes/l1/challenge/.first-pod"
if ! kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : il n'y a pas de Deployment $D. Faites d'abord la tâche 1."
  exit 1
fi
if [ ! -s "$REF" ]; then
  echo "ÉCHEC : la tâche 1 n'a pas encore été vérifiée. Lancez sa vérification avant de supprimer le pod."
  exit 1
fi
UID_REF=$(cut -d' ' -f1 "$REF"); OLD=$(cut -d' ' -f2 "$REF")
if [ "$(kubectl get deployment "$D" -o jsonpath='{.metadata.uid}')" != "$UID_REF" ]; then
  echo "ÉCHEC : le Deployment $D n'est plus celui de la tâche 1 : il a été supprimé puis recréé."
  echo "  Relancez la vérification de la tâche 1, puis supprimez seulement le pod."
  exit 1
fi
if kubectl get pod "$OLD" >/dev/null 2>&1; then
  if [ -z "$(kubectl get pod "$OLD" -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null)" ]; then
    echo "ÉCHEC : le pod de la tâche 1, $OLD, tourne toujours. Supprimez-le, sans toucher au Deployment."
    exit 1
  fi
  kubectl wait --for=delete pod/"$OLD" --timeout=30s >/dev/null 2>&1
fi
if ! kubectl rollout status deployment/"$D" --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : l'ancien pod a disparu, mais son remplaçant n'est pas encore prêt. Réessayez dans quelques secondes."
  exit 1
fi
NEW=$(kubectl get pods -l app="$D" -o name | grep -v "^pod/$OLD\$" | head -1)
if [ -z "$NEW" ]; then
  echo "ÉCHEC : aucun nouveau pod de $D n'est visible."
  exit 1
fi
echo "RÉUSSI : $OLD a disparu, et ${NEW#pod/} l'a remplacé sans que vous le demandiez."
```

````solution
```bash
kubectl delete pod -l app=challenge-web
kubectl rollout status deployment/challenge-web --timeout=50s
```

`-l app=challenge-web` désigne le pod par son étiquette ; `kubectl delete pod` suivi de son nom revient au même.
````

# Tâche 3 : la panne du conteneur

## Faire planter l'application sans perdre le pod

L'image contient une page `/crash`, qui arrête le programme du conteneur. Servez-vous-en pour provoquer l'autre panne : celle où le pod reste, et où son conteneur est relancé.

**Critères :**

- le conteneur du pod de `challenge-web` a **redémarré au moins une fois** ;
- le pod est de nouveau **prêt**.

Pour joindre l'application depuis votre machine, utilisez le port **28612**.

```hint
L'application écoute sur le port 80 du conteneur, à l'intérieur du cluster. Il faut un tunnel entre votre machine et le pod pour lui demander une page.
```

```hint
`kubectl port-forward deployment/challenge-web 28612:80` ouvre ce tunnel. Il occupe le terminal : lancez-le en arrière-plan, ou ouvrez un second terminal.
```

```hint
Tunnel ouvert, demandez `http://localhost:28612/crash` avec votre navigateur ou avec `curl`. La colonne `RESTARTS` de `kubectl get pods` doit passer à 1.
```

```verify
D=challenge-web
if ! kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : il n'y a pas de Deployment $D. Faites d'abord la tâche 1."
  exit 1
fi
RESTARTS=$(kubectl get pods -l app="$D" -o jsonpath='{range .items[*]}{.status.containerStatuses[0].restartCount}{"\n"}{end}' | sort -n | tail -1)
if [ -z "$RESTARTS" ] || [ "$RESTARTS" -lt 1 ]; then
  echo "ÉCHEC : le conteneur du pod de $D n'a jamais redémarré (RESTARTS vaut ${RESTARTS:-0})."
  echo "  Supprimer le pod ne compte pas : son remplaçant repart avec RESTARTS à 0."
  exit 1
fi
if ! kubectl rollout status deployment/"$D" --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : le conteneur a bien planté, mais le pod n'est pas encore prêt. Réessayez dans quelques secondes."
  exit 1
fi
echo "RÉUSSI : le conteneur a planté et Kubernetes l'a redémarré dans le même pod (RESTARTS vaut $RESTARTS)."
```

````solution
```bash
kubectl port-forward deployment/challenge-web 28612:80 > /dev/null 2>&1 &
sleep 3
curl -s http://localhost:28612/crash
sleep 8
kubectl rollout status deployment/challenge-web --timeout=40s
```
````

# Tâche 4 : le gardien du nombre

## Noter le nom du ReplicaSet dans un fichier

Entre le Deployment et son pod, un objet que vous n'avez pas écrit maintient le nombre de pods. Retrouvez son nom et gardez-en une trace dans le fichier `~/byte-layers/kubernetes/l1/challenge/replicaset.txt`, en y envoyant la réponse de `kubectl` avec `>`, ou en y collant le nom avec votre éditeur.

**Critères :**

- le fichier `~/byte-layers/kubernetes/l1/challenge/replicaset.txt` existe ;
- il contient le **nom du ReplicaSet** de `challenge-web`.

```hint
`kubectl get` sait lister plusieurs types d'objets d'un coup, séparés par des virgules : les Deployments, les ReplicaSets et les pods.
```

```hint
Le nom du ReplicaSet est celui du Deployment suivi d'un suffixe, le premier des deux suffixes du nom du pod.
```

```verify
D=challenge-web
REPORT="$HOME/byte-layers/kubernetes/l1/challenge/replicaset.txt"
if ! kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : il n'y a pas de Deployment $D. Faites d'abord la tâche 1."
  exit 1
fi
if [ ! -f "$REPORT" ]; then
  echo "ÉCHEC : le fichier ~/byte-layers/kubernetes/l1/challenge/replicaset.txt n'existe pas."
  exit 1
fi
RS=$(kubectl get replicasets -l app="$D" -o jsonpath='{range .items[?(@.status.replicas>0)]}{.metadata.name}{"\n"}{end}' | head -1)
if [ -z "$RS" ]; then
  echo "ÉCHEC : aucun ReplicaSet actif pour $D. Le Deployment est-il prêt ?"
  exit 1
fi
if ! grep -Eq "(^|[/[:space:]])$RS([[:space:]]|\$)" "$REPORT"; then
  echo "ÉCHEC : le nom du ReplicaSet de $D n'est pas dans replicaset.txt."
  echo "  Le fichier contient : $(head -1 "$REPORT")"
  echo "  Le nom cherché commence par $D- et porte un seul suffixe."
  exit 1
fi
echo "RÉUSSI : le ReplicaSet $RS est bien celui qui maintient le pod de $D."
```

````solution
```bash
kubectl get replicasets -l app=challenge-web -o name > ~/byte-layers/kubernetes/l1/challenge/replicaset.txt
```

`kubectl get deployments,replicasets,pods` montre le même nom, à recopier dans le fichier.
````

# Tâche 5 : ne rien laisser derrière soi

## Supprimer le Deployment, le tunnel et le dossier

Supprimez tout ce que ce challenge a créé : le Deployment `challenge-web`, avec son ReplicaSet et son pod, le tunnel du port 28612 s'il tourne encore, et le dossier `~/byte-layers/kubernetes/l1/challenge`, par exemple avec `rm -r` suivi de son chemin.

**Critères :** plus de Deployment `challenge-web`, plus aucun pod étiqueté `app: challenge-web`, plus de tunnel, plus de dossier. Le Deployment `vote` n'est pas concerné.

```hint
Supprimer les pods ne suffit pas : tant que le Deployment existe, il les recrée. Supprimez l'état souhaité, et toute la chaîne part avec lui.
```

```hint
Un tunnel lancé en arrière-plan s'arrête avec `pkill -f` suivi d'un morceau de sa ligne de commande, entre guillemets. Pour un dossier et tout ce qu'il contient : `rm -r`, en vérifiant bien le chemin avant de valider.
```

```verify
D=challenge-web
if kubectl get deployment "$D" >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment $D existe encore."
  exit 1
fi
LEFT=$(kubectl get pods -l app="$D" -o name)
for i in 1 2 3 4 5 6 7 8 9 10; do
  [ -z "$LEFT" ] && break
  sleep 3
  LEFT=$(kubectl get pods -l app="$D" -o name)
done
if [ -n "$LEFT" ]; then
  echo "ÉCHEC : il reste un pod de $D : $LEFT"
  exit 1
fi
if pgrep -f "port-forward.*2861[2]" >/dev/null 2>&1; then
  echo "ÉCHEC : un tunnel kubectl port-forward tourne encore sur le port 28612."
  exit 1
fi
if [ -e "$HOME/byte-layers/kubernetes/l1/challenge" ]; then
  echo "ÉCHEC : le dossier ~/byte-layers/kubernetes/l1/challenge existe encore."
  exit 1
fi
echo "RÉUSSI : tout est propre. Vous savez déployer une application, et dire comment Kubernetes la répare."
```

````solution
```bash
kubectl delete deployment challenge-web
pkill -f "port-forward deployment/challenge-web 28612"
rm -r ~/byte-layers/kubernetes/l1/challenge
```
````
