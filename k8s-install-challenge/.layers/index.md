---
title: "Vérifier son cluster, sans guide"
description: "Dresser la fiche d'un cluster Kubernetes et prouver qu'il fait tourner une application, à partir d'un cahier des charges, sans guide."
tags:
  - challenge
  - kubernetes
  - containers
difficulty: beginner
duration: 25
introTitle: "Le défi"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

On vous confie un cluster, et on vous demande s'il est prêt pour le parcours. Ce challenge prouve que vous savez répondre à cette question. Il n'y a **aucune commande à recopier** : chaque tâche donne un objectif et des critères, puis un bouton vérifie l'état réel du cluster. Toute suite de commandes qui produit le bon résultat est acceptée.

Si vous êtes bloqué, les **indices** se dévoilent un par un, et la **solution** reste en dernier recours. Les utiliser ne coûte rien : seule la vérification compte.

## Avant de commencer

- **Ce qu'il faut savoir** : ce qu'enseigne le lab [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install).
- **Ce qu'il faut avoir** : `curl`, le port **28601** libre, et le cluster `layers-k8s` démarré, qui répond à `kubectl get nodes`.
- **Un dossier de travail** : `~/byte-layers/kubernetes/l0`, où `~` désigne votre dossier personnel, est créé pour vous, et la dernière tâche le supprime. Vous y écrirez un petit fichier à la main, avec l'éditeur de votre choix. Dans le terminal, `nano ~/byte-layers/kubernetes/l0/audit.txt` l'ouvre ; `Ctrl-O` puis `Entrée` enregistre, et `Ctrl-X` quitte.
- **Les noms comptent** : les vérifications cherchent les noms exacts. Le pod s'appelle `layers-l0-probe`, et rien d'autre n'est inspecté ni supprimé dans votre cluster.
- **L'ordre compte** : la tâche 3 reprend le pod de la tâche 2. Faites-les dans l'ordre.

# Tâche 1 : la fiche du cluster

## Noter le nœud, son état, la version et le namespace du cluster

```setup
mkdir -p ~/byte-layers/kubernetes/l0
```

Créez le fichier `~/byte-layers/kubernetes/l0/audit.txt` avec votre éditeur, et notez-y la fiche de ce cluster, avec une ligne par information, au format `nom=valeur`.

**Critères :**

- `node=` donne le nom du nœud du cluster ;
- `status=` donne l'état de ce nœud ;
- `version=` donne la version de Kubernetes qui tourne dans le cluster ;
- `system-namespace=` donne le nom du namespace où tournent les composants du cluster lui-même.

Le fichier doit donc ressembler à ceci, avec les vraies valeurs de votre cluster :

```text
node=mon-noeud
status=Ready
version=v1.30.0
system-namespace=un-namespace
```

```hint
Une seule commande du lab affiche le nom du nœud, son état et sa version. Lisez sa réponse, puis recopiez les valeurs dans le fichier.
```

```hint
`kubectl get nodes` donne les colonnes `NAME`, `STATUS` et `VERSION`. `kubectl version` donne aussi la version du cluster, à la ligne `Server Version`.
```

```hint
`kubectl get namespaces` liste les namespaces. Le lab dit lequel est celui du cluster lui-même : son nom commence par `kube-`.
```

```verify
F="$HOME/byte-layers/kubernetes/l0/audit.txt"
if ! kubectl get nodes --request-timeout=20s >/dev/null 2>&1; then
  echo "ÉCHEC : le cluster ne répond pas à kubectl get nodes. Démarrez-le, puis réessayez."
  exit 1
fi
if [ ! -f "$F" ]; then
  echo "ÉCHEC : le fichier ~/byte-layers/kubernetes/l0/audit.txt n'existe pas."
  exit 1
fi
val() { grep -iE "^[[:space:]]*$1[[:space:]]*=" "$F" | tail -n 1 | sed 's/^[^=]*=//' | tr -d ' \r'; }
fail=0
NODE=$(val node)
if [ -z "$NODE" ]; then
  echo "ÉCHEC : il manque la ligne node=."
  fail=1
elif ! kubectl get node "$NODE" --request-timeout=20s >/dev/null 2>&1; then
  echo "ÉCHEC : node=$NODE n'est pas un nœud de ce cluster."
  fail=1
else
  REAL=NotReady
  [ "$(kubectl get node "$NODE" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = "True" ] && REAL=Ready
  STATUS=$(val status)
  if [ -z "$STATUS" ]; then
    echo "ÉCHEC : il manque la ligne status=."
    fail=1
  elif [ "$STATUS" != "$REAL" ]; then
    echo "ÉCHEC : status=$STATUS ne correspond pas à l'état de ce nœud."
    fail=1
  fi
fi
VERSION=$(val version | sed 's/^v//')
V1=$(kubectl version 2>/dev/null | sed -n 's/^Server Version: v//p')
V2=$(kubectl get nodes -o jsonpath='{.items[0].status.nodeInfo.kubeletVersion}' | sed 's/^v//')
if [ -z "$VERSION" ]; then
  echo "ÉCHEC : il manque la ligne version=."
  fail=1
elif [ "$VERSION" != "$V1" ] && [ "$VERSION" != "$V2" ]; then
  echo "ÉCHEC : version=$(val version) ne correspond pas à ce cluster."
  fail=1
fi
SYS=$(val system-namespace)
if [ -z "$SYS" ]; then
  echo "ÉCHEC : il manque la ligne system-namespace=."
  fail=1
elif [ "$SYS" != "kube-system" ]; then
  echo "ÉCHEC : system-namespace=$SYS n'est pas le namespace des composants du cluster."
  fail=1
fi
[ "$fail" -eq 0 ] || exit 1
echo "RÉUSSI : la fiche correspond à ce cluster."
```

````solution
Recopier les valeurs à la main suffit. Cette commande fait la même chose sans rien retaper : `-o jsonpath=…` n'affiche que le champ demandé, `$(…)` place la réponse dans la phrase, et `>` envoie le tout dans le fichier.

```bash
{
  echo "node=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"
  echo "status=Ready"
  echo "version=$(kubectl get nodes -o jsonpath='{.items[0].status.nodeInfo.kubeletVersion}')"
  echo "system-namespace=kube-system"
} > ~/byte-layers/kubernetes/l0/audit.txt
```
````

# Tâche 2 : un pod témoin

## Lancer un pod et attendre qu'il soit prêt

Prouvez que ce cluster sait télécharger une image et faire tourner une application.

**Critères :**

- un pod s'appelle exactement `layers-l0-probe` ;
- il utilise l'image `ghcr.io/bngams/kube-vote:2.0` ;
- il est **prêt** : sa condition `Ready` est vraie.

```hint
Une seule commande crée un pod à partir d'une image. Le lab l'utilise avec un autre nom et une autre version de l'image.
```

```hint
Un pod n'est pas prêt à l'instant où il est créé : l'image doit d'abord être téléchargée. Une commande du lab attend à votre place, et `kubectl get pods` montre où il en est.
```

```hint
Si le nom ou l'image ne sont pas les bons, supprimez le pod avec `kubectl delete pod`, puis recréez-le.
```

```verify
P=layers-l0-probe
if ! kubectl get pod "$P" --request-timeout=20s >/dev/null 2>&1; then
  echo "ÉCHEC : aucun pod ne s'appelle $P dans votre namespace."
  exit 1
fi
IMG=$(kubectl get pod "$P" -o jsonpath='{.spec.containers[0].image}')
case "$IMG" in
  ghcr.io/bngams/kube-vote:2.0) : ;;
  *) echo "ÉCHEC : $P utilise l'image « $IMG », et ghcr.io/bngams/kube-vote:2.0 est attendue."
     echo "  L'image d'un pod ne se change pas après coup : supprimez-le et recréez-le."
     exit 1 ;;
esac
if ! kubectl wait --for=condition=Ready "pod/$P" --timeout=40s >/dev/null 2>&1; then
  echo "ÉCHEC : $P existe, mais il n'est pas prêt (état : $(kubectl get pod "$P" -o jsonpath='{.status.phase}'))."
  echo "  L'image se télécharge peut-être encore : regardez kubectl get pods, puis réessayez."
  exit 1
fi
echo "RÉUSSI : $P tourne avec l'image $IMG, et il est prêt."
```

````solution
```bash
kubectl run layers-l0-probe --image=ghcr.io/bngams/kube-vote:2.0
kubectl wait --for=condition=Ready pod/layers-l0-probe --timeout=120s
```
````

# Tâche 3 : la page du pod

## Enregistrer dans un fichier la page servie par le pod

Prouvez que l'application du pod répond : demandez sa page d'accueil depuis votre terminal, par le port **28601** de votre machine, et gardez-la dans le fichier `~/byte-layers/kubernetes/l0/page.html`.

**Critères :**

- le pod `layers-l0-probe` est toujours prêt ;
- le fichier `~/byte-layers/kubernetes/l0/page.html` existe ;
- il contient la page de vote **servie par ce pod** : on y lit son nom, `layers-l0-probe`, et son badge de version, `v2.0`.

```hint
Le pod tourne dans le cluster : il faut d'abord relier un port de votre machine à son port 80. Le lab le fait avec un tunnel temporaire.
```

```hint
`kubectl port-forward pod/layers-l0-probe 28601:80` ouvre le tunnel. Lancé tel quel, il occupe le terminal : ouvrez-en un second, ou lancez-le en arrière-plan comme dans le lab.
```

```hint
`curl` affiche la page dans le terminal. Avec `>` suivi d'un nom de fichier, sa réponse va dans ce fichier au lieu de s'afficher.
```

```verify
P=layers-l0-probe
F="$HOME/byte-layers/kubernetes/l0/page.html"
if ! kubectl wait --for=condition=Ready "pod/$P" --timeout=20s >/dev/null 2>&1; then
  echo "ÉCHEC : le pod $P n'existe pas ou n'est pas prêt. Faites d'abord la tâche 2."
  exit 1
fi
if [ ! -s "$F" ]; then
  echo "ÉCHEC : le fichier ~/byte-layers/kubernetes/l0/page.html n'existe pas, ou il est vide."
  echo "  S'il est vide, curl n'a rien reçu : le tunnel vers le port 28601 était-il ouvert ?"
  exit 1
fi
if ! grep -q "Servi par" "$F"; then
  echo "ÉCHEC : le fichier ne contient pas la page de vote."
  exit 1
fi
if ! grep -q "$P" "$F"; then
  echo "ÉCHEC : la page enregistrée n'a pas été servie par $P."
  echo "  Elle vient d'ailleurs : vérifiez que le tunnel du port 28601 mène bien à ce pod."
  exit 1
fi
if ! grep -q "v2.0" "$F"; then
  echo "ÉCHEC : la page enregistrée n'affiche pas le badge v2.0."
  exit 1
fi
echo "RÉUSSI : la page a bien été servie par $P, en version 2.0."
```

````solution
Le tunnel est lancé en arrière-plan, `sleep 3` lui laisse le temps de s'ouvrir, `curl -s` demande la page sans afficher sa progression, et `pkill` referme le tunnel.

```bash
kubectl port-forward pod/layers-l0-probe 28601:80 > /dev/null 2>&1 &
sleep 3
curl -s http://localhost:28601 > ~/byte-layers/kubernetes/l0/page.html
pkill -f "port-forward pod/layers-l0-probe 28601"
```
````

# Tâche 4 : ne rien laisser derrière soi

## Supprimer le pod, fermer le tunnel et supprimer le dossier

Supprimez tout ce que ce challenge a créé : le pod `layers-l0-probe`, le tunnel vers le port 28601 s'il est encore ouvert, et le dossier `~/byte-layers/kubernetes/l0`. Le cluster, lui, reste démarré : la suite du parcours s'en sert.

**Critères :** plus de pod `layers-l0-probe`, plus de tunnel, plus de dossier, et un cluster qui répond toujours.

```hint
Le lab supprime son pod de test avec une commande `kubectl`, et arrête son tunnel avec `pkill -f`. Un tunnel lancé au premier plan s'arrête avec `Ctrl-C`.
```

```hint
Pour un dossier et tout ce qu'il contient : `rm -r`, en vérifiant bien le chemin avant de valider.
```

```verify
if ! kubectl get nodes --request-timeout=20s >/dev/null 2>&1; then
  echo "ÉCHEC : le cluster ne répond plus. Il doit rester démarré : relancez-le."
  exit 1
fi
if kubectl get pod layers-l0-probe --request-timeout=20s >/dev/null 2>&1; then
  echo "ÉCHEC : le pod layers-l0-probe existe encore."
  exit 1
fi
if pgrep -f "port-forward pod/layers-l0-probe" >/dev/null 2>&1; then
  echo "ÉCHEC : un tunnel vers layers-l0-probe est encore ouvert."
  exit 1
fi
if [ -e "$HOME/byte-layers/kubernetes/l0" ]; then
  echo "ÉCHEC : le dossier ~/byte-layers/kubernetes/l0 existe encore."
  exit 1
fi
echo "RÉUSSI : tout est propre, et le cluster répond. Vous savez vérifier un cluster."
```

````solution
```bash
kubectl delete pod layers-l0-probe
pkill -f "port-forward pod/layers-l0-probe"
rm -r ~/byte-layers/kubernetes/l0
```
````
