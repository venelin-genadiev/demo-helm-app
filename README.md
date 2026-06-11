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
  --create-namespace
```

The default Argo CD username is `admin`. Get the initial admin password with:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

## App Chart

Install the Death Metal app chart after the shared add-ons are running:

```bash
helm upgrade --install death-metal ./helm/app
```

The app chart currently packages the existing manifests from `app/cluster` as-is, without values overrides. It includes the `death-metal` namespace manifest, so the first install creates the namespace and app resources together.

If the `death-metal` namespace already exists and was not created by this Helm release, delete it first or remove the namespace manifest from the chart. Helm cannot adopt an existing namespace unless it already has Helm ownership labels and annotations.

## Verify

```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n kube-system
kubectl get pods -n argocd
kubectl top nodes
kubectl get all -n death-metal
kubectl get ingress -n death-metal
```
