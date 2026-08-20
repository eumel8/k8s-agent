---
name: k8s-incident-response
description: Structured playbook for Kubernetes production incidents including diagnosis, escalation, and communication
license: MIT
compatibility: opencode
metadata:
  audience: devops
  workflow: incident-response
---

# Kubernetes Incident Response Playbook

## Severity Classification

| Severity | Criteria | Response Time |
|---|---|---|
| **P1 — Critical** | Production down, data loss risk, all users affected | Immediate |
| **P2 — High** | Major feature broken, >50% users affected, degraded performance | < 15 min |
| **P3 — Medium** | Minor feature broken, workaround available | < 2h |
| **P4 — Low** | Cosmetic issues, single user affected | Next business day |

---

## Step 1 — Confirm the Incident

Before anything else, verify the problem is real and understand scope:

```bash
# Cluster health
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# Recent events across all namespaces
kubectl get events -A --sort-by='.lastTimestamp' | tail -30

# Any firing alerts?
# → use alertmanager MCP: getAlertingSummary
```

Ask:
- Is this affecting production, staging, or both?
- Since when? (check `kubectl get events` timestamps)
- Is it getting worse, stable, or recovering?

---

## Step 2 — Identify the Blast Radius

```bash
# Which namespaces are affected?
kubectl get pods -A | grep -v Running | grep -v Completed

# Which deployments have unavailable replicas?
kubectl get deployments -A | awk '$4 != $3'

# Node pressure?
kubectl describe nodes | grep -A5 "Conditions:"
kubectl top nodes
```

---

## Step 3 — Diagnose Root Cause

### Pod-level issues

```bash
# CrashLoopBackOff / OOMKilled / Error
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous --tail=100

# What changed recently?
kubectl rollout history deployment/<name> -n <ns>
```

### Node-level issues

```bash
kubectl describe node <node>
# Look for: MemoryPressure, DiskPressure, PIDPressure, NotReady

# Check kubelet
# kubectl debug node/<node> -it --image=busybox
```

### Network / Service issues

```bash
kubectl get endpoints -n <ns>
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -qO- http://<svc>.<ns>.svc.cluster.local
```

### Use MCP tools for deeper insight

```
# Prometheus: resource trends leading up to incident
prom_range: container_memory_usage_bytes, last 2h

# Alertmanager: correlate alerts to find root cause
correlateAlerts, investigateAlert
```

---

## Step 4 — Immediate Mitigation

Choose the least destructive option first:

| Symptom | Mitigation |
|---|---|
| Bad deployment | `kubectl rollout undo deployment/<name> -n <ns>` |
| OOMKilled | `kubectl patch deployment <name> -n <ns> --type=merge -p '{"spec":{"template":{"spec":{"containers":[{"name":"<c>","resources":{"limits":{"memory":"512Mi"}}}]}}}}'` |
| Node NotReady | Cordon + drain (confirm with user first!) |
| ImagePullBackOff | Fix imagePullSecrets or image tag |
| High load | Scale up replicas temporarily |

**Always confirm with user before:**
- `kubectl drain` / `kubectl cordon`
- Deleting namespace-scoped or cluster-scoped resources
- Modifying `kube-system`

---

## Step 5 — Verify Recovery

```bash
# Watch pods come back
kubectl get pods -n <ns> -w

# Rollout status
kubectl rollout status deployment/<name> -n <ns>

# Endpoints healthy?
kubectl get endpoints -n <ns>

# Alerts cleared?
# → use alertmanager MCP: getAlertingSummary
```

---

## Step 6 — Communicate

### During incident (P1/P2)

- Update stakeholders every **15 minutes** with:
  - Current status (investigating / mitigating / monitoring)
  - Impact description
  - ETA if known

### After resolution

Post an incident summary with:
1. **Timeline** — when detected, when mitigated, when resolved
2. **Root cause** — what actually happened
3. **Impact** — duration, affected users/services
4. **Mitigation** — what fixed it
5. **Follow-up actions** — what prevents recurrence

```
# Create GitLab issue for follow-up
# → use gitlab MCP: manage_issues
# Title: "[Post-Incident] <short description> <date>"
# Label: incident, post-mortem
```

---

## Common Runbooks

### CrashLoopBackOff
1. `kubectl logs <pod> -n <ns> --previous` — read the actual error
2. Check exit code in `kubectl describe pod`
3. Exit 137 = OOMKilled → increase memory limit
4. Exit 1/2 = application error → check config, secrets, env vars
5. Exit 0 = process exits cleanly → check liveness probe

### Pod Stuck Pending
1. `kubectl describe pod <pod> -n <ns>` → check Events section
2. Insufficient resources → `kubectl describe node`, check allocatable vs requested
3. PVC not bound → `kubectl get pvc -n <ns>`
4. Node selector / affinity mismatch → check pod spec vs node labels
5. Tolerations missing → check node taints

### Node NotReady
1. `kubectl describe node <node>` → check Conditions
2. DiskPressure → clean up or expand disk
3. MemoryPressure → evict or scale down workloads
4. NetworkPlugin not ready → check CNI pods in kube-system
5. Kubelet stopped → requires node-level access (SSH/console)
