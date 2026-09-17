---
tags: [research, monitoring, prometheus, grafana, alertmanager, pve-exporter, ntfy]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Monitoring & Alerting Stack (Prometheus + Grafana)

Research date: 2026-09-17.

## Architecture

- node_exporter on the PVE host (and each Linux guest/LXC you care about) - CPU, RAM, disk, net, temps.
- prometheus-pve-exporter (port 9221, official/first-party, prompve/prometheus-pve-exporter) - talks to the PVE REST API with a read-only API token (PVEAuditor) and exposes pve_* metrics.
- Prometheus :9090 or VictoriaMetrics :8428 as TSDB.
- Grafana :3000 + Alertmanager :9093.
- Notifications: ntfy (self-host or ntfy.sh) or Telegram.

VictoriaMetrics vs Prometheus: VM = single Go binary, drop-in (accepts prometheus.yml, PromQL), ~2.5x less disk and roughly half the RAM. Start with Prometheus; switch to VM only if you want >6 months retention. Grafana sees VM as a prometheus datasource either way.

## Deployment

Docker-compose in one Debian 12 unprivileged LXC. RAM total 1.5-2GB, 2 cores, ~10-20GB disk. Back up the data volume or mount a ZFS dataset.

## Metrics to track

- Host (node_exporter): node_cpu_seconds_total, node_memory_MemAvailable_bytes, node_filesystem_avail_bytes, node_network_receive_bytes_total, node_hwmon_temp_celsius.
- PVE (pve-exporter): pve_up{id="node/x|qemu/100|lxc/101|storage/..."}, pve_disk_usage_bytes, pve_disk_size_bytes, pve_memory_usage_bytes, pve_cpu_usage_ratio, pve_guest_info{vmid,status,type}, pve_ha_state, pve_subscription_info.
- ZFS: node_exporter covers ARC/io but NOT pool health - add a textfile collector running zpool status (metric node_zpool_health{pool}).
- Guest in-VM fs usage needs a node_exporter per guest; pve-exporter only shows allocated disk.

## pve-exporter setup

```yaml
# pve.yml (mount to /etc/prometheus/pve.yml)
default:
  user: monitoring@pve
  token_name: prometheus
  token_value: "uuid-secret"
  verify_ssl: false
```

```yaml
# prometheus.yml scrape job
- job_name: pve
  metrics_path: /pve
  params: {module: [default], cluster: ['1'], node: ['1']}
  static_configs: [{targets: ['10.10.10.x:9221']}]
```

```yaml
services:
  prometheus:    {image: prom/prometheus:v3.3.0, volumes: [prom_data:/prometheus]}
  grafana:       {image: grafana/grafana:12.4.1, ports: ["3000:3000"], volumes: [grafana_data:/var/lib/grafana]}
  alertmanager:  {image: prom/alertmanager:v0.28.0, volumes: [./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro]}
  pve-exporter:  {image: prompve/prometheus-pve-exporter:latest, volumes: [./pve.yml:/etc/prometheus/pve.yml:ro], ports: ["9221:9221"]}
```

## Grafana + alerting quick win

- Dashboards: 10347 "Proxmox via Prometheus" (current reference), 1860 "Node Exporter Full", 24550 (adds ZFS + sensors). One Prometheus datasource, pick instance.

```yaml
# alert rules (rules.yml)
groups:
- name: homelab
  rules:
  - alert: NodeDown
    expr: up{job=~"node.*|pve"} == 0
    for: 5m
    labels: {severity: critical}
    annotations: {summary: "{{ $labels.instance }} down ({{ $labels.job }})"}
  - alert: DiskSpaceHigh
    expr: node_filesystem_avail_bytes / node_filesystem_size_bytes * 100 < 10
    for: 10m
    labels: {severity: warning}
  - alert: PVEGuestDown
    expr: pve_up{id=~"qemu/.+|lxc/.+"} == 0
    for: 5m
    labels: {severity: critical}
  - alert: ZFSPoolDegraded
    expr: node_zpool_health != 1
    for: 5m
    labels: {severity: critical}
```

- Wire Alertmanager -> ntfy via webhook (POST /publish?topic=); Prometheus can webhook directly to ntfy, skip Alertmanager at first.

## Effort guidance (solo lab - don't overbuild)

- Phase 1 (day 1, ~1hr): node_exporter + pve-exporter + Prometheus + Grafana, dashboards 10347+1860. No alerting.
- Phase 2 (+30min): ntfy topic + 3 alert rules via Prometheus rules.
- Phase 3 (when needed): Alertmanager (routing/templates), ZFS textfile collector, per-guest node_exporter, Uptime Kuma.

## Sources

- https://github.com/prometheus-pve/prometheus-pve-exporter
- https://grafana.com/grafana/dashboards/10347-proxmox-via-prometheus
- https://grafana.com/grafana/dashboards/1860-node-exporter-full
- https://github.com/prometheus/node_exporter
- https://docs.victoriametrics.com/single-server-victoriametrics