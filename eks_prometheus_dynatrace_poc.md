# EKS Prometheus to Dynatrace — Discovery Findings and Technical PoC Proposal

**Status:** Proposed technical PoC — pending environment validation  
**Date:** 28 October 2026  
**Platform:** Amazon EKS on EC2  
**Audience:** Platform Engineering, Observability, SRE, Application Engineering and Security

## 1. Executive summary

The EKS environment currently runs Prometheus (reported build version **3.12.0**) and already has a Dynatrace installation. A targeted discovery identified JVM, process, HTTP, Kubernetes state and node CPU metrics. Container-level cAdvisor CPU/memory and Istio request metrics were not found in the tested queries.

**Recommendation:** Keep the existing Prometheus and Dynatrace deployments unchanged initially. Validate native Dynatrace Kubernetes coverage, then deploy a **single-replica OpenTelemetry Collector** in a dedicated PoC namespace to scrape **one approved JVM application's Prometheus endpoint** and export a small metric allowlist to Dynatrace. Compare the same workloads and time windows across both platforms. Do not enable broad cluster-wide scraping, duplicate instrumentation or full-stack monitoring changes without a separate review.

### Intended outcomes

1. Document actual Prometheus jobs, targets, labels and metric inventory.
2. Establish useful Prometheus dashboard queries for EKS infrastructure and JVM/HTTP applications.
3. Confirm existing Dynatrace Operator, ActiveGate and OneAgent modes and native EKS coverage.
4. Demonstrate selected Prometheus metrics arriving in Dynatrace through OpenTelemetry.
5. Assess metric parity, missing visibility, duplication, cardinality, operational overhead and security.
6. Produce a production rollout recommendation, not an automatic Prometheus decommissioning decision.

## 2. Confirmed findings and gaps

The following metric names have returned data in Prometheus, as reported during discovery:

| Domain | Metric | Discovery status | Intended use |
|---|---|---|---|
| Process | `process_resident_memory_bytes` | Available | Resident process memory |
| Process | `process_cpu_usage` | Available | Process CPU usage; verify metric type and units |
| JVM | `jvm_memory_used_bytes` | Available | Heap/non-heap memory |
| JVM | `jvm_gc_pause_seconds_count` | Available | GC pause event rate |
| HTTP | `http_server_requests_seconds_count` | Available | Request throughput and response-code rates |
| Kubernetes | `kube_pod_status_phase` | Available | Running, Pending, Failed etc. |
| Kubernetes | `kube_pod_status_ready` | Available | Pod readiness |
| Kubernetes | `kube_pod_container_status_restarts_total` | Available | Restart increases |
| Kubernetes | `kube_pod_container_status_last_terminated_reason` | Available | Last recorded termination reason |
| Kubernetes | `kube_pod_container_resource_limits` | Available | Configured CPU/memory limits |
| Node | `node_cpu_seconds_total` | Available | Node CPU utilisation |
| Container | `container_cpu_usage_seconds_total` | No data in test | Container CPU usage gap |
| Container | `container_memory_working_set_bytes` | No data in test | Container memory usage gap |
| Service mesh | `istio_requests_total` | No data in test | Istio traffic gap, if Istio is in scope |

**Interpretation:** “No data” is not proof that the exporter or mesh is absent. Confirm scrape configuration, endpoint health, metric naming, time range and label filters. The HTTP request-rate and HTTP error-rate tests refer to the **same underlying metric**, not separate metrics.

### Unknowns requiring validation

- EKS version, worker-node OS, Prometheus namespace/service/Helm chart and scrape configuration.
- Actual Prometheus job names, target endpoints and HTTP label schema (`status`, `outcome`, `uri`, `method`, etc.).
- Existing Dynatrace Operator version, DynaKube monitoring mode, ActiveGate health, Kubernetes coverage and tenant endpoint/authentication method.
- Whether the chosen JVM application exposes `/actuator/prometheus`, which port it uses and whether the Collector namespace can reach it.
- Whether `jvm_gc_pause_seconds_sum`, `http_server_requests_seconds_sum` and histogram buckets are available.
- Whether missing container metrics are already available through Dynatrace's native Kubernetes monitoring.

## 3. Discovery commands — read-only

### 3.1 EKS and Dynatrace

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get dynakubes -A
kubectl get pods -A | grep -Ei 'dynatrace|oneagent|activegate'
kubectl get daemonsets,deployments -A | grep -Ei 'dynatrace|oneagent|activegate'
helm list -A | grep -Ei 'dynatrace|prometheus|opentelemetry'
```

Inspect DynaKube configuration **without printing secrets**:

```bash
kubectl get dynakubes -A -o json | jq '
  .items[] | {
    namespace: .metadata.namespace,
    name: .metadata.name,
    apiVersion: .apiVersion,
    activeGateCapabilities: .spec.activeGate.capabilities,
    kubernetesMonitoring: .spec.kubernetesMonitoring,
    oneAgentModes: (.spec.oneAgent | keys?)
  }'
```

Check the deployed Operator's actual CRD schema/version before changing a DynaKube. Existing monitoring may use `spec.activeGate.capabilities` or a version-specific Kubernetes-monitoring configuration.

### 3.2 Prometheus installation and target discovery

```bash
kubectl get pods,svc -A | grep -Ei 'prometheus|kube-state|node-exporter'
kubectl get crd | grep -Ei 'servicemonitors|podmonitors' || true
kubectl get servicemonitors,podmonitors -A  # Only if the CRDs exist
```

Port-forward the **actual** Prometheus service after discovering its namespace and name:

```bash
kubectl -n <prometheus-namespace> port-forward svc/<prometheus-service> 9090:9090
```

In another shell:

```bash
export PROM_URL=http://localhost:9090
curl -fsS "$PROM_URL/api/v1/label/__name__/values" \
  | jq -r '.data[]' | sort

curl -fsS "$PROM_URL/api/v1/targets?state=active" \
  | jq -r '.data.activeTargets[] | [.labels.job, .labels.instance, .health, .scrapeUrl] | @tsv'
```

Prometheus UI queries:

```promql
up
count by (job) (up)
count by (job) (up == 1)
up == 0
prometheus_tsdb_head_series
```

Do not circulate target URLs containing credentials or internal-sensitive paths. Scrape endpoints and metric labels can expose internal topology.

## 4. Prometheus dashboards — initial query catalogue

Use **Graph** for time series and **Table** for current state. Queries assume the usual exporter labels; inspect actual label sets before relying on namespace, pod, job or HTTP status grouping.

### Dashboard A — EKS infrastructure and workload health

**Node CPU utilisation (%) — Graph**

```promql
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

**Top 10 busiest node-exporter instances — Graph/Table**

```promql
topk(10, 100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))))
```

**Pod count by namespace and phase — Table**

```promql
sum by (namespace, phase) (kube_pod_status_phase == 1)
```

**Not-ready pods — Table**

```promql
kube_pod_status_ready{condition="false"} == 1
```

Also check `condition="unknown"` where relevant.

**Restarts over the last hour — Table**

```promql
sum by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[1h])) > 0
```

**Last termination recorded as OOMKilled — Table**

```promql
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
```

This is **not** a historical count of OOM events.

**Configured memory limits (MiB) — Table**

```promql
kube_pod_container_resource_limits{resource="memory",unit="byte"} / 1024 / 1024
```

**Configured CPU limits (cores) — Table**

```promql
kube_pod_container_resource_limits{resource="cpu",unit="core"}
```

Resource limits do not indicate actual usage. Verify `resource` and `unit` labels in your installation.

### Dashboard B — JVM application performance

**Heap memory (MiB) — Graph**

```promql
sum by (job) (jvm_memory_used_bytes{area="heap"}) / 1024 / 1024
```

**GC events per second — Graph**

```promql
sum by (job) (rate(jvm_gc_pause_seconds_count[5m]))
```

**GC pause time per second — Graph; requires additional metric**

```promql
sum by (job) (rate(jvm_gc_pause_seconds_sum[5m]))
```

**Average GC pause duration (seconds/event) — Graph; requires `_sum`**

```promql
sum by (job) (rate(jvm_gc_pause_seconds_sum[5m]))
/
sum by (job) (rate(jvm_gc_pause_seconds_count[5m]))
```

**Top 10 process memory series (MiB) — Graph/Table**

```promql
topk(10, process_resident_memory_bytes / 1024 / 1024)
```

**Process CPU — Graph; verify gauge semantics**

```promql
100 * process_cpu_usage
```

`process_cpu_usage` is commonly a fractional Micrometer gauge. Verify `HELP`/`TYPE` before interpreting its units.

### Dashboard C — HTTP service performance

**Requests per second by job — Graph**

```promql
sum by (job) (rate(http_server_requests_seconds_count[5m]))
```

**5xx errors per second — Graph**

```promql
sum by (job) (rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
```

**5xx error percentage — Graph**

```promql
100 *
sum by (job) (rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
clamp_min(sum by (job) (rate(http_server_requests_seconds_count[5m])), 0.000001)
```

Check that `status` is the correct response-code label. A 5xx percentage does not represent every possible application/business failure.

**Mean request duration — Graph; requires `_sum`**

```promql
sum by (job) (rate(http_server_requests_seconds_sum[5m]))
/
sum by (job) (rate(http_server_requests_seconds_count[5m]))
```

For p95 latency, first verify the histogram representation. Classic buckets support `histogram_quantile(0.95, sum by (job, le) (rate(http_server_requests_seconds_bucket[5m])))`; native histograms require different query syntax.

### Additional metrics to test

```promql
jvm_gc_pause_seconds_sum
http_server_requests_seconds_sum
http_server_requests_seconds_bucket
count by (job) (up{job=~".*kubelet.*|.*cadvisor.*"})
```

## 5. Dynatrace integration options

| Option | Deployment approach | Benefits | Considerations |
|---|---|---|---|
| A. Existing Dynatrace native Kubernetes monitoring | Reuse current DynaKube/ActiveGate/OneAgent | Node, workload, pod, container and topology coverage | Confirm installed mode, permissions and existing data |
| B. OTel Collector scraping Prometheus-format endpoints | Dedicated Helm deployment in EKS | Vendor-neutral, controlled metric ingestion, no Prometheus replacement | Extra scraper and potential duplicated ingestion; counter temporality and label mapping |
| C. Dynatrace Prometheus ingestion/extension | Supported Dynatrace integration for selected endpoints | May minimise separate collector operations | Validate current supported deployment path, feature limits and scale |

**Chosen PoC:** A + B. Existing Prometheus remains in place for baseline and comparison. This PoC does not migrate historical Prometheus TSDB data or automatically translate PromQL dashboards into DQL.

## 6. Proposed PoC topology and security controls

```text
AWS EKS (EC2 nodes)
  ├─ JVM application /actuator/prometheus ──┬── Existing Prometheus
  │                                         └── New OTel Collector (one replica)
  │                                              └── OTLP/HTTP + TLS → Dynatrace
  ├─ kube-state-metrics / node-exporter ──────── Existing Prometheus
  └─ Kubernetes API / node and container telemetry
       └── Existing Dynatrace Operator / ActiveGate → Dynatrace

Dynatrace: Kubernetes views + metric explorer / Notebooks / dashboards
```

Controls:

- Dedicated `observability-poc` namespace and narrowly scoped application target.
- No change to existing Dynatrace or Prometheus Helm releases.
- No privileged host mounts, cluster-wide discovery or Kubernetes API RBAC required for the initial static-target collector.
- Dedicated metric-ingest token stored as a Kubernetes Secret; prefer the organisation's approved secret-management integration for production.
- Restrict outbound network access to the approved Dynatrace endpoint or ActiveGate; use TLS and proxy/private connectivity where required.
- Limit metric families, label cardinality and ingestion volume; review sensitive labels before export.
- Pin reviewed Helm chart and image versions/digests before rollout.
- Avoid printing API tokens, Secrets or full rendered manifests containing credentials.
- Obtain application and platform owner approval for duplicate scraping and ingestion cost.

## 7. Deployment-ready Helm PoC — parameterised, pending validation

This configuration is **ready to adapt and test**, not approved for unattended deployment. Replace placeholders and validate the chart version, Collector configuration and Dynatrace token requirements first.

### 7.1 Prerequisites

- `kubectl` context points to the intended EKS cluster.
- Helm 3 installed and permission to create a namespace, Secret and Deployment.
- An application Service DNS name, port and Prometheus metrics path verified from existing Prometheus targets.
- Dynatrace environment endpoint and appropriate OTLP metric-ingest authentication confirmed.
- Network policy, firewall and proxy routes permit Collector-to-application scraping and Collector-to-Dynatrace egress.

### 7.2 Namespace and Secret

```bash
kubectl create namespace observability-poc

# Example for an environment accepting Dynatrace API-token authentication.
# Confirm the tenant-specific OTLP base endpoint and token scope first.
export DT_ENDPOINT='https://<environment-id>.live.dynatrace.com/api/v2/otlp'
read -rsp 'Dynatrace API token: ' DT_API_TOKEN; echo

kubectl -n observability-poc create secret generic dynatrace-otlp \
  --from-literal=endpoint="$DT_ENDPOINT" \
  --from-literal=token="$DT_API_TOKEN"
unset DT_API_TOKEN
```

For environments requiring a platform token, configure the approved endpoint and `Bearer` authentication rather than assuming `Api-Token` is valid.

### 7.3 `otel-values.yaml`

The upstream chart can merge its default Collector configuration with overrides. **Inspect the rendered ConfigMap and remove unwanted default receivers/pipelines before installing.**

```yaml
mode: deployment
replicaCount: 1

image:
  repository: otel/opentelemetry-collector-contrib
command:
  name: otelcol-contrib

presets:
  logsCollection:
    enabled: false
  hostMetrics:
    enabled: false
  kubeletMetrics:
    enabled: false
  kubernetesAttributes:
    enabled: false
  clusterMetrics:
    enabled: false

extraEnvs:
  - name: DT_OTLP_ENDPOINT
    valueFrom:
      secretKeyRef:
        name: dynatrace-otlp
        key: endpoint
  - name: DT_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: dynatrace-otlp
        key: token

config:
  receivers:
    prometheus:
      config:
        scrape_configs:
          - job_name: eks-jvm-poc
            scrape_interval: 30s
            metrics_path: /actuator/prometheus
            static_configs:
              - targets:
                  - my-app.my-namespace.svc.cluster.local:8080
            metric_relabel_configs:
              - source_labels: [__name__]
                regex: 'jvm_.*|process_.*|http_server_requests_seconds_(count|sum)'
                action: keep

  processors:
    memory_limiter:
      check_interval: 5s
      limit_mib: 400
      spike_limit_mib: 100
    cumulative_to_delta:
      max_staleness: 5m
    batch:
      timeout: 10s

  exporters:
    otlphttp/dynatrace:
      endpoint: ${env:DT_OTLP_ENDPOINT}
      headers:
        Authorization: "Api-Token ${env:DT_API_TOKEN}"

  service:
    pipelines:
      metrics:
        receivers: [prometheus]
        processors: [memory_limiter, cumulative_to_delta, batch]
        exporters: [otlphttp/dynatrace]

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**Configuration notes:**

- Verify that the chosen Collector Contrib version includes the `prometheus` receiver and `cumulative_to_delta` processor and that Dynatrace supports the emitted metric temporality.
- Check the actual Prometheus metric naming/label transformations after export.
- The sample metric relabel rule limits names, but does **not** cap label cardinality or block sensitive labels. Review and add label filtering where necessary.
- A single replica with static targets avoids duplicate scrape streams. For multiple replicas, use a deliberate sharding/target-allocation strategy.
- Set a Kubernetes resource limit higher than the Collector's internal memory limiter; tune both based on actual load.
- The Dynatrace exporter endpoint is the OTLP **base** URL; the exporter adds the metrics path as appropriate.
- Helm chart defaults may include extra receiver and pipeline configuration. Validate them rather than assuming this values fragment replaces the entire default config.

### 7.4 Render, validate and install

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm search repo open-telemetry/opentelemetry-collector --versions | head

# Pin a reviewed chart version before installation.
export OTEL_CHART_VERSION='<approved-chart-version>'

helm template otel-dynatrace-poc open-telemetry/opentelemetry-collector \
  --version "$OTEL_CHART_VERSION" \
  -n observability-poc -f otel-values.yaml > rendered.yaml

# Review ConfigMap, Deployment, image, pipelines, permissions and service account.
# Prefer: helm lint, peer review, and a dry-run in the approved environment.
helm lint open-telemetry/opentelemetry-collector \
  --version "$OTEL_CHART_VERSION" -f otel-values.yaml

helm upgrade --install otel-dynatrace-poc \
  open-telemetry/opentelemetry-collector \
  --version "$OTEL_CHART_VERSION" \
  -n observability-poc -f otel-values.yaml \
  --wait --timeout 5m

kubectl get pods,deployments -n observability-poc
kubectl logs -n observability-poc \
  -l app.kubernetes.io/instance=otel-dynatrace-poc \
  --all-containers --tail=100
```

**Before deployment:** Confirm whether the selected chart requires `ports`, `service`, or other chart-specific settings, and validate the final Collector configuration with the pinned binary/image. A successful Helm installation is not evidence of successful ingestion.

## 8. Dynatrace validation and dashboard mapping

First use Dynatrace metric discovery to find the **actual ingested metric keys** and dimensions. Do not assume Prometheus names and label keys survive ingestion unchanged.

| Prometheus panel | Dynatrace source | Parity check |
|---|---|---|
| JVM heap memory | OTel-ingested JVM metrics | MiB, job/pod dimensions, time trend |
| JVM GC frequency | OTel-ingested GC count | Events/second over matching intervals |
| Process resident memory | OTel-ingested process metric | Same instance and bytes/MiB conversion |
| HTTP throughput | OTel-ingested HTTP counter | Requests/second over matching intervals |
| HTTP 5xx percentage | OTel-ingested HTTP counter | Response-code labels and same denominator |
| Node CPU | Existing native Dynatrace monitoring | Comparable nodes and utilisation semantics |
| Pod phase/readiness | Existing Kubernetes monitoring | Same namespaces, workloads and observation time |
| Container restarts | Existing Kubernetes monitoring | Counter versus event/state semantics |
| Container CPU/memory | Existing native monitoring, if available | Document whether native coverage closes Prometheus gap |

If using DQL, build queries only after verifying metric keys, metric type and dimensions in Dynatrace Notebooks or Data Explorer. Validate counter-to-delta conversion and rate calculations before interpreting GC or HTTP metrics.

## 9. Acceptance criteria

- [ ] Existing Dynatrace installation and Kubernetes monitoring mode documented.
- [ ] Prometheus jobs, healthy targets, selected app endpoint and relevant labels recorded.
- [ ] Native Dynatrace EKS nodes, workloads and container metrics assessed.
- [ ] OTel Collector runs as one approved Deployment without changes to existing releases.
- [ ] JVM memory, GC and HTTP metric families appear in Dynatrace.
- [ ] At least one JVM application has comparable Prometheus/Dynatrace metrics for the same time window.
- [ ] Any differences in metric names, labels, units, aggregation and counter temporality documented.
- [ ] No unexpected duplicate series, sensitive attributes or unacceptable ingestion/cardinality growth.
- [ ] Logs show no persistent scrape, authentication, throttling or exporter failures.
- [ ] Rollback demonstrated without disrupting Prometheus or existing Dynatrace monitoring.

## 10. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Duplicate metrics or costs | Scope to one application and allowlisted metric families; compare existing Dynatrace keys |
| Scrape failures | Reuse known-good Prometheus targets; test network/DNS/path/port from Collector namespace |
| Counter/rate mismatch | Verify metric temporality, cumulative-to-delta processing and equivalent rollups |
| Missing labels or high cardinality | Review label inventory; remove sensitive/high-cardinality dimensions |
| Secret leakage | Kubernetes Secret, limited access, no tokens in Git or screenshots |
| Unapproved cluster impact | Single replica, bounded CPU/memory, no broad scrape or Operator changes |
| Missing container metrics | Verify native Dynatrace coverage before enabling kubelet/cAdvisor scraping |
| Chart/version drift | Pin chart and image versions; render, lint and review before deployment |
| Production alert mismatch | Keep existing alerting until validated; no automatic replacement in PoC |

## 11. Rollback

```bash
helm uninstall otel-dynatrace-poc -n observability-poc
kubectl -n observability-poc delete secret dynatrace-otlp
# Delete the namespace only after confirming it contains no unrelated resources.
```

The existing Prometheus and Dynatrace deployments remain untouched.

## 12. Suggested delivery sequence and decisions

| Stage | Activity | Deliverable |
|---|---|---|
| 1 | Read-only EKS, Prometheus and Dynatrace discovery | Inventory and gap report |
| 2 | Validate PromQL dashboards and labels | Infrastructure/JVM/HTTP baseline |
| 3 | Approve endpoint, egress, token and chart version | Deployment change approval |
| 4 | Deploy one OTel Collector | First Dynatrace-ingested metric set |
| 5 | Compare with native Dynatrace Kubernetes data | Parity and added-value assessment |
| 6 | Review cost, security, resilience and scaling | Production recommendation |

**Decisions requested from technical team:**

1. Which JVM application and namespace are approved for the first scrape?
2. Which Dynatrace tenant, endpoint and token type should be used?
3. Is existing native Dynatrace Kubernetes monitoring complete enough to cover container CPU/memory?
4. What metric-ingest/cardinality budget and network egress policy apply?
5. Who owns the PoC dashboards and sign-off criteria?

## 13. References

- OpenTelemetry Collector Helm charts: https://github.com/open-telemetry/opentelemetry-helm-charts
- OpenTelemetry Collector configuration: https://opentelemetry.io/docs/collector/configuration/
- OpenTelemetry Prometheus receiver: https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/prometheusreceiver
- Dynatrace Prometheus ingestion: https://docs.dynatrace.com/docs/ingest-from/extend-dynatrace/extend-metrics/ingestion-methods/prometheus
- Dynatrace OTel Collector Prometheus use case: https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector-use-cases/prometheus
- Dynatrace Kubernetes deployment: https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/deployment
- Prometheus query language: https://prometheus.io/docs/prometheus/latest/querying/basics/

---

*This document is based on user-reported metric discovery and the stated EKS/Dynatrace environment. No live cluster configuration, metric values or Dynatrace deployment details have yet been independently verified.*
