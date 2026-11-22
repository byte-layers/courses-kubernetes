---
title: "Ce que Kubernetes demande : attention au choix"
description: "Avant d'adopter Kubernetes : la complexité, les compétences, les choix d'infrastructure, une montée de version par an, et le coût qui en découle. Quand ce n'est pas le bon choix, qui fait quoi, les questions à poser à son équipe, et quatre projets pour s'exercer."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 20
requires: "0.4.0"
---

Kubernetes est un orchestrateur de conteneurs : il fait tourner une application sur un groupe de machines, le cluster, la répare quand une pièce tombe et la répartit quand la charge monte. Ces gains ont un prix, et il ne se résume pas à la facture des serveurs. Cette page dit **ce que Kubernetes demande** : de la complexité, des compétences, des choix d'infrastructure, des montées de version régulières, et le coût qui découle de tout cela. Elle donne ensuite une grille pour décider, projet par projet, et vous l'appliquez à quatre projets fictifs.

**Pour qui ?** Pour tout le monde, et en particulier pour les profils non techniques. Il n'y a rien à installer ni à lancer : ni cluster, ni terminal, et aucun fichier n'est créé sur votre machine. Comptez une vingtaine de minutes si vous prenez le temps de réfléchir aux quatre cas.

**Ce qui est supposé** : vous savez ce que Kubernetes apporte par rapport à une machine virtuelle et à des conteneurs, ce que raconte [Ce que Kubernetes apporte](https://byte-layers.com/fr/bits/k8s-benefits). Il n'est pas nécessaire d'avoir déjà utilisé Kubernetes : les pages pratiques sont citées là où elles servent.

Vous repartirez en sachant :

- ce que Kubernetes demande à une équipe, et pourquoi le coût en découle ;
- quand Kubernetes est pertinent, et quand il ne l'est pas ;
- qui fait quoi dans un projet sur Kubernetes ;
- quelles questions poser à votre équipe.

# Ce que Kubernetes demande : attention au choix

La page suit l'ordre où les questions se posent : la complexité, les compétences, l'infrastructure et les montées de version, puis le coût qui en découle. Viennent ensuite les situations où Kubernetes n'est pas le bon choix, les rôles, et quatre projets pour vous exercer.

## Ce que Kubernetes apporte, en une phrase

La haute disponibilité, la mise à l'échelle sur plusieurs nœuds, une façon standard de déployer partout et une exploitation décrite dans des fichiers : [Pourquoi Kubernetes : de la machine au cluster](https://byte-layers.com/fr/bits/vm-to-kubernetes) détaille chacun de ces apports. Voici maintenant ce qu'ils demandent en échange.

## La complexité

Kubernetes ajoute **beaucoup de pièces** entre l'application et la machine. Il y a d'abord les objets qui décrivent l'application : une dizaine pour une application ordinaire, chacun avec ses réglages. Il y a ensuite les outils qu'on installe autour, pour la supervision, la sécurité ou la chaîne de livraison (la CI/CD). Chaque pièce est une source de panne possible, et une erreur de configuration peut empêcher une application de démarrer : [Lire une panne : CrashLoopBackOff, ImagePullBackOff, OOMKilled](https://byte-layers.com/fr/bits/k8s-failure-catalog) en montre quelques-unes. Là où Compose tient en un fichier sur une machine, un cluster demande de **comprendre un système entier**.

## Les compétences

Exploiter un cluster demande trois familles de compétences, rarement réunies chez une seule personne. L'**administration système** : les machines, le réseau, le stockage, les sauvegardes. Le **développement** : des applications qui démarrent vite, disent si elles vont bien, et rangent leur configuration hors de l'image. La **sécurité** : les droits d'accès, les mots de passe, l'origine des images. Une équipe sans compétence Kubernetes ne devrait pas s'y lancer seule pour un projet critique, et la formation prend du temps.

## Les choix d'infrastructure

Un cluster de production se conçoit avant de s'installer. La **topologie** d'abord : combien de machines pour la partie qui décide, le plan de contrôle, pour qu'il survive à la perte de l'une d'elles. Le **dimensionnement** ensuite : combien de nœuds, de quelle taille, pour faire tenir les applications avec une marge. Il faut aussi décider où le cluster tourne, sur ses propres machines ou chez un fournisseur. Ces choix sont détaillés en fin de parcours, dans la partie « Vers la prod » : [La topologie d'un cluster Kubernetes](https://byte-layers.com/fr/bits/k8s-cluster-topology), [Dimensionner les nœuds d'un cluster Kubernetes](https://byte-layers.com/fr/bits/k8s-cluster-sizing) et [Vers la prod : installer Kubernetes sur vos machines ou chez un fournisseur](https://byte-layers.com/fr/bits/k8s-on-prem-vs-cloud).

## Le cycle de vie : au moins une montée de version par an

Kubernetes avance vite. Le projet publie environ **trois versions mineures par an** (1.35, 1.36, 1.37…), et chacune reçoit des correctifs, de sécurité notamment, pendant **environ 14 mois**. Pour rester sur une version maintenue, il faut donc monter de version **au moins une fois par an**, et en pratique souvent plus. Chaque montée de version se prépare : lire ce qui change, vérifier les outils installés autour, tester sur un cluster de recette. Avec un service géré, le fournisseur fait une partie du travail et suit son propre calendrier, mais la montée de version reste à planifier.

## Le coût, qui découle de tout ça

Le coût d'un projet sur Kubernetes vient de tout ce qui précède : des personnes formées, une infrastructure qui tourne en permanence, des outils, et du temps pour les montées de version. Pour chaque poste, le tableau donne les questions à poser.

| Poste | Questions à poser |
|---|---|
| **Les personnes** | La formation, le recrutement, les astreintes : c'est souvent le premier poste de dépense |
| **Les nœuds**, c'est-à-dire les serveurs | Combien ? De quelle taille ? Toujours allumés ? Plusieurs nœuds sont nécessaires pour la haute disponibilité, même quand l'application est peu utilisée. Leur nombre dépend de ce que les applications réservent, ce que montre [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending) |
| **Le cluster géré** | Le fournisseur facture-t-il la gestion du cluster lui-même ? C'est souvent quelques dizaines d'euros par mois et par cluster |
| **Les environnements** | Un cluster par environnement (développement, recette, production), ou des espaces séparés sur un cluster partagé ? Peut-on éteindre la recette la nuit ? |
| **Le stockage et le réseau** | Les disques où le cluster garde les données, les sauvegardes, le trafic sortant, les répartiteurs de charge |
| **L'outillage** | La supervision, la centralisation des journaux (les *logs*), la sécurité des images : des outils open source à exploiter soi-même, ou des services payants |
| **Les montées de version** | Qui les fait, à quel rythme, et combien de temps cela prend-il à chaque fois ? |

> 💡 La démarche **FinOps**, qui consiste à piloter les coûts du cloud, s'applique pleinement à Kubernetes : étiqueter les ressources par équipe ou par projet, régler des réservations réalistes, éteindre ce qui ne sert pas, et suivre la facture régulièrement.

## Quand Kubernetes n'est probablement pas le bon choix

Quatre situations où il vaut mieux s'en passer :

- Un **site vitrine**, un blog, une petite application interne avec peu d'utilisateurs : un hébergement simple ou un service « serverless », où le fournisseur lance votre code à la demande, suffit largement.
- Une **seule application monolithique**, c'est-à-dire d'un seul bloc, rarement mise à jour, sans exigence forte de disponibilité.
- Une équipe **sans compétences** Kubernetes, et sans budget pour en acquérir.
- Un **prototype**, ou une première version (un MVP) qui doit sortir vite : on pourra y venir plus tard, et les conteneurs facilitent la migration.

> 🧠 **La bonne question n'est pas « Kubernetes ou pas ? », mais « quel niveau de plateforme ? ».** Entre un simple serveur et un cluster Kubernetes géré en interne, il existe beaucoup d'intermédiaires : des plateformes qui font tourner des conteneurs sans exposer Kubernetes (Google Cloud Run, AWS App Runner, Azure Container Apps, Scaleway Serverless Containers…), ou un Kubernetes **managé**, dont le fournisseur exploite une partie à votre place.

## Les options, en bref

Les options vont de **pas de Kubernetes du tout** (un hébergement classique, une plateforme qui prend votre code ou votre image, du serverless) à un Kubernetes **managé** par un fournisseur de cloud, une **plateforme d'entreprise** fournie par un éditeur, ou un cluster **installé soi-même**. Elles se distinguent par qui gère quoi : [Choisir une distribution Kubernetes](https://byte-layers.com/fr/bits/k8s-distribution-choice) les compare, et [Vers la prod : installer Kubernetes sur vos machines ou chez un fournisseur](https://byte-layers.com/fr/bits/k8s-on-prem-vs-cloud) y revient pour la production.

## Qui fait quoi dans un projet sur Kubernetes

Les responsabilités se répartissent souvent ainsi, même si les intitulés varient d'une organisation à l'autre. La dernière colonne cite une page où tenir chacun de ces rôles, en miniature.

| Rôle | Responsabilités | Où le pratiquer |
|---|---|---|
| **Développeurs** | le code, le Dockerfile, les sondes de santé, la configuration attendue | l'image de [Échauffement : la voting app en conteneurs](https://byte-layers.com/fr/labs/voting-app-containers-warmup), puis les sondes de [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes) |
| **DevOps, équipe applicative** | les fichiers YAML de l'application, la chaîne de livraison, les mises à jour | [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), puis [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update) |
| **Équipe plateforme, Ops, SRE** | le cluster lui-même, les nœuds, l'Ingress, la supervision, la sécurité, les sauvegardes, les montées de version | [Démarrer son cluster Kubernetes avec minikube](https://byte-layers.com/fr/labs/k8s-minikube-install), en version miniature |
| **Chef de projet, PO** | les exigences (disponibilité, performance, budget), le planning des mises en production, la communication en cas d'incident | un cahier des charges à tenir, et une panne à raconter |
| **Sécurité, RSSI** | les droits d'accès, les Secrets, la sécurité des images, la conformité | [Ranger un mot de passe dans un Secret](https://byte-layers.com/fr/skills/k8s-secret) |

Dans ce tableau, « Ops » désigne l'exploitation, SRE (*Site Reliability Engineering*) l'ingénierie de la fiabilité, PO le *product owner*, qui porte les besoins du produit, et RSSI le responsable de la sécurité des systèmes d'information. Les mots techniques de la colonne Responsabilités (Deployment, Ingress, Secret, sonde) sont expliqués dans les pages citées.

> 🗣️ La question « qui est d'astreinte si le cluster tombe à 3 h du matin ? » doit avoir une réponse **avant** la mise en production.

## Quatre projets, six questions

À vous d'appliquer la grille. Voici quatre projets fictifs. Pour chacun, prenez deux ou trois minutes, et répondez pour vous-même à ces six questions, par écrit si possible :

1. **Besoin** : ce projet a-t-il vraiment besoin de ce que Kubernetes apporte : la haute disponibilité, la mise à l'échelle, des livraisons fréquentes, plusieurs pièces à faire tourner ensemble ?
2. **Décision** : Kubernetes, une plateforme plus simple, ou pas de conteneurs du tout ? Si c'est Kubernetes : managé, plateforme d'entreprise ou installé soi-même ?
3. **Équipe** : qui ferait quoi ? Quelles compétences manquent ?
4. **Coûts** : quels postes pèsent le plus ?
5. **Risques** : quels sont les deux principaux risques, et comment les réduire ?
6. **Questions à l'équipe technique** : quelles trois questions le chef de projet doit-il poser avant de valider le choix ?

Sous chaque cas, une question porte sur la décision. Répondez-y avant de lire l'explication : elle donne les éléments de réponse, et il n'y a pas toujours une seule bonne décision.

## Cas A : le site de l'office de tourisme

Une collectivité veut refondre le site de son office de tourisme : des pages d'information, un agenda des événements, un formulaire de contact. Il reçoit environ 2 000 visites par jour, avec un pic l'été. Le prestataire est une petite agence web de 5 personnes, sans compétence Kubernetes. Le budget est limité, et la mise en ligne est prévue dans 3 mois.

```quiz
prompt: "Cas A, l'office de tourisme : que décidez-vous ?"
answers:
  - "Un Kubernetes managé, pour tenir le pic de l'été"
  - "Pas de Kubernetes : un CMS hébergé, un hébergement mutualisé ou un PaaS"
  - "Un Kubernetes installé par l'agence sur ses propres serveurs, pour maîtriser les coûts"
correct: 2
explanation: |
  Kubernetes n'est pas pertinent ici. Un CMS hébergé (un outil de gestion de site, fourni en ligne), un hébergement mutualisé (un serveur partagé entre plusieurs clients) ou un PaaS (une plateforme à laquelle on confie son code, qui s'occupe du reste) suffisent : l'agence n'a pas les compétences, et le besoin de disponibilité reste modéré. Si l'agence travaille déjà avec des conteneurs, une plateforme serverless de conteneurs est une option.
```

## Cas B : la plateforme de réservation

Une entreprise de transport régional lance une plateforme de réservation en ligne : recherche de trajets, paiement, billets électroniques, application mobile. Le trafic est très variable : faible la nuit, très fort aux heures de pointe et pendant les grèves des autres modes de transport. L'application compte une dizaine de services développés par trois équipes, avec des livraisons chaque semaine. Une indisponibilité coûte directement du chiffre d'affaires.

```quiz
prompt: "Cas B, la plateforme de réservation : que décidez-vous ?"
answers:
  - "Une seule grosse machine virtuelle avec Compose : c'est plus simple à exploiter"
  - "Un hébergement mutualisé, moins cher"
  - "Un Kubernetes managé"
correct: 3
explanation: |
  C'est le cas typique pour un Kubernetes **managé** : beaucoup de services, plusieurs équipes, des livraisons fréquentes, une charge très variable qui appelle une mise à l'échelle automatique (*autoscaling*), et un coût élevé des indisponibilités. Les points d'attention : les compétences, la supervision, le paiement (sécurité, conformité à la norme PCI-DSS des cartes bancaires), une base de données probablement managée hors du cluster, et des tests de charge avant les pics.
```

## Cas C : l'application métier interne

Un groupe industriel veut moderniser une application interne de gestion des maintenances, utilisée par 300 techniciens. C'est une application Java unique, avec une base de données. Elle est mise à jour deux fois par an. Le service informatique dispose déjà d'un cluster OpenShift, la plateforme Kubernetes de Red Hat, géré par une équipe plateforme, qui héberge d'autres applications du groupe.

```quiz
prompt: "Cas C, l'application métier interne : que décidez-vous ?"
answers:
  - "L'héberger sur le cluster OpenShift existant, une fois conteneurisée proprement"
  - "Créer un nouveau cluster Kubernetes managé, réservé à cette application"
  - "Écarter tout cluster : une application unique n'a rien à y faire"
correct: 1
explanation: |
  Une application unique, rarement mise à jour, ne *nécessite* pas Kubernetes… mais le cluster OpenShift existe déjà, avec son équipe, qui en porte la complexité et les montées de version. L'héberger dessus peut être le choix le plus économique, si l'application est conteneurisée proprement. La décision dépend aussi du **contexte existant**.
```

## Cas D : le service de données de santé

Une start-up développe un service d'analyse de données médicales pour des hôpitaux. Les données doivent rester hébergées en France, chez un hébergeur certifié pour les données de santé (HDS). L'équipe technique compte 4 développeurs, dont un seul a déjà utilisé Kubernetes. Les clients exigent une forte disponibilité, et chaque hôpital doit disposer d'un environnement isolé.

```quiz
prompt: "Cas D, les données de santé : que décidez-vous ?"
answers:
  - "Le Kubernetes managé le moins cher du marché, quel que soit le fournisseur"
  - "Un cluster installé par les 4 développeurs sur des serveurs loués"
  - "Un Kubernetes managé, ou une offre dédiée, chez un fournisseur certifié HDS"
correct: 3
explanation: |
  La contrainte d'hébergement HDS en France commande le choix : un fournisseur certifié qui propose du Kubernetes managé, ou une offre dédiée. Les environnements isolés par hôpital plaident pour des espaces séparés dans un même cluster (des namespaces, avec des quotas, des droits et des règles réseau), ou pour des clusters séparés, selon le niveau d'isolation exigé. Le risque principal est la compétence : une seule personne connaît Kubernetes, ce qui fait d'elle un point unique de défaillance. Une formation et un accompagnement sont à prévoir.
```

## Les questions à poser à votre équipe

Voici une liste à garder pour vos propres projets. Les pages citées en donnent le sens et les font pratiquer.

**Disponibilité et incidents**

- Combien d'exemplaires pour chaque application ? Sur combien de nœuds, et dans combien de zones, ces centres de données distincts d'un même fournisseur ?
- Nos applications ont-elles des **sondes de santé**, celles de [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes) ?
- Comment revient-on en arrière si une mise en production se passe mal ? En combien de temps ?
- Qui est d'astreinte ? Où voit-on les journaux et les alertes ?

**Données**

- Où sont les données ? Sur des **volumes persistants** du cluster, ou dans une base managée hors du cluster ?
- Comment sont faites les **sauvegardes** ? A-t-on déjà testé une restauration ?

**Sécurité**

- Où sont rangés les **Secrets** ? Qui peut les lire ? Sont-ils dans Git ? [Un Secret est encodé, pas chiffré](https://byte-layers.com/fr/bits/k8s-secret-is-encoded) dit pourquoi la question compte.
- D'où viennent nos images ? Sont-elles analysées pour détecter les failles connues ?
- Qui a accès au cluster, et avec quels droits ?

**Livraison et cycle de vie**

- Comment une nouvelle version arrive-t-elle en production ? Les fichiers YAML sont-ils dans Git, comme le veut [Le fichier fait foi](https://byte-layers.com/fr/bits/file-is-the-source) ?
- Les mêmes images sont-elles utilisées en recette et en production, avec seulement une configuration différente ?
- Qui fait les **montées de version** du cluster, à quel rythme, et sur quel cluster les teste-t-on d'abord ?

**Coûts**

- Combien coûte chaque environnement par mois ? Peut-on éteindre les environnements hors production la nuit ?
- Les réservations de ressources sont-elles réalistes, ou surdimensionnées « par sécurité » ?

> 📖 [Les versions de Kubernetes et leur support](https://kubernetes.io/releases/) · [La période de support des versions](https://kubernetes.io/releases/patch-releases/) · [Un cluster pour la production](https://kubernetes.io/docs/setup/production-environment/) · [Qu'est-ce que le FinOps ?](https://www.finops.org/introduction/what-is-finops/)

## Kubernetes n'est pas toujours le bon choix

Kubernetes apporte beaucoup à une application qui en a besoin, et il coûte cher à celle qui n'en a pas besoin. La décision se prend projet par projet, en regardant l'équipe et le contexte autant que la technique. Pour aller plus loin : le cluster lui-même, dans [Kubernetes en un coup d'œil : le cluster, puis ses pièces](https://byte-layers.com/fr/bits/k8s-overview), puis le choix d'une distribution, dans [Choisir une distribution Kubernetes](https://byte-layers.com/fr/bits/k8s-distribution-choice).

## Récap

- Kubernetes demande d'abord d'accepter une **complexité** : beaucoup de pièces, et un système entier à comprendre.
- Il demande des **compétences** en administration système, en développement et en sécurité.
- Un cluster de production se conçoit : **topologie**, **dimensionnement**, sur site ou dans le cloud.
- Environ trois versions mineures par an, chacune maintenue environ 14 mois : **au moins une montée de version par an**.
- Le **coût** découle de tout cela, et son premier poste est souvent humain.
- Entre « rien » et « Kubernetes installé soi-même », il existe de nombreux intermédiaires. Un chef de projet n'a pas besoin de savoir écrire du YAML, mais il doit savoir **poser les bonnes questions**.

```ack
J'ai compris
```
