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

    classDef stringSecret fill:#f5f5f5,stroke:#9e9e9e
    classDef service fill:#bbdefb,stroke:#1976d2
    classDef dragonfly fill:#c8e6c9,stroke:#388e3c
    classDef cnpg fill:#b2dfdb,stroke:#00796b
    classDef s3 fill:#ffe0b2,stroke:#e65100
    classDef workload fill:#e1bee7,stroke:#7b1fa2
    classDef configmap fill:#fff9c4,stroke:#f9a825
    classDef certificate fill:#c5e1a5,stroke:#558b2f
    classDef ingress fill:#ffcdd2,stroke:#c62828
    classDef oidc fill:#b3e5fc,stroke:#0277bd
    classDef external fill:#cfd8dc,stroke:#455a64

    class SS_redis,SS_django,SS_db stringSecret
    class SVC_minio,SVC_ml,SVC_gr service
    class DRAG dragonfly
    class CNPG,DB cnpg
    class POL,S3U,BUCK s3
    class DEP,STS workload
    class CM configmap
    class CERT_ml,CERT_gr certificate
    class ING_ml,ING_gr ingress
    class OIDC_ml,OIDC_gr oidc
    class MINIO,LE,PM,MON external
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
