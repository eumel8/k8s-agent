# MCP Servers for the Kubernetes DevOps Agent

## Overview

Three MCP servers are relevant for this agent, all run **locally** and
**no external cloud services** required:

| MCP Server | Purpose | Package |
|---|---|---|
| GitLab | Issues, MRs, Pipelines, CI/CD | `@structured-world/gitlab-mcp` |
| Prometheus | Query metrics, check targets | `prometheus-mcp-server` |
| Alertmanager | Alerts, silences, root-cause analysis | `mcp-alertmanager` |

---

## Prerequisites

- Node.js >= 24 (for GitLab and Prometheus MCP)
- Access to the internal GitLab server
- GitLab Personal Access Token (scope: `api` or `read_api`)
- Prometheus and Alertmanager reachable (URL or via kubeconfig)

---

## Configuration Locations

The `opencode.json` can exist in two places — both are merged:

| Location | Purpose | Git-safe |
|---|---|---|
| `~/.config/opencode/opencode.json` | Global for all projects | No (contains secrets) |
| `<project-directory>/opencode.json` | Project-specific | Yes (without secrets) |

**Recommendation:** MCP servers with tokens go in the **global config**. Never store
secrets in plaintext — use variable substitution:

```jsonc
// Env variable:
"GITLAB_TOKEN": "{env:GITLAB_TOKEN}"

// File (e.g. chmod 600):
"PROMETHEUS_PASSWORD": "{file:~/.secrets/prometheus-password}"
```

---

## Configuration in opencode.json

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {

    // --- Agentmemory ---
    "agentmemory": {
      "type": "local",
      "command": ["npx", "-y", "@agentmemory/mcp"],
      "enabled": true
    },

    // --- GitLab ---
    "gitlab": {
      "type": "local",
      "command": ["npx", "-y", "@structured-world/gitlab-mcp"],
      "enabled": true,
      "environment": {
        "GITLAB_TOKEN": "glpat-xxxxxxxxxxxxxxxxxxxx",
        "GITLAB_API_URL": "https://gitlab.example.com"
      }
    },

    // --- Prometheus (Basic Auth) ---
    "prometheus": {
      "type": "local",
      "command": ["npx", "-y", "prometheus-mcp-server"],
      "enabled": true,
      "environment": {
        "PROMETHEUS_URL": "https://prometheus.example.com",
        "PROMETHEUS_USERNAME": "monitoring-user",
        "PROMETHEUS_PASSWORD": "secret",
        // Optional: ignore self-signed certs
        // "PROMETHEUS_INSECURE": "true"
      }
    },

    // --- Alertmanager ---
    // Three options — see Alertmanager section below.
    //
    // Option A: K8s auto-connect (no Basic Auth needed, recommended inside cluster)
    "alertmanager": {
      "type": "local",
      "command": ["npx", "-y", "mcp-alertmanager@latest"],
      "enabled": true
    }
    //
    // Option B: Basic Auth via Ingress (Python, ntk148v)
    // "alertmanager": {
    //   "type": "local",
    //   "command": ["uv", "run", "--directory", "/opt/alertmanager-mcp-server",
    //               "src/alertmanager_mcp_server/server.py"],
    //   "enabled": true,
    //   "environment": {
    //     "ALERTMANAGER_URL": "https://alertmanager.example.com",
    //     "ALERTMANAGER_USERNAME": "admin",
    //     "ALERTMANAGER_PASSWORD": "secret"
    //   }
    // }
    //
    // Option C: Basic Auth / Bearer Token via CLI flag (Go binary, zekker6)
    // "alertmanager": {
    //   "type": "local",
    //   "command": ["/usr/local/bin/mcp-alertmanager",
    //               "-url", "https://alertmanager.example.com",
    //               "-username", "admin",
    //               "-password-file", "/etc/alertmanager-mcp/password"],
    //   "enabled": true
    // }

  },
  "plugin": ["./plugins/agentmemory-capture.ts"]
}
```

---

## Agentmemory

**Package:** `@agentmemory/mcp`
**Tools:** 12 memory categories

| Tool | Description |
|---|---|
| `agentmemory_memory_save` | Save insights/decisions to long-term memory |
| `agentmemory_memory_recall` | Search past session observations |
| `agentmemory_memory_smart_search` | Hybrid semantic+keyword search |
| `agentmemory_memory_sessions` | List recent sessions |
| `agentmemory_memory_export` | Export all memory as JSON |
| `agentmemory_memory_audit` | View audit trail |
| `agentmemory_memory_governance_delete` | Delete memories with audit trail |

```jsonc
  "mcp": {
    "agentmemory": {
      "type": "local",
      "command": ["npx", "-y", "@agentmemory/mcp"],
      "enabled": true
    }
  },
  "plugin": ["./plugins/agentmemory-capture.ts"]
```

## GitLab MCP

**Package:** `@structured-world/gitlab-mcp` (v7.x)
**Tools:** 44 tools across 18 entity types

### Key Tools

| Tool | Description |
|---|---|
| `browse_projects` | Search and list projects |
| `browse_merge_requests` | Filter MRs (state, author, etc.) |
| `manage_issues` | Create, read, close issues |
| `browse_pipelines` | Pipeline status and logs |
| `browse_files` | Read files in repository |

### Feature Flags (selective disable)

Unused tool groups can be disabled via environment variables to reduce token consumption:

```jsonc
"environment": {
  "GITLAB_TOKEN": "glpat-xxxx",
  "GITLAB_API_URL": "https://gitlab.example.com",
  "USE_WEBHOOKS": "false",
  "USE_SNIPPETS": "false",
  "USE_INTEGRATIONS": "false",
  "USE_WIKI": "false"
}
```

### Example Prompts

```
Show me all open MRs in the k8s-agent project.

Create an issue "OOMKilled in production namespace" with label "bug".

What is the status of the latest pipeline on the main branch?

Show me the values.yaml in the Helm chart repo.
```

---

## Prometheus MCP

**Package:** `prometheus-mcp-server` (v1.x)
**Tools:** 5 tools

### Authentication (Basic Auth)

`prometheus-mcp-server` supports Basic Auth natively via environment variables:

```jsonc
"environment": {
  "PROMETHEUS_URL": "https://prometheus.example.com",
  "PROMETHEUS_USERNAME": "monitoring-user",
  "PROMETHEUS_PASSWORD": "secret"
}
```

Additional auth options:

| Variable | Description |
|---|---|
| `PROMETHEUS_USERNAME` | Basic Auth username |
| `PROMETHEUS_PASSWORD` | Basic Auth password |
| `PROMETHEUS_TOKEN` | Bearer token (alternative to Basic Auth) |
| `PROMETHEUS_INSECURE` | `true` to ignore self-signed TLS certs |

### Tools

| Tool | Description |
|---|---|
| `prom_query` | PromQL instant query (current value) |
| `prom_range` | PromQL range query (time series / trends) |
| `prom_discover` | List available metrics |
| `prom_metadata` | Metric type and description |
| `prom_targets` | Scrape target status and health |

### Example Prompts

```
Which pods are consuming the most memory right now in the production namespace?

Show me the CPU load of all nodes over the last 2 hours.

Which scrape targets are currently down?

Are there metrics for the ingress controller?
```

---

## Alertmanager MCP

There are multiple implementations with different auth options:

| Project | Auth | Language | Installation |
|---|---|---|---|
| `jeanlopezxyz/mcp-alertmanager` | K8s auto-connect (kubeconfig), no Basic Auth | Go | `npx` |
| `ntk148v/alertmanager-mcp-server` | Basic Auth via env variables, pagination | Python | `uv` + clone |
| `zekker6/mcp-alertmanager` | Basic Auth via CLI flag, arbitrary headers (Bearer) | Go | Build binary |

---

### Option A: `jeanlopezxyz/mcp-alertmanager` — K8s Auto-Connect (no Ingress needed)

**Recommended when:** Alertmanager runs in the same cluster and kubeconfig is available.

**Not suitable for:** Access via Ingress with Basic Auth.

```jsonc
"alertmanager": {
  "type": "local",
  "command": [
    "npx", "-y", "mcp-alertmanager@latest",
    "--namespace", "monitoring",
    "--service", "alertmanager-operated",
    "--service-port", "9093",
    "--service-scheme", "http"
  ],
  "enabled": true
}
```

**Tools (12):** `getAlerts`, `getCriticalAlerts`, `getAlertingSummary`, `getAlertGroups`,
`investigateAlert`, `correlateAlerts`, `getAlertHistory`, `getSilences`,
`createSilence`, `deleteSilence`, `getAlertmanagerStatus`, `getReceivers`

---

### Option B: `ntk148v/alertmanager-mcp-server` — Basic Auth via Env (Python)

**Recommended when:** Access via Ingress with Basic Auth, or Keycloak token as Bearer.

**Prerequisites:** Python 3.12+, [uv](https://github.com/astral-sh/uv)

```bash
git clone https://github.com/ntk148v/alertmanager-mcp-server.git /opt/alertmanager-mcp-server
cd /opt/alertmanager-mcp-server && make setup
```

```jsonc
"alertmanager": {
  "type": "local",
  "command": [
    "uv", "run",
    "--directory", "/opt/alertmanager-mcp-server",
    "src/alertmanager_mcp_server/server.py"
  ],
  "enabled": true,
  "environment": {
    "ALERTMANAGER_URL": "https://alertmanager.example.com",
    "ALERTMANAGER_USERNAME": "admin",
    "ALERTMANAGER_PASSWORD": "secret"
    // Optional for Keycloak Bearer token instead of Basic Auth:
    // Token must be fetched externally and passed as ALERTMANAGER_PASSWORD
    // (no native OAuth flow, but Bearer via header is possible)
  }
}
```

**Notable feature: Pagination** — returns alerts in pages (default: 10 per page) to
avoid context overflow with many alerts.

**Tools (8):** `get_status`, `get_alerts` (paginated), `get_silences` (paginated),
`post_silence`, `delete_silence`, `get_receivers`, `get_alert_groups` (paginated),
`create_alert`

---

### Option C: `zekker6/mcp-alertmanager` — Basic Auth + Custom Headers (Go Binary)

**Recommended when:** Go binary preferred, no Python desired, or Bearer token
(e.g. Keycloak) needed via `-header` flag.

**Prerequisites:** Go 1.22+, [task](https://taskfile.dev) build tool

```bash
git clone https://github.com/zekker6/mcp-alertmanager.git /opt/zekker6-alertmanager-mcp
cd /opt/zekker6-alertmanager-mcp && task build
cp bin/mcp-alertmanager /usr/local/bin/
```

Basic Auth via password file (more secure than plaintext):
```bash
echo -n "secret-password" > /etc/alertmanager-mcp/password
chmod 600 /etc/alertmanager-mcp/password
```

```jsonc
// Basic Auth:
"alertmanager": {
  "type": "local",
  "command": [
    "/usr/local/bin/mcp-alertmanager",
    "-url", "https://alertmanager.example.com",
    "-username", "admin",
    "-password-file", "/etc/alertmanager-mcp/password"
  ],
  "enabled": true
}

// Alternatively: Keycloak Bearer token via header:
"alertmanager": {
  "type": "local",
  "command": [
    "/usr/local/bin/mcp-alertmanager",
    "-url", "https://alertmanager.example.com",
    "-header", "Authorization: Bearer <keycloak-access-token>"
  ],
  "enabled": true
}
```

> **Keycloak note:** The token must be fetched externally and entered manually —
> no automatic OAuth flow. For automatic token renewal, a wrapper script would be needed.

**Tools (5):** `list_alerts`, `list_silences`, `get_silence`, `create_silence`, `delete_silence`

### Example Prompts

```
Which alerts are firing right now?

Are there critical alerts in the production namespace?

Give me a summary of all active alerts by severity.

Investigate the "KubePodCrashLooping" alert — what could be the cause?

Find correlated alerts to identify the root cause.

Create a 2-hour silence for the "PodCrashLooping" alert in the staging namespace.
```

---

## Note: Token Consumption

Every active MCP server adds its tool definitions to the context on **every request** —
regardless of whether the tools are actually used. Approximate values:

| MCP Server | Tools | Approx. token overhead |
|---|---|---|
| GitLab (full) | 44 | ~8,000–12,000 |
| GitLab (reduced) | ~20 | ~4,000–6,000 |
| Prometheus | 5 | ~500–800 |
| Alertmanager (jeanlopezxyz) | 12 | ~1,500–2,000 |
| Alertmanager (ntk148v) | 8 | ~1,000–1,500 |
| Alertmanager (zekker6) | 5 | ~500–800 |

Recommendation: Disable unused GitLab feature flags and only enable MCP servers
when needed for the current task.
