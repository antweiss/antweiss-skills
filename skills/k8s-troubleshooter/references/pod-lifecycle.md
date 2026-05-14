# Pod Lifecycle: Startup and Runtime Problems

## Quick Triage

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.metadata.creationTimestamp' | grep <pod-name>
```

The pod state tells you *where* it failed. The events tell you *why*.

---

## ImagePullBackOff / ErrImagePull

The container image couldn't be downloaded. The pod never reaches scheduling.

**Diagnose:**
```bash
kubectl describe pod <pod-name>   # look at Events section
```

**Case 1 — Wrong image name or tag:**
```
Failed to pull image "myapp:v2": manifest for myapp:v2 not found
```
Fix: correct the image name/tag in the Deployment/Pod spec and redeploy.

**Case 2 — Registry rate-limited or unreachable:**
```
Failed to pull image: toomanyrequests - You have reached your pull rate limit
Failed to pull image: dial tcp: lookup registry-1.docker.io: no such host
```
Fix: use a private registry or mirror, wait for limits to reset, check node outbound network access.

**Case 3 — Missing registry credentials:**
```
Failed to pull image "private-registry/app:1.0": unauthorized: authentication required
```
Fix: create or fix the `imagePullSecret`, ensure it exists in the same namespace, reference it in the Pod or ServiceAccount.

---

## Pod Stuck in Pending

Scheduled hasn't placed the pod on any node.

**Diagnose:**
```bash
kubectl describe pod <pod-name>  # look for FailedScheduling events
```

**Common causes:**

### Insufficient resources
```
0/3 nodes are available: 3 Insufficient cpu.
```
Fix: reduce resource requests, scale up nodes, or check if requests are inflated.

### Taint/toleration mismatch
```
0/3 nodes are available: node(s) had taint {node-role.kubernetes.io/control-plane: NoSchedule}
```
Fix: add the matching toleration to the pod spec, or adjust node taints.

### Node affinity / node selector mismatch
```
0/5 nodes are available: node(s) didn't match node affinity
```
Fix: check pod `nodeSelector` and `affinity` against actual node labels:
```bash
kubectl get nodes --show-labels
```

### LimitRange injecting defaults
LimitRanges can silently add resource defaults that exceed available capacity.
```bash
kubectl get limitrange -n <namespace>
```

---

## Pod Stuck in ContainerCreating

Pod was scheduled but never reached Running. Usually a networking, volume, or ConfigMap issue.

**Diagnose:**
```bash
kubectl describe pod <pod-name>  # Events section is key
```

**Common causes:**

### CNI failure / IP exhaustion
```
Failed to create pod sandbox: failed to assign IP address
```
Fix: check CNI plugin health, look at node-level CNI logs, check IP exhaustion (see `references/node-components.md`).

### Missing ConfigMap or Secret
```
configmap "app-config" not found
secret "db-password" not found
```
Fix: create the missing resource in the same namespace.

### Volume mount failure
```
Unable to attach or mount volumes: timed out waiting for the condition
```
Fix: see `references/storage.md` for volume troubleshooting.

---

## CrashLoopBackOff

The container starts but immediately exits. Kubernetes backs off before retrying.

**Diagnose:**
```bash
# See exit code and last state
kubectl describe pod <pod-name>

# Catch startup logs (even across restarts)
stern <deployment-name> -n <namespace>

# Or get previous container logs
kubectl logs <pod-name> --previous
```

**Exit codes:**

| Code | Meaning |
|------|---------|
| 0 | App exited cleanly — wrong restart policy or wrong command |
| 1 or 255 | Application error — check logs |
| 137 | OOMKilled (memory limit exceeded) — see `references/resources-scaling.md` |
| 143 | SIGTERM received — graceful shutdown during rolling update or eviction |

**InitContainers:** If the main container never starts, check InitContainer status:
```bash
kubectl describe pod <pod-name>  # look for Init Container section
kubectl logs <pod-name> -c <init-container-name>
```
An InitContainer failure blocks everything. Fix it first.

---

## Reading Logs Efficiently

Use `stern` for logs across multiple pods or across restarts:
```bash
# All pods whose name contains "backend"
stern backend -n <namespace>

# By label selector
stern -l app=api -n <namespace>

# Multi-container pods (shows all containers)
stern <pod-name> -n <namespace> --all-containers
```

`stern` auto-attaches when a pod restarts, so you catch the crash window that `kubectl logs` misses.
