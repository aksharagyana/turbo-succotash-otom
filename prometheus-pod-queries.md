
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

## References

- [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [cAdvisor metric catalogue](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)
- [kube-state-metrics pod metrics](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md)
- [Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)
