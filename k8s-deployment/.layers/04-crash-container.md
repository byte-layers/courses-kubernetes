---
title: "Faire planter le conteneur"
description: "Ouvrir la page /crash de l'application, puis lire le nom du pod et la colonne RESTARTS"
---

Supprimer un pod n'est pas la seule panne possible. Très souvent, c'est l'**application elle-même** qui plante : un bug, un manque de mémoire… La version de `vote` utilisée ici contient une page spéciale, `/crash`, qui simule ce plantage en arrêtant le programme du conteneur.

## Provoquer le plantage avec la page /crash

Rouvrez le tunnel vers l'application :

```exec
kubectl port-forward deployment/vote 28610:80 > /dev/null 2>&1 &
```

Regardez le pod avant la panne, pour retenir son nom et sa colonne `RESTARTS`, qui compte les redémarrages de son conteneur :

```exec
kubectl get pods
```

Le compteur est à zéro. S'il ne l'est pas chez vous, par exemple parce que le cluster a été arrêté puis relancé depuis, retenez sa valeur : c'est son augmentation qui compte.

```text
NAME                    READY   STATUS    RESTARTS   AGE
vote-644d47956b-qm48q   1/1     Running   0          2m
```

Faites une prévision : après le plantage, le pod aura-t-il changé de nom, comme à l'étape précédente ?

À vous de provoquer la panne : ouvrez dans votre navigateur l'adresse de l'application suivie de **`/crash`**, soit `http://localhost:28610/crash`. Sur un poste hébergé, ajoutez `/crash` à la fin de l'adresse que donne l'onglet **PORTS** pour le port 28610. La page affiche un message avec le nom de votre pod :

```text
💥 Le conteneur de vote-644d47956b-qm48q va s'arrêter... Kubernetes va le redémarrer.
```

Laissez passer une dizaine de secondes, puis regardez de nouveau le pod :

```exec
kubectl get pods
```

Le pod a gardé **le même nom**, mais la colonne `RESTARTS` est passée à **1**, avec l'âge de ce redémarrage :

```text
NAME                    READY   STATUS    RESTARTS     AGE
vote-644d47956b-qm48q   1/1     Running   1 (6s ago)   2m
```

Si la colonne `STATUS` affiche encore `Completed`, le conteneur vient de s'arrêter et n'est pas encore reparti : relancez la commande quelques secondes plus tard. Si `RESTARTS` vaut 2, votre navigateur a demandé la page deux fois : ce n'est pas grave, attendez une vingtaine de secondes que le pod redevienne `Running`.

```hint
La page `/crash` s'ouvre comme la page de vote, par le tunnel : il doit être ouvert, ce que fait la première commande de cette section.
```

```hint
Sans navigateur, `curl` ouvre la même adresse depuis le terminal : `curl -s http://localhost:28610/crash`.
```

```verify
if ! kubectl version --request-timeout=10s >/dev/null 2>&1; then
  echo "kubectl ne joint pas le cluster. Vérifiez qu'il est démarré avec kubectl get nodes, puis réessayez."
  exit 1
fi
if ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "Il n'y a pas de Deployment vote. Relancez kubectl apply -f ~/byte-layers/kubernetes/l1/vote.deploy.yml, puis reprenez cette étape."
  exit 1
fi
RESTARTS=$(kubectl get pods -l app=vote -o jsonpath='{range .items[*]}{.status.containerStatuses[0].restartCount}{"\n"}{end}' | sort -n | tail -1)
if [ -z "$RESTARTS" ] || [ "$RESTARTS" -lt 1 ]; then
  echo "Le conteneur du pod de vote n'a jamais redémarré (RESTARTS vaut ${RESTARTS:-0}). Ouvrez http://localhost:28610/crash, tunnel ouvert, puis réessayez."
  exit 1
fi
if ! kubectl rollout status deployment/vote --timeout=40s >/dev/null 2>&1; then
  echo "Le conteneur a bien planté, mais il n'est pas encore reparti. Attendez quelques secondes, regardez kubectl get pods, puis réessayez."
  exit 1
fi
echo "Le conteneur a planté et Kubernetes l'a redémarré dans le même pod : RESTARTS vaut $RESTARTS."
```

````solution
La même visite, faite depuis le terminal avec `curl`, suivie d'une attente :

```bash
curl -s http://localhost:28610/crash
sleep 8
kubectl rollout status deployment/vote --timeout=40s
```
````

## Le même pod, un conteneur redémarré

Kubernetes a **redémarré le conteneur** à l'intérieur du pod existant : il n'a pas créé de nouveau pod. Ce redémarrage ne doit rien au Deployment : c'est l'agent de Kubernetes sur le nœud, le **kubelet**, qui relance le conteneur d'un pod quand il s'arrête. Demandez la page d'accueil, sans relancer le tunnel :

```exec
curl -s http://localhost:28610 | grep "Servi par"
```

Elle répond, avec le même nom de pod qu'avant la panne :

```text
          Servi par : <strong>vote-644d47956b-qm48q</strong>
```

Contrairement à l'étape précédente, le tunnel a tenu : il est branché sur le pod, et le pod n'a pas changé. Vous avez donc vu deux pannes, et deux réparations différentes. Quand le pod a disparu, le **Deployment** en a créé un **nouveau**, sous un autre nom. Quand le conteneur a planté, Kubernetes l'a **redémarré dans le même pod**, et `RESTARTS` a augmenté de 1.

Arrêtez le tunnel avant de passer à la suite :

```exec
pkill -f "port-forward deployment/vote 28610"
```

```ack
J'ai vu RESTARTS passer à 1, sans changement de nom
```
