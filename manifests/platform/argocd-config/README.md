# argocd-cm — source of truth, applied by hand

`configmap.yaml` here is the full, intended content of the `argocd-cm` ConfigMap on
`prod-cluster`. It is **not** wired into an ArgoCD Application — `bootstrap.sh` installs
ArgoCD by raw-applying the upstream `install.yaml` once, and ArgoCD does not self-manage
its own control-plane config in this lab. This directory exists so that config has a
source of truth at all, since it was previously hand-edited on the cluster with none.

## Applying a change

```bash
kubectl --context prod apply -f manifests/platform/argocd-config/configmap.yaml
```

`kubectl apply` is safe for *adding or changing* keys here. It is **not** reliable for
*removing* one — with no prior `last-applied-configuration` on an object (true here,
since it was never previously applied), apply's three-way merge can't distinguish "never
mentioned" from "intentionally deleted," and may leave a stale key behind. To guarantee a
key's removal, patch it out explicitly with an untyped `null`, then let this file describe
the resulting state:

```bash
kubectl --context prod -n argocd patch configmap argocd-cm --type merge \
  -p '{"data":{"resource.customizations.ignoreResourceUpdates.all": null}}'
```

## Why `ignoreResourceUpdates.all: /status` was removed

It's a scale-tuning pattern for clusters running thousands of Applications, where
status-only updates on hot resources can flood the reconcile queue. At this lab's scale
that problem doesn't exist, and the rule has a sharp edge: any CRD whose *health* is
derived from its own `.status` (`ClusterSecretStore`, `ExternalSecret`, cert-manager
`Certificate`) can get stuck reporting whatever health it had the moment the rule started
ignoring further updates — because the update that would prove "healthy now" is a status
change, and status changes are exactly what's suppressed. `cluster-secret-store` hit this
exactly: `Degraded` frozen since first failure, while the live object had already
recovered and `reconciledAt` kept advancing underneath the stale health.
