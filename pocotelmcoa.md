# MCOA: what is GA vs not, and how to use it
Implementation: [stolostron/multicluster-observability-addon](https://github.com/stolostron/multicluster-observability-addon)
## Support status
| Signal | MCOA status | How you turn it on |
|---|---|---|
| **Metrics** (platform and user workloads) | **GA** | `spec.capabilities.platform.metrics` and `spec.capabilities.userWorkloads.metrics` |
| **Logs** (ClusterLogForwarder) | **Not GA** | `spec.capabilities.*.logs` |
| **Traces / OpenTelemetry** (collector + instrumentation) | **Not GA** | `spec.capabilities.userWorkloads.traces` |
Do not install MCOA from GitHub, Kustomize, Helm, or `make addon-deploy`.  
ACM’s `multicluster-observability-operator` deploys the addon manager when you set capabilities on `MultiClusterObservability`.
Logs, traces, and OpenTelemetry in MCOA are preview/experimental API surfaces. They are not a supported GA ACM feature. Use them only in labs. For production logging and tracing, use the supported OpenShift operators on each cluster (Cluster Logging, Tempo, Red Hat build of OpenTelemetry) until those MCOA capabilities go GA.
---
## What MCOA actually does
MCOA is not a store. Thanos (ACM Observability) still holds **metrics**. Loki/Tempo/CloudWatch are out of band.
When capabilities are enabled, MCO:
1. Deploys `multicluster-observability-addon-manager` in `open-cluster-management-observability`
2. Creates `ClusterManagementAddOn/multicluster-observability-addon`
3. Installs the addon on managed clusters via addon-manager (no `ManagedClusterAddOn` for you to create)
| Capability | Spoke operators / resources |
|---|---|
| Metrics (GA) | Prometheus Agent + ScrapeConfig + PrometheusRule (COO / `monitoring.rhobs`) |
| Logs (not GA) | Cluster Logging Operator + copied `ClusterLogForwarder` |
| Traces (not GA) | Red Hat build of OpenTelemetry + copied `OpenTelemetryCollector` / `Instrumentation` |
---
## Part 1 — GA: metrics
### Prerequisites
- ACM Observability already running (`MultiClusterObservability/observability`)
- Cluster Observability Operator available (ACM docs require it for this add-on path)
- Managed clusters in a ManagedClusterSet
### Enable
Platform metrics are required; user-workload metrics are optional.
```bash
oc patch mco observability --type=merge -p '{
  "spec": {
    "capabilities": {
      "platform": {
        "metrics": { "default": { "enabled": true } }
      },
      "userWorkloads": {
        "metrics": { "default": { "enabled": true } }
      }
    }
  }
}'
```
After this, MCO **stops** deploying the legacy metrics-collector and uses MCOA Prometheus Agents instead.
### Verify (you did not install anything)
```bash
oc -n open-cluster-management-observability get deploy | grep mcoa
oc get cma multicluster-observability-addon
oc get prometheusagents,scrapeconfigs,prometheusrules -n open-cluster-management-observability
oc get managedclusteraddon -A | grep observability
```
Default hub configs (also listed in the [MCOA README](https://github.com/stolostron/multicluster-observability-addon/blob/main/README.md)):
- `PrometheusAgent/acm-platform-metrics-collector-default`
- `ScrapeConfig/platform-metrics-default`
- `PrometheusRule/platform-rules-default`
- `PrometheusAgent/acm-user-workload-metrics-collector-default` (if user workloads enabled)
Further GA procedures (custom metrics, relabel, remote-write, alerts) are in the ACM Observability docs, section **Multicluster observability add-on**.
---
## Part 2 — Not GA: logs, traces, OpenTelemetry
Treat this as a lab. Same addon, already running from Part 1 (or from any capabilities flag). Still **do not install** the addon.
Your original CR enables the non-GA signals and leaves the collector off:
```yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  capabilities:
    platform:
      logs:
        collection:
          enabled: true          # not GA
    userWorkloads:
      logs:
        collection:
          clusterLogForwarder:
            enabled: true        # not GA
      traces:
        collection:
          instrumentation:
            enabled: true        # not GA
          openTelemetryCollector:
            enabled: false       # not GA; false = no spoke collector
```
Flags only install spoke operators and wait for hub stanzas. They do not create Loki, Tempo, or a pipeline.
If you experiment with traces, set:
```yaml
openTelemetryCollector:
  enabled: true
```
Otherwise instrumentation has nowhere local to export.
### Stanzas MCOA copies (from the repo)
Create these **on the hub** in `open-cluster-management-observability`, **name: `instance`**, `managementState: unmanaged` so the hub operators do not run them. MCOA copies `spec` to spokes.
CMA `placements[].configs` should reference (README):
```yaml
- group: observability.openshift.io
  resource: clusterlogforwarders
  name: instance
  namespace: open-cluster-management-observability
- group: opentelemetry.io
  resource: opentelemetrycollectors
  name: instance
  namespace: open-cluster-management-observability
- group: opentelemetry.io
  resource: instrumentations
  name: instance
  namespace: open-cluster-management-observability
```
You still create:
1. The stanza CRs
2. Secrets for outputs/exporters (stanza namespace or spoke cluster namespace on the hub)
Repo samples: [`hack/addon-install/`](https://github.com/stolostron/multicluster-observability-addon/tree/main/hack/addon-install)
On the spoke, the collector is `OpenTelemetryCollector/mcoa-instance` in `mcoa-opentelemetry`. Apps send to:
```
http://mcoa-instance-collector.mcoa-opentelemetry.svc.cluster.local:4318
```
### Collector stanza (not GA)
From [`otelcol-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/otelcol-instance.yaml). Point `endpoint` at **your** OTLP store (hub collector Route, Tempo, …). MCOA does not create that store.
```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: instance
  namespace: open-cluster-management-observability
spec:
  managementState: unmanaged
  mode: deployment
  config:
    receivers:
      otlp:
        protocols:
          grpc: {}
          http: {}
    processors: {}
    exporters:
      debug: {}
      otlp:
        endpoint: HUB_OTLP_HOST:443
        tls:
          insecure: false
          ca_file: /tracing-otlp-auth/ca-bundle.crt
          cert_file: /tracing-otlp-auth/tls.crt
          key_file: /tracing-otlp-auth/tls.key
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [otlp, debug]
  volumeMounts:
    - name: tracing-otlp-auth
      mountPath: /tracing-otlp-auth
  volumes:
    - name: tracing-otlp-auth
      secret:
        secretName: tracing-otlp-auth
```
```bash
oc -n open-cluster-management-observability create secret generic tracing-otlp-auth \
  --from-file=tls.crt=client.crt \
  --from-file=tls.key=client.key \
  --from-file=ca-bundle.crt=ca.crt
```
### Instrumentation stanza (not GA)
From [`instrumentation-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/instrumentation-instance.yaml):
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: instance
  namespace: open-cluster-management-observability
spec:
  exporter:
    endpoint: http://mcoa-instance-collector.mcoa-opentelemetry.svc.cluster.local:4318
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  propagators:
    - jaeger
    - b3
```
Workloads must opt in on the **pod template**:
```yaml
annotations:
  instrumentation.opentelemetry.io/inject-java: "true"
```
### Log stanza (not GA)
From [`clf-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/clf-instance.yaml). Spoke SA is `openshift-logging/mcoa-logcollector`.
Without a `ClusterLogForwarder/instance`, log flags do nothing.
### Lab verify
```bash
oc get cma multicluster-observability-addon -o yaml
oc -n <spoke> get managedclusteraddon,manifestwork
# on spoke:
oc -n mcoa-opentelemetry get opentelemetrycollector,instrumentation,pods,svc
```
---
## Do not
- `kubectl apply -k deploy/` or `make addon-deploy` on a supported ACM hub
- Expect ACM Grafana/Thanos to show MCOA traces or logs
- Ship logs/traces via these capabilities in production while they are not GA
## Do
- Use MCOA **metrics** capabilities for GA fleet metrics
- Leave log/trace/OTEL capabilities off unless you are explicitly preview-testing
- For supported logs/traces today: configure CLO / Tempo / OTEL on the clusters themselves, not via MCOA
