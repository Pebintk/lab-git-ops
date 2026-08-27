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

## Bootstrap

```bash
kubectl kustomize manifests/lab-app/overlays/dev   # render and inspect before applying
kubectl apply -f apps/lab-app.yaml                  # one-time: registers the Application
```

## Access

```bash
kubectl -n dev port-forward svc/lab-app 8082:80     # the app
kubectl -n argocd port-forward svc/argocd-server 8081:443   # the ArgoCD UI
```
