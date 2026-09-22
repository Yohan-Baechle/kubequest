# KUBE - Application

Conversion de la `sample-app` fournie en docker-compose (Laravel + MySQL) vers un
chart Helm déployé par GitOps.

## Analyse de l'existant

Le `docker-compose.yaml` décrit trois services.

| Service | Conversion |
|---|---|
| `traefik` | supprimé, le routage est assuré par l'ingress-nginx du cluster |
| `app` | Deployment, image construite en arm64 et poussée sur la registry privée |
| `db` | chart officiel Bitnami MySQL, volume persistant sur l'EFS |

Points à traiter lors de la conversion :

- `APP_KEY`, `DB_PASSWORD` et `MYSQL_ROOT_PASSWORD` sont en clair dans le
  compose, ils doivent passer par des Sealed Secrets
- `APP_DEBUG=true` et `APP_ENV=dev` sont à basculer en production
- le volume `./mysql-init` référencé n'est pas présent dans l'archive fournie
- le schéma se crée avec `php artisan migrate` puis `db:seed`, à exécuter dans un
  Job d'initialisation
- le service `app` n'a pas de healthcheck, il faut ajouter une route de
  readiness, qui servira aussi à la démonstration du rollout

## À faire

- [ ] Dockerfile multi-stage en arm64
- [ ] chart Helm : Deployment avec réplicas et anti-affinité, Service, Ingress
      TLS, ConfigMap, Secret, probes, limits et requests
- [ ] dépendance vers le chart Bitnami MySQL et volume persistant EFS
- [ ] Job de migration
- [ ] CronJob de sauvegarde `mysqldump` vers l'EFS
- [ ] manifestes kustomize et Application ArgoCD dans `gitops/`
- [ ] version fautive pour la démonstration du rollout
