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

## Ablageort der Konfiguration

Die `opencode.json` kann an zwei Orten liegen — beide werden zusammengeführt:

| Ort | Zweck | Git-tauglich |
|---|---|---|
| `~/.config/opencode/opencode.json` | Global für alle Projekte | Nein (enthält Secrets) |
| `<projektverzeichnis>/opencode.json` | Projektspezifisch | Ja (nur ohne Secrets) |

**Empfehlung:** MCP-Server mit Tokens in die **globale Config**. Secrets nie im Klartext,
sondern per Variablen-Substitution:

```jsonc
// Env-Variable:
"GITLAB_TOKEN": "{env:GITLAB_TOKEN}"

// Datei (z.B. chmod 600):
"PROMETHEUS_PASSWORD": "{file:~/.secrets/prometheus-password}"
```

---

## Konfiguration in opencode.json

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

    // --- Prometheus (Basic Auth) ---
    "prometheus": {
      "type": "local",
      "command": ["npx", "-y", "prometheus-mcp-server"],
      "enabled": true,
      "environment": {
        "PROMETHEUS_URL": "https://prometheus.example.com",
        "PROMETHEUS_USERNAME": "monitoring-user",
        "PROMETHEUS_PASSWORD": "secret",
        // Optional: self-signed certs ignorieren
        // "PROMETHEUS_INSECURE": "true"
      }
    },

    // --- Alertmanager ---
    // Drei Optionen — siehe Alertmanager-Abschnitt weiter unten.
    //
    // Option A: K8s-Auto-Connect (kein Basic Auth nötig, empfohlen im Cluster)
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
    // Option C: Basic Auth / Bearer Token via CLI-Flag (Go Binary, zekker6)
    // "alertmanager": {
    //   "type": "local",
    //   "command": ["/usr/local/bin/mcp-alertmanager",
    //               "-url", "https://alertmanager.example.com",
    //               "-username", "admin",
    //               "-password-file", "/etc/alertmanager-mcp/password"],
    //   "enabled": true
    // }

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

### Authentifizierung (Basic Auth)

`prometheus-mcp-server` unterstützt Basic Auth nativ via Umgebungsvariablen:

```jsonc
"environment": {
  "PROMETHEUS_URL": "https://prometheus.example.com",
  "PROMETHEUS_USERNAME": "monitoring-user",
  "PROMETHEUS_PASSWORD": "secret"
}
```

Weitere Auth-Optionen:

| Variable | Beschreibung |
|---|---|
| `PROMETHEUS_USERNAME` | Basic Auth Benutzername |
| `PROMETHEUS_PASSWORD` | Basic Auth Passwort |
| `PROMETHEUS_TOKEN` | Bearer Token (alternativ zu Basic Auth) |
| `PROMETHEUS_INSECURE` | `true` um self-signed TLS-Certs zu ignorieren |

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

Es gibt mehrere Implementierungen mit unterschiedlichen Auth-Optionen:

| Projekt | Auth | Sprache | Installation |
|---|---|---|---|
| `jeanlopezxyz/mcp-alertmanager` | K8s-Auto-Connect (kubeconfig), kein Basic Auth | Go | `npx` |
| `ntk148v/alertmanager-mcp-server` | Basic Auth via Env-Variablen, Pagination | Python | `uv` + clone |
| `zekker6/mcp-alertmanager` | Basic Auth via CLI-Flag, beliebige Headers (Bearer) | Go | Binary selbst bauen |

---

### Option A: `jeanlopezxyz/mcp-alertmanager` — K8s-Auto-Connect (kein Ingress nötig)

**Empfohlen wenn:** Alertmanager läuft im gleichen Cluster und kubeconfig verfügbar ist.

**Nicht geeignet für:** Zugriff über Ingress mit Basic Auth.

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

**Empfohlen wenn:** Zugriff über Ingress mit Basic Auth, oder Keycloak-Token als Bearer.

**Voraussetzungen:** Python 3.12+, [uv](https://github.com/astral-sh/uv)

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
    // Optional für Keycloak Bearer Token statt Basic Auth:
    // Token muss extern geholt und als ALERTMANAGER_PASSWORD übergeben werden
    // (kein natives OAuth-Flow, aber Bearer via Header möglich)
  }
}
```

**Besonderheit: Pagination** — gibt Alerts seitenweise zurück (Standard: 10 pro Seite)
um Context-Overflow bei vielen Alerts zu vermeiden.

**Tools (8):** `get_status`, `get_alerts` (paginiert), `get_silences` (paginiert),
`post_silence`, `delete_silence`, `get_receivers`, `get_alert_groups` (paginiert),
`create_alert`

---

### Option C: `zekker6/mcp-alertmanager` — Basic Auth + custom Headers (Go Binary)

**Empfohlen wenn:** Go-Binary bevorzugt, kein Python gewünscht, oder Bearer Token
(z.B. Keycloak) via `-header` Flag nötig.

**Voraussetzungen:** Go 1.22+, [task](https://taskfile.dev) Build-Tool

```bash
git clone https://github.com/zekker6/mcp-alertmanager.git /opt/zekker6-alertmanager-mcp
cd /opt/zekker6-alertmanager-mcp && task build
cp bin/mcp-alertmanager /usr/local/bin/
```

Basic Auth via Password-File (sicherer als Klartext):
```bash
echo -n "geheimes-passwort" > /etc/alertmanager-mcp/password
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

// Alternativ: Keycloak Bearer Token via Header:
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

> **Hinweis Keycloak:** Der Token muss extern beschafft und manuell eingetragen
> werden — kein automatischer OAuth-Flow. Für automatische Token-Erneuerung
> wäre ein Wrapper-Script nötig.

**Tools (5):** `list_alerts`, `list_silences`, `get_silence`, `create_silence`, `delete_silence`

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
| Alertmanager (jeanlopezxyz) | 12 | ~1.500–2.000 |
| Alertmanager (ntk148v) | 8 | ~1.000–1.500 |
| Alertmanager (zekker6) | 5 | ~500–800 |

Empfehlung: Nicht benötigte GitLab Feature Flags deaktivieren und MCP Server
nur aktivieren wenn sie für die aktuelle Aufgabe gebraucht werden.
