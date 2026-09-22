# KUBE - Infrastructure

Provisionnement du cluster Kubernetes et de ses composants transverses.
Le dépôt applicatif est dans `../kube-app`.

## Infrastructure

Compte AWS `302805792326`, région `eu-central-1`, AZ `eu-central-1a`.

| Nœud | Rôle | IP privée | IP publique | Instance |
|---|---|---|---|---|
| node-1 | control plane | `10.0.0.239` | `52.28.139.102` (Elastic) | `i-053b2016e9a5dc459` |
| node-2 | worker | `10.0.0.55` | non fixe | `i-0402d50c33a35cf19` |
| node-3 | worker | `10.0.0.4` | non fixe | `i-029a3af4299b00e26` |

Les instances sont des `t4g.medium` : architecture ARM64, 2 vCPU et 4 Go de RAM
chacune. L'accès se fait par AWS SSM Session Manager, aucune clé SSH n'est
associée aux instances au départ.

Les machines sont éteintes automatiquement chaque soir. Seule l'IP Elastic de
node-1 survit à ce cycle, c'est donc le point d'entrée du cluster et la base des
noms de domaine, résolus via `*.52.28.139.102.sslip.io`.

Le système de fichiers EFS partagé `fs-0a103a839747b0ff3` est monté sur les trois
nœuds et sert de support aux volumes persistants. Sa cible de montage se trouve
dans la même zone et le même groupe de sécurité que les nœuds.

VPC `vpc-00b4a783bb870d2ce`, sous-réseau `subnet-0910c5fbc38f2e216`, groupe de
sécurité `sg-0673c4428720f8ea8`.

## Contraintes

Ces quatre points conditionnent la plupart des choix qui suivent.

1. Toutes les images déployées doivent exister en `linux/arm64`.
2. 12 Go de RAM au total pour la plateforme et l'application : il faut une
   distribution légère et des `requests` mesurées.
3. Aucun load balancer disponible, l'exposition passe par un Ingress en NodePort
   sur node-1.
4. Les IP publiques des workers changent à chaque redémarrage, rien ne doit en
   dépendre : Ansible rebondit sur node-1 et les nœuds communiquent par leurs
   adresses privées.

## Choix techniques

**k3s** plutôt que kubeadm. Avec 4 Go par nœud, un control plane kubeadm complet
consomme une part disproportionnée des ressources. k3s fournit un cluster
conforme dans un seul binaire, containerd inclus, et s'installe en une commande,
ce qui rend le rôle Ansible court et rejouable.

**ingress-nginx** en NodePort. L'infrastructure ne fournit pas de load balancer,
et la Gateway API demanderait une implémentation supplémentaire sans bénéfice
ici. ingress-nginx est le contrôleur le mieux documenté pour ce mode
d'exposition et s'intègre directement à cert-manager.

**Headlamp** comme dashboard. Il gère OIDC par configuration, là où
kubernetes-dashboard s'intègre plus difficilement à une chaîne dex, alors que le
sujet demande de couvrir tous les outils.

**kube-prometheus-stack** pour le monitoring. Il apporte Prometheus, Alertmanager
et Grafana déjà configurés avec les règles d'alerte système. Grafana sert aussi
de visualisation pour Loki, ce qui évite un second outil. La rétention est
réduite à 24 h compte tenu des ressources.

**Loki** avec Grafana Alloy comme collecteur, Promtail étant déprécié.

**ArgoCD** comme opérateur GitOps. Son interface montre directement l'état de
synchronisation et de santé attendu en soutenance, et il embarque dex, qui est
de toute façon imposé pour l'authentification.

**Sealed Secrets** pour les secrets. External Secrets supposerait de déployer et
sécuriser un backend externe. Sealed Secrets chiffre vers le cluster via un seul
contrôleur, ce qui suffit à garantir qu'aucun secret en clair ne soit commité.

**Keycloak** comme fournisseur d'identité, fédéré par dex. C'est l'exemple du
sujet et il permet une vraie gestion des utilisateurs et des groupes, nécessaire
pour mapper des rôles RBAC.

**registry:2** avec authentification htpasswd et TLS pour la registry privée.
Harbor est plus complet mais lourd en mémoire et son support arm64 est incertain.

**cert-manager** avec Let's Encrypt en HTTP-01, sans gestion DNS grâce à
sslip.io.

## Utilisation

```bash
cd ansible
ansible-playbook playbooks/ping.yml
ansible-playbook playbooks/site.yml
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes -o wide
```

### Accès initial aux nœuds

Aucune clé SSH n'étant associée aux instances, la première connexion se fait par
Session Manager, où l'on dépose ensuite une clé publique pour Ansible.

```bash
aws ssm start-session --target i-053b2016e9a5dc459 --region eu-central-1
```

## Ordre de déploiement

1. Cluster k3s et montage EFS avec Ansible
2. ingress-nginx et cert-manager, HTTPS validé sur un service témoin
3. ArgoCD, puis le reste des composants réconciliés depuis `gitops/`
4. Keycloak et dex, OIDC sur l'API Kubernetes puis sur les outils
5. kube-prometheus-stack, Loki et Headlamp
6. Registry privée, Sealed Secrets et ValidatingAdmissionPolicy
7. Application, voir `../kube-app`

## État actuel

Provisionnement du cluster écrit, manifestes `gitops/` à venir.
