---
title: "Écrire le Secret"
description: "Ranger le nom d'utilisateur et le mot de passe de la base dans un Secret, et l'appliquer"
---

Les informations sensibles (mots de passe, clés d'accès, certificats) se rangent dans un objet dédié, le **Secret** : une liste de paires `clé: valeur`, comme les réglages d'une application, mais dans un type d'objet à part. On peut ainsi autoriser une personne à lire les réglages d'une application **sans** lui donner accès aux Secrets, et les administrateurs peuvent faire chiffrer les Secrets dans la base où Kubernetes range tout son état.

## Où est le mot de passe aujourd'hui

Regardez d'abord le problème. Demandez au cluster les variables d'environnement que le Deployment `db` donne à son conteneur. `-o jsonpath` n'affiche qu'un champ de la fiche de l'objet, ici la liste `env` du premier conteneur du modèle de pod :

```exec
kubectl get deployment db -o jsonpath='{.spec.template.spec.containers[0].env}'
```

Le mot de passe est écrit en toutes lettres, dans le cluster comme dans le fichier `base/db.yml` de l'étape de départ. Toute personne qui ouvre ce fichier, ou le dépôt Git où il serait rangé, le lit :

```text
[{"name":"POSTGRES_USER","value":"postgres"},{"name":"POSTGRES_PASSWORD","value":"postgres"}]
```

## Écrire le fichier db.secret.yml, avec ses deux identifiants

Créez le fichier du Secret, avec ses deux blancs. La commande l'écrit dans votre dossier de travail, comme les fichiers de l'étape précédente :

```exec
cat > ~/byte-layers/kubernetes/l5/db.secret.yml <<'EOF'
# Les identifiants de la base, rangés dans un Secret plutôt qu'en clair dans le Deployment
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  POSTGRES_USER: # TODO : le nom d'utilisateur de la base, postgres
  POSTGRES_PASSWORD: # TODO : son mot de passe, postgres
EOF
```

Ouvrez `~/byte-layers/kubernetes/l5/db.secret.yml` dans votre éditeur, puis remplacez chaque `# TODO` **et tout le texte qui le suit sur la ligne** par la valeur indiquée. Enregistrez. Ce tableau dit ce que porte chaque champ :

| Champ | Rôle |
|---|---|
| `kind: Secret` | on décrit une information sensible |
| `metadata.name` | son nom, `db-credentials`, que le Deployment utilisera pour le retrouver |
| `type: Opaque` | un Secret « générique », fait de paires `clé: valeur` |
| `stringData` | les valeurs, écrites en clair **dans ce fichier seulement** : Kubernetes les encode à l'enregistrement |

> ⚠️ **Pourquoi garder `postgres` comme identifiants ?** Les programmes `worker` et `result` ont ces identifiants écrits en dur dans leur code. Si vous les changiez, la base les accepterait, mais eux ne pourraient plus s'y connecter. Une application bien conçue lit ses identifiants dans des variables d'environnement, précisément pour qu'on puisse les changer sans la reconstruire.

## Appliquer le Secret et le retrouver dans le cluster

Transmettez le fichier au cluster, comme pour un Deployment :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l5/db.secret.yml
```

`kubectl` confirme la création de l'objet dans le cluster :

```text
secret/db-credentials created
```

Listez maintenant les Secrets du namespace :

```exec
kubectl get secrets
```

La colonne `DATA` compte 2 valeurs, et aucune n'est affichée :

```text
NAME             TYPE     DATA   AGE
db-credentials   Opaque   2      0s
```

```hint
Une ligne complétée ressemble à `POSTGRES_USER: postgres` : la clé, deux-points, une espace, puis la valeur. Il ne doit plus rester de `#` sur la ligne.
```

```hint
Les deux valeurs sont `postgres`. Si `kubectl apply` répond par une erreur, relisez l'indentation : les deux clés sont décalées de deux espaces sous `stringData:`.
```

```verify
if ! kubectl get secret db-credentials >/dev/null 2>&1; then
  echo "Il n'y a pas de Secret db-credentials dans le namespace courant."
  echo "Complétez ~/byte-layers/kubernetes/l5/db.secret.yml, puis lancez : kubectl apply -f ~/byte-layers/kubernetes/l5/db.secret.yml"
  exit 1
fi
U=$(kubectl get secret db-credentials -o 'go-template={{with .data}}{{with .POSTGRES_USER}}{{. | base64decode}}{{end}}{{end}}' 2>/dev/null)
P=$(kubectl get secret db-credentials -o 'go-template={{with .data}}{{with .POSTGRES_PASSWORD}}{{. | base64decode}}{{end}}{{end}}' 2>/dev/null)
if [ -z "$U" ] || [ -z "$P" ]; then
  echo "Le Secret db-credentials existe, mais il lui manque POSTGRES_USER ou POSTGRES_PASSWORD."
  echo "Remplacez les deux # TODO du fichier, enregistrez, puis relancez kubectl apply."
  exit 1
fi
if [ "$U" != "postgres" ] || [ "$P" != "postgres" ]; then
  echo "Les deux identifiants doivent valoir postgres : worker et result n'en connaissent pas d'autres."
  echo "Corrigez le fichier, enregistrez, puis relancez kubectl apply."
  exit 1
fi
echo "Le Secret db-credentials porte les deux identifiants de la base."
```

````solution
Le fichier complet, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l5/db.secret.yml <<'EOF'
# Les identifiants de la base, rangés dans un Secret plutôt qu'en clair dans le Deployment
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
EOF
kubectl apply -f ~/byte-layers/kubernetes/l5/db.secret.yml
```
````

> 📖 [Secrets, documentation officielle](https://kubernetes.io/fr/docs/concepts/configuration/secret/)
