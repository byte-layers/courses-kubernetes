---
title: "Les stratégies de déploiement dans Kubernetes : Recreate et RollingUpdate"
description: "Les deux stratégies qu'un Deployment Kubernetes connaît pour passer à une nouvelle version : Recreate, qui arrête tout puis relance, et RollingUpdate, qui remplace les pods un par un. Les réglages maxSurge et maxUnavailable, une mise à jour qui se bloque, le retour arrière, et comment construire les autres stratégies, à la main ou avec Argo Rollouts et Flagger."
tags:
  - bit
  - kubernetes
  - deployment
difficulty: beginner
duration: 10
requires: "0.6.0"
---

Sur Kubernetes, une application tourne dans des **pods**, des copies identiques que gère un **Deployment**. Pour livrer une nouvelle version, on change l'image du Deployment, et Kubernetes remplace les pods. La façon de les remplacer, c'est la **stratégie de déploiement** : elle s'écrit dans le manifeste du Deployment, sous `strategy.type`. Kubernetes en connaît deux, `Recreate` et `RollingUpdate`, celle par défaut.

**Pour qui ?** Pour tout le monde, que vous ayez déjà tapé `kubectl` ou non. Rien à installer, rien à lancer.

**Ce qui est supposé** : rien de plus que l'idée d'un conteneur. Pour pratiquer sur votre cluster : [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), [Donner une adresse stable avec un Service](https://byte-layers.com/fr/skills/k8s-services) et [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update).

# Les stratégies de déploiement dans Kubernetes

## Recreate : tout arrêter, puis tout relancer

La stratégie la plus simple tient en une ligne du manifeste.

```yaml
spec:
  replicas: 3
  strategy:
    type: Recreate
```

Quand l'image change, voici ce qui se passe.

```scene
lesson: assets/k8s-rollout-mechanics.scene.json
step: recreate
```

Entre l'arrêt de la v1 et le démarrage de la v2, **plus rien ne répond**. On la choisit pourtant quand deux versions ne peuvent **pas tourner ensemble** : un volume qui ne s'attache qu'à un seul pod, une base que la v2 transforme. Ou pour une application **peu critique**, qu'une courte coupure ne gêne pas, même si ce n'est pas le cas le plus courant sur Kubernetes.

## RollingUpdate : un pod de plus, d'abord

La stratégie par défaut remplace les pods un par un, sans coupure. Deux réglages disent comment.

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # pods en plus du nombre demandé
      maxUnavailable: 0  # pods prêts qui peuvent manquer
```

Ici, le réglage le plus prudent. Avec trois pods, c'est aussi ce que donnent les valeurs par défaut, 25 % chacune.

```scene
lesson: assets/k8s-rollout-mechanics.scene.json
step: surge
```

Le Deployment ne remplace pas les pods lui-même : il fait monter le **nouveau ReplicaSet** et descendre l'ancien. Un pod v2 ne reçoit rien tant qu'il n'est pas **prêt**, et la capacité ne baisse jamais. Le prix : un pod de plus pendant l'opération, donc de la place pour lui dans le cluster.

## RollingUpdate : un pod de moins, d'abord

Pas de place pour un pod de plus ? On inverse : `maxSurge: 0`, `maxUnavailable: 1`.

```scene
lesson: assets/k8s-rollout-mechanics.scene.json
step: unavailable
```

Aucun pod en plus, mais pendant tout le remplacement, **deux pods sur trois** portent le trafic. Si l'application était déjà juste, c'est là qu'elle ralentit. Les deux réglages se combinent : `maxSurge: 1` et `maxUnavailable: 1` vont plus vite, au prix d'un peu des deux.

## Quand la v2 n'est jamais prête

Tout repose sur la **sonde de disponibilité** (*readiness probe*) : c'est elle qui dit qu'un pod est prêt.

```scene
lesson: assets/k8s-rollout-mechanics.scene.json
step: stuck
```

Le pod v2 échoue, la mise à jour s'arrête là, et les pods v1 continuent de répondre. Passé `progressDeadlineSeconds`, dix minutes par défaut, le Deployment signale l'échec **sans rien annuler**. C'est `kubectl rollout undo` qui revient à l'image précédente.

> ⚠️ **Si les pods n'ont pas de sonde de disponibilité**, Kubernetes les croit prêts dès que leur conteneur tourne. Un pod v2 qui ne sait pas répondre reçoit alors du trafic, et le rolling continue.

## Et les autres stratégies ?

Blue/green, canary, A/B testing, feature flag : ce que chacune change pour les utilisateurs est dans [Les stratégies de déploiement](https://byte-layers.com/fr/bits/deployment-strategies). Aucune n'a de réglage tout prêt dans un Deployment : elles se construisent avec les objets que vous connaissez.

| Stratégie | Dans Kubernetes |
|---|---|
| **Blue/green** | deux Deployments, un Service : la bascule change le sélecteur du Service |
| **Canary** | deux Deployments derrière un Service : 9 pods v1 et 1 pod v2 donnent environ 10 % ; pour un dosage précis, des poids à la porte d'entrée du cluster |
| **A/B testing** | des règles à la porte d'entrée, selon un en-tête ou un cookie |
| **Feature flag** | rien : l'interrupteur est dans le code, sa valeur peut venir d'une ConfigMap |

> 💡 **Remarque.** La mise à jour de la base, lancée une seule fois avant le déploiement, se confie souvent à un **Job** : l'objet Kubernetes qui exécute une tâche puis s'arrête.

## Et avec un outil ?

À la main, c'est vous qui changez les poids, surveillez les mesures et décidez de la suite. Des outils le font à votre place, palier par palier : c'est la **livraison progressive** (*progressive delivery*). Vous décrivez les paliers, par exemple 10 %, une pause, une analyse des mesures, puis 50 %. Si une analyse échoue, l'outil **revient en arrière tout seul**.

| | Argo Rollouts | Flagger |
|---|---|---|
| **Ce qu'on écrit** | un objet `Rollout`, à la place du Deployment | un objet `Canary`, à côté du Deployment existant |
| **Stratégies** | canary, blue/green | canary, blue/green, A/B testing |
| **Retour arrière** | automatique si une analyse échoue | automatique si une analyse échoue |
| **Famille** | Argo, avec Argo CD | Flux |

Pour doser le trafic au pourcentage près, l'un comme l'autre pilotent la porte d'entrée du cluster (un contrôleur Ingress, la Gateway API) ou un service mesh. Sans eux, Argo Rollouts dose en nombre de pods, comme le tableau précédent.

> 💡 **Remarque.** Flux et Argo CD ne livrent pas par paliers : ils appliquent au cluster ce qui est écrit dans Git, c'est le **GitOps**. Argo Rollouts et Flagger viennent par-dessus, pour la façon de livrer. Argo CD a sa place dans [Un cluster de référence, pièce par pièce](https://byte-layers.com/fr/bits/k8s-reference-cluster).

> 📖 [La stratégie d'un Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) · [Une mise à jour qui n'avance plus](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#failed-deployment) · [Les sondes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) · [Un canary avec deux Deployments](https://kubernetes.io/docs/concepts/workloads/management/#canary-deployments) · [Les Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) · [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) · [Flagger](https://docs.flagger.app/)

## Ce qu'il faut retenir

- Un Deployment connaît deux stratégies : `Recreate` coupe le service, `RollingUpdate`, par défaut, remplace les pods un par un.
- Au changement d'image, le Deployment crée un second ReplicaSet et fait passer les pods de l'un à l'autre.
- Le Service n'envoie du trafic qu'aux pods prêts : un pod qui démarre ou qui s'arrête ne reçoit rien.
- `maxSurge` : combien de pods en plus ; `maxUnavailable` : combien de pods prêts peuvent manquer.
- Sans pod v2 prêt, la mise à jour se bloque, la v1 continue de servir ; `kubectl rollout undo` revient en arrière.
- Blue/green et canary se construisent avec deux Deployments.
- Argo Rollouts et Flagger avancent par paliers, lisent les mesures et reviennent en arrière seuls.

```quiz
prompt: "Un Deployment de 4 pods est réglé avec maxSurge: 0 et maxUnavailable: 1. Pendant la mise à jour, combien de pods prêts restent au minimum pour répondre ?"
answers:
  - "4 : un rolling ne baisse jamais la capacité"
  - "3 : un pod prêt peut manquer"
  - "5 : un pod de plus démarre d'abord"
correct: 2
explanation: |
  `maxUnavailable: 1` autorise un pod prêt de moins que demandé : le Deployment arrête un pod v1 avant de démarrer un pod v2, et il en reste 3. Avec `maxSurge: 0`, aucun pod en plus n'est créé.
```

```quiz
prompt: "Après un changement d'image, `kubectl get pods` montre trois pods v1 Running et un pod v2 Running mais 0/1 prêt, depuis vingt minutes. Que vivent les utilisateurs ?"
answers:
  - "Une coupure : la mise à jour est en échec"
  - "Rien de particulier : la v1 répond toujours, le pod v2 ne reçoit pas de trafic"
  - "Une requête sur quatre tombe en erreur sur le pod v2"
correct: 2
explanation: |
  Un pod qui n'est pas prêt n'est pas dans la liste du Service : il ne reçoit rien. Avec `maxUnavailable: 0`, aucun pod v1 ne s'est arrêté. La mise à jour est bloquée, et `kubectl rollout undo` la défait.
```

```ack
J'ai compris
```
