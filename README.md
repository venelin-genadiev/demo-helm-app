# Helm

This folder contains the application Helm chart. Shared cluster add-ons are installed separately because they are reused by every app in the cluster.

## Repositories

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

## Shared Cluster Add-ons

Install ingress-nginx once per cluster. App charts can then create `Ingress` resources with `ingressClassName: nginx`.

Latest stable pin: ingress-nginx chart `4.15.1`, controller `v1.15.1`.

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.15.1 \
  -n ingress-nginx \
  --create-namespace
```

Install metrics-server once per cluster. App HPAs use the Kubernetes metrics API that it provides.

Latest installable pin from the Helm repo index: metrics-server chart `3.13.0`, metrics-server `v0.8.0`.

### Local Metrics Server

For local clusters like minikube, install metrics-server with insecure kubelet TLS enabled:

```bash
helm upgrade --install metrics-server metrics-server/metrics-server \
  --version 3.13.0 \
  -n kube-system \
  --set "args[0]=--kubelet-insecure-tls"
```

### Production Metrics Server

For production, do not use `--kubelet-insecure-tls` unless the cluster explicitly requires it:

```bash
helm upgrade --install metrics-server metrics-server/metrics-server \
  --version 3.13.0 \
  -n kube-system
```

Install Argo CD once per cluster to manage GitOps-based application deployments.

```bash
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  --create-namespace \
  -f argo/argocd-values.yaml
```

Create the dev and prod Argo CD Applications after Argo CD is installed:

```bash
helm upgrade --install argocd-apps argo/argocd-apps \
  -n argocd \
  -f argo/values.yaml
```

The `argo/argocd-values.yaml` file configures the `argo/argo-cd` chart for
local development (HTTP, `argo.example.com`). Cloud deployments use
`argo/argocd-values-tls.yaml` (HTTPS via cert-manager). The `argo/values.yaml`
file configures the `argo/argocd-apps` chart.

The default Argo CD username is `admin`. Get the initial admin password with:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

With ingress-nginx installed and `127.0.0.1 argo.example.com` in `/etc/hosts`,
open Argo CD at `http://argo.example.com`.

Alternatively, port-forward the Argo CD server locally, then open
`https://localhost:8080`:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd \
  8080:443
```

## Ingress TLS (cloud vs local)

The app chart supports optional TLS via cert-manager. Defaults in `app/values.yaml` keep TLS **disabled** so local clusters (minikube) work over plain HTTP:

```yaml
ingress:
  className: nginx
  host: ""
  tls:
    enabled: false
    secretName: ""
```

Cloud deployments override TLS in environment-specific values files:

- `app/values-dev.yaml` — `ingress.tls.enabled: true`, `secretName: death-metal-web-dev-tls`
- `app/values-prod.yaml` — `ingress.tls.enabled: true`, `secretName: death-metal-web-prod-tls`
- `argo/argocd-values-tls.yaml` — Argo CD HTTPS ingress (used by `infra/` and `infra-terraform/`)

When `tls.enabled` is `true`, the Ingress template adds:

- `cert-manager.io/cluster-issuer: letsencrypt-prod`
- a `tls:` block referencing `ingress.tls.secretName`

cert-manager must be installed in the cluster (see `infra/ansible/playbooks/05-dns-and-tls.yml`) and the ClusterIssuer name must match (`letsencrypt-prod`).

For Argo CD HTTPS in cloud, update hostnames in `argo/argocd-values-tls.yaml` to match your domain. `infra/` and `infra-terraform/` install Argo CD with that file automatically.
