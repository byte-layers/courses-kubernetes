---
title: "Sans sonde, une version qui plante retire des pods sains"
description: "Déployer une version qui plante au démarrage et compter les pods sains qui restent, puis rétablir le service"
---

L'équipe de développement a livré une v3 de `vote`, avec un titre personnalisable. Cette version exige un réglage, `VOTE_TITLE`, que personne ne lui a donné : sans lui, elle **plante au démarrage**. Vous allez la déployer telle quelle, pour voir ce que Kubernetes fait des pods sains pendant ce temps.

## Déployer la v3, qui plante au démarrage

Déployer une nouvelle version, c'est changer l'image dans le fichier du Deployment. Ouvrez `~/byte-layers/kubernetes/l6/vote.deploy.yml` dans votre éditeur et repérez la ligne marquée `# TODO` :

```yaml
          image: ghcr.io/bngams/kube-vote:2.0    # TODO : seule la version, après les deux-points, change
```

Changez **seulement les chiffres de la fin** pour passer en version `3.0`, enregistrez, puis appliquez :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Kubernetes remplace les pods un par un : il crée un pod de la nouvelle version, attend qu'il soit **prêt**, retire un pod de l'ancienne, et recommence. Suivez cette mise à jour pendant 45 secondes. Faites une prévision : puisque les pods de la v3 plantent, Kubernetes va-t-il retirer des pods de la v2 ?

```exec
kubectl rollout status deployment/vote --timeout=45s
```

La mise à jour reste bloquée en chemin, et la commande abandonne au bout du délai. La première ligne se répète ; en voici l'essentiel :

```text
Waiting for deployment "vote" rollout to finish: 1 out of 3 new replicas have been updated...
error: timed out waiting for the condition
```

Si une ligne `2 out of 3 new replicas have been updated` s'est glissée avant l'erreur, Kubernetes a déjà créé un deuxième pod de la v3 : la section suivante dit ce que cela veut dire.

## Laisser à Kubernetes l'occasion de retirer un pod sain

Un conteneur de la v3 ne tourne qu'une fraction de seconde avant de planter. Or, sans autre information, Kubernetes compte un pod comme **prêt** dès que son conteneur a démarré. S'il regarde le pod pendant cette fraction de seconde, il le croit prêt, et retire aussitôt un pod de la v2. S'il regarde un instant trop tard, il ne se passe rien jusqu'au redémarrage suivant du conteneur, et ces redémarrages s'espacent de plus en plus.

C'est donc une affaire de chance. Lors de nos essais, un pod sain avait déjà été retiré au bout de ces 45 secondes environ une fois sur deux.

Pour ne pas dépendre de cette chance, la commande qui suit donne à la panne d'autres occasions. Elle attend 6 secondes que le Deployment déclare qu'il lui manque un pod disponible. Si rien ne vient, elle rejoue la livraison : elle revient à la version précédente, la v2, avec `kubectl rollout undo`, puis elle applique de nouveau votre fichier, ce qui démarre un pod de la v3 tout neuf. Elle recommence ainsi jusqu'à sept fois, et s'arrête dès qu'un pod sain a été retiré.

```exec
for essai in 1 2 3 4 5 6 7; do
  kubectl wait deployment/vote --for=condition=Available=False --timeout=6s && break
  kubectl rollout undo deployment/vote && kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
done
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `for essai in 1 2 3 4 5 6 7; do … done` | répète ce qui est entre `do` et `done`, sept fois au plus |
> | `kubectl wait deployment/vote` | attend qu'une condition soit remplie sur le Deployment `vote`, puis rend la main |
> | `--for=condition=Available=False` | la condition attendue : le Deployment déclare qu'il n'a plus son compte de pods disponibles, ici 3 (sa condition `Available` passe à `False`) |
> | `--timeout=6s` | abandonne au bout de 6 secondes si la condition n'est pas remplie |
> | `&& break` | `&&` veut dire « si la commande qui précède a réussi » : `break` sort alors de la boucle |
> | `kubectl rollout undo deployment/vote` | remet le Deployment sur sa version précédente, ici la v2 : le pod de la v3 est supprimé |
> | `&& kubectl apply -f …/vote.deploy.yml` | si le retour en arrière a réussi, applique de nouveau votre fichier, qui demande la v3 |

La dernière ligne de la réponse doit être celle-ci :

```text
deployment.apps/vote condition met
```

Si elle est seule, un pod sain avait déjà été retiré pendant les 45 premières secondes. Sinon, chaque essai manqué a d'abord affiché trois lignes, `timed out waiting for the condition`, `rolled back` et `configured`. Lors de nos essais, il n'a jamais fallu plus de quatre essais. Si la commande se termine chez vous sans `condition met`, relancez-la.

## Compter les pods sains qui restent

Regardez les pods de `vote`. L'option `-l app=vote` limite l'affichage aux pods qui portent l'étiquette `app: vote` :

```exec
kubectl get pods -l app=vote
```

Les pods de la v3 sont les plus récents, dans la colonne `AGE`, et leur colonne `READY` dit `0/1`. Leur statut alterne entre `Error` (le conteneur vient de s'arrêter en erreur) et `CrashLoopBackOff` (il plante en boucle, et Kubernetes le relance en attendant de plus en plus longtemps entre deux essais), et leur colonne `RESTARTS` augmente à chaque relance.

```text
NAME                    READY   STATUS    RESTARTS   AGE
vote-566d594846-cx8lp   1/1     Running   0          68s
vote-566d594846-qr8nl   1/1     Running   0          68s
vote-56f645655-8nq6r    0/1     Error     0          2s
vote-56f645655-t4w7q    0/1     Error     0          1s
```

Il y a maintenant **deux** pods de la v3 : Kubernetes en a créé un deuxième parce qu'il a cru le premier prêt. Regardez ce que cela a coûté, sur le Deployment :

```exec
kubectl get deployments vote
```

La colonne `READY` ne dit plus `3/3` :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   2/3     2            2           68s
```

L'application répond encore, grâce aux pods de la v2 qui restent. Mais il n'en reste que **2 sur 3** : un pod sain a été retiré ! Si vous lisez `1/3`, un deuxième pod sain est parti de la même façon, et rien n'empêche le dernier de suivre.

> 🧠 **Pourquoi Kubernetes a-t-il retiré un pod sain ?** Parce qu'il a cru un pod de la v3 **prêt**. Sans autre information, un pod est considéré comme prêt dès que son conteneur démarre… une fraction de seconde avant que la v3 ne plante. La mise à jour a donc continué sur une version cassée.

```hint
Sur la ligne `image:` de `vote.deploy.yml`, remplacez `2.0` par `3.0`, enregistrez, puis lancez `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml`.
```

```hint
Une fois le fichier appliqué, il faut quelques secondes pour qu'un pod de la v3 ait planté au moins une fois : `kubectl get pods -l app=vote` montre sa colonne `RESTARTS`.
```

```hint
Si `kubectl get deployments vote` dit encore `3/3`, aucun pod sain n'a été retiré pour l'instant : relancez la commande de la section « Laisser à Kubernetes l'occasion de retirer un pod sain », jusqu'à ce qu'elle se termine par `condition met`.
```

```verify
IMG=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "Le Deployment vote est introuvable dans le namespace courant. Rejouez l'étape « Partir de l'application de vote en version 2.0 »."
  exit 1
fi
case "$IMG" in
  *kube-vote:3.0) : ;;
  *) echo "Le cluster demande l'image $IMG, pas la 3.0. Dans vote.deploy.yml, passez la version à 3.0, enregistrez, puis appliquez le fichier."
     exit 1 ;;
esac
if [ -n "$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}' 2>/dev/null)" ]; then
  echo "Ce Deployment a déjà une sonde de readiness : le constat de cette étape se fait sans sonde. Rejouez l'étape « Partir de l'application de vote en version 2.0 », puis celle-ci."
  exit 1
fi
for i in 1 2 3 4 5 6 7 8 9; do
  R=$(kubectl get pods -l app=vote -o jsonpath='{range .items[*]}{.spec.containers[0].image}{" "}{.status.containerStatuses[0].restartCount}{"\n"}{end}' 2>/dev/null | grep 'kube-vote:3.0 [1-9]')
  A=$(kubectl get deployment vote -o jsonpath='{.status.availableReplicas}' 2>/dev/null)
  [ -n "$R" ] && [ "${A:-0}" -lt 3 ] && break
  sleep 5
done
if [ -z "$R" ]; then
  echo "La v3 est demandée, mais aucun de ses pods n'a encore planté. Attendez quelques secondes, regardez kubectl get pods -l app=vote, puis réessayez."
  exit 1
fi
if [ "${A:-0}" -ge 3 ]; then
  echo "La v3 est déployée sans sonde et ses pods plantent, mais les 3 pods de la v2 sont encore disponibles : Kubernetes n'a pas encore cru un pod de la v3 prêt. Relancez la commande de la section « Laisser à Kubernetes l'occasion de retirer un pod sain », puis réessayez."
  exit 1
fi
echo "La v3 est déployée sans sonde, ses pods plantent, et il ne reste que ${A:-0} pod(s) disponible(s) sur 3 : Kubernetes a retiré au moins un pod sain."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué. La boucle de la fin est celle de la section « Laisser à Kubernetes l'occasion de retirer un pod sain » :

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
          image: ghcr.io/bngams/kube-vote:3.0
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
for essai in 1 2 3 4 5 6 7; do
  kubectl wait deployment/vote --for=condition=Available=False --timeout=6s && break
  kubectl rollout undo deployment/vote && kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
done
```
````

## Rétablir le service en revenant à la version précédente

Lancez la vérification ci-dessus **maintenant**, pendant que la panne est visible : la suite de cette étape la fait disparaître. Rétablissez ensuite le service. Kubernetes garde un historique des versions d'un Deployment, et `kubectl rollout undo`, la commande dont la boucle s'est servie, le remet sur la précédente, ici la v2. Lancez-la **une seule fois** : un second `undo` reviendrait à la v3 cassée.

```exec
kubectl rollout undo deployment/vote
```

La réponse confirme le retour en arrière :

```text
deployment.apps/vote rolled back
```

Cette commande a modifié le cluster sans toucher à votre fichier, qui demande toujours la `3.0` : l'étape suivante le réécrit. Attendez la fin du retour en arrière :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Quand la commande répond `successfully rolled out`, les pods de la v3 ont disparu, et les 3 pods de la v2 sont de retour : Kubernetes a recréé celui qui manquait.
