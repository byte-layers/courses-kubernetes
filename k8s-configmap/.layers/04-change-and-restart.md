---
title: "Changer un réglage et redémarrer"
description: "Modifier la ConfigMap seule, constater que les pods gardent l'ancienne valeur, puis les redémarrer avec kubectl rollout restart"
---

Changez de nouveau la question, cette fois en ne modifiant **que** la ConfigMap.

## Modifier la ConfigMap sans toucher au Deployment

Ouvrez `~/byte-layers/kubernetes/l5/vote.config.yml` dans votre éditeur, remplacez les deux valeurs par de nouvelles options de votre choix, par exemple `Été` et `Hiver`, puis enregistrez. Appliquez ensuite le fichier :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l5/vote.config.yml
```

`kubectl` répond cette fois `configured` : l'objet existait, il a été modifié.

```text
configmap/vote-config configured
```

Faites une prévision avant la commande suivante : les pods de `vote` vont-ils répondre par les nouvelles options, ou par les anciennes ?

```exec
kubectl exec deployment/vote -- printenv OPTION_A OPTION_B
```

Ils répondent par les **anciennes**. Si vous rouvriez la page de vote, la question n'aurait pas changé non plus.

```text
Montagne
Mer
```

> 🧠 **Pourquoi ?** Une variable d'environnement est lue **une seule fois**, au démarrage du conteneur. Les pods qui tournent ont démarré avec les anciennes valeurs, et la ConfigMap modifiée ne les concerne pas tant qu'ils ne redémarrent pas. Rien dans le Deployment n'a changé, donc Kubernetes n'a aucune raison de les remplacer.

## Redémarrer les pods avec kubectl rollout restart

Demandez explicitement le redémarrage progressif des pods de `vote`. `rollout restart` veut dire « refais une mise à jour progressive, sans rien changer d'autre » :

```exec
kubectl rollout restart deployment vote
```

`kubectl` prend note de la demande et rend la main aussitôt :

```text
deployment.apps/vote restarted
```

Les pods sont remplacés un par un, sans coupure. Attendez la fin de la mise à jour :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Reposez alors la question aux pods, avec la même commande qu'avant le redémarrage :

```exec
kubectl exec deployment/vote -- printenv OPTION_A OPTION_B
```

Les nouveaux pods ont lu la ConfigMap à leur démarrage, et répondent par vos nouvelles options :

```text
Été
Hiver
```

## Voir la nouvelle question sur la page de vote

Rouvrez le tunnel vers la page de vote, comme à l'étape précédente :

```exec
kubectl port-forward deployment/vote 28650:80 > /dev/null 2>&1 &
```

Rechargez `http://localhost:28650` dans votre navigateur, ou le port 28650 de l'onglet **PORTS** sur un poste hébergé : la nouvelle question s'affiche. Refermez ensuite le tunnel :

```exec
pkill -f "port-forward deployment/vote 28650"
```

> 🗣️ **En réunion projet.** « *On a changé la config, il faut redémarrer les pods* » : c'est exactement ce que vous venez de faire. Un changement de configuration se déploie donc comme une petite mise en production. Un mauvais réglage peut casser l'application aussi sûrement qu'un bug, d'où l'intérêt de versionner ces fichiers dans Git.

```hint
Deux choses doivent être vraies : la ConfigMap du cluster porte de nouvelles options, différentes de celles de l'étape précédente, et les pods de `vote` ont redémarré depuis.
```

```hint
Après avoir modifié et enregistré `vote.config.yml` : `kubectl apply -f` sur ce fichier, puis `kubectl rollout restart deployment vote`, puis `kubectl rollout status deployment/vote`.
```

```verify
A=$(kubectl get configmap vote-config -o jsonpath='{.data.OPTION_A}' 2>/dev/null)
B=$(kubectl get configmap vote-config -o jsonpath='{.data.OPTION_B}' 2>/dev/null)
if [ -z "$A" ] || [ -z "$B" ]; then
  echo "La ConfigMap vote-config n'a pas ses deux options (OPTION_A : « $A », OPTION_B : « $B »). Reprenez l'étape « Écrire la ConfigMap »."
  exit 1
fi
OLD=$(cat ~/byte-layers/kubernetes/l5/.vote-options 2>/dev/null)
if [ -n "$OLD" ] && [ "$OLD" = "$A|$B" ]; then
  echo "La ConfigMap du cluster porte encore les options de l'étape précédente ($A et $B)."
  echo "Changez-les dans ~/byte-layers/kubernetes/l5/vote.config.yml, enregistrez, puis relancez kubectl apply -f sur ce fichier."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "La mise à jour de vote n'est pas terminée. Regardez l'état des pods : kubectl get pods -l app=vote"
  exit 1
fi
GA=""; GB=""
for i in 1 2 3 4 5; do
  GA=$(kubectl exec deployment/vote -- printenv OPTION_A 2>/dev/null)
  GB=$(kubectl exec deployment/vote -- printenv OPTION_B 2>/dev/null)
  if [ "$GA" = "$A" ] && [ "$GB" = "$B" ]; then break; fi
  sleep 2
done
if [ "$GA" != "$A" ] || [ "$GB" != "$B" ]; then
  echo "La ConfigMap dit « $A » et « $B », mais les pods de vote tournent encore avec « $GA » et « $GB »."
  echo "Une variable d'environnement n'est lue qu'au démarrage : kubectl rollout restart deployment vote"
  exit 1
fi
echo "Les pods de vote ont redémarré et lisent les nouvelles options : $A ou $B."
```

````solution
Le fichier modifié, écrit par une commande, puis l'application et le redémarrage :

```bash
cat > ~/byte-layers/kubernetes/l5/vote.config.yml <<'EOF'
# Les réglages de vote, rangés à part : on les change sans toucher à l'image ni au Deployment
apiVersion: v1
kind: ConfigMap
metadata:
  name: vote-config
data:
  OPTION_A: Été
  OPTION_B: Hiver
EOF
kubectl apply -f ~/byte-layers/kubernetes/l5/vote.config.yml
kubectl rollout restart deployment vote
kubectl rollout status deployment/vote --timeout=50s
```
````

## Questions sur la ConfigMap

```quiz
title: "Sortir les réglages de l'image"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Pourquoi ranger les réglages dans une ConfigMap plutôt que dans l'image ?"
    answers:
      - "Pour que l'image démarre plus vite"
      - "Pour garder une seule image, testée une fois, avec des réglages différents par environnement"
      - "Parce qu'une image ne peut pas contenir de variables d'environnement"
    correct: 2
    explanation: "Avec les réglages dans l'image, il faudrait une image par environnement. La ConfigMap les sépare de l'image et du Deployment."
  - prompt: "Vous modifiez une ConfigMap lue par `envFrom`, puis vous lancez `kubectl apply`. Que font les pods qui tournent ?"
    answers:
      - "Ils relisent la ConfigMap tout de suite"
      - "Ils redémarrent tout seuls"
      - "Rien : ils gardent les valeurs lues à leur démarrage"
    correct: 3
    explanation: "Une variable d'environnement est lue une seule fois, au démarrage du conteneur. Il faut `kubectl rollout restart` pour que de nouveaux pods lisent la nouvelle valeur."
  - prompt: "De nouveaux pods restent en `CreateContainerConfigError`. Quelle est la cause la plus probable ?"
    answers:
      - "L'image n'existe pas dans le registre"
      - "Le Deployment cite une ConfigMap introuvable, par exemple un nom mal écrit"
      - "Le cluster n'a plus de mémoire"
    correct: 2
    explanation: "`kubectl describe pod` le dit en bas : `configmap \"…\" not found`. Les anciens pods continuent de servir pendant ce temps."
```

## Récap

- Une **ConfigMap** sépare les réglages de l'image : une seule image, des réglages par environnement.
- `envFrom` avec `configMapRef` importe toutes les clés d'une ConfigMap comme variables d'environnement d'un conteneur.
- Les variables d'environnement sont lues **au démarrage** : après un changement de ConfigMap, il faut `kubectl rollout restart`.

L'application de vote reste en place dans `vote-app`, avec sa ConfigMap `vote-config` : la suite du parcours repart de cet état. Vos fichiers restent dans `~/byte-layers/kubernetes/l5/`.

## Pour aller plus loin

- **Une configuration par environnement** : une ConfigMap n'est visible que dans son namespace. Avec un namespace `recette` et sa propre `vote-config`, la même image et le même `vote.deploy.yml` y afficheraient d'autres options : c'est ainsi qu'on sépare recette et production.
- **Un réglage sous forme de fichier** : une ConfigMap peut aussi contenir un fichier entier, une configuration NGINX par exemple, monté dans le conteneur comme un vrai fichier. Quand la ConfigMap change, le fichier est mis à jour dans les pods au bout d'une minute environ, sans redémarrage, à condition que l'application relise son fichier.

> 📖 [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) · [Configurer un pod avec une ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) · [`kubectl rollout restart`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_restart/)
