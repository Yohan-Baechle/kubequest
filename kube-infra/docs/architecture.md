# Architecture

## Vue physique

Les trois instances EC2, le réseau et le stockage partagé.

```mermaid
graph TB
    subgraph internet["Internet"]
        user["Utilisateur<br/>navigateur et kubectl"]
        le["Let's Encrypt"]
        gh["GitHub<br/>Yohan-Baechle/kubequest"]
    end

    subgraph vpc["AWS eu-central-1a - vpc-00b4a783bb870d2ce"]
        n1["node-1<br/>control plane et worker<br/>10.0.0.239<br/>IP Elastic 52.28.139.102"]
        n2["node-2<br/>worker<br/>10.0.0.55"]
        n3["node-3<br/>worker<br/>10.0.0.4"]
        efs["EFS fs-0a103a839747b0ff3<br/>cible de montage 10.0.0.176<br/>monte sur /mnt/efs"]
    end

    user -->|"HTTPS 443<br/>*.52.28.139.102.sslip.io"| n1
    user -->|"API Kubernetes 6443"| n1
    le -->|"challenge HTTP-01 port 80"| n1
    n1 <-->|"6443 et reseau de pods"| n2
    n1 <-->|"6443 et reseau de pods"| n3
    n1 -.->|"NFS 2049"| efs
    n2 -.->|"NFS 2049"| efs
    n3 -.->|"NFS 2049"| efs
    n1 -.->|"ArgoCD tire les manifestes"| gh
```

Les workers n'ont pas d'adresse publique stable, tout le trafic entrant passe
par l'IP Elastic de node-1.

## Composants du cluster

```mermaid
graph TB
    ext["Trafic entrant<br/>*.52.28.139.102.sslip.io"]

    subgraph k3s["Cluster k3s"]
        traefik["Traefik<br/>publie 80 et 443 via servicelb"]

        subgraph plateforme["Plateforme"]
            argo["ArgoCD"]
            headlamp["Headlamp"]
            registry["registry 2<br/>privee et authentifiee"]
        end

        subgraph observabilite["Observabilite"]
            prom["Prometheus"]
            am["Alertmanager"]
            graf["Grafana"]
            loki["Loki"]
            alloy["Grafana Alloy"]
        end

        subgraph securite["Securite"]
            certm["cert-manager"]
            keycloak["Keycloak"]
            dex["dex"]
            sealed["Sealed Secrets"]
        end

        subgraph application["Application"]
            web["sample-app<br/>2 replicas anti-affinite"]
            mysql["MySQL<br/>chart officiel"]
            cron["CronJob mysqldump"]
        end

        api["kube-apiserver"]
        vap["ValidatingAdmissionPolicy"]
        pv["PersistentVolume sur EFS"]
    end

    ext --> traefik
    traefik --> argo
    traefik --> headlamp
    traefik --> graf
    traefik --> keycloak
    traefik --> registry
    traefik --> web

    web --> mysql
    mysql --> pv
    cron --> mysql
    cron --> pv

    alloy -->|"logs des pods"| loki
    prom -->|"metriques"| am
    graf --> prom
    graf --> loki

    web -.->|"image tiree"| registry
    certm -.->|"certificats TLS"| traefik
    api --> vap
```

## Chaîne d'authentification

Un seul compte donne accès à tous les outils et à l'API Kubernetes.

```mermaid
graph LR
    user["Utilisateur"]

    subgraph outils["Outils exposes"]
        graf["Grafana"]
        argo["ArgoCD"]
        headlamp["Headlamp"]
    end

    kubectl["kubectl"]
    api["kube-apiserver<br/>configure avec l'issuer dex"]
    dex["dex"]
    keycloak["Keycloak<br/>utilisateurs et groupes"]

    user --> graf
    user --> argo
    user --> headlamp
    user --> kubectl

    graf -->|"OIDC"| dex
    argo -->|"OIDC"| dex
    headlamp -->|"OIDC"| dex
    kubectl -->|"jeton OIDC"| api
    api -->|"validation du jeton"| dex
    dex -->|"federation"| keycloak
```

## Flux de déploiement GitOps

Aucun déploiement manuel, tout passe par un commit.

```mermaid
graph LR
    dev["Developpeur"]
    build["Build de l'image<br/>linux/arm64"]
    registry["Registry privee"]

    subgraph git["GitHub"]
        infra["kube-infra/gitops<br/>composants"]
        appgit["kube-app/gitops<br/>application"]
    end

    argo["ArgoCD"]
    vap["ValidatingAdmissionPolicy"]
    cluster["Ressources du cluster"]

    dev --> build
    build --> registry
    dev -->|"commit"| infra
    dev -->|"commit"| appgit
    argo -->|"surveille"| infra
    argo -->|"surveille"| appgit
    argo -->|"applique"| vap
    vap -->|"accepte ou refuse"| cluster
    cluster -.->|"tire les images"| registry
    argo -.->|"etat Synced et Healthy"| dev
```

## Déploiement progressif

Comportement attendu lors d'une mise à jour, y compris fautive.

```mermaid
graph TB
    commit["Commit d'une nouvelle version"]
    argo["ArgoCD applique le changement"]
    rollout["Kubernetes demarre les nouveaux pods<br/>sans arreter les anciens"]
    probe{"Readiness probe"}
    ok["Trafic bascule<br/>anciens pods supprimes"]
    ko["Nouveaux pods non prets<br/>anciens pods continuent de servir"]

    commit --> argo --> rollout --> probe
    probe -->|"succes"| ok
    probe -->|"echec"| ko
```
