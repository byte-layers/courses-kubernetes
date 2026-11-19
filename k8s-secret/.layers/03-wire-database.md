---
title: "Brancher le Secret sur la base"
description: "Remplacer les identifiants écrits en clair dans le Deployment db par une référence au Secret, et vérifier que les votes arrivent toujours"
---

Le Secret existe, mais la base ne le lit pas encore : son Deployment contient toujours les deux valeurs en clair.

## Remplacer env par envFrom dans db.yml

Voici le fichier de la base, celui de l'étape de départ, où les cinq lignes du bloc `env:` sont remplacées par trois lignes. La commande l'écrit dans votre dossier de travail, avec un blanc à compléter. La ligne qui commence par `---` sépare le Deployment du Service :

```exec
cat > ~/byte-layers/kubernetes/l5/db.yml <<'EOF'
# db : les identifiants viennent du Secret "db-credentials" (à créer avant).
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: ghcr.io/bngams/kube-postgres:15-alpine
          envFrom:              # "prends des variables d'environnement…"
            - secretRef:        # "… dans un Secret…"
                name: # TODO : "… qui s'appelle" (le nom de votre Secret)
          ports:
            - containerPort: 5432
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
--- # objet suivant
apiVersion: v1
kind: Service
metadata:
  name: db
  labels:
    app: db
spec:
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
EOF
```

Ouvrez `~/byte-layers/kubernetes/l5/db.yml` dans votre éditeur, remplacez le `# TODO` et tout le texte qui le suit sur la ligne, puis enregistrez. Les trois lignes se lisent comme une phrase :

| Ligne | Rôle |
|---|---|
| `envFrom:` | « prends des variables d'environnement… » Elle remplace `env:`, où chaque variable était listée une par une avec sa valeur |
| `- secretRef:` | « … dans un Secret… » Le tiret ouvre une liste : un conteneur peut lire plusieurs sources |
| `name:` | « … qui s'appelle ainsi ». C'est le `metadata.name` du Secret |

`envFrom` importe d'un coup **toutes** les clés du Secret : le conteneur recevra `POSTGRES_USER` et `POSTGRES_PASSWORD`, sous les mêmes noms qu'avant. Le fichier ne contient plus aucune valeur sensible.

## Appliquer et attendre la nouvelle base

Appliquez cette nouvelle version :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l5/db.yml
```

Seul le Deployment a changé, le Service est resté tel quel :

```text
deployment.apps/db configured
service/db unchanged
```

Modifier le modèle de pod déclenche le remplacement du pod de la base. Attendez la fin de cette mise à jour avec `kubectl rollout status`, qui rend la main quand le nouveau pod est prêt :

```exec
kubectl rollout status deployment/db --timeout=120s
```

Regardez ensuite l'ensemble des pods de l'application :

```exec
kubectl get pods
```

Le pod de `db` est neuf, comme le montre son âge, plus petit que celui des autres :

```text
NAME                      READY   STATUS    RESTARTS      AGE
db-78fb95db9-xtnr2        1/1     Running   0             22s
redis-5b95d4d69f-zn7fh    1/1     Running   0             59s
result-656b779d76-hqdqw   1/1     Running   1 (21s ago)   58s
vote-7b9d75fc6f-k7m7q     1/1     Running   0             40s
vote-7b9d75fc6f-pvkp4     1/1     Running   0             41s
vote-7b9d75fc6f-xh8vp     1/1     Running   0             39s
worker-7b7bc56f88-8db8z   1/1     Running   1 (20s ago)   58s
```

Regardez aussi la colonne `RESTARTS` de `worker` et de `result` : elle est passée à 1. Ils ont perdu leur connexion à la base pendant son remplacement, ont planté, et Kubernetes les a relancés. Si elle affiche encore 0, ou si l'ancien pod de `db` apparaît encore en `Terminating`, relancez la commande une vingtaine de secondes plus tard.

Reposez enfin au Deployment la question du début de ce cours : que contient maintenant son champ `env` ? Faites une prévision avant de lancer la commande.

```exec
kubectl get deployment db -o jsonpath='{.spec.template.spec.containers[0].env}'
```

La commande ne répond rien : le champ `env` n'existe plus, et le mot de passe n'est plus écrit dans le Deployment. Le conteneur, lui, le reçoit toujours. `kubectl exec` lance une commande **dans** un conteneur, comme `docker exec`, et `printenv` affiche la valeur d'une variable d'environnement :

```exec
kubectl exec deployment/db -- printenv POSTGRES_USER POSTGRES_PASSWORD
```

Les deux identifiants viennent cette fois du Secret :

```text
postgres
postgres
```

> ⚠️ **Symptôme :** le nouveau pod de `db` reste en `CreateContainerConfigError`, et `kubectl rollout status` ne se termine pas.
> **Cause :** le Deployment cherche un Secret qui n'existe pas, par exemple parce que son nom est mal écrit. `kubectl describe pod <nom-du-pod>` l'indique en bas : `Error: secret "db-credentials" not found`.
> **Correctif :** vérifiez le nom dans les deux fichiers, puis appliquez de nouveau celui que vous avez corrigé. Le pod démarre tout seul quelques secondes plus tard.

## Vérifier que les votes arrivent dans la nouvelle base

La base est neuve : il reste à voir que l'application sait encore y écrire. Ouvrez un tunnel entre le port **28651** de votre machine et le port 80 d'un pod de `vote` :

```exec
kubectl port-forward deployment/vote 28651:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward deployment/vote` | ouvre un tunnel temporaire vers un pod du Deployment `vote` |
> | `28651:80` | le port 28651 de votre machine mène au port 80 du pod |
> | `> /dev/null` | jette ce que la commande afficherait, pour ne pas encombrer le terminal |
> | `2>&1` | jette aussi ses messages d'erreur, au même endroit |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main, et le tunnel reste ouvert |

Votez une fois. Dans un navigateur, ouvrez `http://localhost:28651` et cliquez sur une option ; sur un poste hébergé, passez par l'onglet **PORTS** de VS Code. Vous pouvez aussi voter depuis le terminal : cette commande envoie le formulaire de la page, comme le ferait le navigateur. `-d "vote=a"` est le choix de la première option, et `-s -o /dev/null` évite d'afficher la page reçue en réponse. La commande ne répond donc rien ; lancée dans la seconde qui suit l'ouverture du tunnel, elle peut échouer sans rien dire, et il suffit alors de la relancer :

```exec
curl -s -o /dev/null -d "vote=a" http://localhost:28651/
```

Refermez le tunnel. `pkill -f` arrête le programme dont la ligne de commande contient ce texte :

```exec
pkill -f "port-forward deployment/vote 28651"
```

Demandez maintenant à la base ce qu'elle a enregistré. `psql` est le client de PostgreSQL, présent dans le conteneur : `-U postgres` donne l'utilisateur, et `-c` une requête, qui compte ici les votes par option :

```exec
kubectl exec deployment/db -- psql -U postgres -c "SELECT vote, COUNT(*) FROM votes GROUP BY vote"
```

Votre vote est arrivé : `vote` l'a déposé dans `redis`, et `worker` l'a enregistré dans la nouvelle base, avec les identifiants que celle-ci tient du Secret. Si la commande répond `relation "votes" does not exist`, `worker` n'a pas encore retrouvé la base : relancez-la quelques secondes plus tard.

```text
 vote | count
------+-------
 a    |     1
(1 row)
```

En revanche, il n'y a là **que** ce vote : ceux d'avant le remplacement ont disparu. La base n'a pas de **volume persistant**, ses données vivaient dans le pod remplacé.

> 🗣️ **En réunion projet.** Voilà pourquoi une base de données dans Kubernetes demande des précautions : un volume persistant, des sauvegardes, et souvent un outil spécialisé. Beaucoup d'équipes préfèrent d'ailleurs une base **managée** par leur fournisseur cloud, hors du cluster.

```hint
Le nom attendu est celui que vous avez donné au Secret dans `db.secret.yml`, à la ligne `name:` sous `metadata:`.
```

```hint
La ligne complétée est `name: db-credentials`, décalée de quatre espaces sous `- secretRef:`. Après l'avoir enregistrée, relancez `kubectl apply -f ~/byte-layers/kubernetes/l5/db.yml`.
```

```verify
REF=$(kubectl get deployment db -o jsonpath='{.spec.template.spec.containers[*].envFrom[*].secretRef.name} {.spec.template.spec.containers[*].env[*].valueFrom.secretKeyRef.name}' 2>/dev/null)
REF=$(echo $REF)
case " $REF " in
  *" db-credentials "*) : ;;
  *) echo "Le Deployment db ne lit pas le Secret db-credentials (référence trouvée : ${REF:-aucune})."
     echo "Complétez la ligne name: de ~/byte-layers/kubernetes/l5/db.yml, puis relancez kubectl apply -f sur ce fichier."
     exit 1 ;;
esac
CLEAR=$(kubectl get deployment db -o jsonpath='{.spec.template.spec.containers[*].env[?(@.name=="POSTGRES_PASSWORD")].value}' 2>/dev/null)
if [ -n "$CLEAR" ]; then
  echo "Le mot de passe est encore écrit en clair dans le Deployment db, sous env:. Retirez ce bloc : envFrom suffit."
  exit 1
fi
if ! kubectl rollout status deployment/db --timeout=40s >/dev/null 2>&1; then
  echo "Le nouveau pod de db n'est pas prêt. Regardez son état : kubectl get pods -l app=db"
  echo "S'il est en CreateContainerConfigError, le Secret cité n'existe pas : kubectl describe pod <nom-du-pod> le dit en bas."
  exit 1
fi
WANT=$(kubectl get secret db-credentials -o 'go-template={{with .data}}{{with .POSTGRES_PASSWORD}}{{. | base64decode}}{{end}}{{end}}' 2>/dev/null)
GOT=""
for i in 1 2 3 4 5; do
  GOT=$(kubectl exec deployment/db -- printenv POSTGRES_PASSWORD 2>/dev/null)
  if [ -n "$WANT" ] && [ "$GOT" = "$WANT" ]; then break; fi
  sleep 2
done
if [ -z "$WANT" ] || [ "$GOT" != "$WANT" ]; then
  echo "Le conteneur de db n'a pas reçu POSTGRES_PASSWORD depuis le Secret db-credentials."
  echo "Vérifiez que le Secret existe (kubectl get secrets), puis relancez kubectl apply -f sur db.yml."
  exit 1
fi
TABLE=""
for i in 1 2 3 4 5 6 7 8 9 10; do
  TABLE=$(kubectl exec deployment/db -- psql -U postgres -tAc "SELECT to_regclass('public.votes')" 2>/dev/null)
  if [ "$TABLE" = "votes" ]; then break; fi
  sleep 2
done
if [ "$TABLE" != "votes" ]; then
  echo "La base lit le Secret, mais worker n'y a pas encore recréé la table des votes : il n'a pas retrouvé la base."
  echo "Relancez-le, ainsi que result : kubectl rollout restart deployment worker result, puis réessayez."
  exit 1
fi
echo "La base lit ses identifiants dans le Secret db-credentials, et worker s'y est reconnecté."
```

````solution
Le fichier complet, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l5/db.yml <<'EOF'
# db : les identifiants viennent du Secret "db-credentials" (à créer avant).
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: ghcr.io/bngams/kube-postgres:15-alpine
          envFrom:
            - secretRef:
                name: db-credentials
          ports:
            - containerPort: 5432
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
--- # objet suivant
apiVersion: v1
kind: Service
metadata:
  name: db
  labels:
    app: db
spec:
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
EOF
kubectl apply -f ~/byte-layers/kubernetes/l5/db.yml
kubectl rollout status deployment/db --timeout=50s
```
````

## Récap

- Un **Secret** range les informations sensibles à part du Deployment, dans un type d'objet qu'on peut protéger par des droits d'accès.
- `stringData` reçoit les valeurs en clair dans le fichier ; `envFrom` avec `secretRef` les donne au conteneur comme variables d'environnement.
- Remplacer le pod d'une base sans volume persistant efface ses données : une base dans Kubernetes demande des précautions.

L'application de vote reste en place dans `vote-app`, et sa base lit ses identifiants dans le Secret `db-credentials` : la suite du parcours repart de cet état. Vos fichiers restent dans `~/byte-layers/kubernetes/l5/`.

Ce Secret protège-t-il vraiment le mot de passe ? C'est la question que pose [Un Secret est encodé, pas chiffré](https://byte-layers.com/fr/bits/k8s-secret-is-encoded), à lire maintenant que vous l'avez manipulé.

> 📖 [Secrets](https://kubernetes.io/fr/docs/concepts/configuration/secret/) · [Donner des identifiants à un pod avec un Secret](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
