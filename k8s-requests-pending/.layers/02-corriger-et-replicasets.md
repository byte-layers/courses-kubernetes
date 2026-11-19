---
title: "Une réservation réaliste, et un ReplicaSet par version"
description: "Corriger la réservation à un dixième de processeur, voir les pods remplacés un par un, puis lire les ReplicaSets que chaque changement a créés"
---

Le pod à 64 processeurs n'aura jamais de place. Vous corrigez la réservation avec une valeur réaliste, puis vous regardez ce que chacun de vos changements a laissé dans le cluster.

## Revenir à une réservation réaliste

Notre petite application se contente d'un dixième de processeur. Dans `~/byte-layers/kubernetes/l2/vote.deploy.yml`, la ligne indique `cpu: 64`. Remplacez la valeur, puis enregistrez. Voici la ligne à compléter, avec ses voisines pour vous repérer :

```yaml
          resources:
            requests:
              cpu: # TODO : un dixième de processeur, écrit 100m
              memory: 64Mi
```

Transmettez cette nouvelle réservation au cluster, en appliquant le fichier :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
```

Kubernetes remplace alors les pods un par un : pendant quelques secondes, des nouveaux pods `Running` côtoient des anciens en train de s'arrêter. Attendez la fin du remplacement. `kubectl rollout status` suit le déploiement et rend la main quand il est terminé, ou au bout de 50 secondes :

```exec
kubectl rollout status deployment/vote --timeout=50s
```

Avant de lister les pods, faites une prévision : le début de leur nom sera-t-il celui des anciens pods, celui du pod `Pending`, ou un troisième ?

```exec
kubectl get pods
```

Il ne reste que trois pods, tous avec un **nouveau début de nom** :

```text
NAME                  READY   STATUS    RESTARTS   AGE
vote-fc57495c-8gxtn   1/1     Running   0          12s
vote-fc57495c-sfzlh   1/1     Running   0          11s
vote-fc57495c-t952v   1/1     Running   0          13s
```

```hint
Une seule valeur change : celle de la ligne `cpu:`. Un dixième de processeur s'écrit en millièmes, avec la lettre `m`.
```

```hint
La ligne devient `              cpu: 100m`. Enregistrez, puis relancez `kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml`.
```

```verify
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote n'existe pas. Appliquez le fichier : kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml"
  exit 1
fi
CPU=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')
MEM=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')
case "$CPU" in
  100m|0.1) : ;;
  *) echo "Le Deployment vote réserve « ${CPU:-rien} » de processeur, et non 100m. Corrigez la ligne cpu, enregistrez, puis relancez kubectl apply."
     exit 1 ;;
esac
if [ "$MEM" != "64Mi" ]; then
  echo "Le Deployment vote réserve « ${MEM:-rien} » de mémoire, et non 64Mi. Gardez la ligne memory: 64Mi."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "Le remplacement des pods n'est pas terminé. Regardez kubectl get pods, puis réessayez."
  exit 1
fi
READY=$(kubectl get deployment vote -o jsonpath='{.status.readyReplicas}')
PENDING=$(kubectl get pods -l app=vote --field-selector=status.phase=Pending -o name | wc -l | tr -d ' ')
if [ "$READY" != "3" ] || [ "$PENDING" != "0" ]; then
  echo "Attendu : 3 pods prêts et aucun pod Pending. Trouvé : ${READY:-0} prêt(s), $PENDING Pending. Regardez kubectl get pods."
  exit 1
fi
echo "3 pods prêts, chacun réservant un dixième de processeur et 64 Mi de mémoire."
```

````solution
Le fichier complet, solution finale du cours, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l2/vote.deploy.yml <<'EOF'
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
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

## Un nouveau ReplicaSet à chaque changement du modèle de pod

Pour comprendre ce changement de nom, listez les ReplicaSets :

```exec
kubectl get replicasets
```

Vous en lisez trois, alors qu'un seul fait tourner des pods :

```text
NAME              DESIRED   CURRENT   READY   AGE
vote-584fbb796    0         0         0       40s
vote-644d47956b   0         0         0       6m
vote-fc57495c     3         3         3       13s
```

Chaque modification du modèle de pod a créé un **nouveau ReplicaSet** : `644d47956b` pour la version d'origine, `584fbb796` pour la version à 64 processeurs (jamais démarrée), `fc57495c` pour la version actuelle. Les anciens sont gardés à 0 : ils serviront à **revenir en arrière**, ce que montre un prochain cours.

Changer seulement le nombre de `replicas`, en revanche, ne crée **pas** de nouveau ReplicaSet : le modèle de pod ne change pas, c'est le même ReplicaSet qui ajoute ou retire des pods.

> 🗣️ **En réunion projet.** Les *requests* sont la base du **dimensionnement** et des **coûts** : la somme des réservations dit combien de nœuds il faut, donc combien de serveurs payer. Un pod `Pending` en production signifie souvent « le cluster est plein » : il faut alors ajouter des nœuds (beaucoup de clouds le font automatiquement), ou revoir les réservations à la baisse.

```ack
J'ai retrouvé mes trois versions dans la liste des ReplicaSets
```

> 📖 [Gestion des ressources des conteneurs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [Les ReplicaSets](https://kubernetes.io/fr/docs/concepts/workloads/controllers/replicaset/) · [Les Deployments](https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/)

## Récap

- Les **requests** réservent du processeur et de la mémoire. L'ordonnanceur compte les réservations, pas la consommation réelle.
- Sans place disponible, un pod reste `Pending`, et `kubectl describe` en donne la raison dans ses événements.
- Lors d'une mise à jour, les anciens pods continuent de servir tant que les nouveaux ne sont pas prêts.
- Chaque modification du modèle de pod crée un **nouveau ReplicaSet**, ce qui prépare les mises à jour et les retours en arrière.

À la fin de ce cours, le Deployment `vote` tourne avec **3 réplicas, chacun réservant `100m` de processeur et `64Mi` de mémoire**, et `~/byte-layers/kubernetes/l2/vote.deploy.yml` dit la même chose : c'est l'état dont partent les cours suivants. Le fichier complet est dans [`solution/vote.deploy.yml`](../solution/vote.deploy.yml).

## Questions sur la correction et les ReplicaSets

```quiz
title: "Une réservation réaliste, et un ReplicaSet par version"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Lequel de ces changements crée un nouveau ReplicaSet ?"
    answers:
      - "Passer `replicas` de 3 à 5"
      - "Changer la réservation de processeur du conteneur"
      - "Supprimer un pod"
    correct: 2
    explanation: "Un nouveau ReplicaSet naît quand le modèle de pod change. Le nombre de réplicas n'en fait pas partie : le même ReplicaSet ajoute ou retire des pods."
  - prompt: "`kubectl get replicasets` affiche deux anciens ReplicaSets à 0. À quoi servent-ils ?"
    answers:
      - "À rien : c'est un oubli de Kubernetes, à nettoyer à la main"
      - "Ils gardent les versions précédentes du modèle de pod, pour pouvoir y revenir"
      - "Ils font tourner des pods cachés"
    correct: 2
    explanation: "Chaque ReplicaSet correspond à une version du modèle de pod. Gardés à 0, ils ne font rien tourner."
  - prompt: "En production, un pod reste `Pending` avec le message `Insufficient memory`. Quelles réponses sont raisonnables ? (plusieurs réponses)"
    answers:
      - "Ajouter un nœud au cluster"
      - "Revoir les réservations à la baisse, si elles sont surestimées"
      - "Supprimer le pod en boucle jusqu'à ce qu'il démarre"
    correct: [1, 2]
    explanation: "Le cluster est plein au sens des réservations : on ajoute de la place, ou on réserve moins. Recréer le pod ne change rien à la place disponible."

```

## Pour aller plus loin

Trois sujets prolongent ce cours, dans la documentation officielle :

- [Les limites](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) : à côté de `requests`, le champ `limits` fixe un plafond que le conteneur ne peut pas dépasser.
- [`kubectl top`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/) : affiche le processeur et la mémoire réellement consommés, à comparer aux réservations.
- [Le passage à l'échelle automatique](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/) : un *HorizontalPodAutoscaler* ajuste lui-même le nombre de réplicas selon la charge, et se sert des *requests* pour calculer un pourcentage.
