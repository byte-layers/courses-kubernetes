---
title: "Dimensionner une application, sans guide"
description: "Déployer une application en plusieurs exemplaires, changer leur nombre durablement, provoquer puis corriger un pod Pending, et l'éteindre sans la supprimer, à partir d'un cahier des charges."
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

Ce challenge prouve que vous savez dimensionner une application dans Kubernetes : choisir son nombre de réplicas, réserver ses ressources, et comprendre ce qui se passe quand le cluster n'a plus de place. Il n'y a **aucune commande à recopier** : chaque tâche donne un objectif et des critères, puis un bouton vérifie l'état réel du cluster. Toute suite de commandes qui produit le bon résultat est acceptée.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : ce qu'enseignent [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling) et [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending), et savoir écrire le fichier d'un Deployment, ce qu'enseigne [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment).
- **Ce qu'il faut avoir** : un cluster qui répond à `kubectl get nodes`, et un éditeur de texte. Aucun port de votre machine n'est utilisé.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l2/challenge` est créé pour vous, et la dernière tâche le supprime.
- **Une application à vous** : vous déployez `kiosk`, une seconde borne de vote qui réutilise l'image `ghcr.io/bngams/kube-vote:1.0`. Les vérifications cherchent ce nom exact et l'étiquette `app: kiosk`. Le Deployment `vote` et son fichier ne sont jamais inspectés ni supprimés.
- **Le fichier fait foi** : à chaque tâche, le fichier `kiosk.deploy.yml` et le cluster doivent dire la même chose. Un changement fait seulement dans le cluster ne passe pas la vérification.
- **L'ordre compte** : chaque tâche reprend le Deployment de la précédente. Faites-les dans l'ordre.

# Tâche 1 : deux exemplaires, depuis un fichier

## Déployer kiosk en deux exemplaires

```setup
mkdir -p ~/byte-layers/kubernetes/l2/challenge
```

Écrivez le fichier d'un Deployment pour la borne `kiosk`, puis faites-le exister dans le cluster.

**Critères :**

- le fichier `~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml` existe ;
- un Deployment s'appelle exactement `kiosk` ;
- ses pods portent l'étiquette `app: kiosk` ;
- son conteneur utilise l'image `ghcr.io/bngams/kube-vote:1.0` ;
- le fichier et le cluster demandent **2 réplicas**, et les 2 pods sont prêts.

```hint
Le fichier de `vote` est un bon point de départ : un Deployment a toujours la même forme. Trois choses changent ici : le nom, l'étiquette, et le nombre de réplicas.
```

```hint
L'étiquette apparaît à trois endroits du fichier, et ils doivent s'accorder : `metadata.labels`, `spec.selector.matchLabels` et `spec.template.metadata.labels`.
```

```hint
Le fichier se transmet au cluster avec `kubectl apply -f`, et `kubectl rollout status deployment/kiosk` attend que les pods soient prêts.
```

```verify
F="$HOME/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml"
if ! kubectl get nodes >/dev/null 2>&1; then
  echo "ÉCHEC : le cluster ne répond pas. Démarrez-le et réessayez."
  exit 1
fi
if [ ! -f "$F" ]; then
  echo "ÉCHEC : le fichier ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml n'existe pas."
  echo "  Écrivez-y le Deployment kiosk, puis appliquez-le."
  exit 1
fi
if ! kubectl get deployment kiosk >/dev/null 2>&1; then
  echo "ÉCHEC : aucun Deployment ne s'appelle kiosk."
  echo "  Le fichier existe : il reste à l'appliquer, ou à corriger metadata.name."
  exit 1
fi
IMG=$(kubectl get deployment kiosk -o jsonpath='{.spec.template.spec.containers[0].image}')
if [ "$IMG" != "ghcr.io/bngams/kube-vote:1.0" ]; then
  echo "ÉCHEC : kiosk utilise l'image « $IMG », et non ghcr.io/bngams/kube-vote:1.0."
  exit 1
fi
LABEL=$(kubectl get deployment kiosk -o jsonpath='{.spec.template.metadata.labels.app}')
if [ "$LABEL" != "kiosk" ]; then
  echo "ÉCHEC : les pods de kiosk portent l'étiquette app: ${LABEL:-absente}, et non app: kiosk."
  echo "  Un sélecteur ne se modifie pas après coup : supprimez le Deployment, corrigez le fichier, puis appliquez-le."
  exit 1
fi
IN_FILE=$(sed -E 's/#.*//' "$F" | grep -E '^[[:space:]]*replicas:' | head -1 | tr -cd '0-9')
WANT=$(kubectl get deployment kiosk -o jsonpath='{.spec.replicas}')
if [ "$IN_FILE" != "2" ] || [ "$WANT" != "2" ]; then
  echo "ÉCHEC : attendu 2 réplicas dans le fichier et dans le cluster."
  echo "  Trouvé : ${IN_FILE:-rien} dans le fichier, $WANT dans le cluster."
  exit 1
fi
if ! kubectl rollout status deployment/kiosk --timeout=40s >/dev/null 2>&1 || [ "$(kubectl get deployment kiosk -o jsonpath='{.status.readyReplicas}')" != "2" ]; then
  echo "ÉCHEC : les 2 pods de kiosk ne sont pas prêts."
  echo "  Regardez leur état : kubectl get pods -l app=kiosk"
  exit 1
fi
echo "RÉUSSI : kiosk tourne en 2 exemplaires, depuis votre fichier."
```

````solution
```bash
mkdir -p ~/byte-layers/kubernetes/l2/challenge
cat > ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiosk
  labels:
    app: kiosk
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kiosk
  template:
    metadata:
      labels:
        app: kiosk
    spec:
      containers:
        - name: kiosk
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml
kubectl rollout status deployment/kiosk --timeout=50s
```
````

# Tâche 2 : trois exemplaires, durablement

## Passer kiosk à trois réplicas sans créer d'écart

La borne a du succès : il faut un exemplaire de plus, et ce changement doit **survivre** au prochain `kubectl apply` d'un collègue.

**Critères :**

- le cluster demande **3 réplicas** pour `kiosk`, et les 3 pods sont prêts ;
- le fichier `kiosk.deploy.yml` demande lui aussi 3 réplicas.

```hint
Deux chemins mènent à 3 pods. Un seul laisse une trace que le prochain `apply` respectera.
```

```hint
Modifiez la ligne `replicas` du fichier, puis appliquez-le. Si vous avez pris le raccourci `kubectl scale`, il reste à reporter la valeur dans le fichier.
```

```verify
F="$HOME/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml"
if ! kubectl get deployment kiosk >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment kiosk n'existe pas. Faites d'abord la tâche 1."
  exit 1
fi
WANT=$(kubectl get deployment kiosk -o jsonpath='{.spec.replicas}')
if [ "$WANT" != "3" ]; then
  echo "ÉCHEC : le cluster demande $WANT réplica(s) pour kiosk, et non 3."
  exit 1
fi
IN_FILE=$(sed -E 's/#.*//' "$F" 2>/dev/null | grep -E '^[[:space:]]*replicas:' | head -1 | tr -cd '0-9')
if [ "$IN_FILE" != "3" ]; then
  echo "ÉCHEC : le cluster a bien 3 réplicas, mais le fichier en demande ${IN_FILE:-un nombre illisible}."
  echo "  Le prochain kubectl apply effacerait votre changement : reportez la valeur dans le fichier."
  exit 1
fi
if ! kubectl rollout status deployment/kiosk --timeout=40s >/dev/null 2>&1 || [ "$(kubectl get deployment kiosk -o jsonpath='{.status.readyReplicas}')" != "3" ]; then
  echo "ÉCHEC : les 3 pods de kiosk ne sont pas prêts."
  echo "  Regardez leur état : kubectl get pods -l app=kiosk"
  exit 1
fi
echo "RÉUSSI : 3 exemplaires de kiosk, et le fichier dit la même chose."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiosk
  labels:
    app: kiosk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kiosk
  template:
    metadata:
      labels:
        app: kiosk
    spec:
      containers:
        - name: kiosk
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml
kubectl rollout status deployment/kiosk --timeout=50s
```
````

# Tâche 3 : demander l'impossible

## Provoquer un pod Pending et garder la raison dans un fichier

Faites réserver à chaque pod de `kiosk` **64 Gi de mémoire**, bien plus que ce que votre nœud possède, et **50m** de processeur. Retrouvez ensuite la raison que donne l'ordonnanceur, et gardez-la dans le fichier `~/byte-layers/kubernetes/l2/challenge/pending.txt`, en y envoyant la réponse de `kubectl` avec `>`, ou en y collant le message avec votre éditeur.

**Critères :**

- le modèle de pod de `kiosk` réserve `64Gi` de mémoire ;
- un pod de `kiosk` est bloqué en `Pending`, faute de place ;
- les 3 anciens pods répondent toujours ;
- le fichier `pending.txt` contient la raison donnée par l'ordonnanceur, telle que Kubernetes l'écrit.

```hint
Une réservation s'écrit sous le conteneur, dans `resources.requests`, avec une ligne `cpu` et une ligne `memory`. La mémoire prend un suffixe : `Mi`, `Gi`.
```

```hint
La raison d'un pod bloqué se lit dans ses événements, à la fin de `kubectl describe`. Vous pouvez désigner les pods de `kiosk` par leur étiquette, avec `-l app=kiosk`.
```

```hint
Le message à garder ressemble à celui que vous connaissez pour le processeur, avec un autre mot après `Insufficient`.
```

```verify
D="$HOME/byte-layers/kubernetes/l2/challenge"
if ! kubectl get deployment kiosk >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment kiosk n'existe pas. Faites d'abord les tâches 1 et 2."
  exit 1
fi
MEM=$(kubectl get deployment kiosk -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')
case "$MEM" in
  64Gi|65536Mi|68719476736) : ;;
  "") echo "ÉCHEC : kiosk ne réserve pas de mémoire. Ajoutez resources.requests sous le conteneur, puis appliquez le fichier."
      exit 1 ;;
  *)  echo "ÉCHEC : kiosk réserve $MEM de mémoire, et non 64Gi."
      exit 1 ;;
esac
found=0
for i in 1 2 3 4 5 6 7 8 9 10; do
  REASON=$(kubectl get pods -l app=kiosk --field-selector=status.phase=Pending -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="PodScheduled")].reason}{" "}{end}')
  case "$REASON" in *Unschedulable*) found=1; break ;; esac
  sleep 2
done
if [ "$found" != 1 ]; then
  echo "ÉCHEC : aucun pod de kiosk n'est bloqué en Pending faute de place."
  echo "  Regardez : kubectl get pods -l app=kiosk"
  exit 1
fi
AVAIL=$(kubectl get deployment kiosk -o jsonpath='{.status.availableReplicas}')
if [ "$AVAIL" != "3" ]; then
  echo "ÉCHEC : seulement ${AVAIL:-0} ancien(s) pod(s) de kiosk sur 3 répondent encore."
  echo "  Revenez à l'état de la tâche 2, attendez les 3 pods, puis refaites le changement."
  exit 1
fi
if [ ! -f "$D/pending.txt" ]; then
  echo "ÉCHEC : le pod est bien Pending, mais le fichier ~/byte-layers/kubernetes/l2/challenge/pending.txt n'existe pas."
  exit 1
fi
if ! grep -q "Insufficient memory" "$D/pending.txt"; then
  echo "ÉCHEC : pending.txt ne contient pas la raison donnée par l'ordonnanceur."
  echo "  Elle se lit dans les événements du pod bloqué, à la fin de kubectl describe."
  exit 1
fi
echo "RÉUSSI : un pod attend faute de mémoire, les 3 anciens servent toujours, et la raison est notée."
```

````solution
La dernière commande attend dix secondes, le temps que l'ordonnanceur écrive son événement, puis envoie le détail des pods de `kiosk` dans le fichier :

```bash
cat > ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiosk
  labels:
    app: kiosk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kiosk
  template:
    metadata:
      labels:
        app: kiosk
    spec:
      containers:
        - name: kiosk
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Gi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml
sleep 10
kubectl describe pods -l app=kiosk > ~/byte-layers/kubernetes/l2/challenge/pending.txt
```
````

# Tâche 4 : une réservation réaliste

## Corriger la réservation et laisser la mise à jour se terminer

Donnez à `kiosk` une réservation que le cluster peut honorer : **50m** de processeur et **32 Mi** de mémoire par pod.

**Critères :**

- le modèle de pod de `kiosk` réserve `50m` de processeur et `32Mi` de mémoire, dans le cluster comme dans le fichier ;
- les 3 pods sont prêts, et tous sont de la nouvelle version ;
- plus aucun pod de `kiosk` n'est `Pending`.

```hint
Une seule valeur est à corriger dans le fichier. Le pod bloqué n'a pas besoin d'être supprimé à la main : Kubernetes s'en charge quand le modèle change.
```

```hint
Après le `kubectl apply`, `kubectl rollout status deployment/kiosk` rend la main quand les 3 pods ont été remplacés.
```

```verify
F="$HOME/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml"
if ! kubectl get deployment kiosk >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment kiosk n'existe pas. Faites d'abord les tâches précédentes."
  exit 1
fi
CPU=$(kubectl get deployment kiosk -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')
MEM=$(kubectl get deployment kiosk -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')
case "$CPU" in
  50m|0.05) : ;;
  *) echo "ÉCHEC : kiosk réserve « ${CPU:-rien} » de processeur, et non 50m."
     exit 1 ;;
esac
if [ "$MEM" != "32Mi" ]; then
  echo "ÉCHEC : kiosk réserve « ${MEM:-rien} » de mémoire, et non 32Mi."
  exit 1
fi
if ! sed -E 's/#.*//' "$F" 2>/dev/null | grep -Eq 'memory:[[:space:]]*"?32Mi"?[[:space:]]*$'; then
  echo "ÉCHEC : le cluster est à jour, mais le fichier kiosk.deploy.yml ne réserve pas 32Mi de mémoire."
  echo "  Le prochain kubectl apply ramènerait l'ancienne valeur : corrigez le fichier."
  exit 1
fi
if ! kubectl rollout status deployment/kiosk --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : la mise à jour de kiosk n'est pas terminée."
  echo "  Regardez : kubectl get pods -l app=kiosk"
  exit 1
fi
READY=$(kubectl get deployment kiosk -o jsonpath='{.status.readyReplicas}')
PENDING=$(kubectl get pods -l app=kiosk --field-selector=status.phase=Pending -o name | wc -l | tr -d ' ')
if [ "$READY" != "3" ] || [ "$PENDING" != "0" ]; then
  echo "ÉCHEC : attendu 3 pods prêts et aucun pod Pending. Trouvé : ${READY:-0} prêt(s), $PENDING Pending."
  exit 1
fi
echo "RÉUSSI : 3 pods de kiosk prêts, chacun réservant 50m de processeur et 32Mi de mémoire."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiosk
  labels:
    app: kiosk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kiosk
  template:
    metadata:
      labels:
        app: kiosk
    spec:
      containers:
        - name: kiosk
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml
kubectl rollout status deployment/kiosk --timeout=50s
```
````

# Tâche 5 : éteindre sans supprimer

## Arrêter tous les pods de kiosk en gardant le Deployment

La borne ferme pour la nuit. Arrêtez tous ses pods, sans perdre le Deployment ni ses réglages.

**Critères :**

- le Deployment `kiosk` **existe toujours**, avec sa réservation de la tâche 4 ;
- le fichier et le cluster demandent **0 réplica** ;
- plus aucun pod de `kiosk` ne tourne.

```hint
Supprimer le Deployment arrêterait bien les pods, mais ferait échouer le premier critère. Un chiffre suffit.
```

```verify
F="$HOME/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml"
if ! kubectl get deployment kiosk >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment kiosk n'existe plus. Il fallait l'éteindre, pas le supprimer."
  echo "  Remettez-le en place avec votre fichier, puis demandez zéro réplica."
  exit 1
fi
IN_FILE=$(sed -E 's/#.*//' "$F" 2>/dev/null | grep -E '^[[:space:]]*replicas:' | head -1 | tr -cd '0-9')
WANT=$(kubectl get deployment kiosk -o jsonpath='{.spec.replicas}')
if [ "$IN_FILE" != "0" ] || [ "$WANT" != "0" ]; then
  echo "ÉCHEC : attendu 0 réplica dans le fichier et dans le cluster."
  echo "  Trouvé : ${IN_FILE:-rien} dans le fichier, $WANT dans le cluster."
  exit 1
fi
if [ "$(kubectl get deployment kiosk -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')" != "32Mi" ]; then
  echo "ÉCHEC : kiosk a perdu sa réservation de 32Mi de mémoire. Gardez le reste du fichier tel qu'à la tâche 4."
  exit 1
fi
kubectl wait --for=delete pod -l app=kiosk --timeout=40s >/dev/null 2>&1
LEFT=$(kubectl get pods -l app=kiosk -o name | wc -l | tr -d ' ')
if [ "$LEFT" != "0" ]; then
  echo "ÉCHEC : il reste $LEFT pod(s) de kiosk en train de s'arrêter. Attendez quelques secondes, puis réessayez."
  exit 1
fi
echo "RÉUSSI : plus aucun pod, et le Deployment kiosk est toujours là."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiosk
  labels:
    app: kiosk
spec:
  replicas: 0
  selector:
    matchLabels:
      app: kiosk
  template:
    metadata:
      labels:
        app: kiosk
    spec:
      containers:
        - name: kiosk
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml
kubectl wait --for=delete pod -l app=kiosk --timeout=50s
```
````

# Tâche 6 : ne rien laisser derrière soi

## Supprimer le Deployment kiosk et le dossier du challenge

Supprimez tout ce que ce challenge a créé : le Deployment `kiosk` et le dossier `~/byte-layers/kubernetes/l2/challenge`.

**Critères :** plus de Deployment `kiosk`, plus aucun pod portant `app: kiosk`, plus de dossier `challenge`. Le Deployment `vote` et le fichier `~/byte-layers/kubernetes/l2/vote.deploy.yml` ne sont pas concernés : laissez-les en place.

```hint
`kubectl delete -f` supprime ce qu'un fichier décrit : faites-le avant d'effacer le fichier.
```

```hint
Pour un dossier et tout ce qu'il contient : `rm -r`, en vérifiant bien le chemin avant de valider.
```

```verify
if ! kubectl get nodes >/dev/null 2>&1; then
  echo "ÉCHEC : le cluster ne répond pas. Démarrez-le et réessayez."
  exit 1
fi
if kubectl get deployment kiosk >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment kiosk existe encore."
  exit 1
fi
kubectl wait --for=delete pod -l app=kiosk --timeout=40s >/dev/null 2>&1
LEFT=$(kubectl get pods -l app=kiosk -o name | wc -l | tr -d ' ')
if [ "$LEFT" != "0" ]; then
  echo "ÉCHEC : il reste $LEFT pod(s) portant app: kiosk."
  exit 1
fi
if [ -e "$HOME/byte-layers/kubernetes/l2/challenge" ]; then
  echo "ÉCHEC : le dossier ~/byte-layers/kubernetes/l2/challenge existe encore."
  exit 1
fi
echo "RÉUSSI : tout est propre. Vous savez dimensionner une application."
```

````solution
```bash
kubectl delete -f ~/byte-layers/kubernetes/l2/challenge/kiosk.deploy.yml
rm -r ~/byte-layers/kubernetes/l2/challenge
```
````
