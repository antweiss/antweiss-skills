# Resources and Scaling: Performance and Stability

## Requests vs Limits — The Core Concept

**Requests** are what the scheduler uses to place a pod. The scheduler treats them as *guaranteed* capacity — it won't put the pod on a node that can't meet them.

**Limits** are the runtime cap. If a container exceeds its memory limit, the kernel kills it (OOMKill, exit code 137). If it exceeds its CPU limit, it gets throttled.

Mismatches between requests/limits and actual usage are the root cause of most scaling and stability problems.

---

## CPU Problems

### Throttling (but pod stays running)
The container is capped by its CPU limit. It looks healthy, but latency rises under load.
```bash
# Check if limits are set
kubectl describe pod <pod-name> | grep -A5 Limits

# CPU throttling shows in metrics — look for cpu_throttled_seconds_total in Prometheus
# Or check with top inside the container:
kubectl exec -it <pod-name> -- top
```
Fix: raise CPU limit, or remove it entirely if the workload bursts safely.

### Pod stuck Pending due to high CPU request
```
0/3 nodes are available: 3 Insufficient cpu.
```
Fix: reduce CPU request to match realistic usage, not peak.

---

## Memory Problems

### OOMKilled (exit code 137)
The container used more memory than its limit. The kernel killed it.
```bash
kubectl describe pod <pod-name>
# Look for:
# Last State: Terminated, Reason: OOMKilled, Exit Code: 137
```
Fix: increase the memory limit, or find and fix the memory leak in the application.

### Eviction due to node memory pressure
Pods evicted because the *node* ran low on memory:
```bash
kubectl describe node <node-name>  # look for MemoryPressure condition
kubectl get events -n <namespace> | grep Evicted
```
Fix: reduce pod memory requests, add nodes, or fix over-committed cluster sizing.

---

## QoS Classes (determines eviction priority)

| QoS Class | When | Risk |
|---|---|---|
| Guaranteed | requests == limits for all containers | Last to be evicted |
| Burstable | requests < limits | Evicted under pressure |
| BestEffort | no requests or limits set | First to be evicted |

Check a pod's QoS class:
```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
```
Production workloads should be Guaranteed or Burstable with sane values — never BestEffort.

---

## ResourceQuota

Quotas cap total resource usage per namespace. If a quota is full, new pods get rejected.

```bash
kubectl describe resourcequota -n <namespace>
# Look for: Used vs Hard columns
```

If Used ≈ Hard for cpu or memory, new pods will fail to schedule. Fix: request a quota increase, clean up unused workloads, or move to a different namespace.

---

## LimitRanges

LimitRanges inject *default* requests and limits when none are specified. They can silently over-constrain pods.

```bash
kubectl get limitrange -n <namespace> -o yaml
```

If a pod is mysteriously Pending with resource pressure but you didn't set any requests, LimitRange is the likely culprit.

---

## HPA Flapping

The Horizontal Pod Autoscaler scales too aggressively — pods go up and down repeatedly.

```bash
kubectl describe hpa <hpa-name> -n <namespace>
# Look for: Current replicas, Desired replicas, scaling events
kubectl get events -n <namespace> | grep HPA
```

**Common causes:**
- `stabilizationWindowSeconds` too short (default 300s for scale-down, 0 for scale-up)
- Metric thresholds too sensitive relative to load variance
- CPU/memory requests not matching actual usage — HPA ratio math is based on requests

Fix: tune `behavior.scaleDown.stabilizationWindowSeconds`, adjust thresholds, or fix resource requests first.

---

## HPA Showing `<unknown>` Metrics

The HPA can't read metrics. It won't scale at all.

```bash
kubectl describe hpa <hpa-name>
# Look for: unable to get metrics, metrics not available
```

**Common causes:**
- `metrics-server` not installed or not working:
  ```bash
  kubectl get pods -n kube-system | grep metrics-server
  kubectl top pods -n <namespace>  # should work if metrics-server is healthy
  ```
- Custom metrics adapter (Prometheus Adapter, etc.) not configured correctly
- Resource requests not set on pods (HPA can't compute utilization %)

Fix: install/fix metrics-server, or configure the correct metrics adapter.

---

## VPA and HPA Conflict

Running both VPA (Vertical Pod Autoscaler) and HPA on the same workload based on CPU/memory causes them to fight each other. VPA changes requests, which changes HPA's scaling math, causing oscillation.

If both are needed: use HPA on custom/external metrics and VPA on CPU/memory, or use them on different workloads.
