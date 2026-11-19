---
title: "Revenir en arrière avec kubectl rollout undo"
description: "Rétablir la version précédente en une commande, puis remettre le fichier en cohérence avec le cluster"
---

En production, quand une nouvelle version pose problème, la priorité est de **rétablir le service**, avant même de corriger le bug. Kubernetes garde l'historique : revenir à la révision précédente tient en une commande.

## Rétablir la version précédente

`kubectl rollout undo` remet le Deployment sur la révision d'avant, ici la v2 :

```exec
kubectl rollout undo deployment/vote
```

La réponse confirme le retour en arrière :

```text
deployment.apps/vote rolled back
```

> ⚠️ **Une seule fois !** Lancez `kubectl rollout undo` **une seule fois** : un second `undo` reviendrait à la révision d'avant… c'est-à-dire la v3 cassée. En cas de doute, `kubectl get deployments vote -o wide` montre, dans sa colonne `IMAGES`, la version que le cluster demande.

Suivez ce retour en arrière comme une mise à jour, puisque c'en est une : les pods de la v3 disparaissent, et si un pod de la v2 avait été retiré, il est recréé.

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Quand la commande répond `successfully rolled out`, regardez le Deployment :

```exec
kubectl get deployments vote
```

Vous retrouvez 3 pods sains en v2, en quelques secondes :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   3/3     3            3           2m13s
```

## Constater que le fichier ne fait plus foi

`kubectl rollout undo` a modifié le cluster **sans toucher à votre fichier**. Demandez au cluster quelle image il fait tourner. L'option `-o wide` ajoute des colonnes, dont `IMAGES` :

```exec
kubectl get deployments vote -o wide
```

Le cluster est bien revenu en `2.0` :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                         SELECTOR
vote   3/3     3            3           2m13s   vote         ghcr.io/bngams/kube-vote:2.0   app=vote
```

Regardez maintenant ce que dit votre fichier. `grep` affiche les lignes du fichier qui contiennent le mot `image:` :

```exec
grep image: ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Le fichier, lui, demande toujours la `3.0`. La ligne est suivie du commentaire `# TODO` si vous l'avez gardé :

```text
          image: ghcr.io/bngams/kube-vote:3.0
```

> ⚠️ **Le fichier ne fait plus foi !** C'est la même dérive qu'avec `kubectl scale`, celle que décrit [Le fichier fait foi](https://byte-layers.com/fr/bits/file-is-the-source) : le cluster et le fichier ne disent plus la même chose. Au prochain `kubectl apply`, la v3 cassée reviendrait.

## Remettre le fichier en cohérence avec le cluster

Dans `~/byte-layers/kubernetes/l6/vote.deploy.yml`, repassez l'image en version `2.0`, celle qui tourne réellement, puis enregistrez. Appliquez ensuite le fichier :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Le terminal répond `configured`, mais aucun pod ne change, puisque le cluster est déjà en v2 : `kubectl` a seulement noté que votre fichier et le cluster sont de nouveau d'accord.

```hint
La ligne `image:` de votre fichier doit dire la même chose que la colonne `IMAGES` du cluster : `ghcr.io/bngams/kube-vote:2.0`.
```

```hint
Après avoir enregistré le fichier, il faut l'appliquer : `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml`. La vérification regarde aussi quel fichier a été appliqué en dernier.
```

```verify
F="$HOME/byte-layers/kubernetes/l6/vote.deploy.yml"
IMG=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "Le Deployment vote est introuvable dans le namespace courant. Rejouez l'étape « Partir de l'application de vote en version 1.0 »."
  exit 1
fi
case "$IMG" in
  *kube-vote:2.0) : ;;
  *) echo "Le cluster demande l'image $IMG, pas la 2.0. Revenez en arrière, une seule fois, avec kubectl rollout undo deployment/vote."
     exit 1 ;;
esac
if ! grep -Eq '^[[:space:]]*image:[[:space:]]*"?ghcr\.io/bngams/kube-vote:2\.0' "$F" 2>/dev/null; then
  echo "Le cluster est revenu en 2.0, mais votre fichier vote.deploy.yml demande encore une autre version. Remettez 2.0 sur la ligne image:, enregistrez, puis appliquez."
  exit 1
fi
LAST=$(kubectl get deployment vote -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' 2>/dev/null)
case "$LAST" in
  *kube-vote:2.0*) : ;;
  *) echo "Votre fichier dit bien 2.0, mais le dernier fichier appliqué au cluster demandait une autre version. Lancez kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml"
     exit 1 ;;
esac
if ! kubectl rollout status deployment/vote --timeout=45s >/dev/null 2>&1; then
  echo "Le retour à la 2.0 n'est pas terminé. Regardez kubectl get pods -l app=vote, puis réessayez."
  exit 1
fi
echo "Le cluster tourne en 2.0 avec ses 3 pods, et votre fichier dit la même chose."
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

> 🗣️ **En réunion projet.** « *On a fait un rollback* » : on est revenu à la version précédente. C'est un geste normal, qui doit être **rapide** (quelques secondes ici) et **prévu à l'avance**. La question à poser avant chaque mise en production : « *comment revient-on en arrière si ça se passe mal ?* ». Attention toutefois : on ne revient facilement en arrière que sur le **code** ; si la nouvelle version a modifié des données, c'est une autre histoire.

## Récap

- Changer l'image dans le fichier suffit à déployer une nouvelle version : Kubernetes fait une **mise à jour progressive**, sans coupure.
- Chaque version du modèle de pod est une **révision**, gardée dans l'historique du Deployment.
- Face à une panne : **`describe`** pour l'état et les événements, **`logs`** pour la cause côté application.
- **`kubectl rollout undo`** rétablit la version précédente en quelques secondes. Ensuite, on remet le **fichier** en cohérence.

Le fichier final de ce cours se trouve dans [`solution/vote.deploy.yml`](../solution/vote.deploy.yml). L'application reste en place, avec `vote` en version 2.0 : la v3 n'est pas encore livrée, et pendant qu'elle plantait, Kubernetes a pu retirer un pod sain. Un prochain cours règle ces deux points.

## Pour aller plus loin

- Revenir à une révision précise, avec `--to-revision` : [`kubectl rollout undo`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_undo/).
- Régler le rythme de la mise à jour progressive, avec `maxSurge` et `maxUnavailable` : [la stratégie d'un Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy).

> 📖 [Revenir en arrière sur un Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment) · [`kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
