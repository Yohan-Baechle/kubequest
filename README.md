# KUBE

Projet Kubernetes : déploiement d'un cluster complet sur AWS et conversion d'une
application docker-compose vers un déploiement GitOps.

Le sujet impose de séparer les deux périmètres, d'où deux répertoires distincts.

- [`kube-infra/`](kube-infra) : provisionnement Ansible du cluster et composants
  transverses (ingress, observabilité, sécurité, GitOps)
- [`kube-app/`](kube-app) : chart Helm et manifestes de l'application

Pour reprendre le projet, suivre la section Mise en route de
[`kube-infra/README.md`](kube-infra/README.md).
