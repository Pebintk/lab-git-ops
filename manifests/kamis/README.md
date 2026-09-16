# KAMIS services — Kustomize layout

Six Go services: `asset`, `finance`, `profile`, `project`, `purchase`, `resource`.
They are near-identical. Only four things differ: the name, the port, the
database, and which peer services each one calls.

`template` is not here. It is the scaffold service the other five were cut from —
`Makefile` defaults `SERVICE ?= template` — and its `.env.example` carries
`PORT=8085`, the same port as `resource`. It is a starting point for a new
service, not a service.

Source of truth for every value below is `kamis-be-go`: `pkg/config/config.go`,
`pkg/auth/jwt.go`, and each service's `.env.example`.

```
manifests/kamis/
├── base/                      one copy of the workload, fully parameterised
│   ├── config.yaml            service settings, injected with envFrom
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── kustomization.yaml
├── secrets/                   its own ArgoCD Application, wave -10
│   ├── namespaces.yaml        kamis + kamis-profile, wave -15
│   ├── externalsecret-public-kamis.yaml
│   ├── externalsecret-public-profile.yaml
│   ├── externalsecret-private.yaml
│   └── kustomization.yaml
└── overlays/dev/<service>/    ~20-line patch each
    ├── kustomization.yaml
    └── patch.yaml
```

## Namespaces

| Namespace | Services | Why |
|---|---|---|
| `kamis` | asset, finance, project, purchase, resource | The five that verify tokens |
| `kamis-profile` | profile | The one that issues them |

`profile` is separated because a Secret is namespace-scoped, not pod-scoped. Two
same-namespace ExternalSecrets give **no** isolation: any pod in the namespace can
mount `kamis-jwt-private` with a volume, and RBAC `resourceNames` on the Secret
does not stop it — that only gates API reads, not kubelet mounts. The namespace is
the boundary. This is not claimed anywhere as RBAC-based isolation, because it is
not.

The cost: profile's peers address it across the boundary, and it addresses them
the same way, so every peer URL in the overlays is a full
`<service>-app.<namespace>.svc.cluster.local` name.

## Why the overlays never patch `env`

`base/deployment.yaml` builds `DATABASE_URL` from two earlier `env` entries with
the `$(VAR)` syntax. Kubernetes expands that **only** against variables declared
earlier in the same list.

A strategic-merge patch that touches `env` re-emits the entries it touched ahead
of the ones it did not. An early version of the `asset` overlay patched
`DATABASE_URL`, which moved it above `POSTGRES_PASSWORD` and left the literal
string `$(POSTGRES_PASSWORD)` inside the DSN. Rendering the overlay is what caught
it; the manifest alone looked correct.

So, two rules:

1. Per-service settings live in the `<service>-config` ConfigMap and reach the pod
   through `envFrom`. Overlays patch the ConfigMap, never the container's `env`.
2. `JWT_SECRET_KEY` is in the base as a `secretKeyRef` marked `optional: true`, so
   profile can have it without any overlay adding to `env`.

Both exist only to keep rule 0: **no overlay patch names `env`.** If you ever must,
re-state the whole block in the patch with the order preserved, or have ESO
template the DSN instead of using `$(VAR)`.

## What each overlay sets

| Service | Port | Database | Peer URLs |
|---|---|---|---|
| profile | 8080 | kamis_profile | project, resource, asset, purchase |
| asset | 8081 | kamis_asset | finance |
| finance | 8082 | kamis_finance | project, purchase |
| project | 8083 | kamis_project | profile, resource, asset, finance |
| purchase | 8084 | kamis_purchase | profile, resource, asset, finance |
| resource | 8085 | kamis_resource | — |

`asset` and `purchase` also mount an `emptyDir` for uploads. See their patches.

Two values that must stay equal in each overlay, or vmagent silently drops the
pod: `prometheus.io/port` and the container's `containerPort`. The scrape rule in
`manifests/monitoring/victoria-metrics/base/scrape-config.yaml` keeps only the
container port that equals the annotation.

## Not done here

- **Images do not exist yet.** `newTag: dev` is a placeholder. Pods will sit in
  `ImagePullBackOff` until the images are built and pushed to
  `us-central1-docker.pkg.dev/ops-lab-506804/lab-images/kamis-<service>`. The tag
  should become the built commit SHA, the way `lab-app`'s overlay carries
  `ed62660`.
- **`GCS_BUCKET` is unset** for asset and purchase, so uploads go to an emptyDir
  and do not survive a restart. Making them survive needs Workload Identity: a
  Google service account, an `iam.workloadIdentityUser` binding, and the
  `iam.gke.io/gcp-service-account` annotation on the ServiceAccount here.
- **Resource requests and limits are placeholders.** No KAMIS service has run on
  this cluster, so the numbers are copied from `lab-app`'s shape rather than
  measured. Replace them once the memory-ratio dashboard has real data.
- **`FRONTEND_URL` is `http://localhost:5173`** in the base. Only CORS reads it,
  and only for the browser; peer services never send an Origin header.
