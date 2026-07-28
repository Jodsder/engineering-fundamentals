# Argo CD

Helm chart that installs Argo CD on the AKS cluster. It is a thin wrapper around the
upstream [`argo-cd`](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)
chart — `Chart.yaml` pins the dependency version, `values.yaml` holds our configuration.

| | |
|---|---|
| Cluster | `aks-lernreise-engineering-jos` (RG `engineering-lernreise-jos`) |
| Namespace | `argocd` |
| Release name | `argocd` |
| Argo CD version | v3.4.5 (chart 10.2.1) |

## Prerequisites

```bash
az aks get-credentials -g engineering-lernreise-jos -n aks-lernreise-engineering-jos
```

## Install / upgrade

```bash
cd deploy/argocd
helm dependency update .                     # fetches charts/argo-cd-10.2.1.tgz
helm upgrade --install argocd . \
  --namespace argocd --create-namespace \
  --wait --timeout 10m
```

Preview the rendered manifests without touching the cluster:

```bash
helm template argocd . -n argocd
```

## Access the UI

The server is a `ClusterIP` service, so it is not reachable from outside the cluster.
Port-forward to reach it locally:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Then open <https://localhost:8080> and accept the self-signed certificate.

Username is `admin`. The bootstrap password is generated at install time:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

Change it after the first login (`argocd account update-password`) and delete the
secret — Argo CD only keeps it for bootstrapping.

## What is configured

Deliberately minimal, so this is *not* a production setup:

* Non-HA — one replica of each component, no autoscaling, no Redis HA.
* No ingress — access via port-forward only.
* No SSO — Dex is disabled and the local `admin` account is used.
* Notifications controller disabled.
* CRDs are installed by the chart and kept on uninstall (`crds.keep: true`), so
  `Application` / `AppProject` objects survive a `helm uninstall`.

## Uninstall

```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
# CRDs are intentionally left behind; remove them explicitly if you really want to:
# kubectl delete crd applications.argoproj.io applicationsets.argoproj.io appprojects.argoproj.io
```
