---
title: "Les stratégies de déploiement : livrer une nouvelle version sans tout risquer"
description: "Rolling, blue/green, canary, A/B testing et feature flag : ce que chaque stratégie change pour les utilisateurs, ce qu'elle coûte, comment on revient en arrière, ce qu'elle impose à la base de données."
tags:
  - bit
  - kubernetes
  - deployment
difficulty: beginner
duration: 15
requires: "0.6.0"
---

Livrer une nouvelle version, c'est prendre un risque : une coupure, ou un défaut que personne n'avait vu. Rolling, blue/green, canary… chaque stratégie réduit ce risque à sa façon. Cette page les pose une à une sur le même schéma.

**Pour qui ?** Pour tout le monde, profils non techniques compris. Rien à installer, rien à lancer.

**Ce qui est supposé** : vous avez déjà vu une application passer d'une version à l'autre, par exemple dans [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update).

# Les stratégies de déploiement

Chaque section a sa scène : **Jouer** l'ouvre en grand, **pas à pas** l'avance un temps à la fois.

Le décor ne change pas. Les utilisateurs envoient des requêtes, les petits paquets, à un **répartiteur** qui les distribue aux **exemplaires** de l'application. La **v1** est en bleu, la **v2** en vert, et les encarts du haut donnent l'état du moment.

À chaque stratégie, demandez-vous : y a-t-il une **coupure** ? Qui subit un **défaut** de la v2 ? Comment **revient-on en arrière** ?

## Tout arrêter, tout relancer

La façon la plus simple de livrer : on arrête toute la v1, puis on démarre la v2.

```scene
lesson: assets/deployment-strategies.scene.json
step: recreate
```

Entre les deux, **plus rien ne répond**. Et si la v2 a un défaut, tout le monde le subit. Son seul avantage : les deux versions ne tournent **jamais ensemble**.

## Rolling : un par un

Plutôt que tout arrêter, on remplace les exemplaires **un à un**. Une v2 démarre ; quand elle est prête, une v1 s'arrête.

```scene
lesson: assets/deployment-strategies.scene.json
step: rolling
```

Il reste toujours de quoi répondre : **pas de coupure**. En échange, tant que le remplacement dure, la v1 et la v2 **répondent ensemble**, et un même utilisateur peut tomber sur l'une puis sur l'autre. Pour revenir en arrière, on refait un rolling dans l'autre sens.

## Blue/green : une bascule

On installe la v2 à côté de la v1, dans un second environnement complet : le **vert**. Personne ne le voit encore, l'équipe peut le tester.

```scene
lesson: assets/deployment-strategies.scene.json
step: blue-green
```

Puis on **bascule** le trafic vers le vert. Un souci ? On rebascule vers le **bleu**, resté intact : le retour arrière est simple, rien n'est à réinstaller. Le prix : pendant l'opération, il faut **deux fois les machines**.

## Canary : un peu de trafic d'abord

On ajoute un seul exemplaire en v2, et on ne lui envoie qu'une **petite part du trafic** réel, 10 % par exemple.

```scene
lesson: assets/deployment-strategies.scene.json
step: canary
```

On surveille ses erreurs et ses temps de réponse. Tout va bien ? On augmente la part, **palier par palier**. Sinon, on le retire : seule une petite part des requêtes a vu le défaut. Encore faut-il des **mesures fiables** pour décider.

> 💡 **Remarque.** Le nom vient des canaris que les mineurs emportaient au fond : l'oiseau réagissait au gaz avant eux.

## Deux versions, une seule base

Les exemplaires se remplacent, mais la **base de données** reste. Pendant un rolling ou un canary, la v1 et la v2 lisent et écrivent dans **la même**.

```scene
lesson: assets/deployment-strategies.scene.json
step: db-break
```

La base range les données dans des tableaux, et chaque colonne a un nom. Imaginons que la v2 a besoin de renommer une colonne. Si on change la base d'un coup, la v1 cherche encore l'ancien nom… et **tombe en erreur**. Le « tout arrêter, tout relancer » n'a pas ce problème : la v1 est déjà arrêtée.

## Changer la base en trois temps

La parade : ne jamais retirer ce dont la v1 a **encore besoin**.

```scene
lesson: assets/deployment-strategies.scene.json
step: database
```

On **élargit** d'abord, en ajoutant la nouvelle colonne à côté de l'ancienne. On **déploie** la v2, qui sait lire et écrire les deux. Et quand il ne reste plus aucune v1, on **resserre** en retirant l'ancienne. Jusque-là, on peut encore revenir à la v1.

> ⚠️ **Si l'application met elle-même la base à jour au démarrage** : pendant un rolling, plusieurs exemplaires démarrent presque en même temps, et chacun risque de lancer la même mise à jour. Mieux vaut la lancer **une seule fois**, à part, avant de déployer.

## A/B testing : comparer, pas livrer

Ici aussi deux versions tournent, mais chaque **groupe** d'utilisateurs voit toujours la même.

```scene
lesson: assets/deployment-strategies.scene.json
step: ab-testing
```

La question n'est plus « la v2 marche-t-elle ? » mais « laquelle fait le plus de clics, d'inscriptions, d'achats ? ». C'est un **test produit**, souvent mené par le marketing ou l'UX. La gagnante est ensuite livrée avec l'une des stratégies précédentes.

## Feature flag : livrer, puis allumer

Le même code tourne partout. La nouveauté est déjà dedans, cachée derrière un interrupteur : le **flag**.

```scene
lesson: assets/deployment-strategies.scene.json
step: feature-flag
```

On l'allume d'abord pour quelques **testeurs**, en avant-première, puis pour tout le monde, **sans rien redéployer**. Un problème ? On l'éteint, là encore sans redéployer.

> ⚠️ **Si un flag reste en place** une fois la fonctionnalité ouverte à tous, il devient une dette : un chemin de code de plus à maintenir et à tester. On le retire.

## Laquelle choisir ?

| Stratégie | Coupure | Qui voit la v2 au début | Retour arrière | Ce qu'elle demande | Et la base ? |
|---|---|---|---|---|---|
| **Tout d'un coup** | oui | tout le monde | relancer la v1, avec une seconde coupure | rien | mise à jour libre, pendant la coupure |
| **Rolling** | non | une part qui grandit | un rolling dans l'autre sens | des exemplaires qui supportent de cohabiter | une base qui convient aux deux versions |
| **Blue/green** | non | tout le monde, à la bascule | rebasculer vers le bleu | deux fois les machines | la même, si la base est partagée |
| **Canary** | non | une petite part du trafic | retirer le canari | un répartiteur qui dose, et des mesures | une base qui convient aux deux versions |
| **A/B testing** | non | un groupe choisi | fermer le test | un critère, et une mesure d'usage | une base qui convient aux deux versions |
| **Feature flag** | non | ceux pour qui le flag est allumé | éteindre le flag, sans redéployer | du code prévu pour | une base qui sert les deux chemins |

Elles se combinent : un rolling peut livrer un code dont les nouveautés dorment derrière des flags.

## Et dans Kubernetes ?

Un Deployment sait faire le rolling, par défaut, et le « tout arrêter, tout relancer ». Les autres stratégies se construisent avec deux Deployments ou des règles à la porte d'entrée du cluster. Tout cela, avec ses scènes, est dans [Les stratégies de déploiement dans Kubernetes : Recreate et RollingUpdate](https://byte-layers.com/fr/bits/k8s-rollout-mechanics).

> 📖 [Blue/green, par Martin Fowler](https://martinfowler.com/bliki/BlueGreenDeployment.html) · [Canary release](https://martinfowler.com/bliki/CanaryRelease.html) · [Feature toggles](https://martinfowler.com/articles/feature-toggles.html) · [Changer une base en trois temps](https://martinfowler.com/bliki/ParallelChange.html)

## Ce qu'il faut retenir

- Tout remplacer d'un coup coupe le service et expose tout le monde.
- Le rolling évite la coupure, mais mélange les deux versions un moment.
- Le blue/green bascule d'un environnement à l'autre et permet un retour simple, au prix de machines en double.
- Le canary n'expose qu'une petite part du trafic, et ne vaut que par ses mesures.
- L'A/B testing est un test produit, pas une façon de livrer.
- Le feature flag sépare déployer et activer.
- Deux versions, une base : élargir, déployer, resserrer.

```quiz
prompt: "Une équipe veut livrer la v2 de son site de vente. Elle ne veut jamais que deux versions répondent en même temps, et veut pouvoir revenir à la v1 simplement, sans rien réinstaller. Quelle stratégie répond aux deux exigences sans coupure ?"
answers:
  - "Le rolling : il n'y a pas de coupure"
  - "Le blue/green : une bascule, et l'ancien environnement reste prêt"
  - "Le canary : seuls 10 % des utilisateurs sont exposés"
correct: 2
explanation: |
  Le blue/green bascule tout le trafic d'un coup : les utilisateurs ne voient jamais les deux versions mélangées, et l'environnement bleu, resté intact, permet de rebasculer sans rien réinstaller. Le rolling et le canary font tous les deux cohabiter la v1 et la v2.
```

```quiz
prompt: "La nouvelle page de paiement est en production depuis lundi, mais aucun client ne la voit. Jeudi, l'équipe l'active pour 20 % des clients, sans rien redéployer. De quoi s'agit-il ?"
answers:
  - "D'un canary : 20 % du trafic va vers la nouvelle version"
  - "D'un feature flag : le code était livré, il vient d'être allumé"
  - "D'un blue/green : la bascule a eu lieu jeudi"
correct: 2
explanation: |
  Le code était déjà déployé, derrière un interrupteur éteint : c'est un feature flag. Dans un canary, la part de trafic va vers des exemplaires d'une autre version, qu'il faut donc déployer ; ici, il n'y a qu'une version, et rien n'a été redéployé jeudi.
```

```quiz
prompt: "La v2 renomme une colonne de la base. L'équipe déploie en rolling, et renomme la colonne dans la base juste avant. Que se passe-t-il ?"
answers:
  - "Rien : le rolling évite toute coupure"
  - "Les exemplaires encore en v1 tombent en erreur : ils cherchent la colonne sous son ancien nom"
  - "La v2 ne démarre pas tant qu'il reste un exemplaire en v1"
correct: 2
explanation: |
  Pendant un rolling, la v1 et la v2 utilisent la même base. Une fois la colonne renommée, la v1 ne la trouve plus. Il fallait élargir d'abord, en ajoutant la nouvelle colonne à côté de l'ancienne, déployer la v2, puis retirer l'ancienne colonne une fois la v1 disparue.
```

```ack
J'ai compris
```
