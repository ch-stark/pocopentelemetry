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
