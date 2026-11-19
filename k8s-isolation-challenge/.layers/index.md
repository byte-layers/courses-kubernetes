---
title: "Isoler une application, sans guide"
description: "Ranger une application dans son namespace, la plafonner avec un quota et fermer son réseau avec des NetworkPolicies, à partir d'un cahier des charges, sans guide."
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

Ce challenge prouve que vous savez isoler une application dans un cluster : la ranger dans son namespace, plafonner ce namespace, puis fermer son réseau. Il n'y a **aucune commande à recopier** : chaque tâche donne un objectif et des critères, puis un bouton vérifie l'état réel du cluster. Toute suite de commandes qui produit le bon résultat est acceptée.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : ce qu'enseignent [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas) et [Fermer le réseau avec des NetworkPolicies](https://byte-layers.com/fr/skills/k8s-network-policies). La suppression d'un namespace est montrée dans [Dans un cluster, le réseau est ouvert par défaut](https://byte-layers.com/fr/bits/k8s-network-open-by-default).
- **Ce qu'il faut avoir** : un cluster `layers-k8s` auquel `kubectl` répond, et qui sait appliquer des règles réseau, et un éditeur de texte. Aucun port n'est utilisé.
- **L'application** : une petite boutique en deux morceaux. `front` est sa page web, sur le port 80, et `cache` est la base Redis où elle range ses données, sur le port 6379. Leurs Deployments et leurs Services sont décrits dans un seul fichier, créé pour vous : `~/byte-layers/kubernetes/l4/challenge/shop.yml`. Il ne cite aucun namespace.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l4/challenge` est créé pour vous, et la dernière tâche le supprime.
- **Les noms comptent** : tout se passe dans un namespace nommé exactement `layers-l4-shop`. Rien d'autre n'est modifié dans votre cluster : votre application de vote et le namespace `vote-app` ne sont pas concernés.
- **Le namespace courant** : laissez `kubectl` travailler dans le namespace où il est, et adressez vos commandes à `layers-l4-shop` avec l'option `-n`. Les vérifications lancent un court pod de test dans votre namespace courant.
- **L'ordre compte** : chaque tâche reprend le résultat de la précédente. Faites-les dans l'ordre.

# Tâche 1 : un namespace pour la boutique

## Déployer la boutique dans son propre namespace

```setup
mkdir -p ~/byte-layers/kubernetes/l4/challenge
cat > ~/byte-layers/kubernetes/l4/challenge/shop.yml <<'EOF'
# front : la page web de la boutique. 2 réplicas, joignables sous le nom "front".
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front
  labels:
    app: front
spec:
  replicas: 2
  selector:
    matchLabels:
      app: front
  template:
    metadata:
      labels:
        app: front
    spec:
      containers:
        - name: front
          image: ghcr.io/bngams/kube-vote:1.0
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
  name: front
  labels:
    app: front
spec:
  selector:
    app: front
  ports:
    - port: 80
      targetPort: 80
--- # objet suivant
# cache : la base Redis où la boutique range ses données. Joignable sous le nom "cache".
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
  labels:
    app: cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
        - name: cache
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
  name: cache
  labels:
    app: cache
spec:
  selector:
    app: cache
  ports:
    - port: 6379
      targetPort: 6379
EOF
```

Déployez la boutique décrite dans `~/byte-layers/kubernetes/l4/challenge/shop.yml`, sans modifier ce fichier, dans un namespace qui n'appartient qu'à elle.

**Critères :**

- un namespace s'appelle exactement `layers-l4-shop` ;
- il contient le Deployment `front`, avec ses **2** exemplaires prêts, et le Service `front` ;
- il contient le Deployment `cache`, avec son exemplaire prêt, et le Service `cache` ;
- aucun autre namespace ne contient de Deployment `front` ou `cache`.

```hint
Deux gestes suffisent : créer le namespace, puis appliquer le fichier en l'adressant à ce namespace.
```

```hint
Un fichier qui ne cite aucun namespace part dans le namespace courant, sauf si la commande en désigne un autre avec `-n`.
```

```hint
`kubectl create namespace layers-l4-shop`, puis `kubectl apply -f` suivi du chemin du fichier et de `-n layers-l4-shop`. Si la boutique est partie dans un autre namespace, supprimez-la de là avec `kubectl delete -f`, le même chemin et `-n` suivi de ce namespace.
```

```verify
NS=layers-l4-shop
if ! kubectl get namespace kube-system >/dev/null 2>&1; then
  echo "ÉCHEC : kubectl ne répond pas. Vérifiez que votre cluster est démarré, puis réessayez."
  exit 1
fi
if [ "$(kubectl get namespace "$NS" -o jsonpath='{.status.phase}' 2>/dev/null)" != "Active" ]; then
  echo "ÉCHEC : aucun namespace actif ne s'appelle $NS."
  exit 1
fi
for o in deployment/front service/front deployment/cache service/cache; do
  if ! kubectl get "$o" -n "$NS" >/dev/null 2>&1; then
    echo "ÉCHEC : $o n'est pas dans le namespace $NS."
    echo "  Appliquez ~/byte-layers/kubernetes/l4/challenge/shop.yml en l'adressant à ce namespace."
    exit 1
  fi
done
stray=$(kubectl get deployments -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\n"}{end}' | grep -E '/(front|cache)$' | grep -v "^$NS/" | tr '\n' ' ')
if [ -n "$stray" ]; then
  echo "ÉCHEC : la boutique est aussi déployée ailleurs : $stray"
  echo "  Supprimez ces copies : la boutique ne vit que dans $NS."
  exit 1
fi
if ! kubectl wait --for=condition=Available deployment front cache -n "$NS" --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : front et cache sont dans $NS, mais leurs pods ne sont pas tous prêts."
  echo "  Regardez kubectl get pods -n $NS, attendez un peu, puis réessayez."
  exit 1
fi
ready=$(kubectl get deployment front -n "$NS" -o jsonpath='{.status.readyReplicas}')
if [ "$ready" != "2" ]; then
  echo "ÉCHEC : front a $ready exemplaire(s) prêt(s), au lieu de 2. Le fichier en demande 2 : appliquez-le sans le modifier."
  exit 1
fi
echo "RÉUSSI : la boutique tourne dans $NS, avec 2 front et 1 cache."
```

````solution
```bash
kubectl create namespace layers-l4-shop
kubectl apply -f ~/byte-layers/kubernetes/l4/challenge/shop.yml -n layers-l4-shop
kubectl wait --for=condition=Available deployment --all -n layers-l4-shop --timeout=50s
```
````

# Tâche 2 : un plafond pour la boutique

## Poser un quota de 5 pods et le voir refuser

L'équipe de la boutique a droit à **5 pods**, pas un de plus. Posez ce plafond sur son namespace, puis prouvez qu'il tient : demandez **6** exemplaires de `front` et laissez Kubernetes en refuser une partie.

**Critères :**

- un quota du namespace `layers-l4-shop` plafonne le nombre de pods à **5** ;
- le Deployment `front` **demande** 6 exemplaires ;
- il n'en a obtenu que **4** : avec `cache`, le namespace compte 5 pods, son plafond.

Les événements du namespace donnent la raison du refus : `kubectl get events`, adressé à ce namespace, suffit à la lire.

```hint
Un quota se crée en une commande, avec un nom de votre choix et un plafond écrit `ressource=valeur`. Adressez-la au bon namespace.
```

```hint
`kubectl create quota <nom> --hard=pods=5 -n layers-l4-shop` pose le plafond. Pour demander 6 exemplaires sans toucher au fichier, il existe un raccourci qui change le nombre de réplicas d'un Deployment.
```

```hint
`kubectl scale deployment front --replicas=6 -n layers-l4-shop`, puis `kubectl get events -n layers-l4-shop --field-selector reason=FailedCreate` pour lire le refus.
```

```verify
NS=layers-l4-shop
if ! kubectl get deployment front -n "$NS" >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment front n'est pas dans $NS. Faites d'abord la tâche 1."
  exit 1
fi
hard=$(kubectl get resourcequota -n "$NS" -o jsonpath='{range .items[*]}{.spec.hard.pods}{"\n"}{end}' 2>/dev/null | grep .)
if [ -z "$hard" ]; then
  echo "ÉCHEC : aucun quota ne plafonne le nombre de pods dans $NS."
  exit 1
fi
if ! echo "$hard" | grep -qx 5; then
  echo "ÉCHEC : le plafond de pods de $NS n'est pas 5 (trouvé : $(echo $hard))."
  echo "  Supprimez ce quota et recréez-le avec le bon plafond."
  exit 1
fi
want=$(kubectl get deployment front -n "$NS" -o jsonpath='{.spec.replicas}')
if [ "$want" != "6" ]; then
  echo "ÉCHEC : le Deployment front demande $want exemplaire(s), pas 6."
  exit 1
fi
have=""
for i in 1 2 3 4 5 6 7 8 9 10; do
  have=$(kubectl get deployment front -n "$NS" -o jsonpath='{.status.replicas}')
  if [ "$have" = "4" ]; then break; fi
  sleep 2
done
if [ "$have" != "4" ]; then
  echo "ÉCHEC : front demande 6 exemplaires et en a $have, au lieu des 4 que le quota laisse passer."
  echo "  Si le quota a été posé après les pods, il ne les supprime pas : revenez à 2 exemplaires, puis redemandez-en 6."
  echo "  Pour voir ce que compte le quota : kubectl describe quota -n $NS"
  exit 1
fi
echo "RÉUSSI : le quota de 5 pods tient. front demande 6 exemplaires et n'en a que 4."
```

````solution
```bash
kubectl create quota shop-pods --hard=pods=5 -n layers-l4-shop
kubectl scale deployment front --replicas=6 -n layers-l4-shop
kubectl get events -n layers-l4-shop --field-selector reason=FailedCreate
```
````

# Tâche 3 : fermer le réseau de la boutique

## Tout fermer, puis ouvrir les deux flux utiles

Pour l'instant, n'importe quel pod du cluster peut joindre `cache`. Remettez d'abord `front` dans l'état que décrit son fichier, puis fermez le réseau du namespace en n'ouvrant que ce dont la boutique a besoin. Écrivez vos règles dans un fichier, par exemple `~/byte-layers/kubernetes/l4/challenge/shop.netpol.yml`.

**Critères :**

- le Deployment `front` est revenu à **2** exemplaires, tous prêts ;
- dans `layers-l4-shop`, une NetworkPolicy fait que, par défaut, **aucun pod n'accepte de connexion entrante** ;
- `front` accepte **tout le monde**, sur le port **80** ;
- `cache` n'accepte que les pods `app=front`, sur le port **6379** ;
- un pod de `front` peut toujours ouvrir une connexion vers `cache`, et un pod étranger au namespace ne le peut pas.

La vérification lit vos règles, puis teste le réseau pour de bon : depuis un pod de `front`, et depuis un pod de test qu'elle lance dans votre namespace courant.

```hint
Le fichier de la boutique dit 2 exemplaires : le réappliquer suffit à revenir à cet état.
```

```hint
Il vous faut trois NetworkPolicies, de la même forme que celles de l'application de vote : une qui désigne tous les pods et n'autorise rien, une pour `front` avec un port et sans `from`, une pour `cache` avec un `from` et un port.
```

```hint
Dans la règle de `cache`, `from` contient un `podSelector` qui désigne les pods `app: front`. N'oubliez pas `-n layers-l4-shop` quand vous appliquez le fichier : des règles appliquées dans un autre namespace n'auraient aucun effet sur la boutique.
```

```verify
NS=layers-l4-shop
if ! kubectl get deployment front -n "$NS" >/dev/null 2>&1; then
  echo "ÉCHEC : le Deployment front n'est pas dans $NS. Faites d'abord les tâches 1 et 2."
  exit 1
fi
want=$(kubectl get deployment front -n "$NS" -o jsonpath='{.spec.replicas}')
if [ "$want" != "2" ]; then
  echo "ÉCHEC : le Deployment front demande $want exemplaires, pas 2. Réappliquez son fichier."
  exit 1
fi
if ! kubectl rollout status deployment/front -n "$NS" --timeout=20s >/dev/null 2>&1; then
  echo "ÉCHEC : front demande bien 2 exemplaires, mais ils ne sont pas tous prêts. Attendez un peu, puis réessayez."
  exit 1
fi
names=$(kubectl get networkpolicy -n "$NS" -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)
if [ -z "$names" ]; then
  echo "ÉCHEC : aucune NetworkPolicy dans $NS."
  echo "  Si vous en avez appliqué, elles sont peut-être parties dans un autre namespace : kubectl get networkpolicies -A"
  exit 1
fi
if ! kubectl get networkpolicy -n "$NS" -o jsonpath='{range .items[*]}{.spec.podSelector}{"|"}{.spec.ingress}{"\n"}{end}' | grep -qx '{}|'; then
  echo "ÉCHEC : aucune règle ne ferme tout le namespace."
  echo "  Il faut une NetworkPolicy qui désigne tous les pods (podSelector: {}) et n'autorise aucune entrée."
  exit 1
fi
pol=""
for p in $names; do
  target=$(kubectl get networkpolicy "$p" -n "$NS" -o jsonpath='{.spec.podSelector.matchLabels.app}{.spec.podSelector.matchExpressions[*].values[*]}')
  if [ "$target" = "cache" ]; then pol="$p"; fi
done
if [ -z "$pol" ]; then
  echo "ÉCHEC : aucune règle ne protège les pods app=cache."
  exit 1
fi
who=$(kubectl get networkpolicy "$pol" -n "$NS" -o jsonpath='{.spec.ingress[*].from[*].podSelector.matchLabels.app} {.spec.ingress[*].from[*].podSelector.matchExpressions[*].values[*]}' | tr ' ' '\n' | grep . | sort -u | tr '\n' ' ')
if [ "$who" != "front " ]; then
  echo "ÉCHEC : la règle $pol doit laisser entrer les pods app=front, et eux seuls (trouvé : ${who:-personne de précis})."
  exit 1
fi
open=$(kubectl get networkpolicy "$pol" -n "$NS" -o jsonpath='{range .spec.ingress[*]}{.from}{"\n"}{end}{range .spec.ingress[*].from[*]}{.podSelector}{.namespaceSelector}{.ipBlock}{"\n"}{end}')
if printf '%s\n' "$open" | grep -qx -e '' -e '{}'; then
  echo "ÉCHEC : la règle $pol contient une autorisation sans condition : elle laisse entrer plus que front."
  echo "  from et ports doivent être sous le même tiret, et chaque élément de from doit désigner des pods par leur étiquette."
  exit 1
fi
ports=$(kubectl get networkpolicy "$pol" -n "$NS" -o jsonpath='{.spec.ingress[*].ports[*].port}')
if [ "$ports" != "6379" ]; then
  echo "ÉCHEC : la règle $pol doit ouvrir le port 6379, et lui seul (trouvé : ${ports:-tous les ports})."
  exit 1
fi
if ! kubectl exec deployment/front -n "$NS" -- python -c "import socket; socket.create_connection(('cache', 6379), 3)" >/dev/null 2>&1; then
  echo "ÉCHEC : un pod de front ne peut plus ouvrir de connexion vers cache, port 6379."
  echo "  Vos règles ferment trop : relisez celle qui protège cache."
  exit 1
fi
kubectl delete pod layers-l4-probe --ignore-not-found >/dev/null 2>&1
out=$(kubectl run layers-l4-probe --rm -i --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- sh -c "nc -z -w 3 cache.$NS 6379 && echo cache-open || echo cache-closed; nc -z -w 3 front.$NS 80 && echo front-open || echo front-closed" 2>&1)
case "$out" in
  *cache-open*)
    echo "ÉCHEC : un pod étranger au namespace joint encore cache."
    echo "  Si vos règles sont justes, c'est que le cluster ne les applique pas : kubectl get pods -n kube-system -l k8s-app=calico-node"
    exit 1 ;;
esac
case "$out" in
  *cache-closed*) : ;;
  *) echo "ÉCHEC : le pod de test n'a pas pu être lancé dans votre namespace courant."
     echo "  kubectl a répondu : $(echo "$out" | head -2 | tr '\n' ' ')"
     exit 1 ;;
esac
case "$out" in
  *front-open*) : ;;
  *) echo "ÉCHEC : un pod étranger au namespace ne joint plus front sur le port 80."
     echo "  front est une page web : sa règle doit ouvrir le port 80 à tout le monde, sans from."
     exit 1 ;;
esac
echo "RÉUSSI : le réseau de $NS est fermé, sauf le port 80 de front pour tous, et cache pour front."
```

````solution
```bash
kubectl apply -f ~/byte-layers/kubernetes/l4/challenge/shop.yml -n layers-l4-shop
cat > ~/byte-layers/kubernetes/l4/challenge/shop.netpol.yml <<'EOF'
# Règles réseau de la boutique : tout est fermé par défaut, puis on n'ouvre que les flux utiles.
# 1. Par défaut, aucun pod du namespace n'accepte de connexion entrante.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shop-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress]
--- # règle 2
# 2. front est la page web : ouverte à tous, sur le port 80.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shop-front-open
spec:
  podSelector:
    matchLabels: { app: front }
  policyTypes: [Ingress]
  ingress:
    - ports:
        - port: 80
--- # règle 3
# 3. cache n'accepte que front, sur le port 6379.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shop-cache-from-front
spec:
  podSelector:
    matchLabels: { app: cache }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: front }
      ports:
        - port: 6379
EOF
kubectl apply -f ~/byte-layers/kubernetes/l4/challenge/shop.netpol.yml -n layers-l4-shop
kubectl rollout status deployment/front -n layers-l4-shop --timeout=50s
```
````

# Tâche 4 : ne rien laisser derrière soi

## Supprimer le namespace et le dossier

Supprimez tout ce que ce challenge a créé : le namespace `layers-l4-shop`, avec tout ce qu'il contient, et le dossier `~/byte-layers/kubernetes/l4/challenge`.

**Critères :** plus de namespace `layers-l4-shop`, plus de dossier `challenge`. Le reste de `~/byte-layers/kubernetes/l4`, votre application de vote et le namespace `vote-app` ne sont pas concernés.

```hint
Supprimer un namespace supprime tout ce qu'il contient : Deployments, Services, quota, règles réseau. La commande peut mettre une trentaine de secondes à rendre la main.
```

```hint
Pour supprimer un dossier et tout ce qu'il contient : `rm -r` suivi de son chemin, en vérifiant bien ce chemin avant de valider.
```

```verify
NS=layers-l4-shop
if ! kubectl get namespace kube-system >/dev/null 2>&1; then
  echo "ÉCHEC : kubectl ne répond pas. Vérifiez que votre cluster est démarré, puis réessayez."
  exit 1
fi
if kubectl get namespace "$NS" >/dev/null 2>&1; then
  if [ -z "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null)" ]; then
    echo "ÉCHEC : le namespace $NS existe encore."
    exit 1
  fi
  if ! kubectl wait --for=delete "namespace/$NS" --timeout=45s >/dev/null 2>&1; then
    echo "ÉCHEC : le namespace $NS est en cours de suppression, mais ce n'est pas fini. Réessayez dans quelques secondes."
    exit 1
  fi
fi
if [ -e "$HOME/byte-layers/kubernetes/l4/challenge" ]; then
  echo "ÉCHEC : le dossier ~/byte-layers/kubernetes/l4/challenge existe encore."
  exit 1
fi
echo "RÉUSSI : tout est propre. Vous savez isoler une application dans un cluster."
```

````solution
```bash
kubectl delete namespace layers-l4-shop
rm -r ~/byte-layers/kubernetes/l4/challenge
```
````
