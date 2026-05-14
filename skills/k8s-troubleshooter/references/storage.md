# Storage & Persistence

Storage failures in Kubernetes almost always show up as pods stuck in ContainerCreating or Pending. Events explain why.

```bash
kubectl describe pvc <pvc-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
# Events section is the key diagnostic
```

---

## PVC Stuck in Pending

The PVC can't find or create a matching volume.

**Most common cause: StorageClass mismatch**

```bash
kubectl describe pvc <pvc-name>
# Look for events like:
# storageclass.storage.k8s.io "fast-ssd" not found
# failed to provision volume with StorageClass "standard": no available volumes
```

Checks:
```bash
# List available StorageClasses
kubectl get storageclass

# Check which is default (look for "(default)" annotation)
kubectl get storageclass -o wide

# Verify PVC requests a supported access mode
kubectl get pvc <pvc-name> -o yaml | grep -A5 accessModes
```

Fix: correct the `storageClassName` in the PVC, or mark a StorageClass as default if none is set.

---

## WaitForFirstConsumer — Expected Pending

Some StorageClasses use `WaitForFirstConsumer` binding mode. The PVC stays Pending *intentionally* until a pod is scheduled, so the volume can be created in the same availability zone.

```bash
kubectl describe pvc <pvc-name>
# Events will say:
# waiting for first consumer to be created before binding
```

This is not a bug. But if the pod can't be scheduled, the PVC will also stay Pending. The real problem is the pod scheduling — check `references/pod-lifecycle.md`.

```bash
kubectl describe pod <pod-name>
# Look for scheduling errors: 0/5 nodes are available: insufficient CPU
```

Fix the pod scheduling issue first. Once the pod is placed, the PVC binds automatically.

---

## Multi-Attach Error (Volume Already Attached)

Block volumes (EBS, GCE PD, Azure Disk) can only attach to one node at a time. If a node crashes, the volume may remain "attached" to the old node when Kubernetes tries to attach it to a new one.

```bash
kubectl describe pod <pod-name>
# Events:
# Multi-Attach error for volume "pvc-xxx": Volume is already exclusively attached to one node
```

Fix: wait for Kubernetes to automatically detach the volume (may take several minutes after node failure), or manually force-detach at the cloud provider level if Kubernetes is stuck.

---

## Permission Denied on Mounted Volumes

The pod starts but the application crashes with file permission errors. The container is running as a non-root user but the volume is owned by root.

```bash
kubectl logs <pod-name>
# Look for:
# open /data/config.yaml: permission denied
# failed to create directory /var/lib/app: permission denied
```

Fix: set `fsGroup` in the pod's `securityContext` — Kubernetes will chown the mounted volume to that GID before the container starts:

```yaml
spec:
  securityContext:
    fsGroup: 1000   # GID that matches the container's user
  containers:
    - name: app
      securityContext:
        runAsUser: 1000
```

---

## reclaimPolicy: Accidental Data Loss Risk

Every StorageClass has a `reclaimPolicy` that determines what happens when a PVC is deleted.

```bash
kubectl get storageclass <storageclass-name> -o yaml | grep reclaimPolicy
```

| Policy | Behavior |
|---|---|
| `Delete` | Underlying volume (cloud disk) is *permanently deleted* when PVC is deleted |
| `Retain` | Volume and data survive PVC deletion — requires manual cleanup |

For databases and critical stateful applications, use `Retain`. The default in most cloud environments is `Delete` — deleting a PVC accidentally means the data is gone.

---

## Attach/Detach Errors During Rolling Updates

During rolling updates of stateful sets, old pods must fully detach their volumes before new pods can attach them. If old pods are slow to terminate, new pods hang in ContainerCreating.

```bash
kubectl get events -n <namespace> | grep FailedAttachVolume
kubectl describe pod <new-pod-name> -n <namespace>
```

Fix: ensure `terminationGracePeriodSeconds` gives the old pod enough time to cleanly detach. If pods are stuck terminating, check for finalizers:

```bash
kubectl get pod <pod-name> -o jsonpath='{.metadata.finalizers}'
```
