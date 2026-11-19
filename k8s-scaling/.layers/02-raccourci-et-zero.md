---
title: "Le raccourci kubectl scale, puis zéro réplica"
description: "Changer le nombre de réplicas sans passer par le fichier, voir ce changement effacé, puis éteindre et rallumer l'application"
---

Modifier un fichier puis l'appliquer, c'est un peu long. Il existe un raccourci, que vous allez essayer pour voir ce qu'il coûte. Vous éteignez ensuite l'application sans la supprimer, en demandant zéro réplica.

## Changer le nombre de réplicas sans le fichier : kubectl scale

`kubectl scale` change directement le nombre de réplicas d'un Deployment, sans passer par le fichier. Demandez 5 exemplaires :

```exec
kubectl scale deployment vote --replicas=5
```

Le terminal confirme avec `deployment.apps/vote scaled`. Attendez que les nouveaux pods soient prêts :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Regardez ensuite le Deployment, pour lire combien de pods sont prêts :

```exec
kubectl get deployments
```

Ça fonctionne : 5 pods tournent.

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   5/5     5            5           104s
```

Regardez maintenant ce que dit le fichier. `grep` affiche les lignes d'un fichier qui contiennent un mot, ici `replicas` :

```exec
grep replicas ~/byte-layers/kubernetes/l2/vote.deploy.yml
```

Le fichier n'a pas bougé, il indique toujours 3 réplicas :

```text
  replicas: 3
```

**Le cluster et le fichier ne disent plus la même chose.**

```ack
Le cluster a 5 pods, le fichier en demande 3
```

## Le prochain apply efface le raccourci

Imaginez maintenant qu'un collègue applique ce fichier demain, pour une toute autre raison. Jouez ce collègue. Avant de lancer, faites une prévision : combien de pods restera-t-il ?

```exec
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
```

Le terminal répond `configured` : quelque chose a donc changé dans le cluster.

```text
deployment.apps/vote configured
```

Laissez au Deployment le temps de se mettre à jour, puis regardez-le :

```exec
kubectl rollout status deployment/vote --timeout=120s
kubectl get deployments
```

Retour à 3 réplicas : deux pods ont été arrêtés. Si vous lisez encore `5/3`, ces deux pods sont en train de s'arrêter : relancez la commande.

```text
deployment "vote" successfully rolled out
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   3/3     3            3           107s
```

Le changement fait avec `kubectl scale` a été **effacé sans prévenir**. Kubernetes a simplement appliqué le dernier état souhaité qu'on lui a transmis.

> 🧠 **Le fichier fait foi.** En entreprise, les fichiers YAML sont rangés dans un outil de gestion de versions (Git) : on sait qui a changé quoi, quand et pourquoi, et l'on peut revenir en arrière. Certaines équipes vont jusqu'à interdire les modifications directes : un outil applique automatiquement ce qui est dans Git (c'est le principe du **GitOps**).
>
> ⚖️ `kubectl scale` reste utile en urgence, par exemple pour absorber un pic de visiteurs. Mais il faut **reporter ensuite la valeur dans le fichier**, sinon elle sera perdue au prochain `apply`.

```ack
Le raccourci a été effacé par le fichier
```

## Éteindre sans supprimer : zéro réplica

Pour arrêter une application, vous savez supprimer son Deployment. Il existe une solution plus douce : demander **zéro** réplica. Le Deployment reste alors en place dans le cluster, avec ses réglages et l'historique de ses versions, et les autres objets qui pointent vers lui continuent de fonctionner. Pour rallumer, il suffit de changer un chiffre.

Dans `~/byte-layers/kubernetes/l2/vote.deploy.yml`, la ligne indique `replicas: 3`. Remplacez le chiffre pour ne demander aucun exemplaire, puis enregistrez. Voici la ligne à compléter :

```yaml
spec:
  replicas: # TODO : aucun exemplaire
```

Transmettez ce nouvel état souhaité au cluster, en appliquant le fichier :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
```

Attendez que les pods aient disparu. `kubectl wait --for=delete` rend la main quand les pods désignés n'existent plus, ou au bout de 30 secondes. Si les pods ont déjà fini de s'arrêter quand vous la lancez, elle rend la main tout de suite :

```exec
kubectl wait --for=delete pod -l app=vote --timeout=30s
```

Regardez d'abord le Deployment, pour vérifier qu'il existe encore :

```exec
kubectl get deployments
```

Il affiche `0/0` : aucun pod prêt, sur aucun demandé.

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
vote   0/0     0            0           2m
```

Regardez ensuite les pods, avec la commande habituelle :

```exec
kubectl get pods
```

Plus aucun pod ne tourne, mais le Deployment est **toujours là**.

```text
No resources found in default namespace.
```

```hint
C'est la même ligne qu'à l'étape précédente, celle qui commence par `replicas:`. « Aucun exemplaire » s'écrit avec un chiffre.
```

```hint
La ligne devient `  replicas: 0`. Enregistrez, puis relancez `kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml`.
```

```verify
F="$HOME/byte-layers/kubernetes/l2/vote.deploy.yml"
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote n'existe plus : il fallait demander zéro réplica, pas le supprimer. Remettez-le en place avec kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml"
  exit 1
fi
IN_FILE=$(sed -E 's/#.*//' "$F" 2>/dev/null | grep -E '^[[:space:]]*replicas:' | head -1 | tr -cd '0-9')
if [ "$IN_FILE" != "0" ]; then
  echo "Le fichier vote.deploy.yml ne demande pas 0 réplica (trouvé : ${IN_FILE:-rien}). Écrivez replicas: 0, enregistrez, puis relancez kubectl apply."
  exit 1
fi
WANT=$(kubectl get deployment vote -o jsonpath='{.spec.replicas}')
if [ "$WANT" != "0" ]; then
  echo "Le fichier demande 0 réplica, mais le cluster en demande encore $WANT : il reste à lancer kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml"
  exit 1
fi
kubectl wait --for=delete pod -l app=vote --timeout=40s >/dev/null 2>&1
LEFT=$(kubectl get pods -l app=vote -o name | wc -l | tr -d ' ')
if [ "$LEFT" != "0" ]; then
  echo "Il reste $LEFT pod(s) de vote en train de s'arrêter. Attendez quelques secondes, puis réessayez."
  exit 1
fi
echo "Plus aucun pod, et le Deployment vote est toujours là."
```

````solution
Le fichier complet à cette étape, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l2/vote.deploy.yml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: vote
spec:
  replicas: 0
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
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
kubectl wait --for=delete pod -l app=vote --timeout=50s
```
````

## Rallumer l'application

À vous de rallumer l'application, sans bloc à copier : remettez `3` dans la ligne `replicas` du fichier, enregistrez, puis appliquez de nouveau avec la commande que vous connaissez. Quelques secondes plus tard, `kubectl get deployments` doit afficher `3/3`. Voici la ligne à compléter :

```yaml
spec:
  replicas: # TODO : de nouveau 3 exemplaires
```

> 🗣️ **En réunion projet.** « *On éteint les environnements de recette le soir et le week-end* » : c'est souvent exactement cela, des réplicas à 0. Dans le cloud, on paie les serveurs à l'heure : moins de pods, c'est moins de serveurs nécessaires, donc une facture plus légère, à condition que le cluster retire les serveurs devenus inutiles (beaucoup de clouds savent le faire automatiquement). Le lundi matin, on remet les réplicas, et tout redémarre à l'identique.

```hint
La commande est toujours la même : `kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml`. Pour attendre que les pods soient prêts : `kubectl rollout status deployment/vote --timeout=120s`.
```

```verify
F="$HOME/byte-layers/kubernetes/l2/vote.deploy.yml"
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Le Deployment vote n'existe pas. Appliquez le fichier : kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml"
  exit 1
fi
IN_FILE=$(sed -E 's/#.*//' "$F" 2>/dev/null | grep -E '^[[:space:]]*replicas:' | head -1 | tr -cd '0-9')
WANT=$(kubectl get deployment vote -o jsonpath='{.spec.replicas}')
if [ "$IN_FILE" != "3" ] || [ "$WANT" != "3" ]; then
  echo "Attendu : 3 réplicas dans le fichier et dans le cluster. Trouvé : ${IN_FILE:-rien} dans le fichier, $WANT dans le cluster."
  echo "Si le cluster a 3 et le fichier 0, vous avez pris le raccourci : le prochain apply éteindrait tout. Corrigez le fichier, puis appliquez-le."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "L'application se rallume encore. Regardez les pods avec kubectl get pods, puis réessayez."
  exit 1
fi
if [ "$(kubectl get deployment vote -o jsonpath='{.status.readyReplicas}')" != "3" ]; then
  echo "Les 3 pods ne sont pas tous prêts. Regardez leur état avec kubectl get pods, puis réessayez."
  exit 1
fi
echo "L'application est rallumée : 3 pods prêts, comme le dit le fichier."
```

````solution
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
EOF
kubectl apply -f ~/byte-layers/kubernetes/l2/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

> 📖 [`kubectl scale`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/) · [Gérer ses objets avec des fichiers de configuration](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/) · [Les Deployments](https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/)

## Récap

- Le nombre d'exemplaires est un **état souhaité** comme un autre : on change `replicas`, Kubernetes calcule le reste.
- Les réplicas sont des **copies identiques**. Pour leur répartir les visiteurs, il faudra un **Service**, dans un prochain cours.
- **Le fichier fait foi** : un raccourci comme `kubectl scale` est effacé par le prochain `apply`.
- **0 réplica** éteint l'application sans la supprimer : pratique pour économiser.

À la fin de ce cours, le Deployment `vote` tourne avec **3 réplicas**, et `~/byte-layers/kubernetes/l2/vote.deploy.yml` dit la même chose. Le fichier complet est dans [`solution/vote.deploy.yml`](../solution/vote.deploy.yml).

## Questions sur le raccourci et l'extinction

```quiz
title: "Le raccourci kubectl scale, puis zéro réplica"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Le fichier dit `replicas: 3`. Vous lancez `kubectl scale deployment vote --replicas=5`, puis un collègue applique le fichier. Combien de pods tournent ensuite ?"
    answers:
      - "5 : la dernière commande tapée l'emporte toujours"
      - "3 : `kubectl apply` transmet l'état souhaité du fichier, et le raccourci est effacé"
      - "8 : les deux demandes s'additionnent"
    correct: 2
    explanation: "Kubernetes applique le dernier état souhaité qu'on lui transmet. Le fichier disait 3 : les deux pods en trop sont arrêtés."
  - prompt: "Après un `kubectl scale` fait en urgence, que faut-il faire pour que le changement dure ?"
    answers:
      - "Rien : le cluster s'en souvient"
      - "Reporter la nouvelle valeur dans le fichier"
      - "Supprimer le fichier"
    correct: 2
    explanation: "Tant que le fichier dit autre chose, le prochain `apply` annule le changement."
  - prompt: "Quelle différence entre demander `replicas: 0` et supprimer le Deployment ?"
    answers:
      - "Aucune : dans les deux cas, il ne reste rien dans le cluster"
      - "À 0 réplica, le Deployment reste dans le cluster avec ses réglages : il suffit de changer un chiffre pour rallumer"
      - "À 0 réplica, les pods restent en place mais ne répondent plus"
    correct: 2
    explanation: "Zéro réplica arrête les pods et garde le Deployment. C'est ce qu'on fait souvent pour éteindre un environnement de test le soir."
```
