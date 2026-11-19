---
title: "Ranger l'application dans son namespace"
description: "Créer le namespace vote-app, en faire le namespace courant de kubectl, puis y redéployer l'application avec les mêmes fichiers"
---

`default` ne contient plus l'application. Vous allez créer le namespace de l'application, demander à `kubectl` d'y travailler, puis redéployer l'application avec les mêmes fichiers, sans en changer une ligne.

## Créer le namespace vote-app

Un namespace est un objet comme un autre : on le crée, on le liste, on le supprime. Créez celui de l'application de vote. `kubectl create namespace` le crée sans fichier : pour un objet qui tient en un nom, ce raccourci est courant.

```exec
kubectl create namespace vote-app
```

Kubernetes confirme la création :

```text
namespace/vote-app created
```

```verify
if [ "$(kubectl get namespace vote-app -o jsonpath='{.status.phase}' 2>/dev/null)" != "Active" ]; then
  echo "Le namespace vote-app n'existe pas, ou il est en cours de suppression. Lancez kubectl create namespace vote-app, puis vérifiez avec kubectl get namespaces."
  exit 1
fi
echo "Le namespace vote-app existe."
```

## Faire de vote-app le namespace courant

Pour travailler dans `vote-app`, vous pourriez ajouter `-n vote-app` à chaque commande. On préfère demander à `kubectl` de l'utiliser **par défaut** : toutes vos commandes viseront alors `vote-app`, sans rien ajouter.

```exec
kubectl config set-context --current --namespace=vote-app
```

`kubectl` confirme que son réglage a changé. Sur un poste hébergé, le nom entre guillemets est `k3d-layers-k8s`.

```text
Context "layers-k8s" modified.
```

Le tableau détaille la commande :

| Morceau | Rôle |
|---|---|
| `kubectl config` | règle `kubectl` lui-même, pas le cluster : rien n'est créé ni modifié dans Kubernetes |
| `set-context --current` | modifie le **contexte** en cours, c'est-à-dire le réglage qui dit à quel cluster `kubectl` parle, et dans quel namespace |
| `--namespace=vote-app` | le namespace que `kubectl` utilisera quand vous n'en précisez pas |

Ce réglage est écrit dans le fichier de configuration de `kubectl` : il vaut pour tous vos terminaux. Un redémarrage du cluster peut le remettre à `default` : si un jour `kubectl get pods` vous répond pour `default`, relancez cette commande. La même commande avec `--namespace=default` vous ramènerait dans `default`. Relancez la commande que vous tapez depuis le début :

```exec
kubectl get pods
```

La réponse ne parle plus de `default` :

```text
No resources found in vote-app namespace.
```

```verify
ns=$(kubectl config view --minify -o jsonpath='{..namespace}' 2>/dev/null)
if [ "$ns" != "vote-app" ]; then
  echo "Le namespace courant de kubectl est « ${ns:-default} », pas vote-app."
  echo "Lancez : kubectl config set-context --current --namespace=vote-app"
  exit 1
fi
echo "kubectl travaille maintenant dans vote-app."
```

## Redéployer l'application dans vote-app

Vos fichiers ne citent aucun namespace. Vous allez relancer exactement la commande qui avait déployé l'application dans `default`. Avant de valider, faites votre prévision : où les objets vont-ils être créés ?

```exec
kubectl apply -f ~/byte-layers/kubernetes/l4/app/
```

Attendez que les cinq Deployments soient prêts. `kubectl wait` rend la main quand tous les Deployments du namespace (`--all`) ont leurs pods prêts (`Available`) :

```exec
kubectl wait --for=condition=Available deployment --all --timeout=120s
```

Puis listez les pods :

```exec
kubectl get pods
```

Les 7 pods de l'application sont de retour, avec de nouveaux noms :

```text
NAME                      READY   STATUS    RESTARTS   AGE
db-58c6c6b486-vv47f       1/1     Running   0          3s
redis-5b95d4d69f-njttx    1/1     Running   0          3s
result-656b779d76-pj89h   1/1     Running   0          3s
vote-fc57495c-2xn4z       1/1     Running   0          3s
vote-fc57495c-cvrrx       1/1     Running   0          3s
vote-fc57495c-krn2j       1/1     Running   0          3s
worker-7b7bc56f88-xqjf7   1/1     Running   0          3s
```

Grâce au réglage de la section précédente, tout est parti dans `vote-app`. Pour le voir d'un seul regard, l'option `-A` (pour *all namespaces*) liste les objets de tout le cluster, avec une colonne `NAMESPACE` :

```exec
kubectl get deployments -A
```

Vos cinq Deployments sont dans `vote-app`, à côté de ceux du cluster lui-même, et plus aucun n'est dans `default` :

```text
NAMESPACE       NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
ingress-nginx   ingress-nginx-controller   1/1     1            1           2d
kube-system     calico-kube-controllers    1/1     1            1           2d
kube-system     coredns                    1/1     1            1           2d
vote-app        db                         1/1     1            1           3s
vote-app        redis                      1/1     1            1           3s
vote-app        result                     1/1     1            1           3s
vote-app        vote                       3/3     3            3           3s
vote-app        worker                     1/1     1            1           3s
```

> 🧠 **Ce qui vient de se passer.** L'application est identique, mais elle vit maintenant dans son propre espace. Un `kubectl get pods` sans option ne montre plus que ses pods, et un autre projet pourrait avoir ses propres `vote` ou `redis` dans un autre namespace, sans conflit. L'Ingress `vote` a suivi : une porte d'entrée envoie ses visites vers un Service de son propre namespace.

```verify
if [ "$(kubectl get namespace vote-app -o jsonpath='{.status.phase}' 2>/dev/null)" != "Active" ]; then
  echo "Le namespace vote-app n'existe pas. Reprenez cette étape depuis sa première section."
  exit 1
fi
for d in redis db worker result vote; do
  if ! kubectl get deployment "$d" -n vote-app >/dev/null 2>&1; then
    echo "Le Deployment $d n'est pas dans vote-app."
    echo "Vérifiez le namespace courant (kubectl get pods doit répondre pour vote-app), puis relancez kubectl apply -f ~/byte-layers/kubernetes/l4/app/"
    exit 1
  fi
done
if ! kubectl wait --for=condition=Available deployment redis db worker result vote -n vote-app --timeout=45s >/dev/null 2>&1; then
  echo "Les cinq Deployments sont dans vote-app, mais tous leurs pods ne sont pas prêts. Regardez kubectl get pods, attendez un peu, puis réessayez."
  exit 1
fi
left=$(kubectl get deployments -n default -o name 2>/dev/null | grep -E '/(vote|redis|db|worker|result)$' | tr '\n' ' ')
if [ -n "$left" ]; then
  echo "L'application est dans vote-app, mais il en reste une copie dans default : $left"
  echo "Supprimez-la : kubectl delete -f ~/byte-layers/kubernetes/l4/app/ -n default"
  exit 1
fi
echo "Les cinq Deployments de l'application tournent dans vote-app, et plus aucun dans default."
```

## Revoir la page de vote depuis vote-app

Il reste à voir la page, pour s'assurer que rien n'a changé pour un visiteur. `kubectl port-forward` ouvre un tunnel temporaire entre un port de votre machine et l'application. Lancé tel quel, il occuperait votre terminal ; cette forme le laisse tourner en arrière-plan :

```exec
kubectl port-forward deployment/vote 28640:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward deployment/vote` | ouvre un tunnel vers un pod du Deployment `vote`, dans le namespace courant |
> | `28640:80` | le port 28640 de votre machine mène au port 80 du pod |
> | `> /dev/null` | envoie les messages du tunnel dans `/dev/null`, un fichier spécial qui jette ce qu'on y écrit, pour qu'ils n'encombrent pas le terminal |
> | `2>&1` | y envoie aussi ses messages d'erreur |
> | `&` (à la fin) | lance la commande en arrière-plan : le terminal vous rend la main tout de suite |

Ouvrez `http://localhost:28640` dans votre navigateur ; sur un poste hébergé, passez par l'onglet **PORTS** de VS Code. Si la page ne s'ouvre pas, le port est peut-être déjà pris : relancez la commande sans `> /dev/null 2>&1 &` pour lire l'erreur. Vous retrouvez la page de vote, avec la mention « Servi par », suivie du nom d'un des trois nouveaux pods. Quand vous avez vu la page, fermez le tunnel. `pkill -f` arrête le programme dont la ligne de commande contient le texte donné :

```exec
pkill -f "port-forward deployment/vote 28640"
```

> 📖 [Les namespaces](https://kubernetes.io/fr/docs/concepts/overview/working-with-objects/namespaces/) · [`kubectl config set-context`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_set-context/)
