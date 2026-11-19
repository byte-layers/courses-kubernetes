---
title: "Kubernetes dans votre projet : quand, et à quel prix"
description: "Une grille de lecture pour décider : ce que Kubernetes apporte et coûte, quand ce n'est pas le bon choix, qui fait quoi, et les questions à poser à son équipe. Avec quatre projets pour s'exercer."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 20
requires: "0.4.0"
---

Vous savez maintenant ce que fait Kubernetes, et ce qu'on voit dans un projet qui l'utilise. Reste la question que se pose tout chef de projet : **faut-il du Kubernetes pour *mon* projet ?** Et si oui, que faut-il prévoir, qui fait quoi, combien cela coûte ? Cette page vous donne une grille de lecture, puis vous l'appliquez à quatre projets fictifs.

**Pour qui ?** Pour tout le monde, et en particulier pour les profils non techniques. Il n'y a rien à installer ni à lancer : ni cluster, ni terminal, et aucun fichier n'est créé sur votre machine. Comptez une vingtaine de minutes si vous prenez le temps de réfléchir aux quatre cas.

**Ce qui est supposé** : vous avez déjà déployé une application sur un cluster et vous l'avez vue se réparer, grandir et se mettre à jour. C'est ce qu'enseignent [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling) et [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update). Cette page se lit après [De la VM à Kubernetes : ce qui change vraiment](https://byte-layers.com/fr/bits/vm-to-kubernetes), qui montre ce que Kubernetes ajoute à une machine virtuelle et aux conteneurs, et le piège du cluster d'une seule machine.

Vous repartirez en sachant :

- quand Kubernetes est pertinent, et quand il ne l'est pas ;
- quelles sont les grandes options : Kubernetes « managé » dans le cloud, installé chez soi, ou pas de Kubernetes du tout ;
- quels rôles, quels coûts et quels risques porte un projet sur Kubernetes ;
- quelles questions poser à votre équipe.

# Kubernetes dans votre projet : quand, et à quel prix

La grille tient en trois temps : ce que Kubernetes apporte, ce qu'il coûte, et les situations où il vaut mieux s'en passer. Viennent ensuite les options, les rôles et les postes de coût, puis quatre projets pour vous exercer.

## Ce que Kubernetes apporte

La grille commence par les besoins auxquels Kubernetes répond : vous les avez tous rencontrés dans ce parcours.

| Besoin du projet | Ce que Kubernetes apporte | Où vous l'avez vu |
|---|---|---|
| L'application doit rester disponible malgré les pannes | la réparation automatique, plusieurs réplicas, la répartition sur plusieurs serveurs | [Déployer une application avec un Deployment](https://byte-layers.com/fr/skills/k8s-deployment), [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling) |
| La charge varie : des pics, une saisonnalité | le passage à l'échelle, manuel ou automatique | [Passer à l'échelle avec les réplicas](https://byte-layers.com/fr/skills/k8s-scaling) |
| On livre souvent, sans coupure | des mises à jour progressives, un retour arrière en quelques secondes | [Mettre à jour sans coupure et revenir en arrière](https://byte-layers.com/fr/skills/k8s-rolling-update) |
| L'application a plusieurs morceaux, dans plusieurs technologies | chaque morceau dans son conteneur, reliés par des Services | [Donner une adresse stable avec un Service](https://byte-layers.com/fr/skills/k8s-services), [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app) |
| Plusieurs environnements (développement, recette, production), plusieurs équipes | des namespaces, une configuration séparée de l'image | [Ranger et encadrer avec un namespace et un quota](https://byte-layers.com/fr/skills/k8s-namespaces-quotas), [Sortir les réglages de l'image avec une ConfigMap](https://byte-layers.com/fr/skills/k8s-configmap) |
| On veut éviter de dépendre d'un fournisseur | les mêmes fichiers YAML fonctionnent chez AWS, Google, Azure, OVH ou dans son propre datacenter | tout le parcours |

## Ce que Kubernetes coûte

En face de ces apports, quatre coûts. Le premier est souvent le plus lourd.

| Coût | En pratique |
|---|---|
| **Compétences** | Il faut des personnes formées pour l'exploiter : c'est souvent le coût principal. Une équipe sans compétence Kubernetes ne doit pas s'y lancer seule pour un projet critique |
| **Complexité** | Plus d'objets, plus de concepts, plus d'outils autour : la supervision, la sécurité, la chaîne de livraison (CI/CD). Une erreur de configuration peut tout bloquer, comme celles de [Lire une panne : CrashLoopBackOff, ImagePullBackOff, OOMKilled](https://byte-layers.com/fr/bits/k8s-failure-catalog) |
| **Infrastructure** | Un cluster tourne en permanence, avec plusieurs nœuds pour la haute disponibilité (l'application répond encore quand une machine tombe), même quand l'application est peu utilisée |
| **Outillage** | La supervision, la centralisation des journaux (les *logs*), les sauvegardes, la sécurité des images : tout cela est à prévoir et à financer |

## Quand Kubernetes n'est probablement pas le bon choix

Quatre situations où il vaut mieux s'en passer :

- Un **site vitrine**, un blog, une petite application interne avec peu d'utilisateurs : un hébergement simple ou un service « serverless », où le fournisseur lance votre code à la demande, suffit largement.
- Une **seule application monolithique**, c'est-à-dire d'un seul bloc, rarement mise à jour, sans exigence forte de disponibilité.
- Une équipe **sans compétences** Kubernetes, et sans budget pour en acquérir.
- Un **prototype**, ou une première version (un MVP) qui doit sortir vite : on pourra y venir plus tard, et les conteneurs facilitent la migration.

> 🧠 **La bonne question n'est pas « Kubernetes ou pas ? », mais « quel niveau de plateforme ? ».** Entre un simple serveur et un cluster Kubernetes géré en interne, il existe beaucoup d'intermédiaires : des plateformes qui font tourner des conteneurs sans exposer Kubernetes (Google Cloud Run, AWS App Runner, Azure Container Apps, Scaleway Serverless Containers…), ou un Kubernetes **managé**, dont le fournisseur exploite une partie à votre place.

## Les grandes options, et qui gère quoi

Les options se distinguent par une question : qui gère quoi ? Deux mots servent à lire le tableau. Le *control plane*, ou plan de contrôle, est le « cerveau » du cluster : l'API, l'ordonnanceur, la base etcd. Un PaaS, pour *Platform as a Service*, est une plateforme à laquelle on confie son code ou son image, et qui s'occupe du reste.

| Option | Qui gère quoi ? | Pour qui ? |
|---|---|---|
| **Pas de Kubernetes** : hébergement classique, PaaS, serverless | le fournisseur gère presque tout | les petits projets, les premières versions, les équipes réduites |
| **Kubernetes managé** : EKS (AWS), GKE (Google), AKS (Azure), OVHcloud, Scaleway… | le fournisseur gère le plan de contrôle, l'équipe gère ses applications | la plupart des projets qui ont besoin de Kubernetes |
| **Plateforme d'entreprise** : OpenShift, Rancher, Tanzu… | un éditeur fournit Kubernetes, des outils et un support | les grandes organisations, les exigences de support et de conformité |
| **Kubernetes installé soi-même** | l'équipe gère tout, des serveurs à l'application | des contraintes fortes (souveraineté, datacenter existant), une équipe très compétente |

## Qui fait quoi dans un projet sur Kubernetes

Les responsabilités se répartissent souvent ainsi, même si les intitulés varient d'une organisation à l'autre. La dernière colonne vous rappelle que vous avez tenu chacun de ces rôles, en miniature.

| Rôle | Responsabilités | Ce que vous avez fait dans ce rôle |
|---|---|---|
| **Développeurs** | le code, le Dockerfile, les sondes de santé, la configuration attendue | l'image construite dans [Échauffement : la voting app en conteneurs](https://byte-layers.com/fr/labs/voting-app-containers-warmup), puis les sondes et le réglage `VOTE_TITLE` de [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes) |
| **DevOps, équipe applicative** | les fichiers YAML de l'application, la chaîne de livraison, les mises à jour | tout le parcours, du premier Deployment à la mise à jour sans coupure |
| **Équipe plateforme, Ops, SRE** | le cluster lui-même, les nœuds, l'Ingress, la supervision, la sécurité, les sauvegardes | [Démarrer son cluster Kubernetes](https://byte-layers.com/fr/labs/k8s-install), en version miniature |
| **Chef de projet, PO** | les exigences (disponibilité, performance, budget), le planning des mises en production, la communication en cas d'incident | un cahier des charges à tenir, et une panne à raconter |
| **Sécurité, RSSI** | les droits d'accès, les Secrets, la sécurité des images, la conformité | [Ranger un mot de passe dans un Secret](https://byte-layers.com/fr/skills/k8s-secret) |

Dans ce tableau, « Ops » désigne l'exploitation, SRE (*Site Reliability Engineering*) l'ingénierie de la fiabilité, PO le *product owner*, qui porte les besoins du produit, et RSSI le responsable de la sécurité des systèmes d'information.

> 🗣️ La question « qui est d'astreinte si le cluster tombe à 3 h du matin ? » doit avoir une réponse **avant** la mise en production.

## Les coûts, poste par poste

Le coût d'un projet sur Kubernetes ne se résume pas à la facture des serveurs. Pour chaque poste, le tableau donne les questions à poser.

| Poste | Questions à poser |
|---|---|
| **Les nœuds**, c'est-à-dire les serveurs | Combien ? De quelle taille ? Toujours allumés ? La somme des réservations (*requests*) détermine le nombre de nœuds nécessaires, comme vous l'avez vu dans [Réserver des ressources et comprendre un pod Pending](https://byte-layers.com/fr/skills/k8s-requests-pending) |
| **Le cluster managé** | Le fournisseur facture-t-il la gestion du cluster lui-même ? C'est souvent quelques dizaines d'euros par mois et par cluster |
| **Les environnements** | Un cluster par environnement, ou des namespaces sur un cluster partagé ? Peut-on éteindre la recette la nuit, en passant à 0 réplica ? |
| **Le stockage et le réseau** | Les volumes persistants, c'est-à-dire les disques où le cluster garde les données, les sauvegardes, le trafic sortant, les répartiteurs de charge (*LoadBalancer*) |
| **L'outillage** | La supervision, les journaux, la sécurité : des outils open source à exploiter soi-même, ou des services payants |
| **Les personnes** | La formation, le recrutement, les astreintes : c'est souvent le premier poste de dépense |

> 💡 La démarche **FinOps**, qui consiste à piloter les coûts du cloud, s'applique pleinement à Kubernetes : étiqueter les ressources par équipe ou par projet, régler des réservations réalistes, éteindre ce qui ne sert pas, et suivre la facture régulièrement.

## Quatre projets, six questions

À vous d'appliquer la grille. Voici quatre projets fictifs. Pour chacun, prenez deux ou trois minutes, et répondez pour vous-même à ces six questions, par écrit si possible :

1. **Besoin** : quels besoins du tableau « Ce que Kubernetes apporte » ce projet a-t-il vraiment ?
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
  Kubernetes n'est pas pertinent ici. Un CMS hébergé (un outil de gestion de site, fourni en ligne), un hébergement mutualisé (un serveur partagé entre plusieurs clients) ou un PaaS suffisent : l'agence n'a pas les compétences, et le besoin de disponibilité reste modéré. Si l'agence travaille déjà avec des conteneurs, une plateforme serverless de conteneurs est une option.
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

Un groupe industriel veut moderniser une application interne de gestion des maintenances, utilisée par 300 techniciens. C'est une application Java unique, avec une base de données. Elle est mise à jour deux fois par an. Le service informatique dispose déjà d'un cluster OpenShift, géré par une équipe plateforme, qui héberge d'autres applications du groupe.

```quiz
prompt: "Cas C, l'application métier interne : que décidez-vous ?"
answers:
  - "L'héberger sur le cluster OpenShift existant, une fois conteneurisée proprement"
  - "Créer un nouveau cluster Kubernetes managé, réservé à cette application"
  - "Écarter tout cluster : une application unique n'a rien à y faire"
correct: 1
explanation: |
  Une application unique, rarement mise à jour, ne *nécessite* pas Kubernetes… mais le cluster OpenShift existe déjà, avec son équipe. L'héberger dessus peut être le choix le plus économique, si l'application est conteneurisée proprement. La décision dépend aussi du **contexte existant**.
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
  La contrainte d'hébergement HDS en France commande le choix : un fournisseur certifié qui propose du Kubernetes managé, ou une offre dédiée. Les environnements isolés par hôpital plaident pour des namespaces, avec des quotas, des droits et des NetworkPolicies, ou pour des clusters séparés, selon le niveau d'isolation exigé. Le risque principal est la compétence : une seule personne connaît Kubernetes, ce qui fait d'elle un point unique de défaillance. Une formation et un accompagnement sont à prévoir.
```

## Les questions à poser à votre équipe

Voici une liste à garder pour vos propres projets. Vous en comprenez maintenant le sens et les enjeux.

**Disponibilité et incidents**

- Combien de réplicas pour chaque application ? Sur combien de nœuds, et dans combien de zones, ces centres de données distincts d'un même fournisseur ?
- Nos applications ont-elles des **sondes de santé**, celles de [Dire à Kubernetes si l'application va bien : les sondes](https://byte-layers.com/fr/skills/k8s-probes) ?
- Comment revient-on en arrière si une mise en production se passe mal ? En combien de temps ?
- Qui est d'astreinte ? Où voit-on les journaux et les alertes ?

**Données**

- Où sont les données ? Sont-elles sur des **volumes persistants**, ou dans une base managée hors du cluster ?
- Comment sont faites les **sauvegardes** ? A-t-on déjà testé une restauration ?

**Sécurité**

- Où sont rangés les **Secrets** ? Qui peut les lire ? Sont-ils dans Git ? [Un Secret est encodé, pas chiffré](https://byte-layers.com/fr/bits/k8s-secret-is-encoded) dit pourquoi la question compte.
- D'où viennent nos images ? Sont-elles analysées pour détecter les failles connues ?
- Qui a accès au cluster, et avec quels droits ?

**Livraison**

- Comment une nouvelle version arrive-t-elle en production ? Les fichiers YAML sont-ils dans Git, comme le veut [Le fichier fait foi](https://byte-layers.com/fr/bits/file-is-the-source) ?
- Les mêmes images sont-elles utilisées en recette et en production, avec seulement une configuration différente ?

**Coûts**

- Combien coûte chaque environnement par mois ? Peut-on éteindre les environnements hors production la nuit ?
- Les réservations (*requests*) sont-elles réalistes, ou surdimensionnées « par sécurité » ?

> 📖 [Un cluster pour la production](https://kubernetes.io/docs/setup/production-environment/) · [Qu'est-ce que le FinOps ?](https://www.finops.org/introduction/what-is-finops/)

## Récap

- Kubernetes répond à des besoins précis : la **disponibilité**, la **charge variable**, les **livraisons fréquentes**, les **applications à plusieurs morceaux**, la **portabilité**.
- Son principal coût est **humain** : les compétences, l'exploitation, les astreintes.
- Entre « rien » et « Kubernetes installé soi-même », il existe de nombreux intermédiaires : le **Kubernetes managé** est le choix le plus courant.
- Un chef de projet n'a pas besoin de savoir écrire du YAML, mais il doit savoir **poser les bonnes questions** : vous avez maintenant la liste, et vous en comprenez le sens.

```ack
J'ai compris
```
