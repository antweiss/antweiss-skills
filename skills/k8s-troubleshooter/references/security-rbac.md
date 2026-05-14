# Security & Identity: RBAC and Security Contexts

---

## RBAC: Access Denied

Every pod talks to the Kubernetes API as its ServiceAccount. If a pod can't list/get/update a resource, the ServiceAccount lacks the permission.

### Quick diagnostic: `kubectl auth can-i`

```bash
# Test permission for a specific ServiceAccount
kubectl auth can-i list pods \
  --as=system:serviceaccount:<namespace>:<serviceaccount-name> \
  -n <namespace>

# Test what the current user can do
kubectl auth can-i --list -n <namespace>
```

### Find which ServiceAccount a pod is using
```bash
kubectl describe pod <pod-name> | grep "Service Account"
# or
kubectl get pod <pod-name> -o jsonpath='{.spec.serviceAccountName}'
```

### Check existing bindings
```bash
kubectl get rolebindings,clusterrolebindings -n <namespace> \
  | grep <serviceaccount-name>
kubectl describe rolebinding <binding-name> -n <namespace>
```

### RBAC failure in events
```
Warning Forbidden pods is forbidden: User "system:serviceaccount:example:default"
cannot list resource "pods" in API group "" in the namespace "default"
```

### Fix: create a Role and bind it
```yaml
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: <namespace>
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader-binding
  namespace: <namespace>
subjects:
  - kind: ServiceAccount
    name: <serviceaccount-name>
    namespace: <namespace>
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Key rule:** Use `RoleBinding` (namespace-scoped) for application workloads. Only use `ClusterRoleBinding` when cluster-wide access is genuinely needed — it's powerful and easy to over-grant.

The default ServiceAccount has almost no permissions. Always create a dedicated ServiceAccount per workload and grant only what it needs.

---

## Security Context: Operation Not Permitted

Containers run with a limited set of Linux capabilities by default. System-level operations (network config, kernel params, time sync) are blocked.

```bash
kubectl logs <pod-name>
# Look for: Operation not permitted
```

The pod usually starts but the container crashes immediately. There are no Kubernetes events — this is a runtime Linux permission failure.

Fix: identify which capability the app needs and add *only* that:
```yaml
spec:
  containers:
    - name: app
      securityContext:
        capabilities:
          add: ["NET_ADMIN"]    # only if genuinely required
```

Avoid `privileged: true` unless absolutely necessary — it removes all capability restrictions.

---

## Security Context: Read-Only Root Filesystem

`readOnlyRootFilesystem: true` is a security best practice, but many apps write temp files, logs, or caches to default paths and fail when they can't.

```bash
kubectl logs <pod-name>
# Look for:
# open /tmp/app-cache.tmp: read-only file system
# open /var/log/app.log: read-only file system
```

Fix: mount a writable volume for paths the app needs to write to:
```yaml
spec:
  containers:
    - name: app
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /var/log
  volumes:
    - name: tmp
      emptyDir: {}
    - name: logs
      emptyDir: {}
```

---

## Security Context: Non-Root and Port Binding

Linux requires root privileges to bind ports below 1024. A non-root container trying to listen on port 80 or 443 fails with:
```
listen tcp :80: bind: permission denied
```

Fix: configure the app to use a port ≥ 1024 inside the container (e.g., 8080), then expose it through a Service on port 80.

---

## Pod Security Standards (Admission-Level Rejection)

Pod Security Standards (PSS) enforce security policies at the namespace level. Violations are rejected *before the pod is created* — you won't see ContainerCreating, you'll get an admission error.

```bash
# Check namespace security labels
kubectl get ns <namespace> --show-labels | grep pod-security

# Typical error during apply:
# Error from server (Forbidden): pods "app" is forbidden: violates PodSecurity "restricted:latest"
```

Three levels: `privileged` (no restrictions), `baseline` (blocks known escalations), `restricted` (strict production defaults).

**Diagnose what the pod is doing that's not allowed:**
```bash
kubectl describe pod <pod-name>  # check securityContext fields
# Common violations in "restricted" profile:
# - runAsRoot
# - privileged: true
# - hostPath volumes
# - forbidden capabilities
```

Fix options:
1. Modify the pod spec to comply with the enforced profile (preferred)
2. Move the workload to a namespace with a less strict profile if justified
3. Request namespace relabeling from cluster admin

---

## Debugging Tips

When RBAC errors appear in application logs (not kubectl), look for HTTP 403 responses to the Kubernetes API:
```bash
kubectl logs <pod-name> | grep -i "forbidden\|403\|unauthorized\|401"
```

Use `kubectl auth can-i` liberally — it's cheap and gives a definitive answer.
