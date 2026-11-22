---
title: "Au-delà du cluster : pratiques, contraintes, outillage"
description: "Ce qui entoure un cluster Kubernetes de production : l'outillage, posé sur un schéma en huit questions (qui fait tourner les conteneurs, qui déploie, qui fait entrer le trafic, où sont les secrets, qui applique les règles, qui adapte la capacité, qui sauvegarde, qui observe), puis les pratiques des équipes et les contraintes qui pèsent sur les choix. État des lieux de novembre 2026."
tags:
  - bit
  - kubernetes
  - containers
difficulty: beginner
duration: 20
requires: "0.6.0"
---

Vous avez tout pratiqué sur un cluster d'un seul nœud, sur votre machine : l'application de vote, et `kubectl` entre vos mains. En production, le cluster n'est qu'une partie du travail. Il est entouré d'outils, et leurs noms reviennent dans toutes les conversations : Argo CD, Cilium, Vault, Grafana… Cette page les pose un à un sur un même schéma, en répondant chaque fois à une question simple. Vous saurez à quoi sert chaque pièce, et ce qu'elle remplace ou complète dans ce que vous connaissez déjà. Viennent ensuite les façons de travailler qui vont avec ces outils, et les contraintes de l'entreprise qui pèsent sur leur choix.

**Pour qui ?** Pour vous, que vous soyez technique ou non : il s'agit de lire une carte, pas d'installer quoi que ce soit. Rien à lancer, ni cluster ni terminal, et aucun fichier n'est créé sur votre machine.

**Ce qui est supposé** : vous savez ce qu'il y a dans un cluster, ce que décrit [Ce qu'il y a dans un cluster Kubernetes](https://byte-layers.com/fr/bits/k8s-cluster-anatomy), et vous avez déjà déployé, exposé et configuré une application, par exemple avec [Déployer la voting app complète](https://byte-layers.com/fr/labs/k8s-voting-app), [Ouvrir une porte d'entrée avec un Ingress](https://byte-layers.com/fr/skills/k8s-ingress) et [Ranger un mot de passe dans un Secret](https://byte-layers.com/fr/skills/k8s-secret).

> ⚠️ **Ce schéma porte une date : novembre 2026.** Le choix des outils suit les tendances du moment. D'autres outils remplissent les mêmes rôles, et cette page sera revue. Retenez d'abord les questions : elles, elles ne changent pas.

# Au-delà du cluster : pratiques, contraintes, outillage

Le schéma se construit en huit questions, puis une vue d'ensemble ; les pratiques et les contraintes viennent après. Chaque section a sa scène : le bouton **Jouer** l'ouvre en grand, chaque section nomme ses pièces nouvelles, celles des sections précédentes restent à leur place, et la pastille rose d'une carte ouvre le détail de la pièce. Personne n'installe tout cela d'un coup : c'est une cible, vers laquelle une équipe avance pièce par pièce.

## Qui fait tourner les conteneurs ?

Des machines, les nœuds, avec les pièces que tout cluster possède. Vous en connaissez déjà plusieurs : elles tournaient dans votre cluster.

```scene
lesson: assets/reference-cluster.scene.json
step: runtime
```

Le tableau donne le rôle de chaque pièce de ce socle.

| Pièce | À quoi elle sert | Ce qu'elle remplace ou complète |
|---|---|---|
| **Talos Linux** | le système des nœuds : minimal, immuable (on ne le modifie pas, on le remplace), sans SSH ni shell (personne ne s'y connecte pour taper des commandes), réglé par une API | une distribution Linux classique, à durcir et à tenir à jour machine par machine |
| **kube-apiserver** et **etcd** | l'API du cluster, et la base où elle range tout | rien : ce sont les pièces d'origine de Kubernetes |
| **containerd** | lance les conteneurs sur chaque nœud ; Kubernetes lui parle par une interface standard, la CRI | rien : un moteur de conteneurs tournait déjà sur le nœud de votre cluster |
| **Cilium** | le réseau du cluster, appuyé sur eBPF, une technologie du noyau Linux qui permet d'y brancher de petits programmes ; son outil Hubble montre qui parle à qui | il tient la place du plugin réseau (le plugin CNI) de votre cluster, Calico avec minikube, et peut remplacer kube-proxy, la pièce d'origine qui dirige le trafic des Services vers les pods |
| **CSI** | l'interface standard par laquelle Kubernetes obtient des volumes, les disques où une application garde ses données ; derrière, le stockage du cloud ou Longhorn, un stockage installé dans le cluster, qui copie chaque volume sur plusieurs nœuds | rien que ce parcours ait abordé : les volumes sont un sujet à venir |
| **Registre d'images** | garde les images, que chaque nœud télécharge | rien : vous utilisiez déjà `ghcr.io` |

## Qui déploie ?

Jusqu'ici, c'était vous, avec `kubectl apply`. Dans ce cluster, plus personne ne le fait à la main : tout ce qui doit tourner est décrit dans un dépôt Git, et un outil installé dans le cluster vient y lire l'état souhaité. Cette façon de faire s'appelle le **GitOps** : pour changer le cluster, on change Git.

```scene
lesson: assets/reference-cluster.scene.json
step: deploy
```

C'est le premier des trois trajets du schéma : de Git vers Argo CD, puis vers les applications.

| Pièce | À quoi elle sert | Ce qu'elle remplace ou complète |
|---|---|---|
| **Git** | la source de vérité : la description de tout ce qui tourne, avec son historique | les fichiers YAML gardés sur un poste |
| **Helm** et **Kustomize** | Helm emballe une application en paquet, le *chart* ; Kustomize adapte les mêmes fichiers à chaque environnement | les fichiers YAML copiés puis retouchés à la main |
| **Argo CD** | compare en permanence le cluster avec Git, signale tout écart et peut le corriger | les `kubectl apply` lancés à la main ou depuis une chaîne d'intégration continue (CI), l'automate qui construit et teste le code |

Vous retrouvez l'idée de l'[état souhaité](https://byte-layers.com/fr/bits/desired-state), un étage plus haut : Kubernetes rapproche le cluster de ce que dit l'API, Argo CD rapproche l'API de ce que dit Git.

## Qui fait entrer le trafic ?

Les utilisateurs arrivent en HTTPS sur une seule porte, une passerelle (*gateway*), qui route chaque requête vers la bonne application. C'est le deuxième trajet : de l'utilisateur vers la Gateway, puis vers les applications.

```scene
lesson: assets/reference-cluster.scene.json
step: traffic
```

Deux pièces se partagent ce travail.

| Pièce | À quoi elle sert | Ce qu'elle remplace ou complète |
|---|---|---|
| **Envoy Gateway** | fait entrer le trafic, à partir des objets de la **Gateway API** | l'Ingress : la Gateway API est le projet officiel de Kubernetes qui prend sa suite. L'API Ingress reste disponible, mais elle est gelée et n'évoluera plus |
| **cert-manager** | crée les certificats TLS, ceux qui permettent le HTTPS, et les renouvelle avant leur expiration | le renouvellement à la main, et l'oubli qui va avec |

## Où sont les secrets ?

Dans un coffre, hors du cluster et hors de Git. [Un Secret est encodé, pas chiffré](https://byte-layers.com/fr/bits/k8s-secret-is-encoded) : le garder dans un dépôt revient à y écrire le mot de passe.

```scene
lesson: assets/reference-cluster.scene.json
step: secrets
```

L'application, elle, ne voit pas la différence : elle lit toujours un Secret Kubernetes.

| Pièce | À quoi elle sert | Ce qu'elle remplace ou complète |
|---|---|---|
| **Vault** | le coffre : il centralise les secrets, contrôle qui y accède et en garde la trace | les mots de passe écrits dans des fichiers |
| **External Secrets Operator** | lit les secrets dans le coffre et en fait automatiquement des Secrets Kubernetes | les Secrets créés à la main, ou rangés dans Git |

## Qui applique les règles et surveille la sécurité ?

Trois outils, à trois moments : avant qu'un objet existe, une fois qu'il est déployé, et pendant que l'application tourne.

```scene
lesson: assets/reference-cluster.scene.json
step: policies
```

Aucun des trois ne remplace les deux autres.

| Pièce | Quand elle agit | À quoi elle sert |
|---|---|---|
| **Kyverno** | au moment où l'on demande quelque chose au cluster, avant que l'objet existe | applique des règles écrites comme des objets Kubernetes : il valide, modifie ou refuse ce qu'on demande au cluster |
| **Trivy Operator** | une fois l'objet déployé | cherche les failles connues dans les images et les erreurs de configuration, et range ses rapports dans le cluster |
| **Falco** | pendant l'exécution | observe ce que font réellement les applications sur chaque nœud, et alerte quand un comportement est suspect |

Ces outils complètent ce que Kubernetes fournit déjà : les NetworkPolicies, et les droits d'accès à l'API, le RBAC.

## Qui adapte la capacité ?

Vous avez changé le nombre de réplicas à la main. Ici, deux outils le font, à deux étages : l'un ajuste le nombre de pods, l'autre le nombre de machines.

```scene
lesson: assets/reference-cluster.scene.json
step: capacity
```

Les deux se complètent : quand les pods ajoutés par le premier ne tiennent plus sur les nœuds, le second en ajoute.

| Pièce | À quoi elle sert | Ce qu'elle remplace ou complète |
|---|---|---|
| **KEDA** | ajoute ou retire des pods selon des événements, comme le nombre de messages en attente ; il peut descendre à zéro pod | il complète l'outil d'origine de Kubernetes, le HorizontalPodAutoscaler, qui règle le nombre de réplicas selon la charge, sans le remplacer |
| **Karpenter** | ajoute des nœuds quand des pods ne trouvent pas de place, et retire ceux qui ne servent plus | l'ajout de machines à la main ; il lui faut un fournisseur capable de créer des machines à la demande |

## Qui sauvegarde ?

Un outil copie régulièrement le contenu du cluster vers un stockage extérieur. Si le cluster est perdu, on en recrée un, et on restaure.

```scene
lesson: assets/reference-cluster.scene.json
step: backup
```

La sauvegarde sort du cluster : elle ne doit pas disparaître avec lui.

| Pièce | À quoi elle sert | Ce qu'elle remplace ou complète |
|---|---|---|
| **Velero** | sauvegarde et restaure les objets du cluster et les volumes persistants ; sert aussi à migrer vers un autre cluster | les sauvegardes faites à la main, machine par machine |
| **Stockage d'objets** (S3, MinIO) | un stockage qui range des fichiers par leur nom et qu'on atteint par le réseau ; il garde les sauvegardes, hors du cluster | un disque du cluster, qui tomberait avec lui |

## Qui observe ?

Un second cluster, réservé à l'observabilité : si le premier va mal, on voit encore ce qui s'y passe. C'est le troisième trajet : des applications vers Alloy, puis vers le cluster d'observabilité. Ce qui circule s'appelle la télémétrie, et elle a trois formes : les métriques, les journaux et les traces.

```scene
lesson: assets/reference-cluster.scene.json
step: observe
```

Chaque forme de télémétrie a son rangement, et un seul écran les réunit.

| Pièce | À quoi elle sert | Ce qu'elle remplace ou complète |
|---|---|---|
| **Grafana Alloy** | collecte la télémétrie dans le cluster de travail et l'envoie plus loin | il succède à Grafana Agent, arrivé en fin de vie le 1er novembre 2025 |
| **Collector OpenTelemetry** | le point d'entrée du cluster d'observabilité : il reçoit la télémétrie en OTLP, le protocole d'OpenTelemetry, le standard ouvert de la télémétrie, et la répartit | un collecteur par outil |
| **Mimir** | garde les métriques, des chiffres relevés dans le temps | rien que vous ayez utilisé : il donne un stockage de longue durée aux métriques |
| **Loki** | garde les journaux | `kubectl logs`, pod par pod |
| **Tempo** | garde les traces : le parcours d'une requête d'un service à l'autre | un outil de plus à installer |
| **Grafana** | l'écran unique : on y interroge et on y affiche les trois, et on y règle les alertes | un écran par outil |

Les trois rangements déposent leurs données de longue durée dans le même stockage d'objets que les sauvegardes.

## Le cluster en entier, et ses trois trajets

Voici le schéma complet. Les trois trajets y circulent ensemble : le déploiement, de Git vers Argo CD puis les applications ; le trafic, de l'utilisateur vers la Gateway puis les applications ; la télémétrie, des applications vers Alloy puis le cluster d'observabilité.

```scene
lesson: assets/reference-cluster.scene.json
step: overview
```

Un détail compte : les pièces roses, les services de la plateforme, sont elles-mêmes déployées par Argo CD depuis Git, comme les applications. La plateforme se décrit et se livre de la même façon que ce qu'elle porte.

## Les pratiques : comment l'équipe travaille

Les outils servent des façons de travailler. Quatre reviennent dans la plupart des équipes qui exploitent Kubernetes.

| Pratique | Ce qu'elle veut dire | Dans ce schéma |
|---|---|---|
| **GitOps** | Git décrit tout ce qui tourne ; un outil applique cette description au cluster | Argo CD |
| **Infrastructure as code** | les machines, les réseaux et le cluster lui-même sont décrits dans des fichiers versionnés, et créés par un outil (Terraform, OpenTofu, Ansible) | en dehors : c'est ce qui crée le cluster |
| **Règles écrites comme du code** (*policy as code*) | les règles de sécurité et d'usage sont des fichiers, relus et testés comme le code | Kyverno |
| **Observabilité et démarche SRE** | on mesure ce que vivent les utilisateurs, on fixe des objectifs de service (les SLO) et on décide à partir de ces chiffres ; le SRE (*Site Reliability Engineering*) est la démarche née chez Google qui en a fait un métier | Alloy, Grafana et le cluster d'observabilité |

Le point commun : ce qui était un geste à la main devient un fichier qu'on relit, qu'on versionne et qu'on rejoue.

## Les contraintes : ce qui décide avant vous

Une équipe ne choisit pas ses outils dans le vide. Quatre contraintes de l'entreprise reviennent souvent, et elles pèsent plus sur les choix que la mode du moment.

| Contrainte | Ce qu'elle peut imposer |
|---|---|
| **Conformité** | les règles d'un secteur ou d'une loi (le RGPD pour les données personnelles, l'hébergement de données de santé…) : tracer les accès, chiffrer, fournir des preuves lors d'un audit |
| **Souveraineté des données** | où vivent les données et quel droit s'y applique : un hébergeur précis, ou vos propres machines, ce que compare [Vers la prod : installer Kubernetes sur vos machines ou chez un fournisseur](https://byte-layers.com/fr/bits/k8s-on-prem-vs-cloud) |
| **Réseau de l'entreprise** | un pare-feu et un proxy, pas d'accès direct à Internet : les images et les paquets passent par des miroirs internes, les adresses et les certificats suivent les règles de la maison |
| **Plusieurs environnements** | développement, recette, production : un cluster ou un namespace pour chacun, la même description dans Git adaptée à chacun (Kustomize, Helm), et une version qui passe de l'un à l'autre |

## Une carte pour la suite

Chaque famille de pièces de ce schéma est un sujet à part entière, avec ses gestes et ses pièges :

- les paquets, avec Helm et Kustomize ;
- la livraison par Git, le GitOps ;
- les volumes et les données qui durent ;
- la mise à l'échelle automatique ;
- la sécurité et les droits d'accès, le RBAC ;
- la Gateway et les certificats ;
- l'observabilité ;
- l'installation d'un vrai cluster, à plusieurs nœuds.

D'ici là, ce schéma vous sert de carte : quand un outil est cité en réunion, vous savez dans quelle case le ranger, et à quelle question il répond.

> 📖 Le socle : [Talos Linux](https://docs.siderolabs.com/talos/latest/overview/what-is-talos) · [Composants de Kubernetes](https://kubernetes.io/docs/concepts/overview/components/) · [Cilium](https://docs.cilium.io/en/stable/overview/intro/) · [Longhorn](https://longhorn.io/docs/)
>
> 📖 Déployer et faire entrer : [Helm](https://helm.sh/docs/) · [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) · [Argo CD](https://argo-cd.readthedocs.io/en/stable/) · [Gateway API](https://gateway-api.sigs.k8s.io/) · [Envoy Gateway](https://gateway.envoyproxy.io/docs/) · [cert-manager](https://cert-manager.io/docs/)
>
> 📖 Secrets et sécurité : [External Secrets Operator](https://external-secrets.io/latest/) · [Vault](https://developer.hashicorp.com/vault/docs/about-vault/what-is-vault) · [Kyverno](https://kyverno.io/docs/introduction/) · [Trivy Operator](https://aquasecurity.github.io/trivy-operator/latest/) · [Falco](https://falco.org/docs/)
>
> 📖 Capacité et sauvegarde : [KEDA](https://keda.sh/docs/latest/concepts/) · [Karpenter](https://karpenter.sh/docs/) · [Velero](https://velero.io/docs/)
>
> 📖 Observer : [Grafana Alloy](https://grafana.com/docs/alloy/latest/) · [Collector OpenTelemetry](https://opentelemetry.io/docs/collector/) · [Mimir](https://grafana.com/docs/mimir/latest/) · [Loki](https://grafana.com/docs/loki/latest/) · [Tempo](https://grafana.com/docs/tempo/latest/) · [Grafana](https://grafana.com/docs/grafana/latest/)

## Récap

- Un cluster de production répond à huit questions : qui fait tourner les conteneurs, qui déploie, qui fait entrer le trafic, où sont les secrets, qui applique les règles, qui adapte la capacité, qui sauvegarde, qui observe.
- Trois trajets le traversent : le déploiement depuis Git, le trafic des utilisateurs, et la télémétrie vers un cluster à part.
- Autour des outils, des pratiques : le GitOps, l'infrastructure as code, les règles écrites comme du code, l'observabilité et la démarche SRE.
- Des contraintes décident souvent avant vous : la conformité, la souveraineté des données, le réseau de l'entreprise, plusieurs environnements.
- Les outils de ce schéma sont ceux de novembre 2026. Ils changeront ; les questions resteront.

```quiz
title: "Au-delà du cluster"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Dans ce cluster, comment une nouvelle version d'une application arrive-t-elle en production ?"
    answers:
      - "Quelqu'un lance kubectl apply depuis son poste"
      - "On modifie la description dans Git, et Argo CD l'applique au cluster"
      - "Envoy Gateway la télécharge depuis le registre"
    correct: 2
    explanation: "C'est le GitOps : Git est la source de vérité, et Argo CD rapproche le cluster de ce que Git décrit."
  - prompt: "Un mot de passe de base de données doit être fourni à une application. Où est-il rangé ?"
    answers:
      - "Dans le dépôt Git, à côté du Deployment"
      - "Dans l'image de l'application"
      - "Dans le coffre, Vault, d'où External Secrets Operator le lit pour en faire un Secret Kubernetes"
    correct: 3
    explanation: "Le secret reste hors de Git et hors du cluster. L'application, elle, lit un Secret Kubernetes comme d'habitude."
  - prompt: "Pourquoi l'observabilité a-t-elle son propre cluster ?"
    answers:
      - "Parce que Grafana ne fonctionne pas sur un cluster qui porte des applications"
      - "Pour voir encore ce qui se passe quand le cluster de travail va mal"
      - "Parce que les journaux ne peuvent pas quitter le cluster qui les produit"
    correct: 2
    explanation: "Si les outils d'observation tombaient avec le cluster qu'ils observent, on perdrait la vue au moment où l'on en a le plus besoin."
```

```ack
J'ai compris
```
