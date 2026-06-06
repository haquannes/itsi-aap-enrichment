# ITSI → AAP Closed-Loop Enrichment

When Splunk **ITSI** opens an episode for a failing workload, a NEAP action launches an
**AAP** job template running [`enrich_diagnostics.yml`](enrich_diagnostics.yml). The playbook
queries the **OpenShift API** for platform evidence (no SSH to the broken node — it never
starts), builds a root-cause summary, and writes it back to Splunk via **HEC** into
`otel_logs` (`sourcetype=itsi:aap:enrichment`). The enrichment then shows up in the ITSI
episode drilldown / Event Data Search.

```
ITSI episode (NEAP action)
   └─► AAP job_templates/<id>/launch/  (extra_vars: episode_id, kpi, severity, namespace, workload…)
          └─► enrich_diagnostics.yml  →  OCP API (deployment image:tag, events, pod status)
                 └─► Splunk HEC  →  otel_logs / itsi:aap:enrichment  (keyed by episode_id)
                        └─► ITSI episode drilldown
```

## Parameters (passed by ITSI as `extra_vars`)

| var | meaning | default |
|-----|---------|---------|
| `episode_id` | ITSI episode/group id — correlation key | `MANUAL-TEST` |
| `itsi_service` | service name | `AAP Managed Nodes` |
| `kpi` | triggering KPI | `Image Pull Failure Rate` |
| `severity` | episode severity | `critical` |
| `namespace` | OCP namespace | `ansible-targets` |
| `workload` | deployment name | `ubuntu-target` |

## Secrets (NOT passed by ITSI — held in AAP)

Provided to the job via environment / AAP credential, never hard-coded:

- `OCP_API`, `OCP_TOKEN` — OpenShift API access (AAP execution env reaches the internal API)
- `HEC_TOKEN` — Splunk HEC token for `otel_logs`

## Manual test

```bash
HEC_TOKEN=xxxx OCP_TOKEN=sha256~xxxx OCP_API=https://api...:6443 \
  ansible-playbook enrich_diagnostics.yml \
  -e episode_id=DEMO-123 -e workload=ubuntu-target -e namespace=ansible-targets
```
