---
title: "Exposer une application, sans guide"
description: "Donner un Service à une application, réparer un Service qui ne mène nulle part et ouvrir une porte d'entrée par nom de domaine, à partir d'un cahier des charges."
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

Ce challenge prouve que vous savez rendre une application joignable : à l'intérieur du cluster par un Service, et depuis l'extérieur par un Ingress. Il n'y a **aucune commande de solution à recopier** : chaque tâche donne un objectif et des critères, puis un bouton vérifie l'état réel du cluster, y compris par un appel lancé depuis l'intérieur. Tout fichier et toute suite de commandes qui produisent le bon résultat sont acceptés.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : ce qu'enseignent [Donner une adresse stable avec un Service](https://byte-layers.com/fr/skills/k8s-services) et [Ouvrir une porte d'entrée avec un Ingress](https://byte-layers.com/fr/skills/k8s-ingress). Le lab [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app) en donne un exemple complet.
- **Ce qu'il faut avoir** : un cluster qui répond à `kubectl get nodes`, avec son contrôleur d'Ingress, celui que le cours sur l'Ingress fait activer, et un éditeur de texte. Aucun port de votre machine n'est nécessaire, sauf le **28639** si vous essayez vous-même la tâche 3 par un tunnel.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l3/challenge` est créé à la première étape, et la dernière tâche le supprime. Écrivez-y vos fichiers : dans Kubernetes, le fichier fait foi.
- **Les noms comptent** : les vérifications cherchent les noms exacts, qui commencent tous par `survey`. Rien d'autre n'est inspecté ni supprimé dans votre cluster : l'application `vote` et ses voisins ne sont pas concernés.
- **L'ordre compte** : chaque tâche reprend ce que la précédente a laissé. Faites-les dans l'ordre.

# Préparer l'application à exposer

## Déployer survey, deux pods sans adresse stable

Cette étape est un **support** : elle met en place l'application que vous allez exposer, et ses commandes se lancent telles quelles. Créez d'abord le dossier de travail :

```exec
mkdir -p ~/byte-layers/kubernetes/l3/challenge
```

Écrivez ensuite le fichier du Deployment `survey`. C'est l'application de vote sous un autre nom, en 2 réplicas : ses pods portent l'étiquette `app: survey`, écoutent sur le port 80, et leur page `/healthz` renvoie le nom du pod qui répond.

```exec
cat > ~/byte-layers/kubernetes/l3/challenge/survey.deploy.yml <<'EOF'
# survey : l'application à exposer, 2 réplicas qui écoutent sur le port 80
apiVersion: apps/v1
kind: Deployment
metadata:
  name: survey
  labels:
    app: survey
spec:
  replicas: 2
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
> | `cat > …/survey.deploy.yml` | écrit dans ce fichier ce que `cat` reçoit en entrée, en remplaçant son contenu s'il existe déjà |
> | `<<'EOF'` | donne en entrée à `cat` toutes les lignes qui suivent, telles quelles, jusqu'à la ligne `EOF` |
> | `EOF` | marque la fin du contenu ; ce mot n'est pas écrit dans le fichier |

Transmettez-le au cluster, puis attendez que les 2 pods soient prêts :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l3/challenge/survey.deploy.yml
kubectl rollout status deployment/survey --timeout=120s
```

```ack
survey tourne avec ses 2 pods
```

# Tâche 1 : une adresse stable pour survey

## Créer un Service qui écoute sur le port 8080

Les 2 pods de `survey` tournent, mais aucun autre pod ne peut les joindre par un nom. Donnez-leur un Service.

**Critères :**

- un Service s'appelle exactement `survey` ;
- il écoute sur le port **8080** et renvoie les requêtes vers le port **80** des pods ;
- il sert les **2 pods** de `survey`, et eux seuls ;
- depuis un pod du cluster, l'adresse `http://survey:8080/healthz` répond.

```hint
Un Service repère ses pods par leur étiquette, avec `spec.selector`. Celle des pods de `survey` est écrite dans `survey.deploy.yml`.
```

```hint
Le port d'écoute du Service et le port des pods sont deux champs différents : `port` et `targetPort`. Ici, ils n'ont pas la même valeur.
```

```hint
Pour essayer vous-même, lancez un pod de test avec `kubectl run`, comme dans le cours sur le Service, en appelant `http://survey:8080/healthz`. `kubectl describe service survey` doit montrer deux adresses dans `Endpoints`.
```

```verify
S=survey
if ! kubectl get deployment survey >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment survey n'existe pas. Jouez d'abord l'étape « Préparer l'application à exposer »."
  exit 1
fi
if ! kubectl get service "$S" >/dev/null 2>&1; then
  echo "ÉCHEC : aucun Service ne s'appelle $S."
  echo "  Écrivez-le dans un fichier, puis transmettez ce fichier au cluster."
  exit 1
fi
PORT=$(kubectl get service "$S" -o jsonpath='{.spec.ports[0].port}')
if [ "$PORT" != "8080" ]; then
  echo "ÉCHEC : le Service $S écoute sur le port ${PORT:-?}, et non sur le port 8080."
  exit 1
fi
PODS=$(kubectl get endpointslices -l kubernetes.io/service-name="$S" -o jsonpath='{range .items[*].endpoints[*]}{.targetRef.name}{"\n"}{end}' | grep .)
N=$(printf '%s\n' "$PODS" | grep -c '^survey-')
OTHER=$(printf '%s\n' "$PODS" | grep -v '^survey-' | grep -c .)
if [ "$N" != "2" ] || [ "$OTHER" != "0" ]; then
  echo "ÉCHEC : le Service $S sert $N pod(s) de survey et $OTHER autre(s) pod(s). Il doit servir les 2 pods de survey, et eux seuls."
  echo "  Son selector est : $(kubectl get service "$S" -o jsonpath='{.spec.selector}')"
  echo "  L'étiquette des pods se lit avec : kubectl get pods --show-labels"
  exit 1
fi
OUT=""
for i in 1 2 3; do
  kubectl delete pod l3-survey-check --ignore-not-found >/dev/null 2>&1
  OUT=$(kubectl run l3-survey-check --rm -i --quiet --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- wget -qO- -T 5 http://survey:8080/healthz 2>/dev/null)
  case "$OUT" in *'"hostname":"survey-'*) break ;; esac
  sleep 2
done
case "$OUT" in
  *'"hostname":"survey-'*) : ;;
  *) echo "ÉCHEC : depuis un pod du cluster, http://survey:8080/healthz ne répond pas."
     echo "  Le Service sert bien les 2 pods : regardez vers quel port des pods il renvoie (targetPort). Ils écoutent sur le port 80."
     exit 1 ;;
esac
echo "RÉUSSI : http://survey:8080 mène aux 2 pods de survey."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l3/challenge/survey.svc.yml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: survey
  labels:
    app: survey
spec:
  selector:
    app: survey
  ports:
    - port: 8080
      targetPort: 80
EOF
kubectl apply -f ~/byte-layers/kubernetes/l3/challenge/survey.svc.yml
```
````

# Tâche 2 : un Service qui ne mène nulle part

## Trouver pourquoi survey-legacy ne répond pas, et le réparer

Un collègue a écrit un second Service pour la même application, `survey-legacy`, pour un ancien programme qui appelle encore ce nom-là sur le port 80. Il ne répond pas. Cette commande, la seule de la tâche, écrit son fichier et le transmet au cluster tel qu'il vous a été laissé :

```exec
cat > ~/byte-layers/kubernetes/l3/challenge/survey-legacy.svc.yml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: survey-legacy
spec:
  selector:
    app: servey
  ports:
    - port: 80
      targetPort: 8080
EOF
kubectl apply -f ~/byte-layers/kubernetes/l3/challenge/survey-legacy.svc.yml
```

Trouvez ce qui ne va pas, corrigez le fichier `survey-legacy.svc.yml` et transmettez-le de nouveau. Il y a peut-être plus d'une erreur.

**Critères :**

- le Service `survey-legacy` existe toujours, et écoute toujours sur le port **80** ;
- il sert les **2 pods** de `survey` ;
- depuis un pod du cluster, l'adresse `http://survey-legacy/healthz` répond ;
- le Deployment `survey` et ses pods n'ont pas été modifiés : c'est le Service qui est faux.

```hint
Demandez le détail du Service et lisez sa ligne `Endpoints` : elle dit à quels pods il transmet les requêtes. Vide, elle désigne le premier coupable.
```

```hint
Comparez le `selector` du Service, lettre à lettre, avec l'étiquette des pods : `kubectl get pods --show-labels` l'affiche.
```

```hint
Une fois les `Endpoints` remplis, regardez le port qui suit chaque adresse : c'est le `targetPort`. Les pods de `survey` écoutent-ils sur ce port ?
```

```verify
S=survey-legacy
if ! kubectl get service "$S" >/dev/null 2>&1; then
  echo "ÉCHEC : le Service $S n'existe pas. Lancez la commande de cette tâche, puis corrigez son fichier."
  exit 1
fi
PORT=$(kubectl get service "$S" -o jsonpath='{.spec.ports[0].port}')
if [ "$PORT" != "80" ]; then
  echo "ÉCHEC : le Service $S écoute sur le port ${PORT:-?} : l'ancien programme l'appelle sur le port 80, ne changez pas ce port-là."
  exit 1
fi
LABEL=$(kubectl get deployment survey -o jsonpath='{.spec.template.metadata.labels}' 2>/dev/null)
case "$LABEL" in
  *servey*) echo "ÉCHEC : les pods de survey ont reçu l'étiquette du Service. C'est le Service qu'il faut corriger : remettez survey.deploy.yml tel qu'il était, puis kubectl apply."
            exit 1 ;;
esac
PODS=$(kubectl get endpointslices -l kubernetes.io/service-name="$S" -o jsonpath='{range .items[*].endpoints[*]}{.targetRef.name}{"\n"}{end}' | grep .)
N=$(printf '%s\n' "$PODS" | grep -c '^survey-')
if [ "$N" != "2" ]; then
  echo "ÉCHEC : le Service $S sert $N pod(s) de survey, et non 2."
  echo "  Son selector est : $(kubectl get service "$S" -o jsonpath='{.spec.selector}')"
  exit 1
fi
OUT=""
for i in 1 2 3; do
  kubectl delete pod l3-survey-check --ignore-not-found >/dev/null 2>&1
  OUT=$(kubectl run l3-survey-check --rm -i --quiet --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- wget -qO- -T 5 http://survey-legacy/healthz 2>/dev/null)
  case "$OUT" in *'"hostname":"survey-'*) break ;; esac
  sleep 2
done
case "$OUT" in
  *'"hostname":"survey-'*) : ;;
  *) echo "ÉCHEC : le Service $S sert bien les 2 pods, mais http://survey-legacy/healthz ne répond pas depuis un pod du cluster."
     echo "  Il renvoie vers le port $(kubectl get service "$S" -o jsonpath='{.spec.ports[0].targetPort}') des pods. Sur quel port écoutent-ils ?"
     exit 1 ;;
esac
echo "RÉUSSI : survey-legacy mène de nouveau aux 2 pods de survey."
```

````solution
Deux erreurs : le `selector` disait `servey` au lieu de `survey`, donc le Service ne servait aucun pod ; et le `targetPort` valait 8080, alors que les pods écoutent sur le port 80.

```bash
cat > ~/byte-layers/kubernetes/l3/challenge/survey-legacy.svc.yml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: survey-legacy
spec:
  selector:
    app: survey
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl apply -f ~/byte-layers/kubernetes/l3/challenge/survey-legacy.svc.yml
```
````

# Tâche 3 : une porte d'entrée par nom de domaine

## Écrire un Ingress pour survey.example.com

Ouvrez `survey` aux visites venues de l'extérieur, mais seulement à celles qui demandent le nom `survey.example.com`. Les autres règles du cluster, s'il y en a, ne doivent pas être gênées.

**Critères :**

- un Ingress s'appelle exactement `survey` ;
- sa règle vaut pour le nom de domaine `survey.example.com`, sur le chemin `/` et tout ce qui le suit ;
- elle envoie les visites vers le Service `survey`, sur le port où il écoute ;
- une visite reçue par le contrôleur d'Ingress sous le nom `survey.example.com` arrive à un pod de `survey`.

```hint
Le champ qui limite une règle à un nom de domaine est `host`. Il s'écrit dans la même règle que `http` : la ligne `- host: survey.example.com` porte le tiret, et la ligne `http:` vient dessous, sans tiret, alignée sur le mot `host`. Deux tirets feraient deux règles.
```

```hint
Le port à donner dans `backend.service.port.number` est celui du Service, pas celui des pods : relisez la tâche 1.
```

```hint
Pour essayer vous-même, ouvrez un tunnel vers le contrôleur comme dans le cours sur l'Ingress, sur le port 28639, puis demandez la page en annonçant le nom voulu : `curl -H "Host: survey.example.com" http://localhost:28639/healthz`. Arrêtez le tunnel ensuite.
```

```verify
I=survey
if ! kubectl get ingress "$I" >/dev/null 2>&1; then
  echo "ÉCHEC : aucun Ingress ne s'appelle $I."
  echo "  Si kubectl apply répond « failed calling webhook », le contrôleur d'Ingress n'est pas prêt : attendez une minute et relancez kubectl apply."
  echo "  S'il n'a jamais été activé, reprenez la section du cours sur l'Ingress qui l'active."
  exit 1
fi
HOSTS=$(kubectl get ingress "$I" -o jsonpath='{range .spec.rules[*]}[{.host}]{end}')
if [ "$HOSTS" != "[survey.example.com]" ]; then
  echo "ÉCHEC : l'Ingress $I doit avoir une seule règle, pour le nom survey.example.com. Trouvé : ${HOSTS:-aucune règle} ([] veut dire « tous les noms »)."
  exit 1
fi
SVC=$(kubectl get ingress "$I" -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}')
PORT=$(kubectl get ingress "$I" -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.port.number}')
PATHV=$(kubectl get ingress "$I" -o jsonpath='{.spec.rules[0].http.paths[0].path}')
PTYPE=$(kubectl get ingress "$I" -o jsonpath='{.spec.rules[0].http.paths[0].pathType}')
if [ "$SVC" != "survey" ] || [ "$PORT" != "8080" ]; then
  echo "ÉCHEC : la règle envoie les visites vers le Service « ${SVC:-vide} », port ${PORT:-vide}. Il faut le Service survey, sur son port d'écoute, 8080."
  exit 1
fi
if [ "$PATHV" != "/" ] || [ "$PTYPE" = "Exact" ]; then
  echo "ÉCHEC : la règle porte sur le chemin « ${PATHV:-vide} » avec pathType ${PTYPE:-vide}. Il faut le chemin / et tout ce qui le suit : pathType Prefix."
  exit 1
fi
if kubectl get service -n ingress-nginx ingress-nginx-controller >/dev/null 2>&1; then
  URL=http://ingress-nginx-controller.ingress-nginx/healthz
elif kubectl get service -n kube-system traefik >/dev/null 2>&1; then
  URL=http://traefik.kube-system/healthz
else
  echo "ÉCHEC : aucun contrôleur d'Ingress n'a été trouvé dans le cluster. Le cours sur l'Ingress montre comment l'activer."
  exit 1
fi
OUT=""
for i in 1 2 3; do
  kubectl delete pod l3-survey-check --ignore-not-found >/dev/null 2>&1
  OUT=$(kubectl run l3-survey-check --rm -i --quiet --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- wget -qO- -T 5 --header "Host: survey.example.com" "$URL" 2>/dev/null)
  case "$OUT" in *'"hostname":"survey-'*) break ;; esac
  sleep 3
done
case "$OUT" in
  *'"hostname":"survey-'*) : ;;
  *) echo "ÉCHEC : la règle est bien écrite, mais une visite envoyée au contrôleur sous le nom survey.example.com n'arrive pas à survey."
     echo "  Vérifiez que le Service survey répond encore (tâche 1), attendez quelques secondes, puis réessayez."
     exit 1 ;;
esac
echo "RÉUSSI : survey.example.com mène à survey par la porte d'entrée."
```

````solution
```bash
cat > ~/byte-layers/kubernetes/l3/challenge/survey.ingress.yml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: survey
spec:
  rules:
    - host: survey.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: survey
                port:
                  number: 8080
EOF
kubectl apply -f ~/byte-layers/kubernetes/l3/challenge/survey.ingress.yml
```
````

# Tâche 4 : ne rien laisser derrière soi

## Supprimer les objets survey et le dossier

Supprimez tout ce que ce challenge a créé : l'Ingress `survey`, les Services `survey` et `survey-legacy`, le Deployment `survey`, et le dossier `~/byte-layers/kubernetes/l3/challenge`.

**Critères :** plus aucun de ces objets dans le cluster, plus de dossier. Vos autres objets, `vote` et ses voisins, ne sont pas concernés, ni les autres fichiers de `~/byte-layers/kubernetes/l3`.

```hint
`kubectl delete -f` accepte un dossier : il supprime alors ce que décrivent tous les fichiers qu'il contient. Faites-le avant d'effacer le dossier.
```

```hint
Pour un dossier et tout ce qu'il contient : `rm -r`, en vérifiant bien le chemin avant de valider.
```

```verify
left=""
kubectl get ingress survey >/dev/null 2>&1 && left="$left ingress/survey"
kubectl get service survey >/dev/null 2>&1 && left="$left service/survey"
kubectl get service survey-legacy >/dev/null 2>&1 && left="$left service/survey-legacy"
kubectl get deployment survey >/dev/null 2>&1 && left="$left deployment/survey"
if ! kubectl get namespaces >/dev/null 2>&1; then
  echo "ÉCHEC : le cluster ne répond pas. Vérifiez kubectl get nodes, puis réessayez."
  exit 1
fi
if [ -n "$left" ]; then
  echo "ÉCHEC : il reste :$left"
  echo "  Supprimez chacun avec kubectl delete suivi de ce nom, par exemple : kubectl delete service/survey"
  exit 1
fi
if [ -e "$HOME/byte-layers/kubernetes/l3/challenge" ]; then
  echo "ÉCHEC : le dossier ~/byte-layers/kubernetes/l3/challenge existe encore."
  exit 1
fi
echo "RÉUSSI : tout est propre. Vous savez exposer une application."
```

````solution
```bash
kubectl delete -f ~/byte-layers/kubernetes/l3/challenge
rm -r ~/byte-layers/kubernetes/l3/challenge
```
````
