# Kargo + Argo CD Helm demo

This repository deploys a small NGINX Helm chart to `dev`, `stage`, and `prod`.
Kargo discovers NGINX image versions and promotes the selected image tag through
three Argo CD Applications.

## Layout

- `helm/demo`: chart and environment-specific values.
- `argocd`: AppProject and three Applications.
- `kargo`: Project, Warehouse, and promotion Stages.

## Configure

The configured source is `https://github.com/aadi308/Kargo-ArgoCD.git`. Keep the
URL identical in `argocd/*.yaml` and `kargo/*.yaml` because Kargo uses it to
select the Argo CD source to update.

Argo CD and Kargo (including their CRDs) must already be installed. For a private
repository, configure repository credentials independently in Argo CD. This demo
does not commit back to Git, so Kargo needs registry access but no Git write token.

## Validate and install

```sh
helm lint helm/demo
helm template demo helm/demo -f helm/demo/values-dev.yaml >/dev/null
helm template demo helm/demo -f helm/demo/values-stage.yaml >/dev/null
helm template demo helm/demo -f helm/demo/values-prod.yaml >/dev/null

kubectl apply -f kargo/project.yaml
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/applications.yaml
kubectl apply -f kargo/warehouse.yaml
kubectl apply -f kargo/stages.yaml
```

Apply the Kargo `Project` first because it creates the `kargo-demo` namespace.
Apply the Argo CD `AppProject` before its Applications. `CreateNamespace=true`
allows Argo CD to create each workload namespace.

## Promotion flow

1. The `nginx` Warehouse discovers images matching `^1.27.0`.
2. Freight is promoted directly to `dev`, then from `dev` to `stage`, and from
   `stage` to `prod`.
3. Each promotion adds/updates Argo CD's `image.tag` Helm parameter and triggers
   a sync. The values file remains the environment baseline; the promoted Helm
   parameter has higher precedence.

The authorization annotation on every Application is required for Kargo's
`argocd-update` step.

## Argo CD sync troubleshooting

```sh
argocd app get demo-dev --show-operation
argocd app manifests demo-dev
argocd app diff demo-dev
kubectl -n argocd describe application demo-dev
kubectl -n argocd logs deploy/argocd-application-controller --since=15m
```

Common causes addressed here are missing destination namespaces, an AppProject
that rejects the repository/destination, inconsistent repository URLs, missing
Kargo authorization annotations, and orphaned resources. Automated sync, prune,
self-heal, namespace creation, and prune-last are enabled. If an Application is
still failing, its exact `status.conditions` and controller error are needed to
diagnose cluster-specific RBAC, repository credentials, CRDs, or admission rules.

Note: this simple demo lets Kargo update the live Application's Helm parameter.
For stricter GitOps/auditing, the next iteration should have Kargo update the
environment values file on a stage branch, commit it, and promote that commit. 
