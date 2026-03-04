# Runbook: AWS Dev ArgoCD OutOfSync/Missing Resources

Scope:

1. AWS dev cluster rebuild or restore scenarios.
2. Symptom: Argo child apps (`observability-baseline`, `pulsecart-workloads`) show `OutOfSync` and `Missing`.

## Trigger Signals

1. `argocd app list` shows child apps `OutOfSync/Missing`.
2. Namespaces such as `observability` or `pulsecart` do not exist.
3. Argo sync output includes missing kinds (`ExternalSecret`, `SecretStore`).

## First-Principles Checks

```bash
argocd app get external-secrets
argocd app get observability-baseline
argocd app get pulsecart-workloads

kubectl get crd externalsecrets.external-secrets.io secretstores.external-secrets.io clustersecretstores.external-secrets.io
kubectl get ns observability pulsecart
```

Interpretation:

1. If CRDs are missing, Argo cannot create dependent resources and child apps remain `Missing`.
2. If CRDs exist, inspect repo/path/revision mismatch in the failing `Application` spec.

## Recovery Steps

1. Ensure Argo bootstrap is complete:

```bash
/Users/lseino/triad-platform/triad-kubernetes-platform/scripts/bootstrap-argocd.sh
```

2. If CRDs are missing, render and apply external-secrets chart with CRDs:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm template external-secrets external-secrets/external-secrets \
  --version 0.20.4 \
  --namespace kube-system \
  --set installCRDs=true > /tmp/external-secrets-rendered.yaml

kubectl apply --server-side --force-conflicts -f /tmp/external-secrets-rendered.yaml
```

3. Re-sync in dependency order:

```bash
argocd app sync external-secrets-prereqs --prune --force
argocd app sync external-secrets --prune --force
argocd app sync observability-baseline --prune --force
argocd app sync pulsecart-workloads --prune --force
argocd app list
```

## Exit Criteria

1. `external-secrets`, `observability-baseline`, and `pulsecart-workloads` are `Synced` and `Healthy`.
2. `observability` and `pulsecart` namespaces exist with expected workloads.
3. Public health check responds from `api-gateway`.
