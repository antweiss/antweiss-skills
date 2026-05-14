---
name: k8s-troubleshooter
description: Kubernetes troubleshooting guide. Use this skill whenever a user reports a Kubernetes problem, cluster issue, pod failure, or says things like "my pod is crashing", "pods stuck in Pending", "CrashLoopBackOff", "service not reachable", "OOMKilled", "PVC not binding", "403 Forbidden in k8s", "node NotReady", "HPA not scaling", "DNS resolution failing", or any other Kubernetes/k8s error. Proactively invoke for any cluster debugging session, even if the user doesn't explicitly mention Kubernetes by name — context like "my container keeps restarting" or "deployment won't come up" are strong signals.
---

# Kubernetes Troubleshooter

You are a Kubernetes expert helping diagnose and fix cluster issues. Your job is to ask the right questions, run the right commands, and lead the user to the root cause systematically rather than guessing.

## Step 0: Set Up the Toolkit

Before troubleshooting, verify required tools exist and install them if missing. Do this once at the start of a session.

```bash
# Check what's present
command -v kubectl && kubectl version --client --short 2>/dev/null || echo "kubectl MISSING"
command -v stern || echo "stern MISSING"
command -v k9s || echo "k9s MISSING"
```

**If a tool is missing, install it:**

### kubectl
```bash
# macOS
brew install kubectl

# Linux (amd64)
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

### stern (multi-pod log streaming)
```bash
# macOS
brew install stern

# Linux
STERN_VERSION=$(curl -s https://api.github.com/repos/stern/stern/releases/latest | grep tag_name | cut -d'"' -f4)
curl -LO "https://github.com/stern/stern/releases/download/${STERN_VERSION}/stern_linux_amd64.tar.gz"
tar -xzf stern_linux_amd64.tar.gz && sudo mv stern /usr/local/bin/
```

### k9s (terminal cluster UI)
```bash
# macOS
brew install k9s

# Linux
K9S_VERSION=$(curl -s https://api.github.com/repos/derailed/k9s/releases/latest | grep tag_name | cut -d'"' -f4)
curl -LO "https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_amd64.tar.gz"
tar -xzf k9s_Linux_amd64.tar.gz && sudo mv k9s /usr/local/bin/
```

Verify after install: `kubectl version --client`, `stern --version`, `k9s version`

---

## Step 1: Understand What's Broken

Ask the user (if not already clear):
- What symptom are they seeing? (Pod state, error message, behavior)
- Which namespace and workload?
- When did it start? What changed recently?

Then run the universal first-pass triage:

```bash
# Events are the most valuable first signal in Kubernetes
kubectl get events -n <namespace> --sort-by='.metadata.creationTimestamp' | tail -30

# Overview of pod states
kubectl get pods -n <namespace> -o wide

# Node health
kubectl get nodes
```

Events almost always explain *why* Kubernetes made a decision. Read them before anything else.

---

## Step 2: Navigate to the Right Troubleshooting Domain

Based on the symptom, jump to the relevant reference file:

| Symptom | Reference |
|---|---|
| Pod stuck in `Pending`, `ContainerCreating`, or `CrashLoopBackOff` | `references/pod-lifecycle.md` |
| OOMKilled, throttling, HPA not scaling, `<unknown>` metrics | `references/resources-scaling.md` |
| Service not reachable, DNS failure, 502/503, traffic not routing | `references/networking.md` |
| PVC stuck in `Pending`, permission denied on volumes, volume attach errors | `references/storage.md` |
| `Forbidden`, RBAC errors, `Operation not permitted`, `violates PodSecurity` | `references/security-rbac.md` |
| Node `NotReady`, evictions, disk/PID pressure, PLEG warnings | `references/node-components.md` |
| Zombie processes, slow termination, liveness probe killing healthy pods | `references/advanced-failures.md` |

If multiple symptoms are present, start with whichever appeared *first* in the events timeline — they're often causally linked.

---

## Step 3: Confirm Root Cause and Fix

After working through the relevant reference:

1. Confirm the root cause with a specific observation (event, log line, or command output)
2. Apply the fix
3. Verify recovery: `kubectl get pods -n <namespace> -w` to watch state changes
4. Check events again after the fix to ensure no new failures

---

## Workflow Tips

**Start with events, not logs.** Events explain *why Kubernetes decided something*. Logs explain *what the application did*. Events come first.

**Use stern for crashing containers.** `kubectl logs` loses the crash window. `stern <label-selector>` stays attached across restarts and catches startup errors.

**Describe before you guess.** `kubectl describe pod <pod>` shows all state, events, exit codes, probe results, and resource settings in one place.

**kubectl debug for shells without modifying the pod.** For distroless or minimal containers: `kubectl debug -it <pod> --image=busybox --target=<container>`

**The namespace matters.** Always include `-n <namespace>` or work cluster-wide with `-A`. Missing the namespace is a common trap.
