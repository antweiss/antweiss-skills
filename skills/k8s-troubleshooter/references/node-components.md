# Node Components: Kubelet, Kube-Proxy, CNI

Node-level problems affect all pods on the node, not just one workload. When symptoms span multiple pods or a whole node, look here.

---

## Node Health Overview

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
# Look for: Conditions section (MemoryPressure, DiskPressure, PIDPressure, Ready)
kubectl get events --all-namespaces | grep <node-name>
```

A node in `NotReady` state means the kubelet stopped communicating with the API server. Pods on that node may be evicted or stuck.

---

## Kubelet: PLEG Issues

PLEG (Pod Lifecycle Event Generator) is how the kubelet tracks container state. When the container runtime (containerd or CRI-O) is slow or overloaded, PLEG falls behind.

Symptoms: pods stuck in strange states, restarts taking longer than usual, missing status updates.

**Kubelet logs** are the source of truth:
```bash
# On nodes using systemd (most Linux systems)
journalctl -u kubelet --since "10 minutes ago"

# Or via a debug pod on the node:
kubectl debug node/<node-name> -it --image=busybox
```

Look for:
```
PLEG is not healthy: pleg was last seen active <time> ago, threshold is 3m0s
```

Fix: investigate container runtime health (`systemctl status containerd`), reduce pod density on the node, or cordon and drain if the runtime is unrecoverable.

---

## Kubelet: Eviction Manager

When a node runs critically low on resources, the kubelet evicts pods to protect itself.

```bash
kubectl describe node <node-name> | grep -A10 "Conditions:"
# Look for: MemoryPressure=True, DiskPressure=True, PIDPressure=True
kubectl get events --all-namespaces | grep Evicted
```

### Disk Pressure
Kubernetes tracks two filesystems separately:
- `nodefs`: node's root filesystem (`/`)
- `imagefs`: where container images are stored (often `/var/lib/containerd`)

Either filling up triggers evictions even if CPU and memory look fine.

```bash
# On the node (via debug pod or SSH)
df -h /
df -h /var/lib/containerd  # or /var/lib/docker
```

Fix: clean up unused images (`crictl rmi --prune`), increase disk, or set image GC thresholds in kubelet config.

### PID Pressure
Every process and container uses a PID. If the node runs out, new containers can't start.

```bash
cat /proc/sys/kernel/pid_max
ps aux | wc -l  # current process count
```

Fix: reduce pod density, fix zombie process leaks (see `references/advanced-failures.md`), or increase `pid_max` on the node.

---

## Kube-Proxy: IPTables vs IPVS

Kube-proxy programs node networking so Services can route to pods. It runs in either `iptables` or `ipvs` mode.

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
```

**iptables mode**: simple, widely used, but creates many rules. On busy nodes, rule updates can lag. When rules are stale, traffic fails even though Services and Endpoints look correct.

**IPVS mode**: faster and scales better, but less familiar to debug. Use `ipvsadm -Ln` on the node to inspect rules.

If traffic behaves differently across nodes, check whether kube-proxy is running correctly on each:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=30
```

---

## CNI: Networking Plugin Issues

The CNI plugin assigns pod IPs and wires pod networking. CNI failures cause `ContainerCreating` or pods running without network access.

```bash
# CNI configuration lives here on each node
ls /etc/cni/net.d/
```

If this directory is missing or has conflicting configs, pod networking breaks.

**Diagnose CNI logs** (plugin-specific; varies by CNI):
```bash
# For Calico
kubectl logs -n calico-system -l k8s-app=calico-node --tail=50

# For Flannel
kubectl logs -n kube-system -l app=flannel --tail=50

# For Cilium
kubectl logs -n kube-system -l k8s-app=cilium --tail=50
```

---

## IP Address Exhaustion

CNI plugins allocate pod IPs from a fixed subnet. When the subnet is full, new pods can't get networking — they're stuck in ContainerCreating even though they're scheduled.

```bash
kubectl get events -n <namespace> | grep "failed to assign IP"

# For Calico: check IP pool usage
kubectl get ipamblocks  # requires Calico CRDs

# For AWS VPC CNI
kubectl describe node <node-name> | grep -i "vpc-cni\|eni\|ipv4"
```

Signs: scale-up events fail, rolling updates stall, cluster looks healthy but new pods hang.

Fix: expand the subnet CIDR, add IP pools, or redesign network subnetting for expected peak pod count. This requires planning — you can't easily change IP ranges in a running cluster without node replacement.

---

## Draining a Node for Maintenance

When a node is problematic, drain it before investigating:
```bash
# Cordon (prevent new scheduling)
kubectl cordon <node-name>

# Drain (evict existing pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Restore after fix
kubectl uncordon <node-name>
```
