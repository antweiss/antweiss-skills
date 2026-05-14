# Networking & Connectivity

Networking failures are the hardest to diagnose in Kubernetes because traffic flows through multiple layers. A small mistake anywhere breaks the whole path — silently.

Work through these layers in order: DNS → Service → Ingress → Network Policies.

---

## DNS Troubleshooting

### Step 1: Verify DNS resolves correctly from inside a pod
```bash
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>.<namespace>.svc.cluster.local
```

If this fails but the pod itself is running, DNS is the problem.

### Step 2: Check CoreDNS health
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

**Signs of CoreDNS problems:**
```
[ERROR] plugin/errors: 2 kubernetes.default.svc.cluster.local. A: read udp ...: i/o timeout
```
→ CoreDNS can't reach its upstream DNS. Check node-level DNS and firewalls.

```
[ERROR] plugin/errors: 2 myservice.default.svc.cluster.local. AAAA: no such host
```
→ Service doesn't exist or wasn't registered in DNS.

### Step 3: Check CoreDNS scaling
Under load, too few CoreDNS replicas cause intermittent DNS timeouts:
```bash
kubectl get deployment coredns -n kube-system
```
Scale up if needed. CoreDNS should scale with cluster size.

### The ndots:5 Problem
By default, Kubernetes sets `ndots:5`. Any domain with fewer than 5 dots gets tried against several internal search paths before going external. This adds latency to outbound requests — not app slowness, but DNS lookup slowness.

```bash
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
# Look for: options ndots:5
```

Fix: use fully-qualified domain names (trailing dot) for external calls, or reduce ndots in pod dnsConfig.

---

## Service Troubleshooting

### Verify the Service has Endpoints
If a Service has no endpoints, traffic goes nowhere — DNS resolves fine but connections fail.
```bash
kubectl describe svc <service-name> -n <namespace>
# Look for: Endpoints: <none>

kubectl get endpoints <service-name> -n <namespace>
```

If Endpoints is empty, the Service selector doesn't match any pod labels:
```bash
kubectl get pods -n <namespace> --show-labels
# Compare to the selector in: kubectl describe svc <service-name>
```

Labels must match *exactly* — a missing or misspelled label means zero endpoints.

### port vs targetPort confusion
- `port`: what clients connect to on the Service
- `targetPort`: what port the container actually listens on

If these don't match, connections reach the pod but fail. The pod looks healthy, requests fail:
```bash
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A5 ports
```

---

## Ingress / Gateway API Troubleshooting

### 502 Bad Gateway
The Ingress controller can't reach the backend pod.
- Check that the Service has Endpoints (see above)
- Check that `targetPort` matches the container's port

### 503 Service Unavailable
Service exists but no healthy pods behind it. Usually readiness probe failures:
```bash
kubectl describe pods -n <namespace> | grep -A5 "Readiness"
kubectl get events -n <namespace> | grep Unhealthy
```
Fix: correct the readiness probe path, port, or timeout. Make sure the app is actually ready before receiving traffic.

### TLS Certificate Issues
Connections fail during handshake — looks like random connectivity problems from outside.
```bash
kubectl describe ingress <ingress-name> -n <namespace>
kubectl get secret -n <namespace>  # verify TLS secret exists
```

### Gateway API (HTTPRoute / Gateway)
```bash
kubectl get gateways -n <namespace>
kubectl describe gateway <gateway-name> -n <namespace>
# Look for conditions: Accepted: False or Programmed: False
```

Common issues: route not attached to a valid Gateway, hostname mismatch, backend service doesn't exist or has no endpoints.

---

## Network Policies

NetworkPolicies silently drop traffic — no error message, just timeouts. The pod looks healthy.

```bash
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>
```

If traffic works *within* a namespace but fails *across* namespaces, a NetworkPolicy is almost certainly the cause.

Check both ingress *and* egress rules. Missing either direction blocks traffic.

**Quick test — temporarily delete the policy** (in a non-prod environment):
```bash
kubectl delete networkpolicy <policy-name> -n <namespace>
# Test connectivity. If it works, the policy was the issue.
```

Then fix the policy to allow the required traffic rather than leaving it deleted.

---

## Debugging with a Network Debug Pod

When you need to test connectivity from inside the cluster:
```bash
kubectl run netdebug --image=busybox --rm -it --restart=Never -n <namespace> -- sh
# Inside the pod:
nslookup <service-name>
wget -qO- http://<service-name>:<port>/health
nc -zv <service-ip> <port>
```

Or attach an ephemeral debug container to an existing pod:
```bash
kubectl debug -it <pod-name> -n <namespace> --image=busybox --target=<container-name>
```
