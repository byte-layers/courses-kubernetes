---
title: "Décrire l'état souhaité : le Deployment"
description: "Écrire un Deployment en YAML, le transmettre avec kubectl apply, lire kubectl get deployments, puis ouvrir l'application avec kubectl port-forward"
---

Au lieu de dire à Kubernetes **quoi faire**, vous allez lui décrire **ce que vous voulez obtenir** : « je veux en permanence 1 exemplaire de `vote` en version 1.0 ». On appelle cela l'**état souhaité** (*desired state*). Kubernetes se charge ensuite, en continu, de faire correspondre la réalité à cette description.

## La forme d'un Deployment en YAML

Cette description s'écrit dans un fichier texte au format **YAML** : des lignes `clé: valeur`, organisées par **indentation** (des espaces en début de ligne, comme les niveaux d'un sommaire). Voici d'abord la forme générale, **à lire seulement, pas à copier** :

```yaml
apiVersion: apps/v1     # la "version du formulaire" à remplir
kind: Deployment        # le type d'objet qu'on décrit
metadata:               # l'étiquette sur la boîte : son nom
  name: vote
spec:                   # le contenu de la demande : ce qu'on veut obtenir
  replicas: 1           # combien d'exemplaires
  template:             # le modèle de pod à fabriquer
    ...
```

Le tableau ci-dessous détaille les champs du fichier complet. On y note `spec.replicas` pour désigner « le champ `replicas` rangé sous `spec` », et `containers[]` pour signaler que `containers` est une **liste** (chaque élément commence par un tiret `-`).

| Champ | Rôle |
|---|---|
| `kind: Deployment` | on décrit un Deployment, c'est-à-dire un objet qui **maintient** un nombre de pods identiques |
| `metadata.name` | le nom du Deployment ; ses pods porteront ce nom suivi de suffixes aléatoires |
| `metadata.labels` | des étiquettes collées sur le Deployment lui-même, pour le retrouver facilement |
| `spec.replicas` | le **nombre d'exemplaires** souhaités, qu'on appelle des **réplicas** |
| `spec.selector.matchLabels` | comment le Deployment **reconnaît ses pods** : tous ceux qui portent l'étiquette `app: vote` |
| `spec.template` | le **modèle** utilisé pour fabriquer chaque pod. Il porte l'étiquette `app: vote`, pour que le Deployment le reconnaisse |
| `containers[].image` | l'image à lancer dans chaque pod, la même que celle du pod `vote-solo` |
| `containers[].ports` | le port sur lequel l'application écoute dans le conteneur (80, le port web standard) |

> 🏷️ **Pourquoi `app: vote` apparaît-il plusieurs fois ?** Une étiquette (*label*) est un post-it collé sur un objet. Le `template` colle le post-it `app: vote` sur chaque pod fabriqué, et le `selector` dit au Deployment : « les pods qui portent ce post-it sont à toi ». Ces deux-là doivent donc être **identiques**.

## Écrire vote.deploy.yml en complétant les blancs

Créez le fichier de départ, avec ses quatre blancs. Cette commande l'écrit dans le dossier du cours :

```exec
cat > ~/byte-layers/kubernetes/l1/vote.deploy.yml <<'EOF'
# L'état souhaité : "je veux 1 exemplaire de l'application vote, en version 1.0"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: # TODO : le nom du Deployment : vote
  labels:
    app: vote
spec:
  replicas: # TODO : combien d'exemplaires voulez-vous ? (1 pour commencer)
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: # TODO : la même étiquette que dans le selector, juste au-dessus (seulement sa valeur)
    spec:
      containers:
        - name: vote
          image: # TODO : l'image du pod vote-solo, sans le "--image=" (seulement ghcr.io/...)
          ports:
            - containerPort: 80
EOF
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `cat` | recopie ce qu'il reçoit |
> | `> ~/byte-layers/kubernetes/l1/vote.deploy.yml` | envoie cette copie dans le fichier, au lieu de l'afficher ; le fichier est créé, ou remplacé s'il existe |
> | `<<'EOF'` | donne à `cat` tout le texte qui suit, tel quel, jusqu'à la ligne `EOF` |
> | `EOF` | la ligne qui marque la fin du texte ; elle n'est pas écrite dans le fichier |

Ouvrez `~/byte-layers/kubernetes/l1/vote.deploy.yml` dans votre éditeur (`~` désigne votre dossier personnel), puis remplacez chaque `# TODO` en vous aidant du tableau de la section précédente. Dans un fichier YAML, tout ce qui suit un `#` sur une ligne est un commentaire, que Kubernetes ignore. Ne relancez pas la commande ci-dessus une fois le fichier complété : elle le remplacerait par le fichier de départ.

**Règle de remplissage :** remplacez `# TODO` **et tout le texte qui le suit sur la ligne** par la valeur. Par exemple, `name: # TODO : le nom du Deployment : vote` devient `name: vote`.

N'oubliez pas d'**enregistrer** le fichier (Ctrl+S, ou Cmd+S sur Mac).

> ⚠️ **Piège : l'indentation.** En YAML, les espaces en début de ligne font partie du sens. `replicas` doit être **au même niveau** que `selector` et `template`, c'est-à-dire avec 2 espaces de plus que `spec`. Gardez exactement l'alignement du modèle : si vous ne modifiez que la fin des lignes `# TODO`, tout restera bien aligné.

## Transmettre le fichier avec kubectl apply

Votre fichier décrit ce que vous voulez. Il faut maintenant le **transmettre** au cluster. C'est le rôle de `kubectl apply`, qu'on utilise pour **tous** les fichiers YAML qui décrivent un objet Kubernetes ; `-f` lui donne le fichier :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l1/vote.deploy.yml
```

Le terminal confirme la création :

```text
deployment.apps/vote created
```

Si la commande affiche une erreur, pas d'inquiétude : rien n'a été créé. Repérez votre message dans le tableau, corrigez le fichier, enregistrez, puis relancez la commande.

| Message d'erreur | Cause probable |
|---|---|
| `error parsing` … `did not find expected key` | un problème d'**indentation** à la ligne indiquée |
| `mapping values are not allowed in this context` | une ligne contient deux fois `clé:`, par exemple `app: app: vote` : ne gardez que la valeur |
| `Name: ""` … `resource name may not be empty` | le `# TODO` de `name` n'a pas été remplacé |
| `` `selector` does not match template `labels` `` | le `# TODO` de `labels` n'a pas été remplacé, ou la valeur n'est pas `vote` |
| `containers[0].image: Required value` | le `# TODO` de `image` n'a pas été remplacé |
| `the path "…" does not exist` | le fichier n'est pas enregistré, ou pas à cet endroit |

Kubernetes a maintenant un objectif, et il lui faut quelques secondes pour l'atteindre. `kubectl rollout status` suit le Deployment et rend la main quand ses pods sont prêts, ou au bout de deux minutes :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

La dernière ligne doit être `deployment "vote" successfully rolled out`. Regardez alors le Deployment :

```exec
kubectl get deployments
```

Il annonce un pod prêt sur un souhaité :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   1/1     1            1           10s
```

Regardez ensuite ses pods :

```exec
kubectl get pods
```

Vous n'avez lancé aucun pod : c'est le Deployment qui l'a fabriqué, d'après son modèle.

```text
NAME                    READY   STATUS    RESTARTS   AGE
vote-644d47956b-sz9mc   1/1     Running   0          10s
```

Voici comment lire ces deux tableaux :

| Colonne | Ce qu'elle dit |
|---|---|
| `READY 1/1` (deployment) | 1 pod prêt sur 1 souhaité : **la réalité correspond à l'état souhaité** |
| `NAME vote-644d47956b-sz9mc` (pod) | le nom du Deployment (`vote`), suivi de deux suffixes générés automatiquement. **Le vôtre sera différent** |
| `STATUS Running` | le conteneur tourne |
| `RESTARTS 0` | le conteneur n'a jamais redémarré |

> ⚠️ **Le pod reste en `ErrImagePull`, `ImagePullBackOff` ou `InvalidImageName` ?** Kubernetes n'arrive pas à télécharger l'image : vérifiez la ligne `image:`, qui doit être exactement `ghcr.io/bngams/kube-vote:1.0`. Corrigez, enregistrez, puis relancez `kubectl apply`.

```hint
Les quatre valeurs sont dans le texte : le nom est `vote`, il faut `1` exemplaire, l'étiquette du modèle est la même que celle du `selector`, et l'image est celle que vous avez donnée à `kubectl run`.
```

```hint
Après `image:`, il ne doit rester que `ghcr.io/bngams/kube-vote:1.0`. Si `kubectl apply` affiche une erreur, le tableau des messages dit quelle ligne revoir.
```

```verify
if ! kubectl version --request-timeout=10s >/dev/null 2>&1; then
  echo "kubectl ne joint pas le cluster. Vérifiez qu'il est démarré avec kubectl get nodes, puis réessayez."
  exit 1
fi
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Il n'y a pas de Deployment vote dans le cluster. Remplacez les quatre # TODO du fichier, enregistrez, puis relancez kubectl apply -f ~/byte-layers/kubernetes/l1/vote.deploy.yml et lisez son message."
  exit 1
fi
if grep -q '# TODO' "$HOME/byte-layers/kubernetes/l1/vote.deploy.yml" 2>/dev/null; then
  echo "Le Deployment vote existe, mais il reste un # TODO dans ~/byte-layers/kubernetes/l1/vote.deploy.yml. Remplacez-le, enregistrez, puis relancez kubectl apply."
  exit 1
fi
REPLICAS=$(kubectl get deployment vote -o jsonpath='{.spec.replicas}')
if [ "$REPLICAS" != "1" ]; then
  echo "Le Deployment vote demande $REPLICAS exemplaires, et non 1. Corrigez replicas dans le fichier, puis relancez kubectl apply."
  exit 1
fi
IMAGE=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}')
if [ "$IMAGE" != "ghcr.io/bngams/kube-vote:1.0" ]; then
  echo "L'image du Deployment vote est « $IMAGE », et non ghcr.io/bngams/kube-vote:1.0. Corrigez la ligne image du fichier, puis relancez kubectl apply."
  exit 1
fi
LABEL=$(kubectl get deployment vote -o jsonpath='{.spec.template.metadata.labels.app}')
if [ "$LABEL" != "vote" ]; then
  echo "Le modèle de pod porte l'étiquette app: « $LABEL », et non app: vote. Corrigez-la dans le fichier, sous template, puis relancez kubectl apply."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "Le Deployment vote existe, mais son pod n'est pas prêt. Lisez la colonne STATUS de kubectl get pods : ErrImagePull ou ImagePullBackOff désigne la ligne image."
  exit 1
fi
echo "Le Deployment vote maintient 1 pod prêt, en version 1.0."
```

````solution
Le fichier complet, écrit par une commande, puis transmis au cluster :

```bash
cat > ~/byte-layers/kubernetes/l1/vote.deploy.yml <<'EOF'
# L'état souhaité : "je veux 1 exemplaire de l'application vote, en version 1.0"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: vote
spec:
  replicas: 1
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
EOF
kubectl apply -f ~/byte-layers/kubernetes/l1/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

> 📖 [Les Deployments, documentation officielle](https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/) · [`kubectl apply`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/) · [Les étiquettes (labels)](https://kubernetes.io/fr/docs/concepts/overview/working-with-objects/labels/)

## Ouvrir l'application par un tunnel : kubectl port-forward

L'application tourne **dans** le cluster, qui est isolé du monde extérieur. Pour l'ouvrir dans votre navigateur, on crée un **tunnel** temporaire entre votre machine et le pod avec `kubectl port-forward`. Un **port** est simplement un numéro de « porte » sur lequel une application attend les visiteurs : le tunnel relie une porte de votre côté à la porte de l'application.

Lancée seule, cette commande occupe le terminal tant que le tunnel est ouvert. La fin de la ligne la fait tourner en arrière-plan, pour que vous gardiez la main :

```exec
kubectl port-forward deployment/vote 28610:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward` | ouvre un tunnel entre votre machine et le cluster |
> | `deployment/vote` | vers un pod du Deployment `vote` |
> | `28610:80` | le port **28610 de votre côté** est relié au port **80 du conteneur** |
> | `> /dev/null` | jette ce que la commande afficherait (une ligne à chaque visite) ; `/dev/null` est un fichier spécial qui ne garde rien |
> | `2>&1` | y envoie aussi ses messages d'erreur |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main tout de suite |

Le terminal affiche une ligne du genre `[1] 12345`, le numéro du programme lancé en arrière-plan, et attend votre commande suivante. Plus tard, une ligne `terminated` ou `Terminé` à son sujet signalera seulement que ce programme s'est arrêté : ce n'est pas une erreur. Ouvrez maintenant `http://localhost:28610` dans votre navigateur. Sur un poste hébergé, passez par l'onglet **PORTS** de VS Code, qui donne l'adresse à ouvrir pour le port 28610.

Vous devez voir la page « Chats ou Chiens ? », avec un badge **v1.0** en haut à droite. En bas de la page, la ligne **« Servi par : vote-644d47956b-sz9mc »** affiche le **nom du pod** qui a fabriqué la page : c'est celui que montrait `kubectl get pods`.

La même page se lit depuis le terminal. `curl` demande la page, et `grep` n'en garde que la ligne qui contient « Servi par » :

```exec
curl -s http://localhost:28610 | grep "Servi par"
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `curl -s http://localhost:28610` | demande la page et affiche son code HTML ; `-s` retire la barre de progression |
> | `\|` | passe ce que `curl` affiche à la commande suivante, au lieu de l'afficher |
> | `grep "Servi par"` | ne garde que les lignes qui contiennent ce texte |

La ligne est entourée de balises HTML, et le nom du pod est le vôtre :

```text
          Servi par : <strong>vote-644d47956b-sz9mc</strong>
```

Si la commande n'affiche rien, le tunnel n'est pas ouvert : relancez sa commande sans la fin `> /dev/null 2>&1 &` pour lire son message d'erreur, par exemple un port 28610 déjà pris, puis arrêtez-la avec Ctrl+C.

Arrêtez le tunnel avant de passer à la suite. `pkill -f` arrête le programme dont la ligne de commande contient le texte entre guillemets :

```exec
pkill -f "port-forward deployment/vote 28610"
```

```ack
J'ai vu la page de vote, et le nom de mon pod en bas
```
