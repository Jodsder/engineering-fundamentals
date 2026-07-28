# Deployment (Part E - GitOps)

```
deploy/
├── argocd/                     # Helm chart that installs Argo CD itself (bootstrap, run by hand)
├── ipt-lr-eng/                 # Helm chart for the React app - this is what Argo CD watches
└── argocd-applications/        # Argo CD Application resources (bootstrap, applied by hand)
    └── ipt-lr-eng.yaml
```

Two things are applied manually, once: the Argo CD install and the `Application`
resource. Everything after that is driven by Git.

## 1. Install Argo CD

See [argocd/README.md](argocd/README.md).

## 2. Register the application

```bash
kubectl apply -f deploy/argocd-applications/ipt-lr-eng.yaml
```

Argo CD now watches `deploy/ipt-lr-eng` on `main` and reconciles it into the
`user` namespace.

## The loop

Change anything under `deploy/ipt-lr-eng/` — the image tag, the replica count,
a template — commit, push to `main`, done. Argo CD polls Git every 3 minutes
(`timeout.reconciliation` in the Argo CD chart) and rolls the change out.

`syncPolicy.automated` is configured with:

* `prune: true` — resources deleted from Git are deleted from the cluster.
* `selfHeal: true` — changes made directly against the cluster (`kubectl edit`,
  `kubectl scale`) get reverted back to what Git says.

To skip the polling interval and sync immediately:

```bash
kubectl -n argocd patch app ipt-lr-eng --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"main"}}}'
# or: argocd app sync ipt-lr-eng
```

## Reaching the app

The service is `ClusterIP`, so it is not exposed publicly:

```bash
kubectl port-forward -n user svc/ipt-lr-eng 3000:80
# http://localhost:3000
```

Set `service.type: LoadBalancer` in [ipt-lr-eng/values.yaml](ipt-lr-eng/values.yaml)
to get a public IP from Azure instead.

## Checking status

```bash
kubectl get application -n argocd ipt-lr-eng
kubectl get pods,svc -n user
```
