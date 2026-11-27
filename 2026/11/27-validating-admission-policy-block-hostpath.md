# Blocking `hostPath` Volumes with a ValidatingAdmissionPolicy

*Containers & Kubernetes security · 2026-11-27*

## What it is

`ValidatingAdmissionPolicy` (VAP, `admissionregistration.k8s.io/v1`, GA since 1.30) runs CEL expressions inside the API server, so you can enforce a rule like "no `hostPath` volumes" without deploying a webhook, Kyverno, or Gatekeeper. A policy holds the logic; a separate `ValidatingAdmissionPolicyBinding` decides where it applies and what happens on failure (`Deny`, `Warn`, `Audit`). Because there is no external service, there is no webhook outage to fail open or closed.

## Example

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: no-hostpath
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: "!has(object.spec.volumes) || object.spec.volumes.all(v, !has(v.hostPath))"
    message: "hostPath volumes are not allowed"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: no-hostpath-binding
spec:
  policyName: no-hostpath
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchExpressions:
      - {key: kubernetes.io/metadata.name, operator: NotIn, values: [kube-system]}
```

- `failurePolicy: Fail`: a broken expression rejects instead of silently allowing.
- `matchConstraints`: the policy only evaluates Pod create/update.
- `object.spec.volumes.all(...)`: CEL macro; `has()` guards against the field being absent.
- `validationActions: [Deny]`: start with `[Warn, Audit]` to see who breaks, then flip to `Deny` (`Deny` and `Warn` can't be combined).
- `namespaceSelector`: exempts `kube-system`, where CNI/CSI DaemonSets legitimately mount the host.

## When to use it / why it matters

`hostPath` is one of the shortest paths from "can create pods" to node compromise: mount `/` or `/var/run/containerd/containerd.sock` and you own the host. Pod Security Admission's `baseline` level already bans it, but VAP lets you express narrower or custom rules (allowlist specific paths, read-only only, per-team exceptions) that PSA can't.

Gotchas: matching only `pods` means a bad Deployment is *accepted* and the failure shows up later as ReplicaSet events, so also match workload kinds (or check `object.spec.template.spec`) for fast feedback. Check `kubectl get vap no-hostpath -o yaml` for `status.typeChecking` warnings, and remember a policy with no binding does nothing. When pentesting, `kubectl auth can-i create pods` plus no hostPath control is a finding worth reporting.

## Sources

- [Kubernetes docs: Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [API reference: ValidatingAdmissionPolicyBinding v1](https://kubernetes.io/docs/reference/kubernetes-api/admissionregistration/validating-admission-policy-binding-v1/)
- [Kubernetes docs: Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
