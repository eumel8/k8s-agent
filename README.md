# Kubernetes DevOps Agent

A **project-specific agent configuration** for Kubernetes DevOps work using [OpenCode](https://opencode.ai). This repo provides structured instructions, MCP server configurations, skills, and best practices for an AI agent that diagnoses, troubleshoots, and resolves problems on Kubernetes clusters.

## Repository Structure

```
k8s-agent/
├── AGENTS.md                          # Agent role definition and troubleshooting playbook
├── MCP.md                             # MCP server configuration guide
├── Modelfile                          # Ollama model definition for local inference
├── opencode.json                      # Active OpenCode configuration (Ollama provider)
├── .opencode/
│   ├── skills/
│   │   └── k8s-incident-response/
│   │       └── SKILL.md               # Incident response skill with runbooks
│   └── node_modules/                  # Skill dependencies (gitignored)
├── .gitignore                         # Ignores node_modules, JSON configs, session files
```

## Files Overview

### `AGENTS.md`

Core agent instructions defining the senior DevOps Engineer role. Covers:

- **Operating principles** — inspect before acting, follow the evidence, structured troubleshooting
- **Kubernetes tooling** — kubectl, helm, flux, kustomize, curl, jq
- **Resource troubleshooting patterns** — Pod, Deployment, PVC, Service, Ingress, Node
- **Controller and operator investigation** — how to trace ownership chains
- **Cluster API** — Machine, MachineDeployment, KubeadmControlPlane
- **Flux** — Kustomization, HelmRelease, source reconciliation
- **Storage** — PVC → PV → StorageClass → CSI driver dependency chain
- **Networking** — pod networking, DNS, Ingress, CNI, NetworkPolicy
- **Safety rules** — reversible changes, no blind deletions, verification required

### `MCP.md`

Configuration guide for three local MCP servers (no external cloud services required):

| MCP Server | Purpose | Token Overhead |
|---|---|---|
| **GitLab** (`@structured-world/gitlab-mcp`) | Issues, MRs, pipelines, CI/CD (44 tools) | ~8,000–12,000 |
| **Prometheus** (`prometheus-mcp-server`) | PromQL queries, metrics discovery (5 tools) | ~500–800 |
| **Alertmanager** (`mcp-alertmanager`) | Alert investigation, silences (5–12 tools) | ~500–2,000 |

Includes auth options (Basic Auth, Bearer Token, K8s auto-connect), feature flags, and example prompts. See [`MCP.md`](MCP.md) for full details.

### `Modelfile`

Ollama model definition for running a local Kubernetes-optimized agent. Creates the `ornith-k8s` model with:

- Base model: `ornith`
- Temperature: 0.2, context window: 65k tokens
- System prompt tuned for execution-oriented DevOps work (no narration, tool-first)

Usage:
```bash
ollama create ornith-k8s -f Modelfile
```

### `opencode.json`

Active OpenCode configuration with:

- **Ollama provider** — connects to a local Ollama instance (ornith-k8s model, 262k context)
- **Agentmemory MCP** — long-term memory across sessions
- **Permissions** — bash, edit, write, read, task all allowed
- **Plugin** — `agentmemory-capture.ts` for automatic memory capture

### `.opencode/skills/k8s-incident-response/`

Structured incident response playbook with:

- **Severity classification** (P1–P4) with response times
- **6-step workflow** — confirm, blast radius, diagnose, mitigate, verify, communicate
- **Common runbooks** — CrashLoopBackOff, Pod Pending, Node NotReady
- **Communication templates** — during-incident updates and post-incident summaries

## Prerequisites

- [OpenCode](https://opencode.ai) CLI
- Node.js >= 24 (for MCP servers)
- Access to a Kubernetes cluster via `kubectl`
- Optional: Ollama for local model inference
- Optional: GitLab access token for GitLab MCP
- Optional: Prometheus/Alertmanager endpoints for monitoring MCP

## Quick Start

1. Clone this repository
2. Configure your `opencode.json` (or use the existing one with Ollama)
3. Optionally set up MCP servers — see [`MCP.md`](MCP.md)
4. Start OpenCode in this directory

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes with tests
5. Submit a pull request

## Credits

Frank Kloeker <f.kloeker@telekom.de>

Life is for sharing. If you have an issue with the code or want to improve it, feel free to open an issue or an pull request.
