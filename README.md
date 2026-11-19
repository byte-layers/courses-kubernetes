# courses-kubernetes

Les contenus du stack Kubernetes de byte_layers. Un dossier par contenu, à plat.

**Le stack Kubernetes, de L0 à L7.** Le parcours qui assemble les contenus est décrit dans `stacks/kubernetes/stack.yml`. Chaque contenu a son tag, de la forme `<dossier>/vN`, et le stack a le sien, `stacks/kubernetes/vN`. Le bit `orchestrator-why`, cité par le premier layer, vit dans le dépôt `courses-containers`.

Une seule application du début à la fin : la voting app (vote, redis, worker, db, result), dans la version de [bngams/example-voting-app](https://github.com/bngams/example-voting-app). Ses images sont publiques, sur `ghcr.io/bngams/kube-*`, en amd64 et arm64.

Prérequis : un terminal, Docker qui répond à `docker ps`, puis un cluster, que le premier layer fait démarrer : minikube sur votre poste (profil `layers-k8s`), ou k3d sur un poste hébergé. Les fichiers vont dans `~/byte-layers/kubernetes/lN/`, où N est le numéro du layer. Les ports sont `286N0` à `286N9`.

Ces contenus viennent de la formation « Kubernetes : vue d'ensemble » ([bngams/kube-overview-labs](https://github.com/bngams/kube-overview-labs)).

Licence : CC BY-NC-SA 4.0.
