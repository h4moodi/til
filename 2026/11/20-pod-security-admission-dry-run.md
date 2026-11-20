# Rolling Out Pod Security Admission with Server-Side Dry-Run
*Containers & Kubernetes security · 2026-11-20*

## What it is

Pod Security Admission (PSA) is the built-in admission controller that enforces the Pod Security Standards (`privileged`, `baseline`, `restricted`) per namespace via labels. Each namespace can set three independent modes: `enforce` (reject), `audit` (annotate the audit event) and `warn` (return a warning to the client). Before flipping `enforce`, you can ask the API server which *already running* pods would violate a level by applying the label with `--dry-run=server`.

## Example

```bash
# Preview violations across every namespace without changing anything
kubectl label --dry-run=server --overwrite ns --all \
  pod-security.kubernetes.io/enforce=restricted

# Roll out safely: enforce baseline, pin the version, warn+audit on restricted
kubectl label --overwrite ns payments \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=v1.34 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

- `--dry-run=server`: the label change goes through admission but isn't persisted, so PSA evaluates existing pods and prints `Warning: existing pods in namespace "x" violate the new PodSecurity enforce level ...` with the offending pods and fields.
- `--all` / `--overwrite`: check every namespace and replace any existing labels.
- `enforce-version`: pins the policy to a Kubernetes minor version so a cluster upgrade can't tighten the rules under you (default is `latest`).
- `warn`/`audit=restricted`: shows the next tightening step without breaking deploys.

## When to use it / why it matters

Defensively, this is the cheapest way to get from "everything privileged" to `restricted` without an outage: dry-run, fix the listed pods, enforce `baseline`, then ratchet up. For a pentester or reviewer, `kubectl get ns -L pod-security.kubernetes.io/enforce` quickly finds namespaces with no level or `privileged`, which is where `hostPath`, `hostPID` or privileged containers can still be scheduled.

Gotchas:
- `enforce` only checks **Pod** objects. A Deployment with a bad template is accepted, then its ReplicaSet silently fails to create pods. `warn` and `audit` *do* fire on workload templates, so always set them alongside `enforce`.
- Enforcing doesn't evict running pods; violations only block new or updated ones.
- Cluster-wide defaults and exemptions (usernames, runtime classes, namespaces) live in the API server's `AdmissionConfiguration`, not labels, so check it for broad exemptions.

## Sources

- [Enforce Pod Security Standards with Namespace Labels (kubernetes.io)](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Pod Security Admission (kubernetes.io)](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Apply Pod Security Standards at the Cluster Level (kubernetes.io)](https://kubernetes.io/docs/tutorials/security/cluster-level-pss/)
