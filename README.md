# cluster-observability

Prometheus + Grafana + Alertmanager stack for the EKS cluster, deployed via
ArgoCD alongside [`gitops-manifests`](https://github.com/David-Uka/gitops-manifests).

## What's here

- `helm/` — values overlays for [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack),
  layered as `values-base.yaml` + `values-<env>.yaml` (dev / staging / prod).
- `manifests/` — Grafana dashboards (as ConfigMaps, picked up by the Grafana
  sidecar) and Prometheus alert rules (`PrometheusRule` CRDs), built with Kustomize
  per environment.
- `dashboards/` and `alerts/` — the raw dashboard JSON / alert rule source,
  copied into `manifests/base/` as Kubernetes objects (single source of truth
  is the raw file; run a diff before editing the copy directly).
- `argocd/` — Application manifests. One `app-of-apps` bootstrap, plus per-env
  Applications: `kube-prometheus-stack-<env>` (the chart) and
  `cluster-observability-extras-<env>` (dashboards + alerts).

## Deploying

This repo doesn't deploy itself — apply `argocd/app-of-apps.yaml` once against
a cluster that has ArgoCD installed, and it manages the rest:

```
kubectl apply -f argocd/app-of-apps.yaml
```

Each `kube-prometheus-stack-<env>` Application uses ArgoCD's multi-source
feature: chart from the `prometheus-community` Helm repo, values from this git
repo. Bump `targetRevision` (chart version) deliberately in
`argocd/apps/kube-prometheus-stack-*.yaml` — it's pinned, not floating on latest.

`prod` has `selfHeal: false`. Prometheus config changes in prod should go
through a PR + manual sync in ArgoCD, not silently auto-revert/auto-apply.

## Adding a dashboard or alert

1. Drop the dashboard JSON in `dashboards/`, or add a rule group in
   `alerts/node-and-workload-rules.yaml`.
2. Wrap/copy it into `manifests/base/` as a `ConfigMap` (dashboard, labeled
   `grafana_dashboard: "1"`) or `PrometheusRule` (alert).
3. Open a PR — CI runs `helm template` + `kustomize build` through
   `kubeconform` for all three environments.

## Local validation

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm template kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 65.5.1 -f helm/values-base.yaml -f helm/values-dev.yaml | less

kustomize build manifests/overlays/dev | less
```
