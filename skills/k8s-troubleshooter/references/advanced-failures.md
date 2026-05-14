# Advanced and Rare Failures

These problems appear in production under real load, after the cluster has been running a while, or during shutdowns and restarts. They're hard to find unless you know what to look for.

---

## PID 1: Why It Matters

PID 1 is the main process inside a container. It has special OS responsibilities:
- Must handle system signals (SIGTERM, SIGKILL)
- Must reap zombie child processes

Most applications are *not* written to run as PID 1. When they are, signals may be ignored and child processes accumulate as zombies.

**Diagnose: is the app running as PID 1?**
```bash
kubectl exec -it <pod-name> -- ps aux
# PID 1 should be the app or an init process (tini, dumb-init), not bash/sh
```

If you see `bash` or `sh` as PID 1, signals from Kubernetes won't reach the actual app.

Fix: use `exec` form in the Dockerfile/Pod, or add a proper init:
```dockerfile
# Wrong - shell wraps the app, swallows signals
CMD ["sh", "-c", "myapp"]

# Right - exec form, app is PID 1
CMD ["myapp"]

# Or use tini as init
ENTRYPOINT ["/sbin/tini", "--", "myapp"]
```

In a pod spec:
```yaml
spec:
  containers:
    - name: app
      command: ["myapp"]  # exec form, not wrapped in sh -c
```

---

## Signal Propagation: Slow/Stuck Termination

When Kubernetes deletes a pod, it sends SIGTERM and waits `terminationGracePeriodSeconds` (default 30s). If the app doesn't stop, it gets SIGKILL. If signals aren't reaching the app, you see pods stuck in `Terminating`.

**Diagnose:**
```bash
kubectl delete pod <pod-name>
kubectl get pod <pod-name> -w  # watch — does it hang in Terminating?
kubectl get events -n <namespace> | grep Killing
# Look for: Container failed to stop in time
```

**Common cause:** shell script wraps the app — SIGTERM reaches the shell, not the app:
```bash
#!/bin/sh
exec myapp "$@"   # "exec" replaces the shell — correct signal forwarding
# vs
myapp "$@"        # shell stays as PID 1 — signal doesn't reach myapp
```

Fix: use `exec` in shell scripts to replace the shell process with the app, or use an init process that forwards signals.

Increase `terminationGracePeriodSeconds` if the app legitimately needs more time to drain:
```yaml
spec:
  terminationGracePeriodSeconds: 60
```

---

## Zombie Processes and Node Instability

Zombie processes occur when child processes exit but are never reaped. They consume no CPU or memory, but they hold PIDs.

Over time, a node can exhaust its PID space. When that happens, no new containers can start — pods fail for no obvious reason.

**Diagnose:**
```bash
# On the node (via debug pod or SSH)
ps aux | grep '<defunct>'   # zombies show as <defunct>
cat /proc/sys/kernel/pid_max
ls /proc | wc -l  # rough current PID count
```

Fix: use **tini** as an init process inside containers. Tini:
- Acts as a proper PID 1
- Forwards signals correctly  
- Reaps zombie child processes

```dockerfile
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--", "myapp"]
```

Or enable tini in Docker: `docker run --init ...` (but this doesn't help in Kubernetes without embedding tini in the image).

---

## Liveness Probes: When the "Fix" Becomes the Problem

Liveness probes are meant to detect broken apps and restart them. Misconfigured, they cause unnecessary restarts and can cause cascading failures.

**Common mistakes:**

### Probe too aggressive under load
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5    # too short — app not ready yet
  periodSeconds: 5          # too frequent
  failureThreshold: 1       # one slow response = restart
  timeoutSeconds: 1         # too tight
```
Under high load, the app responds slowly. Kubernetes kills and restarts it, making load worse. This is a self-inflicted restart loop.

Fix: tune timeouts generously, increase `failureThreshold`, and separate readiness (controls traffic) from liveness (controls restarts).

### Exec-based probes and zombie accumulation
Every `exec` probe creates a new process. If those processes aren't reaped (PID 1 problem again), they contribute to zombie buildup.

```bash
kubectl describe pod <pod-name>
# Look for: Liveness probe failed
kubectl get events -n <namespace> | grep Unhealthy
```

Fix: prefer `httpGet` over `exec` probes where possible. If using exec, ensure an init process is cleaning up child processes.

---

## Debugging Node-Level Issues

When the problem is on the node itself (not inside a pod), use a privileged debug pod:

```bash
# Launch debug pod on the node
kubectl debug node/<node-name> -it --image=ubuntu

# Inside the debug pod, you have host access
chroot /host
journalctl -u kubelet --since "30 minutes ago"
systemctl status containerd
df -h
ps aux | grep '<defunct>'
```

In managed clusters (EKS, GKE, AKS), direct SSH may be restricted. Use `kubectl debug node/` instead — it's the recommended approach.

---

## Real-World Patterns to Watch For

**Label changes during upgrades:** Kubernetes 1.24 removed the `node-role.kubernetes.io/master` label, replacing it with `control-plane`. Components that select nodes by label (CNI plugins, scheduling rules) can break silently after an upgrade. Always audit label selectors when upgrading.

**CoreDNS over-scaling down:** Autoscalers can reduce CoreDNS replicas too aggressively. Under load, DNS lookups fail intermittently — apps look healthy, but can't find each other. Monitor CoreDNS query failure rates, not just pod count.

**Node bootstrap failures:** New nodes joining the cluster may appear Ready but fail to actually run workloads if bootstrap scripts have unreachable dependencies (package repos, container registries). Pods get scheduled but never start. Check node Events and kubelet logs on the new node, not just the pod events.
