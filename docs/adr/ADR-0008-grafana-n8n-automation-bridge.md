# ADR-0008: Grafana → n8n Automation Bridge (Logs + Metrics)

**Date:** 2025-11-10  
**Status:** Approved  
**Deciders:** Owner  
**Supersedes:** None  
**Relates to:** ADR-0006 (Observability Core), ADR-0007 (Observability Stack)

---

## Context
With Grafana serving as the central alerting and visualization layer for the observability stack, we need a standardized mechanism to route actionable alerts into the automation platform (**n8n**) for further orchestration and remediation.

The goal is to enable **alert-driven automation** (from both log and metric alerts) while preserving auditability and separation of concerns:
- Grafana = detection and decision layer  for Loki (logs) and Prometheus (metrics)
- n8n = execution layer  

---

## Decision
Adopt a **Grafana → n8n automation bridge** using **Grafana’s webhook alert channel**, supporting alerts from both **Loki** and **Prometheus**, to trigger **n8n** workflows.

### Automation Flow
```
Prometheus / Loki → Grafana (alert rule triggers)
        ↓
Grafana Alert Webhook → n8n (HTTP Webhook node)
        ↓
n8n Workflow executes:
  - Generate reports or GitHub issues
  - Apply updates, restarts, or notifications
  - Log structured results back into Loki
```
Alerts may originate from log-based conditions (Loki) or metric thresholds (Prometheus).
Grafana’s unified alerting system handles both paths transparently through the same webhook integration.

---

## Implementation Overview
1. **Webhook Channel:**  
   - Each alert rule in Grafana is linked to a dedicated n8n webhook URL.  
   - Payload includes alert metadata, rule name, and labels.
   - Works uniformly for alerts sourced from Loki or Prometheus.

2. **n8n Workflow Handling:**  
   - Parse incoming payload and identify the alert source (Loki or Prometheus).  
   - Determine workflow type (update, notification, diff report, etc.).  
   - Execute defined automation.  
   - Emit structured log entry back to Loki (via HTTP output or local logging).

3. **Security:**  
   - All webhooks are protected by Cloudflare Zero Trust Access.  
   - Optional API key validation in n8n.  
   - No direct unauthenticated access to n8n endpoints.

4. **Traceability:**  
   - Each automation action links back to the originating Grafana alert ID.  
   - Logs stored in Loki for full audit chain (alert → automation → result).

---

## Consequences
- Grafana remains the single point deciding *what triggers automation*.  
- n8n becomes the only executor for post-alert actions.  
- Alert misconfiguration or accidental triggering cannot directly impact runtime services without passing through observability rules.  
- All automation activity is automatically observable via the same stack.
- Unified alerting allows automation from both infrastructure metrics and application logs, enabling proactive and reactive workflows within the same pipeline.

---

## Notes
This pattern supports future extensions such as:
- Two-way feedback (n8n reporting resolution back to Grafana).  
- Alert classification and prioritization for selective automation.  
- Integration with GitHub Projects for workflow tracking.
- Future support for trace-based triggers (Tempo) may be added once adopted in the observability core.
