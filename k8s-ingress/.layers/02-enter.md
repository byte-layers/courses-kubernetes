---
title: "Entrer par la porte, puis la refermer"
description: "Joindre vote par le contrôleur d'Ingress, voir les visites se répartir, situer les façons d'exposer une application, retirer l'Ingress"
---

La règle est en place et le contrôleur l'a prise en charge. Il reste à entrer par cette porte, pour voir ce qu'elle apporte de plus qu'un tunnel vers l'application.

## Joindre le contrôleur depuis votre machine

Sur un poste hébergé, la porte d'entrée est déjà ouverte sur le port 8000 du poste, prévu à la création du cluster. Ne lancez donc ni la commande du tunnel ci-dessous, ni celle qui l'arrête à la fin de l'étape, et tapez les commandes `curl` à la main dans le terminal, en écrivant `8000` à la place de `28632`.

Sur votre poste, quand minikube tourne dans Docker, l'adresse du nœud affichée dans la colonne `ADDRESS` de l'Ingress n'est pas joignable depuis votre machine. Vous ouvrez donc un tunnel, non plus vers l'application, mais vers le **contrôleur** : le port 28632 de votre machine mène au port 80 du Service `ingress-nginx-controller`, rangé dans le namespace `ingress-nginx`.

```exec
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 28632:80 > /dev/null 2>&1 &
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `kubectl port-forward` | ouvre un tunnel entre votre machine et le cluster |
> | `-n ingress-nginx service/ingress-nginx-controller` | la cible du tunnel : le Service du contrôleur, dans son namespace |
> | `28632:80` | le port 28632 de votre machine mène au port 80 de ce Service |
> | `> /dev/null` | jette ce que la commande affiche, pour ne pas encombrer le terminal |
> | `2>&1` | jette aussi ses messages d'erreur, au même endroit |
> | `&` | lance la commande en arrière-plan : le terminal vous rend la main, et le tunnel reste ouvert |

Le terminal affiche un numéro de tâche, du type `[1] 12345`, et vous rend la main. Vérifiez que la porte répond. `curl` demande une page depuis le terminal, ici la page d'accueil de `vote`, et `-s` retire l'affichage de la progression. Le `|` passe la réponse à `grep`, qui n'en garde que la ligne contenant « Servi par », celle où la page donne le nom du pod qui l'a fabriquée :

```exec
curl -s http://localhost:28632/ | grep "Servi par"
```

La réponse vient d'un pod de `vote`, alors que vous vous êtes adressé au contrôleur : il a appliqué votre règle. Si rien ne s'affiche, le tunnel n'a pas fini de s'ouvrir : relancez la commande une seconde plus tard.

```text
          Servi par : <strong>vote-fc57495c-smqsr</strong>
```

```ack
vote répond par la porte d'entrée
```

## Voir les visites se répartir entre les pods

Un tunnel ouvert directement vers `vote` ne mène qu'à un seul pod, toujours le même. Faites une prévision : par la porte d'entrée, 6 visites de suite arriveront-elles au même pod ?

```exec
for i in 1 2 3 4 5 6; do curl -s http://localhost:28632/ | grep "Servi par"; done
```

> 🔍 **Comprendre la commande**
>
> | Morceau | Rôle |
> |---|---|
> | `for i in 1 2 3 4 5 6` | répète ce qui suit 6 fois, une fois par valeur de la liste |
> | `do … done` | entoure la commande à répéter |
> | `curl -s … \| grep "Servi par"` | la commande de la section précédente : demander la page et n'en garder que la ligne « Servi par » |

Le nom du pod change au fil des lignes :

```text
          Servi par : <strong>vote-fc57495c-bkvsf</strong>
          Servi par : <strong>vote-fc57495c-sjkxq</strong>
          Servi par : <strong>vote-fc57495c-sjkxq</strong>
          Servi par : <strong>vote-fc57495c-smqsr</strong>
          Servi par : <strong>vote-fc57495c-sjkxq</strong>
          Servi par : <strong>vote-fc57495c-sjkxq</strong>
```

Cette fois, le contrôleur d'Ingress lit la liste des pods du Service, ses `Endpoints`, puis répartit lui-même **chaque visite** entre les 3 réplicas. Le même pod peut répondre deux fois de suite : relancez la commande si l'un d'eux manque à l'appel.

Faites le même essai dans votre navigateur : ouvrez `http://localhost:28632`, ou passez par l'onglet **PORTS** de VS Code sur un poste hébergé, et rechargez la page plusieurs fois en regardant la ligne « Servi par ». **Le nom du pod change.**

> 💡 **Une seule application par chemin.** Votre règle envoie tout ce qui arrive sur `/` vers `vote`. Une autre application écrite pour être servie à la racine d'un site ne peut donc pas passer par la même règle. En entreprise, on donne à chaque application son **propre nom de domaine** (`vote.example.com`, `result.example.com`) : un Ingress sait aussi aiguiller selon le nom demandé, grâce au champ `host`.

```ack
J'ai vu plusieurs pods répondre par la même porte
```

## Les façons d'exposer une application

Vous connaissez maintenant le Service et l'Ingress. Voici le paysage complet, pour vous y retrouver quand une équipe en parle :

| Objet | Joignable depuis | Usage typique |
|---|---|---|
| Service `ClusterIP` (par défaut) | l'intérieur du cluster seulement | faire communiquer les morceaux d'une application |
| Service `NodePort` | l'extérieur, sur un port (30000 à 32767) de chaque nœud | tests, petits environnements |
| Service `LoadBalancer` | l'extérieur, via un répartiteur de charge fourni par le cloud (une adresse externe, souvent publique) | exposer un service non web, ou un contrôleur d'Ingress : sur un poste hébergé, Traefik est justement exposé ainsi |
| **Ingress** | l'extérieur, en HTTP et HTTPS, avec des règles par nom et par chemin | le cas le plus courant pour les sites et les API web |

`kubectl port-forward` n'est pas dans ce tableau : c'est un outil de test, qui ne sert que la personne qui l'a lancé.

> 🗣️ **En réunion projet.** « *Il faut ajouter une règle d'Ingress* » signifie qu'on veut rendre une application (ou une nouvelle adresse) accessible depuis l'extérieur. C'est aussi au niveau de l'Ingress qu'on installe généralement les **certificats HTTPS** et qu'on branche les noms de domaine. Vous entendrez sûrement parler de **Gateway API** : c'est le successeur de l'Ingress, plus riche. L'API Ingress n'évolue plus, et le contrôleur NGINX historique (celui de minikube) n'est plus maintenu par le projet Kubernetes depuis 2026 : les nouveaux projets partent de plus en plus sur Gateway API. Les notions restent les mêmes : des règles d'aiguillage et un contrôleur qui les applique.

```ack
Je sais situer ClusterIP, NodePort, LoadBalancer et Ingress
```

## Refermer la porte

Il reste à ranger. Arrêtez d'abord le tunnel, qui tourne encore en arrière-plan. `pkill -f` arrête les programmes dont la ligne de commande contient le texte donné :

```exec
pkill -f "port-forward -n ingress-nginx service/ingress-nginx-controller 28632"
```

Retirez ensuite l'Ingress `vote`, sans bloc à copier cette fois. Une règle sur `/` sans nom de domaine capte toutes les visites du contrôleur : laissée en place, elle gênerait la prochaine application que vous voudrez exposer à cette adresse. `kubectl delete -f`, suivi du chemin d'un fichier, supprime du cluster ce que ce fichier décrit. Le fichier `vote.ingress.yml`, lui, reste dans votre dossier.

Le Deployment `vote`, son Service et le contrôleur d'Ingress restent en place.

```hint
La commande est `kubectl delete -f ~/byte-layers/kubernetes/l3/vote.ingress.yml`. `kubectl get ingress` doit ensuite répondre qu'il ne trouve rien.
```

```verify
if kubectl get ingress vote >/dev/null 2>&1; then
  echo "L'Ingress vote existe encore. Supprimez-le avec kubectl delete -f suivi du chemin de vote.ingress.yml."
  exit 1
fi
if ! kubectl get service vote >/dev/null 2>&1 || ! kubectl get deployment vote >/dev/null 2>&1; then
  echo "L'Ingress est retiré, mais vote ou son Service a disparu aussi. Remettez-les : kubectl apply -f ~/byte-layers/kubernetes/l3/vote.deploy.yml -f ~/byte-layers/kubernetes/l3/vote.svc.yml"
  exit 1
fi
echo "La porte est refermée, et vote tourne toujours derrière son Service."
```

````solution
```bash
kubectl delete -f ~/byte-layers/kubernetes/l3/vote.ingress.yml
```
````

## Récap

- Un **Ingress** est une liste de règles d'aiguillage : tel nom ou tel chemin mène à tel Service.
- Il ne fait rien sans un **contrôleur d'Ingress** (NGINX, Traefik…), le programme qui reçoit les visites et applique les règles.
- Le contrôleur répartit chaque visite entre les pods du Service, là où un `port-forward` n'en joint qu'un seul.
- `ClusterIP` pour l'intérieur, Ingress pour le web extérieur : c'est la combinaison la plus courante.
- **Gateway API** prend la suite de l'Ingress, avec les mêmes notions.

Le fichier final se trouve dans [`solution/vote.ingress.yml`](../solution/vote.ingress.yml).

> 📖 [Ingress, documentation officielle](https://kubernetes.io/fr/docs/concepts/services-networking/ingress/) · [Les types de Service](https://kubernetes.io/fr/docs/concepts/services-networking/service/) · [Gateway API](https://gateway-api.sigs.k8s.io/) (en anglais)

## Questions sur l'Ingress

```quiz
title: "L'Ingress"
tracksProgress: false
showNotice: false
questions:
  - prompt: "Vous appliquez un Ingress dans un cluster qui n'a aucun contrôleur d'Ingress. Que se passe-t-il ?"
    answers:
      - "L'application devient joignable depuis l'extérieur"
      - "Les règles sont enregistrées, mais personne ne les applique : rien n'entre"
      - "Kubernetes installe un contrôleur tout seul"
    correct: 2
    explanation: "L'Ingress n'est qu'une liste de règles. C'est le contrôleur qui reçoit les visites et les applique."
  - prompt: "Dans une règle d'Ingress, que désigne `backend.service.name` ?"
    answers:
      - "Le pod qui doit répondre"
      - "Le Deployment de l'application"
      - "Le Service vers lequel envoyer les visites"
    correct: 3
    explanation: "L'Ingress s'appuie sur un Service, qui sait à tout moment quels pods sont vivants."
  - prompt: "Deux morceaux d'une application doivent se parler à l'intérieur du cluster, sans être joignables de l'extérieur. Que choisir ?"
    answers:
      - "Un Service `ClusterIP`"
      - "Un Service `NodePort`"
      - "Un Ingress"
    correct: 1
    explanation: "`ClusterIP`, le type par défaut, n'est joignable que de l'intérieur du cluster : c'est ce qu'il faut entre les morceaux d'une application."
  - prompt: "En réunion, quelqu'un dit : « il faut ajouter une règle d'Ingress ». Que veut-il obtenir ?"
    answers:
      - "Plus de réplicas pour l'application"
      - "Qu'une application, ou une nouvelle adresse, soit accessible depuis l'extérieur"
      - "Une sauvegarde de la base de données"
    correct: 2
    explanation: "Une règle d'Ingress relie une adresse extérieure, un nom ou un chemin, à un Service du cluster."
```
