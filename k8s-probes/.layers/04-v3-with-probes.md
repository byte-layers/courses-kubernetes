---
title: "La v3 qui plante, avec les sondes : les anciens pods continuent de servir"
description: "Rejouer la même panne avec les sondes en place, constater que les 3 pods sains restent, puis revenir en arrière"
---

Les sondes sont en place. Rejouez maintenant la même erreur qu'au début du cours : déployez la v3, qui plante toujours au démarrage, puisque personne ne lui a encore donné son réglage.

## Redéployer la v3 avec les sondes

Dans `~/byte-layers/kubernetes/l6/vote.deploy.yml`, passez l'image de `2.0` à `3.0`, sans toucher aux sondes. Enregistrez, puis appliquez :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Suivez la mise à jour pendant 45 secondes. Faites une prévision : cette fois, combien de pods de la v2 Kubernetes va-t-il retirer ?

```exec
kubectl rollout status deployment/vote --timeout=45s
```

La commande abandonne de nouveau au bout du délai. Lisez où elle est restée bloquée : un seul nouveau pod a été créé, et Kubernetes n'est pas allé plus loin :

```text
Waiting for deployment "vote" rollout to finish: 1 out of 3 new replicas have been updated...
error: timed out waiting for the condition
```

## Compter les pods sains avec les sondes

Regardez les pods de `vote` :

```exec
kubectl get pods -l app=vote
```

Un seul pod de la v3 a été créé, et il plante en boucle. Les trois autres sont ceux de la v2 :

```text
NAME                    READY   STATUS             RESTARTS      AGE
vote-68596c5d8c-zjd4q   0/1     CrashLoopBackOff   2 (19s ago)   45s
vote-76d84bc99b-2mlg9   1/1     Running            0             51s
vote-76d84bc99b-9bkw8   1/1     Running            0             47s
vote-76d84bc99b-gxgjw   1/1     Running            0             49s
```

Regardez le Deployment :

```exec
kubectl get deployments vote
```

La colonne `READY` dit toujours `3/3`, et `UP-TO-DATE` indique qu'un seul pod est à la nouvelle version :

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   3/3     1            3           106s
```

Cette fois, **les 3 pods v2 sont intacts** : le pod v3 n'a jamais répondu à la sonde de *readiness*, il n'a donc jamais été compté comme prêt, et la mise à jour s'est arrêtée là. Ce n'est plus une affaire de chance : vous pourriez rejouer cette livraison dix fois, aucun pod de la v2 ne partirait. Les utilisateurs ne voient rien.

```hint
Seule la ligne `image:` change : `2.0` devient `3.0`. Les blocs `readinessProbe` et `livenessProbe` restent tels quels. Enregistrez, puis appliquez le fichier.
```

```hint
Une fois le fichier appliqué, il faut une vingtaine de secondes pour que le pod de la v3 ait planté au moins une fois : `kubectl get pods -l app=vote` montre sa colonne `RESTARTS`.
```

```verify
IMG=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "Le Deployment vote est introuvable dans le namespace courant. Reprenez le cours à sa première étape."
  exit 1
fi
if [ "$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.path}' 2>/dev/null)" != "/healthz" ]; then
  echo "Le Deployment vote n'a plus sa sonde de readiness sur /healthz. Reprenez l'étape précédente : la v3 doit être déployée avec les sondes."
  exit 1
fi
case "$IMG" in
  *kube-vote:3.0) : ;;
  *) echo "Le cluster demande l'image $IMG, pas la 3.0. Dans vote.deploy.yml, passez la version à 3.0 sans toucher aux sondes, enregistrez, puis appliquez le fichier."
     exit 1 ;;
esac
for i in 1 2 3 4 5 6 7 8; do
  R=$(kubectl get pods -l app=vote -o jsonpath='{range .items[*]}{.spec.containers[0].image}{" "}{.status.containerStatuses[0].restartCount}{"\n"}{end}' 2>/dev/null | grep 'kube-vote:3.0 [1-9]')
  [ -n "$R" ] && break
  sleep 5
done
if [ -z "$R" ]; then
  echo "La v3 est demandée, mais son pod n'a pas encore planté. Attendez quelques secondes, regardez kubectl get pods -l app=vote, puis réessayez."
  exit 1
fi
A=$(kubectl get deployment vote -o jsonpath='{.status.availableReplicas}' 2>/dev/null)
if [ "${A:-0}" -lt 3 ]; then
  echo "Il ne reste que ${A:-0} pod(s) disponible(s) sur 3, alors que les sondes devaient protéger les pods de la v2. Regardez kubectl get pods -l app=vote : s'ils viennent d'être recréés, attendez qu'ils soient prêts, puis réessayez."
  exit 1
fi
echo "Le pod de la v3 plante, et les 3 pods de la v2 sont toujours disponibles : la sonde de readiness a arrêté la mise à jour."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l6/vote.deploy.yml <<'EOF'
# vote : 3 réplicas, réglages lus dans la ConfigMap vote-config, sondes de santé
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
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```
````

> 🧠 **Les sondes sont le filet de sécurité des mises à jour.** Sans elles, Kubernetes ne sait qu'une chose : « le programme a démarré ». Avec elles, il sait « l'application répond correctement ». C'est l'une des premières choses à vérifier dans un projet : *nos applications ont-elles des sondes de santé ?*

## Revenir en arrière

Lancez la vérification ci-dessus **maintenant**, pendant que la panne est visible : la suite de cette étape la fait disparaître. Il ne reste ensuite qu'à retirer le pod de la v3, en revenant à la version précédente, **une seule fois** :

```exec
kubectl rollout undo deployment/vote
```

Attendez que le retour en arrière soit terminé :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

La commande répond `successfully rolled out` : le pod de la v3 a disparu, et les 3 pods de la v2 n'ont pas bougé.

## Remettre le fichier en cohérence avec le cluster

`kubectl rollout undo` a modifié le cluster **sans toucher à votre fichier** : `vote.deploy.yml` demande toujours la `3.0`, alors que le cluster est revenu en `2.0`. Tant que les deux ne disent pas la même chose, le prochain `kubectl apply` ramènerait la v3 cassée.

Dans `~/byte-layers/kubernetes/l6/vote.deploy.yml`, repassez l'image en `2.0`, celle qui tourne réellement, sans toucher aux sondes. Enregistrez, puis appliquez :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Le terminal répond `configured`, mais aucun pod ne change, puisque le cluster est déjà en v2 : `kubectl` a seulement noté que votre fichier et le cluster sont de nouveau d'accord.

```hint
La ligne `image:` de votre fichier doit redevenir `image: ghcr.io/bngams/kube-vote:2.0`. Enregistrez, puis lancez `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml`.
```

```verify
F="$HOME/byte-layers/kubernetes/l6/vote.deploy.yml"
IMG=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
if [ -z "$IMG" ]; then
  echo "Le Deployment vote est introuvable dans le namespace courant. Reprenez le cours à sa première étape."
  exit 1
fi
if ! grep -Eq '^[[:space:]]*image:[[:space:]]*"?ghcr\.io/bngams/kube-vote:2\.0' "$F" 2>/dev/null; then
  echo "Votre fichier vote.deploy.yml ne demande pas l'image ghcr.io/bngams/kube-vote:2.0. Remettez 2.0 sur la ligne image:, enregistrez, puis appliquez."
  exit 1
fi
LAST=$(kubectl get deployment vote -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' 2>/dev/null)
case "$LAST" in
  *kube-vote:2.0*) : ;;
  *) echo "Votre fichier dit bien 2.0, mais le dernier fichier appliqué au cluster demandait une autre version. Lancez kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml"
     exit 1 ;;
esac
case "$IMG" in
  *kube-vote:2.0) : ;;
  *) echo "Le cluster demande l'image $IMG, pas la 2.0. Appliquez votre fichier."
     exit 1 ;;
esac
if [ "$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.path}' 2>/dev/null)" != "/healthz" ]; then
  echo "Le Deployment vote n'a plus sa sonde de readiness sur /healthz : seule la ligne image: devait changer. Reprenez le fichier de l'étape précédente."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=45s >/dev/null 2>&1; then
  echo "Le retour à la 2.0 n'est pas terminé. Regardez kubectl get pods -l app=vote, puis réessayez."
  exit 1
fi
echo "Le cluster tourne en 2.0 avec ses sondes, et votre fichier dit la même chose."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l6/vote.deploy.yml <<'EOF'
# vote : 3 réplicas, réglages lus dans la ConfigMap vote-config, sondes de santé
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
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````
