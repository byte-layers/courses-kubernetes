---
title: "Démarrer son cluster Kubernetes avec minikube"
description: "Démarrer un cluster Kubernetes rien qu'à soi, avec minikube sur son poste ou k3d sur un poste hébergé, vérifier qu'il répond, y lancer un premier pod et l'ouvrir dans le navigateur."
tags:
  - lab
  - kubernetes
  - containers
difficulty: beginner
duration: 40
introTitle: "Présentation"
requires: "0.4.0"
options:
  allowManualProgress: false
  autoCompleteOnNext: unverifiable
---

Pour apprendre Kubernetes, on **émule un cluster sur son poste** avec une distribution légère ([Choisir une distribution Kubernetes](https://byte-layers.com/fr/bits/k8s-distribution-choice) présente les autres choix). C'est ce que fait ce lab. Vous allez démarrer **votre propre cluster Kubernetes**, vérifier qu'il répond, y lancer une première application et l'ouvrir dans votre navigateur. À la fin, vous avez un cluster prêt pour les labs Kubernetes, et vous savez l'arrêter le soir et le relancer le lendemain.

**Niveau** : débutant. Il suffit de savoir ouvrir un terminal et d'y lancer une commande. Chaque commande est expliquée quand elle arrive.

**Prérequis** : savoir ce qu'est un conteneur et avoir Docker en état de marche, ce à quoi vous amène le lab [Installer Docker et vérifier son poste](https://byte-layers.com/fr/labs/docker-install).

**Ce qu'il faut** : sur votre poste, Docker qui répond à `docker ps`, le droit d'installer deux outils, au moins 2 processeurs, 2 Go de mémoire libres et 20 Go de disque pour le cluster, une connexion à Internet, `curl`, une commande qui demande une page web depuis le terminal, et le port **28600** libre. Sur un poste hébergé, c'est-à-dire un VS Code ouvert dans votre navigateur et fourni par un formateur, tout est déjà installé.

**Ce que le lab crée** : un cluster nommé `layers-k8s`, qui **reste en place** à la fin, car tous les cours suivants s'en servent. Le pod de test est supprimé avant la fin. Vous n'écrivez aucun fichier ; l'outil qui démarre le cluster note seulement son adresse dans `~/.kube/config`, le fichier où `kubectl`, l'outil en ligne de commande de Kubernetes, lit à quel cluster parler.

# Choisir où tournera le cluster

## Deux modes : sur votre poste, ou sur un poste hébergé

Un **cluster** est un ensemble de serveurs pilotés ensemble par Kubernetes. Pour apprendre, un seul serveur suffit, et il peut tourner à deux endroits. Les deux modes donnent exactement le même résultat : un cluster Kubernetes **rien qu'à vous**. Ils diffèrent seulement par l'endroit où ce cluster tourne.

| | Sur votre poste | Sur un poste hébergé |
|---|---|---|
| Où tourne le cluster ? | Sur votre poste, dans Docker, grâce à **minikube** | Sur un serveur préparé par un formateur, grâce à **k3d** |
| Ce qu'il faut sur le poste | Docker, minikube, kubectl | **Rien**, un navigateur suffit |
| Éditeur et terminal | VS Code installé sur le poste, terminal intégré | VS Code **dans le navigateur**, terminal intégré |
| Quand le choisir ? | Docker est déjà installé et fonctionne | On vous a fourni l'adresse d'un poste hébergé |

> 🧠 **minikube, k3d… ce n'est pas « un autre Kubernetes » ?** Non. Ce sont deux **distributions légères** qui installent **un vrai Kubernetes en miniature** (un seul serveur) pour apprendre ou tester. Les commandes `kubectl` que vous taperez ensuite sont les mêmes que sur un cluster de production chez AWS, Google ou Azure.

C'est dans ce lab que les gestes diffèrent le plus selon le mode. Une fois le cluster démarré, les commandes `kubectl` sont les mêmes ; ce qui change encore, comme la façon d'ouvrir une application dans le navigateur, est dit à chaque fois.

```ack
J'ai choisi mon mode : mon poste, ou un poste hébergé
```

# Démarrer le cluster

## Sur votre poste : vérifier que Docker répond

Sur un poste hébergé, passez directement à la section « Sur un poste hébergé : vérifier son cluster k3d, ou le créer ».

minikube fait tourner Kubernetes **à l'intérieur de Docker**. Docker doit donc être démarré avant toute chose. Ouvrez un terminal, par exemple celui de VS Code, menu **Terminal > Nouveau terminal**, et demandez sa version à Docker :

```exec
docker version
```

Vous devez voir deux blocs, `Client` et `Server`. Si le bloc `Server` affiche une erreur, le moteur Docker n'est pas démarré : avec Docker Desktop, lancez l'application, attendez qu'elle indique que le moteur tourne, puis recommencez.

## Sur votre poste : installer minikube et kubectl

Deux outils sont nécessaires :

| Outil | Rôle |
|---|---|
| `minikube` | **crée et démarre** le cluster sur votre poste |
| `kubectl` | **parle** au cluster : c'est la télécommande de Kubernetes, celle que vous utiliserez dans tous les cours |

L'installation dépend de votre système et ne se fait qu'une fois : suivez les pages officielles, [Installer minikube](https://minikube.sigs.k8s.io/docs/start/) et [Installer kubectl](https://kubernetes.io/fr/docs/tasks/tools/), en y choisissant votre système. Sous macOS, avec le gestionnaire de paquets Homebrew, elle tient en une commande :

```bash
brew install minikube kubectl
```

Sous Windows, les commandes de ce lab, et des labs Kubernetes, sont écrites pour un terminal de type Linux, ce que PowerShell n'est pas : travaillez dans le terminal de WSL, le Linux intégré à Windows et suivez-y les instructions **Linux** des deux pages officielles.

> 💡 **Remarque.** Dans PowerShell, `winget install Kubernetes.minikube` puis `winget install Kubernetes.kubectl` installent aussi les deux outils, mais pour PowerShell seulement : le terminal de WSL ne les voit pas, et plusieurs commandes des labs Kubernetes ne s'écrivent pas de la même façon dans PowerShell.

Fermez puis rouvrez votre terminal pour que les nouvelles commandes soient reconnues. Si VS Code était ouvert, **fermez-le complètement et relancez-le** : son terminal intégré ne verrait pas encore `minikube` ni `kubectl`.

Demandez sa version à chacun, pour voir qu'ils sont bien là. Commencez par minikube :

```exec
minikube version
```

Il répond par son numéro de version. Ce cours a été écrit avec celle-ci :

```text
minikube version: v1.37.0
commit: 65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3
```

Faites de même pour `kubectl`. L'option `--client` lui demande sa propre version, sans chercher à joindre un cluster, puisqu'il n'y en a pas encore :

```exec
kubectl version --client
```

La réponse tient en deux lignes :

```text
Client Version: v1.34.1
Kustomize Version: v5.7.1
```

Si le terminal répond `command not found`, l'outil n'est pas installé, ou le terminal n'a pas été rouvert depuis l'installation.

## Sur votre poste : démarrer le cluster avec minikube

Tout est prêt pour créer le cluster. Cette commande le crée la première fois ; les fois suivantes, elle relance le cluster que vous aviez arrêté. Tant qu'il tourne, il n'y a pas à la relancer.

```exec
minikube start -p layers-k8s --driver=docker --cni=calico
```

| Option | Rôle |
|---|---|
| `-p layers-k8s` | le nom du cluster, que minikube appelle un **profil**. Si vous avez déjà un autre cluster minikube, il n'est pas touché. Toutes les commandes `minikube` des labs Kubernetes portent cette option |
| `--driver=docker` | faire tourner le cluster dans Docker |
| `--cni=calico` | utiliser le réseau Calico, qui sait appliquer les règles réseau qu'un prochain cours vous fera écrire |

Le premier démarrage télécharge Kubernetes et prend quelques minutes. Voici ce que le terminal affiche ; selon le vôtre, des emoji remplacent les `*`.

```text
* [layers-k8s] minikube v1.37.0 on Darwin 26.6.2 (arm64)
* Using the docker driver based on user configuration
* Using Docker Desktop driver with root privileges
* Starting "layers-k8s" primary control-plane node in "layers-k8s" cluster
* Pulling base image v0.0.48 ...
* Configuring Calico (Container Networking Interface) ...
* Verifying Kubernetes components...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Enabled addons: storage-provisioner, default-storageclass
* Done! kubectl is now configured to use "layers-k8s" cluster and "default" namespace by default
```

La dernière ligne est celle qui compte : `kubectl` sait maintenant à quel cluster parler. Passez à la dernière section de cette étape, « Dans les deux modes : faire vérifier que kubectl parle au cluster ».

> ⚠️ **Symptôme :** la commande s'arrête sur une ligne `Exiting due to …` qui nomme le driver `docker`.
> **Cause :** le plus souvent, minikube n'arrive pas à joindre le moteur Docker.
> **Correctif :** vérifiez que `docker version` affiche bien son bloc `Server`, puis relancez la commande.

## Sur un poste hébergé : vérifier son cluster k3d, ou le créer

Cette section ne concerne que le poste hébergé. Votre formateur vous a donné l'adresse de votre poste et son mot de passe : ouvrez-la, et VS Code s'affiche dans le navigateur. Ouvrez-y un terminal, menu ☰ > **Terminal** > **New Terminal**. Les outils sont déjà installés. Ces deux commandes le montrent, en demandant sa version à chacun ; l'option `--client` demande à `kubectl` sa propre version, sans chercher à joindre un cluster :

```exec
kubectl version --client
k3d version
```

Votre cluster, nommé `layers-k8s`, a normalement **déjà été créé** pour vous, dans le Docker de votre poste hébergé, qui n'est partagé avec personne. Demandez la liste des clusters :

```exec
k3d cluster list
```

Si le cluster existe, le terminal affiche une ligne `layers-k8s` :

```text
NAME         SERVERS   AGENTS   LOADBALANCER
layers-k8s   1/1       0/0      true
```

Passez alors directement à la section suivante. Si la liste est vide, créez le cluster vous-même :

```exec
k3d cluster create layers-k8s --port "8000:80@loadbalancer"
```

| Élément | Rôle |
|---|---|
| `cluster create layers-k8s` | crée un cluster nommé `layers-k8s` |
| `--port "8000:80@loadbalancer"` | ouvre la porte d'entrée du cluster sur le port 8000. Elle sert à un prochain cours, sur l'Ingress, mais elle doit être prévue dès la création |

La commande prend environ 30 secondes. Elle se termine par les lignes suivantes :

```text
INFO[0032] Cluster 'layers-k8s' created successfully!
INFO[0032] You can now use it like this:
kubectl cluster-info
```

## Dans les deux modes : faire vérifier que kubectl parle au cluster

Quel que soit votre mode, le cluster `layers-k8s` est maintenant démarré. La vérification contrôle que `kubectl` est installé, qu'il parle bien à ce cluster, et que le cluster lui répond.

```verify
if ! command -v kubectl >/dev/null 2>&1; then
  echo "La commande kubectl est introuvable. Sur votre poste, installez-la, puis rouvrez votre terminal."
  exit 1
fi
CTX=$(kubectl config current-context 2>/dev/null)
case "$CTX" in
  layers-k8s|k3d-layers-k8s) : ;;
  *) echo "kubectl ne parle pas au cluster layers-k8s (cluster courant : ${CTX:-aucun})."
     echo "Sur votre poste : minikube start -p layers-k8s --driver=docker --cni=calico. Sur un poste hébergé : k3d cluster create layers-k8s --port \"8000:80@loadbalancer\"."
     exit 1 ;;
esac
if ! kubectl get nodes --request-timeout=20s >/dev/null 2>&1; then
  echo "Le cluster layers-k8s est connu de kubectl, mais il ne répond pas. Il est peut-être arrêté : relancez la commande de démarrage de votre mode."
  exit 1
fi
echo "kubectl parle au cluster $CTX, et le cluster répond."
```

# Vérifier que le cluster répond

## Lister les nœuds du cluster

À partir d'ici, **tout est identique** dans les deux modes. Demandez à Kubernetes la liste de ses serveurs (on dit des **nœuds**, *nodes* en anglais) :

```exec
kubectl get nodes
```

Vous devez voir **un nœud** à l'état `Ready`. Voici la réponse sur un poste, avec minikube :

```text
NAME         STATUS   ROLES           AGE     VERSION
layers-k8s   Ready    control-plane   5m41s   v1.34.0
```

Sur un poste hébergé, le nœud s'appelle `k3d-layers-k8s-server-0`, et sa version se termine par `+k3s1`. Si `STATUS` indique `NotReady`, le cluster finit de démarrer : attendez quelques secondes et relancez la commande.

> 🧠 **Ce qui vient de se passer.** `kubectl` a envoyé une question au cluster (« quels sont tes nœuds ? ») et le cluster a répondu. Votre télécommande est branchée : vous êtes prêt.

```verify
if ! kubectl get nodes --request-timeout=20s >/dev/null 2>&1; then
  echo "Le cluster ne répond pas à kubectl get nodes. Est-il démarré ? Reprenez l'étape « Démarrer le cluster »."
  exit 1
fi
READY=$(kubectl get nodes --request-timeout=20s -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' | grep -c '^True$')
if [ "$READY" -lt 1 ]; then
  echo "Le cluster répond, mais aucun nœud n'est Ready. Attendez quelques secondes, puis réessayez : kubectl get nodes"
  exit 1
fi
echo "Le cluster répond : $READY nœud Ready."
```

## Lire la version du cluster et son adresse

Tout à l'heure, `kubectl version --client` ne donnait que la version de la télécommande. Sans cette option, `kubectl` demande aussi la sienne au cluster :

```exec
kubectl version
```

Une ligne `Server Version` s'est ajoutée : c'est la version de Kubernetes qui tourne dans votre cluster. Les deux numéros n'ont pas à être égaux : un écart d'une version mineure, le deuxième nombre, est prévu.

```text
Client Version: v1.34.1
Kustomize Version: v5.7.1
Server Version: v1.34.0
```

Pour savoir où `kubectl` envoie ses questions, demandez l'adresse du cluster :

```exec
kubectl cluster-info
```

La première ligne donne l'adresse du **control plane**, la partie du cluster qui reçoit les demandes et prend les décisions. Sur votre poste, `127.0.0.1` désigne votre propre machine : le cluster tourne bien chez vous. Le numéro de port, lui, change d'un poste à l'autre. La deuxième ligne nomme CoreDNS, l'annuaire interne du cluster : il ne sert pas dans ce lab.

```text
Kubernetes control plane is running at https://127.0.0.1:52006
CoreDNS is running at https://127.0.0.1:52006/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Lister les namespaces

Un cluster range ce qu'il contient dans des **namespaces**, des espaces de rangement qui séparent les projets ou les équipes. Demandez leur liste :

```exec
kubectl get namespaces
```

Un cluster neuf en a quatre :

```text
NAME              STATUS   AGE
default           Active   5m41s
kube-node-lease   Active   5m41s
kube-public       Active   5m41s
kube-system       Active   5m41s
```

Deux comptent pour l'instant. `default` est le vôtre : tout ce que vous créez sans rien préciser y va. `kube-system` est celui du cluster lui-même, où tournent ses propres composants. Vous n'y créez rien.

# Lancer un pod de test et ouvrir l'application

## Lancer l'application de vote dans un pod

Un cluster qui répond, c'est bien ; un cluster qui fait tourner une application, c'est la preuve qu'il fonctionne. Kubernetes ne lance pas un conteneur tout seul : il l'enveloppe dans un **pod**, la plus petite unité qu'il sait faire tourner. Lancez l'application de vote dans un pod :

```exec
kubectl run layers-l0-vote --image=ghcr.io/bngams/kube-vote:1.0
```

| Élément | Rôle |
|---|---|
| `kubectl run` | crée un pod et le démarre |
| `layers-l0-vote` | le nom du pod |
| `--image=ghcr.io/bngams/kube-vote:1.0` | l'image du conteneur que le pod enveloppe : l'application de vote, version 1.0 |

`kubectl` répond aussitôt `pod/layers-l0-vote created`, mais le pod n'est pas encore prêt : le cluster doit d'abord télécharger l'image. Plutôt que de deviner combien de temps attendre, demandez à `kubectl` d'attendre pour vous :

```exec
kubectl wait --for=condition=Ready pod/layers-l0-vote --timeout=120s
```

| Élément | Rôle |
|---|---|
| `kubectl wait` | ne rend la main que lorsqu'une condition est vraie |
| `--for=condition=Ready` | la condition attendue : Kubernetes tient pour chaque pod une liste de conditions, et `Ready` veut dire « prêt à recevoir des visiteurs » |
| `pod/layers-l0-vote` | l'objet surveillé : son type, puis son nom |
| `--timeout=120s` | renonce après deux minutes, au lieu d'attendre sans fin |

Quand la commande répond `pod/layers-l0-vote condition met`, listez les pods :

```exec
kubectl get pods
```

Votre pod tourne. `READY 1/1` veut dire que son seul conteneur est prêt :

```text
NAME             READY   STATUS    RESTARTS   AGE
layers-l0-vote   1/1     Running   0          4s
```

> ⚖️ **Un raccourci, pour ce test seulement.** `kubectl run` donne un ordre, comme `docker run` : si ce pod disparaît, personne ne le recrée. Le cours suivant montre la façon normale de déployer une application, qui consiste à décrire ce qu'on veut dans un fichier.

## Ouvrir l'application avec un port-forward

Le pod tourne **dans** le cluster, et rien ne le relie encore à votre machine. Pour un essai, `kubectl port-forward` ouvre un tunnel temporaire entre un port de votre machine et un port du pod. Cette commande relie le port **28600** de votre machine au port 80 du pod, et tourne en arrière-plan pour vous rendre la main :

```exec
kubectl port-forward pod/layers-l0-vote 28600:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward pod/layers-l0-vote` | ouvre un tunnel vers le pod `layers-l0-vote` |
> | `28600:80` | le port de votre machine, puis le port du pod : l'application y attend sur le port 80 |
> | `> /dev/null` | le tunnel écrit une ligne à chaque visite ; `>` envoie ce qu'il écrit vers `/dev/null`, la « poubelle » du système, pour ne pas encombrer le terminal |
> | `2>&1` | envoie aussi ses messages d'erreur au même endroit |
> | `&` | lance la commande en arrière-plan : le tunnel reste ouvert, et le terminal vous rend la main |

Le terminal affiche un numéro, celui du programme lancé en arrière-plan. Demandez la page par le tunnel, avec `curl`, qui affiche dans le terminal la page qu'on lui demande :

```exec
curl http://localhost:28600
```

`curl` affiche le code de la page de vote. Vers la fin, vous y lisez le nom de votre pod :

```text
          Servi par : <strong>layers-l0-vote</strong>
```

Si `curl` répond par une erreur, le tunnel n'a pas fini de s'ouvrir : attendez une seconde et relancez. Pour voir la page, ouvrez `http://localhost:28600` dans votre navigateur. Sur un poste hébergé, utilisez plutôt l'onglet **PORTS** de VS Code, dans le panneau du bas, qui donne le lien de ce port. Vous voyez « Chats ou Chiens ? », avec un badge **v1.0** : **ne cliquez pas sur les boutons**, l'application n'est pas complète et le vote déclencherait une erreur.

Le même principe vaut pour les labs Kubernetes : chacun annonce son port, entre 28600 et 28679, et vous ouvrez `http://localhost:` suivi de ce port, ou l'onglet **PORTS**.

Un tunnel ne se ferme pas tout seul. Arrêtez-le : `pkill -f` arrête le programme dont la ligne de commande contient le texte entre guillemets.

```exec
pkill -f "port-forward pod/layers-l0-vote 28600"
```

```verify
if ! kubectl get pod layers-l0-vote --request-timeout=20s >/dev/null 2>&1; then
  echo "Le pod layers-l0-vote n'existe pas. Lancez la commande kubectl run de cette étape."
  exit 1
fi
case "$(kubectl get pod layers-l0-vote -o jsonpath='{.spec.containers[0].image}')" in
  *kube-vote:*) : ;;
  *) echo "Le pod layers-l0-vote n'utilise pas l'image ghcr.io/bngams/kube-vote:1.0. Supprimez-le avec kubectl delete pod layers-l0-vote, puis relancez la commande kubectl run."; exit 1 ;;
esac
if ! kubectl wait --for=condition=Ready pod/layers-l0-vote --timeout=40s >/dev/null 2>&1; then
  echo "Le pod layers-l0-vote existe, mais il n'est pas prêt (état : $(kubectl get pod layers-l0-vote -o jsonpath='{.status.phase}'))."
  echo "L'image se télécharge peut-être encore. Regardez : kubectl get pods"
  exit 1
fi
echo "Le pod layers-l0-vote tourne dans votre cluster, et il est prêt."
```

## Supprimer le pod de test

Le cluster a fait sa preuve, et ce pod ne sert plus. Supprimez-le :

```exec
kubectl delete pod layers-l0-vote
```

`kubectl` attend que le pod soit vraiment arrêté, ce qui peut prendre quelques secondes, puis il confirme :

```text
pod "layers-l0-vote" deleted from default namespace
```

```verify
if ! kubectl get nodes --request-timeout=20s >/dev/null 2>&1; then
  echo "Le cluster ne répond pas : il doit rester démarré. Relancez-le avec la commande de démarrage de votre mode, à l'étape « Démarrer le cluster »."
  exit 1
fi
if kubectl get pod layers-l0-vote --request-timeout=20s >/dev/null 2>&1; then
  echo "Le pod layers-l0-vote existe encore. Supprimez-le : kubectl delete pod layers-l0-vote"
  exit 1
fi
if pgrep -f "port-forward pod/layers-l0-vote 28600" >/dev/null 2>&1; then
  echo "Le tunnel vers le port 28600 est encore ouvert. Arrêtez-le avec la commande pkill de la section précédente."
  exit 1
fi
echo "Le pod de test est supprimé, le tunnel est fermé, et le cluster est prêt pour la suite."
```

# Arrêter et relancer son cluster

## Arrêter le soir, relancer le lendemain

Le cluster consomme de la mémoire tant qu'il tourne. Arrêtez-le quand vous avez fini de travailler, et relancez-le en reprenant : **votre travail est conservé**. Ces commandes ne sont pas à lancer maintenant, elles sont là pour le jour où vous en aurez besoin.

| | Sur votre poste | Sur un poste hébergé |
|---|---|---|
| Arrêter | `minikube stop -p layers-k8s` | `k3d cluster stop layers-k8s` |
| Relancer | `minikube start -p layers-k8s` | `k3d cluster start layers-k8s` |
| Tout supprimer, quand vous n'en avez plus besoin | `minikube delete -p layers-k8s` | `k3d cluster delete layers-k8s` |

Pour relancer, `minikube start -p layers-k8s` suffit : minikube se souvient des options données à la création. Après une relance, la colonne `RESTARTS` de vos pods n'est plus à zéro : c'est normal, leurs conteneurs ont redémarré avec le cluster.

Sur votre poste, une commande dit à tout moment où en est le cluster :

```exec
minikube status -p layers-k8s
```

Quand tout tourne, elle répond ceci. Après un arrêt, ces lignes n'indiquent plus `Running`.

```text
layers-k8s
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

```ack
Je sais arrêter, relancer et supprimer mon cluster
```

## Récap

- Un cluster d'apprentissage est un vrai Kubernetes en miniature : minikube le fait tourner dans Docker sur votre poste, k3d sur un poste hébergé.
- `kubectl` est la télécommande : `kubectl get nodes` dit si le cluster répond, `kubectl version` et `kubectl cluster-info` donnent sa version et son adresse.
- Un cluster range tout dans des namespaces : `default` est le vôtre, `kube-system` celui du cluster.
- `kubectl run` lance un pod de test, `kubectl wait` attend qu'il soit prêt, `kubectl port-forward` ouvre un tunnel temporaire vers lui, et `kubectl delete pod` le supprime.
- Le cluster s'arrête et se relance sans rien perdre.

> 📖 [Démarrer avec minikube](https://minikube.sigs.k8s.io/docs/start/) · [`minikube start`](https://minikube.sigs.k8s.io/docs/commands/start/) · [Installer kubectl](https://kubernetes.io/fr/docs/tasks/tools/) · [k3d](https://k3d.io/stable/) · [`kubectl port-forward`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/) · [Les namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

Pour vous tester sans guide, faites le challenge [Vérifier son cluster, sans guide](https://byte-layers.com/fr/challenges/k8s-install-challenge).
