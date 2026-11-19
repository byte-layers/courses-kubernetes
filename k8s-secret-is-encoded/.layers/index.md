---
title: "Un Secret est encodé, pas chiffré"
description: "Ce qu'un Secret Kubernetes protège, et ce qu'il ne protège pas : le base64 se décode, kubectl apply garde le mot de passe en clair, et un pod le lit dans ses variables."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 10
requires: "0.4.0"
---

Vous avez rangé le mot de passe de la base dans un Secret, et `kubectl get secrets` ne l'affiche plus. Est-il protégé pour autant ? Vous allez le retrouver de trois façons, en trois commandes, et vous saurez d'où vient réellement la protection d'un Secret.

**Prérequis** : avoir fait [Ranger un mot de passe dans un Secret](https://byte-layers.com/fr/skills/k8s-secret). Les commandes lisent le Secret `db-credentials` et la base `db` que ce cours laisse dans le namespace `vote-app`. Rien n'est créé ni modifié ici.

# Un Secret est encodé, pas chiffré

Un **Secret** range une information sensible à part du Deployment. Il ressemble à un coffre, mais Kubernetes ne **chiffre** pas ce qu'on y met : il l'**encode**, c'est-à-dire qu'il l'écrit autrement, sans clé, de façon que n'importe qui puisse le relire.

## Deux surprises dans le YAML du Secret

Regardez le contenu complet du Secret tel que le cluster le garde. `-o yaml` affiche toute la fiche de l'objet au format YAML :

```exec
kubectl get secret db-credentials -o yaml
```

Le début de la réponse ressemble à ceci, la longue ligne étant raccourcie ici :

```text
apiVersion: v1
data:
  POSTGRES_PASSWORD: cG9zdGdyZXM=
  POSTGRES_USER: cG9zdGdyZXM=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret",…,"stringData":{"POSTGRES_PASSWORD":"postgres","POSTGRES_USER":"postgres"},…}
```

Il y a deux surprises dans ces quelques lignes :

- Dans `data`, le mot de passe est **encodé** en *base64*, une façon d'écrire n'importe quelle donnée avec des lettres et des chiffres. C'est une simple transcription, **pas un chiffrement** : n'importe qui peut la décoder.
- Juste en dessous, l'annotation `last-applied-configuration`, ajoutée par `kubectl apply` pour se souvenir du dernier fichier appliqué, contient le mot de passe… **en clair** !

## Décoder le base64 en une commande

Décodez tout de même le base64, pour vous en convaincre. Cette commande demande à `kubectl` de n'afficher qu'une valeur du Secret, et de la décoder au passage :

```exec
kubectl get secret db-credentials -o "go-template={{.data.POSTGRES_PASSWORD | base64decode}}"
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `-o "go-template=…"` | met en forme la réponse selon un modèle, au lieu du YAML complet |
> | `{{.data.POSTGRES_PASSWORD}}` | « prends la valeur `POSTGRES_PASSWORD` de la section `data` » |
> | `\| base64decode` | « et décode-la » : le `\|` passe la valeur à la fonction qui suit |

Le terminal affiche `postgres`, collé à l'invite de commande suivante, ou suivi d'un `%` sous macOS, car la réponse ne se termine pas par un retour à la ligne. Aucune clé, aucun mot de passe n'a été demandé pour décoder.

## Le lire depuis le pod, dans ses variables d'environnement

Enfin, toute personne qui peut entrer dans le pod de la base le lit aussi, directement dans ses variables d'environnement. `kubectl exec` lance une commande dans un conteneur, et `printenv` affiche la valeur d'une variable :

```exec
kubectl exec deployment/db -- printenv POSTGRES_PASSWORD
```

Le conteneur répond par le mot de passe, en clair, puisque c'est sous cette forme que la base en a besoin :

```text
postgres
```

## D'où vient alors la protection

> 🛡️ **À retenir.** Un Secret n'est pas un coffre-fort : sa protection vient des **droits d'accès** (qui a le droit de lire les Secrets, ou d'entrer dans les pods ?) et du **chiffrement** de la base où Kubernetes range son état, que les administrateurs doivent activer : il ne l'est pas par défaut. Et surtout, un fichier comme `db.secret.yml` ne doit **jamais** être déposé tel quel dans Git. En entreprise, on utilise des outils dédiés : un coffre comme HashiCorp Vault ou celui du fournisseur cloud, ou des outils qui chiffrent les Secrets avant de les ranger dans Git, comme Sealed Secrets ou SOPS.

La même règle vaut pour une image de conteneur : un mot de passe n'a pas sa place dans ce qu'on distribue, ce que montre [Un secret ne va pas dans l'image](https://byte-layers.com/fr/bits/secret-not-in-image).

> 📖 [Secrets](https://kubernetes.io/fr/docs/concepts/configuration/secret/) · [Bonnes pratiques pour les Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/) · [Chiffrer les données au repos](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

## Récap

- Dans un Secret, les valeurs sont **encodées** en base64, pas chiffrées : une commande suffit à les décoder.
- `kubectl apply` garde le dernier fichier appliqué dans une annotation : avec `stringData`, le mot de passe y reste en clair.
- Qui peut entrer dans un pod lit le Secret dans ses variables d'environnement.
- La protection vient des droits d'accès et du chiffrement activé par les administrateurs, et un fichier de Secret ne va jamais en clair dans Git.

```quiz
prompt: "Un collègue a le droit de lire les Secrets du namespace. Que lui faut-il de plus pour connaître le mot de passe rangé dans `db-credentials` ?"
answers:
  - "La clé de chiffrement du cluster"
  - "Rien : le base64 se décode sans clé"
  - "Le fichier `db.secret.yml` d'origine"
correct: 2
explanation: |
  Le base64 est une transcription, pas un chiffrement. C'est pour cela que la protection d'un Secret repose sur les droits d'accès : qui peut lire les Secrets, et qui peut entrer dans les pods.
```

```ack
J'ai compris
```
