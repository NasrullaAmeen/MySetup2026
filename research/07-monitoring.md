---
tags: [research, homelab, proxmox, monitoring, prometheus, grafana]
created: 2026-09-17 21:24:01 +05
modified: 2026-09-17 21:42:45 +05
---

# Monitoring - Prometheus, pve-exporter, Grafana

Research date: 2026-09-17.

## Prometheus stack for PVE

`prometheus-pve-exporter` (v3.10, PyPI prometheus-pve-exporter) proxies the Proxmox REST API to Prometheus as a multi-target proxy on .pve path. One exporter covers a cluster. Use a read-only API token:

```
pveum user add monitoring@pve -comment "monitoring"
pveum role add monitoring -privs "VM.Audit,Datastore.Audit,Sys.Audit,SDN.Audit"
pveum aclmod / -user monitoring@pve -role monitoring
pveum user token add monitoring@pve monitoring --privsep 0   # privsep=0 critical, else 401
```

pve.yml:

```yaml
default:
  user: monitoring@pve
  token_name: monitoring
  token_value: "xxxx-...-uuid-secret"
  verify_ssl: false
```

Prometheus scrape:

```yaml
- job_name: proxmox
  metrics_path: /pve
  params: {module: [default], cluster: ["1"], node: ["1"], target: ["PVE_IP"]}
  static_configs: [{targets: ["pve-exporter:9221"]}]
```

Test: `curl "http://localhost:9221/pve?target=<pve-ip>&cluster=1&node=1"`.

## Alternatives (1 node)

| Tool | Effort | Capability | Verdict |
|------|--------|-----------|---------|
| Netdata | ~5 min | real-time, VM/LXC discovery, ZFS/SMART; weak retention | best fast option |
| Zabbix | medium-high | strong alerts, PVE template | heaviest |
| Telegraf/Grafana Agent | low-medium | metric shipping only | needs Prom/Grafana |
| Prometheus+Grafana | medium-high | most flexible, history queries | best long-term |

## Dashboards (Grafana.com)

| ID | Name |
|----|------|
| 24550 | Proxmox VE - pve-exporter (newest, node/VM/LXC/storage/ZFS/sensors) |
| 10347 | Proxmox via Prometheus (Saccardi, classic) |
| 1860 | Node Exporter Full (temp/SMART) |
| 16514 / 22381 | SMART/NVMe drive health (smartctl_exporter) |

## Alert rules (Alertmanager / Grafana)

```yaml
- alert: PVENodeDown
  expr: pve_up{id=~"node/.*"} == 0
  for: 2m
- alert: PVEStorageAlmostFull
  expr: pve_disk_usage_bytes{id=~"storage/.*"} / pve_disk_size_bytes{id=~"storage/.*"} * 100 > 95
  for: 2m
- alert: PVEVMCTDown
  expr: pve_up{id=~"(qemu|lxc)/.*"} == 0
  for: 5m
- alert: PVEHighCPU
  expr: pve_cpu_usage_ratio * 100 > 90
  for: 5m
```

Also add exporter-up rule (`up{job="proxmox"}==0`) to avoid a silent blind spot, plus ZFS ZED + smartd. Backup-failure alerts via PVE notification system.

## PVE built-in vs external

- Web UI RRD graphs: live-only, no alerting. PVE 9.x ships external Metric Server (Graphite / InfluxDB / OpenTelemetry) - pure export, no alerting.
- Go external for trends > 24h, real alerts, multi-guest queries - worth it immediately for a homelab.

## Deployment (unprivileged LXC)

Run Prometheus/Grafana/Alertmanager/pve-exporter in an unprivileged Debian LXC via Docker Compose; pve-exporter bound to 127.0.0.1:9221; Prometheus `--storage.tsdb.retention.time=30d`; Grafana behind reverse proxy or Tailscale. On host: node_exporter (`--collector.zfs --collector.diskstats`) + smartctl_exporter. Slow HDD-backed LXC: 30-60 s scrape, 14-30 d retention.

## Sources

- https://github.com/prometheus-pve/prometheus-pve-exporter
- https://www.nxsi.io/blog/proxmox-monitoring-grafana-guide
- https://samber.github.io/awesome-prometheus-alerts/
- https://pve.proxmox.com/wiki/External_Metric_Server
- https://grafana.com/grafana/dashboards/24550-proxmox-ve-pve-exporter/