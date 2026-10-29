# OpenStack-Helm en GitOps : la démo

Dépôt de la démo du talk **« OpenStack-Helm : déployez votre plateforme de cloud privé sur
Kubernetes »** (Capitole du Libre 2026). Les slides sont dans
[`slides/`](slides/openstack-helm-capitole-du-libre-2026.pptx).

On y déploie, avec **ArgoCD**, un cloud **OpenStack 2026.1** complet sur un cluster
Kubernetes : MetalLB pour le réseau, Rook-Ceph pour le stockage, puis les services
OpenStack-Helm (Keystone, Glance, Cinder, Placement, Neutron, Nova, Heat, Horizon).

> [!WARNING]
> **Dépôt de démonstration.** Les mots de passe sont **en clair** dans les fichiers
> `apps/openstack/*/values-mdp.yaml` (valeur bidon : `B4teau123`). Ne réutilisez pas ce
> dépôt tel quel pour autre chose qu'un lab. En production, les secrets passent par un
> gestionnaire de secrets (OpenBao/Vault + External Secrets, SOPS…), jamais en clair dans git.

## Le principe : un dépôt « umbrella »

Une seule Application ArgoCD est appliquée à la main, la **racine**. Elle synchronise le
dossier `clusters/demo/`, qui déclare tout le reste (modèle *app-of-apps*) :

```
bootstrap/root.yaml                      Application racine (à appliquer une fois)
└── clusters/demo/
    ├── projects.yaml                    AppProjects platform / storage / openstack + namespace openstack
    ├── platform/metallb.yaml            chart MetalLB
    ├── platform/metallb-config.yaml     pool d'IP + annonce L2
    ├── storage/rook-ceph-operator.yaml  opérateur Rook
    ├── storage/rook-ceph-cluster.yaml   CephCluster + StorageClass « general »
    └── openstack/appset.yaml            ApplicationSet : une Application par service OpenStack
```

| Type de composant | Méthode |
|---|---|
| Chart Helm publique (MetalLB, charts OpenStack-Helm) | Application ArgoCD **multi-source** : la chart d'origine + nos `values.yaml` versionnés ici (`$values/apps/...`). Aucune chart recopiée. |
| Objets en plus d'une chart OpenStack (Ingress, alias de Service) | **`extraObjects`** dans les values du service : tout le service se lit dans un seul dossier. |
| Manifests sans chart (CephCluster, pool MetalLB, pod client) | **Kustomize** (`apps/**/kustomization.yaml`). |
| Opérateur Rook | Kustomize sur les manifests upstream épinglés à un commit, + patchs locaux. |
| Services OpenStack | **ApplicationSet** (`clusters/demo/openstack/appset.yaml`) : une ligne par service = une Application. |

Le contenu (values Helm et bases Kustomize) est dans `apps/` :

```
apps/
├── platform/metallb/              values de la chart MetalLB
├── platform/metallb-config/       IPAddressPool + L2Advertisement
├── storage/rook-ceph-operator/    manifests Rook épinglés + patchs
├── storage/rook-ceph-cluster/     CephCluster, CephBlockPool, StorageClass
├── openstack/<service>/
│   ├── values.yaml                overrides OpenStack-Helm (images, endpoints publics…)
│   │                              + extraObjects : alias de Service
│   ├── values-mdp.yaml            mots de passe du service (en clair, démo)
│   └── values-*.yaml              autres overrides (stockage Glance, réseau Neutron,
│                                  horizon/values-ingress.yaml : l'Ingress de tout OpenStack)
└── openstack/client/manifests/    pod openstack-client (Kustomize, pas de chart)
```

L'ordre de déploiement du socle est donné par les annotations `argocd.argoproj.io/sync-wave`
(projets → MetalLB → Rook → Ceph). Pour OpenStack, chaque Application porte un label
`openstack-helm-demo/tier`, à synchroniser dans l'ordre
**infra → identity → services → compute → apps**.

## Prérequis

- Un cluster Kubernetes (la démo tourne sur **RKE2, 3 nœuds**) avec un **ingress-nginx**
  (classe `nginx`, fourni par RKE2).
- **ArgoCD** installé dans le namespace `argocd`.
- Un **disque ou une partition libre** pour l'OSD Ceph.
- Les **labels** OpenStack-Helm sur les nœuds :

  ```bash
  # nœud(s) qui portent les API, la base, RabbitMQ, les agents Neutron…
  kubectl label node <controleur> openstack-control-plane=enabled
  # nœuds qui hébergent les VM (libvirt + nova-compute)
  kubectl label node <compute> openstack-compute-node=enabled
  # Open vSwitch : sur le contrôleur ET les computes
  kubectl label node <controleur> <compute> openvswitch=enabled
  ```

## À adapter avant de déployer

| Quoi | Où |
|---|---|
| URL du dépôt (si vous le forkez) | `bootstrap/root.yaml` et `clusters/demo/` (`grep -r openstack-helm-demo.git`) |
| Plage d'IP MetalLB | `apps/platform/metallb-config/metallb-config.yaml` |
| Nœud et disque de Ceph (`controlplane-01`, `/dev/nvme0n1p3`) | `apps/storage/rook-ceph-cluster/cephcluster.yaml`, `apps/storage/rook-ceph-operator/patch-*.yaml` |
| Interface physique du réseau « public » (`enp1s0`) | `apps/openstack/neutron/values-simple.yaml` |
| Domaine public (`*.openstack.example.com`) | `apps/openstack/*/values.yaml`, `apps/openstack/horizon/values-custom.yaml`, `apps/openstack/horizon/values-ingress.yaml` |
| Mots de passe | `apps/openstack/*/values-mdp.yaml` |

Pour changer le domaine d'un coup :

```bash
grep -rl 'example.com' apps/ | xargs sed -i 's/example\.com/mon-domaine.fr/g'
```

Les endpoints publics sont déclarés en **HTTPS sur le port 443** : dans la démo, un reverse
proxy placé devant le cluster termine le TLS et renvoie en HTTP vers l'ingress (les Ingress
n'ont donc pas de bloc `tls`). Sans reverse proxy, ajoutez du TLS aux Ingress (cert-manager)
ou passez les `scheme`/`port` publics en `http`/`80` dans les `values.yaml`.

## Déployer

```bash
# 1. La racine : ArgoCD crée les projets, les Applications du socle et celles d'OpenStack
kubectl apply -f bootstrap/root.yaml

# 2. Le socle (synchro manuelle : on regarde le diff avant)
argocd app sync metallb metallb-config
argocd app sync rook-ceph-operator rook-ceph-cluster
kubectl -n rook-ceph get cephcluster        # attendre HEALTH_OK

# 3. OpenStack, tier par tier
for tier in infra identity services compute apps; do
  argocd app sync -l openstack-helm-demo/tier=$tier
  argocd app wait -l openstack-helm-demo/tier=$tier --health --timeout 1800
done
```

## Utiliser le cloud

- **Horizon** : `https://openstack.example.com`, utilisateur `admin`, mot de passe `B4teau123`,
  domaine `default`.
- **CLI** : un pod `openstack-client` est déployé dans le namespace `openstack`.

  ```bash
  kubectl -n openstack exec -it openstack-client -- bash
  export OS_AUTH_URL=http://keystone-api.openstack.svc.cluster.local:5000/v3
  export OS_USERNAME=admin OS_PASSWORD=B4teau123 OS_PROJECT_NAME=admin
  export OS_USER_DOMAIN_NAME=default OS_PROJECT_DOMAIN_NAME=default
  openstack service list
  ```

## Bon à savoir

- **Pas de caractères spéciaux dans les mots de passe.** Helm encode le mot de passe dans les
  URL de connexion (`!` devient `%21`) et certains scripts des charts ne le décodent pas :
  avec `B4teau123!`, le job d’attente du cluster RabbitMQ échouerait à s’authentifier.
- **Les `values-mdp.yaml` couvrent tous les comptes** que chaque chart définit, pour qu'aucun
  ne garde le mot de passe par défaut `password`. Les comptes partagés (admin Keystone,
  root MariaDB, admin RabbitMQ) ont la même valeur dans toutes les charts : c'est obligatoire.
- **`extraObjects` : un seul fichier de values par service.** C'est une liste, et Helm ne
  fusionne pas les listes entre fichiers : si deux fichiers d'un même service en définissent,
  le dernier écrase l'autre sans prévenir. Ici, ils sont dans `values.yaml`, sauf pour Horizon
  (`values-ingress.yaml`).
- **Synchro manuelle partout** (sauf la racine) : chaque synchro passe par une relecture du diff.
  OpenStack-Helm crée beaucoup de Jobs (db-init, ks-user…), immuables : en cas de diff sur un
  Job, le supprimer avant de resynchroniser.
- **Ceph mono-nœud** (pool `replicapool`, `size: 1`) : aucune redondance, c'est un lab.
- **Rook** est installé depuis un commit de la branche master (comme sur le cluster de la
  démo) ; pour un vrai déploiement, préférez la chart `rook-ceph` en version stable.
- Charts OpenStack-Helm : version **2026.1**, révision `a659ab8a2`
  (dépôt `https://tarballs.opendev.org/openstack/openstack-helm`).

## Pour aller plus loin

- Documentation OpenStack-Helm : <https://docs.openstack.org/openstack-helm/latest/>
- Sources des charts : <https://opendev.org/openstack/openstack-helm>
