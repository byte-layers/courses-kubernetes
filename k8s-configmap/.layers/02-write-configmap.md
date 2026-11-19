---
title: "Écrire la ConfigMap"
description: "Ranger les deux options du vote dans un objet à part, la ConfigMap, et l'appliquer"
---

Dans [Échauffement : la voting app en conteneurs](https://byte-layers.com/fr/labs/voting-app-containers-warmup), vous avez changé la question du vote avec `docker run -e OPTION_A=Thé -e OPTION_B=Café`. Ces **variables d'environnement** sont des réglages donnés à l'application au démarrage. Dans Kubernetes, on les range dans un objet dédié, la **ConfigMap** : une liste de paires `clé: valeur`, séparée de l'image **et** du Deployment.

> 🧠 **Pourquoi séparer ?** La même image doit pouvoir tourner en test, en recette et en production, avec des réglages différents. Si les réglages étaient dans l'image, il faudrait une image par environnement. Avec une ConfigMap par environnement, on garde **une seule image**, testée une fois.

## Écrire le fichier vote.config.yml, avec ses deux options

Créez le fichier de la ConfigMap, avec ses deux blancs. La commande l'écrit dans votre dossier de travail, comme les fichiers de l'étape précédente :

```exec
cat > ~/byte-layers/kubernetes/l5/vote.config.yml <<'EOF'
# Les réglages de vote, rangés à part : on les change sans toucher à l'image ni au Deployment
apiVersion: v1
kind: ConfigMap
metadata:
  name: vote-config
data:
  OPTION_A: # TODO : la première option du vote, par exemple Montagne
  OPTION_B: # TODO : la seconde option, par exemple Mer
EOF
```

Ouvrez `~/byte-layers/kubernetes/l5/vote.config.yml` dans votre éditeur, puis remplacez chaque `# TODO` **et tout le texte qui le suit sur la ligne** par votre valeur. Enregistrez. Ce tableau dit ce que porte chaque champ :

| Champ | Rôle |
|---|---|
| `kind: ConfigMap` | on décrit un ensemble de réglages |
| `metadata.name` | son nom, `vote-config`, que le Deployment utilisera pour la retrouver |
| `data` | les réglages, sous forme `clé: valeur`. Les clés `OPTION_A` et `OPTION_B` sont celles que l'application sait lire. Si une valeur est un nombre ou un mot comme `Yes`, `No`, `true` ou `false`, mettez-la entre guillemets (`"Yes"`, `"12"`) : sinon YAML la prend pour autre chose qu'un texte, et l'`apply` échoue |

## Appliquer la ConfigMap et la retrouver dans le cluster

Transmettez le fichier au cluster, comme pour un Deployment :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l5/vote.config.yml
```

`kubectl` confirme la création de l'objet dans le cluster :

```text
configmap/vote-config created
```

Listez maintenant les ConfigMaps du namespace :

```exec
kubectl get configmaps
```

La colonne `DATA` compte 2 réglages. `kube-root-ca.crt` est une ConfigMap créée automatiquement par Kubernetes dans chaque namespace : ignorez-la.

```text
NAME               DATA   AGE
kube-root-ca.crt   1      3h
vote-config        2      0s
```

```hint
Une ligne complétée ressemble à `OPTION_A: Montagne` : la clé, deux-points, une espace, puis la valeur. Il ne doit plus rester de `#` sur la ligne.
```

```hint
Si `kubectl apply` répond par une erreur, relisez l'indentation : `OPTION_A` et `OPTION_B` sont décalées de deux espaces sous `data:`, et une valeur comme `Yes` ou `12` se met entre guillemets.
```

```verify
if ! kubectl get configmap vote-config >/dev/null 2>&1; then
  echo "Il n'y a pas de ConfigMap vote-config dans le namespace courant."
  echo "Complétez ~/byte-layers/kubernetes/l5/vote.config.yml, puis lancez : kubectl apply -f ~/byte-layers/kubernetes/l5/vote.config.yml"
  exit 1
fi
A=$(kubectl get configmap vote-config -o jsonpath='{.data.OPTION_A}')
B=$(kubectl get configmap vote-config -o jsonpath='{.data.OPTION_B}')
if [ -z "$A" ] || [ -z "$B" ]; then
  echo "La ConfigMap vote-config existe, mais il lui manque une valeur (OPTION_A : « $A », OPTION_B : « $B »)."
  echo "Remplacez les deux # TODO du fichier par vos options, enregistrez, puis relancez kubectl apply."
  exit 1
fi
case "$A$B" in
  *TODO*) echo "Une option contient encore le mot TODO : remplacez toute la fin de la ligne par votre valeur, puis relancez kubectl apply."; exit 1 ;;
esac
echo "La ConfigMap vote-config porte vos deux options : $A et $B."
```

````solution
Le fichier complet, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l5/vote.config.yml <<'EOF'
# Les réglages de vote, rangés à part : on les change sans toucher à l'image ni au Deployment
apiVersion: v1
kind: ConfigMap
metadata:
  name: vote-config
data:
  OPTION_A: Montagne
  OPTION_B: Mer
EOF
kubectl apply -f ~/byte-layers/kubernetes/l5/vote.config.yml
```
````

> 📖 [ConfigMaps, documentation officielle](https://kubernetes.io/docs/concepts/configuration/configmap/)
