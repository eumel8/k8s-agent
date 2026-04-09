# MCP Server für den Kubernetes DevOps Agent

## Überblick

Für diesen Agenten sind drei MCP Server relevant, die alle **lokal** betrieben werden und
**keine externen Cloud-Dienste** benötigen:

| MCP Server | Zweck | Package |
|---|---|---|
| GitLab | Issues, MRs, Pipelines, CI/CD | `@structured-world/gitlab-mcp` |
| Prometheus | Metrics abfragen, Targets prüfen | `prometheus-mcp-server` |
| Alertmanager | Alerts, Silences, Root-cause | `mcp-alertmanager` |

---

## Voraussetzungen

- Node.js >= 24 (für GitLab und Prometheus MCP)
- Zugang zum internen GitLab Server
- GitLab Personal Access Token (Scope: `api` oder `read_api`)
- Prometheus und Alertmanager erreichbar (URL oder via kubeconfig)

---

## Konfiguration in opencode.jsonc

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {

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

    // --- Prometheus ---
    "prometheus": {
      "type": "local",
      "command": ["npx", "-y", "prometheus-mcp-server"],
      "enabled": true,
      "environment": {
        "PROMETHEUS_URL": "http://prometheus.monitoring.svc.cluster.local:9090",
        // Optional: Bearer Token wenn Prometheus auth-geschützt ist
        // "PROMETHEUS_TOKEN": "Bearer eyJ...",
        // Optional: self-signed certs ignorieren
        // "PROMETHEUS_INSECURE": "true"
      }
    },

    // --- Alertmanager ---
    // Verbindet sich automatisch via kubeconfig (kein ALERTMANAGER_URL nötig)
    // Default: openshift-monitoring/alertmanager-operated:9093
    "alertmanager": {
      "type": "local",
      "command": ["npx", "-y", "mcp-alertmanager@latest"],
      "enabled": true,
      // Alternativ direkte URL statt K8s-Auto-Connect:
      // "environment": {
      //   "ALERTMANAGER_URL": "http://alertmanager.monitoring.svc.cluster.local:9093"
      // }
    }

  }
}
```

---

## GitLab MCP

**Package:** `@structured-world/gitlab-mcp` (v7.x)
**Tools:** 44 Tools über 18 Entity-Typen

### Wichtigste Tools

| Tool | Beschreibung |
|---|---|
| `browse_projects` | Projekte suchen und auflisten |
| `browse_merge_requests` | MRs filtern (state, author, ...) |
| `manage_issues` | Issues erstellen, lesen, schließen |
| `browse_pipelines` | Pipeline-Status und Logs |
| `browse_files` | Dateien im Repository lesen |

### Feature Flags (selektiv deaktivieren)

Nicht benötigte Tool-Gruppen lassen sich per Umgebungsvariable abschalten,
um den Token-Verbrauch zu reduzieren:

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

### Beispiel-Prompts

```
Zeig mir alle offenen MRs im Projekt k8s-agent.

Erstelle ein Issue "OOMKilled in production namespace" mit Label "bug".

Was ist der Status der letzten Pipeline auf dem main Branch?

Zeig mir die values.yaml im Helm-Chart Repo.
```

---

## Prometheus MCP

**Package:** `prometheus-mcp-server` (v1.x)
**Tools:** 5 Tools

### Tools

| Tool | Beschreibung |
|---|---|
| `prom_query` | PromQL Instant Query (aktueller Wert) |
| `prom_range` | PromQL Range Query (Zeitreihe / Trends) |
| `prom_discover` | Verfügbare Metrics auflisten |
| `prom_metadata` | Metric-Typ und Beschreibung |
| `prom_targets` | Scrape-Target Status und Health |

### Beispiel-Prompts

```
Welche Pods verbrauchen gerade am meisten Memory im Namespace production?

Zeig mir die CPU-Last aller Nodes der letzten 2 Stunden.

Welche Scrape-Targets sind aktuell down?

Gibt es Metrics für den Ingress-Controller?
```

---

## Alertmanager MCP

**Package:** `mcp-alertmanager` (v2.x, Go-Binary)
**Tools:** 12 Tools

Der Alertmanager MCP verbindet sich **nativ via kubeconfig** ohne `kubectl port-forward`.
Standardmäßig wird `openshift-monitoring/alertmanager-operated:9093` gesucht.
Für andere Namespaces oder Services CLI-Flags verwenden:

```jsonc
"command": [
  "npx", "-y", "mcp-alertmanager@latest",
  "--namespace", "monitoring",
  "--service", "alertmanager-operated",
  "--service-port", "9093",
  "--service-scheme", "http"
]
```

### Tools

| Tool | Beschreibung |
|---|---|
| `getAlerts` | Alle aktiven Alerts (mit Filtern) |
| `getCriticalAlerts` | Nur Critical-Alerts |
| `getAlertingSummary` | Übersicht: Anzahl nach Severity |
| `getAlertGroups` | Alerts gruppiert nach Routing |
| `investigateAlert` | Deep-dive in einen einzelnen Alert |
| `correlateAlerts` | Korrelierte Alerts zur Root-cause-Analyse |
| `getAlertHistory` | Alert-Historie und Analyse |
| `getSilences` | Aktive Silences auflisten |
| `createSilence` | Silence anlegen |
| `deleteSilence` | Silence löschen |
| `getAlertmanagerStatus` | Server-Status und Cluster-Info |
| `getReceivers` | Konfigurierte Notification-Receiver |

### Beispiel-Prompts

```
Welche Alerts feuern gerade?

Gibt es kritische Alerts im Namespace production?

Gib mir eine Zusammenfassung aller aktiven Alerts nach Severity.

Untersuche den Alert "KubePodCrashLooping" — was könnte die Ursache sein?

Finde korrelierte Alerts um die Root Cause zu identifizieren.

Lege eine 2-Stunden Silence für den Alert "PodCrashLooping" im Namespace staging an.
```

---

## Hinweis: Token-Verbrauch

Jeder aktive MCP Server fügt seine Tool-Definitionen bei **jedem Request** zum Kontext
hinzu — unabhängig davon ob die Tools genutzt werden. Richtwerte:

| MCP Server | Tools | ca. Token overhead |
|---|---|---|
| GitLab (voll) | 44 | ~8.000–12.000 |
| GitLab (reduziert) | ~20 | ~4.000–6.000 |
| Prometheus | 5 | ~500–800 |
| Alertmanager | 12 | ~1.500–2.000 |

Empfehlung: Nicht benötigte GitLab Feature Flags deaktivieren und MCP Server
nur aktivieren wenn sie für die aktuelle Aufgabe gebraucht werden.
