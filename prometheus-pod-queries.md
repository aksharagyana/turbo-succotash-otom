# Kubernetes pod monitoring with Prometheus

14 practical queries and six checks to see which metrics your server collects.

## How to use

Start in Prometheus Table view, then switch to Graph for time trends. Add namespace="your-namespace" to selectors to narrow the results.

- CPU, memory, network and filesystem queries require kubelet/cAdvisor metrics.
- Pod health queries require kube-state-metrics.
- PVC queries require kubelet volume metrics and a supporting storage driver.
- These queries use standard PromQL supported by Prometheus 2.x and 3.x. Metric availability, names and labels depend on exporter versions and scrape configuration.
- Assume one cluster and one scrape series per container. For multiple clusters, add your cluster label to every grouping and join and filter to the intended scrape jobs.
- Network queries use max per pod/interface to reduce repeated observations of shared pod networking. Check your runtime's labels and accounting; host-network pods may not have isolated pod traffic.
- Five-minute rate windows need enough samples; increase the window for long scrape intervals.
- An empty result may mean no matching problem, missing metrics, or filters that do not match your labels.

## 14 demonstration queries

### 1. Top 10 pods by CPU usage

CPU cores; 0.5 means half a core.

```promql
topk(10,
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{pod!="", container!="", container!="POD"}[5m])
)
)
```

### 2. Top 10 pods by memory usage

Memory working set in MiB.

```promql
topk(10,
sum by (namespace, pod) (
  container_memory_working_set_bytes{pod!="", container!="", container!="POD"}
) / 1024 / 1024
)
```

### 3. CPU throttled periods

Percentage of CPU enforcement periods in which throttling occurred; not percentage of CPU capacity lost.

```promql
100 *
sum by (namespace, pod) (
  rate(container_cpu_cfs_throttled_periods_total{pod!="", container!="", container!="POD"}[5m])
)
/
(
sum by (namespace, pod) (
  rate(container_cpu_cfs_periods_total{pod!="", container!="", container!="POD"}[5m])
) > 0
)
```

### 4. Top 10 pods by received network traffic

MiB per second.

```promql
topk(10,
sum by (namespace, pod) (
  max by (namespace, pod, interface) (
    rate(container_network_receive_bytes_total{pod!="", interface!="lo"}[5m])
  )
) / 1024 / 1024
)
```

### 5. Top 10 pods by transmitted network traffic

MiB per second.

```promql
topk(10,
sum by (namespace, pod) (
  max by (namespace, pod, interface) (
    rate(container_network_transmit_bytes_total{pod!="", interface!="lo"}[5m])
  )
) / 1024 / 1024
)
```

### 6. Network packet drops

Received plus transmitted packet drops per second.

```promql
sum by (namespace, pod) (
  max by (namespace, pod, interface) (
    rate(container_network_receive_packets_dropped_total{pod!="", interface!="lo"}[5m])
  )
)
+
sum by (namespace, pod) (
  max by (namespace, pod, interface) (
    rate(container_network_transmit_packets_dropped_total{pod!="", interface!="lo"}[5m])
  )
)
```

### 7. Top 10 pods by container filesystem usage

GiB. Typically container writable-layer usage; not total mounted PVC usage.

```promql
topk(10,
sum by (namespace, pod) (
  container_fs_usage_bytes{pod!="", container!="", container!="POD"}
) / 1024 / 1024 / 1024
)
```

### 8. Top 10 pods by disk reads

MiB per second.

```promql
topk(10,
sum by (namespace, pod) (
  rate(container_fs_reads_bytes_total{pod!="", container!="", container!="POD"}[5m])
) / 1024 / 1024
)
```

### 9. Top 10 pods by disk writes

MiB per second.

```promql
topk(10,
sum by (namespace, pod) (
  rate(container_fs_writes_bytes_total{pod!="", container!="", container!="POD"}[5m])
) / 1024 / 1024
)
```

### 10. Persistent volume usage

PVC capacity used, in percent. Grouped by PVC rather than pod; requires volume-driver support.

```promql
100 *
max by (namespace, persistentvolumeclaim) (
  kubelet_volume_stats_used_bytes
)
/
(
  max by (namespace, persistentvolumeclaim) (
    kubelet_volume_stats_capacity_bytes
  ) > 0
)
```

### 11. Pods that restarted in the last hour

Estimated restart count during the last hour; increase() can return fractional values.

```promql
sum by (namespace, pod) (
  increase(kube_pod_container_status_restarts_total[1h])
) > 0
```

### 12. Containers in CrashLoopBackOff

Matching containers have value 1.

```promql
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1
```

### 13. Containers last terminated by OOM

Last recorded termination reason; this is not restricted to recent events.

```promql
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
```

### 14. Running pods that are not ready

Matches running pods whose Ready condition is false.

```promql
(kube_pod_status_ready{condition="true"} == 0)
and on (namespace, pod)
(kube_pod_status_phase{phase="Running"} == 1)
```

## Six metric availability checks

Run each separately. A positive count means current series exist; an empty result means none currently match. These check presence, not correctness or scrape health.

### Check 1: container_cpu_usage_seconds_total

```promql
count(container_cpu_usage_seconds_total)
```

### Check 2: container_memory_working_set_bytes

```promql
count(container_memory_working_set_bytes)
```

### Check 3: container_network_receive_bytes_total

```promql
count(container_network_receive_bytes_total)
```

### Check 4: container_fs_writes_bytes_total

```promql
count(container_fs_writes_bytes_total)
```

### Check 5: kubelet_volume_stats_used_bytes

```promql
count(kubelet_volume_stats_used_bytes)
```

### Check 6: kube_pod_container_status_restarts_total

```promql
count(kube_pod_container_status_restarts_total)
```

## Bonus: Prometheus version

Read the version label; the sample value is always 1.

```promql
prometheus_build_info
```


```promql
count by (instance, version) (prometheus_build_info)
```

This requires Prometheus self-scraping. Alternatively, open /api/v1/status/buildinfo on the server. In an aggregated data source, build metrics may describe upstream scraped servers.

```promql
sum by (namespace, pod) (
  increase(kube_pod_container_status_restarts_total[1h])
) > 0
```

```promql
kube_pod_container_status_waiting_reason{
  reason="CrashLoopBackOff"
} == 1
```

```promql
kube_pod_container_status_last_terminated_reason{
  reason="OOMKilled"
} == 1
```

```promql
sum by (namespace) (
  rate(pod_cpu_usage_seconds_total{pod!=""}[5m])
)  
```

```promql
sum(
  rate(pod_cpu_usage_seconds_total{pod!=""}[5m])
)
```

```promql
sum by (namespace) (
  pod_memory_working_set_bytes{pod!=""}
) / 1024 / 1024 / 1024
```

```promql
sum(
  pod_memory_working_set_bytes{pod!=""}
) / 1024 / 1024 / 1024
```

```promql
count by (__name__) (
  {__name__=~"pod_(cpu|memory)_.*"}
)
```

```promql
count by (__name__) (
  {__name__=~".*(cpu_usage|memory_working_set|memory_usage).*"}
)
```

```promql
process_cpu_usage
```


```promql
max by (namespace, pod) (
  process_cpu_usage
) * 100
```

```promql
count by (__name__) (
  {__name__=~"(jvm_memory_.*|process_resident_memory_.*|process_virtual_memory_.*)"}
)
```


```promql
sum by (namespace, pod, instance) (
  jvm_memory_used_bytes{area="heap"}
) / 1024 / 1024
```


```promql
sum by (namespace, pod, instance) (
  jvm_memory_used_bytes{area="nonheap"}
) / 1024 / 1024
```


```promql
process_resident_memory_bytes / 1024 / 1024
```

## References

- [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [cAdvisor metric catalogue](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)
- [kube-state-metrics pod metrics](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md)
- [Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)

