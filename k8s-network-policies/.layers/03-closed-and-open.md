---
title: "Constater ce qui est fermé et ce qui reste ouvert"
description: "Relancer le pod de test, voter pour vérifier que l'application fonctionne toujours, voir qu'une étiquette suffit pour passer, puis situer les NetworkPolicies parmi les autres règles d'un namespace"
---

Les quatre règles sont dans le cluster. Il reste à voir ce qu'elles changent pour un pod étranger, ce qu'elles ne changent pas pour l'application, et ce qu'elles laissent ouvert.

## Relancer le pod de test étranger

Relancez le pod de test de la première étape, exactement le même. Avant de valider, faites votre prévision : que répond-il cette fois ?

```exec
kubectl run test --rm -i --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- sh -c "nc -z -w 3 db 5432 && echo db: ouvert || echo db: bloqué; nc -z -w 3 redis 6379 && echo redis: ouvert || echo redis: bloqué"
```

La réponse met quelques secondes de plus à venir, le temps que les deux essais de connexion abandonnent :

```text
db: bloqué
redis: bloqué
pod "test" deleted from vote-app namespace
```

Le pod de test ne joint plus ni la base ni `redis` : il ne porte aucune des étiquettes autorisées.

> ⚠️ **Encore `ouvert` ?** Votre cluster n'applique pas les règles réseau. Revoyez la section « Vérifier que le cluster sait appliquer des règles réseau » de la première étape : sur votre poste, minikube doit avoir été démarré avec Calico.

## Voter après la fermeture du réseau

Fermer le réseau ne doit rien casser pour l'application. Vérifiez-le en votant. Ouvrez un tunnel vers la page de vote, en arrière-plan :

```exec
kubectl port-forward deployment/vote 28641:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward deployment/vote 28641:80` | ouvre un tunnel temporaire : le port 28641 de votre machine mène au port 80 d'un pod de `vote` |
> | `> /dev/null` | envoie les messages du tunnel dans `/dev/null`, un fichier spécial qui jette ce qu'on y écrit, pour qu'ils n'encombrent pas le terminal |
> | `2>&1` | y envoie aussi ses messages d'erreur |
> | `&` (à la fin) | lance la commande en arrière-plan : le terminal vous rend la main tout de suite |

Vous pouvez ouvrir `http://localhost:28641` dans votre navigateur et voter ; sur un poste hébergé (un VS Code dans le navigateur, fourni par un formateur), passez par l'onglet **PORTS** de VS Code. Votez ensuite une fois depuis le terminal : cette commande envoie à la page le même formulaire qu'un clic sur le premier bouton.

```exec
curl -s -o /dev/null -w "%{http_code}\n" -d "vote=a" http://localhost:28641/
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `curl` … `http://localhost:28641/` | envoie une requête à la page de vote, par le tunnel |
> | `-d "vote=a"` | le contenu du formulaire : un vote pour le premier choix |
> | `-s -o /dev/null` | n'affiche ni la progression (`-s`) ni la page renvoyée (`-o /dev/null`) |
> | `-w "%{http_code}\n"` | affiche seulement le code de la réponse |

Vous devez lire `200`, le code d'une réponse réussie. Si vous lisez `000`, le tunnel n'était pas encore ouvert : relancez la commande. Si elle répond encore `000`, le port 28641 est peut-être déjà pris, et relancer la commande du tunnel sans `> /dev/null 2>&1 &` vous montrera l'erreur.

Un vote suit tout le chemin de l'application : `vote` le dépose dans `redis`, `worker` l'en sort et l'enregistre dans `db`. Demandez directement à la base ce qu'elle a reçu. `kubectl exec` lance une commande **dans** un conteneur, ici `psql`, le client de PostgreSQL. `-U postgres` donne l'utilisateur de la base, et `-c` la requête à exécuter, qui compte les votes :

```exec
kubectl exec deployment/db -- psql -U postgres -c "select vote, count(*) from votes group by vote"
```

La base contient votre vote :

```text
 vote | count 
------+-------
 a    |     1
(1 row)
```

Si le tableau est vide, `worker` n'a pas encore traité le vote : relancez la commande. Le vote est arrivé jusqu'à la base : `vote` a toujours le droit de parler à `redis`, et `worker` à `redis` et à `db`. Fermez le tunnel. `pkill -f` arrête le programme dont la ligne de commande contient le texte donné :

```exec
pkill -f "port-forward deployment/vote 28641"
```

```ack
J'ai vu mon vote dans la base, alors que le pod de test est bloqué
```

## Donner au pod de test l'étiquette de worker

Qu'est-ce qui distingue `worker` du pod de test, aux yeux des règles ? Seulement son étiquette. Relancez le pod de test qui se présente comme `worker`, celui qui était bloqué tant que la règle de `db` manquait. Avant de valider, faites votre prévision.

```exec
kubectl run test --labels=app=worker --rm -i --restart=Never --image=ghcr.io/bngams/kube-busybox:1.37 -- sh -c "nc -z -w 3 db 5432 && echo db: ouvert || echo db: bloqué"
```

Cette fois, il passe :

```text
db: ouvert
pod "test" deleted from vote-app namespace
```

> 🧠 **Ce qui vient de se passer.** Les règles s'appuient sur les **étiquettes** des pods, comme les Services et les Deployments : « les pods `app: worker` peuvent parler aux pods `app: db` ». Elles suivent donc les pods automatiquement, même quand ils sont remplacés et changent d'adresse.

Le revers se voit aussi : quiconque a le droit de créer un pod dans `vote-app` peut lui donner l'étiquette qui ouvre la base. Une NetworkPolicy ne dit pas **qui a le droit de créer quoi** dans le namespace ; c'est le rôle des droits d'accès, le RBAC.

## Ce que ces règles laissent ouvert

Deux choses restent ouvertes, et c'est voulu : le port 80 des pages web, et tout le trafic sortant. `vote` et `result` acceptent tout le monde sur le port 80 : ce sont des pages web, et le contrôleur d'Ingress, qui vit dans un autre namespace, doit pouvoir leur amener les visites. Quant aux pods de `vote-app`, ils peuvent toujours **sortir** vers où ils veulent.

> ⚖️ Ces règles ne portent que sur le trafic **entrant**. On peut aussi limiter le trafic **sortant** (*egress*), par exemple pour qu'aucun pod ne puisse joindre Internet, mais il faut alors penser à autoriser l'annuaire du cluster (le DNS), sans quoi plus rien ne se trouve par son nom.

## Les autres règles d'un namespace

Le namespace est l'endroit où l'on pose les règles d'une équipe ou d'une application. Vous en avez posé deux ; voici le paysage complet, à connaître sans entrer dans les détails :

| Règle | Question à laquelle elle répond |
|---|---|
| **ResourceQuota**, **LimitRange** | combien de ressources ce namespace peut-il consommer ? |
| **NetworkPolicy** | qui peut parler à qui ? |
| **RBAC** (*Role-Based Access Control*, les droits d'accès) | qui a le droit de faire quoi dans ce namespace ? (« l'équipe A peut déployer dans `vote-app`, mais pas lire ses Secrets ») |
| **Pod Security** (la sécurité des pods) | quel niveau de sécurité les pods doivent-ils respecter ? (interdire les conteneurs qui tournent en administrateur, par exemple) |

Ces règles sont nombreuses et faciles à oublier. Des outils vérifient automatiquement que tout ce qui entre dans le cluster les respecte : ce sont des **moteurs de politiques**.

| Outil | Exemples de règles |
|---|---|
| **Kyverno** | « toute image doit venir de notre registre », « tout pod doit avoir des réservations », « interdire la version `:latest` », « tout namespace reçoit automatiquement un quota et une règle `tout-fermer` » |
| **OPA Gatekeeper** | les mêmes besoins, avec un autre langage de règles |

> 🗣️ **En réunion projet.** « *Le déploiement a été refusé par une policy* » : un moteur comme Kyverno a bloqué un fichier qui ne respectait pas une règle de l'entreprise. C'est voulu : les règles de sécurité et de bonnes pratiques sont vérifiées **automatiquement**, avant même d'arriver en production.

> 📖 [Les NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) · [Les droits d'accès (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) · [La sécurité des pods](https://kubernetes.io/docs/concepts/security/pod-security-standards/) · [Kyverno](https://kyverno.io/docs/introduction/)

## Récap

- Par défaut, **le réseau est ouvert** : tout pod peut ouvrir une connexion vers tout autre.
- Une **NetworkPolicy** désigne des pods par leurs étiquettes, et dit qui peut les contacter, sur quel port.
- La bonne pratique : **tout fermer**, puis n'ouvrir que les flux utiles. Les autorisations s'additionnent.
- Une règle n'a d'effet que si le réseau du cluster (le CNI, ici Calico) sait l'appliquer.
- Les règles suivent les étiquettes, pas les adresses : elles tiennent quand les pods sont remplacés.
- Droits d'accès, sécurité des pods, moteurs de politiques comme **Kyverno** : le namespace est le lieu où l'on encadre une équipe.

## Questions sur les règles réseau

```quiz
title: "Les NetworkPolicies"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Un namespace ne contient qu'une NetworkPolicy : `podSelector: {}`, `policyTypes: [Ingress]`, et pas de champ `ingress`. Que se passe-t-il ?"
    answers:
      - "Rien : une règle sans champ `ingress` est ignorée"
      - "Aucun pod du namespace n'accepte plus de connexion entrante"
      - "Les pods du namespace ne peuvent plus sortir vers Internet"
    correct: 2
    explanation: "`{}` désigne tous les pods du namespace, et l'absence de `ingress` veut dire que rien n'a le droit d'entrer. C'est la règle `tout-fermer`. Elle ne dit rien du trafic sortant."
  - prompt: "Avec les quatre règles de l'application, vous ajoutez un sixième composant, `stats`, qui doit lire `db`. Que faut-il faire ?"
    answers:
      - "Rien : `stats` est dans le même namespace que `db`"
      - "Ajouter `stats` à la liste des pods autorisés par la règle qui protège `db`"
      - "Supprimer la règle `tout-fermer`"
    correct: 2
    explanation: "Être dans le même namespace ne suffit pas : `db` n'accepte que les pods désignés par sa règle. On ouvre un flux de plus, sans rouvrir le reste."
  - prompt: "Vous appliquez des NetworkPolicies sur un cluster dont le réseau ne sait pas les appliquer. Que se passe-t-il ?"
    answers:
      - "`kubectl apply` refuse le fichier"
      - "Les règles sont acceptées et listées, mais elles n'ont aucun effet"
      - "Tout le trafic est bloqué par précaution"
    correct: 2
    explanation: "Une NetworkPolicy n'est qu'un objet rangé dans le cluster. C'est le composant réseau, le CNI, qui la fait respecter : sans lui, aucune erreur, et aucun effet. D'où le test avec un pod étranger."
  - prompt: "Dans votre projet, « le déploiement a été refusé par une policy ». Que s'est-il passé ?"
    answers:
      - "Une NetworkPolicy a bloqué le téléchargement de l'image"
      - "Un moteur de politiques, comme Kyverno, a refusé un objet qui ne respectait pas une règle de l'entreprise"
      - "Le quota du namespace est atteint"
    correct: 2
    explanation: "Un moteur de politiques vérifie chaque objet envoyé au cluster avant de l'accepter. Le refus est voulu : la règle est appliquée automatiquement."
```
