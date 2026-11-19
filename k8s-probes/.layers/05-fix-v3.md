---
title: "Corriger la v3 pour de bon"
description: "Ajouter le réglage manquant à la ConfigMap, livrer la v3 avec ses sondes, et voir le titre dans la page"
---

Le service est rétabli, il reste à livrer la v3 **correctement**. Elle a besoin d'un réglage `VOTE_TITLE`, et un réglage se range dans la ConfigMap, comme le montre [Sortir les réglages de l'image avec une ConfigMap](https://byte-layers.com/fr/skills/k8s-configmap).

## Ajouter le réglage manquant à la ConfigMap

Ouvrez `~/byte-layers/kubernetes/l6/vote.config.yml` dans votre éditeur. Gardez les deux options telles quelles et ajoutez une ligne `VOTE_TITLE` en dessous, au même niveau qu'elles, avec le titre de votre choix entre guillemets, parce qu'un titre peut contenir un `?` ou un `:`. Voici le bloc `data` attendu, avec la ligne à ajouter et à compléter :

```yaml
data:
  OPTION_A: Montagne
  OPTION_B: Mer
  VOTE_TITLE: # TODO : un titre entre guillemets, par exemple "Votre destination de vacances ?"
```

Enregistrez, puis appliquez la ConfigMap **d'abord** : les pods de la v3 lisent leurs réglages à leur démarrage, le réglage doit donc exister avant eux.

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.config.yml
```

## Livrer la v3 avec son réglage

Dans `~/byte-layers/kubernetes/l6/vote.deploy.yml`, passez de nouveau l'image de `2.0` à `3.0`, enregistrez, puis appliquez :

```exec
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
```

Suivez la mise à jour. Faites une prévision : va-t-elle aller au bout, cette fois ?

```exec
kubectl rollout status deployment/vote --timeout=120s
```

Les trois pods sont remplacés un par un, chacun n'étant compté comme prêt qu'après une bonne réponse à sa sonde de readiness, et la commande se termine par :

```text
deployment "vote" successfully rolled out
```

La v3 était donc saine : il lui manquait seulement sa configuration. Le fichier et le cluster sont cohérents, tous deux en `3.0`.

```hint
Dans `vote.config.yml`, la nouvelle ligne s'écrit sous `OPTION_B`, avec deux espaces devant, comme elle : `VOTE_TITLE: "Votre destination de vacances ?"`.
```

```hint
L'ordre compte : `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.config.yml`, puis l'image en `3.0` dans `vote.deploy.yml`, puis `kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml`.
```

```verify
T=$(kubectl get configmap vote-config -o jsonpath='{.data.VOTE_TITLE}' 2>/dev/null)
if [ -z "$T" ]; then
  echo "La ConfigMap vote-config n'a pas de réglage VOTE_TITLE. Ajoutez la ligne dans vote.config.yml, enregistrez, puis lancez kubectl apply -f ~/byte-layers/kubernetes/l6/vote.config.yml"
  exit 1
fi
IMG=$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
case "$IMG" in
  *kube-vote:3.0) : ;;
  *) echo "Le réglage est là, mais le cluster demande l'image ${IMG:-(aucune)}, pas la 3.0. Passez la version à 3.0 dans vote.deploy.yml, enregistrez, puis appliquez le fichier."
     exit 1 ;;
esac
if [ "$(kubectl get deployment vote -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.path}' 2>/dev/null)" != "/healthz" ]; then
  echo "Le Deployment vote n'a plus sa sonde de readiness sur /healthz : la v3 doit être livrée avec ses sondes. Reprenez le fichier de l'étape « Ajouter une sonde de readiness et une sonde de liveness »."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=45s >/dev/null 2>&1; then
  echo "La v3 est demandée, mais ses pods ne sont pas tous prêts. Si certains plantaient avant l'ajout du réglage, ils le liront à leur prochain redémarrage : attendez une minute, puis réessayez. Sinon, regardez-les avec kubectl get pods -l app=vote."
  exit 1
fi
echo "vote tourne en version 3.0 sur ses 3 pods, avec le titre « $T » lu dans la ConfigMap."
```

````solution
Les deux fichiers complets, écrits par des commandes, puis appliqués dans l'ordre : la ConfigMap d'abord.

```bash
cat > ~/byte-layers/kubernetes/l6/vote.config.yml <<'EOF'
# Réglages de vote : VOTE_TITLE est exigé par la version 3.0
apiVersion: v1
kind: ConfigMap
metadata:
  name: vote-config
data:
  OPTION_A: Montagne
  OPTION_B: Mer
  VOTE_TITLE: "Votre destination de vacances ?"
EOF
cat > ~/byte-layers/kubernetes/l6/vote.deploy.yml <<'EOF'
# vote : version 3.0, réglages dans vote-config, sondes de santé sur /healthz
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
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.config.yml
kubectl apply -f ~/byte-layers/kubernetes/l6/vote.deploy.yml
kubectl rollout status deployment/vote --timeout=50s
```
````

## Voir le titre dans la page de vote

Il reste à voir le résultat. Ouvrez un tunnel entre le port **28661** de votre machine et le Service `vote` :

```exec
kubectl port-forward service/vote 28661:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward service/vote 28661:80` | ouvre un tunnel temporaire : ce qui arrive sur le port 28661 de votre machine est envoyé au port 80 du Service `vote` |
> | `> /dev/null` | jette ce que la commande affiche, pour ne pas encombrer le terminal |
> | `2>&1` | jette aussi ses messages d'erreur, au même endroit |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main tout de suite, et le tunnel reste ouvert |

Ouvrez `http://localhost:28661` dans votre navigateur. Sur un poste hébergé, passez par l'onglet **PORTS** de VS Code, qui donne l'adresse à ouvrir pour le port 28661. Le badge en haut à droite indique **v3.0**, et votre titre s'affiche en haut de la page.

Fermez ensuite le tunnel. `pkill -f` arrête le programme dont la ligne de commande contient le texte entre guillemets :

```exec
pkill -f "port-forward service/vote 28661"
```

> 🗣️ **En réunion projet.** Ce scénario est un grand classique : une nouvelle version exige un nouveau réglage, et le réglage n'a pas été livré avec. D'où l'intérêt de livrer **ensemble** le code et sa configuration, et de décrire les réglages requis dans les notes de version.

## Récap

- Sans sonde, Kubernetes ne sait qu'une chose d'un pod : « le programme a démarré ». Une version qui plante juste après peut donc remplacer des pods sains.
- La sonde de **readiness** demande « peux-tu recevoir des visiteurs ? » : tant que la réponse est non, le pod ne reçoit pas de trafic, et une mise à jour attend.
- La sonde de **liveness** demande « es-tu encore en vie ? » : après plusieurs échecs de suite, Kubernetes redémarre le conteneur.
- Une nouvelle version et le réglage qu'elle exige se livrent **ensemble**, le réglage d'abord.

Les fichiers finaux de ce cours se trouvent dans [`solution/vote.deploy.yml`](../solution/vote.deploy.yml) et [`solution/vote.config.yml`](../solution/vote.config.yml). L'application reste en place, en version 3.0.

## Pour aller plus loin

- Une troisième sonde, `startupProbe`, laisse à une application lente le temps de démarrer avant que les deux autres ne commencent : [les sondes de liveness, readiness et startup](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/).
- Une sonde peut aussi lancer une commande dans le conteneur ou ouvrir une connexion, au lieu d'une requête HTTP : [configurer les sondes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

> 📖 [Les sondes de liveness, readiness et startup](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) · [Configurer les sondes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) · [Le cycle de vie d'un pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

## Questions sur les sondes

```quiz
title: "Les sondes de santé"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Sans sonde de readiness, à partir de quand Kubernetes considère-t-il un pod comme prêt ?"
    answers:
      - "Dès que son conteneur a démarré"
      - "Quand l'application a répondu à une première requête"
      - "Au bout d'une minute sans erreur"
    correct: 1
    explanation: "Sans autre information, « le programme a démarré » suffit. C'est pour cela qu'une version qui plante juste après son démarrage a pu remplacer un pod sain."
  - prompt: "La sonde de readiness d'un pod échoue. Que fait Kubernetes ?"
    answers:
      - "Il supprime le pod"
      - "Il le retire du Service, et pendant une mise à jour il ne le compte pas comme prêt"
      - "Il redémarre le conteneur"
    correct: 2
    explanation: "La readiness décide si le pod reçoit des visiteurs. Redémarrer le conteneur, c'est le rôle de la sonde de liveness."
  - prompt: "Avec les sondes, la v3 cassée a été déployée une seconde fois. Combien de pods de la v2 ont été retirés ?"
    answers:
      - "Un"
      - "Deux"
      - "Aucun"
    correct: 3
    explanation: "Le pod de la v3 n'a jamais répondu à la sonde de readiness : il n'a jamais été compté comme prêt, et la mise à jour s'est arrêtée avant de toucher aux pods sains."
```
