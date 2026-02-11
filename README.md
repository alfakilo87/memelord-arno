# memelord-arno

MemeLord (Django) ja Grafana deployment Kubernetesile – namespace `memelord-arno`.

| Rakendus | URL |
|----------|-----|
| MemeLord | https://memelord-arno.ee-lte-1.codemowers.io |
| Grafana | https://grafana-arno.ee-lte-1.codemowers.io |

## Repo failid

| Fail | Kirjeldus |
|------|-----------|
| `memelord.yaml` | MemeLord: StringSecrets (redis, django, database), Service minio (ExternalName), Dragonfly, CNPG Cluster/Database, S3 Policy/S3User/Bucket, Deployment, Service memelord, Certificate, Ingress, OIDCClient |
| `grafana.yaml` | Grafana: ConfigMap (datasources Prometheus/Loki), StatefulSet (sqlite PVC), Service, Certificate, Ingress, OIDCClient |
| `grafana-promote-user.sh` | Helper: tõsta OIDC kasutaja Grafana Admin’iks (SQLite DB muudatus) |

**Välised sõltuvused (klaster):** `minio/default`, `letsencrypt` ClusterIssuer, `passmower` (OIDC), `monitoring` namespace (Prometheus, Loki).

## Elementide sõltuvused (Mermaid)

```mermaid
flowchart TB
    subgraph memelord["memelord.yaml"]
        SS_redis[StringSecret memelord-arno-redis]
        SS_django[StringSecret memelord-arno-django]
        SS_db[StringSecret memelord-arno-database]
        SVC_minio[Service minio]
        DRAG[Dragonfly memelord-arno-redis]
        CNPG[CNPG Cluster memelord-arno-database]
        DB[(Database memelord-arno)]
        POL[Policy memelord-arno-policy]
        S3U[S3User memelord-arno-bucket]
        BUCK[Bucket memelord-arno]
        DEP[Deployment memelord-arno]
        SVC_ml[Service memelord]
        CERT_ml[Certificate memelord-arno]
        ING_ml[Ingress memelord-arno]
        OIDC_ml[OIDCClient memelord-arno]
    end

    subgraph grafana["grafana.yaml"]
        CM[ConfigMap grafana-datasources]
        STS[StatefulSet grafana]
        SVC_gr[Service grafana]
        CERT_gr[Certificate grafana-arno]
        ING_gr[Ingress grafana-arno]
        OIDC_gr[OIDCClient grafana-arno]
    end

    subgraph external["Välised (klaster)"]
        MINIO[minio/default]
        LE[letsencrypt]
        PM[passmower]
        MON[monitoring]
    end

    SS_redis --> DRAG
    SS_django --> DEP
    SS_db --> CNPG
    CNPG --> DB
    POL --> S3U
    S3U --> BUCK
    S3U --> DEP
    BUCK --> DEP
    MINIO --> POL
    MINIO --> S3U
    MINIO --> BUCK
    DRAG --> DEP
    DB --> DEP
    OIDC_ml --> DEP
    SVC_minio -.-> DEP
    DEP --> SVC_ml
    SVC_ml --> ING_ml
    CERT_ml --> ING_ml
    LE --> CERT_ml
    LE --> CERT_gr

    CM --> STS
    OIDC_gr --> STS
    STS --> SVC_gr
    SVC_gr --> ING_gr
    CERT_gr --> ING_gr
    MON -.-> CM
```

## Deploy (manuaalne)

```bash
kubectl create namespace memelord-arno

# MemeLord
kubectl apply -n memelord-arno -f memelord.yaml
kubectl create configmap settings --from-file=settings.py=./settings.py -n memelord-arno --dry-run=client -o yaml | kubectl apply -f -

# Grafana
kubectl apply -n memelord-arno -f grafana.yaml

kubectl rollout restart deployment/memelord-arno -n memelord-arno
```

**Grafana kasutaja Admin’iks:** `./grafana-promote-user.sh memelord-arno arno.kender@gmail.com`

## ArgoCD

- **ArgoCD** jookseb **ee-west-1** klustris; **rakendused** deploy’itakse **ee-lte-1** klustrisse.
- **Repo:** https://github.com/alfakilo87/memelord-arno  
- **UI:** https://argocd.ee-west-1.codemowers.io/applications  
- **Application:** name = `memelord-arno`, project = `default`, path = `.`, destination = ee-lte-1, namespace = `memelord-arno`.
