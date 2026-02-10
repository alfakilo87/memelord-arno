# memelord-arno

MemeLord (Django) deployment for Kubernetes – namespace `memelord-arno`.

- **URL:** https://memelord-arno.ee-lte-1.codemowers.io
- **ArgoCD:** Track this repo with Application name = `memelord-arno`, path `.`, project `default`.

## Repo contents

| File | Description |
|------|-------------|
| `failinimi.yaml` | Main manifest: StringSecrets, Dragonfly, CNPG, S3 Policy/Bucket, Deployment, Service, Certificate, Ingress, OIDCClient |
| `settings.py` | Django settings (ConfigMap source: `kubectl create configmap settings --from-file=settings.py=settings.py -n memelord-arno ...`) |
| `memelord-reset-password.sh` | Helper to reset app user password via pod |
| `minio-bucket-alias.yaml` | Optional: Service in `minio` namespace for S3 virtual-hosted DNS |

## Deploy (manual)

```bash
kubectl create namespace memelord-arno
kubectl apply -n memelord-arno -f failinimi.yaml
kubectl create configmap settings --from-file=settings.py=settings.py -n memelord-arno --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f minio-bucket-alias.yaml   # if present
kubectl rollout restart deployment/memelord-arno -n memelord-arno
```

## ArgoCD

- **UI:** https://argocd.ee-west-1.codemowers.io/applications  
- **Application:** name = `memelord-arno`, namespace = `memelord-arno`, project = `default`, path = `.`, destination cluster = (cluster API URL, e.g. `https://10.254.10.31:6443`).
