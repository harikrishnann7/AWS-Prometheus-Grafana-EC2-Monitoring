# AWS EC2 Infrastructure Monitoring with Prometheus and Grafana

A hands-on AWS project that monitors four Ubuntu EC2 Linux instances using Prometheus, Node Exporter, and Grafana.

## Project overview

This lab uses one monitoring/control EC2 instance and three worker EC2 instances. Node Exporter exposes Linux host metrics on each monitored node, Prometheus scrapes those metrics, and Grafana visualizes them in a dashboard.

### Architecture

```text
                 +---------------------------+
                 |       Grafana :3000       |
                 | AWS EC2 Infrastructure    |
                 | Monitoring dashboard     |
                 +-------------+-------------+
                               |
                               | Prometheus data source
                               v
                 +---------------------------+
                 |      Prometheus :9090     |
                 | Scrapes targets every 15s |
                 +-------------+-------------+
                               |
             +-----------------+------------------+
             |                 |                  |
             v                 v                  v
      Control EC2         Worker EC2 #1      Worker EC2 #2/#3
      Node Exporter       Node Exporter       Node Exporter
        :9100               :9100               :9100
```

## Lab inventory

| Role | EC2 name / label | Private IP |
|---|---|---|
| Monitoring and control | Prometheus host | 172.31.12.25 |
| Worker 1 | noob-1-prometheus | 172.31.0.18 |
| Worker 2 | noob-2-prometheus | 172.31.11.186 |
| Worker 3 | noob-3-prometheus | 172.31.12.138 |

These are private addresses from the lab environment. They can change if instances are replaced or private addressing is changed. Update the Prometheus scrape configuration accordingly.

## Components

- AWS EC2 running Ubuntu
- Prometheus 3.16.0
- Grafana 13.3.1
- Prometheus Node Exporter (including 1.10.2 on a worker node)
- systemd for service management
- PromQL for querying metrics

## Ports and connectivity

| Component | Port | Purpose |
|---|---:|---|
| Grafana | TCP 3000 | Dashboard web interface |
| Prometheus | TCP 9090 | Metrics UI and API |
| Node Exporter | TCP 9100 | Linux host metrics endpoint |

**Security:** Do not expose port 9100 or Prometheus port 9090 publicly. Restrict inbound access using EC2 security groups. Allow TCP 9100 only from the monitoring instance's security group (recommended) or its private IP. Restrict Grafana access to trusted administrator IPs or a secure VPN/bastion path.

## Prometheus scrape configuration

Example configuration for this lab. Save as `/etc/prometheus/prometheus.yml` on the monitoring instance.

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "control-node"
    static_configs:
      - targets: ["localhost:9100"]
        labels:
          server_name: "prometheus"

  - job_name: "noob-1-prometheus"
    static_configs:
      - targets: ["172.31.0.18:9100"]
        labels:
          server_name: "noob-1-prometheus"

  - job_name: "noob-2-prometheus"
    static_configs:
      - targets: ["172.31.11.186:9100"]
        labels:
          server_name: "noob-2-prometheus"

  - job_name: "noob-3-prometheus"
    static_configs:
      - targets: ["172.31.12.138:9100"]
        labels:
          server_name: "noob-3-prometheus"
```

Validate and reload/restart Prometheus after changing the configuration:

```bash
sudo /opt/prometheus/promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
sudo systemctl status prometheus --no-pager
```

## Install and run Node Exporter

Run on each Ubuntu EC2 instance that should be monitored. The following is a basic example; verify the current official release and checksum before using it in a new environment.

```bash
# Create a dedicated service account
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter

# Download and unpack the chosen official Node Exporter release,
# then install the node_exporter binary in /usr/local/bin.
# Example service unit:
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter --no-pager
```

Check the endpoint locally:

```bash
curl -s http://localhost:9100/metrics | head
```

If Node Exporter was installed from the Ubuntu package, the service may instead be named `prometheus-node-exporter`. Use the service name that exists on that host.

## Grafana setup

1. Open Grafana at `http://<monitoring-ec2-address>:3000` through a secured network path.
2. Add a Prometheus data source.
3. Set the URL to `http://localhost:9090` when Grafana and Prometheus run on the same EC2 instance.
4. Save and test the data source.
5. Create or import panels using the PromQL examples below.
6. The lab dashboard is named **AWS EC2 Infrastructure Monitoring**.

## Useful PromQL queries

### Target availability

```promql
up{job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"}
```

### Number of available monitored Linux nodes

```promql
sum(up{job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"})
```

### CPU utilization percentage

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

### Memory utilization percentage

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

### Root filesystem utilization percentage

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

### Network receive throughput (bits per second)

```promql
rate(node_network_receive_bytes_total{
  device!="lo",
  job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
}[5m]) * 8
```

### Network transmit throughput (bits per second)

```promql
rate(node_network_transmit_bytes_total{
  device!="lo",
  job=~"control-node|noob-1-prometheus|noob-2-prometheus|noob-3-prometheus"
}[5m]) * 8
```

## Validation and troubleshooting

On the Prometheus host:

```bash
# Check services
sudo systemctl status prometheus --no-pager
sudo systemctl status grafana-server --no-pager

# Check listening ports
sudo ss -lntp | grep -E ':3000|:9090|:9100'

# Check Prometheus targets in the browser
# http://<monitoring-host>:9090/targets

# Check the Prometheus API
curl -s 'http://localhost:9090/api/v1/query?query=up'
```

If a target is down:

1. Confirm Node Exporter is active on the target EC2 instance.
2. Run `curl -s http://localhost:9100/metrics | head` on that instance.
3. From the monitoring host, test `curl -s --max-time 5 http://<worker-private-ip>:9100/metrics | head`.
4. Check the worker and monitoring EC2 security groups, subnet routing, and network ACLs.
5. Check the target address and Prometheus configuration, then validate with `promtool`.

If Grafana shows **No data**, first confirm the Prometheus data source test succeeds, then check the Prometheus Targets page and run `up` in Prometheus.

## AWS cost and operational notes

- Stop or terminate lab instances when they are not needed.
- Remember that stopped EC2 instances can still incur EBS storage charges; public IPv4 addresses and other attached resources may also incur charges.
- Check EC2 vCPU quotas in the correct AWS Region and instance-family quota before launching additional instances.
- Set an AWS Budget and billing alerts.
- Do not commit AWS access keys, private SSH keys, passwords, tokens, or sensitive account information.

## Project status and scope

This is a learning/lab implementation demonstrating Linux infrastructure monitoring on AWS. It is not a production-hardened monitoring platform. A production deployment should additionally consider TLS, authentication and network segmentation, durable storage and backups, alert rules and notification routing, high availability, retention sizing, and controlled configuration management.

## Skills demonstrated

AWS EC2 | Ubuntu Linux | Prometheus | Grafana | Node Exporter | systemd | Linux troubleshooting | PromQL | Monitoring and observability | AWS security groups | Cloud operations
