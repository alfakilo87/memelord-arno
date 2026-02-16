# memelord-arno

MemeLord (Django) + Grafana Helm chart. Mitu instantsi võimalik ({{ .Release.Name }}, values-*.yaml).

| Rakendus | URL |
|----------|-----|
| MemeLord | https://memelord-arno.ee-lte-1.codemowers.io |
| Grafana | https://grafana-arno.ee-lte-1.codemowers.io |

## Repo failid

| Fail | Kirjeldus |
|------|-----------|
| `Chart.yaml`, `values.yaml`, `values-memelord-arno.yaml`, `values-memelord-arno2.yaml` | Helm chart ja values |
| `templates/memelord.yaml`, `templates/grafana.yaml`, `templates/monitoring.yaml` | Templatiseeritud manifestid |
| `grafana-promote-user.sh` | Helper: tõsta OIDC kasutaja Grafana Admin’iks (SQLite DB muudatus) |

**Välised sõltuvused (klaster):** `minio/default`, `letsencrypt` ClusterIssuer, `passmower` (OIDC), `monitoring` namespace (Prometheus, Loki).

## Elementide sõltuvused (Mermaid)

```mermaid
flowchart TB
    subgraph memelord["memelord.yaml"]
        SS_redis["StringSecret<br/>(memelord-arno-redis)"]
        SS_django["StringSecret<br/>(memelord-arno-django)"]
        SS_db["StringSecret<br/>(memelord-arno-database)"]
        SVC_minio["Service<br/>(minio)"]
        DRAG["Dragonfly<br/>(memelord-arno-redis)"]
        CNPG["CNPG Cluster<br/>(memelord-arno-database)"]
        DB[("Database<br/>(memelord-arno)")]
        POL["Policy<br/>(memelord-arno-policy)"]
        S3U["S3User<br/>(memelord-arno-bucket)"]
        BUCK["Bucket<br/>(memelord-arno)"]
        DEP["Deployment<br/>(memelord-arno)"]
        SVC_ml["Service<br/>(memelord)"]
        CERT_ml["Certificate<br/>(memelord-arno)"]
        ING_ml["Ingress<br/>(memelord-arno)"]
        OIDC_ml["OIDCClient<br/>(memelord-arno)"]
    end

    subgraph grafana["grafana.yaml"]
        CM["ConfigMap<br/>(grafana-datasources)"]
        STS["StatefulSet<br/>(grafana)"]
        SVC_gr["Service<br/>(grafana)"]
        CERT_gr["Certificate<br/>(grafana-arno)"]
        ING_gr["Ingress<br/>(grafana-arno)"]
        OIDC_gr["OIDCClient<br/>(grafana-arno)"]
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

    classDef stringSecret fill:#f5f5f5,stroke:#424242,color:#1a1a1a
    classDef service fill:#bbdefb,stroke:#0d47a1,color:#0d47a1
    classDef dragonfly fill:#c8e6c9,stroke:#1b5e20,color:#1b5e20
    classDef cnpg fill:#b2dfdb,stroke:#004d40,color:#004d40
    classDef s3 fill:#ffe0b2,stroke:#bf360c,color:#bf360c
    classDef workload fill:#e1bee7,stroke:#4a148c,color:#4a148c
    classDef configmap fill:#fff9c4,stroke:#f57f17,color:#5d4037
    classDef certificate fill:#c5e1a5,stroke:#33691e,color:#33691e
    classDef ingress fill:#ffcdd2,stroke:#b71c1c,color:#b71c1c
    classDef oidc fill:#b3e5fc,stroke:#01579b,color:#01579b
    classDef external fill:#cfd8dc,stroke:#263238,color:#263238

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

    linkStyle default stroke:#2d3748,stroke-width:2
    style memelord fill:#e2e8f0,stroke:#4a5568,color:#1a202c
    style grafana fill:#e2e8f0,stroke:#4a5568,color:#1a202c
    style external fill:#e2e8f0,stroke:#4a5568,color:#1a202c
```

## Deploy (Helm)

```bash
kubectl create namespace memelord-arno

# MemeLord
kubectl create configmap settings --from-file=settings.py=./settings.py -n memelord-arno --dry-run=client -o yaml | kubectl apply -f -
helm upgrade --install memelord-arno . -n memelord-arno -f values-memelord-arno.yaml
```

**Grafana kasutaja Admin’iks:** `./grafana-promote-user.sh memelord-arno arno.kender@gmail.com`

## ArgoCD

- **ArgoCD** jookseb **ee-west-1** klustris; **rakendused** deploy’itakse **ee-lte-1** klustrisse.
- **Repo:** https://github.com/alfakilo87/memelord-arno  
- **UI:** https://argocd.ee-west-1.codemowers.io/applications  
- **memelord-arno:** `argocd-application-memelord-arno.yaml` | **memelord-arno2:** `argocd-application-memelord-arno2.yaml`
