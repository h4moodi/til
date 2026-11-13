# Pod User Namespaces with `hostUsers: false`

*Containers & Kubernetes security · 2026-11-13*

## What it is

Setting `hostUsers: false` on a Pod gives it its own Linux user namespace: UID 0 inside the container is mapped to an unprivileged, per-pod range of 65,536 high UIDs on the node. Capabilities granted inside the pod (even `CAP_SYS_ADMIN` or `CAP_NET_ADMIN`) only act on namespaced resources, not the host. The feature went GA in Kubernetes v1.36 and needs no image changes.

## Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: userns-demo
spec:
  hostUsers: false          # opt in: new user namespace for the whole pod
  containers:
  - name: app
    image: debian
    command: ["sleep", "infinity"]
    securityContext:
      runAsUser: 0          # root in the container, NOT root on the node
```

Verify it took effect:

```bash
kubectl exec userns-demo -- cat /proc/self/uid_map
#          0  833617920      65536
# ^in-pod UID  ^host UID start  ^range size (must be 65536)
kubectl exec userns-demo -- readlink /proc/self/ns/user   # differs from the host's
```

If `uid_map` shows `0 0 4294967295`, the pod is still in the host user namespace.

## When to use it / why it matters

- **Defensive:** it cuts the blast radius of a container breakout. A process that escapes via a kernel bug or a bad mount lands on the node as a nobody UID, so it can't read root-owned host files or touch other pods' files (each pod gets a non-overlapping range). Several past HIGH-severity escapes, such as CVE-2021-25741 (symlink host file access), were not exploitable with it enabled.
- **Good default for:** workloads that "need root" or extra caps (build tools, legacy images, network sidecars) where you can't easily drop privileges.
- **Gotchas:** incompatible with `hostNetwork`, `hostPID` and `hostIPC`. Volumes rely on idmapped mounts, so the node kernel, filesystem and runtime must support them (recent kernel; tmpfs-backed Secrets/ConfigMaps need idmap support for tmpfs; a userns-aware containerd/CRI-O plus runc/crun). Check node support before rolling out, or pin these pods to capable nodes.
- **Not a sandbox:** the kernel is still shared, so a kernel exploit is still a kernel exploit. Pair it with seccomp, dropped capabilities, and a Pod Security Admission or admission-policy rule that requires `hostUsers: false` for untrusted namespaces.
- **Pentest angle:** when assessing a cluster, `cat /proc/self/uid_map` from a foothold quickly tells you whether "root in the pod" is worth anything on the node.

## Sources

- [Kubernetes v1.36: User Namespaces in Kubernetes are finally GA](https://kubernetes.io/blog/2026/04/23/kubernetes-v1-36-userns-ga/)
- [Use a User Namespace With a Pod (k8s docs)](https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/)
- [KEP-127: Support User Namespaces](https://www.kubernetes.dev/resources/keps/127/)
- [Edera: Kubernetes finally has user namespace support, the shared kernel problem remains](https://edera.dev/stories/kubernetes-finally-has-user-namespace-support-the-shared-kernel-problem-remains)
