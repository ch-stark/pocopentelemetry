# MCOA + resource flap detection
Sources:
- Addon behaviour: [stolostron/multicluster-observability-addon](https://github.com/stolostron/multicluster-observability-addon)
- Collector pipeline (review of Michael Lang’s tutorial): [resource-flap-detection](https://github.com/michaelalang/otc-example-configurations/tree/main/resource-flap-detection) and [multi-cluster](https://github.com/michaelalang/otc-example-configurations/tree/main/resource-flap-detection/multi-cluster)
**This flap-detection scenario is unsupported by Red Hat** (same disclaimer as the upstream example).
Do **not** install MCOA yourself (`kubectl apply -k deploy/`, Helm, `make addon-deploy`). The `multicluster-observability-operator` deploys the addon when you set `MultiClusterObservability.spec.capabilities`.
Do **not** apply `otc.yml` onto spokes with `oc create -f`. On an ACM hub, that collector spec is a **hub stanza** that MCOA copies.
Do **not** use Grafana. If you graph the metric, use **Observe → Dashboards (Perses)**. This tutorial does not create a Perses dashboard.
---
## Support status
| Signal | MCOA | Use here |
|---|---|---|
| Metrics (platform / user workloads) | **GA** | Optional; separate from flap metrics |
| Logs (`ClusterLogForwarder`) | **Not GA** | **Off** — Lang’s example forbids CLO so CLF and OTC do not race for logs |
| Traces / instrumentation | **Not GA** | **Off** — this pipeline is audit logs → metrics, not app traces |
| OpenTelemetry Collector | **Not GA** | **On** — only path used for flap detection |
---
## What Lang’s example does
From the [README](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/README.md):
1. **No** Cluster Logging / `ClusterLogForwarder` on the cluster.
2. Privileged-ish SA + SCC so the collector can read host audit logs (not for production).
3. Permissions **first**, collector **second**.
4. `OpenTelemetryCollector` as a **DaemonSet**:
   - Standalone / east / central: masters only (`nodeSelector: node-role.kubernetes.io/master`), files `/var/log/kube-apiserver/audit.log` and `/var/log/openshift-apiserver/audit.log`.
   - ACM hub with Hosted Control Planes: all nodes, extra `filelog/hcpaudit` on pod audit logs ([`multi-cluster/acm/otc.yml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/multi-cluster/acm/otc.yml)). HCP guest clusters are **not** deployed separately; the hosting cluster tails them.
5. Pipeline: `filelog` → parse JSON audit → keep `update`/`patch` → `count` connector → metric **`k8s_resource_flapping`** → **Prometheus OTLP** (`/api/v1/otlp`).
6. Multi-cluster: each cluster writes **directly** to Prometheus. `CLUSTERNAME` is an env on the OTC (`acm`, `east`, `central`, …).
7. Prometheus must be started with:
```
--web.enable-otlp-receiver
--enable-feature=otlp-deltatocumulative
```
Upstream visualizes with a Grafana JSON. **Ignore that.** Use Perses in the OpenShift console if you need a graph; do not import those JSON files.
```
host audit.log
    → filelog (DaemonSet)
    → transform/audit_parser + filter (update|patch)
    → count/flapping → k8s_resource_flapping
    → otlphttp/prometheusremotewrite → Prometheus /api/v1/otlp
    → optional: Observe → Dashboards (Perses)
```
---
## Prerequisites
- ACM hub with Observability (`MultiClusterObservability/observability`)
- Managed clusters in a ManagedClusterSet
- `cluster-admin`
- Prometheus that accepts OTLP at `/api/v1/otlp` (flags above)
- Cluster Observability Operator if you want Perses in the console (UI only; no dashboard CR in this tutorial)
- Red Hat build of OpenTelemetry CRDs on the **hub** so you can create an unmanaged stanza (hub operator must not run that CR)
---
## Step 1 — Enable MCOA (capabilities only)
Patch the existing MCO. Do not recreate storage.
```yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  capabilities:
    platform:
      metrics:
        default:
          enabled: true          # GA; omit if you are not switching metrics to MCOA
    userWorkloads:
      traces:
        collection:
          instrumentation:
            enabled: false
          openTelemetryCollector:
            enabled: true        # not GA; required for this lab
```
Leave **all log/CLF capabilities false**. Lang’s tutorial requires no CLO.
```bash
oc get cma multicluster-observability-addon
oc -n open-cluster-management-observability get deploy | grep mcoa
```
CMA `placements[].configs` must include:
```yaml
- group: opentelemetry.io
  resource: opentelemetrycollectors
  name: instance
  namespace: open-cluster-management-observability
```
Do not add `clusterlogforwarders` or `instrumentations` for this lab.
---
## Step 2 — Permissions on each spoke (Lang order)
Lang: `oc create -k resource-flap-detection` **before** the OTC.
MCOA does **not** copy SCC, SA, or RoleBindings. Apply the kustomize set from the example on every spoke (Policy is fine), but **rebind namespace** from `openshift-logging` to **`mcoa-opentelemetry`**.
Files: [`kustomization.yaml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/kustomization.yaml) (`namespace.yml`, `sa.yml` `clfotlp`, `scc.yml`, `scc-binding.yml`, `clusterrole.yml`, `rolebinding.yml` collect-*-logs, `attrrole-binding.yml`).
Change every subject/namespace `openshift-logging` → `mcoa-opentelemetry`. Create namespace `mcoa-opentelemetry` if the addon has not yet.
Collector must not start until this exists (same reason as Lang’s two-step apply).
---
## Step 3 — Hub OTC stanza (do not `oc create -f otc.yml` on spokes)
Copy **spec** from:
- Standalone / non-HCP spokes: [`multi-cluster/east/otc.yml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/multi-cluster/east/otc.yml) (or root [`otc.yml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/otc.yml))
- Hub / HCP host: [`multi-cluster/acm/otc.yml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/multi-cluster/acm/otc.yml)
MCOA identity (only metadata + managementState change vs Lang):
```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: instance
  namespace: open-cluster-management-observability
spec:
  managementState: unmanaged
  # paste spec from east/otc.yml or acm/otc.yml below
```
Keep from the example:
- `mode: daemonset`
- `serviceAccount: clfotlp`
- `securityContext` (`runAsUser: 0`, hostPath volumes, master tolerations / nodeSelector)
- `filelog/audit` (and `filelog/hcpaudit` only on the ACM/HCP spec)
- processors, `count/flapping`, `otlphttp/prometheusremotewrite`
Set the exporter to your Prometheus:
```yaml
exporters:
  otlphttp/prometheusremotewrite:
    endpoint: 'https://prometheus.apps.example.com/api/v1/otlp'
```
**`CLUSTERNAME`:** Lang sets it per cluster in `env`. MCOA copies **one** spec to all clusters in a placement. Use separate placements/stanzas per cluster set, or accept a shared name. Do not expect MCOA to template `east` vs `acm`.
Spoke object is `OpenTelemetryCollector/mcoa-instance` in `mcoa-opentelemetry`, not `flapper` in `openshift-logging`.
---
## Step 4 — Verify
Hub:
```bash
oc get opentelemetrycollector instance -n open-cluster-management-observability
oc -n <spoke> get managedclusteraddon,manifestwork
```
Spoke:
```bash
oc -n mcoa-opentelemetry get ds,opentelemetrycollector,sa
oc -n mcoa-opentelemetry logs -l app.kubernetes.io/name=mcoa-instance-collector --tail=50
```
Prometheus: series `k8s_resource_flapping` or `k8s_resource_flapping_total` (OTLP often adds `_total`).
---
## Visualization (Perses only, no dashboard in this tutorial)
Lang’s repo ships Grafana JSON. **Do not import it.**
On the hub, if COO monitoring Perses is enabled (`UIPlugin` `spec.monitoring.perses.enabled: true`), open **Observe → Dashboards (Perses)**. Query the Prometheus that receives OTLP. Building a `PersesDashboard` is out of scope here.
Useful PromQL (from Lang’s multi-cluster dashboard idea, for ad-hoc query):
```promql
sum by (cluster) (increase(k8s_resource_flapping_total[$__rate_interval]))
sum by (resource_name, source_user) (
  changes(k8s_resource_flapping_total{replica="0"}[1h])
)
```
---
## Cleanup
Disable `openTelemetryCollector` on the MCO (or remove the hub `instance` CR). Delete spoke Policy/SCC/SA. Do not leave hostPath audit collectors running in production.
Lang’s local cleanup (`oc delete -f otc.yml` then `oc delete -k`) applies only if you deployed the example **without** MCOA. With MCOA, delete the hub stanza and spoke RBAC instead.
---
## Mapping Lang → MCOA
| Lang tutorial | This ACM path |
|---|---|
| `oc create -k` then `oc create -f otc.yml` on each cluster | kustomize (retargeted) on spokes, then **one hub** `OpenTelemetryCollector/instance` |
| `metadata.name: flapper`, `openshift-logging` | `instance` in `open-cluster-management-observability` → spoke `mcoa-instance` / `mcoa-opentelemetry` |
| Install OTC operator yourself | MCOA installs OTEL operator on spokes when the collector capability is on |
| Grafana JSON | Perses console; no dashboard created here |
| CLO / CLF | Must stay off |




# Fleet audit-to-metrics with MCOA and OpenTelemetry (lab)
How to run [resource-flap-detection](https://github.com/michaelalang/otc-example-configurations/tree/main/resource-flap-detection) through ACM’s MultiCluster Observability Addon (MCOA) instead of applying a collector to every cluster by hand.
**This is a lab.** The example is unsupported. MCOA **metrics** are GA. MCOA **logs, traces, and OpenTelemetry** are **not GA**. Do not use this pattern in production.
Visualization, if any, is **Observe → Dashboards (Perses)**. This article does not ship a dashboard. Do not use Grafana.
---
## What MCOA is
[MCOA](https://github.com/stolostron/multicluster-observability-addon) is not a metrics/logs/traces store. It is the addon ACM uses to install signal operators on managed clusters and to copy hub configuration (“stanzas”) onto those clusters.
You **do not install the addon**. `multicluster-observability-operator` deploys it when you set `spec.capabilities` on `MultiClusterObservability`.
| Capability | Status | This lab |
|---|---|---|
| Platform / user-workload **metrics** | **GA** | Optional; independent of flap metrics |
| **Logs** (`ClusterLogForwarder`) | Not GA | **Off** |
| **Traces / instrumentation** | Not GA | **Off** |
| **OpenTelemetryCollector** | Not GA | **On** |
The example **must not** run Cluster Logging. A `ClusterLogForwarder` and this collector would compete for the same log files.
---
## What the example does
Repo: [otc-example-configurations/resource-flap-detection](https://github.com/michaelalang/otc-example-configurations/tree/main/resource-flap-detection)  
Multi-cluster layout: [resource-flap-detection/multi-cluster](https://github.com/michaelalang/otc-example-configurations/tree/main/resource-flap-detection/multi-cluster)
It detects **resource flapping**: two controllers or CI systems racing `update`/`patch` on the same object. It does that by reading **kube-apiserver and openshift-apiserver audit logs**, not by instrumenting apps.
```
audit.log on the node
  → OpenTelemetry Collector (DaemonSet)
  → parse JSON, keep verb=update|patch
  → count connector → metric k8s_resource_flapping
  → OTLP HTTP → Prometheus /api/v1/otlp
```
Collector shape:
- **Standalone / typical spoke:** DaemonSet on **control-plane** nodes; hostPath `/var/log/kube-apiserver` and `/var/log/openshift-apiserver`. Spec: [`east/otc.yml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/multi-cluster/east/otc.yml) or root [`otc.yml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/otc.yml).
- **ACM hub hosting HCPs:** DaemonSet on **all** nodes; extra receiver for hosted control-plane pod audit logs. Spec: [`acm/otc.yml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/multi-cluster/acm/otc.yml). Guest HCP clusters are **not** given their own collector.
Each cluster remote-writes **directly** to Prometheus. `CLUSTERNAME` is an env var on the collector (`acm`, `east`, `central`, …).
Prometheus must enable:
```
--web.enable-otlp-receiver
--enable-feature=otlp-deltatocumulative
```
The repo applies **permissions first**, then the `OpenTelemetryCollector`. Same order with MCOA: spoke SCC/SA/RBAC before the copied collector can schedule.
The collector ServiceAccount is privileged enough to read host audit logs. Treat that as lab-only.
---
## How MCOA changes the example
Do **not** run `oc create -f otc.yml` on spokes.
| Example (per cluster) | ACM hub |
|---|---|
| `OpenTelemetryCollector` `flapper` in `openshift-logging` | Hub stanza `instance` in `open-cluster-management-observability` |
| Operator installed locally | MCOA installs Red Hat build of OpenTelemetry on the spoke |
| `oc create -k` then `oc create -f otc.yml` | Same kustomize **on the spoke** (retargeted), stanza **on the hub** |
| Grafana JSON in the repo | Ignore it. Use Perses in the console if you query the metric |
MCOA copies the hub `spec` to the spoke as `mcoa-instance` in namespace **`mcoa-opentelemetry`**.
---
## Procedure
### 1. Capabilities (no addon install)
Patch the existing `MultiClusterObservability`. Keep log and instrumentation flags off.
```yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  capabilities:
    platform:
      metrics:
        default:
          enabled: true    # GA; omit if you are not moving metrics to MCOA
    userWorkloads:
      traces:
        collection:
          instrumentation:
            enabled: false
          openTelemetryCollector:
            enabled: true  # not GA
```
Confirm MCO created the addon:
```bash
oc get cma multicluster-observability-addon
```
`placements[].configs` must reference `opentelemetrycollectors` / `instance` / `open-cluster-management-observability`. Do not add ClusterLogForwarder or Instrumentation configs for this lab.
### 2. Spoke permissions (before the collector)
MCOA does not copy SCC, ServiceAccount, or RoleBindings. Apply the example kustomize from [`resource-flap-detection/kustomization.yaml`](https://github.com/michaelalang/otc-example-configurations/blob/main/resource-flap-detection/kustomization.yaml) on **each spoke** (Policy is fine).
Retarget every `openshift-logging` namespace/subject to **`mcoa-opentelemetry`**. Keep SA name `clfotlp` so it matches `spec.serviceAccount` in the collector.
That set includes hostPath SCC, collect-audit-logs bindings, and k8s attribute read rights. Without it, the DaemonSet cannot read audit files.
### 3. Hub stanza
Create CRDs on the hub by having the OpenTelemetry operator installed there, but set **`managementState: unmanaged`** so the hub does not run this collector.
```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: instance
  namespace: open-cluster-management-observability
spec:
  managementState: unmanaged
  # remainder: copy spec from east/otc.yml or acm/otc.yml
```
Keep from the example: `mode: daemonset`, `serviceAccount: clfotlp`, securityContext, hostPath volumes, filelog receivers, transform/filter, `count/flapping`, `otlphttp/prometheusremotewrite`.
Point the exporter at your Prometheus:
```yaml
exporters:
  otlphttp/prometheusremotewrite:
    endpoint: 'https://prometheus.apps.example.com/api/v1/otlp'
```
**Cluster name:** the example sets `CLUSTERNAME` per cluster. MCOA copies **one** spec per placement. Use different placements/stanzas per cluster set, or accept a shared label.
### 4. Check rollout
```bash
oc get opentelemetrycollector instance -n open-cluster-management-observability
oc -n <spoke> get managedclusteraddon,manifestwork
# on the spoke
oc -n mcoa-opentelemetry get ds,opentelemetrycollector,sa
oc -n mcoa-opentelemetry logs -l app.kubernetes.io/name=mcoa-instance-collector --tail=50
```
Prometheus should show `k8s_resource_flapping` or `k8s_resource_flapping_total` (OTLP often appends `_total`).
### 5. Perses (optional, no dashboard here)
If Cluster Observability Operator has the monitoring UI plugin with Perses enabled, open **Observe → Dashboards (Perses)** and query the Prometheus that receives OTLP. Do not import the example’s Grafana JSON. Do not apply a `PersesDashboard` as part of this procedure.
Ad-hoc PromQL:
```promql
sum by (cluster) (increase(k8s_resource_flapping_total[$__rate_interval]))
sum by (resource_name, source_user) (
  changes(k8s_resource_flapping_total{replica="0"}[1h])
)
```
---
## Cleanup
Remove the hub `OpenTelemetryCollector/instance` (or set `openTelemetryCollector.enabled: false`). Delete spoke SCC/SA/bindings. Do not leave hostPath audit DaemonSets in production.
---
## Takeaways
1. Enable MCOA only via `MultiClusterObservability.spec.capabilities`.
2. GA = fleet **metrics**. This lab uses the **non-GA** OpenTelemetry collector capability only.
3. The example is audit logs → Prometheus, not Tempo traces and not ClusterLogForwarder.
4. Hub stanza `instance` + spoke RBAC in `mcoa-opentelemetry`; never `oc apply` the example collector as `flapper`/`openshift-logging` on an MCOA spoke.
5. Graph in **Perses** if you want; this write-up does not create a dashboard.
