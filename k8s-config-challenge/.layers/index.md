---
title: "Configurer une application, sans guide"
description: "Donner ses réglages et un mot de passe à une application à partir d'un cahier des charges : ConfigMap, Secret, redémarrage et lecture d'un Secret, sans guide."
tags:
  - challenge
  - kubernetes
  - containers
difficulty: beginner
duration: 40
introTitle: "Le défi"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Ce challenge prouve que vous savez configurer une application dans Kubernetes : lui donner ses réglages par une ConfigMap, un mot de passe par un Secret, et faire prendre en compte un changement. Il n'y a **aucune commande à recopier** dans les tâches : chacune donne un objectif et des critères, puis un bouton vérifie l'état réel du cluster. Toute suite de commandes qui produit le bon résultat est acceptée.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : ce qu'enseignent [Sortir les réglages de l'image avec une ConfigMap](https://byte-layers.com/fr/skills/k8s-configmap) et [Ranger un mot de passe dans un Secret](https://byte-layers.com/fr/skills/k8s-secret), et ce que montre [Un Secret est encodé, pas chiffré](https://byte-layers.com/fr/bits/k8s-secret-is-encoded).
- **Ce qu'il faut avoir** : un cluster `layers-k8s` auquel `kubectl` répond, et un éditeur de texte. Aucun port de votre machine n'est utilisé.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l5/challenge` est créé pour vous, et la dernière tâche le supprime.
- **Les noms comptent** : les vérifications cherchent les noms exacts, qui commencent tous par `survey`. L'application de vote, qui tourne dans le même namespace, n'est ni inspectée ni modifiée : n'y touchez pas.
- **L'ordre compte** : chaque tâche reprend ce que la précédente a créé. Faites-les dans l'ordre.

# Avant la première tâche

## Se placer dans le namespace vote-app

Cette étape est un support, la seule du challenge à donner ses commandes : elle place `kubectl` dans le namespace `vote-app`, où tout le challenge se passe. Créez d'abord ce namespace. S'il existe déjà, la commande répond `AlreadyExists`, sans conséquence :

```exec
kubectl create namespace vote-app
```

Faites-en ensuite le namespace par défaut de vos commandes :

```exec
kubectl config set-context --current --namespace=vote-app
```

```ack
kubectl travaille dans vote-app
```

# Tâche 1 : une application qui lit ses réglages

## Déployer survey avec une ConfigMap

```setup
mkdir -p ~/byte-layers/kubernetes/l5/challenge
```

Une équipe veut son propre sondage, `survey`, à côté de l'application de vote. Il utilise la même image, `ghcr.io/bngams/kube-vote:1.0`, qui lit ses deux options dans les variables d'environnement `OPTION_A` et `OPTION_B`. Déployez-le avec ses réglages rangés à part. Écrivez vos fichiers dans `~/byte-layers/kubernetes/l5/challenge`.

**Critères :**

- une ConfigMap s'appelle exactement `survey-config`, avec `OPTION_A` qui vaut `Train` et `OPTION_B` qui vaut `Avion` ;
- un Deployment s'appelle exactement `survey`, avec **1 réplica** de l'image `ghcr.io/bngams/kube-vote:1.0`, et ses pods portent l'étiquette `app: survey` ;
- son conteneur réserve **50m** de processeur et **64Mi** de mémoire, comme chaque morceau de l'application de vote ;
- le conteneur reçoit `OPTION_A` et `OPTION_B` **depuis la ConfigMap** : leurs valeurs ne sont pas écrites dans le Deployment ;
- le pod est prêt.

```hint
Deux fichiers suffisent : un pour la ConfigMap, un pour le Deployment. Le Deployment ressemble à celui de `vote`, avec un autre nom, une autre étiquette et un seul réplica.
```

```hint
Dans le conteneur, `envFrom` avec `configMapRef` importe toutes les clés d'une ConfigMap. Les réservations s'écrivent sous `resources.requests`.
```

```hint
Appliquez les deux fichiers avec `kubectl apply -f`, puis suivez le démarrage avec `kubectl rollout status deployment/survey`. Pour voir ce que le pod a reçu : `kubectl exec deployment/survey -- printenv OPTION_A`.
```

```verify
if ! kubectl get namespace >/dev/null 2>&1; then
  echo "ÉCHEC : kubectl ne joint pas le cluster. Démarrez-le et réessayez."
  exit 1
fi
if ! kubectl get configmap survey-config >/dev/null 2>&1; then
  echo "ÉCHEC : aucune ConfigMap ne s'appelle survey-config dans le namespace courant."
  exit 1
fi
A=$(kubectl get configmap survey-config -o jsonpath='{.data.OPTION_A}')
B=$(kubectl get configmap survey-config -o jsonpath='{.data.OPTION_B}')
if [ "$A" != "Train" ] || [ "$B" != "Avion" ]; then
  echo "ÉCHEC : survey-config doit porter OPTION_A=Train et OPTION_B=Avion (trouvé : « $A » et « $B »)."
  exit 1
fi
if ! kubectl get deployment survey >/dev/null 2>&1; then
  echo "ÉCHEC : aucun Deployment ne s'appelle survey dans le namespace courant."
  exit 1
fi
IMG=$(kubectl get deployment survey -o jsonpath='{.spec.template.spec.containers[0].image}')
if [ "$IMG" != "ghcr.io/bngams/kube-vote:1.0" ]; then
  echo "ÉCHEC : survey utilise l'image « $IMG », au lieu de ghcr.io/bngams/kube-vote:1.0."
  exit 1
fi
N=$(kubectl get deployment survey -o jsonpath='{.spec.replicas}')
if [ "$N" != "1" ]; then
  echo "ÉCHEC : survey demande $N réplicas, au lieu de 1."
  exit 1
fi
L=$(kubectl get deployment survey -o jsonpath='{.spec.template.metadata.labels.app}')
if [ "$L" != "survey" ]; then
  echo "ÉCHEC : les pods de survey doivent porter l'étiquette app: survey (trouvé : « $L »)."
  exit 1
fi
CPU=$(kubectl get deployment survey -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')
MEM=$(kubectl get deployment survey -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')
if [ "$CPU" != "50m" ] || [ "$MEM" != "64Mi" ]; then
  echo "ÉCHEC : le conteneur doit réserver 50m de processeur et 64Mi de mémoire (trouvé : « $CPU » et « $MEM »)."
  exit 1
fi
REF=$(kubectl get deployment survey -o jsonpath='{.spec.template.spec.containers[0].envFrom[*].configMapRef.name} {.spec.template.spec.containers[0].env[*].valueFrom.configMapKeyRef.name}')
case " $REF " in
  *" survey-config "*) : ;;
  *) echo "ÉCHEC : le Deployment survey ne lit pas la ConfigMap survey-config."
     echo "  Les valeurs ne doivent pas être écrites dans le Deployment : il doit citer la ConfigMap."
     exit 1 ;;
esac
if ! kubectl rollout status deployment/survey --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : le pod de survey n'est pas prêt. Regardez son état : kubectl get pods -l app=survey"
  echo "  Puis le détail, en bas de : kubectl describe pod -l app=survey"
  exit 1
fi
GA=""; GB=""
for i in 1 2 3 4 5; do
  GA=$(kubectl exec deployment/survey -- printenv OPTION_A 2>/dev/null)
  GB=$(kubectl exec deployment/survey -- printenv OPTION_B 2>/dev/null)
  if [ "$GA" = "Train" ] && [ "$GB" = "Avion" ]; then break; fi
  sleep 2
done
if [ "$GA" != "Train" ] || [ "$GB" != "Avion" ]; then
  echo "ÉCHEC : le pod de survey n'a pas reçu les deux options (trouvé : « $GA » et « $GB »)."
  echo "  Si la ConfigMap a changé après le démarrage du pod, il faut le redémarrer."
  exit 1
fi
echo "RÉUSSI : survey tourne et lit Train et Avion dans survey-config."
```

````solution
```bash
mkdir -p ~/byte-layers/kubernetes/l5/challenge
cat > ~/byte-layers/kubernetes/l5/challenge/survey.config.yml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: survey-config
data:
  OPTION_A: Train
  OPTION_B: Avion
EOF
cat > ~/byte-layers/kubernetes/l5/challenge/survey.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: survey
  labels:
    app: survey
spec:
  replicas: 1
  selector:
    matchLabels:
      app: survey
  template:
    metadata:
      labels:
        app: survey
    spec:
      containers:
        - name: survey
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: survey-config
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l5/challenge/survey.config.yml -f ~/byte-layers/kubernetes/l5/challenge/survey.deploy.yml
kubectl rollout status deployment/survey --timeout=50s
```
````

# Tâche 2 : changer un réglage

## Faire lire une nouvelle option au pod qui tourne

L'équipe change d'avis : la seconde option devient `Bateau`. Le pod de `survey` doit répondre par cette nouvelle valeur, sans que le fichier du Deployment change.

**Critères :**

- dans la ConfigMap `survey-config`, `OPTION_B` vaut `Bateau`, et `OPTION_A` vaut toujours `Train` ;
- le pod de `survey` qui tourne a `OPTION_B=Bateau` dans ses variables d'environnement ;
- le pod est prêt.

```hint
Modifier la ConfigMap ne suffit pas : demandez au pod sa variable, vous verrez laquelle il connaît.
```

```hint
Une variable d'environnement est lue une seule fois, au démarrage du conteneur. Une sous-commande de `kubectl rollout` refait une mise à jour progressive sans rien changer d'autre.
```

```verify
A=$(kubectl get configmap survey-config -o jsonpath='{.data.OPTION_A}' 2>/dev/null)
B=$(kubectl get configmap survey-config -o jsonpath='{.data.OPTION_B}' 2>/dev/null)
if [ "$A" != "Train" ] || [ "$B" != "Bateau" ]; then
  echo "ÉCHEC : survey-config doit porter OPTION_A=Train et OPTION_B=Bateau (trouvé : « $A » et « $B »)."
  exit 1
fi
if ! kubectl rollout status deployment/survey --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : le pod de survey n'est pas prêt. Regardez son état : kubectl get pods -l app=survey"
  exit 1
fi
GB=""
for i in 1 2 3 4 5; do
  GB=$(kubectl exec deployment/survey -- printenv OPTION_B 2>/dev/null)
  if [ "$GB" = "Bateau" ]; then break; fi
  sleep 2
done
if [ "$GB" != "Bateau" ]; then
  echo "ÉCHEC : la ConfigMap dit Bateau, mais le pod de survey tourne encore avec « $GB »."
  echo "  Il a lu ses variables à son démarrage : il lui faut un redémarrage."
  exit 1
fi
echo "RÉUSSI : le pod de survey a redémarré et lit Bateau."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l5/challenge/survey.config.yml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: survey-config
data:
  OPTION_A: Train
  OPTION_B: Bateau
EOF
kubectl apply -f ~/byte-layers/kubernetes/l5/challenge/survey.config.yml
kubectl rollout restart deployment survey
kubectl rollout status deployment/survey --timeout=50s
```
````

# Tâche 3 : un mot de passe hors du Deployment

## Donner ADMIN_TOKEN à survey par un Secret

Le sondage aura bientôt une page d'administration, protégée par un jeton que l'application lira dans la variable d'environnement `ADMIN_TOKEN`. Donnez-lui ce jeton, qui vaut `layers-2026`, sans l'écrire dans le Deployment. Le pod doit continuer de lire ses options dans la ConfigMap.

**Critères :**

- un Secret s'appelle exactement `survey-credentials`, et sa clé `ADMIN_TOKEN` vaut `layers-2026` une fois décodée ;
- le conteneur de `survey` reçoit `ADMIN_TOKEN` **depuis le Secret** : la valeur `layers-2026` n'apparaît pas dans le Deployment ;
- le pod de `survey` qui tourne a `ADMIN_TOKEN=layers-2026` et toujours `OPTION_A=Train` ;
- le pod est prêt.

```hint
Un Secret s'écrit comme une ConfigMap, avec `kind: Secret`, `type: Opaque`, et les valeurs en clair sous `stringData`.
```

```hint
`envFrom` est une liste : à côté de `- configMapRef:`, un second élément `- secretRef:` importe les clés d'un Secret.
```

```hint
Appliquez le Secret **avant** le Deployment modifié : sinon le nouveau pod reste en `CreateContainerConfigError` tant que le Secret n'existe pas.
```

```verify
if ! kubectl get secret survey-credentials >/dev/null 2>&1; then
  echo "ÉCHEC : aucun Secret ne s'appelle survey-credentials dans le namespace courant."
  exit 1
fi
T=$(kubectl get secret survey-credentials -o 'go-template={{with .data}}{{with .ADMIN_TOKEN}}{{. | base64decode}}{{end}}{{end}}' 2>/dev/null)
if [ "$T" != "layers-2026" ]; then
  echo "ÉCHEC : la clé ADMIN_TOKEN de survey-credentials ne vaut pas layers-2026 une fois décodée (trouvé : « $T »)."
  exit 1
fi
REF=$(kubectl get deployment survey -o jsonpath='{.spec.template.spec.containers[0].envFrom[*].secretRef.name} {.spec.template.spec.containers[0].env[*].valueFrom.secretKeyRef.name}' 2>/dev/null)
case " $REF " in
  *" survey-credentials "*) : ;;
  *) echo "ÉCHEC : le Deployment survey ne lit pas le Secret survey-credentials."
     exit 1 ;;
esac
if kubectl get deployment survey -o jsonpath='{.spec.template}' | grep -q 'layers-2026'; then
  echo "ÉCHEC : la valeur layers-2026 est écrite dans le Deployment survey. Elle ne doit se trouver que dans le Secret."
  exit 1
fi
if ! kubectl rollout status deployment/survey --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : le pod de survey n'est pas prêt. Regardez son état : kubectl get pods -l app=survey"
  echo "  En CreateContainerConfigError, le bas de kubectl describe pod -l app=survey dit quel objet manque."
  exit 1
fi
GT=""; GA=""
for i in 1 2 3 4 5; do
  GT=$(kubectl exec deployment/survey -- printenv ADMIN_TOKEN 2>/dev/null)
  GA=$(kubectl exec deployment/survey -- printenv OPTION_A 2>/dev/null)
  if [ "$GT" = "layers-2026" ] && [ "$GA" = "Train" ]; then break; fi
  sleep 2
done
if [ "$GT" != "layers-2026" ]; then
  echo "ÉCHEC : le pod de survey n'a pas ADMIN_TOKEN=layers-2026 (trouvé : « $GT »)."
  exit 1
fi
if [ "$GA" != "Train" ]; then
  echo "ÉCHEC : le pod de survey a perdu OPTION_A : il doit lire à la fois la ConfigMap et le Secret."
  exit 1
fi
echo "RÉUSSI : survey lit son jeton dans survey-credentials, et ses options dans survey-config."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l5/challenge/survey.secret.yml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: survey-credentials
type: Opaque
stringData:
  ADMIN_TOKEN: layers-2026
EOF
cat > ~/byte-layers/kubernetes/l5/challenge/survey.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: survey
  labels:
    app: survey
spec:
  replicas: 1
  selector:
    matchLabels:
      app: survey
  template:
    metadata:
      labels:
        app: survey
    spec:
      containers:
        - name: survey
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: survey-config
            - secretRef:
                name: survey-credentials
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l5/challenge/survey.secret.yml -f ~/byte-layers/kubernetes/l5/challenge/survey.deploy.yml
kubectl rollout status deployment/survey --timeout=50s
```
````

# Tâche 4 : lire un Secret qu'on n'a pas écrit

## Retrouver la valeur rangée dans survey-vault

```setup
mkdir -p ~/byte-layers/kubernetes/l5/challenge
kubectl get secret survey-vault >/dev/null 2>&1 || kubectl create secret generic survey-vault --from-literal=code="layers-$(date +%s)"
```

Un Secret `survey-vault` vient d'être créé pour vous dans le namespace, avec une seule clé, `code`, dont personne ne vous a donné la valeur. Retrouvez-la, et écrivez-la en clair dans le fichier `~/byte-layers/kubernetes/l5/challenge/code.txt`, en y envoyant la réponse de `kubectl` avec `>`, ou en l'y collant avec votre éditeur.

**Critères :**

- le fichier `~/byte-layers/kubernetes/l5/challenge/code.txt` existe ;
- il contient la valeur **décodée** de la clé `code` du Secret `survey-vault`, et rien d'autre.

```hint
`kubectl get secret survey-vault -o yaml` montre la valeur telle que le cluster la garde. Elle n'est pas chiffrée.
```

```hint
Elle est encodée en base64. `kubectl` sait la décoder lui-même, avec `-o "go-template=…"` et la fonction `base64decode`.
```

```verify
if ! kubectl get secret survey-vault >/dev/null 2>&1; then
  echo "ÉCHEC : le Secret survey-vault n'existe pas dans le namespace courant."
  echo "  Rouvrez cette tâche pour qu'il soit créé, ou créez-le : kubectl create secret generic survey-vault --from-literal=code=layers-1234"
  exit 1
fi
F="$HOME/byte-layers/kubernetes/l5/challenge/code.txt"
if [ ! -f "$F" ]; then
  echo "ÉCHEC : le fichier $F n'existe pas."
  exit 1
fi
WANT=$(kubectl get secret survey-vault -o 'go-template={{.data.code | base64decode}}')
GOT=$(tr -d ' \n\r\t%' < "$F")
if [ "$GOT" = "$(kubectl get secret survey-vault -o jsonpath='{.data.code}')" ]; then
  echo "ÉCHEC : le fichier contient la valeur encodée en base64. Il reste à la décoder."
  exit 1
fi
if [ -z "$WANT" ] || [ "$GOT" != "$WANT" ]; then
  echo "ÉCHEC : le contenu du fichier n'est pas la valeur décodée de la clé code."
  exit 1
fi
echo "RÉUSSI : vous avez lu le Secret survey-vault sans aucune clé. Sa protection vient des droits d'accès."
```

````solution
```bash
kubectl get secret survey-vault -o "go-template={{.data.code | base64decode}}" > ~/byte-layers/kubernetes/l5/challenge/code.txt
```
````

# Tâche 5 : nettoyer

## Supprimer tout ce que le challenge a créé

Supprimez tout ce que ce challenge a créé, et rien d'autre : l'application de vote doit rester intacte.

**Critères :**

- le Deployment `survey` et ses pods n'existent plus ;
- la ConfigMap `survey-config` et les Secrets `survey-credentials` et `survey-vault` n'existent plus ;
- le dossier `~/byte-layers/kubernetes/l5/challenge` n'existe plus.

```hint
`kubectl delete` accepte un type et un nom, ou `-f` suivi d'un fichier : il supprime alors ce que le fichier décrit.
```

```hint
`survey-vault` n'a pas de fichier : supprimez-le par son nom, avec `kubectl delete secret`. Le dossier se supprime avec `rm -rf`.
```

```verify
if ! kubectl get namespace >/dev/null 2>&1; then
  echo "ÉCHEC : kubectl ne joint pas le cluster : impossible de vérifier le nettoyage."
  exit 1
fi
LEFT=""
kubectl get deployment survey >/dev/null 2>&1 && LEFT="$LEFT deployment/survey"
kubectl get configmap survey-config >/dev/null 2>&1 && LEFT="$LEFT configmap/survey-config"
kubectl get secret survey-credentials >/dev/null 2>&1 && LEFT="$LEFT secret/survey-credentials"
kubectl get secret survey-vault >/dev/null 2>&1 && LEFT="$LEFT secret/survey-vault"
if [ -n "$LEFT" ]; then
  echo "ÉCHEC : il reste des objets du challenge :$LEFT"
  exit 1
fi
if [ -n "$(kubectl get pods -l app=survey -o name 2>/dev/null)" ]; then
  echo "ÉCHEC : un pod de survey est encore en train de s'arrêter. Réessayez dans quelques secondes."
  exit 1
fi
if [ -e "$HOME/byte-layers/kubernetes/l5/challenge" ]; then
  echo "ÉCHEC : le dossier ~/byte-layers/kubernetes/l5/challenge existe encore."
  exit 1
fi
echo "RÉUSSI : il ne reste rien du challenge."
```

````solution
```bash
kubectl delete deployment survey
kubectl delete configmap survey-config
kubectl delete secret survey-credentials survey-vault
kubectl wait --for=delete pod -l app=survey --timeout=50s
rm -rf ~/byte-layers/kubernetes/l5/challenge
```
````
