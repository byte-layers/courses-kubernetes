# Contenus à revoir

Les remarques de relecture de Boris qui ne sont pas encore traitées, contenu par contenu. Une ligne
part d'ici quand le contenu corrigé est publié (avec son tag).

## À revoir

### `k8s-on-prem-vs-cloud` (Vers la prod : on-prem ou cloud)
- **Le théâtre est vide** (relecture du 22/11, http://localhost:3016/fr/bits/k8s-on-prem-vs-cloud/step-1).
  Piste : le bit mélange deux kits, `k8s-managed-cluster` (plan, 0.3) et `k8s-on-prem-cluster`
  (Feuilleté, format 0.2). Passer la scène sur site au kit plan, ou corriger le moteur si le mélange
  doit marcher.
- Vérifier la répartition « à vous / au fournisseur » de la scène cloud : il s'agit peut-être plutôt
  de **vos données** (modèle de responsabilité partagée, comme le schéma de Scaleway) (21/11).
- Le cas d'une installation complexe faite par vous-même, en briques génériques : infrastructure des
  control planes (un ou plusieurs, etcd avec eux ou sur son propre cluster, un répartiteur devant
  l'API), infrastructure des nœuds de travail, infrastructure des volumes. À écrire en un bit à part,
  à côté de la vue simple (21/11).
- Ajouter des exemples de déploiement chez des fournisseurs (21/11).

### `k8s-reference-cluster` (Au-delà du cluster)
- La première version (`k8s-reference-cluster/v1`, kit **Bloc**) montrait mieux le cluster et toutes
  ses pièces internes. La v2, passée en Feuilleté le 21/11, se lit moins bien comme cluster.
  À trancher : revenir au Bloc pour cette scène (une exception à « Feuilleté pour tout »), ou
  retravailler la version Feuilleté (22/11).
- La partie « pratiques, contraintes » du nouveau titre reste à écrire.

### Kit plan (scènes 2D)
- En thème sombre, la boîte Kubernetes se distingue mal des boîtes neutres.
- Sur 390 px, les scènes larges (deux piles côte à côte) restent petites.

### Schémas, idées
- Un schéma avec **etcd à part** : un cluster etcd sur ses propres nœuds (21/11).

## Recouvrements gardés (deux versions)
- `vm-to-kubernetes` (publié, sorti du stack Kubernetes) et `orchestrator-why` (stacks Docker)
  recouvrent `vm-to-orchestrator`. Les trois sont gardés ; le cas est un cas de test du compilateur
  (`docs/ETAPES_CHOIX_COMPILATION.md` et `docs/ETAPES_CHOIX_CONNAISSANCE.md` du monorepo, « Encore ouvert »).
- `k8s-install` (lab publié) est remplacé dans le stack par `k8s-minikube-install` ; l'ancienne
  page reste en ligne telle quelle.
