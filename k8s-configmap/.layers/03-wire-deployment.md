---
title: "Brancher la ConfigMap sur vote"
description: "Dire au Deployment vote de lire ses variables d'environnement dans la ConfigMap, et voir la question changer"
---

La ConfigMap existe, mais `vote` ne la lit pas encore : il faut le dire dans son Deployment.

## Ce que les pods de vote reçoivent pour l'instant

Avant de changer quoi que ce soit, demandez à un pod de `vote` la valeur de sa variable `OPTION_A`. `kubectl exec` lance une commande **dans** un conteneur, comme `docker exec` ; `deployment/vote` désigne un des pods du Deployment, et tout ce qui suit `--` est la commande à lancer, ici `printenv`, qui affiche la valeur d'une variable d'environnement.

Faites d'abord une prévision : la ConfigMap contient votre option, le pod va-t-il la connaître ?

```exec
kubectl exec deployment/vote -- printenv OPTION_A
```

Le pod ne connaît pas cette variable : `printenv` n'affiche rien et se termine en erreur, ce que `kubectl` rapporte ainsi.

```text
command terminated with exit code 1
```

Créer une ConfigMap ne change donc rien aux pods qui tournent. Tant qu'aucun Deployment ne la cite, personne ne la lit.

## Ajouter envFrom dans vote.deploy.yml

Voici le Deployment de `vote`, celui de l'étape de départ, avec trois lignes de plus entre `ports` et `resources`. La commande l'écrit dans votre dossier de travail, avec un blanc à compléter :

```exec
cat > ~/byte-layers/kubernetes/l5/vote.deploy.yml <<'EOF'
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
          image: ghcr.io/bngams/kube-vote:1.0
          ports:
            - containerPort: 80
          envFrom:              # "prends des variables d'environnement…"
            - configMapRef:     # "… dans une ConfigMap…"
                name: # TODO : "… qui s'appelle" (le nom de votre ConfigMap)
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF
```

Ouvrez `~/byte-layers/kubernetes/l5/vote.deploy.yml` dans votre éditeur, remplacez le `# TODO` et tout le texte qui le suit sur la ligne, puis enregistrez. Les trois lignes se lisent comme une phrase :

| Ligne | Rôle |
|---|---|
| `envFrom:` | « prends des variables d'environnement… » Elle est au même décalage que `ports:` et `resources:`, car elle décrit le même conteneur |
| `- configMapRef:` | « … dans une ConfigMap… » Le tiret ouvre une liste : un conteneur peut lire plusieurs sources |
| `name:` | « … qui s'appelle ainsi ». C'est le `metadata.name` de la ConfigMap |

> 💡 **`env` ou `envFrom` ?** Dans le fichier de la base, à l'étape de départ, chaque variable est listée une par une sous `env:`. `envFrom:` importe d'un coup **toutes** les clés d'une ConfigMap. Et l'ordre des champs (`ports`, `envFrom`, `resources`…) n'a aucune importance : seule l'indentation compte.

## Appliquer et suivre la mise à jour progressive

Appliquez la nouvelle version du Deployment :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l5/vote.deploy.yml
```

Modifier le modèle de pod déclenche une **mise à jour progressive** (*rollout*, en anglais), comme quand vous avez ajouté des réservations dans [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending) : les pods sont remplacés un par un. Suivez-la jusqu'au bout avec `kubectl rollout status`, qui rend la main quand les trois nouveaux pods sont prêts :

```exec
kubectl rollout status deployment/vote --timeout=120s
```

La dernière ligne annonce la fin de la mise à jour :

```text
deployment "vote" successfully rolled out
```

Regardez les pods de `vote`. `-l app=vote` ne garde que ceux qui portent cette étiquette :

```exec
kubectl get pods -l app=vote
```

Trois nouveaux pods tournent, avec un nouveau début de nom et un âge de quelques secondes. Un ancien pod peut encore apparaître un instant, en `Terminating` :

```text
NAME                   READY   STATUS    RESTARTS   AGE
vote-fb9787cdf-b9lr4   1/1     Running   0          17s
vote-fb9787cdf-gm8xn   1/1     Running   0          2s
vote-fb9787cdf-wccwd   1/1     Running   0          1s
```

Reposez maintenant la question du début à un de ces pods :

```exec
kubectl exec deployment/vote -- printenv OPTION_A OPTION_B
```

Cette fois, il répond par vos deux options, par exemple :

```text
Montagne
Mer
```

> ⚠️ **Symptôme :** les nouveaux pods restent en `CreateContainerConfigError`, et `kubectl rollout status` ne se termine pas.
> **Cause :** le Deployment cherche une ConfigMap qui n'existe pas, par exemple parce que son nom est mal écrit, ou parce que la ConfigMap n'a pas été appliquée. `kubectl describe pod <nom-du-pod>` l'indique en bas : `Error: configmap "vote-config" not found`.
> **Correctif :** vérifiez le nom dans les deux fichiers, puis appliquez de nouveau celui que vous avez corrigé. Les pods démarrent tout seuls quelques secondes plus tard. Pendant ce temps, les anciens pods continuent de servir la page.

## Voir la nouvelle question sur la page de vote

Il reste à voir la page elle-même. Ouvrez un tunnel entre le port **28650** de votre machine et le port 80 d'un pod de `vote` :

```exec
kubectl port-forward deployment/vote 28650:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward deployment/vote` | ouvre un tunnel temporaire vers un pod du Deployment `vote` |
> | `28650:80` | le port 28650 de votre machine mène au port 80 du pod |
> | `> /dev/null` | jette ce que la commande afficherait, pour ne pas encombrer le terminal |
> | `2>&1` | jette aussi ses messages d'erreur, au même endroit |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main, et le tunnel reste ouvert |

Ouvrez `http://localhost:28650` dans votre navigateur. Sur un poste hébergé, passez par l'onglet **PORTS** de VS Code et ouvrez le port 28650. La question est devenue « **Montagne ou Mer ?** », ou les options que vous avez choisies. L'image n'a pas changé, seule la configuration a changé.

Refermez le tunnel avant de continuer. `pkill -f` arrête le programme dont la ligne de commande contient ce texte :

```exec
pkill -f "port-forward deployment/vote 28650"
```

```hint
Le nom attendu est celui que vous avez donné à la ConfigMap dans `vote.config.yml`, à la ligne `name:` sous `metadata:`.
```

```hint
La ligne complétée est `name: vote-config`, décalée de quatre espaces sous `- configMapRef:`. Après l'avoir enregistrée, relancez `kubectl apply -f ~/byte-layers/kubernetes/l5/vote.deploy.yml`.
```

```verify
REF=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[*].envFrom[*].configMapRef.name} {.spec.template.spec.containers[*].env[*].valueFrom.configMapKeyRef.name}' 2>/dev/null)
REF=$(echo $REF)
case " $REF " in
  *" vote-config "*) : ;;
  *) echo "Le Deployment vote ne lit pas la ConfigMap vote-config (référence trouvée : ${REF:-aucune})."
     echo "Complétez la ligne name: de ~/byte-layers/kubernetes/l5/vote.deploy.yml, puis relancez kubectl apply -f sur ce fichier."
     exit 1 ;;
esac
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "La mise à jour de vote n'est pas terminée. Regardez l'état des pods : kubectl get pods -l app=vote"
  echo "S'ils sont en CreateContainerConfigError, la ConfigMap citée n'existe pas : kubectl describe pod <nom-du-pod> le dit en bas."
  exit 1
fi
A=$(kubectl get configmap vote-config -o jsonpath='{.data.OPTION_A}' 2>/dev/null)
B=$(kubectl get configmap vote-config -o jsonpath='{.data.OPTION_B}' 2>/dev/null)
GOT=""
for i in 1 2 3 4 5; do
  GOT=$(kubectl exec deployment/vote -- printenv OPTION_A 2>/dev/null)
  if [ -n "$A" ] && [ "$GOT" = "$A" ]; then break; fi
  sleep 2
done
if [ -z "$A" ] || [ "$GOT" != "$A" ]; then
  echo "Les pods de vote n'ont pas reçu OPTION_A depuis la ConfigMap (ConfigMap : « $A », pod : « $GOT »)."
  echo "Vérifiez que la ConfigMap vote-config contient OPTION_A et OPTION_B, puis relancez kubectl apply -f sur vote.deploy.yml."
  exit 1
fi
mkdir -p ~/byte-layers/kubernetes/l5
printf '%s|%s\n' "$A" "$B" > ~/byte-layers/kubernetes/l5/.vote-options
echo "Les pods de vote lisent la ConfigMap vote-config : $A ou $B."
```

````solution
Le fichier complet, écrit par une commande, puis appliqué :

```bash
cat > ~/byte-layers/kubernetes/l5/vote.deploy.yml <<'EOF'
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
          image: ghcr.io/bngams/kube-vote:1.0
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
kubectl apply -f ~/byte-layers/kubernetes/l5/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

> 📖 [Configurer un pod avec une ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
