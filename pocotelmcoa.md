# MCOA tutorial (logs + traces)
Source of truth: [stolostron/multicluster-observability-addon](https://github.com/stolostron/multicluster-observability-addon)
MCOA is an Open Cluster Management addon. It does **not** store signals. It:
1. Installs signal operators on managed clusters
2. Copies hub “stanza” CRs onto those clusters
3. Ships exporter/output secrets with those CRs
| Signal | Operator MCOA installs on the spoke | Hub stanza MCOA copies |
|---|---|---|
| Logs | Cluster Logging Operator | `ClusterLogForwarder` named `instance` |
| Traces | Red Hat build of OpenTelemetry | `OpenTelemetryCollector` named `instance` |
| Auto-instrumentation | same OTEL operator | `Instrumentation` named `instance` |
| Metrics | requires Multicluster Observability Operator | `PrometheusAgent` / `ScrapeConfig` / `PrometheusRule` |
You still provide the **central store** (Loki, CloudWatch, Tempo, a hub OpenTelemetry Collector, …). MCOA only forwards to whatever you put in the stanza.
Official sample CRs live in [`hack/addon-install/`](https://github.com/stolostron/multicluster-observability-addon/tree/main/hack/addon-install).
---
## How MCOA is wired
```
Hub
  MultiClusterObservability.spec.capabilities   (ACM 2.12+)
       or AddOnDeploymentConfig.customizedVariables
            |
            v
  ClusterManagementAddOn/multicluster-observability-addon
       placements[].configs  -->  stanza CRs in open-cluster-management-observability
            |
            v
Spoke
  ManagedClusterAddOn + ManifestWork
  OpenTelemetryCollector/mcoa-instance in namespace mcoa-opentelemetry
  Instrumentation/mcoa-instance
  ClusterLogForwarder (spec copied; SA set by MCOA)
```
Required stanza names/namespaces (from the [README](https://github.com/stolostron/multicluster-observability-addon/blob/main/README.md)):
```yaml
configs:
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
On the spoke, MCOA deploys the collector as `mcoa-instance` in `mcoa-opentelemetry`. Apps send OTLP to:
```
http://mcoa-instance-collector.mcoa-opentelemetry.svc.cluster.local:4318
```
That endpoint is hard-coded in [`hack/addon-install/templates/instrumentation-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/instrumentation-instance.yaml).
---
## Prerequisites
- Hub with OCM/ACM and at least one managed cluster in a ManagedClusterSet
- ACM 2.12+ if you install MCOA through `MultiClusterObservability.spec.capabilities`
- `cluster-admin` on the hub
- A **trace store** reachable from spokes (hub OpenTelemetry Collector Route is what the repo sample uses)
- A **log store** if you enable logs (the repo sample uses CloudWatch)
- `oc` on the hub unless a step says otherwise
Hub operators you typically need so the stanza CRDs exist:
- Red Hat build of OpenTelemetry (create stanzas with `managementState: unmanaged`)
- Cluster Logging Operator CRDs if you enable logs
MCOA installs the spoke operators. Do not rely on the hub OTEL operator to run the stanza; keep it `unmanaged`.
---
## Step 1 — Enable MCOA
### Path A — ACM / MultiClusterObservability (usual)
This is the [README “Installing via MCO”](https://github.com/stolostron/multicluster-observability-addon/blob/main/README.md) path. Patch the existing MCO; do not replace storage config.
**Change from the README sample:** set `openTelemetryCollector.enabled: true`. The README leaves it `false`, which never deploys a spoke collector.
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
          enabled: true
    userWorkloads:
      logs:
        collection:
          clusterLogForwarder:
            enabled: true
      traces:
        collection:
          instrumentation:
            enabled: true
          openTelemetryCollector:
            enabled: true
```
```bash
oc patch mco observability --type=merge -p '{
  "spec": {
    "capabilities": {
      "platform": {
        "logs": { "collection": { "enabled": true } }
      },
      "userWorkloads": {
        "logs": {
          "collection": { "clusterLogForwarder": { "enabled": true } }
        },
        "traces": {
          "collection": {
            "instrumentation": { "enabled": true },
            "openTelemetryCollector": { "enabled": true }
          }
        }
      }
    }
  }
}'
```
Custom addon image (from the README):
```yaml
metadata:
  annotations:
    mco-multicluster_observability_addon-image: quay.io/YOUR_ORG/multicluster-observability-addon:YOUR_TAG
```
### Path B — Install the addon from this repo
From [README “Installing via Kustomize”](https://github.com/stolostron/multicluster-observability-addon/blob/main/README.md) and [CONTRIBUTING.md](https://github.com/stolostron/multicluster-observability-addon/blob/main/CONTRIBUTING.md):
```bash
git clone https://github.com/stolostron/multicluster-observability-addon.git
cd multicluster-observability-addon
make install-crds
kubectl apply -k deploy/
```
Dev loop:
```bash
export REGISTRY_BASE=quay.io/YOUR_QUAY_ID
oc create namespace open-cluster-management-observability
make oci
make addon-deploy
# after a rebuild:
oc -n open-cluster-management-observability delete pod -l app=multicluster-observability-addon-manager
```
Then enable signals with `AddOnDeploymentConfig` ([CONTRIBUTING.md](https://github.com/stolostron/multicluster-observability-addon/blob/main/CONTRIBUTING.md)):
```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: AddOnDeploymentConfig
metadata:
  name: multicluster-observability-addon
  namespace: open-cluster-management-observability
spec:
  customizedVariables:
    - name: platformLogsCollection
      value: clusterlogforwarders.v1.observability.openshift.io
    - name: userWorkloadLogsCollection
      value: clusterlogforwarders.v1.observability.openshift.io
    - name: userWorkloadTracesCollection
      value: opentelemetrycollectors.v1beta1.opentelemetry.io
    - name: userWorkloadInstrumentation
      value: instrumentations.v1alpha1.opentelemetry.io
```
Multiple collectors on one key are semicolon-separated, e.g. logs via CLF **and** OTEL:
```yaml
- name: userWorkloadLogsCollection
  value: clusterlogforwarders.v1.observability.openshift.io;opentelemetrycollectors.v1beta1.opentelemetry.io
```
### Confirm the addon exists
```bash
kubectl get ClusterManagementAddOn multicluster-observability-addon
oc -n open-cluster-management-observability get deploy | grep mcoa
```
MCOA is installed on spokes by the addon-manager. Clusters only need to belong to a ManagedClusterSet. Change target clusters in `ClusterManagementAddOn.spec.installStrategy`.
---
## Step 2 — Confirm CMA points at the stanzas
```bash
oc get cma multicluster-observability-addon -o yaml
```
`spec.installStrategy.placements[].configs` must include the three resources from the README (`clusterlogforwarders/instance`, `opentelemetrycollectors/instance`, `instrumentations/instance` in `open-cluster-management-observability`). Add them if ACM did not.
The addon does **not** start forwarding until those CRs exist. From CONTRIBUTING.md, you create:
1. Stanzas for `ClusterLogForwarder` and `OpenTelemetryCollector` (and `Instrumentation` if enabled)
2. Secrets for Outputs/Exporters — in the stanza namespace **or** in the spoke’s hub namespace (`<managed-cluster-name>`)
---
## Step 3 — Create exporter secrets
### Traces — `tracing-otlp-auth`
The sample collector mounts this secret ([`otelcol-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/otelcol-instance.yaml), [`otelcol-secret.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/otelcol-secret.yaml)):
| Key | Use |
|---|---|
| `tls.crt` | client cert |
| `tls.key` | client key |
| `ca-bundle.crt` | CA that signed the hub collector cert |
Create it in `open-cluster-management-observability` (copied to every selected spoke) **or** in the spoke cluster namespace on the hub (that spoke only). The helm sample uses the spoke namespace:
```bash
oc -n open-cluster-management-observability create secret generic tracing-otlp-auth \
  --from-file=tls.crt=client.crt \
  --from-file=tls.key=client.key \
  --from-file=ca-bundle.crt=ca.crt
```
`hack/addon-install/values.yaml` fields: `hubCollector.route`, `otelCollectorSecret.tlsKey`, `tlsCrt`, `caCrt`.
### Logs — `aws-credentials` (repo sample)
The CLF sample sends infrastructure logs to CloudWatch. Create `aws-credentials` with `aws_access_key_id` and `aws_secret_access_key`, or change the CLF output to Loki/Kafka/etc. MCOA supports every CLO output; it ships the secrets you reference.
Spoke CLF ServiceAccount is **`openshift-logging/mcoa-logcollector`** (required for AWS STS).
---
## Step 4 — Create hub stanzas (`name: instance`)
Use `managementState: unmanaged` on the hub so the hub operators do not reconcile these CRs. MCOA copies `spec` to the spoke and sets the spoke copy to managed ([comment in the repo](https://github.com/stolostron/multicluster-observability-addon/commit/4d2fbc37cb22aaeaeb97d46460885f1cc8be3946)).
### OpenTelemetryCollector
From [`hack/addon-install/templates/otelcol-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/otelcol-instance.yaml). Replace `HUB_OTLP_HOST:443` with your hub collector Route (that is `hubCollector.route` in values.yaml).
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
      jaeger:
        protocols:
          grpc: {}
      otlp:
        protocols:
          grpc: {}
          http: {}
    processors: {}
    exporters:
      debug: {}
      otlp:
        endpoint: HUB_OTLP_HOST:443
        headers:
          x-scope-orgid: spoke-1
        tls:
          insecure: false
          ca_file: /tracing-otlp-auth/ca-bundle.crt
          cert_file: /tracing-otlp-auth/tls.crt
          key_file: /tracing-otlp-auth/tls.key
    service:
      pipelines:
        traces:
          receivers: [jaeger, otlp]
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
MCOA supports all collector components in the OpenShift OTEL docs. Auth is whatever the collector supports; MCOA just ships the secrets.
### Instrumentation
From [`hack/addon-install/templates/instrumentation-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/instrumentation-instance.yaml). The exporter is the **spoke** collector, not Tempo.
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
  python:
    env:
      - name: OTEL_LOG_LEVEL
        value: "debug"
      - name: OTEL_TRACES_EXPORTER
        value: otlp
      - name: OTEL_METRICS_EXPORTER
        value: none
```
### ClusterLogForwarder (if logs stay enabled)
From [`hack/addon-install/templates/clf-instance.yaml`](https://github.com/stolostron/multicluster-observability-addon/blob/main/hack/addon-install/templates/clf-instance.yaml). Spoke spec is an exact copy except `serviceAccountName`, which MCOA sets.
```yaml
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: open-cluster-management-observability
spec:
  managementState: Unmanaged
  serviceAccount:
    name: mcoa-logcollector
  outputs:
    - name: cw
      type: cloudwatch
      cloudwatch:
        region: eu-central-1
        groupName: mcoa-logs
        authentication:
          type: awsAccessKey
          awsAccessKey:
            keyId:
              secretName: aws-credentials
              key: aws_access_key_id
            keySecret:
              secretName: aws-credentials
              key: aws_secret_access_key
  pipelines:
    - name: infra-cw
      inputRefs:
        - infrastructure
      outputRefs:
        - cw
```
### Optional: apply the repo helm chart instead of YAML
```bash
# fill hack/addon-install/values.yaml first
helm -n default upgrade --install addon-install hack/addon-install/
```
---
## Step 5 — Hub collector Route (what `hubCollector.route` is)
MCOA does not create this. The sample exporter is a **hub OpenTelemetry Collector** exposed on a Route, with mTLS using `tracing-otlp-auth`. That hub collector then writes to Tempo or another backend.
You can point `exporters.otlp.endpoint` at any OTLP store the spoke can reach. The repo’s contract is: stanza exporter + secrets.
---
## Step 6 — Verify MCOA rolled out to spokes
Hub:
```bash
oc get managedclusteraddon -A | grep observability
oc -n <spoke-cluster-name> get manifestwork
oc -n <spoke-cluster-name> get manifestworks addon-multicluster-observability-addon-deploy-0
oc -n <spoke-cluster-name> describe managedclusteraddon multicluster-observability-addon
```
Spoke:
```bash
oc get csv -A | grep -iE 'opentelemetry|cluster-logging'
oc -n mcoa-opentelemetry get opentelemetrycollector,instrumentation,pods,svc
```
Expect:
- `OpenTelemetryCollector/mcoa-instance` (from the tracing chart)
- `Instrumentation/mcoa-instance`
- Service `mcoa-instance-collector` on 4317/4318
MCOA deploys a **single** OpenTelemetryCollector and a **single** ClusterLogForwarder per spoke, templated from the hub stanza.
---
## Step 7 — Opt in workloads
Instrumentation is opt-in. Annotate `spec.template.metadata`:
```yaml
annotations:
  instrumentation.opentelemetry.io/inject-java: "true"
  # inject-nodejs / inject-python / inject-dotnet / inject-go
```
Or send OTLP yourself to `mcoa-instance-collector.mcoa-opentelemetry.svc:4317` / `:4318`.
---
## Step 8 — Confirm signals leave the spoke
```bash
oc -n mcoa-opentelemetry logs -l app.kubernetes.io/name=mcoa-instance-collector --tail=100
```
Then check the store behind `hubCollector.route` (hub collector logs, Tempo, CloudWatch, …).
---
## Capabilities cheat sheet
| MCO field | AddOnDeploymentConfig key | Spoke effect |
|---|---|---|
| `platform.logs.collection.enabled` | `platformLogsCollection=clusterlogforwarders.v1.observability.openshift.io` | CLO + CLF for platform logs |
| `userWorkloads.logs.collection.clusterLogForwarder.enabled` | `userWorkloadLogsCollection=...clusterlogforwarders...` | CLF for app logs |
| `userWorkloads.traces.collection.openTelemetryCollector.enabled` | `userWorkloadTracesCollection=opentelemetrycollectors.v1beta1.opentelemetry.io` | OTEL operator + collector instance |
| `userWorkloads.traces.collection.instrumentation.enabled` | `userWorkloadInstrumentation=instrumentations.v1alpha1.opentelemetry.io` | Instrumentation CR |
Enabling a flag without the matching `instance` CR does not forward data.
---
## Troubleshooting
| Symptom | Cause in MCOA terms |
|---|---|
| CMA exists, nothing on spoke | Stanza `instance` missing, or not referenced in `placements[].configs` |
| No collector pods | `openTelemetryCollector.enabled: false` (README default) |
| Hub spins up `instance-collector` | Hub stanza not `managementState: unmanaged` |
| Addon Degraded | Missing `tracing-otlp-auth` / output secret, or bad collector config |
| Instrumented pods have no init container | Annotation not on the pod template |
| Logs not flowing | CLF stanza missing, or SA not `mcoa-logcollector` for STS |
```bash
oc get addondeploymentconfig -n open-cluster-management-observability -o yaml
oc -n <spoke> get manifestwork -o yaml | grep -A30 opentelemetry
```
---
## What “done” means for MCOA
1. `ClusterManagementAddOn/multicluster-observability-addon` exists
2. Capabilities or `AddOnDeploymentConfig` enable logs/traces/instrumentation
3. Hub CRs named `instance` exist in `open-cluster-management-observability`
4. Exporter secrets exist and are referenced by those CRs
5. `ManagedClusterAddOn` is Available and ManifestWorks applied
6. Spoke has `mcoa-instance` in `mcoa-opentelemetry`
7. Workloads export OTLP (annotation or SDK) to the spoke collector
