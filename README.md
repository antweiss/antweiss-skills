# Claude Code Skills Collection

This repository is a collection of specialized [Claude Code](https://claude.ai/code) skills designed to extend the agent's capabilities with domain-specific knowledge, workflows, and automated tools.

## Skills

### ☸️ k8s-troubleshooter

The `k8s-troubleshooter` skill transforms Claude Code into a senior Kubernetes engineer. It provides expert diagnostic guidance, automated tool installation, and structured troubleshooting workflows.

#### Key Features
- **Deep Knowledge Base**: Comprehensive reference guides for Networking (DNS, Ingress), Storage (PVCs, permissions), Security (RBAC, Pod Security Standards), Pod Lifecycle (exit codes, OOMKills), Resources & Scaling (HPA, limits), and Node Components.
- **Tool Management**: Automatically checks for and installs `kubectl` if missing.
- **Diagnostic Decision Tree**: Guided workflows to quickly move from symptoms to root cause identification.
- **Evidence-Based**: Based on the established patterns found in [The Ultimate Kubernetes Troubleshooting Guide by PerfectScale](https://info.perfectscale.io/kubernetes-troubleshooting-guide).

#### Installation

```
/plugin marketplace add antweiss/claude-skills
/plugin install k8s-troubleshooter@claude-skills
```

#### Usage

Once installed, the skill triggers automatically when you describe a Kubernetes problem. Examples:

> "My pods are stuck in CrashLoopBackOff after today's deployment"

> "Service exists but requests are timing out — DNS lookup is failing"

> "HPA shows `<unknown>` for CPU metrics and isn't scaling"

---

## Contributing

Contributions of new skills or improvements to existing ones are welcome. Each skill lives in its own subdirectory under `skills/` with a `SKILL.md` file and optional `references/` for domain-specific guides.
