# PromQL Queries for AWS EC2 Monitoring

These queries target the control node and three worker-node jobs in the example configuration. Update the job names if your Prometheus configuration changes.

## Target availability

```promql
up{job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"}
```

## Count of available monitored Linux nodes

```promql
sum(up{job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"})
```

## CPU utilization (%)

```promql
100 - (
  avg by (instance, server_name) (
    rate(node_cpu_seconds_total{
      mode="idle",
      job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
    }[5m])
  ) * 100
)
```

## Memory utilization (%)

```promql
100 * (
  1 - (
    node_memory_MemAvailable_bytes{
      job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
    }
    /
    node_memory_MemTotal_bytes{
      job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
    }
  )
)
```

## Root filesystem utilization (%)

```promql
100 * (
  1 -
  (
    node_filesystem_avail_bytes{
      mountpoint="/",
      fstype!~"tmpfs|overlay",
      job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
    }
    /
    node_filesystem_size_bytes{
      mountpoint="/",
      fstype!~"tmpfs|overlay",
      job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
    }
  )
)
```

## Network receive throughput (bits/second)

```promql
rate(node_network_receive_bytes_total{
  device!="lo",
  job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
}[5m]) * 8
```

## Network transmit throughput (bits/second)

```promql
rate(node_network_transmit_bytes_total{
  device!="lo",
  job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
}[5m]) * 8
```
