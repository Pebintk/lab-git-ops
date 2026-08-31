# lab-gitops

The Git source of truth for everything running inside the `lab` GKE cluster.

This is one repo in a four-repo lab:

| Repo | Owns |
|---|---|
| `lab-infra/` | Terraform: GKE cluster, Jenkins VM, Artifact Registry, IAM |
| `lab-ansible/` | OS config of the Jenkins VM |
| `lab-app/` | Go source, Dockerfile, Jenkinsfile — produces a tagged image |
| **`lab-gitops/`** | **this repo — desired state of the cluster** |

**Boundary:** Terraform owns the cluster and everything below it (nodes, networking).
This repo owns everything inside it, via the Kubernetes API. It contains no application
source, no Dockerfile, and no Terraform — it only references images that already exist in
Artifact Registry.

After the ArgoCD `Application` is bootstrapped, the only way anything reaches the cluster is
via a Git commit to this repo. No `kubectl apply` of manifests directly.

## Where the image tag lives

`manifests/lab-app/overlays/dev/kustomization.yaml` — the `newTag` field is the single point
of change for a deployment. Edit that line, commit, push. ArgoCD detects the diff and rolls
the new pods; nothing else needs to change.

## Secrets

No secret value is in this repo, and none can be. `manifests/lab-app/base/externalsecret.yaml`
holds a *reference*; External Secrets Operator authenticates to GCP Secret Manager and creates
the Kubernetes Secret `lab-app-secret` in the cluster. Rotation happens in Secret Manager —
changing a value there reaches the pods within `refreshInterval` with no commit and no `kubectl`.

That is the property Sealed Secrets does not have. Sealed Secrets commits ciphertext, so
rotation means re-encrypting and re-committing; the encrypted material is still in Git history.
(For a homelab with no cloud secret store, Sealed Secrets is still the right answer.)

Auth is Workload Identity, in two halves that only work together:

| Half | Where |
|---|---|
| `iam.gke.io/gcp-service-account` on the ESO KSA | `apps/external-secrets.yaml` |
| `roles/iam.workloadIdentityUser` on the Google SA | `lab-infra/platform/external-secrets.tf` |

With one missing, Secret Manager returns 403 and the ExternalSecret reports `SecretSyncedError`.
There is no JSON key anywhere.

## Sync waves

`apps/` Applications are ordered by `argocd.argoproj.io/sync-wave`:

| Wave | Application | Why |
|---|---|---|
| `-10` | `external-secrets`, `ingress-nginx` | install CRDs and the nginx IngressClass |
| `-5` | `cluster-secret-store` | a CRD *instance*; needs the CRDs to exist |
| `0` | `lab-app` | its ExternalSecret needs both the CRDs and the store |

Without the ordering, the first sync of a fresh cluster fails with
`no matches for kind "ExternalSecret"` — the API server rejects a resource whose CRD has not
been registered yet. ArgoCD waits for each wave to report Healthy before starting the next.

## Ingress

Exactly **one** LoadBalancer in the cluster: the ingress-nginx controller. Every additional
`type=LoadBalancer` Service or GCE Ingress provisions its own billable GCP forwarding rule
(~$18/month), so everything else routes through this one.

The hostname is `lab.<dashed-ip>.nip.io` — nip.io is wildcard DNS that resolves an IP embedded
in the hostname back to that IP, free and with no zone to manage. It is stable across cluster
rebuilds only because the address is reserved in `lab-infra/bootstrap`.

**ArgoCD is deliberately not exposed.** It has cluster-wide write access, and putting it behind
the ingress controller it manages creates a loop: break the controller and you lose the UI you
would use to fix it. Access stays on `kubectl port-forward`.

## Bootstrap

```bash
kubectl kustomize manifests/lab-app/overlays/dev   # render and inspect before applying
kubectl apply -f apps/root.yaml                     # one-time: root app-of-apps
```

## Access

```bash
kubectl -n dev port-forward svc/lab-app 8082:80     # the app
kubectl -n argocd port-forward svc/argocd-server 8081:443   # the ArgoCD UI
```
