---
title: "Déployer la v2 sans coupure"
description: "Changer l'image dans le fichier, appliquer, voir les pods remplacés un par un pendant que l'application répond, puis lire l'historique des versions"
---

La v2 apporte un nouveau design. Déployer une nouvelle version, c'est **changer l'image** dans l'état souhaité, plus précisément dans le **modèle de pod** (le bloc `template` du Deployment, à partir duquel chaque pod est fabriqué). Vous allez le faire pendant que l'application répond, et regarder qui répond.

## Demander à vote quelle version répond

L'application de vote a une page technique, `/healthz`, qui donne le nom du pod qui répond et sa version. Pour l'interroger depuis l'intérieur du cluster, lancez un pod de test jetable, comme dans [Donner une adresse stable avec un Service](https://byte-layers.com/fr/skills/k8s-services) : il exécute une seule commande, `wget`, qui télécharge la page à l'adresse du Service `vote` et l'affiche, puis il est supprimé.

```exec
kubectl run l6-test --rm -i --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- wget -qO- http://vote/healthz
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl run l6-test` | lance un pod nommé `l6-test` |
> | `--rm -i` | affiche ce que le pod écrit, puis le supprime quand il a fini |
> | `--restart=Never` | ne le relance pas : il n'a qu'une commande à exécuter |
> | `--image=ghcr.io/bngams/kube-busybox:1.37` | une petite image qui contient les outils réseau de base |
> | `--` | sépare les options de `kubectl` de la commande à exécuter dans le pod |
> | `wget -qO- http://vote/healthz` | télécharge la page `/healthz` du Service `vote` et l'affiche (`-q` : sans commentaire, `-O-` : à l'écran plutôt que dans un fichier) |

Un des trois pods de `vote` a répondu. Son nom sera différent chez vous :

```text
{"hostname":"vote-fb9787cdf-vbxnt","status":"ok","version":"1.0"}
pod "l6-test" deleted from vote-app namespace
```

`"version":"1.0"` : c'est bien la version 1.0 qui tourne. `"hostname"` est le nom du pod qui a répondu : en relançant la commande, vous verrez ce nom changer d'un essai à l'autre, parce que le Service répartit les demandes entre les trois pods.

## Changer la version dans le fichier

Ouvrez `~/byte-layers/kubernetes/l6/vote.deploy.yml` dans votre éditeur et repérez la ligne de l'image, marquée `# TODO` :

```yaml
          image: ghcr.io/bngams/kube-vote:1.0    # TODO : seule la version, après les deux-points, change
```

Changez **seulement les chiffres de la fin** pour passer en version `2.0`, puis enregistrez le fichier. Le commentaire `# TODO` peut rester ou disparaître : Kubernetes ne lit pas ce qui suit un `#`.

## Appliquer, et interroger vote pendant la mise à jour

Avant de lancer la commande, faites une prévision : pendant que Kubernetes remplace les pods de la v1 par ceux de la v2, l'application va-t-elle cesser de répondre, même un instant ?

Les deux lignes qui suivent s'enchaînent sans attendre. La première applique votre fichier. La seconde relance le pod de test, qui interroge cette fois `/healthz` **vingt fois de suite**, une fois par seconde, pendant que la mise à jour se déroule :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
kubectl run l6-watch --rm -i --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- sh -c 'for i in $(seq 1 20); do wget -qO- -T 2 http://vote/healthz || echo "pas de réponse"; sleep 1; done'
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl run l6-watch … --` | le même pod de test jetable que tout à l'heure, sous un autre nom |
> | `sh -c '…'` | fait exécuter par le pod tout ce qui est entre apostrophes, comme une seule commande |
> | `for i in $(seq 1 20); do … done` | répète vingt fois ce qui est entre `do` et `done` ; `$(seq 1 20)` est remplacé par les nombres de 1 à 20 |
> | `wget -qO- -T 2 http://vote/healthz` | interroge `vote` ; `-T 2` abandonne au bout de 2 secondes sans réponse |
> | `\|\| echo "pas de réponse"` | `\|\|` veut dire « sinon » : si `wget` échoue, la ligne `pas de réponse` s'affiche à la place |
> | `sleep 1` | attend une seconde avant le tour suivant |

Voici ce que le pod de test a affiché pendant une vraie mise à jour. La première ligne est un message de `kubectl`, sans importance ici. Vos noms de pods seront différents, et le passage d'une version à l'autre peut arriver quelques lignes plus tôt ou plus tard :

```text
If you don't see a command prompt, try pressing enter.
{"hostname":"vote-fb9787cdf-24qtk","status":"ok","version":"1.0"}
{"hostname":"vote-fb9787cdf-vbxnt","status":"ok","version":"1.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-g7c4x","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-g7c4x","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-kj4t2","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-kj4t2","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-g7c4x","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-kj4t2","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-g7c4x","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-g7c4x","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-wjcrl","status":"ok","version":"2.0"}
{"hostname":"vote-566d594846-kj4t2","status":"ok","version":"2.0"}
pod "l6-watch" deleted from vote-app namespace
```

Pendant quelques secondes, les deux versions cohabitent : `vote` répond en **1.0**, puis de plus en plus souvent en **2.0**, puis seulement en **2.0**. Aucune ligne `pas de réponse` : à aucun moment l'application n'a cessé de répondre. Si vous ne lisez que des `1.0`, votre fichier n'a pas été enregistré avec la nouvelle version, ou l'image 2.0 se télécharge encore, et la section suivante attend la fin ; si vous ne lisez que des `2.0`, la mise à jour était déjà finie quand le pod de test a démarré.

## Suivre la mise à jour jusqu'au bout

Pour suivre une mise à jour jusqu'à sa fin, Kubernetes propose une commande dédiée, celle qui a servi d'attente à l'étape précédente :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Elle rend la main avec cette ligne quand tous les pods sont à la nouvelle version :

```text
deployment "vote" successfully rolled out
```

Regardez maintenant les pods de `vote`. L'option `-l app=vote` limite l'affichage aux pods qui portent l'étiquette `app: vote` :

```exec
kubectl get pods -l app=vote
```

Les trois pods sont neufs : leur nom a changé, et leur âge se compte en secondes.

```text
NAME                    READY   STATUS    RESTARTS   AGE
vote-566d594846-g7c4x   1/1     Running   0          20s
vote-566d594846-kj4t2   1/1     Running   0          23s
vote-566d594846-wjcrl   1/1     Running   0          21s
```

> 🧠 **La mise à jour progressive** (*rolling update*). Avec 3 réplicas et les réglages par défaut, Kubernetes crée **un** pod de la nouvelle version, attend qu'il soit prêt, retire **un** pod de l'ancienne, et recommence. Il y a toujours assez de pods pour servir les visiteurs : c'est ce qui permet de livrer en pleine journée, sans « fenêtre de maintenance ». Pendant quelques secondes, les deux versions cohabitent : les équipes de développement doivent en tenir compte (par exemple pour les changements de base de données).

```hint
Sur la ligne `image:` de `vote.deploy.yml`, seuls les chiffres après `kube-vote:` changent. Pensez à enregistrer le fichier avant d'appliquer.
```

```hint
La ligne attendue est `image: ghcr.io/bngams/kube-vote:2.0`, puis `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml`.
```

```verify
F="$HOME/byte-layers/kubernetes/l6/vote.deploy.yml"
IMG=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "Le Deployment vote est introuvable dans le namespace courant. Rejouez l'étape « Partir de l'application de vote en version 1.0 »."
  exit 1
fi
if ! grep -Eq '^[[:space:]]*image:[[:space:]]*"?ghcr\.io/bngams/kube-vote:2\.0' "$F" 2>/dev/null; then
  echo "Votre fichier vote.deploy.yml ne demande pas l'image ghcr.io/bngams/kube-vote:2.0. Changez la version sur la ligne image:, enregistrez, puis appliquez."
  exit 1
fi
case "$IMG" in
  *kube-vote:2.0) : ;;
  *) echo "Le cluster demande encore l'image $IMG. Appliquez votre fichier : kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml"
     exit 1 ;;
esac
if ! kubectl rollout status deployment/vote --timeout=45s >/dev/null 2>&1; then
  echo "La mise à jour vers la 2.0 n'est pas terminée. Regardez où elle en est avec kubectl get pods -l app=vote, puis réessayez."
  exit 1
fi
echo "vote tourne en version 2.0, et la mise à jour est terminée."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l6/vote.deploy.yml <<'EOF'
# vote : 3 réplicas, réglages lus dans la ConfigMap vote-config
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
          image: ghcr.io/bngams/kube-vote:2.0
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: vote-config
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

## Voir le nouveau design dans le navigateur

La v2 se reconnaît aussi à l'œil. Pour ouvrir la page de vote depuis votre machine, ouvrez un tunnel entre le port **28660** de votre machine et le Service `vote` :

```exec
kubectl port-forward service/vote 28660:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward service/vote 28660:80` | ouvre un tunnel temporaire : ce qui arrive sur le port 28660 de votre machine est envoyé au port 80 du Service `vote` |
> | `> /dev/null` | jette ce que la commande affiche, pour ne pas encombrer le terminal |
> | `2>&1` | jette aussi ses messages d'erreur, au même endroit |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main tout de suite, et le tunnel reste ouvert |

Le terminal affiche un numéro entre crochets, celui de la commande lancée en arrière-plan, et vous rend la main. Ouvrez `http://localhost:28660` dans votre navigateur. Sur un poste hébergé, passez par l'onglet **PORTS** de VS Code, qui donne l'adresse à ouvrir pour le port 28660. La page affiche le nouveau design, et le badge en haut à droite indique **v2.0**.

Un tunnel de ce genre reste attaché au pod qu'il a choisi à son ouverture : il ne survit pas à une mise à jour. Fermez-le maintenant. `pkill -f` arrête le programme dont la ligne de commande contient le texte entre guillemets :

```exec
pkill -f "port-forward service/vote 28660"
```

## Lire l'historique des versions

Chaque modification du modèle de pod crée un nouveau **ReplicaSet**, le « gardien du nombre » de pods rencontré dans [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment). L'ancien n'est pas supprimé : il est gardé, à 0 pod. Listez ceux de `vote` :

```exec
kubectl get replicasets -l app=vote
```

Vous devez en voir au moins deux : celui de la v2, avec ses 3 pods, et celui de la v1, vide.

```text
NAME              DESIRED   CURRENT   READY   AGE
vote-566d594846   3         3         3       23s
vote-fb9787cdf    0         0         0       26s
```

Kubernetes se sert de ces ReplicaSets gardés comme d'un **historique** :

```exec
kubectl rollout history deployment/vote
```

Chaque ligne est une **révision**, c'est-à-dire une version du modèle de pod :

```text
deployment.apps/vote 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Votre liste peut être plus longue si vous avez déjà modifié ce Deployment dans les cours précédents ; la dernière ligne est toujours la version en cours. La colonne `CHANGE-CAUSE` est vide : en entreprise, on l'alimente avec une note attachée au Deployment (une *annotation*) pour savoir **pourquoi** chaque version a été déployée.

> 📖 [Mettre à jour un Deployment](https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/) · [`kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)

## Questions sur la mise à jour progressive

```quiz
title: "Déployer une nouvelle version"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Pour déployer une nouvelle version de l'application, que modifie-t-on dans le fichier du Deployment ?"
    answers:
      - "Le nombre de réplicas"
      - "L'image, dans le modèle de pod"
      - "Le nom du Deployment"
    correct: 2
    explanation: "Une nouvelle version, c'est une nouvelle image. Kubernetes voit que le modèle de pod a changé et remplace les pods."
  - prompt: "Pendant la mise à jour, le pod de test a reçu des réponses en 1.0 et en 2.0. Pourquoi ?"
    answers:
      - "Le Service s'est trompé de pods"
      - "Les pods sont remplacés un par un : pendant quelques secondes, les deux versions cohabitent derrière le même Service"
      - "La v2 n'était pas encore téléchargée"
    correct: 2
    explanation: "C'est le principe de la mise à jour progressive. Les équipes de développement doivent prévoir cette cohabitation."
  - prompt: "Après la mise à jour, que devient le ReplicaSet de l'ancienne version ?"
    answers:
      - "Il est supprimé"
      - "Il est gardé, à 0 pod : c'est une révision de l'historique"
      - "Il garde un pod de secours"
    correct: 2
    explanation: "Kubernetes garde les anciens ReplicaSets vides. Ils servent d'historique, et permettent de revenir en arrière."
```
