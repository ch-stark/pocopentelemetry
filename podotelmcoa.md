# MCOA OpenTelemetry traces — step-by-step tutorial

End-to-end path for **MCOA traces**: spoke apps → spoke OpenTelemetry Collector → hub ingest collector → Tempo. ACM metrics storage (Thanos) is not used for traces.

The original CR with `openTelemetryCollector.enabled: false` stops after installing operators. This tutorial turns the collector on and wires the missing CRs.

## Architecture

```
Spoke app  --OTLP-->  Spoke collector (mcoa-opentelemetry)
                         |
                         | OTLP + mTLS
                         v
                   Hub Route  -->  Hub ingest collector  -->  TempoStack
```

MCOA copies hub “stanza” CRs onto managed clusters. Names matter:

| Role | Hub CR | Spoke result |
|---|---|---|
| Collector stanza | `OpenTelemetryCollector/instance` in `open-cluster-management-observability` | `mcoa-instance` in `mcoa-opentelemetry` |
| Instrumentation stanza | `Instrumentation/instance` in `open-cluster-management-observability` | Copied to the spoke |

Workloads on the spoke send to:

`http://mcoa-instance-collector.mcoa-opentelemetry.svc.cluster.local:4318`

## Prerequisites

- ACM 2.12+ with Observability already enabled (`MultiClusterObservability/observability` exists)
- At least one managed cluster in a ManagedClusterSet
- `cluster-admin` on the hub
- Object storage for Tempo (S3, MinIO, ODF, GCS, or Azure). You can reuse the same bucket as Thanos with a different prefix
- `oc` logged into the **hub** unless a step says otherwise

Replace these placeholders as you go:

| Placeholder | Example |
|---|---|
| `TEMPO_NS` | `tempo` |
| `TENANT` | `acm` |
| `HUB_APPS_DOMAIN` | `apps.hub.example.com` |
| `SPOKE` | managed cluster name, e.g. `cluster-east` |

---

## Step 1 — Confirm Observability is healthy

```bash
oc get mco observability -o yaml
oc get pods -n open-cluster-management-observability
oc get managedcluster
```

You should already have the observability namespace and managed clusters. Do not recreate the `MultiClusterObservability` CR from scratch if it exists; patch it later.

---

## Step 2 — Install operators on the hub

Install from OperatorHub (all namespaces, automatic updates):

1. **Tempo Operator** (Red Hat OpenShift distributed tracing platform)
2. **Red Hat build of OpenTelemetry** — needed so hub CRDs exist. Hub stanzas must stay `unmanaged` so this operator does not run them locally
3. Optional: **Cluster Observability Operator** if you want **Observe → Traces** in the console

Wait until CSVs are `Succeeded`:

```bash
oc get csv -A | grep -iE 'tempo|opentelemetry'
```

MCOA installs the OpenTelemetry operator on **spokes**. You do not install it there yourself.

---

## Step 3 — Create the Tempo namespace and object-storage secret

```bash
oc new-project tempo
```

Create a secret that matches your storage. S3/MinIO example:

```bash
oc -n tempo create secret generic tempo-s3 \
  --from-literal=bucket=tempo-traces \
  --from-literal=endpoint=https://s3.example.com \
  --from-literal=access_key_id='YOUR_KEY' \
  --from-literal=access_key_secret='YOUR_SECRET' \
  --from-literal=region=us-east-1
```

Use the field names your Tempo Operator version expects (`endpoint`, `bucket`, keys). If you already have `thanos-object-storage` in `open-cluster-management-observability`, copy that pattern rather than inventing a new schema.

---

## Step 4 — Deploy TempoStack on the hub

```yaml
# tempo-stack.yaml
apiVersion: tempo.grafana.com/v1alpha1
kind: TempoStack
metadata:
  name: acm
  namespace: tempo
spec:
  storage:
    secret:
      name: tempo-s3
      type: s3
  storageSize: 10Gi
  resources:
    total:
      limits:
        memory: 4Gi
        cpu: "2"
  tenants:
    mode: openshift
    authentication:
      - tenantName: acm
        tenantId: acm
  template:
    gateway:
      enabled: true
    queryFrontend:
      jaegerQuery:
        enabled: true
```

```bash
oc apply -f tempo-stack.yaml
oc -n tempo get tempostack acm -w
oc -n tempo get pods
```

Wait until gateway, distributor, ingester, querier, and query-frontend are running.

---

## Step 5 — Allow a hub collector to write traces

Spoke collectors cannot use OpenShift Tempo gateway auth (that is SAR against hub service accounts). A **hub ingest collector** writes to Tempo with a hub ServiceAccount.

```yaml
# tempo-write-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-hub-writer
  namespace: tempo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempostack-traces-write
rules:
  - apiGroups: ["tempo.grafana.com"]
    resources: ["tempostacks"]
    resourceNames: ["acm"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tempostack-traces-write-otel-hub
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tempostack-traces-write
subjects:
  - kind: ServiceAccount
    name: otel-hub-writer
    namespace: tempo
```

Tempo’s exact write ClusterRole name can differ by operator version. If ingest fails with 403, check the Tempo docs for `traces` create on `tempo.grafana.com` and bind that role to `otel-hub-writer`.

```bash
oc apply -f tempo-write-rbac.yaml
```

---

## Step 6 — Create mTLS certs for spoke → hub ingest

Generate a small CA, a server cert for the hub Route, and a client cert for spokes.

```bash
mkdir -p /tmp/otel-mtls && cd /tmp/otel-mtls

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 825 \
  -subj "/CN=mcoa-otel-ca" -out ca.crt

# Server cert for the hub Route host
export HUB_OTLP_HOST="otel-gateway-collector-tempo.${HUB_APPS_DOMAIN}"

openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=${HUB_OTLP_HOST}" -out server.csr
cat > server.ext <<EOF
subjectAltName=DNS:${HUB_OTLP_HOST}
extendedKeyUsage=serverAuth
EOF
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 825 -sha256 -extfile server.ext

# Client cert used by spoke collectors
openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/CN=mcoa-spoke-collector" -out client.csr
cat > client.ext <
