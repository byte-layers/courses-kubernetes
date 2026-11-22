---
title: "Pourquoi Kubernetes : face à Swarm, Mesos et Nomad"
description: "Pourquoi Kubernetes s'est imposé parmi les orchestrateurs de conteneurs : cinq candidats pour un même rôle (Mesos et Marathon, Kubernetes, Amazon ECS, Nomad, Docker Swarm), les rivaux qui l'adoptent en 2017-2018, une API qu'on étend, l'état des lieux de novembre 2026, et quand le plus simple reste le bon choix."
tags:
  - bit
  - kubernetes
  - containers
  - docker
difficulty: beginner
duration: 12
requires: "0.8.0"
---

Un orchestrateur de conteneurs fait tourner une application sur un groupe de machines, le **cluster** : il pose les copies, les relie et relance ailleurs ce qui tombe. Plusieurs outils tiennent ce rôle. Cette page explique **pourquoi Kubernetes l'a emporté**, concrètement, face à Docker Swarm, Apache Mesos, Nomad et Amazon ECS, et dans quels cas un outil plus simple reste le bon choix. L'exemple qui sert de fil est l'application de vote, cinq pièces à faire tourner sur plusieurs machines.

**Pour qui ?** Pour tout le monde, y compris les profils non techniques. Rien à installer, rien à lancer.

**Ce qui est supposé** : savoir ce qu'est un conteneur, et pourquoi on confie des conteneurs à un orchestrateur, ce que raconte [Pourquoi un orchestrateur](https://byte-layers.com/fr/bits/vm-to-orchestrator).

> ⚠️ **Cette page porte une date : novembre 2026.** L'état de chaque outil, ses soutiens et sa licence peuvent changer, et cette page sera revue. Retenez d'abord les critères : qui le porte, ce qu'il y a autour, ce qu'il faut faire tourner.

# Pourquoi Kubernetes : face à Swarm, Mesos et Nomad

## Cinq orchestrateurs pour un même rôle

Vers 2016, une équipe qui veut faire tourner l'application de vote sur plusieurs machines a l'embarras du choix.

```scene
lesson: assets/why-kubernetes.scene.json
step: contenders
```

**Mesos** vient de l'université de Berkeley (2009) ; avec **Marathon**, il fait tourner des conteneurs à grande échelle. **Kubernetes** est ouvert par Google en 2014. AWS lance **ECS**, un service tout prêt, mais chez lui seulement. HashiCorp sort **Nomad** en 2015, Docker intègre le **mode swarm** à son moteur en 2016. Tous décrivent, posent et relancent ; chacun a **son format** : JSON, YAML, HCL ou le fichier Compose.

## 2017-2018 : les rivaux adoptent Kubernetes

Ce qui a tranché n'est pas une fonction de plus, mais un mouvement du marché.

```scene
lesson: assets/why-kubernetes.scene.json
step: convergence
```

Google propose Kubernetes en **service géré** dès 2015. En 2017, Mesosphere l'ajoute à DC/OS et Docker l'annonce à côté de Swarm : les **rivaux eux-mêmes** l'intègrent. En 2018, Azure et AWS ouvrent AKS et EKS. Kubernetes appartient à une fondation neutre, la CNCF, et non à un éditeur : chacun peut bâtir dessus sans dépendre d'un concurrent. Il devient le **terrain commun**.

## Une API qu'on étend : de nouveaux types d'objets

Une autre raison tient à la façon dont Kubernetes se pilote : tout passe par son **API**.

```scene
lesson: assets/why-kubernetes.scene.json
step: extensible
```

Un outil peut **ajouter un type d'objet** à l'API, une *CustomResourceDefinition* (CRD), avec un contrôleur qui le réalise. cert-manager ajoute `Certificate`, Argo CD `Application`, CloudNativePG un `Cluster` PostgreSQL. On les décrit dans le même fichier, avec la même commande, `kubectl apply`, et les mêmes droits. Swarm et Nomad n'ont pas d'équivalent : c'est sur Kubernetes que les éditeurs **bâtissent** leurs outils.

## L'état des lieux, novembre 2026

Dix ans après, que sont devenus les cinq candidats ?

```scene
lesson: assets/why-kubernetes.scene.json
step: today
```

La fondation Apache a **archivé** Mesos en 2025, faute d'activité ; DC/OS avait pris fin en octobre 2021, et Marathon est archivé. Les autres sont **maintenus**. Swarm vit dans Docker Engine, et Mirantis a annoncé en 2025 cinq ans de support de plus. Nomad, chez HashiCorp racheté par IBM, est sous licence BSL depuis 2023. ECS reste un service AWS. Kubernetes est **quasiment** le standard de fait.

> 💡 **Remarque.** La licence BSL (*Business Source License*) laisse lire et utiliser le code, mais interdit d'en faire un service concurrent de celui de l'éditeur. Ce n'est pas une licence libre au sens de l'*Open Source Initiative*.

## Ce qu'il faut faire tourner : le plus simple a sa place

Kubernetes a gagné le marché. Est-il pour autant le bon choix pour chaque équipe ?

```scene
lesson: assets/why-kubernetes.scene.json
step: right-size
```

Swarm tient dans le moteur Docker et reprend le fichier Compose ; Nomad tient dans **un seul binaire**. Kubernetes demande un control plane et plusieurs pièces sur chaque nœud, à installer, surveiller et mettre à jour. Un service géré en prend une partie, pas tout. Une petite équipe sur Compose, avec trois machines, est souvent mieux servie par Swarm : le bon outil est celui qu'elle sait **exploiter**.

## Les cinq, côte à côte

Le tableau reprend les critères de la page. L'état est celui de novembre 2026.

| | Porté par | On décrit avec | Ce qu'il faut faire tourner | État en novembre 2026 | Un bon choix quand… |
|---|---|---|---|---|---|
| **Kubernetes** | la CNCF (Linux Foundation), ouvert par Google en 2014 | des manifestes YAML, et des types ajoutés (CRD) | un control plane, et plusieurs pièces par nœud ; ou un service géré | maintenu, projet diplômé de la CNCF depuis 2018, proposé par les grands clouds | il faut un écosystème, de la portabilité entre fournisseurs, une équipe outillée ou un service géré |
| **Docker Swarm** | Docker, et Mirantis pour l'offre entreprise | le fichier Compose | le moteur Docker, mode swarm intégré | maintenu ; Mirantis a annoncé en 2025 cinq ans de support de plus | une petite équipe, déjà sur Compose, quelques machines |
| **Nomad** | HashiCorp, racheté par IBM en 2025 | un *job* en HCL | un binaire, serveur ou client, et un moteur pour les conteneurs | maintenu, licence BSL depuis 2023 | des conteneurs et d'autres tâches (programmes, Java), une équipe déjà sur les outils HashiCorp |
| **Amazon ECS** | AWS | une *task definition* en JSON | rien : c'est un service | maintenu, propriétaire | tout est chez AWS et doit y rester |
| **Mesos + Marathon** | la fondation Apache, Mesosphere (D2iQ) | une application Marathon en JSON | des maîtres Mesos, ZooKeeper, des agents | archivé (Apache Attic, 2025) ; DC/OS en fin de vie depuis octobre 2021 | plus aucun cas : migrer |

> 📖 [Kubernetes à la CNCF](https://www.cncf.io/projects/kubernetes/) · [Les CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) · [Docker Swarm](https://docs.docker.com/engine/swarm/) · [Mirantis prolonge le support de Swarm (juillet 2025)](https://www.mirantis.com/company/press-center/company-news/mirantis-extends-swarm-support-another-five-years/) · [Nomad](https://developer.hashicorp.com/nomad) · [IBM finalise le rachat de HashiCorp (février 2025)](https://newsroom.ibm.com/2025-02-27-ibm-completes-acquisition-of-hashicorp,-creates-comprehensive,-end-to-end-hybrid-cloud-platform) · [Amazon ECS](https://aws.amazon.com/ecs/) · [Apache Mesos dans l'Apache Attic](https://attic.apache.org/projects/mesos.html) · [La fin de DC/OS, FAQ de D2iQ](https://support.d2iq.com/hc/en-us/articles/4409225352340-DC-OS-Frequently-Asked-Questions-FAQ)

## Ce qu'il faut retenir

- Mesos et Marathon, Kubernetes, ECS, Nomad et Swarm tiennent **le même rôle** : décrire l'application, poser les copies, relancer ce qui tombe.
- Kubernetes l'a emporté parce que les **clouds** l'ont proposé en service géré et que ses **rivaux** l'ont intégré à leurs produits, en 2017 et 2018.
- Il appartient à une **fondation neutre**, la CNCF : aucun éditeur ne le contrôle seul.
- Son **API s'étend** : un outil y ajoute ses types d'objets (CRD) et son contrôleur. C'est pourquoi les éditeurs bâtissent dessus.
- En novembre 2026, Mesos est **archivé** ; Swarm, Nomad et ECS sont **maintenus**.
- Kubernetes demande **plus de pièces** à faire tourner. Pour une petite équipe sur Compose, Swarm ou Nomad restent des choix légitimes.

## Trois questions pour vérifier ce que vous avez retenu

```quiz
title: "Pourquoi Kubernetes"
tracksProgress: false
showNotice: false
place: stage
questions:
  - prompt: "En 2017 et 2018, qu'est-ce qui a fait de Kubernetes le terrain commun des orchestrateurs ?"
    answers:
      - "Il était le seul à savoir relancer un conteneur qui tombe"
      - "Les grands clouds l'ont proposé en service géré, et Mesosphere comme Docker l'ont intégré à leurs produits"
      - "Docker a arrêté Swarm pour le remplacer"
    correct: 2
    explanation: "Tous les orchestrateurs relancent ce qui tombe. Ce qui a tranché, c'est le marché : GKE, AKS et EKS, et les rivaux qui ajoutent Kubernetes à leurs produits. Swarm, lui, n'a pas été arrêté."
  - prompt: "Une équipe installe cert-manager sur son cluster. Que gagne l'API Kubernetes ?"
    answers:
      - "Un nouveau type d'objet, Certificate, qu'on décrit et applique comme les autres, avec un contrôleur qui le réalise"
      - "Rien : cert-manager tourne à côté du cluster, avec sa propre commande"
      - "Une seconde API, réservée aux certificats"
    correct: 1
    explanation: "cert-manager ajoute une CustomResourceDefinition, Certificate, et son contrôleur. On décrit un certificat dans un manifeste, on l'applique avec kubectl apply : même fichier, même commande, mêmes droits."
  - prompt: "Une équipe de trois personnes fait tourner son application avec Docker Compose. Elle veut passer sur trois machines, sans service géré. Que lui conseiller d'abord ?"
    answers:
      - "Kubernetes, puisque c'est le standard de fait"
      - "Apache Mesos, conçu pour les grands clusters"
      - "Docker Swarm, qui reprend son fichier Compose et tient dans le moteur Docker"
    correct: 3
    explanation: "Swarm est maintenu, reprend le format Compose et n'ajoute aucune pièce au moteur Docker : il colle à cette équipe. Kubernetes demanderait un control plane à exploiter. Mesos, lui, est archivé depuis 2025."
```

## Récap

Kubernetes ne s'est pas imposé parce qu'il était seul, mais parce que les clouds, les éditeurs et une fondation neutre se sont rangés derrière lui, et que son API s'étend. Swarm et Nomad restent des choix légitimes pour une petite équipe ; Mesos est archivé. Pour voir ce que Kubernetes apporte sur l'application de vote, un standard, la haute disponibilité et la mise à l'échelle, lisez [Ce que Kubernetes apporte](https://byte-layers.com/fr/bits/k8s-benefits).

```ack
J'ai compris
```
