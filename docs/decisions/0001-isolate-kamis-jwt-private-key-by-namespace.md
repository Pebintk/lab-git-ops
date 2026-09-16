# 0001 — Isolate profile's JWT private key by namespace

**Date:** 2026-09-16
**Status:** Accepted
**Scope:** `lab-gitops`, namespaces `kamis` and `kamis-profile`

## Context

KAMIS uses an RS256 keypair. `profile` signs access tokens with the private key;
all six other services verify them with the public key. Seven services, one
signer.

Both keys reach the cluster through External Secrets Operator, which projects
them into Kubernetes Secrets. A Kubernetes Secret is **namespace-scoped, not
pod-scoped**. That distinction is the whole decision.

Two ExternalSecrets sitting in one namespace do not isolate anything. Any pod in
that namespace can mount the private-key Secret as a volume and read it, and
`resourceNames` on the Secret does not prevent that — RBAC gates reads through
the API server, and a kubelet mount does not go through the API server. The only
in-cluster boundary that a mount cannot cross is the namespace.

So the question was never "which Secret does profile get"; it was "whose
namespace does the private key live in".

## Decision

`profile` runs in its own namespace, `kamis-profile`. The five verifying
services — `asset`, `finance`, `project`, `purchase`, `resource` — run in `kamis`.

The private-key ExternalSecret is created only in `kamis-profile`. The public key
is created in both namespaces, because `secretKeyRef` is namespace-local and
profile needs to verify tokens as well as issue them.

Every peer URL in the overlays is therefore a fully qualified
`<service>-app.<namespace>.svc.cluster.local` name rather than a short one.

## Options considered

**Separate namespace** — chosen. A real boundary, enforced by Kubernetes itself
rather than by policy that has to be installed and maintained. Costs one
namespace and makes peer URLs verbose. Simplest to defend.

**Admission policy** (ValidatingAdmissionPolicy or Kyverno) — rejected for now.
Realistic, and closer to what a production cluster would do, but it needs a
policy engine on the cluster and it is only as good as the policy. It also
guards the *reference* to the Secret name rather than the mount, so it is a
different kind of control, not a stronger version of the namespace.

Combining both is the strongest option and remains open. The namespace boundary
does not obstruct adding a policy later; it is the layer that would still be
there if the policy were misconfigured.

**RBAC `resourceNames` on the Secret** — rejected outright. It does not prevent
a same-namespace pod from mounting the Secret, so it would be a control that
looks like isolation and is not. Claiming it as isolation is worse than having
no isolation, because it stops anyone looking further.

## Consequences

- Six services, two namespaces. Cluster-wide resources (`ClusterSecretStore`,
  CRDs, Postgres) are unaffected.
- Every cross-boundary URL is explicit. `profile` is the only service whose peers
  all live on the far side of the boundary.
- The public-key ExternalSecret exists twice verbatim, differing only in
  `metadata.namespace`. This is duplication by necessity, not an oversight:
  Kubernetes has no way to project one Secret into two namespaces.
- Reversal is cheap but not free: move `profile`'s overlay to `namespace: kamis`,
  delete the `kamis-profile` namespace and its two ExternalSecrets, and shorten
  the peer URLs. Nothing else depends on the split.
- An admission policy restricting the private Secret's name to profile's
  ServiceAccount is the natural second control, and is not built. See the "Not
  done here" section of `manifests/kamis/README.md`.

## Not claimed

The public key is not protected and does not need to be — it is handed to every
service and anyone can verify with it. Only `kamis-jwt-private` is the subject of
this decision.
